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

    def on_connect(self, gateway_id, room_id, user_id):
        """观众连接时注册路由"""
        # 记录：该房间在哪些网关上有观众
        self.redis.sadd(f"ws:room:{room_id}", gateway_id)
        # 记录：该网关服务哪些房间
        self.redis.sadd(f"ws:gateway:{gateway_id}", room_id)
        # 记录：用户的连接信息（用于定向推送）
        self.redis.set(f"ws:user:{user_id}", gateway_id)

    def on_disconnect(self, gateway_id, room_id, user_id):
        """观众断开时清理路由"""
        # 从网关的房间连接列表中移除
        # 如果该房间在此网关上无观众了，取消订阅
        remaining = self.redis.scard(f"ws:room_connections:{room_id}:{gateway_id}")
        if remaining <= 1:
            self.redis.srem(f"ws:room:{room_id}", gateway_id)
            self.redis.srem(f"ws:gateway:{gateway_id}", room_id)
```

**网关宕机恢复：**

```
1. 心跳检测发现网关 G5 宕机
2. 清理 G5 的路由信息
3. G5 上的 5 万个连接客户端自动检测断线
4. 客户端自动重连 → DNS/负载均衡分配到其他网关
5. 新网关重新注册路由
6. 重新订阅 Redis Pub/Sub

恢复时间：约 5-10 秒（取决于客户端重连策略）
```

### 付费弹幕的可靠送达

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

### 带宽计算

**下行带宽（弹幕推送给观众）：**
- 正常直播间：每 200ms 约 5 条弹幕 × 50 字节 = 1.25 KB/s/观众
- 热门直播间：每 200ms 约 25 条弹幕 × 50 字节 = 6.25 KB/s/观众
- 1000 万观众 × 平均 2 KB/s = 20 GB/s 出站带宽

**上行带宽（观众发弹幕）：**
- 100 万条/秒 × 50 字节 = 50 MB/s → 可忽略

**结论：** 下行带宽是瓶颈，约 20 GB/s。需要 CDN 或边缘节点分发。

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
- **VR 直播弹幕**：3D 空间中的弹幕定位——弹幕不再只是水平飘过，而是在 3D 空间中定位。需要客户端做 3D 渲染，服务端下发 3D 坐标。