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