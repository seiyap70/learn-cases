# F. 关联数据一致性

跨表、跨行、跨服务的数据需要保持一致性，是并发问题中最复杂的类别。

---

## F1. 主子表级联更新

### 场景

订单主表金额 = 所有订单明细金额之和：

```sql
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  total_amount DECIMAL(18,2) NOT NULL DEFAULT 0,
  status ENUM('DRAFT','CONFIRMED','PAID')
) ENGINE=InnoDB;

CREATE TABLE order_item (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  order_id BIGINT NOT NULL,
  product_id BIGINT NOT NULL,
  quantity INT NOT NULL,
  price DECIMAL(10,2) NOT NULL,
  INDEX idx_order (order_id)
) ENGINE=InnoDB;
```

### 常见问题

**问题1：主表金额与明细不一致**

```sql
-- 添加明细
INSERT INTO order_item (order_id, product_id, quantity, price)
VALUES (1, 100, 2, 50.00);

-- 更新主表金额（两步操作，非原子）
UPDATE orders SET total_amount = total_amount + 100.00 WHERE id = 1;
-- 如果 INSERT 成功但 UPDATE 失败 → 金额不一致
```

**问题2：并发修改同一订单的明细**

```sql
-- 事务A：添加明细1
INSERT INTO order_item (...) VALUES (1, 100, 2, 50.00);
UPDATE orders SET total_amount = total_amount + 100.00 WHERE id = 1;

-- 事务B：添加明细2
INSERT INTO order_item (...) VALUES (1, 200, 1, 30.00);
UPDATE orders SET total_amount = total_amount + 30.00 WHERE id = 1;
-- 两个事务都修改 orders.id=1 的 total_amount → 行锁串行化
```

### 解决方案

**方案一：同一事务内操作（基本要求）**

```sql
START TRANSACTION;
INSERT INTO order_item (order_id, product_id, quantity, price)
VALUES (1, 100, 2, 50.00);

UPDATE orders SET total_amount = total_amount + 100.00 WHERE id = 1;
COMMIT;
-- 事务保证原子性：要么都成功，要么都失败
```

**方案二：主表金额不实时计算，查询时汇总（推荐）**

```sql
-- 不维护 total_amount 字段，查询时计算
SELECT o.*,
  (SELECT SUM(quantity * price) FROM order_item WHERE order_id = o.id) AS total_amount
FROM orders o
WHERE o.id = 1;
```

优点：无需维护一致性，无并发问题。
缺点：查询时需聚合计算，大数据量下性能差。

**方案三：应用层计算 + 一次性写入**

```sql
START TRANSACTION;
-- 锁住主表行
SELECT * FROM orders WHERE id = 1 FOR UPDATE;

-- 插入明细
INSERT INTO order_item (...) VALUES (...);

-- 重新计算主表金额
UPDATE orders
SET total_amount = (
  SELECT SUM(quantity * price) FROM order_item WHERE order_id = 1
)
WHERE id = 1;
COMMIT;
```

### 最佳实践

- 主子表操作必须在同一事务内
- 优先考虑"不维护冗余汇总字段"，查询时计算
- 必须维护汇总字段时，用 `total_amount = total_amount + delta` 而非重新计算
- 高并发下，汇总字段的一致性可用定时对账任务保障

---

## F2. 跨表余额转移

### 场景

A 账户向 B 账户转账 100 元，涉及两行更新，必须原子且一致：

```sql
CREATE TABLE account (
  id BIGINT PRIMARY KEY,
  balance DECIMAL(18,2) NOT NULL,
  version INT NOT NULL DEFAULT 0
) ENGINE=InnoDB;
```

### 常见问题

**问题1：部分成功（A 扣了 B 没加）**

```sql
-- 不在事务内
UPDATE account SET balance = balance - 100 WHERE id = 1;  -- A 扣款成功
-- 系统崩溃
UPDATE account SET balance = balance + 100 WHERE id = 2;  -- B 未加款
-- → 100 元凭空消失
```

**问题2：死锁**

```sql
-- 事务A：A→B 转账
UPDATE account SET balance = balance - 100 WHERE id = 1;  -- 锁 A
UPDATE account SET balance = balance + 100 WHERE id = 2;  -- 等待 B

-- 事务B：B→A 转账
UPDATE account SET balance = balance - 100 WHERE id = 2;  -- 锁 B
UPDATE account SET balance = balance + 100 WHERE id = 1;  -- 等待 A
-- → AB-BA 死锁
```

**问题3：总金额不一致**

两个事务并发执行，如果丢失更新导致余额计算错误，A+B 的总额会变化。

### 解决方案

**方案一：事务 + 固定顺序加锁（推荐）**

```sql
START TRANSACTION;
-- 按 id 升序加锁，无论转账方向
SELECT * FROM account WHERE id = 1 FOR UPDATE;  -- 先锁小 id
SELECT * FROM account WHERE id = 2 FOR UPDATE;  -- 再锁大 id

-- 校验余额
-- A.balance >= 100 → 允许

UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

**方案二：事务 + 单条 SQL（更简洁）**

```sql
START TRANSACTION;
-- 用 CASE 在一条 SQL 中完成
UPDATE account SET balance = balance + CASE id
  WHEN 1 THEN -100
  WHEN 2 THEN 100
END
WHERE id IN (1, 2);
COMMIT;
```

注意：`WHERE id IN (1, 2)` 的加锁顺序取决于 InnoDB 的扫描顺序（按主键索引），天然按 id 升序，不会死锁。

**方案三：流水表 + 异步执行**

```sql
-- 记录转账流水
INSERT INTO transfer_log (from_id, to_id, amount, status)
VALUES (1, 2, 100, 'PENDING');

-- 异步任务执行转账
START TRANSACTION;
UPDATE account SET balance = balance - 100 WHERE id = 1 AND balance >= 100;
UPDATE account SET balance = balance + 100 WHERE id = 2;
-- 更新流水状态
UPDATE transfer_log SET status = 'DONE' WHERE id = #{logId};
COMMIT;
```

### 最佳实践

- 转账**必须**在同一事务内，两步更新原子化
- 加锁顺序按 id 升序，这是防止死锁的铁律
- 扣款方加 `balance >= amount` 条件，防止超扣
- 转账流水表记录每笔操作，用于对账和补偿
- `innodb_deadlock_detect = ON`（默认），让 InnoDB 自动检测死锁并回滚

---

## F3. 跨服务分布式事务

### 场景

微服务架构下，订单服务创建订单 + 库存服务扣减库存 + 支付服务发起支付，三个服务三个数据库：

```
订单服务 DB: INSERT INTO orders ...
库存服务 DB: UPDATE product SET stock = stock - 1 ...
支付服务 DB: INSERT INTO payment ...
```

任一步失败，需要回滚其他步骤。

### 常见问题

MySQL 单机事务无法跨服务（跨数据库），传统 XA 事务性能差、协调者单点。

### 解决方案

**方案一：Saga 模式（推荐）**

每个步骤有对应的补偿操作：

```
正向流程:
  1. 创建订单 → 补偿: 取消订单
  2. 扣减库存 → 补偿: 恢复库存
  3. 发起支付 → 补偿: 退款

执行:
  1 → 2 → 3 全部成功 → 完成
  1 → 2 失败 → 执行 2的补偿(恢复库存) → 1的补偿(取消订单)
```

**方案二：本地消息表**

```sql
-- 订单服务本地消息表
CREATE TABLE outbox_message (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  topic VARCHAR(100),
  payload JSON,
  status ENUM('PENDING','SENT'),
  created_at DATETIME NOT NULL
) ENGINE=InnoDB;
```

```sql
-- 订单创建和消息写入在同一本地事务内
START TRANSACTION;
INSERT INTO orders (...) VALUES (...);
INSERT INTO outbox_message (topic, payload, status, created_at)
VALUES ('stock_deduct', '{"product_id":1, "qty":1}', 'PENDING', NOW());
COMMIT;
-- 保证：订单创建成功则消息一定存在，不存在则订单也回滚
```

定时任务扫描 PENDING 消息，发送到 MQ，发送成功后标记 SENT。

**方案三：Seata AT 模式**

阿里开源的分布式事务框架，通过 SQL 拦截 + undo log 实现自动补偿：

```
1. 拦截业务 SQL，生成前镜像（before image）
2. 执行业务 SQL
3. 生成后镜像（after image）
4. 写 undo log
5. 全局提交 → 删除 undo log
6. 全局回滚 → 用 before image 反向补偿
```

优点：对业务代码侵入小。
缺点：性能有损耗，极端场景下 undo log 可能不一致。

### 最佳实践

| 方案 | 适用场景 | 一致性 |
|---|---|---|
| Saga | 长流程、多步骤 | 最终一致 |
| 本地消息表 | 异步通知、解耦 | 最终一致 |
| Seata AT | 短流程、强一致需求 | 读已提交级别一致 |
| XA | 传统架构、强一致 | 强一致（性能差） |

- 微服务下优先用 Saga 或本地消息表
- 本地消息表是最简单可靠的方案，不需要额外中间件
- 每个服务本地事务 + 消息表，保证"本地事务成功则消息一定发出"
- 补偿操作必须幂等（重复执行结果一致）

---

## F 章总结

| 场景 | 核心问题 | 推荐方案 |
|---|---|---|
| 主子表级联 | 冗余汇总字段不一致 | 不维护汇总字段 / 同事务内操作 |
| 跨表余额转移 | 原子性 + 死锁 | 事务 + 固定顺序加锁 |
| 跨服务分布式事务 | 跨数据库原子性 | Saga / 本地消息表 |

**一条贯穿的规律：关联数据一致性的核心是"原子性边界"——能放在一个事务内的就放一个事务内，不能的就用补偿/消息保证最终一致。**
