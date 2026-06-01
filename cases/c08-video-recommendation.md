# C08: 视频平台的推荐架构

## 业务场景

某短视频平台（类似抖音/B站），需要为 1 亿 DAU 的用户提供个性化视频推荐。推荐是平台的核心——用户打开 App 看到的每一条视频都是推荐系统选择的，推荐的精准度直接决定了用户停留时长和广告收入。

**已知数据：**
- DAU：1 亿
- 视频库：5000 万条视频
- 每日新增视频：50 万条
- 用户特征维度：约 2000 维（历史行为、兴趣标签、人口统计学等）
- 视频特征维度：约 1000 维（内容标签、时长、质量、热度等）
- 推荐延迟要求：< 200ms（从请求到返回结果）
- 每个用户每次请求返回 10-20 条候选视频

**推荐流程的三个阶段：**
1. **召回（Recall）**：从 5000 万视频中筛选出 500-1000 条候选
2. **粗排（Pre-ranking）**：对候选集做快速打分，选 100-200 条
3. **精排（Ranking）**：用复杂模型对 100-200 条精确打分，取 Top 10-20

**为什么需要三阶段而非一次性排序？**

精排模型（深度神经网络）的单次推理约 10ms，如果对 5000 万视频都做精排，需要 5000 万 × 10ms = 50 万秒——完全不可行。三阶段是精度与性能的权衡。

## 核心挑战

### 挑战 1：召回的覆盖率 vs 速度

召回需要从 5000 万视频中找到 500 条候选。如果只按用户历史兴趣召回，会产生"信息茧房"——用户永远只看到自己已经喜欢的内容。需要在精准召回和探索性召回之间取得平衡。

### 挑战 2：特征实时性

用户 5 分钟前点击了一个美食视频，推荐系统应该在几分钟内捕捉到这个兴趣变化，而不是等到第二天。但实时特征管道的延迟和成本远高于离线管道。

### 挑战 3：新视频的冷启动

50 万条新视频每天入库，它们没有历史互动数据，推荐系统不知道该推给谁。如果新视频得不到曝光，创作者就会离开平台。

### 挑战 4：实时性 vs 准确性的权衡

更复杂的模型更准确但更慢。200ms 的延迟限制下，精排模型的复杂度有上限。

## 设计约束

- 推荐延迟 < 200ms（用户体验硬约束，超过 200ms 用户会感知到卡顿）
- 精排模型更新频率：实时（在线学习）
- 特征存储：用户特征实时更新，视频特征近实时更新
- GPU 推理集群：A100 × 20 张

## 数据库与存储设计

推荐系统的存储架构直接影响延迟和实时性。以下是核心数据表与存储选型。

### MySQL 核心表（持久化存储）

```sql
-- 用户特征表：存储用户画像与实时兴趣
CREATE TABLE user_features (
    user_id          BIGINT PRIMARY KEY,
    nickname         VARCHAR(64) NOT NULL,
    gender           TINYINT COMMENT '0-未知,1-男,2-女',
    age_group        TINYINT COMMENT '年龄段编码',
    city_tier        TINYINT COMMENT '城市等级1-5',
    register_time    DATETIME NOT NULL,
    last_active_time DATETIME NOT NULL,
    -- 兴趣标签（JSON存储，实时更新到Redis后异步落库）
    interest_tags    JSON COMMENT '{"美食":0.82,"科技":0.65,"音乐":0.41}',
    -- 统计特征
    total_watch_cnt  INT DEFAULT 0,
    total_like_cnt   INT DEFAULT 0,
    avg_watch_ratio  FLOAT DEFAULT 0 COMMENT '平均完播率',
    avg_session_dur  INT DEFAULT 0 COMMENT '平均单次会话时长(秒)',
    -- 状态
    is_active        TINYINT DEFAULT 1,
    created_at       DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at       DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_last_active (last_active_time),
    INDEX idx_age_group (age_group)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 视频特征表：视频属性与内容理解结果
CREATE TABLE video_features (
    video_id         BIGINT PRIMARY KEY,
    author_id        BIGINT NOT NULL,
    title            VARCHAR(200) NOT NULL,
    description      VARCHAR(1000),
    duration         INT NOT NULL COMMENT '时长(秒)',
    category_id      INT NOT NULL COMMENT '一级分类',
    sub_category_id  INT COMMENT '二级分类',
    tags             JSON COMMENT '["搞笑","猫咪","日常"]',
    -- 内容理解特征（上传时离线计算）
    visual_embedding BLOB COMMENT '视觉向量(128维float32)',
    text_embedding   BLOB COMMENT '文本向量(128维float32)',
    audio_embedding  BLOB COMMENT '音频向量(64维float32)',
    fusion_embedding BLOB COMMENT '融合向量(128维float32)',
    -- 统计特征
    play_count       BIGINT DEFAULT 0,
    like_count       INT DEFAULT 0,
    comment_count    INT DEFAULT 0,
    share_count      INT DEFAULT 0,
    completion_rate  FLOAT DEFAULT 0 COMMENT '完播率',
    like_rate        FLOAT DEFAULT 0 COMMENT '点赞率',
    -- 冷启动流量池
    traffic_pool     TINYINT DEFAULT 0 COMMENT '0-初始池,1-二级,...,4-全站',
    pool_enter_time  DATETIME COMMENT '进入当前池的时间',
    -- 状态
    status           TINYINT DEFAULT 1 COMMENT '0-下架,1-正常,2-审核中',
    upload_time      DATETIME NOT NULL,
    created_at       DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at       DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_author (author_id),
    INDEX idx_category (category_id),
    INDEX idx_upload_time (upload_time),
    INDEX idx_traffic_pool (traffic_pool, pool_enter_time),
    INDEX idx_play_count (play_count)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 召回索引表：协同过滤的"看了又看"关联
CREATE TABLE recall_index (
    id               BIGINT AUTO_INCREMENT PRIMARY KEY,
    source_video_id  BIGINT NOT NULL COMMENT '源视频',
    target_video_id  BIGINT NOT NULL COMMENT '关联视频',
    score            FLOAT NOT NULL COMMENT '关联强度',
    recall_type      TINYINT NOT NULL COMMENT '1-协同过滤,2-内容相似,3-同作者',
    created_at       DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_source_target (source_video_id, target_video_id, recall_type),
    INDEX idx_source (source_video_id, score DESC),
    INDEX idx_recall_type (recall_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 推荐日志表：记录每次推荐结果，用于离线分析和模型训练
CREATE TABLE recommendation_log (
    id               BIGINT AUTO_INCREMENT PRIMARY KEY,
    request_id       VARCHAR(36) NOT NULL COMMENT '请求唯一ID',
    user_id          BIGINT NOT NULL,
    video_id         BIGINT NOT NULL,
    position         TINYINT NOT NULL COMMENT '推荐位(1-10)',
    recall_channel   VARCHAR(32) COMMENT '召回通道:cf/vector/tag/hot/follow/explore',
    pre_rank_score   FLOAT COMMENT '粗排分数',
    rank_score       FLOAT COMMENT '精排分数',
    final_score      FLOAT COMMENT '最终分数(重排后)',
    -- 用户反馈
    is_exposed       TINYINT DEFAULT 0 COMMENT '是否曝光',
    is_clicked       TINYINT DEFAULT 0 COMMENT '是否点击',
    watch_duration   INT DEFAULT 0 COMMENT '观看时长(秒)',
    is_liked         TINYINT DEFAULT 0,
    is_shared        TINYINT DEFAULT 0,
    is_commented     TINYINT DEFAULT 0,
    -- 实验信息
    experiment_id    VARCHAR(32) COMMENT 'A/B实验ID',
    experiment_group VARCHAR(16) COMMENT '实验分组',
    created_at       DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_time (user_id, created_at),
    INDEX idx_video_time (video_id, created_at),
    INDEX idx_experiment (experiment_id, experiment_group, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 按月分区（日志量大，按月分区便于过期清理）
-- ALTER TABLE recommendation_log PARTITION BY RANGE (TO_DAYS(created_at)) (
--   PARTITION p202601 VALUES LESS THAN (TO_DAYS('2026-02-01')),
--   PARTITION p202602 VALUES LESS THAN (TO_DAYS('2026-03-01')),
--   PARTITION p_future VALUES LESS THAN MAXVALUE
-- );
```

### 存储选型与分层

| 数据类型 | 存储引擎 | 数据量 | 延迟要求 | 更新频率 |
|---------|---------|--------|---------|---------|
| 用户画像特征 | Redis Cluster + MySQL | 1亿用户 | < 5ms | 实时 |
| 视频属性特征 | Redis Cluster + MySQL | 5000万视频 | < 5ms | 近实时(分钟级) |
| 用户行为序列 | Redis List | 最近50条/用户 | < 5ms | 实时 |
| 向量索引(用户塔) | Redis Cache | 1亿×128维 | < 10ms | 小时级 |
| 向量索引(视频塔) | FAISS (IVF-PQ) | 5000万×128维 | < 20ms | 日级全量重建 |
| 协同过滤索引 | Redis Hash + MySQL | 约10亿条关联 | < 10ms | 日级离线计算 |
| 推荐日志 | Kafka → Hive/ClickHouse | 每日10亿+ | 秒级 | 实时写入 |
| 模型参数 | Redis + 本地SSD缓存 | 数GB | < 2ms | 小时级 |

**Redis 内存估算：**
- 用户特征：1亿 × 2KB = 200GB
- 视频特征：5000万 × 1KB = 50GB
- 行为序列：1亿 × 0.5KB = 50GB
- 向量缓存：1亿 × 512B = 50GB
- 协同过滤索引：10亿 × 0.1KB = 100GB
- 总计约 450GB，需 6 节点 Redis Cluster（每节点 128GB，含冗余）

## 请先独立思考（限时 40 分钟）

1. 召回阶段至少需要几路召回策略？每路召回的具体实现是什么？
2. 粗排模型的复杂度如何选择？为什么不能直接用精排模型做粗排？
3. 用户实时兴趣如何捕获？点击一个视频后，多快能反映到推荐结果中？
4. 新视频的冷启动方案：如何让一个零播放的视频获得曝光机会？

---

## 设计解析

### 三阶段推荐架构

```
用户请求 → 召回层（多路召回）→ 粗排层 → 精排层 → 重排层 → 返回
            5000万 → 500条   → 100条  → 20条  → 10条
            < 50ms           < 30ms   < 80ms  < 20ms
                                              总计 < 180ms
```

### 召回层：多路召回策略

**为什么需要多路？** 单路召回只能覆盖一个维度，多路召回保证候选集的多样性。

| 召回通道 | 策略 | 候选数 | 延迟 | 目的 |
|---------|------|--------|------|------|
| 协同过滤 | 看过同样视频的人还看了什么 | 200 | 10ms | 精准推荐 |
| 向量召回 | 用户向量和视频向量最近邻 | 200 | 20ms | 兴趣扩展 |
| 标签召回 | 按兴趣标签匹配 | 100 | 5ms | 兜底保障 |
| 热门召回 | 全站热门视频 | 50 | 3ms | 新用户兜底 |
| 关注召回 | 关注的创作者新视频 | 50 | 5ms | 社交关系 |
| 探索召回 | 随机选取冷门视频 | 20 | 3ms | 打破茧房 |

**1. 协同过滤召回（完整实现）**

协同过滤的核心思想是"相似用户喜欢相似内容"。实现时需要离线计算关联索引，在线查询。

```python
class CollaborativeFilterRecaller:
    """基于 ItemCF 的协同过滤召回——'看了这个视频的人还看了'"""

    def __init__(self, redis_client, mysql_client, config):
        self.redis = redis_client
        self.mysql = mysql_client
        self.recall_count = config.get('recall_count', 200)
        self.similar_item_limit = config.get('similar_item_limit', 50)

    def recall(self, user_id, count=200):
        # 1. 获取用户最近互动的视频（优先取正反馈：完播/点赞/分享）
        recent_videos = self._get_recent_positive_interactions(user_id, limit=50)

        if not recent_videos:
            # 冷启动用户：退回到热门召回
            return self._get_hot_videos(count)

        # 2. 对每个互动视频，查找其关联视频（ItemCF 索引）
        candidates = []
        for video_id in recent_videos:
            similar = self._get_similar_items(video_id, limit=self.similar_item_limit)
            for item_id, score in similar:
                if item_id not in set(recent_videos):  # 排除已看过的
                    candidates.append((item_id, score))

        # 3. 合并去重，加权求分（互动类型加权：分享>点赞>完播>点击）
        merged = {}
        for item_id, score in candidates:
            weight = self._get_interaction_weight(user_id, item_id)
            merged[item_id] = merged.get(item_id, 0) + score * weight

        # 4. 按综合分排序取 Top N
        ranked = sorted(merged.items(), key=lambda x: -x[1])
        return [item_id for item_id, _ in ranked[:count]]

    def _get_recent_positive_interactions(self, user_id, limit=50):
        """从 Redis 获取用户近期的正反馈视频列表"""
        # Redis Sorted Set：score 为互动时间戳，member 为视频ID
        interactions = self.redis.zrevrange(
            f"user:interactions:{user_id}", 0, limit - 1, withscores=True
        )
        return [int(vid) for vid, _ in interactions]

    def _get_similar_items(self, video_id, limit=50):
        """从 Redis/MySQL 获取 ItemCF 关联视频"""
        # 优先从 Redis 缓存读取
        cache_key = f"itemcf:similar:{video_id}"
        cached = self.redis.zrevrange(cache_key, 0, limit - 1, withscores=True)
        if cached:
            return [(int(vid), float(score)) for vid, score in cached]

        # 缓存未命中，从 MySQL recall_index 表查询
        rows = self.mysql.query(
            "SELECT target_video_id, score FROM recall_index "
            "WHERE source_video_id = %s AND recall_type = 1 "
            "ORDER BY score DESC LIMIT %s",
            (video_id, limit)
        )
        result = [(row['target_video_id'], row['score']) for row in rows]

        # 回填缓存，TTL 24小时
        if result:
            pipe = self.redis.pipeline()
            for item_id, score in result:
                pipe.zadd(cache_key, {str(item_id): score})
            pipe.expire(cache_key, 86400)
            pipe.execute()

        return result

    def _get_interaction_weight(self, user_id, video_id):
        """不同互动类型的权重"""
        # 检查 Redis Hash 中该用户对该视频的互动类型
        action = self.redis.hget(f"user:action:{user_id}", str(video_id))
        weight_map = {'share': 4.0, 'like': 3.0, 'complete': 2.0, 'click': 1.0}
        return weight_map.get(action, 1.0)

    def _get_hot_videos(self, count):
        """兜底：返回热门视频"""
        return [int(vid) for vid in self.redis.zrevrange("global:hot_videos", 0, count - 1)]
```

**ItemCF 离线计算（Spark 作业）：**

```python
# 每日离线计算 ItemCF 相似度矩阵，结果写入 recall_index 表
def compute_itemcf_similarity():
    """
    基于 Spark 的 ItemCF 离线计算流程
    输入：用户-视频互动日志
    输出：视频-视频相似度，写入 recall_index 表
    """
    # 1. 读取近30天的互动日志
    interactions = spark.read.parquet("/data/recommendation_log/")
        .filter("created_at >= date_sub(current_date(), 30)")
        .filter("is_clicked = 1")  # 只取有点击的记录

    # 2. 计算每个视频的互动用户集合（用于相似度计算）
    video_users = interactions.groupBy("video_id").agg(
        collect_set("user_id").alias("user_set")
    )

    # 3. 自连接计算视频对共现用户数
    pairs = video_users.alias("a").join(
        video_users.alias("b"),
        col("a.video_id") < col("b.video_id")  # 避免重复计算
    )

    # 4. 计算 Jaccard 相似度
    similarity = pairs.withColumn(
        "intersection_size",
        size(intersect(col("a.user_set"), col("b.user_set")))
    ).withColumn(
        "union_size",
        size(union(col("a.user_set"), col("b.user_set")))
    ).withColumn(
        "score",
        col("intersection_size") / col("union_size")  # Jaccard 系数
    ).filter("score > 0.01")  # 过滤极弱关联

    # 5. 每个视频取 Top 200 相似视频
    top_similar = Window.partitionBy("a.video_id").orderBy(col("score").desc())
    result = similarity.withColumn("rank", row_number().over(top_similar)) \
        .filter("rank <= 200") \
        .select(
            col("a.video_id").alias("source_video_id"),
            col("b.video_id").alias("target_video_id"),
            col("score"),
            lit(1).alias("recall_type")  # 1 = 协同过滤
        )

    # 6. 写入 recall_index 表（先清空旧数据再写入）
    spark.sql("DELETE FROM recall_index WHERE recall_type = 1")
    result.write.mode("append").jdbc(MYSQL_URL, "recall_index")
```

**2. 向量召回（双塔模型 + ANN 索引完整实现）**

```
离线训练双塔模型：
  用户塔：用户特征 → 用户向量（128维）
  视频塔：视频特征 → 视频向量（128维）

在线召回：
  1. 计算当前用户向量
  2. 在向量索引（FAISS/HNSW）中搜索最近邻视频
  3. 返回 Top 200
```

```python
import faiss
import numpy as np

class VectorRecaller:
    """基于双塔模型的向量召回，使用 FAISS IVF-PQ 索引"""

    def __init__(self, config):
        self.redis = config['redis']
        self.feature_store = config['feature_store']
        self.user_tower = config['user_tower_model']  # 预加载的用户塔模型
        self.faiss_index = None
        self.video_id_map = []  # FAISS 索引位置 → video_id 的映射
        self._load_index()

    def _load_index(self):
        """加载 FAISS 索引（从共享存储或 NFS 读取）"""
        # IVF-PQ 索引参数：nlist=4096, m=32（将128维切成32个子空间）
        index_path = "/data/faiss_index/video_index.ivfpq"
        map_path = "/data/faiss_index/video_id_map.npy"

        self.faiss_index = faiss.read_index(index_path)
        self.video_id_map = np.load(map_path)
        self.faiss_index.nprobe = 32  # 搜索时探查的聚类数，精度与速度的权衡

    def recall(self, user_id, count=200):
        # 1. 获取用户向量（预计算并缓存，TTL=1小时）
        user_vector = self._get_user_vector(user_id)

        # 2. 向量近邻搜索（FAISS IVF-PQ）
        user_vector_2d = user_vector.reshape(1, -1).astype('float32')
        distances, indices = self.faiss_index.search(user_vector_2d, count)

        # 3. 将索引位置映射回 video_id
        video_ids = []
        for idx in indices[0]:
            if idx >= 0:  # FAISS 返回 -1 表示不足 count 个结果
                video_ids.append(int(self.video_id_map[idx]))

        return video_ids

    def _get_user_vector(self, user_id):
        """获取用户向量，优先缓存，缓存未命中则实时计算"""
        cache_key = f"user_vector:{user_id}"
        cached = self.redis.get(cache_key)
        if cached:
            return np.frombuffer(cached, dtype=np.float32)

        # 缓存未命中，通过用户塔模型计算
        user_features = self.feature_store.get_user_features(user_id)
        user_vector = self.user_tower.predict(user_features)

        # 写入缓存，TTL=1小时
        self.redis.setex(cache_key, 3600, user_vector.tobytes())
        return user_vector


class FAISSIndexBuilder:
    """FAISS 索引的离线构建与每日全量重建"""

    def build_index(self, video_vectors, video_ids):
        """
        构建_IVF-PQ_索引
        video_vectors: numpy array, shape=(N, 128), dtype=float32
        video_ids: numpy array, shape=(N,)
        """
        n_vectors, dim = video_vectors.shape
        nlist = 4096    # 聚类中心数
        m = 32          # PQ 子量化器数（128/32=4维/子空间）
        n_bits = 8      # 每个子量化器的比特数

        # 1. 训练量化器
        quantizer = faiss.IndexFlatL2(dim)
        index = faiss.IndexIVFPQ(quantizer, dim, nlist, m, n_bits)

        # 2. 训练聚类中心（需要足够多的样本）
        print(f"Training FAISS index with {n_vectors} vectors...")
        index.train(video_vectors)

        # 3. 添加向量
        index.add(video_vectors)
        print(f"Index built: {index.ntotal} vectors, size={faiss.serialize_index(index).nbytes/1e9:.2f}GB")

        # 4. 保存索引和 ID 映射
        faiss.write_index(index, "/data/faiss_index/video_index.ivfpq")
        np.save("/data/faiss_index/video_id_map.npy", video_ids)

        return index

    def incremental_update(self, new_video_vectors, new_video_ids):
        """
        增量更新：白天每小时将新视频向量追加到索引
        注意：追加后索引精度略降，仍依赖夜间全量重建
        """
        index = faiss.read_index("/data/faiss_index/video_index.ivfpq")
        id_map = np.load("/data/faiss_index/video_id_map.npy")

        # 追加新向量（IVF 索引支持动态添加）
        index.add(new_video_vectors.astype('float32'))
        new_id_map = np.concatenate([id_map, new_video_ids])
        np.save("/data/faiss_index/video_id_map.npy", new_id_map)
        faiss.write_index(index, "/data/faiss_index/video_index.ivfpq")
```

**3. 基于内容的召回**

```python
class ContentBasedRecaller:
    """基于视频内容标签的召回：按用户兴趣标签匹配同类视频"""

    def __init__(self, config):
        self.redis = config['redis']
        self.mysql = config['mysql']
        self.tag_weight_decay = 0.95  # 标签权重随时间衰减

    def recall(self, user_id, count=100):
        # 1. 获取用户兴趣标签及权重
        interest_tags = self._get_user_interest_tags(user_id)

        if not interest_tags:
            return self._get_category_diverse_videos(count)

        # 2. 按权重排序，优先召回高兴趣标签的视频
        sorted_tags = sorted(interest_tags.items(), key=lambda x: -x[1])
        candidates = []
        remaining = count

        for tag, weight in sorted_tags:
            if remaining <= 0:
                break
            # 每个标签按权重比例分配召回名额
            tag_count = max(5, int(count * weight / sum(interest_tags.values())))
            tag_count = min(tag_count, remaining)

            tag_videos = self._get_videos_by_tag(tag, limit=tag_count * 2)
            # 过滤已看过的
            watched = self._get_watched_set(user_id)
            tag_videos = [v for v in tag_videos if v not in watched][:tag_count]
            candidates.extend(tag_videos)
            remaining -= len(tag_videos)

        return candidates

    def _get_user_interest_tags(self, user_id):
        """从 Redis Hash 获取用户兴趣标签权重"""
        raw = self.redis.hgetall(f"user:interest:{user_id}")
        if not raw:
            return {}
        # 解码并转换为 float 权重
        return {k.decode(): float(v) for k, v in raw.items()}

    def _get_videos_by_tag(self, tag, limit=200):
        """按标签从倒排索引获取视频"""
        # Redis Sorted Set：score 为视频热度，member 为视频ID
        video_ids = self.redis.zrevrange(f"tag:videos:{tag}", 0, limit - 1)
        return [int(vid) for vid in video_ids]

    def _get_watched_set(self, user_id):
        """获取用户最近看过的视频集合（用于去重）"""
        return set(int(vid) for vid in self.redis.lrange(f"user:recent:{user_id}", 0, -1))

    def _get_category_diverse_videos(self, count):
        """兜底：返回各分类的热门视频，保证多样性"""
        categories = self.redis.smembers("global:categories")
        per_category = max(1, count // len(categories))
        result = []
        for cat in categories:
            vids = self.redis.zrevrange(f"category:hot:{cat.decode()}", 0, per_category - 1)
            result.extend(int(v) for v in vids)
        return result[:count]
```

**4. 多路召回合并与去重**

```python
class MultiChannelRecaller:
    """多路召回调度器：并行执行多路召回，合并去重后输出候选集"""

    CHANNEL_CONFIGS = [
        {'name': 'cf',      'class': CollaborativeFilterRecaller, 'count': 200, 'weight': 1.0},
        {'name': 'vector',  'class': VectorRecaller,              'count': 200, 'weight': 0.9},
        {'name': 'content', 'class': ContentBasedRecaller,         'count': 100, 'weight': 0.7},
        {'name': 'hot',     'class': HotRecaller,                  'count': 50,  'weight': 0.5},
        {'name': 'follow',  'class': FollowRecaller,               'count': 50,  'weight': 1.0},
        {'name': 'explore', 'class': ExplorationRecaller,          'count': 20,  'weight': 0.3},
    ]

    def recall(self, user_id, total_count=500):
        # 1. 并行执行各路召回
        from concurrent.futures import ThreadPoolExecutor, as_completed
        channel_results = {}

        with ThreadPoolExecutor(max_workers=len(self.CHANNEL_CONFIGS)) as executor:
            futures = {}
            for config in self.CHANNEL_CONFIGS:
                recaller = self._get_recaller(config)
                future = executor.submit(recaller.recall, user_id, config['count'])
                futures[future] = config['name']

            for future in as_completed(futures):
                channel_name = futures[future]
                try:
                    channel_results[channel_name] = future.result(timeout=0.05)  # 50ms超时
                except Exception as e:
                    # 单路召回失败不影响整体，记录日志后继续
                    channel_results[channel_name] = []
                    logging.warning(f"Recall channel {channel_name} failed: {e}")

        # 2. 合并去重，保留通道来源和分数
        merged = {}  # video_id -> {'channels': [], 'weighted_score': float}
        for config in self.CHANNEL_CONFIGS:
            channel_name = config['name']
            weight = config['weight']
            for rank, video_id in enumerate(channel_results.get(channel_name, [])):
                if video_id not in merged:
                    merged[video_id] = {'channels': [], 'weighted_score': 0}
                # 分数 = 通道权重 × 排名衰减
                rank_score = weight * (1.0 / (rank + 1))
                merged[video_id]['channels'].append(channel_name)
                merged[video_id]['weighted_score'] += rank_score

        # 3. 按加权分排序取 Top N
        ranked = sorted(merged.items(), key=lambda x: -x[1]['weighted_score'])
        return [video_id for video_id, _ in ranked[:total_count]]
```

**FAISS 索引的维护：**
- 5000 万视频向量，128 维，约 25GB
- 使用 IVF-PQ 索引：搜索精度约 90%，搜索延迟 < 20ms
- 每日全量重建索引（新视频入库后），白天每小时增量追加

**3. 探索召回（打破信息茧房）**

```python
class ExplorationRecaller:
    """随机选取冷门视频，保证新视频有曝光机会"""
    
    def recall(self, user_id, count=20):
        # ε-贪心策略：10% 的推荐位给随机视频
        candidates = []
        
        # 50% 从新视频池（24小时内上传）中随机选
        new_videos = self.get_new_videos(last_hours=24)
        candidates.extend(random.sample(new_videos, min(count // 2, len(new_videos))))
        
        # 50% 从低曝光视频池中随机选
        low_exposure = self.get_low_exposure_videos(max_views=100)
        candidates.extend(random.sample(low_exposure, min(count // 2, len(low_exposure))))

        return candidates
```

**5. 热门召回（多时间窗口融合）**

热门召回作为新用户的兜底策略，同时也为活跃用户提供"大家都在看"的社交信号。关键在于多时间窗口融合——1小时热榜捕捉突发热点，24小时热榜捕捉稳定热门，72小时热榜提供长尾热门。

```python
import time

class HotRecaller:
    """热门召回：多时间窗口热度融合，作为新用户兜底和多样性补充"""

    def __init__(self, config):
        self.redis = config['redis']
        self.mysql = config['mysql']
        # 多时间窗口：1h 捕捉突发热点，6h 捕捉短期热门，24h/72h 稳定热门
        self.time_windows = [
            {'hours': 1,  'weight': 2.0, 'key': 'hot:videos:1h'},
            {'hours': 6,  'weight': 1.5, 'key': 'hot:videos:6h'},
            {'hours': 24, 'weight': 1.0, 'key': 'hot:videos:24h'},
            {'hours': 72, 'weight': 0.5, 'key': 'hot:videos:72h'},
        ]

    def recall(self, user_id, count=50):
        # 1. 获取用户已看过的视频（去重用）
        watched = self._get_watched_set(user_id)

        # 2. 多时间窗口热门视频融合
        candidate_map = {}  # video_id -> weighted_score
        for window in self.time_windows:
            videos = self.redis.zrevrange(window['key'], 0, count * 2 - 1, withscores=True)
            for vid, raw_score in videos:
                vid = int(vid)
                if vid in watched:
                    continue
                # 时间衰减 + 原始热度分
                weighted_score = raw_score * window['weight']
                candidate_map[vid] = candidate_map.get(vid, 0) + weighted_score

        # 3. 分类多样性控制：每个一级分类最多占 30%
        category_count = {}
        result = []
        for vid, score in sorted(candidate_map.items(), key=lambda x: -x[1]):
            cat = self._get_video_category(vid)
            if category_count.get(cat, 0) >= int(count * 0.3):
                continue
            result.append(vid)
            category_count[cat] = category_count.get(cat, 0) + 1
            if len(result) >= count:
                break

        return result

    def _get_watched_set(self, user_id):
        return set(int(vid) for vid in self.redis.lrange(f"user:recent:{user_id}", 0, -1))

    def _get_video_category(self, video_id):
        """从缓存获取视频分类"""
        cat = self.redis.hget(f"video:info:{video_id}", "category_id")
        return int(cat) if cat else 0

    def refresh_hot_lists(self):
        """定时任务：每小时从 MySQL 聚合数据刷新 Redis 热门榜"""
        for window in self.time_windows:
            hours = window['hours']
            cutoff = f"DATE_SUB(NOW(), INTERVAL {hours} HOUR)"
            rows = self.mysql.query(f"""
                SELECT video_id,
                       SUM(play_count) * 1.0 + SUM(like_count) * 3.0 +
                       SUM(share_count) * 5.0 + SUM(comment_count) * 2.0 AS hot_score
                FROM video_hourly_stats
                WHERE stat_hour >= {cutoff}
                GROUP BY video_id
                ORDER BY hot_score DESC
                LIMIT 500
            """)
            pipe = self.redis.pipeline()
            pipe.delete(window['key'])
            for row in rows:
                pipe.zadd(window['key'], {str(row['video_id']): float(row['hot_score'])})
            pipe.expire(window['key'], hours * 3600 + 1800)
            pipe.execute()
```

**6. 关注召回（社交关系链）**

关注召回利用用户的社交关系，推荐其关注的创作者新发布的视频。这类视频的互动率通常比冷召回高出 3-5 倍，因为存在社交信任。

```python
class FollowRecaller:
    """关注召回：用户关注的创作者近期发布的新视频"""

    def __init__(self, config):
        self.redis = config['redis']
        self.mysql = config['mysql']
        self.recent_hours = 72  # 最近 72 小时内的新视频

    def recall(self, user_id, count=50):
        # 1. 获取用户关注的创作者列表
        following = self._get_following_list(user_id)
        if not following:
            return []

        # 2. 获取这些创作者近期发布的视频
        candidates = []
        cutoff_time = int(time.time()) - self.recent_hours * 3600
        for author_id in following:
            # Redis Sorted Set: score 为发布时间戳
            videos = self.redis.zrevrangebyscore(
                f"author:videos:{author_id}", '+inf', cutoff_time
            )
            candidates.extend(int(vid) for vid in videos)

        # 3. 按发布时间倒序取 Top N（最新发布的优先）
        candidates.sort(key=lambda vid: self._get_publish_time(vid), reverse=True)
        return candidates[:count]

    def _get_following_list(self, user_id):
        """获取用户关注的创作者列表，优先缓存"""
        cache_key = f"user:following:{user_id}"
        cached = self.redis.smembers(cache_key)
        if cached:
            return [int(uid) for uid in cached]

        rows = self.mysql.query(
            "SELECT target_user_id FROM user_follows WHERE user_id = %s", (user_id,)
        )
        result = [row['target_user_id'] for row in rows]
        if result:
            pipe = self.redis.pipeline()
            for uid in result:
                pipe.sadd(cache_key, uid)
            pipe.expire(cache_key, 3600)
            pipe.execute()
        return result

    def _get_publish_time(self, video_id):
        """获取视频发布时间戳"""
        return float(self.redis.zscore("global:videos:by_time", str(video_id)) or 0)
```

**7. 多路召回融合与去重策略（增强版）**

多路召回的核心难题不是并行执行，而是融合。不同通道的分数量纲不同（协同过滤是 Jaccard 系数、向量召回是余弦距离、标签召回是标签权重），直接相加没有意义。需要将各通道分数归一化后，再用加权策略融合。

```python
class EnhancedMultiChannelRecaller:
    """
    增强版多路召回调度器：
    1. 并行执行各路召回（每路独立超时）
    2. 各通道内部分数归一化到 [0, 1]
    3. 多通道加权融合，被多通道命中的视频获得额外加权
    4. 多样性打散，防止同类视频扎堆
    5. 通道降级：某通道超时或失败时自动降级，不影响整体
    """

    CHANNEL_CONFIGS = [
        {'name': 'cf',      'class': 'CollaborativeFilterRecaller', 'count': 200, 'weight': 1.0,  'timeout_ms': 15},
        {'name': 'vector',  'class': 'VectorRecaller',              'count': 200, 'weight': 0.9,  'timeout_ms': 25},
        {'name': 'content', 'class': 'ContentBasedRecaller',         'count': 100, 'weight': 0.7,  'timeout_ms': 10},
        {'name': 'hot',     'class': 'HotRecaller',                  'count': 50,  'weight': 0.5,  'timeout_ms': 5},
        {'name': 'follow',  'class': 'FollowRecaller',               'count': 50,  'weight': 1.0,  'timeout_ms': 8},
        {'name': 'explore', 'class': 'ExplorationRecaller',          'count': 20,  'weight': 0.3,  'timeout_ms': 5},
    ]

    def __init__(self, config):
        self.recallers = self._init_recallers(config)
        self.redis = config['redis']
        # 通道健康状态：连续失败计数，用于熔断
        self.channel_health = {c['name']: {'fail_count': 0, 'last_fail_time': 0}
                               for c in self.CHANNEL_CONFIGS}
        self.fuse_threshold = 3       # 连续失败 3 次触发熔断
        self.fuse_recovery_s = 60     # 熔断恢复时间 60 秒

    def recall(self, user_id, total_count=500):
        # 1. 并行执行各路召回（带超时和熔断）
        from concurrent.futures import ThreadPoolExecutor, as_completed
        channel_results = {}

        with ThreadPoolExecutor(max_workers=len(self.CHANNEL_CONFIGS)) as executor:
            futures = {}
            for config in self.CHANNEL_CONFIGS:
                if self._is_fused(config['name']):
                    channel_results[config['name']] = []
                    continue
                recaller = self.recallers[config['name']]
                future = executor.submit(recaller.recall, user_id, config['count'])
                futures[future] = config

            for future in as_completed(futures):
                config = futures[future]
                try:
                    result = future.result(timeout=config['timeout_ms'] / 1000.0)
                    channel_results[config['name']] = result
                    self._record_success(config['name'])
                except Exception as e:
                    channel_results[config['name']] = []
                    self._record_failure(config['name'])
                    logging.warning(f"Recall channel {config['name']} failed: {e}")

        # 2. 各通道内部分数归一化 + 加权融合
        merged = {}  # video_id -> {'channels': [], 'normalized_scores': {}, 'fusion_score': float}
        for config in self.CHANNEL_CONFIGS:
            channel_name = config['name']
            weight = config['weight']
            results = channel_results.get(channel_name, [])
            if not results:
                continue

            # 通道内归一化：排名位置归一化，rank=1 得分 1.0，rank=N 得分接近 0
            n = len(results)
            for rank, video_id in enumerate(results):
                if video_id not in merged:
                    merged[video_id] = {'channels': [], 'normalized_scores': {}, 'fusion_score': 0}
                # 归一化分数 = (N - rank) / N，避免除零
                norm_score = (n - rank) / max(n, 1)
                merged[video_id]['channels'].append(channel_name)
                merged[video_id]['normalized_scores'][channel_name] = norm_score

        # 3. 融合打分：加权求和 + 多通道命中加权
        MULTI_HIT_BONUS = 0.15  # 每多一个通道命中，额外加 15%
        for video_id, info in merged.items():
            base_score = 0
            for channel_name, norm_score in info['normalized_scores'].items():
                weight = next(c['weight'] for c in self.CHANNEL_CONFIGS if c['name'] == channel_name)
                base_score += norm_score * weight
            # 多通道命中加权：被 3 个通道同时命中的视频质量通常更高
            hit_count = len(info['channels'])
            bonus = MULTI_HIT_BONUS * (hit_count - 1) if hit_count > 1 else 0
            info['fusion_score'] = base_score + bonus

        # 4. 按融合分排序
        ranked = sorted(merged.items(), key=lambda x: -x[1]['fusion_score'])

        # 5. 多样性打散：MMR（Maximal Marginal Relevance）策略
        final = self._diversity_rerank(ranked, total_count)

        return final

    def _diversity_rerank(self, ranked_list, count, lambda_param=0.7):
        """
        MMR 多样性打散：
        每次选择分数最高且与已选集合差异最大的视频
        score_mmr = lambda * relevance - (1 - lambda) * max_similarity_to_selected
        """
        if not ranked_list:
            return []

        selected = [ranked_list[0][0]]  # 第一个直接选
        selected_categories = {self._get_category(rankd_list[0][0])}

        remaining = list(ranked_list[1:])
        while len(selected) < count and remaining:
            best_mmr = -float('inf')
            best_idx = -1
            for i, (video_id, info) in enumerate(remaining):
                relevance = info['fusion_score']
                # 与已选集合的最大相似度（用分类代理）
                cat = self._get_category(video_id)
                category_overlap = 1.0 if cat in selected_categories else 0.0
                mmr = lambda_param * relevance - (1 - lambda_param) * category_overlap
                if mmr > best_mmr:
                    best_mmr = mmr
                    best_idx = i

            if best_idx >= 0:
                chosen_vid = remaining.pop(best_idx)[0]
                selected.append(chosen_vid)
                selected_categories.add(self._get_category(chosen_vid))

        return selected

    def _get_category(self, video_id):
        """获取视频分类（用于多样性计算）"""
        cat = self.redis.hget(f"video:info:{video_id}", "category_id")
        return int(cat) if cat else 0

    def _is_fused(self, channel_name):
        """检查通道是否处于熔断状态"""
        health = self.channel_health[channel_name]
        if health['fail_count'] >= self.fuse_threshold:
            # 熔断恢复检查
            if time.time() - health['last_fail_time'] > self.fuse_recovery_s:
                health['fail_count'] = 0  # 半开状态，允许尝试
                return False
            return True
        return False

    def _record_success(self, channel_name):
        self.channel_health[channel_name]['fail_count'] = 0

    def _record_failure(self, channel_name):
        health = self.channel_health[channel_name]
        health['fail_count'] += 1
        health['last_fail_time'] = time.time()
```

**融合策略关键设计决策：**

| 设计点 | 选择 | 理由 |
|--------|------|------|
| 归一化方法 | 排名归一化 | 不同通道的原始分数量纲不同，排名归一化消除了量纲差异 |
| 多通道加权 | 线性加权 + 命中奖励 | 被多通道同时命中的视频质量更可靠，需要额外加分 |
| 多样性策略 | MMR 打散 | 简单的分类限制会损失太多相关性，MMR 在相关性和多样性间取得平衡 |
| 通道熔断 | 连续失败 3 次熔断，60 秒后半开 | 单通道故障不应拖垮整体召回，但需要自动恢复机制 |

### 粗排层：轻量模型快速过滤

粗排的目标是从 500 条候选中筛选出 100 条。粗排模型必须在 30ms 内完成 500 次打分。

**模型选择：双塔模型（与向量召回共享）**

```python
class PreRanker:
    """粗排：双塔模型打分，每条 < 0.05ms"""

    def rank(self, user_id, candidates, count=100):
        # 1. 获取用户向量
        user_vector = self.get_user_vector(user_id)
        
        # 2. 批量获取视频向量
        video_vectors = self.batch_get_video_vectors(candidates)
        
        # 3. 计算内积（用户向量和视频向量的相似度）
        scores = np.dot(video_vectors, user_vector)
        
        # 4. 取 Top N
        top_indices = np.argsort(scores)[-count:][::-1]
        return [(candidates[i], scores[i]) for i in top_indices]
```

**为什么粗排不能用精排模型？**
- 精排模型（DNN，3 层全连接 + 注意力）单次推理约 2ms
- 500 条 × 2ms = 1000ms → 超出 30ms 预算
- 双塔模型计算内积，500 条 × 0.05ms = 25ms → 可接受
- 精度损失：约 5-10%（粗排不会漏掉真正优质的视频）

### 精排层：复杂模型精确打分

```python
class Ranker:
    """精排：深度神经网络，综合多维度特征"""

    def rank(self, user_id, candidates, count=20):
        # 1. 构建特征向量
        features = self.build_features(user_id, candidates)
        
        # 2. 模型推理（GPU 批量推理）
        scores = self.model.predict_batch(features)  # 100条 × 0.8ms = 80ms
        
        # 3. 取 Top N
        ranked = sorted(zip(candidates, scores), key=lambda x: -x[1])
        return ranked[:count]

    def build_features(self, user_id, candidates):
        """构建精排特征向量"""
        user_features = self.feature_store.get_user_features(user_id)
        
        batch_features = []
        for video_id, _ in candidates:
            video_features = self.feature_store.get_video_features(video_id)
            
            # 交叉特征
            cross_features = self.build_cross_features(user_features, video_features)
            
            # 拼接所有特征
            feature_vector = np.concatenate([
                user_features.dense,           # 用户画像（200维）
                video_features.dense,          # 视频属性（100维）
                cross_features,                # 交叉特征（50维）
                user_features.sequence,        # 用户行为序列（最近50个视频，50×32维）
            ])
            
            batch_features.append(feature_vector)
        
        return np.array(batch_features)
    
    def build_cross_features(self, user_features, video_features):
        """交叉特征：用户与视频的互动信号"""
        return np.array([
            user_features.category_affinity.get(video_features.category, 0),
            user_features.author_affinity.get(video_features.author_id, 0),
            user_features.avg_watch_time / max(video_features.duration, 1),
            # ... 更多交叉特征
        ])
```

### 重排层：业务规则 + 多样性

精排的 Top 20 可能是同类视频（如 20 条美食视频），需要重排保证多样性。

```python
class ReRanker:
    """重排：业务规则过滤 + 多样性保证"""

    def rerank(self, ranked_list, count=10):
        result = []
        category_count = {}
        author_count = {}
        ad_inserted = False

        for video, score in ranked_list:
            # 1. 同类别限制：同一类别最多连续 3 条
            cat = video.category
            if len(result) >= 3 and all(v.category == cat for v in result[-3:]):
                continue  # 跳过，避免连续同类

            # 2. 同作者限制：同作者最多 2 条
            if author_count.get(video.author_id, 0) >= 2:
                continue

            # 3. 广告插入：第 4 条位置插入一条广告
            if len(result) == 3 and not ad_inserted:
                ad = self.get_sponsored_video(video.category)
                if ad:
                    result.append(ad)
                    ad_inserted = True

            result.append(video)
            category_count[cat] = category_count.get(cat, 0) + 1
            author_count[video.author_id] = author_count.get(video.author_id, 0) + 1

            if len(result) >= count:
                break

        return result
```

### 实时特征管道（完整实现）

实时特征管道是推荐系统"活"的关键。离线特征只能反映用户昨天的兴趣，而实时特征管道让推荐系统在用户行为发生后数秒内感知变化。

**整体架构：**

```
用户行为事件 → SDK 上报 → Kafka Topic → Flink 实时计算 → 特征存储更新
    ↓                                                         ↓
  埋点采集                                              Redis（实时特征）
  (客户端/服务端)                                        MySQL（落库持久化）
                                                             ↓
                                                    离线训练数据（Hive/ClickHouse）
```

**1. 用户行为事件定义**

```python
from dataclasses import dataclass
from enum import Enum
import time
import json

class ActionType(Enum):
    EXPOSE = 1       # 曝光
    CLICK = 2        # 点击
    PLAY = 3         # 播放
    PAUSE = 4        # 暂停
    COMPLETE = 5     # 完播
    LIKE = 6         # 点赞
    DISLIKE = 7      # 踩
    COMMENT = 8      # 评论
    SHARE = 9        # 分享
    FOLLOW = 10      # 关注作者
    SEARCH = 11      # 搜索
    SWIPE_AWAY = 12  # 滑走（负反馈）

@dataclass
class UserActionEvent:
    """用户行为事件：客户端埋点上报"""
    event_id: str        # 事件唯一 ID
    user_id: int
    video_id: int
    action_type: ActionType
    timestamp: int       # 事件发生时间（毫秒时间戳）
    # 上下文信息
    session_id: str      # 会话 ID
    position: int        # 推荐位（1-10）
    recall_channel: str  # 召回通道
    experiment_id: str   # A/B 实验组
    # 播放相关
    watch_duration: int = 0   # 观看时长（秒）
    video_duration: int = 0   # 视频总时长（秒）
    # 设备信息
    device_type: str = ""     # ios/android/web
    network_type: str = ""    # wifi/4g/5g
    app_version: str = ""

    def to_kafka_message(self) -> str:
        return json.dumps({
            'event_id': self.event_id,
            'user_id': self.user_id,
            'video_id': self.video_id,
            'action_type': self.action_type.value,
            'timestamp': self.timestamp,
            'session_id': self.session_id,
            'position': self.position,
            'recall_channel': self.recall_channel,
            'experiment_id': self.experiment_id,
            'watch_duration': self.watch_duration,
            'video_duration': self.video_duration,
            'device_type': self.device_type,
            'network_type': self.network_type,
            'app_version': self.app_version,
        })
```

**2. Kafka 事件上报服务**

```python
from kafka import KafkaProducer
from kafka.errors import KafkaError
import logging
import uuid

class EventReporter:
    """用户行为事件上报服务：客户端 SDK 调用此服务将事件写入 Kafka"""

    def __init__(self, config):
        self.producer = KafkaProducer(
            bootstrap_servers=config['kafka_brokers'],
            # 事件 key = user_id，保证同一用户的事件有序
            key_serializer=lambda k: str(k).encode('utf-8'),
            value_serializer=lambda v: v.encode('utf-8'),
            # 可靠性配置
            acks='all',                    # 等待所有副本确认
            retries=3,                     # 重试 3 次
            retry_backoff_ms=100,
            max_in_flight_requests_per_connection=1,  # 保证有序
            # 性能配置
            linger_ms=10,                  # 批量发送等待 10ms
            batch_size=16384,              # 批量大小 16KB
            compression_type='lz4',        # LZ4 压缩（低 CPU 开销）
            buffer_memory=33554432,        # 缓冲区 32MB
        )
        self.topic = config.get('topic', 'user_action_events')
        self.fallback_queue = []  # Kafka 不可用时的本地缓冲队列

    def report(self, event: UserActionEvent):
        """上报单个事件"""
        message = event.to_kafka_message()
        try:
            future = self.producer.send(
                self.topic,
                key=event.user_id,
                value=message,
            )
            future.add_callback(self._on_send_success)
            future.add_errback(self._on_send_error)
        except KafkaError as e:
            # Kafka 不可用时写入本地缓冲队列（最多缓冲 10000 条）
            if len(self.fallback_queue) < 10000:
                self.fallback_queue.append(message)
            logging.error(f"Kafka send failed, buffered locally: {e}")

    def report_batch(self, events: list):
        """批量上报事件（服务端聚合后调用）"""
        for event in events:
            self.report(event)
        self.producer.flush(timeout=5)

    def _on_send_success(self, record_metadata):
        pass  # 成功无需处理

    def _on_send_error(self, exc):
        logging.error(f"Kafka send error: {exc}")
```

**3. Flink 实时特征计算作业**

```python
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.connectors import FlinkKafkaConsumer, FlinkKafkaProducer
from pyflink.common import SimpleStringSchema, Time
from pyflink.datastream.window import EventTimeSessionWindows

class RealtimeFeatureJob:
    """
    Flink 实时特征计算作业：
    消费 Kafka 用户行为事件 → 计算实时特征 → 更新 Redis 特征存储
    """

    def __init__(self, config):
        self.config = config
        self.redis_client = self._init_redis(config['redis'])
        self.mysql_client = self._init_mysql(config['mysql'])

    def run(self):
        env = StreamExecutionEnvironment.get_execution_environment()
        env.set_parallelism(64)

        # 1. 消费 Kafka 事件
        kafka_source = FlinkKafkaConsumer(
            topics='user_action_events',
            deserialization_schema=SimpleStringSchema(),
            properties={
                'bootstrap.servers': self.config['kafka_brokers'],
                'group.id': 'realtime-feature-job',
                'auto.offset.reset': 'latest',
            }
        )
        event_stream = env.add_source(kafka_source)

        # 2. 解析事件
        parsed = event_stream.map(self._parse_event)

        # 3. 分支处理：不同类型事件触发不同的特征更新
        # 3a. 行为序列更新（所有互动事件）
        parsed.filter(lambda e: e['action_type'] in [2, 3, 5, 6, 8, 9]) \
              .key_by(lambda e: e['user_id']) \
              .process(self._update_behavior_sequence)

        # 3b. 兴趣权重更新（点击/完播/点赞/分享）
        parsed.filter(lambda e: e['action_type'] in [2, 5, 6, 9]) \
              .key_by(lambda e: e['user_id']) \
              .process(self._update_interest_weights)

        # 3c. 实时统计特征更新（会话窗口，30 分钟超时）
        parsed.key_by(lambda e: e['user_id']) \
              .window(EventTimeSessionWindows.with_gap(Time.minutes(30))) \
              .process(self._update_session_stats)

        # 3d. 视频统计特征更新
        parsed.key_by(lambda e: e['video_id']) \
              .process(self._update_video_stats)

        # 4. 异步落库到 MySQL（不阻塞主流程）
        parsed.add_sink(self._async_persist_to_mysql)

        env.execute("RealtimeFeatureJob")

    def _parse_event(self, raw_json):
        """解析 Kafka 消息为事件字典"""
        import json
        return json.loads(raw_json)

    def _update_behavior_sequence(self, event, ctx):
        """
        更新用户行为序列（Redis List）
        保留最近 50 条正反馈行为，用于精排模型的行为序列特征
        """
        user_id = event['user_id']
        video_id = event['video_id']
        action_type = event['action_type']

        # 互动类型加权：不同行为有不同的序列权重
        weight_map = {
            2: 1.0,   # 点击
            3: 1.5,   # 播放
            5: 2.0,   # 完播
            6: 2.5,   # 点赞
            8: 2.0,   # 评论
            9: 3.0,   # 分享
        }
        weight = weight_map.get(action_type, 1.0)

        # Redis 操作：LPUSH + LTRIM 保证序列长度
        key = f"user:behavior_seq:{user_id}"
        pipe = self.redis_client.pipeline()
        # 存储格式：video_id:action_type:weight:timestamp
        value = f"{video_id}:{action_type}:{weight}:{event['timestamp']}"
        pipe.lpush(key, value)
        pipe.ltrim(key, 0, 49)  # 保留最近 50 条
        pipe.expire(key, 86400 * 7)  # TTL 7 天
        pipe.execute()

        # 输出计数器
        ctx.output('behavior_seq_updated', 1)

    def _update_interest_weights(self, event, ctx):
        """
        更新用户兴趣标签权重（Redis Hash）
        采用指数移动平均（EMA）策略：新行为的权重逐渐增大，旧兴趣逐渐衰减
        """
        user_id = event['user_id']
        video_id = event['video_id']
        action_type = event['action_type']

        # 获取视频所属分类
        category = self.redis_client.hget(f"video:info:{video_id}", "category_id")
        if not category:
            return
        category = category.decode()

        # 获取视频标签
        tags_raw = self.redis_client.hget(f"video:info:{video_id}", "tags")
        tags = json.loads(tags_raw.decode()) if tags_raw else []

        # 行为强度：分享 > 点赞 > 完播 > 点击
        intensity_map = {9: 4.0, 6: 3.0, 5: 2.5, 2: 1.0}
        intensity = intensity_map.get(action_type, 1.0)

        # EMA 衰减系数
        alpha = 0.3  # 新值的权重

        key = f"user:interest:{user_id}"
        pipe = self.redis_client.pipeline()

        # 更新分类兴趣
        current_cat_score = float(self.redis_client.hget(key, f"cat:{category}") or 0)
        new_cat_score = alpha * intensity + (1 - alpha) * current_cat_score
        pipe.hset(key, f"cat:{category}", round(new_cat_score, 4))

        # 更新标签兴趣
        for tag in tags:
            current_tag_score = float(self.redis_client.hget(key, f"tag:{tag}") or 0)
            new_tag_score = alpha * intensity * 0.8 + (1 - alpha) * current_tag_score
            pipe.hset(key, f"tag:{tag}", round(new_tag_score, 4))

        # 更新活跃时间戳
        pipe.hset(key, "last_active_ts", event['timestamp'])
        pipe.expire(key, 86400 * 30)  # TTL 30 天
        pipe.execute()

    def _update_session_stats(self, user_id, events, ctx):
        """
        会话级统计特征：用户一次打开 App 到关闭的完整行为统计
        包括：本次会话观看数、点赞数、平均完播率、浏览深度等
        """
        events_list = list(events)
        if not events_list:
            return

        watch_count = 0
        like_count = 0
        share_count = 0
        total_watch_ratio = 0.0

        for event in events_list:
            at = event['action_type']
            if at == 3:  # PLAY
                watch_count += 1
                if event.get('video_duration', 0) > 0:
                    total_watch_ratio += event['watch_duration'] / event['video_duration']
            elif at == 6:  # LIKE
                like_count += 1
            elif at == 9:  # SHARE
                share_count += 1

        avg_completion = total_watch_ratio / max(watch_count, 1)

        # 更新 Redis 会话统计
        key = f"user:session_stats:{user_id}"
        pipe = self.redis_client.pipeline()
        pipe.hset(key, mapping={
            'session_watch_count': watch_count,
            'session_like_count': like_count,
            'session_share_count': share_count,
            'session_avg_completion': round(avg_completion, 4),
            'session_start_ts': events_list[0]['timestamp'],
            'session_end_ts': events_list[-1]['timestamp'],
        })
        pipe.expire(key, 3600)  # 会话统计 TTL 1 小时
        pipe.execute()

    def _update_video_stats(self, event, ctx):
        """
        更新视频实时统计：播放数、点赞数等
        使用 Redis INBY 实现原子计数，定时落库到 MySQL
        """
        video_id = event['video_id']
        action_type = event['action_type']

        key = f"video:realtime_stats:{video_id}"
        incr_map = {
            2: 'realtime_clicks',
            3: 'realtime_plays',
            6: 'realtime_likes',
            8: 'realtime_comments',
            9: 'realtime_shares',
        }

        if action_type in incr_map:
            field = incr_map[action_type]
            self.redis_client.hincrby(key, field, 1)
            self.redis_client.expire(key, 86400)

        # 更新视频热度分（用于热门召回）
        if action_type in [2, 3, 6, 9]:
            hot_score_incr = {2: 1.0, 3: 1.0, 6: 3.0, 9: 5.0}.get(action_type, 0)
            self.redis_client.zincrby('hot:videos:1h', hot_score_incr, str(video_id))

    def _async_persist_to_mysql(self, event):
        """异步落库：将事件写入 MySQL 推荐日志表，用于离线分析和模型训练"""
        try:
            self.mysql_client.execute("""
                INSERT INTO recommendation_log
                (request_id, user_id, video_id, position, recall_channel,
                 is_clicked, watch_duration, is_liked, is_shared, is_commented,
                 experiment_id, experiment_group, created_at)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, FROM_UNIXTIME(%s/1000))
            """, (
                event.get('event_id', ''),
                event['user_id'],
                event['video_id'],
                event.get('position', 0),
                event.get('recall_channel', ''),
                1 if event['action_type'] == 2 else 0,
                event.get('watch_duration', 0),
                1 if event['action_type'] == 6 else 0,
                1 if event['action_type'] == 9 else 0,
                1 if event['action_type'] == 8 else 0,
                event.get('experiment_id', ''),
                event.get('experiment_group', ''),
                event['timestamp'],
            ))
        except Exception as e:
            logging.error(f"MySQL persist failed: {e}")
```

**4. 特征存储服务（Feature Store）**

```python
class FeatureStore:
    """
    特征存储服务：统一管理实时特征和离线特征的读写
    读路径：优先读 Redis 实时特征，补读离线特征
    写路径：实时特征写 Redis，异步落库 MySQL
    """

    def __init__(self, config):
        self.redis = config['redis']
        self.mysql = config['mysql']

    def get_user_features(self, user_id):
        """获取用户完整特征（实时 + 离线）"""
        features = {}

        # 1. 实时特征（Redis）
        # 1a. 画像特征
        profile = self.redis.hgetall(f"user:profile:{user_id}")
        features.update({k.decode(): v.decode() for k, v in profile.items()})

        # 1b. 兴趣标签权重
        interest = self.redis.hgetall(f"user:interest:{user_id}")
        features['interest_weights'] = {k.decode(): float(v) for k, v in interest.items()}

        # 1c. 行为序列（最近 50 条）
        behavior_seq = self.redis.lrange(f"user:behavior_seq:{user_id}", 0, 49)
        features['behavior_sequence'] = [item.decode() for item in behavior_seq]

        # 1d. 会话统计
        session_stats = self.redis.hgetall(f"user:session_stats:{user_id}")
        features['session_stats'] = {k.decode(): v.decode() for k, v in session_stats.items()}

        # 2. 离线特征（MySQL，带 Redis 缓存）
        cache_key = f"user:offline_features:{user_id}"
        offline = self.redis.get(cache_key)
        if offline:
            features['offline'] = json.loads(offline)
        else:
            row = self.mysql.query(
                "SELECT * FROM user_features WHERE user_id = %s", (user_id,)
            )
            if row:
                features['offline'] = row[0]
                self.redis.setex(cache_key, 3600, json.dumps(row[0], default=str))

        return features

    def get_video_features(self, video_id):
        """获取视频完整特征"""
        features = {}

        # 1. 实时统计（Redis）
        realtime = self.redis.hgetall(f"video:realtime_stats:{video_id}")
        features['realtime_stats'] = {k.decode(): int(v) for k, v in realtime.items()}

        # 2. 基础信息（Redis 缓存）
        info = self.redis.hgetall(f"video:info:{video_id}")
        if info:
            features['info'] = {k.decode(): v.decode() for k, v in info.items()}
        else:
            # 缓存未命中，从 MySQL 加载
            row = self.mysql.query(
                "SELECT * FROM video_features WHERE video_id = %s", (video_id,)
            )
            if row:
                features['info'] = row[0]
                # 回填缓存
                pipe = self.redis.pipeline()
                for k, v in row[0].items():
                    if v is not None and k not in ('visual_embedding', 'text_embedding',
                                                     'audio_embedding', 'fusion_embedding'):
                        pipe.hset(f"video:info:{video_id}", k, str(v))
                pipe.expire(f"video:info:{video_id}", 86400)
                pipe.execute()

        return features
```

**实时性分析：**
- 行为上报 → Kafka → Flink：约 1-3 秒
- 更新 Redis：约 1ms
- 用户下次请求（平均 30 秒后）使用新特征
- **从用户点击到推荐结果变化的延迟：约 30-35 秒**（主要取决于用户下次请求的时间）

**特征管道监控指标：**

| 监控指标 | 目标值 | 告警阈值 |
|---------|--------|---------|
| Kafka 消费延迟 | < 1s | > 10s |
| Flink 事件处理延迟 | < 2s | > 5s |
| Redis 写入延迟 P99 | < 3ms | > 10ms |
| 行为序列更新成功率 | > 99.9% | < 99.5% |
| 兴趣权重更新成功率 | > 99.9% | < 99.5% |

### 新视频冷启动方案（完整实现）

**问题：** 新视频零播放、零互动，推荐系统没有信号。每日 50 万条新视频如果得不到曝光，创作者就会离开平台。

**方案总览：**

```
新视频上传 → 内容理解（提取特征/向量）→ 初始流量池（100次曝光）
              │                              │
              │                              ├─ 数据好 → 二级池 → ... → 全站推荐
              │                              └─ 数据差 → 停止推流
              │
              └─ 内容相似度匹配 → 推荐给相似受众（内容理解辅助冷启动）
```

**1. 流量池赛马机制**

```python
class ColdStartHandler:
    """新视频冷启动：流量池赛马 + 内容理解辅助"""

    POOLS = {
        0: {"name": "初始池",   "impressions": 100,    "next_pool": 1, "promote_threshold": {"completion_rate": 0.25, "like_rate": 0.04}},
        1: {"name": "二级池",   "impressions": 1000,   "next_pool": 2, "promote_threshold": {"completion_rate": 0.28, "like_rate": 0.045}},
        2: {"name": "三级池",   "impressions": 10000,  "next_pool": 3, "promote_threshold": {"completion_rate": 0.30, "like_rate": 0.05}},
        3: {"name": "四级池",   "impressions": 100000, "next_pool": 4, "promote_threshold": {"completion_rate": 0.32, "like_rate": 0.055}},
        4: {"name": "全站推荐", "impressions": -1,     "next_pool": None, "promote_threshold": None},
    }

    def __init__(self, config):
        self.redis = config['redis']
        self.mysql = config['mysql']
        self.content_understanding = ContentUnderstanding(config)

    def on_new_video_upload(self, video_id, video_data):
        """新视频上传时的冷启动初始化"""
        # 1. 提取内容特征（异步，不阻塞上传流程）
        self._async_extract_content_features(video_id, video_data)

        # 2. 分配初始流量池
        self._assign_initial_pool(video_id)

        # 3. 基于内容理解找到潜在受众
        target_users = self.content_understanding.find_target_users_for_new_video(video_id)
        self._seed_initial_audience(video_id, target_users)

    def evaluate(self, video_id, current_pool):
        """评估视频是否可以进入下一级流量池（定时任务每 5 分钟执行一次）"""
        metrics = self._get_video_metrics(video_id)
        pool_config = self.POOLS[current_pool]

        # 检查是否达到当前池的曝光量
        if metrics['impressions'] < pool_config['impressions']:
            return  # 还没曝光够，继续

        # 评估核心指标
        thresholds = pool_config['promote_threshold']
        if thresholds is None:
            return  # 已经是最高级流量池

        if (metrics['completion_rate'] >= thresholds['completion_rate'] and
            metrics['like_rate'] >= thresholds['like_rate']):
            self.promote(video_id, pool_config['next_pool'])
        elif metrics['completion_rate'] < 0.10:
            # 完播率极低，直接停止推流
            self.demote(video_id)
        else:
            # 数据一般，继续在当前池观察（不升不降）
            pass

    def promote(self, video_id, next_pool):
        """晋升到下一级流量池"""
        pool_name = self.POOLS[next_pool]['name']
        logging.info(f"Video {video_id} promoted to {pool_name}")

        # 1. 更新数据库
        self.mysql.execute(
            "UPDATE video_features SET traffic_pool = %s, pool_enter_time = NOW() WHERE video_id = %s",
            (next_pool, video_id)
        )

        # 2. 更新 Redis 缓存
        self.redis.hset(f"video:info:{video_id}", "traffic_pool", next_pool)

        # 3. 更新各流量池的视频集合（用于召回时过滤）
        self.redis.smove(f"pool:videos:{next_pool - 1}", f"pool:videos:{next_pool}", video_id)

    def demote(self, video_id):
        """停止推流"""
        logging.info(f"Video {video_id} demoted, stopping promotion")
        self.mysql.execute(
            "UPDATE video_features SET traffic_pool = -1 WHERE video_id = %s",
            (video_id,)
        )
        self.redis.hset(f"video:info:{video_id}", "traffic_pool", -1)

    def _get_video_metrics(self, video_id):
        """获取视频在当前流量池的表现指标"""
        # 优先从 Redis 实时统计读取
        realtime = self.redis.hgetall(f"video:realtime_stats:{video_id}")
        if realtime:
            plays = int(realtime.get(b'realtime_plays', b'0'))
            likes = int(realtime.get(b'realtime_likes', b'0'))
            clicks = int(realtime.get(b'realtime_clicks', b'0'))
            total_watch = int(realtime.get(b'realtime_total_watch', b'0'))
        else:
            # 降级到 MySQL
            row = self.mysql.query(
                "SELECT play_count, like_count, completion_rate, like_rate FROM video_features WHERE video_id = %s",
                (video_id,)
            )
            if not row:
                return {'impressions': 0, 'completion_rate': 0, 'like_rate': 0}
            plays = row[0]['play_count']
            likes = row[0]['like_count']

        # 计算核心指标
        impressions = max(clicks, 1)
        completion_rate = (total_watch / (plays * self._get_avg_duration(video_id))) if plays > 0 else 0
        like_rate = likes / max(plays, 1)

        return {
            'impressions': impressions,
            'completion_rate': min(completion_rate, 1.0),
            'like_rate': min(like_rate, 1.0),
        }

    def _assign_initial_pool(self, video_id):
        """分配初始流量池"""
        self.mysql.execute(
            "UPDATE video_features SET traffic_pool = 0, pool_enter_time = NOW() WHERE video_id = %s",
            (video_id,)
        )
        self.redis.sadd("pool:videos:0", video_id)
        self.redis.hset(f"video:info:{video_id}", "traffic_pool", 0)

    def _seed_initial_audience(self, video_id, target_users):
        """为新视频种子初始受众——在探索召回中优先推给这些用户"""
        for user_id in target_users[:200]:  # 限制种子用户数
            self.redis.sadd(f"video:seed_audience:{video_id}", user_id)
            self.redis.expire(f"video:seed_audience:{video_id}", 86400)

    def _async_extract_content_features(self, video_id, video_data):
        """异步提取内容特征（不阻塞上传流程）"""
        # 通过消息队列异步处理
        self._publish_to_mq("content_extraction", {
            "video_id": video_id,
            "video_path": video_data['path'],
            "thumbnail": video_data.get('thumbnail'),
            "title": video_data.get('title', ''),
            "description": video_data.get('description', ''),
        })

    def _publish_to_mq(self, topic, message):
        """发送消息到 MQ"""
        import json
        self.redis.lpush(f"mq:{topic}", json.dumps(message))
```

**2. 新用户冷启动方案**

新用户没有行为数据，推荐系统无法做个性化。需要一套从"零特征"到"个性化"的渐进策略。

```python
class NewUserColdStartHandler:
    """
    新用户冷启动策略：
    Phase 0（注册时）：人口统计学特征 + 注册来源初始化
    Phase 1（前 10 次请求）：热门 + 多样性探索，快速收集信号
    Phase 2（10-50 次请求）：轻量个性化，基于少量行为的协同过滤
    Phase 3（50+ 次请求）：完全个性化，全量推荐策略
    """

    def __init__(self, config):
        self.redis = config['redis']
        self.mysql = config['mysql']
        self.hot_recaller = HotRecaller(config)
        self.explore_recaller = ExplorationRecaller()

    def on_user_register(self, user_id, register_info):
        """
        用户注册时初始化特征
        register_info: {"gender": 1, "age_group": 3, "city_tier": 2, "source": "invite", ...}
        """
        # 1. 写入 MySQL 用户画像
        self.mysql.execute(
            """INSERT INTO user_features
            (user_id, gender, age_group, city_tier, register_time, last_active_time)
            VALUES (%s, %s, %s, %s, NOW(), NOW())""",
            (user_id, register_info.get('gender', 0),
             register_info.get('age_group', 0), register_info.get('city_tier', 0))
        )

        # 2. 初始化 Redis 特征
        pipe = self.redis.pipeline()
        pipe.hset(f"user:profile:{user_id}", mapping={
            'gender': register_info.get('gender', 0),
            'age_group': register_info.get('age_group', 0),
            'city_tier': register_info.get('city_tier', 0),
            'is_new_user': 1,
            'request_count': 0,
        })

        # 3. 基于人口统计学特征分配初始兴趣（预计算的人群偏好）
        initial_interests = self._get_demographic_interests(register_info)
        for tag, weight in initial_interests.items():
            pipe.hset(f"user:interest:{user_id}", f"tag:{tag}", weight)

        # 4. 注册来源辅助（如从某活动页注册，说明对该活动相关内容感兴趣）
        if register_info.get('source') == 'invite':
            inviter_interests = self.redis.hgetall(f"user:interest:{register_info['inviter_id']}")
            for k, v in inviter_interests.items():
                pipe.hset(f"user:interest:{user_id}", k, float(v) * 0.5)  # 继承邀请人 50% 兴趣

        pipe.execute()

    def recommend_for_new_user(self, user_id, count=10):
        """根据新用户阶段返回推荐结果"""
        request_count = int(self.redis.hget(f"user:profile:{user_id}", "request_count") or 0)

        if request_count < 10:
            # Phase 1：热门 + 多样性探索（5:3:2 比例）
            return self._phase1_recommend(user_id, count, request_count)
        elif request_count < 50:
            # Phase 2：轻量个性化
            return self._phase2_recommend(user_id, count, request_count)
        else:
            # Phase 3：完全个性化（交给正常推荐流程）
            self.redis.hset(f"user:profile:{user_id}", "is_new_user", 0)
            return None  # 返回 None 表示交给正常推荐

    def _phase1_recommend(self, user_id, count, request_count):
        """Phase 1：热门为主，穿插探索内容"""
        hot_count = int(count * 0.5)
        explore_count = int(count * 0.3)
        demographic_count = count - hot_count - explore_count

        # 热门视频（带人口统计学过滤）
        hot_videos = self.hot_recaller.recall(user_id, hot_count)

        # 探索内容（多样性保证）
        explore_videos = self.explore_recaller.recall(user_id, explore_count)

        # 人口统计学偏好（同类人群喜欢的内容）
        demographic_videos = self._get_demographic_videos(user_id, demographic_count)

        # 合并去重
        all_videos = hot_videos + explore_videos + demographic_videos
        seen = set()
        result = []
        for vid in all_videos:
            if vid not in seen:
                seen.add(vid)
                result.append(vid)

        # 更新请求计数
        self.redis.hincrby(f"user:profile:{user_id}", "request_count", 1)

        return result[:count]

    def _phase2_recommend(self, user_id, count, request_count):
        """Phase 2：基于少量行为的轻量个性化"""
        # 已有少量行为，可以使用协同过滤
        from collections import Counter

        # 获取用户最近互动的视频
        behavior_seq = self.redis.lrange(f"user:behavior_seq:{user_id}", 0, -1)
        if not behavior_seq:
            return self._phase1_recommend(user_id, count, request_count)

        # 统计用户互动过的视频分类
        category_counter = Counter()
        for item in behavior_seq:
            parts = item.decode().split(':')
            video_id = int(parts[0])
            weight = float(parts[2])
            cat = self.redis.hget(f"video:info:{video_id}", "category_id")
            if cat:
                category_counter[int(cat)] += weight

        # 按分类权重分配名额
        total_weight = sum(category_counter.values())
        result = []
        for cat, weight in category_counter.most_common(5):
            cat_count = max(2, int(count * weight / total_weight))
            cat_videos = self.redis.zrevrange(f"category:hot:{cat}", 0, cat_count - 1)
            result.extend(int(vid) for vid in cat_videos)

        # 补充热门和探索
        if len(result) < count:
            remaining = count - len(result)
            hot_videos = self.hot_recaller.recall(user_id, remaining)
            result.extend(vid for vid in hot_videos if vid not in set(result))

        self.redis.hincrby(f"user:profile:{user_id}", "request_count", 1)
        return result[:count]

    def _get_demographic_interests(self, register_info):
        """基于人口统计学特征获取初始兴趣（预计算的人群偏好表）"""
        # Redis Hash: key = "demographic:{gender}:{age_group}:{city_tier}"
        demo_key = f"demographic:{register_info.get('gender', 0)}:" \
                   f"{register_info.get('age_group', 0)}:" \
                   f"{register_info.get('city_tier', 0)}"
        raw = self.redis.hgetall(demo_key)
        if raw:
            return {k.decode(): float(v) for k, v in raw.items()}

        # 降级：使用全局热门标签
        global_tags = self.redis.zrevrange("global:popular_tags", 0, 19, withscores=True)
        return {tag.decode(): float(score) for tag, score in global_tags}

    def _get_demographic_videos(self, user_id, count):
        """获取同类人群喜欢的视频"""
        profile = self.redis.hgetall(f"user:profile:{user_id}")
        gender = profile.get(b'gender', b'0').decode()
        age_group = profile.get(b'age_group', b'0').decode()
        demo_key = f"demographic:videos:{gender}:{age_group}"
        videos = self.redis.zrevrange(demo_key, 0, count - 1)
        return [int(vid) for vid in videos]
```

**3. 冷启动效果评估指标：**

| 指标 | 初始值（Phase 0） | Phase 1 结束 | Phase 2 结束 | Phase 3 |
|------|------------------|-------------|-------------|---------|
| 点击率 CTR | 8%（热门兜底） | 12% | 18% | 25%+ |
| 完播率 | 15% | 22% | 28% | 35%+ |
| 新视频曝光率 | 5% | 8% | 12% | 15% |
| 用户留存率（次日） | 30% | 40% | 50% | 60%+ |

### 性能预算分配（详细分析）

**端到端推荐延迟分解：**

| 阶段 | 延迟预算 | 典型耗时 P50 | P99 | 计算 |
|------|---------|-------------|-----|------|
| 召回 | 50ms | 多路并行 18ms + 合并去重 4ms | 35ms | 22ms |
| 粗排 | 30ms | 500 次内积计算 22ms | 28ms | 22ms |
| 精排 | 80ms | 特征拼装 15ms + GPU 批量推理 50ms | 70ms | 65ms |
| 重排 | 20ms | 规则过滤 + 排序 8ms | 15ms | 8ms |
| 网络传输 | 20ms | 10 条视频元数据序列化 5ms + 网络传输 10ms | 18ms | 15ms |
| **总计** | **200ms** | **132ms** | **166ms** | **132ms** |

留有 68ms 余量（P50）和 34ms 余量（P99），应对 GC 停顿、网络抖动等。

**各阶段延迟的深入分析：**

**1. 召回层延迟分解（目标 < 50ms）：**

| 操作 | 延迟 | 说明 |
|------|------|------|
| 协同过滤通道 | 8-12ms | Redis ZREVRANGE 50 个 key × 0.2ms |
| 向量召回通道 | 15-20ms | FAISS IVF-PQ search, nprobe=32 |
| 标签召回通道 | 5-8ms | Redis ZREVRANGE + 去重过滤 |
| 热门召回通道 | 2-5ms | Redis ZREVRANGE, 单 key |
| 关注召回通道 | 5-8ms | Redis ZREVRANGEBYSCORE × N 个关注 |
| 探索召回通道 | 1-3ms | Redis SRANDMEMBER |
| 合并去重 | 3-5ms | 内存操作，O(N) 去重 + 加权排序 |
| **并行最大** | **15-20ms** | **取决于最慢通道（向量召回）** |

**2. 精排层延迟分解（目标 < 80ms）：**

| 操作 | 延迟 | 说明 |
|------|------|------|
| 特征读取（Redis） | 5-8ms | 100 条 × 用户特征 + 视频特征批量读取 |
| 特征拼装（CPU） | 8-12ms | 100 条交叉特征计算 + 向量拼接 |
| GPU 推理 | 35-50ms | batch_size=100, A100, DNN 3 层全连接 + Attention |
| 后处理 | 2-3ms | 分数提取 + 排序 |
| **总计** | **50-73ms** | |

**3. GPU 推理资源分析：**

| 指标 | 数值 | 说明 |
|------|------|------|
| 单次推理延迟 | 0.5-0.8ms | A100, batch_size=1 |
| 批量推理延迟 | 35-50ms | batch_size=100 |
| GPU 利用率 | 60-70% | 峰值 QPS 下 |
| 每秒最大推理次数 | ~2500 次 | A100 单卡 |
| A100 数量 | 20 张 | 峰值 QPS 需求 |
| 单卡 QPS 容量 | ~500 QPS | 含 batch overhead |
| 峰值 QPS | ~10,000 | 1 亿 DAU, 峰值 10:1 |
| **GPU 集群总容量** | **~10,000 QPS** | 20 × 500 |

**4. 特征存储容量分析：**

| 数据类型 | 单条大小 | 总条数 | 总容量 | 存储引擎 |
|---------|---------|--------|--------|---------|
| 用户画像特征 | ~2KB | 1 亿 | 200GB | Redis Cluster |
| 视频属性特征 | ~1KB | 5000 万 | 50GB | Redis Cluster |
| 用户行为序列 | ~0.5KB | 1 亿 | 50GB | Redis List |
| 用户向量缓存 | 512B (128×float32) | 1 亿 | 50GB | Redis Cache |
| 协同过滤索引 | ~0.1KB/条 | 10 亿条 | 100GB | Redis Hash |
| 视频向量索引 | 128×4B=512B | 5000 万 | 25GB (压缩后) | FAISS IVF-PQ |
| 视频基础信息缓存 | ~0.5KB | 5000 万 | 25GB | Redis Hash |
| **Redis 总计** | | | **~500GB** | **6 节点 × 128GB** |

**5. 带宽分析：**

| 数据流 | 每秒数据量 | 说明 |
|--------|----------|------|
| Kafka 事件摄入 | ~50MB/s | 峰值 5 万条/秒 × 1KB/条 |
| Flink → Redis 写入 | ~10MB/s | 特征更新（远小于摄入量，因为聚合后只写增量） |
| Redis → 推荐服务读取 | ~200MB/s | 峰值 1 万 QPS × 20KB/请求 |
| GPU 模型推理输入 | ~50MB/s | 1 万 QPS × 5KB/请求（特征向量） |

### A/B 测试集成（完整实现）

推荐系统的任何变更（新模型、新特征、新召回策略）都必须通过 A/B 测试验证效果后才能全量上线。A/B 测试框架需要解决三个核心问题：(1) 用户如何分桶？(2) 实验如何互斥？(3) 效果如何评估？

**1. 用户分桶与实验分配**

```python
import hashlib
import struct

class ABTestingService:
    """
    A/B 测试服务：
    - 基于用户 ID 的确定性分桶（同一用户始终在同一组）
    - 支持多层实验（互不干扰的实验可以并行运行）
    - 支持实验互斥组（同一互斥组内只能运行一个实验）
    """

    # 实验层定义：不同层的实验互不干扰
    LAYERS = {
        'recall': 0,      # 召回策略实验层
        'ranking': 1,     # 排序模型实验层
        'rerank': 2,      # 重排策略实验层
        'ui': 3,          # UI 展示实验层
    }

    # 互斥组定义：同一互斥组内的实验不能同时运行
    MUTEX_GROUPS = {
        'ranking_model': ['ranking_v2', 'ranking_v3', 'ranking_transformer'],
        'recall_strategy': ['recall_cf_weight', 'recall_vector_weight'],
    }

    def __init__(self, config):
        self.redis = config['redis']
        self.mysql = config['mysql']
        self.total_buckets = 1000  # 1000 个桶，支持 0.1% 粒度的流量分配

    def get_experiment_group(self, user_id, experiment_id):
        """
        获取用户在指定实验中的分组
        返回: (group_name, experiment_config) 或 (None, None) 表示未命中实验
        """
        # 1. 从 Redis 加载实验配置（实时更新）
        experiment_config = self._load_experiment_config(experiment_id)
        if not experiment_config:
            return None, None

        # 2. 检查实验状态
        if experiment_config['status'] != 'running':
            return None, None

        # 3. 计算用户在实验层中的桶号
        layer = experiment_config['layer']
        bucket = self._hash_user_to_bucket(user_id, layer)

        # 4. 检查用户是否在实验流量范围内
        if bucket >= experiment_config['traffic_percent'] * self.total_buckets / 100:
            return None, None  # 不在实验流量内

        # 5. 根据分组规则确定实验组
        group = self._assign_group(bucket, experiment_config)
        return group, experiment_config['groups'][group]

    def _hash_user_to_bucket(self, user_id, layer):
        """
        将用户 ID 确定性映射到 [0, total_buckets) 的桶号
        使用分层哈希：不同层使用不同的盐值，保证层间独立性
        """
        salt = f"ab_layer_{layer}"
        hash_input = f"{salt}:{user_id}".encode('utf-8')
        hash_value = hashlib.md5(hash_input).digest()
        # 取前 4 字节转换为整数，对桶数取模
        int_value = struct.unpack('<I', hash_value[:4])[0]
        return int_value % self.total_buckets

    def _assign_group(self, bucket, experiment_config):
        """
        根据桶号和分组规则确定实验组
        分组规则示例：{"control": 50, "treatment_a": 25, "treatment_b": 25}
        表示 control 占 50% 流量，treatment_a 占 25%，treatment_b 占 25%
        """
        groups = experiment_config['groups']
        traffic_percent = experiment_config['traffic_percent']
        # 将桶号归一化到实验流量范围内
        normalized_bucket = bucket % (traffic_percent * self.total_buckets // 100)

        # 按分组比例分配
        cumulative = 0
        for group_name, group_config in groups.items():
            group_percent = group_config.get('percent', 0)
            cumulative += group_percent * self.total_buckets / 100
            if normalized_bucket < cumulative:
                return group_name

        # 默认返回 control 组
        return 'control'

    def _load_experiment_config(self, experiment_id):
        """从 Redis 加载实验配置（管理员后台修改后实时生效）"""
        cache_key = f"ab:experiment:{experiment_id}"
        cached = self.redis.get(cache_key)
        if cached:
            import json
            return json.loads(cached)

        # Redis 未命中，从 MySQL 加载
        row = self.mysql.query(
            "SELECT * FROM ab_experiments WHERE experiment_id = %s", (experiment_id,)
        )
        if not row:
            return None

        config = row[0]
        # 回填缓存
        self.redis.setex(cache_key, 60, json.dumps(config, default=str))  # TTL 60 秒
        return config
```

**2. A/B 实验配置表**

```sql
-- A/B 实验配置表
CREATE TABLE ab_experiments (
    id                INT AUTO_INCREMENT PRIMARY KEY,
    experiment_id     VARCHAR(64) NOT NULL UNIQUE COMMENT '实验唯一标识',
    experiment_name   VARCHAR(128) NOT NULL COMMENT '实验名称',
    description       TEXT COMMENT '实验描述',
    layer             VARCHAR(32) NOT NULL COMMENT '实验层: recall/ranking/rerank/ui',
    mutex_group       VARCHAR(64) COMMENT '互斥组ID，同组实验不能同时运行',
    traffic_percent   FLOAT NOT NULL DEFAULT 10.0 COMMENT '流量占比(%)',
    groups            JSON NOT NULL COMMENT '分组配置，如{"control":{"percent":50},"treatment":{"percent":50,"model":"v3"}}',
    status            ENUM('draft', 'running', 'paused', 'completed') DEFAULT 'draft',
    start_time        DATETIME COMMENT '实验开始时间',
    end_time          DATETIME COMMENT '实验结束时间',
    created_by        VARCHAR(64) NOT NULL,
    created_at        DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at        DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_status (status),
    INDEX idx_layer (layer, status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- A/B 实验结果表（每小时聚合）
CREATE TABLE ab_experiment_results (
    id                BIGINT AUTO_INCREMENT PRIMARY KEY,
    experiment_id     VARCHAR(64) NOT NULL,
    group_name        VARCHAR(32) NOT NULL,
    stat_hour         DATETIME NOT NULL COMMENT '统计时段',
    -- 样本量
    user_count        INT NOT NULL COMMENT '分组用户数',
    request_count     BIGINT NOT NULL COMMENT '请求数',
    exposure_count    BIGINT NOT NULL COMMENT '曝光数',
    -- 核心指标
    ctr               FLOAT COMMENT '点击率',
    completion_rate   FLOAT COMMENT '完播率',
    avg_watch_time    FLOAT COMMENT '平均观看时长(秒)',
    like_rate         FLOAT COMMENT '点赞率',
    share_rate        FLOAT COMMENT '分享率',
    -- 商业指标
    ad_revenue_per_user FLOAT COMMENT '人均广告收入',
    -- 统计显著性
    p_value_ctr       FLOAT COMMENT 'CTR 的 p-value',
    confidence_ctr    FLOAT COMMENT 'CTR 的置信区间',
    is_significant    TINYINT DEFAULT 0 COMMENT '是否统计显著',
    created_at        DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_exp_group_hour (experiment_id, group_name, stat_hour),
    INDEX idx_experiment (experiment_id, stat_hour)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**3. 指标收集与统计显著性评估**

```python
import numpy as np
from scipy import stats

class ABMetricsCollector:
    """A/B 测试指标收集与统计评估"""

    def __init__(self, config):
        self.redis = config['redis']
        self.mysql = config['mysql']

    def record_impression(self, user_id, experiment_id, group_name, video_id, metrics):
        """
        记录一次推荐展示的用户反馈
        metrics: {"is_clicked": 1, "watch_duration": 15, "is_liked": 0, ...}
        """
        # 实时计数器（Redis HyperLogLog 用于 UV，普通计数器用于 PV）
        hour_key = f"ab:metrics:{experiment_id}:{group_name}:{self._current_hour()}"

        pipe = self.redis.pipeline()
        # UV（去重用户数）
        pipe.pfadd(f"{hour_key}:users", str(user_id))
        # PV（请求数）
        pipe.incr(f"{hour_key}:requests")
        # 曝光数
        pipe.incr(f"{hour_key}:exposures")
        # 点击数
        if metrics.get('is_clicked'):
            pipe.incr(f"{hour_key}:clicks")
        # 观看时长
        if metrics.get('watch_duration', 0) > 0:
            pipe.incrbyfloat(f"{hour_key}:total_watch", metrics['watch_duration'])
            pipe.incr(f"{hour_key}:watch_count")
        # 点赞/分享
        if metrics.get('is_liked'):
            pipe.incr(f"{hour_key}:likes")
        if metrics.get('is_shared'):
            pipe.incr(f"{hour_key}:shares")
        # TTL 2 小时（保证足够时间聚合）
        pipe.expire(f"{hour_key}:users", 7200)
        pipe.execute()

    def aggregate_hourly_metrics(self, experiment_id):
        """每小时聚合指标，写入 MySQL（定时任务执行）"""
        groups = self._get_experiment_groups(experiment_id)
        stat_hour = self._previous_hour()

        for group_name in groups:
            hour_key = f"ab:metrics:{experiment_id}:{group_name}:{stat_hour.strftime('%Y%m%d%H')}"
            result = {
                'experiment_id': experiment_id,
                'group_name': group_name,
                'stat_hour': stat_hour,
            }

            # 读取 Redis 聚合数据
            result['user_count'] = int(self.redis.pfcount(f"{hour_key}:users") or 0)
            result['request_count'] = int(self.redis.get(f"{hour_key}:requests") or 0)
            result['exposure_count'] = int(self.redis.get(f"{hour_key}:exposures") or 0)

            clicks = int(self.redis.get(f"{hour_key}:clicks") or 0)
            total_watch = float(self.redis.get(f"{hour_key}:total_watch") or 0)
            watch_count = int(self.redis.get(f"{hour_key}:watch_count") or 0)
            likes = int(self.redis.get(f"{hour_key}:likes") or 0)
            shares = int(self.redis.get(f"{hour_key}:shares") or 0)

            # 计算核心指标
            result['ctr'] = clicks / max(result['exposure_count'], 1)
            result['completion_rate'] = watch_count / max(result['exposure_count'], 1)
            result['avg_watch_time'] = total_watch / max(watch_count, 1)
            result['like_rate'] = likes / max(result['exposure_count'], 1)
            result['share_rate'] = shares / max(result['exposure_count'], 1)

            # 写入 MySQL
            self.mysql.execute("""
                INSERT INTO ab_experiment_results
                (experiment_id, group_name, stat_hour, user_count, request_count,
                 exposure_count, ctr, completion_rate, avg_watch_time, like_rate, share_rate)
                VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON DUPLICATE KEY UPDATE
                user_count=VALUES(user_count), ctr=VALUES(ctr),
                completion_rate=VALUES(completion_rate), avg_watch_time=VALUES(avg_watch_time)
            """, (result['experiment_id'], result['group_name'], result['stat_hour'],
                  result['user_count'], result['request_count'], result['exposure_count'],
                  result['ctr'], result['completion_rate'], result['avg_watch_time'],
                  result['like_rate'], result['share_rate']))

    def evaluate_significance(self, experiment_id):
        """
        评估实验结果的统计显著性
        使用双比例 Z 检验（CTR）和双样本 t 检验（观看时长）
        """
        # 从 MySQL 读取聚合数据
        rows = self.mysql.query("""
            SELECT group_name, SUM(user_count) as users, SUM(exposure_count) as exposures,
                   SUM(clicks) as clicks, SUM(total_watch) as total_watch,
                   SUM(watch_count) as watch_count
            FROM ab_experiment_results
            WHERE experiment_id = %s
            GROUP BY group_name
        """, (experiment_id,))

        if len(rows) < 2:
            return {"status": "insufficient_data", "message": "至少需要 2 个分组"}

        control = next((r for r in rows if r['group_name'] == 'control'), None)
        treatment = next((r for r in rows if r['group_name'] != 'control'), None)

        if not control or not treatment:
            return {"status": "error", "message": "需要 control 和 treatment 分组"}

        result = {"experiment_id": experiment_id}

        # 1. CTR 的双比例 Z 检验
        ctrl_ctr = control['clicks'] / max(control['exposures'], 1)
        treat_ctr = treatment['clicks'] / max(treatment['exposures'], 1)
        pooled_ctr = (control['clicks'] + treatment['clicks']) / \
                     max(control['exposures'] + treatment['exposures'], 1)

        se = np.sqrt(pooled_ctr * (1 - pooled_ctr) *
                     (1/max(control['exposures'], 1) + 1/max(treatment['exposures'], 1)))
        z_score = (treat_ctr - ctrl_ctr) / max(se, 1e-10)
        p_value = 2 * (1 - stats.norm.cdf(abs(z_score)))

        result['ctr'] = {
            'control': round(ctrl_ctr, 6),
            'treatment': round(treat_ctr, 6),
            'lift': round((treat_ctr - ctrl_ctr) / max(ctrl_ctr, 1e-10) * 100, 2),  # 提升百分比
            'z_score': round(z_score, 4),
            'p_value': round(p_value, 6),
            'is_significant': p_value < 0.05,
        }

        # 2. 平均观看时长的双样本 t 检验（使用聚合统计量近似）
        ctrl_avg = control['total_watch'] / max(control['watch_count'], 1)
        treat_avg = treatment['total_watch'] / max(treatment['watch_count'], 1)
        # 近似标准差（基于历史数据的经验值：std ≈ 1.5 × mean）
        ctrl_std = 1.5 * ctrl_avg
        treat_std = 1.5 * treat_avg
        t_stat, p_value_watch = stats.ttest_ind_from_stats(
            ctrl_avg, ctrl_std, control['watch_count'],
            treat_avg, treat_std, treatment['watch_count']
        )

        result['watch_time'] = {
            'control': round(ctrl_avg, 2),
            'treatment': round(treat_avg, 2),
            'lift': round((treat_avg - ctrl_avg) / max(ctrl_avg, 1e-10) * 100, 2),
            'p_value': round(p_value_watch, 6),
            'is_significant': p_value_watch < 0.05,
        }

        # 3. 综合判断
        result['overall'] = {
            'recommend_decision': self._make_decision(result),
            'min_sample_size': self._calculate_min_sample_size(ctrl_ctr, 0.01),
            'current_sample': min(control['exposures'], treatment['exposures']),
        }

        return result

    def _make_decision(self, result):
        """
        基于实验结果做决策
        规则：
        - CTR 显著提升 > 1% 且 p < 0.05 → 全量上线
        - CTR 显著下降 > 1% 且 p < 0.05 → 停止实验
        - 不显著 → 继续运行或扩大流量
        """
        ctr_result = result['ctr']
        if ctr_result['is_significant']:
            if ctr_result['lift'] > 1.0:
                return 'launch'  # 全量上线
            elif ctr_result['lift'] < -1.0:
                return 'rollback'  # 回滚
        return 'continue'  # 继续观察

    def _calculate_min_sample_size(self, baseline_ctr, min_detectable_effect, alpha=0.05, power=0.8):
        """计算最小样本量"""
        p1 = baseline_ctr
        p2 = baseline_ctr * (1 + min_detectable_effect)
        p_avg = (p1 + p2) / 2

        z_alpha = stats.norm.ppf(1 - alpha / 2)
        z_beta = stats.norm.ppf(power)

        n = (z_alpha * np.sqrt(2 * p_avg * (1 - p_avg)) +
             z_beta * np.sqrt(p1 * (1 - p1) + p2 * (1 - p2))) ** 2 / (p2 - p1) ** 2
        return int(np.ceil(n))

    def _current_hour(self):
        from datetime import datetime
        return datetime.now().strftime('%Y%m%d%H')

    def _previous_hour(self):
        from datetime import datetime, timedelta
        return datetime.now() - timedelta(hours=1)

    def _get_experiment_groups(self, experiment_id):
        config = self._load_experiment_config(experiment_id)
        if config:
            return list(config['groups'].keys())
        return []
```

**4. 推荐请求中嵌入 A/B 实验**

```python
class RecommendationService:
    """推荐服务：在请求处理中集成 A/B 测试"""

    def recommend(self, user_id, count=10):
        # 1. 获取用户命中的所有实验
        experiments = self._get_user_experiments(user_id)

        # 2. 根据实验配置选择不同的推荐策略
        recall_config = experiments.get('recall', {})
        ranking_config = experiments.get('ranking', {})

        # 3. 执行推荐流程
        candidates = self.recaller.recall(user_id, config=recall_config)
        pre_ranked = self.pre_ranker.rank(user_id, candidates)
        ranked = self.ranker.rank(user_id, pre_ranked, config=ranking_config)
        result = self.re_ranker.rerank(ranked, count)

        # 4. 在返回结果中携带实验信息（用于指标归因）
        experiment_info = {
            exp_id: group for exp_id, group in experiments.items()
        }

        return result, experiment_info

    def _get_user_experiments(self, user_id):
        """获取用户命中的所有活跃实验"""
        # 从 Redis 加载所有活跃实验列表
        active_experiments = self.redis.smembers("ab:active_experiments")
        experiments = {}

        for exp_id in active_experiments:
            exp_id = exp_id.decode() if isinstance(exp_id, bytes) else exp_id
            group, config = self.ab_service.get_experiment_group(user_id, exp_id)
            if group:
                layer = config.get('layer', 'default')
                experiments[layer] = {
                    'experiment_id': exp_id,
                    'group': group,
                    'config': config,
                }

        return experiments
```

## 常见陷阱（深度分析）

### 陷阱 1：只用一路召回

**问题：** 只用协同过滤召回 → 用户只看到"和你相似的人看了什么"→ 信息茧房

**具体表现：** 用户只喜欢看猫视频，推荐系统永远只推荐猫视频。用户想看新闻时看不到。

**解决方案：** 至少 4 路召回（协同过滤 + 向量召回 + 标签召回 + 探索召回），每路贡献不同类型的候选。

### 陷阱 2：精排模型直接用于全量排序

**计算量分析：**
- 5000 万视频 × 2ms/次 = 100,000 秒 → 不可行
- 即使 GPU 批量推理：5000 万 × 0.1ms = 5000 秒 → 仍不可行
- 精排只能用于 100-200 条候选

### 陷阱 3：新视频零曝光

**后果：** 创作者上传了视频但无人观看 → 创作者离开平台 → 内容枯竭

**数据：** 某平台统计，新视频如果在 24 小时内获得 < 10 次曝光，创作者的再次上传概率下降 70%。

**解决方案：** 流量池赛马 + 内容理解辅助推荐。

### 陷阱 4：特征不实时

**场景：** 用户 5 分钟前点击了"滑雪"视频，但推荐系统仍在使用昨天的特征（用户兴趣是"美食"）→ 推荐美食视频 → 用户此时想看滑雪 → 体验差

**实时特征的价值：** 某平台 A/B 测试显示，实时特征（5分钟级更新）比离线特征（T+1 更新）的点击率提升 15%。

## 异常场景与容错方案

### 场景 1：特征存储数据漂移导致排序质量退化

**场景描述：** Flink 作业因上游 Kafka 分区 rebalance 导致消费延迟突增 5 分钟，期间大量用户特征未及时更新。精排模型使用过期的用户兴趣特征打分，导致推荐结果严重偏离用户当前兴趣——用户刚连续点赞了 5 个科技视频，推荐系统仍然基于"美食"兴趣打分，推荐了 20 条美食视频。

**根因分析：** 特征管道的实时性依赖于 Kafka → Flink → Redis 链路的低延迟。当任一环节出现延迟（Kafka rebalance、Flink 背压、Redis 写入抖动），特征存储中的数据就会与真实用户状态产生"漂移"。漂移越大，模型打分越不准。

**检测机制：**

```python
class FeatureDriftDetector:
    """特征数据漂移检测器：实时监控特征新鲜度"""

    def __init__(self, config):
        self.redis = config['redis']
        self.alert_threshold = config.get('drift_threshold_seconds', 300)  # 5 分钟

    def check_feature_freshness(self, user_id):
        """
        检查用户特征的新鲜度
        返回：特征是否过期、最后更新时间
        """
        # 读取用户兴趣标签的最后更新时间戳
        last_active_ts = self.redis.hget(f"user:interest:{user_id}", "last_active_ts")

        if not last_active_ts:
            return True, None  # 无时间戳，特征可能从未更新过

        last_update = int(last_active_ts)
        current_ts = int(time.time() * 1000)
        drift_seconds = (current_ts - last_update) / 1000

        is_stale = drift_seconds > self.alert_threshold
        return is_stale, drift_seconds

    def monitor_pipeline_lag(self):
        """监控 Kafka → Flink 链路的消费延迟（定时任务每 30 秒执行）"""
        from kafka import KafkaConsumer
        consumer = KafkaConsumer(
            'user_action_events',
            bootstrap_servers=self.config['kafka_brokers'],
            group_id='drift-monitor'
        )

        for partition in consumer.partitions_for_topic('user_action_events'):
            tp = TopicPartition('user_action_events', partition)
            consumer.assign([tp])
            end_offset = consumer.end_offsets([tp])[tp]
            current_offset = consumer.committed(tp) or 0
            lag = end_offset - current_offset

            if lag > 100000:  # 堆积超过 10 万条
                self._alert_pipeline_lag(partition, lag)

    def _alert_pipeline_lag(self, partition, lag):
        """告警：特征管道延迟过大"""
        logging.critical(
            f"Feature pipeline lag alert: partition={partition}, lag={lag}. "
            f"Feature drift risk: user features may be stale."
        )
        # 触发自动降级策略
        self._trigger_feature_degradation()
```

**自动降级与恢复：**

```python
class FeatureDegradationHandler:
    """特征漂移时的自动降级策略"""

    def __init__(self, config):
        self.redis = config['redis']
        self.mysql = config['mysql']
        self.drift_detector = FeatureDriftDetector(config)
        self.degradation_active = False

    def on_recommendation_request(self, user_id):
        """推荐请求前的特征新鲜度检查"""
        is_stale, drift_seconds = self.drift_detector.check_feature_freshness(user_id)

        if not is_stale:
            return 'normal'  # 特征新鲜，正常推荐

        # 特征过期，根据漂移程度选择降级策略
        if drift_seconds < 600:  # 5-10 分钟漂移
            # 策略 1：使用离线特征兜底（MySQL 中的 T+1 特征）
            return 'fallback_offline'
        elif drift_seconds < 3600:  # 10-60 分钟漂移
            # 策略 2：降级为热门推荐（不可信特征不用于个性化）
            return 'fallback_hot'
        else:
            # 策略 3：超长漂移，完全降级为热门 + 多样性
            return 'fallback_hot_diverse'

    def apply_degradation(self, user_id, strategy):
        """应用降级策略"""
        if strategy == 'normal':
            return None  # 无需降级

        if strategy == 'fallback_offline':
            # 使用 MySQL 离线特征替代 Redis 实时特征
            offline_features = self.mysql.query(
                "SELECT interest_tags FROM user_features WHERE user_id = %s",
                (user_id,)
            )
            if offline_features:
                # 临时覆盖 Redis 中的过期特征
                self.redis.hset(f"user:interest:{user_id}", "_using_offline", 1)
                return offline_features[0]

        elif strategy == 'fallback_hot':
            # 完全降级为热门推荐
            return {'degraded': True, 'strategy': 'hot'}

        elif strategy == 'fallback_hot_diverse':
            # 热门 + 多样性推荐
            return {'degraded': True, 'strategy': 'hot_diverse'}

    def recover_from_degradation(self):
        """
        从降级中恢复（Flink 消费追上后调用）
        1. 清除降级标记
        2. 验证特征新鲜度
        3. 逐步切回实时特征
        """
        # 检查 Flink 消费延迟是否恢复
        is_stale, _ = self.drift_detector.check_feature_freshness("health_check_user")
        if is_stale:
            return  # 尚未恢复

        # 清除所有降级标记
        # 使用 SCAN 而非 KEYS 避免阻塞 Redis
        cursor = 0
        while True:
            cursor, keys = self.redis.scan(cursor, match="user:interest:*", count=1000)
            for key in keys:
                self.redis.hdel(key, "_using_offline")
            if cursor == 0:
                break

        self.degradation_active = False
        logging.info("Feature degradation recovered, switched back to realtime features")
```

### 场景 2：模型部署失败与自动回滚

**场景描述：** 凌晨 3 点自动部署新版精排模型（v3），新模型加载后 GPU 推理结果全部返回 NaN——原因是模型训练时使用了新版本的 TensorFlow，但线上推理服务的 TensorFlow 版本未同步升级，导致模型文件格式不兼容。所有推荐请求返回空结果或异常分数。

**检测与自动回滚：**

```python
class ModelDeploymentManager:
    """模型部署管理器：支持金丝雀发布 + 自动回滚"""

    def __init__(self, config):
        self.redis = config['redis']
        self.mysql = config['mysql']
        self.model_dir = config['model_dir']  # /data/models/ranking/
        self.current_model_version = None
        self.rollback_version = None
        # 健康检查指标
        self.health_metrics = {
            'nan_rate_threshold': 0.01,      # NaN 比例超过 1% 触发回滚
            'error_rate_threshold': 0.05,    # 错误率超过 5% 触发回滚
            'latency_p99_threshold': 150,    # P99 延迟超过 150ms 触发回滚
            'ctr_drop_threshold': -0.10,     # CTR 下降超过 10% 触发回滚
        }

    def deploy_model(self, new_version, canary_percent=5):
        """
        金丝雀部署新模型：
        1. 先将 5% 流量切到新模型
        2. 观察 10 分钟
        3. 指标正常则逐步扩大流量
        4. 指标异常则自动回滚
        """
        # 1. 保存当前版本作为回滚目标
        self.rollback_version = self._get_current_serving_version()

        # 2. 加载新模型到 GPU（热加载，不中断服务）
        load_success = self._load_model_to_gpu(new_version)
        if not load_success:
            logging.critical(f"Model {new_version} failed to load, aborting deployment")
            return False

        # 3. 快速验证：用 100 条测试数据检查模型输出
        validation_result = self._validate_model_output(new_version)
        if not validation_result['passed']:
            logging.critical(
                f"Model {new_version} validation failed: {validation_result['reason']}. "
                f"Rolling back to {self.rollback_version}"
            )
            self._rollback()
            return False

        # 4. 设置金丝雀流量比例
        self.redis.set(f"model:ranking:canary_version", new_version)
        self.redis.set(f"model:ranking:canary_percent", canary_percent)

        # 5. 启动健康监控（10 分钟观察期）
        self._start_canary_monitor(new_version, duration_minutes=10)

        return True

    def _validate_model_output(self, model_version):
        """使用测试数据验证模型输出是否正常"""
        test_features = self._load_test_features()  # 100 条预置测试数据

        try:
            model = self._get_model(model_version)
            scores = model.predict_batch(test_features)
        except Exception as e:
            return {'passed': False, 'reason': f'Predict error: {e}'}

        # 检查 1：是否有 NaN
        nan_count = np.sum(np.isnan(scores))
        if nan_count > 0:
            return {
                'passed': False,
                'reason': f'NaN detected: {nan_count}/{len(scores)} scores are NaN'
            }

        # 检查 2：分数分布是否合理（不应该全部为同一值）
        score_std = np.std(scores)
        if score_std < 1e-6:
            return {
                'passed': False,
                'reason': f'No score variance: std={score_std}, all scores may be identical'
            }

        # 检查 3：分数范围是否在合理区间 [0, 1]
        if np.min(scores) < -0.5 or np.max(scores) > 1.5:
            return {
                'passed': False,
                'reason': f'Score out of range: min={np.min(scores)}, max={np.max(scores)}'
            }

        return {'passed': True}

    def _start_canary_monitor(self, canary_version, duration_minutes=10):
        """金丝雀期间的健康监控"""
        start_time = time.time()
        baseline_ctr = self._get_recent_ctr()  # 部署前的 CTR 基线

        while time.time() - start_time < duration_minutes * 60:
            time.sleep(30)  # 每 30 秒检查一次

            # 收集金丝雀组的指标
            metrics = self._collect_canary_metrics(canary_version)

            # 检查各项健康指标
            if metrics['nan_rate'] > self.health_metrics['nan_rate_threshold']:
                logging.critical(
                    f"Canary NaN rate {metrics['nan_rate']:.4f} exceeds threshold "
                    f"{self.health_metrics['nan_rate_threshold']}. Rolling back."
                )
                self._rollback()
                return

            if metrics['error_rate'] > self.health_metrics['error_rate_threshold']:
                logging.critical(
                    f"Canary error rate {metrics['error_rate']:.4f} exceeds threshold. Rolling back."
                )
                self._rollback()
                return

            if metrics['latency_p99'] > self.health_metrics['latency_p99_threshold']:
                logging.critical(
                    f"Canary P99 latency {metrics['latency_p99']}ms exceeds threshold. Rolling back."
                )
                self._rollback()
                return

            # CTR 下降检查（需要足够样本量）
            if metrics['sample_count'] > 1000:
                ctr_change = (metrics['ctr'] - baseline_ctr) / max(baseline_ctr, 1e-10)
                if ctr_change < self.health_metrics['ctr_drop_threshold']:
                    logging.critical(
                        f"Canary CTR dropped {ctr_change*100:.1f}%. Rolling back."
                    )
                    self._rollback()
                    return

            logging.info(
                f"Canary check: nan_rate={metrics['nan_rate']:.4f}, "
                f"error_rate={metrics['error_rate']:.4f}, "
                f"p99={metrics['latency_p99']}ms, "
                f"ctr={metrics['ctr']:.4f}"
            )

        # 观察期通过，逐步扩大流量
        self._expand_canary_traffic(canary_version)

    def _expand_canary_traffic(self, canary_version):
        """逐步扩大金丝雀流量：5% → 25% → 50% → 100%"""
        stages = [25, 50, 100]
        for percent in stages:
            self.redis.set(f"model:ranking:canary_percent", percent)
            logging.info(f"Expanding canary traffic to {percent}%")

            # 等待 5 分钟观察
            time.sleep(300)

            metrics = self._collect_canary_metrics(canary_version)
            if metrics['error_rate'] > 0.02 or metrics['nan_rate'] > 0.005:
                logging.critical(f"Metrics degraded at {percent}%, rolling back")
                self._rollback()
                return

        # 全量切换完成
        self.current_model_version = canary_version
        self.redis.delete("model:ranking:canary_version")
        self.redis.delete("model:ranking:canary_percent")
        self.redis.set("model:ranking:current_version", canary_version)
        logging.info(f"Model {canary_version} fully deployed")

    def _rollback(self):
        """回滚到上一个稳定版本"""
        logging.critical(f"Rolling back to model version {self.rollback_version}")

        # 1. 立即切换流量回旧模型
        self.redis.delete("model:ranking:canary_version")
        self.redis.delete("model:ranking:canary_percent")
        self.redis.set("model:ranking:current_version", self.rollback_version)

        # 2. 卸载有问题的模型，释放 GPU 内存
        # （GPU 模型管理器会根据 Redis 配置切换）

        # 3. 记录回滚事件
        self.mysql.execute("""
            INSERT INTO model_deployment_log
            (model_version, action, reason, created_at)
            VALUES (%s, 'rollback', %s, NOW())
        """, (self.current_model_version, 'Auto rollback: health check failed'))

        # 4. 发送告警
        self._send_alert(f"Model deployment rollback: {self.current_model_version} → {self.rollback_version}")

    def _collect_canary_metrics(self, canary_version):
        """收集金丝雀分组的实时指标"""
        # 从 Redis 实时计数器读取
        minute_key = f"model:metrics:{canary_version}:{int(time.time()/60)}"
        total = int(self.redis.get(f"{minute_key}:total") or 0)
        errors = int(self.redis.get(f"{minute_key}:errors") or 0)
        nans = int(self.redis.get(f"{minute_key}:nans") or 0)
        clicks = int(self.redis.get(f"{minute_key}:clicks") or 0)
        exposures = int(self.redis.get(f"{minute_key}:exposures") or 0)

        # 从延迟直方图获取 P99
        latency_p99 = self.redis.get(f"{minute_key}:latency_p99") or 0

        return {
            'sample_count': total,
            'error_rate': errors / max(total, 1),
            'nan_rate': nans / max(total, 1),
            'ctr': clicks / max(exposures, 1),
            'latency_p99': int(latency_p99),
        }

    def _send_alert(self, message):
        """发送告警（钉钉/飞书/短信）"""
        # 实际实现调用告警 SDK
        logging.critical(f"ALERT: {message}")
```

### 场景 3：爆款视频引发缓存击穿

**场景描述：** 一个猫咪搞笑视频突然爆火，1 小时内从 1000 播放量飙升至 500 万。大量用户同时请求该视频的特征数据，Redis 中该视频的特征缓存恰好过期（TTL 到期），数万请求同时穿透到 MySQL，导致 MySQL 瞬间负载飙升，连接池耗尽，推荐服务整体超时。

**缓存击穿的根因：** 热点 key 的 TTL 同时过期 + 高并发请求同时发现缓存失效 → 大量请求同时回源数据库。

**防护方案：**

```python
class CacheStampedeProtection:
    """缓存击穿防护：热点 key 永不过期 + 互斥回源 + 空值缓存"""

    def __init__(self, config):
        self.redis = config['redis']
        self.mysql = config['mysql']
        self.lock_timeout = 5  # 互斥锁超时 5 秒
        self.hot_key_threshold = 1000  # 1 分钟内访问 1000 次视为热点 key

    def get_video_features(self, video_id):
        """
        获取视频特征（带缓存击穿防护）
        策略层次：本地缓存 → Redis 缓存 → 互斥回源 → MySQL
        """
        # 1. 本地内存缓存（进程级，最快，无网络开销）
        local_cached = self._get_local_cache(video_id)
        if local_cached:
            return local_cached

        # 2. Redis 缓存
        cache_key = f"video:info:{video_id}"
        cached = self.redis.hgetall(cache_key)
        if cached:
            result = {k.decode(): v.decode() for k, v in cached.items()}
            self._set_local_cache(video_id, result, ttl=10)  # 本地缓存 10 秒

            # 热点 key 检测：异步更新热点标记
            self._check_and_mark_hot_key(video_id)
            return result

        # 3. 缓存未命中——互斥回源（防止击穿）
        return self._mutex_rebuild_cache(video_id, cache_key)

    def _mutex_rebuild_cache(self, video_id, cache_key):
        """互斥回源：只允许一个请求回源数据库，其他请求等待"""
        lock_key = f"lock:cache_rebuild:{video_id}"

        # 尝试获取互斥锁（SETNX + 过期时间）
        acquired = self.redis.set(lock_key, 1, nx=True, ex=self.lock_timeout)

        if acquired:
            # 获得锁：回源数据库
            try:
                result = self._load_from_mysql_and_rebuild_cache(video_id, cache_key)
                return result
            finally:
                # 释放锁
                self.redis.delete(lock_key)
        else:
            # 未获得锁：等待其他线程重建缓存
            for _ in range(50):  # 最多等 500ms
                time.sleep(0.01)
                cached = self.redis.hgetall(cache_key)
                if cached:
                    return {k.decode(): v.decode() for k, v in cached.items()}

            # 超时仍未获得缓存，返回降级数据
            logging.warning(f"Cache rebuild timeout for video {video_id}, returning degraded data")
            return self._get_degraded_features(video_id)

    def _load_from_mysql_and_rebuild_cache(self, video_id, cache_key):
        """从 MySQL 加载特征并重建缓存"""
        row = self.mysql.query(
            "SELECT * FROM video_features WHERE video_id = %s", (video_id,)
        )

        if not row:
            # 数据库中也无数据——缓存空值，防止缓存穿透
            self.redis.setex(f"video:info:null:{video_id}", 60, "1")  # 空值缓存 60 秒
            return {}

        # 重建缓存——热点 key 永不过期（后台异步刷新）
        is_hot = self.redis.sismember("hot:keys:videos", video_id)
        ttl = None if is_hot else 86400  # 热点 key 不设 TTL

        pipe = self.redis.pipeline()
        for k, v in row[0].items():
            if v is not None and k not in ('visual_embedding', 'text_embedding',
                                             'audio_embedding', 'fusion_embedding'):
                pipe.hset(cache_key, k, str(v))
        if ttl:
            pipe.expire(cache_key, ttl)
        pipe.execute()

        # 设置本地缓存
        result = row[0]
        self._set_local_cache(video_id, result, ttl=30)

        return result

    def _check_and_mark_hot_key(self, video_id):
        """热点 key 检测与标记（异步执行）"""
        # 使用滑动窗口计数器：1 分钟内访问次数
        minute_key = f"hot:counter:{video_id}:{int(time.time()/60)}"
        count = self.redis.incr(minute_key)
        if count == 1:
            self.redis.expire(minute_key, 120)  # 计数器 TTL 2 分钟

        if count >= self.hot_key_threshold:
            # 标记为热点 key
            self.redis.sadd("hot:keys:videos", video_id)
            # 热点 key 取消 TTL（永不过期，后台异步刷新）
            self.redis.persist(f"video:info:{video_id}")

    def _get_local_cache(self, video_id):
        """获取本地内存缓存（使用进程内 LRU 缓存）"""
        # 实际实现使用 cachetools.LRUCache 或自定义缓存
        return getattr(self, '_local_cache', {}).get(video_id)

    def _set_local_cache(self, video_id, data, ttl=10):
        """设置本地内存缓存"""
        if not hasattr(self, '_local_cache'):
            self._local_cache = {}
        self._local_cache[video_id] = data

    def _get_degraded_features(self, video_id):
        """降级特征：返回最基础的视频信息"""
        # 从 Redis 全局视频列表中获取最基本信息
        return {
            'video_id': video_id,
            'degraded': True,
            'category_id': 0,
        }


class HotKeyBackgroundRefresher:
    """热点 key 后台异步刷新（避免 TTL 过期导致击穿）"""

    def __init__(self, config):
        self.redis = config['redis']
        self.mysql = config['mysql']
        self.refresh_interval = 300  # 每 5 分钟刷新一次热点 key

    def refresh_hot_keys(self):
        """定时任务：刷新所有热点 key 的缓存数据"""
        hot_keys = self.redis.smembers("hot:keys:videos")
        for video_id in hot_keys:
            video_id = int(video_id) if isinstance(video_id, bytes) else int(video_id)
            try:
                self._refresh_single_key(video_id)
            except Exception as e:
                logging.error(f"Failed to refresh hot key {video_id}: {e}")

        # 清理已不再是热点的 key
        self._cleanup_cold_keys()

    def _refresh_single_key(self, video_id):
        """刷新单个热点 key 的缓存"""
        cache_key = f"video:info:{video_id}"
        row = self.mysql.query(
            "SELECT * FROM video_features WHERE video_id = %s", (video_id,)
        )
        if row:
            pipe = self.redis.pipeline()
            pipe.delete(cache_key)  # 先删后写，保证一致性
            for k, v in row[0].items():
                if v is not None and k not in ('visual_embedding', 'text_embedding',
                                                 'audio_embedding', 'fusion_embedding'):
                    pipe.hset(cache_key, k, str(v))
            # 热点 key 不设 TTL
            pipe.execute()
            logging.debug(f"Refreshed hot key cache for video {video_id}")

    def _cleanup_cold_keys(self):
        """清理已不再热点队列的 key（重新设置 TTL）"""
        hot_keys = self.redis.smembers("hot:keys:videos")
        for video_id in hot_keys:
            # 检查最近 5 分钟的访问计数
            minute_key = f"hot:counter:{video_id}:{int(time.time()/60)}"
            count = int(self.redis.get(minute_key) or 0)
            if count < self.hot_key_threshold * 0.1:  # 访问量降到阈值的 10% 以下
                self.redis.srem("hot:keys:videos", video_id)
                # 恢复正常 TTL
                self.redis.expire(f"video:info:{video_id}", 86400)
```

**多层防护总结：**

| 防护层 | 机制 | 防护目标 |
|--------|------|---------|
| 第 1 层 | 本地内存缓存（10-30 秒 TTL） | 减少对 Redis 的请求量 |
| 第 2 层 | Redis 热点 key 永不过期 | 避免 TTL 过期导致的击穿 |
| 第 3 层 | 互斥回源（分布式锁） | 防止多请求同时穿透到 MySQL |
| 第 4 层 | 空值缓存 | 防止对不存在数据的缓存穿透 |
| 第 5 层 | 后台异步刷新 | 热点 key 数据始终是最新的 |
| 第 6 层 | 降级返回 | 所有缓存都不可用时的兜底 |

### 场景 4：召回通道故障与优雅降级

**场景描述：** FAISS 向量索引服务所在节点因硬件故障宕机，向量召回通道完全不可用。同时，协同过滤通道依赖的 Redis Cluster 正在做 slot 迁移，响应延迟从 10ms 飙升到 500ms。6 路召回中有 2 路异常，剩余 4 路召回的候选数量不足，推荐结果变窄，用户体验下降。

**降级策略与实现：**

```python
class RecallDegradationManager:
    """召回通道故障降级管理器"""

    # 召回通道优先级与降级方案
    CHANNEL_DEGRADATION_PLAN = {
        'cf': {
            'priority': 1,           # 优先级最高
            'timeout_ms': 15,        # 正常超时
            'degraded_timeout_ms': 50,  # 降级后放宽超时
            'fallback_channel': 'hot',  # 降级后替代通道
            'min_candidates': 50,    # 最少需要返回的候选数
        },
        'vector': {
            'priority': 2,
            'timeout_ms': 25,
            'degraded_timeout_ms': 60,
            'fallback_channel': 'content',  # 向量召回降级为内容召回
            'min_candidates': 100,
        },
        'content': {
            'priority': 3,
            'timeout_ms': 10,
            'degraded_timeout_ms': 30,
            'fallback_channel': 'hot',
            'min_candidates': 50,
        },
        'hot': {
            'priority': 4,           # 最可靠的兜底通道
            'timeout_ms': 5,
            'degraded_timeout_ms': 10,
            'fallback_channel': None,  # 热门无降级替代
            'min_candidates': 30,
        },
        'follow': {
            'priority': 3,
            'timeout_ms': 8,
            'degraded_timeout_ms': 20,
            'fallback_channel': 'hot',
            'min_candidates': 20,
        },
        'explore': {
            'priority': 5,           # 最低优先级
            'timeout_ms': 5,
            'degraded_timeout_ms': 10,
            'fallback_channel': 'hot',
            'min_candidates': 10,
        },
    }

    def __init__(self, config):
        self.redis = config['redis']
        self.channel_health = {}  # 通道健康状态
        self.fuse_state = {}      # 熔断状态

    def recall_with_degradation(self, user_id, total_count=500):
        """带降级策略的多路召回"""
        channel_results = {}
        degraded_channels = set()

        # 1. 评估各通道健康状态
        for channel_name, plan in self.CHANNEL_DEGRADATION_PLAN.items():
            health = self._check_channel_health(channel_name)
            self.channel_health[channel_name] = health

            if health['status'] == 'fused':
                # 通道已熔断，直接使用降级方案
                degraded_channels.add(channel_name)
                fallback = plan['fallback_channel']
                if fallback and fallback not in degraded_channels:
                    # 增加替代通道的召回数量
                    channel_results[channel_name] = self._recall_from_fallback(
                        user_id, fallback, plan['min_candidates']
                    )
                else:
                    channel_results[channel_name] = []
                continue

        # 2. 执行正常通道的召回（带动态超时）
        from concurrent.futures import ThreadPoolExecutor, as_completed

        normal_channels = [name for name in self.CHANNEL_DEGRADATION_PLAN
                           if name not in degraded_channels]

        with ThreadPoolExecutor(max_workers=len(normal_channels)) as executor:
            futures = {}
            for channel_name in normal_channels:
                plan = self.CHANNEL_DEGRADATION_PLAN[channel_name]
                health = self.channel_health[channel_name]

                # 根据健康状态调整超时
                timeout = plan['degraded_timeout_ms'] if health['status'] == 'degraded' \
                          else plan['timeout_ms']

                recaller = self._get_recaller(channel_name)
                future = executor.submit(recaller.recall, user_id, plan['min_candidates'] * 2)
                futures[future] = (channel_name, timeout)

            for future in as_completed(futures):
                channel_name, timeout_ms = futures[future]
                try:
                    result = future.result(timeout=timeout_ms / 1000.0)
                    channel_results[channel_name] = result
                    self._record_channel_success(channel_name)
                except Exception as e:
                    channel_results[channel_name] = []
                    self._record_channel_failure(channel_name)
                    logging.warning(f"Recall channel {channel_name} failed: {e}")

        # 3. 如果候选总数不足，扩大健康通道的召回数量
        total_candidates = sum(len(v) for v in channel_results.values())
        if total_candidates < total_count:
            logging.warning(
                f"Recall candidates insufficient: {total_candidates} < {total_count}. "
                f"Expanding healthy channels."
            )
            shortage = total_count - total_candidates
            # 按优先级选择健康通道扩大召回
            for channel_name, plan in sorted(self.CHANNEL_DEGRADATION_PLAN.items(),
                                              key=lambda x: x[1]['priority']):
                if channel_name in degraded_channels:
                    continue
                if shortage <= 0:
                    break
                extra = self._expand_channel_recall(user_id, channel_name, shortage)
                channel_results[channel_name].extend(extra)
                shortage -= len(extra)

        # 4. 最终兜底：如果仍不足，从热门视频补充
        if sum(len(v) for v in channel_results.values()) < total_count * 0.5:
            hot_fallback = self._get_hot_videos(total_count)
            channel_results['hot_emergency'] = hot_fallback
            logging.critical(
                "Emergency fallback: using hot videos to fill insufficient recall candidates"
            )

        return channel_results

    def _check_channel_health(self, channel_name):
        """
        检查通道健康状态
        返回: {'status': 'healthy'|'degraded'|'fused', 'fail_rate': float, 'latency_p99': int}
        """
        health_key = f"recall:health:{channel_name}"
        health_data = self.redis.hgetall(health_key)

        if not health_data:
            return {'status': 'healthy', 'fail_rate': 0, 'latency_p99': 0}

        fail_rate = float(health_data.get(b'fail_rate', 0))
        latency_p99 = int(health_data.get(b'latency_p99', 0))
        consecutive_fails = int(health_data.get(b'consecutive_fails', 0))

        if consecutive_fails >= 5:
            status = 'fused'
        elif fail_rate > 0.3 or latency_p99 > 200:
            status = 'degraded'
        else:
            status = 'healthy'

        return {'status': status, 'fail_rate': fail_rate, 'latency_p99': latency_p99}

    def _record_channel_success(self, channel_name):
        """记录通道成功"""
        health_key = f"recall:health:{channel_name}"
        pipe = self.redis.pipeline()
        pipe.hset(health_key, 'consecutive_fails', 0)
        pipe.hincrby(health_key, 'success_count', 1)
        pipe.expire(health_key, 300)
        pipe.execute()

    def _record_channel_failure(self, channel_name):
        """记录通道失败"""
        health_key = f"recall:health:{channel_name}"
        pipe = self.redis.pipeline()
        pipe.hincrby(health_key, 'consecutive_fails', 1)
        pipe.hincrby(health_key, 'fail_count', 1)
        # 计算失败率
        fail_count = int(self.redis.hget(health_key, 'fail_count') or 0)
        success_count = int(self.redis.hget(health_key, 'success_count') or 0)
        total = fail_count + success_count
        if total > 0:
            pipe.hset(health_key, 'fail_rate', round(fail_count / total, 4))
        pipe.expire(health_key, 300)
        pipe.execute()

        # 检查是否需要触发熔断
        consecutive_fails = int(self.redis.hget(health_key, 'consecutive_fails') or 0)
        if consecutive_fails >= 5:
            logging.critical(
                f"Recall channel {channel_name} fused: {consecutive_fails} consecutive failures"
            )

    def _recall_from_fallback(self, user_id, fallback_channel, count):
        """从降级替代通道召回"""
        recaller = self._get_recaller(fallback_channel)
        try:
            return recaller.recall(user_id, count)
        except Exception:
            return []

    def _expand_channel_recall(self, user_id, channel_name, extra_count):
        """扩大某个通道的召回数量"""
        recaller = self._get_recaller(channel_name)
        try:
            return recaller.recall(user_id, extra_count)
        except Exception:
            return []

    def _get_hot_videos(self, count):
        """紧急兜底：返回全站热门视频"""
        return [int(vid) for vid in self.redis.zrevrange("global:hot_videos", 0, count - 1)]
```

**降级策略决策矩阵：**

| 故障通道 | 降级替代 | 候选补充策略 | 影响评估 |
|---------|---------|------------|---------|
| 协同过滤 | 热门召回（带分类过滤） | 热门通道扩大到 100 条 | CTR 预计下降 5-8% |
| 向量召回 | 内容召回（标签匹配） | 内容通道扩大到 200 条 | 兴趣扩展能力下降，信息茧房风险 |
| 标签召回 | 热门召回 | 热门通道扩大 | 多样性略降 |
| 热门召回 | 无（最可靠通道） | 其他通道扩大 | 极少发生 |
| 关注召回 | 热门召回 | 社交信号丢失 | 互动率下降 3-5% |
| 探索召回 | 无（可牺牲） | 探索性内容减少 | 信息茧房风险增加 |

## 延伸思考

- **多目标优化**：推荐系统不只是优化点击率，还要优化停留时长、完播率、互动率、商业化收益。如何设计多目标损失函数？
- **推荐可解释性**：用户问"为什么推荐这个视频"，系统如何给出人类可理解的解释？
- **隐私保护**：如何在不收集用户行为数据的前提下做推荐？联邦学习方案？
## 推荐服务层完整实现

```python
class RecommendationServingService:
    """推荐服务层：多源合并 + 多样性重排 + 业务过滤"""

    def get_recommendations(self, user_id, scene="home", count=20):
        """获取推荐列表"""
        # 1. 多源召回
        cf_items = self.cf_recaller.recall(user_id, limit=50)
        content_items = self.content_recaller.recall(user_id, limit=50)
        hot_items = self.hot_recaller.recall(limit=20)
        new_items = self.new_recaller.recall(limit=10)

        # 2. 合并去重
        all_items = self._merge_and_dedup(
            [cf_items, content_items, hot_items, new_items])

        # 3. 粗排（轻量模型打分）
        scored = self.ranking_model.predict(user_id, all_items)

        # 4. 业务过滤
        filtered = self._apply_business_rules(user_id, scored)

        # 5. 精排 + 多样性重排（MMR）
        reranked = self._diversity_rerank(filtered, count, lambda_diversity=0.5)

        # 6. 新鲜度加权
        for item in reranked:
            hours_since_publish = (now() - item["publish_time"]).total_seconds() / 3600
            freshness_boost = max(0, 1 - hours_since_publish / 168)  # 7天衰减
            item["final_score"] = item["score"] * (1 + freshness_boost * 0.2)

        reranked.sort(key=lambda x: x["final_score"], reverse=True)

        # 7. A/B 测试分桶
        bucket = self._get_ab_bucket(user_id, scene)

        return {
            "items": reranked[:count],
            "bucket": bucket,
            "request_id": str(uuid4())
        }

    def _apply_business_rules(self, user_id, items):
        """业务规则过滤"""
        user = self.db.get_user(user_id)
        result = []
        paid_count = 0

        for item in items:
            # 年龄限制
            if item.get("age_restriction", 0) > (user.get("age", 100)):
                continue
            # 付费内容配额（每次推荐最多 2 个付费）
            if item.get("is_paid"):
                paid_count += 1
                if paid_count > 2:
                    continue
            # 已看过的内容
            if self.redis.sismember(f"watched:{user_id}", item["id"]):
                continue
            result.append(item)

        return result

    def _diversity_rerank(self, items, count, lambda_diversity=0.5):
        """MMR 多样性重排"""
        selected = []
        remaining = list(items)

        # 选第一个（最高分）
        if remaining:
            selected.append(remaining.pop(0))

        while len(selected) < count and remaining:
            best_mmr = -float('inf')
            best_idx = 0

            for i, item in enumerate(remaining):
                # 与已选内容的最小相似度
                min_sim = min(
                    self._similarity(item, s) for s in selected)
                mmr = lambda_diversity * item["score"] - \
                    (1 - lambda_diversity) * min_sim
                if mmr > best_mmr:
                    best_mmr = mmr
                    best_idx = i

            selected.append(remaining.pop(best_idx))

        return selected
```

## 特征存储架构

```python
class FeatureStore:
    """特征存储：离线计算 + 在线服务"""

    def get_online_features(self, user_id, feature_names):
        """获取在线特征（从 Redis）"""
        features = {}
        for name in feature_names:
            value = self.redis.hget(f"features:{user_id}", name)
            if value:
                features[name] = json.loads(value)

        # 检查特征新鲜度
        stale_features = []
        for name, feat in features.items():
            age = (now() - datetime.fromisoformat(feat["updated_at"])).total_seconds()
            if age > 3600:  # 1 小时以上 → 过期
                stale_features.append(name)

        if stale_features:
            # 触发异步刷新
            self.refresh_queue.publish(json.dumps({
                "user_id": user_id, "features": stale_features
            }))

        return {name: feat["value"] for name, feat in features.items()}

    def compute_offline_features(self, date):
        """离线计算特征（Spark 作业）"""
        # 用户画像特征
        self.spark.sql("""
            INSERT OVERWRITE user_features PARTITION (dt = '{date}')
            SELECT
                user_id,
                AVG(watch_duration_minutes) as avg_watch_duration,
                COUNT(DISTINCT video_id) as unique_videos_watched,
                COUNT(DISTINCT category_id) as category_diversity,
                SUM(CASE WHEN is_liked THEN 1 ELSE 0 END) * 1.0 /
                    COUNT(*) as like_rate,
                MAX(watch_time) as last_active_time
            FROM user_watch_log
            WHERE dt = '{date}'
            GROUP BY user_id
        """.format(date=date))

        # 同步到 Redis
        features = self.spark.sql(
            "SELECT * FROM user_features WHERE dt = '{date}'".format(date=date))
        for row in features.collect():
            key = f"features:{row['user_id']}"
            for col in features.columns:
                if col not in ("user_id", "dt"):
                    self.redis.hset(key, col, json.dumps({
                        "value": row[col], "updated_at": now().isoformat()
                    }))
```

## 冷启动解决方案

```python
class ColdStartService:
    """冷启动解决方案"""

    def recommend_for_new_user(self, user_info):
        """新用户推荐"""
        # 1. 基于注册信息推断初始兴趣
        interests = self._infer_from_registration(user_info)

        # 2. 基于人口统计学推荐
        demo_items = self._demographic_recommend(
            user_info.get("age_group"), user_info.get("gender"))

        # 3. 热门内容兜底
        hot_items = self.hot_recaller.recall(limit=20)

        # 4. 探索性推荐（epsilon-greedy）
        epsilon = 0.3  # 30% 概率探索
        all_items = interests + demo_items + hot_items
        if random.random() < epsilon:
            # 探索：加入随机内容
            random_items = self._get_random_diverse(limit=5)
            all_items.extend(random_items)

        # 5. 去重排序
        deduped = self._dedup(all_items)
        return deduped[:20]

    def recommend_for_new_video(self, video_info):
        """新视频推荐（解决内容冷启动）"""
        # 1. 基于 title/tags 生成 embedding
        embedding = self.embedding_model.encode(
            video_info["title"] + " " + " ".join(video_info.get("tags", [])))

        # 2. 找相似视频 → 推荐给看过相似视频的用户
        similar_videos = self.vector_db.search(embedding, top_k=20)
        potential_users = set()
        for sv in similar_videos:
            viewers = self.redis.smembers(f"video_viewers:{sv['id']}")
            potential_users.update(viewers)

        # 3. 探索流量：分配曝光机会
        explore_users = self._allocate_explore_traffic(
            video_info["id"], ratio=0.05)  # 5% 流量用于探索

        return {
            "video_id": video_info["id"],
            "similar_video_ids": [sv["id"] for sv in similar_videos],
            "potential_user_count": len(potential_users),
            "explore_user_count": len(explore_users)
        }
```

## 异常场景补充

### 场景：特征服务超时

```
触发：Redis 特征服务延迟飙升 → 推荐请求超时
检测：
  1. 特征获取延迟 > 100ms → 告警
  2. 推荐接口 P99 > 500ms → 严重告警
处理：
  1. 降级：使用缓存特征（即使过期）
  2. 无缓存 → 使用默认特征值
  3. 推荐结果降级为热门列表
预防：特征本地缓存 + 默认值兜底 + 热门列表降级
```

### 场景：冷启动推荐质量过低

```
触发：新用户首次推荐点击率 < 5% → 远低于老用户 15%
检测：
  1. 新用户 CTR 持续 < 5% → 冷启动效果差
  2. 新用户次日留存 < 20% → 推荐未留住用户
处理：
  1. 提高探索比例（epsilon 0.3 → 0.5）
  2. 增加注册引导（选择兴趣标签）
  3. 前 3 次推荐人工精选高质量内容
预防：注册兴趣引导 + 精选冷启动池 + 探索流量
```

## 实时推荐流水线完整实现

```python
class RealtimeRecommendationPipeline:
    """实时推荐流水线：Kafka Streams + 实时特征更新"""

    def process_user_action(self, action):
        """处理用户行为 → 更新实时特征 → 刷新推荐"""
        # 1. 写入行为日志
        self.db.insert("user_actions", {
            "user_id": action["user_id"],
            "video_id": action["video_id"],
            "action_type": action["type"],  # watch / like / share / skip
            "duration_seconds": action.get("duration", 0),
            "timestamp": now()
        })

        # 2. 更新实时特征（Redis）
        self._update_realtime_features(action)

        # 3. 失效推荐缓存
        self.redis.delete(f"rec_cache:{action['user_id']}")

        # 4. 发布事件到 Kafka
        self.kafka.produce("user_actions", json.dumps({
            "user_id": action["user_id"],
            "video_id": action["video_id"],
            "action_type": action["type"],
            "timestamp": now().isoformat()
        }))

    def _update_realtime_features(self, action):
        """更新实时特征"""
        user_key = f"rt_features:{action['user_id']}"
        pipe = self.redis.pipeline()

        if action["type"] == "watch":
            pipe.hincrbyfloat(user_key, "total_watch_minutes",
                action.get("duration", 0) / 60)
            pipe.hincrby(user_key, "videos_watched_today", 1)
        elif action["type"] == "like":
            pipe.hincrby(user_key, "likes_today", 1)
            # 更新用户兴趣向量（简化：标签加权）
            video_tags = self.db.get_video_tags(action["video_id"])
            for tag in video_tags:
                pipe.hincrbyfloat(user_key, f"tag_weight:{tag}", 0.1)
        elif action["type"] == "skip":
            pipe.hincrby(user_key, "skips_today", 1)

        pipe.hset(user_key, "last_active", now().isoformat())
        pipe.execute()
```

## 推荐 A/B 测试框架

```python
class RecommendationABTest:
    """推荐 A/B 测试框架"""

    def assign_bucket(self, user_id, experiment_id):
        """分配实验桶（一致性哈希）"""
        experiment = self.db.get_experiment(experiment_id)
        hash_value = int(hashlib.md5(
            f"{user_id}:{experiment_id}".encode()).hexdigest(), 16) % 100

        # 根据实验配置分配桶
        for bucket in json.loads(experiment["buckets"]):
            if hash_value < bucket["cumulative_percentage"]:
                return {"bucket": bucket["name"], "algorithm": bucket["algorithm"]}

        return {"bucket": "control", "algorithm": "default"}

    def collect_metrics(self, experiment_id, bucket_name):
        """收集实验指标"""
        return {
            "ctr": self._calc_ctr(experiment_id, bucket_name),
            "avg_watch_time": self._calc_avg_watch_time(experiment_id, bucket_name),
            "diversity_score": self._calc_diversity(experiment_id, bucket_name),
            "sample_size": self.db.count("experiment_assignments",
                experiment_id=experiment_id, bucket=bucket_name)
        }

    def check_significance(self, experiment_id):
        """统计显著性检验"""
        buckets = self.db.query(
            "SELECT * FROM experiment_buckets WHERE experiment_id = %s",
            experiment_id)

        control_metrics = self.collect_metrics(experiment_id, "control")
        results = []

        for bucket in buckets:
            if bucket["name"] == "control":
                continue
            treatment_metrics = self.collect_metrics(experiment_id, bucket["name"])

            # Z-test for CTR difference
            ctr_diff = treatment_metrics["ctr"] - control_metrics["ctr"]
            p_value = self._z_test(
                control_metrics["ctr"], treatment_metrics["ctr"],
                control_metrics["sample_size"], treatment_metrics["sample_size"])

            results.append({
                "bucket": bucket["name"],
                "ctr_lift": f"{ctr_diff / control_metrics['ctr'] * 100:.1f}%",
                "watch_time_lift": f"{(treatment_metrics['avg_watch_time'] - control_metrics['avg_watch_time']) / control_metrics['avg_watch_time'] * 100:.1f}%",
                "p_value": round(p_value, 4),
                "significant": p_value < 0.05
            })

        return results
```

## 推荐可解释性

```python
class RecommendationExplainer:
    """推荐可解释性：告诉用户为什么推荐"""

    EXPLANATION_TEMPLATES = {
        "cf_similar": "因为您看过《{similar_video}》",
        "cf_copreference": "和您品味相似的用户也在看",
        "content_similar": "与您喜欢的《{similar_video}》风格相似",
        "hot_trending": "正在热播，{view_count}人正在看",
        "new_release": "新上线内容",
        "category_interest": "因为您对{category}感兴趣",
    }

    def explain(self, user_id, video_id, recommendation_source):
        """生成推荐解释"""
        video = self.db.get_video(video_id)

        if recommendation_source["type"] == "cf":
            # 找到用户看过的最相似视频
            similar = self._find_similar_watched(user_id, video_id)
            if similar:
                return self.EXPLANATION_TEMPLATES["cf_similar"].format(
                    similar_video=similar["title"])
            return self.EXPLANATION_TEMPLATES["cf_copreference"]

        elif recommendation_source["type"] == "content":
            similar = self._find_content_similar_watched(user_id, video_id)
            return self.EXPLANATION_TEMPLATES["content_similar"].format(
                similar_video=similar["title"])

        elif recommendation_source["type"] == "hot":
            return self.EXPLANATION_TEMPLATES["hot_trending"].format(
                view_count=video["view_count_24h"])

        elif recommendation_source["type"] == "new":
            return self.EXPLANATION_TEMPLATES["new_release"]

        return self.EXPLANATION_TEMPLATES["category_interest"].format(
            category=video["category_name"])
```

## 异常场景补充

### 场景：Kafka 消费延迟导致推荐过时

```
触发：Kafka consumer lag 增长 → 用户行为未及时反映 → 推荐不更新
检测：
  1. Consumer lag > 100000 → 告警
  2. 推荐结果长时间不变 → 可能过时
处理：
  1. 增加消费者实例（自动扩容）
  2. 降级：使用定时批量更新替代实时更新
  3. 推荐缓存 TTL 缩短，强制刷新
预防：消费者自动扩容 + 降级策略 + 缓存 TTL
```

### 场景：A/B 测试桶分配不一致

```
触发：同一用户在不同请求中被分配到不同桶 → 实验结果污染
检测：
  1. 桶分配日志中同一 user_id 出现不同 bucket → 不一致
  2. 不一致率 > 0.1% → 告警
处理：
  1. 检查哈希算法是否一致
  2. 使用一致性缓存：首次分配后写入 Redis
  3. 后续分配从缓存读取
预防：首次分配缓存 + 一致性哈希 + 分配日志审计
```

## 推荐效果评估体系完整实现

```python
class RecommendationEvaluator:
    """推荐效果评估：离线 + 在线指标"""

    def evaluate_offline(self, algorithm_name, test_data):
        """离线评估"""
        metrics = {
            "precision@5": self._precision_at_k(test_data, k=5),
            "precision@10": self._precision_at_k(test_data, k=10),
            "recall@20": self._recall_at_k(test_data, k=20),
            "ndcg@10": self._ndcg_at_k(test_data, k=10),
            "coverage": self._catalog_coverage(test_data),
            "novelty": self._novelty(test_data),
            "diversity": self._diversity(test_data),
        }
        return metrics

    def _precision_at_k(self, test_data, k):
        """Precision@K：推荐的前K个中有多少用户实际观看"""
        hits = 0
        total = 0
        for user_id, recommended, actual in test_data:
            recommended_k = recommended[:k]
            hits += len(set(recommended_k) & set(actual))
            total += k
        return hits / total if total > 0 else 0

    def _ndcg_at_k(self, test_data, k):
        """NDCG@K：归一化折损累积增益"""
        ndcg_sum = 0
        for user_id, recommended, actual in test_data:
            dcg = 0
            for i, item in enumerate(recommended[:k]):
                if item in actual:
                    dcg += 1 / math.log2(i + 2)  # 位置 i 的增益
            # 理想排序
            idcg = sum(1 / math.log2(i + 2) for i in range(min(len(actual), k)))
            ndcg_sum += dcg / idcg if idcg > 0 else 0
        return ndcg_sum / len(test_data) if test_data else 0

    def _catalog_coverage(self, test_data):
        """覆盖率：推荐系统覆盖了多少比例的视频库"""
        recommended_items = set()
        total_items = self.db.count("videos")
        for _, recommended, _ in test_data:
            recommended_items.update(recommended)
        return len(recommended_items) / total_items if total_items > 0 else 0

    def _novelty(self, test_data):
        """新颖度：推荐了多少长尾内容"""
        novelty_sum = 0
        count = 0
        for _, recommended, _ in test_data:
            for item in recommended:
                popularity = self.redis.get(f"video_popularity:{item}")
                if popularity:
                    # 越不流行 → 越新颖
                    novelty_sum += -math.log2(int(popularity) / self.total_views + 1e-10)
                    count += 1
        return novelty_sum / count if count > 0 else 0

    def _diversity(self, test_data):
        """多样性：推荐列表中视频的类别差异"""
        diversity_sum = 0
        for _, recommended, _ in test_data:
            categories = [self.db.get_video(v)["category_id"]
                         for v in recommended[:10]]
            unique = len(set(categories))
            diversity_sum += unique / len(categories) if categories else 0
        return diversity_sum / len(test_data) if test_data else 0
```

## 推荐模型版本管理

```python
class RecommendationModelRegistry:
    """推荐模型版本管理"""

    def register_model(self, name, version, artifact_path, metrics):
        """注册模型版本"""
        model_id = str(uuid4())
        self.db.insert("rec_models", {
            "model_id": model_id,
            "name": name, "version": version,
            "artifact_path": artifact_path,
            "offline_metrics": json.dumps(metrics),
            "status": "staging",  # staging → canary → production
            "created_at": now()
        })
        return model_id

    def promote_to_production(self, model_id):
        """晋升模型到生产环境"""
        model = self.db.get_model(model_id)

        # 检查 canary 测试结果
        canary = self.db.query_one(
            "SELECT * FROM model_canary_tests WHERE model_id = %s "
            "AND status = 'completed'", model_id)
        if not canary or canary["ctr_lift"] < 0:
            raise PromotionFailedError("Canary 测试未通过")

        # 旧模型降级
        self.db.update("rec_models",
            {"status": "previous_production"},
            {"name": model["name"], "status": "production"})

        # 新模型上线
        self.db.update("rec_models",
            {"status": "production", "promoted_at": now()},
            {"model_id": model_id})

        # 加载新模型
        self.serving.load_model(model["artifact_path"])
```

## 异常场景补充

### 场景：推荐效果评估指标冲突

```
触发：新模型 CTR 提升 5% 但多样性下降 10% → 无法判断是否应该上线
检测：
  1. 单一指标优化导致其他指标退化
  2. CTR↑ + Diversity↓ → 典型的"信息茧房"信号
处理：
  1. 使用综合评分：0.4×CTR + 0.3×WatchTime + 0.2×Diversity + 0.1×Novelty
  2. 设置每个指标的下限（Diversity 不能低于 baseline）
  3. 违反下限 → 模型不可上线
预防：多指标综合评估 + 指标下限约束 + 信息茧房检测
```

### 场景：模型服务加载失败

```
触发：新模型文件损坏 → 加载失败 → 推荐服务不可用
检测：
  1. 模型加载异常 → 服务启动失败
  2. 推荐接口返回 500 → 严重告警
处理：
  1. 自动回滚到上一个生产模型
  2. 模型文件校验（SHA256）→ 不匹配则拒绝加载
  3. 通知模型团队修复
预防：模型文件校验 + 自动回滚 + 预加载验证
```

## 内容理解流水线完整实现

```python
class ContentUnderstandingPipeline:
    """内容理解：视频/音频/文本多模态特征提取"""

    def extract_features(self, video_id):
        """提取视频多模态特征"""
        video = self.db.get_video(video_id)

        # 1. 视觉特征（CNN 嵌入）
        visual_embedding = self._extract_visual(video["file_url"])

        # 2. 音频特征（频谱嵌入）
        audio_embedding = self._extract_audio(video["file_url"])

        # 3. 文本特征（BERT 嵌入）
        text = f"{video['title']} {video.get('description', '')} {' '.join(video.get('tags', []))}"
        text_embedding = self.bert_model.encode(text)

        # 4. 多模态融合（加权拼接）
        fused_embedding = self._fuse_embeddings(
            visual_embedding, audio_embedding, text_embedding,
            weights=[0.4, 0.2, 0.4])

        # 5. 写入 FAISS 索引
        self.faiss_index.add(video_id, fused_embedding)

        # 6. 存储到数据库
        self.db.insert("video_embeddings", {
            "video_id": video_id,
            "visual_embedding": json.dumps(visual_embedding[:64].tolist()),
            "audio_embedding": json.dumps(audio_embedding[:64].tolist()),
            "text_embedding": json.dumps(text_embedding[:64].tolist()),
            "fused_embedding": json.dumps(fused_embedding[:64].tolist()),
            "model_version": "v2.1",
            "created_at": now()
        })

        return {"video_id": video_id, "embedding_dim": len(fused_embedding)}

    def _extract_visual(self, video_url):
        """提取视觉特征：采样帧 → ResNet → 平均池化"""
        frames = self.video_processor.sample_frames(video_url, n=8)
        embeddings = []
        for frame in frames:
            emb = self.resnet_model.encode(frame)
            embeddings.append(emb)
        # 平均池化
        return sum(embeddings) / len(embeddings)

    def _extract_audio(self, video_url):
        """提取音频特征：音频 → 频谱图 → CNN"""
        spectrogram = self.audio_processor.to_spectrogram(video_url)
        return self.audio_model.encode(spectrogram)

    def _fuse_embeddings(self, visual, audio, text, weights):
        """多模态融合（加权拼接 + 降维）"""
        import numpy as np
        # 归一化
        visual = np.array(visual) / np.linalg.norm(visual)
        audio = np.array(audio) / np.linalg.norm(audio)
        text = np.array(text) / np.linalg.norm(text)
        # 加权
        fused = np.concatenate([
            visual * weights[0],
            audio * weights[1],
            text * weights[2]
        ])
        return fused / np.linalg.norm(fused)

    def find_similar(self, video_id, top_k=20):
        """查找相似视频"""
        distances, indices = self.faiss_index.search(video_id, top_k)
        similar = []
        for dist, idx in zip(distances, indices):
            similar.append({
                "video_id": self.faiss_index.get_id(idx),
                "similarity": 1 - dist
            })
        return similar
```

## 用户画像系统

```python
class UserProfileService:
    """用户画像：长期兴趣 + 短期兴趣 + 漂移检测"""

    def get_profile(self, user_id):
        """获取用户画像"""
        return {
            "long_term_interests": self._get_long_term(user_id),
            "short_term_interests": self._get_short_term(user_id),
            "interest_drift": self._detect_drift(user_id),
        }

    def _get_long_term(self, user_id):
        """长期兴趣（加权平均观看视频 embedding）"""
        # 最近 90 天观看的视频 embedding 加权平均
        watched = self.db.query(
            "SELECT v.embedding, w.watch_duration / v.duration as weight "
            "FROM user_watches w JOIN video_embeddings v ON w.video_id = v.video_id "
            "WHERE w.user_id = %s AND w.created_at > NOW() - INTERVAL 90 DAY "
            "ORDER BY w.created_at DESC LIMIT 200", user_id)

        if not watched:
            return None

        import numpy as np
        weighted_sum = np.zeros(64)
        weight_total = 0
        for w in watched:
            emb = np.array(json.loads(w["embedding"]))
            weighted_sum += emb * w["weight"]
            weight_total += w["weight"]

        return (weighted_sum / weight_total).tolist()

    def _get_short_term(self, user_id):
        """短期兴趣（最近 50 个行为，指数衰减）"""
        actions = self.db.query(
            "SELECT v.embedding, a.type FROM user_actions a "
            "JOIN video_embeddings v ON a.video_id = v.video_id "
            "WHERE a.user_id = %s ORDER BY a.created_at DESC LIMIT 50",
            user_id)

        import numpy as np
        action_weights = {"watch": 1.0, "like": 1.5, "share": 2.0, "skip": -0.5}
        weighted_sum = np.zeros(64)
        weight_total = 0

        for i, a in enumerate(actions):
            decay = math.exp(-0.05 * i)  # 指数衰减
            emb = np.array(json.loads(a["embedding"]))
            w = action_weights.get(a["type"], 0.5) * decay
            weighted_sum += emb * w
            weight_total += abs(w)

        return (weighted_sum / weight_total).tolist() if weight_total > 0 else None

    def _detect_drift(self, user_id):
        """兴趣漂移检测（长期 vs 短期余弦相似度）"""
        long_term = self._get_long_term(user_id)
        short_term = self._get_short_term(user_id)

        if not long_term or not short_term:
            return {"drift_score": None, "status": "insufficient_data"}

        import numpy as np
        similarity = np.dot(long_term, short_term) / (
            np.linalg.norm(long_term) * np.linalg.norm(short_term))

        drift_score = 1 - similarity  # 越大漂移越严重
        return {
            "drift_score": round(float(drift_score), 3),
            "status": "significant" if drift_score > 0.5 else
                     "moderate" if drift_score > 0.3 else "stable"
        }
```

## 推荐监控看板

```python
class RecommendationDashboard:
    """推荐监控看板"""

    def get_overview(self):
        """推荐系统总览"""
        return {
            "quality_metrics": self._quality_metrics(),
            "latency": self._latency_metrics(),
            "cache": self._cache_metrics(),
            "serving": self._serving_metrics(),
            "experiments": self._experiment_status(),
        }

    def _quality_metrics(self):
        """推荐质量指标"""
        return {
            "ctr_1h": round(self.redis.get("rec:ctr:1h") or 0, 4),
            "avg_watch_time_1h": round(self.redis.get("rec:watch_time:1h") or 0, 1),
            "diversity_score": round(self.redis.get("rec:diversity:1h") or 0, 3),
            "novelty_score": round(self.redis.get("rec:novelty:1h") or 0, 3),
            "coverage_rate": round(self.redis.get("rec:coverage:daily") or 0, 3),
        }

    def _latency_metrics(self):
        """延迟指标"""
        return {
            "rec_p50_ms": int(self.redis.get("rec:latency:p50") or 0),
            "rec_p99_ms": int(self.redis.get("rec:latency:p99") or 0),
            "model_inference_ms": int(self.redis.get("rec:model:latency") or 0),
            "feature_fetch_ms": int(self.redis.get("rec:feature:latency") or 0),
        }

    def _cache_metrics(self):
        """缓存指标"""
        return {
            "hit_rate": round(self.redis.get("rec:cache:hit_rate") or 0, 3),
            "rec_cache_ttl_seconds": 300,
            "feature_cache_freshness_s": int(self.redis.get("rec:feature:freshness") or 0),
        }

    def _experiment_status(self):
        """实验状态"""
        return self.db.query(
            "SELECT id, name, status, sample_size, started_at "
            "FROM ab_experiments WHERE status IN ('running', 'concluding')")
```

## 异常场景补充

### 场景：Embedding 索引损坏

```
触发：FAISS 索引文件损坏 → 相似视频检索返回空结果
检测：
  1. 相似视频检索返回 0 结果 → 索引异常
  2. 检索延迟飙升 → 索引加载问题
处理：
  1. 从数据库重建 FAISS 索引（全量 embedding）
  2. 重建期间降级到基于标签的相似推荐
  3. 重建完成后验证召回率
预防：索引定期快照 + 增量更新 + 降级推荐策略
```

### 场景：用户画像漂移导致推荐不相关

```
触发：用户兴趣突变（如从科技转向育儿）→ 长期兴趣仍偏科技 → 推荐不匹配
检测：
  1. 漂移分数 > 0.5 → 兴趣突变
  2. 近期 CTR 下降 > 30% → 推荐不相关
处理：
  1. 短期兴趣权重提升（长期 0.3 → 短期 0.7）
  2. 加快长期兴趣更新频率
  3. 触发探索性推荐（新领域内容）
预防：漂移检测 + 动态权重调整 + 兴趣探索
```

## 推荐系统故障应急完整实现

```python
class RecommendationEmergencyService:
    """推荐系统故障应急：降级 + 兜底 + 恢复"""

    DEGRADATION_LEVELS = [
        {"level": 0, "name": "normal", "description": "正常服务"},
        {"level": 1, "name": "reduced_freshness", "description": "降低实时性（缓存 TTL 从 5 分钟延长到 30 分钟）"},
        {"level": 2, "name": "offline_model", "description": "使用离线预计算结果（无实时特征）"},
        {"level": 3, "name": "popular_fallback", "description": "降级到热门列表（无个性化）"},
        {"level": 4, "name": "static_fallback", "description": "静态兜底页面（分类导航）"},
    ]

    def get_current_level(self):
        """获取当前降级级别"""
        level = int(self.redis.get("rec:degradation_level") or 0)
        return self.DEGRADATION_LEVELS[level]

    def escalate_degradation(self, reason):
        """升级降级级别"""
        current = int(self.redis.get("rec:degradation_level") or 0)
        if current >= 4:
            return {"level": 4, "message": "已是最高降级"}

        new_level = current + 1
        self.redis.set("rec:degradation_level", new_level)

        self.db.insert("rec_incidents", {
            "incident_id": str(uuid4()),
            "action": "degradation_escalate",
            "from_level": current,
            "to_level": new_level,
            "reason": reason,
            "timestamp": now()
        })

        self.alert(f"推荐系统降级至 Level {new_level}: "
                   f"{self.DEGRADATION_LEVELS[new_level]['description']}")

        return {"level": new_level,
                "description": self.DEGRADATION_LEVELS[new_level]["description"]}

    def recover_degradation(self):
        """恢复降级级别"""
        current = int(self.redis.get("rec:degradation_level") or 0)
        if current == 0:
            return {"level": 0, "message": "已正常运行"}

        # 逐步恢复（每次降一级，验证 5 分钟）
        new_level = current - 1
        self.redis.set("rec:degradation_level", new_level)

        # 验证恢复后系统稳定
        self.scheduler.schedule(
            run_date=now() + timedelta(minutes=5),
            task=self._verify_recovery,
            args={"level": new_level})

        return {"level": new_level}

    def _verify_recovery(self, level):
        """验证恢复后系统稳定"""
        error_rate = float(self.redis.get("rec:error_rate:5m") or 0)
        latency_p99 = int(self.redis.get("rec:latency:p99:5m") or 0)

        if error_rate > 0.01 or latency_p99 > 500:
            # 不稳定 → 回退
            self.redis.set("rec:degradation_level", level + 1)
            self.alert(f"恢复验证失败，回退至 Level {level + 1}")
        elif level > 0:
            # 稳定 → 继续恢复
            self.recover_degradation()
```

## 推荐特征管理

```python
class RecommendationFeatureStore:
    """推荐特征管理：特征注册 + 在线/离线一致性 + 监控"""

    def register_feature(self, name, feature_type, source, description,
                         default_value=None, freshness_sla_seconds=300):
        """注册特征"""
        self.db.insert("rec_features", {
            "name": name,
            "type": feature_type,  # numerical / categorical / embedding
            "source": source,      # online / offline / streaming
            "description": description,
            "default_value": default_value,
            "freshness_sla_seconds": freshness_sla_seconds,
            "registered_at": now()
        })

    def get_feature_vector(self, user_id, feature_names):
        """获取特征向量"""
        features = {}
        for name in feature_names:
            value = self.redis.hget(f"rec_feature:{user_id}", name)
            if value is not None:
                features[name] = json.loads(value)
            else:
                # 使用默认值
                feature_def = self.db.query_one(
                    "SELECT default_value FROM rec_features WHERE name = %s", name)
                features[name] = feature_def["default_value"] if feature_def else None

        return features

    def monitor_feature_freshness(self):
        """监控特征新鲜度"""
        features = self.db.query(
            "SELECT name, freshness_sla_seconds FROM rec_features")
        stale_features = []

        for f in features:
            last_update = self.redis.get(f"rec_feature_updated:{f['name']}")
            if last_update:
                age = (now() - datetime.fromisoformat(last_update)).total_seconds()
                if age > f["freshness_sla_seconds"]:
                    stale_features.append({
                        "feature": f["name"],
                        "age_seconds": int(age),
                        "sla_seconds": f["freshness_sla_seconds"]
                    })
            else:
                stale_features.append({
                    "feature": f["name"],
                    "age_seconds": None,
                    "sla_seconds": f["freshness_sla_seconds"]
                })

        return {"stale_count": len(stale_features),
                "stale_features": stale_features}
```

## 异常场景补充

### 场景：推荐服务完全不可用

```
触发：推荐服务集群全部宕机 → 无法返回推荐结果
检测：
  1. 健康检查全部失败 → 服务不可用
  2. 客户端超时 → 严重告警
处理：
  1. 客户端降级：使用本地缓存的推荐列表
  2. 最终降级：显示分类导航页
  3. 服务恢复后逐步回切
预防：客户端本地缓存 + 多级降级 + 快速重启
```

### 场景：特征新鲜度严重滞后

```
触发：实时特征管道故障 → 特征数小时未更新 → 推荐结果过时
检测：
  1. 特征新鲜度监控 > SLA → 告警
  2. 推荐结果重复或不相关 → 特征过时
处理：
  1. 降级：使用离线特征替代实时特征
  2. 修复特征管道
  3. 管道恢复后刷新所有特征缓存
预防：特征新鲜度监控 + 离线特征兜底 + 自动降级
```

## 推荐冷启动解决方案完整实现

```python
class RecommendationColdStartService:
    """推荐冷启动：新用户 + 新视频 + 探索策略"""

    def handle_new_user(self, user_id, registration_info):
        """新用户冷启动"""
        # 1. 基于注册信息预测兴趣
        predicted_interests = self._predict_from_registration(registration_info)

        # 2. 从相似注册画像的用户中获取兴趣
        similar_users = self._find_similar_registration_profiles(
            registration_info, top_k=50)
        if similar_users:
            collab_interests = self._aggregate_interests(similar_users)
            predicted_interests = self._merge_interests(
                predicted_interests, collab_interests)

        # 3. 初始化用户画像
        self.db.insert("user_cold_start_profiles", {
            "user_id": user_id,
            "predicted_interests": json.dumps(predicted_interests),
            "phase": "exploration",  # exploration → warm → normal
            "exploration_quota": 0.3,  # 30% 探索内容
            "created_at": now()
        })

        # 4. 生成首个推荐列表（热门 + 多样化 + 预测兴趣匹配）
        feed = self._generate_cold_start_feed(predicted_interests)

        return {"interests": predicted_interests, "feed_size": len(feed)}

    def handle_new_video(self, video_id):
        """新视频冷启动"""
        video = self.db.get_video(video_id)

        # 1. 基于内容特征预测受众
        content_features = self._extract_content_features(video)
        target_audience = self.content_model.predict_audience(content_features)

        # 2. 分配曝光配额
        exploration_budget = self._allocate_exploration_budget(video_id)

        # 3. Thompson Sampling 探索-利用
        # 初始化 Beta 分布参数 (α=1, β=1 → 均匀先验)
        self.redis.set(f"ts_alpha:{video_id}", "1")
        self.redis.set(f"ts_beta:{video_id}", "1")

        # 4. 预计算 embedding 用于相似推荐
        self.embedding_service.compute_and_store(video_id)

        return {"target_audience": target_audience,
                "exploration_budget": exploration_budget}

    def thompson_sample(self, candidate_videos, user_id):
        """Thompson Sampling 选择探索视频"""
        scores = []
        for vid in candidate_videos:
            alpha = float(self.redis.get(f"ts_alpha:{vid}") or 1)
            beta = float(self.redis.get(f"ts_beta:{vid}") or 1)
            # 从 Beta 分布采样
            sample = random.betavariate(alpha, beta)
            scores.append((vid, sample))

        scores.sort(key=lambda x: x[1], reverse=True)
        return [vid for vid, score in scores]

    def update_thompson_params(self, video_id, clicked):
        """更新 Thompson Sampling 参数"""
        if clicked:
            self.redis.incr(f"ts_alpha:{video_id}")
        else:
            self.redis.incr(f"ts_beta:{video_id}")

    def _predict_from_registration(self, info):
        """从注册信息预测兴趣"""
        interests = {}
        age = info.get("age", 25)
        if age < 18:
            interests.update({"游戏": 0.8, "动画": 0.7, "音乐": 0.6})
        elif age < 30:
            interests.update({"科技": 0.7, "娱乐": 0.6, "美食": 0.5})
        elif age < 50:
            interests.update({"新闻": 0.7, "生活": 0.6, "教育": 0.5})
        else:
            interests.update({"健康": 0.7, "新闻": 0.6, "戏曲": 0.5})

        if info.get("gender") == "M":
            interests.setdefault("体育", 0.6)
        elif info.get("gender") == "F":
            interests.setdefault("美妆", 0.6)

        return interests
```

## 推荐公平性与偏见缓解

```python
class RecommendationFairnessService:
    """推荐公平性：偏见检测 + 多样性 + 过滤气泡"""

    def audit_fairness(self, period_days=7):
        """公平性审计"""
        return {
            "demographic_parity": self._check_demographic_parity(period_days),
            "creator_exposure": self._check_creator_exposure(period_days),
            "filter_bubble": self._detect_filter_bubbles(period_days),
            "recommendations": self._generate_mitigation_recommendations(),
        }

    def _check_demographic_parity(self, days):
        """人口统计平权检查"""
        # 按创作者群体统计曝光量
        exposure = self.db.query(
            "SELECT creator_demographic, COUNT(*) as impressions "
            "FROM recommendation_impressions "
            "WHERE timestamp > NOW() - INTERVAL %s DAY "
            "GROUP BY creator_demographic", days)

        total = sum(e["impressions"] for e in exposure)
        if total == 0:
            return {"status": "no_data"}

        # 计算各群体曝光占比 vs 用户占比
        ratios = []
        for e in exposure:
            demo = e["creator_demographic"]
            exposure_pct = e["impressions"] / total
            population_pct = self._get_population_pct(demo)
            ratio = exposure_pct / max(population_pct, 0.01)
            ratios.append({
                "demographic": demo,
                "exposure_pct": round(exposure_pct, 3),
                "population_pct": round(population_pct, 3),
                "ratio": round(ratio, 2),
                "flagged": ratio < 0.5 or ratio > 2.0
            })

        return {"ratios": ratios,
                "fair": not any(r["flagged"] for r in ratios)}

    def _detect_filter_bubbles(self, days):
        """检测过滤气泡（用户兴趣窄化）"""
        # 比较用户本周 vs 上月的兴趣分布熵
        users = self.db.query(
            "SELECT user_id FROM users WHERE active = 1 LIMIT 1000")

        narrowing_count = 0
        for user in users:
            recent_entropy = self._interest_entropy(user["user_id"], 7)
            baseline_entropy = self._interest_entropy(user["user_id"], 30)

            if baseline_entropy > 0 and recent_entropy < baseline_entropy * 0.7:
                narrowing_count += 1

        narrowing_pct = narrowing_count / max(len(users), 1)
        return {"narrowing_pct": round(narrowing_pct, 3),
                "status": "concerning" if narrowing_pct > 0.3 else "ok"}

    def _interest_entropy(self, user_id, days):
        """计算用户兴趣分布熵"""
        distribution = self.db.query(
            "SELECT category, COUNT(*) as count "
            "FROM user_actions WHERE user_id = %s "
            "AND created_at > NOW() - INTERVAL %s DAY "
            "GROUP BY category", user_id, days)

        if not distribution:
            return 0

        total = sum(d["count"] for d in distribution)
        probs = [d["count"] / total for d in distribution]
        return -sum(p * math.log2(p) for p in probs if p > 0)

    def enforce_diversity(self, feed_items, max_same_creator=3):
        """多样性强制：同一创作者最多 N 条"""
        creator_count = {}
        filtered = []
        for item in feed_items:
            creator = item.get("creator_id")
            creator_count[creator] = creator_count.get(creator, 0) + 1
            if creator_count[creator] <= max_same_creator:
                filtered.append(item)
        return filtered
```

## 异常场景补充

### 场景：冷启动探索导致用户流失

```
触发：新用户看到太多不相关的探索内容 → 误以为推荐差 → 流失
检测：
  1. 新用户次日留存率下降 → 冷启动问题
  2. 新用户前 10 次推荐 CTR < 5% → 推荐不匹配
处理：
  1. 减少探索配额（30% → 15%）
  2. 增加热门内容权重
  3. 快速收集反馈（"不感兴趣"按钮）
预防：探索配额动态调整 + 快速反馈收集 + 热门兜底
```

### 场景：公平性约束过度降低推荐质量

```
触发：强制多样性 + 公平性约束 → 推荐相关性下降 → CTR 降低 20%
检测：
  1. A/B 测试：公平性组 vs 对照组 CTR 差异
  2. 公平性组 CTR 下降 > 15% → 过度约束
处理：
  1. 放宽多样性约束（同创作者 3→5）
  2. 公平性加权而非硬约束
  3. 寻找公平性与质量的帕累托最优点
预防：软约束替代硬约束 + A/B 测试 + 帕累托优化
```

## 推荐系统 A/B 测试框架完整实现

```python
class RecommendationABTestService:
    """推荐 A/B 测试：实验管理 + 指标收集 + 统计分析"""

    def create_experiment(self, name, control_model, treatment_model,
                         traffic_pct=50, duration_days=7):
        """创建推荐实验"""
        experiment_id = str(uuid4())
        self.db.insert("rec_experiments", {
            "experiment_id": experiment_id,
            "name": name,
            "control_model": control_model,
            "treatment_model": treatment_model,
            "traffic_pct": traffic_pct,
            "duration_days": duration_days,
            "status": "running",
            "started_at": now()
        })

        # 配置流量分割
        self.redis.set(f"rec_exp:{experiment_id}:traffic", traffic_pct)

        return {"experiment_id": experiment_id}

    def assign_variant(self, experiment_id, user_id):
        """分配实验变体（一致性哈希）"""
        # 检查是否已分配
        cached = self.redis.get(f"rec_exp:{experiment_id}:user:{user_id}")
        if cached:
            return cached

        # 基于用户 ID 的确定性分配
        hash_val = int(hashlib.md5(
            f"{experiment_id}:{user_id}".encode()).hexdigest(), 16) % 100
        traffic_pct = int(self.redis.get(f"rec_exp:{experiment_id}:traffic") or 50)

        variant = "treatment" if hash_val < traffic_pct else "control"
        self.redis.setex(f"rec_exp:{experiment_id}:user:{user_id}", 86400, variant)

        return variant

    def collect_metrics(self, experiment_id):
        """收集实验指标"""
        exp = self.db.get_experiment(experiment_id)

        metrics = {}
        for variant in ["control", "treatment"]:
            model = exp["control_model"] if variant == "control" else exp["treatment_model"]

            # 核心指标
            metrics[variant] = {
                "ctr": self._get_ctr(experiment_id, variant),
                "watch_time_avg": self._get_avg_watch_time(experiment_id, variant),
                "diversity_score": self._get_diversity(experiment_id, variant),
                "freshness_score": self._get_freshness(experiment_id, variant),
                "retention_d1": self._get_retention(experiment_id, variant, 1),
                "retention_d7": self._get_retention(experiment_id, variant, 7),
                "sample_size": self._get_sample_size(experiment_id, variant),
            }

        # 统计检验
        significance = self._test_significance(metrics)

        return {"experiment_id": experiment_id, "metrics": metrics,
                "significance": significance}

    def _test_significance(self, metrics):
        """统计显著性检验"""
        from scipy import stats

        results = {}
        for metric_name in ["ctr", "watch_time_avg", "retention_d1"]:
            control = metrics["control"].get(metric_name, 0)
            treatment = metrics["treatment"].get(metric_name, 0)
            n_c = metrics["control"]["sample_size"]
            n_t = metrics["treatment"]["sample_size"]

            if n_c < 1000 or n_t < 1000:
                results[metric_name] = {"status": "insufficient_data"}
                continue

            # Z 检验
            se = math.sqrt(control * (1 - control) / n_c +
                          treatment * (1 - treatment) / n_t)
            if se == 0:
                continue
            z = (treatment - control) / se
            p_value = 2 * (1 - stats.norm.cdf(abs(z)))

            lift = (treatment - control) / abs(control) * 100 if control != 0 else None

            results[metric_name] = {
                "control": round(control, 4),
                "treatment": round(treatment, 4),
                "lift_pct": round(lift, 2) if lift else None,
                "p_value": round(p_value, 4),
                "significant": p_value < 0.05
            }

        return results
```

## 异常场景补充

### 场景：A/B 实验样本污染

```
触发：用户清除 Cookie 后重新分配 → 跨变体污染 → 实验结果失真
检测：
  1. 同一用户 ID 出现在两个变体 → 污染
  2. 变体样本量差异 > 预期 → 可能污染
处理：
  1. 基于用户 ID（而非 Cookie）分配
  2. 清除污染数据
  3. 延长实验周期
预防：基于用户 ID 分配 + SRM 检测 + 污染数据排除
```

### 场景：实验期间模型版本更新

```
触发：实验运行中 → treatment 模型被更新 → 实验结果不可比
检测：
  1. 模型版本变更但实验仍在运行 → 配置变更
  2. 实验指标突然变化 → 模型更新
处理：
  1. 锁定实验模型版本（运行中不允许更新）
  2. 如果已更新 → 重启实验
预防：实验模型版本锁定 + 变更审批 + 运行中禁止更新
```

## 推荐系统特征工程完整实现

```python
class FeatureEngineeringService:
    """推荐特征工程：用户特征 + 物品特征 + 交叉特征"""

    def build_user_features(self, user_id):
        """构建用户特征向量"""
        user = self.db.get_user(user_id)
        recent_actions = self.db.query(
            "SELECT * FROM user_actions WHERE user_id = %s "
            "AND created_at > NOW() - INTERVAL 30 DAY "
            "ORDER BY created_at DESC LIMIT 500", user_id)

        features = {}

        # 1. 统计特征
        features["total_watch_hours"] = sum(
            a.get("watch_duration_s", 0) for a in recent_actions) / 3600
        features["avg_watch_duration_s"] = statistics.mean(
            [a.get("watch_duration_s", 0) for a in recent_actions]) if recent_actions else 0
        features["like_rate"] = sum(1 for a in recent_actions if a["action"] == "like") / max(len(recent_actions), 1)
        features["share_rate"] = sum(1 for a in recent_actions if a["action"] == "share") / max(len(recent_actions), 1)
        features["completion_rate"] = sum(1 for a in recent_actions
            if a.get("watch_duration_s", 0) > a.get("video_duration_s", 0) * 0.9) / max(len(recent_actions), 1)

        # 2. 兴趣分布（类别占比）
        category_counts = {}
        for a in recent_actions:
            cat = a.get("video_category")
            if cat:
                category_counts[cat] = category_counts.get(cat, 0) + 1
        total = sum(category_counts.values()) or 1
        features["interest_distribution"] = {k: round(v/total, 3) for k, v in category_counts.items()}

        # 3. 活跃时间特征
        hour_counts = [0] * 24
        for a in recent_actions:
            hour_counts[a["created_at"].hour] += 1
        peak_hour = hour_counts.index(max(hour_counts))
        features["peak_active_hour"] = peak_hour
        features["active_hours_spread"] = len([h for h in hour_counts if h > 0])

        # 4. 探索性（观看新类别的比例）
        seen_categories = set()
        new_category_count = 0
        for a in sorted(recent_actions, key=lambda x: x["created_at"]):
            cat = a.get("video_category")
            if cat and cat not in seen_categories:
                new_category_count += 1
                seen_categories.add(cat)
        features["exploration_rate"] = new_category_count / max(len(seen_categories), 1)

        return features

    def build_video_features(self, video_id):
        """构建视频特征向量"""
        video = self.db.get_video(video_id)

        features = {}

        # 1. 内容特征
        features["duration_seconds"] = video.get("duration_s", 0)
        features["has_subtitles"] = 1 if video.get("subtitle_urls") else 0
        features["resolution"] = video.get("width", 0) * video.get("height", 0)

        # 2. 互动特征
        features["total_views"] = video.get("view_count", 0)
        features["like_count"] = video.get("like_count", 0)
        features["share_count"] = video.get("share_count", 0)
        features["comment_count"] = video.get("comment_count", 0)
        features["like_view_ratio"] = features["like_count"] / max(features["total_views"], 1)

        # 3. 时效特征
        hours_since_upload = (now() - video["created_at"]).total_seconds() / 3600
        features["hours_since_upload"] = hours_since_upload
        features["recency_score"] = math.exp(-0.05 * hours_since_upload)

        # 4. 人群特征
        viewer_demographics = self.db.query(
            "SELECT age_group, gender, COUNT(*) as count "
            "FROM video_viewers WHERE video_id = %s "
            "GROUP BY age_group, gender", video_id)
        features["viewer_demographics"] = [
            {"age_group": d["age_group"], "gender": d["gender"],
             "pct": d["count"] / max(sum(d["count"] for d in viewer_demographics), 1)}
            for d in viewer_demographics
        ]

        return features

    def build_cross_features(self, user_features, video_features):
        """构建交叉特征（用户-视频匹配度）"""
        cross = {}

        # 兴趣匹配度
        user_interests = user_features.get("interest_distribution", {})
        video_category = video_features.get("category")
        cross["interest_match"] = user_interests.get(video_category, 0)

        # 完播率预测（基于用户历史完播率 × 视频时长）
        avg_completion = user_features.get("completion_rate", 0.5)
        video_duration = video_features.get("duration_seconds", 60)
        if video_duration < 60:
            cross["predicted_completion"] = min(1.0, avg_completion * 1.2)
        elif video_duration < 300:
            cross["predicted_completion"] = avg_completion
        else:
            cross["predicted_completion"] = avg_completion * 0.8

        # 时段匹配（用户活跃时段 vs 视频发布时段）
        user_peak = user_features.get("peak_active_hour", 12)
        cross["time_match"] = 1.0 if abs(user_peak - now().hour) < 3 else 0.5

        return cross
```

## 异常场景补充

### 场景：特征工程延迟导致推荐过时

```
触发：用户特征更新延迟 → 推荐仍基于 1 小时前的兴趣 → 不相关
检测：
  1. 用户刚看完类别 A 但仍推荐类别 B → 特征过时
  2. 特征计算延迟 > 30 分钟 → 过时
处理：
  1. 实时特征流（Kafka + Flink 在线计算）
  2. 关键特征（当前兴趣）实时更新
  3. 非关键特征（人口统计）批量更新
预防：实时 + 批量双路特征 + 关键特征优先
```

### 场景：新视频缺乏互动特征

```
触发：新视频无播放/点赞数据 → 互动特征全为 0 → 无法推荐
检测：
  1. 视频特征中互动指标全为 0 → 新视频
  2. 新视频曝光率极低 → 冷启动问题
处理：
  1. 使用内容特征替代互动特征
  2. 新视频分配探索配额
  3. 基于相似视频的互动数据推断
预防：内容特征为主 + 探索配额 + 相似视频迁移
```

## 推荐系统实时特征计算完整实现

```python
class RealtimeFeatureService:
    """实时特征计算：滑动窗口 + 事件流 + 特征向量"""

    WINDOW_SIZES = {
        "last_5min": 300,
        "last_1hour": 3600,
        "last_24hour": 86400,
    }

    def update_realtime_features(self, user_id, action_event):
        """更新用户实时特征（每个事件触发）"""
        event_type = action_event["action"]
        video_id = action_event.get("video_id")
        timestamp = action_event.get("timestamp", now())

        # 1. 写入事件流
        self.redis.zadd(f"rt_events:{user_id}",
            {json.dumps({"action": event_type, "video_id": video_id,
                        "timestamp": timestamp.isoformat()}): timestamp.timestamp()})

        # 2. 设置过期（保留 24 小时）
        self.redis.expire(f"rt_events:{user_id}", 86400)

        # 3. 更新计数器
        self.redis.hincrby(f"rt_counts:{user_id}:{event_type}",
            self._time_bucket(timestamp), 1)

        # 4. 更新类别偏好（实时）
        if video_id:
            video = self.db.get_video(video_id)
            if video:
                category = video.get("category")
                if category:
                    self.redis.zincrby(f"rt_categories:{user_id}", 1, category)

        # 5. 更新会话特征
        self._update_session_features(user_id, action_event)

    def get_realtime_feature_vector(self, user_id):
        """获取用户实时特征向量"""
        features = {}

        # 1. 各窗口行为计数
        for window_name, window_seconds in self.WINDOW_SIZES.items():
            cutoff = now() - timedelta(seconds=window_seconds)
            events = self.redis.zrangebyscore(
                f"rt_events:{user_id}", cutoff.timestamp(), "+inf")
            events = [json.loads(e) for e in events]

            features[f"{window_name}_view_count"] = sum(1 for e in events if e["action"] == "view")
            features[f"{window_name}_like_count"] = sum(1 for e in events if e["action"] == "like")
            features[f"{window_name}_share_count"] = sum(1 for e in events if e["action"] == "share")
            features[f"{window_name}_comment_count"] = sum(1 for e in events if e["action"] == "comment")
            features[f"{window_name}_total_actions"] = len(events)

        # 2. 实时类别偏好
        categories = self.redis.zrevrange(f"rt_categories:{user_id}", 0, 9, withscores=True)
        total_category_actions = sum(score for _, score in categories) or 1
        for cat, score in categories:
            features[f"rt_category_{cat}_pct"] = round(score / total_category_actions, 3)

        # 3. 会话特征
        session = self.redis.hgetall(f"rt_session:{user_id}")
        if session:
            features["session_duration_minutes"] = round(float(session.get(b"duration", 0)) / 60, 1)
            features["session_video_count"] = int(session.get(b"video_count", 0))
            features["session_avg_watch_pct"] = float(session.get(b"avg_watch_pct", 0))

        # 4. 趋势特征（最近 5 分钟 vs 最近 1 小时）
        recent_rate = features.get("last_5min_total_actions", 0) / max(self.WINDOW_SIZES["last_5min"], 1)
        baseline_rate = features.get("last_1hour_total_actions", 0) / max(self.WINDOW_SIZES["last_1hour"], 1)
        features["activity_trend"] = round(recent_rate / max(baseline_rate, 0.001), 2)

        return features

    def _update_session_features(self, user_id, event):
        """更新会话特征"""
        session_key = f"rt_session:{user_id}"

        # 检查是否是新会话（30 分钟无活动 → 新会话）
        last_active = self.redis.get(f"{session_key}:last_active")
        if last_active:
            idle_minutes = (now() - datetime.fromisoformat(last_active.decode())).total_seconds() / 60
            if idle_minutes > 30:
                # 新会话 → 重置
                self.redis.delete(session_key)

        # 更新会话数据
        session_start = self.redis.hget(session_key, "start_time")
        if not session_start:
            self.redis.hset(session_key, "start_time", now().isoformat())
            self.redis.hset(session_key, "video_count", 0)
            self.redis.hset(session_key, "total_watch_pct", 0)

        if event["action"] == "view":
            self.redis.hincrby(session_key, "video_count", 1)
            watch_pct = event.get("watch_percentage", 0)
            self.redis.hincrbyfloat(session_key, "total_watch_pct", watch_pct)

        # 计算平均观看比例
        video_count = int(self.redis.hget(session_key, "video_count") or 0)
        total_watch = float(self.redis.hget(session_key, "total_watch_pct") or 0)
        if video_count > 0:
            self.redis.hset(session_key, "avg_watch_pct", round(total_watch / video_count, 3))

        # 更新持续时间
        start = self.redis.hget(session_key, "start_time")
        if start:
            duration = (now() - datetime.fromisoformat(start.decode())).total_seconds()
            self.redis.hset(session_key, "duration", round(duration, 0))

        self.redis.setex(f"{session_key}:last_active", 1800, now().isoformat())
        self.redis.expire(session_key, 7200)

    def _time_bucket(self, timestamp):
        """时间分桶（按小时）"""
        return timestamp.strftime("%Y%m%d%H")
```

## 异常场景补充

### 场景：实时特征计算延迟

```
触发：Kafka 消费积压 → 事件延迟 5 分钟 → 特征向量基于旧数据 → 推荐不准
检测：
  1. Kafka consumer lag > 10000 → 积压
  2. 特征向量时间戳与当前时间差 > 5 分钟 → 延迟
处理：
  1. 增加 consumer 实例
  2. 降级到批量特征（实时特征不可用时用离线特征）
  3. 关键特征优先计算（类别偏好 > 行为计数）
预防：consumer 自动扩容 + 降级策略 + 优先级队列
```

### 场景：特征漂移导致推荐质量下降

```
触发：用户兴趣快速变化 → 实时特征窗口太长 → 特征滞后 → 推荐不匹配
检测：
  1. 推荐点击率持续下降 → 可能特征漂移
  2. 用户类别偏好 5 分钟窗口与 1 小时窗口差异大 → 兴趣变化快
处理：
  1. 缩短特征窗口（1 小时 → 15 分钟）
  2. 增加衰减因子（近期行为权重更高）
  3. 检测到漂移 → 加大实时特征权重
预防：动态窗口 + 衰减因子 + 漂移检测
```

## 推荐系统实时特征计算完整实现

```python
import time
import json
import logging
import threading
import hashlib
from dataclasses import dataclass, field
from typing import List, Optional, Dict, Tuple, Callable, Any
from enum import Enum
from collections import defaultdict, deque
from datetime import datetime, timedelta
from abc import ABC, abstractmethod

logger = logging.getLogger(__name__)


class FeatureComputeError(Exception):
    """特征计算异常"""
    pass


class FeatureDriftError(Exception):
    """特征漂移异常"""
    pass


class KafkaConsumerError(Exception):
    """Kafka消费异常"""
    pass


class EventType(Enum):
    """用户行为事件类型"""
    VIEW = "view"
    CLICK = "click"
    LIKE = "like"
    SHARE = "share"
    DISLIKE = "dislike"
    SEARCH = "search"
    PLAY = "play"
    PAUSE = "pause"
    COMPLETE = "complete"
    SKIP = "skip"


@dataclass
class UserEvent:
    """用户行为事件"""
    event_id: str
    user_id: str
    event_type: EventType
    video_id: str
    video_category: str
    video_tags: List[str]
    watch_duration_seconds: float
    timestamp: datetime
    device_type: str = ""
    position: int = -1  # 推荐列表中的位置

    def to_dict(self) -> Dict:
        return {
            "event_id": self.event_id,
            "user_id": self.user_id,
            "event_type": self.event_type.value,
            "video_id": self.video_id,
            "video_category": self.video_category,
            "video_tags": self.video_tags,
            "watch_duration_seconds": self.watch_duration_seconds,
            "timestamp": self.timestamp.isoformat(),
            "device_type": self.device_type,
            "position": self.position,
        }


@dataclass
class FeatureVector:
    """用户特征向量"""
    user_id: str
    version: int
    computed_at: datetime

    # 分类偏好特征(滑动窗口内各类别的权重)
    category_weights: Dict[str, float] = field(default_factory=dict)

    # 标签偏好特征
    tag_weights: Dict[str, float] = field(default_factory=dict)

    # 行为频率特征
    view_count: int = 0
    click_count: int = 0
    like_count: int = 0
    share_count: int = 0
    skip_count: int = 0
    complete_count: int = 0

    # 时长特征
    avg_watch_duration: float = 0.0
    total_watch_duration: float = 0.0
    completion_rate: float = 0.0  # 完播率

    # 交互深度特征
    click_through_rate: float = 0.0  # 点击率(点击/曝光)
    engagement_score: float = 0.0    # 综合互动分

    # 时段偏好特征(24小时各时段活跃度)
    hourly_activity: Dict[int, float] = field(default_factory=dict)

    # 位置偏好特征
    avg_click_position: float = 0.0
    position_bias: float = 0.0

    # 多样性特征
    category_entropy: float = 0.0  # 分类信息熵,越低说明兴趣越集中
    tag_diversity: float = 0.0

    # 漂移检测
    drift_score: float = 0.0  # 与上一版本特征的差异度

    def to_ranking_features(self) -> Dict[str, float]:
        """转换为排序模型可用的特征字典"""
        features = {
            "view_count": float(self.view_count),
            "click_count": float(self.click_count),
            "like_count": float(self.like_count),
            "share_count": float(self.share_count),
            "skip_count": float(self.skip_count),
            "complete_count": float(self.complete_count),
            "avg_watch_duration": self.avg_watch_duration,
            "total_watch_duration": self.total_watch_duration,
            "completion_rate": self.completion_rate,
            "click_through_rate": self.click_through_rate,
            "engagement_score": self.engagement_score,
            "avg_click_position": self.avg_click_position,
            "position_bias": self.position_bias,
            "category_entropy": self.category_entropy,
            "tag_diversity": self.tag_diversity,
            "drift_score": self.drift_score,
        }
        # 展开分类权重为Top-K特征
        top_categories = sorted(
            self.category_weights.items(), key=lambda x: -x[1]
        )[:10]
        for i, (cat, weight) in enumerate(top_categories):
            features[f"top_cat_{i}_weight"] = weight

        # 展开标签权重为Top-K特征
        top_tags = sorted(
            self.tag_weights.items(), key=lambda x: -x[1]
        )[:20]
        for i, (tag, weight) in enumerate(top_tags):
            features[f"top_tag_{i}_weight"] = weight

        # 展开时段活跃度
        for hour, activity in self.hourly_activity.items():
            features[f"hour_{hour}_activity"] = activity

        return features


class EventConsumer(ABC):
    """事件消费接口"""

    @abstractmethod
    def consume(self, max_records: int = 100) -> List[UserEvent]:
        pass

    @abstractmethod
    def commit(self):
        pass


class KafkaEventConsumer(EventConsumer):
    """Kafka事件消费者"""

    def __init__(self, bootstrap_servers: str, topic: str, group_id: str):
        self._bootstrap_servers = bootstrap_servers
        self._topic = topic
        self._group_id = group_id
        self._consumer = None
        self._buffer: deque = deque(maxlen=10000)
        self._connected = False
        self._consecutive_errors = 0
        self._max_consecutive_errors = 10

    def connect(self):
        """建立Kafka连接(模拟)"""
        try:
            logger.info(
                f"连接Kafka: servers={self._bootstrap_servers}, "
                f"topic={self._topic}, group={self._group_id}"
            )
            self._connected = True
            self._consecutive_errors = 0
        except Exception as e:
            self._consecutive_errors += 1
            raise KafkaConsumerError(
                f"Kafka连接失败(连续{self._consecutive_errors}次): {e}"
            )

    def consume(self, max_records: int = 100) -> List[UserEvent]:
        """从Kafka消费事件"""
        if not self._connected:
            self.connect()

        events = []
        try:
            while len(events) < max_records and self._buffer:
                raw = self._buffer.popleft()
                event = self._deserialize_event(raw)
                if event:
                    events.append(event)
            self._consecutive_errors = 0
        except Exception as e:
            self._consecutive_errors += 1
            if self._consecutive_errors >= self._max_consecutive_errors:
                logger.error(
                    f"Kafka消费连续失败{self._consecutive_errors}次,触发熔断"
                )
                self._connected = False
            raise KafkaConsumerError(f"消费事件失败: {e}")

        return events

    def commit(self):
        if self._connected:
            logger.debug("Kafka offset已提交")

    def _deserialize_event(self, raw: Dict) -> Optional[UserEvent]:
        try:
            return UserEvent(
                event_id=raw.get("event_id", ""),
                user_id=raw["user_id"],
                event_type=EventType(raw["event_type"]),
                video_id=raw["video_id"],
                video_category=raw.get("video_category", "unknown"),
                video_tags=raw.get("video_tags", []),
                watch_duration_seconds=raw.get("watch_duration_seconds", 0.0),
                timestamp=datetime.fromisoformat(raw["timestamp"]),
                device_type=raw.get("device_type", ""),
                position=raw.get("position", -1),
            )
        except (KeyError, ValueError) as e:
            logger.warning(f"事件反序列化失败: {e}, raw={raw}")
            return None

    def inject_test_events(self, events: List[UserEvent]):
        """注入测试事件(用于测试和调试)"""
        for e in events:
            self._buffer.append(e.to_dict())


class SlidingWindowFeatureAggregator:
    """滑动窗口特征聚合器"""

    def __init__(self, window_sizes: Dict[str, timedelta] = None):
        self._window_sizes = window_sizes or {
            "short": timedelta(minutes=5),
            "medium": timedelta(hours=1),
            "long": timedelta(hours=24),
        }
        self._windows: Dict[str, deque] = {
            name: deque(maxlen=5000) for name in self._window_sizes
        }

    def add_event(self, event: UserEvent):
        """将事件加入所有窗口"""
        for name, window in self._windows.items():
            window.append(event)

    def purge_expired(self, now: datetime):
        """清除窗口中过期的事件"""
        for name, window in self._windows.items():
            cutoff = now - self._window_sizes[name]
            while window and window[0].timestamp < cutoff:
                window.popleft()

    def get_events(self, window_name: str) -> List[UserEvent]:
        """获取指定窗口内的所有事件"""
        return list(self._windows.get(window_name, []))

    def get_window_size(self, window_name: str) -> timedelta:
        return self._window_sizes.get(window_name, timedelta(hours=1))


class RealtimeFeatureService:
    """
    推荐系统实时特征计算服务

    从Kafka消费用户行为事件,在滑动窗口内计算实时特征向量,
    供排序模型使用。支持多粒度时间窗口、特征漂移检测、
    自动降级等机制。
    """

    FEATURE_VERSION = 3
    DRIFT_THRESHOLD = 0.4  # 特征漂移阈值
    COMPUTE_TIMEOUT_SECONDS = 5
    MAX_EVENTS_PER_COMPUTE = 5000
    DECAY_FACTOR = 0.95  # 时间衰减因子

    def __init__(
        self,
        event_consumer: EventConsumer,
        window_aggregator: Optional[SlidingWindowFeatureAggregator] = None,
    ):
        self._consumer = event_consumer
        self._aggregator = window_aggregator or SlidingWindowFeatureAggregator()
        self._feature_cache: Dict[str, FeatureVector] = {}
        self._previous_features: Dict[str, FeatureVector] = {}
        self._lock = threading.Lock()
        self._running = False
        self._stats = defaultdict(int)
        self._last_compute_time: Dict[str, datetime] = {}

    def start(self):
        """启动实时特征计算循环"""
        self._running = True
        thread = threading.Thread(target=self._consume_loop, daemon=True)
        thread.start()
        logger.info("实时特征计算服务已启动")

    def stop(self):
        """停止服务"""
        self._running = False
        logger.info("实时特征计算服务已停止")

    def _consume_loop(self):
        """持续消费事件并更新特征"""
        while self._running:
            try:
                events = self._consumer.consume(max_records=500)
                if events:
                    self._process_events(events)
                    self._consumer.commit()
                else:
                    time.sleep(0.1)
            except KafkaConsumerError as e:
                logger.error(f"Kafka消费异常: {e}, 等待重连...")
                time.sleep(5)
            except Exception as e:
                logger.error(f"事件处理异常: {e}")
                self._stats["processing_errors"] += 1
                time.sleep(1)

    def _process_events(self, events: List[UserEvent]):
        """处理一批事件,更新特征"""
        user_events: Dict[str, List[UserEvent]] = defaultdict(list)
        for event in events:
            user_events[event.user_id].append(event)

        for user_id, user_evts in user_events.items():
            for event in user_evts:
                self._aggregator.add_event(event)

            # 仅在距上次计算超过一定间隔后才重算
            last_time = self._last_compute_time.get(user_id)
            now = datetime.now()
            if last_time and (now - last_time) < timedelta(seconds=10):
                continue

            try:
                feature = self._compute_features(user_id, now)
                if feature:
                    self._feature_cache[user_id] = feature
                    self._last_compute_time[user_id] = now
                    self._stats["features_computed"] += 1
            except FeatureComputeError as e:
                logger.warning(f"用户{user_id}特征计算失败: {e}")
                self._stats["compute_errors"] += 1

    def compute_for_user(
        self, user_id: str, now: Optional[datetime] = None
    ) -> FeatureVector:
        """
        为指定用户计算实时特征(主动调用接口)

        Returns:
            FeatureVector: 用户的实时特征向量
        """
        now = now or datetime.now()
        feature = self._compute_features(user_id, now)
        if feature:
            with self._lock:
                self._feature_cache[user_id] = feature
        return feature

    def get_feature_vector(self, user_id: str) -> Optional[FeatureVector]:
        """获取缓存的用户特征向量"""
        with self._lock:
            return self._feature_cache.get(user_id)

    def get_ranking_features(self, user_id: str) -> Optional[Dict[str, float]]:
        """获取用户排序模型特征"""
        feature = self.get_feature_vector(user_id)
        if feature:
            return feature.to_ranking_features()
        return None

    def _compute_features(
        self, user_id: str, now: datetime
    ) -> Optional[FeatureVector]:
        """核心特征计算逻辑"""
        start_time = time.time()

        # 获取多粒度窗口事件
        short_events = self._aggregator.get_events("short")
        medium_events = self._aggregator.get_events("medium")
        long_events = self._aggregator.get_events("long")

        # 过滤出当前用户的事件
        user_short = [e for e in short_events if e.user_id == user_id]
        user_medium = [e for e in medium_events if e.user_id == user_id]
        user_long = [e for e in long_events if e.user_id == user_id]

        if not user_long:
            return None

        # 按时间排序
        user_long.sort(key=lambda e: e.timestamp)
        user_medium.sort(key=lambda e: e.timestamp)

        feature = FeatureVector(
            user_id=user_id,
            version=self.FEATURE_VERSION,
            computed_at=now,
        )

        # 1. 分类偏好(时间衰减加权)
        feature.category_weights = self._compute_category_weights(
            user_long, now
        )

        # 2. 标签偏好(时间衰减加权)
        feature.tag_weights = self._compute_tag_weights(user_long, now)

        # 3. 行为频率统计
        feature.view_count = sum(
            1 for e in user_medium if e.event_type == EventType.VIEW
        )
        feature.click_count = sum(
            1 for e in user_medium if e.event_type == EventType.CLICK
        )
        feature.like_count = sum(
            1 for e in user_medium if e.event_type == EventType.LIKE
        )
        feature.share_count = sum(
            1 for e in user_medium if e.event_type == EventType.SHARE
        )
        feature.skip_count = sum(
            1 for e in user_medium if e.event_type == EventType.SKIP
        )
        feature.complete_count = sum(
            1 for e in user_medium if e.event_type == EventType.COMPLETE
        )

        # 4. 时长特征
        watch_events = [
            e for e in user_medium
            if e.event_type in (EventType.PLAY, EventType.COMPLETE, EventType.VIEW)
            and e.watch_duration_seconds > 0
        ]
        if watch_events:
            feature.avg_watch_duration = (
                sum(e.watch_duration_seconds for e in watch_events)
                / len(watch_events)
            )
            feature.total_watch_duration = sum(
                e.watch_duration_seconds for e in watch_events
            )
            feature.completion_rate = feature.complete_count / max(1, feature.view_count)

        # 5. 交互深度
        if feature.view_count > 0:
            feature.click_through_rate = feature.click_count / feature.view_count
        feature.engagement_score = self._compute_engagement_score(feature)

        # 6. 时段偏好
        feature.hourly_activity = self._compute_hourly_activity(user_long, now)

        # 7. 位置偏好
        click_with_pos = [
            e for e in user_medium
            if e.event_type == EventType.CLICK and e.position >= 0
        ]
        if click_with_pos:
            feature.avg_click_position = (
                sum(e.position for e in click_with_pos) / len(click_with_pos)
            )
            # 位置偏好偏差: 负值表示偏好靠前位置
            feature.position_bias = 5.0 - feature.avg_click_position

        # 8. 多样性特征
        feature.category_entropy = self._compute_entropy(
            feature.category_weights
        )
        feature.tag_diversity = len(feature.tag_weights) / max(
            1, sum(feature.tag_weights.values())
        )

        # 9. 特征漂移检测
        feature.drift_score = self._compute_drift_score(user_id, feature)

        # 超时检查
        elapsed = time.time() - start_time
        if elapsed > self.COMPUTE_TIMEOUT_SECONDS:
            logger.warning(
                f"用户{user_id}特征计算耗时{elapsed:.2f}s,超过阈值"
            )
            self._stats["compute_timeouts"] += 1

        return feature

    def _compute_category_weights(
        self, events: List[UserEvent], now: datetime
    ) -> Dict[str, float]:
        """计算分类偏好权重(带时间衰减)"""
        weights: Dict[str, float] = defaultdict(float)
        for event in events:
            hours_ago = max(0, (now - event.timestamp).total_seconds() / 3600)
            decay = self.DECAY_FACTOR ** hours_ago
            # 不同行为类型的权重
            type_weight = {
                EventType.VIEW: 1.0,
                EventType.CLICK: 2.0,
                EventType.LIKE: 5.0,
                EventType.SHARE: 8.0,
                EventType.COMPLETE: 3.0,
                EventType.SKIP: -1.0,
                EventType.DISLIKE: -5.0,
            }.get(event.event_type, 0.0)
            weights[event.video_category] += type_weight * decay

        # 归一化
        total = sum(max(0, v) for v in weights.values())
        if total > 0:
            weights = {k: max(0, v) / total for k, v in weights.items()}

        return dict(weights)

    def _compute_tag_weights(
        self, events: List[UserEvent], now: datetime
    ) -> Dict[str, float]:
        """计算标签偏好权重(带时间衰减)"""
        weights: Dict[str, float] = defaultdict(float)
        for event in events:
            hours_ago = max(0, (now - event.timestamp).total_seconds() / 3600)
            decay = self.DECAY_FACTOR ** hours_ago
            type_weight = {
                EventType.VIEW: 0.5,
                EventType.CLICK: 1.0,
                EventType.LIKE: 3.0,
                EventType.SHARE: 5.0,
                EventType.COMPLETE: 2.0,
                EventType.SKIP: -0.5,
                EventType.DISLIKE: -3.0,
            }.get(event.event_type, 0.0)
            for tag in event.video_tags:
                weights[tag] += type_weight * decay

        # 保留正权重并归一化
        weights = {k: v for k, v in weights.items() if v > 0}
        total = sum(weights.values())
        if total > 0:
            weights = {k: v / total for k, v in weights.items()}

        return weights

    def _compute_engagement_score(self, feature: FeatureVector) -> float:
        """计算综合互动得分"""
        score = (
            feature.click_count * 1.0
            + feature.like_count * 3.0
            + feature.share_count * 5.0
            + feature.complete_count * 2.0
            - feature.skip_count * 2.0
        )
        # 按曝光量归一化
        if feature.view_count > 0:
            score /= feature.view_count
        return min(10.0, max(0.0, score))

    def _compute_hourly_activity(
        self, events: List[UserEvent], now: datetime
    ) -> Dict[int, float]:
        """计算24小时各时段活跃度"""
        hourly: Dict[int, float] = defaultdict(float)
        for event in events:
            hours_ago = max(0, (now - event.timestamp).total_seconds() / 3600)
            decay = self.DECAY_FACTOR ** hours_ago
            hour = event.timestamp.hour
            hourly[hour] += decay

        total = sum(hourly.values())
        if total > 0:
            hourly = {h: v / total for h, v in hourly.items()}

        return dict(hourly)

    def _compute_entropy(self, distribution: Dict[str, float]) -> float:
        """计算概率分布的信息熵"""
        import math
        if not distribution:
            return 0.0
        probs = [v for v in distribution.values() if v > 0]
        if not probs:
            return 0.0
        return -sum(p * math.log2(p) for p in probs)

    def _compute_drift_score(
        self, user_id: str, current: FeatureVector
    ) -> float:
        """计算与上一版本特征的漂移分数(0-1)"""
        with self._lock:
            previous = self._previous_features.get(user_id)

        if not previous:
            with self._lock:
                self._previous_features[user_id] = current
            return 0.0

        # 比较分类权重分布的变化(JS散度近似)
        drift = self._distribution_drift(
            previous.category_weights, current.category_weights
        )

        # 比较行为率的变化
        if previous.click_through_rate > 0:
            ctr_drift = abs(
                current.click_through_rate - previous.click_through_rate
            ) / previous.click_through_rate
            drift = max(drift, min(1.0, ctr_drift))

        if previous.completion_rate > 0:
            comp_drift = abs(
                current.completion_rate - previous.completion_rate
            ) / previous.completion_rate
            drift = max(drift, min(1.0, comp_drift))

        # 更新前一次特征
        with self._lock:
            self._previous_features[user_id] = current

        return drift

    def _distribution_drift(
        self, dist_a: Dict[str, float], dist_b: Dict[str, float]
    ) -> float:
        """计算两个概率分布之间的漂移(简化的JS散度)"""
        all_keys = set(dist_a.keys()) | set(dist_b.keys())
        if not all_keys:
            return 0.0

        total_diff = 0.0
        for key in all_keys:
            a = dist_a.get(key, 0.0)
            b = dist_b.get(key, 0.0)
            total_diff += abs(a - b)

        # 归一化到0-1
        return min(1.0, total_diff / 2.0)

    def get_stats(self) -> Dict[str, int]:
        return dict(self._stats)

    def get_cached_user_count(self) -> int:
        with self._lock:
            return len(self._feature_cache)
```

## 异常场景补充

### 场景：实时特征计算延迟

```
触发条件: 当Kafka消费积压(Lag)超过10000条消息时,特征计算延迟显著上升;
          或特征计算单次耗时超过5秒(如用户行为事件量极大导致滑动窗口内事件过多);
          或JVM/Python GC暂停导致消费线程长时间阻塞;
          或下游特征存储(Redis)写入超时导致缓存更新失败
检测手段: 1. 监控Kafka consumer lag指标——通过kafka-consumer-groups命令或Burrow监控消费延迟,
          Lag > 10000时触发WARN告警, > 100000时触发CRITICAL告警;
          2. 在_compute_features方法中记录wall clock耗时,超过COMPUTE_TIMEOUT_SECONDS=5秒时
          记录WARN日志并增加compute_timeouts计数器;
          3. 监控特征缓存命中率——若大量请求命中缓存但缓存中的computed_at已超过30秒,
          则说明特征更新不及时;
          4. 端到端延迟探针——在事件入口注入带时间戳的探针事件,在特征输出端检测其处理耗时
处理方式: 1. 短期降级:当延迟超过阈值时,跳过短期窗口(5分钟)仅用中长期窗口计算特征,
          减少计算量;极端情况下仅返回缓存的旧特征(标记stale=true);
          2. 扩容消费组:动态增加Kafka消费分区数和消费者实例数,提升并行消费能力;
          3. 采样策略:对活跃用户(事件量>1000/小时)的事件进行采样,
          仅保留具有代表性的事件(如每种EventType保留最近N条)而非全量;
          4. 特征计算异步化:将实时计算与实时查询解耦——计算线程持续更新缓存,
          查询线程直接读缓存,两者通过读写锁隔离;
          5. 对Redis写入超时,切换为本地内存缓存(如LRU Cache),待Redis恢复后异步回写
预防措施: 1. 对滑动窗口内事件数量设置硬上限(MAX_EVENTS_PER_COMPUTE=5000),
          超出时截断最旧的事件;
          2. 按用户活跃度分级计算频率——高活跃用户每10秒重算一次,
          低活跃用户每60秒重算一次;
          3. 特征计算逻辑避免复杂聚合操作——预聚合中间结果,
          新事件到达时增量更新而非全量重算;
          4. Kafka topic按用户ID hash分区,保证同一用户的事件有序且可被同一消费者处理
```

### 场景：特征漂移导致推荐质量下降

```
触发条件: 用户兴趣发生真实变化(如从游戏视频转向美食视频),但模型仍基于旧特征推荐游戏内容;
          或外部因素导致短期行为波动(如节假日观看模式变化)被特征系统捕获后
          导致推荐列表大幅偏移;或数据质量问题(如视频分类标签错误)导致特征向量指向错误方向
检测手段: 1. 监控drift_score指标——当大量用户的drift_score超过DRIFT_THRESHOLD=0.4时触发告警;
          2. 监控推荐系统业务指标——若推荐点击率(CTR)在短时间内下降超过15%,
          且下降与特征更新时间相关,则疑似漂移问题;
          3. 离线A/B对比——将使用实时特征的推荐结果与使用离线T+1特征的推荐结果对比,
          若实时特征版本的CTR显著低于离线版本,则说明实时特征引入了噪声;
          4. 特征分布监控——对全局用户特征分布做统计,若某特征维度(如category_entropy)
          的均值/方差在短时间内发生大幅变化,则标记为漂移;
          5. 用户反馈信号——"不感兴趣"点击率上升是推荐质量下降的直接信号
处理方式: 1. 紧急回滚:将特征服务切换为使用上一个稳定版本的特征缓存,
          停止实时特征更新直到定位问题;
          2. 漂移平滑:对特征更新引入指数移动平均(EMA)而非直接替换——
          new_feature = alpha * realtime_feature + (1-alpha) * previous_feature,
          alpha=0.3时可以平滑掉短期波动;
          3. 置信度加权:当drift_score较高时,降低实时特征在排序模型中的权重,
          增加离线长期特征的权重;
          4. 数据质量修复:若漂移由标签错误导致,紧急修正错误标签并重新计算受影响用户的特征;
          5. 对确实发生了兴趣变化的用户,加速特征更新(提高alpha),
          但设置单次最大变化幅度限制(如任何特征维度单次变化不超过30%)
预防措施: 1. 特征更新引入平滑机制——不直接替换特征向量,而是与历史特征做加权融合;
          2. 设置特征变化的合理边界——任何单一特征维度(如某个category权重)
          单次更新变化幅度不超过50%,超出则截断;
          3. 双通道特征架构——同时维护实时特征和离线T+1特征,排序模型同时使用两个通道,
          实时特征负责捕捉短期兴趣,离线特征提供稳定基线;
          4. 特征质量监控仪表盘——实时展示各特征维度的全局分布、漂移分位数、异常用户占比;
          5. 灰度发布特征版本——新版特征计算逻辑先对5%用户生效,对比CTR指标后再全量
```

## 视频推荐冷启动与用户画像完整实现

```python
class ColdStartRecommendationService:
    """冷启动推荐：新用户画像 → 探索策略 → 快速学习 → 效果评估"""

    def build_new_user_profile(self, user_id, registration_data):
        """构建新用户画像"""
        # 1. 人口统计学特征
        age = registration_data.get("age")
        gender = registration_data.get("gender")
        location = registration_data.get("location")
        device = registration_data.get("device_type", "mobile")

        # 2. 从同类群体继承兴趣
        cohort_interests = self._get_cohort_interests(age, gender, location)

        # 3. 注册来源特征
        source = registration_data.get("source", "organic")
        source_categories = {
            "search": registration_data.get("search_query_categories", []),
            "referral": registration_data.get("referral_categories", []),
            "ad": registration_data.get("ad_categories", []),
        }

        # 4. 合并兴趣权重
        interest_weights = {}
        for category, weight in cohort_interests.items():
            interest_weights[category] = weight * 0.5  # 群体权重 50%

        for source_type, categories in source_categories.items():
            for cat in categories:
                if cat in interest_weights:
                    interest_weights[cat] += 0.3
                else:
                    interest_weights[cat] = 0.3

        # 5. 设置探索预算（新用户 30% 探索）
        exploration_budget = 0.30

        # 6. 存储画像
        self.db.insert("user_profiles", {
            "user_id": user_id,
            "profile_version": 1,
            "age": age,
            "gender": gender,
            "location": location,
            "device": device,
            "interest_weights": json.dumps(interest_weights),
            "exploration_budget": exploration_budget,
            "interaction_count": 0,
            "cohort_id": self._assign_cohort(age, gender),
            "created_at": now(),
            "updated_at": now()
        })

        return {"user_id": user_id, "interests": interest_weights,
                "exploration_budget": exploration_budget,
                "cohort_id": self._assign_cohort(age, gender)}

    def handle_first_interactions(self, user_id, interactions):
        """处理首批交互"""
        profile = self.db.get_user_profile(user_id)

        if not profile:
            return {"status": "profile_not_found"}

        interest_weights = json.loads(profile["interest_weights"])
        interaction_count = profile["interaction_count"]
        current_exploration = profile["exploration_budget"]

        # 1. 学习率随交互次数递减
        if interaction_count < 10:
            learning_rate = 0.3
        elif interaction_count < 50:
            learning_rate = 0.15
        else:
            learning_rate = 0.05

        # 2. 更新兴趣权重
        for interaction in interactions:
            video_id = interaction["video_id"]
            action = interaction["action"]  # view/like/share/skip/watch_time

            # 获取视频类别
            video = self.db.get_video(video_id)
            if not video:
                continue

            categories = video.get("categories", [])

            # 根据行为计算反馈
            feedback = self._calculate_feedback(action, interaction)

            for category in categories:
                old_weight = interest_weights.get(category, 0)
                new_weight = old_weight + learning_rate * feedback

                # 权重截断在 [0, 1]
                interest_weights[category] = max(0, min(1, new_weight))

            # 记录交互
            self.db.insert("user_interactions", {
                "interaction_id": str(uuid4()),
                "user_id": user_id,
                "video_id": video_id,
                "action": action,
                "feedback": feedback,
                "watch_time_seconds": interaction.get("watch_time", 0),
                "created_at": now()
            })

        # 3. 更新交互计数
        new_count = interaction_count + len(interactions)

        # 4. 逐步降低探索预算
        if new_count < 5:
            new_exploration = 0.30
        elif new_count < 20:
            new_exploration = 0.20
        elif new_count < 50:
            new_exploration = 0.10
        else:
            new_exploration = 0.05  # 成熟用户只保留 5% 探索

        # 5. 对未探索类别增加权重（鼓励发现）
        all_categories = self._get_all_categories()
        explored = set(interest_weights.keys())
        unexplored = [c for c in all_categories if c not in explored]

        for category in unexplored:
            interest_weights[category] = 0.05  # 低基线权重

        # 6. 更新画像
        self.db.update("user_profiles",
            {"interest_weights": json.dumps(interest_weights),
             "interaction_count": new_count,
             "exploration_budget": new_exploration,
             "updated_at": now()},
            {"user_id": user_id})

        return {"user_id": user_id,
                "interaction_count": new_count,
                "exploration_budget": new_exploration,
                "top_interests": sorted(interest_weights.items(),
                                       key=lambda x: -x[1])[:5]}

    def get_cold_start_recommendations(self, user_id, count=10):
        """获取冷启动推荐"""
        profile = self.db.get_user_profile(user_id)

        if not profile:
            # 完全新用户 → 返回热门 + 多样化
            return self._get_popular_diverse(count)

        interaction_count = profile["interaction_count"]
        exploration_budget = profile["exploration_budget"]
        interest_weights = json.loads(profile["interest_weights"])

        # 1. 根据成熟度选择策略
        if interaction_count < 5:
            # 极少交互 → 依赖群体
            cohort_recs = self._get_cohort_recommendations(
                profile["cohort_id"], count=int(count * 0.6))
            explore_recs = self._get_exploration_recommendations(
                interest_weights, count=int(count * 0.3))
            popular_recs = self._get_popular_diverse(int(count * 0.1))

        elif interaction_count < 50:
            # 有一些交互 → 偏好学习
            preference_recs = self._get_preference_recommendations(
                interest_weights, count=int(count * (1 - exploration_budget)))
            explore_recs = self._get_exploration_recommendations(
                interest_weights, count=int(count * exploration_budget))
            cohort_recs = []
            popular_recs = []

        else:
            # 成熟用户 → 交给主推荐引擎
            return self.main_recommendation_engine.get_recommendations(
                user_id, count)

        # 2. 合并去重
        all_recs = preference_recs if interaction_count >= 5 else []
        all_recs += cohort_recs + explore_recs + popular_recs

        seen = set()
        unique_recs = []
        for rec in all_recs:
            if rec["video_id"] not in seen:
                seen.add(rec["video_id"])
                unique_recs.append(rec)

        # 3. 多样性保证（同类不超过 2 个）
        final_recs = []
        category_count = {}

        for rec in unique_recs:
            cat = rec.get("category", "unknown")
            if category_count.get(cat, 0) < 2:
                final_recs.append(rec)
                category_count[cat] = category_count.get(cat, 0) + 1

            if len(final_recs) >= count:
                break

        return final_recs[:count]

    def evaluate_cold_start_quality(self, user_id, days_since_registration):
        """评估冷启动效果"""
        # 1. 点击率
        total_recs = self.db.count("recommendation_impressions",
            user_id=user_id,
            created_at__gte=now()-timedelta(days=days_since_registration))
        clicked = self.db.count("user_interactions",
            user_id=user_id,
            action__in=["view", "like", "share"],
            created_at__gte=now()-timedelta(days=days_since_registration))

        ctr = clicked / max(total_recs, 1)

        # 2. 成熟用户 CTR 对比
        mature_ctr = self.db.query_one(
            "SELECT AVG(ctr) as avg FROM user_engagement_stats "
            "WHERE interaction_count > 100")["avg"] or 0.05

        ctr_ratio = ctr / max(mature_ctr, 0.001)

        # 3. 首次有意义交互时间
        first_meaningful = self.db.query_one(
            "SELECT MIN(created_at) as first FROM user_interactions "
            "WHERE user_id = %s AND action IN ('like', 'share', 'favorite') "
            "AND watch_time_seconds > 30",
            user_id)["first"]

        if first_meaningful:
            registration = self.db.get_user_profile(user_id)["created_at"]
            time_to_engagement = (first_meaningful - registration).total_seconds() / 3600
        else:
            time_to_engagement = None

        # 4. 内容多样性
        categories_consumed = self.db.query(
            "SELECT DISTINCT v.category FROM user_interactions ui "
            "JOIN videos v ON ui.video_id = v.id "
            "WHERE ui.user_id = %s AND ui.action != 'skip'",
            user_id)
        diversity = len(categories_consumed)

        return {
            "user_id": user_id,
            "days_since_registration": days_since_registration,
            "ctr": round(ctr, 4),
            "mature_user_ctr": round(mature_ctr, 4),
            "ctr_vs_mature": round(ctr_ratio, 2),
            "time_to_engagement_hours": round(time_to_engagement, 1) if time_to_engagement else None,
            "content_diversity": diversity,
            "quality_rating": "good" if ctr_ratio > 0.6 else "needs_improvement"
        }

    def _calculate_feedback(self, action, interaction):
        """计算反馈值"""
        if action == "like":
            return 1.0
        elif action == "share":
            return 1.2
        elif action == "favorite":
            return 1.5
        elif action == "view":
            watch_time = interaction.get("watch_time", 0)
            video_duration = interaction.get("duration", 60)
            completion_rate = watch_time / max(video_duration, 1)
            if completion_rate > 0.8:
                return 0.8
            elif completion_rate > 0.5:
                return 0.3
            else:
                return -0.1
        elif action == "skip":
            return -0.5
        elif action == "dislike":
            return -1.0
        return 0

    def _get_cohort_interests(self, age, gender, location):
        """获取群体兴趣"""
        cohort = self._assign_cohort(age, gender)

        result = self.db.query_one(
            "SELECT avg_interests FROM cohort_profiles "
            "WHERE cohort_id = %s", cohort)

        if result:
            return json.loads(result["avg_interests"])

        return {"entertainment": 0.5, "education": 0.3, "news": 0.2}

    def _assign_cohort(self, age, gender):
        """分配群体"""
        age_group = "young" if age and age < 25 else "mid" if age and age < 45 else "senior"
        return f"{age_group}_{gender or 'unknown'}"

    def _get_all_categories(self):
        """获取所有类别"""
        return [r["category"] for r in self.db.query(
            "SELECT DISTINCT category FROM videos WHERE status = 'active'")]

    def _get_popular_diverse(self, count):
        """获取热门多样化推荐"""
        categories = self._get_all_categories()
        per_category = max(1, count // max(len(categories), 1))

        results = []
        for cat in categories[:count]:
            videos = self.db.query(
                "SELECT id as video_id, category, title FROM videos "
                "WHERE category = %s AND status = 'active' "
                "ORDER BY view_count DESC LIMIT %s", cat, per_category)
            results.extend(videos)

        return results[:count]

    def _get_cohort_recommendations(self, cohort_id, count):
        """获取群体推荐"""
        return self.db.query(
            "SELECT v.id as video_id, v.category, v.title "
            "FROM videos v JOIN cohort_recommendations cr ON v.id = cr.video_id "
            "WHERE cr.cohort_id = %s ORDER BY cr.score DESC LIMIT %s",
            cohort_id, count)

    def _get_exploration_recommendations(self, interest_weights, count):
        """获取探索推荐（低权重类别）"""
        sorted_interests = sorted(interest_weights.items(), key=lambda x: x[1])
        low_interest_cats = [c for c, w in sorted_interests[:5]]

        results = []
        for cat in low_interest_cats:
            videos = self.db.query(
                "SELECT id as video_id, category, title FROM videos "
                "WHERE category = %s AND status = 'active' "
                "ORDER BY created_at DESC LIMIT 2", cat)
            results.extend(videos)

        return results[:count]

    def _get_preference_recommendations(self, interest_weights, count):
        """获取偏好推荐"""
        top_categories = sorted(interest_weights.items(),
                               key=lambda x: -x[1])[:5]

        results = []
        for cat, weight in top_categories:
            limit = max(1, int(count * weight / sum(w for _, w in top_categories)))
            videos = self.db.query(
                "SELECT id as video_id, category, title FROM videos "
                "WHERE category = %s AND status = 'active' "
                "ORDER BY quality_score DESC LIMIT %s", cat, limit)
            results.extend(videos)

        return results[:count]
```

## 异常场景补充

### 场景：冷启动推荐陷入信息茧房

```
触发：新用户首次点击了 3 个娱乐视频 → 算法强化娱乐权重 → 后续只推荐娱乐 → 用户看不到其他内容
检测：
  1. 推荐列表类别单一（>80% 同一类）→ 信息茧房
  2. 用户消费内容多样性持续下降 → 陷入茧房
处理：
  1. 强制多样性约束（同类不超过 40%）
  2. 探索预算随时间恢复
  3. 定期插入"惊喜推荐"
预防：多样性约束 + 探索恢复 + 惊喜推荐
```

### 场景：用户画像与实际偏好严重偏离

```
触发：用户账号被家人借用 → 观看儿童节目 → 画像偏向儿童 → 自己使用时推荐不相关内容
检测：
  1. 用户连续跳过推荐内容 → 偏好不匹配
  2. 行为模式突变 → 可能换人使用
处理：
  1. 检测到行为突变 → 触发画像重新学习
  2. 近期行为权重 > 历史权重
  3. 提供"不感兴趣"反馈按钮
预防：行为突变检测 + 近期加权 + 负反馈
```

### 场景：大量新用户同时注册导致推荐延迟

```
触发：营销活动 → 1 小时内 10 万新用户注册 → 画像构建计算量大 → 推荐延迟 > 5 秒
检测：
  1. 推荐响应时间 P99 > 3 秒 → 延迟
  2. 新用户注册速率 > 平时 10 倍 → 流量洪峰
处理：
  1. 群体画像缓存（同类用户共享画像）
  2. 新用户先返回热门推荐（无需画像）
  3. 异步构建画像
预防：群体缓存 + 热门降级 + 异步构建
```
