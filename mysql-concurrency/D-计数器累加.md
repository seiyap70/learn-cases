# D. 计数器 / 累加

高频累加操作是并发热点，容易产生性能瓶颈和数据不一致。

---

## D1. 文章阅读数 / 点赞数

### 场景

每秒数百次 `UPDATE SET count = count + 1`：

```sql
CREATE TABLE article (
  id BIGINT PRIMARY KEY,
  view_count INT NOT NULL DEFAULT 0,
  like_count INT NOT NULL DEFAULT 0
) ENGINE=InnoDB;
```

### 问题分析

**问题1：行锁争用**

```sql
-- 每次浏览都执行
UPDATE article SET view_count = view_count + 1 WHERE id = 1;
```

所有请求都要获取 id=1 的**排他 record lock**，串行执行。QPS 高时锁等待严重，响应时间飙升。

**问题2：binlog 量膨胀**

每次 +1 都产生一条 binlog，高并发下 binlog 写入成为瓶颈，主从复制延迟增大。

**问题3：计数不精确（极少见）**

InnoDB 的 `count = count + 1` 是原子操作（在锁保护下），不会丢失更新。但如果应用层先 SELECT 再 SET（而不是 SET count=count+1），就会丢失更新。

### 解决方案

**方案一：Redis 缓冲 + 定时批量写入（推荐）**

```
1. 每次浏览：Redis INCR article:view:1
2. 定时任务（每5秒）：读取 Redis 计数，批量 UPDATE 到 MySQL
   UPDATE article SET view_count = view_count + #{delta} WHERE id = 1;
3. 清零 Redis 缓冲
```

优点：
- MySQL 写入频率从每秒数百次降到每5秒1次
- 行锁持有时间极短
- binlog 量大幅减少

注意：Redis 和 MySQL 之间有短暂不一致，阅读数允许最终一致。

**方案二：独立计数表**

```sql
CREATE TABLE article_view_count (
  article_id BIGINT PRIMARY KEY,
  count INT NOT NULL DEFAULT 0
) ENGINE=InnoDB;
```

将计数从主表分离，避免更新计数时锁住主表行（影响其他字段的读取）。

**方案三：分段计数**

```sql
CREATE TABLE article_view_count (
  article_id BIGINT NOT NULL,
  slot TINYINT NOT NULL,       -- 0~9 共10个槽
  count INT NOT NULL DEFAULT 0,
  PRIMARY KEY (article_id, slot)
) ENGINE=InnoDB;
```

每次 +1 随机选一个 slot：

```sql
UPDATE article_view_count
SET count = count + 1
WHERE article_id = 1 AND slot = RAND()*10;
```

查询总计数：

```sql
SELECT SUM(count) FROM article_view_count WHERE article_id = 1;
```

将单行锁争用分散到 10 行，并发能力提升约 10 倍。

### 最佳实践

| QPS 量级 | 方案 |
|---|---|
| < 100 | 直接 `UPDATE SET count=count+1` |
| 100 ~ 1000 | 独立计数表 |
| > 1000 | Redis 缓冲 + 批量写入，或分段计数 |

- 阅读数等允许最终一致的场景，优先用 Redis 缓冲
- 需要强一致的计数，用分段计数分散锁争用
- 永远不要用 `SELECT count → 应用层+1 → UPDATE` 的方式

---

## D2. 账户余额累加

### 场景

账户余额被并发充值和扣款：

```sql
CREATE TABLE account (
  id BIGINT PRIMARY KEY,
  balance DECIMAL(18,2) NOT NULL
) ENGINE=InnoDB;
```

### 问题分析

与 D1 不同，余额**必须强一致**，不能用 Redis 缓冲或最终一致。

```
-- 并发操作
事务A: 充值 +500   → balance = balance + 500
事务B: 扣款 -200   → balance = balance - 200
事务C: 充值 +100   → balance = balance + 100
```

如果 balance 初始为 1000，最终必须为 1400，不能有任何丢失。

InnoDB 的 `UPDATE SET balance = balance + N` 在 record lock 保护下是原子的，不会丢失更新。但高并发下行锁争用严重。

### 解决方案

**方案一：流水表 + 同事务更新余额（推荐）**

```sql
CREATE TABLE account_journal (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  account_id BIGINT NOT NULL,
  amount DECIMAL(18,2) NOT NULL,    -- 正=充值，负=扣款
  balance_after DECIMAL(18,2),      -- 变更后余额（可选）
  created_at DATETIME NOT NULL,
  INDEX idx_account (account_id)
) ENGINE=InnoDB;
```

每次变动在同事务内写流水 + 更新余额：

```sql
START TRANSACTION;
INSERT INTO account_journal (account_id, amount, created_at)
VALUES (1, 500, NOW());

UPDATE account SET balance = balance + 500 WHERE id = 1;
COMMIT;
```

这个方案已经优化了锁争用：流水 INSERT 每次插入不同行，不争用行锁；只有余额 UPDATE 争用同一行的 record lock。

**单账户 TPS 上限测算：** 单行 record lock 下，一次 UPDATE 约 0.1-0.5ms，单账户 TPS 约 2000-10000。绝大多数业务单账户 TPS < 100，不会成为瓶颈。

**为什么不推荐异步汇总：**

异步汇总（流水表只写 INSERT，余额由定时任务汇总）本质是**最终一致**，与余额必须强一致的要求矛盾：

```
10:00:00  充值 +500，写入流水
10:00:01  用户查余额 → 返回旧值（定时任务未汇总）
10:00:01  扣款 WHERE balance >= 50 → 基于旧余额判断 → 超扣或误拒
```

余额是金融数据，异步汇总不可接受。

**方案二：分库分表（按 account_id）**

如果单账户 TPS 确实极高，将不同账户分散到不同数据库分片，各分片独立处理，互不影响。同一账户的操作仍在同一分片内强一致执行。

**方案三：子账户拆分**

将一个账户按业务维度拆成多个子余额行：

```sql
CREATE TABLE account_sub (
  id BIGINT PRIMARY KEY,
  account_id BIGINT NOT NULL,
  sub_type ENUM('CASH','BONUS','FROZEN'),
  balance DECIMAL(18,2) NOT NULL,
  UNIQUE INDEX uk_account_sub (account_id, sub_type)
) ENGINE=InnoDB;
```

不同子类型的操作锁不同行，并发度提升。查询总余额时聚合：

```sql
SELECT SUM(balance) FROM account_sub WHERE account_id = 1;
```

### 最佳实践

- 余额变更必须写流水表，这是审计要求也是对账基础
- **流水和余额更新在同一事务内**，保证强一致
- `UPDATE SET balance = balance + N` 是原子操作，不会丢失更新
- 单账户 TPS < 2000 时，直接同事务更新即可，不需要异步
- 单账户 TPS > 2000 时，从架构层面解决（分片/子账户），不要牺牲一致性
- 余额字段用 DECIMAL，不用 FLOAT/DOUBLE（精度问题）
- 扣款必须加 `balance >= amount` 条件防超扣

---

## D3. 分布式序列号生成

### 场景

多实例同时生成不重复的递增序列号（订单号、流水号等）。

### 方案

**方案一：AUTO_INCREMENT（最简单）**

```sql
CREATE TABLE orders (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  ...
) ENGINE=InnoDB;
```

InnoDB 的 AUTO_INCREMENT 使用 **auto-inc lock**：
- `innodb_autoinc_lock_mode = 0`：传统模式，每次 INSERT 都加表级 auto-inc lock
- `innodb_autoinc_lock_mode = 1`：连续模式，批量 INSERT 加锁，单行 INSERT 互斥
- `innodb_autoinc_lock_mode = 2`：交叉模式，不保证连续，性能最好

**方案二：号段模式（推荐）**

```sql
CREATE TABLE sequence (
  name VARCHAR(50) PRIMARY KEY,
  current_val BIGINT NOT NULL,
  step INT NOT NULL          -- 每次取的号段长度
) ENGINE=InnoDB;
```

应用启动时取一段号（如 1000 个），用完再取：

```sql
UPDATE sequence
SET current_val = current_val + step
WHERE name = 'order_id';
-- 返回 current_val - step + 1 到 current_val 的号段
```

优点：数据库访问频率降低 1000 倍，锁争用极低。

**方案三：Redis INCR**

```
Redis: SET order_id_seq 1000000
每次: Redis INCR order_id_seq → 返回唯一递增ID
```

优点：单线程原子操作，性能极高。
缺点：Redis 持久化策略影响可靠性（RDB 可能丢数据，AOF 较可靠）。

### 最佳实践

| 需求 | 方案 |
|---|---|
| 单调递增 + 连续 | AUTO_INCREMENT（lock_mode=1） |
| 单调递增 + 不要求连续 | AUTO_INCREMENT（lock_mode=2） |
| 高性能 + 不要求严格递增 | 号段模式 |
| 极高性能 + 允许偶尔跳号 | Redis INCR / Snowflake |

---

## D 章总结

| 场景 | 核心问题 | 推荐方案 |
|---|---|---|
| 阅读数/点赞数 | 行锁争用 + binlog 膨胀 | Redis 缓冲 + 批量写入 |
| 余额累加 | 强一致 + 行锁争用 | 流水表 + 同事务更新（不要异步汇总） |
| 序列号生成 | auto-inc lock 争用 | 号段模式 |

**一条贯穿的规律：高频累加的核心矛盾是单行锁争用，解法都是"分散写入"——缓冲、分段、号段，本质相同。但余额等金融数据必须强一致，只能从架构层面（分片/子账户）分散，不能用最终一致的缓冲方案。**
