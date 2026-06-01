# C16: 跨境支付的多币种结算

## 业务场景

某跨境电商平台，用户在中国用人民币购买海外商品，商家在日本/美国/欧洲收款。资金流跨越多个国家的银行、支付渠道和监管体系。

**已知数据：**
- 日均交易笔数：100 万笔
- 日均交易额：$2000 万
- 涉及币种：CNY, USD, EUR, JPY, GBP, AUD 等 20+
- 涉及支付渠道：支付宝、微信、银联、PayPal、Stripe、本地银行转账
- 汇率更新频率：每 15 秒
- 结算周期：T+2（交易后 2 个工作日商家收到款）

**为什么这是专家级难题？**

跨境支付不是简单的"汇率换算"，而是一个涉及多个独立系统的分布式事务，每个系统都有自己的合规要求、结算周期和故障模式：

- **汇率波动**：用户下单时看到的汇率和实际结算时的汇率可能不同，差价谁来承担？
- **合规审查**：每笔跨境交易需要经过反洗钱（AML）筛查，筛查可能耗时数分钟
- **对账闭环**：内部账、渠道账、银行账三方必须一致，任何一分钱的差异都需解释
- **资金安全**：跨境转账一旦发出不可撤回，任何错误都是真金白银的损失

## 核心挑战

### 挑战 1：汇率敞口风险

用户下单时 CNY/USD = 7.20，2 天后结算时 CNY/USD = 7.25。如果按下单时汇率结算，平台承担 0.7% 的汇率差；如果按结算时汇率，用户可能多付钱。

### 挑战 2：多系统协调的分布式事务

一笔跨境支付涉及：
1. 用户支付（支付宝/微信）→ 人民币扣款
2. 汇率锁定 → 确定兑换金额
3. AML 合规筛查 → 可能拒绝
4. 跨境购汇 → 人民币换美元
5. 跨境汇款 → 美元汇到商家海外账户
6. 商家结算 → 美元入账

6 步操作，任何一步都可能失败，且部分失败不可简单回滚（跨境汇款发出后无法撤回）。

### 挑战 3：合规审查

- 单笔 > $5000 的交易需人工审核
- 某些商品（军工、药品）禁止跨境销售
- 用户在制裁名单（OFAC）上 → 整笔交易冻结
- 每个国家的个人年度购汇额度不同（中国 $50000/年）

### 挑战 4：三方对账

内部订单系统、支付渠道（支付宝）、结算银行三方的记录必须一致。但三方的结算周期不同：内部实时、渠道 T+1、银行 T+2。

## 设计约束

- 汇率锁定时效：下单后 30 分钟内有效
- 合规筛查延迟：< 5 分钟（自动），< 2 小时（人工）
- 資金安全：任何情况下不能多付或少付
- 幂等性：同一笔交易不能重复执行

## 数据库设计

跨境支付涉及多个业务实体，每个表的字段必须精确到足以支持对账、合规审计和故障恢复。

### 核心表结构

```sql
-- 支付订单表：跨境支付的主表，记录完整的支付生命周期
CREATE TABLE payment_orders (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id            VARCHAR(64) NOT NULL UNIQUE COMMENT '业务订单号，幂等键',
    user_id             VARCHAR(64) NOT NULL,
    merchant_id         VARCHAR(64) NOT NULL,
    product_category    VARCHAR(32) NOT NULL COMMENT '商品品类，用于合规检查',

    -- 金额与币种
    amount_cny          DECIMAL(18,2) NOT NULL COMMENT '用户支付金额（人民币）',
    source_currency     VARCHAR(3) NOT NULL DEFAULT 'CNY',
    target_currency     VARCHAR(3) NOT NULL COMMENT '商家结算币种',
    target_amount       DECIMAL(18,2) NULL COMMENT '实际兑换后金额',
    locked_rate         DECIMAL(12,6) NULL COMMENT '锁定汇率',
    rate_lock_id        VARCHAR(64) NULL COMMENT '关联汇率锁定ID',

    -- 支付渠道
    payment_method      VARCHAR(32) NOT NULL COMMENT 'alipay/wechat/unionpay',
    channel_tx_id       VARCHAR(128) NULL COMMENT '渠道侧交易号',
    channel_callback_id VARCHAR(128) NULL COMMENT '渠道回调ID',

    -- 银行信息
    bank_tx_id          VARCHAR(128) NULL COMMENT '银行侧交易号',
    swift_message_id    VARCHAR(64) NULL COMMENT 'SWIFT报文参考号',
    merchant_bank_account VARCHAR(64) NOT NULL COMMENT '商家银行账户',

    -- Saga 状态机
    saga_state          VARCHAR(32) NOT NULL DEFAULT 'INIT' COMMENT 'INIT/PAYMENT_DONE/AML_PASSED/FOREX_DONE/TRANSFER_DONE/SETTLED/COMPENSATING/COMPENSATED/FAILED',
    saga_step           INT NOT NULL DEFAULT 0 COMMENT '当前执行到第几步',
    saga_id             VARCHAR(64) NULL COMMENT 'Saga实例ID',

    -- 状态与时间
    status              VARCHAR(32) NOT NULL DEFAULT 'created' COMMENT 'created/processing/completed/failed/refunded',
    aml_status          VARCHAR(16) NULL COMMENT 'pending/approved/rejected/manual_review',
    aml_review_id       VARCHAR(64) NULL COMMENT '人工审核ID',
    settlement_date     DATE NULL COMMENT '商家结算日期（T+2）',

    -- 审计字段
    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    completed_at        DATETIME NULL,
    idempotency_key     VARCHAR(128) NULL COMMENT '外部幂等键',

    INDEX idx_user_id (user_id),
    INDEX idx_merchant_id (merchant_id),
    INDEX idx_saga_state (saga_state),
    INDEX idx_status_created (status, created_at),
    INDEX idx_settlement_date (settlement_date),
    INDEX idx_channel_tx_id (channel_tx_id),
    INDEX idx_bank_tx_id (bank_tx_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='跨境支付订单表';

-- 汇率锁定表：记录每笔汇率锁定的完整生命周期
CREATE TABLE forex_locks (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    lock_id             VARCHAR(64) NOT NULL UNIQUE COMMENT '锁定ID',
    order_id            VARCHAR(64) NOT NULL COMMENT '关联订单号',
    from_currency       VARCHAR(3) NOT NULL,
    to_currency         VARCHAR(3) NOT NULL,
    rate                DECIMAL(12,6) NOT NULL COMMENT '锁定汇率',
    amount              DECIMAL(18,2) NOT NULL COMMENT '源币种金额',
    converted_amount    DECIMAL(18,2) NOT NULL COMMENT '目标币种金额',
    status              VARCHAR(16) NOT NULL DEFAULT 'locked' COMMENT 'locked/executed/expired/reversed',
    locked_at           DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    expires_at          DATETIME NOT NULL COMMENT '锁定过期时间',
    executed_at         DATETIME NULL COMMENT '实际执行购汇时间',
    actual_rate         DECIMAL(12,6) NULL COMMENT '实际执行汇率（可能与锁定汇率不同）',
    rate_diff_bps       INT NULL COMMENT '汇率差异基点（1bp = 0.01%）',
    provider            VARCHAR(32) NOT NULL COMMENT 'forex_provider名称',
    provider_tx_id      VARCHAR(128) NULL COMMENT '供应商交易号',
    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_order_id (order_id),
    INDEX idx_status_expires (status, expires_at),
    INDEX idx_provider_tx_id (provider_tx_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='汇率锁定表';

-- AML筛查记录表：每次筛查的完整检查项和结果
CREATE TABLE aml_screening (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    screening_id        VARCHAR(64) NOT NULL UNIQUE COMMENT '筛查ID',
    order_id            VARCHAR(64) NOT NULL,
    user_id             VARCHAR(64) NOT NULL,
    user_name           VARCHAR(128) NOT NULL,

    -- 检查项与结果
    amount_check        VARCHAR(16) NOT NULL COMMENT 'pass/enhanced/manual',
    sanctions_check     VARCHAR(16) NOT NULL COMMENT 'pass/match/false_positive',
    sanctions_lists     VARCHAR(256) NULL COMMENT '匹配的制裁名单，如OFAC/EU/UN',
    quota_check         VARCHAR(16) NOT NULL COMMENT 'pass/exceeded',
    yearly_total_usd    DECIMAL(18,2) NULL COMMENT '用户本年已用购汇额度',
    product_check       VARCHAR(16) NOT NULL COMMENT 'pass/restricted',
    frequency_check     VARCHAR(16) NOT NULL COMMENT 'pass/high_frequency',

    -- 综合结果
    status              VARCHAR(16) NOT NULL DEFAULT 'pending' COMMENT 'pending/approved/rejected/manual_review',
    review_id           VARCHAR(64) NULL COMMENT '人工审核ID',
    reviewer_id         VARCHAR(64) NULL COMMENT '审核人ID',
    review_result       VARCHAR(16) NULL COMMENT 'approved/rejected',
    review_comment      TEXT NULL COMMENT '审核备注',
    reviewed_at         DATETIME NULL,

    -- 性能
    screening_duration_ms INT NULL COMMENT '筛查耗时（毫秒）',
    provider            VARCHAR(32) NOT NULL COMMENT 'AML筛查供应商',

    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_order_id (order_id),
    INDEX idx_user_id (user_id),
    INDEX idx_status (status),
    INDEX idx_review_id (review_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='AML合规筛查表';

-- 结算记录表：商家结算的完整信息
CREATE TABLE settlement_records (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    settlement_id       VARCHAR(64) NOT NULL UNIQUE COMMENT '结算ID',
    order_id            VARCHAR(64) NOT NULL,
    merchant_id         VARCHAR(64) NOT NULL,

    -- 结算金额
    settlement_currency VARCHAR(3) NOT NULL,
    settlement_amount   DECIMAL(18,2) NOT NULL,
    platform_fee        DECIMAL(18,2) NOT NULL COMMENT '平台手续费',
    channel_fee         DECIMAL(18,2) NOT NULL COMMENT '渠道手续费',
    forex_spread_fee    DECIMAL(18,2) NOT NULL DEFAULT 0 COMMENT '汇兑差价收入',
    net_amount          DECIMAL(18,2) NOT NULL COMMENT '商家实际到账金额',

    -- 结算状态
    status              VARCHAR(16) NOT NULL DEFAULT 'scheduled' COMMENT 'scheduled/processing/completed/failed/returned',
    scheduled_date      DATE NOT NULL COMMENT '计划结算日期',
    completed_date      DATE NULL COMMENT '实际结算日期',

    -- 银行信息
    bank_name           VARCHAR(64) NOT NULL,
    bank_account        VARCHAR(64) NOT NULL,
    bank_country        VARCHAR(3) NOT NULL,
    swift_code          VARCHAR(16) NULL COMMENT '银行SWIFT代码',
    bank_reference      VARCHAR(128) NULL COMMENT '银行参考号',

    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_order_id (order_id),
    INDEX idx_merchant_id (merchant_id),
    INDEX idx_status_scheduled (status, scheduled_date),
    INDEX idx_bank_reference (bank_reference)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商家结算记录表';

-- 对账差异表：记录所有对账差异及处理过程
CREATE TABLE reconciliation_diffs (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    batch_id            VARCHAR(64) NOT NULL COMMENT '对账批次ID',
    reconcile_date      DATE NOT NULL COMMENT '对账日期',

    -- 差异定位
    source_pair         VARCHAR(32) NOT NULL COMMENT 'internal_vs_channel/internal_vs_bank/channel_vs_bank',
    diff_type           VARCHAR(32) NOT NULL COMMENT 'missing_in_a/missing_in_b/amount_mismatch/duplicate/rate_mismatch',

    -- 交易信息
    internal_tx_id      VARCHAR(128) NULL,
    channel_tx_id       VARCHAR(128) NULL,
    bank_tx_id          VARCHAR(128) NULL,

    -- 金额
    internal_amount     DECIMAL(18,2) NULL,
    external_amount     DECIMAL(18,2) NULL,
    diff_amount         DECIMAL(18,2) NULL COMMENT '差异金额',

    -- 处理状态
    status              VARCHAR(16) NOT NULL DEFAULT 'open' COMMENT 'open/auto_resolved/manual_resolved/escalated/closed',
    resolution          VARCHAR(32) NULL COMMENT 'timing_diff/rate_diff/system_error/fraud/other',
    resolution_detail   TEXT NULL COMMENT '处理说明',
    resolved_by         VARCHAR(64) NULL COMMENT '处理人',
    resolved_at         DATETIME NULL,
    auto_resolve_rule   VARCHAR(64) NULL COMMENT '自动处理的规则ID',

    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_batch_id (batch_id),
    INDEX idx_reconcile_date (reconcile_date),
    INDEX idx_status (status),
    INDEX idx_source_pair (source_pair),
    INDEX idx_internal_tx_id (internal_tx_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='对账差异表';

-- 对账批次表：记录每次对账的执行情况
CREATE TABLE reconciliation_batches (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    batch_id            VARCHAR(64) NOT NULL UNIQUE,
    reconcile_date      DATE NOT NULL,
    source_pair         VARCHAR(32) NOT NULL,
    internal_count      INT NOT NULL DEFAULT 0,
    external_count      INT NOT NULL DEFAULT 0,
    matched_count       INT NOT NULL DEFAULT 0,
    diff_count          INT NOT NULL DEFAULT 0,
    auto_resolved_count INT NOT NULL DEFAULT 0,
    manual_count        INT NOT NULL DEFAULT 0,
    status              VARCHAR(16) NOT NULL DEFAULT 'running' COMMENT 'running/completed/failed',
    started_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    completed_at        DATETIME NULL,
    error_message       TEXT NULL,

    INDEX idx_reconcile_date (reconcile_date),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='对账批次表';

-- Saga 实例表：持久化 Saga 状态，支持崩溃恢复和超时管理
CREATE TABLE saga_instances (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    saga_id             VARCHAR(64) NOT NULL UNIQUE COMMENT 'Saga实例唯一ID',
    order_id            VARCHAR(64) NOT NULL COMMENT '关联业务订单号',
    saga_type           VARCHAR(64) NOT NULL DEFAULT 'cross_border_payment' COMMENT 'Saga类型',
    current_step        INT NOT NULL DEFAULT 0 COMMENT '当前已完成的步骤序号（0=尚未开始任何步骤）',
    total_steps         INT NOT NULL DEFAULT 5 COMMENT '总步骤数',
    state               VARCHAR(32) NOT NULL DEFAULT 'PENDING' COMMENT 'PENDING/RUNNING/SUSPENDED/COMPENSATING/COMPENSATED/COMPLETED/FAILED/TIMED_OUT',
    direction           VARCHAR(16) NOT NULL DEFAULT 'forward' COMMENT 'forward/compensate',
    suspend_reason      VARCHAR(64) NULL COMMENT '暂停原因，如 manual_review',
    suspend_ref_id      VARCHAR(64) NULL COMMENT '暂停关联ID，如审核ID',
    timeout_at          DATETIME NULL COMMENT '当前步骤的超时时间',
    retry_count         INT NOT NULL DEFAULT 0 COMMENT '当前步骤重试次数',
    max_retries         INT NOT NULL DEFAULT 3 COMMENT '最大重试次数',
    context_data        JSON NULL COMMENT 'Saga上下文数据（各步骤的中间结果）',
    started_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    completed_at        DATETIME NULL,

    INDEX idx_order_id (order_id),
    INDEX idx_state (state),
    INDEX idx_timeout_at (timeout_at),
    INDEX idx_saga_type_state (saga_type, state)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='Saga实例表-持久化状态用于崩溃恢复';

-- Saga 状态机日志表：记录 Saga 每一步的执行和补偿
CREATE TABLE saga_state_log (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    saga_id             VARCHAR(64) NOT NULL,
    order_id            VARCHAR(64) NOT NULL,
    step_name           VARCHAR(64) NOT NULL COMMENT '步骤名称',
    step_index          INT NOT NULL COMMENT '步骤序号',
    direction           VARCHAR(16) NOT NULL COMMENT 'forward/compensate',
    status              VARCHAR(16) NOT NULL COMMENT 'started/completed/failed/skipped',
    input_data          JSON NULL COMMENT '步骤输入',
    output_data         JSON NULL COMMENT '步骤输出',
    error_message       TEXT NULL,
    retry_count         INT NOT NULL DEFAULT 0,
    idempotency_key     VARCHAR(128) NOT NULL COMMENT '步骤级幂等键',
    started_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    completed_at        DATETIME NULL,
    duration_ms         INT NULL COMMENT '执行耗时',

    INDEX idx_saga_id (saga_id),
    INDEX idx_order_id (order_id),
    INDEX idx_idempotency_key (idempotency_key),
    INDEX idx_step_status (step_name, status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='Saga状态机日志表';
```

### 表间关系说明

```
payment_orders (1) ──→ (1) forex_locks        : 一个订单对应一次汇率锁定
payment_orders (1) ──→ (1) aml_screening      : 一个订单对应一次AML筛查
payment_orders (1) ──→ (1) settlement_records  : 一个订单对应一条结算记录
payment_orders (1) ──→ (1) saga_instances       : 一个订单对应一个Saga实例
payment_orders (1) ──→ (N) saga_state_log      : 一个订单对应多条Saga日志
saga_instances (1) ──→ (N) saga_state_log      : 一个Saga实例对应多条步骤日志
reconciliation_batches (1) ──→ (N) reconciliation_diffs : 一个批次包含多条差异
```

关键设计决策：
- **金额字段使用 DECIMAL(18,2)**：绝对不用浮点数，避免精度丢失导致对账不平。汇率字段用 DECIMAL(12,6) 保留 6 位小数，因为日元等小单位货币需要更高精度。
- **saga_state 与 status 分离**：saga_state 驱动流程推进，status 表示业务状态，两者独立演变。例如 saga_state=COMPENSATING 时 status 仍可能是 processing。
- **所有外部交互字段均独立**：channel_tx_id、bank_tx_id、swift_message_id 分开存储，因为三方对账需要精确匹配。
- **saga_state_log 表**：记录每一步的完整输入输出和幂等键，这是故障恢复和审计的核心依据。

## 请先独立思考（限时 45 分钟）

1. 汇率敞口如何管理？锁定汇率 vs 实时汇率，各自的风险和成本？
2. 画出跨境支付的完整 Saga 流程，标注每一步的补偿操作。特别注意：跨境汇款发出后无法撤回，如何补偿？
3. AML 筛查如何与支付流程集成？筛查期间资金应该处于什么状态？
4. 三方对账的差异类型有哪些？如何自动发现和处理？

---

## 设计解析

### 汇率管理：锁定汇率 + 平台承担小额敞口

**方案：下单时锁定汇率，平台承担 30 分钟内的汇率波动**

```python
class ExchangeRateService:
    def lock_rate(self, from_currency, to_currency, amount):
        """锁定汇率，有效期 30 分钟"""
        current_rate = self.get_realtime_rate(from_currency, to_currency)
        
        lock = {
            "lock_id": uuid4(),
            "from_currency": from_currency,
            "to_currency": to_currency,
            "rate": current_rate,
            "amount": amount,
            "converted_amount": amount * current_rate,
            "locked_at": now(),
            "expires_at": now() + timedelta(minutes=30),
            "status": "locked"
        }
        
        self.redis.setex(
            f"rate_lock:{lock['lock_id']}", 1800,
            json.dumps(lock)
        )
        
        return lock

    def execute_conversion(self, lock_id):
        """执行汇率兑换（在锁定期内）"""
        lock = self.get_lock(lock_id)
        
        if now() > lock.expires_at:
            # 锁过期，用当前汇率重新计算
            current_rate = self.get_realtime_rate(lock.from_currency, lock.to_currency)
            diff = abs(current_rate - lock.rate) / lock.rate
            
            if diff > 0.02:  # 汇率波动 > 2%
                # 波动太大，需要用户确认新汇率
                raise RateExpiredError(current_rate, lock.rate)
            
            # 波动 < 2%，平台承担差价
            lock.rate = current_rate
            lock.converted_amount = lock.amount * current_rate
        
        # 执行购汇
        result = self.forex_provider.convert(
            lock.from_currency, lock.to_currency,
            lock.amount, lock.rate
        )
        
        return result
```

**汇率敞口的成本估算：**

主要币对的日波动率约 0.5-1%。平台承担 30 分钟内的波动：
- 30 分钟波动约 0.01-0.05%
- 日均交易额 $2000 万 × 0.03% = $6000/天
- 月成本约 $18 万

这是可控的。如果波动超过 2%（极端行情），要求用户确认新汇率。

### 跨境支付 Saga 流程

```
Step 1: 用户支付（支付宝/微信扣款）→ 补偿: 退款
Step 2: AML 合规筛查                → 补偿: 退款（筛查拒绝）
Step 3: 汇率锁定 + 购汇             → 补偿: 退汇（将外币换回人民币）
Step 4: 跨境汇款到商家              → 补偿: 无法撤回！→ 通知商家退回
Step 5: 商家结算确认                → 补偿: 商家退款
```

**Step 4 的补偿是关键难点：跨境汇款一旦发出无法撤回。**

```python
class CrossBorderPaymentSaga:
    STEPS = [
        {"name": "user_payment",   "compensate": "refund_user"},
        {"name": "aml_screening",  "compensate": "refund_user"},
        {"name": "forex_convert",  "compensate": "forex_reverse"},
        {"name": "cross_border_transfer", "compensate": "request_merchant_return"},
        {"name": "merchant_settle", "compensate": "merchant_refund"},
    ]

    def execute(self, order):
        saga_id = uuid4()
        
        # Step 1: 用户支付
        payment_result = self.payment_service.charge(
            order.user_id, order.amount_cny, order.payment_method
        )
        if not payment_result.success:
            return PaymentFailed(payment_result.error)
        
        # Step 2: AML 筛查（可能耗时数分钟）
        aml_result = self.aml_service.screen(order)
        if aml_result.status == "rejected":
            self.payment_service.refund(payment_result.payment_id)
            return AMLRejected(aml_result.reason)
        elif aml_result.status == "manual_review":
            # 进入人工审核，暂停 Saga
            self.suspend_for_review(saga_id, aml_result.review_id)
            return PendingReview(aml_result.review_id)
        
        # Step 3: 汇率锁定 + 购汇
        rate_lock = self.rate_service.lock_rate("CNY", order.merchant_currency, order.amount_cny)
        forex_result = self.forex_service.convert(rate_lock)
        if not forex_result.success:
            self.payment_service.refund(payment_result.payment_id)
            return ForexFailed(forex_result.error)
        
        # Step 4: 跨境汇款（不可撤回！）
        transfer_result = self.transfer_service.send(
            forex_result.converted_amount,
            order.merchant_currency,
            order.merchant_bank_account
        )
        if not transfer_result.success:
            # 汇款失败 → 退汇 + 退款
            self.forex_service.reverse(forex_result.conversion_id)
            self.payment_service.refund(payment_result.payment_id)
            return TransferFailed(transfer_result.error)
        
        # Step 5: 商家结算（异步，T+2）
        self.settlement_service.schedule(
            order.merchant_id, 
            forex_result.converted_amount,
            order.merchant_currency,
            settlement_date=now() + timedelta(days=2)
        )
        
        return PaymentSuccess(
            amount_cny=order.amount_cny,
            amount_foreign=forex_result.converted_amount,
            rate=rate_lock.rate
        )
```

**Step 4 汇款成功但后续失败的补偿：**

```
跨境汇款已发出 → 商家银行已收到 → 无法撤回

补偿方案：
1. 通知商家"该笔汇款需要退回"（通过商家后台 + 邮件）
2. 商家通过银行将款项退回（SWIFT 退款，耗时 3-5 个工作日）
3. 收到退款后，执行退汇 + 退款给用户

在此期间：
- 用户已扣款但订单未完成 → 通知用户"处理中"
- 商家已收款但需要退回 → 通知商家"请退回款项"
```

### AML 合规筛查

**筛查流程：**

```python
class AMLScreeningService:
    def screen(self, order):
        checks = []
        
        # 1. 金额检查
        if order.amount_usd > 50000:
            checks.append({"rule": "high_value", "action": "manual_review"})
        elif order.amount_usd > 5000:
            checks.append({"rule": "medium_value", "action": "enhanced_screening"})
        
        # 2. 制裁名单检查（OFAC/EU/UN）
        if self.sanctions_list.check(order.user_name, order.user_id):
            return AMLResult(status="rejected", reason="sanctions_list_match")
        
        # 3. 用户年度购汇额度检查
        yearly_total = self.get_yearly_forex_total(order.user_id)
        if yearly_total + order.amount_usd > 50000:  # 中国个人年度额度 $50000
            return AMLResult(status="rejected", reason="forex_quota_exceeded")
        
        # 4. 商品合规检查
        if self.is_restricted_product(order.product_category):
            return AMLResult(status="rejected", reason="restricted_product")
        
        # 5. 频率异常检查（短时多笔跨境交易）
        recent_count = self.get_recent_cross_border_count(order.user_id, hours=24)
        if recent_count > 10:
            checks.append({"rule": "high_frequency", "action": "manual_review"})
        
        # 决定
        if any(c["action"] == "manual_review" for c in checks):
            review_id = self.create_manual_review(order, checks)
            return AMLResult(status="manual_review", review_id=review_id)
        
        return AMLResult(status="approved")
```

**筛查期间资金状态：**

```
用户支付 → 资金在平台账户（已扣款但未结算）
         → AML 筛查中 → 资金冻结
         → 筛查通过 → 继续购汇+汇款
         → 筛查拒绝 → 退款给用户
         → 人工审核中 → 资金保持冻结，等待审核结果
```

### 年度购汇额度追踪

**中国外汇管理局规定个人年度购汇额度为等值 $50000。平台必须在每笔跨境交易前校验用户剩余额度，并在交易完成后更新已用额度。额度超用将面临外管局处罚，严重者可被暂停跨境业务资格。**

#### 数据库设计

```sql
-- 用户年度购汇额度表：按用户+年份追踪已用额度
CREATE TABLE forex_quota_usage (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id             VARCHAR(64) NOT NULL,
    quota_year          SMALLINT NOT NULL COMMENT '额度年份，如 2024',
    currency            VARCHAR(3) NOT NULL DEFAULT 'USD' COMMENT '额度基准币种',
    used_amount         DECIMAL(18,2) NOT NULL DEFAULT 0 COMMENT '已用额度（等值美元）',
    reserved_amount     DECIMAL(18,2) NOT NULL DEFAULT 0 COMMENT '冻结中额度（AML审核中的交易）',
    quota_limit         DECIMAL(18,2) NOT NULL DEFAULT 50000 COMMENT '年度额度上限',
    remaining_amount    DECIMAL(18,2) GENERATED ALWAYS AS (
                            quota_limit - used_amount - reserved_amount
                        ) STORED COMMENT '剩余可用额度',
    last_updated_order  VARCHAR(64) NULL COMMENT '最近一次更新的订单号',
    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    UNIQUE KEY uk_user_year (user_id, quota_year),
    INDEX idx_remaining (remaining_amount)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户年度购汇额度表';

-- 购汇额度明细表：每笔交易的额度占用/释放记录
CREATE TABLE forex_quota_detail (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id             VARCHAR(64) NOT NULL,
    order_id            VARCHAR(64) NOT NULL,
    quota_year          SMALLINT NOT NULL,
    action              VARCHAR(16) NOT NULL COMMENT 'reserve/consume/release/refund',
    amount_usd          DECIMAL(18,2) NOT NULL COMMENT '额度变动金额（等值美元）',
    original_currency   VARCHAR(3) NOT NULL COMMENT '原始交易币种',
    original_amount     DECIMAL(18,2) NOT NULL COMMENT '原始交易金额',
    exchange_rate       DECIMAL(12,6) NOT NULL COMMENT '折算汇率',
    balance_after       DECIMAL(18,2) NOT NULL COMMENT '操作后已用额度',
    idempotency_key     VARCHAR(128) NOT NULL UNIQUE COMMENT '幂等键',
    remark              VARCHAR(256) NULL COMMENT '备注',
    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_user_year (user_id, quota_year),
    INDEX idx_order_id (order_id),
    INDEX idx_action (action)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='购汇额度明细表';

-- 额度调整记录表：特殊额度调整（政策变化、审核增额等）
CREATE TABLE forex_quota_adjustments (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id             VARCHAR(64) NOT NULL,
    quota_year          SMALLINT NOT NULL,
    adjustment_type     VARCHAR(32) NOT NULL COMMENT 'policy_increase/manual_increase/policy_decrease/correction',
    original_limit      DECIMAL(18,2) NOT NULL,
    new_limit           DECIMAL(18,2) NOT NULL,
    adjusted_amount     DECIMAL(18,2) NOT NULL COMMENT '调整金额 = new_limit - original_limit',
    reason              VARCHAR(256) NOT NULL,
    approved_by         VARCHAR(64) NOT NULL COMMENT '审批人',
    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_user_year (user_id, quota_year),
    INDEX idx_adjustment_type (adjustment_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='额度调整记录表';
```

#### 额度服务实现

```python
class ForexQuotaService:
    """
    年度购汇额度追踪服务。
    额度生命周期：reserve（AML 前冻结）→ consume（购汇完成确认）→ release（交易取消释放）
    """

    # 各国个人年度购汇额度上限（等值美元）
    COUNTRY_QUOTA_LIMITS = {
        "CN": Decimal("50000"),    # 中国：$50000/年
        "JP": Decimal("1000000"),  # 日本：无实质限制，设大额
        "KR": Decimal("50000"),    # 韩国：$50000/年
        "IN": Decimal("250000"),   # 印度：LRS 限额 $250000/年
        "AU": Decimal("999999"),   # 澳大利亚：无实质限制
    }

    # 非美元货币到美元的折算缓存（每 15 秒更新）
    _rate_cache: dict = {}
    _cache_updated_at: datetime = None

    def reserve_quota(
        self, user_id: str, order_id: str,
        amount: Decimal, currency: str,
    ) -> dict:
        """
        冻结额度（AML 筛查前调用）。
        将金额从"可用额度"移入"冻结额度"。
        如果额度不足，直接拒绝交易。

        幂等键：reserve-{order_id}，同一订单不会重复冻结。
        """
        idempotency_key = f"reserve-{order_id}"

        # 幂等检查
        existing = self.db.query_one(
            "SELECT * FROM forex_quota_detail "
            "WHERE idempotency_key = %s",
            [idempotency_key]
        )
        if existing:
            return {
                "status": "already_reserved",
                "amount_usd": existing.amount_usd,
                "message": "额度已冻结",
            }

        # 折算为等值美元
        amount_usd = self._convert_to_usd(amount, currency)
        current_year = now().year

        # 获取用户额度记录（加行锁防并发）
        quota = self.db.query_one_for_update(
            "SELECT * FROM forex_quota_usage "
            "WHERE user_id = %s AND quota_year = %s "
            "FOR UPDATE",
            [user_id, current_year]
        )

        if not quota:
            # 首次使用 → 创建额度记录
            user = self.user_service.get_user(user_id)
            country = user.country_code if user else "CN"
            quota_limit = self.COUNTRY_QUOTA_LIMITS.get(
                country, Decimal("50000")
            )
            quota = self.db.insert("forex_quota_usage", {
                "user_id": user_id,
                "quota_year": current_year,
                "currency": "USD",
                "used_amount": Decimal("0"),
                "reserved_amount": Decimal("0"),
                "quota_limit": quota_limit,
                "created_at": now(),
            })

        # 检查剩余额度
        remaining = quota.quota_limit - quota.used_amount - quota.reserved_amount
        if remaining < amount_usd:
            return {
                "status": "quota_exceeded",
                "remaining_usd": remaining,
                "requested_usd": amount_usd,
                "shortage_usd": amount_usd - remaining,
                "yearly_used_usd": quota.used_amount,
                "yearly_reserved_usd": quota.reserved_amount,
                "yearly_limit_usd": quota.quota_limit,
                "message": (
                    f"年度购汇额度不足：剩余 ${remaining:,.2f}，"
                    f"需 ${amount_usd:,.2f}，不足 ${amount_usd - remaining:,.2f}"
                ),
            }

        # 冻结额度
        self.db.update("forex_quota_usage", {
            "reserved_amount": quota.reserved_amount + amount_usd,
            "last_updated_order": order_id,
        }, {"id": quota.id})

        # 记录明细
        self.db.insert("forex_quota_detail", {
            "user_id": user_id,
            "order_id": order_id,
            "quota_year": current_year,
            "action": "reserve",
            "amount_usd": amount_usd,
            "original_currency": currency,
            "original_amount": amount,
            "exchange_rate": self._get_usd_rate(currency),
            "balance_after": quota.used_amount,  # 已用额度不变
            "idempotency_key": idempotency_key,
            "remark": f"AML筛查前冻结，订单 {order_id}",
        })

        return {
            "status": "reserved",
            "amount_usd": amount_usd,
            "remaining_usd": remaining - amount_usd,
            "yearly_used_usd": quota.used_amount,
            "yearly_reserved_usd": quota.reserved_amount + amount_usd,
        }

    def consume_quota(self, user_id: str, order_id: str):
        """
        确认使用额度（购汇完成后调用）。
        将冻结额度转为已用额度。

        幂等键：consume-{order_id}
        """
        idempotency_key = f"consume-{order_id}"

        existing = self.db.query_one(
            "SELECT * FROM forex_quota_detail "
            "WHERE idempotency_key = %s",
            [idempotency_key]
        )
        if existing:
            return {"status": "already_consumed"}

        current_year = now().year

        # 查找该订单的冻结记录
        reserve_record = self.db.query_one(
            "SELECT * FROM forex_quota_detail "
            "WHERE order_id = %s AND action = 'reserve' "
            "AND quota_year = %s "
            "ORDER BY created_at DESC LIMIT 1",
            [order_id, current_year]
        )
        if not reserve_record:
            raise ValueError(f"未找到订单 {order_id} 的额度冻结记录")

        amount_usd = reserve_record.amount_usd

        # 更新额度：冻结 → 已用
        quota = self.db.query_one_for_update(
            "SELECT * FROM forex_quota_usage "
            "WHERE user_id = %s AND quota_year = %s FOR UPDATE",
            [user_id, current_year]
        )

        self.db.update("forex_quota_usage", {
            "used_amount": quota.used_amount + amount_usd,
            "reserved_amount": quota.reserved_amount - amount_usd,
            "last_updated_order": order_id,
        }, {"id": quota.id})

        # 记录明细
        self.db.insert("forex_quota_detail", {
            "user_id": user_id,
            "order_id": order_id,
            "quota_year": current_year,
            "action": "consume",
            "amount_usd": amount_usd,
            "original_currency": reserve_record.original_currency,
            "original_amount": reserve_record.original_amount,
            "exchange_rate": reserve_record.exchange_rate,
            "balance_after": quota.used_amount + amount_usd,
            "idempotency_key": idempotency_key,
            "remark": f"购汇完成，冻结转已用，订单 {order_id}",
        })

        return {
            "status": "consumed",
            "amount_usd": amount_usd,
            "yearly_used_usd": quota.used_amount + amount_usd,
            "remaining_usd": (
                quota.quota_limit
                - quota.used_amount - amount_usd
                - quota.reserved_amount + amount_usd
            ),
        }

    def release_quota(
        self, user_id: str, order_id: str, reason: str = "cancelled"
    ):
        """
        释放额度（交易取消/AML 拒绝时调用）。
        将冻结额度释放回可用额度。

        幂等键：release-{order_id}
        """
        idempotency_key = f"release-{order_id}"

        existing = self.db.query_one(
            "SELECT * FROM forex_quota_detail "
            "WHERE idempotency_key = %s",
            [idempotency_key]
        )
        if existing:
            return {"status": "already_released"}

        current_year = now().year

        # 查找冻结记录
        reserve_record = self.db.query_one(
            "SELECT * FROM forex_quota_detail "
            "WHERE order_id = %s AND action = 'reserve' "
            "AND quota_year = %s "
            "ORDER BY created_at DESC LIMIT 1",
            [order_id, current_year]
        )
        if not reserve_record:
            return {"status": "no_reserve_found"}

        amount_usd = reserve_record.amount_usd

        # 更新额度：冻结 → 释放
        quota = self.db.query_one_for_update(
            "SELECT * FROM forex_quota_usage "
            "WHERE user_id = %s AND quota_year = %s FOR UPDATE",
            [user_id, current_year]
        )

        self.db.update("forex_quota_usage", {
            "reserved_amount": max(
                quota.reserved_amount - amount_usd, Decimal("0")
            ),
            "last_updated_order": order_id,
        }, {"id": quota.id})

        # 记录明细
        self.db.insert("forex_quota_detail", {
            "user_id": user_id,
            "order_id": order_id,
            "quota_year": current_year,
            "action": "release",
            "amount_usd": amount_usd,
            "original_currency": reserve_record.original_currency,
            "original_amount": reserve_record.original_amount,
            "exchange_rate": reserve_record.exchange_rate,
            "balance_after": quota.used_amount,  # 已用额度不变
            "idempotency_key": idempotency_key,
            "remark": f"额度释放，原因：{reason}，订单 {order_id}",
        })

        return {"status": "released", "amount_usd": amount_usd}

    def refund_quota(
        self, user_id: str, order_id: str,
        refund_amount_usd: Decimal, reason: str = "partial_refund",
    ):
        """
        退回已用额度（部分退款时调用）。
        将已用额度减少，等同于增加可用额度。

        幂等键：refund-{order_id}-{refund_amount_usd}
        """
        # 部分退款可能多次调用，金额作为幂等键的一部分
        idempotency_key = (
            f"refund-{order_id}-{refund_amount_usd:.2f}"
        )

        existing = self.db.query_one(
            "SELECT * FROM forex_quota_detail "
            "WHERE idempotency_key = %s",
            [idempotency_key]
        )
        if existing:
            return {"status": "already_refunded"}

        current_year = now().year
        quota = self.db.query_one_for_update(
            "SELECT * FROM forex_quota_usage "
            "WHERE user_id = %s AND quota_year = %s FOR UPDATE",
            [user_id, current_year]
        )

        # 退回额度：减少已用额度
        new_used = max(quota.used_amount - refund_amount_usd, Decimal("0"))
        self.db.update("forex_quota_usage", {
            "used_amount": new_used,
            "last_updated_order": order_id,
        }, {"id": quota.id})

        self.db.insert("forex_quota_detail", {
            "user_id": user_id,
            "order_id": order_id,
            "quota_year": current_year,
            "action": "refund",
            "amount_usd": refund_amount_usd,
            "original_currency": "USD",
            "original_amount": refund_amount_usd,
            "exchange_rate": Decimal("1.0"),
            "balance_after": new_used,
            "idempotency_key": idempotency_key,
            "remark": f"额度退回，原因：{reason}，订单 {order_id}",
        })

        return {
            "status": "refunded",
            "amount_usd": refund_amount_usd,
            "yearly_used_usd": new_used,
        }

    def check_quota(self, user_id: str, amount: Decimal, currency: str) -> dict:
        """
        查询用户剩余额度（只读，不冻结）。
        用于前端展示和下单前的预校验。
        """
        amount_usd = self._convert_to_usd(amount, currency)
        current_year = now().year

        quota = self.db.query_one(
            "SELECT * FROM forex_quota_usage "
            "WHERE user_id = %s AND quota_year = %s",
            [user_id, current_year]
        )

        if not quota:
            user = self.user_service.get_user(user_id)
            country = user.country_code if user else "CN"
            quota_limit = self.COUNTRY_QUOTA_LIMITS.get(
                country, Decimal("50000")
            )
            return {
                "has_enough": True,
                "remaining_usd": quota_limit,
                "yearly_used_usd": Decimal("0"),
                "yearly_reserved_usd": Decimal("0"),
                "yearly_limit_usd": quota_limit,
                "requested_usd": amount_usd,
            }

        remaining = quota.quota_limit - quota.used_amount - quota.reserved_amount
        return {
            "has_enough": remaining >= amount_usd,
            "remaining_usd": remaining,
            "yearly_used_usd": quota.used_amount,
            "yearly_reserved_usd": quota.reserved_amount,
            "yearly_limit_usd": quota.quota_limit,
            "requested_usd": amount_usd,
        }

    def _convert_to_usd(self, amount: Decimal, currency: str) -> Decimal:
        """将任意币种金额折算为等值美元"""
        if currency == "USD":
            return amount
        rate = self._get_usd_rate(currency)
        return (amount / rate).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)

    def _get_usd_rate(self, currency: str) -> Decimal:
        """获取货币兑美元汇率"""
        if currency == "USD":
            return Decimal("1.0")
        # 从汇率缓存获取
        if (self._cache_updated_at
            and (now() - self._cache_updated_at).seconds < 15):
            return self._rate_cache.get(currency, Decimal("1.0"))
        # 刷新缓存
        self._refresh_rate_cache()
        return self._rate_cache.get(currency, Decimal("1.0"))

    def _refresh_rate_cache(self):
        """刷新汇率缓存（每 15 秒一次）"""
        rates = self.rate_service.get_all_rates_against_usd()
        for curr, rate in rates.items():
            self._rate_cache[curr] = Decimal(str(rate))
        self._cache_updated_at = now()

    def generate_quota_report(self, user_id: str) -> dict:
        """生成用户额度使用报告（用户可查看）"""
        current_year = now().year
        quota = self.db.query_one(
            "SELECT * FROM forex_quota_usage "
            "WHERE user_id = %s AND quota_year = %s",
            [user_id, current_year]
        )
        details = self.db.query(
            "SELECT * FROM forex_quota_detail "
            "WHERE user_id = %s AND quota_year = %s "
            "ORDER BY created_at DESC LIMIT 50",
            [user_id, current_year]
        )

        return {
            "user_id": user_id,
            "year": current_year,
            "limit_usd": quota.quota_limit if quota else 50000,
            "used_usd": quota.used_amount if quota else 0,
            "reserved_usd": quota.reserved_amount if quota else 0,
            "remaining_usd": (
                quota.quota_limit - quota.used_amount - quota.reserved_amount
                if quota else 50000
            ),
            "usage_pct": f"{quota.used_amount / quota.quota_limit * 100:.1f}%"
                if quota else "0.0%",
            "recent_details": [
                {
                    "date": d.created_at.isoformat(),
                    "action": d.action,
                    "amount_usd": str(d.amount_usd),
                    "original": f"{d.original_amount} {d.original_currency}",
                    "order_id": d.order_id,
                }
                for d in details
            ],
        }
```

#### 额度与 AML 筛查的集成

```python
class AMLScreeningWithQuota(AMLScreeningService):
    """集成额度检查的 AML 筛查服务"""

    def screen(self, order):
        # === 先冻结额度，再做 AML 筛查 ===
        # 原因：AML 筛查可能耗时数分钟，期间不能让其他交易占掉额度
        amount_usd = self.forex_quota_service._convert_to_usd(
            order.amount_cny, order.source_currency
        )

        reserve_result = self.forex_quota_service.reserve_quota(
            user_id=order.user_id,
            order_id=order.order_id,
            amount=order.amount_cny,
            currency=order.source_currency,
        )

        if reserve_result["status"] == "quota_exceeded":
            # 额度不足 → 直接拒绝，不需要做 AML 筛查
            return AMLResult(
                status="rejected",
                reason="forex_quota_exceeded",
                detail=reserve_result,
            )

        # === 执行 AML 筛查 ===
        aml_result = super().screen(order)

        if aml_result.status == "rejected":
            # AML 拒绝 → 释放冻结的额度
            self.forex_quota_service.release_quota(
                user_id=order.user_id,
                order_id=order.order_id,
                reason="aml_rejected",
            )
        elif aml_result.status == "manual_review":
            # 人工审核 → 额度保持冻结，等待审核结果
            pass  # 冻结额度不变
        # approved 的情况：额度保持冻结，等购汇完成后 consume

        return aml_result
```

**额度生命周期状态图：**

```
                    reserve（AML前冻结）
  可用额度 ────────────────────────────→ 冻结额度
     ↑                                      │
     │                     ┌────────────────┤
     │                     │                │
     │            release（AML拒绝/取消）   consume（购汇完成）
     │                     │                │
     │                     ↓                ↓
     └────────────── 可用额度          已用额度
                                         │
                              refund（部分退款）
                                         │
                                         ↓
                                    可用额度（部分回退）
```

### 三方对账

**对账数据源：**

| 数据源 | 结算周期 | 数据格式 | 延迟 |
|--------|---------|---------|------|
| 内部订单系统 | 实时 | 数据库 | 0 |
| 支付渠道（支付宝） | T+1 | 对账文件（CSV） | 次日 10:00 |
| 结算银行 | T+2 | SWIFT MT940 | T+2 |

**对账流程：**

```python
class ReconciliationService:
    def reconcile(self, date):
        # 1. 加载三方数据
        internal = self.load_internal_records(date)
        channel = self.load_channel_statement(date)      # T+1 才有
        bank = self.load_bank_statement(date)             # T+2 才有
        
        # 2. 内部 vs 渠道对账
        internal_channel_diff = self.compare(internal, channel, key="channel_tx_id")
        
        # 3. 内部 vs 银行对账
        internal_bank_diff = self.compare(internal, bank, key="bank_tx_id")
        
        # 4. 处理差异
        for diff in internal_channel_diff:
            self.handle_diff(diff, source="internal_vs_channel")
        
        for diff in internal_bank_diff:
            self.handle_diff(diff, source="internal_vs_bank")
    
    def compare(self, source_a, source_b, key):
        """比对两个数据源"""
        map_a = {getattr(r, key): r for r in source_a}
        map_b = {getattr(r, key): r for r in source_b}
        
        diffs = []
        
        # A 有 B 无
        for k, r in map_a.items():
            if k not in map_b:
                diffs.append({"type": "missing_in_b", "key": k, "record_a": r})
        
        # B 有 A 无
        for k, r in map_b.items():
            if k not in map_a:
                diffs.append({"type": "missing_in_a", "key": k, "record_b": r})
        
        # 金额不一致
        for k in set(map_a.keys()) & set(map_b.keys()):
            if map_a[k].amount != map_b[k].amount:
                diffs.append({
                    "type": "amount_mismatch",
                    "key": k,
                    "amount_a": map_a[k].amount,
                    "amount_b": map_b[k].amount,
                    "diff": map_a[k].amount - map_b[k].amount
                })
        
        return diffs
    
    def handle_diff(self, diff, source):
        if diff["type"] == "missing_in_b":
            # 渠道/银行无记录 → 可能是延迟，3天后仍无则人工处理
            if diff.get("record_a").created_at < now() - timedelta(days=3):
                self.alert_team(f"对账差异: {source}, {diff}")

        elif diff["type"] == "missing_in_a":
            # 内部无记录 → 渠道/银行有我们不知道的交易 → 严重，立即告警
            self.alert_team(f"未知交易: {source}, {diff}")

        elif diff["type"] == "amount_mismatch":
            # 金额不一致 → 检查是否是汇率差异（可容忍）还是真正的错误
            if abs(diff["diff"]) > Decimal("0.01"):
                self.alert_team(f"金额差异: {source}, {diff}")
```

### 对账差异自动解决引擎

**对账差异不应全部依赖人工处理。通过分类规则引擎，80% 以上的差异可自动归类和处理，仅 20% 需要人工介入。自动解决不是简单忽略差异，而是根据业务规则准确归因并记录处理依据。**

#### 差异自动分类体系

| 差异类别 | 差异代码 | 典型场景 | 自动处理策略 | 风险等级 |
|---------|---------|---------|------------|---------|
| 时间差 | `TIMING_DIFF` | T+0 内部记录 vs T+1 渠道文件 | 等待次日数据后自动匹配 | 低 |
| 手续费差异 | `FEE_DIFF` | 渠道扣除手续费但未提前通知 | 差额 ≤ ¥1 且方向一致→自动标记 | 低 |
| 汇率精度差异 | `RATE_PRECISION_DIFF` | 内部 6 位精度 vs 渠道 2 位精度四舍五入 | 差额 ≤ ¥0.05 → 自动标记 | 低 |
| 币种换算差异 | `CURRENCY_ROUNDING_DIFF` | 跨币种金额四舍五入导致的 ¥0.01-0.50 差异 | 差额 ≤ ¥0.50 → 自动标记 | 低 |
| 缺失记录 | `MISSING_RECORD` | 一方有另一方无 | 延迟 → 等待；>3 天 → 告警 | 中 |
| 重复记录 | `DUPLICATE_RECORD` | 渠道重复入账 | 自动识别重复 → 请求渠道冲正 | 中 |
| 系统错误 | `SYSTEM_ERROR` | 内部系统 bug 或渠道接口异常 | 立即告警 → 人工修复 | 高 |
| 欺诈交易 | `FRAUD_SUSPECTED` | 渠道有但内部无（可能是伪造交易） | 立即告警 → 安全团队介入 | 高 |

```python
class ReconciliationAutoResolutionEngine:
    """
    对账差异自动解决引擎。
    核心思路：每条差异经过规则链逐条匹配，命中规则则自动处理并记录依据，
    未命中则升级为人工处理。
    """

    # 规则链（按优先级排列，高优先级规则先匹配）
    RESOLUTION_RULES = [
        {
            "rule_id": "R001",
            "rule_name": "小额手续费差异",
            "diff_type": "amount_mismatch",
            "conditions": {
                "diff_amount_abs_lte": Decimal("1.00"),   # |差异| ≤ ¥1.00
                "direction": "external_less",             # 外部金额 ≤ 内部金额
            },
            "resolution": "FEE_DIFF",
            "action": "auto_resolve",
            "confidence": "high",
        },
        {
            "rule_id": "R002",
            "rule_name": "汇率精度差异",
            "diff_type": "amount_mismatch",
            "conditions": {
                "diff_amount_abs_lte": Decimal("0.05"),   # |差异| ≤ ¥0.05
                "involves_forex": True,                   # 涉及汇率换算
            },
            "resolution": "RATE_PRECISION_DIFF",
            "action": "auto_resolve",
            "confidence": "high",
        },
        {
            "rule_id": "R003",
            "rule_name": "币种换算四舍五入差异",
            "diff_type": "amount_mismatch",
            "conditions": {
                "diff_amount_abs_lte": Decimal("0.50"),   # |差异| ≤ ¥0.50
                "cross_currency": True,                   # 跨币种交易
            },
            "resolution": "CURRENCY_ROUNDING_DIFF",
            "action": "auto_resolve",
            "confidence": "medium",
        },
        {
            "rule_id": "R004",
            "rule_name": "时间差（T+N 延迟）",
            "diff_type": "missing_in_b",
            "conditions": {
                "delay_days_lte": 2,                      # 延迟 ≤ 2 天
                "source_pair": "internal_vs_channel",     # 内部 vs 渠道
            },
            "resolution": "TIMING_DIFF",
            "action": "defer_and_recheck",                # 延迟处理，等待次日数据
            "confidence": "high",
        },
        {
            "rule_id": "R005",
            "rule_name": "银行 T+2 延迟",
            "diff_type": "missing_in_b",
            "conditions": {
                "delay_days_lte": 3,                      # 延迟 ≤ 3 天
                "source_pair": "internal_vs_bank",        # 内部 vs 银行
            },
            "resolution": "TIMING_DIFF",
            "action": "defer_and_recheck",
            "confidence": "high",
        },
        {
            "rule_id": "R006",
            "rule_name": "渠道重复入账",
            "diff_type": "duplicate",
            "conditions": {
                "same_amount": True,                      # 金额相同
                "same_timestamp_window": 300,             # 5 分钟内
            },
            "resolution": "DUPLICATE_RECORD",
            "action": "auto_resolve_and_chargeback",      # 自动标记 + 请求冲正
            "confidence": "medium",
        },
        {
            "rule_id": "R007",
            "rule_name": "未知交易（内部无记录）",
            "diff_type": "missing_in_a",
            "conditions": {},                             # 无条件匹配
            "resolution": "UNKNOWN_TXN",
            "action": "escalate",                         # 必须人工处理
            "confidence": "low",
        },
        {
            "rule_id": "R008",
            "rule_name": "大额差异",
            "diff_type": "amount_mismatch",
            "conditions": {
                "diff_amount_abs_gt": Decimal("1.00"),    # |差异| > ¥1.00
            },
            "resolution": "LARGE_AMOUNT_DIFF",
            "action": "escalate",                         # 大额差异必须人工
            "confidence": "low",
        },
    ]

    def auto_resolve_batch(self, batch_id: str) -> dict:
        """
        批量自动处理一个对账批次中的所有差异。
        返回处理统计信息。
        """
        diffs = self.db.query(
            "SELECT * FROM reconciliation_diffs "
            "WHERE batch_id = %s AND status = 'open' "
            "ORDER BY diff_amount ASC",  # 小额差异先处理（更容易自动解决）
            [batch_id]
        )

        stats = {
            "total": len(diffs),
            "auto_resolved": 0,
            "deferred": 0,
            "escalated": 0,
            "by_resolution_type": {},
            "by_rule_id": {},
        }

        for diff in diffs:
            result = self._evaluate_rules(diff)

            if result["action"] == "auto_resolve":
                self._apply_auto_resolution(diff, result)
                stats["auto_resolved"] += 1
            elif result["action"] == "defer_and_recheck":
                self._defer_diff(diff, result)
                stats["deferred"] += 1
            elif result["action"] == "escalate":
                self._escalate_diff(diff, result)
                stats["escalated"] += 1

            # 统计
            resolution = result["resolution"]
            stats["by_resolution_type"][resolution] = \
                stats["by_resolution_type"].get(resolution, 0) + 1
            rule_id = result["rule_id"]
            stats["by_rule_id"][rule_id] = \
                stats["by_rule_id"].get(rule_id, 0) + 1

        # 更新批次统计
        self.db.update("reconciliation_batches", {
            "auto_resolved_count": stats["auto_resolved"],
            "manual_count": stats["escalated"],
        }, {"batch_id": batch_id})

        return stats

    def _evaluate_rules(self, diff) -> dict:
        """
        逐条评估规则链，返回第一个匹配的规则。
        如果没有规则匹配，默认升级为人工处理。
        """
        for rule in self.RESOLUTION_RULES:
            if self._rule_matches(rule, diff):
                return {
                    "rule_id": rule["rule_id"],
                    "rule_name": rule["rule_name"],
                    "resolution": rule["resolution"],
                    "action": rule["action"],
                    "confidence": rule["confidence"],
                }

        # 无规则匹配 → 默认升级
        return {
            "rule_id": "DEFAULT",
            "rule_name": "无匹配规则",
            "resolution": "UNCLASSIFIED",
            "action": "escalate",
            "confidence": "low",
        }

    def _rule_matches(self, rule: dict, diff) -> bool:
        """检查单条规则是否匹配该差异"""
        conditions = rule.get("conditions", {})

        # 差异类型必须匹配
        if rule["diff_type"] != diff.diff_type:
            return False

        # 检查各条件
        if "diff_amount_abs_lte" in conditions:
            if abs(diff.diff_amount or 0) > conditions["diff_amount_abs_lte"]:
                return False

        if "diff_amount_abs_gt" in conditions:
            if abs(diff.diff_amount or 0) <= conditions["diff_amount_abs_gt"]:
                return False

        if "direction" in conditions:
            if conditions["direction"] == "external_less":
                if (diff.external_amount or 0) > (diff.internal_amount or 0):
                    return False

        if "delay_days_lte" in conditions:
            delay = (now().date() - diff.reconcile_date).days
            if delay > conditions["delay_days_lte"]:
                return False

        if "source_pair" in conditions:
            if diff.source_pair != conditions["source_pair"]:
                return False

        if conditions.get("involves_forex"):
            order = self.db.get_payment_order(diff.internal_tx_id)
            if not order or order.source_currency == order.target_currency:
                return False

        if conditions.get("cross_currency"):
            order = self.db.get_payment_order(diff.internal_tx_id)
            if not order or order.source_currency == order.target_currency:
                return False

        if conditions.get("same_amount"):
            # 检查是否有相同金额的重复记录
            dupes = self.db.query(
                "SELECT * FROM reconciliation_diffs "
                "WHERE batch_id = %s AND internal_tx_id = %s "
                "AND ABS(internal_amount - %s) < 0.01 "
                "AND id != %s",
                [diff.batch_id, diff.internal_tx_id,
                 diff.internal_amount, diff.id]
            )
            if not dupes:
                return False

        if "same_timestamp_window" in conditions:
            window = conditions["same_timestamp_window"]
            dupes = self.db.query(
                "SELECT * FROM reconciliation_diffs "
                "WHERE batch_id = %s AND internal_tx_id = %s "
                "AND TIMESTAMPDIFF(SECOND, created_at, %s) < %s "
                "AND id != %s",
                [diff.batch_id, diff.internal_tx_id,
                 diff.created_at, window, diff.id]
            )
            if not dupes:
                return False

        return True

    def _apply_auto_resolution(self, diff, rule_result: dict):
        """应用自动解决：更新差异状态并记录解决依据"""
        resolution_detail = {
            "rule_id": rule_result["rule_id"],
            "rule_name": rule_result["rule_name"],
            "resolution_type": rule_result["resolution"],
            "confidence": rule_result["confidence"],
            "internal_amount": str(diff.internal_amount),
            "external_amount": str(diff.external_amount),
            "diff_amount": str(diff.diff_amount),
            "resolved_at": now().isoformat(),
            "resolved_by": "auto_resolution_engine",
        }

        self.db.update("reconciliation_diffs", {
            "status": "auto_resolved",
            "resolution": rule_result["resolution"],
            "resolution_detail": json.dumps(resolution_detail, ensure_ascii=False),
            "auto_resolve_rule": rule_result["rule_id"],
            "resolved_at": now(),
        }, {"id": diff.id})

        # 如果是手续费差异 → 记录到手续费差异汇总表
        if rule_result["resolution"] == "FEE_DIFF":
            self.db.insert("fee_diff_summary", {
                "batch_id": diff.batch_id,
                "source_pair": diff.source_pair,
                "channel_tx_id": diff.channel_tx_id,
                "internal_amount": diff.internal_amount,
                "external_amount": diff.external_amount,
                "fee_diff_amount": diff.diff_amount,
                "rule_id": rule_result["rule_id"],
                "created_at": now(),
            })

    def _defer_diff(self, diff, rule_result: dict):
        """延迟处理：等待次日数据后重新检查"""
        self.db.update("reconciliation_diffs", {
            "resolution": rule_result["resolution"],
            "auto_resolve_rule": rule_result["rule_id"],
            "resolution_detail": json.dumps({
                "rule_id": rule_result["rule_id"],
                "reason": "等待次日对账数据自动匹配",
                "deferred_at": now().isoformat(),
                "recheck_date": (now() + timedelta(days=1)).date().isoformat(),
            }, ensure_ascii=False),
        }, {"id": diff.id})

        # 调度次日重新检查
        self.scheduler.schedule(
            "recheck_diff",
            run_at=now() + timedelta(days=1),
            callback=self._recheck_deferred_diff,
            args=[diff.id],
        )

    def _recheck_deferred_diff(self, diff_id: int):
        """重新检查延迟的差异：次日数据到达后是否能匹配"""
        diff = self.db.get_reconciliation_diff(diff_id)
        if diff.status != "open":
            return  # 已处理

        # 用次日数据重新匹配
        next_day = diff.reconcile_date + timedelta(days=1)
        external_records = self._load_external_records(
            diff.source_pair, next_day
        )

        matched = self._try_match(diff, external_records)
        if matched:
            # 匹配成功 → 自动解决
            self.db.update("reconciliation_diffs", {
                "status": "auto_resolved",
                "resolution": "TIMING_DIFF",
                "resolution_detail": json.dumps({
                    "rule_id": "R004_RECHECK",
                    "original_date": diff.reconcile_date.isoformat(),
                    "matched_date": next_day.isoformat(),
                    "matched_record": matched,
                    "resolved_at": now().isoformat(),
                }, ensure_ascii=False),
                "resolved_at": now(),
            }, {"id": diff_id})
        else:
            # 次日仍未匹配 → 如果延迟超过 3 天则升级
            if (now().date() - diff.reconcile_date).days > 3:
                self._escalate_diff(diff, {
                    "rule_id": "R004_TIMEOUT",
                    "rule_name": "时间差超时",
                    "resolution": "TIMING_DIFF_TIMEOUT",
                    "confidence": "low",
                })

    def _escalate_diff(self, diff, rule_result: dict):
        """升级为人工处理：创建告警并分配给对账团队"""
        self.db.update("reconciliation_diffs", {
            "status": "escalated",
            "resolution": rule_result["resolution"],
            "auto_resolve_rule": rule_result["rule_id"],
            "resolution_detail": json.dumps({
                "rule_id": rule_result["rule_id"],
                "escalation_reason": rule_result["rule_name"],
                "escalated_at": now().isoformat(),
                "requires_manual_review": True,
            }, ensure_ascii=False),
        }, {"id": diff.id})

        # 根据风险等级决定告警方式
        if rule_result["resolution"] in ("UNKNOWN_TXN", "FRAUD_SUSPECTED"):
            # 高风险 → 即时告警
            self.alert_team(
                f"[紧急] 对账差异需人工处理：{rule_result['resolution']}\n"
                f"差异ID: {diff.id}, 批次: {diff.batch_id}\n"
                f"内部交易号: {diff.internal_tx_id}\n"
                f"内部金额: {diff.internal_amount}, 外部金额: {diff.external_amount}\n"
                f"差异金额: {diff.diff_amount}"
            )
        else:
            # 普通差异 → 加入待处理队列
            self.db.insert("reconciliation_work_queue", {
                "diff_id": diff.id,
                "batch_id": diff.batch_id,
                "priority": "normal",
                "assigned_team": "reconciliation",
                "status": "pending",
                "created_at": now(),
            })

    def generate_resolution_report(self, date) -> dict:
        """生成对账差异处理报告"""
        diffs = self.db.query(
            "SELECT * FROM reconciliation_diffs "
            "WHERE reconcile_date = %s",
            [date]
        )

        total = len(diffs)
        auto_resolved = sum(1 for d in diffs if d.status == "auto_resolved")
        deferred = sum(1 for d in diffs if d.status == "open"
                       and d.auto_resolve_rule in ("R004", "R005"))
        escalated = sum(1 for d in diffs if d.status == "escalated")
        manual_resolved = sum(1 for d in diffs if d.status == "manual_resolved")

        # 按差异类别统计
        by_resolution = {}
        for d in diffs:
            key = d.resolution or "UNCLASSIFIED"
            by_resolution[key] = by_resolution.get(key, 0) + 1

        # 按规则命中统计
        by_rule = {}
        for d in diffs:
            key = d.auto_resolve_rule or "NO_RULE"
            by_rule[key] = by_rule.get(key, 0) + 1

        return {
            "date": str(date),
            "total_diffs": total,
            "auto_resolved": auto_resolved,
            "auto_resolve_rate": f"{auto_resolved / total * 100:.1f}%"
                if total > 0 else "N/A",
            "deferred": deferred,
            "escalated": escalated,
            "manual_resolved": manual_resolved,
            "by_resolution_type": by_resolution,
            "by_rule_hit": by_rule,
            "target_auto_resolve_rate": "80%",
            "meets_target": auto_resolved / total >= 0.8 if total > 0 else True,
        }
```
```

### 幂等性保证

**跨境支付的幂等性比普通支付更关键——重复执行一笔跨境汇款意味着多付一次钱，无法撤回。**

```python
class IdempotentCrossBorderPayment:
    def execute_payment(self, order):
        # 幂等键：order_id
        existing = self.db.get_payment_by_order(order.id)
        if existing:
            if existing.status == "completed":
                return existing  # 已完成，直接返回
            elif existing.status == "processing":
                return self.wait_for_completion(order.id)
            elif existing.status == "failed":
                pass  # 允许重试

        # 记录开始处理
        self.db.create_payment(order.id, status="processing")

        try:
            result = self._do_execute(order)
            self.db.update_payment(order.id, status="completed", result=result)
            return result
        except Exception as e:
            self.db.update_payment(order.id, status="failed", error=str(e))
            raise
```

### Saga 每步骤幂等性完整实现

**仅靠订单级幂等性不够。当 Saga 执行到第 4 步时系统崩溃，恢复后必须跳过前 3 步已完成的结果，从第 4 步继续。每一步都必须有独立的幂等保证，防止重复扣款、重复购汇、重复汇款。**

核心原则：
1. **每个步骤执行前必须查询 step_log**，如果该步骤已有 `completed` 记录，直接使用已有结果跳过
2. **幂等键由 saga_id + step_name 组成**，全局唯一，不可重复
3. **外部调用必须携带幂等键**，渠道/银行侧也有幂等保证
4. **补偿操作同样需要幂等**，防止重复退款或重复退汇

```python
class IdempotentSagaStepExecutor:
    """
    带完整幂等保证的 Saga 步骤执行器。
    每个步骤执行前检查 step_log，确保不会重复执行已完成的步骤。
    """

    # 步骤定义：名称 → 执行函数、补偿函数、外部幂等键模板
    STEP_DEFINITIONS = {
        0: {
            "name": "user_payment",
            "compensate_name": "refund_user",
            "idempotency_key_template": "pay-{order_id}",
            "compensate_idempotency_key_template": "refund-{order_id}",
            "timeout_seconds": 30,
        },
        1: {
            "name": "aml_screening",
            "compensate_name": "refund_user_aml",
            "idempotency_key_template": "aml-{order_id}",
            "compensate_idempotency_key_template": "aml-refund-{order_id}",
            "timeout_seconds": 300,  # AML 筛查最长 5 分钟
        },
        2: {
            "name": "forex_convert",
            "compensate_name": "forex_reverse",
            "idempotency_key_template": "forex-{order_id}",
            "compensate_idempotency_key_template": "forex-rev-{order_id}",
            "timeout_seconds": 60,
        },
        3: {
            "name": "cross_border_transfer",
            "compensate_name": "request_merchant_return",
            "idempotency_key_template": "transfer-{order_id}",
            "compensate_idempotency_key_template": "transfer-ret-{order_id}",
            "timeout_seconds": 120,
        },
        4: {
            "name": "merchant_settle",
            "compensate_name": "merchant_refund",
            "idempotency_key_template": "settle-{order_id}",
            "compensate_idempotency_key_template": "settle-refund-{order_id}",
            "timeout_seconds": 60,
        },
    }

    def execute_step(self, saga_id: str, order, step_index: int):
        """
        执行 Saga 的某一步骤，带完整幂等保证。
        返回步骤执行结果，如果步骤已完成则返回已有结果。
        """
        step_def = self.STEP_DEFINITIONS[step_index]
        step_name = step_def["name"]

        # === 1. 构造步骤级幂等键 ===
        idempotency_key = step_def["idempotency_key_template"].format(
            order_id=order.order_id
        )

        # === 2. 查询 step_log：该步骤是否已成功完成 ===
        existing_log = self.db.query_one(
            "SELECT * FROM saga_state_log "
            "WHERE saga_id = %s AND step_name = %s "
            "AND direction = 'forward' AND status = 'completed' "
            "ORDER BY completed_at DESC LIMIT 1",
            [saga_id, step_name]
        )

        if existing_log:
            # 步骤已完成 → 直接返回已有结果，跳过执行
            logger.info(
                f"Step {step_name} already completed for saga {saga_id}, "
                f"skipping. Result: {existing_log.output_data}"
            )
            return StepResult(
                status="skipped_completed",
                data=json.loads(existing_log.output_data),
                idempotency_key=idempotency_key,
                was_recovered=True,
            )

        # === 3. 检查是否有正在执行中的记录（防并发） ===
        executing_log = self.db.query_one(
            "SELECT * FROM saga_state_log "
            "WHERE saga_id = %s AND step_name = %s "
            "AND direction = 'forward' AND status = 'started' "
            "AND started_at > DATE_SUB(NOW(), INTERVAL %s SECOND) "
            "ORDER BY started_at DESC LIMIT 1",
            [saga_id, step_name, step_def["timeout_seconds"]]
        )

        if executing_log:
            # 另一个进程正在执行该步骤 → 等待其完成
            logger.info(
                f"Step {step_name} is being executed by another process "
                f"for saga {saga_id}, waiting..."
            )
            return self._wait_for_step_completion(
                saga_id, step_name, step_def["timeout_seconds"]
            )

        # === 4. 写入步骤开始日志（占位，防止并发执行） ===
        log_id = self.db.insert("saga_state_log", {
            "saga_id": saga_id,
            "order_id": order.order_id,
            "step_name": step_name,
            "step_index": step_index,
            "direction": "forward",
            "status": "started",
            "input_data": json.dumps(self._get_step_input(order, step_index)),
            "idempotency_key": idempotency_key,
            "started_at": now(),
        })

        # === 5. 执行步骤 ===
        try:
            result = self._do_execute_step(order, step_index, idempotency_key)

            # 更新步骤日志为完成
            self.db.update("saga_state_log", {
                "status": "completed",
                "output_data": json.dumps(result),
                "completed_at": now(),
                "duration_ms": (now() - started).total_seconds() * 1000,
            }, {"id": log_id})

            return StepResult(
                status="completed",
                data=result,
                idempotency_key=idempotency_key,
                was_recovered=False,
            )

        except Exception as e:
            # 更新步骤日志为失败
            self.db.update("saga_state_log", {
                "status": "failed",
                "error_message": str(e),
                "completed_at": now(),
                "duration_ms": (now() - started).total_seconds() * 1000,
            }, {"id": log_id})
            raise

    def compensate_step(self, saga_id: str, order, step_index: int):
        """
        执行某一步骤的补偿操作，带幂等保证。
        补偿操作同样可能被多次调用（崩溃恢复），必须幂等。
        """
        step_def = self.STEP_DEFINITIONS[step_index]
        step_name = step_def["name"]
        compensate_name = step_def["compensate_name"]

        # 构造补偿幂等键
        idempotency_key = step_def["compensate_idempotency_key_template"].format(
            order_id=order.order_id
        )

        # === 1. 检查补偿是否已完成 ===
        existing_comp = self.db.query_one(
            "SELECT * FROM saga_state_log "
            "WHERE saga_id = %s AND step_name = %s "
            "AND direction = 'compensate' AND status = 'completed' "
            "ORDER BY completed_at DESC LIMIT 1",
            [saga_id, step_name]
        )

        if existing_comp:
            logger.info(
                f"Compensation for {step_name} already completed "
                f"for saga {saga_id}, skipping."
            )
            return StepResult(
                status="skipped_compensated",
                data=json.loads(existing_comp.output_data),
                idempotency_key=idempotency_key,
                was_recovered=True,
            )

        # === 2. 写入补偿开始日志 ===
        log_id = self.db.insert("saga_state_log", {
            "saga_id": saga_id,
            "order_id": order.order_id,
            "step_name": step_name,
            "step_index": step_index,
            "direction": "compensate",
            "status": "started",
            "input_data": json.dumps({"action": compensate_name}),
            "idempotency_key": idempotency_key,
            "started_at": now(),
        })

        # === 3. 执行补偿 ===
        try:
            result = self._do_compensate_step(
                order, step_index, idempotency_key
            )

            self.db.update("saga_state_log", {
                "status": "completed",
                "output_data": json.dumps(result),
                "completed_at": now(),
            }, {"id": log_id})

            return StepResult(
                status="completed",
                data=result,
                idempotency_key=idempotency_key,
                was_recovered=False,
            )

        except Exception as e:
            self.db.update("saga_state_log", {
                "status": "failed",
                "error_message": str(e),
                "completed_at": now(),
            }, {"id": log_id})
            raise

    def _do_execute_step(self, order, step_index: int, idempotency_key: str):
        """实际执行步骤，携带幂等键调用外部服务"""
        if step_index == 0:
            # Step 1: 用户支付
            return self.payment_service.charge(
                user_id=order.user_id,
                amount=order.amount_cny,
                method=order.payment_method,
                idempotency_key=idempotency_key,  # 渠道侧幂等
            )
        elif step_index == 1:
            # Step 2: AML 筛查
            return self.aml_service.screen(
                order=order,
                idempotency_key=idempotency_key,
            )
        elif step_index == 2:
            # Step 3: 汇率锁定 + 购汇
            rate_lock = self.rate_service.lock_rate(
                order.source_currency, order.target_currency,
                order.amount_cny,
                idempotency_key=idempotency_key,
            )
            return self.forex_service.convert(
                lock=rate_lock,
                idempotency_key=idempotency_key,  # 外汇供应商侧幂等
            )
        elif step_index == 3:
            # Step 4: 跨境汇款（最危险，幂等最关键）
            return self.transfer_service.send(
                amount=order.target_amount,
                currency=order.target_currency,
                beneficiary_account=order.merchant_bank_account,
                idempotency_key=idempotency_key,  # 银行侧幂等
            )
        elif step_index == 4:
            # Step 5: 商家结算
            return self.settlement_service.schedule(
                merchant_id=order.merchant_id,
                amount=order.target_amount,
                currency=order.target_currency,
                settlement_date=now() + timedelta(days=2),
                idempotency_key=idempotency_key,
            )

    def _do_compensate_step(self, order, step_index: int, idempotency_key: str):
        """实际执行补偿操作"""
        if step_index == 0:
            # 补偿 Step 1: 退款给用户
            return self.payment_service.refund(
                channel_tx_id=order.channel_tx_id,
                amount=order.amount_cny,
                idempotency_key=idempotency_key,  # 渠道退款幂等
            )
        elif step_index == 1:
            # 补偿 Step 2: AML 拒绝后退款
            return self.payment_service.refund(
                channel_tx_id=order.channel_tx_id,
                amount=order.amount_cny,
                idempotency_key=idempotency_key,
                reason="aml_rejected",
            )
        elif step_index == 2:
            # 补偿 Step 3: 退汇（外币换回人民币）
            forex_result = self._get_step_output(order.order_id, 2)
            return self.forex_service.reverse(
                conversion_id=forex_result["conversion_id"],
                idempotency_key=idempotency_key,
            )
        elif step_index == 3:
            # 补偿 Step 4: 通知商家退回款项（无法撤回）
            transfer_result = self._get_step_output(order.order_id, 3)
            return self.merchant_notify_service.request_return(
                merchant_id=order.merchant_id,
                order_id=order.order_id,
                amount=order.target_amount,
                currency=order.target_currency,
                transfer_reference=transfer_result["reference"],
                idempotency_key=idempotency_key,
            )
        elif step_index == 4:
            # 补偿 Step 5: 商家退款
            return self.merchant_notify_service.request_refund(
                merchant_id=order.merchant_id,
                order_id=order.order_id,
                idempotency_key=idempotency_key,
            )

    def _get_step_output(self, order_id: str, step_index: int) -> dict:
        """从 step_log 中获取某步骤的输出（用于补偿时获取前置步骤的结果）"""
        log = self.db.query_one(
            "SELECT output_data FROM saga_state_log "
            "WHERE order_id = %s AND step_index = %s "
            "AND direction = 'forward' AND status = 'completed' "
            "ORDER BY completed_at DESC LIMIT 1",
            [order_id, step_index]
        )
        if log:
            return json.loads(log.output_data)
        return {}

    def _wait_for_step_completion(
        self, saga_id: str, step_name: str, timeout: int
    ) -> StepResult:
        """等待另一个进程完成该步骤的执行"""
        start = now()
        while (now() - start).total_seconds() < timeout:
            log = self.db.query_one(
                "SELECT * FROM saga_state_log "
                "WHERE saga_id = %s AND step_name = %s "
                "AND direction = 'forward' "
                "AND status IN ('completed', 'failed') "
                "ORDER BY completed_at DESC LIMIT 1",
                [saga_id, step_name]
            )
            if log:
                if log.status == "completed":
                    return StepResult(
                        status="completed_by_another",
                        data=json.loads(log.output_data),
                        idempotency_key=log.idempotency_key,
                        was_recovered=True,
                    )
                else:
                    raise StepFailed(
                        f"Step {step_name} failed in another process: "
                        f"{log.error_message}"
                    )
            time.sleep(1)

        raise StepTimeout(
            f"Step {step_name} timed out waiting for another process "
            f"after {timeout}s"
        )


class StepResult:
    """步骤执行结果"""
    def __init__(self, status, data, idempotency_key, was_recovered=False):
        self.status = status
        self.data = data
        self.idempotency_key = idempotency_key
        self.was_recovered = was_recovered  # 是否是从已有日志恢复的


class IdempotentSagaOrchestrator:
    """使用幂等步骤执行器的完整 Saga 编排器"""

    def __init__(self):
        self.step_executor = IdempotentSagaStepExecutor()

    def execute(self, order):
        """执行完整的跨境支付 Saga"""
        saga_id = f"SAGA-{order.order_id}"

        # 恢复或创建 Saga 实例
        saga = self.db.get_saga_instance(saga_id)
        if not saga:
            saga = self.db.insert("saga_instances", {
                "saga_id": saga_id,
                "order_id": order.order_id,
                "saga_type": "cross_border_payment",
                "current_step": 0,
                "total_steps": 5,
                "state": "RUNNING",
                "direction": "forward",
                "context_data": json.dumps({"order_id": order.order_id}),
                "started_at": now(),
            })
        else:
            if saga.state == "COMPLETED":
                return saga  # 已完成，直接返回
            elif saga.state in ("COMPENSATING", "COMPENSATED"):
                # 需要继续补偿
                self._resume_compensation(saga_id, order, saga.current_step)
                return

        # 从 current_step 继续执行
        start_step = saga.current_step if saga else 0

        for step_index in range(start_step, 5):
            try:
                # 幂等执行：如果该步骤已完成，会自动跳过
                result = self.step_executor.execute_step(
                    saga_id, order, step_index
                )

                # 更新 Saga 实例的当前步骤
                self.db.update("saga_instances", {
                    "current_step": step_index + 1,  # 指向下一步
                    "updated_at": now(),
                }, {"saga_id": saga_id})

                # 将步骤结果存入上下文（后续步骤或补偿可能需要）
                if result.data:
                    self._update_saga_context(
                        saga_id, step_index, result.data
                    )

                # Step 1 特殊处理：AML 可能需要暂停
                if step_index == 1 and result.data.get("status") == "manual_review":
                    self.db.update("saga_instances", {
                        "state": "SUSPENDED",
                        "suspend_reason": "manual_review",
                        "suspend_ref_id": result.data["review_id"],
                    }, {"saga_id": saga_id})
                    return  # 等待审核回调

            except Exception as e:
                # 步骤失败 → 开始补偿
                self._start_compensation(saga_id, order, step_index - 1)
                raise

        # 全部完成
        self.db.update("saga_instances", {
            "state": "COMPLETED",
            "completed_at": now(),
        }, {"saga_id": saga_id})

    def _start_compensation(self, saga_id, order, failed_before_step):
        """从失败的步骤开始逆序补偿"""
        self.db.update("saga_instances", {
            "state": "COMPENSATING",
            "direction": "compensate",
        }, {"saga_id": saga_id})

        for step_index in range(failed_before_step, -1, -1):
            try:
                self.step_executor.compensate_step(
                    saga_id, order, step_index
                )
            except Exception as e:
                # 补偿也失败 → 记录并告警（需要人工介入）
                logger.error(
                    f"Compensation failed for step {step_index} "
                    f"in saga {saga_id}: {e}"
                )
                self.alert_team(
                    f"补偿失败！saga={saga_id}, step={step_index}, "
                    f"error={str(e)}。需要人工介入。"
                )
                raise

        self.db.update("saga_instances", {
            "state": "COMPENSATED",
            "completed_at": now(),
        }, {"saga_id": saga_id})

    def _update_saga_context(self, saga_id, step_index, data):
        """更新 Saga 上下文中的步骤输出"""
        saga = self.db.get_saga_instance(saga_id)
        context = json.loads(saga.context_data) if saga.context_data else {}
        context[f"step_{step_index}_output"] = data
        self.db.update("saga_instances", {
            "context_data": json.dumps(context),
        }, {"saga_id": saga_id})

    def resume_after_review(self, saga_id: str, review_result: str):
        """人工审核完成后恢复 Saga"""
        saga = self.db.get_saga_instance(saga_id)
        order = self.db.get_payment_order(saga.order_id)

        if review_result == "approved":
            self.db.update("saga_instances", {
                "state": "RUNNING",
                "suspend_reason": None,
            }, {"saga_id": saga_id})
            # 从 Step 2（index=2）继续
            self.execute(order)
        else:
            # 审核拒绝 → 退款
            self._start_compensation(saga_id, order, 0)
```

**幂等键设计总结：**

| 步骤 | 幂等键格式 | 外部系统幂等保证 | 补偿幂等键格式 |
|------|-----------|---------------|--------------|
| 用户支付 | `pay-{order_id}` | 支付宝/微信 order_id 去重 | `refund-{order_id}` |
| AML 筛查 | `aml-{order_id}` | AML 供应商 request_id 去重 | `aml-refund-{order_id}` |
| 汇率购汇 | `forex-{order_id}` | 外汇供应商 trade_id 去重 | `forex-rev-{order_id}` |
| 跨境汇款 | `transfer-{order_id}` | SWIRT instruction_id 去重 | `transfer-ret-{order_id}` |
| 商家结算 | `settle-{order_id}` | 结算系统 settlement_id 去重 | `settle-refund-{order_id}` |

**关键保障机制：**

1. **step_log 是唯一真相来源**：每个步骤执行前必须查询 `saga_state_log` 表，`status=completed` 的记录表示该步骤已成功完成，可直接跳过
2. **占位防并发**：步骤开始时先写入 `status=started` 记录，其他进程看到后等待而非重复执行
3. **幂等键三重保证**：应用层（step_log 查询） + 数据库层（idempotency_key 唯一索引） + 外部系统层（渠道/银行幂等）
4. **补偿同样幂等**：补偿操作也有独立的 step_log 记录和幂等键，崩溃恢复时不会重复退款

## 常见陷阱（深度分析）

### 陷阱 1：用下单时汇率但不锁定

**后果：** 用户下单时 CNY/USD = 7.20，2天后结算时 = 7.30。如果按下单时汇率，平台每笔亏 1.4%。日均 $2000 万 × 1.4% = $28 万/天的汇率损失。

### 陷阱 2：跨境汇款失败后简单退款

**问题：** 跨境汇款可能"半成功"——SWIFT 消息已发出但银行处理失败。此时资金在途中，既不在用户账户也不在商家账户。简单退款可能退一笔不存在的钱。

**正确做法：** 等待银行明确返回失败（SWIFT MT942 退回通知），确认资金回到平台账户后再退款给用户。

### 陷阱 3：不检查年度购汇额度

**后果：** 用户年度购汇超过 $50000 的监管限额 → 外管局处罚 → 平台被暂停跨境业务。

### 陷阱 4：对账只看总额不看逐笔

**问题：** 内部总额 = 渠道总额，但逐笔不一致：
- 内部记录：交易 A $100，交易 B $200
- 渠道记录：交易 A $200，交易 B $100
- 总额都是 $300，但交易 A 多付了 $100

**必须逐笔比对，不能只看总额。**

## 延伸思考

- **多级结算**：如果平台不是直接付给商家，而是先付给区域结算中心再付给商家，增加了一个中间环节。如何保证整个链条的一致性？
- **数字货币结算**：如果用 USDC/USDT 做跨境结算，省去了传统银行的 SWIFT 通道，但引入了链上交易的不确定性。架构如何调整？
- **汇率对冲**：平台如何通过远期合约或期权对冲汇率敞口，降低汇率波动成本？
## 性能与成本分析

**系统规模估算（日均 100 万笔、$2000 万交易额）：**

| 组件 | 规格 | 数量 | 月成本 |
|------|------|------|-------|
| 支付网关 | 8c16G | 10 台 | ¥5 万 |
| Saga 编排器 | 4c8G | 5 台 | ¥1 万 |
| AML 筛查服务 | 4c8G + GPU | 3 台 | ¥2 万 |
| MySQL（订单+结算） | 8c64G | 主从 4 台 | ¥4 万 |
| Redis（汇率锁+幂等） | 8c32G | 3 节点 | ¥1.5 万 |
| Kafka | 8c32G + 1TB | 6 节点 | ¥3 万 |
| **合计** | | | **¥16.5 万** |

**汇率敞口成本：**
- 日均交易额 $2000 万 × 30 分钟内平均波动 0.03% = $6000/天
- 月敞口成本约 $18 万（¥130 万）→ 需用远期合约对冲降低 50%

### 每步骤延迟分解

**一笔完整的跨境支付（无人工审核）的端到端延迟拆解：**

```
总延迟（P50）: 15.8s    总延迟（P99）: 45.2s
==========================================

Step 1: 用户支付        ██████░░░░░░░░░░░░  P50: 2.5s   P99: 8.0s
  ├─ 渠道 API 调用       █████░░░░░░░░░░░░  1.8s        6.0s
  ├─ 幂等检查+DB写入     █░░░░░░░░░░░░░░░  0.3s        0.5s
  └─ 渠道回调等待        █░░░░░░░░░░░░░░░  0.4s        1.5s

Step 2: AML 筛查        ██████████░░░░░░░  P50: 5.0s   P99: 180s
  ├─ 制裁名单匹配        ████████░░░░░░░░  3.5s        5.0s
  ├─ 额度检查+冻结       █░░░░░░░░░░░░░░░  0.3s        0.5s
  ├─ 频率异常检查        █░░░░░░░░░░░░░░░  0.2s        0.3s
  └─ 商品合规检查        ░░░░░░░░░░░░░░░░  1.0s        174.2s (人工审核)

Step 3: 汇率锁定+购汇   ████░░░░░░░░░░░░░  P50: 1.8s   P99: 4.0s
  ├─ 汇率查询+锁定       ██░░░░░░░░░░░░░░  0.5s        1.0s
  ├─ 外汇供应商 API       █░░░░░░░░░░░░░░░  1.0s        2.5s
  └─ 额度 consume        ░░░░░░░░░░░░░░░░  0.3s        0.5s

Step 4: 跨境汇款        ████████████████░  P50: 5.0s   P99: 25.0s
  ├─ SWIFT 报文生成       █░░░░░░░░░░░░░░░  0.3s        0.5s
  ├─ SWIFT 网络提交       █████████████░░░  3.0s        15.0s
  ├─ ACK 等待            █░░░░░░░░░░░░░░░  1.5s        9.0s
  └─ DB 状态更新          ░░░░░░░░░░░░░░░░  0.2s        0.5s

Step 5: 结算调度        █░░░░░░░░░░░░░░░░  P50: 1.5s   P99: 3.0s
  ├─ 结算记录创建         ░░░░░░░░░░░░░░░░  0.3s        0.5s
  ├─ 批次分配             ░░░░░░░░░░░░░░░░  0.2s        0.5s
  └─ DB写入               ░░░░░░░░░░░░░░░░  1.0s        2.0s
```

**延迟瓶颈分析：**

| 瓶颈步骤 | 原因 | 优化方案 | 预期改善 |
|---------|------|---------|---------|
| AML 筛查 (P99=180s) | 人工审核 | 优化规则减少假阳性 → 1%；增加白名单 | P99: 180s → 10s |
| SWIFT 等待 (P99=25s) | SWIFT 网络延迟 | 迁移到 SWIFT gpi（跟踪+加速） | P99: 25s → 5s |
| 渠道 API (P99=6s) | 支付宝/微信接口慢 | 预连接 + 连接池 + 超时优化 | P99: 6s → 3s |
| 外汇 API (P99=2.5s) | 供应商接口波动 | 多供应商热备 + 本地汇率缓存 | P99: 2.5s → 1.5s |

### 每笔交易成本分析

**单笔跨境支付的成本构成（以 ¥5000 / $694 的交易为例）：**

| 成本项 | 金额（¥） | 金额（$） | 占比 | 承担方 | 说明 |
|-------|----------|----------|------|-------|------|
| 支付渠道手续费 | 12.50 | 1.74 | 20% | 商家 | 支付宝/微信 0.25% |
| AML 筛查费 | 0.50 | 0.07 | 1% | 平台 | 每次筛查成本约 ¥0.5 |
| 外汇价差 | 35.00 | 4.86 | 56% | 平台收入 | 0.7% 价差（含对冲成本） |
| 外汇对冲成本 | 5.00 | 0.69 | 8% | 平台 | 远期合约成本 |
| SWIFT 汇款费 | 7.50 | 1.04 | 12% | 商家 | 银行收取（批量摊薄后） |
| 系统基础设施 | 0.55 | 0.08 | 1% | 平台 | ¥16.5 万/月 ÷ 3000 万笔/月 |
| DB + Redis | 0.37 | 0.05 | 1% | 平台 | ¥11 万/月 ÷ 3000 万笔/月 |
| **合计** | **61.42** | **8.53** | **100%** | | |
| **平台净收入** | **24.50** | **3.40** | | | 价差 ¥35 - 对冲 ¥5 - AML ¥0.5 - 基础设施 ¥5 |
| **商家成本** | **20.00** | **2.78** | | | 渠道费 ¥12.5 + 汇款费 ¥7.5 |
| **用户成本** | **0** | **0** | | | 用户无额外费用 |

**规模效应与成本趋势：**

| 交易量（日） | 单笔基础设施成本 | 单笔AML成本 | 批量汇款费/笔 | 单笔总成本 | 单笔利润 |
|------------|----------------|------------|-------------|----------|---------|
| 10 万笔 | ¥0.55 | ¥0.50 | ¥15.00 | ¥18.55 | ¥16.45 |
| 50 万笔 | ¥0.11 | ¥0.50 | ¥3.00 | ¥5.11 | ¥29.89 |
| 100 万笔 | ¥0.055 | ¥0.50 | ¥1.50 | ¥3.06 | ¥31.94 |
| 500 万笔 | ¥0.011 | ¥0.50 | ¥0.30 | ¥1.31 | ¥33.69 |

**关键性能指标：**

| 指标 | 目标 | 实际 |
|------|------|------|
| 支付处理延迟（无人工审核） | < 30s | 15.8s |
| 支付处理延迟（P99，无人工审核） | < 60s | 45.2s |
| AML 自动筛查延迟 | < 5min | 5.0s |
| AML 人工审核延迟 | < 2h | 30-90min |
| 汇率锁定时效 | 30min | Redis SETEX 保证 |
| SWIFT 提交到 ACK | < 30s | P50: 5s, P99: 25s |
| 批量结算吞吐 | > 5000 批/日 | 实际 2000-5000 批 |
| 三方对账准确率 | 99.99% | 逐笔比对 + 自动告警 |
| 幂等防重复率 | 100% | step_log + 唯一索引保证 |
| 对账差异自动解决率 | > 80% | 目标 85% |
| 单笔利润率 | > 0.5% | 0.49%（价差 0.7% - 成本 0.21%） |

**交易量分析：**

| 渠道 | 日均笔数 | 平均金额 | 峰值 TPS |
|------|---------|---------|---------|
| 支付宝 | 40 万 | ¥500 | 200 |
| 微信 | 30 万 | ¥300 | 150 |
| 银联 | 20 万 | ¥1000 | 80 |
| PayPal | 10 万 | $100 | 50 |


## 跨境支付路由优化

```python
class PaymentRouteOptimizer:
    """支付路由优化：选择最优支付渠道"""

    def select_channel(self, source_currency, target_currency, amount):
        """选择最优支付渠道"""
        channels = self.db.query(
            "SELECT * FROM payment_channels WHERE source_currency = %s "
            "AND target_currency = %s AND enabled = 1", source_currency, target_currency)

        scored = []
        for ch in channels:
            # 评分维度：费率、速度、成功率、额度
            fee_score = self._score_fee(ch, amount)
            speed_score = self._score_speed(ch)
            success_score = self._score_success_rate(ch)
            limit_score = self._score_limit(ch, amount)

            total = (fee_score * 0.4 + speed_score * 0.2 +
                     success_score * 0.3 + limit_score * 0.1)
            scored.append({"channel": ch, "score": total})

        # 返回得分最高的渠道
        best = max(scored, key=lambda x: x["score"])
        return best["channel"]

    def _score_fee(self, channel, amount):
        """费率评分：总费用越低分越高"""
        total_fee = channel["fixed_fee"] + amount * channel["percentage_fee"]
        # 汇率加价
        fx_markup = amount * channel["fx_markup"]
        total_cost = total_fee + fx_markup
        # 0-100 分：0 成本=100分，5%成本=0分
        cost_rate = total_cost / amount
        return max(0, 100 - cost_rate * 2000)

    def _score_success_rate(self, channel):
        """成功率评分：基于最近 30 天数据"""
        rate = self.db.query_one(
            "SELECT COUNT(CASE WHEN status='success' THEN 1 END) * 100.0 / COUNT(*) "
            "FROM payment_transactions WHERE channel_id = %s "
            "AND created_at > NOW() - INTERVAL 30 DAY", channel["id"])["rate"]
        return rate  # 0-100
```

## 合规检查完整实现

```python
class ComplianceCheckService:
    """跨境合规检查"""

    def check_transaction(self, transaction):
        """执行全量合规检查"""
        checks = []

        # 1. AML 反洗钱检查
        aml_result = self.aml_screen(transaction)
        checks.append({"check": "aml", "result": aml_result})

        # 2. 制裁名单检查
        sanction_result = self.sanction_screen(transaction)
        checks.append({"check": "sanction", "result": sanction_result})

        # 3. 外汇额度检查
        forex_result = self.forex_quota_check(transaction)
        checks.append({"check": "forex_quota", "result": forex_result})

        # 4. 金额阈值检查
        threshold_result = self.threshold_check(transaction)
        checks.append({"check": "threshold", "result": threshold_result})

        # 任何检查不通过 → 阻止交易
        blocked = [c for c in checks if c["result"]["status"] == "blocked"]
        if blocked:
            return {"status": "blocked", "reasons": blocked}

        # 需要人工审核
        review = [c for c in checks if c["result"]["status"] == "review"]
        if review:
            return {"status": "review", "reasons": review}

        return {"status": "approved"}

    def sanction_screen(self, transaction):
        """制裁名单筛查"""
        # 检查收款人是否在 OFAC / EU / UN 制裁名单
        sender_match = self.sanction_db.search(transaction.sender_name, transaction.sender_country)
        receiver_match = self.sanction_db.search(transaction.receiver_name, transaction.receiver_country)

        if sender_match or receiver_match:
            return {"status": "blocked",
                    "reason": "制裁名单匹配",
                    "matches": sender_match or receiver_match}
        return {"status": "approved"}

    def threshold_check(self, transaction):
        """金额阈值检查"""
        # 中国：个人年度 5 万美元购汇额度
        if transaction.source_country == "CN":
            yearly_total = self.db.query_one(
                "SELECT COALESCE(SUM(amount_usd), 0) as total "
                "FROM cross_border_transactions "
                "WHERE sender_id = %s AND YEAR(created_at) = YEAR(NOW())",
                transaction.sender_id)["total"]

            if yearly_total + transaction.amount_usd > 50000:
                return {"status": "review",
                        "reason": f"年度购汇额度接近上限: {yearly_total + transaction.amount_usd}/50000 USD"}
        return {"status": "approved"}
```

## 异常场景补充

### 场景：支付渠道中断切换

```python
class ChannelFailoverService:
    """支付渠道故障切换"""
    def handle_channel_failure(self, channel_id):
        """处理渠道故障"""
        # 1. 标记渠道不可用
        self.db.update("payment_channels",
            {"status": "unavailable", "down_since": now()},
            {"id": channel_id})

        # 2. 将待处理交易路由到备用渠道
        pending = self.db.query(
            "SELECT * FROM payment_transactions "
            "WHERE channel_id = %s AND status = 'pending'", channel_id)
        for tx in pending:
            alt_channel = self.route_optimizer.select_channel(
                tx.source_currency, tx.target_currency, tx.amount)
            self.db.update("payment_transactions",
                {"channel_id": alt_channel.id, "routed_at": now()},
                {"id": tx.id})

        # 3. 告警通知
        self.alert(f"支付渠道 {channel_id} 中断，{len(pending)} 笔交易已重路由")
```

### 场景：汇率剧烈波动

```
触发：GBP/USD 突然波动 5%（英国脱欧公投）
      → 锁定的汇率与市场汇率差异巨大
检测：
  1. 汇率变动 > 2%/小时 → 告警
  2. 汇率变动 > 5%/小时 → 紧急告警
处理：
  1. 已锁定汇率但未结算的交易 → 按锁定汇率执行
  2. 新交易 → 暂停该币对，等待汇率稳定
  3. 平台汇兑损失 > 日限额 → 暂停所有外汇交易
预防：汇率锁定有效期 10 分钟 + 单日损失上限
```

### 场景：重复退款

```
触发：用户退款请求被处理两次 → 双倍退款
检测：
  1. 退款幂等键：original_transaction_id + refund_type
  2. 退款前查询是否已退款
处理：
  1. 退款前检查 refund_records 表是否已有该笔
  2. 已退款 → 拒绝重复退款
  3. 并发退款 → 数据库 UNIQUE 约束阻止
预防：退款幂等键 + 数据库唯一约束
```

## FX 汇率管理完整实现

```python
class FXRateManager:
    """实时汇率管理"""

    def __init__(self):
        self.rate_cache = {}  # currency_pair → (rate, timestamp)
        self.lock_ttl = 600  # 锁定 10 分钟

    def get_rate(self, source_currency, target_currency):
        """获取实时汇率"""
        pair = f"{source_currency}/{target_currency}"

        # 1. 检查缓存
        cached = self.redis.get(f"fx_rate:{pair}")
        if cached:
            rate_data = json.loads(cached)
            if (now() - datetime.fromisoformat(rate_data["timestamp"])).seconds < 10:
                return rate_data["rate"]

        # 2. 从实时汇率源获取
        try:
            rate = self.fx_provider.get_live_rate(source_currency, target_currency)
            self.redis.setex(f"fx_rate:{pair}", 10, json.dumps({
                "rate": rate, "timestamp": now().isoformat()
            }))
            return rate
        except FXFeedError:
            # 3. 实时源失败 → 使用昨日收盘价
            fallback = self.db.query_one(
                "SELECT close_rate FROM fx_rate_history "
                "WHERE pair = %s ORDER BY date DESC LIMIT 1", pair)
            if fallback:
                self.alert(f"FX 实时源失败，使用昨日收盘价: {pair}")
                return fallback["close_rate"]
            raise FXRateUnavailableError(f"无法获取汇率: {pair}")

    def lock_rate(self, transaction_id, source_currency, target_currency):
        """锁定汇率（10 分钟有效期）"""
        rate = self.get_rate(source_currency, target_currency)
        lock_key = f"fx_lock:{transaction_id}"

        self.redis.setex(lock_key, self.lock_ttl, json.dumps({
            "pair": f"{source_currency}/{target_currency}",
            "rate": rate,
            "locked_at": now().isoformat(),
            "expire_at": (now() + timedelta(seconds=self.lock_ttl)).isoformat()
        }))

        return {"rate": rate, "expires_in_seconds": self.lock_ttl}

    def get_locked_rate(self, transaction_id):
        """获取已锁定的汇率"""
        lock_key = f"fx_lock:{transaction_id}"
        locked = self.redis.get(lock_key)
        if not locked:
            raise FXRateExpiredError(f"汇率锁定已过期: {transaction_id}")
        return json.loads(locked)
```

## 支付状态追踪

```python
class PaymentStatusTracker:
    """跨系统支付状态追踪"""

    STATUS_MAP = {
        # gateway_status → internal_status
        "PENDING": "processing",
        "AUTHORIZED": "authorized",
        "CAPTURED": "captured",
        "SETTLED": "settled",
        "FAILED": "failed",
        "REFUNDED": "refunded",
        "CANCELLED": "cancelled",
    }

    def get_aggregated_status(self, transaction_id):
        """聚合多系统状态"""
        # 1. 支付网关状态
        gateway_status = self.gateway.query_status(transaction_id)
        # 2. 银行状态
        bank_status = self.bank_client.query_status(transaction_id)
        # 3. 合规状态
        compliance_status = self.db.get_compliance_status(transaction_id)

        # 综合判定：取最保守的状态
        internal_status = self.STATUS_MAP.get(gateway_status, "unknown")

        return {
            "transaction_id": transaction_id,
            "gateway_status": gateway_status,
            "bank_status": bank_status,
            "compliance_status": compliance_status,
            "internal_status": internal_status,
            "is_terminal": internal_status in ("settled", "failed", "refunded", "cancelled")
        }

    def on_status_change(self, transaction_id, new_status):
        """状态变更通知"""
        # 1. 更新内部状态
        self.db.update("payment_transactions",
            {"status": self.STATUS_MAP[new_status], "updated_at": now()},
            {"id": transaction_id})

        # 2. 通知商户
        transaction = self.db.get_transaction(transaction_id)
        self.webhook_client.notify(transaction["merchant_id"], {
            "event": "payment_status_changed",
            "transaction_id": transaction_id,
            "new_status": self.STATUS_MAP[new_status],
            "timestamp": now().isoformat()
        })
```

## 异常场景补充

### 场景：FX 汇率源断连

```
触发：实时汇率 WebSocket 断开 → 无法获取最新汇率
检测：
  1. WebSocket 心跳超时 → 告警
  2. 最后更新时间 > 30 秒 → 降级告警
处理：
  1. 自动切换到备用汇率源
  2. 无备用 → 使用 Redis 缓存汇率（TTL 10 秒已过期）
  3. 使用昨日收盘价作为兜底
  4. 新交易暂停汇率锁定 → 等待恢复
预防：多汇率源 + 自动故障切换 + 缓存兜底
```

### 场景：结算批次部分失败

```python
class SettlementBatchProcessor:
    """结算批次处理"""
    def process_batch(self, batch_id):
        transactions = self.db.get_batch_transactions(batch_id)
        for tx in transactions:
            try:
                self.bank_client.settle(tx)
                self.db.update("payment_transactions",
                    {"status": "settled"}, {"id": tx.id})
            except Exception as e:
                # 部分失败不影响其他交易
                self.db.update("payment_transactions",
                    {"status": "settlement_failed", "error": str(e)},
                    {"id": tx.id})
                self.db.insert("settlement_retry_queue", {
                    "transaction_id": tx.id, "error": str(e),
                    "retry_count": 0, "next_retry_at": now() + timedelta(hours=1)
                })
```

### 场景：合规规则更新阻断交易

```
触发：新制裁名单生效 → 在途交易被阻断
检测：
  1. 合规检查返回 "blocked" → 告警
  2. 在途交易状态变为 "compliance_blocked"
处理：
  1. 立即停止该交易的处理
  2. 通知商户交易被阻断
  3. 已结算交易 → 退款处理
  4. 人工审核是否为误拦截
预防：合规规则灰度生效 + 误拦截快速解除机制
```

## 批量结算完整实现

```python
class SettlementBatchService:
    """跨境支付批量结算"""

    def settle_batch(self, date):
        """按币对分组批量结算"""
        # 按源币+目标币分组
        groups = self.db.query(
            "SELECT source_currency, target_currency, "
            "COUNT(*) as tx_count, SUM(amount) as total_amount "
            "FROM payment_transactions "
            "WHERE status = 'captured' AND DATE(captured_at) = %s "
            "GROUP BY source_currency, target_currency", date)

        settlement_results = []
        for group in groups:
            # 净额计算：同币对双向交易抵消
            net = self._calc_net_settlement(group["source_currency"],
                group["target_currency"], date)

            if abs(net["net_amount"]) < 0.01:
                continue  # 完全抵消，无需结算

            # 调用银行结算 API
            try:
                result = self.bank_client.settle({
                    "source_currency": net["settlement_currency"],
                    "target_currency": net["receiving_currency"],
                    "amount": abs(net["net_amount"]),
                    "direction": "pay" if net["net_amount"] > 0 else "receive",
                    "value_date": date,
                })
                # 标记所有交易为已结算
                self.db.execute(
                    "UPDATE payment_transactions SET status = 'settled', "
                    "settled_at = NOW(), settlement_id = %s "
                    "WHERE source_currency = %s AND target_currency = %s "
                    "AND status = 'captured' AND DATE(captured_at) = %s",
                    result["settlement_id"],
                    group["source_currency"], group["target_currency"], date)

                settlement_results.append({
                    "pair": f"{group['source_currency']}/{group['target_currency']}",
                    "tx_count": group["tx_count"],
                    "net_amount": net["net_amount"],
                    "settlement_id": result["settlement_id"],
                    "status": "settled"
                })
            except Exception as e:
                settlement_results.append({
                    "pair": f"{group['source_currency']}/{group['target_currency']}",
                    "status": "failed", "error": str(e)
                })

        return settlement_results

    def _calc_net_settlement(self, source_ccy, target_ccy, date):
        """净额计算：双向抵消"""
        outgoing = self.db.query_one(
            "SELECT COALESCE(SUM(amount), 0) as total "
            "FROM payment_transactions WHERE source_currency = %s "
            "AND target_currency = %s AND status = 'captured' "
            "AND DATE(captured_at) = %s", source_ccy, target_ccy, date)
        incoming = self.db.query_one(
            "SELECT COALESCE(SUM(amount), 0) as total "
            "FROM payment_transactions WHERE source_currency = %s "
            "AND target_currency = %s AND status = 'captured' "
            "AND DATE(captured_at) = %s", target_ccy, source_ccy, date)

        net_amount = outgoing["total"] - incoming["total"]
        return {
            "net_amount": net_amount,
            "settlement_currency": source_ccy if net_amount > 0 else target_ccy,
            "receiving_currency": target_ccy if net_amount > 0 else source_ccy,
        }
```

## 性能分析详细数据

**支付处理性能：**

| 操作 | TPS | 延迟 P99 | 说明 |
|------|-----|---------|------|
| 支付创建 | 500 | 500ms | 含合规检查+FX锁定 |
| 状态查询 | 5000 | 10ms | Redis 缓存 |
| 退款处理 | 100 | 1s | 含银行退款API |
| 批量结算 | 10 批/天 | 5min/批 | 含净额计算 |
| 对账 | 1 次/天 | 30min | 含银行API |

**月度成本：**

| 组件 | 规格 | 月成本 |
|------|------|-------|
| 应用服务器 | 8c32G × 5台 | ¥3 万 |
| MySQL | 8c64G + SSD × 4台 | ¥4 万 |
| Redis | 8c32G × 3节点 | ¥1.5 万 |
| Kafka | 6节点 | ¥3 万 |
| **合计** | | **¥11.5 万** |

## 汇率实时管理与锁定系统

跨境支付对汇率的实时性和准确性要求极高。汇率数据不仅需要实时更新，还需要支持锁定、缓存、降级等多种策略，确保在任何情况下交易都能以合理的汇率完成。

### FXRateProvider：实时汇率订阅与分发

```python
class FXRateProvider:
    """
    实时汇率数据提供者。
    通过 WebSocket 连接外汇数据供应商（如 Reuters、Bloomberg），
    实时接收汇率变动推送，并分发到 Redis 缓存供业务系统使用。

    设计原则：
    1. 多数据源冗余：主源（Reuters）+ 备源（Bloomberg），自动切换
    2. 本地缓存：Redis 10 秒 TTL，减少对外部 API 的依赖
    3. 数据校验：同一币对从不同源获取的汇率差异 > 0.5% → 告警
    4. 优雅降级：主源断线 → 备源 → 昨日收盘价 → 人工介入
    """

    # 支持的币对及其精度要求
    CURRENCY_PAIRS = {
        "CNY/USD": {"precision": 6, "max_spread_bps": 50},
        "CNY/EUR": {"precision": 6, "max_spread_bps": 60},
        "CNY/JPY": {"precision": 4, "max_spread_bps": 80},
        "CNY/GBP": {"precision": 6, "max_spread_bps": 70},
        "CNY/AUD": {"precision": 6, "max_spread_bps": 80},
        "EUR/USD": {"precision": 6, "max_spread_bps": 30},
        "GBP/USD": {"precision": 6, "max_spread_bps": 40},
    }

    # Redis 缓存配置
    RATE_CACHE_TTL_SECONDS = 10      # 汇率缓存有效期 10 秒
    RATE_LOCK_TTL_SECONDS = 600      # 汇率锁定有效期 10 分钟（600 秒）
    YESTERDAY_CLOSE_CACHE_TTL = 86400 # 昨日收盘价缓存 24 小时

    def __init__(self, redis_client, db_client):
        self.redis = redis_client
        self.db = db_client
        self.primary_ws = None        # 主数据源 WebSocket 连接
        self.secondary_ws = None      # 备数据源 WebSocket 连接
        self._running = False
        self._last_primary_rate = {}  # 主源最近一次汇率
        self._last_secondary_rate = {}# 备源最近一次汇率
        self._rate_update_count = 0   # 汇率更新计数（用于统计）
        self._alert_cooldown = {}     # 告警冷却（防止重复告警）

    def start(self):
        """启动汇率实时订阅"""
        self._running = True
        # 启动主源 WebSocket
        self._connect_primary()
        # 启动备源 WebSocket
        self._connect_secondary()
        # 启动健康检查线程
        threading.Thread(target=self._health_check_loop, daemon=True).start()
        # 启动昨日收盘价加载
        self._load_yesterday_close_rates()

    def stop(self):
        """停止汇率订阅"""
        self._running = False
        if self.primary_ws:
            self.primary_ws.close()
        if self.secondary_ws:
            self.secondary_ws.close()

    def get_rate(self, from_currency: str, to_currency: str) -> Decimal:
        """
        获取实时汇率（优先从 Redis 缓存读取）。
        降级顺序：Redis 缓存 → 备源缓存 → 昨日收盘价。
        """
        pair_key = f"{from_currency}/{to_currency}"
        reverse_key = f"{to_currency}/{from_currency}"

        # 1. 尝试从 Redis 缓存获取
        cached = self.redis.get(f"fx_rate:{pair_key}")
        if cached:
            rate_data = json.loads(cached)
            return Decimal(str(rate_data["rate"]))

        # 2. 尝试反向汇率
        cached_reverse = self.redis.get(f"fx_rate:{reverse_key}")
        if cached_reverse:
            rate_data = json.loads(cached_reverse)
            return Decimal("1") / Decimal(str(rate_data["rate"]))

        # 3. 降级：使用昨日收盘价
        yesterday_rate = self.redis.get(f"fx_rate_yesterday:{pair_key}")
        if yesterday_rate:
            rate_data = json.loads(yesterday_rate)
            self._alert_rate_fallback(pair_key, "yesterday_close")
            return Decimal(str(rate_data["rate"]))

        # 4. 最终降级：从数据库读取最近一条汇率记录
        last_known = self.db.query_one(
            "SELECT rate FROM fx_rate_history "
            "WHERE from_currency = %s AND to_currency = %s "
            "ORDER BY timestamp DESC LIMIT 1",
            [from_currency, to_currency]
        )
        if last_known:
            self._alert_rate_fallback(pair_key, "last_known_db")
            return Decimal(str(last_known["rate"]))

        # 5. 无任何汇率数据 → 严重告警，拒绝交易
        self._alert_rate_fallback(pair_key, "no_rate_available")
        raise RateUnavailableError(f"无法获取 {pair_key} 的汇率数据")

    def lock_rate(self, from_currency: str, to_currency: str,
                  amount: Decimal, order_id: str) -> dict:
        """
        为一笔交易锁定汇率，有效期 10 分钟。
        锁定期间即使市场汇率波动，该交易仍使用锁定汇率。
        锁定信息写入 Redis（快速读写）和数据库（持久化）。

        Returns:
            {
                "lock_id": "lock-xxx",
                "from_currency": "CNY",
                "to_currency": "USD",
                "locked_rate": 7.200000,
                "amount": 5000.00,
                "converted_amount": 694.44,
                "locked_at": "2024-05-07T14:00:00Z",
                "expires_at": "2024-05-07T14:10:00Z",
                "status": "locked"
            }
        """
        lock_id = f"lock-{order_id}"
        current_rate = self.get_rate(from_currency, to_currency)
        pair_config = self.CURRENCY_PAIRS.get(
            f"{from_currency}/{to_currency}", {"precision": 6}
        )
        precision = pair_config["precision"]

        converted_amount = (amount / current_rate).quantize(
            Decimal("0.01"), rounding=ROUND_HALF_UP
        )

        lock_data = {
            "lock_id": lock_id,
            "order_id": order_id,
            "from_currency": from_currency,
            "to_currency": to_currency,
            "locked_rate": float(current_rate),
            "amount": float(amount),
            "converted_amount": float(converted_amount),
            "locked_at": now().isoformat(),
            "expires_at": (now() + timedelta(seconds=self.RATE_LOCK_TTL_SECONDS)).isoformat(),
            "status": "locked",
        }

        # Redis 锁定（快速校验）
        self.redis.setex(
            f"rate_lock:{lock_id}",
            self.RATE_LOCK_TTL_SECONDS,
            json.dumps(lock_data)
        )

        # 数据库持久化（崩溃恢复）
        self.db.insert("forex_locks", {
            "lock_id": lock_id,
            "order_id": order_id,
            "from_currency": from_currency,
            "to_currency": to_currency,
            "rate": current_rate,
            "amount": amount,
            "converted_amount": converted_amount,
            "status": "locked",
            "locked_at": now(),
            "expires_at": now() + timedelta(seconds=self.RATE_LOCK_TTL_SECONDS),
        })

        return lock_data

    def execute_locked_rate(self, lock_id: str) -> dict:
        """执行已锁定的汇率兑换"""
        lock_json = self.redis.get(f"rate_lock:{lock_id}")
        if not lock_json:
            # Redis 无记录 → 从数据库恢复
            lock_record = self.db.query_one(
                "SELECT * FROM forex_locks WHERE lock_id = %s", [lock_id]
            )
            if not lock_record:
                raise LockNotFoundError(f"汇率锁定 {lock_id} 不存在")
            if lock_record["status"] != "locked":
                raise LockAlreadyExecutedError(
                    f"汇率锁定 {lock_id} 状态为 {lock_record['status']}"
                )
            if now() > lock_record["expires_at"]:
                raise LockExpiredError(f"汇率锁定 {lock_id} 已过期")
            lock_data = {
                "lock_id": lock_record["lock_id"],
                "from_currency": lock_record["from_currency"],
                "to_currency": lock_record["to_currency"],
                "locked_rate": float(lock_record["rate"]),
                "amount": float(lock_record["amount"]),
                "converted_amount": float(lock_record["converted_amount"]),
            }
        else:
            lock_data = json.loads(lock_json)

        # 调用外汇供应商执行购汇
        result = self.forex_provider.convert(
            from_currency=lock_data["from_currency"],
            to_currency=lock_data["to_currency"],
            amount=Decimal(str(lock_data["amount"])),
            rate=Decimal(str(lock_data["locked_rate"])),
        )

        # 更新锁定状态
        self.db.update("forex_locks", {
            "status": "executed",
            "executed_at": now(),
            "actual_rate": Decimal(str(lock_data["locked_rate"])),
            "provider_tx_id": result.get("transaction_id"),
        }, {"lock_id": lock_id})

        # 删除 Redis 锁定键
        self.redis.delete(f"rate_lock:{lock_id}")

        return {"status": "executed", "result": result}

    # ---- WebSocket 连接管理 ----

    def _connect_primary(self):
        """连接主数据源 WebSocket（Reuters）"""
        def on_message(ws, message):
            rate_update = json.loads(message)
            self._handle_rate_update(rate_update, source="primary")

        def on_error(ws, error):
            logger.error(f"主数据源 WebSocket 错误: {error}")
            self._alert_ws_disconnect("primary", str(error))

        def on_close(ws, close_status, close_msg):
            logger.warning(f"主数据源 WebSocket 关闭: {close_status} {close_msg}")
            if self._running:
                # 5 秒后自动重连，指数退避
                delay = min(5 * (2 ** self._primary_reconnect_count), 60)
                self._primary_reconnect_count += 1
                threading.Timer(delay, self._connect_primary).start()

        def on_open(ws):
            logger.info("主数据源 WebSocket 连接成功")
            self._primary_reconnect_count = 0

        self.primary_ws = websocket.WebSocketApp(
            "wss://fx-api.reuters.com/v2/stream",
            on_message=on_message,
            on_error=on_error,
            on_close=on_close,
            on_open=on_open,
            header={"Authorization": f"Bearer {self.primary_api_key}"}
        )
        threading.Thread(target=self.primary_ws.run_forever, daemon=True).start()

    def _connect_secondary(self):
        """连接备数据源 WebSocket（Bloomberg）"""
        # 结构与主源相同，独立连接，确保主备完全解耦
        def on_message(ws, message):
            rate_update = json.loads(message)
            self._handle_rate_update(rate_update, source="secondary")

        def on_error(ws, error):
            logger.error(f"备数据源 WebSocket 错误: {error}")

        def on_close(ws, close_status, close_msg):
            if self._running:
                threading.Timer(10.0, self._connect_secondary).start()

        self.secondary_ws = websocket.WebSocketApp(
            "wss://fx-api.bloomberg.com/v2/stream",
            on_message=on_message,
            on_error=on_error,
            on_close=on_close,
            header={"Authorization": f"Bearer {self.secondary_api_key}"}
        )
        threading.Thread(target=self.secondary_ws.run_forever, daemon=True).start()

    def _handle_rate_update(self, rate_update: dict, source: str):
        """处理汇率更新推送，写入 Redis 缓存"""
        pair = rate_update["pair"]  # 如 "CNY/USD"
        rate = Decimal(str(rate_update["rate"]))
        timestamp = rate_update.get("timestamp", now().isoformat())

        # 写入 Redis 缓存（10 秒 TTL）
        cache_data = {
            "rate": float(rate),
            "source": source,
            "timestamp": timestamp,
        }
        self.redis.setex(
            f"fx_rate:{pair}",
            self.RATE_CACHE_TTL_SECONDS,
            json.dumps(cache_data)
        )

        # 记录最近汇率（用于数据校验）
        if source == "primary":
            self._last_primary_rate[pair] = rate
        else:
            self._last_secondary_rate[pair] = rate

        # 数据校验：主源与备源汇率差异检查
        self._check_rate_consistency(pair)

        # 异步写入数据库（批量写入提高性能）
        self._enqueue_rate_history(pair, rate, source, timestamp)

        self._rate_update_count += 1

    def _check_rate_consistency(self, pair: str):
        """检查主源与备源汇率一致性"""
        primary_rate = self._last_primary_rate.get(pair)
        secondary_rate = self._last_secondary_rate.get(pair)

        if primary_rate and secondary_rate:
            diff_bps = abs(primary_rate - secondary_rate) / primary_rate * 10000
            pair_config = self.CURRENCY_PAIRS.get(pair, {})
            max_spread = pair_config.get("max_spread_bps", 100)

            if diff_bps > max_spread:
                self._alert_rate_inconsistency(
                    pair, primary_rate, secondary_rate, diff_bps
                )

    def _load_yesterday_close_rates(self):
        """加载昨日收盘价作为降级数据"""
        rates = self.db.query(
            "SELECT from_currency, to_currency, rate "
            "FROM fx_rate_daily_close "
            "WHERE close_date = CURDATE() - INTERVAL 1 DAY"
        )
        for r in rates:
            pair_key = f"{r['from_currency']}/{r['to_currency']}"
            self.redis.setex(
                f"fx_rate_yesterday:{pair_key}",
                self.YESTERDAY_CLOSE_CACHE_TTL,
                json.dumps({
                    "rate": float(r["rate"]),
                    "source": "yesterday_close"
                })
            )

    def _alert_rate_fallback(self, pair: str, fallback_type: str):
        """汇率降级告警（带冷却机制防止告警风暴）"""
        key = f"{pair}:{fallback_type}"
        last_alert = self._alert_cooldown.get(key, datetime.min)
        if (now() - last_alert).seconds < 300:  # 5 分钟冷却
            return
        self._alert_cooldown[key] = now()
        logger.warning(f"汇率降级: {pair} 使用 {fallback_type}")
        self.alert_team(
            f"[汇率降级] {pair} 无法获取实时汇率，"
            f"当前使用 {fallback_type}。请检查数据源连接。"
        )

    def _alert_ws_disconnect(self, source: str, error: str):
        """WebSocket 断线告警"""
        self.alert_team(
            f"[汇率数据源断线] {source} 数据源 WebSocket 断开: {error}。"
            f"系统将自动重连，期间使用备源或缓存数据。"
        )

    def _alert_rate_inconsistency(self, pair, primary, secondary, diff_bps):
        """汇率不一致告警"""
        self.alert_team(
            f"[汇率不一致] {pair} 主源={primary} 备源={secondary}，"
            f"差异={diff_bps:.1f}bps，超过阈值。请人工确认。"
        )

    def _health_check_loop(self):
        """定期健康检查：确保汇率数据持续更新"""
        while self._running:
            time.sleep(30)  # 每 30 秒检查一次
            for pair in self.CURRENCY_PAIRS:
                cached = self.redis.get(f"fx_rate:{pair}")
                if not cached:
                    # 缓存已过期且无新推送 → 数据源可能断线
                    self._alert_rate_stale(pair)
                    # 尝试主动拉取
                    self._fetch_rate_fallback(pair)

    def _fetch_rate_fallback(self, pair: str):
        """主动拉取汇率（WebSocket 断线时的降级方案）"""
        try:
            rate = self.forex_http_client.get_rate(pair)
            self.redis.setex(
                f"fx_rate:{pair}",
                self.RATE_CACHE_TTL_SECONDS,
                json.dumps({"rate": float(rate), "source": "http_fallback"})
            )
        except Exception as e:
            logger.error(f"HTTP 降级拉取汇率失败: {pair}, error={e}")

    def _alert_rate_stale(self, pair: str):
        """汇率数据过期告警"""
        self.alert_team(
            f"[汇率数据过期] {pair} 超过 30 秒无更新，"
            f"请检查 WebSocket 连接状态。"
        )

    def get_provider_stats(self) -> dict:
        """获取汇率提供者运行统计"""
        return {
            "total_updates": self._rate_update_count,
            "primary_connected": (
                self.primary_ws is not None and self.primary_ws.sock is not None
            ),
            "secondary_connected": (
                self.secondary_ws is not None and self.secondary_ws.sock is not None
            ),
            "cached_pairs": len([
                k for k in self.redis.keys("fx_rate:*") if k
            ]),
            "active_locks": len([
                k for k in self.redis.keys("rate_lock:*") if k
            ]),
        }
```

**汇率缓存与锁定时序图：**

```
汇率更新流程：
  Reuters WebSocket → FXRateProvider._handle_rate_update()
  → Redis SETEX fx_rate:CNY/USD 10s {rate:7.20, source:"primary"}
  → 数据校验（主备差异检查）
  → 异步写入 fx_rate_history 表

汇率锁定流程：
  用户下单 → FXRateProvider.lock_rate(CNY, USD, 5000, order_id)
  → Redis GET fx_rate:CNY/USD → 获取当前汇率 7.20
  → Redis SETEX rate_lock:lock-order123 600s {rate:7.20, ...}
  → DB INSERT forex_locks {status:"locked", expires_at:now()+10min}
  → 返回锁定结果

汇率锁定执行：
  购汇请求 → FXRateProvider.execute_locked_rate(lock_id)
  → Redis GET rate_lock:lock-order123 → 获取锁定汇率
  → 调用外汇供应商 convert API
  → DB UPDATE forex_locks SET status='executed'
  → Redis DEL rate_lock:lock-order123

降级路径：
  实时汇率不可用 → Redis 缓存（10秒内有效）
  → 备数据源 → 昨日收盘价 → DB 历史记录 → 人工介入
```

### 支付状态跨系统追踪

跨境支付涉及支付网关、银行、合规三大独立系统，每个系统对同一笔交易有自己的状态定义。状态追踪服务需要聚合三方状态，映射为统一的内部状态，并在状态变更时通知商户。

```python
class PaymentStatusAggregator:
    """
    支付状态聚合器：合并支付网关、银行、合规三方的状态。
    核心问题：三方系统的状态定义不同，更新时机不同，需要统一映射和合并。
    """

    # 网关状态 → 内部状态映射
    GATEWAY_STATUS_MAP = {
        "PAYMENT_CREATED":     "pending",
        "PAYMENT_PROCESSING":  "processing",
        "PAYMENT_COMPLETED":   "completed",
        "PAYMENT_FAILED":      "failed",
        "PAYMENT_REFUNDED":    "refunded",
        "PAYMENT_CANCELLED":   "cancelled",
        "PAYMENT_PENDING_REVIEW": "manual_review",
    }

    # 银行状态 → 内部状态映射
    BANK_STATUS_MAP = {
        "INITIATED":           "processing",
        "ACCEPTED":            "processing",
        "REJECTED":            "failed",
        "COMPLETED":           "completed",
        "RETURNED":            "refunded",
        "PENDING_COMPLIANCE":  "compliance_review",
        "ON_HOLD":             "on_hold",
    }

    # 合规状态 → 内部状态映射
    COMPLIANCE_STATUS_MAP = {
        "SCREENING":           "compliance_screening",
        "APPROVED":            "compliance_approved",
        "REJECTED":            "compliance_rejected",
        "MANUAL_REVIEW":       "manual_review",
        "ESCALATED":           "compliance_escalated",
        "EXCEPTION":           "compliance_exception",
    }

    # 状态优先级（数值越大优先级越高，用于多系统状态冲突时决定最终状态）
    STATUS_PRIORITY = {
        "failed":                100,
        "compliance_rejected":   95,
        "refunded":              90,
        "on_hold":               80,
        "compliance_escalated":  75,
        "manual_review":         70,
        "compliance_review":     65,
        "compliance_screening":  60,
        "compliance_exception":  55,
        "compliance_approved":   50,
        "processing":            40,
        "pending":               30,
        "completed":             20,
        "cancelled":             15,
    }

    def __init__(self, db, redis, webhook_service):
        self.db = db
        self.redis = redis
        self.webhook_service = webhook_service

    def aggregate_status(self, order_id: str) -> dict:
        """
        聚合三方系统的状态，返回统一的支付状态。
        优先级规则：取优先级最高的状态作为当前状态。
        """
        # 获取三方最新状态
        gateway_status = self._get_gateway_status(order_id)
        bank_status = self._get_bank_status(order_id)
        compliance_status = self._get_compliance_status(order_id)

        # 映射为内部状态
        internal_gateway = self.GATEWAY_STATUS_MAP.get(
            gateway_status, "unknown"
        )
        internal_bank = self.BANK_STATUS_MAP.get(
            bank_status, "unknown"
        )
        internal_compliance = self.COMPLIANCE_STATUS_MAP.get(
            compliance_status, "unknown"
        )

        # 取优先级最高的状态
        all_statuses = [internal_gateway, internal_bank, internal_compliance]
        primary_status = max(
            all_statuses,
            key=lambda s: self.STATUS_PRIORITY.get(s, 0)
        )

        # 检测状态不一致
        unique_statuses = set(all_statuses) - {"unknown"}
        is_consistent = len(unique_statuses) <= 1

        result = {
            "order_id": order_id,
            "primary_status": primary_status,
            "gateway_status": gateway_status,
            "bank_status": bank_status,
            "compliance_status": compliance_status,
            "internal_gateway": internal_gateway,
            "internal_bank": internal_bank,
            "internal_compliance": internal_compliance,
            "is_consistent": is_consistent,
            "aggregated_at": now().isoformat(),
        }

        # 缓存聚合结果
        self.redis.setex(
            f"payment_status:{order_id}",
            60,  # 60 秒缓存
            json.dumps(result)
        )

        # 状态不一致告警
        if not is_consistent:
            self._alert_status_inconsistency(order_id, result)

        return result

    def update_and_notify(self, order_id: str, source: str, new_status: str):
        """
        收到某系统状态更新时，重新聚合并通知商户。
        source: "gateway" | "bank" | "compliance"
        """
        # 获取之前的状态
        prev_cached = self.redis.get(f"payment_status:{order_id}")
        prev_status = None
        if prev_cached:
            prev_data = json.loads(prev_cached)
            prev_status = prev_data.get("primary_status")

        # 更新来源系统的状态
        self._update_source_status(order_id, source, new_status)

        # 重新聚合
        aggregated = self.aggregate_status(order_id)
        current_status = aggregated["primary_status"]

        # 状态变更 → 通知商户
        if prev_status and prev_status != current_status:
            self._notify_merchant_status_change(
                order_id, prev_status, current_status, source
            )

        return aggregated

    def _notify_merchant_status_change(
        self, order_id: str, prev: str, current: str, source: str
    ):
        """通过 Webhook 通知商户支付状态变更"""
        order = self.db.query_one(
            "SELECT * FROM payment_orders WHERE order_id = %s", [order_id]
        )
        if not order:
            return

        webhook_payload = {
            "event": "payment_status_changed",
            "order_id": order_id,
            "merchant_id": order["merchant_id"],
            "previous_status": prev,
            "current_status": current,
            "changed_by": source,
            "timestamp": now().isoformat(),
            "amount_cny": float(order["amount_cny"]),
            "target_currency": order["target_currency"],
            "target_amount": (
                float(order["target_amount"]) if order["target_amount"] else None
            ),
        }

        # 记录通知日志
        self.db.insert("status_change_notifications", {
            "order_id": order_id,
            "merchant_id": order["merchant_id"],
            "prev_status": prev,
            "new_status": current,
            "source": source,
            "payload": json.dumps(webhook_payload, ensure_ascii=False),
            "created_at": now(),
        })

        # 发送 Webhook（幂等键确保不重复通知）
        merchant_webhook_url = self._get_merchant_webhook_url(
            order["merchant_id"]
        )
        if merchant_webhook_url:
            self.webhook_service.send(
                url=merchant_webhook_url,
                payload=webhook_payload,
                max_retries=3,
                idempotency_key=f"status-notify-{order_id}-{current}",
            )

    def _alert_status_inconsistency(self, order_id: str, result: dict):
        """状态不一致告警"""
        self.alert_team(
            f"[支付状态不一致] 订单 {order_id}："
            f"网关={result['internal_gateway']}，"
            f"银行={result['internal_bank']}，"
            f"合规={result['internal_compliance']}。"
            f"请人工核查三方系统数据。"
        )

    def _get_gateway_status(self, order_id: str) -> str:
        """从支付网关查询状态"""
        order = self.db.query_one(
            "SELECT channel_tx_id, status FROM payment_orders "
            "WHERE order_id = %s", [order_id]
        )
        if not order or not order["channel_tx_id"]:
            return "UNKNOWN"
        try:
            result = self.gateway_client.query(order["channel_tx_id"])
            return result.get("status", "UNKNOWN")
        except Exception:
            return order.get("status", "UNKNOWN")

    def _get_bank_status(self, order_id: str) -> str:
        """从银行查询状态"""
        order = self.db.query_one(
            "SELECT bank_tx_id FROM payment_orders WHERE order_id = %s",
            [order_id]
        )
        if not order or not order["bank_tx_id"]:
            return "UNKNOWN"
        try:
            return self.bank_client.query_status(order["bank_tx_id"])
        except Exception:
            return "UNKNOWN"

    def _get_compliance_status(self, order_id: str) -> str:
        """查询合规筛查状态"""
        screening = self.db.query_one(
            "SELECT status FROM aml_screening WHERE order_id = %s",
            [order_id]
        )
        return screening["status"].upper() if screening else "UNKNOWN"

    def _update_source_status(self, order_id: str, source: str, status: str):
        """更新来源系统的最新状态到数据库"""
        self.db.insert("payment_status_updates", {
            "order_id": order_id,
            "source": source,
            "raw_status": status,
            "created_at": now(),
        })
```

**状态映射对照表：**

| 网关状态 | 内部状态 | 银行状态 | 内部状态 | 合规状态 | 内部状态 |
|---------|---------|---------|---------|---------|---------|
| PAYMENT_CREATED | pending | INITIATED | processing | SCREENING | compliance_screening |
| PAYMENT_PROCESSING | processing | ACCEPTED | processing | APPROVED | compliance_approved |
| PAYMENT_COMPLETED | completed | COMPLETED | completed | REJECTED | compliance_rejected |
| PAYMENT_FAILED | failed | REJECTED | failed | MANUAL_REVIEW | manual_review |
| PAYMENT_REFUNDED | refunded | RETURNED | refunded | ESCALATED | compliance_escalated |
| PAYMENT_CANCELLED | cancelled | ON_HOLD | on_hold | EXCEPTION | compliance_exception |

### 结算与对账完整实现

```python
class SettlementService:
    """
    批量结算服务：按币种分组，每日批量执行结算。
    结算流程：将 T+2 日到期的结算记录按币种分组 → 计算净额 →
    提交银行批量汇款 → 匹配银行回执 → 更新结算状态。
    """

    def daily_settlement(self, settlement_date: date) -> dict:
        """执行每日批量结算"""
        batch_id = f"settle-{settlement_date.isoformat()}"

        # 1. 查询当日待结算记录
        pending = self.db.query(
            "SELECT * FROM settlement_records "
            "WHERE scheduled_date = %s AND status = 'scheduled' "
            "ORDER BY settlement_currency, merchant_id",
            [settlement_date]
        )

        if not pending:
            return {"batch_id": batch_id, "status": "no_records", "count": 0}

        # 2. 按币种分组
        by_currency = defaultdict(list)
        for record in pending:
            by_currency[record["settlement_currency"]].append(record)

        # 3. 按币种计算净结算金额
        settlement_reports = []
        for currency, records in by_currency.items():
            report = self._settle_by_currency(batch_id, currency, records)
            settlement_reports.append(report)

        # 4. 生成结算报告
        report = self._generate_settlement_report(
            batch_id, settlement_date, settlement_reports
        )

        return report

    def _settle_by_currency(
        self, batch_id: str, currency: str, records: list
    ) -> dict:
        """按币种执行批量结算，同商户合并为一笔汇款"""
        total_gross = sum(r["settlement_amount"] for r in records)
        total_fees = sum(r["platform_fee"] + r["channel_fee"] for r in records)
        total_forex_spread = sum(r["forex_spread_fee"] for r in records)
        total_net = sum(r["net_amount"] for r in records)

        # 按商户聚合（同一商户同一币种合并为一笔汇款）
        by_merchant = defaultdict(lambda: {
            "gross": Decimal("0"),
            "fees": Decimal("0"),
            "net": Decimal("0"),
            "count": 0,
        })
        for r in records:
            key = r["merchant_id"]
            by_merchant[key]["gross"] += r["settlement_amount"]
            by_merchant[key]["fees"] += r["platform_fee"] + r["channel_fee"]
            by_merchant[key]["net"] += r["net_amount"]
            by_merchant[key]["count"] += 1

        # 执行银行批量汇款
        transfer_results = []
        for merchant_id, amounts in by_merchant.items():
            merchant = self.db.get_merchant(merchant_id)
            try:
                result = self.bank_client.batch_transfer(
                    currency=currency,
                    amount=amounts["net"],
                    beneficiary_account=merchant["bank_account"],
                    beneficiary_name=merchant["company_name"],
                    swift_code=merchant["swift_code"],
                    reference=f"{batch_id}-{merchant_id}",
                )
                transfer_results.append({
                    "merchant_id": merchant_id,
                    "currency": currency,
                    "net_amount": float(amounts["net"]),
                    "bank_reference": result.get("reference"),
                    "status": result.get("status", "submitted"),
                })

                # 更新结算记录状态
                self.db.update(
                    "settlement_records",
                    {
                        "status": "processing",
                        "bank_reference": result.get("reference"),
                    },
                    {
                        "merchant_id": merchant_id,
                        "scheduled_date": settlement_date,
                    }
                )
            except Exception as e:
                # 单笔失败不影响其他商户
                transfer_results.append({
                    "merchant_id": merchant_id,
                    "currency": currency,
                    "net_amount": float(amounts["net"]),
                    "status": "failed",
                    "error": str(e),
                })
                self.alert_team(
                    f"[结算失败] 商户 {merchant_id} 币种 {currency} "
                    f"结算失败: {e}。将加入次日重试。"
                )

        return {
            "currency": currency,
            "total_records": len(records),
            "total_gross": float(total_gross),
            "total_fees": float(total_fees),
            "total_forex_spread": float(total_forex_spread),
            "total_net": float(total_net),
            "merchant_count": len(by_merchant),
            "transfers": transfer_results,
        }

    def _generate_settlement_report(
        self, batch_id: str, settlement_date: date, reports: list
    ) -> dict:
        """生成结算报告：包含各币种净额汇总"""
        currency_pair_summary = {}
        for r in reports:
            currency = r["currency"]
            net_in_currency = r["total_net"]
            if currency != "CNY":
                rate = self.rate_provider.get_rate(currency, "CNY")
                net_in_cny = net_in_currency * rate
            else:
                net_in_cny = net_in_currency
            currency_pair_summary[currency] = {
                "net_amount": net_in_currency,
                "net_in_cny": float(net_in_cny),
                "merchant_count": r["merchant_count"],
            }

        return {
            "batch_id": batch_id,
            "settlement_date": settlement_date.isoformat(),
            "total_records": sum(r["total_records"] for r in reports),
            "currencies": len(reports),
            "by_currency": {r["currency"]: r for r in reports},
            "currency_pair_summary": currency_pair_summary,
            "generated_at": now().isoformat(),
        }

    def reconcile_with_bank(self, settlement_date: date) -> dict:
        """
        银行对账：将结算记录与银行 MT940 对账单逐笔匹配。
        匹配规则：
        1. 首先按 bank_reference 精确匹配
        2. 未匹配的按金额+日期+商户模糊匹配
        3. 仍未匹配的记录为差异项
        """
        # 获取银行对账单
        bank_statement = self.bank_client.get_mt940(settlement_date)
        # 获取内部结算记录
        internal_records = self.db.query(
            "SELECT * FROM settlement_records "
            "WHERE scheduled_date = %s AND status IN ('processing', 'completed')",
            [settlement_date]
        )

        matched = []
        unmatched_internal = []
        unmatched_bank = list(bank_statement)

        # 第一步：按 bank_reference 精确匹配
        for record in internal_records:
            ref = record.get("bank_reference")
            bank_match = None
            for i, bank_entry in enumerate(unmatched_bank):
                if bank_entry.get("reference") == ref:
                    bank_match = bank_entry
                    unmatched_bank.pop(i)
                    break

            if bank_match:
                # 检查金额一致性
                if abs(record["net_amount"] - bank_match["amount"]) <= Decimal("0.01"):
                    matched.append({
                        "type": "exact_match",
                        "order_id": record["order_id"],
                        "internal_amount": float(record["net_amount"]),
                        "bank_amount": float(bank_match["amount"]),
                    })
                    self.db.update("settlement_records", {
                        "status": "completed",
                        "completed_date": settlement_date,
                    }, {"order_id": record["order_id"]})
                else:
                    # 金额不一致 → 记录差异
                    self.db.insert("reconciliation_diffs", {
                        "batch_id": f"settle-recon-{settlement_date.isoformat()}",
                        "reconcile_date": settlement_date,
                        "source_pair": "internal_vs_bank",
                        "diff_type": "amount_mismatch",
                        "internal_tx_id": record["order_id"],
                        "bank_tx_id": bank_match.get("reference"),
                        "internal_amount": record["net_amount"],
                        "external_amount": bank_match["amount"],
                        "diff_amount": abs(
                            record["net_amount"] - bank_match["amount"]
                        ),
                    })
            else:
                unmatched_internal.append(record)

        # 第二步：按金额+日期模糊匹配
        still_unmatched = []
        for record in unmatched_internal:
            fuzzy_match = None
            for i, bank_entry in enumerate(unmatched_bank):
                if (abs(record["net_amount"] - bank_entry["amount"]) <= Decimal("0.50")
                        and bank_entry.get("currency") == record["settlement_currency"]):
                    fuzzy_match = (i, bank_entry)
                    break

            if fuzzy_match:
                idx, bank_entry = fuzzy_match
                unmatched_bank.pop(idx)
                matched.append({
                    "type": "fuzzy_match",
                    "order_id": record["order_id"],
                    "internal_amount": float(record["net_amount"]),
                    "bank_amount": float(bank_entry["amount"]),
                })
            else:
                still_unmatched.append(record)

        return {
            "settlement_date": settlement_date.isoformat(),
            "total_internal": len(internal_records),
            "total_bank": len(bank_statement),
            "exact_matched": sum(1 for m in matched if m["type"] == "exact_match"),
            "fuzzy_matched": sum(1 for m in matched if m["type"] == "fuzzy_match"),
            "unmatched_internal": len(still_unmatched),
            "unmatched_bank": len(unmatched_bank),
            "match_rate": f"{len(matched) / max(len(internal_records), 1) * 100:.1f}%",
        }
```

**结算报告示例：**

```
结算批次: settle-2024-05-07
结算日期: 2024-05-07
总记录数: 250
涉及币种: 3

USD:
  总金额: $1,500,000
  平台手续费: $22,500 (1.5%)
  渠道手续费: $3,750 (0.25%)
  汇兑差价收入: $10,500 (0.7%)
  商家净额: $1,463,250
  商户数: 45
  汇款笔数: 45

EUR:
  总金额: EUR 800,000
  平台手续费: EUR 12,000
  渠道手续费: EUR 2,000
  汇兑差价收入: EUR 5,600
  商家净额: EUR 780,400
  商户数: 22

JPY:
  总金额: JPY 50,000,000
  平台手续费: JPY 750,000
  渠道手续费: JPY 125,000
  汇兑差价收入: JPY 350,000
  商家净额: JPY 48,775,000
  商户数: 15
```

### 更多异常场景

#### 场景：汇率数据源断连

```
触发：Reuters 外汇数据 API 全面故障（供应商侧宕机），
      WebSocket 连接持续断开，HTTP 降级接口也超时。
      所有币对无法获取实时汇率。
检测：
  1. WebSocket 重连失败 3 次 → 告警
  2. HTTP 降级接口超时 > 30 秒 → 告警
  3. Redis 中所有 fx_rate:* 键过期（10 秒后）→ 严重告警
  4. 上游依赖健康检查：每 30 秒 ping 供应商 API
处理：
  1. 自动切换到备数据源（Bloomberg WebSocket）
  2. 备源也不可用 → 降级到昨日收盘价
  3. 昨日收盘价模式：暂停新交易汇率锁定，已锁定交易正常执行
  4. 在降级汇率模式下增加汇率保护：交易金额上限降低 50%
  5. 通知运营团队，要求人工确认是否暂停跨境交易
  6. 数据源恢复后 → 自动切回实时汇率 → 通知运营团队
预防：多数据源冗余 + 自动降级 + 定期切换演练
```

#### 场景：结算批次部分失败

```
触发：每日批量结算时，银行批量汇款接口部分成功。
      100 笔结算中 85 笔成功，15 笔被银行拒绝。
      拒绝原因：商户银行账户变更（5 笔）、银行限额（3 笔）、
      SWIFT 格式错误（7 笔）。
检测：
  1. 银行批量汇款返回部分失败 → 解析每笔的状态
  2. 部分失败的笔数 > 5% → 告警
  3. SWIFT 格式错误 → 系统性问题，立即告警
处理：
  1. 成功的 85 笔：正常更新状态为 completed
  2. 账户变更的 5 笔：通知商户更新银行账户信息
  3. 银行限额的 3 笔：拆分为多笔小额重新提交
  4. SWIFT 格式错误的 7 笔：检查报文生成逻辑，修复后重新提交
  5. 所有失败笔记录到 settlement_exceptions 表，持续跟踪
  6. 次日重试：自动将未结算记录加入次日批次
预防：结算前预校验（验证银行账户有效性）+ SWIFT 报文格式测试
```

#### 场景：合规规则更新导致交易批量阻断

```
触发：外管局发布新规，调整个人购汇额度或新增制裁名单。
      规则更新后，大量正在处理中的交易突然不满足新规则。
      例如：新制裁名单新增 500 个实体 → 20 笔在途交易命中。
检测：
  1. 合规规则更新事件 → 系统自动触发在途交易复审
  2. 复审发现不满足新规则的交易 → 逐笔标记
  3. 受影响交易数量 > 10 → 批量告警
处理：
  1. 命中新制裁名单的交易 → 立即冻结，进入人工审核
  2. 额度规则变更 → 已超额的交易暂停，通知用户
  3. 已完成结算的交易 → 追溯审查，如违规则启动退款流程
  4. 批量通知受影响的商户和用户
  5. 合规团队逐笔审核，决定放行或终止
  6. 更新后端系统的规则缓存，确保新交易使用最新规则
预防：合规规则版本管理 + 灰度生效（先标记再阻断）+ 规则回滚机制
```

## FX 汇率管理系统完整实现

```python
class FXRateManager:
    """FX 汇率管理：多源报价 + 仲裁 + 锁定 + 审计"""

    MAJOR_PAIRS = ["USD/CNY", "EUR/USD", "GBP/USD", "JPY/USD", "AUD/USD"]
    RATE_CACHE_TTL = {
        "major": 5,    # 主要货币对 5 秒
        "minor": 30,   # 次要货币对 30 秒
        "exotic": 60,  # 外来货币对 60 秒
    }

    def get_rate(self, base_currency, quote_currency):
        """获取汇率"""
        pair = f"{base_currency}/{quote_currency}"

        # 1. 缓存检查
        cached = self.redis.get(f"fx_rate:{pair}")
        if cached:
            return json.loads(cached)

        # 2. 从多个数据源获取
        rates = []
        for provider in self.fx_providers:
            try:
                rate = provider.get_rate(base_currency, quote_currency)
                rates.append({"provider": provider.name, "rate": rate,
                             "timestamp": now()})
            except Exception:
                continue

        if not rates:
            raise FXRateUnavailableError(f"无法获取 {pair} 汇率")

        # 3. 仲裁：取中位数（排除异常值）
        rate_values = [r["rate"] for r in rates]
        median_rate = statistics.median(rate_values)

        # 排除偏差 > 2% 的异常源
        valid_rates = [r for r in rates
            if abs(r["rate"] - median_rate) / median_rate < 0.02]

        if not valid_rates:
            valid_rates = rates  # 全部偏差大 → 使用全部

        final_rate = statistics.mean([r["rate"] for r in valid_rates])

        # 4. 加上点差
        spread = self._get_spread(pair)
        final_rate_with_spread = final_rate * (1 + spread) if base_currency == "USD" \
            else final_rate / (1 + spread)

        # 5. 缓存
        ttl = self.RATE_CACHE_TTL.get(
            "major" if pair in self.MAJOR_PAIRS else "minor", 30)
        rate_data = {
            "pair": pair, "rate": round(final_rate_with_spread, 6),
            "raw_rate": round(final_rate, 6),
            "spread": spread,
            "sources": len(valid_rates),
            "updated_at": now().isoformat()
        }
        self.redis.setex(f"fx_rate:{pair}", ttl, json.dumps(rate_data))

        return rate_data

    def lock_rate(self, transaction_id, base_currency, quote_currency,
                  lock_seconds=30):
        """锁定汇率（交易期间不变）"""
        rate_data = self.get_rate(base_currency, quote_currency)
        lock_key = f"fx_lock:{transaction_id}"

        self.redis.setex(lock_key, lock_seconds, json.dumps({
            "pair": f"{base_currency}/{quote_currency}",
            "locked_rate": rate_data["rate"],
            "locked_at": now().isoformat(),
            "expires_at": (now() + timedelta(seconds=lock_seconds)).isoformat()
        }))

        # 存储历史汇率（审计）
        self.db.insert("fx_rate_history", {
            "transaction_id": transaction_id,
            "pair": f"{base_currency}/{quote_currency}",
            "rate": rate_data["rate"],
            "raw_rate": rate_data["raw_rate"],
            "source_count": rate_data["sources"],
            "locked_at": now()
        })

        return {"locked_rate": rate_data["rate"], "expires_in_seconds": lock_seconds}

    def _get_spread(self, pair):
        """获取点差（客户等级 + 货币对）"""
        base_spreads = {
            "USD/CNY": 0.001, "EUR/USD": 0.0005,
            "GBP/USD": 0.0008, "JPY/USD": 0.001,
        }
        return base_spreads.get(pair, 0.002)
```

## 制裁名单筛查

```python
class SanctionsScreeningService:
    """制裁名单筛查：OFAC/EU/UN + 模糊匹配 + 白名单"""

    def screen_payment(self, payment):
        """筛查支付交易"""
        # 1. 检查付款方
        sender_hits = self._screen_entity(
            name=payment["sender_name"],
            country=payment["sender_country"],
            dob=payment.get("sender_dob"),
            id_number=payment.get("sender_id"))

        # 2. 检查收款方
        beneficiary_hits = self._screen_entity(
            name=payment["beneficiary_name"],
            country=payment["beneficiary_country"],
            dob=payment.get("beneficiary_dob"),
            id_number=payment.get("beneficiary_id"))

        # 3. 检查交易特征
        risk_flags = self._check_transaction_risk(payment)

        all_hits = sender_hits + beneficiary_hits

        if not all_hits:
            return {"status": "clear", "hits": []}

        # 4. 分类处理
        blocked = [h for h in all_hits if h["match_type"] == "exact"]
        potential = [h for h in all_hits if h["match_type"] == "fuzzy"]

        if blocked:
            return {"status": "blocked", "hits": blocked}

        if potential:
            # 检查白名单
            whitelisted = self._check_whitelist(payment, potential)
            if whitelisted:
                return {"status": "whitelisted", "hits": potential}

            return {"status": "pending_review", "hits": potential}

        return {"status": "clear", "hits": []}

    def _screen_entity(self, name, country, dob=None, id_number=None):
        """筛查实体"""
        hits = []

        # 精确匹配
        exact = self.db.query(
            "SELECT * FROM sanctions_list WHERE name = %s "
            "AND (country IS NULL OR country = %s)", name, country)
        for match in exact:
            hits.append({"match_type": "exact", "list": match["source"],
                        "name": match["name"], "country": match["country"]})

        if hits:
            return hits

        # 模糊匹配（编辑距离 ≤ 2）
        fuzzy = self.db.query(
            "SELECT * FROM sanctions_list WHERE "
            "LEVENSHTEIN(name, %s) <= 2 "
            "AND (country IS NULL OR country = %s)", name, country)
        for match in fuzzy:
            similarity = 1 - self._levenshtein(name, match["name"]) / max(len(name), len(match["name"]))
            hits.append({
                "match_type": "fuzzy",
                "similarity": round(similarity, 3),
                "list": match["source"],
                "name": match["name"], "country": match["country"]
            })

        return hits

    def _check_whitelist(self, payment, hits):
        """检查白名单"""
        for hit in hits:
            wl = self.db.query_one(
                "SELECT * FROM screening_whitelist "
                "WHERE name = %s AND country = %s "
                "AND approved = 1 AND expires_at > NOW()",
                hit["name"], hit["country"])
            if wl:
                return True
        return False

    def _levenshtein(self, s1, s2):
        """编辑距离"""
        if len(s1) < len(s2):
            return self._levenshtein(s2, s1)
        if len(s2) == 0:
            return len(s1)
        previous_row = range(len(s2) + 1)
        for i, c1 in enumerate(s1):
            current_row = [i + 1]
            for j, c2 in enumerate(s2):
                insertions = previous_row[j + 1] + 1
                deletions = current_row[j] + 1
                substitutions = previous_row[j] + (c1 != c2)
                current_row.append(min(insertions, deletions, substitutions))
            previous_row = current_row
        return previous_row[-1]
```

## 异常场景补充

### 场景：FX 汇率源宕机

```
触发：所有汇率数据源不可用 → 无法获取实时汇率 → 交易暂停
检测：
  1. 汇率 API 连续失败 > 3 次 → 告警
  2. 缓存汇率过期 → 严重告警
处理：
  1. 使用最后已知汇率（标注"非实时"）
  2. 加大点差对冲风险（spread × 2）
  3. 大额交易暂停（> $10000）
  4. 汇率恢复后自动切换
预防：多数据源冗余 + 最后已知汇率兜底 + 大额交易保护
```

### 场景：制裁筛查误拦截紧急付款

```
触发：模糊匹配命中制裁名单 → 交易被拦截 → 紧急货款无法支付
检测：
  1. 客户投诉付款被拦截 → 误判
  2. 白名单审批延迟 → 影响业务
处理：
  1. 紧急白名单审批流程（合规官 15 分钟内审批）
  2. 相似度 > 0.9 的命中 → 人工快速审核
  3. 误判案例 → 加入永久白名单
预防：分级审核 + 紧急审批流程 + 误判学习优化
```

## FX 汇率管理系统（完整版）

### 实时汇率源集成与仲裁

```python
import json
import statistics
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from typing import List, Dict, Optional, Tuple
from enum import Enum
from collections import defaultdict


class FXPairCategory(Enum):
    MAJOR = "major"       # 主要货币对（USD/CNY, EUR/USD 等）
    MINOR = "minor"       # 次要货币对
    EXOTIC = "exotic"     # 外来货币对


class CustomerTier(Enum):
    STANDARD = "standard"     # 普通客户
    SILVER = "silver"         # 银牌客户
    GOLD = "gold"             # 金牌客户
    PLATINUM = "platinum"     # 白金客户


@dataclass
class FXRateQuote:
    """单个数据源的汇率报价"""
    provider: str
    pair: str
    rate: float
    timestamp: datetime
    bid: Optional[float] = None
    ask: Optional[float] = None


@dataclass
class FXRateData:
    """仲裁后的汇率数据"""
    pair: str
    rate: float                    # 最终汇率（含点差）
    raw_rate: float                # 原始汇率（不含点差）
    spread: float                  # 点差
    spread_bps: int                # 点差（基点）
    source_count: int              # 有效数据源数量
    category: FXPairCategory       # 货币对分类
    updated_at: datetime
    expires_at: datetime


@dataclass
class RateLock:
    """汇率锁定"""
    transaction_id: str
    pair: str
    locked_rate: float
    raw_rate: float
    locked_at: datetime
    expires_at: datetime
    customer_tier: CustomerTier


class FXProviderArbiter:
    """
    汇率源仲裁器
    多数据源 → 排除异常值 → 中位数/均值 → 加点差
    """

    OUTLIER_THRESHOLD = 0.02      # 偏差 > 2% 视为异常
    MIN_PROVIDERS = 1             # 最少有效数据源数

    def __init__(self, providers: List):
        self.providers = providers

    async def get_arbitrated_rate(self, base: str,
                                   quote: str) -> Tuple[float, int]:
        """
        获取仲裁汇率
        返回：(仲裁汇率, 有效数据源数量)
        """
        pair = f"{base}/{quote}"

        # 从所有数据源获取报价
        quotes: List[FXRateQuote] = []
        for provider in self.providers:
            try:
                rate = await provider.get_rate(base, quote)
                quotes.append(FXRateQuote(
                    provider=provider.name,
                    pair=pair,
                    rate=rate,
                    timestamp=datetime.utcnow()
                ))
            except Exception:
                continue

        if not quotes:
            raise FXRateUnavailableError(
                f"所有数据源不可用: {pair}")

        # 仲裁步骤 1：计算中位数
        rate_values = [q.rate for q in quotes]
        median_rate = statistics.median(rate_values)

        # 仲裁步骤 2：排除偏差 > 2% 的异常源
        valid_quotes = [
            q for q in quotes
            if abs(q.rate - median_rate) / median_rate
            < self.OUTLIER_THRESHOLD
        ]

        if not valid_quotes:
            # 全部偏差大 → 降级使用全部报价
            valid_quotes = quotes

        # 仲裁步骤 3：取有效报价的均值
        valid_rates = [q.rate for q in valid_quotes]
        arbitrated_rate = statistics.mean(valid_rates)

        return arbitrated_rate, len(valid_quotes)


class FXRateCache:
    """
    汇率缓存
    按货币对分类设置不同 TTL
    """

    TTL_CONFIG = {
        FXPairCategory.MAJOR: 5,      # 主要货币对：5 秒
        FXPairCategory.MINOR: 30,     # 次要货币对：30 秒
        FXPairCategory.EXOTIC: 60,    # 外来货币对：60 秒
    }

    MAJOR_PAIRS = {
        "USD/CNY", "EUR/USD", "GBP/USD",
        "JPY/USD", "AUD/USD", "USD/CAD", "USD/CHF"
    }

    EXOTIC_PAIRS = {
        "USD/THB", "USD/VND", "USD/PHP",
        "USD/IDR", "USD/MYR", "USD/BRL"
    }

    def __init__(self, redis_client):
        self.redis = redis_client

    def get(self, pair: str) -> Optional[FXRateData]:
        """获取缓存的汇率"""
        cached = self.redis.get(f"fx_rate:{pair}")
        if cached:
            data = json.loads(cached)
            return FXRateData(
                pair=data["pair"],
                rate=data["rate"],
                raw_rate=data["raw_rate"],
                spread=data["spread"],
                spread_bps=data.get("spread_bps", 0),
                source_count=data["sources"],
                category=FXPairCategory(data.get("category", "minor")),
                updated_at=datetime.fromisoformat(data["updated_at"]),
                expires_at=datetime.fromisoformat(data["expires_at"])
            )
        return None

    def set(self, rate_data: FXRateData) -> None:
        """缓存汇率"""
        ttl = self.TTL_CONFIG.get(rate_data.category, 30)
        self.redis.setex(
            f"fx_rate:{rate_data.pair}",
            ttl,
            json.dumps({
                "pair": rate_data.pair,
                "rate": rate_data.rate,
                "raw_rate": rate_data.raw_rate,
                "spread": rate_data.spread,
                "spread_bps": rate_data.spread_bps,
                "sources": rate_data.source_count,
                "category": rate_data.category.value,
                "updated_at": rate_data.updated_at.isoformat(),
                "expires_at": rate_data.expires_at.isoformat()
            })
        )

    def classify_pair(self, pair: str) -> FXPairCategory:
        """货币对分类"""
        if pair in self.MAJOR_PAIRS:
            return FXPairCategory.MAJOR
        elif pair in self.EXOTIC_PAIRS:
            return FXPairCategory.EXOTIC
        return FXPairCategory.MINOR


class FXRateLockService:
    """
    汇率锁定服务
    为待处理交易锁定汇率（30 秒窗口）
    """

    DEFAULT_LOCK_SECONDS = 30

    def __init__(self, redis_client, db_pool):
        self.redis = redis_client
        self.db = db_pool

    async def lock_rate(self, transaction_id: str,
                         base: str, quote: str,
                         rate: float, raw_rate: float,
                         lock_seconds: int = DEFAULT_LOCK_SECONDS,
                         customer_tier: CustomerTier = CustomerTier.STANDARD
                         ) -> RateLock:
        """锁定汇率"""
        now = datetime.utcnow()
        expires_at = now + timedelta(seconds=lock_seconds)

        lock = RateLock(
            transaction_id=transaction_id,
            pair=f"{base}/{quote}",
            locked_rate=rate,
            raw_rate=raw_rate,
            locked_at=now,
            expires_at=expires_at,
            customer_tier=customer_tier
        )

        # Redis 锁（带 TTL 自动过期）
        self.redis.setex(
            f"fx_lock:{transaction_id}",
            lock_seconds,
            json.dumps({
                "pair": lock.pair,
                "locked_rate": lock.locked_rate,
                "raw_rate": lock.raw_rate,
                "locked_at": lock.locked_at.isoformat(),
                "expires_at": lock.expires_at.isoformat(),
                "customer_tier": customer_tier.value
            })
        )

        # DB 审计记录
        await self.db.execute(
            """INSERT INTO fx_rate_history
               (transaction_id, pair, rate, raw_rate,
                source_count, locked_at, customer_tier)
               VALUES (%s, %s, %s, %s, %s, %s, %s)""",
            transaction_id, lock.pair, lock.locked_rate,
            lock.raw_rate, 1, now, customer_tier.value
        )

        return lock

    async def get_locked_rate(self,
                               transaction_id: str) -> Optional[RateLock]:
        """获取锁定的汇率"""
        cached = self.redis.get(f"fx_lock:{transaction_id}")
        if cached:
            data = json.loads(cached)
            return RateLock(
                transaction_id=transaction_id,
                pair=data["pair"],
                locked_rate=data["locked_rate"],
                raw_rate=data["raw_rate"],
                locked_at=datetime.fromisoformat(data["locked_at"]),
                expires_at=datetime.fromisoformat(data["expires_at"]),
                customer_tier=CustomerTier(
                    data.get("customer_tier", "standard"))
            )
        return None

    async def extend_lock(self, transaction_id: str,
                           additional_seconds: int = 30) -> bool:
        """延长汇率锁定时间"""
        lock = await self.get_locked_rate(transaction_id)
        if not lock:
            return False

        new_expiry = lock.expires_at + timedelta(seconds=additional_seconds)
        self.redis.setex(
            f"fx_lock:{transaction_id}",
            additional_seconds + int(
                (lock.expires_at - datetime.utcnow()).total_seconds()),
            json.dumps({
                "pair": lock.pair,
                "locked_rate": lock.locked_rate,
                "raw_rate": lock.raw_rate,
                "locked_at": lock.locked_at.isoformat(),
                "expires_at": new_expiry.isoformat(),
                "customer_tier": lock.customer_tier.value
            })
        )
        return True


class FXSpreadManager:
    """
    点差管理器
    按货币对 + 客户等级设置点差
    """

    # 基础点差（基点 bps，1 bps = 0.01%）
    BASE_SPREAD_BPS = {
        "USD/CNY": 10,      # 0.10%
        "EUR/USD": 5,       # 0.05%
        "GBP/USD": 8,       # 0.08%
        "JPY/USD": 10,      # 0.10%
        "AUD/USD": 12,      # 0.12%
    }

    # 客户等级点差折扣
    TIER_DISCOUNT = {
        CustomerTier.STANDARD: 1.0,    # 无折扣
        CustomerTier.SILVER: 0.85,     # 85 折
        CustomerTier.GOLD: 0.7,        # 7 折
        CustomerTier.PLATINUM: 0.5,    # 5 折
    }

    DEFAULT_SPREAD_BPS = 20   # 默认 0.20%

    def get_spread(self, pair: str,
                    tier: CustomerTier = CustomerTier.STANDARD) -> float:
        """获取点差（返回小数形式，如 0.001 = 0.1%）"""
        base_bps = self.BASE_SPREAD_BPS.get(
            pair, self.DEFAULT_SPREAD_BPS)
        discount = self.TIER_DISCOUNT.get(tier, 1.0)
        final_bps = base_bps * discount
        return final_bps / 10000  # bps → 小数

    def get_spread_bps(self, pair: str,
                        tier: CustomerTier = CustomerTier.STANDARD) -> int:
        """获取点差（基点形式）"""
        base_bps = self.BASE_SPREAD_BPS.get(
            pair, self.DEFAULT_SPREAD_BPS)
        discount = self.TIER_DISCOUNT.get(tier, 1.0)
        return int(base_bps * discount)


class FXRateManager:
    """
    FX 汇率管理器（完整版）
    集成：多源报价 → 仲裁 → 缓存 → 锁定 → 点差 → 审计
    """

    def __init__(self, providers: List, redis_client, db_pool):
        self.arbiter = FXProviderArbiter(providers)
        self.cache = FXRateCache(redis_client)
        self.lock_service = FXRateLockService(redis_client, db_pool)
        self.spread_manager = FXSpreadManager()
        self.redis = redis_client
        self.db = db_pool

    async def get_rate(self, base: str, quote: str,
                        tier: CustomerTier = CustomerTier.STANDARD
                        ) -> FXRateData:
        """获取汇率（含缓存 + 仲裁 + 点差）"""
        pair = f"{base}/{quote}"
        category = self.cache.classify_pair(pair)

        # 1. 检查缓存
        cached = self.cache.get(pair)
        if cached and cached.expires_at > datetime.utcnow():
            return cached

        # 2. 仲裁获取原始汇率
        raw_rate, source_count = await self.arbiter.get_arbitrated_rate(
            base, quote)

        # 3. 计算点差
        spread = self.spread_manager.get_spread(pair, tier)
        spread_bps = self.spread_manager.get_spread_bps(pair, tier)

        # 4. 应用点差
        if base == "USD":
            rate = raw_rate * (1 + spread)
        else:
            rate = raw_rate / (1 + spread)

        # 5. 构建结果
        now = datetime.utcnow()
        ttl = self.cache.TTL_CONFIG.get(category, 30)
        rate_data = FXRateData(
            pair=pair,
            rate=round(rate, 6),
            raw_rate=round(raw_rate, 6),
            spread=spread,
            spread_bps=spread_bps,
            source_count=source_count,
            category=category,
            updated_at=now,
            expires_at=now + timedelta(seconds=ttl)
        )

        # 6. 缓存
        self.cache.set(rate_data)

        # 7. 存储历史汇率（审计）
        await self._store_rate_audit(rate_data)

        return rate_data

    async def lock_rate_for_transaction(
            self, transaction_id: str, base: str, quote: str,
            tier: CustomerTier = CustomerTier.STANDARD,
            lock_seconds: int = 30) -> RateLock:
        """为交易锁定汇率"""
        rate_data = await self.get_rate(base, quote, tier)

        lock = await self.lock_service.lock_rate(
            transaction_id, base, quote,
            rate_data.rate, rate_data.raw_rate,
            lock_seconds, tier
        )

        return lock

    async def _store_rate_audit(self, rate_data: FXRateData) -> None:
        """存储汇率审计记录"""
        await self.db.execute(
            """INSERT INTO fx_rate_audit
               (pair, rate, raw_rate, spread, spread_bps,
                source_count, category, recorded_at)
               VALUES (%s, %s, %s, %s, %s, %s, %s, %s)""",
            rate_data.pair, rate_data.rate, rate_data.raw_rate,
            rate_data.spread, rate_data.spread_bps,
            rate_data.source_count, rate_data.category.value,
            rate_data.updated_at
        )
```

### FX 汇率相关 DB 表

```sql
-- 汇率历史表（审计用）
CREATE TABLE fx_rate_audit (
    id              BIGSERIAL PRIMARY KEY,
    pair            VARCHAR(16) NOT NULL,
    rate            DECIMAL(18, 6) NOT NULL,
    raw_rate        DECIMAL(18, 6) NOT NULL,
    spread          DECIMAL(10, 6) NOT NULL,
    spread_bps      INT NOT NULL,
    source_count    INT NOT NULL,
    category        VARCHAR(16) NOT NULL,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_fx_audit_pair_time
    ON fx_rate_audit(pair, recorded_at DESC);

-- 汇率锁定记录（审计用）
CREATE TABLE fx_rate_history (
    id              BIGSERIAL PRIMARY KEY,
    transaction_id  VARCHAR(64) NOT NULL,
    pair            VARCHAR(16) NOT NULL,
    rate            DECIMAL(18, 6) NOT NULL,
    raw_rate        DECIMAL(18, 6) NOT NULL,
    source_count    INT,
    locked_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    customer_tier   VARCHAR(16) NOT NULL DEFAULT 'standard'
);

CREATE INDEX idx_fx_history_tx ON fx_rate_history(transaction_id);
CREATE INDEX idx_fx_history_pair_time
    ON fx_rate_history(pair, locked_at DESC);
```

## 制裁名单筛查集成（完整版）

### 多制裁名单数据源

```python
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from typing import List, Dict, Optional, Set, Tuple
from enum import Enum
import statistics
import json
import re


class SanctionsListSource(Enum):
    OFAC_SDNL = "ofac_sdnl"           # OFAC 特别指定国民清单
    OFAC_SECTOR = "ofac_sector"        # OFAC 行业制裁清单
    EU_CONSOLIDATED = "eu_consolidated" # EU 综合制裁清单
    UN_SECURITY = "un_security"         # 联合国安理会制裁清单
    UK_HMT = "uk_hmt"                   # 英国财政部制裁清单


class ScreeningResult(Enum):
    CLEAR = "clear"                     # 通过
    BLOCKED = "blocked"                 # 精确命中，阻断
    PENDING_REVIEW = "pending_review"   # 模糊匹配，待审核
    WHITELISTED = "whitelisted"         # 白名单通过
    ESCALATED = "escalated"             # 升级审核


class EntityType(Enum):
    INDIVIDUAL = "individual"           # 个人
    ENTITY = "entity"                   # 组织/公司
    VESSEL = "vessel"                   # 船只


@dataclass
class SanctionsEntry:
    """制裁名单条目"""
    entry_id: str
    source: SanctionsListSource
    name: str
    aliases: List[str] = field(default_factory=list)
    entity_type: EntityType = EntityType.INDIVIDUAL
    country: Optional[str] = None
    date_of_birth: Optional[str] = None
    national_id: Optional[str] = None
    program: Optional[str] = None       # 制裁项目
    listing_date: Optional[datetime] = None
    additional_info: Optional[str] = None


@dataclass
class ScreeningHit:
    """筛查命中"""
    matched_entry: SanctionsEntry
    match_type: str                     # exact / fuzzy / alias
    similarity: float                   # 相似度 0-1
    matched_field: str                  # name / alias / country / dob
    screening_timestamp: datetime = field(
        default_factory=datetime.utcnow)


@dataclass
class ScreeningResult:
    """筛查结果"""
    payment_id: str
    result: str                         # clear / blocked / pending_review
    hits: List[ScreeningHit]
    screened_at: datetime
    screened_by: str                    # 自动 / 人工
    review_notes: Optional[str] = None


class SanctionsListIngestion:
    """
    制裁名单数据摄入
    每日从 OFAC/EU/UN 更新制裁名单
    """

    def __init__(self, db_pool, storage_client):
        self.db = db_pool
        self.storage = storage_client

    async def daily_update(self) -> Dict:
        """每日更新制裁名单"""
        results = {}

        for source in SanctionsListSource:
            try:
                # 下载最新名单
                raw_data = await self._download_list(source)

                # 解析名单
                entries = self._parse_list(source, raw_data)

                # 增量更新（新增、修改、移除）
                changes = await self._apply_delta(source, entries)

                results[source.value] = {
                    "total_entries": len(entries),
                    "added": changes["added"],
                    "updated": changes["updated"],
                    "removed": changes["removed"],
                    "updated_at": datetime.utcnow().isoformat()
                }
            except Exception as e:
                results[source.value] = {
                    "error": str(e),
                    "updated_at": datetime.utcnow().isoformat()
                }

        # 更新版本标记
        await self.db.execute(
            """INSERT INTO sanctions_list_version
               (source, version, entry_count, updated_at)
               VALUES (%s, %s, %s, NOW())
               ON CONFLICT (source) DO UPDATE SET
                 version = EXCLUDED.version,
                 entry_count = EXCLUDED.entry_count,
                 updated_at = NOW()""",
            "all", datetime.utcnow().strftime("%Y%m%d"),
            sum(r.get("total_entries", 0) for r in results.values()
                if "error" not in r)
        )

        return results

    async def _download_list(self,
                              source: SanctionsListSource) -> bytes:
        """下载制裁名单文件"""
        urls = {
            SanctionsListSource.OFAC_SDNL:
                "https://www.treasury.gov/ofac/downloads/sdn.xml",
            SanctionsListSource.EU_CONSOLIDATED:
                "https://webgate.ec.europa.eu/fsd/fsf/public/files/"
                "xmlFullSanctionsList/content",
            SanctionsListSource.UN_SECURITY:
                "https://scsanctions.un.org/resources/xml/en/"
                "consolidated.xml",
        }
        # 实际实现使用 HTTP 客户端下载
        pass

    def _parse_list(self, source: SanctionsListSource,
                     raw_data: bytes) -> List[SanctionsEntry]:
        """解析制裁名单（各源格式不同）"""
        # 简化实现：各源解析逻辑不同
        entries = []
        # ... 解析逻辑 ...
        return entries

    async def _apply_delta(self, source: SanctionsListSource,
                            entries: List[SanctionsEntry]) -> Dict:
        """增量更新制裁名单"""
        added = 0
        updated = 0
        removed = 0

        # 获取当前名单
        current_ids = set(await self.db.fetch_val(
            """SELECT ARRAY_AGG(entry_id) FROM sanctions_list
               WHERE source = %s""",
            source.value
        ) or [])

        new_ids = {e.entry_id for e in entries}

        # 新增
        to_add = [e for e in entries if e.entry_id not in current_ids]
        for entry in to_add:
            await self.db.execute(
                """INSERT INTO sanctions_list
                   (entry_id, source, name, aliases, entity_type,
                    country, date_of_birth, national_id,
                    program, listing_date, additional_info)
                   VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                   ON CONFLICT (entry_id, source) DO UPDATE SET
                     name = EXCLUDED.name,
                     aliases = EXCLUDED.aliases,
                     updated_at = NOW()""",
                entry.entry_id, entry.source.value, entry.name,
                json.dumps(entry.aliases), entry.entity_type.value,
                entry.country, entry.date_of_birth,
                entry.national_id, entry.program,
                entry.listing_date, entry.additional_info
            )
            added += 1

        # 移除
        to_remove = current_ids - new_ids
        if to_remove:
            await self.db.execute(
                """UPDATE sanctions_list
                   SET status = 'removed', removed_at = NOW()
                   WHERE entry_id = ANY(%s) AND source = %s""",
                list(to_remove), source.value
            )
            removed = len(to_remove)

        return {"added": added, "updated": updated, "removed": removed}


class SanctionsScreeningService:
    """
    制裁名单筛查服务（完整版）
    实时筛查 + 模糊匹配 + 白名单 + 升级审核 + 审计跟踪
    """

    FUZZY_THRESHOLD = 0.75   # 模糊匹配相似度阈值

    def __init__(self, db_pool, redis_client, alert_service):
        self.db = db_pool
        self.redis = redis_client
        self.alert = alert_service

    async def screen_payment(self, payment: Dict) -> Dict:
        """
        实时筛查支付交易
        返回：clear / blocked / pending_review / escalated
        """
        payment_id = payment["payment_id"]

        # 1. 筛查付款方
        sender_hits = await self._screen_entity(
            name=payment["sender_name"],
            country=payment["sender_country"],
            dob=payment.get("sender_dob"),
            id_number=payment.get("sender_id"),
            entity_type=EntityType.INDIVIDUAL
        )

        # 2. 筛查收款方
        beneficiary_hits = await self._screen_entity(
            name=payment["beneficiary_name"],
            country=payment["beneficiary_country"],
            dob=payment.get("beneficiary_dob"),
            id_number=payment.get("beneficiary_id"),
            entity_type=EntityType.ENTITY
        )

        # 3. 筛查中间行（如有）
        intermediary_hits = []
        if payment.get("intermediary_bank_name"):
            intermediary_hits = await self._screen_entity(
                name=payment["intermediary_bank_name"],
                country=payment.get("intermediary_bank_country"),
                entity_type=EntityType.ENTITY
            )

        all_hits = sender_hits + beneficiary_hits + intermediary_hits

        # 4. 记录筛查审计
        await self._record_screening_audit(payment_id, all_hits)

        # 5. 分类处理
        if not all_hits:
            return self._build_result(
                payment_id, "clear", [])

        # 精确命中 → 阻断
        exact_hits = [h for h in all_hits
                      if h.match_type == "exact"]
        if exact_hits:
            await self.alert.send_critical(
                title=f"制裁名单精确命中 - {payment_id}",
                message=(
                    f"付款方: {payment['sender_name']}\n"
                    f"收款方: {payment['beneficiary_name']}\n"
                    f"命中条目: {[h.matched_entry.name for h in exact_hits]}\n"
                    f"交易已自动阻断"
                )
            )
            return self._build_result(
                payment_id, "blocked", exact_hits)

        # 模糊命中 → 检查白名单
        fuzzy_hits = [h for h in all_hits
                      if h.match_type == "fuzzy"]

        if fuzzy_hits:
            # 白名单检查
            whitelisted = await self._check_whitelist(payment, fuzzy_hits)
            if whitelisted:
                return self._build_result(
                    payment_id, "whitelisted", fuzzy_hits)

            # 高相似度命中 → 升级审核
            high_similarity = [
                h for h in fuzzy_hits
                if h.similarity >= 0.9
            ]
            if high_similarity:
                await self._escalate_for_review(
                    payment_id, high_similarity)
                return self._build_result(
                    payment_id, "escalated", fuzzy_hits)

            # 普通模糊匹配 → 待审核
            return self._build_result(
                payment_id, "pending_review", fuzzy_hits)

        return self._build_result(payment_id, "clear", [])

    async def _screen_entity(self, name: str,
                              country: Optional[str] = None,
                              dob: Optional[str] = None,
                              id_number: Optional[str] = None,
                              entity_type: EntityType = EntityType.INDIVIDUAL
                              ) -> List[ScreeningHit]:
        """筛查实体（个人/组织）"""
        hits = []

        # 步骤 1：精确匹配（名称 + 国家 + DOB）
        exact_matches = await self.db.fetch_all(
            """SELECT * FROM sanctions_list
               WHERE status = 'active'
                 AND (name ILIKE %s
                      OR %s = ANY(aliases))
                 AND (country IS NULL OR country = %s)
                 AND (date_of_birth IS NULL OR date_of_birth = %s)""",
            name, name, country, dob
        )

        for match in exact_matches:
            entry = self._row_to_entry(match)
            hits.append(ScreeningHit(
                matched_entry=entry,
                match_type="exact",
                similarity=1.0,
                matched_field="name"
            ))

        if hits:
            return hits

        # 步骤 2：模糊匹配（编辑距离 ≤ 2 + 国家匹配）
        fuzzy_matches = await self.db.fetch_all(
            """SELECT *,
                 LEVENSHTEIN(LOWER(name), LOWER(%s)) AS edit_dist
               FROM sanctions_list
               WHERE status = 'active'
                 AND LEVENSHTEIN(LOWER(name), LOWER(%s)) <= 2
                 AND (country IS NULL OR country = %s)
               ORDER BY edit_dist
               LIMIT 10""",
            name, name, country
        )

        for match in fuzzy_matches:
            entry = self._row_to_entry(match)
            # 计算相似度
            max_len = max(len(name), len(entry.name))
            similarity = (1 - match["edit_dist"] / max_len
                          if max_len > 0 else 0)

            if similarity >= self.FUZZY_THRESHOLD:
                hits.append(ScreeningHit(
                    matched_entry=entry,
                    match_type="fuzzy",
                    similarity=round(similarity, 3),
                    matched_field="name"
                ))

        # 步骤 3：别名匹配
        alias_matches = await self.db.fetch_all(
            """SELECT *,
                 LEVENSHTEIN(LOWER(alias), LOWER(%s)) AS edit_dist
               FROM sanctions_list, UNNEST(aliases) AS alias
               WHERE status = 'active'
                 AND LEVENSHTEIN(LOWER(alias), LOWER(%s)) <= 2
               ORDER BY edit_dist
               LIMIT 5""",
            name, name
        )

        for match in alias_matches:
            entry = self._row_to_entry(match)
            max_len = max(len(name), len(match.get("alias", "")))
            similarity = (1 - match["edit_dist"] / max_len
                          if max_len > 0 else 0)

            if similarity >= self.FUZZY_THRESHOLD:
                hits.append(ScreeningHit(
                    matched_entry=entry,
                    match_type="fuzzy",
                    similarity=round(similarity, 3),
                    matched_field="alias"
                ))

        return hits

    async def _check_whitelist(self, payment: Dict,
                                hits: List[ScreeningHit]) -> bool:
        """检查白名单"""
        for hit in hits:
            wl = await self.db.fetch_one(
                """SELECT * FROM screening_whitelist
                   WHERE name = %s
                     AND country = %s
                     AND approved = TRUE
                     AND expires_at > NOW()""",
                hit.matched_entry.name,
                hit.matched_entry.country
            )
            if wl:
                return True
        return False

    async def add_to_whitelist(self, name: str, country: str,
                                reason: str, approver: str,
                                expires_days: int = 365) -> None:
        """添加白名单（需审批流程）"""
        await self.db.execute(
            """INSERT INTO screening_whitelist
               (name, country, reason, approver,
                approved, expires_at, created_at)
               VALUES (%s, %s, %s, %s, TRUE,
                       NOW() + INTERVAL '%s days', NOW())""",
            name, country, reason, approver, expires_days
        )

    async def _escalate_for_review(self, payment_id: str,
                                     hits: List[ScreeningHit]) -> None:
        """升级到合规官审核"""
        await self.db.execute(
            """INSERT INTO screening_escalations
               (payment_id, hit_count, max_similarity,
                status, escalated_at)
               VALUES (%s, %s, %s, 'pending', NOW())""",
            payment_id, len(hits),
            max(h.similarity for h in hits)
        )

        await self.alert.send_warning(
            title=f"制裁筛查升级审核 - {payment_id}",
            message=(
                f"高相似度命中: {len(hits)} 个\n"
                f"最高相似度: {max(h.similarity for h in hits):.1%}\n"
                f"命中名单: {[h.matched_entry.name for h in hits]}\n"
                f"请合规官在 2 小时内审核"
            )
        )

    async def _record_screening_audit(self, payment_id: str,
                                        hits: List[ScreeningHit]) -> None:
        """记录筛查审计"""
        result = "clear" if not hits else (
            "blocked" if any(h.match_type == "exact" for h in hits)
            else "fuzzy_match"
        )

        await self.db.execute(
            """INSERT INTO screening_audit
               (payment_id, result, hit_count, details, screened_at)
               VALUES (%s, %s, %s, %s, NOW())""",
            payment_id, result, len(hits),
            json.dumps([{
                "entry_name": h.matched_entry.name,
                "match_type": h.match_type,
                "similarity": h.similarity,
                "source": h.matched_entry.source.value,
                "matched_field": h.matched_field
            } for h in hits], ensure_ascii=False)
        )

    def _build_result(self, payment_id: str, result: str,
                       hits: List[ScreeningHit]) -> Dict:
        """构建筛查结果"""
        return {
            "payment_id": payment_id,
            "result": result,
            "hit_count": len(hits),
            "hits": [{
                "name": h.matched_entry.name,
                "source": h.matched_entry.source.value,
                "match_type": h.match_type,
                "similarity": h.similarity,
                "country": h.matched_entry.country,
                "matched_field": h.matched_field
            } for h in hits],
            "screened_at": datetime.utcnow().isoformat()
        }

    def _row_to_entry(self, row: Dict) -> SanctionsEntry:
        """DB 行转为 SanctionsEntry"""
        return SanctionsEntry(
            entry_id=row["entry_id"],
            source=SanctionsListSource(row["source"]),
            name=row["name"],
            aliases=json.loads(row.get("aliases", "[]")),
            entity_type=EntityType(
                row.get("entity_type", "individual")),
            country=row.get("country"),
            date_of_birth=row.get("date_of_birth"),
            national_id=row.get("national_id"),
            program=row.get("program"),
            listing_date=row.get("listing_date"),
            additional_info=row.get("additional_info")
        )
```

### 制裁筛查相关 DB 表

```sql
-- 制裁名单表
CREATE TABLE sanctions_list (
    id              BIGSERIAL PRIMARY KEY,
    entry_id        VARCHAR(64) NOT NULL,
    source          VARCHAR(32) NOT NULL,
    name            TEXT NOT NULL,
    aliases         JSONB DEFAULT '[]',
    entity_type     VARCHAR(16) DEFAULT 'individual',
    country         VARCHAR(4),
    date_of_birth   VARCHAR(20),
    national_id     VARCHAR(64),
    program         VARCHAR(128),
    listing_date    TIMESTAMPTZ,
    additional_info TEXT,
    status          VARCHAR(16) DEFAULT 'active',
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    removed_at      TIMESTAMPTZ,
    UNIQUE (entry_id, source)
);

-- 制裁名单搜索索引
CREATE INDEX idx_sanctions_name ON sanctions_list
    USING gin (name gin_trgm_ops);
CREATE INDEX idx_sanctions_country ON sanctions_list(country)
    WHERE status = 'active';

-- 筛查白名单
CREATE TABLE screening_whitelist (
    id          BIGSERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    country     VARCHAR(4),
    reason      TEXT NOT NULL,
    approver    VARCHAR(64) NOT NULL,
    approved    BOOLEAN DEFAULT TRUE,
    expires_at  TIMESTAMPTZ NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- 筛查审计
CREATE TABLE screening_audit (
    id          BIGSERIAL PRIMARY KEY,
    payment_id  VARCHAR(64) NOT NULL,
    result      VARCHAR(16) NOT NULL,
    hit_count   INT DEFAULT 0,
    details     JSONB,
    screened_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_screening_audit_payment
    ON screening_audit(payment_id);

-- 升级审核
CREATE TABLE screening_escalations (
    id              BIGSERIAL PRIMARY KEY,
    payment_id      VARCHAR(64) NOT NULL,
    hit_count       INT DEFAULT 0,
    max_similarity  FLOAT,
    status          VARCHAR(16) DEFAULT 'pending',
    reviewer        VARCHAR(64),
    review_notes    TEXT,
    escalated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    reviewed_at     TIMESTAMPTZ
);

-- 制裁名单版本
CREATE TABLE sanctions_list_version (
    source          VARCHAR(32) PRIMARY KEY,
    version         VARCHAR(32),
    entry_count     INT,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## 异常场景补充（续）

### 场景：FX 汇率源全面宕机导致使用过时汇率

```
触发：Reuters、Bloomberg、ECB 三个汇率数据源同时故障。
      Reuters：API 维护升级（计划内但未提前通知）。
      Bloomberg：WebSocket 连接断开，重连超时。
      ECB：HTTP 降级接口返回 503。
      Redis 中所有 fx_rate:* 键在 60 秒后全部过期。
      系统无法获取任何实时汇率，跨境交易全部报错。
检测：
  1. 汇率 API 连续失败 > 3 次 → 告警
  2. 所有活跃数据源不可用 → P0 告警
  3. Redis 中 fx_rate:* 键数量从 200 降到 0 → 严重告警
  4. 跨境支付接口错误率飙升到 100% → 业务影响
处理：
  1. 紧急降级：使用 DB 中最近一次有效汇率
     （标注"非实时，可能存在偏差"）
  2. 在降级模式下加大点差（spread × 3），对冲汇率风险
  3. 大额交易暂停：金额 > $10,000 的交易需人工确认汇率
  4. 小额交易（< $1,000）允许使用过时汇率，但增加确认提示
  5. 通知运营团队，要求联系数据源供应商确认恢复时间
  6. 数据源恢复后 → 自动切回实时汇率 → 通知运营团队
  7. 事后审计：检查降级期间所有交易的汇率偏差
预防：多数据源冗余 + 最后有效汇率缓存 + 降级模式 + 大额交易保护
```

### 场景：制裁筛查误命中阻断紧急付款

```
触发：某紧急货款付款的收款方名称与制裁名单上的一个实体
      模糊匹配（编辑距离 1，相似度 0.92）。
      系统自动将交易标记为"escalated"（升级审核），
      需要合规官在 2 小时内审核。但合规官正在处理其他案件，
      导致这笔紧急货款被阻断 4 小时，产生违约金 $50,000。
      收款方名称："Shanghai XinHua Trading Co."
      制裁名单名称："Shanghai Xinhua Trading Corp."（实际是完全不同的公司）
检测：
  1. 客户投诉紧急付款被阻断 → 误判
  2. 升级审核队列积压 > 10 笔 → 合规团队人手不足
  3. 单笔审核时间 > 2 小时 → SLA 违规
处理：
  1. 合规官立即审核该笔交易，确认是误判后放行
  2. 将误判案例加入白名单（有效期 1 年，需定期复审）
  3. 优化模糊匹配算法：增加行业分类和地址匹配维度
     （相同名称但不同行业/地址 → 降低相似度权重）
  4. 建立紧急审核通道：VIP 客户的交易可申请 30 分钟内加急审核
  5. 增加合规团队人手或使用 AI 辅助初审（自动排除明显误判）
  6. 事后分析：过去 30 天的误判率（目标 < 5%）
预防：紧急审核通道 + 白名单自动更新 + 多维度匹配 + AI 辅助初审 + SLA 监控

## 跨境支付反洗钱监控完整实现

```python
class AMLMonitoringService:
    """反洗钱监控：交易模式分析 → 可疑行为检测 → SAR 报告 → 合规审查"""

    RISK_INDICATORS = {
        "rapid_movement": "快速转移（入账后短时间内全额转出）",
        "structuring": "拆分交易（将大额拆为多笔小额规避阈值）",
        "round_amounts": "整数金额（非正常交易模式）",
        "dormant_account": "休眠账户突然活跃",
        "high_risk_jurisdiction": "高风险地区交易",
        "unusual_volume": "交易量异常增长",
        "layering": "多层转账（资金经过多个账户中转）",
    }

    HIGH_RISK_COUNTRIES = ["XX", "YY", "ZZ"]  # FATF 黑名单

    def analyze_transaction(self, transaction):
        """分析单笔交易"""
        user_id = transaction["user_id"]
        amount = Decimal(str(transaction["amount"]))
        risk_score = 0
        risk_factors = []

        # 1. 金额检查
        if amount >= Decimal("50000"):
            risk_score += 30
            risk_factors.append({"type": "large_amount", "detail": f"单笔 {amount}"})

        # 2. 整数金额
        if amount == amount.to_integral_value():
            risk_score += 5
            risk_factors.append({"type": "round_amounts"})

        # 3. 高风险地区
        if transaction.get("counterparty_country") in self.HIGH_RISK_COUNTRIES:
            risk_score += 40
            risk_factors.append({"type": "high_risk_jurisdiction",
                "detail": transaction["counterparty_country"]})

        # 4. 拆分交易检测
        recent_transactions = self.db.query(
            "SELECT * FROM transactions "
            "WHERE user_id = %s AND created_at > NOW() - INTERVAL 24 HOUR "
            "ORDER BY created_at", user_id)

        if len(recent_transactions) > 5:
            total_24h = sum(Decimal(t["amount"]) for t in recent_transactions) + amount
            avg_amount = total_24h / (len(recent_transactions) + 1)

            # 多笔小额且总额接近报告阈值
            if all(Decimal(t["amount"]) < Decimal("50000") for t in recent_transactions) \
               and amount < Decimal("50000") \
               and total_24h >= Decimal("48000"):
                risk_score += 50
                risk_factors.append({
                    "type": "structuring",
                    "detail": f"24h 内 {len(recent_transactions)+1} 笔小额交易，总额 {total_24h}"
                })

        # 5. 快速转移检测
        recent_incoming = [t for t in recent_transactions if t["direction"] == "incoming"]
        if recent_incoming and transaction["direction"] == "outgoing":
            last_incoming = recent_incoming[-1]
            time_diff = (now() - last_incoming["created_at"]).total_seconds()
            incoming_amount = Decimal(last_incoming["amount"])

            if time_diff < 3600 and abs(amount - incoming_amount) < Decimal("100"):
                risk_score += 35
                risk_factors.append({
                    "type": "rapid_movement",
                    "detail": f"入账 {incoming_amount} 后 {time_diff/60:.0f} 分钟内转出 {amount}"
                })

        # 6. 休眠账户
        last_activity = self.db.query_one(
            "SELECT MAX(created_at) as last FROM transactions WHERE user_id = %s",
            user_id)["last"]
        if last_activity:
            dormant_days = (now() - last_activity).days
            if dormant_days > 90:
                risk_score += 20
                risk_factors.append({
                    "type": "dormant_account",
                    "detail": f"休眠 {dormant_days} 天后突然活跃"
                })

        # 确定风险等级
        if risk_score >= 70:
            risk_level = "high"
        elif risk_score >= 40:
            risk_level = "medium"
        else:
            risk_level = "low"

        result = {
            "transaction_id": transaction["id"],
            "user_id": user_id,
            "amount": str(amount),
            "risk_score": risk_score,
            "risk_level": risk_level,
            "risk_factors": risk_factors,
            "analyzed_at": now().isoformat()
        }

        # 高风险 → 阻断 + 创建审查
        if risk_level == "high":
            self._block_transaction(transaction["id"], risk_factors)
            self._create_sar(transaction, risk_score, risk_factors)

        # 中风险 → 放行但标记
        elif risk_level == "medium":
            self._flag_transaction(transaction["id"], risk_factors)

        return result

    def _block_transaction(self, transaction_id, risk_factors):
        """阻断交易"""
        self.db.update("transactions",
            {"status": "blocked", "blocked_reason": json.dumps(risk_factors),
             "blocked_at": now()},
            {"id": transaction_id})

        self.notification.send("compliance_team",
            f"交易 {transaction_id} 因反洗钱规则被阻断，风险因素: {risk_factors}")

    def _flag_transaction(self, transaction_id, risk_factors):
        """标记可疑交易"""
        self.db.update("transactions",
            {"aml_flag": True, "aml_factors": json.dumps(risk_factors)},
            {"id": transaction_id})

    def _create_sar(self, transaction, risk_score, risk_factors):
        """创建可疑活动报告（SAR）"""
        sar_id = str(uuid4())

        self.db.insert("suspicious_activity_reports", {
            "sar_id": sar_id,
            "transaction_id": transaction["id"],
            "user_id": transaction["user_id"],
            "risk_score": risk_score,
            "risk_factors": json.dumps(risk_factors),
            "status": "pending_review",
            "created_at": now()
        })

        return sar_id

    def review_sar(self, sar_id, reviewer_id, decision, notes=None):
        """审查可疑活动报告"""
        sar = self.db.get_sar(sar_id)

        if sar["status"] != "pending_review":
            return {"status": "already_reviewed"}

        self.db.update("suspicious_activity_reports",
            {"status": decision, "reviewer_id": reviewer_id,
             "review_notes": notes, "reviewed_at": now()},
            {"sar_id": sar_id})

        if decision == "confirmed":
            # 确认可疑 → 上报监管机构
            self._report_to_regulator(sar)
            # 冻结账户
            self.db.update("users",
                {"status": "frozen", "frozen_reason": "AML"},
                {"id": sar["user_id"]})

        elif decision == "cleared":
            # 排除可疑 → 解除阻断
            self.db.update("transactions",
                {"status": "completed", "aml_flag": False},
                {"id": sar["transaction_id"]})

        return {"sar_id": sar_id, "decision": decision}
```

## 异常场景补充

### 场景：AML 误报导致合法交易被阻断

```
触发：商家大额跨境采购 → 触发反洗钱规则 → 交易被阻断 → 商家资金链断裂
检测：
  1. 合规审查排除率 > 50% → 规则过严
  2. 商家投诉交易被无故阻断 → 误报
处理：
  1. 提供快速申诉通道
  2. 已验证商家降低风险评分权重
  3. 规则定期校准
预防：快速申诉 + 白名单机制 + 规则校准
```

### 场景：新型洗钱模式未被检测

```
触发：犯罪分子使用新手法（加密货币混合器 → 法币出金）→ 现有规则未覆盖 → 漏报
检测：
  1. 定期审计发现未标记的可疑交易 → 规则遗漏
  2. 监管机构指出漏报 → 合规风险
处理：
  1. 机器学习模型补充规则引擎
  2. 定期更新规则库
  3. 人工抽查随机交易
预防：ML 模型 + 规则更新 + 人工抽查
```

## 跨境支付外汇对冲完整实现

```python
class FxHedgingService:
    """外汇对冲：远期合约 → 头寸监控 → 到期结算 → 风险评估"""

    def create_hedge_position(self, payment_id, amount, from_currency, to_currency,
                              maturity_date):
        """创建对冲头寸"""
        # 1. 获取远期汇率
        forward_rate = self.fx_provider.get_forward_rate(
            from_currency, to_currency, maturity_date)

        if not forward_rate:
            return {"status": "rate_unavailable"}

        # 2. 计算对冲成本
        days_to_maturity = (maturity_date - now()).days
        volatility = self._get_currency_volatility(from_currency, to_currency)
        hedge_premium = amount * Decimal(str(forward_rate)) * Decimal(str(volatility)) * \
                       Decimal(str(days_to_maturity)) / Decimal("365")

        # 3. 创建远期合约
        position_id = str(uuid4())
        self.db.insert("fx_hedge_positions", {
            "position_id": position_id,
            "payment_id": payment_id,
            "amount": str(amount),
            "from_currency": from_currency,
            "to_currency": to_currency,
            "forward_rate": str(forward_rate),
            "spot_rate_at_creation": str(
                self.fx_provider.get_spot_rate(from_currency, to_currency)),
            "maturity_date": maturity_date,
            "hedge_premium": str(hedge_premium.quantize(Decimal("0.01"))),
            "volatility": str(volatility),
            "status": "open",
            "created_at": now()
        })

        # 4. 扣除对冲费用
        self.db.insert("fx_hedge_payments", {
            "payment_id": str(uuid4()),
            "position_id": position_id,
            "amount": str(hedge_premium.quantize(Decimal("0.01"))),
            "currency": from_currency,
            "description": f"远期合约 {position_id} 对冲费用",
            "status": "completed",
            "created_at": now()
        })

        return {"position_id": position_id, "forward_rate": str(forward_rate),
                "hedge_premium": str(hedge_premium.quantize(Decimal("0.01"))),
                "maturity_date": maturity_date.isoformat()}

    def monitor_hedge_positions(self):
        """监控对冲头寸"""
        open_positions = self.db.query(
            "SELECT * FROM fx_hedge_positions WHERE status = 'open'")

        alerts = []

        for position in open_positions:
            # 1. 获取当前即期汇率
            spot_rate = self.fx_provider.get_spot_rate(
                position["from_currency"], position["to_currency"])

            if not spot_rate:
                continue

            # 2. 计算市值损益（mark-to-market P&L）
            forward_rate = Decimal(str(position["forward_rate"]))
            amount = Decimal(str(position["amount"]))

            # 如果汇率朝有利方向移动 → 盈利；否则 → 亏损
            mtm_pl = amount * (forward_rate - Decimal(str(spot_rate)))

            # 3. 更新市值
            self.db.update("fx_hedge_positions",
                {"current_spot_rate": str(spot_rate),
                 "mtm_pl": str(mtm_pl.quantize(Decimal("0.01"))),
                 "mtm_updated_at": now()},
                {"position_id": position["position_id"]})

            # 4. 到期预警（3 天内到期）
            days_to_maturity = (position["maturity_date"] - now()).days

            if days_to_maturity <= 3 and days_to_maturity > 0:
                alerts.append({
                    "position_id": position["position_id"],
                    "alert": "approaching_maturity",
                    "days_remaining": days_to_maturity,
                    "mtm_pl": str(mtm_pl.quantize(Decimal("0.01")))
                })

            # 5. 汇率大幅波动预警
            initial_spot = Decimal(str(position["spot_rate_at_creation"]))
            rate_change_pct = abs(Decimal(str(spot_rate)) - initial_spot) / initial_spot * 100

            if rate_change_pct > 5:
                alerts.append({
                    "position_id": position["position_id"],
                    "alert": "large_rate_movement",
                    "change_pct": str(rate_change_pct.quantize(Decimal("0.1"))),
                    "direction": "favorable" if mtm_pl > 0 else "unfavorable"
                })

        return {"monitored_positions": len(open_positions), "alerts": alerts}

    def settle_hedge_position(self, position_id):
        """结算对冲头寸"""
        position = self.db.get_hedge_position(position_id)

        if position["status"] != "open":
            return {"status": "already_settled"}

        # 1. 获取结算汇率
        settlement_rate = self.fx_provider.get_spot_rate(
            position["from_currency"], position["to_currency"])

        forward_rate = Decimal(str(position["forward_rate"]))
        amount = Decimal(str(position["amount"]))

        # 2. 计算对冲损益
        hedge_pl = amount * (forward_rate - Decimal(str(settlement_rate"))))

        # 3. 记录结算
        self.db.update("fx_hedge_positions",
            {"status": "settled",
             "settlement_rate": str(settlement_rate),
             "realized_pl": str(hedge_pl.quantize(Decimal("0.01"))),
             "settled_at": now()},
            {"position_id": position_id})

        # 4. 如果对冲盈利 → 补贴支付
        if hedge_pl > 0:
            self.db.insert("fx_hedge_settlements", {
                "settlement_id": str(uuid4()),
                "position_id": position_id,
                "payment_id": position["payment_id"],
                "amount": str(hedge_pl.quantize(Decimal("0.01"))),
                "currency": position["to_currency"],
                "type": "hedge_gain",
                "created_at": now()
            })

        return {"position_id": position_id, "status": "settled",
                "settlement_rate": str(settlement_rate),
                "realized_pl": str(hedge_pl.quantize(Decimal("0.01")))}

    def calculate_hedge_requirement(self, merchant_id):
        """计算对冲需求"""
        # 获取商家未来跨境支付计划
        upcoming_payments = self.db.query(
            "SELECT * FROM cross_border_payments "
            "WHERE merchant_id = %s AND status = 'pending' "
            "AND expected_settlement_date > NOW() "
            "ORDER BY expected_settlement_date",
            merchant_id)

        # 按币对和时间桶分组
        exposure = {}
        for payment in upcoming_payments:
            pair = f"{payment['from_currency']}/{payment['to_currency']}"
            settlement_date = payment["expected_settlement_date"]
            days_out = (settlement_date - now()).days

            # 时间桶
            if days_out <= 7:
                bucket = "1_week"
            elif days_out <= 30:
                bucket = "1_month"
            elif days_out <= 90:
                bucket = "3_month"
            else:
                bucket = "3_month_plus"

            key = f"{pair}:{bucket}"
            exposure.setdefault(key, {
                "pair": pair, "bucket": bucket,
                "total_amount": Decimal("0"),
                "payment_count": 0
            })
            exposure[key]["total_amount"] += Decimal(str(payment["amount"]))
            exposure[key]["payment_count"] += 1

        # 推荐对冲量（基于风险容忍度）
        risk_tolerance = self._get_merchant_risk_tolerance(merchant_id)

        recommendations = []
        for key, exp in exposure.items():
            # 保守 → 对冲 100%；激进 → 对冲 50%
            hedge_pct = Decimal("1.0") if risk_tolerance == "conservative" else \
                       Decimal("0.75") if risk_tolerance == "moderate" else \
                       Decimal("0.5")

            recommended_amount = exp["total_amount"] * hedge_pct

            recommendations.append({
                "currency_pair": exp["pair"],
                "time_bucket": exp["bucket"],
                "exposure_amount": str(exp["total_amount"].quantize(Decimal("0.01"))),
                "recommended_hedge": str(recommended_amount.quantize(Decimal("0.01"))),
                "hedge_percentage": str(hedge_pct * 100),
                "payment_count": exp["payment_count"]
            })

        return {"merchant_id": merchant_id,
                "risk_tolerance": risk_tolerance,
                "recommendations": recommendations}

    def _get_currency_volatility(self, from_cur, to_cur):
        """获取货币波动率"""
        # 从缓存获取
        vol_key = f"fx_volatility:{from_cur}:{to_cur}"
        cached = self.redis.get(vol_key)
        if cached:
            return float(cached)

        # 从历史数据计算（30 日年化波动率）
        rates = self.fx_provider.get_historical_rates(
            from_cur, to_cur, days=30)

        if len(rates) < 2:
            return 0.10  # 默认 10%

        import statistics
        returns = [(rates[i] - rates[i-1]) / rates[i-1]
                   for i in range(1, len(rates))]
        daily_vol = statistics.stdev(returns) if len(returns) > 1 else 0.005
        annualized_vol = daily_vol * (252 ** 0.5)  # 年化

        self.redis.setex(vol_key, 3600, str(annualized_vol))
        return annualized_vol

    def _get_merchant_risk_tolerance(self, merchant_id):
        """获取商家风险容忍度"""
        merchant = self.db.get_merchant(merchant_id)
        return merchant.get("fx_risk_tolerance", "moderate")
```

## 异常场景补充

### 场景：对冲头寸到期但支付未完成

```
触发：远期合约到期 → 但对应的跨境支付因合规审查被延迟 → 没有实际支付来结算 → 对冲头寸悬空
检测：
  1. 远期合约到期日 < 当前日期且 status 仍为 open → 过期未结算
  2. 关联支付状态非 completed → 支付未完成
处理：
  1. 到期前 3 天检查关联支付状态
  2. 如支付延迟 → 延展对冲头寸（支付展期费用）
  3. 无法延展 → 现金结算对冲损益
预防：提前检查 + 自动延展 + 现金结算
```

### 场景：汇率剧烈波动导致对冲不足

```
触发：本币突发贬值 15% → 远期合约仅对冲了 50% 风险敞口 → 剩余 50% 损失巨大
检测：
  1. 即期汇率偏离远期汇率 > 10% → 对冲不足
  2. 未对冲敞口损失超过阈值 → 风险暴露
处理：
  1. 高波动时期提高对冲比例至 80-100%
  2. 设置止损对冲（汇率触及阈值自动追加对冲）
  3. 紧急追加远期合约
预防：动态对冲比例 + 止损对冲 + 紧急追加
```
