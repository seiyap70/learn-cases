# C31 - 商家结算系统

## 一、系统概述

### 业务场景

电商平台连接数万商家，每天产生百万级订单。商家需要定期收到货款（扣除平台佣金后），同时平台需要处理退款冲抵、优惠券分摊、税费计算等复杂场景。结算系统是平台与商家之间的资金桥梁，任何错误都直接导致资金损失。

### 核心挑战

1. **结算精度** — 金额计算到分级别，浮点误差不可接受
2. **优惠券分摊** — 平台券、商家券、混合券按不同规则分摊
3. **退款冲抵** — 跨周期退款需从待结算金额或保证金中扣除
4. **结算周期** — T+1/T+7/月结，不同商家不同周期
5. **对账一致性** — 订单金额 = 商家收入 + 平台佣金 + 优惠分摊
6. **高并发** — 每日百万级订单需实时计入待结算

### 系统架构

```
订单完成 → 结算计算引擎 → 待结算账本 → 周期结算 → 打款 → 对账
                ↓               ↓
          优惠分摊计算      退款冲抵处理
                ↓               ↓
          佣金规则引擎      保证金管理
```

## 二、核心流程

### 2.1 结算计算引擎

```
订单完成
  │
  ├── 1. 计算订单实付金额
  │     └── 商品金额 + 运费 - 优惠券 - 满减
  │
  ├── 2. 优惠券分摊
  │     ├── 平台券 → 平台承担
  │     ├── 商家券 → 商家承担
  │     └── 混合券 → 按比例分摊
  │
  ├── 3. 计算平台佣金
  │     └── (商品金额 - 商家券分摊) × 佣金比例
  │
  ├── 4. 计算商家应得
  │     └── 商品金额 - 商家券分摊 - 佣金 + 平台券分摊
  │
  └── 5. 写入待结算账本
```

### 2.2 退款冲抵流程

```
退款请求
  │
  ├── 1. 查找原订单结算记录
  │
  ├── 2. 判断结算状态
  │     ├── 未结算 → 直接从待结算中扣除
  │     ├── 已结算未打款 → 从待打款中扣除
  │     └── 已打款 → 从保证金/下期待结算中扣除
  │
  ├── 3. 退款金额分摊（与原订单同比例）
  │     └── 退款金额 × 各方承担比例
  │
  └── 4. 平台退还佣金 + 平台券部分
```

## 三、完整实现

### 3.1 结算计算引擎

```python
from decimal import Decimal, ROUND_HALF_UP
from datetime import datetime, timedelta
import json
import hashlib
from uuid import uuid4

class SettlementCalculationEngine:
    """结算计算引擎：精确到分的金额计算 + 优惠分摊 + 佣金计算"""

    def calculate_order_settlement(self, order):
        """计算订单结算明细"""
        order_id = order["order_id"]
        merchant_id = order["merchant_id"]

        # 使用 Decimal 避免浮点误差
        item_amount = Decimal(str(order["item_amount"]))  # 商品金额
        shipping_fee = Decimal(str(order.get("shipping_fee", 0)))  # 运费
        total_amount = item_amount + shipping_fee  # 订单总金额

        # 1. 优惠券分摊
        coupon_details = self._allocate_coupons(order)

        # 2. 计算佣金基数（商品金额 - 商家承担的优惠券）
        merchant_coupon_burden = sum(
            Decimal(str(c["merchant_burden"])) for c in coupon_details
        )
        commission_base = item_amount - merchant_coupon_burden

        # 3. 计算平台佣金
        commission_rate = Decimal(str(self._get_commission_rate(merchant_id)))
        commission = (commission_base * commission_rate / Decimal("100")).quantize(
            Decimal("0.01"), rounding=ROUND_HALF_UP)

        # 4. 计算商家应得
        platform_coupon_burden = sum(
            Decimal(str(c["platform_burden"])) for c in coupon_details
        )
        merchant_income = (item_amount - merchant_coupon_burden - commission
                          + platform_coupon_burden + shipping_fee)

        # 5. 验证：商家收入 + 佣金 + 商家承担券 = 商品金额 + 运费 + 平台承担券
        # 等价验证：实付金额 = 商品金额 + 运费 - 全部优惠券
        total_coupon = sum(Decimal(str(c["amount"])) for c in coupon_details)
        actual_payment = total_amount - total_coupon
        expected_payment = Decimal(str(order["actual_payment"]))

        if abs(actual_payment - expected_payment) > Decimal("0.01"):
            raise SettlementCalculationError(
                f"结算金额不一致: 计算={actual_payment}, 订单={expected_payment}")

        result = {
            "order_id": order_id,
            "merchant_id": merchant_id,
            "item_amount": str(item_amount),
            "shipping_fee": str(shipping_fee),
            "total_amount": str(total_amount),
            "coupon_details": coupon_details,
            "total_coupon": str(total_coupon),
            "commission_base": str(commission_base),
            "commission_rate": str(commission_rate),
            "commission": str(commission),
            "merchant_income": str(merchant_income),
            "actual_payment": str(actual_payment),
            "calculated_at": datetime.now().isoformat()
        }

        return result

    def _allocate_coupons(self, order):
        """优惠券分摊计算"""
        coupons = order.get("coupons", [])
        if not coupons:
            return []

        item_amount = Decimal(str(order["item_amount"]))
        results = []

        for coupon in coupons:
            coupon_amount = Decimal(str(coupon["amount"]))
            coupon_type = coupon["type"]  # platform / merchant / hybrid
            coupon_id = coupon["coupon_id"]

            if coupon_type == "platform":
                # 平台券 → 平台全额承担
                results.append({
                    "coupon_id": coupon_id,
                    "type": "platform",
                    "amount": str(coupon_amount),
                    "merchant_burden": "0.00",
                    "platform_burden": str(coupon_amount),
                })

            elif coupon_type == "merchant":
                # 商家券 → 商家全额承担
                results.append({
                    "coupon_id": coupon_id,
                    "type": "merchant",
                    "amount": str(coupon_amount),
                    "merchant_burden": str(coupon_amount),
                    "platform_burden": "0.00",
                })

            elif coupon_type == "hybrid":
                # 混合券 → 按比例分摊
                platform_pct = Decimal(str(coupon.get("platform_pct", 50)))
                platform_burden = (coupon_amount * platform_pct / Decimal("100")).quantize(
                    Decimal("0.01"), rounding=ROUND_HALF_UP)
                merchant_burden = coupon_amount - platform_burden

                results.append({
                    "coupon_id": coupon_id,
                    "type": "hybrid",
                    "amount": str(coupon_amount),
                    "merchant_burden": str(merchant_burden),
                    "platform_burden": str(platform_burden),
                })

        return results

    def _get_commission_rate(self, merchant_id):
        """获取商家佣金比例"""
        merchant = self.db.get_merchant(merchant_id)
        category = merchant.get("category", "default")

        rates = {
            "electronics": 3.0,
            "clothing": 5.0,
            "food": 8.0,
            "digital": 2.0,
            "default": 5.0,
        }

        # 商家特殊费率优先
        if merchant.get("custom_commission_rate"):
            return merchant["custom_commission_rate"]

        return rates.get(category, rates["default"])
```

### 3.2 待结算账本

```python
class SettlementLedgerService:
    """待结算账本：实时入账 + 退款冲抵 + 周期结算"""

    def record_settlement(self, settlement_detail):
        """记录待结算"""
        merchant_id = settlement_detail["merchant_id"]
        order_id = settlement_detail["order_id"]

        # 1. 幂等检查
        existing = self.db.query_one(
            "SELECT * FROM settlement_ledger "
            "WHERE order_id = %s AND status != 'cancelled'",
            order_id)
        if existing:
            return {"status": "already_recorded", "ledger_id": existing["id"]}

        # 2. 写入待结算账本
        ledger_id = str(uuid4())
        merchant_income = Decimal(settlement_detail["merchant_income"])
        commission = Decimal(settlement_detail["commission"])

        self.db.insert("settlement_ledger", {
            "ledger_id": ledger_id,
            "order_id": order_id,
            "merchant_id": merchant_id,
            "item_amount": settlement_detail["item_amount"],
            "shipping_fee": settlement_detail["shipping_fee"],
            "commission": settlement_detail["commission"],
            "coupon_details": json.dumps(settlement_detail["coupon_details"]),
            "merchant_income": str(merchant_income),
            "status": "pending",  # pending / settled / offset_by_refund
            "settlement_period": None,
            "created_at": datetime.now()
        })

        # 3. 更新商家待结算汇总
        self.redis.hincrbyfloat(
            f"merchant_pending:{merchant_id}",
            "total_income", float(merchant_income))
        self.redis.hincrbyfloat(
            f"merchant_pending:{merchant_id}",
            "total_commission", float(commission))

        return {"ledger_id": ledger_id, "status": "pending",
                "merchant_income": str(merchant_income)}

    def handle_refund_offset(self, refund_request):
        """处理退款冲抵"""
        order_id = refund_request["order_id"]
        refund_amount = Decimal(str(refund_request["refund_amount"]))

        # 1. 查找原结算记录
        original = self.db.query_one(
            "SELECT * FROM settlement_ledger "
            "WHERE order_id = %s AND status != 'cancelled'",
            order_id)

        if not original:
            return {"status": "no_original_settlement"}

        original_income = Decimal(original["merchant_income"])
        original_commission = Decimal(original["commission"])
        original_item = Decimal(original["item_amount"])

        # 2. 按原比例计算退款分摊
        if original_item > 0:
            refund_ratio = refund_amount / original_item
        else:
            refund_ratio = Decimal("1")

        refund_merchant_burden = (original_income * refund_ratio).quantize(
            Decimal("0.01"), rounding=ROUND_HALF_UP)
        refund_commission_return = (original_commission * refund_ratio).quantize(
            Decimal("0.01"), rounding=ROUND_HALF_UP)

        # 3. 根据结算状态处理
        merchant_id = original["merchant_id"]

        if original["status"] == "pending":
            # 未结算 → 直接从待结算中扣除
            self.db.update("settlement_ledger",
                {"merchant_income": str(original_income - refund_merchant_burden),
                 "commission": str(original_commission - refund_commission_return),
                 "refund_offset": str(refund_merchant_burden),
                 "refund_commission_return": str(refund_commission_return),
                 "updated_at": datetime.now()},
                {"ledger_id": original["ledger_id"]})

            # 更新汇总
            self.redis.hincrbyfloat(f"merchant_pending:{merchant_id}",
                "total_income", -float(refund_merchant_burden))

        elif original["status"] == "settled":
            # 已结算 → 从保证金或下期待结算中扣除
            security_deposit = self._get_security_deposit(merchant_id)
            if security_deposit >= refund_merchant_burden:
                self._deduct_security_deposit(merchant_id, refund_merchant_burden)
            else:
                # 保证金不足 → 记入待扣除
                self.db.insert("pending_deductions", {
                    "deduction_id": str(uuid4()),
                    "merchant_id": merchant_id,
                    "order_id": order_id,
                    "amount": str(refund_merchant_burden),
                    "reason": "refund_offset",
                    "status": "pending",
                    "created_at": datetime.now()
                })

        # 4. 记录退款冲抵日志
        self.db.insert("refund_offset_log", {
            "log_id": str(uuid4()),
            "order_id": order_id,
            "merchant_id": merchant_id,
            "refund_amount": str(refund_amount),
            "merchant_burden": str(refund_merchant_burden),
            "commission_return": str(refund_commission_return),
            "original_status": original["status"],
            "created_at": datetime.now()
        })

        return {
            "order_id": order_id,
            "refund_amount": str(refund_amount),
            "merchant_burden": str(refund_merchant_burden),
            "commission_return": str(refund_commission_return),
            "offset_method": "pending_deduction" if original["status"] == "settled" else "direct_offset"
        }

    def _get_security_deposit(self, merchant_id):
        """获取商家保证金余额"""
        result = self.db.query_one(
            "SELECT balance FROM merchant_security_deposits "
            "WHERE merchant_id = %s", merchant_id)
        return Decimal(result["balance"]) if result else Decimal("0")

    def _deduct_security_deposit(self, merchant_id, amount):
        """扣除保证金"""
        self.db.execute(
            "UPDATE merchant_security_deposits "
            "SET balance = balance - %s, updated_at = %s "
            "WHERE merchant_id = %s AND balance >= %s",
            str(amount), datetime.now(), merchant_id, str(amount))
```

### 3.3 周期结算与打款

```python
class SettlementCycleService:
    """周期结算：T+1/T+7/月结 → 生成账单 → 打款"""

    CYCLE_CONFIG = {
        "T1": {"name": "T+1 结算", "delay_days": 1},
        "T7": {"name": "T+7 结算", "delay_days": 7},
        "MONTHLY": {"name": "月结", "day_of_month": 15},
    }

    def execute_settlement_cycle(self, merchant_id):
        """执行商家周期结算"""
        merchant = self.db.get_merchant(merchant_id)
        cycle_type = merchant.get("settlement_cycle", "T1")
        config = self.CYCLE_CONFIG[cycle_type]

        # 1. 确定结算时间段
        period_start, period_end = self._get_settlement_period(
            merchant_id, cycle_type, config)

        # 2. 查询待结算记录
        pending_records = self.db.query(
            "SELECT * FROM settlement_ledger "
            "WHERE merchant_id = %s AND status = 'pending' "
            "AND created_at BETWEEN %s AND %s "
            "ORDER BY created_at",
            merchant_id, period_start, period_end)

        if not pending_records:
            return {"status": "no_pending_records", "merchant_id": merchant_id}

        # 3. 汇总计算
        total_income = sum(Decimal(r["merchant_income"]) for r in pending_records)
        total_commission = sum(Decimal(r["commission"]) for r in pending_records)

        # 4. 扣除待扣除项（退款冲抵）
        pending_deductions = self.db.query(
            "SELECT * FROM pending_deductions "
            "WHERE merchant_id = %s AND status = 'pending'",
            merchant_id)
        total_deductions = sum(Decimal(d["amount"]) for d in pending_deductions)

        # 5. 计算实际打款金额
        settlement_amount = total_income - total_deductions

        # 6. 最低结算金额检查
        min_settlement = Decimal(str(merchant.get("min_settlement_amount", 100)))
        if settlement_amount < min_settlement:
            return {"status": "below_minimum",
                    "amount": str(settlement_amount),
                    "minimum": str(min_settlement)}

        # 7. 生成结算账单
        bill_id = str(uuid4())
        self.db.insert("settlement_bills", {
            "bill_id": bill_id,
            "merchant_id": merchant_id,
            "period_start": period_start,
            "period_end": period_end,
            "cycle_type": cycle_type,
            "order_count": len(pending_records),
            "total_income": str(total_income),
            "total_commission": str(total_commission),
            "total_deductions": str(total_deductions),
            "settlement_amount": str(settlement_amount),
            "status": "generated",
            "generated_at": datetime.now()
        })

        # 8. 标记记录为已结算
        for record in pending_records:
            self.db.update("settlement_ledger",
                {"status": "settled", "settlement_period": period_end,
                 "bill_id": bill_id, "settled_at": datetime.now()},
                {"ledger_id": record["ledger_id"]})

        # 9. 标记待扣除项为已处理
        for deduction in pending_deductions:
            self.db.update("pending_deductions",
                {"status": "processed", "processed_at": datetime.now(),
                 "bill_id": bill_id},
                {"deduction_id": deduction["deduction_id"]})

        # 10. 清除缓存
        self.redis.delete(f"merchant_pending:{merchant_id}")

        # 11. 通知商家
        self.notification.send(merchant_id,
            f"结算账单已生成: {settlement_amount} 元，"
            f"包含 {len(pending_records)} 笔订单")

        return {
            "bill_id": bill_id,
            "merchant_id": merchant_id,
            "period": f"{period_start} ~ {period_end}",
            "order_count": len(pending_records),
            "total_income": str(total_income),
            "total_deductions": str(total_deductions),
            "settlement_amount": str(settlement_amount),
            "status": "generated"
        }

    def execute_payout(self, bill_id):
        """执行打款"""
        bill = self.db.get_settlement_bill(bill_id)

        if bill["status"] != "generated":
            return {"status": "cannot_payout", "current_status": bill["status"]}

        merchant = self.db.get_merchant(bill["merchant_id"])
        settlement_amount = Decimal(bill["settlement_amount"])
        bank_account = merchant.get("bank_account")

        if not bank_account:
            return {"status": "no_bank_account"}

        # 1. 调用支付通道打款
        payout_result = self.payment_gateway.transfer(
            amount=float(settlement_amount),
            bank_account=bank_account,
            reference_id=bill_id,
            remark=f"商家结算 {bill['period_start']}~{bill['period_end']}")

        if payout_result["success"]:
            # 2. 更新账单状态
            self.db.update("settlement_bills",
                {"status": "paid", "paid_at": datetime.now(),
                 "payment_id": payout_result["transaction_id"],
                 "payment_channel": payout_result.get("channel")},
                {"bill_id": bill_id})

            # 3. 通知商家
            self.notification.send(bill["merchant_id"],
                f"结算款项 {settlement_amount} 元已到账")

            return {"status": "paid", "amount": str(settlement_amount),
                    "transaction_id": payout_result["transaction_id"]}
        else:
            # 打款失败
            self.db.update("settlement_bills",
                {"status": "payout_failed",
                 "failure_reason": payout_result.get("error", "unknown")},
                {"bill_id": bill_id})

            return {"status": "payout_failed",
                    "reason": payout_result.get("error")}

    def _get_settlement_period(self, merchant_id, cycle_type, config):
        """确定结算时间段"""
        if cycle_type == "T1":
            # T+1: 结算昨天及之前的所有待结算
            period_end = datetime.now().replace(
                hour=0, minute=0, second=0, microsecond=0)
            period_start = datetime(2020, 1, 1)  # 从最早记录开始

        elif cycle_type == "T7":
            period_end = datetime.now() - timedelta(days=1)
            period_end = period_end.replace(
                hour=23, minute=59, second=59)
            period_start = datetime(2020, 1, 1)

        elif cycle_type == "MONTHLY":
            # 月结: 上月1号到月末
            today = datetime.now()
            if today.day >= config["day_of_month"]:
                period_end = today.replace(day=1) - timedelta(days=1)
                period_end = period_end.replace(
                    hour=23, minute=59, second=59)
                period_start = period_end.replace(day=1, hour=0, minute=0)
            else:
                # 上上月
                last_month = today.replace(day=1) - timedelta(days=1)
                period_end = last_month.replace(
                    hour=23, minute=59, second=59)
                period_start = last_month.replace(day=1, hour=0, minute=0)

        return period_start, period_end
```

### 3.4 结算对账

```python
class SettlementReconciliationService:
    """结算对账：订单金额 vs 结算金额 + 三方对齐"""

    def reconcile_daily(self, date):
        """日粒度对账"""
        # 1. 订单侧总额
        order_total = self.db.query_one(
            "SELECT "
            "COUNT(*) as order_count, "
            "COALESCE(SUM(actual_payment), 0) as total_payment "
            "FROM orders "
            "WHERE status = 'completed' AND DATE(completed_at) = %s", date)

        # 2. 结算侧总额
        settlement_total = self.db.query_one(
            "SELECT "
            "COUNT(*) as record_count, "
            "COALESCE(SUM(merchant_income), 0) as total_income, "
            "COALESCE(SUM(commission), 0) as total_commission "
            "FROM settlement_ledger "
            "WHERE DATE(created_at) = %s", date)

        # 3. 验证：商家收入 + 佣金 ≈ 订单实付
        income = Decimal(settlement_total["total_income"])
        commission = Decimal(settlement_total["total_commission"])
        payment = Decimal(order_total["total_payment"])

        # 考虑优惠券分摊的差异
        expected = income + commission
        discrepancy = payment - expected

        result = {
            "date": date,
            "order_side": {
                "count": order_total["order_count"],
                "total_payment": str(payment),
            },
            "settlement_side": {
                "count": settlement_total["record_count"],
                "total_income": str(income),
                "total_commission": str(commission),
                "total": str(expected),
            },
            "discrepancy": str(discrepancy),
            "is_balanced": abs(discrepancy) < Decimal("1.00"),
        }

        # 4. 不平衡 → 逐笔排查
        if not result["is_balanced"]:
            self._investigate_discrepancy(date, discrepancy)

        self.db.insert("settlement_reconciliation", {
            "id": str(uuid4()), "date": date,
            "result": json.dumps(result, default=str),
            "status": "balanced" if result["is_balanced"] else "unbalanced",
            "checked_at": datetime.now()
        })

        return result

    def _investigate_discrepancy(self, date, discrepancy):
        """排查差异"""
        # 找出结算侧没有对应订单的记录
        orphan_settlements = self.db.query(
            "SELECT sl.* FROM settlement_ledger sl "
            "LEFT JOIN orders o ON sl.order_id = o.id "
            "WHERE o.id IS NULL AND DATE(sl.created_at) = %s", date)

        # 找出有订单但无结算记录的
        missing_settlements = self.db.query(
            "SELECT o.* FROM orders o "
            "LEFT JOIN settlement_ledger sl ON o.id = sl.order_id "
            "WHERE sl.order_id IS NULL AND o.status = 'completed' "
            "AND DATE(o.completed_at) = %s", date)

        self.alert(
            f"结算对账差异 {discrepancy} 元: "
            f"孤立结算 {len(orphan_settlements)} 条, "
            f"缺失结算 {len(missing_settlements)} 条")
```

## 四、异常场景

### 场景 1：优惠券分摊金额不平

```
触发：混合券 10 元，平台承担 50% → 5.00 + 5.00 = 10.00
  但 10.01 元时 → 5.005 + 5.005 → 四舍五入 → 5.01 + 5.01 = 10.02 ≠ 10.01
检测：
  1. 分摊总额 ≠ 优惠券金额 → 分摊不平
  2. 对账发现商家收入 + 佣金 + 平台承担 ≠ 实付金额
处理：
  1. 最后一分归平台承担（平台让利）
  2. 分摊计算时使用 Decimal 而非 float
  3. 对账检查分摊总额
预防：Decimal 计算 + 末分归平台 + 对账验证
```

### 场景 2：打款失败后重复打款

```
触发：银行打款成功但回调超时 → 系统认为失败 → 重新打款 → 商家收到两次
检测：
  1. 同一账单出现两次成功打款记录 → 重复打款
  2. 商家投诉多收钱 → 资金错误
处理：
  1. 打款前检查账单状态（非 generated 不允许打款）
  2. 打款操作幂等（同一 bill_id 只能打款一次）
  3. 发现重复 → 从下期结算中扣除
预防：状态检查 + 打款幂等 + 重复检测
```

### 场景 3：跨周期退款冲抵失败

```
触发：商家已提现 → 保证金不足 → 退款需从下期扣除 → 商家下期无结算 → 欠款
检测：
  1. 保证金余额 < 待扣除金额 → 不足
  2. 待扣除项持续累积 → 商家欠款
处理：
  1. 降低商家等级（限制提现）
  2. 要求商家补充保证金
  3. 严重欠款 → 暂停商家经营
预防：保证金预警 + 等级限制 + 经营暂停
```

### 场景 4：高并发下结算记录重复

```
触发：订单完成回调触发两次 → 创建两条结算记录 → 商家多收一次
检测：
  1. 同一 order_id 出现两条 settlement_ledger → 重复
  2. 结算总额 > 订单总额 → 重复入账
处理：
  1. 幂等检查（order_id 唯一约束）
  2. 重复记录 → 标记为 cancelled
  3. 对账发现重复 → 冲正
预防：唯一约束 + 幂等检查 + 对账冲正
```

### 场景 5：结算周期切换导致遗漏

```
触发：T+1 结算在 00:00 执行 → 但 23:59:59 的订单刚入账 → 被分到两天 → 某天遗漏
检测：
  1. 结算记录数与完成订单数不一致 → 遗漏
  2. 对账发现订单无对应结算 → 遗漏
处理：
  1. 结算扫描使用 created_at 而非 completed_at
  2. 结算前等待 5 分钟（确保所有入账完成）
  3. 遗漏记录 → 补录到下期
预防：延迟结算 + 补录机制 + 对账验证
```

### 3.5 保证金管理

```python
class MerchantSecurityDepositService:
    """保证金管理：缴纳 → 扣除 → 补缴预警 → 退还"""

    DEPOSIT_TIERS = {
        "standard": {"min_deposit": 10000, "max_settlement_amount": 500000},
        "premium": {"min_deposit": 50000, "max_settlement_amount": 5000000},
        "enterprise": {"min_deposit": 200000, "max_settlement_amount": 50000000},
    }

    def initialize_deposit(self, merchant_id, tier="standard"):
        """初始化保证金"""
        config = self.DEPOSIT_TIERS[tier]
        min_deposit = Decimal(str(config["min_deposit"]))

        self.db.insert("merchant_security_deposits", {
            "merchant_id": merchant_id,
            "tier": tier,
            "balance": "0.00",
            "required_amount": str(min_deposit),
            "max_settlement_amount": str(config["max_settlement_amount"]),
            "status": "insufficient",  # insufficient / sufficient / frozen
            "created_at": datetime.now()
        })

        return {"merchant_id": merchant_id, "tier": tier,
                "required_amount": str(min_deposit)}

    def pay_deposit(self, merchant_id, amount):
        """缴纳保证金"""
        amount = Decimal(str(amount))
        deposit = self.db.query_one(
            "SELECT * FROM merchant_security_deposits "
            "WHERE merchant_id = %s", merchant_id)

        new_balance = Decimal(deposit["balance"]) + amount
        required = Decimal(deposit["required_amount"])

        status = "sufficient" if new_balance >= required else "insufficient"

        self.db.update("merchant_security_deposits",
            {"balance": str(new_balance), "status": status,
             "updated_at": datetime.now()},
            {"merchant_id": merchant_id})

        # 记录缴纳日志
        self.db.insert("deposit_transactions", {
            "transaction_id": str(uuid4()),
            "merchant_id": merchant_id,
            "type": "payment",
            "amount": str(amount),
            "balance_after": str(new_balance),
            "created_at": datetime.now()
        })

        # 保证金充足 → 激活商家
        if status == "sufficient" and deposit["status"] == "insufficient":
            self.db.update("merchants",
                {"status": "active"}, {"id": merchant_id})
            self.notification.send(merchant_id, "保证金已充足，商家已激活")

        return {"balance": str(new_balance), "status": status}

    def deduct_deposit(self, merchant_id, amount, reason):
        """扣除保证金"""
        amount = Decimal(str(amount))
        deposit = self.db.query_one(
            "SELECT * FROM merchant_security_deposits "
            "WHERE merchant_id = %s", merchant_id)

        balance = Decimal(deposit["balance"])

        if balance < amount:
            # 保证金不足 → 冻结商家
            self.db.update("merchant_security_deposits",
                {"status": "frozen", "updated_at": datetime.now()},
                {"merchant_id": merchant_id})
            self.db.update("merchants",
                {"status": "suspended"}, {"id": merchant_id})
            self.alert(f"商家 {merchant_id} 保证金不足被冻结")

        new_balance = balance - amount
        self.db.update("merchant_security_deposits",
            {"balance": str(new_balance), "updated_at": datetime.now()},
            {"merchant_id": merchant_id})

        self.db.insert("deposit_transactions", {
            "transaction_id": str(uuid4()),
            "merchant_id": merchant_id,
            "type": "deduction",
            "amount": str(amount),
            "reason": reason,
            "balance_after": str(new_balance),
            "created_at": datetime.now()
        })

        return {"balance": str(new_balance),
                "status": "frozen" if new_balance < Decimal(deposit["required_amount"]) else "sufficient"}

    def check_deposit_health(self):
        """批量检查保证金健康度"""
        deposits = self.db.query(
            "SELECT * FROM merchant_security_deposits "
            "WHERE status != 'frozen'")

        alerts = []
        for deposit in deposits:
            balance = Decimal(deposit["balance"])
            required = Decimal(deposit["required_amount"])
            ratio = balance / required if required > 0 else 0

            if ratio < 0.3:
                # 保证金不足 30% → 紧急补缴
                alerts.append({
                    "merchant_id": deposit["merchant_id"],
                    "level": "critical",
                    "balance": str(balance),
                    "required": str(required),
                    "ratio": round(float(ratio), 2),
                    "message": "保证金严重不足，请立即补缴"
                })
                self.notification.send(deposit["merchant_id"],
                    f"保证金严重不足({ratio:.0%})，请立即补缴，否则将冻结账户")
            elif ratio < 0.6:
                # 保证金不足 60% → 预警
                alerts.append({
                    "merchant_id": deposit["merchant_id"],
                    "level": "warning",
                    "balance": str(balance),
                    "required": str(required),
                    "ratio": round(float(ratio), 2),
                    "message": "保证金偏低，建议补缴"
                })

        return {"checked": len(deposits), "alerts": alerts}
```

### 3.6 商家结算税率计算

```python
class SettlementTaxService:
    """结算税率：增值税 + 代扣税 + 跨境税"""

    TAX_CONFIG = {
        "domestic_normal": {"vat_rate": 6, "withholding_rate": 0},
        "domestic_small": {"vat_rate": 3, "withholding_rate": 0},
        "cross_border": {"vat_rate": 0, "withholding_rate": 10},
        "overseas": {"vat_rate": 0, "withholding_rate": 20},
    }

    def calculate_tax(self, merchant_id, settlement_amount):
        """计算结算税费"""
        merchant = self.db.get_merchant(merchant_id)
        tax_category = self._determine_tax_category(merchant)
        config = self.TAX_CONFIG[tax_category]

        settlement_amount = Decimal(str(settlement_amount))

        # 1. 增值税
        vat_rate = Decimal(str(config["vat_rate"]))
        if vat_rate > 0:
            vat = (settlement_amount * vat_rate / (Decimal("100") + vat_rate)).quantize(
                Decimal("0.01"), rounding=ROUND_HALF_UP)
            amount_before_vat = settlement_amount - vat
        else:
            vat = Decimal("0.00")
            amount_before_vat = settlement_amount

        # 2. 代扣税（针对跨境/海外商家）
        withholding_rate = Decimal(str(config["withholding_rate"]))
        if withholding_rate > 0:
            withholding = (amount_before_vat * withholding_rate / Decimal("100")).quantize(
                Decimal("0.01"), rounding=ROUND_HALF_UP)
        else:
            withholding = Decimal("0.00")

        # 3. 实际打款金额
        actual_payout = amount_before_vat - withholding

        return {
            "merchant_id": merchant_id,
            "tax_category": tax_category,
            "settlement_amount": str(settlement_amount),
            "vat_rate": str(vat_rate),
            "vat": str(vat),
            "amount_before_vat": str(amount_before_vat),
            "withholding_rate": str(withholding_rate),
            "withholding": str(withholding),
            "actual_payout": str(actual_payout),
            "tax_total": str(vat + withholding)
        }

    def _determine_tax_category(self, merchant):
        """确定税务类别"""
        if merchant.get("is_overseas"):
            return "overseas"
        elif merchant.get("is_cross_border"):
            return "cross_border"
        elif merchant.get("taxpayer_type") == "small":
            return "domestic_small"
        else:
            return "domestic_normal"

    def generate_tax_report(self, merchant_id, period_start, period_end):
        """生成税务报表"""
        settlements = self.db.query(
            "SELECT * FROM settlement_bills "
            "WHERE merchant_id = %s "
            "AND period_start >= %s AND period_end <= %s "
            "AND status = 'paid'",
            merchant_id, period_start, period_end)

        total_settlement = Decimal("0")
        total_vat = Decimal("0")
        total_withholding = Decimal("0")

        for bill in settlements:
            tax = self.calculate_tax(merchant_id, Decimal(bill["settlement_amount"]))
            total_settlement += Decimal(tax["settlement_amount"])
            total_vat += Decimal(tax["vat"])
            total_withholding += Decimal(tax["withholding"])

        return {
            "merchant_id": merchant_id,
            "period": f"{period_start} ~ {period_end}",
            "bill_count": len(settlements),
            "total_settlement": str(total_settlement),
            "total_vat": str(total_vat),
            "total_withholding": str(total_withholding),
            "total_tax": str(total_vat + total_withholding),
            "net_payout": str(total_settlement - total_vat - total_withholding)
        }
```

### 3.7 商家账单详情查询

```python
class SettlementQueryService:
    """结算查询：账单详情 + 流水明细 + 汇总统计"""

    def get_bill_detail(self, bill_id):
        """获取账单详情"""
        bill = self.db.get_settlement_bill(bill_id)

        # 获取账单关联的所有结算记录
        records = self.db.query(
            "SELECT * FROM settlement_ledger WHERE bill_id = %s "
            "ORDER BY created_at", bill_id)

        # 获取退款冲抵明细
        deductions = self.db.query(
            "SELECT * FROM pending_deductions WHERE bill_id = %s", bill_id)

        # 获取税费信息
        tax = self.tax_service.calculate_tax(
            bill["merchant_id"], Decimal(bill["settlement_amount"]))

        return {
            "bill_id": bill_id,
            "merchant_id": bill["merchant_id"],
            "period": f"{bill['period_start']} ~ {bill['period_end']}",
            "status": bill["status"],
            "summary": {
                "order_count": bill["order_count"],
                "total_income": bill["total_income"],
                "total_commission": bill["total_commission"],
                "total_deductions": bill["total_deductions"],
                "settlement_amount": bill["settlement_amount"],
            },
            "tax": tax,
            "actual_payout": tax["actual_payout"],
            "records": [{
                "order_id": r["order_id"],
                "item_amount": r["item_amount"],
                "commission": r["commission"],
                "merchant_income": r["merchant_income"],
                "created_at": r["created_at"].isoformat()
            } for r in records],
            "deductions": [{
                "order_id": d["order_id"],
                "amount": d["amount"],
                "reason": d["reason"],
            } for d in deductions],
            "payment": {
                "paid_at": bill.get("paid_at"),
                "payment_id": bill.get("payment_id"),
                "payment_channel": bill.get("payment_channel"),
            } if bill.get("paid_at") else None
        }

    def get_merchant_settlement_summary(self, merchant_id, months=3):
        """获取商家结算汇总"""
        start_date = datetime.now() - timedelta(days=months * 30)

        bills = self.db.query(
            "SELECT * FROM settlement_bills "
            "WHERE merchant_id = %s AND generated_at >= %s "
            "ORDER BY generated_at DESC",
            merchant_id, start_date)

        total_settled = sum(Decimal(b["settlement_amount"]) for b in bills
                          if b["status"] in ("paid", "generated"))
        total_commission = sum(Decimal(b["total_commission"]) for b in bills)
        total_deductions = sum(Decimal(b["total_deductions"]) for b in bills)

        # 待结算金额
        pending = self.db.query_one(
            "SELECT COALESCE(SUM(merchant_income), 0) as total "
            "FROM settlement_ledger "
            "WHERE merchant_id = %s AND status = 'pending'",
            merchant_id)

        return {
            "merchant_id": merchant_id,
            "period_months": months,
            "total_bills": len(bills),
            "total_settled": str(total_settled),
            "total_commission": str(total_commission),
            "total_deductions": str(total_deductions),
            "pending_settlement": str(pending["total"]),
            "recent_bills": [{
                "bill_id": b["bill_id"],
                "period": f"{b['period_start']} ~ {b['period_end']}",
                "amount": b["settlement_amount"],
                "status": b["status"],
            } for b in bills[:5]]
        }
```

## 五、数据库设计

### 核心表结构

```sql
-- 结算账本
CREATE TABLE settlement_ledger (
    ledger_id VARCHAR(36) PRIMARY KEY,
    order_id VARCHAR(64) NOT NULL,
    merchant_id VARCHAR(36) NOT NULL,
    item_amount DECIMAL(12,2) NOT NULL,
    shipping_fee DECIMAL(12,2) DEFAULT 0,
    commission DECIMAL(12,2) NOT NULL,
    coupon_details JSON,
    merchant_income DECIMAL(12,2) NOT NULL,
    refund_offset DECIMAL(12,2) DEFAULT 0,
    refund_commission_return DECIMAL(12,2) DEFAULT 0,
    status ENUM('pending','settled','offset_by_refund','cancelled') DEFAULT 'pending',
    settlement_period DATE,
    bill_id VARCHAR(36),
    created_at DATETIME NOT NULL,
    settled_at DATETIME,
    UNIQUE KEY uk_order (order_id),
    KEY idx_merchant_status (merchant_id, status),
    KEY idx_created (created_at)
) ENGINE=InnoDB;

-- 结算账单
CREATE TABLE settlement_bills (
    bill_id VARCHAR(36) PRIMARY KEY,
    merchant_id VARCHAR(36) NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    cycle_type VARCHAR(10) NOT NULL,
    order_count INT NOT NULL,
    total_income DECIMAL(14,2) NOT NULL,
    total_commission DECIMAL(14,2) NOT NULL,
    total_deductions DECIMAL(14,2) DEFAULT 0,
    settlement_amount DECIMAL(14,2) NOT NULL,
    status ENUM('generated','paid','payout_failed','cancelled') DEFAULT 'generated',
    payment_id VARCHAR(64),
    payment_channel VARCHAR(32),
    paid_at DATETIME,
    failure_reason TEXT,
    generated_at DATETIME NOT NULL,
    KEY idx_merchant (merchant_id),
    KEY idx_period (period_start, period_end)
) ENGINE=InnoDB;

-- 保证金
CREATE TABLE merchant_security_deposits (
    merchant_id VARCHAR(36) PRIMARY KEY,
    tier VARCHAR(20) NOT NULL,
    balance DECIMAL(14,2) NOT NULL DEFAULT 0,
    required_amount DECIMAL(14,2) NOT NULL,
    max_settlement_amount DECIMAL(14,2) NOT NULL,
    status ENUM('insufficient','sufficient','frozen') DEFAULT 'insufficient',
    created_at DATETIME NOT NULL,
    updated_at DATETIME
) ENGINE=InnoDB;

-- 待扣除项（退款冲抵）
CREATE TABLE pending_deductions (
    deduction_id VARCHAR(36) PRIMARY KEY,
    merchant_id VARCHAR(36) NOT NULL,
    order_id VARCHAR(64) NOT NULL,
    amount DECIMAL(12,2) NOT NULL,
    reason VARCHAR(100) NOT NULL,
    status ENUM('pending','processed','cancelled') DEFAULT 'pending',
    bill_id VARCHAR(36),
    created_at DATETIME NOT NULL,
    processed_at DATETIME,
    KEY idx_merchant_status (merchant_id, status)
) ENGINE=InnoDB;

-- 保证金交易记录
CREATE TABLE deposit_transactions (
    transaction_id VARCHAR(36) PRIMARY KEY,
    merchant_id VARCHAR(36) NOT NULL,
    type ENUM('payment','deduction','refund') NOT NULL,
    amount DECIMAL(12,2) NOT NULL,
    reason VARCHAR(200),
    balance_after DECIMAL(14,2) NOT NULL,
    created_at DATETIME NOT NULL,
    KEY idx_merchant (merchant_id)
) ENGINE=InnoDB;
```

## 六、扩展场景

### 场景 6：商家更换银行账户

```
触发：商家申请更换银行账户 → 有待打款账单 → 账户变更可能导致打款失败
检测：
  1. 商家银行账户变更请求 → 检查是否有待打款账单
  2. 打款到旧账户失败 → 账户已注销
处理：
  1. 变更前完成所有待打款
  2. 变更需要审批（防止账户被盗）
  3. 变更后首笔打款小额测试
预防：变更前结算 + 审批流程 + 小额测试
```

### 场景 7：大批量退款导致保证金耗尽

```
触发：质量问题 → 平台批量退款 1000 单 → 保证金从 10 万降至 0 → 商家被冻结
检测：
  1. 保证金余额快速下降 → 批量退款
  2. 保证金余额 < 30% → 预警
处理：
  1. 批量退款前检查保证金是否足够
  2. 不够 → 限制退款速度 + 通知商家补缴
  3. 冻结前给予 24 小时补缴窗口
预防：批量退款检查 + 限速 + 补缴窗口
```

### 场景 8：跨境结算汇率波动

```
触发：商家在海外 → 结算金额 USD → 打款时汇率变化 → 实际到账与预期不符
检测：
  1. 结算货币与打款货币不同 → 汇率风险
  2. 实际到账金额与账单金额差异 > 2% → 汇率损失
处理：
  1. 结算时锁定汇率（有效期 24 小时）
  2. 汇率波动 > 2% → 通知商家确认
  3. 商家可选择结算币种
预防：汇率锁定 + 波动通知 + 币种选择
```

### 3.8 商家佣金规则引擎

```python
class CommissionRuleEngine:
    """佣金规则引擎：品类费率 + 阶梯费率 + 促销费率 + 特殊协议"""

    def calculate_commission(self, merchant_id, order):
        """计算订单佣金"""
        merchant = self.db.get_merchant(merchant_id)
        category = order.get("category", merchant.get("category", "default"))

        # 1. 基础费率
        base_rate = self._get_base_rate(merchant_id, category)

        # 2. 阶梯费率（月累计 GMV 越高 → 费率越低）
        tier_rate = self._get_tier_rate(merchant_id, base_rate)

        # 3. 促销费率（活动期间降低佣金吸引商家参与）
        promo_rate = self._get_promo_rate(merchant_id, category)

        # 4. 特殊协议费率（大商家有自定义费率）
        contract_rate = self._get_contract_rate(merchant_id)

        # 5. 取最低费率
        final_rate = min(base_rate, tier_rate, promo_rate)
        if contract_rate is not None:
            final_rate = min(final_rate, contract_rate)

        # 6. 计算佣金基数（商品金额 - 商家券）
        item_amount = Decimal(str(order["item_amount"]))
        merchant_coupon = Decimal(str(order.get("merchant_coupon_amount", 0)))
        commission_base = item_amount - merchant_coupon

        # 7. 计算佣金
        commission = (commission_base * Decimal(str(final_rate)) / Decimal("100")).quantize(
            Decimal("0.01"), rounding=ROUND_HALF_UP)

        return {
            "merchant_id": merchant_id,
            "category": category,
            "base_rate": str(base_rate),
            "tier_rate": str(tier_rate),
            "promo_rate": str(promo_rate) if promo_rate < base_rate else None,
            "contract_rate": str(contract_rate) if contract_rate else None,
            "final_rate": str(final_rate),
            "commission_base": str(commission_base),
            "commission": str(commission)
        }

    def _get_base_rate(self, merchant_id, category):
        """获取基础费率"""
        rates = {
            "electronics": 3.0, "clothing": 5.0, "food": 8.0,
            "beauty": 6.0, "home": 5.0, "digital": 2.0,
            "default": 5.0,
        }
        return rates.get(category, rates["default"])

    def _get_tier_rate(self, merchant_id, base_rate):
        """获取阶梯费率"""
        # 本月累计 GMV
        monthly_gmv = self.db.query_one(
            "SELECT COALESCE(SUM(item_amount), 0) as total "
            "FROM settlement_ledger "
            "WHERE merchant_id = %s "
            "AND created_at >= DATE_FORMAT(NOW(), '%%Y-%%m-01')",
            merchant_id)["total"]

        gmv = Decimal(str(monthly_gmv))

        # 阶梯：GMV 越高费率越低
        tiers = [
            (Decimal("1000000"), Decimal("0.8")),   # 100 万以上 → 8 折
            (Decimal("500000"), Decimal("0.9")),     # 50 万以上 → 9 折
            (Decimal("100000"), Decimal("0.95")),    # 10 万以上 → 95 折
        ]

        discount = Decimal("1.0")
        for threshold, tier_discount in tiers:
            if gmv >= threshold:
                discount = tier_discount
                break

        return round(base_rate * float(discount), 2)

    def _get_promo_rate(self, merchant_id, category):
        """获取促销费率"""
        promo = self.db.query_one(
            "SELECT * FROM commission_promotions "
            "WHERE (merchant_id = %s OR category = %s) "
            "AND start_date <= NOW() AND end_date >= NOW() "
            "ORDER BY discount_rate ASC LIMIT 1",
            merchant_id, category)

        if promo:
            return promo["discount_rate"]
        return float('inf')  # 无促销 → 不影响最终费率

    def _get_contract_rate(self, merchant_id):
        """获取特殊协议费率"""
        contract = self.db.query_one(
            "SELECT * FROM merchant_commission_contracts "
            "WHERE merchant_id = %s AND status = 'active' "
            "AND effective_date <= NOW() AND expiry_date >= NOW()",
            merchant_id)
        return float(contract["rate"]) if contract else None
```

### 3.9 结算审批工作流

```python
class SettlementApprovalWorkflow:
    """结算审批：大额审批 → 异常审批 → 自动审批"""

    APPROVAL_THRESHOLDS = {
        "auto_approve": Decimal("50000"),    # 5 万以下自动审批
        "manager_approve": Decimal("200000"), # 5-20 万需经理审批
        "director_approve": Decimal("1000000"), # 20-100 万需总监审批
        # 100 万以上需 VP 审批
    }

    def submit_for_approval(self, bill_id):
        """提交审批"""
        bill = self.db.get_settlement_bill(bill_id)
        amount = Decimal(bill["settlement_amount"])

        # 1. 自动审批检查
        if amount <= self.APPROVAL_THRESHOLDS["auto_approve"]:
            # 5 万以下 → 自动审批
            self.db.update("settlement_bills",
                {"status": "approved", "approved_at": now(),
                 "approval_type": "auto"},
                {"bill_id": bill_id})
            return {"status": "auto_approved", "bill_id": bill_id}

        # 2. 确定审批级别
        if amount <= self.APPROVAL_THRESHOLDS["manager_approve"]:
            approval_level = "manager"
        elif amount <= self.APPROVAL_THRESHOLDS["director_approve"]:
            approval_level = "director"
        else:
            approval_level = "vp"

        # 3. 创建审批流程
        approval_id = str(uuid4())
        self.db.insert("settlement_approvals", {
            "approval_id": approval_id,
            "bill_id": bill_id,
            "merchant_id": bill["merchant_id"],
            "amount": str(amount),
            "required_level": approval_level,
            "status": "pending",
            "created_at": now()
        })

        # 4. 通知审批人
        approvers = self._get_approvers(approval_level)
        for approver in approvers:
            self.notification.send(approver["id"],
                f"待审批结算: 商家 {bill['merchant_id']}, 金额 {amount} 元, "
                f"级别 {approval_level}")

        return {"status": "pending_approval", "approval_id": approval_id,
                "required_level": approval_level}

    def approve(self, approval_id, approver_id, decision, note=None):
        """审批结算"""
        approval = self.db.get_approval(approval_id)

        # 1. 验证审批人权限
        approver = self.db.get_user(approver_id)
        if approver["level"] < self._level_to_int(approval["required_level"]):
            raise PermissionDeniedError("审批权限不足")

        if approval["status"] != "pending":
            return {"status": "already_processed"}

        # 2. 更新审批状态
        self.db.update("settlement_approvals",
            {"status": decision, "approver_id": approver_id,
             "approval_note": note, "approved_at": now()},
            {"approval_id": approval_id})

        # 3. 更新账单状态
        if decision == "approved":
            self.db.update("settlement_bills",
                {"status": "approved", "approved_at": now(),
                 "approval_type": "manual"},
                {"bill_id": approval["bill_id"]})
        elif decision == "rejected":
            self.db.update("settlement_bills",
                {"status": "rejected", "rejection_reason": note},
                {"bill_id": approval["bill_id"]})

        # 4. 通知商家
        self.notification.send(approval["merchant_id"],
            f"结算{'已批准' if decision == 'approved' else '被驳回'}")

        return {"approval_id": approval_id, "decision": decision}

    def _level_to_int(self, level):
        """级别映射"""
        return {"manager": 1, "director": 2, "vp": 3}.get(level, 0)

    def _get_approvers(self, level):
        """获取审批人列表"""
        return self.db.query(
            "SELECT id FROM users WHERE level >= %s AND role = 'approver'",
            self._level_to_int(level))
```

### 3.10 结算异常检测

```python
class SettlementAnomalyDetector:
    """结算异常检测：金额异常 → 频率异常 → 模式异常"""

    def detect_anomalies(self, merchant_id):
        """检测商家结算异常"""
        anomalies = []

        # 1. 单笔金额异常
        recent = self.db.query(
            "SELECT * FROM settlement_ledger "
            "WHERE merchant_id = %s AND created_at > NOW() - INTERVAL 30 DAY "
            "ORDER BY created_at DESC LIMIT 100", merchant_id)

        if recent:
            amounts = [Decimal(r["merchant_income"]) for r in recent]
            avg = statistics.mean(amounts)
            std = statistics.stdev(amounts) if len(amounts) > 1 else 0

            for r in recent[-10:]:  # 检查最近 10 笔
                income = Decimal(r["merchant_income"])
                if std > 0 and abs(income - avg) > 3 * std:
                    anomalies.append({
                        "type": "unusual_amount",
                        "severity": "high",
                        "order_id": r["order_id"],
                        "amount": str(income),
                        "expected_range": f"{avg - 2*std:.2f} ~ {avg + 2*std:.2f}",
                        "message": f"单笔金额异常: {income} 远超均值 {avg:.2f}"
                    })

        # 2. 结算频率异常（短时间大量订单）
        hourly_count = self.db.query_one(
            "SELECT COUNT(*) as count FROM settlement_ledger "
            "WHERE merchant_id = %s AND created_at > NOW() - INTERVAL 1 HOUR",
            merchant_id)["count"]

        avg_hourly = self.db.query_one(
            "SELECT AVG(hourly_count) as avg FROM ("
            "  SELECT COUNT(*) as hourly_count, DATE_FORMAT(created_at, '%%Y-%%m-%%d %%H:00') as hour "
            "  FROM settlement_ledger WHERE merchant_id = %s "
            "  AND created_at > NOW() - INTERVAL 30 DAY "
            "  GROUP BY hour) sub",
            merchant_id)["avg"] or 0

        if hourly_count > avg_hourly * 5 and hourly_count > 50:
            anomalies.append({
                "type": "frequency_spike",
                "severity": "high",
                "current_hourly": hourly_count,
                "average_hourly": round(avg_hourly, 1),
                "message": f"结算频率异常: 当前 {hourly_count} 笔/小时, 均值 {avg_hourly:.1f}"
            })

        # 3. 退款率异常
        total_orders = self.db.count("settlement_ledger",
            merchant_id=merchant_id,
            created_at__gte=now()-timedelta(days=7))
        refund_count = self.db.count("refund_offset_log",
            merchant_id=merchant_id,
            created_at__gte=now()-timedelta(days=7))

        refund_rate = refund_count / max(total_orders, 1)
        if refund_rate > 0.2:
            anomalies.append({
                "type": "high_refund_rate",
                "severity": "medium",
                "refund_rate": round(refund_rate, 3),
                "message": f"退款率异常: {refund_rate:.1%} (7天内)"
            })

        # 4. 佣金比例异常（佣金占比远低于预期）
        total_income = sum(Decimal(r["merchant_income"]) for r in recent) if recent else 0
        total_commission = sum(Decimal(r["commission"]) for r in recent) if recent else 0

        if total_income > 0:
            commission_ratio = total_commission / total_income
            expected_ratio = Decimal("0.05")  # 预期 5% 左右
            if commission_ratio < expected_ratio * Decimal("0.5"):
                anomalies.append({
                    "type": "low_commission_ratio",
                    "severity": "high",
                    "actual_ratio": round(float(commission_ratio), 4),
                    "expected_ratio": round(float(expected_ratio), 4),
                    "message": f"佣金比例异常低: {commission_ratio:.2%}"
                })

        return {"merchant_id": merchant_id,
                "anomaly_count": len(anomalies),
                "anomalies": anomalies}
```

### 3.11 商家提现申请

```python
class MerchantWithdrawalService:
    """商家提现：申请 → 审核 → 打款 → 到账确认"""

    WITHDRAWAL_LIMITS = {
        "min_amount": Decimal("100"),
        "max_single": Decimal("500000"),
        "max_daily": Decimal("2000000"),
        "max_monthly": Decimal("10000000"),
    }

    def apply_withdrawal(self, merchant_id, amount, bank_account_id):
        """申请提现"""
        amount = Decimal(str(amount))

        # 1. 检查最低提现金额
        if amount < self.WITHDRAWAL_LIMITS["min_amount"]:
            return {"status": "below_minimum",
                    "minimum": str(self.WITHDRAWAL_LIMITS["min_amount"])}

        # 2. 检查单笔上限
        if amount > self.WITHDRAWAL_LIMITS["max_single"]:
            return {"status": "exceed_single_limit",
                    "limit": str(self.WITHDRAWAL_LIMITS["max_single"])}

        # 3. 检查可提现余额
        available = self._get_available_balance(merchant_id)
        if amount > available:
            return {"status": "insufficient_balance",
                    "available": str(available), "requested": str(amount)}

        # 4. 检查日/月限额
        today_withdrawn = self._get_today_withdrawn(merchant_id)
        if today_withdrawn + amount > self.WITHDRAWAL_LIMITS["max_daily"]:
            return {"status": "exceed_daily_limit",
                    "remaining": str(self.WITHDRAWAL_LIMITS["max_daily"] - today_withdrawn)}

        month_withdrawn = self._get_month_withdrawn(merchant_id)
        if month_withdrawn + amount > self.WITHDRAWAL_LIMITS["max_monthly"]:
            return {"status": "exceed_monthly_limit"}

        # 5. 检查银行账户
        bank = self.db.get_bank_account(bank_account_id)
        if not bank or bank["merchant_id"] != merchant_id:
            return {"status": "invalid_bank_account"}

        # 6. 冻结金额
        withdrawal_id = str(uuid4())
        self.db.insert("merchant_withdrawals", {
            "withdrawal_id": withdrawal_id,
            "merchant_id": merchant_id,
            "amount": str(amount),
            "bank_account_id": bank_account_id,
            "status": "pending",
            "created_at": now()
        })

        # 冻结余额
        self.redis.hincrbyfloat(f"merchant_pending:{merchant_id}",
            "frozen", float(amount))

        # 7. 通知审核
        self.notification.send("finance_team",
            f"新提现申请: 商家 {merchant_id}, 金额 {amount} 元")

        return {"withdrawal_id": withdrawal_id, "status": "pending",
                "amount": str(amount)}

    def approve_withdrawal(self, withdrawal_id, approver_id):
        """审核通过提现"""
        withdrawal = self.db.get_withdrawal(withdrawal_id)

        if withdrawal["status"] != "pending":
            return {"status": "invalid_state"}

        # 1. 执行打款
        bank = self.db.get_bank_account(withdrawal["bank_account_id"])
        amount = Decimal(withdrawal["amount"])

        payout = self.payment_gateway.transfer(
            amount=float(amount),
            bank_account=bank["account_number"],
            bank_name=bank["bank_name"],
            reference_id=withdrawal_id)

        if payout["success"]:
            self.db.update("merchant_withdrawals",
                {"status": "completed", "completed_at": now(),
                 "payment_id": payout["transaction_id"],
                 "approved_by": approver_id},
                {"withdrawal_id": withdrawal_id})
            return {"status": "completed", "transaction_id": payout["transaction_id"]}
        else:
            self.db.update("merchant_withdrawals",
                {"status": "payout_failed", "failure_reason": payout.get("error")},
                {"withdrawal_id": withdrawal_id})
            return {"status": "payout_failed", "reason": payout.get("error")}

    def _get_available_balance(self, merchant_id):
        """获取可提现余额"""
        pending = self.db.query_one(
            "SELECT COALESCE(SUM(merchant_income), 0) as total "
            "FROM settlement_ledger "
            "WHERE merchant_id = %s AND status = 'pending'",
            merchant_id)
        frozen = self.db.query_one(
            "SELECT COALESCE(SUM(amount), 0) as total "
            "FROM merchant_withdrawals "
            "WHERE merchant_id = %s AND status = 'pending'",
            merchant_id)
        deductions = self.db.query_one(
            "SELECT COALESCE(SUM(amount), 0) as total "
            "FROM pending_deductions "
            "WHERE merchant_id = %s AND status = 'pending'",
            merchant_id)

        available = Decimal(pending["total"]) - Decimal(frozen["total"]) - Decimal(deductions["total"])
        return max(Decimal("0"), available)

    def _get_today_withdrawn(self, merchant_id):
        """获取今日已提现金额"""
        result = self.db.query_one(
            "SELECT COALESCE(SUM(amount), 0) as total "
            "FROM merchant_withdrawals "
            "WHERE merchant_id = %s AND status = 'completed' "
            "AND DATE(completed_at) = CURDATE()", merchant_id)
        return Decimal(result["total"])

    def _get_month_withdrawn(self, merchant_id):
        """获取本月已提现金额"""
        result = self.db.query_one(
            "SELECT COALESCE(SUM(amount), 0) as total "
            "FROM merchant_withdrawals "
            "WHERE merchant_id = %s AND status = 'completed' "
            "AND DATE_FORMAT(completed_at, '%%Y-%%m') = DATE_FORMAT(NOW(), '%%Y-%%m')",
            merchant_id)
        return Decimal(result["total"])
```

## 七、监控与告警

### 关键指标

| 指标 | 告警阈值 | 说明 |
|------|---------|------|
| 结算计算错误率 | > 0.01% | 金额不一致的订单比例 |
| 打款成功率 | < 99% | 打款失败比例 |
| 保证金不足商家数 | > 5 | 保证金低于要求的商家数 |
| 结算延迟 | > 2 小时 | 超过 SLA 的结算批次数 |
| 退款冲抵失败率 | > 1% | 冲抵失败的退款比例 |

### 监控实现

```python
class SettlementMonitorService:
    """结算监控：关键指标 + 异常告警 + 看板"""

    def get_dashboard_metrics(self):
        """获取结算看板指标"""
        today = now().date()

        # 今日结算概况
        today_settled = self.db.query_one(
            "SELECT COUNT(*) as count, COALESCE(SUM(settlement_amount), 0) as total "
            "FROM settlement_bills WHERE DATE(generated_at) = %s", today)

        today_paid = self.db.query_one(
            "SELECT COUNT(*) as count, COALESCE(SUM(settlement_amount), 0) as total "
            "FROM settlement_bills WHERE DATE(paid_at) = %s AND status = 'paid'", today)

        # 待处理
        pending_bills = self.db.count("settlement_bills", status="generated")
        pending_payout_failed = self.db.count("settlement_bills", status="payout_failed")

        # 保证金预警
        low_deposit_count = self.db.query_one(
            "SELECT COUNT(*) as count FROM merchant_security_deposits "
            "WHERE balance < required_amount * 0.3")["count"]

        # 待审批
        pending_approvals = self.db.count("settlement_approvals", status="pending")

        return {
            "date": today.isoformat(),
            "today_settled": {
                "count": today_settled["count"],
                "amount": today_settled["total"]
            },
            "today_paid": {
                "count": today_paid["count"],
                "amount": today_paid["total"]
            },
            "pending_bills": pending_bills,
            "payout_failed": pending_payout_failed,
            "low_deposit_merchants": low_deposit_count,
            "pending_approvals": pending_approvals
        }

    def check_health(self):
        """健康检查"""
        issues = []

        # 检查积压
        pending = self.db.count("settlement_ledger", status="pending")
        if pending > 100000:
            issues.append(f"待结算记录积压: {pending} 条")

        # 检查打款失败
        failed = self.db.count("settlement_bills",
            status="payout_failed",
            generated_at__gte=now()-timedelta(days=1))
        if failed > 10:
            issues.append(f"近 24 小时打款失败 {failed} 笔")

        # 检查待扣除积压
        pending_deductions = self.db.count("pending_deductions", status="pending")
        if pending_deductions > 1000:
            issues.append(f"待扣除项积压: {pending_deductions} 条")

        return {"healthy": len(issues) == 0, "issues": issues}
```

## 八、更多异常场景

### 场景 9：结算批处理中断

```
触发：周期结算执行到一半 → 数据库连接断开 → 部分记录已标记 settled → 账单未生成
检测：
  1. 存在 settled 记录但无 bill_id → 中断
  2. 结算任务状态长时间为 running → 超时
处理：
  1. 事务包裹整个结算批次（失败全部回滚）
  2. 中断后自动重试
  3. 修复孤立记录（有 bill_id 的保留，无 bill_id 的恢复为 pending）
预防：事务原子性 + 自动重试 + 孤立记录修复
```

### 场景 10：佣金费率配置错误

```
触发：运营误将食品类佣金从 8% 设为 0.8% → 商家收入虚高 → 平台收入损失
检测：
  1. 费率变更幅度 > 50% → 可能配置错误
  2. 佣金比例异常低 → 费率配置问题
处理：
  1. 费率变更需审批
  2. 费率变更限制幅度（单次不超过 3%）
  3. 已结算的追溯修正
预防：审批流程 + 变更幅度限制 + 追溯修正
```

### 场景 11：商家欺诈刷单

```
触发：商家自买自卖 → 订单量虚高 → 结算后退款 → 平台佣金损失
检测：
  1. 同一 IP 下买家和卖家 → 刷单嫌疑
  2. 高退款率商家 → 欺诈风险
  3. 结算后立即退款 → 异常模式
处理：
  1. 结算延迟（T+7 而非 T+1）
  2. 高风险商家结算冻结
  3. 人工审核异常订单
预防：延迟结算 + 风险冻结 + 人工审核
```

### 3.12 多币种结算支持

```python
class MultiCurrencySettlementService:
    """多币种结算：汇率转换 + 币种对账 + 汇率锁定"""

    SUPPORTED_CURRENCIES = {
        "CNY": {"name": "人民币", "symbol": "¥", "decimal_places": 2},
        "USD": {"name": "美元", "symbol": "$", "decimal_places": 2},
        "EUR": {"name": "欧元", "symbol": "€", "decimal_places": 2},
        "JPY": {"name": "日元", "symbol": "¥", "decimal_places": 0},
        "GBP": {"name": "英镑", "symbol": "£", "decimal_places": 2},
    }

    def settle_in_merchant_currency(self, settlement_detail, merchant_settlement_currency):
        """以商家结算币种结算"""
        order_currency = settlement_detail.get("currency", "CNY")

        if order_currency == merchant_settlement_currency:
            # 同币种 → 无需转换
            return settlement_detail

        # 1. 获取汇率（锁定汇率，24 小时有效）
        rate = self._get_locked_rate(order_currency, merchant_settlement_currency)

        # 2. 转换所有金额
        converted = {
            "order_id": settlement_detail["order_id"],
            "merchant_id": settlement_detail["merchant_id"],
            "original_currency": order_currency,
            "settlement_currency": merchant_settlement_currency,
            "exchange_rate": str(rate),
            "rate_locked_at": now().isoformat(),
        }

        for field in ["item_amount", "shipping_fee", "commission",
                      "merchant_income", "actual_payment"]:
            if field in settlement_detail:
                original = Decimal(str(settlement_detail[field]))
                converted_amount = (original * rate).quantize(
                    Decimal("0.01"), rounding=ROUND_HALF_UP)
                converted[field] = str(converted_amount)
                converted[f"{field}_original"] = settlement_detail[field]

        # 3. 优惠券分摊也需转换
        if "coupon_details" in settlement_detail:
            converted_coupons = []
            for coupon in settlement_detail["coupon_details"]:
                converted_coupon = dict(coupon)
                for burden_field in ["amount", "merchant_burden", "platform_burden"]:
                    if burden_field in coupon:
                        original = Decimal(str(coupon[burden_field]))
                        converted_coupon[burden_field] = str(
                            (original * rate).quantize(
                                Decimal("0.01"), rounding=ROUND_HALF_UP))
                converted_coupons.append(converted_coupon)
            converted["coupon_details"] = converted_coupons

        return converted

    def _get_locked_rate(self, from_currency, to_currency):
        """获取锁定汇率"""
        cache_key = f"fx_rate:{from_currency}:{to_currency}"

        cached = self.redis.get(cache_key)
        if cached:
            rate_data = json.loads(cached)
            # 检查是否在有效期内
            locked_at = datetime.fromisoformat(rate_data["locked_at"])
            if (now() - locked_at).total_seconds() < 86400:  # 24 小时
                return Decimal(rate_data["rate"])

        # 从外汇 API 获取实时汇率
        rate = self.forex_api.get_rate(from_currency, to_currency)

        # 锁定汇率
        self.redis.setex(cache_key, 86400, json.dumps({
            "rate": str(rate),
            "from": from_currency,
            "to": to_currency,
            "locked_at": now().isoformat()
        }))

        return Decimal(str(rate))

    def reconcile_currency_difference(self, merchant_id, period_start, period_end):
        """币种差异对账"""
        # 找出有汇率转换的结算记录
        records = self.db.query(
            "SELECT * FROM settlement_ledger "
            "WHERE merchant_id = %s "
            "AND original_currency != settlement_currency "
            "AND created_at BETWEEN %s AND %s",
            merchant_id, period_start, period_end)

        total_fx_difference = Decimal("0")
        for record in records:
            # 重新用当前汇率计算
            current_rate = self.forex_api.get_rate(
                record["original_currency"], record["settlement_currency"])
            locked_rate = Decimal(record["exchange_rate"])

            # 汇率差异
            original_income = Decimal(record["merchant_income_original"])
            settled_income = Decimal(record["merchant_income"])
            current_income = (original_income * current_rate).quantize(
                Decimal("0.01"), rounding=ROUND_HALF_UP)

            fx_diff = settled_income - current_income
            total_fx_difference += fx_diff

        return {
            "merchant_id": merchant_id,
            "period": f"{period_start} ~ {period_end}",
            "records_with_fx": len(records),
            "total_fx_difference": str(total_fx_difference),
            "fx_impact": "gain" if total_fx_difference > 0 else "loss"
        }
```

### 3.13 结算数据归档

```python
class SettlementArchiveService:
    """结算归档：冷热分离 + 归档策略 + 归档查询"""

    ARCHIVE_RULES = {
        "settlement_ledger": {"hot_days": 90, "warm_days": 365, "cold_days": 2555},
        "settlement_bills": {"hot_days": 180, "warm_days": 730, "cold_days": 3650},
        "refund_offset_log": {"hot_days": 90, "warm_days": 365, "cold_days": 1825},
        "deposit_transactions": {"hot_days": 180, "warm_days": 730, "cold_days": 3650},
    }

    def archive_settlement_data(self):
        """执行归档"""
        archived = {}

        for table, rules in self.ARCHIVE_RULES.items():
            hot_cutoff = now() - timedelta(days=rules["hot_days"])

            # 1. 从热库迁移到温库
            count = self._move_to_warm_storage(table, hot_cutoff)
            archived[table] = {"moved_to_warm": count}

            # 2. 从温库迁移到冷库
            warm_cutoff = now() - timedelta(days=rules["warm_days"])
            cold_count = self._move_to_cold_storage(table, warm_cutoff)
            archived[table]["moved_to_cold"] = cold_count

        return {"archived_at": now().isoformat(), "details": archived}

    def _move_to_warm_storage(self, table, cutoff_date):
        """迁移到温存储（同集群不同库）"""
        date_column = "created_at" if table != "settlement_bills" else "generated_at"

        # 批量迁移
        batch_size = 10000
        total_moved = 0

        while True:
            records = self.db.query(
                f"SELECT * FROM {table} "
                f"WHERE {date_column} < %s AND storage_tier = 'hot' "
                f"LIMIT %s", cutoff_date, batch_size)

            if not records:
                break

            # 写入温库
            for record in records:
                self.warm_db.insert(table, record)

            # 标记为已迁移
            ids = [r["id"] if "id" in r else r.get("ledger_id", r.get("bill_id")) for r in records]
            self.db.execute(
                f"UPDATE {table} SET storage_tier = 'warm' "
                f"WHERE id IN %s", tuple(ids))

            total_moved += len(records)

        return total_moved

    def _move_to_cold_storage(self, table, cutoff_date):
        """迁移到冷存储（对象存储，Parquet 格式）"""
        date_column = "created_at" if table != "settlement_bills" else "generated_at"

        records = self.db.query(
            f"SELECT * FROM {table} "
            f"WHERE {date_column} < %s AND storage_tier = 'warm'",
            cutoff_date)

        if not records:
            return 0

        # 转换为 Parquet 并上传
        import pandas as pd
        df = pd.DataFrame(records)
        partition_key = f"settlement_archive/{table}/{cutoff_date.strftime('%Y%m')}"
        self.object_storage.upload_dataframe(partition_key, df)

        # 标记为冷存储
        ids = [r.get("id", r.get("ledger_id", r.get("bill_id"))) for r in records]
        self.db.execute(
            f"UPDATE {table} SET storage_tier = 'cold' "
            f"WHERE id IN %s", tuple(ids))

        return len(records)

    def query_archived_data(self, table, query_params):
        """查询归档数据（自动路由到对应存储层）"""
        date = query_params.get("date_range", {}).get("start")

        if not date:
            return self.db.query(f"SELECT * FROM {table} LIMIT 100")

        # 根据日期路由到不同存储
        hot_days = self.ARCHIVE_RULES.get(table, {}).get("hot_days", 90)
        warm_days = self.ARCHIVE_RULES.get(table, {}).get("warm_days", 365)

        query_date = datetime.fromisoformat(date)
        days_ago = (now() - query_date).days

        if days_ago <= hot_days:
            return self.db.query(f"SELECT * FROM {table} WHERE ...", query_params)
        elif days_ago <= warm_days:
            return self.warm_db.query(f"SELECT * FROM {table} WHERE ...", query_params)
        else:
            # 冷存储 → 从 Parquet 文件读取
            partition_key = f"settlement_archive/{table}/{query_date.strftime('%Y%m')}"
            return self.object_storage.query_parquet(partition_key, query_params)
```

### 3.14 平台收入统计

```python
class PlatformRevenueService:
    """平台收入统计：佣金收入 + 服务费 + 利息收入 + 分月/分类统计"""

    def get_revenue_summary(self, period_start, period_end):
        """获取平台收入汇总"""
        # 1. 佣金收入
        commission = self.db.query_one(
            "SELECT COALESCE(SUM(commission), 0) as total "
            "FROM settlement_ledger "
            "WHERE created_at BETWEEN %s AND %s AND status != 'cancelled'",
            period_start, period_end)["total"]

        # 2. 佣金退回（退款时退还给商家的佣金）
        commission_return = self.db.query_one(
            "SELECT COALESCE(SUM(refund_commission_return), 0) as total "
            "FROM refund_offset_log "
            "WHERE created_at BETWEEN %s AND %s",
            period_start, period_end)["total"]

        # 3. 净佣金收入
        net_commission = Decimal(commission) - Decimal(commission_return)

        # 4. 按品类统计
        by_category = self.db.query(
            "SELECT m.category, "
            "COALESCE(SUM(sl.commission), 0) as total_commission, "
            "COUNT(*) as order_count "
            "FROM settlement_ledger sl "
            "JOIN merchants m ON sl.merchant_id = m.id "
            "WHERE sl.created_at BETWEEN %s AND %s AND sl.status != 'cancelled' "
            "GROUP BY m.category ORDER BY total_commission DESC",
            period_start, period_end)

        # 5. 按日统计趋势
        daily_trend = self.db.query(
            "SELECT DATE(created_at) as date, "
            "COALESCE(SUM(commission), 0) as daily_commission, "
            "COUNT(*) as daily_orders "
            "FROM settlement_ledger "
            "WHERE created_at BETWEEN %s AND %s AND status != 'cancelled' "
            "GROUP BY DATE(created_at) ORDER BY date",
            period_start, period_end)

        # 6. 平台券补贴（平台承担的优惠券金额）
        platform_coupon_subsidy = self.db.query_one(
            "SELECT COALESCE(SUM(JSON_EXTRACT(coupon.value, '$.platform_burden')), 0) as total "
            "FROM settlement_ledger, JSON_TABLE(coupon_details, '$[*]' COLUMNS("
            "  value VARCHAR(200) PATH '$')) AS coupon "
            "WHERE created_at BETWEEN %s AND %s",
            period_start, period_end)

        return {
            "period": f"{period_start} ~ {period_end}",
            "gross_commission": commission,
            "commission_return": commission_return,
            "net_commission": str(net_commission),
            "by_category": by_category,
            "daily_trend": daily_trend,
            "platform_coupon_subsidy": platform_coupon_subsidy.get("total", "0") if platform_coupon_subsidy else "0"
        }

    def get_merchant_ranking(self, period_start, period_end, limit=100):
        """商家结算排名"""
        return self.db.query(
            "SELECT sl.merchant_id, m.name, "
            "COUNT(*) as order_count, "
            "COALESCE(SUM(sl.merchant_income), 0) as total_income, "
            "COALESCE(SUM(sl.commission), 0) as total_commission "
            "FROM settlement_ledger sl "
            "JOIN merchants m ON sl.merchant_id = m.id "
            "WHERE sl.created_at BETWEEN %s AND %s AND sl.status != 'cancelled' "
            "GROUP BY sl.merchant_id, m.name "
            "ORDER BY total_income DESC LIMIT %s",
            period_start, period_end, limit)
```

## 九、系统设计总结

### 数据流全景

```
订单完成
  │
  ├── 结算计算引擎 (SettlementCalculationEngine)
  │     ├── 优惠券分摊 (_allocate_coupons)
  │     ├── 佣金计算 (_get_commission_rate)
  │     └── 金额验证 (Decimal 精度)
  │
  ├── 待结算账本 (SettlementLedgerService)
  │     ├── 实时入账 (record_settlement)
  │     └── 退款冲抵 (handle_refund_offset)
  │
  ├── 周期结算 (SettlementCycleService)
  │     ├── 确定结算周期 (_get_settlement_period)
  │     ├── 生成账单 (execute_settlement_cycle)
  │     └── 执行打款 (execute_payout)
  │
  ├── 保证金管理 (MerchantSecurityDepositService)
  │     ├── 缴纳/扣除/预警
  │     └── 健康检查 (check_deposit_health)
  │
  ├── 佣金规则引擎 (CommissionRuleEngine)
  │     ├── 基础费率 + 阶梯费率 + 促销费率
  │     └── 特殊协议费率
  │
  ├── 审批工作流 (SettlementApprovalWorkflow)
  │     ├── 自动审批 / 人工审批
  │     └── 分级审批 (manager/director/vp)
  │
  ├── 提现管理 (MerchantWithdrawalService)
  │     ├── 限额检查 + 余额验证
  │     └── 审核 + 打款
  │
  ├── 税费计算 (SettlementTaxService)
  │     ├── 增值税 + 代扣税
  │     └── 跨境税务处理
  │
  ├── 多币种 (MultiCurrencySettlementService)
  │     ├── 汇率锁定 + 币种转换
  │     └── 币种差异对账
  │
  ├── 异常检测 (SettlementAnomalyDetector)
  │     ├── 金额/频率/退款率/佣金比异常
  │     └── 实时检测
  │
  ├── 对账 (SettlementReconciliationService)
  │     ├── 日粒度对账
  │     └── 差异排查
  │
  ├── 归档 (SettlementArchiveService)
  │     ├── 冷热分离
  │     └── Parquet 存储
  │
  └── 监控 (SettlementMonitorService)
        ├── 看板指标
        └── 健康检查
```

### 关键设计决策

| 决策 | 选择 | 原因 |
|------|------|------|
| 金额类型 | Decimal | 浮点误差不可接受，Decimal 精确到分 |
| 结算原子性 | 事务 + 状态机 | 防止部分结算、重复结算 |
| 退款冲抵 | 优先级链（待结算→保证金→下期扣除） | 确保资金安全 |
| 汇率处理 | 24 小时锁定 | 避免结算周期内汇率波动 |
| 审批级别 | 按金额分级 | 小额自动、大额人工，平衡效率与风险 |
| 存储 | 冷热分离 + Parquet | 结算数据量大，历史数据归档降成本 |

### 3.15 商家评级与费率动态调整

```python
class MerchantRatingService:
    """商家评级：综合评分 → 费率调整 → 风险预警"""

    RATING_DIMENSIONS = {
        "order_completion": {"weight": 0.3, "max_score": 100},
        "complaint_rate": {"weight": 0.25, "max_score": 100},
        "response_time": {"weight": 0.2, "max_score": 100},
        "product_quality": {"weight": 0.15, "max_score": 100},
        "logistics_speed": {"weight": 0.1, "max_score": 100},
    }

    RATING_LEVELS = {
        "S": {"min_score": 90, "commission_discount": 20},
        "A": {"min_score": 80, "commission_discount": 10},
        "B": {"min_score": 60, "commission_discount": 0},
        "C": {"min_score": 40, "commission_discount": -5},
        "D": {"min_score": 0, "commission_discount": -10},
    }

    def calculate_merchant_rating(self, merchant_id):
        """计算商家综合评级"""
        scores = {}

        # 1. 订单完成率
        total_orders = self.db.count("orders", merchant_id=merchant_id,
            created_at__gte=now()-timedelta(days=30))
        completed_orders = self.db.count("orders", merchant_id=merchant_id,
            status="completed", created_at__gte=now()-timedelta(days=30))
        scores["order_completion"] = round(completed_orders / max(total_orders, 1) * 100, 1)

        # 2. 投诉率（越低越好 → 反转）
        complaints = self.db.count("complaints", merchant_id=merchant_id,
            created_at__gte=now()-timedelta(days=30))
        complaint_rate = complaints / max(total_orders, 1)
        scores["complaint_rate"] = round(max(0, 100 - complaint_rate * 1000), 1)

        # 3. 响应时间
        avg_response = self.db.query_one(
            "SELECT AVG(TIMESTAMPDIFF(MINUTE, created_at, first_response_at)) as avg_min "
            "FROM customer_messages "
            "WHERE merchant_id = %s AND created_at > NOW() - INTERVAL 30 DAY",
            merchant_id)["avg_min"] or 60
        scores["response_time"] = round(max(0, 100 - avg_response), 1)

        # 4. 商品质量（好评率）
        good_reviews = self.db.count("product_reviews", merchant_id=merchant_id,
            rating__gte=4, created_at__gte=now()-timedelta(days=30))
        total_reviews = self.db.count("product_reviews", merchant_id=merchant_id,
            created_at__gte=now()-timedelta(days=30))
        scores["product_quality"] = round(good_reviews / max(total_reviews, 1) * 100, 1)

        # 5. 物流速度
        avg_delivery = self.db.query_one(
            "SELECT AVG(TIMESTAMPDIFF(HOUR, shipped_at, delivered_at)) as avg_hours "
            "FROM orders "
            "WHERE merchant_id = %s AND status = 'delivered' "
            "AND delivered_at > NOW() - INTERVAL 30 DAY",
            merchant_id)["avg_hours"] or 72
        scores["logistics_speed"] = round(max(0, 100 - avg_delivery), 1)

        # 加权总分
        overall = sum(scores[k] * self.RATING_DIMENSIONS[k]["weight"]
                      for k in scores)

        # 确定等级
        level = "D"
        for level_name, config in sorted(self.RATING_LEVELS.items(),
                                          key=lambda x: -x[1]["min_score"]):
            if overall >= config["min_score"]:
                level = level_name
                break

        # 保存评级
        self.db.upsert("merchant_ratings", {
            "merchant_id": merchant_id,
            "overall_score": round(overall, 1),
            "level": level,
            "dimension_scores": json.dumps(scores),
            "rated_at": now()
        }, conflict_columns=["merchant_id"])

        return {"merchant_id": merchant_id, "overall_score": round(overall, 1),
                "level": level, "scores": scores}

    def adjust_commission_rate(self, merchant_id):
        """根据评级调整佣金费率"""
        rating = self.db.query_one(
            "SELECT * FROM merchant_ratings WHERE merchant_id = %s",
            merchant_id)

        if not rating:
            return {"status": "no_rating"}

        level = rating["level"]
        config = self.RATING_LEVELS[level]
        discount = config["commission_discount"]

        # 获取当前费率
        merchant = self.db.get_merchant(merchant_id)
        current_rate = Decimal(str(merchant.get("commission_rate", 5.0)))

        # 计算新费率
        if discount > 0:
            new_rate = current_rate * (Decimal("100") - Decimal(str(discount))) / Decimal("100")
        elif discount < 0:
            new_rate = current_rate * (Decimal("100") - Decimal(str(discount))) / Decimal("100")
        else:
            new_rate = current_rate

        new_rate = new_rate.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)

        # 限制费率范围
        new_rate = max(Decimal("1.00"), min(Decimal("15.00"), new_rate))

        # 更新费率
        self.db.update("merchants",
            {"commission_rate": str(new_rate), "rate_adjusted_at": now()},
            {"id": merchant_id})

        # 记录费率变更
        self.db.insert("commission_rate_changes", {
            "change_id": str(uuid4()),
            "merchant_id": merchant_id,
            "old_rate": str(current_rate),
            "new_rate": str(new_rate),
            "reason": f"评级调整: {level}, 折扣 {discount}%",
            "changed_at": now()
        })

        # 通知商家
        if discount > 0:
            self.notification.send(merchant_id,
                f"评级 {level} 级，佣金费率降至 {new_rate}%")
        elif discount < 0:
            self.notification.send(merchant_id,
                f"评级 {level} 级，佣金费率调整为 {new_rate}%，请提升服务质量")

        return {"merchant_id": merchant_id, "level": level,
                "old_rate": str(current_rate), "new_rate": str(new_rate)}

    def suspend_merchant(self, merchant_id, reason):
        """暂停商家（评级过低）"""
        rating = self.db.query_one(
            "SELECT * FROM merchant_ratings WHERE merchant_id = %s",
            merchant_id)

        if not rating or rating["level"] != "D":
            return {"status": "not_eligible_for_suspension"}

        # 冻结商家
        self.db.update("merchants",
            {"status": "suspended", "suspended_reason": reason,
             "suspended_at": now()},
            {"id": merchant_id})

        # 冻结结算
        self.db.update("settlement_ledger",
            {"status": "frozen"},
            {"merchant_id": merchant_id, "status": "pending"})

        self.notification.send(merchant_id,
            f"商家已暂停: {reason}，请提升服务质量后申请恢复")

        return {"merchant_id": merchant_id, "status": "suspended", "reason": reason}
```

### 3.16 结算数据实时事件驱动

```python
class SettlementEventService:
    """结算事件驱动：Kafka 消费 → 结算处理 → 事件发布"""

    def on_order_completed(self, event):
        """订单完成事件 → 计算结算"""
        order = event["data"]
        order_id = order["order_id"]

        # 1. 幂等检查
        existing = self.redis.setnx(f"settlement_processed:{order_id}", "1")
        if not existing:
            return {"status": "already_processed"}
        self.redis.expire(f"settlement_processed:{order_id}", 86400)

        try:
            # 2. 计算结算
            detail = self.calc_engine.calculate_order_settlement(order)

            # 3. 写入待结算账本
            result = self.ledger.record_settlement(detail)

            # 4. 发布结算完成事件
            self.kafka.produce("settlement.recorded", {
                "order_id": order_id,
                "merchant_id": order["merchant_id"],
                "merchant_income": detail["merchant_income"],
                "commission": detail["commission"],
                "recorded_at": now().isoformat()
            })

            return {"status": "recorded", "order_id": order_id}

        except Exception as e:
            # 失败 → 发送到死信队列
            self.kafka.produce("settlement.failed", {
                "order_id": order_id,
                "error": str(e),
                "original_event": event,
                "failed_at": now().isoformat()
            })
            raise

    def on_refund_created(self, event):
        """退款创建事件 → 处理冲抵"""
        refund = event["data"]

        result = self.ledger.handle_refund_offset({
            "order_id": refund["order_id"],
            "refund_amount": refund["amount"]
        })

        # 发布冲抵事件
        self.kafka.produce("settlement.refund_offset", {
            "order_id": refund["order_id"],
            "offset_method": result.get("offset_method"),
            "merchant_burden": result.get("merchant_burden"),
            "commission_return": result.get("commission_return")
        })

        return result

    def on_merchant_status_changed(self, event):
        """商家状态变更事件 → 冻结/解冻结算"""
        merchant_id = event["data"]["merchant_id"]
        new_status = event["data"]["new_status"]

        if new_status == "suspended":
            # 冻结所有待结算
            self.db.execute(
                "UPDATE settlement_ledger SET status = 'frozen' "
                "WHERE merchant_id = %s AND status = 'pending'",
                merchant_id)
            self.redis.delete(f"merchant_pending:{merchant_id}")

        elif new_status == "active":
            # 解冻
            self.db.execute(
                "UPDATE settlement_ledger SET status = 'pending' "
                "WHERE merchant_id = %s AND status = 'frozen'",
                merchant_id)

        return {"merchant_id": merchant_id, "status": new_status}

    def replay_events(self, topic, start_offset, end_offset):
        """重放事件（恢复用）"""
        messages = self.kafka.consume_range(topic, start_offset, end_offset)

        replayed = 0
        failed = 0

        for msg in messages:
            try:
                if topic == "order.completed":
                    self.on_order_completed(msg)
                elif topic == "refund.created":
                    self.on_refund_created(msg)
                elif topic == "merchant.status_changed":
                    self.on_merchant_status_changed(msg)
                replayed += 1
            except Exception:
                failed += 1

        return {"replayed": replayed, "failed": failed}
```

### 3.17 平台与商家分账

```python
class ProfitSharingService:
    """分账：分账规则 → 分账计算 → 分账执行 → 对账"""

    def create_sharing_rule(self, rule_name, category, splits):
        """创建分账规则"""
        # 验证分账比例总和 = 100%
        total_pct = sum(Decimal(str(s["percentage"])) for s in splits)
        if total_pct != Decimal("100"):
            raise InvalidSplitError(f"分账比例总和 {total_pct}% ≠ 100%")

        rule_id = str(uuid4())
        self.db.insert("profit_sharing_rules", {
            "rule_id": rule_id,
            "name": rule_name,
            "category": category,
            "splits": json.dumps(splits),
            "status": "active",
            "created_at": now()
        })

        return {"rule_id": rule_id, "total_pct": str(total_pct)}

    def calculate_split(self, order_id, amount):
        """计算分账"""
        order = self.db.get_order(order_id)
        category = order.get("category", "default")

        # 1. 查找适用的分账规则
        rule = self.db.query_one(
            "SELECT * FROM profit_sharing_rules "
            "WHERE category = %s AND status = 'active' "
            "ORDER BY created_at DESC LIMIT 1", category)

        if not rule:
            # 默认规则：商家 95% + 平台 5%
            splits = [{"account": "merchant", "percentage": 95},
                      {"account": "platform", "percentage": 5}]
        else:
            splits = json.loads(rule["splits"])

        # 2. 计算各方金额
        amount = Decimal(str(amount))
        split_results = []
        allocated = Decimal("0")

        for i, split in enumerate(splits):
            pct = Decimal(str(split["percentage"]))

            if i == len(splits) - 1:
                # 最后一方：取剩余金额（避免分摊不平）
                split_amount = amount - allocated
            else:
                split_amount = (amount * pct / Decimal("100")).quantize(
                    Decimal("0.01"), rounding=ROUND_HALF_UP)
                allocated += split_amount

            split_results.append({
                "account": split["account"],
                "account_id": split.get("account_id", order["merchant_id"]),
                "percentage": str(pct),
                "amount": str(split_amount)
            })

        return {"order_id": order_id, "total_amount": str(amount),
                "splits": split_results, "rule_id": rule["rule_id"] if rule else None}

    def execute_split(self, order_id, splits):
        """执行分账（转入各方子账户）"""
        for split in splits:
            self.db.insert("profit_split_records", {
                "split_id": str(uuid4()),
                "order_id": order_id,
                "account": split["account"],
                "account_id": split["account_id"],
                "amount": split["amount"],
                "status": "completed",
                "executed_at": now()
            })

            # 更新子账户余额
            self.redis.hincrbyfloat(
                f"account_balance:{split['account_id']}",
                split["account"], float(split["amount"]))

        return {"order_id": order_id, "splits_executed": len(splits)}

    def reconcile_splits(self, order_id):
        """对账：验证分账总额 = 订单金额"""
        records = self.db.query(
            "SELECT * FROM profit_split_records WHERE order_id = %s",
            order_id)

        total_split = sum(Decimal(r["amount"]) for r in records)
        order = self.db.get_order(order_id)
        order_amount = Decimal(str(order["actual_payment"]))

        is_balanced = abs(total_split - order_amount) < Decimal("0.01")

        return {"order_id": order_id, "order_amount": str(order_amount),
                "total_split": str(total_split), "is_balanced": is_balanced}
```

### 3.18 商家自助服务门户

```python
class MerchantSelfServicePortal:
    """商家自助门户：余额查询 + 交易明细 + 报告下载 + 账户管理 + 争议提交"""

    def get_balance_overview(self, merchant_id):
        """获取余额概览"""
        # 待结算
        pending = self.db.query_one(
            "SELECT COALESCE(SUM(merchant_income), 0) as total "
            "FROM settlement_ledger "
            "WHERE merchant_id = %s AND status = 'pending'",
            merchant_id)["total"]

        # 冻结
        frozen = self.db.query_one(
            "SELECT COALESCE(SUM(amount), 0) as total "
            "FROM merchant_withdrawals "
            "WHERE merchant_id = %s AND status = 'pending'",
            merchant_id)["total"]

        # 保证金
        deposit = self.db.query_one(
            "SELECT balance, required_amount FROM merchant_security_deposits "
            "WHERE merchant_id = %s", merchant_id)

        # 待扣除
        deductions = self.db.query_one(
            "SELECT COALESCE(SUM(amount), 0) as total "
            "FROM pending_deductions "
            "WHERE merchant_id = %s AND status = 'pending'",
            merchant_id)["total"]

        available = Decimal(pending) - Decimal(frozen) - Decimal(deductions)

        return {
            "merchant_id": merchant_id,
            "pending_settlement": pending,
            "frozen_amount": frozen,
            "pending_deductions": deductions,
            "available_for_withdrawal": str(max(Decimal("0"), available)),
            "security_deposit": {
                "balance": deposit["balance"] if deposit else "0",
                "required": deposit["required_amount"] if deposit else "0",
                "status": "sufficient" if deposit and Decimal(deposit["balance"]) >= Decimal(deposit["required_amount"]) else "insufficient"
            }
        }

    def get_transaction_history(self, merchant_id, page=1, page_size=20,
                                start_date=None, end_date=None,
                                transaction_type=None):
        """获取交易明细"""
        conditions = ["merchant_id = %s"]
        params = [merchant_id]

        if start_date:
            conditions.append("created_at >= %s")
            params.append(start_date)
        if end_date:
            conditions.append("created_at <= %s")
            params.append(end_date)

        where = " AND ".join(conditions)

        # 结算记录
        settlements = self.db.query(
            f"SELECT 'settlement' as type, order_id as ref_id, "
            f"merchant_income as amount, commission, created_at "
            f"FROM settlement_ledger WHERE {where} "
            f"ORDER BY created_at DESC LIMIT %s OFFSET %s",
            *params, page_size, (page-1)*page_size)

        # 退款冲抵
        refunds = self.db.query(
            f"SELECT 'refund_offset' as type, order_id as ref_id, "
            f"amount, NULL as commission, created_at "
            f"FROM refund_offset_log WHERE {where} "
            f"ORDER BY created_at DESC LIMIT %s OFFSET %s",
            *params, page_size, (page-1)*page_size)

        # 提现记录
        withdrawals = self.db.query(
            f"SELECT 'withdrawal' as type, withdrawal_id as ref_id, "
            f"amount, NULL as commission, created_at "
            f"FROM merchant_withdrawals WHERE {where} "
            f"ORDER BY created_at DESC LIMIT %s OFFSET %s",
            *params, page_size, (page-1)*page_size)

        all_transactions = sorted(
            settlements + refunds + withdrawals,
            key=lambda x: x["created_at"], reverse=True)

        return {
            "merchant_id": merchant_id,
            "transactions": all_transactions[:page_size],
            "page": page
        }

    def download_settlement_report(self, merchant_id, period_start, period_end,
                                   format="csv"):
        """下载结算报告"""
        records = self.db.query(
            "SELECT sl.*, o.order_number "
            "FROM settlement_ledger sl "
            "LEFT JOIN orders o ON sl.order_id = o.id "
            "WHERE sl.merchant_id = %s "
            "AND sl.created_at BETWEEN %s AND %s "
            "ORDER BY sl.created_at",
            merchant_id, period_start, period_end)

        if format == "csv":
            import csv
            import io
            output = io.StringIO()
            writer = csv.writer(output)
            writer.writerow(["订单号", "商品金额", "运费", "佣金",
                           "商家收入", "状态", "创建时间"])

            for r in records:
                writer.writerow([
                    r.get("order_number", r["order_id"]),
                    r["item_amount"], r["shipping_fee"],
                    r["commission"], r["merchant_income"],
                    r["status"], r["created_at"]
                ])

            report_url = self._upload_report(
                f"settlement_{merchant_id}_{period_start}_{period_end}.csv",
                output.getvalue().encode())
            return {"format": "csv", "url": report_url, "record_count": len(records)}

        elif format == "pdf":
            # PDF 生成（简化）
            report_url = self._generate_pdf_report(merchant_id, records,
                period_start, period_end)
            return {"format": "pdf", "url": report_url, "record_count": len(records)}

    def update_bank_account(self, merchant_id, new_bank_info, verification_code):
        """更新银行账户"""
        # 1. 验证验证码
        cached_code = self.redis.get(f"bank_verify:{merchant_id}")
        if not cached_code or cached_code.decode() != verification_code:
            return {"status": "invalid_verification_code"}

        # 2. 检查是否有待打款
        pending_payout = self.db.count("settlement_bills",
            merchant_id=merchant_id, status="generated")
        if pending_payout > 0:
            return {"status": "has_pending_payout",
                    "message": "请等待待打款完成后再更改银行账户"}

        # 3. 更新银行账户
        self.db.insert("merchant_bank_accounts", {
            "account_id": str(uuid4()),
            "merchant_id": merchant_id,
            "bank_name": new_bank_info["bank_name"],
            "account_number": self._encrypt_account(new_bank_info["account_number"]),
            "account_name": new_bank_info["account_name"],
            "branch": new_bank_info.get("branch"),
            "status": "pending_verification",
            "created_at": now()
        })

        # 4. 发起小额打款验证
        verify_amount = round(0.01 + hash(merchant_id) % 99 / 100, 2)
        self.payment_gateway.transfer(
            amount=verify_amount,
            bank_account=new_bank_info["account_number"],
            bank_name=new_bank_info["bank_name"],
            reference_id=f"verify_{merchant_id}")

        return {"status": "pending_verification",
                "message": f"已向新账户打入 {verify_amount} 元，请确认到账金额完成验证"}

    def submit_dispute(self, merchant_id, bill_id, dispute_reason, evidence_urls=None):
        """提交结算争议"""
        bill = self.db.get_settlement_bill(bill_id)

        if bill["merchant_id"] != merchant_id:
            raise PermissionDeniedError("非本人账单")

        if bill["status"] not in ["generated", "paid"]:
            return {"status": "cannot_dispute", "current_status": bill["status"]}

        dispute_id = str(uuid4())
        self.db.insert("settlement_disputes", {
            "dispute_id": dispute_id,
            "merchant_id": merchant_id,
            "bill_id": bill_id,
            "reason": dispute_reason,
            "evidence_urls": json.dumps(evidence_urls or []),
            "status": "submitted",
            "created_at": now()
        })

        # 通知财务团队
        self.notification.send("finance_team",
            f"商家 {merchant_id} 对账单 {bill_id} 提出争议: {dispute_reason}")

        return {"dispute_id": dispute_id, "status": "submitted"}

    def _encrypt_account(self, account_number):
        """加密银行账号"""
        from cryptography.fernet import Fernet
        key = self.config.get("encryption_key")
        f = Fernet(key.encode())
        return f.encrypt(account_number.encode()).decode()

    def _upload_report(self, filename, content):
        """上传报告到对象存储"""
        return self.object_storage.upload(filename, content)

    def _generate_pdf_report(self, merchant_id, records, start, end):
        """生成 PDF 报告"""
        # 使用 PDF 生成库
        filename = f"settlement_{merchant_id}_{start}_{end}.pdf"
        self.object_storage.upload(filename, b"PDF content placeholder")
        return self.object_storage.get_url(filename)
```

## 十、更多异常场景

### 场景 12：商家评级被恶意刷评

```
触发：竞争对手恶意给商家差评 → 投诉率虚高 → 评级下降 → 佣金费率上升
检测：
  1. 短时间大量差评来自同一用户/IP → 恶意刷评
  2. 评分与历史趋势大幅偏离 → 异常
处理：
  1. 过滤异常评价（不纳入评级计算）
  2. 评级计算使用加权平均（近期权重更高）
  3. 人工审核异常评分
预防：异常过滤 + 加权平均 + 人工审核
```

### 场景 13：分账执行部分失败

```
触发：分账 3 方（商家/平台/物流）→ 商家成功/平台成功/物流失败 → 分账不一致
检测：
  1. 分账记录部分 status=failed → 部分失败
  2. 分账总额 ≠ 订单金额 → 不一致
处理：
  1. 分账操作使用分布式事务
  2. 失败自动重试（3 次）
  3. 重试仍失败 → 回滚全部分账
预防：分布式事务 + 自动重试 + 全量回滚
```

### 场景 14：自助提现被恶意操作

```
触发：商家账户被盗 → 恶意申请大额提现到攻击者银行账户 → 资金损失
检测：
  1. 提现到新银行账户 → 高风险
  2. 大额提现（超过历史均值 3 倍）→ 异常
  3. 异地 IP 操作 → 可疑
处理：
  1. 新银行账户需小额验证
  2. 大额提现需人工审核
  3. 异常操作二次验证（短信/邮箱确认）
预防：新账户验证 + 大额审核 + 二次确认
```

### 场景 15：结算报告数据泄露

```
触发：商家下载结算报告 → 报告 URL 被泄露 → 竞争对手获取销售数据 → 商业机密泄露
检测：
  1. 报告下载次数 > 1 → 可能泄露
  2. 非商家 IP 访问报告 URL → 泄露
处理：
  1. 报告 URL 设置短期有效期（1 小时）
  2. 下载需身份验证
  3. 报告加水印（商家 ID + 时间）
预防：短期有效 + 身份验证 + 水印
```

### 3.19 商家结算数据导出与报表

```python
class SettlementExportService:
    """结算数据导出：多格式 + 大文件分片 + 定时导出"""

    EXPORT_FORMATS = {
        "csv": "CSV 文件",
        "excel": "Excel 文件",
        "pdf": "PDF 报表",
        "json": "JSON 数据",
    }

    def create_export_task(self, merchant_id, export_config):
        """创建导出任务"""
        task_id = str(uuid4())

        # 1. 估算数据量
        record_count = self.db.count("settlement_ledger",
            merchant_id=merchant_id,
            created_at__gte=export_config.get("start_date", "2000-01-01"),
            created_at__lte=export_config.get("end_date", "2099-12-31"))

        # 2. 大文件标记
        is_large = record_count > 100000

        self.db.insert("export_tasks", {
            "task_id": task_id,
            "merchant_id": merchant_id,
            "format": export_config.get("format", "csv"),
            "start_date": export_config.get("start_date"),
            "end_date": export_config.get("end_date"),
            "record_count": record_count,
            "is_large": is_large,
            "status": "pending",
            "created_at": now()
        })

        # 3. 异步执行
        if is_large:
            self.task_queue.submit(self._execute_large_export, task_id)
        else:
            self.task_queue.submit(self._execute_export, task_id)

        return {"task_id": task_id, "record_count": record_count,
                "is_large": is_large,
                "estimated_time_seconds": max(10, record_count // 10000)}

    def _execute_export(self, task_id):
        """执行普通导出"""
        task = self.db.get_export_task(task_id)

        self.db.update("export_tasks",
            {"status": "processing", "started_at": now()},
            {"task_id": task_id})

        records = self.db.query(
            "SELECT sl.*, o.order_number, o.customer_id "
            "FROM settlement_ledger sl "
            "LEFT JOIN orders o ON sl.order_id = o.id "
            "WHERE sl.merchant_id = %s "
            "AND sl.created_at BETWEEN %s AND %s "
            "ORDER BY sl.created_at",
            task["merchant_id"], task["start_date"], task["end_date"])

        if task["format"] == "csv":
            content = self._generate_csv(records)
        elif task["format"] == "excel":
            content = self._generate_excel(records)
        elif task["format"] == "pdf":
            content = self._generate_pdf(records, task)
        else:
            content = self._generate_json(records)

        # 上传到对象存储
        filename = f"settlement_export/{task['merchant_id']}/{task_id}.{task['format']}"
        url = self.object_storage.upload(filename, content)

        # 更新任务
        self.db.update("export_tasks",
            {"status": "completed", "completed_at": now(),
             "file_url": url, "file_size": len(content)},
            {"task_id": task_id})

        # 设置 URL 有效期（1 小时）
        self.redis.setex(f"export_url:{task_id}", 3600, url)

        return {"task_id": task_id, "url": url}

    def _execute_large_export(self, task_id):
        """执行大文件导出（分片）"""
        task = self.db.get_export_task(task_id)
        chunk_size = 50000

        self.db.update("export_tasks",
            {"status": "processing", "started_at": now()},
            {"task_id": task_id})

        # 分片查询和写入
        all_content = []
        offset = 0

        while True:
            records = self.db.query(
                "SELECT sl.*, o.order_number FROM settlement_ledger sl "
                "LEFT JOIN orders o ON sl.order_id = o.id "
                "WHERE sl.merchant_id = %s "
                "AND sl.created_at BETWEEN %s AND %s "
                "ORDER BY sl.created_at LIMIT %s OFFSET %s",
                task["merchant_id"], task["start_date"], task["end_date"],
                chunk_size, offset)

            if not records:
                break

            chunk_content = self._generate_csv(records)
            all_content.append(chunk_content)

            offset += chunk_size

            # 更新进度
            progress = min(100, int(offset / task["record_count"] * 100))
            self.redis.set(f"export_progress:{task_id}", progress)

        # 合并并上传
        full_content = b"".join(c if isinstance(c, bytes) else c.encode() for c in all_content)
        filename = f"settlement_export/{task['merchant_id']}/{task_id}.csv"
        url = self.object_storage.upload(filename, full_content)

        self.db.update("export_tasks",
            {"status": "completed", "completed_at": now(),
             "file_url": url, "file_size": len(full_content)},
            {"task_id": task_id})

        return {"task_id": task_id, "url": url}

    def get_export_status(self, task_id):
        """获取导出任务状态"""
        task = self.db.get_export_task(task_id)

        result = {
            "task_id": task_id,
            "status": task["status"],
            "record_count": task["record_count"],
        }

        if task["status"] == "processing":
            progress = self.redis.get(f"export_progress:{task_id}")
            result["progress_pct"] = int(progress) if progress else 0

        if task["status"] == "completed":
            result["file_url"] = task["file_url"]
            result["file_size"] = task["file_size"]

        return result

    def _generate_csv(self, records):
        """生成 CSV"""
        import csv, io
        output = io.StringIO()
        writer = csv.writer(output)
        writer.writerow(["订单号", "商品金额", "运费", "佣金",
                        "商家收入", "退款冲抵", "状态", "创建时间"])
        for r in records:
            writer.writerow([
                r.get("order_number", r["order_id"]),
                r["item_amount"], r["shipping_fee"],
                r["commission"], r["merchant_income"],
                r.get("refund_offset", "0.00"),
                r["status"], str(r["created_at"])
            ])
        return output.getvalue().encode("utf-8-sig")

    def _generate_excel(self, records):
        """生成 Excel"""
        # 简化实现
        return self._generate_csv(records)  # 实际使用 openpyxl

    def _generate_pdf(self, records, task):
        """生成 PDF 报表"""
        # 简化实现
        return b"PDF content placeholder"

    def _generate_json(self, records):
        """生成 JSON"""
        return json.dumps(records, default=str, ensure_ascii=False).encode("utf-8")
```

### 3.20 商家合同与费率协议管理

```python
class MerchantContractService:
    """商家合同管理：合同签署 → 费率协议 → 到期提醒 → 续约"""

    CONTRACT_STATUSES = {
        "draft": "草稿",
        "pending_signature": "待签署",
        "active": "生效中",
        "expiring_soon": "即将到期",
        "expired": "已过期",
        "terminated": "已终止",
    }

    def create_contract(self, merchant_id, contract_data):
        """创建合同"""
        contract_id = str(uuid4())

        # 1. 验证费率合理性
        commission_rate = Decimal(str(contract_data.get("commission_rate", 5.0)))
        if commission_rate < Decimal("1.0") or commission_rate > Decimal("20.0"):
            raise InvalidCommissionRateError("佣金费率需在 1%-20% 之间")

        # 2. 检查是否有冲突的活跃合同
        active = self.db.query_one(
            "SELECT * FROM merchant_contracts "
            "WHERE merchant_id = %s AND status = 'active'",
            merchant_id)

        if active:
            # 新合同生效时自动终止旧合同
            contract_data["supersedes_contract_id"] = active["contract_id"]

        # 3. 创建合同
        self.db.insert("merchant_contracts", {
            "contract_id": contract_id,
            "merchant_id": merchant_id,
            "commission_rate": str(commission_rate),
            "settlement_cycle": contract_data.get("settlement_cycle", "T1"),
            "min_settlement_amount": str(contract_data.get("min_settlement_amount", 100)),
            "special_terms": json.dumps(contract_data.get("special_terms", [])),
            "effective_date": contract_data.get("effective_date", now()),
            "expiry_date": contract_data.get("expiry_date",
                now() + timedelta(days=365)),
            "status": "pending_signature",
            "created_at": now()
        })

        # 4. 通知商家签署
        self.notification.send(merchant_id,
            "新合同已创建，请登录商家后台签署")

        return {"contract_id": contract_id, "status": "pending_signature"}

    def sign_contract(self, contract_id, merchant_id, signature_data):
        """签署合同"""
        contract = self.db.get_contract(contract_id)

        if contract["merchant_id"] != merchant_id:
            raise PermissionDeniedError("非本人合同")

        if contract["status"] != "pending_signature":
            return {"status": "cannot_sign", "current_status": contract["status"]}

        # 1. 验证签名
        if not self._verify_signature(signature_data):
            return {"status": "signature_verification_failed"}

        # 2. 激活合同
        self.db.update("merchant_contracts",
            {"status": "active", "signed_at": now(),
             "signature_data": json.dumps(signature_data)},
            {"contract_id": contract_id})

        # 3. 如果取代旧合同
        if contract.get("supersedes_contract_id"):
            self.db.update("merchant_contracts",
                {"status": "superseded", "superseded_at": now()},
                {"contract_id": contract["supersedes_contract_id"]})

        # 4. 应用费率
        self.db.update("merchants",
            {"commission_rate": contract["commission_rate"],
             "settlement_cycle": contract["settlement_cycle"],
             "min_settlement_amount": contract["min_settlement_amount"]},
            {"id": merchant_id})

        return {"contract_id": contract_id, "status": "active"}

    def check_expiring_contracts(self):
        """检查即将到期的合同"""
        # 30 天内到期
        expiring_soon = self.db.query(
            "SELECT * FROM merchant_contracts "
            "WHERE status = 'active' "
            "AND expiry_date BETWEEN NOW() AND NOW() + INTERVAL 30 DAY")

        for contract in expiring_soon:
            days_left = (contract["expiry_date"] - now()).days

            self.db.update("merchant_contracts",
                {"status": "expiring_soon"},
                {"contract_id": contract["contract_id"]})

            self.notification.send(contract["merchant_id"],
                f"合同将于 {days_left} 天后到期，请及时续约")

            if days_left <= 7:
                self.alert(f"商家 {contract['merchant_id']} 合同 {days_left} 天后到期")

        # 已过期
        expired = self.db.query(
            "SELECT * FROM merchant_contracts "
            "WHERE status IN ('active', 'expiring_soon') "
            "AND expiry_date < NOW()")

        for contract in expired:
            self.db.update("merchant_contracts",
                {"status": "expired"},
                {"contract_id": contract["contract_id"]})

            # 合同过期 → 应用默认费率
            self.db.update("merchants",
                {"commission_rate": "5.0", "settlement_cycle": "T7"},
                {"id": contract["merchant_id"]})

            self.notification.send(contract["merchant_id"],
                "合同已过期，已应用默认费率，请尽快续约")

        return {"expiring_soon": len(expiring_soon), "expired": len(expired)}

    def terminate_contract(self, contract_id, reason, terminated_by):
        """终止合同"""
        contract = self.db.get_contract(contract_id)

        if contract["status"] not in ["active", "expiring_soon"]:
            return {"status": "cannot_terminate"}

        # 1. 终止合同
        self.db.update("merchant_contracts",
            {"status": "terminated", "terminated_at": now(),
             "termination_reason": reason,
             "terminated_by": terminated_by},
            {"contract_id": contract_id})

        # 2. 冻结结算
        self.db.update("settlement_ledger",
            {"status": "frozen"},
            {"merchant_id": contract["merchant_id"], "status": "pending"})

        # 3. 通知商家
        self.notification.send(contract["merchant_id"],
            f"合同已终止: {reason}，结算已冻结")

        return {"contract_id": contract_id, "status": "terminated"}

    def _verify_signature(self, signature_data):
        """验证电子签名"""
        # 简化实现：检查签名数据完整性
        return bool(signature_data.get("signature_image") or
                   signature_data.get("digital_signature"))
```

### 3.21 结算系统灾备与恢复

```python
class SettlementDisasterRecoveryService:
    """结算灾备：数据备份 → 故障切换 → 数据恢复 → 一致性验证"""

    BACKUP_SCOPE = [
        "settlement_ledger",
        "settlement_bills",
        "pending_deductions",
        "merchant_security_deposits",
        "deposit_transactions",
        "merchant_withdrawals",
        "refund_offset_log",
    ]

    def create_backup(self):
        """创建结算数据备份"""
        backup_id = str(uuid4())
        backup_tables = {}

        for table in self.BACKUP_SCOPE:
            # 获取增量变更（基于 updated_at）
            last_backup_time = self.redis.get(f"last_backup:{table}")
            if last_backup_time:
                records = self.db.query(
                    f"SELECT * FROM {table} "
                    f"WHERE updated_at > %s OR created_at > %s "
                    f"ORDER BY id",
                    last_backup_time.decode(), last_backup_time.decode())
            else:
                records = self.db.query(f"SELECT * FROM {table} ORDER BY id")

            # 计算校验和
            content = json.dumps(records, default=str)
            checksum = hashlib.sha256(content.encode()).hexdigest()

            # 上传到备份存储
            backup_key = f"settlement_backup/{backup_id}/{table}.json"
            self.backup_storage.upload(backup_key, content.encode())

            backup_tables[table] = {
                "record_count": len(records),
                "checksum": checksum,
                "backup_key": backup_key
            }

            # 更新最后备份时间
            self.redis.set(f"last_backup:{table}", now().isoformat())

        # 保存备份元数据
        self.db.insert("settlement_backups", {
            "backup_id": backup_id,
            "tables": json.dumps(backup_tables),
            "status": "completed",
            "created_at": now()
        })

        return {"backup_id": backup_id, "tables": backup_tables}

    def verify_backup_integrity(self, backup_id):
        """验证备份完整性"""
        backup = self.db.get_backup(backup_id)
        tables = json.loads(backup["tables"])

        verification = {}
        for table, meta in tables.items():
            # 下载备份文件
            content = self.backup_storage.download(meta["backup_key"])

            # 计算校验和
            actual_checksum = hashlib.sha256(content).hexdigest()
            expected_checksum = meta["checksum"]

            verification[table] = {
                "checksum_match": actual_checksum == expected_checksum,
                "record_count": meta["record_count"]
            }

        all_ok = all(v["checksum_match"] for v in verification.values())

        return {"backup_id": backup_id, "all_tables_ok": all_ok,
                "details": verification}

    def recover_from_backup(self, backup_id, target_tables=None):
        """从备份恢复"""
        backup = self.db.get_backup(backup_id)
        tables = json.loads(backup["tables"])

        if target_tables:
            tables = {k: v for k, v in tables.items() if k in target_tables}

        recovered = {}

        for table, meta in tables.items():
            # 1. 下载备份
            content = self.backup_storage.download(meta["backup_key"])
            records = json.loads(content.decode())

            # 2. 清空目标表
            self.db.execute(f"TRUNCATE TABLE {table}")

            # 3. 恢复数据
            for record in records:
                self.db.insert(table, record)

            recovered[table] = {"records_restored": len(records)}

        # 4. 验证恢复后数据一致性
        consistency = self._verify_data_consistency()

        return {"backup_id": backup_id, "recovered_tables": recovered,
                "consistency_check": consistency}

    def _verify_data_consistency(self):
        """验证数据一致性"""
        issues = []

        # 检查 1：待结算总额 = 各商家待结算汇总
        ledger_total = self.db.query_one(
            "SELECT COALESCE(SUM(merchant_income), 0) as total "
            "FROM settlement_ledger WHERE status = 'pending'")["total"]

        # 检查 2：账单金额 = 关联记录金额之和
        bills = self.db.query(
            "SELECT bill_id, settlement_amount FROM settlement_bills "
            "WHERE status IN ('generated', 'paid')")

        for bill in bills:
            records_total = self.db.query_one(
                "SELECT COALESCE(SUM(merchant_income), 0) as total "
                "FROM settlement_ledger WHERE bill_id = %s",
                bill["bill_id"])["total"]

            if abs(Decimal(records_total) - Decimal(bill["settlement_amount"])) > Decimal("0.01"):
                issues.append(f"账单 {bill['bill_id']} 金额不一致")

        return {"issues": issues, "is_consistent": len(issues) == 0}
```

## 十一、性能优化

### 高并发写入优化

```python
class SettlementBatchWriter:
    """批量写入：攒批 + 异步写入 + 写入确认"""

    BATCH_SIZE = 500
    FLUSH_INTERVAL_SECONDS = 5

    def __init__(self):
        self.buffer = []
        self.last_flush = now()

    def add_record(self, record):
        """添加记录到缓冲区"""
        self.buffer.append(record)

        if len(self.buffer) >= self.BATCH_SIZE:
            self._flush()

    def _flush(self):
        """刷新缓冲区到数据库"""
        if not self.buffer:
            return

        # 批量插入
        values = []
        placeholders = []
        for record in self.buffer:
            placeholders.append("(%s, %s, %s, %s, %s, %s, %s, %s, %s)")
            values.extend([
                record["ledger_id"], record["order_id"],
                record["merchant_id"], record["item_amount"],
                record["commission"], record["merchant_income"],
                record["status"], json.dumps(record.get("coupon_details", [])),
                record["created_at"]
            ])

        sql = f"INSERT INTO settlement_ledger VALUES {','.join(placeholders)}"
        self.db.execute(sql, *values)

        # 批量更新 Redis
        pipe = self.redis.pipeline()
        for record in self.buffer:
            pipe.hincrbyfloat(f"merchant_pending:{record['merchant_id']}",
                "total_income", float(record["merchant_income"]))
        pipe.execute()

        self.buffer = []
        self.last_flush = now()
```

### 查询优化

```python
class SettlementQueryOptimizer:
    """查询优化：物化视图 + 读写分离 + 缓存"""

    def get_merchant_summary_cached(self, merchant_id):
        """带缓存的商家汇总查询"""
        cache_key = f"merchant_summary:{merchant_id}"
        cached = self.redis.get(cache_key)
        if cached:
            return json.loads(cached)

        # 从只读副本查询
        result = self.read_replica.query_one(
            "SELECT "
            "  COALESCE(SUM(CASE WHEN status='pending' THEN merchant_income ELSE 0 END), 0) as pending, "
            "  COALESCE(SUM(CASE WHEN status='settled' THEN merchant_income ELSE 0 END), 0) as settled, "
            "  COUNT(*) as total_orders "
            "FROM settlement_ledger "
            "WHERE merchant_id = %s", merchant_id)

        self.redis.setex(cache_key, 300, json.dumps(result, default=str))
        return result
```

## 十二、API 接口设计

### 核心接口

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /api/v1/settlement/calculate | 计算订单结算 |
| GET | /api/v1/merchants/{id}/balance | 余额概览 |
| GET | /api/v1/merchants/{id}/transactions | 交易明细 |
| POST | /api/v1/settlement/bills/generate | 生成账单 |
| POST | /api/v1/settlement/bills/{id}/payout | 执行打款 |
| GET | /api/v1/settlement/bills/{id} | 账单详情 |
| POST | /api/v1/merchants/{id}/withdrawals | 申请提现 |
| POST | /api/v1/refunds/{id}/offset | 退款冲抵 |
| GET | /api/v1/merchants/{id}/deposit | 保证金信息 |
| POST | /api/v1/merchants/{id}/disputes | 提交争议 |
| POST | /api/v1/exports | 创建导出任务 |
| GET | /api/v1/exports/{id} | 导出状态 |
| GET | /api/v1/reconciliation/daily | 日对账 |
| POST | /api/v1/contracts | 创建合同 |
| GET | /api/v1/merchants/{id}/rating | 商家评级 |

## 商家促销活动结算完整实现

```python
from datetime import datetime, timedelta
from decimal import Decimal, ROUND_HALF_UP
from typing import Dict, List, Optional, Tuple
from enum import Enum
import logging

logger = logging.getLogger(__name__)


class PromotionType(Enum):
    """促销活动类型"""
    FULL_REDUCTION = "full_reduction"           # 满减
    DISCOUNT = "discount"                       # 折扣
    COUPON = "coupon"                           # 优惠券
    FLASH_SALE = "flash_sale"                   # 限时秒杀
    BUY_ONE_GET_ONE = "buy_one_get_one"         # 买一赠一
    NEW_USER_DISCOUNT = "new_user_discount"     # 新用户专享
    GROUP_BUY = "group_buy"                     # 拼团


class SubsidyBearer(Enum):
    """补贴承担方"""
    PLATFORM = "platform"       # 平台全额承担
    MERCHANT = "merchant"       # 商家全额承担
    SHARED = "shared"           # 平台与商家分摊


class PromotionStatus(Enum):
    """促销活动状态"""
    DRAFT = "draft"
    ACTIVE = "active"
    PAUSED = "paused"
    ENDED = "ended"
    SETTLED = "settled"
    OVER_BUDGET = "over_budget"


class PromotionOrder:
    """促销订单数据模型"""
    def __init__(
        self,
        order_id: str,
        promotion_id: str,
        merchant_id: str,
        original_amount: Decimal,
        paid_amount: Decimal,
        promotion_type: PromotionType,
        subsidy_bearer: SubsidyBearer,
        platform_share_ratio: Decimal,
        merchant_share_ratio: Decimal,
        order_time: datetime,
        is_refunded: bool = False,
        refund_amount: Decimal = Decimal("0"),
    ):
        self.order_id = order_id
        self.promotion_id = promotion_id
        self.merchant_id = merchant_id
        self.original_amount = original_amount
        self.paid_amount = paid_amount
        self.promotion_type = promotion_type
        self.subsidy_bearer = subsidy_bearer
        self.platform_share_ratio = platform_share_ratio
        self.merchant_share_ratio = merchant_share_ratio
        self.order_time = order_time
        self.is_refunded = is_refunded
        self.refund_amount = refund_amount
        self.subsidy_amount = original_amount - paid_amount

    def calculate_platform_subsidy(self) -> Decimal:
        """计算平台补贴金额"""
        if self.subsidy_bearer == SubsidyBearer.PLATFORM:
            return self.subsidy_amount
        elif self.subsidy_bearer == SubsidyBearer.MERCHANT:
            return Decimal("0")
        elif self.subsidy_bearer == SubsidyBearer.SHARED:
            return (self.subsidy_amount * self.platform_share_ratio).quantize(
                Decimal("0.01"), rounding=ROUND_HALF_UP
            )
        return Decimal("0")

    def calculate_merchant_subsidy(self) -> Decimal:
        """计算商家补贴金额"""
        if self.subsidy_bearer == SubsidyBearer.MERCHANT:
            return self.subsidy_amount
        elif self.subsidy_bearer == SubsidyBearer.PLATFORM:
            return Decimal("0")
        elif self.subsidy_bearer == SubsidyBearer.SHARED:
            return (self.subsidy_amount * self.merchant_share_ratio).quantize(
                Decimal("0.01"), rounding=ROUND_HALF_UP
            )
        return Decimal("0")


class PromotionActivity:
    """促销活动配置"""
    def __init__(
        self,
        promotion_id: str,
        promotion_type: PromotionType,
        name: str,
        start_time: datetime,
        end_time: datetime,
        total_budget: Decimal,
        platform_budget: Decimal,
        merchant_budget: Decimal,
        subsidy_bearer: SubsidyBearer,
        platform_share_ratio: Decimal,
        merchant_share_ratio: Decimal,
        status: PromotionStatus = PromotionStatus.DRAFT,
    ):
        self.promotion_id = promotion_id
        self.promotion_type = promotion_type
        self.name = name
        self.start_time = start_time
        self.end_time = end_time
        self.total_budget = total_budget
        self.platform_budget = platform_budget
        self.merchant_budget = merchant_budget
        self.subsidy_bearer = subsidy_bearer
        self.platform_share_ratio = platform_share_ratio
        self.merchant_share_ratio = merchant_share_ratio
        self.status = status
        self.used_budget = Decimal("0")
        self.used_platform_budget = Decimal("0")
        self.used_merchant_budget = Decimal("0")
        self.order_count = 0
        self.total_gmv = Decimal("0")


class PromotionSettlementService:
    """商家促销活动结算服务

    负责促销活动中平台补贴与商家补贴的精确计算、活动结束后的对账结算、
    超预算处理以及结算报表生成。支持满减、折扣、优惠券、秒杀等多种促销类型，
    以及平台承担、商家承担、平台商家分摊三种补贴模式。
    """

    def __init__(self, db_session, budget_alert_threshold: Decimal = Decimal("0.8")):
        self.db = db_session
        self.budget_alert_threshold = budget_alert_threshold
        self._promotion_cache: Dict[str, PromotionActivity] = {}
        self._order_buffer: List[PromotionOrder] = []

    def calculate_promotion_subsidy(
        self,
        order: PromotionOrder,
    ) -> Dict[str, Decimal]:
        """计算促销订单的平台补贴和商家补贴

        根据促销活动的补贴承担方式，精确拆分每笔促销订单的补贴金额为
        平台承担部分和商家承担部分。对于已退款订单，需扣减对应补贴。

        Args:
            order: 促销订单对象

        Returns:
            包含 platform_subsidy, merchant_subsidy, net_subsidy 的字典

        Raises:
            ValueError: 订单金额或分摊比例无效
        """
        if order.original_amount <= Decimal("0"):
            raise ValueError(f"订单原价必须大于0, 当前: {order.original_amount}")
        if order.paid_amount < Decimal("0"):
            raise ValueError(f"订单实付金额不能为负, 当前: {order.paid_amount}")
        if order.paid_amount > order.original_amount:
            raise ValueError(
                f"实付金额不能超过原价: paid={order.paid_amount}, original={order.original_amount}"
            )

        # 校验分摊比例
        if order.subsidy_bearer == SubsidyBearer.SHARED:
            total_ratio = order.platform_share_ratio + order.merchant_share_ratio
            if total_ratio != Decimal("1"):
                raise ValueError(
                    f"分摊比例之和必须为1, 当前: platform={order.platform_share_ratio}, "
                    f"merchant={order.merchant_share_ratio}, total={total_ratio}"
                )

        platform_subsidy = order.calculate_platform_subsidy()
        merchant_subsidy = order.calculate_merchant_subsidy()

        # 处理退款扣减
        if order.is_refunded and order.refund_amount > Decimal("0"):
            refund_ratio = (order.refund_amount / order.paid_amount).quantize(
                Decimal("0.0001"), rounding=ROUND_HALF_UP
            )
            platform_subsidy = (platform_subsidy * (Decimal("1") - refund_ratio)).quantize(
                Decimal("0.01"), rounding=ROUND_HALF_UP
            )
            merchant_subsidy = (merchant_subsidy * (Decimal("1") - refund_ratio)).quantize(
                Decimal("0.01"), rounding=ROUND_HALF_UP
            )

        net_subsidy = platform_subsidy + merchant_subsidy

        result = {
            "order_id": order.order_id,
            "promotion_id": order.promotion_id,
            "merchant_id": order.merchant_id,
            "original_amount": order.original_amount,
            "paid_amount": order.paid_amount,
            "total_subsidy": order.subsidy_amount,
            "platform_subsidy": platform_subsidy,
            "merchant_subsidy": merchant_subsidy,
            "net_subsidy": net_subsidy,
            "subsidy_bearer": order.subsidy_bearer.value,
            "is_refunded": order.is_refunded,
        }

        # 更新活动预算使用量
        self._update_budget_usage(order.promotion_id, platform_subsidy, merchant_subsidy)

        logger.info(
            f"促销补贴计算完成: order={order.order_id}, "
            f"platform={platform_subsidy}, merchant={merchant_subsidy}"
        )
        return result

    def _update_budget_usage(
        self,
        promotion_id: str,
        platform_subsidy: Decimal,
        merchant_subsidy: Decimal,
    ) -> None:
        """更新活动预算使用量并检查预算阈值"""
        activity = self._get_promotion_activity(promotion_id)
        if not activity:
            return

        activity.used_platform_budget += platform_subsidy
        activity.used_merchant_budget += merchant_subsidy
        activity.used_budget = activity.used_platform_budget + activity.used_merchant_budget
        activity.order_count += 1

        # 检查预算使用率
        platform_usage_rate = activity.used_platform_budget / activity.platform_budget
        if platform_usage_rate >= self.budget_alert_threshold:
            logger.warning(
                f"促销活动 {promotion_id} 平台预算使用率已达 {platform_usage_rate:.2%}, "
                f"阈值 {self.budget_alert_threshold:.2%}"
            )

        if activity.used_platform_budget >= activity.platform_budget:
            self.handle_promotion_over_budget(
                promotion_id, activity.used_platform_budget, activity.platform_budget
            )

    def settle_promotion_reconciliation(
        self,
        promotion_id: str,
        settlement_period_start: datetime,
        settlement_period_end: datetime,
    ) -> Dict:
        """促销活动对账结算

        活动结束后，汇总所有促销订单的补贴金额，与平台和商家的预算进行对账，
        生成结算单。处理超预算部分、退款冲抵、尾差调整等。

        Args:
            promotion_id: 促销活动ID
            settlement_period_start: 结算周期开始时间
            settlement_period_end: 结算周期结束时间

        Returns:
            对账结算结果字典
        """
        activity = self._get_promotion_activity(promotion_id)
        if not activity:
            raise ValueError(f"促销活动不存在: {promotion_id}")

        # 查询该活动所有订单
        orders = self._query_promotion_orders(
            promotion_id, settlement_period_start, settlement_period_end
        )

        # 按商家分组汇总
        merchant_summary: Dict[str, Dict] = {}
        total_platform_subsidy = Decimal("0")
        total_merchant_subsidy = Decimal("0")
        total_refund_offset = Decimal("0")
        total_order_count = 0
        total_gmv = Decimal("0")

        for order in orders:
            subsidy_result = self.calculate_promotion_subsidy(order)
            mid = order.merchant_id

            if mid not in merchant_summary:
                merchant_summary[mid] = {
                    "merchant_id": mid,
                    "order_count": 0,
                    "gmv": Decimal("0"),
                    "platform_subsidy": Decimal("0"),
                    "merchant_subsidy": Decimal("0"),
                    "refund_offset": Decimal("0"),
                }

            merchant_summary[mid]["order_count"] += 1
            merchant_summary[mid]["gmv"] += order.original_amount
            merchant_summary[mid]["platform_subsidy"] += subsidy_result["platform_subsidy"]
            merchant_summary[mid]["merchant_subsidy"] += subsidy_result["merchant_subsidy"]

            total_platform_subsidy += subsidy_result["platform_subsidy"]
            total_merchant_subsidy += subsidy_result["merchant_subsidy"]
            total_order_count += 1
            total_gmv += order.original_amount

            if order.is_refunded:
                refund_offset = subsidy_result["platform_subsidy"] + subsidy_result["merchant_subsidy"]
                merchant_summary[mid]["refund_offset"] += refund_offset
                total_refund_offset += refund_offset

        # 计算预算差异
        platform_budget_diff = activity.platform_budget - total_platform_subsidy
        merchant_budget_diff = activity.merchant_budget - total_merchant_subsidy

        # 尾差调整：确保分摊金额之和与总补贴精确一致
        rounding_adjustment = Decimal("0")
        for mid, summary in merchant_summary.items():
            expected_total = summary["platform_subsidy"] + summary["merchant_subsidy"]
            actual_subsidy = sum(
                o.subsidy_amount for o in orders if o.merchant_id == mid and not o.is_refunded
            )
            diff = actual_subsidy - expected_total
            if abs(diff) <= Decimal("0.01"):
                rounding_adjustment += diff
                summary["platform_subsidy"] += diff

        # 更新活动状态
        activity.status = PromotionStatus.SETTLED

        reconciliation_result = {
            "promotion_id": promotion_id,
            "promotion_name": activity.name,
            "promotion_type": activity.promotion_type.value,
            "settlement_period": {
                "start": settlement_period_start.isoformat(),
                "end": settlement_period_end.isoformat(),
            },
            "total_orders": total_order_count,
            "total_gmv": str(total_gmv),
            "total_platform_subsidy": str(total_platform_subsidy),
            "total_merchant_subsidy": str(total_merchant_subsidy),
            "total_refund_offset": str(total_refund_offset),
            "platform_budget": str(activity.platform_budget),
            "merchant_budget": str(activity.merchant_budget),
            "platform_budget_diff": str(platform_budget_diff),
            "merchant_budget_diff": str(merchant_budget_diff),
            "rounding_adjustment": str(rounding_adjustment),
            "merchant_details": {
                mid: {k: str(v) if isinstance(v, Decimal) else v for k, v in summary.items()}
                for mid, summary in merchant_summary.items()
            },
            "settled_at": datetime.now().isoformat(),
        }

        # 持久化结算结果
        self._save_reconciliation_result(reconciliation_result)

        logger.info(
            f"促销对账结算完成: promotion={promotion_id}, "
            f"platform_subsidy={total_platform_subsidy}, merchant_subsidy={total_merchant_subsidy}, "
            f"orders={total_order_count}"
        )
        return reconciliation_result

    def get_promotion_settlement_report(
        self,
        promotion_id: str,
        include_details: bool = False,
    ) -> Dict:
        """生成促销活动结算报表

        汇总促销活动的整体结算数据，包括补贴使用情况、预算执行率、
        各商家补贴明细、退款影响等。

        Args:
            promotion_id: 促销活动ID
            include_details: 是否包含订单级明细

        Returns:
            结算报表字典
        """
        activity = self._get_promotion_activity(promotion_id)
        if not activity:
            raise ValueError(f"促销活动不存在: {promotion_id}")

        # 计算预算执行率
        platform_execution_rate = (
            (activity.used_platform_budget / activity.platform_budget * Decimal("100"))
            .quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
            if activity.platform_budget > Decimal("0") else Decimal("0")
        )
        merchant_execution_rate = (
            (activity.used_merchant_budget / activity.merchant_budget * Decimal("100"))
            .quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
            if activity.merchant_budget > Decimal("0") else Decimal("0")
        )
        total_execution_rate = (
            (activity.used_budget / activity.total_budget * Decimal("100"))
            .quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
            if activity.total_budget > Decimal("0") else Decimal("0")
        )

        # 按促销类型统计
        type_statistics = self._get_type_statistics(promotion_id)

        # 时段分布统计
        time_distribution = self._get_time_distribution(promotion_id)

        report = {
            "promotion_id": promotion_id,
            "promotion_name": activity.name,
            "promotion_type": activity.promotion_type.value,
            "status": activity.status.value,
            "period": {
                "start": activity.start_time.isoformat(),
                "end": activity.end_time.isoformat(),
                "duration_hours": (
                    (activity.end_time - activity.start_time).total_seconds() / 3600
                ),
            },
            "budget_summary": {
                "total_budget": str(activity.total_budget),
                "platform_budget": str(activity.platform_budget),
                "merchant_budget": str(activity.merchant_budget),
                "used_total": str(activity.used_budget),
                "used_platform": str(activity.used_platform_budget),
                "used_merchant": str(activity.used_merchant_budget),
                "remaining_total": str(activity.total_budget - activity.used_budget),
                "remaining_platform": str(activity.platform_budget - activity.used_platform_budget),
                "remaining_merchant": str(activity.merchant_budget - activity.used_merchant_budget),
                "total_execution_rate": f"{total_execution_rate}%",
                "platform_execution_rate": f"{platform_execution_rate}%",
                "merchant_execution_rate": f"{merchant_execution_rate}%",
            },
            "order_statistics": {
                "total_orders": activity.order_count,
                "total_gmv": str(activity.total_gmv),
                "avg_subsidy_per_order": str(
                    (activity.used_budget / activity.order_count).quantize(
                        Decimal("0.01"), rounding=ROUND_HALF_UP
                    ) if activity.order_count > 0 else Decimal("0")
                ),
                "subsidy_rate": str(
                    (activity.used_budget / activity.total_gmv * Decimal("100")).quantize(
                        Decimal("0.01"), rounding=ROUND_HALF_UP
                    ) if activity.total_gmv > Decimal("0") else Decimal("0")
                ) + "%",
            },
            "type_statistics": type_statistics,
            "time_distribution": time_distribution,
            "generated_at": datetime.now().isoformat(),
        }

        if include_details:
            report["order_details"] = self._get_order_details(promotion_id)

        return report

    def handle_promotion_over_budget(
        self,
        promotion_id: str,
        used_amount: Decimal,
        budget_limit: Decimal,
    ) -> Dict:
        """处理促销活动超预算情况

        当促销补贴金额超过预算时，执行超预算处理策略：
        1. 暂停活动，停止接单
        2. 通知运营和商家
        3. 根据配置决定是否自动追加预算或等待人工审批
        4. 对超预算部分进行标记，后续结算时特殊处理

        Args            promotion_id: 促销活动ID
            used_amount: 已使用金额
            budget_limit: 预算上限

        Returns:
            超预算处理结果字典
        """
        activity = self._get_promotion_activity(promotion_id)
        if not activity:
            raise ValueError(f"促销活动不存在: {promotion_id}")

        over_amount = used_amount - budget_limit
        over_rate = (over_amount / budget_limit * Decimal("100")).quantize(
            Decimal("0.01"), rounding=ROUND_HALF_UP
        )

        # 更新活动状态
        activity.status = PromotionStatus.OVER_BUDGET

        # 超预算等级判定
        if over_rate <= Decimal("5"):
            severity = "minor"       # 轻微超支(5%以内)
            auto_action = "auto_supplement"
        elif over_rate <= Decimal("20"):
            severity = "moderate"    # 中度超支(5%-20%)
            auto_action = "pause_and_notify"
        else:
            severity = "critical"    # 严重超支(20%以上)
            auto_action = "freeze_and_escalate"

        # 执行自动处理动作
        action_result = self._execute_budget_action(
            promotion_id, auto_action, over_amount, severity
        )

        # 生成超预算记录
        over_budget_record = {
            "promotion_id": promotion_id,
            "promotion_name": activity.name,
            "budget_limit": str(budget_limit),
            "used_amount": str(used_amount),
            "over_amount": str(over_amount),
            "over_rate": f"{over_rate}%",
            "severity": severity,
            "auto_action": auto_action,
            "action_result": action_result,
            "status": activity.status.value,
            "detected_at": datetime.now().isoformat(),
        }

        # 持久化超预算记录
        self._save_over_budget_record(over_budget_record)

        # 发送通知
        self._notify_budget_overrun(
            promotion_id=promotion_id,
            over_amount=over_amount,
            over_rate=over_rate,
            severity=severity,
        )

        logger.warning(
            f"促销活动超预算: promotion={promotion_id}, "
            f"budget={budget_limit}, used={used_amount}, "
            f"over={over_amount}({over_rate}%), severity={severity}"
        )

        return over_budget_record

    def _execute_budget_action(
        self,
        promotion_id: str,
        action: str,
        over_amount: Decimal,
        severity: str,
    ) -> Dict:
        """执行超预算自动处理动作"""
        if action == "auto_supplement":
            # 自动追加预算（小额度）
            supplement_amount = over_amount * Decimal("1.1")
            self._supplement_promotion_budget(promotion_id, supplement_amount)
            return {
                "action": "auto_supplement",
                "supplement_amount": str(supplement_amount),
                "status": "completed",
            }
        elif action == "pause_and_notify":
            # 暂停活动并通知
            self._pause_promotion(promotion_id)
            self._send_budget_notification(promotion_id, "paused", severity)
            return {
                "action": "pause_and_notify",
                "status": "completed",
            }
        elif action == "freeze_and_escalate":
            # 冻结活动并升级处理
            self._freeze_promotion(promotion_id)
            self._send_budget_notification(promotion_id, "frozen", severity)
            self._escalate_to_management(promotion_id, over_amount, severity)
            return {
                "action": "freeze_and_escalate",
                "status": "completed",
            }
        return {"action": action, "status": "unknown"}

    def _get_promotion_activity(self, promotion_id: str) -> Optional[PromotionActivity]:
        """获取促销活动配置（含缓存）"""
        if promotion_id in self._promotion_cache:
            return self._promotion_cache[promotion_id]
        activity = self.db.query(PromotionActivity).filter_by(promotion_id=promotion_id).first()
        if activity:
            self._promotion_cache[promotion_id] = activity
        return activity

    def _query_promotion_orders(
        self, promotion_id: str, start: datetime, end: datetime
    ) -> List[PromotionOrder]:
        """查询促销活动下的所有订单"""
        return self.db.query(PromotionOrder).filter(
            PromotionOrder.promotion_id == promotion_id,
            PromotionOrder.order_time >= start,
            PromotionOrder.order_time <= end,
        ).all()

    def _get_type_statistics(self, promotion_id: str) -> Dict:
        """按促销类型统计补贴数据"""
        orders = self.db.query(PromotionOrder).filter_by(promotion_id=promotion_id).all()
        stats = {}
        for order in orders:
            ptype = order.promotion_type.value
            if ptype not in stats:
                stats[ptype] = {"count": 0, "subsidy": Decimal("0"), "gmv": Decimal("0")}
            stats[ptype]["count"] += 1
            stats[ptype]["subsidy"] += order.subsidy_amount
            stats[ptype]["gmv"] += order.original_amount
        return {k: {kk: str(vv) if isinstance(vv, Decimal) else vv for kk, vv in v.items()} for k, v in stats.items()}

    def _get_time_distribution(self, promotion_id: str) -> Dict:
        """按时段统计订单分布"""
        orders = self.db.query(PromotionOrder).filter_by(promotion_id=promotion_id).all()
        distribution = {}
        for order in orders:
            hour_bucket = order.order_time.replace(minute=0, second=0, microsecond=0)
            key = hour_bucket.isoformat()
            if key not in distribution:
                distribution[key] = {"count": 0, "subsidy": Decimal("0")}
            distribution[key]["count"] += 1
            distribution[key]["subsidy"] += order.subsidy_amount
        return {k: {kk: str(vv) if isinstance(vv, Decimal) else vv for kk, vv in v.items()} for k, v in distribution.items()}

    def _get_order_details(self, promotion_id: str) -> List[Dict]:
        """获取订单级明细"""
        orders = self.db.query(PromotionOrder).filter_by(promotion_id=promotion_id).all()
        return [
            {
                "order_id": o.order_id,
                "merchant_id": o.merchant_id,
                "original_amount": str(o.original_amount),
                "paid_amount": str(o.paid_amount),
                "subsidy": str(o.subsidy_amount),
                "platform_subsidy": str(o.calculate_platform_subsidy()),
                "merchant_subsidy": str(o.calculate_merchant_subsidy()),
                "is_refunded": o.is_refunded,
                "order_time": o.order_time.isoformat(),
            }
            for o in orders
        ]

    def _save_reconciliation_result(self, result: Dict) -> None:
        """持久化对账结算结果"""
        self.db.insert("promotion_reconciliation", result)
        self.db.commit()

    def _save_over_budget_record(self, record: Dict) -> None:
        """持久化超预算记录"""
        self.db.insert("promotion_over_budget", record)
        self.db.commit()

    def _notify_budget_overrun(self, **kwargs) -> None:
        """发送超预算通知"""
        logger.warning(f"预算超支通知: {kwargs}")

    def _supplement_promotion_budget(self, promotion_id: str, amount: Decimal) -> None:
        """追加促销预算"""
        activity = self._get_promotion_activity(promotion_id)
        if activity:
            activity.platform_budget += amount
            activity.total_budget += amount
            self.db.commit()

    def _pause_promotion(self, promotion_id: str) -> None:
        """暂停促销活动"""
        activity = self._get_promotion_activity(promotion_id)
        if activity:
            activity.status = PromotionStatus.PAUSED
            self.db.commit()

    def _freeze_promotion(self, promotion_id: str) -> None:
        """冻结促销活动"""
        activity = self._get_promotion_activity(promotion_id)
        if activity:
            activity.status = PromotionStatus.OVER_BUDGET
            self.db.commit()

    def _send_budget_notification(self, promotion_id: str, status: str, severity: str) -> None:
        """发送预算状态通知"""
        logger.info(f"发送预算通知: promotion={promotion_id}, status={status}, severity={severity}")

    def _escalate_to_management(self, promotion_id: str, amount: Decimal, severity: str) -> None:
        """升级到管理层处理"""
        logger.critical(f"预算超支升级: promotion={promotion_id}, amount={amount}, severity={severity}")
```

## 异常场景补充

### 场景：促销补贴计算错误
```
触发条件: 促销订单补贴金额计算结果与实际扣款不一致，常见原因包括：(1)分摊比例配置错误导致平台和商家承担金额之和与总补贴不等；(2)满减规则叠加时多次扣减导致补贴重复计算；(3)退款订单的补贴扣减使用了错误的退款比例；(4)促销活动变更后已存在订单仍使用旧规则计算

检测方式: (1)每日对账任务比对补贴汇总表与订单明细表，发现平台补贴+商家补贴≠总补贴的差异记录；(2)结算前对每笔订单执行校验：platform_subsidy + merchant_subsidy == original_amount - paid_amount，不匹配则标记异常；(3)引入幂等校验，同一订单多次计算结果必须一致，否则触发告警；(4)审计日志对比规则变更时间点前后的计算结果

处理流程: (1)立即暂停受影响促销活动的结算，冻结相关商家账户的结算流程；(2)根据订单时间戳和规则变更记录，确定错误影响的订单范围和时间段；(3)对错误订单进行修正计算：优先以原始促销规则为准重新计算，退款订单按退款协议逐笔修正；(4)生成差异调整单，经运营和财务双人审核后执行补扣或补退；(5)修正完成后重新执行对账，确认差异为零后恢复结算

预防措施: (1)促销规则变更走灰度发布，新旧规则并行计算并比对结果一致后才切换；(2)补贴计算增加断言校验：assert platform_subsidy + merchant_subsidy == total_subsidy，不通过则拒绝结算；(3)退款扣减统一使用退款金额/实付金额作为扣减比例，禁止使用其他计算方式；(4)对叠加促销场景进行排他性校验，同一订单不允许参与互斥活动
```

### 场景：促销活动超预算
```
触发条件: 促销活动进行中，已使用补贴金额超过预设预算上限，常见原因包括：(1)高并发场景下预算扣减未做分布式锁保护，导致超卖；(2)预算预警阈值设置过高（如95%），预警触发后运营未及时处理；(3)限时秒杀活动流量远超预期，短时间内大量订单涌入导致预算迅速耗尽；(4)退款回流金额未及时补充到可用预算中

检测方式: (1)每次补贴扣减后实时比对 used_platform_budget 与 platform_budget，超限立即触发告警；(2)设置多级预算预警：70%/80%/90%分别发送不同级别告警通知；(3)定时任务每5分钟检查一次预算使用率，与实时监控形成双重保障；(4)对账系统每日校验预算消耗曲线是否异常陡峭，斜率超过阈值则提前预警

处理流程: (1)轻微超支(5%以内)：自动追加预算并通知运营，活动继续进行；(2)中度超支(5%-20%)：自动暂停活动，生成超支报告，等待运营决策（追加预算/终止活动/调整分摊比例）；(3)严重超支(20%以上)：立即冻结活动，升级至管理层，启动事后审计；(4)超预算期间产生的订单按"超预算结算规则"处理：超出部分由责任方（配置方）承担；(5)恢复活动前必须确认预算充足并设置更严格的监控阈值

预防措施: (1)预算扣减使用Redis分布式锁+Lua脚本保证原子性，防止并发超卖；(2)设置合理的预警阈值（建议60%/75%/85%三级），确保留出足够响应时间；(3)秒杀等高流量活动提前压测，按峰值1.5倍设置预算；(4)退款回流金额实时补充到可用预算，而非T+1延迟补充；(5)预算使用率达到90%后自动降级：新订单按商家承担模式处理，平台不再追加补贴
```

## 结算系统灰度发布完整实现

```python
from datetime import datetime
from decimal import Decimal, ROUND_HALF_UP
from typing import Dict, List, Optional, Any, Callable
from enum import Enum
import hashlib
import json
import logging
import threading

logger = logging.getLogger(__name__)


class GrayRuleType(Enum):
    """灰度规则类型"""
    MERCHANT_ID_LIST = "merchant_id_list"         # 指定商家ID列表
    MERCHANT_ID_RANGE = "merchant_id_range"       # 商家ID范围
    PERCENTAGE = "percentage"                     # 按百分比灰度
    MERCHANT_TAG = "merchant_tag"                 # 按商家标签
    REGION = "region"                             # 按地区灰度
    SETTLEMENT_TYPE = "settlement_type"           # 按结算类型


class GrayRuleStatus(Enum):
    """灰度规则状态"""
    DRAFT = "draft"
    ACTIVE = "active"
    PAUSED = "paused"
    PROMOTED = "promoted"       # 已全量
    ROLLED_BACK = "rolled_back"


class ComparisonStatus(Enum):
    """对比结果状态"""
    MATCH = "match"
    MISMATCH = "mismatch"
    TOLERANCE = "tolerance"     # 在容忍范围内的差异
    ERROR = "error"


class GrayRule:
    """灰度规则配置"""
    def __init__(
        self,
        rule_id: str,
        rule_name: str,
        rule_type: GrayRuleType,
        rule_config: Dict,
        new_logic_version: str,
        old_logic_version: str,
        tolerance_threshold: Decimal = Decimal("0.01"),
        status: GrayRuleStatus = GrayRuleStatus.DRAFT,
        created_by: str = "",
    ):
        self.rule_id = rule_id
        self.rule_name = rule_name
        self.rule_type = rule_type
        self.rule_config = rule_config
        self.new_logic_version = new_logic_version
        self.old_logic_version = old_logic_version
        self.tolerance_threshold = tolerance_threshold
        self.status = status
        self.created_by = created_by
        self.created_at = datetime.now()
        self.updated_at = datetime.now()
        self.match_count = 0
        self.mismatch_count = 0
        self.tolerance_count = 0
        self.error_count = 0


class ComparisonResult:
    """新旧逻辑对比结果"""
    def __init__(
        self,
        comparison_id: str,
        rule_id: str,
        merchant_id: str,
        order_id: str,
        old_result: Dict,
        new_result: Dict,
        status: ComparisonStatus,
        diff_fields: List[str],
        max_diff_amount: Decimal,
    ):
        self.comparison_id = comparison_id
        self.rule_id = rule_id
        self.merchant_id = merchant_id
        self.order_id = order_id
        self.old_result = old_result
        self.new_result = new_result
        self.status = status
        self.diff_fields = diff_fields
        self.max_diff_amount = max_diff_amount
        self.compared_at = datetime.now()


class SettlementGrayReleaseService:
    """结算系统灰度发布服务

    管理结算系统新旧逻辑的灰度切换，支持按商家ID、百分比、标签、地区等
    多种维度定义灰度规则，自动对比新旧逻辑的结算结果，确保切换前后结果
    一致后才全量发布。核心流程：创建灰度规则 -> 灰度运行 -> 结果对比 ->
    问题修复 -> 全量发布。
    """

    def __init__(self, db_session, config: Optional[Dict] = None):
        self.db = db_session
        self.config = config or {}
        self._rules: Dict[str, GrayRule] = {}
        self._comparison_results: Dict[str, List[ComparisonResult]] = {}
        self._lock = threading.Lock()
        self._new_logic_handler: Optional[Callable] = None
        self._old_logic_handler: Optional[Callable] = None

        # 灰度发布安全配置
        self.max_mismatch_rate = self.config.get("max_mismatch_rate", Decimal("0.01"))
        self.min_sample_size = self.config.get("min_sample_size", 1000)
        self.auto_promote_enabled = self.config.get("auto_promote_enabled", False)

    def register_logic_handlers(
        self,
        old_handler: Callable,
        new_handler: Callable,
    ) -> None:
        """注册新旧逻辑处理函数

        Args:
            old_handler: 旧结算逻辑处理函数，接收订单数据返回结算结果
            new_handler: 新结算逻辑处理函数，接口同上
        """
        self._old_logic_handler = old_handler
        self._new_logic_handler = new_handler
        logger.info("新旧逻辑处理函数已注册")

    def create_gray_rule(
        self,
        rule_name: str,
        rule_type: GrayRuleType,
        rule_config: Dict,
        new_logic_version: str,
        old_logic_version: str,
        tolerance_threshold: Decimal = Decimal("0.01"),
        created_by: str = "",
    ) -> Dict:
        """创建灰度规则

        定义哪些商家使用新结算逻辑，支持多种灰度维度。

        Args:
            rule_name: 规则名称
            rule_type: 规则类型（商家列表/百分比/标签/地区等）
            rule_config: 规则配置，不同类型有不同格式
            new_logic_version: 新逻辑版本号
            old_logic_version: 旧逻辑版本号
            tolerance_threshold: 容忍差异阈值
            created_by: 创建人

        Returns:
            创建的灰度规则信息
        """
        # 校验规则配置
        self._validate_rule_config(rule_type, rule_config)

        rule_id = self._generate_rule_id(rule_name, new_logic_version)

        with self._lock:
            rule = GrayRule(
                rule_id=rule_id,
                rule_name=rule_name,
                rule_type=rule_type,
                rule_config=rule_config,
                new_logic_version=new_logic_version,
                old_logic_version=old_logic_version,
                tolerance_threshold=tolerance_threshold,
                created_by=created_by,
            )

            self._rules[rule_id] = rule
            self._comparison_results[rule_id] = []

        # 持久化
        self._save_gray_rule(rule)

        logger.info(
            f"灰度规则创建成功: rule_id={rule_id}, name={rule_name}, "
            f"type={rule_type.value}, version={new_logic_version}"
        )

        return {
            "rule_id": rule_id,
            "rule_name": rule_name,
            "rule_type": rule_type.value,
            "rule_config": rule_config,
            "new_logic_version": new_logic_version,
            "old_logic_version": old_logic_version,
            "tolerance_threshold": str(tolerance_threshold),
            "status": GrayRuleStatus.DRAFT.value,
            "created_at": rule.created_at.isoformat(),
        }

    def _validate_rule_config(self, rule_type: GrayRuleType, config: Dict) -> None:
        """校验灰度规则配置的合法性"""
        if rule_type == GrayRuleType.MERCHANT_ID_LIST:
            if "merchant_ids" not in config or not isinstance(config["merchant_ids"], list):
                raise ValueError("商家列表规则必须包含 merchant_ids 数组")
            if len(config["merchant_ids"]) == 0:
                raise ValueError("商家列表不能为空")

        elif rule_type == GrayRuleType.PERCENTAGE:
            if "percentage" not in config:
                raise ValueError("百分比规则必须包含 percentage 字段")
            pct = Decimal(str(config["percentage"]))
            if pct <= Decimal("0") or pct > Decimal("100"):
                raise ValueError(f"灰度百分比必须在(0, 100]范围内, 当前: {pct}")

        elif rule_type == GrayRuleType.MERCHANT_TAG:
            if "tags" not in config or not isinstance(config["tags"], list):
                raise ValueError("标签规则必须包含 tags 数组")

        elif rule_type == GrayRuleType.REGION:
            if "regions" not in config or not isinstance(config["regions"], list):
                raise ValueError("地区规则必须包含 regions 数组")

        elif rule_type == GrayRuleType.SETTLEMENT_TYPE:
            if "settlement_types" not in config or not isinstance(config["settlement_types"], list):
                raise ValueError("结算类型规则必须包含 settlement_types 数组")

    def evaluate_gray_rule(
        self,
        rule_id: str,
        merchant_id: str,
        context: Optional[Dict] = None,
    ) -> Dict:
        """判断商家是否命中灰度规则

        根据灰度规则配置，判断指定商家应使用新逻辑还是旧逻辑。
        同时记录判断结果用于后续审计。

        Args:
            rule_id: 灰度规则ID
            merchant_id: 商家ID
            context: 上下文信息（地区、标签等）

        Returns:
            判断结果，包含 use_new_logic, rule_type, version 等
        """
        rule = self._rules.get(rule_id)
        if not rule:
            rule = self._load_gray_rule(rule_id)
            if not rule:
                raise ValueError(f"灰度规则不存在: {rule_id}")

        if rule.status not in (GrayRuleStatus.ACTIVE, GrayRuleStatus.PROMOTED):
            return {
                "use_new_logic": False,
                "reason": f"规则状态为 {rule.status.value}，不生效",
                "rule_id": rule_id,
                "version": rule.old_logic_version,
            }

        # 全量发布后所有商家都走新逻辑
        if rule.status == GrayRuleStatus.PROMOTED:
            return {
                "use_new_logic": True,
                "reason": "已全量发布",
                "rule_id": rule_id,
                "version": rule.new_logic_version,
            }

        use_new_logic = self._evaluate_rule_logic(rule, merchant_id, context or {})

        # 如果命中灰度，则同时执行新旧逻辑进行对比
        if use_new_logic and self._old_logic_handler and self._new_logic_handler:
            self._run_comparison(rule, merchant_id, context)

        result = {
            "use_new_logic": use_new_logic,
            "rule_id": rule_id,
            "rule_type": rule.rule_type.value,
            "version": rule.new_logic_version if use_new_logic else rule.old_logic_version,
            "evaluated_at": datetime.now().isoformat(),
        }

        return result

    def _evaluate_rule_logic(
        self, rule: GrayRule, merchant_id: str, context: Dict,
    ) -> bool:
        """执行具体的规则判断逻辑"""
        if rule.rule_type == GrayRuleType.MERCHANT_ID_LIST:
            return merchant_id in rule.rule_config.get("merchant_ids", [])

        elif rule.rule_type == GrayRuleType.MERCHANT_ID_RANGE:
            start_id = rule.rule_config.get("start_id", "")
            end_id = rule.rule_config.get("end_id", "")
            return start_id <= merchant_id <= end_id

        elif rule.rule_type == GrayRuleType.PERCENTAGE:
            percentage = Decimal(str(rule.rule_config["percentage"]))
            # 使用商家ID的哈希值进行一致性分桶
            hash_value = int(
                hashlib.md5(merchant_id.encode()).hexdigest()[:8], 16
            )
            bucket = (hash_value % 10000) / 100.0
            return bucket < float(percentage)

        elif rule.rule_type == GrayRuleType.MERCHANT_TAG:
            merchant_tags = context.get("tags", [])
            rule_tags = set(rule.rule_config.get("tags", []))
            return bool(set(merchant_tags) & rule_tags)

        elif rule.rule_type == GrayRuleType.REGION:
            merchant_region = context.get("region", "")
            return merchant_region in rule.rule_config.get("regions", [])

        elif rule.rule_type == GrayRuleType.SETTLEMENT_TYPE:
            settlement_type = context.get("settlement_type", "")
            return settlement_type in rule.rule_config.get("settlement_types", [])

        return False

    def compare_results(
        self,
        rule_id: str,
        order_data: Dict,
    ) -> ComparisonResult:
        """对比新旧逻辑的结算结果

        对同一笔订单分别使用新旧逻辑计算，对比结果差异。
        如果差异在容忍阈值内标记为 tolerance，否则标记为 mismatch。

        Args:
            rule_id: 灰度规则ID
            order_data: 订单数据

        Returns:
            对比结果对象
        """
        rule = self._rules.get(rule_id)
        if not rule:
            raise ValueError(f"灰度规则不存在: {rule_id}")

        if not self._old_logic_handler or not self._new_logic_handler:
            raise RuntimeError("新旧逻辑处理函数未注册，无法执行对比")

        order_id = order_data.get("order_id", "unknown")
        merchant_id = order_data.get("merchant_id", "unknown")

        # 执行新旧逻辑
        try:
            old_result = self._old_logic_handler(order_data)
        except Exception as e:
            logger.error(f"旧逻辑执行失败: order={order_id}, error={e}")
            old_result = {"error": str(e)}

        try:
            new_result = self._new_logic_handler(order_data)
        except Exception as e:
            logger.error(f"新逻辑执行失败: order={order_id}, error={e}")
            new_result = {"error": str(e)}

        # 对比结果
        status, diff_fields, max_diff = self._compute_diff(
            old_result, new_result, rule.tolerance_threshold
        )

        comparison_id = f"cmp_{rule_id}_{order_id}_{datetime.now().strftime('%Y%m%d%H%M%S')}"

        result = ComparisonResult(
            comparison_id=comparison_id,
            rule_id=rule_id,
            merchant_id=merchant_id,
            order_id=order_id,
            old_result=old_result,
            new_result=new_result,
            status=status,
            diff_fields=diff_fields,
            max_diff_amount=max_diff,
        )

        # 更新规则统计
        with self._lock:
            if rule_id not in self._comparison_results:
                self._comparison_results[rule_id] = []
            self._comparison_results[rule_id].append(result)

            if status == ComparisonStatus.MATCH:
                rule.match_count += 1
            elif status == ComparisonStatus.TOLERANCE:
                rule.tolerance_count += 1
            elif status == ComparisonStatus.MISMATCH:
                rule.mismatch_count += 1
            else:
                rule.error_count += 1

        # 持久化对比结果
        self._save_comparison_result(result)

        # 检查不一致率是否超标
        self._check_mismatch_rate(rule)

        logger.info(
            f"灰度对比完成: rule={rule_id}, order={order_id}, "
            f"status={status.value}, diff_fields={diff_fields}, max_diff={max_diff}"
        )

        return result

    def _compute_diff(
        self,
        old_result: Dict,
        new_result: Dict,
        tolerance: Decimal,
    ) -> Tuple[ComparisonStatus, List[str], Decimal]:
        """计算新旧结果的差异"""
        if "error" in old_result or "error" in new_result:
            return ComparisonStatus.ERROR, [], Decimal("0")

        diff_fields = []
        max_diff = Decimal("0")

        # 对比金额类字段
        amount_fields = [
            "settlement_amount", "platform_fee", "merchant_amount",
            "commission", "service_fee", "tax_amount",
        ]

        for field in amount_fields:
            old_val = Decimal(str(old_result.get(field, "0")))
            new_val = Decimal(str(new_result.get(field, "0")))
            diff = abs(old_val - new_val)

            if diff > Decimal("0"):
                max_diff = max(max_diff, diff)
                if diff <= tolerance:
                    diff_fields.append(f"{field}(tolerance:{diff})")
                else:
                    diff_fields.append(f"{field}(mismatch:{diff})")

        # 对比非金额字段
        for field in ["settlement_type", "fee_tier", "status"]:
            old_val = old_result.get(field)
            new_val = new_result.get(field)
            if old_val != new_val:
                diff_fields.append(f"{field}({old_val}->{new_val})")

        if not diff_fields:
            return ComparisonStatus.MATCH, [], Decimal("0")

        has_mismatch = any("mismatch:" in f for f in diff_fields)
        if has_mismatch:
            return ComparisonStatus.MISMATCH, diff_fields, max_diff
        else:
            return ComparisonStatus.TOLERANCE, diff_fields, max_diff

    def _check_mismatch_rate(self, rule: GrayRule) -> None:
        """检查不一致率是否超过阈值"""
        total = rule.match_count + rule.tolerance_count + rule.mismatch_count
        if total < 10:  # 样本太少不判断
            return

        mismatch_rate = Decimal(str(rule.mismatch_count)) / Decimal(str(total))
        if mismatch_rate > self.max_mismatch_rate:
            logger.error(
                f"灰度不一致率超标: rule={rule.rule_id}, "
                f"mismatch_rate={mismatch_rate:.4f}, threshold={self.max_mismatch_rate}, "
                f"样本数={total}"
            )
            # 自动暂停灰度规则
            self._pause_gray_rule(rule.rule_id)
            self._alert_mismatch_exceeded(rule, mismatch_rate)

    def promote_to_full(
        self,
        rule_id: str,
        approved_by: str,
        approval_comment: str = "",
    ) -> Dict:
        """将灰度规则全量发布

        在确认新旧逻辑结果一致率达标后，将所有商家切换到新逻辑。
        全量发布前需要满足以下条件：
        1. 样本量达到最低要求
        2. 不一致率低于阈值
        3. 无未处理的严重不一致问题
        4. 已通过人工审批

        Args:
            rule_id: 灰度规则ID
            approved_by: 审批人
            approval_comment: 审批备注

        Returns:
            全量发布结果

        Raises:
            RuntimeError: 不满足全量发布条件
        """
        rule = self._rules.get(rule_id)
        if not rule:
            raise ValueError(f"灰度规则不存在: {rule_id}")

        if rule.status == GrayRuleStatus.PROMOTED:
            return {
                "rule_id": rule_id,
                "status": "already_promoted",
                "message": "该规则已经全量发布",
            }

        # 校验全量发布条件
        total_comparisons = (
            rule.match_count + rule.tolerance_count
            + rule.mismatch_count + rule.error_count
        )

        validation_errors = []

        # 条件1: 样本量检查
        if total_comparisons < self.min_sample_size:
            validation_errors.append(
                f"样本量不足: 需要 {self.min_sample_size}, 当前 {total_comparisons}"
            )

        # 条件2: 不一致率检查
        if total_comparisons > 0:
            mismatch_rate = Decimal(str(rule.mismatch_count)) / Decimal(str(total_comparisons))
            if mismatch_rate > self.max_mismatch_rate:
                validation_errors.append(
                    f"不一致率超标: 阈值 {self.max_mismatch_rate}, 当前 {mismatch_rate:.4f}"
                )

        # 条件3: 无未处理的严重不一致
        recent_mismatches = self._get_unresolved_mismatches(rule_id)
        if recent_mismatches:
            validation_errors.append(
                f"存在 {len(recent_mismatches)} 条未处理的严重不一致记录"
            )

        if validation_errors:
            raise RuntimeError(
                f"不满足全量发布条件: " + "; ".join(validation_errors)
            )

        # 执行全量发布
        with self._lock:
            old_status = rule.status
            rule.status = GrayRuleStatus.PROMOTED
            rule.updated_at = datetime.now()

        # 记录发布历史
        promote_record = {
            "rule_id": rule_id,
            "rule_name": rule.rule_name,
            "old_status": old_status.value,
            "new_status": GrayRuleStatus.PROMOTED.value,
            "old_logic_version": rule.old_logic_version,
            "new_logic_version": rule.new_logic_version,
            "statistics": {
                "total_comparisons": total_comparisons,
                "match_count": rule.match_count,
                "tolerance_count": rule.tolerance_count,
                "mismatch_count": rule.mismatch_count,
                "error_count": rule.error_count,
                "match_rate": str(
                    Decimal(str(rule.match_count)) / Decimal(str(total_comparisons))
                    if total_comparisons > 0 else Decimal("0")
                ),
            },
            "approved_by": approved_by,
            "approval_comment": approval_comment,
            "promoted_at": datetime.now().isoformat(),
        }

        self._save_promote_record(promote_record)
        self._save_gray_rule(rule)

        # 通知全量发布
        self._notify_promotion(rule, approved_by)

        logger.info(
            f"灰度全量发布完成: rule={rule_id}, "
            f"version={rule.new_logic_version}, approved_by={approved_by}"
        )

        return promote_record

    def _run_comparison(
        self, rule: GrayRule, merchant_id: str, context: Optional[Dict],
    ) -> None:
        """异步执行新旧逻辑对比"""
        try:
            order_data = context or {}
            order_data["merchant_id"] = merchant_id
            self.compare_results(rule.rule_id, order_data)
        except Exception as e:
            logger.error(f"灰度对比执行异常: rule={rule.rule_id}, error={e}")

    def _pause_gray_rule(self, rule_id: str) -> None:
        """暂停灰度规则"""
        rule = self._rules.get(rule_id)
        if rule:
            rule.status = GrayRuleStatus.PAUSED
            rule.updated_at = datetime.now()
            self._save_gray_rule(rule)

    def _alert_mismatch_exceeded(self, rule: GrayRule, rate: Decimal) -> None:
        """发送不一致率超标告警"""
        logger.critical(
            f"灰度不一致率超标告警: rule={rule.rule_id}, rate={rate:.4f}"
        )

    def _get_unresolved_mismatches(self, rule_id: str) -> List[Dict]:
        """获取未处理的严重不一致记录"""
        results = self._comparison_results.get(rule_id, [])
        return [
            {"comparison_id": r.comparison_id, "order_id": r.order_id, "diff_fields": r.diff_fields}
            for r in results
            if r.status == ComparisonStatus.MISMATCH
        ]

    def _generate_rule_id(self, name: str, version: str) -> str:
        """生成灰度规则ID"""
        raw = f"{name}_{version}_{datetime.now().strftime('%Y%m%d%H%M%S')}"
        return f"gray_{hashlib.md5(raw.encode()).hexdigest()[:12]}"

    def _load_gray_rule(self, rule_id: str) -> Optional[GrayRule]:
        """从数据库加载灰度规则"""
        record = self.db.query(GrayRule).filter_by(rule_id=rule_id).first()
        if record:
            self._rules[rule_id] = record
        return record

    def _save_gray_rule(self, rule: GrayRule) -> None:
        """持久化灰度规则"""
        self.db.merge(rule)
        self.db.commit()

    def _save_comparison_result(self, result: ComparisonResult) -> None:
        """持久化对比结果"""
        record = {
            "comparison_id": result.comparison_id,
            "rule_id": result.rule_id,
            "merchant_id": result.merchant_id,
            "order_id": result.order_id,
            "old_result": json.dumps(result.old_result, default=str),
            "new_result": json.dumps(result.new_result, default=str),
            "status": result.status.value,
            "diff_fields": json.dumps(result.diff_fields),
            "max_diff_amount": str(result.max_diff_amount),
            "compared_at": result.compared_at.isoformat(),
        }
        self.db.insert("gray_comparison_results", record)
        self.db.commit()

    def _save_promote_record(self, record: Dict) -> None:
        """持久化全量发布记录"""
        self.db.insert("gray_promote_history", record)
        self.db.commit()

    def _notify_promotion(self, rule: GrayRule, approved_by: str) -> None:
        """通知全量发布"""
        logger.info(
            f"全量发布通知: rule={rule.rule_id}, version={rule.new_logic_version}, "
            f"approved_by={approved_by}"
        )
```

## 异常场景补充

### 场景：灰度结果不一致
```
触发条件: 灰度发布期间，同一笔订单在新旧结算逻辑下计算出不同的结果，常见原因包括：(1)新逻辑修复了旧逻辑的已知bug，导致结果"正确地不一致"；(2)新逻辑引入了精度计算方式变更（如从float改为Decimal），导致尾差；(3)新逻辑的费率配置加载时机不同，使用了不同时间点的费率；(4)新逻辑对边界条件（零金额、全额退款等）的处理方式不同

检测方式: (1)灰度期间对命中商家的每笔订单同时执行新旧逻辑，自动对比结算金额、费率、状态等关键字段；(2)设置容忍阈值（默认0.01元），超出阈值的不一致标记为严重等级；(3)实时监控不一致率 = mismatch_count / total_count，超过1%自动暂停灰度；(4)每日生成不一致分析报告，按差异类型分类统计

处理流程: (1)立即暂停灰度规则，停止新商家进入灰度范围；(2)对不一致记录分类：bug修复类（记录但放行）、精度差异类（评估是否可接受）、配置错误类（必须修复）、逻辑错误类（必须修复）；(3)bug修复类：在灰度规则中配置豁免字段，标记该差异为预期差异；(4)精度差异类：如差异在0.01元以内，调整容忍阈值；超出则排查精度计算逻辑；(5)配置错误类：修复配置加载逻辑，确保新旧逻辑使用同一时间点的配置；(6)逻辑错误类：修复新逻辑bug，重新开始灰度验证；(7)所有不一致问题解决后，清零统计计数，重新开始灰度

预防措施: (1)新逻辑上线前必须在测试环境运行全量历史订单对比，一致率需达到99.99%；(2)对于预期内的差异（bug修复），在灰度规则中预先配置差异说明，避免误报；(3)金额计算统一使用Decimal类型，禁止使用float；(4)配置加载使用快照机制，确保计算过程中配置不变化；(5)灰度初期设置较低的灰度比例（如1%），观察稳定后再逐步扩大
```

### 场景：灰度切换导致部分商家结算错误
```
触发条件: 灰度规则切换过程中，部分商家的结算使用了错误的逻辑版本，常见原因包括：(1)灰度规则更新时存在短暂的服务间不一致窗口，A服务使用新规则判断，B服务仍使用旧规则；(2)一致性哈希分桶算法存在边界问题，某些商家ID在不同服务实例上的分桶结果不同；(3)灰度规则回滚时，正在处理中的订单使用了回滚后的规则导致结果错误；(4)商家标签变更后灰度规则未及时重新评估

检测方式: (1)每笔结算订单记录使用的逻辑版本号，事后审计比对版本号是否与灰度规则匹配；(2)对每个商家定期校验：最近N笔订单的
### 3.26 商家结算对账差异处理

```python
class SettlementDiscrepancyHandler:
    """对账差异处理：自动差异分类 → 人工审核 → 调整入账"""

    DISCREPANCY_TYPES = {
        "timing": "时间差（跨日结算导致）",
        "coupon_rounding": "优惠分摊尾差",
        "refund_timing": "退款时间差",
        "duplicate": "重复入账",
        "missing": "遗漏入账",
        "rate_change": "费率变更时间差",
        "unknown": "未知原因",
    }

    def classify_discrepancy(self, discrepancy):
        """自动分类差异"""
        order_id = discrepancy["order_id"]
        diff_amount = Decimal(str(discrepancy["diff_amount"]))

        # 1. 检查是否跨日
        order = self.db.get_order(order_id)
        settlement = self.db.query_one(
            "SELECT * FROM settlement_ledger WHERE order_id = %s", order_id)

        if order and settlement:
            order_date = order["completed_at"].date()
            settlement_date = settlement["created_at"].date()

            if order_date != settlement_date:
                return {"type": "timing",
                        "detail": f"订单完成日 {order_date} 与结算日 {settlement_date} 不同"}

        # 2. 检查优惠分摊尾差
        if abs(diff_amount) <= Decimal("0.05"):
            coupon_details = json.loads(settlement.get("coupon_details", "[]"))
            hybrid_coupons = [c for c in coupon_details if c.get("type") == "hybrid"]
            if hybrid_coupons:
                return {"type": "coupon_rounding",
                        "detail": f"混合券分摊尾差 {diff_amount}"}

        # 3. 检查退款时间差
        refunds = self.db.query(
            "SELECT * FROM refunds WHERE order_id = %s", order_id)
        if refunds:
            for refund in refunds:
                if settlement["created_at"] < refund["created_at"]:
                    return {"type": "refund_timing",
                            "detail": f"退款在结算之后处理"}

        # 4. 检查重复入账
        duplicate = self.db.query(
            "SELECT * FROM settlement_ledger "
            "WHERE order_id = %s AND status != 'cancelled'", order_id)
        if len(duplicate) > 1:
            return {"type": "duplicate",
                    "detail": f"同一订单有 {len(duplicate)} 条结算记录"}

        # 5. 检查费率变更
        merchant_id = discrepancy.get("merchant_id")
        rate_changes = self.db.query(
            "SELECT * FROM commission_rate_changes "
            "WHERE merchant_id = %s "
            "AND changed_at BETWEEN %s AND %s",
            merchant_id,
            order["completed_at"] - timedelta(hours=1),
            order["completed_at"] + timedelta(hours=1))
        if rate_changes:
            return {"type": "rate_change",
                    "detail": "结算时费率发生了变更"}

        return {"type": "unknown", "detail": f"差异金额 {diff_amount}"}

    def auto_resolve_discrepancy(self, discrepancy):
        """自动解决小差异"""
        diff_amount = Decimal(str(discrepancy["diff_amount"]))
        classification = self.classify_discrepancy(discrepancy)

        # 尾差自动调整
        if classification["type"] == "coupon_rounding" and abs(diff_amount) <= Decimal("0.05"):
            self._adjust_settlement(discrepancy, diff_amount,
                reason="优惠分摊尾差自动调整")
            return {"status": "auto_resolved", "type": "coupon_rounding"}

        # 时间差自动标记
        if classification["type"] == "timing":
            self._mark_as_timing_difference(discrepancy)
            return {"status": "auto_resolved", "type": "timing"}

        # 重复入账自动冲正
        if classification["type"] == "duplicate":
            self._cancel_duplicate(discrepancy)
            return {"status": "auto_resolved", "type": "duplicate"}

        # 其他 → 人工审核
        self._create_review_task(discrepancy, classification)
        return {"status": "pending_review", "type": classification["type"]}

    def _adjust_settlement(self, discrepancy, diff_amount, reason):
        """调整结算记录"""
        order_id = discrepancy["order_id"]
        settlement = self.db.query_one(
            "SELECT * FROM settlement_ledger WHERE order_id = %s AND status != 'cancelled'",
            order_id)

        new_income = Decimal(settlement["merchant_income"]) + diff_amount
        self.db.update("settlement_ledger",
            {"merchant_income": str(new_income),
             "adjustment": str(diff_amount),
             "adjustment_reason": reason,
             "adjusted_at": now()},
            {"ledger_id": settlement["ledger_id"]})

        self.db.insert("settlement_adjustments", {
            "adjustment_id": str(uuid4()),
            "order_id": order_id,
            "ledger_id": settlement["ledger_id"],
            "adjustment_amount": str(diff_amount),
            "reason": reason,
            "type": "auto",
            "created_at": now()
        })

    def _cancel_duplicate(self, discrepancy):
        """取消重复记录"""
        records = self.db.query(
            "SELECT * FROM settlement_ledger "
            "WHERE order_id = %s AND status != 'cancelled' "
            "ORDER BY created_at ASC",
            discrepancy["order_id"])

        # 保留最早的一条，取消其余
        for record in records[1:]:
            self.db.update("settlement_ledger",
                {"status": "cancelled", "cancelled_reason": "重复入账",
                 "cancelled_at": now()},
                {"ledger_id": record["ledger_id"]})

    def _create_review_task(self, discrepancy, classification):
        """创建人工审核任务"""
        self.db.insert("discrepancy_reviews", {
            "review_id": str(uuid4()),
            "discrepancy_id": discrepancy.get("id"),
            "order_id": discrepancy["order_id"],
            "merchant_id": discrepancy.get("merchant_id"),
            "diff_amount": discrepancy["diff_amount"],
            "classification_type": classification["type"],
            "classification_detail": classification["detail"],
            "status": "pending",
            "created_at": now()
        })
```

### 3.27 结算系统配置管理

```python
class SettlementConfigService:
    """结算配置管理：参数管理 → 变更审批 → 版本管理 → 灰度生效"""

    CONFIG_DEFINITIONS = {
        "commission_rates": {
            "type": "json",
            "description": "各品类佣金费率",
            "requires_approval": True,
        },
        "min_settlement_amount": {
            "type": "decimal",
            "description": "最低结算金额",
            "requires_approval": True,
            "min_value": 1,
            "max_value": 10000,
        },
        "settlement_batch_size": {
            "type": "integer",
            "description": "结算批处理大小",
            "requires_approval": False,
            "min_value": 10,
            "max_value": 10000,
        },
        "payout_retry_max": {
            "type": "integer",
            "description": "打款重试最大次数",
            "requires_approval": False,
            "min_value": 1,
            "max_value": 10,
        },
        "auto_approve_threshold": {
            "type": "decimal",
            "description": "自动审批金额阈值",
            "requires_approval": True,
            "min_value": 0,
            "max_value": 1000000,
        },
    }

    def update_config(self, config_key, new_value, operator_id, reason=None):
        """更新配置"""
        if config_key not in self.CONFIG_DEFINITIONS:
            raise ValueError(f"未知配置: {config_key}")

        definition = self.CONFIG_DEFINITIONS[config_key]

        # 1. 类型验证
        self._validate_value(config_key, new_value, definition)

        # 2. 获取当前值
        current_value = self._get_current_value(config_key)

        # 3. 检查是否需要审批
        if definition["requires_approval"]:
            change_id = str(uuid4())
            self.db.insert("config_change_requests", {
                "change_id": change_id,
                "config_key": config_key,
                "current_value": str(current_value),
                "proposed_value": str(new_value),
                "operator_id": operator_id,
                "reason": reason,
                "status": "pending_approval",
                "created_at": now()
            })

            # 通知审批人
            self.notification.send("config_approvers",
                f"配置变更审批: {config_key} 从 {current_value} 改为 {new_value}")

            return {"status": "pending_approval", "change_id": change_id}

        # 4. 不需要审批 → 直接生效
        self._apply_config(config_key, new_value, operator_id, reason)
        return {"status": "applied", "config_key": config_key,
                "old_value": str(current_value), "new_value": str(new_value)}

    def approve_config_change(self, change_id, approver_id, decision, note=None):
        """审批配置变更"""
        change = self.db.get_config_change(change_id)

        if change["status"] != "pending_approval":
            return {"status": "already_processed"}

        if decision == "approved":
            self._apply_config(change["config_key"], change["proposed_value"],
                change["operator_id"], change.get("reason"))
            self.db.update("config_change_requests",
                {"status": "approved", "approver_id": approver_id,
                 "approval_note": note, "approved_at": now()},
                {"change_id": change_id})
        else:
            self.db.update("config_change_requests",
                {"status": "rejected", "approver_id": approver_id,
                 "rejection_note": note},
                {"change_id": change_id})

        return {"change_id": change_id, "decision": decision}

    def _apply_config(self, config_key, value, operator_id, reason=None):
        """应用配置"""
        # 保存旧值到版本历史
        old_value = self._get_current_value(config_key)

        self.db.insert("config_version_history", {
            "version_id": str(uuid4()),
            "config_key": config_key,
            "old_value": str(old_value) if old_value else None,
            "new_value": str(value),
            "operator_id": operator_id,
            "reason": reason,
            "applied_at": now()
        })

        # 写入当前配置
        self.db.upsert("settlement_configs", {
            "config_key": config_key,
            "config_value": str(value),
            "updated_at": now(),
            "updated_by": operator_id
        }, conflict_columns=["config_key"])

        # 更新缓存
        self.redis.set(f"settlement_config:{config_key}", str(value))

    def _get_current_value(self, config_key):
        """获取当前配置值"""
        cached = self.redis.get(f"settlement_config:{config_key}")
        if cached:
            return cached.decode() if isinstance(cached, bytes) else cached

        result = self.db.query_one(
            "SELECT config_value FROM settlement_configs WHERE config_key = %s",
            config_key)
        return result["config_value"] if result else None

    def _validate_value(self, config_key, value, definition):
        """验证配置值"""
        if definition["type"] == "decimal":
            try:
                val = Decimal(str(value))
            except Exception:
                raise ValueError(f"配置 {config_key} 需要数值类型")

            if "min_value" in definition and val < Decimal(str(definition["min_value"])):
                raise ValueError(f"配置 {config_key} 不能小于 {definition['min_value']}")
            if "max_value" in definition and val > Decimal(str(definition["max_value"])):
                raise ValueError(f"配置 {config_key} 不能大于 {definition['max_value']}")

        elif definition["type"] == "integer":
            try:
                val = int(value)
            except Exception:
                raise ValueError(f"配置 {config_key} 需要整数类型")

            if "min_value" in definition and val < definition["min_value"]:
                raise ValueError(f"配置 {config_key} 不能小于 {definition['min_value']}")

        elif definition["type"] == "json":
            try:
                json.loads(str(value))
            except Exception:
                raise ValueError(f"配置 {config_key} 需要有效 JSON")
```

### 3.28 商家结算通知中心

```python
class SettlementNotificationService:
    """结算通知：多渠道通知 + 模板管理 + 通知偏好 + 已读追踪"""

    NOTIFICATION_TYPES = {
        "settlement_recorded": "结算入账通知",
        "bill_generated": "账单生成通知",
        "payout_success": "打款成功通知",
        "payout_failed": "打款失败通知",
        "refund_offset": "退款冲抵通知",
        "deposit_warning": "保证金预警",
        "rate_change": "费率变更通知",
        "contract_expiring": "合同到期通知",
        "dispute_update": "争议处理通知",
    }

    CHANNELS = {
        "in_app": "站内信",
        "sms": "短信",
        "email": "邮件",
        "wechat": "微信",
    }

    def send_notification(self, merchant_id, notification_type, data,
                         channels=None):
        """发送结算通知"""
        if notification_type not in self.NOTIFICATION_TYPES:
            raise ValueError(f"未知通知类型: {notification_type}")

        # 1. 获取商家通知偏好
        preferences = self._get_notification_preferences(merchant_id)
        target_channels = channels or preferences.get(notification_type, ["in_app"])

        # 2. 渲染通知内容
        template = self._get_template(notification_type)
        content = self._render_template(template, data)

        # 3. 发送到各渠道
        notification_id = str(uuid4())
        results = {}

        for channel in target_channels:
            if channel not in self.CHANNELS:
                continue

            try:
                if channel == "in_app":
                    self._send_in_app(merchant_id, notification_id, content)
                elif channel == "sms":
                    self._send_sms(merchant_id, content)
                elif channel == "email":
                    self._send_email(merchant_id, content)
                elif channel == "wechat":
                    self._send_wechat(merchant_id, content)

                results[channel] = "sent"
            except Exception as e:
                results[channel] = f"failed: {str(e)}"

        # 4. 记录通知
        self.db.insert("settlement_notifications", {
            "notification_id": notification_id,
            "merchant_id": merchant_id,
            "type": notification_type,
            "content": json.dumps(content),
            "channels": json.dumps(results),
            "data": json.dumps(data, default=str),
            "status": "sent" if all(v == "sent" for v in results.values()) else "partial",
            "sent_at": now()
        })

        return {"notification_id": notification_id, "channels": results}

    def mark_as_read(self, notification_id, merchant_id):
        """标记已读"""
        self.db.update("settlement_notifications",
            {"read_at": now()},
            {"notification_id": notification_id, "merchant_id": merchant_id})

    def get_unread_count(self, merchant_id):
        """获取未读数量"""
        return self.db.count("settlement_notifications",
            merchant_id=merchant_id, read_at=None)

    def get_notification_history(self, merchant_id, page=1, page_size=20,
                                notification_type=None):
        """获取通知历史"""
        conditions = ["merchant_id = %s"]
        params = [merchant_id]

        if notification_type:
            conditions.append("type = %s")
            params.append(notification_type)

        where = " AND ".join(conditions)

        results = self.db.query(
            f"SELECT * FROM settlement_notifications WHERE {where} "
            f"ORDER BY sent_at DESC LIMIT %s OFFSET %s",
            *params, page_size, (page-1)*page_size)

        return {"notifications": results, "page": page}

    def _get_notification_preferences(self, merchant_id):
        """获取商家通知偏好"""
        prefs = self.redis.get(f"notification_prefs:{merchant_id}")
        if prefs:
            return json.loads(prefs)

        # 默认偏好
        return {
            "settlement_recorded": ["in_app"],
            "bill_generated": ["in_app", "email"],
            "payout_success": ["in_app", "sms", "email"],
            "payout_failed": ["in_app", "sms"],
            "refund_offset": ["in_app"],
            "deposit_warning": ["in_app", "sms"],
            "rate_change": ["in_app", "email"],
            "contract_expiring": ["in_app", "sms", "email"],
            "dispute_update": ["in_app", "email"],
        }

    def _get_template(self, notification_type):
        """获取通知模板"""
        templates = {
            "settlement_recorded": "您有一笔新的结算入账：{amount}元（订单号：{order_id}）",
            "bill_generated": "结算账单已生成：{amount}元（{period}），包含{order_count}笔订单",
            "payout_success": "结算款项{amount}元已到账，请查收",
            "payout_failed": "结算打款失败：{reason}，我们将尽快重试",
            "refund_offset": "订单{order_id}退款{amount}元已从待结算中扣除",
            "deposit_warning": "保证金余额不足：当前{balance}元，要求{required}元，请及时补缴",
            "rate_change": "您的佣金费率已调整为{new_rate}%（{reason}）",
            "contract_expiring": "您的合同将于{days_left}天后到期，请及时续约",
            "dispute_update": "您提交的结算争议已{status}：{detail}",
        }
        return templates.get(notification_type, "{message}")

    def _render_template(self, template, data):
        """渲染模板"""
        try:
            return template.format(**data)
        except KeyError:
            return template

    def _send_in_app(self, merchant_id, notification_id, content):
        """站内信"""
        self.redis.rpush(f"inbox:{merchant_id}", json.dumps({
            "notification_id": notification_id,
            "content": content,
            "sent_at": now().isoformat()
        }))

    def _send_sms(self, merchant_id, content):
        """短信"""
        merchant = self.db.get_merchant(merchant_id)
        phone = merchant.get("phone")
        if phone:
            self.sms_gateway.send(phone, content)

    def _send_email(self, merchant_id, content):
        """邮件"""
        merchant = self.db.get_merchant(merchant_id)
        email = merchant.get("email")
        if email:
            self.email_gateway.send(email, "结算通知", content)

    def _send_wechat(self, merchant_id, content):
        """微信"""
        merchant = self.db.get_merchant(merchant_id)
        openid = merchant.get("wechat_openid")
        if openid:
            self.wechat_gateway.send_template(openid, content)
```

### 3.29 结算系统性能监控

```python
class SettlementPerformanceMonitor:
    """性能监控：关键路径耗时 → 瓶颈定位 → 容量规划"""

    CRITICAL_PATHS = {
        "calculate_settlement": {"p99_target_ms": 50, "p95_target_ms": 20},
        "record_settlement": {"p99_target_ms": 100, "p95_target_ms": 50},
        "generate_bill": {"p99_target_ms": 5000, "p95_target_ms": 2000},
        "execute_payout": {"p99_target_ms": 3000, "p95_target_ms": 1000},
        "refund_offset": {"p99_target_ms": 200, "p95_target_ms": 100},
        "reconciliation": {"p99_target_ms": 30000, "p95_target_ms": 10000},
    }

    def record_latency(self, path_name, duration_ms, metadata=None):
        """记录关键路径耗时"""
        self.redis.lpush(f"latency:{path_name}", json.dumps({
            "duration_ms": duration_ms,
            "timestamp": now().isoformat(),
            "metadata": metadata or {}
        }))
        # 只保留最近 1000 条
        self.redis.ltrim(f"latency:{path_name}", 0, 999)

        # 检查是否超 SLA
        target = self.CRITICAL_PATHS.get(path_name, {})
        if duration_ms > target.get("p99_target_ms", float('inf')):
            self.alert(f"结算性能超 SLA: {path_name} 耗时 {duration_ms}ms "
                      f"(目标 P99: {target.get('p99_target_ms')}ms)")

    def get_latency_stats(self, path_name, window_minutes=60):
        """获取延迟统计"""
        data = self.redis.lrange(f"latency:{path_name}", 0, -1)

        if not data:
            return {"path": path_name, "sample_count": 0}

        durations = []
        cutoff = now() - timedelta(minutes=window_minutes)

        for item in data:
            record = json.loads(item)
            timestamp = datetime.fromisoformat(record["timestamp"])
            if timestamp >= cutoff:
                durations.append(record["duration_ms"])

        if not durations:
            return {"path": path_name, "sample_count": 0}

        durations.sort()

        return {
            "path": path_name,
            "window_minutes": window_minutes,
            "sample_count": len(durations),
            "p50": durations[int(len(durations) * 0.5)],
            "p90": durations[int(len(durations) * 0.9)],
            "p95": durations[int(len(durations) * 0.95)],
            "p99": durations[int(len(durations) * 0.99)] if len(durations) > 1 else durations[0],
            "max": durations[-1],
            "min": durations[0],
            "avg": round(sum(durations) / len(durations), 1),
        }

    def detect_performance_degradation(self):
        """检测性能退化"""
        degradations = []

        for path_name, targets in self.CRITICAL_PATHS.items():
            stats = self.get_latency_stats(path_name)

            if stats["sample_count"] < 10:
                continue

            # 比较当前 P99 与目标
            if stats["p99"] > targets["p99_target_ms"]:
                degradations.append({
                    "path": path_name,
                    "current_p99": stats["p99"],
                    "target_p99": targets["p99_target_ms"],
                    "degradation_pct": round(
                        (stats["p99"] - targets["p99_target_ms"]) / targets["p99_target_ms"] * 100, 1),
                    "sample_count": stats["sample_count"]
                })

        degradations.sort(key=lambda d: d["degradation_pct"], reverse=True)

        return {"degradation_count": len(degradations),
                "degradations": degradations}

    def capacity_planning_report(self):
        """容量规划报告"""
        # 当前吞吐量
        daily_settlements = self.db.count("settlement_ledger",
            created_at__gte=now()-timedelta(days=1))

        # 峰值吞吐量
        peak_hour = self.db.query_one(
            "SELECT DATE_FORMAT(created_at, '%%Y-%%m-%%d %%H:00') as hour, "
            "COUNT(*) as count FROM settlement_ledger "
            "WHERE created_at > NOW() - INTERVAL 7 DAY "
            "GROUP BY hour ORDER BY count DESC LIMIT 1")

        # 平均处理时间
        avg_settlement_time = self.get_latency_stats("record_settlement")

        # 预测
        growth_rate = self._estimate_growth_rate()
        projected_daily = daily_settlements * (1 + growth_rate)

        return {
            "current_daily_throughput": daily_settlements,
            "peak_hourly_throughput": peak_hour["count"] if peak_hour else 0,
            "avg_record_latency_ms": avg_settlement_time.get("avg", 0),
            "growth_rate_monthly": round(growth_rate * 100, 1),
            "projected_daily_30d": round(projected_daily),
            "recommendation": self._generate_capacity_recommendation(
                daily_settlements, projected_daily, avg_settlement_time)
        }

    def _estimate_growth_rate(self):
        """估算增长率"""
        this_week = self.db.count("settlement_ledger",
            created_at__gte=now()-timedelta(days=7))
        last_week = self.db.count("settlement_ledger",
            created_at__gte=now()-timedelta(days=14),
            created_at__lt=now()-timedelta(days=7))

        if last_week > 0:
            return (this_week - last_week) / last_week
        return 0.05  # 默认 5% 月增长

    def _generate_capacity_recommendation(self, current, projected, latency_stats):
        """生成容量建议"""
        recommendations = []

        if projected > current * 1.5:
            recommendations.append("预计 30 天内吞吐量增长 50%，建议扩容")

        p99 = latency_stats.get("p99", 0)
        if p99 > 200:
            recommendations.append("结算入账延迟过高，建议优化数据库写入或增加写入节点")

        return recommendations or ["当前容量充足，无需调整"]
```

## 十三、更多异常场景

### 场景 16：通知发送失败导致商家错过重要信息

```
触发：打款失败通知发送失败 → 商家不知晓 → 未及时处理 → 资金延迟
检测：
  1. 通知发送失败率 > 1% → 通知问题
  2. 关键通知（打款失败）无送达记录 → 发送失败
处理：
  1. 关键通知多渠道冗余发送
  2. 发送失败自动重试（3 次）
  3. 重试仍失败 → 人工通知
预防：多渠道冗余 + 自动重试 + 人工兜底
```

### 场景 17：配置变更未生效导致旧逻辑结算

```
触发：费率从 5% 改为 3% → 缓存未更新 → 仍按 5% 结算 → 商家少收
检测：
  1. 配置变更后仍使用旧值 → 缓存未刷新
  2. 结算结果与预期费率不符 → 配置未生效
处理：
  1. 配置变更后主动刷新缓存
  2. 结算时双读（缓存 + DB 交叉验证）
  3. 发现不一致 → 追溯修正
预防：主动刷新 + 双读验证 + 追溯修正
```

### 场景 18：结算系统性能突然退化

```
触发：数据库连接池耗尽 → 结算入账耗时从 50ms 暴增到 5s → 订单积压
检测：
  1. P99 延迟超过 SLA → 性能退化
  2. 待结算记录数突增 → 处理不过来
处理：
  1. 扩大数据库连接池
  2. 降级：批量写入替代逐条写入
  3. 紧急扩容数据库节点
预防：连接池监控 + 批量写入 + 弹性扩容
```
逻辑版本是否一致，如不一致且该商家灰度规则未变更则标记异常；(3)监控系统检测到某商家在短时间内结算金额出现跳变（与历史均值偏差超过20%），触发告警；(4)灰度切换后24小时内对全量商家执行一次新旧逻辑对比抽检

处理流程: (1)立即暂停灰度规则，回滚到旧逻辑版本，确保不再产生新的错误结算；(2)定位受影响商家清单：查询灰度切换时间窗口内所有结算记录，比对逻辑版本与灰度规则是否匹配；(3)对错误结算的商家逐笔修正：回退错误结算、按正确逻辑重新结算、补差或扣差；(4)排查根因：是服务间同步延迟、哈希边界问题还是规则缓存失效；(5)修复根因后，从更小的灰度比例（0.5%）重新开始灰度验证；(6)增加灰度切换的灰度比例调整步长限制：每次最多扩大一倍，且需观察至少24小时

预防措施: (1)灰度规则使用配置中心下发，确保所有服务实例在同一时间点收到规则变更，变更前等待所有实例确认接收；(2)一致性哈希使用确定性的分桶算法，对所有服务实例使用相同的哈希种子，禁止使用随机种子；(3)灰度规则变更期间设置缓冲期：新规则下发后延迟5分钟生效，确保所有实例完成同步；(4)灰度回滚操作必须等待当前处理中的订单完成后再执行，禁止强制中断；(5)商家标签变更后自动触发灰度规则重新评估，并通知相关服务更新缓存
```

## 商家结算风险控制完整实现

```python
from datetime import datetime, timedelta
from decimal import Decimal, ROUND_HALF_UP
from typing import Dict, List, Optional, Tuple
from enum import Enum
import logging
import threading

logger = logging.getLogger(__name__)


class RiskLevel(Enum):
    """风险等级"""
    LOW = "low"                 # 低风险：正常监控
    MEDIUM = "medium"           # 中风险：延迟结算
    HIGH = "high"               # 高风险：增加保证金
    CRITICAL = "critical"       # 极高风险：冻结账户


class RiskMeasureType(Enum):
    """风险措施类型"""
    NORMAL_MONITORING = "normal_monitoring"         # 正常监控
    DELAYED_SETTLEMENT = "delayed_settlement"       # 延迟结算
    INCREASED_DEPOSIT = "increased_deposit"         # 增加保证金
    PARTIAL_FREEZE = "partial_freeze"               # 部分冻结
    FULL_FREEZE = "full_freeze"                     # 全额冻结
    ACCOUNT_RESTRICTION = "account_restriction"     # 账户限制


class RiskFactorType(Enum):
    """风险因子类型"""
    REFUND_RATE = "refund_rate"                     # 退款率
    COMPLAINT_RATE = "complaint_rate"               # 投诉率
    TRANSACTION_ANOMALY = "transaction_anomaly"     # 交易异常
    ACCOUNT_AGE = "account_age"                     # 账户年龄
    AVERAGE_ORDER_VALUE = "average_order_value"     # 平均订单金额
    SETTLEMENT_FREQUENCY = "settlement_frequency"   # 结算频率
    CHARGEBACK_RATE = "chargeback_rate"             # 拒付率
    FRAUD_HISTORY = "fraud_history"                 # 欺诈历史


class RiskAssessment:
    """风险评估结果"""
    def __init__(
        self,
        merchant_id: str,
        overall_score: Decimal,
        risk_level: RiskLevel,
        factor_scores: Dict[str, Decimal],
        contributing_factors: List[str],
        recommendation: str,
    ):
        self.merchant_id = merchant_id
        self.overall_score = overall_score
        self.risk_level = risk_level
        self.factor_scores = factor_scores
        self.contributing_factors = contributing_factors
        self.recommendation = recommendation
        self.assessed_at = datetime.now()


class RiskDecision:
    """风险决策"""
    def __init__(
        self,
        decision_id: str,
        merchant_id: str,
        risk_level: RiskLevel,
        measure_type: RiskMeasureType,
        measure_details: Dict,
        effective_from: datetime,
        effective_until: Optional[datetime],
        is_auto: bool,
        decided_by: str,
        status: str = "active",
        review_status: str = "pending" if is_auto else "approved",
    ):
        self.decision_id = decision_id
        self.merchant_id = merchant_id
        self.risk_level = risk_level
        self.measure_type = measure_type
        self.measure_details = measure_details
        self.effective_from = effective_from
        self.effective_until = effective_until
        self.is_auto = is_auto
        self.decided_by = decided_by
        self.status = status
        self.review_status = review_status
        self.reviewed_by: Optional[str] = None
        self.review_comment: Optional[str] = None
        self.reviewed_at: Optional[datetime] = None


class MerchantRiskProfile:
    """商家风险档案"""
    def __init__(
        self,
        merchant_id: str,
        total_orders: int,
        total_amount: Decimal,
        refund_orders: int,
        refund_amount: Decimal,
        complaint_count: int,
        chargeback_count: int,
        chargeback_amount: Decimal,
        account_created_at: datetime,
        last_settlement_at: datetime,
        avg_order_value: Decimal,
        settlement_count_30d: int,
        fraud_flagged: bool = False,
        transaction_velocity_score: Decimal = Decimal("0"),
    ):
        self.merchant_id = merchant_id
        self.total_orders = total_orders
        self.total_amount = total_amount
        self.refund_orders = refund_orders
        self.refund_amount = refund_amount
        self.complaint_count = complaint_count
        self.chargeback_count = chargeback_count
        self.chargeback_amount = chargeback_amount
        self.account_created_at = account_created_at
        self.last_settlement_at = last_settlement_at
        self.avg_order_value = avg_order_value
        self.settlement_count_30d = settlement_count_30d
        self.fraud_flagged = fraud_flagged
        self.transaction_velocity_score = transaction_velocity_score


class SettlementRiskControlService:
    """商家结算风险控制服务

    通过多维度风险评估模型对商家进行风险打分，根据风险等级自动或人工
    执行风险控制措施（延迟结算、增加保证金、冻结账户等），并提供风险
    仪表盘和人工审核机制，平衡风险控制与商家体验。
    """

    # 风险因子权重配置
    FACTOR_WEIGHTS = {
        RiskFactorType.REFUND_RATE: Decimal("0.20"),
        RiskFactorType.COMPLAINT_RATE: Decimal("0.15"),
        RiskFactorType.TRANSACTION_ANOMALY: Decimal("0.20"),
        RiskFactorType.ACCOUNT_AGE: Decimal("0.10"),
        RiskFactorType.AVERAGE_ORDER_VALUE: Decimal("0.10"),
        RiskFactorType.SETTLEMENT_FREQUENCY: Decimal("0.10"),
        RiskFactorType.CHARGEBACK_RATE: Decimal("0.10"),
        RiskFactorType.FRAUD_HISTORY: Decimal("0.05"),
    }

    # 风险等级阈值
    RISK_THRESHOLDS = {
        RiskLevel.LOW: Decimal("30"),       # 0-30: 低风险
        RiskLevel.MEDIUM: Decimal("60"),    # 30-60: 中风险
        RiskLevel.HIGH: Decimal("80"),      # 60-80: 高风险
        RiskLevel.CRITICAL: Decimal("100"), # 80-100: 极高风险
    }

    # 退款率风险阈值
    REFUND_RATE_THRESHOLDS = {
        "normal": Decimal("0.05"),     # 5%以下正常
        "elevated": Decimal("0.10"),   # 5%-10%关注
        "high": Decimal("0.20"),       # 10%-20%高风险
        "critical": Decimal("0.30"),   # 20%以上极高风险
    }

    def __init__(self, db_session, config: Optional[Dict] = None):
        self.db = db_session
        self.config = config or {}
        self._lock = threading.Lock()
        self._decision_counter = 0
        self._auto_freeze_enabled = self.config.get("auto_freeze_enabled", True)
        self._review_required_for = [RiskMeasureType.FULL_FREEZE, RiskMeasureType.ACCOUNT_RESTRICTION]

    def assess_merchant_risk(
        self,
        merchant_id: str,
        profile: Optional[MerchantRiskProfile] = None,
    ) -> RiskAssessment:
        """评估商家风险

        综合退款率、投诉率、交易异常、账户年龄等多个风险因子，
        计算商家综合风险评分和风险等级。

        Args:
            merchant_id: 商家ID
            profile: 商家风险档案，如未提供则从数据库查询

        Returns:
            风险评估结果
        """
        if not profile:
            profile = self._load_merchant_profile(merchant_id)
            if not profile:
                raise ValueError(f"商家风险档案不存在: {merchant_id}")

        factor_scores: Dict[str, Decimal] = {}
        contributing_factors: List[str] = []

        # 1. 退款率评分（0-100，越高越危险）
        refund_rate = self._calculate_refund_rate(profile)
        refund_score = self._score_refund_rate(refund_rate)
        factor_scores["refund_rate"] = refund_score
        factor_scores["refund_rate_value"] = refund_rate
        if refund_rate > self.REFUND_RATE_THRESHOLDS["elevated"]:
            contributing_factors.append(f"退款率偏高: {refund_rate:.2%}")

        # 2. 投诉率评分
        complaint_rate = self._calculate_complaint_rate(profile)
        complaint_score = self._score_complaint_rate(complaint_rate)
        factor_scores["complaint_rate"] = complaint_score
        factor_scores["complaint_rate_value"] = complaint_rate
        if complaint_rate > Decimal("0.03"):
            contributing_factors.append(f"投诉率偏高: {complaint_rate:.2%}")

        # 3. 交易异常评分
        anomaly_score = self._score_transaction_anomaly(profile)
        factor_scores["transaction_anomaly"] = anomaly_score
        if anomaly_score > Decimal("50"):
            contributing_factors.append(f"交易异常评分偏高: {anomaly_score}")

        # 4. 账户年龄评分（新账户风险更高）
        account_age_score = self._score_account_age(profile)
        factor_scores["account_age"] = account_age_score
        if account_age_score > Decimal("50"):
            account_days = (datetime.now() - profile.account_created_at).days
            contributing_factors.append(f"账户注册时间短: {account_days}天")

        # 5. 平均订单金额评分
        aov_score = self._score_average_order_value(profile)
        factor_scores["average_order_value"] = aov_score
        if aov_score > Decimal("50"):
            contributing_factors.append(f"平均订单金额异常: {profile.avg_order_value}")

        # 6. 结算频率评分
        freq_score = self._score_settlement_frequency(profile)
        factor_scores["settlement_frequency"] = freq_score

        # 7. 拒付率评分
        chargeback_rate = self._calculate_chargeback_rate(profile)
        chargeback_score = self._score_chargeback_rate(chargeback_rate)
        factor_scores["chargeback_rate"] = chargeback_score
        factor_scores["chargeback_rate_value"] = chargeback_rate
        if chargeback_rate > Decimal("0.01"):
            contributing_factors.append(f"拒付率偏高: {chargeback_rate:.2%}")

        # 8. 欺诈历史评分
        fraud_score = Decimal("100") if profile.fraud_flagged else Decimal("0")
        factor_scores["fraud_history"] = fraud_score
        if profile.fraud_flagged:
            contributing_factors.append("存在欺诈标记")

        # 加权计算综合风险评分
        overall_score = Decimal("0")
        for factor_type, weight in self.FACTOR_WEIGHTS.items():
            factor_key = factor_type.value
            if factor_key in factor_scores:
                overall_score += factor_scores[factor_key] * weight

        overall_score = overall_score.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)

        # 确定风险等级
        risk_level = self._determine_risk_level(overall_score)

        # 生成建议
        recommendation = self._generate_recommendation(risk_level, contributing_factors)

        assessment = RiskAssessment(
            merchant_id=merchant_id,
            overall_score=overall_score,
            risk_level=risk_level,
            factor_scores=factor_scores,
            contributing_factors=contributing_factors,
            recommendation=recommendation,
        )

        # 持久化评估结果
        self._save_assessment(assessment)

        # 根据评估结果自动执行风险措施
        if risk_level in (RiskLevel.HIGH, RiskLevel.CRITICAL):
            self.apply_risk_measures(merchant_id, assessment, auto=True)

        logger.info(
            f"商家风险评估完成: merchant={merchant_id}, score={overall_score}, "
            f"level={risk_level.value}, factors={contributing_factors}"
        )

        return assessment

    def apply_risk_measures(
        self,
        merchant_id: str,
        assessment: RiskAssessment,
        auto: bool = False,
        override_measure: Optional[RiskMeasureType] = None,
    ) -> RiskDecision:
        """执行风险控制措施

        根据风险等级执行相应的控制措施：
        - 低风险：正常监控
        - 中风险：延迟结算（T+1改为T+3等）
        - 高风险：增加保证金（提高保证金比例）
        - 极高风险：冻结账户

        Args:
            merchant_id: 商家ID
            assessment: 风险评估结果
            auto: 是否为自动执行
            override_measure: 覆盖默认措施类型

        Returns:
            风险决策记录
        """
        # 确定措施类型
        if override_measure:
            measure_type = override_measure
        else:
            measure_type = self._get_default_measure(assessment.risk_level)

        # 构建措施详情
        measure_details = self._build_measure_details(
            merchant_id, assessment.risk_level, measure_type
        )

        # 生成决策ID
        with self._lock:
            self._decision_counter += 1
            decision_id = f"risk_{datetime.now().strftime('%Y%m%d')}_{self._decision_counter:06d}"

        # 计算生效时间
        effective_from = datetime.now()
        effective_until = self._calculate_effective_until(measure_type, assessment.risk_level)

        decision = RiskDecision(
            decision_id=decision_id,
            merchant_id=merchant_id,
            risk_level=assessment.risk_level,
            measure_type=measure_type,
            measure_details=measure_details,
            effective_from=effective_from,
            effective_until=effective_until,
            is_auto=auto,
            decided_by="system" if auto else "manual",
        )

        # 检查是否需要人工审核
        if measure_type in self._review_required_for and auto:
            decision.review_status = "pending"
            self._notify_review_required(decision)
            logger.warning(
                f"高风险措施需人工审核: merchant={merchant_id}, "
                f"measure={measure_type.value}, decision={decision_id}"
            )
        else:
            decision.review_status = "approved"
            # 执行措施
            self._execute_measure(decision)

        # 持久化决策
        self._save_decision(decision)

        logger.info(
            f"风险措施已执行: merchant={merchant_id}, "
            f"measure={measure_type.value}, auto={auto}, "
            f"decision={decision_id}, review={decision.review_status}"
        )

        return decision

    def review_risk_decision(
        self,
        decision_id: str,
        reviewer: str,
        action: str,  # "approve", "reject", "modify"
        comment: str = "",
        modified_measure: Optional[RiskMeasureType] = None,
        modified_details: Optional[Dict] = None,
    ) -> RiskDecision:
        """人工审核风险决策

        对自动触发的风险决策进行人工审核，可以批准、驳回或修改措施。

        Args:
            decision_id: 决策ID
            reviewer: 审核人
            action: 审核动作（approve/reject/modify）
            comment: 审核意见
            modified_measure: 修改后的措施类型（action=modify时使用）
            modified_details: 修改后的措施详情（action=modify时使用）

        Returns:
            更新后的决策记录
        """
        decision = self._load_decision(decision_id)
        if not decision:
            raise ValueError(f"决策记录不存在: {decision_id}")

        if decision.review_status == "approved" and decision.reviewed_by:
            raise ValueError(f"决策已被 {decision.reviewed_by} 审核，不可重复审核")

        decision.reviewed_by = reviewer
        decision.review_comment = comment
        decision.reviewed_at = datetime.now()

        if action == "approve":
            decision.review_status = "approved"
            self._execute_measure(decision)
            logger.info(
                f"风险决策审核通过: decision={decision_id}, "
                f"reviewer={reviewer}, merchant={decision.merchant_id}"
            )

        elif action == "reject":
            decision.review_status = "rejected"
            decision.status = "cancelled"
            # 如果是冻结措施被驳回，需要解除已执行的临时限制
            if decision.measure_type in (RiskMeasureType.FULL_FREEZE, RiskMeasureType.PARTIAL_FREEZE):
                self._lift_temporary_restriction(decision.merchant_id)
            logger.info(
                f"风险决策审核驳回: decision={decision_id}, "
                f"reviewer={reviewer}, merchant={decision.merchant_id}"
            )

        elif action == "modify":
            if not modified_measure:
                raise ValueError("修改审核必须提供 modified_measure")
            decision.measure_type = modified_measure
            if modified_details:
                decision.measure_details.update(modified_details)
            decision.review_status = "approved"
            self._execute_measure(decision)
            logger.info(
                f"风险决策审核修改: decision={decision_id}, "
                f"reviewer={reviewer}, new_measure={modified_measure.value}"
            )

        else:
            raise ValueError(f"不支持的审核动作: {action}")

        self._save_decision(decision)
        return decision

    def get_risk_dashboard(
        self,
        time_range: Optional[Tuple[datetime, datetime]] = None,
    ) -> Dict:
        """获取风险控制仪表盘数据

        汇总展示商家风险分布、待审核决策、措施执行情况等。

        Args:
            time_range: 时间范围，默认最近7天

        Returns:
            仪表盘数据字典
        """
        if not time_range:
            time_range = (
                datetime.now() - timedelta(days=7),
                datetime.now(),
            )

        start, end = time_range

        # 风险等级分布
        level_distribution = self._get_risk_level_distribution(start, end)

        # 待审核决策
        pending_reviews = self._get_pending_reviews()

        # 措施执行统计
        measure_statistics = self._get_measure_statistics(start, end)

        # 高风险商家清单
        high_risk_merchants = self._get_high_risk_merchants()

        # 风险趋势（按天统计）
        risk_trend = self._get_risk_trend(start, end)

        # 审核效率
        review_metrics = self._get_review_metrics(start, end)

        dashboard = {
            "time_range": {
                "start": start.isoformat(),
                "end": end.isoformat(),
            },
            "risk_level_distribution": level_distribution,
            "pending_reviews": {
                "count": len(pending_reviews),
                "items": pending_reviews[:20],  # 最多展示20条
            },
            "measure_statistics": measure_statistics,
            "high_risk_merchants": {
                "count": len(high_risk_merchants),
                "items": high_risk_merchants[:20],
            },
            "risk_trend": risk_trend,
            "review_metrics": review_metrics,
            "generated_at": datetime.now().isoformat(),
        }

        return dashboard

    # ---- 评分辅助方法 ----

    def _calculate_refund_rate(self, profile: MerchantRiskProfile) -> Decimal:
        """计算退款率"""
        if profile.total_orders == 0:
            return Decimal("0")
        return (Decimal(str(profile.refund_orders)) / Decimal(str(profile.total_orders))).quantize(
            Decimal("0.0001"), rounding=ROUND_HALF_UP
        )

    def _score_refund_rate(self, rate: Decimal) -> Decimal:
        """退款率评分（0-100）"""
        if rate <= self.REFUND_RATE_THRESHOLDS["normal"]:
            return Decimal("0")
        elif rate <= self.REFUND_RATE_THRESHOLDS["elevated"]:
            return ((rate - Decimal("0.05")) / Decimal("0.05") * Decimal("30")).quantize(
                Decimal("0.01")
            )
        elif rate <= self.REFUND_RATE_THRESHOLDS["high"]:
            return (Decimal("30") + (rate - Decimal("0.10")) / Decimal("0.10") * Decimal("40")).quantize(
                Decimal("0.01")
            )
        else:
            return min(
                Decimal("70") + (rate - Decimal("0.20")) / Decimal("0.10") * Decimal("30"),
                Decimal("100"),
            ).quantize(Decimal("0.01"))

    def _calculate_complaint_rate(self, profile: MerchantRiskProfile) -> Decimal:
        """计算投诉率"""
        if profile.total_orders == 0:
            return Decimal("0")
        return (Decimal(str(profile.complaint_count)) / Decimal(str(profile.total_orders))).quantize(
            Decimal("0.0001"), rounding=ROUND_HALF_UP
        )

    def _score_complaint_rate(self, rate: Decimal) -> Decimal:
        """投诉率评分"""
        if rate <= Decimal("0.01"):
            return Decimal("0")
        elif rate <= Decimal("0.03"):
            return (rate / Decimal("0.03") * Decimal("30")).quantize(Decimal("0.01"))
        elif rate <= Decimal("0.05"):
            return (Decimal("30") + (rate - Decimal("0.03")) / Decimal("0.02") * Decimal("40")).quantize(
                Decimal("0.01")
            )
        else:
            return min(
                Decimal("70") + (rate - Decimal("0.05")) / Decimal("0.05") * Decimal("30"),
                Decimal("100"),
            ).quantize(Decimal("0.01"))

    def _score_transaction_anomaly(self, profile: MerchantRiskProfile) -> Decimal:
        """交易异常评分"""
        score = Decimal("0")
        # 交易速度异常
        if profile.transaction_velocity_score > Decimal("80"):
            score += Decimal("40")
        elif profile.transaction_velocity_score > Decimal("50"):
            score += Decimal("20")
        # 订单金额波动
        if profile.avg_order_value > Decimal("10000"):
            score += Decimal("30")
        elif profile.avg_order_value > Decimal("5000"):
            score += Decimal("15")
        # 结算频率异常
        if profile.settlement_count_30d > 50:
            score += Decimal("30")
        elif profile.settlement_count_30d > 30:
            score += Decimal("15")
        return min(score, Decimal("100"))

    def _score_account_age(self, profile: MerchantRiskProfile) -> Decimal:
        """账户年龄评分（新账户风险更高）"""
        days = (datetime.now() - profile.account_created_at).days
        if days > 365:
            return Decimal("0")
        elif days > 180:
            return Decimal("20")
        elif days > 90:
            return Decimal("40")
        elif days > 30:
            return Decimal("60")
        else:
            return Decimal("90")

    def _score_average_order_value(self, profile: MerchantRiskProfile) -> Decimal:
        """平均订单金额评分"""
        aov = profile.avg_order_value
        if aov <= Decimal("500"):
            return Decimal("0")
        elif aov <= Decimal("2000"):
            return Decimal("10")
        elif aov <= Decimal("5000"):
            return Decimal("30")
        elif aov <= Decimal("10000"):
            return Decimal("50")
        else:
            return Decimal("80")

    def _score_settlement_frequency(self, profile: MerchantRiskProfile) -> Decimal:
        """结算频率评分"""
        freq = profile.settlement_count_30d
        if freq <= 5:
            return Decimal("0")
        elif freq <= 15:
            return Decimal("10")
        elif freq <= 30:
            return Decimal("25")
        elif freq <= 50:
            return Decimal("45")
        else:
            return Decimal("70")

    def _calculate_chargeback_rate(self, profile: MerchantRiskProfile) -> Decimal:
        """计算拒付率"""
        if profile.total_orders == 0:
            return Decimal("0")
        return (Decimal(str(profile.chargeback_count)) / Decimal(str(profile.total_orders))).quantize(
            Decimal("0.0001"), rounding=ROUND_HALF_UP
        )

    def _score_chargeback_rate(self, rate: Decimal) -> Decimal:
        """拒付率评分"""
        if rate <= Decimal("0.005"):
            return Decimal("0")
        elif rate <= Decimal("0.01"):
            return Decimal("30")
        elif rate <= Decimal("0.02"):
            return Decimal("60")
        else:
            return Decimal("100")

    def _determine_risk_level(self, score: Decimal) -> RiskLevel:
        """根据评分确定风险等级"""
        if score <= self.RISK_THRESHOLDS[RiskLevel.LOW]:
            return RiskLevel.LOW
        elif score <= self.RISK_THRESHOLDS[RiskLevel.MEDIUM]:
            return RiskLevel.MEDIUM
        elif score <= self.RISK_THRESHOLDS[RiskLevel.HIGH]:
            return RiskLevel.HIGH
        else:
            return RiskLevel.CRITICAL

    def _get_default_measure(self, risk_level: RiskLevel) -> RiskMeasureType:
        """获取风险等级对应的默认措施"""
        mapping = {
            RiskLevel.LOW: RiskMeasureType.NORMAL_MONITORING,
            RiskLevel.MEDIUM: RiskMeasureType.DELAYED_SETTLEMENT,
            RiskLevel.HIGH: RiskMeasureType.INCREASED_DEPOSIT,
            RiskLevel.CRITICAL: RiskMeasureType.FULL_FREEZE,
        }
        return mapping.get(risk_level, RiskMeasureType.NORMAL_MONITORING)

    def _build_measure_details(
        self, merchant_id: str, risk_level: RiskLevel, measure_type: RiskMeasureType,
    ) -> Dict:
        """构建措施详情"""
        if measure_type == RiskMeasureType.DELAYED_SETTLEMENT:
            delay_days = {RiskLevel.MEDIUM: 3, RiskLevel.HIGH: 7}.get(risk_level, 1)
            return {
                "original_settlement_cycle": "T+1",
                "new_settlement_cycle": f"T+{delay_days}",
                "delay_days": delay_days,
                "reason": f"风险等级{risk_level.value}，结算周期调整为T+{delay_days}",
            }
        elif measure_type == RiskMeasureType.INCREASED_DEPOSIT:
            deposit_increase = {RiskLevel.HIGH: Decimal("1.5"), RiskLevel.CRITICAL: Decimal("2.0")}.get(
                risk_level, Decimal("1.2")
            )
            return {
                "deposit_multiplier": str(deposit_increase),
                "reason": f"风险等级{risk_level.value}，保证金倍数调整为{deposit_increase}",
            }
        elif measure_type == RiskMeasureType.FULL_FREEZE:
            return {
                "freeze_scope": "all",
                "freeze_reason": f"风险等级{risk_level.value}，全额冻结账户",
                "requires_review": True,
            }
        elif measure_type == RiskMeasureType.PARTIAL_FREEZE:
            freeze_ratio = Decimal("0.5")
            return {
                "freeze_ratio": str(freeze_ratio),
                "reason": f"风险等级{risk_level.value}，冻结{freeze_ratio}结算金额",
            }
        return {}

    def _calculate_effective_until(
        self, measure_type: RiskMeasureType, risk_level: RiskLevel,
    ) -> Optional[datetime]:
        """计算措施结束时间"""
        if measure_type == RiskMeasureType.NORMAL_MONITORING:
            return None
        elif measure_type == RiskMeasureType.DELAYED_SETTLEMENT:
            return datetime.now() + timedelta(days=30)
        elif measure_type == RiskMeasureType.INCREASED_DEPOSIT:
            return datetime.now() + timedelta(days=60)
        elif measure_type == RiskMeasureType.FULL_FREEZE:
            return None  # 冻结需人工解冻
        return None

    def _generate_recommendation(
        self, risk_level: RiskLevel, factors: List[str],
    ) -> str:
        """生成风险控制建议"""
        if risk_level == RiskLevel.LOW:
            return "商家风险低，继续正常监控即可"
        elif risk_level == RiskLevel.MEDIUM:
            return f"商家存在中风险因素({', '.join(factors[:3])})，建议延迟结算并密切监控"
        elif risk_level == RiskLevel.HIGH:
            return f"商家存在高风险因素({', '.join(factors[:3])})，建议增加保证金并限制部分功能"
        else:
            return f"商家风险极高({', '.join(factors[:3])})，建议立即冻结账户并启动人工审核"

    def _execute_measure(self, decision: RiskDecision) -> None:
        """执行风险控制措施"""
        if decision.measure_type == RiskMeasureType.DELAYED_SETTLEMENT:
            self._apply_delayed_settlement(decision.merchant_id, decision.measure_details)
        elif decision.measure_type == RiskMeasureType.INCREASED_DEPOSIT:
            self._apply_increased_deposit(decision.merchant_id, decision.measure_details)
        elif decision.measure_type == RiskMeasureType.FULL_FREEZE:
            self._apply_full_freeze(decision.merchant_id, decision.measure_details)
        elif decision.measure_type == RiskMeasureType.PARTIAL_FREEZE:
            self._apply_partial_freeze(decision.merchant_id, decision.measure_details)

    def _apply_delayed_settlement(self, merchant_id: str, details: Dict) -> None:
        """应用延迟结算"""
        logger.info(f"应用延迟结算: merchant={merchant_id}, details={details}")

    def _apply_increased_deposit(self, merchant_id: str, details: Dict) -> None:
        """应用增加保证金"""
        logger.info(f"应用增加保证金: merchant={merchant_id}, details={details}")

    def _apply_full_freeze(self, merchant_id: str, details: Dict) -> None:
        """应用全额冻结"""
        logger.critical(f"应用全额冻结: merchant={merchant_id}, details={details}")

    def _apply_partial_freeze(self, merchant_id: str, details: Dict) -> None:
        """应用部分冻结"""
        logger.warning(f"应用部分冻结: merchant={merchant_id}, details={details}")

    def _lift_temporary_restriction(self, merchant_id: str) -> None:
        """解除临时限制"""
        logger.info(f"解除临时限制: merchant={merchant_id}")

    def _notify_review_required(self, decision: RiskDecision) -> None:
        """通知需要人工审核"""
        logger.warning(
            f"需要人工审核: decision={decision.decision_id}, "
            f"merchant={decision.merchant_id}, measure={decision.measure_type.value}"
        )

    def _load_merchant_profile(self, merchant_id: str) -> Optional[MerchantRiskProfile]:
        """加载商家风险档案"""
        return self.db.query(MerchantRiskProfile).filter_by(merchant_id=merchant_id).first()

    def _load_decision(self, decision_id: str) -> Optional[RiskDecision]:
        """加载决策记录"""
        return self.db.query(RiskDecision).filter_by(decision_id=decision_id).first()

    def _save_assessment(self, assessment: RiskAssessment) -> None:
        """持久化评估结果"""
        record = {
            "merchant_id": assessment.merchant_id,
            "overall_score": str(assessment.overall_score),
            "risk_level": assessment.risk_level.value,
            "factor_scores": {k: str(v) for k, v in assessment.factor_scores.items()},
            "contributing_factors": assessment.contributing_factors,
            "recommendation": assessment.recommendation,
            "assessed_at": assessment.assessed_at.isoformat(),
        }
        self.db.insert("risk_assessments", record)
        self.db.commit()

    def _save_decision(self, decision: RiskDecision) -> None:
        """持久化决策记录"""
        self.db.merge(decision)
        self.db.commit()

    def _get_risk_level_distribution(self, start: datetime, end: datetime) -> Dict:
        """获取风险等级分布"""
        assessments = self.db.query(RiskAssessment).filter(
            RiskAssessment.assessed_at >= start,
            RiskAssessment.assessed_at <= end,
        ).all()
        distribution = {level.value: 0 for level in RiskLevel}
        for a in assessments:
            distribution[a.risk_level.value] += 1
        return distribution

    def _get_pending_reviews(self) -> List[Dict]:
        """获取待审核决策"""
        decisions = self.db.query(RiskDecision).filter_by(review_status="pending").all()
        return [
            {
                "decision_id": d.decision_id,
                "merchant_id": d.merchant_id,
                "risk_level": d.risk_level.value,
                "measure_type": d.measure_type.value,
                "decided_by": d.decided_by,
            }
            for d in decisions
        ]

    def _get_measure_statistics(self, start: datetime, end: datetime) -> Dict:
        """获取措施执行统计"""
        decisions = self.db.query(RiskDecision).filter(
            RiskDecision.effective_from >= start,
            RiskDecision.effective_from <= end,
        ).all()
        stats = {m.value: {"auto": 0, "manual": 0} for m in RiskMeasureType}
        for d in decisions:
            key = "auto" if d.is_auto else "manual"
            if d.measure_type.value in stats:
                stats[d.measure_type.value][key] += 1
        return stats

    def _get_high_risk_merchants(self) -> List[Dict]:
        """获取高风险商家清单"""
        # 取最近评分最高的商家
        assessments = self.db.query(RiskAssessment).filter(
            RiskAssessment.risk_level.in_([RiskLevel.HIGH, RiskLevel.CRITICAL])
        ).order_by(RiskAssessment.overall_score.desc()).limit(20).all()
        return [
            {
                "merchant_id": a.merchant_id,
                "risk_score": str(a.overall_score),
                "risk_level": a.risk_level.value,
                "factors": a.contributing_factors,
            }
            for a in assessments
        ]

    def _get_risk_trend(self, start: datetime, end: datetime) -> List[Dict]:
        """获取风险趋势数据"""
        days = (end - start).days
        trend = []
        for i in range(days + 1):
            day = start + timedelta(days=i)
            day_end = day + timedelta(days=1)
            day_assessments = self.db.query(RiskAssessment).filter(
                RiskAssessment.assessed_at >= day,
                RiskAssessment.assessed_at < day_end,
            ).all()
            avg_score = (
                sum(a.overall_score for a in day_assessments) / len(day_assessments)
                if day_assessments else Decimal("0")
            )
            trend.append({
                "date": day.strftime("%Y-%m-%d"),
                "count": len(day_assessments),
                "avg_score": str(avg_score.quantize(Decimal("0.01"))),
            })
        return trend

    def _get_review_metrics(self, start: datetime, end: datetime) -> Dict:
        """获取审核效率指标"""
        decisions = self.db.query(RiskDecision).filter(
            RiskDecision.reviewed_at >= start,
            RiskDecision.reviewed_at <= end,
            RiskDecision.is_auto == True,
        ).all()
        approved = sum(1 for d in decisions if d.review_status == "approved")
        rejected = sum(1 for d in decisions if d.review_status == "rejected")
        total = len(decisions)
        avg_hours = Decimal("0")
        if total > 0:
            total_hours = sum(
                (d.reviewed_at - d.effective_from).total_seconds() / 3600
                for d in decisions if d.reviewed_at
            )
            avg_hours = Decimal(str(total_hours / total)).quantize(Decimal("0.1"))
        return {
            "total_auto_decisions": total,
            "approved": approved,
            "rejected": rejected,
            "approval_rate": str(Decimal(str(approved)) / Decimal(str(total)) * 100 if total else Decimal("0")) + "%",
            "avg_review_hours": str(avg_hours),
        }
```

## 异常场景补充

### 场景：风险误判导致商家冻结
```
触发条件: 风险控制系统将正常商家误判为高风险并自动冻结其账户，常见原因包括：(1)退款率计算未区分"商家原因退款"和"买家原因退款"，将买家主观退款计入商家风险指标；(2)新商家因为初始订单量少，一两笔退款导致退款率畸高（如2单退1单=50%退款率）；(3)促销活动期间订单量暴增触发了交易异常评分，被误判为刷单；(4)投诉率统计包含了已撤诉的投诉记录；(5)风险评分模型阈值设置过于敏感，正常波动也被判定为风险

检测方式: (1)每笔自动冻结决策必须在2小时内完成人工审核，审核通过率低于90%说明误判率过高；(2)对被冻结商家的历史数据进行回溯分析，如冻结前7天无任何异常交易记录则标记为疑似误判；(3)设置"误判率"监控指标：被审核驳回的自动决策占比，超过10%则触发告警；(4)定期对冻结商家进行满意度调研，收集误判案例
处理流程: (1)收到商家申诉或审核人员发现误判后，立即进入紧急解冻流程；(2)审核人员调取商家完整风险档案和评分明细，确认误判原因（退款分类错误/样本量不足/促销期间误判等）；(3)解除冻结：恢复账户正常状态，结算周期恢复为T+1，保证金恢复原比例；(4)补偿评估：计算商家因冻结产生的直接损失（未能及时结算的资金占用成本）和间接损失（订单流失、排名下降），制定补偿方案；(5)修正风险模型：根据误判原因调整评分逻辑——退款率只计入商家原因退款、新商家(订单<20)采用贝叶斯平滑、促销期间调整交易异常阈值；(6)回溯检查：用修正后的模型对近期所有自动冻结决策重新评分，发现其他可能的误判并主动解冻

预防措施: (1)退款率计算区分退款原因，只将"商家原因退款"（商品质量问题、发货延迟、虚假宣传等）计入风险指标，"买家原因退款"（不想要了、拍错了等）不计入；(2)新商家（订单量<50）使用贝叶斯平滑算法：adjusted_rate = (refund_count + prior) / (total_orders + 2*prior)，避免小样本极端值；(3)促销活动期间自动切换为宽松评分模式，交易异常阈值上调50%；(4)已撤诉的投诉不计入投诉率统计；(5)自动冻结决策必须经过人工审核才能最终执行，禁止全自动冻结（保留自动延迟结算和增加保证金）；(6)定期用历史数据回测风险模型，确保误判率低于5%
```

### 场景：高风险商家持续欺诈
```
触发条件: 高风险商家在受到风险控制措施后，通过变换手法规避风控持续进行欺诈，常见手法包括：(1)被冻结后使用关联账户继续交易，关联账户使用了不同的法人但实际控制人相同；(2)延迟结算期间故意降低退款率，通过小额真实交易稀释指标，但大额欺诈订单仍在持续；(3)利用平台不同业务线之间的风控数据隔离，在A业务线被风控后转到B业务线继续欺诈；(4)雇佣"刷手"制造虚假好评和低退款率的假象；(5)分拆大额订单为多笔小额订单，规避单笔金额异常检测

检测方式: (1)关联账户分析：通过设备指纹、IP地址、银行卡号、收货地址等维度识别关联账户，当关联账户中有被风控的商家时，新账户自动提升风险评分；(2)指标稀释检测：监控商家退款率的"分母增长率"和"分子增长率"，如果订单量暴增但投诉/退款绝对数不变，说明在刻意稀释指标；(3)跨业务线风控数据共享：实时同步各业务线的风控标记，某商家在任一业务线被风控则全局生效；(4)刷单特征识别：检测异常好评模式（集中在某时间段、评论文本相似度高、购买后极短时间内评价）；(5)分拆订单检测：同一商家对同一买家在短时间内产生多笔接近阈值的订单

处理流程: (1)确认欺诈行为后，立即执行全局冻结：关联账户、当前账户、所有业务线同时冻结；(2)冻结的同时保留所有电子证据（交易记录、聊天记录、物流信息、资金流向）；(3)启动资金追回流程：联系支付渠道拦截未到账资金，对已结算资金发起追回；(4)向反欺诈部门报告，视涉案金额决定是否报警；(5)将商家及相关信息加入平台黑名单和行业共享黑名单；(6)复盘风控漏洞：为何关联账户未被发现、指标稀释为何未被检测、跨业务线数据隔离如何被利用，修复相应漏洞

预防措施: (1)建立商家关联图谱，在开户环节就识别关联关系，关联账户共享风险评分；(2)监控指标稀释行为：当商家订单增长率>200%且同时退款率下降>30%时自动触发深度审查；(3)打破业务线风控数据孤岛，建设统一风控平台，风控标记全局实时生效；(4)对好评行为进行异常检测：集中时段评价、评论文本相似度、评价时间与确认收货时间间隔过短等特征均标记为可疑；(5)对同一买家同一商家的分拆订单进行合并计算，判断是否规避金额阈值；(6)定期更新欺诈手法知识库，将新发现的手法纳入检测规则
```

## 跨境商家结算完整实现

```python
from datetime import datetime, timedelta
from decimal import Decimal, ROUND_HALF_UP
from typing import Dict, List, Optional, Tuple
from enum import Enum
import hashlib
import json
import logging
import re
import threading

logger = logging.getLogger(__name__)


class MerchantVerificationStatus(Enum):
    """商家验证状态"""
    PENDING = "pending"
    KYC_SUBMITTED = "kyc_submitted"
    KYC_VERIFIED = "kyc_verified"
    TAX_VERIFIED = "tax_verified"
    FULLY_VERIFIED = "fully_verified"
    REJECTED = "rejected"
    SUSPENDED = "suspended"


class RemittanceStatus(Enum):
    """汇款状态"""
    INITIATED = "initiated"           # 已发起
    BANK_PROCESSING = "bank_processing"  # 银行处理中
    COMPLETED = "completed"            # 已完成
    RETURNED = "returned"             # 已退回
    FAILED = "failed"                  # 失败
    CANCELLED = "cancelled"            # 已取消
    UNDER_REVIEW = "under_review"      # 合规审查中


class ComplianceCheckResult(Enum):
    """合规检查结果"""
    PASSED = "passed"
    FLAGGED = "flagged"             # 标记待审
    BLOCKED = "blocked"             # 阻断
    ERROR = "error"                  # 检查异常


class CrossBorderMerchant:
    """跨境商家信息"""
    def __init__(
        self,
        merchant_id: str,
        company_name: str,
        country: str,
        registration_number: str,
        tax_id: str,
        bank_account: str,
        bank_swift_code: str,
        bank_country: str,
        beneficial_owners: List[Dict],
        kyc_documents: List[Dict],
        verification_status: MerchantVerificationStatus = MerchantVerificationStatus.PENDING,
        risk_rating: str = "medium",
    ):
        self.merchant_id = merchant_id
        self.company_name = company_name
        self.country = country
        self.registration_number = registration_number
        self.tax_id = tax_id
        self.bank_account = bank_account
        self.bank_swift_code = bank_swift_code
        self.bank_country = bank_country
        self.beneficial_owners = beneficial_owners
        self.kyc_documents = kyc_documents
        self.verification_status = verification_status
        self.risk_rating = risk_rating
        self.registered_at = datetime.now()
        self.last_verified_at: Optional[datetime] = None


class ExchangeRateQuote:
    """汇率报价"""
    def __init__(
        self,
        quote_id: str,
        from_currency: str,
        to_currency: str,
        rate: Decimal,
        fee_rate: Decimal,
        valid_until: datetime,
        provider: str,
    ):
        self.quote_id = quote_id
        self.from_currency = from_currency
        self.to_currency = to_currency
        self.rate = rate
        self.fee_rate = fee_rate
        self.valid_until = valid_until
        self.provider = provider


class RemittanceRecord:
    """汇款记录"""
    def __init__(
        self,
        remittance_id: str,
        merchant_id: str,
        amount: Decimal,
        from_currency: str,
        to_currency: str,
        exchange_rate: Decimal,
        converted_amount: Decimal,
        withholding_tax: Decimal,
        remittance_fee: Decimal,
        net_amount: Decimal,
        status: RemittanceStatus,
        bank_reference: str = "",
        created_at: Optional[datetime] = None,
    ):
        self.remittance_id = remittance_id
        self.merchant_id = merchant_id
        self.amount = amount
        self.from_currency = from_currency
        self.to_currency = to_currency
        self.exchange_rate = exchange_rate
        self.converted_amount = converted_amount
        self.withholding_tax = withholding_tax
        self.remittance_fee = remittance_fee
        self.net_amount = net_amount
        self.status = status
        self.bank_reference = bank_reference
        self.created_at = created_at or datetime.now()
        self.updated_at = datetime.now()
        self.status_history: List[Dict] = [
            {"status": status.value, "timestamp": datetime.now().isoformat()}
        ]


class CrossBorderSettlementService:
    """跨境商家结算服务

    处理跨境商家的注册审核（KYC/税务）、结算计算（汇率转换+预扣税+汇款手续费）、
    汇款提交与追踪、以及合规审查（制裁名单/反洗钱）等全流程。
    """

    # 各国预扣税率配置
    WITHHOLDING_TAX_RATES = {
        "US": Decimal("0.30"),    # 美国：30%
        "GB": Decimal("0.20"),    # 英国：20%
        "DE": Decimal("0.25"),    # 德国：25%（含团结附加税）
        "JP": Decimal("0.2042"),  # 日本：20.42%
        "SG": Decimal("0.00"),    # 新加坡：0%
        "HK": Decimal("0.00"),    # 香港：0%
        "AU": Decimal("0.30"),    # 澳大利亚：30%
        "KR": Decimal("0.22"),    # 韩国：22%（含地方税）
        "CA": Decimal("0.25"),    # 加拿大：25%
        "FR": Decimal("0.25"),    # 法国：25%
        "DEFAULT": Decimal("0.20"), # 默认：20%
    }

    # 汇款手续费配置
    REMITTANCE_FEE_CONFIG = {
        "base_fee": Decimal("15.00"),     # 基础手续费（USD）
        "percentage_fee": Decimal("0.001"),  # 比例手续费（0.1%）
        "min_fee": Decimal("15.00"),      # 最低手续费
        "max_fee": Decimal("200.00"),     # 最高手续费
        "urgent_surcharge": Decimal("25.00"),  # 加急费
    }

    # KYC必需文件类型
    REQUIRED_KYC_DOCUMENTS = [
        "business_registration",   # 营业执照
        "tax_certificate",        # 税务登记证
        "bank_statement",          # 银行对账单
        "id_document",             # 身份证明
        "address_proof",          # 地址证明
    ]

    def __init__(self, db_session, config: Optional[Dict] = None):
        self.db = db_session
        self.config = config or {}
        self._lock = threading.Lock()
        self._remittance_counter = 0
        self._sanctions_lists = self._load_sanctions_lists()
        self._exchange_rate_provider = self.config.get("exchange_rate_provider", "reuters")

    def register_overseas_merchant(
        self,
        company_name: str,
        country: str,
        registration_number: str,
        tax_id: str,
        bank_account: str,
        bank_swift_code: str,
        bank_country: str,
        beneficial_owners: List[Dict],
        kyc_documents: List[Dict],
    ) -> Dict:
        """注册海外商家

        执行KYC验证和税务ID验证，审核通过后创建商家账户。
        包括：企业信息验证、受益所有人识别、文件审核、税务合规检查。

        Args:
            company_name: 公司名称
            country: 注册国家代码
            registration_number: 注册号
            tax_id: 税务ID
            bank_account: 银行账户
            bank_swift_code: 银行SWIFT代码
            bank_country: 银行所在国家
            beneficial_owners: 受益所有人列表
            kyc_documents: KYC文件列表

        Returns:
            注册结果
        """
        # 1. 基础信息校验
        validation_errors = []

        if not company_name or len(company_name.strip()) < 2:
            validation_errors.append("公司名称无效")
        if not re.match(r"^[A-Z]{2}$", country):
            validation_errors.append(f"国家代码格式错误: {country}")
        if not registration_number:
            validation_errors.append("注册号不能为空")
        if not bank_swift_code or not re.match(r"^[A-Z]{4}[A-Z]{2}[A-Z0-9]{2}([A-Z0-9]{3})?$", bank_swift_code):
            validation_errors.append(f"SWIFT代码格式错误: {bank_swift_code}")

        # 2. 税务ID验证
        tax_validation = self._validate_tax_id(tax_id, country)
        if not tax_validation["valid"]:
            validation_errors.append(f"税务ID验证失败: {tax_validation['reason']}")

        # 3. KYC文件完整性检查
        doc_types = {doc["type"] for doc in kyc_documents}
        missing_docs = set(self.REQUIRED_KYC_DOCUMENTS) - doc_types
        if missing_docs:
            validation_errors.append(f"缺少必要KYC文件: {', '.join(missing_docs)}")

        # 检查文件有效期
        for doc in kyc_documents:
            if "expiry_date" in doc:
                expiry = datetime.fromisoformat(doc["expiry_date"])
                if expiry < datetime.now() + timedelta(days=30):
                    validation_errors.append(f"文件 {doc['type']} 即将过期或已过期")

        # 4. 受益所有人验证
        if not beneficial_owners or len(beneficial_owners) == 0:
            validation_errors.append("至少需要提供一名受益所有人")
        else:
            for owner in beneficial_owners:
                if "ownership_percentage" in owner:
                    pct = Decimal(str(owner["ownership_percentage"]))
                    if pct >= Decimal("25"):
                        # 超过25%持股的受益所有人需要额外验证
                        if "id_document" not in owner.get("documents", []):
                            validation_errors.append(
                                f"受益所有人 {owner.get('name', 'unknown')} 持股超过25%，需提供身份证明"
                            )

        if validation_errors:
            return {
                "status": "rejected",
                "errors": validation_errors,
                "message": "注册信息校验未通过",
            }

        # 5. 制裁名单检查
        sanctions_result = self._check_sanctions_list(
            company_name, country, beneficial_owners
        )
        if sanctions_result["hit"]:
            return {
                "status": "blocked",
                "reason": "命中制裁名单",
                "details": sanctions_result,
                "message": "该企业或受益所有人命中国际制裁名单，注册被阻止",
            }

        # 6. 创建商家记录
        merchant_id = self._generate_merchant_id(company_name, country)

        merchant = CrossBorderMerchant(
            merchant_id=merchant_id,
            company_name=company_name,
            country=country,
            registration_number=registration_number,
            tax_id=tax_id,
            bank_account=bank_account,
            bank_swift_code=bank_swift_code,
            bank_country=bank_country,
            beneficial_owners=beneficial_owners,
            kyc_documents=kyc_documents,
            verification_status=MerchantVerificationStatus.KYC_SUBMITTED,
        )

        # 7. 保存商家信息
        self._save_merchant(merchant)

        # 8. 发起KYC审核任务
        review_task_id = self._create_kyc_review_task(merchant)

        logger.info(
            f"海外商家注册提交: merchant={merchant_id}, "
            f"company={company_name}, country={country}, "
            f"review_task={review_task_id}"
        )

        return {
            "status": "submitted",
            "merchant_id": merchant_id,
            "company_name": company_name,
            "country": country,
            "verification_status": MerchantVerificationStatus.KYC_SUBMITTED.value,
            "review_task_id": review_task_id,
            "message": "注册信息已提交，等待KYC审核",
        }

    def calculate_cross_border_settlement(
        self,
        merchant_id: str,
        settlement_amount_cny: Decimal,
        target_currency: str = "USD",
        is_urgent: bool = False,
        tax_treaty_rate: Optional[Decimal] = None,
    ) -> Dict:
        """计算跨境结算金额

        将人民币结算金额转换为目标货币，并扣除预扣税和汇款手续费。
        支持税收协定优惠税率。

        Args:
            merchant_id: 商家ID
            settlement_amount_cny: 人民币结算金额
            target_currency: 目标货币代码
            is_urgent: 是否加急
            tax_treaty_rate: 税收协定优惠税率（如有）

        Returns:
            跨境结算明细
        """
        merchant = self._load_merchant(merchant_id)
        if not merchant:
            raise ValueError(f"商家不存在: {merchant_id}")

        if merchant.verification_status != MerchantVerificationStatus.FULLY_VERIFIED:
            raise ValueError(
                f"商家未通过完整验证，当前状态: {merchant.verification_status.value}"
            )

        if settlement_amount_cny <= Decimal("0"):
            raise ValueError(f"结算金额必须大于0, 当前: {settlement_amount_cny}")

        # 1. 获取汇率报价
        exchange_quote = self._get_exchange_rate("CNY", target_currency)
        if not exchange_quote:
            raise RuntimeError(f"无法获取 CNY->{target_currency} 汇率报价")

        # 检查报价有效期
        if exchange_quote.valid_until < datetime.now():
            raise RuntimeError("汇率报价已过期，请重新获取")

        # 2. 货币转换
        converted_amount = (settlement_amount_cny * exchange_quote.rate).quantize(
            Decimal("0.01"), rounding=ROUND_HALF_UP
        )

        # 3. 计算预扣税
        applicable_tax_rate = tax_treaty_rate or self._get_withholding_tax_rate(merchant.country)
        withholding_tax = (converted_amount * applicable_tax_rate).quantize(
            Decimal("0.01"), rounding=ROUND_HALF_UP
        )

        after_tax_amount = converted_amount - withholding_tax

        # 4. 计算汇款手续费
        base_fee = self.REMITTANCE_FEE_CONFIG["base_fee"]
        pct_fee = (after_tax_amount * self.REMITTANCE_FEE_CONFIG["percentage_fee"]).quantize(
            Decimal("0.01"), rounding=ROUND_HALF_UP
        )
        remittance_fee = max(base_fee, pct_fee)
        remittance_fee = min(remittance_fee, self.REMITTANCE_FEE_CONFIG["max_fee"])
        remittance_fee = max(remittance_fee, self.REMITTANCE_FEE_CONFIG["min_fee"])

        if is_urgent:
            remittance_fee += self.REMITTANCE_FEE_CONFIG["urgent_surcharge"]

        # 5. 计算商家实际到账金额
        net_amount = (after_tax_amount - remittance_fee).quantize(
            Decimal("0.01"), rounding=ROUND_HALF_UP
        )

        if net_amount < Decimal("0"):
            raise ValueError(
                f"结算金额不足以支付税费和手续费: "
                f"converted={converted_amount}, tax={withholding_tax}, fee={remittance_fee}"
            )

        # 6. 计算综合成本率
        total_cost_rate = (
            (withholding_tax + remittance_fee) / converted_amount * Decimal("100")
        ).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)

        result = {
            "merchant_id": merchant_id,
            "original_amount_cny": str(settlement_amount_cny),
            "exchange_rate": str(exchange_quote.rate),
            "exchange_rate_provider": exchange_quote.provider,
            "exchange_quote_id": exchange_quote.quote_id,
            "exchange_quote_valid_until": exchange_quote.valid_until.isoformat(),
            "converted_amount": str(converted_amount),
            "target_currency": target_currency,
            "withholding_tax_rate": str(applicable_tax_rate),
            "withholding_tax_rate_source": "tax_treaty" if tax_treaty_rate else "default",
            "withholding_tax": str(withholding_tax),
            "after_tax_amount": str(after_tax_amount),
            "remittance_fee": str(remittance_fee),
            "remittance_fee_breakdown": {
                "base_fee": str(base_fee),
                "percentage_fee": str(pct_fee),
                "urgent_surcharge": str(
                    self.REMITTANCE_FEE_CONFIG["urgent_surcharge"] if is_urgent else Decimal("0")
                ),
            },
            "net_amount": str(net_amount),
            "total_cost": str(withholding_tax + remittance_fee),
            "total_cost_rate": f"{total_cost_rate}%",
            "is_urgent": is_urgent,
            "calculated_at": datetime.now().isoformat(),
        }

        logger.info(
            f"跨境结算计算完成: merchant={merchant_id}, "
            f"CNY {settlement_amount_cny} -> {target_currency} {net_amount}, "
            f"tax={withholding_tax}, fee={remittance_fee}"
        )

        return result

    def submit_remittance(
        self,
        merchant_id: str,
        settlement_calculation: Dict,
        purpose_code: str = "TRADE_SETTLEMENT",
        reference_info: Optional[Dict] = None,
    ) -> Dict:
        """提交跨境汇款

        向合作银行提交跨境汇款指令，包含商家银行信息、汇款金额、
        汇款用途等信息。提交前执行最终合规检查。

        Args:
            merchant_id: 商家ID
            settlement_calculation: 结算计算结果
            purpose_code: 汇款用途代码
            reference_info: 参考信息

        Returns:
            汇款提交结果
        """
        merchant = self._load_merchant(merchant_id)
        if not merchant:
            raise ValueError(f"商家不存在: {merchant_id}")

        # 1. 最终合规检查
        compliance_result = self.handle_regulatory_compliance(
            merchant_id=merchant_id,
            amount=Decimal(str(settlement_calculation["net_amount"])),
            currency=settlement_calculation["target_currency"],
            purpose_code=purpose_code,
            beneficiary_info={
                "company_name": merchant.company_name,
                "country": merchant.country,
                "bank_account": merchant.bank_account,
                "swift_code": merchant.bank_swift_code,
            },
        )

        if compliance_result["status"] == ComplianceCheckResult.BLOCKED.value:
            return {
                "status": "blocked",
                "reason": "合规检查未通过",
                "compliance_details": compliance_result,
                "message": "汇款被合规审查阻断，请联系合规部门",
            }

        # 2. 检查汇率报价是否仍有效
        quote_valid_until = datetime.fromisoformat(
            settlement_calculation["exchange_quote_valid_until"]
        )
        if quote_valid_until < datetime.now():
            return {
                "status": "expired",
                "reason": "汇率报价已过期",
                "message": "请重新计算结算金额以获取最新汇率",
            }

        # 3. 生成汇款ID
        with self._lock:
            self._remittance_counter += 1
            remittance_id = (
                f"CBR{datetime.now().strftime('%Y%m%d%H%M%S')}"
                f"{self._remittance_counter:06d}"
            )

        # 4. 创建汇款记录
        remittance = RemittanceRecord(
            remittance_id=remittance_id,
            merchant_id=merchant_id,
            amount=Decimal(str(settlement_calculation["original_amount_cny"])),
            from_currency="CNY",
            to_currency=settlement_calculation["target_currency"],
            exchange_rate=Decimal(str(settlement_calculation["exchange_rate"])),
            converted_amount=Decimal(str(settlement_calculation["converted_amount"])),
            withholding_tax=Decimal(str(settlement_calculation["withholding_tax"])),
            remittance_fee=Decimal(str(settlement_calculation["remittance_fee"])),
            net_amount=Decimal(str(settlement_calculation["net_amount"])),
            status=RemittanceStatus.INITIATED,
        )

        # 5. 提交银行
        try:
            bank_response = self._submit_to_bank(
                remittance, merchant, purpose_code, reference_info
            )
            remittance.status = RemittanceStatus.BANK_PROCESSING
            remittance.bank_reference = bank_response.get("bank_reference", "")
            remittance.status_history.append({
                "status": RemittanceStatus.BANK_PROCESSING.value,
                "timestamp": datetime.now().isoformat(),
                "bank_reference": remittance.bank_reference,
            })
        except Exception as e:
            remittance.status = RemittanceStatus.FAILED
            remittance.status_history.append({
                "status": RemittanceStatus.FAILED.value,
                "timestamp": datetime.now().isoformat(),
                "error": str(e),
            })
            self._save_remittance(remittance)

            logger.error(f"跨境汇款提交失败: remittance={remittance_id}, error={e}")
            return {
                "status": "failed",
                "remittance_id": remittance_id,
                "reason": str(e),
                "message": "汇款提交银行失败，请稍后重试",
            }

        # 6. 保存汇款记录
        self._save_remittance(remittance)

        # 7. 合规审查标记处理
        if compliance_result["status"] == ComplianceCheckResult.FLAGGED.value:
            remittance.status = RemittanceStatus.UNDER_REVIEW
            self._flag_for_review(remittance, compliance_result)

        result = {
            "status": remittance.status.value,
            "remittance_id": remittance_id,
            "merchant_id": merchant_id,
            "bank_reference": remittance.bank_reference,
            "amount_cny": str(remittance.amount),
            "target_currency": remittance.to_currency,
            "net_amount": str(remittance.net_amount),
            "submitted_at": datetime.now().isoformat(),
        }

        logger.info(
            f"跨境汇款提交成功: remittance={remittance_id}, "
            f"merchant={merchant_id}, bank_ref={remittance.bank_reference}"
        )

        return result

    def track_remittance_status(
        self,
        remittance_id: str,
    ) -> Dict:
        """追踪跨境汇款状态

        查询汇款记录的当前状态，包括银行处理进度、
        预计到账时间等。

        Args:
            remittance_id: 汇款ID

        Returns:
            汇款状态详情
        """
        remittance = self._load_remittance(remittance_id)
        if not remittance:
            raise ValueError(f"汇款记录不存在: {remittance_id}")

        # 查询银行最新状态
        if remittance.status in (
            RemittanceStatus.BANK_PROCESSING,
            RemittanceStatus.INITIATED,
        ):
            bank_status = self._query_bank_status(remittance.bank_reference)
            if bank_status and bank_status.get("status") != remittance.status.value:
                old_status = remittance.status
                remittance.status = RemittanceStatus(bank_status["status"])
                remittance.updated_at = datetime.now()
                remittance.status_history.append({
                    "status": remittance.status.value,
                    "timestamp": datetime.now().isoformat(),
                    "bank_status_detail": bank_status.get("detail", ""),
                })
                self._save_remittance(remittance)

                # 如果汇款被退回，触发退回处理
                if remittance.status == RemittanceStatus.RETURNED:
                    self._handle_returned_remittance(remittance, bank_status)

        # 估算到账时间
        estimated_arrival = self._estimate_arrival_time(remittance)

        return {
            "remittance_id": remittance.remittance_id,
            "merchant_id": remittance.merchant_id,
            "status": remittance.status.value,
            "amount_cny": str(remittance.amount),
            "target_currency": remittance.to_currency,
            "net_amount": str(remittance.net_amount),
            "exchange_rate": str(remittance.exchange_rate),
            "bank_reference": remittance.bank_reference,
            "estimated_arrival": estimated_arrival,
            "status_history": remittance.status_history,
            "created_at": remittance.created_at.isoformat(),
            "updated_at": remittance.updated_at.isoformat(),
        }

    def handle_regulatory_compliance(
        self,
        merchant_id: str,
        amount: Decimal,
        currency: str,
        purpose_code: str,
        beneficiary_info: Dict,
    ) -> Dict:
        """处理合规审查

        执行制裁名单检查、反洗钱规则检查、金额阈值检查等合规审查。
        任何一项检查不通过都将阻断交易。

        Args:
            merchant_id: 商家ID
            amount: 交易金额
            currency: 货币代码
            purpose_code: 交易用途代码
            beneficiary_info: 收款方信息

        Returns:
            合规检查结果
        """
        checks_performed = []
        overall_status = ComplianceCheckResult.PASSED

        # 1. 制裁名单检查（OFAC、EU、UN制裁名单）
        sanctions_check = self._check_sanctions_for_transaction(
            beneficiary_info, amount, currency
        )
        checks_performed.append({
            "check_type": "sanctions_screening",
            "result": sanctions_check["result"],
            "details": sanctions_check.get("details", ""),
        })
        if sanctions_check["result"] == "blocked":
            overall_status = ComplianceCheckResult.BLOCKED

        # 2. 反洗钱（AML）检查
        aml_check = self._perform_aml_check(
            merchant_id, amount, currency, purpose_code
        )
        checks_performed.append({
            "check_type": "aml_screening",
            "result": aml_check["result"],
            "details": aml_check.get("details", ""),
            "risk_score": aml_check.get("risk_score", "0"),
        })
        if aml_check["result"] == "blocked":
            overall_status = ComplianceCheckResult.BLOCKED
        elif aml_check["result"] == "flagged" and overall_status != ComplianceCheckResult.BLOCKED:
            overall_status = ComplianceCheckResult.FLAGGED

        # 3. 高风险国家/地区检查
        country_risk_check = self._check_country_risk(beneficiary_info.get("country", ""))
        checks_performed.append({
            "check_type": "country_risk",
            "result": country_risk_check["result"],
            "country": beneficiary_info.get("country", ""),
            "risk_level": country_risk_check.get("risk_level", "unknown"),
        })
        if country_risk_check["result"] == "blocked":
            overall_status = ComplianceCheckResult.BLOCKED

        # 4. 交易金额阈值检查
        amount_threshold_check = self._check_amount_threshold(amount, currency)
        checks_performed.append({
            "check_type": "amount_threshold",
            "result": amount_threshold_check["result"],
            "amount": str(amount),
            "currency": currency,
        })
        if amount_threshold_check["result"] == "flagged" and overall_status == ComplianceCheckResult.PASSED:
            overall_status = ComplianceCheckResult.FLAGGED

        # 5. 交易频率检查（同一商家短时间内多次汇款）
        frequency_check = self._check_transaction_frequency(merchant_id)
        checks_performed.append({
            "check_type": "transaction_frequency",
            "result": frequency_check["result"],
            "recent_count": frequency_check.get("recent_count", 0),
        })
        if frequency_check["result"] == "flagged" and overall_status == ComplianceCheckResult.PASSED:
            overall_status = ComplianceCheckResult.FLAGGED

        result = {
            "merchant_id": merchant_id,
            "amount": str(amount),
            "currency": currency,
            "purpose_code": purpose_code,
            "status": overall_status.value,
            "checks_performed": checks_performed,
            "checked_at": datetime.now().isoformat(),
        }

        # 持久化合规检查结果
        self._save_compliance_check(result)

        logger.info(
            f"合规审查完成: merchant={merchant_id}, status={overall_status.value}, "
            f"checks={len(checks_performed)}"
        )

        return result

    # ---- 辅助方法 ----

    def _validate_tax_id(self, tax_id: str, country: str) -> Dict:
        """验证税务ID"""
        if not tax_id or len(tax_id.strip()) < 5:
            return {"valid": False, "reason": "税务ID格式无效"}

        # 国家特定的税务ID格式验证
        patterns = {
            "US": r"^\d{2}-\d{7}$",         # EIN
            "GB": r"^\d{9}$|^[A-Z]{2}\d{6}",  # UTR or VAT
            "DE": r"^\d{2}/\d{3}/\d{4}/\d{3}$", # Steuernummer
            "JP": r"^[A-Z]\d{9}$",           # 法人番号
            "SG": r"^\d{10}$|^[A-Z]{2}\d{8}[A-Z]$", # UEN
            "KR": r"^\d{3}-\d{2}-\d{5}$",    # 사업자등록번호
        }

        if country in patterns:
            if not re.match(patterns[country], tax_id):
                return {"valid": False, "reason": f"{country}税务ID格式不匹配"}

        return {"valid": True, "reason": "验证通过"}

    def _get_withholding_tax_rate(self, country: str) -> Decimal:
        """获取预扣税率"""
        return self.WITHHOLDING_TAX_RATES.get(country, self.WITHHOLDING_TAX_RATES["DEFAULT"])

    def _get_exchange_rate(self, from_currency: str, to_currency: str) -> Optional[ExchangeRateQuote]:
        """获取汇率报价"""
        # 模拟汇率获取
        rate = self._fetch_exchange_rate_from_provider(from_currency, to_currency)
        if rate:
            return ExchangeRateQuote(
                quote_id=f"QR_{datetime.now().strftime('%Y%m%d%H%M%S')}_{from_currency}_{to_currency}",
                from_currency=from_currency,
                to_currency=to_currency,
                rate=rate,
                fee_rate=Decimal("0.002"),
                valid_until=datetime.now() + timedelta(minutes=15),
                provider=self._exchange_rate_provider,
            )
        return None

    def _fetch_exchange_rate_from_provider(self, from_curr: str, to_curr: str) -> Optional[Decimal]:
        """从汇率提供商获取实时汇率"""
        # 实际实现调用外部API
        return Decimal("0.1380")  # 示例：CNY->USD

    def _check_sanctions_list(
        self, company_name: str, country: str, beneficial_owners: List[Dict],
    ) -> Dict:
        """检查制裁名单"""
        # 检查公司名
        for sanctions_list in self._sanctions_lists:
            if self._fuzzy_match(company_name, sanctions_list.get("name", "")):
                return {"hit": True, "list": sanctions_list["list_name"], "entity": company_name}

        # 检查受益所有人
        for owner in beneficial_owners:
            for sanctions_list in self._sanctions_lists:
                if self._fuzzy_match(owner.get("name", ""), sanctions_list.get("name", "")):
                    return {
                        "hit": True,
                        "list": sanctions_list["list_name"],
                        "entity": owner.get("name", ""),
                    }

        return {"hit": False}

    def _fuzzy_match(self, name1: str, name2: str) -> bool:
        """模糊匹配制裁名单"""
        # 简化实现，实际应使用更复杂的匹配算法
        return name1.lower().strip() == name2.lower().strip()

    def _load_sanctions_lists(self) -> List[Dict]:
        """加载制裁名单"""
        return []

    def _check_sanctions_for_transaction(
        self, beneficiary_info: Dict, amount: Decimal, currency: str,
    ) -> Dict:
        """交易级别制裁名单检查"""
        company = beneficiary_info.get("company_name", "")
        country = beneficiary_info.get("country", "")
        for sl in self._sanctions_lists:
            if self._fuzzy_match(company, sl.get("name", "")):
                return {"result": "blocked", "details": f"命中制裁名单: {sl['list_name']}"}
        return {"result": "passed"}

    def _perform_aml_check(
        self, merchant_id: str, amount: Decimal, currency: str, purpose_code: str,
    ) -> Dict:
        """反洗钱检查"""
        # 大额交易标记（超过等值1万美元）
        threshold = Decimal("10000")
        if amount >= threshold:
            return {
                "result": "flagged",
                "details": f"大额跨境交易: {amount} {currency}",
                "risk_score": "medium",
            }

        # 结构性交易检查（多笔小额交易规避报告阈值）
        recent_transactions = self._get_recent_transactions(merchant_id, days=7)
        total_recent = sum(Decimal(str(t["amount"])) for t in recent_transactions)
        if total_recent >= threshold * Decimal("0.9") and len(recent_transactions) >= 3:
            return {
                "result": "flagged",
                "details": "疑似结构性交易：多笔小额交易接近报告阈值",
                "risk_score": "high",
            }

        return {"result": "passed", "risk_score": "low"}

    def _check_country_risk(self, country: str) -> Dict:
        """检查国家/地区风险"""
        high_risk_countries = {"KP", "IR", "SY", "CU", "VE"}  # 示例
        if country in high_risk_countries:
            return {"result": "blocked", "risk_level": "high"}
        medium_risk_countries = {"RU", "BY", "MM", "AF"}  # 示例
        if country in medium_risk_countries:
            return {"result": "flagged", "risk_level": "medium"}
        return {"result": "passed", "risk_level": "low"}

    def _check_amount_threshold(self, amount: Decimal, currency: str) -> Dict:
        """检查金额阈值"""
        # 单笔超过5万美元需要额外审查
        if amount >= Decimal("50000"):
            return {"result": "flagged"}
        return {"result": "passed"}

    def _check_transaction_frequency(self, merchant_id: str) -> Dict:
        """检查交易频率"""
        recent = self._get_recent_transactions(merchant_id, days=1)
        if len(recent) >= 5:
            return {"result": "flagged", "recent_count": len(recent)}
        return {"result": "passed", "recent_count": len(recent)}

    def _get_recent_transactions(self, merchant_id: str, days: int = 7) -> List[Dict]:
        """获取近期交易记录"""
        return []

    def _submit_to_bank(
        self,
        remittance: RemittanceRecord,
        merchant: CrossBorderMerchant,
        purpose_code: str,
        reference_info: Optional[Dict],
    ) -> Dict:
        """提交银行汇款"""
        bank_ref = f"BNK{datetime.now().strftime('%Y%m%d%H%M%S')}{hashlib.md5(remittance.remittance_id.encode()).hexdigest()[:6]}"
        return {"bank_reference": bank_ref, "status": "accepted"}

    def _query_bank_status(self, bank_reference: str) -> Optional[Dict]:
        """查询银行处理状态"""
        return {"status": RemittanceStatus.BANK_PROCESSING.value, "detail": "银行处理中"}

    def _estimate_arrival_time(self, remittance: RemittanceRecord) -> str:
        """估算到账时间"""
        if remittance.status == RemittanceStatus.COMPLETED:
            return "已到账"
        elif remittance.status == RemittanceStatus.BANK_PROCESSING:
            # 跨境汇款通常1-3个工作日
            estimated = datetime.now() + timedelta(days=2)
            return f"预计 {estimated.strftime('%Y-%m-%d')} 到账"
        elif remittance.status == RemittanceStatus.UNDER_REVIEW:
            return "合规审查中，审查完成后预计1-3个工作日到账"
        elif remittance.status == RemittanceStatus.RETURNED:
            return "汇款已退回"
        elif remittance.status == RemittanceStatus.FAILED:
            return "汇款失败"
        return "未知"

    def _handle_returned_remittance(self, remittance: RemittanceRecord, bank_status: Dict) -> None:
        """处理退回的汇款"""
        logger.warning(
            f"跨境汇款被退回: remittance={remittance.remittance_id}, "
            f"merchant={remittance.merchant_id}, reason={bank_status.get('detail', 'unknown')}"
        )
        # 通知商家和运营
        self._notify_remittance_return(remittance, bank_status)

    def _notify_remittance_return(self, remittance: RemittanceRecord, bank_status: Dict) -> None:
        """通知汇款退回"""
        logger.info(
            f"汇款退回通知: remittance={remittance.remittance_id}, "
            f"bank_detail={bank_status.get('detail', '')}"
        )

    def _flag_for_review(self, remittance: RemittanceRecord, compliance_result: Dict) -> None:
        """标记汇款待合规审查"""
        logger.warning(
            f"汇款被标记待审查: remittance={remittance.remittance_id}, "
            f"reason={compliance_result.get('checks_performed', [])}"
        )

    def _generate_merchant_id(self, name: str, country: str) -> str:
        """生成商家ID"""
        raw = f"{name}_{country}_{datetime.now().strftime('%Y%m%d%H%M%S')}"
        return f"CB{hashlib.md5(raw.encode()).hexdigest()[:10].upper()}"

    def _load_merchant(self, merchant_id: str) -> Optional[CrossBorderMerchant]:
        """加载商家信息"""
        return self.db.query(CrossBorderMerchant).filter_by(merchant_id=merchant_id).first()

    def _save_merchant(self, merchant: CrossBorderMerchant) -> None:
        """保存商家信息"""
        self.db.merge(merchant)
        self.db.commit()

    def _load_remittance(self, remittance_id: str) -> Optional[RemittanceRecord]:
        """加载汇款记录"""
        return self.db.query(RemittanceRecord).filter_by(remittance_id=remittance_id).first()

    def _save_remittance(self, remittance: RemittanceRecord) -> None:
        """保存汇款记录"""
        self.db.merge(remittance)
        self.db.commit()

    def _create_kyc_review_task(self, merchant: CrossBorderMerchant) -> str:
        """创建KYC审核任务"""
        task_id = f"KYC_{datetime.now().strftime('%Y%m%d%H%M%S')}_{merchant.merchant_id}"
        self.db.insert("kyc_review_tasks", {
            "task_id": task_id,
            "merchant_id": merchant.merchant_id,
            "status": "pending",
            "created_at": datetime.now().isoformat(),
        })
        self.db.commit()
        return task_id

    def _save_compliance_check(self, result: Dict) -> None:
        """保存合规检查结果"""
        self.db.insert("compliance_checks", result)
        self.db.commit()
```

## 异常场景补充

### 场景：跨境汇款被银行退回
```
触发条件: 已提交的跨境汇款被收款银行或中间行退回，常见原因包括：(1)商家银行账户信息有误（账户号、SWIFT代码错误）；(2)收款银行因合规原因拒绝入账（收款人名称与账户名不匹配、涉及受制裁国家）；(3)中间行无法路由到目标银行（SWIFT代码对应的银行不存在或已关闭）；(4)汇款附注信息不符合收款国监管要求；（5）汇款金额超过收款账户的收汇额度

检测方式: (1)银行通过MT103/MT199报文返回退回通知，解析SWIFT报文中的退回原因代码；(2)汇款提交后超过5个工作日仍未到账，自动查询银行状态并标记为疑似退回；(3)每日对账比对已提交汇款总额与银行确认入账总额，差异部分追踪为疑似退回；(4)退回通知中包含退回原因代码，根据代码分类处理

处理流程: (1)收到银行退回通知后，更新汇款记录状态为RETURNED，记录退回原因代码和详细说明；(2)核算退回金额：退回金额 = 原汇款净额 - 银行扣收的退回手续费（通常10-50美元）；(3)将退回金额（扣除手续费后）退回商家结算账户余额，预扣税部分根据是否已缴纳分别处理：已缴纳则需申请退税，未缴纳则直接退回；(4)联系商家核实银行信息：如果是账户信息错误，要求商家提供正确的银行账户和SWIFT代码；(5)如果是合规原因退回，需合规部门介入：确认商家是否涉及受制裁实体、收款国家是否有特殊要求；(6)信息修正后重新提交汇款，但需先解决导致退回的根因问题；(7)对连续3次退回的商家暂停跨境汇款功能，要求其完成信息验证和合规审查后才能恢复

预防措施: (1)汇款提交前通过银行API预校验收款账户信息（IBAN校验、SWIFT代码验证、账户名匹配），校验通过才提交；(2)对商家银行信息变更建立严格的审核流程，变更后需先做小额测试汇款（1美元）验证账户可达后再恢复大额汇款；(3)建立收款国合规要求知识库，汇款前自动检查汇款附注、金额等是否符合收款国要求；(4)对高风险国家的汇款设置更长的审核周期，先做合规预审再提交银行；(5)汇款提交时在附注中填写完整、规范的交易描述，避免因附注不合规被退回
```

### 场景：反洗钱审查拦截合法交易
```
触发条件: AML（反洗钱）审查系统将合法的跨境交易错误标记为可疑交易并拦截，常见原因包括：(1)商家属于高频小额交易类型（如SaaS订阅服务），每月有大量小额结算，被误判为结构性交易；(2)商家业务涉及高风险国家但属于合法贸易（如出口到东南亚的制造业商家），被国家风险规则误判；(3)商家单笔大额结算（如季度结算、大客户订单付款），超过AML阈值被标记为大额可疑交易；(4)商家是新注册的海外商家，缺乏历史交易数据，AML模型对新实体默认给予较高风险评分；(5)商家收款人名称与KYC注册名称因翻译或缩写差异不完全匹配，触发名称匹配规则

检测方式: (1)监控AML拦截决策的人工审核驳回率，驳回率超过15%说明误拦截率过高；(2)对被拦截商家进行回溯分析：检查其历史交易是否有真实的洗钱迹象，如果没有则标记为误判；(3)按行业分类统计AML拦截率，如果某行业（如SaaS、制造业）的拦截率显著高于平均水平，说明该行业的合法交易模式被误判；(4)商家申诉渠道数据分析：统计因AML拦截导致的商家投诉量，投诉量增加则说明误拦截加剧；(5)定期对比被AML拦截的交易与实际发生洗钱的交易，计算假阳性率

处理流程: (1)商家通过申诉渠道提交交易说明和合规证明（如贸易合同、物流单据、发票等）；(2)合规审核人员48小时内完成审查：核实交易背景、检查商家历史合规记录、评估交易是否具有合理的商业目的；(3)如果确认交易合法，立即解除拦截并重新提交汇款，同时更新商家AML风险评分降低后续误拦截概率；(4)如果商家属于被系统性误判的行业，调整AML规则：为SaaS类商家设置结构性交易的豁免规则（允许高频小额）、为出口贸易商家设置高风险国家白名单、为新商家设置更合理的初始风险评分；(5)对误拦截造成的延迟进行补偿评估：商家因结算延迟产生的资金成本和业务损失；(6)将误拦截案例加入AML模型训练集，用于优化后续判断

预防措施: (1)AML规则按行业进行差异化配置，不同行业的交易模式不同，不能用同一套阈值判断所有行业；(2)结构性交易检测增加"时间密度"维度：SaaS月度订阅的规律性扣款与洗钱的结构性交易在时间分布上有本质区别；(3)建立高风险国家白名单机制：对于有合法贸易关系的商家，其涉及高风险国家的交易经核实后可加入白名单；(4)新商家的AML初始评分采用"低起点+快速适应"策略：初始评分不过高，根据实际交易表现快速调整；(5)名称匹配使用音译映射和别名库，解决中文->英文翻译差异导致的误匹配；(6)AML拦截后必须有人工审核环节，不能仅靠系统自动决定交易命运；(7)建立误拦截反馈闭环：审核驳回的案例定期用于优化AML规则和模型
```
