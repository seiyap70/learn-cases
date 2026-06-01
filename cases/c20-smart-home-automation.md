# C20: 智能家居的设备联动与规则引擎

## 业务场景

某智能家居平台，接入 1000+ 种设备（灯光、空调、窗帘、门锁、传感器等），500 万用户设置了自动化规则。核心诉求："当我到家时，自动开灯+开空调+关窗帘"。

**已知数据：**
- 接入设备数：1000 万台
- 用户数：500 万
- 日均规则触发：2000 万次
- 设备控制延迟：< 2 秒
- 规则类型：条件触发、定时触发、手动触发
- 设备协议：WiFi、Zigbee、BLE、Matter

**为什么是难题？**

核心矛盾是**规则组合爆炸与冲突**：
- 10 个设备之间可能有 90 种联动组合
- 两条规则可能矛盾："温度>28°C 开空调" vs "离家模式关所有设备"
- 设备离线时规则仍触发 → 上线后如何补执行？
- 规则延迟叠加：传感器检测 → 云端判断 → 下发指令 → 设备执行，全链路 1-3 秒

**一个典型的用户痛点：**

用户设置了两条规则：
1. 温度 > 28°C → 开空调 26°C
2. 离家模式 → 关所有设备

当他离家时室温 > 28°C，两条规则同时触发 → 设备收到矛盾指令（开空调 + 关空调）→ 行为不可预测。

## 核心挑战

### 挑战 1：规则冲突检测与解决

两条规则对同一设备产生矛盾操作（一条说开灯，另一条说关灯），如何检测并自动解决？

### 挑战 2：设备状态同步延迟

用户点击"关灯"→ 云端处理 → 发送 MQTT 指令 → 设备执行 → 状态上报 → 云端更新。全链路 1-3 秒。期间用户看到"灯开着"但实际已关 → 反复操作。

### 挑战 3：离线设备的处理

设备离线时规则仍触发 → 设备上线后补执行还是丢弃？定时规则（"每天7:00开灯"）应该补执行；条件规则（"温度>28开空调"）可能不需要了（温度已降）。

### 挑战 4：规则数量与性能

500 万用户 × 平均 5 条规则 = 2500 万条规则。每个传感器事件需要评估所有相关规则 → 性能瓶颈。

## 设计约束

- 设备控制延迟 < 2 秒
- 规则评估延迟 < 500ms
- 离线设备上线后 30 秒内同步
- 规则变更即时生效

## 数据库设计

核心表结构支撑设备管理、规则定义、状态日志和冲突审计四大领域。

### devices 表

```sql
CREATE TABLE devices (
    device_id         VARCHAR(64)  NOT NULL COMMENT '设备唯一标识，如 zigbee-thermo-001',
    user_id           BIGINT       NOT NULL COMMENT '所属用户',
    home_id           BIGINT       NOT NULL COMMENT '所属家庭',
    room_id           BIGINT       NOT NULL COMMENT '所属房间',
    device_type       VARCHAR(32)  NOT NULL COMMENT '设备类型: light/thermostat/curtain/lock/sensor',
    protocol          VARCHAR(16)  NOT NULL COMMENT '通信协议: wifi/zigbee/ble/matter',
    firmware_version  VARCHAR(32)  DEFAULT NULL COMMENT '固件版本号',
    status            TINYINT      NOT NULL DEFAULT 1 COMMENT '1=在线 2=离线 3=维护中',
    last_heartbeat    DATETIME     DEFAULT NULL COMMENT '最近一次心跳时间',
    created_at        DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at        DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (device_id),
    INDEX idx_user_home (user_id, home_id),
    INDEX idx_room (room_id),
    INDEX idx_status (status),
    INDEX idx_heartbeat (last_heartbeat)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='设备注册表';
```

### rules 表

```sql
CREATE TABLE rules (
    rule_id           BIGINT       NOT NULL AUTO_INCREMENT,
    user_id           BIGINT       NOT NULL COMMENT '规则所有者',
    home_id           BIGINT       NOT NULL COMMENT '所属家庭',
    rule_name         VARCHAR(128) NOT NULL COMMENT '规则名称，如"离家关灯"',
    rule_type         TINYINT      NOT NULL COMMENT '1=条件触发 2=定时触发 3=手动触发',
    priority          INT          NOT NULL DEFAULT 0 COMMENT '优先级：数字越大越优先',
    scope             VARCHAR(16)  NOT NULL DEFAULT 'home' COMMENT 'home/room/device',
    enabled           TINYINT      NOT NULL DEFAULT 1 COMMENT '0=禁用 1=启用',
    schedule_cron     VARCHAR(64)  DEFAULT NULL COMMENT 'Cron 表达式，定时规则专用',
    preconditions     JSON         DEFAULT NULL COMMENT '前置条件 JSON，如 [{"property":"anyone_home","value":true}]',
    cooldown_seconds  INT          NOT NULL DEFAULT 0 COMMENT '规则冷却时间，防止频繁触发',
    max_daily_triggers INT         NOT NULL DEFAULT 0 COMMENT '每日最大触发次数，0=不限',
    created_at        DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at        DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (rule_id),
    INDEX idx_user (user_id),
    INDEX idx_home (home_id),
    INDEX idx_enabled_type (enabled, rule_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='自动化规则表';
```

### rule_conditions 表

```sql
CREATE TABLE rule_conditions (
    condition_id      BIGINT       NOT NULL AUTO_INCREMENT,
    rule_id           BIGINT       NOT NULL COMMENT '关联规则',
    device_id         VARCHAR(64)  NOT NULL COMMENT '触发设备',
    property          VARCHAR(64)  NOT NULL COMMENT '设备属性: temperature/door_state/motion',
    operator          VARCHAR(16)  NOT NULL COMMENT 'GT/LT/EQ/NE/GTE/LTE/CHANGED/BETWEEN',
    threshold_value   VARCHAR(128) NOT NULL COMMENT '阈值，BETWEEN 类型存 JSON 如 "[25,30]"',
    duration_seconds  INT          NOT NULL DEFAULT 0 COMMENT '条件持续时长，0=立即触发',
    logic_order       INT          NOT NULL DEFAULT 0 COMMENT '条件组合顺序，用于 AND 组合',
    PRIMARY KEY (condition_id),
    INDEX idx_rule (rule_id),
    INDEX idx_device (device_id),
    INDEX idx_device_property (device_id, property)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='规则条件表';
```

### rule_actions 表

```sql
CREATE TABLE rule_actions (
    action_id         BIGINT       NOT NULL AUTO_INCREMENT,
    rule_id           BIGINT       NOT NULL COMMENT '关联规则',
    device_id         VARCHAR(64)  NOT NULL COMMENT '目标设备',
    command           VARCHAR(64)  NOT NULL COMMENT '指令: turn_on/turn_off/set_temperature/set_color',
    params            JSON         DEFAULT NULL COMMENT '指令参数，如 {"temperature":26,"mode":"cool"}',
    delay_seconds     INT          NOT NULL DEFAULT 0 COMMENT '动作延迟执行（秒）',
    execution_order   INT          NOT NULL DEFAULT 0 COMMENT '同一规则内动作执行顺序',
    PRIMARY KEY (action_id),
    INDEX idx_rule (rule_id),
    INDEX idx_device (device_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='规则动作表';
```

### device_state_log 表

```sql
CREATE TABLE device_state_log (
    log_id            BIGINT       NOT NULL AUTO_INCREMENT,
    device_id         VARCHAR(64)  NOT NULL,
    property          VARCHAR(64)  NOT NULL COMMENT '变更属性',
    old_value         VARCHAR(256) DEFAULT NULL,
    new_value         VARCHAR(256) NOT NULL,
    change_source     VARCHAR(16)  NOT NULL COMMENT '来源: user/rule/device/schedule',
    trigger_rule_id   BIGINT       DEFAULT NULL COMMENT '规则触发的变更，记录规则ID',
    request_id        VARCHAR(64)  DEFAULT NULL COMMENT '请求追踪ID',
    created_at        DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (log_id),
    INDEX idx_device_time (device_id, created_at),
    INDEX idx_rule (trigger_rule_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='设备状态变更日志'
PARTITION BY RANGE (TO_DAYS(created_at)) (
    PARTITION p_202601 VALUES LESS THAN (TO_DAYS('2026-02-01')),
    PARTITION p_202602 VALUES LESS THAN (TO_DAYS('2026-03-01')),
    PARTITION p_202603 VALUES LESS THAN (TO_DAYS('2026-04-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

### conflict_log 表

```sql
CREATE TABLE conflict_log (
    conflict_id       BIGINT       NOT NULL AUTO_INCREMENT,
    home_id           BIGINT       NOT NULL COMMENT '冲突发生的家庭',
    device_id         VARCHAR(64)  NOT NULL COMMENT '冲突设备',
    winner_rule_id    BIGINT       NOT NULL,
    loser_rule_ids    JSON         NOT NULL COMMENT '被覆盖的规则ID列表',
    winner_action     JSON         NOT NULL COMMENT '胜出动作详情',
    loser_actions     JSON         NOT NULL COMMENT '被覆盖动作详情',
    winner_priority   INT          NOT NULL,
    loser_priorities  JSON         NOT NULL COMMENT '被覆盖规则优先级列表',
    resolved_at       DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (conflict_id),
    INDEX idx_home_time (home_id, resolved_at),
    INDEX idx_device (device_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='规则冲突审计日志';
```

**设计要点：**
- `rule_conditions` 与 `rule_actions` 拆分存储，支持一条规则多条件 AND 组合、多动作顺序执行
- `device_state_log` 按月分区，保留 180 天热数据，历史数据归档至对象存储
- `conflict_log` 为审计表，用于用户投诉回溯和规则优先级调优参考
- `rules.cooldown_seconds` 防止传感器频繁抖动导致规则风暴（如温湿度传感器每秒上报）
- `rules.max_daily_triggers` 限制每日触发上限，避免规则逻辑错误导致设备反复开关

## 请先独立思考（限时 30 分钟）

1. 规则冲突如何检测？按什么优先级解决？"离家模式"和"温控"冲突时谁赢？
2. 设备状态同步的 1-3 秒延迟如何缓解？乐观更新 vs 确认回执？
3. 2500 万条规则如何高效评估？事件驱动 vs 定时轮询？
4. 离线设备的指令如何队列化？补执行的判断逻辑？

---

## 设计解析

### 规则引擎：条件-动作模型 + 优先级冲突解决

```python
class Rule:
    def __init__(self, rule_id, conditions, actions, priority=0, scope="home"):
        self.rule_id = rule_id
        self.conditions = conditions    # 触发条件列表
        self.actions = actions          # 执行动作列表
        self.priority = priority        # 优先级（数字越大越优先）
        self.scope = scope              # 作用域：home / room / device

class Condition:
    def __init__(self, device_id, property, operator, value):
        self.device_id = device_id
        self.property = property        # temperature / motion / door_state
        self.operator = operator        # GT / LT / EQ / CHANGED
        self.value = value

    def evaluate(self, event):
        if event.device_id != self.device_id:
            return False
        actual = event.properties.get(self.property)
        if self.operator == "GT":   return actual > self.value
        if self.operator == "LT":   return actual < self.value
        if self.operator == "EQ":   return actual == self.value
        if self.operator == "CHANGED": return True
        return False

class Action:
    def __init__(self, device_id, command, params=None):
        self.device_id = device_id
        self.command = command          # turn_on / turn_off / set_temperature
        self.params = params or {}
```

### 规则评估：事件驱动 + 索引

**关键优化：不是每条事件都评估 2500 万条规则，而是用设备 ID 做索引。**

```python
class RuleEngine:
    def __init__(self):
        self.rules = {}                        # rule_id → Rule
        self.device_rule_index = defaultdict(set)  # device_id → set of rule_ids

    def add_rule(self, rule):
        """添加规则并建立索引"""
        self.rules[rule.rule_id] = rule
        for condition in rule.conditions:
            self.device_rule_index[condition.device_id].add(rule.rule_id)

    def on_event(self, event):
        """设备事件触发规则评估"""
        # 1. 通过索引快速找到相关规则（而非全量扫描）
        related_rule_ids = self.device_rule_index.get(event.device_id, set())
        
        # 2. 评估相关规则的条件
        triggered = []
        for rule_id in related_rule_ids:
            rule = self.rules[rule_id]
            if all(c.evaluate(event) for c in rule.conditions):
                triggered.append(rule)

        if not triggered:
            return []

        # 3. 冲突检测与解决
        resolved = self.conflict_resolver.resolve(triggered)

        # 4. 执行动作
        results = []
        for rule in resolved:
            for action in rule.actions:
                result = self.device_controller.execute(action)
                results.append(result)

        return results
```

**性能分析：**

| 维度 | 全量扫描 | 索引查找 |
|------|---------|---------|
| 事件评估规则数 | 2500 万 | 平均 5 条（同设备规则） |
| 评估延迟 | > 1 秒 | < 10ms |
| 内存占用 | 低（只有规则） | 中（额外索引） |

索引将评估从 O(全部规则) 降到 O(同设备规则) → 延迟从秒级降到毫秒级。

### 冲突解决：同一设备取最高优先级

```python
class ConflictResolver:
    def __init__(self, conflict_log_repo=None):
        self.conflict_log_repo = conflict_log_repo

    def resolve(self, triggered_rules):
        """检测并解决规则冲突"""
        # 按设备分组：同一设备的所有动作
        actions_by_device = defaultdict(list)
        for rule in triggered_rules:
            for action in rule.actions:
                actions_by_device[action.device_id].append((rule, action))

        resolved = []
        conflicts_log = []

        for device_id, rule_actions in actions_by_device.items():
            if len(rule_actions) == 1:
                resolved.append(rule_actions[0])
            else:
                # 检测是否为真正的冲突：同设备同属性的矛盾操作
                command_conflicts = self._detect_command_conflicts(rule_actions)

                if not command_conflicts:
                    # 不同属性不冲突（如一条设温度、一条设风速）→ 全部执行
                    for ra in rule_actions:
                        resolved.append(ra)
                else:
                    # 真正冲突：按规则优先级选最高的
                    rule_actions.sort(key=lambda x: x[0].priority, reverse=True)
                    winner = rule_actions[0]
                    resolved.append(winner)

                    # 记录被覆盖的规则（审计用）
                    for rule, action in rule_actions[1:]:
                        conflicts_log.append({
                            "winner_rule": winner[0].rule_id,
                            "loser_rule": rule.rule_id,
                            "device_id": device_id,
                            "winner_priority": winner[0].priority,
                            "loser_priority": rule.priority
                        })

        # 异步写入冲突日志
        self.log_conflicts(conflicts_log)
        return [r for r, _ in resolved]

    def _detect_command_conflicts(self, rule_actions):
        """检测同一设备上的命令是否真正冲突
        例如 turn_on 和 turn_off 对同一设备是冲突的，
        但 set_temperature=26 和 set_fan_speed=high 不冲突
        """
        CONFLICT_PAIRS = {
            frozenset({"turn_on", "turn_off"}),
            frozenset({"open", "close"}),
            frozenset({"lock", "unlock"}),
        }
        commands = [action.command for _, action in rule_actions]
        for i in range(len(commands)):
            for j in range(i + 1, len(commands)):
                pair = frozenset({commands[i], commands[j]})
                if pair in CONFLICT_PAIRS:
                    return True
        # 同一命令不同参数也算冲突（set_temperature=26 vs set_temperature=22）
        command_params = {}
        for _, action in rule_actions:
            if action.command in command_params:
                if command_params[action.command] != action.params:
                    return True
            command_params[action.command] = action.params
        return False

    def log_conflicts(self, conflicts):
        """异步写入冲突日志到数据库"""
        if not conflicts or not self.conflict_log_repo:
            return
        # 使用后台线程异步写入，避免阻塞规则执行链路
        threading.Thread(
            target=self.conflict_log_repo.batch_insert,
            args=(conflicts,),
            daemon=True
        ).start()
```

**优先级设计：**

```
场景模式（离家/回家/睡眠）→ priority = 100
安全规则（烟雾报警→开窗）→ priority = 90
舒适规则（温度控制）     → priority = 50
便利规则（自动关灯）     → priority = 30
```

**优先级冲突解决的完整流程：**

```
事件到达 → 索引查找相关规则 → 评估条件 → 筛选触发规则
    ↓
按设备分组动作 → 检测命令冲突
    ↓
┌─ 无冲突：全部执行
└─ 有冲突：取最高优先级规则执行，其余记录冲突日志
    ↓
执行动作 → 乐观更新状态 → 推送App
```

**冲突示例与解决：**

```
规则A（priority=50）：温度>28°C → 开空调 26°C
规则B（priority=100）：离家模式 → 关所有设备

用户离家时温度>28°C → 两条规则同时触发
→ B 优先级更高 → 关空调（离家模式覆盖温控）
→ 冲突日志：规则B 覆盖 规则A，设备：空调

用户在家时温度>28°C → 只有 A 触发 → 开空调

规则C（priority=50）：温度>28°C → 开空调 26°C
规则D（priority=50）：湿度>80% → 开空调 除湿模式

两条规则同优先级且动作冲突（同设备不同模式）
→ 先到先得（按触发时间排序）→ 记录冲突日志，可推送用户选择
```

### 设备状态同步：乐观更新 + 确认回执

```python
class DeviceController:
    def __init__(self, mqtt_bridge, state_cache, notification_service):
        self.mqtt = mqtt_bridge
        self.state_cache = state_cache
        self.notification = notification_service
        self.pending_acks = {}    # request_id → (device_id, action, timestamp)
        self.retry_counts = {}    # request_id → retry count

    def execute(self, action):
        """发送设备指令并乐观更新状态"""
        # 1. 乐观更新：指令发出后立即更新缓存（不等设备确认）
        self.state_cache.update(action.device_id, {
            action.command: action.params
        })
        # 推送给用户 App（让用户立即看到状态变更）
        self.push_state_to_app(action.device_id)

        # 2. 发送 MQTT 指令
        request_id = uuid4()
        self.mqtt.publish(f"cmd/{action.device_id}", json.dumps({
            "command": action.command,
            "params": action.params,
            "request_id": request_id,
            "timestamp": now_ms()
        }))

        # 3. 设置确认超时（5秒）
        self.schedule_ack_check(action.device_id, request_id, timeout=5)

        return {"status": "sent", "device_id": action.device_id}

    def on_device_ack(self, device_id, request_id, actual_state):
        """设备确认执行成功"""
        # 用设备上报的真实状态覆盖乐观更新
        self.state_cache.update(device_id, actual_state)
        self.push_state_to_app(device_id)  # 更新 App 显示
        self.cancel_ack_check(device_id, request_id)
        # 清除重试计数
        self.retry_counts.pop(request_id, None)

    def on_ack_timeout(self, device_id, request_id):
        """设备未确认 → 状态可能不准，标记为 stale"""
        retry_count = self.retry_counts.get(request_id, 0)

        if retry_count < 2:
            # 后台重试（最多2次）
            self.retry_counts[request_id] = retry_count + 1
            action = self.pending_acks.get(request_id)
            if action:
                self.mqtt.publish(f"cmd/{action.device_id}", json.dumps({
                    "command": action.command,
                    "params": action.params,
                    "request_id": request_id,
                    "timestamp": now_ms(),
                    "retry": True
                }))
                self.schedule_ack_check(device_id, request_id, timeout=5)
        else:
            # 超过重试上限 → 标记 stale，通知用户
            self.state_cache.mark_stale(device_id)
            self.notify_user(device_id, "设备可能未响应，请检查")
            self.pending_acks.pop(request_id, None)
            self.retry_counts.pop(request_id, None)

    def push_state_to_app(self, device_id):
        """通过 WebSocket 推送状态变更到用户 App"""
        state = self.state_cache.get(device_id)
        user_ids = self.get_device_owners(device_id)
        for uid in user_ids:
            self.notification.send(uid, {
                "type": "device_state_update",
                "device_id": device_id,
                "state": state,
                "timestamp": now_ms()
            })
```

**乐观更新的好处与风险：**

| 方案 | 用户感知延迟 | 数据一致性 | 复杂度 |
|------|------------|-----------|-------|
| 等设备确认 | 1-3 秒 | 强一致 | 低 |
| 乐观更新 | < 100ms | 最终一致 | 中 |
| 乐观更新+回滚 | < 100ms | 最终一致+回滚 | 高 |

选择"乐观更新"：用户感知延迟从 1-3 秒降到 < 100ms，牺牲的是极少数设备离线时的状态不一致（通过 stale 标记和超时提示解决）。

**乐观更新回滚机制：**

```python
class StateCache:
    """设备状态缓存，支持乐观更新与回滚"""

    def __init__(self, redis_client):
        self.redis = redis_client
        self.optimistic_updates = {}  # device_id → {"original": state, "updated_at": ts}

    def update(self, device_id, new_state):
        """乐观更新：保存原始状态以支持回滚"""
        current = self.redis.get(f"device_state:{device_id}")
        if current:
            self.optimistic_updates[device_id] = {
                "original": json.loads(current),
                "updated_at": time.time()
            }
        # 更新缓存
        self.redis.set(f"device_state:{device_id}", json.dumps(new_state), ex=3600)

    def rollback(self, device_id):
        """回滚到原始状态（设备确认失败时）"""
        saved = self.optimistic_updates.get(device_id)
        if saved:
            self.redis.set(
                f"device_state:{device_id}",
                json.dumps(saved["original"]),
                ex=3600
            )
            self.optimistic_updates.pop(device_id, None)
            return True
        return False

    def confirm(self, device_id, actual_state):
        """设备确认 → 用真实状态覆盖并清除回滚记录"""
        self.redis.set(f"device_state:{device_id}", json.dumps(actual_state), ex=3600)
        self.optimistic_updates.pop(device_id, None)

    def mark_stale(self, device_id):
        """标记状态不可信"""
        self.redis.set(f"device_stale:{device_id}", "1", ex=300)  # 5分钟过期

    def is_stale(self, device_id):
        return self.redis.exists(f"device_stale:{device_id}")

    def get(self, device_id):
        data = self.redis.get(f"device_state:{device_id}")
        return json.loads(data) if data else None
```

### 设备影子（Device Shadow）：期望状态与上报状态的双向同步

设备影子是 IoT 领域的核心模式，源自 AWS IoT Device Shadow。它解耦了"用户期望设备处于什么状态"与"设备实际上报了什么状态"之间的鸿沟，尤其在设备离线、网络不稳定时价值显著。

**核心概念：**

```
┌─────────────────────────────────────────────────────────┐
│                    Device Shadow                         │
│                                                         │
│  desired (期望状态)          reported (上报状态)          │
│  ┌──────────────┐           ┌──────────────┐            │
│  │ power: on    │           │ power: off   │            │
│  │ temp: 26     │  delta    │ temp: 28     │            │
│  │ mode: cool   │ ────────→ │ mode: fan    │            │
│  └──────────────┘           └──────────────┘            │
│         ↑                         ↑                     │
│    用户/App 设置             设备实际上报                  │
│                                                         │
│  metadata: { version: 5, last_desired: ts,              │
│              last_reported: ts, last_delta: ts }         │
└─────────────────────────────────────────────────────────┘
```

- **desired**：用户/规则引擎期望设备达到的目标状态
- **reported**：设备实际上报的当前状态
- **delta**：desired 与 reported 的差异，设备需要据此执行操作以对齐到期望状态

**冲突场景与解决策略：**

| 冲突类型 | 示例 | 解决策略 |
|---------|------|---------|
| 设备上报与期望一致 | desired.power=on, reported.power=on | 无 delta，无需操作 |
| 设备上报与期望不同 | desired.power=on, reported.power=off | 产生 delta，下发指令 |
| 设备上报非期望的新状态 | desired.temp=26, reported.temp=22 | 设备用户手动调了温度，需判断是否覆盖 |
| 期望状态过期 | desired.power=on（2小时前设置），设备刚上线 | 检查时效性，过期则清除 desired |

```python
import hashlib
from enum import Enum
from dataclasses import dataclass, field
from typing import Optional, Dict, Any


class ShadowConflictPolicy(Enum):
    """设备影子冲突解决策略"""
    DESIRED_WINS = "desired_wins"      # 期望状态优先（强制覆盖设备）
    REPORTED_WINS = "reported_wins"    # 上报状态优先（尊重设备实际）
    NEWER_WINS = "newer_wins"          # 时间戳更新的优先
    MERGE = "merge"                    # 属性级合并，相同属性取 desired


@dataclass
class ShadowMetadata:
    """影子元数据：版本、时间戳，用于乐观锁和冲突检测"""
    version: int = 0
    last_desired_update: Optional[float] = None   # desired 最后更新时间
    last_reported_update: Optional[float] = None   # reported 最后更新时间
    last_delta_computed: Optional[float] = None    # delta 最后计算时间


class DeviceShadowService:
    """完整的设备影子服务：desired/reported 双向同步、delta 计算、冲突解决"""

    # 默认配置
    DESIRED_TTL_SECONDS = 86400 * 7     # desired 状态 7 天过期
    DELTA_RETRY_MAX = 3                  # delta 同步最大重试次数
    DELTA_RETRY_INTERVAL = 5             # delta 重试间隔（秒）
    DESIRED_EXPIRY_SECONDS = 7200        # desired 状态 2 小时后视为过期

    def __init__(self, redis_client, mqtt_bridge, device_controller,
                 conflict_policy=ShadowConflictPolicy.DESIRED_WINS):
        self.redis = redis_client
        self.mqtt = mqtt_bridge
        self.device_controller = device_controller
        self.conflict_policy = conflict_policy

        # Redis Key 模板
        self.KEY_DESIRED = "shadow:desired:{device_id}"
        self.KEY_REPORTED = "shadow:reported:{device_id}"
        self.KEY_METADATA = "shadow:meta:{device_id}"
        self.KEY_DELTA_RETRY = "shadow:delta_retry:{device_id}"

    # ─────────────── desired 状态操作 ───────────────

    def update_desired(self, device_id: str, desired_state: Dict[str, Any],
                       source: str = "user") -> Dict:
        """更新期望状态（用户/规则引擎调用）

        Args:
            device_id: 设备ID
            desired_state: 期望状态字典，如 {"power": "on", "temperature": 26}
            source: 来源标识 user/rule/schedule

        Returns:
            更新结果，包含新版本号和产生的 delta
        """
        # 1. 获取当前元数据（乐观锁）
        meta = self._get_metadata(device_id)
        new_version = meta.version + 1

        # 2. 合并 desired：部分更新而非全量替换
        current_desired = self._get_desired(device_id)
        merged_desired = {**current_desired, **desired_state}

        # 3. 写入 desired + 更新元数据（Redis 事务保证原子性）
        pipe = self.redis.pipeline()
        pipe.set(
            self.KEY_DESIRED.format(device_id=device_id),
            json.dumps(merged_desired),
            ex=self.DESIRED_TTL_SECONDS
        )
        meta.version = new_version
        meta.last_desired_update = time.time()
        pipe.set(
            self.KEY_METADATA.format(device_id=device_id),
            json.dumps(asdict(meta)),
            ex=self.DESIRED_TTL_SECONDS
        )
        pipe.execute()

        # 4. 计算 delta 并下发
        reported = self._get_reported(device_id)
        delta = self._compute_delta(merged_desired, reported)

        if delta:
            self._publish_delta(device_id, delta, new_version)
            # 启动同步重试
            self._schedule_delta_retry(device_id, delta, new_version)

        # 5. 审计日志
        logging.info(
            f"Shadow desired updated: device={device_id} version={new_version} "
            f"source={source} delta={delta}"
        )

        return {
            "device_id": device_id,
            "version": new_version,
            "desired": merged_desired,
            "delta": delta,
            "source": source
        }

    # ─────────────── reported 状态操作 ───────────────

    def update_reported(self, device_id: str, reported_state: Dict[str, Any],
                        source: str = "device") -> Dict:
        """更新设备上报状态（设备通过 MQTT 上报时调用）

        Args:
            device_id: 设备ID
            reported_state: 设备实际上报的状态
            source: 来源标识 device/sync

        Returns:
            更新结果，包含是否与 desired 存在偏差
        """
        # 1. 获取当前元数据
        meta = self._get_metadata(device_id)
        new_version = meta.version + 1

        # 2. 合并 reported
        current_reported = self._get_reported(device_id)
        merged_reported = {**current_reported, **reported_state}

        # 3. 检查是否与 desired 存在冲突
        desired = self._get_desired(device_id)
        conflict = self._detect_desired_reported_conflict(desired, merged_reported)

        if conflict:
            resolved_reported = self._resolve_conflict(
                device_id, desired, merged_reported, conflict
            )
            merged_reported = resolved_reported

        # 4. 写入 reported + 更新元数据
        pipe = self.redis.pipeline()
        pipe.set(
            self.KEY_REPORTED.format(device_id=device_id),
            json.dumps(merged_reported),
            ex=self.DESIRED_TTL_SECONDS
        )
        meta.version = new_version
        meta.last_reported_update = time.time()
        pipe.set(
            self.KEY_METADATA.format(device_id=device_id),
            json.dumps(asdict(meta)),
            ex=self.DESIRED_TTL_SECONDS
        )
        pipe.execute()

        # 5. 重新计算 delta（上报后 delta 可能变化）
        delta = self._compute_delta(desired, merged_reported)

        if delta:
            # 仍有偏差 → 可能需要再次下发
            logging.info(
                f"Shadow delta remains after reported update: "
                f"device={device_id} delta={delta}"
            )
            self._publish_delta(device_id, delta, new_version)
        else:
            # desired 与 reported 完全一致 → delta 消除
            logging.info(f"Shadow delta resolved: device={device_id}")
            self._cancel_delta_retry(device_id)

        return {
            "device_id": device_id,
            "version": new_version,
            "reported": merged_reported,
            "delta": delta,
            "conflict": conflict
        }

    # ─────────────── delta 计算 ───────────────

    def _compute_delta(self, desired: Dict, reported: Dict) -> Dict[str, Any]:
        """计算 desired 与 reported 之间的差异

        只返回 desired 中存在但 reported 中不同或缺失的属性。
        reported 中多出的属性不计入 delta（设备额外状态不强制同步）。

        示例:
            desired  = {"power": "on", "temp": 26, "mode": "cool"}
            reported = {"power": "off", "temp": 26, "mode": "fan", "fan_speed": "high"}
            delta    = {"power": "on", "mode": "cool"}  # fan_speed 不在 delta 中
        """
        delta = {}
        for key, desired_value in desired.items():
            reported_value = reported.get(key)
            if reported_value != desired_value:
                delta[key] = desired_value
        return delta

    # ─────────────── 冲突检测与解决 ───────────────

    def _detect_desired_reported_conflict(self, desired: Dict,
                                          reported: Dict) -> Optional[Dict]:
        """检测 desired 与 reported 之间的冲突

        冲突定义：同一属性 desired 和 reported 都有值，但值不同
        且 reported 的更新时间比 desired 更新（设备在 desired 设置后
        又被手动改变了）
        """
        conflicts = {}
        for key, desired_val in desired.items():
            reported_val = reported.get(key)
            if reported_val is not None and reported_val != desired_val:
                conflicts[key] = {
                    "desired": desired_val,
                    "reported": reported_val
                }
        return conflicts if conflicts else None

    def _resolve_conflict(self, device_id: str, desired: Dict,
                          reported: Dict, conflict: Dict) -> Dict:
        """根据冲突策略解决 desired 与 reported 的冲突

        Args:
            conflict: 冲突属性字典 {key: {desired: val, reported: val}}

        Returns:
            解决后的 reported 状态
        """
        resolved = dict(reported)

        if self.conflict_policy == ShadowConflictPolicy.DESIRED_WINS:
            # 期望状态优先：强制覆盖设备上报
            for key in conflict:
                resolved[key] = desired.get(key)

        elif self.conflict_policy == ShadowConflictPolicy.REPORTED_WINS:
            # 上报状态优先：尊重设备实际状态，清除 desired 中冲突的属性
            for key in conflict:
                # 更新 desired 使其与 reported 一致
                if key in desired:
                    desired[key] = reported.get(key)
            self.redis.set(
                self.KEY_DESIRED.format(device_id=device_id),
                json.dumps(desired),
                ex=self.DESIRED_TTL_SECONDS
            )

        elif self.conflict_policy == ShadowConflictPolicy.NEWER_WINS:
            # 时间戳更新的优先
            meta = self._get_metadata(device_id)
            if meta.last_desired_update and meta.last_reported_update:
                if meta.last_desired_update > meta.last_reported_update:
                    for key in conflict:
                        resolved[key] = desired.get(key)
                else:
                    for key in conflict:
                        desired[key] = reported.get(key)
                    self.redis.set(
                        self.KEY_DESIRED.format(device_id=device_id),
                        json.dumps(desired),
                        ex=self.DESIRED_TTL_SECONDS
                    )

        elif self.conflict_policy == ShadowConflictPolicy.MERGE:
            # 属性级合并：相同属性取 desired，额外属性保留
            for key in conflict:
                resolved[key] = desired.get(key)

        return resolved

    # ─────────────── delta 下发与同步重试 ───────────────

    def _publish_delta(self, device_id: str, delta: Dict, version: int):
        """下发 delta 到设备"""
        self.mqtt.publish_to_device(
            device_id=device_id,
            command_type="shadow_delta",
            payload={
                "type": "shadow_delta",
                "delta": delta,
                "version": version,
                "timestamp": int(time.time() * 1000)
            },
            qos=1
        )

    def _schedule_delta_retry(self, device_id: str, delta: Dict, version: int):
        """调度 delta 同步重试

        设备可能因网络抖动未收到 delta，需要重试。
        使用 Redis 的过期机制实现简单定时器。
        """
        retry_key = self.KEY_DELTA_RETRY.format(device_id=device_id)
        retry_info = {
            "delta": delta,
            "version": version,
            "attempts": 0,
            "max_attempts": self.DELTA_RETRY_MAX,
            "interval": self.DELTA_RETRY_INTERVAL,
            "scheduled_at": time.time()
        }
        self.redis.set(
            retry_key,
            json.dumps(retry_info),
            ex=self.DELTA_RETRY_INTERVAL * self.DELTA_RETRY_MAX + 60
        )

    def _cancel_delta_retry(self, device_id: str):
        """delta 已同步，取消重试"""
        retry_key = self.KEY_DELTA_RETRY.format(device_id=device_id)
        self.redis.delete(retry_key)

    async def process_delta_retries(self):
        """定时扫描并执行 delta 重试（由后台任务每 5 秒调用一次）

        重试策略：
        - 第 1 次重试：5 秒后
        - 第 2 次重试：10 秒后
        - 第 3 次重试：20 秒后（指数退避）
        - 超过 3 次：标记设备 stale，通知用户
        """
        pattern = "shadow:delta_retry:*"
        for key in self.redis.scan_iter(match=pattern):
            try:
                data = json.loads(self.redis.get(key))
                elapsed = time.time() - data["scheduled_at"]
                interval = data["interval"] * (2 ** data["attempts"])  # 指数退避

                if elapsed < interval:
                    continue  # 未到重试时间

                if data["attempts"] >= data["max_attempts"]:
                    # 超过最大重试次数
                    device_id = key.split(":")[-1]
                    logging.warning(
                        f"Shadow delta sync failed after {data['max_attempts']} "
                        f"retries for device {device_id}"
                    )
                    self.state_cache.mark_stale(device_id)
                    self.redis.delete(key)
                    continue

                # 执行重试
                device_id = key.split(":")[-1]
                self._publish_delta(device_id, data["delta"], data["version"])
                data["attempts"] += 1
                data["scheduled_at"] = time.time()
                self.redis.set(key, json.dumps(data), ex=300)

            except Exception as e:
                logging.error(f"Error processing delta retry for key {key}: {e}")

    # ─────────────── desired 过期清理 ───────────────

    def check_desired_expiry(self, device_id: str) -> bool:
        """检查 desired 状态是否已过期

        场景：用户 2 小时前设置 desired.power=on，设备一直离线。
        如果设备现在上线，是否还应执行该指令？
        策略：超过 DESIRED_EXPIRY_SECONDS 的 desired 属性视为过期，应清除。
        """
        meta = self._get_metadata(device_id)
        if not meta.last_desired_update:
            return True  # 无 desired 记录，视为过期

        elapsed = time.time() - meta.last_desired_update
        if elapsed > self.DESIRED_EXPIRY_SECONDS:
            # desired 已过期，清除
            logging.info(
                f"Shadow desired expired for device {device_id}, "
                f"age={elapsed:.0f}s, clearing"
            )
            self.redis.delete(self.KEY_DESIRED.format(device_id=device_id))
            return True
        return False

    # ─────────────── 辅助方法 ───────────────

    def _get_desired(self, device_id: str) -> Dict:
        data = self.redis.get(self.KEY_DESIRED.format(device_id=device_id))
        return json.loads(data) if data else {}

    def _get_reported(self, device_id: str) -> Dict:
        data = self.redis.get(self.KEY_REPORTED.format(device_id=device_id))
        return json.loads(data) if data else {}

    def _get_metadata(self, device_id: str) -> ShadowMetadata:
        data = self.redis.get(self.KEY_METADATA.format(device_id=device_id))
        if data:
            d = json.loads(data)
            return ShadowMetadata(**d)
        return ShadowMetadata()

    def get_full_shadow(self, device_id: str) -> Dict:
        """获取设备影子完整信息（调试/展示用）"""
        desired = self._get_desired(device_id)
        reported = self._get_reported(device_id)
        meta = self._get_metadata(device_id)
        delta = self._compute_delta(desired, reported)

        return {
            "device_id": device_id,
            "desired": desired,
            "reported": reported,
            "delta": delta,
            "metadata": asdict(meta)
        }
```

**设备影子完整生命周期：**

```
用户点击"开灯" → update_desired(device, {power: on})
    ↓
计算 delta: {power: on}（reported.power=off）
    ↓
下发 delta 到设备（MQTT QoS 1）
    ↓
启动同步重试定时器（5s/10s/20s 指数退避）
    ↓
┌─ 设备收到并执行 → 上报 reported.power=on
│       ↓
│   delta 变为空 → 同步完成 → 取消重试定时器
│
└─ 设备未响应 → 5s 后重试 → 10s 后重试 → 20s 后重试
        ↓
    超过 3 次 → 标记 stale → 通知用户"设备可能未响应"
```

**desired 与 reported 冲突的典型场景：**

```
时间线：
T1: 用户设置 desired.temp=26（通过 App）
T2: 家人手动将空调调到 22°C → 设备上报 reported.temp=22
T3: 影子检测到冲突：desired=26 vs reported=22

解决策略：
- DESIRED_WINS：重新下发 temp=26（强制覆盖手动操作）
- REPORTED_WINS：更新 desired.temp=22（尊重手动操作）
- NEWER_WINS：比较 T1 和 T2 时间戳，新的优先
- MERGE：同 DESIRED_WINS，但保留设备额外属性

推荐策略：REPORTED_WINS（物理操作优先于远程指令）
原因：用户在设备上的手动操作通常代表即时意图，远程规则可能已过时
```

### 离线设备：指令队列 + 条件重评估

```python
class OfflineCommandQueue:
    def execute(self, action):
        if self.is_online(action.device_id):
            return self.device_controller.execute(action)
        else:
            # 设备离线 → 放入队列
            self.redis.lpush(
                f"cmd_queue:{action.device_id}",
                json.dumps({
                    "action": action.__dict__,
                    "enqueued_at": now().isoformat(),
                    "rule_id": action.rule_id,
                    "rule_type": self.get_rule_type(action.rule_id)
                })
            )
            # 最多保留 50 条指令（避免队列无限增长）
            self.redis.ltrim(f"cmd_queue:{action.device_id}", 0, 49)
            return {"status": "queued"}

    def on_device_online(self, device_id):
        """设备上线 → 处理积压指令"""
        commands = self.redis.lrange(f"cmd_queue:{device_id}", 0, -1)
        
        executed = 0
        skipped = 0
        
        for cmd_json in reversed(commands):  # 从最早到最晚
            cmd = json.loads(cmd_json)
            rule_type = cmd.get("rule_type")
            
            if rule_type == "scheduled":
                # 定时规则（"每天7:00开灯"）→ 补执行
                self.device_controller.execute(Action(**cmd["action"]))
                executed += 1
            elif rule_type == "conditional":
                # 条件规则（"温度>28开空调"）→ 重新评估条件
                if self.should_still_execute(cmd):
                    self.device_controller.execute(Action(**cmd["action"]))
                    executed += 1
                else:
                    skipped += 1
            else:
                # 未知类型 → 丢弃（安全优先）
                skipped += 1
        
        self.redis.delete(f"cmd_queue:{device_id}")
        
        return {"executed": executed, "skipped": skipped}

    def should_still_execute(self, cmd):
        """重新评估条件规则是否仍需执行"""
        # 获取设备当前状态
        current_state = self.state_cache.get(cmd["action"]["device_id"])
        if not current_state:
            return False  # 无法评估 → 不执行（安全优先）

        # 检查条件是否仍然满足
        # 如"温度>28开空调" → 检查当前温度是否仍>28
        condition = self.get_rule_condition(cmd["rule_id"])
        return condition.evaluate_current(current_state)

    def cleanup_expired_queues(self):
        """定时清理过期指令队列（每小时执行）"""
        # 扫描所有设备队列，清除超过24小时的积压指令
        cutoff = now() - timedelta(hours=24)
        pattern = "cmd_queue:*"
        for key in self.redis.scan_iter(match=pattern):
            device_id = key.split(":")[1]
            # 只保留24小时内的指令
            commands = self.redis.lrange(key, 0, -1)
            valid = []
            for cmd_json in commands:
                cmd = json.loads(cmd_json)
                enqueued = datetime.fromisoformat(cmd["enqueued_at"])
                if enqueued > cutoff:
                    valid.append(cmd_json)
                # 定时规则：超过一个执行周期也丢弃
                if cmd.get("rule_type") == "scheduled":
                    rule = self.get_rule(cmd["rule_id"])
                    if rule and rule.schedule_cron:
                        next_run = self.get_next_cron_time(rule.schedule_cron)
                        if enqueued < next_run - timedelta(hours=1):
                            continue  # 过期的定时指令
            # 替换队列
            self.redis.delete(key)
            if valid:
                self.redis.rpush(key, *valid)
```

**离线指令处理策略总结：**

| 规则类型 | 设备上线后策略 | 原因 |
|---------|-------------|------|
| 定时规则（每天7:00开灯） | 补执行最近一次 | 用户明确期望定时执行，错过也应补上 |
| 条件规则（温度>28开空调） | 重新评估当前条件 | 条件可能已不满足（温度已降），无需执行 |
| 手动规则（用户点击开灯） | 丢弃并通知用户 | 用户意图是即时操作，延迟执行无意义 |
| 安全规则（烟雾报警开窗） | 立即执行+告警 | 安全优先，无论条件是否仍满足 |
```

### 定时触发规则：时区感知与夏令时处理

```python
import asyncio
from croniter import croniter
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger
from apscheduler.triggers.date import DateTrigger
from pytz import timezone, utc, UnknownTimeZoneError
from enum import Enum


class MisfirePolicy(Enum):
    """误触发策略：调度器因系统负载等原因错过执行时间时的处理方式"""
    FIRE_NOW = "fire_now"       # 立即补执行（适用于重要的定时任务，如"每天7:00开灯"）
    SKIP = "skip"               # 跳过本次执行（适用于周期性任务，如"每30分钟检查温度"）
    FIRE_NEXT = "fire_next"     # 等待下一次调度时间执行（折中方案）


class ScheduledRuleExecutor:
    """完整的定时规则执行器：Cron 调度 + 时区处理 + 夏令时 + 误触发策略"""

    # 全局时区配置：用户家庭可能分布在不同时区
    # 中国全境 UTC+8 无夏令时；欧美用户需处理 DST 切换
    DEFAULT_TIMEZONE = "Asia/Shanghai"

    # DST 切换期间的 Cron 规则修正
    # 例如：Cron 设置 2:30 AM，但 DST 春季前进导致 2:00→3:00，2:30 不存在
    DST_FALLBACK_HOUR = 1  # 回退方案：不存在的时刻改为 1:00 执行

    def __init__(self, rule_repo, device_controller, state_cache,
                 conflict_resolver=None):
        self.rule_repo = rule_repo
        self.device_controller = device_controller
        self.state_cache = state_cache
        self.conflict_resolver = conflict_resolver

        # 调度器配置：协程合并、单实例保证
        self.scheduler = AsyncIOScheduler(
            job_defaults={
                "coalesce": True,       # 合并同一规则的堆积任务
                "max_instances": 1,     # 同一规则同一时刻最多 1 个实例
                "misfire_grace_time": None  # 由我们自行控制，不使用 APScheduler 默认策略
            },
            timezone=utc  # 调度器内部使用 UTC，规则按时区转换
        )
        self.job_map = {}       # rule_id → {"job_id": str, "timezone": str, "misfire_policy": str}
        self._running = False

        # 误触发处理线程：独立扫描错过的规则执行
        self._misfire_check_interval = 10  # 每 10 秒检查一次

    async def start(self):
        """启动调度器，加载所有启用的定时规则"""
        self.scheduler.start()
        self._running = True

        # 加载所有启用的定时规则
        rules = await self.rule_repo.find_by_type_and_enabled(
            rule_type="scheduled", enabled=True
        )
        loaded = 0
        for rule in rules:
            if self._add_job(rule):
                loaded += 1

        # 启动误触发检查后台任务
        asyncio.create_task(self._misfire_checker_loop())

        logging.info(
            f"ScheduledRuleExecutor started, loaded {loaded}/{len(rules)} rules"
        )

    async def stop(self):
        """优雅停止"""
        self._running = False
        self.scheduler.shutdown(wait=False)

    # ─────────────── 时区处理 ───────────────

    def _get_rule_timezone(self, rule) -> str:
        """获取规则关联的时区

        时区来源优先级：
        1. 规则自身的 timezone 字段（用户手动设置）
        2. 用户家庭所在时区（根据用户注册地址推断）
        3. 默认时区 Asia/Shanghai
        """
        if hasattr(rule, 'timezone') and rule.timezone:
            try:
                timezone(rule.timezone)  # 验证时区有效性
                return rule.timezone
            except UnknownTimeZoneError:
                logging.warning(
                    f"Invalid timezone {rule.timezone} for rule {rule.rule_id}, "
                    f"falling back to home timezone"
                )

        # 从家庭设置获取时区
        home_tz = self._get_home_timezone(rule.home_id)
        return home_tz or self.DEFAULT_TIMEZONE

    def _get_home_timezone(self, home_id: int) -> str:
        """获取家庭所在时区（从缓存或数据库查询）"""
        # 实际场景中根据用户注册地址或 GPS 定位推断
        # 此处简化为缓存查询
        return self.home_timezone_cache.get(home_id, self.DEFAULT_TIMEZONE)

    def _convert_cron_to_utc(self, cron_expr: str, tz_name: str) -> str:
        """将用户本地时区的 Cron 表达式转换为 UTC

        示例：
        - 用户设置 "0 7 * * *"（北京时间 7:00）→ UTC "0 23 * * *"（前一天 23:00）
        - 用户设置 "0 7 * * *"（纽约时间 7:00）→ UTC "0 12 * * *"（冬令时）
                                                  → UTC "0 11 * * *"（夏令时）

        注意：固定偏移的转换无法正确处理 DST，需要动态判断。
        因此本方法仅用于初始化，实际触发时间由 APScheduler 的时区感知调度处理。
        """
        tz = timezone(tz_name)
        # 使用 croniter 在指定时区解析下一个执行时间
        local_now = datetime.now(tz)
        cron = croniter(cron_expr, local_now)
        next_local = cron.get_next(datetime)
        next_utc = next_local.astimezone(utc)
        return next_utc

    # ─────────────── 夏令时处理 ───────────────

    def _check_dst_transition(self, rule, tz_name: str) -> Optional[Dict]:
        """检测规则执行时间是否受夏令时切换影响

        DST 问题场景：
        1. 春季前进（Spring Forward）：2:00 AM → 3:00 AM
           Cron "30 2 * * *" 指向的时刻 2:30 AM 不存在 → 需要修正
        2. 秋季后退（Fall Back）：2:00 AM → 1:00 AM
           Cron "30 1 * * *" 指向的时刻 1:30 AM 出现两次 → 可能重复执行

        Returns:
            None = 不受影响
            Dict = DST 影响详情及修正建议
        """
        tz = timezone(tz_name)

        # 检查时区是否有 DST（中国、日本等没有 DST）
        if tz.dst(datetime.now(tz)) == timedelta(0):
            # 再检查明天是否也是 0，如果变化则说明在过渡期
            tomorrow = datetime.now(tz) + timedelta(days=1)
            if tz.dst(tomorrow) == timedelta(0):
                return None  # 无 DST 的时区，不受影响

        # 解析 Cron 表达式中的小时
        cron_parts = rule.schedule_cron.split()
        if len(cron_parts) < 2:
            return None

        try:
            cron_hour = int(cron_parts[1])
        except ValueError:
            # 复杂表达式（如范围、步进），跳过 DST 检查
            return None

        # 检查 Cron 时刻在 DST 切换日是否有效
        local_now = datetime.now(tz)
        # 构造今天的 Cron 目标时刻
        try:
            target_time = local_now.replace(
                hour=cron_hour,
                minute=int(cron_parts[0]),
                second=0,
                microsecond=0
            )
            # 尝试本地化：如果该时刻不存在（Spring Forward），会抛异常
            localized = tz.localize(target_time)
        except Exception:
            # 该时刻不存在（Spring Forward 场景）
            return {
                "type": "spring_forward",
                "original_hour": cron_hour,
                "recommended_hour": self.DST_FALLBACK_HOUR,
                "message": (
                    f"规则 {rule.rule_id} 的执行时间 {cron_hour}:00 在夏令时切换日"
                    f"不存在，将自动调整为 {self.DST_FALLBACK_HOUR}:00 执行"
                )
            }

        # 检查是否有歧义（Fall Back 场景：同一时刻出现两次）
        try:
            # ambiguous=True 表示该时刻有歧义
            tz.localize(target_time, is_dst=None)
        except Exception:
            return {
                "type": "fall_back",
                "original_hour": cron_hour,
                "message": (
                    f"规则 {rule.rule_id} 的执行时间 {cron_hour}:00 在夏令时切换日"
                    f"存在两次，将使用夏令时时刻（is_dst=True）执行一次"
                )
            }

        return None

    def _handle_dst_adjustment(self, rule, tz_name: str):
        """根据 DST 检测结果调整调度

        策略：
        - Spring Forward：调整到前一小时的整点执行
        - Fall Back：使用 is_dst=True 消除歧义，只执行一次
        """
        dst_info = self._check_dst_transition(rule, tz_name)
        if not dst_info:
            return rule.schedule_cron  # 无需调整

        if dst_info["type"] == "spring_forward":
            # 调整 Cron 的小时部分
            cron_parts = rule.schedule_cron.split()
            cron_parts[1] = str(dst_info["recommended_hour"])
            adjusted_cron = " ".join(cron_parts)
            logging.warning(
                f"DST Spring Forward: adjusted rule {rule.rule_id} "
                f"cron from {rule.schedule_cron} to {adjusted_cron}"
            )
            return adjusted_cron

        return rule.schedule_cron

    # ─────────────── 误触发策略 ───────────────

    def _get_misfire_policy(self, rule) -> MisfirePolicy:
        """获取规则的误触发策略

        策略选择逻辑：
        - 重要定时规则（如"每天7:00开灯"）→ FIRE_NOW：补执行
        - 高频周期规则（如"每30分钟检查温度"）→ SKIP：错过就等下一次
        - 用户手动规则 → FIRE_NEXT：折中方案
        """
        if hasattr(rule, 'misfire_policy') and rule.misfire_policy:
            return MisfirePolicy(rule.misfire_policy)

        # 根据规则特征自动判断
        cron_parts = rule.schedule_cron.split() if rule.schedule_cron else []
        if len(cron_parts) >= 1 and cron_parts[0].startswith("*/"):
            # 周期性高频规则 → 跳过
            return MisfirePolicy.SKIP

        if rule.priority >= 80:
            # 高优先级规则（如安全相关）→ 立即补执行
            return MisfirePolicy.FIRE_NOW

        # 默认：等待下一次
        return MisfirePolicy.FIRE_NEXT

    async def _misfire_checker_loop(self):
        """后台任务：定期检查错过的规则执行

        场景：调度器因 GC 停顿、CPU 飙升等原因错过执行时间。
        本循环每 10 秒扫描一次，检测是否有规则被错过。
        """
        while self._running:
            try:
                await asyncio.sleep(self._misfire_check_interval)
                await self._check_misfires()
            except asyncio.CancelledError:
                break
            except Exception as e:
                logging.error(f"Misfire checker error: {e}")

    async def _check_misfires(self):
        """扫描所有已调度的规则，检测是否有错过执行的"""
        now_utc = datetime.now(utc)

        for rule_id, job_info in list(self.job_map.items()):
            try:
                rule = await self.rule_repo.get_by_id(rule_id)
                if not rule or not rule.enabled:
                    continue

                tz_name = job_info.get("timezone", self.DEFAULT_TIMEZONE)
                tz = timezone(tz_name)
                local_now = now_utc.astimezone(tz)

                # 计算规则上一次应该执行的时间
                cron = croniter(rule.schedule_cron, local_now)
                last_scheduled = cron.get_prev(datetime)

                # 检查上一次执行是否被错过
                last_actual = await self._get_last_trigger_time(rule_id)
                if last_actual:
                    last_actual_local = last_actual.astimezone(tz)
                    # 如果上次实际执行时间早于上次调度时间超过 60 秒
                    # 说明该次调度被错过了
                    gap = (last_scheduled - last_actual_local).total_seconds()
                    if gap > 60:
                        await self._handle_misfire(rule, last_scheduled, job_info)
                else:
                    # 从未有执行记录，检查是否应该已经执行过
                    if last_scheduled.date() == local_now.date():
                        # 今天应该执行但未执行
                        await self._handle_misfire(rule, last_scheduled, job_info)

            except Exception as e:
                logging.error(f"Error checking misfire for rule {rule_id}: {e}")

    async def _handle_misfire(self, rule, missed_time, job_info):
        """处理误触发的规则

        Args:
            rule: 被错过的规则
            missed_time: 应该执行但错过的时间（本地时区）
            job_info: 调度信息
        """
        policy = self._get_misfire_policy(rule)

        if policy == MisfirePolicy.FIRE_NOW:
            # 立即补执行
            logging.warning(
                f"Misfire detected for rule {rule.rule_id}, "
                f"missed={missed_time}, policy=FIRE_NOW, executing immediately"
            )
            await self._execute_rule(rule, is_misfire=True)

        elif policy == MisfirePolicy.SKIP:
            # 跳过，等下一次调度
            logging.info(
                f"Misfire detected for rule {rule.rule_id}, "
                f"missed={missed_time}, policy=SKIP, waiting for next schedule"
            )
            # 记录跳过日志（用户可查看）
            await self._record_misfire(rule.rule_id, missed_time, "skipped")

        elif policy == MisfirePolicy.FIRE_NEXT:
            # 等待下一次调度时间
            # 如果距下一次调度 < 5 分钟，则不补执行；否则补执行
            tz_name = job_info.get("timezone", self.DEFAULT_TIMEZONE)
            tz = timezone(tz_name)
            local_now = datetime.now(tz)
            cron = croniter(rule.schedule_cron, local_now)
            next_scheduled = cron.get_next(datetime)
            time_to_next = (next_scheduled - local_now).total_seconds()

            if time_to_next < 300:  # 5 分钟内
                logging.info(
                    f"Misfire for rule {rule.rule_id}, next run in {time_to_next:.0f}s, "
                    f"policy=FIRE_NEXT, skipping misfire"
                )
                await self._record_misfire(rule.rule_id, missed_time, "deferred")
            else:
                logging.warning(
                    f"Misfire for rule {rule.rule_id}, next run in {time_to_next:.0f}s, "
                    f"policy=FIRE_NEXT, executing now as next run is far away"
                )
                await self._execute_rule(rule, is_misfire=True)

    # ─────────────── 规则调度管理 ───────────────

    def _add_job(self, rule) -> bool:
        """将规则添加到调度器（含时区和 DST 处理）"""
        if not rule.schedule_cron:
            logging.warning(f"Rule {rule.rule_id} has no cron expression, skipping")
            return False

        try:
            # 1. 获取规则时区
            tz_name = self._get_rule_timezone(rule)
            tz = timezone(tz_name)

            # 2. 检查并处理 DST 影响
            cron_expr = self._handle_dst_adjustment(rule, tz_name)

            # 3. 创建时区感知的 Cron 触发器
            trigger = CronTrigger.from_crontab(
                cron_expr,
                timezone=tz  # APScheduler 原生支持时区感知调度
            )

            # 4. 获取误触发策略
            misfire_policy = self._get_misfire_policy(rule)

            # 5. 添加调度任务
            job = self.scheduler.add_job(
                self._execute_rule,
                trigger=trigger,
                id=f"rule_{rule.rule_id}",
                args=[rule],
                kwargs={"is_misfire": False},
                replace_existing=True,
                misfire_grace_time=None,  # 由我们自行管理误触发
            )

            self.job_map[rule.rule_id] = {
                "job_id": job.id,
                "timezone": tz_name,
                "misfire_policy": misfire_policy.value,
                "cron_expr": cron_expr,
                "added_at": time.time()
            }

            logging.info(
                f"Scheduled rule {rule.rule_id}: cron={cron_expr} "
                f"tz={tz_name} misfire_policy={misfire_policy.value}"
            )
            return True

        except Exception as e:
            logging.error(f"Failed to schedule rule {rule.rule_id}: {e}")
            return False

    async def on_rule_created(self, rule):
        """规则新增/修改时动态更新调度"""
        if rule.rule_id in self.job_map:
            self._remove_job(rule.rule_id)
        if rule.enabled and rule.schedule_cron:
            self._add_job(rule)

    async def on_rule_deleted(self, rule_id):
        """规则删除时移除调度"""
        self._remove_job(rule_id)

    async def on_timezone_changed(self, home_id: int, new_timezone: str):
        """用户家庭时区变更时，重新调度该家庭所有定时规则

        场景：用户从北京搬到纽约，需要将所有定时规则从 UTC+8 切换到 UTC-5
        """
        try:
            timezone(new_timezone)  # 验证新时区
        except UnknownTimeZoneError:
            logging.error(f"Invalid timezone: {new_timezone}")
            return

        # 找到该家庭的所有定时规则
        rules = await self.rule_repo.find_by_home_and_type(
            home_id=home_id, rule_type="scheduled"
        )

        for rule in rules:
            if rule.rule_id in self.job_map:
                self._remove_job(rule.rule_id)
                # 更新规则的时区设置
                rule.timezone = new_timezone
                self._add_job(rule)
                logging.info(
                    f"Rescheduled rule {rule.rule_id} for timezone change "
                    f"to {new_timezone}"
                )

    def _remove_job(self, rule_id):
        job_info = self.job_map.pop(rule_id, None)
        if job_info:
            try:
                self.scheduler.remove_job(job_info["job_id"])
            except Exception:
                pass

    # ─────────────── 规则执行 ───────────────

    async def _execute_rule(self, rule, is_misfire: bool = False):
        """定时触发规则的完整执行流程

        Args:
            rule: 要执行的规则
            is_misfire: 是否为误触发补执行
        """
        start_time = time.monotonic()
        execution_source = "misfire" if is_misfire else "scheduled"
        logging.info(
            f"Executing {execution_source} rule {rule.rule_id}: {rule.rule_name}"
        )

        try:
            # 1. 检查每日触发次数上限
            if rule.max_daily_triggers > 0:
                today_count = await self._get_today_trigger_count(rule.rule_id)
                if today_count >= rule.max_daily_triggers:
                    logging.info(
                        f"Rule {rule.rule_id} reached daily limit {rule.max_daily_triggers}"
                    )
                    return

            # 2. 检查冷却时间
            if rule.cooldown_seconds > 0:
                last_trigger = await self._get_last_trigger_time(rule.rule_id)
                if last_trigger and (now() - last_trigger).total_seconds() < rule.cooldown_seconds:
                    logging.info(f"Rule {rule.rule_id} in cooldown period")
                    return

            # 3. 检查前置条件（如"如果有人在家"）
            if rule.preconditions:
                for cond in rule.preconditions:
                    if not await self._evaluate_precondition(cond):
                        logging.info(
                            f"Rule {rule.rule_id} precondition not met: {cond}"
                        )
                        return

            # 4. 冲突检测：与同一设备的其他活跃规则比较
            if self.conflict_resolver:
                # 检查是否有更高优先级的规则最近触发了同一设备的动作
                conflict_check = await self.conflict_resolver.check_scheduled_conflict(
                    rule, execution_source
                )
                if conflict_check.get("blocked"):
                    logging.info(
                        f"Rule {rule.rule_id} blocked by higher priority rule "
                        f"{conflict_check.get('blocking_rule_id')}"
                    )
                    return

            # 5. 按顺序执行动作（支持延迟动作）
            for action in sorted(rule.actions, key=lambda a: a.execution_order):
                if action.delay_seconds > 0:
                    await asyncio.sleep(action.delay_seconds)
                result = await self.device_controller.execute(action)
                logging.info(
                    f"Rule {rule.rule_id} action executed: "
                    f"device={action.device_id} cmd={action.command} result={result}"
                )

            # 6. 记录触发日志
            await self._record_trigger(rule.rule_id, source=execution_source)
            elapsed = (time.monotonic() - start_time) * 1000
            logging.info(
                f"Rule {rule.rule_id} ({execution_source}) executed in {elapsed:.1f}ms"
            )

        except asyncio.TimeoutError:
            logging.error(f"Rule {rule.rule_id} execution timeout")
            await self._notify_rule_failure(rule, "执行超时")
        except Exception as e:
            logging.error(f"Rule {rule.rule_id} execution failed: {e}")
            await self._notify_rule_failure(rule, str(e))

    async def _evaluate_precondition(self, cond):
        """评估前置条件，如"有人在家"需查询家庭成员定位状态"""
        if cond["property"] == "anyone_home":
            members = await self._get_home_members_presence(cond["home_id"])
            return any(m["is_home"] for m in members) == cond["value"]
        elif cond["property"] == "device_online":
            return not self.state_cache.is_stale(cond["device_id"])
        elif cond["property"] == "time_range":
            # 如 {"property":"time_range","value":{"start":"07:00","end":"22:00"}}
            current = now().strftime("%H:%M")
            return cond["value"]["start"] <= current <= cond["value"]["end"]
        return False

    async def _get_today_trigger_count(self, rule_id):
        """从 Redis 获取今日触发次数"""
        key = f"rule_trigger_count:{rule_id}:{now().strftime('%Y%m%d')}"
        count = self.state_cache.redis.get(key)
        return int(count) if count else 0

    async def _record_trigger(self, rule_id, source="scheduled"):
        """记录规则触发"""
        key = f"rule_trigger_count:{rule_id}:{now().strftime('%Y%m%d')}"
        self.state_cache.redis.incr(key)
        self.state_cache.redis.expire(key, 86400 * 2)  # 2天过期
        # 记录最后触发时间（含来源）
        self.state_cache.redis.set(
            f"rule_last_trigger:{rule_id}",
            json.dumps({"time": now().isoformat(), "source": source}),
            ex=86400
        )

    async def _record_misfire(self, rule_id, missed_time, action_taken):
        """记录误触发事件"""
        self.state_cache.redis.lpush(
            f"rule_misfire_log:{rule_id}",
            json.dumps({
                "missed_time": missed_time.isoformat(),
                "action_taken": action_taken,
                "detected_at": now().isoformat()
            })
        )
        self.state_cache.redis.ltrim(f"rule_misfire_log:{rule_id}", 0, 99)  # 保留最近100条

    async def _notify_rule_failure(self, rule, reason):
        """通知用户规则执行失败"""
        pass
```

**Cron 表达式示例与使用场景：**

| Cron 表达式 | 含义 | 典型场景 | 推荐误触发策略 |
|------------|------|---------|-------------|
| `0 7 * * *` | 每天 7:00 | 起床开灯+开窗帘 | FIRE_NOW（补执行） |
| `30 8 * * 1-5` | 工作日 8:30 | 上班自动关空调 | FIRE_NOW |
| `0 22 * * 0,6` | 周末 22:00 | 周末晚关灯 | FIRE_NEXT |
| `*/30 * * * *` | 每 30 分钟 | 定期检查温度并调节 | SKIP（等下一次） |
| `0 0 1 1 *` | 每年 1 月 1 日 | 新年灯光秀 | FIRE_NOW |

**定时规则的容错机制：**

- **误触发策略**（misfire_policy）：根据规则类型自动选择——重要规则补执行，高频规则跳过
- **时区感知调度**：APScheduler 原生支持 pytz 时区，调度器内部使用 UTC，触发时间按时区转换
- **夏令时处理**：Spring Forward 自动调整执行时间，Fall Back 消除歧义只执行一次
- **单实例保证**（max_instances=1）：同一规则不会并发执行，防止重复触发
- **冷却时间**（cooldown_seconds）：防止规则在短时间内被反复触发
- **每日上限**（max_daily_triggers）：避免规则逻辑错误导致的设备反复操作

**时区与夏令时处理完整流程：**

```
用户在北京设置"每天 7:00 开灯"（Cron: 0 7 * * *，时区: Asia/Shanghai）
    ↓
调度器在 UTC 23:00 触发（北京时间 7:00 = UTC 23:00 前一天）
    ↓
用户搬到纽约（时区: America/New_York）
    ↓
触发 on_timezone_changed → 重新调度
新 Cron 在 UTC 12:00 触发（纽约冬令时 7:00 = UTC 12:00）
    ↓
3 月夏令时开始 → 自动切换
新 Cron 在 UTC 11:00 触发（纽约夏令时 7:00 = UTC 11:00）
    ↓
DST 切换日检查：
- Spring Forward：2:00→3:00，如 Cron 指向 2:30 → 自动调整为 1:00
- Fall Back：2:00→1:00，如 Cron 指向 1:30 → 只执行一次（is_dst=True）
```

### MQTT 通信层：完整的连接管理

#### MQTT QoS 层级与场景映射

| QoS 级别 | 语义 | 智能家居场景 | 丢包代价 |
|----------|------|-------------|---------|
| QoS 0 | 最多一次，不保证送达 | 传感器高频状态上报（温度/湿度每秒上报，丢一条无所谓） | 低，下次上报会覆盖 |
| QoS 1 | 至少一次，保证送达但可能重复 | 设备控制指令（开灯/关灯）、状态变更通知 | 中，重复执行通常幂等 |
| QoS 2 | 恰好一次，保证送达且不重复 | 门锁开关、安防布防、OTA 升级指令 | 高，重复执行有安全隐患 |

```python
import ssl
import paho.mqtt.client as mqtt
from collections import defaultdict
from threading import Lock
import time
import logging


class MQTTConnectionManager:
    """MQTT 连接管理器：处理连接生命周期、QoS、会话持久化、遗嘱消息"""

    def __init__(self, broker_config, device_shadow_service, rule_engine):
        self.broker_config = broker_config
        self.device_shadow = device_shadow_service
        self.rule_engine = rule_engine

        # 连接池：按 home_id 分组，每个家庭一个连接
        self.connections = {}       # home_id → mqtt.Client
        self.connection_lock = Lock()

        # 遗嘱消息注册表
        self.will_messages = {}     # device_id → will_topic

        # 会话状态追踪
        self.session_state = {}     # client_id → {"clean": bool, "connected_at": float}

        # QoS 级别映射：按消息类型决定
        self.message_qos = {
            "sensor_report": 0,     # 传感器上报：QoS 0
            "device_command": 1,    # 设备控制指令：QoS 1
            "device_ack": 1,        # 设备确认回执：QoS 1
            "security_command": 2,  # 安防指令：QoS 2
            "ota_command": 2,       # OTA 升级指令：QoS 2
            "state_sync": 1,        # 状态同步：QoS 1
        }

        # 重连参数
        self.reconnect_delay = 1       # 初始重连延迟（秒）
        self.reconnect_delay_max = 30  # 最大重连延迟
        self.reconnect_attempts = 0

    def create_connection(self, home_id, clean_session=False):
        """创建 MQTT 连接

        Args:
            home_id: 家庭 ID，用于连接隔离
            clean_session: 
                True  = 清除会话，每次连接全新开始（适合临时监控）
                False = 持久会话，断线重连后恢复订阅和离线消息（适合常驻设备）
        """
        client_id = f"cloud-bridge-{home_id}"

        client = mqtt.Client(
            client_id=client_id,
            clean_session=clean_session,    # 持久会话：断线期间的消息会被保留
            protocol=mqtt.MQTTv311
        )

        # TLS 加密
        client.tls_set(
            ca_certs=self.broker_config["ca_cert"],
            certfile=self.broker_config["client_cert"],
            keyfile=self.broker_config["client_key"],
            tls_version=ssl.PROTOCOL_TLSv1_2
        )

        # 认证
        client.username_pw_set(
            self.broker_config["username"],
            self.broker_config["password"]
        )

        # 设置遗嘱消息：连接异常断开时自动发布
        # 这是设备离线检测的核心机制
        will_topic = f"home/{home_id}/bridge/status"
        will_payload = json.dumps({
            "status": "offline",
            "timestamp": int(time.time() * 1000),
            "reason": "unexpected_disconnect"
        })
        client.will_set(will_topic, will_payload, qos=1, retain=True)

        # 回调注册
        client.on_connect = self._on_connect
        client.on_disconnect = self._on_disconnect
        client.on_message = self._on_message
        client.on_publish = self._on_publish
        client.on_subscribe = self._on_subscribe

        # 自动重连配置
        client.reconnect_delay_set(
            min_delay=self.reconnect_delay,
            max_delay=self.reconnect_delay_max
        )

        # 连接到 Broker 集群（支持多节点故障转移）
        connected = False
        for broker_host in self.broker_config["hosts"]:
            try:
                client.connect(
                    broker_host,
                    port=self.broker_config["port"],
                    keepalive=60     # 60 秒心跳
                )
                connected = True
                logging.info(f"Connected to MQTT broker {broker_host} for home {home_id}")
                break
            except Exception as e:
                logging.warning(f"Failed to connect to {broker_host}: {e}")

        if not connected:
            raise ConnectionError(f"All MQTT brokers unavailable for home {home_id}")

        # 记录会话状态
        self.session_state[client_id] = {
            "clean": clean_session,
            "connected_at": time.time(),
            "broker_host": client._host
        }

        with self.connection_lock:
            self.connections[home_id] = client

        client.loop_start()
        return client

    def _on_connect(self, client, userdata, flags, rc):
        """连接成功回调"""
        if rc == 0:
            logging.info(f"MQTT connected: {client._client_id}")
            # 持久会话（clean_session=False）恢复时：
            #   - Broker 自动恢复之前的订阅
            #   - 离线期间的 QoS 1/2 消息会被投递
            #   - 不需要重新订阅
            if not flags.get("session present", False):
                # 新会话，需要重新订阅
                self._setup_subscriptions(client)
            else:
                logging.info(f"Session restored for {client._client_id}, skipping re-subscribe")

            self.reconnect_attempts = 0
        else:
            error_codes = {
                1: "协议版本不正确",
                2: "客户端标识符无效",
                3: "服务器不可用",
                4: "用户名或密码错误",
                5: "未授权"
            }
            logging.error(f"MQTT connect failed: rc={rc} {error_codes.get(rc, '未知错误')}")

    def _setup_subscriptions(self, client):
        """设置订阅主题"""
        home_id = client._client_id.decode().split("-")[-1] if isinstance(client._client_id, bytes) \
            else client._client_id.split("-")[-1]

        subscriptions = [
            (f"home/{home_id}/device/+/status", 1),       # 设备状态上报
            (f"home/{home_id}/device/+/ack", 1),           # 设备指令确认
            (f"home/{home_id}/device/+/event", 1),         # 设备事件（门铃、报警）
            (f"home/{home_id}/device/+/shadow/delta", 1),  # 影子设备增量同步
            (f"home/{home_id}/bridge/status", 1),          # 桥接状态
        ]
        for topic, qos in subscriptions:
            client.subscribe(topic, qos=qos)

    def _on_disconnect(self, client, userdata, rc):
        """断开连接回调"""
        if rc == 0:
            logging.info(f"MQTT gracefully disconnected: {client._client_id}")
        else:
            # 异常断开：遗嘱消息会被 Broker 自动发布
            logging.warning(f"MQTT unexpected disconnect: {client._client_id}, rc={rc}")
            self.reconnect_attempts += 1
            # paho-mqtt 会自动重连，此处记录状态即可
            home_id = self._extract_home_id(client)
            if home_id:
                # 标记该家庭所有设备状态为不确定
                self._mark_home_devices_uncertain(home_id)

    def _on_message(self, client, userdata, msg):
        """消息分发处理"""
        try:
            topic_parts = msg.topic.split("/")
            payload = json.loads(msg.payload.decode())

            # 按主题路由消息
            if "/status" in msg.topic:
                self._handle_device_status(topic_parts, payload)
            elif "/ack" in msg.topic:
                self._handle_device_ack(topic_parts, payload)
            elif "/event" in msg.topic:
                self._handle_device_event(topic_parts, payload)
            elif "/shadow/delta" in msg.topic:
                self._handle_shadow_delta(topic_parts, payload)
            elif "/bridge/status" in msg.topic:
                self._handle_bridge_status(topic_parts, payload)

        except json.JSONDecodeError:
            logging.error(f"Invalid JSON on topic {msg.topic}")
        except Exception as e:
            logging.error(f"Error processing message on {msg.topic}: {e}")

    def _handle_device_status(self, topic_parts, payload):
        """处理设备状态上报（QoS 0 或 1）"""
        device_id = payload.get("device_id") or topic_parts[3]
        state = payload.get("state", {})

        # 更新设备影子 reported 状态
        self.device_shadow.update_reported(device_id, state)

        # 更新缓存
        self.state_cache.update(device_id, state)

        # 触发规则引擎评估
        event = DeviceEvent(device_id=device_id, properties=state)
        self.rule_engine.on_event(event)

        # 推送状态给用户 App
        self.push_state_to_app(device_id)

    def _handle_device_ack(self, topic_parts, payload):
        """处理设备指令确认"""
        device_id = payload["device_id"]
        request_id = payload["request_id"]
        success = payload.get("success", False)

        if success:
            actual_state = payload.get("state", {})
            self.device_controller.on_device_ack(device_id, request_id, actual_state)
        else:
            error_code = payload.get("error_code", "UNKNOWN")
            logging.error(f"Device {device_id} rejected command: {error_code}")
            self.state_cache.rollback(device_id)

    def _handle_device_event(self, topic_parts, payload):
        """处理设备事件（门铃、报警等）"""
        device_id = payload["device_id"]
        event_type = payload["event_type"]
        logging.info(f"Device event: {device_id} → {event_type}")

        # 安全事件立即推送
        if event_type in ("smoke_alarm", "water_leak", "intrusion"):
            self.notification_service.send_urgent(device_id, event_type, payload)

    def _handle_shadow_delta(self, topic_parts, payload):
        """处理影子设备增量消息"""
        device_id = payload["device_id"]
        delta = payload.get("delta", {})
        logging.info(f"Shadow delta for {device_id}: {delta}")
        # 推送 delta 到设备
        self.publish_to_device(device_id, "shadow_delta", delta, qos=1)

    def _handle_bridge_status(self, topic_parts, payload):
        """处理桥接状态变更（含遗嘱消息触发的离线通知）"""
        status = payload.get("status")
        if status == "offline":
            home_id = topic_parts[1]
            logging.warning(f"Bridge offline for home {home_id}, marking devices uncertain")
            self._mark_home_devices_uncertain(home_id)

    def publish_to_device(self, device_id, command_type, payload, qos=None):
        """向设备发布指令"""
        home_id = self._get_home_id(device_id)
        if not home_id:
            logging.error(f"Unknown device {device_id}, cannot publish")
            return False

        # 自动选择 QoS
        if qos is None:
            qos = self.message_qos.get(command_type, 1)

        client = self.connections.get(home_id)
        if not client or not client.is_connected():
            logging.error(f"No MQTT connection for home {home_id}")
            return False

        topic = f"home/{home_id}/device/{device_id}/command"
        message = json.dumps({
            **payload,
            "timestamp": int(time.time() * 1000),
            "qos": qos
        })

        result = client.publish(topic, message, qos=qos)
        if result.rc != mqtt.MQTT_ERR_SUCCESS:
            logging.error(f"Failed to publish to {topic}: rc={result.rc}")
            return False

        return True

    def register_device_will(self, device_id, home_id):
        """注册设备遗嘱消息：设备异常断开时 Broker 自动发布

        遗嘱消息机制是 MQTT 协议的内置特性：
        - 设备连接时注册遗嘱主题和内容
        - Broker 检测到设备连接断开（心跳超时）时自动发布遗嘱
        - 无需额外心跳检测逻辑，Broker 代劳
        """
        will_topic = f"home/{home_id}/device/{device_id}/status"
        will_payload = json.dumps({
            "device_id": device_id,
            "status": "offline",
            "reason": "connection_lost",
            "timestamp": int(time.time() * 1000)
        })
        self.will_messages[device_id] = will_topic
        return will_topic, will_payload

    def _mark_home_devices_uncertain(self, home_id):
        """标记家庭内所有设备状态为不确定"""
        devices = self._get_home_devices(home_id)
        for device_id in devices:
            self.state_cache.mark_stale(device_id)

    def _extract_home_id(self, client):
        """从 client_id 提取 home_id"""
        cid = client._client_id
        if isinstance(cid, bytes):
            cid = cid.decode()
        parts = cid.split("-")
        return parts[-1] if parts else None

    def _get_home_id(self, device_id):
        """查询设备所属家庭"""
        # 从缓存或数据库获取
        return self.device_home_cache.get(device_id)

    def _get_home_devices(self, home_id):
        """获取家庭内所有设备"""
        return self.home_device_cache.get(home_id, [])

    def on_device_offline(self, device_id):
        """设备离线（由遗嘱消息或心跳超时触发）"""
        self.state_cache.mark_offline(device_id)
        self.notify_user(device_id, "设备已离线")

    def graceful_shutdown(self):
        """优雅关闭所有连接：发布 offline 状态（非遗嘱消息），然后断开"""
        for home_id, client in self.connections.items():
            # 主动发布离线状态（区别于遗嘱消息，表示计划内下线）
            status_topic = f"home/{home_id}/bridge/status"
            client.publish(
                status_topic,
                json.dumps({"status": "offline", "reason": "planned_shutdown"}),
                qos=1,
                retain=True
            )
            client.disconnect()
            client.loop_stop()
        logging.info("All MQTT connections gracefully closed")
```

**MQTT 会话模式对比：**

| 特性 | Clean Session (=true) | Persistent Session (=false) |
|------|----------------------|---------------------------|
| 断线期间消息 | 丢弃 | Broker 缓存 QoS 1/2 消息 |
| 重连后订阅 | 需重新订阅 | 自动恢复订阅 |
| 适用场景 | 临时监控面板、调试工具 | 常驻设备、规则引擎 |
| Broker 内存占用 | 低（不缓存） | 中（需缓存离线消息和订阅） |
| 推荐使用 | 管理端 | 设备端和云端桥接 |

**遗嘱消息离线检测流程：**

```
设备连接 → 注册遗嘱消息(topic=device/status, payload=offline)
    ↓
Broker 维持心跳（keepalive=60s）
    ↓
┌─ 设备正常断开 → Broker 丢弃遗嘱消息（不发布）
└─ 设备异常断开（心跳超时/网络中断）→ Broker 自动发布遗嘱消息
    ↓
云端收到遗嘱消息 → 标记设备离线 → 触发离线处理流程
    ↓
设备重连 → 发布 online 状态 → 触发上线补执行流程
```

## 常见陷阱（深度分析）

### 陷阱 1：规则不检测冲突

**后果：** 同一设备收到矛盾指令（开灯+关灯）→ 行为不可预测（取决于哪个先到达）→ 用户困惑。

**解决方案：** 冲突检测 + 优先级解决。场景模式优先级 > 舒适规则 > 便利规则。

### 陷阱 2：设备状态不同步

**后果：** 用户看到 App 上"灯开着"但实际已关 → 反复操作 → 体验差。

**解决方案：** 乐观更新 + 设备 ACK 确认 + 超时标记 stale。

### 陷阱 3：离线指令无限积压

**后果：** 设备离线 30 天 → 积压 30 天的定时指令 → 上线后疯狂执行 → 异常行为（凌晨 3 点所有灯亮起）。

**解决方案：** 队列最大 50 条 + 条件规则重新评估 + 定时规则只补执行最近的。

### 陷阱 4：规则硬编码

**后果：** 新设备类型需要改代码发版 → 无法快速支持新产品 → 上市周期长。

**解决方案：** 规则数据化（JSON 配置），引擎通用化。新设备只需注册属性（如"开关状态"、"温度值"），规则引擎自动支持。

### 陷阱 5：全量扫描规则

**后果：** 每个传感器事件评估 2500 万条规则 → 延迟 > 1 秒 → 规则触发不及时。

**解决方案：** 设备 ID 索引 → 只评估同设备相关规则（平均 5 条）→ 延迟 < 10ms。

## 延伸思考

- **Matter 协议**：跨厂商标准，统一设备模型 → 减少适配层代码。新设备自动发现和注册。
- **语音控制**：NLU 解析自然语言为规则条件/动作。"帮我把卧室温度调到 26 度" → 解析为 set_temperature(bedroom, 26)。
- **场景推荐**：AI 分析用户习惯，自动推荐自动化规则。"你通常 22:00 关灯，是否设为自动？"
## 设备影子完整实现

```python
class DeviceShadowService:
    """设备影子：期望状态 vs 报告状态"""

    def update_desired(self, device_id, desired_state):
        """用户/规则更新期望状态"""
        shadow = self.redis.hgetall(f"shadow:{device_id}")
        shadow["desired"] = json.dumps(desired_state)
        shadow["desired_version"] = int(shadow.get("desired_version", 0)) + 1
        shadow["desired_timestamp"] = now().isoformat()
        self.redis.hset(f"shadow:{device_id}", mapping=shadow)

        # 计算差异并发送命令
        reported = json.loads(shadow.get("reported", "{}"))
        delta = self._compute_delta(desired_state, reported)
        if delta:
            self.mqtt_client.publish(f"cmd/{device_id}", json.dumps(delta), qos=1)

    def update_reported(self, device_id, reported_state):
        """设备上报当前状态"""
        shadow = self.redis.hgetall(f"shadow:{device_id}")
        shadow["reported"] = json.dumps(reported_state)
        shadow["reported_version"] = int(shadow.get("reported_version", 0)) + 1
        shadow["reported_timestamp"] = now().isoformat()
        self.redis.hset(f"shadow:{device_id}", mapping=shadow)

        # 检查是否与期望一致
        desired = json.loads(shadow.get("desired", "{}"))
        delta = self._compute_delta(desired, reported_state)
        if delta:
            # 设备状态与期望不一致 → 重新发送命令
            self.mqtt_client.publish(f"cmd/{device_id}", json.dumps(delta), qos=1)

    def _compute_delta(self, desired, reported):
        """计算期望和报告之间的差异"""
        delta = {}
        for key, value in desired.items():
            if reported.get(key) != value:
                delta[key] = value
        return delta if delta else None
```

## 定时规则执行器

```python
class CronRuleExecutor:
    """Cron 定时规则执行器"""

    def execute_scheduled_rules(self, current_time):
        """执行到期的定时规则"""
        due_rules = self.db.query(
            "SELECT * FROM scheduled_rules WHERE next_fire_time <= %s AND enabled = 1",
            current_time)

        for rule in due_rules:
            try:
                # 执行规则动作
                result = self.execute_action(rule["action"], rule["params"])

                # 更新下次触发时间
                next_fire = self._calc_next_fire(rule["cron_expression"], current_time)
                self.db.update("scheduled_rules",
                    {"last_fire_time": current_time, "next_fire_time": next_fire},
                    {"id": rule["id"]})

            except TimeoutError:
                # 规则执行超时 → 记录但不影响其他规则
                self.db.update("scheduled_rules",
                    {"last_error": "timeout", "error_count": rule["error_count"] + 1},
                    {"id": rule["id"]})

    def _calc_next_fire(self, cron_expr, after_time):
        """计算下次触发时间（支持时区和夏令时）"""
        from croniter import croniter
        cron = croniter(cron_expr, after_time)
        return cron.get_next(datetime)
```

## 规则冲突解决引擎

```python
class RuleConflictResolver:
    """规则冲突解决：优先级 + 作用域"""

    def resolve(self, device_id, conflicting_actions):
        """解决对同一设备的冲突动作"""
        # 按优先级排序（高优先级优先）
        sorted_actions = sorted(conflicting_actions,
            key=lambda a: (a["priority"], a["scope_priority"]), reverse=True)

        winner = sorted_actions[0]
        losers = sorted_actions[1:]

        # 记录冲突日志
        self.db.insert("rule_conflict_log", {
            "device_id": device_id,
            "winner_rule_id": winner["rule_id"],
            "winner_action": winner["action"],
            "loser_rule_ids": json.dumps([l["rule_id"] for l in losers]),
            "resolved_at": now()
        })

        return winner
```

## 异常场景补充

### 场景：固件 OTA 失败与回滚

```python
class FirmwareOTAManager:
    """固件 OTA 升级管理"""
    def upgrade(self, device_id, firmware_version):
        """安全 OTA 升级"""
        # 1. 下载固件到设备
        self.mqtt_client.publish(f"ota/{device_id}", json.dumps({
            "action": "download", "url": self.get_firmware_url(firmware_version),
            "checksum": self.get_firmware_checksum(firmware_version)
        }), qos=2)

        # 2. 等待设备确认下载完成
        download_result = self.wait_for_ack(device_id, timeout=120)
        if not download_result.success:
            raise OTADownloadFailedError(download_result.error)

        # 3. 指示设备应用固件
        self.mqtt_client.publish(f"ota/{device_id}", json.dumps({
            "action": "apply"
        }), qos=2)

        # 4. 等待设备重启并上报新版本
        apply_result = self.wait_for_ack(device_id, timeout=180)
        if not apply_result.success:
            # 固件应用失败 → 设备自动回滚到旧版本
            self.alert(f"OTA 失败: {device_id}, 设备将自动回滚")
            return {"status": "failed", "rollback": "auto"}
```

### 场景：网络分区导致设备状态不同步

```
触发：家庭网络断开 → 设备离线 → 用户通过 App 改变期望状态
      → 设备恢复后 → 影子同步期望状态到设备
检测：
  1. 设备离线：MQTT Will 消息 → 标记离线
  2. 期望状态版本 > 报告状态版本 → 存在未同步的命令
处理：
  1. 设备重连 → 发送影子完整状态（不只是 delta）
  2. 设备应用后 → 上报新状态 → 影子更新
预防：设备重连时总是做完整同步 + 版本号校验
```

### 场景：规则执行超时级联

```
触发：规则 A 控制智能灯泡 → 灯泡响应慢（5 秒）
      → 规则 A 超时 → 阻塞规则 B（控制空调）执行
检测：
  1. 规则执行时间 > 3 秒 → 告警
  2. 规则队列积压 > 10 → 严重告警
处理：
  1. 每个规则独立执行（不阻塞其他规则）
  2. 规则执行超时 → 跳过该规则，继续执行下一个
  3. 超时规则标记为 "slow" → 降低优先级
预防：规则并行执行 + 每个规则独立超时（3 秒）
```

## OTA 固件管理完整实现

```python
class FirmwareOTAManager:
    """OTA 固件升级管理"""

    def deploy_firmware(self, firmware_id, target_devices):
        """渐进式固件部署"""
        deployment_id = str(uuid4())
        firmware = self.db.get_firmware(firmware_id)

        # 验证固件完整性
        if not self._verify_checksum(firmware):
            raise FirmwareIntegrityError("固件校验失败")

        phases = [
            {"name": "canary", "percentage": 0.01, "duration_minutes": 60},
            {"name": "early", "percentage": 0.10, "duration_minutes": 360},
            {"name": "majority", "percentage": 0.50, "duration_minutes": 720},
            {"name": "full", "percentage": 1.00, "duration_minutes": 1440},
        ]

        for phase in phases:
            # 选择目标设备
            devices = target_devices[:int(len(target_devices) * phase["percentage"])]

            # 推送固件
            for device in devices:
                compatibility = self._check_compatibility(device, firmware)
                if not compatibility["compatible"]:
                    continue  # 跳过不兼容设备
                self.mqtt_client.publish(f"ota/{device.id}", json.dumps({
                    "action": "update",
                    "version": firmware["version"],
                    "url": firmware["download_url"],
                    "checksum": firmware["sha256"],
                    "size_bytes": firmware["size_bytes"]
                }), qos=2)

            # 等待并监控
            result = self._monitor_deployment(deployment_id, phase["duration_minutes"])

            if result["failure_rate"] > 0.05:
                # 失败率 > 5% → 自动回滚
                self._rollback_deployment(deployment_id, devices, firmware)
                self.alert(f"OTA 回滚: 失败率 {result['failure_rate']:.1%}")
                return {"status": "rolled_back", "reason": "high_failure_rate"}

        return {"status": "deployed", "deployment_id": deployment_id}

    def _check_compatibility(self, device, firmware):
        """检查设备与固件兼容性"""
        return self.db.query_one(
            "SELECT * FROM firmware_compatibility "
            "WHERE device_model = %s AND firmware_version = %s",
            device.model, firmware["version"])
```

## 场景自动化引擎

```python
class SceneAutomationEngine:
    """场景自动化引擎"""

    def execute_scene(self, scene_id, trigger_context=None):
        """执行场景（如"早安场景"、"离家场景"）"""
        scene = self.db.get_scene(scene_id)

        # 检查条件
        for condition in scene["conditions"]:
            if not self._evaluate_condition(condition, trigger_context):
                return {"executed": False, "reason": f"条件不满足: {condition['name']}"}

        # 按顺序执行动作
        executed_actions = []
        for action in scene["actions"]:
            try:
                result = self._execute_action(action)
                executed_actions.append({"action": action["name"], "result": "success"})
                # 动作间延迟（避免设备过载）
                time.sleep(action.get("delay_seconds", 0.5))
            except Exception as e:
                executed_actions.append({"action": action["name"], "result": "failed", "error": str(e)})

        # 记录场景执行日志
        self.db.insert("scene_execution_log", {
            "scene_id": scene_id,
            "trigger_context": json.dumps(trigger_context),
            "executed_actions": json.dumps(executed_actions),
            "executed_at": now()
        })

        return {"executed": True, "actions": executed_actions}

    def _evaluate_condition(self, condition, context):
        """评估场景条件"""
        if condition["type"] == "time_range":
            now_time = now().time()
            return condition["start_time"] <= now_time <= condition["end_time"]
        elif condition["type"] == "weekday":
            return now().weekday() < 5  # Mon-Fri
        elif condition["type"] == "device_state":
            state = self.device_shadow.get_state(condition["device_id"])
            return state.get(condition["property"]) == condition["value"]
        elif condition["type"] == "presence":
            return self.presence_service.is_home(condition["user_id"])
        return True
```

## 异常场景补充

### 场景：设备恢复出厂设置

```
触发：用户误操作恢复出厂 → 设备状态丢失 → 无法通过 App 控制
检测：
  1. 设备上线但报告初始固件版本 → 出厂重置检测
  2. 设备 shadow 版本号重置为 0
处理：
  1. 设备自动注册到云端（设备序列号不变）
  2. 云端推送最新固件
  3. 恢复用户的个性化配置
  4. 通知用户重新配对设备
预防：出厂重置后自动恢复注册 + 配置备份
```

### 场景：MQTT Broker 故障转移

```
触发：主 MQTT Broker 宕机 → 所有设备断开
检测：
  1. Broker 心跳检测 5 秒超时 → 告警
  2. 设备连接数归零 → 严重告警
处理：
  1. DNS 切换到备用 Broker（TTL 30s）
  2. 设备自动重连到备用 Broker
  3. 恢复订阅关系
  4. 通知用户设备可能短暂离线
预防：MQTT Broker 集群 + 自动故障转移 + 设备端多 Broker 配置
```

### 场景：规则执行死锁

```
触发：规则 A 触发设备 X → 规则 B 监听 X 变化 → 触发设备 Y
      → 规则 A 又监听 Y 变化 → 循环触发 → 死锁
检测：
  1. 同一规则 10 秒内执行 > 3 次 → 振荡检测
  2. 设备状态在两个值之间反复切换 → 死锁检测
处理：
  1. 振荡检测 → 自动暂停涉事规则
  2. 通知用户"规则冲突，已自动暂停"
  3. 用户修改规则条件消除循环
预防：规则 DAG 检测循环依赖 + 执行频率限制
```

## 能耗监控与优化完整实现

```python
class EnergyMonitorService:
    """设备能耗监控与优化"""

    def track_consumption(self, device_id, watts, duration_seconds):
        """记录设备能耗"""
        kwh = watts * duration_seconds / 3600000
        hour_key = now().strftime('%Y%m%d%H')
        self.redis.hincrbyfloat(f"energy:{device_id}:{hour_key}", "kwh", kwh)
        self.redis.expire(f"energy:{device_id}:{hour_key}", 86400)

        # 异常检测：能耗 3 倍于正常值
        baseline = self._get_baseline(device_id, now().hour)
        if kwh > baseline * 3:
            self.alert(f"设备 {device_id} 能耗异常: {kwh:.3f} kWh (基线 {baseline:.3f})")

    def get_daily_report(self, home_id, date):
        """获取每日能耗报告"""
        devices = self.db.query(
            "SELECT * FROM devices WHERE home_id = %s", home_id)
        report = {"home_id": home_id, "date": date, "devices": [], "total_kwh": 0}
        for device in devices:
            kwh = self._get_daily_consumption(device["id"], date)
            baseline = self._get_device_baseline(device["id"])
            report["devices"].append({
                "device_id": device["id"],
                "device_name": device["name"],
                "kwh": kwh,
                "baseline_kwh": baseline,
                "deviation": (kwh - baseline) / max(baseline, 0.001) * 100
            })
            report["total_kwh"] += kwh

        # 节能建议
        report["suggestions"] = self._generate_suggestions(report["devices"])
        return report

    def _generate_suggestions(self, devices):
        """生成节能建议"""
        suggestions = []
        for d in devices:
            if d["deviation"] > 50:
                suggestions.append(f"{d['device_name']} 能耗偏高 {d['deviation']:.0f}%，建议检查是否需要维修")
            if d["kwh"] > 5 and "空调" in d["device_name"]:
                suggestions.append(f"空调日耗 {d['kwh']:.1f} kWh，建议设置定时关闭或提高温度 1°C")
        return suggestions
```

## MQTT QoS 完整处理

```python
class MQTTQoSManager:
    """MQTT QoS 分级处理"""

    QOS_MAP = {
        "command": 1,       # 控制命令：至少一次
        "state_report": 0,  # 状态上报：最多一次（丢失可接受）
        "alert": 2,         # 告警：恰好一次
        "ota": 2,           # 固件更新：恰好一次
        "heartbeat": 0,     # 心跳：最多一次
    }

    def publish(self, topic_type, device_id, payload):
        """按消息类型选择 QoS"""
        qos = self.QOS_MAP.get(topic_type, 1)
        topic = f"{topic_type}/{device_id}"

        if qos == 2:
            # QoS 2: 恰好一次，需要 PUBREC/PUBREL/PUBCOMP
            message_id = self.mqtt.publish_with_qos2(topic, payload)
            # 等待 PUBCOMP 确认
            confirmed = self.wait_for_pubcomp(message_id, timeout=5)
            if not confirmed:
                self.db.insert("mqtt_unconfirmed", {
                    "message_id": message_id, "topic": topic,
                    "payload": payload, "qos": 2,
                    "created_at": now()
                })
        else:
            self.mqtt.publish(topic, payload, qos=qos)

    def setup_will_message(self, device_id):
        """设置遗嘱消息：设备离线时自动发布"""
        self.mqtt.set_will(
            topic=f"status/{device_id}",
            payload=json.dumps({"status": "offline", "timestamp": now().isoformat()}),
            qos=1, retain=True)
```

## 异常场景补充

### 场景：设备固件 OTA 失败与回滚

```python
class OTARollbackHandler:
    """OTA 失败自动回滚"""
    def handle_failure(self, device_id, target_version):
        # 设备自动回滚到上一版本
        current = self.db.get_device_firmware(device_id)
        prev = self.db.get_previous_version(device_id)
        self.mqtt.publish(f"ota/{device_id}", json.dumps({
            "action": "rollback", "version": prev["version"],
            "url": prev["download_url"], "checksum": prev["sha256"]
        }), qos=2)
        self.alert(f"设备 {device_id} OTA 失败，回滚到 {prev['version']}")
```

### 场景：MQTT Broker 故障转移

```
触发：主 MQTT Broker 宕机 → 所有设备断开
处理：
  1. DNS 切换到备用 Broker（TTL 30s）
  2. 设备配置多个 Broker 地址 → 自动重连
  3. 恢复订阅关系 → 从 Redis 加载
  4. 遗嘱消息触发 → 标记设备离线 → 重连后恢复在线
预防：MQTT 集群 + 多 Broker 配置 + 会话持久化
```

### 场景：规则执行死锁

```
触发：规则 A → 控制灯 → 规则 B 监听灯变化 → 控制灯 → 循环
检测：同一设备 10 秒内状态变更 > 5 次 → 振荡
处理：
  1. 自动暂停触发振荡的规则
  2. 通知用户规则冲突
  3. 规则 DAG 检测循环依赖
预防：规则编辑时检测循环 + 执行频率限制
```

## Zigbee Mesh 网络管理完整实现

```python
class ZigbeeMeshManager:
    """Zigbee Mesh 网络管理"""

    def pair_device(self, gateway_id, device_info):
        """设备配对入网"""
        # 1. 开启网关允许加入模式（60秒超时）
        self.zigbee_client.permit_join(gateway_id, duration=60)

        # 2. 等待设备加入
        join_event = self.wait_for_join(gateway_id, timeout=60)
        if not join_event:
            raise DevicePairingTimeoutError("设备配对超时")

        # 3. 分配网络地址
        nwk_addr = self.zigbee_client.assign_address(gateway_id, join_event["ieee_addr"])

        # 4. 建立父子关系
        parent = self._find_best_parent(gateway_id, join_event["ieee_addr"])
        self.db.insert("zigbee_network", {
            "ieee_addr": join_event["ieee_addr"],
            "nwk_addr": nwk_addr,
            "parent_nwk_addr": parent["nwk_addr"],
            "device_type": device_info["type"],
            "gateway_id": gateway_id,
            "joined_at": now()
        })
        return {"nwk_addr": nwk_addr, "parent": parent["nwk_addr"]}

    def heal_network(self, gateway_id):
        """网络自愈：重新路由断线设备"""
        offline = self.db.query(
            "SELECT * FROM zigbee_network WHERE gateway_id = %s "
            "AND last_heartbeat < NOW() - INTERVAL 5 MINUTE", gateway_id)

        healed = 0
        for device in offline:
            # 尝试通过其他路径到达
            new_parent = self._find_best_parent(gateway_id, device["ieee_addr"])
            if new_parent:
                self.zigbee_client.rejoin(device["nwk_addr"], new_parent["nwk_addr"])
                self.db.update("zigbee_network",
                    {"parent_nwk_addr": new_parent["nwk_addr"], "last_heartbeat": now()},
                    {"ieee_addr": device["ieee_addr"]})
                healed += 1

        return {"total_offline": len(offline), "healed": healed}
```

## 语音助手集成

```python
class VoiceAssistantIntegrator:
    """语音助手集成：意图解析 → 设备控制"""

    INTENT_MAP = {
        "turn_on": ["开灯", "打开", "启动", "开启"],
        "turn_off": ["关灯", "关闭", "关掉", "关"],
        "set_temperature": ["设温度", "调到", "温度"],
        "set_brightness": ["调亮度", "亮一点", "暗一点"],
        "scene_good_morning": ["早安", "早上好", "起床"],
        "scene_good_night": ["晚安", "睡觉", "休息"],
    }

    def process_voice_command(self, user_id, text):
        """处理语音指令"""
        # 1. 意图识别
        intent = self._match_intent(text)
        if not intent:
            return {"action": "clarify", "message": "我没听懂，请再说一次"}

        # 2. 提取参数
        params = self._extract_params(text, intent)

        # 3. 映射到设备动作
        action = self._map_to_action(intent, params, user_id)

        # 4. 执行
        result = self.device_controller.execute(action)
        return {"action": "executed", "intent": intent, "result": result}

    def _match_intent(self, text):
        """匹配意图"""
        for intent, keywords in self.INTENT_MAP.items():
            for keyword in keywords:
                if keyword in text:
                    return intent
        return None

    def _extract_params(self, text, intent):
        """提取参数（温度值、亮度值、设备名）"""
        import re
        params = {}
        if intent == "set_temperature":
            match = re.search(r'(\d+)\s*度', text)
            if match:
                params["temperature"] = int(match.group(1))
        elif intent == "set_brightness":
            match = re.search(r'(\d+)%', text)
            if match:
                params["brightness"] = int(match.group(1))
        return params
```

## 家庭安防系统

```python
class HomeSecurityService:
    """家庭安防系统"""

    def handle_intrusion_alert(self, home_id, sensor_id, alert_data):
        """处理入侵告警"""
        # 1. 验证告警（排除误报：宠物触发）
        if self._is_false_alarm(sensor_id, alert_data):
            return {"action": "ignore", "reason": "possible_pet"}

        # 2. 分级告警
        severity = self._assess_severity(alert_data)
        if severity == "critical":
            # 3a. 紧急：推送 + 电话 + 视频验证
            self.push_notification(home_id, "检测到入侵！")
            self.emergency_call(home_id)
            # 启动摄像头录制
            self.start_recording(home_id)
        elif severity == "warning":
            # 3b. 警告：推送通知
            self.push_notification(home_id, "检测到异常活动")
            # 请求视频确认
            self.request_video_verification(home_id)

        # 4. 记录告警
        self.db.insert("security_alerts", {
            "home_id": home_id, "sensor_id": sensor_id,
            "severity": severity, "data": json.dumps(alert_data),
            "triggered_at": now()
        })

        return {"action": "alerted", "severity": severity}
```

## 异常场景补充

### 场景：Zigbee 网络分区

```
触发：中间路由设备断电 → 下游设备无法与网关通信
检测：
  1. 多个设备同时心跳超时 → 网络分区检测
  2. 检查是否是同一父节点的子设备
处理：
  1. 自愈：下游设备尝试通过其他路径重连
  2. 自愈失败 → 通知用户恢复中间设备
  3. 恢复后设备自动重新入网
预防：关键路由设备使用有线供电 + 网络拓扑监控
```

### 场景：语音助手误识别

```
触发：用户说"关掉客厅灯" → 识别为"关掉全部灯"
检测：
  1. 指令影响设备 > 3 个 → 二次确认
  2. "全部"、"所有"关键词 → 要求确认
处理：
  1. 语音回复："您要关闭所有灯，确认吗？"
  2. 用户确认 → 执行
  3. 用户取消 → 不执行
预防：影响范围大的指令需要二次确认
```

## 设备分组管理完整实现

```python
class DeviceGroupService:
    """设备分组：房间分组、场景联动分组"""

    def create_room(self, home_id, room_name, device_ids):
        """创建房间分组"""
        room_id = str(uuid4())
        self.db.insert("rooms", {
            "room_id": room_id, "home_id": home_id,
            "name": room_name, "created_at": now()
        })
        # 将设备分配到房间
        for device_id in device_ids:
            self.db.update("devices",
                {"room_id": room_id}, {"id": device_id})
        return room_id

    def control_room(self, room_id, action):
        """房间级控制（如"关灯"控制房间内所有灯）"""
        devices = self.db.query(
            "SELECT * FROM devices WHERE room_id = %s AND type IN ('light', 'switch')",
            room_id)
        for device in devices:
            self.device_shadow.update_desired(device["id"], {"power": action})
```

## 家庭成员权限管理

```python
class HomeMemberService:
    """家庭成员权限管理"""

    ROLES = {
        "owner": {"permissions": ["all"]},
        "admin": {"permissions": ["device_control", "scene_manage", "member_invite"]},
        "member": {"permissions": ["device_control", "scene_use"]},
        "guest": {"permissions": ["device_control_limited"]},
    }

    def add_member(self, home_id, user_id, role="member"):
        """添加家庭成员"""
        if role == "owner":
            raise InvalidRoleError("每个家庭只能有一个 owner")

        self.db.insert("home_members", {
            "home_id": home_id, "user_id": user_id,
            "role": role, "joined_at": now()
        })

    def check_permission(self, user_id, home_id, action):
        """检查用户是否有权限执行操作"""
        member = self.db.query_one(
            "SELECT * FROM home_members WHERE home_id = %s AND user_id = %s",
            home_id, user_id)
        if not member:
            return False

        role_perms = self.ROLES[member["role"]]["permissions"]
        if "all" in role_perms:
            return True
        return action in role_perms
```

## 异常场景补充

### 场景：设备离线后恢复同步

```
触发：智能灯泡断电 → 设备离线 → 用户通过 App 改变期望状态
      → 灯泡恢复供电 → 需要同步到最新期望状态
检测：
  1. 设备重连 → 比对设备影子 desired vs reported
  2. 不一致 → 需要同步
处理：
  1. 设备重连后推送完整影子状态（不只是 delta）
  2. 设备应用后上报新状态 → 影子更新
  3. 同步期间设备操作排队等待
预防：设备重连时总是做完整同步 + 版本号校验
```

### 场景：多用户同时控制同一设备

```
触发：父亲用 App 开灯 → 母亲同时用语音关灯 → 冲突
检测：
  1. 设备影子 desired 版本冲突
  2. 两个控制指令时间差 < 1 秒
处理：
  1. 后到者覆盖（Last Write Wins）
  2. 推送最终状态给所有用户
  3. 设备执行最终状态
预防：Last Write Wins + 实时状态推送 + 冲突日志
```

## 场景自动化调度器完整实现

```python
class SceneScheduler:
    """场景自动化调度器：定时 + 日出日落 + 地理围栏"""

    def schedule_scene(self, home_id, scene_id, trigger):
        """调度场景触发"""
        schedule_id = str(uuid4())
        self.db.insert("scene_schedules", {
            "schedule_id": schedule_id,
            "home_id": home_id, "scene_id": scene_id,
            "trigger_type": trigger["type"],  # cron / sunrise / geofence
            "trigger_config": json.dumps(trigger),
            "enabled": True,
            "created_at": now()
        })

        if trigger["type"] == "cron":
            self._schedule_cron(schedule_id, trigger["cron"])
        elif trigger["type"] == "sunrise":
            self._schedule_sunrise(schedule_id, home_id, trigger)
        elif trigger["type"] == "geofence":
            self._register_geofence(schedule_id, home_id, trigger)

    def _schedule_sunrise(self, schedule_id, home_id, trigger):
        """日出/日落感知调度"""
        # 获取家庭位置
        home = self.db.get_home(home_id)
        # 计算明天的日出/日落时间
        sun_times = self.sun_calculator.get_times(
            home["latitude"], home["longitude"], tomorrow())
        trigger_time = sun_times["sunrise" if trigger["event"] == "sunrise" else "sunset"]
        trigger_time += timedelta(minutes=trigger.get("offset_minutes", 0))

        # 调度到具体时间
        self.scheduler.schedule_at(schedule_id, trigger_time, 
            lambda: self.execute_scene(home_id, trigger["scene_id"]))

    def _register_geofence(self, schedule_id, home_id, trigger):
        """地理围栏触发"""
        # 当用户进入/离开家 500 米范围
        user_id = trigger["user_id"]
        self.geofence_service.register(
            user_id=user_id, home_id=home_id,
            radius_meters=500,
            on_enter=lambda: self.execute_scene(home_id, trigger["enter_scene"]),
            on_exit=lambda: self.execute_scene(home_id, trigger["exit_scene"]))

    def enable_vacation_mode(self, home_id):
        """度假模式：模拟有人在家（防盗）"""
        # 随机开关灯 + 拉窗帘
        schedule = [
            {"time": "18:30", "actions": {"living_room_light": "on", "curtain": "closed"}},
            {"time": "19:00", "actions": {"kitchen_light": "on"}},
            {"time": "21:30", "actions": {"bedroom_light": "on", "living_room_light": "off"}},
            {"time": "23:00", "actions": {"bedroom_light": "off"}},
        ]
        # 加入随机偏移（±30 分钟）避免规律性
        for item in schedule:
            offset = random.randint(-30, 30)
            actual_time = (datetime.strptime(item["time"], "%H:%M") +
                timedelta(minutes=offset)).strftime("%H:%M")
            self._schedule_cron(str(uuid4()),
                f"0 {actual_time.split(':')[1]} {actual_time.split(':')[0]} * * *")
```

## 设备能力抽象层

```python
class DeviceCapabilityAdapter:
    """设备能力抽象：统一接口 + 协议适配"""

    CAPABILITY_MAP = {
        "light": ["power", "brightness", "color_temp", "color_rgb"],
        "ac": ["power", "temperature", "mode", "fan_speed"],
        "curtain": ["power", "position"],
        "lock": ["power", "lock_state"],
        "sensor_temp": ["temperature", "humidity"],
    }

    def control(self, device_id, capability, value):
        """统一控制接口"""
        device = self.db.get_device(device_id)

        # 检查设备是否支持该能力
        supported = self.CAPABILITY_MAP.get(device["type"], [])
        if capability not in supported:
            raise CapabilityNotSupportedError(
                f"设备 {device_id} ({device['type']}) 不支持 {capability}")

        # 协议适配
        protocol = device["protocol"]  # zigbee / wifi / ble
        adapter = self._get_adapter(protocol)
        adapter.send_command(device_id, capability, value)

    def _get_adapter(self, protocol):
        """获取协议适配器"""
        adapters = {
            "zigbee": ZigbeeAdapter(),
            "wifi": WiFiAdapter(),
            "ble": BLEAdapter(),
        }
        return adapters.get(protocol)
```

## 异常场景补充

### 场景：时区设置错误

```
触发：用户搬家到不同时区 → 日出日落场景触发时间错误
检测：
  1. 用户 IP 地理位置与家庭时区不一致 → 提示
  2. 日出时间与预期差异 > 1 小时 → 可能时区错误
处理：
  1. 重新检测家庭位置 → 更新时区
  2. 重新计算日出/日落时间
  3. 调整所有日出/日落相关场景
预防：位置变更时自动更新时区 + 用户提示
```

### 场景：固件更新后能力不兼容

```
触发：设备固件更新 → 新增/移除能力 → 规则引用不存在的功能
检测：
  1. 固件更新后检查能力变化
  2. 规则引用的能力不在新能力列表 → 不兼容
处理：
  1. 禁用引用已移除能力的规则
  2. 通知用户规则已禁用
  3. 提供规则更新建议
预防：固件更新前兼容性检查 + 能力变更通知
```

## 家庭能耗管理系统完整实现

```python
class HomeEnergyManager:
    """家庭能耗管理：实时追踪 + 太阳能优化 + 储能调度"""

    def track_realtime_consumption(self, home_id):
        """实时能耗追踪"""
        # 1. 汇总所有设备实时功率
        devices = self.db.query(
            "SELECT * FROM devices WHERE home_id = %s AND type IN "
            "('ac', 'heater', 'light', 'washer', 'ev_charger')", home_id)

        total_power_w = 0
        device_power = []
        for device in devices:
            shadow = self.device_shadow.get_shadow(device["id"])
            power = shadow.get("reported", {}).get("power_consumption_w", 0)
            total_power_w += power
            device_power.append({"device_id": device["id"],
                "name": device["name"], "power_w": power})

        # 2. 太阳能发电量
        solar_output = self._get_solar_output(home_id)

        # 3. 净功率 = 消耗 - 发电
        net_power = total_power_w - solar_output

        # 4. 存储实时数据
        self.timeseries.write("home_energy", {
            "home_id": home_id,
            "total_consumption_w": total_power_w,
            "solar_output_w": solar_output,
            "net_power_w": net_power,
            "timestamp": now()
        })

        return {
            "total_consumption_w": total_power_w,
            "solar_output_w": solar_output,
            "net_power_w": net_power,
            "devices": device_power,
            "cost_rate": "buying" if net_power > 0 else "selling"
        }

    def optimize_battery_schedule(self, home_id):
        """储能调度：低谷充电、高峰放电"""
        battery = self.db.get_battery(home_id)
        if not battery:
            return None

        # 获取明天电价时段
        price_schedule = self._get_price_schedule(home_id)

        schedule = []
        for period in price_schedule:
            if period["rate"] == "off_peak":
                # 低谷 → 充电
                schedule.append({
                    "time": period["start"],
                    "action": "charge",
                    "target_soc": 100,
                    "reason": "低谷充电"
                })
            elif period["rate"] == "peak":
                # 高峰 → 放电
                schedule.append({
                    "time": period["start"],
                    "action": "discharge",
                    "min_soc": 20,
                    "reason": "高峰放电节省电费"
                })

        return schedule

    def _get_solar_output(self, home_id):
        """获取太阳能实时发电量"""
        home = self.db.get_home(home_id)
        if not home.get("has_solar"):
            return 0
        # 基于天气和时间的简单估算
        hour = now().hour
        if 6 <= hour <= 18:
            # 白天：根据天气调整
            weather = self.weather_api.get_current(home["city"])
            cloud_factor = {"sunny": 1.0, "cloudy": 0.4, "rainy": 0.15}
            peak_kw = home["solar_peak_kw"]
            # 日出日落曲线
            hour_factor = max(0, 1 - abs(hour - 12) / 6)
            return int(peak_kw * 1000 * hour_factor *
                cloud_factor.get(weather["condition"], 0.5))
        return 0
```

## 多协议网关桥接

```python
class ProtocolBridgeService:
    """多协议网关桥接：统一 MQTT 接入"""

    ADAPTERS = {
        "zigbee": ZigbeeAdapter(),
        "ble": BLEAdapter(),
        "wifi": WiFiAdapter(),
    }

    def register_device(self, device_info):
        """注册设备（自动选择协议适配器）"""
        protocol = device_info["protocol"]
        adapter = self.ADAPTERS.get(protocol)
        if not adapter:
            raise UnsupportedProtocolError(f"不支持的协议: {protocol}")

        # 1. 协议层配对
        adapter.pair(device_info)

        # 2. 创建统一设备模型
        device_id = str(uuid4())
        capabilities = adapter.discover_capabilities(device_info["address"])

        self.db.insert("devices", {
            "id": device_id,
            "home_id": device_info["home_id"],
            "name": device_info["name"],
            "type": device_info["type"],
            "protocol": protocol,
            "address": device_info["address"],
            "capabilities": json.dumps(capabilities),
            "status": "online",
            "registered_at": now()
        })

        # 3. 订阅设备事件（统一转到 MQTT）
        adapter.subscribe(device_info["address"],
            lambda event: self._on_device_event(device_id, event))

        return device_id

    def _on_device_event(self, device_id, event):
        """设备事件统一处理"""
        # 转换为统一格式并发布到 MQTT
        unified_event = {
            "device_id": device_id,
            "event_type": event["type"],
            "data": event["data"],
            "timestamp": now().isoformat()
        }
        self.mqtt_client.publish(f"device/events/{device_id}",
            json.dumps(unified_event), qos=1)
```

## 异常场景补充

### 场景：能耗数据采集延迟

```
触发：设备上报频率降低 → 能耗数据不连续 → 计费不准
检测：
  1. 设备心跳间隔 > 2 分钟 → 告警
  2. 能耗数据缺失 > 5 分钟 → 数据不连续
处理：
  1. 使用插值填补缺失数据（线性插值）
  2. 标记插值数据为"估算"
  3. 设备恢复后使用实际数据修正
预防：设备心跳监控 + 数据插值 + 恢复后修正
```

### 场景：网关协议翻译错误

```
触发：Zigbee 属性值翻译为 MQTT 时类型错误 → 设备状态异常
检测：
  1. 翻译后的值超出合理范围 → 翻译错误
  2. 设备状态频繁在 0/1 切换 → 可能是布尔值翻译反了
处理：
  1. 回滚到上一个已知正确的翻译配置
  2. 通知管理员修复翻译规则
  3. 修复后重新验证所有设备状态
预防：翻译规则版本化 + 翻译结果校验 + 灰度发布
```

## 家庭安防自动化完整实现

```python
class HomeSecurityAutomation:
    """家庭安防自动化：入侵检测 + 报警升级 + 视频验证"""

    ALARM_ESCALATION = [
        {"delay": 0, "action": "push_notification", "description": "推送通知"},
        {"delay": 30, "action": "phone_call", "description": "电话呼叫"},
        {"delay": 60, "action": "emergency_contact", "description": "紧急联系人"},
        {"delay": 120, "action": "police_alert", "description": "报警"},
    ]

    def detect_intrusion(self, home_id, sensor_event):
        """检测入侵"""
        # 1. 确认是否在布防模式
        security = self.db.get_security_config(home_id)
        if security["mode"] == "disarmed":
            return {"action": "ignore", "reason": "未布防"}

        # 2. 排除家庭成员
        members_home = self.redis.smembers(f"members_home:{home_id}")
        if sensor_event.get("user_id") in members_home:
            return {"action": "ignore", "reason": "家庭成员"}

        # 3. 宠物检测（减少误报）
        if sensor_event["sensor_type"] == "motion":
            has_pet = security.get("has_pet", False)
            if has_pet and sensor_event.get("motion_size") == "small":
                return {"action": "ignore", "reason": "宠物活动"}

        # 4. 确认入侵 → 触发报警
        alarm_id = str(uuid4())
        self.db.insert("security_alarms", {
            "alarm_id": alarm_id, "home_id": home_id,
            "trigger_sensor": sensor_event["sensor_id"],
            "trigger_type": sensor_event["sensor_type"],
            "status": "triggered",
            "triggered_at": now()
        })

        # 5. 启动报警升级链
        self._start_escalation(alarm_id, home_id)

        # 6. 启动视频验证
        self._start_video_verification(alarm_id, home_id)

        return {"alarm_id": alarm_id, "action": "alarm_triggered"}

    def _start_escalation(self, alarm_id, home_id):
        """启动报警升级链"""
        for step in self.ALARM_ESCALATION:
            self.scheduler.schedule(
                run_date=now() + timedelta(seconds=step["delay"]),
                task=self._execute_escalation_step,
                args={"alarm_id": alarm_id, "home_id": home_id,
                      "action": step["action"]})

    def _execute_escalation_step(self, alarm_id, home_id, action):
        """执行升级步骤（如果报警未解除）"""
        alarm = self.db.get_alarm(alarm_id)
        if alarm["status"] == "dismissed":
            return  # 已解除，停止升级

        if action == "push_notification":
            self.notify_home_members(home_id, "检测到入侵，请确认是否安全")
        elif action == "phone_call":
            self.call_service.call(home_id, "检测到入侵，请确认是否安全")
        elif action == "emergency_contact":
            contacts = self.db.get_emergency_contacts(home_id)
            for contact in contacts:
                self.call_service.call_contact(contact["phone"],
                    f"{home_id} 检测到入侵")
        elif action == "police_alert":
            self.alert_police(home_id)

    def disarm_alarm(self, alarm_id, user_id, method="app"):
        """解除报警"""
        self.db.update("security_alarms",
            {"status": "dismissed", "dismissed_by": user_id,
             "dismiss_method": method, "dismissed_at": now()},
            {"alarm_id": alarm_id})
```

## 房间级自动化管理

```python
class RoomAutomationService:
    """房间级自动化管理"""

    def get_room_status(self, room_id):
        """获取房间状态聚合"""
        devices = self.db.query(
            "SELECT * FROM devices WHERE room_id = %s", room_id)

        status = {
            "occupied": False,
            "active_device_count": 0,
            "total_power_w": 0,
            "lights_on": 0,
            "ac_running": False,
            "temperature": None,
        }

        for device in devices:
            shadow = self.device_shadow.get_shadow(device["id"])
            reported = shadow.get("reported", {})

            if reported.get("power") == "on":
                status["active_device_count"] += 1
                status["total_power_w"] += reported.get("power_consumption_w", 0)

            if device["type"] == "light" and reported.get("power") == "on":
                status["lights_on"] += 1
            if device["type"] == "ac" and reported.get("power") == "on":
                status["ac_running"] = True
                status["temperature"] = reported.get("temperature")
            if device["type"] == "sensor_motion" and reported.get("motion_detected"):
                status["occupied"] = True

        return status

    def auto_off_empty_room(self, room_id):
        """自动关闭空房间设备"""
        status = self.get_room_status(room_id)

        if not status["occupied"] and status["active_device_count"] > 0:
            # 空房间有设备开着 → 延迟 15 分钟后关闭
            self.scheduler.schedule(
                run_date=now() + timedelta(minutes=15),
                task=self._turn_off_room, args={"room_id": room_id})
```

## 异常场景补充

### 场景：安防误报（宠物触发）

```
触发：宠物在布防区域活动 → 触发入侵检测 → 误报
检测：
  1. 入侵报警后 30 秒内用户解除 → 疑似误报
  2. 同一传感器 24 小时内误报 > 3 次 → 误报率高
处理：
  1. 开启宠物模式：小体积运动忽略
  2. 调整传感器灵敏度
  3. 高误报区域建议更换宠物免疫传感器
预防：宠物检测 + 灵敏度调整 + 误报率监控
```

### 场景：房间状态判断错误

```
触发：人体传感器检测到运动 → 判断有人 → 实际无人（传感器故障）
检测：
  1. 传感器持续检测到运动 > 2 小时 → 疑似故障
  2. 与其他传感器交叉验证（门窗传感器、摄像头）
处理：
  1. 交叉验证：多个传感器一致 → 有人在
  2. 不一致 → 降低该传感器权重 + 标记故障
  3. 通知用户检查传感器
预防：多传感器交叉验证 + 传感器健康检查
```

## 设备 OTA 固件管理完整实现

```python
class DeviceOTAManager:
    """设备 OTA 固件管理：版本注册 + 灰度发布 + 自动回滚"""

    def register_firmware(self, device_model, version, binary_url, sha256,
                          release_notes, compatibility_matrix):
        """注册固件版本"""
        firmware_id = str(uuid4())
        self.db.insert("firmware_versions", {
            "firmware_id": firmware_id,
            "device_model": device_model,
            "version": version,
            "binary_url": binary_url,
            "sha256": sha256,
            "size_bytes": compatibility_matrix.get("size_bytes", 0),
            "release_notes": release_notes,
            "min_current_version": compatibility_matrix.get("min_version", "0.0.1"),
            "compatibility_matrix": json.dumps(compatibility_matrix),
            "status": "released",
            "created_at": now()
        })
        return firmware_id

    def staged_rollout(self, firmware_id, device_model, stages=None):
        """灰度发布：1% → 5% → 25% → 100%"""
        stages = stages or [
            {"percentage": 1, "hold_hours": 2, "max_error_rate": 0.05},
            {"percentage": 5, "hold_hours": 4, "max_error_rate": 0.03},
            {"percentage": 25, "hold_hours": 8, "max_error_rate": 0.02},
            {"percentage": 100, "hold_hours": 0, "max_error_rate": 0.01},
        ]

        rollout_id = str(uuid4())
        self.db.insert("ota_rollouts", {
            "rollout_id": rollout_id,
            "firmware_id": firmware_id,
            "device_model": device_model,
            "stages": json.dumps(stages),
            "current_stage": 0,
            "status": "in_progress",
            "started_at": now()
        })

        # 启动第一阶
        self._execute_stage(rollout_id, 0)
        return rollout_id

    def _execute_stage(self, rollout_id, stage_index):
        """执行发布阶段"""
        rollout = self.db.get_rollout(rollout_id)
        stages = json.loads(rollout["stages"])
        stage = stages[stage_index]

        # 获取目标设备（按百分比随机选择）
        target_devices = self.db.query(
            "SELECT * FROM devices WHERE device_model = %s "
            "AND firmware_version < (SELECT version FROM firmware_versions WHERE id = %s) "
            "AND id NOT IN (SELECT device_id FROM ota_history WHERE rollout_id = %s) "
            "ORDER BY RAND() LIMIT "
            "(SELECT COUNT(*) FROM devices WHERE device_model = %s) * %s / 100",
            rollout["device_model"], rollout["firmware_id"], rollout_id,
            rollout["device_model"], stage["percentage"])

        for device in target_devices:
            self._push_update(device["id"], rollout["firmware_id"])

        # 设置阶段检查（hold 时间后检查错误率）
        self.scheduler.schedule(
            run_date=now() + timedelta(hours=stage["hold_hours"]),
            task=self._check_stage_result,
            args={"rollout_id": rollout_id, "stage_index": stage_index})

    def _check_stage_result(self, rollout_id, stage_index):
        """检查阶段结果 → 决定继续或回滚"""
        rollout = self.db.get_rollout(rollout_id)
        stages = json.loads(rollout["stages"])
        stage = stages[stage_index]

        # 计算错误率
        total_updated = self.db.count("ota_history",
            rollout_id=rollout_id, status="completed")
        error_count = self.db.count("ota_history",
            rollout_id=rollout_id, status="failed")
        error_rate = error_count / max(total_updated, 1)

        if error_rate > stage["max_error_rate"]:
            # 错误率超标 → 自动回滚
            self._auto_rollback(rollout_id, error_rate)
        elif stage_index < len(stages) - 1:
            # 进入下一阶段
            self.db.update("ota_rollouts",
                {"current_stage": stage_index + 1},
                {"rollout_id": rollout_id})
            self._execute_stage(rollout_id, stage_index + 1)
        else:
            # 全部完成
            self.db.update("ota_rollouts",
                {"status": "completed", "completed_at": now()},
                {"rollout_id": rollout_id})

    def _auto_rollback(self, rollout_id, error_rate):
        """自动回滚"""
        self.db.update("ota_rollouts",
            {"status": "rolled_back", "rolled_back_at": now(),
             "rollback_reason": f"错误率 {error_rate:.1%} 超标"},
            {"rollout_id": rollout_id})

        # 推送旧版本到已更新的设备
        updated_devices = self.db.query(
            "SELECT device_id FROM ota_history "
            "WHERE rollout_id = %s AND status = 'completed'", rollout_id)
        for device in updated_devices:
            previous_version = self.db.get_device(device["device_id"])["previous_firmware_version"]
            self._push_rollback(device["device_id"], previous_version)

        self.alert(f"OTA {rollout_id} 自动回滚: 错误率 {error_rate:.1%}")
```

## 语音助手集成

```python
class VoiceAssistantIntegration:
    """语音助手集成：意图解析 + 设备控制"""

    INTENT_MAP = {
        "turn_on": ["打开", "开启", "开一下"],
        "turn_off": ["关闭", "关掉", "关一下"],
        "set_temperature": ["设置温度", "调到", "温度设为"],
        "activate_scene": ["执行", "启动", "打开场景"],
        "query_status": ["查询", "什么状态", "几度"],
    }

    def process_command(self, home_id, user_id, text):
        """处理语音指令"""
        # 1. 意图解析
        intent = self._parse_intent(text)
        if not intent:
            return {"response": "抱歉，我不理解您的指令"}

        # 2. 设备解析
        device = self._resolve_device(home_id, text)
        if not device:
            return {"response": "找不到对应的设备，请确认设备名称"}

        # 3. 破坏性操作确认
        if intent["action"] in ["unlock", "turn_off_security"]:
            return {"response": f"确认要{intent['action_desc']}吗？请再说一次确认",
                    "pending_confirmation": {
                        "action": intent["action"], "device_id": device["id"]}}

        # 4. 执行
        result = self._execute_action(device, intent)

        # 5. 生成响应
        return {"response": f"已{intent['action_desc']}{device['name']}",
                "executed": True}

    def _parse_intent(self, text):
        """解析意图"""
        for action, keywords in self.INTENT_MAP.items():
            for keyword in keywords:
                if keyword in text:
                    return {"action": action, "keyword": keyword,
                            "action_desc": keyword}
        return None

    def _resolve_device(self, home_id, text):
        """解析设备名称（模糊匹配）"""
        devices = self.db.query(
            "SELECT * FROM devices WHERE home_id = %s", home_id)

        # 精确匹配
        for d in devices:
            if d["name"] in text:
                return d

        # 模糊匹配
        for d in devices:
            if any(alias in text for alias in d.get("aliases", [])):
                return d

        # 房间 + 类型匹配
        for d in devices:
            room = self.db.get_room(d.get("room_id"))
            if room and room["name"] in text and d["type"] in text:
                return d

        return None
```

## 异常场景补充

### 场景：OTA 固件导致设备变砖

```
触发：固件更新后设备无法启动 → 完全无法通信
检测：
  1. 设备更新后 5 分钟无心跳 → 变砖嫌疑
  2. 同批更新设备大量离线 → 固件问题
处理：
  1. 自动暂停同版本 OTA 推送
  2. 尝试恢复模式重启（bootloader 保留）
  3. 恢复失败 → 人工更换设备
  4. 固件标记为 defective → 全量回滚
预防：固件验证（签名校验+完整性检查）+ 双分区启动 + 灰度发布
```

### 场景：语音指令误执行

```
触发：电视中"打开客厅灯"被语音助手误识别 → 意外操作
检测：
  1. 非注册用户的语音指令 → 误执行
  2. 短时间内同类指令重复 → 电视干扰
处理：
  1. 语音声纹识别：只响应注册用户
  2. 增加唤醒词（"小智，打开客厅灯"）
  3. 破坏性操作必须二次确认
预防：声纹识别 + 唤醒词 + 破坏性操作确认
```

## 自动化规则测试框架完整实现

```python
class AutomationTestFramework:
    """自动化规则测试：模拟执行 + 断言 + 回归"""

    def test_scene(self, scene_id, test_cases):
        """测试场景自动化"""
        results = []
        for tc in test_cases:
            # 1. 初始化虚拟设备状态
            self._setup_virtual_devices(tc["initial_state"])

            # 2. 触发场景
            self.scene_executor.execute(scene_id)

            # 3. 验证设备状态
            assertions_passed = True
            failures = []
            for assertion in tc["assertions"]:
                device_id = assertion["device_id"]
                expected = assertion["expected_state"]
                actual = self.virtual_device.get_state(device_id)

                if actual != expected:
                    assertions_passed = False
                    failures.append({
                        "device_id": device_id,
                        "expected": expected, "actual": actual
                    })

            results.append({
                "test_case": tc["name"],
                "passed": assertions_passed,
                "failures": failures
            })

        return {"scene_id": scene_id, "results": results,
                "passed": sum(1 for r in results if r["passed"]),
                "total": len(results)}

    def _setup_virtual_devices(self, initial_state):
        """初始化虚拟设备"""
        for device_id, state in initial_state.items():
            self.virtual_device.set_state(device_id, state)

    def run_regression_suite(self):
        """运行全量回归测试"""
        scenes = self.db.query(
            "SELECT * FROM scenes WHERE enabled = 1")
        results = []
        for scene in scenes:
            test_cases = self.db.query(
                "SELECT * FROM scene_test_cases WHERE scene_id = %s",
                scene["id"])
            if test_cases:
                result = self.test_scene(scene["id"], test_cases)
                results.append(result)

                # 测试失败 → 通知
                if not result["passed"] == result["total"]:
                    self.alert(f"场景 {scene['name']} 测试失败: "
                              f"{result['passed']}/{result['total']}")

        total_passed = sum(r["passed"] for r in results)
        total_cases = sum(r["total"] for r in results)
        return {"total_passed": total_passed, "total_cases": total_cases,
                "pass_rate": round(total_passed / total_cases, 3) if total_cases else 0}
```

## 智能家居数据分析

```python
class SmartHomeAnalytics:
    """智能家居数据分析：能耗 + 使用 + 异常"""

    def generate_daily_report(self, home_id, date):
        """生成日报"""
        # 1. 能耗分解
        devices = self.db.query(
            "SELECT * FROM devices WHERE home_id = %s", home_id)
        consumption = {}
        for d in devices:
            kwh = self.timeseries.query(
                f"SELECT SUM(power_consumption) / 1000 / 60 as kwh "
                f"FROM device_energy WHERE device_id = '{d['id']}' "
                f"AND DATE(timestamp) = '{date}'")
            if kwh and kwh[0]["kwh"]:
                consumption[d["name"]] = round(kwh[0]["kwh"], 2)

        # 2. 与昨日对比
        yesterday = (datetime.strptime(date, "%Y-%m-%d") - timedelta(days=1)).strftime("%Y-%m-%d")
        yesterday_total = self._get_total_consumption(home_id, yesterday)
        today_total = sum(consumption.values())
        change_pct = (today_total - yesterday_total) / max(yesterday_total, 0.01) * 100

        # 3. 异常检测（3σ）
        baseline = self._get_7day_baseline(home_id)
        anomaly = None
        if today_total > baseline["mean"] + 3 * baseline["std"]:
            anomaly = f"今日能耗 {today_total:.1f} kWh，显著高于 7 日均值 {baseline['mean']:.1f} kWh"

        # 4. 自动化节省估算
        automation_savings = self._estimate_automation_savings(home_id, date)

        return {
            "date": date, "total_kwh": round(today_total, 1),
            "change_from_yesterday": f"{change_pct:+.1f}%",
            "breakdown": consumption,
            "anomaly": anomaly,
            "automation_savings_kwh": automation_savings,
            "cost_estimate": round(today_total * 0.85, 2)  # 假设 0.85 元/kWh
        }

    def _estimate_automation_savings(self, home_id, date):
        """估算自动化节省的电量"""
        # 自动关闭空房间设备节省的电量
        auto_off_events = self.db.query(
            "SELECT * FROM automation_logs WHERE home_id = %s "
            "AND action = 'auto_off' AND DATE(timestamp) = %s",
            home_id, date)
        savings = 0
        for event in auto_off_events:
            # 估算：如果没关，会继续运行多久（到下一次手动操作）
            device_power = self.db.get_device(event["device_id"]).get("avg_power_w", 50)
            savings += device_power * 2 / 1000  # 假设平均节省 2 小时
        return round(savings, 2)
```

## 异常场景补充

### 场景：测试通过但生产环境失败

```
触发：自动化规则在测试环境通过 → 生产环境执行失败
检测：
  1. 场景执行日志中有 failed → 生产环境问题
  2. 虚拟设备与真实设备行为差异
处理：
  1. 检查真实设备是否支持该指令
  2. 设备协议适配器兼容性检查
  3. 修复后重新测试
预防：生产环境灰度测试 + 设备兼容性矩阵 + 实时监控
```

### 场景：网络中断影响设备控制

```
触发：家庭 WiFi 断开 → 云端无法控制设备 → 自动化失效
检测：
  1. 设备心跳超时 > 30 秒 → 网络中断
  2. 自动化规则执行失败 → 告警
处理：
  1. 关键自动化规则下沉到网关（本地执行）
  2. 网关离线模式：按预设时间表执行
  3. 网络恢复后同步状态
预防：关键规则本地化 + 网关离线模式 + 状态同步
```

## 家庭网络管理完整实现

```python
class HomeNetworkManager:
    """家庭网络管理：设备连接 + 带宽 + WiFi 优化"""

    def monitor_device_connectivity(self, home_id):
        """监控设备网络连接状态"""
        devices = self.db.query(
            "SELECT * FROM devices WHERE home_id = %s", home_id)

        status = {"online": 0, "offline": 0, "weak_signal": 0, "devices": []}
        for device in devices:
            heartbeat = self.redis.get(f"device_heartbeat:{device['id']}")
            if not heartbeat:
                device_status = "offline"
                status["offline"] += 1
            else:
                last_seen = datetime.fromisoformat(heartbeat)
                rssi = self.redis.hget(f"device_stats:{device['id']}", "rssi")
                if rssi and int(rssi) < -70:
                    device_status = "weak_signal"
                    status["weak_signal"] += 1
                else:
                    device_status = "online"
                    status["online"] += 1

            status["devices"].append({
                "id": device["id"], "name": device["name"],
                "status": device_status,
                "rssi": int(rssi) if rssi else None
            })

        return status

    def optimize_wifi_channel(self, home_id):
        """WiFi 信道优化"""
        # 1. 扫描周围 WiFi 信道使用情况
        scan_result = self.gateway.scan_wifi_channels(home_id)

        # 2. 找到最不拥挤的信道
        channel_usage = {}
        for ap in scan_result:
            ch = ap["channel"]
            channel_usage[ch] = channel_usage.get(ch, 0) + 1

        best_channel = min(channel_usage, key=channel_usage.get) if channel_usage else 6

        # 3. 切换信道
        self.gateway.set_wifi_channel(home_id, best_channel)

        return {"previous_channel": self._get_current_channel(home_id),
                "new_channel": best_channel,
                "channel_congestion": channel_usage}

    def get_bandwidth_usage(self, home_id):
        """带宽使用统计"""
        devices = self.db.query(
            "SELECT id, name, type FROM devices WHERE home_id = %s "
            "AND status = 'online'", home_id)

        usage = []
        for d in devices:
            rx_bytes = self.redis.hget(f"device_bandwidth:{d['id']}", "rx_bytes")
            tx_bytes = self.redis.hget(f"device_bandwidth:{d['id']}", "tx_bytes")
            if rx_bytes or tx_bytes:
                usage.append({
                    "device": d["name"], "type": d["type"],
                    "rx_mb": round(int(rx_bytes or 0) / 1048576, 1),
                    "tx_mb": round(int(tx_bytes or 0) / 1048576, 1)
                })

        return sorted(usage, key=lambda x: x["rx_mb"], reverse=True)
```

## 异常场景补充

### 场景：WiFi 信道切换导致设备断连

```
触发：自动切换 WiFi 信道 → 部分设备不支持新信道 → 断连
检测：
  1. 切换后设备离线数 > 切换前 → 断连
  2. 离线设备包含关键设备（门锁、传感器）→ 严重
处理：
  1. 自动回滚到原信道
  2. 逐步切换：先切 5GHz → 验证 → 再切 2.4GHz
  3. 关键设备优先在 2.4GHz 稳定信道
预防：信道切换前检查设备兼容性 + 分频段切换 + 自动回滚
```

### 场景：网关固件更新导致协议不兼容

```
触发：网关更新后 Zigbee 协议版本不兼容 → 所有 Zigbee 设备离线
检测：
  1. 更新后大量设备离线 → 协议问题
  2. 同类设备全部离线 → 系统性问题
处理：
  1. 回滚网关固件
  2. 检查协议兼容性矩阵
  3. 修复后重新灰度更新
预防：固件更新前协议兼容性检查 + 灰度更新 + 快速回滚
```

## 家庭自动化规则测试框架（增强版）

```python
from datetime import datetime
from enum import Enum
from typing import Dict, List, Optional, Any
import json

class DeviceState(Enum):
    """虚拟设备状态枚举"""
    ON = "on"
    OFF = "off"
    DIM = "dim"
    UNREACHABLE = "unreachable"
    ERROR = "error"

class VirtualDeviceMock:
    """
    虚拟设备 Mock：响应自动化指令并维护可配置的状态
    用于场景回归测试，替代真实 IoT 设备
    支持可配置的响应延迟、故障模式、状态回放
    """

    def __init__(self, device_id: str, device_type: str, initial_state: dict = None):
        self.device_id = device_id
        self.device_type = device_type  # light, thermostat, lock, curtain, etc.
        self.state = initial_state or self._default_state()
        self.command_log = []  # 记录所有接收到的指令
        self.response_delay_ms = 50  # 模拟设备响应延迟
        self.failure_mode = None  # 可配置的故障模式
        self.state_history = []  # 状态变更历史

    def _default_state(self) -> dict:
        """根据设备类型返回默认状态"""
        defaults = {
            "light": {"power": "off", "brightness": 0, "color_temp": 4000},
            "thermostat": {"power": "off", "temperature": 25, "mode": "auto"},
            "lock": {"locked": True, "battery": 80},
            "curtain": {"position": 0, "open_percent": 0},  # 0=全关, 100=全开
            "switch": {"power": "off"},
            "sensor_temp": {"temperature": 25.0, "humidity": 50},
            "sensor_motion": {"motion_detected": False},
            "camera": {"power": "on", "recording": False},
        }
        return defaults.get(self.device_type, {"power": "off"})

    def send_command(self, command: str, params: dict = None) -> dict:
        """
        模拟接收指令并更新状态
        command: "turn_on", "turn_off", "set_temperature", "set_brightness", etc.
        """
        # 记录指令日志
        self.command_log.append({
            "command": command,
            "params": params or {},
            "timestamp": datetime.utcnow().isoformat(),
        })

        # 如果设置了故障模式，返回错误
        if self.failure_mode:
            return {"success": False, "error": self.failure_mode}

        # 根据指令更新状态
        result = self._apply_command(command, params)
        return result

    def _apply_command(self, command: str, params: dict) -> dict:
        """应用指令到设备状态"""
        if command == "turn_on":
            self.state["power"] = "on"
            if self.device_type == "light":
                self.state["brightness"] = 100
            self._record_state_change(command)
            return {"success": True, "state": self.state}

        elif command == "turn_off":
            self.state["power"] = "off"
            if self.device_type == "light":
                self.state["brightness"] = 0
            self._record_state_change(command)
            return {"success": True, "state": self.state}

        elif command == "set_temperature":
            temp = params.get("temperature", 25)
            self.state["temperature"] = temp
            self.state["power"] = "on"
            self._record_state_change(command)
            return {"success": True, "state": self.state}

        elif command == "set_brightness":
            brightness = params.get("brightness", 100)
            self.state["brightness"] = brightness
            if brightness > 0:
                self.state["power"] = "on"
            self._record_state_change(command)
            return {"success": True, "state": self.state}

        elif command == "lock":
            self.state["locked"] = True
            self._record_state_change(command)
            return {"success": True, "state": self.state}

        elif command == "unlock":
            self.state["locked"] = False
            self._record_state_change(command)
            return {"success": True, "state": self.state}

        elif command == "set_position":
            position = params.get("position", 0)
            self.state["open_percent"] = position
            self._record_state_change(command)
            return {"success": True, "state": self.state}

        return {"success": False, "error": f"未知指令: {command}"}

    def _record_state_change(self, command: str):
        """记录状态变更历史"""
        self.state_history.append({
            "command": command,
            "state": dict(self.state),
            "timestamp": datetime.utcnow().isoformat(),
        })

    def get_state(self) -> dict:
        """获取当前设备状态"""
        return {"device_id": self.device_id, "type": self.device_type, **self.state}

    def set_failure_mode(self, mode: str):
        """设置故障模式（模拟设备故障）"""
        self.failure_mode = mode

    def reset(self):
        """重置为默认状态"""
        self.state = self._default_state()
        self.command_log = []
        self.failure_mode = None
        self.state_history = []

    def get_command_log(self) -> List[dict]:
        """获取指令日志"""
        return self.command_log


class SceneAssertionFramework:
    """
    场景断言框架：验证场景触发后的设备状态
    示例: "客厅灯应该打开", "空调温度应该设为26度"
    """

    def __init__(self, virtual_devices: Dict[str, VirtualDeviceMock]):
        self.virtual_devices = virtual_devices

    def assert_device_state(self, device_id: str, expected_state: dict, message: str = ""):
        """断言设备状态"""
        device = self.virtual_devices.get(device_id)
        if not device:
            raise AssertionError(f"设备 {device_id} 不存在. {message}")

        current = device.get_state()
        for key, expected_value in expected_state.items():
            actual_value = current.get(key)
            if actual_value != expected_value:
                raise AssertionError(
                    f"断言失败: 设备 {device_id} 的 {key} 期望 {expected_value}, "
                    f"实际 {actual_value}. {message}"
                )

    def assert_device_on(self, device_id: str, message: str = ""):
        """断言设备已打开"""
        self.assert_device_state(device_id, {"power": "on"}, message or f"{device_id} 应该打开")

    def assert_device_off(self, device_id: str, message: str = ""):
        """断言设备已关闭"""
        self.assert_device_state(device_id, {"power": "off"}, message or f"{device_id} 应该关闭")

    def assert_temperature(self, device_id: str, expected_temp: int, message: str = ""):
        """断言温度设置"""
        self.assert_device_state(
            device_id, {"temperature": expected_temp},
            message or f"{device_id} 温度应该设为 {expected_temp} 度"
        )

    def assert_device_locked(self, device_id: str, message: str = ""):
        """断言门锁已锁"""
        self.assert_device_state(device_id, {"locked": True}, message or f"{device_id} 应该已锁")

    def assert_command_received(self, device_id: str, command: str, message: str = ""):
        """断言设备收到了指定指令"""
        device = self.virtual_devices.get(device_id)
        if not device:
            raise AssertionError(f"设备 {device_id} 不存在. {message}")
        commands = [log["command"] for log in device.get_command_log()]
        if command not in commands:
            raise AssertionError(
                f"断言失败: 设备 {device_id} 未收到指令 '{command}', "
                f"已收到: {commands}. {message}"
            )

    def assert_no_command_received(self, device_id: str, command: str, message: str = ""):
        """断言设备未收到指定指令"""
        device = self.virtual_devices.get(device_id)
        if not device:
            return
        commands = [log["command"] for log in device.get_command_log()]
        if command in commands:
            raise AssertionError(
                f"断言失败: 设备 {device_id} 不应收到指令 '{command}'. {message}"
            )


class SceneSimulationEngine:
    """场景模拟引擎：在虚拟设备上执行场景"""

    def __init__(self, virtual_devices: Dict[str, VirtualDeviceMock], db):
        self.virtual_devices = virtual_devices
        self.db = db

    def simulate_scene(self, scene_id: str, trigger_type: str = "manual", trigger_params: dict = None):
        """在虚拟设备上执行场景"""
        scene = self.db.get("SELECT * FROM scenes WHERE scene_id = %s", scene_id)
        if not scene:
            raise ValueError(f"场景 {scene_id} 不存在")

        self._simulate_trigger(trigger_type, trigger_params)

        actions = json.loads(scene["actions"])
        results = []
        for action in actions:
            device_id = action["device_id"]
            command = action["command"]
            params = action.get("params", {})
            device = self.virtual_devices.get(device_id)
            if device:
                result = device.send_command(command, params)
                results.append({
                    "device_id": device_id, "command": command,
                    "params": params, "result": result,
                })

        return {
            "scene_id": scene_id,
            "scene_name": scene["name"],
            "trigger_type": trigger_type,
            "actions_executed": len(results),
            "results": results,
        }

    def _simulate_trigger(self, trigger_type: str, trigger_params: dict):
        """模拟触发条件"""
        if trigger_type == "sensor" and trigger_params:
            device_id = trigger_params.get("device_id")
            sensor_state = trigger_params.get("state", {})
            device = self.virtual_devices.get(device_id)
            if device:
                device.state.update(sensor_state)


class SceneRegressionTestSuite:
    """场景回归测试套件：运行所有场景并验证预期结果"""

    def __init__(self, db, result_reporter=None):
        self.db = db
        self.result_reporter = result_reporter or TestResultReporter()

    def run_all_scenes(self, home_id: str) -> dict:
        """运行该家庭的所有场景回归测试"""
        scenes = self.db.query(
            "SELECT * FROM scenes WHERE home_id = %s AND status = 'active'",
            home_id
        )

        total = len(scenes)
        passed = 0
        failed = 0
        results = []

        for scene in scenes:
            virtual_devices = self._create_virtual_devices(home_id)
            engine = SceneSimulationEngine(virtual_devices, self.db)
            assertion = SceneAssertionFramework(virtual_devices)

            try:
                sim_result = engine.simulate_scene(scene["scene_id"])
                expected = json.loads(scene.get("expected_results", "{}"))
                self._verify_expected(assertion, expected)
                results.append({
                    "scene_id": scene["scene_id"],
                    "scene_name": scene["name"],
                    "status": "passed",
                    "actions_executed": sim_result["actions_executed"],
                })
                passed += 1
            except AssertionError as e:
                results.append({
                    "scene_id": scene["scene_id"],
                    "scene_name": scene["name"],
                    "status": "failed",
                    "error": str(e),
                })
                failed += 1
            except Exception as e:
                results.append({
                    "scene_id": scene["scene_id"],
                    "scene_name": scene["name"],
                    "status": "error",
                    "error": str(e),
                })
                failed += 1

        report = self.result_reporter.generate_report(home_id, results, passed, failed, total)
        return report

    def _create_virtual_devices(self, home_id: str) -> Dict[str, VirtualDeviceMock]:
        """为家庭创建虚拟设备"""
        devices = self.db.query("SELECT * FROM devices WHERE home_id = %s", home_id)
        virtual = {}
        for d in devices:
            virtual[d["device_id"]] = VirtualDeviceMock(
                device_id=d["device_id"], device_type=d["device_type"]
            )
        return virtual

    def _verify_expected(self, assertion: SceneAssertionFramework, expected: dict):
        """验证场景预期结果"""
        for device_id, expected_state in expected.items():
            for key, value in expected_state.items():
                assertion.assert_device_state(device_id, {key: value})


class TestResultReporter:
    """测试报告生成器"""

    def generate_report(self, home_id: str, results: list, passed: int, failed: int, total: int) -> dict:
        """生成测试报告"""
        report = {
            "home_id": home_id,
            "test_type": "scene_regression",
            "total": total, "passed": passed, "failed": failed,
            "pass_rate": f"{passed/total*100:.1f}%" if total > 0 else "N/A",
            "results": results,
            "generated_at": datetime.utcnow().isoformat(),
        }
        if failed > 0:
            failed_scenes = [r for r in results if r["status"] != "passed"]
            report["attention_required"] = True
            report["failed_scenes"] = failed_scenes
        return report
```

## 智能家居数据分析（增强版）

```python
class SmartHomeAnalyticsEnhanced:
    """智能家居数据分析增强版：能耗报告 + 设备统计 + 异常检测 + 节省估算 + 同区域对比"""

    def __init__(self, db, redis):
        self.db = db
        self.redis = redis

    def generate_energy_report(self, home_id: str, period: str = "daily"):
        """
        能耗报告生成（日/周/月）
        包含：按设备类型分解、与历史对比、费用计算
        """
        now = datetime.utcnow()
        time_ranges = {
            "daily": (now - timedelta(days=1), now),
            "weekly": (now - timedelta(weeks=1), now),
            "monthly": (now - timedelta(days=30), now),
        }
        start, end = time_ranges[period]

        # 按设备类型汇总能耗
        energy_by_type = self.db.query(
            "SELECT device_type, "
            "       SUM(energy_kwh) as total_kwh, "
            "       COUNT(DISTINCT device_id) as device_count "
            "FROM device_energy_log "
            "WHERE home_id = %s AND timestamp BETWEEN %s AND %s "
            "GROUP BY device_type ORDER BY total_kwh DESC",
            home_id, start, end
        )

        # 与昨日/上周对比
        prev_start, prev_end = self._get_previous_period(start, end, period)
        prev_total = self.db.get(
            "SELECT SUM(energy_kwh) as total_kwh "
            "FROM device_energy_log "
            "WHERE home_id = %s AND timestamp BETWEEN %s AND %s",
            home_id, prev_start, prev_end
        )
        current_total = sum(e["total_kwh"] for e in energy_by_type)
        prev_total_kwh = prev_total["total_kwh"] if prev_total and prev_total["total_kwh"] else 0
        change_pct = ((current_total - prev_total_kwh) / prev_total_kwh * 100) if prev_total_kwh > 0 else 0

        # 费用计算（阶梯电价）
        cost = self._calculate_cost(current_total)

        return {
            "home_id": home_id, "period": period,
            "start": start.isoformat(), "end": end.isoformat(),
            "total_kwh": current_total,
            "cost_cny": cost,
            "comparison": {
                "previous_period_kwh": prev_total_kwh,
                "change_pct": round(change_pct, 1),
                "trend": "up" if change_pct > 0 else "down" if change_pct < 0 else "stable",
            },
            "breakdown": energy_by_type,
        }

    def get_device_usage_statistics(self, home_id: str):
        """设备使用统计：最常用设备 + 闲置设备（>30天未用）"""
        most_used = self.db.query(
            "SELECT d.device_id, d.name, d.device_type, "
            "       COUNT(*) as usage_count, "
            "       SUM(e.energy_kwh) as total_kwh "
            "FROM device_energy_log e "
            "JOIN devices d ON e.device_id = d.device_id "
            "WHERE d.home_id = %s AND e.timestamp > NOW() - INTERVAL 7 DAY "
            "GROUP BY d.device_id ORDER BY usage_count DESC LIMIT 10",
            home_id
        )

        idle_devices = self.db.query(
            "SELECT d.device_id, d.name, d.device_type, "
            "       MAX(e.timestamp) as last_used "
            "FROM devices d "
            "LEFT JOIN device_energy_log e ON d.device_id = e.device_id "
            "WHERE d.home_id = %s "
            "GROUP BY d.device_id "
            "HAVING MAX(e.timestamp) < NOW() - INTERVAL 30 DAY "
            "   OR MAX(e.timestamp) IS NULL",
            home_id
        )

        return {
            "most_used_devices": most_used,
            "idle_devices_over_30_days": idle_devices,
            "idle_device_count": len(idle_devices),
        }

    def detect_energy_anomaly(self, home_id: str):
        """
        能耗异常检测：功率尖峰超过基线3σ
        基于过去30天数据建立每小时基线
        """
        baseline = self.db.query(
            "SELECT EXTRACT(HOUR FROM timestamp) as hour, "
            "       AVG(energy_kwh) as avg_kwh, "
            "       STDDEV(energy_kwh) as std_kwh "
            "FROM device_energy_hourly "
            "WHERE home_id = %s AND timestamp > NOW() - INTERVAL 30 DAY "
            "GROUP BY EXTRACT(HOUR FROM timestamp)",
            home_id
        )
        baseline_map = {b["hour"]: {"avg": b["avg_kwh"], "std": b["std_kwh"] or 0} for b in baseline}

        current_hour = datetime.utcnow().hour
        current = self.db.get(
            "SELECT SUM(energy_kwh) as total_kwh "
            "FROM device_energy_log "
            "WHERE home_id = %s AND timestamp > NOW() - INTERVAL 1 HOUR",
            home_id
        )
        current_kwh = current["total_kwh"] if current else 0

        bl = baseline_map.get(current_hour, {"avg": 0, "std": 0})
        anomaly_threshold = bl["avg"] + 3 * bl["std"]
        is_anomaly = current_kwh > anomaly_threshold and anomaly_threshold > 0

        if is_anomaly:
            self.alert(
                f"能耗异常告警: 家庭 {home_id} 当前能耗 {current_kwh:.2f} kWh, "
                f"超过3σ阈值 {anomaly_threshold:.2f} kWh"
            )

        return {
            "home_id": home_id,
            "current_kwh": current_kwh,
            "baseline_avg": bl["avg"], "baseline_std": bl["std"],
            "anomaly_threshold_3sigma": anomaly_threshold,
            "is_anomaly": is_anomaly,
        }

    def estimate_automation_savings(self, home_id: str):
        """
        自动化节省估算：与手动操作对比
        各设备类型有典型的手动浪费率
        """
        actual_kwh = self.db.get(
            "SELECT SUM(energy_kwh) as total FROM device_energy_log "
            "WHERE home_id = %s AND timestamp > NOW() - INTERVAL 30 DAY",
            home_id
        )["total"] or 0

        waste_rates = {
            "light": 0.30,       # 灯：忘记关 → 30%浪费
            "thermostat": 0.20,  # 空调：不及时调节 → 20%浪费
            "curtain": 0.10,     # 窗帘：不及时开合 → 10%浪费
        }

        estimated_manual_kwh = 0
        device_energy = self.db.query(
            "SELECT d.device_type, SUM(e.energy_kwh) as total_kwh "
            "FROM device_energy_log e "
            "JOIN devices d ON e.device_id = d.device_id "
            "WHERE d.home_id = %s AND e.timestamp > NOW() - INTERVAL 30 DAY "
            "GROUP BY d.device_type", home_id
        )

        for de in device_energy:
            waste = waste_rates.get(de["device_type"], 0.15)
            estimated_manual_kwh += de["total_kwh"] * (1 + waste)

        savings_kwh = estimated_manual_kwh - actual_kwh
        savings_pct = (savings_kwh / estimated_manual_kwh * 100) if estimated_manual_kwh > 0 else 0

        return {
            "home_id": home_id,
            "actual_kwh_30d": actual_kwh,
            "estimated_manual_kwh_30d": round(estimated_manual_kwh, 1),
            "savings_kwh": round(savings_kwh, 1),
            "savings_pct": round(savings_pct, 1),
            "savings_cost_cny": round(self._calculate_cost(savings_kwh), 2),
        }

    def compare_with_similar_homes(self, home_id: str):
        """与同区域类似家庭对比分析"""
        home = self.db.get("SELECT * FROM homes WHERE home_id = %s", home_id)
        similar_homes = self.db.query(
            "SELECT h.home_id, SUM(e.energy_kwh) as total_kwh "
            "FROM homes h JOIN device_energy_log e ON h.home_id = e.home_id "
            "WHERE h.city = %s AND h.house_type = %s "
            "AND h.area_sqm BETWEEN %s AND %s "
            "AND e.timestamp > NOW() - INTERVAL 30 DAY "
            "GROUP BY h.home_id",
            home["city"], home["house_type"],
            home["area_sqm"] * 0.8, home["area_sqm"] * 1.2
        )

        if not similar_homes:
            return {"comparison": "no_similar_homes_found"}

        kwh_values = [h["total_kwh"] for h in similar_homes if h["total_kwh"]]
        my_kwh = self.db.get(
            "SELECT SUM(energy_kwh) as total FROM device_energy_log "
            "WHERE home_id = %s AND timestamp > NOW() - INTERVAL 30 DAY",
            home_id
        )["total"] or 0

        avg_kwh = sum(kwh_values) / len(kwh_values) if kwh_values else 0
        percentile = sum(1 for v in kwh_values if v < my_kwh) / len(kwh_values) * 100 if kwh_values else 50

        return {
            "my_kwh_30d": my_kwh,
            "similar_homes_avg_kwh": round(avg_kwh, 1),
            "similar_homes_count": len(kwh_values),
            "my_percentile": round(percentile, 1),
            "vs_avg": "below" if my_kwh < avg_kwh else "above",
        }

    def _calculate_cost(self, kwh: float) -> float:
        """电费计算（阶梯电价）"""
        if kwh <= 200:
            return kwh * 0.55
        elif kwh <= 400:
            return 200 * 0.55 + (kwh - 200) * 0.60
        else:
            return 200 * 0.55 + 200 * 0.60 + (kwh - 400) * 0.85

    def _get_previous_period(self, start, end, period):
        delta = end - start
        return start - delta, start

    def alert(self, message):
        print(f"[SmartHomeAnalytics] {message}")
```

## 家庭网络管理（增强版）

```python
import random

class HomeNetworkManagementEnhanced:
    """家庭网络管理增强版：设备在线监控 + 带宽分配 + 网络质量 + WiFi优化 + 访客隔离"""

    def __init__(self, db, redis, router_api):
        self.db = db
        self.redis = redis
        self.router_api = router_api

    def monitor_device_connectivity(self, home_id: str):
        """
        设备网络连接监控：在线/离线检测
        关键设备离线超5分钟触发告警
        """
        devices = self.db.query(
            "SELECT device_id, name, device_type, ip_address, mac_address "
            "FROM devices WHERE home_id = %s AND status = 'active'",
            home_id
        )

        online_devices = []
        offline_devices = []

        for device in devices:
            is_online = self.router_api.check_device_online(home_id, device["mac_address"])
            status = "online" if is_online else "offline"
            self.db.update("devices",
                {"network_status": status, "last_heartbeat": datetime.utcnow() if is_online else None},
                {"device_id": device["device_id"]})

            if is_online:
                online_devices.append(device)
            else:
                offline_devices.append(device)
                # 关键设备离线超5分钟告警
                if device["device_type"] in ["lock", "camera", "sensor_security"]:
                    last_heartbeat = self.redis.get(f"device:heartbeat:{device['device_id']}")
                    if last_heartbeat:
                        offline_duration = (datetime.utcnow() - datetime.fromisoformat(last_heartbeat)).seconds
                        if offline_duration > 300:
                            self.alert(
                                f"关键设备离线: {device['name']}({device['device_type']}) "
                                f"已离线 {offline_duration//60} 分钟"
                            )

        return {
            "home_id": home_id,
            "total_devices": len(devices),
            "online_count": len(online_devices),
            "offline_count": len(offline_devices),
            "offline_devices": [{"device_id": d["device_id"], "name": d["name"], "type": d["device_type"]} for d in offline_devices],
        }

    def get_bandwidth_usage(self, home_id: str):
        """每设备带宽使用情况"""
        traffic_data = self.router_api.get_device_traffic(home_id)
        device_bandwidth = []
        for item in traffic_data:
            device = self.db.get(
                "SELECT device_id, name, device_type FROM devices WHERE mac_address = %s",
                item["mac_address"]
            )
            if device:
                device_bandwidth.append({
                    "device_id": device["device_id"], "name": device["name"],
                    "device_type": device["device_type"],
                    "download_kbps": item.get("download_kbps", 0),
                    "upload_kbps": item.get("upload_kbps", 0),
                    "total_kbps": item.get("download_kbps", 0) + item.get("upload_kbps", 0),
                })

        device_bandwidth.sort(key=lambda x: x["total_kbps"], reverse=True)
        return {
            "home_id": home_id, "devices": device_bandwidth,
            "total_download_kbps": sum(d["download_kbps"] for d in device_bandwidth),
            "total_upload_kbps": sum(d["upload_kbps"] for d in device_bandwidth),
            "timestamp": datetime.utcnow().isoformat(),
        }

    def get_network_quality_metrics(self, home_id: str):
        """网络质量指标：延迟、丢包率"""
        router_info = self.router_api.get_info(home_id)
        targets = ["8.8.8.8", "114.114.114.114", "223.5.5.5"]
        metrics = []

        for target in targets:
            result = self.router_api.ping(home_id, target, count=10)
            metrics.append({
                "target": target,
                "latency_ms_avg": result.get("avg_ms", 0),
                "latency_ms_max": result.get("max_ms", 0),
                "packet_loss_pct": result.get("loss_pct", 0),
            })

        avg_latency = sum(m["latency_ms_avg"] for m in metrics) / len(metrics)
        avg_loss = sum(m["packet_loss_pct"] for m in metrics) / len(metrics)

        if avg_latency < 20 and avg_loss < 0.1:
            quality = "excellent"
        elif avg_latency < 50 and avg_loss < 1:
            quality = "good"
        elif avg_latency < 100 and avg_loss < 5:
            quality = "fair"
        else:
            quality = "poor"

        return {
            "home_id": home_id, "quality": quality,
            "avg_latency_ms": round(avg_latency, 1),
            "avg_packet_loss_pct": round(avg_loss, 2),
            "details": metrics,
            "wifi_signal_dbm": router_info.get("signal_dbm", 0),
            "connected_clients": router_info.get("connected_clients", 0),
        }

    def auto_optimize_wifi_channel(self, home_id: str):
        """自动WiFi信道优化：扫描干扰 + 切换最优信道"""
        scan_result = self.router_api.scan_wifi_channels(home_id)
        channel_usage = {}
        for ap in scan_result.get("nearby_aps", []):
            ch = ap["channel"]
            channel_usage[ch] = channel_usage.get(ch, 0) + ap.get("signal_strength", 0)

        band_channels = {
            "2.4ghz": [1, 6, 11],
            "5ghz": [36, 40, 44, 48, 149, 153, 157, 161],
        }

        optimal = {}
        for band, channels in band_channels.items():
            best_channel = min(channels, key=lambda ch: channel_usage.get(ch, 0))
            current_channel = self.router_api.get_current_channel(home_id, band)
            optimal[band] = {
                "current_channel": current_channel,
                "recommended_channel": best_channel,
                "needs_change": current_channel != best_channel,
            }
            if current_channel != best_channel:
                self.router_api.set_channel(home_id, band, best_channel)
                self.log(f"WiFi {band} 信道优化: {current_channel} → {best_channel}")

        return {
            "home_id": home_id, "optimization": optimal,
            "nearby_ap_count": len(scan_result.get("nearby_aps", [])),
            "optimized_at": datetime.utcnow().isoformat(),
        }

    def manage_guest_network(self, home_id: str, action: str, settings: dict = None):
        """
        访客网络隔离管理
        访客网络与主网络完全隔离，防止访客访问IoT设备
        """
        if action == "enable":
            guest_settings = {
                "ssid": settings.get("ssid", f"Guest_{home_id[-4:]}"),
                "password": settings.get("password", self._generate_guest_password()),
                "isolation": True,
                "main_network_access": False,
                "bandwidth_limit_kbps": settings.get("bandwidth_limit_kbps", 5000),
                "schedule": settings.get("schedule", None),
            }
            self.router_api.enable_guest_network(home_id, guest_settings)
            self.db.update("home_network_settings",
                {"guest_network_enabled": True, "guest_settings": json.dumps(guest_settings)},
                {"home_id": home_id})
            return {"status": "enabled", "settings": guest_settings}

        elif action == "disable":
            self.router_api.disable_guest_network(home_id)
            self.db.update("home_network_settings",
                {"guest_network_enabled": False}, {"home_id": home_id})
            return {"status": "disabled"}

        elif action == "configure":
            if settings:
                self.router_api.update_guest_network(home_id, settings)
            return {"status": "configured", "settings": settings}

    def _generate_guest_password(self) -> str:
        import string
        chars = string.ascii_letters + string.digits
        return ''.join(random.choices(chars, k=8))

    def alert(self, message):
        print(f"[HomeNetwork] {message}")

    def log(self, message):
        print(f"[HomeNetwork] {message}")
```

## 异常场景补充（增强版）

### 场景：自动化规则测试在模拟中通过但生产环境失败

```
触发：场景回归测试全部通过，但部署到生产环境后场景执行失败
原因：
  1. 虚拟设备 Mock 返回固定成功，而真实设备可能离线/超时/固件版本不同
  2. 模拟环境中网络延迟为50ms，而真实Zigbee/WiFi延迟可能200ms+且不稳定
  3. 虚拟设备状态同步是即时的，而真实设备状态上报有延迟
  4. 模拟环境没有考虑设备并发控制（同一设备同时收到多个指令）
检测：
  1. 生产环境场景执行失败率 > 5% → 与测试环境差异大
  2. 设备状态查询超时 → 真实设备响应慢
  3. 场景执行后设备状态与预期不符 → 状态同步延迟
处理：
  1. 增强 Mock 真实度：添加随机延迟（50-500ms）、随机失败率（1-5%）
  2. 场景执行增加重试机制：设备指令失败后重试3次
  3. 增加状态确认等待：指令执行后等待500ms再检查状态
  4. 引入灰度测试：新场景先在1%的真实设备上验证
  5. 真实设备影子测试：同时发给虚拟设备和真实设备，对比结果
预防：
  - Mock 模拟真实网络条件（延迟、丢包、超时）
  - 场景测试增加失败和超时的用例
  - 生产环境持续监控场景成功率
  - 真实设备集成测试作为上线前必经环节
```

### 场景：网络连接中断影响设备控制

```
触发：家庭路由器故障/ISP 断网 → 所有云连接的IoT设备无法远程控制
影响：
  - 云端场景无法触发（如定时开空调）
  - 远程控制指令无法到达设备
  - 设备状态无法上报到云端
  - 语音助手（云端）无法使用
检测：
  1. 路由器心跳超时 > 60 秒 → 网络中断
  2. 设备离线数量突增 → 可能网络故障
  3. 网关上报连接状态为 disconnected
处理：
  1. 本地网关降级：Zigbee网关保留本地控制能力
     - 定时场景由网关本地执行（已下发的定时规则缓存）
     - 局域网内设备可直连控制（蓝牙/局域网协议）
  2. 云端检测到断网后通知用户
     - 推送："您的家庭网络已断开，远程控制暂不可用"
     - 提供手机热点接入方案
  3. 网络恢复后自动同步
     - 设备状态批量上报
     - 错过执行的定时场景按策略补执行或跳过
  4. 关键设备优先恢复（门锁、安防摄像头）
  5. 网络冗余：支持4G备用链路（高端网关）
预防：
  - 关键场景下发给网关本地执行（不依赖云端）
  - 网关具备本地定时执行能力
  - 网络健康度持续监控和预警
  - 支持 MQTT 断线重连和消息缓存
  - 网关定期心跳检测，网络异常提前告警
```

## 智能家居场景联动引擎完整实现

```python
class SceneAutomationEngine:
    """场景联动引擎：条件触发 + 动作编排 + 冲突检测"""

    OPERATORS = {
        "eq": lambda a, b: a == b,
        "gt": lambda a, b: a > b,
        "lt": lambda a, b: a < b,
        "gte": lambda a, b: a >= b,
        "lte": lambda a, b: a <= b,
        "between": lambda a, b: b[0] <= a <= b[1],
    }

    def evaluate_trigger(self, trigger_event):
        """评估触发条件"""
        device_id = trigger_event["device_id"]
        property_name = trigger_event["property"]
        new_value = trigger_event["value"]

        # 查找匹配的自动化规则
        rules = self.db.query(
            "SELECT * FROM automation_rules "
            "WHERE status = 'active' "
            "AND (device_id = %s OR device_id IS NULL) "
            "ORDER BY priority DESC", device_id)

        triggered_rules = []
        for rule in rules:
            conditions = json.loads(rule["conditions"])

            # 检查所有条件是否满足
            all_met = True
            for condition in conditions:
                if not self._evaluate_condition(condition, trigger_event):
                    all_met = False
                    break

            if all_met:
                triggered_rules.append(rule)

        if not triggered_rules:
            return {"triggered": False}

        # 检测规则间冲突
        conflicts = self._detect_rule_conflicts(triggered_rules)
        if conflicts:
            # 优先级高的规则胜出
            triggered_rules = self._resolve_conflicts(triggered_rules, conflicts)

        # 执行动作
        results = []
        for rule in triggered_rules:
            result = self._execute_actions(rule, trigger_event)
            results.append(result)

        return {"triggered": True, "rules_triggered": len(triggered_rules),
                "results": results, "conflicts_resolved": len(conflicts)}

    def _evaluate_condition(self, condition, event):
        """评估单个条件"""
        # 设备属性条件
        if condition["type"] == "device_property":
            device = self.db.get_device(condition["device_id"])
            value = device.get(condition["property"])
            target = condition["value"]
            operator = condition.get("operator", "eq")
            return self.OPERATORS[operator](value, target)

        # 时间条件
        elif condition["type"] == "time":
            current_hour = now().hour
            return condition["start_hour"] <= current_hour < condition["end_hour"]

        # 天气条件
        elif condition["type"] == "weather":
            weather = self.weather_service.get_current()
            return weather.get(condition["property"]) == condition["value"]

        # 用户位置条件
        elif condition["type"] == "user_location":
            user_location = self._get_user_location(condition["user_id"])
            if condition["location_type"] == "home":
                return self._is_at_home(user_location)
            elif condition["location_type"] == "away":
                return not self._is_at_home(user_location)

        return False

    def _detect_rule_conflicts(self, rules):
        """检测规则冲突（对同一设备设置矛盾状态）"""
        conflicts = []
        device_actions = {}

        for rule in rules:
            actions = json.loads(rule["actions"])
            for action in actions:
                if action["type"] == "set_property":
                    device_id = action["device_id"]
                    prop = action["property"]

                    key = f"{device_id}:{prop}"
                    if key in device_actions:
                        # 同一设备同一属性 → 不同值 → 冲突
                        if device_actions[key]["value"] != action["value"]:
                            conflicts.append({
                                "rule_a": device_actions[key]["rule_id"],
                                "rule_b": rule["id"],
                                "device": device_id,
                                "property": prop,
                                "value_a": device_actions[key]["value"],
                                "value_b": action["value"]
                            })
                    else:
                        device_actions[key] = {
                            "rule_id": rule["id"],
                            "value": action["value"]
                        }

        return conflicts

    def _resolve_conflicts(self, rules, conflicts):
        """解决冲突（优先级高的胜出）"""
        conflict_rule_ids = set()
        for c in conflicts:
            # 优先级低的规则被排除
            rule_a = next(r for r in rules if r["id"] == c["rule_a"])
            rule_b = next(r for r in rules if r["id"] == c["rule_b"])

            if rule_a["priority"] >= rule_b["priority"]:
                conflict_rule_ids.add(c["rule_b"])
            else:
                conflict_rule_ids.add(c["rule_a"])

        return [r for r in rules if r["id"] not in conflict_rule_ids]

    def _execute_actions(self, rule, trigger_event):
        """执行自动化动作"""
        actions = json.loads(rule["actions"])
        results = []

        for action in actions:
            if action["type"] == "set_property":
                # 控制设备
                result = self.device_service.set_property(
                    action["device_id"], action["property"], action["value"])
                results.append(result)

            elif action["type"] == "delay":
                # 延迟执行
                self.scheduler.schedule_action(
                    rule["id"], action["delay_seconds"],
                    action["subsequent_actions"])

            elif action["type"] == "notify":
                # 发送通知
                self.notification.send(action["user_id"],
                    action["message"], "automation")

            elif action["type"] == "scene":
                # 激活场景
                self.scene_service.activate_scene(action["scene_id"])

        self.db.insert("automation_execution_log", {
            "rule_id": rule["id"],
            "trigger_event": json.dumps(trigger_event),
            "actions": json.dumps(actions),
            "results": json.dumps(results),
            "executed_at": now()
        })

        return {"rule_id": rule["id"], "actions_executed": len(results)}
```

## 异常场景补充

### 场景：联动规则循环触发

```
触发：规则 A 开灯 → 规则 B 检测灯开 → 关灯 → 规则 A 检测灯关 → 开灯 → 循环
检测：
  1. 同一规则 1 分钟内触发 > 3 次 → 可能循环
  2. 设备状态频繁切换 → 循环触发
处理：
  1. 规则执行后设置冷却期（30 秒内不再触发）
  2. 检测循环并自动禁用规则
  3. 通知用户修改规则
预防：规则冷却期 + 循环检测 + 触发频率限制
```

### 场景：联动动作执行失败

```
触发：自动开空调 → 空调离线 → 动作失败 → 用户回家后室温不适
检测：
  1. 设备离线 → 动作会失败
  2. 动作执行返回 error → 失败
处理：
  1. 动作失败 → 通知用户
  2. 提供备选动作（如开风扇替代空调）
  3. 设备上线后补执行
预防：动作失败通知 + 备选动作 + 设备上线补执行
```

## 智能家居设备影子与状态同步完整实现

```python
class DeviceShadowService:
    """设备影子：期望状态 vs 报告状态 + 离线指令队列 + 状态差异检测"""

    def update_desired_state(self, device_id, desired_state, source="user"):
        """更新期望状态"""
        # 1. 获取当前影子
        shadow = self._get_shadow(device_id)

        # 2. 合并期望状态（部分更新）
        if shadow.get("desired"):
            merged_desired = {**shadow["desired"], **desired_state}
        else:
            merged_desired = desired_state

        # 3. 保存影子
        self.redis.set(f"device_shadow:{device_id}", json.dumps({
            "desired": merged_desired,
            "reported": shadow.get("reported", {}),
            "metadata": {
                "desired_updated_at": now().isoformat(),
                "desired_updated_by": source,
                "reported_updated_at": shadow.get("metadata", {}).get("reported_updated_at"),
            },
            "version": shadow.get("version", 0) + 1
        }))

        # 4. 设备在线 → 立即下发
        if self._is_device_online(device_id):
            delta = self._compute_delta(shadow.get("reported", {}), merged_desired)
            if delta:
                self._send_commands(device_id, delta)
        else:
            # 设备离线 → 入队列，上线后推送
            for key, value in desired_state.items():
                self.redis.rpush(f"device_pending:{device_id}",
                    json.dumps({"property": key, "value": value, "queued_at": now().isoformat()}))

        # 5. 记录变更日志
        self.db.insert("device_shadow_log", {
            "log_id": str(uuid4()),
            "device_id": device_id,
            "action": "update_desired",
            "source": source,
            "changes": json.dumps(desired_state),
            "created_at": now()
        })

        return {"device_id": device_id, "desired": merged_desired,
                "device_online": self._is_device_online(device_id)}

    def update_reported_state(self, device_id, reported_state):
        """设备上报当前状态"""
        shadow = self._get_shadow(device_id)

        # 1. 更新报告状态
        merged_reported = {**shadow.get("reported", {}), **reported_state}

        # 2. 保存影子
        self.redis.set(f"device_shadow:{device_id}", json.dumps({
            "desired": shadow.get("desired", {}),
            "reported": merged_reported,
            "metadata": {
                "desired_updated_at": shadow.get("metadata", {}).get("desired_updated_at"),
                "reported_updated_at": now().isoformat(),
            },
            "version": shadow.get("version", 0) + 1
        }))

        # 3. 检查期望状态与报告状态的差异
        if shadow.get("desired"):
            delta = self._compute_delta(merged_reported, shadow["desired"])
            if delta:
                # 报告状态与期望不一致 → 重发指令
                self._send_commands(device_id, delta)

                # 超过 3 次仍不一致 → 告警
                retry_count = self.redis.incr(f"shadow_retry:{device_id}")
                self.redis.expire(f"shadow_retry:{device_id}", 300)
                if retry_count > 3:
                    self.alert(f"设备 {device_id} 状态同步失败: 期望 {delta}")
                    self.redis.delete(f"shadow_retry:{device_id}")

        # 4. 触发自动化规则
        self.automation_engine.evaluate_trigger({
            "device_id": device_id,
            "property": list(reported_state.keys())[0] if reported_state else None,
            "value": list(reported_state.values())[0] if reported_state else None
        })

        return {"device_id": device_id, "reported": merged_reported}

    def get_shadow(self, device_id):
        """获取设备影子"""
        shadow = self._get_shadow(device_id)

        if not shadow:
            return {"device_id": device_id, "desired": {}, "reported": {},
                    "delta": {}, "version": 0}

        delta = self._compute_delta(shadow.get("reported", {}),
                                    shadow.get("desired", {}))

        return {
            "device_id": device_id,
            "desired": shadow.get("desired", {}),
            "reported": shadow.get("reported", {}),
            "delta": delta,
            "version": shadow.get("version", 0),
            "metadata": shadow.get("metadata", {})
        }

    def handle_device_online(self, device_id):
        """设备上线 → 推送离线期间积压的指令"""
        # 1. 推送期望状态
        shadow = self._get_shadow(device_id)
        if shadow and shadow.get("desired"):
            delta = self._compute_delta(shadow.get("reported", {}), shadow["desired"])
            if delta:
                self._send_commands(device_id, delta)

        # 2. 推送离线队列
        pending = self.redis.lrange(f"device_pending:{device_id}", 0, -1)
        for cmd_json in pending:
            cmd = json.loads(cmd_json)
            self._send_commands(device_id, {cmd["property"]: cmd["value"]})

        self.redis.delete(f"device_pending:{device_id}")

        return {"device_id": device_id, "pending_commands_sent": len(pending)}

    def _compute_delta(self, reported, desired):
        """计算期望与报告的差异"""
        delta = {}
        for key, value in desired.items():
            if key not in reported or reported[key] != value:
                delta[key] = value
        return delta

    def _get_shadow(self, device_id):
        """从 Redis 获取影子"""
        data = self.redis.get(f"device_shadow:{device_id}")
        return json.loads(data) if data else None

    def _is_device_online(self, device_id):
        """检查设备是否在线"""
        return self.redis.exists(f"device_online:{device_id}")
```

## 异常场景补充

### 场景：设备影子与实际状态持续不一致

```
触发：期望温度 25°C → 设备报告 24°C → 重发指令 → 仍报告 24°C → 死循环
检测：
  1. 影子重试次数 > 3 → 持续不一致
  2. 期望值与报告值差距始终不变 → 设备无法达到目标
处理：
  1. 停止重试，标记设备异常
  2. 通知用户设备可能故障
  3. 接受设备实际能力范围内的值
预防：重试上限 + 异常标记 + 容差范围
```

### 场景：离线指令队列积压

```
触发：设备离线 7 天 → 积压 1000 条指令 → 设备上线后全部推送 → 设备过载 → 重启
检测：
  1. 离线指令队列长度 > 100 → 积压过多
  2. 设备上线后立即掉线 → 过载
处理：
  1. 只推送最新状态（丢弃中间状态）
  2. 队列合并（同一属性只保留最新值）
  3. 分批推送（每秒最多 5 条）
预防：队列合并 + 分批推送 + 只保留最新值
```

## 智能家居设备固件 OTA 升级完整实现

```python
import time
import uuid
import random
import logging
from enum import Enum
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Set, Tuple
from datetime import datetime, timedelta
from collections import defaultdict

logger = logging.getLogger(__name__)


class UpgradeStatus(Enum):
    PENDING = "pending"
    DOWNLOADING = "downloading"
    INSTALLING = "installing"
    VERIFYING = "verifying"
    SUCCESS = "success"
    FAILED = "failed"
    ROLLED_BACK = "rolled_back"


class FirmwareType(Enum):
    NORMAL = "normal"
    SECURITY_PATCH = "security_patch"
    HOTFIX = "hotfix"


@dataclass
class FirmwareVersion:
    firmware_id: str
    device_type: str
    version: str
    firmware_type: FirmwareType
    release_notes: str
    download_url: str
    file_size: int  # bytes
    checksum: str
    min_base_version: str  # minimum version required to upgrade from
    is_forced: bool = False
    created_at: datetime = field(default_factory=datetime.now)
    deprecated: bool = False


@dataclass
class UpgradeTask:
    task_id: str
    firmware_id: str
    device_type: str
    target_version: str
    canary_percentage: float  # 0.0 ~ 100.0
    is_forced: bool
    created_at: datetime = field(default_factory=datetime.now)
    status: str = "created"
    total_devices: int = 0
    upgraded_devices: int = 0
    failed_devices: int = 0
    rolled_back_devices: int = 0
    current_batch: int = 0


@dataclass
class DeviceUpgradeRecord:
    device_id: str
    task_id: str
    firmware_id: str
    target_version: str
    previous_version: str
    status: UpgradeStatus = UpgradeStatus.PENDING
    progress: float = 0.0  # 0.0 ~ 100.0
    started_at: Optional[datetime] = None
    completed_at: Optional[datetime] = None
    error_message: Optional[str] = None
    retry_count: int = 0
    max_retries: int = 3


class FirmwareOTAService:
    """智能家居设备固件 OTA 升级服务，管理固件版本注册、升级任务创建、
    设备升级进度跟踪、升级失败自动回滚，以及关键安全补丁的强制升级。"""

    MAX_CONCURRENT_DOWNLOADS = 500
    CANARY_BATCH_INTERVAL = timedelta(minutes=30)
    ROLLBACK_TIMEOUT = timedelta(minutes=10)
    MAX_ROLLBACK_ATTEMPTS = 2

    def __init__(self):
        # firmware_id -> FirmwareVersion
        self.firmware_registry: Dict[str, FirmwareVersion] = {}
        # device_type -> list of firmware_ids sorted by version
        self.type_firmware_index: Dict[str, List[str]] = defaultdict(list)
        # task_id -> UpgradeTask
        self.upgrade_tasks: Dict[str, UpgradeTask] = {}
        # (device_id, task_id) -> DeviceUpgradeRecord
        self.device_records: Dict[Tuple[str, str], DeviceUpgradeRecord] = {}
        # device_id -> current firmware version
        self.device_versions: Dict[str, str] = {}
        # device_id -> set of previous versions for rollback
        self.device_version_history: Dict[str, List[str]] = defaultdict(list)
        # active download counter
        self.active_downloads = 0
        # task_id -> scheduled batches of device_ids
        self.task_batches: Dict[str, List[List[str]]] = {}

    def register_firmware(
        self,
        device_type: str,
        version: str,
        firmware_type: FirmwareType,
        release_notes: str,
        download_url: str,
        file_size: int,
        checksum: str,
        min_base_version: str,
        is_forced: bool = False,
    ) -> FirmwareVersion:
        """注册新的固件版本到版本注册表。"""
        firmware_id = f"fw_{device_type}_{version}_{uuid.uuid4().hex[:8]}"

        # 检查同设备类型下版本号是否已存在
        for fid in self.type_firmware_index.get(device_type, []):
            existing = self.firmware_registry[fid]
            if existing.version == version and not existing.deprecated:
                raise ValueError(
                    f"Firmware version {version} already registered for {device_type}"
                )

        # 检查最小基础版本格式
        if not self._validate_version_string(min_base_version):
            raise ValueError(f"Invalid min_base_version format: {min_base_version}")

        firmware = FirmwareVersion(
            firmware_id=firmware_id,
            device_type=device_type,
            version=version,
            firmware_type=firmware_type,
            release_notes=release_notes,
            download_url=download_url,
            file_size=file_size,
            checksum=checksum,
            min_base_version=min_base_version,
            is_forced=is_forced,
        )

        self.firmware_registry[firmware_id] = firmware
        self.type_firmware_index[device_type].append(firmware_id)

        # 安全补丁自动标记强制
        if firmware_type == FirmwareType.SECURITY_PATCH:
            firmware.is_forced = True
            logger.info(
                f"Security patch {firmware_id} auto-marked as forced upgrade"
            )

        logger.info(
            f"Registered firmware {firmware_id}: {device_type} v{version} "
            f"(type={firmware_type.value}, forced={firmware.is_forced})"
        )
        return firmware

    def create_upgrade_task(
        self,
        firmware_id: str,
        target_device_ids: List[str],
        canary_percentage: float = 10.0,
        is_forced: bool = False,
    ) -> UpgradeTask:
        """创建升级任务，支持灰度发布（canary percentage）和强制升级。"""
        if firmware_id not in self.firmware_registry:
            raise ValueError(f"Firmware {firmware_id} not found in registry")

        firmware = self.firmware_registry[firmware_id]

        # 过滤出可升级的设备
        eligible_devices = []
        for device_id in target_device_ids:
            current_version = self.device_versions.get(device_id)
            if current_version is None:
                logger.warning(f"Device {device_id} version unknown, skipping")
                continue
            if current_version == firmware.version:
                logger.info(f"Device {device_id} already on target version, skipping")
                continue
            if not self._is_version_gte(current_version, firmware.min_base_version):
                logger.warning(
                    f"Device {device_id} on {current_version}, below minimum "
                    f"{firmware.min_base_version}, skipping"
                )
                continue
            eligible_devices.append(device_id)

        if not eligible_devices:
            raise ValueError("No eligible devices for this upgrade task")

        # 决定是否强制升级
        effective_forced = is_forced or firmware.is_forced

        task_id = f"task_{uuid.uuid4().hex[:12]}"
        task = UpgradeTask(
            task_id=task_id,
            firmware_id=firmware_id,
            device_type=firmware.device_type,
            target_version=firmware.version,
            canary_percentage=canary_percentage,
            is_forced=effective_forced,
            total_devices=len(eligible_devices),
        )

        # 分批次：canary 阶段只升级指定百分比的设备
        batches = self._create_canary_batches(
            eligible_devices, canary_percentage
        )
        self.task_batches[task_id] = batches

        # 为每个设备创建升级记录
        for device_id in eligible_devices:
            record = DeviceUpgradeRecord(
                device_id=device_id,
                task_id=task_id,
                firmware_id=firmware_id,
                target_version=firmware.version,
                previous_version=self.device_versions[device_id],
            )
            self.device_records[(device_id, task_id)] = record

        self.upgrade_tasks[task_id] = task
        logger.info(
            f"Created upgrade task {task_id}: {firmware.device_type} -> "
            f"v{firmware.version}, {len(eligible_devices)} devices, "
            f"canary={canary_percentage}%, forced={effective_forced}, "
            f"batches={len(batches)}"
        )
        return task

    def _create_canary_batches(
        self, device_ids: List[str], canary_percentage: float
    ) -> List[List[str]]:
        """创建灰度发布批次。第一轮只升级 canary_percentage 的设备，
        后续轮次逐步扩大直到全部设备。"""
        if not device_ids:
            return []

        shuffled = device_ids.copy()
        random.shuffle(shuffled)

        total = len(shuffled)
        canary_count = max(1, int(total * canary_percentage / 100.0))

        batches = []
        # 第一批：canary
        batches.append(shuffled[:canary_count])
        remaining = shuffled[canary_count:]

        # 后续批次：按递增比例分配（10% -> 30% -> 60% -> 100%）
        expansion_ratios = [0.10, 0.30, 0.60, 1.0]
        for ratio in expansion_ratios:
            if not remaining:
                break
            batch_count = max(1, int(total * ratio)) - sum(len(b) for b in batches)
            if batch_count <= 0:
                continue
            batch_count = min(batch_count, len(remaining))
            batches.append(remaining[:batch_count])
            remaining = remaining[batch_count:]

        if remaining:
            batches.append(remaining)

        return batches

    def get_upgrade_progress(self, task_id: str, device_id: Optional[str] = None) -> Dict:
        """获取升级任务的进度信息，可指定设备查看详细进度。"""
        if task_id not in self.upgrade_tasks:
            raise ValueError(f"Task {task_id} not found")

        task = self.upgrade_tasks[task_id]

        if device_id:
            record = self.device_records.get((device_id, task_id))
            if not record:
                raise ValueError(f"No record for device {device_id} in task {task_id}")
            return {
                "task_id": task_id,
                "device_id": device_id,
                "status": record.status.value,
                "progress": record.progress,
                "previous_version": record.previous_version,
                "target_version": record.target_version,
                "error_message": record.error_message,
                "retry_count": record.retry_count,
                "started_at": record.started_at.isoformat() if record.started_at else None,
                "completed_at": record.completed_at.isoformat() if record.completed_at else None,
            }

        # 汇总任务级进度
        status_distribution = defaultdict(int)
        total_progress = 0.0
        for (did, tid), record in self.device_records.items():
            if tid == task_id:
                status_distribution[record.status.value] += 1
                total_progress += record.progress

        device_count = max(1, task.total_devices)
        avg_progress = total_progress / device_count

        return {
            "task_id": task_id,
            "target_version": task.target_version,
            "is_forced": task.is_forced,
            "current_batch": task.current_batch,
            "total_batches": len(self.task_batches.get(task_id, [])),
            "total_devices": task.total_devices,
            "status_distribution": dict(status_distribution),
            "average_progress": round(avg_progress, 2),
            "upgraded_devices": task.upgraded_devices,
            "failed_devices": task.failed_devices,
            "rolled_back_devices": task.rolled_back_devices,
            "success_rate": (
                round(task.upgraded_devices / device_count * 100, 2)
                if task.upgraded_devices > 0 else 0.0
            ),
        }

    def rollback_device(self, device_id: str, task_id: str) -> DeviceUpgradeRecord:
        """对升级失败的设备执行回滚操作，恢复到升级前的固件版本。"""
        record = self.device_records.get((device_id, task_id))
        if not record:
            raise ValueError(f"No record for device {device_id} in task {task_id}")

        if record.status == UpgradeStatus.SUCCESS:
            raise ValueError(
                f"Device {device_id} upgrade succeeded, rollback not allowed "
                f"unless explicitly requested"
            )

        if record.status == UpgradeStatus.ROLLED_BACK:
            logger.warning(f"Device {device_id} already rolled back")
            return record

        task = self.upgrade_tasks[task_id]
        previous_version = record.previous_version

        # 检查是否可达之前的版本
        version_history = self.device_version_history.get(device_id, [])
        if previous_version not in version_history and previous_version not in self.device_versions.get(device_id, ""):
            logger.error(
                f"Cannot rollback device {device_id}: previous version "
                f"{previous_version} not found in history"
            )
            record.status = UpgradeStatus.FAILED
            record.error_message = f"Rollback failed: version {previous_version} not available"
            task.failed_devices += 1
            return record

        # 执行回滚
        try:
            rollback_success = self._execute_rollback(
                device_id, previous_version
            )
            if rollback_success:
                self.device_versions[device_id] = previous_version
                record.status = UpgradeStatus.ROLLED_BACK
                record.completed_at = datetime.now()
                task.rolled_back_devices += 1
                logger.info(
                    f"Device {device_id} rolled back to v{previous_version}"
                )
            else:
                record.status = UpgradeStatus.FAILED
                record.error_message = "Rollback execution failed on device"
                record.retry_count += 1
                task.failed_devices += 1
                logger.error(f"Rollback execution failed for device {device_id}")
        except Exception as e:
            record.status = UpgradeStatus.FAILED
            record.error_message = f"Rollback exception: {str(e)}"
            task.failed_devices += 1
            logger.exception(f"Rollback exception for device {device_id}")

        return record

    def _execute_rollback(self, device_id: str, target_version: str) -> bool:
        """模拟在设备上执行固件回滚。"""
        # 实际场景中，通过 MQTT/CoAP 向设备发送回滚指令
        # 这里模拟回滚成功率（99%）
        return random.random() < 0.99

    def process_batch(self, task_id: str) -> List[DeviceUpgradeRecord]:
        """处理升级任务的下一批次设备。"""
        if task_id not in self.upgrade_tasks:
            raise ValueError(f"Task {task_id} not found")

        task = self.upgrade_tasks[task_id]
        batches = self.task_batches.get(task_id, [])

        if task.current_batch >= len(batches):
            logger.info(f"Task {task_id} all batches completed")
            return []

        # 检查前一批次是否有高失败率，决定是否继续
        if task.current_batch > 0:
            failure_rate = task.failed_devices / max(1, task.upgraded_devices + task.failed_devices)
            if failure_rate > 0.15:  # 超过15%失败率暂停
                task.status = "paused"
                logger.warning(
                    f"Task {task_id} paused: failure rate {failure_rate:.1%} exceeds 15%"
                )
                return []

        current_batch = batches[task.current_batch]
        results = []

        for device_id in current_batch:
            record = self.device_records[(device_id, task_id)]
            result = self._upgrade_device(record)
            results.append(result)

            if result.status == UpgradeStatus.SUCCESS:
                task.upgraded_devices += 1
                self.device_versions[device_id] = record.target_version
                self.device_version_history[device_id].append(record.previous_version)
            elif result.status == UpgradeStatus.FAILED:
                if task.is_forced:
                    # 强制升级：重试
                    record.retry_count += 1
                    if record.retry_count <= record.max_retries:
                        logger.info(
                            f"Forced upgrade: retrying device {device_id} "
                            f"(attempt {record.retry_count})"
                        )
                    else:
                        # 重试耗尽，尝试回滚
                        self.rollback_device(device_id, task_id)
                else:
                    self.rollback_device(device_id, task_id)

        task.current_batch += 1
        if task.current_batch >= len(batches):
            task.status = "completed"
        else:
            task.status = "in_progress"

        return results

    def _upgrade_device(self, record: DeviceUpgradeRecord) -> DeviceUpgradeRecord:
        """模拟单个设备的固件升级流程。"""
        record.started_at = datetime.now()

        # 下载阶段
        record.status = UpgradeStatus.DOWNLOADING
        record.progress = 20.0
        if random.random() < 0.02:  # 2% 下载失败
            record.status = UpgradeStatus.FAILED
            record.error_message = "Download failed: network timeout"
            record.completed_at = datetime.now()
            return record

        # 安装阶段
        record.status = UpgradeStatus.INSTALLING
        record.progress = 60.0
        if random.random() < 0.03:  # 3% 安装失败
            record.status = UpgradeStatus.FAILED
            record.error_message = "Installation failed: checksum mismatch"
            record.completed_at = datetime.now()
            return record

        # 校验阶段
        record.status = UpgradeStatus.VERIFYING
        record.progress = 90.0
        if random.random() < 0.01:  # 1% 校验失败
            record.status = UpgradeStatus.FAILED
            record.error_message = "Verification failed: firmware integrity check failed"
            record.completed_at = datetime.now()
            return record

        # 成功
        record.status = UpgradeStatus.SUCCESS
        record.progress = 100.0
        record.completed_at = datetime.now()
        return record

    @staticmethod
    def _validate_version_string(version: str) -> bool:
        """验证版本号格式 (semver: x.y.z)。"""
        parts = version.split(".")
        if len(parts) != 3:
            return False
        return all(p.isdigit() for p in parts)

    @staticmethod
    def _is_version_gte(version: str, min_version: str) -> bool:
        """比较版本号，判断 version >= min_version。"""
        v_parts = [int(p) for p in version.split(".")]
        m_parts = [int(p) for p in min_version.split(".")]
        return v_parts >= m_parts
```

## 异常场景补充

### 场景：OTA 升级导致设备变砖
```
触发条件：固件包存在缺陷（如 flash 写入不完整、分区表错误、bootloader 不兼容），设备安装后无法正常启动或响应任何指令

检测机制：
  - 设备升级后心跳超时（超过 5 分钟无心跳上报）
  - 设备状态上报异常（status=offline 且 last_seen 在升级窗口内）
  - 升级任务中单设备 verify 阶段超时无响应
  - 批量升级中成功率骤降（某批次成功率 < 80%）

处理策略：
  - 立即暂停同批次所有待升级设备的升级任务
  - 通过安全启动分区（A/B 分区）自动回滚到上一稳定版本
  - 若 A/B 分区不可用，尝试通过 bootloader 恢复模式推送应急固件
  - 标记问题固件版本为 deprecated，禁止后续升级任务使用
  - 通知设备所有者升级异常，提供手动恢复指引（物理重置按钮说明）
  - 启动固件回溯分析，定位变砖根因（分区写入、内存溢出、硬件不兼容）

预防措施：
  - 固件发布前必须通过全量回归测试（含不同硬件批次）
  - OTA 升级强制启用 A/B 双分区机制，确保始终可回滚
  - 灰度发布首批不超过 1%，观察 24 小时无异常后再扩大
  - 固件包增加预校验步骤（checksum + 数字签名验证 + 分区表完整性检查）
  - 建立"熔断"机制：连续 3 台设备变砖则自动终止全量升级
```

### 场景：批量升级网络拥塞
```
触发条件：大规模设备同时下载固件包，导致 CDN 带宽打满或 MQTT Broker 连接数超限，设备下载超时或控制指令延迟

检测机制：
  - CDN 出口带宽利用率超过 90% 持续 5 分钟
  - MQTT Broker 并发连接数接近上限（如 50 万）
  - 设备下载阶段平均耗时超过预期的 3 倍
  - 升级任务中 downloading 状态设备数量堆积超过阈值
  - 网络监控告警：丢包率 > 5% 或 TCP 重传率异常

处理策略：
  - 立即对升级任务的下载队列实施限流（令牌桶算法，控制并发下载数）
  - 将待下载设备按地域分片，错峰安排下载时间窗口
  - 启用 CDN 多节点分发，将固件包推送到边缘节点缓存
  - 对 MQTT 指令实施 QoS 降级（QoS 2 -> QoS 1），减少通信开销
  - 开启固件包本地网关缓存：同局域网设备通过网关代理共享下载
  - 动态调整批次间隔，从 30 分钟延长到 2 小时

预防措施：
  - 升级任务默认分批执行，单批并发下载数不超过 500
  - 固件包压缩并支持增量更新（delta patch），减少传输体积
  - 预热 CDN 缓存：在正式升级前将固件包推送到各边缘节点
  - 支持断点续传：设备中断后可从已下载位置继续
  - 建立带宽预留机制：为控制指令预留 20% 带宽，避免升级挤占控制通道
  - 制定错峰升级策略：利用凌晨低谷时段执行大规模升级
```
