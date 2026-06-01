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

## 完整数据库设计

### 核心表结构

```sql
-- ============================================================
-- 1. 会话表：存储所有会话（单聊 / 群聊）元信息
-- ============================================================
CREATE TABLE conversations (
    conversation_id VARCHAR(64) PRIMARY KEY,  -- 会话ID，单聊=sorted(uid1_uid2)，群聊=group_id
    conv_type TINYINT NOT NULL,               -- 1=单聊, 2=群聊
    max_seq BIGINT NOT NULL DEFAULT 0,        -- 当前最大 seq（原子递增）
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_updated (updated_at)            -- 按活跃时间排序
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ============================================================
-- 2. 消息表：按 conversation_id 分库，按月分表
--    分库：hash(conversation_id) % 256
--    分表：messages_{db_index}_{yyyyMM}
-- ============================================================
CREATE TABLE messages (
    msg_id BIGINT PRIMARY KEY,                -- Snowflake ID，全局唯一
    seq BIGINT NOT NULL,                      -- 会话内自增序号，排序和同步用
    conversation_id VARCHAR(64) NOT NULL,
    sender_id VARCHAR(64) NOT NULL,
    content_type TINYINT NOT NULL,            -- 1=文本, 2=图片, 3=文件, 4=视频, 5=已撤回, 6=系统消息
    content TEXT,                              -- 文本内容或文件URL（加密时为密文）
    client_msg_id VARCHAR(128) NOT NULL,      -- 客户端生成，用于服务端去重
    msg_status TINYINT NOT NULL DEFAULT 0,    -- 0=正常, 1=已撤回, 2=已删除
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_conv_seq (conversation_id, seq),
    UNIQUE KEY uk_client_msg (sender_id, client_msg_id),
    INDEX idx_conv_created (conversation_id, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ============================================================
-- 3. 用户连接表：记录 WebSocket 长连接的路由信息
-- ============================================================
CREATE TABLE user_connections (
    user_id VARCHAR(64) NOT NULL,
    device_id VARCHAR(64) NOT NULL,           -- 支持多设备同时在线
    gateway_id VARCHAR(64) NOT NULL,          -- 接入网关实例ID
    conn_state TINYINT NOT NULL DEFAULT 1,    -- 1=在线, 0=离线
    connected_at TIMESTAMP NOT NULL,
    last_heartbeat TIMESTAMP NOT NULL,
    client_type TINYINT NOT NULL,             -- 1=iOS, 2=Android, 3=Web, 4=Desktop
    PRIMARY KEY (user_id, device_id),
    INDEX idx_gateway (gateway_id)            -- 网关故障时批量清理
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ============================================================
-- 4. 群成员表：存储群聊成员关系和角色
-- ============================================================
CREATE TABLE group_members (
    group_id VARCHAR(64) NOT NULL,
    user_id VARCHAR(64) NOT NULL,
    role TINYINT NOT NULL DEFAULT 0,          -- 0=普通成员, 1=管理员, 2=群主
    join_seq BIGINT NOT NULL DEFAULT 0,       -- 加入时的群 seq（决定能看到哪些历史消息）
    leave_seq BIGINT BIGINT DEFAULT NULL,     -- 退出时的群 seq（NULL 表示仍在群中）
    join_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (group_id, user_id),
    INDEX idx_user (user_id)                  -- 查询用户所在的所有群
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ============================================================
-- 5. 已读进度表：每个用户每个会话只存一条记录
-- ============================================================
CREATE TABLE read_progress (
    user_id VARCHAR(64) NOT NULL,
    conversation_id VARCHAR(64) NOT NULL,
    read_seq BIGINT NOT NULL DEFAULT 0,       -- 已读到的最大 seq
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, conversation_id),
    INDEX idx_conv_read (conversation_id, read_seq)  -- 聚合已读人数
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ============================================================
-- 6. 会话序号表：原子递增 seq
-- ============================================================
CREATE TABLE conversation_seq (
    conversation_id VARCHAR(64) PRIMARY KEY,
    current_seq BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 分库分表路由规则

```python
class MessageSharding:
    """消息表分库分表路由"""

    DB_COUNT = 256  # 分库数
    TABLE_PER_DB = 12  # 每库按月分表

    def get_db_index(self, conversation_id: str) -> int:
        """根据 conversation_id 路由到数据库实例"""
        return int(hashlib.md5(conversation_id.encode()).hexdigest(), 16) % self.DB_COUNT

    def get_table_name(self, conversation_id: str, timestamp: datetime) -> str:
        """根据时间路由到月份表"""
        db_index = self.get_db_index(conversation_id)
        month_suffix = timestamp.strftime("%Y%m")
        return f"messages_{db_index}_{month_suffix}"

    def get_all_tables_for_query(self, conversation_id: str,
                                  start_time: datetime, end_time: datetime) -> list:
        """跨月查询时返回所有涉及的表名"""
        tables = []
        current = start_time.replace(day=1)
        while current <= end_time:
            tables.append(self.get_table_name(conversation_id, current))
            # 移动到下个月
            if current.month == 12:
                current = current.replace(year=current.year + 1, month=1)
            else:
                current = current.replace(month=current.month + 1)
        return tables
```

### 数据量估算

```
会话表：1 亿用户 × 平均 100 个会话 = 100 亿条 → 分 256 库，每库约 390 万条 → 可行
消息表：50 亿条/天 × 30 天 = 1500 亿条 → 按月分表，每月 1500 亿 / 256 库 = 5860 万条/库/月 → 可行
用户连接表：3000 万 DAU × 1.2 设备 = 3600 万条 → 单库可行
群成员表：10 万人大群 × 1000 个 = 1 亿条 → 分库可行
已读进度表：3000 万 DAU × 100 会话 = 30 亿条 → 分 256 库，每库约 117 万条 → 可行
```

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

#### 完整的 At-Least-Once 投递 + 去重实现

```python
import time
import logging
from dataclasses import dataclass
from typing import Optional

logger = logging.getLogger(__name__)

@dataclass
class Message:
    msg_id: int = 0
    seq: int = 0
    conversation_id: str = ""
    sender_id: str = ""
    client_msg_id: str = ""      # 客户端生成的唯一ID
    content_type: int = 1
    content: str = ""
    created_at: float = 0.0

class MessageDeliveryService:
    """消息投递服务：实现 at-least-once 语义 + 去重"""

    def __init__(self, storage, redis, presence, push_service):
        self.storage = storage
        self.redis = redis
        self.presence = presence       # 在线状态服务
        self.push_service = push_service
        self.dedup = MessageDeduplicator(redis)
        self.seq_generator = SeqGenerator(storage)

    def send_message(self, msg: Message) -> dict:
        """
        消息发送完整流程：
        1. 服务端去重（防止客户端重试导致重复消息）
        2. 生成全局唯一 msg_id + 会话内自增 seq
        3. 持久化到存储（先存后推的 WAL 原则）
        4. 异步推送（在线走 WebSocket，离线走厂商推送）
        5. 确认发送方
        """
        # ---- 第一步：服务端去重 ----
        dedup_result = self.dedup.check_and_mark(msg.sender_id, msg.client_msg_id)
        if dedup_result is not None:
            # 重复消息，返回已处理的 msg_id（幂等）
            logger.info(f"重复消息 dedup: sender={msg.sender_id}, "
                       f"client_msg_id={msg.client_msg_id}, "
                       f"existing_msg_id={dedup_result}")
            return {"msg_id": dedup_result, "status": "duplicate"}

        # ---- 第二步：生成 msg_id + seq ----
        msg.msg_id = snowflake_id()
        msg.seq = self.seq_generator.next_seq(msg.conversation_id)
        msg.created_at = time.time()

        # ---- 第三步：持久化（先存后推） ----
        try:
            self.storage.save_message(msg)
        except Exception as e:
            # 存储失败 → 清除去重标记，允许重试
            self.dedup.clear(msg.sender_id, msg.client_msg_id)
            logger.error(f"消息存储失败: conv={msg.conversation_id}, error={e}")
            return {"status": "storage_failed", "error": str(e)}

        # ---- 第四步：异步推送 ----
        # 推送失败不影响发送方确认，接收方上线后可通过 seq 差量拉取
        try:
            self._deliver_to_recipients(msg)
        except Exception as e:
            # 推送失败只记录日志，不回滚
            logger.warning(f"推送失败（接收方可后续拉取）: conv={msg.conversation_id}, "
                          f"error={e}")

        # ---- 第五步：确认发送方 ----
        return {"msg_id": msg.msg_id, "seq": msg.seq, "status": "ok"}

    def _deliver_to_recipients(self, msg: Message):
        """投递给接收方：在线推送 + 离线推送"""
        if self._is_group(msg.conversation_id):
            group_id = msg.conversation_id
            member_count = self.get_group_member_count(group_id)

            if member_count <= 500:
                # 小群：写扩散——为每个成员写收件箱
                self._write_fanout(msg, group_id)
            else:
                # 大群：读扩散——只写群消息表 + 推送在线成员
                self._read_fanout(msg, group_id)
        else:
            # 单聊：直接推送
            recipient_id = self._get_peer_id(msg.conversation_id, msg.sender_id)
            self._deliver_to_user(recipient_id, msg)

    def _deliver_to_user(self, user_id: str, msg: Message):
        """投递给单个用户"""
        if self.presence.is_online(user_id):
            # 在线：WebSocket 推送，等待 ACK
            success = self.push_service.push_online(user_id, msg)
            if not success:
                # WebSocket 推送失败 → 写入离线消息队列
                self._save_offline_message(user_id, msg)
        else:
            # 离线：写入离线消息队列 + 厂商推送
            self._save_offline_message(user_id, msg)
            self.push_service.push_offline(user_id, msg)

    def _save_offline_message(self, user_id: str, msg: Message):
        """保存离线消息，用户上线后拉取"""
        key = f"offline_msg:{user_id}:{msg.conversation_id}"
        self.redis.zadd(key, {msg.msg_id: msg.seq})
        # 设置 7 天过期，超期未拉取则靠 seq 差量从 DB 拉取
        self.redis.expire(key, 7 * 86400)


class SeqGenerator:
    """会话内自增 seq 生成器（保证同会话严格有序）"""

    def __init__(self, storage):
        self.storage = storage
        self.local_cache = {}  # conversation_id -> (current_seq, last_access_time)
        self.lock = threading.Lock()

    def next_seq(self, conversation_id: str) -> int:
        """
        生成下一个 seq。
        使用 Redis 原子递增做分布式锁保护，避免并发时 seq 重复。
        """
        key = f"conv_seq:{conversation_id}"
        seq = self.redis.incr(key)
        # Redis 中的 seq 做缓存，定期与 DB 同步
        if seq == 1:
            # 首次访问，从 DB 加载当前值
            db_seq = self.storage.get_current_seq(conversation_id)
            seq = self.redis.incrby(key, db_seq)
        return seq


class MessageDeduplicator:
    """服务端消息去重器：基于 client_msg_id 去重"""

    def __init__(self, redis):
        self.redis = redis

    def check_and_mark(self, sender_id: str, client_msg_id: str) -> Optional[int]:
        """
        检查是否重复并标记。
        返回 None 表示非重复，返回 msg_id 表示重复。
        """
        key = f"msg_dedup:{sender_id}:{client_msg_id}"
        # SETNX 保证原子性
        if self.redis.setnx(key, ""):
            # 非重复消息，设置 5 分钟过期
            self.redis.expire(key, 300)
            return None
        else:
            # 重复消息，返回已处理的 msg_id
            existing = self.redis.get(key)
            return int(existing) if existing else 0

    def update_msg_id(self, sender_id: str, client_msg_id: str, msg_id: int):
        """消息处理完成后，将 msg_id 写入去重记录"""
        key = f"msg_dedup:{sender_id}:{client_msg_id}"
        self.redis.set(key, str(msg_id), ex=300)

    def clear(self, sender_id: str, client_msg_id: str):
        """处理失败时清除去重标记，允许重试"""
        key = f"msg_dedup:{sender_id}:{client_msg_id}"
        self.redis.delete(key)
```

#### 推送 ACK 与重试机制

```python
class PushAckManager:
    """
    推送确认机制：
    - 在线推送后等待客户端 ACK，超时未确认则重试
    - 最多重试 3 次，仍失败则转离线消息队列
    """

    MAX_RETRIES = 3
    ACK_TIMEOUT = 5  # 秒

    def __init__(self, redis, push_service, offline_queue):
        self.redis = redis
        self.push_service = push_service
        self.offline_queue = offline_queue

    def push_with_ack(self, user_id: str, msg: Message):
        """推送消息并等待 ACK"""
        pending_key = f"push_pending:{user_id}:{msg.msg_id}"

        for attempt in range(self.MAX_RETRIES):
            # 推送消息
            self.push_service.send_to_user(user_id, msg)

            # 记录待确认状态
            self.redis.setex(pending_key, self.ACK_TIMEOUT, str(attempt))

            # 等待 ACK（异步事件驱动，这里用简化逻辑表示）
            ack_received = self._wait_for_ack(user_id, msg.msg_id, self.ACK_TIMEOUT)
            if ack_received:
                self.redis.delete(pending_key)
                return True

            logger.warning(f"推送 ACK 超时: user={user_id}, msg_id={msg.msg_id}, "
                          f"attempt={attempt + 1}")

        # 重试耗尽 → 转入离线队列
        self.offline_queue.enqueue(user_id, msg)
        self.redis.delete(pending_key)
        return False

    def on_ack_received(self, user_id: str, msg_id: int):
        """收到客户端 ACK"""
        pending_key = f"push_pending:{user_id}:{msg_id}"
        self.redis.delete(pending_key)

    def _wait_for_ack(self, user_id, msg_id, timeout):
        """等待 ACK（实际实现为事件驱动，此处简化）"""
        # 实际实现使用 Redis Pub/Sub 或内存事件
        deadline = time.time() + timeout
        while time.time() < deadline:
            if self.redis.get(f"ack:{user_id}:{msg_id}"):
                return True
            time.sleep(0.1)
        return False
```

#### 已读回执聚合

```python
class ReadReceiptAggregator:
    """
    已读回执聚合器：
    - 客户端上报已读进度时，先写入 Redis 缓冲区
    - 定期（每 10 秒）批量聚合写入 DB
    - 避免每条消息每人都触发一次 DB 写入
    """

    BATCH_SIZE = 500
    FLUSH_INTERVAL = 10  # 秒

    def __init__(self, redis, db):
        self.redis = redis
        self.db = db

    def report_read(self, user_id: str, conversation_id: str, read_seq: int):
        """
        客户端上报已读进度。
        只在 read_seq 大于当前值时才更新。
        """
        key = f"read_buf:{user_id}:{conversation_id}"
        current = self.redis.get(key)
        if current is None or int(current) < read_seq:
            self.redis.set(key, str(read_seq))
            # 加入待刷写队列
            self.redis.sadd("read_flush_queue", f"{user_id}:{conversation_id}")

    def flush_to_db(self):
        """定时批量刷写到 DB"""
        # 获取所有待刷写的键
        keys = self.redis.spop("read_flush_queue", self.BATCH_SIZE)
        if not keys:
            return

        updates = []
        for key_str in keys:
            user_id, conv_id = key_str.split(":", 1)
            buf_key = f"read_buf:{user_id}:{conv_id}"
            read_seq = self.redis.get(buf_key)
            if read_seq:
                updates.append((user_id, conv_id, int(read_seq)))
                self.redis.delete(buf_key)

        if updates:
            # 批量 UPSERT
            self.db.executemany("""
                INSERT INTO read_progress (user_id, conversation_id, read_seq)
                VALUES (%s, %s, %s)
                ON DUPLICATE KEY UPDATE
                    read_seq = GREATEST(read_seq, VALUES(read_seq)),
                    updated_at = CURRENT_TIMESTAMP
            """, updates)

        logger.info(f"已读回执刷写完成: {len(updates)} 条记录")

    def get_read_count(self, conversation_id: str, msg_seq: int) -> int:
        """
        获取群聊中某条消息的已读人数。
        先查 Redis 缓存，缓存未命中则查 DB 并回填。
        """
        cache_key = f"read_count:{conversation_id}:{msg_seq}"
        count = self.redis.get(cache_key)
        if count is not None:
            return int(count)

        # DB 查询
        count = self.db.query_one("""
            SELECT COUNT(*) FROM read_progress
            WHERE conversation_id = %s AND read_seq >= %s
        """, conversation_id, msg_seq)

        # 缓存 1 分钟
        self.redis.setex(cache_key, 60, str(count))
        return count
```

**为什么先存后推？**

| 顺序 | 存储 | 推送 | 后果 |
|------|------|------|------|
| 先推后存 | 失败 | 成功 | 消息"消失"（接收方看到了但刷新后找不到） |
| 先存后推 | 成功 | 失败 | 接收方未收到，但可主动拉取 → 不丢消息 |

先存后推：存储成功但推送失败 → 接收方未收到 → 上线时通过 seq 差量拉取 → 不丢消息。

### 大群消息：读扩散为主 + Write-Behind 优化

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

#### Write-Behind：大群在线成员的增量同步

大群在线成员可能有数万人，逐个推送延迟过高。采用 write-behind 策略：消息先写入增量日志，客户端定期拉取。

```python
class LargeGroupFanout:
    """
    大群扇出优化：
    1. 消息写入群消息表（1 次写入）
    2. 更新群的 max_seq（1 次 Redis INCR）
    3. 推送"有新消息"通知给在线成员（轻量级信号，非完整消息体）
    4. 客户端收到通知后主动拉取增量消息
    """

    def __init__(self, redis, db, presence):
        self.redis = redis
        self.db = db
        self.presence = presence

    def on_new_message(self, msg, group_id: str):
        """
        大群新消息处理流程
        """
        # 1. 写入群消息表（只写 1 次）
        self.db.save_message(msg)

        # 2. 更新群 max_seq
        self.redis.set(f"group_max_seq:{group_id}", msg.seq)

        # 3. 写入增量日志（Redis Sorted Set，按 seq 排序）
        incr_key = f"group_incr:{group_id}"
        self.redis.zadd(incr_key, {
            self._serialize_msg_meta(msg): msg.seq
        })
        # 增量日志保留 5 分钟，超期靠 DB 拉取
        self.redis.expire(incr_key, 300)

        # 4. 推送轻量级通知给在线成员
        #    只告知"群 X 有新消息，最新 seq=Y"，不推送完整消息体
        online_members = self.presence.get_online_members(group_id)
        notification = {
            "type": "group_update",
            "group_id": group_id,
            "max_seq": msg.seq,
            "sender_id": msg.sender_id  # 用于推送提示
        }
        self._batch_notify(online_members, notification)

    def _batch_notify(self, user_ids: list, notification: dict):
        """批量推送通知，按网关分组减少网络开销"""
        # 按网关实例分组
        gateway_groups = {}
        for uid in user_ids:
            gateway_id = self.presence.get_gateway(uid)
            if gateway_id:
                gateway_groups.setdefault(gateway_id, []).append(uid)

        # 每个网关发一次批量通知
        for gateway_id, uids in gateway_groups.items():
            self.redis.publish(
                f"gateway:{gateway_id}:notifications",
                json.dumps({"uids": uids, "notification": notification})
            )

    def _serialize_msg_meta(self, msg) -> str:
        """序列化消息元数据（用于增量日志）"""
        return json.dumps({
            "msg_id": msg.msg_id,
            "seq": msg.seq,
            "sender_id": msg.sender_id,
            "content_type": msg.content_type,
            "content": msg.content[:200] if msg.content else ""  # 截断
        })


class IncrementalSyncService:
    """
    增量同步服务：客户端拉取大群增量消息
    """

    def sync_group_messages(self, user_id: str, group_id: str,
                            client_max_seq: int) -> dict:
        """
        客户端上报自己的 max_seq，服务端返回增量消息。

        优先从 Redis 增量日志读取（快），
        未命中则从 DB 读取（稍慢但不丢）。
        """
        group_max_seq = int(self.redis.get(f"group_max_seq:{group_id}") or 0)

        if client_max_seq >= group_max_seq:
            return {"has_new": False, "messages": [], "max_seq": group_max_seq}

        # 尝试从 Redis 增量日志读取
        incr_key = f"group_incr:{group_id}"
        raw_messages = self.redis.zrangebyscore(
            incr_key, client_max_seq + 1, group_max_seq
        )

        if raw_messages:
            messages = [json.loads(m) for m in raw_messages]
        else:
            # Redis 增量日志过期，从 DB 拉取
            messages = self.db.query("""
                SELECT msg_id, seq, sender_id, content_type, content
                FROM messages
                WHERE conversation_id = %s AND seq > %s AND seq <= %s
                ORDER BY seq ASC
                LIMIT 100
            """, group_id, client_max_seq, group_max_seq)

        return {
            "has_new": True,
            "messages": messages,
            "max_seq": group_max_seq
        }
```

#### Write-Behind 批量持久化策略

对于需要写扩散的场景（如小群收件箱更新），采用 write-behind 模式减少 DB 压力：

```python
class WriteBehindBuffer:
    """
    Write-Behind 缓冲区：
    - 写操作先进入内存缓冲区
    - 定时或缓冲区满时批量刷写到 DB
    - 减少小群写扩散的 DB 写入频次
    """

    BUFFER_SIZE = 1000
    FLUSH_INTERVAL_MS = 200  # 200ms

    def __init__(self, redis, db):
        self.redis = redis
        self.db = db
        self.buffer = {}  # (user_id, conv_id) -> max_seq

    def add_inbox_update(self, user_id: str, conversation_id: str, seq: int):
        """写入缓冲区"""
        key = (user_id, conversation_id)
        current = self.buffer.get(key, 0)
        if seq > current:
            self.buffer[key] = seq

        # 缓冲区满 → 触发刷写
        if len(self.buffer) >= self.BUFFER_SIZE:
            self.flush()

    def flush(self):
        """批量刷写缓冲区到 DB"""
        if not self.buffer:
            return

        updates = list(self.buffer.items())
        self.buffer.clear()

        # 批量 UPSERT
        batch = [(uid, conv_id, seq) for (uid, conv_id), seq in updates]
        self.db.executemany("""
            INSERT INTO user_inbox (user_id, conversation_id, max_seq)
            VALUES (%s, %s, %s)
            ON DUPLICATE KEY UPDATE
                max_seq = GREATEST(max_seq, VALUES(max_seq))
        """, batch)

        logger.info(f"Write-behind 刷写: {len(batch)} 条收件箱更新")
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

### 在线状态管理：Redis Bitmap + Presence Server

#### Presence Server 完整实现

```python
class PresenceServer:
    """
    在线状态管理服务（Presence Server）：
    1. 维护全局在线状态（Redis Bitmap）
    2. 管理用户连接路由（哪个网关）
    3. 处理多设备同时在线
    4. 发布在线状态变更通知
    5. 支持批量查询（大群推送时需要）
    """

    def __init__(self, redis, db):
        self.redis = redis
        self.db = db
        self.BITMAP_SHARD_SIZE = 1_000_000  # 每片 100 万用户

    def on_user_connect(self, user_id: str, device_id: str,
                        gateway_id: str, client_type: int):
        """用户 WebSocket 连接建立"""
        # 1. 记录连接路由信息（支持多设备）
        conn_key = f"user_conn:{user_id}"
        self.redis.hset(conn_key, device_id, json.dumps({
            "gateway_id": gateway_id,
            "client_type": client_type,
            "connected_at": time.time()
        }))

        # 2. 设置在线状态 Bitmap
        self._set_online_bitmap(user_id, online=True)

        # 3. 记录活跃会话列表（用于上线后同步）
        self.redis.sadd(f"online_users:{gateway_id}", user_id)

        # 4. 发布在线状态变更事件
        self.redis.publish("presence_events", json.dumps({
            "type": "online",
            "user_id": user_id,
            "device_id": device_id,
            "timestamp": time.time()
        }))

    def on_user_disconnect(self, user_id: str, device_id: str,
                           gateway_id: str):
        """用户 WebSocket 断开"""
        conn_key = f"user_conn:{user_id}"

        # 1. 删除该设备的连接记录
        self.redis.hdel(conn_key, device_id)

        # 2. 检查是否还有其他设备在线
        remaining = self.redis.hlen(conn_key)
        if remaining == 0:
            # 所有设备都离线 → 延迟 30 秒设为离线（避免短暂断连）
            self.redis.setex(
                f"offline_pending:{user_id}", 30, str(time.time())
            )
            # 延迟任务检查是否真的离线
            self._schedule_offline_check(user_id, delay=30)

        # 3. 从网关在线用户集合中移除
        self.redis.srem(f"online_users:{gateway_id}", user_id)

    def _schedule_offline_check(self, user_id: str, delay: int):
        """延迟检查用户是否真的离线"""
        # 使用 Redis Sorted Set 做延迟队列
        check_time = time.time() + delay
        self.redis.zadd("offline_check_queue", {user_id: check_time})

    def process_offline_checks(self):
        """定时处理离线检查队列"""
        now = time.time()
        # 获取所有到期的检查任务
        pending = self.redis.zrangebyscore("offline_check_queue", 0, now)
        for user_id in pending:
            # 检查用户是否仍然没有设备在线
            conn_key = f"user_conn:{user_id}"
            if self.redis.hlen(conn_key) == 0:
                # 确认离线
                self._set_online_bitmap(user_id, online=False)
                self.redis.delete(f"offline_pending:{user_id}")

                # 发布离线事件
                self.redis.publish("presence_events", json.dumps({
                    "type": "offline",
                    "user_id": user_id,
                    "timestamp": time.time()
                }))

            # 移除已处理的检查任务
            self.redis.zrem("offline_check_queue", user_id)

    def _set_online_bitmap(self, user_id: str, online: bool):
        """设置用户在线状态 Bitmap"""
        uid = int(user_id)
        shard_index = uid // self.BITMAP_SHARD_SIZE
        bit_offset = uid % self.BITMAP_SHARD_SIZE
        key = f"online:shard:{shard_index}"
        self.redis.setbit(key, bit_offset, 1 if online else 0)

    def is_online(self, user_id: str) -> bool:
        """查询单个用户在线状态"""
        uid = int(user_id)
        shard_index = uid // self.BITMAP_SHARD_SIZE
        bit_offset = uid % self.BITMAP_SHARD_SIZE
        return bool(self.redis.getbit(f"online:shard:{shard_index}", bit_offset))

    def batch_check_online(self, user_ids: list) -> dict:
        """批量查询在线状态（大群推送时使用）"""
        result = {}
        # 按分片分组，减少 Redis 命令数
        shard_groups = {}
        for uid_str in user_ids:
            uid = int(uid_str)
            shard_index = uid // self.BITMAP_SHARD_SIZE
            bit_offset = uid % self.BITMAP_SHARD_SIZE
            shard_groups.setdefault(shard_index, []).append(
                (uid_str, bit_offset)
            )

        # 每个分片一次 BITFIELD 批量查询
        for shard_index, items in shard_groups.items():
            key = f"online:shard:{shard_index}"
            # 使用 BITFIELD 批量读取多个位
            for uid_str, bit_offset in items:
                val = self.redis.getbit(key, bit_offset)
                result[uid_str] = bool(val)

        return result

    def get_online_members(self, group_id: str) -> list:
        """
        获取群组中在线的成员列表。
        优化：先用 Bitmap 快速过滤，再查连接路由。
        """
        # 获取群成员列表
        members = self.db.query("""
            SELECT user_id FROM group_members
            WHERE group_id = %s AND leave_seq IS NULL
        """, group_id)

        # 批量检查在线状态
        online_status = self.batch_check_online([m['user_id'] for m in members])
        return [uid for uid, online in online_status.items() if online]

    def get_gateway(self, user_id: str) -> str:
        """获取用户当前连接的网关 ID"""
        conn_key = f"user_conn:{user_id}"
        conns = self.redis.hgetall(conn_key)
        if not conns:
            return None
        # 返回最近活跃的设备所在网关
        latest = max(conns.values(), key=lambda c: json.loads(c)['connected_at'])
        return json.loads(latest)['gateway_id']

    def on_gateway_failure(self, gateway_id: str):
        """
        网关故障处理：
        批量清理该网关下所有用户的连接记录和在线状态
        """
        # 获取该网关所有在线用户
        user_ids = self.redis.smembers(f"online_users:{gateway_id}")

        for uid in user_ids:
            conn_key = f"user_conn:{uid}"
            # 删除该网关相关的设备连接
            conns = self.redis.hgetall(conn_key)
            for device_id, conn_info in conns.items():
                if json.loads(conn_info)['gateway_id'] == gateway_id:
                    self.redis.hdel(conn_key, device_id)

            # 检查是否还有其他网关的连接
            if self.redis.hlen(conn_key) == 0:
                self._set_online_bitmap(uid, online=False)
                self.redis.publish("presence_events", json.dumps({
                    "type": "offline",
                    "user_id": uid,
                    "reason": "gateway_failure",
                    "timestamp": time.time()
                }))

        # 清理网关在线用户集合
        self.redis.delete(f"online_users:{gateway_id}")
        logger.warning(f"网关故障处理完成: gateway={gateway_id}, "
                      f"影响用户数={len(user_ids)}")
```

#### Redis Bitmap 内存分析

```redis
# 用户在线状态：按用户ID分片的 Bitmap
SETBIT online:shard:{userId // 1000000} {userId % 1000000} 1

# 查询在线状态
GETBIT online:shard:{userId // 1000000} {userId % 1000000}

# 每片约 125KB（100万位），100片覆盖 1 亿用户，总计约 12.5MB
```

**为什么用 Bitmap 而非 Set？**

| 方案 | 内存占用 | 查询延迟 | 批量查询 | 适用 |
|------|---------|---------|---------|------|
| Redis Set | ~500MB（1 亿用户）| O(1) | SMEMBERS 全量 | 需要遍历在线用户列表 |
| Redis Bitmap | ~12.5MB | O(1) | BITFIELD 批量 | 只需判断在线/离线 |
| Redis Hash | ~200MB | O(1) | HMGET 批量 | 需要存储额外元信息 |

IM 场景只需要判断"某用户是否在线" → Bitmap 足够，内存节省 97%。

#### 在线状态变更订阅

```python
class PresenceSubscriber:
    """
    订阅在线状态变更事件：
    - 消息服务订阅，用于决策在线推送 vs 离线推送
    - UI 层订阅，用于展示联系人在线状态
    """

    def __init__(self, redis):
        self.redis = redis
        self.callbacks = {"online": [], "offline": []}

    def subscribe(self):
        pubsub = self.redis.pubsub()
        pubsub.subscribe("presence_events")
        for message in pubsub.listen():
            if message["type"] == "message":
                event = json.loads(message["data"])
                event_type = event["type"]
                for callback in self.callbacks.get(event_type, []):
                    callback(event)

    def on_online(self, callback):
        self.callbacks["online"].append(callback)

    def on_offline(self, callback):
        self.callbacks["offline"].append(callback)
```

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

### 消息搜索：完整的 Elasticsearch 实现

#### 搜索架构总览

```
消息写入 MySQL → Binlog CDC → Kafka → Flink 消费 → Elasticsearch 双写

核心设计原则：
1. 按用户分索引，实现数据隔离（用户只能搜到自己参与的消息）
2. Kafka 保证消息写入与索引构建的最终一致性
3. Flink 做实时流处理，保证搜索延迟 < 5 秒
```

#### Elasticsearch 索引设计

```python
# ============================================================
# ES 索引设计：按用户分索引（messages_{userId}）
# 每个用户一个独立索引，天然实现数据隔离
# 优势：权限控制零成本，用户只能搜索自己的索引
# 劣势：索引数量多（1 亿用户 = 1 亿索引）→ 使用索引别名 + Rollover 管理
# ============================================================

MESSAGE_INDEX_TEMPLATE = {
    "index_patterns": ["messages_*"],
    "settings": {
        "number_of_shards": 1,          # 单用户索引通常 1 分片足够
        "number_of_replicas": 1,
        "analysis": {
            "analyzer": {
                "ik_smart_analyzer": {   # 中文智能分词
                    "type": "custom",
                    "tokenizer": "ik_smart"
                },
                "ik_max_word_analyzer": { # 中文最细粒度分词（搜索时用）
                    "type": "custom",
                    "tokenizer": "ik_max_word"
                },
                "pinyin_analyzer": {      # 拼音搜索
                    "type": "custom",
                    "tokenizer": "pinyin"
                }
            }
        },
        "index.lifecycle.name": "message_search_policy",  # ILM 策略
        "index.lifecycle.rollover_alias": "messages"
    },
    "mappings": {
        "properties": {
            "msg_id": {"type": "long"},
            "seq": {"type": "long"},
            "conversation_id": {"type": "keyword"},
            "conv_type": {"type": "keyword"},       # 1=单聊, 2=群聊
            "conv_name": {                           # 会话名称（群名/对方昵称）
                "type": "text",
                "analyzer": "ik_smart_analyzer",
                "fields": {
                    "keyword": {"type": "keyword"},
                    "pinyin": {"type": "text", "analyzer": "pinyin_analyzer"}
                }
            },
            "sender_id": {"type": "keyword"},
            "sender_name": {                         # 发送者名称
                "type": "text",
                "analyzer": "ik_smart_analyzer",
                "fields": {
                    "keyword": {"type": "keyword"},
                    "pinyin": {"type": "text", "analyzer": "pinyin_analyzer"}
                }
            },
            "content_type": {"type": "keyword"},
            "content": {                             # 消息内容
                "type": "text",
                "analyzer": "ik_smart_analyzer",
                "search_analyzer": "ik_max_word_analyzer",
                "fields": {
                    "keyword": {"type": "keyword", "ignore_above": 256},
                    "pinyin": {"type": "text", "analyzer": "pinyin_analyzer"}
                }
            },
            "file_name": {"type": "keyword"},        # 文件名（文件消息可搜文件名）
            "timestamp": {"type": "date"},
            "is_recalled": {"type": "boolean"},       # 是否已撤回
            "is_edited": {"type": "boolean"},          # 是否已编辑
            "edit_version": {"type": "integer"},       # 编辑版本号
            "attachments": {                           # 附件信息
                "type": "nested",
                "properties": {
                    "file_type": {"type": "keyword"},
                    "file_name": {
                        "type": "text",
                        "analyzer": "ik_smart_analyzer"
                    }
                }
            }
        }
    }
}
```

#### Kafka + Flink 实时索引同步

```python
import json
import logging
from datetime import datetime
from confluent_kafka import Consumer, KafkaError
from elasticsearch import Elasticsearch, helpers

logger = logging.getLogger(__name__)


class KafkaMessageIndexConsumer:
    """
    Kafka 消费者：监听消息写入事件，实时同步到 ES。
    数据流：MySQL Binlog → Debezium → Kafka → 本消费者 → ES

    保证机制：
    1. 每条消息的 Kafka offset 记录在 ES 的 _meta 字段中
    2. 消费者重启时从上次提交的 offset 继续消费
    3. 使用幂等写入（doc_id = msg_id），重复消费不产生副作用
    """

    def __init__(self, es_client: Elasticsearch, kafka_config: dict):
        self.es = es_client
        self.consumer = Consumer({
            **kafka_config,
            "group.id": "message-search-indexer",
            "enable.auto.commit": False,  # 手动提交 offset
            "max.poll.records": 500,
        })
        self.consumer.subscribe(["message_binlog_events"])
        self.buffer = []       # 批量写入缓冲区
        self.BUFFER_SIZE = 200  # 每 200 条批量写入一次
        self.last_commit_offset = 0

    def consume_loop(self):
        """主消费循环"""
        while True:
            msgs = self.consumer.consume(num_messages=100, timeout=1.0)
            for msg in msgs:
                if msg is None:
                    continue
                if msg.error():
                    if msg.error().code() == KafkaError._PARTITION_EOF:
                        continue
                    logger.error(f"Kafka 消费错误: {msg.error()}")
                    continue

                event = json.loads(msg.value().decode("utf-8"))
                self._handle_event(event, msg.offset())

            # 批量写入 ES
            if self.buffer:
                self._flush_to_es()
                # 手动提交 offset
                self.consumer.commit(asynchronous=False)

    def _handle_event(self, event: dict, kafka_offset: int):
        """处理单个 Binlog 事件"""
        operation = event.get("op")  # c=create, u=update, d=delete

        if operation == "c":
            # 新消息写入 → 创建搜索索引
            self._index_new_message(event["after"], kafka_offset)
        elif operation == "u":
            # 消息更新（编辑/撤回）→ 更新搜索索引
            self._update_message_index(event["after"], kafka_offset)
        elif operation == "d":
            # 消息删除 → 删除搜索索引
            self._delete_message_index(event["before"])

    def _index_new_message(self, row: dict, kafka_offset: int):
        """新消息索引构建：为会话中每个参与者写一份索引"""
        conversation_id = row["conversation_id"]
        participants = self._get_participants(conversation_id)

        for user_id in participants:
            doc = {
                "msg_id": row["msg_id"],
                "seq": row["seq"],
                "conversation_id": conversation_id,
                "conv_type": row.get("conv_type", "1"),
                "conv_name": self._get_conv_name(conversation_id, user_id),
                "sender_id": row["sender_id"],
                "sender_name": self._get_user_name(row["sender_id"]),
                "content_type": row["content_type"],
                "content": row.get("content", ""),
                "timestamp": row["created_at"],
                "is_recalled": False,
                "is_edited": False,
                "edit_version": 0,
                "_meta": {"kafka_offset": kafka_offset}
            }
            self.buffer.append({
                "_index": f"messages_{user_id}",
                "_id": str(row["msg_id"]),
                "_source": doc
            })

        if len(self.buffer) >= self.BUFFER_SIZE:
            self._flush_to_es()

    def _update_message_index(self, row: dict, kafka_offset: int):
        """消息更新索引（编辑/撤回）"""
        conversation_id = row["conversation_id"]
        participants = self._get_participants(conversation_id)

        update_fields = {}
        if row.get("msg_status") == 1:
            # 撤回
            update_fields = {"is_recalled": True, "content": "此消息已撤回"}
        elif row.get("edit_version", 0) > 0:
            # 编辑
            update_fields = {
                "content": row["content"],
                "is_edited": True,
                "edit_version": row["edit_version"]
            }

        if not update_fields:
            return

        for user_id in participants:
            self.buffer.append({
                "_index": f"messages_{user_id}",
                "_id": str(row["msg_id"]),
                "_source": {"doc": update_fields},
                "_op_type": "update"
            })

        if len(self.buffer) >= self.BUFFER_SIZE:
            self._flush_to_es()

    def _delete_message_index(self, row: dict):
        """消息删除：从所有用户索引中移除"""
        conversation_id = row["conversation_id"]
        participants = self._get_participants(conversation_id)

        for user_id in participants:
            self.buffer.append({
                "_index": f"messages_{user_id}",
                "_id": str(row["msg_id"]),
                "_op_type": "delete"
            })

    def _flush_to_es(self):
        """批量写入 ES（使用 bulk API）"""
        try:
            success, errors = helpers.bulk(self.es, self.buffer,
                                           raise_on_error=False)
            if errors:
                for err in errors:
                    logger.error(f"ES 写入失败: {err}")
            logger.info(f"ES 批量写入: 成功 {success} 条, 失败 {len(errors)} 条")
        except Exception as e:
            logger.error(f"ES 批量写入异常: {e}")
            # 写入失败 → 重试或写入死信队列
            self._write_to_dead_letter_queue(self.buffer)
        finally:
            self.buffer.clear()

    def _get_participants(self, conversation_id: str) -> list:
        """获取会话参与者列表（带缓存）"""
        cache_key = f"participants:{conversation_id}"
        cached = self.redis.get(cache_key)
        if cached:
            return json.loads(cached)

        # 单聊：解析 uid1_uid2
        if "_" in conversation_id and not conversation_id.startswith("grp_"):
            parts = conversation_id.split("_")
            participants = [parts[0], parts[1]]
        else:
            # 群聊：查询群成员
            members = self.db.query("""
                SELECT user_id FROM group_members
                WHERE group_id = %s AND leave_seq IS NULL
            """, conversation_id)
            participants = [m["user_id"] for m in members]

        # 缓存 10 分钟
        self.redis.setex(cache_key, 600, json.dumps(participants))
        return participants

    def _write_to_dead_letter_queue(self, failed_buffer):
        """写入死信队列，后续人工或自动重试"""
        for item in failed_buffer:
            self.redis.rpush("es_dead_letter_queue", json.dumps(item))


class MessageSearchService:
    """
    消息搜索服务：提供面向用户的消息搜索能力
    - 按用户索引隔离，天然权限控制
    - 支持全文搜索、按会话过滤、按时间范围、按发送者过滤
    - 支持高亮、拼音搜索、文件名搜索
    """

    def __init__(self, es_client: Elasticsearch):
        self.es = es_client

    def search(self, user_id: str, query: str,
               conversation_id: str = None,
               sender_id: str = None,
               start_time: str = None,
               end_time: str = None,
               content_type: str = None,
               page: int = 1,
               page_size: int = 20) -> dict:
        """
        搜索用户的消息
        - user_id: 当前用户，用于索引隔离
        - query: 搜索关键词（支持中文分词、拼音）
        - conversation_id: 指定会话搜索
        """
        must = []

        # 关键词匹配（主搜索条件）
        if query:
            must.append({
                "multi_match": {
                    "query": query,
                    "fields": ["content", "content.pinyin",
                               "conv_name", "conv_name.pinyin",
                               "sender_name", "sender_name.pinyin",
                               "file_name"],
                    "type": "best_fields",
                    "fuzziness": "AUTO"  # 允许模糊匹配
                }
            })

        # 过滤条件
        filters = []
        if conversation_id:
            filters.append({"term": {"conversation_id": conversation_id}})
        if sender_id:
            filters.append({"term": {"sender_id": sender_id}})
        if start_time or end_time:
            range_filter = {"range": {"timestamp": {}}}
            if start_time:
                range_filter["range"]["timestamp"]["gte"] = start_time
            if end_time:
                range_filter["range"]["timestamp"]["lte"] = end_time
            filters.append(range_filter)
        if content_type:
            filters.append({"term": {"content_type": content_type}})

        # 排除已撤回的消息（或标记为已撤回但仍可搜到）
        filters.append({"term": {"is_recalled": False}})

        body = {
            "query": {
                "bool": {
                    "must": must if must else [{"match_all": {}}],
                    "filter": filters
                }
            },
            "sort": [{"timestamp": "desc"}],
            "from": (page - 1) * page_size,
            "size": page_size,
            "highlight": {
                "pre_tags": ["<em>"],
                "post_tags": ["</em>"],
                "fields": {
                    "content": {"fragment_size": 150, "number_of_fragments": 3},
                    "conv_name": {},
                    "sender_name": {}
                }
            }
        }

        result = self.es.search(index=f"messages_{user_id}", body=body)

        hits = []
        for hit in result["hits"]["hits"]:
            source = hit["_source"]
            source["highlight"] = hit.get("highlight", {})
            hits.append(source)

        return {
            "total": result["hits"]["total"]["value"],
            "page": page,
            "page_size": page_size,
            "results": hits
        }

    def search_by_conversation(self, user_id: str, conversation_id: str,
                                query: str, page: int = 1) -> dict:
        """在指定会话内搜索"""
        return self.search(user_id, query,
                          conversation_id=conversation_id, page=page)

    def search_files(self, user_id: str, query: str,
                     file_type: str = None) -> dict:
        """搜索文件消息（按文件名搜索）"""
        must = [{"match": {"file_name": query}}]
        if file_type:
            must.append({"term": {"attachments.file_type": file_type}})

        result = self.es.search(index=f"messages_{user_id}", body={
            "query": {"bool": {"must": must}},
            "sort": [{"timestamp": "desc"}],
            "size": 50
        })
        return [hit["_source"] for hit in result["hits"]["hits"]]
```

#### 搜索索引生命周期管理

```python
# ILM（Index Lifecycle Management）策略：
# - 热阶段：0-30 天，SSD 存储，支持实时搜索
# - 温阶段：30-90 天，HDD 存储，搜索稍慢
# - 冷阶段：90 天以上，冻结索引，搜索需解冻

ILM_POLICY = {
    "policy": {
        "phases": {
            "hot": {
                "min_age": "0ms",
                "actions": {
                    "rollover": {"max_size": "50gb", "max_age": "30d"},
                    "set_priority": {"priority": 100}
                }
            },
            "warm": {
                "min_age": "30d",
                "actions": {
                    "shrink": {"number_of_shards": 1},
                    "forcemerge": {"max_num_segments": 1},
                    "set_priority": {"priority": 50}
                }
            },
            "cold": {
                "min_age": "90d",
                "actions": {
                    "freeze": {},
                    "set_priority": {"priority": 0}
                }
            }
        }
    }
}

# 搜索存储成本估算：
# 日均 50 亿条消息，每条索引约 500 字节（含分词倒排索引开销）
# 日增量：50 亿 × 500B = 2.5TB
# 热数据（30 天）：75TB SSD → ¥22.5 万/月
# 温数据（31-90 天）：150TB HDD → ¥4.5 万/月
# 冷数据（90 天+）：冻结，几乎无成本
# 合计：¥27 万/月 = ¥324 万/年
# 优化：非活跃用户索引延迟构建，日活 3000 万 → 只需维护 30% 索引 → ¥97 万/年
```

### 消息存储：分库分表设计

**50 亿条/天 × 永久存储 → 存储成本是核心问题。**

```sql
-- 分库分表策略：按 conversation_id 分 256 个库，每库按月分表
-- conversation_id 哈希 → db_index = hash(conv_id) % 256
-- 表名：messages_{db_index}_{yyyyMM}

CREATE TABLE messages (
    msg_id BIGINT PRIMARY KEY,        -- Snowflake ID
    seq INT NOT NULL,                  -- 会话内序号
    conversation_id VARCHAR(64) NOT NULL,
    sender_id VARCHAR(64) NOT NULL,
    content_type TINYINT NOT NULL,     -- 1=文本, 2=图片, 3=文件, 4=视频
    content TEXT,                       -- 文本内容或文件URL
    created_at TIMESTAMP NOT NULL,
    INDEX idx_conv_seq (conversation_id, seq)
);

-- 存储成本估算：
-- 每条消息平均 200 字节
-- 50 亿条/天 × 200B = 1TB/天
-- 30 天 = 30TB（热数据，MySQL）
-- 1 年 = 365TB → 归档到对象存储（S3/OSS）
-- MySQL 只保留最近 30 天，历史数据在 S3
```

**冷热分层存储：**

| 层 | 存储 | 保留时间 | 查询延迟 | 成本 |
|------|------|---------|---------|------|
| 热数据 | MySQL 分库分表 | 最近 30 天 | < 10ms | ¥0.5/GB/月 |
| 温数据 | Elasticsearch | 最近 90 天 | < 100ms | ¥0.3/GB/月 |
| 冷数据 | 对象存储 | 永久 | 1-5 秒 | ¥0.05/GB/月 |

**成本对比（1 年 365TB）：**

| 方案 | 年成本 |
|------|-------|
| 全量 MySQL | ¥182 万（365TB × ¥0.5/GB） |
| 冷热分层 | ¥11 万（30TB热 + 335TB冷） |

### 消息编辑与撤回：完整实现

#### 数据库扩展

```sql
-- 消息表增加编辑相关字段
ALTER TABLE messages ADD COLUMN edit_version INT NOT NULL DEFAULT 0;     -- 编辑版本号，0=原始
ALTER TABLE messages ADD COLUMN edited_at TIMESTAMP NULL;                -- 最后编辑时间
ALTER TABLE messages ADD COLUMN recalled_at TIMESTAMP NULL;              -- 撤回时间

-- 消息编辑历史表：保留每次编辑的记录（审计 + 撤销）
CREATE TABLE message_edit_history (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    msg_id BIGINT NOT NULL,
    edit_version INT NOT NULL,              -- 编辑版本号
    old_content TEXT,                        -- 编辑前内容
    new_content TEXT,                        -- 编辑后内容
    edited_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    editor_id VARCHAR(64) NOT NULL,         -- 编辑者（通常等于 sender_id）
    INDEX idx_msg (msg_id, edit_version)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### 消息编辑服务

```python
import time
import logging
from datetime import datetime, timedelta
from dataclasses import dataclass

logger = logging.getLogger(__name__)


class MessageEditService:
    """
    消息编辑服务：
    - 仅支持 2 分钟内编辑
    - 编辑不生成新 seq，在原 seq 上更新 content
    - 每次编辑递增 edit_version
    - 通知所有接收方消息已编辑
    - 保留编辑历史（审计 + 可追溯）
    """

    EDIT_TIMEOUT = timedelta(minutes=2)
    MAX_EDIT_COUNT = 5  # 单条消息最多编辑 5 次

    def __init__(self, db, redis, presence, push_service):
        self.db = db
        self.redis = redis
        self.presence = presence
        self.push_service = push_service

    def edit_message(self, user_id: str, conversation_id: str,
                     msg_seq: int, new_content: str) -> dict:
        """
        编辑消息完整流程：
        1. 校验权限和时效
        2. 保存编辑历史
        3. 更新消息内容
        4. 通知所有接收方
        """
        # ---- 第一步：查询原消息 ----
        msg = self.db.query_one("""
            SELECT msg_id, seq, sender_id, content, content_type,
                   edit_version, created_at, msg_status
            FROM messages
            WHERE conversation_id = %s AND seq = %s
        """, conversation_id, msg_seq)

        if not msg:
            return {"status": "not_found"}

        # ---- 第二步：校验 ----
        # 权限校验：只能编辑自己的消息
        if msg["sender_id"] != user_id:
            return {"status": "permission_denied", "error": "只能编辑自己发的消息"}

        # 时效校验：2 分钟内可编辑
        now = datetime.now()
        if now - msg["created_at"] > self.EDIT_TIMEOUT:
            return {"status": "timeout", "error": "超过 2 分钟无法编辑"}

        # 状态校验：已撤回的消息不能编辑
        if msg["msg_status"] == 1:
            return {"status": "recalled", "error": "已撤回的消息不能编辑"}

        # 次数校验
        if msg["edit_version"] >= self.MAX_EDIT_COUNT:
            return {"status": "max_edits", "error": f"最多编辑 {self.MAX_EDIT_COUNT} 次"}

        # 内容校验
        if not new_content or len(new_content.strip()) == 0:
            return {"status": "invalid", "error": "消息内容不能为空"}

        if len(new_content) > 4096:
            return {"status": "invalid", "error": "消息内容过长"}

        # ---- 第三步：保存编辑历史（先写历史再更新，保证审计完整） ----
        new_version = msg["edit_version"] + 1
        self.db.execute("""
            INSERT INTO message_edit_history
                (msg_id, edit_version, old_content, new_content, editor_id)
            VALUES (%s, %s, %s, %s, %s)
        """, msg["msg_id"], new_version, msg["content"], new_content, user_id)

        # ---- 第四步：更新消息内容 ----
        self.db.execute("""
            UPDATE messages
            SET content = %s,
                edit_version = %s,
                edited_at = %s,
                content_type = 1
            WHERE msg_id = %s
        """, new_content, new_version, now, msg["msg_id"])

        # ---- 第五步：清除相关缓存 ----
        self._clear_message_cache(msg["msg_id"], conversation_id)

        # ---- 第六步：通知所有接收方 ----
        edit_notification = {
            "type": "message_edited",
            "msg_id": msg["msg_id"],
            "seq": msg_seq,
            "conversation_id": conversation_id,
            "new_content": new_content,
            "edit_version": new_version,
            "edited_at": now.isoformat(),
            "editor_id": user_id
        }
        self._broadcast_edit_notification(conversation_id, user_id, edit_notification)

        logger.info(f"消息编辑成功: msg_id={msg['msg_id']}, "
                   f"edit_version={new_version}, editor={user_id}")

        return {
            "status": "ok",
            "msg_id": msg["msg_id"],
            "seq": msg_seq,
            "edit_version": new_version,
            "edited_at": now.isoformat()
        }

    def _broadcast_edit_notification(self, conversation_id: str,
                                      sender_id: str, notification: dict):
        """广播编辑通知给所有接收方（包括发送方的其他设备）"""
        if self._is_group(conversation_id):
            # 群聊：推送给所有群成员
            members = self.db.query("""
                SELECT user_id FROM group_members
                WHERE group_id = %s AND leave_seq IS NULL
            """, conversation_id)
            recipient_ids = [m["user_id"] for m in members]
        else:
            # 单聊：推送给对方 + 发送方（多设备同步）
            peer_id = self._get_peer_id(conversation_id, sender_id)
            recipient_ids = [peer_id, sender_id]

        # 批量推送
        online_status = self.presence.batch_check_online(recipient_ids)
        for uid, online in online_status.items():
            if online:
                self.push_service.push_online(uid, notification)
            # 离线用户上线后通过 seq 差量拉取即可看到编辑后的内容

    def _clear_message_cache(self, msg_id: int, conversation_id: str):
        """清除消息相关缓存"""
        # 清除搜索索引缓存
        self.redis.delete(f"msg_cache:{msg_id}")
        # 清除会话最新消息缓存
        self.redis.delete(f"conv_latest:{conversation_id}")

    def get_edit_history(self, msg_id: int) -> list:
        """获取消息的编辑历史"""
        return self.db.query("""
            SELECT edit_version, old_content, new_content, edited_at, editor_id
            FROM message_edit_history
            WHERE msg_id = %s
            ORDER BY edit_version ASC
        """, msg_id)


class MessageRecallService:
    """
    消息撤回服务：
    - 仅支持 2 分钟内撤回
    - 撤回不删除消息，只标记为已撤回（保留审计记录）
    - 通知所有接收方消息已撤回
    - 接收方客户端收到撤回通知后，将消息替换为"此消息已撤回"
    """

    RECALL_TIMEOUT = timedelta(minutes=2)

    def __init__(self, db, redis, presence, push_service):
        self.db = db
        self.redis = redis
        self.presence = presence
        self.push_service = push_service

    def recall_message(self, user_id: str, conversation_id: str,
                       msg_seq: int) -> dict:
        """
        撤回消息完整流程：
        1. 校验权限和时效
        2. 标记消息为已撤回（不删除）
        3. 通知所有接收方
        4. 清除搜索索引中的内容
        """
        # ---- 第一步：查询原消息 ----
        msg = self.db.query_one("""
            SELECT msg_id, seq, sender_id, content, content_type,
                   created_at, msg_status
            FROM messages
            WHERE conversation_id = %s AND seq = %s
        """, conversation_id, msg_seq)

        if not msg:
            return {"status": "not_found"}

        # ---- 第二步：校验 ----
        # 权限校验：只能撤回自己的消息（群主/管理员可撤回他人消息，此处简化）
        if msg["sender_id"] != user_id:
            # 检查是否为群主/管理员
            if not self._is_group_admin(conversation_id, user_id):
                return {"status": "permission_denied", "error": "只能撤回自己发的消息"}

        # 时效校验：2 分钟内可撤回
        now = datetime.now()
        if now - msg["created_at"] > self.RECALL_TIMEOUT:
            return {"status": "timeout", "error": "超过 2 分钟无法撤回"}

        # 状态校验：已撤回的消息不能重复撤回
        if msg["msg_status"] == 1:
            return {"status": "already_recalled", "error": "消息已撤回"}

        # ---- 第三步：标记消息为已撤回（保留审计记录，不删除） ----
        recalled_content = "此消息已撤回"
        self.db.execute("""
            UPDATE messages
            SET msg_status = 1,
                content_type = 5,
                content = %s,
                recalled_at = %s
            WHERE msg_id = %s
        """, recalled_content, now, msg["msg_id"])

        # ---- 第四步：清除相关缓存 ----
        self._clear_message_cache(msg["msg_id"], conversation_id)

        # ---- 第五步：通知所有接收方 ----
        recall_notification = {
            "type": "message_recalled",
            "msg_id": msg["msg_id"],
            "seq": msg_seq,
            "conversation_id": conversation_id,
            "recalled_by": user_id,
            "recalled_at": now.isoformat(),
            "original_sender": msg["sender_id"]
        }
        self._broadcast_recall_notification(
            conversation_id, user_id, recall_notification
        )

        # ---- 第六步：更新 ES 搜索索引（标记为已撤回） ----
        self._update_search_index(msg["msg_id"], conversation_id)

        logger.info(f"消息撤回成功: msg_id={msg['msg_id']}, "
                   f"recalled_by={user_id}")

        return {
            "status": "ok",
            "msg_id": msg["msg_id"],
            "seq": msg_seq,
            "recalled_at": now.isoformat()
        }

    def _broadcast_recall_notification(self, conversation_id: str,
                                        sender_id: str, notification: dict):
        """广播撤回通知给所有接收方"""
        if self._is_group(conversation_id):
            members = self.db.query("""
                SELECT user_id FROM group_members
                WHERE group_id = %s AND leave_seq IS NULL
            """, conversation_id)
            recipient_ids = [m["user_id"] for m in members]
        else:
            peer_id = self._get_peer_id(conversation_id, sender_id)
            recipient_ids = [peer_id, sender_id]

        # 撤回通知必须可靠送达（否则接收方仍能看到原消息内容）
        online_status = self.presence.batch_check_online(recipient_ids)
        for uid, online in online_status.items():
            if online:
                # 在线用户：WebSocket 推送撤回通知
                success = self.push_service.push_online(uid, notification)
                if not success:
                    # 推送失败 → 写入离线撤回通知队列
                    self._save_offline_recall(uid, notification)
            else:
                # 离线用户：写入离线撤回通知队列
                self._save_offline_recall(uid, notification)

    def _save_offline_recall(self, user_id: str, notification: dict):
        """
        保存离线撤回通知。
        关键：撤回通知必须比原消息优先处理！
        用户上线后，先处理撤回通知，再拉取未读消息。
        """
        key = f"offline_recall:{user_id}"
        # 使用 Sorted Set，score = msg_seq，保证按序处理
        self.redis.zadd(key, {
            json.dumps(notification): notification["seq"]
        })
        # 撤回通知 30 天过期
        self.redis.expire(key, 30 * 86400)

    def _update_search_index(self, msg_id: int, conversation_id: str):
        """更新 ES 搜索索引，标记消息为已撤回"""
        # 通过 Kafka 发送撤回事件，Flink 消费后更新 ES
        self.kafka_produce("message_recall_events", {
            "msg_id": msg_id,
            "conversation_id": conversation_id,
            "is_recalled": True,
            "content": "此消息已撤回"
        })

    def _clear_message_cache(self, msg_id: int, conversation_id: str):
        """清除消息相关缓存"""
        self.redis.delete(f"msg_cache:{msg_id}")
        self.redis.delete(f"conv_latest:{conversation_id}")

    def _is_group_admin(self, conversation_id: str, user_id: str) -> bool:
        """检查用户是否为群管理员"""
        member = self.db.query_one("""
            SELECT role FROM group_members
            WHERE group_id = %s AND user_id = %s AND leave_seq IS NULL
        """, conversation_id, user_id)
        return member is not None and member["role"] >= 1
```

#### 客户端处理编辑和撤回通知

```python
class ClientMessageHandler:
    """
    客户端消息处理器：处理编辑和撤回通知
    """

    def on_message_edited(self, notification: dict):
        """收到消息编辑通知"""
        msg_id = notification["msg_id"]
        new_content = notification["new_content"]
        edit_version = notification["edit_version"]

        # 更新本地消息
        local_msg = self.local_db.get_message(msg_id)
        if local_msg:
            # 保留编辑历史（可选，用于"查看编辑记录"功能）
            if local_msg.edit_version < edit_version:
                self.local_db.save_edit_history(
                    msg_id, local_msg.content, new_content, edit_version
                )
                self.local_db.update_message(msg_id, {
                    "content": new_content,
                    "edit_version": edit_version,
                    "edited_at": notification["edited_at"]
                })
                # 刷新 UI
                self.refresh_message_ui(msg_id)

    def on_message_recalled(self, notification: dict):
        """收到消息撤回通知"""
        msg_id = notification["msg_id"]

        # 将本地消息替换为"此消息已撤回"
        local_msg = self.local_db.get_message(msg_id)
        if local_msg and local_msg.msg_status != 1:
            self.local_db.update_message(msg_id, {
                "content": "此消息已撤回",
                "content_type": 5,  # 已撤回
                "msg_status": 1,
                "recalled_at": notification["recalled_at"]
            })
            # 刷新 UI
            self.refresh_message_ui(msg_id)

    def on_offline_recalls(self, recalls: list):
        """
        上线后处理离线撤回通知（优先于普通消息拉取）
        """
        for recall in sorted(recalls, key=lambda r: r["seq"]):
            self.on_message_recalled(recall)
```

### 离线推送策略

```python
class OfflinePushService:
    def push_offline(self, user_id, msg):
        """离线用户推送"""
        # 1. 判断推送优先级
        if msg.content_type == "text" and len(msg.content) < 50:
            # 短文本 → 直接推送内容
            self.push_service.send(user_id, {
                "title": msg.sender_name,
                "body": msg.content,
                "type": "message"
            })
        elif msg.content_type == "image":
            # 图片 → 推送"发送了一张图片"
            self.push_service.send(user_id, {
                "title": msg.sender_name,
                "body": "发送了一张图片",
                "type": "message"
            })
        else:
            # 其他 → 推送"发送了一条消息"
            self.push_service.send(user_id, {
                "title": msg.sender_name,
                "body": "发送了一条消息",
                "type": "message"
            })

        # 2. 推送频率控制（避免骚扰）
        # 每分钟最多 3 条推送通知
        key = f"push_limit:{user_id}"
        count = self.redis.incr(key)
        if count == 1:
            self.redis.expire(key, 60)
        if count > 3:
            # 超限 → 合并为一条"有N条新消息"
            self.redis.set(f"push_merged:{user_id}", count)
            return  # 不单独推送

    def on_user_online(self, user_id):
        """用户上线 → 取消合并推送，推送未读消息数"""
        merged = self.redis.get(f"push_merged:{user_id}")
        if merged:
            self.push_service.cancel(user_id)  # 取消合并通知
            self.push_to_user(user_id, f"你有 {merged} 条未读消息")
```

**推送渠道选择：**

| 渠道 | 延迟 | 到达率 | 适用 |
|------|------|-------|------|
| APNs（iOS） | 1-3 秒 | 95% | 苹果设备 |
| FCM（Android国际） | 1-3 秒 | 80% | 海外安卓 |
| 厂商推送（华为/小米/OPPO） | < 1 秒 | 90% | 国内安卓 |
| SMS | 10-30 秒 | 99% | 推送失败兜底 |

### 合规审计：完整实现

金融、政务等受监管行业要求：消息不可篡改、可追溯、可审计。以下实现涵盖内容监控、保留策略、法律封存三大核心能力。

#### 审计数据库设计

```sql
-- ============================================================
-- 1. 操作审计日志表：记录所有敏感操作（WORM 存储，不可篡改）
-- ============================================================
CREATE TABLE audit_log (
    audit_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    operation_type VARCHAR(32) NOT NULL,     -- send/edit/recall/delete/forward/export
    operator_id VARCHAR(64) NOT NULL,         -- 操作者
    target_type VARCHAR(32) NOT NULL,         -- message/conversation/group/member
    target_id VARCHAR(128) NOT NULL,          -- 操作对象 ID
    conversation_id VARCHAR(64),              -- 所属会话
    before_snapshot TEXT,                      -- 操作前快照（JSON）
    after_snapshot TEXT,                       -- 操作后快照（JSON）
    client_ip VARCHAR(64),                    -- 客户端 IP
    device_id VARCHAR(64),                    -- 设备 ID
    user_agent VARCHAR(256),                  -- 客户端 UA
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_operator (operator_id, created_at),
    INDEX idx_target (target_type, target_id),
    INDEX idx_conv (conversation_id, created_at),
    INDEX idx_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ============================================================
-- 2. 消息内容监控规则表
-- ============================================================
CREATE TABLE compliance_rules (
    rule_id INT PRIMARY KEY AUTO_INCREMENT,
    rule_name VARCHAR(128) NOT NULL,
    rule_type TINYINT NOT NULL,              -- 1=关键词, 2=正则, 3=敏感图片, 4=文件类型
    rule_pattern TEXT NOT NULL,               -- 匹配模式（关键词列表/正则表达式/文件类型）
    severity TINYINT NOT NULL DEFAULT 1,     -- 1=警告, 2=拦截, 3=告警+拦截
    scope_type TINYINT NOT NULL DEFAULT 0,   -- 0=全局, 1=指定部门, 2=指定用户
    scope_value TEXT,                         -- 适用范围（部门ID列表/用户ID列表）
    action TINYINT NOT NULL DEFAULT 0,       -- 0=仅记录, 1=拦截, 2=拦截+通知管理员
    is_active TINYINT NOT NULL DEFAULT 1,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ============================================================
-- 3. 合规告警表
-- ============================================================
CREATE TABLE compliance_alerts (
    alert_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    rule_id INT NOT NULL,
    msg_id BIGINT NOT NULL,
    conversation_id VARCHAR(64) NOT NULL,
    sender_id VARCHAR(64) NOT NULL,
    alert_type TINYINT NOT NULL,             -- 1=关键词命中, 2=正则命中, 3=敏感图片, 4=违规文件
    matched_content TEXT,                     -- 命中的内容片段
    severity TINYINT NOT NULL,
    status TINYINT NOT NULL DEFAULT 0,       -- 0=待处理, 1=已确认, 2=已忽略, 3=已处理
    handler_id VARCHAR(64),                  -- 处理人
    handle_note TEXT,                         -- 处理备注
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    handled_at TIMESTAMP NULL,
    INDEX idx_status (status, created_at),
    INDEX idx_sender (sender_id, created_at),
    INDEX idx_rule (rule_id, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ============================================================
-- 4. 数据保留策略表
-- ============================================================
CREATE TABLE retention_policies (
    policy_id INT PRIMARY KEY AUTO_INCREMENT,
    policy_name VARCHAR(128) NOT NULL,
    scope_type TINYINT NOT NULL,             -- 1=全局, 2=部门, 3=会话类型
    scope_value VARCHAR(256),                -- 适用范围
    retention_days INT NOT NULL,             -- 保留天数（-1 表示永久保留）
    action_after_expiry TINYINT NOT NULL,    -- 1=删除, 2=归档到冷存储, 3=匿名化
    is_default TINYINT NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- ============================================================
-- 5. 法律封存表（Legal Hold）
-- ============================================================
CREATE TABLE legal_holds (
    hold_id INT PRIMARY KEY AUTO_INCREMENT,
    case_id VARCHAR(128) NOT NULL,           -- 案件编号
    hold_reason TEXT NOT NULL,                -- 封存原因
    scope_type TINYINT NOT NULL,             -- 1=指定用户, 2=指定群, 3=指定时间范围, 4=全局
    scope_value TEXT NOT NULL,                -- 封存范围（用户ID列表/群ID列表/时间范围）
    start_time TIMESTAMP NOT NULL,           -- 封存数据起始时间
    end_time TIMESTAMP,                      -- 封存数据结束时间（NULL=无限期）
    issued_by VARCHAR(64) NOT NULL,          -- 发起人
    status TINYINT NOT NULL DEFAULT 1,       -- 1=生效, 0=已解除
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    released_at TIMESTAMP NULL,
    released_by VARCHAR(64),
    INDEX idx_case (case_id),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 法律封存标记表：标记哪些消息/会话被法律封存
CREATE TABLE legal_hold_items (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    hold_id INT NOT NULL,
    item_type TINYINT NOT NULL,              -- 1=消息, 2=会话, 3=用户所有消息
    item_id VARCHAR(128) NOT NULL,           -- msg_id / conversation_id / user_id
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_hold_item (hold_id, item_type, item_id),
    INDEX idx_item (item_type, item_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### 消息内容监控服务

```python
import re
import json
import logging
from datetime import datetime
from concurrent.futures import ThreadPoolExecutor

logger = logging.getLogger(__name__)


class ComplianceMonitorService:
    """
    合规监控服务：
    - 消息发送前实时检测敏感内容
    - 命中规则后根据严重程度执行不同动作
    - 异步写入合规告警表
    - 支持关键词、正则、敏感图片检测
    """

    def __init__(self, db, redis, kafka_producer):
        self.db = db
        self.redis = redis
        self.kafka_producer = kafka_producer
        self.executor = ThreadPoolExecutor(max_workers=10)
        self._rule_cache = None
        self._rule_cache_time = 0
        self.RULE_CACHE_TTL = 300  # 规则缓存 5 分钟

    def check_message_before_send(self, msg) -> dict:
        """
        消息发送前的合规检查（同步调用，在消息投递链路中）
        返回：
          {"allowed": True}  → 允许发送
          {"allowed": False, "reason": "..."} → 拦截
          {"allowed": True, "alert": {...}} → 允许但告警
        """
        # 加载活跃规则（带缓存）
        rules = self._get_active_rules()

        # 检查消息发送者适用的规则
        applicable_rules = self._filter_rules_by_scope(rules, msg.sender_id)

        alerts = []
        for rule in applicable_rules:
            matched = self._match_rule(rule, msg)
            if matched:
                alert = {
                    "rule_id": rule["rule_id"],
                    "rule_name": rule["rule_name"],
                    "msg_id": msg.msg_id,
                    "sender_id": msg.sender_id,
                    "conversation_id": msg.conversation_id,
                    "matched_content": matched,
                    "severity": rule["severity"],
                    "created_at": datetime.now().isoformat()
                }
                alerts.append(alert)

                # 根据动作决定是否拦截
                if rule["action"] in (1, 2):
                    # 拦截消息
                    self._async_save_alert(alert)
                    if rule["action"] == 2:
                        # 通知管理员
                        self._notify_admin(alert)
                    return {
                        "allowed": False,
                        "reason": f"消息违反合规规则: {rule['rule_name']}",
                        "alert_id": alert
                    }

        # 所有规则检查通过，保存告警（如果有）
        for alert in alerts:
            self._async_save_alert(alert)

        if alerts:
            return {"allowed": True, "alert": alerts[0]}

        return {"allowed": True}

    def _match_rule(self, rule: dict, msg) -> str:
        """
        根据规则类型匹配消息内容
        返回命中的内容片段，未命中返回 None
        """
        if rule["rule_type"] == 1:
            # 关键词匹配
            keywords = json.loads(rule["rule_pattern"])
            content_lower = msg.content.lower()
            for keyword in keywords:
                if keyword.lower() in content_lower:
                    return keyword
            return None

        elif rule["rule_type"] == 2:
            # 正则匹配
            try:
                match = re.search(rule["rule_pattern"], msg.content)
                if match:
                    return match.group(0)
            except re.error:
                logger.error(f"正则规则无效: rule_id={rule['rule_id']}")
            return None

        elif rule["rule_type"] == 3:
            # 敏感图片检测（调用外部 AI 审核服务）
            if msg.content_type == 2:  # 图片消息
                image_url = msg.content
                result = self._check_image_content(image_url)
                if result.get("sensitive"):
                    return result.get("label", "sensitive_image")
            return None

        elif rule["rule_type"] == 4:
            # 文件类型检测
            if msg.content_type == 3:  # 文件消息
                file_types = json.loads(rule["rule_pattern"])
                file_ext = self._get_file_extension(msg.content)
                if file_ext in file_types:
                    return file_ext
            return None

        return None

    def _get_active_rules(self) -> list:
        """获取活跃的合规规则（带缓存）"""
        now = time.time()
        if self._rule_cache and (now - self._rule_cache_time) < self.RULE_CACHE_TTL:
            return self._rule_cache

        rules = self.db.query("""
            SELECT * FROM compliance_rules WHERE is_active = 1
        """)
        self._rule_cache = rules
        self._rule_cache_time = now
        return rules

    def _filter_rules_by_scope(self, rules: list, sender_id: str) -> list:
        """过滤适用于发送者的规则"""
        applicable = []
        for rule in rules:
            if rule["scope_type"] == 0:
                # 全局规则
                applicable.append(rule)
            elif rule["scope_type"] == 1:
                # 部门规则：检查发送者是否属于该部门
                departments = json.loads(rule["scope_value"])
                user_dept = self._get_user_department(sender_id)
                if user_dept in departments:
                    applicable.append(rule)
            elif rule["scope_type"] == 2:
                # 指定用户规则
                users = json.loads(rule["scope_value"])
                if sender_id in users:
                    applicable.append(rule)
        return applicable

    def _async_save_alert(self, alert: dict):
        """异步保存合规告警"""
        self.executor.submit(self._save_alert, alert)

    def _save_alert(self, alert: dict):
        """保存合规告警到数据库"""
        try:
            self.db.execute("""
                INSERT INTO compliance_alerts
                    (rule_id, msg_id, conversation_id, sender_id,
                     alert_type, matched_content, severity, status)
                VALUES (%s, %s, %s, %s, %s, %s, %s, 0)
            """, alert["rule_id"], alert["msg_id"], alert["conversation_id"],
                alert["sender_id"], 1, alert["matched_content"],
                alert["severity"])
        except Exception as e:
            logger.error(f"保存合规告警失败: {e}")

    def _notify_admin(self, alert: dict):
        """通知合规管理员"""
        # 通过 Kafka 发送告警事件，由通知服务消费
        self.kafka_producer.produce("compliance_alert_events", {
            "type": "compliance_alert",
            "alert": alert,
            "timestamp": datetime.now().isoformat()
        })


class RetentionPolicyService:
    """
    数据保留策略服务：
    - 根据策略自动清理或归档过期消息
    - 法律封存的数据不受保留策略影响
    - 支持按部门/会话类型设置不同保留期
    """

    def __init__(self, db, redis, s3_client):
        self.db = db
        self.redis = redis
        self.s3_client = s3_client

    def enforce_retention(self):
        """
        定时执行保留策略（每天凌晨运行）
        1. 加载所有保留策略
        2. 查找过期数据
        3. 检查是否被法律封存
        4. 执行清理/归档/匿名化
        """
        policies = self.db.query("""
            SELECT * FROM retention_policies
        """)

        for policy in policies:
            cutoff_date = datetime.now() - timedelta(days=policy["retention_days"])

            if policy["scope_type"] == 1:
                # 全局策略
                self._process_expired_data(
                    cutoff_date, policy["action_after_expiry"]
                )
            elif policy["scope_type"] == 2:
                # 部门策略
                departments = json.loads(policy["scope_value"])
                for dept_id in departments:
                    self._process_expired_data_by_dept(
                        dept_id, cutoff_date, policy["action_after_expiry"]
                    )

    def _process_expired_data(self, cutoff_date: datetime, action: int):
        """处理过期数据"""
        # 查找过期消息（按月份表逐表扫描）
        expired_tables = self._get_tables_before(cutoff_date)
        for table in expired_tables:
            # 分批处理，避免大事务
            offset = 0
            batch_size = 1000
            while True:
                messages = self.db.query(f"""
                    SELECT msg_id, conversation_id, sender_id, content
                    FROM {table}
                    WHERE created_at < %s
                    LIMIT %s OFFSET %s
                """, cutoff_date, batch_size, offset)

                if not messages:
                    break

                for msg in messages:
                    # 检查是否被法律封存
                    if self._is_under_legal_hold(msg["msg_id"], msg["conversation_id"]):
                        continue  # 跳过封存数据

                    if action == 1:
                        # 删除
                        self.db.execute(f"""
                            DELETE FROM {table} WHERE msg_id = %s
                        """, msg["msg_id"])
                    elif action == 2:
                        # 归档到冷存储
                        self._archive_to_cold_storage(msg)
                        self.db.execute(f"""
                            DELETE FROM {table} WHERE msg_id = %s
                        """, msg["msg_id"])
                    elif action == 3:
                        # 匿名化：保留元数据，清除内容
                        self.db.execute(f"""
                            UPDATE {table}
                            SET content = '[已根据保留策略匿名化]',
                                content_type = 6
                            WHERE msg_id = %s
                        """, msg["msg_id"])

                offset += batch_size

    def _is_under_legal_hold(self, msg_id: int, conversation_id: str) -> bool:
        """检查消息是否被法律封存"""
        # 检查消息级别封存
        msg_hold = self.db.query_one("""
            SELECT 1 FROM legal_hold_items
            WHERE item_type = 1 AND item_id = %s
            LIMIT 1
        """, str(msg_id))
        if msg_hold:
            return True

        # 检查会话级别封存
        conv_hold = self.db.query_one("""
            SELECT 1 FROM legal_hold_items lhi
            JOIN legal_holds lh ON lhi.hold_id = lh.hold_id
            WHERE lhi.item_type = 2 AND lhi.item_id = %s AND lh.status = 1
            LIMIT 1
        """, conversation_id)
        if conv_hold:
            return True

        return False

    def _archive_to_cold_storage(self, msg: dict):
        """归档消息到 S3 冷存储（Parquet 格式）"""
        archive_key = f"messages/{msg['conversation_id']}/{msg['msg_id']}.json"
        self.s3_client.put_object(
            bucket="im-archive",
            key=archive_key,
            body=json.dumps(msg, ensure_ascii=False)
        )


class LegalHoldService:
    """
    法律封存服务：
    - 接到法律要求后，封存指定用户/群/时间范围的所有消息
    - 封存数据不可被保留策略清理
    - 提供封存数据导出能力（用于法律取证）
    - 封存解除后恢复正常的保留策略
    """

    def __init__(self, db, redis, s3_client):
        self.db = db
        self.redis = redis
        self.s3_client = s3_client

    def create_legal_hold(self, case_id: str, reason: str,
                          scope_type: int, scope_value: str,
                          start_time: datetime, end_time: datetime = None,
                          issued_by: str = "") -> dict:
        """
        创建法律封存：
        1. 写入封存记录
        2. 标记所有涉及的消息/会话
        3. 缓存封存状态（加速查询）
        """
        # 写入封存主记录
        self.db.execute("""
            INSERT INTO legal_holds
                (case_id, hold_reason, scope_type, scope_value,
                 start_time, end_time, issued_by, status)
            VALUES (%s, %s, %s, %s, %s, %s, %s, 1)
        """, case_id, reason, scope_type, scope_value,
             start_time, end_time, issued_by)

        hold_id = self.db.last_insert_id()

        # 根据范围标记涉及的消息/会话
        if scope_type == 1:
            # 指定用户
            user_ids = json.loads(scope_value)
            for uid in user_ids:
                self.db.execute("""
                    INSERT IGNORE INTO legal_hold_items
                        (hold_id, item_type, item_id)
                    VALUES (%s, 3, %s)
                """, hold_id, uid)
                # 缓存标记
                self.redis.sadd(f"legal_hold:users", uid)

        elif scope_type == 2:
            # 指定群
            group_ids = json.loads(scope_value)
            for gid in group_ids:
                self.db.execute("""
                    INSERT IGNORE INTO legal_hold_items
                        (hold_id, item_type, item_id)
                    VALUES (%s, 2, %s)
                """, hold_id, gid)
                self.redis.sadd(f"legal_hold:conversations", gid)

        logger.info(f"法律封存创建: case_id={case_id}, hold_id={hold_id}, "
                   f"scope_type={scope_type}")

        return {"hold_id": hold_id, "status": "active"}

    def release_legal_hold(self, hold_id: int, released_by: str) -> dict:
        """解除法律封存"""
        self.db.execute("""
            UPDATE legal_holds
            SET status = 0, released_at = %s, released_by = %s
            WHERE hold_id = %s
        """, datetime.now(), released_by, hold_id)

        # 重建缓存（清除已解除的封存标记）
        self._rebuild_legal_hold_cache()

        return {"hold_id": hold_id, "status": "released"}

    def export_legal_hold_data(self, hold_id: int,
                                export_format: str = "json") -> str:
        """
        导出法律封存数据（用于法律取证）
        返回导出文件的 S3 路径
        """
        hold = self.db.query_one("""
            SELECT * FROM legal_holds WHERE hold_id = %s
        """, hold_id)

        items = self.db.query("""
            SELECT * FROM legal_hold_items WHERE hold_id = %s
        """, hold_id)

        export_data = {
            "case_id": hold["case_id"],
            "hold_reason": hold["hold_reason"],
            "issued_by": hold["issued_by"],
            "created_at": hold["created_at"].isoformat(),
            "messages": []
        }

        for item in items:
            if item["item_type"] == 3:
                # 用户所有消息
                messages = self.db.query("""
                    SELECT * FROM messages
                    WHERE sender_id = %s
                    AND created_at BETWEEN %s AND %s
                    ORDER BY created_at ASC
                """, item["item_id"], hold["start_time"],
                    hold["end_time"] or "2099-12-31")
                export_data["messages"].extend(messages)

            elif item["item_type"] == 2:
                # 群消息
                messages = self.db.query("""
                    SELECT * FROM messages
                    WHERE conversation_id = %s
                    AND created_at BETWEEN %s AND %s
                    ORDER BY created_at ASC
                """, item["item_id"], hold["start_time"],
                    hold["end_time"] or "2099-12-31")
                export_data["messages"].extend(messages)

        # 写入 S3
        export_key = f"legal_exports/{hold['case_id']}/{hold_id}.{export_format}"
        self.s3_client.put_object(
            bucket="im-legal-exports",
            key=export_key,
            body=json.dumps(export_data, ensure_ascii=False, default=str)
        )

        return f"s3://im-legal-exports/{export_key}"

    def _rebuild_legal_hold_cache(self):
        """重建法律封存缓存"""
        # 清除旧缓存
        self.redis.delete("legal_hold:users")
        self.redis.delete("legal_hold:conversations")

        # 从 DB 加载生效中的封存
        active_holds = self.db.query("""
            SELECT lhi.item_type, lhi.item_id
            FROM legal_hold_items lhi
            JOIN legal_holds lh ON lhi.hold_id = lh.hold_id
            WHERE lh.status = 1
        """)

        for item in active_holds:
            if item["item_type"] == 3:
                self.redis.sadd("legal_hold:users", item["item_id"])
            elif item["item_type"] == 2:
                self.redis.sadd("legal_hold:conversations", item["item_id"])
```

#### 审计日志记录切面

```python
class AuditLogInterceptor:
    """
    审计日志拦截器：
    在消息操作（发送/编辑/撤回/删除/转发/导出）前后自动记录审计日志。
    采用 AOP 切面模式，业务代码无侵入。
    """

    AUDITED_OPERATIONS = {
        "send_message", "edit_message", "recall_message",
        "delete_message", "forward_message", "export_message"
    }

    def __init__(self, db, redis):
        self.db = db
        self.redis = redis
        self.async_queue = []  # 异步写入队列

    def before_operation(self, operation: str, operator_id: str,
                         target_type: str, target_id: str,
                         current_snapshot: dict = None,
                         context: dict = None):
        """操作前记录"""
        if operation not in self.AUDITED_OPERATIONS:
            return

        # 保存操作前快照（用于 after 时对比）
        key = f"audit_before:{operator_id}:{operation}:{target_id}"
        if current_snapshot:
            self.redis.setex(key, 300, json.dumps(current_snapshot))

    def after_operation(self, operation: str, operator_id: str,
                        target_type: str, target_id: str,
                        conversation_id: str = None,
                        new_snapshot: dict = None,
                        context: dict = None):
        """操作后记录审计日志"""
        if operation not in self.AUDITED_OPERATIONS:
            return

        # 获取操作前快照
        key = f"audit_before:{operator_id}:{operation}:{target_id}"
        before_snapshot = self.redis.get(key)
        self.redis.delete(key)

        # 写入审计日志
        self.db.execute("""
            INSERT INTO audit_log
                (operation_type, operator_id, target_type, target_id,
                 conversation_id, before_snapshot, after_snapshot,
                 client_ip, device_id, user_agent)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
        """, operation, operator_id, target_type, target_id,
             conversation_id,
             before_snapshot,
             json.dumps(new_snapshot) if new_snapshot else None,
             context.get("client_ip") if context else None,
             context.get("device_id") if context else None,
             context.get("user_agent") if context else None)

        # 审计日志写入 WORM 存储（追加写入对象存储，不可篡改）
        self._write_to_worm_storage(operation, operator_id, target_id,
                                     before_snapshot, new_snapshot)
```

### 多设备同步：完整实现

同一账号在手机 + PC + Web 同时在线时，需要保证消息同步和已读状态一致。

#### 多设备连接管理

```sql
-- 扩展 user_connections 表，增加同步相关字段
ALTER TABLE user_connections ADD COLUMN device_max_seq BIGINT NOT NULL DEFAULT 0;  -- 该设备已同步到的最大 seq
ALTER TABLE user_connections ADD COLUMN last_sync_at TIMESTAMP NOT NULL;           -- 最后同步时间
```

```python
import json
import time
import logging
from datetime import datetime
from collections import defaultdict

logger = logging.getLogger(__name__)


class MultiDeviceSyncService:
    """
    多设备同步服务：
    - 管理同一用户多个设备的消息同步
    - 已读回执跨设备统一
    - 消息发送方在其他设备同步显示
    - 设备间实时同步已读状态变更
    """

    def __init__(self, redis, db, presence, push_service):
        self.redis = redis
        self.db = db
        self.presence = presence
        self.push_service = push_service

    def on_user_connect(self, user_id: str, device_id: str,
                        gateway_id: str, client_type: int):
        """
        设备上线处理：
        1. 注册设备连接
        2. 推送该设备缺失的增量消息
        3. 同步已读状态
        """
        # 注册设备连接
        conn_key = f"device_conn:{user_id}"
        self.redis.hset(conn_key, device_id, json.dumps({
            "gateway_id": gateway_id,
            "client_type": client_type,
            "connected_at": time.time()
        }))

        # 获取该设备上次同步到的位置
        device_seq = self._get_device_max_seq(user_id, device_id)
        device_read_seqs = self._get_device_read_seqs(user_id, device_id)

        # 推送增量消息
        self._sync_incremental_messages(user_id, device_id, device_seq)

        # 同步已读状态（取所有设备中最新的已读位置）
        self._sync_read_progress(user_id, device_id, device_read_seqs)

    def on_new_message_for_user(self, user_id: str, msg):
        """
        收到新消息时推送到用户的所有在线设备
        - 发送方的消息也要推送到发送方的其他设备（多端可见）
        - 接收方的消息推送到所有设备
        """
        devices = self._get_online_devices(user_id)

        for device_id, conn_info in devices.items():
            gateway_id = conn_info["gateway_id"]
            # 推送消息到该设备
            success = self.push_service.push_to_device(
                user_id, device_id, gateway_id, msg
            )
            if success:
                # 更新该设备的 max_seq
                self._update_device_max_seq(user_id, device_id, msg.seq)

    def on_read_progress_update(self, user_id: str, device_id: str,
                                 conversation_id: str, read_seq: int):
        """
        某设备上报已读进度时的处理：
        1. 更新该设备的已读进度
        2. 取所有设备中最大 read_seq 作为全局已读进度
        3. 通知其他设备同步已读状态
        """
        # 更新该设备的已读进度
        device_key = f"device_read:{user_id}:{device_id}"
        self.redis.hset(device_key, conversation_id, str(read_seq))

        # 计算全局已读进度（取所有设备最大值）
        global_read_seq = self._compute_global_read_seq(
            user_id, conversation_id
        )

        # 更新全局已读进度
        self._update_global_read_progress(user_id, conversation_id, global_read_seq)

        # 通知其他设备同步已读状态
        other_devices = self._get_online_devices(user_id)
        notification = {
            "type": "read_progress_sync",
            "conversation_id": conversation_id,
            "read_seq": global_read_seq,
            "updated_by_device": device_id
        }
        for did, conn_info in other_devices.items():
            if did != device_id:
                self.push_service.push_to_device(
                    user_id, did, conn_info["gateway_id"], notification
                )

    def _compute_global_read_seq(self, user_id: str,
                                  conversation_id: str) -> int:
        """
        计算全局已读进度：取所有设备中最大的 read_seq
        逻辑：用户在手机上读了 seq=100，在电脑上只读了 seq=80
        → 全局已读进度 = max(100, 80) = 100
        → 电脑收到通知后，将本地 read_seq 更新为 100
        """
        devices = self._get_all_devices(user_id)
        max_read_seq = 0

        for device_id in devices:
            device_key = f"device_read:{user_id}:{device_id}"
            device_seq = self.redis.hget(device_key, conversation_id)
            if device_seq:
                max_read_seq = max(max_read_seq, int(device_seq))

        # 也考虑 DB 中存储的全局已读进度
        db_progress = self.db.query_one("""
            SELECT read_seq FROM read_progress
            WHERE user_id = %s AND conversation_id = %s
        """, user_id, conversation_id)
        if db_progress:
            max_read_seq = max(max_read_seq, db_progress["read_seq"])

        return max_read_seq

    def _sync_incremental_messages(self, user_id: str, device_id: str,
                                    device_max_seq: int):
        """
        增量消息同步：推送该设备缺失的消息
        1. 获取用户所有会话的最新 seq
        2. 对比设备本地 seq，推送差量
        """
        # 获取用户所有会话
        conversations = self._get_user_conversations(user_id)

        for conv_id, conv_max_seq in conversations.items():
            if device_max_seq >= conv_max_seq:
                continue  # 该设备已是最新

            # 拉取差量消息
            messages = self.db.query("""
                SELECT * FROM messages
                WHERE conversation_id = %s AND seq > %s
                ORDER BY seq ASC
                LIMIT 200
            """, conv_id, device_max_seq)

            if messages:
                conn_info = self._get_device_connection(user_id, device_id)
                if conn_info:
                    self.push_service.push_to_device(
                        user_id, device_id, conn_info["gateway_id"],
                        {"type": "incremental_sync", "messages": messages}
                    )

    def _sync_read_progress(self, user_id: str, device_id: str,
                             device_read_seqs: dict):
        """
        同步已读状态：
        将全局最新已读进度推送到刚上线的设备
        """
        conversations = self._get_user_conversations(user_id)

        sync_data = {}
        for conv_id in conversations:
            global_read_seq = self._compute_global_read_seq(user_id, conv_id)
            device_seq = device_read_seqs.get(conv_id, 0)
            if global_read_seq > device_seq:
                sync_data[conv_id] = global_read_seq

        if sync_data:
            conn_info = self._get_device_connection(user_id, device_id)
            if conn_info:
                self.push_service.push_to_device(
                    user_id, device_id, conn_info["gateway_id"],
                    {"type": "read_progress_init", "progress": sync_data}
                )

    def _get_online_devices(self, user_id: str) -> dict:
        """获取用户所有在线设备"""
        conn_key = f"device_conn:{user_id}"
        devices = self.redis.hgetall(conn_key)
        return {did: json.loads(info) for did, info in devices.items()}

    def _get_all_devices(self, user_id: str) -> list:
        """获取用户所有设备（含离线）"""
        # 从 DB 查询（包括历史连接过的设备）
        result = self.db.query("""
            SELECT DISTINCT device_id FROM user_connections
            WHERE user_id = %s
        """, user_id)
        return [r["device_id"] for r in result]

    def _get_device_max_seq(self, user_id: str, device_id: str) -> int:
        """获取设备的同步进度"""
        key = f"device_seq:{user_id}:{device_id}"
        seq = self.redis.get(key)
        return int(seq) if seq else 0

    def _update_device_max_seq(self, user_id: str, device_id: str, seq: int):
        """更新设备的同步进度"""
        key = f"device_seq:{user_id}:{device_id}"
        current = self.redis.get(key)
        if current is None or int(current) < seq:
            self.redis.set(key, str(seq))

    def _get_device_read_seqs(self, user_id: str, device_id: str) -> dict:
        """获取设备的已读进度"""
        key = f"device_read:{user_id}:{device_id}"
        data = self.redis.hgetall(key)
        return {conv_id: int(seq) for conv_id, seq in data.items()}

    def _get_device_connection(self, user_id: str, device_id: str) -> dict:
        """获取设备连接信息"""
        conn_key = f"device_conn:{user_id}"
        info = self.redis.hget(conn_key, device_id)
        return json.loads(info) if info else None

    def _get_user_conversations(self, user_id: str) -> dict:
        """获取用户所有会话及其最新 seq"""
        # 单聊会话
        convs = self.db.query("""
            SELECT conversation_id, max_seq
            FROM user_conversations
            WHERE user_id = %s
        """, user_id)

        # 补充 Redis 缓存中的最新 seq
        result = {}
        for conv in convs:
            conv_id = conv["conversation_id"]
            cached_max = self.redis.get(f"conv_max_seq:{conv_id}")
            result[conv_id] = int(cached_max) if cached_max else conv["max_seq"]

        return result

    def _update_global_read_progress(self, user_id: str,
                                      conversation_id: str, read_seq: int):
        """更新全局已读进度（DB + Redis 缓存）"""
        self.db.execute("""
            INSERT INTO read_progress (user_id, conversation_id, read_seq)
            VALUES (%s, %s, %s)
            ON DUPLICATE KEY UPDATE
                read_seq = GREATEST(read_seq, VALUES(read_seq)),
                updated_at = CURRENT_TIMESTAMP
        """, user_id, conversation_id, read_seq)

        # 更新 Redis 缓存
        buf_key = f"read_buf:{user_id}:{conversation_id}"
        self.redis.set(buf_key, str(read_seq))
```

```python
class WebSocketGateway:
    """WebSocket 接入网关"""

    def on_connect(self, user_id, connection):
        # 1. 记录用户连接信息
        self.redis.hset(f"ws_conn:{user_id}", {
            "gateway_id": self.gateway_id,
            "connected_at": now().isoformat()
        })

        # 2. 设置在线状态
        self.set_online(user_id)

        # 3. 推送离线期间的消息
        self.sync_offline_messages(user_id)

    def on_disconnect(self, user_id):
        # 1. 清除连接信息
        self.redis.hdel(f"ws_conn:{user_id}")

        # 2. 设置离线状态（延迟 30 秒，避免短暂断连）
        self.schedule_set_offline(user_id, delay=30)

    def on_message_from_client(self, user_id, data):
        """客户端发送消息"""
        msg = self.parse_message(data)
        
        # 转发给消息服务处理
        self.message_service.send_message(msg)
```

**网关集群设计：**

- 100 台网关服务器
- 每台维持 30 万 WebSocket 连接（3000 万 DAU / 100 = 30 万）
- 每台 16GB 内存（每个连接约 50KB）
- 网关无状态 → 用户可以连任意网关

### 消息不丢不重的完整流程

```
发送方 → 网关 → 消息服务 → 存储 → 推送 → 接收方

1. 发送方发送 → 网关转发 → 消息服务
2. 消息服务：生成 msg_id + seq → 存储到 MySQL → 返回确认给发送方
3. 推送：查询在线状态 → 推送给在线用户 → 离线推送
4. 接收方收到 → 客户端去重（msg_id）→ 显示

失败场景：
  步骤1失败 → 发送方未收到确认 → 重试 → 消息服务收到两条 → 用 client_msg_id 去重
  步骤2失败 → 存储失败 → 不推送 → 发送方收到失败确认 → 重试
  步骤3失败 → 推送失败 → 接收方上线时主动拉取 → 不丢消息
  步骤4重复 → 客户端去重 → 不重显示
```

**服务端去重（client_msg_id）：**

```python
class MessageDeduplicator:
    def dedup(self, msg):
        """服务端去重：同一 client_msg_id 只处理一次"""
        key = f"msg_dedup:{msg.sender_id}:{msg.client_msg_id}"
        if self.redis.setnx(key, msg.msg_id):
            self.redis.expire(key, 300)  # 5 分钟过期
            return False  # 非重复
        else:
            # 重复消息 → 返回已处理的 msg_id
            existing_msg_id = self.redis.get(key)
            return existing_msg_id
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

### 陷阱 5：不做推送频率控制

**后果：** 活跃群聊中离线用户每分钟收到 30+ 条推送通知 → 手机震动不停 → 用户体验极差 → 关闭通知 → 后续消息无法触达。

**解决方案：** 每分钟最多 3 条推送，超限合并为"有N条新消息"。

### 陷阱 6：全量 MySQL 存储

**后果：** 50 亿条/天 × 永久存储 → 1 年 = 365TB → MySQL 年成本 ¥182 万 → 不可承受。

**解决方案：** 冷热分层——MySQL 只存 30 天热数据，历史归档到对象存储（年成本 ¥11 万）。

### 陷阱 7：忽略消息乱序

**后果：** 客户端按 msg_id 排序，但 Snowflake ID 因时钟偏差导致乱序 → 对话逻辑混乱，用户看到回复在问题之前。

**解决方案：** 客户端始终按 seq 排序而非 msg_id；服务端推送消息时携带 seq；客户端检测 seq 不连续时主动拉取补齐。

### 陷阱 8：群成员变更期间消息投递不一致

**后果：** 用户 A 刚被移出群聊，但仍有未处理的消息推送到达 → A 看到了不该看到的消息；或用户 B 刚加入群聊，但错过入群后的前几条消息。

**解决方案：** 群成员变更时记录 join_seq / leave_seq，消息投递前校验成员关系，客户端按 seq 范围拉取。

## 异常场景深度分析

### 场景 1：消息乱序与修复

```python
class MessageOrderFixer:
    """
    消息乱序修复：
    客户端检测到 seq 不连续时，主动拉取补齐。
    """

    def on_message_received(self, msg, local_max_seq):
        """
        收到新消息时检查 seq 连续性。
        如果有间隔，说明中间有消息丢失或乱序到达。
        """
        expected_seq = local_max_seq + 1

        if msg.seq == expected_seq:
            # 连续，正常处理
            self.insert_message(msg)
            return

        elif msg.seq > expected_seq:
            # 不连续，有间隔 [expected_seq, msg.seq)
            # 先将当前消息存入待处理缓冲区
            self.pending_buffer[msg.seq] = msg

            # 主动拉取间隔中的消息
            missing_messages = self.fetch_missing(
                msg.conversation_id,
                from_seq=expected_seq,
                to_seq=msg.seq - 1
            )

            # 按序插入缺失消息
            for m in sorted(missing_messages, key=lambda x: x.seq):
                self.insert_message(m)

            # 插入当前消息
            self.insert_message(msg)

            # 清理缓冲区
            self.pending_buffer.pop(msg.seq, None)

        elif msg.seq <= local_max_seq:
            # 重复消息或已处理过的消息，去重
            return

    def fetch_missing(self, conversation_id, from_seq, to_seq):
        """从服务端拉取缺失消息"""
        response = self.api_client.get(
            "/messages/sync",
            params={
                "conversation_id": conversation_id,
                "from_seq": from_seq,
                "to_seq": to_seq
            }
        )
        return response.get("messages", [])
```

### 场景 2：服务端故障导致消息投递中断

```python
class ServerFailoverHandler:
    """
    服务端故障时的消息投递保障：
    1. 消息服务故障 → 网关本地缓冲，故障恢复后重发
    2. 网关故障 → 客户端自动重连其他网关，通过 seq 差量拉取补齐
    3. 存储服务故障 → 消息写入 Kafka 缓冲，存储恢复后消费写入
    """

    def on_message_service_unavailable(self, gateway):
        """消息服务不可用时的网关侧处理"""
        # 网关将客户端消息暂存到本地 Kafka/RocksDB
        gateway.enable_local_buffer()

    def on_message_service_recovered(self, gateway):
        """消息服务恢复后，刷写缓冲消息"""
        buffered_messages = gateway.drain_local_buffer()
        for msg in buffered_messages:
            self.message_service.send_message(msg)

    def on_gateway_failure(self, gateway_id, user_ids):
        """
        网关故障时的处理：
        1. 客户端检测到连接断开，自动重连其他网关
        2. 新网关接管后，客户端上报本地 max_seq，服务端返回差量消息
        """
        # 清理故障网关的连接记录
        self.presence.on_gateway_failure(gateway_id)

        # 通知客户端重连
        for uid in user_ids:
            self.push_service.send_reconnect_notification(uid)

    def on_client_reconnect(self, user_id, new_gateway_id):
        """客户端重连后的消息补齐"""
        # 获取用户所有会话的本地 max_seq
        client_seqs = self.get_client_reported_seqs(user_id)

        for conv_id, client_max_seq in client_seqs.items():
            server_max_seq = self.get_server_max_seq(conv_id)
            if client_max_seq < server_max_seq:
                # 推送差量消息
                messages = self.db.query("""
                    SELECT * FROM messages
                    WHERE conversation_id = %s AND seq > %s
                    ORDER BY seq ASC
                    LIMIT 200
                """, conv_id, client_max_seq)
                self.push_to_user(user_id, messages)
```

### 场景 3：群成员变更期间消息投递

```python
class GroupMemberChangeHandler:
    """
    群成员变更期间的消息一致性保障：
    - 加入时间用 join_seq 标记，只拉取 join_seq 之后的消息
    - 退出时间用 leave_seq 标记，不再推送 leave_seq 之后的消息
    - 变更操作与消息投递需要串行化（同一个群的操作走同一个队列）
    """

    def on_member_join(self, group_id: str, user_id: str):
        """成员加入群聊"""
        # 获取当前群 seq 作为 join_seq
        current_seq = self.redis.get(f"group_max_seq:{group_id}")
        join_seq = int(current_seq or 0)

        # 写入成员表
        self.db.execute("""
            INSERT INTO group_members (group_id, user_id, join_seq)
            VALUES (%s, %s, %s)
        """, group_id, user_id, join_seq)

        # 推送群历史消息摘要（不推送全部历史，只推送最近 N 条）
        recent_messages = self.db.query("""
            SELECT * FROM messages
            WHERE conversation_id = %s AND seq > %s
            ORDER BY seq DESC LIMIT 50
        """, group_id, max(0, join_seq - 50))
        self.push_to_user(user_id, reversed(recent_messages))

        # 通知其他成员有新人加入
        self.broadcast_system_message(group_id, {
            "type": "member_joined",
            "user_id": user_id,
            "seq": join_seq
        })

    def on_member_leave(self, group_id: str, user_id: str):
        """成员退出群聊"""
        # 获取当前群 seq 作为 leave_seq
        current_seq = int(self.redis.get(f"group_max_seq:{group_id}") or 0)

        # 标记成员退出（不删除记录，保留审计）
        self.db.execute("""
            UPDATE group_members
            SET leave_seq = %s
            WHERE group_id = %s AND user_id = %s
        """, current_seq, group_id, user_id)

        # 从推送列表中移除
        self.presence.remove_from_group(group_id, user_id)

        # 清理该用户在此群的已读进度（可选，节省存储）
        # self.db.execute("""
        #     DELETE FROM read_progress
        #     WHERE user_id = %s AND conversation_id = %s
        # """, user_id, group_id)

    def validate_member_access(self, group_id: str, user_id: str,
                                msg_seq: int) -> bool:
        """
        校验用户是否有权查看某条消息。
        在群成员变更与消息投递并发时使用。
        """
        member = self.db.query_one("""
            SELECT join_seq, leave_seq FROM group_members
            WHERE group_id = %s AND user_id = %s
        """, group_id, user_id)

        if not member:
            return False  # 非群成员

        if msg_seq < member['join_seq']:
            return False  # 消息在加入之前

        if member['leave_seq'] is not None and msg_seq > member['leave_seq']:
            return False  # 消息在退出之后

        return True

    def on_message_to_group(self, msg, group_id: str):
        """
        群消息投递时过滤成员：
        只推送给 join_seq <= msg.seq 且 leave_seq 为 NULL 的成员
        """
        members = self.db.query("""
            SELECT user_id FROM group_members
            WHERE group_id = %s AND leave_seq IS NULL
        """, group_id)

        online_status = self.presence.batch_check_online(
            [m['user_id'] for m in members]
        )

        for uid, online in online_status.items():
            if online:
                self.push_service.push_online(uid, msg)
            else:
                self.push_service.push_offline(uid, msg)
```

### 场景 4：离线消息积压与批量拉取

```python
class OfflineMessageBacklog:
    """
    离线消息积压处理：
    - 用户长时间离线后上线，可能有大量未读消息
    - 不能一次性推送所有消息（客户端内存和带宽有限）
    - 采用分页拉取策略，先推未读计数，再按需拉取
    """

    MAX_PUSH_BATCH = 200  # 单次推送上限
    MAX_SYNC_WINDOW = 5000  # 单次同步最大 seq 范围

    def on_user_online(self, user_id: str):
        """用户上线后的离线消息同步"""
        # 1. 获取用户所有会话的未读摘要
        summaries = self._get_unread_summaries(user_id)

        # 2. 先推送未读摘要（仅计数，不推送消息体）
        self.push_to_user(user_id, {
            "type": "unread_summary",
            "conversations": summaries
        })

        # 3. 对每个会话，推送最近的未读消息（最多 MAX_PUSH_BATCH 条）
        for summary in summaries:
            conv_id = summary["conversation_id"]
            unread_count = summary["unread_count"]

            if unread_count <= self.MAX_PUSH_BATCH:
                # 未读数不多，直接推送
                messages = self._fetch_unread(conv_id, summary["read_seq"])
                self.push_to_user(user_id, messages)
            else:
                # 未读数很多，只推送最新的一批 + 标记"有更多历史消息"
                latest = self._fetch_latest(conv_id, self.MAX_PUSH_BATCH)
                self.push_to_user(user_id, latest)
                self.push_to_user(user_id, {
                    "type": "has_more_history",
                    "conversation_id": conv_id,
                    "total_unread": unread_count,
                    "oldest_seq_in_batch": latest[0]["seq"]
                })

    def _get_unread_summaries(self, user_id: str) -> list:
        """获取所有会话的未读消息摘要"""
        read_progress = self.db.query("""
            SELECT conversation_id, read_seq FROM read_progress
            WHERE user_id = %s
        """, user_id)

        summaries = []
        for progress in read_progress:
            conv_id = progress["conversation_id"]
            max_seq = self._get_conv_max_seq(conv_id)
            unread_count = max_seq - progress["read_seq"]

            if unread_count > 0:
                summaries.append({
                    "conversation_id": conv_id,
                    "read_seq": progress["read_seq"],
                    "max_seq": max_seq,
                    "unread_count": unread_count
                })

        return summaries

    def fetch_history(self, user_id: str, conversation_id: str,
                      before_seq: int, limit: int = 50) -> list:
        """
        客户端主动拉取历史消息（向上翻页）。
        按需拉取，避免一次性推送过多。
        """
        return self.db.query("""
            SELECT * FROM messages
            WHERE conversation_id = %s AND seq < %s
            ORDER BY seq DESC
            LIMIT %s
        """, conversation_id, before_seq, limit)
```

### 场景 5：接收方多设备同时连接的消息投递

```python
import json
import time
import logging
from datetime import datetime
from collections import defaultdict

logger = logging.getLogger(__name__)


class MultiDeviceDeliveryHandler:
    """
    多设备同时在线的消息投递处理：
    - 同一用户手机 + PC + Web 同时在线
    - 消息需要推送到所有设备，但不同设备的 ACK 时序不同
    - 需要处理：设备间 ACK 独立、重试独立、已读状态聚合
    - 关键：一个设备 ACK 失败不影响其他设备的投递
    """

    def __init__(self, redis, db, presence, push_service):
        self.redis = redis
        self.db = db
        self.presence = presence
        self.push_service = push_service

    def deliver_to_user_multi_device(self, user_id: str, msg) -> dict:
        """
        向用户的所有在线设备投递消息
        返回每个设备的投递结果
        """
        # 获取用户所有在线设备
        devices = self._get_online_devices(user_id)

        if not devices:
            # 所有设备离线 → 走离线推送
            self._save_offline_message(user_id, msg)
            self.push_service.push_offline_notification(user_id, msg)
            return {"status": "offline", "devices": {}}

        # 向每个设备独立推送
        results = {}
        pending_acks = []

        for device_id, conn_info in devices.items():
            gateway_id = conn_info["gateway_id"]
            client_type = conn_info["client_type"]

            try:
                # 根据设备类型调整推送内容
                adapted_msg = self._adapt_message_for_device(msg, client_type)

                # 推送到指定设备的网关
                success = self.push_service.push_to_device(
                    user_id, device_id, gateway_id, adapted_msg
                )

                if success:
                    # 记录待 ACK 状态（按设备独立跟踪）
                    ack_key = f"msg_ack:{user_id}:{device_id}:{msg.msg_id}"
                    self.redis.setex(ack_key, 10, "pending")  # 10 秒超时
                    pending_acks.append((device_id, msg.msg_id))
                    results[device_id] = "delivered"
                else:
                    # 该设备推送失败 → 写入该设备的离线队列
                    self._save_device_offline_message(user_id, device_id, msg)
                    results[device_id] = "failed_queued"

            except Exception as e:
                logger.error(f"设备推送异常: user={user_id}, device={device_id}, "
                           f"error={e}")
                results[device_id] = "error"
                self._save_device_offline_message(user_id, device_id, msg)

        # 异步等待所有设备的 ACK
        self._schedule_ack_check(user_id, msg, pending_acks)

        return {"status": "delivered", "devices": results}

    def on_device_ack(self, user_id: str, device_id: str, msg_id: int):
        """
        收到某设备的消息 ACK
        - 标记该设备已收到
        - 更新该设备的 max_seq
        - 检查是否所有设备都已 ACK（可选优化）
        """
        # 清除 ACK 跟踪
        ack_key = f"msg_ack:{user_id}:{device_id}:{msg_id}"
        self.redis.delete(ack_key)

        # 更新设备的同步进度
        msg = self._get_message_by_id(msg_id)
        if msg:
            device_seq_key = f"device_seq:{user_id}:{device_id}"
            current = self.redis.get(device_seq_key)
            if current is None or int(current) < msg["seq"]:
                self.redis.set(device_seq_key, str(msg["seq"]))

        # 检查是否所有设备都已 ACK（用于更新全局投递状态）
        self._check_all_devices_acked(user_id, msg_id)

    def _schedule_ack_check(self, user_id: str, msg, pending_acks: list):
        """
        异步检查各设备 ACK 情况，超时重试
        每个设备独立重试，互不影响
        """
        for device_id, msg_id in pending_acks:
            # 延迟 10 秒后检查
            check_time = time.time() + 10
            self.redis.zadd("ack_check_queue", {
                json.dumps({
                    "user_id": user_id,
                    "device_id": device_id,
                    "msg_id": msg_id,
                    "retry_count": 0
                }): check_time
            })

    def process_ack_check_queue(self):
        """定时处理 ACK 检查队列"""
        MAX_RETRIES = 3

        now = time.time()
        pending = self.redis.zrangebyscore("ack_check_queue", 0, now)

        for item_json in pending:
            item = json.loads(item_json)
            user_id = item["user_id"]
            device_id = item["device_id"]
            msg_id = item["msg_id"]
            retry_count = item["retry_count"]

            # 检查该设备的 ACK 是否已收到
            ack_key = f"msg_ack:{user_id}:{device_id}:{msg_id}"
            if not self.redis.exists(ack_key):
                # 已收到 ACK，无需重试
                self.redis.zrem("ack_check_queue", item_json)
                continue

            if retry_count >= MAX_RETRIES:
                # 重试耗尽 → 写入该设备的离线队列
                msg = self._get_message_by_id(msg_id)
                if msg:
                    self._save_device_offline_message(user_id, device_id, msg)
                self.redis.delete(ack_key)
                self.redis.zrem("ack_check_queue", item_json)
                logger.warning(f"设备 ACK 重试耗尽: user={user_id}, "
                             f"device={device_id}, msg_id={msg_id}")
                continue

            # 重试推送
            conn_info = self._get_device_connection(user_id, device_id)
            if conn_info:
                msg = self._get_message_by_id(msg_id)
                if msg:
                    self.push_service.push_to_device(
                        user_id, device_id, conn_info["gateway_id"], msg
                    )

            # 重新加入队列，增加重试计数
            self.redis.zrem("ack_check_queue", item_json)
            item["retry_count"] = retry_count + 1
            next_check = time.time() + 10
            self.redis.zadd("ack_check_queue", {json.dumps(item): next_check})

    def _adapt_message_for_device(self, msg, client_type: int):
        """
        根据设备类型适配消息内容
        - Mobile: 不推送大文件内容，仅推送缩略图 URL
        - Web/Desktop: 推送完整内容
        """
        if client_type in (1, 2):  # iOS / Android
            adapted = dict(msg)
            if msg.content_type in (2, 4):  # 图片/视频
                # 只推送缩略图，大文件由客户端按需下载
                adapted["content"] = json.dumps({
                    "thumbnail_url": self._get_thumbnail(msg.content),
                    "full_url": msg.content,  # 客户端按需拉取
                    "preload": False
                })
            return adapted
        else:  # Web / Desktop
            return msg

    def _get_online_devices(self, user_id: str) -> dict:
        """获取用户所有在线设备"""
        conn_key = f"device_conn:{user_id}"
        devices = self.redis.hgetall(conn_key)
        return {did: json.loads(info) for did, info in devices.items()}

    def _get_device_connection(self, user_id: str, device_id: str) -> dict:
        """获取设备连接信息"""
        conn_key = f"device_conn:{user_id}"
        info = self.redis.hget(conn_key, device_id)
        return json.loads(info) if info else None

    def _save_offline_message(self, user_id: str, msg):
        """保存全局离线消息"""
        key = f"offline_msg:{user_id}:{msg.conversation_id}"
        self.redis.zadd(key, {str(msg.msg_id): msg.seq})
        self.redis.expire(key, 7 * 86400)

    def _save_device_offline_message(self, user_id: str, device_id: str, msg):
        """保存设备级离线消息（该设备上线后拉取）"""
        key = f"device_offline:{user_id}:{device_id}"
        self.redis.zadd(key, {str(msg.msg_id): msg.seq})
        self.redis.expire(key, 7 * 86400)

    def _check_all_devices_acked(self, user_id: str, msg_id: int):
        """检查所有设备是否都已 ACK（可选优化，用于全局投递状态）"""
        devices = self._get_online_devices(user_id)
        all_acked = True
        for device_id in devices:
            ack_key = f"msg_ack:{user_id}:{device_id}:{msg_id}"
            if self.redis.exists(ack_key):
                all_acked = False
                break

        if all_acked:
            # 所有设备都已确认 → 可更新全局投递状态
            pass

    def _get_message_by_id(self, msg_id: int) -> dict:
        """根据 msg_id 查询消息"""
        return self.db.query_one("""
            SELECT * FROM messages WHERE msg_id = %s
        """, msg_id)

    def _get_thumbnail(self, content: str) -> str:
        """获取缩略图 URL"""
        # 简化：在原 URL 基础上添加缩略图后缀
        return content.replace("/original/", "/thumbnail/")
```

### 场景 6：大群（10K+ 成员）消息发送限流

```python
import time
import logging
from datetime import datetime, timedelta

logger = logging.getLogger(__name__)


class LargeGroupRateLimiter:
    """
    大群消息发送限流器：
    - 10K+ 成员的大群，单条消息扇出成本极高
    - 必须对发送频率进行限流，防止消息洪泛
    - 多级限流：用户级 + 群级 + 全局级
    - 限流不丢消息，超限消息排队延迟投递
    """

    # 限流阈值配置
    GROUP_RATE_LIMITS = {
        # (min_members, max_members) → (max_msgs_per_sec, max_msgs_per_min)
        (0, 500):       (100, 3000),     # 小群：宽松
        (500, 5000):    (50, 1500),      # 中群
        (5000, 10000):  (20, 600),       # 大群
        (10000, 50000): (10, 300),       # 超大群
        (50000, float('inf')): (5, 150),  # 巨型群（5万+）
    }

    # 用户在群内的发送限流
    USER_GROUP_RATE = {
        "normal":   (5, 30),    # 普通用户：5条/秒, 30条/分钟
        "admin":    (10, 60),   # 管理员：10条/秒, 60条/分钟
        "owner":    (20, 120),  # 群主：20条/秒, 120条/分钟
    }

    def __init__(self, redis, db, message_queue):
        self.redis = redis
        self.db = db
        self.message_queue = message_queue  # 延迟投递队列

    def check_and_acquire(self, group_id: str, user_id: str,
                          member_count: int, user_role: int = 0) -> dict:
        """
        检查并发送许可：
        返回 {"allowed": True} 或 {"allowed": False, "retry_after": X}
        """
        # ---- 第一级：群级限流 ----
        group_result = self._check_group_rate(group_id, member_count)
        if not group_result["allowed"]:
            return group_result

        # ---- 第二级：用户在群内的限流 ----
        user_result = self._check_user_group_rate(group_id, user_id, user_role)
        if not user_result["allowed"]:
            return user_result

        # ---- 第三级：全局限流（防止突发流量打垮系统） ----
        global_result = self._check_global_rate()
        if not global_result["allowed"]:
            return global_result

        return {"allowed": True}

    def _check_group_rate(self, group_id: str, member_count: int) -> dict:
        """群级限流：使用滑动窗口算法"""
        # 确定该群规模的限流阈值
        max_per_sec, max_per_min = self._get_group_limits(member_count)

        # 检查秒级限流
        sec_key = f"rate:group:sec:{group_id}"
        sec_count = self.redis.incr(sec_key)
        if sec_count == 1:
            self.redis.expire(sec_key, 1)
        if sec_count > max_per_sec:
            return {"allowed": False, "retry_after": 1,
                    "reason": f"群消息频率超限({max_per_sec}条/秒)"}

        # 检查分钟级限流
        min_key = f"rate:group:min:{group_id}"
        min_count = self.redis.incr(min_key)
        if min_count == 1:
            self.redis.expire(min_key, 60)
        if min_count > max_per_min:
            return {"allowed": False, "retry_after": 60,
                    "reason": f"群消息频率超限({max_per_min}条/分钟)"}

        return {"allowed": True}

    def _check_user_group_rate(self, group_id: str, user_id: str,
                                user_role: int) -> dict:
        """用户在群内的发送限流"""
        role_key = "owner" if user_role == 2 else "admin" if user_role == 1 else "normal"
        max_per_sec, max_per_min = self.USER_GROUP_RATE[role_key]

        # 秒级限流
        sec_key = f"rate:user:group:sec:{user_id}:{group_id}"
        sec_count = self.redis.incr(sec_key)
        if sec_count == 1:
            self.redis.expire(sec_key, 1)
        if sec_count > max_per_sec:
            return {"allowed": False, "retry_after": 1,
                    "reason": f"个人发送频率超限({max_per_sec}条/秒)"}

        # 分钟级限流
        min_key = f"rate:user:group:min:{user_id}:{group_id}"
        min_count = self.redis.incr(min_key)
        if min_count == 1:
            self.redis.expire(min_key, 60)
        if min_count > max_per_min:
            return {"allowed": False, "retry_after": 60,
                    "reason": f"个人发送频率超限({max_per_min}条/分钟)"}

        return {"allowed": True}

    def _check_global_rate(self) -> dict:
        """全局限流：保护系统整体吞吐"""
        max_global_qps = 500000  # 50 万/秒 全局上限

        key = "rate:global:qps"
        count = self.redis.incr(key)
        if count == 1:
            self.redis.expire(key, 1)
        if count > max_global_qps:
            return {"allowed": False, "retry_after": 1,
                    "reason": "系统繁忙，请稍后重试"}

        return {"allowed": True}

    def _get_group_limits(self, member_count: int) -> tuple:
        """根据群规模获取限流阈值"""
        for (min_m, max_m), limits in self.GROUP_RATE_LIMITS.items():
            if min_m <= member_count < max_m:
                return limits
        return (5, 150)  # 默认最严格

    def enqueue_delayed_message(self, msg, group_id: str, delay_seconds: int):
        """
        限流消息延迟投递：
        超限消息不丢弃，而是进入延迟队列
        """
        delivery_time = time.time() + delay_seconds
        self.redis.zadd("delayed_message_queue", {
            json.dumps({
                "msg_id": msg.msg_id,
                "conversation_id": msg.conversation_id,
                "group_id": group_id
            }): delivery_time
        })
        logger.info(f"消息延迟投递: msg_id={msg.msg_id}, "
                   f"delay={delay_seconds}s")

    def process_delayed_queue(self):
        """定时处理延迟消息队列"""
        now = time.time()
        pending = self.redis.zrangebyscore("delayed_message_queue", 0, now)

        for item_json in pending:
            item = json.loads(item_json)
            group_id = item["group_id"]
            member_count = self._get_group_member_count(group_id)

            # 重新检查限流
            group_result = self._check_group_rate(group_id, member_count)
            if group_result["allowed"]:
                # 可以投递了
                msg = self._get_message(item["msg_id"])
                if msg:
                    self._deliver_group_message(msg, group_id)
                self.redis.zrem("delayed_message_queue", item_json)
            # 否则继续等待下次检查

    def _get_group_member_count(self, group_id: str) -> int:
        """获取群成员数"""
        cache_key = f"grp_member_count:{group_id}"
        count = self.redis.get(cache_key)
        if count:
            return int(count)

        count = self.db.query_one("""
            SELECT COUNT(*) as cnt FROM group_members
            WHERE group_id = %s AND leave_seq IS NULL
        """, group_id)["cnt"]
        self.redis.setex(cache_key, 300, str(count))
        return count
```

### 场景 7：端到端加密密钥轮换

```python
import json
import time
import logging
import hashlib
from datetime import datetime, timedelta

logger = logging.getLogger(__name__)


class E2EKeyRotationService:
    """
    端到端加密密钥轮换服务：
    - 每个会话/群有独立的加密密钥
    - 定期轮换密钥（24小时或每1000条消息）
    - 密钥轮换期间不影响正在进行的消息收发
    - 历史消息用旧密钥解密，新消息用新密钥加密
    - 群聊密钥轮换需所有成员协商
    """

    KEY_ROTATION_INTERVAL = timedelta(hours=24)  # 24小时轮换
    KEY_ROTATION_MSG_COUNT = 1000  # 每1000条消息轮换
    KEY_VERSION_FORMAT = "v{conversation_id}_{timestamp}_{index}"

    def __init__(self, redis, db, kms_client, push_service):
        self.redis = redis
        self.db = db
        self.kms = kms_client  # 密钥管理服务
        self.push_service = push_service

    def get_current_key(self, conversation_id: str) -> dict:
        """
        获取当前会话的加密密钥信息
        返回：{"key_id": "...", "key_version": "...", "key": bytes, "created_at": ...}
        """
        cache_key = f"e2e_key:current:{conversation_id}"
        cached = self.redis.get(cache_key)
        if cached:
            return json.loads(cached)

        # 从 KMS 获取
        key_info = self.kms.get_current_key(conversation_id)
        if key_info:
            self.redis.setex(cache_key, 300, json.dumps(key_info))
        return key_info

    def should_rotate_key(self, conversation_id: str) -> bool:
        """判断是否需要轮换密钥"""
        key_info = self.get_current_key(conversation_id)
        if not key_info:
            return True  # 无密钥，需要创建

        # 条件1：超过轮换时间
        created_at = datetime.fromisoformat(key_info["created_at"])
        if datetime.now() - created_at > self.KEY_ROTATION_INTERVAL:
            return True

        # 条件2：超过消息数阈值
        msg_count_key = f"e2e_key:msg_count:{conversation_id}"
        msg_count = int(self.redis.get(msg_count_key) or 0)
        if msg_count >= self.KEY_ROTATION_MSG_COUNT:
            return True

        return False

    def rotate_key_for_conversation(self, conversation_id: str):
        """
        单聊密钥轮换：
        1. 生成新密钥
        2. 用双方公钥加密新密钥
        3. 推送密钥更新通知给双方
        4. 新消息使用新密钥，旧消息仍用旧密钥
        """
        logger.info(f"开始密钥轮换: conversation={conversation_id}")

        # 1. 获取当前密钥版本
        current_key = self.get_current_key(conversation_id)
        current_version = current_key["key_version"] if current_key else "v0"

        # 2. 生成新密钥
        new_key_id = self.kms.generate_key_id()
        new_key_version = f"v{conversation_id}_{int(time.time())}"
        new_key = self.kms.generate_data_key()  # 256-bit AES key

        # 3. 存储新密钥到 KMS（服务端只存储加密后的密钥）
        self.kms.store_key(conversation_id, new_key_id, new_key_version, new_key)

        # 4. 记录密钥版本映射（用于解密历史消息）
        self.db.execute("""
            INSERT INTO e2e_key_versions
                (conversation_id, key_id, key_version, predecessor_version,
                 created_at, status)
            VALUES (%s, %s, %s, %s, %s, 1)
        """, conversation_id, new_key_id, new_key_version,
             current_version, datetime.now())

        # 5. 更新缓存
        new_key_info = {
            "key_id": new_key_id,
            "key_version": new_key_version,
            "created_at": datetime.now().isoformat()
        }
        self.redis.set(f"e2e_key:current:{conversation_id}",
                       json.dumps(new_key_info))
        self.redis.set(f"e2e_key:msg_count:{conversation_id}", "0")

        # 6. 为每个参与者加密新密钥并推送
        participants = self._get_participants(conversation_id)
        for user_id in participants:
            # 用用户的公钥加密新密钥
            user_public_key = self.kms.get_user_public_key(user_id)
            encrypted_key = self.kms.encrypt_with_public_key(
                new_key, user_public_key
            )

            # 推送密钥更新通知
            self.push_service.push_to_user(user_id, {
                "type": "e2e_key_rotation",
                "conversation_id": conversation_id,
                "new_key_id": new_key_id,
                "new_key_version": new_key_version,
                "encrypted_key": encrypted_key.hex(),
                "predecessor_version": current_version,
                "timestamp": datetime.now().isoformat()
            })

        logger.info(f"密钥轮换完成: conversation={conversation_id}, "
                   f"new_version={new_key_version}")

    def rotate_key_for_group(self, group_id: str):
        """
        群聊密钥轮换（比单聊更复杂）：
        1. 生成新群密钥
        2. 用每个成员的公钥加密新密钥
        3. 推送密钥更新通知给所有成员
        4. 处理轮换期间成员变更
        """
        logger.info(f"开始群密钥轮换: group={group_id}")

        # 获取群成员列表
        members = self.db.query("""
            SELECT user_id FROM group_members
            WHERE group_id = %s AND leave_seq IS NULL
        """, group_id)

        if not members:
            return

        # 生成新密钥
        new_key_id = self.kms.generate_key_id()
        new_key_version = f"v{group_id}_{int(time.time())}"
        new_key = self.kms.generate_data_key()

        # 存储新密钥
        self.kms.store_key(group_id, new_key_id, new_key_version, new_key)

        # 记录密钥版本
        current_key = self.get_current_key(group_id)
        current_version = current_key["key_version"] if current_key else "v0"
        self.db.execute("""
            INSERT INTO e2e_key_versions
                (conversation_id, key_id, key_version, predecessor_version,
                 created_at, status)
            VALUES (%s, %s, %s, %s, %s, 1)
        """, group_id, new_key_id, new_key_version,
             current_version, datetime.now())

        # 为每个成员加密并推送
        for member in members:
            user_id = member["user_id"]
            user_public_key = self.kms.get_user_public_key(user_id)
            encrypted_key = self.kms.encrypt_with_public_key(
                new_key, user_public_key
            )
            self.push_service.push_to_user(user_id, {
                "type": "e2e_key_rotation",
                "conversation_id": group_id,
                "new_key_id": new_key_id,
                "new_key_version": new_key_version,
                "encrypted_key": encrypted_key.hex(),
                "predecessor_version": current_version
            })

        # 更新缓存
        new_key_info = {
            "key_id": new_key_id,
            "key_version": new_key_version,
            "created_at": datetime.now().isoformat()
        }
        self.redis.set(f"e2e_key:current:{group_id}", json.dumps(new_key_info))
        self.redis.set(f"e2e_key:msg_count:{group_id}", "0")

    def on_member_join_group(self, group_id: str, user_id: str):
        """
        新成员加入群聊时的密钥处理：
        安全要求：新成员不能解密加入前的历史消息
        → 必须在成员加入时轮换密钥
        """
        # 立即轮换密钥
        self.rotate_key_for_group(group_id)

        # 新成员只获得新密钥，无法解密旧消息
        logger.info(f"新成员入群触发密钥轮换: group={group_id}, user={user_id}")

    def on_member_leave_group(self, group_id: str, user_id: str):
        """
        成员退出群聊时的密钥处理：
        安全要求：退出成员不能解密后续消息
        → 必须在成员退出时轮换密钥
        """
        # 立即轮换密钥（退出成员不会收到新密钥）
        self.rotate_key_for_group(group_id)

        # 清除退出成员的密钥缓存
        self.redis.delete(f"e2e_key:user:{user_id}:{group_id}")

        logger.info(f"成员退群触发密钥轮换: group={group_id}, user={user_id}")

    def decrypt_message(self, conversation_id: str, msg_id: int,
                         encrypted_content: bytes,
                         key_version: str) -> str:
        """
        解密消息：
        根据消息的 key_version 找到对应的密钥版本解密
        """
        # 查找密钥版本
        key_info = self.db.query_one("""
            SELECT key_id, key_version FROM e2e_key_versions
            WHERE conversation_id = %s AND key_version = %s
        """, conversation_id, key_version)

        if not key_info:
            raise KeyError(f"密钥版本不存在: {key_version}")

        # 从 KMS 获取密钥（客户端侧操作，服务端通常无法解密）
        key = self.kms.get_key(key_info["key_id"])
        decrypted = self.kms.decrypt(encrypted_content, key)
        return decrypted

    def on_message_sent(self, conversation_id: str):
        """消息发送后递增计数，检查是否需要轮换"""
        count_key = f"e2e_key:msg_count:{conversation_id}"
        count = self.redis.incr(count_key)

        if count >= self.KEY_ROTATION_MSG_COUNT:
            # 异步触发轮换（不阻塞消息发送）
            self._async_rotate(conversation_id)

    def _async_rotate(self, conversation_id: str):
        """异步触发密钥轮换"""
        # 使用分布式锁防止并发轮换
        lock_key = f"e2e_key:rotate_lock:{conversation_id}"
        if self.redis.setnx(lock_key, "1"):
            self.redis.expire(lock_key, 60)
            try:
                if self._is_group(conversation_id):
                    self.rotate_key_for_group(conversation_id)
                else:
                    self.rotate_key_for_conversation(conversation_id)
            finally:
                self.redis.delete(lock_key)

    def _get_participants(self, conversation_id: str) -> list:
        """获取会话参与者"""
        if "_" in conversation_id and not conversation_id.startswith("grp_"):
            return conversation_id.split("_")
        else:
            members = self.db.query("""
                SELECT user_id FROM group_members
                WHERE group_id = %s AND leave_seq IS NULL
            """, conversation_id)
            return [m["user_id"] for m in members]
```

#### 密钥版本管理数据库表

```sql
-- E2E 加密密钥版本表
CREATE TABLE e2e_key_versions (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    conversation_id VARCHAR(64) NOT NULL,
    key_id VARCHAR(128) NOT NULL,            -- KMS 中的密钥 ID
    key_version VARCHAR(128) NOT NULL,       -- 密钥版本标识
    predecessor_version VARCHAR(128),         -- 前一版本（用于版本链追溯）
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    status TINYINT NOT NULL DEFAULT 1,       -- 1=生效, 0=已轮换, -1=已吊销
    UNIQUE KEY uk_conv_version (conversation_id, key_version),
    INDEX idx_conv_status (conversation_id, status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## 性能与成本分析

### 消息存储成本：热/温/冷分层明细

```
日均消息量：50 亿条
每条消息平均大小：200 字节（含元数据）

日增量：
  50 亿 × 200B = 1TB/天

月增量：
  1TB × 30 = 30TB

年增量：
  1TB × 365 = 365TB
```

#### 消息存储成本逐层明细

```
=== 热数据层（MySQL 分库分表，0-30 天）===

数据量：30TB
存储介质：NVMe SSD（云盘）
单价：¥0.5/GB/月

成本计算：
  存储费：30TB × ¥0.5/GB = 30 × 1024 × 0.5 = ¥15,360/月
  计算费：256 分库 × 2 主从 × 4核8G = 512 实例
          512 × ¥300/月 = ¥153,600/月
  合计：¥168,960/月 ≈ ¥202 万/年

热数据查询延迟：< 10ms（本地 SSD 缓存命中 < 1ms）
热数据 QPS 能力：256 分库 × 5000 QPS/库 = 128 万 QPS

=== 温数据层（Elasticsearch，31-90 天）===

数据量：60TB（31-90 天，共 60 天增量）
存储介质：SSD（ES 要求低延迟）
单价：¥0.3/GB/月（含副本约 ¥0.6/GB/月）

成本计算：
  存储费：60TB × ¥0.6/GB = 60 × 1024 × 0.6 = ¥36,864/月
  计算费：12 数据节点 × 8核32G = 12 × ¥1200/月 = ¥14,400/月
  合计：¥51,264/月 ≈ ¥61 万/年

温数据查询延迟：< 100ms
温数据 QPS 能力：12 节点 × 1000 QPS/节点 = 1.2 万 QPS

=== 冷数据层（S3/OSS 对象存储，91 天+）===

数据量：275TB（全年累计，91 天以上）
存储介质：对象存储标准/低频
单价：¥0.05/GB/月（标准），¥0.02/GB/月（归档）

成本计算（分级存储）：
  标准存储（91-365 天）：185TB × ¥0.05/GB = ¥9,472/月
  归档存储（1 年以上）：90TB × ¥0.02/GB = ¥1,843/月
  合计：¥11,315/月 ≈ ¥13.6 万/年

冷数据查询延迟：1-5 秒（标准），5-60 分钟（归档需解冻）
冷数据 QPS 能力：对象存储几乎无限

=== 总存储成本对比 ===

冷热分层方案：
  热数据：¥202 万/年
  温数据：¥61 万/年
  冷数据：¥13.6 万/年
  合计：¥276.6 万/年

全量 MySQL 方案：
  365TB × ¥0.5/GB/月 + 计算费 = ¥1,872 + ¥1,843 = ¥371 万/年
  → 冷热分层节省 25%

全量 S3 方案：
  365TB × ¥0.05/GB/月 = ¥18.7 万/年
  → 延迟不可接受，但可用于合规归档

最优方案：冷热分层 + 计算资源优化
  优化后热数据计算费：读写分离，只读副本按需扩缩
  优化后约 ¥220 万/年
```

### 网络带宽分析：不同消息类型的流量

```
=== 消息类型流量分布 ===

消息类型分布（日均 50 亿条）：
  文本消息：60% = 30 亿条/天，平均 100 字节
  图片消息：15% = 7.5 亿条/天，平均消息体 200B + 图片 500KB
  文件消息：10% = 5 亿条/天，平均消息体 200B + 文件 2MB
  视频消息：5% = 2.5 亿条/天，平均消息体 200B + 视频 10MB
  系统消息：10% = 5 亿条/天，平均 50 字节

=== 带宽计算 ===

1. 文本消息带宽：
   发送：30 亿 × 100B = 30GB/天
   推送（平均 2 个接收方）：30 亿 × 2 × 100B = 60GB/天
   合计：90GB/天

2. 图片消息带宽：
   消息信令推送：7.5 亿 × 200B = 1.5GB/天
   图片下载（缩略图 50KB × 2 接收方）：7.5 亿 × 2 × 50KB = 750TB/天
   图片原文件下载（10% 用户点开大图）：7.5 亿 × 2 × 10% × 500KB = 750TB/天
   合计：~1500TB/天（图片是带宽大头！）

3. 文件消息带宽：
   消息信令推送：5 亿 × 200B = 1GB/天
   文件下载（30% 用户下载）：5 亿 × 2 × 30% × 2MB = 600TB/天
   合计：~600TB/天

4. 视频消息带宽：
   消息信令推送：2.5 亿 × 200B = 0.5GB/天
   视频播放（20% 用户播放）：2.5 亿 × 2 × 20% × 10MB = 1000TB/天
   合计：~1000TB/天

5. 系统消息带宽：
   5 亿 × 50B × 2 = 5GB/天

=== 带宽汇总 ===

消息信令（WebSocket 推送）：~100GB/天 → 峰值 ~200MB/s → 1.6Gbps
富媒体文件下载（CDN）：     ~3150TB/天 → 峰值 ~50Gbps（CDN 分担）

关键发现：
  - 消息信令带宽只占总带宽的 3%，97% 是富媒体文件下载
  - 图片和视频占文件流量的 80%+
  - 必须使用 CDN，否则源站带宽成本不可承受

=== 带宽成本估算 ===

WebSocket 信令带宽（直连）：
  峰值 1.6Gbps × ¥3000/Mbps/月 = ¥480 万/月 → 太贵！
  优化：使用云内网传输 + BGP 精品带宽
  实际：1.6Gbps × ¥500/Mbps/月 = ¥80 万/月 = ¥960 万/年
  进一步优化：消息压缩 + 增量推送 → 压缩到 0.5Gbps
  最终：¥300 万/年

CDN 富媒体带宽：
  日均 3150TB × ¥0.20/GB = ¥63 万/天 = ¥1,890 万/月 → 太贵！
  优化：缩略图走 CDN，原文件按需下载 + P2P 分享
  实际：CDN 日均 500TB × ¥0.15/GB = ¥75 万/天 = ¥2,250 万/月 → 仍贵
  进一步优化：客户端缓存 + 预加载 + 就近接入
  最终：¥800 万/年

带宽总成本：¥1,100 万/年（信令 ¥300 万 + CDN ¥800 万）
```

### 群聊扇出成本深度对比

```
=== 场景参数 ===
群规模：10 万成员
在线率：30%（3 万人在线）
消息速率：10 条/秒
平均消息大小：200 字节

=== 方案 1：纯写扩散 ===

写入量：
  每条消息 × 10 万次收件箱写入 = 10 条/秒 × 10 万 = 100 万次/秒

存储开销：
  收件箱存储：10 万 × 200B = 20MB/条消息
  每秒：10 × 20MB = 200MB/秒写入

Redis 集群成本：
  100 万 QPS ÷ 4 万 QPS/节点 = 25 节点
  25 × ¥3000/月 = ¥7.5 万/月

延迟：
  10 万次串行写入 ~500ms → 不可接受
  并行写入 ~50ms（10 并发批次）→ 勉强可接受

优势：接收方读取快（收件箱预存）
劣势：写入成本极高，存储膨胀

=== 方案 2：纯读扩散 ===

写入量：
  每条消息 × 1 次群消息表写入 = 10 次/秒

通知推送量：
  10 条/秒 × 3 万在线 = 30 万次/秒轻量通知

读取量：
  每个在线用户主动拉取：3 万 × 10 次/秒 = 30 万次/秒读取
  每次读取从 Redis 增量日志获取（快）或 DB（慢）

MySQL 成本：
  2 节点 = ¥0.6 万/月

Redis 成本（增量日志 + 通知）：
  5 节点 = ¥1.5 万/月

延迟：
  通知推送：~20ms
  客户端拉取：~50-100ms（含网络往返）
  总延迟：~70-120ms → 可接受

优势：写入成本低，存储精简
劣势：读取有额外延迟，依赖客户端主动拉取

=== 方案 3：混合扩散（推荐）===

小群（≤ 500 人）→ 写扩散：
  写入量可控：500 人群 × 10 条/小时 × 500 = 250 万次/小时 ≈ 700 次/秒
  适合 Redis 收件箱方案

大群（> 500 人）→ 读扩散 + 轻量通知：
  如方案 2 分析

超大群（> 5 万人）→ 读扩散 + 分级通知：
  第一级：通知群管理员（实时推送完整消息）
  第二级：通知普通成员（轻量信号，客户端拉取）
  第三级：离线成员（不主动推送，上线后拉取）

=== 三方案成本对比 ===

              纯写扩散     纯读扩散     混合扩散（推荐）
写入 QPS      100 万/秒    10/秒       ~1000/秒（含小群）
读取 QPS      ~0           30 万/秒    ~15 万/秒
存储/月       ¥7.5 万      ¥2.1 万     ¥3 万
延迟          50-500ms     70-120ms    小群<20ms, 大群70-120ms
实现复杂度    低           中          中高
推荐度        不推荐       可用        最佳

=== 10 万人大群月度成本明细（混合方案）===

群消息表写入（MySQL）：
  10 条/秒 × 200B = 2KB/秒 = 5.2GB/月
  存储：5.2GB × ¥0.5/GB = ¥2.6/月 → 可忽略
  计算费：2 节点 × ¥300/月 = ¥600/月

增量日志（Redis）：
  10 条/秒 × 200B × 300秒保留 = 600KB 常驻
  + 在线通知数据：3 万 × 200B = 6MB
  5 节点 × ¥3000/月 = ¥1.5 万/月

轻量通知推送：
  30 万次/秒 × 50B = 15MB/秒 = 38.9TB/月
  带宽：¥38.9 万/月（按 ¥1/GB）

总成本：¥40.5 万/月 → 主要成本在通知推送带宽
优化：通知合并（100ms 窗口内合并）、按需拉取 → 降至 ¥10 万/月
```

### 接入网关服务器规模

```
日活用户：3000 万
峰值在线：2000 万（日活 × 70%）
单台网关维持连接数：30 万（16GB 内存，每连接约 50KB）

网关服务器数量：
  2000 万 / 30 万 = 67 台 → 按 1.5 倍冗余 = 100 台

网关服务器成本（4 核 16GB 云服务器）：
  100 台 × ¥500/月 = ¥5 万/月 = ¥60 万/年

带宽成本：
  峰值 QPS 100 万 × 平均消息大小 200B = 200MB/s
  峰值带宽：200MB/s × 8 = 1.6Gbps
  月带宽费（按 95 峰值计费）：¥8 万/月 = ¥96 万/年
```

### 扇出写放大分析

```
写扩散方案（小群 ≤ 500 人）：
  1 条消息 × 500 次写入 = 写放大 500 倍
  500 人群 × 10 条/小时 × 500 = 250 万次写入/小时
  适合：群数多但群规模小，总写入可控

读扩散方案（大群 > 500 人）：
  1 条消息 × 1 次写入 = 写放大 1 倍
  但在线成员需要推送通知 → 10 万人 × 1 次通知推送
  通知推送是轻量级操作（仅告知有新消息），非完整消息体写入
  适合：群规模大，写扩散不可行

混合方案成本对比（10 万人大群，10 条/秒）：

  纯写扩散：
    写入量：10 × 10 万 = 100 万次/秒（Redis）
    Redis 集群：25 节点（每节点 4 万 QPS）
    月成本：25 × ¥3000 = ¥7.5 万

  读扩散 + 轻量通知：
    写入量：10 × 1 = 10 次/秒（MySQL）
    通知推送：10 × 3 万（30%在线）= 30 万次/秒
    Redis 月成本：5 节点 = ¥1.5 万
    MySQL 月成本：2 节点 = ¥0.6 万
    合计：¥2.1 万/月 → 节省 72%
```

### 系统总体成本估算

```
年度总成本（不含人力）：

  存储层：  ¥220 万（冷热分层，含热/温/冷 + 计算）
  网关层：  ¥60 万（100 台网关）
  信令带宽：¥300 万（WebSocket 推送，压缩后）
  CDN带宽： ¥800 万（图片/视频/文件分发）
  Redis：   ¥18 万（在线状态 + 缓存 + 去重，6 集群）
  MySQL：   ¥36 万（消息分库 + 元数据，16 分库 × 2 主从）
  Kafka：   ¥12 万（消息缓冲 + 异步解耦，3 集群）
  ES：      ¥97 万（消息搜索，活跃用户索引）
  推送：    ¥20 万（APNs + FCM + 厂商推送）
  合规审计：¥15 万（WORM 存储 + 审计日志存储）

  合计：¥1,578 万/年
  日均：¥4.32 万/天
  每千条消息：¥43,200 / (50亿/1000) = ¥0.00864/千条

成本结构分析：
  带宽（信令+CDN）占比：1100/1578 = 69.7% → 带宽是最大成本项
  存储占比：220/1578 = 13.9%
  计算占比：258/1578 = 16.3%

优化优先级：
  1. CDN 带宽优化（图片压缩 + 视频转码 + P2P 分享）→ 预计可省 40%
  2. 信令压缩（Protobuf 替代 JSON + 增量推送）→ 预计可省 30%
  3. 存储冷热分层 + 非活跃用户索引延迟构建 → 已优化
```

## 延伸思考

- **端到端加密**：服务器无法解密消息内容 → 搜索只能在客户端本地做。架构需增加密钥管理服务（KMS），每对会话/每个群有独立密钥，通过 Diffie-Hellman 密钥交换协商。消息体存储密文，元数据（sender_id, seq, timestamp）仍明文存储以支持投递和同步。密钥轮换时需处理历史消息的密钥版本映射。
- **音视频通话信令**：与消息系统共用 WebSocket 通道，但需要 SDP 交换和 ICE 协商。信令消息（offer/answer/candidate）走消息通道，但媒体流走 P2P 或 TURN 中继。需在接入网关层区分信令消息和普通消息的路由。
- **跨设备同步**：手机和电脑同时登录，消息和已读状态如何多端一致？使用 seq 作为同步锚点。每台设备独立维护本地 max_seq，上报到服务端后取所有设备最大值作为 read_progress。设备间通过服务端中转"已读同步消息"实时同步已读状态。
- **消息编辑**：编辑后的消息如何同步给所有接收方？方案：编辑不生成新 seq，而是在原 seq 上更新 content 和 edit_version 字段。客户端收到编辑通知后本地更新。需处理编辑与已读回执的交互——编辑前的已读状态仍然有效。
- **合规审计**：金融/政务场景要求消息不可篡改、可追溯。方案：消息写入后内容不可变（撤回只标记，不删除），增加操作日志表记录所有敏感操作（发送、撤回、编辑、删除），配合 WORM 存储确保审计数据不可篡改。
- **消息多模态扩展**：图片/文件/视频/位置/名片等富媒体消息。方案：content_type 区分消息类型，content 字段存储 JSON 结构体（包含 URL、缩略图、尺寸等元信息），文件存储走对象存储 + CDN 加速，消息体只存引用 URL。
- **柔性可用与降级策略**：存储服务故障时，消息先写入 Kafka 缓冲，保证"可发送"，但提示"消息可能延迟送达"；Redis 故障时，在线状态降级为"默认在线"，推送走全量拉取模式；网关过载时，新连接排队等待，已连接用户不受影响。
## IM 消息存储与归档完整实现

```python
class MessageStorageService:
    """IM 消息存储：分层 + 归档 + 搜索索引"""

    STORAGE_TIERS = {
        "hot": {"retention_days": 90, "storage": "MySQL + Redis"},
        "warm": {"retention_days": 365, "storage": "MySQL 冷表"},
        "cold": {"retention_days": None, "storage": "S3 + Elasticsearch"},
    }

    def store_message(self, message):
        """存储消息"""
        msg_id = str(uuid4())
        self.db.insert("messages_hot", {
            "msg_id": msg_id,
            "conversation_id": message["conversation_id"],
            "sender_id": message["sender_id"],
            "msg_type": message["msg_type"],  # text/image/video/file
            "content": message["content"],
            "created_at": now()
        })

        # 写入搜索索引
        if message["msg_type"] == "text":
            self.elasticsearch.index(index="messages", id=msg_id, body={
                "conversation_id": message["conversation_id"],
                "sender_id": message["sender_id"],
                "content": message["content"],
                "created_at": now().isoformat()
            })

        return msg_id

    def search_messages(self, user_id, query, conversation_id=None,
                        limit=20):
        """搜索消息"""
        # 1. 验证用户权限
        if conversation_id:
            if not self._is_member(user_id, conversation_id):
                return {"results": [], "error": "无权限"}

        # 2. Elasticsearch 搜索
        must = [{"match": {"content": query}}]
        if conversation_id:
            must.append({"term": {"conversation_id": conversation_id}})

        # 只搜索用户参与的会话
        user_conversations = self._get_user_conversations(user_id)
        must.append({"terms": {"conversation_id": user_conversations}})

        result = self.elasticsearch.search(index="messages", body={
            "query": {"bool": {"must": must}},
            "size": limit,
            "sort": [{"created_at": "desc"}]
        })

        messages = []
        for hit in result["hits"]["hits"]:
            messages.append({
                "msg_id": hit["_id"],
                "conversation_id": hit["_source"]["conversation_id"],
                "sender_id": hit["_source"]["sender_id"],
                "content": hit["_source"]["content"],
                "created_at": hit["_source"]["created_at"]
            })

        return {"results": messages, "total": result["hits"]["total"]["value"]}

    def archive_old_messages(self, days_threshold=90):
        """归档旧消息"""
        cutoff = now() - timedelta(days=days_threshold)

        # 从热表迁移到冷表
        old_messages = self.db.query(
            "SELECT * FROM messages_hot WHERE created_at < %s LIMIT 10000",
            cutoff)

        if not old_messages:
            return {"archived": 0}

        # 批量插入冷表
        for msg in old_messages:
            self.db.insert("messages_warm", msg)

        # 删除热表中的已归档数据
        self.db.execute(
            "DELETE FROM messages_hot WHERE created_at < %s LIMIT 10000",
            cutoff)

        return {"archived": len(old_messages)}
```

## IM 端到端加密

```python
class E2EEncryptionService:
    """IM 端到端加密：密钥交换 + 加密传输 + 密钥轮换"""

    def setup_session(self, user_a, user_b):
        """建立端到端加密会话（Signal 协议简化版）"""
        # 1. 获取双方预密钥
        prekey_a = self._get_prekey(user_a)
        prekey_b = self._get_prekey(user_b)

        # 2. X3DH 密钥协商
        shared_secret = self._x3dh(prekey_a, prekey_b)

        # 3. 派生会话密钥
        chain_key = self._hkdf(shared_secret, info="im_chain_key")
        message_key = self._hkdf(chain_key, info="im_message_key")

        # 存储会话状态
        session_id = str(uuid4())
        self.redis.setex(f"e2e_session:{user_a}:{user_b}", 86400 * 30,
            json.dumps({
                "session_id": session_id,
                "chain_key": chain_key.hex(),
                "message_count": 0,
                "created_at": now().isoformat()
            }))

        return {"session_id": session_id}

    def encrypt_message(self, sender_id, receiver_id, plaintext):
        """加密消息"""
        session = self._get_session(sender_id, receiver_id)
        if not session:
            session = self.setup_session(sender_id, receiver_id)

        # 派生消息密钥（棘轮推进）
        message_key = self._ratchet(session)

        # AES-256-GCM 加密
        nonce = os.urandom(12)
        cipher = self._aes_gcm_encrypt(message_key, nonce, plaintext)

        return {
            "session_id": session["session_id"],
            "nonce": nonce.hex(),
            "ciphertext": cipher.hex(),
            "message_number": session["message_count"]
        }

    def decrypt_message(self, receiver_id, sender_id, encrypted):
        """解密消息"""
        session = self._get_session(sender_id, receiver_id)
        if not session:
            raise DecryptionError("无加密会话")

        # 获取对应消息编号的密钥
        message_key = self._get_message_key(session, encrypted["message_number"])

        # AES-256-GCM 解密
        nonce = bytes.fromhex(encrypted["nonce"])
        ciphertext = bytes.fromhex(encrypted["ciphertext"])
        plaintext = self._aes_gcm_decrypt(message_key, nonce, ciphertext)

        return plaintext
```

## 异常场景补充

### 场景：消息搜索索引延迟

```
触发：Elasticsearch 写入延迟 → 新消息搜不到 → 用户投诉
检测：
  1. 搜索结果缺少最近 5 分钟消息 → 索引延迟
  2. ES 写入队列深度 > 10000 → 积压
处理：
  1. 增大 ES 写入并发
  2. 搜索时额外查询数据库补充最新消息
  3. 修复后验证索引同步
预防：双路查询（ES + DB） + 索引健康监控 + 写入扩容
```

### 场景：端到端加密密钥丢失

```
触发：用户更换设备 → 本地密钥丢失 → 无法解密历史消息
检测：
  1. 用户报告无法查看历史消息 → 密钥丢失
  2. 解密失败率突增 → 可能密钥问题
处理：
  1. 从云端备份恢复密钥（如果用户启用了备份）
  2. 未备份 → 历史消息不可恢复
  3. 新设备建立新会话 → 新消息可正常加密
预防：密钥云端备份（加密存储） + 新设备密钥同步提示
```

## IM 群组管理与权限完整实现

```python
class IMGroupManagementService:
    """IM 群组管理：创建 + 权限 + 成员管理 + 退出"""

    GROUP_TYPES = {
        "private": {"max_members": 200, "join_type": "invite_only"},
        "public": {"max_members": 500, "join_type": "anyone"},
        "enterprise": {"max_members": 5000, "join_type": "admin_approval"},
    }

    def create_group(self, creator_id, name, group_type="private",
                    description=None):
        """创建群组"""
        config = self.GROUP_TYPES[group_type]
        group_id = str(uuid4())

        self.db.insert("im_groups", {
            "group_id": group_id,
            "name": name,
            "group_type": group_type,
            "description": description,
            "max_members": config["max_members"],
            "join_type": config["join_type"],
            "creator_id": creator_id,
            "created_at": now()
        })

        # 创建者自动成为管理员
        self.db.insert("group_members", {
            "group_id": group_id,
            "user_id": creator_id,
            "role": "admin",
            "joined_at": now()
        })

        return {"group_id": group_id, "name": name}

    def join_group(self, group_id, user_id):
        """加入群组"""
        group = self.db.get_group(group_id)

        # 1. 检查人数限制
        current = self.db.count("group_members", group_id=group_id)
        if current >= group["max_members"]:
            return {"status": "full"}

        # 2. 加入方式
        if group["join_type"] == "invite_only":
            invite = self.db.query_one(
                "SELECT * FROM group_invitations "
                "WHERE group_id = %s AND invitee_id = %s AND status = 'pending'",
                group_id, user_id)
            if not invite:
                return {"status": "need_invitation"}

        elif group["join_type"] == "admin_approval":
            # 提交加入申请
            self.db.insert("group_join_requests", {
                "group_id": group_id, "user_id": user_id,
                "status": "pending", "created_at": now()
            })
            return {"status": "pending_approval"}

        # 3. 加入
        self.db.insert("group_members", {
            "group_id": group_id, "user_id": user_id,
            "role": "member", "joined_at": now()
        })

        # 4. 发送欢迎消息
        self._send_system_message(group_id,
            f"用户 {user_id} 加入了群组")

        return {"status": "joined"}

    def set_group_permission(self, group_id, admin_id, permissions):
        """设置群组权限"""
        # 验证操作者是管理员
        member = self.db.query_one(
            "SELECT role FROM group_members "
            "WHERE group_id = %s AND user_id = %s",
            group_id, admin_id)
        if member["role"] != "admin":
            raise PermissionDeniedError("只有管理员可以设置权限")

        # 权限配置
        self.db.update("im_groups",
            {"permissions": json.dumps(permissions)},
            {"group_id": group_id})

        # 默认权限
        default_perms = {
            "send_message": True,
            "send_image": True,
            "send_file": True,
            "invite_member": False,  # 仅管理员可邀请
            "edit_group_info": False,  # 仅管理员可编辑
            "pin_message": False,
            "mute_member": False,
        }

        return {"permissions": permissions}

    def mute_member(self, group_id, admin_id, target_id,
                   duration_minutes=10):
        """禁言成员"""
        # 验证权限
        member = self.db.query_one(
            "SELECT role FROM group_members "
            "WHERE group_id = %s AND user_id = %s",
            group_id, admin_id)
        if member["role"] != "admin":
            raise PermissionDeniedError("只有管理员可以禁言")

        self.db.insert("group_mutes", {
            "group_id": group_id,
            "user_id": target_id,
            "muted_by": admin_id,
            "duration_minutes": duration_minutes,
            "expires_at": now() + timedelta(minutes=duration_minutes),
            "created_at": now()
        })

        # 设置 Redis 过期键（快速检查）
        self.redis.setex(f"mute:{group_id}:{target_id}",
            duration_minutes * 60, "1")

        # 通知被禁言者
        self._send_system_message(group_id,
            f"用户 {target_id} 荣誉禁言 {duration_minutes} 分钟")

        return {"status": "muted", "duration": duration_minutes}
```

## 异常场景补充

### 场景：群组成员数据不一致

```
触发：Redis 群组成员数与 DB 不一致 → 群组显示 200 人但实际 210 人 → 超过限制
检测：
  1. 群组人数超过 max_members → 不一致
  2. DB 成员数与缓存不一致 → 数据问题
处理：
  1. 定期从 DB 重建群组缓存
  2. 超员时阻止新成员加入
  3. 修正缓存数据
预防：定期重建缓存 + DB 校验 + 加入时双写
```

### 场景：禁言到期但仍然无法发言

```
触发：禁言 10 分钟到期 → Redis 过期键已删除 → 但 DB 记录未更新 → 用户仍被禁言
检测：
  1. 用户投诉无法发言但禁言已到期 → 状态不一致
  2. Redis 无禁言键但查询仍返回禁言 → 逻辑错误
处理：
  1. 发言时优先检查 Redis（过期 = 未禁言）
  2. Redis 无键 → 不禁言（忽略 DB 过期记录）
  3. 定期清理 DB 过期禁言记录
预防：Redis 优先 + 定期清理 DB + 过期自动通知
```

## IM 消息已读回执完整实现

```python
class MessageReadReceiptService:
    """消息已读回执：已读状态 + 群组已读数 + 隐私设置"""

    def mark_as_read(self, user_id, conversation_id, last_read_msg_id):
        """标记消息已读"""
        # 1. 验证消息属于该会话
        msg = self.db.get_message(last_read_msg_id)
        if not msg or msg["conversation_id"] != conversation_id:
            return {"status": "invalid"}

        # 2. 更新已读位置（只向前推进）
        current = self.redis.get(f"read_pos:{conversation_id}:{user_id}")
        if current:
            current_msg_id = current.decode() if isinstance(current, bytes) else current
            if last_read_msg_id <= current_msg_id:
                return {"status": "no_change"}

        self.redis.set(f"read_pos:{conversation_id}:{user_id}", last_read_msg_id)

        # 3. 计算未读数
        total_msgs = self.db.count("messages",
            conversation_id=conversation_id)
        unread = self._count_unread(conversation_id, user_id, last_read_msg_id)

        # 4. 通知发送者（如果隐私设置允许）
        if self._should_send_receipt(user_id, conversation_id):
            partner_id = self._get_conversation_partner(conversation_id, user_id)
            self.notification.send(partner_id,
                f"消息已读", "read_receipt",
                reference_id=last_read_msg_id)

        # 5. 更新会话未读数缓存
        self.redis.hset(f"conv_unread:{conversation_id}", user_id, unread)

        return {"status": "read", "unread_count": unread}

    def get_conversation_unread(self, user_id):
        """获取所有会话未读数"""
        conversations = self.db.query(
            "SELECT conversation_id FROM conversation_members "
            "WHERE user_id = %s AND status = 'active'", user_id)

        result = []
        total_unread = 0
        for conv in conversations:
            conv_id = conv["conversation_id"]
            read_pos = self.redis.get(f"read_pos:{conv_id}:{user_id}")
            read_msg_id = read_pos.decode() if read_pos else None

            if read_msg_id:
                unread = self.db.count("messages",
                    conversation_id=conv_id,
                    id__gt=read_msg_id)
            else:
                unread = self.db.count("messages",
                    conversation_id=conv_id)

            total_unread += unread
            result.append({"conversation_id": conv_id, "unread": unread})

        return {"total_unread": total_unread, "conversations": result}

    def get_group_read_stats(self, conversation_id, msg_id):
        """获取群组消息已读统计"""
        # 获取群成员
        members = self.db.query(
            "SELECT user_id FROM conversation_members "
            "WHERE conversation_id = %s AND status = 'active'",
            conversation_id)

        read_count = 0
        unread_members = []
        for member in members:
            read_pos = self.redis.get(f"read_pos:{conversation_id}:{member['user_id']}")
            if read_pos and read_pos.decode() >= msg_id:
                read_count += 1
            else:
                unread_members.append(member["user_id"])

        return {"msg_id": msg_id, "total_members": len(members),
                "read_count": read_count,
                "unread_count": len(unread_members),
                "unread_members": unread_members}

    def _should_send_receipt(self, user_id, conversation_id):
        """检查隐私设置是否允许发送已读回执"""
        privacy = self.db.query_one(
            "SELECT read_receipt_enabled FROM user_privacy_settings "
            "WHERE user_id = %s", user_id)
        return privacy["read_receipt_enabled"] if privacy else True
```

## 异常场景补充

### 场景：已读回执风暴

```
触发：500 人群组 → 每条消息 500 个已读回执 → 消息服务器过载
检测：
  1. 群组已读回执 QPS > 1000 → 风暴
  2. 服务器 CPU 飙升 → 回执过载
处理：
  1. 群组已读回执批量上报（每 10 秒一次）
  2. 大群（> 100 人）只显示已读人数，不逐一通知
  3. 已读回执限流
预防：批量上报 + 大群简化 + 限流
```

### 场景：已读位置不一致

```
触发：用户在多设备同时使用 → 设备 A 标记已读到 msg100 → 设备 B 仍显示未读 → 状态不一致
检测：
  1. 不同设备已读位置不同 → 多设备同步问题
  2. 用户投诉未读数不准确 → 状态不一致
处理：
  1. 已读位置取最大值（任一设备已读即已读）
  2. 多设备同步已读状态
  3. 定期从服务器拉取最新已读位置
预防：取最大值 + 多设备同步 + 定期拉取
```

## IM 消息搜索与全文检索完整实现

```python
class IMMessageSearchService:
    """IM 消息搜索：全文检索 + 多维过滤 + 上下文展示"""

    def search_messages(self, user_id, query, filters=None, page=1, page_size=20):
        """搜索消息"""
        # 1. 获取用户可搜索的会话列表
        conversations = self.db.query(
            "SELECT conversation_id FROM conversation_members "
            "WHERE user_id = %s AND status = 'active'", user_id)
        conv_ids = [c["conversation_id"] for c in conversations]

        if not conv_ids:
            return {"results": [], "total": 0}

        # 2. 构建 ES 查询
        search_body = {
            "query": {
                "bool": {
                    "must": [
                        {"match": {"content": {"query": query, "analyzer": "ik_max_word"}}},
                        {"terms": {"conversation_id": conv_ids}}
                    ],
                    "filter": []
                }
            },
            "highlight": {
                "fields": {"content": {"fragment_size": 100, "number_of_fragments": 1}},
                "pre_tags": ["<em>"], "post_tags": ["</em>"]
            },
            "sort": [{"timestamp": "desc"}],
            "from": (page - 1) * page_size,
            "size": page_size
        }

        # 3. 添加过滤条件
        if filters:
            if filters.get("sender_id"):
                search_body["query"]["bool"]["filter"].append(
                    {"term": {"sender_id": filters["sender_id"]}})
            if filters.get("conversation_id"):
                search_body["query"]["bool"]["filter"].append(
                    {"term": {"conversation_id": filters["conversation_id"]}})
            if filters.get("message_type"):
                search_body["query"]["bool"]["filter"].append(
                    {"term": {"message_type": filters["message_type"]}})
            if filters.get("start_time") and filters.get("end_time"):
                search_body["query"]["bool"]["filter"].append(
                    {"range": {"timestamp": {
                        "gte": filters["start_time"],
                        "lte": filters["end_time"]}}})
            if filters.get("has_file"):
                search_body["query"]["bool"]["filter"].append(
                    {"exists": {"field": "file_url"}})

        # 4. 执行搜索
        es_result = self.elasticsearch.search(index="im_messages", body=search_body)

        # 5. 格式化结果
        results = []
        for hit in es_result.get("hits", {}).get("hits", []):
            source = hit["_source"]
            highlight = hit.get("highlight", {}).get("content", [source["content"][:100]])

            results.append({
                "message_id": source["message_id"],
                "conversation_id": source["conversation_id"],
                "sender_id": source["sender_id"],
                "content_preview": highlight[0],
                "timestamp": source["timestamp"],
                "message_type": source.get("message_type", "text"),
                "conversation_name": self._get_conversation_name(source["conversation_id"])
            })

        total = es_result.get("hits", {}).get("total", {}).get("value", 0)

        return {"results": results, "total": total,
                "page": page, "page_size": page_size}

    def get_message_context(self, message_id, context_size=5):
        """获取消息上下文（前后各 5 条）"""
        msg = self.db.get_message(message_id)
        if not msg:
            return {"message": None, "context": []}

        # 前后各取 context_size 条
        before = self.db.query(
            "SELECT * FROM messages "
            "WHERE conversation_id = %s AND timestamp < %s "
            "ORDER BY timestamp DESC LIMIT %s",
            msg["conversation_id"], msg["timestamp"], context_size)
        before.reverse()

        after = self.db.query(
            "SELECT * FROM messages "
            "WHERE conversation_id = %s AND timestamp > %s "
            "ORDER BY timestamp ASC LIMIT %s",
            msg["conversation_id"], msg["timestamp"], context_size)

        context = before + [msg] + after

        return {
            "message": {
                "message_id": msg["id"],
                "sender_id": msg["sender_id"],
                "content": msg["content"],
                "timestamp": msg["timestamp"].isoformat()
            },
            "context": [{
                "message_id": m["id"],
                "sender_id": m["sender_id"],
                "content": m["content"],
                "timestamp": m["timestamp"].isoformat()
            } for m in context],
            "context_range": f"{context[0]['timestamp'].isoformat()} ~ {context[-1]['timestamp'].isoformat()}"
        }

    def index_message(self, message):
        """索引消息到 ES"""
        doc = {
            "message_id": message["id"],
            "conversation_id": message["conversation_id"],
            "sender_id": message["sender_id"],
            "content": message["content"],
            "message_type": message.get("type", "text"),
            "timestamp": message["created_at"].isoformat(),
            "file_url": message.get("file_url"),
        }

        self.elasticsearch.index(
            index="im_messages",
            id=message["id"],
            body=doc)

        # 文件内容提取（PDF/Word 等）→ 也加入索引
        if message.get("file_url"):
            file_content = self._extract_file_content(message["file_url"])
            if file_content:
                self.elasticsearch.update(
                    index="im_messages",
                    id=message["id"],
                    body={"doc": {"file_content": file_content[:5000]}})
```

## 异常场景补充

### 场景：消息搜索返回被删除的内容

```
触发：用户删除了消息 → DB 中已标记删除 → ES 中仍存在 → 搜索可看到已删除内容
检测：
  1. 搜索结果中包含已删除消息 → 紧一致性问题
  2. 用户投诉能看到删除的消息 → 紧不一致
处理：
  1. 消息删除时同步删除 ES 索引
  2. 搜索结果返回前过滤（检查消息是否已删除）
  3. 定期同步 DB 删除状态到 ES
预防：删除同步 + 结果过滤 + 定期对齐
```

### 场景：文件内容提取导致搜索延迟

```
触发：用户发送 50MB PDF → 提取文本需要 30 秒 → 消息索引延迟 → 搜索不到
检测：
  1. 新发送文件消息在搜索中不可见 → 索引延迟
  2. 文件索引延迟 > 10 秒 → 提取耗时
处理：
  1. 消息先索引（不含文件内容），文件内容异步提取后更新
  2. 大文件使用 OCR 提取（超时限制 10 秒）
  3. 提取失败 → 只索引文件名
预防：异步提取 + 超时限制 + 降级处理
```

## IM 端到端加密消息完整实现

```python
class E2EEncryptionService:
    """端到端加密：密钥协商 + 消息加密 + 密钥轮换"""

    def generate_key_pair(self, user_id, device_id):
        """生成用户设备密钥对"""
        from cryptography.hazmat.primitives.asymmetric import x25519
        from cryptography.hazmat.primitives import serialization

        # 1. 生成 Identity Key（长期密钥）
        identity_key = x25519.X25519PrivateKey.generate()
        identity_pub = identity_key.public_key()

        # 2. 生成 Signed Pre-Key（中期密钥，30 天轮换）
        signed_prekey = x25519.X25519PrivateKey.generate()
        signed_prekey_pub = signed_prekey.public_key()

        # 3. 用 Identity Key 签名 Signed Pre-Key
        from cryptography.hazmat.primitives.asymmetric import ed25519
        signing_key = ed25519.Ed25519PrivateKey.generate()
        signature = signing_key.sign(
            signed_prekey_pub.public_bytes(
                encoding=serialization.Encoding.Raw,
                format=serialization.PublicFormat.Raw))

        # 4. 生成 One-Time Pre-Keys（一次性密钥，每次使用后删除）
        otp_keys = []
        for _ in range(100):
            otp = x25519.X25519PrivateKey.generate()
            otp_pub = otp.public_key()
            otp_keys.append({
                "key_id": str(uuid4()),
                "public_key": otp_pub.public_bytes(
                    encoding=serialization.Encoding.Raw,
                    format=serialization.PublicFormat.Raw).hex(),
                "private_key": otp.private_bytes(
                    encoding=serialization.Encoding.Raw,
                    format=serialization.PrivateFormat.Raw,
                    encryption_algorithm=serialization.NoEncryption()).hex()
            })

        # 5. 上传公钥到服务器
        self.db.insert("e2e_key_bundles", {
            "user_id": user_id,
            "device_id": device_id,
            "identity_key": identity_pub.public_bytes(
                encoding=serialization.Encoding.Raw,
                format=serialization.PublicFormat.Raw).hex(),
            "signed_prekey": signed_prekey_pub.public_bytes(
                encoding=serialization.Encoding.Raw,
                format=serialization.PublicFormat.Raw).hex(),
            "signed_prekey_signature": signature.hex(),
            "otp_key_count": len(otp_keys),
            "created_at": now()
        })

        # 6. 存储 OTP 密钥
        for otp in otp_keys:
            self.db.insert("e2e_otp_keys", {
                "key_id": otp["key_id"],
                "user_id": user_id,
                "device_id": device_id,
                "public_key": otp["public_key"],
                "used": False,
                "created_at": now()
            })

        return {"user_id": user_id, "device_id": device_id,
                "otp_key_count": len(otp_keys)}

    def encrypt_message(self, sender_id, recipient_id, plaintext):
        """加密消息"""
        # 1. 获取收件人密钥束
        bundle = self.db.query_one(
            "SELECT * FROM e2e_key_bundles "
            "WHERE user_id = %s ORDER BY created_at DESC LIMIT 1",
            recipient_id)

        if not bundle:
            return {"status": "no_key_bundle", "recipient_id": recipient_id}

        # 2. 获取收件人 OTP 密钥
        otp = self.db.query_one(
            "SELECT * FROM e2e_otp_keys "
            "WHERE user_id = %s AND used = False "
            "ORDER BY created_at ASC LIMIT 1", recipient_id)

        if not otp:
            # OTP 用完 → 使用 Signed Pre-Key
            self.alert(f"用户 {recipient_id} OTP 密钥用完，请补充")
            recipient_pub_key = bytes.fromhex(bundle["signed_prekey"])
        else:
            recipient_pub_key = bytes.fromhex(otp["public_key"])
            # 标记 OTP 已使用
            self.db.update("e2e_otp_keys",
                {"used": True, "used_at": now()},
                {"key_id": otp["key_id"]})

        # 3. 使用 AES-256-GCM 加密消息
        from cryptography.hazmat.primitives.ciphers.aead import AESGCM
        import os

        # 生成消息密钥（实际应使用 X3DH 协议协商）
        message_key = os.urandom(32)
        nonce = os.urandom(12)

        aesgcm = AESGCM(message_key)
        ciphertext = aesgcm.encrypt(nonce, plaintext.encode(), None)

        # 4. 用收件人公钥加密消息密钥
        # （简化版，实际使用 X3DH + Double Ratchet）

        encrypted_msg = {
            "message_id": str(uuid4()),
            "sender_id": sender_id,
            "recipient_id": recipient_id,
            "ciphertext": ciphertext.hex(),
            "nonce": nonce.hex(),
            "used_otp_key_id": otp["key_id"] if otp else None,
            "encrypted_at": now().isoformat()
        }

        return encrypted_msg

    def decrypt_message(self, recipient_id, encrypted_msg, private_key_store):
        """解密消息"""
        from cryptography.hazmat.primitives.ciphers.aead import AESGCM

        ciphertext = bytes.fromhex(encrypted_msg["ciphertext"])
        nonce = bytes.fromhex(encrypted_msg["nonce"])

        # 1. 从本地密钥存储获取消息密钥（通过 X3DH 协商得到）
        message_key = private_key_store.get_message_key(
            encrypted_msg["sender_id"],
            encrypted_msg.get("used_otp_key_id"))

        if not message_key:
            return {"status": "decryption_failed", "reason": "no_message_key"}

        # 2. 解密
        aesgcm = AESGCM(message_key)
        try:
            plaintext = aesgcm.decrypt(nonce, ciphertext, None)
            return {"status": "decrypted",
                    "plaintext": plaintext.decode(),
                    "message_id": encrypted_msg["message_id"]}
        except Exception as e:
            return {"status": "decryption_failed", "reason": str(e)}
```

## 异常场景补充

### 场景：OTP 密钥耗尽

```
触发：用户 100 个 OTP 密钥用完 → 新消息无法建立加密会话 → 消息发送失败
检测：
  1. OTP 密钥剩余 < 10 → 低库存预警
  2. 用户设备上线时 OTP 用完 → 发送失败
处理：
  1. 设备上线时自动补充 OTP 密钥
  2. OTP 不足时回退到 Signed Pre-Key
  3. 通知用户重新生成密钥
预防：自动补充 + 回退机制 + 低库存预警
```

### 场景：用户更换设备后无法解密历史消息

```
触发：用户换新手机 → 新设备没有旧私钥 → 无法解密历史消息 → 消息丢失
检测：
  1. 用户在新设备上解密失败 → 无密钥
  2. 历史消息全部不可读 → 设备更换
处理：
  1. 提供密钥导出/导入功能（加密备份）
  2. 多设备密钥同步
  3. 无法解密 → 标记为"加密消息，此设备无法读取"
预防：密钥备份 + 多设备同步 + 降级显示
```

## IM 群聊管理与消息分发完整实现

```python
class GroupChatService:
    """群聊管理：群组创建 → 成员管理 → 消息分发 → 权限控制"""

    GROUP_ROLES = {
        "owner": {"can_invite": True, "can_remove": True,
                  "can_set_admin": True, "can_dismiss": True,
                  "can_update_info": True, "max_groups": 50},
        "admin": {"can_invite": True, "can_remove": True,
                  "can_set_admin": False, "can_dismiss": False,
                  "can_update_info": True, "max_groups": 100},
        "member": {"can_invite": False, "can_remove": False,
                   "can_set_admin": False, "can_dismiss": False,
                   "can_update_info": False, "max_groups": 500},
    }

    MAX_GROUP_SIZE = 500

    def create_group(self, owner_id, group_name, group_type="public"):
        """创建群组"""
        # 1. 检查用户群组数量限制
        current_groups = self.db.count("group_members",
            user_id=owner_id, role="owner")

        if current_groups >= self.GROUP_ROLES["owner"]["max_groups"]:
            return {"status": "limit_reached",
                    "max": self.GROUP_ROLES["owner"]["max_groups"]}

        # 2. 创建群组
        group_id = str(uuid4())
        self.db.insert("groups", {
            "group_id": group_id,
            "name": group_name,
            "type": group_type,  # public / private / restricted
            "owner_id": owner_id,
            "member_count": 1,
            "max_members": self.MAX_GROUP_SIZE,
            "created_at": now()
        })

        # 3. 添加创建者为群主
        self.db.insert("group_members", {
            "group_id": group_id,
            "user_id": owner_id,
            "role": "owner",
            "joined_at": now()
        })

        # 4. 初始化群组消息队列
        self.redis.set(f"group_last_msg:{group_id}", "0")
        self.redis.set(f"group_member_count:{group_id}", "1")

        return {"group_id": group_id, "name": group_name,
                "owner_id": owner_id}

    def invite_member(self, group_id, inviter_id, invitee_id):
        """邀请成员"""
        # 1. 检查邀请者权限
        member = self.db.query_one(
            "SELECT * FROM group_members "
            "WHERE group_id = %s AND user_id = %s",
            group_id, inviter_id)

        if not member:
            return {"status": "not_member"}

        role_config = self.GROUP_ROLES.get(member["role"], {})
        if not role_config.get("can_invite"):
            return {"status": "no_permission"}

        # 2. 检查群组人数上限
        current_count = int(self.redis.get(f"group_member_count:{group_id}") or 0)
        if current_count >= self.MAX_GROUP_SIZE:
            return {"status": "group_full", "max": self.MAX_GROUP_SIZE}

        # 3. 检查是否已在群中
        existing = self.db.query_one(
            "SELECT * FROM group_members "
            "WHERE group_id = %s AND user_id = %s",
            group_id, invitee_id)

        if existing:
            return {"status": "already_member"}

        # 4. 私有群需审批
        group = self.db.get_group(group_id)
        if group["type"] == "private":
            self.db.insert("group_invitations", {
                "invitation_id": str(uuid4()),
                "group_id": group_id,
                "inviter_id": inviter_id,
                "invitee_id": invitee_id,
                "status": "pending",
                "created_at": now()
            })
            self.notification.send(invitee_id,
                f"您被邀请加入群组 {group['name']}，请确认")
            return {"status": "invitation_sent"}

        # 5. 直接加入
        self._add_member(group_id, invitee_id, "member")
        return {"status": "joined", "group_id": group_id}

    def _add_member(self, group_id, user_id, role):
        """添加成员"""
        self.db.insert("group_members", {
            "group_id": group_id,
            "user_id": user_id,
            "role": role,
            "joined_at": now()
        })

        self.redis.incr(f"group_member_count:{group_id}")

        # 更新群组成员列表缓存
        self.redis.sadd(f"group_members_set:{group_id}", user_id)

        # 加入群聊消息频道
        self.pubsub.subscribe(f"group_channel:{group_id}", user_id)

    def remove_member(self, group_id, remover_id, target_user_id):
        """移除成员"""
        # 1. 权限检查
        remover = self.db.query_one(
            "SELECT * FROM group_members "
            "WHERE group_id = %s AND user_id = %s",
            group_id, remover_id)

        target = self.db.query_one(
            "SELECT * FROM group_members "
            "WHERE group_id = %s AND user_id = %s",
            group_id, target_user_id)

        if not remover or not target:
            return {"status": "not_found"}

        # 不能移除群主
        if target["role"] == "owner":
            return {"status": "cannot_remove_owner"}

        # 管理员只能移除普通成员
        if target["role"] == "admin" and remover["role"] != "owner":
            return {"status": "insufficient_permission"}

        # 2. 移除成员
        self.db.delete("group_members",
            group_id=group_id, user_id=target_user_id)

        self.redis.decr(f"group_member_count:{group_id}")
        self.redis.srem(f"group_members_set:{group_id}", target_user_id)

        # 3. 通知被移除者
        self.notification.send(target_user_id,
            f"您已被移出群组 {self.db.get_group(group_id)['name']}")

        # 4. 系统消息通知群组
        self._send_system_message(group_id,
            f"用户 {target_user_id} 已被移出群组")

        return {"status": "removed"}

    def distribute_group_message(self, group_id, sender_id, message):
        """分发群聊消息"""
        # 1. 检查发送者是否在群中
        if not self.redis.sismember(f"group_members_set:{group_id}", sender_id):
            return {"status": "not_member"}

        # 2. 消息存储
        msg_id = str(uuid4())
        self.db.insert("group_messages", {
            "msg_id": msg_id,
            "group_id": group_id,
            "sender_id": sender_id,
            "content": message["content"],
            "msg_type": message.get("type", "text"),
            "created_at": now()
        })

        # 3. 更新最新消息 ID
        self.redis.set(f"group_last_msg:{group_id}", msg_id)

        # 4. 分发到所有成员
        members = self.redis.smembers(f"group_members_set:{group_id}")

        for member_id in members:
            uid = member_id.decode() if isinstance(member_id, bytes) else member_id

            # 在线成员 → 即时推送
            if self.redis.sismember("online_users", uid):
                self.push_service.send(uid, {
                    "msg_id": msg_id,
                    "group_id": group_id,
                    "sender_id": sender_id,
                    "content": message["content"],
                    "type": message.get("type", "text")
                })
            # 离线成员 → 离线消息队列
            else:
                self.redis.rpush(f"offline_msgs:{uid}", json.dumps({
                    "msg_id": msg_id,
                    "group_id": group_id,
                    "sender_id": sender_id,
                    "content": message["content"],
                    "created_at": now().isoformat()
                }))

        return {"msg_id": msg_id, "delivered_to": len(members)}

    def _send_system_message(self, group_id, content):
        """发送系统消息"""
        self.db.insert("group_messages", {
            "msg_id": str(uuid4()),
            "group_id": group_id,
            "sender_id": "system",
            "content": content,
            "msg_type": "system",
            "created_at": now()
        })
```

## 异常场景补充

### 场景：群聊消息分发延迟

```
触发：大群 500 人 → 消息需推送 500 次 → 推送队列堆积 → 消息延迟 5 秒 → 体验差
检测：
  1. 群消息推送延迟 > 2 秒 → 分发慢
  2. 推送队列深度 > 1000 → 堆积
处理：
  1. 大群消息使用广播而非逐人推送
  2. 批量推送（每批 50 人）
  3. 离线成员延迟推送
预防：广播推送 + 批量 + 离线延迟
```

### 场景：群组权限被滥用

```
触发：管理员频繁踢人 → 群成员流失 → 群氛围恶化 → 用户投诉
检测：
  1. 群成员数快速下降 → 异常踢人
  2. 单个管理员踢人频率 > 10 次/小时 → 权限滥用
处理：
  1. 踢人操作需群主确认
  2. 被踢成员申诉机制
  3. 管理员踢人频率限制
预防：确认机制 + 申诉 + 频率限制
```

## IM 消息搜索与全文检索完整实现

```python
import re
import time
import hashlib
import math
from datetime import datetime, timedelta
from collections import defaultdict, Counter
from typing import List, Dict, Optional, Tuple, Set, Any


class MessageSearchService:
    """IM消息搜索与全文检索服务，支持CJK分词、n-gram索引和实时搜索建议"""

    CJK_DICT = {
        "消息": 2, "搜索": 2, "全文": 2, "检索": 2, "即时": 2, "通讯": 2,
        "系统": 2, "用户": 2, "发送": 2, "接收": 2, "通知": 2, "群组": 2,
        "会话": 2, "聊天": 2, "记录": 2, "文件": 2, "图片": 2, "视频": 2,
        "语音": 2, "会议": 2, "日程": 2, "任务": 2, "项目": 2, "文档": 2,
        "审批": 2, "公告": 2, "打卡": 2, "请假": 2, "报销": 2, "合同": 2,
        "客户": 2, "订单": 2, "产品": 2, "服务": 2, "支持": 2, "问题": 2,
        "解决": 2, "方案": 2, "需求": 2, "开发": 2, "测试": 2, "部署": 2,
        "上线": 2, "版本": 2, "更新": 2, "修复": 2, "优化": 2, "性能": 2,
        "安全": 2, "权限": 2, "配置": 2, "管理": 2, "监控": 2, "报警": 2,
        "数据": 2, "分析": 2, "报告": 2, "统计": 2, "平台": 2, "功能": 2,
        "企业": 2, "团队": 2, "部门": 2, "成员": 2, "负责人": 3, "管理员": 3,
        "操作": 2, "确认": 2, "取消": 2, "提交": 2, "审核": 2, "通过": 2,
        "拒绝": 2, "删除": 2, "修改": 2, "查看": 2, "下载": 2, "上传": 2,
        "分享": 2, "转发": 2, "回复": 2, "引用": 2, "收藏": 2, "标记": 2,
        "已读": 2, "未读": 2, "在线": 2, "离线": 2, "忙碌": 2, "离开": 2,
    }

    MAX_DICT_WORD_LEN = 4

    def __init__(self, es_client, db_client, cache_client, config: Dict[str, Any]):
        self.es_client = es_client
        self.db_client = db_client
        self.cache_client = cache_client
        self.config = config
        self.index_name = config.get("es_index_name", "im_messages")
        self.batch_size = config.get("index_batch_size", 1000)
        self.suggestion_cache_ttl = config.get("suggestion_cache_ttl", 3600)
        self.term_freq_cache = {}
        self.user_search_history = defaultdict(list)
        self.global_trending_terms = []
        self.trending_last_updated = 0
        self.index_metadata = {}

    def _is_cjk_character(self, char: str) -> bool:
        code_point = ord(char)
        if 0x4E00 <= code_point <= 0x9FFF:
            return True
        if 0x3400 <= code_point <= 0x4DBF:
            return True
        if 0x2E80 <= code_point <= 0x2EFF:
            return True
        if 0xF900 <= code_point <= 0xFAFF:
            return True
        return False

    def _extract_mentions(self, content: str) -> List[str]:
        mention_pattern = re.compile(r'@(\w+)')
        matches = mention_pattern.findall(content)
        unique_mentions = list(set(matches))
        return unique_mentions

    def _extract_hashtags(self, content: str) -> List[str]:
        hashtag_pattern = re.compile(r'#(\w+)')
        matches = hashtag_pattern.findall(content)
        unique_hashtags = list(set(matches))
        return unique_hashtags

    def _extract_text_content(self, message: Dict[str, Any]) -> str:
        message_type = message.get("message_type", "text")
        if message_type == "text":
            return message.get("content", "")
        elif message_type == "image":
            alt_text = message.get("alt_text", "")
            caption = message.get("caption", "")
            file_name = message.get("file_name", "")
            parts = [p for p in [alt_text, caption, file_name] if p]
            return " ".join(parts) if parts else ""
        elif message_type == "file":
            file_name = message.get("file_name", "")
            description = message.get("description", "")
            parts = [p for p in [file_name, description] if p]
            return " ".join(parts) if parts else ""
        elif message_type == "voice":
            transcription = message.get("transcription", "")
            return transcription
        elif message_type == "video":
            title = message.get("title", "")
            description = message.get("description", "")
            parts = [p for p in [title, description] if p]
            return " ".join(parts) if parts else ""
        elif message_type == "rich_text":
            raw_html = message.get("content", "")
            clean_text = re.sub(r'<[^>]+>', ' ', raw_html)
            clean_text = re.sub(r'\s+', ' ', clean_text).strip()
            return clean_text
        elif message_type == "card":
            card_title = message.get("card_title", "")
            card_content = message.get("card_content", "")
            parts = [p for p in [card_title, card_content] if p]
            return " ".join(parts) if parts else ""
        else:
            return message.get("content", "")

    def _forward_maximum_matching(self, text: str) -> List[str]:
        tokens = []
        i = 0
        n = len(text)
        while i < n:
            matched = False
            for length in range(min(self.MAX_DICT_WORD_LEN, n - i), 1, -1):
                candidate = text[i:i + length]
                if candidate in self.CJK_DICT:
                    tokens.append(candidate)
                    i += length
                    matched = True
                    break
            if not matched:
                if self._is_cjk_character(text[i]):
                    tokens.append(text[i])
                elif text[i].isalnum():
                    word_start = i
                    while i < n and (text[i].isalnum() or text[i] == '_'):
                        i += 1
                    tokens.append(text[word_start:i])
                    continue
                else:
                    if text[i] not in (' ', '\t', '\n', '\r'):
                        tokens.append(text[i])
                i += 1
        return tokens

    def _tokenize(self, text: str) -> List[str]:
        if not text:
            return []
        cjk_segments = []
        current_segment = ""
        for char in text:
            if self._is_cjk_character(char):
                current_segment += char
            else:
                if current_segment:
                    cjk_segments.append(current_segment)
                    current_segment = ""
        if current_segment:
            cjk_segments.append(current_segment)

        tokens = []
        text_lower = text.lower()
        ascii_words = re.findall(r'[a-zA-Z0-9_]+', text_lower)
        tokens.extend(ascii_words)

        for segment in cjk_segments:
            segment_tokens = self._forward_maximum_matching(segment)
            tokens.extend(segment_tokens)

        return tokens

    def _generate_ngrams(self, tokens: List[str], min_n: int = 1, max_n: int = 3) -> List[str]:
        ngrams = []
        for n in range(min_n, min(max_n + 1, len(tokens) + 1)):
            for i in range(len(tokens) - n + 1):
                ngram = "".join(tokens[i:i + n])
                ngrams.append(ngram)
        return ngrams

    def _update_term_frequency_cache(self, conversation_id: str, tokens: List[str]) -> None:
        if conversation_id not in self.term_freq_cache:
            self.term_freq_cache[conversation_id] = Counter()
        for token in tokens:
            self.term_freq_cache[conversation_id][token] += 1
        cache_key = f"term_freq:{conversation_id}"
        top_terms = dict(self.term_freq_cache[conversation_id].most_common(200))
        self.cache_client.set(cache_key, top_terms, ttl=86400)

    def index_message(self, message: Dict[str, Any]) -> Dict[str, Any]:
        message_id = message.get("message_id")
        if not message_id:
            return {"status": "error", "reason": "missing message_id"}

        text_content = self._extract_text_content(message)
        mentions = self._extract_mentions(text_content)
        hashtags = self._extract_hashtags(text_content)
        tokens = self._tokenize(text_content)
        ngrams = self._generate_ngrams(tokens)

        conversation_id = message.get("conversation_id", "unknown")

        doc = {
            "message_id": message_id,
            "content": text_content,
            "content_tokens": tokens,
            "content_ngrams": ngrams,
            "sender_id": message.get("sender_id"),
            "conversation_id": conversation_id,
            "timestamp": message.get("timestamp", datetime.utcnow().isoformat()),
            "mentions": mentions,
            "hashtags": hashtags,
            "message_type": message.get("message_type", "text"),
            "indexed_at": datetime.utcnow().isoformat(),
        }

        es_result = self.es_client.index(
            index=self.index_name,
            id=message_id,
            body=doc,
            refresh=False,
        )

        self._update_term_frequency_cache(conversation_id, tokens)

        if not self.user_search_history.get(message.get("sender_id")):
            self.user_search_history[message.get("sender_id", "")] = []

        return {
            "status": "indexed",
            "message_id": message_id,
            "es_result": es_result.get("result", "unknown"),
            "tokens_count": len(tokens),
            "ngrams_count": len(ngrams),
            "mentions": mentions,
            "hashtags": hashtags,
        }

    def _check_user_conversation_access(self, user_id: str, conversation_id: str) -> bool:
        cache_key = f"user_conv_access:{user_id}:{conversation_id}"
        cached = self.cache_client.get(cache_key)
        if cached is not None:
            return cached
        access_record = self.db_client.query(
            "SELECT 1 FROM conversation_members WHERE user_id = %s AND conversation_id = %s AND is_active = 1",
            (user_id, conversation_id),
        )
        has_access = len(access_record) > 0
        self.cache_client.set(cache_key, has_access, ttl=300)
        return has_access

    def _get_user_accessible_conversations(self, user_id: str) -> List[str]:
        cache_key = f"user_convs:{user_id}"
        cached = self.cache_client.get(cache_key)
        if cached is not None:
            return cached
        records = self.db_client.query(
            "SELECT conversation_id FROM conversation_members WHERE user_id = %s AND is_active = 1",
            (user_id,),
        )
        conversation_ids = [r["conversation_id"] for r in records]
        self.cache_client.set(cache_key, conversation_ids, ttl=300)
        return conversation_ids

    def _build_es_query(
        self,
        user_id: str,
        query: str,
        filters: Optional[Dict[str, Any]] = None,
        cursor: Optional[str] = None,
        page_size: int = 20,
    ) -> Dict[str, Any]:
        accessible_conversations = self._get_user_accessible_conversations(user_id)
        if not accessible_conversations:
            return {"query": {"bool": {"must": [{"match_none": {}}]}}}

        must_clauses = [
            {"terms": {"conversation_id": accessible_conversations}},
        ]
        should_clauses = []
        must_not_clauses = []

        query_tokens = self._tokenize(query)
        if query_tokens:
            combined_query = " ".join(query_tokens)
            must_clauses.append({
                "bool": {
                    "should": [
                        {"match": {"content": {"query": combined_query, "boost": 2.0}}},
                        {"match": {"content_ngrams": {"query": combined_query, "boost": 0.5}}},
                        {"match": {"content_tokens": {"query": combined_query, "boost": 1.5}}},
                    ],
                    "minimum_should_match": 1,
                }
            })

        exact_match = re.search(r'"([^"]+)"', query)
        if exact_match:
            must_clauses.append({"match_phrase": {"content": exact_match.group(1)}})

        mention_match = re.search(r'@(\w+)', query)
        if mention_match:
            must_clauses.append({"term": {"mentions": mention_match.group(1)}})

        hashtag_match = re.search(r'#(\w+)', query)
        if hashtag_match:
            must_clauses.append({"term": {"hashtags": hashtag_match.group(1)}})

        if filters:
            if "date_range" in filters:
                date_range = filters["date_range"]
                range_clause = {"range": {"timestamp": {}}}
                if "start" in date_range:
                    range_clause["range"]["timestamp"]["gte"] = date_range["start"]
                if "end" in date_range:
                    range_clause["range"]["timestamp"]["lte"] = date_range["end"]
                must_clauses.append(range_clause)

            if "sender_id" in filters:
                if isinstance(filters["sender_id"], list):
                    must_clauses.append({"terms": {"sender_id": filters["sender_id"]}})
                else:
                    must_clauses.append({"term": {"sender_id": filters["sender_id"]}})

            if "conversation_id" in filters:
                conv_ids = filters["conversation_id"] if isinstance(filters["conversation_id"], list) else [filters["conversation_id"]]
                accessible_filtered = [c for c in conv_ids if c in accessible_conversations]
                if accessible_filtered:
                    must_clauses[0] = {"terms": {"conversation_id": accessible_filtered}}
                else:
                    must_clauses.append({"match_none": {}})

            if "message_type" in filters:
                msg_types = filters["message_type"] if isinstance(filters["message_type"], list) else [filters["message_type"]]
                must_clauses.append({"terms": {"message_type": msg_types}})

            if "exclude_sender" in filters:
                exclude_ids = filters["exclude_sender"] if isinstance(filters["exclude_sender"], list) else [filters["exclude_sender"]]
                must_not_clauses.append({"terms": {"sender_id": exclude_ids}})

            if "has_mentions" in filters and filters["has_mentions"]:
                must_clauses.append({"exists": {"field": "mentions"}})

            if "has_hashtags" in filters and filters["has_hashtags"]:
                must_clauses.append({"exists": {"field": "hashtags"}})

        es_query = {
            "query": {
                "bool": {
                    "must": must_clauses,
                    "should": should_clauses if should_clauses else None,
                    "must_not": must_not_clauses if must_not_clauses else None,
                }
            },
            "size": page_size + 1,
            "sort": [
                {"_score": {"order": "desc"}},
                {"timestamp": {"order": "desc"}},
            ],
            "highlight": {
                "fields": {
                    "content": {
                        "fragment_size": 200,
                        "number_of_fragments": 3,
                        "pre_tags": ["<em class='highlight'>"],
                        "post_tags": ["</em>"],
                    },
                    "content_tokens": {
                        "fragment_size": 150,
                        "number_of_fragments": 2,
                        "pre_tags": ["<em class='highlight'>"],
                        "post_tags": ["</em>"],
                    },
                },
            },
        }

        if cursor:
            try:
                cursor_data = self._decode_cursor(cursor)
                es_query["search_after"] = cursor_data["sort_values"]
            except Exception:
                pass

        es_query["query"]["bool"] = {
            k: v for k, v in es_query["query"]["bool"].items() if v is not None
        }

        return es_query

    def _decode_cursor(self, cursor: str) -> Dict[str, Any]:
        import base64
        import json
        decoded = base64.b64decode(cursor).decode("utf-8")
        return json.loads(decoded)

    def _encode_cursor(self, sort_values: List[Any]) -> str:
        import base64
        import json
        cursor_data = {"sort_values": sort_values}
        encoded = json.dumps(cursor_data).encode("utf-8")
        return base64.b64encode(encoded).decode("utf-8")

    def search_messages(
        self,
        user_id: str,
        query: str,
        filters: Optional[Dict[str, Any]] = None,
        cursor: Optional[str] = None,
        page_size: int = 20,
    ) -> Dict[str, Any]:
        if not query or len(query.strip()) == 0:
            return {"status": "error", "reason": "empty query", "results": [], "total": 0}

        es_query = self._build_es_query(user_id, query, filters, cursor, page_size)

        try:
            es_response = self.es_client.search(index=self.index_name, body=es_query)
        except Exception as e:
            return {
                "status": "error",
                "reason": f"elasticsearch error: {str(e)}",
                "results": [],
                "total": 0,
            }

        hits = es_response.get("hits", {}).get("hits", [])
        total = es_response.get("hits", {}).get("total", {}).get("value", 0)

        results = []
        for hit in hits[:page_size]:
            source = hit.get("_source", {})
            highlights = hit.get("highlight", {})
            result = {
                "message_id": source.get("message_id"),
                "content": source.get("content"),
                "sender_id": source.get("sender_id"),
                "conversation_id": source.get("conversation_id"),
                "timestamp": source.get("timestamp"),
                "message_type": source.get("message_type"),
                "mentions": source.get("mentions", []),
                "hashtags": source.get("hashtags", []),
                "relevance_score": hit.get("_score", 0),
                "highlights": highlights.get("content", highlights.get("content_tokens", [])),
            }
            results.append(result)

        results.sort(key=lambda x: x["relevance_score"], reverse=True)

        next_cursor = None
        if len(hits) > page_size:
            last_hit = hits[page_size - 1]
            sort_values = last_hit.get("sort", [])
            next_cursor = self._encode_cursor(sort_values)

        self.user_search_history[user_id].append({
            "query": query,
            "timestamp": datetime.utcnow().isoformat(),
            "result_count": total,
        })
        if len(self.user_search_history[user_id]) > 100:
            self.user_search_history[user_id] = self.user_search_history[user_id][-100:]

        return {
            "status": "success",
            "results": results,
            "total": total,
            "next_cursor": next_cursor,
            "has_more": len(hits) > page_size,
        }

    def _get_user_recent_searches(self, user_id: str, limit: int = 20) -> List[Dict[str, Any]]:
        cache_key = f"recent_searches:{user_id}"
        cached = self.cache_client.get(cache_key)
        if cached:
            return cached[:limit]
        history = self.user_search_history.get(user_id, [])
        return history[-limit:]

    def _get_conversation_top_terms(self, user_id: str, limit: int = 50) -> List[Tuple[str, int]]:
        accessible_conversations = self._get_user_accessible_conversations(user_id)
        combined_terms = Counter()
        for conv_id in accessible_conversations[:20]:
            cache_key = f"term_freq:{conv_id}"
            cached = self.cache_client.get(cache_key)
            if cached:
                for term, freq in cached.items():
                    combined_terms[term] += freq
            elif conv_id in self.term_freq_cache:
                for term, freq in self.term_freq_cache[conv_id].items():
                    combined_terms[term] += freq
        return combined_terms.most_common(limit)

    def _update_global_trending(self) -> List[Tuple[str, int]]:
        now = time.time()
        if now - self.trending_last_updated < 3600:
            return self.global_trending_terms
        try:
            trending_result = self.es_client.search(
                index=self.index_name,
                body={
                    "size": 0,
                    "aggs": {
                        "trending_terms": {
                            "terms": {
                                "field": "content_tokens",
                                "size": 100,
                                "order": {"_count": "desc"},
                            }
                        }
                    },
                    "query": {
                        "range": {
                            "timestamp": {
                                "gte": "now-24h/h",
                            }
                        }
                    },
                },
            )
            buckets = trending_result.get("aggregations", {}).get("trending_terms", {}).get("buckets", [])
            self.global_trending_terms = [(b["key"], b["doc_count"]) for b in buckets]
            self.trending_last_updated = now
        except Exception:
            pass
        return self.global_trending_terms

    def get_search_suggestions(self, user_id: str, partial_query: str) -> Dict[str, Any]:
        if not partial_query or len(partial_query.strip()) < 1:
            return {"status": "success", "suggestions": []}

        partial_lower = partial_query.lower().strip()
        scored_suggestions = {}

        recent_searches = self._get_user_recent_searches(user_id, limit=20)
        for search_entry in recent_searches:
            past_query = search_entry.get("query", "")
            if past_query.lower().startswith(partial_lower):
                recency_hours = (datetime.utcnow() - datetime.fromisoformat(search_entry["timestamp"])).total_seconds() / 3600
                recency_score = max(0, 1.0 - recency_hours / 168)
                frequency_bonus = sum(1 for s in recent_searches if s["query"].lower() == past_query.lower()) * 0.1
                scored_suggestions[past_query] = scored_suggestions.get(past_query, 0) + recency_score * 3.0 + frequency_bonus

        conv_top_terms = self._get_conversation_top_terms(user_id, limit=50)
        for term, freq in conv_top_terms:
            if term.lower().startswith(partial_lower):
                relevance_score = math.log1p(freq) / 10.0
                scored_suggestions[term] = scored_suggestions.get(term, 0) + relevance_score * 2.0

        global_trending = self._update_global_trending()
        for term, count in global_trending:
            if term.lower().startswith(partial_lower):
                trending_score = math.log1p(count) / 20.0
                scored_suggestions[term] = scored_suggestions.get(term, 0) + trending_score * 1.0

        partial_tokens = self._tokenize(partial_lower)
        for term in scored_suggestions:
            term_tokens = self._tokenize(term.lower())
            common_tokens = set(partial_tokens) & set(term_tokens)
            if common_tokens:
                overlap_ratio = len(common_tokens) / max(len(partial_tokens), 1)
                scored_suggestions[term] += overlap_ratio * 0.5

        sorted_suggestions = sorted(scored_suggestions.items(), key=lambda x: x[1], reverse=True)
        top_10 = [s[0] for s in sorted_suggestions[:10]]

        return {
            "status": "success",
            "suggestions": top_10,
            "partial_query": partial_query,
        }

    def rebuild_search_index(self, conversation_id: str) -> Dict[str, Any]:
        index_lock_key = f"index_rebuild_lock:{conversation_id}"
        lock_acquired = self.cache_client.set_if_not_exists(index_lock_key, "locked", ttl=1800)
        if not lock_acquired:
            return {
                "status": "error",
                "reason": "index rebuild already in progress for this conversation",
                "conversation_id": conversation_id,
            }

        try:
            old_alias = f"{self.index_name}_{conversation_id}"
            try:
                self.es_client.indices.delete(index=old_alias, ignore=[404])
            except Exception:
                pass

            mapping = {
                "mappings": {
                    "properties": {
                        "message_id": {"type": "keyword"},
                        "content": {"type": "text", "analyzer": "standard"},
                        "content_tokens": {"type": "text", "analyzer": "standard"},
                        "content_ngrams": {"type": "text", "analyzer": "standard"},
                        "sender_id": {"type": "keyword"},
                        "conversation_id": {"type": "keyword"},
                        "timestamp": {"type": "date"},
                        "mentions": {"type": "keyword"},
                        "hashtags": {"type": "keyword"},
                        "message_type": {"type": "keyword"},
                        "indexed_at": {"type": "date"},
                    }
                }
            }
            new_index_name = f"{self.index_name}_{conversation_id}_{int(time.time())}"
            self.es_client.indices.create(index=new_index_name, body=mapping, ignore=[400])

            total_db_count = self.db_client.query(
                "SELECT COUNT(*) as cnt FROM messages WHERE conversation_id = %s",
                (conversation_id,),
            )[0]["cnt"]

            indexed_count = 0
            offset = 0
            failed_batches = 0

            while offset < total_db_count:
                messages = self.db_client.query(
                    "SELECT message_id, content, sender_id, conversation_id, timestamp, "
                    "message_type, alt_text, caption, file_name, description, transcription, "
                    "title, card_title, card_content "
                    "FROM messages WHERE conversation_id = %s ORDER BY timestamp ASC "
                    "LIMIT %s OFFSET %s",
                    (conversation_id, self.batch_size, offset),
                )

                if not messages:
                    break

                bulk_body = []
                for msg in messages:
                    text_content = self._extract_text_content(msg)
                    mentions = self._extract_mentions(text_content)
                    hashtags = self._extract_hashtags(text_content)
                    tokens = self._tokenize(text_content)
                    ngrams = self._generate_ngrams(tokens)

                    doc = {
                        "message_id": msg["message_id"],
                        "content": text_content,
                        "content_tokens": tokens,
                        "content_ngrams": ngrams,
                        "sender_id": msg.get("sender_id"),
                        "conversation_id": conversation_id,
                        "timestamp": msg.get("timestamp", datetime.utcnow().isoformat()),
                        "mentions": mentions,
                        "hashtags": hashtags,
                        "message_type": msg.get("message_type", "text"),
                        "indexed_at": datetime.utcnow().isoformat(),
                    }

                    bulk_body.append({"index": {"_index": new_index_name, "_id": msg["message_id"]}})
                    bulk_body.append(doc)
                    indexed_count += 1

                if bulk_body:
                    try:
                        self.es_client.bulk(body=bulk_body, refresh=False)
                    except Exception as bulk_error:
                        failed_batches += 1
                        if failed_batches > 5:
                            return {
                                "status": "error",
                                "reason": f"too many bulk indexing failures: {str(bulk_error)}",
                                "conversation_id": conversation_id,
                                "indexed_count": indexed_count,
                            }

                offset += self.batch_size

            self.es_client.indices.refresh(index=new_index_name)

            es_count_result = self.es_client.count(index=new_index_name)
            es_doc_count = es_count_result.get("count", 0)

            if es_doc_count != total_db_count:
                discrepancy = abs(total_db_count - es_doc_count)
                if discrepancy > total_db_count * 0.01:
                    return {
                        "status": "warning",
                        "reason": f"document count mismatch: db={total_db_count}, es={es_doc_count}",
                        "conversation_id": conversation_id,
                        "db_count": total_db_count,
                        "es_count": es_doc_count,
                        "indexed_count": indexed_count,
                    }

            self.es_client.indices.put_alias(index=new_index_name, name=old_alias)

            self.index_metadata[conversation_id] = {
                "index_name": new_index_name,
                "alias": old_alias,
                "rebuild_time": datetime.utcnow().isoformat(),
                "doc_count": es_doc_count,
                "db_count": total_db_count,
                "status": "active",
            }
            cache_key = f"index_metadata:{conversation_id}"
            self.cache_client.set(cache_key, self.index_metadata[conversation_id], ttl=86400)

            self.term_freq_cache.pop(conversation_id, None)

            return {
                "status": "success",
                "conversation_id": conversation_id,
                "db_count": total_db_count,
                "es_count": es_doc_count,
                "indexed_count": indexed_count,
                "index_name": new_index_name,
                "failed_batches": failed_batches,
            }
        finally:
            self.cache_client.delete(index_lock_key)
```

## 异常场景补充

### 场景：搜索索引与数据库不一致
```
trigger: Elasticsearch集群因磁盘满导致部分分片写入失败，或bulk索引请求超时，导致数据库有100条消息而ES只索引了92条。用户搜索时发现最近发送的重要消息无法被搜到，投诉率上升。
detection: 1) 定时任务每小时对比每个会话的ES文档数与数据库消息数，差异超过0.5%则告警；2) 在index_message方法中记录每条消息的索引状态到Redis集合，定时扫描未索引消息；3) 用户搜索时如果结果少于预期，前端显示"可能未包含全部消息"提示。
handling: 1) 立即将告警通知搜索运维团队，标记受影响的会话；2) 对不一致的会话执行rebuild_search_index，使用批量重索引修复缺失文档；3) 在重索引期间，搜索请求降级为先查数据库再补充ES结果；4) 修复完成后验证文档数一致，关闭告警；5) 排查ES写入失败根因（磁盘扩容、增加副本、调整bulk超时）。
prevention: 1) index_message使用confirm模式，写入后立即读取验证文档存在；2) 实现消息索引的可靠队列（Kafka），保证at-least-once投递；3) ES集群配置磁盘水位告警，水位达85%时提前扩容；4) 每日自动执行全量一致性校验，自动修复小范围不一致。
```

### 场景：CJK 分词导致搜索结果不准确
```
trigger: 用户搜索"上海自来水"时，前向最大匹配将其分词为"上海"+"自来水"，导致单独搜索"海自"时无法匹配。用户搜索"长城汽车"被分为"长城"+"汽车"，但"长城"既是地名也是品牌，导致搜索"长城"时返回大量无关的北京长城旅游消息。中文未登录词（如新品牌"小鹏"）无法正确分词，搜索丢失大量相关结果。
detection: 1) 监控搜索无结果率（zero-result rate），超过5%则告警；2) 对搜索结果进行点击率分析，点击率低于10%说明相关性差；3) 用户反馈"搜索不到"的工单监控，关键词提取后定位分词问题；4) A/B测试新分词策略，对比搜索满意度评分。
handling: 1) 紧急将搜索查询词同时使用多种分词策略（FMM+BMM+单字分词），取并集扩大召回；2) 对未登录词建立临时补丁词典，从搜索无结果日志中提取高频词；3) 将"长城汽车"等歧义词添加到组合词表，搜索时同时匹配"长城汽车"和"长城"+"汽车"；4) 24小时内发布词典更新，重建受影响会话的索引。
prevention: 1) 分词器采用FMM+BMM双向匹配+歧义消解，选取切分路径最优者；2) 建立领域词典持续更新机制，每周从新消息中提取高频组合词；3) 索引时同时存储n-gram，保证部分匹配能力；4) 搜索时使用match_phrase与match组合查询，兼顾精确匹配和模糊匹配；5) 引入用户搜索行为反馈闭环，自动优化分词策略。
```

### 场景：大量消息同时索引导致 ES 过载
```
trigger: 公司全员大会结束后，10万员工同时在群聊中发送消息，消息索引写入QPS从日常的5000飙升至200000，ES集群CPU使用率超过95%，写入延迟从10ms升至5秒以上，部分bulk请求超时失败，搜索查询响应时间从50ms升至3秒，大量用户搜索超时报错。
detection: 1) ES集群监控：CPU>80%、写入延迟>500ms、rejected线程池任务>0时触发告警；2) 消息队列积压监控：索引队列深度超过10万条时告警；3) 搜索P99延迟监控：超过1秒时自动降级搜索服务；4) 消息索引失败率监控：超过1%时触发紧急响应。
handling: 1) 立即启用消息索引限流，将索引QPS限制在ES集群承受范围内（如50000），多余消息进入Kafka队列延迟索引；2) 搜索请求降级：暂停高亮和n-gram搜索，仅执行基础content匹配，减少ES计算压力；3) 临时增加ES节点，将部分索引分片迁移到新节点；4) 对非实时消息（如历史消息导入）暂停索引；5) 峰值过后逐步消化积压队列，恢复全部搜索功能；6) 事后分析峰值特征，调整集群容量和限流阈值。
prevention: 1) 消息索引采用Kafka队列+消费者组模式，消费者根据ES负载动态调整消费速率（背压机制）；2) ES集群预留30%冗余容量应对突发流量；3) 实现索引批量合并策略，将100条小消息合并为一次bulk请求减少网络开销；4) 搜索与索引使用不同的ES节点（冷热分离），索引压力不影响搜索；5) 定期进行压测，确保集群可承受3倍日常峰值的写入量；6) 配置ES circuit breaker，在内存不足时优雅降级而非崩溃。
```
