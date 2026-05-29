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
                # 冲突：按规则优先级选最高的
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
```

**优先级设计：**

```
场景模式（离家/回家/睡眠）→ priority = 100
安全规则（烟雾报警→开窗）→ priority = 90
舒适规则（温度控制）     → priority = 50
便利规则（自动关灯）     → priority = 30
```

**冲突示例与解决：**

```
规则A（priority=50）：温度>28°C → 开空调 26°C
规则B（priority=100）：离家模式 → 关所有设备

用户离家时温度>28°C → 两条规则同时触发
→ B 优先级更高 → 关空调（离家模式覆盖温控）
→ 冲突日志：规则B 覆盖 规则A，设备：空调

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

    def on_ack_timeout(self, device_id, request_id):
        """设备未确认 → 状态可能不准，标记为 stale"""
        self.state_cache.mark_stale(device_id)
        self.notify_user(device_id, "设备可能未响应，请检查")
        # 后台重试一次
        self.retry_command(device_id, request_id)
```

**乐观更新的好处与风险：**

| 方案 | 用户感知延迟 | 数据一致性 | 复杂度 |
|------|------------|-----------|-------|
| 等设备确认 | 1-3 秒 | 强一致 | 低 |
| 乐观更新 | < 100ms | 最终一致 | 中 |
| 乐观更新+回滚 | < 100ms | 最终一致+回滚 | 高 |

选择"乐观更新"：用户感知延迟从 1-3 秒降到 < 100ms，牺牲的是极少数设备离线时的状态不一致（通过 stale 标记和超时提示解决）。

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
```

### 定时触发规则

```python
class ScheduledRuleExecutor:
    def schedule_rule(self, rule):
        """注册定时规则（如"每天7:00开灯"）"""
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

        for action in rule.actions:
            self.device_controller.execute(action)
```

### MQTT 通信层

```python
class MQTTBridge:
    """MQTT 消息桥接：设备 ↔ 云端"""
    
    def on_device_state_report(self, device_id, state):
        """设备状态上报"""
        # 更新缓存
        self.state_cache.update(device_id, state)
        
        # 触发规则引擎评估
        event = DeviceEvent(device_id=device_id, properties=state)
        self.rule_engine.on_event(event)
        
        # 推送状态给用户 App
        self.push_state_to_app(device_id)
    
    def on_device_offline(self, device_id):
        """设备离线"""
        self.state_cache.mark_offline(device_id)
        self.notify_user(device_id, "设备已离线")
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