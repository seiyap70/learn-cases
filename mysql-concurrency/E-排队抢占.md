# E. 排队 / 抢占

瞬时高并发竞争有限资源，是并发问题中最极端的场景。

---

## E1. 秒杀

### 场景

100 件商品，10000 人同时抢：

```sql
CREATE TABLE flash_sale (
  id BIGINT PRIMARY KEY,
  product_id BIGINT NOT NULL,
  total_count INT NOT NULL,           -- 总库存
  remaining_count INT NOT NULL        -- 剩余数量
) ENGINE=InnoDB;

CREATE TABLE flash_sale_order (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  sale_id BIGINT NOT NULL,
  user_id BIGINT NOT NULL,
  UNIQUE INDEX uk_sale_user (sale_id, user_id)  -- 防同一用户重复下单
) ENGINE=InnoDB;
```

### 问题分析

**问题1：数据库行锁串行化**

10000 人抢 100 件，都要获取 `flash_sale` 行的排他锁 → 99% 的请求在排队等锁，响应时间线性增长。

**问题2：连接池耗尽**

等锁的请求持有数据库连接不释放，连接池被占满，正常业务无法访问数据库。

### 解决方案

**方案一：Redis 原子预扣减 + MQ 异步落库（推荐）**

```
1. 活动开始前：加载库存到 Redis
   SET flash_sale:1:remaining 100

2. 抢购：用 Lua 脚本保证原子性
   local r = redis.call('DECR', KEYS[1])
   if r < 0 then
     redis.call('INCR', KEYS[1])  -- 恢复
     return -1                    -- 已抢完
   else
     return r                     -- 抢成功
   end

3. 抢成功后：发 MQ 消息，异步写数据库
   INSERT INTO flash_sale_order ...

4. 定时对账：Redis 余额 vs 数据库订单数
```

**为什么必须用 Lua 脚本：** `DECR` 返回值 < 0 时，在 `DECR` 和 `INCR` 之间如果应用崩溃，Redis 中的库存会停留在 -1，导致后续请求全部失败。Lua 脚本在 Redis 内原子执行，不存在这个窗口。

**MQ 消息丢失风险：** 如果 MQ 消息丢失，Redis 中已扣减但数据库无记录 → 库存泄漏。缓解措施：
- MQ 开启生产者确认（publisher confirm）
- 消费者幂等处理（唯一索引兜底）
- 定时对账：Redis remaining + 数据库订单数 = total_count，不一致则告警

**方案二：条件 UPDATE（直接打数据库时）**

```sql
UPDATE flash_sale
SET remaining_count = remaining_count - 1
WHERE id = 1 AND remaining_count > 0;
-- affected rows = 1 → 抢成功
-- affected rows = 0 → 已抢完
```

这不是"乐观锁"，没有 version 字段，不需要重试。它是**条件 UPDATE**——直接在 SET 中做减法，WHERE 中做库存校验，一条 SQL 原子完成。抢失败就是没了，不需要重试。

适用条件：QPS < 500，且能接受行锁串行化带来的响应时间增长。

**不推荐的方案：INSERT...SELECT COUNT**

```sql
-- ❌ 不要这样做
INSERT INTO flash_sale_order (sale_id, user_id)
SELECT 1, #{userId}
FROM DUAL
WHERE (SELECT COUNT(*) FROM flash_sale_order WHERE sale_id = 1) < 100;
```

问题：子查询是快照读，两个事务都看到 COUNT < 100，都 INSERT → 超过 100 条。唯一索引 `(sale_id, user_id)` 只防同一用户重复，**不防总记录数超限**。这个方案无法保证总量限制。

**不推荐的方案：SELECT FOR UPDATE**

```sql
-- ❌ 秒杀场景不要这样做
START TRANSACTION;
SELECT * FROM flash_sale WHERE id = 1 FOR UPDATE;
-- 应用层判断 remaining_count > 0
UPDATE flash_sale SET remaining_count = remaining_count - 1 WHERE id = 1;
COMMIT;
```

FOR UPDATE 获取排他锁后持有到事务结束。10000 个请求串行获取同一行的锁，99% 在等待。等锁期间持有数据库连接，连接池很快耗尽，**整个系统不可用**。

### 最佳实践

| 规模 | 方案 |
|---|---|
| < 500 QPS | 条件 UPDATE（`WHERE remaining_count > 0`） |
| 500 ~ 5000 QPS | Redis 原子预扣减 + MQ 异步落库 |
| > 5000 QPS | Redis 预扣减 + 前端限流 + MQ 异步落库 |

- 秒杀场景**不要让流量直接打到数据库**，Redis 承担流量过滤
- 抢失败的请求快速返回，不要重试
- 唯一索引 `(sale_id, user_id)` 防重复下单，兜底保障
- Redis 预扣减必须用 Lua 脚本，不要拆成 DECR + INCR
- MQ 异步落库需处理消息丢失风险：生产者确认 + 消费者幂等 + 定时对账

---

## E2. 抢红包

### 场景

100 元红包分 10 个，每人抢到随机金额：

```sql
CREATE TABLE red_packet (
  id BIGINT PRIMARY KEY,
  total_amount DECIMAL(18,2) NOT NULL,   -- 总金额
  total_count INT NOT NULL,              -- 总个数
  remaining_amount DECIMAL(18,2) NOT NULL,
  remaining_count INT NOT NULL
) ENGINE=InnoDB;

CREATE TABLE red_packet_claim (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  packet_id BIGINT NOT NULL,
  user_id BIGINT NOT NULL,
  amount DECIMAL(18,2) NOT NULL,
  UNIQUE INDEX uk_packet_user (packet_id, user_id)
) ENGINE=InnoDB;
```

### 与秒杀的区别

秒杀和抢红包看起来相似，但有一个关键区别：

| | 秒杀 | 抢红包 |
|---|---|---|
| 每份资源 | 固定价格（1件商品） | **随机金额** |
| 库存扣减 | remaining_count - 1 | remaining_count - 1 **且** remaining_amount - 随机值 |
| 一致性要求 | 数量不超即可 | 数量不超 **且** 总金额不超 **且** 最后一人分到剩余全部 |

随机金额分配需要保证：总领取金额 ≤ 总金额，且最后一人必须分到剩余全部金额，不能出现"剩余 0.01 元但还有 1 人未领"的情况。

### 解决方案

**Redis 预分配 + MQ 落库（推荐）**

红包场景更适合在创建时就预分配好每个红包的金额：

```
1. 创建红包时：预先计算好每个红包的金额，存入 Redis List
   RPUSH red_packet:1:amounts 30.50 12.30 5.00 ...  （10个金额，总和=100）

2. 抢红包：Lua 脚本原子弹出
   local amount = redis.call('LPOP', KEYS[1])
   if amount then
     return amount    -- 抢到，返回金额
   else
     return -1       -- 已抢完
   end

3. 抢成功后：MQ 异步写数据库
```

预分配保证了金额分配的正确性（总和 = 总金额），LPOP 保证了原子扣减。

**为什么不能在抢的时候实时计算随机金额：**

```
Lua 脚本中实时计算：
  remaining_amount = 100, remaining_count = 10
  随机金额 = random(0.01, remaining_amount / remaining_count * 2)

问题：
  - 二项分布随机，最后一人可能分到 0.01 元（体验差）
  - 浮点精度问题，总和可能不等于总金额
  - 需要特殊处理最后一人（补齐差额）
```

预分配方案在创建时一次性算好所有金额，避免实时计算的复杂性和精度问题。

### 最佳实践

- 红包金额**预分配**，不要实时随机计算
- 用 Redis List 存储 + LPOP 原子弹出
- 唯一索引 `(packet_id, user_id)` 防重复领取
- MQ 异步落库 + 定时对账

---

## E3. 选座 / 抢票

### 场景

电影选座、火车票，同一座位被多人同时选择：

```sql
CREATE TABLE seat (
  id BIGINT PRIMARY KEY,
  schedule_id BIGINT NOT NULL,    -- 场次/车次
  seat_no VARCHAR(10) NOT NULL,   -- 座位号
  status ENUM('AVAILABLE','LOCKED','SOLD'),
  locked_by VARCHAR(50),          -- 锁定者
  locked_at DATETIME,
  UNIQUE INDEX uk_schedule_seat (schedule_id, seat_no),
  INDEX idx_status_locked (status, locked_at)  -- 超时释放查询用
) ENGINE=InnoDB;
```

### 问题分析

与秒杀不同，选座有三个特点：
1. 用户需要**看到座位图**再选择 → 先读后写
2. 用户选了座位可能**放弃不买** → 需要锁座 + 超时释放
3. 一个用户可能选**多个座位** → 批量加锁

### 解决方案

**方案一：CAS 锁座（推荐）**

```sql
UPDATE seat
SET status = 'LOCKED',
    locked_by = 'user_123',
    locked_at = NOW()
WHERE schedule_id = 1
  AND seat_no = 'A3'
  AND status = 'AVAILABLE';
-- affected rows = 1 → 锁座成功
-- affected rows = 0 → 座位已被选
```

CAS 锁座不需要 FOR UPDATE，一条 UPDATE 原子完成：读当前状态 + 判断 + 修改。如果应用层需要知道锁座前后的完整信息，可以用 `RETURNING`（MySQL 8.0+）或再 SELECT 一次。

**方案二：Redis 分布式锁 + 数据库确认**

```
1. 用户选座：Redis SETNX seat:1:A3 user_123 EX 30
   → 成功：锁座
   → 失败：已被选

2. 用户确认购买：写数据库（UPDATE seat status='SOLD'）+ 删 Redis 锁

3. 超时未确认：Redis key 自动过期，数据库定时任务将 LOCKED 改回 AVAILABLE
```

**注意：Redis key 不能包含 user_id。** 如果 key 是 `seat:1:A3:user_123`，则 user_456 的 key 是 `seat:1:A3:user_456`，两人都能 SETNX 成功，都以为锁到了同一座位。key 应该是 `seat:1:A3`，value 是 user_id，这样第二个用户 SETNX 才会失败。

**超时释放（定时任务）：**

```sql
-- 释放超过5分钟未确认的锁座
-- idx_status_locked 索引支持此查询
UPDATE seat
SET status = 'AVAILABLE',
    locked_by = NULL,
    locked_at = NULL
WHERE status = 'LOCKED'
  AND locked_at < NOW() - INTERVAL 5 MINUTE;
```

必须加 `(status, locked_at)` 索引，否则全表扫描，扫描期间锁住大量行影响正常选座。

**多座位批量锁（按 seat_no 升序，防止死锁）：**

```sql
START TRANSACTION;
-- 按 id 升序加锁（比 seat_no 更可靠，主键天然有序）
SELECT * FROM seat
WHERE schedule_id = 1 AND seat_no IN ('A3', 'A4')
ORDER BY id FOR UPDATE;

-- 检查：两个座位是否都 AVAILABLE？
-- 是 → 批量锁座
-- 否 → 只锁可用的，或全部回滚（取决于业务需求）

UPDATE seat SET status = 'LOCKED', locked_by = 'user_123', locked_at = NOW()
WHERE schedule_id = 1 AND seat_no IN ('A3', 'A4') AND status = 'AVAILABLE';

-- 应用层检查 affected rows：
-- = 2 → 两个都锁成功
-- < 2 → 部分成功，回滚整个事务，提示用户重新选座
COMMIT;
```

**批量锁必须处理"部分成功"的情况。** 两个座位中如果只有一个 AVAILABLE，UPDATE 的 affected_rows = 1。应用层必须检查是否全部锁成功，否则回滚重试。不能只锁到一部分座位就提交。

### 最佳实践

- 单座位锁座用 CAS 式 UPDATE，不需要 FOR UPDATE
- 多座位按 id 升序 FOR UPDATE，防止死锁
- 批量锁座后检查 affected rows，部分成功则整体回滚
- 超时释放必须有 `(status, locked_at)` 索引支持
- Redis 锁的 key 不包含 user_id，value 存 user_id
- 用户确认购买后再改状态为 SOLD

---

## E4. 任务分发

### 场景

多 worker 并发从任务表领取待处理任务：

```sql
CREATE TABLE task (
  id BIGINT PRIMARY KEY,
  status ENUM('PENDING','PROCESSING','DONE','FAILED'),
  assigned_to VARCHAR(50),
  payload JSON,
  INDEX idx_status (status)
) ENGINE=InnoDB;
```

### 问题分析

**朴素实现（有问题）：**

```sql
-- Worker 查询待处理任务
SELECT * FROM task WHERE status = 'PENDING' LIMIT 1;

-- 领取任务
UPDATE task SET status = 'PROCESSING', assigned_to = 'worker_1' WHERE id = 123;
```

问题：两个 worker 同时 SELECT 到同一条 PENDING 任务，都尝试 UPDATE → 两个 worker 处理同一任务。

**即使 FOR UPDATE 也有问题：**

```sql
START TRANSACTION;
SELECT * FROM task WHERE status = 'PENDING' LIMIT 1 FOR UPDATE;
-- 如果 status 列没有索引，FOR UPDATE 走全表扫描 → 锁住所有行
-- 其他 worker 完全被阻塞
```

即使有索引，RR 级别下 `WHERE status = 'PENDING'` 会在二级索引上加 next-key lock，锁定范围可能比预期大。多个 worker 争抢同一条 PENDING 记录，只有第一个拿到锁，其余等待。

### 解决方案

**方案一：SELECT FOR UPDATE SKIP LOCKED（MySQL 8.0+，推荐）**

```sql
START TRANSACTION;
SELECT * FROM task
WHERE status = 'PENDING'
ORDER BY id
LIMIT 1
FOR UPDATE SKIP LOCKED;
-- SKIP LOCKED：跳过已被其他事务锁定的行
-- Worker A 锁住 id=123 → Worker B 跳过 123，自动选 id=124
-- 无需等待，无需重试

UPDATE task SET status = 'PROCESSING', assigned_to = 'worker_1'
WHERE id = ?;  -- 上一步选中的 ID
COMMIT;
```

这是任务分发场景的理想方案：
- 不等待：跳过被锁行，直接取下一个
- 不重试：每个 worker 天然拿到不同的任务
- `status` 列必须有索引，否则退化为全表扫描 + 全表锁

**方案二：条件 UPDATE（MySQL 5.7）**

```sql
UPDATE task
SET status = 'PROCESSING', assigned_to = 'worker_1'
WHERE id = (
  SELECT id FROM (
    SELECT id FROM task
    WHERE status = 'PENDING'
    ORDER BY id
    LIMIT 1
  ) AS tmp
)
AND status = 'PENDING';
-- affected rows = 1 → 领取成功
-- affected rows = 0 → 被其他 worker 抢走，需重试
```

**必须加外层 `AND status = 'PENDING'`。** 不加的话：子查询快照读选中 id=123（PENDING），但 worker A 已将其改为 PROCESSING，worker B 的 UPDATE 会把 id=123 的 assigned_to 覆盖为 'worker_2'，但 status 已经是 PROCESSING → 两个 worker 都以为自己在处理 id=123。

加了 `AND status = 'PENDING'` 后，worker B 的 UPDATE 匹配不到行（status 已不是 PENDING），affected_rows=0，知道被抢走了，重试即可。

缺点：冲突时需要重试，高并发下重试次数多。不如 SKIP LOCKED 高效。

**方案三：Redis 队列**

```
1. 任务入 Redis List：LPUSH task_queue task_id
2. Worker 领取：BRPOP task_queue 30  （阻塞30秒，无任务则等待）
3. 处理完成：更新数据库状态
```

需要处理的问题：

| 问题 | 解决 |
|---|---|
| Worker 领取后崩溃，任务丢失 | 可见性超时：BRPOP 后设超时，超时未确认则任务重新入队 |
| Redis 重启丢数据 | 使用 AOF 持久化（appendonly yes），或用 Redis Stream 替代 List |
| 任务无法按优先级排序 | 用多个 List（high/medium/low），优先 pop 高优先级 |
| 大量任务堆积 | Redis List 无分页，BRPOP 只能 pop 末端，考虑 Redis Stream |

### 最佳实践

| 方案 | 适用场景 | MySQL 版本 |
|---|---|---|
| FOR UPDATE SKIP LOCKED | 任务在数据库中，最推荐 | 8.0+ |
| 条件 UPDATE + 重试 | 任务在数据库中 | 5.7 |
| Redis List/Stream | 高吞吐量，任务不在数据库中 | 任意 |

- `status` 列必须加索引，否则 FOR UPDATE 锁全表
- 条件 UPDATE 子查询**必须**加外层 `AND status = 'PENDING'`
- 任务超时未完成需要重新入队（定时任务扫描 PROCESSING 状态超时的任务）
- Redis 队列需处理 worker 崩溃和 Redis 持久化

---

## E 章总结

| 场景 | 核心问题 | 推荐方案 |
|---|---|---|
| 秒杀 | 流量直接打 DB | Redis 原子预扣减（Lua）+ MQ 异步落库 |
| 抢红包 | 随机金额分配的一致性 | Redis 预分配金额 List + LPOP |
| 选座/抢票 | 锁座 + 超时 + 多座位 | CAS 锁座 + 超时释放（加索引）+ 批量锁检查 affected rows |
| 任务分发 | 重复领取 | FOR UPDATE SKIP LOCKED（MySQL 8.0+） |

**一条贯穿的规律：抢占场景的核心是"快速过滤"——让大部分请求在到达数据库之前就被拒绝，只让少量请求竞争数据库行锁。必须打到数据库的请求，用 SKIP LOCKED 或条件 UPDATE 减少锁等待。**
