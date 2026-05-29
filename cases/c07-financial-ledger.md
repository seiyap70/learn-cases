# C07: 金融级账户余额系统

## 业务场景

某第三方支付平台的核心账户系统，管理所有用户的资金余额。这是整个支付平台最核心的系统——出错意味着资金损失，而资金损失在金融行业是不被容忍的。

**业务操作：**
- 充值：用户从银行卡充值到平台余额
- 提现：用户从平台余额提现到银行卡
- 转账：用户之间转账
- 支付：用户向商户付款
- 退款：商户向用户退款
- 结算：平台向商户结算

**已知数据：**
- 用户数：5000 万
- 日均交易笔数：2000 万笔
- 峰值 TPS：5000（早晚高峰）
- 单笔交易金额：0.01 元 - 50 万元
- 资金安全要求：零差错，任何情况下余额不能为负
- 日终对账：内部账与银行账必须一致，差异 < 0.01 元

**为什么这个系统特别难？**

金融系统有几个独特的约束，让常规的"高并发"方案全部失效：

1. **余额不能为负** —— 电商超卖可以赔付，但余额为负意味着用户花了不存在的钱，这是金融犯罪
2. **精度要求极高** —— `0.1 + 0.2 = 0.30000000000000004` 在金融系统中是不可接受的
3. **审计不可绕过** —— 每一分钱必须有来源和去向，监管随时可能审查
4. **不可删除修改** —— 任何数据一旦写入不可修改，只能追加

## 核心挑战

### 挑战 1：并发扣款导致余额为负

**场景：** 用户余额 100 元，同时发起两笔 80 元的扣款。

```
时刻 T1: 请求A 读取余额 = 100
时刻 T2: 请求B 读取余额 = 100
时刻 T3: 请求A 扣款 100 - 80 = 20 ✓
时刻 T4: 请求B 扣款 100 - 80 = 20 ✓

结果：两笔扣款都成功，但 100 - 80 - 80 = -60，余额为负！
```

这不是理论问题——2016 年某支付平台就因为此 bug 导致大量用户余额为负，最终赔付数百万。

### 挑战 2：浮点精度

```python
# Python 中的浮点精度问题
>>> 0.1 + 0.2
0.30000000000000004
>>> 0.1 + 0.2 == 0.3
False

# 金融场景下
>>> account_balance = 0.0
>>> for _ in range(1000000):
...     account_balance += 0.1
>>> account_balance
100000.0000000111  # 应该是 100000.00，差了 0.0000000111
```

0.0000000111 元看似微小，但经过上亿次运算后，累积误差可达数元甚至数十元。金融系统中 1 分钱的差异都会导致对账失败。

### 挑战 3：数据不可变与审计

传统数据库设计中，UPDATE 是常见操作。但金融系统中直接 UPDATE 余额是危险的：
- 无法追溯余额是怎么变化的
- 如果 UPDATE 出错，历史数据被覆盖，无法恢复
- 审计时无法回答"用户 X 在时刻 T 的余额是多少"

### 挑战 4：对账闭环

每天的交易需要与银行对账。如果内部记录显示用户充值 100 元，但银行只记录了 99.99 元（因为精度问题），对账就不通过。

## 设计约束

- 数据库：MySQL（需强一致性，金融系统不能接受最终一致性）
- 不可使用 Redis 存储余额（内存数据不可靠，宕机后可能丢失，且不满足审计要求）
- 单用户 TPS 上限约 200（同一用户的高频扣款场景——如自动续费、批量代扣）
- 所有金额运算必须精确到分，使用 DECIMAL 类型
- 数据不可删除或修改，只能追加

## 请先独立思考（限时 45 分钟）

1. 设计一个表结构，使得：余额不能为负、每笔交易可追溯、支持高并发扣款。写出完整的 CREATE TABLE 语句。
2. 写出扣款 80 元的完整 SQL 和应用层代码，确保在并发场景下不会出现余额为负。至少考虑两种方案（乐观锁 vs 行锁）。
3. 如果不用直接 UPDATE 余额，而是用事件溯源（只追加事件，余额由事件计算），如何处理"查询当前余额"的性能问题？
4. 设计日终对账的完整流程。假设内部有 2000 万笔交易记录，银行提供了一个 2000 万行的对账文件。

---

## 设计解析

### 核心设计原则：事件溯源 + 聚合快照

**原则 1：余额不是存储的值，而是计算的结果**

传统方案直接存储余额：
```sql
-- 危险方案
UPDATE accounts SET balance = balance - 80 WHERE user_id = 'U12345';
```

事件溯源方案只追加事件：
```sql
-- 安全方案：追加事件
INSERT INTO account_events (account_id, event_type, amount, transaction_id)
VALUES ('U12345', 'WITHDRAW', 80.00, 'TXN-001');
```

余额 = 初始余额 + SUM(充值事件) - SUM(扣款事件)

**为什么这更安全？**
- 事件是追加写入，不存在并发覆盖问题
- 任何时刻都可以重新计算余额（可验证）
- 审计时可以看到每一笔资金变动

**原则 2：用快照加速查询，用事件保证正确性**

如果每次查余额都要 SUM 几千条事件，性能不可接受。引入快照：

```
快照（1小时前）：余额 = 5000.00
事件（1小时内）：充值100, 支付80, 支付30
当前余额 = 5000 + 100 - 80 - 30 = 4990.00
```

### 完整表结构设计

```sql
-- 账户表（基本信息）
CREATE TABLE accounts (
    account_id VARCHAR(32) PRIMARY KEY,
    user_id VARCHAR(32) NOT NULL,
    account_type VARCHAR(16) NOT NULL,     -- PERSONAL / MERCHANT / PLATFORM
    currency CHAR(3) DEFAULT 'CNY',
    status VARCHAR(16) DEFAULT 'ACTIVE',   -- ACTIVE / FROZEN / CLOSED
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_user (user_id)
);

-- 账户事件表（追加写入，永不修改/删除）
CREATE TABLE account_events (
    event_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id VARCHAR(32) NOT NULL,
    event_type VARCHAR(20) NOT NULL,       -- RECHARGE / WITHDRAW / TRANSFER_IN / TRANSFER_OUT / REFUND / SETTLEMENT
    amount DECIMAL(15,2) NOT NULL,         -- 正数=入账, 负数=出账
    currency CHAR(3) DEFAULT 'CNY',
    transaction_id VARCHAR(64) NOT NULL,   -- 关联业务交易ID
    counter_account_id VARCHAR(32),        -- 对方账户ID（转账场景）
    biz_type VARCHAR(32),                  -- 业务类型（PAYMENT/REFUND/SETTLEMENT/...）
    biz_no VARCHAR(64),                    -- 业务单号
    version BIGINT NOT NULL,               -- 乐观锁版本号（每账户递增）
    remark VARCHAR(255),
    created_at TIMESTAMP(3) DEFAULT CURRENT_TIMESTAMP(3),

    -- 幂等键：同一交易+同一事件类型只能写入一次
    UNIQUE KEY uk_txn_event (transaction_id, event_type),
    
    -- 按账户+版本查询（获取最新N条事件）
    INDEX idx_account_version (account_id, version),
    
    -- 按时间查询（对账用）
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 账户快照表（定期更新，加速余额查询）
CREATE TABLE account_snapshots (
    account_id VARCHAR(32) NOT NULL,
    snapshot_time TIMESTAMP(3) NOT NULL,
    balance DECIMAL(15,2) NOT NULL,
    last_event_id BIGINT NOT NULL,         -- 快照对应最后一条事件ID
    last_version BIGINT NOT NULL,          -- 快照版本
    created_at TIMESTAMP(3) DEFAULT CURRENT_TIMESTAMP(3),
    
    PRIMARY KEY (account_id, snapshot_time),
    INDEX idx_account_latest (account_id, snapshot_time DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 余额查询的完整实现

```python
class AccountService:
    def get_balance(self, account_id):
        """
        查询当前余额：快照 + 增量事件
        """
        # 1. 获取最新快照
        snapshot = self.db.query_one("""
            SELECT balance, last_event_id, last_version
            FROM account_snapshots
            WHERE account_id = %s
            ORDER BY snapshot_time DESC LIMIT 1
        """, account_id)

        if snapshot is None:
            # 无快照，从全部事件计算（首次查询或快照被清理）
            return self._calculate_from_events(account_id)

        # 2. 获取快照之后的事件
        delta = self.db.query_one("""
            SELECT 
                COALESCE(SUM(CASE WHEN event_type IN ('RECHARGE','TRANSFER_IN','REFUND','SETTLEMENT')
                             THEN amount ELSE -amount END), 0) AS delta_balance
            FROM account_events
            WHERE account_id = %s AND event_id > %s
        """, account_id, snapshot.last_event_id)

        # 3. 余额 = 快照 + 增量
        return snapshot.balance + delta.delta_balance
```

**性能分析：**
- 快照查询：< 1ms（主键索引）
- 增量事件查询：< 5ms（account_id + event_id 范围索引）
- 总延迟：< 10ms

**快照更新频率：** 每 1000 条事件或每 5 分钟更新一次快照。活跃账户每小时约 100 笔交易，5 分钟约 8 笔，增量计算极快。

### 扣款的完整实现（保证余额不为负）

**方案 A：悲观锁（行锁）——最安全但性能略低**

```python
class AccountService:
    def withdraw(self, account_id, amount, transaction_id, biz_type):
        """
        扣款：使用行锁保证余额不为负
        """
        # 1. 开启事务
        with self.db.transaction() as tx:
            # 2. 加行锁读取当前余额
            snapshot = tx.query_one("""
                SELECT balance, last_event_id, last_version
                FROM account_snapshots
                WHERE account_id = %s
                ORDER BY snapshot_time DESC LIMIT 1
                FOR UPDATE
            """, account_id)

            # 3. 计算当前余额（快照 + 增量）
            delta = tx.query_one("""
                SELECT COALESCE(SUM(CASE WHEN event_type IN ('RECHARGE','TRANSFER_IN','REFUND','SETTLEMENT')
                                       THEN amount ELSE -amount END), 0) AS delta_balance
                FROM account_events
                WHERE account_id = %s AND event_id > %s
            """, account_id, snapshot.last_event_id)
            
            current_balance = snapshot.balance + delta.delta_balance

            # 4. 检查余额是否充足
            if current_balance < amount:
                raise InsufficientBalanceError(
                    f"余额不足: current={current_balance}, required={amount}"
                )

            # 5. 写入扣款事件（幂等）
            next_version = snapshot.last_version + 1  # 简化，实际需考虑增量事件
            try:
                tx.execute("""
                    INSERT INTO account_events 
                    (account_id, event_type, amount, transaction_id, biz_type, version)
                    VALUES (%s, 'WITHDRAW', %s, %s, %s, %s)
                """, account_id, amount, transaction_id, biz_type, next_version)
            except DuplicateKeyError:
                # 幂等：同一 transaction_id + event_type 已存在
                return self.get_event_result(transaction_id, 'WITHDRAW')

            # 6. 更新快照（将当前余额和增量合并到新快照）
            new_balance = current_balance - amount
            last_event_id = tx.last_insert_id()
            tx.execute("""
                INSERT INTO account_snapshots 
                (account_id, snapshot_time, balance, last_event_id, last_version)
                VALUES (%s, NOW(3), %s, %s, %s)
            """, account_id, new_balance, last_event_id, next_version)

        return {"balance": new_balance, "event_id": last_event_id}
```

**`FOR UPDATE` 为什么能保证不超扣？**

`FOR UPDATE` 对快照行加排他锁。同一账户的两个扣款请求：
- 请求 A 获取锁 → 读取余额 100 → 扣 80 → 余额 20 → 释放锁
- 请求 B 等待锁 → 获取锁 → 读取余额 20 → 余额不足 → 拒绝

`FOR UPDATE` 的代价：同一账户的扣款请求串行执行。单账户 TPS 上限约 200（单次事务 5ms）。

**方案 B：乐观锁——无锁但需重试**

```python
class AccountService:
    def withdraw_optimistic(self, account_id, amount, transaction_id, biz_type):
        """
        扣款：使用乐观锁（版本号）保证余额不为负
        适用于低冲突场景（同一用户并发扣款概率低）
        """
        max_retries = 3
        
        for attempt in range(max_retries):
            # 1. 读取当前余额和版本号（不加锁）
            current_balance, current_version = self.get_balance_with_version(account_id)

            # 2. 检查余额是否充足
            if current_balance < amount:
                raise InsufficientBalanceError()

            # 3. 写入事件（条件：版本号未变）
            # 使用 INSERT + 乐观锁快照更新
            new_balance = current_balance - amount
            new_version = current_version + 1

            # 3a. 写入事件（幂等保护）
            try:
                self.db.execute("""
                    INSERT INTO account_events 
                    (account_id, event_type, amount, transaction_id, biz_type, version)
                    VALUES (%s, 'WITHDRAW', %s, %s, %s, %s)
                """, account_id, amount, transaction_id, biz_type, new_version)
            except DuplicateKeyError:
                return self.get_event_result(transaction_id, 'WITHDRAW')

            # 3b. 更新快照（CAS 操作）
            affected = self.db.execute("""
                UPDATE account_snapshots 
                SET balance = %s, last_version = %s
                WHERE account_id = %s AND last_version = %s
            """, new_balance, new_version, account_id, current_version)

            if affected > 0:
                # 更新成功
                return {"balance": new_balance}
            else:
                # 版本冲突，重试
                # 删除刚写入的事件（因为快照未更新成功）
                self.db.execute("""
                    DELETE FROM account_events 
                    WHERE transaction_id = %s AND event_type = 'WITHDRAW'
                """, transaction_id)
                continue

        raise ConcurrentConflictError("乐观锁重试耗尽")
```

**乐观锁的问题：** 在高冲突场景下（同一用户频繁扣款），重试概率高，性能可能不如悲观锁。

**方案选择建议：**
- 个人用户（低冲突）→ 乐观锁
- 商户账户（高冲突，结算频繁）→ 悲观锁
- 平台账户（极高冲突）→ 分账户（按业务线拆分）

### DECIMAL 精度保证

**MySQL DECIMAL 类型：**

```sql
-- DECIMAL(15,2) 的含义：
-- 总共 15 位数字，其中 2 位小数
-- 最大值：999,999,999,999,999.99
-- 精度：精确到分（0.01 元）

-- 验证精度
SELECT 0.1 + 0.2;           -- 浮点：0.30000000000000004
SELECT DECIMAL('0.1') + DECIMAL('0.2');  -- DECIMAL：0.3 ✓
```

**应用层精度保证（Python）：**

```python
from decimal import Decimal, ROUND_HALF_UP

# 正确：使用 Decimal
amount = Decimal('80.00')
balance = Decimal('100.00')
new_balance = balance - amount  # 精确结果：20.00

# 错误：使用 float
amount = 80.00  # float!
balance = 100.00  # float!
new_balance = balance - amount  # 可能不精确

# 金额运算必须使用 Decimal
class Money:
    """金额值对象，强制使用 Decimal"""
    def __init__(self, value):
        if isinstance(value, float):
            raise TypeError("不可使用 float 构造金额，请使用字符串: Money('80.00')")
        self.value = Decimal(str(value))

    def __sub__(self, other):
        result = self.value - other.value
        # 金额精度：保留 2 位小数，四舍五入
        return Money(str(result.quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)))
```

### 转账的原子性保证

转账涉及两个账户的余额变动，必须在同一个事务中完成：

```python
class TransferService:
    def transfer(self, from_account, to_account, amount, transaction_id):
        """
        转账：扣减 A + 增加 B，原子操作
        """
        with self.db.transaction() as tx:
            # 1. 按账户 ID 排序加锁（避免死锁）
            accounts = sorted([from_account, to_account])
            
            for account_id in accounts:
                tx.query_one("""
                    SELECT balance FROM account_snapshots
                    WHERE account_id = %s ORDER BY snapshot_time DESC LIMIT 1
                    FOR UPDATE
                """, account_id)

            # 2. 计算转出账户余额
            from_balance = self._calculate_balance(tx, from_account)
            if from_balance < amount:
                raise InsufficientBalanceError()

            # 3. 写入两个事件（原子）
            version_from = self._get_next_version(tx, from_account)
            version_to = self._get_next_version(tx, to_account)

            # 转出事件
            tx.execute("""
                INSERT INTO account_events
                (account_id, event_type, amount, transaction_id, counter_account_id, version)
                VALUES (%s, 'TRANSFER_OUT', %s, %s, %s, %s)
            """, from_account, amount, transaction_id, to_account, version_from)

            # 转入事件
            tx.execute("""
                INSERT INTO account_events
                (account_id, event_type, amount, transaction_id, counter_account_id, version)
                VALUES (%s, 'TRANSFER_IN', %s, %s, %s, %s)
            """, to_account, amount, transaction_id, from_account, version_to)

            # 4. 更新两个账户的快照
            self._update_snapshot(tx, from_account, from_balance - amount, version_from)
            self._update_snapshot(tx, to_account, 
                                  self._calculate_balance(tx, to_account) + amount, version_to)
```

**为什么按账户 ID 排序加锁？**

如果两个转账 A→B 和 B→A 同时执行：
- 转账1 先锁 A 再锁 B
- 转账2 先锁 B 再锁 A
- 死锁！

按 ID 排序后，两个转账都先锁 A 再锁 B，不会死锁。

### 日终对账的完整流程

**对账数据量：** 2000 万笔交易 vs 银行 2000 万行对账文件。

**步骤 1：内部对账（余额重算验证）**

```python
class ReconciliationService:
    def internal_reconciliation(self, date):
        """
        内部对账：重新计算所有账户余额，与快照对比
        """
        # 1. 获取所有活跃账户
        accounts = self.db.query("SELECT account_id FROM accounts WHERE status = 'ACTIVE'")
        
        discrepancies = []
        for account in accounts:
            # 2. 从事件全量计算余额
            calculated_balance = self._calculate_from_all_events(account.account_id)
            
            # 3. 获取快照余额
            snapshot_balance = self._get_snapshot_balance(account.account_id)
            
            # 4. 对比
            if calculated_balance != snapshot_balance:
                discrepancies.append({
                    "account_id": account.account_id,
                    "calculated": calculated_balance,
                    "snapshot": snapshot_balance,
                    "diff": calculated_balance - snapshot_balance
                })
        
        # 5. 处理差异
        for d in discrepancies:
            # 以事件计算为准，修正快照
            self._fix_snapshot(d.account_id, d.calculated)
            self._alert_discrepancy(d)
        
        return discrepancies
```

**性能优化：** 5000 万账户逐个重算太慢。优化方案：
- 并行计算：按 account_id 分片，多线程并行重算
- 增量校验：只重算当天有变动的账户（约 500 万个）
- 预计耗时：500 万账户 × 5ms/账户 = 25000 秒 ≈ 7 小时（不可接受）
- 进一步优化：使用批量 SQL 聚合，一次查询计算所有账户的余额变动

```sql
-- 批量计算所有账户今日余额变动
SELECT account_id, 
       SUM(CASE WHEN event_type IN ('RECHARGE','TRANSFER_IN','REFUND','SETTLEMENT')
                THEN amount ELSE -amount END) AS delta_balance
FROM account_events
WHERE created_at >= CURDATE() AND created_at < CURDATE() + INTERVAL 1 DAY
GROUP BY account_id;
```

此查询约 10-30 分钟完成（2000 万行聚合）。

**步骤 2：外部对账（与银行）**

```python
def external_reconciliation(self, date):
    """
    外部对账：与银行对账文件逐笔核对
    """
    # 1. 加载银行对账文件（CSV/固定宽度格式）
    bank_records = self.load_bank_statement(date)  # 2000万行
    
    # 2. 加载内部充值/提现记录
    internal_records = self.db.query("""
        SELECT transaction_id, amount, event_type, created_at
        FROM account_events
        WHERE created_at >= %s AND created_at < %s
          AND event_type IN ('RECHARGE', 'WITHDRAW')
    """, date, date + timedelta(days=1))
    
    # 3. 建立索引（用 transaction_id 或 bank_flow_no 做关联键）
    internal_map = {r.transaction_id: r for r in internal_records}
    
    # 4. 逐笔比对
    matched = 0
    bank_only = []    # 银行有我方无
    internal_only = [] # 我方有银行无
    amount_mismatch = []  # 金额不一致
    
    for bank_rec in bank_records:
        if bank_rec.flow_no in internal_map:
            internal_rec = internal_map[bank_rec.flow_no]
            if internal_rec.amount == bank_rec.amount:
                matched += 1
            else:
                amount_mismatch.append((bank_rec, internal_rec))
            del internal_map[bank_rec.flow_no]  # 已匹配，移除
        else:
            bank_only.append(bank_rec)
    
    # 剩余的 internal_map 条目 = 我方有银行无
    internal_only = list(internal_map.values())
    
    # 5. 处理差异
    for rec in bank_only:
        # 银行有我方无 → 可能是渠道延迟，检查后补录
        self.check_and_record(rec)
    
    for rec in internal_only:
        # 我方有银行无 → 可能是交易失败但事件已写入，需要修正
        self.check_and_compensate(rec)
    
    for bank_rec, internal_rec in amount_mismatch:
        # 金额不一致 → 人工介入
        self.alert_amount_mismatch(bank_rec, internal_rec)
    
    # 6. 总账对账
    total_internal = sum(r.amount for r in internal_records if r.event_type == 'RECHARGE') - \
                     sum(r.amount for r in internal_records if r.event_type == 'WITHDRAW')
    total_bank = sum(r.amount for r in bank_records if r.type == 'CREDIT') - \
                 sum(r.amount for r in bank_records if r.type == 'DEBIT')
    
    if total_internal != total_bank:
        self.alert_total_mismatch(total_internal, total_bank)
```

**步骤 3：总账平衡校验**

```
∑ 所有用户余额 + ∑ 所有商户余额 + 平台收入账户余额 = 银行总存款 + 在途资金

如果不等 → 告警 + 人工排查
```

这个等式是金融系统的"能量守恒定律"——钱不会凭空产生或消失。

### 分库分表策略

**为什么需要分库分表？**

单表 2000 万笔/天 × 365 天 = 73 亿条事件记录，单表无法承受。

**分片键：account_id**

```python
# 分片规则：account_id 哈希取模
shard = hash(account_id) % 16

# 表命名：account_events_{shard:02d}
# 如 account_events_00, account_events_01, ..., account_events_15
```

**为什么按 account_id 分片而非 transaction_id？**
- 余额计算需要查询同一账户的所有事件，按 account_id 分片可以避免跨库查询
- 对账时按 account_id 范围扫描，也避免了跨库

**跨账户转账的问题：**
- A→B 转账，A 和 B 可能在不同分片
- 方案：使用分布式事务（XA 或本地消息表），但金融场景下可以用更简单的方案——

**拆分为两个单账户操作：**

```python
# 不在一个事务中同时操作两个分片
# 而是拆分为两步：

# Step 1: 扣减 A 的余额（A 的分片上执行）
# 写入事件：TRANSFER_OUT, amount=-80, transaction_id=TXN-001

# Step 2: 增加 B 的余额（B 的分片上执行）
# 写入事件：TRANSFER_IN, amount=+80, transaction_id=TXN-001

# 如何保证两步的原子性？
# 使用本地消息表 + 事件驱动：
# - A 扣减成功后，写入本地消息表"待转入"
# - 消息消费者读取后，执行 B 的转入
# - 如果 B 转入失败，重试或退回 A
```

### 热账户优化

**问题：** 某些商户账户（如大型电商商户）的 TPS 极高，单账户 200 TPS 上限不够。

**方案：子账户拆分**

```
商户主账户：MERCHANT-001
  ├─ 子账户 1：MERCHANT-001-S01  （分摊 1/4 流量）
  ├─ 子账户 2：MERCHANT-001-S02  （分摊 1/4 流量）
  ├─ 子账户 3：MERCHANT-001-S03  （分摊 1/4 流量）
  └─ 子账户 4：MERCHANT-001-S04  （分摊 1/4 流量）

扣款时：随机选一个子账户扣款
查询总余额时：SUM 四个子账户的余额
```

4 个子账户 → TPS 上限从 200 提升到 800。

## 常见陷阱（深度分析）

### 陷阱 1：用 FLOAT/DOUBLE 存金额

**具体的错误累积案例：**

```python
# 1 亿次 0.01 元的累加
total = 0.0
for _ in range(100_000_000):
    total += 0.01

# 预期：1,000,000.00
# 实际：999,999.9999998412（差了约 0.0000001588 元）
# 看似微小，但在对账时：
# 内部总额 = 999,999.9999998412
# 银行总额 = 1,000,000.00
# 差异 = 0.0000001588 → 对账失败
```

### 陷阱 2：直接 UPDATE 余额

**并发超扣的具体构造：**

```
用户余额 = 100 元

T1: 请求A: SELECT balance FROM accounts WHERE id='U1'  → 100
T2: 请求B: SELECT balance FROM accounts WHERE id='U1'  → 100
T3: 请求A: UPDATE accounts SET balance = 100 - 80 = 20 WHERE id='U1'  ✓
T4: 请求B: UPDATE accounts SET balance = 100 - 80 = 20 WHERE id='U1'  ✓

结果：balance = 20，但应该是 -60 → 超扣 80 元
```

**修复：加 WHERE 条件**

```sql
UPDATE accounts SET balance = balance - 80 
WHERE id = 'U1' AND balance >= 80;
```

如果 affected_rows = 0，说明余额不足。但这仍然有并发问题——两个 UPDATE 可能都看到 balance >= 80。

**最终修复：行锁**

```sql
SELECT balance FROM accounts WHERE id = 'U1' FOR UPDATE;
-- 此时其他请求等待
UPDATE accounts SET balance = balance - 80 WHERE id = 'U1' AND balance >= 80;
```

### 陷阱 3：无事件日志

如果只有 `accounts.balance` 列，没有任何事件记录：
- 用户投诉"我充了 100 元但余额没变"→ 无法证明用户是否真的充了
- 银行对账发现差异 → 无法追溯到具体哪笔交易
- 监管审查 → 无法提供资金流向

### 陷阱 4：不处理补偿失败

扣款事件写入后，如果对应的业务操作（如创建订单）失败，需要退回扣款。如果退回也失败呢？

必须有：自动重试 → 告警 → 人工补偿 的完整链路。否则用户会永久丢失资金。

## 延伸思考

- **冻结金额**：提现场景下，用户申请提现后金额先"冻结"（不可用但仍在余额中），银行到账后正式扣减。设计一个冻结/解冻/扣减的完整状态机。
- **多币种账户**：如果支持 USD/EUR/JPY 等多种货币，每个币种独立记账，跨币种转账涉及汇率转换。汇率差如何处理？谁来承担？
- **快照策略**：快照的更新频率如何平衡存储成本和查询性能？如果账户 3 年没有交易，是否需要保留 3 年前的快照？