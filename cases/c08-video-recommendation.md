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

**1. 协同过滤召回**

```python
class CollaborativeFilterRecaller:
    def recall(self, user_id, count=200):
        # 1. 获取用户最近互动的视频
        recent_videos = self.get_recent_interactions(user_id, limit=50)
        
        # 2. 查找"看过相同视频"的用户（相似用户）
        similar_users = self.get_similar_users(recent_videos, limit=100)
        
        # 3. 获取相似用户看过的、当前用户没看过的视频
        candidates = []
        for similar_user in similar_users:
            user_videos = self.get_user_interactions(similar_user, limit=20)
            candidates.extend([v for v in user_videos if v not in recent_videos])
        
        # 4. 按频次排序取 Top N
        counter = Counter(candidates)
        return [v for v, _ in counter.most_common(count)]
```

**2. 向量召回（双塔模型）**

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
class VectorRecaller:
    def recall(self, user_id, count=200):
        # 1. 获取用户向量（预计算并缓存）
        user_vector = self.redis.get(f"user_vector:{user_id}")
        if not user_vector:
            user_features = self.feature_store.get_user_features(user_id)
            user_vector = self.user_tower.predict(user_features)
            self.redis.setex(f"user_vector:{user_id}", 3600, user_vector)
        
        # 2. 向量近邻搜索（FAISS）
        distances, video_ids = self.faiss_index.search(user_vector, count)
        
        return video_ids
```

**FAISS 索引的维护：**
- 5000 万视频向量，128 维，约 25GB
- 使用 IVF-PQ 索引：搜索精度约 90%，搜索延迟 < 20ms
- 每日全量重建索引（新视频入库后）

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

### 实时特征管道

**用户实时兴趣更新：**

```
用户点击视频A → 事件上报 → Kafka → Flink → 更新用户特征 Redis
                                            ↓
                                    下次推荐请求使用新特征
```

```python
class RealtimeFeatureUpdater:
    def on_user_action(self, action):
        """用户行为实时更新特征"""
        user_id = action.user_id
        video_id = action.video_id
        
        # 1. 更新用户最近行为序列（Redis List）
        self.redis.lpush(f"user:recent:{user_id}", video_id)
        self.redis.ltrim(f"user:recent:{user_id}", 0, 49)  # 保留最近50个

        # 2. 更新用户兴趣权重（Redis Hash）
        video_info = self.get_video_info(video_id)
        category = video_info.category
        self.redis.hincrby(f"user:interest:{user_id}", category, 1)

        # 3. 更新用户向量（异步，不阻塞主流程）
        self.mq.produce("user-vector-update", {"user_id": user_id})
```

**实时性分析：**
- 行为上报 → Kafka → Flink：约 1-3 秒
- 更新 Redis：约 1ms
- 用户下次请求（平均 30 秒后）使用新特征
- **从用户点击到推荐结果变化的延迟：约 30-35 秒**（主要取决于用户下次请求的时间）

### 新视频冷启动方案

**问题：** 新视频零播放、零互动，推荐系统没有信号。

**方案：流量池赛马机制**

```
新视频上传 → 进入初始流量池（100次曝光）
              │
              ├─ 完播率 > 30% 且 点赞率 > 5% → 进入二级流量池（1000次曝光）
              │     │
              │     ├─ 数据继续好 → 三级池（10000次）→ ... → 全站推荐
              │     └─ 数据差 → 停止推流
              │
              └─ 完播率 < 15% → 停止推流
```

```python
class ColdStartHandler:
    POOLS = {
        0: {"impressions": 100,   "next_pool": 1},  # 初始池
        1: {"impressions": 1000,  "next_pool": 2},  # 二级池
        2: {"impressions": 10000, "next_pool": 3},  # 三级池
        3: {"impressions": 100000, "next_pool": 4}, # 四级池
        4: {"impressions": -1,    "next_pool": None}, # 全站推荐
    }

    def evaluate(self, video_id, current_pool):
        """评估视频是否可以进入下一级流量池"""
        metrics = self.get_video_metrics(video_id)
        pool_config = self.POOLS[current_pool]
        
        # 检查是否达到当前池的曝光量
        if metrics.impressions < pool_config["impressions"]:
            return  # 还没曝光够，继续
        
        # 评估核心指标
        if metrics.completion_rate > 0.3 and metrics.like_rate > 0.05:
            # 进入下一级流量池
            self.promote(video_id, pool_config["next_pool"])
        else:
            # 停止推流
            self.demote(video_id)

    def promote(self, video_id, next_pool):
        """晋升到下一级流量池"""
        self.db.update_video_pool(video_id, next_pool)
        # 视频的推荐权重提升，会被更多召回通道选中
```

**冷启动的补充策略：内容理解**

```python
class ContentUnderstanding:
    """利用视频内容特征做冷启动推荐"""
    
    def extract_features(self, video):
        # 1. 视频封面图像 → CNN 提取视觉特征
        visual_features = self.cnn.extract(video.thumbnail)
        
        # 2. 视频标题/描述 → NLP 提取文本特征
        text_features = self.nlp.extract(video.title + " " + video.description)
        
        # 3. 视频音频 → 提取音频特征（背景音乐、语音内容）
        audio_features = self.audio_model.extract(video.audio)
        
        # 4. 综合生成视频向量
        video_vector = self.fusion_model.predict(
            visual_features, text_features, audio_features
        )
        
        return video_vector
    
    def recommend_for_new_video(self, video_vector):
        """基于内容特征找到相似的视频，从而推荐给相似受众"""
        # 找内容相似的视频
        similar_videos = self.faiss_index.search(video_vector, count=50)
        
        # 这些相似视频的受众就是新视频的目标用户
        target_users = set()
        for vid, _ in similar_videos:
            users = self.get_video_audience(vid, limit=100)
            target_users.update(users)
        
        return list(target_users)
```

### 性能预算分配

| 阶段 | 延迟预算 | 具体耗时 | 计算 |
|------|---------|---------|------|
| 召回 | 50ms | 多路并行 20ms + 合并 5ms | 25ms |
| 粗排 | 30ms | 500次内积计算 | 25ms |
| 精排 | 80ms | 100次DNN推理（GPU批量） | 70ms |
| 重排 | 20ms | 规则过滤 + 排序 | 10ms |
| 网络传输 | 20ms | 10条视频元数据 | 15ms |
| **总计** | **200ms** | | **145ms** |

留有 55ms 余量，应对 GC 停顿、网络抖动等。

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

## 延伸思考

- **多目标优化**：推荐系统不只是优化点击率，还要优化停留时长、完播率、互动率、商业化收益。如何设计多目标损失函数？
- **推荐可解释性**：用户问"为什么推荐这个视频"，系统如何给出人类可理解的解释？
- **隐私保护**：如何在不收集用户行为数据的前提下做推荐？联邦学习方案？