# C26: 供应链的库存同步与防超卖

## 业务场景

某大型零售集团，线下 500 家门店 + 线上电商 + 小程序，共享同一库存池。核心需求：无论用户从哪个渠道下单，库存都不能超卖。

**已知数据：**
- SKU 数量：10 万个
- 日均订单量：50 万
- 库存变更 TPS 峰值：2 万/秒（大促期间）
- 门店数：500 家
- 渠道：APP、小程序、天猫、京东、POS（门店收银）

**为什么比纯线上超卖更难？**

纯线上只需 Redis 扣减库存。但多渠道共享库存时：
- 天猫下单 → 线上扣减 → 但门店 POS 还不知道 → 门店继续卖 → 超卖
- 门店退货 → 库存回补 → 但线上不知道 → 显示"缺货"→ 丢失销售机会
- 各渠道库存扣减独立 → "总可用库存 = 总库存 - 各渠道已占用"如何始终正确？

## 核心挑战

### 挑战 1：多渠道库存一致性

4 个渠道同时卖同一商品，共享库存 100 件。天猫卖 30 + APP 卖 20 + 门店卖 40 = 90 件。天猫再卖 10 + 门店也卖 10 → 超卖。

### 挑战 2：冻结 vs 扣减

用户下单未付款 → 库存"冻结"（预占）30 分钟。冻结期间其他渠道可能扣减 → 释放时库存变负数。

**具体场景：** 库存 100 件，A 下单冻结 1 件（available=99, frozen=1）。B 下单扣减 99 件（available=0, frozen=1）。A 取消 → available+1=1。正确！但如果 B 不是冻结而是直接扣减（available-99），A 取消时 available+1=1，total 不变 → 库存多了 1 件。

### 挑战 3：门店离线

门店 POS 系统可能离线（网络故障），离线期间的销售数据无法实时同步 → 库存不准。

## 设计约束

- 多渠道共享库存，不允许超卖
- 下单后库存冻结 30 分钟，超时自动释放
- 门店离线销售数据上线后需同步
- 库存变更可追溯（审计）

## 请先独立思考（限时 35 分钟）

1. 库存模型：total = available + frozen + sold，三个字段如何协调？下单、付款、取消各如何操作？
2. Redis 原子扣减如何实现？Lua 脚本的逻辑是什么？
3. 多渠道配额如何分配？天猫大促期间如何防止占尽库存？
4. 门店离线销售如何同步？超卖如何处理？

---

## 设计解析

### 库存模型：total = available + frozen + sold

```
操作          available  frozen  sold  total
初始(100件)     100       0       0     100
A下单(冻结1)    99        1       0     100
B下单(冻结1)    98        2       0     100
A付款(确认)     98        1       1     99   ← total减少，frozen减少
B取消(释放)     99        0       1     99   ← available增加，frozen减少
```

**为什么需要 frozen？**

没有 frozen 概念时，下单直接 `stock -= 1`，取消直接 `stock += 1`：
- 重复取消（消息重试）→ stock += 1 多次 → stock > total → 超卖
- 无法区分"库存是被占用还是已售出" → 审计困难
- 无法知道"有多少库存被冻结但未付款" → 运营决策缺数据

有了 frozen：
- 取消幂等：检查 frozen 中是否有该订单的冻结记录，有才释放
- 可审计：available + frozen + sold = total 始终成立
- 运营可见：frozen 值 = 待付款订单占用的库存

### Redis 原子扣减：Lua 脚本

```python
class InventoryService:
    # Lua 脚本保证原子性：检查 + 扣减在一个 Redis 命令中完成
    FREEZE_LUA = """
    local total = tonumber(redis.call('GET', KEYS[1]))
    local frozen = tonumber(redis.call('GET', KEYS[2]))
    if total == nil or frozen == nil then
        return -1  -- 数据不存在
    end
    local available = total - frozen
    local qty = tonumber(ARGV[1])
    
    if available >= qty then
        redis.call('INCRBY', KEYS[2], qty)
        return 1   -- 冻结成功
    else
        return 0   -- 库存不足
    end
    """

    def freeze(self, sku_id, quantity, channel, order_id):
        """冻结库存（下单未付款）"""
        result = self.redis.eval(
            self.FREEZE_LUA, 2,
            f"inv:total:{sku_id}", f"inv:frozen:{sku_id}",
            quantity
        )

        if result == 1:
            # 记录冻结明细（30 分钟超时自动释放）
            self.redis.setex(
                f"inv:freeze:{order_id}", 1800,
                json.dumps({
                    "sku_id": sku_id,
                    "quantity": quantity,
                    "channel": channel,
                    "frozen_at": now().isoformat()
                })
            )
            # 审计日志
            self.audit_log("freeze", sku_id, quantity, channel, order_id)
            return True
        elif result == 0:
            return False  # 库存不足
        else:
            raise InventoryDataNotFound(sku_id)

    def confirm(self, sku_id, order_id):
        """确认扣减（付款成功）：frozen → sold"""
        data = self.redis.get(f"inv:freeze:{order_id}")
        if not data:
            raise FreezeNotFound(order_id)

        info = json.loads(data)
        
        # frozen 减少，total 减少（sold 隐含在 total 减少中）
        with self.redis.pipeline() as pipe:
            pipe.decrby(f"inv:total:{sku_id}", info["quantity"])
            pipe.decrby(f"inv:frozen:{sku_id}", info["quantity"])
            pipe.delete(f"inv:freeze:{order_id}")
            pipe.execute()

        # 异步同步到数据库（最终一致性）
        self.mq.produce("inventory_sync", {
            "sku_id": sku_id, "order_id": order_id,
            "quantity": info["quantity"], "action": "confirm"
        })

        self.audit_log("confirm", sku_id, info["quantity"], info["channel"], order_id)

    def release(self, sku_id, order_id):
        """释放冻结（超时未付款 / 手动取消）"""
        data = self.redis.get(f"inv:freeze:{order_id}")
        if not data:
            return  # 已过期或已确认（幂等）

        info = json.loads(data)
        
        # frozen 减少，available 增加（frozen → available）
        self.redis.decrby(f"inv:frozen:{sku_id}", info["quantity"])
        self.redis.delete(f"inv:freeze:{order_id}")

        self.audit_log("release", sku_id, info["quantity"], info["channel"], order_id)
```

**为什么用 Lua 脚本？**

不用 Lua 的风险：先 GET 再 SET 不是原子操作 → 并发下两个请求都看到 available=2 → 都冻结 1 → available 变成 0 而非 -1 → 看起来没超卖，但实际超卖了 1 件。

Lua 脚本在 Redis 中单线程执行 → 检查和扣减是原子的 → 不可能超卖。

### 多渠道配额控制

```python
class ChannelQuotaManager:
    """各渠道的库存配额 → 防止某渠道占尽库存"""

    def allocate(self, sku_id, total_available):
        """按渠道分配库存配额"""
        # 天猫 40%、APP 30%、小程序 20%、门店 10%
        quotas = {
            "tmall": int(total_available * 0.4),
            "app": int(total_available * 0.3),
            "miniprogram": int(total_available * 0.2),
            "store": int(total_available * 0.1),
        }
        # 共享池：各渠道配额用完后可借用
        shared = total_available - sum(quotas.values())

        for channel, quota in quotas.items():
            self.redis.set(f"inv:quota:{sku_id}:{channel}", quota)
        self.redis.set(f"inv:shared:{sku_id}", shared)

    def freeze_with_quota(self, sku_id, quantity, channel, order_id):
        """带配额控制的冻结"""
        # 先检查渠道配额
        channel_quota = int(self.redis.get(f"inv:quota:{sku_id}:{channel}") or 0)
        shared = int(self.redis.get(f"inv:shared:{sku_id}") or 0)

        if channel_quota >= quantity:
            # 渠道配额够 → 直接扣渠道配额
            self.redis.decrby(f"inv:quota:{sku_id}:{channel}", quantity)
            return self.freeze(sku_id, quantity, channel, order_id)
        elif channel_quota + shared >= quantity:
            # 渠道配额不够 → 借用共享池
            borrow = quantity - channel_quota
            self.redis.decrby(f"inv:quota:{sku_id}:{channel}", channel_quota)
            self.redis.decrby(f"inv:shared:{sku_id}", borrow)
            return self.freeze(sku_id, quantity, channel, order_id)
        else:
            # 所有配额都不够 → 库存不足
            return False
```

**配额释放：** 订单取消或超时时，配额也要归还：

```python
    def release_quota(self, sku_id, quantity, channel):
        """归还渠道配额"""
        # 先还到渠道配额（而非共享池）
        self.redis.incrby(f"inv:quota:{sku_id}:{channel}", quantity)
```

### 门店离线同步

```python
class StoreSyncService:
    def on_store_online(self, store_id):
        """门店重新上线 → 上报离线销售数据"""
        offline_sales = self.store_local_db.get_offline_sales(store_id)
        
        oversell_count = 0
        for sale in offline_sales:
            available = self.get_available(sale.sku_id)
            if available < sale.quantity:
                # 超卖！需要人工处理
                self.alert_oversell(store_id, sale.sku_id, 
                                    sale.quantity, available)
                oversell_count += 1
                continue
            
            # 补扣库存
            self.freeze(sale.sku_id, sale.quantity, "store", sale.order_id)
            self.confirm(sale.sku_id, sale.order_id)

        # 清除门店本地离线数据
        self.store_local_db.clear_offline_sales(store_id)
        
        if oversell_count > 0:
            self.alert_team(f"门店{store_id}有{oversell_count}笔超卖需处理")
```

### 冻结超时自动释放：双重保障

```python
class FreezeTimeoutCleaner:
    """定时扫描过期的冻结记录（兜底机制）"""

    def cleanup(self):
        """每分钟扫描一次"""
        # 方式1：Redis SETEX 自动过期后，冻结记录键消失
        # 但 frozen 值可能未减少 → 需要补偿
        
        # 方式2：从数据库获取已超时但未释放的冻结记录
        expired_orders = self.db.query("""
            SELECT order_id, sku_id, quantity 
            FROM order_freeze_log
            WHERE status = 'frozen' 
              AND frozen_at < NOW() - INTERVAL 30 MINUTE
        """)
        
        for order in expired_orders:
            # 检查 Redis 中是否还有冻结键
            if self.redis.exists(f"inv:freeze:{order.order_id}"):
                # 键还在（不应该）→ 手动释放
                self.release(order.sku_id, order.order_id)
            
            # 更新数据库状态
            self.db.update("order_freeze_log", 
                          {"status": "expired"}, 
                          {"order_id": order.order_id})
```

**双重保障：**
1. Redis SETEX 30 分钟自动过期 → 冻结键消失
2. 定时扫描数据库 → 兜底释放遗漏的冻结记录

### Redis 与数据库的对账

```python
class InventoryReconciler:
    """定期对账：Redis 与数据库的库存一致性"""

    def reconcile(self, sku_id):
        """单个 SKU 对账"""
        # Redis 中的库存
        redis_total = int(self.redis.get(f"inv:total:{sku_id}") or 0)
        redis_frozen = int(self.redis.get(f"inv:frozen:{sku_id}") or 0)
        redis_available = redis_total - redis_frozen

        # 数据库中的库存
        db_total = self.db.query_one(
            "SELECT total_stock FROM inventory WHERE sku_id = %s", sku_id
        )

        if redis_total != db_total:
            # 不一致 → 以数据库为准，修正 Redis
            self.redis.set(f"inv:total:{sku_id}", db_total)
            self.alert_team(f"库存不一致: sku={sku_id}, redis={redis_total}, db={db_total}")
```

## 常见陷阱（深度分析）

### 陷阱 1：各渠道独立库存池

**后果：** 渠道 A 积压 50 件库存，渠道 B 缺货 → 整体库存利用率低 → 销售损失。

**解决方案：** 共享库存 + 渠道配额 + 共享池借用。

### 陷阱 2：冻结无超时释放

**后果：** 30 分钟未付款的订单永远占着库存 → 其他渠道买不到 → 库存"假缺"。特别是恶意占库存攻击：大量下单不付款 → 库存被冻结 → 真实用户无法购买。

**解决方案：** Redis SETEX 30 分钟自动过期 + 定时扫描数据库兜底。

### 陷阱 3：不做配额控制

**后果：** 天猫大促占尽库存 → 门店无货可卖 → 线下客户流失。线上退货率高 → 实际库存浪费在冻结-释放循环中。

**解决方案：** 渠道配额 + 共享池。天猫最多占 40% 库存，其他渠道有保障。

### 陷阱 4：Redis 与数据库不一致

**后果：** Redis 库存=0 但数据库还有 → 用户看到"缺货"但实际有货 → 丢失销售机会。或反之 → 超卖。

**解决方案：** 定期对账（每小时）+ 异步同步（每次操作后 MQ 消息同步到 DB）。

## 延伸思考

- **库存预测**：基于历史销量预测各渠道需求，动态调整配额比例。如双 11 前自动提高天猫配额。
- **跨仓调拨**：A 仓缺货但 B 仓有货 → 自动调拨。调拨期间库存状态如何管理？需增加"调拨中"状态。
- **预售模式**：库存为 0 时仍可下单（预售），到货后优先发货。需增加"预售库存"维度：total = available + frozen + pre_sale + sold。