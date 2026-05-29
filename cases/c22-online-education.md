# C22: 在线教育的万人直播课堂

## 业务场景

某在线教育平台，支持万人同时在线的直播课堂。教育直播与娱乐直播的核心区别在于**强互动**：老师提问 → 学生举手 → 老师点名 → 学生回答 → 全班看到。同时需要白板同步、课件翻页、课堂练习等功能。

**已知数据：**
- 单课堂最大人数：1 万学生
- 峰值同时开课：200 堂
- 互动延迟要求：< 1 秒（老师点名到学生收到）
- 白板同步延迟：< 500ms
- 课堂时长：45-90 分钟
- 回放保留：永久

**为什么是难题？**

教育直播是"1 对 N 的互动"而非单向广播：
- 老师的操作（翻页、画线、点名）要推给 1 万学生 → **扇出问题**
- 学生的操作（举手、答题）要推给老师 → **汇聚问题**
- 1 万学生同时提交答案 → 瞬间 1 万条消息 → 老师端如何展示？
- 白板笔画需要精确同步 → 弱网学生笔画断裂

## 核心挑战

### 挑战 1：老师操作的扇出

老师翻一页课件 → 1 万学生同时收到翻页指令。如果逐个 WebSocket send → 1 万次 → 延迟可能超过 1 秒。

### 挑战 2：学生互动的汇聚

老师发起一道选择题 → 1 万学生同时提交答案 → 老师端 1 万条消息涌入 → 不可能逐条看。需要实时聚合：选A的 3000 人，选B的 5000 人...

### 挑战 3：白板数据量大且需可靠

老师用电子白板手写公式 → 每秒约 100 个笔画点 → 每个点约 20 字节 → 2KB/秒。1 万学生 × 2KB/秒 = 20MB/秒出站带宽。弱网学生丢包 → 笔画断裂。

### 挑战 4：课堂回放

所有课堂事件（翻页、白板、问答、举手）需要按时间轴回放。回放时弹幕/互动需要与视频进度同步。

## 设计约束

- WebSocket 长连接
- 互动延迟 < 1 秒
- 白板同步延迟 < 500ms
- 课堂事件永久存储（回放用）

## 请先独立思考（限时 30 分钟）

1. 老师操作如何高效扇出给 1 万学生？逐个推送 vs Redis Pub/Sub vs 消息队列？
2. 学生答案如何聚合？逐条推给老师 vs 窗口聚合 vs 实时统计？
3. 白板数据如何处理弱网？增量推送 + 断线补偿如何实现？
4. 课堂回放如何实现？事件流如何与视频进度同步？

---

## 设计解析

### 架构：信令服务 + 数据服务分离

```
老师操作（翻页/点名/白板）→ 信令服务 → Redis Pub/Sub → 各 WebSocket 网关 → 学生

学生互动（举手/答题）→ WebSocket 网关 → 聚合服务 → 老师
```

### 老师操作扇出：Redis Pub/Sub + 批量推送

```python
class TeacherActionDispatcher:
    def dispatch(self, classroom_id, action):
        """老师操作推送给所有学生"""
        # 1. 写入课堂事件流（回放用）
        self.event_store.append(classroom_id, {
            "type": action.type,
            "data": action.data,
            "timestamp": now_ms()
        })

        # 2. 通过 Redis Pub/Sub 广播（一次 publish，所有网关收到）
        self.redis.publish(f"classroom:{classroom_id}", json.dumps(action))
```

**为什么用 Redis Pub/Sub？**

- 热门课堂的 1 万学生分布在约 200 台 WebSocket 网关上
- 老师操作只需 publish 1 次 → 200 个网关各自收到 → 各网关本地广播
- 发布 O(1)，总推送 O(网关数) 而非 O(学生数)

```python
class ClassroomGateway:
    def on_classroom_message(self, classroom_id, message):
        """WebSocket 网关收到 Pub/Sub 消息 → 批量推送给本地学生"""
        students = self.get_local_students(classroom_id)
        # 使用 WebSocket 库的 broadcast 方法（批量 send，比逐个快 10 倍）
        self.ws_server.broadcast(students, message)
```

### 学生互动聚合：2 秒窗口 + 实时统计

**关键洞察：老师不需要逐条看每个学生的答案，只需要看聚合统计。**

```python
class AnswerAggregator:
    def __init__(self):
        self.buffers = {}  # question_id → Counter
        self.window_ms = 2000  # 2 秒聚合窗口

    def on_student_answer(self, classroom_id, question_id, answer, student_id):
        """学生提交答案 → 实时聚合"""
        if question_id not in self.buffers:
            self.buffers[question_id] = Counter()
            # 注册 2 秒后刷新
            self.schedule_flush(question_id, delay=2)

        self.buffers[question_id][answer] += 1

    def flush_to_teacher(self, question_id):
        """每 2 秒推送聚合结果给老师"""
        counts = dict(self.buffers.get(question_id, {}))
        total = sum(counts.values())

        if total == 0:
            return

        self.push_to_teacher({
            "type": "answer_aggregate",
            "question_id": question_id,
            "counts": counts,        # {"A": 3000, "B": 5000, "C": 1500, "D": 500}
            "total": total,
            "percentage": {k: round(v/total*100, 1) for k, v in counts.items()},
            "timestamp": now_ms()
        })

        # 清空缓冲（下一轮聚合）
        self.buffers[question_id] = Counter()
        self.schedule_flush(question_id, delay=2)
```

**聚合效果：**
- 1 万条答案 → 聚合为 4 个选项的百分比 → 信息完整，展示清晰
- 老师端只收到 1 条聚合消息/2秒 → 而非 1 万条/秒

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
            "answer": answer
        })

        # 只推送给老师最近的 5 条答案（滚动展示）
        self.push_to_teacher({
            "type": "subjective_answer_sample",
            "question_id": question_id,
            "answer": answer,
            "student_name": self.get_student_name(student_id)
        })
```

### 白板同步：增量推送 + 断线补偿

```python
class WhiteboardSync:
    def __init__(self):
        self.seq_counter = {}  # classroom_id → seq

    def on_stroke_point(self, classroom_id, point):
        """老师白板笔画点 → 增量推送给所有学生"""
        # 分配序号（用于断线补偿）
        seq = self.next_seq(classroom_id)
        
        self.redis.publish(f"whiteboard:{classroom_id}", json.dumps({
            "type": "point",
            "stroke_id": point.stroke_id,
            "x": point.x, "y": point.y,
            "seq": seq,
            "timestamp": now_ms()
        }))

        # 同时写入事件流（断线补偿用）
        self.event_store.append(classroom_id, {
            "type": "whiteboard_point",
            "seq": seq,
            "data": point.__dict__
        })

    def on_student_sync_request(self, classroom_id, student_id, last_seq):
        """弱网学生请求补发缺失的笔画点"""
        # 从事件流中获取 last_seq 之后的所有点
        points = self.event_store.get_since(classroom_id, last_seq, event_type="whiteboard_point")
        self.push_to_student(student_id, {
            "type": "whiteboard_compensate",
            "points": points,
            "from_seq": last_seq + 1
        })
```

**弱网补偿流程：**

```
学生检测到笔画断裂（seq 不连续）→ 发送 sync_request(last_seq)
→ 服务端补发 last_seq 之后的所有点
→ 学生重连笔画 → 恢复完整画面
```

**客户端侧检测：**

```javascript
// 学生端白板渲染
let expectedSeq = 0;

function onWhiteboardPoint(data) {
    if (data.seq > expectedSeq + 1) {
        // 检测到断裂 → 请求补偿
        requestSync(expectedSeq);
    }
    renderPoint(data);
    expectedSeq = data.seq;
}
```

### 举手与点名

```python
class HandRaiseService:
    def on_hand_raise(self, classroom_id, student_id):
        """学生举手"""
        self.redis.sadd(f"hands:{classroom_id}", student_id)
        # 只推给老师（不推给全班，避免干扰）
        self.push_to_teacher({
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
    def replay(self, classroom_id, video_start_time):
        """回放课堂：按时间轴重放所有事件"""
        # 从事件流获取课堂所有事件
        events = self.event_store.get_all(classroom_id)
        
        # 按时间戳排序
        events.sort(key=lambda e: e.timestamp)
        
        return {
            "classroom_id": classroom_id,
            "events": events,
            "duration_ms": events[-1].timestamp - events[0].timestamp
        }

    def get_events_at_progress(self, classroom_id, progress_ms):
        """根据视频播放进度获取对应时间点的事件"""
        base_time = self.get_classroom_start_time(classroom_id)
        target_time = base_time + progress_ms
        
        return self.event_store.query_range(
            classroom_id,
            from_time=target_time - 500,   # 前 500ms
            to_time=target_time
        )
```

**客户端回放逻辑：**

```javascript
// 视频播放进度变化时，请求对应时间点的事件
videoElement.ontimeupdate = (event) => {
    const progressMs = event.currentTime * 1000;
    fetchEvents(classroomId, progressMs).then(events => {
        events.forEach(applyEvent);  // 应用事件（翻页、白板、问答等）
    });
};
```

## 常见陷阱（深度分析）

### 陷阱 1：学生答案逐条推给老师

**后果：** 1 万条/秒涌入老师端 → 无法阅读 → 互动瘫痪。老师只能看到消息刷屏，无法做出教学决策。

**解决方案：** 2 秒聚合窗口 → 推送百分比统计。选择题聚合为 4 个选项的计数，主观题采样展示最近 5 条。

### 陷阱 2：白板全量推送

**后果：** 每次推完整画面 → 1 万学生 × 100KB 画面 = 1GB/秒出站带宽 → 不可行。

**解决方案：** 增量推送（只推新的笔画点）→ 约 2KB/秒/学生 → 总带宽 20MB/秒。

### 陷阱 3：不处理弱网补偿

**后果：** 移动网络丢包 → 笔画断裂 → 公式看不懂 → 学习体验差。学生可能因此退出课堂。

**解决方案：** 序号标记 + 断线请求补发。客户端检测 seq 不连续 → 请求补偿 → 服务端补发缺失数据。

### 陷阱 4：课堂事件不持久化

**后果：** 无法回放 → 学生错过课堂无法复习 → 教学价值减半。合规要求也可能需要保留课堂记录。

**解决方案：** 事件流追加写入对象存储，永久保留。回放时按时间轴重放事件。

## 延伸思考

- **分组讨论**：1 万人分 20 个 500 人小组 → 动态分组 + 组内消息隔离 + 组间切换。需要额外的"分组路由"逻辑。
- **AI 助教**：实时分析答题分布，提示老师"选 B 的比例异常高（65%），建议重新讲解这个概念"。需要 NLP 分析答案内容。
- **VR 课堂**：3D 虚拟教室，学生位置和朝向需同步。白板变为 3D 空间中的书写面，带宽需求增加 10 倍。