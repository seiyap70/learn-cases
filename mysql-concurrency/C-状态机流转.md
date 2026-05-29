# C. 状态机流转

业务对象的状态变更必须遵循预定义的流转规则，并发下容易出现跳步、回退、重复流转。

---

## C1. 订单状态变更

### 场景

订单状态流转：`PENDING → PAID → SHIPPED → COMPLETED`

```sql
CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  status ENUM('PENDING','PAID','SHIPPED','COMPLETED','CANCELLED'),
  amount DECIMAL(10,2)
) ENGINE=InnoDB;
```

### 常见问题

**问题1：跳步**

```sql
-- 用户支付成功回调和管理员手动发货并发
-- 期望：PENDING → PAID → SHIPPED
-- 实际：发货请求将 PENDING 直接改为 SHIPPED，跳过 PAID

UPDATE orders SET status = 'SHIPPED' WHERE id = 1;
-- 没有检查当前状态，直接跳步！
```

**问题2：回退**

```sql
-- 订单已发货，用户申请取消
-- 期望：已发货不可取消
-- 实际：取消请求将 SHIPPED 改为 CANCELLED

UPDATE orders SET status = 'CANCELLED' WHERE id = 1;
-- 没有检查状态是否允许取消！
```

**问题3：重复流转**

```sql
-- 支付回调被重复投递，执行两次 PENDING → PAID
-- 虽然幂等（两次结果一样），但如果 PAID 有副作用（扣款、发消息），就会重复执行
```

### 解决方案

**方案一：CAS 式状态更新（推荐）**

```sql
UPDATE orders
SET status = 'PAID'
WHERE id = 1 AND status = 'PENDING';
-- affected rows = 1 → 流转成功
-- affected rows = 0 → 状态已变更（已支付/已取消/其他），跳过或报错
```

每次状态变更都在 WHERE 中限定**当前状态**，只有满足条件才能变更。

**方案二：状态机校验表**

```sql
CREATE TABLE order_status_transition (
  from_status VARCHAR(20),
  to_status VARCHAR(20),
  PRIMARY KEY (from_status, to_status)
);

INSERT INTO order_status_transition VALUES
  ('PENDING', 'PAID'),
  ('PENDING', 'CANCELLED'),
  ('PAID', 'SHIPPED'),
  ('SHIPPED', 'COMPLETED'),
  ('PAID', 'CANCELLED');  -- 退款取消
```

应用层查询允许的流转，校验后再 UPDATE：

```sql
-- 先校验
SELECT 1 FROM order_status_transition
WHERE from_status = 'PENDING' AND to_status = 'SHIPPED';
-- 空结果 → 不允许跳步

-- 再更新
UPDATE orders SET status = 'PAID'
WHERE id = 1 AND status = 'PENDING';
```

**方案三：SELECT FOR UPDATE + 应用层校验**

```sql
START TRANSACTION;
SELECT status FROM orders WHERE id = 1 FOR UPDATE;
-- 获取排他锁，读最新状态

-- 应用层判断：当前状态是否允许流转到目标状态
-- PENDING → PAID ✅
-- PENDING → SHIPPED ❌

UPDATE orders SET status = 'PAID' WHERE id = 1;
COMMIT;
```

适用于状态流转前需要做复杂业务判断的场景（如校验支付金额、库存等）。

### 最佳实践

- **WHERE 中必须限定当前状态**，这是最基本也最有效的防护
- 状态定义用 ENUM 或 CHECK 约束，防止非法值
- 副作用（扣款、发消息）的触发也以 `affected rows = 1` 为条件
- 状态流转日志单独记录，便于排查和回溯

---

## C2. 审批流

### 场景

多人审批同一工单，如请假审批：`提交 → 主管审批 → HR审批 → 完成`

```sql
CREATE TABLE approval (
  id BIGINT PRIMARY KEY,
  current_step INT NOT NULL,       -- 当前审批步骤
  total_steps INT NOT NULL,        -- 总步骤数
  status ENUM('PENDING','APPROVED','REJECTED')
) ENGINE=InnoDB;
```

### 常见问题

**问题1：同一步骤被审批两次**

```sql
-- 主管A和主管B同时审批"主管审批"步骤
UPDATE approval SET status = 'APPROVED', current_step = 2
WHERE id = 1;
-- 两人都执行成功，但只应有一次审批有效
```

**问题2：跳步审批**

```sql
-- HR 直接审批了 step=2（本应主管先审批）
UPDATE approval SET current_step = 3 WHERE id = 1;
-- 跳过了主管审批步骤
```

### 解决方案

**CAS 式审批：**

```sql
-- 主管审批
UPDATE approval
SET current_step = 2, status = 'PENDING'
WHERE id = 1 AND current_step = 1 AND status = 'PENDING';
-- affected rows = 1 → 审批成功，流转到下一步
-- affected rows = 0 → 已被其他人审批或状态已变
```

**审批记录去重：**

```sql
CREATE TABLE approval_log (
  approval_id BIGINT NOT NULL,
  step INT NOT NULL,
  approver VARCHAR(50) NOT NULL,
  action ENUM('APPROVE','REJECT'),
  created_at DATETIME NOT NULL,
  PRIMARY KEY (approval_id, step, approver)
) ENGINE=InnoDB;
```

联合主键保证同一审批人在同一步骤只能审批一次。

### 最佳实践

- 每步审批用 CAS 式 UPDATE，WHERE 中限定 `current_step`
- 审批记录表用唯一索引防重复审批
- 并发审批时，先审批者成功，后审批者得到 `affected rows = 0`，提示"已处理"

---

## C3. 账户启用/禁用

### 场景

管理员禁用账户的同时，用户正在操作（登录、交易等），产生冲突：

```sql
CREATE TABLE account (
  id BIGINT PRIMARY KEY,
  status ENUM('ACTIVE','DISABLED'),
  balance DECIMAL(10,2)
) ENGINE=InnoDB;
```

### 常见问题

```sql
-- 管理员禁用账户
UPDATE account SET status = 'DISABLED' WHERE id = 1;

-- 同时用户发起交易（扣款）
UPDATE account SET balance = balance - 100 WHERE id = 1;
-- 扣款成功，但账户已被禁用，不应该允许交易
```

### 解决方案

**业务操作中检查状态：**

```sql
-- 扣款时限定账户状态
UPDATE account
SET balance = balance - 100
WHERE id = 1 AND status = 'ACTIVE' AND balance >= 100;
-- affected rows = 0 → 账户已禁用或余额不足
```

**SELECT FOR UPDATE + 校验：**

```sql
START TRANSACTION;
SELECT status, balance FROM account WHERE id = 1 FOR UPDATE;
-- status = ACTIVE 且 balance >= 100 → 允许
-- status = DISABLED → 拒绝

UPDATE account SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

### 最佳实践

- 关键业务操作（交易、提现等）SQL 中必须加 `status = 'ACTIVE'` 条件
- 禁用操作用 `SELECT FOR UPDATE` 确保读到最新状态
- 禁用后未完成的交易应在应用层做补偿/回滚

---

## C 章总结

| 场景 | 核心问题 | 推荐方案 |
|---|---|---|
| 订单状态变更 | 跳步/回退/重复流转 | CAS 式 UPDATE，WHERE 限定当前状态 |
| 审批流 | 同步重复审批/跳步 | CAS + 审批记录唯一索引 |
| 账户启用/禁用 | 禁用后仍可交易 | 业务操作 SQL 加状态条件 |

**一条贯穿的规律：状态流转必须用 CAS 式 UPDATE，WHERE 条件限定"从哪个状态变更"，让数据库来保证状态机的正确性。**
