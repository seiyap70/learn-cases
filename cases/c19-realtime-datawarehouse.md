# C19: 数据平台的实时数仓建设

## 业务场景

某电商公司数据平台，业务运营团队每天早上 9 点看昨日数据报表，但昨日数据只能看到 T+1（延迟一天）。双十一大促时，运营需要在 5 分钟内知道"哪个品类爆了"来调整资源位——但数据要等到明天才能看到。

**目前架构的问题：**

```
MySQL → 每日凌晨 Sqoop 抽取 → Hive 分区表 → Spark 离线计算 → 报表（T+1）
```

业务方要求：关键指标（GMV、订单量、转化率）实时化，延迟 < 5 分钟。

**已知数据：**
- 日均订单量：100 万
- 日均事件量：5 亿条（订单、点击、搜索、曝光、加购）
- 数据源：MySQL 业务库（10 个分库）+ Kafka 实时事件流
- 实时查询 QPS：约 100（运营看板）
- 离线查询 QPS：约 50（分析师报表）
- 峰值数据量：大促期间事件量 ×10

**核心矛盾：实时与离线的数据口径必须一致**

"过去 1 小时的 GMV"加上"今天剩余 23 小时的 GMV"必须等于"今天的总 GMV"。但实时数据和离线数据走不同的计算管道，容易出现口径不一致：

| 口径偏差来源 | 具体原因 |
|------------|---------|
| 迟到数据 | 10:00 的订单 10:05 才到 Kafka → 实时层计入了 10:05，离线层计入 10:00 |
| 计算精度 | Flink 用 FLOAT，Spark 用 DECIMAL → 精度差异 |
| 过滤条件 | Flink 过滤 `status='PAID'`，Spark 过滤 `status IN ('PAID','REFUNDED')` |
| 去重逻辑 | Flink 用 event_id 去重，Spark 用 order_id 去重 → 可能不一致 |

## 核心挑战

### 挑战 1：Lambda 架构的维护成本

实时层用 Flink，离线层用 Spark。同样的业务逻辑需要写两套代码（Flink SQL + Spark SQL），逻辑必须一致但框架不同 → 维护噩梦。更糟的是，Flink 和 Spark 的 SQL 语法、函数行为有微妙差异（如 Flink 的 `COUNT(DISTINCT)` 在滚动窗口中是精确的，Spark 是近似的 HyperLogLog）。

### 挑战 2：实时层的迟到数据

用户 10:00 下单，订单系统 10:05 才写入数据库（5 分钟延迟）。Flink 用事件时间还是处理时间？

| 时间语义 | GMV归属 | 优点 | 缺点 |
|---------|--------|------|------|
| 事件时间 | 10:00 | 与离线层一致 | 需要等迟到数据，窗口延迟关闭 |
| 处理时间 | 10:05 | 实时性好 | 与离线层不一致 → 口径偏差 |

### 挑战 3：口径校准

实时层和离线层的同一指标可能差几万元（迟到数据、计算精度差异）。如何自动发现和修正偏差？

### 挑战 4：数据回填与重算

Flink 任务出错或逻辑 Bug → 需要重算过去 N 天的实时数据。但 Kafka 只保留 7 天数据 → 无法回填。

## 设计约束

- 实时指标延迟 < 5 分钟
- 实时与离线数据口径偏差 < 0.1%
- 离线数仓（Hive/Spark）不可废弃（分析师依赖）
- 分析师使用 SQL 查询（不学新语言）
- Flink 任务故障恢复时间 < 10 分钟

## 请先独立思考（限时 35 分钟）

1. Lambda 架构 vs Kappa 架构，在本场景下的优劣？纯 Kappa 是否可行？
2. Flink 的 Watermark 策略如何设置？5 分钟迟到数据如何处理？超过 5 分钟的呢？
3. 实时层和离线层的口径校准机制如何设计？校准频率和数据流程？
4. 如何解决 Lambda 双套代码不一致问题？指标平台如何设计？

---

## 设计解析

### 方案选择：改良版 Lambda

**为什么不选 Kappa（纯实时）？**

| 问题 | Kappa 的困境 | Lambda 的解决 |
|------|-------------|-------------|
| 历史重算 | Kafka 只保留 7 天 → 无法重算 30 天 | Hive 分区表可无限重算 |
| 大范围查询 | "过去 30 天趋势"需全量扫描 Kafka | Hive 分区表按天分区，高效扫描 |
| 复杂 SQL | Flink 不支持多表 JOIN + 窗口函数 | Spark 支持完整 SQL |
| 口径校准 | 无离线基准 → 无法校准 | 离线层作为校准基准 |

**改良版 Lambda：实时覆盖当天，离线覆盖历史**

```
查询路由策略：
  "过去1小时GMV" → ClickHouse（Flink 实时写入，1 分钟粒度）
  "过去30天趋势" → Hive（Spark 离线计算，天级粒度）
  "今天总GMV"   → ClickHouse 实时数据（待校准）
  "昨天总GMV"   → Hive 离线数据（校准后，权威值）
```

### 数据流架构：端到端

```
MySQL ──CDC──→ Kafka ──Flink──→ ClickHouse（实时层，当天数据）
   │                                      ↓
   └──Sqoop──→ Hive ──Spark──→ Hive 报表（离线层，T+1 数据）
                                        ↓
                                  口径校准（凌晨自动执行）
```

### CDC 管道：MySQL → Kafka

**Debezium 配置（完整）：**

```yaml
# Debezium Connector 配置
name: order-connector
config:
  connector.class: io.debezium.connector.mysql.MySqlConnector
  database.hostname: mysql-primary
  database.port: 3306
  database.user: debezium
  database.password: ${DEBEZIUM_PASSWORD}
  database.server.id: 184054
  database.server.name: order_db
  database.include.list: order_db,payment_db,user_db
  table.include.list: order_db.orders,order_db.order_items,payment_db.payments,user_db.users
  database.history.kafka.bootstrap.servers: kafka:9092
  database.history.kafka.topic: schema-changes.order_db
  
  # 关键配置：快照模式
  snapshot.mode: schema_only  # 只读表结构，不读历史数据（历史数据由Sqoop处理）
  
  # 关键配置：消息格式
  key.converter: org.apache.kafka.connect.json.JsonConverter
  value.converter: org.apache.kafka.connect.json.JsonConverter
  
  # 关键配置：GTID 支持（高可用）
  gtid.source.includes: server1,server2
  
  # 关键配置：心跳（检测连接存活）
  heartbeat.interval.ms: 10000
```

**CDC 消息格式（完整字段）：**

```json
{
  "schema": {...},
  "payload": {
    "before": null,
    "after": {
      "order_id": "ORD-20241115-001",
      "user_id": "U-12345",
      "amount": 299.00,
      "discount_amount": 30.00,
      "actual_amount": 269.00,
      "category": "electronics",
      "status": "PAID",
      "created_at": "2024-11-15T10:00:00Z",
      "updated_at": "2024-11-15T10:00:05Z"
    },
    "source": {
      "version": "2.4.0.Final",
      "connector": "mysql",
      "name": "order_db",
      "ts_ms": 1700038805000,
      "snapshot": "false",
      "db": "order_db",
      "table": "orders",
      "server_id": 1,
      "gtid": "3E11FA47-30CA-11EC-9E2B-0242AC130002:1",
      "file": "mysql-bin.000003",
      "pos": 157,
      "row": 0,
      "thread": 7,
      "query": null
    },
    "op": "u",  // c=create, u=update, d=delete, r=read(snapshot)
    "ts_ms": 1700038805123,
    "transaction": {"id": "3E11FA47-30CA-11EC-9E2B-0242AC130002:1", "total_order": 1, "data_collection_order": 1}
  }
}
```

### Flink 实时计算：完整 SQL

```sql
-- 1. Kafka Source 表（CDC 格式）
CREATE TABLE kafka_orders (
    order_id STRING,
    user_id STRING,
    amount DECIMAL(10,2),
    discount_amount DECIMAL(10,2),
    actual_amount DECIMAL(10,2),
    category STRING,
    status STRING,
    created_at TIMESTAMP(3),
    updated_at TIMESTAMP(3),
    op STRING METADATA FROM 'value.payload.op',
    -- Watermark：允许 5 分钟迟到
    WATERMARK FOR updated_at AS updated_at - INTERVAL '5' MINUTE
) WITH (
    'connector' = 'kafka',
    'topic' = 'order_db.orders',
    'properties.bootstrap.servers' = 'kafka:9092',
    'properties.group.id' = 'flink-gmv-aggregation',
    'format' = 'debezium-json',
    'scan.startup.mode' = 'latest-offset',  -- 实时任务从最新开始
    -- 关键配置：CDC 去重
    'debezium-json.schema-include' = 'false'
);

-- 2. 1 分钟滚动窗口：各品类实时 GMV
INSERT INTO clickhouse_realtime_gmv
SELECT
    TUMBLE_START(updated_at, INTERVAL '1' MINUTE) AS bucket,
    category,
    SUM(actual_amount) AS gmv,               -- 实付金额（非标价）
    SUM(discount_amount) AS discount_total,   -- 优惠总额
    COUNT(DISTINCT order_id) AS order_count,  -- 订单数
    COUNT(DISTINCT user_id) AS buyer_count,   -- 买家数
    CURRENT_TIMESTAMP AS computed_at          -- 计算时间（用于判断数据新鲜度）
FROM kafka_orders
WHERE status = 'PAID'                        -- 只计已付款订单
  AND op IN ('c', 'u')                       -- 只处理新建和更新
GROUP BY
    TUMBLE(updated_at, INTERVAL '1' MINUTE),
    category;

-- 3. Flink Retract 机制：同一订单状态变更
-- CDC 的 op='u' 会触发 retract：
--   旧状态（status=CREATED, amount=299）→ 撤回
--   新状态（status=PAID, amount=269）→ 重新聚合
-- 这保证了：实时 GMV 只计算 PAID 状态的订单
```

**Watermark = `updated_at - 5 MINUTE` 的含义：**

```
时间轴：  10:00  10:01  10:02  10:03  10:04  10:05
数据到达：  ↑                                    ↑
         正常到达                          迟到 5 分钟到达

10:05 到达的订单（事件时间 10:00）→ 仍归属到 10:00 窗口 ✓
10:06 到达的订单（事件时间 10:00）→ 超过 5 分钟 Watermark → 丢弃
丢弃的数据由离线层兜底（离线层直接读 MySQL，无迟到问题）
```

**迟到数据的量化分析：**

| 迟到时间 | 占比 | 累计覆盖 |
|---------|------|---------|
| < 1 分钟 | 99.0% | 99.0% |
| < 3 分钟 | 99.7% | 99.7% |
| < 5 分钟 | 99.9% | 99.9% |
| < 10 分钟 | 99.95% | 99.95% |
| > 10 分钟 | 0.05% | 离线层兜底 |

5 分钟 Watermark 覆盖 99.9% 的数据 → 实时与离线偏差 < 0.1%。

### ClickHouse 实时存储

```sql
-- 实时 GMV 明细表（1 分钟粒度）
CREATE TABLE realtime_gmv (
    bucket DateTime NOT NULL,
    category LowCardinality(String) NOT NULL,
    gmv Decimal(15,2) NOT NULL,
    discount_total Decimal(15,2) NOT NULL,
    order_count UInt64 NOT NULL,
    buyer_count UInt64 NOT NULL,
    computed_at DateTime NOT NULL
) ENGINE = AggregatingMergeTree()
ORDER BY (bucket, category)
PARTITION BY toYYYYMMDD(bucket)
TTL toDateTime(bucket) + INTERVAL 7 DAY;  -- 只保留 7 天（历史由 Hive 管理）

-- 聚合表（加速查询）
CREATE MATERIALIZED VIEW realtime_gmv_daily
ENGINE = SummingMergeTree()
ORDER BY (day, category)
AS SELECT
    toDate(bucket) AS day,
    category,
    SUM(gmv) AS gmv,
    SUM(discount_total) AS discount_total,
    SUM(order_count) AS order_count,
    SUM(buyer_count) AS buyer_count
FROM realtime_gmv
GROUP BY day, category;

-- 校准表（存储离线层修正值）
CREATE TABLE realtime_gmv_calibrated (
    day Date NOT NULL,
    category LowCardinality(String) NOT NULL,
    realtime_gmv Decimal(15,2) NOT NULL,
    offline_gmv Decimal(15,2) NOT NULL,
    diff_amount Decimal(15,2) NOT NULL,
    diff_pct Float64 NOT NULL,
    calibrated_at DateTime NOT NULL
) ENGINE = MergeTree()
ORDER BY (day, category);
```

**ClickHouse 选择理由：**

| 维度 | ClickHouse | Doris | Druid |
|------|-----------|-------|-------|
| 写入速度 | 50 万行/秒 | 10 万行/秒 | 5 万行/秒 |
| 查询延迟 | < 100ms | < 200ms | < 500ms |
| SQL 完整性 | 高（支持 JOIN） | 高 | 中 |
| 运维复杂度 | 低 | 中 | 高 |
| 适用场景 | 实时报表+分析 | 实时报表 | 时序分析 |

### 离线层：Hive T+1 流程

```
每日 02:00 Sqoop 全量抽取昨日的 MySQL 数据 → Hive 分区表
每日 03:00 Spark SQL 计算离线指标 → hive_gmv_daily
每日 04:00 口径校准任务 → 对比实时层和离线层
每日 05:00 校准值写入 ClickHouse → 实时查询自动修正
```

```sql
-- Hive 离线指标表（天级粒度，权威数据源）
CREATE TABLE hive_gmv_daily (
    dt STRING COMMENT '日期分区 yyyy-MM-dd',
    category STRING,
    gmv DECIMAL(15,2),
    discount_total DECIMAL(15,2),
    order_count BIGINT,
    buyer_count BIGINT
)
PARTITIONED BY (dt STRING)
STORED AS ORC;

-- Spark SQL 计算（每日凌晨执行）
INSERT OVERWRITE TABLE hive_gmv_daily PARTITION (dt='${dt}')
SELECT
    category,
    SUM(actual_amount) AS gmv,
    SUM(discount_amount) AS discount_total,
    COUNT(DISTINCT order_id) AS order_count,
    COUNT(DISTINCT user_id) AS buyer_count
FROM hive_orders
WHERE dt = '${dt}'
  AND status = 'PAID'
GROUP BY category;
```

### 口径校准机制

```python
class DataCalibrator:
    """每天凌晨自动校准实时层与离线层数据"""

    def calibrate(self, date):
        """校准指定日期的数据"""
        # 1. 获取离线层权威数据
        offline = self.hive.query("""
            SELECT category, gmv, order_count, buyer_count
            FROM hive_gmv_daily WHERE dt = '%s'
        """, date)

        # 2. 获取实时层数据
        realtime = self.clickhouse.query("""
            SELECT category, SUM(gmv) AS gmv, SUM(order_count) AS order_count,
                   SUM(buyer_count) AS buyer_count
            FROM realtime_gmv
            WHERE toDate(bucket) = '%s'
            GROUP BY category
        """, date)

        # 3. 逐品类比对差异
        diffs = []
        for cat in set(offline.keys()) | set(realtime.keys()):
            off = offline.get(cat, {"gmv": 0})
            rt = realtime.get(cat, {"gmv": 0})
            diff_amount = off["gmv"] - rt["gmv"]
            diff_pct = abs(diff_amount) / max(abs(off["gmv"]), 1) * 100

            diffs.append({
                "category": cat,
                "offline_gmv": off["gmv"],
                "realtime_gmv": rt["gmv"],
                "diff_amount": diff_amount,
                "diff_pct": round(diff_pct, 3)
            })

        # 4. 分级处理
        for d in diffs:
            if d["diff_pct"] > 1.0:
                # 严重偏差 → 告警 + 人工介入
                self.alert_critical(date, d)
            elif d["diff_pct"] > 0.1:
                # 轻微偏差 → 自动修正
                self.write_calibration(date, d)
            else:
                # 偏差在可接受范围内 → 仅记录
                pass

        # 5. 写入校准记录
        for d in diffs:
            self.clickhouse.execute("""
                INSERT INTO realtime_gmv_calibrated
                (day, category, realtime_gmv, offline_gmv, diff_amount, diff_pct, calibrated_at)
                VALUES ('%s', '%s', %s, %s, %s, %s, NOW())
            """, date, d["category"], d["realtime_gmv"], d["offline_gmv"],
                 d["diff_amount"], d["diff_pct"])

        return diffs

    def write_calibration(self, date, diff):
        """向 ClickHouse 写入修正值"""
        self.clickhouse.execute("""
            INSERT INTO realtime_gmv
            (bucket, category, gmv, discount_total, order_count, buyer_count, computed_at)
            VALUES ('%s 23:59:00', '%s', %s, 0, 0, 0, NOW())
        """, date, diff["category"], diff["diff_amount"])

    def alert_critical(self, date, diff):
        """严重偏差告警"""
        self.slack.send(
            channel="#data-quality",
            message=f"⚠️ 口径严重偏差: date={date}, category={diff['category']}, "
                    f"offline={diff['offline_gmv']}, realtime={diff['realtime_gmv']}, "
                    f"diff={diff['diff_pct']}%"
        )
```

### 指标平台：统一口径定义

**核心问题：** Lambda 双套代码不一致的根因是 Flink SQL 和 Spark SQL 各写各的，没有统一口径。

**解决方案：指标平台——一次定义，两处生成**

```yaml
# 指标定义文件（YAML，版本管理）
metric:
  name: gmv
  display_name: 实付GMV
  description: 订单实际支付金额（含优惠），只计已付款订单
  unit: CNY
  
  # 计算逻辑（统一口径）
  calculation:
    measure: SUM(actual_amount)
    filters:
      - field: status
        operator: EQ
        value: PAID
    dimensions:
      - category
      - province
    time_field: updated_at  # 统一用 updated_at（非 created_at）
    
  # 实时层配置
  realtime:
    source: kafka_orders
    engine: flink
    window: TUMBLE(1 MINUTE)
    watermark: 5 MINUTE
    sink: clickhouse.realtime_gmv
    
  # 离线层配置
  offline:
    source: hive_orders
    engine: spark
    granularity: DAY
    sink: hive.hive_gmv_daily
    
  # 校准配置
  calibration:
    threshold_pct: 0.1
    action: auto_fix
    schedule: "0 4 * * *"
```

```python
class MetricCodeGenerator:
    """从指标定义自动生成 Flink SQL 和 Spark SQL"""

    def generate_flink_sql(self, metric_def):
        filters = " AND ".join(
            f"{f['field']} = '{f['value']}'" for f in metric_def["calculation"]["filters"]
        )
        dimensions = ", ".join(metric_def["calculation"]["dimensions"])
        measure = metric_def["calculation"]["measure"]
        window = metric_def["realtime"]["window"]
        time_field = metric_def["calculation"]["time_field"]
        
        return f"""
        SELECT
            TUMBLE_START({time_field}, {window}) AS bucket,
            {dimensions},
            {measure} AS {metric_def['name']}
        FROM {metric_def['realtime']['source']}
        WHERE {filters}
        GROUP BY TUMBLE({time_field}, {window}), {dimensions}
        """

    def generate_spark_sql(self, metric_def):
        filters = " AND ".join(
            f"{f['field']} = '{f['value']}'" for f in metric_def["calculation"]["filters"]
        )
        dimensions = ", ".join(metric_def["calculation"]["dimensions"])
        measure = metric_def["calculation"]["measure"]
        time_field = metric_def["calculation"]["time_field"]
        
        return f"""
        SELECT
            {dimensions},
            {measure} AS {metric_def['name']}
        FROM {metric_def['offline']['source']}
        WHERE {filters}
        GROUP BY {dimensions}
        """
```

**指标平台的价值：** 口径变更只需修改 YAML → 自动重新生成 Flink SQL 和 Spark SQL → 不再有人工翻译导致的不一致。

### Flink 任务运维

```python
class FlinkTaskMonitor:
    """Flink 任务监控与自动恢复"""

    def check_task_health(self):
        """检查所有 Flink 任务状态"""
        tasks = self.flink_client.list_running_jobs()
        
        for task in tasks:
            # 检查消费延迟（关键指标）
            lag = self.get_kafka_consumer_lag(task.group_id)
            
            if lag > 100000:  # 积压 > 10 万条
                self.alert(f"Flink任务 {task.name} 消费积压: {lag}")
                self.auto_scale(task)  # 自动扩容
            
            # 检查 Checkpoint 成功率
            checkpoint_failures = self.get_checkpoint_failures(task.job_id)
            if checkpoint_failures > 3:
                self.alert(f"Flink任务 {task.name} Checkpoint 连续失败 {checkpoint_failures} 次")
                self.restart_task(task)
            
            # 检查数据新鲜度
            freshness = self.get_data_freshness(task.sink_table)
            if freshness > timedelta(minutes=10):
                self.alert(f"实时数据不新鲜: {task.sink_table} 延迟 {freshness}")
```

## 常见陷阱（深度分析）

### 陷阱 1：Lambda 双套代码不一致

**具体案例：** Flink SQL 用 `SUM(actual_amount)` 计算实付 GMV，Spark SQL 用 `SUM(amount) - SUM(discount_amount)` 计算。当存在部分退款时（actual_amount 可能不等于 amount - discount_amount）→ 口径偏差 5%。

**解决方案：** 指标平台统一口径定义 → 代码自动生成 → 消除人工翻译差异。

### 陷阱 2：Watermark 设为 0

**后果：** 所有迟到数据被丢弃 → 实时 GMV 比实际少约 0.3-0.5% → 口径偏差超限。

**量化：** Watermark=0 时，约 0.5% 的订单被丢弃（1 分钟以上延迟的订单）。Watermark=5min 时，仅 0.05% 丢弃。

**解决方案：** Watermark 设为 5 分钟，超过 5 分钟的迟到数据由离线层兜底。

### 陷阱 3：不做口径校准

**后果：** 实时层和离线层的 GMV 差异逐渐积累 → 无人发现 → 业务运营看实时数据（偏高），分析师看离线数据（偏低）→ 决策矛盾。例如运营根据实时数据判断"电子产品 GMV 超 1000 万"追加了资源位，但实际只有 980 万 → 资源浪费。

**解决方案：** 每日凌晨自动口径校准 + 偏差分级告警。

### 陷阱 4：纯 Kappa 架构

**后果：**
- 无法重算 30 天历史数据（Kafka 只保留 7 天）
- 分析师无法做复杂 ad-hoc 查询（Flink 不支持多表 JOIN）
- 无校准基准 → 实时数据准确性无法验证

### 陷阱 5：Flink 任务不设 Checkpoint

**后果：** 任务故障后从最新 offset 恢复 → 丢失故障期间的所有数据 → 实时层缺数据。

**解决方案：** Checkpoint 间隔 30 秒，保留最近 3 个 Checkpoint，故障后从最近 Checkpoint 恢复。

## 延伸思考

- **Hudi/Iceberg**：在 Hive 上支持近实时更新（5-10 分钟），缩小实时与离线的差距。长远可替代 Lambda 架构。
- **Flink CDC 2.0**：无锁读取 + 精确一次语义，替代 Debezium + Sqoop 双管道 → 简化架构。
- **指标平台进阶**：自动检测口径变更影响范围（"修改 status 过滤条件会影响 12 个下游指标"），变更审批流。