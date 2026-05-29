# G. 读写分离下的并发

主从复制架构引入了数据同步延迟，在并发场景下产生独特的一致性问题。

---

## G1. 写后读延迟

### 场景

用户提交表单后立即查询，读到旧数据：

```
应用 → 写主库
应用 → 读从库（主从同步延迟）→ 读到旧数据
```

```sql
-- 主库执行
UPDATE user_profile SET nickname = 'NewName' WHERE user_id = 1;

-- 立即从从库读
SELECT nickname FROM user_profile WHERE user_id = 1;
-- → 返回 'OldName'（主从同步尚未完成）
```

### 问题分析

MySQL 主从复制流程：

```
主库写入 binlog → dump thread 发送 → 从库 IO thread 接收 → 写 relay log
→ 从库 SQL thread 执行 → 数据生效
```

延迟来源：
- 主库 binlog 刷盘
- 网络传输
- 从库单线程回放（MySQL 5.6 以前）或多线程回放（5.7+，按库/按组并行）
- 大事务阻塞从库回放

典型延迟：正常 < 1ms，高峰期 10ms~数秒，大事务/从库压力下可达分钟级。

### 解决方案

**方案一：关键路径强制走主库（推荐）**

```go
// 写操作后短时间内，该用户的读请求走主库
func GetUserProfile(userID int64) {
    if isRecentlyWritten(userID) {  // 检查是否刚写过
        return readFromMaster(userID)
    }
    return readFromSlave(userID)
}
```

实现"刚写过"的判断：
- Redis 记录 `{user_id} → 写入时间戳`，TTL 1-3 秒
- 该用户的读请求在 TTL 内走主库

**方案二：中间件路由控制**

```go
// 使用读写分离中间件（如 ShardingSphere, Vitess）
// 在 SQL 中加 hint 强制走主库
db.Exec("/* #master */ SELECT * FROM user_profile WHERE user_id = ?", userID)
```

**方案三：等待从库同步（WAIT）

**

```sql
-- 写主库后，等待从库同步
SET @sync_result = MASTER_POS_WAIT(@binlog_file, @binlog_pos, 3);
-- 超时3秒，返回 NULL 表示超时
-- 返回 >= 0 表示从库已同步到该位置
```

或使用 MySQL 5.7+ 的 `WAIT_FOR_EXECUTED_GTID_SET`：

```sql
-- 写主库后获取 GTID
SELECT @@GLOBAL.GTID_EXECUTED INTO @gtid;

-- 等从库同步到该 GTID
SELECT WAIT_FOR_EXECUTED_GTID_SET(@gtid, 3);
-- 0 = 已同步，其他 = 超时或错误
```

**方案四：业务层补偿**

```
1. 写主库成功
2. 读从库，检查数据版本
3. 如果版本与预期不符 → 读主库
```

### 最佳实践

| 方案 | 复杂度 | 一致性 | 适用场景 |
|---|---|---|---|
| 关键路径走主库 | 低 | 最终一致 | 大多数业务 |
| 中间件路由 | 中 | 最终一致 | 有中间件的基础设施 |
| WAIT 同步 | 中 | 强一致 | 对一致性要求极高 |
| 业务层补偿 | 高 | 最终一致 | 特殊场景 |

- 写后读走主库是最简单有效的方案，用 Redis 标记"刚写过"的用户
- 标记 TTL 设为主从延迟的 2-3 倍（通常 1-3 秒）
- 不是所有读都需要走主库，只对"写后立即读"的场景特殊处理

---

## G2. 从库长事务

### 场景

从库执行大查询（报表、导出），阻塞主从复制回放：

```sql
-- 从库执行长查询
SELECT * FROM orders WHERE create_time > '2024-01-01';
-- 扫描 1000 万行，执行 30 秒

-- 同时从库 SQL thread 需要在 orders 表上回放主库的变更
-- 如果长查询持有 MDL 锁或行锁，SQL thread 被阻塞
-- → 主从延迟增大
```

### 问题分析

MySQL 5.7 多线程并行回放（基于组提交）改善了并行度，但：
- 同一 database 内的 DML 仍然可能串行
- 长查询的 MDL 锁会阻塞 DDL 回放
- 行锁冲突导致 SQL thread 等待

关键指标：`Seconds_Behind_Master` 持续增大。

### 解决方案

**方案一：长查询走专用只读从库**

```
主库 → 从库1（正常读写分离，禁止长查询）
     → 从库2（专用报表/导出，允许长查询）
     → 从库3（专用报表/导出，轮换使用）
```

从库2的延迟是可接受的，不影响从库1的正常业务。

**方案二：长查询加 MAX_EXECUTION_TIME**

```sql
-- 限制查询最长执行时间
SET SESSION MAX_EXECUTION_TIME = 30000;  -- 30秒超时

SELECT * FROM orders WHERE create_time > '2024-01-01';
-- 超过30秒自动终止
```

**方案三：分页查询代替全量查询**

```sql
-- 不要一次性查全量
SELECT * FROM orders WHERE create_time > '2024-01-01';

-- 分页查询，每页短事务
SELECT * FROM orders
WHERE create_time > '2024-01-01'
ORDER BY id LIMIT 1000 OFFSET 0;

SELECT * FROM orders
WHERE create_time > '2024-01-01' AND id > #{last_id}
ORDER BY id LIMIT 1000;  -- 游标分页，更高效
```

**方案四：ClickHouse 等分析引擎**

报表和导出场景不适合 MySQL，迁移到列式存储分析引擎：
- 数据通过 CDC（Canal/Debezium）同步到 ClickHouse
- 长查询在 ClickHouse 执行，不影响 MySQL 从库

### 最佳实践

- 生产从库**禁止**长查询，用 `max_execution_time` 限制
- 长查询走专用报表从库或分析引擎
- 分页查询代替全量查询
- 监控 `Seconds_Behind_Master`，超过阈值告警
- MySQL 8.0+ 可用 `performance_schema.replication_applier_status_by_worker` 监控回放进度

---

## G 章总结

| 场景 | 核心问题 | 推荐方案 |
|---|---|---|
| 写后读延迟 | 主从同步延迟导致读到旧数据 | 关键路径走主库 + Redis 标记 |
| 从库长事务 | 阻塞复制回放，延迟增大 | 专用报表从库 / 分页查询 / 分析引擎 |

**一条贯穿的规律：读写分离的核心矛盾是"一致性 vs 性能"——走主库保证一致但牺牲读扩展性，走从库扩展性好但有一致性延迟。根据业务对一致性的要求做取舍。**
