# C03: 社交Feed流的实时推送

## 业务场景

某社交平台，用户可以关注其他用户并看到其发布的内容。类似微博/Twitter 的 Feed 流。Feed 流是用户打开 App 看到的第一个页面，是整个产品的核心体验。

**已知数据：**
- 日活用户：5000 万
- 人均关注：200 人（中位数 80 人，均值被头部用户拉高）
- 头部大V：100+ 人，每人关注者 100 万-5000 万
- 腰部创作者：约 10 万人，关注者 1 万-100 万
- 每秒新增帖子：5000 条（其中大V贡献约 50 条/秒，其余为普通用户）
- Feed 刷新延迟要求：< 1 秒（新帖发布后 1 秒内出现在粉丝 Feed 中）
- 用户浏览 Feed 时需支持"下拉刷新"和"上拉加载更多"
- 帖子生命周期：通常 24-48 小时后不再被浏览

**用户行为特征：**
- 每次打开 App 刷新 3-5 次，每次浏览 20-50 条帖子
- 高峰时段（早 8-9 点、午 12-13 点、晚 20-22 点）流量是平峰的 3 倍
- 用户对"刷新后看到新内容"的期望极高——如果刷新 3 次都看到同样的帖子，用户会觉得 App 卡了

## 核心挑战

### 挑战 1：推模型 vs 拉模型的根本矛盾

**纯推模型（写扩散）：** 用户发帖时，将帖子 ID 写入每个粉丝的收件箱。

- 普通用户（200 粉丝）发帖：写入 200 次 → 可接受
- 大V（100 万粉丝）发帖：写入 100 万次 → 不可接受
- 超级大V（5000 万粉丝）发帖：写入 5000 万次 → 完全不可行

5000 条/秒 × 平均粉丝数 200 = 100 万次/秒的 Redis 写入。看起来可接受，但问题是长尾——大V发帖瞬间的写入量是普通用户的 50 万倍。

**纯拉模型（读扩散）：** 用户刷 Feed 时，实时读取所有关注对象的最新帖子并合并。

- 关注 200 人 → 读取 200 个数据源并合并
- 5000 万 DAU × 每人每天刷新 5 次 = 2.5 亿次读取
- 每次读取 200 个数据源 → 500 亿次 Redis 读取/天
- 峰值 QPS：约 30 万次读取 → 可接受，但每次读取需要 200 次 Redis 调用，延迟高

### 挑战 2：大V的扇出灾难

一个大V发一条帖子，在推模型下需要写入 100 万-5000 万个收件箱。这不是"慢一点"的问题，而是：

- 100 万次 Redis ZADD，即使管道化也需要约 10 秒
- 5000 万次 ZADD，即使分布式并行也需要约 30 秒
- 在写入完成之前，部分粉丝看不到这条帖子 → 不满足 1 秒延迟要求

### 挑战 3：取关后 Feed 中的历史帖子

用户取关某人后，Feed 中该人的历史帖子应该如何处理？

- 如果从收件箱中删除：需要遍历整个收件箱找出该用户的所有帖子，O(N) 操作，成本极高
- 如果不删除：用户取关后还能看到该人的帖子，体验差

### 挑战 4：Feed 排序不是简单的时间倒序

实际产品中，Feed 排序可能涉及：
- 时间序（最简单）
- 算法推荐序（基于兴趣、互动历史）
- 社交权重（好友的帖子排在前面）
- 商业插入（广告帖子混入）

## 设计约束

- Feed 缓存使用 Redis，单集群容量约 1TB（5000 万用户 × 每用户收件箱约 20KB）
- 帖子内容存储在 MySQL
- 消息队列可用（Kafka）
- 成本敏感：不能为每个用户维护无限长的 Feed 副本
- 1 秒延迟是硬性要求——不是"尽力而为"，而是必须满足

## 请先独立思考（限时 40 分钟）

1. 画出纯推模型和纯拉模型的数据流图，标出每一步的读写量和延迟。
2. 设计一个混合方案，使得大V和普通用户都能满足 1 秒延迟要求。给出具体的阈值选择理由。
3. 用户 A 关注了 5 个大V和 195 个普通用户，刷新 Feed 时需要读取多少次 Redis？延迟是多少？
4. 取关后 Feed 中历史帖子的处理方案，比较至少 3 种方案的优劣。

---

## 设计解析

### 方案：推拉结合的混合模型

**核心思路：普通用户用推模型，大V用拉模型。** 这不是新想法（Twitter 2012 年就采用了），但关键在于阈值选择和边界条件的处理。

**阈值选择：粉丝 1 万为分界线**

为什么是 1 万而不是 10 万？

- 粉丝 1 万：推模型写入 1 万次 Redis ZADD，管道化约 100ms → 可接受
- 粉丝 10 万：推模型写入 10 万次，管道化约 1 秒 → 临界值，勉强可接受
- 粉丝 100 万：推模型写入 100 万次 → 不可接受

选择 1 万作为阈值，是因为：
- 1 万粉丝的用户占比 < 1%，但贡献了 > 80% 的扇出写入量
- 阈值设得越低，推模型的写入量越小，但拉模型读取时合并的大V越多
- 1 万阈值意味着普通用户关注列表中平均只有 2-3 个大V，拉模型合并量极小

### 完整架构

```
发帖流程：
  用户发帖 → 帖子服务 → Kafka → Feed 分发服务
                                    │
                                    ├─ 粉丝数 < 1万 → 推模型
                                    │   ├─ 批量 ZADD 到粉丝收件箱
                                    │   └─ 同时写入发帖人的发件箱
                                    │
                                    └─ 粉丝数 ≥ 1万 → 拉模型
                                        └─ 只写入发帖人的发件箱

读取流程：
  用户刷新 Feed → Feed 服务
                    │
                    ├─ 1. 读取收件箱（推模型结果）
                    │     ZREVRANGE feed:inbox:{userId} 0 19 WITHSCORES
                    │
                    ├─ 2. 读取关注的大V列表
                    │     SMEMBERS feed:bigv:{userId}
                    │
                    ├─ 3. 并行读取每个大V的发件箱
                    │     ZREVRANGE feed:outbox:{bigvId} 0 19 WITHSCORES
                    │
                    ├─ 4. 合并排序（按时间戳降序）
                    │
                    ├─ 5. 去重后取 Top 20
                    │
                    └─ 6. 批量读取帖子详情
                          MGET post:{postId1} post:{postId2} ...
```

### 数据结构的完整设计

**收件箱（Inbox）：每个用户一个 Sorted Set**

```redis
Key: feed:inbox:{userId}
Type: Sorted Set
Score: 发帖时间戳（毫秒）
Value: {postId}:{authorId}  ← 包含作者ID，用于取关过滤

写入：ZADD feed:inbox:U12345 1715000000123 "P789:A456"
读取：ZREVRANGE feed:inbox:U12345 0 19 WITHSCORES
裁剪：ZREMRANGEBYRANK feed:inbox:U12345 1000 -1  ← 只保留最近 1000 条
```

**发件箱（Outbox）：每个大V一个 Sorted Set**

```redis
Key: feed:outbox:{userId}
Type: Sorted Set
Score: 发帖时间戳（毫秒）
Value: {postId}

写入：ZADD feed:outbox:A456 1715000000123 "P789"
读取：ZREVRANGE feed:outbox:A456 0 19 WITHSCORES
裁剪：ZREMRANGEBYRANK feed:outbox:A456 500 -1  ← 只保留最近 500 条
```

**为什么 Inbox 的 value 要包含 authorId？**
- 取关过滤：读取 Feed 时，检查每条帖子的 authorId 是否仍在关注列表中
- 如果不含 authorId，需要额外查询帖子的作者，增加 Redis/MySQL 读取

**大V列表：**

```redis
Key: feed:bigv:{userId}
Type: Set
Value: 大V的 userId

维护时机：
- 用户关注新大V → SADD feed:bigv:{userId} {bigvId}
- 用户取关大V → SREM feed:bigv:{userId} {bigvId}
- 大V粉丝数突破 1 万 → 所有粉丝的 bigv set 中加入该用户
```

**帖子详情缓存：**

```redis
Key: post:{postId}
Type: String (JSON)
Value: {"id":"P789","authorId":"A456","content":"...","images":[...],"createdAt":1715000000123}
TTL: 24 小时（帖子 24 小时后很少被访问）
```

### 完整数据库设计（MySQL）

Feed 系统依赖 MySQL 作为持久化存储，Redis 仅作为加速层。以下为核心表的完整设计：

**users 表：用户基础信息**

```sql
CREATE TABLE users (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '用户ID',
    username VARCHAR(64) NOT NULL COMMENT '用户名',
    nickname VARCHAR(128) NOT NULL DEFAULT '' COMMENT '昵称',
    avatar_url VARCHAR(512) NOT NULL DEFAULT '' COMMENT '头像URL',
    bio VARCHAR(512) NOT NULL DEFAULT '' COMMENT '个人简介',
    status TINYINT NOT NULL DEFAULT 1 COMMENT '状态: 1=正常, 2=冻结, 3=注销',
    follower_count INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '粉丝数（实时计数器）',
    following_count INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '关注数（实时计数器）',
    post_count INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '帖子数（实时计数器）',
    is_bigv TINYINT NOT NULL DEFAULT 0 COMMENT '是否大V标记: 0=否, 1=是',
    bigv_threshold_at DATETIME NULL COMMENT '首次达到1万粉丝的时间',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uk_username (username),
    KEY idx_follower_count (follower_count),
    KEY idx_is_bigv (is_bigv, follower_count)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';
```

**注意：** `follower_count` 使用实时计数器而非 `COUNT(follows)` 查询，因为粉丝数是高频访问的热点数据，实时计算在千万级数据量下不可行。计数器通过 Redis INCR 维护，定期回写 MySQL。

**posts 表：帖子内容**

```sql
CREATE TABLE posts (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '帖子ID',
    author_id BIGINT UNSIGNED NOT NULL COMMENT '作者用户ID',
    content TEXT NOT NULL COMMENT '帖子正文（支持长文本）',
    images JSON NULL COMMENT '图片列表 [{"url":"...","width":800,"height":600}]',
    video_url VARCHAR(512) NULL COMMENT '视频URL（可选）',
    category VARCHAR(32) NOT NULL DEFAULT 'general' COMMENT '内容分类: general, tech, entertainment, sports, news...',
    tags JSON NULL COMMENT '标签列表 ["#话题1", "#话题2"]',
    repost_source_id BIGINT UNSIGNED NULL COMMENT '转发源帖子ID（0表示原创）',
    reply_count INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '回复数',
    like_count INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '点赞数',
    repost_count INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '转发数',
    view_count INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '浏览数',
    is_deleted TINYINT NOT NULL DEFAULT 0 COMMENT '是否删除: 0=否, 1=是',
    deleted_at DATETIME NULL COMMENT '删除时间',
    visibility TINYINT NOT NULL DEFAULT 1 COMMENT '可见性: 1=公开, 2=仅粉丝, 3=仅自己',
    spam_score FLOAT NOT NULL DEFAULT 0.0 COMMENT '反垃圾评分(0-1, 越高越可疑)',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '发帖时间',
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    KEY idx_author_created (author_id, created_at DESC),
    KEY idx_created_at (created_at DESC),
    KEY idx_category_created (category, created_at DESC),
    KEY idx_deleted (is_deleted, created_at DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='帖子表';
```

**索引设计理由：**
- `idx_author_created`：拉模型读取大V的发件箱历史数据时，按作者+时间倒序查询，是最高频的访问路径
- `idx_category_created`：推荐排序按分类筛选候选集
- `idx_deleted`：清理服务批量扫描已删除帖子时使用

**follows 表：关注关系**

```sql
CREATE TABLE follows (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '关系ID',
    follower_id BIGINT UNSIGNED NOT NULL COMMENT '关注者用户ID',
    followee_id BIGINT UNSIGNED NOT NULL COMMENT '被关注者用户ID',
    status TINYINT NOT NULL DEFAULT 1 COMMENT '状态: 1=正常关注, 2=静默关注(不推送到Feed), 3=已取关',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '关注时间',
    unfollowed_at DATETIME NULL COMMENT '取关时间（status=3时记录）',
    PRIMARY KEY (id),
    UNIQUE KEY uk_follower_followee (follower_id, followee_id),
    KEY idx_followee (followee_id, status),       -- 查询某人的粉丝列表
    KEY idx_follower (follower_id, status),        -- 查询某人关注了谁
    KEY idx_unfollowed (status, unfollowed_at)     -- 批量清理取关数据
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='关注关系表';
```

**follows 表的关键设计：**
- `status` 字段支持"静默关注"——用户关注某人但不希望其帖子出现在 Feed 中（类似微博的"悄悄关注"），Feed 分发服务在推送时检查 status=1 才推送
- 取关不物理删除，而是标记 `status=3`，保留 `unfollowed_at` 时间。这为后续数据分析（取关趋势）和异常检测（大规模取关事件）提供数据支撑
- `uk_follower_followee` 唯一索引防止重复关注

**feed_items 表：Feed 物化视图（MySQL 回溯查询用）**

```sql
CREATE TABLE feed_items (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '记录ID',
    user_id BIGINT UNSIGNED NOT NULL COMMENT 'Feed 所属用户ID',
    post_id BIGINT UNSIGNED NOT NULL COMMENT '帖子ID',
    author_id BIGINT UNSIGNED NOT NULL COMMENT '帖子作者ID',
    source TINYINT NOT NULL COMMENT '来源: 1=推模型推送, 2=拉模型回溯',
    is_read TINYINT NOT NULL DEFAULT 0 COMMENT '是否已读',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '帖子创建时间',
    inserted_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '写入Feed的时间',
    PRIMARY KEY (id),
    UNIQUE KEY uk_user_post (user_id, post_id),
    KEY idx_user_created (user_id, created_at DESC),
    KEY idx_author (author_id, user_id)  -- 取关时按作者查找Feed记录
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='Feed物化视图表';
```

**feed_items 表的作用：** 这个表不是 Feed 读取的主路径（主路径是 Redis），而是作为以下场景的兜底：
- Redis 收件箱数据丢失（故障恢复）时的重建来源
- 游标分页中，当用户翻页超过 Redis 收件箱保留的 500 条时，回溯到 MySQL 查询
- 数据分析：统计每条帖子的 Feed 曝光量

**bigv_flags 表：大V标记与阈值迁移日志**

```sql
CREATE TABLE bigv_flags (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '记录ID',
    user_id BIGINT UNSIGNED NOT NULL COMMENT '大V用户ID',
    follower_count_at_mark INT UNSIGNED NOT NULL COMMENT '标记时的粉丝数',
    strategy VARCHAR(16) NOT NULL DEFAULT 'pull' COMMENT '分发策略: push, pull',
    migration_status TINYINT NOT NULL DEFAULT 0 COMMENT '迁移状态: 0=待迁移, 1=迁移中, 2=迁移完成, 3=迁移失败',
    migration_started_at DATETIME NULL COMMENT '迁移开始时间',
    migration_completed_at DATETIME NULL COMMENT '迁移完成时间',
    rollback_count INT UNSIGNED NOT NULL DEFAULT 0 COMMENT '回滚次数（拉→推迁移）',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uk_user (user_id),
    KEY idx_migration_status (migration_status, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='大V标记与迁移日志表';
```

**bigv_flags 表的设计理由：**
- 记录每次阈值迁移的完整状态，确保迁移过程可追踪、可回滚
- `migration_status` 状态机：待迁移→迁移中→迁移完成（或迁移失败回滚）
- 如果迁移过程中系统故障，重启后可以根据 `migration_status=1`（迁移中）继续未完成的迁移
- `rollback_count` 记录回滚次数，如果某用户频繁在 1 万阈值上下波动（"薛定谔的大V"），超过 3 次回滚后直接锁定为拉模型，避免反复迁移的系统开销

**数据量估算与分库分表：**

| 表 | 预估日增量 | 预估总量(1年) | 是否需要分表 |
|----|-----------|-------------|------------|
| users | ~5万/天 | 5亿+ | 按 id 取模分 16 库 |
| posts | 5000/秒=4.32亿/天 | 超千亿(需冷热分离) | 按 author_id 分 64 库，热数据存 SSD |
| follows | ~100万/天 | 100亿+ | 按 follower_id 分 32 库 |
| feed_items | 峰值75万/秒写入 | 超万亿(需冷热分离) | 按 user_id 分 128 库，仅保留 7 天热数据 |
| bigv_flags | 极少量 | ~10万 | 单表即可 |

**posts 和 feed_items 的冷热分离策略：**
- 热数据（7天内）：存 MySQL + Redis 缓存，SSD 存储
- 冷数据（7天以上）：归档到 ClickHouse 或 HBase，仅用于数据分析
- MySQL 中 `posts` 表通过 `is_deleted` 和 `created_at` 索引定期做物理删除（PT-DELETE 工具分批删除，避免锁表）

### 推模型的写入优化

**普通用户发帖的推送流程：**

```python
class FeedPushService:
    def on_new_post(self, post):
        author_id = post.author_id
        followers = self.get_followers(author_id)  # 从 Redis/DB 获取粉丝列表
        
        if len(followers) > 10000:
            # 大V → 只写发件箱
            self.write_outbox(author_id, post)
            return
        
        # 普通用户 → 推模型 + 写发件箱
        # 1. 写入发帖人的发件箱
        self.write_outbox(author_id, post)
        
        # 2. 批量推送到粉丝收件箱（Redis Pipeline）
        # 将粉丝分批，每批 500 人
        for batch in self.chunk(followers, size=500):
            pipe = self.redis.pipeline()
            for follower_id in batch:
                pipe.zadd(
                    f"feed:inbox:{follower_id}",
                    {f"{post.id}:{author_id}": post.created_at_ms}
                )
                # 裁剪到 1000 条
                pipe.zremrangebyrank(f"feed:inbox:{follower_id}", 0, -(1001))
            pipe.execute()
```

**写入性能分析：**

| 场景 | 粉丝数 | 批次数 | 预估耗时 |
|------|--------|--------|---------|
| 普通用户 | 200 | 1 | 5ms |
| 小V | 5000 | 10 | 50ms |
| 中V | 10000 | 20 | 100ms |
| 大V(>1万) | 100万+ | N/A | 不推送，走拉模型 |

**Redis 写入量估算：**

- 5000 帖/秒 × 平均粉丝 200 × (1万阈值过滤后约 150) = 75 万次 ZADD/秒
- 3 节点 Redis Cluster，每节点 25 万 QPS → 可承受
- 管道化后实际网络往返大幅减少

### 拉模型的读取实现

**Feed 服务的合并读取：**

```python
class FeedReadService:
    def get_feed(self, user_id, count=20, max_score=None):
        """
        获取用户的 Feed 列表
        max_score: 分页游标（上次最后一条帖子的时间戳）
        """
        # 1. 读取收件箱（推模型的结果）
        if max_score:
            inbox_items = self.redis.zrevrangebyscore(
                f"feed:inbox:{user_id}", max_score - 1, "-inf",
                start=0, num=count, withscores=True
            )
        else:
            inbox_items = self.redis.zrevrange(
                f"feed:inbox:{user_id}", 0, count - 1, withscores=True
            )
        
        # 2. 获取关注的大V列表
        bigv_ids = self.redis.smembers(f"feed:bigv:{user_id}")
        
        # 3. 并行读取每个大V的发件箱
        outbox_items = []
        if bigv_ids:
            # 使用 asyncio 并行读取
            outbox_items = await self.gather_outbox_reads(
                bigv_ids, count, max_score
            )
        
        # 4. 合并排序
        all_items = inbox_items + outbox_items
        # 去重（同一个 postId 可能同时出现在收件箱和发件箱中）
        seen = set()
        deduped = []
        for item in sorted(all_items, key=lambda x: -x[1]):  # 按时间戳降序
            post_id = item[0].split(":")[0] if ":" in item[0] else item[0]
            if post_id not in seen:
                seen.add(post_id)
                deduped.append(item)
        
        # 5. 取 Top N
        feed_items = deduped[:count]
        
        # 6. 取关过滤：检查每条帖子的作者是否仍在关注列表中
        following_set = self.redis.smembers(f"following:{user_id}")
        filtered = [
            item for item in feed_items
            if self.get_author_id(item[0]) in following_set
        ]
        
        # 7. 批量读取帖子详情
        post_ids = [self.extract_post_id(item[0]) for item in filtered]
        posts = self.batch_get_posts(post_ids)
        
        return posts

    async def gather_outbox_reads(self, bigv_ids, count, max_score):
        """并行读取多个大V的发件箱"""
        tasks = []
        for bigv_id in bigv_ids:
            tasks.append(self.read_outbox(bigv_id, count, max_score))
        
        results = await asyncio.gather(*tasks)
        
        # 展平结果
        items = []
        for result in results:
            items.extend(result)
        return items
    
    async def read_outbox(self, bigv_id, count, max_score):
        """读取单个大V的发件箱"""
        key = f"feed:outbox:{bigv_id}"
        if max_score:
            return await self.redis.zrevrangebyscore(
                key, max_score - 1, "-inf",
                start=0, num=count, withscores=True
            )
        else:
            return await self.redis.zrevrange(
                key, 0, count - 1, withscores=True
            )
```

### 完整的游标分页实现

Feed 分页必须使用游标分页（Cursor-based Pagination），不能用 offset 分页。原因：

1. **数据动态性**：Feed 内容实时变化，offset 分页在两次请求间如果有新帖插入，会导致重复或遗漏
2. **性能**：Redis `ZREVRANGE` 的 offset 参数在大偏移量时性能下降（需跳过前面所有元素）
3. **一致性**：游标分页基于时间戳，天然与 Feed 的时间排序语义对齐

**游标设计：**

```
游标格式：Base64(timestamp_ms + "_" + post_id_last_4chars)
示例：cx7x8Q_3f2a  →  解码后 1715000000123_3f2a

为什么游标包含 post_id 后4位？
→ 同一毫秒可能有多个帖子，仅用时间戳会导致分页不精确
→ post_id 后4位作为 tie-breaker，确保分页位置唯一确定
```

**完整的 Feed 读取服务（含游标分页）：**

```python
import base64
import time
import asyncio
from dataclasses import dataclass
from typing import Optional, List, Tuple

@dataclass
class FeedCursor:
    """分页游标"""
    timestamp_ms: int
    post_id_hint: str  # post_id 后4位，用于同一毫秒的 tie-break

    @staticmethod
    def encode(cursor: 'FeedCursor') -> str:
        raw = f"{cursor.timestamp_ms}_{cursor.post_id_hint}"
        return base64.urlsafe_b64encode(raw.encode()).decode()

    @staticmethod
    def decode(cursor_str: str) -> 'FeedCursor':
        raw = base64.urlsafe_b64decode(cursor_str.encode()).decode()
        parts = raw.split("_", 1)
        return FeedCursor(
            timestamp_ms=int(parts[0]),
            post_id_hint=parts[1] if len(parts) > 1 else ""
        )

@dataclass
class FeedItem:
    """Feed 条目"""
    post_id: str
    author_id: str
    score: float  # 时间戳（毫秒）或推荐分数

@dataclass
class FeedPage:
    """Feed 分页结果"""
    items: List[FeedItem]
    has_more: bool
    next_cursor: Optional[str]  # 下一页游标，None 表示没有更多

class FeedReadServiceV2:
    """
    完整的 Feed 读取服务，支持游标分页
    设计原则：
    1. 推拉混合读取，收件箱 + 大V发件箱合并
    2. 基于 score（时间戳）的游标分页，而非 offset
    3. 取关过滤在合并后执行，避免额外查询
    4. 超出 Redis 保留量时回溯 MySQL
    """

    INBOX_MAX_SIZE = 500        # Redis 收件箱保留条数
    OUTBOX_READ_COUNT = 30      # 每个大V发件箱读取条数（多读以应对去重和过滤后的截断）
    CANDIDATE_MULTIPLIER = 2    # 候选集倍数：读取 count*2 条候选，过滤后取 count 条
    REDIS_FALLBACK_THRESHOLD = 400  # 游标 score 超出 Redis 保留范围时回溯 MySQL

    def get_feed(self, user_id: str, count: int = 20,
                 cursor_str: Optional[str] = None) -> FeedPage:
        """
        获取用户的 Feed 列表（游标分页）

        Args:
            user_id: 用户ID
            count: 请求条数
            cursor_str: 分页游标（None 表示首页）
        """
        # 解析游标
        cursor = FeedCursor.decode(cursor_str) if cursor_str else None
        max_score = cursor.timestamp_ms if cursor else None

        # 扩大候选集以应对过滤后的截断
        fetch_count = count * self.CANDIDATE_MULTIPLIER

        # ===== 第一步：从 Redis 读取收件箱（推模型结果）=====
        inbox_items = self._read_inbox(user_id, fetch_count, max_score)

        # ===== 第二步：判断是否需要回溯 MySQL =====
        # 如果游标位置已超出 Redis 收件箱保留范围，切换到 MySQL 回溯模式
        need_mysql_fallback = False
        if max_score:
            inbox_len = self.redis.zcard(f"feed:inbox:{user_id}")
            if inbox_len > 0:
                # 获取收件箱中最旧条目的 score
                oldest_score = self.redis.zrange(
                    f"feed:inbox:{user_id}", -1, -1, withscores=True
                )
                if oldest_score and max_score < oldest_score[0][1]:
                    need_mysql_fallback = True

        if need_mysql_fallback:
            return self._get_feed_from_mysql(user_id, count, cursor)

        # ===== 第三步：读取大V发件箱（拉模型）=====
        bigv_ids = self.redis.smembers(f"feed:bigv:{user_id}")
        outbox_items = []
        if bigv_ids:
            outbox_items = asyncio.get_event_loop().run_until_complete(
                self._gather_outbox_reads(bigv_ids, fetch_count, max_score)
            )

        # ===== 第四步：合并 + 去重 + 排序 =====
        all_items = inbox_items + outbox_items
        seen = set()
        deduped = []
        for item in sorted(all_items, key=lambda x: -x[1]):  # 按 score 降序
            post_id = item[0].split(":")[0] if ":" in item[0] else item[0]
            if post_id not in seen:
                seen.add(post_id)
                deduped.append(FeedItem(
                    post_id=post_id,
                    author_id=item[0].split(":")[1] if ":" in item[0] else "",
                    score=item[1]
                ))

        # ===== 第五步：取关过滤 =====
        following_set = self.redis.smembers(f"following:{user_id}")
        filtered = [
            item for item in deduped
            if item.author_id in following_set or not item.author_id
        ]

        # ===== 第六步：截断并生成下一页游标 =====
        has_more = len(filtered) > count
        page_items = filtered[:count]

        # 生成下一页游标
        next_cursor = None
        if has_more and page_items:
            last_item = page_items[-1]
            next_cursor = FeedCursor.encode(FeedCursor(
                timestamp_ms=int(last_item.score),
                post_id_hint=last_item.post_id[-4:]
            ))

        # ===== 第七步：批量读取帖子详情 =====
        if page_items:
            post_ids = [item.post_id for item in page_items]
            posts = self._batch_get_posts(post_ids)
            # 保持排序
            post_map = {p.id: p for p in posts}
            page_items = [
                FeedItem(item.post_id, item.author_id, item.score)
                for item in page_items if item.post_id in post_map
            ]

        return FeedPage(items=page_items, has_more=has_more, next_cursor=next_cursor)

    def _read_inbox(self, user_id: str, count: int,
                    max_score: Optional[int] = None) -> List[Tuple]:
        """读取收件箱，支持游标分页"""
        key = f"feed:inbox:{user_id}"
        if max_score:
            # 游标分页：读取 score < max_score 的条目
            return self.redis.zrevrangebyscore(
                key, f"({max_score}", "-inf",  # 使用 ( 表示开区间：不含 max_score 本身
                start=0, num=count, withscores=True
            )
        else:
            # 首页：读取最新的条目
            return self.redis.zrevrange(key, 0, count - 1, withscores=True)

    def _get_feed_from_mysql(self, user_id: str, count: int,
                              cursor: FeedCursor) -> FeedPage:
        """
        MySQL 回溯模式：当游标位置超出 Redis 收件箱时使用
        场景：用户持续上拉加载，翻页超过 500 条
        """
        fetch_count = count * self.CANDIDATE_MULTIPLIER

        # 查询 feed_items 表
        rows = self.db.query(
            """
            SELECT fi.post_id, fi.author_id, fi.created_at,
                   UNIX_TIMESTAMP(fi.created_at) * 1000 AS score_ms
            FROM feed_items fi
            WHERE fi.user_id = %s
              AND fi.created_at < FROM_UNIXTIME(%s / 1000)
            ORDER BY fi.created_at DESC
            LIMIT %s
            """,
            user_id, cursor.timestamp_ms, fetch_count
        )

        items = [
            FeedItem(
                post_id=str(row.post_id),
                author_id=str(row.author_id),
                score=row.score_ms
            )
            for row in rows
        ]

        has_more = len(items) > count
        page_items = items[:count]

        next_cursor = None
        if has_more and page_items:
            last_item = page_items[-1]
            next_cursor = FeedCursor.encode(FeedCursor(
                timestamp_ms=int(last_item.score),
                post_id_hint=last_item.post_id[-4:]
            ))

        return FeedPage(items=page_items, has_more=has_more, next_cursor=next_cursor)

    async def _gather_outbox_reads(self, bigv_ids: set, count: int,
                                    max_score: Optional[int]) -> List[Tuple]:
        """并行读取多个大V的发件箱"""
        tasks = [
            self._read_outbox(bigv_id, count, max_score)
            for bigv_id in bigv_ids
        ]
        results = await asyncio.gather(*tasks, return_exceptions=True)

        items = []
        for result in results:
            if isinstance(result, Exception):
                # 大V发件箱读取失败时，跳过该大V，不影响其他数据
                self.logger.warning(f"Outbox read failed: {result}")
                continue
            items.extend(result)
        return items

    async def _read_outbox(self, bigv_id: str, count: int,
                           max_score: Optional[int]) -> List[Tuple]:
        """读取单个大V的发件箱"""
        key = f"feed:outbox:{bigv_id}"
        if max_score:
            return await self.redis.zrevrangebyscore(
                key, f"({max_score}", "-inf",
                start=0, num=count, withscores=True
            )
        else:
            return await self.redis.zrevrange(key, 0, count - 1, withscores=True)

    def _batch_get_posts(self, post_ids: List[str]) -> list:
        """批量获取帖子详情，Redis 缓存优先，MySQL 兜底"""
        # 1. 批量从 Redis MGET
        cache_keys = [f"post:{pid}" for pid in post_ids]
        cached = self.redis.mget(cache_keys)

        # 2. 找出缓存未命中的帖子
        missed_ids = []
        for i, val in enumerate(cached):
            if val is None:
                missed_ids.append(post_ids[i])

        # 3. 批量查 MySQL 补充
        if missed_ids:
            missed_posts = self.db.query(
                "SELECT * FROM posts WHERE id IN %s AND is_deleted = 0",
                (missed_ids,)
            )
            # 回写 Redis
            pipe = self.redis.pipeline()
            for post in missed_posts:
                cache_key = f"post:{post.id}"
                pipe.setex(cache_key, 86400, json.dumps(post.to_dict()))  # TTL 24h
            pipe.execute()

        # 4. 合并结果
        posts = []
        for i, val in enumerate(cached):
            if val:
                posts.append(Post.from_json(val))
            else:
                # 从 MySQL 查询结果中找
                for p in missed_posts:
                    if str(p.id) == post_ids[i]:
                        posts.append(p)
                        break
        return posts
```

**游标分页的边界情况处理：**

| 边界情况 | 处理方式 |
|---------|---------|
| 游标指向已被裁剪的旧数据 | 回溯 MySQL feed_items 表 |
| 同一毫秒多个帖子（score 相同） | 游标包含 post_id_hint 作为 tie-breaker |
| 刷新期间有新帖写入 | 游标是开区间 `({max_score}, -inf)`，不会重复 |
| 取关后翻页看到已取关用户的帖子 | 每页都执行取关过滤，实时生效 |
| 游标过期或无效 | 返回错误码，客户端回退到首页（cursor=null） |
| 大V发件箱读取超时 | 跳过该大V的数据，不影响其他结果 |

**为什么不能用 offset 分页？具体举例：**

```
时刻 T1: 用户请求第1页 offset=0, count=20 → 返回帖子 [A, B, C, ..., T]
时刻 T2: 用户浏览期间，大V发了新帖 X，被推送到收件箱（score最高）
时刻 T3: 用户请求第2页 offset=20, count=20
  → offset=20 实际上跳过了帖子 T（因为 X 插入后 T 变成了第21条）
  → 用户看到了帖子 U（本应第2页第1条），帖子 T 被遗漏

使用游标分页则不会出现此问题：
  T3: 游标=T.score → ZREVRANGEBYSCORE (T.score, -inf) → 正确返回 T 之后的帖子
```

### 读取性能的详细计算

**场景：用户关注 5 个大V + 195 个普通用户**

```
操作 1: 读取收件箱       → 1 次 Redis 调用  → 1ms
操作 2: 读取大V列表      → 1 次 Redis 调用  → 1ms
操作 3: 读取 5 个发件箱  → 5 次 Redis 调用（并行）→ 2ms
操作 4: 合并排序+去重    → 内存操作          → < 1ms
操作 5: 取关过滤         → 1 次 Redis SMEMBERS → 1ms
操作 6: 批量读取帖子详情 → 1 次 Redis MGET   → 2ms
                                      总计: ~8ms
```

**缓存未命中时：**
- 帖子详情不在 Redis → 查 MySQL → 额外 10-20ms
- 大V列表不在 Redis → 查 MySQL → 额外 5-10ms
- 总计最坏情况：~30ms

**远优于 1 秒延迟要求。**

**极端场景：用户关注 100 个大V**

- 并行读取 100 个发件箱 → 协程并行，取决于最慢的那一个 → 约 5ms
- 但 100 个 Redis 调用并发，需要连接池足够大
- 合并数据量：100 × 20 = 2000 条，排序需要 ~5ms
- 总计：~15ms

仍然远优于 1 秒。

### 取关处理的 3 种方案对比

**方案 A：不删除，读取时过滤（推荐）**

```
取关时：SREM following:{userId} {unfollowedId}
读取时：检查每条帖子的 authorId 是否在 following set 中
```

- 优点：取关操作 O(1)，不影响 Feed 性能
- 缺点：收件箱中残留历史帖子，占用存储（但量极小——取关用户每天最多发几条帖，占收件箱的极小比例）
- 问题：收件箱裁剪到 1000 条后，历史帖子自然被淘汰

**方案 B：异步批量删除**

```
取关时：发送消息到 Kafka → 消费者扫描收件箱删除该用户的帖子
```

- 优点：Feed 中不会出现已取关用户的帖子
- 缺点：扫描收件箱 O(N)，200 万用户同时取关某大V时产生 200 万次扫描
- 风险：删除操作与读取操作并发，可能导致 Feed 闪烁

**方案 C：双缓冲切换（最彻底但最复杂）**

```
取关时：标记"待清理" → 后台生成新的收件箱（不含取关用户的帖子）→ 原子切换
```

- 优点：Feed 完全不含取关用户的帖子，读取时无需过滤
- 缺点：实现极其复杂，切换期间短暂不可用
- 不推荐：过度设计

**结论：选方案 A。** 理由：取关是低频操作，残留帖子在收件箱裁剪时自然消失，读取时过滤的额外开销可忽略（1 次 SMEMBERS 查询）。

### 粉丝数跨越阈值时的迁移

**场景：某用户粉丝数从 9900 增长到 10100，跨越了 1 万的阈值。**

需要将该用户的发帖策略从"推模型"切换为"拉模型"。这是整个系统中最脆弱的操作——迁移过程中如果出现故障，可能导致帖子丢失或重复。因此必须使用事务性的原子切换方案。

**核心设计原则：双写 + 原子切换 + 兜底补偿**

迁移不是一步完成的，而是一个多阶段过程：
1. 双写阶段：推模型和拉模型同时工作，确保不丢帖
2. 原子切换：在一个事务中完成策略标记的更新
3. 兜底补偿：定时任务检查迁移状态，修复不一致

**完整的阈值迁移服务：**

```python
import enum
import logging
from datetime import datetime
from typing import Optional

class MigrationStatus(enum.IntEnum):
    PENDING = 0       # 待迁移
    DUAL_WRITE = 1    # 双写阶段：推+拉同时工作
    SWITCHING = 2     # 切换中：正在修改策略标记
    COMPLETED = 3     # 迁移完成
    FAILED = 4        # 迁移失败，需人工介入

class ThresholdMigrationService:
    """
    粉丝数跨越阈值时的迁移服务

    迁移流程（推→拉）：
    Phase 1: 双写开始 → 新帖子同时写入收件箱(推)和发件箱(拉)
    Phase 2: 回填发件箱历史数据 → 确保拉模型能读到历史帖子
    Phase 3: 更新所有粉丝的大V列表 → 确保拉模型路径可用
    Phase 4: 原子切换策略标记 → 此后新帖子只走拉模型
    Phase 5: 停止推送 → 不再向粉丝收件箱写入

    迁移流程（拉→推，用户掉粉时）：
    简化版：直接切换策略标记 + 清理大V列表，收件箱靠自然推送填充
    """

    THRESHOLD = 10000
    OUTBOX_HISTORY_COUNT = 500       # 回填发件箱的历史帖子数
    DUAL_WRITE_DURATION_SEC = 300    # 双写持续5分钟，确保边界窗口不丢帖
    MIGRATION_LOCK_TIMEOUT = 60      # 迁移锁超时60秒

    def __init__(self, redis, db, kafka_producer):
        self.redis = redis
        self.db = db
        self.kafka = kafka_producer
        self.logger = logging.getLogger("ThresholdMigration")

    # ========== 触发迁移 ==========

    def on_follower_count_changed(self, user_id: str, new_count: int):
        """
        粉丝数变化时的回调
        由 follows 表的触发器或粉丝计数器服务调用
        """
        current_strategy = self._get_strategy(user_id)

        if new_count >= self.THRESHOLD and current_strategy == "push":
            self.logger.info(
                f"User {user_id} crossed push→pull threshold "
                f"(followers={new_count}), triggering migration"
            )
            # 发送迁移事件到 Kafka，由消费者异步执行迁移
            # 不在粉丝计数的热路径上执行重操作
            self.kafka.produce(
                topic="feed-migration",
                key=user_id,
                value={
                    "user_id": user_id,
                    "direction": "push_to_pull",
                    "follower_count": new_count,
                    "triggered_at": datetime.utcnow().isoformat()
                }
            )

        elif new_count < self.THRESHOLD and current_strategy == "pull":
            # 拉→推 迁移（极少发生）
            # 检查是否频繁来回切换（"薛定谔的大V"）
            rollback_count = self._get_rollback_count(user_id)
            if rollback_count >= 3:
                self.logger.warning(
                    f"User {user_id} has rollbacked {rollback_count} times, "
                    f"locking to pull strategy permanently"
                )
                self.redis.set(f"feed:strategy:{user_id}", "pull_locked")
                return

            self.logger.info(
                f"User {user_id} crossed pull→push threshold "
                f"(followers={new_count}), triggering rollback"
            )
            self.kafka.produce(
                topic="feed-migration",
                key=user_id,
                value={
                    "user_id": user_id,
                    "direction": "pull_to_push",
                    "follower_count": new_count,
                    "triggered_at": datetime.utcnow().isoformat()
                }
            )

    # ========== 推→拉 迁移（核心流程）==========

    def migrate_push_to_pull(self, user_id: str):
        """
        从推模型切换到拉模型（完整的事务性迁移）

        关键安全保证：
        1. 迁移过程中不丢失帖子
        2. 迁移过程中不出现帖子重复（去重逻辑兜底）
        3. 迁移失败可回滚
        4. 迁移状态可追踪、可恢复
        """
        # Step 0: 获取分布式锁，防止并发迁移
        lock_key = f"migration:lock:{user_id}"
        lock_acquired = self.redis.set(
            lock_key, "locked", nx=True, ex=self.MIGRATION_LOCK_TIMEOUT
        )
        if not lock_acquired:
            self.logger.warning(f"Migration already in progress for {user_id}")
            return

        try:
            # Step 1: 创建迁移记录
            migration_id = self._create_migration_record(user_id, "push_to_pull")
            self._update_migration_status(migration_id, MigrationStatus.DUAL_WRITE)

            # Step 2: 进入双写阶段
            # 设置双写标记：此后新帖子同时走推模型和拉模型
            self.redis.set(
                f"feed:dualwrite:{user_id}", "1",
                ex=self.DUAL_WRITE_DURATION_SEC  # 双写标记自动过期
            )

            # Step 3: 回填发件箱历史数据
            # 从 MySQL 加载最近 N 条帖子到 Redis 发件箱
            self._backfill_outbox(user_id)
            self.logger.info(f"Outbox backfilled for {user_id}")

            # Step 4: 更新所有粉丝的大V列表
            # 这是最耗时的步骤，粉丝可能有 1 万+
            self._update_bigv_lists(user_id)
            self.logger.info(f"BigV lists updated for {user_id}")

            # Step 5: 等待双写稳定期
            # 双写期间，推模型仍在向粉丝收件箱推送
            # 确保拉模型路径完全可用后，再关闭推送
            import time
            time.sleep(5)  # 等待大V列表更新传播到所有 Feed 服务节点

            # Step 6: 原子切换策略标记（核心操作）
            # 使用 Redis Lua 脚本保证原子性
            self._atomic_switch_strategy(user_id, "push", "pull")
            self._update_migration_status(migration_id, MigrationStatus.SWITCHING)

            # Step 7: 验证切换结果
            # 模拟一次拉模型读取，确保数据完整
            verification_ok = self._verify_migration(user_id)
            if not verification_ok:
                raise Exception(f"Migration verification failed for {user_id}")

            # Step 8: 更新用户表的 bigv 标记
            self.db.execute(
                "UPDATE users SET is_bigv = 1, bigv_threshold_at = %s "
                "WHERE id = %s",
                datetime.utcnow(), user_id
            )

            # Step 9: 迁移完成
            self._update_migration_status(migration_id, MigrationStatus.COMPLETED)
            self.redis.delete(f"feed:dualwrite:{user_id}")
            self.logger.info(f"Migration push→pull completed for {user_id}")

        except Exception as e:
            self.logger.error(f"Migration failed for {user_id}: {e}")
            self._update_migration_status(migration_id, MigrationStatus.FAILED)
            # 迁移失败：回滚到推模型
            self._rollback_to_push(user_id)
            raise
        finally:
            # 释放分布式锁
            self.redis.delete(lock_key)

    def _atomic_switch_strategy(self, user_id: str, from_strategy: str,
                                 to_strategy: str):
        """
        原子切换策略标记

        使用 Redis Lua 脚本保证：
        1. 检查当前策略是 from_strategy（防止并发修改）
        2. 设置新策略为 to_strategy
        3. 两步在同一脚本中原子完成
        """
        lua_script = """
        local key = KEYS[1]
        local expected = ARGV[1]
        local new_value = ARGV[2]

        local current = redis.call('GET', key)
        if current == expected then
            redis.call('SET', key, new_value)
            return 1
        else
            return 0
        end
        """
        result = self.redis.eval(
            lua_script, 1,
            f"feed:strategy:{user_id}",
            from_strategy, to_strategy
        )
        if result == 0:
            raise Exception(
                f"Atomic switch failed: strategy for {user_id} "
                f"is not '{from_strategy}' (may have been changed by another process)"
            )

    def _backfill_outbox(self, user_id: str):
        """
        回填发件箱：从 MySQL 加载历史帖子到 Redis

        为什么需要回填？
        → 推模型下，用户的帖子分散在粉丝收件箱中，发件箱可能为空或只有少量数据
        → 切换到拉模型后，粉丝需要从发件箱读取帖子
        → 如果发件箱没有历史数据，粉丝会"看不到"该用户最近的帖子
        """
        recent_posts = self.db.query(
            "SELECT id, UNIX_TIMESTAMP(created_at) * 1000 AS created_at_ms "
            "FROM posts WHERE author_id = %s AND is_deleted = 0 "
            "ORDER BY created_at DESC LIMIT %s",
            user_id, self.OUTBOX_HISTORY_COUNT
        )

        if not recent_posts:
            return

        # 使用 Pipeline 批量写入
        pipe = self.redis.pipeline()
        outbox_key = f"feed:outbox:{user_id}"
        for post in recent_posts:
            pipe.zadd(outbox_key, {str(post.id): post.created_at_ms})
        # 裁剪到 500 条
        pipe.zremrangebyrank(outbox_key, 0, -(501))
        pipe.execute()

    def _update_bigv_lists(self, user_id: str):
        """
        更新所有粉丝的大V列表：将该用户加入每个粉丝的 feed:bigv:{followerId}

        这是最耗时的操作，1万粉丝需要 1万次 SADD
        优化策略：Pipeline 批量 + 分批执行
        """
        followers = self._get_all_followers(user_id)
        batch_size = 500

        for i in range(0, len(followers), batch_size):
            batch = followers[i:i + batch_size]
            pipe = self.redis.pipeline()
            for follower_id in batch:
                pipe.sadd(f"feed:bigv:{follower_id}", user_id)
            pipe.execute()

            # 每批之间短暂让出 CPU，避免阻塞 Redis
            if i + batch_size < len(followers):
                import time
                time.sleep(0.01)  # 10ms

    def _verify_migration(self, user_id: str) -> bool:
        """
        验证迁移结果：模拟一次拉模型读取

        检查项：
        1. 发件箱中有数据
        2. 策略标记已更新为 pull
        3. 至少一个粉丝的大V列表中包含该用户
        """
        # 检查策略标记
        strategy = self.redis.get(f"feed:strategy:{user_id}")
        if strategy != b"pull":
            self.logger.error(f"Strategy not updated: got {strategy}")
            return False

        # 检查发件箱
        outbox_len = self.redis.zcard(f"feed:outbox:{user_id}")
        if outbox_len == 0:
            self.logger.error(f"Outbox is empty after backfill")
            return False

        # 抽样检查粉丝的大V列表
        followers = self._get_all_followers(user_id)
        if followers:
            sample_follower = followers[0]
            is_member = self.redis.sismember(
                f"feed:bigv:{sample_follower}", user_id
            )
            if not is_member:
                self.logger.error(
                    f"BigV list not updated for follower {sample_follower}"
                )
                return False

        return True

    # ========== 拉→推 迁移（回滚）==========

    def migrate_pull_to_push(self, user_id: str):
        """
        从拉模型回滚到推模型（用户掉粉时）

        比推→拉简单得多：
        1. 切换策略标记
        2. 从粉丝的大V列表中移除该用户
        3. 此后新帖子走推模型推送

        不需要回填收件箱——新推送自然填充，历史帖子靠收件箱中残留数据 + 发件箱读取
        """
        lock_key = f"migration:lock:{user_id}"
        lock_acquired = self.redis.set(
            lock_key, "locked", nx=True, ex=self.MIGRATION_LOCK_TIMEOUT
        )
        if not lock_acquired:
            return

        try:
            # 原子切换
            self._atomic_switch_strategy(user_id, "pull", "push")

            # 从粉丝的大V列表中移除
            followers = self._get_all_followers(user_id)
            batch_size = 500
            for i in range(0, len(followers), batch_size):
                batch = followers[i:i + batch_size]
                pipe = self.redis.pipeline()
                for follower_id in batch:
                    pipe.srem(f"feed:bigv:{follower_id}", user_id)
                pipe.execute()

            # 更新用户表
            self.db.execute(
                "UPDATE users SET is_bigv = 0 WHERE id = %s", user_id
            )

            # 记录回滚次数
            self._increment_rollback_count(user_id)

            self.logger.info(f"Migration pull→push completed for {user_id}")

        except Exception as e:
            self.logger.error(f"Rollback failed for {user_id}: {e}")
            raise
        finally:
            self.redis.delete(lock_key)

    def _rollback_to_push(self, user_id: str):
        """
        推→拉迁移失败后的紧急回滚
        恢复推模型，确保系统可用性
        """
        try:
            self.redis.set(f"feed:strategy:{user_id}", "push")
            self.redis.delete(f"feed:dualwrite:{user_id}")
            self.logger.warning(
                f"Emergency rollback to push for {user_id}"
            )
        except Exception as e:
            self.logger.critical(
                f"Emergency rollback FAILED for {user_id}: {e}. "
                f"Manual intervention required!"
            )

    # ========== 兜底补偿任务 ==========

    def compensation_check(self):
        """
        定时补偿任务：每分钟执行一次
        检查是否有卡在迁移中状态的记录，尝试继续或回滚
        """
        stuck_migrations = self.db.query(
            """
            SELECT id, user_id, created_at
            FROM bigv_flags
            WHERE migration_status IN (0, 1, 2)
              AND created_at < DATE_SUB(NOW(), INTERVAL 10 MINUTE)
            """
        )

        for migration in stuck_migrations:
            self.logger.warning(
                f"Stuck migration found: id={migration.id}, "
                f"user={migration.user_id}, retrying..."
            )
            try:
                self.migrate_push_to_pull(migration.user_id)
            except Exception as e:
                self.logger.error(
                    f"Compensation retry failed for {migration.user_id}: {e}"
                )

    # ========== 辅助方法 ==========

    def _get_strategy(self, user_id: str) -> str:
        """获取当前分发策略"""
        strategy = self.redis.get(f"feed:strategy:{user_id}")
        if strategy:
            val = strategy.decode() if isinstance(strategy, bytes) else strategy
            if val == "pull_locked":
                return "pull"
            return val
        # 默认推模型
        return "push"

    def _get_all_followers(self, user_id: str) -> list:
        """获取所有粉丝ID（分页查询避免 OOM）"""
        followers = []
        page = 0
        page_size = 5000
        while True:
            batch = self.db.query(
                "SELECT follower_id FROM follows "
                "WHERE followee_id = %s AND status = 1 "
                "ORDER BY follower_id LIMIT %s OFFSET %s",
                user_id, page_size, page * page_size
            )
            if not batch:
                break
            followers.extend([str(row.follower_id) for row in batch])
            page += 1
        return followers

    def _create_migration_record(self, user_id: str, direction: str) -> int:
        """创建迁移记录"""
        result = self.db.execute(
            "INSERT INTO bigv_flags (user_id, strategy, migration_status, "
            "migration_started_at) VALUES (%s, %s, 0, %s)",
            user_id, "pull" if direction == "push_to_pull" else "push",
            datetime.utcnow()
        )
        return result.lastrowid

    def _update_migration_status(self, migration_id: int, status: MigrationStatus):
        """更新迁移状态"""
        self.db.execute(
            "UPDATE bigv_flags SET migration_status = %s WHERE id = %s",
            status, migration_id
        )

    def _get_rollback_count(self, user_id: str) -> int:
        """获取该用户的回滚次数"""
        row = self.db.query_one(
            "SELECT rollback_count FROM bigv_flags WHERE user_id = %s "
            "ORDER BY created_at DESC LIMIT 1", user_id
        )
        return row.rollback_count if row else 0

    def _increment_rollback_count(self, user_id: str):
        """增加回滚次数"""
        self.db.execute(
            "UPDATE bigv_flags SET rollback_count = rollback_count + 1 "
            "WHERE user_id = %s ORDER BY created_at DESC LIMIT 1",
            user_id
        )
```

**双写期间的 Feed 分发逻辑：**

```python
class FeedDispatchService:
    """Feed 分发服务（处理双写逻辑）"""

    def on_new_post(self, post):
        author_id = post.author_id
        strategy = self._get_strategy(author_id)
        is_dual_write = self.redis.exists(f"feed:dualwrite:{author_id}")

        if strategy == "pull" and not is_dual_write:
            # 纯拉模型：只写发件箱
            self._write_outbox(author_id, post)
            return

        if strategy == "push" and not is_dual_write:
            # 纯推模型：推送 + 写发件箱
            self._push_to_followers(author_id, post)
            self._write_outbox(author_id, post)
            return

        # 双写阶段：同时执行推送和发件箱写入
        if is_dual_write:
            self._push_to_followers(author_id, post)
            self._write_outbox(author_id, post)
            return
```

**迁移过程中的数据一致性保证总结：**

| 时间窗口 | 推模型路径 | 拉模型路径 | 一致性保证 |
|---------|----------|----------|---------|
| 迁移前 | 正常推送 | 不可用 | 仅推模型工作 |
| 双写阶段 | 正常推送 | 发件箱回填中 | 双写保证不丢帖 |
| 策略切换瞬间 | 停止推送 | 发件箱可用 | 去重逻辑处理可能重复 |
| 切换后 | 不推送 | 正常读取 | 仅拉模型工作 |
| 收件箱残留过期 | 旧推送数据自然淘汰 | 拉模型覆盖 | 无需额外处理 |

**迁移过程中的数据一致性：**
- 在策略更新前，推模型仍在工作 → 不会丢失帖子
- 在策略更新后，新帖子只写发件箱 → 粉丝通过拉模型读取
- 短暂窗口内，帖子可能同时出现在收件箱和发件箱 → 去重逻辑处理

### 算法推荐排序的架构演进

如果 Feed 从时间序改为推荐序，架构需要增加一个"重排层"：

```
原架构：Feed 服务 → 合并读取 → 时间排序 → 返回

新架构：Feed 服务 → 合并读取 → 拉取候选集(200条)
       → 推荐服务打分 → 重排 → 返回 Top 20
```

**推荐服务的设计：**

```python
class FeedRankingService:
    def rank(self, user_id, candidate_posts):
        """
        对候选帖子打分并重排
        """
        # 1. 获取用户特征
        user_features = self.get_user_features(user_id)

        # 2. 对每个帖子计算得分
        scored_posts = []
        for post in candidate_posts:
            # 帖子特征
            post_features = self.get_post_features(post.id)

            # 交互特征（用户与帖子作者的历史互动）
            interaction_features = self.get_interaction_features(
                user_id, post.author_id
            )

            # 打分
            score = self.model.predict(
                user_features, post_features, interaction_features
            )
            scored_posts.append((post, score))

        # 3. 按分数降序排序
        scored_posts.sort(key=lambda x: -x[1])

        # 4. 多样性处理（避免连续 5 条同类型帖子）
        diversified = self.diversify(scored_posts)

        return diversified[:20]

    def diversify(self, scored_posts):
        """确保 Feed 内容多样性"""
        result = []
        category_count = {}  # 连续同类帖子计数
        last_category = None

        for post, score in scored_posts:
            category = post.category
            if category == last_category:
                category_count[category] = category_count.get(category, 0) + 1
                if category_count[category] >= 3:
                    continue  # 跳过：连续 3 条同类型
            else:
                category_count = {category: 1}
                last_category = category

            result.append((post, score))

        return result
```

**性能影响：** 推荐打分增加了约 50-100ms 延迟（模型推理），总延迟从 8ms 增加到 ~110ms。仍满足 1 秒要求，但需要关注 P99 延迟。

### 内存成本计算

**收件箱：** 5000 万用户 × 1000 条 × 平均 30 字节/条 = 1.5TB
**发件箱：** 10 万大V × 500 条 × 平均 20 字节/条 = 1GB
**帖子缓存：** 热门 100 万条 × 2KB/条 = 2GB
**大V列表：** 5000 万用户 × 平均 3 个大V × 20 字节 = 3GB

**总计：约 1.5TB** —— 需要一个 Redis Cluster（6 主 6 从，每节点 256GB）。

**成本优化：**
- 收件箱裁剪到 500 条 → 内存减半至 750GB
- 帖子缓存 TTL 12 小时 → 缓存减半
- 冷门用户的收件箱不预创建（首次访问时创建）→ 再减少 30%

优化后约 400GB，6 节点 Redis Cluster（每节点 128GB）可承受。

### 性能与成本深度分析

#### Redis Cluster 内存详细计算（5000 万用户）

**收件箱内存（最大项）：**

```
每个收件箱条目结构：{postId}:{authorId}
  - postId: 8 字节 (BIGINT 字符串)
  - 分隔符 ":": 1 字节
  - authorId: 8 字节 (BIGINT 字符串)
  - Redis Sorted Set 开销: 约 64 字节/条（指针 + score + skiplist节点）
  - 每条总计: 约 81 字节

单个收件箱（500条）: 81 × 500 = 40.5 KB
5000万活跃用户: 40.5KB × 50,000,000 = 1,976 GB ≈ 1.93 TB

实际优化后：
  - 冷启动用户(30%): 不预创建收件箱 → 节省 593 GB
  - 低活跃用户(20%): 收件箱仅保留200条 → 再节省 197 GB
  - 实际收件箱内存: 1,976 - 593 - 197 = 1,186 GB ≈ 1.16 TB
```

**发件箱内存：**

```
每个发件箱条目结构：{postId}
  - postId: 8 字节
  - Redis 开销: 约 60 字节/条
  - 每条总计: 约 68 字节

10万大V × 500条: 68 × 500 × 100,000 = 3.3 GB
头部100大V × 500条: 68 × 500 × 100 = 3.3 MB (可忽略)
```

**关注关系 Redis 缓存：**

```
following:{userId}  → Set, 每个元素 8 字节 + Set开销约 32 字节
  - 人均 200 关注: (8 + 32) × 200 = 8 KB/用户
  - 5000万用户: 8KB × 50,000,000 = 381 GB

优化: 仅缓存活跃用户的关注列表(70%), 其余按需加载
  - 实际: 381 × 0.7 = 267 GB
```

**帖子详情缓存：**

```
每条帖子缓存约 2KB (JSON + Redis String 开销)
热门帖子(100万条): 2KB × 1,000,000 = 2 GB
TTL 24小时, 缓存命中率约 85%
```

**Redis Cluster 总内存预算：**

| 数据类型 | 未优化 | 优化后 |
|---------|--------|-------|
| 收件箱 | 1.93 TB | 1.16 TB |
| 发件箱 | 3.3 GB | 3.3 GB |
| 关注关系缓存 | 381 GB | 267 GB |
| 帖子详情缓存 | 2 GB | 2 GB |
| 大V列表 | 3 GB | 3 GB |
| 策略/双写标记等 | < 1 GB | < 1 GB |
| Redis自身开销(30%) | 698 GB | 430 GB |
| **总计** | **3.01 TB** | **1.87 TB** |

**推荐集群配置：**

```
方案 A（高可用）: 12 节点 Redis Cluster (6主6从)
  - 每节点: 384 GB 内存 (实际数据约 310 GB, 留 20% 水位)
  - 机型: 阿里云 redis.r6.24xlarge 或 AWS ElastiCache r6.24xlarge
  - 月成本: 约 12 × 8,000 = 96,000 元/月

方案 B（性价比）: 8 节点 Redis Cluster (4主4从)
  - 每节点: 640 GB 内存
  - 需要进一步优化收件箱（裁剪到 300 条）
  - 月成本: 约 8 × 12,000 = 96,000 元/月
  - 风险: 单节点故障影响范围更大

推荐方案 A: 更小的故障爆炸半径
```

#### 扇出写入成本分析（峰值场景）

**峰值写入量计算：**

```
峰值时段 QPS 是平峰的 3 倍:
  - 平峰发帖: 5,000 条/秒
  - 峰值发帖: 15,000 条/秒

扇出写入量 = 峰值发帖QPS × 平均扇出数
  - 普通用户平均扇出: 150 (1万阈值过滤后)
  - 峰值扇出写入: 15,000 × 150 = 225 万次 ZADD/秒

Redis Cluster 写入能力:
  - 单节点写入: ~10 万 QPS (含 Pipeline 优化)
  - 6 主节点: 60 万 QPS
  - 差距: 225万 - 60万 = 165 万 QPS 不足！
```

**解决方案：Kafka 削峰 + 异步推送**

```
架构: 发帖 → Kafka → Feed推送消费者（多实例并行）

Kafka 配置:
  - Topic: feed-push, 64 分区
  - 消费者组: feed-push-worker, 64 实例

每个推送消费者:
  - 处理速度: 约 5 万次 ZADD/秒 (Pipeline 批处理)
  - 64 实例总吞吐: 320 万次 ZADD/秒 > 225 万峰值

消息堆积应对:
  - Kafka 保留 24 小时消息
  - 消费延迟监控: > 5 秒告警, > 30 秒扩容消费者
  - 极端情况: 降级为大V帖子延迟推送(保证不丢, 允许延迟)
```

**扇出写入的 CPU 与网络成本：**

```
单次 ZADD 操作:
  - CPU: ~0.5μs (Redis 单线程处理)
  - 网络: ~100 bytes (请求 + 响应)

225万 ZADD/秒 峰值:
  - CPU: 225万 × 0.5μs = 1.125 秒/秒 → 需要分散到多个 Redis 节点
  - 网络: 225万 × 100 bytes = 213 MB/s → 需要万兆网卡

Pipeline 批处理优化:
  - 每 500 条批量: ZADD 次数/500 = 网络 RTT 减少 500 倍
  - 225万 / 500 = 4500 次 Pipeline 执行/秒
  - 网络带宽: 213 MB/s → 实际约 25 MB/s (批处理减少协议开销)
```

#### 读取延迟详细分解

**标准场景：关注 5 个大V + 195 个普通用户，首页加载**

```
步骤                        操作              延迟(P50)    延迟(P99)
─────────────────────────────────────────────────────────────────
1. 读取收件箱               ZREVRANGE          0.5ms       2ms
2. 读取大V列表              SMEMBERS            0.3ms       1ms
3. 并行读取5个发件箱         5×ZREVRANGE        1.0ms       3ms
4. 合并排序+去重            内存操作            0.1ms       0.5ms
5. 取关过滤                 SMEMBERS            0.3ms       1ms
6. 帖子详情(MGET)           MGET               1.0ms       3ms
─────────────────────────────────────────────────────────────────
总计(串行关键路径)                              ~3ms        ~10ms
```

**P99 延迟的瓶颈分析：**

```
P99 延迟主要来自:
  1. Redis 节点间的 gossip 协议占用 CPU → 解决: 专用管理网络
  2. 大 Key 的 ZREVRANGE (某用户关注了 50+ 大V) → 解决: 限制关注上限
  3. MGET 跨 slot 访问 (帖子详情分布在不同节点) → 解决: hash tag 路由

优化后 P99: ~5ms
```

**缓存未命中时的降级延迟：**

```
场景: 帖子详情不在 Redis, 需查 MySQL

步骤                        延迟(P50)    延迟(P99)
─────────────────────────────────────────────────
Redis MGET (命中部分)        1ms          3ms
MySQL 批量查询(未命中部分)    5ms          15ms
回写 Redis 缓存              1ms          2ms
─────────────────────────────────────────────────
总计                          ~7ms         ~20ms

仍远优于 1 秒延迟要求
```

**极端场景：用户关注 100 个大V**

```
步骤                        延迟(P50)    延迟(P99)
─────────────────────────────────────────────────
读取收件箱                   0.5ms        2ms
读取大V列表                  0.3ms        1ms
并行读取100个发件箱           3ms          8ms    (取决于最慢节点)
合并100×20=2000条数据        2ms          5ms
取关过滤(检查100+作者)        0.5ms        2ms
帖子详情                     1ms          3ms
─────────────────────────────────────────────────
总计                          ~7ms         ~21ms

注意: 100个并行Redis连接, 需连接池≥200
如果超过200个大V, 建议限制关注数上限或改为批量Pipeline读取
```

**MySQL 回溯场景（翻页超过 Redis 保留量）：**

```
步骤                        延迟(P50)    延迟(P99)
─────────────────────────────────────────────────
MySQL 查询 feed_items        10ms         50ms    (有索引, 但量大)
合并排序                      1ms          3ms
帖子详情                      2ms          5ms
─────────────────────────────────────────────────
总计                          ~13ms        ~58ms

用户体验: 深度翻页时可能出现短暂卡顿, 但仍在可接受范围
优化: feed_items 表按 user_id 分库, 单库数据量可控
```

## 常见陷阱（深度分析）

### 陷阱 1：纯推模型

**Twitter 的真实教训：** 2010 年前 Twitter 使用纯推模型。随着用户增长，Redis 写入量从 1000/秒飙升到 30 万/秒。每次 Lady Gaga 发推，系统需要写入 3000 万次。最终 Twitter 在 2012 年切换到推拉混合模型。

### 陷阱 2：纯拉模型

**延迟计算：** 关注 200 人，每次刷新需要 200 次 Redis 调用。即使使用 Pipeline：
- 200 次 ZREVRANGE，Pipeline 批量执行 → 约 20ms
- 但如果 200 人中有 50 人的发件箱不在 Redis 中（冷数据）→ 50 次 MySQL 查询 → 约 500ms
- 加上合并排序和帖子详情查询 → 总延迟可能超过 1 秒

### 陷阱 3：取关时遍历删除

**具体的性能问题：** 大V A 有 1000 万粉丝，某天 A 发表争议言论，100 万用户同时取关。
- 每次取关需要扫描收件箱（1000 条）→ 100 万 × 1000 = 10 亿次 Redis 操作
- 即使分布式执行也需要数小时
- 期间 Feed 服务可能被这些删除操作拖慢

### 陷阱 4：Feed 无限增长

**如果不裁剪：**
- 每天新增 5000 × 86400 = 4.32 亿条帖子
- 每个用户的收件箱每天新增约 200 × 86400 / 86400 = 200 条
- 30 天后收件箱 6000 条 × 30 字节 = 180KB/用户
- 5000 万用户 × 180KB = 9TB → Redis 集群容量不足

裁剪到 1000 条后：5000 万 × 30KB = 1.5TB，可控。

### 陷阱 5：Redis Cluster 故障期间的扇出丢失

**场景：** Redis Cluster 某主节点宕机，正在执行大V的扇出推送。

```
故障时间线：
  T0: 大V A 发帖，粉丝 500 万
  T1: Feed 推送服务开始批量 ZADD，已写入 200 万个收件箱
  T2: Redis Cluster 主节点 3 宕机（承载了约 1/6 的收件箱数据）
  T3: 剩余 300 万次 ZADD 中，约 50 万次写入节点 3 的 slot → 失败
  T4: 从节点提升为主节点（约 15 秒）
  T5: 但已失败的 50 万次写入丢失 → 50 万粉丝看不到此帖
```

**完整的容灾方案：**

```python
class ResilientFeedPushService:
    """
    带容灾的 Feed 推送服务

    设计原则：
    1. 推送失败不丢弃，写入补偿队列
    2. 定时补偿任务重试失败的推送
    3. 推模型丢失时，拉模型兜底
    """

    def push_to_followers(self, author_id: str, post):
        """推送帖子到所有粉丝，失败时写入补偿队列"""
        followers = self._get_followers(author_id)
        failed_followers = []

        for batch in self._chunk(followers, size=500):
            pipe = self.redis.pipeline()
            batch_followers = []  # 记录本批次实际的粉丝

            for follower_id in batch:
                try:
                    pipe.zadd(
                        f"feed:inbox:{follower_id}",
                        {f"{post.id}:{author_id}": post.created_at_ms}
                    )
                    pipe.zremrangebyrank(
                        f"feed:inbox:{follower_id}", 0, -(501)
                    )
                    batch_followers.append(follower_id)
                except Exception:
                    # 记录失败的单个粉丝
                    failed_followers.append(follower_id)

            try:
                pipe.execute()
            except redis.exceptions.ClusterDownError:
                # 整个批次失败，全部加入补偿队列
                failed_followers.extend(batch_followers)
                self.logger.error(
                    f"Cluster down during push, {len(batch_followers)} "
                    f"followers deferred for retry"
                )
            except redis.exceptions.ConnectionError:
                failed_followers.extend(batch_followers)

        # 将失败的推送写入补偿队列（Kafka）
        if failed_followers:
            self._enqueue_compensation(author_id, post, failed_followers)

        # 同时写入发件箱（兜底：即使推送全部失败，拉模型也能读到）
        self._write_outbox(author_id, post)

    def _enqueue_compensation(self, author_id, post, failed_followers):
        """将失败的推送写入补偿队列"""
        # 分批发送，避免单条消息过大
        for batch in self._chunk(failed_followers, size=1000):
            self.kafka.produce(
                topic="feed-push-compensation",
                key=f"{author_id}:{post.id}",
                value={
                    "post_id": post.id,
                    "author_id": author_id,
                    "created_at_ms": post.created_at_ms,
                    "failed_followers": batch,
                    "retry_count": 0,
                    "max_retries": 3,
                    "enqueued_at": datetime.utcnow().isoformat()
                }
            )

    def process_compensation(self):
        """
        补偿消费者：重试失败的推送
        由独立消费者组消费 feed-push-compensation topic
        """
        for message in self.kafka.consume("feed-push-compensation"):
            data = message.value

            if data["retry_count"] >= data["max_retries"]:
                # 超过最大重试次数，记录到死信队列
                self.logger.error(
                    f"Push compensation exhausted for post {data['post_id']}, "
                    f"{len(data['failed_followers'])} followers permanently missed"
                )
                self._write_to_dead_letter_queue(data)
                continue

            # 尝试重试推送
            still_failed = []
            for batch in self._chunk(data["failed_followers"], size=500):
                pipe = self.redis.pipeline()
                batch_ok = True
                for follower_id in batch:
                    try:
                        pipe.zadd(
                            f"feed:inbox:{follower_id}",
                            {f"{data['post_id']}:{data['author_id']}": data["created_at_ms"]}
                        )
                    except Exception:
                        batch_ok = False
                try:
                    pipe.execute()
                    if not batch_ok:
                        still_failed.extend(batch)
                except Exception:
                    still_failed.extend(batch)

            if still_failed:
                # 仍有失败，增加重试计数，重新入队
                data["retry_count"] += 1
                data["failed_followers"] = still_failed
                # 延迟重试：指数退避
                delay = 2 ** data["retry_count"]  # 2s, 4s, 8s
                self._delayed_enqueue("feed-push-compensation", data, delay)
            else:
                self.logger.info(
                    f"Push compensation succeeded for post {data['post_id']}"
                )
```

**Redis Cluster 故障恢复期间的 Feed 读取降级策略：**

```
降级层级：
  L0 (正常): 推模型收件箱 + 拉模型发件箱 → 完整 Feed
  L1 (节点故障): 仅拉模型发件箱 → 可能缺少普通用户的新帖
  L2 (集群不可用): 本地缓存(Caffeine, TTL 30s) → 展示缓存中的旧 Feed
  L3 (完全故障): 静态兜底页面 → 展示热门帖子（预先计算的全局热榜）

Feed 服务实现降级：
  1. 捕获 Redis ClusterDownError
  2. 切换到 L1: 只读大V发件箱 (不同 slot, 可能仍有可用节点)
  3. 如果所有 Redis 不可用, 切换到 L2: 本地缓存
  4. L2 也过期时, 切换到 L3: 返回全局热榜 (MySQL 预计算)
```

### 陷阱 6：大V大规模取关事件

**场景：** 大V A 发表争议言论，1 小时内 200 万粉丝取关。

```
取关风暴的连锁反应：
  T0: 大V A 粉丝 500 万（拉模型）
  T1: 争议言论发布
  T2-T3: 每秒 5000 用户取关 (200万 / 3600秒 ≈ 556/秒, 峰值可达 5000/秒)

  每个"取关"操作涉及:
    1. follows 表更新 (MySQL) → 可承受
    2. following:{userId} SREM (Redis) → 可承受 (单次 O(1))
    3. follower_count INCR (Redis) → 可承受
    4. feed:bigv:{userId} SREM (Redis) → 可承受 (单次 O(1))

  但级联效应：
    - follower_count 从 500 万降到 300 万 → 仍在 1 万阈值以上, 不触发迁移
    - 如果降到 1 万以下 → 触发拉→推迁移, 1 万粉丝的 bigv list 清理
    - 200 万用户同时刷新 Feed → 读取压力骤增 (取关后立即刷新看效果)
```

**应对方案：取关限流 + 异步处理**

```python
class UnfollowRateLimiter:
    """
    取关限流器：防止单个大V的取关操作压垮系统

    策略：
    1. 单用户取关频率限制: 同一用户 1 分钟内最多取关 10 人
    2. 大V取关全局限流: 同一大V每秒最多处理 1000 次取关
    3. 超出限流的取关请求进入队列，异步处理
    """

    def unfollow(self, follower_id: str, followee_id: str):
        # 1. 用户级限流
        user_key = f"rate:unfollow:{follower_id}"
        user_count = self.redis.incr(user_key)
        if user_count == 1:
            self.redis.expire(user_key, 60)
        if user_count > 10:
            # 进入异步队列
            self._enqueue_unfollow(follower_id, followee_id)
            return "deferred"

        # 2. 大V级限流
        bigv_key = f"rate:unfollow:bigv:{followee_id}"
        bigv_count = self.redis.incr(bigv_key)
        if bigv_count == 1:
            self.redis.expire(bigv_key, 1)  # 1 秒窗口
        if bigv_count > 1000:
            self._enqueue_unfollow(follower_id, followee_id)
            return "deferred"

        # 3. 执行取关
        self._execute_unfollow(follower_id, followee_id)
        return "ok"

    def _execute_unfollow(self, follower_id: str, followee_id: str):
        """实际执行取关操作"""
        # MySQL 更新
        self.db.execute(
            "UPDATE follows SET status = 3, unfollowed_at = %s "
            "WHERE follower_id = %s AND followee_id = %s",
            datetime.utcnow(), follower_id, followee_id
        )

        # Redis 更新
        pipe = self.redis.pipeline()
        pipe.srem(f"following:{follower_id}", followee_id)
        pipe.srem(f"feed:bigv:{follower_id}", followee_id)  # 如果是大V
        pipe.decr(f"follower:count:{followee_id}")
        pipe.execute()

        # 注意：不删除收件箱中的历史帖子（方案 A：读取时过滤）
```

**大V掉粉到阈值以下时的特殊处理：**

```
如果大V A 从 12000 粉丝降到 9800:
  1. 触发拉→推迁移
  2. 但此时可能有大量取关还在异步队列中
  3. 实际粉丝可能已降到 5000 → 迁移后推模型写入量很小
  4. 如果短时间内粉丝又涨回 1 万 → 再次迁移

"薛定谔的大V"问题：
  - 用户 A 在 1 万阈值附近反复横跳
  - 每次迁移需要更新所有粉丝的 bigv 列表 → 系统开销巨大
  - 解决: rollback_count >= 3 后锁定为拉模型, 不再降级回推模型
  - 即使粉丝降到 5000, 仍然走拉模型 (读取开销增加可忽略)
```

### 陷阱 7：帖子删除的传播

**场景：** 大V A 删除了一条帖子，但该帖子已经被推送到 100 万粉丝的收件箱中。

```
问题：
  - 推模型下，帖子副本分散在 100 万个收件箱中
  - 遍历 100 万个收件箱删除一条帖子 → 不可行
  - 不删除 → 粉丝点击后看到 "帖子已删除" 的错误页 → 体验差
```

**完整的帖子删除传播方案：**

```python
class PostDeletionService:
    """
    帖子删除处理服务

    策略：不主动从收件箱删除，而是标记删除 + 读取时过滤
    """

    def delete_post(self, post_id: str, author_id: str):
        """
        删除帖子的完整流程
        """
        # 1. MySQL 标记删除（软删除）
        self.db.execute(
            "UPDATE posts SET is_deleted = 1, deleted_at = %s WHERE id = %s",
            datetime.utcnow(), post_id
        )

        # 2. 删除帖子详情缓存
        self.redis.delete(f"post:{post_id}")

        # 3. 从发件箱中删除（如果是大V）
        self.redis.zrem(f"feed:outbox:{author_id}", post_id)

        # 4. 将删除事件写入 Kafka（用于异步清理和其他业务）
        self.kafka.produce(
            topic="post-deleted",
            key=post_id,
            value={
                "post_id": post_id,
                "author_id": author_id,
                "deleted_at": datetime.utcnow().isoformat()
            }
        )

        # 5. 删除标记集合（读取时过滤用）
        # 使用 Bloom Filter 或 Set 标记已删除的帖子
        self.redis.sadd("feed:deleted_posts", post_id)
        # 设置 TTL 48 小时（帖子删除 48 小时后，收件箱中的副本自然被裁剪淘汰）
        self.redis.expire("feed:deleted_posts", 172800)

    def is_post_deleted(self, post_id: str) -> bool:
        """
        检查帖子是否已删除
        Feed 读取服务在获取帖子详情时调用
        """
        # 先检查 Redis 删除标记集合
        if self.redis.sismember("feed:deleted_posts", post_id):
            return True

        # 如果帖子详情缓存存在且未标记删除 → 未删除
        cached = self.redis.get(f"post:{post_id}")
        if cached:
            return False

        # 缓存不存在，查 MySQL
        post = self.db.query_one(
            "SELECT is_deleted FROM posts WHERE id = %s", post_id
        )
        if post and post.is_deleted:
            # 回写删除标记
            self.redis.sadd("feed:deleted_posts", post_id)
            return True

        return False
```

**删除传播的性能分析：**

```
方案: 删除标记集合 + 读取时过滤

  删除帖子时:
    - SADD feed:deleted_posts {postId}: O(1), 约 0.1ms
    - DEL post:{postId}: O(1), 约 0.1ms
    - ZREM feed:outbox:{authorId} {postId}: O(log N), 约 0.2ms

  读取帖子详情时:
    - SISMEMBER feed:deleted_posts {postId}: O(1), 约 0.1ms
    - 额外延迟: 0.1ms (几乎可忽略)

  内存开销:
    - 删除标记集合: 每天约 1 万帖子被删除
    - 48 小时 TTL → 集合中最多约 2 万个帖子 ID
    - 每个 ID 约 20 字节 → 400 KB (可忽略)

  收件箱中的副本处理:
    - 不主动删除 → 收件箱裁剪时自然淘汰
    - 500 条收件箱, 每天新增 200 条 → 最慢 2.5 天后删除帖子被淘汰
    - 期间用户如果看到删除帖子 → 帖子详情返回 "已删除" 提示
```

### 陷阱 8：发件箱中的过期数据

**场景：** 大V A 的发件箱保留了最近 500 条帖子，但其中部分帖子已被删除或设为仅自己可见。

```
问题链：
  1. 大V A 发了帖子 P1（公开）
  2. P1 被推送到发件箱 feed:outbox:A
  3. 大V A 将 P1 改为 "仅自己可见" (visibility=3)
  4. 发件箱中仍保留 P1
  5. 粉丝读取发件箱时看到 P1 → 获取详情时被权限拦截 → 体验差

类似问题：
  - 帖子被审核删除 (spam_score 过高)
  - 帖子被作者删除
  - 帖子因版权投诉被下架
```

**解决方案：帖子详情获取时的多重过滤**

```python
class PostDetailService:
    """
    帖子详情获取服务
    统一处理可见性、删除、审核等过滤逻辑
    """

    def batch_get_visible_posts(self, user_id: str, post_ids: list) -> list:
        """
        批量获取用户可见的帖子详情

        过滤逻辑（按优先级）：
        1. 帖子是否已删除
        2. 帖子可见性权限
        3. 帖子是否被审核下架
        """
        if not post_ids:
            return []

        # 1. 批量检查删除标记
        deleted_set = self.redis.smismember("feed:deleted_posts", *post_ids)
        active_ids = [
            pid for pid, deleted in zip(post_ids, deleted_set)
            if not deleted
        ]

        if not active_ids:
            return []

        # 2. 批量获取帖子详情 (Redis MGET)
        cache_keys = [f"post:{pid}" for pid in active_ids]
        cached_posts = self.redis.mget(cache_keys)

        # 3. 补查 MySQL 未命中的帖子
        missed_ids = [
            pid for pid, cached in zip(active_ids, cached_posts)
            if cached is None
        ]

        missed_posts = {}
        if missed_ids:
            rows = self.db.query(
                "SELECT * FROM posts WHERE id IN %s AND is_deleted = 0",
                (missed_ids,)
            )
            pipe = self.redis.pipeline()
            for row in rows:
                missed_posts[str(row.id)] = row
                pipe.setex(f"post:{row.id}", 86400, json.dumps(row.to_dict()))
            pipe.execute()

        # 4. 合并结果并过滤可见性
        visible_posts = []
        for pid in active_ids:
            post = self._get_post_from_cache_or_missed(
                pid, cached_posts, missed_posts
            )
            if not post:
                continue

            # 可见性过滤
            if not self._is_visible_to(user_id, post):
                continue

            # 审核状态过滤
            if post.spam_score > 0.8:
                continue  # 高风险帖子不展示

            visible_posts.append(post)

        return visible_posts

    def _is_visible_to(self, user_id: str, post) -> bool:
        """
        检查帖子对用户是否可见
        """
        # 自己的帖子始终可见
        if post.author_id == user_id:
            return True

        if post.visibility == 1:
            # 公开帖子
            return True
        elif post.visibility == 2:
            # 仅粉丝可见
            return self.redis.sismember(
                f"followers:{post.author_id}", user_id
            )
        elif post.visibility == 3:
            # 仅自己可见
            return False

        return False
```

**过期数据的主动清理：**

```
发件箱清理策略（低优先级后台任务）：

  1. 定时扫描: 每小时扫描每个大V的发件箱
     ZREVRANGE feed:outbox:{bigvId} 0 -1 WITHSCORES
     → 检查每个 postId 是否已删除
     → 已删除的 ZREM 移除

  2. 惰性清理: 读取发件箱时，如果获取帖子详情返回空/已删除
     → ZREM 从发件箱中移除
     → 不影响读取性能（已经是获取详情的一部分）

  3. 批量清理: 每天凌晨低峰期，消费 post-deleted Kafka topic
     → 对每条删除事件，检查发件箱中是否存在
     → ZREM 移除
```

## 异常场景与容灾设计

### 场景 1：Redis Cluster 主节点宕机

**影响范围：** 约 1/6 的收件箱数据不可用（假设 6 主节点）

**故障检测与恢复时间线：**

```
T+0s:   主节点 3 宕机
T+5s:   Cluster sentinel 检测到故障
T+15s:  从节点 3' 提升为主节点
T+15-30s: 客户端刷新 slot 映射表
T+30s:  服务恢复正常

影响：
  - 推送: T+0 到 T+30s 期间，涉及节点 3 的 ZADD 失败
    → 补偿队列兜底（见陷阱 5 方案）
  - 读取: T+0 到 T+15s 期间，涉及节点 3 的收件箱/发件箱不可读
    → 降级为仅读取其他节点的数据 + 大V发件箱兜底
```

**跨机房容灾：**

```
部署架构（同城双活）：

  机房 A:
    - Redis Cluster: 主 1, 主 2, 主 3
    - Feed 服务: 实例 1-32
    - Kafka: Broker 1-4

  机房 B:
    - Redis Cluster: 从 1, 从 2, 从 3 + 主 4, 主 5, 主 6
    - Feed 服务: 实例 33-64
    - Kafka: Broker 5-8

  正常情况: 读写分散到两个机房
  机房 A 故障: 所有流量切换到机房 B
    - 从节点提升为主节点
    - Feed 服务实例扩容
    - 预计切换时间: 30-60 秒
    - 期间: 用户看到旧 Feed 数据（收件箱数据有 1-2 秒延迟）
```

### 场景 2：Kafka 分区 Leader 切换

**影响：** Feed 推送消息短暂堆积

```
T+0s:   Kafka feed-push topic 分区 5 的 Leader 宕机
T+5s:   Controller 选举新 Leader
T+10s:  消费者重新分配 partition
T+15s:  消费者从新 Leader 恢复消费

影响:
  - 分区 5 的推送延迟约 15 秒
  - 该分区对应的粉丝（约 1/64 的用户）看到新帖延迟 15 秒
  - 超出 1 秒 SLA，但属于极端故障场景，可接受

预防:
  - Kafka min.insync.replicas = 2
  - 消费者端监控消费延迟 > 5 秒告警
```

### 场景 3：全局流量突增（突发事件）

**场景：** 重大新闻事件，全平台用户同时刷新 Feed。

```
正常峰值: 30 万 QPS
突发事件: 100 万+ QPS (3 倍以上)

降级策略（逐级启用）:

  L1 - 扩容 (响应时间: 1-3 分钟)
    - Feed 服务 HPA 自动扩容
    - Kafka 消费者扩容

  L2 - 限流 (立即生效)
    - Feed 刷新接口限流: 每用户每秒 1 次
    - 超出限流返回缓存数据（304 Not Modified）

  L3 - 降级 (立即生效)
    - 关闭推荐排序，切换到纯时间序
    - 减少 Feed 拉取条数: 20 → 10
    - 跳过大V发件箱合并，仅返回收件箱数据

  L4 - 熔断 (最后手段)
    - 关闭"上拉加载更多"功能
    - 仅支持"下拉刷新"，返回固定数量的 Feed
    - 所有 Feed 请求走 CDN 缓存（TTL 10 秒）
```

## Feed 反垃圾与互动操纵检测

### 垃圾帖检测

Feed 系统面临多种垃圾内容问题，需要在分发环节进行过滤：

**1. 批量发帖检测（刷屏）**

```python
class SpamDetectionService:
    """
    Feed 反垃圾服务

    检测维度：
    1. 发帖频率异常（同一作者短时间内大量发帖）
    2. 互动操纵（刷赞、刷评论）
    3. 重复内容（同一内容反复发布）
    4. 关注链异常（互关群刷量）
    """

    # ===== 发帖频率检测 =====

    def check_post_frequency(self, author_id: str) -> dict:
        """
        检查用户发帖频率是否异常
        返回: {"is_spam": bool, "spam_score": float, "reason": str}
        """
        # 滑动窗口计数器：最近 1 小时/10 分钟/1 分钟的发帖数
        now = time.time()
        windows = {
            "1min": now - 60,
            "10min": now - 600,
            "1hour": now - 3600
        }

        post_counts = {}
        for window_name, window_start in windows.items():
            # 使用 Redis Sorted Set 记录发帖时间戳
            key = f"spam:post_freq:{author_id}"
            # 清理过期数据
            self.redis.zremrangebyscore(key, 0, window_start)
            # 统计窗口内发帖数
            post_counts[window_name] = self.redis.zcard(key)

        # 判定规则
        rules = [
            ("1min", 5, 0.9),     # 1分钟内5条 → 高度可疑
            ("1min", 3, 0.6),     # 1分钟内3条 → 中度可疑
            ("10min", 20, 0.8),   # 10分钟内20条
            ("1hour", 50, 0.7),   # 1小时内50条
        ]

        max_score = 0.0
        reason = ""
        for window, threshold, score in rules:
            if post_counts[window] >= threshold:
                if score > max_score:
                    max_score = score
                    reason = f"post_freq_{window}: {post_counts[window]}/{threshold}"

        return {
            "is_spam": max_score >= 0.6,
            "spam_score": max_score,
            "reason": reason
        }

    def record_post(self, author_id: str, post_id: str):
        """记录用户发帖（供 Feed 推送服务调用）"""
        now = time.time()
        key = f"spam:post_freq:{author_id}"
        pipe = self.redis.pipeline()
        pipe.zadd(key, {post_id: now})
        # 保留最近 1 小时数据
        pipe.zremrangebyscore(key, 0, now - 3600)
        pipe.expire(key, 3600)
        pipe.execute()

    # ===== 重复内容检测 =====

    def check_content_similarity(self, author_id: str, content: str) -> dict:
        """
        检查帖子内容是否与该用户最近发布的帖子高度相似
        使用 SimHash 或 MinHash 进行内容去重
        """
        content_hash = self._simhash(content)

        # 获取该用户最近 50 条帖子的 SimHash
        key = f"spam:content_hash:{author_id}"
        recent_hashes = self.redis.lrange(key, 0, 49)

        for stored_hash in recent_hashes:
            similarity = self._hamming_similarity(
                content_hash, stored_hash.decode()
            )
            if similarity > 0.85:
                return {
                    "is_spam": True,
                    "spam_score": similarity,
                    "reason": f"duplicate_content: similarity={similarity:.2f}"
                }

        # 记录本次内容的 SimHash
        self.redis.lpush(key, content_hash)
        self.redis.ltrim(key, 0, 49)  # 只保留最近 50 条
        self.redis.expire(key, 86400)

        return {"is_spam": False, "spam_score": 0, "reason": ""}

    def _simhash(self, text: str) -> str:
        """SimHash 算法：将文本映射为固定长度的哈希值"""
        import hashlib
        # 简化实现：使用分词 + 哈希
        words = text.split()
        hash_bits = 64 * [0]

        for word in words:
            word_hash = int(hashlib.md5(word.encode()).hexdigest(), 16)
            for i in range(64):
                if word_hash & (1 << i):
                    hash_bits[i] += 1
                else:
                    hash_bits[i] -= 1

        # 生成 SimHash 值
        result = 0
        for i in range(64):
            if hash_bits[i] > 0:
                result |= (1 << i)

        return format(result, '016x')

    def _hamming_similarity(self, hash1: str, hash2: str) -> float:
        """计算两个 SimHash 的相似度"""
        val1 = int(hash1, 16)
        val2 = int(hash2, 16)
        xor = val1 ^ val2
        hamming_dist = bin(xor).count('1')
        return 1.0 - hamming_dist / 64.0
```

**2. 互动操纵检测（刷赞/刷评论）**

```python
class EngagementManipulationDetector:
    """
    互动操纵检测器

    检测模式：
    1. 短时间大量点赞（来自同一批用户）
    2. 点赞-关注链异常（互关群集中互动）
    3. 评论内容高度重复
    """

    def check_like_manipulation(self, post_id: str) -> dict:
        """
        检测帖子是否存在刷赞行为
        """
        now = time.time()

        # 1. 短时间点赞增速异常
        like_key = f"spam:like_ts:{post_id}"
        self.redis.zremrangebyscore(like_key, 0, now - 3600)
        recent_like_count = self.redis.zcard(like_key)

        # 最近 10 分钟的点赞数
        recent_10min = self.redis.zcount(
            like_key, now - 600, now
        )

        # 正常帖子: 10分钟内点赞增速应该递减
        # 操纵帖子: 点赞匀速甚至加速（机器行为）
        total_likes = self.redis.get(f"post:like_count:{post_id}")
        total_likes = int(total_likes) if total_likes else 0

        # 2. 点赞用户群体异常
        # 采样最近 100 个点赞用户，检查他们的互动模式
        recent_liker_ids = self.redis.zrevrange(like_key, 0, 99)
        if recent_liker_ids:
            mutual_follow_rate = self._calculate_mutual_follow_rate(
                recent_liker_ids, post_id
            )
            # 互关率过高 → 刷赞嫌疑
            if mutual_follow_rate > 0.5:
                return {
                    "is_manipulated": True,
                    "confidence": 0.8,
                    "reason": f"mutual_follow_rate={mutual_follow_rate:.2f}"
                }

        # 3. 匀速点赞检测（机器行为特征）
        if recent_10min > 50:
            intervals = self._calculate_like_intervals(like_key, now - 600, now)
            if intervals:
                # 计算间隔的变异系数 (CV)
                cv = self._coefficient_of_variation(intervals)
                # 人类行为 CV 通常 > 0.5, 机器行为 CV < 0.2
                if cv < 0.2:
                    return {
                        "is_manipulated": True,
                        "confidence": 0.7,
                        "reason": f"like_interval_cv={cv:.3f} (too uniform)"
                    }

        return {"is_manipulated": False, "confidence": 0, "reason": ""}

    def _calculate_mutual_follow_rate(self, user_ids: list,
                                       post_author_id: str) -> float:
        """计算点赞用户之间的互关率"""
        mutual_count = 0
        sample_size = min(len(user_ids), 30)

        for i in range(sample_size):
            for j in range(i + 1, sample_size):
                # 检查 user_i 是否关注 user_j
                is_following = self.redis.sismember(
                    f"following:{user_ids[i]}", user_ids[j]
                )
                if is_following:
                    mutual_count += 1

        total_pairs = sample_size * (sample_size - 1) / 2
        return mutual_count / total_pairs if total_pairs > 0 else 0

    def _calculate_like_intervals(self, key: str,
                                   start: float, end: float) -> list:
        """计算点赞时间间隔"""
        timestamps = self.redis.zrangebyscore(
            key, start, end, withscores=True
        )
        if len(timestamps) < 2:
            return []

        intervals = []
        for i in range(1, len(timestamps)):
            interval = timestamps[i][1] - timestamps[i-1][1]
            intervals.append(interval)
        return intervals

    def _coefficient_of_variation(self, values: list) -> float:
        """计算变异系数 (标准差/均值)"""
        if not values:
            return 0
        mean = sum(values) / len(values)
        if mean == 0:
            return 0
        variance = sum((x - mean) ** 2 for x in values) / len(values)
        std_dev = variance ** 0.5
        return std_dev / mean
```

**3. Feed 分发时的反垃圾过滤**

```python
class FeedSpamFilter:
    """
    Feed 读取时的反垃圾过滤器

    在 Feed 读取流程中插入过滤层：
    合并排序 → 反垃圾过滤 → 帖子详情获取 → 返回

    过滤策略：
    1. 高 spam_score 帖子降权（推到 Feed 底部）
    2. 刷屏帖聚合（同一作者连续多帖合并为 "查看更多"）
    3. 操纵互动帖降权（虚假热度帖子不排在前面）
    """

    def filter_feed_items(self, user_id: str, items: list) -> list:
        """
        对 Feed 候选条目进行反垃圾过滤
        """
        filtered = []

        # 按作者分组，检测刷屏
        author_count = {}
        author_items = {}

        for item in items:
            author_id = item.author_id
            author_count[author_id] = author_count.get(author_id, 0) + 1
            if author_id not in author_items:
                author_items[author_id] = []
            author_items[author_id].append(item)

        for item in items:
            author_id = item.author_id

            # 规则 1: 刷屏聚合
            # 同一作者在 Feed 中超过 3 条，只保留前 3 条 + "查看更多"标记
            if author_count[author_id] > 3:
                author_items[author_id] = author_items[author_id][:3]
                # 在最后一条后插入 "查看更多" 聚合卡片
                # (由前端渲染)

            # 规则 2: 高 spam_score 降权
            spam_score = self.redis.get(f"spam:score:{item.post_id}")
            if spam_score and float(spam_score) > 0.7:
                # 降权：调整排序分数（降低时间戳权重）
                item.score = item.score * 0.3  # 大幅降权

            # 规则 3: 互动操纵检测标记
            manipulation = self.redis.get(
                f"spam:manipulation:{item.post_id}"
            )
            if manipulation:
                # 降权但不完全隐藏
                item.score = item.score * 0.5

            filtered.append(item)

        # 重新排序（降权后的项目可能排到后面）
        filtered.sort(key=lambda x: -x.score)

        return filtered
```

## 延伸思考

- **关注话题（超话）**：超话是一个独立的"虚拟用户"，关注超话 = 关注这个虚拟用户。帖子带有超话标签时，推送到超话的"发件箱"，用户刷新时额外合并关注超话的发件箱。
- **"看过"标记**：每个用户维护一个 Bloom Filter（`feed:seen:{userId}`），标记已读帖子 ID。刷新时过滤掉已读帖子。Bloom Filter 约 0.1% 误判率（可能漏掉未读帖子），但用户体验影响极小。
- **视频 Feed（抖音模式）**：关键差异——不是"刷列表"而是"逐条滑动"，每次只加载 1 条。预加载下 3 条即可，不需要维护收件箱。架构简化为：推荐服务直接返回下一条帖子 ID。
## Feed 流缓存架构完整实现

```python
class FeedCacheManager:
    """Feed 流多级缓存管理"""

    def get_feed(self, user_id, page_size=20, last_id=None):
        """获取用户 Feed（多级缓存）"""
        # L1: 本地内存缓存（热点用户，TTL 30s）
        cache_key = f"feed:{user_id}:{last_id or 'first'}"
        cached = self.local_cache.get(cache_key)
        if cached:
            return cached

        # L2: Redis 缓存（所有用户，TTL 60s）
        cached = self.redis.get(cache_key)
        if cached:
            feed = json.loads(cached)
            self.local_cache.set(cache_key, feed, ttl=30)
            return feed

        # L3: 数据库查询
        feed = self._query_from_db(user_id, page_size, last_id)

        # 写入缓存
        self.redis.setex(cache_key, 60, json.dumps(feed))
        self.local_cache.set(cache_key, feed, ttl=30)
        return feed

    def invalidate_feed(self, user_id):
        """Feed 缓存失效"""
        # 删除该用户的所有 Feed 缓存页
        keys = self.redis.keys(f"feed:{user_id}:*")
        if keys:
            self.redis.delete(*keys)
        self.local_cache.delete(f"feed:{user_id}")
```

## 内容审核集成

```python
class ContentModerationIntegration:
    """Feed 流内容审核集成"""

    def check_before_publish(self, post):
        """发布前审核"""
        # 1. 文本审核：敏感词 + 机器学习模型
        text_result = self.text_moderator.check(post.content)
        if text_result["action"] == "block":
            return {"allowed": False, "reason": "文本违规"}

        # 2. 图片审核：色情/暴力/政治
        for image in post.images:
            img_result = self.image_moderator.check(image)
            if img_result["action"] == "block":
                return {"allowed": False, "reason": "图片违规"}

        # 3. 通过审核 → 发布 + 入 Feed
        if text_result["action"] == "review":
            # 需人工审核 → 先发布，但标记为"审核中"
            return {"allowed": True, "status": "pending_review"}

        return {"allowed": True, "status": "approved"}
```

## 异常场景补充

### 场景：大 V 发帖导致写扩散风暴

```
触发：1000 万粉丝的大 V 发帖 → 写扩散模式需推送 1000 万个收件箱
      → Redis 写入压力巨大 → 延迟 10+ 分钟
检测：
  1. 单次写扩散目标 > 100 万 → 风暴告警
  2. Redis 写入延迟 > 1 秒 → 告警
处理：
  1. 超大 V（粉丝 > 100 万）→ 自动切换到读扩散模式
  2. 该帖子不推入收件箱 → 粉丝读取时实时拉取
  3. 设置标记：该用户的帖子走读扩散路径
预防：粉丝 > 100 万的用户自动走读扩散
```

### 场景：Feed 流内容重复

```
触发：同一内容出现在 Feed 中两次 → 用户体验差
检测：
  1. Feed 返回结果中有重复 post_id
  2. 翻页时第一页末尾 = 第二页开头（边界重复）
处理：
  1. API 层去重：返回前检查 post_id 列表
  2. 翻页去重：客户端传 last_seen_ids，服务端过滤
  3. 缓存合并：多个来源的 Feed 合并后去重
预防：API 返回前强制去重 + 翻页 cursor 包含 post_id
```

### 场景：Redis 缓存雪崩

```
触发：大量 Feed 缓存同时过期 → 请求全部打到数据库
      → 数据库连接池耗尽 → 服务不可用
检测：
  1. Redis miss rate > 50% → 告警
  2. 数据库连接池 > 90% → 严重告警
处理：
  1. 缓存 TTL 加随机偏移（60s ± 10s）避免同时过期
  2. 缓存空值：无内容的页也缓存空结果（TTL 10s）
  3. 限流：Feed API 限流 100 QPS/用户
预防：TTL 随机化 + 缓存预热 + 限流保护
```

## 推拉混合模型完整实现

```python
class HybridFeedService:
    """推拉混合模型：普通用户推，大V拉"""

    FAN_OUT_THRESHOLD = 10000  # 粉丝 > 1万 → 读扩散

    def publish_post(self, post):
        """发布帖子：推拉混合分发"""
        author = self.db.get_user(post["author_id"])
        follower_count = self.db.get_follower_count(post["author_id"])

        if follower_count <= self.FAN_OUT_THRESHOLD:
            # 写扩散：推入所有粉丝的收件箱
            self._fan_out_write(post, author)
        else:
            # 读扩散：只写入作者的发件箱
            self._fan_out_read(post, author)

        # 所有帖子都写入作者发件箱（用于拉取）
        self.redis.lpush(f"outbox:{post['author_id']}", json.dumps({
            "post_id": post["id"], "created_at": post["created_at"].isoformat()
        }))

    def _fan_out_write(self, post, author):
        """写扩散：推入每个粉丝的收件箱"""
        followers = self.db.get_followers(post["author_id"])
        pipe = self.redis.pipeline()
        for follower in followers:
            pipe.lpush(f"inbox:{follower['id']}", json.dumps({
                "post_id": post["id"],
                "author_id": post["author_id"],
                "created_at": post["created_at"].isoformat()
            }))
            pipe.ltrim(f"inbox:{follower['id']}", 0, 999)  # 保留最新 1000 条
        pipe.execute()

    def _fan_out_read(self, post, author):
        """读扩散：标记为大V帖子，读取时实时拉取"""
        self.redis.sadd("read_fanout_authors", post["author_id"])

    def get_feed(self, user_id, page_size=20, last_id=None):
        """获取 Feed：混合推拉"""
        # 1. 从收件箱获取推送内容
        pushed = self.redis.lrange(f"inbox:{user_id}", 0, page_size - 1)

        # 2. 获取关注的大V列表
        big_authors = self.redis.smembers(f"big_authors:{user_id}")

        # 3. 拉取大V最新帖子
        pulled = []
        for author_id in big_authors:
            posts = self.redis.lrange(f"outbox:{author_id}", 0, 5)
            pulled.extend([json.loads(p) for p in posts])

        # 4. 合并 + 按时间排序 + 去重
        all_posts = [json.loads(p) for p in pushed] + pulled
        all_posts.sort(key=lambda x: x["created_at"], reverse=True)
        seen = set()
        deduped = []
        for p in all_posts:
            if p["post_id"] not in seen:
                seen.add(p["post_id"])
                deduped.append(p)

        return deduped[:page_size]
```

## 热点帖子限流

```python
class HotPostRateLimiter:
    """热点帖子评论/点赞限流"""

    def check_like_rate(self, post_id):
        """检查点赞速率"""
        key = f"like_rate:{post_id}"
        count = self.redis.incr(key)
        if count == 1:
            self.redis.expire(key, 1)
        if count > 10000:  # 每秒 1 万点赞 → 热帖
            # 降级：点赞计数改为异步
            return {"mode": "async", "estimated_likes": count}
        return {"mode": "sync"}

    def check_comment_rate(self, post_id):
        """检查评论速率"""
        key = f"comment_rate:{post_id}"
        count = self.redis.incr(key)
        if count == 1:
            self.redis.expire(key, 1)
        if count > 1000:  # 每秒 1000 评论 → 热帖
            # 降级：评论先入 Kafka，异步写入
            return {"mode": "async", "queue": "hot_comments"}
        return {"mode": "sync"}
```

## 异常场景补充

### 场景：Feed 流排序不一致

```
触发：同一用户两次请求 Feed，帖子顺序不同
原因：
  1. 推拉混合 → 推送内容和拉取内容时间戳精度不同
  2. Redis 列表操作非原子 → 并发修改
处理：
  1. 统一时间戳精度为毫秒
  2. 排序时增加 post_id 作为第二排序键（稳定性）
  3. 客户端使用 cursor 分页（基于 post_id + timestamp）
预防：确定性排序键 + cursor 分页
```

### 场景：大V发帖延迟

```
触发：5000万粉丝大V发帖 → 写扩散需要 30 秒
      → 部分粉丝延迟看到
检测：
  1. 写扩散延迟 > 5 秒 → 告警
  2. 粉丝反馈"看不到新帖" → 告警
处理：
  1. 超大V（>1000万粉丝）→ 自动切换到读扩散
  2. 中等大V → 异步写扩散（先返回成功，后台逐步推入）
  3. 粉丝刷新 Feed → 拉取到大V帖子
预防：粉丝阈值自动切换 + 异步写扩散
```

## 内容推荐算法集成

```python
class FeedRecommendationService:
    """Feed 流推荐算法集成"""

    def get_recommended_feed(self, user_id, page_size=20):
        """获取推荐 Feed（协同过滤 + 内容匹配）"""
        # 1. 协同过滤：相似用户喜欢的帖子
        similar_users = self.redis.smembers(f"similar_users:{user_id}")
        cf_posts = self._get_posts_from_users(similar_users, limit=10)

        # 2. 内容匹配：基于用户兴趣标签
        user_tags = self.redis.smembers(f"user_tags:{user_id}")
        content_posts = self._get_posts_by_tags(user_tags, limit=10)

        # 3. 热门帖子补充（保证内容多样性）
        hot_posts = self._get_hot_posts(limit=5)

        # 4. 合并 + 去重 + 打散（避免同类内容连续出现）
        all_posts = cf_posts + content_posts + hot_posts
        deduped = self._dedup_and_diversify(all_posts, page_size)

        return deduped[:page_size]

    def _dedup_and_diversify(self, posts, target_size):
        """去重 + 多样性打散（MMR 策略）"""
        seen_authors = set()
        seen_tags = set()
        result = []
        for post in posts:
            if post["id"] in [p["id"] for p in result]:
                continue
            # 同一作者最多连续出现 2 次
            if post["author_id"] in seen_authors and len(seen_authors) < 3:
                continue
            result.append(post)
            seen_authors.add(post["author_id"])
            if len(result) >= target_size:
                break
        return result
```

## 性能分析详细数据

**Feed 流系统性能：**

| 操作 | QPS | 延迟 P99 | 说明 |
|------|-----|---------|------|
| 获取 Feed | 50000 | 50ms | 多级缓存 |
| 发布帖子 | 5000 | 100ms | 含写扩散 |
| 点赞/评论 | 20000 | 20ms | Redis 计数 |
| 推荐接口 | 10000 | 200ms | 含算法推理 |

**月度成本：**

| 组件 | 规格 | 月成本 |
|------|------|-------|
| 应用服务器 | 8c32G × 10台 | ¥6 万 |
| Redis 集群 | 16c64G × 6节点 | ¥4 万 |
| MySQL | 8c64G + SSD × 4台 | ¥4 万 |
| Kafka | 8c32G × 6节点 | ¥3 万 |
| Elasticsearch | 8c32G × 3节点 | ¥3 万 |
| **合计** | | **¥20 万** |

## 评论系统完整实现

```python
class CommentService:
    """评论系统：嵌套回复 + 热评排序 + 审核队列"""

    def post_comment(self, user_id, post_id, content, parent_id=None):
        """发表评论"""
        # 1. 内容审核
        moderation = self.content_moderator.check(content)
        if moderation["action"] == "reject":
            return {"status": "rejected", "reason": moderation["reason"]}

        # 2. 嵌套层级限制（最多 3 层）
        if parent_id:
            parent = self.db.get_comment(parent_id)
            depth = parent.get("depth", 0) + 1
            if depth > 3:
                # 超过 3 层 → 回复到根评论
                parent_id = parent["root_id"]
                depth = 3
            root_id = parent.get("root_id", parent_id)
        else:
            depth = 0
            root_id = None

        comment_id = str(uuid4())
        status = "pending_review" if moderation["action"] == "review" else "published"

        self.db.insert("comments", {
            "comment_id": comment_id, "post_id": post_id,
            "user_id": user_id, "content": content,
            "parent_id": parent_id, "root_id": root_id,
            "depth": depth, "status": status,
            "like_count": 0, "reply_count": 0,
            "created_at": now()
        })

        # 更新帖子评论计数
        if status == "published":
            self.redis.hincrby(f"post_stats:{post_id}", "comment_count", 1)

        # @mention 通知
        mentions = self._extract_mentions(content)
        for mentioned_user in mentions:
            self.notify(mentioned_user, f"{user_id} 在评论中提到了你")

        return {"comment_id": comment_id, "status": status}

    def get_hot_comments(self, post_id, limit=10):
        """获取热评（加权排序：点赞数 × 3 + 回复数 × 2 + 时间衰减）"""
        comments = self.db.query(
            "SELECT * FROM comments WHERE post_id = %s AND status = 'published' "
            "AND parent_id IS NULL ORDER BY "
            "(like_count * 3 + reply_count * 2) / "
            "POW(GREATEST(TIMESTAMPDIFF(HOUR, created_at, NOW()), 1), 0.5) DESC "
            "LIMIT %s", post_id, limit)
        return comments

    def get_comment_tree(self, root_id):
        """获取评论回复树"""
        replies = self.db.query(
            "SELECT * FROM comments WHERE root_id = %s AND status = 'published' "
            "ORDER BY created_at ASC", root_id)
        return self._build_tree(replies)

    def _build_tree(self, comments):
        """构建评论树"""
        tree = {}
        for c in comments:
            tree[c["comment_id"]] = {**c, "replies": []}
        roots = []
        for c in comments:
            if c["parent_id"] and c["parent_id"] in tree:
                tree[c["parent_id"]]["replies"].append(tree[c["comment_id"]])
            else:
                roots.append(tree[c["comment_id"]])
        return roots
```

## 社交关系图谱

```python
class SocialGraphService:
    """社交关系图谱：关注/粉丝/互关/推荐"""

    def follow(self, follower_id, followee_id):
        """关注用户"""
        # 1. 写入 MySQL（持久化）
        self.db.insert("social_edges", {
            "follower_id": follower_id, "followee_id": followee_id,
            "created_at": now()
        })

        # 2. 写入 Redis（快速查询）
        self.redis.sadd(f"following:{follower_id}", followee_id)
        self.redis.sadd(f"followers:{followee_id}", follower_id)

        # 3. 检查互关
        if self.redis.sismember(f"following:{followee_id}", follower_id):
            self.redis.sadd(f"mutual:{follower_id}", followee_id)
            self.redis.sadd(f"mutual:{followee_id}", follower_id)

        # 4. 更新计数
        self.redis.incr(f"following_count:{follower_id}")
        self.redis.incr(f"followers_count:{followee_id}")

    def recommend_friends(self, user_id, limit=10):
        """好友推荐（共同关注）"""
        # 找到 2 度关系：我关注的人也关注了谁
        my_following = self.redis.smembers(f"following:{user_id}")
        candidate_scores = {}
        for friend_id in my_following:
            their_following = self.redis.smembers(f"following:{friend_id}")
            for candidate in their_following:
                if candidate == user_id or candidate in my_following:
                    continue
                candidate_scores[candidate] = candidate_scores.get(candidate, 0) + 1

        # 按共同关注数排序
        sorted_candidates = sorted(candidate_scores.items(),
            key=lambda x: x[1], reverse=True)[:limit]
        return [{"user_id": uid, "common_follows": score}
                for uid, score in sorted_candidates]
```

## 异常场景补充

### 场景：评论刷量攻击

```
触发：机器人短时间内发布大量垃圾评论 → 影响正常用户
检测：
  1. 同一用户 1 分钟内评论 > 5 条 → 刷量嫌疑
  2. 同一 IP 1 分钟内评论 > 20 条 → 攻击
处理：
  1. 超频用户 → 临时禁言（5 分钟冷却）
  2. 攻击 IP → CAPTCHA 验证
  3. 已发布垃圾评论 → 批量删除
预防：评论频率限制 + 内容审核 + CAPTCHA
```

### 场景：关注关系 Redis 与 DB 不一致

```
触发：Redis SET 与 MySQL social_edges 表数据不一致
检测：
  1. 定期对账：Redis SISMEMBER vs MySQL SELECT
  2. 计数差异 > 5% → 告警
处理：
  1. 以 MySQL 为准 → 重建 Redis SET
  2. 重建过程：全量加载 social_edges → 写入 Redis
预防：双写事务 + 定期对账 + 计数一致性检查
```

## 评论系统深度实现

### 嵌套回复与评论树（parent_id 邻接表）

```python
class CommentTreeService:
    """评论树服务：支持多级嵌套回复，邻接表模型"""

    def __init__(self, db, redis, mq):
        self.db = db
        self.redis = redis
        self.mq = mq

    def create_comment(self, post_id, user_id, content, parent_id=None, mention_ids=None):
        """创建评论（支持嵌套回复，最多3层）"""
        # 1. 内容安全检测
        risk_score = self._content_risk_score(content)
        status = "pending_review" if risk_score > 0.7 else "published"

        # 2. 嵌套深度校验
        depth = 0
        root_id = None
        reply_to_user_id = None
        if parent_id:
            parent = self.db.get("comments", parent_id)
            if not parent:
                raise ValueError("父评论不存在")
            depth = parent["depth"] + 1
            if depth > 3:
                depth = 3
                root_id = parent["root_id"] or parent["id"]
            else:
                root_id = parent["root_id"] or parent["id"]
            reply_to_user_id = parent["user_id"]

        # 3. 写入评论
        comment_id = self.db.insert("comments", {
            "post_id": post_id,
            "user_id": user_id,
            "content": content,
            "parent_id": parent_id,
            "root_id": root_id,
            "reply_to_user_id": reply_to_user_id,
            "depth": depth,
            "status": status,
            "like_count": 0,
            "reply_count": 0,
            "created_at": now()
        })

        # 4. 更新帖子评论计数
        if status == "published":
            self.redis.incr(f"post:comment_count:{post_id}")
            if parent_id:
                self.db.update("comments",
                    {"reply_count": raw("reply_count + 1")},
                    {"id": parent_id})

        # 5. @提及通知
        if mention_ids:
            self._send_mention_notifications(comment_id, user_id, mention_ids, post_id)

        # 6. 回复通知
        if reply_to_user_id and reply_to_user_id != user_id:
            self.mq.publish("notification_events", {
                "type": "comment_reply",
                "to_user_id": reply_to_user_id,
                "from_user_id": user_id,
                "comment_id": comment_id,
                "post_id": post_id
            })

        # 7. 评论计数写入互动权重
        self.redis.zincrby(f"post_engagement:{post_id}", 2.0, "comment")

        return {"comment_id": comment_id, "status": status}

    def get_comment_tree(self, post_id, cursor=None, page_size=20):
        """获取评论树（两级加载：顶层评论 + 子评论预览）"""
        # 第一级：顶层评论分页
        if cursor:
            comments = self.db.query(
                "SELECT * FROM comments WHERE post_id = %s "
                "AND parent_id IS NULL AND status = 'published' "
                "AND id < %s ORDER BY id DESC LIMIT %s",
                post_id, cursor, page_size)
        else:
            comments = self.db.query(
                "SELECT * FROM comments WHERE post_id = %s "
                "AND parent_id IS NULL AND status = 'published' "
                "ORDER BY id DESC LIMIT %s",
                post_id, page_size)

        if not comments:
            return {"comments": [], "next_cursor": None}

        # 批量获取每个顶层评论的前3条子评论
        root_ids = [c["id"] for c in comments]
        sub_comments = self.db.query(
            "SELECT * FROM comments WHERE root_id IN %s "
            "AND status = 'published' ORDER BY id ASC",
            tuple(root_ids))

        sub_map = defaultdict(list)
        for sc in sub_comments:
            sub_map[sc["root_id"]].append(sc)

        result = []
        for c in comments:
            item = dict(c)
            item["replies_preview"] = sub_map.get(c["id"], [])[:3]
            item["reply_count"] = max(c.get("reply_count", 0),
                                      len(sub_map.get(c["id"], [])))
            item["has_more_replies"] = item["reply_count"] > 3
            result.append(item)

        next_cursor = comments[-1]["id"] if len(comments) == page_size else None
        return {"comments": result, "next_cursor": next_cursor}

    def get_sub_comments(self, root_id, cursor=None, page_size=20):
        """加载子评论（按root_id查询，避免递归）"""
        if cursor:
            return self.db.query(
                "SELECT * FROM comments WHERE root_id = %s "
                "AND status = 'published' AND id > %s "
                "ORDER BY id ASC LIMIT %s",
                root_id, cursor, page_size)
        return self.db.query(
            "SELECT * FROM comments WHERE root_id = %s "
            "AND status = 'published' ORDER BY id ASC LIMIT %s",
            root_id, page_size)
```

### 游标分页实现

```python
class CommentPaginationService:
    """评论游标分页：避免深度分页性能问题"""

    def __init__(self, db, redis):
        self.db = db
        self.redis = redis

    def get_comments_by_cursor(self, post_id, cursor=None, sort="time", page_size=20):
        """游标分页查询"""
        if sort == "time":
            order_col = "id"
            order_dir = "DESC"
            op = "<"
        elif sort == "hot":
            order_col = "like_count"
            order_dir = "DESC"
            op = "<="

        base_where = "post_id = %s AND parent_id IS NULL AND status = 'published'"

        if cursor:
            cursor_data = self.redis.hgetall(f"comment_cursor:{cursor}")
            if sort == "hot":
                rows = self.db.query(
                    f"SELECT * FROM comments WHERE {base_where} "
                    f"AND ({order_col}, id) {op} (%s, %s) "
                    f"ORDER BY {order_col} {order_dir}, id DESC LIMIT %s",
                    post_id,
                    cursor_data["sort_val"], cursor_data["id"],
                    page_size)
            else:
                rows = self.db.query(
                    f"SELECT * FROM comments WHERE {base_where} "
                    f"AND {order_col} {op} %s "
                    f"ORDER BY {order_col} {order_dir} LIMIT %s",
                    post_id, cursor_data["sort_val"], page_size)
        else:
            rows = self.db.query(
                f"SELECT * FROM comments WHERE {base_where} "
                f"ORDER BY {order_col} {order_dir} LIMIT %s",
                post_id, page_size)

        # 生成下一页游标
        next_cursor = None
        if len(rows) == page_size:
            next_cursor = str(uuid4())
            last = rows[-1]
            self.redis.hset(f"comment_cursor:{next_cursor}", mapping={
                "id": last["id"],
                "sort_val": last[order_col]
            })
            self.redis.expire(f"comment_cursor:{next_cursor}", 3600)

        return {"comments": rows, "next_cursor": next_cursor}
```

### 热评排序（Wilson 区间 + 时间衰减）

```python
class HotCommentRanking:
    """热评排序：Wilson区间 + 时间衰减"""

    def get_hot_comments(self, post_id, page_size=10):
        """获取热评排行"""
        # 从Redis缓存获取（5分钟更新）
        cache_key = f"hot_comments:{post_id}"
        cached = self.redis.get(cache_key)
        if cached:
            return json.loads(cached)

        # 获取高互动评论
        comments = self.db.query(
            "SELECT * FROM comments WHERE post_id = %s "
            "AND parent_id IS NULL AND status = 'published' "
            "AND like_count > 5 ORDER BY like_count DESC LIMIT 100",
            post_id)

        # 计算热度分数
        scored = []
        for c in comments:
            score = self._hot_score(c["like_count"], c["reply_count"],
                                     c["created_at"])
            scored.append((score, c))
        scored.sort(key=lambda x: x[0], reverse=True)

        result = [c for _, c in scored[:page_size]]
        self.redis.setex(cache_key, 300, json.dumps(result, default=str))
        return result

    def _hot_score(self, likes, replies, created_at, decay_hours=24):
        """热度分数 = Wilson区间 x 时间衰减"""
        n = likes + replies
        if n == 0:
            return 0
        z = 1.96
        p = likes / n
        try:
            wilson = (p + z*z/(2*n) - z * sqrt((p*(1-p)+z*z/(4*n))/n)) / (1+z*z/n)
        except ValueError:
            wilson = p

        hours_elapsed = (now() - created_at).total_seconds() / 3600
        if hours_elapsed < decay_hours:
            decay = 1.0 - 0.3 * (hours_elapsed / decay_hours)
        else:
            decay = 0.7 * (0.95 ** (hours_elapsed - decay_hours))

        return wilson * decay * 1000
```

### 评论审核队列

```python
class CommentModerationQueue:
    """评论审核队列"""

    def submit_for_review(self, comment_id, reason="auto_risk"):
        """提交审核"""
        self.db.insert("comment_review_queue", {
            "comment_id": comment_id,
            "reason": reason,
            "status": "pending",
            "priority": "high" if reason == "user_report" else "normal",
            "created_at": now()
        })
        self.mq.publish("comment_review_tasks", {
            "comment_id": comment_id,
            "priority": "high" if reason == "user_report" else "normal"
        })

    def approve_comment(self, comment_id, reviewer_id):
        """审核通过"""
        comment = self.db.get("comments", comment_id)
        self.db.update("comments",
            {"status": "published", "reviewed_by": reviewer_id},
            {"id": comment_id})
        self.db.update("comment_review_queue",
            {"status": "approved", "reviewer_id": reviewer_id},
            {"comment_id": comment_id})
        # 补计数
        self.redis.incr(f"post:comment_count:{comment['post_id']}")
        self.redis.zincrby(f"post_engagement:{comment['post_id']}", 2.0, "comment")

    def reject_comment(self, comment_id, reviewer_id, reason_code):
        """审核拒绝"""
        self.db.update("comments",
            {"status": "rejected", "reviewed_by": reviewer_id},
            {"id": comment_id})
        self.db.update("comment_review_queue",
            {"status": "rejected", "reviewer_id": reviewer_id, "reason_code": reason_code},
            {"comment_id": comment_id})
        # 累计违规（3次封禁评论功能）
        self.redis.incr(f"user_violations:comment:{comment_id}")

    def get_review_queue(self, reviewer_id, status="pending", page_size=20):
        """获取审核队列（按优先级排序）"""
        return self.db.query(
            "SELECT q.*, c.content, c.user_id, c.post_id "
            "FROM comment_review_queue q "
            "JOIN comments c ON q.comment_id = c.id "
            "WHERE q.status = %s "
            "ORDER BY FIELD(q.priority, 'high', 'normal'), q.created_at ASC "
            "LIMIT %s", status, page_size)
```

### @提及通知

```python
class MentionNotificationService:
    """@提及通知服务"""

    MENTION_PATTERN = re.compile(r'@(\w+)')

    def extract_mentions(self, content):
        """从评论内容中提取@提及的用户"""
        usernames = self.MENTION_PATTERN.findall(content)
        if not usernames:
            return []
        # 批量查询用户ID
        users = self.db.query(
            "SELECT id, username FROM users WHERE username IN %s",
            tuple(usernames))
        return [u["id"] for u in users]

    def send_mention_notifications(self, comment_id, from_user_id,
                                    mention_user_ids, post_id):
        """发送@提及通知"""
        for uid in mention_user_ids:
            # 检查用户通知偏好
            prefs = self.redis.hget(f"user_notification_prefs:{uid}", "mention")
            if prefs == "off":
                continue
            self.mq.publish("notification_events", {
                "type": "comment_mention",
                "to_user_id": uid,
                "from_user_id": from_user_id,
                "comment_id": comment_id,
                "post_id": post_id
            })

    def batch_notify_mentions(self, comment_id, from_user_id, content, post_id):
        """批量处理@提及（提取 + 通知一体化）"""
        mention_ids = self.extract_mentions(content)
        # 排除自己@自己
        mention_ids = [uid for uid in mention_ids if uid != from_user_id]
        if mention_ids:
            self.send_mention_notifications(
                comment_id, from_user_id, mention_ids, post_id)
        return mention_ids
```

## 社交关系图深度实现

### 邻接表 MySQL + Redis SET 双存储

```python
class SocialGraphDualStorage:
    """社交关系图：MySQL邻接表 + Redis SET 双存储"""

    def __init__(self, db, redis):
        self.db = db
        self.redis = redis

    def follow(self, user_id, target_id):
        """关注用户（双写保障）"""
        if user_id == target_id:
            raise ValueError("不能关注自己")

        # 1. MySQL写入（持久化，事务内记录补偿任务）
        with self.db.transaction():
            try:
                self.db.insert("user_follows", {
                    "follower_id": user_id,
                    "followee_id": target_id,
                    "created_at": now()
                })
            except DuplicateKeyError:
                return {"status": "already_followed"}
            # 记录Redis待写操作（事务内）
            self.db.insert("pending_redis_ops", {
                "op_type": "follow",
                "payload": json.dumps({
                    "follower_id": user_id,
                    "followee_id": target_id
                }),
                "status": "pending",
                "created_at": now()
            })

        # 2. Redis SET更新（查询加速）
        try:
            self._update_redis_follow(user_id, target_id)
        except RedisError:
            # Redis失败不影响主流程，补偿任务会异步修复
            pass

        # 3. 互关检测
        is_mutual = self.redis.sismember(f"following:{target_id}", user_id)
        if is_mutual:
            pipe = self.redis.pipeline()
            pipe.sadd(f"mutual_friends:{user_id}", target_id)
            pipe.sadd(f"mutual_friends:{target_id}", user_id)
            pipe.execute()

        # 4. 更新计数
        pipe = self.redis.pipeline()
        pipe.incr(f"user:following_count:{user_id}")
        pipe.incr(f"user:follower_count:{target_id}")
        pipe.execute()

        # 5. Feed流更新（写扩散）
        if self._is_small_account(target_id):
            self._fanout_recent_posts(target_id, user_id)

        return {"status": "followed", "is_mutual": is_mutual}

    def unfollow(self, user_id, target_id):
        """取关用户"""
        deleted = self.db.delete("user_follows",
            {"follower_id": user_id, "followee_id": target_id})
        if not deleted:
            return {"status": "not_following"}

        # Redis清理
        pipe = self.redis.pipeline()
        pipe.srem(f"following:{user_id}", target_id)
        pipe.srem(f"followers:{target_id}", user_id)
        pipe.srem(f"mutual_friends:{user_id}", target_id)
        pipe.srem(f"mutual_friends:{target_id}", user_id)
        pipe.decr(f"user:following_count:{user_id}")
        pipe.decr(f"user:follower_count:{target_id}")
        pipe.execute()

        # Feed流标记（查询时过滤，不立即删除）
        self.redis.setex(f"unfollow_marker:{user_id}:{target_id}", 86400 * 7, "1")
        return {"status": "unfollowed"}

    def _update_redis_follow(self, follower_id, followee_id):
        """更新Redis关注关系SET"""
        pipe = self.redis.pipeline()
        pipe.sadd(f"following:{follower_id}", followee_id)
        pipe.sadd(f"followers:{followee_id}", follower_id)
        pipe.execute()

    def _is_small_account(self, user_id):
        """判断是否为小账号（粉丝<1万，使用推模型）"""
        count = self.redis.get(f"user:follower_count:{user_id}")
        return count is None or int(count) < 10000

    def _fanout_recent_posts(self, author_id, new_follower_id):
        """将作者最新帖子推送到新粉丝的Feed"""
        recent_posts = self.db.query(
            "SELECT id, created_at FROM posts WHERE author_id = %s "
            "AND status = 'published' AND created_at > %s "
            "ORDER BY created_at DESC LIMIT 50",
            author_id, now() - timedelta(hours=48))
        if recent_posts:
            pipe = self.redis.pipeline()
            for post in recent_posts:
                pipe.zadd(f"feed:{new_follower_id}",
                    {str(post["id"]): post["created_at"].timestamp()})
            pipe.execute()
```

### 互关检测与二度人脉

```python
class SocialGraphQuery:
    """社交关系图查询：互关检测、好友推荐、二度人脉"""

    def __init__(self, db, redis):
        self.db = db
        self.redis = redis

    def get_mutual_follows(self, user_id, page_size=50):
        """获取互关好友列表"""
        # 使用Redis SET交集
        following = self.redis.smembers(f"following:{user_id}")
        followers = self.redis.smembers(f"followers:{user_id}")
        mutual_ids = following & followers

        mutual_list = list(mutual_ids)[:page_size]
        if not mutual_list:
            return {"mutual_friends": [], "total": 0}

        users = self.db.query(
            "SELECT id, nickname, avatar FROM users WHERE id IN %s",
            tuple(mutual_list))
        return {"mutual_friends": users, "total": len(mutual_ids)}

    def is_mutual_follow(self, user_id_a, user_id_b):
        """判断两个用户是否互关（O(1)）"""
        a_follows_b = self.redis.sismember(f"following:{user_id_a}", user_id_b)
        b_follows_a = self.redis.sismember(f"following:{user_id_b}", user_id_a)
        return a_follows_b and b_follows_a

    def recommend_friends(self, user_id, page_size=20):
        """好友推荐：基于共同关注数排序"""
        my_following = self.redis.smembers(f"following:{user_id}")
        if not my_following:
            return self._recommend_by_hot_users(page_size)

        # 遍历关注对象的关注列表，统计共同关注
        candidate_scores = defaultdict(int)
        for followee_id in list(my_following)[:50]:
            their_following = self.redis.smembers(f"following:{followee_id}")
            for candidate_id in their_following:
                if candidate_id == user_id or candidate_id in my_following:
                    continue
                candidate_scores[candidate_id] += 1

        ranked = sorted(candidate_scores.items(), key=lambda x: x[1], reverse=True)
        top_candidates = [uid for uid, score in ranked[:page_size * 2]]

        if not top_candidates:
            return []

        users = self.db.query(
            "SELECT id, nickname, avatar FROM users WHERE id IN %s",
            tuple(top_candidates))
        user_map = {u["id"]: u for u in users}

        result = []
        for uid, score in ranked[:page_size]:
            if uid in user_map:
                u = dict(user_map[uid])
                u["recommend_reason"] = f"{score}位共同关注"
                u["common_follow_count"] = score
                result.append(u)
        return result

    def get_second_degree_connections(self, user_id, page_size=20):
        """二度人脉：朋友的朋友"""
        first_degree = self.redis.smembers(f"following:{user_id}")

        second_degree = set()
        connection_count = defaultdict(int)

        for friend_id in list(first_degree)[:30]:
            their_friends = self.redis.smembers(f"following:{friend_id}")
            for sf in their_friends:
                if sf not in first_degree and sf != user_id:
                    second_degree.add(sf)
                    connection_count[sf] += 1

        ranked = sorted(connection_count.items(), key=lambda x: x[1], reverse=True)
        top_ids = [uid for uid, _ in ranked[:page_size]]

        if not top_ids:
            return []

        users = self.db.query(
            "SELECT id, nickname, avatar FROM users WHERE id IN %s",
            tuple(top_ids))
        user_map = {u["id"]: u for u in users}

        result = []
        for uid, count in ranked[:page_size]:
            if uid in user_map:
                u = dict(user_map[uid])
                u["connection_count"] = count
                result.append(u)
        return result

    def _recommend_by_hot_users(self, page_size):
        """冷启动推荐：热门用户"""
        hot_uids = self.redis.zrevrange("hot_users_rank", 0, page_size - 1)
        if not hot_uids:
            return []
        return self.db.query(
            "SELECT id, nickname, avatar FROM users WHERE id IN %s",
            tuple(hot_uids))
```

## 内容创作流水线深度实现

### 草稿自动保存 + 媒体上传转码

```python
class ContentCreationPipeline:
    """内容创作流水线：草稿→发布→归档→删除"""

    def __init__(self, db, redis, mq, oss_client):
        self.db = db
        self.redis = redis
        self.mq = mq
        self.oss = oss_client

    # ===== 草稿自动保存 =====

    def auto_save_draft(self, user_id, draft_id, content_delta):
        """草稿自动保存（增量保存，每5秒触发一次）"""
        draft_key = f"draft:temp:{user_id}:{draft_id}"
        current = self.redis.hgetall(draft_key) or {}
        current.update(content_delta)
        current["last_saved"] = now().isoformat()

        self.redis.hset(draft_key, mapping=current)
        self.redis.expire(draft_key, 86400 * 7)

        # 防抖：30秒内不重复写DB
        db_sync_key = f"draft:db_sync:{user_id}:{draft_id}"
        if not self.redis.exists(db_sync_key):
            self.redis.setex(db_sync_key, 30, "1")
            self._sync_draft_to_db(user_id, draft_id, current)

        return {"draft_id": draft_id, "status": "saved"}

    def _sync_draft_to_db(self, user_id, draft_id, content):
        """将草稿同步到MySQL"""
        self.db.upsert("content_drafts", {
            "id": draft_id,
            "user_id": user_id,
            "title": content.get("title", ""),
            "body": content.get("body", ""),
            "attachments": content.get("attachments", "[]"),
            "updated_at": now()
        }, conflict_key="id")

    # ===== 图片/视频上传与转码 =====

    def upload_media(self, user_id, file_data, media_type):
        """上传媒体文件"""
        media_id = str(uuid4())
        original_key = f"media/{user_id}/{media_id}/original"
        self.oss.put_object(original_key, file_data)

        self.db.insert("media_assets", {
            "id": media_id,
            "user_id": user_id,
            "type": media_type,
            "original_key": original_key,
            "status": "uploading",
            "created_at": now()
        })

        if media_type == "image":
            self.mq.publish("image_processing", {
                "media_id": media_id,
                "original_key": original_key,
                "tasks": ["compress", "thumbnail", "watermark", "content_detect"]
            })
        elif media_type == "video":
            self.mq.publish("video_processing", {
                "media_id": media_id,
                "original_key": original_key,
                "tasks": ["transcode_h264", "transcode_h265",
                           "thumbnail", "dash_packaging", "content_detect"]
            })

        return {"media_id": media_id, "status": "processing"}

    def on_image_processed(self, media_id, results):
        """图片处理完成回调"""
        updates = {
            "status": "ready",
            "compressed_key": results.get("compressed_key"),
            "thumbnail_key": results.get("thumbnail_key"),
            "width": results.get("width"),
            "height": results.get("height"),
            "file_size": results.get("compressed_size"),
            "content_detect_result": results.get("content_detect", {})
        }
        self.db.update("media_assets", updates, {"id": media_id})

        detect = results.get("content_detect", {})
        if detect.get("risk_score", 0) > 0.7:
            self.db.update("media_assets",
                {"status": "risk_blocked"}, {"id": media_id})

    def on_video_transcoded(self, media_id, results):
        """视频转码完成回调"""
        updates = {
            "status": "ready",
            "h264_key": results.get("h264_key"),
            "h265_key": results.get("h265_key"),
            "dash_manifest_key": results.get("dash_key"),
            "thumbnail_key": results.get("thumbnail_key"),
            "duration_sec": results.get("duration"),
            "width": results.get("width"),
            "height": results.get("height"),
            "content_detect_result": results.get("content_detect", {})
        }
        self.db.update("media_assets", updates, {"id": media_id})

    # ===== 内容质量评分 =====

    def calculate_quality_score(self, content):
        """计算内容质量分（0-100）"""
        score = 0
        text = content.get("body", "")

        # 文本质量（40分）
        text_score = 0
        if len(text) >= 50:
            text_score += 15
        if len(text) >= 200:
            text_score += 10
        paragraphs = text.split("\n")
        if len(paragraphs) >= 2:
            text_score += 5
        if any(p in text for p in ["。", "！", "？", "，"]):
            text_score += 5
        if not any(kw in text for kw in ["转发", "求赞", "互关"]):
            text_score += 5
        score += min(text_score, 40)

        # 媒体质量（30分）
        media_score = 0
        media_ids = content.get("media_ids", [])
        if media_ids:
            media_score += 10
            if len(media_ids) >= 3:
                media_score += 10
            media_assets = self.db.query(
                "SELECT type FROM media_assets WHERE id IN %s",
                tuple(media_ids))
            if any(m["type"] == "video" for m in media_assets):
                media_score += 10
        score += min(media_score, 30)

        # 原创性（15分）
        if not content.get("is_repost", False):
            score += 10
        if content.get("original_url"):
            score += 5
        score = min(score, 85)

        # 互动引导（15分）
        if "#" in text:
            score += 5
        if "@" in text:
            score += 5
        if content.get("poll_id"):
            score += 5

        return min(score, 100)

    # ===== 定时发布 =====

    def schedule_publish(self, content_id, scheduled_time):
        """定时发布"""
        if scheduled_time < now() + timedelta(minutes=5):
            raise ValueError("定时发布时间至少5分钟后")
        if scheduled_time > now() + timedelta(days=7):
            raise ValueError("定时发布时间不能超过7天")

        self.db.update("contents", {
            "status": "scheduled",
            "scheduled_at": scheduled_time
        }, {"id": content_id})

        delay_ms = int((scheduled_time - now()).total_seconds() * 1000)
        self.mq.publish_delayed("content_publish", {
            "content_id": content_id,
            "action": "publish"
        }, delay_ms=delay_ms)

        return {"content_id": content_id, "scheduled_at": scheduled_time.isoformat()}

    def execute_scheduled_publish(self, content_id):
        """执行定时发布"""
        content = self.db.get("contents", content_id)
        if not content or content["status"] != "scheduled":
            return {"status": "skipped"}

        if content.get("review_status") == "rejected":
            self.db.update("contents",
                {"status": "publish_failed", "fail_reason": "review_rejected"},
                {"id": content_id})
            return {"status": "failed", "reason": "review_rejected"}

        self._publish_content(content)
        return {"status": "published"}

    # ===== 内容生命周期管理 =====

    def _publish_content(self, content):
        """发布内容（draft → published）"""
        content_id = content["id"]
        author_id = content["user_id"]

        self.db.update("contents", {
            "status": "published",
            "published_at": now()
        }, {"id": content_id})

        # 写扩散：推送到粉丝Feed
        self.mq.publish("feed_fanout", {
            "content_id": content_id,
            "author_id": author_id,
            "published_at": now().isoformat()
        })

        # 清除草稿缓存
        self.redis.delete(f"draft:temp:{author_id}:{content_id}")

    def archive_content(self, content_id, user_id):
        """归档内容（published → archived）"""
        content = self.db.get("contents", content_id)
        if not content or content["user_id"] != user_id:
            raise PermissionError("无权操作")
        if content["status"] != "published":
            raise ValueError("只能归档已发布的内容")

        self.db.update("contents", {
            "status": "archived",
            "archived_at": now()
        }, {"id": content_id})

        self.redis.delete(f"post:detail:{content_id}")
        self.mq.publish("feed_remove", {"content_id": content_id})

    def delete_content(self, content_id, user_id, reason="user_delete"):
        """删除内容（任何状态 → deleted）"""
        content = self.db.get("contents", content_id)
        if not content or content["user_id"] != user_id:
            raise PermissionError("无权操作")

        self.db.update("contents", {
            "status": "deleted",
            "deleted_at": now(),
            "delete_reason": reason
        }, {"id": content_id})

        # 清理所有缓存
        keys_to_delete = [
            f"post:detail:{content_id}",
            f"post:comment_count:{content_id}",
            f"post_engagement:{content_id}",
            f"hot_comments:{content_id}"
        ]
        self.redis.delete(*keys_to_delete)
        self.mq.publish("feed_remove", {"content_id": content_id})
        self.mq.cancel_delayed("content_publish", content_id)
```

## 异常场景：评论刷量攻击（深度分析）

```
触发：黑产账号批量发布垃圾评论 → 正常用户评论被淹没 → 评论区不可用
攻击特征：
  1. 单用户1分钟内评论 > 20条
  2. 评论内容高度相似（编辑距离 < 3）
  3. 同一IP下多账号并发评论
  4. 短时间大量@不同用户
检测：
  1. 速率限制：用户级5条/分钟，IP级50条/分钟，帖子级100条/分钟
  2. 内容指纹：对评论内容simhash去重，1分钟内相同指纹>3次 → 标记
  3. 行为模式：新注册账号立即大量评论 → 风控标记
  4. 集群检测：同一IP下注册的多个账号协同评论 → 关联分析
处理：
  1. 超速率限制 → 429响应 + 验证码挑战
  2. 疑似刷量 → 评论进入审核队列，不直接展示
  3. 确认刷量 → 账号临时封禁24小时，清理所有评论
  4. 严重攻击 → IP维度封禁 + 设备指纹封禁
恢复：
  1. 清理垃圾评论 → 重新计算帖子评论计数
  2. 恢复正常评论展示顺序
  3. 将攻击特征加入风控模型（增量学习）
预防：评论速率限制 + 内容指纹去重 + 新账号评论审核 + 设备指纹关联检测
```

## 异常场景：社交关系图Redis SET与DB不一致（深度分析）

```
触发：关注操作写入MySQL成功但Redis SET更新失败 → 查询结果显示未关注
根因分析：
  1. Redis写入超时（网络抖动）→ MySQL已提交但Redis未更新
  2. 应用崩溃 → MySQL事务提交后Redis操作未执行
  3. Redis内存不足导致SET操作被拒绝
  4. 大V粉丝数百万级，SADD操作部分失败
检测：
  1. 定期对账任务：抽样1%用户，比对Redis SET与MySQL follow记录
  2. 差异率 > 0.1% → 告警
  3. 用户反馈："我明明关注了但显示未关注"
  4. 计数校验：Redis SISMEMBER总数 vs MySQL COUNT差异
处理：
  1. 紧急修复：以MySQL为准，重建差异用户的Redis SET
     - 获取差异用户列表
     - 从MySQL全量加载关注关系
     - 重建Redis SET（DEL + SADD）
  2. 修复计数：重新统计following_count / follower_count
  3. Feed流修正：差异期间的帖子可能缺失，触发Feed重建
  4. 清理pending_redis_ops补偿表中遗留的失败记录
预防：
  1. 写入顺序调整：先MySQL后Redis，失败时记录补偿任务
  2. 补偿任务表：MySQL事务内写入pending_redis_ops，异步重试
  3. Redis写入重试机制（3次重试 + 指数退避）
  4. Redis内存监控 + 预警（内存使用 > 80%告警）
  5. 每日全量对账（低峰期执行，修复遗漏差异）
  6. 双写一致性：关键操作使用MySQL事务 + Redis Pipeline原子化
```

## 内容推荐排序完整实现

```python
class FeedRankingService:
    """Feed 排序：多信号融合 + Learning-to-Rank + 多样性注入"""

    SIGNAL_WEIGHTS = {
        "recency": 0.20,       # 时效性
        "engagement_velocity": 0.20,  # 互动速度
        "author_authority": 0.15,     # 作者权威度
        "content_quality": 0.15,      # 内容质量
        "personalization": 0.20,      # 个性化
        "diversity_bonus": 0.10,      # 多样性加分
    }

    def rank_feed(self, user_id, candidate_posts, feed_size=50):
        """排序 Feed"""
        # 1. 特征提取
        features = self._extract_features(user_id, candidate_posts)

        # 2. Learning-to-Rank 模型打分
        scores = self.ltr_model.predict(features)

        # 3. 多样性注入（每 5 个位置插入不同类别）
        ranked = self._inject_diversity(
            list(zip(candidate_posts, scores)), interval=5)

        return [post for post, score in ranked[:feed_size]]

    def _extract_features(self, user_id, posts):
        """提取排序特征"""
        user_profile = self.profile_service.get_profile(user_id)
        features_list = []

        for post in posts:
            features = {}

            # 时效性：指数衰减
            hours_since = (now() - post["created_at"]).total_seconds() / 3600
            features["recency"] = math.exp(-0.1 * hours_since)

            # 互动速度：首小时互动量
            engagement = self.redis.get(f"post_engagement:{post['id']}:1h")
            features["engagement_velocity"] = float(engagement or 0) / max(hours_since, 0.1)

            # 作者权威度：粉丝数 + 认证状态
            author = self.db.get_user(post["author_id"])
            features["author_authority"] = min(1.0, math.log10(author.get("follower_count", 1) + 1) / 6)
            if author.get("verified"):
                features["author_authority"] = min(1.0, features["author_authority"] + 0.2)

            # 内容质量：媒体完整度 + 文字长度
            has_image = 1 if post.get("image_urls") else 0
            has_video = 1 if post.get("video_url") else 0
            text_len = min(1.0, len(post.get("text", "")) / 200)
            features["content_quality"] = (has_image * 0.3 + has_video * 0.4 + text_len * 0.3)

            # 个性化：用户兴趣匹配度
            post_categories = set(post.get("categories", []))
            user_interests = set(user_profile.get("interests", []))
            if post_categories and user_interests:
                overlap = len(post_categories & user_interests) / len(post_categories)
                features["personalization"] = overlap
            else:
                features["personalization"] = 0.3

            features_list.append(features)

        return features_list

    def _inject_diversity(self, scored_posts, interval=5):
        """多样性注入：每 N 个位置确保类别不同"""
        result = []
        used_categories = []

        scored_posts.sort(key=lambda x: x[1], reverse=True)
        remaining = list(scored_posts)

        while remaining and len(result) < 100:
            if len(result) % interval == 0 and len(result) > 0:
                # 插入不同类别的最高分内容
                recent_categories = used_categories[-interval:]
                diverse = [p for p in remaining
                          if p[0].get("category") not in recent_categories]
                if diverse:
                    chosen = diverse[0]
                else:
                    chosen = remaining[0]
            else:
                chosen = remaining[0]

            result.append(chosen)
            used_categories.append(chosen[0].get("category"))
            remaining.remove(chosen)

        return result
```

## 社交图谱分析

```python
class SocialGraphAnalytics:
    """社交图谱分析：社区发现 + 影响力 + 异常检测"""

    def detect_communities(self, algorithm="louvain"):
        """社区发现（Louvain 算法）"""
        # 构建图
        edges = self.db.query(
            "SELECT follower_id, followee_id FROM follows WHERE status = 'active'")

        import networkx as nx
        G = nx.DiGraph()
        for e in edges:
            G.add_edge(e["follower_id"], e["followee_id"])

        # Louvain 社区检测
        from community import best_partition
        communities = best_partition(G.to_undirected())

        # 统计社区
        community_stats = {}
        for node, comm_id in communities.items():
            if comm_id not in community_stats:
                community_stats[comm_id] = {"members": 0, "top_users": []}
            community_stats[comm_id]["members"] += 1

        # 每个社区找 Top 用户（度中心性）
        for comm_id in community_stats:
            members = [n for n, c in communities.items() if c == comm_id]
            degrees = [(m, G.degree(m)) for m in members]
            degrees.sort(key=lambda x: x[1], reverse=True)
            community_stats[comm_id]["top_users"] = degrees[:5]

        return {"num_communities": len(community_stats),
                "communities": community_stats}

    def detect_bot_rings(self):
        """检测机器人网络"""
        # 1. 找出高入度+低出度的用户（疑似僵尸粉）
        suspicious = self.db.query(
            "SELECT u.id, u.follower_count, u.following_count, "
            "u.follower_count::float / NULLIF(u.following_count, 0) as ratio "
            "FROM users u WHERE u.follower_count > 1000 "
            "AND u.following_count < 50 AND u.post_count < 5")

        bot_ring_members = []

        for user in suspicious:
            # 2. 检查该用户的粉丝是否有共同关注模式
            followers = self.db.query(
                "SELECT follower_id FROM follows WHERE followee_id = %s "
                "LIMIT 100", user["id"])

            # 3. 共同关注比例（粉丝之间互相关注率）
            mutual_follow_rate = self._calc_mutual_follow_rate(
                [f["follower_id"] for f in followers])

            if mutual_follow_rate > 0.7:
                # 70% 以上互相关注 → 高度疑似机器人网络
                bot_ring_members.append({
                    "target": user["id"],
                    "followers": len(followers),
                    "mutual_follow_rate": mutual_follow_rate
                })

        return {"bot_rings": len(bot_ring_members),
                "members": bot_ring_members}

    def _calc_mutual_follow_rate(self, user_ids):
        """计算互相关注率"""
        if len(user_ids) < 2:
            return 0
        pairs_checked = 0
        mutual_count = 0
        for i in range(min(len(user_ids), 50)):
            for j in range(i + 1, min(len(user_ids), 50)):
                pairs_checked += 1
                if self.redis.sismember(f"following:{user_ids[i]}", user_ids[j]):
                    mutual_count += 1
        return mutual_count / max(pairs_checked, 1)
```

## 异常场景补充

### 场景：排序模型延迟飙升

```
触发：LRT 模型推理延迟从 50ms 升到 500ms → Feed 加载超时
检测：
  1. 推理 P99 > 200ms → 告警
  2. Feed 接口超时率 > 1% → 严重
处理：
  1. 降级到规则排序（无模型）
  2. 检查模型服务负载和 GPU 利用率
  3. 扩容模型服务
预防：规则排序兜底 + 模型服务自动扩容 + 延迟监控
```

### 场景：社区检测 OOM

```
触发：社交图谱节点 > 1 亿 → Louvain 算法内存不足 → OOM
检测：
  1. 内存使用 > 90% → 告警
  2. 社区检测任务超时 → 可能 OOM
处理：
  1. 采样计算（随机采样 10% 节点）
  2. 分区域计算（按地域分片）
  3. 使用增量式社区检测
预防：采样计算 + 分片 + 增量算法 + 内存限制
```

## 内容病毒性预测与传播完整实现

```python
class ViralityPredictionService:
    """内容病毒性预测：早期信号 + 传播模型 + 反操纵"""

    def predict_virality(self, post_id, early_window_hours=1):
        """预测内容病毒性"""
        post = self.db.get_post(post_id)
        hours_since = (now() - post["created_at"]).total_seconds() / 3600

        if hours_since < early_window_hours:
            hours_since = max(hours_since, 0.1)

        # 1. 早期互动速度（关键信号）
        engagement = self.db.query_one(
            "SELECT COUNT(*) as likes, "
            "SUM(CASE WHEN action = 'share' THEN 1 ELSE 0 END) as shares, "
            "SUM(CASE WHEN action = 'comment' THEN 1 ELSE 0 END) as comments "
            "FROM post_actions WHERE post_id = %s "
            "AND created_at < %s",
            post_id, post["created_at"] + timedelta(hours=early_window_hours))

        engagement_velocity = (engagement["likes"] + engagement["shares"] * 3 +
                              engagement["comments"] * 2) / hours_since

        # 2. 分享-浏览比（传播意愿）
        views = self.redis.get(f"post_views:{post_id}:{early_window_hours}h")
        share_to_view = engagement["shares"] / max(int(views or 1), 1)

        # 3. 评论情感（正面情感传播更广）
        sentiment = self._analyze_comment_sentiment(post_id, early_window_hours)

        # 4. 特征向量 → 模型预测
        features = {
            "engagement_velocity": engagement_velocity,
            "share_to_view_ratio": share_to_view,
            "comment_sentiment": sentiment,
            "author_follower_count": post["author_follower_count"],
            "has_media": 1 if post.get("image_urls") else 0,
        }
        virality_score = self.virality_model.predict(features)

        # 5. 传播衰减预测
        peak_time = self._predict_peak_time(virality_score, engagement_velocity)
        decay_timeline = self._predict_decay(virality_score, peak_time)

        return {
            "post_id": post_id,
            "virality_score": round(virality_score, 3),
            "peak_predicted_at": peak_time,
            "decay_timeline": decay_timeline,
            "early_signals": features,
        }

    def amplify_viral_content(self, post_id, virality_score):
        """传播放大策略"""
        if virality_score > 0.7:
            # 高病毒性内容 → 扩大分发范围
            boost_factor = min(3.0, virality_score * 2)

            # 1. 增加在更多用户 Feed 中的曝光
            self.feed_service.boost_distribution(post_id, boost_factor)

            # 2. 推送给关注者的关注者
            author = self.db.get_post(post_id)["author_id"]
            followers = self._get_second_degree_followers(author)
            self.feed_service.inject_to_feeds(post_id, followers)

            # 3. 通知关注者（"您关注的人发布了热门内容"）
            self.notification_service.notify_trending(author, post_id)

    def detect_artificial_engagement(self, post_id):
        """检测人工操纵的互动"""
        # 短时间内大量来自同一 IP 的互动
        recent_actions = self.db.query(
            "SELECT ip_hash, COUNT(*) as count FROM post_actions "
            "WHERE post_id = %s AND created_at > NOW() - INTERVAL 1 HOUR "
            "GROUP BY ip_hash HAVING count > 5", post_id)

        # 新账号集中互动
        new_user_actions = self.db.query(
            "SELECT a.user_id, u.account_age_days FROM post_actions a "
            "JOIN users u ON a.user_id = u.id "
            "WHERE a.post_id = %s AND a.created_at > NOW() - INTERVAL 1 HOUR "
            "AND u.account_age_days < 7", post_id)

        is_artificial = (len(recent_actions) > 3 or len(new_user_actions) > 5)

        if is_artificial:
            # 移除人工互动 → 重新计算病毒性
            self._remove_artificial_actions(post_id, recent_actions, new_user_actions)
            self.alert(f"检测到人工操纵: post={post_id}")

        return {"is_artificial": is_artificial}
```

## Feed 缓存与分页

```python
class FeedCacheService:
    """Feed 缓存：预计算 + 游标分页 + 一致性"""

    def precompute_feed(self, user_id, size=200):
        """预计算用户 Feed（每 5 分钟刷新）"""
        # 从推荐系统获取候选
        candidates = self.recommendation.get_candidates(user_id, size * 2)

        # 排序 + 截断
        ranked = self.ranking.rank_feed(user_id, candidates, size)

        # 缓存（使用版本号确保一致性）
        version = int(now().timestamp())
        cache_key = f"feed_cache:{user_id}:{version}"

        for i, item in enumerate(ranked):
            self.redis.zadd(cache_key, {json.dumps(item): i})

        # 设置当前版本
        self.redis.set(f"feed_current_version:{user_id}", version)
        self.redis.expire(cache_key, 600)  # 10 分钟 TTL

    def get_feed_page(self, user_id, cursor=None, page_size=20):
        """游标分页获取 Feed"""
        version = int(self.redis.get(f"feed_current_version:{user_id}") or 0)
        cache_key = f"feed_cache:{user_id}:{version}"

        # 游标 = 上次读取的位置
        start = int(cursor or 0)
        items = self.redis.zrange(cache_key, start, start + page_size - 1)

        # 解析
        feed = [json.loads(item) for item in items]

        # 无重复保证：跨页去重
        if cursor:
            previous_ids = self.redis.get(f"feed_seen:{user_id}:{version}")
            seen = set(json.loads(previous_ids or "[]"))
            feed = [item for item in feed if item["id"] not in seen]

        # 记录已看 ID
        seen_ids = set(item["id"] for item in feed)
        if cursor:
            seen_ids |= set(json.loads(
                self.redis.get(f"feed_seen:{user_id}:{version}") or "[]"))
        self.redis.setex(f"feed_seen:{user_id}:{version}", 600,
            json.dumps(list(seen_ids)[:page_size * 5]))

        next_cursor = start + page_size if len(feed) == page_size else None

        return {"items": feed, "cursor": next_cursor, "version": version}

    def invalidate_cache(self, user_id, reason):
        """缓存失效"""
        if reason == "new_follow":
            # 新关注 → 需要重新计算
            self.redis.delete(f"feed_current_version:{user_id}")
            self.precompute_feed(user_id)
        elif reason == "new_post_from_followed":
            # 关注者发新帖 → 插入到缓存顶部
            # 不需要全量重算，直接插入
            pass
```

## 异常场景补充

### 场景：病毒性预测制造回音壁

```
触发：仅放大已热门的内容 → 中小创作者永远得不到曝光 → 回音壁
检测：
  1. 新创作者曝光率持续下降 → 回音壁
  2. Top 10 创作者占据 > 50% 曝光 → 集中度过高
处理：
  1. 新创作者曝光配额（至少 10% 给 <1000 粉丝创作者）
  2. 病毒性放大仅对 <5000 粉丝的创作者生效
  3. 热门内容放大上限
预防：创作者曝光配额 + 放大上限 + 新创作者扶持
```

### 场景：Feed 缓存雪崩

```
触发：明星发帖 → 数百万粉丝 Feed 缓存同时失效 → 缓存雪崩 → 数据库过载
检测：
  1. 缓存命中率骤降 → 雪崩
  2. 数据库 QPS 飙升 → 过载
处理：
  1. 热门帖子不触发缓存失效 → 直接插入到现有缓存
  2. 缓存重建限流（每秒最多重建 1000 个用户）
  3. 降级到热门列表
预防：插入而非失效 + 重建限流 + 多级缓存
```

## 社交关系链优化完整实现

```python
class SocialGraphOptimizer:
    """社交关系链优化：分片 + 缓存 + 读写分离"""

    def get_followers(self, user_id, page=1, page_size=50):
        """获取粉丝列表（分片 + 缓存）"""
        # 1. 缓存检查
        cache_key = f"followers:{user_id}:{page}"
        cached = self.redis.get(cache_key)
        if cached:
            return json.loads(cached)

        # 2. 分片查询（按 user_id hash 分片）
        shard = self._get_shard(user_id)
        followers = shard.query(
            "SELECT follower_id, created_at FROM follows "
            "WHERE followee_id = %s AND status = 'active' "
            "ORDER BY created_at DESC LIMIT %s OFFSET %s",
            user_id, page_size, (page - 1) * page_size)

        # 3. 补充用户信息
        result = []
        for f in followers:
            user = self.user_cache.get(f["follower_id"])
            result.append({
                "user_id": f["follower_id"],
                "username": user.get("username", ""),
                "avatar": user.get("avatar_url", ""),
                "followed_at": f["created_at"].isoformat()
            })

        # 4. 缓存（1 分钟）
        self.redis.setex(cache_key, 60, json.dumps(result))

        return result

    def follow(self, follower_id, followee_id):
        """关注操作（写入主库 + 更新缓存 + 发通知）"""
        # 1. 写入主库
        self.db.insert("follows", {
            "follower_id": follower_id,
            "followee_id": followee_id,
            "status": "active",
            "created_at": now()
        })

        # 2. 更新缓存（粉丝数 + 关注数）
        self.redis.hincrby(f"user_stats:{followee_id}", "follower_count", 1)
        self.redis.hincrby(f"user_stats:{follower_id}", "following_count", 1)

        # 3. 清除粉丝列表缓存
        pattern = f"followers:{followee_id}:*"
        for key in self.redis.keys(pattern):
            self.redis.delete(key)

        # 4. 通知被关注者
        self.notification_service.send(followee_id,
            f"用户 {follower_id} 关注了你", "new_follower")

        # 5. 更新 Feed 缓存（关注者的新帖需要出现在 Feed 中）
        self.feed_cache_service.invalidate(follower_id, "new_follow")

        return {"status": "followed"}

    def batch_follow(self, follower_id, followee_ids):
        """批量关注（从通讯录导入等场景）"""
        # 限流：每次最多 100 个
        followee_ids = followee_ids[:100]

        success = 0
        for followee_id in followee_ids:
            try:
                self.follow(follower_id, followee_id)
                success += 1
            except DuplicateFollowError:
                continue

        # 批量操作后批量清除缓存
        self.feed_cache_service.invalidate(follower_id, "batch_follow")

        return {"requested": len(followee_ids), "succeeded": success}

    def _get_shard(self, user_id):
        """获取数据库分片"""
        shard_index = int(hashlib.md5(user_id.encode()).hexdigest(), 16) % len(self.shards)
        return self.shards[shard_index]
```

## 社交内容安全与隐私

```python
class SocialPrivacyService:
    """社交隐私：可见性控制 + 屏蔽 + 举报"""

    VISIBILITY_LEVELS = {
        "public": "所有人可见",
        "friends": "仅好友可见",
        "private": "仅自己可见",
    }

    def check_visibility(self, viewer_id, content_owner_id, visibility):
        """检查内容可见性"""
        if visibility == "public":
            return True
        elif visibility == "friends":
            return self._are_friends(viewer_id, content_owner_id)
        elif visibility == "private":
            return viewer_id == content_owner_id
        return False

    def block_user(self, blocker_id, blocked_id):
        """屏蔽用户"""
        # 双向屏蔽：互相关注取消 + 互不可见
        self.db.insert("user_blocks", {
            "blocker_id": blocker_id,
            "blocked_id": blocked_id,
            "created_at": now()
        })

        # 取消互相关注
        self.db.execute(
            "DELETE FROM follows WHERE "
            "(follower_id = %s AND followee_id = %s) OR "
            "(follower_id = %s AND followee_id = %s)",
            blocker_id, blocked_id, blocked_id, blocker_id)

        # 清除缓存
        self.redis.delete(f"user_stats:{blocker_id}")
        self.redis.delete(f"user_stats:{blocked_id}")

    def report_content(self, reporter_id, content_id, reason, description=None):
        """举报内容"""
        report_id = str(uuid4())
        self.db.insert("content_reports", {
            "report_id": report_id,
            "reporter_id": reporter_id,
            "content_id": content_id,
            "reason": reason,  # spam / harassment / misinformation / inappropriate
            "description": description,
            "status": "pending",
            "created_at": now()
        })

        # 同一内容被举报次数
        report_count = self.db.count("content_reports", content_id=content_id)

        # 累计举报 > 10 次 → 自动下架待审
        if report_count >= 10:
            self.db.update("content",
                {"status": "under_review"},
                {"id": content_id})
            self.moderation_service.prioritize(content_id)

        return {"report_id": report_id, "report_count": report_count}
```

## 异常场景补充

### 场景：关系链分片不均衡

```
触发：大 V 用户（1 亿粉丝）所在分片过热 → 其他分片空闲 → 性能瓶颈
检测：
  1. 某分片 QPS 远高于其他 → 热点
  2. 大 V 粉丝查询延迟 > 1 秒 → 过热
处理：
  1. 大 V 粉丝单独存储（独立缓存层）
  2. 粉丝数 > 100 万 → 读写分离 + 缓存优先
  3. 冷热分片：活跃粉丝单独缓存
预防：大 V 特殊处理 + 多级缓存 + 异步写入
```

### 场景：批量关注被滥用

```
触发：用户通过 API 批量关注 100 人/次 → 被用于刷粉 → 垃圾关注
检测：
  1. 单用户关注频率异常（> 200/小时）→ 刷粉
  2. 关注-取关循环 → 刷粉行为
处理：
  1. 限流：每小时最多关注 50 人
  2. 检测异常模式 → 封禁
  3. 清除垃圾关注
预防：关注频率限制 + 异常检测 + 人机验证
```

## 社交内容安全检测完整实现

```python
class SocialContentSafetyService:
    """社交内容安全：自残检测 + 网络欺凌 + 未成年保护"""

    def detect_self_harm(self, content):
        """检测自残/自杀倾向"""
        indicators = []

        # 1. 关键词检测
        self_harm_keywords = ["不想活", "结束生命", "自杀", "自残", "跳楼", "割腕"]
        text = content.get("text", "").lower()
        for kw in self_harm_keywords:
            if kw in text:
                indicators.append({"type": "keyword", "value": kw})

        # 2. 图片检测（自残图片）
        if content.get("image_urls"):
            for img_url in content["image_urls"]:
                result = self.vision_model.classify(img_url,
                    categories=["self_harm", "cutting", "suicide_note"])
                if result.get("self_harm", 0) > 0.7:
                    indicators.append({"type": "image", "confidence": result["self_harm"]})

        # 3. 上下文分析（近期情绪趋势）
        user_id = content["author_id"]
        recent_posts = self.db.query(
            "SELECT text FROM posts WHERE author_id = %s "
            "AND created_at > NOW() - INTERVAL 7 DAY "
            "ORDER BY created_at DESC LIMIT 10", user_id)
        sentiment_trend = self._analyze_sentiment_trend(recent_posts)
        if sentiment_trend["declining"] and sentiment_trend["current_score"] < -0.5:
            indicators.append({"type": "sentiment_decline",
                "score": sentiment_trend["current_score"]})

        # 4. 触发干预
        if indicators:
            self._trigger_crisis_intervention(user_id, indicators)

        return {"risk_level": "high" if len(indicators) >= 2 else
                "medium" if indicators else "low",
                "indicators": indicators}

    def _trigger_crisis_intervention(self, user_id, indicators):
        """触发危机干预"""
        # 1. 隐藏相关内容（不删除，保护用户隐私）
        # 2. 提供心理援助热线
        # 3. 通知专业团队
        self.db.insert("crisis_interventions", {
            "user_id": user_id,
            "indicators": json.dumps(indicators),
            "status": "active",
            "created_at": now()
        })

        # 发送心理援助信息给用户
        self.notification.send(user_id,
            "如果您正在经历困难时刻，请拨打心理援助热线：400-161-9995",
            "crisis_support", priority="urgent")

    def detect_cyberbullying(self, content):
        """检测网络欺凌"""
        indicators = []

        # 1. 辱骂性语言
        insult_score = self.nlp_model.classify(content["text"], "insult")
        if insult_score > 0.7:
            indicators.append({"type": "insult", "score": insult_score})

        # 2. 针对性攻击（@某人 + 辱骂）
        mentions = re.findall(r'@(\w+)', content["text"])
        if mentions and insult_score > 0.5:
            indicators.append({"type": "targeted_attack", "targets": mentions})

        # 3. 群体性攻击（多人针对同一用户）
        if mentions:
            for target in mentions:
                recent_attacks = self.db.count("content_flags",
                    target_user=target, flag_type="bullying",
                    created_at__gte=now()-timedelta(hours=24))
                if recent_attacks >= 3:
                    indicators.append({"type": "mob_attack",
                        "target": target, "attacker_count": recent_attacks})

        # 4. 图片欺凌（P图丑化）
        if content.get("image_urls") and mentions:
            for img_url in content["image_urls"]:
                result = self.vision_model.classify(img_url,
                    categories=["mockery", "body_shaming", "doctored_image"])
                if any(result.get(k, 0) > 0.6 for k in result):
                    indicators.append({"type": "image_bullying"})

        return {"is_bullying": len(indicators) > 0, "indicators": indicators}
```

## 异常场景补充

### 场景：自残检测误报

```
触发：用户讨论心理健康话题 → 被误判为自残倾向 → 内容被隐藏 → 用户愤怒
检测：
  1. 用户申诉内容被不当隐藏 → 误报
  2. 心理健康讨论区误报率高 → 阈值过严
处理：
  1. 区分"讨论心理健康"和"表达自残倾向"
  2. 心理健康标签内容使用更宽松的阈值
  3. 人工复核高风险判定
预防：上下文分析 + 话题区分 + 人工复核
```

### 场景：网络欺凌检测遗漏

```
触发：使用隐晦语言欺凌 → 关键词和模型都未检测到 → 持续伤害
检测：
  1. 被欺凌用户投诉率高但系统未检测 → 遗漏
  2. 某用户频繁收到负面评论 → 可能被欺凌
处理：
  1. 加入被欺凌者视角检测（某用户频繁收到负面评论）
  2. 扩大模型训练数据（包含隐晦语言）
  3. 用户举报作为补充信号
预防：多角度检测 + 模型迭代 + 举报补充
```

## 社交内容推荐 Feed 流去重完整实现

```python
class FeedDeduplicationService:
    """Feed 流去重：内容指纹 + 语义去重 + 传播链去重"""

    def deduplicate_feed(self, user_id, candidate_posts):
        """对候选 Feed 内容去重"""
        # 1. 精确去重（内容指纹）
        seen_fingerprints = set()
        deduped = []
        for post in candidate_posts:
            fingerprint = self._content_fingerprint(post)
            if fingerprint in seen_fingerprints:
                continue
            seen_fingerprints.add(fingerprint)
            deduped.append(post)

        # 2. 语义去重（相似内容）
        final = []
        for post in deduped:
            is_similar = False
            for existing in final:
                similarity = self._content_similarity(post, existing)
                if similarity > 0.85:
                    # 语义相似 → 保留热度更高的
                    if post.get("score", 0) > existing.get("score", 0):
                        final.remove(existing)
                        final.append(post)
                    is_similar = True
                    break
            if not is_similar:
                final.append(post)

        # 3. 传播链去重（同一来源的转发链只保留一条）
        source_groups = {}
        for post in final:
            origin_id = post.get("origin_post_id") or post["id"]
            source_groups.setdefault(origin_id, []).append(post)

        result = []
        for origin_id, posts in source_groups.items():
            if len(posts) > 1:
                # 保留互动数最高的
                best = max(posts, key=lambda p: p.get("score", 0))
                result.append(best)
            else:
                result.append(posts[0])

        # 保持原始排序
        result.sort(key=lambda p: candidate_posts.index(p) if p in candidate_posts else len(candidate_posts))

        return {"original_count": len(candidate_posts),
                "deduplicated_count": len(result),
                "removed": len(candidate_posts) - len(result),
                "posts": result}

    def _content_fingerprint(self, post):
        """计算内容指纹"""
        text = post.get("text", "")
        # 归一化：去空格、标点、表情
        normalized = re.sub(r'\s+', '', text)
        normalized = re.sub(r'[^\w一-鿿]', '', normalized)
        return hashlib.md5(normalized.encode()).hexdigest()

    def _content_similarity(self, post_a, post_b):
        """计算内容相似度（SimHash 简化版）"""
        text_a = post_a.get("text", "")
        text_b = post_b.get("text", "")

        # 字符级 Jaccard 相似度
        set_a = set(text_a[i:i+3] for i in range(len(text_a)-2))
        set_b = set(text_b[i:i+3] for i in range(len(text_b)-2))

        intersection = len(set_a & set_b)
        union = len(set_a | set_b)

        return intersection / max(union, 1)
```

## 异常场景补充

### 场景：去重过度导致信息茧房

```
触发：语义去重阈值 0.85 → 观点不同的相似内容被去重 → 用户只看到一种观点
检测：
  1. Feed 内容多样性下降 → 信息茧房
  2. 去重率 > 30% → 可能过度
处理：
  1. 语义去重仅去除完全重复，保留不同观点
  2. 去重时保留不同来源的相似内容
  3. 降低语义去重阈值（0.85 → 0.92）
预防：保留观点多样性 + 来源多样性 + 阈值调整
```

### 场景：内容指纹碰撞

```
触发：不同内容归一化后 MD5 相同 → 正常内容被误去重 → 内容丢失
检测：
  1. 去重后用户看不到应该看到的内容 → 误去重
  2. MD5 碰撞导致不同内容被归为同一指纹
处理：
  1. 指纹增加作者 ID（同内容不同作者不去重）
  2. 使用更长哈希（SHA-256）
  3. 精确去重 + 人工确认
预防：指纹包含作者信息 + SHA-256 + 去重日志
```

## 社交 Feed 推荐排序完整实现

```python
class FeedRankingService:
    """Feed 排序：多信号融合 + 个性化权重 + 新鲜度衰减"""

    SIGNAL_WEIGHTS = {
        "social": {"like": 1.0, "comment": 2.0, "share": 3.0, "follow": 0.5},
        "content": {"quality_score": 0.3, "topic_match": 0.2, "media_richness": 0.1},
        "freshness": {"decay_rate": 0.02, "half_life_hours": 12},
        "author": {"authority": 0.2, "relationship_strength": 0.4},
    }

    def rank_feed(self, user_id, candidate_posts, max_results=50):
        """排序 Feed"""
        # 1. 计算每条内容的综合分数
        scored_posts = []
        for post in candidate_posts:
            score = self._calculate_post_score(user_id, post)
            scored_posts.append({"post": post, "score": score})

        # 2. 按分数排序
        scored_posts.sort(key=lambda x: x["score"], reverse=True)

        # 3.多样性注入（避免同一作者连续出现）
        diversified = self._inject_diversity(scored_posts[:max_results * 2],
            max_results)

        # 4. 插入推荐内容（非社交内容）
        final_feed = self._mix_with_recommendations(user_id, diversified)

        return final_feed

    def _calculate_post_score(self, user_id, post):
        """计算内容综合分数"""
        # 社交信号
        social_score = 0
        for signal, weight in self.SIGNAL_WEIGHTS["social"].items():
            count = post.get(f"{signal}_count", 0)
            social_score += count * weight

        # 社交信号归一化（防止热门帖子分数过高）
        social_score = math.log1p(social_score)

        # 内容质量信号
        content_score = (
            post.get("quality_score", 0) * self.SIGNAL_WEIGHTS["content"]["quality_score"] +
            self._topic_match(user_id, post) * self.SIGNAL_WEIGHTS["content"]["topic_match"] +
            self._media_richness(post) * self.SIGNAL_WEIGHTS["content"]["media_richness"]
        )

        # 新鲜度衰减
        hours_since_post = (now() - post["created_at"]).total_seconds() / 3600
        decay_rate = self.SIGNAL_WEIGHTS["freshness"]["decay_rate"]
        freshness_score = math.exp(-decay_rate * hours_since_post)

        # 作者信号
        author_score = (
            post.get("author_authority", 0) * self.SIGNAL_WEIGHTS["author"]["authority"] +
            self._relationship_strength(user_id, post["author_id"]) * self.SIGNAL_WEIGHTS["author"]["relationship_strength"]
        )

        # 综合分数
        total = social_score * 0.4 + content_score * 0.3 + freshness_score * 0.2 + author_score * 0.1

        return round(total, 4)

    def _inject_diversity(self, scored_posts, max_results):
        """多样性注入"""
        result = []
        seen_authors = {}
        author_window = 3  # 同一作者最多连续出现 3 条后需要间隔

        for item in scored_posts:
            author_id = item["post"]["author_id"]
            consecutive = seen_authors.get(author_id, 0)

            if consecutive >= author_window:
                continue  # 跳过，等间隔后再加入

            result.append(item)
            seen_authors[author_id] = consecutive + 1

            # 重置其他作者的连续计数
            for other_author in seen_authors:
                if other_author != author_id:
                seen_authors[other_author] = 0

            if len(result) >= max_results:
                break

        return result

    def _topic_match(self, user_id, post):
        """主题匹配度"""
        user_interests = self.redis.hgetall(f"user_interests:{user_id}")
        post_topics = post.get("topics", [])

        match_count = sum(1 for t in post_topics
            if t in user_interests)
        return match_count / max(len(post_topics), 1)

    def _media_richness(self, post):
        """媒体丰富度"""
        richness = 0
        if post.get("image_urls"):
            richness += 0.5
        if post.get("video_url"):
            richness += 1.0
        if post.get("link_url"):
            richness += 0.2
        return min(1.0, richness)

    def _relationship_strength(self, user_id, author_id):
        """关系强度"""
        # 互关 = 1.0，单向关注 = 0.5，无关注 = 0.0
        follows = self.db.count("follows",
            follower_id=user_id, following_id=author_id)
        followed_by = self.db.count("follows",
            follower_id=author_id, following_id=user_id)

        if follows and followed_by:
            return 1.0
        elif follows:
            return 0.5
        return 0.0
```

## 异常场景补充

### 场景：多样性注入导致高质量内容被过滤

```
触发：某作者发布 4 条高质量内容 → 第 4 条被多样性过滤 → 用户错过好内容
检测：
  1. 被过滤内容的质量分数 > 最终 Feed 中最低分数 → 过滤不当
  2. 用户错过关注者的内容 → Feed 体验差
处理：
  1. 放宽多样性窗口（3 → 5）
  2. 高分数内容不受多样性限制
  3. 只对非关注作者的内容进行多样性控制
预防：放宽窗口 + 高分数例外 + 关注者例外
```

### 场景：排序信号权重漂移

```
触发：社交信号权重从 0.4 变为 0.6（无审批）→ 热门帖子垄断 → 小创作者内容不可见
检测：
  1. Feed 中热门帖子占比 > 80% → 权重偏移
  2. 小创作者曝光率下降 → 生态恶化
处理：
  1. 回退权重配置
  2. 权重变更需审批
  3. 加入曝光公平性监控
预防：权重审批流程 + 曝光公平性监控 + A/B 测试
```

## 社交关系链分析与好友推荐完整实现

```python
class SocialGraphAnalysisService:
    """社交图分析：关系链 + 共同好友 + 二度人脉推荐"""

    def recommend_people(self, user_id, max_results=20):
        """推荐可能认识的人（二度人脉 + 共同好友评分）"""
        # 1. 获取一度人脉（直接关注）
        direct_friends = set(self.db.query_values(
            "SELECT following_id FROM follows "
            "WHERE follower_id = %s AND status = 'active'", user_id))
        direct_friends.add(user_id)  # 排除自己

        # 2. 获取二度人脉（朋友的朋友）
        second_degree = {}
        for friend_id in direct_friends - {user_id}:
            friends_of_friend = self.db.query_values(
                "SELECT following_id FROM follows "
                "WHERE follower_id = %s AND status = 'active'", friend_id)

            for fof in friends_of_friend:
                if fof in direct_friends:
                    continue  # 已是一度人脉 → 跳过
                if fof not in second_degree:
                    second_degree[fof] = {"mutual_friends": [], "score": 0}
                second_degree[fof]["mutual_friends"].append(friend_id)

        # 3. 计算推荐分数
        candidates = []
        for candidate_id, info in second_degree.items():
            mutual_count = len(info["mutual_friends"])

            # 基础分：共同好友数量
            score = mutual_count * 10

            # 加成：共同好友中亲密度高的权重更大
            for mf_id in info["mutual_friends"]:
                strength = self._get_relationship_strength(user_id, mf_id)
                score += strength * 5

            # 加成：兴趣相似度
            interest_similarity = self._calculate_interest_similarity(user_id, candidate_id)
            score += interest_similarity * 20

            # 减分：对方关注人数过多（网红号 → 不推荐）
            candidate_following_count = self.db.count("follows",
                follower_id=candidate_id, status="active")
            if candidate_following_count > 5000:
                score *= 0.3

            candidates.append({
                "candidate_id": candidate_id,
                "mutual_friend_count": mutual_count,
                "mutual_friends": info["mutual_friends"][:5],
                "score": round(score, 1)
            })

        # 4. 排序并返回
        candidates.sort(key=lambda c: c["score"], reverse=True)

        return {
            "user_id": user_id,
            "total_candidates": len(candidates),
            "recommendations": candidates[:max_results]
        }

    def analyze_social_circles(self, user_id):
        """分析社交圈（社区发现）"""
        # 1. 获取社交图
        friends = self.db.query_values(
            "SELECT following_id FROM follows "
            "WHERE follower_id = %s AND status = 'active'", user_id)

        # 2. 构建局部社交图
        graph = {user_id: set(friends)}
        for friend_id in friends:
            friends_of_friend = self.db.query_values(
                "SELECT following_id FROM follows "
                "WHERE follower_id = %s AND status = 'active'", friend_id)
            graph[friend_id] = set(friends_of_friend)

        # 3. 连通分量分析（发现社交圈）
        visited = set()
        circles = []

        for node in graph:
            if node in visited:
                continue
            # BFS 找连通分量
            circle = []
            queue = [node]
            while queue:
                current = queue.pop(0)
                if current in visited:
                    continue
                visited.add(current)
                circle.append(current)
                for neighbor in graph.get(current, set()):
                    if neighbor in graph and neighbor not in visited:
                        queue.append(neighbor)

            if len(circle) > 2:
                circles.append(circle)

        # 4. 按圈子特征分类
        classified_circles = []
        for circle in circles:
            circle_info = self._classify_circle(circle, user_id)
            classified_circles.append(circle_info)

        classified_circles.sort(key=lambda c: c["member_count"], reverse=True)

        return {
            "user_id": user_id,
            "total_circles": len(classified_circles),
            "circles": classified_circles[:5]
        }

    def _get_relationship_strength(self, user_a, user_b):
        """获取关系强度（0-1）"""
        # 互动频率
        interactions = self.db.count("interactions",
            user_id=user_a, target_user_id=user_b,
            created_at__gte=now()-timedelta(days=30))

        # 互动类型加权
        likes = self.db.count("interactions",
            user_id=user_a, target_user_id=user_b, type="like",
            created_at__gte=now()-timedelta(days=30))
        comments = self.db.count("interactions",
            user_id=user_a, target_user_id=user_b, type="comment",
            created_at__gte=now()-timedelta(days=30))

        strength = min(1.0, (likes * 0.1 + comments * 0.3) / 10)
        return strength

    def _calculate_interest_similarity(self, user_a, user_b):
        """计算兴趣相似度"""
        interests_a = self.redis.hgetall(f"user_interests:{user_a}")
        interests_b = self.redis.hgetall(f"user_interests:{user_b}")

        if not interests_a or not interests_b:
            return 0

        set_a = set(interests_a.keys())
        set_b = set(interests_b.keys())

        intersection = len(set_a & set_b)
        union = len(set_a | set_b)

        return intersection / max(union, 1)

    def _classify_circle(self, circle, user_id):
        """分类社交圈"""
        # 分析圈子成员特征
        members_info = self.db.query(
            "SELECT id, company, school, location FROM users "
            "WHERE id IN %s", tuple(circle))

        # 按共同属性分类
        companies = {}
        schools = {}
        locations = {}

        for m in members_info:
            if m.get("company"):
                companies[m["company"]] = companies.get(m["company"], 0) + 1
            if m.get("school"):
                schools[m["school"]] = schools.get(m["school"], 0) + 1
            if m.get("location"):
                locations[m["location"]] = locations.get(m["location"], 0) + 1

        # 找主要特征
        circle_type = "social"
        dominant_feature = None
        if companies:
            top_company = max(companies, key=companies.get)
            if companies[top_company] / len(circle) > 0.5:
                circle_type = "work"
                dominant_feature = top_company
        if schools:
            top_school = max(schools, key=schools.get)
            if schools[top_school] / len(circle) > 0.5:
                circle_type = "school"
                dominant_feature = top_school

        return {
            "member_count": len(circle),
            "type": circle_type,
            "dominant_feature": dominant_feature,
            "members": circle[:10]
        }
```

## 异常场景补充

### 场景：好友推荐暴露用户隐私

```
触发：A 和 B 不是好友 → 但系统推荐"B 可能认识 A"→ A 知道 B 的存在 → 隐私泄露
检测：
  1. 用户投诉不想被推荐 → 隐私侵犯
  2. 推荐来源暴露了用户不想公开的信息
处理：
  1. 提供"不允许被推荐"隐私设置
  2. 不显示共同好友详情（只显示数量）
  3. 隐私模式下不参与推荐
预防：隐私设置 + 隐藏详情 + 隐私模式
```

### 场景：社交图计算性能差

```
触发：用户有 5000 关注 → 二度人脉 50 万 → 推荐计算耗时 > 10 秒 → 超时
检测：
  1. 推荐接口响应时间 > 3 秒 → 性能问题
  2. 大关注量用户推荐超时 → 计算瓶颈
处理：
  1. 预计算推荐结果（离线批处理）
  2. 限制二度人脉搜索深度
  3. 采样计算（不遍历全部二度人脉）
预防：离线预计算 + 深度限制 + 采样
```

## 社交关系链分析与推荐完整实现

```python
import math
import uuid
import logging
from enum import Enum
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Set, Tuple
from datetime import datetime, timedelta
from collections import defaultdict

logger = logging.getLogger(__name__)


class InteractionType(Enum):
    LIKE = "like"
    COMMENT = "comment"
    SHARE = "share"
    MENTION = "mention"
    DM = "dm"  # direct message


@dataclass
class FollowRelation:
    follower_id: str
    followee_id: str
    created_at: datetime
    is_mutual: bool = False


@dataclass
class Interaction:
    user_id: str
    target_user_id: str
    interaction_type: InteractionType
    timestamp: datetime
    weight: float = 1.0  # 不同交互类型的权重


@dataclass
class ConnectionStrength:
    user_a: str
    user_b: str
    strength: float  # 0.0 ~ 1.0
    interaction_count: int
    recency_score: float  # 0.0 ~ 1.0
    interaction_diversity: int  # 不同交互类型数量
    last_interaction_at: Optional[datetime] = None
    breakdown: Dict[str, float] = field(default_factory=dict)


@dataclass
class Recommendation:
    target_user_id: str
    source_user_id: str  # 被推荐给谁
    score: float
    mutual_friend_count: int
    mutual_friends: List[str]
    interaction_similarity: float
    reasons: List[str] = field(default_factory=list)


class SocialGraphService:
    """社交关系链分析与推荐服务，基于关注数据构建社交图，
    计算连接强度，发现共同好友，通过二度人脉推荐可能认识的人。"""

    # 交互类型权重
    INTERACTION_WEIGHTS = {
        InteractionType.LIKE: 1.0,
        InteractionType.COMMENT: 3.0,
        InteractionType.SHARE: 4.0,
        InteractionType.MENTION: 5.0,
        InteractionType.DM: 8.0,
    }

    # 时间衰减参数
    RECENCY_HALF_LIFE_DAYS = 30  # 30天半衰期

    # 推荐参数
    MAX_MUTUAL_FRIENDS_DISPLAY = 5
    MAX_RECOMMENDATIONS = 20
    MIN_RECOMMENDATION_SCORE = 0.1
    MAX_DEGREE = 2  # 二度人脉

    def __init__(self):
        # 邻接表：user_id -> set of followee_ids
        self.follow_graph: Dict[str, Set[str]] = defaultdict(set)
        # 反向邻接表（粉丝）：user_id -> set of follower_ids
        self.follower_graph: Dict[str, Set[str]] = defaultdict(set)
        # 交互记录：(user_id, target_user_id) -> list of Interaction
        self.interactions: Dict[Tuple[str, str], List[Interaction]] = defaultdict(list)
        # 缓存的连接强度
        self.strength_cache: Dict[Tuple[str, str], ConnectionStrength] = {}
        # 缓存的共同好友
        self.mutual_friends_cache: Dict[Tuple[str, str], List[str]] = {}
        # 用户隐私设置：user_id -> set of restricted fields
        self.privacy_settings: Dict[str, Set[str]] = defaultdict(set)
        # 用户黑名单：user_id -> set of blocked_user_ids
        self.block_list: Dict[str, Set[str]] = defaultdict(set)
        # 推荐排除列表：user_id -> set of dismissed_user_ids
        self.dismissed: Dict[str, Set[str]] = defaultdict(set)

    def build_social_graph(
        self, follow_relations: List[FollowRelation]
    ) -> Dict[str, int]:
        """从关注关系数据构建社交图。

        Returns:
            统计信息：{total_users, total_edges, mutual_edges}
        """
        mutual_count = 0

        for rel in follow_relations:
            self.follow_graph[rel.follower_id].add(rel.followee_id)
            self.follower_graph[rel.followee_id].add(rel.follower_id)

            # 检查是否互关
            if rel.follower_id in self.follow_graph.get(rel.followee_id, set()):
                rel.is_mutual = True
                mutual_count += 1

        # 收集所有用户
        all_users = set(self.follow_graph.keys()) | set(self.follower_graph.keys())

        # 清除缓存（图结构变化）
        self.strength_cache.clear()
        self.mutual_friends_cache.clear()

        stats = {
            "total_users": len(all_users),
            "total_edges": len(follow_relations),
            "mutual_edges": mutual_count,
        }

        logger.info(
            f"Social graph built: {stats['total_users']} users, "
            f"{stats['total_edges']} edges, {stats['mutual_edges']} mutual"
        )
        return stats

    def add_interactions(self, interaction_list: List[Interaction]):
        """添加交互记录。"""
        for interaction in interaction_list:
            key = (interaction.user_id, interaction.target_user_id)
            interaction.weight = self.INTERACTION_WEIGHTS.get(
                interaction.interaction_type, 1.0
            )
            self.interactions[key].append(interaction)

        # 清除相关缓存
        affected_pairs = set()
        for interaction in interaction_list:
            pair = tuple(sorted([interaction.user_id, interaction.target_user_id]))
            affected_pairs.add(pair)
        for pair in affected_pairs:
            self.strength_cache.pop(pair, None)

    def calculate_connection_strength(
        self, user_a: str, user_b: str, force_recalculate: bool = False
    ) -> ConnectionStrength:
        """计算两个用户之间的连接强度。

        连接强度 = 交互频率得分 * 时间衰减得分 * 交互多样性得分 * 互关加分

        Args:
            user_a: 用户A的ID
            user_b: 用户B的ID
            force_recalculate: 是否强制重新计算（忽略缓存）

        Returns:
            ConnectionStrength 对象
        """
        pair_key = tuple(sorted([user_a, user_b]))

        if not force_recalculate and pair_key in self.strength_cache:
            return self.strength_cache[pair_key]

        # 收集双向交互
        forward_key = (user_a, user_b)
        backward_key = (user_b, user_a)
        forward_interactions = self.interactions.get(forward_key, [])
        backward_interactions = self.interactions.get(backward_key, [])

        all_interactions = forward_interactions + backward_interactions

        # 1. 交互频率得分
        total_weighted_count = sum(i.weight for i in all_interactions)
        frequency_score = min(1.0, total_weighted_count / 50.0)  # 50次满权重交互 = 满分

        # 2. 时间衰减得分（基于最近一次交互）
        recency_score = 0.0
        last_interaction = None
        if all_interactions:
            latest = max(all_interactions, key=lambda i: i.timestamp)
            last_interaction = latest.timestamp
            days_since = (datetime.now() - last_interaction).days
            recency_score = math.exp(-0.693 * days_since / self.RECENCY_HALF_LIFE_DAYS)

        # 3. 交互多样性得分
        interaction_types = set(i.interaction_type for i in all_interactions)
        diversity_score = min(1.0, len(interaction_types) / len(InteractionType))

        # 4. 互关加分
        mutual_bonus = 0.0
        is_mutual = (
            user_b in self.follow_graph.get(user_a, set())
            and user_a in self.follow_graph.get(user_b, set())
        )
        if is_mutual:
            mutual_bonus = 0.15  # 互关额外加 15%

        # 5. 双向性得分（双向交互比单向更强）
        forward_weight = sum(i.weight for i in forward_interactions)
        backward_weight = sum(i.weight for i in backward_interactions)
        total_weight = forward_weight + backward_weight
        if total_weight > 0:
            balance = min(forward_weight, backward_weight) / total_weight
            bidirectional_score = balance * 2  # 0~1，完全平衡=1
        else:
            bidirectional_score = 0.0

        # 综合得分
        strength = (
            frequency_score * 0.35
            + recency_score * 0.30
            + diversity_score * 0.15
            + bidirectional_score * 0.10
            + mutual_bonus * 0.10
        )
        strength = min(1.0, strength)

        # 按交互类型拆分
        breakdown = defaultdict(float)
        for i in all_interactions:
            breakdown[i.interaction_type.value] += i.weight
        # 归一化
        total_breakdown = sum(breakdown.values())
        if total_breakdown > 0:
            breakdown = {k: round(v / total_breakdown, 3) for k, v in breakdown.items()}

        result = ConnectionStrength(
            user_a=pair_key[0],
            user_b=pair_key[1],
            strength=round(strength, 4),
            interaction_count=len(all_interactions),
            recency_score=round(recency_score, 4),
            interaction_diversity=len(interaction_types),
            last_interaction_at=last_interaction,
            breakdown=dict(breakdown),
        )

        self.strength_cache[pair_key] = result
        return result

    def find_mutual_friends(
        self, user_a: str, user_b: str
    ) -> List[str]:
        """查找两个用户的共同好友（互相关注的用户交集）。"""
        pair_key = tuple(sorted([user_a, user_b]))
        if pair_key in self.mutual_friends_cache:
            return self.mutual_friends_cache[pair_key]

        # user_a 关注的人
        a_following = self.follow_graph.get(user_a, set())
        # user_b 关注的人
        b_following = self.follow_graph.get(user_b, set())

        # 共同关注（二人都关注的人）
        mutual = a_following & b_following

        # 排除彼此
        mutual.discard(user_a)
        mutual.discard(user_b)

        # 按与两人的连接强度排序
        scored_mutual = []
        for friend_id in mutual:
            strength_a = self.calculate_connection_strength(user_a, friend_id).strength
            strength_b = self.calculate_connection_strength(user_b, friend_id).strength
            combined = strength_a + strength_b
            scored_mutual.append((friend_id, combined))

        scored_mutual.sort(key=lambda x: x[1], reverse=True)
        result = [uid for uid, _ in scored_mutual]

        self.mutual_friends_cache[pair_key] = result
        return result

    def recommend_people(
        self, user_id: str, limit: int = 20
    ) -> List[Recommendation]:
        """基于二度人脉推荐"你可能认识的人"。

        推荐逻辑：
        1. 找到用户的所有一度人脉（直接关注的人）
        2. 找到一度人脉的关注列表中不在用户关注列表里的人（二度人脉）
        3. 对每个二度人脉计算推荐分数：
           - 共同好友数量（归一化）
           - 交互相似度（与共同好友的交互模式相似性）
           - 连接强度加权
        4. 过滤隐私限制和黑名单
        5. 返回排序后的推荐列表
        """
        # 获取用户的一度人脉
        following = self.follow_graph.get(user_id, set())

        if not following:
            logger.info(f"User {user_id} has no following, cannot recommend")
            return []

        # 统计二度人脉出现次数和来源
        candidate_sources: Dict[str, List[str]] = defaultdict(list)  # candidate -> [mutual friends]
        for friend_id in following:
            friend_following = self.follow_graph.get(friend_id, set())
            for candidate_id in friend_following:
                # 排除：自己、已关注、已拉黑、已忽略
                if candidate_id == user_id:
                    continue
                if candidate_id in following:
                    continue
                if candidate_id in self.block_list.get(user_id, set()):
                    continue
                if candidate_id in self.dismissed.get(user_id, set()):
                    continue
                # 检查隐私设置
                if "hide_from_recommendations" in self.privacy_settings.get(candidate_id, set()):
                    continue

                candidate_sources[candidate_id].append(friend_id)

        if not candidate_sources:
            return []

        # 为每个候选计算推荐分数
        recommendations = []
        for candidate_id, mutual_friends in candidate_sources.items():
            mutual_count = len(mutual_friends)

            # 1. 共同好友得分（归一化，10个以上共同好友即满分）
            mutual_score = min(1.0, mutual_count / 10.0)

            # 2. 交互相似度得分
            similarity_score = self._calculate_interaction_similarity(
                user_id, candidate_id, mutual_friends
            )

            # 3. 共同好友连接强度加权得分
            strength_score = self._calculate_weighted_strength_score(
                user_id, candidate_id, mutual_friends
            )

            # 综合推荐分数
            final_score = (
                mutual_score * 0.40
                + similarity_score * 0.30
                + strength_score * 0.30
            )

            # 低于最低分数阈值的跳过
            if final_score < self.MIN_RECOMMENDATION_SCORE:
                continue

            # 生成推荐理由
            reasons = self._generate_recommendation_reasons(
                user_id, candidate_id, mutual_friends, mutual_score, similarity_score
            )

            rec = Recommendation(
                target_user_id=candidate_id,
                source_user_id=user_id,
                score=round(final_score, 4),
                mutual_friend_count=mutual_count,
                mutual_friends=mutual_friends[:self.MAX_MUTUAL_FRIENDS_DISPLAY],
                interaction_similarity=round(similarity_score, 4),
                reasons=reasons,
            )
            recommendations.append(rec)

        # 按分数降序排序
        recommendations.sort(key=lambda r: r.score, reverse=True)

        # 返回 top N
        result = recommendations[:min(limit, self.MAX_RECOMMENDATIONS)]

        logger.info(
            f"Generated {len(result)} recommendations for user {user_id} "
            f"(from {len(candidate_sources)} candidates)"
        )
        return result

    def _calculate_interaction_similarity(
        self, user_a: str, user_b: str, mutual_friends: List[str]
    ) -> float:
        """计算两个用户与共同好友的交互模式相似度。

        基于与每个共同好友的交互类型分布计算余弦相似度。
        """
        if not mutual_friends:
            return 0.0

        # 构建交互向量
        vector_a = []
        vector_b = []
        for friend_id in mutual_friends:
            for itype in InteractionType:
                # user_a 与 friend 的该类型交互数
                key_a = (user_a, friend_id)
                count_a = sum(
                    1 for i in self.interactions.get(key_a, [])
                    if i.interaction_type == itype
                )
                vector_a.append(count_a)

                # user_b 与 friend 的该类型交互数
                key_b = (user_b, friend_id)
                count_b = sum(
                    1 for i in self.interactions.get(key_b, [])
                    if i.interaction_type == itype
                )
                vector_b.append(count_b)

        # 余弦相似度
        dot_product = sum(a * b for a, b in zip(vector_a, vector_b))
        norm_a = math.sqrt(sum(a * a for a in vector_a))
        norm_b = math.sqrt(sum(b * b for b in vector_b))

        if norm_a == 0 or norm_b == 0:
            return 0.0

        similarity = dot_product / (norm_a * norm_b)
        return min(1.0, similarity)

    def _calculate_weighted_strength_score(
        self, user_id: str, candidate_id: str, mutual_friends: List[str]
    ) -> float:
        """计算基于共同好友连接强度的加权得分。"""
        if not mutual_friends:
            return 0.0

        total_strength = 0.0
        for friend_id in mutual_friends:
            strength_to_user = self.calculate_connection_strength(
                user_id, friend_id
            ).strength
            strength_to_candidate = self.calculate_connection_strength(
                candidate_id, friend_id
            ).strength
            # 取两个连接的调和平均
            if strength_to_user + strength_to_candidate > 0:
                harmonic = (2 * strength_to_user * strength_to_candidate) / (
                    strength_to_user + strength_to_candidate
                )
            else:
                harmonic = 0.0
            total_strength += harmonic

        # 归一化（3个强连接即满分）
        return min(1.0, total_strength / 3.0)

    def _generate_recommendation_reasons(
        self,
        user_id: str,
        candidate_id: str,
        mutual_friends: List[str],
        mutual_score: float,
        similarity_score: float,
    ) -> List[str]:
        """生成推荐理由文本。"""
        reasons = []

        # 共同好友理由
        if len(mutual_friends) >= 3:
            names = ", ".join(mutual_friends[:3])
            reasons.append(f"有 {len(mutual_friends)} 位共同好友（{names} 等）")
        elif len(mutual_friends) > 0:
            names = ", ".join(mutual_friends[:3])
            reasons.append(f"有 {len(mutual_friends)} 位共同好友（{names}）")

        # 交互相似性理由
        if similarity_score > 0.7:
            reasons.append("与你的互动模式非常相似")
        elif similarity_score > 0.4:
            reasons.append("与你的互动模式较为相似")

        # 检查是否有共同群组或话题（简化判断）
        if mutual_score > 0.8:
            reasons.append("你们在同一个社交圈中")

        return reasons

    def dismiss_recommendation(self, user_id: str, target_user_id: str):
        """用户忽略某个推荐，后续不再推荐。"""
        self.dismissed[user_id].add(target_user_id)
        logger.info(f"User {user_id} dismissed recommendation for {target_user_id}")

    def set_privacy(self, user_id: str, settings: Set[str]):
        """设置用户隐私选项。"""
        self.privacy_settings[user_id] = settings
        logger.info(f"Privacy settings updated for user {user_id}: {settings}")

    def get_graph_stats(self) -> Dict:
        """获取社交图的统计信息。"""
        all_users = set(self.follow_graph.keys()) | set(self.follower_graph.keys())
        total_edges = sum(len(following) for following in self.follow_graph.values())

        # 计算平均关注数和粉丝数
        avg_following = total_edges / max(1, len(all_users))
        avg_followers = total_edges / max(1, len(all_users))

        # 计算互关比例
        mutual_pairs = 0
        checked = set()
        for user, following in self.follow_graph.items():
            for followee in following:
                pair = tuple(sorted([user, followee]))
                if pair not in checked:
                    checked.add(pair)
                    if user in self.follow_graph.get(followee, set()):
                        mutual_pairs += 1

        return {
            "total_users": len(all_users),
            "total_follow_edges": total_edges,
            "avg_following_count": round(avg_following, 2),
            "avg_follower_count": round(avg_followers, 2),
            "mutual_follow_ratio": round(mutual_pairs / max(1, len(checked)), 4),
            "total_interactions": sum(
                len(interactions) for interactions in self.interactions.values()
            ),
            "cached_strengths": len(self.strength_cache),
        }
```

## 异常场景补充

### 场景：社交图构建性能差
```
触发条件：用户规模达到数千万级，关注关系数亿条，全量构建社交图耗时超过数小时，内存占用超过 200GB，无法满足实时推荐需求

检测机制：
  - 图构建任务耗时超过预期 SLA（如 30 分钟）
  - 内存使用率超过 90% 触发 OOM 预警
  - 推荐接口 P99 延迟超过 5 秒
  - 共同好友查询超时率 > 1%
  - 连接强度计算批量任务堆积

处理策略：
  - 将全量图构建改为增量更新：新关注关系实时写入，而非全量重建
  - 采用图数据库（Neo4j / NebulaGraph）替代内存邻接表，利用图数据库的原生图遍历优化
  - 对连接强度计算实施分层缓存：强连接实时计算，弱连接异步批量预计算
  - 共同好友查询使用 Bloom Filter 快速排除无交集用户对
  - 推荐计算从实时改为近实时：每小时预计算 top-N 推荐列表并缓存
  - 对超大度数节点（网红/大V）做特殊处理：其关注列表不参与二度人脉遍历

预防措施：
  - 从设计阶段选择合适的存储引擎，千万级以上用户必须使用图数据库
  - 关注关系采用分片存储（按用户 ID 哈希分片），避免单节点热点
  - 实现增量图更新机制，每日仅处理增量数据而非全量重建
  - 对推荐计算设置合理的超时和降级策略（超时返回缓存结果）
  - 定期清理无效/僵尸关注关系，减小图规模
  - 建立性能基线：每次图结构变更后进行基准测试
```

### 场景：推荐用户侵犯隐私
```
触发条件：推荐算法将用户暴露给不应被推荐的对象（如家暴受害者的新账号被推荐给施暴者、隐身用户被推荐给前同事、未成年人被推荐给可疑用户），导致用户隐私泄露和安全风险

检测机制：
  - 用户举报推荐列表中出现不应被推荐的人
  - 新注册用户（隐私设置未完善）被大量推荐
  - 推荐列表中出现用户已拉黑的人（绕过黑名单）
  - 推荐理由中暴露了用户不想公开的社交关系（如"你们都关注了某心理援助账号"）
  - 未成年人被推荐给年龄差距过大的用户

处理策略：
  - 立即下线受影响的推荐结果，清除缓存
  - 强化隐私过滤层：推荐结果返回前必须通过隐私检查管道
  - 黑名单检查从推荐后过滤改为推荐前排除（在候选生成阶段即排除）
  - 新注册用户默认不进入推荐池，完成隐私设置后才可被推荐
  - 推荐理由不暴露敏感共同关注（如医疗、法律援助类账号）
  - 对未成年人实施额外保护：仅推荐同年龄段用户，且需双向同意

预防措施：
  - 建立多层隐私保护管道：候选生成 -> 隐私过滤 -> 人工抽检 -> 返回
  - 敏感共同关注类别黑名单（医疗、法律、心理健康等）不出现在推荐理由中
  - 新用户冷启动期（前 7 天）默认"仅好友可见"，不参与任何推荐
  - 未成年用户使用独立的推荐算法和候选池
  - 定期由隐私合规团队审计推荐算法的隐私泄露风险
  - 提供清晰的"不想被推荐"开关，且默认对敏感用户群体开启
  - 推荐结果添加"为何看到此推荐"说明，接受用户反馈和举报
  - 实施"信息最小化"原则：推荐计算仅使用必要的最小数据集
```
