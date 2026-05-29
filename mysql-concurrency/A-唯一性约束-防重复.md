# A. 唯一性约束 / 防重复

防止并发场景下产生重复数据，是最常见也最容易出错的并发问题之一。

---

## A1. 先查后插（不存在则插入）

### 场景

业务上要求某条记录只能存在一条，先查是否存在，不存在则插入：

```sql
-- 事务A
SELECT * FROM coupon_claim WHERE user_id=1 AND coupon_id=100;
-- 结果：空，判断未领取

-- 事务B（同时执行）
SELECT * FROM coupon_claim WHERE user_id=1 AND coupon_id=100;
-- 结果：空，判断未领取

-- 事务A
INSERT INTO coupon_claim (user_id, coupon_id) VALUES (1, 100);
-- 成功

-- 事务B
INSERT INTO coupon_claim (user_id, coupon_id) VALUES (1, 100);
-- 也成功！产生了重复记录
```

### 问题分析

SELECT 是快照读，不加锁。两个事务的 SELECT 互相不可见，都判断"不存在"，都执行 INSERT，产生重复。

这是**丢失更新**的一种变体——两个事务基于同一个"不存在"的判断做决策，导致约束被破坏。

时序分析：

```
时间  事务A                              事务B
 t1   SELECT → 空
 t2                                      SELECT → 空（A的INSERT未提交，不可见）
 t3   INSERT → 成功
 t4                                      INSERT → 成功（如果没有唯一索引）
 t5   COMMIT
 t6                                      COMMIT
      → 两条重复记录！
```

### 解决方案

**方案一：唯一索引（推荐）**

```sql
ALTER TABLE coupon_claim
  ADD UNIQUE INDEX uk_user_coupon (user_id, coupon_id);
```

事务 B 的 INSERT 会因唯一索引冲突失败，返回 `Duplicate entry` 错误。

唯一索引冲突时 InnoDB 的加锁行为（RR 级别）：
- INSERT 检测到唯一索引冲突，会在冲突记录上加 **shared next-key lock**
- 这会阻止其他事务在冲突位置附近插入，进一步减少并发重复的可能性

**方案二：SELECT FOR UPDATE + INSERT**

```sql
START TRANSACTION;
SELECT * FROM coupon_claim
  WHERE user_id=1 AND coupon_id=100 FOR UPDATE;
-- 若不存在，则插入
INSERT INTO coupon_claim (user_id, coupon_id) VALUES (1, 100);
COMMIT;
```

注意：如果记录不存在，FOR UPDATE 加的是 **gap lock**，不是 record lock。gap lock 之间可以共存（见前言），所以两个事务可能都获取了 gap lock 然后都尝试 INSERT → **死锁**。

**不推荐此方案**，除非配合唯一索引作为兜底。

**方案三：INSERT ... ON DUPLICATE KEY UPDATE**

```sql
INSERT INTO coupon_claim (user_id, coupon_id, claim_time)
VALUES (1, 100, NOW())
ON DUPLICATE KEY UPDATE claim_time = claim_time;  -- 冲突时不做实际更新
```

依赖唯一索引，冲突时不报错而是静默处理。适合"幂等插入"场景。

**ON DUPLICATE KEY UPDATE 的隐藏问题：**

| 问题 | 说明 |
|---|---|
| AUTO_INCREMENT 跳号 | 冲突走 UPDATE 路径时，预分配的自增 ID 浪费，频繁冲突导致大量跳号 |
| 触发器副作用 | 冲突时 BEFORE INSERT 仍会触发，然后才是 BEFORE UPDATE |
| affected_rows 歧义 | 返回 1=新插入，2=冲突且值有变化，0=冲突且值无变化，语义不直观 |
| 主从一致性（SBR） | statement-based replication 下主从数据不一致时，同一条 SQL 行为不同 |
| 锁范围更大 | 冲突时 shared next-key lock → exclusive lock 升级，锁持有时间更长 |

**方案四：INSERT IGNORE + 应用层判断**

```sql
INSERT IGNORE INTO coupon_claim (user_id, coupon_id, claim_time)
VALUES (1, 100, NOW());
-- affected rows = 1 → 新插入
-- affected rows = 0 → 唯一键冲突，被忽略
```

INSERT IGNORE 比 ON DUPLICATE KEY UPDATE 更轻量：冲突时不走 UPDATE，不触发 BEFORE UPDATE/AFTER UPDATE，锁行为更简单。但 affected_rows=0 无法区分"唯一键冲突"和"其他原因忽略"。

### 最佳实践

| 做法 | 推荐度 | 说明 |
|---|---|---|
| 唯一索引 | ★★★★★ | 根本解决，数据库层面保证约束 |
| INSERT + 捕获 1062 错误 | ★★★★ | 语义最清晰，affected_rows 明确，自增无浪费 |
| INSERT IGNORE | ★★★★ | 轻量幂等，适合不在乎"新建还是已存在"的场景 |
| ON DUPLICATE KEY UPDATE | ★★★ | 防重复可靠，但有自增跳号/触发器/affected_rows 歧义 |
| SELECT FOR UPDATE + INSERT | ★★ | 可能死锁，必须配合唯一索引兜底 |
| 应用层先查后插（无唯一索引） | ★ | 不可靠，并发下必然出问题 |

**选择原则：**
- 需要精确区分"新建"和"已存在"→ INSERT + 捕获 1062 错误
- 不在乎"新建还是已存在"→ INSERT IGNORE
- 冲突时需要更新某些字段 → ON DUPLICATE KEY UPDATE
- 自增 ID 连续性重要 → 不要用 ON DUPLICATE KEY UPDATE

---

## A2. 唯一键冲突插入

### 场景

有唯一索引的情况下，并发 INSERT 同一唯一键值：

```sql
CREATE TABLE user_email (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) NOT NULL,
  UNIQUE INDEX uk_email (email)
) ENGINE=InnoDB;

-- 事务A
INSERT INTO user_email (email) VALUES ('alice@example.com');
-- 成功，持有 uk_email 上 'alice@example.com' 的排他锁

-- 事务B
INSERT INTO user_email (email) VALUES ('alice@example.com');
-- 等待 A 释放锁
```

### 问题分析

InnoDB 处理唯一索引 INSERT 的内部流程：

1. 在二级索引（uk_email）上查找该值是否存在
2. **不存在**：正常插入，获取排他锁
3. **已存在**（冲突）：
   - 在冲突记录上加 **shared next-key lock**（S 型锁）
   - 等待持锁事务提交或回滚
   - 若持锁事务**提交** → 返回 `Duplicate entry` 错误
   - 若持锁事务**回滚** → 重新尝试插入

这个 shared next-key lock 会带来意想不到的副作用——

**死锁场景：**

```sql
-- 初始数据：email='bob@example.com' 存在

-- 事务A
INSERT INTO user_email (email) VALUES ('bob@example.com');
-- 唯一键冲突，在 'bob@example.com' 上加 shared next-key lock，等待

-- 事务B
INSERT INTO user_email (email) VALUES ('bob@example.com');
-- 同样唯一键冲突，请求 shared next-key lock
-- shared lock 之间兼容 → B 也获得了 shared lock ✅

-- 此时如果有人删除了 'bob@example.com' 这条记录：
-- DELETE FROM user_email WHERE email = 'bob@example.com';
-- → A 和 B 都在等待重新插入，可能形成死锁
```

### 解决方案

1. **INSERT ON DUPLICATE KEY UPDATE**：冲突时走更新逻辑，不报错
2. **应用层捕获 Duplicate entry 错误**，按业务语义处理（忽略 / 重试 / 返回提示）
3. **避免并发 INSERT 同一唯一键**：在应用层做请求去重（如 Redis SETNX）

### 最佳实践

- 唯一索引是数据库层面的最后防线，必须有
- 应用层也应做去重，减少到达数据库的冲突请求
- 捕获 `1062 Duplicate entry` 错误码，按业务处理，不要当作意外异常
- ON DUPLICATE KEY UPDATE 在"防重复"维度可靠，但需注意自增跳号、affected_rows 歧义等副作用
- 需要精确区分"新建还是已存在"时，INSERT + 捕获 1062 比 ON DUPLICATE KEY UPDATE 更合适

---

## A3. 幂等消费

### 场景

消息队列（Kafka/RabbitMQ）重复投递，同一业务 ID 被消费两次：

```
消息1: {order_id: "ORD001", action: "pay"}  → 消费者A处理
消息2: {order_id: "ORD001", action: "pay"}  → 消费者B处理（重复投递）
```

如果处理逻辑是"将订单状态改为已支付"，消费两次可能：
- 扣两次款
- 发两次货
- 记两条流水

### 问题分析

幂等消费本质上还是"防重复"，但比 A1/A2 复杂：
- 消息可能在**不同进程/机器**上消费，应用层锁无效
- 消费逻辑可能涉及多表操作，需要整体原子性
- 需要区分"真正的重复"和"相同业务 ID 的不同操作"

### 解决方案

**方案一：唯一索引 + INSERT FIRST**

```sql
-- 先插入幂等记录，插入成功才继续处理
INSERT INTO idempotent_log (biz_id, biz_type)
VALUES ('ORD001', 'ORDER_PAY');
-- 成功 → 继续业务逻辑
-- Duplicate entry → 说明已处理，跳过
```

将幂等记录插入作为**第一步**，利用唯一索引保证只有一个消费者能插入成功。

**方案二：状态机 + CAS**

```sql
UPDATE orders
SET status = 'PAID'
WHERE order_id = 'ORD001' AND status = 'UNPAID';
-- affected rows = 1 → 首次处理，继续
-- affected rows = 0 → 已处理过，跳过
```

利用状态的单向性保证幂等，不需要额外幂等表。

**方案三：分布式锁 + 数据库检查**

```
1. Redis SETNX(order_id) → 获取分布式锁
2. SELECT 检查是否已处理
3. 未处理 → 执行业务 → 标记已处理
4. 释放锁
```

分布式锁只是减少冲突，不能替代唯一索引。锁可能因超时/崩溃而失效。

### 最佳实践

| 层级 | 做法 | 作用 |
|---|---|---|
| 数据库 | 唯一索引 | 最后防线，绝对保证不重复 |
| 应用 | 幂等表 / 状态机 | 判断是否已处理，减少无效操作 |
| 基础设施 | 分布式锁 | 减少并发冲突到达数据库的频率 |
| 消费端 | 消费去重 | Kafka consumer 端按 key 去重，RabbitMQ 消息去重 |

**核心原则：数据库唯一索引是唯一可靠的幂等保证。** 应用层和基础设施层的去重是优化手段，不是保障手段。

---

## A 章总结

| 场景 | 核心问题 | 根本解法 |
|---|---|---|
| 先查后插 | 快照读不加锁，并发判断失效 | 唯一索引 |
| 唯一键冲突 | 冲突时的 shared lock 可能引发死锁 | INSERT + 捕获 1062 / INSERT IGNORE / ON DUPLICATE KEY UPDATE |
| 幂等消费 | 跨进程重复处理 | 唯一索引 + 幂等表 / 状态机 |

**一条贯穿的规律：靠查询判断来防重复是不可靠的，只有唯一索引才是数据库层面的硬保证。**
