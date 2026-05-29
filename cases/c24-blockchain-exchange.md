# C24: 区块链交易所的核心撮合引擎

## 业务场景

某加密货币交易所，需要实现核心的撮合引擎——将买单和卖单按价格优先、时间优先的原则匹配成交。

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

买单按价格从高到低排序（出价高的先成交），卖单按价格从低到高排序（要价低的先成交）。同价格按时间先后排序。如何高效维护这个排序？

### 挑战 2：并发下单的正确性

两个用户同时下买单，价格相同，都匹配到了同一个卖单——但卖单的量不够两个买单都完全成交。谁先成交？如果并发处理不当，可能"双花"（同一卖单的量被扣除两次）。

### 挑战 3：性能与一致性的权衡

撮合必须在 10ms 内完成。但如果用数据库事务保证一致性（加行锁），延迟可能超过 10ms。如果用内存撮合，进程崩溃后数据怎么办？

### 挑战 4：行情推送的高扇出

每次成交都会影响行情（最新价、买一卖一、K 线等），需要推送给所有订阅该交易对的客户端。活跃交易对可能有 10 万+ 订阅者。

## 设计约束

- 撮合延迟 < 10ms
- 成交推送延迟 < 100ms
- 不允许"双花"（同一笔挂单被撮合两次）
- 进程崩溃后数据不能丢失（可恢复到崩溃前最后一笔撮合）
- 支持 100+ 交易对

## 请先独立思考（限时 40 分钟）

1. 撮合引擎用内存还是数据库？内存撮合如何保证崩溃恢复？
2. 买单和卖单的数据结构选型：红黑树 vs 跳表 vs 优先队列？各自的查找、插入、删除时间复杂度？
3. 并发下单如何避免"双花"？单线程撮合 vs 分布式锁？
4. 行情推送的扇出优化：每次成交都推送 vs 聚合推送？

---

## 设计解析

### 核心原则：内存撮合 + 事件溯源

**为什么不在数据库中撮合？**

数据库撮合（每笔订单 INSERT + 锁行 + UPDATE）：
- 行锁等待：高并发下锁竞争严重 → 延迟 50-200ms
- 随机 I/O：订单簿的查找和更新涉及 B+ 树索引 → 延迟不稳定

内存撮合：
- 查找：O(log N) 内存操作 → < 1ms
- 匹配：顺序扫描订单簿 → < 1ms
- 总计：< 2ms（远优于 10ms 要求）

**内存撮合的崩溃恢复：事件溯源**

每次状态变更（下单、撤单、成交）记录为一个事件，追加写入 WAL（Write-Ahead Log）：

```python
class MatchingEngine:
    def __init__(self, symbol):
        self.symbol = symbol
        self.bids = SkipList(reverse=True)   # 买单：价格降序
        self.asks = SkipList(reverse=False)   # 卖单：价格升序
        self.wal = WAL(f"matching_{symbol}.log")

    def place_order(self, order):
        """下订单"""
        # 1. 写入 WAL（先写日志再操作——WAL 原则）
        self.wal.append({
            "type": "PLACE_ORDER",
            "order_id": order.id,
            "side": order.side,      # BUY / SELL
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

        return trades

    def recover(self):
        """从 WAL 恢复引擎状态"""
        for event in self.wal.read_all():
            if event["type"] == "PLACE_ORDER":
                order = Order.from_event(event)
                if event["side"] == "BUY":
                    self.bids.insert(order.price, order)
                else:
                    self.asks.insert(order.price, order)
            elif event["type"] == "TRADE":
                # 重放成交：更新订单簿
                self.replay_trade(event)
            elif event["type"] == "CANCEL_ORDER":
                self.remove_order(event["order_id"])
```

**恢复时间：** 活跃交易对约 1 万笔事件/分钟 × 1 天 = 1440 万事件 → 重放约 10-30 秒。可接受。

### 撮合算法：价格优先、时间优先

```python
class SkipList:
    """跳表：O(log N) 查找、插入、删除"""
    def __init__(self, reverse=False):
        self.reverse = reverse
        self.levels = [Level()]
        self.order_queues = {}  # price -> deque（同价格的订单队列）

    def insert(self, price, order):
        if price not in self.order_queues:
            self.order_queues[price] = deque()
            self._insert_price(price)  # 跳表插入价格节点
        self.order_queues[price].append(order)  # 时间优先：追加到队列尾部

    def get_best(self):
        """获取最优价格的所有订单"""
        best_price = self._get_best_price()
        if best_price is None:
            return []
        return list(self.order_queues[best_price])

    def remove_order(self, price, order_id):
        if price in self.order_queues:
            q = self.order_queues[price]
            # 从队列中移除指定订单
            self.order_queues[price] = deque(
                o for o in q if o.id != order_id
            )
            if not self.order_queues[price]:
                del self.order_queues[price]
                self._remove_price(price)


class MatchingEngine:
    def match(self, order):
        """执行撮合"""
        trades = []

        if order.side == "BUY":
            # 买单：与卖单簿中的最低价匹配
            while order.quantity > 0 and self.asks.has_orders():
                if order.price < self.asks.best_price():
                    break  # 买价低于最低卖价，无法匹配

                # 取最优卖单
                sell_orders = self.asks.get_best()
                for sell_order in sell_orders:
                    if order.quantity <= 0:
                        break

                    match_qty = min(order.quantity, sell_order.quantity)
                    trade = Trade(
                        buy_order_id=order.id,
                        sell_order_id=sell_order.id,
                        price=sell_order.price,  # 成交价 = 挂单方价格
                        quantity=match_qty
                    )
                    trades.append(trade)

                    order.quantity -= match_qty
                    sell_order.quantity -= match_qty

                    if sell_order.quantity <= 0:
                        self.asks.remove_order(sell_order.price, sell_order.id)

        elif order.side == "SELL":
            # 卖单：与买单簿中的最高价匹配
            while order.quantity > 0 and self.bids.has_orders():
                if order.price > self.bids.best_price():
                    break  # 卖价高于最高买价，无法匹配

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

        # 未完全成交的订单进入订单簿
        if order.quantity > 0:
            if order.side == "BUY":
                self.bids.insert(order.price, order)
            else:
                self.asks.insert(order.price, order)

        return trades
```

### 并发控制：每个交易对单线程撮合

**为什么不用分布式锁？**

分布式锁（Redis SETNX）延迟约 1-2ms → 加上撮合逻辑 → 总延迟 3-4ms → 看似可接受。

但问题在于：同一交易对的订单串行化后，1 万 QPS 需要每个订单在 0.1ms 内完成——包括锁获取、撮合、WAL 写入。太紧张。

**方案：每个交易对一个独立线程，该线程串行处理所有订单**

```python
class MatchingEnginePool:
    """每个交易对一个撮合线程"""

    def __init__(self):
        self.engines = {}   # symbol → MatchingEngine
        self.queues = {}    # symbol → Queue

    def start_engine(self, symbol):
        engine = MatchingEngine(symbol)
        queue = Queue()
        self.engines[symbol] = engine
        self.queues[symbol] = queue

        # 单线程事件循环
        thread = Thread(target=self._event_loop, args=(symbol,), daemon=True)
        thread.start()

    def _event_loop(self, symbol):
        engine = self.engines[symbol]
        queue = self.queues[symbol]

        while True:
            order = queue.get()  # 阻塞等待
            trades = engine.place_order(order)
            self.publish_trades(trades)
            self.notify_order_status(order)

    def submit_order(self, symbol, order):
        """提交订单到对应交易对的队列"""
        self.queues[symbol].put(order)
```

**单线程串行化保证了：同一交易对的订单不会并发撮合 → 不可能"双花"。**

### 行情推送：聚合推送

```python
class MarketDataPublisher:
    def __init__(self):
        self.last_push_time = {}  # symbol → last_push_timestamp

    def publish_trades(self, trades):
        for trade in trades:
            symbol = trade.symbol

            # 更新行情缓存
            self.update_ticker(symbol, trade)
            self.update_depth(symbol)
            self.update_kline(symbol, trade)

            # 推送给订阅者（通过 Redis Pub/Sub）
            self.redis.publish(f"market:{symbol}", json.dumps({
                "type": "trade",
                "price": trade.price,
                "quantity": trade.quantity,
                "timestamp": trade.timestamp
            }))

    def publish_depth_snapshot(self, symbol):
        """定期推送订单簿深度快照（每 100ms）"""
        depth = self.engines[symbol].get_depth(levels=20)
        self.redis.publish(f"depth:{symbol}", json.dumps(depth))
```

**推送频率优化：**
- 成交推送：每次成交实时推送（延迟 < 10ms）
- 订单簿深度：每 100ms 推送一次快照（避免高频更新淹没客户端）
- K 线数据：每 1 秒聚合推送

## 常见陷阱（深度分析）

### 陷阱 1：数据库撮合

**延迟分析：** 下单 → INSERT orders → SELECT 限价单（行锁）→ UPDATE 成交 → 约 50-200ms → 违反 10ms 要求。

### 陷阱 2：内存撮合无 WAL

**后果：** 进程崩溃 → 所有订单簿数据丢失 → 用户挂单凭空消失 → 交易所信誉崩塌。

**解决方案：** WAL + 定期快照。崩溃后从最近快照 + WAL 重放恢复。

### 陷阱 3：同价格订单无时间排序

**后果：** 两个同价格买单，后到的先成交 → 违反"价格优先、时间优先"规则 → 用户投诉"抢单"。

**解决方案：** 同价格用 deque 队列，先进先出。

### 陷阱 4：行情全量推送

**后果：** 订单簿每次变更都推送完整深度（5000 挂单 × 100ms = 每秒推送 5 万条数据/订阅者）→ 网络带宽打满。

**解决方案：** 增量推送（只推送变化的部分）+ 100ms 聚合。

## 延伸思考

- **跨交易对套利**：BTC/USDT 和 BTC/BUSD 之间的价差套利，需要同时撮合两个交易对 → 如何保证原子性？
- **合约交易**：永续合约的撮合需要考虑保证金和爆仓机制，比现货撮合复杂得多。
- **去中心化撮合**：链上撮合（如 Uniswap AMM）与链下撮合（CLOB）的权衡。