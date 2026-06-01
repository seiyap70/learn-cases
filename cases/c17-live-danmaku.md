# C17: 直播弹幕的高并发写入

## 业务场景

某直播平台，高峰期 10 万直播间同时在线，每秒产生 100 万条弹幕消息。弹幕是直播互动的核心——没有弹幕的直播间是"死"的。

**已知数据：**
- 同时在线直播间：10 万
- 峰值弹幕 QPS：100 万条/秒
- 单个热门直播间弹幕：1 万条/秒
- 弹幕消息大小：平均 50 字节
- 弹幕历史保留：直播结束后保留 24 小时（回放用）
- 弹幕延迟要求：< 1 秒（从发送到其他观众看到）

**弹幕的特殊性：**
- 写多读更多：1 条弹幕需要推送给该直播间所有观众
- 允许丢失普通弹幕，不允许丢失付费弹幕
- 弹幕不需要严格有序（少量乱序用户感知不到）

## 核心挑战

### 挑战 1：100 万 QPS 写入

100 万条/秒的弹幕写入。Kafka 单集群可以承受，但写入不是瓶颈——分发才是。

### 挑战 2：弹幕分发的扇出问题

这是核心难题。热门直播间 1 万观众，1 万条/秒弹幕 × 1 万观众 = 理论 1 亿次/秒推送。实际不可能做到——但也不需要做到。

**关键洞察：用户屏幕上每秒最多显示约 20-30 条弹幕（再多就看不清了）。** 1 万条/秒的弹幕不需要全部推送给每个观众，聚合后推送 20-30 条即可。

### 挑战 3：付费弹幕不可丢失

付费弹幕（礼物弹幕、超级弹幕）是用户花钱买的，丢失 = 丢钱。必须走可靠通道。

### 挑战 4：WebSocket 集群管理

1000 万 WebSocket 连接分布在 200 台网关服务器上。某台宕机后，上面的 5 万个连接如何恢复？

## 设计约束

- 通信协议：WebSocket
- 弹幕延迟 < 1 秒
- 普通弹幕允许少量丢失（实时性 > 可靠性）
- 付费弹幕不可丢失
- 单台 WebSocket 服务器连接上限约 5 万

## 请先独立思考（限时 30 分钟）

1. 100 万 QPS 写入 Kafka 后，如何高效分发给各直播间的观众？直接推 vs Pub/Sub vs 拉取？
2. 热门直播间的 1 万条/秒弹幕如何聚合？聚合策略对用户体验的影响？
3. 付费弹幕如何保证可靠送达？需要走与普通弹幕不同的通道吗？
4. 200 台 WebSocket 网关的路由管理：某台宕机后，5 万个连接如何快速迁移？

---

## 设计解析

### 数据库设计

```sql
-- 弹幕记录表（持久化存储，用于回放和审计）
CREATE TABLE danmaku (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    danmaku_id VARCHAR(64) NOT NULL,
    room_id VARCHAR(32) NOT NULL,
    user_id VARCHAR(32) NOT NULL,
    content TEXT NOT NULL,
    type VARCHAR(10) DEFAULT 'normal',    -- normal / gift / super
    timestamp_ms BIGINT NOT NULL,         -- 发送时间戳（毫秒）
    is_deleted BOOLEAN DEFAULT FALSE,     -- 审核后删除标记
    created_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE KEY uk_danmaku (danmaku_id),
    INDEX idx_room_time (room_id, timestamp_ms),
    INDEX idx_user_time (user_id, created_at)
);

-- 付费弹幕表（需要持久化和可查证）
CREATE TABLE paid_danmaku (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    danmaku_id VARCHAR(64) NOT NULL,
    room_id VARCHAR(32) NOT NULL,
    user_id VARCHAR(32) NOT NULL,
    content TEXT NOT NULL,
    type VARCHAR(10) NOT NULL,            -- gift / super
    price DECIMAL(10,2) NOT NULL,         -- 付费金额
    delivery_status VARCHAR(20) DEFAULT 'pending', -- pending / delivered / failed / refunded
    delivered_at TIMESTAMP,
    refunded_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE KEY uk_danmaku (danmaku_id),
    INDEX idx_room_time (room_id, created_at),
    INDEX idx_user (user_id),
    INDEX idx_delivery (delivery_status, created_at)
);

-- WebSocket 连接注册表
CREATE TABLE ws_connections (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    connection_id VARCHAR(64) NOT NULL,
    gateway_id VARCHAR(32) NOT NULL,       -- 网关服务器ID
    user_id VARCHAR(32),
    room_id VARCHAR(32) NOT NULL,
    connected_at TIMESTAMP DEFAULT NOW(),
    last_heartbeat TIMESTAMP,
    status VARCHAR(20) DEFAULT 'active',   -- active / disconnected
    
    INDEX idx_room_gateway (room_id, gateway_id),
    INDEX idx_gateway (gateway_id, status),
    INDEX idx_user (user_id)
);
```

### Kafka Topic 设计

```
Topic 设计：
  danmaku-normal  — 普通弹幕（允许聚合、采样）
    partition: 64（按 room_id 哈希分区）
    retention: 24小时
    consumer group: danmaku-aggregator（聚合服务消费）

  danmaku-paid   — 付费弹幕（不可丢失）
    partition: 16
    replication: 3
    retention: 72小时
    consumer group: danmaku-paid-processor（付费弹幕处理+持久化）

  danmaku-replay — 弹幕回放数据（持久化到 HDFS）
    partition: 32（按 room_id 哈希分区）
    retention: 7天
    consumer group: danmaku-archiver（归档服务消费）
```

**分区策略：** 按 `room_id` 哈希分区 → 同一直播间的弹幕在同一 partition → 保证单房间弹幕的顺序性。热门直播间可能产生大量弹幕集中在单个 partition → 热点问题。解决方案：热门直播间使用 `room_id + sub_index` 分区键，sub_index 取随机值（0-3），热门房间分散到 4 个 partition。

**分区容量规划：**

以 `danmaku-normal` 的 64 个分区为例：
- 峰值 100 万 QPS ÷ 64 分区 = 每分区 1.56 万条/秒
- 单条弹幕平均 50 字节 → 每分区写入吞吐 = 0.78 MB/s
- Kafka 单分区写入上限约 10-15 MB/s → 远未触及瓶颈
- 但消费端才是瓶颈：聚合服务单实例消费 8 分区 → 12.5 万条/秒 → CPU 占用约 60%
- 因此聚合服务需要 8 个实例（64 分区 ÷ 8 = 每实例 8 分区）

热点直播间分区倾斜问题：
- 某热门房间 1 万条/秒 → 全部进入同一分区 → 该分区负载 2.5 万条/秒
- 使用 `room_id + sub_index` 后 → 分散到 4 个分区 → 每分区 2500 条/秒
- 消费端需按 room_id 二次聚合 → 聚合服务内维护 room_id → List[弹幕] 的缓冲区

### Kafka 消费者完整实现（含 Offset 管理与 Rebalance 处理）

```python
import json
import logging
from collections import defaultdict
from confluent_kafka import Consumer, KafkaError, KafkaException, TopicPartition
from typing import Dict, List, Optional, Callable

logger = logging.getLogger("danmaku-consumer")


class DanmakuKafkaConsumer:
    """弹幕 Kafka 消费者：完整实现 offset 管理与 rebalance 处理"""

    def __init__(self, config: dict, topics: List[str],
                 message_handler: Callable,
                 rebalance_callback: Optional[Callable] = None):
        self.config = config
        self.topics = topics
        self.message_handler = message_handler
        self.rebalance_callback = rebalance_callback

        # Offset 管理：手动提交，避免重复消费和丢失
        self.pending_offsets: Dict[TopicPartition, int] = {}  # 待提交的 offset
        self.committed_offsets: Dict[TopicPartition, int] = {}  # 已提交的 offset
        self.offset_commit_interval = 5.0  # 每 5 秒提交一次 offset
        self.last_commit_time = 0

        # Rebalance 期间的状态保护
        self.rebalance_in_progress = False
        self.rebalance_lock = threading.Lock()

        # 消费暂停与恢复
        self.paused_partitions: List[TopicPartition] = []

        # 初始化消费者
        consumer_config = {
            'bootstrap.servers': config['kafka_brokers'],
            'group.id': config['group_id'],
            'enable.auto.commit': False,  # 关闭自动提交
            'auto.offset.reset': 'latest',  # 新消费者从最新开始
            'max.poll.records': 500,  # 单次 poll 最大条数
            'max.poll.interval.ms': 300000,  # 5 分钟处理超时
            'session.timeout.ms': 30000,  # 30 秒心跳超时
            'heartbeat.interval.ms': 10000,  # 10 秒心跳间隔
            'partition.assignment.strategy': 'cooperative-sticky',
            # 使用 CooperativeSticky 策略：rebalance 时不再全量回收分区，
            # 只移动需要迁移的分区，减少 rebalance 抖动
        }

        self.consumer = Consumer(consumer_config)

    def start(self):
        """启动消费者"""
        self.consumer.subscribe(
            self.topics,
            on_assign=self._on_assign,
            on_revoke=self._on_revoke,
            on_lost=self._on_lost,
        )

        logger.info(f"消费者启动，订阅 topics: {self.topics}")

        while self.running:
            try:
                msg = self.consumer.poll(timeout=1.0)

                if msg is None:
                    continue

                if msg.error():
                    self._handle_error(msg.error())
                    continue

                # Rebalance 期间暂缓处理（可选策略）
                if self.rebalance_in_progress:
                    # 将消息放回缓冲区，rebalance 完成后重新消费
                    continue

                # 处理消息
                try:
                    self.message_handler(msg)
                    # 记录待提交 offset（处理成功后才标记）
                    tp = TopicPartition(msg.topic(), msg.partition())
                    self.pending_offsets[tp] = msg.offset() + 1
                except Exception as e:
                    logger.error(f"消息处理失败: partition={msg.partition()}, "
                                 f"offset={msg.offset()}, error={e}")
                    # 处理失败的消息：跳过并记录到死信队列
                    self._send_to_dlq(msg, e)
                    # 仍然推进 offset，避免卡住
                    tp = TopicPartition(msg.topic(), msg.partition())
                    self.pending_offsets[tp] = msg.offset() + 1

                # 定期提交 offset
                self._maybe_commit_offsets()

            except KafkaException as e:
                logger.error(f"Kafka 异常: {e}")
                time.sleep(1)

    def _on_assign(self, consumer, partitions):
        """Rebalance 完成：分配到新分区"""
        with self.rebalance_lock:
            logger.info(f"分区分配: {[f'{p.topic}:{p.partition}' for p in partitions]}")

            for p in partitions:
                # 获取已提交的 offset
                committed = consumer.committed([p])[0]
                if committed and committed.offset >= 0:
                    # 从已提交的 offset 开始消费
                    consumer.seek(committed)
                    logger.info(f"分区 {p.topic}:{p.partition} 从 offset {committed.offset} 开始消费")
                else:
                    # 无已提交 offset，从最新开始（auto.offset.reset=latest）
                    logger.info(f"分区 {p.topic}:{p.partition} 无已提交 offset，从最新开始")

            self.rebalance_in_progress = False

            if self.rebalance_callback:
                self.rebalance_callback("assigned", partitions)

    def _on_revoke(self, consumer, partitions):
        """Rebalance 开始：分区即将被回收"""
        with self.rebalance_lock:
            logger.info(f"分区即将回收: {[f'{p.topic}:{p.partition}' for p in partitions]}")
            self.rebalance_in_progress = True

            # 关键：在分区被回收前，提交当前已处理的 offset
            # 避免新消费者重复消费
            self._commit_offsets_sync()

            # 暂停被回收分区的消费
            for p in partitions:
                consumer.pause([p])

            # 通知上层清理分区相关状态（如聚合缓冲区）
            if self.rebalance_callback:
                self.rebalance_callback("revoked", partitions)

    def _on_lost(self, consumer, partitions):
        """分区丢失（如 broker 宕机）"""
        with self.rebalance_lock:
            logger.warning(f"分区丢失: {[f'{p.topic}:{p.partition}' for p in partitions]}")
            self.rebalance_in_progress = True

            # 分区丢失时无法提交 offset → 只能清理本地状态
            for p in partitions:
                self.pending_offsets.pop(p, None)
                self.committed_offsets.pop(p, None)

            if self.rebalance_callback:
                self.rebalance_callback("lost", partitions)

    def _maybe_commit_offsets(self):
        """定期异步提交 offset"""
        now = time.time()
        if now - self.last_commit_time < self.offset_commit_interval:
            return

        if not self.pending_offsets:
            return

        # 异步提交
        offsets = [
            TopicPartition(p.topic, p.partition, offset)
            for p, offset in self.pending_offsets.items()
        ]

        self.consumer.commit(offsets=offsets, asynchronous=True)
        logger.debug(f"异步提交 offset: {len(offsets)} 个分区")

        # 更新已提交记录
        for p, offset in self.pending_offsets.items():
            self.committed_offsets[p] = offset
        self.pending_offsets.clear()
        self.last_commit_time = now

    def _commit_offsets_sync(self):
        """同步提交 offset（用于 rebalance 前的最终提交）"""
        if not self.pending_offsets:
            return

        offsets = [
            TopicPartition(p.topic, p.partition, offset)
            for p, offset in self.pending_offsets.items()
        ]

        try:
            self.consumer.commit(offsets=offsets, asynchronous=False)
            logger.info(f"同步提交 offset 成功: {len(offsets)} 个分区")
            for p, offset in self.pending_offsets.items():
                self.committed_offsets[p] = offset
            self.pending_offsets.clear()
        except KafkaException as e:
            logger.error(f"同步提交 offset 失败: {e}")
            # 提交失败 → 下次 rebalance 后可能重复消费
            # 但弹幕允许少量重复，可接受

    def _handle_error(self, error):
        """处理 Kafka 错误"""
        if error.code() == KafkaError._ALL_BROKERS_DOWN:
            logger.critical("所有 Kafka Broker 不可用")
            time.sleep(5)
        elif error.code() == KafkaError.UNKNOWN_TOPIC_OR_PART:
            logger.warning(f"Topic 或分区不存在: {error}")
        else:
            logger.error(f"Kafka 错误: code={error.code()}, msg={error}")

    def _send_to_dlq(self, msg, error):
        """发送到死信队列"""
        dlq_record = {
            "original_topic": msg.topic(),
            "original_partition": msg.partition(),
            "original_offset": msg.offset(),
            "key": msg.key(),
            "value": msg.value(),
            "error": str(error),
            "timestamp": int(time.time() * 1000),
        }
        self.kafka_produce("danmaku-dlq", dlq_record, key=msg.key())
        logger.warning(f"消息发送到死信队列: topic={msg.topic()}, "
                       f"partition={msg.partition()}, offset={msg.offset()}")

    def graceful_shutdown(self):
        """优雅关闭消费者"""
        logger.info("开始优雅关闭消费者...")
        self.running = False

        # 1. 同步提交最后的 offset
        self._commit_offsets_sync()

        # 2. 关闭消费者
        self.consumer.close()
        logger.info("消费者已关闭")


# 聚合服务的消费者实例化示例
def create_aggregator_consumer():
    """创建聚合服务消费者"""

    def handle_danmaku(msg):
        danmaku = json.loads(msg.value().decode('utf-8'))
        room_id = danmaku["roomId"]

        # 根据分区键判断是否需要二次聚合（热门直播间 sub_index 分区）
        # room_id 哈希到同一聚合桶
        aggregator.add_to_buffer(room_id, danmaku)

    def on_rebalance(event, partitions):
        if event == "revoked":
            # 分区被回收前，清空对应分区的聚合缓冲区
            for p in partitions:
                aggregator.flush_partition(p.topic, p.partition)
        elif event == "assigned":
            # 分配到新分区，初始化对应缓冲区
            for p in partitions:
                aggregator.init_partition(p.topic, p.partition)

    consumer = DanmakuKafkaConsumer(
        config={
            'kafka_brokers': 'kafka-1:9092,kafka-2:9092,kafka-3:9092',
            'group_id': 'danmaku-aggregator',
        },
        topics=['danmaku-normal'],
        message_handler=handle_danmaku,
        rebalance_callback=on_rebalance,
    )

    return consumer
```

**Offset 管理的关键原则：**

| 原则 | 说明 |
|------|------|
| 关闭自动提交 | `enable.auto.commit=False`，避免消息处理失败但 offset 已提交 |
| 至少一次语义 | 处理成功后才标记 offset → 可能重复消费，但不会丢失 |
| 定期异步提交 | 每 5 秒异步提交一次，平衡性能与重复消费量 |
| Rebalance 前同步提交 | 分区被回收前必须同步提交，避免新消费者重复消费 |
| 死信队列兜底 | 处理失败的消息不阻塞消费，转入死信队列人工处理 |

### 整体架构

```
观众发弹幕 → WebSocket网关 → Kafka → 弹幕消费服务(聚合/过滤)
                                        ↓
                                   Redis Pub/Sub(按roomId)
                                        ↓
                                   各WebSocket网关(订阅房间)
                                        ↓
                                   本地广播给该房间的观众
```

### 第一步：弹幕写入

```python
class DanmakuGateway:
    """WebSocket 网关：接收弹幕并写入 Kafka"""

    def on_danmaku(self, websocket, message):
        room_id = websocket.room_id
        user_id = websocket.user_id

        danmaku = {
            "id": uuid4(),
            "roomId": room_id,
            "userId": user_id,
            "content": message.content,
            "type": message.type,  # normal / gift / super
            "timestamp": int(time.time() * 1000),
        }

        # 付费弹幕额外处理
        if danmaku["type"] in ("gift", "super"):
            # 1. 写入数据库（持久化，可查证）
            self.db.insert_paid_danmaku(danmaku)
            # 2. 直接送 Redis Pub/Sub（不经过 Kafka 聚合，保证实时性）
            self.redis.publish(f"danmaku:room:{room_id}", json.dumps(danmaku))
            # 3. 同时写入 Kafka（回放用）
            self.kafka_produce("danmaku-paid", danmaku, key=room_id)
        else:
            # 普通弹幕走 Kafka（允许聚合延迟）
            self.kafka_produce("danmaku", danmaku, key=room_id)
```

**为什么付费弹幕走不同通道？**
- 付费弹幕不能经过聚合（可能被采样丢弃）
- 付费弹幕需要更高的实时性（不经过 Kafka 消费延迟）
- 付费弹幕需要持久化（可查证、可退款）

### 第二步：弹幕聚合

```python
class DanmakuAggregator:
    """消费 Kafka 弹幕，聚合后推送"""

    def __init__(self):
        self.buffers = defaultdict(list)  # roomId -> [弹幕列表]
        self.window_ms = 200             # 200ms 聚合窗口
        self.max_display_per_window = 25  # 每个窗口最多显示 25 条

    def on_message(self, message):
        danmaku = json.loads(message.value)
        self.buffers[danmaku["roomId"]].append(danmaku)

    def flush_all(self):
        """每 200ms 执行一次"""
        for room_id, items in self.buffers.items():
            if not items:
                continue

            if len(items) > 100:
                # 高热度直播间：聚合
                result = self._aggregate_high_volume(items)
            else:
                # 正常直播间：直接推送
                result = items

            # 推送到 Redis Pub/Sub
            self.redis.publish(
                f"danmaku:room:{room_id}",
                json.dumps(result)
            )

        self.buffers.clear()
        # 注册下一次 flush
        asyncio.get_event_loop().call_later(0.2, self.flush_all)

    def _aggregate_high_volume(self, items):
        """
        高热度聚合策略：
        - 付费弹幕全部保留
        - 普通弹幕：取最新 N 条 + 随机 M 条
        """
        paid = [i for i in items if i["type"] != "normal"]
        normal = [i for i in items if i["type"] == "normal"]

        # 保留最新的 15 条 + 随机 5 条（增加多样性）
        latest = normal[-15:]
        sampled = random.sample(normal, min(5, len(normal)))

        return paid + latest + sampled  # 约 20-25 条
```

**聚合窗口的选择：**

| 窗口大小 | 延迟 | 聚合效果 | 适用 |
|---------|------|---------|------|
| 50ms | 极低 | 差（来不及聚合） | 不推荐 |
| 200ms | 可接受 | 好 | 推荐 |
| 500ms | 偏高 | 极好 | 非热门可用 |
| 1000ms | 超过1秒 | 过度 | 不推荐（违反延迟要求） |

200ms 窗口：用户感知延迟约 200ms（聚合）+ 50ms（Pub/Sub 传输）= 250ms，远优于 1 秒要求。

### WebSocket 网关完整实现

```python
import asyncio
import json
import logging
import signal
import time
import uuid
from collections import defaultdict
from dataclasses import dataclass, field
from typing import Dict, Set, Optional, List
import websockets
from websockets.server import WebSocketServerProtocol

logger = logging.getLogger("ws-gateway")


@dataclass
class ConnectionInfo:
    """单个 WebSocket 连接的元信息"""
    ws: WebSocketServerProtocol
    connection_id: str
    user_id: Optional[str] = None
    room_id: Optional[str] = None
    connected_at: float = 0.0
    last_heartbeat_at: float = 0.0
    last_client_heartbeat_at: float = 0.0
    is_authenticated: bool = False
    send_queue: asyncio.Queue = field(default_factory=lambda: asyncio.Queue(maxsize=100))
    miss_heartbeat_count: int = 0


class WebSocketGateway:
    """WebSocket 网关完整实现：连接管理 + 心跳 + 房间订阅 + 优雅关闭"""

    def __init__(self, gateway_id: str, config: dict):
        self.gateway_id = gateway_id
        self.config = config

        # 连接管理
        self.connections: Dict[str, ConnectionInfo] = {}  # connection_id -> ConnectionInfo
        self.room_connections: Dict[str, Set[str]] = defaultdict(set)  # room_id -> {connection_id}
        self.user_connections: Dict[str, str] = {}  # user_id -> connection_id（单设备登录）

        # Redis Pub/Sub 订阅管理
        self.subscribed_rooms: Set[str] = set()  # 当前网关已订阅的房间集合
        self.redis_pubsub = None  # Redis Pub/Sub 连接
        self.redis_client = None  # Redis 普通连接（用于路由注册）

        # 运行状态
        self.running = True
        self.shutting_down = False
        self.server = None

        # 配置
        self.max_connections = config.get('max_connections', 50000)
        self.heartbeat_interval = config.get('heartbeat_interval', 30)  # 服务端心跳间隔（秒）
        self.heartbeat_timeout = config.get('heartbeat_timeout', 90)  # 心跳超时（秒）
        self.client_heartbeat_timeout = config.get('client_heartbeat_timeout', 60)  # 客户端心跳超时
        self.max_miss_heartbeat = config.get('max_miss_heartbeat', 3)  # 最大丢失心跳次数

    async def start(self):
        """启动 WebSocket 网关"""
        # 初始化 Redis 连接
        await self._init_redis()

        # 注册信号处理（优雅关闭）
        loop = asyncio.get_event_loop()
        for sig in (signal.SIGTERM, signal.SIGINT):
            loop.add_signal_handler(sig, lambda: asyncio.create_task(self.graceful_shutdown()))

        # 启动心跳检测协程
        asyncio.create_task(self._heartbeat_checker())

        # 启动 Redis Pub/Sub 消费协程
        asyncio.create_task(self._redis_subscriber())

        # 启动定期统计上报协程
        asyncio.create_task(self._stats_reporter())

        # 启动 WebSocket 服务器
        self.server = await websockets.serve(
            self._handle_connection,
            host=self.config['host'],
            port=self.config['port'],
            max_size=65536,  # 单帧最大 64KB
            ping_interval=None,  # 关闭 websockets 库自带 ping，用业务心跳
            ping_timeout=None,
            close_timeout=10,  # 关闭超时 10 秒
            compression="deflate",  # 启用 WebSocket 压缩（减少带宽）
        )

        logger.info(f"网关 {self.gateway_id} 启动，监听 {self.config['host']}:{self.config['port']}")

    async def _handle_connection(self, websocket: WebSocketServerProtocol, path: str):
        """处理新的 WebSocket 连接"""
        # 检查连接数上限
        if len(self.connections) >= self.max_connections:
            await websocket.close(code=5030, reason="网关连接数已满，请重试")
            logger.warning(f"拒绝新连接：已达上限 {self.max_connections}")
            return

        connection_id = str(uuid.uuid4())
        conn_info = ConnectionInfo(
            ws=websocket,
            connection_id=connection_id,
            connected_at=time.time(),
            last_heartbeat_at=time.time(),
            last_client_heartbeat_at=time.time(),
        )

        self.connections[connection_id] = conn_info

        try:
            # 等待认证消息（5 秒超时）
            auth_msg = await asyncio.wait_for(websocket.recv(), timeout=5.0)
            auth_data = json.loads(auth_msg)

            if not await self._authenticate(conn_info, auth_data):
                await websocket.close(code=4001, reason="认证失败")
                return

            conn_info.is_authenticated = True

            # 注册路由信息到 Redis
            await self._register_route(conn_info)

            # 启动发送协程（从 send_queue 取数据发送）
            send_task = asyncio.create_task(self._send_loop(conn_info))

            # 进入消息循环
            async for raw_msg in websocket:
                await self._on_message(conn_info, raw_msg)

        except asyncio.TimeoutError:
            await websocket.close(code=4002, reason="认证超时")
        except websockets.ConnectionClosed as e:
            logger.debug(f"连接关闭: {connection_id}, code={e.code}, reason={e.reason}")
        except json.JSONDecodeError:
            await websocket.close(code=4003, reason="消息格式错误")
        except Exception as e:
            logger.error(f"连接异常: {connection_id}, error={e}")
        finally:
            await self._on_disconnect(conn_info)
            send_task.cancel() if 'send_task' in dir() else None

    async def _authenticate(self, conn_info: ConnectionInfo, auth_data: dict) -> bool:
        """认证连接"""
        token = auth_data.get("token")
        room_id = auth_data.get("roomId")

        if not token or not room_id:
            return False

        # 验证 token（调用用户服务）
        user_info = await self._verify_token(token)
        if not user_info:
            return False

        conn_info.user_id = user_info["userId"]
        conn_info.room_id = room_id

        # 单设备登录：踢掉同一用户的旧连接
        if conn_info.user_id in self.user_connections:
            old_conn_id = self.user_connections[conn_info.user_id]
            old_conn = self.connections.get(old_conn_id)
            if old_conn:
                logger.info(f"踢掉用户 {conn_info.user_id} 的旧连接 {old_conn_id}")
                await self._kick_connection(old_conn, reason="其他设备登录")

        self.user_connections[conn_info.user_id] = conn_info.connection_id
        return True

    async def _on_message(self, conn_info: ConnectionInfo, raw_msg: str):
        """处理客户端消息"""
        try:
            msg = json.loads(raw_msg)
        except json.JSONDecodeError:
            return

        msg_type = msg.get("type")

        if msg_type == "heartbeat":
            # 客户端心跳响应
            conn_info.last_client_heartbeat_at = time.time()
            conn_info.miss_heartbeat_count = 0
            await conn_info.send_queue.put(json.dumps({
                "type": "heartbeat_ack",
                "serverTime": int(time.time() * 1000),
            }))

        elif msg_type == "danmaku":
            # 用户发送弹幕
            await self._handle_danmaku(conn_info, msg)

        elif msg_type == "change_room":
            # 用户切换房间
            await self._handle_change_room(conn_info, msg)

        elif msg_type == "paid_ack":
            # 付费弹幕送达回执
            await self._handle_paid_ack(conn_info, msg)

        else:
            logger.warning(f"未知消息类型: {msg_type}, conn={conn_info.connection_id}")

    async def _handle_danmaku(self, conn_info: ConnectionInfo, msg: dict):
        """处理用户发送的弹幕"""
        # 限速检查
        allowed, reason = await self._check_rate_limit(conn_info.user_id, conn_info.room_id)
        if not allowed:
            await conn_info.send_queue.put(json.dumps({
                "type": "danmaku_rejected",
                "reason": reason,
            }))
            return

        # 内容过滤
        content = msg.get("content", "")
        is_filtered, filter_reason = self._filter_content(content)
        if is_filtered:
            await conn_info.send_queue.put(json.dumps({
                "type": "danmaku_rejected",
                "reason": filter_reason,
            }))
            return

        # 构造弹幕消息
        danmaku = {
            "id": str(uuid.uuid4()),
            "roomId": conn_info.room_id,
            "userId": conn_info.user_id,
            "content": content,
            "type": msg.get("danmaku_type", "normal"),
            "timestamp": int(time.time() * 1000),
        }

        # 根据类型走不同通道
        if danmaku["type"] in ("gift", "super"):
            # 付费弹幕：写入 DB + 直推 Redis + 写 Kafka 归档
            await self.db.insert_paid_danmaku(danmaku)
            await self.redis_client.publish(
                f"danmaku:room:{conn_info.room_id}", json.dumps(danmaku)
            )
            await self.kafka_produce("danmaku-paid", danmaku, key=conn_info.room_id)
        else:
            # 普通弹幕：写入 Kafka
            await self.kafka_produce("danmaku", danmaku, key=conn_info.room_id)

    async def _handle_change_room(self, conn_info: ConnectionInfo, msg: dict):
        """处理用户切换房间"""
        new_room_id = msg.get("roomId")
        if not new_room_id or new_room_id == conn_info.room_id:
            return

        old_room_id = conn_info.room_id

        # 1. 从旧房间移除
        self.room_connections[old_room_id].discard(conn_info.connection_id)
        if not self.room_connections[old_room_id]:
            del self.room_connections[old_room_id]
            # 该房间在本网关上无观众了 → 取消订阅
            await self._unsubscribe_room(old_room_id)

        # 2. 加入新房间
        conn_info.room_id = new_room_id
        self.room_connections[new_room_id].add(conn_info.connection_id)
        # 如果该房间在本网关上首次有观众 → 订阅 Redis Pub/Sub
        if len(self.room_connections[new_room_id]) == 1:
            await self._subscribe_room(new_room_id)

        # 3. 更新 Redis 路由
        await self._update_route(conn_info, old_room_id, new_room_id)

        await conn_info.send_queue.put(json.dumps({
            "type": "room_changed",
            "roomId": new_room_id,
        }))

    async def _on_disconnect(self, conn_info: ConnectionInfo):
        """连接断开处理"""
        if conn_info.connection_id not in self.connections:
            return

        # 清理连接记录
        del self.connections[conn_info.connection_id]

        if conn_info.user_id and conn_info.user_id in self.user_connections:
            del self.user_connections[conn_info.user_id]

        if conn_info.room_id:
            self.room_connections[conn_info.room_id].discard(conn_info.connection_id)
            if not self.room_connections[conn_info.room_id]:
                del self.room_connections[conn_info.room_id]
                await self._unsubscribe_room(conn_info.room_id)

        # 清理 Redis 路由
        await self._unregister_route(conn_info)

        logger.info(f"连接断开: conn={conn_info.connection_id}, "
                     f"user={conn_info.user_id}, room={conn_info.room_id}, "
                     f"当前连接数={len(self.connections)}")

    async def _send_loop(self, conn_info: ConnectionInfo):
        """发送协程：从队列取数据发送给客户端"""
        try:
            while True:
                msg = await conn_info.send_queue.get()
                try:
                    await conn_info.ws.send(msg)
                except websockets.ConnectionClosed:
                    break
                except Exception as e:
                    logger.error(f"发送失败: conn={conn_info.connection_id}, error={e}")
                    break
        except asyncio.CancelledError:
            pass

    async def _heartbeat_checker(self):
        """心跳检测协程：定期检查所有连接的心跳"""
        while self.running:
            now = time.time()
            timeout_conns = []

            for conn_id, conn_info in list(self.connections.items()):
                # 检查客户端心跳超时
                if now - conn_info.last_client_heartbeat_at > self.client_heartbeat_timeout:
                    timeout_conns.append(conn_info)
                    continue

                # 服务端主动发心跳
                if now - conn_info.last_heartbeat_at > self.heartbeat_interval:
                    try:
                        await conn_info.send_queue.put(json.dumps({
                            "type": "heartbeat",
                            "serverTime": int(now * 1000),
                        }))
                        conn_info.last_heartbeat_at = now
                        conn_info.miss_heartbeat_count += 1

                        # 连续丢失心跳超过阈值 → 关闭连接
                        if conn_info.miss_heartbeat_count > self.max_miss_heartbeat:
                            timeout_conns.append(conn_info)
                    except Exception:
                        timeout_conns.append(conn_info)

            # 关闭超时连接
            for conn_info in timeout_conns:
                logger.info(f"心跳超时关闭连接: conn={conn_info.connection_id}")
                try:
                    await conn_info.ws.close(code=4004, reason="心跳超时")
                except Exception:
                    pass
                await self._on_disconnect(conn_info)

            await asyncio.sleep(5)  # 每 5 秒检查一次

    async def _redis_subscriber(self):
        """Redis Pub/Sub 订阅协程：接收弹幕并推送给本地连接"""
        while self.running:
            try:
                message = await self.redis_pubsub.get_message(
                    ignore_subscribe_messages=True, timeout=1.0
                )
                if message and message["type"] == "message":
                    channel = message["channel"]
                    if isinstance(channel, bytes):
                        channel = channel.decode("utf-8")

                    room_id = channel.split(":")[-1]
                    danmaku_data = message["data"]
                    if isinstance(danmaku_data, bytes):
                        danmaku_data = danmaku_data.decode("utf-8")

                    # 推送给该房间在本地网关上的所有连接
                    await self._broadcast_to_room(room_id, danmaku_data)

            except Exception as e:
                logger.error(f"Redis Pub/Sub 消费异常: {e}")
                await asyncio.sleep(1)

    async def _broadcast_to_room(self, room_id: str, data: str):
        """向本网关上某房间的所有连接广播消息"""
        conn_ids = self.room_connections.get(room_id, set())
        for conn_id in conn_ids:
            conn_info = self.connections.get(conn_id)
            if conn_info and not conn_info.send_queue.full():
                try:
                    await conn_info.send_queue.put(data)
                except asyncio.QueueFull:
                    # 队列满 → 丢弃普通弹幕，付费弹幕走独立逻辑
                    logger.warning(f"发送队列满，丢弃消息: conn={conn_id}")

    async def _subscribe_room(self, room_id: str):
        """订阅房间的 Redis Pub/Sub 频道"""
        if room_id in self.subscribed_rooms:
            return

        await self.redis_pubsub.subscribe(f"danmaku:room:{room_id}")
        self.subscribed_rooms.add(room_id)
        logger.info(f"订阅房间: {room_id}, 当前订阅数={len(self.subscribed_rooms)}")

    async def _unsubscribe_room(self, room_id: str):
        """取消订阅房间的 Redis Pub/Sub 频道"""
        if room_id not in self.subscribed_rooms:
            return

        await self.redis_pubsub.unsubscribe(f"danmaku:room:{room_id}")
        self.subscribed_rooms.discard(room_id)
        logger.info(f"取消订阅房间: {room_id}, 当前订阅数={len(self.subscribed_rooms)}")

    async def graceful_shutdown(self):
        """优雅关闭网关"""
        if self.shutting_down:
            return

        self.shutting_down = True
        logger.info(f"网关 {self.gateway_id} 开始优雅关闭...")

        # 1. 停止接受新连接
        if self.server:
            self.server.close()
            await self.server.wait_closed()

        # 2. 通知所有客户端即将断开（给客户端时间重连）
        close_msg = json.dumps({
            "type": "server_shutdown",
            "reason": "网关维护中，请重连",
            "retryAfter": 3,
        })

        for conn_id, conn_info in list(self.connections.items()):
            try:
                await conn_info.send_queue.put(close_msg)
            except Exception:
                pass

        # 3. 等待 5 秒让客户端处理关闭通知
        await asyncio.sleep(5)

        # 4. 关闭所有 WebSocket 连接
        for conn_id, conn_info in list(self.connections.items()):
            try:
                await conn_info.ws.close(code=1001, reason="服务器关闭")
            except Exception:
                pass

        # 5. 清理 Redis 路由和订阅
        await self.redis_pubsub.unsubscribe()
        await self.redis_client.delete(f"ws:gateway:{self.gateway_id}")
        for room_id in self.subscribed_rooms:
            await self.redis_client.srem(f"ws:room:{room_id}", self.gateway_id)

        self.running = False
        logger.info(f"网关 {self.gateway_id} 关闭完成")

    async def _stats_reporter(self):
        """定期上报网关统计信息"""
        while self.running:
            stats = {
                "gatewayId": self.gateway_id,
                "connectionCount": len(self.connections),
                "subscribedRooms": len(self.subscribed_rooms),
                "timestamp": int(time.time() * 1000),
            }
            await self.redis_client.set(
                f"ws:stats:{self.gateway_id}", json.dumps(stats), ex=60
            )
            await asyncio.sleep(10)


# 网关启动入口
async def main():
    gateway = WebSocketGateway(
        gateway_id="gateway-01",
        config={
            'host': '0.0.0.0',
            'port': 8443,
            'max_connections': 50000,
            'heartbeat_interval': 30,
            'heartbeat_timeout': 90,
            'client_heartbeat_timeout': 60,
            'max_miss_heartbeat': 3,
        }
    )
    await gateway.start()

if __name__ == "__main__":
    asyncio.run(main())
```

**WebSocket 网关关键设计要点：**

| 要点 | 设计 |
|------|------|
| 连接数上限 | 单机 5 万，超限返回 5030 状态码 |
| 认证超时 | 5 秒内必须完成认证，否则断开 |
| 心跳策略 | 服务端 30 秒发心跳，客户端 60 秒无心跳则断开 |
| 发送队列 | 每连接 100 条队列，满则丢弃普通弹幕 |
| 房间订阅 | 按需订阅/取消订阅 Redis Pub/Sub |
| 优雅关闭 | 先通知客户端 → 等待 5 秒 → 关闭连接 → 清理 Redis |
| 压缩传输 | 启用 WebSocket permessage-deflate 压缩 |

### 第三步：Redis Pub/Sub 分发

```python
class GatewaySubscriber:
    """WebSocket 网关订阅 Redis Pub/Sub"""

    def on_room_subscribed(self, room_id):
        """当有观众进入直播间时，网关订阅该房间的弹幕频道"""
        self.redis.subscribe(f"danmaku:room:{room_id}")

    def on_room_unsubscribed(self, room_id):
        """当直播间在该网关上无观众时，取消订阅"""
        self.redis.unsubscribe(f"danmaku:room:{room_id}")

    def on_danmaku_message(self, redis_message):
        """收到弹幕消息，推送给本地连接的观众"""
        room_id = redis_message.channel.split(":")[-1]
        danmaku_list = json.loads(redis_message.data)

        # 遍历该房间在本地网关上的所有 WebSocket 连接
        connections = self.get_room_connections(room_id)
        for ws in connections:
            try:
                ws.send(json.dumps(danmaku_list))
            except ConnectionClosed:
                self.remove_connection(ws)
```

**为什么用 Redis Pub/Sub 而非直接推送？**

- 热门直播间的观众分布在 50+ 台网关上
- 弹幕服务只需发布 1 次到 Redis → 50 个网关各自收到
- 发布 O(1)，订阅 O(1)，总推送 O(网关数) 而非 O(观众数)

**Redis Pub/Sub 的风险和应对：**

| 风险 | 影响 | 应对 |
|------|------|------|
| 消息不持久化 | 网关断线期间丢失弹幕 | 普通弹幕可接受；付费弹幕走独立通道 |
| Redis 单点 | 全局弹幕分发失败 | Redis Sentinel 高可用 |
| 频道数过多 | 10万直播间 = 10万频道 | 只订阅有本网关观众的频道 |

### WebSocket 集群路由管理

```python
class ConnectionRouter:
    """管理 WebSocket 连接的路由信息"""

    def on_connect(self, gateway_id, room_id, user_id, connection_id):
        """观众连接时注册路由"""
        # 记录：该房间在哪些网关上有观众
        self.redis.sadd(f"ws:room:{room_id}", gateway_id)
        # 记录：该网关服务哪些房间
        self.redis.sadd(f"ws:gateway:{gateway_id}", room_id)
        # 记录：该房间在该网关上的连接数
        self.redis.incr(f"ws:room_count:{room_id}:{gateway_id}")
        # 记录：用户的连接信息（用于定向推送）
        self.redis.set(f"ws:user:{user_id}", json.dumps({
            "gateway_id": gateway_id,
            "room_id": room_id,
            "connection_id": connection_id
        }))

        # 如果该房间在该网关上首次有观众 → 订阅 Redis Pub/Sub
        if self.redis.get(f"ws:room_count:{room_id}:{gateway_id}") == "1":
            self.gateway_subscribe(gateway_id, room_id)

    def on_disconnect(self, gateway_id, room_id, user_id):
        """观众断开时清理路由"""
        # 减少连接计数
        count = self.redis.decr(f"ws:room_count:{room_id}:{gateway_id}")

        if count <= 0:
            # 该房间在此网关上无观众了 → 取消订阅
            self.redis.srem(f"ws:room:{room_id}", gateway_id)
            self.redis.srem(f"ws:gateway:{gateway_id}", room_id)
            self.redis.delete(f"ws:room_count:{room_id}:{gateway_id}")
            self.gateway_unsubscribe(gateway_id, room_id)

        # 清除用户连接信息
        self.redis.delete(f"ws:user:{user_id}")
```

**网关宕机恢复：**

```
1. 心跳检测发现网关 G5 宕机
   → 所有连接在 G5 上的用户 WebSocket 断开
   
2. 清理 G5 的路由信息：
   → 读取 ws:gateway:G5 获取所有房间列表
   → 对每个房间：SREM ws:room:{roomId} G5
   → 删除 ws:gateway:G5
   
3. 客户端自动重连：
   → 客户端检测断线 → 500ms 后尝试重连
   → DNS/负载均衡分配到其他网关
   → 新网关注册路由 → 重新订阅 Redis Pub/Sub
   
4. 付费弹幕重试：
   → G5 宕机前未送达的付费弹幕
   → 5秒后重试 → 查 ws:room:{roomId} 获取新网关列表
   → 向新网关推送
   
恢复时间：约 3-10 秒（取决于客户端重连策略）
```

**WebSocket 连接数估算：**

| 资源 | 数量 |
|------|------|
| 1000 万 WebSocket 连接 | 需 200 台网关（每台 5 万连接） |
| 每台网关订阅约 500 个房间 | 500 个 Redis Pub/Sub 频道 |
| 每台网关内存 | 约 2GB（5 万连接 × 40KB/连接） |
| Redis Pub/Sub 内存 | 约 10MB（500 频道 × 20 台网关） |

### 完整的弹幕流转时序

### 付费弹幕的可靠送达

付费弹幕的可靠送达需要解决两个问题：1) 确保推送到所有在线观众 2) 确认送达（可查证）。

```python
class PaidDanmakuService:
    """付费弹幕服务：必须可靠送达"""

    def send_paid_danmaku(self, danmaku):
        room_id = danmaku["roomId"]
        
        # 1. 持久化到数据库
        self.db.insert("paid_danmaku", danmaku)
        
        # 2. 直接推送（不经过 Kafka 聚合）
        # 先尝试 Redis Pub/Sub（快速通道）
        subscribers = self.redis.publish(f"danmaku:room:{room_id}", json.dumps(danmaku))
        
        if subscribers == 0:
            # 没有订阅者（可能网关正在重连）
            # 放入重试队列
            self.redis.rpush(f"danmaku:retry:{room_id}", json.dumps(danmaku))
        
        # 3. 确认送达（客户端回执）
        # 5 秒内未收到回执 → 重试推送
        self.schedule_ack_check(danmaku["id"], room_id, timeout=5)

    def on_ack_received(self, danmaku_id):
        """客户端确认收到付费弹幕"""
        self.redis.set(f"danmaku:ack:{danmaku_id}", "1", ex=3600)

    def check_unacked(self, danmaku_id, room_id):
        """检查付费弹幕是否送达"""
        if self.redis.exists(f"danmaku:ack:{danmaku_id}"):
            return  # 已送达
        
        # 未送达 → 重试推送
        danmaku = self.db.get_paid_danmaku(danmaku_id)
        subscribers = self.redis.publish(f"danmaku:room:{room_id}", json.dumps(danmaku))
        
        if subscribers > 0:
            # 有订阅者了，再次等待回执
            self.schedule_ack_check(danmaku_id, room_id, timeout=5, retry_count=1)
        else:
            # 3 次重试后仍无法送达 → 标记未送达，退款
            self.mark_delivery_failed(danmaku_id)
            self.initiate_refund(danmaku)
```

### 回放弹幕同步

```python
class ReplayService:
    """回放弹幕：按视频时间轴推送"""

    def get_danmaku_segment(self, room_id, video_offset_ms, duration_ms):
        """
        获取视频某个时间段的弹幕
        video_offset_ms: 视频播放进度（相对于直播开始时间）
        duration_ms: 请求的时间段长度（如 10 秒）
        """
        start_time = self.live_start_time(room_id) + timedelta(milliseconds=video_offset_ms)
        end_time = start_time + timedelta(milliseconds=duration_ms)

        # 从 Kafka/HDFS 按时间范围查询弹幕
        danmaku_list = self.storage.query_range(
            topic=f"danmaku-replay-{room_id}",
            start_time=start_time,
            end_time=end_time
        )

        # 按时间戳排序
        danmaku_list.sort(key=lambda d: d["timestamp"])

        return danmaku_list
```

### 弹幕存储与回放完整实现

#### 存储架构

```
实时弹幕写入流：
  Kafka(danmaku-normal) → 聚合服务 → Redis Pub/Sub → 实时推送
  Kafka(danmaku-paid) → 付费弹幕处理 → MySQL(paid_danmaku) + 实时推送

弹幕持久化流（异步）：
  Kafka(danmaku-replay) → 归档服务 → 分级存储
    ├─ 热数据（0-24h）: Redis Sorted Set（按时间戳排序）
    ├─ 温数据（1-7天）: MySQL（弹幕记录表）
    └─ 冷数据（>7天）: HDFS/对象存储（Parquet 格式，按 room_id + 日期分区）
```

#### Redis 热数据存储

```python
class DanmakuHotStorage:
    """弹幕热数据存储：Redis Sorted Set，按时间戳排序"""

    def __init__(self, redis_client):
        self.redis = redis_client
        self.ttl = 86400 * 2  # 48 小时过期（直播结束后 24 小时 + 余量）

    async def store(self, danmaku: dict):
        """存储弹幕到 Redis Sorted Set"""
        room_id = danmaku["roomId"]
        key = f"danmaku:hot:{room_id}"
        timestamp_ms = danmaku["timestamp"]

        # 使用时间戳作为 score，弹幕 JSON 作为 value
        # ZADD 自动按 score 排序，天然支持时间范围查询
        member = json.dumps({
            "id": danmaku["id"],
            "userId": danmaku["userId"],
            "content": danmaku["content"],
            "type": danmaku["type"],
            "timestamp": timestamp_ms,
        })

        pipe = self.redis.pipeline()
        pipe.zadd(key, {member: timestamp_ms})
        pipe.expire(key, self.ttl)

        # 限制单个房间的热数据量（最多保留最近 10 万条）
        # 避免极端热门直播间撑爆 Redis
        pipe.zremrangebyrank(key, 0, -100001)

        await pipe.execute()

    async def query_range(self, room_id: str, start_ms: int, end_ms: int,
                          limit: int = 500) -> List[dict]:
        """按时间范围查询弹幕（用于回放）"""
        key = f"danmaku:hot:{room_id}"

        # ZRANGEBYSCORE：O(log(N) + M)，N 为集合大小，M 为返回条数
        raw_list = await self.redis.zrangebyscore(
            key, start_ms, end_ms,
            start=0, num=limit,
            withscores=False,
        )

        return [json.loads(item) for item in raw_list]

    async def query_latest(self, room_id: str, count: int = 30) -> List[dict]:
        """获取最新 N 条弹幕（用户进入直播间时加载）"""
        key = f"danmaku:hot:{room_id}"

        # ZREVRANGE 取最新的，然后反转按时间正序
        raw_list = await self.redis.zrevrange(key, 0, count - 1, withscores=False)
        result = [json.loads(item) for item in reversed(raw_list)]
        return result

    async def get_room_stats(self, room_id: str) -> dict:
        """获取房间弹幕统计（用于监控）"""
        key = f"danmaku:hot:{room_id}"
        count = await self.redis.zcard(key)
        if count == 0:
            return {"count": 0, "oldest_ms": 0, "newest_ms": 0}

        oldest = await self.redis.zrange(key, 0, 0, withscores=True)
        newest = await self.redis.zrevrange(key, 0, 0, withscores=True)

        return {
            "count": count,
            "oldest_ms": int(oldest[0][1]) if oldest else 0,
            "newest_ms": int(newest[0][1]) if newest else 0,
        }
```

#### MySQL 温数据存储与查询

```python
class DanmakuWarmStorage:
    """弹幕温数据存储：MySQL，支持复杂查询和审计"""

    def __init__(self, db_pool):
        self.db = db_pool

    async def batch_insert(self, danmaku_list: List[dict]):
        """批量写入弹幕（归档服务定期调用）"""
        if not danmaku_list:
            return

        values = []
        placeholders = []
        for d in danmaku_list:
            placeholders.append("(%s, %s, %s, %s, %s, %s)")
            values.extend([
                d["id"], d["roomId"], d["userId"],
                d["content"], d["type"], d["timestamp"],
            ])

        sql = f"""
            INSERT INTO danmaku (danmaku_id, room_id, user_id, content, type, timestamp_ms)
            VALUES {', '.join(placeholders)}
            ON DUPLICATE KEY UPDATE content = VALUES(content)
        """

        await self.db.execute(sql, values)

    async def query_by_time_range(self, room_id: str,
                                   start_ms: int, end_ms: int,
                                   page: int = 1, page_size: int = 200) -> dict:
        """按时间范围分页查询弹幕"""
        offset = (page - 1) * page_size

        # 使用 idx_room_time 索引
        count_sql = """
            SELECT COUNT(*) FROM danmaku
            WHERE room_id = %s AND timestamp_ms BETWEEN %s AND %s
              AND is_deleted = FALSE
        """
        data_sql = """
            SELECT danmaku_id, room_id, user_id, content, type, timestamp_ms
            FROM danmaku
            WHERE room_id = %s AND timestamp_ms BETWEEN %s AND %s
              AND is_deleted = FALSE
            ORDER BY timestamp_ms ASC
            LIMIT %s OFFSET %s
        """

        count = await self.db.fetchone(count_sql, [room_id, start_ms, end_ms])
        rows = await self.db.fetchall(data_sql, [room_id, start_ms, end_ms, page_size, offset])

        return {
            "total": count,
            "page": page,
            "pageSize": page_size,
            "items": rows,
        }

    async def query_by_user(self, user_id: str, limit: int = 50) -> List[dict]:
        """查询用户发送的弹幕（用户中心展示）"""
        sql = """
            SELECT danmaku_id, room_id, content, type, timestamp_ms
            FROM danmaku
            WHERE user_id = %s AND is_deleted = FALSE
            ORDER BY created_at DESC
            LIMIT %s
        """
        return await self.db.fetchall(sql, [user_id, limit])

    async def soft_delete(self, danmaku_ids: List[str], operator: str):
        """软删除弹幕（审核删除）"""
        sql = """
            UPDATE danmaku SET is_deleted = TRUE
            WHERE danmaku_id IN (%s)
        """ % ','.join(['%s'] * len(danmaku_ids))

        await self.db.execute(sql, danmaku_ids)
        logger.info(f"审核删除弹幕: ids={danmaku_ids}, operator={operator}")
```

#### HDFS 冷数据归档与查询

```python
class DanmakuColdStorage:
    """弹幕冷数据归档：HDFS + Parquet 格式"""

    def __init__(self, hdfs_client, config):
        self.hdfs = hdfs_client
        self.config = config
        self.archive_base_path = config.get('archive_path', '/data/danmaku/archive')

    async def archive_partition(self, room_id: str, date_str: str, danmaku_list: List[dict]):
        """归档一天的弹幕到 HDFS（Parquet 格式）"""
        if not danmaku_list:
            return

        # HDFS 路径：/data/danmaku/archive/{roomId}/{date}.parquet
        dir_path = f"{self.archive_base_path}/{room_id}"
        file_path = f"{dir_path}/{date_str}.parquet"

        # 转为 Parquet 格式（列式存储，压缩比高）
        # 使用 PyArrow 写入
        import pyarrow as pa
        import pyarrow.parquet as pq

        table = pa.table({
            'danmaku_id': [d['id'] for d in danmaku_list],
            'room_id': [d['roomId'] for d in danmaku_list],
            'user_id': [d['userId'] for d in danmaku_list],
            'content': [d['content'] for d in danmaku_list],
            'type': [d['type'] for d in danmaku_list],
            'timestamp_ms': [d['timestamp'] for d in danmaku_list],
        })

        # Snappy 压缩：压缩比约 2:1，解压速度快
        pq.write_table(table, file_path, compression='snappy')

        file_size = self.hdfs.get_file_info(file_path).size
        logger.info(f"归档完成: room={room_id}, date={date_str}, "
                     f"count={len(danmaku_list)}, size={file_size / 1024:.1f}KB")

    async def query_cold(self, room_id: str, date_str: str,
                         start_ms: int, end_ms: int) -> List[dict]:
        """从 HDFS 查询冷数据弹幕"""
        import pyarrow.parquet as pq

        file_path = f"{self.archive_base_path}/{room_id}/{date_str}.parquet"

        if not self.hdfs.exists(file_path):
            return []

        table = pq.read_table(file_path)
        df = table.to_pandas()

        # 按时间范围过滤
        mask = (df['timestamp_ms'] >= start_ms) & (df['timestamp_ms'] <= end_ms)
        filtered = df[mask].sort_values('timestamp_ms')

        return filtered.to_dict('records')


class DanmakuArchiveService:
    """弹幕归档服务：消费 Kafka，分级写入存储"""

    def __init__(self, hot_storage, warm_storage, cold_storage):
        self.hot = hot_storage
        self.warm = warm_storage
        self.cold = cold_storage
        self.batch_buffer: Dict[str, List[dict]] = defaultdict(list)
        self.batch_size = 500
        self.flush_interval = 10  # 秒

    async def on_message(self, danmaku: dict):
        """处理 Kafka 弹幕消息"""
        room_id = danmaku["roomId"]

        # 1. 写入 Redis 热数据
        await self.hot.store(danmaku)

        # 2. 缓冲到批量写入列表
        self.batch_buffer[room_id].append(danmaku)

        # 达到批量大小则立即刷盘
        if len(self.batch_buffer[room_id]) >= self.batch_size:
            await self._flush_room(room_id)

    async def _flush_room(self, room_id: str):
        """刷盘一个房间的缓冲弹幕到 MySQL"""
        items = self.batch_buffer.pop(room_id, [])
        if not items:
            return

        try:
            await self.warm.batch_insert(items)
        except Exception as e:
            logger.error(f"MySQL 批量写入失败: room={room_id}, count={len(items)}, error={e}")
            # 写入失败 → 放回缓冲区，下次重试
            self.batch_buffer[room_id] = items + self.batch_buffer.get(room_id, [])

    async def periodic_flush(self):
        """定期刷盘"""
        while True:
            await asyncio.sleep(self.flush_interval)
            for room_id in list(self.batch_buffer.keys()):
                await self._flush_room(room_id)

    async def daily_archive(self):
        """每日归档任务：将 7 天前的 MySQL 数据归档到 HDFS"""
        cutoff_date = (datetime.now() - timedelta(days=7)).strftime('%Y-%m-%d')

        # 查询所有超过 7 天的弹幕
        rooms = await self.warm.db.fetchall(
            "SELECT DISTINCT room_id FROM danmaku WHERE DATE(created_at) < %s", [cutoff_date]
        )

        for room in rooms:
            room_id = room['room_id']
            # 按 room_id + 日期批量导出
            danmaku_list = await self.warm.db.fetchall(
                """SELECT danmaku_id, room_id, user_id, content, type, timestamp_ms
                   FROM danmaku WHERE room_id = %s AND DATE(created_at) < %s
                   ORDER BY timestamp_ms""",
                [room_id, cutoff_date]
            )

            if danmaku_list:
                await self.cold.archive_partition(room_id, cutoff_date, danmaku_list)
                # 归档成功后删除 MySQL 中的记录
                await self.warm.db.execute(
                    "DELETE FROM danmaku WHERE room_id = %s AND DATE(created_at) < %s",
                    [room_id, cutoff_date]
                )
                logger.info(f"归档并清理: room={room_id}, date<{cutoff_date}, "
                             f"count={len(danmaku_list)}")
```

#### 回放弹幕时间轴同步

```python
class DanmakuReplaySync:
    """回放弹幕时间轴同步：确保弹幕与视频进度对齐"""

    def __init__(self, hot_storage, warm_storage, cold_storage):
        self.hot = hot_storage
        self.warm = warm_storage
        self.cold = cold_storage

    async def get_danmaku_timeline(self, room_id: str, live_id: str,
                                   video_offset_ms: int, window_ms: int = 10000) -> dict:
        """
        获取回放弹幕时间轴
        video_offset_ms: 当前视频播放位置（毫秒）
        window_ms: 请求窗口长度（默认 10 秒）

        返回：
          - items: 弹幕列表（按时间排序）
          - next_offset: 下次请求的 offset（用于预加载）
        """
        # 计算弹幕时间范围（相对于直播开始时间）
        live_start_ms = await self._get_live_start_ms(room_id, live_id)
        start_ms = live_start_ms + video_offset_ms
        end_ms = start_ms + window_ms

        # 分级查询：先查 Redis 热数据，再查 MySQL 温数据，最后查 HDFS 冷数据
        danmaku_list = await self.hot.query_range(room_id, start_ms, end_ms)

        if not danmaku_list:
            # 热数据未命中 → 查 MySQL
            result = await self.warm.query_by_time_range(room_id, start_ms, end_ms)
            danmaku_list = result.get("items", [])

        if not danmaku_list:
            # 温数据未命中 → 查 HDFS
            date_str = self._ms_to_date(start_ms)
            danmaku_list = await self.cold.query_cold(room_id, date_str, start_ms, end_ms)

        # 弹幕密度自适应：
        # 如果窗口内弹幕太多（>200 条），进行采样以保证回放体验
        if len(danmaku_list) > 200:
            danmaku_list = self._adaptive_sample(danmaku_list, target_count=150)

        # 为客户端计算预加载建议
        next_offset = video_offset_ms + window_ms
        density = len(danmaku_list) / max(window_ms / 1000, 1)  # 条/秒

        # 如果密度高，建议客户端更频繁地请求
        suggested_window = max(3000, int(10000 / max(density / 10, 1)))

        return {
            "items": danmaku_list,
            "nextOffset": next_offset,
            "density": round(density, 1),
            "suggestedWindow": suggested_window,
        }

    def _adaptive_sample(self, danmaku_list: List[dict], target_count: int) -> List[dict]:
        """自适应采样：保留付费弹幕 + 均匀采样普通弹幕"""
        paid = [d for d in danmaku_list if d.get("type") != "normal"]
        normal = [d for d in danmaku_list if d.get("type") == "normal"]

        if len(paid) >= target_count:
            return paid[:target_count]

        remaining = target_count - len(paid)

        # 均匀采样：将普通弹幕按时间等分为 remaining 个桶，每桶取 1 条
        if len(normal) <= remaining:
            return paid + normal

        step = len(normal) / remaining
        sampled = [normal[int(i * step)] for i in range(remaining)]

        return paid + sampled

    async def _get_live_start_ms(self, room_id: str, live_id: str) -> int:
        """获取直播开始时间（毫秒时间戳）"""
        # 从 Redis 或数据库查询直播记录
        cache_key = f"live:start:{live_id}"
        cached = await self.redis.get(cache_key)
        if cached:
            return int(cached)

        # 查数据库
        live_record = await self.db.fetchone(
            "SELECT start_time_ms FROM live_record WHERE live_id = %s", [live_id]
        )
        if live_record:
            start_ms = live_record['start_time_ms']
            await self.redis.setex(cache_key, 86400, start_ms)
            return start_ms

        return 0

    @staticmethod
    def _ms_to_date(timestamp_ms: int) -> str:
        """毫秒时间戳转日期字符串"""
        from datetime import datetime
        return datetime.fromtimestamp(timestamp_ms / 1000).strftime('%Y-%m-%d')
```

**回放弹幕存储容量估算：**

| 存储层 | 数据范围 | 单条大小 | 单日数据量 | 存储成本 |
|--------|---------|---------|-----------|---------|
| Redis | 0-48h | ~200B（含 ZSET 开销） | 100 万 QPS × 86400s × 200B ≈ 17TB → 不现实 | 仅存热门房间，限制 10 万条/房间 |
| MySQL | 1-7 天 | ~150B（行存储） | 864 亿条/天 × 150B ≈ 13TB/天 → 不现实 | 仅存付费弹幕 + 采样普通弹幕 |
| HDFS | >7 天 | ~50B（Parquet 列存+压缩） | 全量归档，压缩后约 4TB/天 | 低成本 HDD 存储 |

**实际存储策略（折中方案）：**
- Redis 热数据：仅保留最近 2 小时的弹幕，每个房间最多 10 万条 → 10 万房间 × 10 万条 × 200B ≈ 200GB → 可接受
- MySQL 温数据：仅存储付费弹幕全量 + 普通弹幕采样（每秒每房间 1 条）→ 每日约 86 亿 × 150B ≈ 1.3TB → 可接受
- HDFS 冷数据：全量弹幕 Parquet 存储，Snappy 压缩后约 4TB/天 → 年存储成本约 ¥12 万

### 完整的弹幕流转时序图

```
普通弹幕：
  用户发送弹幕 → WS网关
    → Kafka(danmaku-normal, key=roomId)
    → 聚合服务消费(200ms窗口)
      → 如果热门直播间：采样聚合(保留20-25条)
      → 如果普通直播间：直接透传
    → Redis Pub/Sub(danmaku:room:{roomId})
    → 所有订阅该房间的网关
    → 各网关遍历本地WebSocket连接推送

付费弹幕（快速通道）：
  用户发送付费弹幕 → WS网关
    ├─ MySQL持久化(paid_danmaku表)
    ├─ Redis Pub/Sub(danmaku:room:{roomId}) ← 直推，不经Kafka
    └─ Kafka(danmaku-paid) ← 用于回放归档
  → 所有订阅该房间的网关推送
  → 客户端回执ACK → Redis记录送达状态

回放弹幕：
  用户进入回放 → 请求某时段弹幕
  → 回放服务查询HDFS(按room_id+时间范围)
  → 按时间戳排序返回
  → 客户端按视频进度逐条显示
```

### 弹幕限速与防刷

```python
class DanmakuRateLimiter:
    """弹幕发送限速"""

    def check_rate(self, user_id, room_id):
        """检查用户是否超过发送频率"""
        # 1. 全局频率：每用户每秒最多 3 条弹幕
        global_key = f"danmaku:rate:global:{user_id}"
        global_count = self.redis.incr(global_key)
        if global_count == 1:
            self.redis.expire(global_key, 1)  # 1秒窗口
        if global_count > 3:
            return False, "发送太频繁，请稍后再试"

        # 2. 房间频率：每用户每房间每秒最多 3 条
        room_key = f"danmaku:rate:room:{room_id}:{user_id}"
        room_count = self.redis.incr(room_key)
        if room_count == 1:
            self.redis.expire(room_key, 1)
        if room_count > 3:
            return False, "本直播间发言太频繁"

        # 3. 重复内容检测：同一内容 10 秒内不重复
        content_key = f"danmaku:dup:{user_id}:{hash(content)}"
        if self.redis.exists(content_key):
            return False, "请勿重复发送相同内容"
        self.redis.setex(content_key, 10, "1")

        return True, None
```

### 完整反刷系统（内容过滤 + 房间级限速 + AC 自动机关键词黑名单）

#### 多层防刷架构

```
弹幕请求 → 网关层
  ├─ 第1层：用户级限速（Redis 计数器，每用户每秒 3 条）
  ├─ 第2层：房间级限速（Redis 计数器，每房间总 QPS 上限）
  ├─ 第3层：内容预检（长度/格式/特殊字符过滤）
  ├─ 第4层：关键词黑名单（AC 自动机，O(n) 多模式匹配）
  ├─ 第5层：AI 内容审核（异步，延迟约 100ms，先放行后撤回）
  └─ 通过 → 写入 Kafka
```

#### 房间级限速实现

```python
class RoomRateLimiter:
    """房间级限速：防止单个房间被恶意刷屏"""

    def __init__(self, redis_client):
        self.redis = redis_client

        # 房间限速配置（按房间热度动态调整）
        self.room_rate_config = {
            "normal": {"max_qps": 100, "max_user_qps": 3},    # 普通房间：100 条/秒
            "hot": {"max_qps": 1000, "max_user_qps": 3},      # 热门房间：1000 条/秒
            "super_hot": {"max_qps": 10000, "max_user_qps": 5}, # 顶流房间：1 万条/秒
        }

    async def check_room_rate(self, room_id: str, user_id: str) -> tuple:
        """检查房间级别的发送限速"""

        # 获取房间热度等级
        room_tier = await self._get_room_tier(room_id)
        config = self.room_rate_config[room_tier]

        # 1. 房间总 QPS 限速（滑动窗口）
        room_key = f"danmaku:room_rate:{room_id}"
        now_ms = int(time.time() * 1000)
        window_ms = 1000  # 1 秒窗口

        pipe = self.redis.pipeline()
        pipe.zremrangebyscore(room_key, 0, now_ms - window_ms)  # 清理过期记录
        pipe.zcard(room_key)  # 当前窗口内计数
        pipe.zadd(room_key, {str(now_ms): now_ms})  # 加入当前请求
        pipe.expire(room_key, 2)

        results = await pipe.execute()
        current_count = results[1]

        if current_count > config["max_qps"]:
            return False, f"本直播间发言人数过多，请稍后再试"

        # 2. 用户在房间内的 QPS 限速（滑动窗口）
        user_key = f"danmaku:user_room_rate:{room_id}:{user_id}"
        pipe = self.redis.pipeline()
        pipe.zremrangebyscore(user_key, 0, now_ms - window_ms)
        pipe.zcard(user_key)
        pipe.zadd(user_key, {str(now_ms): now_ms})
        pipe.expire(user_key, 2)

        results = await pipe.execute()
        user_count = results[1]

        if user_count > config["max_user_qps"]:
            return False, "发言太频繁，请稍后再试"

        # 3. 用户全局 QPS 限速（跨房间）
        global_key = f"danmaku:global_rate:{user_id}"
        pipe = self.redis.pipeline()
        pipe.zremrangebyscore(global_key, 0, now_ms - window_ms)
        pipe.zcard(global_key)
        pipe.zadd(global_key, {str(now_ms): now_ms})
        pipe.expire(global_key, 2)

        results = await pipe.execute()
        global_count = results[1]

        if global_count > 5:  # 全局每秒最多 5 条
            return False, "操作过于频繁"

        return True, None

    async def _get_room_tier(self, room_id: str) -> str:
        """获取房间热度等级"""
        # 从 Redis 缓存读取房间在线人数
        viewer_count = await self.redis.get(f"room:viewers:{room_id}")
        if not viewer_count:
            return "normal"

        count = int(viewer_count)
        if count >= 50000:
            return "super_hot"
        elif count >= 5000:
            return "hot"
        else:
            return "normal"
```

#### AC 自动机关键词黑名单

```python
from collections import deque, defaultdict
from typing import List, Tuple, Optional


class ACAutomaton:
    """
    Aho-Corasick 自动机：多模式串匹配
    用于弹幕内容关键词过滤，一次扫描即可匹配所有黑名单词

    时间复杂度：构建 O(∑|pattern|)，匹配 O(|text| + |matches|)
    远优于逐个关键词匹配的 O(|text| × |patterns|)
    """

    class Node:
        __slots__ = ['children', 'fail', 'output', 'depth']

        def __init__(self):
            self.children = {}     # char -> Node
            self.fail = None       # 失配指针
            self.output = []       # 匹配到的关键词列表
            self.depth = 0         # 节点深度

    def __init__(self):
        self.root = self.Node()
        self._built = False

    def add_pattern(self, pattern: str, category: str = "default"):
        """添加关键词到自动机"""
        node = self.root
        for char in pattern:
            if char not in node.children:
                node.children[char] = self.Node()
                node.children[char].depth = node.depth + 1
            node = node.children[char]
        node.output.append({"pattern": pattern, "category": category})
        self._built = False  # 添加新词后需要重新构建 fail 指针

    def build(self):
        """构建 fail 指针（BFS）"""
        queue = deque()

        # 第一层节点的 fail 指向 root
        for char, child in self.root.children.items():
            child.fail = self.root
            queue.append(child)

        # BFS 构建后续层的 fail 指针
        while queue:
            current = queue.popleft()

            for char, child in current.children.items():
                # 从 current.fail 开始，沿着 fail 链查找
                fail_node = current.fail
                while fail_node and char not in fail_node.children:
                    fail_node = fail_node.fail

                if fail_node and char in fail_node.children:
                    child.fail = fail_node.children[char]
                else:
                    child.fail = self.root

                # 合并 fail 节点的 output（后缀匹配）
                child.output = child.output + child.fail.output

                queue.append(child)

        self._built = True

    def search(self, text: str) -> List[dict]:
        """在文本中搜索所有匹配的关键词"""
        if not self._built:
            self.build()

        matches = []
        node = self.root

        for i, char in enumerate(text):
            # 沿着 fail 链查找匹配
            while node and char not in node.children:
                node = node.fail

            if not node:
                node = self.root
                continue

            node = node.children[char]

            # 收集当前节点的所有匹配
            for match in node.output:
                matches.append({
                    "pattern": match["pattern"],
                    "category": match["category"],
                    "position": i - len(match["pattern"]) + 1,
                    "end": i,
                })

        return matches

    def contains_any(self, text: str) -> Tuple[bool, Optional[dict]]:
        """快速检查文本是否包含任何关键词（找到第一个即返回）"""
        if not self._built:
            self.build()

        node = self.root
        for i, char in enumerate(text):
            while node and char not in node.children:
                node = node.fail

            if not node:
                node = self.root
                continue

            node = node.children[char]

            if node.output:
                first_match = node.output[0]
                return True, {
                    "pattern": first_match["pattern"],
                    "category": first_match["category"],
                    "position": i - len(first_match["pattern"]) + 1,
                }

        return False, None

    def replace(self, text: str, replace_char: str = "*") -> str:
        """替换文本中的关键词为指定字符"""
        if not self._built:
            self.build()

        result = list(text)
        node = self.root

        for i, char in enumerate(text):
            while node and char not in node.children:
                node = node.fail

            if not node:
                node = self.root
                continue

            node = node.children[char]

            for match in node.output:
                start = i - len(match["pattern"]) + 1
                for j in range(start, i + 1):
                    result[j] = replace_char

        return "".join(result)


class DanmakuContentFilter:
    """弹幕内容过滤服务：集成 AC 自动机 + 规则引擎"""

    def __init__(self, redis_client):
        self.redis = redis_client
        self.ac = ACAutomaton()
        self.room_blacklist = {}  # room_id -> ACAutomaton（房间级自定义黑名单）
        self._init_default_blacklist()

    def _init_default_blacklist(self):
        """初始化全局默认黑名单"""
        # 色情类关键词
        porn_keywords = ["色情关键词1", "色情关键词2"]  # 实际为敏感词列表
        for kw in porn_keywords:
            self.ac.add_pattern(kw, category="porn")

        # 广告类关键词
        ad_keywords = ["加微信", "代刷", "低价出售", "私聊下单", "加V", " VX:"]
        for kw in ad_keywords:
            self.ac.add_pattern(kw, category="ad")

        # 辱骂类关键词
        abuse_keywords = ["脏话1", "脏话2"]  # 实际为敏感词列表
        for kw in abuse_keywords:
            self.ac.add_pattern(kw, category="abuse")

        # 政治类关键词
        political_keywords = ["政治敏感词1"]
        for kw in political_keywords:
            self.ac.add_pattern(kw, category="political")

        self.ac.build()

    def filter_content(self, content: str, room_id: str = None) -> tuple:
        """
        过滤弹幕内容
        返回: (is_blocked: bool, reason: str, filtered_content: str)
        """
        # 1. 基础格式检查
        if len(content) == 0:
            return True, "弹幕内容不能为空", content
        if len(content) > 200:
            return True, "弹幕内容超过200字", content

        # 2. 特殊字符过滤（防注入、防刷屏）
        if self._has_suspicious_chars(content):
            return True, "包含非法字符", content

        # 3. 全局黑名单匹配（AC 自动机）
        is_matched, match_info = self.ac.contains_any(content)
        if is_matched:
            # 根据类别采取不同策略
            category = match_info["category"]
            if category in ("porn", "political"):
                # 色情和政治类：直接拦截
                return True, f"内容违规({category})", content
            elif category in ("ad", "abuse"):
                # 广告和辱骂类：替换后放行
                filtered = self.ac.replace(content)
                return False, None, filtered

        # 4. 房间级自定义黑名单
        if room_id and room_id in self.room_blacklist:
            room_ac = self.room_blacklist[room_id]
            is_matched, match_info = room_ac.contains_any(content)
            if is_matched:
                filtered = room_ac.replace(content)
                return False, None, filtered

        # 5. 重复字符检测（刷屏检测）
        if self._is_spam_pattern(content):
            return True, "疑似刷屏", content

        return False, None, content

    def _has_suspicious_chars(self, content: str) -> bool:
        """检测可疑特殊字符"""
        # 零宽字符（常用于绕过过滤）
        zero_width_chars = ['​', '‌', '‍', '﻿']
        for char in zero_width_chars:
            if char in content:
                return True

        # 控制字符
        for char in content:
            if ord(char) < 0x20 and char not in ('\n', '\t'):
                return True

        return False

    def _is_spam_pattern(self, content: str) -> bool:
        """检测刷屏模式"""
        if len(content) < 4:
            return False

        # 同一字符重复超过 8 次
        from collections import Counter
        char_counts = Counter(content)
        if max(char_counts.values()) >= 8:
            return True

        # 短字符串重复超过 3 次
        for pattern_len in range(1, min(len(content) // 3 + 1, 10)):
            pattern = content[:pattern_len]
            if content.count(pattern) >= 4:
                return True

        return False

    async def reload_blacklist(self):
        """从 Redis 热加载黑名单（运营可实时更新，无需重启服务）"""
        # 从 Redis 读取最新黑名单
        blacklist_data = await self.redis.get("danmaku:blacklist:global")
        if blacklist_data:
            new_ac = ACAutomaton()
            for item in json.loads(blacklist_data):
                new_ac.add_pattern(item["keyword"], category=item["category"])
            new_ac.build()

            # 原子替换（线程安全）
            self.ac = new_ac
            logger.info(f"黑名单热加载完成，共 {len(json.loads(blacklist_data))} 个关键词")

    async def load_room_blacklist(self, room_id: str):
        """加载房间级自定义黑名单"""
        room_data = await self.redis.get(f"danmaku:blacklist:room:{room_id}")
        if room_data:
            room_ac = ACAutomaton()
            for item in json.loads(room_data):
                room_ac.add_pattern(item["keyword"], category=item["category"])
            room_ac.build()
            self.room_blacklist[room_id] = room_ac
```

**AC 自动机性能对比：**

| 方案 | 关键词数 | 匹配复杂度 | 100 字弹幕耗时 |
|------|---------|-----------|--------------|
| 逐个 str.contains() | 1 万 | O(text × patterns) | ~5ms |
| 正则表达式 | 1 万 | O(text × patterns) | ~3ms（编译缓存） |
| AC 自动机 | 1 万 | O(text + matches) | ~0.05ms |
| AC 自动机 | 10 万 | O(text + matches) | ~0.08ms |

AC 自动机的构建时间：1 万关键词约 50ms，10 万关键词约 500ms。构建后可长期使用，热加载时新建实例原子替换即可。

**黑名单关键词规模估算：**
- 色情类：约 2000 词
- 广告类：约 3000 词
- 辱骂类：约 5000 词
- 政治类：约 1000 词
- 总计：约 1.1 万关键词
- AC 自动机内存占用：约 5MB（可接受）

### 带宽优化

**下行带宽（弹幕推送给观众）：**
- 正常直播间：每 200ms 约 5 条弹幕 × 50 字节 = 1.25 KB/s/观众
- 热门直播间：每 200ms 约 25 条弹幕 × 50 字节 = 6.25 KB/s/观众
- 1000 万观众 × 平均 2 KB/s = 20 GB/s 出站带宽

**上行带宽（观众发弹幕）：**
- 100 万条/秒 × 50 字节 = 50 MB/s → 可忽略

**带宽优化策略：**

| 策略 | 效果 | 实现方式 |
|------|------|---------|
| 弹幕压缩 | 减少 50% 带宽 | gzip 压缩 WebSocket 消息 |
| 批量推送 | 减少帧开销 | 每 200ms 合并为一个 WS 帧 |
| 增量推送 | 减少重复数据 | 只推送新增弹幕 ID+内容 |
| 边缘分发 | 减少回源 | CDN 边缘节点缓存弹幕流 |

优化后带宽：20 GB/s → 约 8 GB/s（压缩 + 批量）→ 需要 CDN 或边缘节点分发。

#### 带宽详细计算

```
=== 下行带宽详细计算 ===

1. 单观众带宽（未优化）：
   - 普通房间（100 观众，5 条/秒弹幕）：
     5 条/秒 × 50 字节 = 250 B/s ≈ 0.25 KB/s
   - 热门房间（1 万观众，1 万条/秒弹幕，聚合后 25 条/200ms = 125 条/秒推送）：
     125 条/秒 × 50 字节 = 6.25 KB/s
   - 顶流房间（50 万观众，5 万条/秒弹幕，聚合后 25 条/200ms = 125 条/秒推送）：
     125 条/秒 × 50 字节 = 6.25 KB/s（聚合效果一样）

2. 总下行带宽（未优化）：
   假设观众分布：80% 在普通房间（800 万），15% 在热门房间（150 万），5% 在顶流房间（50 万）
   = 800 万 × 0.25 KB/s + 150 万 × 6.25 KB/s + 50 万 × 6.25 KB/s
   = 2 GB/s + 9.4 GB/s + 3.1 GB/s
   ≈ 14.5 GB/s（纯弹幕数据）

   加上 WebSocket 帧开销：
   - 每帧头：2-14 字节
   - 每 200ms 一帧 = 5 帧/秒
   - 1000 万连接 × 5 帧/秒 × 14 字节 ≈ 0.7 GB/s
   - 总计：14.5 + 0.7 ≈ 15.2 GB/s

3. 压缩后带宽：
   - permessage-deflate 压缩比：文本数据约 50-60% 压缩
   - 14.5 GB/s × 0.45 ≈ 6.5 GB/s
   - 帧开销在压缩模式下更小：≈ 0.3 GB/s
   - 总计：6.5 + 0.3 ≈ 6.8 GB/s

4. 协议优化后带宽：
   - 二进制协议替代 JSON：字段名用数字 ID（类似 Protobuf）
     {"id":"xxx","roomId":"xxx","userId":"xxx","content":"xxx","type":"normal","timestamp":123}
     → JSON 约 120 字节 → 二进制约 50 字节 → 再压缩约 25 字节
   - 6.8 GB/s × (25/60) ≈ 2.8 GB/s

5. 网关出站带宽分摊：
   - 200 台网关 × 2.8 GB/s ÷ 200 = 14 MB/s/网关
   - 单机万兆网卡（10 Gbps ≈ 1.25 GB/s）→ 远未打满
   - 实际峰值考虑热门房间不均匀：单网关峰值约 50 MB/s → 仍可承受

=== Redis Pub/Sub 带宽计算 ===

1. 写入端（聚合服务 → Redis）：
   - 10 万活跃房间 × 5 次/秒 × 聚合后平均 500 字节
   = 10 万 × 5 × 500 = 250 MB/s

2. 读取端（Redis → 各网关）：
   - 每条消息被平均 5 个网关订阅
   = 250 MB/s × 5 = 1.25 GB/s（Redis 出站）
   - 单 Redis 节点出站带宽：1.25 GB/s → 需要 10 Gbps 网络
   - 如果超出单节点能力 → 按 room_id 范围分片到多个 Redis 实例

3. Redis 内存占用估算：
   - 路由信息：10 万房间 × 5 网关 × 50 字节 = 25 MB
   - 用户连接信息：1000 万 × 100 字节 = 1 GB
   - 房间连接计数：10 万房间 × 200 网关 × 30 字节 = 600 MB
   - 限速计数器：100 万活跃用户 × 5 个 key × 50 字节 = 250 MB
   - 付费弹幕 ACK：1 万/小时 × 100 字节 = 1 MB
   - 总计：约 1.9 GB → 8c32G 节点绰绰有余

=== Kafka 集群带宽计算 ===

1. 写入带宽：
   - 普通弹幕：100 万条/秒 × 50 字节 = 50 MB/s
   - 付费弹幕：1 万条/秒 × 100 字节 = 1 MB/s
   - 回放数据：100 万条/秒 × 50 字节 = 50 MB/s（冗余副本）
   - 总写入：约 100 MB/s

2. 读取带宽：
   - 聚合服务消费：100 MB/s
   - 归档服务消费：50 MB/s
   - 总读取：约 150 MB/s

3. 副本带宽：
   - 3 副本 → 额外 200 MB/s 内部复制流量
   - 9 节点集群 → 每节点约 22 MB/s 复制流量

4. 存储容量：
   - 100 MB/s × 86400 秒 = 8.64 TB/天
   - 24 小时 retention → 约 8.64 TB
   - 9 节点 × 2 TB SSD → 18 TB 总容量 → 满足需求
```

#### 带宽优化具体实现

```python
class DanmakuProtocolOptimizer:
    """弹幕协议优化：二进制编码 + 增量推送"""

    # 消息类型枚举（1 字节）
    MSG_DANMAKU_BATCH = 0x01
    MSG_DANMAKU_SINGLE = 0x02
    MSG_PAID_DANMAKU = 0x03
    MSG_HEARTBEAT = 0x04
    MSG_ROOM_CHANGE = 0x05

    def encode_danmaku_batch(self, danmaku_list: List[dict]) -> bytes:
        """
        二进制编码弹幕批量消息

        格式：
          [1B msg_type][2B count][4B base_timestamp]
          重复 count 次：
            [4B timestamp_offset]  — 相对于 base_timestamp 的偏移（节省 4 字节）
            [1B type]              — 0=normal, 1=gift, 2=super
            [2B user_id_len][user_id_bytes]
            [2B content_len][content_bytes]

        示例：5 条弹幕，JSON 约 600 字节 → 二进制约 250 字节 → 压缩约 120 字节
        """
        if not danmaku_list:
            return b''

        buf = bytearray()
        buf.append(self.MSG_DANMAKU_BATCH)

        # count
        count = min(len(danmaku_list), 65535)
        buf.extend(count.to_bytes(2, 'big'))

        # base_timestamp（最小时间戳）
        base_ts = min(d['timestamp'] for d in danmaku_list)
        buf.extend(base_ts.to_bytes(4, 'big'))

        for d in danmaku_list[:count]:
            # timestamp_offset（相对偏移，通常 < 1000ms，2 字节足够）
            ts_offset = d['timestamp'] - base_ts
            buf.extend(ts_offset.to_bytes(2, 'big'))

            # type
            type_map = {"normal": 0, "gift": 1, "super": 2}
            buf.append(type_map.get(d.get("type", "normal"), 0))

            # user_id（UTF-8 编码，2 字节长度前缀）
            uid_bytes = d.get("userId", "").encode("utf-8")
            buf.extend(len(uid_bytes).to_bytes(2, 'big'))
            buf.extend(uid_bytes)

            # content（UTF-8 编码，2 字节长度前缀）
            content_bytes = d.get("content", "").encode("utf-8")
            buf.extend(len(content_bytes).to_bytes(2, 'big'))
            buf.extend(content_bytes)

        return bytes(buf)

    def decode_danmaku_batch(self, data: bytes) -> List[dict]:
        """解码二进制弹幕批量消息"""
        if not data or data[0] != self.MSG_DANMAKU_BATCH:
            return []

        offset = 1
        count = int.from_bytes(data[offset:offset+2], 'big')
        offset += 2

        base_ts = int.from_bytes(data[offset:offset+4], 'big')
        offset += 4

        type_map = {0: "normal", 1: "gift", 2: "super"}
        result = []

        for _ in range(count):
            ts_offset = int.from_bytes(data[offset:offset+2], 'big')
            offset += 2

            msg_type = type_map.get(data[offset], "normal")
            offset += 1

            uid_len = int.from_bytes(data[offset:offset+2], 'big')
            offset += 2
            user_id = data[offset:offset+uid_len].decode("utf-8")
            offset += uid_len

            content_len = int.from_bytes(data[offset:offset+2], 'big')
            offset += 2
            content = data[offset:offset+content_len].decode("utf-8")
            offset += content_len

            result.append({
                "timestamp": base_ts + ts_offset,
                "type": msg_type,
                "userId": user_id,
                "content": content,
            })

        return result


class IncrementalPushOptimizer:
    """增量推送优化：只推送新增弹幕"""

    def __init__(self, redis_client):
        self.redis = redis_client
        self.last_push_id: Dict[str, int] = {}  # room_id -> last_pushed_offset

    async def get_incremental(self, room_id: str) -> dict:
        """
        获取增量弹幕（只返回上次推送后的新增弹幕）

        对比全量推送的节省：
        - 全量：每 200ms 推送 25 条 × 50 字节 = 1250 字节
        - 增量：每 200ms 新增约 5 条 × 50 字节 = 250 字节
        - 节省 80%
        """
        key = f"danmaku:hot:{room_id}"

        # 获取上次推送的最新时间戳
        last_ts = self.last_push_id.get(room_id, 0)

        # 只查询新增的弹幕
        if last_ts > 0:
            new_items = await self.redis.zrangebyscore(
                key, f"({last_ts}", "+inf",  # 注意 '(' 表示开区间，不含 last_ts
                withscores=True,
            )
        else:
            # 首次：返回最近 30 条
            new_items = await self.redis.zrevrange(
                key, 0, 29, withscores=True
            )
            new_items = list(reversed(new_items))

        if new_items:
            # 更新最新时间戳
            _, latest_ts = new_items[-1]
            self.last_push_id[room_id] = latest_ts

        return {
            "roomId": room_id,
            "items": [json.loads(item) for item, _ in new_items],
            "count": len(new_items),
        }
```

**协议优化效果对比：**

| 方案 | 5 条弹幕大小 | 25 条弹幕大小 | 压缩率 |
|------|------------|------------|-------|
| JSON 原始 | ~600B | ~3000B | 基线 |
| JSON + permessage-deflate | ~270B | ~1350B | 55% ↓ |
| 二进制 + permessage-deflate | ~120B | ~500B | 83% ↓ |
| 二进制 + 增量 + 压缩 | ~60B | ~250B | 92% ↓ |

**Redis Pub/Sub 性能分析：**

| 维度 | 数值 |
|------|------|
| 频道数 | 最多 10 万（活跃直播间） |
| 每频道订阅者 | 平均 5 个网关 |
| 推送频率 | 5 次/秒/频道（200ms 窗口） |
| 总推送 QPS | 50 万/秒 |
| Redis 实例 | 3 节点 Sentinel（单节点可承受 50 万 QPS） |

## 常见陷阱（深度分析）

### 陷阱 1：逐条推送所有弹幕

**后果：** 热门直播间 1 万条/秒 × 1 万观众 = 1 亿次推送/秒。即使每次推送 50 字节，也需要 5 GB/s 的 Redis/网络带宽，远超单集群能力。

**解决方案：** 聚合 + 采样，每个窗口只推送 20-25 条。

### 陷阱 2：付费弹幕经过聚合

**后果：** 用户花 10 元发了一条超级弹幕，被聚合采样丢弃 → 弹幕没显示 → 用户投诉 → 退款 + 品牌损失。

**解决方案：** 付费弹幕走独立通道（直接 Pub/Sub + 数据库持久化 + 回执确认）。

### 陷阱 3：网关全量订阅所有房间

**问题：** 每台网关订阅 10 万个房间频道 → Redis Pub/Sub 连接维护 10 万个订阅 → 内存和 CPU 开销巨大。

**解决方案：** 只订阅本网关上有观众的房间。通常每台网关服务约 500-1000 个房间。

### 陷阱 4：无弹幕限速

**问题：** 恶意用户在 1 秒内发送 100 条弹幕（刷屏），影响其他用户体验。

**解决方案：** 网关层限速——每用户每直播间每秒最多 3 条弹幕。

## 延伸思考

- **弹幕互动**：弹幕投票（#选项1）、弹幕抽奖——需要解析弹幕内容并触发业务逻辑，与纯展示弹幕的架构差异在于需要消费弹幕并执行动作。

- **AI 弹幕过滤**：实时检测违规弹幕（色情、广告、辱骂）并拦截。在聚合层增加一步 AI 审核，延迟增加约 100ms。

```python
class AIDanmakuFilter:
    """AI 弹幕内容审核（异步处理，先放行后撤回）"""

    def __init__(self, model_service):
        self.model = model_service
        self.async_queue = asyncio.Queue(maxsize=10000)

    async def quick_check(self, danmaku: dict) -> tuple:
        """快速预检（本地 AC 自动机 + 规则，0 延迟）"""
        content = danmaku.get("content", "")

        # 本地关键词匹配（AC 自动机，~0.05ms）
        is_blocked, reason = self.ac_filter.contains_any(content)
        if is_blocked:
            return True, reason

        # 规则检查（长度、特殊字符等）
        if len(content) > 200 or self._has_suspicious_chars(content):
            return True, "内容违规"

        # 通过预检 → 放行，同时提交异步 AI 审核
        await self.async_queue.put(danmaku)
        return False, None

    async def async_ai_review(self):
        """异步 AI 审核（后台消费队列）"""
        while True:
            danmaku = await self.async_queue.get()

            try:
                # 调用 AI 模型（延迟约 50-100ms）
                result = await self.model.predict(danmaku["content"])

                if result["is_violation"]:
                    # AI 判定违规 → 发送撤回指令
                    await self.redis.publish(
                        f"danmaku:withdraw:{danmaku['roomId']}",
                        json.dumps({
                            "danmaku_id": danmaku["id"],
                            "reason": result["category"],
                        })
                    )

                    # 软删除
                    await self.db.soft_delete([danmaku["id"]], operator="ai_filter")

                    logger.info(f"AI 审核撤回弹幕: id={danmaku['id']}, "
                                 f"category={result['category']}")

            except Exception as e:
                logger.error(f"AI 审核异常: {e}")
```

**AI 审核与 AC 自动机的配合策略：**
- AC 自动机：确定性拦截（已知关键词，0 廞迟），拦截率约 70%
- AI 模型：语义级拦截（隐含违规，50-100ms 廞迟），拦截率约 95%
- 两级配合：AC 自动机快速拦截已知违规，AI 模型补充拦截新变体
- 先放行后撤回：不阻塞正常弹幕推送，AI 审核后违规弹幕发撤回指令
- 撤回延迟约 100ms：极少数观众可能短暂看到违规弹幕 → 用户体验可接受

- **VR 直播弹幕**：3D 空间中的弹幕定位——弹幕不再只是水平飘过，而是在 3D 空间中定位。需要客户端做 3D 渲染，服务端下发 3D 坐标。

- **弹幕互动架构扩展**：弹幕投票、弹幕抽奖需要消费弹幕并触发业务逻辑，架构差异在于需要独立消费组：

```
弹幕互动消费链：
  Kafka(danmaku-normal) → 互动服务（独立消费组 danmaku-interaction）
    → 解析弹幕内容（#选项1 → 投票选项）
    → 触发业务逻辑（更新投票计数 / 抽奖池）
    → 推送互动结果（投票进度条 / 抽奖结果）

注意：互动服务是独立消费组，不影响聚合服务的消费速度
互动服务消费延迟可以更大（1-2 秒可接受），因为互动结果不需要实时弹幕级延迟
```

## 异常场景完整演练

### 场景 1：Redis Pub/Sub 集群宕机

```
触发：Redis 主节点故障 → Sentinel 自动切换 → 切换期间 10-30 秒不可用
影响：所有弹幕无法分发 → 观众看不到新弹幕
处理：
  1. Kafka 仍在消费和聚合弹幕 → 弹幕不丢失（Kafka 有持久化）
  2. 聚合服务检测到 Redis 不可用 → 将弹幕写入本地缓冲队列
  3. Redis 恢复后 → 重放缓冲队列中的弹幕
  4. 网关检测到 Pub/Sub 断连 → 保留当前连接，不主动断开观众
  5. Redis 恢复后 → 网关重新订阅 → 弹幕恢复正常
  6. 付费弹幕在 Redis 不可用期间 → 通过 Kafka 备用通道推送
关键：Kafka 是弹幕的持久化缓冲，Redis 只是实时分发通道
```

### 场景 2：热门直播间突发流量

```
触发：顶流主播开播 → 5 分钟内观众从 10 万涨到 200 万
影响：
  - 弹幕量从 1000/秒 暴增到 10 万/秒
  - 该房间的 Kafka partition 成为热点
  - 聚合服务 CPU 飙升
处理：
  1. 弹幕聚合窗口从 200ms 缩短到 100ms → 减少每次推送的弹幕量
  2. 采样率从 保留25条 降至 15条 → 减少推送量
  3. 热门直播间 partition 自动扩展（从 1 分区扩展到 4 分区）
  4. 聚合服务自动扩容（K8s HPA → 从 3 个 Pod 扩到 10 个）
  5. 付费弹幕不受采样影响，独立推送
监控指标：弹幕延迟从 300ms 升至 800ms → 可接受
```

### 场景 3：网关滚动升级

```
触发：网关服务需要升级（安全补丁）
挑战：滚动升级时每台网关上有 5 万 WebSocket 连接，不能全部断开
处理：
  1. 逐台升级：每次下线 1 台网关
  2. 下线前：
     a. 通知该网关上的客户端"即将重连"
     b. 停止接收新的 WebSocket 连接
     c. 等待现有连接自然结束（最长 30 秒超时）
     d. 超时后强制断开 → 客户端自动重连到其他网关
  3. 其他网关的负载增加约 5%（1/20）
  4. 升级完成后重新上线 → 负载均衡自动分配新连接
  5. 全部 20 台升级完成约 20 分钟
  6. 整个过程用户感知：约 3-5 秒的弹幕暂停（重连期间）
```
### 场景 4：Kafka 消费延迟导致弹幕堆积

```
触发：聚合服务某 Pod OOM 重启 → 消费停止 5 分钟 → Kafka 堆积 300 万条弹幕
影响：恢复后需消化 5 分钟的积压 → 观众突然看到大量历史弹幕刷屏
处理：
  1. 恢复消费后，不是逐条处理，而是批量聚合
  2. 对堆积的弹幕：只保留每个 200ms 窗口中采样后的 20-25 条
  3. 5 分钟 × 5次/秒 × 25条 = 3750 条 → 3-5 秒内推送完
  4. 推送时标记为"历史弹幕"→ 客户端可选择快速滚动或忽略
  5. 付费弹幕在积压期间已通过独立通道推送（不受影响）
  6. 监控：消费 lag 恢复到 0 → 系统正常
预防：聚合服务配置 HPA + PDB（Pod Disruption Budget）
```

#### Kafka 消费延迟处理代码实现

```python
class ConsumerLagHandler:
    """Kafka 消费延迟（Lag）处理：堆积恢复策略"""

    def __init__(self, aggregator, config):
        self.aggregator = aggregator
        self.config = config
        self.lag_threshold = config.get('lag_threshold', 50000)  # 5 万条以上视为堆积
        self.is_catching_up = False

    def on_lag_detected(self, current_lag: int):
        """检测到消费延迟"""
        if current_lag < self.lag_threshold:
            if self.is_catching_up:
                # 延迟已恢复 → 切回正常模式
                self.is_catching_up = False
                self.aggregator.window_ms = 200  # 恢复 200ms 窗口
                self.aggregator.max_display_per_window = 25  # 恢复 25 条
                logger.info(f"消费延迟恢复，切回正常模式")
            return

        if self.is_catching_up:
            return  # 已在追赶模式

        # 进入追赶模式
        self.is_catching_up = True
        logger.warning(f"检测到消费延迟: lag={current_lag}, 进入追赶模式")

        # 调整策略：加速消费
        self.aggregator.window_ms = 500   # 窗口扩大到 500ms（批量处理更多）
        self.aggregator.max_display_per_window = 15  # 减少每窗口推送量

    def process_backlog(self, messages: list) -> list:
        """
        处理堆积消息：按时间窗口分组 + 采样
        不逐条处理，而是按窗口批量聚合后只推送采样结果
        """
        if not messages:
            return []

        # 按时间窗口分组
        window_groups = defaultdict(list)
        for msg in messages:
            danmaku = json.loads(msg.value())
            ts = danmaku["timestamp"]
            window_key = ts // self.aggregator.window_ms  # 整除得到窗口编号
            window_groups[window_key].append(danmaku)

        # 每个窗口只采样 15-20 条
        sampled_results = []
        for window_key in sorted(window_groups.keys()):
            items = window_groups[window_key]
            if len(items) > self.aggregator.max_display_per_window:
                # 使用聚合采样
                sampled = self.aggregator._aggregate_high_volume(items)
            else:
                sampled = items

            # 标记为历史弹幕（客户端可区分展示）
            for item in sampled:
                item["isBacklog"] = True

            sampled_results.extend(sampled)

        return sampled_results
```

#### Kafka 消费延迟监控与告警

```
监控指标：
  - consumer_lag: 消费者组各分区的 lag 值
    采集方式：kafka-consumer-groups --describe
    告警阈值：lag > 5 万 → P2 告警，lag > 50 万 → P1 告警

  - consumer_rate: 消费速率（条/秒）
    正常值：每实例 12.5 万/秒
    告警阈值：< 5 万/秒 → 消费能力下降

  - aggregation_latency: 聚合推送延迟
    正常值：< 300ms
    告警阈值：> 1 秒 → P2 告警，> 3 秒 → P1 告警

  - kafka_disk_usage: Kafka 磁盘使用率
    告警阈值：> 80% → P2，> 90% → P1

自动恢复流程：
  1. 检测 lag > 阈值 → 切换到追赶模式
  2. 追赶模式下：扩大聚合窗口 + 减少推送量 + 增加 max.poll.records
  3. 如果 lag 持续增长 → HPA 扩容聚合服务
  4. lag 恢复到 0 → 切回正常模式
```

### 场景 5：WebSocket 连接风暴

```
触发：大型电竞赛事决赛开始 → 30 秒内 500 万用户同时涌入
影响：
  - 网关服务器瞬间收到大量 WebSocket 连接请求
  - 某几台网关负载飙升 → 部分连接超时 → 用户重试 → 雪崩
  - Redis Pub/Sub 订阅操作暴增 → Redis CPU 飙升
  - 路由注册请求暴增 → Redis 写入瓶颈
处理：
  1. 接入层限流：
     - 负载均衡器设置新连接速率限制（每秒每网关最多 1000 个新连接）
     - 超出限制返回 503 → 客户端指数退避重试（1s, 2s, 4s, 8s...）
  2. 房间级连接分散：
     - 同一房间的连接分散到不同网关（一致性哈希 + 虚拟节点）
     - 避免单网关承载过多同一房间的连接
  3. Redis 批量操作优化：
     - 路由注册使用 Pipeline（每批 100 个操作）
     - 订阅操作合并（先收集 1 秒内的订阅请求，批量 subscribe）
  4. 网关弹性扩容：
     - 监控连接数增长率 → 预测 5 分钟后连接数
     - 提前扩容网关（从 200 台扩到 300 台）
     - 使用 K8s HPA + 自定义指标（WebSocket 连接数）
恢复时间：首次涌入约 2 分钟后稳定
预防：大型赛事前提前扩容 + 压测
```

#### WebSocket 连接风暴防护代码

```python
class ConnectionStormProtector:
    """WebSocket 连接风暴防护"""

    def __init__(self, redis_client, config):
        self.redis = redis_client
        self.max_new_conn_per_sec = config.get('max_new_conn_per_sec', 1000)
        self.max_room_conn_per_gateway = config.get('max_room_conn_per_gateway', 2000)

    async def check_admission(self, gateway_id: str, room_id: str) -> tuple:
        """检查是否允许新连接接入"""
        now = int(time.time())

        # 1. 网关级新连接速率检查
        rate_key = f"ws:admission_rate:{gateway_id}:{now}"
        rate = await self.redis.incr(rate_key)
        if rate == 1:
            await self.redis.expire(rate_key, 2)
        if rate > self.max_new_conn_per_sec:
            return False, "服务器繁忙，请稍后重试"

        # 2. 房间在单网关上的连接数检查
        count_key = f"ws:room_count:{room_id}:{gateway_id}"
        count = await self.redis.get(count_key)
        if count and int(count) > self.max_room_conn_per_gateway:
            # 该房间在此网关上连接过多 → 重定向到其他网关
            return False, "redirect"  # 客户端收到 redirect 后更换网关

        return True, None

    async def batch_register_routes(self, connections: List[dict]):
        """批量注册路由（Pipeline 优化，应对连接风暴）"""
        pipe = self.redis.pipeline()

        subscribe_rooms = set()

        for conn in connections:
            gateway_id = conn["gateway_id"]
            room_id = conn["room_id"]
            user_id = conn["user_id"]
            conn_id = conn["connection_id"]

            pipe.sadd(f"ws:room:{room_id}", gateway_id)
            pipe.sadd(f"ws:gateway:{gateway_id}", room_id)
            pipe.incr(f"ws:room_count:{room_id}:{gateway_id}")
            pipe.set(f"ws:user:{user_id}", json.dumps({
                "gateway_id": gateway_id,
                "room_id": room_id,
                "connection_id": conn_id,
            }))

            # 检查是否需要订阅
            subscribe_rooms.add((gateway_id, room_id))

        await pipe.execute()

        # 批量订阅 Redis Pub/Sub
        for gateway_id, room_id in subscribe_rooms:
            count = await self.redis.get(f"ws:room_count:{room_id}:{gateway_id}")
            if count and int(count) == 1:
                # 首次有观众 → 订阅
                await self.gateway_subscribe(gateway_id, room_id)
```

### 场景 6：Redis 内存压力

```
触发：运营活动导致在线用户暴增 → Redis 路由数据从 2GB 涨到 8GB → 接近 maxmemory
影响：
  - Redis 内存不足 → 开始驱逐 key → 路由信息丢失
  - 限速计数器被驱逐 → 限速失效 → 刷屏攻击
  - 用户连接信息被驱逐 → 无法定向推送付费弹幕
处理：
  1. 紧急扩容 Redis：从 32GB 实例升级到 64GB（云平台在线扩容，约 10 分钟）
  2. 设置 key 过期策略：
     - 限速计数器：已设置 1-2 秒 TTL → 自动淘汰
     - 用户连接信息：设置 5 分钟 TTL + 心跳续期
     - 路由信息：设置 1 小时 TTL + 定期刷新
  3. 内存淘汰策略：allkeys-lru → 优先淘汰限速计数器
     但路由信息不能被淘汰 → 将路由数据迁移到单独的 Redis 实例
  4. 数据拆分：
     - Redis-1（路由专用）：8c64G，maxmemory-policy=noeviction
     - Redis-2（限速专用）：4c8G，maxmemory-policy=allkeys-lru
     - Redis-3（Pub/Sub 专用）：8c32G，Pub/Sub 不占常规内存
预防：
  - 监控 Redis 内存使用率，> 70% 告警
  - 定期清理僵尸连接的路由信息
  - 路由数据与临时数据分实例
```

### 场景 7：付费弹幕投递失败与重试

```
触发：用户发送付费超级弹幕 → Redis Pub/Sub 推送 → 部分网关未收到（网络抖动）
影响：部分观众看不到付费弹幕 → 用户投诉（花了钱却没效果）
处理流程（完整）：
  1. 付费弹幕发送后，写入 paid_danmaku 表，状态为 pending
  2. 推送到 Redis Pub/Sub → 返回订阅者数量
  3. 如果订阅者数量 < 该房间的网关数 → 推送不完整
     → 将缺失的网关 ID 列表记录到 Redis
     → 5 秒后重试推送（此时网关可能已恢复）
  4. 客户端 ACK 机制：
     → 每个网关收到付费弹幕后，统计本地该房间的连接数
     → 网关上报 ACK：{danmaku_id, gateway_id, delivered_count, total_count}
     → 付费弹幕服务汇总所有网关的 ACK
     → 如果 delivered_count == total_count → 标记 delivered
  5. 超时未 ACK：
     → 5 秒后检查：如果 delivered_count < total_count → 重试
     → 最多重试 3 次（间隔 5s, 10s, 30s）
  6. 3 次重试后仍未完全送达：
     → 标记 delivery_status = 'partial_delivered'
     → 通知用户"部分观众可能未收到"
     → 不退款（已尽力送达，且多数观众已收到）
  7. 完全无法送达（0 个订阅者，3 次重试后仍 0 订阅者）：
     → 标记 delivery_status = 'failed'
     → 自动发起退款
     → 通知用户"发送失败，已退款"
```

#### 付费弹幕投递完整实现

```python
class PaidDanmakuDeliveryService:
    """付费弹幕投递服务：完整的投递确认与重试机制"""

    def __init__(self, db, redis, kafka_producer):
        self.db = db
        self.redis = redis
        self.kafka = kafka_producer
        self.max_retries = 3
        self.retry_intervals = [5, 10, 30]  # 重试间隔（秒）

    async def send_paid_danmaku(self, danmaku: dict) -> dict:
        """发送付费弹幕（入口方法）"""
        room_id = danmaku["roomId"]
        danmaku_id = danmaku["id"]

        # 1. 持久化到数据库（状态：pending）
        await self.db.execute(
            """INSERT INTO paid_danmaku
               (danmaku_id, room_id, user_id, content, type, price, delivery_status, created_at)
               VALUES (%s, %s, %s, %s, %s, %s, 'pending', NOW())""",
            [danmaku_id, room_id, danmaku["userId"], danmaku["content"],
             danmaku["type"], danmaku["price"]]
        )

        # 2. 推送到 Redis Pub/Sub
        subscribers = await self.redis.publish(
            f"danmaku:room:{room_id}", json.dumps(danmaku)
        )

        # 3. 获取该房间应有的网关数
        expected_gateways = await self.redis.scard(f"ws:room:{room_id}")

        # 4. 记录投递状态
        delivery_key = f"danmaku:delivery:{danmaku_id}"
        delivery_info = {
            "danmaku_id": danmaku_id,
            "room_id": room_id,
            "expected_gateways": expected_gateways,
            "acked_gateways": 0,
            "subscribers_on_send": subscribers,
            "status": "pending",
            "retry_count": 0,
            "created_at": int(time.time() * 1000),
        }
        await self.redis.setex(delivery_key, 300, json.dumps(delivery_info))

        # 5. 写入 Kafka（归档通道）
        await self.kafka.produce("danmaku-paid", danmaku, key=room_id)

        # 6. 安排 ACK 检查
        if subscribers < expected_gateways:
            # 推送不完整 → 立即安排重试
            await self._schedule_retry(danmaku_id, delay=self.retry_intervals[0])
        else:
            # 推送完整 → 等待 ACK
            await self._schedule_ack_check(danmaku_id, timeout=5)

        return {
            "danmaku_id": danmaku_id,
            "status": "sent",
            "subscribers": subscribers,
        }

    async def on_gateway_ack(self, danmaku_id: str, gateway_id: str,
                              delivered_count: int, total_count: int):
        """网关上报付费弹幕 ACK"""
        delivery_key = f"danmaku:delivery:{danmaku_id}"

        delivery_json = await self.redis.get(delivery_key)
        if not delivery_json:
            logger.warning(f"ACK 到达但投递记录已过期: {danmaku_id}")
            return

        delivery_info = json.loads(delivery_json)
        delivery_info["acked_gateways"] += 1

        # 记录每个网关的 ACK 详情
        ack_detail_key = f"danmaku:ack_detail:{danmaku_id}"
        await self.redis.hset(ack_detail_key, gateway_id, json.dumps({
            "delivered_count": delivered_count,
            "total_count": total_count,
            "acked_at": int(time.time() * 1000),
        }))
        await self.redis.expire(ack_detail_key, 300)

        # 检查是否全部网关已 ACK
        if delivery_info["acked_gateways"] >= delivery_info["expected_gateways"]:
            delivery_info["status"] = "delivered"

            # 更新数据库状态
            await self.db.execute(
                "UPDATE paid_danmaku SET delivery_status = 'delivered', "
                "delivered_at = NOW() WHERE danmaku_id = %s",
                [danmaku_id]
            )

            logger.info(f"付费弹幕投递完成: {danmaku_id}, "
                        f"网关数={delivery_info['acked_gateways']}")
        else:
            # 未全部 ACK → 继续等待
            pass

        await self.redis.setex(delivery_key, 300, json.dumps(delivery_info))

    async def _schedule_ack_check(self, danmaku_id: str, timeout: int = 5):
        """安排 ACK 检查"""
        # 使用延迟队列或定时任务
        await self.redis.zadd(
            "danmaku:ack_check_queue",
            {danmaku_id: int(time.time()) + timeout}
        )

    async def _schedule_retry(self, danmaku_id: str, delay: int):
        """安排重试推送"""
        await self.redis.zadd(
            "danmaku:retry_queue",
            {danmaku_id: int(time.time()) + delay}
        )

    async def process_retry_queue(self):
        """处理重试队列（定时扫描）"""
        now = int(time.time())

        # 获取到期的重试任务
        items = await self.redis.zrangebyscore(
            "danmaku:retry_queue", 0, now, start=0, num=100
        )

        for danmaku_id in items:
            delivery_key = f"danmaku:delivery:{danmaku_id}"
            delivery_json = await self.redis.get(delivery_key)

            if not delivery_json:
                # 投递记录已过期 → 清理
                await self.redis.zrem("danmaku:retry_queue", danmaku_id)
                continue

            delivery_info = json.loads(delivery_json)
            retry_count = delivery_info.get("retry_count", 0)

            if retry_count >= self.max_retries:
                # 超过最大重试次数
                await self._handle_delivery_failed(danmaku_id, delivery_info)
                await self.redis.zrem("danmaku:retry_queue", danmaku_id)
                continue

            # 重试推送
            room_id = delivery_info["room_id"]
            danmaku = await self.db.fetchone(
                "SELECT * FROM paid_danmaku WHERE danmaku_id = %s", [danmaku_id]
            )

            subscribers = await self.redis.publish(
                f"danmaku:room:{room_id}", json.dumps(danmaku)
            )

            delivery_info["retry_count"] = retry_count + 1
            delivery_info["subscribers_on_send"] = subscribers
            await self.redis.setex(delivery_key, 300, json.dumps(delivery_info))

            if subscribers > 0:
                # 有订阅者 → 等待 ACK
                next_delay = self.retry_intervals[min(retry_count + 1, len(self.retry_intervals) - 1)]
                await self._schedule_ack_check(danmaku_id, timeout=next_delay)
            else:
                # 仍然无订阅者 → 继续重试
                next_delay = self.retry_intervals[min(retry_count + 1, len(self.retry_intervals) - 1)]
                await self._schedule_retry(danmaku_id, delay=next_delay)

            await self.redis.zrem("danmaku:retry_queue", danmaku_id)

    async def _handle_delivery_failed(self, danmaku_id: str, delivery_info: dict):
        """处理投递失败"""
        expected = delivery_info.get("expected_gateways", 0)
        acked = delivery_info.get("acked_gateways", 0)

        if acked == 0:
            # 完全无法送达 → 退款
            await self.db.execute(
                "UPDATE paid_danmaku SET delivery_status = 'failed' "
                "WHERE danmaku_id = %s", [danmaku_id]
            )
            await self._initiate_refund(danmaku_id)
            logger.error(f"付费弹幕投递失败（完全无法送达）: {danmaku_id}")

        elif acked < expected:
            # 部分送达 → 标记 partial_delivered，不退款
            await self.db.execute(
                "UPDATE paid_danmaku SET delivery_status = 'partial_delivered', "
                "delivered_at = NOW() WHERE danmaku_id = %s", [danmaku_id]
            )
            logger.warning(f"付费弹幕部分送达: {danmaku_id}, "
                           f"acked={acked}/{expected}")

    async def _initiate_refund(self, danmaku_id: str):
        """发起退款"""
        danmaku = await self.db.fetchone(
            "SELECT * FROM paid_danmaku WHERE danmaku_id = %s", [danmaku_id]
        )
        if not danmaku:
            return

        # 调用支付服务退款
        refund_result = await self.payment_service.refund(
            order_id=danmaku["order_id"],
            amount=danmaku["price"],
            reason="弹幕投递失败",
        )

        if refund_result["success"]:
            await self.db.execute(
                "UPDATE paid_danmaku SET delivery_status = 'refunded', "
                "refunded_at = NOW() WHERE danmaku_id = %s", [danmaku_id]
            )
            # 通知用户
            await self.notification_service.notify(
                user_id=danmaku["user_id"],
                type="refund",
                message=f"您的超级弹幕发送失败，已退款 {danmaku['price']} 元",
            )
        else:
            logger.error(f"退款失败: danmaku_id={danmaku_id}, "
                         f"reason={refund_result.get('reason')}")
            # 人工处理队列
            await self.redis.rpush("danmaku:manual_refund", danmaku_id)
```

**付费弹幕投递状态机：**

```
pending → delivered（全部网关 ACK）
pending → partial_delivered（部分网关 ACK，超时后）
pending → failed（0 个网关 ACK，3 次重试后）
failed → refunded（自动退款成功）
```

| 状态 | 含义 | 是否退款 | 用户体验 |
|------|------|---------|---------|
| pending | 已发送，等待 ACK | 否 | 弹幕已显示 |
| delivered | 全部送达 | 否 | 正常 |
| partial_delivered | 部分送达 | 否 | 多数观众可见 |
| failed | 投递失败 | 是 | 收到退款通知 |
| refunded | 已退款 | 是 | 收到退款通知 |

## 性能与成本分析

**系统规模估算（1000 万观众、1 万直播间）：**

| 组件 | 规格 | 数量 | 月成本 |
|------|------|------|-------|
| WebSocket 网关 | 8c16G | 200 台 | ¥40 万 |
| 聚合服务 | 4c8G | 20 台 | ¥4 万 |
| Kafka 集群 | 8c32G + 2TB SSD | 9 节点 | ¥9 万 |
| Redis Pub/Sub | 8c32G | 3 节点 Sentinel | ¥3 万 |
| MySQL（弹幕存储） | 8c32G | 主从 2 台 | ¥2 万 |
| HDFS（回放存储） | 4c8G + 50TB HDD | 10 节点 | ¥5 万 |
| **合计** | | | **¥63 万** |

**关键性能指标：**

| 指标 | 目标 | 实际 |
|------|------|------|
| 弹幕端到端延迟 | < 500ms | 200-400ms |
| 付费弹幕延迟 | < 200ms | 50-100ms |
| WebSocket 连接数 | 1000 万 | 单机 5 万 × 200 台 |
| 弹幕写入 QPS | 100 万/秒 | Kafka 200 万/秒 |
| 弹幕推送 QPS | 5000 万/秒 | Redis Pub/Sub + 网关本地推送 |
| 付费弹幕送达率 | 99.99% | 有 ACK 重试机制 |

**成本优化空间：**
- 弹幕冷数据归档 HDFS → MySQL 只存 7 天热数据 → 存储成本降 60%
- 非热门直播间（观众 < 100）合并聚合 → 聚合服务资源降 30%
- 网关弹性扩缩（夜间缩到 100 台）→ 网关成本降 25%

### 量化性能分析

#### 弹幕端到端延迟分解

```
普通弹幕端到端延迟（200ms 窗口）：
  ┌─────────────────────────────────────────────────┐
  │ 用户发送 → WS网关接收          ≈ 10ms         │
  │ WS网关 → Kafka 写入            ≈ 5ms          │
  │ Kafka → 聚合服务消费            ≈ 10ms         │
  │ 聚合服务缓冲等待（200ms窗口）   ≈ 100-200ms    │ ← 主要延迟
  │ 聚合服务 → Redis Pub/Sub       ≈ 2ms          │
  │ Redis → 各网关接收              ≈ 5ms          │
  │ 网关 → 客户端推送              ≈ 10ms          │
  ├─────────────────────────────────────────────────┤
  │ 总计                          ≈ 142-242ms     │
  └─────────────────────────────────────────────────┘

付费弹幕端到端延迟（快速通道）：
  ┌─────────────────────────────────────────────────┐
  │ 用户发送 → WS网关接收          ≈ 10ms         │
  │ WS网关 → MySQL 写入            ≈ 5ms          │
  │ WS网关 → Redis Pub/Sub 直推    ≈ 2ms          │
  │ Redis → 各网关接收              ≈ 5ms          │
  │ 网关 → 客户端推送              ≈ 10ms          │
  ├─────────────────────────────────────────────────┤
  │ 总计                          ≈ 32ms          │
  └─────────────────────────────────────────────────┘
```

#### 各组件性能瓶颈与余量

```
组件             │ 当前负载       │ 瓶颈容量      │ 余量
────────────────┼───────────────┼───────────────┼──────
Kafka 写入       │ 100 万条/秒   │ 200 万条/秒   │ 100%
Kafka 存储       │ 8.64 TB/天    │ 18 TB         │ 108%
聚合服务 CPU     │ 60%           │ 80%           │ 33%
Redis Pub/Sub    │ 50 万 QPS     │ 100 万 QPS    │ 100%
Redis 内存       │ 1.9 GB        │ 32 GB         │ 1589%
网关连接数       │ 5 万/台       │ 8 万/台       │ 60%
网关出站带宽     │ 14 MB/s/台    │ 1250 MB/s/台  │ 8821%
MySQL 写入       │ 1 万 TPS      │ 5 万 TPS      │ 400%
```

#### 弹幕推送 QPS 详细分解

```
弹幕推送总量 = ∑(各房间观众数 × 各房间聚合后推送频率)

简化模型：
  - 1 万活跃房间，平均每房间 1000 观众
  - 聚合后推送频率：5 次/秒（200ms 窗口）
  - 每次推送约 25 条弹幕

推送操作：
  1. Redis Pub/Sub 发布：1 万房间 × 5 次/秒 = 5 万 QPS
  2. Redis → 网关分发：5 万 QPS × 平均 5 个网关/房间 = 25 万 QPS
  3. 网关 → 客户端推送：
     - 每网关约 500 房间 × 5 次/秒 × 100 观众/房间 = 25 万推送/秒/网关
     - 200 台网关总计：5000 万推送/秒

各环节 QPS：
  Redis Pub/Sub 发布 QPS：5 万/秒
  Redis Pub/Sub 投递 QPS：25 万/秒
  网关本地推送 QPS：5000 万/秒（分布式的，单机 25 万/秒）
```

#### Redis 内存详细分解

```
Redis 实例内存分配（8c32G，可用约 28 GB）：

1. 路由数据（约 600 MB）：
   - ws:room:{roomId}：10 万 × 5 成员 × 50 字节 = 25 MB
   - ws:gateway:{gatewayId}：200 × 500 成员 × 50 字节 = 5 MB
   - ws:room_count:{roomId}:{gatewayId}：10 万 × 5 × 30 字节 = 15 MB
   - ws:user:{userId}：1000 万 × 100 字节 = 1 GB
   小计：约 1.05 GB

2. 限速计数器（约 150 MB，短生命周期）：
   - 全局限速：100 万活跃用户 × 30 字节 = 30 MB
   - 房间限速：100 万 × 5 个房间 × 30 字节 = 150 MB（但 TTL 短，实际占用 ~50 MB）
   - 重复检测：50 万 × 30 字节 = 15 MB
   小计：约 95 MB

3. 付费弹幕投递状态（约 5 MB）：
   - danmaku:delivery:{id}：1 万/小时 × 300 字节 = 3 MB
   - danmaku:ack:{id}：1 万 × 50 字节 = 0.5 MB
   小计：约 3.5 MB

4. 热数据弹幕（约 200 GB，需要独立 Redis 实例）：
   - danmaku:hot:{roomId}：10 万房间 × 平均 1 万条 × 200 字节 = 200 GB
   → 热数据存储需要独立的 Redis Cluster（5 主 5 从，每节点 64 GB）

5. Pub/Sub 缓冲区（约 200 MB）：
   - 每个订阅者维护独立输出缓冲区
   - 200 网关 × 平均 500 频道 × 1MB 缓冲 = 100 GB → 过大！
   → 优化：控制输出缓冲区上限（client-output-buffer-limit）
   → 设为 256MB/客户端 → 200 × 256MB = 50 GB → 仍过大
   → 实际优化：Pub/Sub 使用独立 Redis 实例 + 分片

内存规划结论：
  - 路由 + 限速 + 投递：需要 1 台 8c32G Redis（实际占用 ~2 GB）
  - 热数据弹幕：需要 5 主 5 从 Redis Cluster（每节点 64 GB）
  - Pub/Sub：需要 3 台 8c32G Sentinel（实际内存占用低，但 CPU/带宽需求高）
```

#### Kafka 分区容量规划

```
Topic: danmaku-normal（64 分区）

写入容量：
  - 峰值 100 万 QPS ÷ 64 = 1.56 万条/秒/分区
  - 单条 50 字节 → 0.78 MB/s/分区
  - Kafka 单分区写入上限：10-15 MB/s → 余量 13-19 倍

存储容量：
  - 100 万条/秒 × 86400 秒 = 864 亿条/天
  - 864 亿 × 50 字节 = 4.32 TB/天（未压缩）
  - Kafka 启用 LZ4 压缩 → 压缩比约 2:1 → 2.16 TB/天
  - 24 小时 retention → 总存储 2.16 TB
  - 9 节点 × 2 TB SSD → 18 TB 总容量 → 余量 8.3 倍

消费能力：
  - 聚合服务 8 实例 × 8 分区/实例 = 64 分区
  - 单实例消费：12.5 万条/秒 → CPU 约 60%
  - 总消费能力：100 万条/秒 → 匹配写入速度
  - HPA 扩容阈值：CPU > 70% 或消费 lag > 5 万

Topic: danmaku-paid（16 分区，3 副本）
  - 峰值 1 万 QPS → 每分区 625 条/秒 → 极低负载
  - 但需要高可靠性 → 3 副本 + min.insync.replicas=2
  - 72 小时 retention → 1 万 × 86400 × 3 × 100 字节 ≈ 26 GB
```

#### 成本优化详细方案

| 优化项 | 当前成本 | 优化后成本 | 节省 | 方案 |
|--------|---------|-----------|------|------|
| 网关弹性扩缩 | ¥40 万 | ¥30 万 | 25% | 夜间缩到 100 台，白天 200 台 |
| 非热门房间合并聚合 | ¥4 万 | ¥2.8 万 | 30% | 观众<100 的房间共享聚合线程 |
| MySQL 冷数据归档 | ¥2 万 | ¥0.8 万 | 60% | 7 天以上数据归档 HDFS |
| Kafka 存储优化 | ¥9 万 | ¥7 万 | 22% | LZ4 压缩 + 缩短 retention 到 12h |
| Redis 热数据按需加载 | ¥3 万 | ¥2 万 | 33% | 非活跃房间热数据不缓存 |
| **合计** | **¥63 万** | **¥44.6 万** | **29%** | |

## 弹幕渲染引擎完整实现

```python
class DanmakuRenderer:
    """弹幕渲染引擎：碰撞检测 + 轨道分配 + 密度控制"""

    MAX_DANMAKU_PER_SCREEN = 50  # 同屏最大弹幕数
    TRACK_HEIGHT = 40  # 轨道高度（像素）
    SCREEN_HEIGHT = 720  # 屏幕高度
    MAX_TRACKS = SCREEN_HEIGHT // TRACK_HEIGHT  # 18 条轨道

    def allocate_track(self, danmaku):
        """为弹幕分配轨道（避免碰撞）"""
        # 1. 计算弹幕显示时长
        text_width = len(danmaku["text"]) * 14  # 14px per char
        speed = (960 + text_width) / danmaku.get("duration", 8)  # px/s

        # 2. 找到不碰撞的轨道
        for track_id in range(self.MAX_TRACKS):
            if self._is_track_available(track_id, text_width, speed):
                return track_id

        # 3. 无可用轨道 → 随机分配（允许少量重叠）
        return random.randint(0, self.MAX_TRACKS - 1)

    def _is_track_available(self, track_id, text_width, speed):
        """检查轨道是否可用（不会与已有弹幕碰撞）"""
        existing = self.redis.lrange(f"track:{track_id}", 0, -1)

        for item in existing:
            d = json.loads(item)
            # 计算是否会在水平方向重叠
            time_diff = (now() - datetime.fromisoformat(d["start_time"])).total_seconds()
            existing_right_edge = d["start_x"] + time_diff * d["speed"]
            new_left_edge = 960  # 从右边缘进入

            if existing_right_edge > new_left_edge - text_width * 0.3:
                return False  # 会碰撞

        return True

    def filter_by_density(self, danmaku_list):
        """密度控制：同屏弹幕过多时过滤"""
        if len(danmaku_list) <= self.MAX_DANMAKU_PER_SCREEN:
            return danmaku_list

        # 按优先级排序：VIP > 普通用户
        danmaku_list.sort(key=lambda d: d.get("priority", 0), reverse=True)
        return danmaku_list[:self.MAX_DANMAKU_PER_SCREEN]
```

## 弹幕审核系统

```python
class DanmakuModerationService:
    """弹幕审核：AI 实时过滤 + 关键词 + 用户信任等级"""

    def moderate(self, danmaku):
        """审核弹幕"""
        user = self.db.get_user(danmaku["user_id"])
        trust_level = user.get("trust_level", "new")

        # 1. 高信任用户 → 直接发布
        if trust_level == "trusted":
            return {"action": "publish", "reason": "trusted_user"}

        # 2. 关键词过滤（精确 + 模糊匹配）
        keyword_result = self.keyword_filter.check(danmaku["text"])
        if keyword_result["matched"]:
            return {"action": "reject", "reason": "keyword",
                    "matched_words": keyword_result["words"]}

        # 3. AI 内容安全检测
        safety_result = self.ai_safety.check(danmaku["text"])
        if safety_result["toxicity"] > 0.8:
            return {"action": "reject", "reason": "toxic",
                    "score": safety_result["toxicity"]}
        if safety_result["toxicity"] > 0.5:
            return {"action": "review", "reason": "suspicious",
                    "score": safety_result["toxicity"]}

        # 4. 新用户 → 进入审核队列
        if trust_level == "new":
            return {"action": "review", "reason": "new_user"}

        return {"action": "publish", "reason": "passed_all_checks"}
```

## 礼物打赏集成

```python
class GiftRewardService:
    """礼物打赏：礼物类型 + 动画 + 结算"""

    GIFT_TYPES = {
        "flower": {"price": 1, "animation": "petal_fall", "danmaku_template": "{user} 送了一朵花"},
        "rocket": {"price": 100, "animation": "rocket_launch", "danmaku_template": "{user} 发射了火箭"},
        "crown": {"price": 500, "animation": "crown_shine", "danmaku_template": "{user} 送上皇冠"},
        "castle": {"price": 5000, "animation": "castle_build", "danmaku_template": "{user} 建了一座城堡"},
    }

    def send_gift(self, user_id, anchor_id, gift_type, quantity=1):
        """送礼物"""
        gift = self.GIFT_TYPES[gift_type]
        total_price = gift["price"] * quantity

        # 1. 扣减用户余额
        balance = self.wallet.get_balance(user_id)
        if balance < total_price:
            return {"status": "insufficient_balance"}

        self.wallet.deduct(user_id, total_price)

        # 2. 连击检测（3 秒内同礼物）
        combo_key = f"gift_combo:{user_id}:{anchor_id}:{gift_type}"
        combo_count = self.redis.incr(combo_key)
        if combo_count == 1:
            self.redis.expire(combo_key, 3)
        is_combo = combo_count > 1

        # 3. 触发动画
        self.animation_service.trigger(anchor_id, {
            "type": gift["animation"],
            "combo": combo_count if is_combo else 1,
            "user_id": user_id
        })

        # 4. 弹幕通知
        danmaku_text = gift["danmamu_template"].format(
            user=self._get_display_name(user_id))
        if is_combo:
            danmaku_text += f" ×{combo_count}"
        self.danmaku_service.publish_system_danmaku(anchor_id, danmaku_text)

        # 5. 更新打赏榜
        self.redis.zincrby(f"gift_leaderboard:{anchor_id}",
            total_price, user_id)

        # 6. 主播收益结算（50% 分成）
        anchor_income = total_price * 0.5
        self.wallet.credit(anchor_id, anchor_income)

        return {"status": "success", "combo": combo_count,
                "anchor_income": anchor_income}

    def get_leaderboard(self, anchor_id, limit=10):
        """获取打赏榜"""
        return self.redis.zrevrange(f"gift_leaderboard:{anchor_id}",
            0, limit - 1, withscores=True)
```

## 异常场景补充

### 场景：弹幕渲染卡顿

```
触发：高密度弹幕同时出现 → 渲染帧率下降 → 用户卡顿
检测：
  1. 同屏弹幕 > 50 条 → 渲染压力大
  2. 用户反馈"弹幕卡顿" → 性能问题
处理：
  1. 触发密度控制：过滤低优先级弹幕
  2. 降级：关闭弹幕动画效果
  3. 极端情况：限制弹幕为固定位置（不滚动）
预防：密度控制 + 动画降级 + Web Worker 渲染
```

### 场景：审核误判飙升

```
触发：AI 审核模型更新后 → 误判率从 2% 升到 15%
检测：
  1. 用户申诉率 > 10% → 误判严重
  2. 审核队列积压 → 告警
处理：
  1. 回滚到上一个审核模型版本
  2. 标记新模型的误判结果 → 重新审核
  3. 误判弹幕恢复显示
预防：模型灰度发布 + 误判率监控 + 快速回滚
```

## 弹幕存储与回放完整实现

```python
class DanmakuStorageService:
    """弹幕存储：写入优化 + 回放加载"""

    def batch_write(self, room_id, danmaku_list):
        """批量写入弹幕（异步落库）"""
        # 1. 先写入 Redis（实时查询用）
        pipe = self.redis.pipeline()
        for d in danmaku_list:
            pipe.zadd(f"danmaku:{room_id}", {
                json.dumps(d): d["timestamp_ms"]
            })
        pipe.execute()

        # 2. 异步写入数据库（通过 Kafka）
        for d in danmaku_list:
            self.kafka.produce("danmaku_persist", json.dumps({
                "room_id": room_id,
                "danmaku_id": d["id"],
                "user_id": d["user_id"],
                "text": d["text"],
                "timestamp_ms": d["timestamp_ms"],
                "type": d.get("type", "scroll"),
                "color": d.get("color", "#FFFFFF"),
            }))

    def load_for_replay(self, room_id, video_id, start_ms, end_ms):
        """加载回放弹幕（按时间范围）"""
        # 1. 从 Redis 加载（最近 24 小时）
        danmaku_list = self.redis.zrangebyscore(
            f"danmaku:{room_id}", start_ms, end_ms)

        if not danmaku_list:
            # 2. 从数据库加载（历史数据）
            danmaku_list = self.db.query(
                "SELECT * FROM danmaku WHERE room_id = %s "
                "AND timestamp_ms BETWEEN %s AND %s "
                "ORDER BY timestamp_ms ASC",
                room_id, start_ms, end_ms)

        return [json.loads(d) if isinstance(d, str) else d
                for d in danmaku_list]

    def compact_historical_data(self, room_id, before_date):
        """压缩历史弹幕数据"""
        # 1. 按分钟聚合（保留每分钟的弹幕数量和摘要）
        self.db.execute(
            "INSERT INTO danmaku_minute_agg (room_id, minute_start, count, sample_texts) "
            "SELECT room_id, "
            "FROM_UNIXTIME(FLOOR(timestamp_ms/60000)*60) as minute_start, "
            "COUNT(*) as count, "
            "SUBSTRING_INDEX(GROUP_CONCAT(text ORDER BY timestamp_ms SEPARATOR '||'), '||', 3) "
            "FROM danmaku WHERE room_id = %s AND DATE(FROM_UNIXTIME(timestamp_ms/1000)) < %s "
            "GROUP BY minute_start "
            "ON DUPLICATE KEY UPDATE count = VALUES(count)",
            room_id, before_date)

        # 2. 删除已聚合的原始数据
        self.db.execute(
            "DELETE FROM danmaku WHERE room_id = %s "
            "AND DATE(FROM_UNIXTIME(timestamp_ms/1000)) < %s",
            room_id, before_date)
```

## 直播间弹幕流量控制

```python
class DanmakuFlowController:
    """弹幕流量控制：防止弹幕风暴"""

    def control_flow(self, room_id, danmaku):
        """弹幕流量控制"""
        # 1. 房间级速率限制
        room_key = f"danmaku_rate:{room_id}"
        room_count = self.redis.incr(room_key)
        if room_count == 1:
            self.redis.expire(room_key, 1)
        if room_count > 500:  # 每秒 500 条上限
            return {"action": "drop", "reason": "room_rate_limit"}

        # 2. 用户级速率限制
        user_key = f"danmaku_user_rate:{room_id}:{danmaku['user_id']}"
        user_count = self.redis.incr(user_key)
        if user_count == 1:
            self.redis.expire(user_key, 1)
        if user_count > 5:  # 每人每秒 5 条上限
            return {"action": "drop", "reason": "user_rate_limit"}

        # 3. 相同内容去重
        content_key = f"danmaku_dedup:{room_id}:{danmaku['text'][:20]}"
        if self.redis.exists(content_key):
            return {"action": "drop", "reason": "duplicate_content"}
        self.redis.setex(content_key, 2, "1")  # 2 秒去重窗口

        # 4. 超流量时降级：只显示 VIP 弹幕
        if room_count > 300:
            if danmaku.get("user_level", 0) < 5:
                return {"action": "delay", "reason": "high_traffic_delay"}

        return {"action": "pass"}
```

## 异常场景补充

### 场景：弹幕数据写入延迟

```
触发：Kafka 消费者 lag 增长 → 弹幕落库延迟 → 回放时弹幕缺失
检测：
  1. Kafka consumer lag > 10000 → 告警
  2. 弹幕落库延迟 > 30 秒 → 严重告警
处理：
  1. 增加消费者实例
  2. 回放优先从 Redis 读取（实时数据）
  3. Redis 数据不足 → 显示"弹幕加载中"
预防：消费者自动扩容 + Redis 缓存兜底
```

### 场景：弹幕服务故障导致直播中断

```
触发：弹幕服务 OOM → 崩溃 → 直播间无法发送弹幕
检测：
  1. 弹幕服务健康检查失败 → 告警
  2. 用户反馈无法发弹幕 → 严重告警
处理：
  1. 弹幕服务与直播流解耦（弹幕故障不影响视频流）
  2. 弹幕服务快速重启
  3. 重启后用户可正常发送（故障期间弹幕丢失可接受）
预防：弹幕与视频流解耦 + 弹幕服务独立部署 + 快速重启
```

## 直播间状态管理

### 直播间生命周期

直播间的状态机：`created → preparing → live → ended → archived`，每个状态有严格的准入和转出规则。

```
状态流转：
  created   → 主播点击"开始直播" → preparing
  preparing → 推流成功/超时     → live / created（推流失败回退）
  live      → 主播点击"结束直播" / 推流中断超时 → ended
  ended     → 24小时后          → archived

不可逆规则：
  - archived 不可回退到任何状态
  - ended 只能前进到 archived
  - live 状态下才能接收弹幕和礼物
```

### 完整直播间状态管理代码

```python
import enum
import time
import uuid
from dataclasses import dataclass, field
from typing import Optional, Dict, List, Set


class RoomState(enum.Enum):
    """直播间状态枚举"""
    CREATED = "created"         # 已创建，未开播
    PREPARING = "preparing"     # 准备中（推流连接建立中）
    LIVE = "live"               # 直播中
    ENDED = "ended"             # 已结束
    ARCHIVED = "archived"       # 已归档


class RoomRole(enum.Enum):
    """直播间角色枚举"""
    OWNER = "owner"             # 主播
    MODERATOR = "moderator"     # 房管
    VIP = "vip"                 # VIP 用户
    VIEWER = "viewer"           # 普通观众


# 状态转换白名单：只有允许的转换才能执行
ALLOWED_TRANSITIONS = {
    (RoomState.CREATED, RoomState.PREPARING),
    (RoomState.PREPARING, RoomState.LIVE),
    (RoomState.PREPARING, RoomState.CREATED),      # 推流失败回退
    (RoomState.LIVE, RoomState.ENDED),
    (RoomState.ENDED, RoomState.ARCHIVED),
}

# 各状态下允许的操作
STATE_PERMISSIONS = {
    RoomState.CREATED: {"edit_config", "delete_room"},
    RoomState.PREPARING: {"cancel_live"},
    RoomState.LIVE: {"send_danmaku", "send_gift", "kick_user",
                     "mute_user", "ban_user", "edit_config", "end_live"},
    RoomState.ENDED: {"replay_danmaku", "export_data"},
    RoomState.ARCHIVED: {"replay_danmaku"},
}


@dataclass
class RoomConfig:
    """直播间配置"""
    room_id: str
    danmaku_enabled: bool = True           # 弹幕开关
    gift_enabled: bool = True              # 礼物开关
    subscriber_only: bool = False          # 仅订阅者可发言
    danmaku_level_limit: int = 0           # 弹幕等级限制（0=无限制）
    max_danmaku_per_sec: int = 500         # 每秒弹幕上限
    auto_end_minutes: int = 480            # 自动结束时长（8小时）


@dataclass
class RoomBanRecord:
    """封禁记录"""
    user_id: str
    banned_at: float
    duration_s: int            # 封禁时长（秒），0=永久
    reason: str
    banned_by: str             # 操作人


class LiveRoomStateManager:
    """直播间状态管理器"""

    # 直播间超时配置
    PREPARING_TIMEOUT_S = 30          # 准备阶段超时 30 秒
    STREAM_INTERRUPT_TIMEOUT_S = 60   # 推流中断超时 60 秒
    AUTO_ARCHIVE_S = 86400            # 结束后 24 小时自动归档

    def __init__(self, redis, db):
        self.redis = redis
        self.db = db
        # 内存缓存：room_id → 状态信息
        self._room_cache: Dict[str, dict] = {}
        # 封禁记录：room_id → {user_id → BanRecord}
        self._ban_records: Dict[str, Dict[str, RoomBanRecord]] = {}
        # 静音记录：room_id → {user_id → unmute_time}
        self._mute_records: Dict[str, Dict[str, float]] = {}

    def transition_state(self, room_id: str, target: RoomState,
                         operator_id: str = None) -> dict:
        """状态转换（严格校验）"""
        current = self._get_state(room_id)
        if current is None:
            return {"ok": False, "error": "room_not_found"}

        # 校验转换合法性
        if (current, target) not in ALLOWED_TRANSITIONS:
            return {"ok": False, "error": "invalid_transition",
                    "from": current.value, "to": target.value}

        # 特殊校验：只有主播可以结束直播
        if target == RoomState.ENDED:
            if operator_id and not self._is_owner(room_id, operator_id):
                return {"ok": False, "error": "permission_denied"}

        # 执行状态转换
        now = time.time()
        self._set_state(room_id, target, now)

        # 状态转换后的副作用
        if target == RoomState.PREPARING:
            self._start_preparing_timer(room_id)
        elif target == RoomState.LIVE:
            self._on_live_start(room_id)
        elif target == RoomState.ENDED:
            self._on_live_end(room_id)
        elif target == RoomState.ARCHIVED:
            self._on_archived(room_id)

        return {"ok": True, "from": current.value, "to": target.value}

    def check_permission(self, room_id: str, action: str) -> bool:
        """检查当前状态下是否允许某操作"""
        state = self._get_state(room_id)
        if state is None:
            return False
        return action in STATE_PERMISSIONS.get(state, set())

    # ── 并发观众计数（Redis HyperLogLog）──

    def track_viewer_join(self, room_id: str, user_id: str):
        """观众加入直播间，用 HyperLogLog 计数"""
        hll_key = f"room_viewers:hll:{room_id}"
        self.redis.pfadd(hll_key, str(user_id))
        # 同时维护在线集合（用于精确查询当前在线）
        online_key = f"room_viewers:online:{room_id}"
        self.redis.sadd(online_key, user_id)
        self.redis.expire(online_key, 300)  # 5 分钟 TTL

        # 更新峰值
        current_count = self.redis.scard(online_key)
        peak_key = f"room_viewers:peak:{room_id}"
        peak = self.redis.get(peak_key)
        if peak is None or current_count > int(peak):
            self.redis.set(peak_key, current_count)

    def track_viewer_leave(self, room_id: str, user_id: str):
        """观众离开直播间"""
        online_key = f"room_viewers:online:{room_id}"
        self.redis.srem(online_key, user_id)

    def get_concurrent_viewers(self, room_id: str) -> int:
        """获取当前并发观众数"""
        online_key = f"room_viewers:online:{room_id}"
        return self.redis.scard(online_key)

    def get_total_unique_viewers(self, room_id: str) -> int:
        """获取累计独立观众数（HyperLogLog 近似值）"""
        hll_key = f"room_viewers:hll:{room_id}"
        return self.redis.pfcount(hll_key)

    def get_peak_viewers(self, room_id: str) -> int:
        """获取峰值观众数"""
        peak_key = f"room_viewers:peak:{room_id}"
        val = self.redis.get(peak_key)
        return int(val) if val else 0

    # ── 房间角色管理 ──

    def _is_owner(self, room_id: str, user_id: str) -> bool:
        """判断是否为主播"""
        room = self._room_cache.get(room_id)
        return room and room.get("owner_id") == user_id

    def get_user_role(self, room_id: str, user_id: str) -> RoomRole:
        """获取用户在直播间的角色"""
        if self._is_owner(room_id, user_id):
            return RoomRole.OWNER
        mod_key = f"room_moderators:{room_id}"
        if self.redis.sismember(mod_key, user_id):
            return RoomRole.MODERATOR
        vip_key = f"room_vips:{room_id}"
        if self.redis.sismember(vip_key, user_id):
            return RoomRole.VIP
        return RoomRole.VIEWER

    def assign_moderator(self, room_id: str, user_id: str,
                         operator_id: str) -> dict:
        """设置房管（仅主播可操作）"""
        if not self._is_owner(room_id, operator_id):
            return {"ok": False, "error": "permission_denied"}
        mod_key = f"room_moderators:{room_id}"
        self.redis.sadd(mod_key, user_id)
        return {"ok": True}

    def revoke_moderator(self, room_id: str, user_id: str,
                         operator_id: str) -> dict:
        """撤销房管"""
        if not self._is_owner(room_id, operator_id):
            return {"ok": False, "error": "permission_denied"}
        mod_key = f"room_moderators:{room_id}"
        self.redis.srem(mod_key, user_id)
        return {"ok": True}

    # ── 踢出/封禁/禁言操作 ──

    def kick_user(self, room_id: str, user_id: str,
                  operator_id: str) -> dict:
        """踢出用户（立即离开，但可以重新进入）"""
        role = self.get_user_role(room_id, operator_id)
        if role not in (RoomRole.OWNER, RoomRole.MODERATOR):
            return {"ok": False, "error": "permission_denied"}
        target_role = self.get_user_role(room_id, user_id)
        if target_role == RoomRole.OWNER:
            return {"ok": False, "error": "cannot_kick_owner"}
        # 踢出：从在线集合移除
        online_key = f"room_viewers:online:{room_id}"
        self.redis.srem(online_key, user_id)
        # 通知 WebSocket 网关断开该用户连接
        self._notify_kick(room_id, user_id)
        return {"ok": True}

    def ban_user(self, room_id: str, user_id: str, duration_s: int,
                 reason: str, operator_id: str) -> dict:
        """封禁用户（指定时长，0=永久，封禁期间不可进入）"""
        role = self.get_user_role(room_id, operator_id)
        if role not in (RoomRole.OWNER, RoomRole.MODERATOR):
            return {"ok": False, "error": "permission_denied"}
        target_role = self.get_user_role(room_id, user_id)
        if target_role in (RoomRole.OWNER, RoomRole.MODERATOR):
            return {"ok": False, "error": "cannot_ban_mod"}

        record = RoomBanRecord(
            user_id=user_id,
            banned_at=time.time(),
            duration_s=duration_s,
            reason=reason,
            banned_by=operator_id,
        )
        if room_id not in self._ban_records:
            self._ban_records[room_id] = {}
        self._ban_records[room_id][user_id] = record

        # Redis 存储（持久化 + 快速查询）
        ban_key = f"room_ban:{room_id}:{user_id}"
        if duration_s > 0:
            self.redis.setex(ban_key, duration_s, reason)
        else:
            self.redis.set(ban_key, reason)  # 永久封禁

        # 同时踢出
        self.kick_user(room_id, user_id, operator_id)
        return {"ok": True}

    def check_ban(self, room_id: str, user_id: str) -> Optional[str]:
        """检查用户是否被封禁，返回封禁原因或 None"""
        ban_key = f"room_ban:{room_id}:{user_id}"
        reason = self.redis.get(ban_key)
        return reason

    def mute_user(self, room_id: str, user_id: str, duration_s: int,
                  operator_id: str) -> dict:
        """禁言用户（可观看但不可发弹幕）"""
        role = self.get_user_role(room_id, operator_id)
        if role not in (RoomRole.OWNER, RoomRole.MODERATOR):
            return {"ok": False, "error": "permission_denied"}

        mute_key = f"room_mute:{room_id}:{user_id}"
        self.redis.setex(mute_key, duration_s, "1")
        if room_id not in self._mute_records:
            self._mute_records[room_id] = {}
        self._mute_records[room_id][user_id] = time.time() + duration_s
        return {"ok": True}

    def check_mute(self, room_id: str, user_id: str) -> bool:
        """检查用户是否被禁言"""
        mute_key = f"room_mute:{room_id}:{user_id}"
        return self.redis.exists(mute_key)

    # ── 内部辅助方法 ──

    def _get_state(self, room_id: str) -> Optional[RoomState]:
        """获取当前状态"""
        state_key = f"room_state:{room_id}"
        val = self.redis.get(state_key)
        if val:
            return RoomState(val)
        # 回退到内存缓存
        room = self._room_cache.get(room_id)
        return RoomState(room["state"]) if room else None

    def _set_state(self, room_id: str, state: RoomState, ts: float):
        """设置状态（Redis + 内存双写）"""
        state_key = f"room_state:{room_id}"
        self.redis.set(state_key, state.value)
        if room_id in self._room_cache:
            self._room_cache[room_id]["state"] = state.value
            self._room_cache[room_id]["state_changed_at"] = ts

    def _start_preparing_timer(self, room_id: str):
        """准备阶段超时定时器"""
        timer_key = f"room_preparing_timer:{room_id}"
        self.redis.setex(timer_key, self.PREPARING_TIMEOUT_S, "1")
        # 延迟检查：超时后自动回退到 created
        # （生产环境由定时任务扫描过期 key）

    def _on_live_start(self, room_id: str):
        """开播后初始化"""
        # 初始化 HyperLogLog
        hll_key = f"room_viewers:hll:{room_id}"
        self.redis.delete(hll_key)
        # 初始化峰值计数
        peak_key = f"room_viewers:peak:{room_id}"
        self.redis.set(peak_key, 0)
        # 记录开播时间
        self.redis.set(f"room_live_start:{room_id}", time.time())

    def _on_live_end(self, room_id: str):
        """下播后清理"""
        # 保存直播时长
        start = self.redis.get(f"room_live_start:{room_id}")
        if start:
            duration = time.time() - float(start)
            self.redis.set(f"room_duration:{room_id}", duration)
        # 保留在线集合供回放统计，24小时后自动过期
        online_key = f"room_viewers:online:{room_id}"
        self.redis.expire(online_key, 86400)
        # 设置自动归档定时器
        archive_key = f"room_archive_timer:{room_id}"
        self.redis.setex(archive_key, self.AUTO_ARCHIVE_S, "1")

    def _on_archived(self, room_id: str):
        """归档后清理所有实时数据"""
        keys_to_delete = [
            f"room_state:{room_id}",
            f"room_viewers:online:{room_id}",
            f"room_viewers:hll:{room_id}",
            f"room_viewers:peak:{room_id}",
            f"room_live_start:{room_id}",
            f"room_duration:{room_id}",
        ]
        for key in keys_to_delete:
            self.redis.delete(key)
        self._room_cache.pop(room_id, None)

    def _notify_kick(self, room_id: str, user_id: str):
        """通知 WebSocket 网关踢出用户"""
        # 发布到 Redis Pub/Sub，由网关订阅处理
        self.redis.publish(
            "room_kick",
            f'{{"room_id":"{room_id}","user_id":"{user_id}"}}'
        )
```

### 直播间配置管理

```python
class RoomConfigManager:
    """直播间配置管理器"""

    # 配置项及默认值
    CONFIG_DEFAULTS = {
        "danmaku_enabled": True,
        "gift_enabled": True,
        "subscriber_only": False,
        "danmaku_level_limit": 0,
        "max_danmaku_per_sec": 500,
        "auto_end_minutes": 480,
    }

    # 配置项的允许范围
    CONFIG_RANGES = {
        "danmaku_level_limit": (0, 30),
        "max_danmaku_per_sec": (50, 2000),
        "auto_end_minutes": (30, 1440),
    }

    def __init__(self, redis, db):
        self.redis = redis
        self.db = db

    def get_config(self, room_id: str) -> dict:
        """获取直播间配置"""
        config_key = f"room_config:{room_id}"
        cached = self.redis.hgetall(config_key)
        if cached:
            return {k: self._parse_value(k, v) for k, v in cached.items()}
        # 从数据库加载
        row = self.db.query(
            "SELECT * FROM room_configs WHERE room_id = %s", room_id)
        if row:
            config = dict(row[0])
            # 写入 Redis 缓存
            self.redis.hmset(config_key, config)
            self.redis.expire(config_key, 3600)
            return config
        return dict(self.CONFIG_DEFAULTS)

    def update_config(self, room_id: str, updates: dict,
                      operator_id: str) -> dict:
        """更新直播间配置"""
        # 权限校验
        state_key = f"room_state:{room_id}"
        state = self.redis.get(state_key)
        if state not in ("created", "live", "preparing"):
            return {"ok": False, "error": "cannot_update_in_current_state"}

        # 范围校验
        for key, value in updates.items():
            if key in self.CONFIG_RANGES:
                lo, hi = self.CONFIG_RANGES[key]
                if not (lo <= value <= hi):
                    return {"ok": False, "error": f"{key}_out_of_range",
                            "range": [lo, hi]}

        # 更新数据库
        self.db.update("room_configs", updates, {"room_id": room_id})
        # 更新缓存
        config_key = f"room_config:{room_id}"
        self.redis.hmset(config_key, updates)
        # 发布配置变更事件
        self.redis.publish("room_config_change",
            f'{{"room_id":"{room_id}","updates":{updates}}}')
        return {"ok": True}

    def _parse_value(self, key: str, raw_value: str):
        """解析配置值类型"""
        if key in ("danmaku_enabled", "gift_enabled", "subscriber_only"):
            return raw_value.lower() in ("true", "1", "yes")
        if key in ("danmaku_level_limit", "max_danmaku_per_sec",
                    "auto_end_minutes"):
            return int(raw_value)
        return raw_value
```

## 弹幕统计与分析

### 弹幕统计聚合服务

```python
import math
import re
from collections import Counter, defaultdict
from datetime import datetime, timedelta


class DanmakuAnalyticsService:
    """弹幕统计与分析服务"""

    # TF-IDF 关键词提取的最小文档频率
    MIN_DF = 5
    # 关键词提取数量
    TOP_KEYWORDS_COUNT = 20
    # 情感分析的正/负面词库
    POSITIVE_WORDS = {"好", "棒", "牛", "厉害", "666", "赞", "好看",
                      "喜欢", "爱", "哈哈", "优秀", "加油", "支持",
                      "感动", "精彩", "太强了", "无敌", "绝了"}
    NEGATIVE_WORDS = {"差", "烂", "垃圾", "无聊", "假", "骗", "恶心",
                      "拉胯", "离谱", "恶心", "太差", "退钱", "差评",
                      "不行", "难看", "失望", "尴尬"}

    def __init__(self, redis, db, clickhouse):
        self.redis = redis
        self.db = db
        self.ch = clickhouse  # ClickHouse 用于分析查询

    # ── 每分钟弹幕量图表 ──

    def get_minute_volume(self, room_id: str,
                          duration_minutes: int = 60) -> list:
        """获取每分钟弹幕量"""
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(minutes=duration_minutes)

        rows = self.ch.query("""
            SELECT
                toStartOfMinute(created_at) AS minute,
                count() AS volume
            FROM danmaku_events
            WHERE room_id = %(room_id)s
              AND created_at >= %(start)s
              AND created_at < %(end)s
            GROUP BY minute
            ORDER BY minute
        """, {"room_id": room_id, "start": start_time, "end": end_time})

        # 填充空缺分钟
        result = []
        current = start_time
        data_map = {r["minute"]: r["volume"] for r in rows}
        while current < end_time:
            minute_key = current.replace(second=0, microsecond=0)
            result.append({
                "time": minute_key.isoformat(),
                "volume": data_map.get(minute_key, 0),
            })
            current += timedelta(minutes=1)
        return result

    # ── 热门关键词提取（TF-IDF）──

    def extract_top_keywords(self, room_id: str,
                             hours: int = 1) -> list:
        """提取热门关键词（TF-IDF 算法）"""
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(hours=hours)

        # 获取时间段内所有弹幕文本
        rows = self.ch.query("""
            SELECT text FROM danmaku_events
            WHERE room_id = %(room_id)s
              AND created_at >= %(start)s
            """, {"room_id": room_id, "start": start_time})

        # 按分钟分组为"文档"
        docs = defaultdict(list)
        for row in rows:
            minute = datetime.utcnow().strftime("%Y%m%d%H%M")
            tokens = self._tokenize(row["text"])
            docs[minute].extend(tokens)

        if not docs:
            return []

        # 计算 TF-IDF
        N = len(docs)
        df = Counter()  # 文档频率
        for doc_tokens in docs.values():
            unique_tokens = set(doc_tokens)
            for token in unique_tokens:
                df[token] += 1

        # 过滤低频词
        df = {k: v for k, v in df.items() if v >= self.MIN_DF}

        # 计算所有文档的平均 TF-IDF
        token_scores = defaultdict(float)
        for doc_tokens in docs.values():
            tf = Counter(doc_tokens)
            total = len(doc_tokens)
            for token, count in tf.items():
                if token in df:
                    idf = math.log(N / df[token]) + 1
                    token_scores[token] += (count / total) * idf

        # 排序取 top N
        sorted_keywords = sorted(
            token_scores.items(), key=lambda x: x[1], reverse=True
        )[:self.TOP_KEYWORDS_COUNT]

        return [{"keyword": kw, "score": round(score, 4)}
                for kw, score in sorted_keywords]

    # ── 情感分析 ──

    def get_sentiment_trend(self, room_id: str,
                            interval_minutes: int = 5,
                            duration_hours: int = 1) -> list:
        """情感趋势分析（正/负/中性比例随时间变化）"""
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(hours=duration_hours)

        rows = self.ch.query("""
            SELECT
                toStartOfInterval(created_at,
                    INTERVAL %(interval)s MINUTE) AS time_bucket,
                text
            FROM danmaku_events
            WHERE room_id = %(room_id)s
              AND created_at >= %(start)s
            """, {"room_id": room_id, "start": start_time,
                  "interval": interval_minutes})

        # 按时间段分组统计情感
        buckets = defaultdict(lambda: {"positive": 0, "negative": 0,
                                        "neutral": 0})
        for row in rows:
            sentiment = self._classify_sentiment(row["text"])
            buckets[row["time_bucket"]][sentiment] += 1

        result = []
        for bucket, counts in sorted(buckets.items()):
            total = sum(counts.values())
            result.append({
                "time": bucket.isoformat(),
                "positive_ratio": round(counts["positive"] / total, 3),
                "negative_ratio": round(counts["negative"] / total, 3),
                "neutral_ratio": round(counts["neutral"] / total, 3),
                "total": total,
            })
        return result

    # ── 用户互动指标 ──

    def get_engagement_metrics(self, room_id: str) -> dict:
        """用户互动指标"""
        # 弹幕发送用户数
        active_users = self.ch.query("""
            SELECT count(DISTINCT user_id) AS count
            FROM danmaku_events
            WHERE room_id = %(room_id)s
        """, {"room_id": room_id})[0]["count"]

        # 总弹幕数
        total_danmaku = self.ch.query("""
            SELECT count() AS count
            FROM danmaku_events
            WHERE room_id = %(room_id)s
        """, {"room_id": room_id})[0]["count"]

        # 在线观众数
        online_key = f"room_viewers:online:{room_id}"
        viewer_count = self.redis.scard(online_key)

        # 人均弹幕数
        danmaku_per_viewer = (
            round(total_danmaku / max(active_users, 1), 2)
        )

        # 弹幕/观众比（弹幕密度）
        danmaku_viewer_ratio = (
            round(total_danmaku / max(viewer_count, 1), 2)
        )

        return {
            "total_danmaku": total_danmaku,
            "active_users": active_users,
            "current_viewers": viewer_count,
            "danmaku_per_user": danmaku_per_viewer,
            "danmaku_viewer_ratio": danmaku_viewer_ratio,
        }

    # ── 峰值互动时刻检测 ──

    def detect_peak_moments(self, room_id: str,
                            threshold_factor: float = 3.0) -> list:
        """检测峰值互动时刻（弹幕量突增）"""
        # 获取每分钟弹幕量
        volumes = self.get_minute_volume(room_id, duration_minutes=60)
        if len(volumes) < 5:
            return []

        # 计算基线（移动平均）
        values = [v["volume"] for v in volumes]
        baseline = sum(values) / len(values)
        if baseline == 0:
            return []

        # 检测超过阈值倍数的时刻
        peaks = []
        for i, v in enumerate(volumes):
            if v["volume"] > baseline * threshold_factor:
                peaks.append({
                    "time": v["time"],
                    "volume": v["volume"],
                    "baseline": round(baseline, 1),
                    "ratio": round(v["volume"] / baseline, 1),
                })
        return peaks

    # ── 内部方法 ──

    def _tokenize(self, text: str) -> list:
        """中文分词（简易实现，生产用 jieba）"""
        # 去除标点和空白
        text = re.sub(r'[^一-鿿 a-zA-Z0-9]', '', text)
        # 提取中文词组（2-4字）和英文单词
        tokens = []
        # 中文二元/三元组
        for n in (2, 3, 4):
            for i in range(len(text) - n + 1):
                chunk = text[i:i+n]
                if all('一' <= c <= '鿿' for c in chunk):
                    tokens.append(chunk)
        return tokens

    def _classify_sentiment(self, text: str) -> str:
        """简易情感分类"""
        pos_count = sum(1 for w in self.POSITIVE_WORDS if w in text)
        neg_count = sum(1 for w in self.NEGATIVE_WORDS if w in text)
        if pos_count > neg_count:
            return "positive"
        elif neg_count > pos_count:
            return "negative"
        return "neutral"
```

## 直播流质量监控

### 推流质量检测

```python
import time
from dataclasses import dataclass
from typing import Optional, Dict, List


@dataclass
class StreamHealthMetrics:
    """推流健康指标"""
    room_id: str
    bitrate_kbps: float           # 当前码率
    framerate_fps: float          # 当前帧率
    keyframe_interval_s: float    # 关键帧间隔
    audio_bitrate_kbps: float     # 音频码率
    packet_loss_rate: float       # 丢包率
    timestamp: float


@dataclass
class ViewerQualityMetrics:
    """观众端质量指标"""
    room_id: str
    viewer_id: str
    buffer_ratio: float           # 卡顿率（缓冲时长/总时长）
    start_delay_ms: float         # 首帧延迟
    avg_latency_ms: float         # 平均延迟
    resolution: str               # 分辨率
    timestamp: float


class StreamQualityMonitor:
    """直播流质量监控服务"""

    # 推流质量阈值
    BITRATE_MIN = 800              # 最低码率 kbps
    BITRATE_DROP_THRESHOLD = 0.30  # 码率下降 30% 触发告警
    FPS_MIN = 15                   # 最低帧率
    KEYFRAME_INTERVAL_MAX = 5.0    # 关键帧最大间隔（秒）
    PACKET_LOSS_MAX = 0.01         # 最大丢包率 1%

    # 观众端质量阈值
    BUFFER_RATIO_MAX = 0.05        # 最大卡顿率 5%
    START_DELAY_MAX_MS = 3000      # 最大首帧延迟 3 秒

    # CDN 节点对比配置
    CDN_NODES = ["cdn-bj-01", "cdn-sh-01", "cdn-gz-01",
                 "cdn-cd-01", "cdn-wh-01"]

    def __init__(self, redis, db, clickhouse, alert_service):
        self.redis = redis
        self.db = db
        self.ch = clickhouse
        self.alert = alert_service

    # ── 推流端健康检测 ──

    def check_stream_health(self, metrics: StreamHealthMetrics) -> dict:
        """检测推流端健康状态"""
        issues = []

        # 码率检测
        if metrics.bitrate_kbps < self.BITRATE_MIN:
            issues.append({
                "type": "low_bitrate",
                "value": metrics.bitrate_kbps,
                "threshold": self.BITRATE_MIN,
                "severity": "critical",
            })

        # 码率骤降检测
        prev_bitrate = self._get_previous_bitrate(metrics.room_id)
        if prev_bitrate and prev_bitrate > 0:
            drop_ratio = (prev_bitrate - metrics.bitrate_kbps) / prev_bitrate
            if drop_ratio > self.BITRATE_DROP_THRESHOLD:
                issues.append({
                    "type": "bitrate_drop",
                    "value": round(drop_ratio, 2),
                    "threshold": self.BITRATE_DROP_THRESHOLD,
                    "prev_bitrate": prev_bitrate,
                    "current_bitrate": metrics.bitrate_kbps,
                    "severity": "critical",
                })

        # 帧率检测
        if metrics.framerate_fps < self.FPS_MIN:
            issues.append({
                "type": "low_framerate",
                "value": metrics.framerate_fps,
                "threshold": self.FPS_MIN,
                "severity": "warning",
            })

        # 关键帧间隔检测
        if metrics.keyframe_interval_s > self.KEYFRAME_INTERVAL_MAX:
            issues.append({
                "type": "large_keyframe_interval",
                "value": metrics.keyframe_interval_s,
                "threshold": self.KEYFRAME_INTERVAL_MAX,
                "severity": "warning",
            })

        # 丢包率检测
        if metrics.packet_loss_rate > self.PACKET_LOSS_MAX:
            issues.append({
                "type": "high_packet_loss",
                "value": metrics.packet_loss_rate,
                "threshold": self.PACKET_LOSS_MAX,
                "severity": "critical",
            })

        # 记录指标
        self._record_stream_metrics(metrics)

        # 发送告警
        if any(i["severity"] == "critical" for i in issues):
            self.alert.send_critical(metrics.room_id, issues)
        elif issues:
            self.alert.send_warning(metrics.room_id, issues)

        return {
            "room_id": metrics.room_id,
            "healthy": len(issues) == 0,
            "issues": issues,
        }

    # ── 观众端质量追踪 ──

    def track_viewer_quality(self, metrics: ViewerQualityMetrics) -> dict:
        """追踪观众端质量"""
        issues = []

        if metrics.buffer_ratio > self.BUFFER_RATIO_MAX:
            issues.append({
                "type": "high_buffer_ratio",
                "value": round(metrics.buffer_ratio, 3),
                "threshold": self.BUFFER_RATIO_MAX,
            })

        if metrics.start_delay_ms > self.START_DELAY_MAX_MS:
            issues.append({
                "type": "high_start_delay",
                "value": metrics.start_delay_ms,
                "threshold": self.START_DELAY_MAX_MS,
            })

        # 写入 ClickHouse
        self.ch.insert("viewer_quality_metrics", [{
            "room_id": metrics.room_id,
            "viewer_id": metrics.viewer_id,
            "buffer_ratio": metrics.buffer_ratio,
            "start_delay_ms": metrics.start_delay_ms,
            "avg_latency_ms": metrics.avg_latency_ms,
            "resolution": metrics.resolution,
            "timestamp": metrics.timestamp,
        }])

        return {"issues": issues}

    def get_room_quality_summary(self, room_id: str) -> dict:
        """获取直播间观众端质量汇总"""
        row = self.ch.query("""
            SELECT
                count() AS total_viewers,
                avg(buffer_ratio) AS avg_buffer_ratio,
                avg(start_delay_ms) AS avg_start_delay,
                quantile(0.95)(start_delay_ms) AS p95_start_delay,
                countIf(buffer_ratio > 0.05) AS buffered_viewers
            FROM viewer_quality_metrics
            WHERE room_id = %(room_id)s
              AND timestamp > now() - INTERVAL 5 MINUTE
        """, {"room_id": room_id})

        if not row:
            return {"error": "no_data"}

        r = row[0]
        return {
            "total_viewers": r["total_viewers"],
            "avg_buffer_ratio": round(r["avg_buffer_ratio"], 4),
            "avg_start_delay_ms": round(r["avg_start_delay"], 0),
            "p95_start_delay_ms": round(r["p95_start_delay"], 0),
            "buffered_viewer_ratio": round(
                r["buffered_viewers"] / max(r["total_viewers"], 1), 3),
        }

    # ── 自适应码率推荐 ──

    def recommend_bitrate(self, room_id: str) -> dict:
        """根据观众端网络质量推荐码率"""
        summary = self.get_room_quality_summary(room_id)

        if summary.get("avg_buffer_ratio", 0) > 0.10:
            recommended = "720p_1500kbps"
        elif summary.get("avg_buffer_ratio", 0) > 0.03:
            recommended = "1080p_3000kbps"
        else:
            recommended = "1080p_4500kbps"

        return {
            "room_id": room_id,
            "current_quality": summary,
            "recommended_bitrate": recommended,
        }

    # ── CDN 节点质量对比 ──

    def compare_cdn_quality(self, room_id: str) -> list:
        """对比各 CDN 节点的观众端质量"""
        rows = self.ch.query("""
            SELECT
                cdn_node,
                count() AS viewer_count,
                avg(buffer_ratio) AS avg_buffer,
                avg(start_delay_ms) AS avg_delay,
                quantile(0.95)(avg_latency_ms) AS p95_latency
            FROM viewer_quality_metrics
            WHERE room_id = %(room_id)s
              AND timestamp > now() - INTERVAL 10 MINUTE
            GROUP BY cdn_node
            ORDER BY avg_buffer ASC
        """, {"room_id": room_id})

        return [dict(r) for r in rows]

    # ── 内部方法 ──

    def _get_previous_bitrate(self, room_id: str) -> Optional[float]:
        """获取上一次的码率"""
        key = f"stream_bitrate:{room_id}"
        val = self.redis.get(key)
        return float(val) if val else None

    def _record_stream_metrics(self, metrics: StreamHealthMetrics):
        """记录推流指标"""
        key = f"stream_bitrate:{metrics.room_id}"
        self.redis.set(key, metrics.bitrate_kbps)
        self.redis.expire(key, 30)

    def start_stream_degradation_watch(self, room_id: str):
        """启动流质量降级监控"""
        # 每 10 秒采样一次，连续 3 次降级则告警
        counter_key = f"stream_degradation:{room_id}"
        count = self.redis.incr(counter_key)
        if count == 1:
            self.redis.expire(counter_key, 60)
        if count >= 3:
            self.alert.send_critical(room_id, [{
                "type": "stream_degradation",
                "message": "推流质量持续降级，可能影响观众体验",
            }])
            self.redis.delete(counter_key)
```

## 异常场景补充（续）

### 场景：直播间状态不一致——已结束但仍收到弹幕

```
触发：直播间已结束（ENDED），但观众端 WebSocket 连接未断开 → 继续发送弹幕
根因：
  1. 状态变更通知延迟：Redis 状态已更新但 WebSocket 网关未收到通知
  2. 网关本地缓存未及时刷新：网关缓存了旧状态 LIVE
  3. 客户端未监听直播间结束事件
检测：
  1. 弹幕写入时校验直播间状态 → 状态非 LIVE → 拒绝写入
  2. 状态不一致告警：ENDED 状态的直播间仍有弹幕写入 → 严重告警
  3. 统计指标：ENDED 房间的弹幕 QPS > 0 → 异常
处理：
  1. 弹幕服务端校验：写入前检查 room_state:{room_id}
     - 非 LIVE → 返回错误码 "room_not_live"
  2. WebSocket 网关：直播间结束时主动推送 "room_ended" 事件
     - 观众收到后停止发送弹幕
     - 网关 5 秒后断开该房间所有 WebSocket 连接
  3. 补偿清理：定时任务扫描 ENDED 房间的误写入弹幕并标记为无效
  4. 网关缓存刷新：直播间结束时发布 Redis Pub/Sub 消息
     - 所有网关订阅 → 清除本地缓存 → 断开相关连接
预防：
  - 状态变更时同步推送通知（Redis Pub/Sub）+ 网关本地缓存 TTL 30s
  - 弹幕写入强制校验状态（最后一道防线）
  - 客户端监听房间状态事件 + 自动禁用输入框
```

### 场景：推流质量降级导致观众大量流失

```
触发：主播网络波动 → 推流码率从 3000kbps 降至 800kbps → 画面模糊/卡顿 → 观众离开
影响链：
  推流码率骤降 → CDN 分发低质量流 → 观众端频繁缓冲
  → 观众体验下降 → 5 分钟内流失 40% 观众 → 互动量骤降
检测：
  1. 推流端：码率下降 > 30% → 立即告警
  2. 观众端：卡顿率 > 10% 的观众比例 > 30% → 严重告警
  3. 业务侧：5 分钟内观众数下降 > 20% → 关联告警
处理：
  1. 自动降级：通知 CDN 切换到低延迟模式（减少缓冲区）
  2. 通知主播：弹幕提示"网络不稳定，建议检查网络连接"
  3. 观众端自适应：自动切换到低清晰度（720p → 480p）
  4. 降级兜底：如果 2 分钟内码率未恢复 → 建议主播重启推流
  5. 观众挽回：推流恢复后向近期离开的观众推送"直播已恢复"通知
预防：
  - 推流端自适应码率（OBS/SDK 内置 ABR）
  - CDN 多线路备份（主线路降质时切换备用线路）
  - 观众端自适应码率 + 降级提示
  - 推流质量实时监控 + 自动告警
```

## 直播间状态管理完整实现

```python
class LiveRoomStateManager:
    """直播间状态管理：生命周期 + 并发 + 权限"""

    def create_room(self, anchor_id, config):
        """创建直播间"""
        room_id = str(uuid4())
        self.db.insert("live_rooms", {
            "room_id": room_id, "anchor_id": anchor_id,
            "title": config["title"],
            "danmaku_enabled": True,
            "gift_enabled": True,
            "subscriber_only_mode": False,
            "status": "created",
            "max_viewers": config.get("max_viewers", 100000),
            "created_at": now()
        })
        return room_id

    def go_live(self, room_id):
        """开始直播"""
        self.db.update("live_rooms",
            {"status": "live", "started_at": now()},
            {"room_id": room_id})

        # 初始化计数器
        self.redis.set(f"room_viewers:{room_id}", 0)
        self.redis.set(f"room_peak_viewers:{room_id}", 0)

    def end_live(self, room_id):
        """结束直播"""
        peak = int(self.redis.get(f"room_peak_viewers:{room_id}") or 0)
        duration = (now() - self.db.get_room(room_id)["started_at"]).total_seconds() / 60

        self.db.update("live_rooms",
            {"status": "ended", "ended_at": now(),
             "peak_viewers": peak, "duration_minutes": round(duration)},
            {"room_id": room_id})

        # 生成直播回放
        self.recording_service.generate_replay(room_id)

    def track_viewer_count(self, room_id):
        """追踪并发观众数（HyperLogLog）"""
        # 每个观众进入时添加到 HyperLogLog
        key = f"room_hll:{room_id}:{now().strftime('%Y%m%d%H')}"
        count = self.redis.pfcount(key)
        # 更新峰值
        current_peak = int(self.redis.get(f"room_peak_viewers:{room_id}") or 0)
        if count > current_peak:
            self.redis.set(f"room_peak_viewers:{room_id}", count)
        return count

    def kick_user(self, room_id, user_id, operator_id, reason):
        """踢出用户"""
        # 验证操作者权限
        operator_role = self._get_room_role(room_id, operator_id)
        if operator_role not in ["owner", "moderator"]:
            raise PermissionDeniedError("无权踢人")

        self.db.insert("room_kicks", {
            "room_id": room_id, "user_id": user_id,
            "operator_id": operator_id, "reason": reason,
            "kicked_at": now(),
            "expires_at": now() + timedelta(hours=1)  # 默认踢出 1 小时
        })
        self.redis.setex(f"room_kicked:{room_id}:{user_id}", 3600, "1")
```

## 弹幕统计分析

```python
class DanmakuAnalyticsService:
    """弹幕统计分析：关键词 + 情感 + 互动"""

    def analyze_room(self, room_id, time_range_minutes=30):
        """分析直播间弹幕"""
        # 获取时间范围内的弹幕
        danmaku_list = self.db.query(
            "SELECT * FROM danmaku WHERE room_id = %s "
            "AND timestamp > NOW() - INTERVAL %s MINUTE "
            "ORDER BY timestamp", room_id, time_range_minutes)

        if not danmaku_list:
            return {"count": 0}

        # 1. 关键词提取（TF-IDF）
        keywords = self._extract_keywords(danmaku_list)

        # 2. 情感分析
        sentiments = self._analyze_sentiment(danmaku_list)

        # 3. 互动指标
        total = len(danmaku_list)
        viewer_count = self.redis.pfcount(
            f"room_hll:{room_id}:{now().strftime('%Y%m%d%H')}")
        engagement_ratio = total / max(viewer_count, 1)

        # 4. 互动峰值时刻检测
        per_minute = self._count_per_minute(danmaku_list)
        peak_minute = max(per_minute, key=per_minute.get) if per_minute else None

        return {
            "total_danmaku": total,
            "keywords": keywords[:10],
            "sentiment": sentiments,
            "engagement_ratio": round(engagement_ratio, 2),
            "peak_minute": peak_minute,
            "peak_count": per_minute.get(peak_minute, 0) if peak_minute else 0
        }

    def _extract_keywords(self, danmaku_list):
        """TF-IDF 关键词提取"""
        from collections import Counter
        import jieba

        # 分词
        all_words = []
        for d in danmaku_list:
            words = jieba.cut(d["text"])
            all_words.extend([w for w in words if len(w) > 1])

        # 词频统计
        counter = Counter(all_words)
        return [{"word": w, "count": c} for w, c in counter.most_common(20)]

    def _analyze_sentiment(self, danmaku_list):
        """情感分析（正/负/中）"""
        positive_words = {"好看", "牛逼", "厉害", "爱了", "哈哈", "666", "棒"}
        negative_words = {"无聊", "垃圾", "差", "恶心", "退钱", "假"}

        pos = neg = neutral = 0
        for d in danmaku_list:
            text = d["text"]
            if any(w in text for w in positive_words):
                pos += 1
            elif any(w in text for w in negative_words):
                neg += 1
            else:
                neutral += 1

        total = len(danmaku_list)
        return {
            "positive": round(pos / total * 100, 1),
            "negative": round(neg / total * 100, 1),
            "neutral": round(neutral / total * 100, 1)
        }
```

## 异常场景补充

### 场景：直播间状态不一致

```
触发：直播已结束但仍在接收弹幕 → 状态不一致
检测：
  1. 弹幕写入时检查直播间状态
  2. status != "live" → 拒绝弹幕
处理：
  1. 拒绝新弹幕
  2. 检查状态不一致原因（end_live 事件延迟）
  3. 修复状态 → 确保一致
预防：弹幕写入前状态校验 + 状态变更事件广播
```

### 场景：直播流质量下降导致观众流失

```
触发：直播流码率从 4Mbps 降到 1Mbps → 观众体验差 → 退出
检测：
  1. 流码率 < 2Mbps → 告警
  2. 观众 5 分钟内退出率 > 20% → 质量问题
处理：
  1. 自动切换 CDN 节点
  2. 降低分辨率保持流畅
  3. 通知主播检查网络
预防：流质量监控 + 自动 CDN 切换 + 降级策略
```

## 弹幕内容过滤与安全完整实现

```python
class DanmakuContentFilterService:
    """弹幕内容过滤：实时过滤 + 上下文检测 + 分级展示"""

    FILTER_LEVELS = {
        "strict": {"profanity": True, "politics": True, "ad": True,
                   "personal_attack": True, "minimum_interval_seconds": 5},
        "normal": {"profanity": True, "politics": True, "ad": True,
                   "personal_attack": True, "minimum_interval_seconds": 2},
        "loose": {"profanity": True, "politics": False, "ad": True,
                  "personal_attack": True, "minimum_interval_seconds": 1},
    }

    def filter_danmaku(self, user_id, room_id, content):
        """过滤弹幕内容"""
        room = self.db.get_room(room_id)
        filter_level = room.get("filter_level", "normal")
        config = self.FILTER_LEVELS[filter_level]

        result = {"content": content, "filtered": False, "reason": None}

        # 1. 发送频率限制
        last_send = self.redis.get(f"danmaku_last:{room_id}:{user_id}")
        if last_send:
            elapsed = (now() - datetime.fromisoformat(last_send.decode())).total_seconds()
            if elapsed < config["minimum_interval_seconds"]:
                return {"filtered": True, "reason": "发送过于频繁",
                        "retry_after_seconds": int(config["minimum_interval_seconds"] - elapsed)}

        # 2. 内容长度限制
        if len(content) > 50:
            return {"filtered": True, "reason": "弹幕过长（最多50字）"}

        # 3. 关键词过滤
        if config["profanity"]:
            if self._contains_profanity(content):
                return {"filtered": True, "reason": "包含不当用语"}

        # 4. 政治敏感词
        if config["politics"]:
            if self._contains_political_content(content):
                return {"filtered": True, "reason": "包含敏感内容"}

        # 5. 广告检测
        if config["ad"]:
            if self._contains_advertisement(content):
                return {"filtered": True, "reason": "包含广告信息"}

        # 6. 人身攻击
        if config["personal_attack"]:
            if self._contains_personal_attack(content):
                return {"filtered": True, "reason": "包含人身攻击"}

        # 7. 重复弹幕检测（同一用户 30 秒内不能发相同内容）
        recent_key = f"danmaku_recent:{user_id}:{hashlib.md5(content.encode()).hexdigest()}"
        if self.redis.exists(recent_key):
            return {"filtered": True, "reason": "重复弹幕"}
        self.redis.setex(recent_key, 30, "1")

        # 8. 记录发送时间
        self.redis.setex(f"danmaku_last:{room_id}:{user_id}",
            config["minimum_interval_seconds"], now().isoformat())

        return result

    def _contains_profanity(self, content):
        """检测不当用语"""
        # 词库匹配 + 模型检测
        profanity_words = self.redis.smembers("profanity_words")
        for word in profanity_words:
            word = word.decode() if isinstance(word, bytes) else word
            if word in content:
                return True
        # NLP 模型补充检测（绕过变体）
        score = self.nlp_model.predict(content, "profanity")
        return score > 0.8

    def _contains_advertisement(self, content):
        """检测广告"""
        # URL 检测
        if re.search(r'https?://|www\.|\.com|\.cn', content, re.IGNORECASE):
            return True
        # 联系方式检测
        if re.search(r'(微信|QQ|电话|加我)[：:]\s*\w+', content):
            return True
        # NLP 模型
        score = self.nlp_model.predict(content, "advertisement")
        return score > 0.7

    def _contains_personal_attack(self, content):
        """检测人身攻击"""
        score = self.nlp_model.predict(content, "personal_attack")
        return score > 0.7

    def _contains_political_content(self, content):
        """检测政治敏感内容"""
        sensitive_words = self.redis.smembers("political_sensitive_words")
        for word in sensitive_words:
            word = word.decode() if isinstance(word, bytes) else word
            if word in content:
                return True
        return False
```

## 弹幕防刷与流量控制

```python
class DanmakuRateLimiter:
    """弹幕防刷：用户级 + 房间级 + 全局级"""

    LIMITS = {
        "user_per_minute": 20,
        "room_per_second": 100,
        "global_per_second": 50000,
    }

    def check_rate_limit(self, user_id, room_id):
        """检查发送频率限制"""
        # 1. 用户级限制
        user_key = f"danmaku_user_rate:{user_id}"
        user_count = self.redis.incr(user_key)
        if user_count == 1:
            self.redis.expire(user_key, 60)
        if user_count > self.LIMITS["user_per_minute"]:
            return {"allowed": False, "reason": "用户发送频率超限",
                    "limit": self.LIMITS["user_per_minute"]}

        # 2. 房间级限制
        room_key = f"danmaku_room_rate:{room_id}"
        room_count = self.redis.incr(room_key)
        if room_count == 1:
            self.redis.expire(room_key, 1)
        if room_count > self.LIMITS["room_per_second"]:
            return {"allowed": False, "reason": "房间弹幕频率超限"}

        # 3. 全局限制
        global_count = self.redis.incr("danmaku_global_rate")
        if global_count == 1:
            self.redis.expire("danmaku_global_rate", 1)
        if global_count > self.LIMITS["global_per_second"]:
            return {"allowed": False, "reason": "全局弹幕频率超限"}

        return {"allowed": True}
```

## 异常场景补充

### 场景：弹幕过滤词库更新不及时

```
触发：新型变体脏话绕过词库 → 过滤失败 → 弹幕环境恶化
检测：
  1. 用户举报弹幕增多 → 过滤失效
  2. 词库覆盖的新词比例下降 → 需要更新
处理：
  1. NLP 模型补充词库不足
  2. 用户举报自动加入候选词库（人工审核后生效）
  3. 定期更新词库（每周）
预防：NLP 补充 + 举报自动收集 + 定期更新
```

### 场景：高并发弹幕导致限流误伤

```
触发：百万人直播 → 房间级限流 100/秒 → 正常用户弹幕被丢弃 → 体验差
检测：
  1. 弹幕发送成功率 < 50% → 限流过严
  2. 用户投诉弹幕发不出去 → 限流误伤
处理：
  1. 房间限流动态调整（按在线人数比例）
  2. 弹幕排队（延迟展示而非丢弃）
  3. VIP 用户更高限额
预防：动态限流 + 排队机制 + 分级限额
```

## 弹幕礼物与虚拟经济完整实现

```python
import time
import uuid
import threading
from enum import Enum
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Tuple
from collections import defaultdict
from datetime import datetime, timedelta


class GiftType(Enum):
    COMMON = "common"
    RARE = "rare"
    LEGENDARY = "legendary"
    EXCLUSIVE = "exclusive"


class GiftAnimation(Enum):
    BASIC_SPARKLE = "basic_sparkle"
    FIREWORK = "firework"
    FULL_SCREEN = "full_screen"
    CUSTOM_3D = "custom_3d"


@dataclass
class GiftDefinition:
    """虚拟礼物类型定义"""
    gift_id: str
    name: str
    price: int  # 虚拟币价格
    gift_type: GiftType
    animation: GiftAnimation
    animation_duration_ms: int
    creator_share_ratio: float  # 创作者分成比例
    platform_share_ratio: float  # 平台分成比例
    combo_threshold: int = 3  # 连击阈值
    combo_window_seconds: float = 3.0  # 连击时间窗口
    is_active: bool = True
    created_at: datetime = field(default_factory=datetime.now)


@dataclass
class GiftRecord:
    """礼物发送记录"""
    record_id: str
    gift_id: str
    sender_id: str
    receiver_id: str  # 主播/创作者ID
    room_id: str
    amount: int  # 单次数量
    combo_count: int  # 连击数
    total_price: int
    animation_triggered: bool
    timestamp: datetime = field(default_factory=datetime.now)


@dataclass
class ComboState:
    """连击状态"""
    sender_id: str
    gift_id: str
    room_id: str
    count: int
    first_send_time: datetime
    last_send_time: datetime


@dataclass
class CreatorRevenue:
    """创作者收益记录"""
    creator_id: str
    total_gift_revenue: int = 0
    pending_settlement: int = 0
    settled: int = 0
    last_settlement_time: Optional[datetime] = None


class DanmakuGiftService:
    """弹幕礼物与虚拟经济完整服务

    管理虚拟礼物的创建、发送、连击、动画触发及收益结算全流程。
    """

    def __init__(self, platform_commission_rate: float = 0.3):
        # 礼物类型注册表
        self._gift_registry: Dict[str, GiftDefinition] = {}
        # 用户余额表 user_id -> balance
        self._user_balances: Dict[str, int] = defaultdict(int)
        # 礼物发送记录
        self._gift_records: List[GiftRecord] = []
        # 连击状态 key: (sender_id, gift_id, room_id)
        self._combo_states: Dict[Tuple[str, str, str], ComboState] = {}
        # 创作者收益
        self._creator_revenues: Dict[str, CreatorRevenue] = {}
        # 动画触发队列
        self._animation_queue: List[Dict] = []
        # 结算锁
        self._settlement_lock = threading.Lock()
        # 连击窗口清理定时器
        self._combo_cleanup_interval = 5.0
        # 平台分成比例
        self._platform_commission_rate = platform_commission_rate
        # 连击回调
        self._combo_callbacks: List[callable] = []

    def create_gift_type(
        self,
        name: str,
        price: int,
        gift_type: GiftType,
        animation: GiftAnimation,
        animation_duration_ms: int = 3000,
        creator_share_ratio: float = 0.7,
        combo_threshold: int = 3,
        combo_window_seconds: float = 3.0,
    ) -> GiftDefinition:
        """创建礼物类型

        Args:
            name: 礼物名称
            price: 礼物价格（虚拟币）
            gift_type: 礼物等级
            animation: 动画类型
            animation_duration_ms: 动画时长
            creator_share_ratio: 创作者分成比例，范围(0, 1)
            combo_threshold: 触发连击效果的最小次数
            combo_window_seconds: 连击时间窗口（秒）

        Returns:
            GiftDefinition: 创建的礼物类型定义

        Raises:
            ValueError: 参数不合法时抛出
        """
        if price <= 0:
            raise ValueError(f"礼物价格必须为正数，当前: {price}")
        if not name or len(name.strip()) == 0:
            raise ValueError("礼物名称不能为空")
        if not (0 < creator_share_ratio < 1):
            raise ValueError(f"创作者分成比例必须在(0,1)之间，当前: {creator_share_ratio}")
        if combo_threshold < 1:
            raise ValueError(f"连击阈值必须>=1，当前: {combo_threshold}")
        if combo_window_seconds <= 0:
            raise ValueError(f"连击时间窗口必须>0，当前: {combo_window_seconds}")

        gift_id = f"gift_{uuid.uuid4().hex[:12]}"
        platform_share = round(1.0 - creator_share_ratio, 4)

        definition = GiftDefinition(
            gift_id=gift_id,
            name=name.strip(),
            price=price,
            gift_type=gift_type,
            animation=animation,
            animation_duration_ms=animation_duration_ms,
            creator_share_ratio=creator_share_ratio,
            platform_share_ratio=platform_share,
            combo_threshold=combo_threshold,
            combo_window_seconds=combo_window_seconds,
        )
        self._gift_registry[gift_id] = definition
        return definition

    def send_gift(
        self,
        sender_id: str,
        receiver_id: str,
        room_id: str,
        gift_id: str,
        amount: int = 1,
    ) -> GiftRecord:
        """发送礼物

        完整流程：校验 -> 扣款 -> 记录 -> 连击检测 -> 触发动画

        Args:
            sender_id: 发送者ID
            receiver_id: 接收者（主播）ID
            room_id: 直播间ID
            gift_id: 礼物类型ID
            amount: 数量

        Returns:
            GiftRecord: 礼物发送记录

        Raises:
            ValueError: 参数不合法
            InsufficientBalanceError: 余额不足
            GiftNotFoundError: 礼物类型不存在
            GiftInactiveError: 礼物类型已下架
        """
        if amount <= 0:
            raise ValueError(f"礼物数量必须为正数，当前: {amount}")

        # 校验礼物类型
        gift_def = self._gift_registry.get(gift_id)
        if gift_def is None:
            raise ValueError(f"礼物类型不存在: {gift_id}")
        if not gift_def.is_active:
            raise ValueError(f"礼物类型已下架: {gift_def.name}")

        total_price = gift_def.price * amount

        # 检查余额
        current_balance = self._user_balances.get(sender_id, 0)
        if current_balance < total_price:
            raise ValueError(
                f"余额不足: 需要{total_price}，当前余额{current_balance}"
            )

        # 扣款（原子操作）
        self._user_balances[sender_id] -= total_price

        # 连击处理
        combo_count, is_combo = self._update_combo_state(
            sender_id, gift_id, room_id, amount
        )

        # 触发动画
        animation_triggered = self._trigger_gift_animation(
            gift_def, combo_count if is_combo else amount, room_id
        )

        # 创建发送记录
        record = GiftRecord(
            record_id=f"gr_{uuid.uuid4().hex[:12]}",
            gift_id=gift_id,
            sender_id=sender_id,
            receiver_id=receiver_id,
            room_id=room_id,
            amount=amount,
            combo_count=combo_count if is_combo else 0,
            total_price=total_price,
            animation_triggered=animation_triggered,
        )
        self._gift_records.append(record)

        # 更新创作者待结算收益
        self._add_creator_pending_revenue(
            receiver_id, int(total_price * gift_def.creator_share_ratio)
        )

        # 连击回调通知
        if is_combo and self._combo_callbacks:
            for cb in self._combo_callbacks:
                try:
                    cb(sender_id, receiver_id, room_id, gift_id, combo_count)
                except Exception:
                    pass

        return record

    def process_combo(
        self, sender_id: str, gift_id: str, room_id: str, new_amount: int = 1
    ) -> int:
        """处理连击逻辑

        在连击时间窗口内，同一用户在同一房间发送同一种礼物，
        累计次数达到阈值后触发连击效果。

        Args:
            sender_id: 发送者ID
            gift_id: 礼物类型ID
            room_id: 直播间ID
            new_amount: 本次发送数量

        Returns:
            int: 当前连击总数
        """
        key = (sender_id, gift_id, room_id)
        now = datetime.now()
        gift_def = self._gift_registry.get(gift_id)
        if gift_def is None:
            return 0

        window = gift_def.combo_window_seconds
        threshold = gift_def.combo_threshold

        state = self._combo_states.get(key)
        if state is None:
            state = ComboState(
                sender_id=sender_id,
                gift_id=gift_id,
                room_id=room_id,
                count=new_amount,
                first_send_time=now,
                last_send_time=now,
            )
            self._combo_states[key] = state
            return state.count

        # 检查是否在时间窗口内
        elapsed = (now - state.last_send_time).total_seconds()
        if elapsed <= window:
            state.count += new_amount
            state.last_send_time = now
            if state.count >= threshold:
                self._trigger_combo_effect(sender_id, gift_id, room_id, state.count)
        else:
            # 超出窗口，重置连击
            state.count = new_amount
            state.first_send_time = now
            state.last_send_time = now

        return state.count

    def settle_creator_revenue(
        self, creator_id: str, min_amount: int = 100
    ) -> Dict:
        """结算创作者收益

        将待结算收益转为已结算，平台同时提取分成。
        仅当待结算金额达到最低结算门槛时才执行。

        Args:
            creator_id: 创作者ID
            min_amount: 最低结算金额

        Returns:
            Dict: 结算结果，包含结算金额、平台分成、结算时间等

        Raises:
            ValueError: 待结算金额不足最低门槛
        """
        with self._settlement_lock:
            revenue = self._creator_revenues.get(creator_id)
            if revenue is None or revenue.pending_settlement < min_amount:
                pending = revenue.pending_settlement if revenue else 0
                raise ValueError(
                    f"待结算金额{pending}未达到最低结算门槛{min_amount}"
                )

            settle_amount = revenue.pending_settlement
            platform_share = int(settle_amount * self._platform_commission_rate)
            creator_final = settle_amount - platform_share

            revenue.settled += creator_final
            revenue.pending_settlement = 0
            revenue.last_settlement_time = datetime.now()

            return {
                "creator_id": creator_id,
                "settle_amount": settle_amount,
                "creator_final": creator_final,
                "platform_share": platform_share,
                "settled_at": revenue.last_settlement_time.isoformat(),
            }

    # ---- 内部辅助方法 ----

    def _update_combo_state(
        self, sender_id: str, gift_id: str, room_id: str, amount: int
    ) -> Tuple[int, bool]:
        """更新连击状态并返回 (连击数, 是否达到连击阈值)"""
        combo_count = self.process_combo(sender_id, gift_id, room_id, amount)
        gift_def = self._gift_registry[gift_id]
        is_combo = combo_count >= gift_def.combo_threshold
        return combo_count, is_combo

    def _trigger_gift_animation(
        self, gift_def: GiftDefinition, display_count: int, room_id: str
    ) -> bool:
        """触发礼物动画，推送到直播间动画队列"""
        try:
            animation_event = {
                "event_id": f"anim_{uuid.uuid4().hex[:8]}",
                "gift_id": gift_def.gift_id,
                "gift_name": gift_def.name,
                "animation_type": gift_def.animation.value,
                "duration_ms": gift_def.animation_duration_ms,
                "display_count": display_count,
                "room_id": room_id,
                "timestamp": datetime.now().isoformat(),
            }
            self._animation_queue.append(animation_event)
            return True
        except Exception:
            return False

    def _trigger_combo_effect(
        self, sender_id: str, gift_id: str, room_id: str, count: int
    ):
        """触发连击特效"""
        gift_def = self._gift_registry.get(gift_id)
        if gift_def is None:
            return
        combo_event = {
            "event_id": f"combo_{uuid.uuid4().hex[:8]}",
            "type": "combo_effect",
            "gift_id": gift_id,
            "gift_name": gift_def.name,
            "combo_count": count,
            "room_id": room_id,
            "sender_id": sender_id,
            "effect_level": min(count // gift_def.combo_threshold, 5),
            "timestamp": datetime.now().isoformat(),
        }
        self._animation_queue.append(combo_event)

    def _add_creator_pending_revenue(self, creator_id: str, amount: int):
        """增加创作者待结算收益"""
        if creator_id not in self._creator_revenues:
            self._creator_revenues[creator_id] = CreatorRevenue(
                creator_id=creator_id
            )
        self._creator_revenues[creator_id].pending_settlement += amount
        self._creator_revenues[creator_id].total_gift_revenue += amount

    def recharge_balance(self, user_id: str, amount: int):
        """充值虚拟币余额"""
        if amount <= 0:
            raise ValueError(f"充值金额必须为正数，当前: {amount}")
        self._user_balances[user_id] += amount

    def get_balance(self, user_id: str) -> int:
        """查询用户余额"""
        return self._user_balances.get(user_id, 0)

    def get_creator_revenue(self, creator_id: str) -> Optional[Dict]:
        """查询创作者收益"""
        revenue = self._creator_revenues.get(creator_id)
        if revenue is None:
            return None
        return {
            "creator_id": revenue.creator_id,
            "total_gift_revenue": revenue.total_gift_revenue,
            "pending_settlement": revenue.pending_settlement,
            "settled": revenue.settled,
            "last_settlement_time": (
                revenue.last_settlement_time.isoformat()
                if revenue.last_settlement_time
                else None
            ),
        }

    def get_animation_queue(self, room_id: str) -> List[Dict]:
        """获取指定房间的动画队列"""
        return [e for e in self._animation_queue if e.get("room_id") == room_id]

    def register_combo_callback(self, callback: callable):
        """注册连击回调函数"""
        self._combo_callbacks.append(callback)

    def deactivate_gift_type(self, gift_id: str):
        """下架礼物类型"""
        gift_def = self._gift_registry.get(gift_id)
        if gift_def:
            gift_def.is_active = False

    def cleanup_expired_combos(self):
        """清理过期的连击状态"""
        now = datetime.now()
        expired_keys = []
        for key, state in self._combo_states.items():
            gift_def = self._gift_registry.get(state.gift_id)
            if gift_def is None:
                expired_keys.append(key)
                continue
            elapsed = (now - state.last_send_time).total_seconds()
            if elapsed > gift_def.combo_window_seconds * 2:
                expired_keys.append(key)
        for key in expired_keys:
            del self._combo_states[key]
```

## 异常场景补充

### 场景：礼物发送扣款但动画未触发
```
trigger: 用户在直播间发送高价值全屏动画礼物后，扣款成功但直播间未播放任何动画效果，用户看到余额减少但无视觉反馈
detection:
  1. 在 send_gift 方法中记录 animation_triggered 标志位，若为 False 则标记为异常记录
  2. 定时任务扫描 _gift_records 中 animation_triggered=False 且 total_price > 阈值的记录
  3. 动画服务消费 _animation_queue 时若推送失败，写入失败日志并触发告警
  4. 前端设置动画超时检测：发送礼物后 N 秒未收到动画事件则上报
handling:
  1. 立即将失败的动画事件重新入队，最多重试3次，间隔递增（1s/3s/5s）
  2. 若重试仍失败，降级为弹幕文字通知："用户XX送出了YY"，确保有基础反馈
  3. 对扣款成功但动画未触发的用户发放等额补偿券，避免用户投诉升级
  4. 记录异常到监控面板，按礼物价值排序展示，优先处理高价值礼物的动画失败
prevention:
  1. send_gift 中扣款与动画触发使用事务性设计：先写入动画队列再扣款，或扣款后异步确认动画已消费
  2. 动画推送引入消息确认机制（ACK），未确认时自动重试
  3. 高价值礼物采用同步触发模式，确认动画服务就绪后才完成扣款
  4. 定期对账：礼物发送记录数 vs 动画触发记录数，差异超过阈值即告警
```

### 场景：大额礼物欺诈
```
trigger: 攻击者利用盗取的账号或支付漏洞，在短时间内向合谋主播大量发送高价值礼物，随后通过私下分赃变现
detection:
  1. 风控系统监控单用户在短时间内的礼物发送总额，超过日均消费的5倍即触发风控
  2. 检测"礼物-收益"闭环：发送者与接收者之间存在频繁资金往来或同一设备登录
  3. 异常时间模式：新注册账号短时间内大额消费、凌晨时段突增大额礼物
  4. 设备指纹关联：同一设备多账号轮换发送大额礼物
handling:
  1. 立即冻结涉事账号的礼物发送权限，暂停主播的收益提现
  2. 对已发送的大额礼物标记为"待审查"，暂不结算给主播
  3. 通知安全团队人工审核，调取账号登录日志、设备信息、支付记录
  4. 确认欺诈后，撤销礼物记录，退还发送者余额，封禁涉事账号
  5. 已提现的收益启动追偿流程，必要时报警处理
prevention:
  1. 新用户/新支付方式设置消费冷静期，首日单笔消费上限及日消费上限
  2. 大额礼物发送需二次验证（短信/人脸），单笔超过5000虚拟币强制验证
  3. 建立"礼物-收益"关联图谱，定期识别可疑的发送-接收集中模式
  4. 引入支付风控评分模型，结合账号年龄、消费历史、设备风险等综合评分
  5. 主播收益提现设置T+7延迟结算，为风控审查预留时间窗口
```

## 弹幕内容审核与过滤完整实现

```python
class DanmakuContentModerationService:
    """弹幕审核：实时过滤 → AI 审核 → 人工复审 → 用户举报"""

    FILTER_LEVELS = {
        "strict": {"auto_block": True, "ai_threshold": 0.7, "human_review": True},
        "normal": {"auto_block": True, "ai_threshold": 0.85, "human_review": False},
        "relaxed": {"auto_block": False, "ai_threshold": 0.95, "human_review": False},
    }

    CONTENT_CATEGORIES = {
        "spam": "刷屏/垃圾信息",
        "profanity": "脏话/侮辱",
        "politics": "政治敏感",
        "porn": "色情内容",
        "violence": "暴力/恐怖",
        "ad": "广告/推广",
        "privacy": "隐私泄露",
        "illegal": "违法信息",
    }

    def moderate_danmaku(self, danmaku):
        """审核弹幕"""
        content = danmaku["content"]
        user_id = danmaku["user_id"]
        room_id = danmaku["room_id"]

        # 1. 用户黑名单检查（瞬间过滤）
        if self.redis.sismember("user_blacklist", user_id):
            return {"status": "blocked", "reason": "user_blacklisted"}

        # 2. 关键词过滤（正则匹配）
        keyword_result = self._keyword_filter(content)
        if keyword_result["blocked"]:
            self._record_moderation(danmaku, "blocked", "keyword",
                                   keyword_result["matched_keyword"])
            return {"status": "blocked", "reason": "keyword_filter",
                    "category": keyword_result["category"]}

        # 3. 频率限制（防刷屏）
        freq_result = self._check_frequency(user_id, room_id)
        if freq_result["blocked"]:
            return {"status": "blocked", "reason": "rate_limit",
                    "detail": freq_result["detail"]}

        # 4. 重复内容检测
        if self._is_duplicate(content, room_id):
            return {"status": "blocked", "reason": "duplicate_spam"}

        # 5. AI 内容审核
        room = self.db.get_live_room(room_id)
        filter_level = room.get("filter_level", "normal")
        level_config = self.FILTER_LEVELS[filter_level]

        ai_result = self._ai_content_filter(content, level_config["ai_threshold"])

        if ai_result["flagged"]:
            if level_config["auto_block"]:
                self._record_moderation(danmaku, "blocked", "ai",
                    ai_result["category"])
                return {"status": "blocked", "reason": "ai_filter",
                        "category": ai_result["category"],
                        "confidence": ai_result["confidence"]}
            else:
                # 不自动阻断但标记
                self._record_moderation(danmaku, "flagged", "ai",
                    ai_result["category"])
                return {"status": "allowed_with_flag",
                        "category": ai_result["category"]}

        # 6. 人工复审（严格模式）
        if level_config["human_review"] and ai_result["confidence"] > 0.5:
            self._queue_for_human_review(danmaku, ai_result)
            return {"status": "pending_review", "content": content}

        # 7. 通过审核
        return {"status": "allowed"}

    def _keyword_filter(self, content):
        """关键词过滤"""
        # 从缓存获取关键词列表
        keywords = self.redis.get("moderation_keywords")
        if not keywords:
            keywords = self._load_keywords_from_db()
            self.redis.setex("moderation_keywords", 300, json.dumps(keywords))
        else:
            keywords = json.loads(keywords)

        for keyword in keywords:
            if keyword["pattern"] in content:
                return {"blocked": True,
                        "matched_keyword": keyword["pattern"],
                        "category": keyword["category"]}

        return {"blocked": False}

    def _check_frequency(self, user_id, room_id):
        """检查发送频率"""
        # 10 秒内最多 5 条
        recent_key = f"danmaku_freq:{room_id}:{user_id}"
        count = self.redis.incr(recent_key)
        if count == 1:
            self.redis.expire(recent_key, 10)

        if count > 5:
            return {"blocked": True,
                    "detail": f"10 秒内发送 {count} 条，超过限制"}

        # 每分钟最多 20 条
        minute_key = f"danmaku_freq_min:{room_id}:{user_id}"
        minute_count = self.redis.incr(minute_key)
        if minute_count == 1:
            self.redis.expire(minute_key, 60)

        if minute_count > 20:
            return {"blocked": True,
                    "detail": f"1 分钟内发送 {minute_count} 条"}

        return {"blocked": False}

    def _is_duplicate(self, content, room_id):
        """检测重复弹幕"""
        content_hash = hashlib.md5(content.encode()).hexdigest()
        key = f"danmaku_dedup:{room_id}"
        seen = self.redis.sismember(key, content_hash)

        if seen:
            return True

        self.redis.sadd(key, content_hash)
        self.redis.expire(key, 60)  # 60 秒内去重
        return False

    def _ai_content_filter(self, content, threshold):
        """AI 内容审核"""
        result = self.ai_client.moderate(content)

        return {
            "flagged": result.get("flagged", False),
            "category": result.get("category", "unknown"),
            "confidence": result.get("confidence", 0),
            "categories": result.get("categories", {})
        }

    def handle_user_report(self, report):
        """处理用户举报"""
        danmaku_id = report["danmaku_id"]
        reporter_id = report["reporter_id"]
        reason = report["reason"]

        # 1. 记录举报
        report_id = str(uuid4())
        self.db.insert("danmaku_reports", {
            "report_id": report_id,
            "danmaku_id": danmaku_id,
            "reporter_id": reporter_id,
            "reason": reason,
            "status": "pending",
            "created_at": now()
        })

        # 2. 统计该弹幕被举报次数
        report_count = self.db.count("danmaku_reports", danmaku_id=danmaku_id)

        # 3. 3 次举报 → 自动隐藏并进入人工审核
        if report_count >= 3:
            self.db.update("danmaku",
                {"status": "hidden", "hidden_reason": "multiple_reports"},
                {"id": danmaku_id})

            self._queue_for_human_review_by_id(danmaku_id)

        return {"report_id": report_id, "status": "recorded"}

    def _record_moderation(self, danmaku, status, method, detail=None):
        """记录审核结果"""
        self.db.insert("danmaku_moderation_log", {
            "log_id": str(uuid4()),
            "danmaku_id": danmaku.get("id"),
            "user_id": danmaku["user_id"],
            "room_id": danmaku["room_id"],
            "content": danmaku["content"],
            "status": status,
            "method": method,
            "detail": detail,
            "created_at": now()
        })

    def _queue_for_human_review(self, danmaku, ai_result):
        """加入人工审核队列"""
        self.db.insert("danmaku_review_queue", {
            "queue_id": str(uuid4()),
            "danmaku_id": danmaku.get("id"),
            "content": danmaku["content"],
            "user_id": danmaku["user_id"],
            "room_id": danmaku["room_id"],
            "ai_result": json.dumps(ai_result),
            "status": "pending",
            "created_at": now()
        })

    def _load_keywords_from_db(self):
        """从数据库加载关键词"""
        return self.db.query("SELECT pattern, category FROM moderation_keywords "
                            "WHERE status = 'active'")
```

## 异常场景补充

### 场景：弹幕审核系统误杀导致正常弹幕被屏蔽

```
触发：AI 审核模型过严 → 正常讨论被标记为敏感 → 用户不满 → 弹幕活跃度下降
检测：
  1. 弹幕审核屏蔽率 > 10% → 可能过严
  2. 用户申诉率 > 20% → 误杀严重
处理：
  1. 提供申诉机制
  2. 申诉通过后调整 AI 阈值
  3. 房主可自定义审核等级
预防：申诉机制 + 阈值调整 + 自定义等级
```

### 场景：弹幕审核延迟导致不良内容短暂可见

```
触发：AI 审核耗时 200ms → 弹幕已显示给观众 → 审核不通过才删除 → 短暂可见
检测：
  1. 审核耗时 > 100ms → 延迟
  2. 不良弹幕在被删除前被截图 → 泄露
处理：
  1. 高风险房间先审后发
  2. AI 审核结果缓存（相似内容秒级返回）
  3. 关键词过滤前置（0ms 延迟）
预防：先审后发 + 结果缓存 + 前置过滤
```

## 弹幕系统实时排行榜与互动打赏完整实现

```python
class LiveInteractionService:
    """直播互动：弹幕排行榜 → 礼物打赏 → 互动游戏 → 数据统计"""

    GIFT_TYPES = {
        "flower": {"price": 1, "animation": "petal_fall", "duration_ms": 2000},
        "rocket": {"price": 50, "animation": "rocket_launch", "duration_ms": 5000},
        "crown": {"price": 200, "animation": "crown_shine", "duration_ms": 8000},
        "castle": {"price": 1000, "animation": "castle_build", "duration_ms": 15000},
    }

    def send_gift(self, sender_id, room_id, gift_type, count=1):
        """发送礼物"""
        # 1. 验证礼物类型
        gift_config = self.GIFT_TYPES.get(gift_type)
        if not gift_config:
            return {"status": "invalid_gift"}

        # 2. 计算总价
        total_price = Decimal(str(gift_config["price"])) * Decimal(str(count))

        # 3. 检查余额
        user_balance = self._get_user_balance(sender_id)
        if user_balance < total_price:
            return {"status": "insufficient_balance",
                    "required": str(total_price),
                    "current": str(user_balance)}

        # 4. 扣除余额
        self._deduct_balance(sender_id, total_price)

        # 5. 分成（主播 50%，平台 30%，公会 20%）
        room = self.db.get_live_room(room_id)
        streamer_id = room["streamer_id"]
        guild_id = room.get("guild_id")

        streamer_share = total_price * Decimal("0.50")
        platform_share = total_price * Decimal("0.30")
        guild_share = total_price * Decimal("0.20") if guild_id else Decimal("0")

        # 6. 记录礼物
        gift_id = str(uuid4())
        self.db.insert("gift_records", {
            "gift_id": gift_id,
            "sender_id": sender_id,
            "streamer_id": streamer_id,
            "room_id": room_id,
            "gift_type": gift_type,
            "count": count,
            "total_price": str(total_price),
            "streamer_share": str(streamer_share),
            "platform_share": str(platform_share),
            "guild_share": str(guild_share),
            "created_at": now()
        })

        # 7. 更新排行榜
        self._update_contribution_rank(room_id, sender_id, total_price)

        # 8. 广播礼物特效
        self.pubsub.publish(f"room:{room_id}:gift", json.dumps({
            "gift_id": gift_id,
            "sender_id": sender_id,
            "gift_type": gift_type,
            "count": count,
            "animation": gift_config["animation"],
            "duration_ms": gift_config["duration_ms"],
            "total_price": str(total_price)
        }))

        # 9. 通知主播
        self.notification.send(streamer_id,
            f"收到 {sender_id} 的 {count} 个 {gift_type} 礼物（价值 {total_price} 元）")

        return {"gift_id": gift_id, "total_price": str(total_price),
                "animation": gift_config["animation"]}

    def _update_contribution_rank(self, room_id, user_id, amount):
        """更新贡献榜"""
        rank_key = f"contribution_rank:{room_id}"

        # 增加用户贡献值
        self.redis.zincrby(rank_key, float(amount), user_id)

        # 设置过期（直播结束后清除）
        self.redis.expireat(rank_key,
            int(now().timestamp()) + 86400)

    def get_contribution_rank(self, room_id, top_n=50):
        """获取贡献排行榜"""
        rank_key = f"contribution_rank:{room_id}"

        # 获取 Top N
        rank_data = self.redis.zrevrange(rank_key, 0, top_n - 1, withscores=True)

        result = []
        for rank_index, (user_id_bytes, score) in enumerate(rank_data):
            user_id = user_id_bytes.decode() if isinstance(user_id_bytes, bytes) else user_id_bytes
            user = self.db.get_user(user_id)

            result.append({
                "rank": rank_index + 1,
                "user_id": user_id,
                "nickname": user.get("nickname", "") if user else "",
                "avatar": user.get("avatar", "") if user else "",
                "contribution": round(score, 2)
            })

        return {"room_id": room_id, "rankings": result}

    def start_interactive_poll(self, room_id, question, options, duration_seconds=60):
        """开始互动投票"""
        poll_id = str(uuid4())

        # 1. 创建投票
        self.db.insert("interactive_polls", {
            "poll_id": poll_id,
            "room_id": room_id,
            "question": question,
            "options": json.dumps(options),
            "status": "active",
            "end_at": now() + timedelta(seconds=duration_seconds),
            "created_at": now()
        })

        # 2. 初始化 Redis 计数
        for i in range(len(options)):
            self.redis.set(f"poll:{poll_id}:option:{i}", 0)

        # 3. 广播投票
        self.pubsub.publish(f"room:{room_id}:poll", json.dumps({
            "poll_id": poll_id,
            "question": question,
            "options": options,
            "duration_seconds": duration_seconds
        }))

        # 4. 设置到期自动结束
        self.task_queue.schedule(
            self.end_interactive_poll, poll_id,
            delay_seconds=duration_seconds)

        return {"poll_id": poll_id, "question": question}

    def vote_poll(self, poll_id, user_id, option_index):
        """参与投票"""
        poll = self.db.get_poll(poll_id)

        if not poll or poll["status"] != "active":
            return {"status": "poll_closed"}

        # 1. 检查是否已投票
        if self.redis.sismember(f"poll:{poll_id}:voters", user_id):
            return {"status": "already_voted"}

        # 2. 记录投票
        self.redis.sadd(f"poll:{poll_id}:voters", user_id)
        self.redis.incr(f"poll:{poll_id}:option:{option_index}")

        return {"status": "voted", "option_index": option_index}

    def end_interactive_poll(self, poll_id):
        """结束投票"""
        poll = self.db.get_poll(poll_id)
        options = json.loads(poll["options"])

        # 1. 统计结果
        results = []
        total_votes = 0

        for i, option in enumerate(options):
            vote_count = int(self.redis.get(f"poll:{poll_id}:option:{i}") or 0)
            total_votes += vote_count
            results.append({
                "option": option,
                "index": i,
                "votes": vote_count
            })

        # 计算百分比
        for r in results:
            r["percentage"] = round(r["votes"] / max(total_votes, 1) * 100, 1)

        # 2. 更新状态
        self.db.update("interactive_polls",
            {"status": "ended", "results": json.dumps(results),
             "total_votes": total_votes, "ended_at": now()},
            {"poll_id": poll_id})

        # 3. 广播结果
        self.pubsub.publish(f"room:{poll['room_id']}:poll_result", json.dumps({
            "poll_id": poll_id,
            "question": poll["question"],
            "results": results,
            "total_votes": total_votes
        }))

        return {"poll_id": poll_id, "results": results, "total_votes": total_votes}

    def get_live_stream_stats(self, room_id):
        """获取直播数据统计"""
        # 1. 在线人数
        online_count = self.redis.scard(f"room_audience:{room_id}")

        # 2. 弹幕数
        danmaku_count = self.db.count("danmaku", room_id=room_id,
            created_at__gte=now() - timedelta(hours=1))

        # 3. 礼物收入
        gift_revenue = self.db.query_one(
            "SELECT COALESCE(SUM(total_price), 0) as total "
            "FROM gift_records "
            "WHERE room_id = %s AND created_at > NOW() - INTERVAL 1 HOUR",
            room_id)["total"]

        # 4. 新增关注
        new_followers = self.db.count("streamer_followers",
            streamer_id=self.db.get_live_room(room_id)["streamer_id"],
            created_at__gte=now() - timedelta(hours=1))

        # 5. 峰值人数
        peak_count = int(self.redis.get(f"room_peak:{room_id}") or 0)
        if online_count > peak_count:
            self.redis.set(f"room_peak:{room_id}", online_count)
            peak_count = online_count

        return {
            "room_id": room_id,
            "online_count": online_count,
            "peak_count": peak_count,
            "danmaku_last_hour": danmaku_count,
            "gift_revenue_last_hour": str(gift_revenue),
            "new_followers_last_hour": new_followers
        }

    def _get_user_balance(self, user_id):
        """获取用户余额"""
        balance = self.redis.get(f"user_balance:{user_id}")
        if balance:
            return Decimal(balance.decode() if isinstance(balance, bytes) else balance)
        result = self.db.query_one(
            "SELECT balance FROM user_wallets WHERE user_id = %s", user_id)
        if result:
            return Decimal(str(result["balance"]))
        return Decimal("0")

    def _deduct_balance(self, user_id, amount):
        """扣除余额"""
        new_balance = self._get_user_balance(user_id) - amount
        self.db.update("user_wallets",
            {"balance": str(new_balance.quantize(Decimal("0.01"))),
             "updated_at": now()},
            {"user_id": user_id})
        self.redis.set(f"user_balance:{user_id}",
            str(new_balance.quantize(Decimal("0.01"))))
```

## 异常场景补充

### 场景：礼物打赏余额并发扣减导致超扣

```
触发：用户余额 100 元 → 同时发送 2 个 80 元礼物 → 两次都扣减成功 → 余额变负
检测：
  1. 用户余额 < 0 → 超扣
  2. 扣减记录总额 > 充值总额 → 不一致
处理：
  1. 余额扣减使用 Redis Lua 原子操作
  2. 数据库层面增加 CHECK 约束（balance >= 0）
  3. 发现超扣 → 冻结账户 + 追溯修复
预防：Lua 原子操作 + CHECK 约束 + 账户冻结
```

### 场景：互动投票被刷票

```
触发：用户使用多账号给同一选项投票 → 结果失真 → 其他用户不满
检测：
  1. 同 IP 多账号投票 → 刷票
  2. 投票速率异常（1 分钟内 > 100 票）→ 机器刷票
处理：
  1. 投票需登录且绑定手机号
  2. 同 IP 投票限制（最多 3 票）
  3. 异常投票不计入统计
预防：登录验证 + IP 限制 + 异常过滤
```

### 场景：直播礼物特效导致观众端卡顿

```
触发：5 秒内收到 20 个城堡礼物 → 特效动画同时播放 → 观众 GPU 占满 → 卡顿
检测：
  1. 观众反馈卡顿 → 特效过载
  2. 特效队列深度 > 5 → 堆积
处理：
  1. 特效队列限制（最多同时播放 3 个）
  2. 相同礼物合并显示（10 个火箭合并为 1 个大火箭）
  3. 低端设备自动降低特效质量
预防：队列限制 + 合并显示 + 质量降级
```
