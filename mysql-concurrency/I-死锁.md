# I. 死锁

死锁是并发系统中最难预测和调试的问题。InnoDB 自动检测死锁并回滚代价最小的事务，但频繁死锁会严重影响业务。

---

## 死锁基础知识

### InnoDB 死锁检测

InnoDB 通过 **wait-for graph**（等待图）检测死锁：
- 每个锁等待是一条有向边：事务 A 等待事务 B 持有的锁 → A → B
- 如果图中出现环 → 死锁
- InnoDB 选择代价最小的事务回滚（通常是修改行数最少的事务）

```sql
-- 查看死锁日志
SHOW ENGINE INNODB STATUS\G
-- LATEST DETECTED DEADLOCK 部分记录最近一次死锁详情

-- 开启死锁日志到表（MySQL 8.0+）
SET GLOBAL innodb_print_all_deadlocks = ON;
-- 死锁信息写入 error log，便于事后分析
```

### 死锁 vs 锁等待

| | 锁等待 | 死锁 |
|---|---|---|
| 表现 | 事务等待，超时后报错 | InnoDB 立即检测，回滚一个事务 |
| 超时 | `innodb_lock_wait_timeout`（默认50秒） | 自动检测，无需等待超时 |
| 影响 | 单个事务阻塞 | 一个事务被回滚，需应用层重试 |
| 处理 | 等待或优化索引 | 修改加锁顺序或业务逻辑 |

---

## I1. AB-BA 死锁

### 场景

两个事务以不同顺序锁定相同的资源集：

```sql
-- 初始数据
INSERT INTO account VALUES (1, 1000), (2, 2000);
```

```sql
-- 事务A: 账户1→账户2 转账          -- 事务B: 账户2→账户1 转账
START TRANSACTION;                  START TRANSACTION;

UPDATE account                      UPDATE account
  SET balance = balance - 100         SET balance = balance - 100
  WHERE id = 1;                       WHERE id = 2;
-- 锁住 id=1 ✅                     -- 锁住 id=2 ✅

UPDATE account                      UPDATE account
  SET balance = balance + 100         SET balance = balance + 100
  WHERE id = 2;                       WHERE id = 1;
-- 等待 B 释放 id=2 ❌              -- 等待 A 释放 id=1 ❌

-- → 死锁！InnoDB 回滚其中一个
```

### 死锁图

```
事务A: id=1 → 等待id=2
事务B: id=2 → 等待id=1

A → B → A  ← 环路 = 死锁
```

### 解决方案

**按固定顺序加锁：**

```sql
-- 无论转账方向，都按 id 升序加锁
-- A→B: 先锁 id=1 再锁 id=2
-- B→A: 也先锁 id=1 再锁 id=2（因为 1 < 2）

START TRANSACTION;
-- 按 id 升序锁定两个账户
SELECT * FROM account WHERE id = LEAST(1, 2) FOR UPDATE;
SELECT * FROM account WHERE id = GREATEST(1, 2) FOR UPDATE;

-- 执行转账逻辑
UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

**或用单条 UPDATE：**

```sql
START TRANSACTION;
UPDATE account SET balance = balance + CASE id
  WHEN 1 THEN -100
  WHEN 2 THEN 100
END
WHERE id IN (1, 2);
-- InnoDB 按主键顺序加锁，天然有序
COMMIT;
```

### 最佳实践

- 多行操作**必须按固定顺序加锁**（如按 id 升序）
- 能用单条 SQL 搞定的不要拆成多条 UPDATE
- 应用层排序后再 FOR UPDATE

---

## I2. Gap Lock + Insert 死锁

### 场景

两个事务先 SELECT FOR UPDATE 探测记录是否存在，不存在则 INSERT：

```sql
CREATE TABLE user_coupon (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT NOT NULL,
  coupon_id BIGINT NOT NULL,
  UNIQUE INDEX uk_user_coupon (user_id, coupon_id)
) ENGINE=InnoDB;

-- 表中无 user_id=1, coupon_id=100 的记录
```

```sql
-- 事务A                              -- 事务B
START TRANSACTION;                    START TRANSACTION;

SELECT * FROM user_coupon              SELECT * FROM user_coupon
  WHERE user_id=1                       WHERE user_id=1
  AND coupon_id=100                     AND coupon_id=100
  FOR UPDATE;                           FOR UPDATE;
-- gap lock on (前一条, 后一条) ✅    -- gap lock on 同一间隙 ✅
-- gap lock 之间兼容！                 -- 都拿到了 gap lock

INSERT INTO user_coupon                 INSERT INTO user_coupon
  (user_id, coupon_id)                   (user_id, coupon_id)
  VALUES (1, 100);                       VALUES (1, 100);
-- 等待 B 的 gap lock ❌              -- 等待 A 的 gap lock ❌
-- → 死锁！
```

### 死锁图

```
事务A: gap lock ✅ → INSERT 等待 gap lock (B持有)
事务B: gap lock ✅ → INSERT 等待 gap lock (A持有)

A → B → A ← 环路
```

这是最经典的 gap lock 死锁模式：gap lock 之间兼容，导致两个事务都拿到了"保护盾"，然后都尝试往被保护的间隙插入，互相等待。

### 解决方案

**方案一：直接 INSERT，用唯一索引兜底（推荐）**

```sql
INSERT INTO user_coupon (user_id, coupon_id)
VALUES (1, 100)
ON DUPLICATE KEY UPDATE user_id = user_id;  -- 冲突时静默
```

不需要先 SELECT，唯一索引保证不会重复。

**方案二：SELECT ... LOCK IN SHARE MODE**

```sql
SELECT * FROM user_coupon
WHERE user_id=1 AND coupon_id=100
LOCK IN SHARE MODE;  -- 共享锁，不是 gap lock
```

`LOCK IN SHARE MODE` 在记录不存在时也会加 gap lock，但共享锁之间兼容，INSERT 时需要排他锁，仍可能死锁。**此方案不完全可靠。**

**方案三：应用层分布式锁**

```
1. Redis SETNX(user_coupon:1:100) → 获取锁
2. SELECT 检查是否存在
3. 不存在 → INSERT
4. 释放锁
```

减少数据库层面的并发冲突，但增加了依赖和复杂度。

### 最佳实践

- **不要用 SELECT FOR UPDATE 判断"不存在则插入"**，这是 gap lock 死锁的高发模式
- 依赖唯一索引 + ON DUPLICATE KEY UPDATE 或 INSERT IGNORE
- 如果必须先查，用分布式锁在应用层串行化

---

## I3. 唯一索引冲突死锁

### 场景

两个事务并发 INSERT 同一唯一键值，然后各自回滚/重试：

```sql
CREATE TABLE email_register (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) NOT NULL,
  UNIQUE INDEX uk_email (email)
) ENGINE=InnoDB;
```

```sql
-- 事务A                              -- 事务B
INSERT INTO email_register              INSERT INTO email_register
  (email) VALUES ('a@b.com');            (email) VALUES ('a@b.com');

-- A 先插入，获取排他锁
-- B 检测唯一冲突，在 'a@b.com' 上加 shared next-key lock
-- B 等待 A 释放排他锁

-- 如果此时事务C也尝试插入 'a@b.com'：
-- C 也加 shared next-key lock（与 B 的 shared lock 兼容）
-- C 等待 A 释放排他锁

-- A 回滚：
-- 'a@b.com' 记录被删除
-- B 和 C 都尝试重新插入
-- B 持有 shared lock → 等待 C 释放 shared lock 才能升级为排他锁
-- C 持有 shared lock → 等待 B 释放 shared lock 才能升级为排他锁
-- → 死锁！
```

### 死锁图

```
事务B: shared lock → 等待排他锁（需等C释放shared lock）
事务C: shared lock → 等待排他锁（需等B释放shared lock）

B → C → B ← 环路
```

### 解决方案

- 应用层做注册去重，减少同一 email 并发注册
- INSERT ON DUPLICATE KEY UPDATE，不报错
- 捕获 1062 Duplicate entry 错误，返回"已注册"提示

### 最佳实践

- 唯一索引冲突时的 shared lock 是 InnoDB 的实现细节，无法完全避免
- 应用层去重 + 唯一索引双重保障
- 注册场景允许少量死锁，捕获异常重试即可

---

## I4. 批量操作顺序不一致

### 场景

不同事务以不同 ORDER 批量更新同一组行：

```sql
-- 事务A：按价格排序，更新商品1和商品2
SELECT * FROM product ORDER BY price LIMIT 2 FOR UPDATE;
-- 按 price 排序：先锁 id=3(price=10)，再锁 id=1(price=20)

-- 事务B：按销量排序，更新商品1和商品2
SELECT * FROM product ORDER BY sales LIMIT 2 FOR UPDATE;
-- 按 sales 排序：先锁 id=1(sales=100)，再锁 id=3(sales=50)

-- A: 锁3 → 等待1
-- B: 锁1 → 等待3
-- → 死锁
```

### 解决方案

**所有批量操作按主键排序：**

```sql
-- 统一按 id 排序
SELECT * FROM product
WHERE id IN (1, 3)
ORDER BY id FOR UPDATE;
-- 无论什么业务需求，加锁顺序始终按 id
```

### 最佳实践

- 批量 SELECT FOR UPDATE **必须 ORDER BY 主键**
- 批量 UPDATE 如果涉及多行，按 id 升序逐行处理
- 应用层将待操作 ID 列表排序后再执行

---

## 死锁排查通用方法

### 1. 查看死锁日志

```sql
SHOW ENGINE INNODB STATUS\G
```

重点关注 `LATEST DETECTED DEADLOCK` 部分：

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
*** (1) TRANSACTION:        ← 事务1
TRANSACTION 12345, ACTIVE 2 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 10, OS thread handle 123456, query id 789 localhost root updating
UPDATE account SET balance = balance - 100 WHERE id = 2   ← 等待的SQL
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 58 page no 4 n bits 72 index PRIMARY of table `test`.`account`
trx id 12345 lock_mode X locks rec but not gap waiting   ← 等待排他record lock
Record lock, heap no 3 PHYSICAL RECORD: ...              ← 锁定的记录

*** (2) TRANSACTION:        ← 事务2
TRANSACTION 12346, ACTIVE 1 sec starting index read
mysql tables in use 1, locked 1
3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 11, OS thread handle 234567, query id 790 localhost root updating
UPDATE account SET balance = balance - 100 WHERE id = 1   ← 等待的SQL
*** (2) HOLDS THE LOCK(S):  ← 已持有的锁
RECORD LOCKS space id 58 page no 4 n bits 72 index PRIMARY of table `test`.`account`
trx id 12346 lock_mode X locks rec but not gap           ← 持有排他record lock
Record lock, heap no 3 PHYSICAL RECORD: ...

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 58 page no 4 n bits 72 index PRIMARY of table `test`.`account`
trx id 12346 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: ...

*** WE ROLL BACK TRANSACTION (2)  ← InnoDB 选择回滚事务2
```

### 2. 开启全量死锁日志

```sql
SET GLOBAL innodb_print_all_deadlocks = ON;
```

死锁信息写入 MySQL error log，便于历史分析。

### 3. 排查步骤

1. 从死锁日志中找到两个事务各自**持有**和**等待**的锁
2. 对照两个事务的 SQL，画出锁的等待关系图
3. 判断死锁类型（AB-BA / gap+insert / 唯一索引冲突）
4. 针对性地修改加锁顺序或业务逻辑

### 4. 应用层重试

死锁被 InnoDB 回滚后，应用层应重试：

```go
func WithRetry(fn func() error, maxRetries int) error {
    for i := 0; i < maxRetries; i++ {
        err := fn()
        if err == nil {
            return nil
        }
        if isDeadlockError(err) {
            log.Warn("deadlock detected, retrying", "attempt", i+1)
            continue
        }
        return err
    }
    return fmt.Errorf("max retries exceeded")
}

func isDeadlockError(err error) bool {
    return strings.Contains(err.Error(), "Deadlock found when trying to get lock")
}
```

MySQL 死锁错误码：`1213`（ER_LOCK_DEADLOCK）

---

## I 章总结

| 死锁类型 | 触发条件 | 根本解法 |
|---|---|---|
| AB-BA | 多行操作加锁顺序不一致 | 固定顺序加锁（按 id 升序） |
| Gap + Insert | SELECT FOR UPDATE 探测不存在后 INSERT | 唯一索引 + ON DUPLICATE KEY UPDATE |
| 唯一索引冲突 | 并发 INSERT 同一唯一键 | 应用层去重 + INSERT ON DUPLICATE |
| 批量顺序不一致 | 不同 ORDER BY 导致不同加锁顺序 | 统一按主键排序 |

**一条贯穿的规律：死锁的根本原因几乎都是"加锁顺序不一致"。固定加锁顺序是防止死锁的通用手段。**
