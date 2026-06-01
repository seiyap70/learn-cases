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

### 状态机驱动的 Saga 编排器（持久化实现）

上面的编排器是一个简化版本。生产级的编排器必须是基于状态机的、可持久化的、可恢复的。下面是完整实现：

**状态机定义：**

```python
from enum import Enum
from dataclasses import dataclass
from typing import Optional, Dict, Any, Callable
import logging

logger = logging.getLogger(__name__)


class SagaStatus(Enum):
    """Saga 实例状态"""
    PENDING = "PENDING"               # 已创建，尚未开始
    EXECUTING = "EXECUTING"           # 正在执行正向流程
    WAITING_PAYMENT = "WAITING_PAYMENT"  # 等待支付回调
    COMPENSATING = "COMPENSATING"     # 正在执行补偿流程
    COMPLETED = "COMPLETED"           # 全部完成
    CANCELLED = "CANCELLED"           # 已补偿取消
    SUSPENDED = "SUSPENDED"           # 暂停（需人工介入）


class StepStatus(Enum):
    """Saga 步骤状态"""
    PENDING = "PENDING"
    EXECUTING = "EXECUTING"
    COMPLETED = "COMPLETED"
    FAILED = "FAILED"
    COMPENSATING = "COMPENSATING"
    COMPENSATED = "COMPENSATED"
    COMPENSATION_FAILED = "COMPENSATION_FAILED"
    SKIPPED = "SKIPPED"


# 合法的状态转换
SAGA_TRANSITIONS = {
    SagaStatus.PENDING: [SagaStatus.EXECUTING],
    SagaStatus.EXECUTING: [SagaStatus.WAITING_PAYMENT, SagaStatus.COMPENSATING, SagaStatus.COMPLETED],
    SagaStatus.WAITING_PAYMENT: [SagaStatus.EXECUTING, SagaStatus.COMPENSATING],
    SagaStatus.COMPENSATING: [SagaStatus.CANCELLED, SagaStatus.SUSPENDED],
    SagaStatus.COMPLETED: [],   # 终态
    SagaStatus.CANCELLED: [],    # 终态
    SagaStatus.SUSPENDED: [SagaStatus.COMPENSATING],  # 人工恢复后继续补偿
}


class IllegalStateTransition(Exception):
    """非法状态转换异常"""
    def __init__(self, from_status: SagaStatus, to_status: SagaStatus):
        self.from_status = from_status
        self.to_status = to_status
        super().__init__(f"非法状态转换: {from_status.value} → {to_status.value}")


@dataclass
class SagaStep:
    """Saga 步骤定义"""
    step_number: int
    service_name: str
    action: str
    compensate_action: str
    optional: bool = False


@dataclass
class SagaInstance:
    """Saga 实例"""
    saga_id: str
    order_id: str
    current_step: int
    status: SagaStatus
    context: Dict[str, Any]       # 运行时上下文，存储跨步骤的中间结果
    created_at: str
    updated_at: str
```

**持久化状态机编排器：**

```python
class PersistentSagaOrchestrator:
    """
    基于状态机的持久化 Saga 编排器

    核心设计原则：
    1. 每次状态变更都先持久化到数据库，再执行业务逻辑（WAL 思想）
    2. 编排器本身无状态，重启后从数据库恢复
    3. 状态转换严格遵循状态机定义，防止非法转换
    """

    # 步骤定义
    STEP_DEFINITIONS = [
        SagaStep(1, "order",     "create",  "cancel"),
        SagaStep(2, "inventory", "freeze",  "unfreeze"),
        SagaStep(3, "points",    "freeze",  "unfreeze", optional=True),
        SagaStep(4, "payment",   "pay",     "refund"),
        SagaStep(5, "shipping",  "notify",  "cancel_notify"),
    ]

    def __init__(self, db: "SagaRepository", service_invoker: "ServiceInvoker"):
        self.db = db
        self.invoker = service_invoker

    def start_saga(self, order_request) -> str:
        """启动一个新的 Saga 实例"""
        saga_id = self._generate_saga_id()

        # 1. 持久化 Saga 实例（PENDING 状态）
        instance = SagaInstance(
            saga_id=saga_id,
            order_id=order_request.order_id,
            current_step=0,
            status=SagaStatus.PENDING,
            context={"order_request": order_request.__dict__},
            created_at=self._now(),
            updated_at=self._now(),
        )
        self.db.save_saga_instance(instance)

        # 2. 持久化所有步骤（PENDING 状态）
        for step_def in self.STEP_DEFINITIONS:
            self.db.save_saga_step(saga_id, step_def.step_number,
                                   step_def.service_name, step_def.action,
                                   StepStatus.PENDING)

        # 3. 转换为 EXECUTING 状态并开始执行
        self._transition_saga_status(saga_id, SagaStatus.PENDING, SagaStatus.EXECUTING)
        self._execute_forward(saga_id)

        return saga_id

    def _execute_forward(self, saga_id: str):
        """执行正向流程"""
        instance = self.db.get_saga_instance(saga_id)

        for step_def in self.STEP_DEFINITIONS:
            step_number = step_def.step_number

            # 更新当前步骤
            self.db.update_saga_current_step(saga_id, step_number)

            # 检查步骤状态（可能是恢复后已完成的步骤）
            step_status = self.db.get_step_status(saga_id, step_number)
            if step_status == StepStatus.COMPLETED:
                continue  # 已完成，跳过

            if step_status == StepStatus.SKIPPED:
                continue  # 已跳过，跳过

            # 可选步骤判断
            if step_def.optional and not self._should_execute_step(step_def, instance):
                self.db.update_step_status(saga_id, step_number, StepStatus.SKIPPED)
                continue

            # 先持久化 EXECUTING 状态，再调用服务
            self.db.update_step_status(saga_id, step_number, StepStatus.EXECUTING)
            request_payload = self._build_request_payload(step_def, instance)

            try:
                # 调用服务
                response = self.invoker.invoke(
                    service=step_def.service_name,
                    action=step_def.action,
                    payload=request_payload,
                    timeout=10,  # 10 秒超时
                )

                # 持久化完成状态和响应
                self.db.update_step_status(saga_id, step_number, StepStatus.COMPLETED,
                                           response_payload=response)
                # 更新上下文
                self._update_context(instance, step_def, response)

            except ServiceCallFailed as e:
                logger.error(f"Saga {saga_id} 步骤 {step_number} 执行失败: {e}")
                # 标记步骤失败
                self.db.update_step_status(saga_id, step_number, StepStatus.FAILED,
                                           error_message=str(e))
                # 转入补偿流程
                self._transition_saga_status(saga_id, instance.status, SagaStatus.COMPENSATING)
                self._execute_compensation(saga_id, failed_step=step_number)
                return

            except ServiceCallTimeout as e:
                logger.warning(f"Saga {saga_id} 步骤 {step_number} 超时，进入待确认状态")
                # 超时不等于失败，需要查询实际状态
                actual_status = self._resolve_timeout(saga_id, step_number, step_def)
                if actual_status == StepStatus.COMPLETED:
                    self.db.update_step_status(saga_id, step_number, StepStatus.COMPLETED)
                else:
                    self.db.update_step_status(saga_id, step_number, StepStatus.FAILED)
                    self._transition_saga_status(saga_id, instance.status, SagaStatus.COMPENSATING)
                    self._execute_compensation(saga_id, failed_step=step_number)
                    return

        # 全部步骤完成
        # 如果 Step 4 是支付步骤，需要等待回调
        payment_step = self.STEP_DEFINITIONS[3]  # Step 4
        if payment_step.service_name == "payment":
            self._transition_saga_status(saga_id, SagaStatus.EXECUTING, SagaStatus.WAITING_PAYMENT)
        else:
            self._transition_saga_status(saga_id, SagaStatus.EXECUTING, SagaStatus.COMPLETED)

    def _execute_compensation(self, saga_id: str, failed_step: int):
        """执行补偿流程：从失败步骤的前一步开始逆向补偿"""
        instance = self.db.get_saga_instance(saga_id)

        for step_number in range(failed_step - 1, 0, -1):
            step_status = self.db.get_step_status(saga_id, step_number)

            # 只有已完成的步骤才需要补偿
            if step_status not in (StepStatus.COMPLETED,):
                continue

            step_def = self.STEP_DEFINITIONS[step_number - 1]

            # 持久化 COMPENSATING 状态
            self.db.update_step_status(saga_id, step_number, StepStatus.COMPENSATING)

            try:
                # 调用补偿操作
                compensation_payload = self._build_compensation_payload(step_def, instance)
                self.invoker.invoke(
                    service=step_def.service_name,
                    action=step_def.compensate_action,
                    payload=compensation_payload,
                    timeout=10,
                )
                self.db.update_step_status(saga_id, step_number, StepStatus.COMPENSATED)

            except Exception as e:
                logger.error(f"Saga {saga_id} 步骤 {step_number} 补偿失败: {e}")
                self.db.update_step_status(saga_id, step_number, StepStatus.COMPENSATION_FAILED)
                # 调度重试
                self._schedule_compensation_retry(saga_id, step_number)
                # 继续补偿后续步骤（不因一个步骤的补偿失败而停止整个补偿流程）
                continue

        # 检查是否所有需补偿的步骤都已补偿完成
        has_uncompensated = self._check_uncompensated_steps(saga_id)
        if has_uncompensated:
            self._transition_saga_status(saga_id, SagaStatus.COMPENSATING, SagaStatus.SUSPENDED)
            self._alert_manual_intervention(saga_id, "部分补偿步骤失败，需人工介入")
        else:
            self._transition_saga_status(saga_id, SagaStatus.COMPENSATING, SagaStatus.CANCELLED)

    def _transition_saga_status(self, saga_id: str, from_status: SagaStatus, to_status: SagaStatus):
        """安全的状态转换：校验合法性后持久化"""
        if to_status not in SAGA_TRANSITIONS.get(from_status, []):
            raise IllegalStateTransition(from_status, to_status)

        # 使用乐观锁更新，防止并发冲突
        affected = self.db.update_saga_status_with_version(
            saga_id, from_status, to_status
        )
        if affected == 0:
            raise IllegalStateTransition(from_status, to_status)

        logger.info(f"Saga {saga_id} 状态转换: {from_status.value} → {to_status.value}")

    def on_payment_callback(self, payment_txn_id: str, payment_status: str):
        """支付回调处理"""
        saga = self.db.find_saga_by_payment_txn(payment_txn_id)

        if saga is None:
            logger.error(f"未找到支付流水 {payment_txn_id} 对应的 Saga")
            return

        if saga.status == SagaStatus.WAITING_PAYMENT:
            if payment_status == "SUCCESS":
                # 标记支付步骤完成
                self.db.update_step_status(saga.saga_id, 4, StepStatus.COMPLETED)
                # 继续执行后续步骤
                self._transition_saga_status(saga.saga_id, SagaStatus.WAITING_PAYMENT, SagaStatus.EXECUTING)
                self._execute_forward(saga.saga_id)
            elif payment_status == "FAILED":
                self.db.update_step_status(saga.saga_id, 4, StepStatus.FAILED)
                self._transition_saga_status(saga.saga_id, SagaStatus.WAITING_PAYMENT, SagaStatus.COMPENSATING)
                self._execute_compensation(saga.saga_id, failed_step=4)

        elif saga.status == SagaStatus.CANCELLED:
            # Saga 已取消，但支付实际成功了 → 需要退款
            if payment_status == "SUCCESS":
                logger.warning(f"Saga {saga.saga_id} 已取消但支付成功，自动退款: {payment_txn_id}")
                self._auto_refund(payment_txn_id, saga.saga_id)

        elif saga.status == SagaStatus.COMPLETED:
            # 重复回调，忽略
            logger.info(f"Saga {saga.saga_id} 已完成，忽略重复回调")

    def _resolve_timeout(self, saga_id: str, step_number: int, step_def: SagaStep) -> StepStatus:
        """
        超时后的状态确认：查询下游服务，确认操作是否实际执行成功

        关键：超时 ≠ 失败。服务端可能已经处理成功，只是响应没有回来。
        """
        max_retries = 3
        for attempt in range(max_retries):
            try:
                # 查询下游服务的实际状态
                result = self.invoker.invoke(
                    service=step_def.service_name,
                    action="query_status",
                    payload={"saga_id": saga_id, "step_number": step_number},
                    timeout=5,
                )
                if result.get("status") == "COMPLETED":
                    return StepStatus.COMPLETED
                elif result.get("status") == "NOT_FOUND":
                    return StepStatus.FAILED
                # 仍在处理中，等待重试
                time.sleep(5 * (attempt + 1))
            except Exception:
                time.sleep(5 * (attempt + 1))

        # 无法确认状态，按失败处理（安全优先）
        return StepStatus.FAILED

    def _should_execute_step(self, step_def: SagaStep, instance: SagaInstance) -> bool:
        """判断可选步骤是否需要执行"""
        if step_def.service_name == "points":
            return instance.context.get("order_request", {}).get("points_amount", 0) > 0
        return True

    def _generate_saga_id(self) -> str:
        """生成全局唯一的 Saga ID"""
        import uuid
        return f"SAGA-{uuid.uuid4().hex[:16]}"

    def _now(self) -> str:
        from datetime import datetime
        return datetime.utcnow().isoformat()
```

**SagaRepository 持久化层：**

```python
class SagaRepository:
    """Saga 状态持久化仓库"""

    def save_saga_instance(self, instance: SagaInstance):
        """保存 Saga 实例"""
        sql = """
        INSERT INTO saga_instances (saga_id, order_id, current_step, saga_status, context, created_at, updated_at, version)
        VALUES (%s, %s, %s, %s, %s, %s, %s, 1)
        """
        self._execute(sql, (
            instance.saga_id, instance.order_id, instance.current_step,
            instance.status.value, json.dumps(instance.context),
            instance.created_at, instance.updated_at
        ))

    def update_saga_status_with_version(self, saga_id: str, from_status: SagaStatus, to_status: SagaStatus) -> int:
        """乐观锁更新 Saga 状态，返回影响行数"""
        sql = """
        UPDATE saga_instances
        SET saga_status = %s, updated_at = NOW(), version = version + 1
        WHERE saga_id = %s AND saga_status = %s AND version = %s
        """
        current_version = self._get_version(saga_id)
        affected = self._execute_update(sql, (to_status.value, saga_id, from_status.value, current_version))
        return affected

    def save_saga_step(self, saga_id: str, step_number: int, service_name: str,
                       action: str, status: StepStatus):
        sql = """
        INSERT INTO saga_steps (saga_id, step_number, service_name, action, status, created_at, updated_at)
        VALUES (%s, %s, %s, %s, %s, NOW(), NOW())
        """
        self._execute(sql, (saga_id, step_number, service_name, action, status.value))

    def update_step_status(self, saga_id: str, step_number: int, status: StepStatus,
                           response_payload=None, error_message=None):
        sql = """
        UPDATE saga_steps
        SET status = %s, response_payload = %s, error_message = %s, updated_at = NOW()
        WHERE saga_id = %s AND step_number = %s
        """
        self._execute(sql, (
            status.value,
            json.dumps(response_payload) if response_payload else None,
            error_message,
            saga_id, step_number
        ))

    def get_saga_instance(self, saga_id: str) -> SagaInstance:
        sql = "SELECT * FROM saga_instances WHERE saga_id = %s"
        row = self._query_one(sql, (saga_id,))
        return self._row_to_instance(row)

    def get_step_status(self, saga_id: str, step_number: int) -> StepStatus:
        sql = "SELECT status FROM saga_steps WHERE saga_id = %s AND step_number = %s"
        row = self._query_one(sql, (saga_id, step_number))
        return StepStatus(row["status"])

    def find_saga_by_payment_txn(self, payment_txn_id: str) -> Optional[SagaInstance]:
        """通过支付流水号查找对应的 Saga 实例"""
        sql = """
        SELECT si.* FROM saga_instances si
        JOIN saga_steps ss ON si.saga_id = ss.saga_id
        WHERE ss.service_name = 'payment'
          AND ss.step_number = 4
          AND ss.response_payload->>'txn_id' = %s
        """
        row = self._query_one(sql, (payment_txn_id,))
        return self._row_to_instance(row) if row else None

    def find_executing_sagas(self) -> List[SagaInstance]:
        """查找所有正在执行的 Saga（用于恢复）"""
        sql = """
        SELECT * FROM saga_instances
        WHERE saga_status IN ('EXECUTING', 'COMPENSATING', 'WAITING_PAYMENT')
        """
        rows = self._query_all(sql)
        return [self._row_to_instance(row) for row in rows]
```

### Saga 状态恢复（编排器崩溃恢复）

编排器可能随时崩溃（进程被杀、机器宕机、OOM）。关键保证：崩溃后恢复，Saga 继续执行，不会丢失进度，不会重复执行。

```python
class SagaRecoveryService:
    """
    Saga 状态恢复服务

    触发时机：
    1. 编排器进程启动时（主动扫描未完成的 Saga）
    2. 定时任务周期性扫描（如每 30 秒）
    3. 手动触发恢复

    恢复策略：
    - EXECUTING 状态：从最后一个已完成的步骤之后继续执行
    - COMPENSATING 状态：从最后一个已补偿的步骤之后继续补偿
    - WAITING_PAYMENT 状态：检查支付超时，决定继续等待还是开始补偿
    """

    def __init__(self, db: SagaRepository, orchestrator: PersistentSagaOrchestrator,
                 payment_resolver: "PaymentStatusResolver"):
        self.db = db
        self.orchestrator = orchestrator
        self.payment_resolver = payment_resolver

    def recover_all(self):
        """恢复所有未完成的 Saga"""
        pending_sagas = self.db.find_executing_sagas()
        logger.info(f"发现 {len(pending_sagas)} 个未完成的 Saga 待恢复")

        for saga in pending_sagas:
            try:
                self._recover_one(saga)
            except Exception as e:
                logger.error(f"恢复 Saga {saga.saga_id} 失败: {e}")

    def _recover_one(self, saga: SagaInstance):
        """恢复单个 Saga"""
        logger.info(f"恢复 Saga {saga.saga_id}, 状态: {saga.status.value}, 当前步骤: {saga.current_step}")

        if saga.status == SagaStatus.EXECUTING:
            self._recover_executing(saga)
        elif saga.status == SagaStatus.COMPENSATING:
            self._recover_compensating(saga)
        elif saga.status == SagaStatus.WAITING_PAYMENT:
            self._recover_waiting_payment(saga)

    def _recover_executing(self, saga: SagaInstance):
        """恢复执行中的 Saga"""
        # 找到最后一个已完成的步骤
        last_completed_step = 0
        for step_num in range(1, 6):
            status = self.db.get_step_status(saga.saga_id, step_num)
            if status == StepStatus.COMPLETED:
                last_completed_step = step_num
            elif status == StepStatus.EXECUTING:
                # 步骤正在执行中 → 需要确认下游服务实际状态
                self._verify_and_fix_step_status(saga, step_num)
                # 重新查询状态
                new_status = self.db.get_step_status(saga.saga_id, step_num)
                if new_status == StepStatus.COMPLETED:
                    last_completed_step = step_num
                elif new_status == StepStatus.FAILED:
                    # 步骤实际失败了，需要补偿
                    self.orchestrator._execute_compensation(saga.saga_id, failed_step=step_num)
                    return
                else:
                    # 仍不确定，稍后重试
                    logger.warning(f"Saga {saga.saga_id} 步骤 {step_num} 状态不确定，稍后重试")
                    return
            elif status == StepStatus.PENDING:
                break

        # 从下一个步骤继续执行
        next_step = last_completed_step + 1
        if next_step <= 5:
            # 更新当前步骤编号，然后继续执行
            self.db.update_saga_current_step(saga.saga_id, next_step)
            self.orchestrator._execute_forward(saga.saga_id)
        else:
            # 所有步骤已完成，转为 COMPLETED
            self.orchestrator._transition_saga_status(
                saga.saga_id, SagaStatus.EXECUTING, SagaStatus.COMPLETED
            )

    def _recover_compensating(self, saga: SagaInstance):
        """恢复补偿中的 Saga"""
        # 找到最后一个已补偿的步骤
        last_compensated_step = 0
        for step_num in range(1, 6):
            status = self.db.get_step_status(saga.saga_id, step_num)
            if status == StepStatus.COMPENSATED:
                last_compensated_step = step_num
            elif status == StepStatus.COMPENSATION_FAILED:
                # 补偿失败的步骤，重新尝试
                self._retry_failed_compensation(saga, step_num)

        # 从下一个步骤继续补偿
        # 补偿是逆向的，从最后完成的步骤向前补偿
        self.orchestrator._execute_compensation(saga.saga_id, failed_step=saga.current_step + 1)

    def _recover_waiting_payment(self, saga: SagaInstance):
        """恢复等待支付的 Saga"""
        # 检查是否已超时
        created_at = datetime.fromisoformat(saga.created_at)
        elapsed = (datetime.utcnow() - created_at).total_seconds()

        if elapsed < 30 * 60:  # 未超时（30 分钟）
            # 继续等待，但主动查询一次支付状态
            payment_step_response = self._get_payment_step_response(saga)
            if payment_step_response:
                txn_id = payment_step_response.get("txn_id")
                status = self.payment_resolver.query_payment_gateway(txn_id)
                if status == "SUCCESS":
                    self.orchestrator.on_payment_callback(txn_id, "SUCCESS")
                elif status in ("FAILED", "NOT_FOUND"):
                    self.orchestrator.on_payment_callback(txn_id, "FAILED")
                # PROCESSING → 继续等待
        else:
            # 已超时，执行完整的超时判定流程
            payment_step_response = self._get_payment_step_response(saga)
            if payment_step_response:
                txn_id = payment_step_response.get("txn_id")
                resolution = self.payment_resolver.resolve(saga.order_id, txn_id)
                if resolution == "PAID":
                    self.orchestrator.on_payment_callback(txn_id, "SUCCESS")
                else:
                    self.orchestrator.on_payment_callback(txn_id, "FAILED")

    def _verify_and_fix_step_status(self, saga: SagaInstance, step_number: int):
        """验证步骤实际状态并修正数据库记录"""
        step_def = PersistentSagaOrchestrator.STEP_DEFINITIONS[step_number - 1]
        try:
            result = self.orchestrator.invoker.invoke(
                service=step_def.service_name,
                action="query_status",
                payload={"saga_id": saga.saga_id, "order_id": saga.order_id},
                timeout=5,
            )
            actual_status = result.get("status")
            if actual_status == "COMPLETED":
                self.db.update_step_status(saga.saga_id, step_number, StepStatus.COMPLETED,
                                           response_payload=result)
            elif actual_status == "NOT_FOUND":
                # 下游服务没有执行记录，标记为失败
                self.db.update_step_status(saga.saga_id, step_number, StepStatus.FAILED)
            # 仍在 PROCESSING 则不改状态
        except Exception as e:
            logger.error(f"验证 Saga {saga.saga_id} 步骤 {step_number} 状态时出错: {e}")

    def _retry_failed_compensation(self, saga: SagaInstance, step_number: int):
        """重试补偿失败的步骤"""
        step_def = PersistentSagaOrchestrator.STEP_DEFINITIONS[step_number - 1]
        try:
            compensation_payload = self.orchestrator._build_compensation_payload(step_def, saga)
            self.orchestrator.invoker.invoke(
                service=step_def.service_name,
                action=step_def.compensate_action,
                payload=compensation_payload,
                timeout=10,
            )
            self.db.update_step_status(saga.saga_id, step_number, StepStatus.COMPENSATED)
            logger.info(f"Saga {saga.saga_id} 步骤 {step_number} 补偿重试成功")
        except Exception as e:
            logger.error(f"Saga {saga.saga_id} 步骤 {step_number} 补偿重试仍失败: {e}")

    def _get_payment_step_response(self, saga: SagaInstance) -> Optional[Dict]:
        """获取支付步骤的响应数据"""
        sql = "SELECT response_payload FROM saga_steps WHERE saga_id = %s AND step_number = 4"
        row = self.db._query_one(sql, (saga.saga_id,))
        if row and row["response_payload"]:
            return json.loads(row["response_payload"])
        return None
```

**启动时自动恢复：**

```python
class SagaOrchestratorApplication:
    """编排器应用启动入口"""

    def __init__(self):
        self.db = SagaRepository()
        self.invoker = ServiceInvoker()
        self.orchestrator = PersistentSagaOrchestrator(self.db, self.invoker)
        self.recovery_service = SagaRecoveryService(self.db, self.orchestrator, PaymentStatusResolver())

    def start(self):
        """应用启动时执行恢复"""
        logger.info("Saga 编排器启动，开始恢复未完成的 Saga...")

        # 1. 立即恢复所有未完成的 Saga
        self.recovery_service.recover_all()

        # 2. 启动定时恢复任务（每 30 秒扫描一次）
        self._start_periodic_recovery(interval_seconds=30)

        # 3. 启动支付超时检测任务（每分钟扫描一次）
        self._start_payment_timeout_detector(interval_seconds=60)

        logger.info("Saga 编排器启动完成")

    def _start_periodic_recovery(self, interval_seconds: int):
        """定时恢复任务：捕获因崩溃、超时等原因暂停的 Saga"""
        import threading

        def periodic_scan():
            while True:
                try:
                    self.recovery_service.recover_all()
                except Exception as e:
                    logger.error(f"定时恢复扫描出错: {e}")
                time.sleep(interval_seconds)

        thread = threading.Thread(target=periodic_scan, daemon=True)
        thread.start()

    def _start_payment_timeout_detector(self, interval_seconds: int):
        """支付超时检测：查找等待支付且已超时的 Saga"""
        import threading

        def timeout_scan():
            while True:
                try:
                    # 查找 WAITING_PAYMENT 且 created_at 超过 30 分钟的 Saga
                    timeout_sagas = self.db.find_payment_timeout_sagas(timeout_minutes=30)
                    for saga in timeout_sagas:
                        self.recovery_service._recover_waiting_payment(saga)
                except Exception as e:
                    logger.error(f"支付超时检测出错: {e}")
                time.sleep(interval_seconds)

        thread = threading.Thread(target=timeout_scan, daemon=True)
        thread.start()
```

**关键设计要点：**

| 设计要点 | 说明 |
|----------|------|
| 先写日志再执行 | 每次状态变更先持久化到数据库，确保即使执行过程中崩溃也不会丢失状态 |
| 乐观锁 | 使用 version 字段防止并发状态转换冲突 |
| 幂等调用 | 恢复后可能重复调用已执行的服务，下游服务必须幂等 |
| 超时≠失败 | EXECUTING 状态的步骤在恢复时先查询实际状态，不假设失败 |
| 逆向恢复 | 补偿恢复时从最后已补偿步骤继续向前补偿 |

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
## 性能与成本分析

**各方案延迟对比：**

| 方案 | 单次事务延迟 | 峰值吞吐量 | 适用场景 |
|------|------------|-----------|---------|
| 2PC (XA) | 50-200ms（锁等待） | 500 TPS | 强一致性需求 |
| Saga 编排 | 5-10s（多步骤累加） | 5000 TPS | 长事务、可补偿 |
| Saga 协作 | 3-8s | 3000 TPS | 服务独立性强 |
| TCC | 300-500ms | 2000 TPS | 准一致性需求 |
| 本地消息表 | 200-500ms | 3000 TPS | 最终一致性 |

**Saga 存储：**

| 数据 | 日增量 | 月增量 | 存储成本 |
|------|--------|--------|---------|
| saga_instance | 100 万行 | 3000 万行 | ~30GB |
| saga_step_log | 500 万行 | 1.5 亿行 | ~150GB |
| saga_compensation_log | 10 万行 | 300 万行 | ~3GB |

**月度基础设施成本：**

| 组件 | 规格 | 数量 | 月成本 |
|------|------|------|-------|
| Saga 编排器 | 4c8G | 5 台 | ¥2 万 |
| Kafka | 8c32G + 1TB | 6 节点 | ¥3 万 |
| MySQL | 8c64G + SSD | 主从 4 台 | ¥4 万 |
| Redis（幂等+锁） | 8c32G | 3 节点 | ¥1.5 万 |
| **合计** | | | **¥10.5 万** |

## 异常场景完整演练

### 场景 1：Saga 步骤超时触发补偿级联

```
触发：Step 3（汇率锁定）超时 30 秒 → 编排器判定失败 → 开始补偿
级联过程：
  1. 补偿 Step 2（AML 筛查）：无需补偿（筛查拒绝=自然终止）
  2. 补偿 Step 1（用户支付）：退款 → 3 秒完成
  3. 通知用户：支付已退款，请重新下单
检测：Step 3 超时 → 编排器设置 status='compensating'
恢复：补偿完成后 → saga_instance.status = 'compensated'
预防：步骤超时设置合理值（汇率锁定 10s，支付 30s）
```

### 场景 2：重复消息导致双重执行

```
触发：Kafka 消息重试 → Step 2（冻结库存）被执行两次
      → 同一订单冻结了 2 件库存（应该只冻结 1 件）
检测：
  1. 幂等检查：step_log 中已有 (saga_id, step_name, status='completed')
  2. 如果发现已完成 → 跳过执行，直接返回上次结果
  3. 如果没有 → 正常执行并写入 step_log
关键：幂等键 = saga_id + step_name，而非 order_id
      因为同一订单可能有不同的 Saga 实例（不同业务流程）
```

### 场景 3：2PC 部分提交（参与者不一致）

```
触发：2PC Phase 2 → Coordinator 发送 COMMIT → 3 个参与者中 2 个提交成功
      → 第 3 个参与者收到 COMMIT 前网络断开 → 未提交 → 数据不一致
检测：
  1. Coordinator 记录每个参与者的 commit 状态
  2. 发现 participant_3.status = 'aborted'（超时未收到 COMMIT）
  3. 但 participant_1 和 participant_2 已提交 → 不可回滚
处理：
  1. Coordinator 持续重试发送 COMMIT 给 participant_3
  2. 直到 participant_3 恢复连接 → 收到 COMMIT → 提交
  3. 最终所有参与者一致（但中间有不一致窗口）
预防：2PC 协议保证最终一致性，但无法保证中间状态一致
      → 这就是为什么 2PC 不适合高并发场景
```

### 场景 4：补偿事务本身失败

```
触发：Saga 补偿 Step 1（退款）失败 → 银行接口超时
影响：用户已付款但服务未提供 → 资金无法退回
处理：
  1. 补偿失败 → 写入 compensation_retry_queue
  2. 重试策略：指数退避（5s → 10s → 30s → 1min → 5min）
  3. 最多重试 5 次 → 如果仍然失败 → 人工介入
  4. 人工介入：运营团队手动退款或联系银行
  5. 全程记录审计日志
预防：补偿操作设计为幂等，重试不会导致重复退款
```

### Saga 编排器完整实现

```python
class SagaOrchestrator:
    """带持久化和崩溃恢复的 Saga 编排器"""

    SAGA_STEPS = [
        {"name": "freeze_inventory",  "compensate": "unfreeze_inventory"},
        {"name": "create_order",      "compensate": "cancel_order"},
        {"name": "payment_preauth",   "compensate": "payment_cancel"},
        {"name": "points_deduct",     "compensate": "points_refund"},
    ]

    def execute(self, order_id, params):
        saga_id = str(uuid4())
        # 持久化 Saga 实例
        self.db.insert("saga_instances", {
            "saga_id": saga_id, "order_id": order_id,
            "status": "running", "current_step": 0,
            "created_at": now()
        })

        try:
            for i, step in enumerate(self.SAGA_STEPS):
                # 幂等检查
                if self.db.exists("saga_step_log",
                    saga_id=saga_id, step_name=step["name"], status="completed"):
                    continue  # 跳过已完成的步骤（崩溃恢复场景）

                # 标记当前步骤
                self.db.update("saga_instances",
                    {"current_step": i, "step_status": "executing"},
                    {"saga_id": saga_id})

                # 执行步骤（带超时）
                result = self._execute_with_timeout(step["name"], params, timeout=30)

                # 记录步骤完成
                self.db.insert("saga_step_log", {
                    "saga_id": saga_id, "step_name": step["name"],
                    "step_index": i, "status": "completed",
                    "result": json.dumps(result), "completed_at": now()
                })

            self.db.update("saga_instances",
                {"status": "completed"}, {"saga_id": saga_id})
            return {"saga_id": saga_id, "status": "completed"}

        except StepTimeoutError as e:
            self._compensate(saga_id, params, i)
            return {"saga_id": saga_id, "status": "compensated", "error": str(e)}
        except StepExecutionError as e:
            self._compensate(saga_id, params, i)
            return {"saga_id": saga_id, "status": "compensated", "error": str(e)}

    def _compensate(self, saga_id, params, failed_step_index):
        """逆序执行补偿"""
        self.db.update("saga_instances",
            {"status": "compensating"}, {"saga_id": saga_id})

        for i in range(failed_step_index, -1, -1):
            step = self.SAGA_STEPS[i]

            # 补偿幂等检查
            if self.db.exists("saga_compensation_log",
                saga_id=saga_id, step_name=step["name"], status="completed"):
                continue  # 跳过已完成的补偿

            try:
                getattr(self, f"_compensate_{step['compensate']}")(params, saga_id)
                self.db.insert("saga_compensation_log", {
                    "saga_id": saga_id, "step_name": step["name"],
                    "status": "completed", "completed_at": now()
                })
            except Exception as e:
                # 补偿失败 → 写入重试队列
                self.db.insert("compensation_retry_queue", {
                    "saga_id": saga_id, "step_name": step["name"],
                    "error": str(e), "retry_count": 0,
                    "next_retry_at": now() + timedelta(seconds=5)
                })

        self.db.update("saga_instances",
            {"status": "compensated"}, {"saga_id": saga_id})

    def recover_incomplete_sagas(self):
        """启动时恢复未完成的 Saga"""
        incomplete = self.db.query(
            "SELECT * FROM saga_instances WHERE status IN ('running', 'compensating')")
        for saga in incomplete:
            order_id = saga["order_id"]
            params = self.db.get_order_params(order_id)

            if saga["status"] == "running":
                last_completed = self.db.get_last_completed_step(saga["saga_id"])
                # 从 last_completed + 1 继续执行
                self._resume_from_step(saga["saga_id"], params, last_completed + 1)
            elif saga["status"] == "compensating":
                last_compensated = self.db.get_last_compensated_step(saga["saga_id"])
                # 从 last_compensated - 1 继续补偿
                self._resume_compensation(saga["saga_id"], params, last_compensated - 1)

    def _execute_with_timeout(self, step_name, params, timeout=30):
        """带超时的步骤执行"""
        import threading
        result = [None]
        error = [None]

        def run():
            try:
                result[0] = getattr(self, f"_execute_{step_name}")(params)
            except Exception as e:
                error[0] = e

        thread = threading.Thread(target=run)
        thread.start()
        thread.join(timeout=timeout)

        if thread.is_alive():
            raise StepTimeoutError(f"Step {step_name} timed out after {timeout}s")
        if error[0]:
            raise StepExecutionError(f"Step {step_name} failed: {error[0]}")
        return result[0]
```

### TCC 完整实现

```python
class TCCOrderService:
    """TCC 模式：Try-Confirm-Cancel"""

    def try_phase(self, order):
        """Try: 预留资源"""
        # 1. 冻结库存
        frozen = self.inventory.try_freeze(order.sku_id, order.quantity)
        if not frozen:
            raise InsufficientInventoryError(order.sku_id)

        # 2. 预授权支付
        preauth = self.payment.try_preauth(order.user_id, order.amount)
        if not preauth:
            self.inventory.unfreeze(order.sku_id, order.quantity)  # 回滚库存冻结
            raise PaymentPreauthFailedError(order.user_id)

        # 3. 预扣积分
        points_frozen = self.points.try_freeze(order.user_id, order.points_amount)
        if not points_frozen:
            self.payment.cancel_preauth(order.user_id, order.amount)  # 回滚支付
            self.inventory.unfreeze(order.sku_id, order.quantity)     # 回滚库存
            raise InsufficientPointsError(order.user_id)

        return {"status": "tried", "order_id": order.id}

    def confirm_phase(self, order):
        """Confirm: 确认所有资源扣减"""
        # 幂等检查
        if self.db.get_order_status(order.id) == "confirmed":
            return  # 已确认，跳过

        # 1. 确认库存扣减（frozen → sold）
        self.inventory.confirm_deduct(order.sku_id, order.quantity)
        # 2. 确认支付扣款（preauth → captured）
        self.payment.confirm_capture(order.user_id, order.amount)
        # 3. 确认积分扣减（frozen → deducted）
        self.points.confirm_deduct(order.user_id, order.points_amount)

        self.db.update_order_status(order.id, "confirmed")

    def cancel_phase(self, order):
        """Cancel: 释放所有预留资源"""
        # 幂等检查
        if self.db.get_order_status(order.id) == "cancelled":
            return  # 已取消，跳过

        # 逆序释放
        self.points.unfreeze(order.user_id, order.points_amount)
        self.payment.cancel_preauth(order.user_id, order.amount)
        self.inventory.unfreeze(order.sku_id, order.quantity)

        self.db.update_order_status(order.id, "cancelled")
```

**TCC vs Saga 对比：**

| 维度 | TCC | Saga |
|------|-----|------|
| 一致性 | 准一致性（Try 后资源已冻结） | 最终一致性（补偿后回滚） |
| 隔离性 | Try 阶段即冻结资源，隔离性好 | 无隔离性，可能出现脏读 |
| 延迟 | 300-500ms（Try+Confirm） | 5-10s（多步骤串行） |
| 实现复杂度 | 高（每个服务需实现3个方法） | 中（每个步骤+补偿） |
| 适用场景 | 支付、库存等强隔离需求 | 订单流转、跨服务编排 |

### Saga 崩溃恢复与状态持久化

```python
class SagaRecoveryService:
    """编排器崩溃后恢复未完成的 Saga"""

    def recover_on_startup(self):
        """服务启动时扫描并恢复所有未完成的 Saga"""
        incomplete = self.db.query(
            "SELECT * FROM saga_instances WHERE status IN ('running', 'compensating')")
        recovered = 0
        for saga in incomplete:
            if saga["status"] == "running":
                # 找到最后一个完成的步骤
                last_completed = self.db.query_one(
                    "SELECT MAX(step_index) as idx FROM saga_step_log "
                    "WHERE saga_id = %s AND status = 'completed'", saga["saga_id"])
                next_step = (last_completed["idx"] or -1) + 1
                if next_step < len(SagaOrchestrator.SAGA_STEPS):
                    # 从 next_step 继续执行
                    self.orchestrator.resume_from_step(saga["saga_id"], next_step)
                    recovered += 1
                else:
                    # 所有步骤都完成了但状态未更新 → 标记完成
                    self.db.update("saga_instances",
                        {"status": "completed"}, {"saga_id": saga["saga_id"]})

            elif saga["status"] == "compensating":
                # 找到最后一个完成的补偿
                last_compensated = self.db.query_one(
                    "SELECT MIN(step_index) as idx FROM saga_compensation_log "
                    "WHERE saga_id = %s AND status = 'completed'", saga["saga_id"])
                next_comp = (last_compensated["idx"] or len(SagaOrchestrator.SAGA_STEPS)) - 1
                if next_comp >= 0:
                    self.orchestrator.resume_compensation(saga["saga_id"], next_comp)
                    recovered += 1

        return recovered

    def check_stuck_sagas(self):
        """定期检查卡住的 Saga（超时未完成）"""
        stuck = self.db.query(
            "SELECT * FROM saga_instances "
            "WHERE status = 'running' AND created_at < NOW() - INTERVAL 10 MINUTE")
        for saga in stuck:
            # 超时 10 分钟仍在运行 → 强制补偿
            self.db.update("saga_instances",
                {"status": "compensating"}, {"saga_id": saga["saga_id"]})
            self.orchestrator._compensate(saga["saga_id"],
                self.db.get_order_params(saga["order_id"]),
                len(SagaOrchestrator.SAGA_STEPS) - 1)
```

### 补偿重试队列

```python
class CompensationRetryWorker:
    """补偿失败后的重试处理器"""

    RETRY_DELAYS = [5, 15, 30, 60, 300]  # 秒，指数退避

    def process_retry_queue(self):
        """处理补偿重试队列"""
        pending = self.db.query(
            "SELECT * FROM compensation_retry_queue "
            "WHERE next_retry_at <= NOW() AND retry_count < 5 "
            "ORDER BY retry_count ASC LIMIT 100")

        for item in pending:
            try:
                # 执行补偿
                self.orchestrator.execute_single_compensation(
                    item["saga_id"], item["step_name"])
                # 成功 → 从队列移除
                self.db.delete("compensation_retry_queue", {"id": item["id"]})
            except Exception as e:
                # 仍然失败 → 更新重试次数和下次重试时间
                next_delay = self.RETRY_DELAYS[min(item["retry_count"], 4)]
                self.db.update("compensation_retry_queue", {
                    "retry_count": item["retry_count"] + 1,
                    "last_error": str(e),
                    "next_retry_at": now() + timedelta(seconds=next_delay)
                }, {"id": item["id"]})

                # 超过最大重试次数 → 人工介入
                if item["retry_count"] + 1 >= 5:
                    self.alert_service.notify(
                        f"Saga {item['saga_id']} 补偿失败 5 次，需人工介入: "
                        f"步骤={item['step_name']}, 错误={str(e)}")
```

### 幂等性保障完整实现

```python
class IdempotentStepExecutor:
    """幂等步骤执行器：保证每步只执行一次"""

    def execute_step(self, saga_id, step_name, step_func, params):
        """幂等执行：检查 step_log 决定是否跳过"""
        idempotency_key = f"{saga_id}:{step_name}"

        # 检查是否已执行
        existing = self.db.query_one(
            "SELECT * FROM saga_step_log "
            "WHERE saga_id = %s AND step_name = %s", saga_id, step_name)

        if existing:
            if existing["status"] == "completed":
                return json.loads(existing["result"])  # 返回上次结果
            elif existing["status"] == "executing":
                raise ConcurrentExecutionError(
                    f"Step {step_name} is being executed by another process")

        # 标记为执行中（防止并发）
        self.db.insert("saga_step_log", {
            "saga_id": saga_id, "step_name": step_name,
            "status": "executing", "started_at": now()
        })

        try:
            result = step_func(params)
            self.db.update("saga_step_log",
                {"status": "completed", "result": json.dumps(result), "completed_at": now()},
                {"saga_id": saga_id, "step_name": step_name})
            return result
        except Exception as e:
            self.db.update("saga_step_log",
                {"status": "failed", "error": str(e)},
                {"saga_id": saga_id, "step_name": step_name})
            raise
```

## TCC 完整实现：订单+库存+支付

```python
class TCCOrderCoordinator:
    """TCC 事务协调器"""

    def execute(self, order):
        """TCC 三阶段执行"""
        tcc_id = str(uuid4())
        self.db.insert("tcc_transactions", {
            "tcc_id": tcc_id, "order_id": order.id,
            "status": "trying", "created_at": now()
        })

        # Phase 1: Try - 预留资源
        try:
            frozen = self.inventory_tcc.try_freeze(tcc_id, order.sku_id, order.quantity)
            preauth = self.payment_tcc.try_preauth(tcc_id, order.user_id, order.amount)
            points = self.points_tcc.try_freeze(tcc_id, order.user_id, order.points)
        except Exception as e:
            # Try 失败 → 取消已预留的资源
            self.cancel_phase(tcc_id, order)
            raise TCCTryFailedError(str(e))

        # Phase 2: Confirm - 确认扣减
        try:
            self.confirm_phase(tcc_id, order)
        except Exception as e:
            # Confirm 失败 → 重试（不能回滚，因为资源已扣减）
            self.db.insert("tcc_confirm_retry_queue", {
                "tcc_id": tcc_id, "error": str(e),
                "retry_count": 0, "next_retry_at": now() + timedelta(seconds=5)
            })
            raise TCCConfirmFailedError(str(e))

        return {"tcc_id": tcc_id, "status": "confirmed"}

    def confirm_phase(self, tcc_id, order):
        """Confirm: 确认所有资源扣减（幂等）"""
        self.inventory_tcc.confirm_deduct(tcc_id, order.sku_id, order.quantity)
        self.payment_tcc.confirm_capture(tcc_id, order.user_id, order.amount)
        self.points_tcc.confirm_deduct(tcc_id, order.user_id, order.points)
        self.db.update("tcc_transactions",
            {"status": "confirmed", "confirmed_at": now()},
            {"tcc_id": tcc_id})

    def cancel_phase(self, tcc_id, order):
        """Cancel: 释放所有预留资源（幂等）"""
        self.inventory_tcc.cancel_unfreeze(tcc_id, order.sku_id, order.quantity)
        self.payment_tcc.cancel_preauth(tcc_id, order.user_id, order.amount)
        self.points_tcc.cancel_unfreeze(tcc_id, order.user_id, order.points)
        self.db.update("tcc_transactions",
            {"status": "cancelled", "cancelled_at": now()},
            {"tcc_id": tcc_id})


class InventoryTCCService:
    """库存服务的 TCC 实现"""

    def try_freeze(self, tcc_id, sku_id, quantity):
        """Try: 冻结库存"""
        # 幂等检查
        existing = self.db.query_one(
            "SELECT * FROM tcc_inventory_log WHERE tcc_id=%s AND action='freeze'",
            tcc_id)
        if existing and existing["status"] == "frozen":
            return existing  # 已冻结，跳过

        # 检查库存并冻结
        affected = self.db.execute(
            "UPDATE inventory SET available = available - %s, frozen = frozen + %s "
            "WHERE sku_id = %s AND available >= %s",
            quantity, quantity, sku_id, quantity)
        if affected == 0:
            raise InsufficientInventoryError(f"SKU {sku_id} 库存不足")

        self.db.insert("tcc_inventory_log", {
            "tcc_id": tcc_id, "sku_id": sku_id, "action": "freeze",
            "quantity": quantity, "status": "frozen", "created_at": now()
        })

    def confirm_deduct(self, tcc_id, sku_id, quantity):
        """Confirm: 冻结→已售（幂等）"""
        existing = self.db.query_one(
            "SELECT * FROM tcc_inventory_log WHERE tcc_id=%s AND action='freeze'",
            tcc_id)
        if not existing:
            return  # 无冻结记录，跳过
        if existing["status"] == "deducted":
            return  # 已扣减，跳过

        self.db.execute(
            "UPDATE inventory SET frozen = frozen - %s, sold = sold + %s "
            "WHERE sku_id = %s", quantity, quantity, sku_id)
        self.db.update("tcc_inventory_log",
            {"status": "deducted"}, {"tcc_id": tcc_id, "action": "freeze"})

    def cancel_unfreeze(self, tcc_id, sku_id, quantity):
        """Cancel: 解冻（幂等）"""
        existing = self.db.query_one(
            "SELECT * FROM tcc_inventory_log WHERE tcc_id=%s AND action='freeze'",
            tcc_id)
        if not existing or existing["status"] in ("unfrozen", "deducted"):
            return  # 无需解冻

        self.db.execute(
            "UPDATE inventory SET available = available + %s, frozen = frozen - %s "
            "WHERE sku_id = %s", quantity, quantity, sku_id)
        self.db.update("tcc_inventory_log",
            {"status": "unfrozen"}, {"tcc_id": tcc_id, "action": "freeze"})


class PaymentTCCService:
    """支付服务的 TCC 实现"""

    def try_preauth(self, tcc_id, user_id, amount):
        """Try: 预授权（冻结金额）"""
        existing = self.db.query_one(
            "SELECT * FROM tcc_payment_log WHERE tcc_id=%s AND action='preauth'",
            tcc_id)
        if existing and existing["status"] == "preauthorized":
            return existing

        # 调用银行预授权接口
        result = self.bank_client.preauth(user_id, amount)
        if not result.success:
            raise PaymentPreauthFailedError(result.message)

        self.db.insert("tcc_payment_log", {
            "tcc_id": tcc_id, "user_id": user_id, "action": "preauth",
            "amount": amount, "bank_preauth_id": result.preauth_id,
            "status": "preauthorized", "created_at": now()
        })

    def confirm_capture(self, tcc_id, user_id, amount):
        """Confirm: 预授权→正式扣款"""
        existing = self.db.query_one(
            "SELECT * FROM tcc_payment_log WHERE tcc_id=%s AND action='preauth'",
            tcc_id)
        if not existing or existing["status"] == "captured":
            return

        self.bank_client.capture(existing["bank_preauth_id"], amount)
        self.db.update("tcc_payment_log",
            {"status": "captured", "captured_at": now()},
            {"tcc_id": tcc_id, "action": "preauth"})

    def cancel_preauth(self, tcc_id, user_id, amount):
        """Cancel: 取消预授权"""
        existing = self.db.query_one(
            "SELECT * FROM tcc_payment_log WHERE tcc_id=%s AND action='preauth'",
            tcc_id)
        if not existing or existing["status"] in ("cancelled", "captured"):
            return

        self.bank_client.void_preauth(existing["bank_preauth_id"])
        self.db.update("tcc_payment_log",
            {"status": "cancelled", "cancelled_at": now()},
            {"tcc_id": tcc_id, "action": "preauth"})
```

**TCC 数据库表：**

```sql
CREATE TABLE tcc_transactions (
    tcc_id VARCHAR(36) PRIMARY KEY,
    order_id VARCHAR(36) NOT NULL,
    status ENUM('trying', 'confirmed', 'cancelled') NOT NULL,
    created_at TIMESTAMP NOT NULL,
    confirmed_at TIMESTAMP NULL,
    cancelled_at TIMESTAMP NULL,
    INDEX idx_status (status),
    INDEX idx_created (created_at)
);

CREATE TABLE tcc_inventory_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tcc_id VARCHAR(36) NOT NULL,
    sku_id VARCHAR(50) NOT NULL,
    action VARCHAR(20) NOT NULL,
    quantity INT NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    UNIQUE KEY uk_tcc_action (tcc_id, action)
);

CREATE TABLE tcc_payment_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tcc_id VARCHAR(36) NOT NULL,
    user_id VARCHAR(36) NOT NULL,
    action VARCHAR(20) NOT NULL,
    amount DECIMAL(12,2) NOT NULL,
    bank_preauth_id VARCHAR(50),
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    captured_at TIMESTAMP NULL,
    cancelled_at TIMESTAMP NULL,
    UNIQUE KEY uk_tcc_action (tcc_id, action)
);
```

## 本地消息表完整实现

```python
class LocalMessageService:
    """本地消息表：保证业务操作和消息发送的原子性"""

    def execute_with_message(self, business_func, message_topic, message_body):
        """在同一个数据库事务中执行业务逻辑和写入消息表"""
        with self.db.transaction() as tx:
            # 1. 执行业务逻辑
            result = business_func(tx)

            # 2. 写入本地消息表（同一事务）
            tx.insert("local_message_table", {
                "message_id": str(uuid4()),
                "topic": message_topic,
                "body": json.dumps(message_body),
                "status": "pending",
                "retry_count": 0,
                "created_at": now(),
                "next_retry_at": now()
            })

            # 事务提交：业务数据 + 消息同时持久化
            return result


class MessageRelayService:
    """消息中继：定时扫描未发送消息，发送到 MQ"""

    RELAY_DELAYS = [0, 5, 15, 30, 60, 120, 300]  # 重试延迟（秒）

    def relay_pending_messages(self):
        """扫描并发送待发送消息"""
        messages = self.db.query(
            "SELECT * FROM local_message_table "
            "WHERE status = 'pending' AND next_retry_at <= NOW() "
            "AND retry_count < 7 ORDER BY created_at ASC LIMIT 200")

        for msg in messages:
            try:
                self.kafka_producer.send(msg["topic"], msg["body"],
                    headers={"message_id": msg["message_id"]})
                self.db.update("local_message_table",
                    {"status": "published", "published_at": now()},
                    {"message_id": msg["message_id"]})
            except Exception as e:
                next_delay = self.RELAY_DELAYS[min(msg["retry_count"] + 1, 6)]
                self.db.update("local_message_table", {
                    "retry_count": msg["retry_count"] + 1,
                    "last_error": str(e),
                    "next_retry_at": now() + timedelta(seconds=next_delay)
                }, {"message_id": msg["message_id"]})

                if msg["retry_count"] + 1 >= 7:
                    self.alert(f"消息发送失败 7 次: {msg['message_id']}")


class MessageConsumerService:
    """消息消费端：幂等消费"""

    def consume(self, message):
        """消费消息（幂等）"""
        message_id = message.headers.get("message_id")

        # 幂等检查：是否已消费
        if self.db.exists("consumed_message_log", message_id=message_id):
            return  # 已消费，跳过

        # 执行业务逻辑
        self.process_business_logic(message.body)

        # 记录已消费
        self.db.insert("consumed_message_log", {
            "message_id": message_id,
            "consumer_group": self.consumer_group,
            "consumed_at": now()
        })
```

**本地消息表 DDL：**

```sql
CREATE TABLE local_message_table (
    message_id VARCHAR(36) PRIMARY KEY,
    topic VARCHAR(100) NOT NULL,
    body TEXT NOT NULL,
    status ENUM('pending', 'published', 'failed') NOT NULL DEFAULT 'pending',
    retry_count INT NOT NULL DEFAULT 0,
    last_error TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    next_retry_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    published_at TIMESTAMP NULL,
    INDEX idx_status_next_retry (status, next_retry_at)
);

CREATE TABLE consumed_message_log (
    message_id VARCHAR(36) NOT NULL,
    consumer_group VARCHAR(50) NOT NULL,
    consumed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (message_id, consumer_group)
);
```

## 分布式事务选型决策

| 决策维度 | 2PC (XA) | Saga 编排 | Saga 协作 | TCC | 本地消息表 |
|---------|----------|----------|----------|-----|-----------|
| 一致性 | 强一致 | 最终一致 | 最终一致 | 准一致 | 最终一致 |
| 隔离性 | 完全隔离 | 无隔离 | 无隔离 | Try阶段隔离 | 无隔离 |
| 延迟 | 50-200ms | 5-10s | 3-8s | 300-500ms | 200-500ms |
| 吞吐量 | 500 TPS | 5000 TPS | 3000 TPS | 2000 TPS | 3000 TPS |
| 实现复杂度 | 低（DB支持） | 中 | 中 | 高 | 低 |
| 适用场景 | 跨DB事务 | 长事务编排 | 服务独立 | 支付/库存 | 通知/异步 |

**选型决策树：**

```
需要强一致性？
├─ 是 → 是否可以接受锁等待？
│   ├─ 是 → 2PC (XA)
│   └─ 否 → 业务可补偿？→ Saga
└─ 否 → 需要资源隔离？
    ├─ 是 → TCC（冻结→确认→取消）
    └─ 否 → 是否需要实时响应？
        ├─ 是 → 本地消息表
        └─ 否 → Saga 协作（事件驱动）
```

**实际场景映射：**

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| 支付+扣库存 | TCC | 需要冻结隔离，防止超卖 |
| 订单流转（创建→支付→发货） | Saga 编排 | 长事务，步骤明确，可补偿 |
| 用户注册→发欢迎邮件 | 本地消息表 | 邮件失败不影响注册 |
| 跨分库转账 | 2PC | 强一致性，金额不能错 |
| 库存扣减→推荐系统更新 | Saga 协作 | 服务独立，事件驱动 |

## 分布式事务监控与可观测性

```python
class DistributedTransactionMonitor:
    """分布式事务监控：实时追踪事务状态"""

    def get_saga_dashboard(self):
        """Saga 仪表盘数据"""
        return {
            "running_sagas": self.db.count("saga_instances", status="running"),
            "compensating_sagas": self.db.count("saga_instances", status="compensating"),
            "completed_sagas_1h": self.db.count("saga_instances",
                status="completed", completed_at__gte=now() - timedelta(hours=1)),
            "failed_sagas_1h": self.db.count("saga_instances",
                status="compensated", completed_at__gte=now() - timedelta(hours=1)),
            "avg_saga_duration_ms": self.db.avg("saga_instances",
                "EXTRACT(EPOCH FROM (completed_at - created_at)) * 1000",
                status="completed", completed_at__gte=now() - timedelta(hours=1)),
            "compensation_rate": self._calc_compensation_rate(),
            "step_latency": self._get_step_latency_breakdown(),
        }

    def _calc_compensation_rate(self):
        """补偿率：需要补偿的事务比例"""
        total = self.db.count("saga_instances", created_at__gte=now() - timedelta(hours=24))
        compensated = self.db.count("saga_instances",
            status="compensated", created_at__gte=now() - timedelta(hours=24))
        return compensated / max(total, 1)

    def _get_step_latency_breakdown(self):
        """每步骤延迟分解"""
        steps = ["freeze_inventory", "create_order", "payment_preauth", "points_deduct"]
        result = {}
        for step in steps:
            result[step] = {
                "p50": self.db.percentile("saga_step_log", "duration_ms", 50,
                    step_name=step, status="completed"),
                "p95": self.db.percentile("saga_step_log", "duration_ms", 95,
                    step_name=step, status="completed"),
                "p99": self.db.percentile("saga_step_log", "duration_ms", 99,
                    step_name=step, status="completed"),
                "failure_rate": self.db.count("saga_step_log",
                    step_name=step, status="failed") / max(self.db.count("saga_step_log",
                    step_name=step), 1)
            }
        return result
```

## 异常场景补充

### 场景：2PC Phase 2 部分提交恢复

```python
class TwoPhaseCommitRecovery:
    """2PC 阶段 2 部分提交恢复"""
    def recover(self, xid, participants):
        """恢复部分提交的 2PC 事务"""
        # 检查每个参与者的状态
        statuses = {}
        for p in participants:
            try:
                status = p.query_status(xid)
                statuses[p.name] = status  # committed / aborted / prepared
            except Exception:
                statuses[p.name] = "unknown"

        committed = [p for p, s in statuses.items() if s == "committed"]
        prepared = [p for p, s in statuses.items() if s == "prepared"]
        aborted = [p for p, s in statuses.items() if s == "aborted"]

        if aborted and committed:
            # 最严重：部分已提交，部分已回滚 → 数据不一致
            # 2PC 协议保证最终一致：持续重试提交给 prepared 状态的参与者
            for p in prepared:
                self._retry_commit(p, xid, max_retries=10)

            # 对 aborted 的参与者 → 需要人工补偿
            self.alert(f"2PC 不一致: xid={xid}, committed={committed}, aborted={aborted}")

        elif prepared and not aborted:
            # 所有参与者都是 prepared → 可以安全提交
            for p in prepared:
                p.commit(xid)

        return {"xid": xid, "committed": committed, "prepared": prepared, "aborted": aborted}
```

### 场景：消息乱序导致状态错误

```
触发：Kafka 分区中消息乱序：
      消息 1: order_cancelled (迟到)
      消息 2: order_created (先到)
      → 消费者先处理 cancel → 订单不存在 → 忽略
      → 再处理 create → 订单被创建但本应被取消
检测：
  1. 每条消息携带时间戳和版本号
  2. 消费者检查版本号：新消息版本 < 当前状态版本 → 丢弃
  3. 顺序异常告警：版本号回退事件
处理：
  1. 消费者维护 entity_version 表
  2. 处理前检查：message.version > entity.version → 处理
  3. message.version <= entity.version → 丢弃（过期消息）
预防：Kafka 保证分区内顺序 + 生产者按 entity_id 路由到同一分区
```

## Saga 协作式完整实现

```python
class SagaChoreographyCoordinator:
    """Saga 协作式：事件驱动，无中心编排器"""

    def on_order_created(self, event):
        """订单创建事件 → 冻结库存"""
        order = event["order"]
        try:
            frozen = self.inventory.freeze(order.sku_id, order.quantity)
            if frozen:
                self.event_bus.publish("InventoryFrozenEvent", {
                    "order_id": order.id, "sku_id": order.sku_id,
                    "quantity": order.quantity, "user_id": order.user_id,
                    "amount": order.amount
                })
            else:
                self.event_bus.publish("InventoryFrozenFailedEvent", {
                    "order_id": order.id, "reason": "insufficient_stock"
                })
        except Exception as e:
            self.event_bus.publish("InventoryFrozenFailedEvent", {
                "order_id": order.id, "reason": str(e)
            })

    def on_inventory_frozen(self, event):
        """库存冻结成功 → 执行支付"""
        try:
            result = self.payment.preauth(event["user_id"], event["amount"])
            if result.success:
                self.event_bus.publish("PaymentCompletedEvent", {
                    "order_id": event["order_id"],
                    "payment_id": result.payment_id
                })
            else:
                self.event_bus.publish("PaymentFailedEvent", {
                    "order_id": event["order_id"], "reason": result.message
                })
        except Exception as e:
            self.event_bus.publish("PaymentFailedEvent", {
                "order_id": event["order_id"], "reason": str(e)
            })

    def on_payment_completed(self, event):
        """支付完成 → 确认订单"""
        self.order.confirm(event["order_id"])
        self.event_bus.publish("OrderConfirmedEvent", {
            "order_id": event["order_id"]
        })

    def on_payment_failed(self, event):
        """支付失败 → 取消冻结库存 + 取消订单"""
        self.inventory.unfreeze(event["order_id"])
        self.order.cancel(event["order_id"], reason=event["reason"])

    def on_inventory_frozen_failed(self, event):
        """库存冻结失败 → 取消订单"""
        self.order.cancel(event["order_id"], reason=event["reason"])
```

**编排式 vs 协作式对比：**

| 维度 | 编排式 (Orchestration) | 协作式 (Choreography) |
|------|----------------------|---------------------|
| 中心化 | 有中心编排器 | 无中心，事件驱动 |
| 可观测性 | 编排器可查完整状态 | 需要事件溯源重建状态 |
| 耦合度 | 编排器依赖所有服务 | 服务只依赖事件 |
| 延迟 | 编排器串行调度，5-10s | 事件并行，3-8s |
| 复杂度 | 编排器逻辑复杂 | 事件链路难追踪 |
| 适用场景 | 步骤明确的流程 | 服务独立性强 |
| 测试 | 编排器可单独测试 | 需要集成测试 |

## 可靠事件发布（Outbox 模式）

```python
class EventOutboxPublisher:
    """Outbox 模式：保证业务操作和事件发布的原子性"""

    def execute_with_event(self, business_func, event_type, event_data):
        """在同一事务中执行业务逻辑和写入事件"""
        with self.db.transaction() as tx:
            # 1. 执行业务逻辑
            result = business_func(tx)

            # 2. 写入 outbox 表（同一事务）
            tx.insert("event_outbox", {
                "event_id": str(uuid4()),
                "event_type": event_type,
                "aggregate_id": result.id,
                "event_data": json.dumps(event_data),
                "status": "pending",
                "created_at": now()
            })
            # 事务提交：业务数据 + 事件同时持久化
            return result

    def publish_pending_events(self):
        """定时扫描 outbox，发布未发送事件"""
        events = self.db.query(
            "SELECT * FROM event_outbox WHERE status = 'pending' "
            "ORDER BY created_at ASC LIMIT 200")
        for event in events:
            try:
                self.kafka_producer.send(event["event_type"], event["event_data"],
                    headers={"event_id": event["event_id"]})
                self.db.update("event_outbox",
                    {"status": "published", "published_at": now()},
                    {"event_id": event["event_id"]})
            except Exception as e:
                self.db.update("event_outbox",
                    {"retry_count": event["retry_count"] + 1,
                     "last_error": str(e)},
                    {"event_id": event["event_id"]})
```

**Outbox 表 DDL：**

```sql
CREATE TABLE event_outbox (
    event_id VARCHAR(36) PRIMARY KEY,
    event_type VARCHAR(100) NOT NULL,
    aggregate_id VARCHAR(36) NOT NULL,
    event_data TEXT NOT NULL,
    status ENUM('pending', 'published', 'failed') DEFAULT 'pending',
    retry_count INT DEFAULT 0,
    last_error TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    published_at TIMESTAMP NULL,
    INDEX idx_status_created (status, created_at)
);
```

## 异常场景补充

### 场景：僵尸 Saga 检测与恢复

```python
class ZombieSagaDetector:
    """检测并恢复卡住的 Saga"""
    def detect_and_recover(self):
        """扫描超时未完成的 Saga"""
        stuck = self.db.query(
            "SELECT * FROM saga_instances "
            "WHERE status = 'running' AND updated_at < NOW() - INTERVAL 10 MINUTE")
        for saga in stuck:
            # 检查是否有步骤正在执行中
            executing = self.db.query_one(
                "SELECT COUNT(*) as cnt FROM saga_step_log "
                "WHERE saga_id = %s AND status = 'executing'", saga["saga_id"])
            if executing["cnt"] > 0:
                # 有步骤在执行中但超时 → 强制失败 → 触发补偿
                self.db.update("saga_step_log",
                    {"status": "timeout"},
                    {"saga_id": saga["saga_id"], "status": "executing"})
                self.db.update("saga_instances",
                    {"status": "compensating"},
                    {"saga_id": saga["saga_id"]})
                self.orchestrator._compensate(saga["saga_id"],
                    self.db.get_order_params(saga["order_id"]),
                    len(SagaOrchestrator.SAGA_STEPS) - 1)
```

### 场景：事件重复消费

```
触发：Kafka 消息重试 → 同一事件被消费两次
      → 库存冻结两次 → 数量错误
检测：
  1. 消费幂等检查：consumed_events 表中已有 event_id
  2. 事件处理前先查询：是否已处理
处理：
  1. 消费前检查 consumed_events 表
  2. 已消费 → 跳过（返回成功，不抛异常）
  3. 未消费 → 处理业务 + 写入 consumed_events（同一事务）
预防：每个消费者维护 consumed_events 去重表
```

## Saga 协作模式：事件驱动的完整实现

与编排模式不同，协作模式（Choreography）没有中心编排器，各服务通过事件自主协调。每个服务监听自己关心的事件，执行操作后发出新事件，形成事件链。

### 事件定义与流转

```
OrderCreatedEvent → InventoryService 监听 → 冻结库存 → 发出 InventoryFrozenEvent
InventoryFrozenEvent → PaymentService 监听 → 处理支付 → 发出 PaymentCompletedEvent
PaymentCompletedEvent → OrderService 监听 → 确认订单
补偿事件：
PaymentFailedEvent → InventoryService 解冻库存，OrderService 取消订单
```

### 完整的 Saga 协作实现

```python
import json
import uuid
import logging
from datetime import datetime
from dataclasses import dataclass, field, asdict
from typing import Dict, Any, Optional, List
from enum import Enum

logger = logging.getLogger("saga_choreography")


# ===== 1. 事件基础定义 =====

class EventType(Enum):
    """Saga 事件类型"""
    ORDER_CREATED = "OrderCreatedEvent"
    INVENTORY_FROZEN = "InventoryFrozenEvent"
    INVENTORY_FREEZE_FAILED = "InventoryFreezeFailedEvent"
    PAYMENT_COMPLETED = "PaymentCompletedEvent"
    PAYMENT_FAILED = "PaymentFailedEvent"
    ORDER_CONFIRMED = "OrderConfirmedEvent"
    ORDER_CANCELLED = "OrderCancelledEvent"
    INVENTORY_RELEASED = "InventoryReleasedEvent"
    POINTS_FROZEN = "PointsFrozenEvent"
    POINTS_FREEZE_FAILED = "PointsFreezeFailedEvent"
    POINTS_RELEASED = "PointsReleasedEvent"


@dataclass
class SagaEvent:
    """Saga 事件基类"""
    event_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    event_type: str = ""
    saga_id: str = ""
    order_id: str = ""
    timestamp: str = field(default_factory=lambda: datetime.utcnow().isoformat())
    payload: Dict[str, Any] = field(default_factory=dict)
    correlation_id: str = ""       # 关联ID，串联同一 Saga 的所有事件
    causation_id: str = ""         # 因果ID，记录触发本事件的上游事件ID

    def to_dict(self):
        return asdict(self)

    @classmethod
    def from_dict(cls, data: dict):
        return cls(**{k: v for k, v in data.items() if k in cls.__dataclass_fields__})


# ===== 2. 订单服务：Saga 发起方 =====

class OrderServiceChoreography:
    """
    订单服务：Saga 协作模式的发起方
    职责：
    - 创建订单并发送 OrderCreatedEvent
    - 监听 InventoryFrozenEvent → 通知支付
    - 监听 PaymentCompletedEvent → 确认订单
    - 监听补偿事件 → 取消订单
    """

    def __init__(self, db, event_bus):
        self.db = db
        self.event_bus = event_bus
        self._register_handlers()

    def _register_handlers(self):
        """注册事件处理器"""
        self.event_bus.subscribe(EventType.INVENTORY_FROZEN.value, self.on_inventory_frozen)
        self.event_bus.subscribe(EventType.PAYMENT_COMPLETED.value, self.on_payment_completed)
        self.event_bus.subscribe(EventType.PAYMENT_FAILED.value, self.on_payment_failed)
        self.event_bus.subscribe(EventType.INVENTORY_FREEZE_FAILED.value, self.on_inventory_freeze_failed)

    def create_order(self, order_request: dict) -> dict:
        """创建订单：Saga 流程的起点"""
        order_id = f"ORD-{uuid.uuid4().hex[:12]}"
        saga_id = f"SAGA-{uuid.uuid4().hex[:12]}"

        # 持久化订单（状态：待支付）
        self.db.insert("orders", {
            "order_id": order_id,
            "saga_id": saga_id,
            "user_id": order_request["user_id"],
            "item_id": order_request["item_id"],
            "quantity": order_request["quantity"],
            "points_amount": order_request.get("points_amount", 0),
            "status": "PENDING_PAYMENT",
            "created_at": datetime.utcnow().isoformat()
        })

        # 发出 OrderCreatedEvent，触发下游服务
        event = SagaEvent(
            event_type=EventType.ORDER_CREATED.value,
            saga_id=saga_id,
            order_id=order_id,
            correlation_id=saga_id,
            causation_id="",
            payload={
                "user_id": order_request["user_id"],
                "item_id": order_request["item_id"],
                "quantity": order_request["quantity"],
                "points_amount": order_request.get("points_amount", 0),
                "amount": order_request["amount"]
            }
        )
        self.event_bus.publish("order-events", event.to_dict())
        logger.info(f"Order created: {order_id}, saga: {saga_id}")

        return {"order_id": order_id, "saga_id": saga_id, "status": "PENDING_PAYMENT"}

    def on_inventory_frozen(self, event_data: dict):
        """监听库存冻结成功事件 → 触发积分冻结/支付"""
        event = SagaEvent.from_dict(event_data)
        logger.info(f"Order {event.order_id}: inventory frozen, proceeding to payment")

        order = self.db.get("orders", order_id=event.order_id)
        if not order or order["status"] != "PENDING_PAYMENT":
            logger.warning(f"Order {event.order_id} not in valid state, skipping")
            return

        # 如果有积分抵扣，先冻结积分；否则直接发起支付
        points_amount = order.get("points_amount", 0)
        if points_amount > 0:
            points_event = SagaEvent(
                event_type="FreezePointsCommand",
                saga_id=event.saga_id,
                order_id=event.order_id,
                correlation_id=event.correlation_id,
                causation_id=event.event_id,
                payload={"user_id": order["user_id"], "points_amount": points_amount}
            )
            self.event_bus.publish("points-events", points_event.to_dict())
        else:
            # 无积分，直接发起支付
            payment_event = SagaEvent(
                event_type="ProcessPaymentCommand",
                saga_id=event.saga_id,
                order_id=event.order_id,
                correlation_id=event.correlation_id,
                causation_id=event.event_id,
                payload={"user_id": order["user_id"], "amount": order.get("amount", 0)}
            )
            self.event_bus.publish("payment-events", payment_event.to_dict())

    def on_payment_completed(self, event_data: dict):
        """监听支付完成事件 → 确认订单"""
        event = SagaEvent.from_dict(event_data)
        logger.info(f"Order {event.order_id}: payment completed, confirming order")

        # 幂等检查
        order = self.db.get("orders", order_id=event.order_id)
        if not order or order["status"] == "CONFIRMED":
            return

        self.db.update("orders",
            {"status": "CONFIRMED", "confirmed_at": datetime.utcnow().isoformat()},
            {"order_id": event.order_id})

        # 发出订单确认事件
        confirm_event = SagaEvent(
            event_type=EventType.ORDER_CONFIRMED.value,
            saga_id=event.saga_id,
            order_id=event.order_id,
            correlation_id=event.correlation_id,
            causation_id=event.event_id,
            payload={"status": "CONFIRMED"}
        )
        self.event_bus.publish("order-events", confirm_event.to_dict())

    def on_payment_failed(self, event_data: dict):
        """监听支付失败事件 → 取消订单（触发补偿链）"""
        event = SagaEvent.from_dict(event_data)
        logger.warning(f"Order {event.order_id}: payment failed, cancelling order")

        order = self.db.get("orders", order_id=event.order_id)
        if not order or order["status"] == "CANCELLED":
            return

        self.db.update("orders",
            {"status": "CANCELLED", "cancelled_at": datetime.utcnow().isoformat(),
             "cancel_reason": "payment_failed"},
            {"order_id": event.order_id})

        # 发出订单取消事件，触发库存释放和积分退回
        cancel_event = SagaEvent(
            event_type=EventType.ORDER_CANCELLED.value,
            saga_id=event.saga_id,
            order_id=event.order_id,
            correlation_id=event.correlation_id,
            causation_id=event.event_id,
            payload={"cancel_reason": "payment_failed"}
        )
        self.event_bus.publish("order-events", cancel_event.to_dict())

    def on_inventory_freeze_failed(self, event_data: dict):
        """监听库存冻结失败事件 → 直接取消订单（无需补偿库存）"""
        event = SagaEvent.from_dict(event_data)
        logger.warning(f"Order {event.order_id}: inventory freeze failed, cancelling")

        self.db.update("orders",
            {"status": "CANCELLED", "cancelled_at": datetime.utcnow().isoformat(),
             "cancel_reason": "inventory_insufficient"},
            {"order_id": event.order_id})


# ===== 3. 库存服务：事件驱动冻结/释放 =====

class InventoryServiceChoreography:
    """
    库存服务：监听订单创建事件冻结库存，监听补偿事件释放库存
    """

    def __init__(self, db, event_bus):
        self.db = db
        self.event_bus = event_bus
        self._register_handlers()

    def _register_handlers(self):
        self.event_bus.subscribe(EventType.ORDER_CREATED.value, self.on_order_created)
        self.event_bus.subscribe(EventType.ORDER_CANCELLED.value, self.on_order_cancelled)
        self.event_bus.subscribe(EventType.PAYMENT_FAILED.value, self.on_payment_failed)

    def on_order_created(self, event_data: dict):
        """监听订单创建事件 → 冻结库存"""
        event = SagaEvent.from_dict(event_data)
        item_id = event.payload["item_id"]
        quantity = event.payload["quantity"]

        # 幂等检查：同一 saga_id 不重复冻结
        existing = self.db.query_one(
            "SELECT * FROM inventory_freeze_log WHERE order_id = %s AND status = 'FROZEN'",
            event.order_id)
        if existing:
            logger.info(f"Inventory already frozen for order {event.order_id}, skipping")
            return

        # 冻结库存
        affected = self.db.execute(
            "UPDATE inventory SET available_stock = available_stock - %s, "
            "frozen_stock = frozen_stock + %s WHERE item_id = %s AND available_stock >= %s",
            quantity, quantity, item_id, quantity)

        if affected > 0:
            # 冻结成功 → 记录日志 → 发出 InventoryFrozenEvent
            self.db.insert("inventory_freeze_log", {
                "order_id": event.order_id, "item_id": item_id,
                "quantity": quantity, "status": "FROZEN",
                "saga_id": event.saga_id, "created_at": datetime.utcnow().isoformat()
            })
            frozen_event = SagaEvent(
                event_type=EventType.INVENTORY_FROZEN.value,
                saga_id=event.saga_id,
                order_id=event.order_id,
                correlation_id=event.correlation_id,
                causation_id=event.event_id,
                payload={"item_id": item_id, "quantity": quantity}
            )
            self.event_bus.publish("inventory-events", frozen_event.to_dict())
            logger.info(f"Inventory frozen: item={item_id}, qty={quantity}, order={event.order_id}")
        else:
            # 库存不足 → 发出冻结失败事件
            failed_event = SagaEvent(
                event_type=EventType.INVENTORY_FREEZE_FAILED.value,
                saga_id=event.saga_id,
                order_id=event.order_id,
                correlation_id=event.correlation_id,
                causation_id=event.event_id,
                payload={"item_id": item_id, "reason": "insufficient_stock"}
            )
            self.event_bus.publish("inventory-events", failed_event.to_dict())
            logger.warning(f"Inventory freeze failed: insufficient stock for item {item_id}")

    def on_order_cancelled(self, event_data: dict):
        """监听订单取消事件 → 释放库存"""
        self._release_inventory(event_data)

    def on_payment_failed(self, event_data: dict):
        """监听支付失败事件 → 释放库存（补偿）"""
        self._release_inventory(event_data)

    def _release_inventory(self, event_data: dict):
        """释放库存（幂等）"""
        event = SagaEvent.from_dict(event_data)

        # 幂等检查
        freeze_log = self.db.query_one(
            "SELECT * FROM inventory_freeze_log WHERE order_id = %s AND status = 'FROZEN'",
            event.order_id)
        if not freeze_log:
            return  # 已释放或无记录

        # 归还库存
        self.db.execute(
            "UPDATE inventory SET available_stock = available_stock + %s, "
            "frozen_stock = frozen_stock - %s WHERE item_id = %s",
            freeze_log["quantity"], freeze_log["quantity"], freeze_log["item_id"])

        # 更新日志
        self.db.update("inventory_freeze_log",
            {"status": "RELEASED"}, {"order_id": event.order_id, "status": "FROZEN"})

        # 发出释放事件
        release_event = SagaEvent(
            event_type=EventType.INVENTORY_RELEASED.value,
            saga_id=event.saga_id,
            order_id=event.order_id,
            correlation_id=event.correlation_id,
            causation_id=event.event_id,
            payload={"item_id": freeze_log["item_id"], "quantity": freeze_log["quantity"]}
        )
        self.event_bus.publish("inventory-events", release_event.to_dict())
        logger.info(f"Inventory released: order={event.order_id}")


# ===== 4. 支付服务：事件驱动支付/退款 =====

class PaymentServiceChoreography:
    """
    支付服务：监听支付命令事件，处理支付后发出成功/失败事件
    """

    def __init__(self, db, event_bus):
        self.db = db
        self.event_bus = event_bus
        self._register_handlers()

    def _register_handlers(self):
        self.event_bus.subscribe("ProcessPaymentCommand", self.on_process_payment)
        self.event_bus.subscribe(EventType.ORDER_CANCELLED.value, self.on_order_cancelled)

    def on_process_payment(self, event_data: dict):
        """监听支付命令事件 → 调用支付网关 → 发出结果事件"""
        event = SagaEvent.from_dict(event_data)

        # 幂等检查
        existing = self.db.query_one(
            "SELECT * FROM payment_log WHERE order_id = %s AND status IN ('COMPLETED', 'PROCESSING')",
            event.order_id)
        if existing and existing["status"] == "COMPLETED":
            return

        # 记录支付开始
        txn_id = f"TXN-{uuid.uuid4().hex[:12]}"
        self.db.insert("payment_log", {
            "txn_id": txn_id, "order_id": event.order_id,
            "saga_id": event.saga_id, "user_id": event.payload["user_id"],
            "amount": event.payload["amount"],
            "status": "PROCESSING", "created_at": datetime.utcnow().isoformat()
        })

        # 调用支付网关
        try:
            result = self.payment_gateway.charge(
                user_id=event.payload["user_id"],
                amount=event.payload["amount"],
                txn_id=txn_id
            )
            if result.success:
                self.db.update("payment_log",
                    {"status": "COMPLETED", "completed_at": datetime.utcnow().isoformat()},
                    {"txn_id": txn_id})
                completed_event = SagaEvent(
                    event_type=EventType.PAYMENT_COMPLETED.value,
                    saga_id=event.saga_id,
                    order_id=event.order_id,
                    correlation_id=event.correlation_id,
                    causation_id=event.event_id,
                    payload={"txn_id": txn_id, "amount": event.payload["amount"]}
                )
                self.event_bus.publish("payment-events", completed_event.to_dict())
            else:
                raise PaymentGatewayError(result.message)
        except Exception as e:
            self.db.update("payment_log",
                {"status": "FAILED", "error": str(e)}, {"txn_id": txn_id})
            failed_event = SagaEvent(
                event_type=EventType.PAYMENT_FAILED.value,
                saga_id=event.saga_id,
                order_id=event.order_id,
                correlation_id=event.correlation_id,
                causation_id=event.event_id,
                payload={"txn_id": txn_id, "reason": str(e)}
            )
            self.event_bus.publish("payment-events", failed_event.to_dict())
            logger.error(f"Payment failed: order={event.order_id}, reason={e}")

    def on_order_cancelled(self, event_data: dict):
        """监听订单取消事件 → 执行退款"""
        event = SagaEvent.from_dict(event_data)
        payment = self.db.query_one(
            "SELECT * FROM payment_log WHERE order_id = %s AND status = 'COMPLETED'",
            event.order_id)
        if not payment:
            return  # 支付未完成，无需退款

        # 执行退款
        self.payment_gateway.refund(payment["txn_id"], payment["amount"])
        self.db.update("payment_log",
            {"status": "REFUNDED", "refunded_at": datetime.utcnow().isoformat()},
            {"txn_id": payment["txn_id"]})
        logger.info(f"Payment refunded: txn={payment['txn_id']}")


# ===== 5. 事件总线（Kafka 实现） =====

class KafkaEventBus:
    """
    基于 Kafka 的事件总线
    - 发布事件到指定 topic
    - 按 event_type 订阅，路由到对应处理器
    - 保证 at-least-once 交付
    """

    def __init__(self, kafka_producer, kafka_consumer_factory):
        self.producer = kafka_producer
        self.consumer_factory = kafka_consumer_factory
        self.handlers: Dict[str, List[callable]] = {}
        self.running = False

    def publish(self, topic: str, event: dict):
        """发布事件到 Kafka topic"""
        event_type = event.get("event_type", "unknown")
        key = event.get("order_id", "")
        self.producer.send(
            topic,
            key=key.encode("utf-8"),
            value=json.dumps(event).encode("utf-8"),
            headers=[("event_type", event_type.encode("utf-8")),
                     ("saga_id", event.get("saga_id", "").encode("utf-8"))]
        )

    def subscribe(self, event_type: str, handler: callable):
        """注册事件处理器"""
        if event_type not in self.handlers:
            self.handlers[event_type] = []
        self.handlers[event_type].append(handler)

    def start_consumers(self, topics: List[str], group_id: str = "saga-choreography"):
        """启动消费者，监听指定 topic"""
        consumer = self.consumer_factory(topics, group_id)
        self.running = True

        for message in consumer:
            if not self.running:
                break
            try:
                event_data = json.loads(message.value.decode("utf-8"))
                event_type = event_data.get("event_type", "")

                # 路由到已注册的处理器
                for handler in self.handlers.get(event_type, []):
                    handler(event_data)
            except Exception as e:
                logger.error(f"Event processing error: {e}, message={message}")
                # 处理失败 → Kafka 不提交 offset，下次重试

    def stop(self):
        self.running = False
```

### 编排 vs 协作对比（详细维度）

| 维度 | 编排模式 (Orchestration) | 协作模式 (Choreography) |
|------|------------------------|------------------------|
| 控制方式 | 中心编排器统一控制流程 | 各服务自主监听事件驱动 |
| 端到端延迟 | 较高（编排器中转 + 同步调用） | 较低（事件直接流转，无中转） |
| 实现复杂度 | 中（编排器状态机 + 持久化） | 低（各服务独立，无状态机） |
| 可测试性 | 好（编排器可单元测试全流程） | 差（需集成测试多个服务） |
| 可观测性 | 好（编排器有全局视图） | 差（需通过事件链追踪） |
| 耦合度 | 较高（编排器知道所有步骤） | 低（服务只关心自己的事件） |
| 扩展性 | 新增步骤需改编排器 | 新增步骤只需订阅已有事件 |
| 故障定位 | 编排器日志可查全流程 | 需查多个服务的事件日志 |
| 适用场景 | 流程固定、步骤明确的业务 | 流程灵活、服务独立演进 |

**选型建议：**
- 订单创建→支付→发货：流程固定，用**编排模式**
- 库存扣减→推荐更新→搜索索引：各服务独立，用**协作模式**
- 混合模式：核心流程用编排，旁路通知用协作

## 可靠事件发布：Outbox 模式完整实现

在分布式事务中，"业务操作 + 事件发布"必须原子性。但数据库操作和 Kafka 发送无法在同一个事务中。Outbox 模式解决此问题：将事件存入业务数据库的 outbox 表，与业务操作在同一个事务中提交，然后由独立的中继服务异步发布。

### EventOutbox 表设计

```sql
-- 事件发件箱表：与业务表在同一个数据库中
CREATE TABLE event_outbox (
    id              BIGSERIAL PRIMARY KEY,
    event_id        VARCHAR(64) NOT NULL UNIQUE,       -- 事件唯一ID（幂等键）
    aggregate_type  VARCHAR(64) NOT NULL,              -- 聚合根类型（如 Order, Inventory）
    aggregate_id    VARCHAR(64) NOT NULL,              -- 聚合根ID（如 order_id）
    event_type      VARCHAR(128) NOT NULL,             -- 事件类型
    payload         JSONB NOT NULL,                     -- 事件内容
    metadata        JSONB,                              -- 元数据（trace_id, saga_id 等）
    status          VARCHAR(20) NOT NULL DEFAULT 'PENDING',  -- PENDING / PUBLISHED / FAILED
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    published_at    TIMESTAMP NULL,
    retry_count     SMALLINT NOT NULL DEFAULT 0,
    max_retries     SMALLINT NOT NULL DEFAULT 5,
    next_retry_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    -- 索引：中继服务查询待发布事件
    INDEX idx_outbox_status_next (status, next_retry_at),
    -- 索引：按聚合根查询事件（排查用）
    INDEX idx_outbox_aggregate (aggregate_type, aggregate_id),
    -- 索引：按事件类型查询
    INDEX idx_outbox_event_type (event_type)
);

-- 已发布事件归档表（定期从 outbox 迁移，避免 outbox 表膨胀）
CREATE TABLE event_outbox_archive (
    id              BIGSERIAL PRIMARY KEY,
    event_id        VARCHAR(64) NOT NULL,
    aggregate_type  VARCHAR(64) NOT NULL,
    aggregate_id    VARCHAR(64) NOT NULL,
    event_type      VARCHAR(128) NOT NULL,
    payload         JSONB NOT NULL,
    metadata        JSONB,
    status          VARCHAR(20) NOT NULL,
    created_at      TIMESTAMP NOT NULL,
    published_at    TIMESTAMP NOT NULL,
    archived_at     TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_archive_aggregate (aggregate_type, aggregate_id)
);
```

### 业务操作 + Outbox 写入（同一事务）

```python
class OrderServiceWithOutbox:
    """订单服务：业务操作与事件发布在同一事务中"""

    def create_order(self, order_request: dict) -> dict:
        """创建订单 + 写入 Outbox 事件 = 原子性"""
        order_id = f"ORD-{uuid.uuid4().hex[:12]}"
        saga_id = f"SAGA-{uuid.uuid4().hex[:12]}"
        event_id = str(uuid.uuid4())

        # 在同一个数据库事务中完成：
        # 1. 插入订单记录
        # 2. 插入 Outbox 事件
        with self.db.transaction() as tx:
            # 业务操作
            tx.execute("""
                INSERT INTO orders (order_id, saga_id, user_id, item_id, quantity, status, created_at)
                VALUES (%s, %s, %s, %s, %s, 'PENDING_PAYMENT', NOW())
            """, (order_id, saga_id, order_request["user_id"],
                  order_request["item_id"], order_request["quantity"]))

            # 写入 Outbox（同一事务）
            tx.execute("""
                INSERT INTO event_outbox (event_id, aggregate_type, aggregate_id, event_type, payload, metadata)
                VALUES (%s, 'Order', %s, 'OrderCreatedEvent', %s, %s)
            """, (event_id, order_id, json.dumps({
                "order_id": order_id, "saga_id": saga_id,
                "user_id": order_request["user_id"],
                "item_id": order_request["item_id"],
                "quantity": order_request["quantity"],
                "points_amount": order_request.get("points_amount", 0),
                "amount": order_request["amount"]
            }), json.dumps({
                "saga_id": saga_id, "correlation_id": saga_id
            })))

        # 事务提交后：业务数据和事件都持久化了
        # 即使应用崩溃，中继服务也会从 outbox 表发布事件
        logger.info(f"Order created with outbox event: order={order_id}, event={event_id}")
        return {"order_id": order_id, "saga_id": saga_id}

    def confirm_order(self, order_id: str, saga_id: str):
        """确认订单 + 写入确认事件"""
        with self.db.transaction() as tx:
            tx.execute("""
                UPDATE orders SET status = 'CONFIRMED', confirmed_at = NOW()
                WHERE order_id = %s AND status != 'CONFIRMED'
            """, (order_id,))

            tx.execute("""
                INSERT INTO event_outbox (event_id, aggregate_type, aggregate_id, event_type, payload, metadata)
                VALUES (%s, 'Order', %s, 'OrderConfirmedEvent', %s, %s)
            """, (str(uuid.uuid4()), order_id, json.dumps({
                "order_id": order_id, "status": "CONFIRMED"
            }), json.dumps({"saga_id": saga_id})))

    def cancel_order(self, order_id: str, saga_id: str, reason: str):
        """取消订单 + 写入取消事件"""
        with self.db.transaction() as tx:
            tx.execute("""
                UPDATE orders SET status = 'CANCELLED', cancelled_at = NOW(), cancel_reason = %s
                WHERE order_id = %s AND status NOT IN ('CANCELLED', 'CONFIRMED')
            """, (reason, order_id))

            tx.execute("""
                INSERT INTO event_outbox (event_id, aggregate_type, aggregate_id, event_type, payload, metadata)
                VALUES (%s, 'Order', %s, 'OrderCancelledEvent', %s, %s)
            """, (str(uuid.uuid4()), order_id, json.dumps({
                "order_id": order_id, "cancel_reason": reason
            }), json.dumps({"saga_id": saga_id})))
```

### EventRelay：中继服务（轮询 Outbox 并发布到 Kafka）

```python
class EventRelay:
    """
    事件中继服务：定期扫描 outbox 表，将待发布事件发送到 Kafka

    设计要点：
    1. 幂等发布：Kafka 生产者启用幂等性 + event_id 去重
    2. 有序发布：同一聚合根的事件按创建顺序发布（Kafka 分区键 = aggregate_id）
    3. 失败重试：发布失败后指数退避重试
    4. 归档清理：已发布事件定期迁移到归档表
    """

    POLL_INTERVAL = 1          # 轮询间隔（秒）
    BATCH_SIZE = 100           # 每批处理事件数
    MAX_RETRIES = 5            # 最大重试次数
    RETRY_DELAYS = [5, 15, 30, 60, 300]  # 重试延迟（秒）

    def __init__(self, db, kafka_producer):
        self.db = db
        self.producer = kafka_producer

    def start(self):
        """启动中继服务"""
        logger.info("EventRelay starting...")
        while True:
            try:
                self.relay_pending_events()
                self.archive_published_events()
            except Exception as e:
                logger.error(f"EventRelay error: {e}")
            time.sleep(self.POLL_INTERVAL)

    def relay_pending_events(self):
        """扫描并发布待发送事件"""
        events = self.db.query("""
            SELECT * FROM event_outbox
            WHERE status = 'PENDING' AND next_retry_at <= NOW()
            ORDER BY created_at ASC
            LIMIT %s
        """, (self.BATCH_SIZE,))

        for event in events:
            try:
                self._publish_event(event)
                # 标记为已发布
                self.db.execute("""
                    UPDATE event_outbox
                    SET status = 'PUBLISHED', published_at = NOW()
                    WHERE id = %s AND status = 'PENDING'
                """, (event["id"],))
                logger.info(f"Event published: event_id={event['event_id']}, "
                           f"type={event['event_type']}")
            except Exception as e:
                logger.error(f"Event publish failed: event_id={event['event_id']}, error={e}")
                self._schedule_retry(event, str(e))

    def _publish_event(self, event: dict):
        """发布事件到 Kafka"""
        # Topic 策略：按聚合根类型分 topic
        topic = f"{event['aggregate_type'].lower()}-events"
        # 分区键 = aggregate_id，保证同一聚合根的事件有序
        partition_key = event["aggregate_id"]

        # 构建 Kafka 消息
        message_value = json.dumps({
            "event_id": event["event_id"],
            "event_type": event["event_type"],
            "aggregate_type": event["aggregate_type"],
            "aggregate_id": event["aggregate_id"],
            "payload": event["payload"],
            "metadata": event["metadata"],
            "created_at": event["created_at"].isoformat()
        })

        # 发送到 Kafka（启用幂等生产者）
        future = self.producer.send(
            topic,
            key=partition_key.encode("utf-8"),
            value=message_value.encode("utf-8"),
            headers=[
                ("event_id", event["event_id"].encode("utf-8")),
                ("event_type", event["event_type"].encode("utf-8"))
            ]
        )
        # 等待确认（同步模式，确保发送成功）
        future.get(timeout=10)

    def _schedule_retry(self, event: dict, error: str):
        """调度重试"""
        retry_count = event["retry_count"] + 1
        if retry_count >= self.MAX_RETRIES:
            self.db.execute("""
                UPDATE event_outbox
                SET status = 'FAILED', retry_count = %s
                WHERE id = %s
            """, (retry_count, event["id"]))
            logger.error(f"Event permanently failed: event_id={event['event_id']}")
            return

        delay = self.RETRY_DELAYS[min(retry_count - 1, len(self.RETRY_DELAYS) - 1)]
        self.db.execute("""
            UPDATE event_outbox
            SET retry_count = %s, next_retry_at = NOW() + INTERVAL '%s seconds'
            WHERE id = %s
        """, (retry_count, delay, event["id"]))

    def archive_published_events(self):
        """归档已发布事件（定期清理 outbox 表）"""
        # 迁移 7 天前的已发布事件到归档表
        self.db.execute("""
            INSERT INTO event_outbox_archive
                (event_id, aggregate_type, aggregate_id, event_type, payload,
                 metadata, status, created_at, published_at)
            SELECT event_id, aggregate_type, aggregate_id, event_type, payload,
                   metadata, status, created_at, published_at
            FROM event_outbox
            WHERE status = 'PUBLISHED' AND published_at < NOW() - INTERVAL '7 days'
        """)
        self.db.execute("""
            DELETE FROM event_outbox
            WHERE status = 'PUBLISHED' AND published_at < NOW() - INTERVAL '7 days'
        """)
```

### 幂等消费者：去重表完整实现

```python
class IdempotentConsumer:
    """
    幂等消费者：确保同一事件不会被重复处理

    去重表存储已消费的 message_id，消费前检查是否已处理
    """

    def __init__(self, db, consumer_group: str):
        self.db = db
        self.consumer_group = consumer_group

    def consume(self, message, handler: callable):
        """
        幂等消费流程：
        1. 检查 message_id 是否已消费
        2. 如果已消费 → 返回上次结果
        3. 如果未消费 → 执行业务逻辑 + 记录消费日志（同一事务）
        4. 提交 Kafka offset
        """
        event_id = message.headers.get("event_id")
        if not event_id:
            logger.warning("Message without event_id, skipping")
            return

        # 幂等检查
        consumed = self.db.query_one(
            "SELECT * FROM consumed_event_log WHERE event_id = %s AND consumer_group = %s",
            event_id, self.consumer_group)

        if consumed:
            if consumed["status"] == "COMPLETED":
                logger.info(f"Event already consumed: {event_id}, skipping")
                return
            elif consumed["status"] == "PROCESSING":
                logger.warning(f"Event being processed: {event_id}, skipping")
                return
            elif consumed["status"] == "FAILED":
                # 上次失败，允许重试
                pass

        # 标记为处理中
        self.db.execute("""
            INSERT INTO consumed_event_log (event_id, consumer_group, status, created_at)
            VALUES (%s, %s, 'PROCESSING', NOW())
            ON CONFLICT (event_id, consumer_group) DO UPDATE SET status = 'PROCESSING'
        """, (event_id, self.consumer_group))

        try:
            # 执行业务逻辑
            result = handler(message)

            # 标记为已完成
            self.db.execute("""
                UPDATE consumed_event_log
                SET status = 'COMPLETED', completed_at = NOW(), result = %s
                WHERE event_id = %s AND consumer_group = %s
            """, (json.dumps(result) if result else None, event_id, self.consumer_group))

            return result
        except Exception as e:
            self.db.execute("""
                UPDATE consumed_event_log
                SET status = 'FAILED', error = %s
                WHERE event_id = %s AND consumer_group = %s
            """, (str(e), event_id, self.consumer_group))
            raise
```

```sql
-- 幂等消费者去重表
CREATE TABLE consumed_event_log (
    event_id        VARCHAR(64) NOT NULL,
    consumer_group  VARCHAR(64) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'PROCESSING',  -- PROCESSING / COMPLETED / FAILED
    result          JSONB,
    error           TEXT,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    completed_at    TIMESTAMP NULL,

    PRIMARY KEY (event_id, consumer_group)
);

CREATE INDEX idx_consumed_created ON consumed_event_log(created_at);
```

## 异常场景补充

### 场景 6：僵尸 Saga（卡在运行状态无法推进）

```
触发：
  Saga 实例状态为 EXECUTING/RUNNING，但超过 30 分钟无任何步骤推进
  可能原因：
  1. 编排器在步骤间崩溃，恢复机制未触发
  2. 步骤服务响应极慢（如支付网关挂起），未超时也未返回
  3. 状态更新 SQL 因主从延迟未生效，恢复服务读到旧状态

检测：
  1. 定时扫描：SELECT * FROM saga_instances
     WHERE status IN ('EXECUTING', 'RUNNING')
       AND updated_at < NOW() - INTERVAL '30 minutes'
  2. 步骤心跳：每个步骤 EXECUTING 状态超过 10 分钟 → 告警
  3. 仪表盘显示"僵尸 Saga 数量"指标

处理：
  1. 恢复服务接管：重新查询各步骤实际状态
  2. 查询步骤服务的真实状态（调用 query_status API）
  3. 根据查询结果：
     - 步骤已完成 → 更新步骤状态，继续推进
     - 步骤未执行 → 重新执行该步骤
     - 步骤正在处理 → 等待并设置更短的超时
     - 步骤状态不确定 → 安全起见，进入补偿流程
  4. 如果僵尸 Saga 反复出现 → 检查恢复服务的调度是否正常

预防：
  1. 每个步骤设置合理的超时（支付 30s，库存 10s）
  2. 编排器启动时立即扫描并恢复
  3. 步骤执行时更新 updated_at，防止误判
  4. 步骤心跳机制：长时间步骤定期更新心跳时间戳
```

### 场景 7：事件重复消费导致双重操作

```
触发：
  Kafka 消费者处理完事件后，提交 offset 失败（如 rebalance）
  → 重启后从上次提交的 offset 重新消费
  → InventoryFrozenEvent 被消费两次
  → 库存被冻结两次（数量错误）

检测：
  1. 幂等消费者去重表拦截：event_id 已存在于 consumed_event_log
  2. 业务层幂等：inventory_freeze_log 中 (order_id, item_id) 唯一约束冲突
  3. 数据校验：定期核对 frozen_stock 与 freeze_log 汇总是否一致

处理：
  1. 消费端幂等：检查 consumed_event_log → 已消费则跳过
  2. 业务层幂等：INSERT ... ON DUPLICATE KEY UPDATE → 忽略重复
  3. 数据修复：如果双重冻结已发生
     a. 查询 freeze_log 找出重复记录
     b. 计算正确的冻结数量
     c. 修正 inventory 表的 available_stock 和 frozen_stock
     d. 删除多余的 freeze_log 记录

预防：
  1. 所有消费者必须实现幂等逻辑
  2. Kafka 生产者启用幂等（enable.idempotence=true）
  3. 消费者使用手动提交 offset（处理完再提交）
  4. 定期数据校验任务：比对 freeze_log 与 inventory 一致性
```

## 分布式锁实现

```python
class DistributedLock:
    """Redis 分布式锁（带自动续期）"""

    def acquire(self, lock_key, holder_id, ttl_seconds=30):
        """获取分布式锁"""
        result = self.redis.set(lock_key, holder_id, nx=True, ex=ttl_seconds)
        if not result:
            current_holder = self.redis.get(lock_key)
            if current_holder == holder_id:
                # 已持有锁，续期
                self.redis.expire(lock_key, ttl_seconds)
                return True
            return False  # 被其他进程持有
        return True

    def release(self, lock_key, holder_id):
        """释放锁（只释放自己持有的）"""
        # Lua 脚本保证原子性：只有持有者才能释放
        script = """
        if redis.call('get', KEYS[1]) == ARGV[1] then
            return redis.call('del', KEYS[1])
        else
            return 0
        end
        """
        self.redis.eval(script, 1, lock_key, holder_id)

    def auto_renew(self, lock_key, holder_id, ttl_seconds=30):
        """自动续期（后台线程）"""
        while True:
            sleep(ttl_seconds / 3)  # 每 TTL/3 续期一次
            if self.redis.get(lock_key) == holder_id:
                self.redis.expire(lock_key, ttl_seconds)
            else:
                break  # 锁已被释放或过期
```

## 事务隔离级别对比

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 性能 | 适用 |
|---------|------|----------|------|------|------|
| READ UNCOMMITTED | 可能 | 可能 | 可能 | 最高 | 几乎不用 |
| READ COMMITTED | 不会 | 可能 | 可能 | 高 | PostgreSQL 默认 |
| REPEATABLE READ | 不会 | 不会 | 可能 | 中 | MySQL 默认 |
| SERIALIZABLE | 不会 | 不会 | 不会 | 最低 | 财务结账 |

**分布式事务中的隔离性保证：**

| 方案 | 隔离性 | 说明 |
|------|--------|------|
| 2PC | SERIALIZABLE | 全局锁，强隔离 |
| TCC | READ COMMITTED | Try 阶段冻结资源，提供逻辑隔离 |
| Saga | 无隔离 | 可能出现脏读，需要业务层处理 |
| 本地消息表 | 无隔离 | 最终一致性，中间状态可见 |

## 性能分析详细数据

**各方案吞吐量与延迟对比：**

| 方案 | 单次延迟 | 峰值 TPS | 资源开销 | 一致性 |
|------|---------|---------|---------|--------|
| 2PC (XA) | 50-200ms | 500 | 高（锁等待） | 强一致 |
| Saga 编排 | 5-10s | 5000 | 中 | 最终一致 |
| Saga 协作 | 3-8s | 3000 | 低 | 最终一致 |
| TCC | 300-500ms | 2000 | 高（3次调用） | 准一致 |
| 本地消息表 | 200-500ms | 3000 | 低 | 最终一致 |

**月度基础设施成本：**

| 组件 | 规格 | 数量 | 月成本 |
|------|------|------|-------|
| Saga 编排器 | 4c8G | 5 台 | ¥2 万 |
| Kafka | 8c32G + 1TB | 6 节点 | ¥3 万 |
| MySQL | 8c64G + SSD | 主从 4 台 | ¥4 万 |
| Redis | 8c32G | 3 节点 | ¥1.5 万 |
| **合计** | | | **¥10.5 万** |

## Saga 编排器完整实现

```python
class SagaOrchestrator:
    """Saga 编排器：步骤定义 + 补偿 + 超时"""

    def define_saga(self, name, steps):
        """定义 Saga"""
        return SagaDefinition(name=name, steps=[
            SagaStep(action=step["action"], compensate=step["compensate"],
                     timeout_seconds=step.get("timeout", 30))
            for step in steps
        ])

    def execute(self, saga_definition, context):
        """执行 Saga"""
        saga_id = str(uuid4())
        instance = SagaInstance(
            saga_id=saga_id,
            definition=saga_definition,
            context=context,
            status="running",
            completed_steps=[],
            started_at=now()
        )

        self.db.insert("saga_instances", {
            "saga_id": saga_id,
            "definition_name": saga_definition.name,
            "context": json.dumps(context),
            "status": "running",
            "started_at": now()
        })

        # 逐步执行
        for i, step in enumerate(saga_definition.steps):
            try:
                # 设置步骤超时
                result = self._execute_with_timeout(
                    step.action, context, step.timeout_seconds)

                instance.completed_steps.append(i)
                context.update(result or {})

                self.db.insert("saga_step_logs", {
                    "saga_id": saga_id, "step_index": i,
                    "action": step.action.__name__, "status": "completed",
                    "result": json.dumps(result or {}),
                    "executed_at": now()
                })

            except Exception as e:
                # 步骤失败 → 开始补偿
                instance.status = "compensating"
                self.db.update("saga_instances",
                    {"status": "compensating"}, {"saga_id": saga_id})

                self._compensate(instance, i, str(e))
                return {"saga_id": saga_id, "status": "failed",
                        "failed_step": i, "error": str(e)}

        instance.status = "completed"
        self.db.update("saga_instances",
            {"status": "completed", "completed_at": now()},
            {"saga_id": saga_id})

        return {"saga_id": saga_id, "status": "completed", "context": context}

    def _compensate(self, instance, failed_step, error):
        """执行补偿（逆序）"""
        for i in reversed(instance.completed_steps):
            step = instance.definition.steps[i]
            try:
                step.compensate(instance.context)
                self.db.insert("saga_step_logs", {
                    "saga_id": instance.saga_id, "step_index": i,
                    "action": step.compensate.__name__, "status": "compensated",
                    "executed_at": now()
                })
            except Exception as comp_error:
                # 补偿失败 → 人工介入
                self.db.insert("saga_step_logs", {
                    "saga_id": instance.saga_id, "step_index": i,
                    "action": step.compensate.__name__, "status": "compensation_failed",
                    "error": str(comp_error), "executed_at": now()
                })
                self.alert(f"Saga {instance.saga_id} 补偿失败 step {i}: {comp_error}")

    def _execute_with_timeout(self, action, context, timeout):
        """带超时的步骤执行"""
        import concurrent.futures
        with concurrent.futures.ThreadPoolExecutor() as executor:
            future = executor.submit(action, context)
            return future.result(timeout=timeout)


class SagaDefinition:
    def __init__(self, name, steps):
        self.name = name
        self.steps = steps

class SagaStep:
    def __init__(self, action, compensate, timeout_seconds=30):
        self.action = action
        self.compensate = compensate
        self.timeout_seconds = timeout_seconds

class SagaInstance:
    def __init__(self, saga_id, definition, context, status, completed_steps, started_at):
        self.saga_id = saga_id
        self.definition = definition
        self.context = context
        self.status = status
        self.completed_steps = completed_steps
        self.started_at = started_at
```

## 事务 Outbox 模式

```python
class TransactionalOutbox:
    """事务 Outbox：双写保证 + 可靠投递"""

    def send_with_outbox(self, session, event):
        """在同一个数据库事务中写入业务数据和 Outbox"""
        # 业务操作和 Outbox 写入在同一个事务中
        session.add(BusinessEntity(...))  # 业务数据
        session.add(OutboxMessage(
            id=str(uuid4()),
            aggregate_type=event["aggregate_type"],
            aggregate_id=event["aggregate_id"],
            event_type=event["event_type"],
            payload=json.dumps(event["payload"]),
            created_at=now(),
            status="pending"
        ))
        # 事务提交后，业务数据和 Outbox 消息一起持久化

    def scan_and_publish(self, batch_size=100):
        """扫描 Outbox 并发布到消息队列"""
        # 1. 获取待发送消息（带租约，防止重复消费）
        messages = self.db.query(
            "SELECT * FROM outbox_messages WHERE status = 'pending' "
            "AND (leased_until IS NULL OR leased_until < NOW()) "
            "ORDER BY created_at ASC LIMIT %s FOR UPDATE SKIP LOCKED",
            batch_size)

        for msg in messages:
            try:
                # 2. 发布到 Kafka
                self.kafka.produce(msg["event_type"], msg["payload"],
                    key=f"{msg['aggregate_type']}:{msg['aggregate_id']}")

                # 3. 标记为已发送
                self.db.update("outbox_messages",
                    {"status": "sent", "sent_at": now()},
                    {"id": msg["id"]})

            except Exception as e:
                # 发送失败 → 释放租约，下次重试
                self.db.update("outbox_messages",
                    {"leased_until": None}, {"id": msg["id"]})

    def compact(self, before_date):
        """压缩 Outbox 表（删除已发送的旧消息）"""
        self.db.execute(
            "DELETE FROM outbox_messages WHERE status = 'sent' "
            "AND sent_at < %s", before_date)
```

## 异常场景补充

### 场景：Saga 编排器崩溃导致部分补偿

```
触发：Saga 执行到 step 3 时编排器崩溃 → step 1,2 已完成但未补偿
检测：
  1. Saga 实例 status=running 但超过超时时间 → 崩溃
  2. 重启后扫描未完成 Saga → 恢复
处理：
  1. 重启编排器后扫描 status=running/compensating 的 Saga
  2. running → 判断最后完成的 step → 继续或补偿
  3. compensating → 继续执行补偿
  4. 补偿失败 → 人工介入
预防：Saga 状态持久化 + 重启恢复 + 定期扫描超时 Saga
```

### 场景：Outbox 消息重复投递

```
触发：Kafka 生产者重试 → 同一 Outbox 消息被发送两次
检测：
  1. 消费端检测到重复消息 ID → 告警
  2. 幂等检查：消息 ID 已处理 → 跳过
处理：
  1. 消费端幂等：基于消息 ID 去重
  2. 幂等表：INSERT IGNORE (message_id) → 重复则跳过
预防：消息幂等消费 + 消息 ID 去重表
```

## TCC 框架完整实现

```python
class TCCTransactionManager:
    """TCC 事务管理器：Try-Confirm-Cancel"""

    def begin_tcc(self, participants, context):
        """开始 TCC 事务"""
        tx_id = str(uuid4())
        self.db.insert("tcc_transactions", {
            "tx_id": tx_id,
            "participants": json.dumps([p["name"] for p in participants]),
            "context": json.dumps(context),
            "status": "trying",
            "started_at": now()
        })

        # Try 阶段：预留资源
        tried = []
        for participant in participants:
            try:
                result = participant["try"](context)
                tried.append(participant["name"])
                self.db.insert("tcc_logs", {
                    "tx_id": tx_id, "participant": participant["name"],
                    "phase": "try", "status": "success",
                    "result": json.dumps(result or {}),
                    "timestamp": now()
                })
            except Exception as e:
                # Try 失败 → Cancel 已预留的资源
                self._cancel_all(tx_id, tried, context)
                return {"tx_id": tx_id, "status": "cancelled",
                        "failed_at": participant["name"], "error": str(e)}

        # 所有 Try 成功 → Confirm
        self.db.update("tcc_transactions",
            {"status": "confirming"}, {"tx_id": tx_id})
        confirmed = self._confirm_all(tx_id, participants, context)

        if confirmed:
            self.db.update("tcc_transactions",
                {"status": "confirmed", "confirmed_at": now()},
                {"tx_id": tx_id})
            return {"tx_id": tx_id, "status": "confirmed"}
        else:
            return {"tx_id": tx_id, "status": "confirm_failed"}

    def _confirm_all(self, tx_id, participants, context):
        """Confirm 所有参与者"""
        for participant in participants:
            try:
                participant["confirm"](context)
                self.db.insert("tcc_logs", {
                    "tx_id": tx_id, "participant": participant["name"],
                    "phase": "confirm", "status": "success",
                    "timestamp": now()
                })
            except Exception as e:
                # Confirm 失败 → 重试（Confirm 必须最终成功）
                self.db.insert("tcc_logs", {
                    "tx_id": tx_id, "participant": participant["name"],
                    "phase": "confirm", "status": "failed",
                    "error": str(e), "timestamp": now()
                })
                self._schedule_confirm_retry(tx_id, participant, context)
                return False
        return True

    def _cancel_all(self, tx_id, tried_participants, context):
        """Cancel 已预留的资源"""
        for name in tried_participants:
            try:
                participant = next(p for p in context["_participants"] if p["name"] == name)
                participant["cancel"](context)
                self.db.insert("tcc_logs", {
                    "tx_id": tx_id, "participant": name,
                    "phase": "cancel", "status": "success",
                    "timestamp": now()
                })
            except Exception as e:
                self.db.insert("tcc_logs", {
                    "tx_id": tx_id, "participant": name,
                    "phase": "cancel", "status": "failed",
                    "error": str(e), "timestamp": now()
                })
                self.alert(f"TCC Cancel 失败: tx={tx_id} participant={name}")

    def _schedule_confirm_retry(self, tx_id, participant, context):
        """调度 Confirm 重试"""
        for attempt in range(3):
            try:
                participant["confirm"](context)
                self.db.insert("tcc_logs", {
                    "tx_id": tx_id, "participant": participant["name"],
                    "phase": "confirm_retry", "status": "success",
                    "attempt": attempt + 1, "timestamp": now()
                })
                return
            except Exception:
                time.sleep(2 ** attempt)  # 指数退避
        self.alert(f"TCC Confirm 重试耗尽: tx={tx_id}")
```

## 分布式锁实现

```python
class DistributedLock:
    """分布式锁：RedLock 算法"""

    LOCK_TTL = 10000  # 10 秒
    RETRY_DELAY = 200  # 200ms
    MAX_RETRIES = 10

    def acquire(self, resource, ttl_ms=None):
        """获取锁（RedLock: 多数派节点确认）"""
        ttl = ttl_ms or self.LOCK_TTL
        token = str(uuid4())
        acquired = 0
        start = time.time()

        for node in self.redis_nodes:
            try:
                result = node.set(
                    f"lock:{resource}", token,
                    nx=True, px=ttl)
                if result:
                    acquired += 1
            except Exception:
                continue

        elapsed = int((time.time() - start) * 1000)
        # 多数派 + 未超时
        if acquired >= len(self.redis_nodes) // 2 + 1 and elapsed < ttl:
            return {"acquired": True, "token": token,
                    "validity_ms": ttl - elapsed}
        else:
            # 获取失败 → 释放已获取的锁
            self._release_all(resource, token)
            return {"acquired": False}

    def renew(self, resource, token, ttl_ms=None):
        """续期锁（Watchdog 机制）"""
        # 使用 Lua 脚本保证原子性：只有持有者才能续期
        lua_script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("pexpire", KEYS[1], ARGV[2])
        else
            return 0
        end
        """
        ttl = ttl_ms or self.LOCK_TTL
        renewed = 0
        for node in self.redis_nodes:
            try:
                result = node.eval(lua_script, 1,
                    f"lock:{resource}", token, ttl)
                if result:
                    renewed += 1
            except Exception:
                continue
        return renewed >= len(self.redis_nodes) // 2 + 1

    def release(self, resource, token):
        """释放锁（Lua 脚本保证原子性）"""
        lua_script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """
        for node in self.redis_nodes:
            try:
                node.eval(lua_script, 1, f"lock:{resource}", token)
            except Exception:
                continue
```

## 异常场景补充

### 场景：TCC Confirm 阶段部分失败

```
触发：3 个参与者中 2 个 Confirm 成功，1 个失败 → 事务不一致
检测：
  1. TCC 日志中 confirm 状态为 failed → 部分失败
  2. 事务状态为 confirm_failed → 需要人工介入
处理：
  1. 自动重试 Confirm（指数退避，最多 3 次）
  2. 重试仍失败 → 人工介入（查看失败原因）
  3. Confirm 必须最终成功（不能 Cancel）
预防：Confirm 幂等 + 自动重试 + 人工兜底
```

### 场景：分布式锁脑裂

```
触发：网络分区 → 节点 A 获得锁 → 分区恢复前锁过期 → 节点 B 也获得锁
检测：
  1. 同一资源存在两个持有者 → 脑裂
  2. 锁续期失败 → 可能脑裂
处理：
  1. 增加 fencing token（单调递增）：资源端只接受更大 token
  2. 节点 A 持有旧 token → 写入被拒绝
  3. 节点 B 持有新 token → 写入成功
预防：fencing token + 资源端校验 + 锁 TTL 合理设置
```

## 幂等性保证框架

### 幂等性键生成策略

```python
import hashlib
import uuid
from enum import Enum
from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import Optional, Any, Dict


class OperationType(Enum):
    """操作类型枚举"""
    ORDER_CREATE = "order_create"
    PAYMENT_PROCESS = "payment_process"
    INVENTORY_DEDUCT = "inventory_deduct"
    POINTS_CONSUME = "points_consume"
    ORDER_CANCEL = "order_cancel"
    REFUND_PROCESS = "refund_process"


@dataclass
class IdempotencyKey:
    """幂等键对象"""
    key: str                          # 最终幂等键
    client_uuid: str                  # 客户端生成的 UUID
    operation_hash: str               # 操作内容哈希
    operation_type: OperationType     # 操作类型
    ttl_seconds: int                  # 去重窗口时长
    created_at: datetime              # 创建时间


class IdempotencyKeyGenerator:
    """
    幂等键生成器
    策略：客户端 UUID + 操作内容哈希，确保：
    1. 客户端 UUID 防止重试产生重复键
    2. 操作内容哈希防止不同操作产生相同键
    """

    # 每种操作类型的去重窗口 TTL（秒）
    OPERATION_TTL = {
        OperationType.ORDER_CREATE: 86400,       # 下单：24 小时
        OperationType.PAYMENT_PROCESS: 7200,     # 支付：2 小时
        OperationType.INVENTORY_DEDUCT: 3600,    # 库存扣减：1 小时
        OperationType.POINTS_CONSUME: 3600,      # 积分消耗：1 小时
        OperationType.ORDER_CANCEL: 7200,        # 取消订单：2 小时
        OperationType.REFUND_PROCESS: 86400,     # 退款：24 小时
    }

    def generate(self, operation_type: OperationType,
                 operation_params: Dict[str, Any],
                 client_uuid: Optional[str] = None) -> IdempotencyKey:
        """
        生成幂等键

        Args:
            operation_type: 操作类型
            operation_params: 操作参数（如 order_id, amount 等）
            client_uuid: 客户端生成的 UUID（推荐由调用方生成，保证重试时不变）
        """
        # 步骤 1：客户端 UUID（如果未提供则自动生成）
        if client_uuid is None:
            client_uuid = str(uuid.uuid4())

        # 步骤 2：操作内容哈希
        # 将操作类型 + 参数排序后序列化，计算 SHA256
        sorted_params = self._sort_dict(operation_params)
        content = f"{operation_type.value}:{sorted_params}"
        operation_hash = hashlib.sha256(content.encode()).hexdigest()[:16]

        # 步骤 3：组合最终幂等键
        # 格式：idem:{operation_type}:{client_uuid}:{operation_hash}
        key = f"idem:{operation_type.value}:{client_uuid}:{operation_hash}"

        ttl = self.OPERATION_TTL.get(operation_type, 3600)

        return IdempotencyKey(
            key=key,
            client_uuid=client_uuid,
            operation_hash=operation_hash,
            operation_type=operation_type,
            ttl_seconds=ttl,
            created_at=datetime.utcnow()
        )

    def _sort_dict(self, d: Dict[str, Any]) -> str:
        """字典排序序列化，确保相同参数产生相同哈希"""
        import json
        return json.dumps(d, sort_keys=True, ensure_ascii=False)


class IdempotencyDedupStore:
    """
    服务端去重存储
    双层存储：Redis SET（快速去重）+ DB 唯一约束（持久化保障）
    """

    def __init__(self, redis_client, db_pool):
        self.redis = redis_client
        self.db = db_pool

    async def check_and_record(self, idem_key: IdempotencyKey) -> tuple[bool, Optional[Dict]]:
        """
        检查并记录幂等键
        返回：(是否重复, 已有结果)
        - (False, None) → 首次请求，可以执行
        - (True, result) → 重复请求，返回缓存结果
        - (True, None)   → 重复请求但结果尚未缓存（请求正在处理中）
        """
        redis_key = f"idem_set:{idem_key.operation_type.value}"
        member = idem_key.key

        # 第一层：Redis SET 原子检查 + 加入
        # SADD 返回 1 表示新加入，0 表示已存在
        added = self.redis.sadd(redis_key, member)

        if added == 0:
            # 键已存在 → 重复请求
            # 尝试从结果缓存获取
            cached = await self._get_cached_result(idem_key.key)
            return True, cached

        # 第二层：DB 唯一约束双重保障
        try:
            await self.db.execute(
                """INSERT INTO idempotency_keys
                   (idempotency_key, operation_type, client_uuid,
                    operation_hash, status, created_at, expires_at)
                   VALUES (%s, %s, %s, %s, 'pending', %s, %s)""",
                idem_key.key,
                idem_key.operation_type.value,
                idem_key.client_uuid,
                idem_key.operation_hash,
                idem_key.created_at,
                idem_key.created_at + timedelta(seconds=idem_key.ttl_seconds)
            )
        except Exception as e:
            if "duplicate key" in str(e).lower() or "unique constraint" in str(e).lower():
                # DB 层也检测到重复 → 以 DB 为准
                # 修正 Redis 中的记录（防止 Redis 和 DB 不一致）
                cached = await self._get_cached_result(idem_key.key)
                return True, cached
            else:
                # 其他 DB 错误 → 记录但允许继续（Redis 已记录）
                await self._log_db_error(idem_key.key, str(e))

        # 设置 Redis 键的过期时间
        self.redis.expire(redis_key, idem_key.ttl_seconds)

        return False, None

    async def record_result(self, idem_key: IdempotencyKey,
                            result: Dict, status: str = "completed"):
        """记录幂等结果，供重复请求直接返回"""
        import json

        # 存入 Redis 结果缓存
        cache_key = f"idem_result:{idem_key.key}"
        self.redis.setex(
            cache_key,
            idem_key.ttl_seconds,
            json.dumps(result, ensure_ascii=False)
        )

        # 更新 DB 状态
        await self.db.execute(
            """UPDATE idempotency_keys
               SET status = %s, result = %s, completed_at = %s
               WHERE idempotency_key = %s""",
            status,
            json.dumps(result, ensure_ascii=False),
            datetime.utcnow(),
            idem_key.key
        )

    async def _get_cached_result(self, key: str) -> Optional[Dict]:
        """从缓存获取幂等结果"""
        import json

        # 先查 Redis
        cache_key = f"idem_result:{key}"
        cached = self.redis.get(cache_key)
        if cached:
            return json.loads(cached)

        # Redis 没有 → 查 DB
        row = await self.db.fetch_one(
            """SELECT result FROM idempotency_keys
               WHERE idempotency_key = %s AND status = 'completed'""",
            key
        )
        if row and row["result"]:
            return json.loads(row["result"])

        return None

    async def _log_db_error(self, key: str, error: str):
        """记录 DB 错误"""
        await self.db.execute(
            """INSERT INTO idempotency_errors
               (idempotency_key, error_message, created_at)
               VALUES (%s, %s, %s)""",
            key, error, datetime.utcnow()
        )


class IdempotencyKeyCollisionDetector:
    """
    幂等键碰撞检测与告警
    当不同操作产生相同幂等键时（极低概率但必须处理）
    """

    def __init__(self, redis_client, alert_service):
        self.redis = redis_client
        self.alert = alert_service

    async def detect_collision(self, idem_key: IdempotencyKey,
                               current_params: Dict) -> bool:
        """
        检测键碰撞：相同幂等键但操作参数不同
        """
        # 获取该键对应的原始参数
        param_key = f"idem_params:{idem_key.key}"
        stored_params = self.redis.get(param_key)

        if stored_params is None:
            # 首次见到此键 → 记录参数
            import json
            self.redis.setex(
                param_key,
                idem_key.ttl_seconds,
                json.dumps(current_params, sort_keys=True, ensure_ascii=False)
            )
            return False

        # 比较参数
        import json
        stored = json.loads(stored_params)
        current_serialized = json.dumps(
            current_params, sort_keys=True, ensure_ascii=False)

        if stored_params.decode() != current_serialized:
            # 碰撞！不同操作产生相同键
            await self.alert.send_critical(
                title="幂等键碰撞告警",
                message=(
                    f"检测到幂等键碰撞！\n"
                    f"键：{idem_key.key}\n"
                    f"原始参数：{stored}\n"
                    f"当前参数：{current_params}\n"
                    f"操作类型：{idem_key.operation_type.value}\n"
                    f"请立即排查！"
                )
            )
            # 记录碰撞事件
            self.redis.rpush("idem_collisions", json.dumps({
                "key": idem_key.key,
                "original_params": stored,
                "current_params": current_params,
                "operation_type": idem_key.operation_type.value,
                "detected_at": datetime.utcnow().isoformat()
            }))
            return True

        return False


# 使用示例
async def process_order_with_idempotency(order_data: dict):
    """
    带幂等保证的订单处理
    """
    key_gen = IdempotencyKeyGenerator()
    dedup_store = IdempotencyDedupStore(redis_cli, db_pool)
    collision_detector = IdempotencyKeyCollisionDetector(
        redis_cli, alert_svc)

    # 1. 生成幂等键
    idem_key = key_gen.generate(
        operation_type=OperationType.ORDER_CREATE,
        operation_params={
            "user_id": order_data["user_id"],
            "product_id": order_data["product_id"],
            "quantity": order_data["quantity"],
            "total_amount": order_data["total_amount"]
        },
        client_uuid=order_data.get("idempotency_uuid")
    )

    # 2. 碰撞检测
    is_collision = await collision_detector.detect_collision(
        idem_key, order_data)
    if is_collision:
        raise Exception("幂等键碰撞，拒绝执行操作")

    # 3. 去重检查
    is_duplicate, cached_result = await dedup_store.check_and_record(idem_key)
    if is_duplicate:
        if cached_result:
            return cached_result  # 返回缓存结果
        else:
            # 请求正在处理中
            return {"status": "processing", "message": "请求处理中，请稍后查询"}

    # 4. 执行业务逻辑
    try:
        result = await create_order(order_data)
        # 5. 记录成功结果
        await dedup_store.record_result(idem_key, result, "completed")
        return result
    except Exception as e:
        # 记录失败结果
        await dedup_store.record_result(
            idem_key,
            {"status": "failed", "error": str(e)},
            "failed"
        )
        raise
```

### DB 表结构

```sql
-- 幂等键表（唯一约束保障持久化去重）
CREATE TABLE idempotency_keys (
    id              BIGSERIAL PRIMARY KEY,
    idempotency_key VARCHAR(128) NOT NULL UNIQUE,  -- 幂等键唯一约束
    operation_type  VARCHAR(32)  NOT NULL,
    client_uuid     VARCHAR(36)  NOT NULL,
    operation_hash  VARCHAR(16)  NOT NULL,
    status          VARCHAR(16)  NOT NULL DEFAULT 'pending',
                   -- pending / completed / failed
    result          JSONB,        -- 缓存的执行结果
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ  NOT NULL
);

-- 索引：按操作类型查询 + 过期清理
CREATE INDEX idx_idem_operation ON idempotency_keys(operation_type, created_at);
CREATE INDEX idx_idem_expires ON idempotency_keys(expires_at)
    WHERE status = 'completed';

-- 定期清理过期记录（pg_cron）
-- SELECT cron.schedule('0 */6 * * *',
--   'DELETE FROM idempotency_keys WHERE expires_at < NOW()');

-- 碰撞错误日志
CREATE TABLE idempotency_errors (
    id              BIGSERIAL PRIMARY KEY,
    idempotency_key VARCHAR(128) NOT NULL,
    error_message   TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## 分布式事务监控看板

### 监控指标体系

```python
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from typing import List, Dict, Optional
from enum import Enum
import statistics


class TxType(Enum):
    SAGA = "saga"
    TCC = "tcc"
    TWO_PC = "2pc"


class TxStatus(Enum):
    RUNNING = "running"
    COMPLETED = "completed"
    COMPENSATING = "compensating"
    COMPENSATED = "compensated"
    FAILED = "failed"
    TIMED_OUT = "timed_out"


@dataclass
class TransactionMetric:
    """单笔事务指标"""
    tx_id: str
    tx_type: TxType
    status: TxStatus
    service_pair: str          # 源服务→目标服务
    started_at: datetime
    completed_at: Optional[datetime]
    steps_total: int
    steps_completed: int
    compensation_count: int = 0


class TransactionMonitor:
    """
    分布式事务监控看板
    核心指标：
    1. 事务量（按类型）：Saga/TCC/2PC 各自的 TPS
    2. 成功率趋势：5 分钟粒度的成功率曲线
    3. 平均耗时（按类型）：各类型事务的平均执行时长
    4. 补偿率：触发补偿操作的比例
    5. 超时率：超时事务占比
    6. 卡住告警：运行超过 5 分钟的事务
    7. 服务对热力图：哪些服务对之间的事务量最大
    8. SLA 合规追踪：99% 事务在 30 秒内完成
    """

    SLA_DURATION_SECONDS = 30
    STUCK_THRESHOLD_SECONDS = 300  # 5 分钟

    def __init__(self, db_pool, redis_client, alert_service):
        self.db = db_pool
        self.redis = redis_client
        self.alert = alert_service

    async def get_volume_by_type(self, start: datetime,
                                  end: datetime) -> Dict[str, int]:
        """事务量统计（按类型）"""
        rows = await self.db.fetch_all(
            """SELECT tx_type, COUNT(*) as cnt
               FROM distributed_transactions
               WHERE started_at BETWEEN %s AND %s
               GROUP BY tx_type""",
            start, end
        )
        return {row["tx_type"]: row["cnt"] for row in rows}

    async def get_success_rate_trend(self, start: datetime,
                                      end: datetime,
                                      window_minutes: int = 5
                                      ) -> List[Dict]:
        """成功率趋势（5 分钟窗口）"""
        rows = await self.db.fetch_all(
            """SELECT
                 time_bucket('%s minutes', started_at) AS bucket,
                 COUNT(*) AS total,
                 COUNT(*) FILTER (WHERE status = 'completed') AS success,
                 ROUND(
                   COUNT(*) FILTER (WHERE status = 'completed')::numeric
                   / NULLIF(COUNT(*), 0) * 100, 2
                 ) AS success_rate_pct
               FROM distributed_transactions
               WHERE started_at BETWEEN %s AND %s
               GROUP BY bucket
               ORDER BY bucket""",
            window_minutes, start, end
        )
        return [dict(row) for row in rows]

    async def get_avg_duration_by_type(self, start: datetime,
                                        end: datetime) -> Dict[str, Dict]:
        """平均耗时（按类型）"""
        rows = await self.db.fetch_all(
            """SELECT tx_type,
                 AVG(EXTRACT(EPOCH FROM (completed_at - started_at))) AS avg_sec,
                 PERCENTILE_CONT(0.5) WITHIN GROUP (
                   ORDER BY EXTRACT(EPOCH FROM (completed_at - started_at))
                 ) AS p50_sec,
                 PERCENTILE_CONT(0.95) WITHIN GROUP (
                   ORDER BY EXTRACT(EPOCH FROM (completed_at - started_at))
                 ) AS p95_sec,
                 PERCENTILE_CONT(0.99) WITHIN GROUP (
                   ORDER BY EXTRACT(EPOCH FROM (completed_at - started_at))
                 ) AS p99_sec
               FROM distributed_transactions
               WHERE started_at BETWEEN %s AND %s
                 AND completed_at IS NOT NULL
               GROUP BY tx_type""",
            start, end
        )
        return {row["tx_type"]: dict(row) for row in rows}

    async def get_compensation_rate(self, start: datetime,
                                     end: datetime) -> Dict:
        """补偿率统计"""
        row = await self.db.fetch_one(
            """SELECT
                 COUNT(*) AS total,
                 COUNT(*) FILTER (
                   WHERE compensation_count > 0
                 ) AS compensated,
                 ROUND(
                   COUNT(*) FILTER (WHERE compensation_count > 0)::numeric
                   / NULLIF(COUNT(*), 0) * 100, 2
                 ) AS compensation_rate_pct,
                 AVG(compensation_count) FILTER (
                   WHERE compensation_count > 0
                 ) AS avg_compensation_steps
               FROM distributed_transactions
               WHERE started_at BETWEEN %s AND %s""",
            start, end
        )
        return dict(row)

    async def get_timeout_rate(self, start: datetime,
                                end: datetime) -> Dict:
        """超时率统计"""
        row = await self.db.fetch_one(
            """SELECT
                 COUNT(*) AS total,
                 COUNT(*) FILTER (WHERE status = 'timed_out') AS timed_out,
                 ROUND(
                   COUNT(*) FILTER (WHERE status = 'timed_out')::numeric
                   / NULLIF(COUNT(*), 0) * 100, 4
                 ) AS timeout_rate_pct
               FROM distributed_transactions
               WHERE started_at BETWEEN %s AND %s""",
            start, end
        )
        return dict(row)

    async def get_stuck_transactions(self) -> List[Dict]:
        """卡住的事务告警（运行超过 5 分钟）"""
        threshold = datetime.utcnow() - timedelta(
            seconds=self.STUCK_THRESHOLD_SECONDS)

        rows = await self.db.fetch_all(
            """SELECT tx_id, tx_type, started_at,
                 EXTRACT(EPOCH FROM (NOW() - started_at)) AS running_sec,
                 steps_total, steps_completed, service_pair
               FROM distributed_transactions
               WHERE status = 'running'
                 AND started_at < %s
               ORDER BY started_at ASC""",
            threshold
        )
        stuck = [dict(row) for row in rows]

        if stuck:
            await self.alert.send_warning(
                title=f"卡住事务告警：{len(stuck)} 笔事务运行超过 5 分钟",
                message=(
                    f"最长运行时间：{max(r['running_sec'] for r in stuck):.0f} 秒\n"
                    f"事务类型分布："
                    f"{dict(statistics.Counter(r['tx_type'] for r in stuck))}\n"
                    f"请检查相关服务是否正常"
                )
            )

        return stuck

    async def get_service_pair_heatmap(self, start: datetime,
                                        end: datetime) -> List[Dict]:
        """服务对热力图：哪些服务对之间的事务量最大"""
        rows = await self.db.fetch_all(
            """SELECT service_pair,
                 COUNT(*) AS tx_count,
                 AVG(EXTRACT(EPOCH FROM (completed_at - started_at))) AS avg_sec,
                 COUNT(*) FILTER (WHERE status != 'completed') AS fail_count,
                 ROUND(
                   COUNT(*) FILTER (WHERE status != 'completed')::numeric
                   / NULLIF(COUNT(*), 0) * 100, 2
                 ) AS fail_rate_pct
               FROM distributed_transactions
               WHERE started_at BETWEEN %s AND %s
               GROUP BY service_pair
               ORDER BY tx_count DESC
               LIMIT 20""",
            start, end
        )
        return [dict(row) for row in rows]

    async def get_sla_compliance(self, start: datetime,
                                  end: datetime) -> Dict:
        """SLA 合规追踪：99% 事务在 30 秒内完成"""
        row = await self.db.fetch_one(
            """SELECT
                 COUNT(*) AS total,
                 COUNT(*) FILTER (
                   WHERE EXTRACT(EPOCH FROM (completed_at - started_at))
                         <= %s
                 ) AS within_sla,
                 ROUND(
                   COUNT(*) FILTER (
                     WHERE EXTRACT(EPOCH FROM (completed_at - started_at))
                           <= %s
                   )::numeric / NULLIF(COUNT(*), 0) * 100, 2
                 ) AS sla_compliance_pct,
                 PERCENTILE_CONT(0.99) WITHIN GROUP (
                   ORDER BY EXTRACT(EPOCH FROM (completed_at - started_at))
                 ) AS p99_sec
               FROM distributed_transactions
               WHERE started_at BETWEEN %s AND %s
                 AND completed_at IS NOT NULL""",
            self.SLA_DURATION_SECONDS,
            self.SLA_DURATION_SECONDS,
            start, end
        )
        result = dict(row)
        result["sla_target_pct"] = 99.0
        result["sla_met"] = result.get("sla_compliance_pct", 0) >= 99.0
        return result

    async def get_dashboard_summary(self) -> Dict:
        """看板汇总（近 1 小时）"""
        now = datetime.utcnow()
        one_hour_ago = now - timedelta(hours=1)

        volume = await self.get_volume_by_type(one_hour_ago, now)
        success_trend = await self.get_success_rate_trend(one_hour_ago, now)
        duration = await self.get_avg_duration_by_type(one_hour_ago, now)
        compensation = await self.get_compensation_rate(one_hour_ago, now)
        timeout = await self.get_timeout_rate(one_hour_ago, now)
        stuck = await self.get_stuck_transactions()
        heatmap = await self.get_service_pair_heatmap(one_hour_ago, now)
        sla = await self.get_sla_compliance(one_hour_ago, now)

        return {
            "period": {"start": one_hour_ago.isoformat(),
                       "end": now.isoformat()},
            "volume_by_type": volume,
            "success_rate_trend": success_trend,
            "avg_duration_by_type": duration,
            "compensation_rate": compensation,
            "timeout_rate": timeout,
            "stuck_transactions": stuck,
            "service_pair_heatmap": heatmap,
            "sla_compliance": sla
        }
```

### 看板 SQL 视图

```sql
-- 事务实时统计视图
CREATE VIEW v_tx_realtime_stats AS
SELECT
  tx_type,
  COUNT(*) FILTER (WHERE created_at > NOW() - INTERVAL '5 minutes') AS last_5m_count,
  COUNT(*) FILTER (
    WHERE created_at > NOW() - INTERVAL '5 minutes' AND status = 'completed'
  ) AS last_5m_success,
  ROUND(
    COUNT(*) FILTER (
      WHERE created_at > NOW() - INTERVAL '5 minutes' AND status = 'completed'
    )::numeric / NULLIF(
      COUNT(*) FILTER (WHERE created_at > NOW() - INTERVAL '5 minutes'), 0
    ) * 100, 2
  ) AS last_5m_success_rate,
  AVG(EXTRACT(EPOCH FROM (completed_at - started_at))) FILTER (
    WHERE completed_at > NOW() - INTERVAL '5 minutes'
  ) AS last_5m_avg_duration
FROM distributed_transactions
GROUP BY tx_type;

-- SLA 违规明细
CREATE VIEW v_sla_violations AS
SELECT tx_id, tx_type, started_at, completed_at,
       EXTRACT(EPOCH FROM (completed_at - started_at)) AS duration_sec,
       service_pair
FROM distributed_transactions
WHERE completed_at IS NOT NULL
  AND EXTRACT(EPOCH FROM (completed_at - started_at)) > 30
ORDER BY started_at DESC;

-- 服务对事务热力图
CREATE MATERIALIZED VIEW mv_service_pair_heatmap AS
SELECT service_pair,
  COUNT(*) AS tx_count,
  AVG(EXTRACT(EPOCH FROM (completed_at - started_at))) AS avg_duration,
  COUNT(*) FILTER (WHERE status != 'completed') AS fail_count
FROM distributed_transactions
WHERE started_at > NOW() - INTERVAL '24 hours'
GROUP BY service_pair
ORDER BY tx_count DESC;

REFRESH MATERIALIZED VIEW CONCURRENTLY mv_service_pair_heatmap;
```

## 异常场景补充（续）

### 场景：幂等键碰撞——不同操作产生相同键

```
触发：两个不同的操作（如 A 用户下单 + B 用户退款）
      碰巧生成了相同的幂等键。
      原因：客户端 UUID 重复（UUID v4 碰撞概率 ~10^-18，
      但客户端库 bug 导致 UUID 生成器种子相同）+
      操作哈希碰巧相同（参数排序后哈希前 16 位一致）。
检测：
  1. 碰撞检测器发现相同键对应不同操作参数 → 立即告警
  2. 第二个请求执行时返回缓存中第一个请求的结果 → 业务异常
  3. 用户 A 收到用户 B 的退款结果 → 客户投诉
处理：
  1. 立即暂停受影响操作类型的幂等检查（降级为仅 DB 唯一约束）
  2. 排查 UUID 生成器 bug，修复后重新部署
  3. 在幂等键中加入更多熵源：机器 ID + 时间戳纳秒 + 随机数
  4. 加长操作哈希到 32 位（SHA256 完整输出）
  5. 增加键碰撞审计日志，定期检查碰撞频率
  6. 已发生的错误结果 → 人工核对并修正
预防：键生成加入多熵源 + 碰撞检测告警 + 定期审计
```

### 场景：监控看板误报高失败率——采样偏差

```
触发：监控看板显示 TCC 事务成功率从 99.5% 骤降到 85%，
      触发 P0 告警。但实际业务未受影响。
      原因：凌晨 3 点低流量时段，只有 20 笔事务，
      其中 3 笔是已知的测试环境回滚（占 15%）。
      采样量太小导致失败率被放大。
检测：
  1. P0 告警：TCC 成功率 < 90%
  2. 值班工程师发现告警时段事务量极低（20 笔 vs 日均 100 万笔）
  3. 失败事务均来自测试环境 → 非生产问题
处理：
  1. 修改告警规则：增加最小样本量门槛
     - 5 分钟窗口内事务量 < 100 → 不触发成功率告警
     - 低流量时段使用 30 分钟窗口替代 5 分钟窗口
  2. 告警规则增加环境过滤：测试环境事务不纳入生产告警
  3. 增加"告警置信度"指标：样本量越大置信度越高
  4. 低置信度告警降级为 P2（仅通知，不拉群）
  5. 事后复盘：梳理所有依赖采样率的告警，统一增加样本量门槛
预防：告警增加最小样本量 + 环境隔离 + 置信度评估 + 分时段阈值

## 幂等性保证框架完整实现

```python
class IdempotencyFramework:
    """幂等性保证框架：去重 + 结果缓存 + 冲突处理"""

    def execute_idempotently(self, idempotency_key, operation, ttl_seconds=86400):
        """幂等执行操作"""
        # 1. 检查去重表
        cached = self.redis.get(f"idempotent:{idempotency_key}")
        if cached:
            cached_result = json.loads(cached)
            if cached_result["status"] == "completed":
                return {"result": cached_result["data"], "cached": True}
            elif cached_result["status"] == "processing":
                # 正在处理中 → 等待或返回
                return {"result": None, "status": "processing"}

        # 2. 数据库去重检查（双重保障）
        existing = self.db.query_one(
            "SELECT * FROM idempotency_log WHERE idempotency_key = %s",
            idempotency_key)
        if existing and existing["status"] == "completed":
            return {"result": json.loads(existing["result"]), "cached": True}

        # 3. 标记为处理中
        self.redis.setex(f"idempotent:{idempotency_key}", ttl_seconds,
            json.dumps({"status": "processing", "started_at": now().isoformat()}))

        # 4. 执行操作
        try:
            result = operation()

            # 5. 缓存结果
            result_data = {"status": "completed", "data": result,
                          "completed_at": now().isoformat()}
            self.redis.setex(f"idempotent:{idempotency_key}", ttl_seconds,
                json.dumps(result_data))

            # 6. 持久化到数据库
            self.db.insert("idempotency_log", {
                "idempotency_key": idempotency_key,
                "status": "completed",
                "result": json.dumps(result),
                "created_at": now()
            })

            return {"result": result, "cached": False}

        except Exception as e:
            # 执行失败 → 清除处理中标记
            self.redis.delete(f"idempotent:{idempotency_key}")
            raise

    def generate_key(self, operation_type, params):
        """生成幂等键（操作类型 + 参数哈希）"""
        param_hash = hashlib.sha256(
            json.dumps(params, sort_keys=True).encode()).hexdigest()[:16]
        return f"{operation_type}:{param_hash}"
```

## 分布式事务监控看板

```python
class TransactionMonitoringDashboard:
    """分布式事务监控看板"""

    def get_overview(self):
        """事务监控总览"""
        return {
            "volume_by_type": self._volume_by_type(),
            "success_rate_trend": self._success_rate_trend(),
            "avg_duration_by_type": self._avg_duration(),
            "compensation_rate": self._compensation_rate(),
            "stuck_transactions": self._stuck_transactions(),
            "sla_compliance": self._sla_compliance(),
        }

    def _volume_by_type(self):
        """按类型统计事务量"""
        return self.db.query(
            "SELECT type, COUNT(*) as count, "
            "SUM(CASE WHEN status = 'completed' THEN 1 ELSE 0 END) as success, "
            "SUM(CASE WHEN status = 'failed' THEN 1 ELSE 0 END) as failed "
            "FROM distributed_transactions "
            "WHERE created_at > NOW() - INTERVAL 1 HOUR "
            "GROUP BY type")

    def _success_rate_trend(self):
        """成功率趋势（每小时）"""
        return self.db.query(
            "SELECT DATE_FORMAT(created_at, '%%Y-%%m-%%d %%H:00') as hour, "
            "COUNT(*) as total, "
            "SUM(CASE WHEN status = 'completed' THEN 1 ELSE 0 END) / COUNT(*) as rate "
            "FROM distributed_transactions "
            "WHERE created_at > NOW() - INTERVAL 24 HOUR "
            "GROUP BY hour ORDER BY hour")

    def _avg_duration(self):
        """平均耗时"""
        return self.db.query(
            "SELECT type, "
            "AVG(TIMESTAMPDIFF(MILLISECOND, created_at, completed_at)) as avg_ms, "
            "PERCENTILE_CONT(0.99) WITHIN GROUP "
            "(ORDER BY TIMESTAMPDIFF(MILLISECOND, created_at, completed_at)) as p99_ms "
            "FROM distributed_transactions "
            "WHERE status = 'completed' AND created_at > NOW() - INTERVAL 1 HOUR "
            "GROUP BY type")

    def _stuck_transactions(self):
        """卡住的事务（运行超过 5 分钟）"""
        return self.db.query(
            "SELECT * FROM distributed_transactions "
            "WHERE status = 'running' AND created_at < NOW() - INTERVAL 5 MINUTE "
            "ORDER BY created_at ASC LIMIT 20")

    def _compensation_rate(self):
        """补偿率"""
        total = self.db.count("distributed_transactions",
            created_at__gte=now()-timedelta(hours=1))
        compensated = self.db.count("distributed_transactions",
            created_at__gte=now()-timedelta(hours=1), status="compensated")
        return round(compensated / max(total, 1), 4)

    def _sla_compliance(self):
        """SLA 合规（99% 事务在 3 秒内完成）"""
        return self.db.query_one(
            "SELECT "
            "SUM(CASE WHEN TIMESTAMPDIFF(SECOND, created_at, completed_at) <= 3 "
            "THEN 1 ELSE 0 END) / COUNT(*) as compliance_rate "
            "FROM distributed_transactions "
            "WHERE status = 'completed' AND created_at > NOW() - INTERVAL 1 HOUR")
```

## 异常场景补充

### 场景：幂等键碰撞

```
触发：两个无关操作生成了相同的幂等键 → 后一个操作被错误去重
概率：SHA256 前 16 位碰撞概率极低（2^-64）
检测：
  1. 同一幂等键的参数不一致 → 碰撞
  2. 操作结果与预期不符 → 碰撞
处理：
  1. 增加幂等键长度（16 位 → 24 位）
  2. 在去重表中存储原始参数 → 比对确认
  3. 碰撞时生成新键重试
预防：幂等键包含更多参数信息 + 参数比对验证
```

### 场景：监控看板误报高失败率

```
触发：采样偏差导致看板显示失败率异常 → 实际正常
检测：
  1. 看板失败率 > 5% → 告警
  2. 实际业务无异常 → 误报
处理：
  1. 检查采样窗口（是否包含低流量时段）
  2. 增大样本量（1 小时 → 6 小时滑动窗口）
  3. 设置最低样本量阈值（< 100 不计算比率）
预防：最低样本量 + 滑动窗口 + 异常值过滤
```

## 分布式事务补偿机制完整实现

```python
class DistributedTransactionCompensator:
    """分布式事务补偿：Saga 补偿 + 重试 + 超时处理"""

    SAGA_STEPS = {
        "create_order": {"compensate": "cancel_order", "timeout_seconds": 10, "max_retries": 3},
        "lock_inventory": {"compensate": "unlock_inventory", "timeout_seconds": 5, "max_retries": 2},
        "process_payment": {"compensate": "refund_payment", "timeout_seconds": 30, "max_retries": 3},
        "create_shipping": {"compensate": "cancel_shipping", "timeout_seconds": 15, "max_retries": 2},
        "send_notification": {"compensate": None, "timeout_seconds": 5, "max_retries": 1},
    }

    def execute_saga(self, saga_id, initial_payload):
        """执行 Saga 事务"""
        self.db.insert("saga_transactions", {
            "saga_id": saga_id,
            "payload": json.dumps(initial_payload),
            "status": "running",
            "started_at": now()
        })

        completed_steps = []
        payload = initial_payload

        for step_name, step_config in self.SAGA_STEPS.items():
            # 执行步骤（带超时和重试）
            result = self._execute_step_with_retry(
                saga_id, step_name, payload, step_config)

            if result["success"]:
                completed_steps.append({
                    "step": step_name,
                    "result": result["data"],
                    "compensate": step_config["compensate"]
                })
                # 传递结果到下一步
                payload = {**payload, **result["data"]}
                self.db.insert("saga_step_log", {
                    "saga_id": saga_id, "step": step_name,
                    "status": "completed", "result": json.dumps(result["data"]),
                    "completed_at": now()
                })
            else:
                # 步骤失败 → 执行补偿
                self._execute_compensation(saga_id, completed_steps, step_name, result["error"])
                return {"saga_id": saga_id, "status": "failed",
                        "failed_step": step_name, "error": result["error"],
                        "compensated_steps": len(completed_steps)}

        # 所有步骤成功
        self.db.update("saga_transactions",
            {"status": "completed", "completed_at": now()},
            {"saga_id": saga_id})

        return {"saga_id": saga_id, "status": "completed",
                "steps_completed": len(completed_steps)}

    def _execute_step_with_retry(self, saga_id, step_name, payload, config):
        """带超时和重试的步骤执行"""
        max_retries = config["max_retries"]
        timeout = config["timeout_seconds"]

        for attempt in range(max_retries + 1):
            try:
                # 调用对应的服务
                handler = self._get_step_handler(step_name)
                result = handler(payload)

                self.db.insert("saga_step_attempts", {
                    "saga_id": saga_id, "step": step_name,
                    "attempt": attempt + 1, "status": "success",
                    "attempted_at": now()
                })

                return {"success": True, "data": result}

            except TimeoutError:
                self.db.insert("saga_step_attempts", {
                    "saga_id": saga_id, "step": step_name,
                    "attempt": attempt + 1, "status": "timeout",
                    "attempted_at": now()
                })
                if attempt < max_retries:
                    continue

            except Exception as e:
                self.db.insert("saga_step_attempts", {
                    "saga_id": saga_id, "step": step_name,
                    "attempt": attempt + 1, "status": "error",
                    "error": str(e), "attempted_at": now()
                })
                # 可重试错误 vs 不可重试错误
                if self._is_retryable(e) and attempt < max_retries:
                    continue

                return {"success": False, "error": str(e)}

        return {"success": False, "error": f"max retries ({max_retries}) exceeded"}

    def _execute_compensation(self, saga_id, completed_steps, failed_step, error):
        """执行补偿（反向执行已完成的步骤）"""
        self.db.update("saga_transactions",
            {"status": "compensating"}, {"saga_id": saga_id})

        # 反向补偿
        for step_info in reversed(completed_steps):
            compensate_action = step_info["compensate"]
            if compensate_action is None:
                continue  # 无需补偿（如通知）

            try:
                handler = self._get_step_handler(compensate_action)
                handler(step_info["result"])

                self.db.insert("saga_compensation_log", {
                    "saga_id": saga_id,
                    "original_step": step_info["step"],
                    "compensate_action": compensate_action,
                    "status": "completed",
                    "compensated_at": now()
                })

            except Exception as e:
                # 补偿失败 → 人工介入
                self.db.insert("saga_compensation_log", {
                    "saga_id": saga_id,
                    "original_step": step_info["step"],
                    "compensate_action": compensate_action,
                    "status": "failed",
                    "error": str(e),
                    "compensated_at": now()
                })
                self.alert(f"Saga {saga_id} 补偿失败: {compensate_action}, 错误: {e}")

        self.db.update("saga_transactions",
            {"status": "compensated", "failed_step": failed_step,
             "error": error, "completed_at": now()},
            {"saga_id": saga_id})

    def _is_retryable(self, error):
        """判断是否可重试"""
        retryable_types = ["TimeoutError", "ConnectionError", "ServiceUnavailableError"]
        return any(t in str(type(error)) for t in retryable_types)
```

## 异常场景补充

### 场景：Saga 补偿顺序错误

```
触发：补偿执行顺序不按反向 → 先退款再解锁库存 → 但库存解锁后又被其他订单锁定 → 退款后库存不足
检测：
  1. 补偿后数据状态不一致 → 补偿顺序问题
  2. 补偿日志顺序不是 reversed → 逻辑错误
处理：
  1. 严格按反向顺序补偿
  2. 补偿时加锁防止并发
  3. 补偿后验证数据一致性
预防：反向补偿 + 补偿锁 + 一致性验证
```

### 场景：Saga 步骤超时但实际已执行

```
触发：支付步骤超时 → 触发补偿 → 但支付实际已成功 → 退款后用户已付款 → 资金错误
检测：
  1. 补偿退款后发现支付记录 → 超时但实际成功
  2. 补偿后用户投诉已付款被退 → 资金问题
处理：
  1. 补偿前查询目标服务确认实际状态
  2. 支付已成功 → 不退款，继续后续步骤
  3. 确认后再决定补偿还是继续
预防：补偿前确认实际状态 + 幂等支付接口 + 状态查询
```

## TCC 分布式事务完整实现

```python
class TCCTransactionManager:
    """TCC 事务：Try-Confirm-Cancel + 超时 + 幂等"""

    def execute_tcc(self, transaction_id, participants, payload):
        """执行 TCC 事务"""
        self.db.insert("tcc_transactions", {
            "transaction_id": transaction_id,
            "participant_count": len(participants),
            "status": "trying",
            "payload": json.dumps(payload),
            "started_at": now()
        })

        # Phase 1: Try
        try_results = []
        all_try_success = True

        for i, participant in enumerate(participants):
            try:
                result = self._call_try(participant, transaction_id, payload)
                try_results.append({"participant": participant["name"],
                    "status": "success", "result": result})
                self.db.insert("tcc_participant_log", {
                    "transaction_id": transaction_id,
                    "participant": participant["name"],
                    "phase": "try", "status": "success",
                    "executed_at": now()
                })
            except Exception as e:
                try_results.append({"participant": participant["name"],
                    "status": "failed", "error": str(e)})
                all_try_success = False
                self.db.insert("tcc_participant_log", {
                    "transaction_id": transaction_id,
                    "participant": participant["name"],
                    "phase": "try", "status": "failed",
                    "error": str(e), "executed_at": now()
                })
                break  # Try 失败 → 立即进入 Cancel

        if all_try_success:
            # Phase 2: Confirm
            self.db.update("tcc_transactions",
                {"status": "confirming"}, {"transaction_id": transaction_id})

            for i, participant in enumerate(participants):
                try:
                    self._call_confirm(participant, transaction_id)
                    self.db.insert("tcc_participant_log", {
                        "transaction_id": transaction_id,
                        "participant": participant["name"],
                        "phase": "confirm", "status": "success",
                        "executed_at": now()
                    })
                except Exception as e:
                    # Confirm 失败 → 重试（Confirm 必须最终成功）
                    self.db.insert("tcc_participant_log", {
                        "transaction_id": transaction_id,
                        "participant": participant["name"],
                        "phase": "confirm", "status": "failed",
                        "error": str(e), "executed_at": now()
                    })
                    self._schedule_confirm_retry(transaction_id, participant)
                    # 不中断，继续 confirm 其他参与者

            self.db.update("tcc_transactions",
                {"status": "confirmed", "completed_at": now()},
                {"transaction_id": transaction_id})
            return {"status": "confirmed", "participants": len(participants)}

        else:
            # Phase 2: Cancel
            self.db.update("tcc_transactions",
                {"status": "cancelling"}, {"transaction_id": transaction_id})

            for participant in participants[:len(try_results)]:
                try:
                    self._call_cancel(participant, transaction_id)
                    self.db.insert("tcc_participant_log", {
                        "transaction_id": transaction_id,
                        "participant": participant["name"],
                        "phase": "cancel", "status": "success",
                        "executed_at": now()
                    })
                except Exception as e:
                    self.db.insert("tcc_participant_log", {
                        "transaction_id": transaction_id,
                        "participant": participant["name"],
                        "phase": "cancel", "status": "failed",
                        "error": str(e), "executed_at": now()
                    })
                    self._schedule_cancel_retry(transaction_id, participant)

            self.db.update("tcc_transactions",
                {"status": "cancelled", "completed_at": now()},
                {"transaction_id": transaction_id})
            return {"status": "cancelled", "failed_at": try_results[-1]["participant"]}

    def _call_try(self, participant, tx_id, payload):
        """调用 Try 接口"""
        url = f"{participant['endpoint']}/try"
        response = requests.post(url, json={
            "transaction_id": tx_id,
            "payload": payload
        }, timeout=participant.get("timeout_seconds", 10))
        response.raise_for_status()
        return response.json()

    def _call_confirm(self, participant, tx_id):
        """调用 Confirm 接口（必须幂等）"""
        url = f"{participant['endpoint']}/confirm"
        response = requests.post(url, json={
            "transaction_id": tx_id
        }, timeout=participant.get("timeout_seconds", 10))
        response.raise_for_status()

    def _call_cancel(self, participant, tx_id):
        """调用 Cancel 接口（必须幂等）"""
        url = f"{participant['endpoint']}/cancel"
        response = requests.post(url, json={
            "transaction_id": tx_id
        }, timeout=participant.get("timeout_seconds", 10))
        response.raise_for_status()

    def _schedule_confirm_retry(self, tx_id, participant):
        """调度 Confirm 重试"""
        self.db.insert("tcc_retry_tasks", {
            "task_id": str(uuid4()),
            "transaction_id": tx_id,
            "participant": participant["name"],
            "phase": "confirm",
            "max_retries": 10,
            "retry_count": 0,
            "next_retry_at": now() + timedelta(seconds=30),
            "created_at": now()
        })

    def _schedule_cancel_retry(self, tx_id, participant):
        """调度 Cancel 重试"""
        self.db.insert("tcc_retry_tasks", {
            "task_id": str(uuid4()),
            "transaction_id": tx_id,
            "participant": participant["name"],
            "phase": "cancel",
            "max_retries": 10,
            "retry_count": 0,
            "next_retry_at": now() + timedelta(seconds=30),
            "created_at": now()
        })
```

## 异常场景补充

### 场景：Try 部分成功后 Cancel 失败

```
触发：3 个参与者 Try 成功 2 个 → 第 3 个 Try 失败 → Cancel 前 2 个 → Cancel 也失败 → 资源冻结
检测：
  1. TCC 事务状态停留在 "cancelling" → Cancel 未完成
  2. 参与者资源被冻结但未释放 → 泄漏
处理：
  1. 定时重试 Cancel（指数退避）
  2. 重试超过上限 → 人工介入
  3. 参与者提供超时自动释放机制
预防：Cancel 重试 + 超时自动释放 + 人工兜底
```

### 场景：Confirm 重试导致重复执行

```
触发：Confirm 请求成功但响应超时 → 重试 Confirm → 参与者执行两次 → 资源重复扣减
检测：
  1. 参与者确认了两次 → 重复执行
  2. 余额/库存异常 → 重复扣减
处理：
  1. Confirm 接口必须幂等（基于 transaction_id 去重）
  2. 已确认的事务直接返回成功
  3. 定期对账发现差异
预防：Confirm 幂等 + 去重 + 对账
```

## 分布式事务本地消息表完整实现

```python
class LocalMessageTableService:
    """本地消息表：写业务 + 写消息 → 定时扫描发送 → 幂等消费"""

    def execute_with_message(self, business_fn, message, db_session):
        """在同一事务中执行业务操作并写入消息表"""
        # 1. 执行业务操作
        business_result = business_fn(db_session)

        # 2. 写入本地消息表（同一事务）
        db_session.insert("local_message_table", {
            "message_id": str(uuid4()),
            "topic": message["topic"],
            "key": message.get("key"),
            "body": json.dumps(message["body"]),
            "status": "pending",
            "retry_count": 0,
            "max_retries": message.get("max_retries", 5),
            "next_retry_at": now(),
            "created_at": now()
        })

        # 事务提交后，消息表中有待发送的消息
        return business_result

    def scan_and_send_messages(self, batch_size=100):
        """定时扫描消息表并发送"""
        # 1. 查询待发送消息
        messages = self.db.query(
            "SELECT * FROM local_message_table "
            "WHERE status = 'pending' AND next_retry_at <= NOW() "
            "ORDER BY created_at ASC LIMIT %s", batch_size)

        if not messages:
            return {"sent": 0, "failed": 0}

        sent = 0
        failed = 0

        for msg in messages:
            try:
                # 2. 发送消息到 MQ
                self.mq_producer.send(
                    topic=msg["topic"],
                    key=msg["key"],
                    body=msg["body"])

                # 3. 标记为已发送
                self.db.update("local_message_table",
                    {"status": "sent", "sent_at": now()},
                    {"message_id": msg["message_id"]})

                sent += 1

            except Exception as e:
                # 4. 发送失败 → 增加重试次数
                retry_count = msg["retry_count"] + 1

                if retry_count >= msg["max_retries"]:
                    # 超过最大重试 → 标记为失败
                    self.db.update("local_message_table",
                        {"status": "failed", "retry_count": retry_count,
                         "error": str(e), "failed_at": now()},
                        {"message_id": msg["message_id"]})
                    self.alert(f"消息发送失败: {msg['message_id']}, topic: {msg['topic']}")
                else:
                    # 指数退避重试
                    delay_seconds = min(300, 2 ** retry_count)
                    next_retry = now() + timedelta(seconds=delay_seconds)

                    self.db.update("local_message_table",
                        {"retry_count": retry_count,
                         "next_retry_at": next_retry,
                         "last_error": str(e)},
                        {"message_id": msg["message_id"]})

                failed += 1

        return {"sent": sent, "failed": failed, "total": len(messages)}

    def consume_message(self, topic, message_body, consumer_fn):
        """消费消息（幂等）"""
        message_id = message_body.get("message_id")
        if not message_id:
            message_id = str(uuid4())

        # 1. 幂等检查（已消费 → 直接返回成功）
        consumed = self.redis.setnx(f"msg_consumed:{message_id}", "1")
        if not consumed:
            return {"status": "already_consumed"}

        self.redis.expire(f"msg_consumed:{message_id}", 86400 * 7)  # 7 天过期

        # 2. 执行消费逻辑
        try:
            result = consumer_fn(message_body)
            return {"status": "consumed", "result": result}
        except Exception as e:
            # 消费失败 → 清除幂等标记（允许重试）
            self.redis.delete(f"msg_consumed:{message_id}")
            raise

    def clean_sent_messages(self, retention_days=7):
        """清理已发送的旧消息"""
        cutoff = now() - timedelta(days=retention_days)

        deleted = self.db.execute(
            "DELETE FROM local_message_table "
            "WHERE status = 'sent' AND sent_at < %s", cutoff)

        return {"deleted": deleted}
```

## 异常场景补充

### 场景：消息表数据膨胀

```
触发：每秒 1000 笔业务 → 消息表日增 8640 万行 → 表膨胀 → 查询变慢
检测：
  1. 消息表行数 > 1 亿 → 膨胀
  2. 扫描耗时 > 5 秒 → 性能问题
处理：
  1. 定期清理已发送消息（保留 7 天）
  2. 按时间分区（每天一个分区）
  3. 归档到冷存储
预防：定期清理 + 分区表 + 归档
```

### 场景：消息重复消费

```
触发：消费成功但确认丢失 → MQ 重发 → 消费两次 → 业务重复执行
检测：
  1. 同一 message_id 被消费两次 → 重复
  2. 业务数据出现重复记录 → 重复消费
处理：
  1. 消费端幂等（基于 message_id 去重）
  2. 业务操作支持幂等
  3. 消费确认后才删除幂等标记
预防：幂等消费 + 业务幂等 + 确认后删除
```

## 分布式事务补偿模式完整实现

```python
class CompensationPatternService:
    """补偿模式：正向操作 + 补偿操作 + 编排执行 + 自动重试"""

    def execute_saga(self, saga_definition, payload):
        """执行 Saga 事务"""
        saga_id = str(uuid4())
        steps = saga_definition["steps"]

        # 记录 Saga 开始
        self.db.insert("saga_instances", {
            "saga_id": saga_id,
            "definition_id": saga_definition["id"],
            "payload": json.dumps(payload),
            "status": "running",
            "current_step": 0,
            "started_at": now()
        })

        completed_steps = []

        try:
            # 正向执行每一步
            for i, step in enumerate(steps):
                self.db.update("saga_instances",
                    {"current_step": i},
                    {"saga_id": saga_id})

                result = self._execute_step(saga_id, step, payload, i)
                completed_steps.append({
                    "step_index": i,
                    "step_name": step["name"],
                    "result": result
                })

                # 记录步骤完成
                self.db.insert("saga_step_log", {
                    "log_id": str(uuid4()),
                    "saga_id": saga_id,
                    "step_index": i,
                    "step_name": step["name"],
                    "action": "execute",
                    "status": "success",
                    "result": json.dumps(result, default=str),
                    "executed_at": now()
                })

            # 全部成功
            self.db.update("saga_instances",
                {"status": "completed", "completed_at": now()},
                {"saga_id": saga_id})

            return {"saga_id": saga_id, "status": "completed",
                    "steps_completed": len(completed_steps)}

        except Exception as e:
            # 某步失败 → 逆向补偿已完成的步骤
            self.db.update("saga_instances",
                {"status": "compensating", "failed_step": i,
                 "error": str(e)},
                {"saga_id": saga_id})

            compensation_errors = []

            for completed in reversed(completed_steps):
                step_index = completed["step_index"]
                step = steps[step_index]

                try:
                    # 执行补偿操作
                    self._execute_compensation(saga_id, step, payload, step_index)

                    self.db.insert("saga_step_log", {
                        "log_id": str(uuid4()),
                        "saga_id": saga_id,
                        "step_index": step_index,
                        "step_name": step["name"],
                        "action": "compensate",
                        "status": "success",
                        "executed_at": now()
                    })

                except Exception as ce:
                    compensation_errors.append({
                        "step": step["name"],
                        "error": str(ce)
                    })

                    self.db.insert("saga_step_log", {
                        "log_id": str(uuid4()),
                        "saga_id": saga_id,
                        "step_index": step_index,
                        "step_name": step["name"],
                        "action": "compensate",
                        "status": "failed",
                        "error": str(ce),
                        "executed_at": now()
                    })

            final_status = "compensated" if not compensation_errors else "compensation_failed"

            self.db.update("saga_instances",
                {"status": final_status,
                 "compensation_errors": json.dumps(compensation_errors),
                 "completed_at": now()},
                {"saga_id": saga_id})

            if compensation_errors:
                self.alert(f"Saga {saga_id} 补偿失败: {compensation_errors}")

            return {"saga_id": saga_id, "status": final_status,
                    "compensation_errors": compensation_errors}

    def _execute_step(self, saga_id, step, payload, step_index):
        """执行正向步骤"""
        service = step["service"]
        action = step["action"]

        # 调用远程服务
        response = self.service_client.call(
            service=service,
            action=action,
            payload={
                "saga_id": saga_id,
                "step_index": step_index,
                **payload
            },
            timeout=step.get("timeout_seconds", 30))

        if not response.get("success"):
            raise StepExecutionError(
                f"步骤 {step['name']} 执行失败: {response.get('error')}")

        return response.get("data", {})

    def _execute_compensation(self, saga_id, step, payload, step_index):
        """执行补偿操作"""
        service = step["service"]
        compensate_action = step["compensate_action"]

        response = self.service_client.call(
            service=service,
            action=compensate_action,
            payload={
                "saga_id": saga_id,
                "step_index": step_index,
                **payload
            },
            timeout=step.get("timeout_seconds", 30))

        if not response.get("success"):
            raise CompensationError(
                f"步骤 {step['name']} 补偿失败: {response.get('error')}")

    def retry_failed_saga(self, saga_id):
        """重试失败的 Saga"""
        saga = self.db.get_saga(saga_id)

        if saga["status"] not in ["compensation_failed"]:
            return {"status": "cannot_retry", "current": saga["status"]}

        # 1. 找出补偿失败的步骤
        failed_compensations = self.db.query(
            "SELECT * FROM saga_step_log "
            "WHERE saga_id = %s AND action = 'compensate' AND status = 'failed' "
            "ORDER BY step_index DESC",
            saga_id)

        # 2. 重试补偿
        for log in failed_compensations:
            # 获取 Saga 定义
            definition = self.db.get_saga_definition(saga["definition_id"])
            step = definition["steps"][log["step_index"]]

            try:
                payload = json.loads(saga["payload"])
                self._execute_compensation(saga_id, step, payload, log["step_index"])

                self.db.update("saga_step_log",
                    {"status": "success"},
                    {"log_id": log["log_id"]})

            except Exception as e:
                return {"status": "retry_failed", "step": step["name"], "error": str(e)}

        self.db.update("saga_instances",
            {"status": "compensated", "completed_at": now()},
            {"saga_id": saga_id})

        return {"status": "compensated", "saga_id": saga_id}
```

## 异常场景补充

### 场景：Saga 补偿顺序错误

```
触发：步骤 A（创建订单）→ 步骤 B（扣库存）→ 步骤 C（支付）失败 → 先补偿 A 而非 B → 库存未恢复
检测：
  1. 补偿后库存/余额不一致 → 补偿顺序错误
  2. 补偿日志显示非逆序执行 → 逻辑错误
处理：
  1. 强制逆序补偿（reversed(completed_steps)）
  2. 补偿前校验步骤依赖关系
  3. 补偿后一致性验证
预防：强制逆序 + 依赖校验 + 一致性验证
```

### 场景：Saga 长时间运行导致资源锁定

```
触发：Saga 执行 10 分钟 → 中间步骤锁定资源 → 其他事务等待 → 系统卡顿
检测：
  1. Saga 执行时间 > 5 分钟 → 长事务
  2. 锁等待超时 → 资源争用
处理：
  1. Saga 步骤拆细（减少单步执行时间）
  2. 每步执行后释放资源
  3. 超时自动触发补偿
预防：步骤拆细 + 资源释放 + 超时补偿
```

## 分布式事务性能优化与监控完整实现

```python
import math
import time
from datetime import datetime, timedelta
from collections import defaultdict, deque
from typing import List, Dict, Tuple, Optional, Set

class TransactionPerformanceService:
    """分布式事务性能优化与监控服务，提供指标记录、慢事务检测、路径优化和仪表盘生成"""

    SLIDING_WINDOW_SECONDS = 60
    PERCENTILE_BUFFER_SIZE = 1000
    LATENCY_SPIKE_MULTIPLIER = 2.0
    DEGRADATION_SLOPE_THRESHOLD = 0.1  # 10% per hour
    SLOW_TRANSACTION_LOOKBACK_HOURS = 1
    PARALLEL_GROUP_MIN_SAVINGS_MS = 50

    def __init__(self):
        self.step_metrics: Dict[str, List[Dict]] = defaultdict(list)
        self.transaction_steps: Dict[str, List[Dict]] = defaultdict(list)
        self.transaction_definitions: Dict[str, Dict] = {}
        self.sliding_window: deque = deque()
        self.percentile_buffer: Dict[str, deque] = defaultdict(deque)
        self.baseline_p99: Dict[str, float] = {}
        self.service_tps: Dict[str, List[Tuple[float, int]]] = defaultdict(list)
        self.latency_history: Dict[str, List[Tuple[float, float]]] = defaultdict(list)
        self.failure_counts: Dict[str, int] = defaultdict(int)
        self.total_counts: Dict[str, int] = defaultdict(int)
        self.sla_thresholds: Dict[str, float] = {}
        self.resource_metrics: Dict[str, List[Tuple[float, Dict]]] = defaultdict(list)

    def record_transaction_metrics(self, transaction_id: str, step_name: str,
                                    duration_ms: float, success: bool) -> Dict:
        """记录事务步骤指标，计算滑动窗口TPS，跟踪延迟百分位，检测延迟尖峰"""
        timestamp = time.time()
        metric = {
            "transaction_id": transaction_id,
            "step_name": step_name,
            "duration_ms": duration_ms,
            "success": success,
            "timestamp": timestamp
        }
        self.step_metrics[step_name].append(metric)
        self.transaction_steps[transaction_id].append(metric)
        self.sliding_window.append({
            "timestamp": timestamp,
            "transaction_id": transaction_id,
            "step_name": step_name
        })
        while (self.sliding_window and
               timestamp - self.sliding_window[0]["timestamp"] > self.SLIDING_WINDOW_SECONDS):
            self.sliding_window.popleft()
        current_tps = len(self.sliding_window) / self.SLIDING_WINDOW_SECONDS
        buf = self.percentile_buffer[step_name]
        buf.append(duration_ms)
        while len(buf) > self.PERCENTILE_BUFFER_SIZE:
            buf.popleft()
        sorted_durations = sorted(buf)
        p50 = self._percentile_from_sorted(sorted_durations, 0.50)
        p90 = self._percentile_from_sorted(sorted_durations, 0.90)
        p99 = self._percentile_from_sorted(sorted_durations, 0.99)
        latency_spike_detected = False
        if step_name in self.baseline_p99:
            if p99 > self.baseline_p99[step_name] * self.LATENCY_SPIKE_MULTIPLIER:
                latency_spike_detected = True
        else:
            if len(buf) >= 50:
                self.baseline_p99[step_name] = p99
        if success:
            self.total_counts[step_name] += 1
        else:
            self.failure_counts[step_name] += 1
            self.total_counts[step_name] += 1
        self.latency_history[step_name].append((timestamp, duration_ms))
        service_name = step_name.split("_")[0] if "_" in step_name else step_name
        self.service_tps[service_name].append((timestamp, 1))
        result = {
            "transaction_id": transaction_id,
            "step_name": step_name,
            "current_tps": round(current_tps, 2),
            "p50_ms": round(p50, 2),
            "p90_ms": round(p90, 2),
            "p99_ms": round(p99, 2),
            "latency_spike_detected": latency_spike_detected,
            "baseline_p99_ms": self.baseline_p99.get(step_name, None),
            "success": success,
            "recorded_at": timestamp
        }
        return result

    def _percentile_from_sorted(self, sorted_list: List[float], percentile: float) -> float:
        """从已排序列表计算百分位值"""
        if not sorted_list:
            return 0.0
        n = len(sorted_list)
        rank = percentile * (n - 1)
        lower_idx = int(math.floor(rank))
        upper_idx = int(math.ceil(rank))
        if lower_idx == upper_idx:
            return sorted_list[lower_idx]
        fraction = rank - lower_idx
        return sorted_list[lower_idx] * (1 - fraction) + sorted_list[upper_idx] * fraction

    def detect_slow_transaction(self, service_name: str) -> Dict:
        """分析最近1小时事务数据，检测慢步骤、性能退化和失败率关联"""
        now = time.time()
        lookback_start = now - self.SLOW_TRANSACTION_LOOKBACK_HOURS * 3600
        relevant_steps = {}
        for step_name, metrics in self.step_metrics.items():
            step_service = step_name.split("_")[0] if "_" in step_name else step_name
            if step_service != service_name:
                continue
            recent = [m for m in metrics if m["timestamp"] >= lookback_start]
            if not recent:
                continue
            relevant_steps[step_name] = recent
        if not relevant_steps:
            return {"service_name": service_name, "status": "no_data", "slow_steps": []}
        step_analysis = []
        for step_name, metrics in relevant_steps.items():
            durations = [m["duration_ms"] for m in metrics]
            failures = [m for m in metrics if not m["success"]]
            avg_latency = sum(durations) / len(durations) if durations else 0
            sorted_durations = sorted(durations)
            p50 = self._percentile_from_sorted(sorted_durations, 0.50)
            p90 = self._percentile_from_sorted(sorted_durations, 0.90)
            p99 = self._percentile_from_sorted(sorted_durations, 0.99)
            failure_rate = len(failures) / len(metrics) if metrics else 0
            sla_threshold = self.sla_thresholds.get(step_name, 1000.0)
            exceeds_sla = p99 > sla_threshold
            timestamps = [m["timestamp"] for m in metrics]
            latency_values = [m["duration_ms"] for m in metrics]
            slope = self._linear_regression_slope(timestamps, latency_values)
            relative_slope = slope / avg_latency if avg_latency > 0 else 0
            is_degrading = relative_slope > self.DEGRADATION_SLOPE_THRESHOLD
            step_analysis.append({
                "step_name": step_name,
                "avg_latency_ms": round(avg_latency, 2),
                "p50_ms": round(p50, 2),
                "p90_ms": round(p90, 2),
                "p99_ms": round(p99, 2),
                "sla_ms": sla_threshold,
                "exceeds_sla": exceeds_sla,
                "failure_rate": round(failure_rate, 4),
                "sample_count": len(metrics),
                "degradation_slope": round(relative_slope, 4),
                "is_degrading": is_degrading,
                "correlation_with_failures": self._correlate_latency_failure(metrics)
            })
        slow_steps = [s for s in step_analysis if s["exceeds_sla"] or s["is_degrading"]]
        slow_steps.sort(key=lambda s: s["p99_ms"], reverse=True)
        return {
            "service_name": service_name,
            "status": "slow_detected" if slow_steps else "healthy",
            "analysis_period_hours": self.SLOW_TRANSACTION_LOOKBACK_HOURS,
            "total_steps_analyzed": len(step_analysis),
            "slow_steps": slow_steps,
            "degrading_steps": [s for s in step_analysis if s["is_degrading"]],
            "high_failure_steps": [s for s in step_analysis if s["failure_rate"] > 0.05]
        }

    def _linear_regression_slope(self, x_values: List[float], y_values: List[float]) -> float:
        """计算线性回归斜率，用于检测延迟退化趋势"""
        n = len(x_values)
        if n < 2:
            return 0.0
        sum_x = sum(x_values)
        sum_y = sum(y_values)
        sum_xy = sum(x * y for x, y in zip(x_values, y_values))
        sum_x2 = sum(x * x for x in x_values)
        denominator = n * sum_x2 - sum_x * sum_x
        if abs(denominator) < 1e-10:
            return 0.0
        slope = (n * sum_xy - sum_x * sum_y) / denominator
        x_range = max(x_values) - min(x_values)
        if x_range > 0:
            slope_per_hour = slope * 3600 / x_range * (sum_y / n)
            return slope_per_hour / (sum_y / n) if (sum_y / n) > 0 else 0.0
        return 0.0

    def _correlate_latency_failure(self, metrics: List[Dict]) -> float:
        """关联延迟和失败率"""
        if len(metrics) < 10:
            return 0.0
        sorted_by_time = sorted(metrics, key=lambda m: m["timestamp"])
        bucket_size = max(1, len(sorted_by_time) // 10)
        buckets = []
        for i in range(0, len(sorted_by_time), bucket_size):
            bucket = sorted_by_time[i:i + bucket_size]
            if bucket:
                avg_latency = sum(m["duration_ms"] for m in bucket) / len(bucket)
                failure_rate = sum(1 for m in bucket if not m["success"]) / len(bucket)
                buckets.append((avg_latency, failure_rate))
        if len(buckets) < 3:
            return 0.0
        latencies = [b[0] for b in buckets]
        failures = [b[1] for b in buckets]
        n = len(buckets)
        sum_x = sum(latencies)
        sum_y = sum(failures)
        sum_xy = sum(x * y for x, y in zip(latencies, failures))
        sum_x2 = sum(x * x for x in latencies)
        sum_y2 = sum(y * y for y in failures)
        denominator = math.sqrt((n * sum_x2 - sum_x ** 2) * (n * sum_y2 - sum_y ** 2))
        if abs(denominator) < 1e-10:
            return 0.0
        correlation = (n * sum_xy - sum_x * sum_y) / denominator
        return round(correlation, 4)

    def optimize_transaction_path(self, transaction_definition: Dict) -> Dict:
        """分析事务步骤DAG，识别可并行化的独立步骤，估算时间节省，建议批处理和快速验证"""
        steps = transaction_definition.get("steps", [])
        dependencies = transaction_definition.get("dependencies", {})
        if not steps:
            return {"status": "no_steps", "optimizations": []}
        for step in steps:
            self.transaction_definitions[step["name"]] = step
        graph = self._build_dependency_graph(steps, dependencies)
        topo_order = self._topological_sort(graph)
        if topo_order is None:
            return {"status": "circular_dependency", "optimizations": []}
        parallel_groups = self._detect_parallel_groups(topo_order, graph)
        sequential_time = sum(step.get("avg_duration_ms", 100) for step in steps)
        optimized_time = 0
        for group in parallel_groups:
            group_max = max(self.transaction_definitions.get(s, {}).get("avg_duration_ms", 100)
                           for s in group)
            optimized_time += group_max
        time_savings_ms = sequential_time - optimized_time
        time_savings_pct = (time_savings_ms / sequential_time * 100) if sequential_time > 0 else 0
        db_writes = [s for s in steps if s.get("type") == "database_write"]
        batch_suggestions = []
        if len(db_writes) >= 2:
            sequential_db_time = sum(s.get("avg_duration_ms", 50) for s in db_writes)
            batch_time = max(s.get("avg_duration_ms", 50) for s in db_writes) * 1.2
            batch_savings = sequential_db_time - batch_time
            batch_suggestions.append({
                "type": "batch_database_writes",
                "steps": [s["name"] for s in db_writes],
                "current_time_ms": sequential_db_time,
                "optimized_time_ms": batch_time,
                "savings_ms": batch_savings,
                "description": f"将{len(db_writes)}个顺序数据库写操作合并为批量提交"
            })
        validation_steps = [s for s in steps if s.get("type") == "validation"]
        fail_fast_suggestions = []
        resource_locking_steps = [s for s in steps if s.get("requires_resource_lock", False)]
        if validation_steps and resource_locking_steps:
            validation_time = sum(s.get("avg_duration_ms", 10) for s in validation_steps)
            fail_fast_suggestions.append({
                "type": "fail_fast_validation",
                "validation_steps": [s["name"] for s in validation_steps],
                "resource_locking_steps": [s["name"] for s in resource_locking_steps],
                "savings_description": f"将验证步骤({validation_time}ms)移至资源锁定前执行，避免无效锁定",
                "estimated_waste_prevention_pct": min(50, len(validation_steps) * 15)
            })
        all_optimizations = []
        if time_savings_ms > self.PARALLEL_GROUP_MIN_SAVINGS_MS:
            all_optimizations.append({
                "type": "parallelization",
                "parallel_groups": parallel_groups,
                "sequential_time_ms": sequential_time,
                "optimized_time_ms": optimized_time,
                "savings_ms": time_savings_ms,
                "savings_pct": round(time_savings_pct, 1)
            })
        all_optimizations.extend(batch_suggestions)
        all_optimizations.extend(fail_fast_suggestions)
        return {
            "status": "optimized",
            "transaction_name": transaction_definition.get("name", "unknown"),
            "step_count": len(steps),
            "sequential_time_ms": sequential_time,
            "optimized_time_ms": optimized_time,
            "total_savings_ms": time_savings_ms + sum(s.get("savings_ms", 0) for s in batch_suggestions),
            "optimizations": all_optimizations,
            "parallel_group_count": len(parallel_groups),
            "topological_order": topo_order
        }

    def _build_dependency_graph(self, steps: List[Dict], dependencies: Dict) -> Dict:
        """构建步骤依赖图"""
        graph = {"nodes": set(), "edges": defaultdict(set), "reverse_edges": defaultdict(set)}
        for step in steps:
            graph["nodes"].add(step["name"])
        for step_name, deps in dependencies.items():
            if isinstance(deps, str):
                deps = [deps]
            for dep in deps:
                graph["edges"][step_name].add(dep)
                graph["reverse_edges"][dep].add(step_name)
        return graph

    def _topological_sort(self, graph: Dict) -> Optional[List[str]]:
        """拓扑排序，检测循环依赖"""
        in_degree = defaultdict(int)
        nodes = graph["nodes"]
        for node in nodes:
            in_degree[node] = len(graph["edges"].get(node, set()))
        queue = deque([n for n in nodes if in_degree[n] == 0])
        result = []
        while queue:
            node = queue.popleft()
            result.append(node)
            for dependent in graph["reverse_edges"].get(node, set()):
                in_degree[dependent] -= 1
                if in_degree[dependent] == 0:
                    queue.append(dependent)
        if len(result) != len(nodes):
            return None
        return result

    def _detect_parallel_groups(self, topo_order: List[str], graph: Dict) -> List[List[str]]:
        """检测可并行执行的步骤组"""
        if not topo_order:
            return []
        groups = []
        completed = set()
        remaining = list(topo_order)
        while remaining:
            parallel_group = []
            new_remaining = []
            for step in remaining:
                deps = graph["edges"].get(step, set())
                if deps.issubset(completed):
                    parallel_group.append(step)
                else:
                    new_remaining.append(step)
            if not parallel_group:
                groups.append([remaining[0]])
                completed.add(remaining[0])
                remaining = remaining[1:]
            else:
                groups.append(parallel_group)
                completed.update(parallel_group)
                remaining = new_remaining
        return groups

    def generate_transaction_dashboard(self) -> Dict:
        """生成事务监控仪表盘：TPS、延迟热力图、失败率趋势、Top10慢事务、资源关联"""
        now = time.time()
        tps_per_service = {}
        for service_name, tps_entries in self.service_tps.items():
            recent_entries = [(ts, count) for ts, count in tps_entries if now - ts < 60]
            tps_per_service[service_name] = len(recent_entries)
        all_step_names = set(self.step_metrics.keys())
        latency_heatmap = {}
        for step_name in all_step_names:
            service = step_name.split("_")[0] if "_" in step_name else "default"
            recent_metrics = [m for m in self.step_metrics[step_name] if now - m["timestamp"] < 300]
            if recent_metrics:
                avg_latency = sum(m["duration_ms"] for m in recent_metrics) / len(recent_metrics)
                color_code = self._latency_to_color(avg_latency)
                if service not in latency_heatmap:
                    latency_heatmap[service] = {}
                latency_heatmap[service][step_name] = {
                    "avg_latency_ms": round(avg_latency, 2),
                    "color": color_code
                }
        hourly_failure_rate = {}
        for hour_offset in range(24):
            hour_start = now - (hour_offset + 1) * 3600
            hour_end = now - hour_offset * 3600
            total_in_hour = 0
            failures_in_hour = 0
            for step_name, metrics in self.step_metrics.items():
                for m in metrics:
                    if hour_start <= m["timestamp"] < hour_end:
                        total_in_hour += 1
                        if not m["success"]:
                            failures_in_hour += 1
            hour_label = datetime.fromtimestamp(hour_start).strftime("%H:00")
            rate = failures_in_hour / total_in_hour if total_in_hour > 0 else 0
            hourly_failure_rate[hour_label] = round(rate, 4)
        slow_transactions = []
        for txn_id, steps in self.transaction_steps.items():
            if not steps:
                continue
            total_duration = sum(s["duration_ms"] for s in steps)
            has_failure = any(not s["success"] for s in steps)
            step_breakdown = [
                {
                    "step_name": s["step_name"],
                    "duration_ms": s["duration_ms"],
                    "success": s["success"]
                }
                for s in sorted(steps, key=lambda s: s["timestamp"])
            ]
            slow_transactions.append({
                "transaction_id": txn_id,
                "total_duration_ms": round(total_duration, 2),
                "step_count": len(steps),
                "has_failure": has_failure,
                "breakdown": step_breakdown
            })
        slow_transactions.sort(key=lambda t: t["total_duration_ms"], reverse=True)
        top_10_slow = slow_transactions[:10]
        resource_correlation = {}
        for step_name in all_step_names:
            if step_name in self.latency_history and step_name in self.resource_metrics:
                latency_vals = self.latency_history[step_name]
                resource_vals = self.resource_metrics[step_name]
                cpu_correlation = self._compute_resource_latency_correlation(
                    latency_vals, resource_vals, "cpu_usage"
                )
                memory_correlation = self._compute_resource_latency_correlation(
                    latency_vals, resource_vals, "memory_usage"
                )
                resource_correlation[step_name] = {
                    "cpu_correlation": round(cpu_correlation, 4),
                    "memory_correlation": round(memory_correlation, 4)
                }
        return {
            "generated_at": now,
            "tps_per_service": tps_per_service,
            "total_tps": sum(tps_per_service.values()),
            "latency_heatmap": latency_heatmap,
            "hourly_failure_rate": hourly_failure_rate,
            "top_10_slow_transactions": top_10_slow,
            "resource_correlation": resource_correlation,
            "active_services": len(tps_per_service),
            "total_steps_monitored": len(all_step_names)
        }

    def _latency_to_color(self, latency_ms: float) -> str:
        """将延迟值映射为热力图颜色"""
        if latency_ms < 100:
            return "green"
        if latency_ms < 500:
            return "yellow"
        if latency_ms < 1000:
            return "orange"
        return "red"

    def _compute_resource_latency_correlation(self, latency_data: List[Tuple[float, float]],
                                               resource_data: List[Tuple[float, Dict]],
                                               resource_key: str) -> float:
        """计算资源使用与延迟的相关系数"""
        latency_dict = {ts: val for ts, val in latency_data}
        resource_dict = {}
        for ts, metrics in resource_data:
            if resource_key in metrics:
                resource_dict[ts] = metrics[resource_key]
        common_timestamps = set(latency_dict.keys()) & set(resource_dict.keys())
        if len(common_timestamps) < 5:
            return 0.0
        paired = [(latency_dict[t], resource_dict[t]) for t in common_timestamps]
        n = len(paired)
        sum_x = sum(p[0] for p in paired)
        sum_y = sum(p[1] for p in paired)
        sum_xy = sum(p[0] * p[1] for p in paired)
        sum_x2 = sum(p[0] ** 2 for p in paired)
        sum_y2 = sum(p[1] ** 2 for p in paired)
        denominator = math.sqrt((n * sum_x2 - sum_x ** 2) * (n * sum_y2 - sum_y ** 2))
        if abs(denominator) < 1e-10:
            return 0.0
        return (n * sum_xy - sum_x * sum_y) / denominator

    def set_sla_threshold(self, step_name: str, threshold_ms: float) -> None:
        """设置步骤的SLA阈值"""
        self.sla_thresholds[step_name] = threshold_ms

    def record_resource_metrics(self, step_name: str, cpu_usage: float,
                                 memory_usage: float) -> None:
        """记录资源使用指标"""
        timestamp = time.time()
        self.resource_metrics[step_name].append((timestamp, {
            "cpu_usage": cpu_usage,
            "memory_usage": memory_usage
        }))
```

## 异常场景补充

### 场景：事务超时导致雪崩效应
```
trigger: 某个核心服务的数据库连接池耗尽，导致该服务事务响应时间从50ms飙升至30秒，上游服务的超时重试进一步加重负载，引发级联故障
detection: 1) 实时监控各服务TPS，当TPS突然下降超过50%时触发雪崩预警；2) 检测重试率突增，当重试请求占比超过总请求的30%时判定异常；3) 监控连接池使用率，当活跃连接数持续超过池容量的90%时告警；4) 检测p99延迟在1分钟内增长超过5倍；5) 监控服务间调用图的扇出系数，识别重试风暴
handling: 1) 立即触发断路器，对超时服务停止新的请求接入，返回降级响应；2) 对已超时的事务执行补偿回滚，释放持有的资源锁；3) 缩短上游服务的事务超时时间，快速失败而非等待；4) 启用请求队列和限流，控制进入系统的请求速率；5) 优先保障核心交易路径，非关键步骤降级为异步处理；6) 通知运维团队扩容受影响服务
prevention: 1) 为每个服务配置合理的超时时间和重试策略（指数退避+最大重试次数）；2) 实现断路器模式，当错误率超过阈值自动熔断；3) 部署舱壁隔离，不同服务使用独立的连接池和线程池；4) 设置请求限流和背压机制，防止过载传播；5) 进行混沌工程测试，定期模拟服务超时验证系统韧性；6) 建立服务降级预案，明确各服务的最低可用保障等级
```

### 场景：监控数据延迟掩盖真实问题
```
trigger: 监控系统自身的消息队列积压，导致性能指标延迟5-10分钟才到达仪表盘，运维人员看到的"正常"指标实际是历史数据，真实的系统故障被延迟暴露
detection: 1) 监控数据采集时间戳与当前时间的差值，当延迟超过30秒时标记为数据过期；2) 在仪表盘上显示数据新鲜度指标，当指标变黄/变红时提醒运维；3) 检测监控管道自身的消息堆积量，超过阈值时告警；4) 对比多个监控源的同一指标，若出现显著时间偏移则判定延迟异常
handling: 1) 在仪表盘上强制显示数据延迟警告横幅，标注当前数据的实际时间范围；2) 切换到备用监控通道（如直接查询应用日志或数据库指标表）；3) 对延迟到达的数据进行时间重标定，确保不会误判趋势；4) 对积压的监控数据启用采样聚合，仅保留关键百分位值加速消费；5) 临时增加监控消费者实例数量处理积压
prevention: 1) 监控系统使用独立的消息队列和存储，与业务系统隔离；2) 为监控数据设置TTL和优先级，关键指标优先处理；3) 实现监控数据的直连通道作为备用，绕过消息队列；4) 在监控仪表盘上始终显示数据新鲜度指示器；5) 定期压测监控管道，确保其吞吐量是业务峰值的3倍以上；6) 部署多级监控：实时采样（1秒级）+ 聚合指标（分钟级）+ 历史归档（小时级）
```

### 场景：事务重试导致重复执行
```
trigger: 分布式事务在提交阶段因网络抖动超时，协调者未收到确认，触发重试逻辑，但参与者的实际操作已经成功执行，重试导致同一操作被执行两次（如重复扣款、重复发货）
detection: 1) 在业务层校验幂等性，检查目标账户的变更记录是否已存在相同transaction_id的条目；2) 监控同一transaction_id的步骤执行次数，超过1次即标记为疑似重复；3) 对账系统检测账户余额与交易流水不一致；4) 数据库唯一约束冲突（duplicate key error）作为重复执行的直接证据
handling: 1) 立即暂停受影响事务的重试队列，防止更多重复执行；2) 执行数据一致性审计，扫描所有在故障窗口期内执行的事务，标记重复操作；3) 对重复的扣款操作发起自动退款流程；4) 对重复的库存扣减执行回补；5) 通知受影响客户并提供对账报告；6) 修复已完成事务的状态标记，确保协调者记录最终一致性
prevention: 1) 每个事务参与者必须实现幂等性，使用transaction_id+step_id作为唯一键，执行前先检查是否已完成；2) 使用TCC（Try-Confirm-Cancel）模式替代简单的重试，Confirm阶段天然幂等；3) 在数据库层面为关键操作添加唯一约束，从物理层面防止重复插入；4) 事务状态机采用"最终状态锁定"，已确认的事务拒绝任何新的状态转换；5) 实现分布式锁确保同一事务同一时刻只有一个执行实例；6) 建立定期对账机制，自动发现和修复不一致
```

## 分布式事务性能优化与监控完整实现

```python
import time
import math
import hashlib
import json
import redis
import threading
from collections import defaultdict, deque
from datetime import datetime, timedelta
from concurrent.futures import ThreadPoolExecutor
from typing import Dict, List, Tuple, Optional, Any


class TransactionPerformanceService:
    """分布式事务性能优化与监控服务，提供指标采集、慢事务检测、路径优化和仪表盘生成"""

    def __init__(self, redis_client: redis.Redis, db_connection, alert_channel=None):
        self.redis = redis_client
        self.db = db_connection
        self.alert_channel = alert_channel
        self.sla_threshold_ms = 500
        self.baseline_window_hours = 24
        self.tps_window_seconds = 60
        self.sliding_window_size = 1000
        self.circular_buffers: Dict[str, deque] = {}
        self.circular_buffer_lock = threading.Lock()
        self.baseline_cache: Dict[str, float] = {}
        self.baseline_cache_timestamp: Dict[str, float] = {}
        self.baseline_cache_ttl = 3600
        self.spike_alert_cooldown: Dict[str, float] = {}
        self.spike_cooldown_seconds = 300

    def record_transaction_metrics(self, transaction_id: str, step_name: str,
                                    duration_ms: float, success: bool,
                                    metadata: Optional[Dict] = None) -> Dict:
        """记录事务步骤指标到Redis有序集合，维护滑动窗口，计算TPS，检测延迟尖峰"""
        timestamp = time.time()
        service_name = step_name.split(":")[0] if ":" in step_name else "default"
        redis_key = f"tx:{service_name}:latency"
        metric_entry = {
            "transaction_id": transaction_id,
            "step_name": step_name,
            "duration_ms": duration_ms,
            "success": success,
            "metadata": metadata or {},
            "timestamp": timestamp
        }
        entry_json = json.dumps(metric_entry)
        self.redis.zadd(redis_key, {entry_json: timestamp})

        cutoff_timestamp = timestamp - 86400
        self.redis.zremrangebyscore(redis_key, "-inf", cutoff_timestamp)

        with self.circular_buffer_lock:
            if service_name not in self.circular_buffers:
                self.circular_buffers[service_name] = deque(maxlen=self.sliding_window_size)
            buf = self.circular_buffers[service_name]
            buf.append({
                "duration_ms": duration_ms,
                "success": success,
                "timestamp": timestamp
            })

        now_dt = datetime.fromtimestamp(timestamp)
        minute_key = now_dt.strftime("%Y%m%d%H%M")
        tps_key = f"tx:{service_name}:tps:{minute_key}"
        self.redis.incr(tps_key)
        self.redis.expire(tps_key, 120)

        tps_window_start = timestamp - self.tps_window_seconds
        tps_count = 0
        with self.circular_buffer_lock:
            buf = self.circular_buffers.get(service_name, deque())
            for entry in buf:
                if entry["timestamp"] >= tps_window_start:
                    tps_count += 1
        current_tps = tps_count / self.tps_window_seconds

        sorted_durations = []
        with self.circular_buffer_lock:
            buf = self.circular_buffers.get(service_name, deque())
            for entry in buf:
                sorted_durations.append(entry["duration_ms"])
        sorted_durations.sort()
        if len(sorted_durations) >= 1:
            p99_index = max(0, int(len(sorted_durations) * 0.99) - 1)
            current_p99 = sorted_durations[p99_index]
        else:
            current_p99 = 0.0

        baseline_p99 = self._get_baseline_p99(service_name, timestamp)

        spike_detected = False
        spike_alert_id = ""
        if baseline_p99 > 0 and current_p99 > 2 * baseline_p99:
            cooldown_key = f"{service_name}:spike_cooldown"
            last_spike_time = self.spike_alert_cooldown.get(cooldown_key, 0)
            if timestamp - last_spike_time > self.spike_cooldown_seconds:
                spike_detected = True
                self.spike_alert_cooldown[cooldown_key] = timestamp
                spike_alert_id = f"spike:{service_name}:{int(timestamp)}"
                alert_data = {
                    "alert_id": spike_alert_id,
                    "service_name": service_name,
                    "current_p99": current_p99,
                    "baseline_p99": baseline_p99,
                    "ratio": current_p99 / baseline_p99 if baseline_p99 > 0 else float("inf"),
                    "timestamp": timestamp,
                    "transaction_id": transaction_id,
                    "step_name": step_name
                }
                alert_json = json.dumps(alert_data)
                self.redis.lpush("tx:alert_queue", alert_json)
                if self.alert_channel:
                    self.alert_channel.publish("transaction_alerts", alert_json)

        success_key = f"tx:{service_name}:success_count:{minute_key}"
        failure_key = f"tx:{service_name}:failure_count:{minute_key}"
        if success:
            self.redis.incr(success_key)
        else:
            self.redis.incr(failure_key)
        self.redis.expire(success_key, 120)
        self.redis.expire(failure_key, 120)

        return {
            "recorded": True,
            "service_name": service_name,
            "current_tps": round(current_tps, 2),
            "current_p99": round(current_p99, 2),
            "baseline_p99": round(baseline_p99, 2),
            "spike_detected": spike_detected,
            "spike_alert_id": spike_alert_id,
            "buffer_size": len(self.circular_buffers.get(service_name, deque()))
        }

    def _get_baseline_p99(self, service_name: str, current_timestamp: float) -> float:
        """计算过去24小时的p99中位数作为基线"""
        cache_key = f"baseline:{service_name}"
        cached_time = self.baseline_cache_timestamp.get(cache_key, 0)
        if current_timestamp - cached_time < self.baseline_cache_ttl:
            return self.baseline_cache.get(cache_key, 0.0)

        redis_key = f"tx:{service_name}:latency"
        start_time = current_timestamp - self.baseline_window_hours * 3600
        entries = self.redis.zrangebyscore(redis_key, start_time, current_timestamp)

        if not entries:
            self.baseline_cache[cache_key] = 0.0
            self.baseline_cache_timestamp[cache_key] = current_timestamp
            return 0.0

        hourly_p99_values = []
        hour_buckets: Dict[str, List[float]] = defaultdict(list)
        for entry_bytes in entries:
            entry = json.loads(entry_bytes)
            entry_time = entry.get("timestamp", 0)
            hour_label = datetime.fromtimestamp(entry_time).strftime("%Y%m%d%H")
            hour_buckets[hour_label].append(entry.get("duration_ms", 0))

        for hour_label, durations in hour_buckets.items():
            durations.sort()
            if len(durations) >= 1:
                p99_idx = max(0, int(len(durations) * 0.99) - 1)
                hourly_p99_values.append(durations[p99_idx])

        if not hourly_p99_values:
            self.baseline_cache[cache_key] = 0.0
            self.baseline_cache_timestamp[cache_key] = current_timestamp
            return 0.0

        hourly_p99_values.sort()
        median_index = len(hourly_p99_values) // 2
        baseline = hourly_p99_values[median_index]

        self.baseline_cache[cache_key] = baseline
        self.baseline_cache_timestamp[cache_key] = current_timestamp
        return baseline

    def detect_slow_transaction(self, service_name: str) -> Dict:
        """检测慢事务：查询过去1小时数据，计算分位数，识别超SLA步骤，执行线性回归和关联分析"""
        current_timestamp = time.time()
        one_hour_ago = current_timestamp - 3600

        query = """
            SELECT step_name, duration_ms, success, timestamp
            FROM transaction_metrics
            WHERE service_name = %s AND timestamp >= %s
            ORDER BY timestamp ASC
        """
        cursor = self.db.cursor()
        cursor.execute(query, (service_name, datetime.fromtimestamp(one_hour_ago)))
        rows = cursor.fetchall()
        cursor.close()

        if not rows:
            return {
                "service_name": service_name,
                "problematic_steps": [],
                "message": "No transaction data found in the last hour"
            }

        step_data: Dict[str, List[Dict]] = defaultdict(list)
        for row in rows:
            step_name = row[0]
            step_data[step_name].append({
                "duration_ms": float(row[1]),
                "success": bool(row[2]),
                "timestamp": float(row[3].timestamp()) if hasattr(row[3], "timestamp") else row[3]
            })

        step_stats = {}
        for step_name, measurements in step_data.items():
            durations = sorted([m["duration_ms"] for m in measurements])
            total = len(durations)
            if total == 0:
                continue

            avg = sum(durations) / total
            p50_index = max(0, int(total * 0.50) - 1)
            p90_index = max(0, int(total * 0.90) - 1)
            p99_index = max(0, int(total * 0.99) - 1)
            p50 = durations[p50_index]
            p90 = durations[p90_index]
            p99 = durations[p99_index]

            failure_count = sum(1 for m in measurements if not m["success"])
            failure_rate = (failure_count / total) * 100 if total > 0 else 0.0

            sorted_by_time = sorted(measurements, key=lambda m: m["timestamp"])
            timestamps = [m["timestamp"] for m in sorted_by_time]
            durations_ts = [m["duration_ms"] for m in sorted_by_time]

            n = len(timestamps)
            if n >= 2:
                t_min = timestamps[0]
                t_max = timestamps[-1]
                t_range = t_max - t_min if t_max > t_min else 1.0
                x_vals = [(t - t_min) / t_range for t in timestamps]
                x_mean = sum(x_vals) / n
                y_mean = sum(durations_ts) / n
                numerator = sum((x_vals[i] - x_mean) * (durations_ts[i] - y_mean) for i in range(n))
                denominator = sum((x_vals[i] - x_mean) ** 2 for i in range(n))
                slope = numerator / denominator if denominator != 0 else 0.0
                slope_pct_per_hour = (slope / y_mean) * 100 if y_mean != 0 else 0.0
            else:
                slope = 0.0
                slope_pct_per_hour = 0.0

            degradation_trend = slope_pct_per_hour > 10

            exceeds_sla = p99 > self.sla_threshold_ms
            critical = exceeds_sla and failure_rate > 5.0

            if critical:
                severity = "critical"
            elif exceeds_sla:
                severity = "warning"
            elif degradation_trend:
                severity = "degrading"
            else:
                severity = "normal"

            root_cause_hints = []
            if critical:
                root_cause_hints.append("High latency combined with elevated failure rate suggests resource exhaustion or downstream dependency failure")
            if exceeds_sla and failure_rate <= 5.0:
                root_cause_hints.append("Latency exceeds SLA but failure rate is normal, possible cause: increased load or inefficient query")
            if degradation_trend:
                root_cause_hints.append("Positive latency trend detected (slope > 10%/hour), indicates progressive performance degradation")
            if failure_rate > 20.0:
                root_cause_hints.append("Very high failure rate suggests the step may be fundamentally broken or its dependency is down")
            if not root_cause_hints and severity == "normal":
                root_cause_hints.append("No issues detected")

            step_stats[step_name] = {
                "avg": round(avg, 2),
                "p50": round(p50, 2),
                "p90": round(p90, 2),
                "p99": round(p99, 2),
                "failure_rate": round(failure_rate, 2),
                "sample_count": total,
                "slope_pct_per_hour": round(slope_pct_per_hour, 2),
                "degradation_trend": degradation_trend,
                "exceeds_sla": exceeds_sla,
                "critical": critical,
                "severity": severity,
                "root_cause_hints": root_cause_hints
            }

        problematic_steps = []
        for step_name, stats in step_stats.items():
            if stats["severity"] != "normal":
                problematic_steps.append((step_name, stats))

        severity_rank = {"critical": 0, "warning": 1, "degrading": 2, "normal": 3}
        problematic_steps.sort(key=lambda x: (severity_rank.get(x[1]["severity"], 99), -x[1]["p99"]))

        return {
            "service_name": service_name,
            "total_steps_analyzed": len(step_stats),
            "problematic_steps": [
                {"step_name": name, **stats} for name, stats in problematic_steps
            ],
            "all_step_stats": step_stats
        }

    def optimize_transaction_path(self, transaction_definition: Dict) -> Dict:
        """解析事务步骤DAG，拓扑排序，识别可并行步骤，建议批量操作和快速失败验证"""
        steps = transaction_definition.get("steps", [])
        dependencies = transaction_definition.get("dependencies", [])
        step_map = {}
        for step in steps:
            step_map[step["name"]] = {
                "name": step["name"],
                "type": step.get("type", "generic"),
                "estimated_duration_ms": step.get("estimated_duration_ms", 100),
                "resource": step.get("resource", None),
                "validates_input": step.get("validates_input", False),
                "locks_resource": step.get("locks_resource", False)
            }

        graph: Dict[str, List[str]] = defaultdict(list)
        in_degree: Dict[str, int] = defaultdict(int)
        reverse_graph: Dict[str, List[str]] = defaultdict(list)
        for dep in dependencies:
            source = dep["from"]
            target = dep["to"]
            graph[source].append(target)
            reverse_graph[target].append(source)
            in_degree[target] += 1

        for step_name in step_map:
            if step_name not in in_degree:
                in_degree[step_name] = 0

        topo_order = []
        queue = [name for name, deg in in_degree.items() if deg == 0]
        queue.sort()
        while queue:
            current = queue.pop(0)
            topo_order.append(current)
            for neighbor in sorted(graph[current]):
                in_degree[neighbor] -= 1
                if in_degree[neighbor] == 0:
                    queue.append(neighbor)
                    queue.sort()

        if len(topo_order) != len(step_map):
            return {
                "error": "Circular dependency detected in transaction definition",
                "topological_sort_incomplete": True
            }

        data_dependencies: Dict[str, set] = defaultdict(set)
        for dep in dependencies:
            if dep.get("data_flow", False):
                data_dependencies[dep["to"]].add(dep["from"])

        parallel_groups = []
        assigned: set = set()
        remaining = set(topo_order)

        while remaining:
            ready = []
            for step_name in remaining:
                prereqs = set(reverse_graph[step_name])
                if prereqs.issubset(assigned):
                    ready.append(step_name)

            if not ready:
                break

            independent_ready = []
            dependent_ready = []
            for step_name in ready:
                if len(data_dependencies[step_name]) == 0:
                    independent_ready.append(step_name)
                else:
                    dependent_ready.append(step_name)

            group = []
            for step_name in independent_ready:
                group.append(step_name)
            for step_name in dependent_ready:
                group.append(step_name)

            for step_name in group:
                assigned.add(step_name)
                remaining.discard(step_name)

            parallel_groups.append(group)

        sequential_time = sum(step_map[name]["estimated_duration_ms"] for name in topo_order)

        parallel_time = 0
        for group in parallel_groups:
            group_max = max(step_map[name]["estimated_duration_ms"] for name in group) if group else 0
            parallel_time += group_max

        time_savings = sequential_time - parallel_time
        savings_pct = (time_savings / sequential_time * 100) if sequential_time > 0 else 0.0

        parallel_suggestions = []
        for i, group in enumerate(parallel_groups):
            if len(group) > 1:
                parallel_suggestions.append({
                    "group_index": i,
                    "parallel_steps": group,
                    "sequential_duration": sum(step_map[n]["estimated_duration_ms"] for n in group),
                    "parallel_duration": max(step_map[n]["estimated_duration_ms"] for n in group),
                    "savings_ms": sum(step_map[n]["estimated_duration_ms"] for n in group) - max(step_map[n]["estimated_duration_ms"] for n in group)
                })

        db_write_steps = [name for name in topo_order if step_map[name]["type"] == "db_write"]
        batch_suggestions = []
        if len(db_write_steps) >= 2:
            consecutive_db_writes = []
            current_batch = []
            for name in topo_order:
                if step_map[name]["type"] == "db_write":
                    current_batch.append(name)
                else:
                    if len(current_batch) >= 2:
                        consecutive_db_writes.append(current_batch)
                    current_batch = []
            if len(current_batch) >= 2:
                consecutive_db_writes.append(current_batch)

            for batch in consecutive_db_writes:
                individual_duration = sum(step_map[n]["estimated_duration_ms"] for n in batch)
                batch_duration = max(step_map[n]["estimated_duration_ms"] for n in batch) + len(batch) * 5
                batch_savings = individual_duration - batch_duration
                batch_suggestions.append({
                    "steps_to_batch": batch,
                    "individual_duration_ms": individual_duration,
                    "batch_duration_ms": batch_duration,
                    "savings_ms": batch_savings,
                    "recommendation": f"Combine {len(batch)} sequential DB writes into a single batch operation"
                })

        validation_steps = [name for name in topo_order if step_map[name].get("validates_input", False)]
        lock_steps = [name for name in topo_order if step_map[name].get("locks_resource", False)]
        fail_fast_suggestions = []
        if validation_steps and lock_steps:
            first_lock_idx = min(topo_order.index(n) for n in lock_steps)
            last_validation_idx = max(topo_order.index(n) for n in validation_steps)
            if last_validation_idx > first_lock_idx:
                moved_validations = [n for n in validation_steps if topo_order.index(n) > first_lock_idx]
                fail_fast_suggestions.append({
                    "type": "move_validation_before_lock",
                    "validation_steps_to_move": moved_validations,
                    "lock_steps_affected": [n for n in lock_steps if topo_order.index(n) < last_validation_idx],
                    "reason": "Move input validation before resource locking to fail fast and avoid unnecessary lock contention",
                    "estimated_savings_ms": sum(step_map[n]["estimated_duration_ms"] for n in moved_validations) * 2
                })

        steps_without_validation = []
        for name in topo_order:
            if step_map[name]["type"] == "db_write" and not step_map[name].get("validates_input", False):
                steps_without_validation.append(name)
        if steps_without_validation:
            fail_fast_suggestions.append({
                "type": "add_input_validation",
                "steps_needing_validation": steps_without_validation,
                "reason": "Add input validation before DB write operations to fail fast on invalid data",
                "estimated_failure_avoidance_pct": 15.0
            })

        total_estimated_savings = time_savings
        for bs in batch_suggestions:
            total_estimated_savings += bs["savings_ms"]
        for ff in fail_fast_suggestions:
            total_estimated_savings += ff.get("estimated_savings_ms", 0)

        optimized_time = max(0, sequential_time - total_estimated_savings)
        total_savings_pct = (total_estimated_savings / sequential_time * 100) if sequential_time > 0 else 0.0

        return {
            "original_sequential_time_ms": sequential_time,
            "optimized_time_ms": optimized_time,
            "total_savings_ms": total_estimated_savings,
            "total_savings_pct": round(total_savings_pct, 2),
            "topological_order": topo_order,
            "parallel_groups": parallel_groups,
            "parallel_suggestions": parallel_suggestions,
            "batch_suggestions": batch_suggestions,
            "fail_fast_suggestions": fail_fast_suggestions,
            "before_after_comparison": {
                "before": {
                    "execution_mode": "sequential",
                    "total_duration_ms": sequential_time,
                    "db_round_trips": len(db_write_steps),
                    "lock_contention_risk": "high" if lock_steps else "none"
                },
                "after": {
                    "execution_mode": "parallel_with_batch_and_fail_fast",
                    "total_duration_ms": optimized_time,
                    "db_round_trips": max(1, len(db_write_steps) - len(batch_suggestions)),
                    "lock_contention_risk": "reduced"
                }
            }
        }

    def generate_transaction_dashboard(self) -> Dict:
        """生成事务仪表盘：实时TPS、延迟热力图、失败率趋势、慢事务排名、资源利用率关联"""
        current_timestamp = time.time()
        services = self._get_all_services()
        tps_per_service = {}
        latency_heatmap: Dict[str, Dict[str, float]] = {}
        failure_rate_trend: Dict[str, List[Dict]] = {}
        top_slow_transactions = []
        resource_alerts = []

        for service_name in services:
            with self.circular_buffer_lock:
                buf = self.circular_buffers.get(service_name, deque())
                recent_entries = [e for e in buf if e["timestamp"] >= current_timestamp - self.tps_window_seconds]
            current_tps = len(recent_entries) / self.tps_window_seconds if self.tps_window_seconds > 0 else 0.0
            tps_per_service[service_name] = round(current_tps, 2)

            redis_key = f"tx:{service_name}:latency"
            step_durations: Dict[str, List[float]] = defaultdict(list)
            recent_entries_all = self.redis.zrangebyscore(redis_key, current_timestamp - 3600, current_timestamp)
            for entry_bytes in recent_entries_all:
                entry = json.loads(entry_bytes)
                step_name = entry.get("step_name", "unknown")
                step_durations[step_name].append(entry.get("duration_ms", 0))

            service_heatmap = {}
            for step_name, durations in step_durations.items():
                durations.sort()
                if durations:
                    p99_idx = max(0, int(len(durations) * 0.99) - 1)
                    service_heatmap[step_name] = round(durations[p99_idx], 2)
            latency_heatmap[service_name] = service_heatmap

            hourly_failure_rates = []
            for hour_offset in range(24):
                hour_start = current_timestamp - (hour_offset + 1) * 3600
                hour_end = current_timestamp - hour_offset * 3600
                hour_entries = self.redis.zrangebyscore(redis_key, hour_start, hour_end)
                total_count = len(hour_entries)
                failure_count = 0
                for entry_bytes in hour_entries:
                    entry = json.loads(entry_bytes)
                    if not entry.get("success", True):
                        failure_count += 1
                failure_rate = (failure_count / total_count * 100) if total_count > 0 else 0.0
                hour_label = datetime.fromtimestamp(hour_start).strftime("%Y-%m-%d %H:00")
                hourly_failure_rates.append({
                    "hour": hour_label,
                    "failure_rate_pct": round(failure_rate, 2),
                    "total_transactions": total_count,
                    "failed_transactions": failure_count
                })
            failure_rate_trend[service_name] = list(reversed(hourly_failure_rates))

        all_transactions = []
        for service_name in services:
            redis_key = f"tx:{service_name}:latency"
            recent = self.redis.zrangebyscore(redis_key, current_timestamp - 3600, current_timestamp)
            tx_steps: Dict[str, List[Dict]] = defaultdict(list)
            for entry_bytes in recent:
                entry = json.loads(entry_bytes)
                tx_id = entry.get("transaction_id", "unknown")
                tx_steps[tx_id].append({
                    "step_name": entry.get("step_name", "unknown"),
                    "duration_ms": entry.get("duration_ms", 0),
                    "success": entry.get("success", True),
                    "timestamp": entry.get("timestamp", 0)
                })

            for tx_id, steps in tx_steps.items():
                total_duration = sum(s["duration_ms"] for s in steps)
                all_steps_succeeded = all(s["success"] for s in steps)
                all_transactions.append({
                    "transaction_id": tx_id,
                    "service_name": service_name,
                    "total_duration_ms": total_duration,
                    "all_steps_succeeded": all_steps_succeeded,
                    "step_count": len(steps),
                    "steps": sorted(steps, key=lambda s: s["timestamp"])
                })

        all_transactions.sort(key=lambda t: t["total_duration_ms"], reverse=True)
        top_slow_transactions = all_transactions[:10]

        cpu_usage = self._get_cpu_usage()
        memory_usage = self._get_memory_usage()
        resource_correlations = []

        if cpu_usage > 80:
            resource_correlations.append({
                "resource": "cpu",
                "usage_pct": cpu_usage,
                "flag": "potential_cause",
                "detail": f"CPU usage at {cpu_usage}% may be causing transaction latency degradation"
            })
            for service_name in services:
                high_latency_steps = []
                for step, p99 in latency_heatmap.get(service_name, {}).items():
                    if p99 > self.sla_threshold_ms:
                        high_latency_steps.append(step)
                if high_latency_steps:
                    resource_alerts.append({
                        "service_name": service_name,
                        "resource": "cpu",
                        "usage_pct": cpu_usage,
                        "affected_steps": high_latency_steps,
                        "recommendation": "Consider scaling horizontally or optimizing CPU-intensive operations"
                    })

        if memory_usage > 85:
            resource_correlations.append({
                "resource": "memory",
                "usage_pct": memory_usage,
                "flag": "potential_cause",
                "detail": f"Memory usage at {memory_usage}% may cause GC pressure affecting transaction latency"
            })

        active_alerts = self.redis.lrange("tx:alert_queue", 0, 49)
        recent_alerts = [json.loads(a) for a in active_alerts]

        return {
            "generated_at": datetime.fromtimestamp(current_timestamp).isoformat(),
            "tps_per_service": tps_per_service,
            "latency_heatmap": latency_heatmap,
            "failure_rate_trend": failure_rate_trend,
            "top_10_slow_transactions": top_slow_transactions,
            "resource_utilization": {
                "cpu_pct": cpu_usage,
                "memory_pct": memory_usage,
                "correlations": resource_correlations
            },
            "resource_alerts": resource_alerts,
            "recent_spike_alerts": recent_alerts
        }

    def _get_all_services(self) -> List[str]:
        """从Redis获取所有已知服务名称"""
        services = set()
        cursor = 0
        while True:
            cursor, keys = self.redis.scan(cursor, match="tx:*:latency", count=100)
            for key in keys:
                key_str = key.decode("utf-8") if isinstance(key, bytes) else key
                parts = key_str.split(":")
                if len(parts) >= 3:
                    services.add(parts[1])
            if cursor == 0:
                break
        return list(services)

    def _get_cpu_usage(self) -> float:
        """获取当前CPU使用率"""
        try:
            with open("/proc/stat", "r") as f:
                line1 = f.readline()
            time.sleep(0.1)
            with open("/proc/stat", "r") as f:
                line2 = f.readline()
            values1 = list(map(int, line1.split()[1:]))
            values2 = list(map(int, line2.split()[1:]))
            d_idle = values2[3] - values1[3]
            d_total = sum(values2) - sum(values1)
            if d_total > 0:
                return round((1 - d_idle / d_total) * 100, 2)
            return 0.0
        except Exception:
            return 0.0

    def _get_memory_usage(self) -> float:
        """获取当前内存使用率"""
        try:
            with open("/proc/meminfo", "r") as f:
                lines = f.readlines()
            mem_info = {}
            for line in lines:
                parts = line.split()
                key = parts[0].rstrip(":")
                value = int(parts[1])
                mem_info[key] = value
            total = mem_info.get("MemTotal", 1)
            available = mem_info.get("MemAvailable", 0)
            used = total - available
            return round(used / total * 100, 2)
        except Exception:
            return 0.0
```

## 异常场景补充

### 场景：事务超时导致雪崩效应
```
trigger: 单个关键事务步骤超时（如数据库锁等待超过30秒），导致依赖该步骤的上游服务线程池耗尽，进而引发级联超时，最终整个事务链路崩溃
detection: (1) 监控单个事务步骤的p99延迟突增超过基线3倍；(2) 线程池活跃线程数持续超过配置阈值的90%；(3) 下游服务请求排队深度持续增长；(4) TPS从正常值骤降到接近0；(5) 连续5个以上关联服务同时报告超时告警
handling: (1) 立即触发断路器，对该超时步骤的服务调用熔断，返回降级响应而非等待；(2) 对排队中的事务请求执行快速失败（fail-fast），释放等待线程；(3) 扩大超时步骤的线程池容量，临时增加50%的并发处理能力；(4) 对非关键事务步骤启用跳过策略（如审计日志写入可异步化）；(5) 逐步恢复：先以10%流量试探性放行，确认正常后按25%、50%、100%梯度恢复
prevention: (1) 为每个事务步骤设置合理的超时上限（根据p99的3倍设定），严禁无超时配置；(2) 实现舱壁模式（Bulkhead），每个步骤的线程池相互隔离，防止资源争抢；(3) 配置断路器，当错误率超过50%时自动熔断；(4) 实现事务降级策略，关键路径保留、非关键路径可跳过；(5) 定期进行混沌工程测试，模拟单个步骤超时验证系统韧性
```

### 场景：监控数据延迟掩盖真实问题
```
trigger: Redis集群发生网络分区或主从切换，导致监控指标写入延迟超过5分钟，仪表盘显示的TPS和延迟数据已过时，运维人员基于过时数据做出错误判断（如认为系统正常而未采取行动）
detection: (1) 监控数据的时间戳与当前时间差超过阈值（如>60秒）；(2) 仪表盘数据刷新间隔异常延长（从秒级变为分钟级）；(3) Redis写入操作的latency突然升高（写入p99 > 100ms）；(4) 监控系统自身的健康检查探针报告数据管道延迟；(5) 对比多个独立数据源（如应用日志与Redis指标），发现时间偏差
handling: (1) 立即在仪表盘上标注"数据可能延迟"警告标志；(2) 切换到备用监控数据源（如直接查询数据库中的事务记录作为实时指标）；(3) 对Redis集群执行强制主从切换，恢复写入能力；(4) 在监控延迟期间，使用应用层日志作为临时指标来源，通过日志分析工具实时聚合；(5) 延迟恢复后，回填缺失的监控数据点并重新计算基线
prevention: (1) 部署双活监控数据管道，Redis主集群和备用集群同时写入，任一可用即可读；(2) 在监控仪表盘中增加数据新鲜度指标，实时显示最后数据更新时间；(3) 设置监控数据延迟告警，当数据延迟超过30秒时触发；(4) 实现降级采集模式，当Redis不可用时自动切换到本地内存缓冲+定期批量写入；(5) 定期演练监控数据管道故障场景，确保降级方案有效
```

### 场景：事务重试导致重复执行
```
trigger: 分布式事务协调者在发送commit指令后网络超时，参与者已成功提交但协调者未收到确认，协调者触发重试，导致参与者重复执行已提交的操作（如重复扣款、重复库存扣减）
detection: (1) 事务日志中出现相同transaction_id的多次commit记录；(2) 业务数据出现异常（如账户余额变动两次、库存数量低于预期）；(3) 参与者端收到重复的commit请求（幂等键重复）；(4) 对账系统发现金额不匹配；(5) 用户投诉重复扣款
handling: (1) 立即暂停受影响事务类型的重试机制，改为手动确认模式；(2) 查询事务日志，按transaction_id聚合所有执行记录，标记重复执行的事务；(3) 对重复执行的事务触发补偿操作（反向交易），将多扣金额退回、多扣库存加回；(4) 在补偿完成后，更新事务状态为"已补偿"，避免重复补偿；(5) 通知受影响用户并提供对账明细
prevention: (1) 所有事务参与者必须实现幂等性：每个操作基于transaction_id+step_id生成唯一幂等键，执行前先查询是否已执行过；(2) 协调者重试前必须查询参与者状态（询问是否已提交），而非盲目重发commit；(3) 使用TCC模式替代简单的重试：Try阶段预留资源、Confirm阶段确认、Cancel阶段回滚，每个阶段天然幂等；(4) 实现全局事务状态表，协调者和参与者共享事务执行状态，避免信息不对称；(5) 设置重试上限（如最多3次），超过后转人工处理，并增加重试间隔的指数退避策略
```
