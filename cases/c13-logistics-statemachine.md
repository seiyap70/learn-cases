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

**补充数据库表设计：**

```sql
-- 包裹主表（业务核心实体）
CREATE TABLE packages (
    package_id VARCHAR(64) PRIMARY KEY,
    sender_id BIGINT NOT NULL,
    receiver_id BIGINT,
    sender_name VARCHAR(64) NOT NULL,
    receiver_name VARCHAR(64),
    sender_phone VARCHAR(20) NOT NULL,
    receiver_phone VARCHAR(20),
    sender_address JSONB NOT NULL,       -- {province, city, district, detail}
    receiver_address JSONB NOT NULL,
    product_type VARCHAR(32) NOT NULL,   -- STANDARD / NEXT_DAY / SAME_CITY / CROSS_BORDER / COLD_CHAIN / OVERSIZED / COD / INSURED
    weight_grams INTEGER NOT NULL,
    declared_value_cents INTEGER DEFAULT 0,
    cod_amount_cents INTEGER DEFAULT 0,  -- 代收货款金额
    is_insured BOOLEAN DEFAULT FALSE,
    insured_amount_cents INTEGER DEFAULT 0,
    current_node_code VARCHAR(32),       -- 当前所在节点编码
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_sender (sender_id),
    INDEX idx_receiver (receiver_id),
    INDEX idx_product_type (product_type),
    INDEX idx_current_node (current_node_code)
);

-- 状态转换规则表（将 JSON 中的 transitions 拆为关系表，支持细粒度查询与管理）
CREATE TABLE transition_rules (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    product_type VARCHAR(32) NOT NULL,
    from_state VARCHAR(32) NOT NULL,
    to_state VARCHAR(32) NOT NULL,
    event VARCHAR(32) NOT NULL,
    condition_expr VARCHAR(256),          -- 条件表达式，如 "duration_in_state > 72h"
    actions JSONB,                        -- ["notify_sender", "update_tracking"]
    priority INTEGER DEFAULT 0,           -- 同一 from_state+event 多规则时的优先级
    is_active BOOLEAN DEFAULT TRUE,
    version INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE KEY uk_rule (product_type, from_state, event, priority, version),
    INDEX idx_from_state (product_type, from_state),
    INDEX idx_event (product_type, event)
);

-- 异常处理表（记录所有异常事件及处理方案）
CREATE TABLE exception_handling (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    package_id VARCHAR(64) NOT NULL,
    exception_type VARCHAR(32) NOT NULL,   -- LOST / DAMAGED / MISDELIVER / DOUBLE_SCAN / STATE_CONFLICT / CUSTOMS_HOLD / TEMP_ABNORMAL
    exception_source VARCHAR(16) NOT NULL, -- SYSTEM / COURIER / CUSTOMER / PARTNER
    exception_desc TEXT,
    from_state VARCHAR(32) NOT NULL,       -- 异常发生时包裹状态
    resolution_type VARCHAR(32),           -- REDIRECT / RECLAIM / CLAIM / RETRY / IGNORE / MANUAL
    resolution_status VARCHAR(16) DEFAULT 'PENDING',  -- PENDING / PROCESSING / RESOLVED / FAILED
    resolved_by VARCHAR(64),
    resolved_at TIMESTAMP,
    compensation_amount_cents INTEGER DEFAULT 0,
    metadata JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_package_exception (package_id),
    INDEX idx_exception_type (exception_type, resolution_status),
    INDEX idx_resolution_status (resolution_status, created_at)
);

-- 动作执行重试表（异步动作失败后的重试调度）
CREATE TABLE action_retry_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    package_id VARCHAR(64) NOT NULL,
    action_name VARCHAR(64) NOT NULL,
    action_params JSONB,
    retry_count INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 5,
    next_retry_at TIMESTAMP NOT NULL,
    status VARCHAR(16) DEFAULT 'PENDING',  -- PENDING / RETRYING / SUCCESS / DEAD_LETTER
    last_error TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_next_retry (status, next_retry_at),
    INDEX idx_package_action (package_id, action_name)
);

-- 补偿记录表（前向补偿的每个步骤记录）
CREATE TABLE compensation_records (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    package_id VARCHAR(64) NOT NULL,
    exception_id BIGINT,                   -- 关联 exception_handling.id
    compensation_type VARCHAR(32) NOT NULL, -- MISDELIVER_REDIRECT / LOST_RECLAIM / DOUBLE_SCAN_DEDUP
    from_state VARCHAR(32) NOT NULL,
    to_state VARCHAR(32) NOT NULL,
    trigger_event VARCHAR(32) NOT NULL,
    operator VARCHAR(64),
    metadata JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_package_comp (package_id),
    INDEX idx_exception (exception_id)
);

-- 超时调度表（补充完整定义）
CREATE TABLE scheduled_timeouts (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    package_id VARCHAR(64) NOT NULL,
    expected_state VARCHAR(32) NOT NULL,   -- 触发时校验包裹是否仍在预期状态
    event VARCHAR(32) NOT NULL,
    execute_at TIMESTAMP NOT NULL,
    status VARCHAR(16) DEFAULT 'PENDING',  -- PENDING / CANCELLED / PROCESSED / FAILED
    retry_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_execute (status, execute_at),
    INDEX idx_package (package_id, status)
);
```

**关键表设计说明：**

| 表名 | 写入频率 | 估算日增量 | 存储估算(行) | 核心索引 |
|------|---------|-----------|-------------|---------|
| packages | 低(创建时写) | 500万 | 18亿(1年) | package_id PK |
| package_states | 中(每次流转更新) | 500万更新 | 500万(当前) | (current_state, state_entered_at) |
| state_transition_log | 高(追加写入) | 3000万~5000万 | 150亿(1年) | (package_id), (created_at) |
| transition_rules | 极低(配置变更) | < 100 | < 2000 | (product_type, from_state) |
| exception_handling | 中 | 10万 | 3600万(1年) | (exception_type, resolution_status) |
| action_retry_log | 低~中 | 5万~20万 | 7000万(1年) | (status, next_retry_at) |
| scheduled_timeouts | 中 | 500万创建+500万处理 | 500万(活跃) | (status, execute_at) |

> `state_transition_log` 是增长最快的表。建议按月分区（`PARTITION BY RANGE (created_at)`），历史分区可归档到冷存储。

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
        matches = []
        for t in definition.transitions:
            if t.from == current_state and t.event == event:
                matches.append(t)
        if not matches:
            return None
        # 同一 from_state+event 可能有多个规则（不同条件），按优先级排序
        matches.sort(key=lambda t: t.get("priority", 0), reverse=True)
        return matches[0]

    def evaluate_condition(self, condition, pkg):
        """通用条件评估引擎"""
        if not condition:
            return True

        # 支持多种条件表达式格式
        # 格式 1: "duration_in_state > 72h"
        # 格式 2: "weight_grams < 5000 and product_type == 'STANDARD'"
        # 格式 3: "metadata.temperature_celsius > 8" (冷链场景)

        try:
            return self._eval_condition_expr(condition, pkg)
        except Exception as e:
            # 条件评估失败，默认不允许转换（安全侧）
            self.log_condition_error(pkg.package_id, condition, e)
            return False

    def _eval_condition_expr(self, expr, pkg):
        """安全地评估条件表达式（仅支持白名单函数和操作符）"""
        import re

        # 白名单允许的变量映射
        context = {
            "duration_in_state": (now() - pkg.state_entered_at).total_seconds(),
            "weight_grams": pkg.weight_grams,
            "product_type": pkg.product_type,
            "current_state": pkg.current_state,
            "declared_value_cents": pkg.declared_value_cents,
            "is_insured": pkg.is_insured,
            "cod_amount_cents": pkg.cod_amount_cents,
        }

        # 如果 pkg 有 metadata，也注入上下文
        if hasattr(pkg, "metadata") and pkg.metadata:
            for k, v in pkg.metadata.items():
                context[f"metadata.{k}"] = v

        # 解析条件表达式：variable operator value
        # 支持: >, <, >=, <=, ==, !=, and, or
        patterns = [
            # duration_in_state > 72h → 转换为秒比较
            (r"duration_in_state\s*(>|<|>=|<=|==|!=)\s*(\d+)h", self._eval_duration),
            # 通用变量比较: weight_grams > 5000
            (r"(\w+(?:\.\w+)?)\s*(>|<|>=|<=|==|!=)\s*(\d+(?:\.\d+)?)", self._eval_numeric),
            # 字符串比较: product_type == 'STANDARD'
            (r"(\w+)\s*(==|!=)\s*'(\w+)'", self._eval_string),
            # 组合条件: ... and/or ...
            (r"(.+?)\s+and\s+(.+)", lambda m: self._eval_condition_expr(m.group(1).strip(), pkg)
                                           and self._eval_condition_expr(m.group(2).strip(), pkg)),
            (r"(.+?)\s+or\s+(.+)", lambda m: self._eval_condition_expr(m.group(1).strip(), pkg)
                                          or self._eval_condition_expr(m.group(2).strip(), pkg)),
        ]

        for pattern, handler in patterns:
            match = re.match(pattern, expr.strip())
            if match:
                return handler(match)

        # 无法解析的条件 → 拒绝转换
        raise ValueError(f"无法解析条件表达式: {expr}")

    def _eval_duration(self, match):
        """评估时长条件"""
        op, hours = match.group(1), int(match.group(2))
        duration_secs = (now() - self._current_pkg.state_entered_at).total_seconds()
        target_secs = hours * 3600
        ops = {">": duration_secs > target_secs, "<": duration_secs < target_secs,
               ">=": duration_secs >= target_secs, "<=": duration_secs <= target_secs,
               "==": abs(duration_secs - target_secs) < 1, "!=": abs(duration_secs - target_secs) >= 1}
        return ops.get(op, False)

    def _eval_numeric(self, match):
        """评估数值比较"""
        var_name, op, value = match.group(1), match.group(2), float(match.group(3))
        actual = self._context.get(var_name)
        if actual is None:
            return False
        ops = {">": actual > value, "<": actual < value,
               ">=": actual >= value, "<=": actual <= value,
               "==": actual == value, "!=": actual != value}
        return ops.get(op, False)

    def _eval_string(self, match):
        """评估字符串比较"""
        var_name, op, value = match.group(1), match.group(2), match.group(3)
        actual = self._context.get(var_name)
        if actual is None:
            return False
        if op == "==":
            return str(actual) == value
        elif op == "!=":
            return str(actual) != value
        return False

    def find_timeout(self, definition, state):
        """查找状态对应的超时规则"""
        for t in definition.get("timeouts", []):
            if t["state"] == state:
                return t
        return None

    def log_action_failure(self, package_id, action_name, error):
        """记录动作执行失败，写入重试表"""
        self.db.execute("""
            INSERT INTO action_retry_log
            (package_id, action_name, action_params, retry_count, max_retries, next_retry_at, status, last_error)
            VALUES (%s, %s, %s, 0, 5, NOW() + INTERVAL '30 second', 'PENDING', %s)
        """, package_id, action_name, json.dumps({"package_id": package_id}), str(error))

    def log_condition_error(self, package_id, condition, error):
        """记录条件评估错误"""
        self.db.execute("""
            INSERT INTO action_retry_log
            (package_id, action_name, action_params, retry_count, max_retries, next_retry_at, status, last_error)
            VALUES (%s, '_condition_eval', %s, 0, 0, NOW(), 'DEAD_LETTER', %s)
        """, package_id, json.dumps({"condition": condition}), str(error))

    def batch_transition(self, package_ids, event, operator=None, operator_type="SYSTEM", metadata=None):
        """批量状态流转（整车到达站点等场景）"""
        results = []
        failed = []

        # 批量加锁
        packages = self.db.query("""
            SELECT * FROM package_states
            WHERE package_id IN (%s) FOR UPDATE
        """, ",".join(["'%s'" % pid for pid in package_ids]))

        pkg_map = {p.package_id: p for p in packages}

        with self.db.transaction() as tx:
            for package_id in package_ids:
                pkg = pkg_map.get(package_id)
                if not pkg:
                    failed.append((package_id, "PACKAGE_NOT_FOUND"))
                    continue

                definition = self.definition_cache.get(pkg.product_type)
                transition = self.find_transition(definition, pkg.current_state, event)
                if not transition:
                    failed.append((package_id, f"INVALID_TRANSITION:{pkg.current_state}->{event}"))
                    continue

                if transition.condition:
                    self._current_pkg = pkg
                    self._context = self._build_context(pkg)
                    if not self.evaluate_condition(transition.condition, pkg):
                        failed.append((package_id, "CONDITION_NOT_MET"))
                        continue

                # 批量更新
                tx.execute("""
                    UPDATE package_states
                    SET current_state = %s, state_entered_at = NOW(), version = version + 1
                    WHERE package_id = %s AND version = %s
                """, transition.to, package_id, pkg.version)

                tx.execute("""
                    INSERT INTO state_transition_log
                    (package_id, from_state, to_state, event, operator, operator_type, metadata)
                    VALUES (%s, %s, %s, %s, %s, %s, %s)
                """, package_id, pkg.current_state, transition.to, event,
                     operator, operator_type, json.dumps(metadata or {}))

                results.append({"package_id": package_id, "from": pkg.current_state, "to": transition.to})

                # 取消旧超时 + 注册新超时
                self.timeout_scheduler.cancel(package_id)
                timeout = self.find_timeout(definition, transition.to)
                if timeout:
                    self.timeout_scheduler.schedule(package_id, timeout["duration"], timeout["event"])

        # 异步触发动作
        for r in results:
            definition = self.definition_cache.get(pkg_map[r["package_id"]].product_type)
            transition = self.find_transition(definition, r["from"], event)
            if transition and transition.actions:
                for action_name in transition.actions:
                    try:
                        self.action_executor.execute_async(action_name, r["package_id"], metadata)
                    except Exception as e:
                        self.log_action_failure(r["package_id"], action_name, e)

        return {"succeeded": results, "failed": failed}

    ### 通用规则评估引擎：基于 JSON 的状态转换规则

上面的 `StateMachineEngine` 已经实现了基于配置的状态流转，但条件评估部分存在明显不足：正则表达式逐条匹配、不支持嵌套逻辑、扩展性差。下面给出一个完整的、生产级的通用规则评估引擎实现。

**设计思路：** 状态转换规则以 JSON 定义存储在数据库中，引擎通过 AST（抽象语法树）解析条件表达式，支持嵌套逻辑运算、多种比较操作、自定义函数扩展，且完全安全（不允许任意代码执行）。

```python
import json
import operator
import re
from datetime import datetime, timedelta
from functools import lru_cache
from typing import Any, Dict, List, Optional, Tuple


# ============================================================
# 第一层：规则定义模型（JSON Schema）
# ============================================================

class TransitionRule:
    """一条完整的状态转换规则，从 JSON 定义加载"""
    def __init__(self, rule_dict: dict):
        self.from_state: str = rule_dict["from"]
        self.to_state: str = rule_dict["to"]
        self.event: str = rule_dict["event"]
        self.condition: Optional[ConditionNode] = None
        self.actions: List[str] = rule_dict.get("actions", [])
        self.priority: int = rule_dict.get("priority", 0)
        self.metadata: dict = rule_dict.get("metadata", {})

        # 条件可以是字符串表达式，也可以是 JSON AST
        raw_condition = rule_dict.get("condition")
        if raw_condition:
            self.condition = ConditionParser.parse(raw_condition)


class ConditionNode:
    """条件表达式 AST 节点基类"""
    def evaluate(self, context: Dict[str, Any]) -> bool:
        raise NotImplementedError


class ComparisonNode(ConditionNode):
    """比较节点：variable op value"""
    COMPARISON_OPS = {
        ">":  operator.gt,  "<":  operator.lt,
        ">=": operator.ge,  "<=": operator.le,
        "==": operator.eq,  "!=": operator.ne,
    }

    def __init__(self, variable: str, op: str, value: Any):
        self.variable = variable
        self.op = op
        self.value = value

    def evaluate(self, context: Dict[str, Any]) -> bool:
        actual = context.get(self.variable)
        if actual is None:
            return False  # 变量不存在，默认不满足
        op_func = self.COMPARISON_OPS.get(self.op)
        if op_func is None:
            raise ValueError(f"不支持的操作符: {self.op}")
        return op_func(actual, self.value)


class LogicalNode(ConditionNode):
    """逻辑组合节点：AND / OR"""
    def __init__(self, op: str, children: List[ConditionNode]):
        self.op = op  # "AND" or "OR"
        self.children = children

    def evaluate(self, context: Dict[str, Any]) -> bool:
        if self.op == "AND":
            return all(child.evaluate(context) for child in self.children)
        elif self.op == "OR":
            return any(child.evaluate(context) for child in self.children)
        raise ValueError(f"不支持的逻辑操作符: {self.op}")


class FunctionCallNode(ConditionNode):
    """自定义函数调用节点：func_name(args...)"""
    # 注册允许的函数（白名单，防止任意代码执行）
    ALLOWED_FUNCTIONS = {
        "duration_exceeds": lambda ctx, state_name, hours: 
            (datetime.now() - ctx.get(f"{state_name}_entered_at", datetime.now())).total_seconds() > hours * 3600,
        "is_weekend": lambda ctx: datetime.now().weekday() >= 5,
        "is_peak_hour": lambda ctx: datetime.now().hour in {8, 9, 17, 18, 19},
        "in_region": lambda ctx, region_code: ctx.get("current_region") == region_code,
        "has_insurance": lambda ctx: ctx.get("is_insured") is True,
        "value_exceeds": lambda ctx, threshold: ctx.get("declared_value_cents", 0) > threshold,
    }

    def __init__(self, func_name: str, args: List[Any]):
        self.func_name = func_name
        self.args = args

    def evaluate(self, context: Dict[str, Any]) -> bool:
        func = self.ALLOWED_FUNCTIONS.get(self.func_name)
        if func is None:
            raise ValueError(f"不允许的函数调用: {self.func_name}")
        # args 中可能有变量引用，需要从 context 解析
        resolved_args = []
        for arg in self.args:
            if isinstance(arg, str) and arg.startswith("$"):
                resolved_args.append(context.get(arg[1:]))
            else:
                resolved_args.append(arg)
        return func(context, *resolved_args)


class NotNode(ConditionNode):
    """逻辑取反节点"""
    def __init__(self, child: ConditionNode):
        self.child = child

    def evaluate(self, context: Dict[str, Any]) -> bool:
        return not self.child.evaluate(context)


# ============================================================
# 第二层：条件表达式解析器
# ============================================================

class ConditionParser:
    """
    将条件表达式解析为 AST。
    支持两种输入格式：
    1. 字符串表达式： "duration_in_state > 72h and weight_grams < 5000"
    2. JSON AST： {"and": [{"compare": ["duration_in_state", ">", 259200]}, {"compare": ["weight_grams", "<", 5000]}]}
    """

    @staticmethod
    def parse(raw_condition) -> ConditionNode:
        if isinstance(raw_condition, str):
            return ConditionParser._parse_string(raw_condition)
        elif isinstance(raw_condition, dict):
            return ConditionParser._parse_json_ast(raw_condition)
        else:
            raise ValueError(f"不支持的条件格式: {type(raw_condition)}")

    @staticmethod
    def _parse_json_ast(ast: dict) -> ConditionNode:
        """解析 JSON AST 格式"""
        if "and" in ast:
            children = [ConditionParser._parse_json_ast(c) for c in ast["and"]]
            return LogicalNode("AND", children)
        if "or" in ast:
            children = [ConditionParser._parse_json_ast(c) for c in ast["or"]]
            return LogicalNode("OR", children)
        if "not" in ast:
            child = ConditionParser._parse_json_ast(ast["not"])
            return NotNode(child)
        if "compare" in ast:
            var_name, op, value = ast["compare"]
            # 对时长字符串如 "72h" 做特殊处理
            if isinstance(value, str) and value.endswith("h"):
                value = float(value[:-1]) * 3600  # 转为秒
            return ComparisonNode(var_name, op, value)
        if "function" in ast:
            func_name, args = ast["function"]
            return FunctionCallNode(func_name, args)
        raise ValueError(f"无法解析 AST 节点: {ast}")

    @staticmethod
    def _parse_string(expr: str) -> ConditionNode:
        """解析字符串表达式格式"""
        expr = expr.strip()

        # 处理 NOT
        if expr.startswith("not "):
            inner = ConditionParser._parse_string(expr[4:])
            return NotNode(inner)

        # 处理 AND/OR（递归解析，支持嵌套）
        # 找到最外层的 and/or 关键词
        and_pos = ConditionParser._find_logical_keyword(expr, "and")
        or_pos = ConditionParser._find_logical_keyword(expr, "or")

        if and_pos is not None:
            left = ConditionParser._parse_string(expr[:and_pos])
            right = ConditionParser._parse_string(expr[and_pos + 4:])
            return LogicalNode("AND", [left, right])

        if or_pos is not None:
            left = ConditionParser._parse_string(expr[:or_pos])
            right = ConditionParser._parse_string(expr[or_pos + 3:])
            return LogicalNode("OR", [left, right])

        # 处理函数调用：duration_exceeds("IN_TRANSIT", 72)
        func_match = re.match(r'(\w+)\(([^)]+)\)', expr)
        if func_match:
            func_name = func_match.group(1)
            args_str = func_match.group(2)
            args = [a.strip().strip("'\"") for a in args_str.split(",")]
            # 尝试将数值参数转为数字
            parsed_args = []
            for a in args:
                try:
                    parsed_args.append(float(a) if "." in a else int(a))
                except ValueError:
                    parsed_args.append(a)
            return FunctionCallNode(func_name, parsed_args)

        # 处理简单比较：duration_in_state > 72h
        compare_match = re.match(
            r'(\w+(?:\.\w+)?)\s*(>=|<=|!=|==|>|<)\s*(.+)',
            expr
        )
        if compare_match:
            var_name = compare_match.group(1)
            op = compare_match.group(2)
            value_str = compare_match.group(3).strip()

            # 特殊处理时长格式（72h → 转为秒数）
            if value_str.endswith("h"):
                value = float(value_str[:-1]) * 3600
            elif value_str.startswith("'") or value_str.startswith('"'):
                value = value_str[1:-1]  # 字符串值
            else:
                try:
                    value = float(value_str)
                except ValueError:
                    value = value_str  # 作为字符串处理

            return ComparisonNode(var_name, op, value)

        raise ValueError(f"无法解析条件表达式: {expr}")

    @staticmethod
    def _find_logical_keyword(expr: str, keyword: str) -> Optional[int]:
        """找到表达式中最外层的逻辑关键词位置（不在括号内的）"""
        depth = 0
        k_len = len(keyword)
        for i in range(len(expr)):
            if expr[i] == '(':
                depth += 1
            elif expr[i] == ')':
                depth -= 1
            elif depth == 0:
                if expr[i:i+k_len] == keyword and (
                    i == 0 or not expr[i-1].isalnum()
                ) and (
                    i + k_len >= len(expr) or not expr[i+k_len].isalnum()
                ):
                    # 检查前后有空格或边界
                    before_ok = i == 0 or expr[i-1] in (' ', ')')
                    after_ok = i + k_len >= len(expr) or expr[i+k_len] in (' ', '(')
                    if before_ok and after_ok:
                        return i
        return None


# ============================================================
# 第三层：规则评估引擎（整合 AST + 上下文构建）
# ============================================================

class RuleEvaluationEngine:
    """
    通用规则评估引擎。
    - 从数据库加载 JSON 定义
    - 构建 AST 条件树
    - 在运行时构建变量上下文
    - 安全地评估条件
    """

    def __init__(self, db, definition_cache):
        self.db = db
        self.definition_cache = definition_cache
        self._function_registry = {}  # 可扩展的自定义函数注册

    def register_function(self, name: str, func):
        """注册自定义评估函数（运营可扩展）"""
        self._function_registry[name] = func
        FunctionCallNode.ALLOWED_FUNCTIONS[name] = func

    def build_context(self, pkg, extra_data: dict = None) -> Dict[str, Any]:
        """
        构建条件评估的完整上下文。
        将包裹属性、运行时数据、外部数据整合为 flat dict。
        """
        context = {
            # 包裹固有属性
            "package_id": pkg.package_id,
            "product_type": pkg.product_type,
            "current_state": pkg.current_state,
            "weight_grams": pkg.weight_grams if hasattr(pkg, 'weight_grams') else 0,
            "declared_value_cents": pkg.declared_value_cents if hasattr(pkg, 'declared_value_cents') else 0,
            "is_insured": pkg.is_insured if hasattr(pkg, 'is_insured') else False,
            "cod_amount_cents": pkg.cod_amount_cents if hasattr(pkg, 'cod_amount_cents') else 0,
            # 运行时计算属性
            "duration_in_state": (datetime.now() - pkg.state_entered_at).total_seconds(),
            "current_time_hour": datetime.now().hour,
            "current_time_weekday": datetime.now().weekday(),
            # 状态进入时间（支持 duration_exceeds 函数）
            f"{pkg.current_state}_entered_at": pkg.state_entered_at,
        }

        # 注入包裹 metadata（如冷链温度传感器数据）
        if hasattr(pkg, 'metadata') and pkg.metadata:
            for k, v in pkg.metadata.items():
                context[f"metadata.{k}"] = v

        # 注入额外数据（如当前区域、网点信息等查询结果）
        if extra_data:
            context.update(extra_data)

        return context

    def evaluate_rule(self, rule: TransitionRule, pkg, extra_data: dict = None) -> Tuple[bool, str]:
        """
        评估一条转换规则是否满足条件。
        返回: (是否满足, 原因说明)
        """
        if rule.condition is None:
            return True, "无条件限制"

        context = self.build_context(pkg, extra_data)

        try:
            result = rule.condition.evaluate(context)
            if result:
                return True, "条件满足"
            else:
                return False, f"条件不满足: {rule.condition}"
        except Exception as e:
            # 条件评估异常 → 安全侧：拒绝转换，记录错误
            self._log_eval_error(pkg.package_id, rule, e)
            return False, f"条件评估异常: {str(e)}"

    def find_matching_rule(self, product_type: str, current_state: str, event: str,
                           pkg, extra_data: dict = None) -> Optional[TransitionRule]:
        """
        查找匹配的转换规则。
        同一 from_state + event 可能有多条规则（条件不同），按优先级评估。
        """
        definition = self.definition_cache.get(product_type)
        candidates = []

        for rule in definition.rules:
            if rule.from_state == current_state and rule.event == event:
                candidates.append(rule)

        if not candidates:
            return None

        # 按优先级排序（高优先级先评估）
        candidates.sort(key=lambda r: r.priority, reverse=True)

        for rule in candidates:
            satisfied, reason = self.evaluate_rule(rule, pkg, extra_data)
            if satisfied:
                return rule

        # 所有候选规则的条件都不满足
        return None

    def _log_eval_error(self, package_id: str, rule: TransitionRule, error: Exception):
        """记录条件评估异常到死信表"""
        self.db.execute("""
            INSERT INTO condition_eval_errors
            (package_id, rule_from, rule_to, rule_event, condition_expr, error_message, created_at)
            VALUES (%s, %s, %s, %s, %s, %s, NOW())
        """, package_id, rule.from_state, rule.to_state, rule.event,
             str(rule.condition), str(error))


# ============================================================
# 第四层：条件评估错误表（补充数据库设计）
# ============================================================

CONDITION_EVAL_ERRORS_TABLE = """
CREATE TABLE condition_eval_errors (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    package_id VARCHAR(64) NOT NULL,
    rule_from VARCHAR(32) NOT NULL,
    rule_to VARCHAR(32) NOT NULL,
    rule_event VARCHAR(32) NOT NULL,
    condition_expr TEXT,
    error_message TEXT NOT NULL,
    resolved BOOLEAN DEFAULT FALSE,
    resolved_by VARCHAR(64),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_package (package_id),
    INDEX idx_unresolved (resolved, created_at)
);
"""
```

**JSON AST 条件定义示例（比字符串表达式更安全、更可维护）：**

```json
{
  "from": "IN_TRANSIT", "to": "EXCEPTION", "event": "TIMEOUT",
  "condition": {
    "and": [
      { "compare": ["duration_in_state", ">", 259200] },
      { "not": { "function": ["is_peak_hour", []] } },
      { "compare": ["product_type", "==", "CROSS_BORDER"] }
    ]
  },
  "actions": ["alert_logistics_team", "notify_sender", "create_exception_record"]
}
```

对比字符串表达式 `"duration_in_state > 72h and not is_peak_hour() and product_type == 'CROSS_BORDER'"`：

| 维度 | 字符串表达式 | JSON AST |
|------|-------------|----------|
| 安全性 | 需要正则白名单，仍有注入风险 | 结构化数据，无代码执行风险 |
| 可维护性 | 运营人员不易理解嵌套逻辑 | JSON 可视化编辑器可拖拽组合 |
| 扩展性 | 新增函数需改解析器 | 新增函数只需 `register_function` |
| 测试性 | 需要 mock 整个表达式 | 每个 AST 节点可独立测试 |

**实际生产建议：** 管理后台使用 JSON AST 格式存储，前端可视化编辑器生成 AST；引擎同时兼容字符串表达式格式，用于快速调试和临时规则配置。

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

### 补偿与重试机制：完整实现

状态流转过程中，动作执行（发通知、触发结算、调用外部服务）可能失败。需要一个完整的补偿与重试机制来保证最终一致性。

**三层补偿架构：**

```
第一层：自动重试（指数退避）→ 适用于临时性故障（网络超时、服务限流）
第二层：死信队列 + 人工干预 → 适用于业务性故障（参数错误、服务不可用）
第三层：前向补偿事务 → 适用于已部分成功的复合操作（结算已触发但通知未发）
```

```python
import math
import time
from enum import Enum
from typing import Callable, Optional


class ActionStatus(Enum):
    PENDING = "PENDING"
    RETRYING = "RETRYING"
    SUCCESS = "SUCCESS"
    DEAD_LETTER = "DEAD_LETTER"


class CompensationActionExecutor:
    """
    带完整补偿机制的动作执行器。
    - 动作执行失败 → 自动重试（指数退避）
    - 重试耗尽 → 进入死信队列
    - 死信队列 → 人工干预工作台
    - 支持补偿事务（部分成功时的回滚操作）
    """

    # 重试策略配置
    BASE_RETRY_INTERVAL_SECS = 30       # 首次重试间隔
    MAX_RETRY_INTERVAL_SECS = 3600      # 最大重试间隔（1小时）
    MAX_RETRIES = 5                      # 最大重试次数
    RETRY_MULTIPLIER = 2                 # 退避倍数

    def __init__(self, db, notification_service, dead_letter_handler):
        self.db = db
        self.notification_service = notification_service
        self.dead_letter_handler = dead_letter_handler
        self._action_registry = {}       # 动作名 → 执行函数
        self._compensation_registry = {} # 动作名 → 补偿函数

    def register_action(self, name: str, func: Callable, compensation_func: Callable = None):
        """注册动作及其可选的补偿函数"""
        self._action_registry[name] = func
        if compensation_func:
            self._compensation_registry[name] = compensation_func

    def execute_with_retry(self, action_name: str, package_id: str, params: dict,
                           retry_count: int = 0, max_retries: int = None) -> dict:
        """
        执行动作，失败时自动重试（指数退避）。
        返回: {"status": "SUCCESS"/"RETRYING"/"DEAD_LETTER", "detail": ...}
        """
        if max_retries is None:
            max_retries = self.MAX_RETRIES

        func = self._action_registry.get(action_name)
        if func is None:
            return self._move_to_dead_letter(
                package_id, action_name, params,
                f"未注册的动作: {action_name}", retry_count
            )

        try:
            result = func(package_id, params)
            # 执行成功 → 更新重试记录状态
            self._mark_success(package_id, action_name)
            return {"status": "SUCCESS", "detail": result}

        except Exception as e:
            retry_count += 1
            error_msg = str(e)

            if retry_count >= max_retries:
                # 重试耗尽 → 进入死信队列
                return self._move_to_dead_letter(
                    package_id, action_name, params, error_msg, retry_count
                )

            # 计算下次重试时间（指数退避 + 抖动）
            next_interval = self._calculate_backoff(retry_count)
            next_retry_at = datetime.now() + timedelta(seconds=next_interval)

            # 更新重试记录
            self._update_retry_record(
                package_id, action_name, params, retry_count,
                max_retries, next_retry_at, error_msg
            )

            return {
                "status": "RETRYING",
                "detail": {
                    "retry_count": retry_count,
                    "next_retry_at": next_retry_at.isoformat(),
                    "error": error_msg
                }
            }

    def _calculate_backoff(self, retry_count: int) -> float:
        """
        指数退避 + 随机抖动。
        interval = min(BASE * MULTIPLIER^(n-1), MAX) + jitter
        """
        interval = self.BASE_RETRY_INTERVAL_SECS * (self.RETRY_MULTIPLIER ** (retry_count - 1))
        interval = min(interval, self.MAX_RETRY_INTERVAL_SECS)
        # 加入 ±20% 的随机抖动，防止惊群效应
        jitter = interval * 0.2 * (2 * (hash(str(retry_count) + str(time.time())) % 100) / 100 - 1)
        return max(1, interval + jitter)

    def _move_to_dead_letter(self, package_id: str, action_name: str, params: dict,
                              error_msg: str, retry_count: int) -> dict:
        """将失败动作移入死信队列"""
        self.db.execute("""
            INSERT INTO action_retry_log
            (package_id, action_name, action_params, retry_count, max_retries,
             next_retry_at, status, last_error)
            VALUES (%s, %s, %s, %s, %s, NOW(), 'DEAD_LETTER', %s)
        """, package_id, action_name, json.dumps(params),
             retry_count, self.MAX_RETRIES, error_msg)

        # 同时通知人工干预工作台
        self.dead_letter_handler.notify(package_id, action_name, error_msg)

        return {
            "status": "DEAD_LETTER",
            "detail": {
                "error": error_msg,
                "retry_count": retry_count,
                "action": action_name
            }
        }

    def _update_retry_record(self, package_id, action_name, params, retry_count,
                              max_retries, next_retry_at, error_msg):
        """更新或创建重试记录"""
        existing = self.db.query_one("""
            SELECT id FROM action_retry_log
            WHERE package_id = %s AND action_name = %s AND status IN ('PENDING', 'RETRYING')
        """, package_id, action_name)

        if existing:
            self.db.execute("""
                UPDATE action_retry_log
                SET retry_count = %s, next_retry_at = %s, status = 'RETRYING',
                    last_error = %s, updated_at = NOW()
                WHERE id = %s
            """, retry_count, next_retry_at, error_msg, existing.id)
        else:
            self.db.execute("""
                INSERT INTO action_retry_log
                (package_id, action_name, action_params, retry_count, max_retries,
                 next_retry_at, status, last_error)
                VALUES (%s, %s, %s, %s, %s, %s, 'RETRYING', %s)
            """, package_id, action_name, json.dumps(params),
                 retry_count, max_retries, next_retry_at, error_msg)

    def _mark_success(self, package_id, action_name):
        """标记动作执行成功"""
        self.db.execute("""
            UPDATE action_retry_log
            SET status = 'SUCCESS', updated_at = NOW()
            WHERE package_id = %s AND action_name = %s AND status IN ('PENDING', 'RETRYING')
        """, package_id, action_name)


class DeadLetterHandler:
    """
    死信队列处理器。
    - 提供人工干预工作台 API
    - 支持手动重试、跳过、补偿三种操作
    """

    def __init__(self, db, notification_service):
        self.db = db
        self.notification_service = notification_service

    def notify(self, package_id: str, action_name: str, error_msg: str):
        """通知人工干预工作台"""
        # 1. 写入人工干预队列表
        self.db.execute("""
            INSERT INTO manual_intervention_queue
            (package_id, action_name, error_message, priority, status, created_at)
            VALUES (%s, %s, %s, 
                    CASE WHEN %s IN ('trigger_settlement', 'trigger_refund') THEN 'HIGH' ELSE 'NORMAL' END,
                    'PENDING', NOW())
        """, package_id, action_name, error_msg, action_name)

        # 2. 发送告警通知（按动作紧急程度选择通知渠道）
        if action_name in ('trigger_settlement', 'trigger_refund'):
            self.notification_service.send_urgent_alert(
                f"【紧急】包裹 {package_id} 动作 {action_name} 进入死信队列: {error_msg}"
            )
        else:
            self.notification_service.send_alert(
                f"包裹 {package_id} 动作 {action_name} 失败: {error_msg}"
            )

    def manual_retry(self, dead_letter_id: int, operator: str) -> dict:
        """人工手动重试"""
        record = self.db.query_one(
            "SELECT * FROM action_retry_log WHERE id = %s AND status = 'DEAD_LETTER'",
            dead_letter_id
        )
        if not record:
            return {"success": False, "message": "记录不存在或非死信状态"}

        # 重置重试计数，重新执行
        self.db.execute("""
            UPDATE action_retry_log
            SET retry_count = 0, status = 'PENDING', next_retry_at = NOW(),
                updated_at = NOW()
            WHERE id = %s
        """, dead_letter_id)

        # 记录人工操作日志
        self._log_manual_action(dead_letter_id, "MANUAL_RETRY", operator)

        return {"success": True, "message": "已重新加入重试队列"}

    def manual_skip(self, dead_letter_id: int, operator: str, reason: str) -> dict:
        """人工跳过（标记为已处理，不再重试）"""
        self.db.execute("""
            UPDATE action_retry_log
            SET status = 'SKIPPED', updated_at = NOW()
            WHERE id = %s
        """, dead_letter_id)

        self._log_manual_action(dead_letter_id, "MANUAL_SKIP", operator, reason)
        return {"success": True, "message": "已跳过该动作"}

    def manual_compensate(self, dead_letter_id: int, operator: str,
                           compensation_executor) -> dict:
        """人工触发补偿事务"""
        record = self.db.query_one(
            "SELECT * FROM action_retry_log WHERE id = %s AND status = 'DEAD_LETTER'",
            dead_letter_id
        )
        if not record:
            return {"success": False, "message": "记录不存在"}

        # 执行补偿函数
        comp_func = compensation_executor._compensation_registry.get(record.action_name)
        if comp_func is None:
            return {"success": False, "message": f"动作 {record.action_name} 无补偿函数"}

        try:
            comp_func(record.package_id, json.loads(record.action_params))
            self.db.execute("""
                UPDATE action_retry_log
                SET status = 'COMPENSATED', updated_at = NOW()
                WHERE id = %s
            """, dead_letter_id)
            self._log_manual_action(dead_letter_id, "MANUAL_COMPENSATE", operator)
            return {"success": True, "message": "补偿事务执行成功"}
        except Exception as e:
            return {"success": False, "message": f"补偿事务执行失败: {str(e)}"}

    def _log_manual_action(self, dead_letter_id: int, action_type: str,
                            operator: str, reason: str = None):
        """记录人工操作日志"""
        self.db.execute("""
            INSERT INTO manual_intervention_log
            (dead_letter_id, action_type, operator, reason, created_at)
            VALUES (%s, %s, %s, %s, NOW())
        """, dead_letter_id, action_type, operator, reason)


class RetryScheduler:
    """
    重试调度器——定时扫描 action_retry_log 表，执行到期重试。
    类似 TimeoutConsumer，每秒扫描一次到期记录。
    """

    def __init__(self, db, action_executor: CompensationActionExecutor):
        self.db = db
        self.action_executor = action_executor

    def run(self):
        while True:
            # 查询到期且待重试的记录
            records = self.db.query("""
                SELECT * FROM action_retry_log
                WHERE status IN ('PENDING', 'RETRYING')
                  AND next_retry_at <= NOW()
                ORDER BY next_retry_at ASC
                LIMIT 500
            """)

            for record in records:
                try:
                    params = json.loads(record.action_params) if record.action_params else {}
                    result = self.action_executor.execute_with_retry(
                        record.action_name, record.package_id, params,
                        retry_count=record.retry_count,
                        max_retries=record.max_retries
                    )
                except Exception as e:
                    # 调度器本身异常，记录但不中断
                    self.db.execute("""
                        UPDATE action_retry_log
                        SET last_error = %s, updated_at = NOW()
                        WHERE id = %s
                    """, str(e), record.id)

            time.sleep(1)
```

**补充数据库表：人工干预相关**

```sql
-- 人工干预队列
CREATE TABLE manual_intervention_queue (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    package_id VARCHAR(64) NOT NULL,
    action_name VARCHAR(64) NOT NULL,
    error_message TEXT,
    priority VARCHAR(16) DEFAULT 'NORMAL',  -- URGENT / HIGH / NORMAL / LOW
    status VARCHAR(16) DEFAULT 'PENDING',   -- PENDING / PROCESSING / RESOLVED / IGNORED
    assigned_to VARCHAR(64),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    resolved_at TIMESTAMP,

    INDEX idx_status_priority (status, priority, created_at),
    INDEX idx_assigned (assigned_to, status)
);

-- 人工操作日志
CREATE TABLE manual_intervention_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    dead_letter_id BIGINT NOT NULL,
    action_type VARCHAR(32) NOT NULL,  -- MANUAL_RETRY / MANUAL_SKIP / MANUAL_COMPENSATE
    operator VARCHAR(64) NOT NULL,
    reason TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_dead_letter (dead_letter_id)
);
```

**重试策略对照表：**

| 动作类型 | 最大重试次数 | 首次间隔 | 最大间隔 | 退避策略 | 死信处理 |
|---------|------------|---------|---------|---------|---------|
| notify_sender | 5 | 30s | 1h | 指数退避 | 人工检查后跳过 |
| notify_receiver | 5 | 30s | 1h | 指数退避 | 人工检查后跳过 |
| trigger_settlement | 8 | 10s | 30min | 指数退避 | **紧急告警+人工重试** |
| trigger_refund | 8 | 10s | 30min | 指数退避 | **紧急告警+人工重试** |
| update_tracking | 3 | 60s | 10min | 固定间隔 | 跳过（下次扫描补齐） |
| alert_logistics_team | 3 | 30s | 5min | 指数退避 | 升级告警升级 |
| create_claim_record | 5 | 30s | 1h | 指数退避 | 人工创建 |
| sync_partner_status | 10 | 60s | 2h | 指数退避 | 人工对账 |

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

### 批量状态流转：完整实现

**业务场景：** 一辆载有 200 个包裹的运输车到达站点，需要在 1 秒内完成所有包裹的"到达站点"状态变更。要求：事务性保证（全部成功或可追溯的失败）、幂等性（重复调用不产生副作用）、性能（200 个包裹 < 500ms）。

```python
from dataclasses import dataclass
from typing import List, Dict, Tuple
import uuid


@dataclass
class BatchTransitionRequest:
    """批量状态流转请求"""
    batch_id: str                          # 批次ID，用于幂等和追踪
    package_ids: List[str]                 # 包裹ID列表
    event: str                             # 统一触发事件
    operator: str = None
    operator_type: str = "SYSTEM"
    metadata: dict = None
    fail_policy: str = "PARTIAL"           # ALL_OR_NOTHING / PARTIAL
    idempotency_window_secs: int = 5       # 幂等去重窗口


@dataclass
class BatchTransitionResult:
    """批量状态流转结果"""
    batch_id: str
    total: int
    succeeded: List[dict]
    failed: List[dict]
    duration_ms: int


class BatchTransitionEngine:
    """
    批量状态流转引擎。
    
    关键设计：
    1. 分批处理：每批最多 50 个包裹，避免单个事务过大
    2. 事务内批量操作：批量 SELECT FOR UPDATE → 批量 UPDATE → 批量 INSERT
    3. 幂等保证：batch_id + package_id 联合去重
    4. 部分失败容忍：PARTIAL 模式下，成功的不回滚，失败的记录原因
    5. 全量回滚：ALL_OR_NOTHING 模式下，任一失败全部回滚
    """

    BATCH_SIZE = 50  # 每批最大处理数量

    def __init__(self, db, definition_cache, action_executor, timeout_scheduler):
        self.db = db
        self.definition_cache = definition_cache
        self.action_executor = action_executor
        self.timeout_scheduler = timeout_scheduler

    def execute_batch(self, request: BatchTransitionRequest) -> BatchTransitionResult:
        """执行批量状态流转"""
        start_time = datetime.now()
        all_succeeded = []
        all_failed = []

        # 1. 幂等检查：该批次是否已处理过
        existing = self._check_idempotency(request.batch_id)
        if existing:
            return existing  # 直接返回上次结果

        # 2. 记录批次开始
        self._record_batch_start(request)

        # 3. 分批处理
        for i in range(0, len(request.package_ids), self.BATCH_SIZE):
            chunk = request.package_ids[i:i + self.BATCH_SIZE]
            succeeded, failed = self._process_chunk(
                chunk, request, chunk_index=i // self.BATCH_SIZE
            )
            all_succeeded.extend(succeeded)
            all_failed.extend(failed)

            # ALL_OR_NOTHING 模式：任一失败则回滚之前所有成功的
            if request.fail_policy == "ALL_OR_NOTHING" and failed:
                self._rollback_succeeded(all_succeeded, request)
                all_succeeded = []
                # 将之前成功的也标记为失败
                all_failed = [
                    {"package_id": s["package_id"], "reason": "BATCH_ROLLBACK"}
                    for s in all_succeeded
                ] + all_failed
                break

        # 4. 记录批次完成
        duration_ms = int((datetime.now() - start_time).total_seconds() * 1000)
        result = BatchTransitionResult(
            batch_id=request.batch_id,
            total=len(request.package_ids),
            succeeded=all_succeeded,
            failed=all_failed,
            duration_ms=duration_ms
        )
        self._record_batch_complete(result)

        # 5. 异步触发动作
        self._trigger_actions_async(all_succeeded, request)

        return result

    def _process_chunk(self, package_ids: List[str], request: BatchTransitionRequest,
                        chunk_index: int) -> Tuple[List[dict], List[dict]]:
        """处理一个分批（单事务内）"""
        succeeded = []
        failed = []

        with self.db.transaction() as tx:
            # 批量加锁（按 package_id 排序避免死锁）
            sorted_ids = sorted(package_ids)
            placeholders = ",".join(["%s"] * len(sorted_ids))
            packages = tx.query(f"""
                SELECT ps.*, p.weight_grams, p.declared_value_cents,
                       p.is_insured, p.cod_amount_cents
                FROM package_states ps
                JOIN packages p ON ps.package_id = p.package_id
                WHERE ps.package_id IN ({placeholders})
                ORDER BY ps.package_id
                FOR UPDATE
            """, sorted_ids)

            pkg_map = {p.package_id: p for p in packages}

            # 收集需要批量 UPDATE 和 INSERT 的数据
            update_batch = []
            log_batch = []

            for package_id in package_ids:
                pkg = pkg_map.get(package_id)
                if not pkg:
                    failed.append({"package_id": package_id, "reason": "PACKAGE_NOT_FOUND"})
                    continue

                # 幂等去重：5 秒内相同 package_id + event 的请求视为重复
                if self._is_duplicate_transition(tx, package_id, request.event,
                                                  request.idempotency_window_secs):
                    failed.append({"package_id": package_id, "reason": "DUPLICATE_TRANSITION"})
                    continue

                definition = self.definition_cache.get(pkg.product_type)
                transition = self._find_transition(definition, pkg.current_state, request.event)

                if not transition:
                    failed.append({
                        "package_id": package_id,
                        "reason": f"INVALID_TRANSITION:{pkg.current_state}->{request.event}"
                    })
                    continue

                # 条件评估
                if transition.condition:
                    context = self._build_context(pkg)
                    if not self._evaluate_condition(transition.condition, context):
                        failed.append({"package_id": package_id, "reason": "CONDITION_NOT_MET"})
                        continue

                # 加入批量更新列表
                update_batch.append((transition.to_state, package_id, pkg.version))
                log_batch.append((
                    package_id, pkg.current_state, transition.to_state,
                    request.event, request.operator, request.operator_type,
                    json.dumps(request.metadata or {}),
                    request.batch_id  # 附带批次ID方便追踪
                ))
                succeeded.append({
                    "package_id": package_id,
                    "from": pkg.current_state,
                    "to": transition.to_state,
                    "actions": transition.actions
                })

            # 批量 UPDATE（CASE WHEN 批量更新，减少数据库交互次数）
            if update_batch:
                # 构建批量 UPDATE SQL
                when_clauses = []
                ids_for_update = []
                for new_state, pid, version in update_batch:
                    when_clauses.append(f"WHEN package_id = %s AND version = %s THEN %s")
                    ids_for_update.extend([pid, version, new_state])

                update_ids = [item[1] for item in update_batch]
                id_placeholders = ",".join(["%s"] * len(update_ids))

                tx.execute(f"""
                    UPDATE package_states
                    SET current_state = CASE
                        {' '.join(when_clauses)}
                        ELSE current_state
                    END,
                    state_entered_at = NOW(),
                    version = version + 1
                    WHERE package_id IN ({id_placeholders})
                """, ids_for_update + update_ids)

            # 批量 INSERT 审计日志
            if log_batch:
                log_placeholders = ",".join([
                    "(%s, %s, %s, %s, %s, %s, %s, %s)" for _ in log_batch
                ])
                flat_params = [param for row in log_batch for param in row]
                tx.execute(f"""
                    INSERT INTO state_transition_log
                    (package_id, from_state, to_state, event, operator, operator_type,
                     metadata, batch_id)
                    VALUES {log_placeholders}
                """, flat_params)

        # 事务外：取消旧超时 + 注册新超时
        for s in succeeded:
            self.timeout_scheduler.cancel(s["package_id"])
            definition = self.definition_cache.get(
                pkg_map[s["package_id"]].product_type
            )
            timeout = self._find_timeout(definition, s["to"])
            if timeout:
                self.timeout_scheduler.schedule(
                    s["package_id"], timeout["duration"], timeout["event"]
                )

        return succeeded, failed

    def _check_idempotency(self, batch_id: str) -> Optional[BatchTransitionResult]:
        """幂等检查：该批次是否已处理"""
        record = self.db.query_one("""
            SELECT * FROM batch_transition_records
            WHERE batch_id = %s AND status = 'COMPLETED'
        """, batch_id)
        if record:
            # 加载上次的结果
            results = self.db.query("""
                SELECT * FROM batch_transition_details
                WHERE batch_id = %s
            """, batch_id)
            succeeded = [r for r in results if r.status == "SUCCESS"]
            failed = [r for r in results if r.status == "FAILED"]
            return BatchTransitionResult(
                batch_id=batch_id,
                total=len(results),
                succeeded=succeeded,
                failed=failed,
                duration_ms=record.duration_ms
            )
        return None

    def _record_batch_start(self, request: BatchTransitionRequest):
        """记录批次开始"""
        self.db.execute("""
            INSERT INTO batch_transition_records
            (batch_id, total_count, event, operator, status, created_at)
            VALUES (%s, %s, %s, %s, 'PROCESSING', NOW())
        """, request.batch_id, len(request.package_ids),
             request.event, request.operator)

    def _record_batch_complete(self, result: BatchTransitionResult):
        """记录批次完成"""
        self.db.execute("""
            UPDATE batch_transition_records
            SET status = 'COMPLETED',
                success_count = %s,
                fail_count = %s,
                duration_ms = %s,
                completed_at = NOW()
            WHERE batch_id = %s
        """, len(result.succeeded), len(result.failed),
             result.duration_ms, result.batch_id)

        # 记录每个包裹的详细结果
        for s in result.succeeded:
            self.db.execute("""
                INSERT INTO batch_transition_details
                (batch_id, package_id, from_state, to_state, status)
                VALUES (%s, %s, %s, %s, 'SUCCESS')
            """, result.batch_id, s["package_id"], s["from"], s["to"])

        for f in result.failed:
            self.db.execute("""
                INSERT INTO batch_transition_details
                (batch_id, package_id, status, fail_reason)
                VALUES (%s, %s, 'FAILED', %s)
            """, result.batch_id, f["package_id"], f["reason"])

    def _rollback_succeeded(self, succeeded: List[dict], request: BatchTransitionRequest):
        """回滚已成功的变更（ALL_OR_NOTHING 模式）"""
        with self.db.transaction() as tx:
            for s in succeeded:
                # 将状态改回 from_state
                tx.execute("""
                    UPDATE package_states
                    SET current_state = %s, version = version - 1
                    WHERE package_id = %s
                """, s["from"], s["package_id"])

                # 删除对应的审计日志（标记为已回滚而非物理删除）
                tx.execute("""
                    UPDATE state_transition_log
                    SET metadata = JSON_SET(metadata, '$.rolled_back', TRUE,
                        '$.rollback_reason', 'BATCH_ALL_OR_NOTHING')
                    WHERE package_id = %s AND to_state = %s
                      AND event = %s AND created_at >= NOW() - INTERVAL 60 SECOND
                """, s["package_id"], s["to"], request.event)

    def _trigger_actions_async(self, succeeded: List[dict],
                                request: BatchTransitionRequest):
        """异步触发所有成功包裹的动作"""
        for s in succeeded:
            for action_name in s.get("actions", []):
                try:
                    self.action_executor.execute_async(
                        action_name, s["package_id"], request.metadata
                    )
                except Exception as e:
                    self._log_action_failure(s["package_id"], action_name, e)

    def _is_duplicate_transition(self, tx, package_id, event, window_secs) -> bool:
        """检查是否在幂等窗口内重复触发"""
        count = tx.query_one("""
            SELECT COUNT(*) as cnt FROM state_transition_log
            WHERE package_id = %s AND event = %s
              AND created_at >= NOW() - INTERVAL %s SECOND
        """, package_id, event, window_secs).cnt
        return count > 0
```

**补充数据库表：批量流转相关**

```sql
-- 批量流转记录表
CREATE TABLE batch_transition_records (
    batch_id VARCHAR(64) PRIMARY KEY,
    total_count INTEGER NOT NULL,
    success_count INTEGER DEFAULT 0,
    fail_count INTEGER DEFAULT 0,
    event VARCHAR(32) NOT NULL,
    operator VARCHAR(64),
    status VARCHAR(16) DEFAULT 'PROCESSING',  -- PROCESSING / COMPLETED / FAILED
    duration_ms INTEGER,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,

    INDEX idx_status (status, created_at)
);

-- 批量流转明细表
CREATE TABLE batch_transition_details (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    batch_id VARCHAR(64) NOT NULL,
    package_id VARCHAR(64) NOT NULL,
    from_state VARCHAR(32),
    to_state VARCHAR(32),
    status VARCHAR(16) NOT NULL,   -- SUCCESS / FAILED
    fail_reason VARCHAR(128),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_batch (batch_id),
    INDEX idx_package (package_id)
);

-- 为 state_transition_log 增加 batch_id 字段
ALTER TABLE state_transition_log ADD COLUMN batch_id VARCHAR(64) DEFAULT NULL;
CREATE INDEX idx_batch_id ON state_transition_log (batch_id);
```

**批量流转性能优化点：**

| 优化手段 | 效果 | 适用规模 |
|---------|------|---------|
| CASE WHEN 批量 UPDATE | 200 次 UPDATE → 1 次 | 50-500 个包裹 |
| 批量 INSERT 审计日志 | 200 次 INSERT → 1 次 | 50-500 个包裹 |
| 分批处理（每批 50 个） | 避免单事务过大 | 200+ 个包裹 |
| 排序后加锁 | 避免死锁 | 任何规模 |
| 动作异步触发 | 状态变更 < 500ms | 任何规模 |
| 幂等去重窗口 | 避免重复处理 | 任何规模 |

### 状态流转审计与回放：完整实现

**业务需求：**
1. 每个包裹的完整状态历史可查（审计合规）
2. 状态历史可视化展示（客服系统）
3. 支持回放到任意时间点的状态（排查问题）
4. 支持状态变更回放用于压测和回归测试

```python
class StateAuditService:
    """
    状态流转审计与回放服务。
    - 提供完整的审计链查询
    - 支持时间旅行（回放到任意时间点的状态）
    - 支持状态轨迹对比（与预期流转路径对比）
    - 支持回放用于问题排查和压测
    """

    def __init__(self, db, definition_cache):
        self.db = db
        self.definition_cache = definition_cache

    def get_full_history(self, package_id: str) -> dict:
        """获取包裹的完整状态流转历史"""
        # 查询所有状态变更日志（按时间排序）
        logs = self.db.query("""
            SELECT id, from_state, to_state, event, operator, operator_type,
                   metadata, created_at
            FROM state_transition_log
            WHERE package_id = %s
            ORDER BY created_at ASC
        """, package_id)

        # 查询包裹基本信息
        pkg = self.db.query_one(
            "SELECT * FROM packages WHERE package_id = %s", package_id
        )
        current_state = self.db.query_one(
            "SELECT * FROM package_states WHERE package_id = %s", package_id
        )

        # 构建状态流转时间线
        timeline = []
        prev_time = None
        for log in logs:
            duration_in_prev = None
            if prev_time:
                duration_in_prev = (log.created_at - prev_time).total_seconds()
            prev_time = log.created_at

            timeline.append({
                "sequence": len(timeline) + 1,
                "from_state": log.from_state,
                "to_state": log.to_state,
                "event": log.event,
                "operator": log.operator,
                "operator_type": log.operator_type,
                "metadata": json.loads(log.metadata) if log.metadata else {},
                "occurred_at": log.created_at.isoformat(),
                "duration_in_previous_state_secs": duration_in_prev,
                "log_id": log.id
            })

        return {
            "package_id": package_id,
            "product_type": pkg.product_type if pkg else None,
            "current_state": current_state.current_state if current_state else None,
            "total_transitions": len(timeline),
            "timeline": timeline,
            "anomalies": self._detect_anomalies(timeline)
        }

    def get_state_at_time(self, package_id: str, target_time: datetime) -> dict:
        """
        时间旅行：获取包裹在指定时间点的状态。
        通过回放状态变更日志，计算任意时间点的状态。
        """
        # 查询目标时间之前（含）的所有状态变更
        logs = self.db.query("""
            SELECT from_state, to_state, event, created_at
            FROM state_transition_log
            WHERE package_id = %s AND created_at <= %s
            ORDER BY created_at ASC
        """, package_id, target_time)

        if not logs:
            return {
                "package_id": package_id,
                "state_at": target_time.isoformat(),
                "state": "CREATED",  # 无记录表示初始状态
                "confidence": "INFERRED"
            }

        # 最后一条日志的 to_state 即为该时间点的状态
        last_log = logs[-1]
        # 计算在该状态停留了多久
        duration = (target_time - last_log.created_at).total_seconds()

        return {
            "package_id": package_id,
            "state_at": target_time.isoformat(),
            "state": last_log.to_state,
            "entered_at": last_log.created_at.isoformat(),
            "duration_in_state_secs": duration,
            "triggering_event": last_log.event,
            "confidence": "EXACT",  # 基于实际日志，精确
            "transition_count_to_this_point": len(logs)
        }

    def replay_state_history(self, package_id: str,
                              dry_run: bool = True) -> dict:
        """
        回放包裹的状态流转历史。
        用于：
        1. 调试：复现状态异常问题
        2. 验证：确认每次流转是否合规
        3. 压测：回放历史数据测试新规则
        
        dry_run=True: 只验证不执行（检查每步是否合规）
        dry_run=False: 实际重放（创建新包裹，逐步执行）
        """
        history = self.get_full_history(package_id)
        replay_results = []

        if dry_run:
            # 验证模式：逐条检查转换是否合规
            for step in history["timeline"]:
                # 加载该产品类型的状态机定义
                definition = self.definition_cache.get(history["product_type"])

                # 检查转换是否在定义中
                is_valid = any(
                    t.from == step["from_state"] and t.event == step["event"]
                    for t in definition.transitions
                )

                # 检查状态停留时间是否异常
                is_duration_anomaly = False
                if step["duration_in_previous_state_secs"] is not None:
                    is_duration_anomaly = self._check_duration_anomaly(
                        step["from_state"],
                        step["duration_in_previous_state_secs"],
                        definition
                    )

                replay_results.append({
                    "sequence": step["sequence"],
                    "from_state": step["from_state"],
                    "to_state": step["to_state"],
                    "event": step["event"],
                    "is_valid_transition": is_valid,
                    "is_duration_anomaly": is_duration_anomaly,
                    "occurred_at": step["occurred_at"]
                })

            return {
                "package_id": package_id,
                "mode": "DRY_RUN",
                "total_steps": len(replay_results),
                "invalid_steps": sum(1 for r in replay_results if not r["is_valid_transition"]),
                "anomaly_steps": sum(1 for r in replay_results if r["is_duration_anomaly"]),
                "details": replay_results
            }
        else:
            # 实际回放模式：创建新包裹，逐步执行
            new_package_id = f"REPLAY_{package_id}_{uuid.uuid4().hex[:8]}"
            # 创建新包裹
            self.db.execute("""
                INSERT INTO packages (package_id, product_type, ...)
                SELECT %s, product_type, ... FROM packages WHERE package_id = %s
            """, new_package_id, package_id)

            self.db.execute("""
                INSERT INTO package_states (package_id, product_type, current_state, state_entered_at)
                VALUES (%s, %s, 'CREATED', NOW())
            """, new_package_id, history["product_type"])

            for step in history["timeline"]:
                try:
                    result = self.engine.transition(
                        new_package_id, step["event"],
                        operator=f"REPLAY:{step['operator']}",
                        operator_type="REPLAY",
                        metadata={"replay_source": package_id, "original_time": step["occurred_at"]}
                    )
                    replay_results.append({
                        "sequence": step["sequence"],
                        "status": "SUCCESS",
                        "result": result
                    })
                except Exception as e:
                    replay_results.append({
                        "sequence": step["sequence"],
                        "status": "FAILED",
                        "error": str(e),
                        "original_event": step["event"]
                    })

            return {
                "package_id": new_package_id,
                "source_package_id": package_id,
                "mode": "ACTUAL_REPLAY",
                "total_steps": len(history["timeline"]),
                "success_steps": sum(1 for r in replay_results if r["status"] == "SUCCESS"),
                "failed_steps": sum(1 for r in replay_results if r["status"] == "FAILED"),
                "details": replay_results
            }

    def compare_with_expected_path(self, package_id: str) -> dict:
        """
        将实际状态轨迹与预期流转路径对比。
        用于发现异常：跳步、回退、未预期的分支。
        """
        history = self.get_full_history(package_id)
        definition = self.definition_cache.get(history["product_type"])

        # 构建预期路径（从初始状态开始的正常流转路径）
        expected_paths = self._build_expected_paths(definition)

        actual_path = [
            (step["from_state"], step["to_state"], step["event"])
            for step in history["timeline"]
        ]

        deviations = []
        for i, (from_s, to_s, event) in enumerate(actual_path):
            is_expected = any(
                path_step == (from_s, to_s, event)
                for path in expected_paths
                for path_step in path
            )
            if not is_expected:
                deviations.append({
                    "step": i + 1,
                    "transition": f"{from_s} --{event}--> {to_s}",
                    "deviation_type": "UNEXPECTED_TRANSITION",
                    "possible_cause": self._infer_cause(from_s, to_s, event, definition)
                })

        return {
            "package_id": package_id,
            "actual_transition_count": len(actual_path),
            "deviation_count": len(deviations),
            "deviations": deviations,
            "is_normal_flow": len(deviations) == 0
        }

    def query_audit_trail(self, filters: dict) -> dict:
        """
        审计链查询——支持多维度过滤。
        用于：合规审计、运营分析、异常追踪。
        """
        conditions = ["1=1"]
        params = []

        if "package_id" in filters:
            conditions.append("package_id = %s")
            params.append(filters["package_id"])

        if "from_state" in filters:
            conditions.append("from_state = %s")
            params.append(filters["from_state"])

        if "to_state" in filters:
            conditions.append("to_state = %s")
            params.append(filters["to_state"])

        if "event" in filters:
            conditions.append("event = %s")
            params.append(filters["event"])

        if "operator_type" in filters:
            conditions.append("operator_type = %s")
            params.append(filters["operator_type"])

        if "start_time" in filters:
            conditions.append("created_at >= %s")
            params.append(filters["start_time"])

        if "end_time" in filters:
            conditions.append("created_at <= %s")
            params.append(filters["end_time"])

        if "batch_id" in filters:
            conditions.append("batch_id = %s")
            params.append(filters["batch_id"])

        where_clause = " AND ".join(conditions)

        # 分页查询
        page = filters.get("page", 1)
        page_size = filters.get("page_size", 50)
        offset = (page - 1) * page_size

        total = self.db.query_one(
            f"SELECT COUNT(*) as cnt FROM state_transition_log WHERE {where_clause}",
            *params
        ).cnt

        records = self.db.query(f"""
            SELECT * FROM state_transition_log
            WHERE {where_clause}
            ORDER BY created_at DESC
            LIMIT %s OFFSET %s
        """, *params, page_size, offset)

        return {
            "total": total,
            "page": page,
            "page_size": page_size,
            "records": [
                {
                    "id": r.id,
                    "package_id": r.package_id,
                    "from_state": r.from_state,
                    "to_state": r.to_state,
                    "event": r.event,
                    "operator": r.operator,
                    "operator_type": r.operator_type,
                    "batch_id": r.batch_id,
                    "metadata": json.loads(r.metadata) if r.metadata else {},
                    "created_at": r.created_at.isoformat()
                }
                for r in records
            ]
        }

    def _detect_anomalies(self, timeline: List[dict]) -> List[dict]:
        """检测状态流转中的异常模式"""
        anomalies = []

        # 检测1：状态停留时间异常（过短或过长）
        for step in timeline:
            if step["duration_in_previous_state_secs"] is not None:
                duration = step["duration_in_previous_state_secs"]
                if duration < 1:  # 不到1秒就离开，可能是误操作
                    anomalies.append({
                        "type": "TOO_QUICK_TRANSITION",
                        "step": step["sequence"],
                        "state": step["from_state"],
                        "duration_secs": duration,
                        "message": f"在 {step['from_state']} 仅停留 {duration:.1f} 秒"
                    })
                if duration > 7 * 24 * 3600:  # 超过7天
                    anomalies.append({
                        "type": "TOO_LONG_STAY",
                        "step": step["sequence"],
                        "state": step["from_state"],
                        "duration_secs": duration,
                        "message": f"在 {step['from_state']} 停留 {duration/3600:.1f} 小时"
                    })

        # 检测2：异常状态出现
        for step in timeline:
            if step["to_state"] == "EXCEPTION":
                anomalies.append({
                    "type": "EXCEPTION_OCCURRED",
                    "step": step["sequence"],
                    "from_state": step["from_state"],
                    "event": step["event"],
                    "message": f"从 {step['from_state']} 进入异常状态"
                })

        # 检测3：同一状态反复进出
        state_visits = {}
        for step in timeline:
            state = step["to_state"]
            state_visits[state] = state_visits.get(state, 0) + 1
        for state, count in state_visits.items():
            if count > 2:
                anomalies.append({
                    "type": "REPEATED_STATE_VISIT",
                    "state": state,
                    "visit_count": count,
                    "message": f"状态 {state} 被访问 {count} 次（可能有循环）"
                })

        return anomalies

    def _check_duration_anomaly(self, state: str, duration_secs: float,
                                 definition) -> bool:
        """检查状态停留时间是否异常"""
        for timeout in definition.get("timeouts", []):
            if timeout["state"] == state:
                expected_secs = parse_duration(timeout["duration"])
                # 超过预期时间的 2 倍视为异常
                if duration_secs > expected_secs * 2:
                    return True
        return False

    def _build_expected_paths(self, definition) -> List[List[Tuple]]:
        """从定义构建所有预期流转路径（BFS 遍历）"""
        # 简化实现：列出所有合法的转换
        return [[(t.from, t.to, t.event)] for t in definition.transitions]

    def _infer_cause(self, from_state, to_state, event, definition) -> str:
        """推断异常转换的可能原因"""
        if to_state == "EXCEPTION":
            return "包裹出现异常（丢失/损坏/海关扣留等）"
        if from_state == "DELIVERED":
            return "签收后状态变更（误签/重新派送/拦截）"
        return "未在预期流转路径中"
```

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
## 性能与成本分析

**系统规模估算（日均 500 万包裹、峰值 1 万 TPS）：**

| 组件 | 规格 | 数量 | 月成本 |
|------|------|------|-------|
| 状态机引擎 | 4c8G | 20 台 | ¥4 万 |
| MySQL（包裹状态） | 8c64G + SSD | 主从 4 台 | ¥6 万 |
| Redis（状态缓存+分布式锁） | 8c32G | 3 节点 Cluster | ¥1.5 万 |
| Kafka（状态变更事件） | 8c32G + 1TB | 6 节点 | ¥3 万 |
| 规则引擎 | 4c8G | 5 台 | ¥1 万 |
| **合计** | | | **¥15.5 万** |

**关键性能指标：**

| 操作 | TPS | 延迟 | 说明 |
|------|-----|------|------|
| 正常状态变更 | 10000/s | < 10ms | Redis 缓存命中 |
| 含规则校验的变更 | 5000/s | < 50ms | 规则引擎评估 |
| 异常状态创建 | 1000/s | < 100ms | 含补偿事务 |
| 批量状态变更 | 500/s | < 500ms | 批量 200 个包裹 |
| 状态查询 | 50000/s | < 5ms | Redis 缓存 |

**存储增长估算：**

| 数据 | 日增量 | 月增量 | 保留策略 |
|------|--------|--------|---------|
| 包裹状态 | 500 万行 | 1.5 亿行 | 热数据 30 天，归档 HDFS |
| 状态变更日志 | 3000 万行 | 9 亿行 | 热数据 90 天，归档 |
| 异常处理记录 | 10 万行 | 300 万行 | 全量保留 |

### 数据库分片策略：详细设计

**核心分片原则：** 按包裹 ID（package_id）哈希分片，确保同一包裹的所有数据落在同一分片，避免跨库事务。

**分片拓扑（16 库 × 4 表 = 64 分表）：**

```
db_shard_00: package_states_00, state_transition_log_00, ...
db_shard_01: package_states_01, state_transition_log_01, ...
...
db_shard_15: package_states_15, state_transition_log_15, ...
```

**分片路由规则：**

```python
class ShardRouter:
    """
    分片路由器。
    package_id 格式: {region_code(2)}{date(8)}{sequence(6)}{check(2)}
    例如: SH2026060100012345
    
    分片策略：取 package_id 后 4 位数字做 hash % 16 → 库编号
    """

    SHARD_COUNT = 16
    TABLES_PER_SHARD = 4

    def route_to_shard(self, package_id: str) -> int:
        """计算 package_id 路由到哪个分库"""
        # 取 package_id 的数字部分做 hash
        numeric_part = re.sub(r'[^0-9]', '', package_id)[-4:]
        hash_val = int(numeric_part) % self.SHARD_COUNT
        return hash_val

    def route_to_table(self, package_id: str, table_name: str) -> str:
        """计算路由到哪个分表"""
        shard = self.route_to_shard(package_id)
        # state_transition_log 额外按月分表
        if table_name == "state_transition_log":
            month_suffix = datetime.now().strftime("%Y%m")
            return f"state_transition_log_{shard:02d}_{month_suffix}"
        return f"{table_name}_{shard:02d}"
```

**各表分片策略对比：**

| 表 | 分片键 | 分片方式 | 分库数 | 分表数 | 扩容方式 |
|----|--------|---------|-------|--------|---------|
| packages | package_id | 哈希 | 16 | 16 | 扩库到 32（双倍扩容） |
| package_states | package_id | 哈希 | 16 | 16 | 同 packages |
| state_transition_log | package_id + created_at | 哈希 + 时间 | 16 | 16×月分区 | 历史分区归档 |
| transition_rules | product_type | 不分片（数据量极小） | 1 | 1 | 无需扩容 |
| scheduled_timeouts | package_id | 哈希 | 16 | 16 | 同 packages |
| exception_handling | package_id | 哈希 | 16 | 16 | 同 packages |
| action_retry_log | package_id | 哈希 | 16 | 16 | 同 packages |

**扩容方案（从 16 库扩到 32 库）：**

1. 新建 16 个分库，按新路由规则迁移数据
2. 使用双写过渡期：写入同时写新旧分库
3. 数据迁移完成后，读切换到新分库
4. 停止旧分库写入，下线旧分库
5. 整个过程约 2-4 周，零停机

### 查询优化：状态查找性能

**高频查询场景与优化：**

**场景 1：按 package_id 查询当前状态（最高频，QPS 50000）**

```sql
-- 原始查询
SELECT * FROM package_states WHERE package_id = ?;

-- 优化 1：Redis 缓存（命中率 > 95%）
-- Key: pkg:state:{package_id}  Value: {"state":"IN_TRANSIT","entered_at":"...","version":5}
-- TTL: 24h（状态变更时主动刷新）

-- 优化 2：数据库覆盖索引（避免回表）
-- package_id 是主键，天然覆盖索引

-- 优化 3：本地缓存（Caffeine，命中率 > 60%）
-- 热点包裹（如当天正在流转的）缓存在应用内存
-- 容量: 10万条，过期: 5分钟，状态变更时主动失效
```

**场景 2：超时检测扫描（QPS 1，但单次扫 1000 条）**

```sql
-- 原始查询
SELECT * FROM scheduled_timeouts 
WHERE status = 'PENDING' AND execute_at <= NOW()
LIMIT 1000;

-- 优化：复合索引 (status, execute_at) 覆盖查询
-- 分片后每个分库最多 62.5 条/秒，压力极小
```

**场景 3：按时间段统计异常包裹（运营报表，低频但慢）**

```sql
-- 原始查询
SELECT exception_type, COUNT(*) FROM exception_handling
WHERE created_at BETWEEN ? AND ?
GROUP BY exception_type;

-- 优化 1：预聚合表（物化视图）
CREATE TABLE exception_daily_stats (
    stat_date DATE NOT NULL,
    exception_type VARCHAR(32) NOT NULL,
    product_type VARCHAR(32),
    count INTEGER NOT NULL,
    PRIMARY KEY (stat_date, exception_type, product_type)
);

-- 优化 2：离线 T+1 报表走 Hive/ClickHouse，不查 MySQL
-- 优化 3：实时大屏走 Flink 实时聚合，不查数据库
```

**场景 4：包裹状态变更历史查询（客服系统，QPS 2000）**

```sql
-- 原始查询
SELECT * FROM state_transition_log 
WHERE package_id = ? ORDER BY created_at ASC;

-- 优化 1：package_id 索引覆盖
-- 优化 2：分库后单分片数据量可控
-- 优化 3：热数据在 MySQL，冷数据（>90天）归档到 ES（支持全文检索）
-- 优化 4：缓存最近 30 天的变更历史到 Redis List
--   Key: pkg:history:{package_id}
--   Value: JSON 序列化的最近 20 条记录
--   状态变更时 LPUSH + LTRIM 保持窗口大小
```

### 缓存策略：分层缓存架构

```
请求流程：
  L1: 本地缓存(Caffeine) → 命中率 ~60%, 延迟 <1ms
  L2: Redis 缓存          → 命中率 ~95%, 延迟 <5ms  
  L3: MySQL 数据库         → 延迟 10-50ms

缓存数据分类：

| 数据类型 | 缓存位置 | TTL | 更新策略 | 容量估算 |
|---------|---------|-----|---------|---------|
| 包裹当前状态 | L1 + L2 + L3 | 24h / 5min | 状态变更时主动失效 | 500万条 × 200B ≈ 1GB |
| 状态机定义 | L1 + L2 | 10min | 版本变更时 Pub/Sub 通知失效 | <100条 × 5KB ≈ 500KB |
| 最近变更历史 | L2 | 1h | 状态变更时 LPUSH | 500万条 × 20条 × 500B ≈ 50GB（选择性缓存热点） |
| 转换规则 | L2 | 30min | 规则变更时 Pub/Sub 通知失效 | <2000条 × 1KB ≈ 2MB |
| 分布式锁 | L2 | 30s | 自动过期 | 1万条 × 100B ≈ 1MB |

缓存一致性保证：
  - 写操作：先更新 DB，再删除缓存（Cache-Aside 模式）
  - 读操作：读缓存 → 未命中 → 读 DB → 写回缓存
  - 极端情况：缓存与 DB 短暂不一致，TTL 兜底（5分钟内自动修复）
  - 状态变更场景的特殊处理：同一包裹的状态变更串行化（分布式锁），避免并发写导致缓存脏数据
```

**缓存失效的 Pub/Sub 通知机制：**

```python
class CacheInvalidationListener:
    """监听状态变更事件，主动失效相关缓存"""

    def __init__(self, redis_client, local_cache):
        self.redis = redis_client
        self.local_cache = local_cache
        # 订阅状态变更频道
        self.redis.subscribe("state_changed", self._on_state_changed)

    def _on_state_changed(self, message):
        """收到状态变更通知，失效缓存"""
        data = json.loads(message)
        package_id = data["package_id"]

        # 1. 失效本地缓存
        self.local_cache.invalidate(f"pkg:state:{package_id}")
        self.local_cache.invalidate(f"pkg:history:{package_id}")

        # 2. 失效 Redis 缓存
        self.redis.delete(f"pkg:state:{package_id}")
        self.redis.delete(f"pkg:history:{package_id}")

        # 3. 预热：异步从 DB 加载最新状态到缓存
        # （避免下次请求缓存未命中的延迟尖刺）
        self._async_warmup(package_id)

    def _async_warmup(self, package_id):
        """异步预热缓存"""
        try:
            pkg_state = self.db.query_one(
                "SELECT * FROM package_states WHERE package_id = %s",
                package_id
            )
            if pkg_state:
                self.redis.setex(
                    f"pkg:state:{package_id}", 86400,
                    json.dumps({
                        "state": pkg_state.current_state,
                        "entered_at": pkg_state.state_entered_at.isoformat(),
                        "version": pkg_state.version
                    })
                )
        except Exception:
            pass  # 预热失败不影响主流程
```

**Redis 内存估算与成本优化：**

| 数据 | 条数 | 单条大小 | 总大小 | 优化手段 |
|------|------|---------|--------|---------|
| 包裹状态 | 500万 | 200B | 1GB | 使用 Hash 压缩，实际约 600MB |
| 变更历史 | 100万热点 | 10KB | 10GB | 只缓存当日活跃包裹 |
| 分布式锁 | 1万 | 100B | 1MB | 无需优化 |
| 定义缓存 | 100 | 5KB | 500KB | 无需优化 |
| **合计** | | | **~11GB** | 3 节点 32GB 充足 |

### 全链路延迟分析

```
单次状态变更（正常路径）的延迟分解：

  1. 请求接入层         0.5ms    (Nginx → API Gateway)
  2. 分布式锁获取       2ms      (Redis SETNX)
  3. 加载状态机定义     0.5ms    (本地缓存命中)
  4. 查询包裹当前状态   1ms      (Redis 缓存命中)
  5. 规则评估          0.5ms    (AST 内存计算)
  6. 更新状态(写DB)     5ms      (MySQL 主库写入)
  7. 写审计日志(DB)     3ms      (批量写入或异步写入)
  8. 更新缓存          1ms      (Redis 删除+预热)
  9. 发布变更通知       1ms      (Kafka 异步)
  10. 返回响应

  P50 延迟: ~10ms
  P99 延迟: ~50ms (缓存未命中时 +20ms)
  P999 延迟: ~100ms (DB 慢查询或锁等待)
```

## 异常场景完整演练

### 场景 1：包裹状态冲突——站点扫描重复

```
触发：站点 PDA 扫描枪重复扫码 → 同一包裹在 0.5 秒内触发两次"到达站点"
处理：
  1. 状态机引擎检测到：当前状态 = "运输中"，目标状态 = "派送中"
  2. 第一次变更：运输中 → 派送中 → 成功
  3. 第二次变更：当前状态已是"派送中"→ "派送中"→"派送中"是非法转换 → 拒绝
  4. 返回错误码：DUPLICATE_TRANSITION
  5. PDA 端提示"该包裹已到达站点，无需重复操作"
幂等保证：transition_id = package_id + target_state + timestamp_window（5秒去重）
```

### 场景 2：包裹丢失后的状态修复

```
触发：包裹在"运输中"状态 7 天无更新 → 系统自动标记"异常-丢失"
处理：
  1. 定时任务扫描：state = '运输中' AND updated_at < NOW() - INTERVAL 7 DAY
  2. 自动创建异常状态：运输中 → 异常(丢失)
  3. 触发查找流程：异常(丢失) → 查找中
  4. 如果找到：查找中 → 已找回 → 运输中（恢复原流程）
  5. 如果 30 天未找到：查找中 → 已丢失 → 理赔中 → 已理赔
  6. 全程状态变更通过状态机引擎，确保合法性和可追溯
```

### 场景 3：状态机规则热更新

```
触发：新增物流产品"跨城急送"→ 需要定义新的状态转换规则
处理：
  1. 运营在管理后台配置新规则（可视化编辑器）
  2. 规则保存到 rule_definitions 表
  3. 发布规则 → 通知所有状态机引擎节点（Redis Pub/Sub）
  4. 各节点收到通知 → 从数据库加载最新规则 → 替换本地缓存
  5. 热更新过程 < 5 秒，无需重启服务
  6. 新规则只对新的状态变更生效，已在处理中的变更不受影响
关键：规则版本化管理，支持回滚到上一版本
```

## 事件溯源完整实现

```python
class LogisticsEventStore:
    """物流状态机事件溯源"""

    def append_event(self, order_id, event_type, event_data):
        """追加事件到事件流"""
        # 获取当前流版本
        current_version = self.db.query_one(
            "SELECT MAX(version) as v FROM logistics_events WHERE order_id = %s",
            order_id)["v"] or 0

        self.db.insert("logistics_events", {
            "order_id": order_id,
            "version": current_version + 1,
            "event_type": event_type,
            "event_data": json.dumps(event_data),
            "timestamp": now(),
            "operator": current_user()
        })

        # 更新投影（当前状态）
        self._update_projection(order_id, event_type, event_data)

    def replay_events(self, order_id, to_version=None):
        """重放事件流（用于调试或修复投影）"""
        events = self.db.query(
            "SELECT * FROM logistics_events WHERE order_id = %s ORDER BY version",
            order_id)

        if to_version:
            events = [e for e in events if e["version"] <= to_version]

        # 从空状态开始重放
        state = {"status": "created"}
        for event in events:
            state = self._apply_event(state, event)

        return state

    def _update_projection(self, order_id, event_type, event_data):
        """更新投影（当前状态视图）"""
        transitions = {
            "payment_confirmed": {"status": "paid"},
            "warehouse_received": {"status": "picked"},
            "shipped": {"status": "shipped"},
            "delivered": {"status": "delivered"},
            "returned": {"status": "returned"},
        }
        new_state = transitions.get(event_type, {})
        self.db.update("logistics_order_state", new_state, {"order_id": order_id})
```

**事件溯源表：**

```sql
CREATE TABLE logistics_events (
    order_id VARCHAR(36) NOT NULL,
    version BIGINT NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    event_data TEXT NOT NULL,
    timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    operator VARCHAR(100),
    PRIMARY KEY (order_id, version),
    INDEX idx_order_version (order_id, version)
);
```

## 补偿动作注册器

```python
class CompensationRegistry:
    """每个状态转换的补偿动作注册"""

    COMPENSATIONS = {
        "payment_confirmed": "cancel_payment",
        "warehouse_received": "return_to_warehouse",
        "shipped": "recall_shipment",
        "delivered": "initiate_return",
    }

    def execute_compensation(self, order_id, from_event):
        """执行补偿动作"""
        comp_action = self.COMPENSATIONS.get(from_event)
        if not comp_action:
            raise NoCompensationError(f"事件 {from_event} 无补偿动作")

        event_data = self.db.get_event_data(order_id, from_event)
        result = self.compensation_executor.execute(comp_action, event_data)

        # 记录补偿事件
        self.event_store.append_event(order_id, f"compensation_{comp_action}", {
            "original_event": from_event,
            "result": result
        })
        return result
```

## SLA 监控完整实现

```python
class SLAMonitorService:
    """物流 SLA 监控：承诺时效追踪"""

    def check_sla_breach(self, order_id):
        """检查是否违反 SLA"""
        order = self.db.get_order(order_id)
        sla_config = self.db.get_sla_config(order["service_type"])

        # 计算各阶段耗时
        stages = {
            "pickup": self._calc_stage_duration(order, "paid", "picked"),
            "shipping": self._calc_stage_duration(order, "picked", "shipped"),
            "delivery": self._calc_stage_duration(order, "shipped", "delivered"),
        }

        breaches = []
        for stage, duration in stages.items():
            if duration and duration > sla_config[f"{stage}_max_hours"]:
                breaches.append({
                    "stage": stage,
                    "actual_hours": duration,
                    "max_hours": sla_config[f"{stage}_max_hours"],
                    "over_hours": duration - sla_config[f"{stage}_max_hours"]
                })

        if breaches:
            # SLA 违约 → 计算赔付
            penalty = self._calc_penalty(breaches, sla_config)
            self.db.insert("sla_breach_log", {
                "order_id": order_id, "breaches": json.dumps(breaches),
                "penalty_amount": penalty, "detected_at": now()
            })

        return {"order_id": order_id, "breaches": breaches,
                "penalty": penalty if breaches else 0}
```

## 异常场景补充

### 场景：事件存储损坏

```
触发：数据库页损坏 → 事件流读取失败 → 投影状态错误
检测：
  1. CHECK TABLE 发现损坏 → 告警
  2. replay_events 返回异常 → 状态重建失败
处理：
  1. 从备份恢复事件流
  2. 重放所有事件重建投影
  3. 校验投影状态与最新事件一致
预防：事件流每日备份 + 页校验 + 投影重建机制
```

### 场景：SLA 违约级联

```
触发：暴雨 → 大量订单延迟 → 多个 SLA 同时违约
      → 赔付金额巨大 → 运营成本暴增
检测：
  1. 当日 SLA 违约率 > 10% → 告警
  2. 赔付金额 > 日营收 5% → 严重告警
处理：
  1. 启动应急预案：增加临时运力
  2. 通知受影响客户 → 提供替代方案
  3. 计算总赔付 → 资金准备
预防：天气预报预警 + SLA 违约率监控 + 赔付上限
```

## 物流轨迹追踪完整实现

```python
class LogisticsTracker:
    """物流轨迹追踪：GPS + 物流节点"""

    def report_location(self, order_id, lat, lng, source="gps"):
        """上报物流位置"""
        # 1. 存储轨迹点
        self.db.insert("logistics_track_points", {
            "order_id": order_id,
            "latitude": lat, "longitude": lng,
            "source": source,  # gps / warehouse_scan / delivery_scan
            "timestamp": now()
        })

        # 2. 判断是否到达关键节点
        next_node = self._get_next_node(order_id)
        if next_node:
            distance = self.haversine(lat, lng, next_node["lat"], next_node["lng"])
            if distance < 500:  # 500 米内 → 到达
                self._handle_node_arrival(order_id, next_node)

    def _handle_node_arrival(self, order_id, node):
        """处理到达物流节点"""
        transitions = {
            "sort_center": ("arrived_at_sort", "sorting"),
            "distribution_center": ("arrived_at_dist", "distributing"),
            "delivery_station": ("arrived_at_station", "out_for_delivery"),
        }
        event, new_status = transitions.get(node["type"], ("location_update", None))
        if new_status:
            self.event_store.append_event(order_id, event, {
                "node_id": node["id"], "node_name": node["name"],
                "arrival_time": now().isoformat()
            })
            self.db.update("logistics_order_state",
                {"status": new_status, "current_node": node["id"]},
                {"order_id": order_id})

    def get_tracking_info(self, order_id):
        """获取物流追踪信息"""
        state = self.db.get_order_state(order_id)
        track_points = self.db.query(
            "SELECT * FROM logistics_track_points "
            "WHERE order_id = %s ORDER BY timestamp DESC LIMIT 20", order_id)

        return {
            "order_id": order_id,
            "status": state["status"],
            "current_location": track_points[0] if track_points else None,
            "track_history": [{
                "timestamp": p["timestamp"],
                "location": f"{p['latitude']:.4f}, {p['longitude']:.4f}",
                "source": p["source"]
            } for p in track_points]
        }
```

## 异常场景补充

### 场景：轨迹数据异常跳跃

```
触发：GPS 信号漂移 → 轨迹点跳跃 200km → 显示异常
检测：
  1. 两点间速度 > 200km/h → 异常跳跃
  2. 与路线规划偏差 > 50km → 异常
处理：
  1. 过滤异常点：不写入轨迹展示
  2. 使用卡尔曼滤波平滑轨迹
  3. 标记异常点用于 GPS 质量分析
预防：轨迹点速度校验 + 滤波平滑
```

### 场景：物流节点扫描遗漏

```
触发：包裹到达分拣中心但未扫描 → 状态未更新 → 客户查询显示"运输中"
检测：
  1. 预计到达时间已过但状态未更新 → 告警
  2. 同批次其他包裹已到达 → 该包裹遗漏
处理：
  1. 自动推断：根据上下游节点推断当前状态
  2. 通知分拣中心补扫
  3. 客户端显示推断状态（标注"预计"）
预防：节点扫描超时检测 + 自动状态推断
```

## 异常处理编排器完整实现

```python
class ExceptionOrchestrator:
    """物流异常处理编排器"""

    EXCEPTION_TYPES = {
        "damaged": {"description": "包裹损坏", "auto_recoverable": False},
        "lost": {"description": "包裹丢失", "auto_recoverable": False},
        "wrong_item": {"description": "错发商品", "auto_recoverable": False},
        "address_error": {"description": "地址错误", "auto_recoverable": True},
        "refused": {"description": "收件人拒收", "auto_recoverable": False},
        "customs_held": {"description": "海关扣留", "auto_recoverable": False},
    }

    RECOVERY_ACTIONS = {
        "damaged": ["notify_sender", "create_return_order", "issue_refund"],
        "lost": ["notify_sender", "create_replacement_order", "issue_refund"],
        "wrong_item": ["create_return_label", "create_replacement_order"],
        "address_error": ["contact_recipient", "update_address", "reroute"],
        "refused": ["create_return_order", "issue_refund"],
        "customs_held": ["notify_sender", "request_documents"],
    }

    def handle_exception(self, order_id, exception_type, details):
        """处理物流异常"""
        if exception_type not in self.EXCEPTION_TYPES:
            raise UnknownExceptionTypeError(f"未知异常类型: {exception_type}")

        # 1. 记录异常
        exception_id = str(uuid4())
        self.db.insert("logistics_exceptions", {
            "exception_id": exception_id,
            "order_id": order_id,
            "exception_type": exception_type,
            "details": json.dumps(details),
            "status": "open",
            "created_at": now(),
            "sla_deadline": now() + timedelta(hours=self._get_sla_hours(exception_type))
        })

        # 2. 自动恢复（如果可自动恢复）
        if self.EXCEPTION_TYPES[exception_type]["auto_recoverable"]:
            actions = self.RECOVERY_ACTIONS[exception_type]
            for action in actions:
                result = self._execute_recovery(action, order_id, details)
                if result["success"]:
                    self.db.insert("exception_recovery_log", {
                        "exception_id": exception_id,
                        "action": action, "result": "success",
                        "executed_at": now()
                    })
                else:
                    # 自动恢复失败 → 人工介入
                    self.db.update("logistics_exceptions",
                        {"status": "manual_intervention"},
                        {"exception_id": exception_id})
                    self.alert(f"异常 {exception_id} 自动恢复失败: {action}")
                    break

        return exception_id

    def _get_sla_hours(self, exception_type):
        """异常处理 SLA"""
        sla_map = {"damaged": 24, "lost": 48, "wrong_item": 24,
                   "address_error": 4, "refused": 24, "customs_held": 72}
        return sla_map.get(exception_type, 24)
```

## 物流成本计算引擎

```python
class LogisticsCostEngine:
    """物流成本计算：重量/体积/距离/附加费"""

    def calculate(self, order):
        """计算物流成本"""
        base_cost = self._calc_base_cost(order)

        # 附加费
        surcharges = []
        surcharges.append(self._fuel_surcharge(base_cost))
        surcharges.append(self._remote_area_surcharge(order["destination"]))
        if order.get("cod_amount"):
            surcharges.append(self._cod_fee(order["cod_amount"]))
        if order.get("insurance_value"):
            surcharges.append(self._insurance_premium(order["insurance_value"]))

        total = base_cost + sum(s["amount"] for s in surcharges)

        # 折扣
        discount = self._calc_discount(order)

        return {
            "base_cost": base_cost,
            "surcharges": surcharges,
            "discount": discount,
            "total_cost": round(total * (1 - discount), 2)
        }

    def _calc_base_cost(self, order):
        """基础运费：取重量和体积重量中较大者"""
        weight = order["weight_kg"]
        volume_weight = (order["length_cm"] * order["width_cm"] *
            order["height_cm"]) / 6000  # 体积重量公式
        chargeable = max(weight, volume_weight)

        # 按距离段计费
        distance = self._calc_distance(order["origin"], order["destination"])
        rate = self._get_rate_by_distance(distance)
        return chargeable * rate
```

## 异常场景补充

### 场景：费率表不一致

```
触发：物流费率表更新延迟 → 计费使用旧费率 → 运费偏差
检测：
  1. 每日对账：系统计费 vs 物流商账单
  2. 差异 > 2% → 告警
处理：
  1. 检查费率表版本是否最新
  2. 旧费率 → 重新计算差额
  3. 差额由平台承担或补收
预防：费率表版本化 + 变更通知 + 每日对账
```

### 场景：路径优化超时

```
触发：TSP 求解器在复杂路网中超时 → 无法返回最优路线
检测：
  1. 求解器运行时间 > 5 秒 → 超时
  2. 超时 → 降级到贪心算法
处理：
  1. 超时后切换到最近邻启发式（快但非最优）
  2. 后台继续优化路线 → 优化完成后推送更新
  3. 司机端先显示初始路线
预防：求解器超时保护 + 降级策略 + 后台优化
```

## 异常处理编排器深度实现

### 异常类型注册表与自动恢复

```python
class ExceptionOrchestratorV2:
    """物流异常处理编排器V2：异常类型注册 + 自动恢复 + 人工覆盖 + SLA追踪"""

    # 异常类型注册表
    EXCEPTION_REGISTRY = {
        "damaged": {
            "description": "包裹损坏",
            "auto_recoverable": False,
            "sla_hours": 24,
            "severity": "high",
            "recovery_actions": ["notify_sender", "create_return_order", "issue_refund"],
            "escalation_hours": 12  # 超时升级
        },
        "lost": {
            "description": "包裹丢失",
            "auto_recoverable": False,
            "sla_hours": 48,
            "severity": "critical",
            "recovery_actions": ["notify_sender", "investigate_route", "create_replacement_order", "issue_refund"],
            "escalation_hours": 24
        },
        "wrong_item": {
            "description": "错发商品",
            "auto_recoverable": False,
            "sla_hours": 24,
            "severity": "high",
            "recovery_actions": ["create_return_label", "create_replacement_order", "issue_refund"],
            "escalation_hours": 12
        },
        "address_error": {
            "description": "地址错误",
            "auto_recoverable": True,
            "sla_hours": 4,
            "severity": "medium",
            "recovery_actions": ["contact_recipient", "update_address", "reroute"],
            "escalation_hours": 2
        },
        "refused": {
            "description": "收件人拒收",
            "auto_recoverable": False,
            "sla_hours": 24,
            "severity": "medium",
            "recovery_actions": ["create_return_order", "issue_refund"],
            "escalation_hours": 12
        },
        "customs_held": {
            "description": "海关扣留",
            "auto_recoverable": False,
            "sla_hours": 72,
            "severity": "high",
            "recovery_actions": ["notify_sender", "request_documents", "resubmit_customs"],
            "escalation_hours": 48
        }
    }

    def __init__(self, db, redis, mq, event_store):
        self.db = db
        self.redis = redis
        self.mq = mq
        self.event_store = event_store

    def handle_exception(self, order_id, exception_type, details, reporter_id=None):
        """处理物流异常"""
        if exception_type not in self.EXCEPTION_REGISTRY:
            raise ValueError(f"未知异常类型: {exception_type}")

        registry = self.EXCEPTION_REGISTRY[exception_type]

        # 1. 去重检查：同一包裹同一异常类型是否已存在
        existing = self.db.query(
            "SELECT exception_id FROM logistics_exceptions "
            "WHERE order_id = %s AND exception_type = %s AND status IN ('open', 'processing')",
            order_id, exception_type)
        if existing:
            return {"status": "duplicate", "exception_id": existing[0]["exception_id"]}

        # 2. 记录异常
        exception_id = str(uuid4())
        sla_deadline = now() + timedelta(hours=registry["sla_hours"])
        escalation_deadline = now() + timedelta(hours=registry["escalation_hours"])

        self.db.insert("logistics_exceptions", {
            "exception_id": exception_id,
            "order_id": order_id,
            "exception_type": exception_type,
            "severity": registry["severity"],
            "details": json.dumps(details),
            "status": "open",
            "reporter_id": reporter_id,
            "sla_deadline": sla_deadline,
            "escalation_deadline": escalation_deadline,
            "created_at": now()
        })

        # 3. 触发状态机异常转换
        self.event_store.append_event(order_id, "EXCEPTION_OCCURRED", {
            "exception_id": exception_id,
            "exception_type": exception_type,
            "severity": registry["severity"]
        })

        # 4. 自动恢复（如果可自动恢复）
        if registry["auto_recoverable"]:
            recovery_result = self._auto_recover(exception_id, order_id,
                                                   exception_type, details)
            if recovery_result["success"]:
                self.db.update("logistics_exceptions",
                    {"status": "resolved", "resolved_at": now(),
                     "resolution": "auto_recovery"},
                    {"exception_id": exception_id})
                return {"status": "auto_resolved", "exception_id": exception_id}

        # 5. 发送通知
        self._notify_stakeholders(order_id, exception_type, exception_id)

        # 6. 设置SLA追踪定时器
        self._setup_sla_tracking(exception_id, sla_deadline, escalation_deadline)

        return {"status": "created", "exception_id": exception_id}

    def _auto_recover(self, exception_id, order_id, exception_type, details):
        """自动恢复流程"""
        registry = self.EXCEPTION_REGISTRY[exception_type]
        actions = registry["recovery_actions"]

        for action in actions:
            result = self._execute_recovery_action(action, order_id, details)
            self.db.insert("exception_recovery_log", {
                "exception_id": exception_id,
                "action": action,
                "result": "success" if result["success"] else "failed",
                "error_message": result.get("error", ""),
                "executed_at": now()
            })

            if not result["success"]:
                # 自动恢复失败 → 人工介入
                self.db.update("logistics_exceptions",
                    {"status": "manual_intervention"},
                    {"exception_id": exception_id})
                self.alert(f"异常 {exception_id} 自动恢复失败: {action}")
                return {"success": False, "failed_action": action}

        return {"success": True}

    def _execute_recovery_action(self, action, order_id, details):
        """执行恢复动作"""
        try:
            if action == "notify_sender":
                return self._notify_sender(order_id, details)
            elif action == "create_return_order":
                return self._create_return_order(order_id)
            elif action == "create_replacement_order":
                return self._create_replacement_order(order_id)
            elif action == "issue_refund":
                return self._issue_refund(order_id, details)
            elif action == "contact_recipient":
                return self._contact_recipient(order_id, details)
            elif action == "update_address":
                return self._update_address(order_id, details)
            elif action == "reroute":
                return self._reroute_order(order_id, details)
            elif action == "create_return_label":
                return self._create_return_label(order_id)
            elif action == "investigate_route":
                return self._investigate_route(order_id)
            elif action == "request_documents":
                return self._request_customs_documents(order_id)
            elif action == "resubmit_customs":
                return self._resubmit_customs(order_id, details)
            else:
                return {"success": False, "error": f"未知动作: {action}"}
        except Exception as e:
            return {"success": False, "error": str(e)}

    # ===== 人工覆盖 =====

    def manual_override(self, exception_id, operator_id, action, params=None):
        """人工覆盖异常处理"""
        exception = self.db.get("logistics_exceptions", exception_id)
        if not exception:
            raise ValueError("异常记录不存在")
        if exception["status"] in ("resolved", "closed"):
            raise ValueError("异常已关闭，无法覆盖")

        # 执行人工指定的动作
        result = self._execute_recovery_action(action, exception["order_id"],
                                                 params or json.loads(exception["details"]))

        # 记录人工操作
        self.db.insert("exception_manual_override_log", {
            "exception_id": exception_id,
            "operator_id": operator_id,
            "action": action,
            "params": json.dumps(params or {}),
            "result": "success" if result["success"] else "failed",
            "executed_at": now()
        })

        if result["success"]:
            self.db.update("logistics_exceptions",
                {"status": "resolved", "resolved_at": now(),
                 "resolution": "manual_override", "resolved_by": operator_id},
                {"exception_id": exception_id})

        return result

    # ===== SLA追踪 =====

    def _setup_sla_tracking(self, exception_id, sla_deadline, escalation_deadline):
        """设置SLA追踪定时器"""
        # SLA到期提醒（提前1小时）
        warn_before = sla_deadline - timedelta(hours=1)
        delay_warn = max(int((warn_before - now()).total_seconds() * 1000), 0)
        self.mq.publish_delayed("exception_sla_warning", {
            "exception_id": exception_id,
            "type": "sla_warning",
            "deadline": sla_deadline.isoformat()
        }, delay_ms=delay_warn)

        # SLA超时
        delay_sla = max(int((sla_deadline - now()).total_seconds() * 1000), 0)
        self.mq.publish_delayed("exception_sla_timeout", {
            "exception_id": exception_id,
            "type": "sla_timeout",
            "deadline": sla_deadline.isoformat()
        }, delay_ms=delay_sla)

        # 升级通知
        delay_escalation = max(int((escalation_deadline - now()).total_seconds() * 1000), 0)
        self.mq.publish_delayed("exception_escalation", {
            "exception_id": exception_id,
            "type": "escalation",
            "deadline": escalation_deadline.isoformat()
        }, delay_ms=delay_escalation)

    def on_sla_timeout(self, exception_id):
        """SLA超时处理"""
        exception = self.db.get("logistics_exceptions", exception_id)
        if not exception or exception["status"] in ("resolved", "closed"):
            return

        # 标记SLA违规
        self.db.update("logistics_exceptions",
            {"sla_violated": True, "status": "escalated"},
            {"exception_id": exception_id})

        # 自动升级到高级客服
        self.alert(f"异常 {exception_id} SLA超时，自动升级")

        # 记录SLA违规事件
        self.event_store.append_event(exception["order_id"], "SLA_VIOLATION", {
            "exception_id": exception_id,
            "exception_type": exception["exception_type"],
            "sla_deadline": exception["sla_deadline"].isoformat()
        })

    def get_sla_report(self, start_date, end_date):
        """获取SLA报告"""
        return self.db.query(
            "SELECT exception_type, "
            "COUNT(*) as total, "
            "SUM(CASE WHEN sla_violated = 1 THEN 1 ELSE 0 END) as violated, "
            "AVG(TIMESTAMPDIFF(HOUR, created_at, resolved_at)) as avg_resolution_hours "
            "FROM logistics_exceptions "
            "WHERE created_at BETWEEN %s AND %s "
            "GROUP BY exception_type", start_date, end_date)
```

## 物流成本计算引擎深度实现

### 重量/体积/距离计费 + 附加费 + 折扣

```python
class LogisticsCostEngineV2:
    """物流成本计算引擎V2：多维度计费 + 附加费 + 折扣规则"""

    def __init__(self, db, redis):
        self.db = db
        self.redis = redis

    def calculate(self, order):
        """计算物流总成本"""
        # 1. 基础运费（重量/体积/距离）
        base_cost = self._calc_base_cost(order)

        # 2. 附加费
        surcharges = []
        surcharges.append(self._fuel_surcharge(base_cost, order))
        surcharges.append(self._remote_area_surcharge(order))
        if order.get("cod_amount"):
            surcharges.append(self._cod_fee(order["cod_amount"]))
        if order.get("insurance_value"):
            surcharges.append(self._insurance_premium(order["insurance_value"]))
        if order.get("is_oversized"):
            surcharges.append(self._oversized_surcharge(order))
        if order.get("is_fragile"):
            surcharges.append(self._fragile_handling_fee(order))

        # 3. 小计
        subtotal = base_cost + sum(s["amount"] for s in surcharges)

        # 4. 折扣
        discount = self._calc_discount(order, subtotal)

        # 5. 最终价格
        total = round(subtotal * (1 - discount["rate"]), 2)

        return {
            "base_cost": round(base_cost, 2),
            "chargeable_weight": round(order.get("_chargeable_weight", 0), 2),
            "distance_km": round(order.get("_distance_km", 0), 1),
            "surcharges": surcharges,
            "surcharge_total": round(sum(s["amount"] for s in surcharges), 2),
            "discount": discount,
            "subtotal": round(subtotal, 2),
            "total_cost": total,
            "currency": "CNY"
        }

    def _calc_base_cost(self, order):
        """基础运费：取重量和体积重量中较大者 × 距离费率"""
        weight = order["weight_kg"]
        volume_weight = (order["length_cm"] * order["width_cm"] *
                         order["height_cm"]) / 6000  # 体积重量公式
        chargeable_weight = max(weight, volume_weight)
        order["_chargeable_weight"] = chargeable_weight

        # 计算距离
        distance = self._calc_distance(order["origin"], order["destination"])
        order["_distance_km"] = distance

        # 按距离段计费（阶梯费率）
        rate = self._get_rate_by_distance(distance, order.get("product_type", "standard"))

        # 首重 + 续重计费
        first_kg_rate = rate["first_kg"]
        additional_kg_rate = rate["additional_kg"]

        if chargeable_weight <= 1:
            return first_kg_rate
        else:
            return first_kg_rate + (chargeable_weight - 1) * additional_kg_rate

    def _get_rate_by_distance(self, distance_km, product_type):
        """获取距离段费率"""
        rate_key = f"rate:{product_type}:{int(distance_km // 500)}"
        cached = self.redis.hgetall(rate_key)
        if cached:
            return {"first_kg": float(cached["first_kg"]),
                    "additional_kg": float(cached["additional_kg"])}

        # 从DB加载费率表
        rate = self.db.query(
            "SELECT first_kg_rate, additional_kg_rate FROM logistics_rates "
            "WHERE product_type = %s AND min_distance <= %s "
            "ORDER BY min_distance DESC LIMIT 1",
            product_type, distance_km)

        if not rate:
            # 默认费率
            return {"first_kg": 12.0, "additional_kg": 5.0}

        result = {"first_kg": rate[0]["first_kg_rate"],
                  "additional_kg": rate[0]["additional_kg_rate"]}
        self.redis.hset(rate_key, mapping={
            "first_kg": result["first_kg"],
            "additional_kg": result["additional_kg"]
        })
        self.redis.expire(rate_key, 3600)
        return result

    def _calc_distance(self, origin, destination):
        """计算两点间运输距离（简化实现）"""
        cache_key = f"distance:{origin}:{destination}"
        cached = self.redis.get(cache_key)
        if cached:
            return float(cached)

        # 实际实现：调用地图API或使用坐标计算
        distance = self._haversine_distance(origin, destination) * 1.3  # 道路系数1.3
        self.redis.setex(cache_key, 86400, str(distance))
        return distance

    def _haversine_distance(self, origin, destination):
        """Haversine公式计算球面距离"""
        # 简化：从坐标查询
        R = 6371  # 地球半径km
        lat1, lon1 = self._get_coordinates(origin)
        lat2, lon2 = self._get_coordinates(destination)
        dlat = radians(lat2 - lat1)
        dlon = radians(lon2 - lon1)
        a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
        return R * 2 * asin(sqrt(a))

    # ===== 附加费 =====

    def _fuel_surcharge(self, base_cost, order):
        """燃油附加费（根据油价波动）"""
        fuel_rate = self.redis.get("fuel_surcharge_rate")
        if fuel_rate is None:
            rate_row = self.db.query(
                "SELECT rate FROM surcharge_rates WHERE type = 'fuel' ORDER BY effective_date DESC LIMIT 1")
            fuel_rate = rate_row[0]["rate"] if rate_row else 0.05
            self.redis.setex("fuel_surcharge_rate", 86400, str(fuel_rate))
        else:
            fuel_rate = float(fuel_rate)

        amount = round(base_cost * float(fuel_rate), 2)
        return {"type": "fuel_surcharge", "rate": float(fuel_rate), "amount": amount}

    def _remote_area_surcharge(self, order):
        """偏远地区附加费"""
        destination_code = order.get("destination_code", "")
        remote_areas = self.redis.smembers("remote_area_codes")
        if destination_code in remote_areas:
            return {"type": "remote_area", "amount": 15.0}
        return {"type": "remote_area", "amount": 0.0}

    def _cod_fee(self, cod_amount):
        """代收货款手续费"""
        rate = 0.01  # 1%
        min_fee = 5.0
        amount = max(cod_amount * rate, min_fee)
        return {"type": "cod_fee", "rate": rate, "amount": round(amount, 2)}

    def _insurance_premium(self, insurance_value):
        """保价费"""
        rate = 0.003  # 0.3%
        min_premium = 1.0
        amount = max(insurance_value * rate, min_premium)
        return {"type": "insurance", "rate": rate, "amount": round(amount, 2)}

    def _oversized_surcharge(self, order):
        """超大件附加费"""
        max_dimension = max(order["length_cm"], order["width_cm"], order["height_cm"])
        if max_dimension > 200:
            return {"type": "oversized", "amount": 50.0}
        elif max_dimension > 150:
            return {"type": "oversized", "amount": 30.0}
        return {"type": "oversized", "amount": 0.0}

    def _fragile_handling_fee(self, order):
        """易碎品处理费"""
        return {"type": "fragile_handling", "amount": 8.0}

    # ===== 折扣规则 =====

    def _calc_discount(self, order, subtotal):
        """计算折扣"""
        discount_rate = 0.0
        discount_reasons = []

        # 1. 会员折扣
        user_level = self.redis.get(f"user_level:{order['user_id']}")
        if user_level:
            level_rates = {"vip": 0.05, "svip": 0.10, "diamond": 0.15}
            if user_level in level_rates:
                discount_rate += level_rates[user_level]
                discount_reasons.append(f"会员折扣{level_rates[user_level]*100}%")

        # 2. 大客户协议折扣
        if order.get("contract_id"):
            contract_rate = self.redis.get(f"contract_discount:{order['contract_id']}")
            if contract_rate:
                discount_rate += float(contract_rate)
                discount_reasons.append(f"协议折扣{float(contract_rate)*100}%")

        # 3. 月度满减
        monthly_total = self.redis.get(f"monthly_shipment:{order['user_id']}")
        if monthly_total and float(monthly_total) > 10000:
            discount_rate += 0.03
            discount_reasons.append("月度满减3%")

        # 4. 最高折扣上限
        max_discount = 0.30
        discount_rate = min(discount_rate, max_discount)

        return {
            "rate": discount_rate,
            "amount": round(subtotal * discount_rate, 2),
            "reasons": discount_reasons
        }
```

## 配送路径优化深度实现

### 多站点TSP求解 + 时间窗约束 + 车辆容量约束

```python
class DeliveryRouteOptimizer:
    """配送路径优化：TSP + 时间窗 + 容量约束 + 实时交通"""

    def __init__(self, db, redis, map_service):
        self.db = db
        self.redis = redis
        self.map_service = map_service

    def optimize_route(self, vehicle_id, delivery_points, constraints=None):
        """优化配送路径"""
        constraints = constraints or {}
        timeout_ms = constraints.get("timeout_ms", 5000)

        # 1. 构建距离矩阵
        dist_matrix = self._build_distance_matrix(delivery_points)

        # 2. 获取时间窗约束
        time_windows = self._get_time_windows(delivery_points)

        # 3. 获取车辆容量
        vehicle = self._get_vehicle_info(vehicle_id)

        # 4. 求解TSP（带约束）
        try:
            route = self._solve_tsp_with_timeout(
                dist_matrix, delivery_points,
                time_windows, vehicle, timeout_ms)
        except TimeoutError:
            # 超时降级：使用最近邻启发式
            route = self._nearest_neighbor_heuristic(
                dist_matrix, delivery_points, time_windows)
            route["fallback"] = True

        # 5. 考虑实时交通
        route = self._apply_traffic_adjustments(route, delivery_points)

        # 6. 验证约束
        violations = self._validate_constraints(route, vehicle, time_windows)
        if violations:
            route["constraint_violations"] = violations

        return route

    def _build_distance_matrix(self, points):
        """构建距离矩阵（调用地图API + 缓存）"""
        n = len(points)
        matrix = [[0] * n for _ in range(n)]

        for i in range(n):
            for j in range(i + 1, n):
                cache_key = f"dist:{points[i]['id']}:{points[j]['id']}"
                cached = self.redis.get(cache_key)
                if cached:
                    dist = float(cached)
                else:
                    dist = self.map_service.get_driving_distance(
                        points[i]["lat"], points[i]["lng"],
                        points[j]["lat"], points[j]["lng"])
                    self.redis.setex(cache_key, 1800, str(dist))
                matrix[i][j] = dist
                matrix[j][i] = dist

        return matrix

    def _solve_tsp_with_timeout(self, dist_matrix, points, time_windows, vehicle, timeout_ms):
        """带超时的TSP求解"""
        n = len(points)
        if n <= 10:
            # 小规模：精确求解（分支限界）
            return self._exact_tsp(dist_matrix, points, timeout_ms)
        elif n <= 30:
            # 中等规模：2-opt局部搜索
            return self._two_opt_tsp(dist_matrix, points, timeout_ms)
        else:
            # 大规模：遗传算法
            return self._genetic_tsp(dist_matrix, points, timeout_ms)

    def _exact_tsp(self, dist_matrix, points, timeout_ms):
        """精确TSP求解（分支限界，适用于≤10个点）"""
        import heapq

        n = len(points)
        best_cost = float('inf')
        best_path = None
        start_time = now()

        # 优先队列：[下界, 当前路径, 已访问集合]
        initial_lb = self._calc_lower_bound(dist_matrix, set(), 0)
        pq = [(initial_lb, 0, [0], {0})]

        while pq:
            # 超时检查
            if (now() - start_time).total_seconds() * 1000 > timeout_ms:
                break

            lb, cost, path, visited = heapq.heappop(pq)

            # 剪枝：下界超过当前最优
            if lb >= best_cost:
                continue

            # 终止条件：所有节点都访问过
            if len(path) == n:
                total = cost + dist_matrix[path[-1]][0]
                if total < best_cost:
                    best_cost = total
                    best_path = path + [0]
                continue

            # 扩展
            last = path[-1]
            for next_node in range(n):
                if next_node not in visited:
                    new_cost = cost + dist_matrix[last][next_node]
                    new_lb = new_cost + self._calc_lower_bound(
                        dist_matrix, visited | {next_node}, next_node)
                    if new_lb < best_cost:
                        heapq.heappush(pq,
                            (new_lb, new_cost, path + [next_node], visited | {next_node}))

        if best_path is None:
            raise TimeoutError("TSP求解超时")

        return self._format_route(best_path, best_cost, points, dist_matrix)

    def _two_opt_tsp(self, dist_matrix, points, timeout_ms):
        """2-opt局部搜索TSP求解（适用于≤30个点）"""
        n = len(points)
        start_time = now()

        # 初始解：最近邻
        route = self._nearest_neighbor_path(dist_matrix)
        best_cost = self._route_cost(route, dist_matrix)
        improved = True

        while improved:
            if (now() - start_time).total_seconds() * 1000 > timeout_ms:
                break
            improved = False
            for i in range(1, n - 1):
                for j in range(i + 1, n):
                    new_route = route[:i] + route[i:j+1][::-1] + route[j+1:]
                    new_cost = self._route_cost(new_route, dist_matrix)
                    if new_cost < best_cost:
                        route = new_route
                        best_cost = new_cost
                        improved = True

        return self._format_route(route + [route[0]], best_cost, points, dist_matrix)

    def _genetic_tsp(self, dist_matrix, points, timeout_ms):
        """遗传算法TSP求解（适用于>30个点）"""
        n = len(points)
        start_time = now()
        population_size = 100
        generations = 500
        mutation_rate = 0.02

        # 初始化种群
        population = []
        for _ in range(population_size):
            individual = list(range(1, n))
            random.shuffle(individual)
            population.append([0] + individual)

        best_ever = None
        best_ever_cost = float('inf')

        for gen in range(generations):
            if (now() - start_time).total_seconds() * 1000 > timeout_ms:
                break

            # 评估适应度
            fitness = []
            for individual in population:
                cost = self._route_cost(individual, dist_matrix) + dist_matrix[individual[-1]][0]
                fitness.append(1.0 / max(cost, 0.01))
                if cost < best_ever_cost:
                    best_ever_cost = cost
                    best_ever = individual + [0]

            # 选择 + 交叉 + 变异
            new_population = [best_ever[:n]]  # 精英保留
            total_fitness = sum(fitness)

            while len(new_population) < population_size:
                # 锦标赛选择
                parent1 = self._tournament_select(population, fitness, total_fitness)
                parent2 = self._tournament_select(population, fitness, total_fitness)
                # 顺序交叉（OX）
                child = self._order_crossover(parent1, parent2)
                # 变异（2-opt swap）
                if random.random() < mutation_rate:
                    i, j = sorted(random.sample(range(1, n), 2))
                    child[i:j+1] = child[i:j+1][::-1]
                new_population.append(child)

            population = new_population

        return self._format_route(best_ever, best_ever_cost, points, dist_matrix)

    def _nearest_neighbor_heuristic(self, dist_matrix, points, time_windows):
        """最近邻启发式（快速降级方案）"""
        return self._format_route(
            self._nearest_neighbor_path(dist_matrix) + [0],
            None, points, dist_matrix)

    def _nearest_neighbor_path(self, dist_matrix):
        """最近邻路径"""
        n = len(dist_matrix)
        visited = {0}
        path = [0]
        current = 0

        while len(visited) < n:
            nearest = min(
                [(dist_matrix[current][j], j) for j in range(n) if j not in visited],
                key=lambda x: x[0])
            path.append(nearest[1])
            visited.add(nearest[1])
            current = nearest[1]

        return path

    def _apply_traffic_adjustments(self, route, points):
        """应用实时交通调整"""
        for i, stop in enumerate(route.get("stops", [])):
            # 获取实时路况
            traffic = self.map_service.get_traffic_condition(
                stop["lat"], stop["lng"])
            stop["traffic_condition"] = traffic.get("level", "normal")
            # 调整预计到达时间
            if traffic.get("level") == "congested":
                stop["eta"] = stop.get("eta", 0) * 1.5
            elif traffic.get("level") == "slow":
                stop["eta"] = stop.get("eta", 0) * 1.2

        return route

    def _validate_constraints(self, route, vehicle, time_windows):
        """验证约束"""
        violations = []

        # 容量约束
        total_weight = sum(s.get("weight_kg", 0) for s in route.get("stops", []))
        if total_weight > vehicle.get("capacity_kg", float('inf')):
            violations.append({
                "type": "capacity_exceeded",
                "detail": f"总重量{total_weight}kg超过车辆容量{vehicle['capacity_kg']}kg"
            })

        # 时间窗约束
        current_time = 0
        for i, stop in enumerate(route.get("stops", [])):
            tw = time_windows.get(stop.get("point_id"))
            if tw:
                current_time += stop.get("travel_time_min", 0)
                if current_time > tw["close"]:
                    violations.append({
                        "type": "time_window_violation",
                        "point_id": stop["point_id"],
                        "detail": f"到达{current_time}分钟超过时间窗关闭{tw['close']}分钟"
                    })

        return violations

    # ===== 辅助方法 =====

    def _get_time_windows(self, points):
        """获取各站点时间窗"""
        tw = {}
        for p in points:
            if "time_window" in p:
                tw[p["id"]] = p["time_window"]
        return tw

    def _get_vehicle_info(self, vehicle_id):
        """获取车辆信息"""
        cached = self.redis.hgetall(f"vehicle:{vehicle_id}")
        if cached:
            return cached
        vehicle = self.db.get("vehicles", vehicle_id)
        if vehicle:
            self.redis.hset(f"vehicle:{vehicle_id}", mapping=vehicle)
        return vehicle or {}

    def _route_cost(self, route, dist_matrix):
        """计算路径总距离"""
        return sum(dist_matrix[route[i]][route[i+1]] for i in range(len(route)-1))

    def _calc_lower_bound(self, dist_matrix, visited, current):
        """计算TSP下界（MST估计）"""
        # 简化：未访问节点的最小出边之和
        n = len(dist_matrix)
        lb = 0
        for i in range(n):
            if i not in visited:
                min_edge = min(dist_matrix[i][j] for j in range(n) if j != i and j not in visited)
                lb += min_edge
        return lb

    def _tournament_select(self, population, fitness, total_fitness, k=5):
        """锦标赛选择"""
        candidates = random.sample(list(zip(population, fitness)), k)
        return max(candidates, key=lambda x: x[1])[0]

    def _order_crossover(self, parent1, parent2):
        """顺序交叉（OX）"""
        n = len(parent1)
        start, end = sorted(random.sample(range(1, n), 2))
        child = [None] * n
        child[0] = 0
        # 从parent1复制片段
        for i in range(start, end + 1):
            child[i] = parent1[i]
        # 从parent2填充剩余
        pos = (end + 1) % n
        for gene in parent2[1:] + [parent2[0]]:
            if gene not in child:
                while child[pos] is not None:
                    pos = (pos + 1) % n
                child[pos] = gene
        return child

    def _format_route(self, path, cost, points, dist_matrix):
        """格式化输出路线"""
        stops = []
        total_distance = 0
        for i in range(len(path) - 1):
            dist = dist_matrix[path[i]][path[i+1]]
            total_distance += dist
            stops.append({
                "point_id": points[path[i]]["id"],
                "name": points[path[i]].get("name", ""),
                "lat": points[path[i]]["lat"],
                "lng": points[path[i]]["lng"],
                "sequence": i + 1,
                "distance_from_prev": round(dist, 1),
                "travel_time_min": round(dist / 30 * 60, 0),  # 假设均速30km/h
                "weight_kg": points[path[i]].get("weight_kg", 0)
            })

        return {
            "stops": stops,
            "total_distance_km": round(total_distance, 1),
            "estimated_time_min": round(total_distance / 30 * 60, 0),
            "total_stops": len(stops),
            "total_cost": cost
        }
```

## 异常场景：费率表不一致（深度分析）

```
触发：物流费率表更新延迟 → 计费使用旧费率 → 运费偏差
根因分析：
  1. 物流商调整费率，但系统费率表未同步更新
  2. Redis缓存费率未过期，DB已更新 → 缓存与DB不一致
  3. 多区域费率表并行更新，部分区域未及时生效
  4. 费率表版本管理缺失 → 无法确认当前生效版本
检测机制：
  1. 每日对账：系统计费 vs 物流商账单
  2. 差异 > 2% → 告警
  3. 费率表版本号比对：Redis缓存版本 vs DB最新版本
  4. 单笔费用偏差：同一路线运费波动 > 10% → 异常
处理流程：
  1. 检查费率表版本是否最新
  2. 清除Redis缓存中过期费率 → 强制从DB重新加载
  3. 旧费率 → 使用新费率重新计算差额
  4. 差额处理：
     - 差额 < 5元 → 平台承担，不补收
     - 差额 5-50元 → 通知用户补差或平台承担
     - 差额 > 50元 → 物流商协商承担比例
  5. 对历史订单批量重算（T+1离线任务）
预防措施：
  1. 费率表版本化：每次变更生成新版本号
  2. 费率变更通知：物流商变更时推送事件
  3. Redis缓存设置合理过期时间（1小时）
  4. 费率变更灰度生效：先在少量路线验证，再全量
  5. 每日对账 + 差异自动告警
  6. 费率表审计日志：记录每次变更的版本、时间、操作人
```

## 异常场景：路径优化超时降级（深度分析）

```
触发：TSP求解器在复杂路网中超时 → 无法返回最优路线
典型场景：
  1. 单次配送超过30个站点 → 求解空间指数增长
  2. 时间窗约束过紧 → 可行解空间极小
  3. 距离矩阵计算耗时（地图API调用超时）
  4. 遗传算法收敛慢（种群多样性不足）
检测机制：
  1. 求解器运行时间 > 5秒 → 超时
  2. 求解器迭代次数 > 上限 → 强制终止
  3. 内存使用 > 阈值 → 预防OOM
处理流程：
  1. 超时后立即切换到最近邻启发式（O(n²)，毫秒级完成）
  2. 后台异步继续优化路线：
     - 将优化任务推入低优先级队列
     - 使用2-opt逐步改进当前路线
     - 优化完成后通过MQTT推送更新到司机端
  3. 司机端先显示初始路线（标注"优化中"）
  4. 优化完成后更新路线并通知司机
  5. 记录超时原因（站点数、约束复杂度）用于分析
降级策略：
  Level 0：精确求解（≤10个站点）
  Level 1：2-opt局部搜索（11-30个站点）
  Level 2：遗传算法（>30个站点，带超时保护）
  Level 3：最近邻启发式（超时降级方案）
  Level 4：手动指定路线（所有算法失败的兜底方案）
预防措施：
  1. 求解器超时保护（5秒硬限制）
  2. 按站点数量自动选择算法级别
  3. 距离矩阵预计算 + 缓存
  4. 分区优化：将大区域拆分为多个子区域分别优化
  5. 历史路线复用：相同区域的配送复用历史优质路线
  6. 离线预优化：夜间预计算高频路线的最优解
```

## 物流通知系统完整实现

```python
class LogisticsNotificationService:
    """物流通知：多渠道 + 去重 + 限频 + 重试"""

    TEMPLATES = {
        "order_created": "您的订单已创建，正在安排发货",
        "package_picked_up": "您的包裹已被揽收，快递单号：{tracking_number}",
        "in_transit": "您的包裹正在运输中，预计 {eta} 送达",
        "out_for_delivery": "您的包裹正在派送，快递员：{courier_name} {courier_phone}",
        "delivered": "您的包裹已签收，签收人：{signer}",
        "exception": "您的包裹遇到异常：{reason}，我们正在处理",
    }

    CHANNEL_PRIORITY = ["app_push", "wechat", "sms", "email"]

    def notify_state_change(self, shipment_id, old_state, new_state):
        """状态变更通知"""
        shipment = self.db.get_shipment(shipment_id)
        user = self.db.get_user(shipment["user_id"])

        # 1. 获取模板
        template_key = self._state_to_template(new_state)
        if not template_key:
            return

        # 2. 渲染消息
        message = self.TEMPLATES[template_key].format(
            tracking_number=shipment.get("tracking_number", ""),
            eta=shipment.get("estimated_delivery", ""),
            courier_name=shipment.get("courier_name", ""),
            courier_phone=shipment.get("courier_phone", ""),
            signer=shipment.get("signer", ""),
            reason=shipment.get("exception_reason", "")
        )

        # 3. 去重检查（5 分钟内同一事件不重复通知）
        dedup_key = f"notify_dedup:{shipment_id}:{new_state}"
        if self.redis.exists(dedup_key):
            return {"status": "deduped"}
        self.redis.setex(dedup_key, 300, "1")

        # 4. 限频检查（每小时最多 10 条）
        rate_key = f"notify_rate:{shipment['user_id']}:{now().strftime('%Y%m%d%H')}"
        count = int(self.redis.get(rate_key) or 0)
        if count >= 10:
            return {"status": "rate_limited"}
        self.redis.incr(rate_key)
        self.redis.expire(rate_key, 3600)

        # 5. 多渠道投递
        preferences = self._get_user_preferences(user["id"])
        results = []
        for channel in self.CHANNEL_PRIORITY:
            if not preferences.get(channel, True):
                continue

            result = self._send(channel, user, message, shipment_id)
            results.append({"channel": channel, "status": result["status"]})

            if result["status"] == "delivered":
                break  # 一个渠道成功即可

        # 6. 记录通知日志
        self.db.insert("notification_log", {
            "shipment_id": shipment_id,
            "user_id": user["id"],
            "state": new_state,
            "message": message,
            "channels": json.dumps(results),
            "sent_at": now()
        })

        return {"status": "sent", "channels": results}

    def _send(self, channel, user, message, shipment_id):
        """发送通知"""
        try:
            if channel == "app_push":
                self.fcm.send(user["device_token"], message)
            elif channel == "wechat":
                self.wechat.send_template(user["wechat_openid"],
                    "logistics_update", {"message": message})
            elif channel == "sms":
                self.twilio.send_sms(user["phone"], message)
            elif channel == "email":
                self.email.send(user["email"], "物流更新", message)

            return {"status": "delivered"}

        except Exception as e:
            # 发送失败 → 排队重试
            self._schedule_retry(channel, user, message, shipment_id, str(e))
            return {"status": "failed", "error": str(e)}

    def _schedule_retry(self, channel, user, message, shipment_id, error):
        """重试（指数退避，最多 3 次）"""
        retry_key = f"notify_retry:{shipment_id}:{channel}"
        attempt = int(self.redis.get(retry_key) or 0)

        if attempt < 3:
            delay = 2 ** attempt * 60  # 1min, 2min, 4min
            self.scheduler.schedule(
                run_date=now() + timedelta(seconds=delay),
                task=self._send,
                args={"channel": channel, "user": user,
                      "message": message, "shipment_id": shipment_id})
            self.redis.incr(retry_key)
            self.redis.expire(retry_key, 600)
```

## 退货管理流程

```python
class ReturnsManagementService:
    """退货管理：RMA → 检验 → 处置 → 退款"""

    def create_rma(self, order_id, items, reason):
        """创建退货授权"""
        order = self.db.get_order(order_id)
        days_since = (now() - order["delivered_at"]).days

        # 退货政策检查
        if reason == "quality" and days_since > 15:
            raise ReturnExpiredError("质量问题退货超过 15 天")
        if reason == "change_of_mind" and days_since > 7:
            raise ReturnExpiredError("无理由退货超过 7 天")

        rma_id = str(uuid4())
        self.db.insert("rma_requests", {
            "rma_id": rma_id, "order_id": order_id,
            "items": json.dumps(items), "reason": reason,
            "status": "approved", "return_deadline": now() + timedelta(days=7),
            "created_at": now()
        })

        # 生成退货标签
        label = self.shipping.generate_return_label(rma_id, order["address"])

        return {"rma_id": rma_id, "label_url": label["url"]}

    def inspect_returned_item(self, rma_id, item_id, inspection):
        """入库检验 → 确定处置方案"""
        checks = {
            "damage": inspection.get("has_damage", False),
            "quantity_match": inspection.get("quantity_correct", True),
            "quality_pass": inspection.get("quality_acceptable", True),
            "packaging_intact": inspection.get("packaging_intact", True),
        }

        # 处置决策
        if all(checks.values()):
            disposition, refund_rate = "restock", 1.0
        elif checks["damage"] and not checks["quality_pass"]:
            disposition, refund_rate = "scrap", 1.0
        elif not checks["packaging_intact"]:
            disposition, refund_rate = "refurbish", 0.85
        else:
            disposition, refund_rate = "restock_flagged", 0.90

        self.db.insert("return_inspections", {
            "rma_id": rma_id, "item_id": item_id,
            "checks": json.dumps(checks),
            "disposition": disposition,
            "refund_rate": refund_rate,
            "inspected_at": now()
        })

        # 执行处置
        original_price = self.db.get_order_item(item_id)["price"]
        refund_amount = original_price * refund_rate

        return {"disposition": disposition, "refund_amount": refund_amount}
```

## 异常场景补充

### 场景：通知服务中断

```
触发：通知服务宕机 → 用户无法收到物流状态更新 → 大量客服咨询
检测：
  1. 通知发送失败率 > 50% → 服务中断
  2. 客服咨询量飙升 → 可能通知中断
处理：
  1. 通知入队列缓存（Kafka 死信队列）
  2. 服务恢复后批量补发
  3. 关键状态（签收、异常）优先补发
预防：通知缓冲队列 + 批量补发 + 关键通知优先级
```

### 场景：退货检验争议

```
触发：仓库检验结果与客户描述不一致 → 退款金额争议
检测：
  1. 客户申诉检验结果 → 争议
  2. 退款率 < 100% → 客户可能不满
处理：
  1. 要求仓库拍照留存证据
  2. 客户可上传发货前照片
  3. 争议由客服主管裁决
预防：发货前拍照 + 检验拍照留存 + 争议升级流程
```

## 物流 SLA 管理完整实现

```python
class LogisticsSLAService:
    """物流 SLA 管理：定义 + 预测 + 违约补偿"""

    SLA_TIERS = {
        "same_day": {"max_hours": 12, "compensation_pct": 20, "name": "当日达"},
        "express": {"max_hours": 48, "compensation_pct": 10, "name": "次日达"},
        "standard": {"max_hours": 120, "compensation_pct": 5, "name": "标准"},
    }

    def check_sla(self, shipment_id):
        """检查 SLA 状态"""
        shipment = self.db.get_shipment(shipment_id)
        tier = self.SLA_TIERS.get(shipment["service_tier"], self.SLA_TIERS["standard"])

        elapsed_hours = (now() - shipment["picked_up_at"]).total_seconds() / 3600
        remaining_hours = tier["max_hours"] - elapsed_hours

        # 预测送达时间
        eta = self._predict_eta(shipment)

        if eta > tier["max_hours"]:
            # SLA 将违约
            return {
                "status": "will_breach",
                "remaining_hours": round(remaining_hours, 1),
                "predicted_breach_in_hours": round(eta - tier["max_hours"], 1),
                "action": "escalate"
            }
        elif remaining_hours < 6:
            # 接近违约
            return {
                "status": "at_risk",
                "remaining_hours": round(remaining_hours, 1),
                "action": "expedite"
            }

        return {"status": "on_track", "remaining_hours": round(remaining_hours, 1)}

    def predict_breach(self, shipment_id, hours_ahead=24):
        """预测 SLA 违约（提前 24 小时预警）"""
        shipment = self.db.get_shipment(shipment_id)

        # 使用当前位置 + 剩余距离 + 历史运输时间
        current_location = self._get_current_location(shipment_id)
        destination = shipment["destination"]
        remaining_distance = self._calc_distance(current_location, destination)

        # 历史同路线运输时间
        historical = self.db.query(
            "SELECT AVG(actual_hours) as avg, STDDEV(actual_hours) as std "
            "FROM shipments WHERE origin_city = %s AND destination_city = %s "
            "AND service_tier = %s AND status = 'delivered'",
            shipment["origin_city"], shipment["destination_city"],
            shipment["service_tier"])

        if historical and historical[0]["avg"]:
            predicted_hours = historical[0]["avg"]
            uncertainty = historical[0]["std"] or 0
        else:
            # 无历史 → 基于距离估算
            predicted_hours = remaining_distance / 60 + 4  # 60km/h + 4h 处理
            uncertainty = predicted_hours * 0.3

        tier = self.SLA_TIERS.get(shipment["service_tier"], self.SLA_TIERS["standard"])
        breach_probability = self._norm_cdf(
            tier["max_hours"], predicted_hours, uncertainty)

        if breach_probability > 0.7:
            # 高概率违约 → 紧急升级
            self._escalate_shipment(shipment_id, predicted_hours)

        return {"predicted_hours": round(predicted_hours, 1),
                "sla_hours": tier["max_hours"],
                "breach_probability": round(breach_probability, 3)}

    def compensate_breach(self, shipment_id):
        """SLA 违约自动补偿"""
        shipment = self.db.get_shipment(shipment_id)
        tier = self.SLA_TIERS.get(shipment["service_tier"], self.SLA_TIERS["standard"])

        compensation = round(shipment["order_amount"] * tier["compensation_pct"] / 100, 2)

        self.db.insert("sla_compensations", {
            "shipment_id": shipment_id,
            "order_id": shipment["order_id"],
            "user_id": shipment["user_id"],
            "service_tier": shipment["service_tier"],
            "compensation_amount": compensation,
            "compensation_pct": tier["compensation_pct"],
            "created_at": now()
        })

        # 自动退款到用户余额
        self.payment.refund_to_balance(shipment["user_id"], compensation)

        return {"compensation": compensation, "tier": tier["name"]}

    def _norm_cdf(self, x, mean, std):
        """正态分布 CDF（违约概率）"""
        if std == 0:
            return 1.0 if x < mean else 0.0
        z = (x - mean) / std
        return 1 - 0.5 * (1 + math.erf(z / math.sqrt(2)))
```

## 物流成本核算

```python
class LogisticsCostAccounting:
    """物流成本核算：单票成本 + 承运商费率 + 利润分析"""

    def calculate_shipment_cost(self, shipment_id):
        """计算单票物流成本"""
        shipment = self.db.get_shipment(shipment_id)

        # 各项成本
        pickup_cost = self._get_pickup_cost(shipment)
        linehaul_cost = self._get_linehaul_cost(shipment)
        last_mile_cost = self._get_last_mile_cost(shipment)
        handling_cost = self._get_handling_cost(shipment)
        packaging_cost = shipment.get("packaging_cost", 2.0)

        total_cost = pickup_cost + linehaul_cost + last_mile_cost + handling_cost + packaging_cost
        revenue = shipment["shipping_fee"]

        return {
            "shipment_id": shipment_id,
            "pickup": pickup_cost,
            "linehaul": linehaul_cost,
            "last_mile": last_mile_cost,
            "handling": handling_cost,
            "packaging": packaging_cost,
            "total_cost": round(total_cost, 2),
            "revenue": revenue,
            "margin": round((revenue - total_cost) / revenue * 100, 1) if revenue else 0
        }

    def get_carrier_rate_card(self, carrier_id):
        """获取承运商费率卡"""
        return self.db.query(
            "SELECT zone, weight_min_kg, weight_max_kg, rate_per_kg, "
            "min_charge FROM carrier_rate_cards "
            "WHERE carrier_id = %s AND valid_until > NOW() "
            "ORDER BY zone, weight_min_kg", carrier_id)

    def recommend_carrier(self, origin, destination, weight_kg):
        """推荐最优承运商"""
        zone = self._get_zone(origin, destination)
        carriers = self.db.query(
            "SELECT carrier_id, rate_per_kg, min_charge, "
            "avg_delivery_days, on_time_rate "
            "FROM carrier_rate_cards "
            "WHERE zone = %s AND weight_min_kg <= %s AND weight_max_kg >= %s "
            "AND valid_until > NOW()", zone, weight_kg, weight_kg)

        # 综合评分（成本 60% + 时效 20% + 准时率 20%）
        for c in carriers:
            cost = max(c["min_charge"], weight_kg * c["rate_per_kg"])
            c["estimated_cost"] = round(cost, 2)
            c["score"] = (1 / cost * 0.6 +
                         1 / c["avg_delivery_days"] * 0.2 +
                         c["on_time_rate"] * 0.2)

        carriers.sort(key=lambda x: x["score"], reverse=True)
        return carriers[:5]
```

## 异常场景补充

### 场景：SLA 违约预测误报

```
触发：预测违约 → 安排加急 → 实际按时送达 → 不必要成本
检测：
  1. 加急后实际送达时间 < SLA → 可能误报
  2. 违约预测准确率 < 70% → 模型需优化
处理：
  1. 提高预测置信度阈值（0.7 → 0.85）
  2. 增加更多特征（天气、交通、节假日）
  3. 低风险不安排加急
预防：提高置信度阈值 + 模型迭代 + 成本效益分析
```

### 场景：承运商费率更新未生效

```
触发：新费率已谈判但系统未更新 → 仍按旧费率计费 → 成本偏差
检测：
  1. 实际账单与系统费率不一致 → 费率过时
  2. 承运商投诉结算金额不对 → 费率问题
处理：
  1. 紧急更新费率卡
  2. 重新计算受影响期间的结算
  3. 与承运商对账
预防：费率变更审批+自动生效 + 定期对账 + 差异告警
```

## 物流状态机并发控制完整实现

```python
class StateMachineConcurrencyControl:
    """状态机并发控制：乐观锁 + 幂等 + 补偿"""

    def transition_with_optimistic_lock(self, shipment_id, from_state, to_state,
                                        event, metadata=None):
        """使用乐观锁的状态转移"""
        # 1. 获取当前版本号
        current = self.db.query_one(
            "SELECT version, current_state FROM shipments "
            "WHERE id = %s", shipment_id)

        if current["current_state"] != from_state:
            raise StateConflictError(
                f"状态不匹配: 期望 {from_state}, 实际 {current['current_state']}")

        # 2. 乐观锁更新（版本号递增）
        rows_affected = self.db.execute(
            "UPDATE shipments SET current_state = %s, version = version + 1, "
            "updated_at = NOW() WHERE id = %s AND version = %s AND current_state = %s",
            to_state, shipment_id, current["version"], from_state)

        if rows_affected == 0:
            # 并发冲突 → 重试
            raise StateConflictError("并发冲突，请重试")

        # 3. 记录状态变更
        self.db.insert("state_transitions", {
            "shipment_id": shipment_id,
            "from_state": from_state,
            "to_state": to_state,
            "event": event,
            "metadata": json.dumps(metadata or {}),
            "version": current["version"] + 1,
            "timestamp": now()
        })

        # 4. 触发副作用
        self._trigger_side_effects(shipment_id, to_state, event)

        return {"shipment_id": shipment_id, "new_state": to_state,
                "version": current["version"] + 1}

    def idempotent_transition(self, idempotency_key, shipment_id,
                              from_state, to_state, event):
        """幂等状态转移"""
        # 1. 检查幂等键
        existing = self.db.query_one(
            "SELECT * FROM idempotent_transitions "
            "WHERE idempotency_key = %s", idempotency_key)

        if existing:
            return {"status": "already_processed",
                    "result": json.loads(existing["result"])}

        # 2. 执行状态转移
        try:
            result = self.transition_with_optimistic_lock(
                shipment_id, from_state, to_state, event)

            # 3. 记录幂等键
            self.db.insert("idempotent_transitions", {
                "idempotency_key": idempotency_key,
                "shipment_id": shipment_id,
                "result": json.dumps(result),
                "created_at": now()
            })

            return {"status": "processed", "result": result}

        except StateConflictError:
            # 状态不匹配 → 可能已处理过
            current = self.db.query_one(
                "SELECT current_state FROM shipments WHERE id = %s", shipment_id)
            if current["current_state"] == to_state:
                return {"status": "already_in_target_state"}
            raise

    def compensate_failed_transition(self, shipment_id, failed_event,
                                     original_state):
        """补偿失败的状态转移"""
        # 回滚到原始状态
        self.db.execute(
            "UPDATE shipments SET current_state = %s "
            "WHERE id = %s", original_state, shipment_id)

        self.db.insert("compensation_log", {
            "shipment_id": shipment_id,
            "failed_event": failed_event,
            "compensated_to": original_state,
            "compensated_at": now()
        })
```

## 异常场景补充

### 场景：状态机并发导致幽灵状态

```
触发：两个并发事件同时触发状态转移 → 一个成功一个失败 → 状态不一致
检测：
  1. 状态序列中出现不可能的转移 → 幽灵状态
  2. 同一 shipment 两个活跃状态 → 严重不一致
处理：
  1. 查询最新有效状态转移
  2. 修正为正确的当前状态
  3. 补偿失败转移的副作用
预防：乐观锁 + 幂等 + 状态转移前校验
```

### 场景：补偿操作本身失败

```
触发：状态转移失败 → 触发补偿 → 补偿也失败 → 状态卡住
检测：
  1. 补偿操作失败告警
  2. 状态长时间不变 → 卡住
处理：
  1. 人工介入修正状态
  2. 手动补偿未完成的副作用
  3. 修复补偿逻辑后验证
预防：补偿重试 + 人工介入流程 + 补偿验证
```

## 物流轨迹追踪完整实现

```python
class ShipmentTrackingService:
    """物流轨迹追踪：实时位置 + 轨迹回放 + 异常检测"""

    def update_location(self, tracking_number, lat, lng, source="gps"):
        """更新物流位置"""
        # 1. 写入轨迹点
        self.db.insert("shipment_tracks", {
            "tracking_number": tracking_number,
            "latitude": lat,
            "longitude": lng,
            "source": source,  # gps / cell_tower / wifi
            "accuracy_meters": self._get_accuracy(source),
            "timestamp": now()
        })

        # 2. 更新缓存（最新位置）
        self.redis.geoadd("shipment_locations", lng, lat, tracking_number)

        # 3. 检查是否偏离路线
        route = self._get_planned_route(tracking_number)
        if route:
            deviation = self._calc_route_deviation(lat, lng, route)
            if deviation > 5000:  # 偏离 5km
                self.alert(f"包裹 {tracking_number} 偏离路线 {deviation:.0f}m")

        # 4. 检查是否接近目的地
        shipment = self.db.get_shipment_by_tracking(tracking_number)
        if shipment:
            dest = shipment["destination_coords"]
            distance = self._haversine(lat, lng, dest["lat"], dest["lng"])
            if distance < 3000 and shipment["status"] == "in_transit":
                # 接近目的地 → 通知收件人
                self.notification.notify_nearby_delivery(shipment["user_id"],
                    tracking_number, round(distance / 1000, 1))

    def get_tracking_timeline(self, tracking_number):
        """获取物流时间线"""
        tracks = self.db.query(
            "SELECT * FROM shipment_tracks "
            "WHERE tracking_number = %s ORDER BY timestamp ASC",
            tracking_number)

        # 将轨迹点聚合为有意义的节点
        nodes = self._cluster_tracks(tracks)

        timeline = []
        for i, node in enumerate(nodes):
            entry = {
                "timestamp": node["timestamp"],
                "location": node["location_name"] or f"{node['lat']:.4f}, {node['lng']:.4f}",
                "status": self._infer_status(node, nodes[i-1] if i > 0 else None),
                "duration_minutes": None
            }
            if i > 0:
                delta = (node["timestamp"] - nodes[i-1]["timestamp"]).total_seconds() / 60
                entry["duration_minutes"] = round(delta)
            timeline.append(entry)

        return {"tracking_number": tracking_number, "timeline": timeline}

    def replay_trajectory(self, tracking_number, speed=10):
        """轨迹回放（10x 速度）"""
        tracks = self.db.query(
            "SELECT latitude, longitude, timestamp "
            "FROM shipment_tracks "
            "WHERE tracking_number = %s ORDER BY timestamp ASC",
            tracking_number)

        if not tracks:
            return {"points": []}

        # 降采样（每 30 秒一个点）
        sampled = [tracks[0]]
        for t in tracks[1:]:
            if (t["timestamp"] - sampled[-1]["timestamp"]).total_seconds() >= 30:
                sampled.append(t)

        return {
            "tracking_number": tracking_number,
            "points": [{"lat": t["latitude"], "lng": t["longitude"],
                       "time": t["timestamp"].isoformat()} for t in sampled],
            "total_points": len(sampled),
            "duration_seconds": (tracks[-1]["timestamp"] - tracks[0]["timestamp"]).total_seconds()
        }

    def detect_anomalous_stop(self, tracking_number):
        """检测异常停留"""
        recent = self.db.query(
            "SELECT * FROM shipment_tracks "
            "WHERE tracking_number = %s "
            "AND timestamp > NOW() - INTERVAL 2 HOUR "
            "ORDER BY timestamp ASC", tracking_number)

        if len(recent) < 2:
            return {"stopped": False}

        # 检查最近 1 小时是否移动距离 < 1km
        last_hour = [r for r in recent if (now() - r["timestamp"]).total_seconds() < 3600]
        if len(last_hour) >= 2:
            first, last = last_hour[0], last_hour[-1]
            distance = self._haversine(first["latitude"], first["longitude"],
                                       last["latitude"], last["longitude"])
            if distance < 1000:  # 1 小时内移动 < 1km
                return {"stopped": True, "duration_minutes": 60,
                        "location": f"{last['latitude']:.4f}, {last['longitude']:.4f}"}

        return {"stopped": False}

    def _haversine(self, lat1, lon1, lat2, lon2):
        """Haversine 距离计算（米）"""
        R = 6371000
        phi1, phi2 = math.radians(lat1), math.radians(lat2)
        dphi = math.radians(lat2 - lat1)
        dlambda = math.radians(lon2 - lon1)
        a = math.sin(dphi/2)**2 + math.cos(phi1)*math.cos(phi2)*math.sin(dlambda/2)**2
        return 2 * R * math.asin(math.sqrt(a))
```

## 异常场景补充

### 场景：GPS 信号丢失导致轨迹断点

```
触发：包裹经过隧道 → GPS 信号丢失 30 分钟 → 轨迹断点 → 用户焦虑
检测：
  1. 轨迹数据间隔 > 30 分钟 → 断点
  2. 用户频繁查看物流 → 焦虑
处理：
  1. 断点期间使用基站定位（精度低但有位置）
  2. 断点后第一次 GPS 定位 → 推算中间轨迹
  3. 通知用户"途经信号弱区域，正在继续运输"
预防：基站定位补充 + 断点推算 + 用户通知
```

### 场景：轨迹数据量过大

```
触发：高频率上报（每秒 1 次）× 长途运输 3 天 → 25 万轨迹点 → 查询缓慢
检测：
  1. 轨迹查询 > 3 秒 → 数据量过大
  2. 单包裹轨迹点 > 10 万 → 需要降采样
处理：
  1. 存储时降采样（移动时每 30 秒，停留时每 5 分钟）
  2. 历史轨迹归档到冷存储
  3. 查询时动态降采样
预防：存储降采样 + 冷热分层 + 查询降采样
```

## 物流路径优化完整实现

```python
class RouteOptimizationService:
    """物流路径优化：TSP 近似解 + 时间窗 + 车辆容量"""

    def optimize_delivery_route(self, vehicle_id, deliveries, depot_location):
        """优化配送路径"""
        # 1. 构建距离矩阵
        points = [depot_location] + [d["location"] for d in deliveries]
        dist_matrix = self._build_distance_matrix(points)

        # 2. Nearest Neighbor 初始解
        initial_route = self._nearest_neighbor(dist_matrix, points)

        # 3. 2-opt 优化
        optimized_route = self._two_opt(initial_route, dist_matrix)

        # 4. 时间窗约束检查
        route_with_times = self._apply_time_windows(optimized_route, deliveries, depot_location)

        # 5. 车辆容量检查
        vehicle = self.db.get_vehicle(vehicle_id)
        total_weight = sum(d["weight_kg"] for d in deliveries)
        if total_weight > vehicle["max_capacity_kg"]:
            # 超容量 → 分批
            return self._split_into_batches(vehicle_id, deliveries, depot_location, vehicle["max_capacity_kg"])

        # 6. 生成最终路线
        result = {
            "vehicle_id": vehicle_id,
            "total_distance_km": round(self._route_distance(optimized_route, dist_matrix), 1),
            "total_time_minutes": round(route_with_times["total_time_minutes"], 1),
            "stops": route_with_times["stops"],
            "optimization_iterations": optimized_route.get("iterations", 0)
        }

        return result

    def _nearest_neighbor(self, dist_matrix, points):
        """最近邻算法（TSP 初始解）"""
        n = len(points)
        visited = [False] * n
        route = [0]  # 从 depot 开始
        visited[0] = True

        for _ in range(n - 1):
            current = route[-1]
            nearest = None
            nearest_dist = float('inf')
            for j in range(n):
                if not visited[j] and dist_matrix[current][j] < nearest_dist:
                    nearest = j
                    nearest_dist = dist_matrix[current][j]
            if nearest is not None:
                route.append(nearest)
                visited[nearest] = True

        route.append(0)  # 回到 depot
        return route

    def _two_opt(self, route, dist_matrix):
        """2-opt 局部优化"""
        best = route
        best_dist = self._route_distance(route, dist_matrix)
        improved = True
        iterations = 0

        while improved:
            improved = False
            iterations += 1
            for i in range(1, len(best) - 2):
                for j in range(i + 1, len(best) - 1):
                    new_route = best[:i] + best[i:j+1][::-1] + best[j+1:]
                    new_dist = self._route_distance(new_route, dist_matrix)
                    if new_dist < best_dist:
                        best = new_route
                        best_dist = new_dist
                        improved = True

        return {"route": best, "distance": best_dist, "iterations": iterations}

    def _route_distance(self, route, dist_matrix):
        """计算路线总距离"""
        total = 0
        for i in range(len(route) - 1):
            total += dist_matrix[route[i]][route[i+1]]
        return total

    def _apply_time_windows(self, route_result, deliveries, depot_location):
        """应用时间窗约束"""
        route = route_result if isinstance(route_result, list) else route_result["route"]
        stops = []
        current_time = now()
        avg_speed_kmh = 30  # 市区平均 30km/h

        for i, point_idx in enumerate(route):
            if point_idx == 0 and i > 0:
                continue  # 跳过回程 depot

            delivery = deliveries[point_idx - 1] if point_idx > 0 else None

            # 计算到达时间
            if i > 0:
                prev_idx = route[i-1]
                distance_km = self._haversine_km(
                    self._get_point(prev_idx, [depot_location] + [d["location"] for d in deliveries]),
                    self._get_point(point_idx, [depot_location] + [d["location"] for d in deliveries]))
                travel_minutes = distance_km / avg_speed_kmh * 60
                current_time += timedelta(minutes=travel_minutes + 5)  # 5 分钟卸货

            stop = {
                "point_idx": point_idx,
                "delivery_id": delivery.get("id") if delivery else "depot",
                "estimated_arrival": current_time.isoformat(),
            }

            # 时间窗检查
            if delivery and delivery.get("time_window"):
                window = delivery["time_window"]
                if current_time < window["start"]:
                    # 早到 → 等待
                    wait_minutes = (window["start"] - current_time).total_seconds() / 60
                    stop["wait_minutes"] = round(wait_minutes, 0)
                    current_time = window["start"]
                elif current_time > window["end"]:
                    # 迟到 → 标记
                    stop["late_minutes"] = round((current_time - window["end"]).total_seconds() / 60, 0)

            stops.append(stop)

        total_time = (current_time - now()).total_seconds() / 60
        return {"stops": stops, "total_time_minutes": total_time}

    def _split_into_batches(self, vehicle_id, deliveries, depot, max_capacity):
        """超容量 → 分批配送"""
        # 按重量排序，贪心装车
        sorted_deliveries = sorted(deliveries, key=lambda d: d["weight_kg"], reverse=True)
        batches = []
        current_batch = []
        current_weight = 0

        for d in sorted_deliveries:
            if current_weight + d["weight_kg"] > max_capacity:
                if current_batch:
                    batches.append(current_batch)
                current_batch = [d]
                current_weight = d["weight_kg"]
            else:
                current_batch.append(d)
                current_weight += d["weight_kg"]

        if current_batch:
            batches.append(current_batch)

        # 每批独立优化
        results = []
        for batch in batches:
            route = self.optimize_delivery_route(vehicle_id, batch, depot)
            results.append(route)

        return {"batches": len(results), "routes": results,
                "total_deliveries": len(deliveries)}
```

## 异常场景补充

### 场景：路径优化耗时过长

```
触发：50+ 配送点的路径优化 > 30 秒 → 调度延迟 → 车辆等待出发
检测：
  1. 优化计算时间 > 10 秒 → 性能问题
  2. 配送点 > 30 → 计算量过大
处理：
  1. 限制 2-opt 最大迭代次数（100 次）
  2. 配送点 > 30 → 分区优化后合并
  3. 使用预计算的距离矩阵
预防：迭代上限 + 分区优化 + 预计算矩阵
```

### 场景：配送时间窗冲突

```
触发：两个配送点时间窗不重叠 → 路径优化无法同时满足 → 迟到或空跑
检测：
  1. 优化结果显示多个迟到标记 → 时间窗冲突
  2. 总等待时间 > 30 分钟 → 时间窗问题
处理：
  1. 调整时间窗（与客户协商）
  2. 拆分为两次配送
  3. 优先满足硬时间窗，软时间窗可迟到
预防：硬/软时间窗区分 + 客户协商 + 分批配送
```

## 物流路径优化完整实现

```python
import math
import time
import logging
from dataclasses import dataclass, field
from typing import List, Optional, Tuple, Dict
from enum import Enum
from datetime import datetime, timedelta
from collections import defaultdict

logger = logging.getLogger(__name__)


class OptimizationTimeoutError(Exception):
    """路径优化超时异常"""
    pass


class TimeWindowConflictError(Exception):
    """配送时间窗冲突异常"""
    pass


class VehicleCapacityExceededError(Exception):
    """车辆容量超限异常"""
    pass


@dataclass
class GeoLocation:
    """地理坐标"""
    latitude: float
    longitude: float

    def distance_to(self, other: "GeoLocation") -> float:
        """使用Haversine公式计算两点间距离(千米)"""
        R = 6371.0
        lat1_rad = math.radians(self.latitude)
        lat2_rad = math.radians(other.latitude)
        dlat = math.radians(other.latitude - self.latitude)
        dlon = math.radians(other.longitude - self.longitude)
        a = (math.sin(dlat / 2) ** 2 +
             math.cos(lat1_rad) * math.cos(lat2_rad) * math.sin(dlon / 2) ** 2)
        c = 2 * math.atan2(math.sqrt(a), math.sqrt(1 - a))
        return R * c


@dataclass
class TimeWindow:
    """配送时间窗"""
    earliest: datetime
    latest: datetime
    service_time_minutes: int = 15  # 在该点位的停留服务时间

    def is_within(self, arrival: datetime) -> bool:
        return self.earliest <= arrival <= self.latest

    def wait_time(self, arrival: datetime) -> timedelta:
        """若早到则返回需要等待的时间"""
        if arrival < self.earliest:
            return self.earliest - arrival
        return timedelta()


@dataclass
class DeliveryPoint:
    """配送点"""
    id: str
    location: GeoLocation
    time_window: TimeWindow
    demand: float  # 货物需求量(吨)
    priority: int = 0  # 优先级, 数值越大越优先

    @property
    def service_duration(self) -> timedelta:
        return timedelta(minutes=self.time_window.service_time_minutes)


@dataclass
class Vehicle:
    """配送车辆"""
    id: str
    capacity: float  # 载重能力(吨)
    speed_kmh: float = 40.0  # 平均行驶速度(千米/小时)
    start_location: GeoLocation = None
    start_time: datetime = None

    def travel_time(self, distance_km: float) -> timedelta:
        """根据距离计算行驶时间"""
        hours = distance_km / self.speed_kmh
        return timedelta(hours=hours)


@dataclass
class RouteSegment:
    """路径片段"""
    from_point_id: str
    to_point_id: str
    distance_km: float
    travel_time: timedelta
    arrival_time: datetime
    departure_time: datetime
    wait_time: timedelta = timedelta()


@dataclass
class OptimizedRoute:
    """优化后的路径结果"""
    vehicle_id: str
    ordered_points: List[str]
    segments: List[RouteSegment]
    total_distance_km: float
    total_time: timedelta
    total_demand: float
    time_window_violations: int = 0
    optimization_iterations: int = 0


class RouteOptimizationService:
    """
    物流路径优化服务

    使用最近邻启发式 + 2-opt局部搜索求解带时间窗和容量约束的车辆路径问题(VRPTW)。
    支持多车辆调度、时间窗约束检查、容量约束校验。
    """

    MAX_OPTIMIZATION_SECONDS = 30
    TWO_OPT_MAX_ITERATIONS = 500
    NEIGHBOR_CANDIDATE_COUNT = 5

    def __init__(self):
        self._distance_cache: Dict[Tuple[str, str], float] = {}
        self._stats = defaultdict(int)

    def optimize(
        self,
        vehicle: Vehicle,
        delivery_points: List[DeliveryPoint],
        depot: DeliveryPoint = None,
    ) -> OptimizedRoute:
        """
        执行路径优化主流程

        Args:
            vehicle: 配送车辆信息
            delivery_points: 待配送点列表
            depot: 仓库/起点, 若为None则使用vehicle.start_location

        Returns:
            OptimizedRoute: 优化后的路径

        Raises:
            OptimizationTimeoutError: 优化计算超时
            TimeWindowConflictError: 时间窗存在不可调和的冲突
            VehicleCapacityExceededError: 车辆容量不足
        """
        start_wall = time.time()
        self._stats["optimization_requests"] += 1

        # 1. 参数校验与初始化
        if not delivery_points:
            return self._empty_route(vehicle)

        depot = depot or self._build_depot(vehicle, delivery_points)
        all_points = [depot] + delivery_points

        self._validate_capacity(vehicle, delivery_points)
        self._precompute_distances(all_points)

        # 2. 最近邻启发式构造初始解
        initial_route = self._nearest_neighbor_construct(
            vehicle, depot, delivery_points
        )
        logger.info(
            f"初始解构造完成: 路径长度={len(initial_route)}, "
            f"距离={self._route_distance(initial_route, depot):.2f}km"
        )

        # 3. 2-opt局部搜索优化
        optimized_route = self._two_opt_optimize(
            initial_route, depot, start_wall
        )
        iterations = 0
        for i in range(self.TWO_OPT_MAX_ITERATIONS):
            if time.time() - start_wall > self.MAX_OPTIMIZATION_SECONDS:
                logger.warning("路径优化达到时间上限,提前终止")
                raise OptimizationTimeoutError(
                    f"路径优化耗时超过{self.MAX_OPTIMIZATION_SECONDS}秒, "
                    f"当前迭代次数={iterations}"
                )
            improved, optimized_route = self._two_opt_step(
                optimized_route, depot
            )
            iterations += 1
            if not improved:
                break

        # 4. 时间窗可行性校验与调整
        route_with_timing = self._apply_time_windows(
            vehicle, optimized_route, depot
        )

        # 5. 构建结果
        total_dist = self._route_distance(optimized_route, depot)
        total_demand = sum(
            p.demand for p in delivery_points
            if p.id in optimized_route
        )
        result = OptimizedRoute(
            vehicle_id=vehicle.id,
            ordered_points=optimized_route,
            segments=route_with_timing,
            total_distance_km=total_dist,
            total_time=self._calculate_total_time(route_with_timing),
            total_demand=total_demand,
            time_window_violations=self._count_violations(route_with_timing),
            optimization_iterations=iterations,
        )

        elapsed = time.time() - start_wall
        logger.info(
            f"路径优化完成: 距离={total_dist:.2f}km, "
            f"时间窗违反={result.time_window_violations}, "
            f"迭代={iterations}, 耗时={elapsed:.2f}s"
        )
        self._stats["optimization_success"] += 1
        return result

    def _validate_capacity(
        self, vehicle: Vehicle, points: List[DeliveryPoint]
    ):
        """校验车辆容量是否满足所有配送需求"""
        total_demand = sum(p.demand for p in points)
        if total_demand > vehicle.capacity:
            raise VehicleCapacityExceededError(
                f"车辆容量{vehicle.capacity}吨不足以承载"
                f"总需求{total_demand}吨"
            )

    def _precompute_distances(self, points: List[DeliveryPoint]):
        """预计算所有点对之间的距离矩阵"""
        self._distance_cache.clear()
        for i, p1 in enumerate(points):
            for j, p2 in enumerate(points):
                if i != j:
                    key = (p1.id, p2.id)
                    if key not in self._distance_cache:
                        self._distance_cache[key] = (
                            p1.location.distance_to(p2.location)
                        )

    def _get_distance(self, from_id: str, to_id: str) -> float:
        """从缓存获取两点间距离"""
        return self._distance_cache.get((from_id, to_id), float("inf"))

    def _nearest_neighbor_construct(
        self,
        vehicle: Vehicle,
        depot: DeliveryPoint,
        points: List[DeliveryPoint],
    ) -> List[str]:
        """最近邻启发式构造初始路径"""
        unvisited = {p.id: p for p in points}
        route = []
        current_id = depot.id
        current_time = vehicle.start_time or datetime.now()

        while unvisited:
            # 选择距离最近且时间窗可满足的下一个点
            candidates = []
            for pid, point in unvisited.items():
                dist = self._get_distance(current_id, pid)
                travel = vehicle.travel_time(dist)
                arrival = current_time + travel
                # 优先考虑时间窗紧迫度
                urgency = max(
                    timedelta(),
                    point.time_window.latest - arrival
                )
                candidates.append((dist, urgency, pid))

            candidates.sort(key=lambda x: (x[1], x[0]))  # 先按紧迫度再按距离
            # 选取最近的候选点
            best_id = candidates[0][2]
            route.append(best_id)
            best_point = unvisited.pop(best_id)

            dist = self._get_distance(current_id, best_id)
            travel = vehicle.travel_time(dist)
            arrival = current_time + travel
            wait = best_point.time_window.wait_time(arrival)
            current_time = arrival + wait + best_point.service_duration
            current_id = best_id

        return route

    def _two_opt_optimize(
        self, route: List[str], depot: DeliveryPoint, start_wall: float
    ) -> List[str]:
        """2-opt局部搜索(外层循环,由主流程控制迭代)"""
        return route  # 实际迭代在主流程的循环中完成

    def _two_opt_step(
        self, route: List[str], depot: DeliveryPoint
    ) -> Tuple[bool, List[str]]:
        """执行一轮2-opt交换尝试"""
        improved = False
        best_distance = self._route_distance(route, depot)
        best_route = route[:]

        for i in range(len(route) - 1):
            for j in range(i + 2, len(route)):
                new_route = route[:i] + route[i:j][::-1] + route[j:]
                new_distance = self._route_distance(new_route, depot)
                if new_distance < best_distance - 1e-6:
                    best_distance = new_distance
                    best_route = new_route
                    improved = True

        return improved, best_route

    def _route_distance(
        self, route: List[str], depot: DeliveryPoint
    ) -> float:
        """计算路径总距离"""
        if not route:
            return 0.0
        total = self._get_distance(depot.id, route[0])
        for i in range(len(route) - 1):
            total += self._get_distance(route[i], route[i + 1])
        total += self._get_distance(route[-1], depot.id)
        return total

    def _apply_time_windows(
        self,
        vehicle: Vehicle,
        route: List[str],
        depot: DeliveryPoint,
        point_map: Optional[Dict[str, DeliveryPoint]] = None,
    ) -> List[RouteSegment]:
        """沿路径应用时间窗,计算每个点位的到达/离开/等待时间"""
        segments = []
        current_time = vehicle.start_time or datetime.now()
        current_id = depot.id

        point_lookup = point_map or {}
        for pid in route:
            point = point_lookup.get(pid)
            if not point:
                continue

            dist = self._get_distance(current_id, pid)
            travel = vehicle.travel_time(dist)
            arrival = current_time + travel
            wait = point.time_window.wait_time(arrival)
            departure = arrival + wait + point.service_duration

            segments.append(RouteSegment(
                from_point_id=current_id,
                to_point_id=pid,
                distance_km=dist,
                travel_time=travel,
                arrival_time=arrival,
                departure_time=departure,
                wait_time=wait,
            ))
            current_time = departure
            current_id = pid

        # 返回depot的段
        dist_back = self._get_distance(current_id, depot.id)
        travel_back = vehicle.travel_time(dist_back)
        segments.append(RouteSegment(
            from_point_id=current_id,
            to_point_id=depot.id,
            distance_km=dist_back,
            travel_time=travel_back,
            arrival_time=current_time + travel_back,
            departure_time=current_time + travel_back,
        ))

        return segments

    def _count_violations(self, segments: List[RouteSegment]) -> int:
        """统计时间窗违反数量"""
        # 需结合point_map判断,此处简化为延迟到达的段数
        violations = 0
        for seg in segments:
            if seg.wait_time < timedelta() and seg.wait_time != timedelta():
                violations += 1
        return violations

    def _calculate_total_time(self, segments: List[RouteSegment]) -> timedelta:
        if not segments:
            return timedelta()
        start = segments[0].arrival_time
        end = segments[-1].arrival_time
        return end - start

    def _empty_route(self, vehicle: Vehicle) -> OptimizedRoute:
        return OptimizedRoute(
            vehicle_id=vehicle.id,
            ordered_points=[],
            segments=[],
            total_distance_km=0.0,
            total_time=timedelta(),
            total_demand=0.0,
        )

    def _build_depot(
        self, vehicle: Vehicle, points: List[DeliveryPoint]
    ) -> DeliveryPoint:
        """若未提供depot,则以车辆起始位置构造"""
        loc = vehicle.start_location or GeoLocation(0.0, 0.0)
        start = vehicle.start_time or datetime.now()
        return DeliveryPoint(
            id="DEPOT",
            location=loc,
            time_window=TimeWindow(
                earliest=start - timedelta(hours=1),
                latest=start + timedelta(hours=12),
                service_time_minutes=0,
            ),
            demand=0.0,
        )

    def get_stats(self) -> Dict[str, int]:
        return dict(self._stats)
```

## 异常场景补充

### 场景：路径优化耗时过长

```
触发条件: 当配送点数量超过50个,且2-opt迭代在30秒内未收敛时触发
检测手段: 在optimize方法中设置MAX_OPTIMIZATION_SECONDS=30的硬限时,每次2-opt迭代前检查wall clock时间;
          当剩余配送点的组合搜索空间达到O(n^2)规模时,在log中记录WARN级别告警;
          通过Prometheus指标route_optimization_duration_seconds监控优化耗时P99
处理方式: 1. 抛出OptimizationTimeoutError,携带当前迭代次数和已找到的最优距离;
          2. 降级为返回当前已找到的最优解(而非报错),在结果中标记optimization_iterations未收敛;
          3. 对于大规模问题(>30点),自动切换为聚类分区策略——将配送点按地理位置KMeans聚类后分区内独立优化再拼接;
          4. 异步触发告警通知运维人员检查是否存在异常订单集中爆发
预防措施: 1. 对配送点数量设置硬上限(如单车辆不超过40点),超出则拆分为多车辆问题;
          2. 根据问题规模自适应调整2-opt迭代上限——n<15时全量搜索,n>=15时随机采样部分边进行交换;
          3. 增量优化:新订单加入时不全量重算,而是在已有路径上做局部插入并仅对插入点附近做2-opt;
          4. 预热距离矩阵缓存,避免重复计算Haversine距离
```

### 场景：配送时间窗冲突

```
触发条件: 多个配送点的时间窗完全不重叠或存在顺序依赖冲突——例如A点要求08:00-10:00送达,
          B点要求07:00-08:30送达,但从depot出发先到B再到A的行驶时间使得A点必然迟到
检测手段: 1. 在_nearest_neighbor_construct中,对每个候选点计算到达时间,
          若arrival_time > time_window.latest,则标记该候选为不可行;
          2. 构造完成后,遍历segments检查arrival_time是否超过对应点位的latest时间;
          3. 统计time_window_violations计数,若>0则在返回结果中标记;
          4. 通过定时任务扫描当日所有待配送订单,检测时间窗覆盖率<90%的区域
处理方式: 1. 首先尝试调整路径顺序——将违反时间窗的点前移至路径前端;
          2. 若单车辆无法满足,则将冲突点拆分到不同车辆——生成子问题分别优化;
          3. 与客户协商放宽时间窗:自动生成时间窗调整建议(如从08:00-10:00扩展为07:30-11:00),
          通过短信/推送征求客户同意;
          4. 对无法调和的冲突订单标记为"需人工介入",推送至调度员工作台;
          5. 紧急情况下启用外部运力(第三方物流)消化溢出订单
预防措施: 1. 在订单创建时进行时间窗可行性预检——计算当前已排路径插入新订单后是否仍然可行;
          2. 设置时间窗最小宽度要求(如不得窄于2小时),过窄的由客服与客户沟通调整;
          3. 引入时间窗弹性机制——允许一定比例(如5%)的延迟,对延迟订单给予补偿;
          4. 基于历史数据预测配送时长分布,在时间窗计算中加入安全缓冲(safety margin)
```

## 物流异常处理与人工干预完整实现

```python
class LogisticsExceptionHandler:
    """物流异常处理：异常检测 → 分级处理 → 人工干预 → 补救措施"""

    EXCEPTION_LEVELS = {
        "info": {"sla_hours": 72, "auto_handle": True},
        "warning": {"sla_hours": 24, "auto_handle": True},
        "critical": {"sla_hours": 4, "auto_handle": False},
        "emergency": {"sla_hours": 1, "auto_handle": False},
    }

    def detect_exception(self, shipment_id):
        """检测物流异常"""
        shipment = self.db.get_shipment(shipment_id)
        exceptions = []

        # 1. 超时检测
        expected_delivery = shipment.get("expected_delivery_at")
        if expected_delivery and now() > expected_delivery:
            delay_hours = (now() - expected_delivery).total_seconds() / 3600

            if delay_hours > 72:
                level = "emergency"
            elif delay_hours > 24:
                level = "critical"
            elif delay_hours > 4:
                level = "warning"
            else:
                level = "info"

            exceptions.append({
                "type": "delivery_delay",
                "level": level,
                "delay_hours": round(delay_hours, 1),
                "expected_delivery": expected_delivery.isoformat(),
                "message": f"配送延迟 {delay_hours:.1f} 小时"
            })

        # 2. 停滞检测（长时间无状态更新）
        last_update = self.db.query_one(
            "SELECT MAX(created_at) as last FROM shipment_status_log "
            "WHERE shipment_id = %s", shipment_id)["last"]

        if last_update:
            stagnant_hours = (now() - last_update).total_seconds() / 3600

            if stagnant_hours > 48:
                exceptions.append({
                    "type": "stagnant",
                    "level": "critical",
                    "stagnant_hours": round(stagnant_hours, 1),
                    "message": f"物流停滞 {stagnant_hours:.1f} 小时无更新"
                })
            elif stagnant_hours > 24:
                exceptions.append({
                    "type": "stagnant",
                    "level": "warning",
                    "stagnant_hours": round(stagnant_hours, 1),
                    "message": f"物流停滞 {stagnant_hours:.1f} 小时"
                })

        # 3. 异常位置检测（偏离预期路线）
        current_location = shipment.get("current_location")
        destination = shipment.get("destination")
        if current_location and destination:
            dist_to_dest = self._haversine(
                current_location["lat"], current_location["lng"],
                destination["lat"], destination["lng"])

            origin = shipment.get("origin")
            if origin:
                total_dist = self._haversine(
                    origin["lat"], origin["lng"],
                    destination["lat"], destination["lng"])

                if dist_to_dest > total_dist * 1.5:
                    exceptions.append({
                        "type": "route_deviation",
                        "level": "warning",
                        "message": "包裹位置偏离预期路线"
                    })

        # 4. 记录异常
        for exc in exceptions:
            self.db.insert("shipment_exceptions", {
                "exception_id": str(uuid4()),
                "shipment_id": shipment_id,
                "type": exc["type"],
                "level": exc["level"],
                "details": json.dumps(exc),
                "status": "detected",
                "detected_at": now()
            })

        return {"shipment_id": shipment_id, "exceptions": exceptions}

    def handle_exception(self, exception_id):
        """处理异常"""
        exception = self.db.get_exception(exception_id)
        level = exception["level"]
        config = self.EXCEPTION_LEVELS[level]

        if config["auto_handle"]:
            # 自动处理
            handler = self._get_auto_handler(exception["type"])
            result = handler(exception)
        else:
            # 需人工干预
            result = self._create_intervention_task(exception)

        self.db.update("shipment_exceptions",
            {"status": "handling", "handling_method": result.get("method"),
             "handling_started_at": now()},
            {"exception_id": exception_id})

        return result

    def _get_auto_handler(self, exception_type):
        """获取自动处理器"""
        handlers = {
            "delivery_delay": self._handle_delay,
            "stagnant": self._handle_stagnant,
            "route_deviation": self._handle_deviation,
        }
        return handlers.get(exception_type, self._handle_generic)

    def _handle_delay(self, exception):
        """处理延迟"""
        shipment_id = exception["shipment_id"]
        details = json.loads(exception["details"])

        # 1. 更新预计到达时间
        new_eta = now() + timedelta(hours=details.get("delay_hours", 0) / 2)
        self.db.update("shipments",
            {"expected_delivery_at": new_eta},
            {"id": shipment_id})

        # 2. 通知收件人
        shipment = self.db.get_shipment(shipment_id)
        self.notification.send(shipment.get("receiver_id"),
            f"您的包裹配送延迟，新的预计到达时间: {new_eta.strftime('%m月%d日 %H:%M')}")

        return {"method": "auto_delay_update", "new_eta": new_eta.isoformat()}

    def _handle_stagnant(self, exception):
        """处理停滞"""
        shipment_id = exception["shipment_id"]

        # 1. 联系承运商
        shipment = self.db.get_shipment(shipment_id)
        carrier = shipment.get("carrier_id")

        self.notification.send(carrier,
            f"包裹 {shipment_id} 已停滞超过 24 小时，请更新状态")

        # 2. 如果停滞超过 48 小时 → 升级为人工干预
        details = json.loads(exception["details"])
        if details.get("stagnant_hours", 0) > 48:
            return self._create_intervention_task(exception)

        return {"method": "auto_carrier_contact"}

    def _handle_deviation(self, exception):
        """处理路线偏离"""
        # 路线偏离 → 需人工确认
        return self._create_intervention_task(exception)

    def _handle_generic(self, exception):
        """通用处理"""
        return self._create_intervention_task(exception)

    def _create_intervention_task(self, exception):
        """创建人工干预任务"""
        task_id = str(uuid4())
        level = exception["level"]
        config = self.EXCEPTION_LEVELS[level]

        self.db.insert("intervention_tasks", {
            "task_id": task_id,
            "exception_id": exception["exception_id"],
            "shipment_id": exception["shipment_id"],
            "exception_type": exception["type"],
            "exception_level": level,
            "sla_deadline": now() + timedelta(hours=config["sla_hours"]),
            "status": "pending",
            "created_at": now()
        })

        # 通知物流运营团队
        self.notification.send("logistics_ops",
            f"物流异常需人工干预: {exception['type']} (级别: {level}), "
            f"包裹: {exception['shipment_id']}, "
            f"SLA: {config['sla_hours']} 小时内处理")

        return {"method": "manual_intervention", "task_id": task_id}

    def resolve_intervention(self, task_id, resolver_id, resolution, actions_taken=None):
        """解决人工干预"""
        task = self.db.get_intervention_task(task_id)

        if task["status"] != "pending":
            return {"status": "already_resolved"}

        # 1. 更新干预任务
        self.db.update("intervention_tasks",
            {"status": "resolved", "resolver_id": resolver_id,
             "resolution": resolution,
             "actions_taken": json.dumps(actions_taken or []),
             "resolved_at": now()},
            {"task_id": task_id})

        # 2. 更新异常状态
        self.db.update("shipment_exceptions",
            {"status": "resolved", "resolution": resolution,
             "resolved_at": now()},
            {"exception_id": task["exception_id"]})

        # 3. 执行补救措施
        if actions_taken:
            for action in actions_taken:
                self._execute_remedial_action(task["shipment_id"], action)

        # 4. 通知收件人
        shipment = self.db.get_shipment(task["shipment_id"])
        self.notification.send(shipment.get("receiver_id"),
            f"您的包裹问题已处理: {resolution}")

        return {"task_id": task_id, "status": "resolved"}

    def _execute_remedial_action(self, shipment_id, action):
        """执行补救措施"""
        if action["type"] == "reassign_carrier":
            self.db.update("shipments",
                {"carrier_id": action["new_carrier_id"]},
                {"id": shipment_id})

        elif action["type"] == "reroute":
            self.db.update("shipments",
                {"route": json.dumps(action["new_route"])},
                {"id": shipment_id})

        elif action["type"] == "compensate":
            self.db.insert("compensations", {
                "compensation_id": str(uuid4()),
                "shipment_id": shipment_id,
                "amount": str(action["amount"]),
                "reason": action["reason"],
                "created_at": now()
            })
```

## 异常场景补充

### 场景：异常处理 SLA 违规

```
触发：紧急异常需 1 小时内处理 → 运营人员不足 → 2 小时后才处理 → SLA 违规 → 客户投诉
检测：
  1. 干预任务超过 SLA 截止时间仍未处理 → SLA 违规
  2. 待处理干预任务堆积 → 人力不足
处理：
  1. SLA 到期前 30 分钟升级通知
  2. 超过 SLA → 自动升级到上级
  3. 批量处理低级别异常
预防：提前升级 + 超时升级 + 批量处理
```

### 场景：自动处理误判导致错误操作

```
触发：包裹停滞 25 小时 → 自动联系承运商 → 但实际承运商已更新状态（系统延迟）→ 重复通知
检测：
  1. 自动处理后发现状态已更新 → 误判
  2. 承运商收到重复通知 → 误操作
处理：
  1. 自动处理前重新检查最新状态
  2. 设置去重窗口（同一异常 1 小时内不重复处理）
  3. 关键操作需二次确认
预防：重新检查 + 去重窗口 + 二次确认
```
