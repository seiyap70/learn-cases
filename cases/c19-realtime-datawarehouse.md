# C19: 数据平台的实时数仓建设

## 业务场景

某电商公司的数据平台需要同时支持两种查询：
- **实时查询**："过去 1 小时各品类的实时 GMV" → 业务运营看板，延迟 < 5 分钟
- **离线查询**："过去 30 天各品类的趋势对比" → 数据分析师报表

目前使用纯离线数仓（Hive/Spark），数据延迟 T+1。业务方要求关键指标实时化。

**已知数据：**
- 日均订单量：100 万
- 日均事件量：5 亿条（订单、点击、搜索、曝光）
- 数据源：MySQL（业务库）+ Kafka（实时事件流）
- 实时查询 QPS：约 100（运营看板）
- 离线查询 QPS：约 50（分析师报表）

**核心矛盾：实时与离线的数据口径必须一致**

"过去 1 小时的 GMV"加上"今天剩余 23 小时的 GMV"必须等于"今天的总 GMV"。但实时数据和离线数据走不同的计算管道，容易出现口径不一致。

## 核心挑战

### 挑战 1：Lambda 架构的维护成本

实时层用 Flink，离线层用 Spark。同样的业务逻辑需要写两套代码（Flink SQL + Spark SQL），逻辑必须一致但框架不同 → 维护噩梦。

### 挑战 2：实时层的迟到数据

用户 10:00 下单，订单系统 10:05 才写入数据库（5 分钟延迟）。Flink 用事件时间还是处理时间？如果用事件时间，GMV 归属到 10:00；如果用处理时间，归属到 10:05 → 口径不同。

### 挑战 3：口径校准

实时层和离线层的同一指标可能差几万元（迟到数据、计算精度差异）。如何自动发现和修正偏差？

## 设计约束

- 实时指标延迟 < 5 分钟
- 实时与离线数据口径偏差 < 0.1%
- 离线数仓（Hive/Spark）不可废弃
- 分析师使用 SQL 查询（不学新语言）

## 请先独立思考（限时 35 分钟）

1. Lambda 架构 vs Kappa 架构，在本场景下的优劣？Kappa 是否可行？
2. Flink 的 Watermark 策略如何设置？5 分钟迟到数据如何处理？
3. 实时层和离线层的口径校准机制如何设计？

---

## 设计解析

### 方案选择：改良版 Lambda

**为什么不选 Kappa？**

Kappa（纯实时无离线层）的问题：
1. 历史重算——Kafka 不保留 30 天全量数据（5 亿条/天 × 30 天 = 7.5TB）
2. 大范围查询——"过去 30 天趋势"在 Flink 中不如在 Hive 分区表中高效
3. 复杂 SQL——分析师的 ad-hoc 查询（多表 JOIN + 窗口函数）在 Flink 中不能执行

**改良版 Lambda：实时覆盖 1 天，离线覆盖历史**

```
查询路由：
  "过去1小时GMV" → ClickHouse（Flink 实时写入）
  "过去30天趋势" → Hive（Spark 离线计算）
  "今天总GMV"   → ClickHouse 实时数据 + Hive 校验
```

### CDC 管道：MySQL → Kafka

```yaml
# Canal/Debezium 配置
# 捕获 MySQL 的 orders、payments、users 表变更
# 写入 Kafka CDC Topic
```

CDC 消息格式：
```json
{
  "op": "INSERT",
  "db": "order_db",
  "table": "orders",
  "ts_ms": 1715000000,
  "data": {
    "order_id": "ORD-001",
    "user_id": "U-12345",
    "amount": 99.00,
    "category": "electronics",
    "status": "PAID",
    "created_at": 1714999995
  }
}
```

### Flink 实时计算

```sql
-- Flink SQL：各品类实时 GMV（1 分钟滚动窗口）

CREATE TABLE kafka_orders (
    order_id STRING,
    user_id STRING,
    amount DECIMAL(10,2),
    category STRING,
    status STRING,
    event_time TIMESTAMP(3) METADATA FROM 'value.ts_ms',
    WATERMARK FOR event_time AS event_time - INTERVAL '5' MINUTE
) WITH (
    'connector' = 'kafka',
    'topic' = 'cdc_orders',
    'format' = 'json'
);

-- 1 分钟窗口聚合
INSERT INTO clickhouse_realtime_gmv
SELECT
    TUMBLE_START(event_time, INTERVAL '1' MINUTE) AS bucket,
    category,
    SUM(amount) AS gmv,
    COUNT(*) AS order_count,
    COUNT(DISTINCT user_id) AS buyer_count
FROM kafka_orders
WHERE status = 'PAID'
GROUP BY
    TUMBLE(event_time, INTERVAL '1' MINUTE),
    category;
```

**Watermark = `event_time - 5 MINUTE` 的含义：**

- 允许迟到 5 分钟的数据参与计算
- 10:05 到达的订单（事件时间 10:00）仍归属到 10:00 的窗口
- 超过 5 分钟的迟到数据被丢弃（由离线层兜底）

### ClickHouse 实时存储

```sql
CREATE TABLE realtime_gmv (
    bucket DateTime NOT NULL,
    category String NOT NULL,
    gmv Decimal(15,2) NOT NULL,
    order_count UInt64 NOT NULL,
    buyer_count UInt64 NOT NULL
) ENGINE = AggregatingMergeTree()
ORDER BY (bucket, category)
PARTITION BY toYYYYMMDD(bucket);
```

ClickHouse 选择理由：
- 写入 50 万行/秒（Flink 输出约 100 行/秒 → 远低于上限）
- 查询 < 100ms（满足运营看板需求）
- SQL 完整（分析师可直接查询）

### 口径校准机制

```python
class DataCalibrator:
    """每天凌晨校准实时层与离线层数据"""

    def calibrate(self, date):
        # 离线层：Spark 计算的"今日各品类总 GMV"
        offline = self.hive.query("""
            SELECT category, SUM(gmv) AS gmv
            FROM hive_gmv_daily WHERE dt = '%s'
            GROUP BY category
        """, date)

        # 实时层：ClickHouse 中"今日各品类总 GMV"
        realtime = self.clickhouse.query("""
            SELECT category, SUM(gmv) AS gmv
            FROM realtime_gmv
            WHERE bucket >= '%s 00:00:00' AND bucket < '%s + 1 day'
            GROUP BY category
        """, date)

        # 比对差异
        diffs = []
        for cat in set(offline.keys()) | set(realtime.keys()):
            off = offline.get(cat, 0)
            rt = realtime.get(cat, 0)
            diff_pct = abs(off - rt) / max(off, rt, 1) * 100

            if diff_pct > 0.1:
                diffs.append({
                    "category": cat,
                    "offline": off,
                    "realtime": rt,
                    "diff_pct": round(diff_pct, 3)
                })

        # 修正：以离线层为准，向 ClickHouse 写入校准值
        if diffs:
            for d in diffs:
                self.clickhouse.execute("""
                    INSERT INTO realtime_gmv_calibrated
                    (bucket, category, gmv, note)
                    VALUES ('%s 23:59:00', '%s', %s, 'calibration')
                """, date, d.category, d.offline - d.realtime)

            self.alert_team(f"口径偏差超过0.1%: {len(diffs)}个品类")

        return diffs
```

### 为什么离线层仍必要

| 场景 | 实时层能力 | 离线层必要性 |
|------|-----------|------------|
| 过去1小时GMV | ✓ Flink + ClickHouse | 不需要 |
| 过去30天趋势 | ✗ ClickHouse只存1天 | ✓ Hive分区表 |
| 复杂ad-hoc SQL | ✗ Flink不支持多表JOIN | ✓ Spark完整SQL |
| 口径校准基准 | ✗ 实时层有迟到数据偏差 | ✓ Hive全量计算 |
| Bug修复后重算 | ✗ Kafka不保留30天数据 | ✓ Hive分区可重算 |

## 常见陷阱（深度分析）

### 陷阱 1：Lambda 双套代码不一致

**后果：** Flink SQL 和 Spark SQL 对同一指标的计算逻辑微妙不同（如 Flink 用 `SUM(amount)` 而 Spark 用 `SUM(amount * discount)`）→ 口径偏差 5% → 业务决策错误。

**解决方案：** 定义统一的指标口径文档 + 自动化口径校准。

### 陷阱 2：Watermark 设为 0

**后果：** 所有迟到数据被丢弃 → 实时 GMV 比实际少约 0.3-0.5% → 口径偏差超限。

**解决方案：** Watermark 设为 5 分钟（覆盖 99.9% 的迟到数据），超过 5 分钟的由离线层兜底。

### 陷阱 3：不做口径校准

**后果：** 实时层和离线层的 GMV 差异逐渐积累 → 无人发现 → 业务运营看实时数据（偏高），分析师看离线数据（偏低）→ 决策矛盾。

### 陷阱 4：纯 Kappa 架构

**后果：** 没有离线层 → 无法重算历史数据、无法做复杂 ad-hoc 查询、没有口径校准基准。

## 延伸思考

- **Hudi/Iceberg**：在 Hive 上支持近实时更新（5-10 分钟），缩小实时与离线的差距。
- **CDC 幂等**：同一订单的多次更新（状态变更），Flink 用 retract 机制保证只计算最终状态。
- **指标平台**：统一指标定义（口径文档 + 代码生成），自动生成 Flink SQL 和 Spark SQL，避免双套代码不一致。