# C24: 区块链交易所的核心撮合引擎

## 业务场景

某加密货币交易所，需要实现核心的撮合引擎——将买单和卖单按价格优先、时间优先的原则匹配成交。

**一个真实的故障：** 2023 年某交易所在 BTC 剧烈波动时，撮合引擎延迟从 5ms 飙升到 500ms → 用户在价格 30000 下单，撮合时价格已变到 29800 → 用户以 29800 成交 → 损失 200 USDT/枚 × 10 枚 = 2000 USDT → 大量用户投诉 → 24 小时内提现 5 亿美元 → 交易所流动性危机。

**已知数据：**
- 交易对：100+ 个（BTC/USDT, ETH/USDT 等）
- 峰值下单 QPS：1 万/秒
- 订单簿深度：活跃交易对约 5000 挂单
- 撮合延迟要求：< 10ms（从下单到撮合结果返回）
- 成交推送延迟：< 100ms
- 24×7 运行，不允许停机

**为什么是难题？**

撮合引擎是整个交易所的核心——撮合的正确性决定了交易所的信誉，撮合的速度决定了用户的交易体验。一个撮合错误可能导致：
- 用户以错误的价格成交 → 资金损失 → 法律纠纷
- 撮合延迟导致用户在价格变动后才成交 → 用户投诉
- 并发下单导致同一笔挂单被撮合两次 → "双花"问题

## 核心挑战

### 挑战 1：价格优先、时间优先的排序

买单按价格从高到低排序（出价高的先成交），卖单按价格从低到高排序（要价低的先成交）。同价格按时间先后排序。5000 个挂单的订单簿，如何高效维护？

### 挑战 2：并发下单的正确性

两个用户同时下买单，价格相同，都匹配到了同一个卖单——但卖单的量不够两个买单都完全成交。谁先成交？如果并发处理不当，可能"双花"（同一卖单的量被扣除两次）。

### 挑战 3：性能与一致性的权衡

撮合必须在 10ms 内完成。但如果用数据库事务保证一致性（加行锁），延迟可能超过 10ms。如果用内存撮合，进程崩溃后数据怎么办？

**数据库撮合 vs 内存撮合延迟对比：**

| 操作 | 数据库（MySQL） | 内存 |
|------|-------------|------|
| 插入订单 | 2-5ms（磁盘 I/O） | < 0.01ms |
| 查找最优价格 | 1-3ms（B+ 树索引） | < 0.01ms（跳表） |
| 更新成交 | 2-5ms（行锁） | < 0.01ms |
| **总计** | **5-13ms** | **< 1ms** |

### 挑战 4：行情推送的高扇出

每次成交都会影响行情（最新价、买一卖一、K 线等），需要推送给所有订阅该交易对的客户端。活跃交易对可能有 10 万+ 订阅者。

### 挑战 5：崩溃恢复

内存撮合 → 进程崩溃 → 订单簿数据丢失 → 用户挂单凭空消失 → 交易所信誉崩塌。如何保证崩溃后数据可恢复？

## 设计约束

- 撮合延迟 < 10ms
- 成交推送延迟 < 100ms
- 不允许"双花"（同一笔挂单被撮合两次）
- 进程崩溃后数据不能丢失（可恢复到崩溃前最后一笔撮合）
- 支持 100+ 交易对
- 恢复时间 < 30 秒

## 请先独立思考（限时 40 分钟）

1. 撮合引擎用内存还是数据库？内存撮合如何保证崩溃恢复？
2. 买单和卖单的数据结构选型：红黑树 vs 跳表 vs 优先队列？各自的查找、插入、删除时间复杂度？
3. 并发下单如何避免"双花"？单线程撮合 vs 分布式锁？
4. 行情推送的扇出优化：每次成交都推送 vs 聚合推送？

---

## 设计解析

### 核心原则：内存撮合 + WAL + 单线程

**为什么不在数据库中撮合？**

数据库撮合（每笔订单 INSERT + 锁行 + UPDATE）：
- 行锁等待：高并发下锁竞争严重 → 延迟 50-200ms
- 随机 I/O：订单簿的查找和更新涉及 B+ 树索引 → 延迟不稳定

内存撮合：
- 查找：O(log N) 内存操作 → < 1ms
- 匹配：顺序扫描订单簿 → < 1ms
- 总计：< 2ms（远优于 10ms 要求）

**为什么单线程撮合？**

单线程 = 无锁 = 无并发 bug = 无"双花"。同一交易对的订单串行处理 → 不会出现两个订单同时匹配同一卖单的情况。

100 个交易对 = 100 个线程 → 每线程处理 1 个交易对 → 整体吞吐量 = 100 × 单线程吞吐量。

**单线程吞吐量够吗？** 1 万 QPS / 100 交易对 = 100 QPS/交易对。单线程处理一个订单 < 0.1ms → 每秒可处理 10000 个 → 远超 100 QPS。

### 内存撮合 + WAL

```python
class MatchingEngine:
    def __init__(self, symbol):
        self.symbol = symbol
        self.bids = SkipList(reverse=True)   # 买单：价格降序
        self.asks = SkipList(reverse=False)   # 卖单：价格升序
        self.wal = WAL(f"matching_{symbol}.log")
        self.order_map = {}  # order_id → Order（快速查找）

    def place_order(self, order):
        """下订单"""
        # 1. 写入 WAL（先写日志再操作——WAL 原则）
        self.wal.append({
            "type": "PLACE_ORDER",
            "order_id": order.id,
            "side": order.side,
            "price": order.price,
            "quantity": order.quantity,
            "timestamp": now_ms()
        })

        # 2. 执行撮合
        trades = self.match(order)

        # 3. 成交也写入 WAL
        for trade in trades:
            self.wal.append({
                "type": "TRADE",
                "trade_id": trade.id,
                "buy_order_id": trade.buy_order_id,
                "sell_order_id": trade.sell_order_id,
                "price": trade.price,
                "quantity": trade.quantity,
                "timestamp": now_ms()
            })

        # 4. 未完全成交的订单进入订单簿
        if order.quantity > 0:
            self.order_map[order.id] = order
            if order.side == "BUY":
                self.bids.insert(order.price, order)
            else:
                self.asks.insert(order.price, order)

        return trades

    def cancel_order(self, order_id):
        """撤单"""
        order = self.order_map.get(order_id)
        if not order:
            return False
        
        # 1. 写 WAL
        self.wal.append({
            "type": "CANCEL_ORDER",
            "order_id": order_id,
            "timestamp": now_ms()
        })
        
        # 2. 从订单簿移除
        if order.side == "BUY":
            self.bids.remove_order(order.price, order_id)
        else:
            self.asks.remove_order(order.price, order_id)
        
        del self.order_map[order_id]
        return True
```

### 崩溃恢复：WAL + 定期快照

```python
class MatchingEngineRecovery:
    def recover(self, symbol):
        """从快照 + WAL 恢复引擎状态"""
        engine = MatchingEngine(symbol)

        # 1. 加载最近的快照
        snapshot = self.load_latest_snapshot(symbol)
        if snapshot:
            engine.restore_from_snapshot(snapshot)
            last_snapshot_seq = snapshot.seq
        else:
            last_snapshot_seq = 0

        # 2. 重放 WAL（从快照之后的事件）
        for event in self.wal.read_since(last_snapshot_seq):
            if event["type"] == "PLACE_ORDER":
                order = Order.from_event(event)
                engine.order_map[order.id] = order
                if order.side == "BUY":
                    engine.bids.insert(order.price, order)
                else:
                    engine.asks.insert(order.price, order)
            elif event["type"] == "TRADE":
                engine.replay_trade(event)
            elif event["type"] == "CANCEL_ORDER":
                engine.cancel_order(event["order_id"])

        return engine

    def take_snapshot(self, engine):
        """定期快照（每 1 分钟）"""
        snapshot = {
            "symbol": engine.symbol,
            "seq": engine.wal.current_seq,
            "bids": engine.bids.serialize(),
            "asks": engine.asks.serialize(),
            "timestamp": now_ms()
        }
        self.save_snapshot(engine.symbol, snapshot)
```

**恢复时间分析：**

| 恢复方式 | 恢复时间 | 数据损失 |
|---------|---------|---------|
| 只用 WAL（1天） | 10-30 秒 | 无 |
| 快照（1分钟间隔）+ WAL | < 1 秒 | 无 |
| 无 WAL | - | 全部丢失 |

### 跳表实现：O(log N) 查找 + 同价格队列

```python
class SkipList:
    """跳表：O(log N) 查找、插入、删除"""
    def __init__(self, reverse=False):
        self.reverse = reverse
        self.price_levels = []      # 跳表价格节点
        self.order_queues = {}      # price → deque（同价格的订单队列，FIFO）

    def insert(self, price, order):
        if price not in self.order_queues:
            self.order_queues[price] = deque()
            self._insert_price(price)  # 跳表插入价格节点 O(log N)
        self.order_queues[price].append(order)  # 时间优先：追加到队列尾部

    def get_best(self):
        """获取最优价格的所有订单"""
        best_price = self._get_best_price()  # O(1)
        if best_price is None:
            return []
        return list(self.order_queues[best_price])

    def remove_order(self, price, order_id):
        if price in self.order_queues:
            q = self.order_queues[price]
            self.order_queues[price] = deque(
                o for o in q if o.id != order_id
            )
            if not self.order_queues[price]:
                del self.order_queues[price]
                self._remove_price(price)  # O(log N)
```

**数据结构对比：**

| 结构 | 查找最优价 | 插入 | 删除 | 同价格排序 |
|------|-----------|------|------|-----------|
| 红黑树 | O(log N) | O(log N) | O(log N) | 需额外链表 |
| 跳表 | O(1)* | O(log N) | O(log N) | deque 队列 |
| 优先队列 | O(1) | O(log N) | O(N)** | 不支持 |

*跳表最高层直接指向最优价格
**优先队列删除指定元素需要 O(N) 扫描

选择跳表：查找 O(1)、插入删除 O(log N)、同价格 FIFO 队列。比优先队列好在删除是 O(log N) 而非 O(N)（撤单场景频繁）。

### 撮合算法

```python
class MatchingEngine:
    def match(self, order):
        """执行撮合"""
        trades = []

        if order.side == "BUY":
            while order.quantity > 0 and self.asks.has_orders():
                if order.price < self.asks.best_price():
                    break  # 买价低于最低卖价，无法匹配

                sell_orders = self.asks.get_best()
                for sell_order in sell_orders:
                    if order.quantity <= 0:
                        break

                    match_qty = min(order.quantity, sell_order.quantity)
                    trade = Trade(
                        buy_order_id=order.id,
                        sell_order_id=sell_order.id,
                        price=sell_order.price,  # 成交价 = 挂单方价格（Maker）
                        quantity=match_qty
                    )
                    trades.append(trade)

                    order.quantity -= match_qty
                    sell_order.quantity -= match_qty

                    if sell_order.quantity <= 0:
                        self.asks.remove_order(sell_order.price, sell_order.id)
                        del self.order_map[sell_order.id]

        elif order.side == "SELL":
            while order.quantity > 0 and self.bids.has_orders():
                if order.price > self.bids.best_price():
                    break

                buy_orders = self.bids.get_best()
                for buy_order in buy_orders:
                    if order.quantity <= 0:
                        break

                    match_qty = min(order.quantity, buy_order.quantity)
                    trade = Trade(
                        buy_order_id=buy_order.id,
                        sell_order_id=order.id,
                        price=buy_order.price,  # 成交价 = 挂单方价格
                        quantity=match_qty
                    )
                    trades.append(trade)

                    order.quantity -= match_qty
                    buy_order.quantity -= match_qty

                    if buy_order.quantity <= 0:
                        self.bids.remove_order(buy_order.price, buy_order.id)
                        del self.order_map[buy_order.id]

        return trades
```

**撮合规则总结：**

| 规则 | 说明 |
|------|------|
| 价格优先 | 买单出价高者优先，卖单要价低者优先 |
| 时间优先 | 同价格先到者优先（FIFO 队列） |
| 成交价 = Maker 价格 | 主动方（Taker）以挂单方（Maker）的价格成交 |

### 并发控制：每个交易对单线程

```python
class MatchingEnginePool:
    """每个交易对一个撮合线程"""

    def __init__(self):
        self.engines = {}
        self.queues = {}

    def start_engine(self, symbol):
        engine = MatchingEngine(symbol)
        queue = Queue()
        self.engines[symbol] = engine
        self.queues[symbol] = queue

        thread = Thread(target=self._event_loop, args=(symbol,), daemon=True)
        thread.start()

    def _event_loop(self, symbol):
        engine = self.engines[symbol]
        queue = self.queues[symbol]

        while True:
            order = queue.get()
            trades = engine.place_order(order)
            
            # 异步推送成交和行情
            if trades:
                self.publish_trades(trades)
            
            self.notify_order_status(order)

    def submit_order(self, symbol, order):
        """提交订单到对应交易对的队列"""
        self.queues[symbol].put(order)
```

**单线程串行化保证了：同一交易对的订单不会并发撮合 → 不可能"双花"。**

### 行情推送：分层频率

```python
class MarketDataPublisher:
    def __init__(self):
        self.depth_timers = {}    # symbol → last depth push time
        self.kline_timers = {}    # symbol → last kline push time

    def publish_trades(self, trades):
        """成交实时推送（每次成交都推）"""
        for trade in trades:
            symbol = trade.symbol
            self.update_ticker(symbol, trade)
            
            # 通过 Redis Pub/Sub 推送成交
            self.redis.publish(f"trade:{symbol}", json.dumps({
                "price": trade.price,
                "quantity": trade.quantity,
                "timestamp": trade.timestamp
            }))

    def publish_depth_snapshot(self, symbol):
        """订单簿深度：每 100ms 推送一次"""
        now_ms_val = now_ms()
        if now_ms_val - self.depth_timers.get(symbol, 0) < 100:
            return  # 100ms 内已推送过
        
        depth = self.engines[symbol].get_depth(levels=20)
        self.redis.publish(f"depth:{symbol}", json.dumps(depth))
        self.depth_timers[symbol] = now_ms_val

    def publish_kline(self, symbol):
        """K 线数据：每 1 秒聚合推送"""
        now_ms_val = now_ms()
        if now_ms_val - self.kline_timers.get(symbol, 0) < 1000:
            return
        
        kline = self.compute_kline(symbol, interval="1s")
        self.redis.publish(f"kline:{symbol}", json.dumps(kline))
        self.kline_timers[symbol] = now_ms_val
```

**推送频率分级：**

| 数据类型 | 推送频率 | 数据量/次 | 订阅者带宽 |
|---------|---------|---------|-----------|
| 成交 | 实时（每次成交） | 100B | ~10KB/s |
| 订单簿深度 | 100ms 一次 | 5KB | 50KB/s |
| K 线 | 1s 一次 | 200B | 200B/s |

### 资金结算：异步 + 对账

```python
class SettlementService:
    """成交后的资金结算（异步，非撮合路径）"""

    def on_trade(self, trade):
        """成交后扣款/加款"""
        # 撮合成功 → 写入结算队列
        self.mq.produce("settlement", {
            "trade_id": trade.id,
            "buy_order_id": trade.buy_order_id,
            "sell_order_id": trade.sell_order_id,
            "price": trade.price,
            "quantity": trade.quantity
        })

    def process_settlement(self, trade):
        """异步结算"""
        # 扣买方 USDT，加买方 BTC
        # 扣卖方 BTC，加卖方 USDT
        with self.db.transaction():
            self.db.execute("""
                UPDATE balances 
                SET usdt = usdt - %s, btc = btc + %s 
                WHERE user_id = (SELECT user_id FROM orders WHERE id = %s)
            """, trade.price * trade.quantity, trade.quantity, trade.buy_order_id)
            
            self.db.execute("""
                UPDATE balances 
                SET btc = btc - %s, usdt = usdt + %s 
                WHERE user_id = (SELECT user_id FROM orders WHERE id = %s)
            """, trade.quantity, trade.price * trade.quantity, trade.sell_order_id)

    def reconcile(self):
        """定期对账：撮合记录 vs 资金变动"""
        # 每分钟检查：撮合引擎的成交总额 vs 数据库的资金变动总额
        matching_total = self.get_matching_total()
        settlement_total = self.get_settlement_total()
        
        if matching_total != settlement_total:
            self.alert("撮合与结算不一致！差额: %s" % (matching_total - settlement_total))
```

**为什么资金结算不在撮合路径上？** 撮合延迟要求 < 10ms → 数据库事务延迟 5-10ms → 撮合+结算 = 10-20ms → 可能超标。异步结算：撮合 < 1ms，结算 < 100ms。对账保证一致性。

## 常见陷阱（深度分析）

### 陷阱 1：数据库撮合

**延迟分析：** 下单 → INSERT orders → SELECT 限价单（行锁）→ UPDATE 成交 → 约 50-200ms → 违反 10ms 要求。1 万 QPS 时锁等待更严重 → 可能到 500ms。

**解决方案：** 内存撮合 + WAL。撮合延迟 < 1ms。

### 陷阱 2：内存撮合无 WAL

**后果：** 进程崩溃 → 所有订单簿数据丢失 → 用户挂单凭空消失 → 交易所信誉崩塌。用户无法撤回已丢失的挂单 → 资金被锁。

**解决方案：** WAL + 定期快照。崩溃后从最近快照 + WAL 重放恢复，恢复时间 < 1 秒。

### 陷阱 3：同价格订单无时间排序

**后果：** 两个同价格买单，后到的先成交 → 违反"价格优先、时间优先"规则 → 用户投诉"抢单"。特别是大额订单可能被"插队" → 用户利益受损。

**解决方案：** 同价格用 deque 队列，先进先出。跳表的每个价格节点挂一个 deque。

### 陷阱 4：行情全量推送

**后果：** 订单簿每次变更都推送完整深度（5000 挂单 × 100ms = 每秒推送 5 万条数据/订阅者）→ 10 万订阅者 × 5 万条 = 50 亿条/秒 → 网络带宽打满。

**解决方案：** 成交实时推送 + 订单簿深度 100ms 聚合 + K 线 1s 聚合。分层推送。

### 陷阱 5：撮合和结算在同一事务

**后果：** 撮合 1ms + 数据库结算 5ms = 6ms → 看似满足 10ms。但峰值时数据库锁等待 → 结算 50ms → 撮合路径总延迟 51ms → 用户体验差。

**解决方案：** 撮合和结算分离。撮合 < 1ms（纯内存），结算异步（< 100ms），对账保证一致性。

### 陷阱 6：单交易对单线程不处理热点

**后果：** BTC/USDT 峰值 5000 QPS → 单线程处理不过来 → 队列积压 → 撮合延迟飙升。

**解决方案：** 热点交易对可以按订单类型分片（限价单 vs 市价单分两个线程），或使用更高效的数据结构。但一般 100 QPS/交易对够用，BTC/USDT 可能需要 2-4 个分片。

## 延伸思考

- **跨交易对套利**：BTC/USDT 和 BTC/BUSD 之间的价差套利，需要同时撮合两个交易对 → 如何保证原子性？两阶段提交 vs 补偿事务？
- **合约交易**：永续合约的撮合需要考虑保证金和爆仓机制，比现货撮合复杂得多。爆仓时需要强制平仓 → 撮合引擎需要与风控引擎交互。
- **去中心化撮合**：链上撮合（如 Uniswap AMM）与链下撮合（CLOB）的权衡。AMM 无需订单簿但滑点大，CLOB 撮合效率高但需要中心化信任。