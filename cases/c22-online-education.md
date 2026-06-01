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

## 数据库设计

### 核心表结构

```sql
-- 课程表
CREATE TABLE courses (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    title           VARCHAR(200) NOT NULL COMMENT '课程名称',
    teacher_id      BIGINT NOT NULL COMMENT '授课老师ID',
    subject         VARCHAR(50) NOT NULL COMMENT '学科',
    grade_level     TINYINT NOT NULL COMMENT '年级段: 1小学 2初中 3高中 4大学 5成人',
    max_students    INT NOT NULL DEFAULT 10000 COMMENT '单课堂最大学生数',
    class_type      TINYINT NOT NULL DEFAULT 1 COMMENT '课堂类型: 1大班课 2小班课 3一对一',
    status          TINYINT NOT NULL DEFAULT 0 COMMENT '0草稿 1已发布 2已归档',
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_teacher (teacher_id),
    INDEX idx_subject_grade (subject, grade_level)
) ENGINE=InnoDB COMMENT='课程表';

-- 课次/课时表（每堂课一个记录）
CREATE TABLE sessions (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    course_id       BIGINT NOT NULL COMMENT '所属课程',
    session_no      INT NOT NULL COMMENT '第几课时',
    title           VARCHAR(200) NOT NULL COMMENT '课时标题',
    teacher_id      BIGINT NOT NULL COMMENT '授课老师',
    start_time      DATETIME NOT NULL COMMENT '计划开课时间',
    end_time        DATETIME NOT NULL COMMENT '计划结束时间',
    actual_start    DATETIME NULL COMMENT '实际开课时间',
    actual_end      DATETIME NULL COMMENT '实际结束时间',
    status          TINYINT NOT NULL DEFAULT 0 COMMENT '0未开始 1直播中 2已结束 3异常终止',
    recording_url   VARCHAR(500) NULL COMMENT '回放视频地址',
    wb_snapshot_key VARCHAR(500) NULL COMMENT '白板最终快照S3 key',
    peak_students   INT NOT NULL DEFAULT 0 COMMENT '峰值在线人数',
    total_events    INT NOT NULL DEFAULT 0 COMMENT '课堂事件总数',
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_course_session (course_id, session_no),
    INDEX idx_teacher_time (teacher_id, start_time),
    INDEX idx_status_time (status, start_time)
) ENGINE=InnoDB COMMENT='课次表';

-- 白板操作表（增量存储，用于回放和断线补偿）
CREATE TABLE whiteboard_ops (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    session_id      BIGINT NOT NULL COMMENT '课次ID',
    seq             BIGINT NOT NULL COMMENT '操作序号（单调递增）',
    op_type         TINYINT NOT NULL COMMENT '操作类型: 1笔画点 2笔画完成 3清除 4撤销 5恢复',
    stroke_id       VARCHAR(64) NULL COMMENT '笔画ID',
    data            JSON NOT NULL COMMENT '操作数据（坐标/颜色/线宽等）',
    operator_id     BIGINT NOT NULL COMMENT '操作者ID（通常为老师）',
    timestamp_ms    BIGINT NOT NULL COMMENT '毫秒时间戳',
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_session_seq (session_id, seq),
    INDEX idx_session_timestamp (session_id, timestamp_ms)
) ENGINE=InnoDB COMMENT='白板操作记录';

-- 录制表（多流录制管理）
CREATE TABLE recordings (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    session_id      BIGINT NOT NULL COMMENT '课次ID',
    stream_type     TINYINT NOT NULL COMMENT '流类型: 1老师摄像头 2白板画面 3学生音频混流 4合流',
    format          VARCHAR(20) NOT NULL DEFAULT 'mp4' COMMENT '视频格式',
    storage_key     VARCHAR(500) NOT NULL COMMENT '对象存储key',
    file_size_mb    DECIMAL(10,2) NOT NULL COMMENT '文件大小MB',
    duration_ms     BIGINT NOT NULL COMMENT '时长毫秒',
    resolution      VARCHAR(20) NULL COMMENT '分辨率 1920x1080',
    bitrate_kbps    INT NULL COMMENT '码率kbps',
    status          TINYINT NOT NULL DEFAULT 0 COMMENT '0录制中 1已合成 2已归档 3录制失败',
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_session (session_id),
    INDEX idx_session_type (session_id, stream_type)
) ENGINE=InnoDB COMMENT='录制记录表';

-- 出勤表（签到/签退/异常记录）
CREATE TABLE attendance (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    session_id      BIGINT NOT NULL COMMENT '课次ID',
    student_id      BIGINT NOT NULL COMMENT '学生ID',
    join_time       DATETIME NULL COMMENT '进入课堂时间',
    leave_time      DATETIME NULL COMMENT '离开课堂时间',
    online_duration INT NOT NULL DEFAULT 0 COMMENT '在线时长秒',
    reconnect_count INT NOT NULL DEFAULT 0 COMMENT '断线重连次数',
    device_type     TINYINT NULL COMMENT '设备: 1PC 2iOS 3Android 4平板',
    network_type    TINYINT NULL COMMENT '网络: 1WiFi 24G 35G',
    is_complete     TINYINT NOT NULL DEFAULT 0 COMMENT '是否完整听课(>=80%时长)',
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_session_student (session_id, student_id),
    INDEX idx_student (student_id),
    INDEX idx_session_complete (session_id, is_complete)
) ENGINE=InnoDB COMMENT='出勤记录表';
```

**写入策略：** 出勤表高频更新（每次断线重连都更新），采用先写 Redis Hash（`attendance:{session_id}`）→ 课堂结束后批量落库。白板操作表是追加写入，课堂期间直接写入 MySQL（QPS 约 100/秒，单课堂完全可承受）。

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

### 白板 CRDT 同步：协作白板的冲突解决

教育场景中，小班课老师可能授权学生也上白板书写，此时出现多写者冲突。CRDT（Conflict-free Replicated Data Type）保证所有客户端最终状态一致，无需中心化冲突解决。

**核心思路：** 每个绘图元素（笔画、文本、图形）是一个 CRDT 元素，通过 Lamport 时间戳 + 节点 ID 保证全局唯一有序。删除使用"墓碑"标记（tombstone），不真正删除，保证各副本一致。

```python
import uuid
from dataclasses import dataclass, field
from typing import Dict, List, Optional

@dataclass
class CRDTElementId:
    """CRDT 元素唯一标识：Lamport 时间戳 + 节点 ID"""
    lamport_ts: int
    node_id: str  # 如 "teacher-1" / "student-42"

    def __lt__(self, other):
        if self.lamport_ts != other.lamport_ts:
            return self.lamport_ts < other.lamport_ts
        return self.node_id < other.node_id

@dataclass
class WhiteboardElement:
    """白板上的一个绘图元素（笔画/文本/图形）"""
    element_id: CRDTElementId
    element_type: str  # "stroke" / "text" / "shape" / "eraser"
    points: List[dict] = field(default_factory=list)  # [{x, y, pressure}]
    color: str = "#000000"
    width: float = 2.0
    text: str = ""       # 文本类型用
    shape: str = ""      # "rect" / "circle" / "line"
    deleted: bool = False  # 墓碑标记
    version: int = 1

class WhiteboardCRDT:
    """基于 CRDT 的白板状态管理"""

    def __init__(self, node_id: str):
        self.node_id = node_id
        self.lamport_clock = 0
        self.elements: Dict[str, WhiteboardElement] = {}  # key: "ts:node_id"

    def _next_id(self) -> CRDTElementId:
        """生成下一个元素 ID（Lamport 时钟递增）"""
        self.lamport_clock += 1
        return CRDTElementId(self.lamport_clock, self.node_id)

    def _element_key(self, eid: CRDTElementId) -> str:
        return f"{eid.lamport_ts}:{eid.node_id}"

    def add_stroke(self, points: list, color="#000000", width=2.0) -> dict:
        """本地添加笔画 → 返回操作日志用于广播"""
        eid = self._next_id()
        element = WhiteboardElement(
            element_id=eid,
            element_type="stroke",
            points=points,
            color=color,
            width=width
        )
        key = self._element_key(eid)
        self.elements[key] = element
        return self._to_op("add", element)

    def delete_element(self, element_key: str) -> Optional[dict]:
        """删除元素（墓碑标记，不真正移除）"""
        if element_key not in self.elements:
            return None
        self.elements[element_key].deleted = True
        self.lamport_clock += 1
        return self._to_op("delete", self.elements[element_key])

    def merge_op(self, op: dict):
        """合并远程操作（CRDT 核心逻辑）"""
        eid = CRDTElementId(op["lamport_ts"], op["node_id"])
        key = self._element_key(eid)

        # 更新 Lamport 时钟：取 max + 1
        self.lamport_clock = max(self.lamport_clock, op["lamport_ts"]) + 1

        if op["op_type"] == "add":
            if key not in self.elements:
                element = WhiteboardElement(
                    element_id=eid,
                    element_type=op["element_type"],
                    points=op.get("points", []),
                    color=op.get("color", "#000000"),
                    width=op.get("width", 2.0),
                    text=op.get("text", ""),
                    shape=op.get("shape", ""),
                )
                self.elements[key] = element
            # 如果已存在，说明本地已处理过（幂等），跳过
        elif op["op_type"] == "delete":
            if key in self.elements:
                self.elements[key].deleted = True
            else:
                # 元素还未到达，先记录墓碑
                element = WhiteboardElement(
                    element_id=eid,
                    element_type=op.get("element_type", "stroke"),
                    deleted=True
                )
                self.elements[key] = element

        elif op["op_type"] == "append_points":
            # 增量追加笔画点（实时绘制中）
            if key in self.elements and not self.elements[key].deleted:
                self.elements[key].points.extend(op.get("new_points", []))
            else:
                # 元素还未到达，缓存等待
                self._cache_pending_op(key, op)

    def get_render_order(self) -> List[WhiteboardElement]:
        """获取渲染顺序：按 Lamport 时间戳排序，过滤已删除"""
        active = [e for e in self.elements.values() if not e.deleted]
        active.sort(key=lambda e: e.element_id)
        return active

    def _to_op(self, op_type: str, element: WhiteboardElement) -> dict:
        """序列化为操作日志"""
        return {
            "op_type": op_type,
            "lamport_ts": element.element_id.lamport_ts,
            "node_id": element.element_id.node_id,
            "element_type": element.element_type,
            "points": element.points,
            "color": element.color,
            "width": element.width,
            "text": element.text,
            "shape": element.shape,
        }

    def _cache_pending_op(self, key: str, op: dict):
        """缓存等待元素到达的操作"""
        if not hasattr(self, '_pending'):
            self._pending: Dict[str, List[dict]] = {}
        if key not in self._pending:
            self._pending[key] = []
        self._pending[key].append(op)
```

**CRDT 白板服务端中继：**

```python
class WhiteboardCRDTRelay:
    """服务端中继：广播白板 CRDT 操作，不做冲突解决（CRDT 保证无冲突）"""

    def __init__(self, redis):
        self.redis = redis

    async def on_local_op(self, classroom_id: str, student_id: str, op: dict):
        """收到本地操作 → 广播给同课堂其他人"""
        op["source"] = student_id
        op["classroom_id"] = classroom_id
        op["timestamp"] = now_ms()

        # 1. 持久化到 Redis Stream（断线补偿 + 回放）
        await self.redis.xadd(
            f"wb_crdt:{classroom_id}",
            {"op": json.dumps(op)},
            maxlen=200000
        )

        # 2. Pub/Sub 广播（实时）
        await self.redis.publish(
            f"wb_crdt:{classroom_id}",
            json.dumps(op)
        )

    async def on_reconnect(self, classroom_id: str, student_id: str, last_lamport: int):
        """断线重连 → 补发缺失操作"""
        ops = await self.redis.xrange(
            f"wb_crdt:{classroom_id}", min=f"-", max="+"
        )
        missed = []
        for _, entry in ops:
            op = json.loads(entry[b"op"])
            if op["lamport_ts"] > last_lamport and op["source"] != student_id:
                missed.append(op)
        return missed
```

**为什么 CRDT 而不是 OT（Operational Transform）？**

| 维度 | OT | CRDT |
|------|----|----|
| 服务端要求 | 需要中心化服务器做转换计算 | 无需服务端参与冲突解决 |
| 延迟 | 依赖服务端 round-trip 确认 | 本地立即生效，异步合并 |
| 实现复杂度 | 极高（需要定义各种 transform 函数） | 中等（定义好数据结构即可） |
| 网络要求 | 需要稳定连接 | 天然支持断线重连后合并 |
| 适用场景 | Google Docs 等文本协作 | 白板/画布等图形协作 |

教育白板场景选择 CRDT 的核心理由：(1) 图形元素之间天然独立（不像文本有插入位置冲突），(2) 弱网学生多，需要离线操作能力，(3) 服务端无需额外计算，降低延迟。

### 白板 CRDT 完整冲突解决与合并引擎

上一节介绍了 CRDT 的基本数据结构，这里深入实现完整的冲突解决引擎，包括：并发编辑冲突检测、笔画点增量合并、撤销/重做的 CRDT 化、以及服务端快照压缩策略。

**冲突场景分析：** 当两个老师（或老师与被授权学生）同时在白板同一区域书写时，可能出现以下冲突：

| 冲突类型 | 示例 | 解决策略 |
|---------|------|---------|
| 同时添加元素 | 两人同时在不同位置画线 | 无冲突，Lamport 排序确定 z-order |
| 同时删除同一元素 | 一人删除另一人正在修改的笔画 | 墓碑优先，修改操作被忽略 |
| 同时修改同一元素 | 两人同时修改同一条线的颜色 | Last-Writer-Wins（LWW），Lamport 时间戳大的胜出 |
| 增量追加冲突 | 两人向同一条笔画追加不同的点 | 合并追加，所有点都保留（按 Lamport 排序） |

```python
from enum import Enum
from typing import Dict, List, Optional, Tuple, Set
import time

class OpType(Enum):
    ADD = "add"
    DELETE = "delete"
    MODIFY = "modify"
    APPEND_POINTS = "append_points"
    UNDO = "undo"
    REDO = "redo"
    CLEAR_ALL = "clear_all"

@dataclass
class CRDTVectorClock:
    """向量时钟：追踪各节点的已知状态，用于并发检测"""
    clock: Dict[str, int] = field(default_factory=dict)  # node_id → lamport_ts

    def increment(self, node_id: str):
        self.clock[node_id] = self.clock.get(node_id, 0) + 1

    def merge(self, other: 'CRDTVectorClock'):
        """合并另一个向量时钟（取各节点最大值）"""
        for node_id, ts in other.clock.items():
            self.clock[node_id] = max(self.clock.get(node_id, 0), ts)

    def happens_before(self, other: 'CRDTVectorClock') -> bool:
        """判断本时钟是否在 other 之前发生"""
        all_leq = all(self.clock.get(nid, 0) <= ts for nid, ts in other.clock.items())
        any_lt = any(self.clock.get(nid, 0) < ts for nid, ts in other.clock.items())
        return all_leq and any_lt

    def is_concurrent(self, other: 'CRDTVectorClock') -> bool:
        """判断两个时钟是否并发（既不在前也不在后）"""
        return not self.happens_before(other) and not other.happens_before(self)


@dataclass
class WhiteboardOp:
    """白板操作的完整描述"""
    op_id: str              # 唯一操作 ID
    op_type: OpType         # 操作类型
    target_key: str         # 目标元素 key（"ts:node_id"）
    lamport_ts: int         # Lamport 时间戳
    node_id: str            # 发起节点
    vector_clock: CRDTVectorClock  # 操作时的向量时钟
    payload: dict           # 操作数据
    timestamp_ms: int       # 物理时间戳


class WhiteboardCRDTEngine:
    """完整的 CRDT 白板引擎：冲突检测、合并、撤销/重做"""

    def __init__(self, node_id: str):
        self.node_id = node_id
        self.lamport_clock = 0
        self.vector_clock = CRDTVectorClock()
        self.elements: Dict[str, WhiteboardElement] = {}
        self.op_log: List[WhiteboardOp] = []      # 全部操作日志（含撤销）
        self.undo_stack: List[str] = []            # 可撤销的操作 ID 栈
        self.redo_stack: List[str] = []            # 可重做的操作 ID 栈
        self.tombstones: Set[str] = set()          # 已删除元素的 key
        self.pending_ops: Dict[str, List[WhiteboardOp]] = {}  # 等待前置元素的缓冲

    def _next_lamport(self) -> int:
        self.lamport_clock += 1
        self.vector_clock.increment(self.node_id)
        return self.lamport_clock

    def _update_lamport(self, remote_ts: int):
        """收到远程操作后更新本地 Lamport 时钟"""
        self.lamport_clock = max(self.lamport_clock, remote_ts) + 1
        self.vector_clock.clock[self.node_id] = self.lamport_clock

    def add_stroke(self, points: list, color="#000000", width=2.0) -> WhiteboardOp:
        """本地添加笔画"""
        lamport = self._next_lamport()
        eid = CRDTElementId(lamport, self.node_id)
        element = WhiteboardElement(
            element_id=eid, element_type="stroke",
            points=points, color=color, width=width, version=1
        )
        key = f"{eid.lamport_ts}:{eid.node_id}"
        self.elements[key] = element

        op = WhiteboardOp(
            op_id=f"op_{lamport}_{self.node_id}",
            op_type=OpType.ADD, target_key=key,
            lamport_ts=lamport, node_id=self.node_id,
            vector_clock=CRDTVectorClock(clock=dict(self.vector_clock.clock)),
            payload={"points": points, "color": color, "width": width, "element_type": "stroke"},
            timestamp_ms=int(time.time() * 1000)
        )
        self.op_log.append(op)
        self.undo_stack.append(op.op_id)
        self.redo_stack.clear()  # 新操作清空重做栈
        return op

    def append_points(self, element_key: str, new_points: list) -> Optional[WhiteboardOp]:
        """向已有笔画追加点（实时绘制中）"""
        if element_key not in self.elements or self.elements[element_key].deleted:
            return None
        self.elements[element_key].points.extend(new_points)
        self.elements[element_key].version += 1

        lamport = self._next_lamport()
        op = WhiteboardOp(
            op_id=f"op_{lamport}_{self.node_id}",
            op_type=OpType.APPEND_POINTS, target_key=element_key,
            lamport_ts=lamport, node_id=self.node_id,
            vector_clock=CRDTVectorClock(clock=dict(self.vector_clock.clock)),
            payload={"new_points": new_points, "version": self.elements[element_key].version},
            timestamp_ms=int(time.time() * 1000)
        )
        self.op_log.append(op)
        return op

    def modify_element(self, element_key: str, modifications: dict) -> Optional[WhiteboardOp]:
        """修改元素属性（颜色/线宽等）——Last-Writer-Wins 策略"""
        if element_key not in self.elements or self.elements[element_key].deleted:
            return None
        element = self.elements[element_key]
        for k, v in modifications.items():
            if hasattr(element, k):
                setattr(element, k, v)
        element.version += 1

        lamport = self._next_lamport()
        op = WhiteboardOp(
            op_id=f"op_{lamport}_{self.node_id}",
            op_type=OpType.MODIFY, target_key=element_key,
            lamport_ts=lamport, node_id=self.node_id,
            vector_clock=CRDTVectorClock(clock=dict(self.vector_clock.clock)),
            payload={**modifications, "version": element.version},
            timestamp_ms=int(time.time() * 1000)
        )
        self.op_log.append(op)
        self.undo_stack.append(op.op_id)
        self.redo_stack.clear()
        return op

    def delete_element(self, element_key: str) -> Optional[WhiteboardOp]:
        """删除元素（墓碑标记）"""
        if element_key not in self.elements or self.elements[element_key].deleted:
            return None
        self.elements[element_key].deleted = True
        self.tombstones.add(element_key)

        lamport = self._next_lamport()
        op = WhiteboardOp(
            op_id=f"op_{lamport}_{self.node_id}",
            op_type=OpType.DELETE, target_key=element_key,
            lamport_ts=lamport, node_id=self.node_id,
            vector_clock=CRDTVectorClock(clock=dict(self.vector_clock.clock)),
            payload={"element_type": self.elements[element_key].element_type},
            timestamp_ms=int(time.time() * 1000)
        )
        self.op_log.append(op)
        self.undo_stack.append(op.op_id)
        self.redo_stack.clear()
        return op

    def undo(self) -> Optional[WhiteboardOp]:
        """撤销最近一个操作"""
        if not self.undo_stack:
            return None
        op_id = self.undo_stack.pop()
        original_op = next((o for o in self.op_log if o.op_id == op_id), None)
        if not original_op:
            return None

        # 生成反向操作
        lamport = self._next_lamport()
        if original_op.op_type == OpType.ADD:
            # 撤销添加 = 删除
            self.elements[original_op.target_key].deleted = True
            self.tombstones.add(original_op.target_key)
            reverse_type = OpType.DELETE
            payload = {"undone_op": op_id}
        elif original_op.op_type == OpType.DELETE:
            # 撤销删除 = 恢复
            self.elements[original_op.target_key].deleted = False
            self.tombstones.discard(original_op.target_key)
            reverse_type = OpType.UNDO
            payload = {"undone_op": op_id}
        elif original_op.op_type == OpType.MODIFY:
            # 撤销修改 = 恢复原始值（需要保存旧值）
            payload = {"undone_op": op_id, "restored_version": original_op.payload.get("version", 1) - 1}
            reverse_type = OpType.UNDO
        else:
            return None

        op = WhiteboardOp(
            op_id=f"op_{lamport}_{self.node_id}",
            op_type=reverse_type, target_key=original_op.target_key,
            lamport_ts=lamport, node_id=self.node_id,
            vector_clock=CRDTVectorClock(clock=dict(self.vector_clock.clock)),
            payload=payload,
            timestamp_ms=int(time.time() * 1000)
        )
        self.op_log.append(op)
        self.redo_stack.append(op.op_id)
        return op

    def merge_remote_op(self, op: WhiteboardOp) -> List[WhiteboardOp]:
        """
        合并远程操作 —— CRDT 核心冲突解决逻辑
        返回本节点新生成的操作（如有）
        """
        # 幂等检查：已处理过则跳过
        if any(o.op_id == op.op_id for o in self.op_log):
            return []

        self._update_lamport(op.lamport_ts)
        self.vector_clock.merge(op.vector_clock)
        new_ops = []

        if op.op_type == OpType.ADD:
            if op.target_key not in self.elements:
                eid = CRDTElementId(op.lamport_ts, op.node_id)
                element = WhiteboardElement(
                    element_id=eid,
                    element_type=op.payload.get("element_type", "stroke"),
                    points=op.payload.get("points", []),
                    color=op.payload.get("color", "#000000"),
                    width=op.payload.get("width", 2.0),
                    text=op.payload.get("text", ""),
                    shape=op.payload.get("shape", ""),
                )
                # 检查是否有墓碑（远程删除先到达）
                if op.target_key in self.tombstones:
                    element.deleted = True
                self.elements[op.target_key] = element
            # 处理缓冲中的追加操作
            if op.target_key in self.pending_ops:
                for pending_op in self.pending_ops.pop(op.target_key):
                    self._apply_append_points(pending_op)
        elif op.op_type == OpType.DELETE:
            if op.target_key in self.elements:
                # 冲突检测：如果本地正在修改该元素，删除优先（墓碑胜出）
                self.elements[op.target_key].deleted = True
                self.tombstones.add(op.target_key)
            else:
                # 元素还未到达，先记录墓碑
                self.tombstones.add(op.target_key)

        elif op.op_type == OpType.MODIFY:
            if op.target_key in self.elements and not self.elements[op.target_key].deleted:
                local_version = self.elements[op.target_key].version
                remote_version = op.payload.get("version", 0)
                # LWW：版本号大的胜出；版本号相同时 Lamport 时间戳大的胜出
                if remote_version >= local_version:
                    for k, v in op.payload.items():
                        if k not in ("version",) and hasattr(self.elements[op.target_key], k):
                            setattr(self.elements[op.target_key], k, v)
                    self.elements[op.target_key].version = max(local_version, remote_version)

        elif op.op_type == OpType.APPEND_POINTS:
            if op.target_key in self.elements and not self.elements[op.target_key].deleted:
                self._apply_append_points(op)
            else:
                # 元素还未到达，缓冲等待
                self.pending_ops.setdefault(op.target_key, []).append(op)

        elif op.op_type == OpType.UNDO:
            # 远程撤销：根据被撤销的操作类型反向执行
            undone_op_id = op.payload.get("undone_op")
            if op.target_key in self.elements:
                # 简化处理：恢复删除或删除添加
                original_op = next((o for o in self.op_log if o.op_id == undone_op_id), None)
                if original_op and original_op.op_type == OpType.DELETE:
                    self.elements[op.target_key].deleted = False
                    self.tombstones.discard(op.target_key)
                elif original_op and original_op.op_type == OpType.ADD:
                    self.elements[op.target_key].deleted = True
                    self.tombstones.add(op.target_key)

        self.op_log.append(op)
        return new_ops

    def _apply_append_points(self, op: WhiteboardOp):
        """应用追加笔画点操作"""
        element = self.elements.get(op.target_key)
        if element and not element.deleted:
            # 合并追加：所有点都保留，按到达顺序排列
            # 避免重复点：用 (x, y, timestamp) 去重
            existing_set = set((p["x"], p["y"]) for p in element.points)
            for pt in op.payload.get("new_points", []):
                if (pt["x"], pt["y"]) not in existing_set:
                    element.points.append(pt)
                    existing_set.add((pt["x"], pt["y"]))
            element.version = max(element.version, op.payload.get("version", element.version))

    def get_state_hash(self) -> str:
        """计算当前状态的哈希（用于快速判断各节点状态是否一致）"""
        import hashlib
        active_keys = sorted(k for k, v in self.elements.items() if not v.deleted)
        hash_input = "|".join(
            f"{k}:{len(self.elements[k].points)}:{self.elements[k].version}"
            for k in active_keys
        )
        return hashlib.md5(hash_input.encode()).hexdigest()

    def serialize_state(self) -> dict:
        """序列化完整状态（用于快照）"""
        return {
            "node_id": self.node_id,
            "lamport_clock": self.lamport_clock,
            "vector_clock": dict(self.vector_clock.clock),
            "elements": {
                k: {
                    "element_type": v.element_type,
                    "points": v.points,
                    "color": v.color,
                    "width": v.width,
                    "text": v.text,
                    "shape": v.shape,
                    "deleted": v.deleted,
                    "version": v.version,
                }
                for k, v in self.elements.items()
            },
            "tombstones": list(self.tombstones),
            "state_hash": self.get_state_hash()
        }

    @classmethod
    def from_snapshot(cls, snapshot: dict) -> 'WhiteboardCRDTEngine':
        """从快照恢复状态"""
        engine = cls(node_id=snapshot["node_id"])
        engine.lamport_clock = snapshot["lamport_clock"]
        engine.vector_clock = CRDTVectorClock(clock=snapshot["vector_clock"])
        for k, v in snapshot["elements"].items():
            parts = k.split(":")
            eid = CRDTElementId(int(parts[0]), parts[1])
            engine.elements[k] = WhiteboardElement(
                element_id=eid,
                element_type=v["element_type"],
                points=v["points"],
                color=v["color"],
                width=v["width"],
                text=v.get("text", ""),
                shape=v.get("shape", ""),
                deleted=v["deleted"],
                version=v["version"]
            )
        engine.tombstones = set(snapshot.get("tombstones", []))
        return engine
```

**CRDT 白板快照压缩策略：** 长时间课堂（90 分钟）可能累积数万个操作日志，断线重连时全量回放代价大。策略：每 5 分钟生成一次快照，重连时先加载最近快照，再回放之后增量。

```python
class WhiteboardSnapshotManager:
    """白板快照管理：定期压缩 CRDT 操作日志"""

    def __init__(self, redis, s3_client, snapshot_interval_ms=300000):
        self.redis = redis
        self.s3 = s3_client
        self.snapshot_interval_ms = snapshot_interval_ms  # 默认 5 分钟

    async def maybe_create_snapshot(self, classroom_id: str, engine: WhiteboardCRDTEngine):
        """检查是否需要创建快照"""
        last_snapshot_ts = await self.redis.get(f"wb_snapshot_ts:{classroom_id}")
        current_ts = int(time.time() * 1000)

        if last_snapshot_ts is None or current_ts - int(last_snapshot_ts) > self.snapshot_interval_ms:
            await self._create_snapshot(classroom_id, engine)

    async def _create_snapshot(self, classroom_id: str, engine: WhiteboardCRDTEngine):
        """创建快照并存储到 S3"""
        snapshot = engine.serialize_state()
        snapshot_key = f"wb_snapshots/{classroom_id}/{engine.lamport_clock}.json"

        # 存储到 S3
        await self.s3.put(snapshot_key, json.dumps(snapshot))

        # 更新快照元信息
        pipe = self.redis.pipeline()
        pipe.set(f"wb_snapshot_ts:{classroom_id}", int(time.time() * 1000))
        pipe.set(f"wb_snapshot_key:{classroom_id}", snapshot_key)
        pipe.set(f"wb_snapshot_lamport:{classroom_id}", engine.lamport_clock)
        await pipe.execute()

    async def load_latest_snapshot(self, classroom_id: str) -> Tuple[Optional[WhiteboardCRDTEngine], int]:
        """加载最近快照，返回（引擎实例, 快照的 Lamport 时间戳）"""
        snapshot_key = await self.redis.get(f"wb_snapshot_key:{classroom_id}")
        if not snapshot_key:
            return None, 0

        data = await self.s3.get(snapshot_key)
        snapshot = json.loads(data)
        engine = WhiteboardCRDTEngine.from_snapshot(snapshot)
        lamport_at_snapshot = int(await self.redis.get(f"wb_snapshot_lamport:{classroom_id}") or 0)
        return engine, lamport_at_snapshot

    async def restore_with_incremental(self, classroom_id: str) -> WhiteboardCRDTEngine:
        """恢复白板状态：快照 + 增量回放"""
        engine, lamport_at_snapshot = await self.load_latest_snapshot(classroom_id)

        if engine is None:
            return WhiteboardCRDTEngine(node_id=f"server-{classroom_id}")

        # 从 Redis Stream 读取快照之后的增量操作
        ops = await self.redis.xrange(
            f"wb_crdt:{classroom_id}",
            min=f"{lamport_at_snapshot + 1}-0", max="+"
        )
        for _, entry in ops:
            op_data = json.loads(entry[b"op"])
            op = WhiteboardOp(
                op_id=op_data["op_id"],
                op_type=OpType(op_data["op_type"]),
                target_key=op_data["target_key"],
                lamport_ts=op_data["lamport_ts"],
                node_id=op_data["node_id"],
                vector_clock=CRDTVectorClock(clock=op_data["vector_clock"]),
                payload=op_data["payload"],
                timestamp_ms=op_data["timestamp_ms"]
            )
            engine.merge_remote_op(op)

        return engine
```

### 多流录制架构：教师摄像头 + 屏幕共享 + 白板 + 学生音频

在线教育课堂需要同时录制多个数据流，课后合成完整回放。与娱乐直播的"单路推流+录流"不同，教育场景至少包含 4 路独立数据流，各有不同的编码参数和可靠性要求。

**多流架构总览：**

```
                    ┌──────────────┐
                    │   老师端     │
                    └──┬───┬───┬───┘
                       │   │   │
            ┌──────────┘   │   └──────────┐
            ▼              ▼              ▼
     ┌──────────┐  ┌──────────┐  ┌──────────┐
     │摄像头流  │  │屏幕共享流│  │白板事件流│
     │H.264    │  │H.264    │  │JSON      │
     │720p/2Mbps│  │1080p/4Mbps│ │~2KB/s   │
     └─────┬────┘  └─────┬────┘  └─────┬────┘
           │             │             │
           └──────┬──────┘             │
                  ▼                    │
          ┌──────────────┐            │
          │ 录制服务集群  │◄───────────┘
          │ (FFmpeg)     │◄──── 学生音频混流
          └──────┬───────┘
                 │
         ┌───────┼───────┐
         ▼       ▼       ▼
    ┌────────┐┌────────┐┌────────┐
    │原始分轨││HLS直播 ││合流回放│
    │存储    ││切片    ││MP4    │
    │(S3)   ││(CDN)  ││(S3)   │
    └────────┘└────────┘└────────┘
```

**各流编码参数：**

| 流类型 | 编码 | 分辨率 | 帧率 | 码率 | 关键帧间隔 | 容器 |
|-------|------|-------|------|------|-----------|------|
| 老师摄像头 | H.264 | 1280x720 | 30fps | 2Mbps | 2秒 | FLV → MP4 |
| 屏幕共享 | H.264 (screen content) | 1920x1080 | 15fps | 4Mbps | 2秒 | FLV → MP4 |
| 白板事件 | JSON | N/A | N/A | ~2KB/s | N/A | JSON Lines |
| 学生音频 | AAC | N/A | 48kHz | 128kbps | N/A | AAC |
| 合流输出 | H.264 + AAC | 1920x1080 | 30fps | 6Mbps | 2秒 | MP4 / HLS |

**录制服务核心实现：**

```python
import subprocess
import asyncio
import os
from dataclasses import dataclass
from typing import Optional

@dataclass
class StreamConfig:
    """单个流的录制配置"""
    stream_type: str       # "camera" / "screen" / "audio"
    input_url: str         # RTMP/SRT 输入地址
    output_prefix: str     # S3 存储前缀
    codec: str             # "libx264" / "copy" / "aac"
    resolution: str        # "1280x720" / "1920x1080"
    bitrate: str           # "2M" / "4M" / "128k"
    framerate: int         # 30 / 15
    keyframe_interval: int # 秒
    format: str            # "flv" / "mp4" / "hls"

class MultiStreamRecorder:
    """多流录制器：同时录制多个流，独立分轨存储"""

    def __init__(self, session_id: str, s3_bucket: str):
        self.session_id = session_id
        self.s3_bucket = s3_bucket
        self.processes: Dict[str, subprocess.Popen] = {}
        self.hls_processes: Dict[str, subprocess.Popen] = {}
        self.recording_status: Dict[str, dict] = {}
        self.start_time: Optional[int] = None
        self._health_check_interval = 5  # 每 5 秒检查一次录制健康状态

    def _build_ffmpeg_cmd(self, config: StreamConfig, output_path: str) -> list:
        """构建 FFmpeg 录制命令"""
        cmd = ["ffmpeg", "-y"]

        # 输入配置
        if config.input_url.startswith("rtmp://") or config.input_url.startswith("srt://"):
            cmd.extend([
                "-i", config.input_url,
                "-timeout", "5000000",    # 5 秒超时
                "-reconnect", "1",
                "-reconnect_streamed", "1",
                "-reconnect_delay_max", "5",
            ])
        else:
            cmd.extend(["-i", config.input_url])

        # 视频编码
        if config.stream_type in ("camera", "screen"):
            if config.codec == "copy":
                cmd.extend(["-c:v", "copy"])
            else:
                cmd.extend([
                    "-c:v", "libx264",
                    "-preset", "veryfast",       # 低延迟预设
                    "-tune", "zerolatency" if config.stream_type == "camera" else "stillimage",
                    "-b:v", config.bitrate,
                    "-maxrate", f"{int(config.bitrate.rstrip('Mk')) * 12 // 10}M",  # 1.2 倍码率上限
                    "-bufsize", f"{int(config.bitrate.rstrip('Mk')) * 2}M",
                    "-s", config.resolution,
                    "-r", str(config.framerate),
                    "-g", str(config.framerate * config.keyframe_interval),  # 关键帧间隔
                    "-keyint_min", str(config.framerate * config.keyframe_interval),
                    "-pix_fmt", "yuv420p",
                ])

            # 音频编码（摄像头流可能包含老师音频）
            if config.stream_type == "camera":
                cmd.extend([
                    "-c:a", "aac",
                    "-b:a", "128k",
                    "-ar", "48000",
                    "-ac", "1",              # 单声道
                ])
        elif config.stream_type == "audio":
            cmd.extend([
                "-c:a", "aac",
                "-b:a", config.bitrate,
                "-ar", "48000",
                "-ac", "1",
            ])

        # 输出格式
        if config.format == "flv":
            cmd.extend(["-f", "flv", output_path])
        elif config.format == "mp4":
            cmd.extend([
                "-f", "mp4",
                "-movflags", "+faststart+frag_keyframe+empty_moov",
                output_path
            ])
        elif config.format == "hls":
            cmd.extend([
                "-f", "hls",
                "-hls_time", "4",            # 每个切片 4 秒
                "-hls_list_size", "6",       # 播放列表保留 6 个切片
                "-hls_flags", "delete_segments+append_list",
                "-hls_segment_filename", output_path.replace(".m3u8", "_%03d.ts"),
                output_path
            ])

        return cmd

    async def start_recording(self, stream_configs: Dict[str, StreamConfig]):
        """启动所有流的录制"""
        self.start_time = int(time.time() * 1000)

        for stream_type, config in stream_configs.items():
            timestamp = int(time.time())
            output_path = f"/tmp/recordings/{self.session_id}/{stream_type}_{timestamp}"

            if config.format == "hls":
                output_path += ".m3u8"
            elif config.format == "flv":
                output_path += ".flv"
            else:
                output_path += ".mp4"

            os.makedirs(os.path.dirname(output_path), exist_ok=True)
            cmd = self._build_ffmpeg_cmd(config, output_path)

            process = subprocess.Popen(
                cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE
            )
            self.processes[stream_type] = process

            self.recording_status[stream_type] = {
                "pid": process.pid,
                "output_path": output_path,
                "start_time": self.start_time,
                "bytes_written": 0,
                "last_keyframe_ts": 0,
                "error_count": 0,
                "status": "recording"
            }

        # 启动健康检查
        asyncio.create_task(self._health_check_loop())

    async def _health_check_loop(self):
        """定期检查各流录制状态"""
        while True:
            await asyncio.sleep(self._health_check_interval)
            for stream_type, process in list(self.processes.items()):
                return_code = process.poll()
                if return_code is not None:
                    # 进程异常退出
                    self.recording_status[stream_type]["status"] = "failed"
                    self.recording_status[stream_type]["error_count"] += 1
                    # 尝试重启录制
                    if self.recording_status[stream_type]["error_count"] < 3:
                        await self._restart_stream(stream_type)

    async def _restart_stream(self, stream_type: str):
        """重启某个流的录制"""
        status = self.recording_status[stream_type]
        config = self._get_stream_config(stream_type)
        if not config:
            return

        timestamp = int(time.time())
        output_path = f"/tmp/recordings/{self.session_id}/{stream_type}_restart_{timestamp}.mp4"
        os.makedirs(os.path.dirname(output_path), exist_ok=True)

        cmd = self._build_ffmpeg_cmd(config, output_path)
        process = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        self.processes[stream_type] = process
        status["pid"] = process.pid
        status["output_path"] = output_path
        status["status"] = "recording"

    async def stop_recording(self) -> dict:
        """停止所有流录制，返回录制结果"""
        results = {}
        for stream_type, process in self.processes.items():
            # 发送 SIGTERM 优雅停止（FFmpeg 会完成最后一个关键帧）
            process.terminate()
            try:
                process.wait(timeout=10)  # 等 10 秒
            except subprocess.TimeoutExpired:
                process.kill()  # 强制杀

            status = self.recording_status[stream_type]
            duration_ms = int(time.time() * 1000) - self.start_time

            # 计算文件大小
            file_size = 0
            if os.path.exists(status["output_path"]):
                file_size = os.path.getsize(status["output_path"])

            results[stream_type] = {
                "output_path": status["output_path"],
                "duration_ms": duration_ms,
                "file_size_bytes": file_size,
                "error_count": status["error_count"],
                "status": "completed" if status["error_count"] == 0 else "recovered"
            }

        return results
```

**课后合流：将多流合成为单个回放 MP4**

```python
class RecordingComposer:
    """课后合成：将摄像头 + 屏幕共享 + 白板 + 音频合成为完整回放"""

    def __init__(self, s3_client, ffmpeg_path="ffmpeg"):
        self.s3 = s3_client
        self.ffmpeg = ffmpeg_path

    async def compose_replay(self, session_id: str, recording_results: dict) -> str:
        """
        合成回放视频布局：
        ┌──────────────────────────────────────┐
        │ 屏幕共享 (1920x1080) - 全屏背景      │
        │ ┌──────────┐                         │
        │ │ 摄像头   │                         │
        │ │ 320x180  │                         │
        │ └──────────┘                         │
        │              白板叠加层（半透明）      │
        └──────────────────────────────────────┘
        """
        camera_path = recording_results["camera"]["output_path"]
        screen_path = recording_results["screen"]["output_path"]
        audio_path = recording_results.get("audio", {}).get("output_path", "")
        output_path = f"/tmp/recordings/{session_id}/replay_composed.mp4"

        # FFmpeg 滤镜：屏幕全屏 + 摄像头画中画
        filter_complex = (
            # 屏幕共享缩放到 1920x1080
            f"[1:v]scale=1920:1080:force_original_aspect_ratio=decrease,"
            f"pad=1920:1080:(ow-iw)/2:(oh-ih)/2[screen];"
            # 摄像头缩放到 320x180
            f"[0:v]scale=320:180[camera];"
            # 叠加摄像头到右上角
            f"[screen][camera]overlay=W-w-20:20:format=auto[composed]"
        )

        cmd = [
            self.ffmpeg, "-y",
            "-i", camera_path,        # 输入 0: 摄像头
            "-i", screen_path,        # 输入 1: 屏幕共享
        ]

        # 如果有独立的音频流
        if audio_path and os.path.exists(audio_path):
            cmd.extend(["-i", audio_path])  # 输入 2: 学生音频混流
            cmd.extend([
                "-filter_complex", filter_complex,
                "-map", "[composed]",       # 合成视频
                "-map", "0:a",              # 摄像头音频（老师）
                "-map", "2:a",              # 学生音频
            ])
        else:
            cmd.extend([
                "-filter_complex", filter_complex,
                "-map", "[composed]",
                "-map", "0:a",
            ])

        cmd.extend([
            "-c:v", "libx264",
            "-preset", "medium",        # 合成时可用较慢预设获得更好画质
            "-b:v", "6M",
            "-c:a", "aac",
            "-b:a", "128k",
            "-movflags", "+faststart",  # Web 播放优化
            output_path
        ])

        process = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        stdout, stderr = process.communicate(timeout=3600)  # 1 小时超时

        if process.returncode != 0:
            raise RuntimeError(f"FFmpeg 合成失败: {stderr.decode()[-500:]}")

        # 上传到 S3
        s3_key = f"replays/{session_id}/replay.mp4"
        await self.s3.upload_file(output_path, self.s3_bucket, s3_key)

        return s3_key

    async def compose_hls_segments(self, session_id: str, recording_results: dict) -> str:
        """
        生成 HLS 切片用于 CDN 分发回放
        相比 MP4 的优势：无需下载完整文件，支持码率自适应
        """
        screen_path = recording_results["screen"]["output_path"]
        camera_path = recording_results["camera"]["output_path"]
        hls_dir = f"/tmp/recordings/{session_id}/hls"
        os.makedirs(hls_dir, exist_ok=True)

        # 生成多码率 HLS（自适应码率）
        # 1080p: 6Mbps, 720p: 3Mbps, 480p: 1.5Mbps
        cmd = [
            self.ffmpeg, "-y",
            "-i", screen_path,
            "-i", camera_path,
            "-filter_complex",
            f"[0:v]scale=1920:1080[screen];"
            f"[1:v]scale=320:180[camera];"
            f"[screen][camera]overlay=W-w-20:20[composed]",
            "-map", "[composed]",
            "-map", "0:a",
            # 三个码率变体
            "-c:v:0", "libx264", "-b:v:0", "6000k", "-s:0", "1920x1080", "-maxrate:v:0", "7200k",
            "-c:v:1", "libx264", "-b:v:1", "3000k", "-s:1", "1280x720",  "-maxrate:v:1", "3600k",
            "-c:v:2", "libx264", "-b:v:2", "1500k", "-s:2", "854x480",   "-maxrate:v:2", "1800k",
            "-c:a:0", "aac", "-b:a:0", "128k",
            "-c:a:1", "aac", "-b:a:1", "128k",
            "-c:a:2", "aac", "-b:a:2", "96k",
            "-var_stream_map", "v:0,a:0 v:1,a:1 v:2,a:2",
            "-master_pl_name", "master.m3u8",
            "-f", "hls",
            "-hls_time", "6",
            "-hls_list_size", "0",
            "-hls_segment_filename", f"{hls_dir}/stream_%v_%03d.ts",
            f"{hls_dir}/stream_%v.m3u8"
        ]

        process = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        process.communicate(timeout=3600)

        # 上传整个 HLS 目录到 S3
        s3_prefix = f"replays/{session_id}/hls/"
        for filename in os.listdir(hls_dir):
            filepath = os.path.join(hls_dir, filename)
            await self.s3.upload_file(filepath, self.s3_bucket, f"{s3_prefix}{filename}")

        return f"{s3_prefix}master.m3u8"
```

**录制存储成本分析：**

| 维度 | 摄像头流 | 屏幕共享流 | 白板事件 | 合流回放 | 合计 |
|------|---------|----------|---------|---------|------|
| 码率 | 2Mbps | 4Mbps | ~16Kbps | 6Mbps | ~12Mbps |
| 每小时文件大小 | 0.9GB | 1.8GB | ~7MB | 2.7GB | ~5.4GB |
| 每小时 S3 标准存储 | $0.023 × 0.9 = $0.021 | $0.023 × 1.8 = $0.041 | ≈ $0 | $0.023 × 2.7 = $0.062 | $0.124/堂/小时 |
| S3 Glacier 归档（90天后） | $0.004 × 0.9 = $0.004 | $0.004 × 1.8 = $0.007 | ≈ $0 | $0.004 × 2.7 = $0.011 | $0.022/堂/小时 |
| 200 堂/天 × 365 天 | - | - | - | - | 年存储 $9,052 (标准) → $1,606 (Glacier) |

策略：90 天内用 S3 标准（频繁回放），90 天后自动生命周期转为 Glacier（降低 81% 成本）。

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

### 小班课 vs 大班课：架构分型与代码差异

在线教育的课堂规模差异极大：1 对 1 辅导（1 名学生）和大班课（1 万名学生）有着根本不同的交互模式。小班课强调**全员实时互动**（每个学生都能发言、上白板），大班课强调**高效扇出 + 采样互动**（只有被点名学生能发言）。混用同一套架构会导致小班课体验差（延迟高、互动弱）或大班课成本爆炸（全员互动的连接和计算开销）。

**核心差异对比：**

| 维度 | 小班课（1-20 人） | 中班课（20-200 人） | 大班课（200-10000 人） |
|------|-------------------|-------------------|----------------------|
| 互动模式 | 全员双向实时 | 有限双向 + 答题 | 采样双向 + 聚合答题 |
| 音频通道 | 多人同时开麦（SFU 转发） | 老师常开 + 1 学生被点名 | 老师单向 + 被点名学生 |
| 白板权限 | 全员可书写（CRDT 多写者） | 老师主写 + 授权 2-3 学生 | 老师独写（单写者） |
| 答题聚合 | 逐条推送（数据量小） | 2 秒窗口聚合 | 2 秒窗口聚合 + 百分比 |
| 延迟要求 | < 300ms（对话延迟） | < 500ms | < 1 秒 |
| 录制方式 | 多轨录制（每路独立） | 多轨 + 合流 | 合流为主 + 白板事件 |
| 单课堂成本 | 低（人数少） | 中等 | 高（扇出带宽大） |
| 网关策略 | 单网关（20 连接） | 单/双网关 | 200 个网关分片 |

**架构分型实现：**

```python
from enum import Enum
from abc import ABC, abstractmethod

class ClassroomScale(Enum):
    SMALL = "small"    # 1-20 人
    MEDIUM = "medium"  # 20-200 人
    LARGE = "large"    # 200-10000 人

class ClassroomStrategy(ABC):
    """课堂策略抽象基类：不同规模使用不同策略"""

    @abstractmethod
    def dispatch_teacher_action(self, classroom_id: str, action: dict):
        """分发老师操作"""
        pass

    @abstractmethod
    def handle_student_interaction(self, classroom_id: str, student_id: int, interaction: dict):
        """处理学生互动"""
        pass

    @abstractmethod
    def get_audio_config(self) -> dict:
        """音频配置"""
        pass

    @abstractmethod
    def get_whiteboard_config(self) -> dict:
        """白板配置"""
        pass

    @abstractmethod
    def get_recording_config(self) -> dict:
        """录制配置"""
        pass


class SmallClassStrategy(ClassroomStrategy):
    """小班课策略：全员实时互动，WebRTC SFU 音视频"""

    def dispatch_teacher_action(self, classroom_id: str, action: dict):
        """小班课：老师操作直接推给所有学生（人少，无需聚合）"""
        # 直接通过 WebSocket 推送，无需 Redis Pub/Sub
        # 20 个连接逐个推送耗时 < 2ms
        students = self.get_connected_students(classroom_id)
        for ws in students:
            ws.send(json.dumps(action))

    def handle_student_interaction(self, classroom_id: str, student_id: int, interaction: dict):
        """小班课：学生互动直接推给老师和其他学生（全员可见）"""
        # 举手、答题、发言请求都直接推给老师
        self.push_to_teacher(classroom_id, interaction)
        # 答题结果也推给全班（20 人以内可以直接推送）
        if interaction["type"] == "answer":
            self.broadcast_classroom(classroom_id, {
                "type": "student_answer",
                "student_name": self.get_student_name(student_id),
                "answer": interaction["answer"]
            })

    def get_audio_config(self) -> dict:
        return {
            "mode": "sfu_multicast",     # SFU 多路转发
            "max_speakers": 6,           # 最多 6 人同时开麦
            "codec": "opus",
            "bitrate": "64kbps",
            "echo_cancel": True,         # 小班课必须开回声消除
            "noise_suppress": True,
            "auto_gain": True
        }

    def get_whiteboard_config(self) -> dict:
        return {
            "mode": "crdt_multi_writer", # CRDT 多写者模式
            "max_writers": 20,           # 全员可写
            "sync_protocol": "full_mesh", # 全量网格同步
            "permission": "all",         # 所有学生可书写
        }

    def get_recording_config(self) -> dict:
        return {
            "mode": "multi_track",       # 每路独立录制
            "tracks": ["camera", "screen", "audio_per_student"],
            "composite": "post_class",   # 课后合流
            "format": "mp4"
        }

    def get_connected_students(self, classroom_id):
        """直接从本地 WebSocket 连接池获取"""
        return self.local_connections.get(classroom_id, [])


class LargeClassStrategy(ClassroomStrategy):
    """大班课策略：高效扇出 + 采样互动"""

    def dispatch_teacher_action(self, classroom_id: str, action: dict):
        """大班课：Redis Pub/Sub 扇出到 200 个网关"""
        event = {
            "event_id": str(uuid4()),
            "type": action["type"],
            "data": action["data"],
            "classroom_id": classroom_id,
            "timestamp": now_ms()
        }
        # 持久化到事件流
        self.event_store.append(classroom_id, event)
        # Redis Pub/Sub 广播到所有网关
        self.redis.publish(f"classroom:{classroom_id}", json.dumps(event))
        # 关键操作同时写 Redis Stream（断线补偿）
        if action["type"] in ("page_flip", "call_on"):
            self.redis.xadd(f"stream:{classroom_id}", event)

    def handle_student_interaction(self, classroom_id: str, student_id: int, interaction: dict):
        """大班课：答案聚合 + 采样展示"""
        if interaction["type"] == "answer":
            # 选择题：实时聚合，2 秒推一次统计
            self.aggregator.on_student_answer(
                classroom_id,
                interaction["question_id"],
                interaction["answer"],
                student_id
            )
        elif interaction["type"] == "subjective_answer":
            # 主观题：采样推给老师
            self.subjective_handler.on_subjective_answer(
                classroom_id,
                interaction["question_id"],
                interaction["answer"],
                student_id
            )
        elif interaction["type"] == "hand_raise":
            # 举手：只推给老师
            self.hand_raise_service.on_hand_raise(classroom_id, student_id)

    def get_audio_config(self) -> dict:
        return {
            "mode": "teacher_broadcast",  # 老师单向广播
            "max_speakers": 2,           # 老师 + 被点名学生
            "codec": "opus",
            "bitrate": "128kbps",        # 更高码率（广播质量）
            "echo_cancel": False,        # 广播模式不需要
            "noise_suppress": False,
            "auto_gain": False,
            "student_audio_mix": True     # 多学生音频服务端混流
        }

    def get_whiteboard_config(self) -> dict:
        return {
            "mode": "single_writer_seq",  # 单写者 + 序号模式
            "max_writers": 1,            # 仅老师可写
            "sync_protocol": "pub_sub",  # Pub/Sub 扇出
            "permission": "teacher_only",
        }

    def get_recording_config(self) -> dict:
        return {
            "mode": "composite_live",     # 实时合流录制
            "tracks": ["camera", "screen", "mixed_audio"],
            "composite": "realtime",      # 实时合流（节省存储）
            "format": "hls+mp4",
            "hls_segment_duration": 4,   # 4 秒切片
        }


class MediumClassStrategy(ClassroomStrategy):
    """中班课策略：有限双向互动"""

    def dispatch_teacher_action(self, classroom_id: str, action: dict):
        """中班课：Redis Pub/Sub 扇出（网关数量少，但也走 Pub/Sub）"""
        event = {
            "event_id": str(uuid4()),
            "type": action["type"],
            "data": action["data"],
            "classroom_id": classroom_id,
            "timestamp": now_ms()
        }
        self.event_store.append(classroom_id, event)
        self.redis.publish(f"classroom:{classroom_id}", json.dumps(event))

    def handle_student_interaction(self, classroom_id: str, student_id: int, interaction: dict):
        """中班课：答题聚合 + 被授权学生可发言"""
        if interaction["type"] == "answer":
            self.aggregator.on_student_answer(
                classroom_id,
                interaction["question_id"],
                interaction["answer"],
                student_id
            )
        elif interaction["type"] == "hand_raise":
            self.hand_raise_service.on_hand_raise(classroom_id, student_id)
        elif interaction["type"] == "speak_request":
            # 中班课：老师可授权 2-3 名学生同时发言
            if self._count_active_speakers(classroom_id) < 3:
                self._grant_speak_permission(classroom_id, student_id)

    def get_audio_config(self) -> dict:
        return {
            "mode": "sfu_limited",       # SFU 有限多路
            "max_speakers": 4,           # 老师 + 最多 3 名学生
            "codec": "opus",
            "bitrate": "64kbps",
            "echo_cancel": True,
            "noise_suppress": True,
        }

    def get_whiteboard_config(self) -> dict:
        return {
            "mode": "crdt_limited_writer",
            "max_writers": 4,            # 老师 + 3 名被授权学生
            "sync_protocol": "pub_sub",  # Pub/Sub + CRDT
            "permission": "teacher_grant",  # 老师授权
        }

    def get_recording_config(self) -> dict:
        return {
            "mode": "multi_track",
            "tracks": ["camera", "screen", "mixed_audio"],
            "composite": "post_class",
            "format": "mp4"
        }


class ClassroomStrategyFactory:
    """课堂策略工厂：根据人数自动选择策略"""

    @staticmethod
    def create(student_count: int) -> ClassroomStrategy:
        if student_count <= 20:
            return SmallClassStrategy()
        elif student_count <= 200:
            return MediumClassStrategy()
        else:
            return LargeClassStrategy()

    @staticmethod
    def from_class_type(class_type: int, expected_students: int) -> ClassroomStrategy:
        """根据课堂类型和预期人数创建策略"""
        if class_type == 3:  # 一对一
            return SmallClassStrategy()
        elif class_type == 2:  # 小班课
            return SmallClassStrategy() if expected_students <= 20 else MediumClassStrategy()
        else:  # 大班课
            return LargeClassStrategy() if expected_students > 200 else MediumClassStrategy()
```

**小班课音频架构（WebRTC SFU）：**

```
             ┌─────────────┐
             │  SFU 服务   │
             │ (mediasoup) │
             └──┬──┬──┬──┬─┘
                │  │  │  │
        ┌───────┘  │  │  └───────┐
        ▼          ▼  ▼          ▼
    ┌──────┐  ┌──────┐ ┌──────┐ ┌──────┐
    │老师  │  │学生1 │ │学生2 │ │学生3 │
    │发+收 │  │发+收 │ │发+收 │ │发+收 │
    └──────┘  └──────┘ └──────┘ └──────┘

    每人发送 1 路上行 → SFU 转发 N-1 路下行
    20 人课堂：每人 1 路上行 + 19 路下行 → SFU 转发 20×19=380 路
```

**大班课音频架构（老师广播 + 点名混流）：**

```
    ┌──────┐     RTMP      ┌──────────┐
    │老师  │──────────────▶│ 旁路直播  │──── CDN ──▶ 全体学生
    │发送  │     推流       │ (RTMP)   │
    └──────┘               └──────────┘

    ┌──────┐     WebSocket  ┌──────────┐
    │学生  │──────────────▶│ 音频混流  │──▶ 老师端
    │发言  │     PCM 数据   │ 服务     │   (只混被点名学生)
    └──────┘               └──────────┘

    10000 人课堂：1 路上行 + 1 路下行 + 被点名学生上行
    SFU 只需转发 1 路主音频 + 1 路学生音频
```

**成本对比（单堂 90 分钟课）：**

| 成本项 | 小班课（20 人） | 大班课（10000 人） | 差异原因 |
|-------|---------------|-------------------|---------|
| 网关服务器 | 0.01 台 | 2 台 | 连接数差异 500 倍 |
| SFU/混流服务器 | 1 台 | 2 台（旁路 + 混流） | 小班用 SFU，大班用旁路 |
| 出站带宽 | 20 × 2Mbps × 90min = 27GB | 10000 × 2Mbps × 90min = 13.5TB | 扇出差异 |
| CDN 费用 | $0.4 | $200 | 流量差异 |
| 录制存储 | 5 路独立录制 = 5GB | 合流 + 分轨 = 5.4GB | 多轨 vs 合流 |
| 白板带宽 | 20 × 2KB/s × 90min = 216MB | 10000 × 2KB/s × 90min = 108GB | 扇出差异 |
| **总成本** | **≈ $0.5** | **≈ $210** | **420 倍** |

### 学生参与度追踪：实时注意力检测 + 参与评分 + 教师仪表盘

教育直播不同于娱乐直播的一个关键特征是：老师需要实时了解学生的参与状态——谁在认真听、谁走神了、谁离开课堂了。这些数据既影响当堂教学决策，也用于课后学习分析。

**参与度数据采集维度：**

| 采集维度 | 数据源 | 采集频率 | 隐私等级 |
|---------|-------|---------|---------|
| 在线状态 | WebSocket 心跳 | 每 5 秒 | 低 |
| 视频观看状态 | 播放器事件（play/pause/seek） | 实时事件 | 低 |
| 页面焦点 | browser visibility API | 每 10 秒 | 中 |
| 互动频率 | 答题/举手/聊天 | 实时事件 | 低 |
| 鼠标/触控活跃度 | 事件采样 | 每 30 秒 | 中 |
| 摄像头分析（可选） | 人脸检测 | 每 10 秒 | 高 |

**注意力检测服务：**

```python
from collections import defaultdict
import time

class StudentEngagementTracker:
    """学生参与度追踪器：综合多维信号计算实时参与度分数"""

    # 参与度评分权重
    WEIGHTS = {
        "online": 0.20,        # 在线状态
        "video_playing": 0.15, # 视频是否在播放
        "page_focused": 0.15, # 页面是否在前台
        "interaction": 0.30,   # 互动频率（答题/举手/聊天）
        "mouse_activity": 0.10,# 鼠标活跃度
        "camera_attention": 0.10,  # 摄像头注意力（可选）
    }

    def __init__(self, redis):
        self.redis = redis
        self.student_signals = {}   # classroom_id → {student_id → signals}
        self.engagement_scores = {} # classroom_id → {student_id → score}
        self.interaction_counts = defaultdict(lambda: defaultdict(int))  # 互动计数

    async def update_signal(self, classroom_id: str, student_id: int, signal_type: str, value):
        """更新学生信号"""
        key = f"engagement:{classroom_id}:{student_id}"
        # 存入 Redis Hash，TTL = 课堂结束时间
        await self.redis.hset(key, signal_type, json.dumps({
            "value": value,
            "timestamp": int(time.time() * 1000)
        }))
        await self.redis.expire(key, 7200)  # 最长 2 小时

    async def record_interaction(self, classroom_id: str, student_id: int, interaction_type: str):
        """记录学生互动事件"""
        interaction_key = f"interactions:{classroom_id}:{student_id}"
        window_key = f"interaction_window:{classroom_id}"

        # 递增互动计数
        count = await self.redis.incr(interaction_key)
        await self.redis.expire(interaction_key, 7200)

        # 记录到滑动窗口（5 分钟内的互动密度）
        now_ms = int(time.time() * 1000)
        await self.redis.zadd(
            window_key,
            {f"{student_id}:{interaction_type}:{now_ms}": now_ms}
        )
        # 清理 5 分钟之前的数据
        await self.redis.zremrangebyscore(window_key, 0, now_ms - 300000)

    async def calculate_engagement_score(self, classroom_id: str, student_id: int) -> float:
        """计算学生实时参与度分数（0-100）"""
        key = f"engagement:{classroom_id}:{student_id}"
        signals = await self.redis.hgetall(key)

        if not signals:
            return 0.0

        scores = {}
        now = int(time.time() * 1000)

        # 1. 在线状态：最近 10 秒有心跳 = 满分
        if "heartbeat" in signals:
            hb = json.loads(signals["heartbeat"])
            scores["online"] = 100 if now - hb["timestamp"] < 10000 else 0
        else:
            scores["online"] = 0

        # 2. 视频播放状态
        if "video_state" in signals:
            vs = json.loads(signals["video_state"])
            scores["video_playing"] = 100 if vs["value"] == "playing" else 0
        else:
            scores["video_playing"] = 50  # 未知状态给中间分

        # 3. 页面焦点
        if "page_visible" in signals:
            pv = json.loads(signals["page_visible"])
            scores["page_focused"] = 100 if pv["value"] else 0
        else:
            scores["page_focused"] = 50

        # 4. 互动频率：5 分钟内的互动次数映射到分数
        interaction_count = await self._get_interaction_count(classroom_id, student_id)
        # 5 分钟内 5 次以上互动 = 满分，0 次 = 0 分
        scores["interaction"] = min(100, interaction_count * 20)

        # 5. 鼠标活跃度
        if "mouse_activity" in signals:
            ma = json.loads(signals["mouse_activity"])
            # 最近 30 秒内的鼠标移动/点击次数
            scores["mouse_activity"] = min(100, ma["value"] * 10)
        else:
            scores["mouse_activity"] = 30  # 默认中等

        # 6. 摄像头注意力（可选功能）
        if "camera_attention" in signals:
            ca = json.loads(signals["camera_attention"])
            scores["camera_attention"] = ca["value"]  # 0-100 AI 评分
        else:
            scores["camera_attention"] = 50  # 未启用给中间分（权重低，影响小）

        # 加权求和
        total = sum(scores.get(k, 0) * v for k, v in self.WEIGHTS.items())
        return round(total, 1)

    async def _get_interaction_count(self, classroom_id: str, student_id: int) -> int:
        """获取 5 分钟内互动次数"""
        window_key = f"interaction_window:{classroom_id}"
        now_ms = int(time.time() * 1000)
        # 统计该学生 5 分钟内的互动记录
        entries = await self.redis.zrangebyscore(window_key, now_ms - 300000, now_ms)
        count = sum(1 for e in entries if str(student_id) in e.decode())
        return count

    async def get_classroom_engagement_summary(self, classroom_id: str) -> dict:
        """获取课堂整体参与度汇总（推送给老师仪表盘）"""
        # 获取课堂所有学生的信号 key
        pattern = f"engagement:{classroom_id}:*"
        keys = await self.redis.keys(pattern)

        if not keys:
            return {"total_students": 0, "avg_score": 0, "distribution": {}}

        scores = []
        attention_levels = {"high": 0, "medium": 0, "low": 0, "offline": 0}
        disengaged_students = []

        for key in keys:
            student_id = int(key.decode().split(":")[-1])
            score = await self.calculate_engagement_score(classroom_id, student_id)
            scores.append(score)

            # 分级
            if score >= 70:
                attention_levels["high"] += 1
            elif score >= 40:
                attention_levels["medium"] += 1
            elif score >= 10:
                attention_levels["low"] += 1
                disengaged_students.append(student_id)
            else:
                attention_levels["offline"] += 1

        avg_score = sum(scores) / len(scores) if scores else 0

        return {
            "total_students": len(scores),
            "avg_score": round(avg_score, 1),
            "distribution": attention_levels,
            "disengaged_students": disengaged_students[:10],  # 最多展示 10 个走神学生
            "online_rate": round(
                (attention_levels["high"] + attention_levels["medium"] + attention_levels["low"])
                / max(len(scores), 1) * 100, 1
            ),
            "timestamp": int(time.time() * 1000)
        }
```

**教师仪表盘：客户端实时展示**

```javascript
class TeacherDashboard {
    constructor(classroomId, ws) {
        this.classroomId = classroomId;
        this.ws = ws;
        this.engagementHistory = [];    // 参与度时间序列
        this.answerHeatmap = {};        // 答题热力图
        this.studentList = new Map();   // 学生状态列表
    }

    onEngagementUpdate(summary) {
        // 每 5 秒收到一次参与度汇总
        this.engagementHistory.push({
            timestamp: summary.timestamp,
            avgScore: summary.avg_score,
            onlineRate: summary.online_rate
        });

        this.renderEngagementChart(summary);
        this.renderAttentionDistribution(summary.distribution);

        // 走神学生告警
        if (summary.disengaged_students.length > 0) {
            this.showAttentionAlert(summary.disengaged_students);
        }

        // 在线率低于 80% 告警
        if (summary.onlineRate < 80) {
            this.showOnlineRateWarning(summary.onlineRate);
        }
    }

    onAnswerAggregate(data) {
        // 实时答题分布图
        this.answerHeatmap[data.question_id] = data;
        this.renderAnswerChart(data.question_id, data.counts, data.percentage);
    }

    renderEngagementChart(summary) {
        // 折线图：平均参与度 + 在线率随时间变化
        const chart = this.engagementChart;
        chart.data.labels.push(new Date(summary.timestamp).toLocaleTimeString());
        chart.data.datasets[0].data.push(summary.avgScore);
        chart.data.datasets[1].data.push(summary.onlineRate);
        chart.update('none');  // 无动画，即时刷新
    }

    renderAttentionDistribution(distribution) {
        // 饼图：高注意力 / 中等 / 低注意力 / 离线
        const labels = ['专注', '一般', '走神', '离线'];
        const colors = ['#4CAF50', '#FF9800', '#F44336', '#9E9E9E'];
        this.attentionPie.data = {
            labels,
            datasets: [{ data: [distribution.high, distribution.medium, distribution.low, distribution.offline], backgroundColor: colors }]
        };
        this.attentionPie.update('none');
    }

    showAttentionAlert(studentIds) {
        // 老师端弹出走神学生提醒
        const names = studentIds.map(id => this.studentList.get(id)?.name || `学生${id}`);
        this.alertContainer.innerHTML = `
            <div class="attention-alert">
                <span>⚠️ ${names.length} 名学生注意力不集中：${names.join('、')}</span>
                <button onclick="teacherDashboard.callOnStudent(${studentIds[0]})">
                    点名提醒
                </button>
            </div>
        `;
    }

    callOnStudent(studentId) {
        // 老师点击"点名提醒" → 点名走神学生
        this.ws.send(JSON.stringify({
            type: "call_on",
            student_id: studentId,
            classroom_id: this.classroomId
        }));
    }
}
```

**参与度评分时间线（课后分析）：**

```python
class EngagementTimelineAnalyzer:
    """课后参与度时间线分析：生成每分钟参与度报告"""

    def analyze_session(self, session_id: str) -> dict:
        """分析一堂课的参与度变化"""
        # 从事件流加载课堂事件
        events = self.event_store.get_all(session_id)

        # 按分钟分桶
        duration_min = (events[-1].timestamp - events[0].timestamp) / 60000
        timeline = []

        for minute in range(int(duration_min)):
            minute_start = events[0].timestamp + minute * 60000
            minute_end = minute_start + 60000

            # 该分钟内的事件
            minute_events = [e for e in events if minute_start <= e.timestamp < minute_end]
            event_types = Counter(e.type for e in minute_events)

            timeline.append({
                "minute": minute,
                "total_events": len(minute_events),
                "answer_count": event_types.get("student_answer", 0),
                "hand_raise_count": event_types.get("hand_raise", 0),
                "engagement_level": self._classify_engagement(event_types)
            })

        return {
            "session_id": session_id,
            "duration_min": int(duration_min),
            "timeline": timeline,
            "peak_engagement_minute": max(range(len(timeline)), key=lambda i: timeline[i]["total_events"]),
            "low_engagement_periods": [
                t["minute"] for t in timeline if t["engagement_level"] == "low"
            ]
        }

    def _classify_engagement(self, event_types: Counter) -> str:
        """根据互动事件分类参与度等级"""
        total = sum(event_types.values())
        if total >= 50:
            return "high"
        elif total >= 10:
            return "medium"
        else:
            return "low"
```

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

## 详细性能分析

### WebSocket 连接容量

**单网关服务器性能基准（16GB 内存 / 8 核 CPU / 10Gbps 网卡）：**

| 指标 | 测量值 | 瓶颈来源 |
|------|-------|---------|
| 最大 WebSocket 连接数 | 10 万 | 内存（每连接 ~50KB：缓冲区 + 上下文） |
| 连接建立速率 | 5000/秒 | CPU（TLS 握手 + 认证） |
| 心跳处理吞吐 | 20 万/秒 | CPU（5 秒间隔 × 10 万连接） |
| 消息广播吞吐 | 10 万条/秒 | CPU（序列化 + 系统调用） |
| 出站带宽 | 10Gbps | 网卡 |

**全集群容量（200 万在线）：**

| 资源 | 需求 | 单机能力 | 所需机器 | 月成本（按需） |
|------|------|---------|---------|--------------|
| WebSocket 连接 | 200 万 | 10 万 | 20 台 | 20 × $200 = $4,000 |
| 出站带宽（白板） | 4GB/秒 | 10Gbps | 4 台 | 已包含 |
| 出站带宽（音视频） | 200 堂 × 2Mbps = 400Mbps | 10Gbps | 1 台 | 已包含 |
| Redis Pub/Sub | 200 堂 × 100 ops/秒 = 2 万 ops/秒 | 10 万 ops/秒 | 1 台 | $150 |
| Redis Stream | 200 堂 × 100 ops/秒 = 2 万 ops/秒 | 10 万 ops/秒 | 1 台 | $150 |

### 白板操作吞吐量

**单课堂白板性能：**

| 操作 | 频率 | 单次大小 | 吞吐量 |
|------|------|---------|--------|
| 笔画点推送 | 100 点/秒 | 20 字节 | 2KB/秒 |
| Redis Pub/Sub publish | 100 次/秒 | 200 字节（含元数据） | 20KB/秒 |
| Redis Stream xadd | 100 次/秒 | 200 字节 | 20KB/秒 |
| MySQL 持久化（批量） | 1 次/秒 | 2KB | 2KB/秒 |
| 笔画完成快照 S3 | 1 次/5秒 | 5-50KB | 1-10KB/秒 |

**大班课扇出性能（1 万学生）：**

| 阶段 | 延迟 | 数据量 |
|------|------|--------|
| 老师端 → 信令服务 | < 5ms | 20 字节 |
| 信令服务 → Redis publish | < 1ms | 200 字节 |
| Redis → 200 个网关 fan-out | < 1ms | 200 × 200 = 40KB |
| 网关 → 本地 50 学生广播 | < 5ms | 50 × 20 = 1KB |
| **端到端延迟** | **< 12ms** | **总出站 1 万 × 20 = 200KB/秒** |

**小班课 CRDT 性能（20 人全员书写）：**

| 操作 | 频率 | 吞吐量 |
|------|------|--------|
| CRDT 操作生成 | 20 人 × 50 点/秒 = 1000 ops/秒 | 1000 × 50 字节 = 50KB/秒 |
| CRDT 操作广播 | 1000 ops/秒 × 19 接收者 | 19,000 ops/秒 |
| CRDT 合并计算 | 1000 ops/秒 | CPU < 1% |
| Lamport 时钟更新 | 1000 次/秒 | 无瓶颈 |
| 快照生成（每 5 分钟） | 1 次/5分钟 | ~100KB |

### 录制存储成本

**单堂 90 分钟课的存储需求：**

| 流 | 码率 | 文件大小 | S3 标准月费 | S3 Glacier 月费 |
|----|------|---------|-----------|----------------|
| 老师摄像头 | 2Mbps | 1.35GB | $0.031 | $0.005 |
| 屏幕共享 | 4Mbps | 2.70GB | $0.062 | $0.011 |
| 白板事件 | ~16Kbps | 10.8MB | $0.000 | $0.000 |
| 学生音频混流 | 128Kbps | 86.4MB | $0.002 | $0.000 |
| 合流回放 | 6Mbps | 4.05GB | $0.093 | $0.016 |
| HLS 切片（3 码率） | ~10.5Mbps | 7.09GB | $0.163 | $0.028 |
| **合计** | | **~15.3GB** | **$0.351** | **$0.060** |

**年度存储成本（200 堂/天 × 365 天）：**

| 存储层 | 90 天内 | 90 天后 | 年总成本 |
|--------|---------|---------|---------|
| S3 标准 | 200 × 365 × $0.351 × 90/365 = $6,318 | - | - |
| S3 Glacier | - | 200 × 365 × $0.060 × 275/365 = $9,041 | - |
| **合计** | | | **$15,359** |

优化策略：(1) 90 天后自动转 Glacier；(2) 合流回放保留永久，原始分轨 30 天后删除；(3) HLS 切片回放结束后 7 天删除（转为 MP4 点播）。

### CDN 带宽成本

**直播 CDN（实时音视频分发）：**

| 场景 | 码率 | 并发观众 | 带宽需求 | 月成本 |
|------|------|---------|---------|--------|
| 大班课单堂 | 2Mbps | 1 万 | 20Gbps | $0.08/GB × 20Gbps × 3600 × 1.5h × 30 = $259,200 |
| 200 堂同时 | 2Mbps | 200 万 | 4Tbps | 远超单 CDN 能力 |

实际优化：(1) 使用 P2P CDN（WebRTC mesh 补充），节省 40-60% CDN 带宽；(2) 分辨率自适应，弱网用户降码率；(3) 课堂高峰期（晚 7-9 点）预扩容 CDN 节点。

**回放 CDN（HLS 切片分发）：**

| 场景 | 平均码率 | 日播放量 | 带宽需求 | 月成本 |
|------|---------|---------|---------|--------|
| 课后回放 | 3Mbps | 10 万次 × 45 分钟 | ~2Tbps 峰值 | $0.02/GB × 10 万 × 1GB ≈ $2,000/天 |

回放 CDN 成本远低于直播（码率自适应 + 长尾分布 + 缓存命中率高）。

## 异常场景深度解析

### 异常场景 1：老师网络中断（课堂中途断连）

**场景描述：** 老师正在讲课，网络突然中断。1 万学生看到画面卡住，不知道发生了什么。如果 30 秒内老师没恢复，学生开始退出。

**影响分析：**

| 影响维度 | 具体影响 | 严重程度 |
|---------|---------|---------|
| 学生体验 | 画面卡住，无任何提示 | 严重 |
| 课堂进度 | 中断，无法继续 | 严重 |
| 心理影响 | 学生焦虑，以为课堂结束 | 中等 |
| 数据完整性 | 白板/翻页操作中断 | 中等 |
| 录制 | 音视频流断开 | 中等 |

**完整处理流程：**

```
老师断连检测（5秒无心跳）
    │
    ├─ 1. 自动暂停课堂 → 通知所有学生"老师网络异常，课堂暂停"
    │
    ├─ 2. 保存课堂状态快照（白板状态 + 当前页码 + 答题进度）
    │
    ├─ 3. 启动倒计时（60秒）→ 期间持续检测老师重连
    │
    ├─ 4a. 60秒内老师重连 → 恢复课堂，同步缺失状态
    │
    └─ 4b. 60秒超时 → 通知学生"课堂已暂停，请等待通知"
           → 通知老师（其他渠道：短信/APP推送）"您的课堂已暂停"
           → 课堂状态标记为"paused"
```

**代码实现：**

```python
class TeacherConnectionMonitor:
    """老师连接监控：检测断连 → 自动暂停 → 重连恢复"""

    HEARTBEAT_TIMEOUT = 5       # 5 秒无心跳视为断连
    RECONNECT_WAIT = 60         # 等待 60 秒重连
    CHECK_INTERVAL = 1          # 每秒检查一次

    def __init__(self, redis, event_store):
        self.redis = redis
        self.event_store = event_store
        self.active_monitors = {}  # classroom_id → monitor_task

    async def start_monitoring(self, classroom_id: str, teacher_id: int):
        """开始监控老师连接状态"""
        self.active_monitors[classroom_id] = asyncio.create_task(
            self._monitor_loop(classroom_id, teacher_id)
        )

    async def on_teacher_heartbeat(self, classroom_id: str):
        """收到老师心跳 → 更新时间戳"""
        await self.redis.set(
            f"teacher_heartbeat:{classroom_id}",
            str(int(time.time() * 1000)),
            ex=10  # 10 秒过期
        )

    async def _monitor_loop(self, classroom_id: str, teacher_id: int):
        """监控循环"""
        while True:
            await asyncio.sleep(self.CHECK_INTERVAL)

            # 检查心跳
            last_hb = await self.redis.get(f"teacher_heartbeat:{classroom_id}")
            if last_hb is not None:
                continue  # 心跳正常

            # 心跳超时 → 老师断连
            await self._handle_teacher_disconnect(classroom_id, teacher_id)
            break  # 退出监控循环

    async def _handle_teacher_disconnect(self, classroom_id: str, teacher_id: int):
        """处理老师断连"""
        disconnect_time = int(time.time() * 1000)

        # 1. 保存课堂状态快照
        snapshot = await self._save_classroom_snapshot(classroom_id)

        # 2. 自动暂停课堂 → 通知所有学生
        await self.redis.set(f"classroom_status:{classroom_id}", "paused")
        await self._broadcast_to_students(classroom_id, {
            "type": "classroom_paused",
            "reason": "teacher_network_error",
            "message": "老师网络异常，课堂已暂停，请稍候...",
            "estimated_wait": 60,
            "timestamp": disconnect_time
        })

        # 3. 通知运营（短信/APP推送给老师）
        await self._notify_teacher_via_fallback(teacher_id, classroom_id, "课堂已暂停，请尽快恢复网络")

        # 4. 等待重连
        reconnected = await self._wait_for_reconnect(classroom_id, self.RECONNECT_WAIT)

        if reconnected:
            # 5a. 老师重连 → 恢复课堂
            await self._resume_classroom(classroom_id, snapshot, disconnect_time)
        else:
            # 5b. 超时 → 课堂终止
            await self._terminate_classroom(classroom_id, teacher_id, disconnect_time)

    async def _save_classroom_snapshot(self, classroom_id: str) -> dict:
        """保存课堂状态快照"""
        # 获取当前白板状态
        wb_state = await self.redis.get(f"wb_state:{classroom_id}")

        # 获取当前课件页码
        current_page = await self.redis.get(f"slide_page:{classroom_id}")

        # 获取进行中的答题
        active_questions = await self.redis.smembers(f"active_questions:{classroom_id}")

        snapshot = {
            "classroom_id": classroom_id,
            "wb_state": wb_state,
            "current_page": int(current_page) if current_page else 1,
            "active_questions": list(active_questions),
            "timestamp": int(time.time() * 1000)
        }

        # 存储快照到 Redis（临时，恢复后删除）
        await self.redis.set(
            f"classroom_snapshot:{classroom_id}",
            json.dumps(snapshot),
            ex=3600  # 1 小时过期
        )

        return snapshot

    async def _wait_for_reconnect(self, classroom_id: str, timeout: int) -> bool:
        """等待老师重连"""
        deadline = time.time() + timeout
        while time.time() < deadline:
            # 检查老师是否重新连接
            status = await self.redis.get(f"teacher_status:{classroom_id}")
            if status and status.decode() == "connected":
                return True
            await asyncio.sleep(1)
        return False

    async def _resume_classroom(self, classroom_id: str, snapshot: dict, disconnect_time: int):
        """老师重连 → 恢复课堂"""
        # 1. 更新课堂状态
        await self.redis.set(f"classroom_status:{classroom_id}", "live")

        # 2. 向老师发送状态同步消息（补发断连期间的事件）
        missed_events = await self.event_store.query_range(
            classroom_id,
            from_time=disconnect_time,
            to_time=int(time.time() * 1000)
        )
        await self._push_to_teacher(classroom_id, {
            "type": "classroom_resumed",
            "snapshot": snapshot,
            "missed_events": missed_events,
            "disconnect_duration_ms": int(time.time() * 1000) - disconnect_time
        })

        # 3. 通知所有学生课堂恢复
        await self._broadcast_to_students(classroom_id, {
            "type": "classroom_resumed",
            "message": "老师已重新连接，课堂继续",
            "timestamp": int(time.time() * 1000)
        })

        # 4. 重新开始监控
        teacher_id = await self.redis.get(f"teacher_id:{classroom_id}")
        await self.start_monitoring(classroom_id, int(teacher_id))

    async def _terminate_classroom(self, classroom_id: str, teacher_id: int, disconnect_time: int):
        """超时 → 终止课堂"""
        await self.redis.set(f"classroom_status:{classroom_id}", "terminated")

        await self._broadcast_to_students(classroom_id, {
            "type": "classroom_terminated",
            "reason": "teacher_unavailable",
            "message": "老师无法恢复连接，课堂已结束。回放将在课后生成。",
            "timestamp": int(time.time() * 1000)
        })

        # 触发课后处理流程（生成回放、保存出勤记录等）
        await self._trigger_post_class_processing(classroom_id)
```

### 异常场景 2：白板同步冲突（两位老师同时编辑）

**场景描述：** 小班课中，老师 A 和老师 B（联合授课）同时在白板同一区域书写。如果使用简单的"最后写入胜出"策略，先写的老师笔画会被覆盖，导致内容丢失。

**冲突时序图：**

```
时间轴 →
老师A: ──添加笔画P1──────修改P1颜色为红──────┐
                                              │ 并发！
老师B: ────────添加笔画P2────修改P1颜色为蓝──┘

结果需要保证：A 和 B 最终看到相同的白板状态
```

**冲突解决策略：基于 CRDT 的确定性合并**

核心原则：(1) 添加操作天然无冲突（不同元素独立）；(2) 修改操作用 LWW（Last-Writer-Wins），Lamport 时间戳大的胜出；(3) 删除操作优先级最高（墓碑胜出）；(4) 所有操作幂等，重复应用不改变结果。

**冲突检测与解决代码：**

```python
class WhiteboardConflictResolver:
    """白板冲突解决器：处理并发编辑冲突"""

    def __init__(self, classroom_id: str, engine: WhiteboardCRDTEngine):
        self.classroom_id = classroom_id
        self.engine = engine
        self.conflict_log = []  # 记录冲突事件（用于调试和审计）

    async def apply_remote_op(self, op: WhiteboardOp) -> ConflictResolution:
        """应用远程操作，检测并解决冲突"""
        target_key = op.target_key

        # 检测并发修改冲突
        if op.op_type == OpType.MODIFY and target_key in self.engine.elements:
            local_element = self.engine.elements[target_key]
            local_version = local_element.version
            remote_version = op.payload.get("version", 0)

            if remote_version == local_version and remote_version > 0:
                # 版本号相同但修改内容不同 → 并发修改冲突
                local_mod_ts = self._get_last_modify_lamport(target_key)
                if local_mod_ts > 0 and op.lamport_ts > local_mod_ts:
                    # 远程更新 → 接受远程修改
                    resolution = ConflictResolution(
                        conflict_type="concurrent_modify",
                        resolution="remote_w
## 自适应学习路径引擎

```python
class AdaptiveLearningEngine:
    """自适应学习路径推荐"""

    def recommend_path(self, user_id, target_skill):
        """推荐个性化学习路径"""
        # 1. 评估当前技能水平
        current_skills = self._assess_skills(user_id)
        # 2. 构建技能图谱
        skill_graph = self._build_skill_graph(target_skill)
        # 3. 找到从当前技能到目标技能的最短路径
        path = self._find_learning_path(current_skills, skill_graph, target_skill)

        return {
            "user_id": user_id,
            "current_skills": current_skills,
            "target_skill": target_skill,
            "learning_path": path,
            "estimated_hours": sum(p["estimated_hours"] for p in path),
            "prerequisites_missing": [p for p in path if p["is_prerequisite"]]
        }

    def _assess_skills(self, user_id):
        """评估用户技能水平"""
        # 基于答题正确率和完成课程
        results = self.db.query(
            "SELECT skill_id, AVG(score) as avg_score "
            "FROM quiz_results WHERE user_id = %s "
            "GROUP BY skill_id", user_id)
        return {r["skill_id"]: r["avg_score"] for r in results}

    def _find_learning_path(self, current, graph, target):
        """BFS 找最短学习路径"""
        visited = set(current.keys())
        queue = [(target, [])]

        while queue:
            skill, path = queue.pop(0)
            if skill in visited:
                continue
            visited.add(skill)

            prerequisites = graph.get(skill, {}).get("prerequisites", [])
            course = self._find_best_course(skill, current)

            new_path = path + [{
                "skill_id": skill,
                "course_id": course["id"],
                "course_name": course["name"],
                "estimated_hours": course["duration_hours"],
                "is_prerequisite": skill not in current
            }]

            if not prerequisites:
                return list(reversed(new_path))

            for prereq in prerequisites:
                if prereq not in visited:
                    queue.append((prereq, new_path))

        return []
```

## 抄袭检测系统

```python
class PlagiarismDetector:
    """作业抄袭检测"""

    def check_submission(self, submission_id):
        """检测提交是否抄袭"""
        submission = self.db.get_submission(submission_id)

        # 1. 文本相似度（TF-IDF + 余弦相似度）
        text_similarities = self._check_text_similarity(submission)

        # 2. 代码相似度（AST 比对）
        code_similarities = self._check_code_similarity(submission)

        # 3. 交叉比对（同一课程所有提交）
        cross_check = self._cross_course_check(submission)

        # 综合判定
        max_similarity = max(
            max(s["similarity"] for s in text_similarities) if text_similarities else 0,
            max(s["similarity"] for s in code_similarities) if code_similarities else 0
        )

        result = {
            "submission_id": submission_id,
            "max_similarity": max_similarity,
            "text_matches": text_similarities[:5],
            "code_matches": code_similarities[:5],
            "verdict": "plagiarism" if max_similarity > 0.8 else
                       "suspicious" if max_similarity > 0.5 else "original"
        }

        if result["verdict"] != "original":
            self.notify_instructor(submission["course_id"], result)

        return result

    def _check_text_similarity(self, submission):
        """文本相似度检测"""
        from sklearn.feature_extraction.text import TfidfVectorizer
        from sklearn.metrics.pairwise import cosine_similarity

        # 获取同一课程所有提交
        peers = self.db.query(
            "SELECT id, content FROM submissions "
            "WHERE course_id = %s AND id != %s",
            submission["course_id"], submission["id"])

        vectorizer = TfidfVectorizer()
        all_texts = [submission["content"]] + [p["content"] for p in peers]
        tfidf = vectorizer.fit_transform(all_texts)

        similarities = []
        for i, peer in enumerate(peers, 1):
            sim = cosine_similarity(tfidf[0:1], tfidf[i:i+1])[0][0]
            if sim > 0.3:
                similarities.append({
                    "submission_id": peer["id"],
                    "similarity": float(sim)
                })

        return sorted(similarities, key=lambda x: x["similarity"], reverse=True)
```

## 异常场景补充

### 场景：直播流中断

```
触发：教师网络不稳定 → 直播流中断 → 学生无法观看
检测：
  1. 推流中断 > 10 秒 → 告警
  2. 学生端黑屏反馈 → 告警
处理：
  1. 自动切换到低码率推流
  2. 显示"教师网络不稳定，已切换低清晰度"
  3. 中断 > 2 分钟 → 自动转为语音模式（只保留音频）
  4. 完全中断 → 自动录制回放 → 通知教师稍后补录
预防：教师端多码率推流 + CDN 边缘节点就近接入
```

### 场景：抄袭检测误报

```
触发：两个学生独立完成相似解法 → 相似度 85% → 误判抄袭
检测：
  1. 教师申诉"非抄袭" → 标记误报
  2. 误报率 > 10% → 阈值需调整
处理：
  1. 教师手动标记为"非抄袭"
  2. 调整相似度阈值（0.8 → 0.85）
  3. 增加上下文分析：提交时间、编辑历史
预防：相似度阈值动态调整 + 多维度综合判定
```

## 直播录制与回放完整实现

```python
class LiveRecordingService:
    """直播录制与回放：HLS 流 + 章节标记 + 字幕搜索"""

    def start_recording(self, class_id):
        """开始录制直播课程"""
        recording_id = str(uuid4())
        # 1. 创建 HLS 录制任务
        self.db.insert("live_recordings", {
            "recording_id": recording_id,
            "class_id": class_id,
            "status": "recording",
            "started_at": now(),
            "hls_path": f"recordings/{class_id}/{recording_id}",
            "duration_seconds": 0
        })

        # 2. 启动 FFmpeg HLS 录制
        self.ffmpeg.start_hls_record(
            input_url=self.get_stream_url(class_id),
            output_path=f"s3://edu-recordings/{class_id}/{recording_id}",
            segment_duration=6  # 6秒一个 TS 片段
        )

        # 3. 开始字幕转录（实时）
        self.transcription_service.start_realtime(class_id, recording_id)

        return recording_id

    def add_chapter_marker(self, recording_id, title, timestamp):
        """添加章节标记（教师点击"标记重点"）"""
        self.db.insert("recording_chapters", {
            "recording_id": recording_id,
            "chapter_title": title,
            "timestamp_seconds": timestamp,
            "created_at": now()
        })

    def search_by_transcript(self, class_id, query):
        """通过字幕搜索回放片段"""
        results = self.elasticsearch.search({
            "index": "transcripts",
            "body": {
                "query": {"match": {"text": query}},
                "filter": [{"term": {"class_id": class_id}}],
                "highlight": {"fields": {"text": {}}}
            }
        })
        return [{
            "timestamp_seconds": r["_source"]["timestamp"],
            "text": r["_source"]["text"],
            "highlight": r.get("highlight", {}).get("text", [])
        } for r in results["hits"]["hits"][:5]]
```

## 课程评价与反馈系统

```python
class CourseReviewService:
    """课程评价与反馈系统"""

    def submit_review(self, user_id, course_id, rating, comment):
        """提交课程评价"""
        # 1. 检查是否已完成课程
        progress = self.db.get_course_progress(user_id, course_id)
        if progress["completion_rate"] < 0.8:
            raise ReviewNotAllowedError("课程完成度不足 80%，无法评价")

        # 2. 检查是否已评价
        existing = self.db.query_one(
            "SELECT * FROM course_reviews WHERE user_id = %s AND course_id = %s",
            user_id, course_id)
        if existing:
            # 更新评价
            self.db.update("course_reviews",
                {"rating": rating, "comment": comment, "updated_at": now()},
                {"id": existing["id"]})
        else:
            self.db.insert("course_reviews", {
                "user_id": user_id, "course_id": course_id,
                "rating": rating, "comment": comment,
                "created_at": now()
            })

        # 3. 更新课程平均评分
        avg_rating = self.db.query_one(
            "SELECT AVG(rating) as avg FROM course_reviews WHERE course_id = %s",
            course_id)["avg"]
        self.db.update("courses",
            {"avg_rating": avg_rating, "review_count": self.db.count("course_reviews", course_id=course_id)},
            {"id": course_id})

    def get_course_reviews(self, course_id, page=1, size=20):
        """获取课程评价列表"""
        reviews = self.db.query(
            "SELECT r.*, u.name as user_name FROM course_reviews r "
            "JOIN users u ON r.user_id = u.id "
            "WHERE r.course_id = %s ORDER BY r.created_at DESC LIMIT %s OFFSET %s",
            course_id, size, (page - 1) * size)

        # 统计评分分布
        distribution = self.db.query(
            "SELECT rating, COUNT(*) as count FROM course_reviews "
            "WHERE course_id = %s GROUP BY rating ORDER BY rating",
            course_id)

        return {
            "reviews": reviews,
            "avg_rating": self.db.get_course(course_id)["avg_rating"],
            "distribution": {d["rating"]: d["count"] for d in distribution}
        }
```

## 异常场景补充

### 场景：直播流卡顿降级

```python
class StreamQualityManager:
    """直播流质量动态调整"""
    def adjust_quality(self, class_id):
        bandwidth = self.get_avg_student_bandwidth(class_id)
        if bandwidth < 1:  # 平均带宽 < 1Mbps
            # 降级到低清流
            self.stream_server.switch_profile(class_id, "360p_500kbps")
            self.notify_students(class_id, "已切换到低清晰度以保证流畅")
        elif bandwidth < 3:
            self.stream_server.switch_profile(class_id, "720p_2mbps")
        else:
            self.stream_server.switch_profile(class_id, "1080p_4mbps")
```

### 场景：考试系统作弊检测

```
触发：在线考试中发现多个学生答案高度相似 → 疑似作弊
检测：
  1. 答案相似度 > 90% 且用时接近 → 疑似
  2. 同一 IP 多个考试 session → 共享屏幕
  3. 切屏次数 > 5 → 查看资料
处理：
  1. 标记疑似作弊 → 通知教师
  2. 教师审核 → 确认作弊 → 成绩作废
  3. 非作弊 → 解除标记
预防：切屏检测 + IP 异常 + 答案相似度分析
```

## 在线考试系统完整实现

```python
class OnlineExamService:
    """在线考试系统：防作弊 + 自动阅卷"""

    def create_exam(self, course_id, config):
        """创建考试"""
        exam_id = str(uuid4())
        # 1. 题目随机组卷
        questions = self._generate_paper(config)

        self.db.insert("exams", {
            "exam_id": exam_id, "course_id": course_id,
            "title": config["title"],
            "duration_minutes": config["duration"],
            "start_time": config["start_time"],
            "end_time": config["end_time"],
            "total_score": sum(q["score"] for q in questions),
            "questions": json.dumps(questions),
            "status": "scheduled",
            "anti_cheat_config": json.dumps({
                "shuffle_questions": True,
                "shuffle_options": True,
                "prevent_copy_paste": True,
                "screen_monitoring": True,
                "max_tab_switches": 3,
            })
        })
        return exam_id

    def submit_answer(self, exam_id, user_id, question_id, answer):
        """提交答案"""
        # 防作弊检查
        exam = self.db.get_exam(exam_id)
        anti_cheat = json.loads(exam["anti_cheat_config"])

        # 切屏次数检查
        switches = self.redis.get(f"tab_switch:{exam_id}:{user_id}")
        if switches and int(switches) > anti_cheat["max_tab_switches"]:
            self._flag_cheating(exam_id, user_id, "too_many_tab_switches")

        # 超时检查
        if now() > exam["end_time"]:
            raise ExamTimeoutError("考试已结束")

        # 保存答案
        self.db.insert("exam_answers", {
            "exam_id": exam_id, "user_id": user_id,
            "question_id": question_id, "answer": answer,
            "submitted_at": now()
        })

    def auto_grade(self, exam_id):
        """自动阅卷"""
        answers = self.db.query(
            "SELECT * FROM exam_answers WHERE exam_id = %s", exam_id)
        questions = json.loads(self.db.get_exam(exam_id)["questions"])
        question_map = {q["id"]: q for q in questions}

        results = []
        for answer in answers:
            question = question_map[answer["question_id"]]
            if question["type"] == "choice":
                score = question["score"] if answer["answer"] == question["correct_answer"] else 0
            elif question["type"] == "fill_blank":
                score = question["score"] * self._fuzzy_match(
                    answer["answer"], question["correct_answer"])
            else:  # essay → 人工阅卷
                score = None

            results.append({
                "user_id": answer["user_id"],
                "question_id": answer["question_id"],
                "score": score, "needs_manual_review": score is None
            })

        return results

    def _flag_cheating(self, exam_id, user_id, reason):
        """标记作弊嫌疑"""
        self.db.insert("exam_cheat_flags", {
            "exam_id": exam_id, "user_id": user_id,
            "reason": reason, "flagged_at": now()
        })
        self.notify_instructor(exam_id,
            f"学生 {user_id} 触发作弊检测: {reason}")
```

## 学习进度追踪

```python
class LearningProgressTracker:
    """学习进度追踪"""

    def update_progress(self, user_id, course_id, lesson_id, progress):
        """更新学习进度"""
        # 1. 记录学习行为
        self.db.insert("learning_activities", {
            "user_id": user_id, "course_id": course_id,
            "lesson_id": lesson_id, "progress": progress,
            "timestamp": now(),
            "duration_seconds": progress.get("watch_seconds", 0)
        })

        # 2. 更新课程总进度
        total_lessons = self.db.count("lessons", course_id=course_id)
        completed_lessons = self.db.count("learning_activities",
            user_id=user_id, course_id=course_id, progress__gte=0.9)
        completion_rate = completed_lessons / total_lessons

        self.db.upsert("course_progress", {
            "user_id": user_id, "course_id": course_id,
            "completion_rate": completion_rate,
            "total_study_hours": self._calc_total_hours(user_id, course_id),
            "last_study_at": now()
        }, keys=["user_id", "course_id"])

        # 3. 进度里程碑通知
        if completion_rate >= 0.5 and not self._milestone_notified(user_id, course_id, "50%"):
            self.notify(user_id, "恭喜完成 50% 课程！继续加油")
        if completion_rate >= 1.0:
            self._issue_certificate(user_id, course_id)

    def _issue_certificate(self, user_id, course_id):
        """发放结业证书"""
        cert_id = str(uuid4())
        self.db.insert("certificates", {
            "cert_id": cert_id, "user_id": user_id,
            "course_id": course_id, "issued_at": now()
        })
        self.notify(user_id, "恭喜完成课程！证书已生成")
```

## 异常场景补充

### 场景：考试服务器过载

```
触发：1000 学生同时在线考试 → 服务器响应慢 → 答案提交超时
检测：
  1. API 延迟 > 3s → 告警
  2. 答案提交失败率 > 1% → 严重告警
处理：
  1. 启用考试专用服务器（隔离其他流量）
  2. 答案先存本地 → 网络恢复后批量上传
  3. 延长考试时间（补偿延迟时间）
预防：考试专用资源池 + 本地缓存 + 流量预估
```

### 场景：证书生成失败

```
触发：PDF 生成服务崩溃 → 证书无法生成
检测：证书生成任务队列堆积 → 告警
处理：
  1. 降级：先发送文字版结业确认
  2. PDF 证书稍后补发
  3. 恢复后批量生成积压证书
预防：证书生成异步化 + 降级方案
```

## 课程搜索与推荐完整实现

```python
class CourseSearchService:
    """课程搜索：Elasticsearch 全文搜索"""

    def search(self, query, filters=None, page=1, size=20):
        """搜索课程"""
        body = {
            "query": {
                "bool": {
                    "must": [
                        {"multi_match": {
                            "query": query,
                            "fields": ["title^3", "description^2", "tags^1.5", "instructor_name"],
                            "type": "best_fields",
                            "fuzziness": "AUTO"
                        }}
                    ],
                    "filter": []
                }
            },
            "sort": [
                "_score",
                {"enrollment_count": {"order": "desc"}}
            ],
            "from": (page - 1) * size,
            "size": size,
            "highlight": {"fields": {"title": {}, "description": {}}}
        }

        # 添加过滤条件
        if filters:
            if filters.get("category"):
                body["query"]["bool"]["filter"].append(
                    {"term": {"category_id": filters["category"]}})
            if filters.get("price_range"):
                r = filters["price_range"]
                body["query"]["bool"]["filter"].append(
                    {"range": {"price": {"gte": r[0], "lte": r[1]}}})
            if filters.get("rating_min"):
                body["query"]["bool"]["filter"].append(
                    {"range": {"avg_rating": {"gte": filters["rating_min"]}}})

        result = self.elasticsearch.search(index="courses", body=body)
        return self._format_results(result)

    def autocomplete(self, prefix):
        """搜索自动补全"""
        return self.elasticsearch.search({
            "index": "courses",
            "body": {
                "suggest": {
                    "course_suggest": {
                        "prefix": prefix,
                        "completion": {"field": "suggest", "size": 8}
                    }
                }
            }
        })
```

## 作业提交与自动批改

```python
class HomeworkService:
    """作业提交与自动批改"""

    def create_assignment(self, course_id, config):
        """创建作业"""
        assignment_id = str(uuid4())
        self.db.insert("assignments", {
            "assignment_id": assignment_id,
            "course_id": course_id,
            "title": config["title"],
            "description": config["description"],
            "deadline": config["deadline"],
            "max_score": config["max_score"],
            "auto_grade": config.get("auto_grade", False),
            "test_cases": json.dumps(config.get("test_cases", [])),
            "created_at": now()
        })
        return assignment_id

    def submit_homework(self, assignment_id, user_id, content):
        """提交作业"""
        assignment = self.db.get_assignment(assignment_id)

        # 超时检查
        if now() > assignment["deadline"]:
            return {"status": "late", "penalty": 0.8}  # 迟交扣 20%

        submission_id = str(uuid4())
        self.db.insert("homework_submissions", {
            "submission_id": submission_id,
            "assignment_id": assignment_id,
            "user_id": user_id,
            "content": content,
            "status": "submitted",
            "submitted_at": now()
        })

        # 自动批改
        if assignment["auto_grade"]:
            result = self._auto_grade(assignment, content)
            self.db.update("homework_submissions",
                {"score": result["score"], "status": "graded",
                 "feedback": json.dumps(result["feedback"])},
                {"submission_id": submission_id})

        return {"submission_id": submission_id, "status": "submitted"}

    def _auto_grade(self, assignment, content):
        """自动批改（编程题）"""
        test_cases = json.loads(assignment["test_cases"])
        passed = 0
        feedback = []

        for tc in test_cases:
            # 在沙箱中执行代码
            result = self.sandbox.execute(content, tc["input"],
                timeout=5, memory_limit_mb=128)
            if result["output"].strip() == tc["expected_output"].strip():
                passed += 1
                feedback.append({"case": tc["name"], "result": "pass"})
            else:
                feedback.append({
                    "case": tc["name"], "result": "fail",
                    "expected": tc["expected_output"],
                    "actual": result["output"].strip()
                })

        score = int(passed / len(test_cases) * assignment["max_score"])
        return {"score": score, "feedback": feedback}
```

## 学习分析看板

```python
class LearningAnalyticsDashboard:
    """学习分析看板"""

    def get_student_dashboard(self, user_id):
        """学生个人学习看板"""
        # 学习时长热力图
        study_heatmap = self._get_study_heatmap(user_id, days=90)

        # 知识点掌握雷达图
        knowledge_radar = self._get_knowledge_radar(user_id)

        # 学习连续天数
        streak = self._get_learning_streak(user_id)

        # 与班级平均对比
        class_comparison = self._compare_with_class(user_id)

        return {
            "study_heatmap": study_heatmap,
            "knowledge_radar": knowledge_radar,
            "streak_days": streak,
            "class_comparison": class_comparison,
            "weekly_report_url": self._generate_weekly_report(user_id)
        }

    def _get_learning_streak(self, user_id):
        """学习连续天数"""
        streak = 0
        date = now().date()
        while True:
            has_activity = self.db.query_one(
                "SELECT COUNT(*) as cnt FROM learning_activities "
                "WHERE user_id = %s AND DATE(timestamp) = %s",
                user_id, date)
            if has_activity["cnt"] > 0:
                streak += 1
                date -= timedelta(days=1)
            else:
                break
        return streak
```

## 异常场景补充

### 场景：ES 索引损坏

```
触发：Elasticsearch 索引损坏 → 课程搜索不可用
检测：
  1. 搜索请求返回 5xx → 索引异常
  2. 集群健康状态为 red → 告警
处理：
  1. 从 MySQL 重建索引（全量同步）
  2. 重建期间搜索降级到数据库 LIKE 查询
  3. 重建完成后切换回 ES
预防：ES 集群冗余 + 定期快照 + 搜索降级方案
```

### 场景：自动批改沙箱逃逸

```
触发：学生提交恶意代码 → 绕过沙箱限制 → 访问服务器资源
检测：
  1. 沙箱进程 CPU/内存异常 → 可疑
  2. 沙箱网络访问 → 逃逸
处理：
  1. 立即杀掉沙箱进程
  2. 标记提交为"安全审查"
  3. 加强沙箱隔离策略
预防：容器级隔离 + 无网络策略 + 资源限制 + 代码预检
```

## 课程目录与搜索深度实现

### 层级分类树

```python
class CourseCategoryService:
    """
    课程分类服务：层级分类树管理
    支持三级分类：一级（学科）→ 二级（子方向）→ 三级（具体领域）
    """

    def get_category_tree(self) -> list:
        """获取完整分类树（带课程数量统计）"""
        # 一次查询所有分类，内存中构建树
        categories = self.db.query("""
            SELECT c.id, c.name, c.parent_id, c.level, c.sort_order,
                   c.icon_url,
                   (SELECT COUNT(*) FROM courses co
                    WHERE co.category_id = c.id AND co.status = 1) as direct_count,
                   (SELECT COUNT(*) FROM courses co
                    INNER JOIN course_categories cc ON co.id = cc.course_id
                    WHERE cc.category_id = c.id AND co.status = 1) as total_count
            FROM categories c
            WHERE c.status = 'active'
            ORDER BY c.level, c.sort_order
        """)

        # 构建树结构
        tree = []
        node_map = {}
        for cat in categories:
            node = {
                "id": cat["id"],
                "name": cat["name"],
                "level": cat["level"],
                "icon_url": cat["icon_url"],
                "course_count": cat["total_count"],
                "children": []
            }
            node_map[cat["id"]] = node

            if cat["parent_id"] is None:
                tree.append(node)
            elif cat["parent_id"] in node_map:
                node_map[cat["parent_id"]]["children"].append(node)

        return tree

    def get_category_path(self, category_id: int) -> list:
        """获取分类路径（面包屑导航）"""
        path = []
        current_id = category_id
        while current_id:
            cat = self.db.query_one(
                "SELECT id, name, parent_id FROM categories WHERE id = %s",
                current_id)
            if not cat:
                break
            path.insert(0, {"id": cat["id"], "name": cat["name"]})
            current_id = cat["parent_id"]
        return path

    def add_category(self, name: str, parent_id: int = None,
                      sort_order: int = 0):
        """添加分类节点"""
        if parent_id:
            parent = self.db.query_one(
                "SELECT level FROM categories WHERE id = %s", parent_id)
            if not parent:
                raise CategoryNotFoundError(parent_id)
            level = parent["level"] + 1
            if level > 3:
                raise MaxLevelExceededError("分类层级不能超过 3 级")
        else:
            level = 1

        self.db.insert("categories", {
            "name": name,
            "parent_id": parent_id,
            "level": level,
            "sort_order": sort_order,
            "status": "active",
            "created_at": now()
        })

        # 清除分类树缓存
        self.redis.delete("category_tree")
```

### Elasticsearch 全文搜索（带权重提升与筛选）

```python
class CourseSearchServiceV2:
    """
    课程搜索服务 V2：基于 Elasticsearch 的全文检索
    权重策略：标题 > 描述 > 标签 > 教师名
    支持筛选：价格、评分、时长、分类、难度
    支持排序：相关度、热度、最新、评分、价格
    """

    # 字段权重配置
    FIELD_BOOST = {
        "title": 10.0,           # 标题权重最高
        "title.std": 8.0,       # 标题精确匹配
        "tags": 5.0,            # 标签
        "teacher_name": 4.0,    # 教师名
        "description": 2.0,     # 描述
        "category_name": 3.0,   # 分类名
    }

    def search(self, query: str, filters: dict = None,
               sort: str = "relevance", page: int = 1,
               size: int = 20) -> dict:
        """
        全文搜索课程
        - query: 搜索关键词
        - filters: 筛选条件（价格、评分、时长、分类）
        - sort: 排序方式（relevance/popularity/newest/rating/price_asc/price_desc）
        """
        es_query = self._build_query(query, filters, sort, page, size)

        result = self.elasticsearch.search(
            index="courses",
            body=es_query
        )

        total = result["hits"]["total"]["value"]
        hits = result["hits"]["hits"]

        courses = []
        for hit in hits:
            source = hit["_source"]
            source["score"] = hit["_score"]
            source["highlight"] = hit.get("highlight", {})
            courses.append(source)

        # 聚合结果（筛选面板数据）
        aggregations = self._parse_aggregations(result.get("aggregations", {}))

        return {
            "total": total,
            "page": page,
            "size": size,
            "courses": courses,
            "aggregations": aggregations,
            "query": query
        }

    def _build_query(self, query: str, filters: dict, sort: str,
                      page: int, size: int) -> dict:
        """构建 ES 查询"""
        # 多字段匹配 + 权重提升
        multi_match = {
            "multi_match": {
                "query": query,
                "type": "best_fields",
                "fields": [f"{k}^{v}" for k, v in self.FIELD_BOOST.items()],
                "fuzziness": "AUTO",
                "prefix_length": 2,
                "cutoff_frequency": 0.01
            }
        }

        # 函数评分：加入新鲜度衰减和热度加分
        function_score = {
            "query": multi_match,
            "functions": [
                {
                    "gauss": {
                        "created_at": {
                            "origin": "now",
                            "scale": "30d",
                            "decay": 0.5
                        }
                    },
                    "weight": 0.3
                },
                {
                    "field_value_factor": {
                        "field": "enrollment_count",
                        "factor": 0.1,
                        "modifier": "log1p"
                    },
                    "weight": 0.2
                },
                {
                    "field_value_factor": {
                        "field": "avg_rating",
                        "factor": 0.5,
                        "modifier": "sqrt"
                    },
                    "weight": 0.3
                }
            ],
            "score_mode": "sum",
            "boost_mode": "multiply"
        }

        # 构建布尔查询
        bool_query = {"must": [function_score]}

        # 添加筛选条件
        if filters:
            filter_clauses = self._build_filters(filters)
            bool_query["filter"] = filter_clauses

        es_query = {
            "query": {"bool": bool_query},
            "from": (page - 1) * size,
            "size": size,
            "highlight": {
                "fields": {
                    "title": {"number_of_fragments": 0},
                    "description": {
                        "fragment_size": 150,
                        "number_of_fragments": 2
                    },
                    "tags": {"number_of_fragments": 0}
                },
                "pre_tags": ["<em>"],
                "post_tags": ["</em>"]
            },
            "aggregations": {
                "price_ranges": {
                    "range": {
                        "field": "price",
                        "ranges": [
                            {"key": "free", "to": 0.01},
                            {"key": "1-99", "from": 0.01, "to": 100},
                            {"key": "100-299", "from": 100, "to": 300},
                            {"key": "300-499", "from": 300, "to": 500},
                            {"key": "500+", "from": 500}
                        ]
                    }
                },
                "rating_ranges": {
                    "range": {
                        "field": "avg_rating",
                        "ranges": [
                            {"key": "4.5+", "from": 4.5},
                            {"key": "4.0-4.5", "from": 4.0, "to": 4.5},
                            {"key": "3.0-4.0", "from": 3.0, "to": 4.0}
                        ]
                    }
                },
                "categories": {
                    "terms": {"field": "category_id", "size": 20}
                },
                "duration_ranges": {
                    "range": {
                        "field": "total_duration_hours",
                        "ranges": [
                            {"key": "0-5h", "to": 5},
                            {"key": "5-20h", "from": 5, "to": 20},
                            {"key": "20-50h", "from": 20, "to": 50},
                            {"key": "50h+", "from": 50}
                        ]
                    }
                }
            }
        }

        # 排序
        if sort == "popularity":
            es_query["sort"] = [{"enrollment_count": "desc"}, "_score"]
        elif sort == "newest":
            es_query["sort"] = [{"created_at": "desc"}, "_score"]
        elif sort == "rating":
            es_query["sort"] = [{"avg_rating": "desc"}, "_score"]
        elif sort == "price_asc":
            es_query["sort"] = [{"price": "asc"}, "_score"]
        elif sort == "price_desc":
            es_query["sort"] = [{"price": "desc"}, "_score"]
        # sort == "relevance" 时使用默认评分排序

        return es_query

    def _build_filters(self, filters: dict) -> list:
        """构建筛选条件"""
        clauses = []

        if "category_id" in filters:
            # 包含子分类
            category_ids = self._get_descendant_categories(filters["category_id"])
            clauses.append({"terms": {"category_id": category_ids}})

        if "price_min" in filters or "price_max" in filters:
            price_range = {}
            if "price_min" in filters:
                price_range["gte"] = filters["price_min"]
            if "price_max" in filters:
                price_range["lte"] = filters["price_max"]
            clauses.append({"range": {"price": price_range}})

        if "rating_min" in filters:
            clauses.append({"range": {"avg_rating": {"gte": filters["rating_min"]}}})

        if "duration_min" in filters or "duration_max" in filters:
            dur_range = {}
            if "duration_min" in filters:
                dur_range["gte"] = filters["duration_min"]
            if "duration_max" in filters:
                dur_range["lte"] = filters["duration_max"]
            clauses.append({"range": {"total_duration_hours": dur_range}})

        if "is_free" in filters and filters["is_free"]:
            clauses.append({"range": {"price": {"lt": 0.01}}})

        if "level" in filters:
            clauses.append({"term": {"difficulty_level": filters["level"]}})

        return clauses

    def _parse_aggregations(self, aggs: dict) -> dict:
        """解析聚合结果"""
        result = {}
        if "price_ranges" in aggs:
            result["price_ranges"] = {
                b["key"]: b["doc_count"]
                for b in aggs["price_ranges"]["buckets"]
            }
        if "rating_ranges" in aggs:
            result["rating_ranges"] = {
                b["key"]: b["doc_count"]
                for b in aggs["rating_ranges"]["buckets"]
            }
        if "categories" in aggs:
            result["categories"] = [
                {"category_id": b["key"], "count": b["doc_count"]}
                for b in aggs["categories"]["buckets"]
            ]
        if "duration_ranges" in aggs:
            result["duration_ranges"] = {
                b["key"]: b["doc_count"]
                for b in aggs["duration_ranges"]["buckets"]
            }
        return result
```

### 搜索自动补全

```python
class SearchAutocompleteService:
    """搜索自动补全：基于用户搜索历史 + 热门搜索 + ES suggest"""

    def suggest(self, prefix: str, limit: int = 10) -> list:
        """搜索自动补全"""
        suggestions = []

        # 1. ES completion suggester（基于课程标题索引）
        es_result = self.elasticsearch.search(
            index="courses",
            body={
                "suggest": {
                    "course_suggest": {
                        "prefix": prefix,
                        "completion": {
                            "field": "title.suggest",
                            "size": limit,
                            "skip_duplicates": True
                        }
                    }
                }
            }
        )

        if es_result.get("suggest", {}).get("course_suggest"):
            for option in es_result["suggest"]["course_suggest"][0]["options"]:
                suggestions.append({
                    "text": option["_source"]["title"],
                    "type": "course_title",
                    "course_id": option["_source"]["id"],
                    "score": option["_score"]
                })

        # 2. 热门搜索词（Redis ZSet 维护）
        hot_searches = self.redis.zrevrangebyscore(
            "hot_searches", "+inf", "-inf", withscores=True, start=0, num=5)
        for term, score in hot_searches:
            if term.decode().startswith(prefix) and len(suggestions) < limit:
                suggestions.append({
                    "text": term.decode(),
                    "type": "hot_search",
                    "popularity": int(score)
                })

        return suggestions[:limit]

    def record_search(self, query: str):
        """记录搜索行为（用于热门搜索统计）"""
        normalized = query.strip().lower()
        if len(normalized) >= 2:
            self.redis.zincrby("hot_searches", 1, normalized)

    def get_hot_searches(self, limit: int = 20) -> list:
        """获取热门搜索词"""
        results = self.redis.zrevrangebyscore(
            "hot_searches", "+inf", "-inf", withscores=True, start=0, num=limit)
        return [{"keyword": r[0].decode(), "count": int(r[1])} for r in results]
```

**搜索性能指标：**

| 指标 | 目标值 | 说明 |
|------|--------|------|
| 搜索响应时间 | < 100ms | ES 查询 + 聚合 |
| 索引更新延迟 | < 5 秒 | 课程信息变更后 |
| 自动补全延迟 | < 50ms | 前端输入 2 字符后触发 |
| 搜索准确率 | > 90% | 人工评测 top5 相关性 |
| 聚合计算时间 | < 30ms | 筛选面板数据 |

## 学习分析仪表盘深度实现

### 学习时间热力图与知识点雷达

```python
class StudyTimeHeatmapService:
    """
    学习时间热力图：以周为周期，展示每小时的学习密度
    用途：帮助学生发现学习习惯，教师了解全班学习节奏
    """

    def get_user_heatmap(self, user_id: int, weeks: int = 4) -> dict:
        """获取用户学习时间热力图"""
        start_date = now() - timedelta(weeks=weeks)

        # 按星期+小时聚合学习时长
        records = self.db.query("""
            SELECT
                DAYOFWEEK(start_time) as day_of_week,
                HOUR(start_time) as hour,
                SUM(duration_seconds) as total_seconds
            FROM learning_activities
            WHERE user_id = %s AND start_time >= %s
            GROUP BY DAYOFWEEK(start_time), HOUR(start_time)
        """, user_id, start_date)

        # 构建热力图数据：7 天 × 24 小时
        heatmap = [[0] * 24 for _ in range(7)]
        for record in records:
            day = record["day_of_week"] - 1  # 0=周日 → 调整为 0=周一
            hour = record["hour"]
            heatmap[day][hour] = record["total_seconds"] // 60  # 转换为分钟

        max_minutes = max(max(row) for row in heatmap) if records else 1

        return {
            "user_id": user_id,
            "weeks": weeks,
            "heatmap": heatmap,
            "max_minutes": max_minutes,
            "total_hours": sum(sum(row) for row in heatmap) / 60,
            "peak_day": self._find_peak_day(heatmap),
            "peak_hour": self._find_peak_hour(heatmap)
        }

    def get_class_heatmap(self, course_id: int) -> dict:
        """获取班级学习时间热力图（全班平均）"""
        records = self.db.query("""
            SELECT
                DAYOFWEEK(la.start_time) as day_of_week,
                HOUR(la.start_time) as hour,
                AVG(la.duration_seconds) as avg_seconds
            FROM learning_activities la
            JOIN enrollments e ON la.user_id = e.user_id
            WHERE e.course_id = %s AND la.start_time >= DATE_SUB(NOW(), INTERVAL 4 WEEK)
            GROUP BY DAYOFWEEK(la.start_time), HOUR(la.start_time)
        """, course_id)

        heatmap = [[0] * 24 for _ in range(7)]
        for record in records:
            day = record["day_of_week"] - 1
            hour = record["hour"]
            heatmap[day][hour] = int(record["avg_seconds"]) // 60

        return {"course_id": course_id, "heatmap": heatmap}

    def _find_peak_day(self, heatmap: list) -> int:
        """找出学习量最大的一天"""
        day_totals = [sum(row) for row in heatmap]
        return day_totals.index(max(day_totals))

    def _find_peak_hour(self, heatmap: list) -> int:
        """找出学习量最大的小时"""
        hour_totals = [0] * 24
        for day in heatmap:
            for h in range(24):
                hour_totals[h] += day[h]
        return hour_totals.index(max(hour_totals))


class KnowledgeMasteryRadar:
    """
    知识点掌握度雷达图
    数据来源：章节测验正确率 + 练习完成率 + 知识点关联分析
    """

    def get_user_radar(self, user_id: int, course_id: int) -> dict:
        """获取用户知识点掌握度雷达图"""
        # 获取课程所有知识点维度
        dimensions = self.db.query("""
            SELECT kp.id, kp.name, kp.category
            FROM knowledge_points kp
            WHERE kp.course_id = %s
            ORDER BY kp.sort_order
        """, course_id)

        # 计算每个知识点的掌握度（0-100）
        mastery_scores = []
        for dim in dimensions:
            score = self._calculate_mastery(user_id, dim["id"])
            mastery_scores.append({
                "knowledge_point_id": dim["id"],
                "name": dim["name"],
                "category": dim["category"],
                "mastery_score": score
            })

        # 与班级平均对比
        class_avg = self._get_class_average_mastery(course_id)

        return {
            "user_id": user_id,
            "course_id": course_id,
            "dimensions": mastery_scores,
            "class_average": class_avg,
            "weak_points": [s for s in mastery_scores if s["mastery_score"] < 60],
            "strong_points": [s for s in mastery_scores if s["mastery_score"] >= 80]
        }

    def _calculate_mastery(self, user_id: int, kp_id: int) -> float:
        """计算单个知识点掌握度"""
        quiz_accuracy = self.db.query_one("""
            SELECT AVG(score) as avg FROM quiz_results
            WHERE user_id = %s AND knowledge_point_id = %s
        """, user_id, kp_id)

        exercise_completion = self.db.query_one("""
            SELECT
                COUNT(CASE WHEN status = 'completed' THEN 1 END) * 100.0 /
                NULLIF(COUNT(*), 0) as rate
            FROM exercise_assignments
            WHERE user_id = %s AND knowledge_point_id = %s
        """, user_id, kp_id)

        review_count = self.db.query_one("""
            SELECT COUNT(*) as cnt FROM learning_activities
            WHERE user_id = %s AND knowledge_point_id = %s
              AND activity_type = 'review'
        """, user_id, kp_id)

        quiz_score = (quiz_accuracy["avg"] or 0)
        exercise_score = (exercise_completion["rate"] or 0)
        review_score = min(100, (review_count["cnt"] or 0) * 20)

        return round(quiz_score * 0.5 + exercise_score * 0.3 + review_score * 0.2, 1)

    def _get_class_average_mastery(self, course_id: int) -> list:
        """获取班级各知识点平均掌握度"""
        return self.db.query("""
            SELECT kp.id, kp.name, AVG(qr.score) as avg_mastery
            FROM knowledge_points kp
            LEFT JOIN quiz_results qr ON kp.id = qr.knowledge_point_id
            LEFT JOIN enrollments e ON qr.user_id = e.user_id
            WHERE kp.course_id = %s AND e.course_id = %s
            GROUP BY kp.id, kp.name
            ORDER BY kp.sort_order
        """, course_id, course_id)
```

### 学习连续天数追踪与周进度报告

```python
class LearningStreakTracker:
    """学习连续天数追踪（类似 GitHub 贡献图）"""

    def update_streak(self, user_id: int):
        """更新学习连续天数"""
        today = now().date()
        yesterday = today - timedelta(days=1)

        today_activity = self.db.query_one("""
            SELECT COUNT(*) as cnt FROM learning_activities
            WHERE user_id = %s AND DATE(start_time) = %s
        """, user_id, today)

        if not today_activity or today_activity["cnt"] == 0:
            return

        streak = self.redis.get(f"streak:{user_id}")
        last_date_str = self.redis.get(f"streak:last_date:{user_id}")

        if streak and last_date_str:
            current_streak = int(streak)
            last_date = datetime.strptime(last_date_str.decode(), "%Y-%m-%d").date()

            if last_date == today:
                return
            elif last_date == yesterday:
                current_streak += 1
            else:
                current_streak = 1
        else:
            current_streak = 1

        self.redis.set(f"streak:{user_id}", current_streak)
        self.redis.set(f"streak:last_date:{user_id}", today.isoformat())

        # 里程碑奖励
        if current_streak in (7, 14, 30, 60, 100, 365):
            self._award_streak_badge(user_id, current_streak)

        self.db.upsert("user_streaks", {
            "user_id": user_id,
            "current_streak": current_streak,
            "longest_streak": self._get_longest_streak(user_id, current_streak),
            "last_active_date": today
        }, keys=["user_id"])

    def _get_longest_streak(self, user_id: int, current: int) -> int:
        record = self.db.query_one(
            "SELECT longest_streak FROM user_streaks WHERE user_id = %s", user_id)
        return max(record["longest_streak"], current) if record else current


class WeeklyProgressReportGenerator:
    """每周学习进度报告生成器"""

    def generate_report(self, user_id: int, course_id: int = None) -> dict:
        """生成周进度报告"""
        week_start = now() - timedelta(days=now().weekday())
        week_end = week_start + timedelta(days=7)

        study_time = self.db.query_one("""
            SELECT SUM(duration_seconds) as total
            FROM learning_activities
            WHERE user_id = %s AND start_time >= %s AND start_time < %s
              AND (%s IS NULL OR course_id = %s)
        """, user_id, week_start, week_end, course_id, course_id)

        completed_lessons = self.db.query_one("""
            SELECT COUNT(*) as cnt FROM lesson_completions
            WHERE user_id = %s AND completed_at >= %s AND completed_at < %s
              AND (%s IS NULL OR course_id = %s)
        """, user_id, week_start, week_end, course_id, course_id)

        quiz_scores = self.db.query("""
            SELECT qr.score, qr.max_score, kp.name as knowledge_point
            FROM quiz_results qr
            LEFT JOIN knowledge_points kp ON qr.knowledge_point_id = kp.id
            WHERE qr.user_id = %s AND qr.created_at >= %s AND qr.created_at < %s
              AND (%s IS NULL OR qr.course_id = %s)
            ORDER BY qr.created_at
        """, user_id, week_start, week_end, course_id, course_id)

        class_comparison = self._compare_with_class(user_id, course_id, week_start)

        weak_points = self.db.query("""
            SELECT kp.name, AVG(qr.score) as avg_score
            FROM quiz_results qr
            JOIN knowledge_points kp ON qr.knowledge_point_id = kp.id
            WHERE qr.user_id = %s AND qr.created_at >= %s AND qr.score < 60
            GROUP BY kp.name ORDER BY avg_score ASC LIMIT 5
        """, user_id, week_start)

        total_seconds = study_time["total"] or 0
        avg_score = (sum(s["score"] for s in quiz_scores) / len(quiz_scores)
                     if quiz_scores else 0)

        return {
            "user_id": user_id,
            "week_start": week_start.isoformat(),
            "week_end": week_end.isoformat(),
            "study_time_hours": round(total_seconds / 3600, 1),
            "lessons_completed": completed_lessons["cnt"],
            "quiz_average": round(avg_score, 1),
            "quiz_count": len(quiz_scores),
            "class_comparison": class_comparison,
            "weak_points": weak_points,
            "streak_days": self.redis.get(f"streak:{user_id}") or 0,
            "improvement_suggestions": self._generate_suggestions(
                weak_points, avg_score, total_seconds)
        }

    def _compare_with_class(self, user_id: int, course_id: int,
                             week_start) -> dict:
        """与班级平均对比"""
        class_stats = self.db.query_one("""
            SELECT
                AVG(total_seconds) as avg_study_time,
                AVG(quiz_avg) as avg_quiz_score,
                AVG(lessons_completed) as avg_lessons
            FROM weekly_user_stats
            WHERE course_id = %s AND week_start = %s
        """, course_id, week_start)

        return {
            "study_time_percentile": self._calculate_percentile(
                user_id, course_id, week_start, "total_seconds"),
            "quiz_score_percentile": self._calculate_percentile(
                user_id, course_id, week_start, "quiz_avg"),
            "class_avg_study_hours": round(
                (class_stats["avg_study_time"] or 0) / 3600, 1),
            "class_avg_quiz_score": round(
                class_stats["avg_quiz_score"] or 0, 1)
        }

    def _generate_suggestions(self, weak_points, avg_score, total_seconds):
        suggestions = []
        if total_seconds < 3600:
            suggestions.append("本周学习时长不足 1 小时，建议每天至少学习 30 分钟")
        if avg_score < 60:
            suggestions.append("测验平均分偏低，建议回顾基础知识点")
        if weak_points:
            names = "、".join(w["name"] for w in weak_points[:3])
            suggestions.append(f"薄弱知识点：{names}，建议加强练习")
        if total_seconds > 20 * 3600:
            suggestions.append("学习时长较长，注意劳逸结合，避免疲劳学习")
        return suggestions
```

## 作业提交与批改深度实现

### 作业创建与文件上传 + 代码提交

```python
class AssignmentServiceV2:
    """
    作业管理服务 V2
    支持：文本作业、文件上传、代码提交
    """

    def create_assignment(self, course_id: int, teacher_id: int,
                           config: dict) -> dict:
        """创建作业"""
        assignment_id = str(uuid4())

        submission_types = config.get("submission_types", ["text"])
        max_file_size_mb = config.get("max_file_size_mb", 50)
        allowed_extensions = config.get("allowed_extensions", None)

        # 代码作业特殊配置
        auto_grade_config = None
        if "code" in submission_types:
            auto_grade_config = {
                "language": config.get("language", "python"),
                "test_cases": config.get("test_cases", []),
                "time_limit_seconds": config.get("time_limit_seconds", 10),
                "memory_limit_mb": config.get("memory_limit_mb", 256),
                "timeout_grace_seconds": 2
            }

        self.db.insert("assignments", {
            "assignment_id": assignment_id,
            "course_id": course_id,
            "teacher_id": teacher_id,
            "title": config["title"],
            "description": config.get("description", ""),
            "deadline": config["deadline"],
            "submission_types": json.dumps(submission_types),
            "max_score": config.get("max_score", 100),
            "max_file_size_mb": max_file_size_mb,
            "allowed_extensions": json.dumps(allowed_extensions),
            "auto_grade_config": json.dumps(auto_grade_config),
            "status": "published",
            "created_at": now()
        })

        # 通知所有已选课学生
        enrolled = self.db.query(
            "SELECT user_id FROM enrollments WHERE course_id = %s", course_id)
        for student in enrolled:
            self.notify_user(student["user_id"], {
                "type": "new_assignment",
                "course_id": course_id,
                "assignment_id": assignment_id,
                "title": config["title"],
                "deadline": config["deadline"].isoformat()
            })

        return {"assignment_id": assignment_id, "status": "published"}

    def submit_homework(self, assignment_id: str, user_id: int,
                         submission: dict) -> dict:
        """提交作业"""
        assignment = self.db.query_one(
            "SELECT * FROM assignments WHERE assignment_id = %s", assignment_id)

        # 1. 检查截止时间
        if now() > assignment["deadline"]:
            late_penalty = self._calculate_late_penalty(assignment["deadline"], now())
        else:
            late_penalty = 0

        # 2. 检查提交次数限制
        existing_submissions = self.db.count("homework_submissions",
            assignment_id=assignment_id, user_id=user_id)
        if existing_submissions >= assignment.get("max_attempts", 3):
            return {"error": "exceeded_max_attempts"}

        # 3. 处理文件上传
        files = []
        if "files" in submission:
            for file_data in submission["files"]:
                if file_data["size"] > assignment["max_file_size_mb"] * 1024 * 1024:
                    return {"error": "file_too_large",
                            "max_size_mb": assignment["max_file_size_mb"]}
                if assignment.get("allowed_extensions"):
                    allowed = json.loads(assignment["allowed_extensions"])
                    ext = file_data["filename"].rsplit(".", 1)[-1].lower()
                    if ext not in allowed:
                        return {"error": "invalid_file_type", "allowed": allowed}
                s3_key = f"homework/{assignment_id}/{user_id}/{file_data['filename']}"
                self.s3.upload(s3_key, file_data["content"])
                files.append({"filename": file_data["filename"],
                              "s3_key": s3_key, "size_bytes": file_data["size"]})

        # 4. 代码提交
        code_content = None
        if "code" in submission:
            code_content = submission["code"]
            code_path = f"code/{assignment_id}/{user_id}/main.{self._get_extension(assignment)}"
            self.s3.upload(code_path, code_content.encode())

        # 5. 保存提交记录
        submission_id = str(uuid4())
        self.db.insert("homework_submissions", {
            "submission_id": submission_id,
            "assignment_id": assignment_id,
            "user_id": user_id,
            "content": submission.get("text", ""),
            "files": json.dumps(files),
            "code": code_content,
            "late_penalty": late_penalty,
            "status": "submitted",
            "submitted_at": now()
        })

        # 6. 触发自动批改（如果是代码作业）
        auto_grade_config = json.loads(assignment.get("auto_grade_config") or "null")
        if auto_grade_config:
            self.mq.produce("auto_grade", {
                "submission_id": submission_id,
                "assignment_id": assignment_id,
                "user_id": user_id,
                "config": auto_grade_config
            })

        # 7. 触发抄袭检测
        self.mq.produce("plagiarism_check", {
            "submission_id": submission_id,
            "assignment_id": assignment_id,
            "user_id": user_id
        })

        return {"submission_id": submission_id, "status": "submitted",
                "late_penalty": late_penalty}

    def _calculate_late_penalty(self, deadline, submitted_at):
        """计算迟交扣分（每小时扣 2%，最多扣 50%）"""
        hours_late = (submitted_at - deadline).total_seconds() / 3600
        return min(50, int(hours_late * 2))
```

### 自动批改（Docker 沙箱测试用例运行器）

```python
class AutoGraderServiceV2:
    """
    自动批改服务 V2：在 Docker 沙箱中运行学生代码并验证测试用例
    安全措施：Docker 容器隔离 + 资源限制 + 超时杀进程
    """

    SANDBOX_CONFIG = {
        "python": {
            "image": "auto-grader/python:3.11",
            "compile_cmd": None,
            "run_cmd": "python {source_file}",
            "source_file": "main.py"
        },
        "java": {
            "image": "auto-grader/java:17",
            "compile_cmd": "javac Main.java",
            "run_cmd": "java Main",
            "source_file": "Main.java"
        },
        "cpp": {
            "image": "auto-grader/cpp:17",
            "compile_cmd": "g++ -o main main.cpp -std=c++17",
            "run_cmd": "./main",
            "source_file": "main.cpp"
        }
    }

    TIMEOUT_EXIT_CODE = 124
    OOM_EXIT_CODE = 137

    def grade_submission(self, submission_id: str, assignment_id: str,
                          user_id: int, config: dict) -> dict:
        """自动批改学生代码"""
        submission = self.db.query_one(
            "SELECT * FROM homework_submissions WHERE submission_id = %s",
            submission_id)

        if not submission or not submission.get("code"):
            return {"error": "no_code_submitted"}

        language = config["language"]
        test_cases = config["test_cases"]
        time_limit = config.get("time_limit_seconds", 10)
        memory_limit = config.get("memory_limit_mb", 256)

        sandbox = self.SANDBOX_CONFIG.get(language)
        if not sandbox:
            return {"error": f"unsupported_language: {language}"}

        results = []
        total_score = 0
        max_score = len(test_cases)

        for i, test_case in enumerate(test_cases):
            case_result = self._run_test_case(
                sandbox, submission["code"], test_case,
                time_limit, memory_limit
            )
            case_result["test_case_index"] = i
            case_result["test_case_name"] = test_case.get("name", f"测试用例 {i+1}")

            if case_result["status"] == "passed":
                total_score += 1

            results.append(case_result)

        final_score = round(total_score / max_score * submission.get("max_score", 100), 1)

        self.db.update("homework_submissions", {
            "auto_grade_score": final_score,
            "auto_grade_results": json.dumps(results),
            "auto_grade_passed": total_score,
            "auto_grade_total": max_score,
            "status": "auto_graded"
        }, {"submission_id": submission_id})

        self.notify_user(user_id, {
            "type": "auto_grade_result",
            "assignment_id": assignment_id,
            "score": final_score,
            "passed": total_score,
            "total": max_score,
            "failed_cases": [r for r in results if r["status"] != "passed"]
        })

        return {"submission_id": submission_id, "score": final_score,
                "passed": total_score, "total": max_score, "results": results}

    def _run_test_case(self, sandbox: dict, code: str, test_case: dict,
                        time_limit: int, memory_limit: int) -> dict:
        """在 Docker 沙箱中运行单个测试用例"""
        import docker
        import tempfile

        with tempfile.TemporaryDirectory() as tmpdir:
            source_path = os.path.join(tmpdir, sandbox["source_file"])
            with open(source_path, "w") as f:
                f.write(code)

            input_data = test_case.get("input", "")
            input_path = os.path.join(tmpdir, "input.txt")
            with open(input_path, "w") as f:
                f.write(input_data)

            try:
                client = docker.from_env()

                if sandbox["compile_cmd"]:
                    compile_container = client.containers.run(
                        image=sandbox["image"],
                        command=sandbox["compile_cmd"],
                        volumes={tmpdir: {"bind": "/workspace", "mode": "rw"}},
                        working_dir="/workspace",
                        mem_limit=f"{memory_limit}m",
                        network_disabled=True,
                        detach=True
                    )
                    compile_result = compile_container.wait(timeout=30)
                    if compile_result["StatusCode"] != 0:
                        compile_logs = compile_container.logs().decode()
                        compile_container.remove()
                        return {"status": "compile_error", "error": compile_logs[:500]}
                    compile_container.remove()

                run_cmd = sandbox["run_cmd"].format(source_file=sandbox["source_file"])
                container = client.containers.run(
                    image=sandbox["image"],
                    command=f"timeout {time_limit + 2} sh -c 'cat input.txt | {run_cmd}'",
                    volumes={tmpdir: {"bind": "/workspace", "mode": "rw"}},
                    working_dir="/workspace",
                    mem_limit=f"{memory_limit}m",
                    network_disabled=True,
                    detach=True, stdout=True, stderr=True
                )

                result = container.wait(timeout=time_limit + 5)
                exit_code = result["StatusCode"]
                output = container.logs().decode()
                container.remove()

                if exit_code == self.TIMEOUT_EXIT_CODE:
                    return {"status": "timeout",
                            "error": f"运行超时（限制 {time_limit} 秒）"}
                elif exit_code == self.OOM_EXIT_CODE:
                    return {"status": "oom",
                            "error": f"内存超限（限制 {memory_limit}MB）"}
                elif exit_code != 0:
                    return {"status": "runtime_error", "exit_code": exit_code,
                            "error": output[-500:]}

                expected_output = test_case.get("expected_output", "").strip()
                actual_output = output.strip()

                if test_case.get("comparison_mode", "exact") == "exact":
                    passed = actual_output == expected_output
                elif test_case["comparison_mode"] == "ignore_whitespace":
                    passed = actual_output.split() == expected_output.split()
                elif test_case["comparison_mode"] == "float_tolerance":
                    try:
                        tolerance = test_case.get("tolerance", 1e-6)
                        passed = abs(float(actual_output) - float(expected_output)) < tolerance
                    except ValueError:
                        passed = False
                else:
                    passed = actual_output == expected_output

                if passed:
                    return {"status": "passed", "output": actual_output[:200]}
                else:
                    return {"status": "wrong_answer",
                            "expected": expected_output[:200],
                            "actual": actual_output[:200]}

            except Exception as e:
                return {"status": "system_error", "error": str(e)[:200]}
```

### 教师批注与抄袭检测集成

```python
class GradingWithAnnotationService:
    """教师批改 + 批注服务"""

    def teacher_review(self, submission_id: str, teacher_id: int,
                        review: dict) -> dict:
        """教师批改作业"""
        submission = self.db.query_one(
            "SELECT * FROM homework_submissions WHERE submission_id = %s",
            submission_id)

        auto_score = submission.get("auto_grade_score", 0)
        teacher_score = review.get("score", 0)
        final_score = teacher_score

        late_penalty = submission.get("late_penalty", 0)
        final_score = max(0, final_score - late_penalty)

        # 保存批注
        annotations = review.get("annotations", [])
        for ann in annotations:
            self.db.insert("submission_annotations", {
                "submission_id": submission_id,
                "teacher_id": teacher_id,
                "annotation_type": ann.get("type", "comment"),
                "position": json.dumps(ann.get("position", {})),
                "content": ann.get("content", ""),
                "score_deduction": ann.get("score_deduction", 0),
                "created_at": now()
            })

        self.db.update("homework_submissions", {
            "teacher_score": teacher_score,
            "final_score": final_score,
            "teacher_comment": review.get("comment", ""),
            "teacher_id": teacher_id,
            "status": "graded",
            "graded_at": now()
        }, {"submission_id": submission_id})

        self.notify_user(submission["user_id"], {
            "type": "homework_graded",
            "submission_id": submission_id,
            "final_score": final_score,
            "auto_score": auto_score,
            "teacher_comment": review.get("comment", ""),
            "has_annotations": len(annotations) > 0
        })

        return {"submission_id": submission_id, "final_score": final_score,
                "status": "graded"}

    def get_plagiarism_report(self, submission_id: str) -> dict:
        """获取抄袭检测结果报告"""
        report = self.db.query_one(
            "SELECT * FROM plagiarism_reports WHERE submission_id = %s",
            submission_id)

        if not report:
            return {"status": "pending", "message": "检测中，请稍后"}

        return {
            "submission_id": submission_id,
            "max_similarity": report["max_similarity"],
            "verdict": report["verdict"],
            "matched_submissions": json.loads(report["matched_submissions"]),
            "detail_url": f"/plagiarism/{report['report_id']}"
        }
```

**作业批改全流程：**

```
学生提交作业
    │
    ├─ 文本作业 → 直接进入教师批改队列
    │
    ├─ 代码作业 → 自动批改（Docker 沙箱 + 测试用例）
    │              │
    │              ├─ 通过 → 自动评分 + 教师复查
    │              ├─ 编译错误 → 0 分 + 通知学生
    │              ├─ 运行超时 → 0 分 + 提示优化
    │              └─ 答案错误 → 部分给分 + 提示失败用例
    │
    ├─ 文件上传 → 病毒扫描 → 存储到 S3
    │
    └─ 抄袭检测（异步，不阻塞批改）
         │
         ├─ 相似度 < 30% → 正常
         ├─ 相似度 30-80% → 标记"疑似"，教师确认
         └─ 相似度 > 80% → 标记"高疑似"，自动通知教师
```

## 补充异常场景

### 场景：Elasticsearch 索引损坏

```
触发：ES 集群磁盘故障 / 分片分配失败 → 索引不可读
检测：
  1. ES 健康检查：status=red → 严重告警
  2. 搜索 API 返回 5xx → 告警
  3. 分片未分配：_cluster/health 显示 unassigned_shards > 0
影响：
  - 课程搜索不可用 → 用户无法找到课程
  - 自动补全不可用
  - 字幕搜索不可用
处理：
  1. 降级：搜索切换到数据库 LIKE 查询（性能差但可用）
  2. 尝试自动修复：
     a. 重新分配分片：_cluster/reroute
     b. 从副本恢复：提升副本分片为主分片
  3. 副本也无法恢复：
     a. 从最近的快照恢复索引
     b. 无快照 → 从数据库全量重建索引
  4. 重建索引期间：搜索结果可能不完整，前端显示"搜索服务恢复中"
预防：
  - ES 集群至少 3 节点 + 每索引 1 副本
  - 每日自动快照到 S3
  - 搜索服务降级开关（一键切换到数据库查询）
```

### 场景：自动批改器遇到无限循环代码

```
触发：学生提交包含 while True: pass 的代码 → Docker 容器永不退出
检测：
  1. 容器运行时间超过 time_limit + 5 秒 → 超时
  2. Docker wait 超时 → 容器仍在运行
影响：
  - 占用 Docker 资源 → 影响其他学生代码的批改
  - 批改队列阻塞 → 其他作业等待时间过长
处理：
  1. 容器启动时设置 Docker 原生超时：
     --stop-timeout=15（SIGTERM 后 15 秒强制 SIGKILL）
  2. 外部看门狗：每 30 秒扫描运行超过 time_limit 的容器 → 强制杀掉
  3. 返回"运行超时"结果给学生，提示"可能存在无限循环"
  4. 记录超时代码用于教学分析（常见的无限循环模式）
代码示例（看门狗）：
  def kill_zombie_containers(self):
      running = self.docker_client.containers.list(
          filters={"label": "auto-grader"})
      for container in running:
          started = container.attrs["State"]["StartedAt"]
          elapsed = (now() - parse_docker_time(started)).total_seconds()
          if elapsed > self.MAX_CONTAINER_LIFETIME:
              container.kill()
              container.remove()
              self.alert(f"杀掉超时容器: {container.id}")
预防：
  - Docker 运行命令使用 timeout 前缀（双重超时保护）
  - 容器设置 CPU 限制（--cpus=1）防止跑满 CPU
  - 设置 PIDs 限制（--pids-limit=50）防止 fork 炸弹
  - 批改队列限流：同时运行的容器不超过 CPU 核数
```

## 在线考试防作弊系统完整实现

```python
class ExamAntiCheatService:
    """考试防作弊：监考 + 行为分析 + 设备检测"""

    def start_proctoring(self, exam_session_id, user_id):
        """开始监考"""
        # 1. 设备环境检测
        env = self._check_device_environment(user_id)
        if env["suspicious_apps"]:
            self._flag_violation(exam_session_id, user_id, "suspicious_apps",
                f"检测到可疑应用: {env['suspicious_apps']}")

        # 2. 人脸验证
        identity = self._verify_identity(user_id)
        if not identity["matched"]:
            return {"status": "identity_mismatch", "allowed": False}

        # 3. 启动持续监考
        self.db.update("exam_sessions",
            {"proctoring_status": "active", "started_at": now()},
            {"id": exam_session_id})

        return {"status": "proctoring_started", "allowed": True}

    def analyze_behavior(self, exam_session_id, behavior_event):
        """分析考试行为"""
        violations = []

        # 1. 视线离开屏幕
        if behavior_event["type"] == "gaze_away":
            duration = behavior_event.get("duration_seconds", 0)
            if duration > 10:
                violations.append({
                    "type": "gaze_away",
                    "severity": "medium" if duration < 30 else "high",
                    "duration": duration
                })

        # 2. 多人出现在画面中
        if behavior_event["type"] == "multiple_faces":
            violations.append({
                "type": "multiple_faces",
                "severity": "high",
                "face_count": behavior_event.get("face_count", 2)
            })

        # 3. 切换窗口/标签页
        if behavior_event["type"] == "tab_switch":
            violations.append({
                "type": "tab_switch",
                "severity": "medium",
                "tab_count": behavior_event.get("count", 1)
            })

        # 4. 复制粘贴操作
        if behavior_event["type"] == "clipboard":
            violations.append({
                "type": "clipboard_operation",
                "severity": "high",
                "content_length": behavior_event.get("content_length", 0)
            })

        # 5. 答题速度异常（过快 → 可能提前获取答案）
        if behavior_event["type"] == "answer_submitted":
            time_per_question = behavior_event.get("time_seconds", 0)
            question = self.db.get_question(behavior_event["question_id"])
            expected_min_time = question.get("min_answer_time_seconds", 10)
            if time_per_question < expected_min_time * 0.3:
                violations.append({
                    "type": "suspicious_speed",
                    "severity": "medium",
                    "actual_time": time_per_question,
                    "expected_min": expected_min_time
                })

        # 记录违规
        for v in violations:
            self._flag_violation(exam_session_id,
                behavior_event["user_id"], v["type"], json.dumps(v))

        return {"violations": violations}

    def generate_cheat_report(self, exam_session_id):
        """生成作弊检测报告"""
        violations = self.db.query(
            "SELECT * FROM exam_violations "
            "WHERE exam_session_id = %s ORDER BY timestamp", exam_session_id)

        # 计算风险分数
        risk_score = 0
        for v in violations:
            severity_scores = {"low": 5, "medium": 15, "high": 30}
            risk_score += severity_scores.get(v["severity"], 5)

        # 多次相同类型违规 → 加权
        type_counts = {}
        for v in violations:
            type_counts[v["violation_type"]] = type_counts.get(v["violation_type"], 0) + 1
        for vtype, count in type_counts.items():
            if count > 3:
                risk_score += (count - 3) * 10

        verdict = "clean" if risk_score < 15 else \
                  "suspicious" if risk_score < 45 else \
                  "likely_cheating" if risk_score < 80 else "confirmed_cheating"

        return {
            "exam_session_id": exam_session_id,
            "violation_count": len(violations),
            "risk_score": risk_score,
            "verdict": verdict,
            "violations": [{"type": v["violation_type"], "severity": v["severity"],
                          "timestamp": v["timestamp"].isoformat()} for v in violations]
        }
```

## 异常场景补充

### 场景：防作弊误判

```
触发：考生思考时视线自然离开屏幕 → 被判定为作弊 → 申诉
检测：
  1. 申诉率 > 10% → 系统过于敏感
  2. "gaze_away" 违规占比 > 50% → 阈值过严
处理：
  1. 放宽视线偏离阈值（10s → 30s）
  2. 加入"思考模式"（考生可标记正在思考）
  3. 人工复核高风险判定
预防：阈值保守设置 + 思考模式 + 人工复核
```

### 场景：监考服务中断

```
触发：监考服务宕机 → 考试进行中无法监考 → 防作弊失效
检测：
  1. 监考服务健康检查失败 → 中断
  2. 考试期间无监考数据 → 失效
处理：
  1. 考试继续（本地录制行为，恢复后上传）
  2. 恢复后补分析录制数据
  3. 严重中断 → 延长考试时间
预防：本地录制兜底 + 恢复后补分析 + 延时补偿
```

## 学习路径推荐完整实现

```python
class LearningPathRecommendationService:
    """学习路径推荐：知识图谱 + 能力评估 + 个性化路径"""

    def generate_learning_path(self, user_id, target_skill):
        """生成个性化学习路径"""
        # 1. 评估当前能力
        current_abilities = self._assess_abilities(user_id)

        # 2. 构建知识图谱
        skill_graph = self._get_skill_graph(target_skill)

        # 3. 找到从当前到目标的差距
        gaps = self._find_skill_gaps(current_abilities, skill_graph, target_skill)

        # 4. 生成路径（拓扑排序 + 优先级）
        path = self._build_path(gaps, skill_graph)

        # 5. 匹配课程资源
        for node in path:
            node["recommended_courses"] = self._find_courses(node["skill"], node["level"])

        return {"user_id": user_id, "target": target_skill,
                "current_abilities": current_abilities,
                "gaps": gaps, "path": path,
                "estimated_hours": sum(n.get("estimated_hours", 0) for n in path)}

    def _assess_abilities(self, user_id):
        """评估用户当前能力"""
        # 基于历史学习记录和考试成绩
        abilities = {}
        completed = self.db.query(
            "SELECT c.skill_tag, c.difficulty_level, uc.score "
            "FROM user_courses uc "
            "JOIN courses c ON uc.course_id = c.id "
            "WHERE uc.user_id = %s AND uc.status = 'completed'",
            user_id)

        for c in completed:
            skill = c["skill_tag"]
            level = c["difficulty_level"]
            score = c["score"]

            if skill not in abilities:
                abilities[skill] = {"level": 0, "confidence": 0}

            # 根据课程难度和得分更新能力等级
            level_map = {"beginner": 1, "intermediate": 2, "advanced": 3, "expert": 4}
            course_level = level_map.get(level, 1)

            if score >= 80:  # 高分 → 确认掌握
                abilities[skill]["level"] = max(abilities[skill]["level"], course_level)
                abilities[skill]["confidence"] = min(1.0, abilities[skill]["confidence"] + 0.3)
            elif score >= 60:  # 及格 → 部分掌握
                abilities[skill]["level"] = max(abilities[skill]["level"], course_level - 0.5)
                abilities[skill]["confidence"] = min(1.0, abilities[skill]["confidence"] + 0.1)

        return abilities

    def _find_skill_gaps(self, current, skill_graph, target):
        """找到技能差距"""
        required_skills = self._get_prerequisite_chain(skill_graph, target)
        gaps = []

        for skill, required_level in required_skills.items():
            current_level = current.get(skill, {}).get("level", 0)
            if current_level < required_level:
                gaps.append({
                    "skill": skill,
                    "current_level": current_level,
                    "required_level": required_level,
                    "gap": required_level - current_level
                })

        return gaps

    def _build_path(self, gaps, skill_graph):
        """构建学习路径（拓扑排序）"""
        # 按依赖关系排序
        path = []
        resolved = set()

        # 迭代解析依赖
        remaining = list(gaps)
        max_iterations = len(remaining) * 2
        iteration = 0

        while remaining and iteration < max_iterations:
            iteration += 1
            for gap in remaining[:]:
                prereqs = skill_graph.get(gap["skill"], {}).get("prerequisites", [])
                # 检查前置技能是否已解决
                unresolved_prereqs = [p for p in prereqs
                    if any(g["skill"] == p for g in remaining) and p not in resolved]

                if not unresolved_prereqs:
                    path.append({
                        "skill": gap["skill"],
                        "level": gap["required_level"],
                        "estimated_hours": int(gap["gap"] * 20),  # 每级约 20 小时
                        "order": len(path) + 1
                    })
                    resolved.add(gap["skill"])
                    remaining.remove(gap)

        return path

    def _find_courses(self, skill, level):
        """查找匹配课程"""
        courses = self.db.query(
            "SELECT * FROM courses "
            "WHERE skill_tag = %s AND difficulty_level = %s "
            "AND status = 'published' "
            "ORDER BY rating DESC LIMIT 3",
            skill, self._level_to_name(level))
        return [{"id": c["id"], "name": c["name"], "rating": c["rating"],
                "duration_hours": c.get("duration_hours", 0)} for c in courses]

    def _level_to_name(self, level):
        """数字等级转名称"""
        mapping = {0: "beginner", 1: "beginner", 2: "intermediate",
                  3: "advanced", 4: "expert"}
        return mapping.get(int(level), "beginner")
```

## 异常场景补充

### 场景：学习路径过长导致用户放弃

```
触发：目标技能需要 15 个前置技能 → 路径 200+ 小时 → 用户望而却步
检测：
  1. 路径完成率 < 10% → 路径过长
  2. 用户在前 3 个节点放弃 → 路径问题
处理：
  1. 提供快捷路径（跳过已部分掌握的技能）
  2. 将长路径拆分为里程碑
  3. 提供"最小可行路径"（只学核心前置）
预防：里程碑拆分 + 最小路径 + 渐进式推荐
```

### 场景：知识图谱依赖循环

```
触发：技能 A 依赖 B，B 依赖 C，C 依赖 A → 拓扑排序死循环
检测：
  1. 路径生成超时 → 可能循环依赖
  2. 知识图谱中存在环 → 数据错误
处理：
  1. 检测并打破循环（移除最弱的依赖边）
  2. 人工修正知识图谱
  3. 路径生成加入最大迭代限制
预防：知识图谱创建时检测环 + 最大迭代限制 + 人工审核
```

## 在线教育直播课堂完整实现

```python
class LiveClassService:
    """直播课堂：创建 + 连麦 + 白板 + 录播"""

    def create_live_class(self, teacher_id, course_id, title,
                         scheduled_time, duration_minutes=90):
        """创建直播课堂"""
        class_id = str(uuid4())

        # 1. 创建课堂
        self.db.insert("live_classes", {
            "class_id": class_id,
            "course_id": course_id,
            "teacher_id": teacher_id,
            "title": title,
            "scheduled_time": scheduled_time,
            "duration_minutes": duration_minutes,
            "status": "scheduled",
            "max_students": 500,
            "enable_chat": True,
            "enable_whiteboard": True,
            "enable_screen_share": True,
            "created_at": now()
        })

        # 2. 创建直播房间
        room = self.live_streaming.create_room(
            room_id=class_id,
            host_id=teacher_id,
            max_participants=500)

        # 3. 预约录制
        self.recording.schedule_recording(class_id, scheduled_time,
            duration_minutes + 30)  # 多录 30 分钟

        # 4. 通知报名学生
        enrolled = self.db.query(
            "SELECT user_id FROM course_enrollments "
            "WHERE course_id = %s AND status = 'active'", course_id)

        for student in enrolled:
            self.notification.send(student["user_id"],
                f"直播课堂提醒: {title}, 时间: {scheduled_time.strftime('%Y-%m-%d %H:%M')}")

        return {"class_id": class_id, "room_id": room["room_id"],
                "stream_url": room["stream_url"]}

    def start_class(self, class_id, teacher_id):
        """开始直播"""
        class_info = self.db.get_live_class(class_id)

        if class_info["teacher_id"] != teacher_id:
            raise PermissionDeniedError("只有授课教师可以开始直播")

        # 1. 更新状态
        self.db.update("live_classes",
            {"status": "live", "started_at": now()},
            {"class_id": class_id})

        # 2. 开始直播推流
        self.live_streaming.start_broadcast(class_id)

        # 3. 开始录制
        self.recording.start_recording(class_id)

        # 4. 通知学生
        enrolled = self.db.query(
            "SELECT user_id FROM course_enrollments "
            "WHERE course_id = %s AND status = 'active'", class_info["course_id"])

        for student in enrolled:
            self.notification.send(student["user_id"],
                f"直播课堂已开始: {class_info['title']}，点击进入")

        return {"status": "live", "stream_url": self.live_streaming.get_stream_url(class_id)}

    def request_microphone(self, class_id, student_id):
        """学生请求连麦"""
        # 1. 检查是否在课堂中
        attendance = self.db.query_one(
            "SELECT * FROM live_class_attendance "
            "WHERE class_id = %s AND user_id = %s",
            class_id, student_id)
        if not attendance:
            return {"status": "not_in_class"}

        # 2. 检查是否有正在进行的连麦请求
        pending = self.db.count("mic_requests",
            class_id=class_id, status="pending")
        if pending >= 3:
            return {"status": "queue_full", "message": "当前连麦排队人数较多"}

        # 3. 创建连麦请求
        request_id = str(uuid4())
        self.db.insert("mic_requests", {
            "request_id": request_id,
            "class_id": class_id,
            "student_id": student_id,
            "status": "pending",
            "created_at": now()
        })

        # 4. 通知教师
        class_info = self.db.get_live_class(class_id)
        self.notification.send(class_info["teacher_id"],
            f"学生 {student_id} 请求连麦")

        return {"request_id": request_id, "status": "pending"}

    def approve_microphone(self, request_id, teacher_id):
        """教师批准连麦"""
        request = self.db.get_mic_request(request_id)
        class_info = self.db.get_live_class(request["class_id"])

        if class_info["teacher_id"] != teacher_id:
            raise PermissionDeniedError("只有教师可以批准连麦")

        # 1. 更新请求状态
        self.db.update("mic_requests",
            {"status": "approved", "approved_at": now()},
            {"request_id": request_id})

        # 2. 开启学生麦克风
        self.live_streaming.enable_microphone(
            request["class_id"], request["student_id"])

        # 3. 通知学生
        self.notification.send(request["student_id"],
            "您的连麦请求已批准，请发言")

        return {"status": "approved"}

    def end_class(self, class_id, teacher_id):
        """结束直播"""
        class_info = self.db.get_live_class(class_id)

        # 1. 更新状态
        self.db.update("live_classes",
            {"status": "ended", "ended_at": now()},
            {"class_id": class_id})

        # 2. 停止直播
        self.live_streaming.stop_broadcast(class_id)

        # 3. 停止录制 → 自动转录播
        recording = self.recording.stop_recording(class_id)
        self.db.update("live_classes",
            {"recording_url": recording["url"],
             "recording_duration": recording["duration_seconds"]},
            {"class_id": class_id})

        # 4. 生成回放
        self._generate_replay(class_id, recording)

        # 5. 记录出勤
        attendance_count = self.db.count("live_class_attendance",
            class_id=class_id)
        self.db.update("live_classes",
            {"attendance_count": attendance_count},
            {"class_id": class_id})

        return {"status": "ended", "recording_url": recording["url"],
                "attendance": attendance_count}

    def _generate_replay(self, class_id, recording):
        """生成录播回放"""
        # 转码 + 生成章节标记
        self.video_processor.transcode(recording["url"],
            formats=["mp4_720p", "mp4_1080p"])

        # 基于白板截图生成章节
        whiteboard_snapshots = self.db.query(
            "SELECT timestamp, title FROM whiteboard_events "
            "WHERE class_id = %s ORDER BY timestamp", class_id)

        chapters = [{"time": s["timestamp"], "title": s["title"]}
                    for s in whiteboard_snapshots]

        self.db.update("live_classes",
            {"chapters": json.dumps(chapters), "replay_available": True},
            {"class_id": class_id})
```

## 异常场景补充

### 场景：直播推流中断

```
触发：教师网络不稳定 → 推流中断 3 分钟 → 学生看到黑屏 → 课堂体验差
检测：
  1. 推流中断 > 30 秒 → 网络问题
  2. 学生端卡顿投诉增多 → 推流问题
处理：
  1. 自动降级推流质量（1080p → 720p）
  2. 显示"教师网络不稳定"提示
  3. 中断超过 5 分钟 → 暂停课堂，稍后继续
预防：推流降级 + 状态提示 + 自动暂停
```

### 场景：录播回放生成延迟

```
触发：90 分钟直播 → 转码需 2 小时 → 学生当天无法看回放 → 学习进度受阻
检测：
  1. 回放生成时间 > 1 小时 → 延迟
  2. 学生投诉回放不可用 → 转码延迟
处理：
  1. 优先生成低质量回放（720p → 5 分钟可用）
  2. 高质量版本异步生成
  3. 增加转码服务器
预防：低质量优先 + 异步高质量 + 转码扩容
```

## 在线教育学习路径推荐完整实现

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Optional
from datetime import datetime, timedelta
import logging
import math

logger = logging.getLogger(__name__)


class LearningStyle(Enum):
    VISUAL = "visual"
    AUDITORY = "auditory"
    READING = "reading"
    KINESTHETIC = "kinesthetic"


class CourseDifficulty(Enum):
    BEGINNER = "beginner"
    INTERMEDIATE = "intermediate"
    ADVANCED = "advanced"
    EXPERT = "expert"


class PathStatus(Enum):
    NOT_STARTED = "not_started"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    ABANDONED = "abandoned"
    PAUSED = "paused"


@dataclass
class Course:
    course_id: str
    title: str
    difficulty: CourseDifficulty
    category: str
    prerequisites: list[str] = field(default_factory=list)
    estimated_hours: float = 0.0
    tags: list[str] = field(default_factory=list)
    curriculum_required: bool = False


@dataclass
class LearningRecord:
    course_id: str
    completed: bool
    quiz_scores: list[float] = field(default_factory=list)
    time_spent_hours: float = 0.0
    completion_date: Optional[datetime] = None
    engagement_score: float = 0.0  # 0-1, how engaged the student was


@dataclass
class KnowledgeGap:
    category: str
    severity: float  # 0-1, how critical the gap is
    missing_courses: list[str] = field(default_factory=list)
    description: str = ""


@dataclass
class PathNode:
    course_id: str
    order: int
    status: PathStatus = PathStatus.NOT_STARTED
    recommended_reason: str = ""
    priority: float = 0.0


@dataclass
class LearningPath:
    path_id: str
    student_id: str
    nodes: list[PathNode] = field(default_factory=list)
    created_at: datetime = field(default_factory=datetime.now)
    updated_at: datetime = field(default_factory=datetime.now)
    total_estimated_hours: float = 0.0
    target_completion_date: Optional[datetime] = None
    status: PathStatus = PathStatus.NOT_STARTED


@dataclass
class StudentProfile:
    student_id: str
    learning_records: list[LearningRecord] = field(default_factory=list)
    learning_style: LearningStyle = LearningStyle.READING
    weekly_available_hours: float = 10.0
    preferred_session_length_minutes: int = 45
    mastery_scores: dict[str, float] = field(default_factory=dict)  # category -> 0-1
    average_quiz_score: float = 0.0
    average_engagement: float = 0.0
    streak_days: int = 0
    completion_rate: float = 0.0  # ratio of started courses that were completed


class LearningPathRecommendationService:
    """在线教育学习路径推荐服务，根据学生历史学习数据分析知识差距并推荐个性化学习路径。"""

    DIFFICULTY_ORDER = {
        CourseDifficulty.BEGINNER: 0,
        CourseDifficulty.INTERMEDIATE: 1,
        CourseDifficulty.ADVANCED: 2,
        CourseDifficulty.EXPERT: 3,
    }

    MASTERY_THRESHOLD = 0.7  # 低于此分数视为未掌握
    ENGAGEMENT_THRESHOLD = 0.4  # 低于此分数视为低参与度
    MAX_PATH_LENGTH = 12  # 单条路径最大课程数
    MIN_WEEKLY_HOURS = 2.0  # 最少每周学习时长

    def __init__(
        self,
        course_catalog: dict[str, Course],
        curriculum_requirements: dict[str, list[str]],
    ):
        self.course_catalog = course_catalog
        self.curriculum_requirements = curriculum_requirements  # category -> required course_ids
        self._path_counter = 0
        self._active_paths: dict[str, LearningPath] = {}

    def _generate_path_id(self) -> str:
        self._path_counter += 1
        return f"LP-{self._path_counter:06d}"

    def analyze_student_profile(self, student_id: str, records: list[LearningRecord]) -> StudentProfile:
        """分析学生历史学习记录，构建学习画像。"""
        if not records:
            logger.warning(f"学生 {student_id} 没有学习记录，将使用默认画像")
            return StudentProfile(student_id=student_id)

        completed_records = [r for r in records if r.completed]
        incomplete_records = [r for r in records if not r.completed]

        # 计算各类别掌握程度
        mastery_scores: dict[str, list[float]] = {}
        for record in completed_records:
            course = self.course_catalog.get(record.course_id)
            if not course:
                continue
            avg_score = sum(record.quiz_scores) / len(record.quiz_scores) if record.quiz_scores else 0.0
            mastery_scores.setdefault(course.category, []).append(avg_score)

        category_mastery: dict[str, float] = {}
        for cat, scores in mastery_scores.items():
            # 用加权平均：最近的分数权重更高
            weighted_sum = 0.0
            weight_total = 0.0
            for i, score in enumerate(scores):
                weight = (i + 1) / len(scores)  # 越新权重越高
                weighted_sum += score * weight
                weight_total += weight
            category_mastery[cat] = weighted_sum / weight_total if weight_total > 0 else 0.0

        # 计算平均参与度
        engagement_scores = [r.engagement_score for r in completed_records if r.engagement_score > 0]
        avg_engagement = sum(engagement_scores) / len(engagement_scores) if engagement_scores else 0.5

        # 判断学习风格（基于时间分布和参与模式）
        total_records = len(records)
        avg_session_time = sum(r.time_spent_hours for r in records) / total_records if total_records else 0
        learning_style = self._infer_learning_style(records, avg_engagement)

        # 完课率
        completion_rate = len(completed_records) / total_records if total_records else 0.0

        # 平均测验分数
        all_scores = [s for r in completed_records for s in r.quiz_scores]
        avg_quiz = sum(all_scores) / len(all_scores) if all_scores else 0.0

        profile = StudentProfile(
            student_id=student_id,
            learning_records=records,
            learning_style=learning_style,
            weekly_available_hours=max(2.0, avg_session_time * 2),
            mastery_scores=category_mastery,
            average_quiz_score=avg_quiz,
            average_engagement=avg_engagement,
            completion_rate=completion_rate,
        )

        logger.info(
            f"学生画像分析完成: {student_id}, 完课率={completion_rate:.2f}, "
            f"平均分数={avg_quiz:.2f}, 参与度={avg_engagement:.2f}"
        )
        return profile

    def _infer_learning_style(self, records: list[LearningRecord], avg_engagement: float) -> LearningStyle:
        """根据学习行为推断学习风格。"""
        if not records:
            return LearningStyle.READING

        avg_time = sum(r.time_spent_hours for r in records) / len(records)
        score_variance = 0.0
        all_scores = [s for r in records for s in r.quiz_scores]
        if len(all_scores) > 1:
            mean = sum(all_scores) / len(all_scores)
            score_variance = sum((s - mean) ** 2 for s in all_scores) / len(all_scores)

        if avg_time < 1.0 and avg_engagement > 0.6:
            return LearningStyle.VISUAL  # 短时高效 = 视觉型
        elif avg_time > 2.0 and avg_engagement > 0.5:
            return LearningStyle.KINESTHETIC  # 长时实践 = 动手型
        elif score_variance < 0.02 and avg_engagement > 0.4:
            return LearningStyle.READING  # 稳定成绩 = 阅读型
        else:
            return LearningStyle.AUDITORY

    def identify_knowledge_gaps(self, profile: StudentProfile) -> list[KnowledgeGap]:
        """对比课程要求，识别知识差距。"""
        gaps: list[KnowledgeGap] = []
        completed_ids = {r.course_id for r in profile.learning_records if r.completed}

        for category, required_ids in self.curriculum_requirements.items():
            missing = []
            weak_areas = []

            for cid in required_ids:
                course = self.course_catalog.get(cid)
                if not course:
                    continue
                if cid not in completed_ids:
                    missing.append(cid)
                else:
                    # 已完成但掌握度不够
                    record = next(
                        (r for r in profile.learning_records if r.course_id == cid and r.completed), None
                    )
                    if record:
                        avg_score = (
                            sum(record.quiz_scores) / len(record.quiz_scores) if record.quiz_scores else 0.0
                        )
                        if avg_score < self.MASTERY_THRESHOLD:
                            weak_areas.append((cid, avg_score))

            # 类别掌握度低也算差距
            category_mastery = profile.mastery_scores.get(category, 0.0)
            if missing or category_mastery < self.MASTERY_THRESHOLD:
                severity = max(
                    len(missing) / max(len(required_ids), 1),
                    1.0 - category_mastery,
                )
                gap = KnowledgeGap(
                    category=category,
                    severity=min(severity, 1.0),
                    missing_courses=missing,
                    description=self._describe_gap(category, missing, weak_areas, category_mastery),
                )
                gaps.append(gap)

        # 按严重程度排序
        gaps.sort(key=lambda g: g.severity, reverse=True)
        logger.info(f"识别到 {len(gaps)} 个知识差距: {[g.category for g in gaps]}")
        return gaps

    def _describe_gap(
        self, category: str, missing: list[str], weak: list[tuple[str, float]], mastery: float
    ) -> str:
        """生成知识差距的描述。"""
        parts = [f"类别 [{category}] 掌握度: {mastery:.0%}"]
        if missing:
            parts.append(f"未完成课程: {', '.join(missing)}")
        if weak:
            weak_desc = ", ".join(f"{cid}({score:.0%})" for cid, score in weak)
            parts.append(f"掌握不足课程: {weak_desc}")
        return "; ".join(parts)

    def recommend_path(
        self,
        profile: StudentProfile,
        gaps: list[KnowledgeGap],
        target_completion_weeks: int = 12,
    ) -> LearningPath:
        """基于知识差距和学生画像推荐学习路径。"""
        if not gaps:
            logger.info(f"学生 {profile.student_id} 没有知识差距，无需推荐")
            return LearningPath(
                path_id=self._generate_path_id(),
                student_id=profile.student_id,
                status=PathStatus.COMPLETED,
            )

        # 收集所有需要学习的课程
        candidate_courses: list[tuple[Course, float]] = []  # (course, priority)
        completed_ids = {r.course_id for r in profile.learning_records if r.completed}

        for gap in gaps:
            for cid in gap.missing_courses:
                course = self.course_catalog.get(cid)
                if not course or cid in completed_ids:
                    continue
                # 检查前置课程是否已完成
                prereqs_met = all(p in completed_ids for p in course.prerequisites)
                if not prereqs_met:
                    # 将缺失的前置课程也加入候选
                    for prereq_id in course.prerequisites:
                        if prereq_id not in completed_ids:
                            prereq_course = self.course_catalog.get(prereq_id)
                            if prereq_course:
                                priority = gap.severity * 1.2  # 前置课程优先级略高
                                candidate_courses.append((prereq_course, priority))
                priority = gap.severity * self._difficulty_fitness(course, profile)
                candidate_courses.append((course, priority))

        # 去重
        seen: set[str] = set()
        unique_candidates: list[tuple[Course, float]] = []
        for course, priority in candidate_courses:
            if course.course_id not in seen:
                seen.add(course.course_id)
                unique_candidates.append((course, priority))

        # 按优先级排序，同时考虑难度递进
        unique_candidates.sort(key=lambda x: (
            self.DIFFICULTY_ORDER[x[0].difficulty],
            -x[1],
        ))

        # 限制路径长度
        weekly_hours = max(profile.weekly_available_hours, self.MIN_WEEKLY_HOURS)
        total_available_hours = weekly_hours * target_completion_weeks
        selected: list[tuple[Course, float]] = []
        accumulated_hours = 0.0

        for course, priority in unique_candidates:
            if len(selected) >= self.MAX_PATH_LENGTH:
                break
            if accumulated_hours + course.estimated_hours > total_available_hours * 1.1:
                continue  # 超出时间预算，跳过
            selected.append((course, priority))
            accumulated_hours += course.estimated_hours

        # 构建学习路径节点
        nodes: list[PathNode] = []
        for order, (course, priority) in enumerate(selected, start=1):
            reason = self._build_recommendation_reason(course, profile, gaps)
            nodes.append(PathNode(
                course_id=course.course_id,
                order=order,
                status=PathStatus.NOT_STARTED,
                recommended_reason=reason,
                priority=priority,
            ))

        target_date = datetime.now() + timedelta(weeks=target_completion_weeks)
        path = LearningPath(
            path_id=self._generate_path_id(),
            student_id=profile.student_id,
            nodes=nodes,
            total_estimated_hours=accumulated_hours,
            target_completion_date=target_date,
            status=PathStatus.NOT_STARTED if nodes else PathStatus.COMPLETED,
        )

        self._active_paths[path.path_id] = path
        logger.info(
            f"为学生 {profile.student_id} 推荐学习路径 {path.path_id}: "
            f"{len(nodes)} 门课程, 预计 {accumulated_hours:.1f} 小时"
        )
        return path

    def _difficulty_fitness(self, course: Course, profile: StudentProfile) -> float:
        """计算课程难度与学生水平的匹配度（0-1）。"""
        current_level = profile.average_quiz_score
        diff_level = self.DIFFICULTY_ORDER[course.difficulty] / max(
            len(self.DIFFICULTY_ORDER) - 1, 1
        )
        # 适度挑战：略高于当前水平时匹配度最高
        optimal_gap = 0.15
        actual_gap = diff_level - current_level
        fitness = 1.0 - abs(actual_gap - optimal_gap)
        return max(fitness, 0.1)

    def _build_recommendation_reason(
        self, course: Course, profile: StudentProfile, gaps: list[KnowledgeGap]
    ) -> str:
        """生成推荐理由。"""
        matching_gaps = [g for g in gaps if course.course_id in g.missing_courses or course.category == g.category]
        reasons = []
        if matching_gaps:
            reasons.append(f"填补 [{course.category}] 知识差距(严重度: {matching_gaps[0].severity:.0%})")
        if course.curriculum_required:
            reasons.append("课程体系必修")
        if course.difficulty == CourseDifficulty.BEGINNER and profile.average_quiz_score < 0.5:
            reasons.append("夯实基础")
        return "; ".join(reasons) if reasons else "补充学习"

    def track_progress(self, path_id: str, course_id: str, quiz_scores: list[float],
                       time_spent_hours: float, engagement: float) -> dict:
        """跟踪学习路径进度并返回进度报告。"""
        path = self._active_paths.get(path_id)
        if not path:
            raise ValueError(f"学习路径 {path_id} 不存在")

        node = next((n for n in path.nodes if n.course_id == course_id), None)
        if not node:
            raise ValueError(f"课程 {course_id} 不在路径 {path_id} 中")

        # 更新节点状态
        avg_score = sum(quiz_scores) / len(quiz_scores) if quiz_scores else 0.0
        is_completed = avg_score >= self.MASTERY_THRESHOLD and time_spent_hours > 0

        node.status = PathStatus.COMPLETED if is_completed else PathStatus.IN_PROGRESS
        path.updated_at = datetime.now()

        # 计算整体进度
        total_nodes = len(path.nodes)
        completed_nodes = sum(1 for n in path.nodes if n.status == PathStatus.COMPLETED)
        in_progress_nodes = sum(1 for n in path.nodes if n.status == PathStatus.IN_PROGRESS)
        progress_ratio = completed_nodes / total_nodes if total_nodes > 0 else 0.0

        # 预测完成时间
        elapsed_hours = sum(
            n.priority for n in path.nodes if n.status == PathStatus.COMPLETED
        )
        if completed_nodes > 0:
            avg_hours_per_course = elapsed_hours / completed_nodes
            remaining_courses = total_nodes - completed_nodes
            estimated_remaining_hours = avg_hours_per_course * remaining_courses
        else:
            estimated_remaining_hours = path.total_estimated_hours

        # 检查是否需要调整路径
        adjustments: list[str] = []
        if engagement < self.ENGAGEMENT_THRESHOLD:
            adjustments.append("参与度偏低，建议缩短单次学习时长或调整学习风格")
        if avg_score < 0.4 and node.status == PathStatus.IN_PROGRESS:
            adjustments.append("当前课程掌握不足，建议回顾前置知识或降低难度")
        if progress_ratio > 0.5 and in_progress_nodes == 0:
            # 过半完成但无进行中课程，可能半途而废
            adjustments.append("进度停滞，建议设置短期目标以恢复动力")

        # 更新路径整体状态
        if completed_nodes == total_nodes:
            path.status = PathStatus.COMPLETED
        elif completed_nodes > 0 or in_progress_nodes > 0:
            path.status = PathStatus.IN_PROGRESS

        report = {
            "path_id": path_id,
            "student_id": path.student_id,
            "current_course": course_id,
            "course_score": avg_score,
            "course_engagement": engagement,
            "overall_progress": f"{progress_ratio:.0%}",
            "completed_courses": completed_nodes,
            "total_courses": total_nodes,
            "estimated_remaining_hours": estimated_remaining_hours,
            "path_status": path.status.value,
            "adjustments": adjustments,
        }

        logger.info(f"路径进度更新: {path_id}, 进度={progress_ratio:.0%}, 课程={course_id}, 分数={avg_score:.2f}")
        return report
```

## 异常场景补充

### 场景：推荐路径不适合学生水平
```
trigger: 学生当前水平为初学者(平均测验分数0.3)，但推荐路径中包含多门高级课程，导致学生无法跟进而放弃学习
detection: 在track_progress中检测到连续3门课程的测验分数低于0.4且参与度持续下降；或首次课程分数即低于0.3
handling: 1) 立即暂停当前路径，标记为PAUSED状态；2) 重新分析学生画像，特别关注average_quiz_score和completion_rate；3) 在recommend_path中插入基础衔接课程，将难度梯度调整为每2门课难度提升一级而非跳级；4) 增加_difficulty_fitness中的optimal_gap从0.15降至0.08，确保推荐课程仅略高于当前水平；5) 在路径前部添加2-3门巩固型课程作为过渡
prevention: 1) recommend_path时强制校验：路径中不允许出现与学生当前水平差距超过2级的课程；2) 设置难度递进校验，确保相邻节点的难度差不超过1级；3) 对低完课率学生(<30%)自动在路径中增加check-point课程用于阶段性检测；4) 新路径生成后用模拟评分验证前3门课程的预期分数应在0.5-0.8之间
```

### 场景：学习路径过长导致放弃
```
trigger: 推荐路径包含10门以上课程，总时长超过学生6个月的可用学习时间，学生在完成前30%后逐渐降低学习频率直至放弃
detection: track_progress中检测到：1) 连续2周无课程进度更新；2) 参与度从0.7以上持续降至0.3以下；3) 距离上次更新时间超过预计单课程时长的3倍
handling: 1) 将当前路径状态标记为ABANDONED风险，发送提醒；2) 将长路径拆分为多个里程碑子路径(每段3-4门课)，每段设置独立目标和奖励；3) 重新计算路径，移除非核心课程，优先保留curriculum_required和severity最高的课程；4) 提供加速选项：对已掌握部分内容的课程允许跳过基础章节
prevention: 1) 硬性限制MAX_PATH_LENGTH为12门，超过时按severity排序截断；2) 推荐时计算路径总时长，确保不超过target_completion_weeks对应可用时长的1.1倍；3) 对weekly_available_hours低于5的学生，路径上限降至8门；4) 在路径中每3门课程设置一个milestone节点，完成milestone时给予正向反馈；5) 路径超过6门时必须包含至少1个"快速胜利"节点(estimated_hours<2且difficulty=BEGINNER)
```

## 在线教育考试防作弊完整实现

```python
class ExamAntiCheatService:
    """考试防作弊：浏览器锁定 → 行为监控 → 答案比对 → 完整性报告"""

    VIOLATION_WEIGHTS = {
        "tab_switch": {"weight": 5, "category": "major",
                       "description": "切换浏览器标签页"},
        "window_resize": {"weight": 2, "category": "minor",
                          "description": "窗口大小变化"},
        "mouse_leave": {"weight": 3, "category": "minor",
                        "description": "鼠标离开考试区域"},
        "copy_paste": {"weight": 8, "category": "major",
                       "description": "复制粘贴操作"},
        "right_click": {"weight": 3, "category": "minor",
                        "description": "右键操作"},
        "multiple_faces": {"weight": 10, "category": "critical",
                           "description": "检测到多张人脸"},
        "no_face": {"weight": 7, "category": "major",
                    "description": "未检测到人脸"},
        "screenshot": {"weight": 6, "category": "major",
                       "description": "截图操作"},
    }

    SUSPICION_THRESHOLD = {
        "auto_submit": 25,  # 自动提交阈值
        "flag_review": 10,  # 标记需人工审查阈值
    }

    def start_exam_session(self, exam_id, user_id):
        """创建考试会话"""
        session_id = str(uuid4())
        exam = self.db.get_exam(exam_id)

        # 1. 验证考试资格
        enrollment = self.db.query_one(
            "SELECT * FROM course_enrollments "
            "WHERE course_id = %s AND user_id = %s AND status = 'active'",
            exam["course_id"], user_id)

        if not enrollment:
            return {"status": "not_enrolled"}

        # 2. 检查是否已参加过
        existing_session = self.db.query_one(
            "SELECT * FROM exam_sessions "
            "WHERE exam_id = %s AND user_id = %s AND status IN ('completed', 'in_progress')",
            exam_id, user_id)

        if existing_session and not exam.get("allow_retake"):
            return {"status": "already_taken"}

        # 3. 随机化题目顺序
        questions = self.db.query(
            "SELECT * FROM exam_questions WHERE exam_id = %s "
            "ORDER BY question_order", exam_id)

        if exam.get("randomize_order"):
            import random
            random.shuffle(questions)
            question_order = [q["id"] for q in questions]
        else:
            question_order = [q["id"] for q in questions]

        # 4. 创建会话
        self.db.insert("exam_sessions", {
            "session_id": session_id,
            "exam_id": exam_id,
            "user_id": user_id,
            "question_order": json.dumps(question_order),
            "current_question_index": 0,
            "suspicion_score": 0,
            "violation_count": 0,
            "major_violation_count": 0,
            "browser_locked": True,
            "watermark_user_id": self._generate_watermark(user_id),
            "started_at": now(),
            "status": "in_progress"
        })

        # 5. 发送浏览器锁定指令
        self.websocket.send(user_id, {
            "type": "exam_start",
            "session_id": session_id,
            "lock_browser": True,
            "disable_features": ["copy", "paste", "right_click",
                                "screenshot", "tab_switch"],
            "watermark": self._generate_watermark(user_id),
            "webcam_required": exam.get("require_webcam", True)
        })

        return {"session_id": session_id, "exam_id": exam_id,
                "question_count": len(questions),
                "time_limit_minutes": exam.get("time_limit_minutes", 60)}

    def monitor_exam_behavior(self, session_id, event):
        """监控考试行为"""
        session = self.db.get_exam_session(session_id)

        if session["status"] != "in_progress":
            return {"status": "session_not_active"}

        violation_config = self.VIOLATION_WEIGHTS.get(event["type"])

        if not violation_config:
            return {"status": "unknown_event_type"}

        # 1. 记录违规
        self.db.insert("exam_violations", {
            "violation_id": str(uuid4()),
            "session_id": session_id,
            "user_id": session["user_id"],
            "event_type": event["type"],
            "category": violation_config["category"],
            "weight": violation_config["weight"],
            "details": json.dumps(event.get("details", {})),
            "timestamp": event.get("timestamp", now().isoformat()),
            "created_at": now()
        })

        # 2. 更新嫌疑分数
        new_suspicion = session["suspicion_score"] + violation_config["weight"]
        new_violation_count = session["violation_count"] + 1

        if violation_config["category"] in ["major", "critical"]:
            new_major = session["major_violation_count"] + 1
        else:
            new_major = session["major_violation_count"]

        self.db.update("exam_sessions",
            {"suspicion_score": new_suspicion,
             "violation_count": new_violation_count,
             "major_violation_count": new_major},
            {"session_id": session_id})

        # 3. 判断是否需要自动提交
        if new_suspicion >= self.SUSPICION_THRESHOLD["auto_submit"] or \
           new_major >= 3:
            self._force_submit(session_id, "suspicion_threshold_exceeded")

            return {"status": "auto_submitted",
                    "reason": "嫌疑分数超阈值或重大违规 ≥ 3 次",
                    "suspicion_score": new_suspicion}

        # 4. 判断是否需要标记
        if new_suspicion >= self.SUSPICION_THRESHOLD["flag_review"]:
            self._flag_for_review(session_id)

            return {"status": "flagged_for_review",
                    "suspicion_score": new_suspicion}

        # 5. 实时警告
        self.websocket.send(session["user_id"], {
            "type": "violation_warning",
            "event_type": event["type"],
            "description": violation_config["description"],
            "current_suspicion_score": new_suspicion
        })

        return {"status": "recorded", "suspicion_score": new_suspicion,
                "violation_type": event["type"]}

    def detect_answer_collusion(self, exam_id):
        """检测答案抄袭"""
        # 1. 获取所有完成的考试会话
        sessions = self.db.query(
            "SELECT * FROM exam_sessions "
            "WHERE exam_id = %s AND status = 'completed' "
            "ORDER BY completed_at", exam_id)

        if len(sessions) < 2:
            return {"status": "insufficient_data"}

        # 2. 计算答案相似度矩阵
        similarity_matrix = []

        for i in range(len(sessions)):
            for j in range(i + 1, len(sessions)):
                answers_i = self._get_session_answers(sessions[i]["session_id"])
                answers_j = self._get_session_answers(sessions[j]["session_id"])

                similarity = self._calculate_answer_similarity(answers_i, answers_j)

                similarity_matrix.append({
                    "session_a": sessions[i]["session_id"],
                    "user_a": sessions[i]["user_id"],
                    "session_b": sessions[j]["session_id"],
                    "user_b": sessions[j]["user_id"],
                    "similarity": similarity,
                    "flagged": similarity > 0.8
                })

        # 3. IP 地址关联
        flagged_pairs = [s for s in similarity_matrix if s["flagged"]]

        for pair in flagged_pairs:
            ip_a = self._get_session_ip(pair["session_a"])
            ip_b = self._get_session_ip(pair["session_b"])

            pair["same_subnet"] = self._is_same_subnet(ip_a, ip_b)
            pair["ip_a"] = ip_a
            pair["ip_b"] = ip_b

        # 4. 时间模式分析
        for pair in flagged_pairs:
            time_a = self._get_answer_times(pair["session_a"])
            time_b = self._get_answer_times(pair["session_b"])

            time_correlation = self._calculate_time_correlation(time_a, time_b)
            pair["time_correlation"] = time_correlation
            pair["timing_synchronized"] = time_correlation > 0.7

        # 5. 生成抄袭报告
        confirmed_cheating = [
            p for p in flagged_pairs
            if p.get("same_subnet") and p.get("timing_synchronized")
        ]

        return {
            "exam_id": exam_id,
            "total_sessions": len(sessions),
            "flagged_pairs": len(flagged_pairs),
            "confirmed_cheating_pairs": len(confirmed_cheating),
            "similarity_matrix": similarity_matrix,
            "confirmed_pairs": confirmed_cheating
        }

    def generate_integrity_report(self, exam_id, user_id):
        """生成诚信报告"""
        session = self.db.query_one(
            "SELECT * FROM exam_sessions "
            "WHERE exam_id = %s AND user_id = %s "
            "ORDER BY completed_at DESC LIMIT 1",
            exam_id, user_id)

        if not session:
            return {"status": "session_not_found"}

        # 1. 行为时间线
        violations = self.db.query(
            "SELECT * FROM exam_violations "
            "WHERE session_id = %s ORDER BY created_at",
            session["session_id"])

        timeline = [{
            "timestamp": v["created_at"].isoformat(),
            "event_type": v["event_type"],
            "category": v["category"],
            "weight": v["weight"],
            "description": self.VIOLATION_WEIGHTS.get(v["event_type"], {}).get("description", "")
        } for v in violations]

        # 2. 嫌疑分数分解
        score_breakdown = {
            "major_violations": sum(v["weight"] for v in violations if v["category"] == "major"),
            "minor_violations": sum(v["weight"] for v in violations if v["category"] == "minor"),
            "critical_violations": sum(v["weight"] for v in violations if v["category"] == "critical"),
            "total": session["suspicion_score"]
        }

        # 3. 答案原创性
        answer_originality = self.detect_answer_collusion(exam_id)
        user_similarity = [s for s in answer_originality.get("similarity_matrix", [])
                          if s["user_a"] == user_id or s["user_b"] == user_id]

        max_similarity = max((s["similarity"] for s in user_similarity), default=0)

        # 4. 与全班统计对比
        class_stats = self._get_class_statistics(exam_id)
        user_answers = self._get_session_answers(session["session_id"])

        # 5. 诚信评估
        if session["suspicion_score"] >= 25 or max_similarity > 0.85:
            integrity = "compromised"
        elif session["suspicion_score"] >= 10 or max_similarity > 0.75:
            integrity = "flagged"
        else:
            integrity = "clean"

        return {
            "session_id": session["session_id"],
            "user_id": user_id,
            "exam_id": exam_id,
            "integrity_assessment": integrity,
            "suspicion_score": session["suspicion_score"],
            "score_breakdown": score_breakdown,
            "violation_timeline": timeline,
            "max_answer_similarity": round(max_similarity, 3),
            "class_comparison": class_stats,
            "recommendation": self._get_integrity_recommendation(integrity)
        }

    def _get_session_answers(self, session_id):
        """获取会话答案"""
        answers = self.db.query(
            "SELECT * FROM exam_answers WHERE session_id = %s "
            "ORDER BY question_index", session_id)
        return {a["question_id"]: a["answer"] for a in answers}

    def _calculate_answer_similarity(self, answers_a, answers_b):
        """计算答案相似度"""
        common_questions = set(answers_a.keys()) & set(answers_b.keys())

        if not common_questions:
            return 0

        identical = 0
        for q_id in common_questions:
            if answers_a[q_id] == answers_b[q_id]:
                identical += 1

        return identical / len(common_questions)

    def _get_session_ip(self, session_id):
        """获取会话 IP"""
        session = self.db.get_exam_session(session_id)
        return session.get("ip_address", "")

    def _is_same_subnet(self, ip_a, ip_b):
        """判断是否同一子网"""
        if not ip_a or not ip_b:
            return False
        # 前 3 段相同 → 同子网
        return ip_a.split(".")[:3] == ip_b.split(".")[:3]

    def _calculate_time_correlation(self, times_a, times_b):
        """计算答题时间相关性"""
        if len(times_a) != len(times_b) or len(times_a) < 3:
            return 0

        # 计算答题间隔的相关性
        intervals_a = [times_a[i+1] - times_a[i] for i in range(len(times_a)-1)]
        intervals_b = [times_b[i+1] - times_b[i] for i in range(len(times_b)-1)]

        # Pearson 相关系数
        n = len(intervals_a)
        mean_a = sum(intervals_a) / n
        mean_b = sum(intervals_b) / n

        cov = sum((a - mean_a) * (b - mean_b) for a, b in zip(intervals_a, intervals_b)) / n
        std_a = (sum((a - mean_a)**2 for a in intervals_a) / n) ** 0.5
        std_b = (sum((b - mean_b)**2 for b in intervals_b) / n) ** 0.5

        if std_a == 0 or std_b == 0:
            return 0

        return cov / (std_a * std_b)

    def _get_answer_times(self, session_id):
        """获取答题时间序列"""
        answers = self.db.query(
            "SELECT answered_at FROM exam_answers "
            "WHERE session_id = %s ORDER BY question_index",
            session_id)
        return [a["answered_at"].timestamp() for a in answers if a["answered_at"]]

    def _get_class_statistics(self, exam_id):
        """获取全班统计"""
        return self.db.query_one(
            "SELECT AVG(score) as avg_score, "
            "STDDEV(score) as std_score, "
            "COUNT(*) as total_students "
            "FROM exam_sessions "
            "WHERE exam_id = %s AND status = 'completed'", exam_id)

    def _generate_watermark(self, user_id):
        """生成水印"""
        return hashlib.sha256(f"{user_id}:{now().timestamp()}".encode()).hexdigest()[:12]

    def _force_submit(self, session_id, reason):
        """强制提交"""
        self.db.update("exam_sessions",
            {"status": "force_submitted",
             "force_submit_reason": reason,
             "completed_at": now()},
            {"session_id": session_id})

        session = self.db.get_exam_session(session_id)
        self.websocket.send(session["user_id"], {
            "type": "exam_force_submitted",
            "reason": reason,
            "suspicion_score": session["suspicion_score"]
        })

    def _flag_for_review(self, session_id):
        """标记需人工审查"""
        self.db.update("exam_sessions",
            {"needs_review": True},
            {"session_id": session_id})

    def _get_integrity_recommendation(self, integrity):
        """获取诚信建议"""
        recommendations = {
            "clean": "成绩有效，无异常行为",
            "flagged": "成绩暂缓公布，需人工审查",
            "compromised": "成绩作废，需学术委员会裁决"
        }
        return recommendations.get(integrity, "需进一步审查")
```

## 异常场景补充

### 场景：防作弊系统误判导致考试中断

```
触发：用户眨眼频率高 → 人脸检测频繁判定"无人脸" → 嫌疑分数飙升 → 自动提交 → 考试中断
检测：
  1. 用户申诉被强制提交 → 可能误判
  2. "无人脸"判定后用户立即提交申诉 → 系统误判
处理：
  1. 人脸检测增加容错（连续 3 秒无人脸才计违规）
  2. 眨眼不计入"无人脸"
  3. 强制提交后提供快速申诉通道
预防：容错机制 + 眨眼排除 + 快速申诉
```

### 场景：考试期间网络波动触发异常检测

```
触发：考试期间网络断开 30 秒 → WebSocket 重连 → 系统判定为"标签切换" → 误判作弊
检测：
  1. 断网后重连被判定为违规 → 网络问题误判
  2. 同一时间多用户出现"标签切换" → 网络波动而非作弊
处理：
  1. 区分主动切换与被动断线（断线无鼠标事件）
  2. 网络波动不计入嫌疑分数
  3. 断线期间暂停计时
预防：区分断线 + 不计嫌疑 + 暂停计时
```

### 场景：新型作弊手段绕过检测

```
触发：考生使用物理遮挡（手机放在桌子下方）→ 人脸检测正常 → 摄像头看不到 → 绕过检测
检测：
  1. 答题速度异常快 → 可能有外部辅助
  2. 答案与网上题库高度一致 → 查题作弊
处理：
  1. 增加"视线追踪"检测（视线不在屏幕区域 → 标记）
  2. 答题时间与题目难度不匹配 → 异常
  3. 与题库答案比对检测
预防：视线追踪 + 时间-难度比对 + 题库比对
```
