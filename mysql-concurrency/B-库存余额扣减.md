# B. 库存 / 余额扣减

库存和余额是电商、支付系统中最核心的并发热点，处理不当直接导致资损。

---

## B1. 超卖问题

### 场景

库存剩余 1 件，两个并发请求同时扣减：

```sql
CREATE TABLE product (
  id BIGINT PRIMARY KEY,
  name VARCHAR(100),
  stock INT NOT NULL  -- 库存
) ENGINE=InnoDB;

-- 初始 stock = 1
```

```sql
-- 请求A                          -- 请求B
START TRANSACTION;                START TRANSACTION;

SELECT stock FROM product          SELECT stock FROM product
  WHERE id = 1;                     WHERE id = 1;
-- stock = 1                      -- stock = 1

UPDATE product SET stock = 0        UPDATE product SET stock = 0
  WHERE id = 1;                     WHERE id = 1;
-- 成功                           -- 成功

COMMIT;                            COMMIT;
-- 最终 stock = 0，但卖了2件！
```

### 问题分析

本质是**丢失更新**：两个事务都读到 stock=1，各自减 1 写回 0，实际应该为 -1。

关键错误在于 `SELECT stock` 是快照读，读到的是旧值，然后用应用层计算新值写回。这等价于 read-modify-write 无锁操作，并发下必然丢失更新。

时序：

```
时间  事务A                              事务B
 t1   SELECT stock → 1
 t2                                      SELECT stock → 1（A未提交，快照读看到旧值）
 t3   UPDATE stock=0 (基于1-1=0)
 t4                                      UPDATE stock=0 (基于1-1=0)
 t5   COMMIT
 t6                                      COMMIT
      → stock=0，但实际扣了2次
```

---

## B2. 乐观锁扣减

### 方案

在表中加 version 字段，UPDATE 时带上 version 条件：

```sql
ALTER TABLE product ADD COLUMN version INT NOT NULL DEFAULT 0;

START TRANSACTION;
SELECT stock, version FROM product WHERE id = 1;
-- stock=1, version=0

UPDATE product
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = 0;
-- affected rows = 1 → 扣减成功
-- affected rows = 0 → 被其他事务抢先修改，需要重试
COMMIT;
```

### 分析

```
时间  事务A                              事务B
 t1   SELECT → stock=1, version=0
 t2                                      SELECT → stock=1, version=0
 t3   UPDATE ... WHERE version=0
      → affected rows=1 ✅
 t4                                      UPDATE ... WHERE version=0
                                          → version 已被 A 改为 1
                                          → affected rows=0 ❌
 t5   COMMIT
 t6                                      应用层重试或返回失败
```

**优点：** 无锁等待，高并发下吞吐量好
**缺点：** 冲突率高时重试开销大；不适用于极端热点（如秒杀）

### 最佳实践

- 乐观锁适合**冲突概率低**的场景（如普通商品库存）
- 重试次数需限制（3-5 次），避免无限循环
- `affected rows = 0` 不等于库存不足，需重新 SELECT 判断是版本冲突还是真的没库存

---

## B3. 悲观锁扣减

### 方案

用 `SELECT FOR UPDATE` 获取排他锁，再更新：

```sql
START TRANSACTION;
SELECT stock FROM product WHERE id = 1 FOR UPDATE;
-- stock = 1，持有 id=1 的排他锁

-- 应用层判断 stock >= 1
UPDATE product SET stock = stock - 1 WHERE id = 1;
COMMIT;
```

或者更简洁——直接用条件 UPDATE：

```sql
UPDATE product SET stock = stock - 1
WHERE id = 1 AND stock >= 1;
-- affected rows = 1 → 扣减成功
-- affected rows = 0 → 库存不足
```

### 分析

`UPDATE ... WHERE stock >= 1` 的加锁过程：

1. 走主键索引找到 `id=1` 的记录
2. 加 **record lock**（X 型排他锁）
3. 检查 stock 值，满足则更新，不满足释放锁
4. 事务 B 的 UPDATE 等待 A 释放锁，释放后重新读取最新 stock 值

**直接用条件 UPDATE 是最简洁可靠的方式**，不需要先 SELECT FOR UPDATE。

但 `SELECT FOR UPDATE` 仍有其适用场景：
- 扣减前需要读取完整行数据做业务判断
- 需要在应用层做复杂的库存校验逻辑

### 注意事项

**二级索引上的 FOR UPDATE 会加 gap lock（RR 级别）：**

```sql
-- 如果 stock 列有索引
SELECT * FROM product WHERE stock = 1 FOR UPDATE;
-- 会在二级索引上加 next-key lock，锁定范围可能大于预期
-- 影响其他不相关的行的插入/更新
```

**扣减顺序要一致，否则死锁：**

```sql
-- 事务A：先锁商品1，再锁商品2
SELECT * FROM product WHERE id IN (1, 2) FOR UPDATE;

-- 事务B：先锁商品2，再锁商品1
SELECT * FROM product WHERE id IN (2, 1) FOR UPDATE;
-- → AB-BA 死锁
```

解决：按 id 升序排列后再加锁。

### 最佳实践

- 简单扣减用 `UPDATE SET stock=stock-1 WHERE stock>=1`，一步到位
- 需要读数据做判断时用 `SELECT FOR UPDATE`
- 多行扣减时按固定顺序（如 id 升序）加锁，避免死锁
- 扣减 SQL 中加 `stock >= 扣减量` 条件，防止超卖

---

## B4. 预扣减 + 确认

### 场景

电商下单流程：用户点击购买 → 锁定库存 → 支付 → 确认扣减。如果支付超时则释放库存。

```sql
CREATE TABLE product (
  id BIGINT PRIMARY KEY,
  total_stock INT NOT NULL,      -- 总库存
  locked_stock INT NOT NULL DEFAULT 0  -- 已锁定库存
) ENGINE=InnoDB;

-- 可用库存 = total_stock - locked_stock
```

### 方案

**第一步：预扣减（锁定库存）**

```sql
UPDATE product
SET locked_stock = locked_stock + 1
WHERE id = 1 AND total_stock - locked_stock >= 1;
-- affected rows=1 → 预扣成功
-- affected rows=0 → 库存不足
```

**第二步：支付成功，确认扣减**

```sql
UPDATE product
SET total_stock = total_stock - 1,
    locked_stock = locked_stock - 1
WHERE id = 1;
```

**第三步：支付超时，释放库存**

```sql
UPDATE product
SET locked_stock = locked_stock - 1
WHERE id = 1;
```

### 分析

**数据库层面没有并发问题。** 预扣减和释放都是对同一行的 `UPDATE`，InnoDB 行锁保证它们串行执行，不存在并发读写交叉的情况。

**真正的问题在应用逻辑层面：** `locked_stock` 只是一个计数器，不记录"谁锁了多少"：

**问题1：重复释放**

```
订单X 预扣减: locked_stock 9→10
订单Y 预扣减: locked_stock 10→11
订单X 超时释放: locked_stock 11→10  ← 正确
订单X 超时释放（重复回调）: locked_stock 10→9  ← 错误！没有对应的锁定
```

计数器无法校验"这次释放是否有对应的预扣减"，重复释放导致 locked_stock 被多减，可用库存虚增（超卖）。

**问题2：库存泄漏**

```
1. 预扣减成功: locked_stock 9→10
2. 应用崩溃，未记录锁定上下文
3. 重启后不知道该释放谁的锁
4. locked_stock 永远停在 10 → 可用库存永久减少
```

**问题3：locked_stock 与实际锁定记录不一致**

定时任务扫描超时订单释放库存时，`locked_stock - 1` 和"标记订单超时"不是原子操作，可能出现：
- 订单已标记超时但 locked_stock 没减 → 库存泄漏
- locked_stock 已减但订单未标记超时 → 重复释放

### 改进方案：用锁定记录替代全局计数器

```sql
CREATE TABLE stock_lock (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  product_id BIGINT NOT NULL,
  order_id BIGINT NOT NULL,
  quantity INT NOT NULL,
  status ENUM('LOCKED','CONFIRMED','RELEASED'),
  locked_at DATETIME NOT NULL,
  UNIQUE INDEX uk_order (order_id),
  INDEX idx_product_status (product_id, status)
) ENGINE=InnoDB;
```

```sql
-- 预扣减：插入锁定记录
INSERT INTO stock_lock (product_id, order_id, quantity, status, locked_at)
VALUES (1, 1001, 1, 'LOCKED', NOW());

-- 校验库存是否充足（用 SQL 聚合计算可用库存）
SELECT p.total_stock - IFNULL(SUM(
  CASE WHEN sl.status = 'LOCKED' THEN sl.quantity ELSE 0 END
), 0) AS available
FROM product p
LEFT JOIN stock_lock sl ON sl.product_id = p.id AND sl.status = 'LOCKED'
WHERE p.id = 1;

-- 确认扣减
START TRANSACTION;
UPDATE stock_lock SET status = 'CONFIRMED' WHERE order_id = 1001 AND status = 'LOCKED';
UPDATE product SET total_stock = total_stock - 1 WHERE id = 1;
COMMIT;

-- 超时释放（定时任务，原子操作）
UPDATE stock_lock SET status = 'RELEASED'
WHERE status = 'LOCKED' AND locked_at < NOW() - INTERVAL 5 MINUTE;
```

每条锁定记录对应一个订单，释放时精确匹配 `order_id`，不会重复释放也不会泄漏。

### 最佳实践

- 预扣减**不要用全局计数器**，用订单级别的锁定记录
- 锁定记录表加唯一索引（order_id），防止同一订单重复锁定
- 确认扣减和超时释放都操作锁定记录的状态字段，与计数器无关
- 高并发场景可用 Redis 预扣减 + 数据库异步确认
- 超时释放用定时任务扫描，不要依赖应用层回调（回调可能丢失）

---

## B5. 多商品批量扣减

### 场景

购物车结算，一次扣减多件商品库存：

```sql
-- 扣减：商品1减2件，商品2减1件
-- 如果商品2库存不足，商品1也不应扣减（原子性）
```

### 方案

**方案一：逐行 FOR UPDATE（按 id 升序）**

```sql
START TRANSACTION;
-- 按 id 升序加锁，防止死锁
SELECT * FROM product WHERE id = 1 FOR UPDATE;
SELECT * FROM product WHERE id = 2 FOR UPDATE;

-- 应用层校验库存是否充足
-- 商品1 stock=5 >= 2 ✅
-- 商品2 stock=0 >= 1 ❌ → 回滚

UPDATE product SET stock = stock - 2 WHERE id = 1;
UPDATE product SET stock = stock - 1 WHERE id = 2;
COMMIT;
```

**方案二：单条 UPDATE + CASE**

```sql
START TRANSACTION;
UPDATE product SET stock = CASE id
  WHEN 1 THEN stock - 2
  WHEN 2 THEN stock - 1
END
WHERE id IN (1, 2) AND stock >= CASE id
  WHEN 1 THEN 2
  WHEN 2 THEN 1
END;
-- 注意：无法保证"全部满足才扣减"，可能部分成功
COMMIT;
```

方案二有部分成功的风险，不推荐。需要配合应用层判断 affected rows。

### 死锁风险

批量扣减最大的风险是**加锁顺序不一致**：

```sql
-- 事务A：购买商品1和商品2
-- 按应用层传入顺序加锁：先锁1再锁2

-- 事务B：购买商品2和商品1
-- 按应用层传入顺序加锁：先锁2再锁1

-- → AB-BA 死锁
```

### 最佳实践

- **必须按固定顺序（如 id 升序）加锁**，这是批量操作的铁律
- 逐行 FOR UPDATE 比单条 UPDATE ... CASE 更可控
- 校验和扣减在同一事务内，库存不足则整体回滚
- 批量扣减行数不宜过多（建议 <= 20），否则锁持有时间长，影响并发

---

## B 章总结

| 场景 | 核心问题 | 推荐方案 |
|---|---|---|
| 超卖 | 丢失更新 | `UPDATE SET stock=stock-1 WHERE stock>=1` |
| 乐观锁 | 冲突重试 | 适合低冲突场景，version 字段 |
| 悲观锁 | 锁等待/死锁 | `SELECT FOR UPDATE` + 固定加锁顺序 |
| 预扣减 | 重复释放 / 库存泄漏 | 订单级锁定记录（不要用全局计数器） |
| 批量扣减 | 部分成功/死锁 | 按 id 升序 FOR UPDATE + 整体事务 |

**一条贯穿的规律：扣减操作永远不要"先读后写"，用原子 UPDATE + 条件判断一步完成。**
