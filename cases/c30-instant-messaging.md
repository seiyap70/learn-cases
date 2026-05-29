# C30: 大规模即时通讯系统

## 业务场景

某企业级即时通讯平台（类似企业微信/钉钉），1 亿用户，日均 50 亿条消息。IM 系统是企业的核心生产力工具——消息丢失意味着工作指令丢失，消息延迟意味着协作效率下降。

**已知数据：**
- 注册用户：1 亿，日活 3000 万
- 日均消息量：50 亿条，峰值 QPS 100 万
- 单聊延迟 < 200ms，群聊延迟 < 500ms
- 最大群规模：10 万人
- 消息永久存储（合规要求）

**为什么比普通 IM 难？**

1. **消息不可丢失** —— 工作指令丢失可能导致严重后果
2. **10 万人大群** —— 一个消息扇出给 10 万人，写扩散不可行
3. **永久存储** —— 50 亿条/天 × 30 天 = 1500 亿条，存储成本巨大
4. **已读回执** —— 发送方需要知道"谁看了这条消息"

## 核心挑战

### 挑战 1：消息投递的"至少一次"保证

消息经过接入网关→消息服务→存储服务→推送服务，任何环节都可能失败。如何保证消息不丢？

### 挑战 2：10 万人大群的扇出

10 万人同时在线，消息必须在 500ms 内送达。写扩散（为每个人写收件箱）= 10 万次写入/条消息 → 不可行。

### 挑战 3：已读回执的存储效率

10 万人群 × 每条消息 × 每人一个已读状态 = 海量数据。

### 挑战 4：消息有序性

同一会话内的消息必须有序（序号连续），且 ID 全局唯一（用于去重和检索）。Snowflake ID 不保证同会话有序（时钟偏差）。

## 设计约束

- WebSocket 长连接协议
- 消息存储：分库分表 MySQL
- 文件存储：对象存储
- 推送通道：APNs + FCM + 厂商推送

## 请先独立思考（限时 45 分钟）

1. 消息 ID 方案：Snowflake vs UUID vs 自增序号？如何保证同一会话内有序？
2. 大群消息的扇出方案：写扩散 vs 读扩散 vs 混合？计算各方案的存储和延迟。
3. 已读回执如何不存"每人每条"的记录，而是用一个更紧凑的方案？
4. 消息"不丢不重"：投递至少一次 + 客户端幂等去重，具体如何实现？

---

## 设计解析

### 消息 ID 设计：Snowflake + 会话自增 seq

**为什么不用纯 Snowflake？**

Snowflake ID 是全局唯一但不保证同一会话有序。两个用户在同一群里同时发消息，可能得到 ID A=1001, B=1002（按时间先后），也可能 A=1002, B=1001（如果 B 的服务器时钟略快）。

**方案：Snowflake 作为全局唯一 ID + 会话内自增 seq 作为排序键**

```sql
-- 每个会话维护自增序号（原子操作）
UPDATE conversation_seq 
SET current_seq = current_seq + 1 
WHERE conversation_id = 'conv_12345'
RETURNING current_seq;
```

消息表包含两个 ID：
- `msg_id`：Snowflake（全局唯一，用于去重和检索）
- `seq`：会话内自增（用于排序和同步）

**客户端同步使用 seq 而非 msg_id：**

```
客户端请求："给我 conv_12345 中 seq > 100 的消息"
服务端返回：seq 101-120 的消息，按 seq 排序
客户端本地存储最大 seq = 120，下次请求 seq > 120
```

**seq 的好处：**
- 同会话严格有序（单点自增，无时钟偏差问题）
- 同步高效（客户端只报 max_seq，服务端返回差量）
- 已读回执可以基于 seq（见下文）

### 消息投递：先存后推

**核心原则：消息必须先持久化到存储，再推送给接收方。**

```python
class MessageDeliveryService:
    def send_message(self, msg):
        # 1. 生成消息 ID 和 seq
        msg.msg_id = snowflake_id()
        msg.seq = self.increment_seq(msg.conversation_id)

        # 2. 持久化消息（先写存储——WAL 原则）
        self.storage.save(msg)

        # 3. 投递给接收方
        recipients = self.get_recipients(msg.conversation_id)
        for recipient in recipients:
            if self.is_online(recipient):
                self.push_online(recipient, msg)
            else:
                self.push_offline(recipient, msg)

        # 4. 确认发送方
        self.ack_sender(msg.sender_id, msg.msg_id)
```

**为什么先存后推？**

| 顺序 | 存储 | 推送 | 后果 |
|------|------|------|------|
| 先推后存 | 失败 | 成功 | 消息"消失"（接收方看到了但刷新后找不到） |
| 先存后推 | 成功 | 失败 | 接收方未收到，但可主动拉取 → 不丢消息 |

先存后推：存储成功但推送失败 → 接收方未收到 → 上线时通过 seq 差量拉取 → 不丢消息。

### 大群消息：读扩散为主

**为什么大群不能用写扩散？**

10 万人大群，写扩散 = 为每个人写一条收件箱记录：
- 一条消息 × 10 万次写入 = 10 万次 Redis/DB 操作
- 活跃大群每秒 10 条消息 × 10 万 = 100 万次/秒写入 → 不可行

**方案：大群（> 500 人）使用读扩散**

```python
class GroupMessageDelivery:
    def deliver(self, msg, group):
        if group.member_count <= 500:
            # 小群：写扩散（为每个成员写收件箱）
            self.write_to_all_inboxes(msg, group.members)
        else:
            # 大群：只写群消息表 + 推送在线成员
            self.save_group_message(msg, group.id)
            online_members = self.get_online_members(group.id)
            self.push_online_batch(online_members, msg)
```

**离线成员上线后的同步：**

```python
class OfflineSync:
    def on_user_online(self, user_id):
        groups = self.get_user_groups(user_id)

        for group in groups:
            # 用户上次阅读的位置 vs 群最新位置
            local_max = self.get_user_read_seq(user_id, group.id)
            group_max = self.get_group_max_seq(group.id)

            if local_max < group_max:
                # 拉取差量消息
                messages = self.load_group_messages(
                    group.id, from_seq=local_max + 1, to_seq=group_max
                )
                self.push_to_user(user_id, messages)
                # 更新用户 read_seq
                self.update_read_seq(user_id, group.id, group_max)
```

**小群和大群的策略对比：**

| 维度 | 小群（≤500人） | 大群（>500人） |
|------|---------------|--------------|
| 写入量 | 500次/条消息 | 1次/条消息 |
| 读取延迟 | 低（收件箱预存） | 中（需拉取） |
| 存储成本 | 高（500份副本） | 低（1份+seq索引） |
| 离线同步 | 收件箱自动积累 | 上线时拉取差量 |

### 已读回执：只存最大已读 seq

**问题：** 10 万人群 × 每条消息 × 每人一个已读状态 = 不可承受的存储量。

**具体计算：** 10 万群 × 10 条/秒 × 86400秒 × 10 万成员 = 86.4 万亿条/天 → 不可行。

**方案：每个用户每个会话只存一个"最大已读 seq"值**

```sql
CREATE TABLE read_progress (
    user_id VARCHAR(64),
    conversation_id VARCHAR(64),
    read_seq BIGINT NOT NULL,    -- 已读到的最大 seq
    updated_at TIMESTAMP,
    PRIMARY KEY (user_id, conversation_id)
);
```

**存储量：** 3000 万 DAU × 平均 100 个会话 = 3 亿条记录 → 可行。

**已读逻辑：**
```
用户打开会话 → 上报最大 seq 为 read_seq
发送方查看已读状态 → 对比消息 seq 和 read_progress
  msg.seq <= read_seq → 已读
  msg.seq > read_seq → 未读
```

**群聊已读人数的近似计算：**

```python
def get_read_count(self, group_id, msg_seq):
    # 不逐个查 read_progress（太慢）
    # 使用近似计数：定期聚合到 Redis
    count = self.redis.get(f"group:{group_id}:msg:{msg_seq}:read_count")
    if count is None:
        count = self.db.query_one("""
            SELECT COUNT(*) FROM read_progress
            WHERE conversation_id = %s AND read_seq >= %s
        """, group_id, msg_seq)
        self.redis.setex(
            f"group:{group_id}:msg:{msg_seq}:read_count", 
            60, str(count)  # 缓存 1 分钟
        )
    return int(count)
```

### 在线状态管理：Redis Bitmap

```redis
# 用户在线状态：按用户ID分片的 Bitmap
SETBIT online:shard:{userId // 1000000} {userId % 1000000} 1

# 查询在线状态
GETBIT online:shard:{userId // 1000000} {userId % 1000000}

# 每片约 125KB（100万位），100片覆盖 1 亿用户，总计约 12.5MB
```

**为什么用 Bitmap 而非 Set？**

| 方案 | 内存占用 | 查询延迟 | 适用 |
|------|---------|---------|------|
| Redis Set | ~500MB（1 亿用户）| O(1) | 需要遍历在线用户列表 |
| Redis Bitmap | ~12.5MB | O(1) | 只需判断在线/离线 |

IM 场景只需要判断"某用户是否在线" → Bitmap 足够，内存节省 97%。

### 客户端幂等去重

```python
class LocalMessageStore:
    def on_message_received(self, msg):
        # 检查本地是否已存在（用 msg_id 去重）
        if self.local_db.exists(msg.msg_id):
            return  # 重复消息，忽略
        self.local_db.insert(msg)
        self.render(msg)
```

**为什么需要去重？** 服务端"至少一次"投递意味着同一消息可能推送多次（如网络超时重试）。客户端用 msg_id 去重 → 保证"恰好一次"语义。

### 消息搜索

```
消息写入 → Kafka → Flink → Elasticsearch

ES 索引设计（按用户分索引，实现数据隔离）：
  索引：messages_{userId}
  字段：conversation_id, sender_id, content(text分词), timestamp, msg_type
```

## 常见陷阱（深度分析）

### 陷阱 1：大群写扩散

**具体计算：** 10 万人大群每秒 10 条消息 × 10 万写入 = 100 万 QPS。3 节点 Redis 集群勉强承受，但如果有 5 个这样的群 = 500 万 QPS → 不可行。

### 陷阱 2：每条消息存已读状态

**存储量：** 10 万群 × 10 条/秒 × 86400秒/天 × 10 万成员 = 86.4 万亿条记录 → 不可行。

最大已读 seq 方案：3000 万 DAU × 平均 100 个会话 = 3 亿条记录 → 可行。

### 陷阱 3：先推后存

**消息丢失场景：** 推送成功但存储失败 → 接收方看到了消息但服务器无记录 → 消息"消失"（接收方刷新后找不到）。

### 陷阱 4：消息 ID 不保证序

**乱序场景：** 两个用户同时发消息，Snowflake ID 可能乱序（时钟偏差）→ 客户端显示顺序不正确 → 对话逻辑混乱。

**解决方案：** 用会话内自增 seq 排序，而非 Snowflake ID 排序。

## 延伸思考

- **端到端加密**：服务器无法解密消息内容 → 搜索只能在客户端本地做。架构需增加密钥管理服务。
- **音视频通话信令**：与消息系统共用 WebSocket 通道，但需要 SDP 交换和 ICE 协商。
- **跨设备同步**：手机和电脑同时登录，消息和已读状态如何多端一致？使用 seq 作为同步锚点。