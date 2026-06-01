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

### 实时数仓分层架构：ODS → DWD → DWS → ADS

数仓分层是保障数据质量和可复用性的基石。实时数仓同样需要分层，而非直接从 CDC 消息跳到 ADS 应用层。

```
┌──────────────────────────────────────────────────────────────────────┐
│  ADS（Application Data Service）应用数据层                           │
│  面向业务场景的宽表/指标表，直接服务 BI 看板和运营系统                  │
│  存储：ClickHouse SummingMergeTree / AggregatingMergeTree            │
├──────────────────────────────────────────────────────────────────────┤
│  DWS（Data Warehouse Summary）汇总数据层                             │
│  按维度聚合的轻度汇总表，1 分钟粒度，支持多维度下钻                     │
│  存储：ClickHouse AggregatingMergeTree                               │
├──────────────────────────────────────────────────────────────────────┤
│  DWD（Data Warehouse Detail）明细数据层                               │
│  清洗后的标准明细数据，统一字段命名、类型、编码，关联维度               │
│  存储：ClickHouse ReplacingMergeTree（支持 CDC 更新去重）             │
├──────────────────────────────────────────────────────────────────────┤
│  ODS（Operational Data Store）操作数据层                              │
│  原始业务数据 1:1 映射，不做任何清洗和转换，保留全量字段               │
│  存储：Kafka Topic（实时） + Hive 分区表（离线）                      │
├──────────────────────────────────────────────────────────────────────┤
│  数据源：MySQL（CDC） + Kafka（事件流） + 日志文件                     │
└──────────────────────────────────────────────────────────────────────┘
```

**ODS 层——原始数据落库（Kafka Topic + Hive 分区表）：**

```sql
-- ODS 层 Kafka Source：与 MySQL 表结构一一对应，不做任何转换
-- 订单表 ODS
CREATE TABLE ods_kafka_orders (
    order_id STRING,
    user_id STRING,
    amount DECIMAL(10,2),
    discount_amount DECIMAL(10,2),
    actual_amount DECIMAL(10,2),
    category STRING,
    status STRING,
    province STRING,
    created_at TIMESTAMP(3),
    updated_at TIMESTAMP(3),
    op STRING METADATA FROM 'value.payload.op',
    source_ts TIMESTAMP(3) METADATA FROM 'value.payload.source.ts_ms',
    WATERMARK FOR updated_at AS updated_at - INTERVAL '5' MINUTE
) WITH (
    'connector' = 'kafka',
    'topic' = 'ods.order_db.orders',
    'properties.bootstrap.servers' = 'kafka:9092',
    'properties.group.id' = 'flink-ods-orders',
    'format' = 'debezium-json',
    'scan.startup.mode' = 'latest-offset',
    'debezium-json.schema-include' = 'false'
);

-- 订单明细表 ODS
CREATE TABLE ods_kafka_order_items (
    item_id STRING,
    order_id STRING,
    product_id STRING,
    product_name STRING,
    quantity INT,
    unit_price DECIMAL(10,2),
    category STRING,
    created_at TIMESTAMP(3),
    op STRING METADATA FROM 'value.payload.op',
    WATERMARK FOR created_at AS created_at - INTERVAL '5' MINUTE
) WITH (
    'connector' = 'kafka',
    'topic' = 'ods.order_db.order_items',
    'properties.bootstrap.servers' = 'kafka:9092',
    'properties.group.id' = 'flink-ods-order-items',
    'format' = 'debezium-json',
    'scan.startup.mode' = 'latest-offset',
    'debezium-json.schema-include' = 'false'
);

-- 支付表 ODS
CREATE TABLE ods_kafka_payments (
    payment_id STRING,
    order_id STRING,
    pay_method STRING,
    pay_amount DECIMAL(10,2),
    pay_status STRING,
    paid_at TIMESTAMP(3),
    op STRING METADATA FROM 'value.payload.op',
    WATERMARK FOR paid_at AS paid_at - INTERVAL '5' MINUTE
) WITH (
    'connector' = 'kafka',
    'topic' = 'ods.payment_db.payments',
    'properties.bootstrap.servers' = 'kafka:9092',
    'properties.group.id' = 'flink-ods-payments',
    'format' = 'debezium-json',
    'scan.startup.mode' = 'latest-offset',
    'debezium-json.schema-include' = 'false'
);

-- ODS 层 Hive 表（离线，T+1 覆盖写入）
CREATE TABLE IF NOT EXISTS ods_hive_orders (
    order_id STRING,
    user_id STRING,
    amount DECIMAL(10,2),
    discount_amount DECIMAL(10,2),
    actual_amount DECIMAL(10,2),
    category STRING,
    status STRING,
    province STRING,
    created_at STRING,
    updated_at STRING
)
PARTITIONED BY (dt STRING)
STORED AS ORC
TBLPROPERTIES ('orc.compress' = 'SNAPPY');
```

**DWD 层——明细数据清洗与标准化：**

```sql
-- ClickHouse DWD 层：订单明细表（ReplacingMergeTree 支持 CDC 更新去重）
CREATE TABLE dwd_order_detail (
    order_id String,
    user_id String,
    amount Decimal(15,2),
    discount_amount Decimal(15,2),
    actual_amount Decimal(15,2),
    category LowCardinality(String),
    status LowCardinality(String),
    province LowCardinality(String),
    pay_method LowCardinality(String) DEFAULT '',
    created_at DateTime,
    updated_at DateTime,
    dt Date MATERIALIZED toDate(updated_at),
    sign Int8 MATERIALIZED 1,          -- ReplacingMergeTree 签名列
    _version UInt64 MATERIALIZED UUIDStringToNum(generateUUIDv4())  -- 版本号
) ENGINE = ReplacingMergeTree(_version)
ORDER BY (dt, order_id)
PARTITION BY toYYYYMM(dt)
TTL dt + INTERVAL 90 DAY;

-- ClickHouse DWD 层：订单商品明细表
CREATE TABLE dwd_order_item_detail (
    item_id String,
    order_id String,
    product_id String,
    product_name String,
    quantity UInt32,
    unit_price Decimal(10,2),
    category LowCardinality(String),
    created_at DateTime,
    dt Date MATERIALIZED toDate(created_at),
    _version UInt64 MATERIALIZED UUIDStringToNum(generateUUIDv4())
) ENGINE = ReplacingMergeTree(_version)
ORDER BY (dt, order_id, item_id)
PARTITION BY toYYYYMM(dt)
TTL dt + INTERVAL 90 DAY;

-- Flink DWD 清洗逻辑：统一字段命名、类型转换、脏数据过滤
INSERT INTO clickhouse_dwd_order_detail
SELECT
    order_id,
    user_id,
    amount,
    discount_amount,
    actual_amount,
    COALESCE(category, 'unknown') AS category,      -- 空值填充
    UPPER(status) AS status,                          -- 统一大写
    COALESCE(province, 'unknown') AS province,
    pay_method,
    created_at,
    updated_at
FROM ods_kafka_orders
WHERE op IN ('c', 'u', 'r')                          -- 过滤 DELETE 事件
  AND order_id IS NOT NULL                            -- 过滤空主键
  AND amount > 0                                      -- 过滤金额异常
  AND actual_amount >= 0;                             -- 实付不能为负
```

**DWS 层——轻度汇总（1 分钟粒度，多维度聚合）：**

```sql
-- ClickHouse DWS 层：品类维度 1 分钟汇总表
CREATE TABLE dws_category_gmv_1min (
    bucket DateTime,
    category LowCardinality(String),
    province LowCardinality(String),
    gmv AggregateFunction(sum, Decimal(15,2)),
    discount_total AggregateFunction(sum, Decimal(15,2)),
    order_count AggregateFunction(count),
    buyer_count AggregateFunction(uniq, String)      -- ClickHouse uniq 近似去重
) ENGINE = AggregatingMergeTree()
ORDER BY (bucket, category, province)
PARTITION BY toYYYYMMDD(bucket)
TTL bucket + INTERVAL 7 DAY;

-- ClickHouse DWS 层：省份维度 1 分钟汇总表
CREATE TABLE dws_province_gmv_1min (
    bucket DateTime,
    province LowCardinality(String),
    gmv AggregateFunction(sum, Decimal(15,2)),
    order_count AggregateFunction(count),
    buyer_count AggregateFunction(uniq, String)
) ENGINE = AggregatingMergeTree()
ORDER BY (bucket, province)
PARTITION BY toYYYYMMDD(bucket)
TTL bucket + INTERVAL 7 DAY;

-- Flink DWS 写入逻辑（1 分钟滚动窗口，多维度同时产出）
INSERT INTO clickhouse_dws_category_gmv_1min
SELECT
    TUMBLE_START(updated_at, INTERVAL '1' MINUTE) AS bucket,
    category,
    province,
    SUM(actual_amount) AS gmv,
    SUM(discount_amount) AS discount_total,
    COUNT(DISTINCT order_id) AS order_count,
    COUNT(DISTINCT user_id) AS buyer_count
FROM dwd_order_detail_stream  -- DWD 清洗后的流
WHERE status = 'PAID'
GROUP BY
    TUMBLE(updated_at, INTERVAL '1' MINUTE),
    category,
    province;
```

**ADS 层——面向业务应用（直接服务 BI 看板）：**

```sql
-- ADS 层：实时 GMV 看板表（品类 × 省份 × 分钟粒度）
CREATE TABLE ads_realtime_gmv_dashboard (
    bucket DateTime,
    category LowCardinality(String),
    province LowCardinality(String),
    gmv Decimal(15,2),
    discount_total Decimal(15,2),
    order_count UInt64,
    buyer_count UInt64,
    computed_at DateTime DEFAULT now()
) ENGINE = SummingMergeTree()
ORDER BY (bucket, category, province)
PARTITION BY toYYYYMMDD(bucket)
TTL bucket + INTERVAL 7 DAY;

-- ADS 层：实时转化率表
CREATE TABLE ads_realtime_conversion (
    bucket DateTime,
    category LowCardinality(String),
    exposure_count UInt64,          -- 曝光次数
    click_count UInt64,             -- 点击次数
    cart_count UInt64,              -- 加购次数
    order_count UInt64,             -- 下单次数
    paid_count UInt64,              -- 付款次数
    click_rate Float64 MATERIALIZED click_count / nullIf(exposure_count, 0),
    cart_rate Float64 MATERIALIZED cart_count / nullIf(click_count, 0),
    order_rate Float64 MATERIALIZED order_count / nullIf(click_count, 0),
    pay_rate Float64 MATERIALIZED paid_count / nullIf(order_count, 0)
) ENGINE = SummingMergeTree()
ORDER BY (bucket, category)
PARTITION BY toYYYYMMDD(bucket)
TTL bucket + INTERVAL 7 DAY;

-- ADS 层：大促实时排行表（TOP-N 品类/商品）
CREATE TABLE ads_realtime_topn (
    bucket DateTime,
    rank_type LowCardinality(String),   -- 'category' / 'product'
    rank_key String,                     -- 品类名或商品ID
    metric_value Decimal(15,2),
    rank_num UInt32,
    computed_at DateTime DEFAULT now()
) ENGINE = ReplacingMergeTree()
ORDER BY (bucket, rank_type, rank_num)
PARTITION BY toYYYYMMDD(bucket)
TTL bucket + INTERVAL 3 DAY;

-- 业务查询示例：当前各品类 GMV TOP 10（5 分钟内）
SELECT category, SUM(gmv) AS total_gmv, SUM(order_count) AS total_orders
FROM ads_realtime_gmv_dashboard
WHERE bucket >= now() - INTERVAL 5 MINUTE
GROUP BY category
ORDER BY total_gmv DESC
LIMIT 10;
```

**分层的数据流向与依赖关系：**

```
MySQL CDC ──→ ODS Kafka Topic ──→ Flink 清洗 ──→ DWD ClickHouse ──→ Flink 聚合 ──→ DWS ClickHouse
                                                                                          │
                                                                                   物化视图 / 定时刷新
                                                                                          ↓
                                                                                    ADS ClickHouse ──→ BI 看板
                                                                                          ↑
Hive T+1 ────────────────────────────────────────────────────────────────────── 口径校准
```

**完整分层 Schema 定义——电商场景全部实体：**

以下补充商品表和用户表的完整四层定义。上文已展示订单和支付相关表，此处补齐完整数仓分层的所有核心实体。

```sql
-- ============================================================
-- ODS 层：用户表原始数据（Kafka Source）
-- ============================================================
CREATE TABLE ods_kafka_users (
    user_id STRING,
    username STRING,
    phone STRING,
    gender STRING,
    age INT,
    province STRING,
    city STRING,
    register_channel STRING,
    vip_level INT,
    created_at TIMESTAMP(3),
    updated_at TIMESTAMP(3),
    op STRING METADATA FROM 'value.payload.op',
    WATERMARK FOR updated_at AS updated_at - INTERVAL '5' MINUTE
) WITH (
    'connector' = 'kafka',
    'topic' = 'ods.user_db.users',
    'properties.bootstrap.servers' = 'kafka:9092',
    'properties.group.id' = 'flink-ods-users',
    'format' = 'debezium-json',
    'scan.startup.mode' = 'latest-offset',
    'debezium-json.schema-include' = 'false'
);

-- ODS 层：商品表原始数据（Kafka Source）
CREATE TABLE ods_kafka_products (
    product_id STRING,
    product_name STRING,
    category_id STRING,
    category_name STRING,
    brand STRING,
    price DECIMAL(10,2),
    stock INT,
    status STRING,
    created_at TIMESTAMP(3),
    updated_at TIMESTAMP(3),
    op STRING METADATA FROM 'value.payload.op',
    WATERMARK FOR updated_at AS updated_at - INTERVAL '5' MINUTE
) WITH (
    'connector' = 'kafka',
    'topic' = 'ods.product_db.products',
    'properties.bootstrap.servers' = 'kafka:9092',
    'properties.group.id' = 'flink-ods-products',
    'format' = 'debezium-json',
    'scan.startup.mode' = 'latest-offset',
    'debezium-json.schema-include' = 'false'
);

-- ODS 层：用户行为事件流（Kafka Source，非CDC，App埋点上报）
CREATE TABLE ods_kafka_user_events (
    event_id STRING,
    user_id STRING,
    event_type STRING,        -- 'exposure' / 'click' / 'search' / 'add_cart' / 'order' / 'pay'
    product_id STRING,
    category STRING,
    search_keyword STRING,
    source_page STRING,
    event_time TIMESTAMP(3),
    device_id STRING,
    platform STRING,          -- 'ios' / 'android' / 'web'
    WATERMARK FOR event_time AS event_time - INTERVAL '3' MINUTE
) WITH (
    'connector' = 'kafka',
    'topic' = 'ods.event.user_behavior',
    'properties.bootstrap.servers' = 'kafka:9092',
    'properties.group.id' = 'flink-ods-user-events',
    'format' = 'json',
    'scan.startup.mode' = 'latest-offset'
);

-- ODS 层 Hive 表：用户（离线，T+1 覆盖写入）
CREATE TABLE IF NOT EXISTS ods_hive_users (
    user_id STRING,
    username STRING,
    phone STRING,
    gender STRING,
    age INT,
    province STRING,
    city STRING,
    register_channel STRING,
    vip_level INT,
    created_at STRING,
    updated_at STRING
)
PARTITIONED BY (dt STRING)
STORED AS ORC
TBLPROPERTIES ('orc.compress' = 'SNAPPY');

-- ODS 层 Hive 表：商品（离线，T+1 覆盖写入，含全量快照）
CREATE TABLE IF NOT EXISTS ods_hive_products (
    product_id STRING,
    product_name STRING,
    category_id STRING,
    category_name STRING,
    brand STRING,
    price DECIMAL(10,2),
    stock INT,
    status STRING,
    created_at STRING,
    updated_at STRING
)
PARTITIONED BY (dt STRING)
STORED AS ORC
TBLPROPERTIES ('orc.compress' = 'SNAPPY');
```

```sql
-- ============================================================
-- DWD 层：用户维度表（ClickHouse ReplacingMergeTree，缓慢变化维度）
-- ============================================================
CREATE TABLE dwd_user_dim (
    user_id String,
    username String,
    gender LowCardinality(String),
    age UInt8,
    province LowCardinality(String),
    city LowCardinality(String),
    register_channel LowCardinality(String),
    vip_level UInt8,
    created_at DateTime,
    updated_at DateTime,
    dt Date MATERIALIZED toDate(updated_at),
    _version UInt64 MATERIALIZED UUIDStringToNum(generateUUIDv4())
) ENGINE = ReplacingMergeTree(_version)
ORDER BY (dt, user_id)
PARTITION BY toYYYYMM(dt)
TTL dt + INTERVAL 180 DAY;

-- DWD 层：商品维度表（ClickHouse ReplacingMergeTree）
CREATE TABLE dwd_product_dim (
    product_id String,
    product_name String,
    category_id String,
    category_name LowCardinality(String),
    brand LowCardinality(String),
    price Decimal(10,2),
    stock Int32,
    status LowCardinality(String),
    created_at DateTime,
    updated_at DateTime,
    dt Date MATERIALIZED toDate(updated_at),
    _version UInt64 MATERIALIZED UUIDStringToNum(generateUUIDv4())
) ENGINE = ReplacingMergeTree(_version)
ORDER BY (dt, product_id)
PARTITION BY toYYYYMM(dt)
TTL dt + INTERVAL 180 DAY;

-- DWD 层：用户行为明细表（ClickHouse，AppendOnly，事件流无需更新）
CREATE TABLE dwd_user_event_detail (
    event_id String,
    user_id String,
    event_type LowCardinality(String),
    product_id String,
    category LowCardinality(String),
    search_keyword String,
    source_page LowCardinality(String),
    event_time DateTime,
    device_id String,
    platform LowCardinality(String),
    dt Date MATERIALIZED toDate(event_time),
    hour UInt8 MATERIALIZED toHour(event_time)
) ENGINE = MergeTree()
ORDER BY (dt, hour, event_type, user_id)
PARTITION BY toYYYYMMDD(event_time)
TTL dt + INTERVAL 30 DAY;

-- Flink DWD 清洗逻辑：用户数据清洗
INSERT INTO clickhouse_dwd_user_dim
SELECT
    user_id,
    username,
    COALESCE(gender, 'unknown') AS gender,
    CASE WHEN age < 0 OR age > 150 THEN 0 ELSE age END AS age,  -- 异常年龄修正
    COALESCE(province, 'unknown') AS province,
    COALESCE(city, 'unknown') AS city,
    COALESCE(register_channel, 'organic') AS register_channel,
    COALESCE(vip_level, 0) AS vip_level,
    created_at,
    updated_at
FROM ods_kafka_users
WHERE op IN ('c', 'u', 'r')
  AND user_id IS NOT NULL
  AND username IS NOT NULL;

-- Flink DWD 清洗逻辑：用户行为事件清洗
INSERT INTO clickhouse_dwd_user_event_detail
SELECT
    event_id,
    user_id,
    UPPER(event_type) AS event_type,            -- 统一大写
    product_id,
    COALESCE(category, 'unknown') AS category,
    search_keyword,
    COALESCE(source_page, 'direct') AS source_page,
    event_time,
    device_id,
    LOWER(platform) AS platform,                -- 统一小写
FROM ods_kafka_user_events
WHERE event_id IS NOT NULL
  AND user_id IS NOT NULL
  AND event_type IN ('exposure', 'click', 'search', 'add_cart', 'order', 'pay');
```

```sql
-- ============================================================
-- DWS 层：用户行为转化漏斗 1 分钟汇总（曝光→点击→加购→下单→支付）
-- ============================================================
CREATE TABLE dws_funnel_1min (
    bucket DateTime,
    category LowCardinality(String),
    platform LowCardinality(String),
    exposure_count UInt64,
    click_count UInt64,
    cart_count UInt64,
    order_count UInt64,
    paid_count UInt64
) ENGINE = SummingMergeTree()
ORDER BY (bucket, category, platform)
PARTITION BY toYYYYMMDD(bucket)
TTL bucket + INTERVAL 7 DAY;

-- DWS 层：用户维度 1 分钟汇总（新注册用户数、活跃用户数）
CREATE TABLE dws_user_metric_1min (
    bucket DateTime,
    province LowCardinality(String),
    register_channel LowCardinality(String),
    new_user_count UInt64,
    active_user_count UInt64
) ENGINE = SummingMergeTree()
ORDER BY (bucket, province, register_channel)
PARTITION BY toYYYYMMDD(bucket)
TTL bucket + INTERVAL 7 DAY;

-- DWS 层：商品维度 1 分钟汇总（销量、库存变动）
CREATE TABLE dws_product_metric_1min (
    bucket DateTime,
    category LowCardinality(String),
    brand LowCardinality(String),
    sold_quantity UInt64,
    sold_amount Decimal(15,2),
    avg_price Decimal(10,2)
) ENGINE = SummingMergeTree()
ORDER BY (bucket, category, brand)
PARTITION BY toYYYYMMDD(bucket)
TTL bucket + INTERVAL 7 DAY;

-- Flink DWS：转化漏斗聚合（1 分钟滚动窗口）
INSERT INTO clickhouse_dws_funnel_1min
SELECT
    TUMBLE_START(event_time, INTERVAL '1' MINUTE) AS bucket,
    category,
    platform,
    SUM(CASE WHEN event_type = 'EXPOSURE' THEN 1 ELSE 0 END) AS exposure_count,
    SUM(CASE WHEN event_type = 'CLICK' THEN 1 ELSE 0 END) AS click_count,
    SUM(CASE WHEN event_type = 'ADD_CART' THEN 1 ELSE 0 END) AS cart_count,
    SUM(CASE WHEN event_type = 'ORDER' THEN 1 ELSE 0 END) AS order_count,
    SUM(CASE WHEN event_type = 'PAY' THEN 1 ELSE 0 END) AS paid_count
FROM dwd_user_event_detail_stream
GROUP BY
    TUMBLE(event_time, INTERVAL '1' MINUTE),
    category,
    platform;

-- Flink DWS：用户指标聚合
INSERT INTO clickhouse_dws_user_metric_1min
SELECT
    TUMBLE_START(updated_at, INTERVAL '1' MINUTE) AS bucket,
    province,
    register_channel,
    SUM(CASE WHEN op = 'c' THEN 1 ELSE 0 END) AS new_user_count,  -- 新注册
    COUNT(DISTINCT user_id) AS active_user_count                    -- 活跃用户
FROM dwd_user_dim_stream
GROUP BY
    TUMBLE(updated_at, INTERVAL '1' MINUTE),
    province,
    register_channel;
```

```sql
-- ============================================================
-- ADS 层：用户画像宽表（维度关联后的应用表，服务用户运营系统）
-- ============================================================
CREATE TABLE ads_user_profile (
    user_id String,
    username String,
    gender LowCardinality(String),
    age UInt8,
    province LowCardinality(String),
    city LowCardinality(String),
    vip_level UInt8,
    register_channel LowCardinality(String),
    total_orders UInt64 DEFAULT 0,
    total_gmv Decimal(15,2) DEFAULT 0,
    last_order_at DateTime DEFAULT toDateTime('1970-01-01'),
    last_login_at DateTime DEFAULT toDateTime('1970-01-01'),
    updated_at DateTime DEFAULT now()
) ENGINE = ReplacingMergeTree(updated_at)
ORDER BY (user_id)
PARTITION BY tuple()  -- 用户画像不做分区，全量存储
TTL updated_at + INTERVAL 365 DAY;

-- ADS 层：商品实时销量排行（TOP-N，1 分钟刷新）
CREATE TABLE ads_product_sales_rank (
    bucket DateTime,
    category LowCardinality(String),
    product_id String,
    product_name String,
    sold_quantity UInt64,
    sold_amount Decimal(15,2),
    rank_num UInt32,
    computed_at DateTime DEFAULT now()
) ENGINE = ReplacingMergeTree()
ORDER BY (bucket, category, rank_num)
PARTITION BY toYYYYMMDD(bucket)
TTL bucket + INTERVAL 3 DAY;

-- ADS 层：实时大屏总览表（服务运营大屏，全局汇总）
CREATE TABLE ads_realtime_overview (
    bucket DateTime,
    total_gmv Decimal(15,2),
    total_orders UInt64,
    total_buyers UInt64,
    avg_order_amount Decimal(10,2),
    paid_rate Float64,
    computed_at DateTime DEFAULT now()
) ENGINE = ReplacingMergeTree()
ORDER BY (bucket)
PARTITION BY toYYYYMMDD(bucket)
TTL bucket + INTERVAL 7 DAY;

-- ADS 层业务查询示例：实时大屏——今日总览
SELECT
    SUM(total_gmv) AS today_gmv,
    SUM(total_orders) AS today_orders,
    SUM(total_buyers) AS today_buyers,
    SUM(total_gmv) / nullIf(SUM(total_orders), 0) AS avg_order_amount
FROM ads_realtime_overview
WHERE bucket >= today();

-- ADS 层业务查询示例：用户画像——高价值用户
SELECT user_id, username, total_gmv, total_orders, vip_level
FROM ads_user_profile
WHERE total_gmv > 10000
  AND vip_level >= 3
ORDER BY total_gmv DESC
LIMIT 100;

-- ADS 层业务查询示例：品类转化漏斗（最近 1 小时）
SELECT
    category,
    SUM(exposure_count) AS exposures,
    SUM(click_count) AS clicks,
    SUM(cart_count) AS carts,
    SUM(order_count) AS orders,
    SUM(paid_count) AS paids,
    SUM(click_count) / nullIf(SUM(exposure_count), 0) AS click_rate,
    SUM(paid_count) / nullIf(SUM(order_count), 0) AS pay_rate
FROM dws_funnel_1min
WHERE bucket >= now() - INTERVAL 1 HOUR
GROUP BY category
ORDER BY SUM(paid_count) DESC;
```

**各层表汇总关系图：**

```
ODS 层（原始层，Kafka Topic + Hive 分区表）
├── ods_kafka_orders          ──→  ods_hive_orders
├── ods_kafka_order_items     ──→  ods_hive_order_items
├── ods_kafka_payments        ──→  ods_hive_payments
├── ods_kafka_users           ──→  ods_hive_users
├── ods_kafka_products        ──→  ods_hive_products
└── ods_kafka_user_events     （无离线，纯实时事件流）

DWD 层（明细层，ClickHouse ReplacingMergeTree / MergeTree）
├── dwd_order_detail          ←  ods_kafka_orders + ods_kafka_payments（JOIN 补充支付方式）
├── dwd_order_item_detail     ←  ods_kafka_order_items
├── dwd_user_dim              ←  ods_kafka_users
├── dwd_product_dim           ←  ods_kafka_products
└── dwd_user_event_detail     ←  ods_kafka_user_events

DWS 层（汇总层，ClickHouse AggregatingMergeTree / SummingMergeTree）
├── dws_category_gmv_1min     ←  dwd_order_detail（品类 × 省份 × 1 分钟）
├── dws_province_gmv_1min     ←  dwd_order_detail（省份 × 1 分钟）
├── dws_funnel_1min           ←  dwd_user_event_detail（品类 × 平台 × 1 分钟漏斗）
├── dws_user_metric_1min      ←  dwd_user_dim（省份 × 渠道 × 1 分钟新登/活跃）
└── dws_product_metric_1min   ←  dwd_order_item_detail + dwd_product_dim（品类 × 品牌 × 1 分钟销量）

ADS 层（应用层，ClickHouse SummingMergeTree / ReplacingMergeTree）
├── ads_realtime_gmv_dashboard ←  dws_category_gmv_1min（品类 × 省份 × 分钟 GMV）
├── ads_realtime_conversion    ←  dws_funnel_1min（品类 × 分钟转化率）
├── ads_realtime_topn          ←  dws_category_gmv_1min（TOP-N 品类/商品排行）
├── ads_realtime_overview      ←  dws_category_gmv_1min（全局总览，服务大屏）
├── ads_user_profile           ←  dwd_user_dim + dwd_order_detail（用户画像宽表）
└── ads_product_sales_rank     ←  dws_product_metric_1min（商品销量排行）
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

**Kafka Topic 规划与容量估算：**

```yaml
# Kafka Topic 详细规划
topics:
  # 订单 CDC Topic
  - name: ods.order_db.orders
    partitions: 12              # 10 个分库 + 2 个冗余，保证分库分表均匀映射
    replication_factor: 3
    retention_ms: 604800000     # 7 天保留
    cleanup_policy: delete
    segment_bytes: 1073741824   # 1GB segment
    # 预估吞吐：日均 100 万订单 × 平均 1KB/消息 = 1GB/天
    # 峰值：大促 ×10 = 10GB/天，峰值写入约 1.2MB/s
    # 12 分区 → 每分区约 100KB/s，远低于 Kafka 单分区上限

  # 订单明细 CDC Topic
  - name: ods.order_db.order_items
    partitions: 24              # 订单明细量是订单的 3-5 倍
    replication_factor: 3
    retention_ms: 604800000
    # 预估吞吐：日均 400 万条 × 0.8KB = 3.2GB/天
    # 峰值：32GB/天，24 分区每分区约 150KB/s

  # 支付 CDC Topic
  - name: ods.payment_db.payments
    partitions: 12
    replication_factor: 3
    retention_ms: 604800000

  # 用户 CDC Topic
  - name: ods.user_db.users
    partitions: 6               # 用户变更频率较低
    replication_factor: 3
    retention_ms: 604800000

  # 商品 CDC Topic
  - name: ods.product_db.products
    partitions: 6
    replication_factor: 3
    retention_ms: 604800000

  # 用户行为事件 Topic（非CDC，App 埋点直写 Kafka）
  - name: ods.event.user_behavior
    partitions: 48              # 日均 5 亿条事件，这是最大吞吐 Topic
    replication_factor: 3
    retention_ms: 259200000     # 3 天保留（事件数据量大，缩短保留期）
    # 预估吞吐：5 亿条 × 0.5KB = 250GB/天
    # 峰值：2.5TB/天，48 分区每分区约 600KB/s
    # 消费端 Flink 并行度建议 ≥ 48
```

**Kafka 集群容量规划：**

```
集群配置：5 Broker 节点，每节点 16 核 64GB 内存，4 块 2TB SSD

容量分析：
  总日志写入量（日常）：~260GB/天
  总日志写入量（大促峰值 ×10）：~2.6TB/天
  副本系数 3：实际存储 = 260 × 3 × 7 天保留 ≈ 5.5TB（日常）
  峰值写入吞吐：2.6TB / 86400s ≈ 30MB/s（5 Broker 均摊 6MB/s，远低于磁盘上限）
  Broker 网络：万兆网卡 10Gbps，可支撑 ~1GB/s 吞吐 → 裕量充足

关键调优参数：
  broker:
    num.io.threads: 16
    num.network.threads: 8
    socket.send.buffer.bytes: 1048576
    socket.receive.buffer.bytes: 1048576
    log.compaction: 不启用（CDC Topic 用 delete 策略）
    auto.create.topics.enable: false  # 禁止自动创建 Topic
```

**完整 CDC 管道：Flink Java 实现（精确一次语义）：**

```java
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.CheckpointingMode;
import org.apache.flink.connector.kafka.source.KafkaSource;
import org.apache.flink.connector.kafka.source.enumerator.initializer.OffsetsInitializer;
import org.apache.flink.connector.kafka.sink.KafkaSink;
import org.apache.flink.connector.jdbc.JdbcConnectionOptions;
import org.apache.flink.connector.jdbc.JdbcExecutionOptions;
import org.apache.flink.connector.jdbc.JdbcSink;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.functions.ProcessFunction;
import org.apache.flink.util.Collector;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.time.Duration;
import java.time.Instant;

/**
 * 完整 CDC 管道：Kafka (Debezium JSON) → Flink 清洗转换 → ClickHouse Sink
 * 精确一次语义保障：Kafka Source + Flink Checkpoint + ClickHouse 事务写入
 */
public class OrderCdcPipeline {

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // ============================================================
        // 1. 精确一次语义配置（核心）
        // ============================================================
        env.enableCheckpointing(30000);  // 30 秒一次 Checkpoint
        env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
        env.getCheckpointConfig().setMinPauseBetweenCheckpoints(10000);  // 两次 Checkpoint 至少间隔 10 秒
        env.getCheckpointConfig().setCheckpointTimeout(60000);           // 超时 60 秒
        env.getCheckpointConfig().setTolerableCheckpointFailureNumber(3); // 允许 3 次 Checkpoint 失败
        env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);        // 最多 1 个并发 Checkpoint

        // Checkpoint 存储：HDFS（生产环境推荐）
        env.getCheckpointConfig().setCheckpointStorage("hdfs://namenode:8020/flink/checkpoints");

        // ============================================================
        // 2. Kafka Source（消费 Debezium CDC 消息）
        // ============================================================
        KafkaSource<String> orderSource = KafkaSource.<String>builder()
            .setBootstrapServers("kafka:9092")
            .setTopics("ods.order_db.orders")
            .setGroupId("flink-cdc-orders")
            .setStartingOffsets(OffsetsInitializer.latest())
            .setValueOnlyDeserializer(new SimpleStringSchema())
            .setProperty("partition.discovery.interval.ms", "30000")  // 动态分区发现
            .setProperty("commit.offsets.on.checkpoint", "true")       // Checkpoint 时提交 offset
            .build();

        // Watermark 策略：允许 5 分钟迟到
        WatermarkStrategy<String> watermarkStrategy = WatermarkStrategy
            .<String>forBoundedOutOfOrderness(Duration.ofMinutes(5))
            .withTimestampAssigner((event, timestamp) -> {
                try {
                    JsonNode node = new ObjectMapper().readTree(event);
                    return node.at("/payload/after/updated_at").asLong();
                } catch (Exception e) {
                    return System.currentTimeMillis();
                }
            })
            .withIdleness(Duration.ofMinutes(1));  // 空闲分区标记，防止 Watermark 卡住

        DataStream<String> orderStream = env
            .fromSource(orderSource, watermarkStrategy, "Kafka-Order-Source")
            .setParallelism(12);  // 与 Kafka 分区数一致

        // ============================================================
        // 3. CDC 消息解析与清洗（DWD 层逻辑）
        // ============================================================
        DataStream<OrderDetail> cleanedStream = orderStream
            .process(new CdcParseAndCleanFunction())
            .setParallelism(12)
            .name("CDC-Parse-Clean");

        // ============================================================
        // 4. ClickHouse Sink（精确一次写入）
        // ============================================================
        // ClickHouse 不支持 XA 事务，使用"幂等写入"实现精确一次：
        //   每条记录带 _version 字段（来自 CDC 的 source.ts_ms + row 位点）
        //   ReplacingMergeTree 按 _version 去重 → 重复写入不会产生副作用
        JdbcConnectionOptions clickhouseOptions = new JdbcConnectionOptions.JdbcConnectionOptionsBuilder()
            .withUrl("jdbc:clickhouse://clickhouse-1:8123/data_warehouse")
            .withDriverName("com.clickhouse.jdbc.ClickHouseDriver")
            .withUsername("flink_writer")
            .withPassword("${CLICKHOUSE_PASSWORD}")
            .build();

        cleanedStream.addSink(JdbcSink.sink(
            "INSERT INTO dwd_order_detail " +
            "(order_id, user_id, amount, discount_amount, actual_amount, category, " +
            " status, province, pay_method, created_at, updated_at, _version) " +
            "VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)",
            (stmt, order) -> {
                stmt.setString(1, order.orderId);
                stmt.setString(2, order.userId);
                stmt.setBigDecimal(3, order.amount);
                stmt.setBigDecimal(4, order.discountAmount);
                stmt.setBigDecimal(5, order.actualAmount);
                stmt.setString(6, order.category);
                stmt.setString(7, order.status);
                stmt.setString(8, order.province);
                stmt.setString(9, order.payMethod);
                stmt.setTimestamp(10, order.createdAt);
                stmt.setTimestamp(11, order.updatedAt);
                stmt.setLong(12, order.version);  // 幂等去重键
            },
            JdbcExecutionOptions.builder()
                .withBatchSize(5000)                  // 每批 5000 条
                .withBatchIntervalMs(1000)            // 最多等 1 秒
                .withMaxRetries(3)                    // 失败重试 3 次
                .build(),
            clickhouseOptions
        )).setParallelism(6).name("ClickHouse-DWD-Sink");

        env.execute("Order-CDC-Pipeline");
    }

    /**
     * CDC 消息解析与清洗函数（DWD 层逻辑）
     */
    static class CdcParseAndCleanFunction extends ProcessFunction<String, OrderDetail> {
        private final ObjectMapper mapper = new ObjectMapper();

        @Override
        public void processElement(String value, Context ctx, Collector<OrderDetail> out) {
            try {
                JsonNode root = mapper.readTree(value);
                JsonNode payload = root.at("/payload");
                String op = payload.at("/op").asText();

                // 过滤 DELETE 事件（只处理 insert 和 update）
                if ("d".equals(op)) return;

                JsonNode after = payload.at("/after");
                JsonNode source = payload.at("/source");

                String orderId = after.at("/order_id").asText("");
                String userId = after.at("/user_id").asText("");

                // DWD 清洗规则
                if (orderId.isEmpty() || userId.isEmpty()) return;  // 过滤空主键

                double amount = after.at("/amount").asDouble(0);
                if (amount <= 0) return;  // 过滤金额异常

                double actualAmount = after.at("/actual_amount").asDouble(0);
                if (actualAmount < 0) return;  // 实付不能为负

                // 构造清洗后的明细记录
                OrderDetail detail = new OrderDetail();
                detail.orderId = orderId;
                detail.userId = userId;
                detail.amount = java.math.BigDecimal.valueOf(amount);
                detail.discountAmount = java.math.BigDecimal.valueOf(after.at("/discount_amount").asDouble(0));
                detail.actualAmount = java.math.BigDecimal.valueOf(actualAmount);
                detail.category = nullToDefault(after.at("/category").asText(null), "unknown");
                detail.status = after.at("/status").asText("").toUpperCase();  // 统一大写
                detail.province = nullToDefault(after.at("/province").asText(null), "unknown");
                detail.payMethod = nullToDefault(after.at("/pay_method").asText(null), "");
                detail.createdAt = new java.sql.Timestamp(after.at("/created_at").asLong(0));
                detail.updatedAt = new java.sql.Timestamp(after.at("/updated_at").asLong(0));
                // 版本号：用 CDC source 位点保证幂等
                detail.version = source.at("/ts_ms").asLong(0) * 1000 + source.at("/row").asInt(0);

                out.collect(detail);
            } catch (Exception e) {
                // 解析失败 → 记录到死信队列（而非丢弃）
                ctx.output(DEAD_TAG, value);
            }
        }

        private String nullToDefault(String val, String defaultVal) {
            return (val == null || val.isEmpty()) ? defaultVal : val;
        }
    }

    static class OrderDetail {
        String orderId, userId, category, status, province, payMethod;
        java.math.BigDecimal amount, discountAmount, actualAmount;
        java.sql.Timestamp createdAt, updatedAt;
        long version;
    }

    static final org.apache.flink.util.OutputTag<String> DEAD_TAG =
        new org.apache.flink.util.OutputTag<String>("dead-letter") {};
}
```

**精确一次语义保障机制详解：**

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    精确一次语义的三个层面保障                                  │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. Kafka Source 层：                                                         │
│     ┌─────────────┐     ┌──────────────┐     ┌──────────────┐               │
│     │ Kafka       │────→│ Flink Source │────→│ Checkpoint   │               │
│     │ Consumer    │     │ 读取消息     │     │ 记录 offset  │               │
│     └─────────────┘     └──────────────┘     └──────┬───────┘               │
│                                                    │                       │
│     offset 仅在 Checkpoint 完成后才提交到 Kafka      │                       │
│     → 故障恢复时从最近 Checkpoint 的 offset 重放     │                       │
│                                                                              │
│  2. Flink 计算层：                                                            │
│     Checkpoint Barrier 对齐 → 算子状态快照 → 保证计算过程不丢不重              │
│                                                                              │
│  3. ClickHouse Sink 层：                                                     │
│     ClickHouse 无 XA 事务支持 → 使用"幂等写入"策略：                           │
│     ┌──────────────────────────────────────────────────────────┐             │
│     │ 每条记录携带 _version 字段（CDC source.ts_ms + row 位点） │             │
│     │ ReplacingMergeTree 按 (order_id, _version) 去重          │             │
│     │ → 即使重复写入，Final 查询也只返回最新版本               │             │
│     │ → 实现语义上的精确一次                                  │             │
│     └──────────────────────────────────────────────────────────┘             │
│                                                                              │
│  容错流程：                                                                   │
│  Flink TaskManager 崩溃                                                      │
│    → JobManager 检测心跳超时                                                  │
│    → 从最近成功的 Checkpoint 恢复                                            │
│    → Source 重新消费 Checkpoint offset 之后的消息                             │
│    → Sink 重新写入（幂等，不影响结果）                                       │
│    → 整体恢复时间 = Checkpoint 加载 + 重新处理延迟 ≈ 2-5 分钟                │
└──────────────────────────────────────────────────────────────────────────────┘
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

### 数据质量监控体系

数据质量是实时数仓的生命线。离线数仓可以通过重跑修复脏数据，但实时层一旦写入错误数据，下游看板就会立即展示错误结果。因此必须在每一层之间设置质量检查关卡。

**质量监控架构：**

```
ODS → [空值检查/格式检查] → DWD → [完整性检查/一致性检查] → DWS → [阈值检查/波动检查] → ADS
                                                                    │
                                                    ┌───────────────┘
                                                    ↓
                                             告警中心（飞书/钉钉/邮件）
                                                    │
                                              自动拦截/降级
```

```python
import logging
from datetime import datetime, timedelta
from typing import Dict, List, Optional, Tuple
from dataclasses import dataclass
from enum import Enum
import statistics

logger = logging.getLogger("data_quality_monitor")


class Severity(Enum):
    CRITICAL = "critical"   # 立即告警 + 暂停下游消费
    WARNING = "warning"     # 告警 + 人工确认
    INFO = "info"           # 仅记录日志


@dataclass
class QualityCheckResult:
    check_name: str
    table_name: str
    layer: str                # ODS / DWD / DWS / ADS
    severity: Severity
    passed: bool
    metric_value: float
    threshold: float
    message: str
    timestamp: datetime


class DataQualityMonitor:
    """实时数仓数据质量监控——四层逐层检查"""

    def __init__(self, clickhouse_client, kafka_admin, flink_client, alert_sender):
        self.ch = clickhouse_client
        self.kafka = kafka_admin
        self.flink = flink_client
        self.alert = alert_sender
        self.history: Dict[str, List[float]] = {}  # 指标历史值，用于波动检测

    # ============================================================
    # 1. 空值检查（DWD 层入场关卡）
    # ============================================================
    def check_null_values(self, table: str, columns: List[str],
                          threshold_pct: float = 0.01) -> List[QualityCheckResult]:
        """
        检查指定表中关键字段的空值率。
        默认阈值 1%：空值率超过 1% 即告警。
        """
        results = []
        for col in columns:
            sql = f"""
                SELECT
                    count() AS total,
                    countIf({col} = '' OR {col} = 'unknown' OR {col} IS NULL) AS null_count,
                    null_count / total AS null_rate
                FROM {table}
                WHERE dt = today()
            """
            row = self.ch.query_one(sql)
            null_rate = row['null_rate']

            result = QualityCheckResult(
                check_name="null_check",
                table_name=table,
                layer="DWD",
                severity=Severity.CRITICAL if null_rate > threshold_pct * 10 else Severity.WARNING,
                passed=null_rate <= threshold_pct,
                metric_value=null_rate,
                threshold=threshold_pct,
                message=f"表 {table} 字段 {col} 空值率 {null_rate:.4f}，阈值 {threshold_pct}",
                timestamp=datetime.now()
            )
            results.append(result)

            if not result.passed:
                self._send_alert(result)

        return results

    # ============================================================
    # 2. Schema 变更检测（ODS → DWD 卡点）
    # ============================================================
    def check_schema_drift(self, table: str, expected_schema: Dict[str, str]) -> QualityCheckResult:
        """
        对比 ClickHouse 表的实际 schema 与预期 schema。
        检测：新增列、删除列、类型变更。
        这是 CDC 管道中"源端表结构变更"导致管道故障的预警。
        """
        actual_columns = self._get_table_schema(table)

        missing_cols = set(expected_schema.keys()) - set(actual_columns.keys())
        extra_cols = set(actual_columns.keys()) - set(expected_schema.keys())
        type_mismatch = []
        for col, expected_type in expected_schema.items():
            if col in actual_columns and actual_columns[col] != expected_type:
                type_mismatch.append(f"{col}: expected={expected_type}, actual={actual_columns[col]}")

        has_drift = bool(missing_cols or extra_cols or type_mismatch)

        message_parts = []
        if missing_cols:
            message_parts.append(f"缺失列: {missing_cols}")
        if extra_cols:
            message_parts.append(f"新增列: {extra_cols}")
        if type_mismatch:
            message_parts.append(f"类型不匹配: {type_mismatch}")

        result = QualityCheckResult(
            check_name="schema_drift",
            table_name=table,
            layer="DWD",
            severity=Severity.CRITICAL,
            passed=not has_drift,
            metric_value=float(has_drift),
            threshold=0.0,
            message=f"表 {table} schema 变更: {'; '.join(message_parts)}" if has_drift else f"表 {table} schema 正常",
            timestamp=datetime.now()
        )

        if not result.passed:
            self._send_alert(result)

        return result

    # ============================================================
    # 3. 数据量波动异常检测（DWS 层关卡）
    # ============================================================
    def check_volume_anomaly(self, table: str, window_minutes: int = 60,
                             z_score_threshold: float = 3.0) -> QualityCheckResult:
        """
        检测数据量异常波动（基于 Z-Score 统计方法）。
        对比当前窗口的数据量与过去 7 天同时段的历史数据。
        """
        # 获取过去 7 天同时段的分钟级数据量
        sql = f"""
            SELECT
                toDate(bucket) AS day,
                toHour(bucket) AS hour,
                count() AS row_count
            FROM {table}
            WHERE bucket >= now() - INTERVAL 7 DAY
              AND toHour(bucket) = toHour(now())
              AND toMinute(bucket) BETWEEN toMinute(now()) - 5 AND toMinute(now()) + 5
            GROUP BY day, hour
            ORDER BY day
        """
        rows = self.ch.query_all(sql)
        historical_counts = [r['row_count'] for r in rows]

        if len(historical_counts) < 3:
            # 历史数据不足，无法统计
            return QualityCheckResult(
                check_name="volume_anomaly",
                table_name=table, layer="DWS",
                severity=Severity.INFO,
                passed=True,
                metric_value=0.0, threshold=z_score_threshold,
                message=f"表 {table} 历史数据不足，跳过波动检测",
                timestamp=datetime.now()
            )

        # 当前窗口数据量
        current_sql = f"""
            SELECT count() AS row_count
            FROM {table}
            WHERE bucket >= now() - INTERVAL {window_minutes} MINUTE
        """
        current_count = self.ch.query_one(current_sql)['row_count']

        # Z-Score 计算
        mean_val = statistics.mean(historical_counts)
        std_val = statistics.stdev(historical_counts) if len(historical_counts) > 1 else 1.0
        z_score = abs(current_count - mean_val) / std_val if std_val > 0 else 0.0

        is_anomaly = z_score > z_score_threshold

        result = QualityCheckResult(
            check_name="volume_anomaly",
            table_name=table, layer="DWS",
            severity=Severity.WARNING if is_anomaly else Severity.INFO,
            passed=not is_anomaly,
            metric_value=z_score,
            threshold=z_score_threshold,
            message=(
                f"表 {table} 数据量异常: 当前={current_count}, "
                f"历史均值={mean_val:.0f}, Z-Score={z_score:.2f}"
                if is_anomaly else
                f"表 {table} 数据量正常: 当前={current_count}, Z-Score={z_score:.2f}"
            ),
            timestamp=datetime.now()
        )

        if not result.passed:
            self._send_alert(result)

        return result

    # ============================================================
    # 4. 数据新鲜度检查（ADS 层关卡，最关键）
    # ============================================================
    def check_data_freshness(self, table: str, max_delay_minutes: int = 5) -> QualityCheckResult:
        """
        检查 ADS 层数据的新鲜度。
        如果最近一条数据的 computed_at 距今超过 max_delay_minutes → 告警。
        这是判断"实时管道是否正常运转"的最直接指标。
        """
        sql = f"""
            SELECT
                max(computed_at) AS latest_computed_at,
                now() - max(computed_at) AS delay_seconds
            FROM {table}
            WHERE bucket >= today()
        """
        row = self.ch.query_one(sql)
        delay_seconds = row['delay_seconds'] or 0
        delay_minutes = delay_seconds / 60.0

        is_stale = delay_minutes > max_delay_minutes

        result = QualityCheckResult(
            check_name="freshness",
            table_name=table, layer="ADS",
            severity=Severity.CRITICAL if delay_minutes > max_delay_minutes * 2 else Severity.WARNING,
            passed=not is_stale,
            metric_value=delay_minutes,
            threshold=float(max_delay_minutes),
            message=(
                f"表 {table} 数据不新鲜: 延迟 {delay_minutes:.1f} 分钟，阈值 {max_delay_minutes} 分钟"
                if is_stale else
                f"表 {table} 数据新鲜: 延迟 {delay_minutes:.1f} 分钟"
            ),
            timestamp=datetime.now()
        )

        if not result.passed:
            self._send_alert(result)
            # 新鲜度异常 → 自动检查 Flink 任务状态
            self._auto_diagnose_freshness(table)

        return result

    # ============================================================
    # 5. 指标阈值检查（ADS 层数值合理性）
    # ============================================================
    def check_metric_threshold(self, table: str, metric_col: str,
                               min_val: float = None, max_val: float = None) -> QualityCheckResult:
        """
        检查指标值是否在合理范围内。
        例如：GMV 不应为负数，订单量不应超过日均值 ×20（异常暴涨）。
        """
        conditions = []
        if min_val is not None:
            conditions.append(f"min({metric_col}) < {min_val}")
        if max_val is not None:
            conditions.append(f"max({metric_col}) > {max_val}")

        sql = f"""
            SELECT
                min({metric_col}) AS min_val,
                max({metric_col}) AS max_val,
                avg({metric_col}) AS avg_val
            FROM {table}
            WHERE bucket >= now() - INTERVAL 1 HOUR
        """
        row = self.ch.query_one(sql)
        actual_min = row['min_val']
        actual_max = row['max_val']

        violations = []
        if min_val is not None and actual_min < min_val:
            violations.append(f"最小值 {actual_min} < 下限 {min_val}")
        if max_val is not None and actual_max > max_val:
            violations.append(f"最大值 {actual_max} > 上限 {max_val}")

        result = QualityCheckResult(
            check_name="metric_threshold",
            table_name=table, layer="ADS",
            severity=Severity.WARNING,
            passed=len(violations) == 0,
            metric_value=float(actual_max or 0),
            threshold=float(max_val or 0),
            message=(
                f"表 {table} 指标 {metric_col} 越界: {'; '.join(violations)}"
                if violations else
                f"表 {table} 指标 {metric_col} 正常: 范围 [{actual_min}, {actual_max}]"
            ),
            timestamp=datetime.now()
        )

        if not result.passed:
            self._send_alert(result)

        return result

    # ============================================================
    # 6. 跨层一致性检查（DWD vs DWS 数量对齐）
    # ============================================================
    def check_cross_layer_consistency(self, dwd_table: str, dws_table: str,
                                       join_key: str = "order_id") -> QualityCheckResult:
        """
        检查 DWD 层和 DWS 层的数据量一致性。
        DWS 层的聚合记录数应与 DWD 层去重后的主键数一致（允许小幅偏差）。
        """
        sql = f"""
            SELECT
                (SELECT count(DISTINCT {join_key}) FROM {dwd_table} WHERE dt = today()) AS dwd_count,
                (SELECT SUM(order_count) FROM {dws_table}
                 WHERE bucket >= today()) AS dws_count
        """
        row = self.ch.query_one(sql)
        dwd_count = row['dwd_count'] or 0
        dws_count = row['dws_count'] or 0

        diff_pct = abs(dwd_count - dws_count) / max(dwd_count, 1) * 100

        result = QualityCheckResult(
            check_name="cross_layer_consistency",
            table_name=f"{dwd_table} vs {dws_table}",
            layer="DWD→DWS",
            severity=Severity.WARNING if diff_pct > 1.0 else Severity.INFO,
            passed=diff_pct <= 1.0,
            metric_value=diff_pct,
            threshold=1.0,
            message=f"DWD={dwd_count}, DWS={dws_count}, 偏差={diff_pct:.2f}%",
            timestamp=datetime.now()
        )

        if not result.passed:
            self._send_alert(result)

        return result

    # ============================================================
    # 定时巡检主流程
    # ============================================================
    def run_full_check(self) -> List[QualityCheckResult]:
        """执行全量质量检查，建议每 5 分钟执行一次"""
        results = []

        # DWD 层检查
        results.extend(self.check_null_values(
            "dwd_order_detail",
            ["order_id", "user_id", "category", "status", "province"],
            threshold_pct=0.01
        ))
        results.append(self.check_schema_drift(
            "dwd_order_detail",
            {"order_id": "String", "user_id": "String", "amount": "Decimal(15,2)",
             "status": "LowCardinality(String)", "province": "LowCardinality(String)"}
        ))

        # DWS 层检查
        results.append(self.check_volume_anomaly("dws_category_gmv_1min", window_minutes=60))
        results.append(self.check_volume_anomaly("dws_funnel_1min", window_minutes=60))

        # ADS 层检查
        results.append(self.check_data_freshness("ads_realtime_gmv_dashboard", max_delay_minutes=5))
        results.append(self.check_data_freshness("ads_realtime_overview", max_delay_minutes=5))
        results.append(self.check_metric_threshold(
            "ads_realtime_gmv_dashboard", "gmv",
            min_val=0, max_val=50000000  # 单品类单分钟 GMV 上限 5000 万
        ))

        # 跨层一致性检查
        results.append(self.check_cross_layer_consistency(
            "dwd_order_detail", "dws_category_gmv_1min", "order_id"
        ))

        # 汇总报告
        critical_count = sum(1 for r in results if r.severity == Severity.CRITICAL and not r.passed)
        warning_count = sum(1 for r in results if r.severity == Severity.WARNING and not r.passed)
        logger.info(f"质量巡检完成: CRITICAL={critical_count}, WARNING={warning_count}, 总检查={len(results)}")

        return results

    # ============================================================
    # 辅助方法
    # ============================================================
    def _get_table_schema(self, table: str) -> Dict[str, str]:
        """获取 ClickHouse 表的 schema 信息"""
        sql = f"SELECT name, type FROM system.columns WHERE table = '{table}'"
        rows = self.ch.query_all(sql)
        return {r['name']: r['type'] for r in rows}

    def _send_alert(self, result: QualityCheckResult):
        """发送告警"""
        self.alert.send(
            channel="#data-quality",
            level=result.severity.value,
            title=f"[{result.layer}] {result.check_name} 检查未通过",
            message=result.message,
            table=result.table_name,
            timestamp=result.timestamp.isoformat()
        )

    def _auto_diagnose_freshness(self, table: str):
        """新鲜度异常时自动诊断"""
        # 检查 Flink 任务是否在运行
        running_jobs = self.flink.list_running_jobs()
        related_job = [j for j in running_jobs if table in j.sink_tables]
        if not related_job:
            self.alert.send(
                channel="#data-quality",
                level="critical",
                title=f"Flink 任务可能已停止",
                message=f"表 {table} 无对应运行中的 Flink 任务，请检查"
            )
        else:
            job = related_job[0]
            # 检查 Kafka 消费延迟
            lag = self.flink.get_consumer_lag(job.group_id)
            if lag > 50000:
                self.alert.send(
                    channel="#data-quality",
                    level="warning",
                    title=f"Kafka 消费积压",
                    message=f"任务 {job.name} 消费延迟 {lag} 条，可能导致数据不新鲜"
                )


# 定时调度（使用 APScheduler 或 Airflow）
if __name__ == "__main__":
    monitor = DataQualityMonitor(
        clickhouse_client=ClickHouseClient("clickhouse-1:8123"),
        kafka_admin=KafkaAdminClient("kafka:9092"),
        flink_client=FlinkClient("flink-jobmanager:8081"),
        alert_sender=FeishuAlertSender(webhook_url="${FEISHU_WEBHOOK}")
    )
    results = monitor.run_full_check()
    for r in results:
        if not r.passed:
            print(f"[{r.severity.value}] {r.message}")
```

**质量监控规则配置表（可动态修改，无需改代码）：**

```sql
-- 质量规则配置表
CREATE TABLE dq_rule_config (
    rule_id String,
    rule_name String,
    target_table String,
    target_layer LowCardinality(String),   -- ODS / DWD / DWS / ADS
    check_type LowCardinality(String),      -- null_check / schema_drift / volume_anomaly / freshness / threshold
    check_params String,                    -- JSON 格式参数
    severity LowCardinality(String),        -- critical / warning / info
    enabled UInt8 DEFAULT 1,
    updated_at DateTime DEFAULT now()
) ENGINE = ReplacingMergeTree(updated_at)
ORDER BY (target_layer, target_table, rule_id);

-- 插入示例规则
INSERT INTO dq_rule_config (rule_id, rule_name, target_table, target_layer, check_type, check_params, severity) VALUES
('r001', '订单空值检查', 'dwd_order_detail', 'DWD', 'null_check',
 '{"columns": ["order_id", "user_id", "category", "status"], "threshold_pct": 0.01}', 'critical'),
('r002', '订单Schema检查', 'dwd_order_detail', 'DWD', 'schema_drift',
 '{"expected_schema": {"order_id": "String", "user_id": "String", "amount": "Decimal(15,2)"}}', 'critical'),
('r003', 'GMV数据量波动', 'dws_category_gmv_1min', 'DWS', 'volume_anomaly',
 '{"window_minutes": 60, "z_score_threshold": 3.0}', 'warning'),
('r004', 'GMV新鲜度检查', 'ads_realtime_gmv_dashboard', 'ADS', 'freshness',
 '{"max_delay_minutes": 5}', 'critical'),
('r005', 'GMV阈值检查', 'ads_realtime_gmv_dashboard', 'ADS', 'threshold',
 '{"metric_col": "gmv", "min_val": 0, "max_val": 50000000}', 'warning'),
('r006', '转化率新鲜度', 'ads_realtime_conversion', 'ADS', 'freshness',
 '{"max_delay_minutes": 10}', 'warning');
```

### ClickHouse 物化视图与刷新策略

ClickHouse 的物化视图（Materialized View）是 DWS → ADS 层自动填充的核心机制。但物化视图的刷新时机、依赖顺序、数据回填都需要精心设计，否则会出现数据缺失或重复计算。

**物化视图的核心原理：**

```
物化视图 = 触发器 + 目标表
  - 当源表有 INSERT 操作时，自动将新插入的数据按 AS SELECT 逻辑转换后写入目标表
  - 只对 INSERT 触发，不对 ALTER / UPDATE / DELETE 触发
  - 历史数据不会自动回填（创建 MV 之前的数据不会被处理）
  - 多个 MV 之间无执行顺序保证 → 需要手动管理依赖
```

**完整物化视图定义：**

```sql
-- ============================================================
-- MV 1: DWS → ADS 品类 GMV 看板（实时写入触发）
-- ============================================================
-- 目标表已在上文 ADS 层定义：ads_realtime_gmv_dashboard
CREATE MATERIALIZED VIEW mv_dws_to_ads_gmv
TO ads_realtime_gmv_dashboard
AS SELECT
    bucket,
    category,
    province,
    sumState(gmv) AS gmv,                    -- 使用 sumState 写入 AggregateFunction 列
    sumState(discount_total) AS discount_total,
    sumState(order_count) AS order_count,
    sumState(buyer_count) AS buyer_count,
    now() AS computed_at
FROM dws_category_gmv_1min
GROUP BY bucket, category, province;

-- ============================================================
-- MV 2: DWS → ADS 实时转化率（漏斗聚合）
-- ============================================================
CREATE MATERIALIZED VIEW mv_dws_to_ads_conversion
TO ads_realtime_conversion
AS SELECT
    bucket,
    category,
    sum(exposure_count) AS exposure_count,
    sum(click_count) AS click_count,
    sum(cart_count) AS cart_count,
    sum(order_count) AS order_count,
    sum(paid_count) AS paid_count
FROM dws_funnel_1min
GROUP BY bucket, category;

-- ============================================================
-- MV 3: DWS → ADS 实时大屏总览（全局聚合，无维度分组）
-- ============================================================
CREATE MATERIALIZED VIEW mv_dws_to_ads_overview
TO ads_realtime_overview
AS SELECT
    bucket,
    sum(gmv) AS total_gmv,
    sum(order_count) AS total_orders,
    sum(buyer_count) AS total_buyers,
    sum(gmv) / nullIf(sum(order_count), 0) AS avg_order_amount,
    0 AS paid_rate,          -- 需要从转化漏斗补充，此处占位
    now() AS computed_at
FROM dws_category_gmv_1min
GROUP BY bucket;

-- ============================================================
-- MV 4: 分钟 → 小时预聚合表（加速跨小时查询）
-- ============================================================
CREATE TABLE ads_hourly_gmv (
    hour_bucket DateTime,
    category LowCardinality(String),
    province LowCardinality(String),
    gmv Decimal(15,2),
    discount_total Decimal(15,2),
    order_count UInt64,
    buyer_count UInt64,
    computed_at DateTime DEFAULT now()
) ENGINE = SummingMergeTree()
ORDER BY (hour_bucket, category, province)
PARTITION BY toYYYYMMDD(hour_bucket)
TTL hour_bucket + INTERVAL 30 DAY;

CREATE MATERIALIZED VIEW mv_minute_to_hour
TO ads_hourly_gmv
AS SELECT
    toStartOfHour(bucket) AS hour_bucket,
    category,
    province,
    sum(gmv) AS gmv,
    sum(discount_total) AS discount_total,
    sum(order_count) AS order_count,
    sum(buyer_count) AS buyer_count,
    now() AS computed_at
FROM ads_realtime_gmv_dashboard
GROUP BY hour_bucket, category, province;
```

**物化视图刷新策略与依赖管理：**

```python
class MaterializedViewManager:
    """物化视图刷新与依赖管理"""

    # MV 依赖关系（DAG）
    MV_DEPENDENCY_GRAPH = {
        "dws_category_gmv_1min": [],                           # 源头：Flink 直接写入
        "dws_funnel_1min": [],                                  # 源头：Flink 直接写入
        "ads_realtime_gmv_dashboard": ["dws_category_gmv_1min"],  # 依赖 DWS
        "ads_realtime_conversion": ["dws_funnel_1min"],           # 依赖 DWS
        "ads_realtime_overview": ["dws_category_gmv_1min"],       # 依赖 DWS
        "ads_hourly_gmv": ["ads_realtime_gmv_dashboard"],         # 依赖 ADS
    }

    def refresh_materialized_views(self, target_date: str):
        """
        手动刷新物化视图（用于数据回填或修正场景）。
        ClickHouse 的 MV 不支持 ALTER … REFRESH，需要手动 INSERT 重新计算。
        """
        # 按依赖拓扑排序，确保上游先刷新
        order = self._topological_sort(self.MV_DEPENDENCY_GRAPH)

        for table in order:
            dependencies = self.MV_DEPENDENCY_GRAPH.get(table, [])
            if not dependencies:
                continue  # 源头表由 Flink 写入，不需要手动刷新

            logger.info(f"刷新物化视图目标表: {table}, 日期: {target_date}")

            # 先删除目标日期的旧数据
            self.ch.execute(f"ALTER TABLE {table} DELETE WHERE toDate(bucket) = '{target_date}'")

            # 从上游表重新计算并插入
            if table == "ads_realtime_gmv_dashboard":
                self.ch.execute(f"""
                    INSERT INTO {table}
                    SELECT
                        bucket, category, province,
                        sum(gmv) AS gmv,
                        sum(discount_total) AS discount_total,
                        sum(order_count) AS order_count,
                        sum(buyer_count) AS buyer_count,
                        now() AS computed_at
                    FROM dws_category_gmv_1min
                    WHERE toDate(bucket) = '{target_date}'
                    GROUP BY bucket, category, province
                """)
            elif table == "ads_realtime_conversion":
                self.ch.execute(f"""
                    INSERT INTO {table}
                    SELECT
                        bucket, category,
                        sum(exposure_count) AS exposure_count,
                        sum(click_count) AS click_count,
                        sum(cart_count) AS cart_count,
                        sum(order_count) AS order_count,
                        sum(paid_count) AS paid_count
                    FROM dws_funnel_1min
                    WHERE toDate(bucket) = '{target_date}'
                    GROUP BY bucket, category
                """)
            elif table == "ads_realtime_overview":
                self.ch.execute(f"""
                    INSERT INTO {table}
                    SELECT
                        bucket,
                        sum(gmv) AS total_gmv,
                        sum(order_count) AS total_orders,
                        sum(buyer_count) AS total_buyers,
                        sum(gmv) / nullIf(sum(order_count), 0) AS avg_order_amount,
                        0 AS paid_rate,
                        now() AS computed_at
                    FROM dws_category_gmv_1min
                    WHERE toDate(bucket) = '{target_date}'
                    GROUP BY bucket
                """)
            elif table == "ads_hourly_gmv":
                self.ch.execute(f"""
                    INSERT INTO {table}
                    SELECT
                        toStartOfHour(bucket) AS hour_bucket,
                        category, province,
                        sum(gmv) AS gmv,
                        sum(discount_total) AS discount_total,
                        sum(order_count) AS order_count,
                        sum(buyer_count) AS buyer_count,
                        now() AS computed_at
                    FROM ads_realtime_gmv_dashboard
                    WHERE toDate(bucket) = '{target_date}'
                    GROUP BY hour_bucket, category, province
                """)

            logger.info(f"表 {table} 刷新完成")

    def _topological_sort(self, graph: Dict[str, List[str]]) -> List[str]:
        """拓扑排序：确保依赖表中上游表先被处理"""
        visited = set()
        order = []

        def dfs(node):
            if node in visited:
                return
            visited.add(node)
            for dep in graph.get(node, []):
                dfs(dep)
            order.append(node)

        for node in graph:
            dfs(node)
        return order

    def check_mv_health(self):
        """
        检查物化视图健康状态：
        1. MV 是否存在且活跃（未被意外 DROP）
        2. MV 写入是否正常（目标表数据量是否在增长）
        3. MV 延迟是否在可接受范围内
        """
        mvs = self.ch.query_all("""
            SELECT table, name, target
            FROM system.tables
            WHERE engine = 'MaterializedView'
        """)

        for mv in mvs:
            mv_name = mv['name']
            target_table = mv['target']

            # 检查目标表最近 10 分钟是否有新数据
            freshness = self.ch.query_one(f"""
                SELECT now() - max(computed_at) AS delay_seconds
                FROM {target_table}
                WHERE bucket >= now() - INTERVAL 1 HOUR
            """)

            if freshness and freshness['delay_seconds'] > 600:
                logger.warning(f"物化视图 {mv_name} 可能异常: 目标表 {target_table} "
                             f"超过 {freshness['delay_seconds']} 秒无新数据")

                # 自动诊断：检查源表是否有新数据
                source_freshness = self.ch.query_one(f"""
                    SELECT now() - max(bucket) AS delay_seconds
                    FROM dws_category_gmv_1min
                    WHERE bucket >= now() - INTERVAL 1 HOUR
                """)
                if source_freshness and source_freshness['delay_seconds'] < 120:
                    # 源表有数据但目标表无新数据 → MV 可能损坏
                    logger.error(f"物化视图 {mv_name} 损坏: 源表正常但目标表无新数据")
                    # 重建 MV
                    self._rebuild_mv(mv_name)
```

**物化视图常见问题与解决：**

| 问题 | 症状 | 根因 | 解决方案 |
|------|------|------|---------|
| MV 未触发 | 目标表无新数据 | Flink 写入使用 INSERT 而非 INSERT INTO（格式不匹配）| 确保 Flink Sink 的 INSERT 语句与 MV 触发条件匹配 |
| MV 重复计算 | 目标表数据量偏大 | MV 对每批 INSERT 都触发，Flink 微批写入导致频繁触发 | 使用 SummingMergeTree 幂等目标表 |
| MV 创建后无历史数据 | 创建 MV 前的数据未被处理 | MV 只对新 INSERT 触发 | 手动 INSERT 回填历史数据 |
| MV 依赖顺序错误 | ADS 表数据为空 | 上游 MV 还没写入，下游 MV 已经读取 | 确保源表在同一批次内写入完成 |
| 分区 TTL 过早清理 | 查询返回空结果 | TTL 设置过短 | 根据业务需求调整 TTL，DWS 7 天、ADS 7 天、小时表 30 天 |

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
## Flink SQL 实时 ETL 完整实现

```sql
-- Source: Kafka CDC 事件（来自 MySQL binlog）
CREATE TABLE orders_cdc (
    order_id STRING,
    user_id STRING,
    product_id STRING,
    quantity INT,
    amount DECIMAL(12, 2),
    status STRING,
    order_time TIMESTAMP(3),
    op_type STRING,  -- INSERT/UPDATE/DELETE
    WATERMARK FOR order_time AS order_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'mysql_cdc.orders',
    'format' = 'debezium-json',
    'properties.bootstrap.servers' = 'kafka:9092'
);

-- 维度表: ClickHouse 中的用户维度
CREATE TABLE user_dim (
    user_id STRING,
    user_name STRING,
    user_tier STRING,
    region STRING,
    register_date DATE,
    PRIMARY KEY (user_id) NOT ENFORCED
) WITH (
    'connector' = 'jdbc',
    'url' = 'jdbc:clickhouse://ch:8123/dw',
    'table-name' = 'dim_user'
);

-- 实时聚合: 每分钟订单指标
CREATE VIEW order_metrics_1min AS
SELECT
    TUMBLE_START(order_time, INTERVAL '1' MINUTE) as window_start,
    region,
    user_tier,
    COUNT(*) as order_count,
    SUM(amount) as total_amount,
    COUNT(DISTINCT user_id) as unique_users,
    AVG(amount) as avg_order_amount
FROM orders_cdc o JOIN user_dim u ON o.user_id = u.user_id
WHERE op_type IN ('INSERT', 'UPDATE') AND status = 'paid'
GROUP BY TUMBLE(order_time, INTERVAL '1' MINUTE), region, user_tier;

-- Sink: 写入 ClickHouse 实时表
INSERT INTO clickhouse_order_metrics
SELECT * FROM order_metrics_1min;
```

## SCD Type 2 缓慢变化维度

```python
class SCDType2Manager:
    """缓慢变化维度 SCD Type 2：保留历史变更"""

    def update_dimension(self, table_name, natural_key, new_values):
        """更新维度记录（SCD Type 2）"""
        # 1. 将当前有效记录标记为过期
        self.db.execute(
            f"UPDATE {table_name} SET is_current = 0, "
            f"expiry_date = CURRENT_DATE WHERE {natural_key[0]} = %s AND is_current = 1",
            natural_key[1])

        # 2. 插入新记录
        new_values["effective_date"] = now().date()
        new_values["expiry_date"] = date(9999, 12, 31)
        new_values["is_current"] = 1
        self.db.insert(table_name, new_values)

    def query_latest(self, table_name, natural_key_value):
        """查询最新版本的维度记录"""
        return self.db.query_one(
            f"SELECT * FROM {table_name} WHERE id = %s AND is_current = 1",
            natural_key_value)

    def query_as_of(self, table_name, natural_key_value, as_of_date):
        """查询历史版本的维度记录"""
        return self.db.query_one(
            f"SELECT * FROM {table_name} WHERE id = %s "
            f"AND effective_date <= %s AND expiry_date > %s",
            natural_key_value, as_of_date, as_of_date)
```

**SCD Type 2 表结构：**

```sql
CREATE TABLE dim_user (
    surrogate_key BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id VARCHAR(36) NOT NULL,
    user_name VARCHAR(100),
    user_tier VARCHAR(20),
    region VARCHAR(50),
    effective_date DATE NOT NULL,
    expiry_date DATE NOT NULL DEFAULT '9999-12-31',
    is_current TINYINT NOT NULL DEFAULT 1,
    INDEX idx_natural_key (user_id),
    INDEX idx_current (user_id, is_current),
    INDEX idx_as_of (user_id, effective_date, expiry_date)
);
```

## 数据质量监控完整实现

```python
class DataQualityMonitor:
    """数据质量监控"""

    def check_all(self, table_name):
        """执行全量数据质量检查"""
        checks = {
            "null_check": self._check_nulls(table_name),
            "freshness_check": self._check_freshness(table_name),
            "volume_check": self._check_volume(table_name),
            "uniqueness_check": self._check_uniqueness(table_name),
            "referential_check": self._check_referential(table_name),
        }
        failed = [k for k, v in checks.items() if v["status"] == "failed"]
        return {
            "table": table_name,
            "checks": checks,
            "overall_status": "failed" if failed else "passed",
            "failed_checks": failed
        }

    def _check_nulls(self, table_name):
        """空值检查"""
        critical_columns = self.db.get_critical_columns(table_name)
        for col in critical_columns:
            null_rate = self.db.query_one(
                f"SELECT COUNT(*) * 100.0 / (SELECT COUNT(*) FROM {table_name}) as rate "
                f"FROM {table_name} WHERE {col} IS NULL")["rate"]
            if null_rate > 5:
                return {"status": "failed", "column": col,
                        "null_rate": null_rate, "threshold": 5}
        return {"status": "passed"}

    def _check_freshness(self, table_name):
        """新鲜度检查"""
        latest = self.db.query_one(
            f"SELECT MAX(updated_at) as latest FROM {table_name}")["latest"]
        age_minutes = (now() - latest).total_seconds() / 60
        if age_minutes > 60:
            return {"status": "failed", "age_minutes": age_minutes,
                    "threshold": 60}
        return {"status": "passed", "age_minutes": age_minutes}

    def _check_volume(self, table_name):
        """数据量异常检查"""
        today_count = self.db.query_one(
            f"SELECT COUNT(*) as cnt FROM {table_name} "
            f"WHERE DATE(created_at) = CURRENT_DATE")["cnt"]
        avg_7day = self.db.query_one(
            f"SELECT AVG(daily_count) as avg FROM ("
            f"SELECT DATE(created_at) as d, COUNT(*) as daily_count "
            f"FROM {table_name} WHERE created_at > CURRENT_DATE - 7 "
            f"GROUP BY DATE(created_at)) t")["avg"]
        if today_count < avg_7day * 0.5:
            return {"status": "failed", "today_count": today_count,
                    "avg_7day": avg_7day}
        return {"status": "passed"}
```

## 异常场景补充

### 场景：Flink Checkpoint 失败

```
触发：Flink Checkpoint 超时（60秒内未完成）
      → 作业无法保存状态 → 重启后从上一次 checkpoint 恢复
      → 两次 checkpoint 之间的数据需要重新消费
检测：
  1. Checkpoint 耗时 > 30 秒 → 告警
  2. Checkpoint 失败 → 严重告警
  3. 连续 3 次 Checkpoint 失败 → 作业重启
处理：
  1. 增大 Checkpoint 间隔（30s → 60s）
  2. 启用增量 Checkpoint（只保存变化部分）
  3. 增加状态后端容量（RocksDB + SSD）
预防：增量 Checkpoint + 合理间隔 + 监控
```

### 场景：ClickHouse 写入拒绝

```
触发：ClickHouse 写入速度超过 max_insert_speed → 拒绝写入
      → Flink Sink 失败 → 数据积压在 Kafka
检测：
  1. Clickhouse 写入错误率 > 0 → 告警
  2. Kafka consumer lag 持续增长 → 告警
处理：
  1. 增加 ClickHouse 分片（水平扩展）
  2. 增大 max_insert_speed 配置
  3. Kafka 积压数据 → 稍后重试写入
预防：写入限流 + 批量写入 + 预分片
```

### 场景：Schema 演进不兼容

```
触发：上游 MySQL 新增字段 → CDC 同步到 Kafka → ClickHouse 无对应列
      → 写入失败
检测：
  1. CDC schema 变更检测 → 告警
  2. ClickHouse 写入报错 "Unknown column"
处理：
  1. ClickHouse ALTER TABLE ADD COLUMN（Nullable 类型）
  2. 重新消费 Kafka 积压数据
  3. 通知数据团队评估新字段用途
预防：Schema Registry 强制兼容性检查 + 自动 DDL 同步
```

## 实时数仓数据模型设计

```sql
-- 事实表：订单事实表（ClickHouse）
CREATE TABLE fact_order (
    order_id String,
    user_id String,
    product_id String,
    category String,
    amount Decimal(12, 2),
    quantity UInt32,
    order_time DateTime,
    pay_time Nullable(DateTime),
    ship_time Nullable(DateTime),
    receive_time Nullable(DateTime),
    region LowCardinality(String),
    channel LowCardinality(String),
    -- 分区键
    date Date MATERIALIZED toDate(order_time)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (order_time, user_id, order_id)
TTL date + INTERVAL 365 DAY;

-- 聚合表：每分钟订单指标（物化视图）
CREATE MATERIALIZED VIEW mv_order_metrics_1min
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(minute)
ORDER BY (minute, category, region)
AS SELECT
    toStartOfMinute(order_time) as minute,
    category,
    region,
    count() as order_count,
    sum(amount) as total_amount,
    avg(amount) as avg_amount,
    uniqExact(user_id) as unique_users
FROM fact_order
GROUP BY minute, category, region;

-- 维度表：用户维度（SCD Type 2）
CREATE TABLE dim_user (
    user_id String,
    user_name String,
    user_tier LowCardinality(String),
    region LowCardinality(String),
    register_date Date,
    effective_date Date,
    expiry_date Date DEFAULT '9999-12-31',
    is_current UInt8 DEFAULT 1
) ENGINE = ReplaceMergeTree()
ORDER BY (user_id, effective_date);
```

## 实时数仓查询优化

```python
class DataWarehouseQueryOptimizer:
    """实时数仓查询优化"""

    QUERY_PATTERNS = {
        "realtime_dashboard": {
            "description": "实时仪表盘查询",
            "optimization": "使用物化视图 + 1 分钟聚合",
            "expected_latency": "<1s"
        },
        "ad_hoc_analysis": {
            "description": "临时分析查询",
            "optimization": "使用预聚合表 + 分区裁剪",
            "expected_latency": "5-30s"
        },
        "batch_report": {
            "description": "批量报表查询",
            "optimization": "使用历史分区 + 并行查询",
            "expected_latency": "1-5min"
        }
    }

    def optimize_query(self, sql):
        """自动优化查询"""
        # 1. 分区裁剪：添加 date 过滤条件
        if "fact_order" in sql and "date" not in sql:
            sql = sql.replace("FROM fact_order",
                "FROM fact_order WHERE date >= today() - INTERVAL 30 DAY")

        # 2. 使用物化视图替代实时聚合
        if "GROUP BY" in sql and "toStartOfMinute" in sql:
            sql = sql.replace("FROM fact_order", "FROM mv_order_metrics_1min")

        return sql
```

## 性能分析详细数据

**ClickHouse 集群性能基准：**

| 操作 | QPS | 延迟 P95 | 数据量 |
|------|-----|---------|--------|
| 实时仪表盘 | 50 | 500ms | 1 分钟聚合 |
| 用户行为分析 | 10 | 3s | 30 天明细 |
| 销售报表 | 5 | 10s | 月度汇总 |
| 漏斗分析 | 5 | 15s | 7 天明细 |
| 归因分析 | 2 | 30s | 30 天明细 |

**存储增长：**

| 数据 | 日增量 | 月增量 | 压缩比 | 月存储 |
|------|--------|--------|--------|--------|
| 事实表 | 5000 万行 | 15 亿行 | 10:1 | 150GB |
| 聚合表 | 5 万行 | 150 万行 | 5:1 | 500MB |
| 维度表 | 1 万行 | 30 万行 | 3:1 | 100MB |

**月度成本：**

| 组件 | 规格 | 月成本 |
|------|------|-------|
| ClickHouse 集群 | 8c64G + SSD × 6节点 | ¥8 万 |
| Flink 集群 | 8c32G × 6节点 | ¥4 万 |
| Kafka | 8c32G + 5TB × 6节点 | ¥3 万 |
| MySQL (CDC 源) | 8c64G × 主从 | ¥2 万 |
| **合计** | | **¥17 万** |

## 数据血缘追踪完整实现

```python
class DataLineageTracker:
    """数据血缘追踪：从源到目标的完整链路"""

    def record_lineage(self, source_table, target_table, transformation):
        """记录数据血缘关系"""
        self.db.insert("data_lineage", {
            "source_table": source_table,
            "target_table": target_table,
            "transformation": transformation,
            "recorded_at": now()
        })

    def get_upstream(self, table_name, depth=5):
        """获取上游依赖（递归）"""
        visited = set()
        result = []
        queue = [table_name]
        while queue and depth > 0:
            current = queue.pop(0)
            if current in visited:
                continue
            visited.add(current)
            parents = self.db.query(
                "SELECT source_table, transformation FROM data_lineage "
                "WHERE target_table = %s", current)
            for p in parents:
                result.append({"from": p["source_table"], "to": current,
                              "transform": p["transformation"]})
                queue.append(p["source_table"])
            depth -= 1
        return result

    def get_downstream(self, table_name, depth=5):
        """获取下游影响（递归）"""
        visited = set()
        result = []
        queue = [table_name]
        while queue and depth > 0:
            current = queue.pop(0)
            if current in visited:
                continue
            visited.add(current)
            children = self.db.query(
                "SELECT target_table, transformation FROM data_lineage "
                "WHERE source_table = %s", current)
            for c in children:
                result.append({"from": current, "to": c["target_table"],
                              "transform": c["transformation"]})
                queue.append(c["target_table"])
            depth -= 1
        return result

    def impact_analysis(self, table_name):
        """影响分析：修改该表会影响哪些下游"""
        downstream = self.get_downstream(table_name)
        return {
            "table": table_name,
            "affected_tables": [d["to"] for d in downstream],
            "affected_count": len(set(d["to"] for d in downstream)),
            "risk": "HIGH" if len(downstream) > 10 else "MEDIUM" if len(downstream) > 3 else "LOW"
        }
```

## 异常场景补充

### 场景：Flink 作业反压

```
触发：Sink 写入速度 < Source 消费速度 → 反压
      → Source 消费速率下降 → 数据延迟
检测：
  1. Flink Web UI 反压指标 > 50% → 告警
  2. Consumer lag 持续增长 → 严重告警
处理：
  1. 增加并行度（parallelism × 2）
  2. 启用背压检测 → 定位慢算子
  3. 优化慢算子：增加缓存、减少 I/O
预防：反压监控 + 自动扩容 + 算子优化
```

### 场景：维度表更新延迟

```
触发：维度表更新延迟 → 事实表 JOIN 使用旧维度
      → 报表数据不准确
检测：
  1. 维度表最新更新时间 > 1 小时前 → 告警
  2. JOIN 结果中维度为 NULL → 数据质量告警
处理：
  1. 触发维度表手动刷新
  2. 事实表使用 SCD Type 2 → 可回溯到历史维度
  3. 延迟恢复后 → 重新计算受影响的聚合
预防：维度表更新监控 + SCD Type 2 + 定期刷新
```

## ClickHouse 实时查询 SQL 模板库

```sql
-- 1. 日活/月活 (DAU/MAU)
SELECT
    toDate(timestamp) as date,
    uniqExact(user_id) as dau,
    uniqExactIf(user_id, timestamp > now() - INTERVAL 30 DAY) as mau
FROM fact_order
GROUP BY date
ORDER BY date DESC LIMIT 30;

-- 2. 留存率 (次日/7日/30日留存)
WITH first_day AS (
    SELECT user_id, MIN(toDate(order_time)) as first_date
    FROM fact_order GROUP BY user_id
),
cohort_size AS (
    SELECT first_date, count() as size FROM first_day GROUP BY first_date
)
SELECT
    c.first_date,
    c.size as cohort_size,
    countIf(toDate(o.order_time) = c.first_date + 1) * 100.0 / c.size as day1_retention,
    countIf(toDate(o.order_time) = c.first_date + 7) * 100.0 / c.size as day7_retention,
    countIf(toDate(o.order_time) = c.first_date + 30) * 100.0 / c.size as day30_retention
FROM first_day c
JOIN fact_order o ON c.user_id = o.user_id
GROUP BY c.first_date, c.size
ORDER BY c.first_date DESC LIMIT 30;

-- 3. 转化漏斗 (浏览→加购→下单→支付)
WITH steps AS (
    SELECT user_id,
        minIf(timestamp, event='view') as step1,
        minIf(timestamp, event='add_cart') as step2,
        minIf(timestamp, event='order') as step3,
        minIf(timestamp, event='pay') as step4
    FROM user_events
    WHERE timestamp > now() - INTERVAL 7 DAY
    GROUP BY user_id
)
SELECT
    count() as step1_view,
    countIf(step2 > step1) as step2_add_cart,
    countIf(step3 > step2) as step3_order,
    countIf(step4 > step3) as step4_pay,
    countIf(step2 > step1) * 100.0 / count() as view_to_cart_rate,
    countIf(step3 > step2) * 100.0 / countIf(step2 > step1) as cart_to_order_rate,
    countIf(step4 > step3) * 100.0 / countIf(step3 > step2) as order_to_pay_rate
FROM steps;

-- 4. 同期群分析 (按注册月份的付费转化)
SELECT
    toYYYYMM(register_date) as cohort,
    count() as total_users,
    countIf(has_paid) as paid_users,
    countIf(has_paid) * 100.0 / count() as paid_rate,
    avgIf(lifetime_revenue, has_paid) as avg_arpu
FROM user_dim
GROUP BY cohort ORDER BY cohort;

-- 5. 实时 GMV 仪表盘
SELECT
    region,
    category,
    sum(amount) as gmv,
    count() as order_count,
    avg(amount) as avg_order_value,
    uniqExact(user_id) as unique_buyers
FROM fact_order
WHERE order_time > now() - INTERVAL 1 HOUR AND status = 'paid'
GROUP BY region, category
ORDER BY gmv DESC;
```

## 数据质量框架完整实现

```python
class DataQualityFramework:
    """数据质量框架：Great Expectations 风格"""

    def validate_table(self, table_name, expectations):
        """验证表数据质量"""
        results = []
        for exp in expectations:
            if exp["type"] == "expect_column_values_not_null":
                result = self._check_not_null(table_name, exp["column"])
            elif exp["type"] == "expect_column_values_unique":
                result = self._check_unique(table_name, exp["column"])
            elif exp["type"] == "expect_column_values_in_range":
                result = self._check_range(table_name, exp["column"],
                    exp["min_value"], exp["max_value"])
            elif exp["type"] == "expect_table_row_count_between":
                result = self._check_row_count(table_name,
                    exp["min_value"], exp["max_value"])
            elif exp["type"] == "expect_column_values_matching_regex":
                result = self._check_regex(table_name, exp["column"], exp["regex"])
            results.append(result)

        passed = sum(1 for r in results if r["success"])
        total = len(results)
        return {
            "table": table_name,
            "success_rate": passed / total,
            "passed": passed, "total": total,
            "details": results
        }

    def _check_not_null(self, table, column):
        null_count = self.db.query_one(
            f"SELECT count() as cnt FROM {table} WHERE {column} IS NULL")["cnt"]
        return {"expectation": "not_null", "column": column,
                "success": null_count == 0, "null_count": null_count}

    def _check_unique(self, table, column):
        total = self.db.query_one(f"SELECT count() as cnt FROM {table}")["cnt"]
        distinct = self.db.query_one(
            f"SELECT count(DISTINCT {column}) as cnt FROM {table}")["cnt"]
        return {"expectation": "unique", "column": column,
                "success": total == distinct, "duplicate_count": total - distinct}

    def _check_range(self, table, column, min_val, max_val):
        out_of_range = self.db.query_one(
            f"SELECT count() as cnt FROM {table} "
            f"WHERE {column} < {min_val} OR {column} > {max_val}")["cnt"]
        return {"expectation": "in_range", "column": column,
                "success": out_of_range == 0, "out_of_range_count": out_of_range}
```

## 异常场景补充

### 场景：数据质量静默降级

```
触发：数据质量检查框架本身故障 → 不再检查 → 坏数据静默流入
检测：
  1. 质量检查任务超时 > 10 分钟 → 告警
  2. 质量检查结果 24 小时未更新 → 告警
处理：
  1. 质量检查失败 → 下游管道暂停（不消费新数据）
  2. 修复质量检查框架 → 重新检查积压数据
  3. 确认数据质量后 → 恢复管道
预防：质量检查框架高可用 + 检查超时自动暂停管道
```

### 场景：数据目录元数据不同步

```
触发：ClickHouse 新增列但 Atlas 中无对应元数据
      → 数据血缘断裂 → 影响分析不完整
检测：
  1. 定期比对 ClickHouse schema vs Atlas 元数据
  2. 不一致 → 告警
处理：
  1. 自动同步：从 ClickHouse INFORMATION_SCHEMA 刷新到 Atlas
  2. 保留手动标注（描述、PII 标记等）
预防：schema 变更时自动触发元数据同步
```

## 实时数据质量框架（Great Expectations 集成）

### Great Expectations 集成与自动检测

```python
import great_expectations as gx
from great_expectations.core import ExpectationSuite
from great_expectations.checkpoint import SimpleCheckpoint
from datetime import datetime, timedelta

class RealtimeDataQualityFramework:
    """
    实时数据质量框架。
    集成 Great Expectations，支持异常检测、坏数据隔离、质量评分。
    """

    def __init__(self, clickhouse_conn, kafka_bootstrap, config):
        self.ch = clickhouse_conn
        self.kafka = kafka_bootstrap
        self.config = config
        self.context = gx.get_context()
        self._init_expectation_suites()
        self._init_anomaly_detectors()

    # ============================================
    # 1. Expectation Suite 初始化
    # ============================================
    def _init_expectation_suites(self):
        """为每张表初始化 Expectation Suite"""
        table_expectations = {
            "fact_order": [
                {"type": "expect_column_values_not_null", "column": "order_id"},
                {"type": "expect_column_values_not_null", "column": "user_id"},
                {"type": "expect_column_values_not_null", "column": "amount"},
                {"type": "expect_column_values_unique", "column": "order_id"},
                {"type": "expect_column_values_in_range",
                 "column": "amount", "min_value": 0.01, "max_value": 1000000},
                {"type": "expect_column_values_in_set",
                 "column": "status", "value_set": ["created", "paid", "cancelled", "refunded"]},
                {"type": "expect_column_values_matching_regex",
                 "column": "order_id", "regex": "^ORD[0-9]{16}$"},
                {"type": "expect_table_row_count_between",
                 "min_value": 1000, "max_value": 50000000},
            ],
            "dim_user": [
                {"type": "expect_column_values_not_null", "column": "user_id"},
                {"type": "expect_column_values_unique", "column": "user_id"},
                {"type": "expect_column_values_in_range",
                 "column": "register_date", "min_value": "2020-01-01", "max_value": "today"},
                {"type": "expect_column_values_in_set",
                 "column": "user_tier", "value_set": ["free", "basic", "premium", "vip"]},
            ],
            "fact_event": [
                {"type": "expect_column_values_not_null", "column": "event_id"},
                {"type": "expect_column_values_not_null", "column": "user_id"},
                {"type": "expect_column_values_unique", "column": "event_id"},
                {"type": "expect_column_values_in_set",
                 "column": "event_type",
                 "value_set": ["view", "click", "add_cart", "order", "pay", "search"]},
            ],
        }

        for table, expectations in table_expectations.items():
            suite = self.context.add_expectation_suite(
                expectation_suite_name=f"{table}_suite")
            for exp in expectations:
                suite.add_expectation(
                    self._build_expectation(exp)
                )
            self.context.add_expectation_suite(suite)

    def _build_expectation(self, exp_config):
        """根据配置构建 Expectation 对象"""
        exp_type = exp_config["type"]
        column = exp_config.get("column")
        kwargs = {k: v for k, v in exp_config.items() if k not in ("type", "column")}

        if exp_type == "expect_column_values_not_null":
            return {"expectation_type": exp_type, "kwargs": {"column": column}}
        elif exp_type == "expect_column_values_unique":
            return {"expectation_type": exp_type, "kwargs": {"column": column}}
        elif exp_type == "expect_column_values_in_range":
            return {"expectation_type": exp_type, "kwargs": {
                "column": column,
                "min_value": kwargs.get("min_value"),
                "max_value": kwargs.get("max_value"),
            }}
        elif exp_type == "expect_column_values_in_set":
            return {"expectation_type": exp_type, "kwargs": {
                "column": column,
                "value_set": kwargs.get("value_set"),
            }}
        elif exp_type == "expect_column_values_matching_regex":
            return {"expectation_type": exp_type, "kwargs": {
                "column": column,
                "regex": kwargs.get("regex"),
            }}
        elif exp_type == "expect_table_row_count_between":
            return {"expectation_type": exp_type, "kwargs": {
                "min_value": kwargs.get("min_value"),
                "max_value": kwargs.get("max_value"),
            }}

    # ============================================
    # 2. 异常检测：基于统计阈值
    # ============================================
    def _init_anomaly_detectors(self):
        """初始化异常检测器"""
        self.anomaly_rules = {
            "fact_order": {
                "hourly_volume": {
                    "metric": "count()",
                    "window": "1 HOUR",
                    "z_score_threshold": 3.0,
                    "min_samples": 168,  # 7 天小时级数据
                },
                "avg_amount": {
                    "metric": "avg(amount)",
                    "window": "1 HOUR",
                    "z_score_threshold": 3.0,
                    "min_samples": 168,
                },
                "null_rate": {
                    "metric": "countIf(user_id IS NULL) / count()",
                    "window": "1 HOUR",
                    "z_score_threshold": 2.5,
                    "min_samples": 168,
                    "max_acceptable": 0.01,  # 硬阈值：NULL 率 > 1% 必定异常
                },
            },
            "fact_event": {
                "hourly_volume": {
                    "metric": "count()",
                    "window": "1 HOUR",
                    "z_score_threshold": 3.0,
                    "min_samples": 168,
                },
            },
        }

    def detect_anomalies(self, table_name):
        """检测表级异常"""
        rules = self.anomaly_rules.get(table_name, {})
        anomalies = []

        for rule_name, rule in rules.items():
            # 获取历史数据计算基线
            history = self._get_metric_history(table_name, rule)
            if len(history) < rule.get("min_samples", 100):
                continue

            # 计算统计基线
            import numpy as np
            values = np.array(history)
            mean = np.mean(values)
            std = np.std(values)

            # 获取当前值
            current = self._get_current_metric(table_name, rule)

            # Z-Score 检测
            z_score = (current - mean) / std if std > 0 else 0
            is_anomaly = abs(z_score) > rule.get("z_score_threshold", 3.0)

            # 硬阈值检测
            hard_threshold = rule.get("max_acceptable")
            hard_breach = hard_threshold is not None and current > hard_threshold

            if is_anomaly or hard_breach:
                anomalies.append({
                    "table": table_name,
                    "rule": rule_name,
                    "current_value": current,
                    "baseline_mean": mean,
                    "baseline_std": std,
                    "z_score": z_score,
                    "detection_type": "hard_threshold" if hard_breach else "z_score",
                    "severity": "CRITICAL" if hard_breach else "WARNING",
                    "timestamp": datetime.utcnow().isoformat(),
                })

        return anomalies

    def _get_metric_history(self, table, rule):
        """获取指标历史值"""
        query = (
            f"SELECT {rule['metric']} as val "
            f"FROM {table} "
            f"WHERE timestamp > now() - INTERVAL 30 DAY "
            f"GROUP BY toStartOfHour(timestamp) "
            f"ORDER BY toStartOfHour(timestamp)"
        )
        result = self.ch.query(query)
        return [row["val"] for row in result]

    def _get_current_metric(self, table, rule):
        """获取指标当前值"""
        query = (
            f"SELECT {rule['metric']} as val FROM {table} "
            f"WHERE timestamp > now() - INTERVAL {rule['window']}"
        )
        return self.ch.query_one(query)["val"]

    # ============================================
    # 3. 坏数据自动隔离
    # ============================================
    def quarantine_bad_data(self, table_name, anomalies):
        """将异常数据隔离到隔离表"""
        quarantine_table = f"quarantine_{table_name}"

        for anomaly in anomalies:
            # 根据异常类型生成隔离条件
            condition = self._build_quarantine_condition(anomaly)

            # 将坏数据移动到隔离表
            insert_sql = (
                f"INSERT INTO {quarantine_table} "
                f"SELECT *, now() as quarantined_at, "
                f"'{anomaly['rule']}' as reason, "
                f"'{anomaly['severity']}' as severity "
                f"FROM {table_name} WHERE {condition}"
            )
            delete_sql = (
                f"ALTER TABLE {table_name} DELETE WHERE {condition}"
            )

            self.ch.execute(insert_sql)
            self.ch.execute(delete_sql)

            # 记录隔离日志
            self._log_quarantine(table_name, anomaly, condition)

        return {
            "table": table_name,
            "quarantined_count": len(anomalies),
            "anomalies": anomalies,
        }

    def _build_quarantine_condition(self, anomaly):
        """根据异常类型构建隔离条件"""
        conditions = {
            "null_rate": "user_id IS NULL",
            "hourly_volume": "1=0",  # 不隔离行，仅告警
            "avg_amount": f"amount > {anomaly['current_value'] * 3}",
        }
        return conditions.get(anomaly["rule"], "1=0")

    # ============================================
    # 4. 质量评分（按表）
    # ============================================
    def calculate_quality_score(self, table_name):
        """计算表级质量评分（0-100）"""
        # 维度 1：Expectation 通过率（权重 40%）
        expectation_result = self._run_expectations(table_name)
        expectation_score = expectation_result["success_rate"] * 100

        # 维度 2：异常检测得分（权重 30%）
        anomalies = self.detect_anomalies(table_name)
        anomaly_score = max(0, 100 - len(anomalies) * 20)

        # 维度 3：数据新鲜度（权重 15%）
        freshness = self._check_freshness(table_name)
        freshness_score = 100 if freshness["delay_minutes"] < 5 else max(0, 100 - freshness["delay_minutes"])

        # 维度 4：完整性（权重 15%）
        completeness = self._check_completeness(table_name)
        completeness_score = completeness["rate"] * 100

        # 综合评分
        total_score = (
            expectation_score * 0.40 +
            anomaly_score * 0.30 +
            freshness_score * 0.15 +
            completeness_score * 0.15
        )

        # 记录评分历史
        self._save_quality_score(table_name, total_score, {
            "expectation": expectation_score,
            "anomaly": anomaly_score,
            "freshness": freshness_score,
            "completeness": completeness_score,
        })

        return {
            "table": table_name,
            "total_score": round(total_score, 1),
            "grade": self._score_to_grade(total_score),
            "dimensions": {
                "expectation": round(expectation_score, 1),
                "anomaly": round(anomaly_score, 1),
                "freshness": round(freshness_score, 1),
                "completeness": round(completeness_score, 1),
            },
            "anomaly_count": len(anomalies),
            "freshness_delay_min": freshness["delay_minutes"],
            "timestamp": datetime.utcnow().isoformat(),
        }

    def _score_to_grade(self, score):
        if score >= 95:
            return "A"
        elif score >= 85:
            return "B"
        elif score >= 70:
            return "C"
        elif score >= 50:
            return "D"
        else:
            return "F"

    def _check_freshness(self, table_name):
        """检查数据新鲜度"""
        result = self.ch.query_one(
            f"SELECT now() - max(timestamp) as delay FROM {table_name}")
        delay_seconds = result["delay"]
        return {"delay_minutes": delay_seconds / 60}

    def _check_completeness(self, table_name):
        """检查数据完整性（对比实时层与离线层行数）"""
        realtime_count = self.ch.query_one(
            f"SELECT count() as cnt FROM {table_name}")["cnt"]
        hive_count = self.hive.query_one(
            f"SELECT count(*) as cnt FROM {table_name}")["cnt"]
        if hive_count == 0:
            return {"rate": 1.0}
        return {"rate": min(realtime_count / hive_count, 1.0)}
```

## 数据目录集成

### Apache Atlas / Hive Metastore 同步

```python
class DataCatalogSync:
    """数据目录同步：ClickHouse ↔ Apache Atlas / Hive Metastore"""

    def __init__(self, clickhouse_conn, atlas_client, hive_metastore):
        self.ch = clickhouse_conn
        self.atlas = atlas_client
        self.hive = hive_metastore

    def sync_schema(self, database_name):
        """同步 ClickHouse schema 到 Atlas 和 Hive Metastore"""
        # 1. 从 ClickHouse 获取最新 schema
        tables = self.ch.query(
            "SELECT table, name, type, comment "
            "FROM system.columns "
            "WHERE database = %(db)s", {"db": database_name})

        # 按 table 分组
        table_columns = {}
        for row in tables:
            table_name = row["table"]
            if table_name not in table_columns:
                table_columns[table_name] = []
            table_columns[table_name].append({
                "name": row["name"],
                "type": row["type"],
                "comment": row["comment"],
            })

        # 2. 同步到 Atlas
        for table_name, columns in table_columns.items():
            self._sync_to_atlas(database_name, table_name, columns)

        # 3. 同步到 Hive Metastore
        for table_name, columns in table_columns.items():
            self._sync_to_hive(database_name, table_name, columns)

        return {"synced_tables": len(table_columns)}

    def _sync_to_atlas(self, database, table, columns):
        """同步到 Apache Atlas"""
        # 创建/更新 Atlas Entity
        entity = {
            "typeName": "clickhouse_table",
            "attributes": {
                "name": table,
                "qualifiedName": f"{database}.{table}@cluster",
                "database": database,
                "owner": self._get_table_owner(database, table),
                "columns": [
                    {
                        "typeName": "clickhouse_column",
                        "attributes": {
                            "name": col["name"],
                            "type": col["type"],
                            "comment": col["comment"],
                            "is_pii": self._classify_pii(table, col["name"]),
                            "owner": self._get_table_owner(database, table),
                        },
                    }
                    for col in columns
                ],
            },
        }
        self.atlas.create_or_update_entity(entity)

    def _sync_to_hive(self, database, table, columns):
        """同步到 Hive Metastore"""
        # 映射 ClickHouse 类型到 Hive 类型
        type_mapping = {
            "String": "STRING",
            "UInt64": "BIGINT",
            "UInt32": "INT",
            "Float64": "DOUBLE",
            "DateTime": "TIMESTAMP",
            "Date": "DATE",
            "Decimal(18,2)": "DECIMAL(18,2)",
        }

        hive_columns = []
        for col in columns:
            hive_type = type_mapping.get(col["type"], "STRING")
            hive_columns.append({
                "name": col["name"],
                "type": hive_type,
                "comment": col["comment"],
            })

        self.hive.create_or_alter_table(database, table, hive_columns)

    # ============================================
    # 列级血缘
    # ============================================
    def trace_column_lineage(self, table_name, column_name, depth=5):
        """追踪列级血缘"""
        lineage = {"upstream": [], "downstream": []}

        # 上游：该列的数据来源
        upstream = self.atlas.search(
            entity_type="clickhouse_column",
            query=f"downstream.{table_name}.{column_name}",
            depth=depth)
        for node in upstream:
            lineage["upstream"].append({
                "table": node["table"],
                "column": node["column"],
                "transformation": node.get("transformation", "direct"),
            })

        # 下游：该列被哪些表/列使用
        downstream = self.atlas.search(
            entity_type="clickhouse_column",
            query=f"upstream.{table_name}.{column_name}",
            depth=depth)
        for node in downstream:
            lineage["downstream"].append({
                "table": node["table"],
                "column": node["column"],
                "transformation": node.get("transformation", "direct"),
            })

        return lineage

    # ============================================
    # PII 分类
    # ============================================
    PII_PATTERNS = {
        "phone": r"(?i)(phone|mobile|tel|cell)",
        "email": r"(?i)(email|mail)",
        "id_card": r"(?i)(id_card|identity|身份证|id_number)",
        "bank_card": r"(?i)(bank_card|credit_card|card_number)",
        "address": r"(?i)(address|addr|地址)",
        "name": r"(?i)(real_name|full_name|姓名|user_name)",
    }

    def _classify_pii(self, table_name, column_name):
        """自动分类 PII 字段"""
        import re
        for pii_type, pattern in self.PII_PATTERNS.items():
            if re.search(pattern, column_name):
                return pii_type
        return None

    def scan_and_tag_pii(self, database_name):
        """扫描数据库中所有表，自动标记 PII 字段"""
        columns = self.ch.query(
            "SELECT table, name FROM system.columns "
            "WHERE database = %(db)s", {"db": database_name})

        pii_findings = []
        for col in columns:
            pii_type = self._classify_pii(col["table"], col["name"])
            if pii_type:
                pii_findings.append({
                    "table": col["table"],
                    "column": col["name"],
                    "pii_type": pii_type,
                })
                # 在 Atlas 中添加 PII 标签
                self.atlas.add_classification(
                    entity_type="clickhouse_column",
                    qualified_name=f"{database_name}.{col['table']}.{col['name']}",
                    classification="PII",
                    attributes={"pii_type": pii_type})

        return pii_findings

    # ============================================
    # 按分类的访问控制
    # ============================================
    def apply_access_control(self, database_name):
        """根据 PII 分类自动应用访问控制"""
        pii_columns = self.scan_and_tag_pii(database_name)

        for pii in pii_columns:
            table = pii["table"]
            column = pii["column"]
            pii_type = pii["pii_type"]

            if pii_type in ("phone", "id_card", "bank_card"):
                # 高敏感：仅数据团队负责人可访问原始数据
                self.ch.execute(
                    f"ALTER TABLE {database_name}.{table} "
                    f"ADD COLUMN {column}_masked String DEFAULT mask({column})")
                # 创建脱敏视图
                self.ch.execute(
                    f"CREATE VIEW IF NOT EXISTS {database_name}.{table}_masked AS "
                    f"SELECT * EXCEPT ({column}), "
                    f"mask({column}) as {column} "
                    f"FROM {database_name}.{table}")
                # 普通用户只能访问脱敏视图
                self.ch.execute(
                    f"GRANT SELECT ON {database_name}.{table}_masked TO analyst_role")
                self.ch.execute(
                    f"GRANT SELECT ON {database_name}.{table} TO data_admin_role")

            elif pii_type in ("email", "name"):
                # 中敏感：可访问但需脱敏
                self.ch.execute(
                    f"GRANT SELECT ON {database_name}.{table}_masked TO analyst_role")
```

## 实时仪表盘 SQL 配方（ClickHouse）

### 10 个常用分析查询

```sql
-- ============================================
-- 1. DAU/MAU 及 DAU/MAU 比率
-- ============================================
SELECT
    toDate(event_time) as date,
    uniqExact(user_id) as dau,
    uniqExactIf(user_id,
        event_time > toStartOfMonth(event_time)) as mau_current_month,
    round(dau / nullIf(
        uniqExactIf(user_id,
            event_time > toStartOfMonth(event_time)), 0) * 100, 2) as dau_mau_ratio
FROM fact_event
WHERE event_time > now() - INTERVAL 30 DAY
GROUP BY date
ORDER BY date DESC;

-- ============================================
-- 2. 留存率（次日/7日/14日/30日）
-- ============================================
WITH user_first_day AS (
    SELECT user_id, MIN(toDate(event_time)) as first_day
    FROM fact_event
    WHERE event_time > now() - INTERVAL 60 DAY
    GROUP BY user_id
),
retention_data AS (
    SELECT
        f.first_day,
        dateDiff('day', f.first_day, toDate(e.event_time)) as day_offset,
        count(DISTINCT e.user_id) as retained_users
    FROM user_first_day f
    JOIN fact_event e ON f.user_id = e.user_id
    GROUP BY f.first_day, day_offset
)
SELECT
    first_day,
    sumIf(retained_users, day_offset = 0) as cohort_size,
    round(sumIf(retained_users, day_offset = 1) * 100.0 /
        nullIf(sumIf(retained_users, day_offset = 0), 0), 2) as day1_retention,
    round(sumIf(retained_users, day_offset = 7) * 100.0 /
        nullIf(sumIf(retained_users, day_offset = 0), 0), 2) as day7_retention,
    round(sumIf(retained_users, day_offset = 14) * 100.0 /
        nullIf(sumIf(retained_users, day_offset = 0), 0), 2) as day14_retention,
    round(sumIf(retained_users, day_offset = 30) * 100.0 /
        nullIf(sumIf(retained_users, day_offset = 0), 0), 2) as day30_retention
FROM retention_data
GROUP BY first_day
ORDER BY first_day DESC LIMIT 30;

-- ============================================
-- 3. 转化漏斗（搜索→浏览→加购→下单→支付）
-- ============================================
WITH funnel_steps AS (
    SELECT user_id,
        minIf(toDateTime(event_time), event_type = 'search') as step1_search,
        minIf(toDateTime(event_time), event_type = 'view') as step2_view,
        minIf(toDateTime(event_time), event_type = 'add_cart') as step3_cart,
        minIf(toDateTime(event_time), event_type = 'order') as step4_order,
        minIf(toDateTime(event_time), event_type = 'pay') as step5_pay
    FROM fact_event
    WHERE event_time > now() - INTERVAL 7 DAY
    GROUP BY user_id
)
SELECT
    count() as step1_search,
    countIf(step2_view > step1_search) as step2_view,
    countIf(step3_cart > step2_view) as step3_cart,
    countIf(step4_order > step3_cart) as step4_order,
    countIf(step5_pay > step4_order) as step5_pay,
    round(countIf(step2_view > step1_search) * 100.0 / nullIf(count(), 0), 2) as search_to_view,
    round(countIf(step3_cart > step2_view) * 100.0 /
        nullIf(countIf(step2_view > step1_search), 0), 2) as view_to_cart,
    round(countIf(step4_order > step3_cart) * 100.0 /
        nullIf(countIf(step3_cart > step2_view), 0), 2) as cart_to_order,
    round(countIf(step5_pay > step4_order) * 100.0 /
        nullIf(countIf(step4_order > step3_cart), 0), 2) as order_to_pay
FROM funnel_steps;

-- ============================================
-- 4. 同期群分析（按注册月份的用户价值）
-- ============================================
SELECT
    toYYYYMM(u.register_date) as cohort_month,
    count(DISTINCT u.user_id) as total_users,
    countDistinctIf(u.user_id, o.order_id IS NOT NULL) as paid_users,
    round(countDistinctIf(u.user_id, o.order_id IS NOT NULL) * 100.0 /
        count(DISTINCT u.user_id), 2) as paid_rate,
    round(sum(o.amount) / nullIf(count(DISTINCT u.user_id), 0), 2) as arpu,
    round(sumIf(o.amount, o.order_id IS NOT NULL) /
        nullIf(countDistinctIf(u.user_id, o.order_id IS NOT NULL), 0), 2) as arppu
FROM dim_user u
LEFT JOIN fact_order o ON u.user_id = o.user_id
GROUP BY cohort_month
ORDER BY cohort_month DESC;

-- ============================================
-- 5. 实时 GMV 仪表盘（按区域+品类）
-- ============================================
SELECT
    region,
    category,
    sum(amount) as gmv,
    count() as order_count,
    round(avg(amount), 2) as avg_order_value,
    uniqExact(user_id) as unique_buyers,
    count() * 100.0 / nullIf(sum(count()) OVER (), 0) as gmv_share
FROM fact_order
WHERE order_time > now() - INTERVAL 1 HOUR AND status = 'paid'
GROUP BY region, category
ORDER BY gmv DESC;

-- ============================================
-- 6. 实时收入趋势（分钟级粒度）
-- ============================================
SELECT
    toStartOfMinute(order_time) as minute_bucket,
    sum(amount) as revenue,
    count() as order_count,
    uniqExact(user_id) as unique_users,
    round(sum(amount) / nullIf(count(), 0), 2) as avg_order_value
FROM fact_order
WHERE order_time > now() - INTERVAL 6 HOUR AND status = 'paid'
GROUP BY minute_bucket
ORDER BY minute_bucket;

-- ============================================
-- 7. Top 热销商品（实时）
-- ============================================
SELECT
    product_id,
    product_name,
    category,
    count() as order_count,
    sum(amount) as total_revenue,
    uniqExact(user_id) as unique_buyers,
    round(sum(amount) / nullIf(count(), 0), 2) as avg_price
FROM fact_order
WHERE order_time > now() - INTERVAL 1 HOUR AND status = 'paid'
GROUP BY product_id, product_name, category
ORDER BY order_count DESC LIMIT 20;

-- ============================================
-- 8. 用户分群 RFM 分析
-- ============================================
WITH rfm_base AS (
    SELECT
        user_id,
        dateDiff('day', max(order_time), now()) as recency_days,
        count() as frequency,
        sum(amount) as monetary
    FROM fact_order
    WHERE status = 'paid' AND order_time > now() - INTERVAL 90 DAY
    GROUP BY user_id
),
rfm_scores AS (
    SELECT
        user_id,
        recency_days,
        frequency,
        monetary,
        ntile(5) OVER (ORDER BY recency_days DESC) as r_score,
        ntile(5) OVER (ORDER BY frequency ASC) as f_score,
        ntile(5) OVER (ORDER BY monetary ASC) as m_score
    FROM rfm_base
)
SELECT
    CASE
        WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN '高价值用户'
        WHEN r_score >= 4 AND f_score >= 3 THEN '活跃用户'
        WHEN r_score <= 2 AND f_score >= 3 THEN '流失风险用户'
        WHEN r_score <= 2 AND f_score <= 2 THEN '流失用户'
        WHEN f_score >= 4 AND m_score >= 4 THEN '忠诚高消费'
        ELSE '普通用户'
    END as user_segment,
    count() as user_count,
    round(avg(monetary), 2) as avg_monetary,
    round(avg(frequency), 2) as avg_frequency
FROM rfm_scores
GROUP BY user_segment
ORDER BY user_count DESC;

-- ============================================
-- 9. 实时渠道归因
-- ============================================
SELECT
    utm_source,
    utm_medium,
    count() as total_events,
    countDistinctIf(user_id, event_type = 'order') as converted_users,
    round(countDistinctIf(user_id, event_type = 'order') * 100.0 /
        nullIf(count(DISTINCT user_id), 0), 2) as conversion_rate,
    sumIf(amount, event_type = 'pay') as revenue,
    round(sumIf(amount, event_type = 'pay') /
        nullIf(countDistinctIf(user_id, event_type = 'order'), 0), 2) as revenue_per_user
FROM fact_event
WHERE event_time > now() - INTERVAL 24 HOUR
GROUP BY utm_source, utm_medium
ORDER BY revenue DESC;

-- ============================================
-- 10. 实时库存预警
-- ============================================
SELECT
    p.product_id,
    p.product_name,
    p.current_stock,
    p.daily_avg_sales,
    round(p.current_stock / nullIf(p.daily_avg_sales, 0), 1) as days_of_inventory,
    CASE
        WHEN p.current_stock / nullIf(p.daily_avg_sales, 0) < 3 THEN '紧急补货'
        WHEN p.current_stock / nullIf(p.daily_avg_sales, 0) < 7 THEN '需要补货'
        WHEN p.current_stock / nullIf(p.daily_avg_sales, 0) < 14 THEN '关注'
        ELSE '正常'
    END as alert_level
FROM (
    SELECT
        i.product_id,
        i.product_name,
        i.stock_quantity as current_stock,
        countIf(o.order_id IS NOT NULL) / 7.0 as daily_avg_sales
    FROM dim_inventory i
    LEFT JOIN fact_order o ON i.product_id = o.product_id
        AND o.order_time > now() - INTERVAL 7 DAY AND o.status = 'paid'
    GROUP BY i.product_id, i.product_name, i.stock_quantity
) p
WHERE p.current_stock / nullIf(p.daily_avg_sales, 0) < 14
ORDER BY days_of_inventory ASC;
```

## 异常场景补充

### 场景：数据质量静默降级

```
触发：数据质量检查框架本身故障 → 不再检查 → 坏数据静默流入下游
      → 报表数据错误但无人发现 → 业务决策基于错误数据
根因分析：
  1. 质量检查框架依赖的 ClickHouse 查询超时 → 检查任务卡住
  2. 检查框架无超时机制 → 永远等待 → 检查结果不再更新
  3. 下游管道未检测质量检查结果是否新鲜 → 坏数据照常消费
  4. 质量评分面板显示的是"缓存值"→ 掩盖了检查已停止的事实
检测：
  1. 质量检查任务超时 > 10 分钟 → 告警
  2. 质量检查结果 24 小时未更新 → 严重告警
  3. 质量评分面板增加"最后检查时间"显示
  4. 定期心跳检查：质量框架每 5 分钟写入心跳 → 未写入则告警
处理：
  1. 立即：暂停受影响表的下游管道（不消费新数据）
  2. 排查：质量框架为何停止（超时/内存/OOM/DB 连接）
  3. 恢复质量检查框架
  4. 对积压数据重新运行质量检查
  5. 确认所有数据质量合格后 → 恢复下游管道
  6. 评估影响：坏数据是否已流入报表 → 需要修正的报表范围
预防：
  - 质量检查框架高可用部署（至少 2 实例）
  - 检查任务设置超时（5 分钟）→ 超时自动标记为失败
  - 心跳机制 + 质量结果新鲜度监控
  - 下游管道在消费前检查质量评分 ≥ B 级
  - 每日自动发送质量报告 → 人工抽查
```

### 场景：数据目录元数据不同步

```
触发：ClickHouse 新增列/修改列但 Atlas/Hive Metastore 中无对应元数据
      → 数据血缘断裂 → 影响分析不完整
      → PII 字段未被标记 → 合规风险
      → 分析师查询 Hive 表时缺列 → 报表报错
根因分析：
  1. DDL 变更未触发元数据同步（手动操作绕过了 CI/CD）
  2. 定时同步任务频率过低（每天一次）→ 新增列延迟 24 小时才同步
  3. 同步任务本身失败但无告警
  4. Atlas Entity 已存在时更新操作被跳过（幂等逻辑 Bug）
检测：
  1. 定期比对 ClickHouse system.columns vs Atlas/Hive 元数据
  2. schema 不一致 → 告警（表级 + 列级差异）
  3. 同步任务失败/超时 → 告警
  4. 血缘关系断链率 > 5% → 告警
处理：
  1. 立即：手动触发全量 schema 同步
  2. 验证：比对同步后的 Atlas/Hive 元数据与 ClickHouse 实际 schema
  3. 补标：对新增列执行 PII 扫描和分类标记
  4. 修复同步任务：确保幂等更新逻辑正确
  5. 评估影响：哪些报表/查询受元数据缺失影响
预防：
  - schema 变更时自动触发元数据同步（DDL 事件驱动）
  - 增加同步频率（每小时增量同步 + 每天全量比对）
  - 同步任务增加成功/失败告警
  - CI/CD 流水线中增加元数据同步步骤
  - 定期执行 schema 一致性校验（ClickHouse ↔ Atlas ↔ Hive）
```

## 实时数据血缘追踪完整实现

```python
class DataLineageTracker:
    """数据血缘追踪：表级 + 列级血缘"""

    def track_table_lineage(self, target_table, source_tables, transformation):
        """追踪表级血缘"""
        lineage_id = str(uuid4())
        self.db.insert("table_lineage", {
            "lineage_id": lineage_id,
            "target_table": target_table,
            "source_tables": json.dumps(source_tables),
            "transformation_type": transformation["type"],  # etl / view / join / aggregate
            "transformation_sql": transformation.get("sql"),
            "pipeline_id": transformation.get("pipeline_id"),
            "created_at": now()
        })

        # 构建血缘图
        self._update_lineage_graph(target_table, source_tables)

        return lineage_id

    def track_column_lineage(self, target_table, target_column,
                             source_table, source_column, transformation_expr):
        """追踪列级血缘"""
        self.db.insert("column_lineage", {
            "target_table": target_table,
            "target_column": target_column,
            "source_table": source_table,
            "source_column": source_column,
            "transformation_expr": transformation_expr,
            "created_at": now()
        })

    def get_upstream(self, table_name, depth=3):
        """获取上游血缘（递归）"""
        visited = set()
        result = []

        def _traverse(table, current_depth):
            if current_depth > depth or table in visited:
                return
            visited.add(table)

            sources = self.db.query(
                "SELECT * FROM table_lineage WHERE target_table = %s", table)
            for source in sources:
                source_tables = json.loads(source["source_tables"])
                for st in source_tables:
                    result.append({
                        "source": st, "target": table,
                        "depth": current_depth,
                        "transformation": source["transformation_type"]
                    })
                    _traverse(st, current_depth + 1)

        _traverse(table_name, 1)
        return result

    def get_downstream(self, table_name, depth=3):
        """获取下游影响范围"""
        visited = set()
        result = []

        def _traverse(table, current_depth):
            if current_depth > depth or table in visited:
                return
            visited.add(table)

            targets = self.db.query(
                "SELECT * FROM table_lineage "
                "WHERE JSON_CONTAINS(source_tables, %s)",
                json.dumps(table))
            for target in targets:
                result.append({
                    "source": table, "target": target["target_table"],
                    "depth": current_depth,
                    "transformation": target["transformation_type"]
                })
                _traverse(target["target_table"], current_depth + 1)

        _traverse(table_name, 1)
        return result

    def impact_analysis(self, table_name, change_type):
        """变更影响分析"""
        downstream = self.get_downstream(table_name, depth=5)
        affected_tables = list(set(d["target"] for d in downstream))

        return {
            "source_table": table_name,
            "change_type": change_type,  # schema_change / data_deletion / column_rename
            "affected_tables": affected_tables,
            "affected_count": len(affected_tables),
            "risk_level": "high" if len(affected_tables) > 10 else
                         "medium" if len(affected_tables) > 3 else "low",
            "recommended_action": self._recommend_action(
                change_type, len(affected_tables))
        }

    def _recommend_action(self, change_type, affected_count):
        if change_type == "schema_change" and affected_count > 5:
            return "建议分阶段变更：先新增列 → 迁移数据 → 删除旧列"
        elif change_type == "data_deletion" and affected_count > 3:
            return "建议软删除：添加 is_deleted 标记，而非物理删除"
        return "可直接变更，但需通知下游表负责人"
```

## 数据管道监控

```python
class DataPipelineMonitor:
    """数据管道监控：延迟 + 数据量 + 质量"""

    def check_pipeline_health(self, pipeline_id):
        """检查管道健康状态"""
        pipeline = self.db.get_pipeline(pipeline_id)

        checks = []

        # 1. 延迟检查：数据新鲜度
        latest_data = self.db.query_one(
            "SELECT MAX(timestamp) as latest FROM {table} "
            .format(table=pipeline["target_table"]))
        if latest_data:
            delay = (now() - latest_data["latest"]).total_seconds() / 60
            checks.append({
                "name": "data_freshness",
                "value": f"{delay:.0f}min",
                "status": "ok" if delay < 15 else "warning" if delay < 60 else "critical"
            })

        # 2. 数据量检查：今日数据量 vs 7 日均值
        today_count = self.db.query_one(
            "SELECT COUNT(*) as cnt FROM {table} WHERE DATE(timestamp) = CURRENT_DATE"
            .format(table=pipeline["target_table"]))["cnt"]
        avg_7d = self.db.query_one(
            "SELECT AVG(daily_count) as avg FROM ("
            "SELECT COUNT(*) as daily_count FROM {table} "
            "GROUP BY DATE(timestamp) ORDER BY DATE(timestamp) DESC LIMIT 7) t"
            .format(table=pipeline["target_table"]))["avg"] or 0

        if avg_7d > 0:
            volume_ratio = today_count / avg_7d
            checks.append({
                "name": "data_volume",
                "value": f"{volume_ratio:.1f}x avg",
                "status": "ok" if 0.5 < volume_ratio < 2 else "warning"
            })

        # 3. 空值率检查
        null_rate = self.db.query_one(
            "SELECT AVG(CASE WHEN {col} IS NULL THEN 1 ELSE 0 END) as rate "
            "FROM {table} WHERE DATE(timestamp) = CURRENT_DATE"
            .format(table=pipeline["target_table"],
                    col=pipeline.get("key_column", "id")))["rate"] or 0
        checks.append({
            "name": "null_rate",
            "value": f"{null_rate:.1%}",
            "status": "ok" if null_rate < 0.01 else "warning"
        })

        return {"pipeline_id": pipeline_id, "checks": checks,
                "overall": "healthy" if all(c["status"] == "ok" for c in checks)
                          else "degraded"}
```

## 异常场景补充

### 场景：血缘图循环引用

```
触发：表 A 依赖表 B，表 B 又依赖表 A → 血缘图出现环
检测：
  1. 血缘图构建时检测环（DFS）
  2. 发现环 → 告警
处理：
  1. 标记循环依赖的表
  2. 通知数据团队拆分循环
  3. 拆分前 → 影响分析不完整（无法递归上游）
预防：ETL 设计时避免循环 + 血缘图环检测
```

### 场景：管道监控误报

```
触发：节假日数据量自然下降 → 体积比 < 0.5 → 触发告警
检测：
  1. 数据量下降 + 是节假日 → 误报
  2. 误报浪费运维精力
处理：
  1. 引入节假日日历 → 调整基线
  2. 使用同周对比（上周同一天）替代 7 日均值
  3. 告警增加上下文（"今日是节假日，数据量下降属正常"）
预防：节假日感知 + 同周对比 + 告警上下文
```

## 实时数据质量框架完整实现

```python
class DataQualityFramework:
    """数据质量框架：Schema 验证 + 值域检查 + 质量门控"""

    QUALITY_DIMENSIONS = {
        "completeness": "完整性（空值率）",
        "validity": "有效性（类型/范围）",
        "consistency": "一致性（外键/跨表）",
        "timeliness": "时效性（延迟/新鲜度）",
        "uniqueness": "唯一性（重复率）",
    }

    def validate_ingestion(self, table_name, records):
        """入厂数据质量验证"""
        schema = self.schema_registry.get_schema(table_name)
        results = {"total": len(records), "passed": 0, "failed": 0, "issues": []}

        for record in records:
            issues = []

            # 1. Schema 验证（字段类型检查）
            for field, field_type in schema["fields"].items():
                if field in record:
                    if not self._check_type(record[field], field_type):
                        issues.append({
                            "field": field, "issue": "type_mismatch",
                            "expected": field_type,
                            "actual": type(record[field]).__name__
                        })

            # 2. 值域验证
            for field, limits in schema.get("range_limits", {}).items():
                value = record.get(field)
                if value is not None:
                    if value < limits["min"] or value > limits["max"]:
                        issues.append({
                            "field": field, "issue": "out_of_range",
                            "value": value,
                            "range": f"[{limits['min']}, {limits['max']}]"
                        })

            # 3. 外键一致性
            for field, ref in schema.get("foreign_keys", {}).items():
                value = record.get(field)
                if value is not None:
                    exists = self.db.query_one(
                        f"SELECT 1 FROM {ref['table']} WHERE {ref['column']} = %s",
                        value)
                    if not exists:
                        issues.append({
                            "field": field, "issue": "fk_violation",
                            "value": value,
                            "reference": f"{ref['table']}.{ref['column']}"
                        })

            if issues:
                results["failed"] += 1
                results["issues"].extend(issues)
            else:
                results["passed"] += 1

        # 4. 重复检测
        event_ids = [r.get("event_id") for r in records if r.get("event_id")]
        if event_ids:
            existing = self.db.query(
                "SELECT event_id FROM processed_events "
                "WHERE event_id IN (%s)" % ",".join(["%s"] * len(event_ids)),
                *event_ids)
            dup_count = len(existing)
            if dup_count > 0:
                results["issues"].append({
                    "issue": "duplicate_events", "count": dup_count
                })

        return results

    def calculate_quality_score(self, table_name):
        """计算数据质量分数（0-100）"""
        # 空值率 → 完整性得分
        null_rate = self._get_null_rate(table_name)
        completeness_score = max(0, 100 - null_rate * 1000)

        # 类型不匹配率 → 有效性得分
        type_mismatch_rate = self._get_type_mismatch_rate(table_name)
        validity_score = max(0, 100 - type_mismatch_rate * 2000)

        # 超范围率
        out_of_range_rate = self._get_out_of_range_rate(table_name)
        range_score = max(0, 100 - out_of_range_rate * 2000)

        # 综合得分
        overall = (completeness_score * 0.4 +
                   validity_score * 0.3 +
                   range_score * 0.3)

        return {
            "table": table_name,
            "overall_score": round(overall, 1),
            "completeness": round(completeness_score, 1),
            "validity": round(validity_score, 1),
            "range_compliance": round(range_score, 1),
            "gate_passed": overall >= 90
        }

    def enforce_quality_gate(self, table_name):
        """质量门控：低于阈值阻断下游"""
        score = self.calculate_quality_score(table_name)

        if not score["gate_passed"]:
            self.db.update("table_metadata",
                {"quality_blocked": True, "quality_score": score["overall_score"]},
                {"table_name": table_name})

            self.alert(f"数据质量门控未通过: {table_name} 得分 {score['overall_score']}")

            # 阻断下游消费
            self.kafka.pause_consumer(f"dw_consumer_{table_name}")

        return score
```

## 数仓成本优化

```python
class DataWarehouseCostOptimizer:
    """数仓成本优化：存储分层 + 计算优化 + 冷归档"""

    def analyze_storage_cost(self):
        """存储成本分析"""
        tables = self.db.query(
            "SELECT table_name, pg_size_pretty(pg_total_relation_size(table_name)) as size, "
            "pg_total_relation_size(table_name) as bytes, "
            "(SELECT COUNT(*) FROM pg_stat_user_tables WHERE relname = table_name) as row_estimate "
            "FROM information_schema.tables WHERE table_schema = 'public'")

        total_bytes = sum(t["bytes"] for t in tables)
        cost_per_gb = 0.023  # $/GB/month (hot tier)

        return {
            "total_size_gb": round(total_bytes / 1073741824, 1),
            "monthly_cost_usd": round(total_bytes / 1073741824 * cost_per_gb, 2),
            "tables": sorted(tables, key=lambda t: t["bytes"], reverse=True)[:20],
            "optimization_potential": self._estimate_savings(tables)
        }

    def recommend_archival(self, table_name, retention_days=365):
        """推荐归档策略"""
        # 分析数据时间分布
        time_column = self._get_time_column(table_name)
        old_data = self.db.query_one(
            f"SELECT COUNT(*) as count FROM {table_name} "
            f"WHERE {time_column} < NOW() - INTERVAL '{retention_days} days'")

        total = self.db.query_one(f"SELECT COUNT(*) as count FROM {table_name}")

        if old_data["count"] == 0:
            return {"recommendation": "no_action", "reason": "无过期数据"}

        old_pct = old_data["count"] / max(total["count"], 1) * 100

        return {
            "table": table_name,
            "total_rows": total["count"],
            "archivable_rows": old_data["count"],
            "archivable_pct": round(old_pct, 1),
            "recommendation": "archive_to_cold" if old_pct > 30 else "keep_in_hot",
            "estimated_savings_pct": round(old_pct * 0.7, 1),  # cold tier is ~30% of hot cost
            "action": f"CREATE TABLE {table_name}_archive AS SELECT * FROM {table_name} "
                      f"WHERE {time_column} < NOW() - INTERVAL '{retention_days} days'"
        }

    def _estimate_savings(self, tables):
        """估算优化潜力"""
        savings = []
        for t in tables[:20]:
            if t["bytes"] > 1073741824:  # > 1GB
                # 检查是否有压缩
                compression_ratio = self._get_compression_ratio(t["table_name"])
                if compression_ratio < 0.5:
                    savings.append({
                        "table": t["table_name"],
                        "opportunity": "enable_columnar_compression",
                        "estimated_saving_pct": round((1 - compression_ratio) * 50, 0)
                    })
        return savings
```

## 异常场景补充

### 场景：Schema Registry 拒绝向后兼容变更

```
触发：新增可选字段被 Schema Registry 拒绝 → 数据管道中断
检测：
  1. Schema 注册失败 → 告警
  2. 下游消费者报字段缺失 → 兼容性问题
处理：
  1. 检查兼容性规则是否过严
  2. 可选字段应允许添加（BACKWARD 兼容）
  3. 更新 Schema Registry 兼容性策略
预防：BACKWARD 兼容模式 + Schema 变更审批 + 兼容性测试
```

### 场景：质量门控阻断关键看板

```
触发：数据质量分 89 分（阈值 90）→ 门控阻断 → 实时看板无数据
检测：
  1. 看板数据空白 → 门控阻断
  2. 质量分仅差 1 分 → 过于严格
处理：
  1. 紧急放宽门控阈值（90 → 85）
  2. 查明质量下降原因并修复
  3. 修复后恢复原阈值
预防：门控阈值分级（85=警告，90=阻断）+ 紧急覆盖机制
```

## 数据仓库查询优化完整实现

```python
class DataWarehouseQueryOptimizer:
    """数据仓库查询优化：慢查询 + 索引推荐 + 物化视图"""

    def detect_slow_queries(self, threshold_seconds=30):
        """检测慢查询"""
        slow = self.db.query(
            "SELECT query_id, query_text, execution_time_ms, "
            "cpu_time_ms, memory_mb, rows_read, rows_returned "
            "FROM query_history "
            "WHERE execution_time_ms > %s "
            "AND executed_at > NOW() - INTERVAL 1 HOUR "
            "ORDER BY execution_time_ms DESC LIMIT 20",
            threshold_seconds * 1000)

        recommendations = []
        for q in slow:
            plan = self._get_explain_plan(q["query_text"])
            issues = self._analyze_plan(plan)
            recommendations.append({
                "query_id": q["query_id"],
                "execution_time_ms": q["execution_time_ms"],
                "issues": issues,
                "suggestions": self._generate_suggestions(issues)
            })

        return recommendations

    def _analyze_plan(self, plan):
        """分析执行计划"""
        issues = []
        for node in plan:
            if node["operation"] == "Seq Scan":
                issues.append({
                    "type": "full_table_scan",
                    "table": node["relation"],
                    "suggestion": f"建议在 {node['relation']}.{node.get('filter_column')} 上创建索引"
                })
            elif node["operation"] == "Hash Join" and node["rows"] > 1000000:
                issues.append({
                    "type": "large_hash_join",
                    "tables": [node["left"], node["right"]],
                    "suggestion": "考虑分区或物化视图减少 JOIN 数据量"
                })
            elif node.get("rows_read", 0) > 10000000:
                issues.append({
                    "type": "excessive_rows_read",
                    "table": node["relation"],
                    "suggestion": "检查分区裁剪是否生效"
                })
        return issues

    def recommend_materialized_views(self):
        """推荐物化视图（识别重复子查询）"""
        # 分析查询日志中频繁出现的 JOIN 和聚合模式
        query_patterns = self.db.query(
            "SELECT MD5(REGEXP_REPLACE(query_text, '\\d+', '?')) as pattern, "
            "COUNT(*) as frequency, AVG(execution_time_ms) as avg_time "
            "FROM query_history WHERE executed_at > NOW() - INTERVAL 7 DAY "
            "GROUP BY pattern HAVING COUNT(*) > 10 "
            "ORDER BY frequency * avg_time DESC LIMIT 10")

        recommendations = []
        for pattern in query_patterns:
            if pattern["avg_time"] > 5000:  # 平均 > 5 秒
                sample_query = self.db.query_one(
                    "SELECT query_text FROM query_history "
                    "WHERE MD5(REGEXP_REPLACE(query_text, '\\d+', '?')) = %s "
                    "LIMIT 1", pattern["pattern"])

                recommendations.append({
                    "pattern_hash": pattern["pattern"],
                    "frequency": pattern["frequency"],
                    "avg_time_ms": round(pattern["avg_time"]),
                    "suggested_view": self._extract_view_definition(sample_query["query_text"]),
                    "estimated_savings_pct": 80  # 物化视图通常节省 80% 查询时间
                })

        return recommendations

    def manage_query_queue(self):
        """查询队列管理（交互式优先）"""
        return {
            "interactive": {"max_concurrent": 20, "priority": 1},
            "batch": {"max_concurrent": 5, "priority": 2},
            "background": {"max_concurrent": 2, "priority": 3},
        }
```

## 异常场景补充

### 场景：自动索引推荐过多降低写入性能

```
触发：索引推荐引擎创建了 50+ 索引 → 写入性能下降 40%
检测：
  1. 写入延迟增加 → 索引过多
  2. 索引数量 > 表列数 → 过度索引
处理：
  1. 移除使用率 < 5% 的索引
  2. 合并相似索引
  3. 限制单表索引数量（< 10）
预防：索引数量上限 + 使用率监控 + 定期清理
```

### 场景：物化视图维护成本过高

```
触发：10 个物化视图 → 每次 REFRESH 耗时 30 分钟 → 影响实时性
检测：
  1. 物化视图刷新耗时 > ETL 窗口 → 成本过高
  2. 数据新鲜度 > 1 小时 → 影响决策
处理：
  1. 减少刷新频率（5 分钟 → 15 分钟）
  2. 改用连续聚合（TimescaleDB）
  3. 删除 ROI 最低的物化视图
预防：刷新成本评估 + ROI 分析 + 连续聚合替代
```

## 实时数据血缘追踪完整实现

```python
class DataLineageTracker:
    """数据血缘追踪：列级血缘 + 影响分析 + 可视化"""

    def track_column_lineage(self, target_table, target_column):
        """追踪列级血缘"""
        lineage = {"target": f"{target_table}.{target_column}", "upstream": []}

        # 查找直接上游
        dependencies = self.db.query(
            "SELECT source_table, source_column, transformation "
            "FROM column_lineage "
            "WHERE target_table = %s AND target_column = %s",
            target_table, target_column)

        for dep in dependencies:
            node = {
                "source": f"{dep['source_table']}.{dep['source_column']}",
                "transformation": dep["transformation"]
            }
            # 递归追踪
            node["upstream"] = self.track_column_lineage(
                dep["source_table"], dep["source_column"])["upstream"]
            lineage["upstream"].append(node)

        return lineage

    def impact_analysis(self, source_table, source_column):
        """影响分析：如果修改此列，哪些下游会受影响"""
        impacted = []

        # 查找直接下游
        downstream = self.db.query(
            "SELECT target_table, target_column, transformation "
            "FROM column_lineage "
            "WHERE source_table = %s AND source_column = %s",
            source_table, source_column)

        for dep in downstream:
            node = {
                "impacted": f"{dep['target_table']}.{dep['target_column']}",
                "transformation": dep["transformation"]
            }
            # 递归查找下游的下游
            sub_impact = self.impact_analysis(
                dep["target_table"], dep["target_column"])
            node["downstream_impact"] = sub_impact

            # 检查是否影响报表
            reports = self.db.query(
                "SELECT report_name FROM report_dependencies "
                "WHERE table_name = %s AND column_name = %s",
                dep["target_table"], dep["target_column"])
            if reports:
                node["affected_reports"] = [r["report_name"] for r in reports]

            impacted.append(node)

        return {"source": f"{source_table}.{source_column}",
                "impacted_count": len(impacted),
                "impacted": impacted}

    def record_lineage(self, etl_job_id, source_table, target_table,
                       column_mappings):
        """记录 ETL 产生的血缘关系"""
        for mapping in column_mappings:
            self.db.insert("column_lineage", {
                "etl_job_id": etl_job_id,
                "source_table": source_table,
                "source_column": mapping["source_column"],
                "target_table": target_table,
                "target_column": mapping["target_column"],
                "transformation": mapping.get("transformation", "direct"),
                "recorded_at": now()
            })

    def check_lineage_freshness(self):
        """检查血缘数据新鲜度"""
        tables = self.db.query(
            "SELECT table_name, MAX(recorded_at) as last_updated "
            "FROM column_lineage GROUP BY table_name")

        stale = []
        for t in tables:
            if t["last_updated"]:
                hours_since = (now() - t["last_updated"]).total_seconds() / 3600
                if hours_since > 48:
                    stale.append({"table": t["table_name"],
                                 "hours_since_update": round(hours_since, 0)})

        return {"stale_count": len(stale), "stale_tables": stale}
```

## 异常场景补充

### 场景：血缘追踪增加 ETL 开销

```
触发：血缘追踪在每次 ETL 运行时额外写入 → ETL 延迟增加 15%
检测：
  1. ETL 运行时间增加 > 10% → 血缘开销
  2. column_lineage 表膨胀 → 存储问题
处理：
  1. 异步写入血缘（先写消息队列，后批量入库）
  2. 定期归档旧血缘数据
  3. 优化血缘表索引
预防：异步写入 + 批量入库 + 定期归档
```

### 场景：血缘数据不完整

```
触发：新上线的 ETL 作业未注册血缘 → 影响分析遗漏下游 → 变更导致报表错误
检测：
  1. 新表/新列没有血缘记录 → 血缘不完整
  2. 影响分析结果与实际不符 → 血缘缺失
处理：
  1. 强制要求 ETL 作业注册血缘才能上线
  2. 定期扫描发现缺失血缘
  3. 补录缺失的血缘关系
预防：ETL 上线前血缘检查 + 定期扫描 + 强制注册
```

## 数仓数据变更管理完整实现

```python
class DataChangeManagementService:
    """数仓数据变更管理：变更审批 + 影响评估 + 回滚"""

    CHANGE_TYPES = {
        "schema_change": {"risk": "high", "requires_approval": True},
        "data_backfill": {"risk": "medium", "requires_approval": True},
        "pipeline_change": {"risk": "medium", "requires_approval": True},
        "query_optimization": {"risk": "low", "requires_approval": False},
    }

    def submit_change(self, change_type, description, sql_statements,
                     submitted_by):
        """提交数据变更"""
        config = self.CHANGE_TYPES.get(change_type)
        if not config:
            raise InvalidChangeTypeError(f"未知变更类型: {change_type}")

        change_id = str(uuid4())

        # 1. 自动影响评估
        impact = self._assess_impact(sql_statements)

        # 2. 生成回滚脚本
        rollback_sql = self._generate_rollback(sql_statements)

        self.db.insert("data_changes", {
            "change_id": change_id,
            "change_type": change_type,
            "description": description,
            "sql_statements": json.dumps(sql_statements),
            "rollback_sql": json.dumps(rollback_sql),
            "impact_assessment": json.dumps(impact),
            "risk_level": config["risk"],
            "requires_approval": config["requires_approval"],
            "status": "pending_approval" if config["requires_approval"] else "approved",
            "submitted_by": submitted_by,
            "submitted_at": now()
        })

        if config["requires_approval"]:
            self._notify_approvers(change_id, impact)

        return {"change_id": change_id, "status": "pending_approval" if config["requires_approval"] else "approved",
                "risk": config["risk"], "impact": impact}

    def execute_change(self, change_id, executed_by):
        """执行数据变更"""
        change = self.db.get_change(change_id)

        if change["status"] != "approved":
            raise ChangeNotApprovedError("变更未审批")

        # 1. 执行前快照
        snapshot_id = self._create_pre_snapshot(change)

        # 2. 执行变更（在事务中）
        sqls = json.loads(change["sql_statements"])
        try:
            for sql in sqls:
                self.db.execute(sql)

            self.db.update("data_changes",
                {"status": "executed", "executed_by": executed_by,
                 "executed_at": now(), "snapshot_id": snapshot_id},
                {"change_id": change_id})

        except Exception as e:
            # 执行失败 → 自动回滚
            self._auto_rollback(change_id, str(e))
            return {"status": "failed", "error": str(e)}

        # 3. 执行后验证
        validation = self._post_execution_validation(change)
        if not validation["passed"]:
            # 验证失败 → 回滚
            self._rollback(change_id)
            return {"status": "validation_failed", "validation": validation}

        return {"status": "executed", "validation": validation}

    def _assess_impact(self, sql_statements):
        """评估变更影响"""
        impact = {"affected_tables": [], "downstream_reports": [], "risk_factors": []}

        for sql in sql_statements:
            tables = self._extract_tables(sql)
            for table in tables:
                # 查找下游依赖
                downstream = self.lineage.get_downstream(table)
                reports = [d for d in downstream if d["type"] == "report"]

                impact["affected_tables"].append(table)
                impact["downstream_reports"].extend(reports)

                # 风险因素
                if "DROP" in sql.upper():
                    impact["risk_factors"].append(f"删除表 {table}")
                if "ALTER" in sql.upper():
                    impact["risk_factors"].append(f"修改表结构 {table}")

        return impact

    def _generate_rollback(self, sql_statements):
        """生成回滚脚本"""
        rollback = []
        for sql in sql_statements:
            if sql.strip().upper().startswith("ALTER TABLE"):
                # ALTER → 生成反向 ALTER
                rollback.append(self._reverse_alter(sql))
            elif sql.strip().upper().startswith("CREATE TABLE"):
                # CREATE → DROP
                table_name = self._extract_table_name(sql)
                rollback.append(f"DROP TABLE IF EXISTS {table_name}")
            elif sql.strip().upper().startswith("INSERT"):
                # INSERT → DELETE（基于条件）
                rollback.append(self._reverse_insert(sql))
        return rollback
```

## 数仓元数据管理

```python
class DataWarehouseMetadataService:
    """数仓元数据管理：表注册 + 指标定义 + 数据字典"""

    def register_table(self, table_name, schema, owner, description,
                      sensitivity_level="internal"):
        """注册表元数据"""
        columns = []
        for col in schema["columns"]:
            columns.append({
                "name": col["name"],
                "type": col["type"],
                "description": col.get("description", ""),
                "nullable": col.get("nullable", True),
                "pii": col.get("pii", False),
            })

        self.db.insert("table_metadata", {
            "table_name": table_name,
            "schema_name": schema.get("schema", "public"),
            "owner": owner,
            "description": description,
            "columns": json.dumps(columns),
            "row_count_estimate": schema.get("row_count_estimate"),
            "size_bytes_estimate": schema.get("size_bytes_estimate"),
            "sensitivity_level": sensitivity_level,  # public / internal / confidential / restricted
            "refresh_frequency": schema.get("refresh_frequency", "daily"),
            "registered_at": now()
        })

    def register_metric(self, metric_name, definition, table_name,
                       calculation_sql, owner):
        """注册指标定义"""
        # 验证 SQL 可执行
        try:
            self.db.execute(f"EXPLAIN {calculation_sql}")
        except Exception as e:
            raise InvalidMetricDefinitionError(f"指标 SQL 无效: {e}")

        self.db.insert("metric_definitions", {
            "metric_name": metric_name,
            "definition": definition,
            "table_name": table_name,
            "calculation_sql": calculation_sql,
            "owner": owner,
            "version": 1,
            "status": "active",
            "registered_at": now()
        })

    def search_data_dictionary(self, keyword):
        """搜索数据字典"""
        results = []

        # 搜索表名
        tables = self.db.query(
            "SELECT * FROM table_metadata "
            "WHERE table_name ILIKE %s OR description ILIKE %s",
            f"%{keyword}%", f"%{keyword}%")

        for t in tables:
            results.append({
                "type": "table",
                "name": t["table_name"],
                "description": t["description"],
                "owner": t["owner"],
                "sensitivity": t["sensitivity_level"]
            })

        # 搜索列名
        all_tables = self.db.query("SELECT table_name, columns FROM table_metadata")
        for t in all_tables:
            cols = json.loads(t["columns"])
            for col in cols:
                if keyword.lower() in col["name"].lower() or keyword.lower() in col.get("description", "").lower():
                    results.append({
                        "type": "column",
                        "table": t["table_name"],
                        "name": col["name"],
                        "column_type": col["type"],
                        "description": col.get("description", ""),
                        "pii": col.get("pii", False)
                    })

        return results
```

## 异常场景补充

### 场景：数据变更回滚脚本不可用

```
触发：Schema 变更执行后发现问题 → 回滚脚本执行失败 → 无法回退
检测：
  1. 回滚脚本执行报错 → 不可用
  2. 数据已被修改但无法恢复 → 严重
处理：
  1. 从执行前快照恢复
  2. 手动编写修复 SQL
  3. 严重时从备份恢复
预防：回滚脚本测试 + 执行前快照 + 备份保留
```

### 场景：元数据注册与实际不符

```
触发：表结构已变更但元数据未更新 → 数据字典过时 → 新用户误用
检测：
  1. 元数据中的列数与实际列数不一致 → 过时
  2. 查询报"列不存在"但字典中有 → 不符
处理：
  1. 自动同步：定期扫描实际表结构并更新元数据
  2. 人工审核差异
  3. 通知表 owner 确认
预防：定期自动同步 + 变更时强制更新元数据
```

## 数仓数据质量规则引擎完整实现

```python
class DataQualityRuleEngine:
    """数据质量规则引擎：规则定义 + 自动检测 + 修复建议"""

    RULE_TYPES = {
        "not_null": "非空检查",
        "unique": "唯一性检查",
        "range": "值域检查",
        "referential": "引用完整性检查",
        "freshness": "新鲜度检查",
        "consistency": "一致性检查",
        "custom_sql": "自定义 SQL 检查",
    }

    def execute_rule(self, rule_id):
        """执行数据质量规则"""
        rule = self.db.get_quality_rule(rule_id)

        rule_type = rule["rule_type"]
        table_name = rule["table_name"]
        column_name = rule.get("column_name")

        result = {
            "rule_id": rule_id,
            "rule_name": rule["name"],
            "table": table_name,
            "column": column_name,
            "executed_at": now()
        }

        if rule_type == "not_null":
            total = self.db.count(f"{table_name}")
            null_count = self.db.query_one(
                f"SELECT COUNT(*) as c FROM {table_name} WHERE {column_name} IS NULL")["c"]
            result["total_rows"] = total
            result["failed_rows"] = null_count
            result["pass_rate"] = round((total - null_count) / max(total, 1), 4)

        elif rule_type == "unique":
            total = self.db.count(f"{table_name}")
            duplicates = self.db.query_one(
                f"SELECT COUNT(*) - COUNT(DISTINCT {column_name}) as c FROM {table_name}")["c"]
            result["total_rows"] = total
            result["failed_rows"] = duplicates
            result["pass_rate"] = round((total - duplicates) / max(total, 1), 4)

        elif rule_type == "range":
            params = json.loads(rule["rule_params"])
            out_of_range = self.db.query_one(
                f"SELECT COUNT(*) as c FROM {table_name} "
                f"WHERE {column_name} < %s OR {column_name} > %s",
                params["min"], params["max"])["c"]
            total = self.db.count(f"{table_name}")
            result["total_rows"] = total
            result["failed_rows"] = out_of_range
            result["pass_rate"] = round((total - out_of_range) / max(total, 1), 4)

        elif rule_type == "freshness":
            params = json.loads(rule["rule_params"])
            max_delay_hours = params.get("max_delay_hours", 24)
            latest = self.db.query_one(
                f"SELECT MAX({column_name}) as latest FROM {table_name}")["latest"]
            if latest:
                delay_hours = (now() - latest).total_seconds() / 3600
                result["latest_update"] = latest.isoformat()
                result["delay_hours"] = round(delay_hours, 1)
                result["pass_rate"] = 1.0 if delay_hours <= max_delay_hours else 0.0
            else:
                result["pass_rate"] = 0.0
                result["error"] = "表中无数据"

        elif rule_type == "consistency":
            params = json.loads(rule["rule_params"])
            # 检查两表关联一致性
            inconsistency = self.db.query_one(
                f"SELECT COUNT(*) as c FROM {table_name} t1 "
                f"LEFT JOIN {params['reference_table']} t2 "
                f"ON t1.{column_name} = t2.{params['reference_column']} "
                f"WHERE t2.{params['reference_column']} IS NULL")["c"]
            total = self.db.count(f"{table_name}")
            result["total_rows"] = total
            result["failed_rows"] = inconsistency
            result["pass_rate"] = round((total - inconsistency) / max(total, 1), 4)

        # 判定结果
        threshold = rule.get("pass_threshold", 0.99)
        result["status"] = "passed" if result.get("pass_rate", 0) >= threshold else "failed"

        # 记录结果
        self.db.insert("quality_rule_results", {
            "result_id": str(uuid4()),
            "rule_id": rule_id,
            "status": result["status"],
            "pass_rate": result.get("pass_rate"),
            "failed_rows": result.get("failed_rows", 0),
            "details": json.dumps(result),
            "executed_at": now()
        })

        # 失败 → 告警
        if result["status"] == "failed":
            self.alert(f"数据质量规则 {rule['name']} 未通过: 通过率 {result.get('pass_rate', 0):.2%}")

        return result

    def generate_repair_suggestions(self, rule_id, result):
        """生成修复建议"""
        rule = self.db.get_quality_rule(rule_id)
        suggestions = []

        if rule["rule_type"] == "not_null" and result["failed_rows"] > 0:
            suggestions.append({
                "action": "填充默认值",
                "sql": f"UPDATE {rule['table_name']} SET {rule['column_name']} = '未知' "
                       f"WHERE {rule['column_name']} IS NULL",
                "risk": "低"
            })

        elif rule["rule_type"] == "freshness" and result["status"] == "failed":
            suggestions.append({
                "action": "检查上游 ETL 作业",
                "reason": f"数据延迟 {result.get('delay_hours', 0)} 小时",
                "risk": "无"
            })

        elif rule["rule_type"] == "consistency" and result["failed_rows"] > 0:
            suggestions.append({
                "action": "清理孤儿记录",
                "sql": f"DELETE FROM {rule['table_name']} WHERE {rule['column_name']} NOT IN "
                       f"(SELECT {rule.get('reference_column', 'id')} FROM {rule.get('reference_table', '')})",
                "risk": "高 - 建议先备份"
            })

        return suggestions
```

## 异常场景补充

### 场景：质量规则误报

```
触发：范围规则设定 max=100 → 合法值 99.9 被四舍五入为 100 → 误报为越界
检测：
  1. 规则失败但人工检查数据正常 → 误报
  2. 同一规则误报率 > 10% → 规则需调整
处理：
  1. 调整规则阈值（100 → 100.5）
  2. 修复数据类型精度问题
  3. 标记误报
预防：阈值留余量 + 精度处理 + 误报标记
```

### 场景：质量规则执行影响查询性能

```
触发：在大表上执行 COUNT(DISTINCT) → 全表扫描 → 锁表 → 影响 ETL
检测：
  1. 规则执行时间 > 5 分钟 → 性能问题
  2. 规则执行期间 ETL 延迟 → 锁冲突
处理：
  1. 质量规则在只读副本执行
  2. 使用采样检查（10% 数据）替代全量
  3. 规则执行时间限制
预防：只读副本执行 + 采样检查 + 超时限制
```

## 数仓数据血缘可视化完整实现

```python
class DataLineageService:
    """数据血缘：表级血缘 + 字段级血缘 + 影响分析"""

    def parse_sql_lineage(self, sql, database="default"):
        """解析 SQL 生成血缘关系"""
        lineage = {"sources": [], "targets": [], "column_mappings": []}

        sql_upper = sql.strip().upper()

        # 解析目标表
        if sql_upper.startswith("INSERT INTO"):
            match = re.search(r'INSERT\s+INTO\s+(\w+(?:\.\w+)*)', sql, re.IGNORECASE)
            if match:
                lineage["targets"].append(match.group(1))

        elif sql_upper.startswith("CREATE TABLE") or sql_upper.startswith("CREATE OR REPLACE TABLE"):
            match = re.search(r'CREATE\s+(?:OR\s+REPLACE\s+)?TABLE\s+(\w+(?:\.\w+)*)', sql, re.IGNORECASE)
            if match:
                lineage["targets"].append(match.group(1))

        # 解析源表
        from_match = re.finditer(r'(?:FROM|JOIN)\s+(\w+(?:\.\w+)*)', sql, re.IGNORECASE)
        for m in from_match:
            table = m.group(1)
            if table.upper() not in ("SELECT", "WHERE", "ON", "AND", "OR", "AS"):
                lineage["sources"].append(table)

        # 解析字段映射
        select_match = re.search(r'SELECT\s+(.*?)\s+FROM', sql, re.IGNORECASE | re.DOTALL)
        if select_match:
            select_clause = select_match.group(1)
            for col_expr in self._split_select_columns(select_clause):
                mapping = self._parse_column_mapping(col_expr)
                if mapping:
                    lineage["column_mappings"].append(mapping)

        # 保存血缘
        for source in lineage["sources"]:
            for target in lineage["targets"]:
                self.db.insert("data_lineage", {
                    "lineage_id": str(uuid4()),
                    "source_table": source,
                    "target_table": target,
                    "sql_hash": hashlib.md5(sql.encode()).hexdigest(),
                    "column_mappings": json.dumps(lineage["column_mappings"]),
                    "database": database,
                    "created_at": now()
                })

        return lineage

    def impact_analysis(self, table_name, change_type="schema_change"):
        """影响分析：修改某表会影响哪些下游"""
        downstream = self._get_all_downstream(table_name, visited=set())
        impact = {
            "source_table": table_name,
            "change_type": change_type,
            "affected_tables": [],
            "affected_reports": [],
            "affected_pipelines": [],
            "total_impact_score": 0
        }

        for ds_table in downstream:
            # 表信息
            table_info = self.db.query_one(
                "SELECT * FROM table_metadata WHERE table_name = %s", ds_table)

            # 关联的报表
            reports = self.db.query(
                "SELECT * FROM report_dependencies WHERE table_name = %s", ds_table)

            # 关联的管道
            pipelines = self.db.query(
                "SELECT * FROM pipeline_dependencies WHERE table_name = %s", ds_table)

            impact["affected_tables"].append({
                "table": ds_table,
                "owner": table_info.get("owner") if table_info else "unknown",
                "sensitivity": table_info.get("sensitivity_level", "internal") if table_info else "unknown"
            })
            impact["affected_reports"].extend(reports)
            impact["affected_pipelines"].extend(pipelines)

        # 影响评分
        impact["total_impact_score"] = (
            len(impact["affected_tables"]) * 10 +
            len(impact["affected_reports"]) * 20 +
            len(impact["affected_pipelines"]) * 15
        )

        return impact

    def _get_all_downstream(self, table_name, visited):
        """递归获取所有下游表"""
        if table_name in visited:
            return []
        visited.add(table_name)

        direct = self.db.query(
            "SELECT DISTINCT target_table FROM data_lineage "
            "WHERE source_table = %s", table_name)

        all_downstream = []
        for d in direct:
            all_downstream.append(d["target_table"])
            all_downstream.extend(self._get_all_downstream(d["target_table"], visited))

        return all_downstream

    def _parse_column_mapping(self, col_expr):
        """解析字段映射"""
        col_expr = col_expr.strip()
        if " as " in col_expr.lower():
            parts = re.split(r'\s+as\s+', col_expr, flags=re.IGNORECASE)
            if len(parts) == 2:
                return {"expression": parts[0].strip(), "alias": parts[1].strip()}
        elif "." in col_expr:
            return {"source_column": col_expr.strip(), "target_column": col_expr.split(".")[-1].strip()}
        return {"expression": col_expr}

    def _split_select_columns(self, select_clause):
        """分割 SELECT 列（考虑括号嵌套）"""
        columns = []
        depth = 0
        current = ""
        for char in select_clause:
            if char == "(":
                depth += 1
            elif char == ")":
                depth -= 1
            elif char == "," and depth == 0:
                columns.append(current.strip())
                current = ""
                continue
            current += char
        if current.strip():
            columns.append(current.strip())
        return columns
```

## 异常场景补充

### 场景：血缘解析失败导致影响分析不全

```
触发：动态 SQL（表名来自参数）→ 静态解析无法识别源表 → 血缘缺失 → 影响分析漏报
检测：
  1. 查询使用了血缘中不存在的源表 → 血缘缺失
  2. 影响分析结果与实际不一致 → 血缘不全
处理：
  1. 动态 SQL 标记为手动维护血缘
  2. 运行时采集实际执行的源表（基于查询日志）
  3. 静态 + 动态血缘互补
预防：动态血缘采集 + 手动补充 + 静态动态互补
```

### 场景：循环依赖

```
触发：表 A 依赖 B，B 依赖 C，C 依赖 A → 递归查询死循环 → 影响分析超时
检测：
  1. 影响分析超过 10 秒 → 可能循环
  2. 数据血缘图中存在环 → 数据错误
处理：
  1. visited set 防止循环递归
  2. 检测环并告警
  3. 要求修正 ETL（消除循环依赖）
预防：visited set + 环检测 + ETL 规范
```

## 数仓 ETL 任务编排完整实现

```python
class ETLOrchestratorService:
    """ETL 编排：DAG 定义 + 依赖调度 + 失败重试 + 增量抽取"""

    def create_etl_pipeline(self, pipeline_name, tasks, schedule_cron=None):
        """创建 ETL 管道（DAG）"""
        pipeline_id = str(uuid4())

        # 1. 验证 DAG 无环
        graph = {t["task_id"]: t.get("depends_on", []) for t in tasks}
        if self._has_cycle(graph):
            raise CyclicDependencyError("任务依赖存在循环")

        # 2. 保存管道定义
        self.db.insert("etl_pipelines", {
            "pipeline_id": pipeline_id,
            "name": pipeline_name,
            "schedule_cron": schedule_cron,
            "status": "active",
            "task_count": len(tasks),
            "created_at": now()
        })

        # 3. 保存任务定义
        for task in tasks:
            self.db.insert("etl_tasks", {
                "task_id": task["task_id"],
                "pipeline_id": pipeline_id,
                "name": task["name"],
                "task_type": task["type"],  # extract / transform / load
                "config": json.dumps(task.get("config", {})),
                "depends_on": json.dumps(task.get("depends_on", [])),
                "max_retries": task.get("max_retries", 3),
                "timeout_seconds": task.get("timeout_seconds", 3600),
                "incremental_column": task.get("incremental_column"),
                "created_at": now()
            })

        return {"pipeline_id": pipeline_id, "task_count": len(tasks)}

    def execute_pipeline(self, pipeline_id, trigger_type="scheduled"):
        """执行 ETL 管道"""
        pipeline = self.db.get_pipeline(pipeline_id)
        tasks = self.db.query(
            "SELECT * FROM etl_tasks WHERE pipeline_id = %s", pipeline_id)

        run_id = str(uuid4())
        self.db.insert("etl_pipeline_runs", {
            "run_id": run_id,
            "pipeline_id": pipeline_id,
            "trigger_type": trigger_type,
            "status": "running",
            "started_at": now()
        })

        # 拓扑排序
        execution_order = self._topological_sort(tasks)

        completed = set()
        failed = False
        task_results = {}

        for task_id in execution_order:
            task = next(t for t in tasks if t["task_id"] == task_id)
            depends_on = json.loads(task.get("depends_on", "[]"))

            # 检查依赖是否都完成
            pending_deps = [d for d in depends_on if d not in completed]
            if pending_deps:
                self.db.insert("etl_task_runs", {
                    "run_id": str(uuid4()), "pipeline_run_id": run_id,
                    "task_id": task_id, "status": "skipped",
                    "reason": f"依赖未完成: {pending_deps}",
                    "executed_at": now()
                })
                continue

            # 如果前面有任务失败 → 跳过后续
            if failed:
                self.db.insert("etl_task_runs", {
                    "run_id": str(uuid4()), "pipeline_run_id": run_id,
                    "task_id": task_id, "status": "skipped",
                    "reason": "上游任务失败", "executed_at": now()
                })
                continue

            # 执行任务
            result = self._execute_task(run_id, task, task_results)
            task_results[task_id] = result

            if result["status"] == "success":
                completed.add(task_id)
            else:
                failed = True

        # 更新管道运行状态
        self.db.update("etl_pipeline_runs",
            {"status": "failed" if failed else "success",
             "completed_at": now()},
            {"run_id": run_id})

        return {"run_id": run_id, "status": "failed" if failed else "success",
                "completed_tasks": len(completed),
                "total_tasks": len(tasks)}

    def _execute_task(self, pipeline_run_id, task, previous_results):
        """执行单个 ETL 任务"""
        config = json.loads(task["config"])
        max_retries = task["max_retries"]

        for attempt in range(max_retries + 1):
            try:
                if task["task_type"] == "extract":
                    result = self._execute_extract(task, config, previous_results)
                elif task["task_type"] == "transform":
                    result = self._execute_transform(task, config, previous_results)
                elif task["task_type"] == "load":
                    result = self._execute_load(task, config, previous_results)
                else:
                    result = {"status": "failed", "error": f"未知任务类型: {task['task_type']}"}

                if result["status"] == "success":
                    self.db.insert("etl_task_runs", {
                        "run_id": str(uuid4()), "pipeline_run_id": pipeline_run_id,
                        "task_id": task["task_id"], "status": "success",
                        "rows_processed": result.get("rows", 0),
                        "attempt": attempt + 1,
                        "executed_at": now()
                    })
                    return result

            except Exception as e:
                if attempt < max_retries:
                    continue

                self.db.insert("etl_task_runs", {
                    "run_id": str(uuid4()), "pipeline_run_id": pipeline_run_id,
                    "task_id": task["task_id"], "status": "failed",
                    "error": str(e), "attempt": attempt + 1,
                    "executed_at": now()
                })
                return {"status": "failed", "error": str(e)}

        return {"status": "failed", "error": "max retries exceeded"}

    def _execute_extract(self, task, config, previous_results):
        """执行抽取任务（支持增量）"""
        source = config["source"]
        incremental_col = task.get("incremental_column")

        if incremental_col:
            # 增量抽取
            last_value = self.db.query_one(
                "SELECT last_value FROM etl_incremental_state "
                "WHERE task_id = %s", task["task_id"])
            if last_value:
                config["where"] = f"{incremental_col} > '{last_value['last_value']}'"

        rows = self.db.query(f"SELECT * FROM {source} "
            + (f" WHERE {config['where']}" if config.get("where") else "")
            + f" LIMIT {config.get('limit', 1000000)}")

        # 更新增量状态
        if incremental_col and rows:
            max_value = max(r[incremental_col] for r in rows)
            self.db.upsert("etl_incremental_state", {
                "task_id": task["task_id"],
                "last_value": str(max_value),
                "updated_at": now()
            }, conflict_columns=["task_id"])

        return {"status": "success", "rows": len(rows), "data": rows}

    def _topological_sort(self, tasks):
        """拓扑排序"""
        graph = {t["task_id"]: json.loads(t.get("depends_on", "[]")) for t in tasks}
        visited = set()
        order = []

        def dfs(node):
            if node in visited:
                return
            visited.add(node)
            for dep in graph.get(node, []):
                dfs(dep)
            order.append(node)

        for node in graph:
            dfs(node)

        return order

    def _has_cycle(self, graph):
        """检测有向图是否有环"""
        WHITE, GRAY, BLACK = 0, 1, 2
        color = {node: WHITE for node in graph}

        def dfs(node):
            color[node] = GRAY
            for neighbor in graph.get(node, []):
                if color.get(neighbor) == GRAY:
                    return True
                if color.get(neighbor, WHITE) == WHITE and dfs(neighbor):
                    return True
            color[node] = BLACK
            return False

        return any(dfs(node) for node in graph if color[node] == WHITE)
```

## 异常场景补充

### 场景：ETL 管道执行超时

```
触发：数据量突增 → Extract 任务执行 2 小时 → 超过 timeout → 管道失败 → 下游报表延迟
检测：
  1. 任务执行时间 > timeout_seconds → 超时
  2. 管道整体执行时间 > SLA → 延迟
处理：
  1. 增加任务超时时间
  2. 优化查询性能（索引/分区）
  3. 分批抽取（每次 10 万行）
预防：动态超时 + 查询优化 + 分批抽取
```

### 场景：增量抽取状态丢失

```
触发：增量状态表被误清空 → 下次全量抽取 → 数据重复 → 报表数据翻倍
检测：
  1. 抽取行数 > 平时的 5 倍 → 可能全量抽取
  2. 数据出现重复记录 → 增量状态问题
处理：
  1. 加载任务增加去重逻辑
  2. 从数据中恢复增量状态（取最大值）
  3. 重新运行管道
预防：去重逻辑 + 状态备份 + 异常检测
```

## 数仓实时入湖与 Schema 演进完整实现

```python
class SchemaEvolutionService:
    """Schema 演进：变更检测 → 兼容性验证 → 自动迁移 → 回滚"""

    SCHEMA_CHANGE_TYPES = {
        "add_column": "新增列（向后兼容）",
        "drop_column": "删除列（可能不兼容）",
        "rename_column": "重命名列（不兼容）",
        "change_type": "修改类型（可能不兼容）",
        "add_table": "新增表（兼容）",
        "drop_table": "删除表（不兼容）",
    }

    COMPATIBILITY_LEVELS = {
        "backward": "向后兼容（新 schema 可以读旧数据）",
        "forward": "向前兼容（旧 schema 可以读新数据）",
        "full": "完全兼容（同时向后和向前兼容）",
        "none": "不兼容",
    }

    def detect_schema_change(self, table_name, new_schema):
        """检测 Schema 变更"""
        current_schema = self._get_current_schema(table_name)

        if not current_schema:
            return {"type": "add_table", "changes": [],
                    "compatibility": "backward"}

        changes = []

        # 1. 检测新增列
        current_columns = {c["name"]: c for c in current_schema["columns"]}
        new_columns = {c["name"]: c for c in new_schema["columns"]}

        for col_name, col_def in new_columns.items():
            if col_name not in current_columns:
                changes.append({
                    "type": "add_column",
                    "column": col_name,
                    "definition": col_def,
                    "compatible": True,
                    "default_value": col_def.get("default")
                })

        # 2. 检测删除列
        for col_name in current_columns:
            if col_name not in new_columns:
                changes.append({
                    "type": "drop_column",
                    "column": col_name,
                    "compatible": False,
                    "affected_queries": self._find_queries_using_column(
                        table_name, col_name)
                })

        # 3. 检测类型变更
        for col_name, col_def in new_columns.items():
            if col_name in current_columns:
                old_type = current_columns[col_name]["type"]
                new_type = col_def["type"]

                if old_type != new_type:
                    compatible = self._check_type_compatibility(old_type, new_type)
                    changes.append({
                        "type": "change_type",
                        "column": col_name,
                        "old_type": old_type,
                        "new_type": new_type,
                        "compatible": compatible,
                        "data_loss_risk": not compatible
                    })

        # 4. 确定兼容性级别
        compatibility = self._determine_compatibility(changes)

        return {
            "table_name": table_name,
            "changes": changes,
            "change_count": len(changes),
            "compatibility": compatibility,
            "requires_migration": any(not c["compatible"] for c in changes)
        }

    def apply_schema_change(self, table_name, changes, strategy="safe"):
        """应用 Schema 变更"""
        migration_id = str(uuid4())

        # 1. 备份当前 Schema
        current_schema = self._get_current_schema(table_name)
        self.db.insert("schema_snapshots", {
            "snapshot_id": str(uuid4()),
            "table_name": table_name,
            "schema": json.dumps(current_schema),
            "created_at": now()
        })

        # 2. 按变更类型处理
        for change in changes:
            if change["type"] == "add_column":
                self._apply_add_column(table_name, change)

            elif change["type"] == "drop_column":
                if strategy == "safe":
                    # 安全策略：先重命名为 _deprecated，不删除
                    self._safe_drop_column(table_name, change)
                else:
                    self._apply_drop_column(table_name, change)

            elif change["type"] == "change_type":
                self._apply_type_change(table_name, change)

        # 3. 更新 Schema 注册
        self._update_schema_registry(table_name, changes)

        # 4. 记录迁移
        self.db.insert("schema_migrations", {
            "migration_id": migration_id,
            "table_name": table_name,
            "changes": json.dumps(changes),
            "strategy": strategy,
            "applied_at": now()
        })

        return {"migration_id": migration_id, "changes_applied": len(changes)}

    def rollback_schema(self, table_name, target_snapshot_id):
        """回滚 Schema"""
        snapshot = self.db.get_schema_snapshot(target_snapshot_id)
        old_schema = json.loads(snapshot["schema"])

        # 1. 重新应用旧 Schema
        current_schema = self._get_current_schema(table_name)

        # 计算逆向变更
        reverse_changes = self._compute_reverse_changes(
            current_schema, old_schema)

        # 2. 应用逆向变更
        for change in reverse_changes:
            if change["type"] == "add_column":
                # 逆向：删除列
                self._apply_drop_column(table_name, change)
            elif change["type"] == "drop_column":
                # 逆向：添加回列
                self._apply_add_column(table_name, change)
            elif change["type"] == "change_type":
                # 逆向：改回旧类型
                change["new_type"] = change["old_type"]
                change["old_type"] = change["new_type"]
                self._apply_type_change(table_name, change)

        return {"table_name": table_name, "rolled_back_to": target_snapshot_id}

    def _apply_add_column(self, table_name, change):
        """添加列"""
        col_name = change["column"]
        col_type = change["definition"]["type"]
        default = change.get("default_value", "NULL")

        self.hive_client.execute(
            f"ALTER TABLE {table_name} ADD COLUMNS "
            f"({col_name} {col_type} COMMENT 'auto-added')")

    def _safe_drop_column(self, table_name, change):
        """安全删除列（重命名）"""
        col_name = change["column"]
        self.hive_client.execute(
            f"ALTER TABLE {table_name} CHANGE COLUMN "
            f"{col_name} _deprecated_{col_name} STRING")

    def _apply_drop_column(self, table_name, change):
        """实际删除列（需重建表）"""
        # Hive 不支持直接删除列 → 需要重建
        self.alert(f"删除列 {change['column']} 需要重建表 {table_name}")

    def _apply_type_change(self, table_name, change):
        """类型变更"""
        col_name = change["column"]
        new_type = change.get("new_type", change["definition"]["type"])
        self.hive_client.execute(
            f"ALTER TABLE {table_name} CHANGE COLUMN "
            f"{col_name} {col_name} {new_type}")

    def _check_type_compatibility(self, old_type, new_type):
        """检查类型兼容性"""
        safe_conversions = {
            ("INT", "BIGINT"), ("FLOAT", "DOUBLE"),
            ("STRING", "VARCHAR"), ("INT", "STRING"),
        }
        return (old_type, new_type) in safe_conversions or old_type == new_type

    def _determine_compatibility(self, changes):
        """确定兼容性级别"""
        if all(c["compatible"] for c in changes):
            return "full"
        elif any(c["type"] == "drop_column" for c in changes):
            return "backward"
        elif any(c["type"] == "change_type" and not c["compatible"] for c in changes):
            return "none"
        else:
            return "forward"

    def _get_current_schema(self, table_name):
        """获取当前 Schema"""
        result = self.db.query_one(
            "SELECT schema FROM schema_registry "
            "WHERE table_name = %s ORDER BY version DESC LIMIT 1",
            table_name)
        return json.loads(result["schema"]) if result else None

    def _find_queries_using_column(self, table_name, column_name):
        """查找使用某列的查询"""
        queries = self.db.query(
            "SELECT query_id, query_text FROM saved_queries "
            "WHERE query_text LIKE %s",
            f"%{table_name}%{column_name}%")
        return [{"query_id": q["query_id"]} for q in queries]

    def _update_schema_registry(self, table_name, changes):
        """更新 Schema 注册"""
        current = self._get_current_schema(table_name)
        version = (current.get("version", 0) + 1) if current else 1

        # 应用变更到 schema
        new_schema = dict(current) if current else {"table_name": table_name, "columns": []}

        for change in changes:
            if change["type"] == "add_column":
                new_schema["columns"].append(change["definition"])
            elif change["type"] == "drop_column":
                new_schema["columns"] = [c for c in new_schema["columns"]
                                        if c["name"] != change["column"]]

        new_schema["version"] = version

        self.db.insert("schema_registry", {
            "table_name": table_name,
            "version": version,
            "schema": json.dumps(new_schema),
            "changes": json.dumps(changes),
            "registered_at": now()
        })

    def _compute_reverse_changes(self, current_schema, target_schema):
        """计算逆向变更"""
        changes = []
        current_cols = {c["name"]: c for c in current_schema.get("columns", [])}
        target_cols = {c["name"]: c for c in target_schema.get("columns", [])}

        for name in target_cols:
            if name not in current_cols:
                changes.append({"type": "add_column", "column": name,
                               "definition": target_cols[name]})

        for name in current_cols:
            if name not in target_cols:
                changes.append({"type": "drop_column", "column": name})

        return changes
```

## 异常场景补充

### 场景：Schema 变更导致下游查询失败

```
触发：删除了 order_amount 列 → 下游报表依赖该列 → 查询报错 → 报表中断
检测：
  1. 下游查询报错率上升 → Schema 不兼容
  2. 删除列前未通知下游 → 沟通不足
处理：
  1. Schema 变更前影响分析（查找依赖该列的查询）
  2. 安全删除（先重命名为 _deprecated）
  3. 下游确认后再真正删除
预防：影响分析 + 安全删除 + 确认流程
```

### 场景：Schema 演进与实时数据竞争

```
触发：Schema 变更正在执行 → 实时数据按新格式写入 → 部分分区旧格式 → 混合格式 → 解析错误
检测：
  1. 同一表不同分区格式不一致 → Schema 混合
  2. 数据解析错误率上升 → 格式问题
处理：
  1. Schema 变更在分区边界执行
  2. 新分区使用新 Schema，旧分区保持不变
  3. 读取时按分区 Schema 版本解析
预防：分区边界变更 + 版本化解析 + 兼容读取
```

## 数仓数据血缘追踪完整实现

```python
class DataLineageService:
    """数据血缘：血缘采集 → 依赖图谱 → 影响分析 → 变更评估"""

    def collect_lineage(self, job_execution):
        """采集作业执行产生的血缘关系"""
        job_id = job_execution["job_id"]
        inputs = job_execution.get("input_tables", [])
        outputs = job_execution.get("output_tables", [])
        transformations = job_execution.get("transformations", [])

        lineage_id = str(uuid4())

        # 1. 记录表级血缘
        for input_table in inputs:
            for output_table in outputs:
                self.db.insert("table_lineage", {
                    "lineage_id": lineage_id,
                    "source_table": input_table["full_name"],
                    "source_db": input_table["database"],
                    "target_table": output_table["full_name"],
                    "target_db": output_table["database"],
                    "job_id": job_id,
                    "job_name": job_execution["job_name"],
                    "transformation_type": self._classify_transformation(
                        input_table, output_table, transformations),
                    "execution_time": job_execution.get("duration_ms"),
                    "row_count_read": input_table.get("row_count"),
                    "row_count_written": output_table.get("row_count"),
                    "recorded_at": now()
                })

        # 2. 记录列级血缘（如果可用）
        for transform in transformations:
            if transform.get("column_mapping"):
                for source_col, target_col in transform["column_mapping"].items():
                    self.db.insert("column_lineage", {
                        "lineage_id": str(uuid4()),
                        "source_table": transform["source_table"],
                        "source_column": source_col,
                        "target_table": transform["target_table"],
                        "target_column": target_col,
                        "transformation": transform.get("expression"),
                        "job_id": job_id,
                        "recorded_at": now()
                    })

        # 3. 更新血缘图缓存
        self._update_lineage_graph_cache(inputs, outputs)

        return {"lineage_id": lineage_id, "edges_recorded": len(inputs) * len(outputs)}

    def analyze_impact(self, table_name, change_type="schema_change"):
        """影响分析：某表变更会影响哪些下游"""
        # 1. 查找直接下游
        direct_downstream = self.db.query(
            "SELECT DISTINCT target_table, target_db, job_name "
            "FROM table_lineage WHERE source_table = %s "
            "ORDER BY target_table", table_name)

        # 2. 递归查找间接下游（BFS）
        all_downstream = []
        visited = {table_name}
        queue = [table_name]
        depth = 0
        max_depth = 10

        while queue and depth < max_depth:
            depth += 1
            next_queue = []

            for current in queue:
                children = self.db.query(
                    "SELECT DISTINCT target_table FROM table_lineage "
                    "WHERE source_table = %s", current)

                for child in children:
                    child_name = child["target_table"]
                    if child_name not in visited:
                        visited.add(child_name)
                        next_queue.append(child_name)

                        # 获取影响路径
                        path = self._find_impact_path(table_name, child_name)

                        all_downstream.append({
                            "table": child_name,
                            "depth": depth,
                            "impact_path": path,
                            "affected_jobs": self._get_affected_jobs(child_name)
                        })

            queue = next_queue

        # 3. 按变更类型评估影响
        impact_assessment = self._assess_impact_severity(
            table_name, change_type, all_downstream)

        return {
            "source_table": table_name,
            "change_type": change_type,
            "direct_downstream_count": len(direct_downstream),
            "total_downstream_count": len(all_downstream),
            "downstream_tables": all_downstream,
            "impact_assessment": impact_assessment,
            "direct_downstream": [{"table": d["target_table"],
                                   "database": d["target_db"],
                                   "job": d["job_name"]}
                                  for d in direct_downstream]
        }

    def find_root_cause(self, table_name, anomaly_time):
        """根因定位：某表数据异常，追溯上游根因"""
        # 1. 查找直接上游
        direct_upstream = self.db.query(
            "SELECT DISTINCT source_table, source_db, job_name "
            "FROM table_lineage WHERE target_table = %s "
            "ORDER BY source_table", table_name)

        # 2. 检查上游数据是否异常
        suspects = []

        for upstream in direct_upstream:
            upstream_table = upstream["source_table"]

            # 检查上游表的最近作业执行情况
            recent_jobs = self.db.query(
                "SELECT * FROM job_executions "
                "WHERE job_name IN ("
                "  SELECT DISTINCT job_name FROM table_lineage "
                "  WHERE source_table = %s) "
                "AND executed_at BETWEEN %s AND %s "
                "ORDER BY executed_at DESC",
                upstream_table,
                anomaly_time - timedelta(hours=24),
                anomaly_time)

            for job in recent_jobs:
                if job["status"] == "failed":
                    suspects.append({
                        "table": upstream_table,
                        "reason": "job_failed",
                        "job_id": job["job_id"],
                        "job_name": job["job_name"],
                        "failed_at": job["executed_at"],
                        "error": job.get("error_message"),
                        "confidence": 0.9
                    })
                elif job.get("row_count_written", 0) == 0:
                    suspects.append({
                        "table": upstream_table,
                        "reason": "zero_rows_written",
                        "job_id": job["job_id"],
                        "confidence": 0.8
                    })
                elif job.get("row_count_written", 0) < job.get("row_count_read", 1) * 0.1:
                    suspects.append({
                        "table": upstream_table,
                        "reason": "significant_data_loss",
                        "expected_rows": job["row_count_read"],
                        "actual_rows": job["row_count_written"],
                        "confidence": 0.7
                    })

        # 3. 递归检查更上游
        if not suspects:
            visited = {table_name}
            queue = [u["source_table"] for u in direct_upstream]

            while queue and not suspects:
                current = queue.pop(0)
                if current in visited:
                    continue
                visited.add(current)

                further_upstream = self.db.query(
                    "SELECT DISTINCT source_table FROM table_lineage "
                    "WHERE target_table = %s", current)

                for fu in further_upstream:
                    queue.append(fu["source_table"])

                    # 检查数据质量
                    quality = self.data_quality.check_data_freshness(fu["source_table"])
                    if quality.get("is_stale"):
                        suspects.append({
                            "table": fu["source_table"],
                            "reason": "stale_data",
                            "last_fresh_time": quality.get("last_fresh_time"),
                            "confidence": 0.5
                        })

        # 4. 按置信度排序
        suspects.sort(key=lambda s: s["confidence"], reverse=True)

        return {
            "anomaly_table": table_name,
            "anomaly_time": anomaly_time.isoformat(),
            "suspect_count": len(suspects),
            "suspects": suspects,
            "upstream_chain": [{"table": u["source_table"],
                               "database": u["source_db"]}
                              for u in direct_upstream]
        }

    def _classify_transformation(self, input_table, output_table, transformations):
        """分类转换类型"""
        if not transformations:
            return "direct_copy"

        types = set()
        for t in transformations:
            if t.get("type") == "filter":
                types.add("filter")
            elif t.get("type") == "aggregate":
                types.add("aggregate")
            elif t.get("type") == "join":
                types.add("join")
            elif t.get("type") == "derive":
                types.add("derive")

        return "+".join(types) if types else "unknown"

    def _find_impact_path(self, source, target):
        """查找影响路径（BFS）"""
        visited = {source}
        queue = [(source, [source])]

        while queue:
            current, path = queue.pop(0)

            children = self.db.query(
                "SELECT DISTINCT target_table FROM table_lineage "
                "WHERE source_table = %s", current)

            for child in children:
                new_path = path + [child["target_table"]]

                if child["target_table"] == target:
                    return new_path

                if child["target_table"] not in visited:
                    visited.add(child["target_table"])
                    queue.append((child["target_table"], new_path))

        return []

    def _get_affected_jobs(self, table_name):
        """获取受影响的作业"""
        return self.db.query(
            "SELECT DISTINCT job_name FROM table_lineage "
            "WHERE source_table = %s", table_name)

    def _assess_impact_severity(self, table_name, change_type, downstream):
        """评估影响严重程度"""
        critical_tables = self._get_critical_tables()

        affected_critical = [d for d in downstream
                            if d["table"] in critical_tables]

        severity = "low"
        if change_type == "drop_table":
            severity = "critical"
        elif change_type == "schema_change" and len(downstream) > 20:
            severity = "high"
        elif change_type == "schema_change" and affected_critical:
            severity = "high"
        elif len(downstream) > 10:
            severity = "medium"

        return {
            "severity": severity,
            "affected_critical_tables": len(affected_critical),
            "total_affected_tables": len(downstream),
            "recommended_action": self._get_recommended_action(severity, change_type)
        }

    def _get_critical_tables(self):
        """获取关键表列表"""
        return set(r["table_name"] for r in self.db.query(
            "SELECT table_name FROM critical_tables WHERE status = 'active'"))

    def _get_recommended_action(self, severity, change_type):
        """获取建议操作"""
        actions = {
            "critical": "暂停变更，需 CTO 审批",
            "high": "需数据团队负责人审批，提前 48 小时通知下游",
            "medium": "需团队负责人审批，提前 24 小时通知下游",
            "low": "正常执行，通知下游团队"
        }
        return actions.get(severity, "正常执行")

    def _update_lineage_graph_cache(self, inputs, outputs):
        """更新血缘图缓存"""
        for inp in inputs:
            for out in outputs:
                self.redis.sadd(f"lineage_downstream:{inp['full_name']}",
                    out["full_name"])
                self.redis.sadd(f"lineage_upstream:{out['full_name']}",
                    inp["full_name"])
```

## 异常场景补充

### 场景：血缘信息缺失导致影响分析不完整

```
触发：某 ETL 作业未采集血缘 → 下游表未被发现 → 变更后下游报表报错 → 未知影响
检测：
  1. 血缘图中断（表有数据但无上游来源记录）→ 血缘缺失
  2. SQL 解析发现表引用但血缘中无记录 → 漏采
处理：
  1. 定期扫描 SQL 解析补充缺失血缘
  2. 作业执行强制采集血缘
  3. 缺失血缘标记为"未验证"
预防：SQL 解析补充 + 强制采集 + 未验证标记
```

### 场景：循环依赖导致影响分析死循环

```
触发：A → B → C → A（ETL 互相依赖）→ BFS 遍历无限循环 → 分析超时
检测：
  1. 血缘图中存在环 → 循环依赖
  2. 影响分析遍历深度超过 20 → 可能循环
处理：
  1. BFS 遍历使用 visited 集合防止重复
  2. 设置最大遍历深度
  3. 检测到循环依赖 → 告警 + 人工修复
预防：visited 集合 + 深度限制 + 循环检测
```

### 场景：根因定位指向错误的上游

```
触发：数据异常 → 根因分析指向上游表 A → 但实际原因是表 B 的数据质量 → 误判
检测：
  1. 修复 A 后异常未消除 → 根因错误
  2. 多个上游同时有问题 → 需要更精确的因果推断
处理：
  1. 根因分析结合数据质量检查（不只看作业状态）
  2. 时序关联分析（异常时间与上游变更时间的相关性）
  3. 人工确认根因
预防：数据质量检查 + 时序关联 + 人工确认
```
