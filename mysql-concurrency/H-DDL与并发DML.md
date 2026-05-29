# H. DDL 与并发 DML

DDL 操作（ALTER TABLE、CREATE INDEX 等）与并发的 DML 产生特殊类型的锁冲突，可能导致业务不可用。

---

## H1. Online DDL 阻塞

### 场景

线上表需要加字段/加索引，DDL 期间 DML 被阻塞：

```sql
-- 加索引
ALTER TABLE orders ADD INDEX idx_created_at (created_at);
```

### 问题分析

MySQL 5.6+ 支持 Online DDL，允许 DDL 期间并发 DML。但"Online"不等于"零影响"：

**Online DDL 三个阶段：**

```
阶段1: 初始化
  → 短暂锁表，获取 MDL 写锁
  → 释放 MDL 写锁，降级为 MDL 读锁

阶段2: 执行（主要耗时）
  → 扫描原表数据，构建新表结构
  → 并发 DML 的变更写入 Online Log
  → 此阶段 DML 可以并发执行 ✅

阶段3: 提交
  → 重新获取 MDL 写锁（短暂）
  → 应用 Online Log 中的增量变更
  → 替换原表
  → 释放 MDL 写锁
```

**阻塞点1：阶段1和阶段3的 MDL 写锁**

如果此时有长事务持有 MDL 读锁，DDL 无法获取 MDL 写锁：

```sql
-- 事务A（长事务）
START TRANSACTION;
SELECT * FROM orders LIMIT 1;  -- 持有 MDL 读锁
-- 事务未提交，MDL 读锁不释放

-- 事务B（DDL）
ALTER TABLE orders ADD INDEX idx_created_at (created_at);
-- 等待 MDL 写锁... 阻塞 ❌

-- 事务C（后续 DML）
SELECT * FROM orders WHERE id = 1;
-- 等待 DDL 的 MDL 写锁... 也阻塞 ❌
-- → 雪崩：所有 DML 都被阻塞
```

**阻塞点2：阶段2的并发度限制**

某些 DDL 操作不支持 Online：
- `ALGORITHM=COPY`：全表拷贝，全程锁表
- 修改列类型、更改字符集等操作不支持 Online

| DDL 操作 | 是否 Online | 是否锁表 |
|---|---|---|
| ADD INDEX | Yes | 不锁表 |
| ADD COLUMN | Yes（Instant 8.0.12+） | 不锁表 |
| DROP INDEX | Yes | 不锁表 |
| CHANGE COLUMN TYPE | No | 锁表 |
| CONVERT CHARACTER SET | No | 锁表 |
| ADD PRIMARY KEY | Yes | 不锁表 |
| DROP PRIMARY KEY | No | 锁表 |

### 解决方案

**方案一：pt-online-schema-change（推荐）**

Percona 工具，通过创建影子表 + 增量同步 + 原子替换实现无损 DDL：

```
1. 创建影子表 _orders_new（新结构）
2. 在原表上创建 AFTER INSERT/UPDATE/DELETE 触发器
3. 分批拷贝原表数据到影子表（小事务，不锁表）
4. 等待增量同步追平
5. RENAME TABLE 原子替换（极短锁表时间）
6. 删除旧表
```

**方案二：gh-ost（GitHub 工具）**

类似 pt-osc，但不用触发器，通过解析 binlog 同步增量变更：

```
1. 创建影子表
2. 拷贝全量数据
3. 连接从库解析 binlog，回放到影子表
4. 原子 RENAME 替换
```

优点：不用触发器，对主库性能影响更小。

**方案三：MySQL 8.0 Instant DDL**

```sql
-- MySQL 8.0.12+ 支持即时 DDL，只修改元数据，不拷贝数据
ALTER TABLE orders ADD COLUMN new_col INT, ALGORITHM=INSTANT;
-- 秒级完成，不影响 DML
```

支持 INSTANT 的操作有限（加列在末尾、改列默认值等），但覆盖了最常见的场景。

**方案四：设置 DDL 超时，避免无限等待**

```sql
-- 设置 DDL 等待 MDL 锁的超时时间
SET SESSION LOCK_WAIT_TIMEOUT = 5;  -- 5秒获取不到锁就放弃
ALTER TABLE orders ADD INDEX idx_created_at (created_at);
-- 如果5秒内拿不到 MDL 锁，DDL 自动失败，不影响后续 DML
```

### 最佳实践

- DDL 操作在业务低峰期执行
- 用 `LOCK_WAIT_TIMEOUT` 限制 DDL 等待时间，防止阻塞 DML
- 优先用 INSTANT DDL（MySQL 8.0.12+）
- 不支持 INSTANT 的用 pt-osc 或 gh-ost
- 监控 DDL 执行状态：`SHOW PROCESSLIST`，`performance_schema.`
- **永远不要在长事务存在时执行 DDL**

---

## H2. 元数据锁（MDL）冲突

### 场景

元数据锁（Metadata Lock）是 MySQL 5.5+ 引入的，保护表结构不被并发修改。问题在于 MDL 是**隐式获取**的，开发者常常意识不到：

```sql
-- 事务A（未提交的长事务）
START TRANSACTION;
SELECT * FROM orders WHERE id = 1;
-- 隐式获取 orders 表的 MDL 读锁
-- 事务未提交，MDL 读锁不释放

-- 事务B（DDL）
ALTER TABLE orders ADD COLUMN remark VARCHAR(100);
-- 等待 MDL 写锁...

-- 事务C（DML）
SELECT * FROM orders WHERE id = 2;
-- 等待事务B的 MDL 写锁...
-- → 全表阻塞
```

### 问题分析

**MDL 的规则：**

| 操作 | 获取的 MDL 锁 | 释放时机 |
|---|---|---|
| SELECT | MDL 共享读锁 | 事务结束 |
| DML（INSERT/UPDATE/DELETE） | MDL 共享写锁 | 事务结束 |
| DDL（ALTER/CREATE/DROP） | MDL 排他写锁 | 语句结束 |

**MDL 锁的兼容矩阵：**

| 请求 \ 持有 | MDL 共享读 | MDL 共享写 | MDL 排他写 |
|---|---|---|---|
| MDL 共享读 | ✅ | ✅ | ❌ |
| MDL 共享写 | ✅ | ✅ | ❌ |
| MDL 排他写 | ❌ | ❌ | ❌ |

关键：**MDL 排他写锁与任何锁都不兼容**，且 MDL 锁的获取有优先级——DDL 的排他写锁请求会阻塞后续所有 MDL 锁请求（包括读锁）。

这导致了"雪崩效应"：
1. 长事务持有 MDL 读锁
2. DDL 等待 MDL 写锁
3. 后续所有 DML 等待 DDL 的 MDL 写锁
4. 连接池耗尽，业务不可用

### 解决方案

**方案一：DDL 设置超时**

```sql
SET SESSION LOCK_WAIT_TIMEOUT = 5;
ALTER TABLE orders ADD INDEX idx_xxx (col);
-- 5秒拿不到锁就放弃，不影响业务
```

**方案二：监控并杀掉长事务**

```sql
-- 查看当前长事务
SELECT * FROM information_schema.INNODB_TRX
WHERE trx_started < NOW() - INTERVAL 30 SECOND;

-- 查看等待 MDL 锁的会话
SELECT * FROM performance_schema.metadata_locks
WHERE LOCK_STATUS = 'PENDING';

-- 杀掉阻塞的长事务
KILL #{blocking_thread_id};
```

**方案三：应用层保证事务短小**

- 事务内只做数据库操作，不要包含 RPC 调用、文件 IO 等
- 设置事务超时：`SET SESSION innodb_lock_wait_timeout = 5`
- 连接池配置合理的超时时间

**方案四：DDL 前检查**

```sql
-- 检查是否有长事务
SELECT trx_id, trx_state, trx_started, trx_query
FROM information_schema.INNODB_TRX
ORDER BY trx_started ASC;

-- 确认无长事务后再执行 DDL
ALTER TABLE orders ADD INDEX idx_xxx (col);
```

### 最佳实践

- **事务一定要及时提交**，这是防止 MDL 冲突的根本
- 事务内不要做耗时操作（RPC、文件 IO、用户交互）
- DDL 用 `LOCK_WAIT_TIMEOUT` 限制等待时间
- DDL 前检查长事务
- 监控 `performance_schema.metadata_locks` 中 PENDING 状态的锁
- MySQL 8.0+ 可设置 `lock_wait_timeout` 全局默认值

---

## H 章总结

| 场景 | 核心问题 | 推荐方案 |
|---|---|---|
| Online DDL | 长事务导致 MDL 冲突，DML 雪崩 | pt-osc / gh-ost / Instant DDL |
| MDL 冲突 | 长事务未提交，DDL 阻塞，DML 雪崩 | 短事务 + DDL 超时 + 监控 |

**一条贯穿的规律：DDL 与并发 DML 的问题，根源几乎都是"长事务"。治理长事务是解决这类问题的根本。**
