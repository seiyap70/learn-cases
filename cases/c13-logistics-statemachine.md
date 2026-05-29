# C13: 物流履约的状态机引擎

## 业务场景

某电商物流平台，包裹从发货到签收需要经历多个状态流转。这看似是一个简单的状态机问题，但实际物流场景的复杂性在于：异常路径远比正常路径多，且异常可能发生在任何节点。

```
已下单 → 已揽收 → 运输中 → 派送中 → 已签收
                ↘ 仓库中 → 运输中
                                 ↘ 异常 → 理赔中 → 已理赔
                                          ↘ 查找中 → 已找回 → 派送中
```

**已知数据：**
- 日均包裹量：500 万
- 状态变更 TPS 峰值：1 万（早晚高峰）
- 物流产品类型：8 种（标准快递/次日达/同城急送/跨境/冷链/大件/代收货款/保价）
- 平均流转节点数：6-12 个
- 异常率：约 2%（日均 10 万个异常包裹）

**为什么不能简单地在代码里写 if-else？**

8 种物流产品 × 平均 8 个状态 × 平均 3 个转换 = 约 200 个转换规则。如果硬编码在代码中：
- 每次新增物流产品或修改规则都需要改代码发版
- 规则散落在多个函数中，无法全局审视
- 测试困难——需要模拟每个状态组合

## 核心挑战

### 挑战 1：非法状态跳转的防止

包裹从"已下单"直接跳到"已签收"是非法的。但在实际运营中，快递员可能误操作，系统必须拒绝非法跳转并给出明确提示。

### 挑战 2：8 种物流产品有不同的流转路径

跨境包裹多一个"清关"节点，冷链包裹多一个"温控异常"分支。不同产品的状态机不同，但引擎必须统一。

### 挑战 3：超时自动流转的可靠性

"运输中"超过 72 小时未更新 → 自动标记"异常"。但 500 万包裹中有多少在"运输中"？定时轮询扫描全表的性能不可接受。

### 挑战 4：状态变更的原子性

状态变更 + 触发动作（发通知、触发结算）必须要么全成功要么全失败。但动作可能失败（如短信网关超时），此时状态是否应该变更？

## 设计约束

- 状态变更必须持久化且可审计（每个包裹的完整状态历史可查）
- 状态机定义需要可热更新（不重启服务）
- 超时检测精度：±30 秒
- 状态变更 TPS 1 万，P99 延迟 < 100ms

## 请先独立思考（限时 35 分钟）

1. 状态机定义用代码硬编码 vs 配置化（JSON/YAML）vs DSL，各自的优劣？考虑到运营人员可能需要修改规则。
2. 状态变更与动作触发应该在同一个事务中吗？如果动作（发短信）失败，状态是否应该回滚？
3. 超时检测用定时轮询 vs 延迟队列 vs 时间轮，哪种方案最适合本场景？考虑 500 万包裹的规模。
4. 如何处理"状态回退"——快递员误操作"已签收"后需要纠正？

---

## 设计解析

### 核心原则：状态流转规则数据化

状态机的定义（有哪些状态、哪些转换、触发什么动作）存储在数据库中，而非硬编码在代码中。引擎是通用的，定义是可配置的。

### 状态机定义模型

```json
{
  "productType": "CROSS_BORDER",
  "version": 3,
  "states": [
    { "name": "CREATED",       "isInitial": true },
    { "name": "PICKED_UP",     "isInitial": false },
    { "name": "CUSTOMS",       "isInitial": false },
    { "name": "IN_TRANSIT",    "isInitial": false },
    { "name": "DELIVERING",    "isInitial": false },
    { "name": "DELIVERED",     "isFinal": true },
    { "name": "EXCEPTION",     "isInitial": false },
    { "name": "CLAIMING",      "isInitial": false },
    { "name": "CLAIMED",       "isFinal": true }
  ],
  "transitions": [
    {
      "from": "CREATED", "to": "PICKED_UP", "event": "PICKUP",
      "condition": null,
      "actions": ["notify_sender", "update_tracking"]
    },
    {
      "from": "PICKED_UP", "to": "CUSTOMS", "event": "ENTER_CUSTOMS",
      "condition": null,
      "actions": ["notify_sender_customs"]
    },
    {
      "from": "CUSTOMS", "to": "IN_TRANSIT", "event": "CUSTOMS_CLEARED",
      "condition": null,
      "actions": ["notify_sender_cleared"]
    },
    {
      "from": "IN_TRANSIT", "to": "DELIVERING", "event": "OUT_FOR_DELIVERY",
      "condition": null,
      "actions": ["notify_receiver"]
    },
    {
      "from": "DELIVERING", "to": "DELIVERED", "event": "SIGN",
      "condition": null,
      "actions": ["trigger_settlement", "notify_sender", "notify_receiver"]
    },
    {
      "from": "IN_TRANSIT", "to": "EXCEPTION", "event": "TIMEOUT",
      "condition": "duration_in_state > 72h",
      "actions": ["alert_logistics_team", "notify_sender"]
    },
    {
      "from": "CUSTOMS", "to": "EXCEPTION", "event": "CUSTOMS_HOLD",
      "condition": null,
      "actions": ["alert_customs_issue", "notify_sender"]
    },
    {
      "from": "EXCEPTION", "to": "CLAIMING", "event": "FILE_CLAIM",
      "condition": null,
      "actions": ["create_claim_record"]
    },
    {
      "from": "CLAIMING", "to": "CLAIMED", "event": "CLAIM_SETTLED",
      "condition": null,
      "actions": ["trigger_refund"]
    }
  ],
  "timeouts": [
    { "state": "IN_TRANSIT", "duration": "72h", "event": "TIMEOUT" },
    { "state": "CUSTOMS",    "duration": "168h", "event": "TIMEOUT" },
    { "state": "DELIVERING", "duration": "48h", "event": "TIMEOUT" }
  ]
}
```

### 数据库设计

```sql
-- 状态机定义表（支持版本管理）
CREATE TABLE workflow_definitions (
    id SERIAL PRIMARY KEY,
    product_type VARCHAR(32) NOT NULL,
    definition JSONB NOT NULL,
    version INTEGER DEFAULT 1,
    is_active BOOLEAN DEFAULT TRUE,
    updated_at TIMESTAMP DEFAULT NOW(),
    updated_by VARCHAR(64),
    
    UNIQUE KEY uk_type_version (product_type, version)
);

-- 包裹状态表
CREATE TABLE package_states (
    package_id VARCHAR(64) PRIMARY KEY,
    product_type VARCHAR(32) NOT NULL,
    current_state VARCHAR(32) NOT NULL,
    state_entered_at TIMESTAMP NOT NULL,
    version INTEGER DEFAULT 0,
    
    INDEX idx_state_timeout (current_state, state_entered_at),
    INDEX idx_product (product_type)
);

-- 状态变更日志（审计，追加写入）
CREATE TABLE state_transition_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    package_id VARCHAR(64) NOT NULL,
    from_state VARCHAR(32),
    to_state VARCHAR(32) NOT NULL,
    event VARCHAR(32) NOT NULL,
    operator VARCHAR(64),
    operator_type VARCHAR(16),   -- SYSTEM / COURIER / ADMIN
    metadata JSONB,
    created_at TIMESTAMP(3) DEFAULT CURRENT_TIMESTAMP(3),
    
    INDEX idx_package_id (package_id),
    INDEX idx_created_at (created_at)
);
```

### 状态流转引擎：完整实现

```python
class StateMachineEngine:
    def __init__(self, db, definition_cache, action_executor, timeout_scheduler):
        self.db = db
        self.definition_cache = definition_cache
        self.action_executor = action_executor
        self.timeout_scheduler = timeout_scheduler

    def transition(self, package_id, event, operator=None, operator_type="SYSTEM", metadata=None):
        """执行状态流转"""
        
        # 1. 加载包裹当前状态（加行锁防止并发）
        pkg = self.db.query_one("""
            SELECT * FROM package_states 
            WHERE package_id = %s FOR UPDATE
        """, package_id)
        if not pkg:
            raise PackageNotFound(package_id)

        # 2. 加载状态机定义（带缓存，版本变化时失效）
        definition = self.definition_cache.get(pkg.product_type)

        # 3. 查找匹配的转换规则
        transition = self.find_transition(definition, pkg.current_state, event)
        if not transition:
            raise InvalidTransition(
                f"不允许从 {pkg.current_state} 通过事件 {event} 转换"
            )

        # 4. 校验前置条件
        if transition.condition:
            if not self.evaluate_condition(transition.condition, pkg):
                raise ConditionNotMet(transition.condition)

        # 5. 执行状态变更（事务内）
        with self.db.transaction() as tx:
            # 更新状态
            affected = tx.execute("""
                UPDATE package_states 
                SET current_state = %s, state_entered_at = NOW(), version = version + 1
                WHERE package_id = %s AND version = %s
            """, transition.to, package_id, pkg.version)
            if affected == 0:
                raise ConcurrentConflict(package_id)

            # 记录审计日志
            tx.execute("""
                INSERT INTO state_transition_log 
                (package_id, from_state, to_state, event, operator, operator_type, metadata)
                VALUES (%s, %s, %s, %s, %s, %s, %s)
            """, package_id, pkg.current_state, transition.to, event, 
                 operator, operator_type, json.dumps(metadata or {}))

            # 取消旧的超时定时器
            self.timeout_scheduler.cancel(package_id)

            # 注册新的超时定时器
            timeout = self.find_timeout(definition, transition.to)
            if timeout:
                self.timeout_scheduler.schedule(
                    package_id, timeout.duration, timeout.event
                )

        # 6. 触发动作（事务外，失败不影响状态变更）
        for action_name in transition.actions:
            try:
                self.action_executor.execute_async(action_name, package_id, metadata)
            except Exception as e:
                # 动作失败不影响状态变更，但需记录
                self.log_action_failure(package_id, action_name, e)

        return {
            "package_id": package_id,
            "from_state": pkg.current_state,
            "to_state": transition.to,
            "event": event
        }

    def find_transition(self, definition, current_state, event):
        """查找匹配的转换规则"""
        for t in definition.transitions:
            if t.from == current_state and t.event == event:
                return t
        return None

    def evaluate_condition(self, condition, pkg):
        """评估前置条件"""
        if condition == "duration_in_state > 72h":
            duration = now() - pkg.state_entered_at
            return duration > timedelta(hours=72)
        # 更多条件...
        return True
```

**关键设计决策：状态变更与动作触发分离**

状态变更和动作触发不在同一个事务中。理由：
- 状态变更是核心操作，必须可靠
- 动作（发通知、触发结算）可能失败（如短信网关超时）
- 如果动作失败导致状态变更回滚 → 包裹状态与实际不一致（快递员已签收但系统显示"派送中"）
- 正确做法：状态变更成功后，异步触发动作；动作失败则重试，最终人工介入

### 超时检测：延迟消息队列

**为什么不用定时轮询？**

定时轮询扫描 `WHERE current_state = 'IN_TRANSIT' AND state_entered_at < NOW() - 72h`：
- 500 万包裹中约 50 万在"运输中"
- 每分钟扫描 50 万行 → 数据库压力大
- 精度取决于扫描频率（每分钟扫一次 = 最多 1 分钟延迟）

**延迟消息方案：**

```python
class TimeoutScheduler:
    def schedule(self, package_id, duration_str, event):
        """进入状态时投递延迟消息"""
        duration = parse_duration(duration_str)
        execute_at = now() + duration
        
        # 写入延迟队列表（比 Redis ZADD 更可靠）
        self.db.execute("""
            INSERT INTO scheduled_timeouts 
            (package_id, expected_state, event, execute_at, status)
            VALUES (%s, %s, %s, %s, 'PENDING')
        """, package_id, current_state, event, execute_at)

    def cancel(self, package_id):
        """状态变更时取消定时器"""
        self.db.execute("""
            UPDATE scheduled_timeouts 
            SET status = 'CANCELLED' 
            WHERE package_id = %s AND status = 'PENDING'
        """, package_id)

class TimeoutConsumer:
    """定时扫描 scheduled_timeouts 表"""
    
    def run(self):
        while True:
            # 每秒扫描一次到期的定时器
            items = self.db.query("""
                SELECT * FROM scheduled_timeouts
                WHERE status = 'PENDING' AND execute_at <= NOW()
                LIMIT 1000
            """)
            
            for item in items:
                # 检查包裹是否仍在预期状态
                pkg = self.db.query_one(
                    "SELECT current_state FROM package_states WHERE package_id = %s",
                    item.package_id
                )
                
                if pkg and pkg.current_state == item.expected_state:
                    # 仍在预期状态 → 触发超时事件
                    try:
                        self.engine.transition(
                            item.package_id, item.event, 
                            operator="SYSTEM", operator_type="SYSTEM"
                        )
                    except InvalidTransition:
                        pass  # 状态已变更，忽略
                
                # 标记为已处理
                self.db.execute(
                    "UPDATE scheduled_timeouts SET status = 'PROCESSED' WHERE id = %s",
                    item.id
                )
            
            time.sleep(1)
```

**为什么用数据库表而非 Redis ZADD？**

- 数据库更可靠——Redis 延迟队列在宕机时可能丢失
- 可审计——每个定时器的创建和执行都有记录
- 性能：每秒扫描 1000 条，每条一个索引查询 → 可接受

### 状态回退：前向补偿而非回退

**原则：永远不回退状态，而是向前补偿。**

```
错误场景：快递员误操作"已签收"

不回退：DELIVERED → DELIVERING（回退，可能已触发结算，回退结算极复杂）

前向补偿：DELIVERED → MISDELIVERED → REDIRECTING → DELIVERING → DELIVERED
          ↑ 新的状态流转，每个步骤都有审计记录
```

```python
class MisdeliveryHandler:
    def handle_misdelivery(self, package_id, reason):
        """处理误签收"""
        # 不回退状态，而是发起"重新派送"流程
        self.engine.transition(
            package_id, "MISDELIVER",
            operator=current_user.id,
            metadata={"reason": reason}
        )
        # MISDELIVER → REDIRECTING → DELIVERING → DELIVERED
```

### 定义热更新

```python
class DefinitionCache:
    def __init__(self):
        self.cache = {}
        self.version_map = {}

    def get(self, product_type):
        # 检查版本是否变化
        current_version = self.db.query_one(
            "SELECT MAX(version) FROM workflow_definitions WHERE product_type = %s AND is_active = TRUE",
            product_type
        ).version

        if self.version_map.get(product_type) != current_version:
            # 版本变化 → 重新加载
            definition = self.db.query_one(
                "SELECT definition FROM workflow_definitions "
                "WHERE product_type = %s AND version = %s",
                product_type, current_version
            )
            self.cache[product_type] = parse_definition(definition)
            self.version_map[product_type] = current_version

        return self.cache[product_type]
```

**版本管理的安全保证：**
- 新版本定义保存后，旧版本标记 `is_active = FALSE`
- 在途包裹继续使用旧版本（因为定义已缓存）
- 新包裹使用新版本
- 如果新版本有问题，可以回滚到旧版本（`is_active = TRUE`）

## 常见陷阱（深度分析）

### 陷阱 1：状态机硬编码

**具体问题：** 运营要求新增"冷链"物流产品，需要增加"温控异常"状态和转换。如果硬编码：
- 需要修改 5+ 个文件（状态枚举、转换逻辑、动作触发、超时规则）
- 需要完整的回归测试（确保不影响现有 7 种产品）
- 发版周期 1-2 周

配置化方案：在管理后台配置新的 JSON 定义 → 即时生效 → 无需发版。

### 陷阱 2：状态变更与动作同事务

**具体问题：** `DELIVERED → trigger_settlement` 在同一事务中。如果结算服务超时（3秒），事务回滚 → 状态仍为"派送中" → 快递员看到"未签收" → 重复操作签收。

**正确做法：** 状态变更独立事务，动作异步触发。

### 陷阱 3：定时轮询扫描超时

**具体场景：** 50 万包裹在"运输中"，每分钟扫描：
```sql
SELECT * FROM package_states 
WHERE current_state = 'IN_TRANSIT' AND state_entered_at < NOW() - INTERVAL 72 HOUR
```
- 每分钟扫描 50 万行 → 数据库 CPU 飙升
- 扫描期间加锁 → 影响正常的查询和更新

延迟队列表方案：每秒只查 `scheduled_timeouts` 中到期的少量记录（通常 < 100 条）。

### 陷阱 4：允许状态回退

**回退的复杂性：**
- "已签收"→ 触发了结算（钱已付给商家）
- 回退到"派送中"→ 结算需要反向操作（退款）
- 如果退款失败 → 状态是"派送中"但钱已付 → 财务不一致

**前向补偿的优势：** 每个步骤都是明确的业务操作（误签→重新派送→签收），财务流程完整可追溯。

## 延伸思考

- **批量流转**：整车 200 个包裹到达站点，需要批量触发"到达站点"事件。方案：批量 INSERT 到 `state_transition_log`，批量 UPDATE `package_states`，减少数据库交互。
- **BPMN 可视化**：将状态机定义映射为 BPMN 图，运营在可视化编辑器中拖拽修改。引擎从 BPMN XML 解析为 JSON 定义。
- **跨公司联运**：顺丰→中通→末端配送，每个公司有自己的状态机。需要一个"联邦状态机"映射各公司的状态到统一的状态模型。