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

### 前置概念：复式记账法（Double-Entry Bookkeeping）

在进入具体设计之前，必须理解金融系统的底层会计原则——复式记账法。这不是可选的，而是监管要求。

**核心规则：有借必有贷，借贷必相等。**

每一笔交易必须同时记录至少两条分录（借方和贷方），且借方总额必须等于贷方总额。这意味着：

```
充值 100 元：
  借：用户资金账户（资产增加）  +100
  贷：银行存款账户（资产减少）  -100
  借贷合计：+100 = -(-100) ✓

支付 80 元（用户向商户）：
  借：商户待结算账户（资产增加）  +80
  贷：用户资金账户（资产减少）    -80
  借贷合计：+80 = -(-80) ✓
```

**为什么必须用复式记账而非单式记账？**

| 维度 | 单式记账 | 复式记账 |
|------|---------|---------|
| 错误检测 | 无法自动检测 | 借贷不平衡立即发现 |
| 资金追溯 | 只能看单方变动 | 完整展示资金来源和去向 |
| 对账 | 只能总额对 | 每笔交易内部自平衡 |
| 监管合规 | 不满足金融监管要求 | 满足央行、银监会的审计要求 |
| 内部舞弊 | 容易篡改单条记录 | 篡改一条必须同时篡改另一条 |

**复式记账的三层模型：**

```
第一层：会计科目（Account Chart）
  └─ 定义所有账户类型：资产类、负债类、权益类、收入类、费用类

第二层：会计分录（Journal Entry）
  └─ 每笔业务操作对应一个会计分录，包含多条分录行（借方/贷方）

第三层：账户余额（Account Balance）
  └─ 由会计分录聚合得出，可随时从分录重算验证
```

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

-- 会计分录表（复式记账核心：每笔交易对应一个分录）
CREATE TABLE journal_entries (
    entry_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    transaction_id VARCHAR(64) NOT NULL,       -- 业务交易ID
    entry_type VARCHAR(32) NOT NULL,           -- RECHARGE/WITHDRAW/PAYMENT/REFUND/SETTLEMENT/TRANSFER
    total_amount DECIMAL(15,2) NOT NULL,       -- 交易总金额
    status VARCHAR(16) DEFAULT 'POSTED',       -- DRAFT/POSTED/REVERSED
    description VARCHAR(255),
    created_by VARCHAR(64),                    -- 操作人/系统标识
    created_at TIMESTAMP(3) DEFAULT CURRENT_TIMESTAMP(3),
    posted_at TIMESTAMP(3),                    -- 过账时间
    reversed_at TIMESTAMP(3),                  -- 冲正时间

    UNIQUE KEY uk_transaction (transaction_id),
    INDEX idx_entry_type (entry_type),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 会计分录行表（借贷分录行：每个分录至少2行，借方+贷方）
CREATE TABLE journal_entry_lines (
    line_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    entry_id BIGINT NOT NULL,                 -- 关联分录ID
    account_id VARCHAR(32) NOT NULL,           -- 会计科目/账户ID
    entry_side VARCHAR(6) NOT NULL,            -- DEBIT(借方) / CREDIT(贷方)
    amount DECIMAL(15,2) NOT NULL,             -- 金额（正数）
    currency CHAR(3) DEFAULT 'CNY',
    description VARCHAR(255),
    created_at TIMESTAMP(3) DEFAULT CURRENT_TIMESTAMP(3),

    INDEX idx_entry_id (entry_id),
    INDEX idx_account_id (account_id),
    
    -- 外键约束（保证分录行必须属于某个分录）
    CONSTRAINT fk_entry FOREIGN KEY (entry_id) REFERENCES journal_entries(entry_id)
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

    def _calculate_from_events(self, account_id):
        """
        从全部事件重新计算余额（无快照时的兜底方案）
        """
        result = self.db.query_one("""
            SELECT COALESCE(SUM(
                CASE WHEN event_type IN ('RECHARGE','TRANSFER_IN','REFUND','SETTLEMENT')
                     THEN amount ELSE -amount END
            ), 0) AS balance
            FROM account_events
            WHERE account_id = %s
        """, account_id)
        return result.balance

    def get_balance_with_version(self, account_id):
        """
        查询余额并返回版本号（乐观锁使用）
        """
        snapshot = self.db.query_one("""
            SELECT balance, last_event_id, last_version
            FROM account_snapshots
            WHERE account_id = %s
            ORDER BY snapshot_time DESC LIMIT 1
        """, account_id)

        if snapshot is None:
            balance = self._calculate_from_events(account_id)
            # 无快照时版本号为 0
            return balance, 0

        delta = self.db.query_one("""
            SELECT COALESCE(SUM(
                CASE WHEN event_type IN ('RECHARGE','TRANSFER_IN','REFUND','SETTLEMENT')
                     THEN amount ELSE -amount END
            ), 0) AS delta_balance
            FROM account_events
            WHERE account_id = %s AND event_id > %s
        """, account_id, snapshot.last_event_id)

        return snapshot.balance + delta.delta_balance, snapshot.last_version
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

### 复式记账完整实现（借贷平衡校验）

复式记账是金融系统的基石。每笔业务操作必须生成借贷平衡的会计分录，否则系统拒绝写入。

```python
from decimal import Decimal
from dataclasses import dataclass
from enum import Enum
from typing import List

class EntrySide(Enum):
    DEBIT = "DEBIT"    # 借方
    CREDIT = "CREDIT"  # 贷方

@dataclass
class JournalLine:
    """会计分录行"""
    account_id: str
    entry_side: EntrySide
    amount: Decimal
    description: str = ""

@dataclass
class JournalEntry:
    """会计分录：由多条分录行组成"""
    transaction_id: str
    entry_type: str
    lines: List[JournalLine]
    description: str = ""

    def validate(self) -> bool:
        """
        核心校验：借贷必相等
        这是最关键的一步——任何不平衡的分录都不允许写入数据库
        """
        debit_total = Decimal('0')
        credit_total = Decimal('0')

        for line in self.lines:
            if line.amount <= 0:
                raise ValueError(f"分录金额必须为正数: {line.amount}")
            if line.entry_side == EntrySide.DEBIT:
                debit_total += line.amount
            else:
                credit_total += line.amount

        if debit_total != credit_total:
            raise UnbalancedJournalError(
                f"借贷不平衡: 借方={debit_total}, 贷方={credit_total}, "
                f"差额={debit_total - credit_total}, "
                f"transaction_id={self.transaction_id}"
            )

        if len(self.lines) < 2:
            raise ValueError("分录至少需要2行（借方和贷方）")

        return True


class JournalService:
    """会计分录服务：所有业务操作必须通过此服务生成复式分录"""

    def post_entry(self, entry: JournalEntry):
        """
        过账：将会计分录写入数据库
        过账前强制校验借贷平衡，任何不平衡分录直接拒绝
        """
        # 1. 校验借贷平衡
        entry.validate()

        # 2. 写入数据库（事务内保证原子性）
        with self.db.transaction() as tx:
            # 2a. 幂等检查：同一 transaction_id 不能重复过账
            existing = tx.query_one("""
                SELECT entry_id FROM journal_entries
                WHERE transaction_id = %s
            """, entry.transaction_id)
            if existing:
                return existing.entry_id

            # 2b. 写入分录头
            tx.execute("""
                INSERT INTO journal_entries
                (transaction_id, entry_type, total_amount, status, description, posted_at)
                VALUES (%s, %s, %s, 'POSTED', %s, NOW(3))
            """, entry.transaction_id, entry.entry_type,
                sum(l.amount for l in entry.lines), entry.description)
            entry_id = tx.last_insert_id()

            # 2c. 写入分录行
            for line in entry.lines:
                tx.execute("""
                    INSERT INTO journal_entry_lines
                    (entry_id, account_id, entry_side, amount, description)
                    VALUES (%s, %s, %s, %s, %s)
                """, entry_id, line.account_id, line.entry_side.value,
                    line.amount, line.description)

            # 2d. 同步写入账户事件表（保持与事件溯源的一致性）
            for line in entry.lines:
                event_type = self._map_to_event_type(line.entry_side, entry.entry_type)
                tx.execute("""
                    INSERT INTO account_events
                    (account_id, event_type, amount, transaction_id, version)
                    VALUES (%s, %s, %s, %s, 
                        (SELECT COALESCE(MAX(version), 0) + 1 
                         FROM account_events ae 
                         WHERE ae.account_id = %s))
                """, line.account_id, event_type, line.amount,
                    entry.transaction_id, line.account_id)

        return entry_id

    def _map_to_event_type(self, side: EntrySide, entry_type: str) -> str:
        """将分录方向映射为事件类型"""
        # 借方：资产增加 or 负债减少
        # 贷方：资产减少 or 负债增加
        mapping = {
            ('DEBIT', 'RECHARGE'): 'RECHARGE',
            ('CREDIT', 'RECHARGE'): 'BANK_OUT',
            ('DEBIT', 'PAYMENT'): 'MERCHANT_IN',
            ('CREDIT', 'PAYMENT'): 'WITHDRAW',
            ('DEBIT', 'REFUND'): 'REFUND',
            ('CREDIT', 'REFUND'): 'MERCHANT_OUT',
            ('DEBIT', 'WITHDRAW'): 'BANK_IN',
            ('CREDIT', 'WITHDRAW'): 'WITHDRAW',
        }
        return mapping.get((side.value, entry_type), 'OTHER')

    # === 各业务操作的复式分录生成 ===

    def create_recharge_entry(self, user_account_id: str, amount: Decimal,
                               transaction_id: str, bank_account_id: str = 'BANK_CNY'):
        """
        充值分录：
          借：用户资金账户（用户资产增加）
          贷：银行存款账户（银行资产减少）
        """
        entry = JournalEntry(
            transaction_id=transaction_id,
            entry_type='RECHARGE',
            lines=[
                JournalLine(user_account_id, EntrySide.DEBIT, amount,
                           "用户充值-资金入账"),
                JournalLine(bank_account_id, EntrySide.CREDIT, amount,
                           "用户充值-银行扣减"),
            ]
        )
        return self.post_entry(entry)

    def create_payment_entry(self, user_account_id: str, merchant_account_id: str,
                              amount: Decimal, transaction_id: str):
        """
        支付分录：
          借：商户待结算账户（商户应收增加）
          贷：用户资金账户（用户资产减少）
        """
        entry = JournalEntry(
            transaction_id=transaction_id,
            entry_type='PAYMENT',
            lines=[
                JournalLine(merchant_account_id, EntrySide.DEBIT, amount,
                           "用户支付-商户应收"),
                JournalLine(user_account_id, EntrySide.CREDIT, amount,
                           "用户支付-资金扣减"),
            ]
        )
        return self.post_entry(entry)

    def create_settlement_entry(self, merchant_account_id: str,
                                 settlement_account_id: str,
                                 amount: Decimal, fee: Decimal,
                                 transaction_id: str):
        """
        结算分录（含手续费）：
          借：银行结算账户（银行支出）
          贷：商户待结算账户（商户应收减少）
          贷：平台收入账户（手续费收入）
        
        关键：借方 = 贷方 = amount（结算金额）
              其中 fee 是平台收取的手续费，商户实际到账 = amount - fee
        """
        merchant_amount = amount - fee
        entry = JournalEntry(
            transaction_id=transaction_id,
            entry_type='SETTLEMENT',
            lines=[
                JournalLine(settlement_account_id, EntrySide.DEBIT, amount,
                           "商户结算-银行支出"),
                JournalLine(merchant_account_id, EntrySide.CREDIT, merchant_amount,
                           "商户结算-商户到账"),
                JournalLine('PLATFORM_REVENUE', EntrySide.CREDIT, fee,
                           "商户结算-平台手续费收入"),
            ]
        )
        return self.post_entry(entry)

    def create_refund_entry(self, user_account_id: str, merchant_account_id: str,
                             amount: Decimal, transaction_id: str):
        """
        退款分录（支付的反向操作）：
          借：用户资金账户（用户资产增加）
          贷：商户待结算账户（商户应收减少）
        """
        entry = JournalEntry(
            transaction_id=transaction_id,
            entry_type='REFUND',
            lines=[
                JournalLine(user_account_id, EntrySide.DEBIT, amount,
                           "退款-用户资金退回"),
                JournalLine(merchant_account_id, EntrySide.CREDIT, amount,
                           "退款-商户应收扣减"),
            ]
        )
        return self.post_entry(entry)

    def create_transfer_entry(self, from_account_id: str, to_account_id: str,
                               amount: Decimal, transaction_id: str):
        """
        转账分录：
          借：转入账户（资产增加）
          贷：转出账户（资产减少）
        """
        entry = JournalEntry(
            transaction_id=transaction_id,
            entry_type='TRANSFER',
            lines=[
                JournalLine(to_account_id, EntrySide.DEBIT, amount,
                           "转账-资金入账"),
                JournalLine(from_account_id, EntrySide.CREDIT, amount,
                           "转账-资金扣减"),
            ]
        )
        return self.post_entry(entry)
```

**借贷平衡校验的触发时机：**

```
业务请求 → 生成会计分录 → 【强制校验借贷平衡】→ 写入数据库
                                    ↓ 失败
                              抛出异常，拒绝写入
                              记录审计日志
                              告警通知运维
```

**为什么在代码层和数据库层都要做校验？**

- 代码层校验：快速失败，避免无效 SQL 执行
- 数据库层校验：防御性编程，即使代码有 bug 也不会写入不平衡分录

```sql
-- 数据库层触发器：强制借贷平衡
DELIMITER //
CREATE TRIGGER validate_journal_balance
AFTER INSERT ON journal_entry_lines
FOR EACH ROW
BEGIN
    DECLARE debit_sum DECIMAL(15,2);
    DECLARE credit_sum DECIMAL(15,2);
    
    SELECT COALESCE(SUM(CASE WHEN entry_side = 'DEBIT' THEN amount ELSE 0 END), 0),
           COALESCE(SUM(CASE WHEN entry_side = 'CREDIT' THEN amount ELSE 0 END), 0)
    INTO debit_sum, credit_sum
    FROM journal_entry_lines
    WHERE entry_id = NEW.entry_id;
    
    IF debit_sum != credit_sum THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = CONCAT('Journal entry unbalanced: debit=', debit_sum, 
                                  ' credit=', credit_sum);
    END IF;
END//
DELIMITER ;
```

注意：触发器在高并发场景下可能成为性能瓶颈，生产环境建议使用应用层校验 + 定期批量校验的组合方案。

### 日终对账的完整流程

**对账数据量：** 2000 万笔交易 vs 银行 2000 万行对账文件。

**对账的整体架构：**

```
┌─────────────┐     ┌──────────────┐     ┌───────────────┐
│  内部数据     │     │  对账引擎      │     │  银行对账文件   │
│  account_    │────→│  ReconEngine  │←────│  bank_stmt    │
│  events      │     │              │     │  .csv         │
└─────────────┘     │  1. 内部对账   │     └───────────────┘
                    │  2. 外部对账   │
                    │  3. 差异处理   │────→ 告警/自动补偿
                    │  4. 报告生成   │────→ 对账报告
                    └──────────────┘
```

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
class ReconciliationService:
    # ... (internal_reconciliation 方法如上)

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

### 自动对账引擎完整实现

下面是对账引擎的完整生产级实现，包含日常自动对账、差异检测、自动补偿和报告生成。

```python
from datetime import date, datetime, timedelta
from decimal import Decimal
from enum import Enum
from typing import Optional, List, Dict
import logging

logger = logging.getLogger(__name__)

class ReconStatus(Enum):
    MATCHED = "MATCHED"           # 匹配成功
    BANK_ONLY = "BANK_ONLY"       # 银行有我方无
    INTERNAL_ONLY = "INTERNAL_ONLY"  # 我方有银行无
    AMOUNT_MISMATCH = "AMOUNT_MISMATCH"  # 金额不一致
    TIME_MISMATCH = "TIME_MISMATCH"  # 时间不一致（跨日）

class DiscrepancySeverity(Enum):
    LOW = "LOW"          # 小额差异，可自动处理
    MEDIUM = "MEDIUM"    # 中等差异，需人工确认
    HIGH = "HIGH"        # 大额差异，必须人工介入
    CRITICAL = "CRITICAL"  # 系统性差异，立即告警

@dataclass
class ReconResult:
    """对账结果"""
    date: date
    total_internal: int        # 内部记录数
    total_bank: int            # 银行记录数
    matched: int               # 匹配数
    bank_only: int             # 银行单边数
    internal_only: int         # 内部单边数
    amount_mismatch: int       # 金额不一致数
    total_internal_amount: Decimal  # 内部总额
    total_bank_amount: Decimal      # 银行总额
    discrepancy_amount: Decimal     # 差异总额

class AutoReconEngine:
    """
    自动对账引擎
    支持定时触发（每日凌晨2:00自动运行）和手动触发
    """

    # 差异金额阈值（元）
    AMOUNT_THRESHOLD_LOW = Decimal('1.00')
    AMOUNT_THRESHOLD_MEDIUM = Decimal('100.00')
    AMOUNT_THRESHOLD_HIGH = Decimal('10000.00')

    # 跨日对账窗口（T-1 到 T+1 的记录都会参与匹配）
    CROSS_DAY_WINDOW_HOURS = 24

    def run_daily_recon(self, target_date: date) -> ReconResult:
        """
        日终自动对账主流程
        每日凌晨 2:00 由定时任务触发
        """
        logger.info(f"开始日终对账: {target_date}")

        # 阶段1：内部对账（余额重算验证）
        internal_result = self._internal_recon(target_date)
        if internal_result['discrepancy_count'] > 0:
            self._handle_internal_discrepancy(internal_result)

        # 阶段2：复式记账平衡校验
        balance_result = self._validate_double_entry(target_date)
        if not balance_result['balanced']:
            self._alert_unbalanced_entries(balance_result)

        # 阶段3：外部对账（与银行）
        external_result = self._external_recon(target_date)

        # 阶段4：总账平衡
        total_result = self._total_balance_recon(target_date)

        # 阶段5：生成对账报告
        recon_result = self._build_result(
            target_date, internal_result, external_result, total_result
        )
        self._save_recon_report(recon_result)

        # 阶段6：自动处理可自动修复的差异
        self._auto_resolve_discrepancies(target_date, external_result)

        logger.info(f"日终对账完成: {target_date}, 匹配率={recon_result.matched / recon_result.total_internal * 100:.4f}%")
        return recon_result

    def _internal_recon(self, target_date: date) -> dict:
        """
        内部对账：验证快照余额与事件重算余额是否一致
        同时验证复式记账的借贷平衡
        """
        # 只检查当天有变动的账户（增量校验）
        active_accounts = self.db.query("""
            SELECT DISTINCT account_id
            FROM account_events
            WHERE created_at >= %s AND created_at < %s
        """, target_date, target_date + timedelta(days=1))

        discrepancies = []
        for row in active_accounts:
            account_id = row.account_id

            # 从事件全量重算余额
            calculated = self.db.query_one("""
                SELECT COALESCE(SUM(
                    CASE WHEN event_type IN ('RECHARGE','TRANSFER_IN','REFUND','SETTLEMENT')
                         THEN amount ELSE -amount END
                ), 0) AS balance
                FROM account_events
                WHERE account_id = %s AND created_at < %s
            """, account_id, target_date + timedelta(days=1))

            # 获取快照余额
            snapshot = self.db.query_one("""
                SELECT balance FROM account_snapshots
                WHERE account_id = %s AND snapshot_time < %s
                ORDER BY snapshot_time DESC LIMIT 1
            """, account_id, target_date + timedelta(days=1))

            if snapshot and calculated.balance != snapshot.balance:
                severity = self._classify_severity(abs(calculated.balance - snapshot.balance))
                discrepancies.append({
                    'account_id': account_id,
                    'calculated_balance': calculated.balance,
                    'snapshot_balance': snapshot.balance,
                    'diff': calculated.balance - snapshot.balance,
                    'severity': severity
                })

        return {
            'total_accounts': len(active_accounts),
            'discrepancy_count': len(discrepancies),
            'discrepancies': discrepancies
        }

    def _validate_double_entry(self, target_date: date) -> dict:
        """
        复式记账平衡校验：验证当天所有分录的借贷是否平衡
        """
        result = self.db.query_one("""
            SELECT 
                je.entry_id,
                je.transaction_id,
                COALESCE(SUM(CASE WHEN jel.entry_side = 'DEBIT' THEN jel.amount ELSE 0 END), 0) AS debit_total,
                COALESCE(SUM(CASE WHEN jel.entry_side = 'CREDIT' THEN jel.amount ELSE 0 END), 0) AS credit_total
            FROM journal_entries je
            JOIN journal_entry_lines jel ON je.entry_id = jel.entry_id
            WHERE je.posted_at >= %s AND je.posted_at < %s
            GROUP BY je.entry_id, je.transaction_id
            HAVING debit_total != credit_total
        """, target_date, target_date + timedelta(days=1))

        unbalanced = []
        if result:
            for row in result:
                unbalanced.append({
                    'entry_id': row.entry_id,
                    'transaction_id': row.transaction_id,
                    'debit_total': row.debit_total,
                    'credit_total': row.credit_total,
                    'diff': row.debit_total - row.credit_total
                })

        return {
            'balanced': len(unbalanced) == 0,
            'unbalanced_entries': unbalanced
        }

    def _external_recon(self, target_date: date) -> dict:
        """
        外部对账：与银行对账文件逐笔比对
        使用两阶段匹配策略：精确匹配 + 模糊匹配
        """
        # 加载银行对账文件
        bank_records = self._load_bank_statement(target_date)

        # 加载内部记录（扩大时间窗口以处理跨日交易）
        window_start = target_date - timedelta(hours=self.CROSS_DAY_WINDOW_HOURS)
        window_end = target_date + timedelta(hours=self.CROSS_DAY_WINDOW_HOURS)

        internal_records = self.db.query("""
            SELECT transaction_id, amount, event_type, created_at, biz_no
            FROM account_events
            WHERE created_at >= %s AND created_at < %s
              AND event_type IN ('RECHARGE', 'WITHDRAW')
        """, window_start, window_end)

        # ===== 阶段1：精确匹配（transaction_id + amount 完全一致）=====
        internal_map = {r.transaction_id: r for r in internal_records}
        matched = []
        bank_only = []
        amount_mismatch = []

        for bank_rec in bank_records:
            if bank_rec.flow_no in internal_map:
                internal_rec = internal_map[bank_rec.flow_no]
                if internal_rec.amount == bank_rec.amount:
                    matched.append({
                        'transaction_id': bank_rec.flow_no,
                        'amount': bank_rec.amount,
                        'status': ReconStatus.MATCHED
                    })
                else:
                    amount_mismatch.append({
                        'transaction_id': bank_rec.flow_no,
                        'internal_amount': internal_rec.amount,
                        'bank_amount': bank_rec.amount,
                        'diff': internal_rec.amount - bank_rec.amount,
                        'status': ReconStatus.AMOUNT_MISMATCH,
                        'severity': self._classify_severity(abs(internal_rec.amount - bank_rec.amount))
                    })
                del internal_map[bank_rec.flow_no]
            else:
                bank_only.append(bank_rec)

        internal_only = list(internal_map.values())

        # ===== 阶段2：模糊匹配（对银行单边和内部单边进行二次匹配）=====
        # 按金额+时间范围模糊匹配（可能是 transaction_id 映射不一致）
        fuzzy_matched, remaining_bank, remaining_internal = \
            self._fuzzy_match(bank_only, internal_only, target_date)

        matched.extend(fuzzy_matched)

        return {
            'total_internal': len(internal_records),
            'total_bank': len(bank_records),
            'matched_count': len(matched),
            'bank_only': remaining_bank,
            'internal_only': remaining_internal,
            'amount_mismatch': amount_mismatch,
            'fuzzy_matched': len(fuzzy_matched)
        }

    def _fuzzy_match(self, bank_only: list, internal_only: list,
                      target_date: date) -> tuple:
        """
        模糊匹配：按金额+时间窗口进行二次匹配
        处理 transaction_id 映射不一致的场景
        """
        fuzzy_matched = []
        remaining_bank = list(bank_only)
        remaining_internal = list(internal_only)

        # 按金额分组
        bank_by_amount: Dict[Decimal, list] = {}
        for rec in remaining_bank:
            key = rec.amount
            bank_by_amount.setdefault(key, []).append(rec)

        for int_rec in remaining_internal[:]:
            candidates = bank_by_amount.get(int_rec.amount, [])
            for bank_rec in candidates[:]:
                # 检查时间是否在允许窗口内
                time_diff = abs((int_rec.created_at - bank_rec.created_at).total_seconds())
                if time_diff <= self.CROSS_DAY_WINDOW_HOURS * 3600:
                    fuzzy_matched.append({
                        'transaction_id': int_rec.transaction_id,
                        'bank_flow_no': bank_rec.flow_no,
                        'amount': int_rec.amount,
                        'time_diff_seconds': time_diff,
                        'status': ReconStatus.TIME_MISMATCH,
                        'match_type': 'FUZZY'
                    })
                    remaining_bank.remove(bank_rec)
                    remaining_internal.remove(int_rec)
                    candidates.remove(bank_rec)
                    break

        return fuzzy_matched, remaining_bank, remaining_internal

    def _total_balance_recon(self, target_date: date) -> dict:
        """
        总账平衡校验：验证"能量守恒定律"
        ∑ 用户余额 + ∑ 商户余额 + 平台收入 = 银行存款 + 在途资金
        """
        # 从分录表聚合，而非逐账户累加（性能更优）
        result = self.db.query_one("""
            SELECT 
                COALESCE(SUM(CASE 
                    WHEN a.account_type = 'PERSONAL' AND jel.entry_side = 'DEBIT' THEN jel.amount
                    WHEN a.account_type = 'PERSONAL' AND jel.entry_side = 'CREDIT' THEN -jel.amount
                    ELSE 0 
                END), 0) AS personal_total,
                COALESCE(SUM(CASE 
                    WHEN a.account_type = 'MERCHANT' AND jel.entry_side = 'DEBIT' THEN jel.amount
                    WHEN a.account_type = 'MERCHANT' AND jel.entry_side = 'CREDIT' THEN -jel.amount
                    ELSE 0 
                END), 0) AS merchant_total,
                COALESCE(SUM(CASE 
                    WHEN a.account_type = 'PLATFORM' AND jel.entry_side = 'DEBIT' THEN jel.amount
                    WHEN a.account_type = 'PLATFORM' AND jel.entry_side = 'CREDIT' THEN -jel.amount
                    ELSE 0 
                END), 0) AS platform_total
            FROM journal_entry_lines jel
            JOIN journal_entries je ON jel.entry_id = je.entry_id
            JOIN accounts a ON jel.account_id = a.account_id
            WHERE je.posted_at >= %s AND je.posted_at < %s
              AND je.status = 'POSTED'
        """, target_date, target_date + timedelta(days=1))

        # 总账应该平衡：所有账户的借贷之和 = 0
        total = result.personal_total + result.merchant_total + result.platform_total
        return {
            'personal_total': result.personal_total,
            'merchant_total': result.merchant_total,
            'platform_total': result.platform_total,
            'grand_total': total,
            'balanced': total == Decimal('0')
        }

    def _classify_severity(self, diff_amount: Decimal) -> DiscrepancySeverity:
        """根据差异金额分类严重程度"""
        if diff_amount >= self.AMOUNT_THRESHOLD_HIGH:
            return DiscrepancySeverity.CRITICAL
        elif diff_amount >= self.AMOUNT_THRESHOLD_MEDIUM:
            return DiscrepancySeverity.HIGH
        elif diff_amount >= self.AMOUNT_THRESHOLD_LOW:
            return DiscrepancySeverity.MEDIUM
        else:
            return DiscrepancySeverity.LOW

    def _auto_resolve_discrepancies(self, target_date: date, external_result: dict):
        """
        自动处理可自动修复的差异
        规则：
        - LOW 级别：自动生成调整分录
        - MEDIUM 级别：标记待人工确认，但不自动调整
        - HIGH/CRITICAL 级别：只告警，不自动处理
        """
        for mismatch in external_result.get('amount_mismatch', []):
            if mismatch['severity'] == DiscrepancySeverity.LOW:
                # 小额差异：生成调整分录，挂账到"待处理差异"科目
                self._create_adjustment_entry(
                    transaction_id=mismatch['transaction_id'],
                    diff_amount=mismatch['diff'],
                    reason=f"对账差异自动调整: 银行={mismatch['bank_amount']}, 内部={mismatch['internal_amount']}"
                )
            elif mismatch['severity'] in (DiscrepancySeverity.HIGH, DiscrepancySeverity.CRITICAL):
                self._send_critical_alert(mismatch)

    def _create_adjustment_entry(self, transaction_id: str, diff_amount: Decimal, reason: str):
        """
        创建调整分录（将差异挂到"待处理差异"科目）
        注意：这不是"抹平差异"，而是将差异显式记录，后续由人工决定如何处理
        """
        if diff_amount > 0:
            # 内部多记了 → 借记待处理差异，贷记相关账户
            entry = JournalEntry(
                transaction_id=f"ADJ-{transaction_id}",
                entry_type='RECON_ADJUSTMENT',
                lines=[
                    JournalLine('PENDING_DIFF', EntrySide.DEBIT, abs(diff_amount), reason),
                    JournalLine('SUSPENSE', EntrySide.CREDIT, abs(diff_amount), reason),
                ]
            )
        else:
            # 内部少记了 → 借记相关账户，贷记待处理差异
            entry = JournalEntry(
                transaction_id=f"ADJ-{transaction_id}",
                entry_type='RECON_ADJUSTMENT',
                lines=[
                    JournalLine('SUSPENSE', EntrySide.DEBIT, abs(diff_amount), reason),
                    JournalLine('PENDING_DIFF', EntrySide.CREDIT, abs(diff_amount), reason),
                ]
            )
        self.journal_service.post_entry(entry)

    def _send_critical_alert(self, mismatch: dict):
        """发送严重差异告警"""
        # 集成告警系统（钉钉/企微/PagerDuty）
        alert_msg = (
            f"[严重对账差异告警]\n"
            f"交易ID: {mismatch['transaction_id']}\n"
            f"内部金额: {mismatch['internal_amount']}\n"
            f"银行金额: {mismatch['bank_amount']}\n"
            f"差异金额: {mismatch['diff']}\n"
            f"严重程度: {mismatch['severity'].value}\n"
            f"请立即人工排查！"
        )
        logger.critical(alert_msg)
        self.alert_service.send_critical(alert_msg)

    def _save_recon_report(self, result: ReconResult):
        """保存对账报告到数据库"""
        self.db.execute("""
            INSERT INTO recon_reports
            (recon_date, total_internal, total_bank, matched, bank_only, internal_only,
             amount_mismatch, total_internal_amount, total_bank_amount, discrepancy_amount,
             status, created_at)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, 'COMPLETED', NOW(3))
        """, result.date, result.total_internal, result.total_bank, result.matched,
            result.bank_only, result.internal_only, result.amount_mismatch,
            result.total_internal_amount, result.total_bank_amount, result.discrepancy_amount)
```

**对账报告表结构：**

```sql
-- 对账报告表
CREATE TABLE recon_reports (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    recon_date DATE NOT NULL,
    total_internal INT NOT NULL,
    total_bank INT NOT NULL,
    matched INT NOT NULL,
    bank_only INT NOT NULL DEFAULT 0,
    internal_only INT NOT NULL DEFAULT 0,
    amount_mismatch INT NOT NULL DEFAULT 0,
    total_internal_amount DECIMAL(15,2) NOT NULL,
    total_bank_amount DECIMAL(15,2) NOT NULL,
    discrepancy_amount DECIMAL(15,2) NOT NULL DEFAULT 0,
    status VARCHAR(16) DEFAULT 'COMPLETED',
    created_at TIMESTAMP(3) DEFAULT CURRENT_TIMESTAMP(3),

    UNIQUE KEY uk_recon_date (recon_date)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 对账差异明细表
CREATE TABLE recon_discrepancies (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    recon_date DATE NOT NULL,
    transaction_id VARCHAR(64),
    discrepancy_type VARCHAR(32) NOT NULL,  -- BANK_ONLY/INTERNAL_ONLY/AMOUNT_MISMATCH
    internal_amount DECIMAL(15,2),
    bank_amount DECIMAL(15,2),
    diff_amount DECIMAL(15,2),
    severity VARCHAR(16) NOT NULL,          -- LOW/MEDIUM/HIGH/CRITICAL
    resolution VARCHAR(32) DEFAULT 'PENDING',  -- PENDING/AUTO_RESOLVED/MANUAL_RESOLVED/ESCALATED
    resolved_by VARCHAR(64),
    resolved_at TIMESTAMP(3),
    remark VARCHAR(512),
    created_at TIMESTAMP(3) DEFAULT CURRENT_TIMESTAMP(3),

    INDEX idx_recon_date (recon_date),
    INDEX idx_severity (severity),
    INDEX idx_resolution (resolution)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**对账引擎的定时调度：**

```python
# 使用 APScheduler 或 Celery Beat 进行定时调度
from apscheduler.schedulers.blocking import BlockingScheduler

scheduler = BlockingScheduler()

# 每日凌晨 2:00 执行对账
scheduler.add_job(
    AutoReconEngine().run_daily_recon,
    'cron',
    hour=2, minute=0,
    args=[date.today() - timedelta(days=1)],  # 对前一天的数据对账
    max_instances=1,  # 不允许并发执行
    misfire_grace_time=3600  # 允许1小时内的延迟执行
)

scheduler.start()
```

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
## 对账引擎完整实现

```python
class DailyReconciliationService:
    """日终自动对账引擎"""

    DISCREPANCY_TYPES = {
        "timing_diff":    "时间差：一方已入账另一方未入账",
        "amount_mismatch": "金额不一致",
        "missing_entry":  "内部有记录但银行无",
        "duplicate_entry": "重复入账",
        "rounding":       "尾差（<0.01 元）",
        "fee_diff":       "手续费差异",
    }

    def reconcile(self, date):
        """执行日终对账"""
        # 1. 获取内部账目和银行对账单
        internal_entries = self.ledger.get_entries_by_date(date)
        bank_statements = self.bank_client.get_statement(date)

        # 2. 按交易号匹配
        matched, unmatched_internal, unmatched_bank = self._match_by_ref(
            internal_entries, bank_statements)

        # 3. 检查匹配记录的金额是否一致
        amount_discrepancies = []
        for int_entry, bank_entry in matched:
            if abs(int_entry.amount - bank_entry.amount) > 0.01:
                amount_discrepancies.append({
                    "type": "amount_mismatch",
                    "ref_id": int_entry.ref_id,
                    "internal_amount": int_entry.amount,
                    "bank_amount": bank_entry.amount,
                    "diff": abs(int_entry.amount - bank_entry.amount),
                })
            elif abs(int_entry.amount - bank_entry.amount) > 0:
                # 尾差自动解决
                self._auto_resolve("rounding", int_entry, diff=int_entry.amount - bank_entry.amount)

        # 4. 处理未匹配记录
        resolved, escalated = self._resolve_unmatched(
            unmatched_internal, unmatched_bank, date)

        # 5. 生成对账报告
        report = {
            "date": date,
            "total_internal": len(internal_entries),
            "total_bank": len(bank_statements),
            "matched": len(matched),
            "amount_discrepancies": len(amount_discrepancies),
            "auto_resolved": len(resolved),
            "escalated": len(escalated),
            "resolved_details": resolved,
            "escalated_details": escalated,
        }
        self.db.insert("reconciliation_reports", report)
        return report

    def _match_by_ref(self, internal, bank):
        """按交易参考号匹配"""
        bank_by_ref = {e.ref_id: e for e in bank}
        matched, unmatched_int, unmatched_bank = [], [], list(bank)

        for int_entry in internal:
            if int_entry.ref_id in bank_by_ref:
                bank_entry = bank_by_ref[int_entry.ref_id]
                matched.append((int_entry, bank_entry))
                unmatched_bank.remove(bank_entry)
            else:
                unmatched_int.append(int_entry)

        return matched, unmatched_int, unmatched_bank

    def _resolve_unmatched(self, unmatched_internal, unmatched_bank, date):
        """自动解决未匹配记录"""
        resolved, escalated = [], []

        for entry in unmatched_internal:
            # 检查是否是时间差（银行可能 T+1 才入账）
            next_day_entry = self.bank_client.find_by_ref(
                entry.ref_id, date + timedelta(days=1))
            if next_day_entry:
                resolved.append({
                    "type": "timing_diff", "ref_id": entry.ref_id,
                    "resolution": "T+1 匹配成功", "amount": entry.amount
                })
            elif self._is_known_fee(entry):
                resolved.append({
                    "type": "fee_diff", "ref_id": entry.ref_id,
                    "resolution": "已知手续费", "amount": entry.amount
                })
            else:
                escalated.append({
                    "type": "missing_entry", "ref_id": entry.ref_id,
                    "amount": entry.amount, "reason": "银行无对应记录，需人工核实"
                })

        for entry in unmatched_bank:
            # 检查是否是重复入账
            existing = self.ledger.find_by_ref(entry.ref_id)
            if existing:
                escalated.append({
                    "type": "duplicate_entry", "ref_id": entry.ref_id,
                    "amount": entry.amount, "reason": "可能重复入账，需人工核实"
                })
            else:
                escalated.append({
                    "type": "missing_entry", "ref_id": entry.ref_id,
                    "amount": entry.amount, "reason": "内部无对应记录，需人工核实"
                })

        return resolved, escalated
```

## 会计期间结账完整实现

```python
class PeriodCloseService:
    """会计期间结账服务"""

    def close_period(self, period_date):
        """执行完整的期间结账流程"""
        # 1. 生成试算平衡表
        trial_balance = self._generate_trial_balance(period_date)
        if not trial_balance["is_balanced"]:
            raise UnbalancedTrialBalanceError(
                f"试算不平衡：借方 {trial_balance['total_debits']} "
                f"≠ 贷方 {trial_balance['total_credits']}")

        # 2. 生成调整分录
        adjusting_entries = self._generate_adjusting_entries(period_date)
        for entry in adjusting_entries:
            self.ledger.post_entry(entry)

        # 3. 重新生成试算平衡表（调整后）
        adjusted_balance = self._generate_trial_balance(period_date)
        if not adjusted_balance["is_balanced"]:
            raise UnbalancedTrialBalanceError("调整后试算不平衡")

        # 4. 生成结账分录（收入/费用 → 未分配利润）
        closing_entries = self._generate_closing_entries(period_date)
        for entry in closing_entries:
            self.ledger.post_entry(entry)

        # 5. 锁定期间
        self.db.insert("accounting_periods", {
            "period_date": period_date,
            "status": "closed",
            "trial_balance": json.dumps(trial_balance),
            "adjusted_balance": json.dumps(adjusted_balance),
            "adjusting_entry_count": len(adjusting_entries),
            "closing_entry_count": len(closing_entries),
            "closed_at": now(),
            "closed_by": current_user()
        })

        return {
            "period": period_date,
            "status": "closed",
            "adjusting_entries": len(adjusting_entries),
            "closing_entries": len(closing_entries)
        }

    def _generate_trial_balance(self, date):
        """生成试算平衡表"""
        accounts = self.db.query(
            "SELECT account_code, account_name, account_type, "
            "SUM(debit_amount) as total_debit, SUM(credit_amount) as total_credit "
            "FROM journal_entry_lines WHERE entry_date <= %s "
            "GROUP BY account_code, account_name, account_type", date)

        total_debits = sum(a["total_debit"] for a in accounts)
        total_credits = sum(a["total_credit"] for a in accounts)

        return {
            "accounts": accounts,
            "total_debits": total_debits,
            "total_credits": total_credits,
            "is_balanced": abs(total_debits - total_credits) < 0.01
        }

    def _generate_adjusting_entries(self, date):
        """生成调整分录：应计、递延"""
        entries = []
        # 应计费用：已发生但未收到发票的费用
        accruals = self.db.query(
            "SELECT * FROM accrued_expenses WHERE period = %s AND status = 'pending'", date)
        for accrual in accruals:
            entries.append(JournalEntry(
                lines=[
                    EntryLine(accrual.expense_account, debit=accrual.amount),
                    EntryLine(accrual.payable_account, credit=accrual.amount)
                ],
                entry_type="adjusting",
                description=f"应计费用调整: {accrual.description}"
            ))

        # 递延收入：已收款但未确认的收入
        deferrals = self.db.query(
            "SELECT * FROM deferred_revenue WHERE period = %s "
            "AND start_date <= %s AND end_date > %s", date, date, date)
        for deferral in deferrals:
            monthly_amount = deferral.total_amount / deferral.months
            entries.append(JournalEntry(
                lines=[
                    EntryLine(deferral.revenue_account, credit=monthly_amount),
                    EntryLine(deferral.unearned_revenue_account, debit=monthly_amount)
                ],
                entry_type="adjusting",
                description=f"递延收入确认: {deferral.description}"
            ))
        return entries

    def _generate_closing_entries(self, date):
        """生成结账分录：收入/费用 → 未分配利润"""
        entries = []
        retained_earnings = "3001"  # 未分配利润科目

        # 收入类科目余额 → 未分配利润
        revenue_accounts = self.db.query(
            "SELECT account_code, SUM(credit_amount - debit_amount) as balance "
            "FROM journal_entry_lines WHERE account_type = 'revenue' "
            "AND entry_date <= %s GROUP BY account_code", date)
        for acc in revenue_accounts:
            if acc["balance"] > 0:
                entries.append(JournalEntry(
                    lines=[
                        EntryLine(acc["account_code"], debit=acc["balance"]),
                        EntryLine(retained_earnings, credit=acc["balance"])
                    ],
                    entry_type="closing",
                    description=f"结转收入: {acc['account_code']}"
                ))

        # 费用类科目余额 → 未分配利润
        expense_accounts = self.db.query(
            "SELECT account_code, SUM(debit_amount - credit_amount) as balance "
            "FROM journal_entry_lines WHERE account_type = 'expense' "
            "AND entry_date <= %s GROUP BY account_code", date)
        for acc in expense_accounts:
            if acc["balance"] > 0:
                entries.append(JournalEntry(
                    lines=[
                        EntryLine(retained_earnings, debit=acc["balance"]),
                        EntryLine(acc["account_code"], credit=acc["balance"])
                    ],
                    entry_type="closing",
                    description=f"结转费用: {acc['account_code']}"
                ))
        return entries
```

## 审计追踪哈希链

```python
class ImmutableAuditLogger:
    """不可变审计日志：哈希链保证数据完整性"""

    def log_entry(self, operation, data):
        """写入审计日志（含哈希链）"""
        # 获取前一条记录的哈希
        prev_record = self.db.query_one(
            "SELECT record_hash FROM audit_trail ORDER BY id DESC LIMIT 1")
        prev_hash = prev_record["record_hash"] if prev_record else "0" * 64

        # 计算当前记录的哈希
        entry_data = json.dumps({
            "operation": operation,
            "data": data,
            "timestamp": now().isoformat(),
            "operator": current_user()
        }, sort_keys=True)
        record_hash = hashlib.sha256(
            (prev_hash + entry_data).encode()).hexdigest()

        # 写入数据库
        self.db.insert("audit_trail", {
            "operation": operation,
            "data": entry_data,
            "prev_hash": prev_hash,
            "record_hash": record_hash,
            "timestamp": now(),
            "operator": current_user()
        })

    def verify_integrity(self, start_id=None, end_id=None):
        """验证审计日志完整性"""
        records = self.db.query(
            "SELECT * FROM audit_trail WHERE id BETWEEN %s AND %s ORDER BY id",
            start_id or 0, end_id or 999999999)

        prev_hash = "0" * 64
        tampered = []
        for record in records:
            expected_hash = hashlib.sha256(
                (prev_hash + record["data"]).encode()).hexdigest()
            if record["record_hash"] != expected_hash:
                tampered.append({
                    "id": record["id"],
                    "expected": expected_hash,
                    "actual": record["record_hash"],
                    "operation": record["operation"]
                })
            prev_hash = record["record_hash"]

        return {
            "total_records": len(records),
            "tampered_count": len(tampered),
            "tampered_records": tampered,
            "is_intact": len(tampered) == 0
        }
```

**审计日志表：**

```sql
CREATE TABLE audit_trail (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    operation VARCHAR(100) NOT NULL,
    data TEXT NOT NULL,
    prev_hash CHAR(64) NOT NULL,
    record_hash CHAR(64) NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    operator VARCHAR(100) NOT NULL,
    INDEX idx_timestamp (timestamp),
    INDEX idx_operation (operation)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 防篡改触发器：禁止 UPDATE 和 DELETE
CREATE TRIGGER prevent_audit_update
BEFORE UPDATE ON audit_trail
FOR EACH ROW SIGNAL SQLSTATE '45000'
SET MESSAGE_TEXT = 'Audit trail is immutable: UPDATE not allowed';

CREATE TRIGGER prevent_audit_delete
BEFORE DELETE ON audit_trail
FOR EACH ROW SIGNAL SQLSTATE '45000'
SET MESSAGE_TEXT = 'Audit trail is immutable: DELETE not allowed';
```

## 异常场景补充

### 场景：借贷不平衡检测与预防

```
触发：JournalEntry 创建时 total_debits ≠ total_credits
检测：
  1. JournalEntry 类的 __post_init__ 方法自动校验
  2. 如果 |total_debits - total_credits| > 0.01 → 拒绝入账
  3. 数据库约束：CHECK (abs(total_debit - total_credit) < 0.01)
处理：
  1. 前端表单提交时实时校验借贷平衡
  2. 不平衡 → 高亮差异行 + 提示"借方差额 X 元"
  3. API 层二次校验：不平衡 → 400 Bad Request
预防：三层防御（前端 → API → 数据库约束）
```

### 场景：并发修改同一账户

```python
class AccountService:
    """带乐观锁的账户服务"""

    def update_balance(self, account_code, amount, entry_type):
        # 乐观锁：读取 version，更新时检查 version 是否变化
        account = self.db.get_account(account_code)
        affected = self.db.execute(
            "UPDATE accounts SET balance = balance + %s, version = version + 1 "
            "WHERE account_code = %s AND version = %s",
            amount, account_code, account["version"])

        if affected == 0:
            # version 不匹配 → 被其他事务修改 → 重试
            raise OptimisticLockError(
                f"Account {account_code} was modified by another transaction")
```

### 场景：期间结账部分失败回滚

```
触发：结账过程中第 3 步（生成结账分录）失败
      但第 1-2 步（试算平衡、调整分录）已完成
处理：
  1. 所有结账操作在同一个数据库事务中执行
  2. 任何步骤失败 → 整个事务回滚 → 不会有部分完成的状态
  3. 事务隔离级别：SERIALIZABLE（防止并发结账）
  4. 结账前先获取分布式锁：lock_key = f"period_close:{period_date}"
预防：结账是单线程操作 + 分布式锁 + 事务原子性保证
```

## 多币种会计完整实现

```python
class MultiCurrencyAccountingService:
    """多币种会计处理"""

    def record_foreign_transaction(self, entry):
        """记录外币交易"""
        base_currency = "CNY"
        fx_rate = self.fx_service.get_rate(entry.currency, base_currency, entry.date)

        # 原始金额（外币）和折算金额（本币）同时记录
        self.db.insert("journal_entry_lines", {
            "entry_id": entry.id,
            "account_code": entry.account_code,
            "debit_amount": entry.debit_amount,
            "credit_amount": entry.credit_amount,
            "currency": entry.currency,
            "fx_rate": fx_rate,
            "debit_amount_base": entry.debit_amount * fx_rate,
            "credit_amount_base": entry.credit_amount * fx_rate,
        })

    def revaluate_at_period_end(self, period_date):
        """期末汇率重估：计算未实现汇兑损益"""
        # 找出所有有外币余额的账户
        foreign_accounts = self.db.query(
            "SELECT account_code, currency, SUM(debit_amount - credit_amount) as balance "
            "FROM journal_entry_lines WHERE currency != 'CNY' "
            "GROUP BY account_code, currency")

        revaluation_entries = []
        for account in foreign_accounts:
            current_rate = self.fx_service.get_rate(
                account["currency"], "CNY", period_date)
            # 原始折算金额
            original_base = self.db.query_one(
                "SELECT SUM(debit_amount_base - credit_amount_base) as base_balance "
                "FROM journal_entry_lines WHERE account_code = %s AND currency = %s",
                account["account_code"], account["currency"])["base_balance"]

            # 当前汇率下的折算金额
            current_base = account["balance"] * current_rate
            unrealized_gain_loss = current_base - original_base

            if abs(unrealized_gain_loss) > 0.01:
                revaluation_entries.append({
                    "account_code": account["account_code"],
                    "currency": account["currency"],
                    "unrealized_gain_loss": unrealized_gain_loss,
                    "original_rate": original_base / account["balance"] if account["balance"] else 0,
                    "current_rate": current_rate,
                })

                # 生成汇兑损益分录
                if unrealized_gain_loss > 0:
                    # 未实现汇兑收益
                    self.db.insert("journal_entry_lines", {
                        "entry_id": f"FX_REVAL_{period_date}_{account['account_code']}",
                        "account_code": account["account_code"],  # 外币资产增加
                        "debit_amount_base": unrealized_gain_loss,
                        "account_code_counterpart": "5501",  # 汇兑损益
                        "credit_amount_base": unrealized_gain_loss,
                        "entry_type": "fx_revaluation",
                        "currency": account["currency"],
                        "fx_rate": current_rate,
                    })

        return revaluation_entries
```

## 期间结账完整流程

```python
class PeriodCloseWorkflow:
    """会计期间结账完整流程"""

    def execute_close(self, period_date):
        """执行完整的期间结账"""
        # Step 1: 试算平衡
        trial = self._trial_balance(period_date)
        if not trial["balanced"]:
            raise UnbalancedError(f"试算不平衡: 借方={trial['total_debit']}, 贷方={trial['total_credit']}")

        # Step 2: 调整分录
        adjustments = self._adjusting_entries(period_date)
        for adj in adjustments:
            self.ledger.post(adj)
        self.db.insert("period_close_log", {
            "period": period_date, "step": "adjusting",
            "entry_count": len(adjustments), "completed_at": now()
        })

        # Step 3: 调整后试算平衡
        adjusted = self._trial_balance(period_date)
        if not adjusted["balanced"]:
            raise UnbalancedError("调整后试算不平衡")

        # Step 4: 结账分录
        closings = self._closing_entries(period_date)
        for closing in closings:
            self.ledger.post(closing)
        self.db.insert("period_close_log", {
            "period": period_date, "step": "closing",
            "entry_count": len(closings), "completed_at": now()
        })

        # Step 5: 锁定期间
        self.db.insert("accounting_periods", {
            "period_date": period_date, "status": "closed",
            "closed_at": now(), "closed_by": current_user()
        })
        return {"period": period_date, "status": "closed",
                "adjustments": len(adjustments), "closings": len(closings)}

    def _closing_entries(self, period_date):
        """结账分录：收入/费用 → 未分配利润"""
        entries = []
        retained_earnings = "3001"

        # 收入类科目余额 → 未分配利润
        revenues = self.db.query(
            "SELECT account_code, SUM(credit_amount_base - debit_amount_base) as balance "
            "FROM journal_entry_lines WHERE account_type='revenue' "
            "AND entry_date <= %s GROUP BY account_code", period_date)
        for rev in revenues:
            if rev["balance"] > 0:
                entries.append(JournalEntry(
                    lines=[
                        EntryLine(rev["account_code"], debit_base=rev["balance"]),
                        EntryLine(retained_earnings, credit_base=rev["balance"]),
                    ], entry_type="closing"))

        # 费用类科目余额 → 未分配利润
        expenses = self.db.query(
            "SELECT account_code, SUM(debit_amount_base - credit_amount_base) as balance "
            "FROM journal_entry_lines WHERE account_type='expense' "
            "AND entry_date <= %s GROUP BY account_code", period_date)
        for exp in expenses:
            if exp["balance"] > 0:
                entries.append(JournalEntry(
                    lines=[
                        EntryLine(retained_earnings, debit_base=exp["balance"]),
                        EntryLine(exp["account_code"], credit_base=exp["balance"]),
                    ], entry_type="closing"))
        return entries
```

## 异常场景补充

### 场景：借贷不平衡检测

```python
class BalancedEntryValidator:
    """借贷不平衡检测与预防"""
    def validate(self, entry):
        total_debit = sum(l.debit_amount for l in entry.lines)
        total_credit = sum(l.credit_amount for l in entry.lines)
        if abs(total_debit - total_credit) > 0.01:
            raise UnbalancedEntryError(
                f"借贷不平衡: 借方={total_debit}, 贷方={total_credit}, "
                f"差额={total_debit - total_credit}")
```

### 场景：并发修改账户（乐观锁）

```python
class AccountBalanceService:
    """带乐观锁的账户余额更新"""
    def update_balance(self, account_code, amount):
        account = self.db.get_account(account_code)
        affected = self.db.execute(
            "UPDATE accounts SET balance = balance + %s, version = version + 1 "
            "WHERE account_code = %s AND version = %s",
            amount, account_code, account["version"])
        if affected == 0:
            raise OptimisticLockError(
                f"Account {account_code} was modified concurrently")
```

### 场景：期间结账部分失败

```
触发：结账过程中数据库连接断开 → 调整分录已写入但结账分录未写入
处理：
  1. 所有结账操作在单个数据库事务中执行
  2. 事务隔离级别 SERIALIZABLE
  3. 任何步骤失败 → 整个事务回滚
  4. 结账前获取分布式锁 lock_key=f"period_close:{period_date}"
预防：单线程结账 + 分布式锁 + 事务原子性
```

## 多币种会计补充

### 场景：汇率锁定与执行汇率差异

```
触发：交易时锁定汇率 7.25 → 实际结算时汇率 7.30
      → 差异 0.05/USD → 10 万美元交易差异 5000 元
处理：
  1. 0-50bp 差异 → 平台吸收（在可控范围内）
  2. 50-200bp → 平台与用户各承担 50%
  3. >200bp → 通知用户确认是否继续
  4. 每日汇兑损失累计上限 → 超过自动暂停该币种交易
预防：汇率锁定有效期 10 分钟 + 自动续期 + 损失上限监控
```

### 场景：外币退款汇率差

```
触发：用户用 USD 支付 → 退款时 USD 汇率已变化
      → 退回的 CNY 金额与原支付金额不同
处理：
  1. 原路退款：按原汇率退回 → 平台吸收差异
  2. 差异记录在汇兑损益科目
  3. 退款金额 = 原支付金额 × 原汇率（用户不受汇率波动影响）
预防：退款策略统一为"原汇率退款"→ 简化用户理解
```

## 性能分析详细数据

**记账吞吐量基准：**

| 操作 | 单条延迟 | 批量(1000条) | 吞吐量 |
|------|---------|-------------|--------|
| 插入分录 | 2ms | 500ms | 2000 TPS |
| 批量插入 | - | 100ms | 10000 TPS |
| 查询余额 | 1ms | - | 10000 QPS |
| 试算平衡 | 2000ms | - | 0.5 QPS |
| 对账匹配 | 30ms/条 | 30s/100万 | - |

**存储增长：**

| 数据 | 日增量 | 月增量 | 年增量 |
|------|--------|--------|--------|
| 分录明细 | 100万行(500MB) | 15亿行(15GB) | 180亿行(180GB) |
| 账户余额快照 | 1万行(5MB) | 30万行(150MB) | 360万行(1.8GB) |
| 审计日志 | 200万行(2GB) | 6000万行(60GB) | 7.2亿行(720GB) |

**月度成本：**

| 组件 | 规格 | 月成本 |
|------|------|-------|
| 应用服务器 | 8c32G × 5台 | ¥3 万 |
| MySQL 主从 | 8c64G + SSD × 4台 | ¥4 万 |
| Redis (缓存+锁) | 8c32G × 3节点 | ¥1.5 万 |
| Kafka (审计日志) | 6节点 | ¥3 万 |
| **合计** | | **¥11.5 万** |

## 复式记账验证引擎

```python
class DoubleEntryValidator:
    """复式记账验证引擎：自动检查借贷平衡"""

    def validate_entry(self, entry):
        """验证分录是否合法"""
        errors = []

        # 1. 借贷平衡检查
        total_debit = sum(line.debit_amount for line in entry.lines)
        total_credit = sum(line.credit_amount for line in entry.lines)
        if abs(total_debit - total_credit) > 0.01:
            errors.append(f"借贷不平衡: 借方={total_debit}, 贷方={total_credit}, 差额={total_debit-total_credit}")

        # 2. 每行至少有借或贷一方有值
        for i, line in enumerate(entry.lines):
            if line.debit_amount == 0 and line.credit_amount == 0:
                errors.append(f"第 {i+1} 行借贷金额均为 0")
            if line.debit_amount < 0 or line.credit_amount < 0:
                errors.append(f"第 {i+1} 行金额为负数")

        # 3. 科目存在性检查
        for line in entry.lines:
            if not self.db.exists("accounts", account_code=line.account_code):
                errors.append(f"科目 {line.account_code} 不存在")

        # 4. 期间检查：不允许在已关闭期间入账
        if self.db.exists("accounting_periods",
            period_date=entry.entry_date, status="closed"):
            errors.append(f"期间 {entry.entry_date} 已关闭，不可入账")

        return {"valid": len(errors) == 0, "errors": errors}
```

## 财务报表生成

```python
class FinancialReportGenerator:
    """财务报表生成：资产负债表、利润表、现金流量表"""

    def generate_balance_sheet(self, as_of_date):
        """生成资产负债表"""
        assets = self.db.query(
            "SELECT a.account_code, a.account_name, "
            "COALESCE(SUM(l.debit_amount_base - l.credit_amount_base), 0) as balance "
            "FROM accounts a LEFT JOIN journal_entry_lines l ON a.account_code = l.account_code "
            "WHERE a.account_type = 'asset' AND l.entry_date <= %s "
            "GROUP BY a.account_code, a.account_name ORDER BY a.account_code", as_of_date)

        liabilities = self.db.query(
            "SELECT a.account_code, a.account_name, "
            "COALESCE(SUM(l.credit_amount_base - l.debit_amount_base), 0) as balance "
            "FROM accounts a LEFT JOIN journal_entry_lines l ON a.account_code = l.account_code "
            "WHERE a.account_type = 'liability' AND l.entry_date <= %s "
            "GROUP BY a.account_code, a.account_name ORDER BY a.account_code", as_of_date)

        equity = self.db.query(
            "SELECT a.account_code, a.account_name, "
            "COALESCE(SUM(l.credit_amount_base - l.debit_amount_base), 0) as balance "
            "FROM accounts a LEFT JOIN journal_entry_lines l ON a.account_code = l.account_code "
            "WHERE a.account_type = 'equity' AND l.entry_date <= %s "
            "GROUP BY a.account_code, a.account_name ORDER BY a.account_code", as_of_date)

        total_assets = sum(a["balance"] for a in assets)
        total_liabilities = sum(l["balance"] for l in liabilities)
        total_equity = sum(e["balance"] for e in equity)

        return {
            "as_of_date": as_of_date,
            "assets": {"items": assets, "total": total_assets},
            "liabilities": {"items": liabilities, "total": total_liabilities},
            "equity": {"items": equity, "total": total_equity},
            "is_balanced": abs(total_assets - (total_liabilities + total_equity)) < 0.01
        }

    def generate_income_statement(self, start_date, end_date):
        """生成利润表"""
        revenue = self.db.query(
            "SELECT a.account_code, a.account_name, "
            "SUM(l.credit_amount_base - l.debit_amount_base) as balance "
            "FROM accounts a JOIN journal_entry_lines l ON a.account_code = l.account_code "
            "WHERE a.account_type = 'revenue' AND l.entry_date BETWEEN %s AND %s "
            "GROUP BY a.account_code, a.account_name", start_date, end_date)

        expenses = self.db.query(
            "SELECT a.account_code, a.account_name, "
            "SUM(l.debit_amount_base - l.credit_amount_base) as balance "
            "FROM accounts a JOIN journal_entry_lines l ON a.account_code = l.account_code "
            "WHERE a.account_type = 'expense' AND l.entry_date BETWEEN %s AND %s "
            "GROUP BY a.account_code, a.account_name", start_date, end_date)

        total_revenue = sum(r["balance"] for r in revenue)
        total_expenses = sum(e["balance"] for e in expenses)

        return {
            "period": f"{start_date} ~ {end_date}",
            "revenue": {"items": revenue, "total": total_revenue},
            "expenses": {"items": expenses, "total": total_expenses},
            "net_income": total_revenue - total_expenses
        }
```

## 异常场景补充

### 场景：重复分录检测

```python
class DuplicateEntryDetector:
    """检测重复分录"""
    def check_duplicate(self, entry):
        """检查是否为重复分录"""
        # 按摘要+金额+日期+科目匹配
        existing = self.db.query(
            "SELECT * FROM journal_entries WHERE "
            "description = %s AND entry_date = %s AND "
            "ABS(total_debit - %s) < 0.01 AND created_at > NOW() - INTERVAL 1 HOUR",
            entry.description, entry.entry_date, entry.total_debit)
        if existing:
            return {"is_duplicate": True, "existing_id": existing[0]["id"]}
        return {"is_duplicate": False}
```

### 场景：已关闭期间违规入账

```
触发：用户尝试在已关闭的 2025-12 期间录入分录
检测：validate_entry 中检查 accounting_periods 表
处理：
  1. 拒绝入账，返回错误"期间已关闭"
  2. 如需调整 → 创建调整分录在当前打开期间
  3. 重新打开期间需要总监审批 + 审计日志
预防：入账前强制检查期间状态 + UI 禁止选择已关闭期间
```

### 场景：账户余额变为负数

```
触发：负债类账户余额变为正数（贷方余额 < 借方余额）
      → 逻辑错误：应付账款不可能为负
检测：
  1. 每次入账后检查相关账户余额
  2. 资产类余额 < 0 或负债类余额 < 0 → 告警
处理：
  1. 阻止入账：验证分录是否会导致余额为负
  2. 如已入账 → 标记为异常 + 通知会计主管
  3. 需要调整分录修正余额
预防：按账户类型设置余额方向约束
```

## 现金流量表生成

```python
class CashFlowStatementGenerator:
    """现金流量表生成"""

    def generate(self, start_date, end_date):
        """生成现金流量表"""
        # 经营活动现金流
        operating_cash = self._calc_operating_cash(start_date, end_date)
        # 投资活动现金流
        investing_cash = self._calc_investing_cash(start_date, end_date)
        # 筹资活动现金流
        financing_cash = self._calc_financing_cash(start_date, end_date)

        net_change = operating_cash["net"] + investing_cash["net"] + financing_cash["net"]

        return {
            "period": f"{start_date} ~ {end_date}",
            "operating": operating_cash,
            "investing": investing_cash,
            "financing": financing_cash,
            "net_change_in_cash": net_change,
            "beginning_cash": self._get_cash_balance(start_date - timedelta(days=1)),
            "ending_cash": self._get_cash_balance(end_date)
        }

    def _calc_operating_cash(self, start, end):
        """经营活动现金流"""
        # 净利润 + 非现金项目调整
        net_income = self.db.query_one(
            "SELECT SUM(credit_amount_base - debit_amount_base) as net "
            "FROM journal_entry_lines WHERE account_type IN ('revenue','expense') "
            "AND entry_date BETWEEN %s AND %s", start, end)["net"]

        # 折旧（非现金支出，加回）
        depreciation = self.db.query_one(
            "SELECT SUM(debit_amount_base) as dep "
            "FROM journal_entry_lines WHERE account_code LIKE '6602%' "
            "AND entry_date BETWEEN %s AND %s", start, end)["dep"]

        # 应收账款变化（增加 → 现金减少）
        ar_change = self._get_balance_change("1122", start, end)

        # 应付账款变化（增加 → 现金增加）
        ap_change = self._get_balance_change("2202", start, end)

        net_cash = net_income + depreciation - ar_change + ap_change
        return {
            "net_income": net_income,
            "depreciation": depreciation,
            "ar_change": ar_change,
            "ap_change": ap_change,
            "net": net_cash
        }
```

## 性能分析详细数据

**记账系统吞吐量：**

| 操作 | QPS | 延迟 P99 | 说明 |
|------|-----|---------|------|
| 插入分录 | 2000 | 5ms | 含借贷平衡校验 |
| 查询余额 | 10000 | 2ms | Redis 缓存 |
| 试算平衡 | 0.5 | 2s | 全量扫描 |
| 对账匹配 | 0.1 | 30s | 100 万条匹配 |
| 生成报表 | 0.05 | 10s | 含 3 张报表 |

**月度基础设施成本：**

| 组件 | 规格 | 月成本 |
|------|------|-------|
| 应用服务器 | 8c32G × 5台 | ¥3 万 |
| MySQL 主从 | 8c64G + SSD × 4台 | ¥4 万 |
| Redis | 8c32G × 3节点 | ¥1.5 万 |
| Kafka | 8c32G × 6节点 | ¥3 万 |
| **合计** | | **¥11.5 万** |

**存储增长预估：**

| 数据 | 日增量 | 月增量 | 年增量 |
|------|--------|--------|--------|
| 分录明细 | 500MB | 15GB | 180GB |
| 审计日志 | 2GB | 60GB | 720GB |
| 账户快照 | 5MB | 150MB | 1.8GB |

## 冻结/解冻机制完整实现

```python
class AccountFreezeService:
    """账户冻结/解冻机制（提现场景）"""

    # 状态机: available → frozen → deducted / unfrozen
    TRANSITIONS = {
        "available": ["frozen"],
        "frozen": ["deducted", "unfrozen"],
        "deducted": [],
        "unfrozen": ["frozen"],
    }

    def freeze(self, account_id, amount, reason):
        """冻结金额"""
        account = self.db.get_account(account_id)
        if account["available_balance"] < amount:
            raise InsufficientBalanceError(f"可用余额不足: {account['available_balance']} < {amount}")

        # 检查状态转换合法性
        self._validate_transition("available", "frozen")

        with self.db.transaction():
            # 1. 扣减可用余额 + 增加冻结金额
            self.db.execute(
                "UPDATE accounts SET available_balance = available_balance - %s, "
                "frozen_balance = frozen_balance + %s, version = version + 1 "
                "WHERE account_id = %s AND version = %s",
                amount, amount, account_id, account["version"])

            # 2. 创建冻结记录
            freeze_id = str(uuid4())
            self.db.insert("account_freeze_log", {
                "freeze_id": freeze_id, "account_id": account_id,
                "amount": amount, "reason": reason,
                "status": "frozen", "frozen_at": now(),
                "expire_at": now() + timedelta(hours=24)  # 24小时未扣款自动解冻
            })

            # 3. 创建冻结分录（借贷平衡）
            self.ledger.post_entry(JournalEntry(
                lines=[
                    EntryLine("1001", debit=amount),  # 冻结资产增加
                    EntryLine("1002", credit=amount),  # 可用资产减少
                ],
                entry_type="freeze",
                description=f"冻结: {reason}"
            ))

        return freeze_id

    def deduct(self, freeze_id):
        """扣款（银行到账确认后）"""
        freeze = self.db.get_freeze(freeze_id)
        self._validate_transition(freeze["status"], "deducted")

        with self.db.transaction():
            # 1. 扣减冻结余额
            self.db.execute(
                "UPDATE accounts SET frozen_balance = frozen_balance - %s, "
                "version = version + 1 WHERE account_id = %s",
                freeze["amount"], freeze["account_id"])

            # 2. 更新冻结记录
            self.db.update("account_freeze_log",
                {"status": "deducted", "deducted_at": now()},
                {"freeze_id": freeze_id})

            # 3. 创建扣款分录
            self.ledger.post_entry(JournalEntry(
                lines=[
                    EntryLine("2001", debit=freeze["amount"]),  # 应付减少
                    EntryLine("1001", credit=freeze["amount"]),  # 冻结资产减少
                ],
                entry_type="deduct",
                description=f"扣款: {freeze['reason']}"
            ))

    def unfreeze(self, freeze_id):
        """解冻（取消提现）"""
        freeze = self.db.get_freeze(freeze_id)
        self._validate_transition(freeze["status"], "unfrozen")

        with self.db.transaction():
            # 1. 恢复可用余额 + 减少冻结余额
            self.db.execute(
                "UPDATE accounts SET available_balance = available_balance + %s, "
                "frozen_balance = frozen_balance - %s, version = version + 1 "
                "WHERE account_id = %s",
                freeze["amount"], freeze["amount"], freeze["account_id"])

            # 2. 更新冻结记录
            self.db.update("account_freeze_log",
                {"status": "unfrozen", "unfrozen_at": now()},
                {"freeze_id": freeze_id})

            # 3. 创建解冻分录（冲销冻结分录）
            self.ledger.post_entry(JournalEntry(
                lines=[
                    EntryLine("1002", debit=freeze["amount"]),  # 可用资产恢复
                    EntryLine("1001", credit=freeze["amount"]),  # 冻结资产减少
                ],
                entry_type="unfreeze",
                description=f"解冻: {freeze['reason']}"
            ))

    def auto_expire_freezes(self):
        """自动解冻超时冻结（24小时未扣款）"""
        expired = self.db.query(
            "SELECT * FROM account_freeze_log "
            "WHERE status = 'frozen' AND expire_at <= NOW()")
        for freeze in expired:
            self.unfreeze(freeze["freeze_id"])
            self.notify(f"冻结已过期自动解冻: {freeze['freeze_id']}")
```

**冻结记录表：**

```sql
CREATE TABLE account_freeze_log (
    freeze_id VARCHAR(36) PRIMARY KEY,
    account_id VARCHAR(36) NOT NULL,
    amount DECIMAL(18,2) NOT NULL,
    reason VARCHAR(200) NOT NULL,
    status ENUM('frozen', 'deducted', 'unfrozen') NOT NULL DEFAULT 'frozen',
    frozen_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deducted_at TIMESTAMP NULL,
    unfrozen_at TIMESTAMP NULL,
    expire_at TIMESTAMP NOT NULL,
    INDEX idx_account_status (account_id, status),
    INDEX idx_expire (status, expire_at)
);
```

## 银行对账 API 集成

```python
class BankReconciliationService:
    """银行对账：自动获取银行流水并匹配"""

    def daily_reconciliation(self, date):
        """日终自动对账"""
        # 1. 获取银行对账单
        try:
            bank_statement = self.bank_api.get_statement(date)
        except BankAPITimeoutError:
            # 银行 API 超时 → 稍后重试
            self.db.insert("reconciliation_retry", {
                "date": date, "retry_count": 0, "next_retry_at": now() + timedelta(minutes=30)
            })
            return {"status": "pending", "reason": "bank_api_timeout"}

        # 2. 获取内部流水
        internal_entries = self.db.query(
            "SELECT * FROM journal_entries WHERE entry_date = %s AND entry_type = 'payment'",
            date)

        # 3. 按交易号匹配
        matched, unmatched_int, unmatched_bank = self._match_by_ref(
            internal_entries, bank_statement)

        # 4. 检查金额一致性
        for int_entry, bank_entry in matched:
            if abs(int_entry.amount - bank_entry.amount) > 0.01:
                self.db.insert("reconciliation_discrepancy", {
                    "date": date,
                    "ref_id": int_entry.ref_id,
                    "internal_amount": int_entry.amount,
                    "bank_amount": bank_entry.amount,
                    "diff": abs(int_entry.amount - bank_entry.amount),
                    "type": "amount_mismatch"
                })

        # 5. 生成对账报告
        report = {
            "date": date,
            "total_internal": len(internal_entries),
            "total_bank": len(bank_statement),
            "matched": len(matched),
            "unmatched_internal": len(unmatched_int),
            "unmatched_bank": len(unmatched_bank),
            "discrepancies": self.db.count("reconciliation_discrepancy", date=date)
        }
        self.db.insert("reconciliation_reports", report)
        return report
```

## 异常场景补充

### 场景：冻结金额超过可用余额

```python
# 在 freeze() 方法中已处理
if account["available_balance"] < amount:
    raise InsufficientBalanceError(
        f"可用余额 {account['available_balance']} < 冻结金额 {amount}")
```

### 场景：银行 API 超时

```
触发：对账时银行 API 响应超过 30 秒
检测：HTTP 请求超时
处理：
  1. 记录到 reconciliation_retry 表
  2. 30 分钟后自动重试
  3. 连续 3 次失败 → 人工介入
预防：银行 API 连接池 + 超时设置 + 自动重试
```

### 场景：哈希链篡改检测

```python
class AuditTrailIntegrityChecker:
    """审计日志完整性校验"""
    def verify_chain(self, start_id=None, end_id=None):
        records = self.db.query(
            "SELECT * FROM audit_trail ORDER BY id "
            "WHERE id BETWEEN %s AND %s", start_id or 0, end_id or 999999999)
        prev_hash = "0" * 64
        tampered = []
        for record in records:
            expected = hashlib.sha256(
                (prev_hash + record["data"]).encode()).hexdigest()
            if record["record_hash"] != expected:
                tampered.append(record["id"])
            prev_hash = record["record_hash"]
        if tampered:
            self.alert(f"审计日志被篡改! 记录ID: {tampered}")
        return {"tampered_count": len(tampered), "tampered_ids": tampered}
```

## 账户对账引擎完整实现

```python
class ReconciliationEngine:
    """账户对账引擎：自动匹配 + 差异检测 + 人工复核"""

    def daily_reconciliation(self, account_id, date):
        """日终自动对账"""
        # 1. 获取账户当日所有流水
        entries = self.db.query(
            "SELECT * FROM journal_entry_lines WHERE account_code LIKE %s "
            "AND entry_date = %s ORDER BY entry_date, entry_id",
            f"{account_id}%", date)

        # 2. 获取银行对账单
        bank_stmt = self.bank_api.get_statement(account_id, date)

        # 3. 双向匹配
        matched, unmatched_int, unmatched_bank = self._bidirectional_match(
            entries, bank_stmt)

        # 4. 生成对账报告
        report = {
            "account_id": account_id,
            "date": date,
            "internal_count": len(entries),
            "bank_count": len(bank_stmt),
            "matched_count": len(matched),
            "unmatched_internal": len(unmatched_int),
            "unmatched_bank": len(unmatched_bank),
            "status": "balanced" if not unmatched_int and not unmatched_bank else "discrepancy"
        }

        # 5. 处理未匹配项
        for item in unmatched_int:
            self.db.insert("reconciliation_exceptions", {
                "account_id": account_id, "date": date,
                "source": "internal", "ref_id": item["entry_id"],
                "amount": item["amount"], "status": "pending_review"
            })
        for item in unmatched_bank:
            self.db.insert("reconciliation_exceptions", {
                "account_id": account_id, "date": date,
                "source": "bank", "ref_id": item["transaction_id"],
                "amount": item["amount"], "status": "pending_review"
            })

        return report

    def _bidirectional_match(self, internal, bank):
        """双向匹配：金额 + 日期 + 参考号"""
        matched = []
        unmatched_int = list(internal)
        unmatched_bank = list(bank)

        for int_item in list(unmatched_int):
            best_match = None
            best_diff = float('inf')
            for bank_item in list(unmatched_bank):
                diff = abs(int_item["amount"] - bank_item["amount"])
                if diff < best_diff and diff <= 0.01:  # 允许 1 分钱差异
                    if int_item.get("ref_id") == bank_item.get("ref_id"):
                        best_match = bank_item
                        best_diff = diff
                        break  # 参考号完全匹配 → 优先
                    elif best_diff > diff:
                        best_match = bank_item
                        best_diff = diff

            if best_match:
                matched.append((int_item, best_match))
                unmatched_int.remove(int_item)
                unmatched_bank.remove(best_match)

        return matched, unmatched_int, unmatched_bank

    def review_exception(self, exception_id, action, adjustment_entry=None):
        """人工复核异常"""
        exception = self.db.get_exception(exception_id)
        if action == "adjust":
            # 创建调整分录
            self.ledger.post_entry(adjustment_entry)
            self.db.update("reconciliation_exceptions",
                {"status": "adjusted", "adjusted_at": now(), "adjustment_entry_id": adjustment_entry.id},
                {"id": exception_id})
        elif action == "ignore":
            self.db.update("reconciliation_exceptions",
                {"status": "ignored", "ignored_at": now()},
                {"id": exception_id})
        elif action == "escalate":
            self.db.update("reconciliation_exceptions",
                {"status": "escalated"}, {"id": exception_id})
            self.notify_supervisor(exception)
```

## 账户快照与历史查询

```python
class AccountSnapshotService:
    """账户快照：任意时间点余额查询"""

    def get_balance_as_of(self, account_code, as_of_date):
        """查询任意历史时间点的账户余额"""
        # 优先查快照表
        snapshot = self.db.query_one(
            "SELECT balance FROM account_balance_snapshots "
            "WHERE account_code = %s AND snapshot_date <= %s "
            "ORDER BY snapshot_date DESC LIMIT 1",
            account_code, as_of_date)

        if snapshot:
            # 从快照日期到目标日期的增量
            delta = self.db.query_one(
                "SELECT COALESCE(SUM(debit_amount_base - credit_amount_base), 0) as delta "
                "FROM journal_entry_lines WHERE account_code = %s "
                "AND entry_date > %s AND entry_date <= %s",
                account_code, snapshot["snapshot_date"], as_of_date)
            return snapshot["balance"] + delta["delta"]
        else:
            # 无快照 → 全量计算
            return self.db.query_one(
                "SELECT COALESCE(SUM(debit_amount_base - credit_amount_base), 0) as balance "
                "FROM journal_entry_lines WHERE account_code = %s AND entry_date <= %s",
                account_code, as_of_date)["balance"]

    def take_daily_snapshot(self):
        """每日快照：加速历史查询"""
        accounts = self.db.query("SELECT account_code FROM accounts")
        for acct in accounts:
            balance = self.db.query_one(
                "SELECT COALESCE(SUM(debit_amount_base - credit_amount_base), 0) as bal "
                "FROM journal_entry_lines WHERE account_code = %s AND entry_date <= CURRENT_DATE",
                acct["account_code"])["bal"]
            self.db.insert("account_balance_snapshots", {
                "account_code": acct["account_code"],
                "balance": balance,
                "snapshot_date": now().date()
            })
```

**快照表 DDL：**

```sql
CREATE TABLE account_balance_snapshots (
    account_code VARCHAR(20) NOT NULL,
    balance DECIMAL(18,2) NOT NULL,
    snapshot_date DATE NOT NULL,
    PRIMARY KEY (account_code, snapshot_date),
    INDEX idx_date (snapshot_date)
);
```

## 异常场景补充

### 场景：对账差异自动分类

```python
class ReconciliationClassifier:
    """对账差异自动分类"""
    def classify(self, exception):
        if exception["source"] == "internal":
            # 内部有记录但银行没有 → 可能是时间差
            entry = self.db.get_entry(exception["ref_id"])
            if entry["entry_date"] == now().date():
                return {"type": "timing", "action": "wait_next_day"}
            return {"type": "missing_bank", "action": "contact_bank"}
        else:
            # 银行有记录但内部没有 → 可能是漏记
            return {"type": "missing_entry", "action": "create_adjustment"}
```

### 场景：快照数据损坏

```
触发：account_balance_snapshots 表数据被误删 → 历史查询变慢
检测：
  1. 快照行数 < 活跃账户数 → 数据缺失
  2. 历史查询延迟 > 10s → 可能缺少快照
处理：
  1. 从 journal_entry_lines 重新计算并生成快照
  2. 重建脚本：遍历所有日期和账户
预防：快照表每日备份 + 行数校验
```

## 多币种账户完整实现

```python
class MultiCurrencyAccountService:
    """多币种账户管理"""

    SUPPORTED_CURRENCIES = ["CNY", "USD", "EUR", "JPY", "GBP"]

    def create_account(self, user_id, currency="CNY"):
        """创建多币种账户"""
        if currency not in self.SUPPORTED_CURRENCIES:
            raise UnsupportedCurrencyError(f"不支持的币种: {currency}")

        account_id = str(uuid4())
        self.db.insert("multi_currency_accounts", {
            "account_id": account_id,
            "user_id": user_id,
            "currency": currency,
            "balance": 0,
            "frozen_balance": 0,
            "available_balance": 0,
            "version": 1,
            "created_at": now()
        })
        return account_id

    def transfer_between_currencies(self, user_id, from_currency, to_currency, amount):
        """币种间转换（先卖出后买入）"""
        if from_currency == to_currency:
            raise SameCurrencyError("源币种和目标币种相同")

        # 1. 锁定汇率
        rate_lock = self.fx_manager.lock_rate(
            str(uuid4()), from_currency, to_currency)

        # 2. 扣减源币种
        from_account = self._get_account(user_id, from_currency)
        if from_account["available_balance"] < amount:
            raise InsufficientBalanceError("余额不足")

        self.db.execute(
            "UPDATE multi_currency_accounts SET "
            "available_balance = available_balance - %s, version = version + 1 "
            "WHERE account_id = %s AND version = %s",
            amount, from_account["account_id"], from_account["version"])

        # 3. 增加目标币种
        target_amount = amount * rate_lock["rate"]
        to_account = self._get_account(user_id, to_currency)
        self.db.execute(
            "UPDATE multi_currency_accounts SET "
            "available_balance = available_balance + %s, version = version + 1 "
            "WHERE account_id = %s AND version = %s",
            target_amount, to_account["account_id"], to_account["version"])

        # 4. 创建分录
        self.ledger.post_entry(JournalEntry(
            lines=[
                EntryLine(f"2101_{to_currency}", debit=target_amount),
                EntryLine(f"1101_{from_currency}", credit=amount),
            ],
            entry_type="currency_conversion",
            description=f"币种转换: {amount} {from_currency} → {target_amount:.2f} {to_currency} @ {rate_lock['rate']}"
        ))

        return {"from_amount": amount, "to_amount": round(target_amount, 2),
                "rate": rate_lock["rate"]}
```

**多币种账户表：**

```sql
CREATE TABLE multi_currency_accounts (
    account_id VARCHAR(36) PRIMARY KEY,
    user_id VARCHAR(36) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    balance DECIMAL(18,2) NOT NULL DEFAULT 0,
    frozen_balance DECIMAL(18,2) NOT NULL DEFAULT 0,
    available_balance DECIMAL(18,2) NOT NULL DEFAULT 0,
    version BIGINT NOT NULL DEFAULT 1,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_user_currency (user_id, currency),
    INDEX idx_user (user_id)
);
```

## 异常场景补充

### 场景：汇率锁定过期导致金额不一致

```
触发：汇率锁定 10 分钟过期 → 币种转换完成但汇率已变
      → 实际兑换金额与锁定时不一致
检测：
  1. 转换时检查 rate_lock 是否有效
  2. 过期 → 拒绝转换，要求重新锁定
处理：
  1. 币种转换在汇率锁定有效期内完成
  2. 超时 → 回滚源币种扣减 → 重新锁定汇率
预防：汇率锁定 TTL 内完成转换 + 超时自动回滚
```

### 场景：并发转账导致余额为负

```
触发：两个并发请求同时扣减同一账户 → 余额变为负数
检测：
  1. 余额 < 0 → 数据一致性错误
  2. 乐观锁：UPDATE WHERE version = X → affected=0 → 冲突
处理：
  1. 使用乐观锁：只有 version 匹配时才能更新
  2. 冲突时重试（重新读取最新余额再扣减）
  3. 数据库约束：CHECK (available_balance >= 0)
预防：乐观锁 + 数据库 CHECK 约束 + 重试机制
```

## 日终结算完整实现

```python
class DailySettlementService:
    """日终结算：资金清算 + 手续费分润"""

    def settle(self, date):
        """执行日终结算"""
        # 1. 汇总当日所有交易
        transactions = self.db.query(
            "SELECT * FROM transactions WHERE DATE(created_at) = %s "
            "AND status = 'completed'", date)

        # 2. 按商户分组结算
        merchant_totals = {}
        for tx in transactions:
            mid = tx["merchant_id"]
            if mid not in merchant_totals:
                merchant_totals[mid] = {"gross": 0, "fee": 0, "net": 0}
            merchant_totals[mid]["gross"] += tx["amount"]
            fee = tx["amount"] * tx["fee_rate"]
            merchant_totals[mid]["fee"] += fee
            merchant_totals[mid]["net"] += tx["amount"] - fee

        # 3. 生成结算单
        settlement_id = str(uuid4())
        for mid, totals in merchant_totals.items():
            self.db.insert("merchant_settlements", {
                "settlement_id": settlement_id,
                "merchant_id": mid,
                "date": date,
                "gross_amount": totals["gross"],
                "fee_amount": totals["fee"],
                "net_amount": totals["net"],
                "status": "pending_transfer",
                "created_at": now()
            })

            # 4. 执行资金划拨
            self.bank_client.transfer({
                "from": "platform_account",
                "to": mid,
                "amount": totals["net"],
                "reference": settlement_id
            })

        # 5. 平台手续费收入入账
        total_fees = sum(t["fee"] for t in merchant_totals.values())
        self.ledger.post_entry(JournalEntry(
            lines=[
                EntryLine("1001_bank", debit=total_fees),
                EntryLine("6001_fee_income", credit=total_fees),
            ],
            entry_type="fee_settlement",
            description=f"日终手续费结算 {date}"
        ))

        return {"settlement_id": settlement_id,
                "merchant_count": len(merchant_totals),
                "total_fees": total_fees}
```

## 利息计算引擎

```python
class InterestCalculationEngine:
    """利息计算引擎：日息/月息/复利"""

    def calculate_daily_interest(self, account_id, date):
        """计算日利息"""
        account = self.db.get_account(account_id)
        balance = account["balance"]
        rate = account["interest_rate"]  # 年化利率

        # 日利息 = 余额 × 年利率 / 365
        daily_interest = balance * rate / 365

        # 记入利息
        self.db.insert("interest_accruals", {
            "account_id": account_id,
            "date": date,
            "balance": balance,
            "rate": rate,
            "interest_amount": round(daily_interest, 2),
            "created_at": now()
        })

        return round(daily_interest, 2)

    def compound_monthly(self, account_id, month):
        """月度复利结算"""
        # 获取当月累计利息
        total_interest = self.db.query_one(
            "SELECT SUM(interest_amount) as total FROM interest_accruals "
            "WHERE account_id = %s AND DATE_FORMAT(date, '%%Y-%%m') = %s",
            account_id, month)["total"]

        if total_interest and total_interest > 0:
            # 利息转入本金
            self.ledger.post_entry(JournalEntry(
                lines=[
                    EntryLine("2101_interest_payable", debit=total_interest),
                    EntryLine("1101_deposit", credit=total_interest),
                ],
                entry_type="interest_capitalization",
                description=f"月度利息结转 {month}"
            ))

        return total_interest or 0
```

## 异常场景补充

### 场景：日终结算银行划拨失败

```
触发：银行 API 超时 → 商户结算资金未到账
检测：
  1. 银行转账 API 返回超时 → 告警
  2. 结算单状态 24 小时仍为 pending_transfer → 告警
处理：
  1. 查询银行交易状态（确认是否已执行）
  2. 已执行 → 更新结算单状态
  3. 未执行 → 重试划拨
  4. 重试 3 次仍失败 → 人工处理
预防：银行 API 超时重试 + 状态查询 + 人工兜底
```

### 场景：利息计算精度丢失

```
触发：浮点数精度问题 → 累计利息与预期差 0.01 元
检测：
  1. 月末利息汇总 ≠ 日利息之和 → 精度误差
  2. 差异 > 0.01 → 告警
处理：
  1. 使用 Decimal 类型替代 float
  2. 利息计算统一使用 4 舍 5 入到分
  3. 差异自动调整到"利息调整"科目
预防：全程使用 Decimal + 规范舍入规则
```

## 账户层级管理完整实现

```python
class AccountHierarchyService:
    """账户层级管理：总账→分类账→明细账"""

    HIERARCHY = {
        "1": "资产类",
        "2": "负债类",
        "3": "共同类",
        "4": "所有者权益类",
        "5": "成本类",
        "6": "损益类",
    }

    def create_account(self, code, name, parent_code=None):
        """创建账户（层级结构）"""
        level = len(code)  # 1级=1位, 2级=2位, 3级=4位, 4级=6位
        if parent_code:
            parent = self.db.get_account(parent_code)
            level = parent["level"] + 1

        self.db.insert("accounts", {
            "account_code": code,
            "account_name": name,
            "parent_code": parent_code,
            "level": level,
            "category": self.HIERARCHY.get(code[0], "未知"),
            "balance_direction": "debit" if code[0] in ["1", "5"] else "credit",
            "is_leaf": level >= 4,  # 4级为最底层明细账
            "created_at": now()
        })

    def get_account_tree(self, root_code=None):
        """获取账户树结构"""
        accounts = self.db.query(
            "SELECT * FROM accounts ORDER BY account_code")
        tree = {}
        for acct in accounts:
            tree[acct["account_code"]] = {
                "code": acct["account_code"],
                "name": acct["account_name"],
                "level": acct["level"],
                "children": []
            }
        roots = []
        for acct in accounts:
            parent = tree.get(acct["parent_code"])
            if parent:
                parent["children"].append(tree[acct["account_code"]])
            else:
                roots.append(tree[acct["account_code"]])
        return roots

    def get_leaf_account_balances(self, date):
        """获取所有明细账余额"""
        return self.db.query(
            "SELECT a.account_code, a.account_name, "
            "COALESCE(SUM(jl.debit_amount_base - jl.credit_amount_base), 0) as balance "
            "FROM accounts a "
            "LEFT JOIN journal_entry_lines jl ON a.account_code = jl.account_code "
            "AND jl.entry_date <= %s "
            "WHERE a.is_leaf = 1 "
            "GROUP BY a.account_code, a.account_name "
            "ORDER BY a.account_code", date)
```

## 财务报表生成

```python
class FinancialReportGenerator:
    """财务报表生成：资产负债表 + 利润表"""

    def generate_balance_sheet(self, date):
        """生成资产负债表"""
        assets = self.db.query(
            "SELECT a.category, SUM(bal.balance) as total "
            "FROM accounts a JOIN account_balances bal ON a.account_code = bal.account_code "
            "WHERE a.account_code LIKE '1%%' AND bal.date = %s "
            "GROUP BY a.category", date)
        liabilities = self.db.query(
            "SELECT a.category, SUM(bal.balance) as total "
            "FROM accounts a JOIN account_balances bal ON a.account_code = bal.account_code "
            "WHERE a.account_code LIKE '2%%' AND bal.date = %s "
            "GROUP BY a.category", date)
        equity = self.db.query(
            "SELECT a.category, SUM(bal.balance) as total "
            "FROM accounts a JOIN account_balances bal ON a.account_code = bal.account_code "
            "WHERE a.account_code LIKE '4%%' AND bal.date = %s "
            "GROUP BY a.category", date)

        total_assets = sum(a["total"] for a in assets)
        total_liabilities = sum(l["total"] for l in liabilities)
        total_equity = sum(e["total"] for e in equity)

        return {
            "date": date,
            "assets": {"items": assets, "total": total_assets},
            "liabilities": {"items": liabilities, "total": total_liabilities},
            "equity": {"items": equity, "total": total_equity},
            "balanced": abs(total_assets - (total_liabilities + total_equity)) < 0.01
        }

    def generate_income_statement(self, start_date, end_date):
        """生成利润表"""
        revenue = self.db.query_one(
            "SELECT COALESCE(SUM(credit_amount_base), 0) as total "
            "FROM journal_entry_lines WHERE account_code LIKE '6%%' "
            "AND entry_date BETWEEN %s AND %s", start_date, end_date)
        expenses = self.db.query_one(
            "SELECT COALESCE(SUM(debit_amount_base), 0) as total "
            "FROM journal_entry_lines WHERE account_code LIKE '5%%' "
            "AND entry_date BETWEEN %s AND %s", start_date, end_date)

        net_income = revenue["total"] - expenses["total"]
        return {
            "period": f"{start_date} ~ {end_date}",
            "revenue": revenue["total"],
            "expenses": expenses["total"],
            "net_income": net_income
        }
```

## 异常场景补充

### 场景：试算平衡表不平

```
触发：借方合计 ≠ 贷方合计 → 试算平衡失败
检测：
  1. 日终试算平衡：SUM(debit) vs SUM(credit)
  2. 差异 > 0.01 → 报错
处理：
  1. 逐笔检查分录：借贷是否相等
  2. 找到不平衡的分录 → 手动调整
  3. 调整后重新试算
预防：分录创建时强制借贷平衡 + 自动试算
```

### 场景：财务报表生成超时

```
触发：月末报表生成 SQL 执行超过 30 分钟 → 超时
检测：报表生成耗时 > 10 分钟 → 告警
处理：
  1. 使用预聚合表加速查询
  2. 报表数据分批计算
  3. 增加并行处理
预防：预聚合 + 分批计算 + 定期刷新余额快照
```

## 监管报表模块完整实现

```python
class RegulatoryReportService:
    """监管报表：大额交易 + 可疑交易 + 税务"""

    def check_large_transaction(self, transaction):
        """大额交易检查（央行要求：单笔 ≥ 5 万元）"""
        threshold = 50000  # CNY
        if transaction["amount_base"] >= threshold:
            self.db.insert("large_transaction_reports", {
                "report_id": str(uuid4()),
                "transaction_id": transaction["id"],
                "amount": transaction["amount_base"],
                "currency": "CNY",
                "counterparty": transaction["counterparty"],
                "purpose": transaction.get("purpose", "未注明"),
                "report_status": "pending",
                "report_deadline": now() + timedelta(days=5),  # 5 个工作日内上报
                "created_at": now()
            })
            self.alert(f"大额交易: {transaction['amount_base']} CNY")

    def generate_sar(self, suspicious_activity):
        """生成可疑交易报告（SAR）"""
        sar_id = str(uuid4())
        # 汇总相关交易
        related_transactions = self.db.query(
            "SELECT * FROM transactions WHERE user_id = %s "
            "AND created_at BETWEEN %s AND %s",
            suspicious_activity["user_id"],
            suspicious_activity["start_date"],
            suspicious_activity["end_date"])

        self.db.insert("suspicious_activity_reports", {
            "sar_id": sar_id,
            "user_id": suspicious_activity["user_id"],
            "activity_type": suspicious_activity["type"],  # structuring / layering / round_trip
            "transaction_count": len(related_transactions),
            "total_amount": sum(t["amount_base"] for t in related_transactions),
            "description": suspicious_activity["description"],
            "report_status": "pending_approval",
            "created_at": now()
        })

        return sar_id

    def calculate_vat(self, transaction):
        """增值税计算"""
        VAT_RATE = 0.06  # 6% (金融服务)
        vat_amount = transaction["amount_base"] * VAT_RATE / (1 + VAT_RATE)
        return {
            "taxable_amount": round(transaction["amount_base"] - vat_amount, 2),
            "vat_amount": round(vat_amount, 2),
            "vat_rate": VAT_RATE
        }
```

## 银行对账自动化

```python
class BankReconciliationAutomation:
    """银行对账自动化：MT940 解析 + 自动匹配"""

    def import_statement(self, bank_id, file_content, format="mt940"):
        """导入银行对账单"""
        if format == "mt940":
            transactions = self._parse_mt940(file_content)
        elif format == "csv":
            transactions = self._parse_csv(file_content)

        matched = 0
        unmatched = 0
        for tx in transactions:
            result = self._auto_match(tx)
            if result["matched"]:
                self.db.update("transactions",
                    {"bank_confirmed": True, "bank_tx_id": tx["reference"]},
                    {"id": result["transaction_id"]})
                matched += 1
            else:
                self.db.insert("unmatched_bank_transactions", {
                    "bank_id": bank_id,
                    "amount": tx["amount"],
                    "date": tx["date"],
                    "reference": tx["reference"],
                    "description": tx.get("description", ""),
                    "status": "pending_review",
                    "imported_at": now()
                })
                unmatched += 1

        return {"total": len(transactions), "matched": matched, "unmatched": unmatched}

    def _auto_match(self, bank_tx):
        """自动匹配规则"""
        # 规则 1：精确匹配（金额 + 日期 + 参考号）
        exact = self.db.query_one(
            "SELECT * FROM transactions WHERE amount_base = %s "
            "AND DATE(created_at) = %s AND reference = %s",
            bank_tx["amount"], bank_tx["date"], bank_tx["reference"])
        if exact:
            return {"matched": True, "transaction_id": exact["id"]}

        # 规则 2：模糊匹配（金额 ±1% + 日期 ±1 天）
        fuzzy = self.db.query_one(
            "SELECT * FROM transactions WHERE "
            "ABS(amount_base - %s) / amount_base < 0.01 "
            "AND ABS(DATEDIFF(created_at, %s)) <= 1 "
            "ORDER BY ABS(amount_base - %s) ASC LIMIT 1",
            bank_tx["amount"], bank_tx["date"], bank_tx["amount"])
        if fuzzy:
            return {"matched": True, "transaction_id": fuzzy["id"], "fuzzy": True}

        return {"matched": False}

    def _parse_mt940(self, content):
        """解析 MT940 格式"""
        transactions = []
        for line in content.split("\n"):
            if line.startswith(":61:"):  # 交易行
                tx = self._parse_mt940_line(line)
                transactions.append(tx)
        return transactions
```

## 财务期间结转

```python
class PeriodCloseService:
    """财务期间结转"""

    def close_period(self, period):
        """结转期间"""
        # 1. 检查前置条件
        checks = self._run_close_checks(period)
        if not all(c["passed"] for c in checks):
            return {"status": "blocked", "failed_checks": checks}

        # 2. 生成自动调整分录
        adjustments = self._generate_closing_entries(period)

        # 3. 重新试算平衡
        trial = self._trial_balance(period)
        if not trial["balanced"]:
            return {"status": "imbalance", "difference": trial["difference"]}

        # 4. 锁定期间（防止回改）
        self.db.update("accounting_periods",
            {"status": "closed", "closed_at": now(), "closed_by": current_user()},
            {"period": period})

        # 5. 结转损益到留存收益
        self._close_income_expense(period)

        return {"status": "closed", "period": period, "adjustments": len(adjustments)}

    def _run_close_checks(self, period):
        """结转前检查"""
        checks = []

        # 所有分录已过账
        unposted = self.db.count("journal_entries",
            period=period, status="draft")
        checks.append({"name": "all_posted", "passed": unposted == 0,
                       "detail": f"{unposted} 条未过账分录"})

        # 试算平衡
        trial = self._trial_balance(period)
        checks.append({"name": "trial_balance", "passed": trial["balanced"],
                       "detail": f"差异 {trial['difference']}"})

        # 上期已结转
        prev_period = self._get_previous_period(period)
        prev = self.db.get_period(prev_period)
        checks.append({"name": "prev_closed", "passed": prev["status"] == "closed"})

        return checks

    def _generate_closing_entries(self, period):
        """生成结转分录"""
        adjustments = []

        # 折旧计提
        depreciation = self._calc_depreciation(period)
        if depreciation > 0:
            adjustments.append(self.ledger.post_entry(JournalEntry(
                lines=[
                    EntryLine("5501_depreciation_expense", debit=depreciation),
                    EntryLine("1601_accumulated_depreciation", credit=depreciation),
                ],
                entry_type="auto_depreciation",
                description=f"期间 {period} 折旧计提"
            )))

        # 应计利息
        accrued_interest = self._calc_accrued_interest(period)
        if accrued_interest > 0:
            adjustments.append(self.ledger.post_entry(JournalEntry(
                lines=[
                    EntryLine("5502_interest_expense", debit=accrued_interest),
                    EntryLine("2202_interest_payable", credit=accrued_interest),
                ],
                entry_type="auto_accrual",
                description=f"期间 {period} 利息计提"
            )))

        return adjustments
```

## 异常场景补充

### 场景：监管报表逾期

```
触发：大额交易报告未在 5 个工作日内上报 → 面临处罚
检测：
  1. 报表 deadline < 当前时间且 status=pending → 逾期
  2. 逾期 > 1 天 → 严重告警
处理：
  1. 立即上报（即使逾期也要补报）
  2. 向合规部门报备逾期原因
  3. 优化报表流程防止再次逾期
预防：报表 deadline 提前 2 天告警 + 自动生成 + 自动上报
```

### 场景：期间结转被阻塞

```
触发：上游系统（支付/交易）有未过账分录 → 无法结转
检测：
  1. 结转检查失败 → 阻塞
  2. 未过账分录来源 → 上游系统
处理：
  1. 通知上游系统完成过账
  2. 紧急情况：手动过账 + 结转
  3. 结转延迟 → 影响报表生成 → 通知管理层
预防：上游系统过账 SLA + 结转前自动检查 + 告警
```

## 监管报表模块完整实现（深度版）

```python
class RegulatoryReportServiceV2:
    """监管报表：央行大额/可疑 + 税务 + 自动归档 + 审计追踪 + 更正流程"""

    # ---- 1. 央行大额交易报告 ----

    LARGE_TX_THRESHOLDS = {
        "single_cash": 50000,        # 单笔现金收付 5 万元
        "single_transfer": 200000,   # 单笔转账 20 万元
        "daily_cash_cumulative": 50000,  # 当日现金累计 5 万元
    }

    def check_and_report_large_transaction(self, transaction):
        """大额交易检查与上报"""
        threshold = self._get_threshold(transaction)
        if transaction["amount_base"] < threshold:
            return {"reported": False, "reason": "未达阈值"}

        # 检查是否已报告
        existing = self.db.query_one(
            "SELECT * FROM large_transaction_reports "
            "WHERE transaction_id = %s", transaction["id"])
        if existing:
            return {"reported": False, "reason": "已上报", "report_id": existing["report_id"]}

        # 生成报告
        report_id = f"LTR-{now().strftime('%Y%m%d')}-{uuid4().hex[:8]}"
        report = {
            "report_id": report_id,
            "transaction_id": transaction["id"],
            "report_type": self._classify_transaction_type(transaction),
            "amount": transaction["amount_base"],
            "currency": "CNY",
            "transaction_date": transaction["created_at"],
            "payer_info": self._extract_party_info(transaction, "payer"),
            "payee_info": self._extract_party_info(transaction, "payee"),
            "purpose": transaction.get("purpose", "未注明"),
            "report_status": "pending_approval",
            "report_deadline": self._calc_report_deadline(),
            "created_at": now(),
            "created_by": current_user()
        }

        self.db.insert("large_transaction_reports", report)

        # 告警
        self.alert_service.send("compliance", "large_transaction", {
            "report_id": report_id,
            "amount": transaction["amount_base"],
            "deadline": report["report_deadline"]
        })

        return {"reported": True, "report_id": report_id}

    def _get_threshold(self, transaction):
        """获取对应阈值"""
        if transaction.get("is_cash"):
            return self.LARGE_TX_THRESHOLDS["single_cash"]
        return self.LARGE_TX_THRESHOLDS["single_transfer"]

    def _classify_transaction_type(self, transaction):
        """分类交易类型"""
        type_map = {
            "deposit": "现金存入",
            "withdrawal": "现金取出",
            "transfer_in": "转账收入",
            "transfer_out": "转账支出",
            "payment": "支付结算",
        }
        return type_map.get(transaction["type"], "其他")

    def _extract_party_info(self, transaction, role):
        """提取交易方信息（脱敏）"""
        party = transaction.get(f"{role}_info", {})
        return {
            "name": party.get("name", ""),
            "id_type": party.get("id_type", ""),
            "id_number": self._mask_id_number(party.get("id_number", "")),
            "account": self._mask_account(party.get("account", "")),
        }

    def _mask_id_number(self, id_number):
        """身份证脱敏：保留前3后4"""
        if len(id_number) >= 10:
            return id_number[:3] + "*" * (len(id_number) - 7) + id_number[-4:]
        return id_number

    def _mask_account(self, account):
        """账号脱敏：保留前4后4"""
        if len(account) >= 10:
            return account[:4] + "*" * (len(account) - 8) + account[-4:]
        return account

    def _calc_report_deadline(self):
        """计算上报截止日期（5个工作日）"""
        deadline = now()
        remaining_days = 5
        while remaining_days > 0:
            deadline += timedelta(days=1)
            if deadline.weekday() < 5:  # 工作日
                remaining_days -= 1
        return deadline.replace(hour=17, minute=0, second=0)

    # ---- 2. 可疑交易报告（SAR） ----

    SUSPICIOUS_PATTERNS = {
        "structuring": {       # 拆分交易规避报告
            "description": "将大额交易拆分为多笔小额交易",
            "threshold_count": 3,
            "threshold_amount": 48000,  # 每笔略低于 5 万
            "time_window_hours": 24,
        },
        "layering": {          # 层层转账掩盖来源
            "description": "资金在多个账户间快速转移",
            "threshold_hops": 3,
            "time_window_hours": 48,
        },
        "round_trip": {        # 往返交易
            "description": "资金在两个账户间往返",
            "threshold_count": 5,
            "time_window_hours": 72,
        },
    }

    def detect_suspicious_activity(self, user_id):
        """检测可疑交易行为"""
        findings = []

        for pattern_name, pattern_def in self.SUSPICIOUS_PATTERNS.items():
            if pattern_name == "structuring":
                result = self._detect_structuring(user_id, pattern_def)
            elif pattern_name == "layering":
                result = self._detect_layering(user_id, pattern_def)
            elif pattern_name == "round_trip":
                result = self._detect_round_trip(user_id, pattern_def)
            else:
                continue

            if result["suspicious"]:
                findings.append({
                    "pattern": pattern_name,
                    "description": pattern_def["description"],
                    "evidence": result["evidence"]
                })

        if findings:
            # 自动生成 SAR
            sar_id = self.generate_sar_report(user_id, findings)
            return {"suspicious": True, "findings": findings, "sar_id": sar_id}

        return {"suspicious": False}

    def _detect_structuring(self, user_id, pattern_def):
        """检测拆分交易"""
        recent_tx = self.db.query(
            "SELECT * FROM transactions WHERE user_id = %s "
            "AND created_at >= %s ORDER BY created_at",
            user_id, now() - timedelta(hours=pattern_def["time_window_hours"]))

        # 检查金额接近阈值但低于阈值的交易
        near_threshold = [
            t for t in recent_tx
            if pattern_def["threshold_amount"] * 0.9 <= t["amount_base"] < 50000
        ]

        if len(near_threshold) >= pattern_def["threshold_count"]:
            return {
                "suspicious": True,
                "evidence": {
                    "count": len(near_threshold),
                    "total_amount": sum(t["amount_base"] for t in near_threshold),
                    "transaction_ids": [t["id"] for t in near_threshold]
                }
            }

        return {"suspicious": False}

    def _detect_layering(self, user_id, pattern_def):
        """检测层层转账"""
        # 追踪资金流向深度
        hops = self._trace_fund_flow(user_id, max_hops=pattern_def["threshold_hops"] + 1)
        if len(hops) >= pattern_def["threshold_hops"]:
            return {
                "suspicious": True,
                "evidence": {"hops": len(hops), "flow": hops}
            }
        return {"suspicious": False}

    def _detect_round_trip(self, user_id, pattern_def):
        """检测往返交易"""
        recent_tx = self.db.query(
            "SELECT * FROM transactions WHERE (payer_id = %s OR payee_id = %s) "
            "AND created_at >= %s ORDER BY created_at",
            user_id, user_id,
            now() - timedelta(hours=pattern_def["time_window_hours"]))

        # 检查与同一对手方的往返交易
        counterparty_counts = {}
        for tx in recent_tx:
            cp = tx["payee_id"] if tx["payer_id"] == user_id else tx["payer_id"]
            counterparty_counts[cp] = counterparty_counts.get(cp, 0) + 1

        suspicious_counterparties = {
            cp: count for cp, count in counterparty_counts.items()
            if count >= pattern_def["threshold_count"]
        }

        if suspicious_counterparties:
            return {
                "suspicious": True,
                "evidence": {"counterparties": suspicious_counterparties}
            }
        return {"suspicious": False}

    def _trace_fund_flow(self, user_id, max_hops=5):
        """追踪资金流向"""
        hops = []
        current_id = user_id
        visited = set()

        for _ in range(max_hops):
            if current_id in visited:
                break
            visited.add(current_id)

            next_tx = self.db.query_one(
                "SELECT * FROM transactions WHERE payer_id = %s "
                "AND created_at >= %s ORDER BY created_at DESC LIMIT 1",
                current_id, now() - timedelta(days=7))

            if not next_tx:
                break

            hops.append({
                "from": current_id,
                "to": next_tx["payee_id"],
                "amount": next_tx["amount_base"],
                "date": next_tx["created_at"]
            })
            current_id = next_tx["payee_id"]

        return hops

    def generate_sar_report(self, user_id, findings):
        """生成可疑交易报告"""
        sar_id = f"SAR-{now().strftime('%Y%m%d')}-{uuid4().hex[:8]}"

        # 汇总相关交易
        related_transactions = self.db.query(
            "SELECT * FROM transactions WHERE user_id = %s "
            "AND created_at >= %s",
            user_id, now() - timedelta(days=30))

        self.db.insert("suspicious_activity_reports", {
            "sar_id": sar_id,
            "user_id": user_id,
            "findings": findings,
            "transaction_count": len(related_transactions),
            "total_amount": sum(t["amount_base"] for t in related_transactions),
            "report_status": "pending_approval",
            "created_at": now(),
            "created_by": current_user()
        })

        # 告警合规部门
        self.alert_service.send("compliance", "suspicious_activity", {
            "sar_id": sar_id,
            "user_id": user_id,
            "findings_count": len(findings)
        })

        return sar_id

    # ---- 3. 税务计算引擎 ----

    def calculate_period_tax(self, period):
        """期间税务计算"""
        # 增值税
        vat_result = self._calculate_vat_withholding(period)

        # 预提所得税
        income_tax_result = self._calculate_income_tax(period)

        # 生成税务分录
        tax_entries = []
        if vat_result["vat_payable"] > 0:
            tax_entries.append(self.ledger.post_entry(JournalEntry(
                lines=[
                    EntryLine("2221_vat_payable", debit=vat_result["vat_payable"]),
                    EntryLine("222101_vat_output", credit=vat_result["vat_output"]),
                    EntryLine("222102_vat_input", debit=vat_result["vat_input"]),
                ],
                entry_type="vat_settlement",
                description=f"期间 {period} 增值税结算"
            )))

        return {
            "period": period,
            "vat": vat_result,
            "income_tax": income_tax_result,
            "entries_created": len(tax_entries)
        }

    def _calculate_vat_withholding(self, period):
        """增值税计算"""
        VAT_RATE = 0.06  # 金融服务 6%

        # 销项税
        revenue = self.db.query_one(
            "SELECT COALESCE(SUM(credit_amount_base), 0) as total "
            "FROM journal_entry_lines WHERE account_code LIKE '6%%' "
            "AND entry_date BETWEEN %s AND %s",
            *self._get_period_range(period))
        vat_output = revenue["total"] * VAT_RATE / (1 + VAT_RATE)

        # 进项税
        expenses = self.db.query_one(
            "SELECT COALESCE(SUM(debit_amount_base), 0) as total "
            "FROM journal_entry_lines WHERE account_code LIKE '5%%' "
            "AND entry_date BETWEEN %s AND %s",
            *self._get_period_range(period))
        vat_input = expenses["total"] * VAT_RATE / (1 + VAT_RATE)

        vat_payable = max(0, vat_output - vat_input)

        return {
            "vat_output": round(vat_output, 2),
            "vat_input": round(vat_input, 2),
            "vat_payable": round(vat_payable, 2),
            "vat_rate": VAT_RATE
        }

    def _calculate_income_tax(self, period):
        """预提所得税计算"""
        TAX_RATE = 0.25  # 企业所得税 25%

        # 计算应纳税所得额
        net_income = self.db.query_one(
            "SELECT "
            "COALESCE(SUM(CASE WHEN account_code LIKE '6%%' THEN credit_amount_base ELSE 0 END), 0) - "
            "COALESCE(SUM(CASE WHEN account_code LIKE '5%%' THEN debit_amount_base ELSE 0 END), 0) "
            "as taxable_income "
            "FROM journal_entry_lines WHERE entry_date BETWEEN %s AND %s",
            *self._get_period_range(period))

        taxable_income = max(0, net_income["taxable_income"])
        tax_amount = taxable_income * TAX_RATE

        return {
            "taxable_income": round(taxable_income, 2),
            "tax_rate": TAX_RATE,
            "tax_amount": round(tax_amount, 2)
        }

    def _get_period_range(self, period):
        """获取期间日期范围"""
        period_info = self.db.get_period(period)
        return (period_info["start_date"], period_info["end_date"])

    # ---- 4. 自动归档排期 ----

    REPORT_SCHEDULE = {
        "large_transaction": {"frequency": "daily", "deadline_hours": 120},   # 5 个工作日
        "suspicious_activity": {"frequency": "on_detect", "deadline_hours": 240},  # 10 个工作日
        "vat_return": {"frequency": "monthly", "deadline_day": 15},
        "income_tax_quarterly": {"frequency": "quarterly", "deadline_day": 15},
    }

    def get_pending_reports(self):
        """获取待上报报表"""
        pending = self.db.query(
            "SELECT * FROM large_transaction_reports WHERE report_status = 'pending_approval' "
            "UNION ALL "
            "SELECT * FROM suspicious_activity_reports WHERE report_status = 'pending_approval' "
            "ORDER BY report_deadline ASC")

        return [{
            "report_id": r["report_id"],
            "type": r.get("report_type", r.get("activity_type", "unknown")),
            "status": r["report_status"],
            "deadline": r["report_deadline"],
            "overdue": r["report_deadline"] < now()
        } for r in pending]

    # ---- 5. 审计追踪 ----

    def get_report_audit_trail(self, report_id):
        """获取报表审计追踪"""
        trail = self.db.query(
            "SELECT * FROM report_audit_log WHERE report_id = %s "
            "ORDER BY created_at", report_id)

        return [{
            "action": t["action"],
            "operator": t["operator"],
            "timestamp": t["created_at"],
            "detail": t.get("detail", "")
        } for t in trail]

    def log_report_action(self, report_id, action, detail=""):
        """记录报表操作日志"""
        self.db.insert("report_audit_log", {
            "report_id": report_id,
            "action": action,   # created / approved / submitted / rejected / amended
            "operator": current_user(),
            "detail": detail,
            "created_at": now()
        })

    # ---- 6. 更正流程 ----

    def amend_report(self, report_id, reason, amendments):
        """更正已提交的报表"""
        original = self.db.get_report(report_id)
        if not original:
            return {"status": "error", "reason": "报表不存在"}

        if original["report_status"] not in ["submitted", "accepted"]:
            return {"status": "error", "reason": "当前状态不支持更正"}

        # 创建更正记录
        amendment_id = f"AMD-{now().strftime('%Y%m%d')}-{uuid4().hex[:6]}"
        self.db.insert("report_amendments", {
            "amendment_id": amendment_id,
            "original_report_id": report_id,
            "reason": reason,
            "amendments": amendments,
            "status": "pending_approval",
            "created_at": now(),
            "created_by": current_user()
        })

        # 审计日志
        self.log_report_action(report_id, "amended", f"更正原因: {reason}")

        return {"status": "amendment_created", "amendment_id": amendment_id}
```

## 银行对账自动化完整实现（深度版）

```python
class BankReconciliationServiceV2:
    """银行对账自动化：多格式导入 + 多级匹配 + 未匹配升级 + 对账仪表盘"""

    # ---- 1. 银行对账单导入（多格式） ----

    def import_statement(self, bank_id, file_content, format="mt940"):
        """导入银行对账单（支持 CSV / SWIFT / MT940）"""
        if format == "mt940":
            transactions = self._parse_mt940(file_content)
        elif format == "csv":
            transactions = self._parse_csv(file_content)
        elif format == "swift":
            transactions = self._parse_swift(file_content)
        else:
            return {"status": "error", "reason": f"不支持格式: {format}"}

        # 导入批次记录
        batch_id = f"BATCH-{now().strftime('%Y%m%d%H%M%S')}"
        self.db.insert("reconciliation_batches", {
            "batch_id": batch_id,
            "bank_id": bank_id,
            "format": format,
            "total_transactions": len(transactions),
            "status": "imported",
            "imported_at": now()
        })

        # 逐笔匹配
        matched = 0
        fuzzy_matched = 0
        unmatched = 0

        for tx in transactions:
            tx["batch_id"] = batch_id
            result = self._auto_match(tx)

            if result["matched"]:
                if result.get("fuzzy"):
                    fuzzy_matched += 1
                else:
                    matched += 1
                self.db.update("internal_transactions",
                    {"bank_confirmed": True, "bank_tx_id": tx["reference"],
                     "match_type": result.get("match_type", "exact")},
                    {"id": result["transaction_id"]})
            else:
                unmatched += 1
                self.db.insert("unmatched_bank_transactions", {
                    "batch_id": batch_id,
                    "bank_id": bank_id,
                    "amount": tx["amount"],
                    "date": tx["date"],
                    "reference": tx["reference"],
                    "description": tx.get("description", ""),
                    "status": "pending_review",
                    "imported_at": now()
                })

        return {
            "batch_id": batch_id,
            "total": len(transactions),
            "matched": matched,
            "fuzzy_matched": fuzzy_matched,
            "unmatched": unmatched,
            "match_rate": round((matched + fuzzy_matched) / max(len(transactions), 1), 3)
        }

    def _parse_csv(self, content):
        """解析 CSV 格式对账单"""
        import csv
        import io

        transactions = []
        reader = csv.DictReader(io.StringIO(content))
        for row in reader:
            transactions.append({
                "date": self._parse_date(row.get("日期", row.get("date", ""))),
                "amount": Decimal(row.get("金额", row.get("amount", "0")).replace(",", "")),
                "reference": row.get("参考号", row.get("reference", "")),
                "description": row.get("摘要", row.get("description", "")),
                "counterparty": row.get("对方账号", row.get("counterparty", "")),
            })
        return transactions

    def _parse_swift(self, content):
        """解析 SWIFT MT940 格式"""
        transactions = []
        current_tx = {}

        for line in content.split("\n"):
            line = line.strip()
            if line.startswith(":61:"):  # 交易行
                if current_tx:
                    transactions.append(current_tx)
                current_tx = self._parse_mt940_61_line(line)
            elif line.startswith(":86:"):  # 补充信息
                if current_tx:
                    current_tx["description"] = line[4:].strip()

        if current_tx:
            transactions.append(current_tx)

        return transactions

    def _parse_mt940_61_line(self, line):
        """解析 MT940 :61: 行"""
        # :61: 日期(6) 金额(15) 类型(1) 参考号(16)
        content = line[4:].strip()
        date_str = content[:6]  # YYMMDD
        # 简化解析
        return {
            "date": self._parse_yymmdd(date_str),
            "amount": Decimal("0"),  # 实际需完整解析
            "reference": content[22:38].strip() if len(content) > 22 else "",
            "description": ""
        }

    # ---- 2. 自动匹配规则（多级） ----

    def _auto_match(self, bank_tx):
        """多级自动匹配"""
        # 第 1 级：精确匹配（金额 + 日期 + 参考号）
        result = self._match_exact(bank_tx)
        if result["matched"]:
            return result

        # 第 2 级：参考号匹配（允许金额/日期微小差异）
        result = self._match_reference(bank_tx)
        if result["matched"]:
            return result

        # 第 3 级：模糊匹配（金额 +-1% + 日期 +-1 天）
        result = self._match_fuzzy(bank_tx)
        if result["matched"]:
            return result

        return {"matched": False}

    def _match_exact(self, bank_tx):
        """精确匹配：金额 + 日期 + 参考号完全一致"""
        match = self.db.query_one(
            "SELECT * FROM internal_transactions "
            "WHERE amount_base = %s "
            "AND DATE(created_at) = %s "
            "AND reference = %s "
            "AND bank_confirmed = FALSE "
            "LIMIT 1",
            bank_tx["amount"], bank_tx["date"], bank_tx["reference"])

        if match:
            return {"matched": True, "transaction_id": match["id"], "match_type": "exact"}
        return {"matched": False}

    def _match_reference(self, bank_tx):
        """参考号匹配：参考号一致，金额/日期允许微小差异"""
        if not bank_tx.get("reference"):
            return {"matched": False}

        match = self.db.query_one(
            "SELECT * FROM internal_transactions "
            "WHERE reference = %s "
            "AND ABS(amount_base - %s) < %s "  # 金额差异 < 50 元
            "AND ABS(DATEDIFF(created_at, %s)) <= 2 "
            "AND bank_confirmed = FALSE "
            "LIMIT 1",
            bank_tx["reference"], bank_tx["amount"], 50, bank_tx["date"])

        if match:
            return {"matched": True, "transaction_id": match["id"],
                    "match_type": "reference", "amount_diff": match["amount_base"] - bank_tx["amount"]}
        return {"matched": False}

    def _match_fuzzy(self, bank_tx):
        """模糊匹配：金额 +-1% + 日期 +-1 天"""
        tolerance = bank_tx["amount"] * Decimal("0.01")  # 1%

        match = self.db.query_one(
            "SELECT * FROM internal_transactions "
            "WHERE ABS(amount_base - %s) <= %s "
            "AND ABS(DATEDIFF(created_at, %s)) <= 1 "
            "AND bank_confirmed = FALSE "
            "ORDER BY ABS(amount_base - %s) ASC "
            "LIMIT 1",
            bank_tx["amount"], tolerance, bank_tx["date"], bank_tx["amount"])

        if match:
            return {"matched": True, "transaction_id": match["id"],
                    "match_type": "fuzzy", "amount_diff": match["amount_base"] - bank_tx["amount"]}
        return {"matched": False}

    # ---- 3. 未匹配项升级 ----

    def escalate_unmatched(self, batch_id, max_age_hours=24):
        """升级处理未匹配项"""
        cutoff = now() - timedelta(hours=max_age_hours)

        unmatched = self.db.query(
            "SELECT * FROM unmatched_bank_transactions "
            "WHERE batch_id = %s AND status = 'pending_review' "
            "AND imported_at < %s",
            batch_id, cutoff)

        escalated = []
        for item in unmatched:
            # 自动升级为人工审核
            self.db.update("unmatched_bank_transactions", item["id"], {
                "status": "escalated",
                "escalated_at": now()
            })

            # 创建人工审核任务
            task_id = self.task_service.create_task({
                "type": "reconciliation_review",
                "priority": "high" if item["amount"] > 100000 else "normal",
                "detail": item,
                "assigned_to": self._get_available_accountant()
            })

            escalated.append({
                "id": item["id"],
                "amount": item["amount"],
                "date": item["date"],
                "task_id": task_id
            })

        return {"escalated_count": len(escalated), "items": escalated}

    def _get_available_accountant(self):
        """获取可用的会计人员"""
        accountants = self.db.query(
            "SELECT user_id FROM users WHERE role = 'accountant' AND status = 'active'")
        # 简单轮询分配
        current_index = int(self.redis.incr("accountant_assign_index"))
        return accountants[current_index % len(accountants)]["user_id"] if accountants else None

    # ---- 4. 对账状态仪表盘 ----

    def get_reconciliation_dashboard(self, period):
        """对账状态仪表盘"""
        # 总体匹配率
        total_internal = self.db.query_one(
            "SELECT COUNT(*) as total FROM internal_transactions "
            "WHERE DATE(created_at) BETWEEN %s AND %s",
            *self._get_period_dates(period))["total"]

        confirmed = self.db.query_one(
            "SELECT COUNT(*) as total FROM internal_transactions "
            "WHERE bank_confirmed = TRUE "
            "AND DATE(created_at) BETWEEN %s AND %s",
            *self._get_period_dates(period))["total"]

        # 未匹配统计
        unmatched = self.db.query(
            "SELECT status, COUNT(*) as count, SUM(amount) as total_amount "
            "FROM unmatched_bank_transactions "
            "WHERE DATE(imported_at) BETWEEN %s AND %s "
            "GROUP BY status",
            *self._get_period_dates(period))

        # 匹配方式分布
        match_types = self.db.query(
            "SELECT match_type, COUNT(*) as count "
            "FROM internal_transactions "
            "WHERE bank_confirmed = TRUE "
            "AND DATE(created_at) BETWEEN %s AND %s "
            "GROUP BY match_type",
            *self._get_period_dates(period))

        return {
            "period": period,
            "total_internal_transactions": total_internal,
            "confirmed": confirmed,
            "unconfirmed": total_internal - confirmed,
            "match_rate": round(confirmed / max(total_internal, 1), 3),
            "unmatched_summary": unmatched,
            "match_type_distribution": match_types,
        }

    def _get_period_dates(self, period):
        """获取期间起止日期"""
        info = self.db.get_period(period)
        return (info["start_date"], info["end_date"])
```

## 财务期间结转完整实现（深度版）

```python
class FinancialPeriodCloseServiceV2:
    """财务期间结转：检查清单 -> 自动分录 -> 期间锁定 -> 审批流程"""

    # ---- 1. 期间结转检查清单 ----

    CLOSE_CHECKLIST = [
        {"id": "all_posted", "name": "所有分录已过账", "severity": "blocker"},
        {"id": "trial_balance", "name": "试算平衡表平衡", "severity": "blocker"},
        {"id": "bank_reconciled", "name": "银行对账已完成", "severity": "blocker"},
        {"id": "depreciation_posted", "name": "折旧已计提", "severity": "blocker"},
        {"id": "accruals_posted", "name": "应计项目已入账", "severity": "blocker"},
        {"id": "prev_period_closed", "name": "上期已结转", "severity": "blocker"},
        {"id": "tax_calculated", "name": "税务已计算", "severity": "warning"},
        {"id": "regulatory_filed", "name": "监管报表已提交", "severity": "warning"},
    ]

    def run_close_checklist(self, period):
        """执行期间结转检查清单"""
        results = []

        for check in self.CLOSE_CHECKLIST:
            result = self._execute_check(check["id"], period)
            results.append({
                "id": check["id"],
                "name": check["name"],
                "severity": check["severity"],
                "passed": result["passed"],
                "detail": result.get("detail", "")
            })

        blockers = [r for r in results if not r["passed"] and r["severity"] == "blocker"]
        warnings = [r for r in results if not r["passed"] and r["severity"] == "warning"]

        return {
            "period": period,
            "all_passed": len(blockers) == 0 and len(warnings) == 0,
            "can_close": len(blockers) == 0,
            "blockers": blockers,
            "warnings": warnings,
            "results": results
        }

    def _execute_check(self, check_id, period):
        """执行单个检查项"""
        if check_id == "all_posted":
            unposted = self.db.count("journal_entries",
                period=period, status="draft")
            return {"passed": unposted == 0, "detail": f"{unposted} 条未过账分录"}

        elif check_id == "trial_balance":
            trial = self._trial_balance(period)
            return {"passed": trial["balanced"],
                    "detail": f"差异 {trial['difference']}"}

        elif check_id == "bank_reconciled":
            unreconciled = self.db.count("internal_transactions",
                period=period, bank_confirmed=False)
            return {"passed": unreconciled == 0,
                    "detail": f"{unreconciled} 笔未对账交易"}

        elif check_id == "depreciation_posted":
            dep_entries = self.db.count("journal_entries",
                period=period, entry_type="auto_depreciation")
            return {"passed": dep_entries > 0,
                    "detail": "已计提" if dep_entries > 0 else "未计提"}

        elif check_id == "accruals_posted":
            accrual_entries = self.db.count("journal_entries",
                period=period, entry_type="auto_accrual")
            return {"passed": accrual_entries > 0,
                    "detail": "已入账" if accrual_entries > 0 else "未入账"}

        elif check_id == "prev_period_closed":
            prev = self._get_previous_period(period)
            if not prev:
                return {"passed": True, "detail": "首期无需检查"}
            prev_status = self.db.get_period(prev)
            return {"passed": prev_status["status"] == "closed",
                    "detail": f"上期状态: {prev_status['status']}"}

        elif check_id == "tax_calculated":
            tax_entries = self.db.count("journal_entries",
                period=period, entry_type="vat_settlement")
            return {"passed": tax_entries > 0,
                    "detail": "已计算" if tax_entries > 0 else "未计算"}

        elif check_id == "regulatory_filed":
            pending = self.db.count("large_transaction_reports",
                period=period, report_status="pending_approval")
            return {"passed": pending == 0,
                    "detail": f"{pending} 份待审批报表"}

        return {"passed": True, "detail": "未知检查项"}

    def _trial_balance(self, period):
        """试算平衡"""
        period_info = self.db.get_period(period)
        result = self.db.query_one(
            "SELECT "
            "COALESCE(SUM(debit_amount_base), 0) as total_debit, "
            "COALESCE(SUM(credit_amount_base), 0) as total_credit "
            "FROM journal_entry_lines "
            "WHERE entry_date BETWEEN %s AND %s "
            "AND entry_id IN (SELECT id FROM journal_entries WHERE status = 'posted')",
            period_info["start_date"], period_info["end_date"])

        difference = abs(result["total_debit"] - result["total_credit"])
        return {
            "total_debit": result["total_debit"],
            "total_credit": result["total_credit"],
            "balanced": difference < Decimal("0.01"),
            "difference": difference
        }

    # ---- 2. 自动结转分录 ----

    def generate_auto_closing_entries(self, period):
        """生成自动结转分录（折旧 + 应计 + 损益结转）"""
        entries = []

        # 折旧计提
        depreciation = self._calc_depreciation(period)
        if depreciation > 0:
            entry = self.ledger.post_entry(JournalEntry(
                lines=[
                    EntryLine("5501_depreciation_expense", debit=depreciation),
                    EntryLine("1601_accumulated_depreciation", credit=depreciation),
                ],
                entry_type="auto_depreciation",
                description=f"期间 {period} 折旧计提"
            ))
            entries.append({"type": "depreciation", "amount": depreciation, "entry_id": entry})

        # 应计利息
        accrued_interest = self._calc_accrued_interest(period)
        if accrued_interest > 0:
            entry = self.ledger.post_entry(JournalEntry(
                lines=[
                    EntryLine("5502_interest_expense", debit=accrued_interest),
                    EntryLine("2202_interest_payable", credit=accrued_interest),
                ],
                entry_type="auto_accrual",
                description=f"期间 {period} 利息计提"
            ))
            entries.append({"type": "accrued_interest", "amount": accrued_interest, "entry_id": entry})

        # 工资应计
        accrued_salary = self._calc_accrued_salary(period)
        if accrued_salary > 0:
            entry = self.ledger.post_entry(JournalEntry(
                lines=[
                    EntryLine("5503_salary_expense", debit=accrued_salary),
                    EntryLine("2203_salary_payable", credit=accrued_salary),
                ],
                entry_type="auto_accrual",
                description=f"期间 {period} 工资计提"
            ))
            entries.append({"type": "accrued_salary", "amount": accrued_salary, "entry_id": entry})

        return entries

    def _calc_depreciation(self, period):
        """计算折旧"""
        assets = self.db.query(
            "SELECT * FROM fixed_assets WHERE status = 'active'")
        total = sum(
            a["cost"] * (1 - a.get("salvage_rate", 0.05)) / (a["useful_life_years"] * 12)
            for a in assets)
        return round(total, 2)

    def _calc_accrued_interest(self, period):
        """计算应计利息"""
        loans = self.db.query(
            "SELECT * FROM loans WHERE status = 'active'")
        total = sum(
            l["principal"] * l["interest_rate"] / 12
            for l in loans)
        return round(total, 2)

    def _calc_accrued_salary(self, period):
        """计算应计工资"""
        period_info = self.db.get_period(period)
        days_in_month = (period_info["end_date"] - period_info["start_date"]).days + 1
        # 简化：按月工资总额计提
        monthly_salary_budget = self.db.query_one(
            "SELECT value FROM system_config WHERE key = 'monthly_salary_budget'")["value"]
        return round(Decimal(str(monthly_salary_budget)), 2)

    # ---- 3. 期间锁定 ----

    def lock_period(self, period):
        """锁定期间（防止回改）"""
        # 检查是否可以锁定
        checklist = self.run_close_checklist(period)
        if not checklist["can_close"]:
            return {"status": "blocked", "blockers": checklist["blockers"]}

        # 锁定期间
        self.db.update("accounting_periods",
            {"status": "locked", "locked_at": now(), "locked_by": current_user()},
            {"period": period})

        # 防止在锁定期间内新增/修改分录
        self.redis.set(f"period_locked:{period}", "1")

        return {"status": "locked", "period": period}

    def is_period_locked(self, period):
        """检查期间是否已锁定"""
        return self.redis.exists(f"period_locked:{period}")

    def prevent_backdated_entry(self, entry_date):
        """防止回溯分录"""
        period = self._get_period_for_date(entry_date)
        if self.is_period_locked(period):
            raise PermissionError(f"期间 {period} 已锁定，不能新增/修改分录")

    def _get_period_for_date(self, date):
        """根据日期获取所属期间"""
        return self.db.query_one(
            "SELECT period FROM accounting_periods "
            "WHERE start_date <= %s AND end_date >= %s",
            date, date)["period"]

    # ---- 4. 结转审批流程 ----

    def submit_close_request(self, period):
        """提交结转申请"""
        # 执行检查清单
        checklist = self.run_close_checklist(period)

        if not checklist["can_close"]:
            return {"status": "blocked", "blockers": checklist["blockers"]}

        # 生成自动分录
        auto_entries = self.generate_auto_closing_entries(period)

        # 重新试算平衡
        trial = self._trial_balance(period)
        if not trial["balanced"]:
            return {"status": "imbalance", "difference": str(trial["difference"])}

        # 创建结转申请
        close_id = f"CLOSE-{period}-{uuid4().hex[:6]}"
        self.db.insert("period_close_requests", {
            "close_id": close_id,
            "period": period,
            "status": "pending_approval",
            "checklist_results": checklist["results"],
            "auto_entries": len(auto_entries),
            "trial_balance": {
                "debit": str(trial["total_debit"]),
                "credit": str(trial["total_credit"])
            },
            "requested_at": now(),
            "requested_by": current_user()
        })

        # 发送审批通知
        self.notification_service.send("finance_director", "period_close_approval", {
            "close_id": close_id,
            "period": period
        })

        return {"status": "pending_approval", "close_id": close_id}

    def approve_close(self, close_id, approver, comments=""):
        """审批结转"""
        close_req = self.db.get_close_request(close_id)

        if close_req["status"] != "pending_approval":
            return {"status": "error", "reason": "当前状态不支持审批"}

        # 执行结转
        period = close_req["period"]

        # 锁定期间
        self.lock_period(period)

        # 结转损益
        self._close_income_expense(period)

        # 更新状态
        self.db.update("period_close_requests", close_id, {
            "status": "approved",
            "approved_by": approver,
            "approved_at": now(),
            "comments": comments
        })

        # 记录审计日志
        self.audit_service.log("period_closed", {
            "period": period,
            "close_id": close_id,
            "approved_by": approver
        })

        return {"status": "closed", "period": period}

    def reject_close(self, close_id, approver, reason):
        """驳回结转"""
        self.db.update("period_close_requests", close_id, {
            "status": "rejected",
            "rejected_by": approver,
            "rejected_at": now(),
            "rejection_reason": reason
        })

        return {"status": "rejected"}

    def _close_income_expense(self, period):
        """结转损益到留存收益"""
        # 收入类科目余额（6 开头）
        revenue = self.db.query_one(
            "SELECT COALESCE(SUM(credit_amount_base - debit_amount_base), 0) as balance "
            "FROM journal_entry_lines WHERE account_code LIKE '6%%' "
            "AND entry_date BETWEEN %s AND %s",
            *self._get_period_dates(period))

        # 费用类科目余额（5 开头）
        expenses = self.db.query_one(
            "SELECT COALESCE(SUM(debit_amount_base - credit_amount_base), 0) as balance "
            "FROM journal_entry_lines WHERE account_code LIKE '5%%' "
            "AND entry_date BETWEEN %s AND %s",
            *self._get_period_dates(period))

        net_income = revenue["balance"] - expenses["balance"]

        # 结转分录
        if net_income != 0:
            self.ledger.post_entry(JournalEntry(
                lines=[
                    EntryLine("6101_revenue_summary", debit=abs(revenue["balance"])),
                    EntryLine("5101_expense_summary", credit=abs(expenses["balance"])),
                    EntryLine("4101_retained_earnings",
                              credit=net_income if net_income > 0 else 0,
                              debit=abs(net_income) if net_income < 0 else 0),
                ],
                entry_type="income_summary_close",
                description=f"期间 {period} 损益结转"
            ))

    def _get_period_dates(self, period):
        """获取期间起止日期"""
        info = self.db.get_period(period)
        return (info["start_date"], info["end_date"])
```

## 异常场景补充

### 场景：监管报表提交截止日期错过

```
触发：大额交易报告/可疑交易报告未在规定时限内上报 -> 面临监管处罚
检测：
  1. 报表 deadline < 当前时间 且 status 仍为 pending -> 逾期
  2. 逾期 < 1 天 -> 一般告警
  3. 逾期 >= 1 天 -> 严重告警，通知合规负责人
处理：
  1. 立即补报（即使逾期也要完成上报）
  2. 向合规部门书面报备逾期原因和补救措施
  3. 分析逾期原因：系统故障/人员疏忽/数据延迟
  4. 针对性修复：增加自动上报机制、设置多级提醒
预防：deadline 前 48 小时自动提醒 + 24 小时升级告警 + 自动生成并提交
```

### 场景：期间结转被未过账分录阻塞

```
触发：上游系统（支付/交易/结算）有未过账分录 -> 结转检查清单失败 -> 无法结转
检测：
  1. 结转检查清单 "all_posted" 项失败 -> 存在未过账分录
  2. 查询未过账分录来源系统 -> 定位阻塞原因
处理：
  1. 通知上游系统尽快完成过账（附未过账分录清单）
  2. 上游系统过账 SLA：收到通知后 2 小时内完成
  3. 超过 SLA 仍无法过账 -> 启动应急流程：
     a. 手动审核未过账分录
     b. 确认无风险后手动过账
     c. 继续结转流程
  4. 结转延迟导致报表延迟 -> 通知管理层并说明原因
预防：上游系统过账 SLA + 结转前 4 小时自动检查 + 未过账分录告警

## 会计科目与借贷平衡完整实现

```python
class AccountingEntryService:
    """会计分录：借贷平衡 + 科目校验 + 自动结转"""

    ACCOUNT_TYPES = {
        "asset": "资产类（借方增加）",
        "liability": "负债类（贷方增加）",
        "equity": "所有者权益类（贷方增加）",
        "revenue": "收入类（贷方增加）",
        "expense": "费用类（借方增加）",
    }

    def create_entry(self, entry_lines, description, created_by):
        """创建会计分录（必须借贷平衡）"""
        # 1. 校验借贷平衡
        total_debit = sum(line["amount"] for line in entry_lines if line["direction"] == "debit")
        total_credit = sum(line["amount"] for line in entry_lines if line["direction"] == "credit")

        if total_debit != total_credit:
            raise UnbalancedEntryError(
                f"借贷不平衡: 借方 {total_debit}, 贷方 {total_credit}, 差额 {abs(total_debit - total_credit)}")

        # 2. 校验科目有效性
        for line in entry_lines:
            account = self.db.get_account(line["account_code"])
            if not account:
                raise InvalidAccountError(f"无效科目: {line['account_code']}")
            if account["status"] != "active":
                raise InactiveAccountError(f"科目已停用: {line['account_code']}")

        # 3. 创建分录
        entry_id = str(uuid4())
        self.db.insert("accounting_entries", {
            "entry_id": entry_id,
            "description": description,
            "total_debit": total_debit,
            "total_credit": total_credit,
            "line_count": len(entry_lines),
            "status": "posted",
            "created_by": created_by,
            "created_at": now()
        })

        # 4. 写入分录明细
        for line in entry_lines:
            self.db.insert("accounting_entry_lines", {
                "entry_id": entry_id,
                "account_code": line["account_code"],
                "direction": line["direction"],
                "amount": line["amount"],
                "currency": line.get("currency", "CNY"),
                "counterpart_account": line.get("counterpart_account"),
                "created_at": now()
            })

            # 5. 更新科目余额
            self._update_account_balance(line["account_code"],
                line["direction"], line["amount"])

        return {"entry_id": entry_id, "total_debit": total_debit,
                "total_credit": total_credit}

    def _update_account_balance(self, account_code, direction, amount):
        """更新科目余额"""
        account = self.db.get_account(account_code)
        account_type = account["account_type"]

        # 判断余额方向
        # 资产、费用：借方增加，贷方减少
        # 负债、权益、收入：贷方增加，借方减少
        if account_type in ["asset", "expense"]:
            delta = amount if direction == "debit" else -amount
        else:
            delta = amount if direction == "credit" else -amount

        # 更新当前期间余额
        current_period = now().strftime("%Y%m")
        self.db.execute(
            "INSERT INTO account_balances (account_code, period, balance) "
            "VALUES (%s, %s, %s) "
            "ON CONFLICT (account_code, period) DO UPDATE "
            "SET balance = account_balances.balance + %s",
            account_code, current_period, delta, delta)

    def period_close(self, period):
        """期末结转（收入/费用 → 本年利润）"""
        # 1. 结转收入
        revenue_accounts = self.db.query(
            "SELECT account_code, balance FROM account_balances "
            "WHERE period = %s AND account_code LIKE '6%%'", period)

        total_revenue = Decimal("0")
        for ra in revenue_accounts:
            if ra["balance"] > 0:
                # 借：收入科目，贷：本年利润
                self.create_entry([
                    {"account_code": ra["account_code"], "direction": "debit",
                     "amount": abs(ra["balance"])},
                    {"account_code": "3101", "direction": "credit",
                     "amount": abs(ra["balance"])},  # 本年利润
                ], f"结转收入 {ra['account_code']} 期间 {period}", "system")
                total_revenue += abs(ra["balance"])

        # 2. 结转费用
        expense_accounts = self.db.query(
            "SELECT account_code, balance FROM account_balances "
            "WHERE period = %s AND account_code LIKE '5%%'", period)

        total_expense = Decimal("0")
        for ea in expense_accounts:
            if ea["balance"] > 0:
                self.create_entry([
                    {"account_code": "3101", "direction": "debit",
                     "amount": abs(ea["balance"])},
                    {"account_code": ea["account_code"], "direction": "credit",
                     "amount": abs(ea["balance"])},
                ], f"结转费用 {ea['account_code']} 期间 {period}", "system")
                total_expense += abs(ea["balance"])

        net_profit = total_revenue - total_expense

        return {"period": period, "total_revenue": total_revenue,
                "total_expense": total_expense,
                "net_profit": net_profit}
```

## 异常场景补充

### 场景：期末结转重复执行

```
触发：结转脚本被重复运行 → 收入/费用被结转两次 → 本年利润虚增
检测：
  1. 期间已结转标记存在 → 重复执行
  2. 结转后收入科目余额应为 0 → 不为 0 则异常
处理：
  1. 结转前检查期间是否已结转
  2. 已结转 → 拒绝重复执行
  3. 已重复 → 反向冲销后重新结转
预防：期间结转锁 + 结转标记 + 冲销机制
```

### 场景：借贷平衡校验被绕过

```
触发：直接操作数据库插入分录 → 绕过校验 → 借贷不平衡 → 财务报表错误
检测：
  1. 定期对账检查：各期间借贷总额是否平衡
  2. 不平衡差异 > 0 → 数据完整性问题
处理：
  1. 找出不平衡的分录
  2. 补录调整分录
  3. 修复写入路径（禁止直接操作 DB）
预防：应用层强制校验 + DB 触发器兜底 + 定期对账
```

## 财务对账自动化完整实现

```python
class FinancialReconciliationService:
    """财务对账：银行流水 vs 系统记录 + 差异分析 + 自动调节"""

    def reconcile_bank_statement(self, account_id, statement_date):
        """对账银行流水"""
        # 1. 获取银行流水
        bank_records = self.db.query(
            "SELECT * FROM bank_statements "
            "WHERE account_id = %s AND statement_date = %s "
            "ORDER BY transaction_date", account_id, statement_date)

        # 2. 获取系统记录
        system_records = self.db.query(
            "SELECT * FROM accounting_entries ae "
            "JOIN accounting_entry_lines ael ON ae.entry_id = ael.entry_id "
            "WHERE ael.account_code = (SELECT bank_account_code FROM bank_accounts WHERE id = %s) "
            "AND DATE(ae.created_at) = %s "
            "ORDER BY ae.created_at", account_id, statement_date)

        # 3. 逐笔匹配
        matched = []
        unmatched_bank = list(bank_records)
        unmatched_system = list(system_records)

        for br in bank_records[:]:
            for sr in system_records[:]:
                if self._is_match(br, sr):
                    matched.append({"bank": br, "system": sr,
                                   "match_type": "exact"})
                    unmatched_bank.remove(br)
                    unmatched_system.remove(sr)
                    break

        # 4. 模糊匹配（金额差 < 1 元）
        for br in unmatched_bank[:]:
            for sr in unmatched_system[:]:
                if abs(br["amount"] - sr["amount"]) < 1:
                    matched.append({"bank": br, "system": sr,
                                   "match_type": "fuzzy",
                                   "difference": round(br["amount"] - sr["amount"], 2)})
                    unmatched_bank.remove(br)
                    unmatched_system.remove(sr)
                    break

        # 5. 计算调节表
        bank_balance = sum(r["amount"] for r in bank_records)
        system_balance = sum(r["amount"] for r in system_records)

        # 银行已入账但系统未记录
        bank_only_total = sum(r["amount"] for r in unmatched_bank)
        # 系统已记录但银行未入账
        system_only_total = sum(r["amount"] for r in unmatched_system)

        adjusted_bank = bank_balance + system_only_total
        adjusted_system = system_balance + bank_only_total

        result = {
            "account_id": account_id,
            "statement_date": statement_date,
            "bank_balance": round(bank_balance, 2),
            "system_balance": round(system_balance, 2),
            "matched_count": len(matched),
            "unmatched_bank_count": len(unmatched_bank),
            "unmatched_system_count": len(unmatched_system),
            "bank_only_items": [{"date": r["transaction_date"], "amount": r["amount"],
                                "description": r["description"]} for r in unmatched_bank],
            "system_only_items": [{"date": r["created_at"], "amount": r["amount"],
                                  "description": r["description"]} for r in unmatched_system],
            "adjusted_bank": round(adjusted_bank, 2),
            "adjusted_system": round(adjusted_system, 2),
            "is_balanced": abs(adjusted_bank - adjusted_system) < 0.01
        }

        # 记录对账结果
        self.db.insert("reconciliation_results", {
            "id": str(uuid4()), "account_id": account_id,
            "statement_date": statement_date,
            "result": json.dumps(result),
            "status": "balanced" if result["is_balanced"] else "unbalanced",
            "created_at": now()
        })

        return result

    def _is_match(self, bank_record, system_record):
        """判断是否匹配"""
        # 金额一致 + 日期差 < 3 天
        amount_match = abs(bank_record["amount"] - abs(system_record["amount"])) < 0.01
        date_diff = abs((bank_record["transaction_date"] - system_record["created_at"]).total_seconds())
        date_match = date_diff < 3 * 86400

        return amount_match and date_match

    def auto_create_adjusting_entries(self, reconciliation_result):
        """自动创建调节分录"""
        adjustments = []

        for item in reconciliation_result.get("bank_only_items", []):
            # 银行已入账但系统未记录 → 补录
            entry = self.accounting_service.create_entry([
                {"account_code": "1002", "direction": "debit", "amount": item["amount"]},
                {"account_code": "6602", "direction": "credit", "amount": item["amount"]},
            ], f"银行对账调节: {item['description']}", "reconciliation")
            adjustments.append(entry)

        return {"adjustments_created": len(adjustments)}
```

## 异常场景补充

### 场景：对账匹配规则误匹配

```
触发：两笔金额相同的交易 → 错误匹配 → 一笔银行记录被标记为已匹配但实际未对上
检测：
  1. 匹配但描述不符 → 可能误匹配
  2. 对账后仍有未匹配项 → 匹配错误
处理：
  1. 匹配规则增加描述相似度检查
  2. 金额相同+日期相近的多笔交易 → 人工确认
  3. 误匹配 → 取消匹配后重新对账
预防：描述匹配 + 多笔同金额人工确认 + 匹配可撤销
```

### 场景：银行流水数据延迟

```
触发：银行 T+1 提供流水 → 对账延迟 1 天 → 当日无法对账 → 财务风险
检测：
  1. 银行流水数据不是最新的 → 延迟
  2. 对账日期与当前日期差 > 1 天 → 延迟
处理：
  1. 实时银行 API 对接（部分银行支持）
  2. 延迟期间使用预估对账
  3. 流水到齐后重新精确对账
预防：实时 API + 预估对账 + 延迟告警
```

## 财务报表自动生成完整实现

```python
class FinancialReportService:
    """财务报表自动生成：资产负债表 + 利润表 + 现金流量表"""

    def generate_balance_sheet(self, entity_id, as_of_date):
        """生成资产负债表"""
        # 1. 资产类
        assets = {}
        asset_accounts = self.db.query(
            "SELECT a.account_code, a.name, a.account_type, "
            "COALESCE(SUM(ab.balance), 0) as balance "
            "FROM accounts a "
            "LEFT JOIN account_balances ab ON a.account_code = ab.account_code "
            "AND ab.period = %s "
            "WHERE a.entity_id = %s AND a.account_type = 'asset' AND a.status = 'active' "
            "GROUP BY a.account_code, a.name, a.account_type "
            "ORDER BY a.account_code", as_of_date[:6], entity_id)

        current_assets = 0
        non_current_assets = 0
        for acc in asset_accounts:
            balance = acc["balance"]
            # 1xxx = 流动资产, 15xx+ = 非流动资产
            if acc["account_code"].startswith("1") and int(acc["account_code"][1:3]) < 5:
                current_assets += balance
            else:
                non_current_assets += balance
            assets[acc["account_code"]] = {"name": acc["name"], "balance": balance}

        total_assets = current_assets + non_current_assets

        # 2. 负债类
        liabilities = {}
        liability_accounts = self.db.query(
            "SELECT a.account_code, a.name, "
            "COALESCE(SUM(ab.balance), 0) as balance "
            "FROM accounts a "
            "LEFT JOIN account_balances ab ON a.account_code = ab.account_code "
            "AND ab.period = %s "
            "WHERE a.entity_id = %s AND a.account_type = 'liability' AND a.status = 'active' "
            "GROUP BY a.account_code, a.name "
            "ORDER BY a.account_code", as_of_date[:6], entity_id)

        current_liabilities = 0
        non_current_liabilities = 0
        for acc in liability_accounts:
            balance = acc["balance"]
            if acc["account_code"].startswith("2") and int(acc["account_code"][1:3]) < 3:
                current_liabilities += balance
            else:
                non_current_liabilities += balance
            liabilities[acc["account_code"]] = {"name": acc["name"], "balance": balance}

        total_liabilities = current_liabilities + non_current_liabilities

        # 3. 所有者权益
        equity_accounts = self.db.query(
            "SELECT a.account_code, a.name, "
            "COALESCE(SUM(ab.balance), 0) as balance "
            "FROM accounts a "
            "LEFT JOIN account_balances ab ON a.account_code = ab.account_code "
            "AND ab.period = %s "
            "WHERE a.entity_id = %s AND a.account_type = 'equity' AND a.status = 'active' "
            "GROUP BY a.account_code, a.name "
            "ORDER BY a.account_code", as_of_date[:6], entity_id)

        total_equity = sum(acc["balance"] for acc in equity_accounts)

        # 4. 验证平衡：资产 = 负债 + 权益
        is_balanced = abs(total_assets - (total_liabilities + total_equity)) < 0.01

        report = {
            "entity_id": entity_id,
            "as_of_date": as_of_date,
            "assets": {
                "current": round(current_assets, 2),
                "non_current": round(non_current_assets, 2),
                "total": round(total_assets, 2),
                "details": assets
            },
            "liabilities": {
                "current": round(current_liabilities, 2),
                "non_current": round(non_current_liabilities, 2),
                "total": round(total_liabilities, 2),
                "details": liabilities
            },
            "equity": {
                "total": round(total_equity, 2),
                "details": {acc["account_code"]: {"name": acc["name"], "balance": acc["balance"]} for acc in equity_accounts}
            },
            "is_balanced": is_balanced,
            "imbalance": round(total_assets - total_liabilities - total_equity, 2) if not is_balanced else 0
        }

        # 保存报表
        self.db.insert("financial_reports", {
            "report_id": str(uuid4()),
            "entity_id": entity_id,
            "report_type": "balance_sheet",
            "period": as_of_date[:6],
            "content": json.dumps(report),
            "is_balanced": is_balanced,
            "generated_at": now()
        })

        return report

    def generate_income_statement(self, entity_id, period_start, period_end):
        """生成利润表"""
        # 1. 营业收入
        revenue = self.db.query(
            "SELECT a.account_code, a.name, COALESCE(SUM(ab.balance), 0) as balance "
            "FROM accounts a LEFT JOIN account_balances ab ON a.account_code = ab.account_code "
            "WHERE a.entity_id = %s AND a.account_type = 'revenue' AND a.status = 'active' "
            "AND ab.period BETWEEN %s AND %s "
            "GROUP BY a.account_code, a.name", entity_id, period_start, period_end)

        total_revenue = sum(r["balance"] for r in revenue)

        # 2. 营业成本
        cost = self.db.query(
            "SELECT a.account_code, a.name, COALESCE(SUM(ab.balance), 0) as balance "
            "FROM accounts a LEFT JOIN account_balances ab ON a.account_code = ab.account_code "
            "WHERE a.entity_id = %s AND a.account_code LIKE '5%%' AND a.status = 'active' "
            "AND ab.period BETWEEN %s AND %s "
            "GROUP BY a.account_code, a.name", entity_id, period_start, period_end)

        total_cost = sum(c["balance"] for c in cost)

        # 3. 营业利润
        operating_profit = total_revenue - total_cost

        # 4. 期间费用
        expenses = self.db.query(
            "SELECT a.account_code, a.name, COALESCE(SUM(ab.balance), 0) as balance "
            "FROM accounts a LEFT JOIN account_balances ab ON a.account_code = ab.account_code "
            "WHERE a.entity_id = %s AND a.account_code LIKE '6%%' AND a.status = 'active' "
            "AND ab.period BETWEEN %s AND %s "
            "GROUP BY a.account_code, a.name", entity_id, period_start, period_end)

        total_expenses = sum(e["balance"] for e in expenses)

        # 5. 利润总额
        profit_before_tax = operating_profit - total_expenses

        # 6. 所得税
        tax_rate = 0.25
        tax = max(0, profit_before_tax * tax_rate)

        # 7. 净利润
        net_profit = profit_before_tax - tax

        report = {
            "entity_id": entity_id,
            "period_start": period_start,
            "period_end": period_end,
            "total_revenue": round(total_revenue, 2),
            "total_cost": round(total_cost, 2),
            "gross_profit": round(total_revenue - total_cost, 2),
            "gross_margin": round((total_revenue - total_cost) / max(total_revenue, 1) * 100, 2),
            "total_expenses": round(total_expenses, 2),
            "operating_profit": round(operating_profit, 2),
            "profit_before_tax": round(profit_before_tax, 2),
            "tax": round(tax, 2),
            "net_profit": round(net_profit, 2),
            "net_margin": round(net_profit / max(total_revenue, 1) * 100, 2),
            "revenue_details": revenue,
            "cost_details": cost,
            "expense_details": expenses
        }

        self.db.insert("financial_reports", {
            "report_id": str(uuid4()),
            "entity_id": entity_id,
            "report_type": "income_statement",
            "period": f"{period_start}-{period_end}",
            "content": json.dumps(report),
            "generated_at": now()
        })

        return report
```

## 异常场景补充

### 场景：资产负债表不平衡

```
触发：生成资产负债表 → 资产 ≠ 负债 + 权益 → 差额 0.03 元 → 浮点精度问题
检测：
  1. 资产负债表不平衡 → 浮点精度或记账错误
  2. 差额 < 0.1 → 可能是浮点精度
  3. 差额 > 0.1 → 记账错误
处理：
  1. 使用 Decimal 类型替代 float
  2. 差额 < 0.1 → 四舍五入调整
  3. 差额 > 0.1 → 排查记账错误
预防：Decimal 类型 + 金额精度规范 + 定期对账
```

### 场景：跨期数据导致报表不准确

```
触发：12 月 31 日生成年度报表 → 但 1 月有跨年调整分录 → 报表数据不准
检测：
  1. 报表生成后又有新分录 → 报表过时
  2. 报表与最新账面余额不一致 → 数据变更
处理：
  1. 报表生成时锁定期间（不允许新分录）
  2. 报表标记生成时间点
  3. 重新生成报表
预防：期间锁定 + 生成时间标记 + 可重新生成
```

## 会计凭证自动生成完整实现

```python
class VoucherAutoGenerationService:
    """凭证自动生成：业务事件 → 凭证模板 → 自动生成凭证 → 审核"""

    VOUCHER_TEMPLATES = {
        "sales_revenue": {
            "name": "销售收入",
            "debits": [{"account": "1122", "name": "应收账款"}],
            "credits": [{"account": "6001", "name": "主营业务收入"},
                       {"account": "2221", "name": "应交税费-销项税"}]
        },
        "purchase_cost": {
            "name": "采购成本",
            "debits": [{"account": "1401", "name": "库存商品"},
                      {"account": "2221_1", "name": "应交税费-进项税"}],
            "credits": [{"account": "2202", "name": "应付账款"}]
        },
        "payment_received": {
            "name": "收到货款",
            "debits": [{"account": "1002", "name": "银行存款"}],
            "credits": [{"account": "1122", "name": "应收账款"}]
        },
        "payment_made": {
            "name": "支付货款",
            "debits": [{"account": "2202", "name": "应付账款"}],
            "credits": [{"account": "1002", "name": "银行存款"}]
        },
        "salary_expense": {
            "name": "工资费用",
            "debits": [{"account": "6601", "name": "管理费用-工资"}],
            "credits": [{"account": "2211", "name": "应付职工薪酬"}]
        },
    }

    def generate_voucher(self, business_event):
        """根据业务事件生成凭证"""
        event_type = business_event["event_type"]

        if event_type not in self.VOUCHER_TEMPLATES:
            return {"status": "no_template", "event_type": event_type}

        template = self.VOUCHER_TEMPLATES[event_type]
        amount = Decimal(str(business_event["amount"]))

        # 1. 计算各科目金额
        if event_type == "sales_revenue":
            tax_rate = Decimal("0.13")
            revenue = (amount / (Decimal("1") + tax_rate)).quantize(
                Decimal("0.01"), rounding=ROUND_HALF_UP)
            tax = amount - revenue

            entries = [
                {"account_code": "1122", "account_name": "应收账款",
                 "debit": str(amount), "credit": "0.00"},
                {"account_code": "6001", "account_name": "主营业务收入",
                 "debit": "0.00", "credit": str(revenue)},
                {"account_code": "2221", "account_name": "应交税费-销项税",
                 "debit": "0.00", "credit": str(tax)},
            ]

        elif event_type == "purchase_cost":
            tax_rate = Decimal("0.13")
            cost = (amount / (Decimal("1") + tax_rate)).quantize(
                Decimal("0.01"), rounding=ROUND_HALF_UP)
            tax = amount - cost

            entries = [
                {"account_code": "1401", "account_name": "库存商品",
                 "debit": str(cost), "credit": "0.00"},
                {"account_code": "2221_1", "account_name": "应交税费-进项税",
                 "debit": str(tax), "credit": "0.00"},
                {"account_code": "2202", "account_name": "应付账款",
                 "debit": "0.00", "credit": str(amount)},
            ]

        elif event_type in ["payment_received", "payment_made", "salary_expense"]:
            # 单一金额映射
            for debit in template["debits"]:
                entries.append({
                    "account_code": debit["account"],
                    "account_name": debit["name"],
                    "debit": str(amount), "credit": "0.00"
                })
            for credit in template["credits"]:
                entries.append({
                    "account_code": credit["account"],
                    "account_name": credit["name"],
                    "debit": "0.00", "credit": str(amount)
                })

        # 2. 验证借贷平衡
        total_debit = sum(Decimal(e["debit"]) for e in entries)
        total_credit = sum(Decimal(e["credit"]) for e in entries)

        if abs(total_debit - total_credit) > Decimal("0.01"):
            # 不平衡 → 调整最后一笔
            diff = total_debit - total_credit
            if diff > 0:
                entries[-1]["credit"] = str(
                    Decimal(entries[-1]["credit"]) + diff)
            else:
                entries[-1]["debit"] = str(
                    Decimal(entries[-1]["debit"]) - diff)

        # 3. 生成凭证
        voucher_number = self._generate_voucher_number()
        voucher_id = str(uuid4())

        self.db.insert("vouchers", {
            "voucher_id": voucher_id,
            "voucher_number": voucher_number,
            "entity_id": business_event["entity_id"],
            "event_type": event_type,
            "event_ref_id": business_event.get("ref_id"),
            "template_name": template["name"],
            "amount": str(amount),
            "entries": json.dumps(entries),
            "total_debit": str(total_debit),
            "total_credit": str(total_credit),
            "is_balanced": abs(total_debit - total_credit) < Decimal("0.01"),
            "status": "auto_generated",  # auto_generated → approved → posted
            "generated_at": now()
        })

        return {"voucher_id": voucher_id, "voucher_number": voucher_number,
                "entries": entries, "is_balanced": True}

    def approve_voucher(self, voucher_id, approver_id):
        """审核凭证"""
        voucher = self.db.get_voucher(voucher_id)

        if voucher["status"] not in ["auto_generated", "pending_review"]:
            return {"status": "cannot_approve", "current": voucher["status"]}

        # 1. 验证借贷平衡
        if not voucher["is_balanced"]:
            return {"status": "not_balanced"}

        # 2. 审核通过
        self.db.update("vouchers",
            {"status": "approved", "approved_by": approver_id,
             "approved_at": now()},
            {"voucher_id": voucher_id})

        # 3. 自动过账
        self._post_voucher(voucher)

        return {"voucher_id": voucher_id, "status": "approved"}

    def _post_voucher(self, voucher):
        """过账（更新科目余额）"""
        entries = json.loads(voucher["entries"])
        period = voucher["generated_at"].strftime("%Y%m")

        for entry in entries:
            account_code = entry["account_code"]
            debit = Decimal(entry["debit"])
            credit = Decimal(entry["credit"])

            # 更新科目余额
            self.db.execute(
                "INSERT INTO account_balances "
                "(account_code, period, debit_total, credit_total, balance, updated_at) "
                "VALUES (%s, %s, %s, %s, %s, %s) "
                "ON CONFLICT (account_code, period) DO UPDATE "
                "SET debit_total = debit_total + %s, "
                "credit_total = credit_total + %s, "
                "balance = balance + %s, "
                "updated_at = %s",
                account_code, period, str(debit), str(credit),
                str(debit - credit), now(),
                str(debit), str(credit), str(debit - credit), now())

        # 更新凭证状态为已过账
        self.db.update("vouchers",
            {"status": "posted", "posted_at": now()},
            {"voucher_id": voucher["voucher_id"]})

    def _generate_voucher_number(self):
        """生成凭证编号"""
        today = now().strftime("%Y%m%d")
        seq = self.redis.incr(f"voucher_seq:{today}")
        return f"PZ-{today}-{seq:04d}"
```

## 异常场景补充

### 场景：自动凭证借贷不平

```
触发：含税金额计算四舍五入 → 借方 100.03 + 贷方 100.01 + 100.01 = 100.02 → 差 0.01
检测：
  1. 凭证总借方 ≠ 总贷方 → 不平衡
  2. 差额 < 0.1 → 可能是精度问题
处理：
  1. 最后一笔金额自动调整（差值归入最后一笔贷方）
  2. 调整记录标记为"尾差调整"
  3. 全程使用 Decimal 计算
预防：Decimal + 尾差调整 + 平衡验证
```

### 场景：凭证重复过账

```
触发：过账操作超时 → 重试 → 重复执行 → 科目余额被加两次 → 余额错误
检测：
  1. 科目余额与明细不一致 → 重复过账
  2. 同一凭证有两条过账记录 → 重复
处理：
  1. 过账操作幂等（凭证状态检查：已过账不再执行）
  2. 余额更新使用 ON CONFLICT 语句
  3. 定期对账验证
预防：幂等检查 + 事务保护 + 定期对账
```

## 财务对账与差错处理完整实现

```python
class FinancialReconciliationService:
    """财务对账：银行对账单匹配 → 差异分类 → 差错处理 → 自动对账规则"""

    def reconcile_period(self, entity_id, period_start, period_end):
        """执行期间对账"""
        # 1. 获取内部账本
        ledger_entries = self.db.query(
            "SELECT * FROM settlement_ledger "
            "WHERE entity_id = %s AND created_at BETWEEN %s AND %s "
            "AND status = 'settled' ORDER BY created_at",
            entity_id, period_start, period_end)

        # 2. 获取银行对账单
        bank_entries = self.db.query(
            "SELECT * FROM bank_statements "
            "WHERE entity_id = %s AND transaction_date BETWEEN %s AND %s "
            "ORDER BY transaction_date",
            entity_id, period_start, period_end)

        # 3. 匹配
        matched = []
        unmatched_ledger = []
        unmatched_bank = list(bank_entries)

        for entry in ledger_entries:
            match = self._find_bank_match(entry, unmatched_bank)

            if match:
                matched.append({
                    "ledger_entry": entry,
                    "bank_entry": match,
                    "match_type": "exact" if self._is_exact_match(entry, match) else "fuzzy"
                })
                unmatched_bank.remove(match)
            else:
                unmatched_ledger.append(entry)

        # 4. 分类差异数
        discrepancies = []

        # 账本有但银行没有 → 可能是时间差或银行遗漏
        for entry in unmatched_ledger:
            discrepancies.append({
                "type": "missing_in_bank",
                "entry": entry,
                "amount": entry["amount"],
                "suggested_action": "timing_difference" if self._is_recent(entry) else "investigate"
            })

        # 银行有但账本没有 → 可能是银行手续费或遗漏入账
        for entry in unmatched_bank:
            discrepancies.append({
                "type": "missing_in_ledger",
                "entry": entry,
                "amount": entry["amount"],
                "suggested_action": "bank_fee" if self._is_small_amount(entry) else "investigate"
            })

        # 5. 记录对账结果
        reconciliation_id = str(uuid4())
        self.db.insert("reconciliations", {
            "reconciliation_id": reconciliation_id,
            "entity_id": entity_id,
            "period_start": period_start,
            "period_end": period_end,
            "ledger_count": len(ledger_entries),
            "bank_count": len(bank_entries),
            "matched_count": len(matched),
            "unmatched_ledger_count": len(unmatched_ledger),
            "unmatched_bank_count": len(unmatched_bank),
            "discrepancy_count": len(discrepancies),
            "status": "completed" if not discrepancies else "has_discrepancies",
            "created_at": now()
        })

        return {
            "reconciliation_id": reconciliation_id,
            "matched": len(matched),
            "unmatched_ledger": len(unmatched_ledger),
            "unmatched_bank": len(unmatched_bank),
            "discrepancies": discrepancies,
            "match_rate": round(len(matched) / max(len(ledger_entries), 1) * 100, 1)
        }

    def handle_discrepancy(self, discrepancy_id, handler_id, action, notes=None):
        """处理差异"""
        discrepancy = self.db.get_discrepancy(discrepancy_id)

        if action == "timing_difference":
            # 时间差 → 标记待自动解决
            self.db.update("reconciliation_discrepancies",
                {"status": "auto_resolve_pending",
                 "handler_id": handler_id,
                 "action": action,
                 "notes": notes,
                 "handled_at": now()},
                {"id": discrepancy_id})

            # 2 天后自动解决
            self.task_queue.schedule(
                self._auto_resolve_timing_difference, discrepancy_id,
                delay_seconds=172800)

        elif action == "create_adjustment":
            # 创建调整分录
            amount = Decimal(str(discrepancy["amount"]))
            adjustment_voucher = self.voucher_service.generate_voucher({
                "event_type": "reconciliation_adjustment",
                "entity_id": discrepancy["entity_id"],
                "amount": amount,
                "ref_id": discrepancy_id
            })

            self.db.update("reconciliation_discrepancies",
                {"status": "adjusted",
                 "adjustment_voucher_id": adjustment_voucher["voucher_id"],
                 "handler_id": handler_id,
                 "action": action,
                 "notes": notes,
                 "handled_at": now()},
                {"id": discrepancy_id})

        elif action == "investigate":
            # 需进一步调查
            self.db.update("reconciliation_discrepancies",
                {"status": "investigating",
                 "handler_id": handler_id,
                 "action": action,
                 "notes": notes,
                 "handled_at": now()},
                {"id": discrepancy_id})

            self.notification.send("finance_team",
                f"对账差异需调查: {discrepancy_id}, 金额: {discrepancy['amount']}")

        return {"discrepancy_id": discrepancy_id, "action": action}

    def _find_bank_match(self, ledger_entry, bank_entries):
        """在银行对账单中查找匹配"""
        amount = Decimal(str(ledger_entry["amount"]))

        for bank_entry in bank_entries:
            bank_amount = Decimal(str(bank_entry["amount"]))

            # 精确匹配：金额 + 日期 + 参考号
            if abs(amount - bank_amount) < Decimal("0.01"):
                if ledger_entry.get("reference") == bank_entry.get("reference"):
                    return bank_entry

                # 日期接近（1 天内）
                date_diff = abs((ledger_entry["created_at"].date() -
                               bank_entry["transaction_date"]).days)
                if date_diff <= 1:
                    return bank_entry

        return None

    def _is_exact_match(self, ledger_entry, bank_entry):
        """是否精确匹配"""
        return (ledger_entry.get("reference") == bank_entry.get("reference") and
                ledger_entry["created_at"].date() == bank_entry["transaction_date"])

    def _is_recent(self, entry):
        """是否最近入账（可能是时间差）"""
        return (now() - entry["created_at"]).days <= 2

    def _is_small_amount(self, entry):
        """是否小额（可能是银行手续费）"""
        return abs(Decimal(str(entry["amount"]))) <= Decimal("50")

    def _auto_resolve_timing_difference(self, discrepancy_id):
        """自动解决时间差"""
        discrepancy = self.db.get_discrepancy(discrepancy_id)

        if discrepancy["status"] != "auto_resolve_pending":
            return

        # 检查银行对账单是否已出现
        entry = discrepancy["entry"]
        bank_match = self.db.query_one(
            "SELECT * FROM bank_statements "
            "WHERE entity_id = %s AND amount = %s "
            "AND transaction_date BETWEEN %s AND %s",
            entry["entity_id"], entry["amount"],
            entry["created_at"] - timedelta(days=1),
            entry["created_at"] + timedelta(days=3))

        if bank_match:
            self.db.update("reconciliation_discrepancies",
                {"status": "auto_resolved",
                 "resolution": "bank_entry_found",
                 "resolved_at": now()},
                {"id": discrepancy_id})
        else:
            # 仍未找到 → 需人工处理
            self.db.update("reconciliation_discrepancies",
                {"status": "escalated",
                 "resolution": "auto_resolve_failed"},
                {"id": discrepancy_id})
```

## 异常场景补充

### 场景：对账发现大量不明差异

```
触发：月末对账发现 200 笔差异 → 大部分无法自动解释 → 财务风险
检测：
  1. 对账差异率 > 5% → 异常
  2. 不明差异金额总计 > 10 万元 → 重大风险
处理：
  1. 暂停该实体自动结算
  2. 逐笔调查差异原因
  3. 必要时联系银行核实
预防：差异率监控 + 暂停结算 + 逐笔调查
```

### 场景：自动调整规则导致累计偏差

```
触发：小额差异（<50 元）自动调整 → 每月 30 笔 → 年累计 1.8 万元 → 财务报表偏差
检测：
  1. 自动调整累计金额超过阈值 → 累计偏差
  2. 自动调整笔数占比 > 30% → 规则过宽
处理：
  1. 自动调整设年度累计上限
  2. 超过上限的调整需人工审批
  3. 定期审查自动调整规则
预防：累计上限 + 人工审批 + 规则审查
```
