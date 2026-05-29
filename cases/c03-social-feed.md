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

需要将该用户的发帖策略从"推模型"切换为"拉模型"：

```python
class ThresholdMigration:
    def on_follower_count_changed(self, user_id, new_count):
        if new_count >= 10000 and self.get_strategy(user_id) == "push":
            # 推 → 拉 迁移
            self.migrate_push_to_pull(user_id)
        elif new_count < 10000 and self.get_strategy(user_id) == "pull":
            # 拉 → 推 迁移（极少发生，用户掉粉到 1 万以下）
            self.migrate_pull_to_push(user_id)

    def migrate_push_to_pull(self, user_id):
        """
        从推模型切换到拉模型
        关键：切换期间不能丢失帖子
        """
        # 1. 更新策略标记
        self.redis.set(f"feed:strategy:{user_id}", "pull")

        # 2. 将该用户加入所有粉丝的大V列表
        followers = self.get_all_followers(user_id)
        pipe = self.redis.pipeline()
        for follower_id in followers:
            pipe.sadd(f"feed:bigv:{follower_id}", user_id)
        pipe.execute()

        # 3. 此后新发帖子只写发件箱，不再推送
        # 注意：已经在收件箱中的帖子不需要删除（自然过期）

        # 4. 确保发件箱有足够的历史数据
        # 从 MySQL 加载最近 500 条帖子到发件箱
        recent_posts = self.db.query(
            "SELECT id, created_at FROM posts WHERE author_id = %s "
            "ORDER BY created_at DESC LIMIT 500", user_id
        )
        pipe = self.redis.pipeline()
        for post in recent_posts:
            pipe.zadd(f"feed:outbox:{user_id}", {post.id: post.created_at_ms})
        pipe.execute()
```

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

## 延伸思考

- **关注话题（超话）**：超话是一个独立的"虚拟用户"，关注超话 = 关注这个虚拟用户。帖子带有超话标签时，推送到超话的"发件箱"，用户刷新时额外合并关注超话的发件箱。
- **"看过"标记**：每个用户维护一个 Bloom Filter（`feed:seen:{userId}`），标记已读帖子 ID。刷新时过滤掉已读帖子。Bloom Filter 约 0.1% 误判率（可能漏掉未读帖子），但用户体验影响极小。
- **视频 Feed（抖音模式）**：关键差异——不是"刷列表"而是"逐条滑动"，每次只加载 1 条。预加载下 3 条即可，不需要维护收件箱。架构简化为：推荐服务直接返回下一条帖子 ID。