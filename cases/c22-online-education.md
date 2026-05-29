# C22: 在线教育的万人直播课堂

## 业务场景

某在线教育平台，支持万人同时在线的直播课堂。教育直播与娱乐直播的核心区别在于**强互动**：老师提问 → 学生举手 → 老师点名 → 学生回答 → 全班看到。同时需要白板同步、课件翻页、课堂练习等功能。

**一个真实的故障：** 2024 年某 K12 平台期末直播课，3 万学生同时在线。老师发起随堂测验 → 3 万学生同时提交答案 → 服务器瞬间收到 3 万条消息 → 消息队列积压 → 老师端 5 分钟后才看到结果 → 课堂节奏被打断 → 大量学生退出 → 投诉 5000+ 条。

**已知数据：**
- 单课堂最大人数：1 万学生
- 峰值同时开课：200 堂
- 互动延迟要求：< 1 秒（老师点名到学生收到）
- 白板同步延迟：< 500ms
- 课堂时长：45-90 分钟
- 回放保留：永久
- 同时在线峰值：200 万（200 堂 × 1 万/堂）

**为什么比娱乐直播难？**

| 维度 | 娱乐直播 | 教育直播 |
|------|---------|---------|
| 数据流 | 单向（主播→观众） | 双向（老师↔学生） |
| 互动频率 | 低（弹幕是异步的） | 高（提问→回答是同步的） |
| 数据一致性 | 弹幕丢了无所谓 | 答题数据不能丢 |
| 延迟容忍 | 3-5 秒可接受 | < 1 秒（互动需要即时反馈） |
| 白板同步 | 无 | 必须（公式一个点都不能少） |

## 核心挑战

### 挑战 1：老师操作的扇出

老师翻一页课件 → 1 万学生同时收到翻页指令。

**量化分析：** 如果逐个 WebSocket send → 1 万次 → 每次 0.1ms → 总延迟 1 秒 → 超过 1 秒限制。更糟的是，1 万次 send 不是并行的——单线程事件循环中是串行的。

### 挑战 2：学生互动的汇聚

老师发起一道选择题 → 1 万学生同时提交答案 → 老师端 1 万条消息涌入 → 不可能逐条看。需要实时聚合：选A的 3000 人，选B的 5000 人...

**量化：** 1 万学生 × 200 字节/条 = 2MB 瞬间涌入 → 老师端 WebSocket 缓冲区可能溢出。

### 挑战 3：白板数据量大且需可靠

老师用电子白板手写公式 → 每秒约 100 个笔画点 → 每个点约 20 字节 → 2KB/秒。

**带宽计算：** 1 万学生 × 2KB/秒 = 20MB/秒出站带宽。一个机房 10Gbps 出口 → 200 堂课 × 20MB/秒 = 4GB/秒 → 占 32% 出口带宽 → 可行但需要优化。弱网学生丢包 → 笔画断裂 → 公式看不懂。

### 挑战 4：课堂回放

所有课堂事件（翻页、白板、问答、举手）需要按时间轴回放。回放时互动需要与视频进度同步。

**数据量：** 90 分钟课堂 × 2KB/秒白板 + 互动事件 ≈ 10MB/堂。200 堂/天 × 365 天 × 10MB = 730GB/年。

## 设计约束

- WebSocket 长连接
- 互动延迟 < 1 秒
- 白板同步延迟 < 500ms
- 课堂事件永久存储（回放用）
- 单课堂出站带宽 < 50MB/秒
- 消息不丢失（答题数据）

## 请先独立思考（限时 30 分钟）

1. 老师操作如何高效扇出给 1 万学生？逐个推送 vs Redis Pub/Sub vs 消息队列？各方案的延迟是多少？
2. 学生答案如何聚合？逐条推给老师 vs 窗口聚合 vs 实时统计？聚合窗口多大合适？
3. 白板数据如何处理弱网？增量推送 + 断线补偿如何实现？客户端如何检测断裂？
4. 课堂回放如何实现？事件流如何与视频进度同步？回放时白板如何重绘？

---

## 设计解析

### 架构：信令服务 + 数据服务分离

```
                    ┌──────────────┐
                    │   老师端     │
                    └──────┬───────┘
                           │ WebSocket
                    ┌──────┴───────┐
                    │  信令服务     │ ← 老师操作（翻页/点名/白板）
                    │  (无状态)     │ ← 学生互动（举手/答题）
                    └──────┬───────┘
                           │ Redis Pub/Sub
              ┌────────────┼────────────┐
              │            │            │
       ┌──────┴──┐  ┌──────┴──┐  ┌──────┴──┐
       │网关 A   │  │网关 B   │  │网关 C   │
       │(50学生) │  │(50学生) │  │(50学生) │
       └─────────┘  └─────────┘  └─────────┘

       网关 A 本地广播 → 50 个 WebSocket send（并行）
       总计：1 次 publish → 200 个网关各自广播
```

### 老师操作扇出：Redis Pub/Sub + 网关批量推送

```python
class TeacherActionDispatcher:
    def dispatch(self, classroom_id, action):
        """老师操作推送给所有学生"""
        # 1. 写入课堂事件流（回放用 + 断线补偿用）
        event = {
            "event_id": uuid4(),
            "type": action.type,        # page_flip / whiteboard / call_on
            "data": action.data,
            "classroom_id": classroom_id,
            "timestamp": now_ms()
        }
        self.event_store.append(classroom_id, event)

        # 2. 通过 Redis Pub/Sub 广播
        # 一次 publish → 所有订阅了该课堂的网关都收到
        self.redis.publish(
            f"classroom:{classroom_id}", 
            json.dumps(event)
        )

        # 3. 如果是关键操作（翻页/点名），同时写 Redis Stream
        # Redis Stream 支持消费者组 → 断线重连后从 last_id 拉取
        if action.type in ("page_flip", "call_on"):
            self.redis.xadd(f"stream:{classroom_id}", event)
```

**为什么用 Redis Pub/Sub 而不是逐个推送？**

| 方案 | 发布延迟 | 总推送次数 | 网络跳数 |
|------|---------|-----------|---------|
| 逐个 WebSocket send | O(N) × 0.1ms | 10000 | 10000 |
| Redis Pub/Sub + 网关广播 | O(1) publish | 1 + 200（网关数） | 201 |
| Kafka | O(1) 但延迟高 | 1 + 200 | 201 |

Redis Pub/Sub 延迟最低（< 1ms publish），但消息不持久化 → 需要同时写 Redis Stream / 事件流做补偿。

```python
class ClassroomGateway:
    """WebSocket 网关：管理本地学生的连接"""

    def __init__(self):
        self.classroom_students = {}  # classroom_id → set of websocket

    def on_classroom_message(self, classroom_id, message):
        """收到 Pub/Sub 消息 → 批量推送给本地学生"""
        students = self.classroom_students.get(classroom_id, set())
        
        if not students:
            return
        
        # 批量推送：使用 asyncio.gather 并行发送
        # 比 for 循环逐个 send 快 10 倍
        data = message.encode('utf-8')
        tasks = [ws.send(data) for ws in students]
        asyncio.gather(*tasks, return_exceptions=True)

    def on_student_connect(self, ws, classroom_id, student_id):
        """学生连接 → 加入课堂"""
        if classroom_id not in self.classroom_students:
            self.classroom_students[classroom_id] = set()
            # 首次有学生订阅该课堂 → subscribe Redis channel
            self.redis.subscribe(f"classroom:{classroom_id}")
        
        self.classroom_students[classroom_id].add(ws)

    def on_student_disconnect(self, ws, classroom_id):
        """学生断连 → 移出课堂"""
        self.classroom_students[classroom_id].discard(ws)
        if not self.classroom_students[classroom_id]:
            # 该课堂无本地学生 → unsubscribe
            self.redis.unsubscribe(f"classroom:{classroom_id}")
            del self.classroom_students[classroom_id]
```

**扇出延迟分析：**

```
老师操作 → Redis publish (< 1ms)
         → 200 个网关收到 (< 1ms, fan-out)
         → 各网关批量推送给本地学生 (并行, < 10ms)
──────────────────────────────────
总延迟: < 15ms (远低于 1 秒限制)
```

### 学生互动聚合：2 秒窗口 + 实时统计

**关键洞察：老师不需要逐条看每个学生的答案，只需要看聚合统计。**

```python
class AnswerAggregator:
    def __init__(self):
        self.buffers = {}      # question_id → Counter
        self.total_answers = {} # question_id → total count
        self.window_ms = 2000  # 2 秒聚合窗口

    def on_student_answer(self, classroom_id, question_id, answer, student_id):
        """学生提交答案 → 实时聚合"""
        if question_id not in self.buffers:
            self.buffers[question_id] = Counter()
            self.total_answers[question_id] = 0
            # 注册 2 秒后首次刷新
            self.schedule_flush(classroom_id, question_id, delay=2)

        self.buffers[question_id][answer] += 1
        self.total_answers[question_id] += 1

        # 同时写入事件流（回放用）
        self.event_store.append(classroom_id, {
            "type": "student_answer",
            "question_id": question_id,
            "student_id": student_id,
            "answer": answer,
            "timestamp": now_ms()
        })

    def flush_to_teacher(self, classroom_id, question_id):
        """每 2 秒推送聚合结果给老师"""
        counts = dict(self.buffers.get(question_id, {}))
        total = self.total_answers.get(question_id, 0)

        if total == 0:
            self.schedule_flush(classroom_id, question_id, delay=2)
            return

        self.push_to_teacher(classroom_id, {
            "type": "answer_aggregate",
            "question_id": question_id,
            "counts": counts,        # {"A": 3000, "B": 5000, "C": 1500, "D": 500}
            "total": total,
            "percentage": {k: round(v/total*100, 1) for k, v in counts.items()},
            "timestamp": now_ms()
        })

        # 清空缓冲（下一轮聚合）
        self.buffers[question_id] = Counter()
        self.schedule_flush(classroom_id, question_id, delay=2)
```

**聚合效果量化：**

| 维度 | 逐条推送 | 2秒聚合 |
|------|---------|--------|
| 老师端消息量 | 1万条/秒 | 1条/2秒 |
| 老师端信息量 | 无法阅读 | 4个选项百分比 |
| 网络带宽 | 2MB/秒 | 200字节/2秒 |
| 数据完整性 | 全量 | 全量（原始答案也存了） |

**主观题的特殊处理：**

```python
class SubjectiveAnswerHandler:
    def on_subjective_answer(self, classroom_id, question_id, answer, student_id):
        """主观题：不能聚合，但可以采样展示"""
        # 存储所有答案（回放用）
        self.event_store.append(classroom_id, {
            "type": "subjective_answer",
            "question_id": question_id,
            "student_id": student_id,
            "answer": answer,
            "timestamp": now_ms()
        })

        # 只推给老师最近的 5 条答案（滚动展示）
        # 避免老师端消息量过大
        self.push_to_teacher(classroom_id, {
            "type": "subjective_answer_sample",
            "question_id": question_id,
            "answer": answer[:200],  # 截断过长答案
            "student_name": self.get_student_name(student_id),
            "timestamp": now_ms()
        })
```

### 白板同步：增量推送 + 序号 + 断线补偿

```python
class WhiteboardSync:
    def __init__(self):
        self.seq_counter = {}  # classroom_id → 当前 seq

    def on_stroke_point(self, classroom_id, point):
        """老师白板笔画点 → 增量推送给所有学生"""
        seq = self.next_seq(classroom_id)
        
        message = {
            "type": "wb_point",
            "stroke_id": point.stroke_id,
            "x": point.x,
            "y": point.y,
            "color": point.color,
            "width": point.width,
            "seq": seq,
            "timestamp": now_ms()
        }
        
        # 1. Redis Pub/Sub 广播（实时推送）
        self.redis.publish(f"wb:{classroom_id}", json.dumps(message))
        
        # 2. 写入 Redis Stream（断线补偿用，保留 2 小时）
        self.redis.xadd(f"wb_stream:{classroom_id}", {
            "seq": str(seq),
            "data": json.dumps(message)
        }, maxlen=100000)  # 最多保留 10 万条（约 16 分钟的笔画）

    def on_student_sync_request(self, classroom_id, student_id, last_seq):
        """弱网学生请求补发缺失的笔画点"""
        # 从 Redis Stream 中读取 last_seq 之后的所有点
        points = self.redis.xrange(
            f"wb_stream:{classroom_id}",
            min=f"{last_seq + 1}-0",
            max="+"
        )
        
        self.push_to_student(student_id, {
            "type": "wb_compensate",
            "points": [json.loads(p[1]["data"]) for p in points],
            "from_seq": last_seq + 1
        })

    def on_stroke_complete(self, classroom_id, stroke_id, all_points):
        """笔画完成 → 持久化到对象存储（回放用）"""
        # 将完整笔画存储到 S3/OSS
        key = f"whiteboard/{classroom_id}/{stroke_id}.json"
        self.s3.put(key, json.dumps(all_points))
        
        # 同时写入课堂事件流
        self.event_store.append(classroom_id, {
            "type": "whiteboard_stroke",
            "stroke_id": stroke_id,
            "s3_key": key,
            "timestamp": now_ms()
        })
```

**客户端侧断线检测与补偿：**

```javascript
class WhiteboardRenderer {
    constructor() {
        this.expectedSeq = 0;
        this.strokes = new Map();  // stroke_id → points[]
        this.syncThrottle = null;  // 防止频繁请求补偿
    }

    onPoint(data) {
        // 检测断裂
        if (data.seq > this.expectedSeq + 1) {
            this.requestSyncThrottled(this.expectedSeq);
        }
        
        // 渲染笔画点
        if (!this.strokes.has(data.stroke_id)) {
            this.strokes.set(data.stroke_id, []);
        }
        this.strokes.get(data.stroke_id).push(data);
        this.renderPoint(data);
        
        this.expectedSeq = data.seq;
    }

    requestSyncThrottled(lastSeq) {
        // 节流：最多 2 秒请求一次补偿
        if (this.syncThrottle) return;
        this.syncThrottle = setTimeout(() => {
            this.ws.send(JSON.stringify({
                type: "wb_sync_request",
                last_seq: lastSeq
            }));
            this.syncThrottle = null;
        }, 2000);
    }

    onCompensate(data) {
        // 收到补偿数据 → 重绘缺失的笔画
        data.points.forEach(point => {
            this.renderPoint(point);
            this.expectedSeq = Math.max(this.expectedSeq, point.seq);
        });
    }
}
```

### 举手与点名

```python
class HandRaiseService:
    def on_hand_raise(self, classroom_id, student_id):
        """学生举手"""
        self.redis.sadd(f"hands:{classroom_id}", student_id)
        # 只推给老师（不推给全班，避免干扰）
        self.push_to_teacher(classroom_id, {
            "type": "hand_raise",
            "student_id": student_id,
            "student_name": self.get_student_name(student_id),
            "timestamp": now_ms()
        })

    def on_call_on(self, classroom_id, student_id):
        """老师点名"""
        self.redis.srem(f"hands:{classroom_id}", student_id)
        
        # 推给被点名学生：允许发言/开麦
        self.push_to_student(student_id, {
            "type": "called_on",
            "classroom_id": classroom_id
        })
        
        # 推给全班：XX 同学正在发言
        self.broadcast_classroom(classroom_id, {
            "type": "student_speaking",
            "student_name": self.get_student_name(student_id)
        })

    def get_hand_raise_list(self, classroom_id):
        """老师查看举手列表"""
        student_ids = self.redis.smembers(f"hands:{classroom_id}")
        return [self.get_student_info(sid) for sid in student_ids]
```

### 课堂回放：事件流 + 时间轴同步

```python
class ReplayService:
    def get_replay_data(self, classroom_id):
        """获取课堂回放数据"""
        events = self.event_store.get_all(classroom_id)
        events.sort(key=lambda e: e.timestamp)
        
        start_time = events[0].timestamp
        # 将绝对时间转换为相对时间（方便回放定位）
        for event in events:
            event.relative_ms = event.timestamp - start_time
        
        return {
            "classroom_id": classroom_id,
            "events": events,
            "duration_ms": events[-1].relative_ms,
            "video_url": self.get_video_url(classroom_id)
        }

    def get_events_at_progress(self, classroom_id, progress_ms, window_ms=500):
        """根据视频播放进度获取对应时间点的事件"""
        return self.event_store.query_range(
            classroom_id,
            from_time=progress_ms - window_ms,
            to_time=progress_ms
        )

    def get_whiteboard_snapshot(self, classroom_id, progress_ms):
        """获取某时间点的白板快照（用于 seek 后重绘）"""
        # 获取该时间点之前的所有完整笔画
        strokes = self.event_store.query_range(
            classroom_id,
            from_time=0,
            to_time=progress_ms,
            event_type="whiteboard_stroke"
        )
        
        # 从 S3 加载笔画数据
        result = []
        for stroke in strokes:
            data = self.s3.get(stroke.s3_key)
            result.append(json.loads(data))
        
        return result
```

**客户端回放逻辑：**

```javascript
class ClassroomReplay {
    constructor(classroomId) {
        this.classroomId = classroomId;
        this.lastProgress = 0;
        this.whiteboard = new WhiteboardRenderer();
    }

    async init() {
        // 一次性加载所有事件的索引（不加载白板点数据）
        this.events = await fetchEvents(this.classroomId);
    }

    onVideoProgress(progressMs) {
        // 查找该时间段内的事件
        const newEvents = this.events.filter(e => 
            e.relative_ms > this.lastProgress && e.relative_ms <= progressMs
        );
        
        newEvents.forEach(event => this.applyEvent(event));
        this.lastProgress = progressMs;
    }

    async onVideoSeek(progressMs) {
        // 用户拖动进度条 → 需要重绘白板
        const snapshot = await fetchWhiteboardSnapshot(this.classroomId, progressMs);
        this.whiteboard.clear();
        snapshot.forEach(stroke => this.whiteboard.renderStroke(stroke));
        this.lastProgress = progressMs;
    }

    applyEvent(event) {
        switch (event.type) {
            case "page_flip":
                this.renderSlide(event.data.page_number);
                break;
            case "answer_aggregate":
                this.renderChart(event.data);
                break;
            case "whiteboard_stroke":
                this.whiteboard.renderStroke(event.data);
                break;
        }
    }
}
```

### 网关容量规划

**单网关容量：**
- WebSocket 连接：10 万（单机内存 16GB，每连接 ~50KB）
- CPU：广播 50 个连接 < 1ms
- 出站带宽：10Gbps

**200 万在线学生 × 每网关 10 万连接 = 20 台网关。**

| 维度 | 需求 | 单网关能力 | 所需网关数 |
|------|------|-----------|-----------|
| WebSocket 连接 | 200 万 | 10 万 | 20 |
| 出站带宽 | 4GB/秒 | 10Gbps | 4 |
| CPU（广播） | 200 堂 × 50 连接 | 1000 堂 | 1 |

瓶颈是连接数 → 需要 20 台网关。

## 常见陷阱（深度分析）

### 陷阱 1：学生答案逐条推给老师

**后果：** 1 万条/秒涌入老师端 → WebSocket 缓冲区溢出 → 老师端卡死 → 课堂中断。3 万学生的那次故障就是这个问题——服务器端逐条转发答案，老师端收到 3 万条消息后浏览器 OOM。

**解决方案：** 2 秒聚合窗口 → 推送百分比统计。选择题聚合为 4 个选项的计数，主观题采样展示最近 5 条。老师端只收到 1 条/2秒。

### 陷阱 2：白板全量推送

**后果：** 每次推完整画面（100KB PNG）→ 1 万学生 × 100KB = 1GB/秒出站带宽 → 远超单机房出口能力。

**解决方案：** 增量推送（只推新的笔画点，约 2KB/秒）→ 1 万学生 × 2KB = 20MB/秒 → 可行。

### 陷阱 3：不处理弱网补偿

**后果：** 移动网络丢包率 2-5% → 每秒 100 个笔画点中丢失 2-5 个 → 公式笔画断裂 → 学生看不懂 → 投诉。

**解决方案：** 序号标记 + 断线请求补发。客户端检测 seq 不连续 → 请求补偿 → 服务端从 Redis Stream 补发缺失数据。

### 陷阱 4：课堂事件不持久化

**后果：** 无法回放 → 学生错过课堂无法复习 → 教学价值减半。合规要求也可能需要保留课堂记录。

**解决方案：** 事件流追加写入对象存储，永久保留。回放时按时间轴重放事件。白板笔画按 S3 key 存储，回放时从 S3 加载。

### 陷阱 5：Redis Pub/Sub 消息丢失

**后果：** Redis Pub/Sub 不持久化 → 网关短暂断连期间的消息全部丢失 → 学生错过翻页 → 白板断裂。

**解决方案：** Pub/Sub（实时）+ Redis Stream（补偿）双重保障。网关重连后从 Stream 的 last_id 拉取缺失消息。

### 陷阱 6：回放时白板只重放增量点

**后果：** 回放到 50 分钟时 seek 到 30 分钟 → 白板只有 30 分钟之后的增量点 → 30 分钟之前的笔画丢失。

**解决方案：** seek 时请求该时间点的白板快照（从 S3 加载完整笔画数据）→ 重绘整个白板。

## 延伸思考

- **分组讨论**：1 万人分 20 个 500 人小组 → 动态分组 + 组内消息隔离 + 组间切换。需要"分组路由"逻辑——每个小组一个 Redis channel。
- **AI 助教**：实时分析答题分布，提示老师"选 B 的比例异常高（65%），建议重新讲解这个概念"。需要 NLP 分析答案内容。
- **VR 课堂**：3D 虚拟教室，学生位置和朝向需同步。白板变为 3D 空间中的书写面，带宽需求增加 10 倍。