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

### 数据库设计

```sql
-- 库存主表
CREATE TABLE inventory (
    sku_id VARCHAR(64) PRIMARY KEY,
    total_stock INT NOT NULL,              -- 总库存
    available_stock INT NOT NULL DEFAULT 0, -- 可用库存
    frozen_stock INT NOT NULL DEFAULT 0,    -- 冻结库存
    sold_stock INT NOT NULL DEFAULT 0,      -- 已售库存
    version INT NOT NULL DEFAULT 0,         -- 乐观锁版本号
    updated_at TIMESTAMP DEFAULT NOW(),
    
    CONSTRAINT chk_inv CHECK (available_stock >= 0 AND frozen_stock >= 0 AND sold_stock >= 0)
);

-- 冻结明细表（审计+幂等）
CREATE TABLE inventory_freeze_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_id VARCHAR(64) NOT NULL,
    sku_id VARCHAR(64) NOT NULL,
    quantity INT NOT NULL,
    channel VARCHAR(20) NOT NULL,          -- tmall / app / miniprogram / store
    status VARCHAR(20) NOT NULL DEFAULT 'frozen', -- frozen / confirmed / released / expired
    frozen_at TIMESTAMP NOT NULL DEFAULT NOW(),
    released_at TIMESTAMP,
    
    UNIQUE KEY uk_order_sku (order_id, sku_id),  -- 幂等键
    INDEX idx_status_time (status, frozen_at),
    INDEX idx_sku (sku_id)
);

-- 渠道配额表
CREATE TABLE channel_quota (
    sku_id VARCHAR(64) NOT NULL,
    channel VARCHAR(20) NOT NULL,
    quota INT NOT NULL,                    -- 渠道配额
    used INT NOT NULL DEFAULT 0,           -- 已使用配额
    updated_at TIMESTAMP DEFAULT NOW(),
    
    PRIMARY KEY (sku_id, channel)
);

-- 库存变更审计日志
CREATE TABLE inventory_audit_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    sku_id VARCHAR(64) NOT NULL,
    action VARCHAR(20) NOT NULL,           -- freeze / confirm / release / adjust
    quantity INT NOT NULL,
    channel VARCHAR(20),
    order_id VARCHAR(64),
    before_available INT,
    before_frozen INT,
    after_available INT,
    after_frozen INT,
    operator VARCHAR(32),                  -- 操作人（system / admin_id）
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_sku_time (sku_id, created_at)
);

-- 门店离线销售记录
CREATE TABLE offline_sales (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    store_id VARCHAR(32) NOT NULL,
    order_id VARCHAR(64) NOT NULL,
    sku_id VARCHAR(64) NOT NULL,
    quantity INT NOT NULL,
    sold_at TIMESTAMP NOT NULL,
    synced_at TIMESTAMP,                   -- 同步时间（NULL=未同步）
    sync_status VARCHAR(20) DEFAULT 'pending', -- pending / synced / oversell
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_store_sync (store_id, sync_status)
);
```

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

### 分布式库存扣减：Saga 补偿事务

Redis Lua 解决了单节点原子扣减，但实际业务中一次下单可能涉及多个服务：库存服务扣减库存 + 订单服务创建订单 + 支付服务预授权 + 积分服务预扣积分。任何一个服务失败都需要回滚已执行的操作。

**Saga 模式：** 将长事务拆分为多个本地事务，每个本地事务有一个对应的补偿操作。如果某步失败，逆序执行已完成步骤的补偿操作。

```python
class InventorySagaOrchestrator:
    """库存扣减的 Saga 编排器：跨服务分布式事务"""

    # Saga 步骤定义
    SAGA_STEPS = [
        {
            "name": "freeze_inventory",
            "action": "freeze",
            "compensate": "release"
        },
        {
            "name": "create_order",
            "action": "create",
            "compensate": "cancel_order"
        },
        {
            "name": "payment_preauth",
            "action": "preauth",
            "compensate": "cancel_preauth"
        },
        {
            "name": "deduct_points",
            "action": "deduct",
            "compensate": "refund_points"
        },
    ]

    def execute_order_saga(self, order_request):
        """执行下单 Saga：全部成功或全部回滚"""
        saga_id = generate_id()
        completed_steps = []

        # 记录 Saga 开始
        self.db.insert("saga_log", {
            "saga_id": saga_id,
            "order_id": order_request.order_id,
            "status": "running",
            "started_at": now()
        })

        try:
            # 步骤 1：冻结库存
            result = self.inventory_service.freeze(
                order_request.sku_id, order_request.quantity,
                order_request.channel, order_request.order_id
            )
            if not result:
                raise InventoryInsufficient(order_request.sku_id)
            completed_steps.append("freeze_inventory")

            # 步骤 2：创建订单
            order = self.order_service.create(order_request)
            completed_steps.append("create_order")

            # 步骤 3：支付预授权
            preauth = self.payment_service.preauth(
                order_request.order_id, order_request.amount
            )
            if not preauth.success:
                raise PaymentPreauthFailed(preauth.reason)
            completed_steps.append("payment_preauth")

            # 步骤 4：扣减积分
            if order_request.use_points > 0:
                points_result = self.points_service.deduct(
                    order_request.user_id, order_request.use_points,
                    order_request.order_id
                )
                if not points_result:
                    raise PointsInsufficient()
                completed_steps.append("deduct_points")

            # 全部成功
            self.db.update("saga_log",
                {"status": "completed", "completed_at": now()},
                {"saga_id": saga_id})
            return OrderResult(success=True, order=order)

        except Exception as e:
            # 补偿：逆序回滚已完成的步骤
            self._compensate(saga_id, completed_steps, order_request)
            self.db.update("saga_log",
                {"status": "compensated", "error": str(e), "completed_at": now()},
                {"saga_id": saga_id})
            return OrderResult(success=False, error=str(e))

    def _compensate(self, saga_id, completed_steps, request):
        """逆序执行补偿操作"""
        for step_name in reversed(completed_steps):
            step = next(s for s in self.SAGA_STEPS if s["name"] == step_name)
            try:
                if step_name == "freeze_inventory":
                    self.inventory_service.release(
                        request.sku_id, request.order_id
                    )
                elif step_name == "create_order":
                    self.order_service.cancel(request.order_id)
                elif step_name == "payment_preauth":
                    self.payment_service.cancel_preauth(request.order_id)
                elif step_name == "deduct_points":
                    self.points_service.refund(
                        request.user_id, request.use_points, request.order_id
                    )

                # 记录补偿成功
                self.db.insert("saga_compensation_log", {
                    "saga_id": saga_id,
                    "step": step_name,
                    "action": step["compensate"],
                    "status": "success"
                })
            except Exception as comp_err:
                # 补偿失败 → 人工介入
                self.db.insert("saga_compensation_log", {
                    "saga_id": saga_id,
                    "step": step_name,
                    "action": step["compensate"],
                    "status": "failed",
                    "error": str(comp_err)
                })
                self.alert_team(
                    f"Saga 补偿失败: saga_id={saga_id}, "
                    f"step={step_name}, error={comp_err}"
                )
```

**Saga 的关键设计点：**

1. **每个补偿操作必须幂等**：`release` 检查冻结记录是否存在才释放，`cancel_order` 检查订单状态才取消，避免重复补偿。
2. **补偿失败需人工介入**：补偿操作本身也可能失败（如支付服务不可用），此时必须告警 + 人工处理。
3. **Saga 日志持久化**：所有步骤和补偿记录写入数据库，故障恢复后可继续执行未完成的补偿。
4. **不隔离**：Saga 没有隔离性，冻结的库存对其他事务可见。这是业务上可接受的——冻结本身就是一种"软隔离"。

**对比两阶段提交（2PC）：**

| 维度 | 2PC | Saga |
|------|-----|------|
| 隔离性 | 强隔离（锁） | 无隔离 |
| 性能 | 同步阻塞，锁持有时间长 | 异步非阻塞 |
| 可用性 | 协调者单点故障风险 | 各服务独立 |
| 补偿 | 自动回滚 | 需显式编写补偿 |
| 适用场景 | 单数据库跨表事务 | 跨服务分布式事务 |

库存场景选 Saga 而非 2PC 的原因：库存冻结本身就是短时间占用（30 分钟），Saga 的"无隔离"在业务上可接受——冻结期间其他请求看到库存减少，这正是期望行为。

### 多渠道配额控制（原子 Lua 实现）

前面独立的配额检查+冻结是非原子的。必须把配额检查和库存冻结放在同一个 Lua 脚本中执行。

```python
class ChannelQuotaManager:
    """各渠道的库存配额 → 防止某渠道占尽库存"""

    # 原子 Lua：配额检查 + 库存冻结在同一个脚本中
    FREEZE_WITH_QUOTA_LUA = """
    local sku_id = ARGV[1]
    local quantity = tonumber(ARGV[2])
    local channel = ARGV[3]
    local order_id = ARGV[4]

    -- 1. 检查渠道配额
    local channel_quota_key = "inv:quota:" .. sku_id .. ":" .. channel
    local shared_key = "inv:shared:" .. sku_id
    local channel_remaining = tonumber(redis.call('GET', channel_quota_key) or 0)
    local shared_remaining = tonumber(redis.call('GET', shared_key) or 0)

    if channel_remaining + shared_remaining < quantity then
        return 0  -- 配额不足
    end

    -- 2. 检查库存是否充足
    local total = tonumber(redis.call('GET', KEYS[1]))
    local frozen = tonumber(redis.call('GET', KEYS[2]))
    if total == nil or frozen == nil then
        return -1  -- 数据不存在
    end
    local available = total - frozen
    if available < quantity then
        return 0  -- 库存不足
    end

    -- 3. 扣减渠道配额（优先用渠道配额，不够借共享池）
    if channel_remaining >= quantity then
        redis.call('DECRBY', channel_quota_key, quantity)
    else
        local borrow = quantity - channel_remaining
        redis.call('SET', channel_quota_key, 0)
        redis.call('DECRBY', shared_key, borrow)
    end

    -- 4. 冻结库存
    redis.call('INCRBY', KEYS[2], quantity)

    -- 5. 记录冻结明细
    redis.call('SETEX', 'inv:freeze:' .. order_id, 1800,
        cjson.encode({sku_id=sku_id, quantity=quantity, channel=channel}))

    return 1  -- 成功
    """

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
        """带配额控制的冻结（原子操作）"""
        result = self.redis.eval(
            self.FREEZE_WITH_QUOTA_LUA, 2,
            f"inv:total:{sku_id}", f"inv:frozen:{sku_id}",
            sku_id, quantity, channel, order_id
        )

        if result == 1:
            self.audit_log("freeze", sku_id, quantity, channel, order_id)
            return True
        elif result == 0:
            return False  # 配额或库存不足
        else:
            raise InventoryDataNotFound(sku_id)

    def release_quota(self, sku_id, quantity, channel):
        """归还渠道配额"""
        # 先还到渠道配额（而非共享池）
        self.redis.incrby(f"inv:quota:{sku_id}:{channel}", quantity)
```

**原子化的关键：** `FREEZE_WITH_QUOTA_LUA` 把配额检查、配额扣减、库存冻结、冻结明细写入全部放在一个 Lua 脚本中。Redis 保证 Lua 脚本执行期间不会被其他命令打断，所以不会出现"配额检查通过了但库存被别人抢走"的问题。

### 门店离线同步

```python
class StoreSyncService:
    def on_store_online(self, store_id):
        """门店重新上线 → 上报离线销售数据"""
        offline_sales = self.store_local_db.get_offline_sales(store_id)

        oversell_count = 0
        synced_count = 0
        for sale in offline_sales:
            # 原子检查+冻结+确认
            result = self.inventory_service.freeze(
                sale.sku_id, sale.quantity, "store", sale.order_id
            )

            if not result:
                # 库存不足 → 超卖！
                self.alert_oversell(store_id, sale.sku_id, sale.quantity)
                # 标记为超卖记录
                self.db.update("offline_sales",
                    {"sync_status": "oversell"},
                    {"id": sale.id})
                oversell_count += 1
                continue

            # 冻结成功 → 立即确认
            self.inventory_service.confirm(sale.sku_id, sale.order_id)
            self.db.update("offline_sales",
                {"sync_status": "synced", "synced_at": now()},
                {"id": sale.id})
            synced_count += 1

        # 清除门店本地离线数据
        self.store_local_db.clear_offline_sales(store_id)

        if oversell_count > 0:
            self.alert_team(f"门店{store_id}有{oversell_count}笔超卖需处理")

        return {"synced": synced_count, "oversell": oversell_count}

    def handle_oversell(self, store_id, sku_id, quantity):
        """超卖处理方案"""
        # 方案1：紧急补货（如果有仓库库存）
        warehouse_stock = self.get_warehouse_stock(sku_id)
        if warehouse_stock >= quantity:
            self.transfer_from_warehouse(sku_id, quantity)
            return "restocked"

        # 方案2：等待其他渠道释放冻结库存
        pending_freezes = self.db.query(
            "SELECT order_id, quantity FROM inventory_freeze_log "
            "WHERE sku_id = %s AND status = 'frozen' "
            "ORDER BY frozen_at ASC", sku_id
        )
        if pending_freezes:
            # 有待释放的冻结 → 等待自动超时释放
            return "waiting_for_release"

        # 方案3：取消订单 + 赔偿
        self.cancel_order(store_id, sku_id)
        self.compensate_customer(store_id, sku_id, amount=50)  # 赔偿¥50
        return "cancelled_with_compensation"
```

**门店离线安全库存机制（预防超卖）：**

门店 POS 系统离线时应限制可售数量，而非无限制销售。每个门店维护一个"离线安全库存量"，离线时仅能销售不超过该数量的商品。

```python
class OfflineSafetyStockManager:
    """门店离线安全库存管理 → 预防离线超卖"""

    def calculate_safety_stock(self, store_id, sku_id):
        """计算门店离线安全库存量"""
        # 基于门店日均销量 × 离线预估时长 × 安全系数
        daily_sales = self.db.query_one(
            "SELECT avg_daily_sales FROM store_sku_stats "
            "WHERE store_id = %s AND sku_id = %s",
            store_id, sku_id
        )
        # 默认安全系数 1.5，离线预估 4 小时
        safety_factor = 1.5
        offline_hours = 4
        safety_stock = int(daily_sales * (offline_hours / 24) * safety_factor)
        return max(safety_stock, 5)  # 最低 5 件保底

    def update_local_stock_cache(self, store_id):
        """门店上线时下载本地库存缓存"""
        # 从中央库存系统获取该门店的安全库存量
        sku_list = self.get_store_sku_list(store_id)
        for sku_id in sku_list:
            safety_qty = self.calculate_safety_stock(store_id, sku_id)
            self.store_local_db.set(
                f"offline_limit:{sku_id}", safety_qty
            )

    def on_offline_sale(self, store_id, sku_id, quantity):
        """离线模式下销售 → 检查本地安全库存上限"""
        remaining = self.store_local_db.get(f"offline_limit:{sku_id}")

        if remaining is None or remaining < quantity:
            # 超过离线安全库存 → 拒绝销售
            return OfflineSaleResult(
                success=False,
                reason="offline_stock_limit",
                message=f"离线库存不足，仅剩{remaining}件"
            )

        # 扣减本地离线库存计数
        self.store_local_db.decr(f"offline_limit:{sku_id}", quantity)

        # 记录离线销售（上线后同步）
        self.store_local_db.insert("offline_sales", {
            "store_id": store_id,
            "sku_id": sku_id,
            "quantity": quantity,
            "sold_at": now(),
            "synced": False
        })

        return OfflineSaleResult(success=True)

    def on_store_going_offline(self, store_id):
        """门店即将离线（网络检测到不稳定）→ 切换到离线模式"""
        # 1. 下载安全库存到本地
        self.update_local_stock_cache(store_id)
        # 2. 通知中央库存系统冻结门店配额（防止线上继续分配给该门店）
        self.inventory_service.freeze_channel_quota(sku_id=None, channel="store")
        # 3. POS 界面切换到"离线模式"提示
        self.notify_store_ui(store_id, mode="offline")
```

**门店离线恢复流程完整编排：**

```
门店网络恢复 → 触发上线流程：
  1. POS 发送心跳 → 中央系统标记门店为 online
  2. POS 上报离线销售记录 → StoreSyncService.on_store_online()
  3. 逐笔处理：freeze → confirm / oversell → 补货 / 取消+赔偿
  4. 全量库存刷新：从中央下载最新库存数据到本地
  5. 重新激活门店配额：恢复门店渠道的库存配额分配
  6. 清除本地离线数据：删除所有 offline_sales 记录
  7. 生成对账报告：门店本地库存 vs 中央库存 → 差异告警
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

**超时释放的边界问题——frozen 值与 SETEX 不同步：**

Redis SETEX 过期只删除了 `inv:freeze:{order_id}` 键，但 `inv:frozen:{sku_id}` 的值并没有自动减少。这意味着如果只依赖 SETEX，会出现"frozen 值偏大，available 偏小"的假缺货。

```python
class FrozenValueCorrector:
    """修正 frozen 值：扫描所有活跃冻结键，求和 vs inv:frozen 值"""

    def correct_frozen_value(self, sku_id):
        """修正单个 SKU 的 frozen 值"""
        # 扫描所有该 SKU 的活跃冻结键
        actual_frozen = 0
        cursor = 0
        while True:
            cursor, keys = self.redis.scan(
                cursor, match=f"inv:freeze:*", count=500
            )
            for key in keys:
                data = self.redis.get(key)
                if data:
                    info = json.loads(data)
                    if info["sku_id"] == sku_id:
                        actual_frozen += info["quantity"]
            if cursor == 0:
                break

        # 对比 Redis 中的 frozen 值
        redis_frozen = int(self.redis.get(f"inv:frozen:{sku_id}") or 0)

        if actual_frozen != redis_frozen:
            diff = redis_frozen - actual_frozen
            # frozen 值偏大 → available 被低估 → 修正
            self.redis.set(f"inv:frozen:{sku_id}", actual_frozen)
            self.alert_team(
                f"frozen 值修正: sku={sku_id}, "
                f"修正前={redis_frozen}, 修正后={actual_frozen}, 差异={diff}"
            )
            self.audit_log("correct_frozen", sku_id, diff, "system", None)
```

**定时任务编排：**

```python
# Cron 调度配置
FREEZE_TIMEOUT_CRON = "*/1 * * * *"    # 每分钟：扫描超时冻结
FROZEN_CORRECT_CRON = "*/5 * * * *"    # 每5分钟：修正 frozen 值
FULL_RECONCILE_CRON = "0 * * * *"      # 每小时：全量对账 Redis vs DB
```

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

### 库存盘点与差异处理（Cycle Count）

对账仅比对 Redis 与数据库的数据一致性。但数据库本身的库存数据也可能与实际物理库存不一致（货物丢失、损坏、偷盗、录入错误）。库存盘点是发现并修正物理库存与系统库存差异的核心机制。

**盘点策略：**

| 策略 | 频率 | 范围 | 适用场景 |
|------|------|------|---------|
| 全量盘点 | 每季度 | 所有 SKU | 财务审计 |
| 循环盘点 | 每日 | ABC 分类轮转 | 日常运营 |
| 抽盘 | 每周 | 高差异率 SKU | 差异监控 |
| 事件触发盘点 | 实时 | 刚发生异常的 SKU | 超卖/负库存后 |

```sql
-- 盘点任务表
CREATE TABLE cycle_count_task (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    task_no VARCHAR(64) NOT NULL UNIQUE,
    warehouse_id VARCHAR(32) NOT NULL,
    sku_id VARCHAR(64) NOT NULL,
    system_qty INT NOT NULL,            -- 系统库存数量
    actual_qty INT,                      -- 实际盘点数量（NULL=未盘）
    difference INT,                      -- 差异 = actual - system
    difference_type VARCHAR(20),         -- surplus / shortage / match
    status VARCHAR(20) DEFAULT 'pending', -- pending / counting / counted / approved / adjusted
    count_by VARCHAR(32),               -- 盘点人
    approved_by VARCHAR(32),            -- 审批人
    created_at TIMESTAMP DEFAULT NOW(),
    counted_at TIMESTAMP,
    approved_at TIMESTAMP,

    INDEX idx_warehouse_status (warehouse_id, status),
    INDEX idx_sku (sku_id)
);

-- 盘点差异调整记录
CREATE TABLE stock_adjustment (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    adjustment_no VARCHAR(64) NOT NULL UNIQUE,
    sku_id VARCHAR(64) NOT NULL,
    warehouse_id VARCHAR(32) NOT NULL,
    before_qty INT NOT NULL,
    adjust_qty INT NOT NULL,            -- 正数=调增，负数=调减
    after_qty INT NOT NULL,
    reason VARCHAR(20) NOT NULL,         -- cycle_count / damage / theft / expiry / correction
    task_no VARCHAR(64),                 -- 关联盘点任务
    operator VARCHAR(32) NOT NULL,
    approved_by VARCHAR(32) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),

    INDEX idx_sku_time (sku_id, created_at)
);
```

```python
class CycleCountService:
    """库存盘点与差异处理"""

    # ABC 分类：A 类高价值 → 每日盘，B 类中价值 → 每周盘，C 类低价值 → 每月盘
    ABC_FREQUENCY = {"A": 1, "B": 7, "C": 30}  # 天

    def generate_daily_tasks(self, warehouse_id):
        """生成每日盘点任务（基于 ABC 分类轮转）"""
        today = now().date()
        day_of_year = today.timetuple().tm_yday

        # A 类：每天都盘
        a_skus = self.db.query(
            "SELECT sku_id FROM sku_abc_class WHERE class = 'A' AND warehouse_id = %s",
            warehouse_id
        )
        # B 类：每周一盘 1/7 的 SKU
        b_skus = self.db.query(
            "SELECT sku_id FROM sku_abc_class WHERE class = 'B' "
            "AND warehouse_id = %s AND MOD(sku_hash, 7) = MOD(%s, 7)",
            warehouse_id, day_of_year
        )
        # C 类：每月一盘 1/30 的 SKU
        c_skus = self.db.query(
            "SELECT sku_id FROM sku_abc_class WHERE class = 'C' "
            "AND warehouse_id = %s AND MOD(sku_hash, 30) = MOD(%s, 30)",
            warehouse_id, day_of_year
        )

        all_skus = a_skus + b_skus + c_skus
        tasks = []
        for sku in all_skus:
            system_qty = self.get_system_stock(warehouse_id, sku.sku_id)
            task_no = f"CC-{warehouse_id}-{sku.sku_id}-{today.strftime('%Y%m%d')}"
            tasks.append({
                "task_no": task_no,
                "warehouse_id": warehouse_id,
                "sku_id": sku.sku_id,
                "system_qty": system_qty,
                "status": "pending"
            })

        self.db.batch_insert("cycle_count_task", tasks)
        return len(tasks)

    def submit_count_result(self, task_no, actual_qty, count_by):
        """提交盘点结果"""
        task = self.db.query_one(
            "SELECT * FROM cycle_count_task WHERE task_no = %s", task_no
        )
        if not task:
            raise TaskNotFound(task_no)
        if task["status"] != "pending":
            raise TaskAlreadyProcessed(task_no)

        difference = actual_qty - task["system_qty"]
        if difference == 0:
            diff_type = "match"
        elif difference > 0:
            diff_type = "surplus"   # 盘盈
        else:
            diff_type = "shortage"  # 盘亏

        self.db.update("cycle_count_task", {
            "actual_qty": actual_qty,
            "difference": difference,
            "difference_type": diff_type,
            "status": "counted",
            "counted_by": count_by,
            "counted_at": now()
        }, {"task_no": task_no})

        # 差异超过阈值 → 自动告警
        threshold = self.get_difference_threshold(task["sku_id"])
        if abs(difference) > threshold:
            self.alert_team(
                f"盘点差异超阈值: task={task_no}, sku={task['sku_id']}, "
                f"系统={task['system_qty']}, 实际={actual_qty}, "
                f"差异={difference}, 阈值={threshold}"
            )

        return {"difference": difference, "type": diff_type}

    def approve_and_adjust(self, task_no, approved_by):
        """审批盘点差异并调整库存"""
        task = self.db.query_one(
            "SELECT * FROM cycle_count_task WHERE task_no = %s AND status = 'counted'",
            task_no
        )
        if not task:
            raise TaskNotFound(task_no)

        if task["difference"] == 0:
            # 无差异 → 直接完成
            self.db.update("cycle_count_task",
                {"status": "approved", "approved_by": approved_by, "approved_at": now()},
                {"task_no": task_no})
            return

        # 执行库存调整（先冻结差异量，审批后调整）
        with self.db.transaction():
            # 调整数据库库存
            before_qty = task["system_qty"]
            adjust_qty = task["difference"]  # 正数=盘盈调增，负数=盘亏调减
            after_qty = before_qty + adjust_qty

            self.db.execute(
                "UPDATE inventory SET total_stock = total_stock + %s, "
                "available_stock = available_stock + %s, "
                "version = version + 1 "
                "WHERE sku_id = %s AND version = %s",
                adjust_qty, adjust_qty, task["sku_id"], task.get("version", 0)
            )

            # 同步更新 Redis
            self.redis.incrby(f"inv:total:{task['sku_id']}", adjust_qty)

            # 记录调整日志
            self.db.insert("stock_adjustment", {
                "adjustment_no": f"ADJ-{task_no}",
                "sku_id": task["sku_id"],
                "warehouse_id": task["warehouse_id"],
                "before_qty": before_qty,
                "adjust_qty": adjust_qty,
                "after_qty": after_qty,
                "reason": "cycle_count",
                "task_no": task_no,
                "operator": approved_by,
                "approved_by": approved_by
            })

            # 更新盘点任务状态
            self.db.update("cycle_count_task",
                {"status": "adjusted", "approved_by": approved_by, "approved_at": now()},
                {"task_no": task_no})

        self.audit_log("cycle_count_adjust", task["sku_id"],
                       adjust_qty, "system", task_no)
```

**盘点差异的根因分析：**

```python
class DiscrepancyAnalyzer:
    """盘点差异根因分析"""

    def analyze(self, sku_id, warehouse_id, difference):
        """分析差异可能原因并给出建议"""
        causes = []

        # 检查1：是否有未同步的入库单
        pending_inbound = self.db.query_one(
            "SELECT COUNT(*) as cnt FROM inbound_order "
            "WHERE sku_id = %s AND warehouse_id = %s AND status = 'receiving'",
            sku_id, warehouse_id
        )
        if pending_inbound["cnt"] > 0:
            causes.append({
                "cause": "pending_inbound",
                "description": f"有{pending_inbound['cnt']}张入库单未完成确认",
                "action": "确认入库单后重新盘点"
            })

        # 检查2：是否有未同步的出库单
        pending_outbound = self.db.query_one(
            "SELECT COUNT(*) as cnt FROM outbound_order "
            "WHERE sku_id = %s AND warehouse_id = %s AND status = 'picking'",
            sku_id, warehouse_id
        )
        if pending_outbound["cnt"] > 0:
            causes.append({
                "cause": "pending_outbound",
                "description": f"有{pending_outbound['cnt']}张出库单正在拣货（实物已移但系统未扣减）",
                "action": "完成出库单后重新盘点"
            })

        # 检查3：近期的库存调整记录
        recent_adjustments = self.db.query(
            "SELECT * FROM stock_adjustment "
            "WHERE sku_id = %s AND created_at > NOW() - INTERVAL 7 DAY "
            "ORDER BY created_at DESC LIMIT 10",
            sku_id
        )
        if len(recent_adjustments) >= 3:
            causes.append({
                "cause": "frequent_adjustments",
                "description": f"近7天有{len(recent_adjustments)}次库存调整，可能存在系统性问题",
                "action": "检查入库/出库流程是否有操作错误"
            })

        # 检查4：门店退货未入库
        pending_returns = self.db.query_one(
            "SELECT SUM(quantity) as total FROM store_return "
            "WHERE sku_id = %s AND warehouse_id = %s AND status = 'in_transit'",
            sku_id, warehouse_id
        )
        if pending_returns["total"] and pending_returns["total"] > 0:
            causes.append({
                "cause": "pending_returns",
                "description": f"有{pending_returns['total']}件退货在途中未入库",
                "action": "等待退货入库后重新盘点"
            })

        return causes
```

## 常见陷阱（深度分析）

### 陷阱 1：各渠道独立库存池

**后果：** 渠道 A 积压 50 件库存，渠道 B 缺货 → 整体库存利用率低 → 销售损失。

**具体场景：** 天猫分配了 100 件，APP 分配了 100 件。天猫卖得慢（日均 10 件），APP 卖得快（日均 50 件）。天猫积压 90 件的同时 APP 缺货。

**解决方案：** 共享库存 + 渠道配额 + 共享池借用。天猫最多占 40% 库存，其他渠道有保障。配额用完后从共享池借用，确保库存不被单一渠道锁死。

### 陷阱 2：冻结无超时释放

**后果：** 30 分钟未付款的订单永远占着库存 → 其他渠道买不到 → 库存"假缺"。

**恶意占库存攻击场景：** 黄牛用 1000 个账号同时下单不付款 → 1000 件库存全部冻结 → 真实用户看到"缺货" → 黄牛在二手平台高价转卖。

**解决方案：**
- Redis SETEX 30 分钟自动过期 + 定时扫描数据库兜底
- 风控：同一用户 10 分钟内下单超过 3 次不付款 → 限制下单 1 小时
- 验证码：首次下单正常，连续下单需验证码

### 陷阱 3：不做配额控制

**后果：** 天猫大促占尽库存 → 门店无货可卖 → 线下客户流失。线上退货率高 → 实际库存浪费在冻结-释放循环中。

**量化分析：** 假设天猫大促日占比 60% 流量，无配额控制时天猫可能占 90%+ 库存。500 家门店日损失约 20% 客流 → 日损失约 ¥10 万。

**解决方案：** 渠道配额 + 共享池。天猫最多占 40% 库存，其他渠道有保障。大促前可动态调整配额。

### 陷阱 4：Redis 与数据库不一致

**后果：** Redis 库存=0 但数据库还有 → 用户看到"缺货"但实际有货 → 丢失销售机会。或反之 → 超卖。

**不一致场景分析：**

| 场景 | Redis 状态 | MySQL 状态 | 原因 |
|------|-----------|-----------|------|
| 正常 | total=90 | total=90 | 同步正常 |
| MQ消息延迟 | total=90 | total=100 | confirm 消息还在 MQ 中 |
| Redis 故障恢复 | total=100 | total=90 | Redis 从 RDB 恢复到旧数据 |
| 并发扣减竞态 | total=98 | total=99 | MQ 消息乱序 |

**解决方案：** 定期对账（每小时）+ 异步同步（每次操作后 MQ 消息同步到 DB）+ Redis 恢复后全量从 DB 加载。

### 陷阱 5：门店离线期间大量超卖

**场景：** 门店网络故障 4 小时，期间卖了 200 件某 SKU，但线上只剩 50 件 → 超卖 150 件。

**处理方案：**
1. 优先从仓库紧急调拨补货
2. 等待其他渠道的冻结库存超时释放
3. 无法补货 → 逐笔取消门店订单 + 赔偿
4. 预防：门店 POS 应有本地库存限制（离线时只允许卖本地安全库存量）

### 跨仓调拨实现

多仓库场景中，A 仓缺货但 B 仓有货时需要自动调拨。调拨是库存系统中最复杂的事务之一：涉及两个仓库的库存增减、运输状态跟踪、异常回滚。

```sql
-- 仓库库存表（每个仓库独立库存）
CREATE TABLE warehouse_inventory (
    warehouse_id VARCHAR(32) NOT NULL,
    sku_id VARCHAR(64) NOT NULL,
    total_stock INT NOT NULL DEFAULT 0,
    available_stock INT NOT NULL DEFAULT 0,
    frozen_stock INT NOT NULL DEFAULT 0,
    transferring_out INT NOT NULL DEFAULT 0,   -- 调出中
    transferring_in INT NOT NULL DEFAULT 0,     -- 调入中
    version INT NOT NULL DEFAULT 0,
    updated_at TIMESTAMP DEFAULT NOW(),

    PRIMARY KEY (warehouse_id, sku_id),
    CONSTRAINT chk_wh_inv CHECK (available_stock >= 0)
);

-- 调拨单
CREATE TABLE transfer_order (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    transfer_no VARCHAR(64) NOT NULL UNIQUE,
    from_warehouse VARCHAR(32) NOT NULL,
    to_warehouse VARCHAR(32) NOT NULL,
    sku_id VARCHAR(64) NOT NULL,
    quantity INT NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'created',
    -- created / confirmed / in_transit / received / completed / cancelled / returned
    created_by VARCHAR(32) NOT NULL,
    confirmed_at TIMESTAMP,
    shipped_at TIMESTAMP,
    received_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),

    INDEX idx_status (status),
    INDEX idx_warehouse (from_warehouse, to_warehouse)
);
```

```python
class WarehouseTransferService:
    """跨仓调拨服务：两阶段调拨（冻结→出库→运输→入库）"""

    TRANSFER_STATUS_FLOW = {
        "created": "confirmed",
        "confirmed": "in_transit",
        "in_transit": "received",
        "received": "completed"
    }

    def create_transfer(self, from_wh, to_wh, sku_id, quantity, created_by):
        """创建调拨单并冻结源仓库库存"""
        # Step 1: 冻结源仓库库存（原子操作）
        lua_script = """
        local key = KEYS[1]
        local qty = tonumber(ARGV[1])
        local available = tonumber(redis.call('HGET', key, 'available'))
        if available == nil or available < qty then
            return 0
        end
        redis.call('HINCRBY', key, 'available', -qty)
        redis.call('HINCRBY', key, 'transferring_out', qty)
        return 1
        """
        result = self.redis.eval(
            lua_script, 1,
            f"wh:inv:{from_wh}:{sku_id}",
            quantity
        )
        if result == 0:
            raise InsufficientStock(from_wh, sku_id)

        # Step 2: 写入调拨单
        transfer_no = f"TF-{from_wh}-{to_wh}-{sku_id}-{generate_id()}"
        self.db.insert("transfer_order", {
            "transfer_no": transfer_no,
            "from_warehouse": from_wh,
            "to_warehouse": to_wh,
            "sku_id": sku_id,
            "quantity": quantity,
            "status": "created",
            "created_by": created_by
        })

        # Step 3: 目标仓库记录调入中
        self.redis.hincrby(
            f"wh:inv:{to_wh}:{sku_id}", "transferring_in", quantity
        )

        return transfer_no

    def confirm_and_ship(self, transfer_no):
        """确认调拨并出库（源仓库库存减少）"""
        order = self.db.query_one(
            "SELECT * FROM transfer_order WHERE transfer_no = %s AND status = 'created'",
            transfer_no
        )
        if not order:
            raise TransferOrderNotFound(transfer_no)

        from_wh = order["from_warehouse"]
        sku_id = order["sku_id"]
        quantity = order["quantity"]

        # 源仓库：transferring_out → total 减少
        self.redis.hincrby(f"wh:inv:{from_wh}:{sku_id}", "transferring_out", -quantity)
        self.redis.hincrby(f"wh:inv:{from_wh}:{sku_id}", "total_stock", -quantity)

        # 同步数据库
        self.db.execute(
            "UPDATE warehouse_inventory SET "
            "transferring_out = transferring_out - %s, "
            "total_stock = total_stock - %s, "
            "version = version + 1 "
            "WHERE warehouse_id = %s AND sku_id = %s",
            quantity, quantity, from_wh, sku_id
        )

        # 更新调拨单状态
        self.db.update("transfer_order",
            {"status": "in_transit", "shipped_at": now()},
            {"transfer_no": transfer_no})

        # 触发物流通知
        self.mq.produce("logistics", {
            "transfer_no": transfer_no,
            "from": from_wh,
            "to": order["to_warehouse"],
            "sku_id": sku_id,
            "quantity": quantity
        })

    def receive_and_complete(self, transfer_no, received_qty, receiver):
        """目标仓库收货确认"""
        order = self.db.query_one(
            "SELECT * FROM transfer_order WHERE transfer_no = %s AND status = 'in_transit'",
            transfer_no
        )
        if not order:
            raise TransferOrderNotFound(transfer_no)

        to_wh = order["to_warehouse"]
        sku_id = order["sku_id"]
        expected_qty = order["quantity"]

        # 处理收货差异
        if received_qty == expected_qty:
            # 正常收货：transferring_in → available + total
            self.redis.hincrby(f"wh:inv:{to_wh}:{sku_id}", "transferring_in", -received_qty)
            self.redis.hincrby(f"wh:inv:{to_wh}:{sku_id}", "total_stock", received_qty)
            self.redis.hincrby(f"wh:inv:{to_wh}:{sku_id}", "available_stock", received_qty)

            self.db.execute(
                "UPDATE warehouse_inventory SET "
                "transferring_in = transferring_in - %s, "
                "total_stock = total_stock + %s, "
                "available_stock = available_stock + %s, "
                "version = version + 1 "
                "WHERE warehouse_id = %s AND sku_id = %s",
                received_qty, received_qty, received_qty, to_wh, sku_id
            )
            self.db.update("transfer_order",
                {"status": "completed", "received_at": now()},
                {"transfer_no": transfer_no})

        elif received_qty < expected_qty:
            # 短收：部分收货，差额退回源仓库
            shortage = expected_qty - received_qty

            # 目标仓库入库实际数量
            self.redis.hincrby(f"wh:inv:{to_wh}:{sku_id}", "transferring_in", -received_qty)
            self.redis.hincrby(f"wh:inv:{to_wh}:{sku_id}", "total_stock", received_qty)
            self.redis.hincrby(f"wh:inv:{to_wh}:{sku_id}", "available_stock", received_qty)

            # 源仓库退回短收数量
            self.redis.hincrby(f"wh:inv:{order['from_warehouse']}:{sku_id}",
                               "available_stock", shortage)
            self.redis.hincrby(f"wh:inv:{order['from_warehouse']}:{sku_id}",
                               "total_stock", shortage)

            self.db.update("transfer_order",
                {"status": "completed", "received_at": now()},
                {"transfer_no": transfer_no})

            self.alert_team(
                f"调拨短收: {transfer_no}, 预期{expected_qty}, 实收{received_qty}, "
                f"差额{shortage}件已退回源仓库"
            )

        else:
            # 多收：超额部分记录为盘盈，需审批
            surplus = received_qty - expected_qty
            self.redis.hincrby(f"wh:inv:{to_wh}:{sku_id}", "transferring_in", -expected_qty)
            self.redis.hincrby(f"wh:inv:{to_wh}:{sku_id}", "total_stock", received_qty)
            self.redis.hincrby(f"wh:inv:{to_wh}:{sku_id}", "available_stock", received_qty)

            self.db.update("transfer_order",
                {"status": "completed", "received_at": now()},
                {"transfer_no": transfer_no})

            self.alert_team(
                f"调拨多收: {transfer_no}, 预期{expected_qty}, 实收{received_qty}, "
                f"超收{surplus}件待审批"
            )

    def cancel_transfer(self, transfer_no, reason):
        """取消调拨单 → 释放冻结库存"""
        order = self.db.query_one(
            "SELECT * FROM transfer_order WHERE transfer_no = %s "
            "AND status IN ('created', 'confirmed')",
            transfer_no
        )
        if not order:
            raise TransferCannotCancel(transfer_no)

        from_wh = order["from_warehouse"]
        to_wh = order["to_warehouse"]
        sku_id = order["sku_id"]
        quantity = order["quantity"]

        # 源仓库：transferring_out → available（释放冻结）
        self.redis.hincrby(f"wh:inv:{from_wh}:{sku_id}", "transferring_out", -quantity)
        self.redis.hincrby(f"wh:inv:{from_wh}:{sku_id}", "available_stock", quantity)

        # 目标仓库：transferring_in 减少
        self.redis.hincrby(f"wh:inv:{to_wh}:{sku_id}", "transferring_in", -quantity)

        # 数据库同步
        self.db.execute(
            "UPDATE warehouse_inventory SET "
            "transferring_out = transferring_out - %s, "
            "available_stock = available_stock + %s, "
            "version = version + 1 "
            "WHERE warehouse_id = %s AND sku_id = %s",
            quantity, quantity, from_wh, sku_id
        )

        self.db.update("transfer_order",
            {"status": "cancelled"},
            {"transfer_no": transfer_no})

        self.audit_log("transfer_cancel", sku_id, quantity, "system", transfer_no)
```

**调拨库存状态流转图：**

```
源仓库库存状态：
  available → (创建调拨) → transferring_out → (出库) → total 减少
  
目标仓库库存状态：
  (创建调拨) → transferring_in → (收货) → available + total 增加

异常路径：
  transferring_out → (取消调拨) → available  （释放冻结）
  transferring_in  → (取消调拨) → transferring_in 减少
  in_transit        → (运输损毁) → 源仓库 available 退回差额
```

**自动调拨触发规则：**

```python
class AutoTransferTrigger:
    """自动调拨触发器：当某仓库库存低于安全值时，自动从其他仓库调拨"""

    def check_and_trigger(self):
        """定时检查所有仓库库存，触发自动调拨"""
        # 查询库存低于安全线的 SKU
        low_stock_items = self.db.query("""
            SELECT wi.warehouse_id, wi.sku_id, wi.available_stock,
                   ss.safety_stock, ss.reorder_point
            FROM warehouse_inventory wi
            JOIN sku_safety_stock ss ON wi.sku_id = ss.sku_id
                AND wi.warehouse_id = ss.warehouse_id
            WHERE wi.available_stock <= ss.reorder_point
        """)

        for item in low_stock_items:
            # 寻找有富余库存的仓库
            source = self.find_best_source(
                item["sku_id"], item["warehouse_id"],
                item["safety_stock"] - item["available_stock"]
            )
            if source:
                transfer_qty = self.calculate_transfer_qty(
                    item["available_stock"], item["safety_stock"],
                    source["available_stock"], source["safety_stock"]
                )
                self.transfer_service.create_transfer(
                    source["warehouse_id"], item["warehouse_id"],
                    item["sku_id"], transfer_qty, "system_auto"
                )

    def find_best_source(self, sku_id, target_wh, needed_qty):
        """寻找最佳调出仓库：距离最近、库存最充裕"""
        candidates = self.db.query("""
            SELECT wi.warehouse_id, wi.available_stock,
                   ss.safety_stock,
                   wd.distance_km,
                   (wi.available_stock - ss.safety_stock) as surplus
            FROM warehouse_inventory wi
            JOIN sku_safety_stock ss ON wi.sku_id = ss.sku_id
                AND wi.warehouse_id = ss.warehouse_id
            JOIN warehouse_distance wd ON wd.from_wh = wi.warehouse_id
                AND wd.to_wh = %s
            WHERE wi.sku_id = %s
              AND wi.available_stock > ss.safety_stock
            ORDER BY wd.distance_km ASC, surplus DESC
            LIMIT 1
        """, target_wh, sku_id)

        return candidates[0] if candidates else None

    def calculate_transfer_qty(self, target_available, target_safety,
                                source_available, source_safety):
        """计算调拨数量：目标补到安全库存，源不低于安全库存"""
        target_need = target_safety - target_available
        source_surplus = source_available - source_safety
        return min(target_need, source_surplus)
```

## 异常场景完整演练

**场景 1：天猫大促瞬间抢空配额**

```
状态：SKU-A 总库存 1000，天猫配额 400，APP 配额 300
事件：天猫大促开始 → 5 分钟内天猫下单 450 件

处理：
  1. 天猫配额 400 用完 → 借用共享池 50 件 → 天猫共 450 件
  2. 共享池剩余 -50 件 → 需要从其他渠道配额补充
  3. 自动缩减小程序配额 50 → 小程序配额变为 150
  4. 通知运营："天猫库存已用 450/400，已从小程序借调 50 件"
  5. 如果天猫继续下单 → APP/小程序显示"库存不足"
  6. 运营可手动调整配额：临时将天猫配额提高到 600
```

**场景 2：Redis 集群故障**

```
事件：Redis 主节点宕机 → Sentinel 切换从节点 → 10 秒不可用

影响：
  - 10 秒内所有冻结/确认请求失败
  - 用户下单失败 → 前端显示"系统繁忙"

处理：
  1. Sentinel 自动切换 → 10 秒后恢复
  2. 恢复后从 MySQL 全量加载库存数据到 Redis
  3. 对账：统计 MySQL 订单数 vs Redis 库存 → 修正差异
  4. 关键：切换期间不能让请求直接打 MySQL（2 万 TPS 会打垮 DB）
     → 降级返回"系统繁忙"让用户稍后重试
```

**场景 3：冻结超时释放与确认的竞态**

```
时刻 T0: 用户 A 下单冻结 1 件 SKU-B，30 分钟支付超时
时刻 T29m50s: 用户 A 开始支付
时刻 T30m00s: 冻结超时释放脚本执行 → 释放 1 件库存
时刻 T30m05s: 支付成功 → confirm 操作 → 但冻结记录已过期

这是最棘手的竞态：超时释放和支付确认几乎同时发生。

解决方案：confirm 时的幂等检查
  1. confirm 从 Redis 读取 inv:freeze:{order_id}
  2. 如果 key 已过期（被 SETEX 删除）→ key 不存在
  3. 查数据库 inventory_freeze_log：
     - status = 'frozen' → 还没超时释放 → 执行 confirm
     - status = 'expired' → 已超时释放 → 退款给用户
  4. 数据库 inventory_freeze_log 是最终真相
```

**场景 4：仓库系统完全离线**

```
事件：A 仓 WMS 系统升级，预计下线 2 小时，实际下线 6 小时

影响：
  - A 仓的入库、出库、调拨全部停止
  - 已创建的调拨单无法执行出库
  - 在途货物无法确认收货
  - 线上订单分配到 A 仓的无法发货

处理方案：
  1. 立即将 A 仓标记为"不可用"，路由分配跳过 A 仓
  2. 将 A 仓的线上订单重新分配到 B 仓（如果 B 仓有库存）
  3. 暂停所有目标为 A 仓的调拨单
  4. 6 小时后恢复流程：
     a. WMS 上线 → 上报离线期间的入库/出库记录
     b. 与中央库存系统对账 → 修正差异
     c. 重新开放 A 仓路由 → 恢复正常
  5. 预防：WMS 升级前提前通知中央系统，预留足够的其他仓库库存
```

**场景 5：部分发货失败**

```
事件：用户下单 3 件 SKU-A + 2 件 SKU-B，A 仓有货，发货时 B 仓库存不足

影响：
  - 订单部分可发，部分缺货
  - 如果拆单：先发 A，B 等补货后发
  - 如果整单：取消订单 + 释放库存

处理方案（拆单策略）：
  1. 有货的 SKU 先发货 → 确认扣减库存
  2. 缺货的 SKU 标记为"待补货"→ 冻结库存保持
  3. 补货超时（48小时）→ 通知用户选择：
     a. 继续等待（预计 X 天到货）
     b. 取消缺货商品 → 释放冻结 → 退款
     c. 替换为其他商品
  4. 拆单不影响用户收货体验：物流信息分两条跟踪

代码示例：
```python
class PartialShipmentHandler:
    """部分发货处理器"""

    def handle_shipment(self, order_id):
        """处理订单发货：拆分可发和缺货商品"""
        items = self.db.query(
            "SELECT oi.sku_id, oi.quantity, wi.available_stock "
            "FROM order_item oi "
            "LEFT JOIN warehouse_inventory wi ON oi.sku_id = wi.sku_id "
            "AND wi.warehouse_id = %s "
            "WHERE oi.order_id = %s",
            self.get_assigned_warehouse(order_id), order_id
        )

        shippable = []   # 可发货
        backordered = [] # 缺货

        for item in items:
            if item["available_stock"] >= item["quantity"]:
                shippable.append(item)
            else:
                backordered.append(item)

        if shippable:
            # 先发可发货部分
            self.create_shipment(order_id, shippable)
            for item in shippable:
                self.inventory_service.confirm(item["sku_id"], order_id)

        if backordered:
            # 缺货部分进入等待
            for item in backordered:
                self.db.insert("backorder_item", {
                    "order_id": order_id,
                    "sku_id": item["sku_id"],
                    "quantity": item["quantity"],
                    "status": "waiting",
                    "deadline": now() + timedelta(hours=48)
                })
                # 冻结库存保持（不释放，等补货）
                # 如果决定取消 → 调用 release
```

**场景 6：超卖恢复**

```
事件：Redis 故障恢复后对账发现某 SKU 超卖 20 件
      （总库存 100，实际已售 120）

影响：
  - 20 个订单无法发货
  - 需要决定哪些订单取消、哪些等待补货

处理方案（超卖恢复优先级）：
  1. 按渠道优先级取消：POS 门店 > 天猫 > APP > 小程序
     原因：门店客户可面对面解释+即时补偿
  2. 按下单时间取消：最后下的单优先取消
  3. 对被取消订单的用户：
     a. 全额退款 + 赔偿金（商品价值的 10%，最低 ¥20）
     b. 优先购买权：下次补货后优先通知
     c. 优惠券：满 100 减 20
  4. 紧急补货：从供应商紧急调货，2-3 天到货
  5. 修正系统库存：将 total_stock 设为实际值（负数清零）
```

```python
class OversellRecoveryService:
    """超卖恢复服务"""

    def recover(self, sku_id, oversold_qty):
        """恢复超卖：选择取消部分订单"""
        # Step 1: 找出所有已确认但未发货的订单
        confirmed_orders = self.db.query("""
            SELECT o.order_id, o.channel, o.created_at, oi.quantity
            FROM orders o
            JOIN order_item oi ON o.order_id = oi.order_id
            WHERE oi.sku_id = %s
              AND o.status = 'paid'
              AND o.shipping_status = 'unshipped'
            ORDER BY
              CASE o.channel
                WHEN 'store' THEN 1
                WHEN 'tmall' THEN 2
                WHEN 'app' THEN 3
                WHEN 'miniprogram' THEN 4
              END,
              o.created_at DESC
        """, sku_id)

        # Step 2: 取消超卖数量的订单
        cancelled_qty = 0
        for order in confirmed_orders:
            if cancelled_qty >= oversold_qty:
                break

            # 取消订单
            self.order_service.cancel(order["order_id"], reason="oversell_recovery")
            # 释放库存确认
            self.redis.incrby(f"inv:total:{sku_id}", order["quantity"])
            # 退款 + 赔偿
            self.payment_service.refund_with_compensation(
                order["order_id"],
                compensation_rate=0.10,
                min_compensation=20
            )
            # 发送通知
            self.notification_service.send(
                order["order_id"],
                template="oversell_apology",
                compensation=True
            )
            cancelled_qty += order["quantity"]

        # Step 3: 修正库存值
        self.db.execute(
            "UPDATE inventory SET total_stock = available_stock + frozen_stock, "
            "sold_stock = sold_stock - %s, version = version + 1 "
            "WHERE sku_id = %s",
            oversold_qty, sku_id
        )
        self.redis.set(f"inv:total:{sku_id}",
                       int(self.redis.get(f"inv:frozen:{sku_id} or 0"))
                       + int(self.redis.get(f"inv:available:{sku_id} or 0")))
```

**场景 7：并发补货竞态**

```
事件：供应商同时送达两批货物，两个仓库管理员同时录入入库
      操作员A：入库 500 件 → available += 500
      操作员B：入库 300 件 → available += 300
      期望：available = 原 + 800

风险：如果 available 的更新是 read-modify-write 模式：
      A 读取 available=100 → B 读取 available=100
      A 写入 available=600 → B 写入 available=400
      结果丢失了 A 的更新（应该是 900）

解决方案：
  1. 使用 Redis INCRBY（原子递增，非 read-modify-write）
  2. 数据库使用乐观锁：UPDATE ... SET available = available + 500, version = version + 1
     WHERE version = 旧版本
  3. 入库单去重：同一入库单号只能操作一次
```

```python
class ConcurrentRestockHandler:
    """并发补货处理器"""

    RESTOCK_LUA = """
    local total_key = KEYS[1]
    local qty = tonumber(ARGV[1])
    local restock_id = ARGV[2]

    -- 幂等检查：同一入库单不能重复操作
    if redis.call('EXISTS', 'restock:' .. restock_id) == 1 then
        return -1  -- 重复操作
    end

    -- 原子递增
    redis.call('INCRBY', total_key, qty)
    -- 标记入库单已处理
    redis.call('SETEX', 'restock:' .. restock_id, 86400, '1')
    return 1
    """

    def restock(self, sku_id, quantity, restock_id, channel="warehouse"):
        """入库补货（原子 + 幂等）"""
        # Step 1: Redis 原子递增
        result = self.redis.eval(
            self.RESTOCK_LUA, 1,
            f"inv:total:{sku_id}",
            quantity, restock_id
        )

        if result == -1:
            raise DuplicateRestockOperation(restock_id)

        # Step 2: 数据库乐观锁更新
        max_retries = 3
        for attempt in range(max_retries):
            inv = self.db.query_one(
                "SELECT available_stock, version FROM inventory "
                "WHERE sku_id = %s FOR UPDATE", sku_id
            )
            affected = self.db.execute(
                "UPDATE inventory SET "
                "total_stock = total_stock + %s, "
                "available_stock = available_stock + %s, "
                "version = version + 1, "
                "updated_at = NOW() "
                "WHERE sku_id = %s AND version = %s",
                quantity, quantity, sku_id, inv["version"]
            )
            if affected > 0:
                break
            if attempt == max_retries - 1:
                raise OptimisticLockConflict(sku_id)

        # Step 3: 异步同步渠道配额（补货后可能需要增加配额）
        self.mq.produce("restock_sync", {
            "sku_id": sku_id,
            "quantity": quantity,
            "restock_id": restock_id
        })

        self.audit_log("restock", sku_id, quantity, channel, restock_id)
```

## 性能分析

**Redis 资源需求：**

| 数据 | 数量 | 单条大小 | 总计 |
|------|------|---------|------|
| 库存(total+frozen) | 10 万 SKU × 2 key | ~50 字节 | 10MB |
| 冻结明细 | 5 万活跃订单 × 1 key | ~200 字节 | 10MB |
| 渠道配额 | 10 万 SKU × 4 渠道 | ~30 字节 | 12MB |
| 共享池 | 10 万 SKU × 1 key | ~20 字节 | 2MB |
| 总计 | | | ~34MB |

**按操作类型的 TPS 详细分析：**

| 操作类型 | 峰值 TPS | Redis 操作 | DB 操作 | MQ 消息 | 端到端延迟 |
|---------|---------|-----------|--------|--------|-----------|
| 冻结库存 | 20,000 | 1 次 Lua (< 2ms) | 异步写入 | 1 条 | < 5ms |
| 确认扣减 | 10,000 | 1 次 Pipeline 3 命令 (< 3ms) | 异步更新 | 1 条 | < 5ms |
| 释放冻结 | 5,000 | 2 次独立命令 (< 3ms) | 异步更新 | 1 条 | < 5ms |
| 配额冻结 | 15,000 | 1 次 Lua (< 3ms) | 异步写入 | 1 条 | < 5ms |
| 查询库存 | 50,000 | 1 次 GET (< 1ms) | 无 | 无 | < 2ms |
| 补货入库 | 500 | 1 次 Lua (< 2ms) | 1 次乐观锁更新 | 1 条 | < 10ms |
| 跨仓调拨 | 200 | 3 次 HINCRBY (< 3ms) | 2 次事务更新 | 1 条 | < 20ms |
| 盘点调整 | 50 | 1 次 INCRBY (< 2ms) | 1 次事务更新 | 1 条 | < 20ms |

**总 TPS 上限分析：**

```
理论 Redis 单节点上限：10 万 QPS
3 节点 Cluster 总 QPS：30 万
当前峰值 QPS：~10 万（含读操作）
余量：3 倍 → 可支撑 3 倍业务增长

瓶颈不在 Redis，而在 MQ 消费端的 DB 写入：
  - DB 写入 TPS：单主 ~5000 TPS
  - 10 万 SKU + 日均 50 万订单 → 平均每秒 DB 写入 ~700 次
  - 峰值 2 万 TPS 的异步写入 → 需要批量合并：
    每 100ms 或累积 50 条后批量 INSERT → 降低 DB 压力
```

**Redis 内存详细估算（10 万 SKU + 多仓库扩展）：**

| 维度 | Key 设计 | 内存计算 | 总计 |
|------|---------|---------|------|
| 全局库存 | `inv:total:{sku}`, `inv:frozen:{sku}` | 10 万 × 2 × 64B | 12.8 MB |
| 渠道配额 | `inv:quota:{sku}:{channel}` | 10 万 × 4 × 48B | 19.2 MB |
| 共享池 | `inv:shared:{sku}` | 10 万 × 1 × 40B | 4.0 MB |
| 冻结明细 | `inv:freeze:{order_id}` | 5 万 × 256B | 12.8 MB |
| 仓库库存 | `wh:inv:{wh}:{sku}` (Hash) | 10 万 × 5 仓 × 128B | 64.0 MB |
| 入库幂等 | `restock:{id}` | 1 万/天 × 64B | 0.6 MB |
| **总计** | | | **113.4 MB** |

单节点 Redis 可用内存通常 8-16 GB，113 MB 占比极低。内存不是瓶颈。

**数据库分片策略：**

10 万 SKU + 50 万日订单 + 审计日志 → 单库单表很快达到瓶颈。

```
分片策略：按 sku_id 哈希分库分表

分库规则：
  db_index = crc32(sku_id) % 8  → 8 个库

分表规则：
  table_index = crc32(sku_id) % 4  → 每库 4 张表

总计：8 × 4 = 32 张表

各表数据分布：
  inventory:          10 万 / 32 ≈ 3125 行/表  → 极小，查询极快
  inventory_freeze_log: 每日 50 万行 / 32 ≈ 1.56 万行/表/天
  inventory_audit_log:  每日 50 万行 / 32 ≈ 1.56 万行/表/天

冷热分离：
  freeze_log / audit_log → 30 天后归档到历史表
  历史表按月分区 → 查询自动裁剪

跨分片查询处理：
  运营大盘查询 → 使用汇总表（每日凌晨 ETL 聚合）
  单 SKU 查询 → 直接路由到对应分片
  跨 SKU 聚合 → 并行查所有分片 + 内存合并
```

**成本估算（月度）：**

| 资源 | 规格 | 数量 | 月成本 |
|------|------|------|--------|
| Redis Cluster | 8GB 主从 | 3 节点 | ¥6,000 |
| MySQL | 8C32GB SSD | 8 分片 × 主从 | ¥32,000 |
| MQ (RocketMQ) | 标准 | 1 集群 | ¥5,000 |
| 监控 (Prometheus) | - | 1 套 | ¥2,000 |
| **总计** | | | **¥45,000/月** |

对比超卖损失：日均 50 万订单 × 超卖率 0.1% × 客单价 ¥200 = ¥10 万/天。系统成本仅相当于半天的超卖损失。

## 延伸思考

- **库存预测**：基于历史销量预测各渠道需求，动态调整配额比例。如双 11 前自动提高天猫配额至 60%，大促后恢复到 40%。
- **跨仓调拨**：A 仓缺货但 B 仓有货 → 自动调拨。调拨期间库存状态：增加"transferring"状态，total = available + frozen + transferring + sold。调拨到达后 transferring → available。
- **预售模式**：库存为 0 时仍可下单（预售），到货后优先发货。需增加"预售库存"维度：total = available + frozen + pre_sale + sold。预售订单不冻结库存，到货后再冻结并发货。

## 需求预测与自动补货

库存管理的终极目标是"在正确的时间补正确的货"。手动补货依赖运营经验，容易出现滞后或过度补货。基于历史销量数据的需求预测可以自动触发补货，减少缺货损失和库存积压。

### 预测模型选型

| 模型 | 适用场景 | 准确度 | 复杂度 | 实时性 |
|------|---------|--------|--------|--------|
| 移动平均 | 稳定销量 | 中 | 低 | 高 |
| 指数平滑 | 趋势+季节性 | 中高 | 低 | 高 |
| ARIMA | 时间序列 | 高 | 中 | 中 |
| Prophet | 多季节性+节假日 | 高 | 中 | 中 |
| LSTM | 非线性模式 | 很高 | 高 | 低 |

实际落地选择**指数平滑 + 节假日修正**：兼顾准确度和工程复杂度。

### 需求预测核心代码

```python
import math
from datetime import timedelta
from collections import defaultdict

class DemandForecaster:
    """基于指数平滑的需求预测"""

    def __init__(self, db, redis):
        self.db = db
        self.redis = redis
        # 平滑系数：alpha 越大，近期数据权重越高
        self.alpha = 0.3
        # 趋势平滑系数
        self.beta = 0.1
        # 季节性平滑系数
        self.gamma = 0.2

    def forecast(self, sku_id, warehouse_id, days_ahead=7):
        """预测未来 N 天的需求量"""
        # 获取最近 90 天的日销量数据
        daily_sales = self.db.query("""
            SELECT DATE(created_at) as sale_date, SUM(quantity) as qty
            FROM order_item oi
            JOIN orders o ON oi.order_id = o.order_id
            WHERE oi.sku_id = %s
              AND o.warehouse_id = %s
              AND o.status IN ('paid', 'shipped', 'completed')
              AND o.created_at >= DATE_SUB(NOW(), INTERVAL 90 DAY)
            GROUP BY DATE(created_at)
            ORDER BY sale_date
        """, sku_id, warehouse_id)

        if len(daily_sales) < 14:
            # 数据不足 → 使用简单移动平均
            return self._simple_moving_average(daily_sales, days_ahead)

        # Holt-Winters 指数平滑（趋势 + 季节性）
        return self._holt_winters(daily_sales, days_ahead)

    def _holt_winters(self, daily_sales, days_ahead):
        """Holt-Winters 双参数指数平滑：水平 + 趋势 + 季节性"""
        values = [s["qty"] for s in daily_sales]
        n = len(values)

        # 初始化
        level = values[0]
        trend = values[1] - values[0] if n > 1 else 0

        # 7 天季节性周期
        season_length = 7
        seasons = [0.0] * season_length
        # 用第一个周期的数据初始化季节因子
        if n >= season_length * 2:
            first_cycle_avg = sum(values[:season_length]) / season_length
            for i in range(season_length):
                seasons[i] = values[i] / first_cycle_avg if first_cycle_avg > 0 else 1.0

        # 迭代计算
        for t in range(1, n):
            season_idx = t % season_length
            prev_level = level

            # 水平方程
            level = self.alpha * (values[t] / seasons[season_idx]) + \
                    (1 - self.alpha) * (level + trend)

            # 趋势方程
            trend = self.beta * (level - prev_level) + (1 - self.beta) * trend

            # 季节性方程
            seasons[season_idx] = self.gamma * (values[t] / level) + \
                                  (1 - self.gamma) * seasons[season_idx]

        # 预测
        forecasts = []
        for d in range(days_ahead):
            season_idx = (n + d) % season_length
            predicted = max(0, (level + trend * (d + 1)) * seasons[season_idx])
            forecasts.append({
                "day_offset": d + 1,
                "predicted_qty": round(predicted),
                "confidence": max(0.5, 1.0 - d * 0.05)  # 越远越不确定
            })

        return forecasts

    def _simple_moving_average(self, daily_sales, days_ahead):
        """简单移动平均（数据不足时的降级方案）"""
        if not daily_sales:
            return [{"day_offset": d + 1, "predicted_qty": 0, "confidence": 0.3}
                    for d in range(days_ahead)]

        recent_avg = sum(s["qty"] for s in daily_sales[-7:]) / min(len(daily_sales), 7)
        return [{
            "day_offset": d + 1,
            "predicted_qty": round(recent_avg),
            "confidence": 0.4
        } for d in range(days_ahead)]
```

### 节假日修正

```python
class HolidayAdjuster:
    """节假日修正器：大促、节假日对销量的倍率影响"""

    # 节假日销量倍率（基于历史数据统计）
    HOLIDAY_MULTIPLIERS = {
        "double_11": 8.0,     # 双 11：8 倍销量
        "double_12": 3.0,     # 双 12：3 倍
        "spring_festival": 0.3,  # 春节：0.3 倍（大部分门店关闭）
        "national_day": 2.0,    # 国庆：2 倍
        "mid_autumn": 1.5,      # 中秋：1.5 倍
        "labor_day": 1.8,       # 五一：1.8 倍
    }

    # 大促预热期：正式日前 N 天开始增量
    PRE_HEAT_DAYS = {
        "double_11": 10,   # 双 11 前 10 天开始预热
        "double_12": 5,
    }

    def adjust_forecast(self, sku_id, forecasts):
        """对预测结果进行节假日修正"""
        adjusted = []
        for f in forecasts:
            target_date = now().date() + timedelta(days=f["day_offset"])

            # 检查是否在节假日
            holiday = self.get_holiday(target_date)
            multiplier = 1.0

            if holiday and holiday in self.HOLIDAY_MULTIPLIERS:
                multiplier = self.HOLIDAY_MULTIPLIERS[holiday]

            # 检查是否在预热期
            pre_heat = self.get_pre_heat_period(target_date)
            if pre_heat:
                # 预热期渐增：越接近大促日，倍率越高
                days_to_event = (self.get_event_date(pre_heat) - target_date).days
                peak_multiplier = self.HOLIDAY_MULTIPLIERS.get(pre_heat, 1.0)
                # 线性增长：预热首日 1.2x → 大促前日 0.7x
                pre_heat_days = self.PRE_HEAT_DAYS.get(pre_heat, 5)
                multiplier = 1.0 + (peak_multiplier - 1.0) * 0.3 * \
                             (1 - days_to_event / pre_heat_days)

            # 渠道修正：不同 SKU 在不同节日的表现不同
            channel_multiplier = self.get_channel_holiday_factor(sku_id, target_date)
            multiplier *= channel_multiplier

            adjusted.append({
                "day_offset": f["day_offset"],
                "predicted_qty": round(f["predicted_qty"] * multiplier),
                "original_qty": f["predicted_qty"],
                "multiplier": round(multiplier, 2),
                "confidence": f["confidence"] * (0.8 if multiplier > 2 else 1.0)
            })

        return adjusted

    def get_channel_holiday_factor(self, sku_id, target_date):
        """SKU 级别的节假日修正因子"""
        # 从历史数据中学习该 SKU 在类似节假日的表现
        sku_holiday_ratio = self.db.query_one("""
            SELECT AVG(actual_qty / predicted_qty) as ratio
            FROM forecast_accuracy_log
            WHERE sku_id = %s
              AND is_holiday = 1
              AND accuracy_date >= DATE_SUB(NOW(), INTERVAL 365 DAY)
        """, sku_id)

        if sku_holiday_ratio and sku_holiday_ratio["ratio"]:
            return min(max(sku_holiday_ratio["ratio"], 0.5), 3.0)
        return 1.0
```

### 自动补货触发

```python
class AutoReplenishmentTrigger:
    """自动补货触发器：基于预测结果 + 安全库存模型"""

    def __init__(self, db, redis, forecaster, holiday_adjuster):
        self.db = db
        self.redis = redis
        self.forecaster = forecaster
        self.holiday_adjuster = holiday_adjuster

    def check_and_trigger(self):
        """每日凌晨运行：检查所有 SKU 是否需要补货"""
        skus = self.db.query(
            "SELECT sku_id, warehouse_id FROM warehouse_inventory "
            "WHERE available_stock > 0 OR sold_stock > 0"
        )

        for item in skus:
            self._check_single_sku(item["sku_id"], item["warehouse_id"])

    def _check_single_sku(self, sku_id, warehouse_id):
        """检查单个 SKU 是否需要补货"""
        # 1. 获取当前库存
        current = self.db.query_one(
            "SELECT available_stock, frozen_stock FROM warehouse_inventory "
            "WHERE warehouse_id = %s AND sku_id = %s",
            warehouse_id, sku_id
        )
        available = current["available_stock"]

        # 2. 获取预测需求
        forecasts = self.forecaster.forecast(sku_id, warehouse_id, days_ahead=14)
        forecasts = self.holiday_adjuster.adjust_forecast(sku_id, forecasts)

        # 3. 计算补货前置时间内的总需求
        lead_time_days = self.get_lead_time(sku_id, warehouse_id)  # 供应商交货天数
        lead_time_demand = sum(
            f["predicted_qty"] for f in forecasts[:lead_time_days]
        )

        # 4. 计算安全库存（考虑需求波动性）
        safety_stock = self._calculate_safety_stock(
            sku_id, warehouse_id, forecasts, lead_time_days
        )

        # 5. 计算补货点
        reorder_point = lead_time_demand + safety_stock

        # 6. 判断是否需要补货
        if available <= reorder_point:
            # 需要补货：计算补货量
            # 目标库存 = 补货前置时间需求 + 安全库存 + 补货周期需求
            review_cycle_days = 7  # 补货周期 7 天
            review_cycle_demand = sum(
                f["predicted_qty"] for f in forecasts[:review_cycle_days]
            )
            target_stock = lead_time_demand + safety_stock + review_cycle_demand

            replenish_qty = target_stock - available

            # 补货量合理性校验
            replenish_qty = self._validate_replenish_qty(
                sku_id, warehouse_id, replenish_qty
            )

            if replenish_qty > 0:
                self._create_replenishment_order(
                    sku_id, warehouse_id, replenish_qty, {
                        "available": available,
                        "reorder_point": reorder_point,
                        "safety_stock": safety_stock,
                        "lead_time_demand": lead_time_demand,
                        "review_cycle_demand": review_cycle_demand
                    }
                )

        # 缓存预测结果（供大盘展示）
        self.redis.setex(
            f"forecast:{warehouse_id}:{sku_id}", 86400,
            json.dumps({
                "forecasts": forecasts,
                "reorder_point": reorder_point,
                "safety_stock": safety_stock,
                "current_available": available
            })
        )

    def _calculate_safety_stock(self, sku_id, warehouse_id, forecasts,
                                 lead_time_days):
        """安全库存 = Z × σ_demand × √lead_time"""
        # 获取历史日销量的标准差
        daily_sales = self.db.query("""
            SELECT SUM(quantity) as qty
            FROM order_item oi
            JOIN orders o ON oi.order_id = o.order_id
            WHERE oi.sku_id = %s AND o.warehouse_id = %s
              AND o.status IN ('paid', 'shipped', 'completed')
              AND o.created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
            GROUP BY DATE(o.created_at)
        """, sku_id, warehouse_id)

        if len(daily_sales) < 7:
            # 数据不足 → 使用预测值的 30% 作为安全库存
            avg_forecast = sum(f["predicted_qty"] for f in forecasts[:7]) / 7
            return int(avg_forecast * 0.3 * lead_time_days)

        # 计算标准差
        quantities = [s["qty"] for s in daily_sales]
        mean = sum(quantities) / len(quantities)
        variance = sum((q - mean) ** 2 for q in quantities) / len(quantities)
        std_dev = math.sqrt(variance)

        # Z 值：95% 服务水平 → Z = 1.65
        z_score = 1.65

        safety_stock = z_score * std_dev * math.sqrt(lead_time_days)
        return max(int(safety_stock), 5)  # 最低 5 件

    def _validate_replenish_qty(self, sku_id, warehouse_id, qty):
        """补货量合理性校验"""
        # 上限：不超过仓库容量
        max_capacity = self.get_warehouse_capacity(warehouse_id, sku_id)
        current_total = self.db.query_one(
            "SELECT total_stock FROM warehouse_inventory "
            "WHERE warehouse_id = %s AND sku_id = %s",
            warehouse_id, sku_id
        )["total_stock"]

        remaining_capacity = max_capacity - current_total
        qty = min(qty, remaining_capacity)

        # 下限：供应商最低起订量
        moq = self.get_supplier_moq(sku_id)
        if qty < moq:
            qty = moq  # 不足起订量则按起订量补

        # 单次上限：不超过供应商最大供货量
        max_order = self.get_supplier_max_order(sku_id)
        qty = min(qty, max_order)

        return qty

    def _create_replenishment_order(self, sku_id, warehouse_id, qty, metrics):
        """创建补货采购单"""
        po_no = f"PO-{warehouse_id}-{sku_id}-{now().strftime('%Y%m%d%H%M%S')}"

        self.db.insert("purchase_order", {
            "po_no": po_no,
            "sku_id": sku_id,
            "warehouse_id": warehouse_id,
            "quantity": qty,
            "status": "pending_approval",
            "trigger_type": "auto_forecast",
            "metrics_json": json.dumps(metrics),
            "created_at": now()
        })

        # 通知采购团队
        self.notification_service.send(
            target="procurement_team",
            template="auto_replenishment",
            data={
                "po_no": po_no,
                "sku_id": sku_id,
                "warehouse_id": warehouse_id,
                "quantity": qty,
                "available_stock": metrics["available"],
                "reorder_point": metrics["reorder_point"],
                "safety_stock": metrics["safety_stock"]
            }
        )

        self.audit_log("auto_replenishment", sku_id, qty, "system", po_no)
```

### 预测准确度监控与反馈

```python
class ForecastAccuracyMonitor:
    """预测准确度监控 → 持续修正模型参数"""

    def calculate_accuracy(self, sku_id, warehouse_id, target_date):
        """计算预测准确度（WAPE: Weighted Absolute Percentage Error）"""
        actual = self.db.query_one("""
            SELECT SUM(quantity) as qty
            FROM order_item oi
            JOIN orders o ON oi.order_id = o.order_id
            WHERE oi.sku_id = %s AND o.warehouse_id = %s
              AND DATE(o.created_at) = %s
              AND o.status IN ('paid', 'shipped', 'completed')
        """, sku_id, warehouse_id, target_date)

        actual_qty = actual["qty"] or 0

        # 从缓存获取预测值
        cached = self.redis.get(f"forecast:{warehouse_id}:{sku_id}")
        if not cached:
            return None

        forecasts = json.loads(cached)["forecasts"]
        day_offset = (target_date - now().date()).days
        predicted_qty = next(
            (f["predicted_qty"] for f in forecasts if f["day_offset"] == day_offset),
            None
        )

        if predicted_qty is None:
            return None

        # WAPE = |actual - predicted| / max(actual, 1)
        error = abs(actual_qty - predicted_qty)
        wape = error / max(actual_qty, 1)

        # 记录准确度日志（用于后续模型调优）
        self.db.insert("forecast_accuracy_log", {
            "sku_id": sku_id,
            "warehouse_id": warehouse_id,
            "target_date": target_date,
            "predicted_qty": predicted_qty,
            "actual_qty": actual_qty,
            "absolute_error": error,
            "wape": wape,
            "is_holiday": self.is_holiday(target_date),
            "created_at": now()
        })

        return {"wape": wape, "actual": actual_qty, "predicted": predicted_qty}

    def get_accuracy_report(self, warehouse_id, days=30):
        """生成准确度报告"""
        report = self.db.query("""
            SELECT
                AVG(wape) as avg_wape,
                COUNT(*) as sample_count,
                SUM(CASE WHEN wape < 0.2 THEN 1 ELSE 0 END) as accurate_count,
                SUM(CASE WHEN wape > 0.5 THEN 1 ELSE 0 END) as poor_count
            FROM forecast_accuracy_log
            WHERE warehouse_id = %s
              AND target_date >= DATE_SUB(NOW(), INTERVAL %s DAY)
        """, warehouse_id, days)

        avg_wape = report[0]["avg_wape"] or 1.0
        accuracy_pct = (1 - avg_wape) * 100

        return {
            "accuracy_percent": round(accuracy_pct, 1),
            "sample_count": report[0]["sample_count"],
            "accurate_rate": round(
                report[0]["accurate_count"] / max(report[0]["sample_count"], 1) * 100, 1
            ),
            "poor_rate": round(
                report[0]["poor_count"] / max(report[0]["sample_count"], 1) * 100, 1
            ),
            "grade": "A" if accuracy_pct >= 85 else
                     "B" if accuracy_pct >= 75 else
                     "C" if accuracy_pct >= 60 else "D"
        }
```

**预测 → 补货完整链路：**

```
每日凌晨 2:00 执行：
  1. DemandForecaster.forecast() → 未来 14 天需求预测
  2. HolidayAdjuster.adjust_forecast() → 节假日修正
  3. AutoReplenishmentTrigger.check_and_trigger() → 生成补货单
  4. 采购团队审批 → 确认 → 供应商发货
  5. 入库 → ConcurrentRestockHandler.restock() → 更新库存

每日凌晨 3:00 执行：
  6. ForecastAccuracyMonitor.calculate_accuracy() → 昨日预测 vs 实际
  7. 月度调整：如果 WAPE > 30% → 调整 alpha/beta/gamma 参数
```

## 系统架构总览

```
                    ┌─────────────────────────────────┐
                    │         渠道接入层               │
                    │  APP / 小程序 / 天猫 / 京东 / POS │
                    └──────────┬──────────────────────┘
                               │
                    ┌──────────▼──────────────────────┐
                    │       网关 / 限流 / 鉴权          │
                    └──────────┬──────────────────────┘
                               │
          ┌────────────────────┼───────────────────────┐
          │                    │                        │
┌─────────▼──────────┐ ┌──────▼───────┐ ┌──────────────▼──────┐
│  库存冻结/确认/释放  │ │  渠道配额管理  │ │   订单路由分发      │
│  (Redis Lua 原子)   │ │ (Redis Lua)   │ │ (仓库分配+调拨)     │
└─────────┬──────────┘ └──────┬───────┘ └──────────────┬──────┘
          │                    │                        │
          └────────────────────┼───────────────────────┘
                               │
                    ┌──────────▼──────────────────────┐
                    │         消息队列 (MQ)             │
                    │  inventory_sync / restock_sync   │
                    └──────────┬──────────────────────┘
                               │
          ┌────────────────────┼───────────────────────┐
          │                    │                        │
┌─────────▼──────────┐ ┌──────▼───────┐ ┌──────────────▼──────┐
│  库存数据库 (分片)   │ │ 对账/盘点服务  │ │  需求预测+自动补货   │
│  8库×4表 MySQL      │ │ (定时任务)    │ │  (定时+事件触发)     │
└─────────┬──────────┘ └──────┬───────┘ └──────────────┬──────┘
          │                    │                        │
          └────────────────────┼───────────────────────┘
                               │
                    ┌──────────▼──────────────────────┐
                    │     审计日志 / 监控告警            │
                    │  Prometheus + Grafana            │
                    └─────────────────────────────────┘
```

**关键监控指标：**

| 指标 | 告警阈值 | 说明 |
|------|---------|------|
| 库存不一致率 | > 0.01% | Redis vs DB 对账差异比例 |
| 冻结超时未释放数 | > 10 | frozen 值与活跃冻结键不一致 |
| 超卖订单数 | > 0 | 任何超卖都需立即告警 |
| 渠道配额耗尽率 | > 80% | 某渠道配额使用率超 80% |
| 预测准确度 WAPE | > 30% | 预测误差过大需调整参数 |
| 补货触发延迟 | > 2 小时 | 低于补货点后 2 小时未触发 |
| Redis 延迟 P99 | > 10ms | Redis 响应变慢 |
| MQ 消息积压 | > 10000 | 消费端处理能力不足 |
## 库存预警与补货引擎

```python
class InventoryAlertEngine:
    """库存预警与自动补货"""

    def check_and_replenish(self):
        """检查所有 SKU 库存，触发补货"""
        alerts = []
        skus = self.db.query("SELECT * FROM sku_inventory")

        for sku in skus:
            # 计算日均消耗
            daily_consumption = self._calc_daily_consumption(sku["sku_id"])
            # 当前库存可维持天数
            days_of_stock = sku["available_qty"] / max(daily_consumption, 0.1)

            if days_of_stock < sku["reorder_point_days"]:
                # 低于补货点 → 触发补货
                replenish_qty = self._calc_replenish_qty(
                    sku, daily_consumption, target_days=30)
                self._create_replenish_order(sku["sku_id"], replenish_qty)
                alerts.append({
                    "sku_id": sku["sku_id"],
                    "current_qty": sku["available_qty"],
                    "days_of_stock": days_of_stock,
                    "reorder_point": sku["reorder_point_days"],
                    "replenish_qty": replenish_qty,
                    "urgency": "CRITICAL" if days_of_stock < 3 else "HIGH"
                })

            elif days_of_stock < sku["warning_point_days"]:
                alerts.append({
                    "sku_id": sku["sku_id"],
                    "current_qty": sku["available_qty"],
                    "days_of_stock": days_of_stock,
                    "urgency": "WARNING"
                })

        return alerts

    def _calc_replenish_qty(self, sku, daily_consumption, target_days):
        """计算补货数量"""
        # 补货量 = 目标库存 - 当前库存 + 在途库存
        target_qty = daily_consumption * target_days
        in_transit = self.db.sum("purchase_orders", "qty",
            sku_id=sku["sku_id"], status="in_transit")
        replenish_qty = target_qty - sku["available_qty"] - in_transit
        return max(0, int(replenish_qty))
```

## 供应商评分模型

```python
class SupplierScoringService:
    """供应商综合评分"""

    DIMENSIONS = {
        "delivery_rate": {"weight": 0.3, "description": "准时交付率"},
        "quality_rate": {"weight": 0.3, "description": "质量合格率"},
        "price_competitiveness": {"weight": 0.2, "description": "价格竞争力"},
        "response_speed": {"weight": 0.2, "description": "响应速度"},
    }

    def score_supplier(self, supplier_id):
        """计算供应商综合评分"""
        scores = {}
        # 准时交付率：按时交付订单数 / 总订单数
        scores["delivery_rate"] = self.db.query_one(
            "SELECT COUNT(CASE WHEN actual_delivery <= promised_delivery THEN 1 END) * 100.0 / COUNT(*) "
            "FROM purchase_orders WHERE supplier_id = %s", supplier_id)["score"]

        # 质量合格率：合格批次 / 总批次
        scores["quality_rate"] = self.db.query_one(
            "SELECT COUNT(CASE WHEN quality_check = 'pass' THEN 1 END) * 100.0 / COUNT(*) "
            "FROM inbound_inspections WHERE supplier_id = %s", supplier_id)["score"]

        # 价格竞争力：与市场均价比较
        scores["price_competitiveness"] = self._calc_price_score(supplier_id)

        # 响应速度：询价到报价的平均时间
        scores["response_speed"] = self._calc_response_score(supplier_id)

        # 加权综合评分
        total = sum(scores[k] * self.DIMENSIONS[k]["weight"] for k in scores)
        return {"supplier_id": supplier_id, "scores": scores,
                "total_score": total, "grade": self._score_to_grade(total)}
```

## 需求预测完整实现

```python
class DemandForecaster:
    """需求预测：季节性分解 + 特征工程"""

    def forecast(self, sku_id, horizon_days=30):
        """预测未来 N 天需求"""
        # 1. 获取历史数据
        history = self.db.query(
            "SELECT date, daily_sales FROM sku_daily_sales "
            "WHERE sku_id = %s ORDER BY date DESC LIMIT 365", sku_id)

        # 2. 季节性分解：趋势 + 季节 + 残差
        trend = self._extract_trend(history)
        seasonal = self._extract_seasonal(history)
        residual = self._extract_residual(history)

        # 3. 特征工程
        features = self._build_features(sku_id, horizon_days)
        # 特征包括：节假日、促销事件、天气、星期几

        # 4. 预测
        forecast_values = []
        for day in range(horror_days):
            date = now().date() + timedelta(days=day)
            base = trend[-1] + seasonal[day % 7]  # 趋势 + 周季节性
            holiday_effect = features.get(date, {}).get("holiday_multiplier", 1.0)
            promo_effect = features.get(date, {}).get("promo_multiplier", 1.0)
            forecast_values.append({
                "date": date,
                "predicted_demand": max(0, int(base * holiday_effect * promo_effect)),
                "confidence_lower": max(0, int(base * 0.8)),
                "confidence_upper": int(base * 1.2)
            })

        return {"sku_id": sku_id, "forecast": forecast_values}

    def _build_features(self, sku_id, horizon_days):
        """构建预测特征"""
        features = {}
        for day in range(horizon_days):
            date = now().date() + timedelta(days=day)
            features[date] = {
                "is_holiday": self.holiday_calendar.is_holiday(date),
                "holiday_multiplier": 1.5 if self.holiday_calendar.is_holiday(date) else 1.0,
                "is_promotion": self.promo_calendar.is_active(sku_id, date),
                "promo_multiplier": 2.0 if self.promo_calendar.is_active(sku_id, date) else 1.0,
                "day_of_week": date.weekday(),
                "is_weekend": date.weekday() >= 5
            }
        return features

    def calculate_accuracy(self, sku_id):
        """计算预测准确率（MAPE）"""
        actuals = self.db.query(
            "SELECT date, daily_sales FROM sku_daily_sales "
            "WHERE sku_id = %s AND date >= %s ORDER BY date",
            sku_id, now() - timedelta(days=30))

        total_ape = 0
        for actual in actuals:
            forecast = self.db.query_one(
                "SELECT predicted_demand FROM demand_forecasts "
                "WHERE sku_id = %s AND date = %s", sku_id, actual["date"])
            if forecast:
                ape = abs(actual["daily_sales"] - forecast["predicted_demand"]) / max(actual["daily_sales"], 1)
                total_ape += ape

        mape = total_ape / len(actuals) * 100
        return {"sku_id": sku_id, "mape": mape, "accuracy": 100 - mape}
```

## 供应商风险评估

```python
class SupplierRiskAssessor:
    """供应商风险评估"""

    RISK_DIMENSIONS = {
        "financial": {"weight": 0.3, "indicators": ["营收增长率", "资产负债率", "现金流"]},
        "geographic": {"weight": 0.2, "indicators": ["政治风险", "自然灾害频率", "物流距离"]},
        "operational": {"weight": 0.3, "indicators": ["产能利用率", "良品率", "交付准时率"]},
        "compliance": {"weight": 0.2, "indicators": ["ISO认证", "环保合规", "劳工标准"]},
    }

    def assess(self, supplier_id):
        """综合风险评估"""
        scores = {}
        for dim, config in self.RISK_DIMENSIONS.items():
            dim_score = self._score_dimension(supplier_id, dim, config["indicators"])
            scores[dim] = dim_score

        # 加权总评分
        total = sum(scores[k] * self.RISK_DIMENSIONS[k]["weight"] for k in scores)
        risk_level = "LOW" if total > 80 else "MEDIUM" if total > 60 else "HIGH"

        # 高风险 → 自动激活应急方案
        if risk_level == "HIGH":
            self._activate_contingency(supplier_id, scores)

        return {
            "supplier_id": supplier_id,
            "scores": scores,
            "total_score": total,
            "risk_level": risk_level,
            "recommendation": self._get_recommendation(total)
        }

    def _activate_contingency(self, supplier_id, scores):
        """激活应急方案"""
        weakest = min(scores, key=scores.get)
        actions = {
            "financial": "寻找替代供应商 + 缩短付款周期",
            "geographic": "启用备用供应商 + 增加安全库存",
            "operational": "增加来料检验频率 + 预留替代产能",
            "compliance": "暂停新订单 + 合规整改审计",
        }
        self.alert(f"供应商 {supplier_id} 风险等级 HIGH: {weakest} 维度得分 {scores[weakest]}")
        self.db.insert("supplier_contingency", {
            "supplier_id": supplier_id,
            "risk_dimension": weakest,
            "action": actions[weakest],
            "activated_at": now()
        })
```

## 异常场景补充

### 场景：需求预测失效

```
触发：突发事件（疫情）→ 历史数据完全失去参考价值
      → 预测准确率 MAPE > 50%
检测：
  1. 最近 7 天 MAPE > 30% → 预测失效告警
  2. 实际销量与预测偏差 > 2 倍 → 异常检测
处理：
  1. 切换到基线预测（最近 7 天移动平均）
  2. 人工调整安全库存倍数（从 1.5 提高到 3.0）
  3. 增加补货频率（从每周改为每天）
预防：预测模型监控 + 自动降级到基线方法
```

### 场景：供应商破产

```
触发：主要供应商突然宣布破产 → 在途订单无法交付
检测：
  1. 供应商破产新闻 → 自动监控
  2. 供应商连续 3 天无法联系 → 告警
处理：
  1. 立即冻结该供应商的所有在途订单
  2. 启动替代供应商紧急采购
  3. 评估受影响 SKU 的安全库存是否充足
  4. 不足 → 紧急调拨 + 替代品推荐
预防：每个 SKU 至少 2 个供应商 + 定期供应商风险评估
```

### 场景：仓库容量溢出

```
触发：促销活动导致库存暴增 → 仓库满载 → 无法收货
检测：
  1. 仓库利用率 > 90% → 告警
  2. 仓库利用率 > 95% → 严重告警
处理：
  1. 暂停非紧急入库
  2. 加速出库：打折促销清理滞销品
  3. 临时租用外部仓库
  4. 调整补货计划：减少补货量 + 延长补货周期
预防：仓库容量预警 + 促销前预评估库存影响
```

## 物流路由优化完整实现

```python
class LogisticsRouteOptimizer:
    """物流路由优化：多约束车辆路径规划"""

    def optimize_routes(self, orders, vehicles, depot):
        """优化配送路线"""
        # 按区域聚类订单
        clusters = self._cluster_by_region(orders, n=len(vehicles))

        routes = []
        for i, cluster in enumerate(clusters):
            vehicle = vehicles[i]
            # TSP 求解：最近邻 + 2-opt 改进
            route = self._solve_tsp(depot, cluster)
            # 检查容量约束
            total_weight = sum(o["weight"] for o in route)
            if total_weight > vehicle["capacity_kg"]:
                # 超容量 → 拆分路线
                sub_routes = self._split_by_capacity(route, vehicle["capacity_kg"])
                routes.extend(sub_routes)
            else:
                routes.append(route)

        return {
            "total_routes": len(routes),
            "total_distance_km": sum(self._calc_distance(r) for r in routes),
            "total_orders": len(orders),
            "avg_orders_per_route": len(orders) / len(routes)
        }

    def _solve_tsp(self, depot, points):
        """2-opt 改进的 TSP 求解"""
        # 初始解：最近邻
        route = [depot]
        remaining = list(points)
        while remaining:
            current = route[-1]
            nearest = min(remaining,
                key=lambda p: self.haversine(current["lat"], current["lng"], p["lat"], p["lng"]))
            route.append(nearest)
            remaining.remove(nearest)
        route.append(depot)  # 返回起点

        # 2-opt 改进
        improved = True
        while improved:
            improved = False
            for i in range(1, len(route) - 2):
                for j in range(i + 1, len(route) - 1):
                    new_route = route[:i] + route[i:j+1][::-1] + route[j+1:]
                    if self._route_distance(new_route) < self._route_distance(route):
                        route = new_route
                        improved = True
        return route

    def reroute_on_delay(self, vehicle_id, current_position, remaining_orders):
        """实时重新路由（遇到交通延误）"""
        # 从当前位置重新规划到剩余订单的最短路径
        return self._solve_tsp(current_position, remaining_orders)
```

## 性能分析详细数据

**仓储操作效率：**

| 操作 | 日处理量 | 延迟 | 准确率 |
|------|---------|------|-------|
| 入库扫描 | 5000 件 | 2s/件 | 99.9% |
| 拣货 | 3000 单 | 3min/单 | 99.5% |
| 包装 | 3000 单 | 2min/单 | 99.8% |
| 出库装车 | 2000 单 | 5min/单 | 99.9% |

**月度成本：**

| 组件 | 规格 | 月成本 |
|------|------|-------|
| WMS 系统 | 8c16G × 3台 | ¥2 万 |
| TMS 系统 | 8c16G × 3台 | ¥2 万 |
| 预测引擎 | GPU 4c + 8c32G | ¥3 万 |
| Kafka + Flink | 6节点 | ¥3 万 |
| MySQL + Redis | 主从 + 3节点 | ¥2 万 |
| **合计** | | **¥12 万** |

## 仓储管理完整实现

```python
class WarehouseManagementService:
    """仓储管理：入库→存储→拣货→出库"""

    def inbound(self, warehouse_id, items):
        """入库流程"""
        receipt_id = str(uuid4())
        for item in items:
            # 1. 分配库位（就近原则：同类商品放相邻库位）
            location = self._allocate_location(warehouse_id, item["sku_id"])
            # 2. 质检
            if item.get("needs_inspection"):
                inspection = self.quality_check(item)
                if not inspection.passed:
                    self.db.insert("inbound_rejection", {
                        "receipt_id": receipt_id, "sku_id": item["sku_id"],
                        "reason": inspection.reason, "rejected_at": now()
                    })
                    continue
            # 3. 上架
            self.db.insert("inventory_locations", {
                "warehouse_id": warehouse_id,
                "sku_id": item["sku_id"],
                "location_code": location["code"],
                "quantity": item["quantity"],
                "batch_id": item.get("batch_id"),
                "stored_at": now()
            })
            # 4. 更新库存
            self.db.execute(
                "UPDATE sku_inventory SET available_qty = available_qty + %s "
                "WHERE sku_id = %s AND warehouse_id = %s",
                item["quantity"], item["sku_id"], warehouse_id)
        return receipt_id

    def pick_order(self, warehouse_id, order_id):
        """拣货：波次拣选优化"""
        items = self.db.get_order_items(order_id)
        # 按库位排序（最短路径）
        items_with_loc = []
        for item in items:
            loc = self.db.query_one(
                "SELECT location_code FROM inventory_locations "
                "WHERE sku_id = %s AND warehouse_id = %s AND quantity >= %s "
                "ORDER BY stored_at ASC LIMIT 1",
                item["sku_id"], warehouse_id, item["quantity"])
            items_with_loc.append({**item, "location": loc["location_code"]})

        # 按库位顺序排列（减少走动距离）
        items_with_loc.sort(key=lambda x: x["location"])

        pick_list = []
        for item in items_with_loc:
            pick_list.append({
                "location": item["location"],
                "sku_id": item["sku_id"],
                "quantity": item["quantity"],
                "picked": False
            })
            # 扣减库存
            self.db.execute(
                "UPDATE inventory_locations SET quantity = quantity - %s "
                "WHERE location_code = %s AND sku_id = %s",
                item["quantity"], item["location"], item["sku_id"])

        return {"order_id": order_id, "pick_list": pick_list,
                "total_items": len(pick_list)}

    def _allocate_location(self, warehouse_id, sku_id):
        """分配库位：同类商品就近存放"""
        # 找到同类商品已有的库位
        existing = self.db.query_one(
            "SELECT location_code FROM inventory_locations "
            "WHERE sku_id = %s AND warehouse_id = %s "
            "AND quantity > 0 ORDER BY location_code LIMIT 1",
            sku_id, warehouse_id)
        if existing:
            return existing  # 放到已有库位旁边

        # 找空库位
        return self.db.query_one(
            "SELECT * FROM warehouse_locations "
            "WHERE warehouse_id = %s AND is_occupied = 0 "
            "ORDER BY aisle, shelf, position LIMIT 1", warehouse_id)
```

## 异常场景补充

### 场景：仓库容量溢出

```python
class WarehouseCapacityGuard:
    """仓库容量保护"""
    def check_before_inbound(self, warehouse_id, additional_pallets):
        capacity = self.db.get_warehouse_capacity(warehouse_id)
        current_usage = self.db.get_warehouse_usage(warehouse_id)
        if current_usage + additional_pallets > capacity * 0.95:
            if current_usage + additional_pallets > capacity:
                raise WarehouseFullError("仓库已满，无法入库")
            self.alert(f"仓库 {warehouse_id} 容量使用率将达 95%+")
```

### 场景：库位冲突（并发拣货）

```
触发：两个订单同时拣同一库位的商品 → 库存超扣
检测：
  1. inventory_locations.quantity < 0 → 数据异常
  2. 乐观锁：UPDATE WHERE quantity >= pick_qty → affected=0 → 冲突
处理：
  1. 拣货操作使用乐观锁：只有库存足够时才能扣减
  2. 冲突时重试：从其他库位拣货
  3. 无其他库位 → 订单部分缺货
预防：库位扣减使用 SELECT FOR UPDATE 或乐观锁
```

### 场景：批次追溯召回

```
触发：某批次商品发现质量问题 → 需要召回
检测：
  1. 质量投诉聚集 → 召回决策
  2. 通过 batch_id 查询所有受影响订单
处理：
  1. 查询该批次的所有出库记录
  2. 通知受影响客户
  3. 在途订单拦截退回
  4. 已签收订单召回退款
预防：批次级追溯 + 质量问题快速定位
```

## 多级库存优化完整实现

```python
class MultiEchelonInventoryOptimizer:
    """多级库存优化：中央仓→区域仓→前置仓"""

    def optimize_replenishment(self):
        """优化各级仓库补货策略"""
        warehouses = self.db.query(
            "SELECT * FROM warehouses ORDER BY level ASC")  # 中央→区域→前置

        for warehouse in warehouses:
            for sku in self.get_active_skus(warehouse["id"]):
                # 计算经济订货量（EOQ）
                daily_demand = self._get_daily_demand(sku["sku_id"], warehouse["id"])
                lead_time_days = self._get_lead_time(warehouse["id"], sku["sku_id"])
                ordering_cost = sku["ordering_cost"]
                holding_cost = sku["holding_cost_per_day"]

                # EOQ = sqrt(2 * D * S / H)
                eoq = math.sqrt(2 * daily_demand * 365 * ordering_cost / holding_cost)

                # 安全库存 = Z * σ * √LT
                demand_stddev = self._get_demand_stddev(sku["sku_id"], warehouse["id"])
                safety_stock = 1.65 * demand_stddev * math.sqrt(lead_time_days)  # Z=1.65 for 95%

                # 再订货点 = 日需求 × 提前期 + 安全库存
                reorder_point = daily_demand * lead_time_days + safety_stock

                # 当前库存
                current = self.db.get_inventory(warehouse["id"], sku["sku_id"])

                if current["available_qty"] <= reorder_point:
                    # 需要补货
                    replenish_qty = eoq
                    self._create_replenish_order(warehouse["id"], sku["sku_id"], replenish_qty)

    def _get_demand_stddev(self, sku_id, warehouse_id):
        """计算需求标准差（最近 90 天）"""
        result = self.db.query_one(
            "SELECT STDDEV(daily_sales) as std FROM sku_daily_sales "
            "WHERE sku_id = %s AND warehouse_id = %s "
            "AND date > CURRENT_DATE - 90", sku_id, warehouse_id)
        return result["std"] or 0
```

## 供应商协同平台

```python
class SupplierCollaborationPortal:
    """供应商协同：订单共享、库存可视、发货通知"""

    def share_purchase_order(self, po_id):
        """向供应商共享采购订单"""
        po = self.db.get_purchase_order(po_id)
        # 生成供应商可见的订单视图（隐藏价格敏感信息）
        supplier_view = {
            "po_id": po["id"],
            "items": [{"sku": item["sku_id"], "name": item["sku_name"],
                       "quantity": item["quantity"], "delivery_date": item["promised_date"]}
                      for item in po["items"]],
            "delivery_address": po["delivery_address"],
            "status": po["status"]
        }
        # 推送到供应商门户
        self.supplier_api.push_order(po["supplier_id"], supplier_view)

    def receive_shipment_notice(self, supplier_id, shipment):
        """接收供应商发货通知（ASN）"""
        asn_id = str(uuid4())
        self.db.insert("advance_shipment_notices", {
            "asn_id": asn_id, "supplier_id": supplier_id,
            "tracking_number": shipment["tracking"],
            "estimated_arrival": shipment["eta"],
            "items": json.dumps(shipment["items"]),
            "status": "in_transit"
        })
        # 更新采购订单状态
        for item in shipment["items"]:
            self.db.update("purchase_order_items",
                {"status": "shipped", "asn_id": asn_id},
                {"po_id": item["po_id"], "sku_id": item["sku_id"]})
```

## 异常场景补充

### 场景：需求预测失效切换

```python
class ForecastFallbackManager:
    """预测失效自动切换"""
    def get_forecast(self, sku_id):
        # 检查模型预测准确率
        mape = self.forecaster.calculate_accuracy(sku_id)
        if mape > 30:
            # 模型失效 → 降级到移动平均
            self.alert(f"SKU {sku_id} 预测MAPE={mape:.0f}%，切换到移动平均")
            return self._moving_average_forecast(sku_id, window=7)
        return self.forecaster.forecast(sku_id)
```

### 场景：供应商破产应急

```
触发：主要供应商突然破产 → 在途订单无法交付
检测：
  1. 供应商连续 3 天无法联系 → 告警
  2. 新闻监控：供应商破产关键词 → 告警
处理：
  1. 冻结该供应商所有在途订单
  2. 启动替代供应商紧急采购
  3. 评估受影响 SKU 安全库存是否充足
  4. 不足 → 紧急调拨 + 替代品推荐
预防：每个 SKU 至少 2 个供应商 + 定期风险评估
```

## 采购订单管理完整实现

```python
class PurchaseOrderService:
    """采购订单管理：审批流 + 收货 + 三单匹配"""

    APPROVAL_THRESHOLDS = [
        {"max_amount": 10000, "approvers": ["manager"]},
        {"max_amount": 100000, "approvers": ["manager", "director"]},
        {"max_amount": float("inf"), "approvers": ["manager", "director", "vp"]},
    ]

    def create_purchase_order(self, supplier_id, items, requester_id):
        """创建采购订单"""
        total = sum(item["unit_price"] * item["quantity"] for item in items)
        po_id = str(uuid4())

        # 确定审批层级
        approvers = self._get_approvers(total)

        self.db.insert("purchase_orders", {
            "po_id": po_id, "supplier_id": supplier_id,
            "requester_id": requester_id,
            "total_amount": total,
            "items": json.dumps(items),
            "status": "pending_approval",
            "required_approvers": json.dumps(approvers),
            "created_at": now()
        })
        return po_id

    def receive_goods(self, po_id, received_items):
        """收货 + 质检"""
        po = self.db.get_purchase_order(po_id)
        po_items = json.loads(po["items"])

        for received in received_items:
            # 匹配采购订单行
            po_line = next(i for i in po_items if i["sku_id"] == received["sku_id"])

            # 质检
            if received.get("quality_check"):
                if not received["quality_check"]["passed"]:
                    self.db.insert("receiving_rejections", {
                        "po_id": po_id, "sku_id": received["sku_id"],
                        "quantity": received["quantity"],
                        "reason": received["quality_check"]["reason"]
                    })
                    continue

            # 入库
            self.warehouse_service.inbound(po["warehouse_id"], [{
                "sku_id": received["sku_id"],
                "quantity": received["quantity"],
                "batch_id": received.get("batch_id")
            }])

            # 记录收货
            self.db.insert("receiving_records", {
                "po_id": po_id, "sku_id": received["sku_id"],
                "ordered_qty": po_line["quantity"],
                "received_qty": received["quantity"],
                "received_at": now()
            })

    def three_way_match(self, po_id, invoice_id):
        """三单匹配：采购单 vs 收货单 vs 发票"""
        po = self.db.get_purchase_order(po_id)
        invoice = self.db.get_invoice(invoice_id)
        receiving = self.db.get_receiving_summary(po_id)

        mismatches = []
        for item in invoice["items"]:
            po_qty = next((i["quantity"] for i in json.loads(po["items"])
                if i["sku_id"] == item["sku_id"]), 0)
            recv_qty = next((r["received_qty"] for r in receiving
                if r["sku_id"] == item["sku_id"]), 0)
            inv_qty = item["quantity"]

            if abs(po_qty - inv_qty) > 0.01:
                mismatches.append({
                    "sku_id": item["sku_id"],
                    "type": "quantity_mismatch",
                    "po_qty": po_qty, "invoice_qty": inv_qty
                })
            if abs(recv_qty - inv_qty) > 0.01:
                mismatches.append({
                    "sku_id": item["sku_id"],
                    "type": "receiving_mismatch",
                    "received_qty": recv_qty, "invoice_qty": inv_qty
                })

        return {"matched": len(mismatches) == 0, "mismatches": mismatches}
```

## 异常场景补充

### 场景：三单匹配失败

```
触发：发票金额与采购单/收货单不一致 → 无法自动付款
检测：
  1. 三单匹配 mismatches > 0 → 需人工处理
处理：
  1. 差异 < 5% → 自动容差通过（记录差异）
  2. 差异 5-20% → 通知采购员确认
  3. 差异 > 20% → 拒绝发票，要求供应商修正
预防：匹配容差策略 + 差异分级处理
```

### 场景：库位调整冲突

```
触发：两个 SKU 同时被分配到同一库位 → 冲突
检测：
  1. 库位占用检查：INSERT 前检查 is_occupied
  2. 乐观锁：UPDATE WHERE version = X
处理：
  1. 冲突时重新分配库位
  2. 库位调整在非业务高峰期执行
预防：库位分配使用 SELECT FOR UPDATE + 批量调整在夜间
```

## 供应链数据分析看板

```python
class SupplyChainAnalytics:
    """供应链数据分析"""

    def get_dashboard(self, date_range=30):
        """供应链看板数据"""
        days = date_range
        return {
            "inventory_turnover": self._inventory_turnover(days),
            "fill_rate": self._fill_rate(days),
            "on_time_delivery": self._on_time_delivery(days),
            "stockout_rate": self._stockout_rate(days),
            "supplier_performance": self._supplier_performance(days),
        }

    def _inventory_turnover(self, days):
        """库存周转率 = 销售成本 / 平均库存"""
        cogs = self.db.query_one(
            "SELECT SUM(cost_of_goods_sold) as total "
            "FROM daily_sales WHERE date > CURRENT_DATE - %s", days)["total"]
        avg_inventory = self.db.query_one(
            "SELECT AVG(daily_inventory_value) as avg "
            "FROM daily_inventory_snapshots "
            "WHERE date > CURRENT_DATE - %s", days)["avg"]
        return round(cogs / avg_inventory, 2) if avg_inventory else 0

    def _fill_rate(self, days):
        """订单满足率 = 按时足额交付订单 / 总订单"""
        total = self.db.count("orders", created_at__gte=now()-timedelta(days=days))
        fulfilled = self.db.count("orders",
            created_at__gte=now()-timedelta(days=days),
            status="delivered_on_time_full")
        return round(fulfilled / total, 3) if total else 0

    def _on_time_delivery(self, days):
        """准时交付率"""
        total = self.db.count("shipments",
            ship_date__gte=now()-timedelta(days=days))
        on_time = self.db.count("shipments",
            ship_date__gte=now()-timedelta(days=days),
            actual_delivery__lte="promised_delivery")
        return round(on_time / total, 3) if total else 0

    def _stockout_rate(self, days):
        """缺货率 = 缺货次数 / 总需求次数"""
        total_demands = self.db.count("demand_events",
            date__gte=now()-timedelta(days=days))
        stockouts = self.db.count("stockout_events",
            date__gte=now()-timedelta(days=days))
        return round(stockouts / total_demands, 3) if total_demands else 0

    def _supplier_performance(self, days):
        """供应商绩效排名"""
        return self.db.query(
            "SELECT s.id, s.name, "
            "COUNT(po.id) as order_count, "
            "AVG(po.on_time) as on_time_rate, "
            "AVG(po.quality_score) as avg_quality, "
            "AVG(po.lead_time_days) as avg_lead_time "
            "FROM suppliers s LEFT JOIN purchase_orders po ON s.id = po.supplier_id "
            "WHERE po.created_at > CURRENT_DATE - %s "
            "GROUP BY s.id ORDER BY on_time_rate DESC", days)
```

**供应链看板 SQL：**

```sql
-- 库存周转率趋势
SELECT DATE_FORMAT(date, '%Y-%m') as month,
    SUM(cost_of_goods_sold) / AVG(daily_inventory_value) as turnover_rate
FROM daily_inventory_snapshots
GROUP BY month ORDER BY month DESC LIMIT 12;

-- 缺货影响分析
SELECT sku_id, sku_name,
    COUNT(*) as stockout_count,
    SUM(lost_revenue) as total_lost_revenue,
    AVG(stockout_duration_hours) as avg_duration
FROM stockout_events
WHERE date > CURRENT_DATE - 30
GROUP BY sku_id ORDER BY total_lost_revenue DESC LIMIT 20;
```

## 异常场景补充

### 场景：库存数据与财务数据不一致

```
触发：库存价值（系统计算）≠ 财务存货科目余额
检测：
  1. 月末对账：库存系统 vs 财务系统
  2. 差异 > 1% → 告警
处理：
  1. 逐笔比对入库/出库记录
  2. 查找时间差（已入库未入账 / 已入账未入库）
  3. 调整分录：差异金额记入"存货差异"科目
预防：库存与财务实时对账 + 入库即入账
```

### 场景：供应商绩效数据延迟

```
触发：供应商交付数据延迟上报 → 绩效看板数据不完整
检测：
  1. 最新绩效数据 > 7 天未更新 → 告警
  2. 看板数据与实际交付记录不匹配
处理：
  1. 从物流系统补充交付数据
  2. 标记延迟数据为"估算"
  3. 供应商补报后修正
预防：多数据源交叉验证 + 交付自动确认
```

## 供应商评估计分卡完整实现

```python
class SupplierScorecardService:
    """供应商评估计分卡"""

    DIMENSIONS = {
        "quality": {"weight": 0.30, "description": "质量"},
        "delivery": {"weight": 0.25, "description": "交付"},
        "cost": {"weight": 0.20, "description": "成本"},
        "responsiveness": {"weight": 0.15, "description": "响应"},
        "sustainability": {"weight": 0.10, "description": "可持续性"},
    }

    TIERS = {
        "gold": {"min_score": 85, "commission_rate": 0.05, "order_priority": 1},
        "silver": {"min_score": 70, "commission_rate": 0.08, "order_priority": 2},
        "bronze": {"min_score": 50, "commission_rate": 0.12, "order_priority": 3},
        "probation": {"min_score": 0, "commission_rate": 0.15, "order_priority": 4},
    }

    def evaluate(self, supplier_id, quarter):
        """季度评估"""
        scores = {}
        for dim, config in self.DIMENSIONS.items():
            raw_score = self._calc_dimension_score(supplier_id, dim, quarter)
            scores[dim] = {
                "raw": raw_score,
                "weighted": raw_score * config["weight"]
            }

        total = sum(s["weighted"] for s in scores.values())
        tier = self._classify_tier(total)

        self.db.insert("supplier_evaluations", {
            "supplier_id": supplier_id,
            "quarter": quarter,
            "quality_score": scores["quality"]["raw"],
            "delivery_score": scores["delivery"]["raw"],
            "cost_score": scores["cost"]["raw"],
            "responsiveness_score": scores["responsiveness"]["raw"],
            "sustainability_score": scores["sustainability"]["raw"],
            "total_score": total,
            "tier": tier,
            "evaluated_at": now()
        })

        return {"supplier_id": supplier_id, "total_score": round(total, 1),
                "tier": tier, "dimensions": scores}

    def _calc_dimension_score(self, supplier_id, dimension, quarter):
        """计算维度得分"""
        if dimension == "quality":
            # 质量得分 = 100 - 退货率 × 100
            reject_rate = self.db.query_one(
                "SELECT AVG(reject_rate) as avg FROM supplier_quality "
                "WHERE supplier_id = %s AND quarter = %s",
                supplier_id, quarter)["avg"] or 0
            return max(0, 100 - reject_rate * 100)
        elif dimension == "delivery":
            # 交付得分 = 准时交付率
            on_time = self.db.query_one(
                "SELECT AVG(CASE WHEN actual_delivery <= promised THEN 1 ELSE 0 END) as rate "
                "FROM purchase_orders WHERE supplier_id = %s AND quarter = %s",
                supplier_id, quarter)["rate"] or 0
            return on_time * 100
        elif dimension == "cost":
            # 成本得分 = 与市场均价比较
            avg_price_ratio = self.db.query_one(
                "SELECT AVG(unit_price / market_avg_price) as ratio "
                "FROM supplier_pricing WHERE supplier_id = %s AND quarter = %s",
                supplier_id, quarter)["ratio"] or 1
            return max(0, min(100, (2 - avg_price_ratio) * 100))

    def _classify_tier(self, score):
        """分级"""
        for tier, config in sorted(self.TIERS.items(),
            key=lambda x: x[1]["min_score"], reverse=True):
            if score >= config["min_score"]:
                return tier
        return "probation"
```

## 需求预测流水线

```python
class DemandForecastPipeline:
    """需求预测：特征工程 + 模型训练 + 准确度追踪"""

    def train_and_forecast(self, sku_id, horizon_days=30):
        """训练模型并预测"""
        # 1. 特征工程
        features = self._engineer_features(sku_id)

        # 2. 训练模型（Prophet）
        from prophet import Prophet
        df = features[["ds", "y"]]
        model = Prophet(
            yearly_seasonality=True,
            weekly_seasonality=True,
            daily_seasonality=False
        )
        # 添加外部特征
        model.add_regressor("is_holiday")
        model.add_regressor("promotion_flag")
        model.add_regressor("temperature")
        model.fit(df)

        # 3. 预测
        future = model.make_future_dataframe(periods=horizon_days)
        forecast = model.predict(future)

        # 4. 计算准确度
        mape = self._calc_mape(sku_id, forecast)

        # 5. 保存预测结果
        for _, row in forecast.tail(horizon_days).iterrows():
            self.db.insert("demand_forecasts", {
                "sku_id": sku_id,
                "forecast_date": row["ds"].date(),
                "predicted_demand": max(0, row["yhat"]),
                "lower_bound": max(0, row["yhat_lower"]),
                "upper_bound": max(0, row["yhat_upper"]),
                "model_mape": mape,
                "created_at": now()
            })

        return {"sku_id": sku_id, "mape": mape, "horizon": horizon_days}

    def _engineer_features(self, sku_id):
        """特征工程"""
        sales = self.db.query(
            "SELECT date, daily_sales, is_holiday, promotion_flag, "
            "temperature FROM sku_daily_features WHERE sku_id = %s "
            "ORDER BY date", sku_id)
        import pandas as pd
        df = pd.DataFrame(sales)
        df = df.rename(columns={"date": "ds", "daily_sales": "y"})
        return df

    def _calc_mape(self, sku_id, forecast):
        """计算 MAPE"""
        actual = self.db.query(
            "SELECT date, daily_sales FROM sku_daily_features "
            "WHERE sku_id = %s AND date < CURRENT_DATE - 7 "
            "ORDER BY date DESC LIMIT 30", sku_id)
        # 比较最近 30 天实际值与预测值
        errors = []
        for a in actual:
            predicted = forecast[forecast["ds"] == a["date"]]["yhat"].values
            if len(predicted) > 0 and a["daily_sales"] > 0:
                errors.append(abs(a["daily_sales"] - predicted[0]) / a["daily_sales"])
        return sum(errors) / len(errors) if errors else 1.0
```

## 异常场景补充

### 场景：预测模型重训练失败

```
触发：Prophet 训练数据异常 → 模型训练报错
检测：
  1. 模型训练任务失败 → 告警
  2. MAPE > 50% → 模型退化
处理：
  1. 降级到移动平均预测
  2. 检查训练数据是否有异常值
  3. 清洗数据后重新训练
预防：训练数据质量检查 + 降级策略 + 自动重试
```

### 场景：单一供应商突然失效

```
触发：供应商工厂火灾 → 完全无法供货 → 影响生产
检测：
  1. 供应商所有订单无法交付 → 严重告警
  2. 供应商 24 小时无响应 → 可能失效
处理：
  1. 立即激活备选供应商
  2. 评估安全库存能否支撑过渡期
  3. 不足 → 紧急采购 + 临时调价
预防：每个 SKU ≥ 2 个供应商 + 供应商风险监控
```

## ABC-XYZ 库存分类完整实现

```python
class InventoryClassifier:
    """ABC-XYZ 库存分类：9 宫格策略"""

    def classify_all(self):
        """全量分类"""
        items = self.db.query(
            "SELECT sku_id, SUM(revenue) as total_revenue, "
            "STDDEV(daily_demand) / NULLIF(AVG(daily_demand), 0) as cv "
            "FROM sku_monthly_stats GROUP BY sku_id")

        # ABC 分类（按收入排序）
        sorted_items = sorted(items, key=lambda x: x["total_revenue"], reverse=True)
        total_revenue = sum(i["total_revenue"] for i in sorted_items)
        cumulative = 0
        for item in sorted_items:
            cumulative += item["total_revenue"]
            ratio = cumulative / total_revenue
            if ratio <= 0.80:
                item["abc"] = "A"
            elif ratio <= 0.95:
                item["abc"] = "B"
            else:
                item["abc"] = "C"

        # XYZ 分类（按变异系数）
        for item in sorted_items:
            cv = item.get("cv") or 999
            if cv <= 0.5:
                item["xyz"] = "X"  # 需求稳定
            elif cv <= 1.0:
                item["xyz"] = "Y"  # 需求波动
            else:
                item["xyz"] = "Z"  # 需求不可预测

        # 更新分类
        for item in sorted_items:
            self.db.update("skus",
                {"abc_class": item["abc"], "xyz_class": item["xyz"],
                 "combined_class": f"{item['abc']}{item['xyz']}"},
                {"id": item["sku_id"]})

        # 统计分布
        matrix = {}
        for item in sorted_items:
            key = f"{item['abc']}{item['xyz']}"
            matrix[key] = matrix.get(key, 0) + 1
        return matrix

    # 9 宫格策略
    STRATEGIES = {
        "AX": "JIT 紧密补货，低安全库存，高频次小批量",
        "AY": "定期补货，中等安全库存，需求预测驱动",
        "AZ": "适度安全库存，关注需求信号，灵活补货",
        "BX": "定期补货，标准安全库存，批量订购",
        "BY": "定期补货，较高安全库存，缓冲期订货",
        "BZ": "较高安全库存，需求监控，谨慎补货",
        "CX": "简单补货，低优先级，长期合同",
        "CY": "批量订购，较高安全库存，低成本优先",
        "CZ": "最大安全库存，或考虑淘汰，按需订购",
    }
```

## 仓库拣货路径优化

```python
class PickPathOptimizer:
    """拣货路径优化：TSP + 波次规划"""

    def optimize_wave(self, warehouse_id, orders):
        """优化一个波次的拣货路径"""
        # 1. 订单分波（按区域）
        zones = self._group_by_zone(warehouse_id, orders)

        results = []
        for zone, zone_orders in zones.items():
            # 2. 合并拣货行（相同 SKU）
            pick_lines = self._merge_pick_lines(zone_orders)

            # 3. 生成拣货路径（TSP）
            locations = [pl["location"] for pl in pick_lines]
            path = self._solve_tsp(locations, warehouse_id)

            # 4. 排序拣货行按路径顺序
            location_order = {loc: i for i, loc in enumerate(path)}
            pick_lines.sort(key=lambda pl: location_order[pl["location"]])

            results.append({
                "zone": zone,
                "pick_lines": pick_lines,
                "path_distance_m": self._calc_path_distance(path),
                "estimated_minutes": len(pick_lines) * 0.5 + len(path) * 0.1
            })

        return results

    def _solve_tsp(self, locations, warehouse_id):
        """TSP 求解（2-opt 近似）"""
        if len(locations) <= 1:
            return locations

        # 获取仓库布局图
        layout = self.db.get_warehouse_layout(warehouse_id)

        # 初始路径：贪心最近邻
        path = [locations[0]]
        remaining = list(locations[1:])
        while remaining:
            current = path[-1]
            nearest = min(remaining,
                key=lambda loc: self._aisle_distance(current, loc, layout))
            path.append(nearest)
            remaining.remove(nearest)

        # 2-opt 优化
        improved = True
        while improved:
            improved = False
            for i in range(1, len(path) - 1):
                for j in range(i + 1, min(i + 10, len(path))):
                    new_path = path[:i] + path[i:j+1][::-1] + path[j+1:]
                    if self._path_cost(new_path, layout) < self._path_cost(path, layout):
                        path = new_path
                        improved = True
        return path
```

## 异常场景补充

### 场景：分类调整引发策略变更

```
触发：月度重分类后 SKU 从 AX 降为 CZ → 安全库存策略突变
      → 大幅减少安全库存 → 可能导致缺货
检测：
  1. 分类变更影响安全库存 > 30% → 告警
  2. 策略从"紧补货"变"高安全库存" → 需要审慎过渡
处理：
  1. 分类变更分阶段执行（不立即切换策略）
  2. 过渡期 2 周：逐步调整安全库存
  3. 监控过渡期缺货率和库存周转
预防：分类变更平滑过渡 + 渐进调整 + 监控
```

### 场景：拣货路径与补货冲突

```
触发：拣货员和补货员在同一通道相遇 → 阻塞
检测：
  1. 同一通道有拣货和补货任务 → 冲突
  2. 拣货延迟 > 预期 2 倍 → 可能被阻塞
处理：
  1. 优先级：拣货 > 补货
  2. 补货任务暂停，等待拣货完成
  3. 长远：补货安排在非拣货高峰期
预防：补货排程避开拣货高峰 + 通道占用通知
```

## 供应链网络可视化完整实现

```python
class SupplyChainNetworkVisualizer:
    """供应链网络可视化：节点 + 边 + 风险叠加"""

    def get_network_graph(self, company_id):
        """获取供应链网络图"""
        # 1. 节点：供应商、仓库、客户
        nodes = []

        suppliers = self.db.query(
            "SELECT id, name, location, risk_score FROM suppliers "
            "WHERE company_id = %s", company_id)
        for s in suppliers:
            nodes.append({
                "id": f"supplier:{s['id']}", "label": s["name"],
                "type": "supplier",
                "location": s["location"],
                "risk_level": self._risk_color(s["risk_score"]),
                "risk_score": s["risk_score"]
            })

        warehouses = self.db.query(
            "SELECT id, name, location, capacity_utilization FROM warehouses "
            "WHERE company_id = %s", company_id)
        for w in warehouses:
            nodes.append({
                "id": f"warehouse:{w['id']}", "label": w["name"],
                "type": "warehouse",
                "location": w["location"],
                "capacity_utilization": w["capacity_utilization"]
            })

        customers = self.db.query(
            "SELECT id, name, region, monthly_volume FROM customers "
            "WHERE company_id = %s", company_id)
        for c in customers:
            nodes.append({
                "id": f"customer:{c['id']}", "label": c["name"],
                "type": "customer",
                "region": c["region"],
                "monthly_volume": c["monthly_volume"]
            })

        # 2. 边：物料流向（粗细=流量）
        edges = []
        flows = self.db.query(
            "SELECT from_type, from_id, to_type, to_id, monthly_volume "
            "FROM supply_chain_flows WHERE company_id = %s", company_id)
        for f in flows:
            edges.append({
                "source": f"{f['from_type']}:{f['from_id']}",
                "target": f"{f['to_type']}:{f['to_id']}",
                "volume": f["monthly_volume"],
                "width": self._volume_to_width(f["monthly_volume"])
            })

        return {"nodes": nodes, "edges": edges}

    def _risk_color(self, score):
        """风险等级颜色"""
        if score >= 80:
            return "red"
        elif score >= 50:
            return "yellow"
        return "green"

    def _volume_to_width(self, volume):
        """流量→线宽"""
        return max(1, min(10, volume / 1000))
```

## 退货管理完整实现

```python
class ReturnsManagementService:
    """退货管理：RMA → 检验 → 处置 → 退款"""

    def create_rma(self, order_id, items, reason):
        """创建退货授权（RMA）"""
        rma_id = str(uuid4())
        # 检查退货政策（7 天无理由 / 15 天质量问题）
        order = self.db.get_order(order_id)
        days_since_delivery = (now() - order["delivered_at"]).days

        for item in items:
            if reason == "quality" and days_since_delivery > 15:
                raise ReturnExpiredError("质量问题退货超过 15 天期限")
            if reason == "change_of_mind" and days_since_delivery > 7:
                raise ReturnExpiredError("无理由退货超过 7 天期限")

        self.db.insert("rma_requests", {
            "rma_id": rma_id, "order_id": order_id,
            "items": json.dumps(items), "reason": reason,
            "status": "approved", "approved_at": now(),
            "return_deadline": now() + timedelta(days=7)  # 7 天内寄回
        })

        # 生成退货物流标签
        label = self.shipping_service.generate_return_label(rma_id, order["address"])

        return {"rma_id": rma_id, "return_label_url": label["url"]}

    def inspect_returned_item(self, rma_id, item_id, inspection):
        """入库检验"""
        checks = {
            "damage_check": inspection.get("has_damage", False),
            "quantity_match": inspection.get("quantity_correct", True),
            "quality_pass": inspection.get("quality_acceptable", True),
            "packaging_intact": inspection.get("packaging_intact", True),
        }

        # 确定处置方案
        if all(checks.values()):
            disposition = "restock"
            refund_rate = 1.0  # 全额退款
        elif checks["damage_check"] and not checks["quality_pass"]:
            disposition = "scrap"
            refund_rate = 1.0  # 质量问题全额退款
        elif not checks["packaging_intact"]:
            disposition = "refurbish"
            refund_rate = 0.85  # 扣 15%
        else:
            disposition = "restock_with_flag"
            refund_rate = 0.9  # 扣 10%

        self.db.insert("return_inspections", {
            "rma_id": rma_id, "item_id": item_id,
            "checks": json.dumps(checks),
            "disposition": disposition,
            "refund_rate": refund_rate,
            "inspected_at": now()
        })

        # 执行处置
        if disposition == "restock":
            self.warehouse.restock(item_id, condition="new")
        elif disposition == "restock_with_flag":
            self.warehouse.restock(item_id, condition="open_box")
        elif disposition == "refurbish":
            self.warehouse.send_to_refurbish(item_id)
        elif disposition == "scrap":
            self.warehouse.mark_scrap(item_id)

        # 计算退款
        original_price = self.db.get_order_item(item_id)["price"]
        refund_amount = original_price * refund_rate

        return {"disposition": disposition, "refund_amount": refund_amount}
```

## 供应链成本模拟

```python
class SupplyChainSimulator:
    """供应链成本模拟：What-If 分析"""

    def simulate_scenario(self, base_scenario, changes):
        """模拟场景变更影响"""
        result = {"baseline": self._calc_metrics(base_scenario), "impact": {}}

        for change in changes:
            modified = self._apply_change(base_scenario, change)
            modified_metrics = self._calc_metrics(modified)
            delta = {
                k: round(modified_metrics[k] - result["baseline"][k], 2)
                for k in modified_metrics
            }
            result["impact"][change["description"]] = delta

        return result

    def _calc_metrics(self, scenario):
        """计算供应链指标"""
        # 安全库存
        safety_stock = sum(
            self._calc_safety_stock(item) for item in scenario["items"])

        # 订单满足率
        fill_rate = self._estimate_fill_rate(scenario)

        # 总成本
        total_cost = (
            safety_stock * scenario["avg_unit_cost"] * 0.25 +  # 持有成本 25%
            scenario.get("expedited_shipping_cost", 0) +
            scenario.get("lost_sales_cost", 0)
        )

        return {
            "safety_stock_units": safety_stock,
            "fill_rate": fill_rate,
            "holding_cost": round(safety_stock * scenario["avg_unit_cost"] * 0.25, 2),
            "total_cost": round(total_cost, 2)
        }

    def _apply_change(self, scenario, change):
        """应用变更"""
        modified = dict(scenario)
        if change["type"] == "lead_time_increase":
            for item in modified["items"]:
                if item["supplier_id"] == change["supplier_id"]:
                    item["lead_time_days"] *= (1 + change["percentage"] / 100)
        elif change["type"] == "demand_surge":
            for item in modified["items"]:
                item["daily_demand"] *= (1 + change["percentage"] / 100)
        elif change["type"] == "supplier_failure":
            modified["items"] = [
                item for item in modified["items"]
                if item["supplier_id"] != change["supplier_id"]]
        return modified

    def monte_carlo(self, scenario, iterations=1000):
        """蒙特卡洛模拟"""
        results = []
        for _ in range(iterations):
            # 随机波动需求
            noisy_scenario = dict(scenario)
            for item in noisy_scenario["items"]:
                item["daily_demand"] *= max(0.5, random.gauss(1.0, 0.2))
            results.append(self._calc_metrics(noisy_scenario))

        # 统计
        fill_rates = [r["fill_rate"] for r in results]
        return {
            "fill_rate_p50": round(sorted(fill_rates)[500], 3),
            "fill_rate_p90": round(sorted(fill_rates)[900], 3),
            "avg_total_cost": round(statistics.mean(r["total_cost"] for r in results), 2),
            "worst_case_cost": round(max(r["total_cost"] for r in results), 2)
        }
```

## 异常场景补充

### 场景：供应商质量回归

```
触发：已通过审批的供应商质量突然下降 → 退货率飙升
检测：
  1. 近 30 天退货率 > 5% → 质量回归
  2. 同一供应商多个 SKU 退货 → 系统性问题
处理：
  1. 暂停该供应商新订单
  2. 紧急质量审核
  3. 审核不通过 → 降级或淘汰
  4. 审核通过 → 恢复但加强检验
预防：供应商质量持续监控 + 退货率告警 + 自动暂停
```

### 场景：模拟模型假设失效

```
触发：需求分布从正态变为长尾 → 模拟结果不准确
检测：
  1. 实际安全库存缺货率 ≠ 模拟预测 → 假设失效
  2. 需求分布拟合检验 p < 0.05 → 不符假设
处理：
  1. 重新拟合需求分布
  2. 更新模拟参数
  3. 重新运行模拟
预防：定期验证需求分布 + 自动重新拟合
```

## 供应链事件溯源完整实现

```python
class SupplyChainEventStore:
    """供应链事件溯源：追加日志 + 状态投影"""

    EVENT_TYPES = [
        "order_placed", "order_confirmed", "shipment_dispatched",
        "shipment_in_transit", "goods_received", "quality_inspected",
        "goods_accepted", "goods_rejected", "invoice_received",
        "payment_authorized", "payment_settled",
    ]

    def append_event(self, entity_type, entity_id, event_type, payload):
        """追加事件"""
        if event_type not in self.EVENT_TYPES:
            raise UnknownEventError(f"未知事件类型: {event_type}")

        event_id = str(uuid4())
        # 获取当前版本号（乐观锁）
        current_version = self._get_current_version(entity_type, entity_id)

        self.db.insert("supply_chain_events", {
            "event_id": event_id,
            "entity_type": entity_type,  # order / shipment / invoice
            "entity_id": entity_id,
            "event_type": event_type,
            "event_version": current_version + 1,
            "payload": json.dumps(payload),
            "timestamp": now(),
            "correlation_id": payload.get("correlation_id")
        })

        # 更新投影（CQRS 读模型）
        self._update_projection(entity_type, entity_id, event_type, payload)

        return event_id

    def get_entity_state(self, entity_type, entity_id):
        """获取实体当前状态（从投影读取）"""
        return self.db.query_one(
            f"SELECT * FROM {entity_type}_projection WHERE id = %s",
            entity_id)

    def rebuild_projection(self, entity_type, entity_id):
        """重建投影（从事件重放）"""
        events = self.db.query(
            "SELECT * FROM supply_chain_events "
            "WHERE entity_type = %s AND entity_id = %s "
            "ORDER BY event_version ASC",
            entity_type, entity_id)

        # 清除旧投影
        self.db.execute(
            f"DELETE FROM {entity_type}_projection WHERE id = %s",
            entity_id)

        # 重放事件
        state = {}
        for event in events:
            state = self._apply_event(state, event["event_type"],
                json.loads(event["payload"]))

        # 写入新投影
        if state:
            state["id"] = entity_id
            self.db.insert(f"{entity_type}_projection", state)

        return state

    def _apply_event(self, state, event_type, payload):
        """应用事件到状态"""
        if event_type == "order_placed":
            return {
                "status": "placed",
                "supplier_id": payload["supplier_id"],
                "items": payload["items"],
                "total_amount": payload["total_amount"],
                "placed_at": payload["timestamp"]
            }
        elif event_type == "order_confirmed":
            state["status"] = "confirmed"
            state["confirmed_at"] = payload["timestamp"]
            state["estimated_delivery"] = payload.get("estimated_delivery")
        elif event_type == "shipment_dispatched":
            state["status"] = "dispatched"
            state["tracking_number"] = payload["tracking_number"]
            state["dispatched_at"] = payload["timestamp"]
        elif event_type == "goods_received":
            state["status"] = "received"
            state["received_at"] = payload["timestamp"]
        elif event_type == "payment_settled":
            state["status"] = "settled"
            state["settled_at"] = payload["timestamp"]
            state["payment_reference"] = payload["reference"]
        return state
```

## 碳足迹追踪

```python
class CarbonFootprintTracker:
    """碳足迹追踪：运输排放 + SKU 碳强度 + ESG 报告"""

    EMISSION_FACTORS = {
        "air": 0.602,    # kgCO2/ton-km
        "sea": 0.008,
        "road": 0.062,
        "rail": 0.022,
    }

    def calculate_shipment_carbon(self, shipment_id):
        """计算运输碳排放"""
        shipment = self.db.get_shipment(shipment_id)
        distance_km = shipment["distance_km"]
        weight_tons = shipment["weight_kg"] / 1000
        transport_mode = shipment["transport_mode"]
        factor = self.EMISSION_FACTORS.get(transport_mode, 0.062)

        carbon_kg = distance_km * weight_tons * factor

        self.db.insert("carbon_emissions", {
            "shipment_id": shipment_id,
            "transport_mode": transport_mode,
            "distance_km": distance_km,
            "weight_tons": weight_tons,
            "emission_factor": factor,
            "carbon_kg": round(carbon_kg, 2),
            "calculated_at": now()
        })

        return {"shipment_id": shipment_id, "carbon_kg": round(carbon_kg, 2)}

    def get_sku_carbon_intensity(self, sku_id):
        """获取 SKU 碳强度"""
        total_carbon = self.db.query_one(
            "SELECT SUM(ce.carbon_kg) as total FROM carbon_emissions ce "
            "JOIN shipment_items si ON ce.shipment_id = si.shipment_id "
            "WHERE si.sku_id = %s", sku_id)["total"] or 0
        total_units = self.db.query_one(
            "SELECT SUM(quantity) as total FROM shipment_items "
            "WHERE sku_id = %s", sku_id)["total"] or 1

        return {
            "sku_id": sku_id,
            "carbon_per_unit_kg": round(total_carbon / total_units, 4),
            "total_carbon_kg": round(total_carbon, 2),
            "total_units": total_units
        }

    def generate_esg_report(self, year):
        """生成 ESG 碳排放报告"""
        monthly = self.db.query(
            "SELECT DATE_FORMAT(calculated_at, '%%Y-%%m') as month, "
            "SUM(carbon_kg) as total, transport_mode "
            "FROM carbon_emissions WHERE YEAR(calculated_at) = %s "
            "GROUP BY month, transport_mode ORDER BY month", year)

        return {
            "year": year,
            "total_carbon_tons": round(
                sum(m["total"] for m in monthly) / 1000, 1),
            "by_month": monthly,
            "target_reduction_pct": 20,  # 年减排 20% 目标
        }
```

## 异常场景补充

### 场景：事件存储损坏

```
触发：事件存储部分数据丢失 → 状态重建不完整 → 投影错误
检测：
  1. 事件版本号不连续 → 数据缺失
  2. 投影状态与实际不一致 → 损坏
处理：
  1. 从事件备份恢复缺失事件
  2. 无备份 → 从当前状态反推（降级）
  3. 重建受影响实体的投影
预防：事件存储备份 + 版本连续性检查 + 定期投影校验
```

### 场景：供应商门户数据同步失败

```
触发：供应商提交 ASN 后数据未同步到内部系统 → 无法收货确认
检测：
  1. ASN 提交成功但内部系统无记录 → 同步失败
  2. 同步延迟 > 10 分钟 → 告警
处理：
  1. 重试同步（幂等）
  2. 仍失败 → 手动录入 ASN 数据
  3. 检查同步接口异常
预防：同步重试 + 死信队列 + 供应商自助查询同步状态
```

## 供应商协作门户完整实现

```python
class SupplierPortalService:
    """供应商协作门户：PO 共享 + ASN + 发票 + 质量文档"""

    def share_purchase_order(self, po_id, supplier_id):
        """共享采购订单给供应商（只读视图）"""
        po = self.db.get_purchase_order(po_id)

        # 生成供应商只读访问令牌
        token = str(uuid4())
        self.db.insert("supplier_portal_access", {
            "token": token,
            "supplier_id": supplier_id,
            "resource_type": "purchase_order",
            "resource_id": po_id,
            "access_level": "read_only",
            "expires_at": now() + timedelta(days=30)
        })

        # 通知供应商
        supplier = self.db.get_supplier(supplier_id)
        self.email_service.send(supplier["contact_email"],
            f"新采购订单 {po['po_number']}",
            f"请查看: /portal/po/{token}")

        return {"token": token}

    def submit_asn(self, supplier_id, asn_data):
        """供应商提交提前发货通知（ASN）"""
        asn_id = str(uuid4())
        # 验证 ASN 对应有效的 PO
        po = self.db.query_one(
            "SELECT * FROM purchase_orders WHERE id = %s "
            "AND supplier_id = %s AND status = 'confirmed'",
            asn_data["po_id"], supplier_id)
        if not po:
            raise InvalidASNError("无对应的已确认采购订单")

        self.db.insert("advance_shipping_notices", {
            "asn_id": asn_id,
            "po_id": asn_data["po_id"],
            "supplier_id": supplier_id,
            "items": json.dumps(asn_data["items"]),
            "estimated_arrival": asn_data["estimated_arrival"],
            "carrier": asn_data.get("carrier"),
            "tracking_number": asn_data.get("tracking_number"),
            "status": "submitted",
            "submitted_at": now()
        })

        # 同步到内部系统
        self._sync_asn_to_internal(asn_id)

        return {"asn_id": asn_id}

    def submit_invoice(self, supplier_id, invoice_data):
        """供应商提交发票"""
        invoice_id = str(uuid4())

        # 三单匹配（PO + ASN + Invoice）
        match_result = self._three_way_match(invoice_data)

        self.db.insert("supplier_invoices", {
            "invoice_id": invoice_id,
            "supplier_id": supplier_id,
            "po_id": invoice_data["po_id"],
            "asn_id": invoice_data.get("asn_id"),
            "invoice_number": invoice_data["invoice_number"],
            "amount": invoice_data["amount"],
            "items": json.dumps(invoice_data["items"]),
            "match_status": match_result["status"],
            "match_details": json.dumps(match_result),
            "status": "submitted",
            "submitted_at": now()
        })

        if match_result["status"] == "matched":
            # 自动安排付款
            self.payment_service.schedule_payment(invoice_id, invoice_data["amount"])
        elif match_result["status"] == "discrepancy":
            # 不匹配 → 人工审核
            self._escalate_discrepancy(invoice_id, match_result)

        return {"invoice_id": invoice_id, "match_status": match_result["status"]}

    def _three_way_match(self, invoice_data):
        """三单匹配"""
        po = self.db.get_purchase_order(invoice_data["po_id"])
        asn = self.db.query_one(
            "SELECT * FROM advance_shipping_notices "
            "WHERE po_id = %s AND status = 'received'",
            invoice_data["po_id"])

        discrepancies = []

        # 金额匹配
        if abs(invoice_data["amount"] - po["total_amount"]) > 0.01:
            discrepancies.append({
                "type": "amount_mismatch",
                "invoice": invoice_data["amount"],
                "po": po["total_amount"]
            })

        # 数量匹配
        for item in invoice_data["items"]:
            po_item = next((i for i in json.loads(po["items"])
                          if i["sku_id"] == item["sku_id"]), None)
            if po_item and item["quantity"] != po_item["quantity"]:
                discrepancies.append({
                    "type": "quantity_mismatch",
                    "sku_id": item["sku_id"],
                    "invoice_qty": item["quantity"],
                    "po_qty": po_item["quantity"]
                })

        if discrepancies:
            return {"status": "discrepancy", "discrepancies": discrepancies}
        return {"status": "matched"}
```

## 异常场景补充

### 场景：ASN 数据同步失败

```
触发：供应商提交 ASN 后内部系统未收到 → 无法收货确认
检测：
  1. ASN 状态为 submitted 但内部系统无记录 → 同步失败
  2. 同步延迟 > 10 分钟 → 告警
处理：
  1. 重试同步（幂等：基于 asn_id）
  2. 仍失败 → 手动录入 ASN 数据
  3. 检查同步接口和网络
预防：同步重试 + 死信队列 + 供应商可查看同步状态
```

### 场景：三单匹配异常

```
触发：发票金额与 PO 不一致 → 匹配失败 → 付款延迟 → 供应商不满
检测：
  1. 匹配状态为 discrepancy → 异常
  2. 付款延迟 > 7 天 → 供应商投诉
处理：
  1. 自动通知采购员审核差异
  2. 小额差异（< 5%）→ 自动批准
  3. 大额差异 → 人工审核 + 供应商沟通
预防：容差自动批准 + 及时审核 SLA + 供应商门户查看匹配状态
```

## 供应商绩效记分卡完整实现

```python
class SupplierScorecardService:
    """供应商记分卡：5 维度评分 + 分级 + 改进计划"""

    DIMENSIONS = {
        "delivery_on_time": {"weight": 0.30, "description": "准时交付率"},
        "quality_acceptance": {"weight": 0.25, "description": "质量合格率"},
        "price_competitiveness": {"weight": 0.20, "description": "价格竞争力"},
        "responsiveness": {"weight": 0.15, "description": "响应速度"},
        "compliance": {"weight": 0.10, "description": "合规性"},
    }

    TIERS = {
        "preferred": {"min_score": 85, "benefits": "优先分配订单，享受快速付款"},
        "standard": {"min_score": 70, "benefits": "正常合作"},
        "probation": {"min_score": 0, "benefits": "限制新订单，需提交改进计划"},
    }

    def calculate_score(self, supplier_id, period="monthly"):
        """计算供应商综合得分"""
        scores = {}
        # 准时交付率
        on_time = self.db.query_one(
            "SELECT "
            "SUM(CASE WHEN actual_delivery <= promised_delivery THEN 1 ELSE 0 END) "
            "/ COUNT(*) as rate FROM shipments WHERE supplier_id = %s "
            "AND promised_delivery > NOW() - INTERVAL 1 MONTH", supplier_id)
        scores["delivery_on_time"] = (on_time["rate"] or 0) * 100

        # 质量合格率
        quality = self.db.query_one(
            "SELECT "
            "SUM(CASE WHEN inspection_result = 'accepted' THEN 1 ELSE 0 END) "
            "/ COUNT(*) as rate FROM quality_inspections "
            "WHERE supplier_id = %s AND inspected_at > NOW() - INTERVAL 1 MONTH",
            supplier_id)
        scores["quality_acceptance"] = (quality["rate"] or 0) * 100

        # 价格竞争力（与市场均价比较）
        avg_price = self.db.query_one(
            "SELECT AVG(unit_price) as avg FROM market_prices "
            "WHERE category = (SELECT category FROM suppliers WHERE id = %s)",
            supplier_id)
        our_price = self.db.query_one(
            "SELECT AVG(unit_price) as avg FROM purchase_orders "
            "WHERE supplier_id = %s AND created_at > NOW() - INTERVAL 1 MONTH",
            supplier_id)
        if avg_price["avg"] and our_price["avg"]:
            price_ratio = our_price["avg"] / avg_price["avg"]
            scores["price_competitiveness"] = min(100, max(0, (2 - price_ratio) * 50))
        else:
            scores["price_competitiveness"] = 70

        # 响应速度（询价响应时间）
        avg_response_hours = self.db.query_one(
            "SELECT AVG(TIMESTAMPDIFF(HOUR, rfq_sent_at, quote_received_at)) as avg "
            "FROM rfq_responses WHERE supplier_id = %s "
            "AND rfq_sent_at > NOW() - INTERVAL 1 MONTH", supplier_id)
        if avg_response_hours["avg"]:
            scores["responsiveness"] = max(0, 100 - avg_response_hours["avg"] * 2)
        else:
            scores["responsiveness"] = 70

        # 合规性
        compliance_issues = self.db.count("compliance_violations",
            supplier_id=supplier_id,
            created_at__gte=now()-timedelta(days=30))
        scores["compliance"] = max(0, 100 - compliance_issues * 20)

        # 加权综合得分
        overall = sum(scores[dim] * self.DIMENSIONS[dim]["weight"]
                     for dim in self.DIMENSIONS)

        # 分级
        tier = "probation"
        for t, config in sorted(self.TIERS.items(),
                                key=lambda x: x[1]["min_score"], reverse=True):
            if overall >= config["min_score"]:
                tier = t
                break

        # 保存
        self.db.insert("supplier_scorecards", {
            "supplier_id": supplier_id,
            "period": period,
            "scores": json.dumps(scores),
            "overall_score": round(overall, 1),
            "tier": tier,
            "calculated_at": now()
        })

        # 降级处理
        if tier == "probation":
            self._trigger_improvement_plan(supplier_id)

        return {"overall": round(overall, 1), "tier": tier, "scores": scores}

    def _trigger_improvement_plan(self, supplier_id):
        """触发供应商改进计划"""
        self.db.insert("supplier_improvement_plans", {
            "supplier_id": supplier_id,
            "status": "required",
            "deadline": now() + timedelta(days=30),
            "created_at": now()
        })
        self.notify_supplier(supplier_id,
            "您的绩效评分低于 70 分，请在 30 天内提交改进计划")
```

## 供应链中断响应手册

```python
class SupplyChainDisruptionPlaybook:
    """供应链中断响应手册"""

    DISRUPTION_TYPES = {
        "supplier_bankruptcy": {"severity": "critical", "rto_days": 14},
        "natural_disaster": {"severity": "high", "rto_days": 7},
        "geopolitical": {"severity": "high", "rto_days": 10},
        "pandemic": {"severity": "critical", "rto_days": 21},
        "quality_recall": {"severity": "medium", "rto_days": 5},
    }

    def handle_disruption(self, disruption_type, affected_suppliers, details):
        """处理供应链中断"""
        playbook = self.DISRUPTION_TYPES[disruption_type]

        # 1. 影响评估
        impact = self._assess_impact(affected_suppliers)

        # 2. 响应策略
        strategies = []
        for supplier_id in affected_suppliers:
            # 查找替代供应商
            alternatives = self._find_alternative_suppliers(supplier_id)
            if alternatives:
                strategies.append({
                    "action": "switch_supplier",
                    "from": supplier_id,
                    "to": alternatives[0]["id"],
                    "estimated_transition_days": alternatives[0]["onboarding_days"]
                })
            else:
                # 无替代 → 使用库存缓冲
                buffer_days = self._check_inventory_buffer(supplier_id)
                if buffer_days < playbook["rto_days"]:
                    strategies.append({
                        "action": "demand_shaping",
                        "supplier": supplier_id,
                        "buffer_days": buffer_days,
                        "shortfall_days": playbook["rto_days"] - buffer_days
                    })

        # 3. 通知利益相关方
        self._notify_stakeholders(impact, strategies)

        # 4. 执行响应
        for strategy in strategies:
            if strategy["action"] == "switch_supplier":
                self._activate_alternative(strategy)

        return {"impact": impact, "strategies": strategies,
                "rto_target_days": playbook["rto_days"]}
```

## 异常场景补充

### 场景：供应商记分卡被操纵

```
触发：供应商准时交付率 95% 但实际服务质量差 → 赶工交付但质量低
检测：
  1. 准时率高 + 质量合格率低 → 赶工现象
  2. 客户投诉率与供应商评分不一致 → 评分被操纵
处理：
  1. 调整评分权重（增加质量权重，降低交付权重）
  2. 加入客户满意度维度
  3. 人工审核高分但低质量供应商
预防：多维评分 + 客户满意度 + 异常组合检测
```

### 场景：中断响应手册过时

```
触发：新型中断（芯片短缺）→ 手册无对应策略 → 响应延迟
检测：
  1. 中断类型不在手册中 → 需要临时决策
  2. 替代供应商信息过时 → 联系不上
处理：
  1. 紧急召开供应链委员会
  2. 制定临时策略
  3. 事后更新手册
预防：手册季度更新 + 供应商信息定期验证 + 演练
```

## 供应链金融完整实现

```python
class SupplyChainFinanceService:
    """供应链金融：应收账款融资 + 信用传递 + 风控"""

    def apply_financing(self, supplier_id, receivable_id, amount):
        """申请应收账款融资"""
        # 1. 验证应收账款
        receivable = self.db.get_receivable(receivable_id)
        if not receivable or receivable["supplier_id"] != supplier_id:
            raise InvalidReceivableError("应收账款无效")

        if receivable["status"] != "confirmed":
            raise InvalidReceivableError("应收账款未确认")

        if receivable["amount"] < amount:
            raise InvalidReceivableError("融资金额超过应收金额")

        # 2. 供应商信用评估
        credit = self._evaluate_supplier_credit(supplier_id)

        # 3. 核心企业信用传递
        core_enterprise = receivable["payer_id"]
        core_rating = self._get_core_enterprise_rating(core_enterprise)

        # 4. 融资利率计算
        base_rate = 0.04  # 基础利率 4%
        risk_premium = (1 - credit["score"] / 100) * 0.03
        rate = base_rate + risk_premium

        # 5. 融资期限
        days_to_maturity = (receivable["due_date"] - now()).days
        if days_to_maturity < 30:
            raise FinancingError("应收账款到期日不足 30 天")

        # 6. 创建融资
        financing_id = str(uuid4())
        interest = amount * rate * days_to_maturity / 365

        self.db.insert("financing_applications", {
            "financing_id": financing_id,
            "supplier_id": supplier_id,
            "receivable_id": receivable_id,
            "core_enterprise_id": core_enterprise,
            "amount": amount,
            "interest": round(interest, 2),
            "rate": round(rate, 4),
            "days_to_maturity": days_to_maturity,
            "credit_score": credit["score"],
            "core_rating": core_rating,
            "status": "approved" if credit["score"] >= 60 else "rejected",
            "created_at": now()
        })

        return {
            "financing_id": financing_id,
            "amount": amount,
            "interest": round(interest, 2),
            "rate": f"{rate*100:.2f}%",
            "net_amount": round(amount - interest, 2),
            "status": "approved" if credit["score"] >= 60 else "rejected"
        }

    def _evaluate_supplier_credit(self, supplier_id):
        """评估供应商信用"""
        scorecard = self.db.query_one(
            "SELECT overall_score FROM supplier_scorecards "
            "WHERE supplier_id = %s ORDER BY calculated_at DESC LIMIT 1",
            supplier_id)

        score = scorecard["overall_score"] if scorecard else 50

        # 检查违约历史
        defaults = self.db.count("financing_defaults",
            supplier_id=supplier_id)
        score -= defaults * 15

        return {"score": max(0, min(100, score))}
```

## 异常场景补充

### 场景：应收账款重复融资

```
触发：同一应收账款被多家金融机构融资 → 重复融资 → 风险暴露
检测：
  1. 应收账款 ID 已有融资记录 → 重复
  2. 同一发票号多笔融资 → 异常
处理：
  1. 拒绝重复融资申请
  2. 通知已有融资方
  3. 调查是否有欺诈
预防：应收账款唯一约束 + 区块链存证 + 跨机构共享
```

### 场景：核心企业信用违约

```
触发：核心企业（付款方）违约 → 供应链金融资金链断裂
检测：
  1. 核心企业评级下调 → 风险
  2. 核心企业逾期付款 → 严重
处理：
  1. 触发信用保险理赔
  2. 冻结新增融资
  3. 追偿已有融资
预防：信用保险 + 多核心企业分散 + 额度管控
```

## 仓储管理完整实现

```python
class WarehouseManagementService:
    """仓储管理：入库 → 库位 → 拣货 → 出库"""

    def inbound_putaway(self, warehouse_id, sku_id, quantity, batch_id):
        """入库上架"""
        # 1. 查找最优库位
        location = self._find_optimal_location(warehouse_id, sku_id)

        if not location:
            raise NoAvailableLocationError("无可用库位")

        # 2. 更新库存
        self.db.insert("inventory_transactions", {
            "transaction_id": str(uuid4()),
            "warehouse_id": warehouse_id,
            "sku_id": sku_id,
            "location_id": location["id"],
            "batch_id": batch_id,
            "quantity": quantity,
            "type": "inbound",
            "created_at": now()
        })

        # 3. 更新库位库存
        self.redis.hincrby(
            f"location_inventory:{warehouse_id}:{location['id']}",
            sku_id, quantity)

        # 4. 更新总库存
        self.redis.hincrby(f"warehouse_inventory:{warehouse_id}", sku_id, quantity)

        return {"location": location["code"], "quantity": quantity}

    def _find_optimal_location(self, warehouse_id, sku_id):
        """查找最优库位（基于 SKU 特性 + 库位规则）"""
        sku = self.db.get_sku(sku_id)

        # 周转率高的 SKU → 靠近出库口的库位
        if sku["turnover_rate"] == "high":
            zone = "A"  # 靠近出库口
        elif sku["turnover_rate"] == "medium":
            zone = "B"
        else:
            zone = "C"  # 远离出库口

        # 查找空位
        locations = self.db.query(
            "SELECT * FROM warehouse_locations "
            "WHERE warehouse_id = %s AND zone = %s "
            "AND current_capacity < max_capacity "
            "ORDER BY current_capacity ASC LIMIT 5",
            warehouse_id, zone)

        return locations[0] if locations else None

    def create_pick_wave(self, warehouse_id, orders, strategy="batch"):
        """创建拣货波次"""
        wave_id = str(uuid4())

        if strategy == "batch":
            # 批量拣货：合并多个订单的相同 SKU
            pick_items = self._batch_picking(orders)
        elif strategy == "zone":
            # 分区拣货：按库区分配拣货员
            pick_items = self._zone_picking(orders, warehouse_id)
        else:
            # 单订单拣货
            pick_items = self._single_order_picking(orders)

        # 优化拣货路径（TSP 问题近似解）
        optimized_path = self._optimize_pick_path(pick_items)

        self.db.insert("pick_waves", {
            "wave_id": wave_id,
            "warehouse_id": warehouse_id,
            "order_count": len(orders),
            "pick_item_count": len(pick_items),
            "strategy": strategy,
            "status": "pending",
            "created_at": now()
        })

        return {"wave_id": wave_id, "pick_items": len(pick_items),
                "estimated_time_minutes": len(pick_items) * 0.5}

    def _batch_picking(self, orders):
        """批量拣货（合并相同 SKU）"""
        sku_quantities = {}
        for order in orders:
            for item in order["items"]:
                key = item["sku_id"]
                sku_quantities[key] = sku_quantities.get(key, 0) + item["quantity"]

        return [{"sku_id": sku, "total_quantity": qty}
                for sku, qty in sku_quantities.items()]
```

## 异常场景补充

### 场景：库位分配冲突

```
触发：两个入库任务分配到同一库位 → 库存数据不一致
检测：
  1. 库位实际容量 > 最大容量 → 超载
  2. 库存记录与实际盘点不一致 → 分配冲突
处理：
  1. 入库时加锁（库位级别乐观锁）
  2. 冲突时重新分配库位
  3. 定期盘点修正
预防：库位锁 + 分配后验证 + 定期盘点
```

### 场景：拣货路径低效

```
触发：批量拣货路径未优化 → 拣货员来回走动 → 效率低下
检测：
  1. 单波次拣货时间 > 预估 2 倍 → 路径低效
  2. 拣货员步数统计异常 → 路径问题
处理：
  1. 优化路径算法（贪心 + 2-opt）
  2. 调整库位分配策略（高频 SKU 靠近出库口）
  3. 增加分区拣货
预防：路径优化算法 + 高频 SKU 前置 + 分区拣货
```

## 供应链碳排放追踪完整实现

```python
class CarbonFootprintTracker:
    """供应链碳排放追踪：产品碳足迹 + 运输排放 + 碳中和"""

    EMISSION_FACTORS = {
        "truck": 0.062,    # kgCO2/ton-km
        "rail": 0.022,
        "air": 0.602,
        "sea": 0.008,
        "warehouse": 0.035,  # kgCO2/sqm-day
    }

    def calculate_product_footprint(self, product_id):
        """计算产品全生命周期碳足迹"""
        product = self.db.get_product(product_id)

        # 1. 原材料碳排放
        materials = self.db.query(
            "SELECT * FROM bill_of_materials WHERE product_id = %s",
            product_id)
        material_emissions = sum(
            m["quantity_kg"] * self._get_material_emission_factor(m["material_type"])
            for m in materials)

        # 2. 制造碳排放
        manufacturing = self._get_manufacturing_emissions(product_id)

        # 3. 运输碳排放
        transport = self._get_transport_emissions(product_id)

        # 4. 仓储碳排放
        storage = self._get_storage_emissions(product_id)

        total = material_emissions + manufacturing + transport + storage

        return {
            "product_id": product_id,
            "total_kg_co2e": round(total, 2),
            "breakdown": {
                "materials_kg": round(material_emissions, 2),
                "manufacturing_kg": round(manufacturing, 2),
                "transport_kg": round(transport, 2),
                "storage_kg": round(storage, 2),
            }
        }

    def _get_transport_emissions(self, product_id):
        """计算运输碳排放"""
        shipments = self.db.query(
            "SELECT s.* FROM shipments s "
            "JOIN order_items oi ON s.order_id = oi.order_id "
            "WHERE oi.product_id = %s", product_id)

        total = 0
        for s in shipments:
            factor = self.EMISSION_FACTORS.get(s["transport_mode"], 0.062)
            distance_km = s.get("distance_km", 0)
            weight_tons = s.get("weight_kg", 0) / 1000
            total += factor * distance_km * weight_tons

        return total

    def generate_carbon_report(self, period="monthly"):
        """生成碳排放报告"""
        # 按运输方式统计
        by_mode = {}
        for mode, factor in self.EMISSION_FACTORS.items():
            if mode == "warehouse":
                continue
            emissions = self.db.query_one(
                "SELECT SUM(distance_km * weight_kg / 1000 * %s) as total "
                "FROM shipments WHERE transport_mode = %s "
                "AND shipped_at > NOW() - INTERVAL 1 MONTH",
                factor, mode)
            by_mode[mode] = round(emissions["total"] or 0, 2)

        # 碳中和进度
        total_emissions = sum(by_mode.values())
        offset_credits = self.db.query_one(
            "SELECT SUM(credits_kg) as total FROM carbon_offsets "
            "WHERE purchased_at > NOW() - INTERVAL 1 MONTH")["total"] or 0

        return {
            "period": period,
            "total_emissions_kg": round(total_emissions, 2),
            "by_transport_mode": by_mode,
            "carbon_offsets_kg": round(offset_credits, 2),
            "net_emissions_kg": round(total_emissions - offset_credits, 2),
            "neutrality_pct": round(min(100, offset_credits / max(total_emissions, 1) * 100), 1)
        }
```

## 异常场景补充

### 场景：碳排放数据不准确

```
触发：运输距离使用直线距离而非实际路线 → 碳排放低估 30%
检测：
  1. 排放因子计算的距离与 GPS 轨迹距离差异 > 20% → 不准确
  2. 审计发现计算方法偏差
处理：
  1. 使用实际运输路线距离
  2. 校准历史数据
  3. 重新发布碳排放报告
预防：使用 GPS 轨迹距离 + 定期校准 + 第三方审计
```

### 场景：碳信用交易欺诈

```
触发：购买的碳信用来自无效项目 → 碳中和声明虚假 → 绿洗风险
检测：
  1. 碳信用价格远低于市场价 → 可能无效
  2. 碳信用项目未被认证机构验证 → 可疑
处理：
  1. 只购买经认证的碳信用（Gold Standard, VCS）
  2. 验证碳信用项目真实性
  3. 替换无效碳信用
预防：只购买认证碳信用 + 项目验证 + 多源采购
```

## 供应链风险预警完整实现

```python
class SupplyChainRiskAlertService:
    """供应链风险预警：供应商风险 + 运输风险 + 地缘风险"""

    RISK_CATEGORIES = {
        "supplier_default": {"weight": 0.3, "threshold": 70},
        "transport_disruption": {"weight": 0.25, "threshold": 60},
        "geopolitical": {"weight": 0.2, "threshold": 50},
        "quality_issue": {"weight": 0.15, "threshold": 60},
        "price_volatility": {"weight": 0.1, "threshold": 70},
    }

    def calculate_supplier_risk(self, supplier_id):
        """计算供应商综合风险分数"""
        scores = {}

        # 1. 财务风险（信用评级变化）
        credit = self.db.query_one(
            "SELECT credit_rating, rating_trend FROM supplier_credit "
            "WHERE supplier_id = %s ORDER BY assessed_at DESC LIMIT 1",
            supplier_id)
        if credit:
            rating_score = {"AAA": 10, "AA": 20, "A": 30, "BBB": 50, "BB": 70, "B": 85, "CCC": 95}
            scores["financial"] = rating_score.get(credit["credit_rating"], 50)
            if credit["rating_trend"] == "downgrading":
                scores["financial"] += 15

        # 2. 交付风险（准时率）
        on_time_rate = self.db.query_one(
            "SELECT AVG(CASE WHEN actual_delivery <= promised_delivery THEN 1 ELSE 0 END) as rate "
            "FROM purchase_orders WHERE supplier_id = %s "
            "AND created_at > NOW() - INTERVAL 90 DAY", supplier_id)["rate"] or 1.0
        scores["delivery"] = round((1 - on_time_rate) * 100)

        # 3. 质量风险（不良品率）
        defect_rate = self.db.query_one(
            "SELECT AVG(defect_rate) as avg FROM quality_inspections "
            "WHERE supplier_id = %s AND inspected_at > NOW() - INTERVAL 90 DAY",
            supplier_id)["avg"] or 0
        scores["quality"] = round(min(100, defect_rate * 1000))

        # 4. 集中度风险（依赖度）
        total_spend = self.db.query_one(
            "SELECT SUM(amount) as total FROM purchase_orders "
            "WHERE supplier_id = %s AND created_at > NOW() - INTERVAL 365 DAY",
            supplier_id)["total"] or 0
        category_spend = self.db.query_one(
            "SELECT SUM(amount) as total FROM purchase_orders "
            "WHERE category = (SELECT category FROM suppliers WHERE id = %s) "
            "AND created_at > NOW() - INTERVAL 365 DAY", supplier_id)["total"] or 1
        concentration = total_spend / category_spend
        scores["concentration"] = round(min(100, concentration * 100))

        # 综合风险
        overall = (scores.get("financial", 0) * 0.3 +
                  scores.get("delivery", 0) * 0.25 +
                  scores.get("quality", 0) * 0.25 +
                  scores.get("concentration", 0) * 0.2)

        level = "critical" if overall >= 70 else "high" if overall >= 50 else "medium" if overall >= 30 else "low"

        return {"supplier_id": supplier_id, "overall_risk": round(overall, 1),
                "level": level, "scores": scores}

    def generate_mitigation_plan(self, supplier_id, risk_result):
        """生成风险缓解方案"""
        plan = []
        scores = risk_result["scores"]

        if scores.get("financial", 0) > 50:
            plan.append({"action": "寻找备选供应商", "priority": "high",
                "reason": "财务风险较高，需降低依赖度"})

        if scores.get("delivery", 0) > 40:
            plan.append({"action": "增加安全库存", "priority": "high",
                "reason": f"交付准时率仅 {100-scores['delivery']}%"})

        if scores.get("concentration", 0) > 60:
            plan.append({"action": "分散采购比例", "priority": "medium",
                "reason": "供应商集中度风险过高"})

        if scores.get("quality", 0) > 30:
            plan.append({"action": "加强来料检验", "priority": "medium",
                "reason": "质量不良率偏高"})

        return {"supplier_id": supplier_id, "plan": plan}
```

## 异常场景补充

### 场景：风险预警过于频繁

```
触发：风险模型灵敏度过高 → 每天数十条预警 → 团队忽视 → 真正风险被遗漏
检测：
  1. 预警数量 > 20 条/天 → 过于频繁
  2. 预警处理率 < 30% → 预警疲劳
处理：
  1. 提高风险阈值
  2. 预警分级（只推送 high/critical）
  3. 合并相似预警
预防：阈值调整 + 分级推送 + 预警去重
```

### 场景：备选供应商无法及时替代

```
触发：主力供应商风险上升 → 触发寻找备选 → 备选供应商认证需 3 个月 → 期间断供
检测：
  1. 无认证备选供应商 → 替代风险
  2. 关键物料只有 1 家供应商 → 单一来源风险
处理：
  1. 预认证备选供应商（常备 2-3 家）
  2. 紧急采购通道（缩短认证流程）
  3. 增加安全库存缓冲
预防：预认证备选 + 紧急通道 + 安全库存
```

## 供应链库存优化完整实现

```python
class InventoryOptimizationService:
    """库存优化：安全库存 + 补货点 + EOQ + ABC 分类"""

    ABC_CATEGORIES = {
        "A": {"revenue_pct": 80, "sku_pct": 20, "service_level": 0.99, "review_cycle_days": 1},
        "B": {"revenue_pct": 15, "sku_pct": 30, "service_level": 0.95, "review_cycle_days": 7},
        "C": {"revenue_pct": 5, "sku_pct": 50, "service_level": 0.90, "review_cycle_days": 30},
    }

    def classify_abc(self, warehouse_id):
        """ABC 分类（按销售额）"""
        sku_sales = self.db.query(
            "SELECT sku_id, SUM(quantity * unit_price) as revenue "
            "FROM order_items oi "
            "JOIN orders o ON oi.order_id = o.id "
            "WHERE o.warehouse_id = %s "
            "AND o.created_at > NOW() - INTERVAL 90 DAY "
            "GROUP BY sku_id ORDER BY revenue DESC", warehouse_id)

        total_revenue = sum(s["revenue"] for s in sku_sales) or 1
        cumulative = 0
        classifications = {}

        for s in sku_sales:
            cumulative += s["revenue"]
            pct = cumulative / total_revenue

            if pct <= 0.80:
                classifications[s["sku_id"]] = "A"
            elif pct <= 0.95:
                classifications[s["sku_id"]] = "B"
            else:
                classifications[s["sku_id"]] = "C"

            self.db.upsert("sku_classifications", {
                "sku_id": s["sku_id"], "warehouse_id": warehouse_id,
                "category": classifications[s["sku_id"]],
                "revenue_pct": round(s["revenue"] / total_revenue, 4),
                "classified_at": now()
            }, conflict_columns=["sku_id", "warehouse_id"])

        counts = {"A": 0, "B": 0, "C": 0}
        for c in classifications.values():
            counts[c] += 1

        return {"warehouse_id": warehouse_id, "counts": counts, "total_skus": len(sku_sales)}

    def calculate_safety_stock(self, sku_id, warehouse_id, service_level=None):
        """计算安全库存"""
        category = self._get_category(sku_id, warehouse_id)
        if service_level is None:
            service_level = self.ABC_CATEGORIES[category]["service_level"]

        # 历史需求统计
        demand_history = self.db.query(
            "SELECT DATE(created_at) as date, SUM(quantity) as demand "
            "FROM order_items oi JOIN orders o ON oi.order_id = o.id "
            "WHERE oi.sku_id = %s AND o.warehouse_id = %s "
            "AND o.created_at > NOW() - INTERVAL 90 DAY "
            "GROUP BY DATE(created_at) ORDER BY date", sku_id, warehouse_id)

        if len(demand_history) < 14:
            return {"safety_stock": 0, "reason": "数据不足"}

        demands = [d["demand"] for d in demand_history]
        avg_daily_demand = statistics.mean(demands)
        std_daily_demand = statistics.stdev(demands) if len(demands) > 1 else 0

        # 供应商提前期统计
        lead_times = self.db.query(
            "SELECT DATEDIFF(actual_delivery, order_date) as days "
            "FROM purchase_orders "
            "WHERE sku_id = %s AND warehouse_id = %s "
            "AND status = 'received' "
            "AND actual_delivery IS NOT NULL "
            "ORDER BY actual_delivery DESC LIMIT 20", sku_id, warehouse_id)

        if not lead_times:
            return {"safety_stock": 0, "reason": "无提前期数据"}

        avg_lead_time = statistics.mean(lt["days"] for lt in lead_times)
        std_lead_time = statistics.stdev(lt["days"] for lt in lead_times) if len(lead_times) > 1 else 0

        # 安全库存 = z * sqrt(L * σd² + d² * σL²)
        z_score = {0.90: 1.28, 0.95: 1.65, 0.99: 2.33}.get(service_level, 1.65)

        safety_stock = z_score * math.sqrt(
            avg_lead_time * std_daily_demand**2 +
            avg_daily_demand**2 * std_lead_time**2
        )

        # 补货点 = 提前期需求 + 安全库存
        reorder_point = avg_daily_demand * avg_lead_time + safety_stock

        return {
            "sku_id": sku_id,
            "safety_stock": round(safety_stock, 0),
            "reorder_point": round(reorder_point, 0),
            "avg_daily_demand": round(avg_daily_demand, 1),
            "avg_lead_time_days": round(avg_lead_time, 1),
            "service_level": service_level
        }

    def calculate_eoq(self, sku_id, warehouse_id):
        """计算经济订货量（EOQ）"""
        annual_demand = self.db.query_one(
            "SELECT SUM(quantity) as total FROM order_items oi "
            "JOIN orders o ON oi.order_id = o.id "
            "WHERE oi.sku_id = %s AND o.warehouse_id = %s "
            "AND o.created_at > NOW() - INTERVAL 365 DAY",
            sku_id, warehouse_id)["total"] or 0

        sku = self.db.get_sku(sku_id)
        unit_cost = sku.get("unit_cost", 0)
        ordering_cost = sku.get("ordering_cost", 50)  # 每次订货成本
        holding_cost_pct = sku.get("holding_cost_pct", 0.25)  # 持有成本比例

        if annual_demand == 0 or unit_cost == 0:
            return {"eoq": 0, "reason": "需求或成本为零"}

        holding_cost = unit_cost * holding_cost_pct

        # EOQ = sqrt(2 * D * S / H)
        eoq = math.sqrt(2 * annual_demand * ordering_cost / holding_cost)

        # 订货次数 = D / EOQ
        order_frequency = annual_demand / eoq

        # 总成本
        total_cost = ordering_cost * order_frequency + holding_cost * eoq / 2

        return {
            "sku_id": sku_id,
            "eoq": round(eoq, 0),
            "annual_demand": annual_demand,
            "order_frequency_per_year": round(order_frequency, 1),
            "total_annual_cost": round(total_cost, 2),
            "ordering_cost_per_order": ordering_cost,
            "holding_cost_per_unit": round(holding_cost, 2)
        }
```

## 异常场景补充

### 场景：安全库存计算基于异常数据

```
触发：疫情期间需求暴增 → 安全库存基于异常期数据 → 正常期库存过高 → 资金占用
检测：
  1. 安全库存 > 历史平均库存 3 倍 → 数据异常
  2. 计算时段内需求方差极大 → 异常波动
处理：
  1. 排除异常期数据（使用 12 个月正常期数据）
  2. 手动设置安全库存上限
  3. 加入异常检测标记
预防：排除异常期 + 上限约束 + 异常标记
```

### 场景：ABC 分类未定期更新

```
触发：SKU 分类基于 90 天数据 → 新爆款商品仍被分类为 C → 库存策略不当 → 缺货
检测：
  1. C 类商品销售额占比 > 10% → 分类过时
  2. 缺货商品集中在 C 类 → 分类不当
处理：
  1. 加速分类更新（90 天 → 30 天）
  2. 新商品前 30 天默认 B 类
  3. 销量突增自动升级分类
预防：缩短分类周期 + 新品默认 B 类 + 自动升级
```

## 供应链需求预测完整实现

```python
class SupplyChainDemandForecastService:
    """需求预测：时间序列 + 季节性 + 促销修正 + 置信区间"""

    FORECAST_METHODS = {
        "moving_average": "移动平均",
        "exponential_smoothing": "指数平滑",
        "seasonal_decomposition": "季节分解",
        "arima": "ARIMA",
    }

    def forecast_demand(self, sku_id, warehouse_id, forecast_days=30,
                       method="exponential_smoothing"):
        """预测需求"""
        # 1. 获取历史需求数据
        history = self.db.query(
            "SELECT DATE(created_at) as date, SUM(quantity) as demand "
            "FROM order_items oi JOIN orders o ON oi.order_id = o.id "
            "WHERE oi.sku_id = %s AND o.warehouse_id = %s "
            "AND o.created_at > NOW() - INTERVAL 365 DAY "
            "GROUP BY DATE(created_at) ORDER BY date",
            sku_id, warehouse_id)

        if len(history) < 30:
            return {"status": "insufficient_data", "data_points": len(history)}

        demands = [h["demand"] for h in history]

        # 2. 选择预测方法
        if method == "moving_average":
            forecast = self._moving_average(demands, forecast_days, window=7)

        elif method == "exponential_smoothing":
            forecast = self._exponential_smoothing(demands, forecast_days, alpha=0.3)

        elif method == "seasonal_decomposition":
            forecast = self._seasonal_decomposition(demands, forecast_days)

        else:
            forecast = self._exponential_smoothing(demands, forecast_days, alpha=0.3)

        # 3. 促销修正（如有计划促销 → 提升预测值）
        promotions = self._get_upcoming_promotions(sku_id, forecast_days)
        if promotions:
            for i, day in enumerate(forecast):
                for promo in promotions:
                    if promo["start_day"] <= i <= promo["end_day"]:
                        forecast[i]["demand"] = round(
                            forecast[i]["demand"] * promo["lift_factor"], 0)

        # 4. 计算置信区间
        residuals = [demands[i] - self._exponential_smoothing_point(
            demands[:i+1], alpha=0.3) for i in range(7, len(demands))]
        std_error = statistics.stdev(residuals) if len(residuals) > 1 else 0

        for day in forecast:
            day["lower_bound"] = max(0, round(day["demand"] - 1.96 * std_error, 0))
            day["upper_bound"] = round(day["demand"] + 1.96 * std_error, 0)

        # 5. 汇总
        total_forecast = sum(d["demand"] for d in forecast)
        avg_daily = total_forecast / forecast_days

        return {
            "sku_id": sku_id,
            "warehouse_id": warehouse_id,
            "method": method,
            "forecast_days": forecast_days,
            "total_forecast": round(total_forecast, 0),
            "avg_daily_demand": round(avg_daily, 1),
            "daily_forecast": forecast,
            "confidence": round(1 - std_error / max(avg_daily, 1), 2),
            "history_data_points": len(demands)
        }

    def _moving_average(self, demands, forecast_days, window=7):
        """移动平均预测"""
        avg = statistics.mean(demands[-window:])
        return [{"day_offset": i + 1, "demand": round(avg, 0)}
                for i in range(forecast_days)]

    def _exponential_smoothing(self, demands, forecast_days, alpha=0.3):
        """指数平滑预测"""
        smoothed = [demands[0]]
        for d in demands[1:]:
            smoothed.append(alpha * d + (1 - alpha) * smoothed[-1])

        # 预测值 = 最后一个平滑值
        forecast_value = smoothed[-1]
        return [{"day_offset": i + 1, "demand": round(forecast_value, 0)}
                for i in range(forecast_days)]

    def _exponential_smoothing_point(self, demands, alpha=0.3):
        """单点指数平滑（用于计算残差）"""
        if not demands:
            return 0
        smoothed = demands[0]
        for d in demands[1:]:
            smoothed = alpha * d + (1 - alpha) * smoothed
        return smoothed

    def _seasonal_decomposition(self, demands, forecast_days):
        """季节分解预测"""
        n = len(demands)
        if n < 14:
            return [{"day_offset": i + 1, "demand": round(statistics.mean(demands), 0)}
                    for i in range(forecast_days)]

        # 计算周季节因子
        weekly_pattern = [0] * 7
        weekly_counts = [0] * 7
        for i, d in enumerate(demands):
            day_of_week = i % 7
            weekly_pattern[day_of_week] += d
            weekly_counts[day_of_week] += 1

        avg_demand = statistics.mean(demands)
        seasonal_factors = [weekly_pattern[i] / max(weekly_counts[i], 1) / max(avg_demand, 1)
                          for i in range(7)]

        # 趋势（线性回归）
        x = list(range(n))
        x_mean = statistics.mean(x)
        y_mean = statistics.mean(demands)
        numerator = sum((x[i] - x_mean) * (demands[i] - y_mean) for i in range(n))
        denominator = sum((x[i] - x_mean) ** 2 for i in range(n))
        slope = numerator / max(denominator, 1)
        intercept = y_mean - slope * x_mean

        forecast = []
        for i in range(forecast_days):
            trend = intercept + slope * (n + i)
            seasonal = seasonal_factors[(n + i) % 7]
            forecast.append({"day_offset": i + 1,
                           "demand": round(max(0, trend * seasonal), 0)})

        return forecast

    def _get_upcoming_promotions(self, sku_id, days_ahead):
        """获取即将到来的促销"""
        return self.db.query(
            "SELECT *, DATEDIFF(start_date, NOW()) as start_day, "
            "DATEDIFF(end_date, NOW()) as end_day "
            "FROM promotions "
            "WHERE sku_id = %s AND end_date > NOW() "
            "AND start_date < NOW() + INTERVAL %s DAY",
            sku_id, days_ahead)
```

## 异常场景补充

### 场景：需求预测严重偏差

```
触发：预测日均需求 100 件 → 实际日均 500 件 → 缺货 → 预测模型失效
检测：
  1. 实际需求超出预测上界 → 预测偏差
  2. 预测 MAPE（平均绝对百分比误差）> 30% → 模型需改进
处理：
  1. 更换预测方法（如从移动平均改为 ARIMA）
  2. 加入更多特征（促销、天气、节假日）
  3. 增加安全库存缓冲
预防：多方法对比 + 特征增强 + 安全库存缓冲
```

### 场景：季节性模式变化

```
触发：往年夏季需求高峰 → 今年因市场变化变为冬季高峰 → 季节模型失效 → 库存积压
检测：
  1. 实际需求与季节模型预测方向相反 → 模式变化
  2. 连续 2 个月预测偏差 > 50% → 需要更新模型
处理：
  1. 缩短历史数据窗口（3 年 → 1 年）
  2. 加大近期数据权重
  3. 模型定期重新训练
预防：短窗口 + 近期权重 + 定期重训练
```

## 供应链库存安全水位完整实现

```python
import math
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from typing import Optional, Dict, List, Tuple
from enum import Enum
from collections import defaultdict


class ReplenishmentStatus(Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    IN_TRANSIT = "in_transit"
    RECEIVED = "received"
    CANCELLED = "cancelled"


class StockLevelStatus(Enum):
    ABOVE_SAFETY = "above_safety"
    AT_SAFETY = "at_safety"
    BELOW_SAFETY = "below_safety"
    CRITICAL = "critical"


@dataclass
class DemandRecord:
    """历史需求记录"""
    date: datetime
    quantity: float


@dataclass
class LeadTimeRecord:
    """历史补货前置时间记录"""
    order_date: datetime
    received_date: datetime
    lead_time_days: float


@dataclass
class WarehouseStock:
    """仓库库存信息"""
    warehouse_id: str
    sku: str
    current_stock: float
    safety_stock: float = 0.0
    reorder_point: float = 0.0
    in_transit_quantity: float = 0.0
    average_daily_demand: float = 0.0
    average_lead_time_days: float = 0.0
    demand_std_dev: float = 0.0
    lead_time_std_dev: float = 0.0
    service_level: float = 0.95


@dataclass
class ReplenishmentOrder:
    """补货订单"""
    order_id: str
    warehouse_id: str
    sku: str
    quantity: float
    status: ReplenishmentStatus = ReplenishmentStatus.PENDING
    created_at: datetime = field(default_factory=datetime.utcnow)
    expected_delivery: Optional[datetime] = None
    confirmed_at: Optional[datetime] = None


@dataclass
class SafetyStockConfig:
    """安全库存配置"""
    sku: str
    default_service_level: float = 0.95
    review_period_days: int = 7
    min_safety_stock: float = 0.0
    max_safety_stock: float = float("inf")
    safety_stock_multiplier: float = 1.0  # 动态调整乘数


class SafetyStockService:
    """供应链库存安全水位服务
    
    基于需求变异性和前置时间变异性计算安全库存水位,
    在库存低于安全水位时触发补货, 支持多仓库安全库存分配。
    """

    # 服务水平对应的 Z 值 (标准正态分布分位数)
    Z_SCORES = {
        0.90: 1.28,
        0.92: 1.41,
        0.95: 1.65,
        0.97: 1.88,
        0.98: 2.05,
        0.99: 2.33,
        0.995: 2.58,
        0.999: 3.09,
    }

    def __init__(self):
        self._warehouse_stocks: Dict[str, Dict[str, WarehouseStock]] = defaultdict(dict)
        self._configs: Dict[str, SafetyStockConfig] = {}
        self._demand_history: Dict[str, List[DemandRecord]] = defaultdict(list)
        self._lead_time_history: Dict[str, List[LeadTimeRecord]] = defaultdict(list)
        self._replenishment_orders: Dict[str, ReplenishmentOrder] = {}
        self._pending_replenishment: Dict[str, Dict[str, float]] = defaultdict(dict)

    def _get_z_score(self, service_level: float) -> float:
        """根据服务水平获取 Z 值"""
        if service_level in self.Z_SCORES:
            return self.Z_SCORES[service_level]
        # 线性插值
        sorted_levels = sorted(self.Z_SCORES.keys())
        for i in range(len(sorted_levels) - 1):
            if sorted_levels[i] <= service_level <= sorted_levels[i + 1]:
                ratio = (
                    (service_level - sorted_levels[i])
                    / (sorted_levels[i + 1] - sorted_levels[i])
                )
                return (
                    self.Z_SCORES[sorted_levels[i]] * (1 - ratio)
                    + self.Z_SCORES[sorted_levels[i + 1]] * ratio
                )
        # 超出范围取最近值
        if service_level < sorted_levels[0]:
            return self.Z_SCORES[sorted_levels[0]]
        return self.Z_SCORES[sorted_levels[-1]]

    def _calculate_std_dev(self, values: List[float]) -> float:
        """计算标准差"""
        if len(values) < 2:
            return 0.0
        mean = sum(values) / len(values)
        variance = sum((x - mean) ** 2 for x in values) / (len(values) - 1)
        return math.sqrt(variance)

    def _calculate_average(self, values: List[float]) -> float:
        """计算平均值"""
        if not values:
            return 0.0
        return sum(values) / len(values)

    def register_warehouse_stock(
        self,
        warehouse_id: str,
        sku: str,
        current_stock: float,
        service_level: float = 0.95,
    ):
        """注册仓库库存"""
        stock = WarehouseStock(
            warehouse_id=warehouse_id,
            sku=sku,
            current_stock=current_stock,
            service_level=service_level,
        )
        self._warehouse_stocks[warehouse_id][sku] = stock
        if sku not in self._configs:
            self._configs[sku] = SafetyStockConfig(
                sku=sku, default_service_level=service_level
            )

    def add_demand_record(self, sku: str, date: datetime, quantity: float):
        """添加需求历史记录"""
        self._demand_history[sku].append(
            DemandRecord(date=date, quantity=quantity)
        )

    def add_lead_time_record(
        self,
        sku: str,
        order_date: datetime,
        received_date: datetime,
    ):
        """添加前置时间历史记录"""
        lead_time_days = (received_date - order_date).days
        self._lead_time_history[sku].append(
            LeadTimeRecord(
                order_date=order_date,
                received_date=received_date,
                lead_time_days=lead_time_days,
            )
        )

    def calculate_safety_stock(
        self,
        warehouse_id: str,
        sku: str,
        service_level: Optional[float] = None,
    ) -> float:
        """计算安全库存水位
        
        使用经典安全库存公式:
        SS = Z * sqrt(LT * sigma_d^2 + d^2 * sigma_lt^2)
        
        其中:
        - Z: 服务水平对应的标准正态分位数
        - LT: 平均前置时间 (天)
        - sigma_d: 日需求标准差
        - d: 平均日需求
        - sigma_lt: 前置时间标准差 (天)
        
        Args:
            warehouse_id: 仓库 ID
            sku: 商品 SKU
            service_level: 目标服务水平 (可选, 默认使用配置值)
            
        Returns:
            安全库存数量
            
        Raises:
            KeyError: 仓库或 SKU 不存在
        """
        if warehouse_id not in self._warehouse_stocks:
            raise KeyError(f"Warehouse {warehouse_id} not found")
        if sku not in self._warehouse_stocks[warehouse_id]:
            raise KeyError(f"SKU {sku} not found in warehouse {warehouse_id}")

        stock = self._warehouse_stocks[warehouse_id][sku]
        config = self._configs.get(sku)

        # 确定服务水平
        target_service_level = service_level
        if target_service_level is None:
            target_service_level = (
                config.default_service_level if config else stock.service_level
            )

        # 计算日需求统计量
        demand_quantities = [
            r.quantity for r in self._demand_history.get(sku, [])
        ]
        avg_daily_demand = self._calculate_average(demand_quantities)
        demand_std_dev = self._calculate_std_dev(demand_quantities)

        # 计算前置时间统计量
        lead_times = [
            r.lead_time_days for r in self._lead_time_history.get(sku, [])
        ]
        avg_lead_time = self._calculate_average(lead_times)
        lead_time_std_dev = self._calculate_std_dev(lead_times)

        # 如果没有历史数据, 使用库存对象中的预估值
        if avg_daily_demand == 0 and stock.average_daily_demand > 0:
            avg_daily_demand = stock.average_daily_demand
        if demand_std_dev == 0 and stock.demand_std_dev > 0:
            demand_std_dev = stock.demand_std_dev
        if avg_lead_time == 0 and stock.average_lead_time_days > 0:
            avg_lead_time = stock.average_lead_time_days
        if lead_time_std_dev == 0 and stock.lead_time_std_dev > 0:
            lead_time_std_dev = stock.lead_time_std_dev

        # 计算安全库存
        z = self._get_z_score(target_service_level)
        safety_stock = z * math.sqrt(
            avg_lead_time * demand_std_dev ** 2
            + avg_daily_demand ** 2 * lead_time_std_dev ** 2
        )

        # 应用动态调整乘数
        if config and config.safety_stock_multiplier != 1.0:
            safety_stock *= config.safety_stock_multiplier

        # 应用上下限约束
        if config:
            safety_stock = max(config.min_safety_stock, safety_stock)
            safety_stock = min(config.max_safety_stock, safety_stock)

        # 更新仓库库存信息
        stock.safety_stock = safety_stock
        stock.reorder_point = safety_stock + avg_daily_demand * avg_lead_time
        stock.average_daily_demand = avg_daily_demand
        stock.average_lead_time_days = avg_lead_time
        stock.demand_std_dev = demand_std_dev
        stock.lead_time_std_dev = lead_time_std_dev
        stock.service_level = target_service_level

        return safety_stock

    def check_stock_levels(
        self, warehouse_id: str, sku: str
    ) -> Tuple[StockLevelStatus, float, float]:
        """检查库存水位状态
        
        Args:
            warehouse_id: 仓库 ID
            sku: 商品 SKU
            
        Returns:
            (status, current_stock, safety_stock): 水位状态, 当前库存, 安全库存
        """
        if warehouse_id not in self._warehouse_stocks:
            raise KeyError(f"Warehouse {warehouse_id} not found")
        if sku not in self._warehouse_stocks[warehouse_id]:
            raise KeyError(f"SKU {sku} not found in warehouse {warehouse_id}")

        stock = self._warehouse_stocks[warehouse_id][sku]

        # 确保 safety_stock 已计算
        if stock.safety_stock == 0:
            self.calculate_safety_stock(warehouse_id, sku)

        effective_stock = stock.current_stock + stock.in_transit_quantity
        safety = stock.safety_stock

        if effective_stock <= safety * 0.5:
            status = StockLevelStatus.CRITICAL
        elif effective_stock < safety:
            status = StockLevelStatus.BELOW_SAFETY
        elif effective_stock <= safety * 1.05:
            status = StockLevelStatus.AT_SAFETY
        else:
            status = StockLevelStatus.ABOVE_SAFETY

        return status, stock.current_stock, stock.safety_stock

    def trigger_replenishment(
        self,
        warehouse_id: str,
        sku: str,
        quantity: Optional[float] = None,
    ) -> Optional[ReplenishmentOrder]:
        """触发补货订单
        
        当库存低于安全水位时自动触发。补货量 = 安全库存 * 2 - 当前库存 - 在途数量,
        确保补货后库存回到安全水位以上。
        
        Args:
            warehouse_id: 仓库 ID
            sku: 商品 SKU
            quantity: 指定补货量 (可选, 默认自动计算)
            
        Returns:
            ReplenishmentOrder 或 None (库存充足无需补货)
        """
        status, current_stock, safety_stock = self.check_stock_levels(
            warehouse_id, sku
        )

        if status == StockLevelStatus.ABOVE_SAFETY:
            return None

        stock = self._warehouse_stocks[warehouse_id][sku]

        # 检查是否已有在途补货
        pending_key = f"{warehouse_id}:{sku}"
        pending_qty = self._pending_replenishment.get(pending_key, {}).get(
            "total_pending", 0.0
        )

        effective_stock = current_stock + pending_qty
        if effective_stock >= safety_stock:
            return None  # 已有足够在途补货

        # 计算补货量: 补到安全水位的 2 倍
        target_stock = safety_stock * 2
        replenish_qty = target_stock - effective_stock

        if quantity is not None:
            replenish_qty = quantity

        # 生成订单
        now = datetime.utcnow()
        order_id = (
            f"RO-{warehouse_id}-{sku}-{now.strftime('%Y%m%d%H%M%S')}"
        )
        expected_delivery = now + timedelta(
            days=stock.average_lead_time_days or 7
        )

        order = ReplenishmentOrder(
            order_id=order_id,
            warehouse_id=warehouse_id,
            sku=sku,
            quantity=replenish_qty,
            status=ReplenishmentStatus.PENDING,
            created_at=now,
            expected_delivery=expected_delivery,
        )

        self._replenishment_orders[order_id] = order
        self._pending_replenishment[pending_key] = {
            "total_pending": pending_qty + replenish_qty,
            "last_order_id": order_id,
        }
        stock.in_transit_quantity = pending_qty + replenish_qty

        return order

    def allocate_safety_stock(
        self,
        sku: str,
        total_safety_stock: float,
        warehouse_weights: Optional[Dict[str, float]] = None,
    ) -> Dict[str, float]:
        """多仓库安全库存分配
        
        根据各仓库的需求占比或指定权重分配总安全库存。
        支持基于需求量的比例分配和自定义权重分配两种模式。
        
        Args:
            sku: 商品 SKU
            total_safety_stock: 需要分配的总安全库存量
            warehouse_weights: 仓库权重字典 (可选)
                - 若提供, 按权重分配
                - 若不提供, 按各仓库平均日需求量比例分配
            
        Returns:
            Dict[warehouse_id, allocated_safety_stock]: 各仓库分配量
            
        Raises:
            ValueError: 总安全库存为负或权重总和为 0
        """
        if total_safety_stock < 0:
            raise ValueError(
                f"Total safety stock cannot be negative: {total_safety_stock}"
            )

        # 收集所有持有该 SKU 的仓库
        relevant_warehouses: Dict[str, WarehouseStock] = {}
        for wh_id, skus in self._warehouse_stocks.items():
            if sku in skus:
                relevant_warehouses[wh_id] = skus[sku]

        if not relevant_warehouses:
            return {}

        # 计算分配权重
        if warehouse_weights:
            # 验证权重
            total_weight = sum(
                warehouse_weights.get(wh_id, 0.0)
                for wh_id in relevant_warehouses
            )
            if total_weight <= 0:
                raise ValueError(
                    f"Total weight must be positive, got {total_weight}"
                )
            weights = {
                wh_id: warehouse_weights.get(wh_id, 0.0)
                for wh_id in relevant_warehouses
            }
        else:
            # 按需求量比例分配
            total_demand = sum(
                s.average_daily_demand for s in relevant_warehouses.values()
            )
            if total_demand <= 0:
                # 需求量未知时平均分配
                equal_weight = 1.0 / len(relevant_warehouses)
                weights = {
                    wh_id: equal_weight
                    for wh_id in relevant_warehouses
                }
            else:
                weights = {
                    wh_id: stock.average_daily_demand / total_demand
                    for wh_id, stock in relevant_warehouses.items()
                }

        # 执行分配
        total_weight = sum(weights.values())
        allocation: Dict[str, float] = {}
        allocated_total = 0.0

        sorted_warehouses = sorted(weights.keys(), key=lambda w: weights[w], reverse=True)
        for i, wh_id in enumerate(sorted_warehouses):
            if i == len(sorted_warehouses) - 1:
                # 最后一个仓库分配剩余量, 避免浮点误差
                allocated = total_safety_stock - allocated_total
            else:
                allocated = total_safety_stock * (weights[wh_id] / total_weight)
                allocated_total += allocated

            # 更新仓库安全库存
            if wh_id in self._warehouse_stocks and sku in self._warehouse_stocks[wh_id]:
                self._warehouse_stocks[wh_id][sku].safety_stock = allocated
                # 更新再订货点
                stock = self._warehouse_stocks[wh_id][sku]
                stock.reorder_point = (
                    allocated
                    + stock.average_daily_demand * stock.average_lead_time_days
                )

            allocation[wh_id] = allocated

        return allocation

    def update_service_level(
        self, sku: str, new_service_level: float
    ) -> Dict[str, float]:
        """动态调整服务水平和安全库存
        
        提高服务水平会增加安全库存, 降低则会减少安全库存。
        根据业务需要动态调整, 如促销期提高服务水平。
        
        Args:
            sku: 商品 SKU
            new_service_level: 新的服务水平 (0.90~0.999)
            
        Returns:
            Dict[warehouse_id, new_safety_stock]: 各仓库更新后的安全库存
        """
        if not (0.90 <= new_service_level <= 0.999):
            raise ValueError(
                f"Service level must be between 0.90 and 0.999, got {new_service_level}"
            )

        if sku in self._configs:
            self._configs[sku].default_service_level = new_service_level

        updated: Dict[str, float] = {}
        for wh_id, skus in self._warehouse_stocks.items():
            if sku in skus:
                new_ss = self.calculate_safety_stock(
                    wh_id, sku, service_level=new_service_level
                )
                updated[wh_id] = new_ss

        return updated

    def confirm_replenishment(self, order_id: str) -> Optional[ReplenishmentOrder]:
        """确认补货订单"""
        if order_id not in self._replenishment_orders:
            return None
        order = self._replenishment_orders[order_id]
        order.status = ReplenishmentStatus.CONFIRMED
        order.confirmed_at = datetime.utcnow()
        return order

    def receive_replenishment(
        self, order_id: str, received_quantity: float
    ) -> Optional[ReplenishmentOrder]:
        """收到补货, 更新库存"""
        if order_id not in self._replenishment_orders:
            return None
        order = self._replenishment_orders[order_id]
        order.status = ReplenishmentStatus.RECEIVED

        # 更新仓库库存
        wh_id = order.warehouse_id
        sku = order.sku
        if wh_id in self._warehouse_stocks and sku in self._warehouse_stocks[wh_id]:
            stock = self._warehouse_stocks[wh_id][sku]
            stock.current_stock += received_quantity
            stock.in_transit_quantity = max(
                0, stock.in_transit_quantity - order.quantity
            )

        # 更新在途数量追踪
        pending_key = f"{wh_id}:{sku}"
        if pending_key in self._pending_replenishment:
            self._pending_replenishment[pending_key]["total_pending"] = max(
                0,
                self._pending_replenishment[pending_key]["total_pending"]
                - order.quantity,
            )
        return order
```

## 异常场景补充

### 场景：安全水位设置过高导致库存积压
```
trigger: 为保证供货率, 将 service_level 从 0.95 提高到 0.999, 导致安全库存从 100 件飙升到 300 件, 仓库长期持有大量滞销库存, 库存周转率从 8 次/年降至 2 次/年

detection:
  1. 监控库存周转率: 当周转率低于阈值 (如 4 次/年) 时发出预警
  2. 计算库存持有成本: 当持有成本超过毛利的 X% 时告警
  3. 对比安全库存与实际消耗: 若连续 30 天库存始终高于安全水位的 150%, 判定为安全水位过高
  4. 定期审计: 每月计算各 SKU 的实际缺货率, 若实际缺货率远低于 (1 - service_level), 说明水位偏高

handling:
  1. 根据实际缺货率反向校准 service_level: 若实际缺货率仅 0.1% 而 target 为 5%, 降低 service_level
  2. 逐步降低安全库存: 每次降低 10%, 观察 2 周内缺货率变化, 避免一次性大幅调整
  3. 对滞销 SKU 实施清仓策略: 降价促销或调拨到其他仓库
  4. 设置安全库存上限 (max_safety_stock): 在 SafetyStockConfig 中配置, 防止计算结果失控
  5. 暂停自动补货: 将积压 SKU 的自动补货暂停, 等待库存消化到合理水位

prevention:
  1. 设置 service_level 调整审批流程: 超过 0.98 的服务水平需要供应链经理审批
  2. 在 calculate_safety_stock 中加入库存成本约束: 当安全库存导致的持有成本超过阈值时, 自动限幅
  3. 实现动态 service_level 调整: 基于实际缺货率自动微调, 避免一次性设置过高
  4. 定期 (每月) 回顾安全水位设置: 比较实际需求与预测需求, 校准需求标准差
  5. 区分 SKU 重要性: A 类商品 (高价值高周转) 使用高服务水平, C 类商品 (低价值低周转) 使用较低水平
```

### 场景：多个仓库同时触发补货导致超订
```
trigger: 3 个仓库同时检测到 SKU-X 库存低于安全水位, 各自独立触发补货订单, 但供应商总产能仅能满足 2 个仓库的需求, 导致总订购量超出供应商产能 50%, 交期被迫延长

detection:
  1. 监控同一 SKU 的跨仓库补货总量: 当所有仓库补货总量 > 供应商产能阈值时告警
  2. 检测补货集中度: 若 24 小时内超过 50% 仓库对同一 SKU 触发补货, 判定为异常集中补货
  3. 对比供应商产能约束: 在 trigger_replenishment 中检查总订购量是否超出供应商单次最大供货量

handling:
  1. 立即暂停所有未确认的补货订单 (status=PENDING), 避免继续超订
  2. 调用 allocate_safety_stock 重新分配有限库存: 根据各仓库需求紧迫度 (库存/安全库存比率) 排序
  3. 优先满足库存最低 (最接近缺货) 的仓库, 按优先级分配供应商产能
  4. 对未分配到产能的仓库, 协调仓库间调拨: 从库存充足的仓库调拨到库存紧张的仓库
  5. 与供应商协商分批交付: 将总订单拆为多批, 在产能范围内分批补货
  6. 更新各仓库的 expected_delivery, 反映实际延长的交期

prevention:
  1. 实现跨仓库补货协调: trigger_replenishment 执行前先检查其他仓库的补货状态, 合并或排队
  2. 在 SafetyStockConfig 中配置供应商产能上限 (supplier_capacity), 补货总量不超过此值
  3. 引入补货时间窗口错开机制: 各仓库的检查周期加随机偏移, 避免同一时刻集中触发
  4. 实现中心化补货调度: 由中心服务统一评估各仓库需求, 按优先级分配供应商产能
  5. 建立仓库间库存调拨机制: 当多仓库同时缺货时, 优先考虑内部调拨而非全部向供应商订购
  6. 在补货订单中添加产能检查: trigger_replenishment 返回前校验供应商是否可承接, 不可承接时排队等待
```
