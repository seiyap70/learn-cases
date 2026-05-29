# C20: 智能家居的设备联动与规则引擎

## 业务场景

某智能家居平台，接入 1000+ 种设备（灯光、空调、窗帘、门锁、传感器等），500 万用户设置了自动化规则："当我到家时，自动开灯+开空调+关窗帘"。

**已知数据：**
- 接入设备数：1000 万台
- 用户数：500 万
- 日均规则触发：2000 万次
- 设备控制延迟：< 2 秒
- 规则类型：条件触发、定时触发、手动触发

**为什么是难题？**

核心矛盾是**规则组合爆炸与冲突**：
- 10 个设备之间可能有 90 种联动组合
- 两条规则可能矛盾："温度>28°C 开空调" vs "离家模式关所有设备"
- 设备离线时规则仍触发 → 上线后如何补执行？

## 核心挑战

### 挑战 1：规则冲突检测与解决

两条规则对同一设备产生矛盾操作（一条说开灯，另一条说关灯），如何检测并自动解决？

### 挑战 2：设备状态同步延迟

用户点击"关灯"→ 云端处理 → 发送 MQTT 指令 → 设备执行 → 状态上报 → 云端更新。全链路 1-3 秒。期间用户看到"灯开着"但实际已关。

### 挑战 3：离线设备的处理

设备离线时规则仍触发 → 设备上线后补执行还是丢弃？

## 设计解析

### 规则引擎：条件-动作模型 + 优先级冲突解决

```python
class Rule:
    def __init__(self, rule_id, conditions, actions, priority=0):
        self.rule_id = rule_id
        self.conditions = conditions  # 条件列表
        self.actions = actions        # 动作列表
        self.priority = priority      # 优先级（数字越大越优先）

class Condition:
    def __init__(self, device_id, property, operator, value):
        self.device_id = device_id
        self.property = property      # temperature / motion / door_state
        self.operator = operator      # GT / LT / EQ / CHANGED
        self.value = value

    def evaluate(self, event):
        if event.device_id != self.device_id:
            return False
        actual = event.properties.get(self.property)
        if self.operator == "GT": return actual > self.value
        if self.operator == "LT": return actual < self.value
        if self.operator == "EQ": return actual == self.value
        if self.operator == "CHANGED": return True
        return False

class Action:
    def __init__(self, device_id, command, params=None):
        self.device_id = device_id
        self.command = command          # turn_on / turn_off / set_temperature
        self.params = params or {}

class RuleEngine:
    def evaluate(self, event):
        """评估事件触发的所有规则"""
        triggered = []
        for rule in self.rules.values():
            if all(c.evaluate(event) for c in rule.conditions):
                triggered.append(rule)

        if not triggered:
            return []

        # 冲突检测与解决
        resolved = self.conflict_resolver.resolve(triggered)

        # 执行动作
        results = []
        for rule in resolved:
            for action in rule.actions:
                result = self.device_controller.execute(action)
                results.append(result)

        return results
```

### 冲突解决：同一设备取最高优先级

```python
class ConflictResolver:
    def resolve(self, triggered_rules):
        """检测并解决规则冲突"""
        # 按设备分组：同一设备的所有动作
        actions_by_device = defaultdict(list)
        for rule in triggered_rules:
            for action in rule.actions:
                actions_by_device[action.device_id].append((rule, action))

        resolved = []
        for device_id, rule_actions in actions_by_device.items():
            if len(rule_actions) == 1:
                # 无冲突
                resolved.append(rule_actions[0])
            else:
                # 冲突：按规则优先级选最高的
                # "离家模式"(priority=20) > "温度控制"(priority=10)
                rule_actions.sort(key=lambda x: x[0].priority, reverse=True)
                winner = rule_actions[0]
                resolved.append(winner)

                # 记录被覆盖的规则（审计用）
                for rule, action in rule_actions[1:]:
                    self.log_conflict(
                        winner_rule=winner[0].rule_id,
                        loser_rule=rule.rule_id,
                        device_id=device_id
                    )

        return [r for r, _ in resolved]
```

**冲突示例：**

```
规则A（priority=10）：温度>28°C → 开空调 26°C
规则B（priority=20）：离家模式 → 关所有设备

用户离家时温度>28°C → 两条规则同时触发 → B 优先级更高 → 关空调

用户在家时温度>28°C → 只有 A 触发 → 开空调
```

### 设备状态同步：乐观更新 + 确认回执

```python
class DeviceController:
    def execute(self, action):
        """发送设备指令并乐观更新状态"""
        # 1. 乐观更新：指令发出后立即更新缓存（不等设备确认）
        self.state_cache.update(action.device_id, {
            action.command: action.params
        })

        # 2. 发送 MQTT 指令
        self.mqtt.publish(f"cmd/{action.device_id}", json.dumps({
            "command": action.command,
            "params": action.params,
            "request_id": uuid4(),
            "timestamp": now_ms()
        }))

        # 3. 设置确认超时（5秒）
        self.schedule_ack_check(action.device_id, timeout=5)

        return {"status": "sent", "device_id": action.device_id}

    def on_device_ack(self, device_id, request_id, actual_state):
        """设备确认执行成功"""
        self.state_cache.update(device_id, actual_state)
        self.cancel_ack_check(device_id)

    def on_ack_timeout(self, device_id):
        """设备未确认 → 状态可能不准，标记为 stale"""
        self.state_cache.mark_stale(device_id)
        self.notify_user(device_id, "设备可能未响应，请检查")
```

### 离线设备：指令队列

```python
class OfflineCommandQueue:
    def execute(self, action):
        if self.is_online(action.device_id):
            return self.device_controller.execute(action)
        else:
            # 设备离线 → 放入队列，上线后补发
            # 只保留最近 N 条指令（避免队列无限增长）
            self.redis.lpush(
                f"cmd_queue:{action.device_id}",
                json.dumps(action.__dict__)
            )
            self.redis.ltrim(f"cmd_queue:{action.device_id}", 0, 49)  # 最多 50 条

            # 判断是否需要补执行
            # 定时类规则（"每天7点开灯"）→ 上线后补执行
            # 条件类规则（"温度>28开空调"）→ 上线后重新评估，可能不需要执行了
            return {"status": "queued"}

    def on_device_online(self, device_id):
        """设备上线 → 发送积压指令"""
        commands = self.redis.lrange(f"cmd_queue:{device_id}", 0, -1)
        for cmd_json in commands:
            cmd = json.loads(cmd_json)
            # 重新评估条件（可能已经不需要执行了）
            if self.should_execute(cmd):
                self.device_controller.execute(Action(**cmd))
        self.redis.delete(f"cmd_queue:{device_id}")
```

### 定时触发规则

```python
class ScheduledRuleExecutor:
    def schedule_rule(self, rule):
        """注册定时规则（如"每天7:00开灯"）"""
        # 使用 cron 表达式
        job_id = self.scheduler.add_job(
            self.execute_rule,
            trigger='cron',
            hour=rule.schedule.hour,
            minute=rule.schedule.minute,
            args=[rule]
        )
        return job_id

    def execute_rule(self, rule):
        """定时触发规则"""
        # 检查前置条件（如"如果人在家"）
        if rule.preconditions:
            if not all(c.evaluate_current() for c in rule.preconditions):
                return  # 条件不满足，跳过

        # 执行动作
        for action in rule.actions:
            self.device_controller.execute(action)
```

## 常见陷阱（深度分析）

### 陷阱 1：规则不检测冲突

**后果：** 同一设备收到矛盾指令（开灯+关灯）→ 行为不可预测（取决于哪个先到达）。

**解决方案：** 冲突检测 + 优先级解决。

### 陷阱 2：设备状态不同步

**后果：** 用户看到 App 上"灯开着"但实际已关 → 反复操作 → 体验差。

**解决方案：** 乐观更新 + 设备 ACK 确认 + 超时标记 stale。

### 陷阱 3：离线指令无限积压

**后果：** 设备离线 30 天 → 积压 30 天的定时指令 → 上线后疯狂执行 → 异常行为。

**解决方案：** 队列最大 50 条 + 重新评估条件。

### 陷阱 4：规则硬编码

**后果：** 新设备类型需要改代码发版 → 无法快速支持新产品。

**解决方案：** 规则数据化（JSON 配置），引擎通用化。

## 延伸思考

- **Matter 协议**：跨厂商标准，统一设备模型 → 减少适配层代码。
- **语音控制**：NLU 解析自然语言为规则条件/动作。
- **场景推荐**：AI 分析用户习惯，自动推荐自动化规则。