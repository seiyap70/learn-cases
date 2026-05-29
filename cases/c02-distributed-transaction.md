# C02: 分布式事务与订单一致性

## 业务场景

某综合电商平台，用户下单购买一件商品，一个看似简单的"下单→支付"流程，在微服务架构下涉及 5 个独立服务的协调：

1. **订单服务**：创建订单记录，状态为"待支付"
2. **库存服务**：扣减商品库存
3. **支付服务**：处理用户支付（对接支付宝/微信）
4. **物流服务**：生成发货预通知
5. **积分服务**：扣减用户使用的积分抵扣

**已知条件：**
- 日订单量 100 万，峰值 5000 QPS
- 用户支付成功后，订单状态必须变为"已支付"，库存必须扣减，积分必须消耗
- 如果支付失败，库存和积分必须回退，订单状态变为"已取消"
- 库存扣减后如果支付超时（用户未在 30 分钟内支付），库存需要自动释放
- 积分抵扣金额可以是 0（用户未使用积分），此时积分服务不需要参与
- 服务间调用存在偶发超时（概率约 0.1%），即每秒约 5 次超时

**为什么这是一个难题？**

在单体应用中，这些操作可以在一个数据库事务中完成：
```sql
BEGIN;
INSERT INTO orders ...;
UPDATE inventory SET stock = stock - 1 ...;
UPDATE points SET balance = balance - 100 ...;
COMMIT;
```

但微服务架构下，5 个服务有 5 个独立数据库。跨数据库的事务要么用 XA 协议（2PC），要么用应用层的补偿机制。XA 在 5000 QPS 下性能不足，所以必须自己解决。

**真实的生产事故案例：**
- 某电商平台：支付成功但库存未扣减 → 超卖 2000 件商品 → 客诉 + 赔偿
- 某外卖平台：订单取消但积分未退回 → 用户投诉 → 人工逐笔退回，耗时 3 天
- 某票务平台：支付回调重复 → 用户被扣款两次 → 严重信任危机

## 核心挑战

### 挑战 1：部分失败

5 个服务调用，任何一个都可能失败。最棘手的不是"全部失败"（可以重试），而是"部分成功、部分失败"：

```
库存扣减 ✓  积分扣减 ✓  支付 ✗

此时库存和积分已经扣了，但支付失败了。怎么办？
→ 需要回退库存和积分，这叫"补偿"
```

### 挑战 2：超时不确定性

用户在支付页面点击"确认支付"后，网络中断。30 分钟过去了，我们不知道支付是否成功：

- 可能：用户没付款，交易未发起 → 应该取消订单
- 可能：用户付了款，但回调通知丢失 → 不应该取消订单
- 可能：支付正在处理中 → 应该再等等

如果判定错误，要么用户付了钱但订单被取消（投诉），要么用户没付钱但库存一直被占用（浪费）。

### 挑战 3：业务语义——不是简单的"全部回滚"

不同失败场景需要不同的处理逻辑：
- 扣库存失败 → 直接取消订单，不需要补偿任何操作
- 积分扣减失败 → 释放库存，取消订单
- 支付失败 → 退回积分，释放库存，取消订单
- 物流通知失败 → 不需要取消订单，只需重试物流通知

这不是数据库事务的"全部回滚"，而是**业务语义驱动的补偿**。

### 挑战 4：性能

5 个服务串行同步调用，响应时间 = 网络 RT × 5 + 各服务处理时间：
- 每次网络 RT 约 5ms
- 各服务处理约 10ms
- 总计：5 × 5 + 5 × 10 = 75ms

看起来可以接受，但 5000 QPS 下，75ms 的平均响应意味着需要约 375 个并发线程（Little's Law: L = λ × W = 5000 × 0.075）。如果有慢请求（如支付网关响应慢），线程池可能耗尽。

## 设计约束

- 各服务数据库独立，无法使用数据库层面的分布式事务
- 不可引入 XA 协议（性能无法满足 5000 QPS——XA 的事务协调者会成为瓶颈，且阻塞式锁定在高并发下不可接受）
- 消息队列可用（Kafka/RocketMQ）
- 服务间调用存在偶发超时（概率约 0.1%）

## 请先独立思考（限时 40 分钟）

1. 2PC、TCC、Saga 三种模式，哪种最适合这个场景？列出每种模式在本场景下的具体问题。
2. 画出完整的 Saga 流程图：包括正常路径和每一步失败后的补偿路径。
3. 支付超时后，如何确定支付到底成功了没有？设计一个具体的查询+判断流程。
4. 写出库存服务的"冻结库存"方案的完整 SQL，包括下单、取消、确认三个操作。
5. 设计幂等性方案：如果支付回调被推送了 3 次，你的系统会怎样？

---

## 设计解析

### 方案选型：为什么是 Saga 而非 TCC 或 2PC

**2PC 的问题：**
- 协调者（Transaction Coordinator）是单点——如果协调者在 Prepare 阶段后宕机，所有参与者持有的锁无法释放
- 阻塞式协议——Phase 1 中参与者锁定资源，直到 Phase 2 完成。如果参与者很多，锁定时间很长
- 性能：每秒 5000 笔分布式事务，每笔涉及 5 个参与者，协调者成为严重瓶颈
- 结论：**不可行**

**TCC 的问题：**
- 需要每个服务实现 3 个接口（Try/Confirm/Cancel），开发成本是普通接口的 3 倍
- Try 阶段的"资源冻结"在业务上不自然——库存服务需要区分"冻结库存"和"可用库存"，积分服务也需要"冻结积分"
- 实际上 TCC 的 Cancel 和 Saga 的补偿在语义上几乎相同
- TCC 的优势是 Try 阶段就隔离资源，不会出现"扣了库存但支付失败导致库存被占用"的窗口期——但这个窗口期在我们的场景中可以接受（30 分钟支付超时）
- 结论：**开发成本高，收益有限**

**Saga 的优势：**
- 每个服务只需要"正向操作"和"补偿操作"两个接口
- 补偿操作就是业务上的反向操作（释放库存、退回积分），语义自然
- 可以用编排模式（Orchestration），由订单服务统一控制流程
- 结论：**最适合本场景**

**Saga 的风险和应对：**
- 风险：缺乏隔离性——Saga 执行过程中，中间状态对外可见（如库存已扣但订单还未支付）
- 应对：本场景中，30 分钟的支付窗口是业务可接受的（电商本来就是"下单后等待支付"）

### Saga 编排模式：完整的正向+补偿流程

```
订单服务（Saga 编排者）

正常路径：
  Step 1: 创建订单(待支付) ───→ Step 2: 冻结库存 ───→ Step 3: 冻结积分(如有) 
                                                                    │
                                                            Step 4: 发起支付
                                                                    │
                                                          [等待支付回调/超时]
                                                                    │
                                                            Step 5: 支付成功
                                                                    │
                                                           Step 6: 生成发货预通知
                                                                    │
                                                               订单完成 ✓

补偿路径（以 Step 4 支付失败为例）：
  Step 4 失败 ← 逆向补偿 ← 退回积分(Step 3补偿) ← 释放库存(Step 2补偿) ← 取消订单(Step 1补偿)
```

**为什么先冻结库存再支付，而不是先支付再扣库存？**

先冻结库存的好处：
- 如果支付失败，库存立即释放，不会影响其他用户
- 如果先支付再扣库存，可能支付成功了但库存已不足（被其他人抢走），需要退款，用户体验差

代价：
- 冻结库存期间（等待支付的 30 分钟），这部分库存不能卖给其他人。如果大量用户下单但不支付，会浪费库存
- 应对：设置合理的支付超时（30 分钟）+ 库存预扣上限（单用户限购 1 件）

### 编排器状态机的完整设计

**状态定义：**

```
订单创建 → 库存冻结中 → 库存已冻结 → 积分冻结中 → 积分已冻结 → 等待支付
→ 支付中 → 支付成功 → 发货通知中 → 订单完成

任何步骤失败 → 补偿中 → 订单已取消
```

**数据库表设计：**

```sql
-- Saga 执行记录
CREATE TABLE saga_instances (
    saga_id VARCHAR(64) PRIMARY KEY,         -- 全局唯一
    order_id VARCHAR(64) NOT NULL,
    current_step SMALLINT NOT NULL,          -- 当前步骤编号
    saga_status VARCHAR(20) NOT NULL,        -- EXECUTING / COMPENSATING / COMPLETED / CANCELLED
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_order (order_id)
);

-- Saga 步骤记录
CREATE TABLE saga_steps (
    saga_id VARCHAR(64) NOT NULL,
    step_number SMALLINT NOT NULL,
    service_name VARCHAR(32) NOT NULL,       -- order / inventory / points / payment / shipping
    action VARCHAR(32) NOT NULL,             -- create / freeze / pay / notify
    status VARCHAR(20) NOT NULL,             -- PENDING / EXECUTING / COMPLETED / COMPENSATING / COMPENSATED / FAILED
    request_payload JSONB,                   -- 请求参数
    response_payload JSONB,                  -- 响应结果
    retry_count SMALLINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    PRIMARY KEY (saga_id, step_number)
);
```

**编排器执行逻辑：**

```python
class SagaOrchestrator:
    """基于状态机的 Saga 编排器"""

    # 步骤定义：正向操作 → 补偿操作
    STEPS = [
        {"service": "order",     "action": "create",  "compensate": "cancel"},
        {"service": "inventory", "action": "freeze",   "compensate": "unfreeze"},
        {"service": "points",    "action": "freeze",   "compensate": "unfreeze", "optional": True},
        {"service": "payment",   "action": "pay",      "compensate": "refund"},
        {"service": "shipping",  "action": "notify",   "compensate": "cancel_notify"},
    ]

    def execute(self, order_request):
        saga_id = generate_id()
        
        # 1. 初始化 Saga 实例
        self.db.insert_saga(saga_id, order_request.order_id)
        for i, step in enumerate(self.STEPS):
            self.db.insert_step(saga_id, i + 1, step)

        # 2. 逐步执行
        try:
            for i, step in enumerate(self.STEPS):
                # 可选步骤跳过
                if step.get("optional") and not self.should_execute(step, order_request):
                    self.db.update_step_status(saga_id, i + 1, "SKIPPED")
                    continue

                # 执行正向操作
                self.db.update_step_status(saga_id, i + 1, "EXECUTING")
                result = self.call_service(step["service"], step["action"], order_request)
                self.db.update_step_status(saga_id, i + 1, "COMPLETED", response=result)
                self.db.update_saga_step(saga_id, i + 1)

        except ServiceCallFailed as e:
            # 3. 正向操作失败 → 开始补偿
            self.db.update_saga_status(saga_id, "COMPENSATING")
            self.compensate(saga_id, failed_step=i)
            raise

    def compensate(self, saga_id, failed_step):
        """从失败步骤的前一步开始逆向补偿"""
        for i in range(failed_step - 1, -1, -1):  # 逆向遍历
            step = self.STEPS[i]
            step_status = self.db.get_step_status(saga_id, i + 1)

            if step_status not in ("COMPLETED",):
                continue  # 未完成的步骤不需要补偿

            try:
                self.db.update_step_status(saga_id, i + 1, "COMPENSATING")
                self.call_service(step["service"], step["compensate"], saga_id)
                self.db.update_step_status(saga_id, i + 1, "COMPENSATED")
            except Exception:
                # 补偿失败 → 重试
                self.db.update_step_status(saga_id, i + 1, "FAILED")
                self.schedule_compensation_retry(saga_id, i + 1)

        self.db.update_saga_status(saga_id, "CANCELLED")

    def should_execute(self, step, order_request):
        """可选步骤的执行条件"""
        if step["service"] == "points":
            return order_request.points_amount > 0  # 有积分抵扣才执行
        return True
```

### 支付超时不确定性的完整处理

这是本案例最复杂的部分。支付网关（支付宝/微信）的回调通知可能因为网络问题丢失，导致我们不知道支付是否成功。

**完整的状态查询流程：**

```python
class PaymentStatusResolver:
    """解决支付状态的不确定性"""

    def resolve(self, order_id, payment_txn_id):
        """
        查询支付状态并做出决策
        返回: PAID / UNPAID / UNKNOWN
        """
        # 1. 主动查询支付网关
        gateway_status = self.query_payment_gateway(payment_txn_id)

        if gateway_status == "SUCCESS":
            return "PAID"
        elif gateway_status == "FAILED" or gateway_status == "NOT_FOUND":
            return "UNPAID"
        elif gateway_status == "PROCESSING":
            # 支付网关也不确定，需要等待
            pass

        # 2. 第一次查询不确定，等待 30 秒后重试
        time.sleep(30)
        gateway_status = self.query_payment_gateway(payment_txn_id)

        if gateway_status == "SUCCESS":
            return "PAID"
        elif gateway_status in ("FAILED", "NOT_FOUND"):
            return "UNPAID"
        elif gateway_status == "PROCESSING":
            # 第二次仍不确定

            # 3. 等待 60 秒后第三次查询
            time.sleep(60)
            gateway_status = self.query_payment_gateway(payment_txn_id)

            if gateway_status == "SUCCESS":
                return "PAID"
            else:
                # 三次查询后仍不确定
                # 安全起见，判定为未支付（宁可少卖不能多卖）
                # 但先冻结这笔支付，人工复核
                self.flag_for_manual_review(order_id, payment_txn_id)
                return "UNPAID"

    def query_payment_gateway(self, txn_id):
        """
        查询支付网关的状态
        注意：查询本身也可能超时
        """
        try:
            response = self.payment_client.query(txn_id, timeout=5)
            return response.trade_status  # SUCCESS / FAILED / PROCESSING / NOT_FOUND
        except TimeoutError:
            return "QUERY_TIMEOUT"  # 查询超时也视为不确定
```

**支付超时后的完整决策树：**

```
30 分钟无支付回调
  │
  ├─ 查询支付网关
  │   ├─ 状态=SUCCESS → 支付成功 → 继续订单流程
  │   ├─ 状态=FAILED → 支付失败 → 开始补偿（取消订单）
  │   ├─ 状态=NOT_FOUND → 交易未发起 → 开始补偿
  │   └─ 状态=PROCESSING/TIMEOUT → 不确定
  │       │
  │       ├─ 30 秒后重试
  │       │   ├─ SUCCESS → 继续
  │       │   ├─ FAILED → 补偿
  │       │   └─ 仍不确定 → 60 秒后重试
  │       │       ├─ SUCCESS → 继续
  │       │       └─ 仍不确定 → 标记人工复核，按"未支付"处理
  │       │
  │       └─ 注意：判定为"未支付"后，如果后续收到支付成功回调
  │           → 仍需处理（创建订单或退款）
```

**为什么最终判定为"未支付"而非一直等？**

- 一直等 → 库存被无限期冻结，影响其他用户
- 判定为"未支付" → 释放库存，如果后续发现实际已支付，可以给用户退款或重新创建订单
- 原则：**宁可少卖，不能多卖；宁可退款，不能丢钱**

**支付回调迟到的处理：**

```python
class PaymentCallbackHandler:
    def on_payment_success(self, payment_txn_id):
        # 查找对应的 Saga 实例
        saga = self.find_saga_by_payment_txn(payment_txn_id)

        if saga.saga_status == "EXECUTING":
            # 正常情况：Saga 还在等待支付
            self.saga_orchestrator.continue_from_payment(saga.saga_id)

        elif saga.saga_status == "CANCELLED":
            # 异常情况：Saga 已经补偿完成（判定为未支付）
            # 但支付实际成功了 → 需要退款或重新创建订单

            # 方案 A：自动退款
            self.refund(payment_txn_id)
            self.notify_user(saga.order_id, "支付已退回")

            # 方案 B：重新创建订单（如果库存仍充足）
            # 实际生产中通常选方案 A，因为更简单且不依赖库存
```

### 库存冻结方案：完整的 SQL 实现

```sql
-- 商品库存表
CREATE TABLE inventory (
    item_id VARCHAR(64) PRIMARY KEY,
    total_stock INT NOT NULL,           -- 总库存（不变）
    available_stock INT NOT NULL,       -- 可用库存
    frozen_stock INT NOT NULL DEFAULT 0, -- 冻结库存
    CONSTRAINT chk_stock CHECK (available_stock >= 0 AND frozen_stock >= 0)
);

-- 库存冻结记录
CREATE TABLE inventory_freeze_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_id VARCHAR(64) NOT NULL,
    item_id VARCHAR(64) NOT NULL,
    quantity INT NOT NULL,
    status VARCHAR(20) NOT NULL,         -- FROZEN / RELEASED / CONFIRMED
    created_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE KEY uk_order_item (order_id, item_id),  -- 幂等键
    INDEX idx_status (status, created_at)
);
```

**冻结库存（下单时）：**

```sql
-- Step 1: 冻结库存（原子操作，利用 chk_stock 约束保证 available_stock 不为负）
UPDATE inventory 
SET available_stock = available_stock - 1,
    frozen_stock = frozen_stock + 1
WHERE item_id = 'item123' 
  AND available_stock >= 1;

-- 如果 affected_rows = 0，说明库存不足

-- Step 2: 记录冻结日志（幂等）
INSERT INTO inventory_freeze_log (order_id, item_id, quantity, status)
VALUES ('ORD-001', 'item123', 1, 'FROZEN')
ON DUPLICATE KEY UPDATE id = id;  -- 重复请求忽略
```

**释放库存（订单取消时）：**

```sql
-- Step 1: 更新冻结日志状态
UPDATE inventory_freeze_log 
SET status = 'RELEASED' 
WHERE order_id = 'ORD-001' AND item_id = 'item123' AND status = 'FROZEN';

-- 如果 affected_rows = 0，说明已经释放过（幂等）

-- Step 2: 归还库存
UPDATE inventory 
SET available_stock = available_stock + 1,
    frozen_stock = frozen_stock - 1
WHERE item_id = 'item123';
```

**确认库存（支付成功后，冻结转扣减）：**

```sql
-- 不需要修改 inventory 表的数值（frozen_stock 已经从可用库存中扣除）
-- 只需更新冻结日志状态，标记为已确认
UPDATE inventory_freeze_log 
SET status = 'CONFIRMED' 
WHERE order_id = 'ORD-001' AND item_id = 'item123' AND status = 'FROZEN';
```

**为什么不直接用 `available_stock -= 1` 而要区分冻结和可用？**

直接扣减可用库存的问题：
- 如果订单取消，需要 `available_stock += 1` 释放库存
- 但此时 available_stock 可能已经被其他订单占满，+1 后超过 total_stock
- 区分冻结/可用，释放时 `frozen -= 1, available += 1`，不会超过 total_stock

### 幂等性：每个服务的完整实现

幂等性是分布式事务的基石。每个服务必须有可靠的幂等机制。

**通用幂等日志表：**

```sql
CREATE TABLE idempotent_log (
    id VARCHAR(128) PRIMARY KEY,     -- 幂等键（如 orderId + action）
    service_name VARCHAR(32),
    action VARCHAR(32),
    request_hash VARCHAR(64),        -- 请求参数哈希（检测同一幂等键不同参数）
    status VARCHAR(20),              -- PROCESSING / COMPLETED / FAILED
    response JSONB,                  -- 缓存首次执行结果
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

**幂等执行模板：**

```python
class IdempotentExecutor:
    def execute(self, idempotent_key, action_fn, request):
        # 1. 查询幂等日志
        log = self.db.get_idempotent_log(idempotent_key)

        if log:
            if log.status == "COMPLETED":
                # 已完成 → 直接返回缓存的结果
                return log.response
            elif log.status == "PROCESSING":
                # 正在处理中 → 可能是并发重复请求，等待结果
                return self.wait_for_completion(idempotent_key)
            elif log.status == "FAILED":
                # 上次失败 → 允许重试
                pass

        # 2. 记录开始处理
        self.db.insert_idempotent_log(idempotent_key, status="PROCESSING")

        try:
            # 3. 执行业务操作
            result = action_fn(request)

            # 4. 记录成功
            self.db.update_idempotent_log(idempotent_key, status="COMPLETED", response=result)
            return result

        except Exception as e:
            # 5. 记录失败
            self.db.update_idempotent_log(idempotent_key, status="FAILED")
            raise
```

**各服务的幂等键设计：**

| 服务 | 操作 | 幂等键 | 说明 |
|------|------|--------|------|
| 订单 | 创建 | `order:{orderId}` | 同一订单不重复创建 |
| 订单 | 取消 | `order-cancel:{orderId}` | 同一订单不重复取消 |
| 库存 | 冻结 | `inventory-freeze:{orderId}:{itemId}` | 同一订单同一商品不重复冻结 |
| 库存 | 释放 | `inventory-release:{orderId}:{itemId}` | 同一订单不重复释放 |
| 积分 | 冻结 | `points-freeze:{orderId}` | 同一订单不重复冻结 |
| 支付 | 发起 | `payment:{paymentTxnId}` | 支付流水号天然幂等 |
| 支付 | 退款 | `payment-refund:{paymentTxnId}` | 同一笔交易不重复退款 |

### 补偿失败的重试与兜底

补偿操作本身也可能失败（如积分服务暂时不可用）。需要重试机制和最终兜底。

```python
class CompensationRetryScheduler:
    """补偿失败的重试调度器"""

    MAX_RETRIES = 3
    RETRY_DELAYS = [30, 120, 600]  # 秒：30秒、2分钟、10分钟

    def schedule_retry(self, saga_id, step_number):
        step = self.db.get_step(saga_id, step_number)
        retry_count = step.retry_count

        if retry_count >= self.MAX_RETRIES:
            # 重试耗尽 → 人工介入
            self.alert_team(f"补偿失败需人工介入: saga={saga_id}, step={step_number}")
            self.db.update_step_status(saga_id, step_number, "NEEDS_MANUAL")
            return

        # 延迟重试
        delay = self.RETRY_DELAYS[retry_count]
        self.db.update_step_retry_count(saga_id, step_number, retry_count + 1)
        
        # 使用延迟消息队列调度重试
        self.mq.send_delay_message(
            topic="saga-compensation-retry",
            body={"saga_id": saga_id, "step_number": step_number},
            delay_seconds=delay
        )

    def on_retry_message(self, message):
        saga_id = message["saga_id"]
        step_number = message["step_number"]
        
        step = self.db.get_step(saga_id, step_number)
        step_def = self.STEPS[step_number - 1]

        try:
            self.call_service(step_def["service"], step_def["compensate"], saga_id)
            self.db.update_step_status(saga_id, step_number, "COMPENSATED")
        except Exception:
            self.schedule_retry(saga_id, step_number)
```

**人工介入流程：**
1. 补偿失败告警发送到运维团队（Slack/钉钉）
2. 运维查看 saga_steps 表中 NEEDS_MANUAL 状态的步骤
3. 确认业务数据状态（如积分是否已退回）
4. 手动执行补偿操作或标记为已补偿
5. 记录处理结果

### 异常场景完整演练

**场景 1：扣库存成功，扣积分失败**

```
Step 1: 创建订单 ✓
Step 2: 冻结库存 ✓
Step 3: 冻结积分 ✗ (积分服务超时)

补偿流程：
  释放库存 (Step 2 补偿) ✓
  取消订单 (Step 1 补偿) ✓

结果：订单已取消，库存已释放，积分未扣
```

**场景 2：积分扣了，支付超时且判定为未支付**

```
Step 1: 创建订单 ✓
Step 2: 冻结库存 ✓
Step 3: 冻结积分 ✓
Step 4: 等待支付 → 30 分钟超时 → 查询网关 → 未支付

补偿流程：
  退回积分 (Step 3 补偿) ✓
  释放库存 (Step 2 补偿) ✓
  取消订单 (Step 1 补偿) ✓

结果：订单已取消，库存和积分已释放
```

**场景 3：支付成功回调到达两次**

```
第一次回调：查询 Saga 状态 = EXECUTING → 继续流程
第二次回调：查询 Saga 状态 = COMPLETED → 忽略（幂等）

关键：支付回调处理时先查询 Saga 状态，再决定行为
```

**场景 4：补偿过程中积分退回失败**

```
Step 3 补偿（退回积分）✗ → 重试 1 (30秒后) ✗ → 重试 2 (2分钟后) ✓

如果三次重试都失败 → 人工介入

在此期间，积分处于"已冻结但未退回"状态：
  - 用户看不到这部分积分（已冻结）
  - 但积分确实属于该用户，只是没有退回到可用余额
  - 人工介入后补退即可
```

**场景 5：编排器在 Step 3 后宕机**

```
编排器重启后：
  1. 从数据库读取 Saga 状态：saga_status = EXECUTING, current_step = 3
  2. 检查各步骤状态：Step 1 ✓, Step 2 ✓, Step 3 ✓, Step 4 PENDING
  3. 从 Step 4 继续执行

关键：编排器是无状态的，所有状态持久化在数据库中
```

### 性能分析

**同步 vs 异步的选择：**

| 步骤 | 方式 | 延迟 | 理由 |
|------|------|------|------|
| 创建订单 | 同步 | 10ms | 需要立即返回 orderId |
| 冻结库存 | 同步 | 15ms | 需要确认库存充足 |
| 冻结积分 | 同步 | 10ms | 需要确认积分充足 |
| 发起支付 | 异步 | 5ms | 支付是长时间操作，提交后等待回调 |
| 生成发货通知 | 异步 | 5ms | 不影响用户，后台执行 |

**同步部分的延迟：** 10 + 15 + 10 + 5 = 40ms（创建→冻结→冻结→提交支付）

**异步部分：** 用户在支付页面等待，不占用后端线程。

**5000 QPS 下的线程需求：** 40ms 响应时间 → 需要 200 个并发线程（5000 × 0.04）。完全可接受。

## 常见陷阱（深度分析）

### 陷阱 1：用 2PC 强一致性

2PC 在 5000 QPS 下的性能瓶颈：
- Phase 1：5 个服务都要执行操作并锁定资源 → 总耗时约 50ms
- Phase 2：5 个服务提交 → 总耗时约 25ms
- 单笔事务 75ms，5000 QPS 需要 375 个并发连接
- 但 2PC 的锁持有时间更长（Phase 1 到 Phase 2 之间），实际吞吐可能只有 1000-2000 TPS

### 陷阱 2：忽略幂等设计

**具体场景：** 库存冻结操作因网络超时被重试，执行了两次。

无幂等保护：available_stock 被扣了 2 次，frozen_stock 增加了 2 次。
有幂等保护：第二次执行查询到幂等日志已存在，直接返回首次结果。

### 陷阱 3：补偿失败无兜底

如果积分退回永远失败（积分服务的数据库损坏），Saga 将永远处于 COMPENSATING 状态。
必须有：重试 → 人工告警 → 人工补偿 的完整链路。

### 陷阱 4：直接扣可用库存而非冻结

**超卖场景：**
- 商品 stock = 1
- 用户 A 下单 → available_stock = 0
- 用户 A 取消订单 → available_stock = 1
- 但在取消的瞬间，用户 B 也下单 → available_stock = 0
- 此时 A 的取消操作 +1，B 的下单操作 -1 → available_stock = 0（正确）

看起来没问题？但如果 A 和 B 的操作并发执行：
```
线程1(A取消): 读取 available_stock = 0
线程2(B下单): 读取 available_stock = 0
线程1(A取消): available_stock = 0 + 1 = 1
线程2(B下单): available_stock = 1 - 1 = 0 (成功)
```

实际上 A 取消后库存应该回到 1（因为 A 之前占的 1 个已经释放），但 B 拿到的是 A 释放的那个——这在业务上是正确的。

真正的问题出现在：如果 available_stock 已经被其他人占满（available_stock=0, frozen_stock=100, total_stock=100），A 取消时 +1 会让 available_stock 变成 1——这是正确的。但如果直接用 `stock -= 1` 没有 frozen 概念，取消时 `stock += 1` 可能导致 stock 超过 total_stock。

**冻结模式的保证：** `available_stock + frozen_stock = total_stock` 是不变量，释放时 `frozen -= 1, available += 1`，不变量不被打破。

## 延伸思考

- **TCC 改造**：如果改为 TCC 模式，库存服务需要 `TryFreeze`（冻结库存）、`ConfirmDeduct`（确认扣减，将冻结转为真实扣减）、`CancelRelease`（取消冻结）。对比 Saga 的 `Freeze` + `Unfreeze`，TCC 的 Confirm 不需要做任何事（冻结就是真实扣减），Cancel 就是 Unfreeze。实际上两者几乎等价。
- **工作流引擎**：如果引入 Temporal/Zeebe，编排器的状态管理、重试、超时检测都可以由引擎处理，开发量减少约 60%。但引入新组件的运维成本和学习曲线需要权衡。
- **跨公司支付**：支付宝/微信的回调通知有 SLA 但不保证 100% 到达。必须依赖主动查询（`alipay.trade.query`）作为兜底。查询频率不能太高（有调用限制），通常每 30 秒查一次，3 次后判定。