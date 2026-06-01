# C05: IoT设备海量时序数据

## 业务场景

某工业物联网平台，接入 10 万台工厂设备（传感器、电机、PLC 等），每台设备每秒上报 10-50 个指标数据点。这些数据用于实时监控、异常告警和历史趋势分析。

**已知数据：**
- 设备数：10 万台
- 每台设备上报频率：1 次/秒
- 每次上报数据点数：平均 20 个（温度、压力、转速、电流等）
- 总写入速率：10 万 × 20 = 200 万数据点/秒
- 数据保留策略：
  - 原始数据保留 30 天（用于故障排查）
  - 1 分钟聚合保留 1 年（用于趋势分析）
  - 1 小时聚合永久保留（用于长期统计）
- 查询场景：
  - 实时仪表盘（最近 1 小时原始数据）——延迟 < 100ms
  - 历史趋势图（1 天-1 年聚合数据）——延迟 < 500ms
  - 异常告警阈值比对 ——延迟 < 5 秒

**设备离线场景：** 设备可能因网络故障离线，重新上线后批量补报离线期间的数据（可能补报数小时的数据）。

**为什么不能简单地把数据存到 MySQL？**

200 万点/秒的持续写入，MySQL 单机写入上限约 5000 QPS，需要 400 台 MySQL 才能扛住写入。而且时序查询（按时间范围+设备ID查询）在 MySQL 中效率低下——没有针对时序数据的压缩和分区优化。

## 核心挑战

### 挑战 1：持续写入 200 万点/秒

这不是一次性的流量洪峰（如秒杀），而是 24×7 的持续写入。任何写入瓶颈都会导致数据积压，进而导致告警延迟。

### 挑战 2：降采样不是简单的"取平均值"

1 分钟聚合时，如果只保留平均值，会丢失异常峰值信息：

```
温度传感器 1 分钟内上报 60 个值：
  [85, 85, 85, 120, 85, 85, ...]  ← 有一个 120° 的异常峰值

平均值：(85×59 + 120) / 60 = 85.58° ← 几乎看不到异常
最大值：120° ← 清楚地显示了异常
```

工业设备的异常检测依赖极值，平均值会"抹平"异常。

### 挑战 3：冷热数据的分层存储

30 天原始数据 + 1 年分钟聚合 + 永久小时聚合 = 海量数据：

```
原始数据（30天）：200万点/秒 × 86400秒/天 × 30天 = 5.18 万亿个数据点
  每个点约 40 字节 → 约 190TB

1分钟聚合（1年）：10万台 × 20指标 × 525600分钟/年 = 1.05 万亿个聚合点
  每个点约 60 字节 → 约 60TB

1小时聚合（永久）：10万台 × 20指标 × 8760小时/年 × N年
  每年约 1TB
```

190TB 的原始数据如果全存 SSD，成本约 ¥19 万/月（¥1/GB/月）。必须做冷热分层。

### 挑战 4：设备离线补写的数据一致性

设备离线 2 小时后批量补报 7200 条数据（2小时 × 3600秒），这些数据需要：
- 写入原始数据表
- 重新计算受影响时间段的 1 分钟聚合
- 确保查询到的是包含补写数据后的正确结果

## 设计约束

- 时序数据库可选：TimescaleDB / InfluxDB / TDengine
- 成本预算：存储成本 < ¥0.5/GB/月
- 告警延迟：< 5 秒（从数据上报到触发告警）
- 补写数据必须参与聚合重算

## 请先独立思考（限时 40 分钟）

1. 时序数据库选型：TimescaleDB vs InfluxDB vs TDengine，从写入性能、查询能力、SQL兼容性、运维成本四个维度对比。
2. 降采样策略：1 分钟聚合应该保留哪些统计量？为什么不能只保留平均值？
3. 设备离线 2 小时后补写 7200 条数据，这些数据如何正确地参与 1 分钟聚合的重算？
4. 190TB 原始数据的冷热分层方案：什么数据放 SSD、什么放 HDD、什么放对象存储？

---

## 设计解析

### 时序数据库选型

| 维度 | TimescaleDB | InfluxDB | TDengine |
|------|-------------|----------|----------|
| 底层 | PostgreSQL 扩展 | 自研存储引擎 | 自研存储引擎 |
| SQL 兼容 | 完整 PostgreSQL SQL | InfluxQL（类SQL）| 类SQL |
| 写入性能 | ~10万行/秒/节点 | ~50万点/秒/节点 | ~100万点/秒/节点 |
| 降采样 | Continuous Aggregates（内置） | Task（需配置） | 自动降采样 |
| 压缩 | 行级压缩（~90%压缩率） | Gorilla压缩 | 列存压缩 |
| 生态 | PostgreSQL 生态（成熟） | InfluxDB 生态 | 国产，生态较新 |
| 运维 | 复杂（需管理 PostgreSQL） | 中等 | 简单 |
| 集群 | 分布式 Hypertable（2.x+） | 付费版才支持 | 原生支持 |

**选择：TimescaleDB**，理由：
1. SQL 兼容——业务开发无需学习新查询语言
2. Continuous Aggregates 原生支持降采样——无需额外管道
3. PostgreSQL 生态成熟——连接池、备份、监控工具齐全
4. 分布式 Hypertable 支持水平扩展

**但需要注意：** TimescaleDB 的单节点写入性能不如 TDengine。200 万点/秒需要 3-5 节点集群。

### 整体架构

```
设备 → MQTT Broker(EMQX) → Kafka → Flink(实时告警) → TimescaleDB(Hot/SSD)
                                               ↓
                                     Continuous Aggregates
                                               ↓
                                     TimescaleDB(Warm/HDD)
                                               ↓
                                     对象存储(S3/OSS)(Cold)
```

### 写入管道：从设备到数据库

**1. MQTT 接入层**

10 万台设备通过 MQTT 协议上报数据。EMQX 集群（3 节点）承载 10 万连接。

每条消息格式：
```json
{
  "device_id": "PLC-A001",
  "timestamp": 1715000000,
  "metrics": {
    "temperature": 85.3,
    "pressure": 2.1,
    "rpm": 1500,
    "current": 12.5,
    "voltage": 380.2,
    "vibration": 0.03
  }
}
```

**2. Kafka 缓冲层**

- Topic 按设备类型分区：`iot-data-plc`、`iot-data-sensor`、`iot-data-motor`
- 分区数 = 20（保证并行度）
- 消息保留 3 天（为离线补写提供缓冲窗口）
- 200 万点/秒 ≈ 10 万行/秒（每行含 20 个指标列），Kafka 单集群轻松承受

**3. Flink 实时告警层**

```java
DataStream<DeviceMetric> stream = env
    .addSource(new FlinkKafkaConsumer<>("iot-data-*", ...))
    .keyBy(metric -> metric.deviceId)
    .process(new AlertFunction());

class AlertFunction extends KeyedProcessFunction<String, DeviceMetric, Alert> {
    @Override
    public void processElement(DeviceMetric metric, Context ctx, Collector<Alert> out) {
        // 检查告警规则
        List<AlertRule> rules = ruleEngine.getRules(metric.deviceId);
        
        for (AlertRule rule : rules) {
            if (rule.check(metric)) {
                out.collect(new Alert(metric.deviceId, rule.name, metric.timestamp));
            }
        }
        
        // 同时将数据写入 TimescaleDB
        timescaleDBSink.write(metric);
    }
}
```

**4. TimescaleDB 批量写入**

```python
class TimescaleDBSink:
    def __init__(self):
        self.buffer = []
        self.buffer_size = 1000
        self.flush_interval = 0.1  # 100ms

    def write(self, metric):
        self.buffer.append(metric)
        if len(self.buffer) >= self.buffer_size:
            self.flush()

    def flush(self):
        # 批量 INSERT
        values = []
        for m in self.buffer:
            values.append(
                f"('{m.device_id}', {m.timestamp}, "
                f"{m.temperature}, {m.pressure}, {m.rpm}, "
                f"{m.current}, {m.voltage}, {m.vibration})"
            )

        sql = f"""
            INSERT INTO device_metrics 
            (time, device_id, temperature, pressure, rpm, current, voltage, vibration)
            VALUES {','.join(values)}
        """
        self.db.execute(sql)
        self.buffer = []
```

**写入性能：** 批量 1000 条 INSERT，单次约 50ms → 吞吐约 2 万行/秒/连接。10 个并行连接 → 20 万行/秒。200 万点/秒 = 约 10 万行/秒，5 个并行连接即可满足。

### 数据表设计

```sql
-- 设备元数据（维度表）
CREATE TABLE devices (
    device_id VARCHAR(50) PRIMARY KEY,
    device_type VARCHAR(30),       -- PLC / Sensor / Motor
    factory_id VARCHAR(20),
    workshop VARCHAR(50),
    location VARCHAR(100),
    installed_at TIMESTAMP,
    metadata JSONB                 -- 设备型号、厂商、额定参数等
);

-- 时序数据表（Hypertable —— TimescaleDB 的分区表）
CREATE TABLE device_metrics (
    time        TIMESTAMPTZ NOT NULL,
    device_id   VARCHAR(50) NOT NULL,
    temperature FLOAT8,
    pressure    FLOAT8,
    rpm         FLOAT8,
    current     FLOAT8,
    voltage     FLOAT8,
    vibration   FLOAT8,
    FOREIGN KEY (device_id) REFERENCES devices(device_id)
);

-- 转换为 Hypertable（按 time 分区，device_id 作为空间分区键）
SELECT create_hypertable('device_metrics', 'time', 'device_id', 3);

-- 7 天后自动压缩（减少 90%+ 存储空间）
SELECT add_compression_policy('device_metrics', INTERVAL '7 days');

-- 30 天后自动删除原始数据分区
SELECT add_retention_policy('device_metrics', INTERVAL '30 days');
```

**告警记录表：**

```sql
-- 告警记录表（普通 PostgreSQL 表，非 Hypertable）
CREATE TABLE alerts (
    id              BIGSERIAL PRIMARY KEY,
    device_id       VARCHAR(50) NOT NULL REFERENCES devices(device_id),
    alert_type      VARCHAR(50) NOT NULL,       -- temperature_high / vibration_anomaly / current_spike
    severity        VARCHAR(20) NOT NULL,       -- critical / warning / info
    threshold_value FLOAT8,                     -- 触发阈值
    actual_value    FLOAT8,                     -- 实际值
    message         TEXT,                       -- 告警描述
    status          VARCHAR(20) DEFAULT 'firing', -- firing / acknowledged / resolved
    fired_at        TIMESTAMPTZ NOT NULL,       -- 触发时间
    resolved_at     TIMESTAMPTZ,                -- 恢复时间
    acknowledged_by VARCHAR(50),                -- 确认人
    metadata        JSONB                       -- 扩展信息（规则ID、关联指标等）
);

-- 告警表索引
CREATE INDEX idx_alerts_device_time ON alerts(device_id, fired_at DESC);
CREATE INDEX idx_alerts_severity_status ON alerts(severity, status) WHERE status = 'firing';
CREATE INDEX idx_alerts_fired_at ON alerts(fired_at DESC);

-- 告警恢复时自动更新状态
CREATE OR REPLACE FUNCTION auto_resolve_alert()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE alerts
    SET status = 'resolved', resolved_at = NOW()
    WHERE device_id = NEW.device_id
      AND alert_type = NEW.alert_type
      AND severity = NEW.severity
      AND status = 'firing'
      AND fired_at < NEW.fired_at;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**降采样规则配置表：**

```sql
-- 降采样规则配置表
CREATE TABLE downsampling_rules (
    id              SERIAL PRIMARY KEY,
    source_table    VARCHAR(100) NOT NULL,     -- 源表名
    target_view     VARCHAR(100) NOT NULL,     -- 目标物化视图名
    bucket_interval INTERVAL NOT NULL,         -- 聚合时间窗口
    retention       INTERVAL NOT NULL,         -- 数据保留时长
    metrics_config  JSONB NOT NULL,            -- 聚合指标配置
    enabled         BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- 插入默认降采样规则
INSERT INTO downsampling_rules (source_table, target_view, bucket_interval, retention, metrics_config) VALUES
('device_metrics', 'device_metrics_1min',  INTERVAL '1 minute', INTERVAL '1 year',
 '{"temperature": ["avg","max","min","count"], "pressure": ["avg","max","min"], "rpm": ["avg","max","min"], "current": ["avg","max","min","count"], "voltage": ["avg","min","max"], "vibration": ["avg","max","min","percentile_95"]}'),
('device_metrics_1min', 'device_metrics_1hour', INTERVAL '1 hour', INTERVAL '100 years',
 '{"avg_temp": ["avg","max","min"], "max_temp": ["max"], "min_temp": ["min"], "avg_pressure": ["avg","max","min"], "max_pressure": ["max"], "min_pressure": ["min"], "avg_rpm": ["avg","max","min"], "max_rpm": ["max"], "min_rpm": ["min"]}');
```

**Hypertable 详细配置：**

```sql
-- 原始数据 Hypertable 的分区配置
SELECT create_hypertable('device_metrics', 'time',
    partitioning_column => 'device_id',
    number_partitions => 3,
    chunk_time_interval => INTERVAL '1 day'   -- 每天一个分区，方便按天删除
);

-- 1 分钟聚合也设为 Hypertable（TimescaleDB 自动处理）
-- 但可以调整分区大小以优化查询
-- 查看当前分区信息
SELECT hypertable_name, num_chunks, chunk_interval
FROM timescaledb_information.hypertables;

-- 设置并行查询（2+ 并行 worker 加速大范围查询）
ALTER TABLE device_metrics SET (timescaledb.compress_segmentby = 'device_id');
ALTER TABLE device_metrics SET (timescaledb.compress_orderby = 'time DESC');

-- 压缩后的段按 device_id 分组，按 time DESC 排序
-- 这样查询单设备时只需解压对应的 segment，而非整个 chunk

-- 为常用查询模式创建索引
CREATE INDEX idx_metrics_device_time ON device_metrics(device_id, time DESC);
CREATE INDEX idx_metrics_time ON device_metrics(time DESC);

-- 跳过索引（Skip Scan）支持 DISTINCT ON 查询的最新值检索
-- TimescaleDB 2.x 会自动利用此类索引优化 "最新值" 查询
```

**为什么分区键是 `(time, device_id)` 而不是只按 time？**

- 只按 time 分区：查询某设备最近 1 小时的数据，需要扫描所有分区的该设备数据
- 按 time + device_id 分区：查询时直接定位到特定分区，减少扫描量
- device_id 的分区数设为 3，意味着每个时间分片再按 device_id 哈希分成 3 份

### 降采样策略：保留极值

**1 分钟聚合：**

```sql
CREATE MATERIALIZED VIEW device_metrics_1min
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 minute', time) AS bucket,
    device_id,
    -- 保留 avg + max + min + count，不丢失异常峰值
    avg(temperature)   AS avg_temp,
    max(temperature)   AS max_temp,
    min(temperature)   AS min_temp,
    count(temperature) AS count_temp,
    avg(pressure)      AS avg_pressure,
    max(pressure)      AS max_pressure,
    min(pressure)      AS min_pressure,
    avg(rpm)           AS avg_rpm,
    max(rpm)           AS max_rpm,
    min(rpm)           AS min_rpm,
    avg(current)       AS avg_current,
    max(current)       AS max_current,
    min(current)       AS min_current
FROM device_metrics
GROUP BY bucket, device_id
WITH NO DATA;  -- 不立即填充，从创建时刻开始增量计算

-- 1 年后删除 1 分钟聚合数据
SELECT add_retention_policy('device_metrics_1min', INTERVAL '1 year');
```

**1 小时聚合（从 1 分钟聚合进一步降采样）：**

```sql
CREATE MATERIALIZED VIEW device_metrics_1hour
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', bucket) AS bucket,
    device_id,
    avg(avg_temp)    AS avg_temp,
    max(max_temp)    AS max_temp,
    min(min_temp)    AS min_temp,
    avg(avg_pressure) AS avg_pressure,
    max(max_pressure) AS max_pressure,
    min(min_pressure) AS min_pressure,
    avg(avg_rpm)     AS avg_rpm,
    max(max_rpm)     AS max_rpm,
    min(min_rpm)     AS min_rpm,
    avg(avg_current) AS avg_current,
    max(max_current) AS max_current,
    min(min_current) AS min_current,
    sum(count_temp)  AS total_samples  -- 累计采样数，判断数据完整性
FROM device_metrics_1min
GROUP BY bucket, device_id
WITH NO DATA;
```

**多级降采样的完整实现逻辑：**

```python
"""
多级降采样调度器
负责管理从原始数据到各级聚合的完整降采样管道
"""
import logging
from datetime import datetime, timedelta
from typing import List, Dict, Any

logger = logging.getLogger(__name__)


class DownsamplingManager:
    """多级降采样管理器"""

    # 降采样层级定义
    LEVELS = [
        {
            "name": "1min",
            "source": "device_metrics",
            "view": "device_metrics_1min",
            "bucket": timedelta(minutes=1),
            "retention": timedelta(days=365),
            "refresh_lag": timedelta(minutes=5),       # 延迟5分钟刷新，确保数据到齐
            "max_refresh_window": timedelta(hours=3),   # 单次最大刷新窗口
        },
        {
            "name": "1hour",
            "source": "device_metrics_1min",
            "view": "device_metrics_1hour",
            "bucket": timedelta(hours=1),
            "retention": timedelta(days=36500),         # 约100年，即永久保留
            "refresh_lag": timedelta(hours=1),          # 延迟1小时刷新
            "max_refresh_window": timedelta(days=3),
        },
    ]

    def __init__(self, db_connection_pool):
        self.db = db_connection_pool

    def refresh_level(self, level: Dict[str, Any],
                      start: datetime, end: datetime) -> int:
        """
        刷新指定层级的聚合数据
        返回受影响的 bucket 数量
        """
        # 限制单次刷新窗口，避免长事务
        window = level["max_refresh_window"]
        actual_end = min(end, start + window)

        sql = """
            CALL refresh_continuous_aggregate(
                %s, %s, %s
            );
        """
        with self.db.getconn() as conn:
            cursor = conn.cursor()
            cursor.execute(sql, [
                level["view"],
                start.isoformat(),
                actual_end.isoformat(),
            ])
            conn.commit()

            # 统计受影响的 bucket 数
            bucket_sql = f"""
                SELECT count(*)
                FROM {level["view"]}
                WHERE bucket >= %s AND bucket < %s
            """
            cursor.execute(bucket_sql, [start, actual_end])
            count = cursor.fetchone()[0]

        logger.info(
            f"Refreshed {level['name']}: {start} ~ {actual_end}, "
            f"{count} buckets affected"
        )
        return count

    def handle_backfill(self, start: datetime, end: datetime) -> Dict[str, int]:
        """
        处理补写数据后的聚合重算
        从最细粒度开始逐级刷新
        """
        results = {}
        for level in self.LEVELS:
            try:
                count = self.refresh_level(level, start, end)
                results[level["name"]] = count
            except Exception as e:
                logger.error(
                    f"Failed to refresh {level['name']}: {e}"
                )
                results[level["name"]] = -1

        return results

    def get_refresh_status(self) -> List[Dict[str, Any]]:
        """
        查询各级聚合的刷新状态
        用于监控面板展示
        """
        status = []
        for level in self.LEVELS:
            sql = """
                SELECT
                    materialization_hypertable_name,
                    last_completed_refresh,
                    refresh_lag
                FROM timescaledb_information.continuous_aggregates
                WHERE materialization_hypertable_name = %s
            """
            with self.db.getconn() as conn:
                cursor = conn.cursor()
                cursor.execute(sql, [level["view"]])
                row = cursor.fetchone()

            status.append({
                "level": level["name"],
                "view": level["view"],
                "last_refresh": row[1] if row else None,
                "lag": row[2] if row else None,
            })

        return status

    def check_refresh_health(self) -> Dict[str, bool]:
        """
        检查各级聚合是否正常刷新
        如果刷新延迟超过阈值则标记为异常
        """
        health = {}
        for level in self.LEVELS:
            lag_threshold = level["refresh_lag"] * 3  # 3倍延迟阈值为异常
            sql = """
                SELECT
                    EXTRACT(EPOCH FROM (
                        NOW() - last_completed_refresh
                    ))::float AS lag_seconds
                FROM timescaledb_information.continuous_aggregates
                WHERE materialization_hypertable_name = %s
            """
            with self.db.getconn() as conn:
                cursor = conn.cursor()
                cursor.execute(sql, [level["view"]])
                row = cursor.fetchone()

            if row and row[0] is not None:
                actual_lag = timedelta(seconds=row[0])
                health[level["name"]] = actual_lag < lag_threshold
            else:
                health[level["name"]] = False

        return health
```

**各级聚合的实际查询逻辑：**

```python
class TimeSeriesQueryService:
    """时序数据查询服务 —— 根据时间范围自动选择最优聚合层级"""

    def query_device_metrics(
        self,
        device_id: str,
        start_time: datetime,
        end_time: datetime,
        metrics: List[str] = None
    ) -> List[Dict[str, Any]]:
        """
        自动选择聚合层级：
        - < 1小时：查原始数据
        - 1小时 ~ 30天：查1分钟聚合
        - > 30天：查1小时聚合
        """
        duration = end_time - start_time

        if duration <= timedelta(hours=1):
            return self._query_raw(device_id, start_time, end_time, metrics)
        elif duration <= timedelta(days=30):
            return self._query_1min(device_id, start_time, end_time, metrics)
        else:
            return self._query_1hour(device_id, start_time, end_time, metrics)

    def _query_raw(self, device_id: str,
                   start: datetime, end: datetime,
                   metrics: List[str]) -> List[Dict]:
        """查询原始数据 —— 用于实时仪表盘和故障排查"""
        cols = ", ".join(metrics) if metrics else "*"
        sql = f"""
            SELECT time, device_id, {cols}
            FROM device_metrics
            WHERE device_id = %s
              AND time >= %s AND time < %s
            ORDER BY time ASC
        """
        with self.db.getconn() as conn:
            cursor = conn.cursor()
            cursor.execute(sql, [device_id, start, end])
            columns = [desc[0] for desc in cursor.description]
            return [dict(zip(columns, row)) for row in cursor.fetchall()]

    def _query_1min(self, device_id: str,
                    start: datetime, end: datetime,
                    metrics: List[str]) -> List[Dict]:
        """查询1分钟聚合 —— 用于趋势分析"""
        select_cols = ["bucket", "device_id"]
        if metrics:
            for m in metrics:
                select_cols.extend([
                    f"avg_{m}", f"max_{m}", f"min_{m}"
                ])
        else:
            select_cols.append("*")

        cols = ", ".join(select_cols)
        sql = f"""
            SELECT {cols}
            FROM device_metrics_1min
            WHERE device_id = %s
              AND bucket >= %s AND bucket < %s
            ORDER BY bucket ASC
        """
        with self.db.getconn() as conn:
            cursor = conn.cursor()
            cursor.execute(sql, [device_id, start, end])
            columns = [desc[0] for desc in cursor.description]
            return [dict(zip(columns, row)) for row in cursor.fetchall()]

    def _query_1hour(self, device_id: str,
                     start: datetime, end: datetime,
                     metrics: List[str]) -> List[Dict]:
        """查询1小时聚合 —— 用于长期统计和报表"""
        select_cols = ["bucket", "device_id"]
        if metrics:
            for m in metrics:
                # 1小时聚合的列名对应1分钟聚合的列名
                if m == "temp":
                    select_cols.extend([
                        "avg_temp", "max_temp", "min_temp"
                    ])
                elif m == "pressure":
                    select_cols.extend([
                        "avg_pressure", "max_pressure", "min_pressure"
                    ])
        else:
            select_cols.append("*")

        cols = ", ".join(select_cols)
        sql = f"""
            SELECT {cols}
            FROM device_metrics_1hour
            WHERE device_id = %s
              AND bucket >= %s AND bucket < %s
            ORDER BY bucket ASC
        """
        with self.db.getconn() as conn:
            cursor = conn.cursor()
            cursor.execute(sql, [device_id, start, end])
            columns = [desc[0] for desc in cursor.description]
            return [dict(zip(columns, row)) for row in cursor.fetchall()]
```

**为什么不只保留平均值——一个具体的教训：**

```
某电机轴承温度 1 分钟内 60 个采样：
  [45, 45, 45, 45, 92, 45, 45, ...]  ← 有一个 92° 的异常峰值

平均值：45.78°（看起来正常）
最大值：92°（严重异常——轴承可能过热）

如果只保留平均值 → 告警系统看不到异常 → 设备故障未被发现 → 停产损失 50 万元
```

### 冷热分层存储

```
┌─────────────────────────────────────────────────────┐
│ Hot（最近7天）    SSD    查询延迟 < 50ms            │
│   device_metrics 原始数据（未压缩）                  │
│   约 7TB（7天 × 1TB/天）                            │
│   成本：¥7万/月                                     │
├─────────────────────────────────────────────────────┤
│ Warm（7-30天）    HDD    查询延迟 < 200ms           │
│   device_metrics 压缩数据（TimescaleDB 内置压缩）    │
│   约 2.5TB（压缩率约 90%）                          │
│   成本：¥0.5万/月                                   │
├─────────────────────────────────────────────────────┤
│ Cold（30天+）     对象存储   查询延迟 1-5 分钟      │
│   device_metrics_1min（1年）约 20TB                  │
│   device_metrics_1hour（永久）约 1TB/年              │
│   成本：¥0.2万/月                                   │
└─────────────────────────────────────────────────────┘

总成本：约 ¥7.7万/月（如果全存 SSD 则需 ¥19万/月，节省 60%）
```

**自动分层实现：**

```sql
-- 7天后自动压缩（Hot → Warm）
SELECT add_compression_policy('device_metrics', INTERVAL '7 days');

-- 30天后自动删除原始数据分区（Hot/Warm → Cold）
-- 注意：删除前确保 1 分钟聚合已计算完成
SELECT add_retention_policy('device_metrics', INTERVAL '30 days');
```

**压缩前后的存储对比：**

| 数据 | 压缩前 | 压缩后 | 压缩率 |
|------|--------|--------|--------|
| 1天原始数据 | ~1TB | ~100GB | 10:1 |
| 7天热数据 | ~7TB | ~7TB（不压缩） | 1:1 |
| 23天温数据 | ~23TB | ~2.3TB | 10:1 |

### 数据生命周期自动管理：Hot → Warm → Cold → Delete 完整实现

上文描述了冷热分层的整体策略，下面给出完整的自动化归档代码，确保数据在各层之间自动流转，并在生命周期结束时安全删除。

**生命周期状态机：**

```
                    写入
                     │
                     ▼
               ┌──────────┐  7天   ┌──────────┐  30天   ┌──────────┐
               │   Hot    │───────▶│   Warm   │───────▶│   Cold   │
               │  (SSD)   │ 压缩   │  (HDD)   │ 归档   │ (S3/OSS) │
               │ 原始未压缩│       │ 压缩数据  │       │  聚合数据  │
               └──────────┘       └──────────┘       └──────────┘
                                                           │
                                                     保留期到期
                                                           │
                                                           ▼
                                                     ┌──────────┐
                                                     │  Delete  │
                                                     │  彻底删除  │
                                                     └──────────┘
```

**生命周期管理器（Python 完整实现）：**

```python
"""
数据生命周期管理器
负责 Hot → Warm → Cold → Delete 的自动化归档和清理
"""
import logging
import time
import threading
from datetime import datetime, timedelta
from typing import Dict, List, Optional, Any
from dataclasses import dataclass, field
from enum import Enum
import psycopg2
import psycopg2.extras
import boto3

logger = logging.getLogger(__name__)


class DataTier(Enum):
    """数据存储层级"""
    HOT = "hot"       # SSD，未压缩原始数据
    WARM = "warm"     # HDD，压缩原始数据
    COLD = "cold"     # S3/OSS，归档聚合数据
    DELETED = "deleted"  # 已删除


@dataclass
class TierConfig:
    """层级配置"""
    name: DataTier
    storage_type: str          # ssd / hdd / s3
    retention: timedelta       # 数据在该层保留时长
    compression_after: Optional[timedelta] = None  # 多久后压缩
    cost_per_gb_month: float = 0.0  # 每GB每月成本


@dataclass
class LifecyclePolicy:
    """生命周期策略"""
    table_name: str
    tiers: List[TierConfig] = field(default_factory=list)
    cold_export_format: str = "parquet"   # 导出格式
    cold_bucket: str = ""                  # S3 bucket 名


class DataLifecycleManager:
    """数据生命周期管理器"""

    # 默认层级配置
    DEFAULT_TIERS = [
        TierConfig(
            name=DataTier.HOT,
            storage_type="ssd",
            retention=timedelta(days=7),
            cost_per_gb_month=1.0,       # ¥1/GB/月
        ),
        TierConfig(
            name=DataTier.WARM,
            storage_type="hdd",
            retention=timedelta(days=23),  # 第7-30天
            compression_after=timedelta(days=0),  # 进入Warm即压缩
            cost_per_gb_month=0.1,        # ¥0.1/GB/月
        ),
        TierConfig(
            name=DataTier.COLD,
            storage_type="s3",
            retention=timedelta(days=335),  # 30天-1年
            cost_per_gb_month=0.02,        # ¥0.02/GB/月
        ),
        TierConfig(
            name=DataTier.DELETED,
            storage_type="none",
            retention=timedelta(days=0),
            cost_per_gb_month=0.0,
        ),
    ]

    def __init__(self, dsn: str, s3_endpoint: str = None,
                 s3_bucket: str = None):
        self.dsn = dsn
        self.s3_client = boto3.client(
            's3', endpoint_url=s3_endpoint
        ) if s3_endpoint else None
        self.s3_bucket = s3_bucket
        self._running = False
        self._thread = None

    def get_conn(self):
        return psycopg2.connect(self.dsn)

    def setup_policies(self):
        """初始化 TimescaleDB 内置的压缩和保留策略"""
        conn = self.get_conn()
        cur = conn.cursor()
        try:
            # ---- Hot → Warm：7天后自动压缩 ----
            cur.execute("""
                SELECT 1 FROM timescaledb_information.jobs
                WHERE proc_name = 'policy_compression'
                  AND hypertable_name = 'device_metrics'
            """)
            if not cur.fetchone():
                cur.execute("""
                    SELECT add_compression_policy(
                        'device_metrics', INTERVAL '7 days'
                    )
                """)
                logger.info("Added compression policy: 7 days")

            # ---- Warm → Cold → Delete：30天后删除原始数据 ----
            cur.execute("""
                SELECT 1 FROM timescaledb_information.jobs
                WHERE proc_name = 'policy_retention'
                  AND hypertable_name = 'device_metrics'
            """)
            if not cur.fetchone():
                cur.execute("""
                    SELECT add_retention_policy(
                        'device_metrics', INTERVAL '30 days'
                    )
                """)
                logger.info("Added retention policy: 30 days")

            # ---- 1分钟聚合：1年后删除 ----
            cur.execute("""
                SELECT 1 FROM timescaledb_information.jobs
                WHERE proc_name = 'policy_retention'
                  AND hypertable_name = 'device_metrics_1min'
            """)
            if not cur.fetchone():
                cur.execute("""
                    SELECT add_retention_policy(
                        'device_metrics_1min', INTERVAL '1 year'
                    )
                """)
                logger.info("Added retention policy for 1min agg: 1 year")

            # ---- Cold 层归档：导出30天+的聚合数据到 S3 ----
            self._setup_cold_archive_policy(cur)

            conn.commit()
            logger.info("All lifecycle policies configured")
        except Exception as e:
            conn.rollback()
            logger.error(f"Failed to setup policies: {e}")
            raise
        finally:
            conn.close()

    def _setup_cold_archive_policy(self, cur):
        """配置 Cold 层自动归档到 S3 的自定义 Job"""
        # TimescaleDB 支持 user-defined action，注册自定义归档函数
        cur.execute("""
            CREATE OR REPLACE FUNCTION archive_to_cold(
                job_id INTEGER, config JSONB
            )
            RETURNS VOID AS $$
            DECLARE
                cutoff TIMESTAMPTZ;
                chunk_name TEXT;
                export_path TEXT;
            BEGIN
                cutoff := NOW() - (config->>'age')::INTERVAL;

                FOR chunk_name IN
                    SELECT chunk_name FROM timescaledb_information.chunks
                    WHERE hypertable_name = 'device_metrics'
                      AND range_end <= cutoff
                      AND NOT is_compressed
                LOOP
                    -- 导出压缩 chunk 到 S3（使用 timescaledb-experimental 的
                    -- chunk repacking 或自定义 COPY TO S3 逻辑）
                    export_path := format(
                        's3://%s/raw/%s.parquet',
                        config->>'bucket', chunk_name
                    );
                    PERform timescaledb_experimental.chunk_to_s3(
                        chunk_name::regclass, export_path
                    );
                END LOOP;
            END;
            $$ LANGUAGE plpgsql;
        """)

    def export_cold_data(self, table_name: str, older_than: timedelta,
                         bucket: str, prefix: str = "iot-archive"):
        """
        将超过指定时间的数据导出到 S3/OSS 冷存储
        使用 COPY + Parquet 格式实现高效导出
        """
        cutoff = datetime.utcnow() - older_than
        conn = self.get_conn()
        cur = conn.cursor()

        try:
            # 查询需要归档的分区（chunk）
            cur.execute("""
                SELECT chunk_schema, chunk_name,
                       range_start, range_end
                FROM timescaledb_information.chunks
                WHERE hypertable_name = %s
                  AND range_end < %s
                  AND range_end > NOW() - INTERVAL '30 days'
                ORDER BY range_start
            """, [table_name, cutoff])

            chunks = cur.fetchall()
            logger.info(
                f"Found {len(chunks)} chunks to archive for {table_name}"
            )

            for chunk_schema, chunk_name, range_start, range_end in chunks:
                # 导出为 Parquet 格式（通过 fdw 或 COPY）
                s3_key = (
                    f"{prefix}/{table_name}/"
                    f"{range_start.strftime('%Y%m%d')}/"
                    f"{chunk_name}.parquet"
                )

                # 使用 PostgreSQL COPY 导出为 CSV（作为示例）
                # 生产环境建议使用 pgbouncer + Parquet fdw
                export_sql = f"""
                    COPY {chunk_schema}.{chunk_name}
                    TO STDOUT WITH (FORMAT csv, HEADER)
                """
                s3_path = f"s3://{bucket}/{s3_key}"

                # 记录归档元数据
                cur.execute("""
                    INSERT INTO archive_metadata
                        (table_name, chunk_name, s3_path,
                         range_start, range_end,
                         archived_at, status)
                    VALUES (%s, %s, %s, %s, %s, NOW(), 'archived')
                    ON CONFLICT (chunk_name) DO UPDATE
                    SET s3_path = EXCLUDED.s3_path,
                        archived_at = NOW(),
                        status = 'archived'
                """, [table_name, chunk_name, s3_path,
                      range_start, range_end])

                logger.info(
                    f"Archived chunk {chunk_name} to {s3_path}"
                )

            conn.commit()
        except Exception as e:
            conn.rollback()
            logger.error(f"Cold archive failed: {e}")
            raise
        finally:
            conn.close()

    def restore_from_cold(self, table_name: str,
                          start_time: datetime,
                          end_time: datetime) -> int:
        """
        从 S3 冷存储恢复数据到 TimescaleDB（用于特殊查询需求）
        返回恢复的行数
        """
        conn = self.get_conn()
        cur = conn.cursor()
        total_rows = 0

        try:
            # 查找归档元数据
            cur.execute("""
                SELECT chunk_name, s3_path
                FROM archive_metadata
                WHERE table_name = %s
                  AND range_start >= %s
                  AND range_end <= %s
                  AND status = 'archived'
                ORDER BY range_start
            """, [table_name, start_time, end_time])

            archives = cur.fetchall()

            for chunk_name, s3_path in archives:
                # 从 S3 下载数据并导入
                logger.info(
                    f"Restoring {chunk_name} from {s3_path}"
                )
                # 生产实现：下载 Parquet → 转换 → COPY INTO
                cur.execute("""
                    UPDATE archive_metadata
                    SET status = 'restored', restored_at = NOW()
                    WHERE chunk_name = %s
                """, [chunk_name])
                total_rows += 1

            conn.commit()
            logger.info(
                f"Restored {total_rows} chunks from cold storage"
            )
        except Exception as e:
            conn.rollback()
            logger.error(f"Cold restore failed: {e}")
            raise
        finally:
            conn.close()

        return total_rows

    def run_maintenance(self):
        """
        执行一轮完整的生命周期维护
        1. 检查压缩策略执行状态
        2. 检查保留策略执行状态
        3. 导出 Cold 层数据
        4. 清理过期 Cold 归档
        """
        conn = self.get_conn()
        cur = conn.cursor()

        try:
            # 1. 检查压缩 Job 状态
            cur.execute("""
                SELECT job_id, hypertable_name,
                       last_run_status, last_started,
                       next_start, total_runs, total_failures
                FROM timescaledb_information.job_stats
                WHERE proc_name = 'policy_compression'
            """)
            for row in cur.fetchall():
                (job_id, hypertable, status, last_start,
                 next_start, runs, failures) = row
                if failures > 0:
                    logger.warning(
                        f"Compression job {job_id} on {hypertable} "
                        f"has {failures} failures"
                    )

            # 2. 检查保留 Job 状态
            cur.execute("""
                SELECT job_id, hypertable_name,
                       last_run_status, total_failures
                FROM timescaledb_information.job_stats
                WHERE proc_name = 'policy_retention'
            """)
            for row in cur.fetchall():
                job_id, hypertable, status, failures = row
                if failures > 0:
                    logger.warning(
                        f"Retention job {job_id} on {hypertable} "
                        f"has {failures} failures"
                    )

            # 3. 导出即将删除的数据到 Cold 层
            # 在原始数据被 retention policy 删除前，先导出到 S3
            self.export_cold_data(
                'device_metrics',
                older_than=timedelta(days=28),  # 留2天缓冲
                bucket=self.s3_bucket or 'iot-archive',
                prefix='raw-data'
            )

            # 4. 清理超过 Cold 保留期的 S3 归档
            self._cleanup_expired_archives(cur)

            conn.commit()
        except Exception as e:
            conn.rollback()
            logger.error(f"Lifecycle maintenance failed: {e}")
            raise
        finally:
            conn.close()

    def _cleanup_expired_archives(self, cur):
        """清理 S3 上超过保留期的归档数据"""
        cur.execute("""
            SELECT chunk_name, s3_path
            FROM archive_metadata
            WHERE status = 'archived'
              AND archived_at < NOW() - INTERVAL '1 year'
        """)
        expired = cur.fetchall()
        for chunk_name, s3_path in expired:
            # 从 S3 删除
            if self.s3_client and self.s3_bucket:
                try:
                    key = s3_path.replace(
                        f"s3://{self.s3_bucket}/", ""
                    )
                    self.s3_client.delete_object(
                        Bucket=self.s3_bucket, Key=key
                    )
                except Exception as e:
                    logger.error(
                        f"Failed to delete S3 object {s3_path}: {e}"
                    )
                    continue

            cur.execute("""
                UPDATE archive_metadata
                SET status = 'deleted', deleted_at = NOW()
                WHERE chunk_name = %s
            """, [chunk_name])
            logger.info(f"Deleted expired cold archive: {chunk_name}")

    def start_background(self, interval_minutes: int = 60):
        """启动后台生命周期管理线程"""
        self._running = True

        def _run():
            while self._running:
                try:
                    self.run_maintenance()
                except Exception as e:
                    logger.error(f"Background maintenance error: {e}")
                time.sleep(interval_minutes * 60)

        self._thread = threading.Thread(
            target=_run, daemon=True, name="lifecycle-manager"
        )
        self._thread.start()
        logger.info(
            f"Lifecycle manager started (interval={interval_minutes}min)"
        )

    def stop_background(self):
        """停止后台管理线程"""
        self._running = False
        if self._thread:
            self._thread.join(timeout=30)
        logger.info("Lifecycle manager stopped")

    def get_storage_report(self) -> Dict[str, Any]:
        """获取当前存储分布报告"""
        conn = self.get_conn()
        cur = conn.cursor()
        report = {}

        try:
            # 各表的总大小和压缩状态
            for table in ['device_metrics', 'device_metrics_1min',
                          'device_metrics_1hour', 'device_metrics_1day']:
                cur.execute("""
                    SELECT
                        pg_size_pretty(
                            total_bytes
                        ) AS total_size,
                        total_bytes,
                        node_name,
                        is_compressed,
                        count(*) AS chunk_count
                    FROM timescaledb_information.chunks
                    WHERE hypertable_name = %s
                    GROUP BY total_bytes, node_name, is_compressed
                """, [table])
                rows = cur.fetchall()
                report[table] = {
                    "chunks": [
                        {
                            "size": r[0],
                            "bytes": r[1],
                            "compressed": r[3],
                            "chunk_count": r[4],
                        }
                        for r in rows
                    ]
                }

            # Cold 层归档统计
            cur.execute("""
                SELECT count(*),
                       sum(pg_column_size(row))
                FROM archive_metadata
                WHERE status = 'archived'
            """)
            row = cur.fetchone()
            report["cold_archive"] = {
                "chunk_count": row[0],
                "estimated_size": row[1],
            }
        finally:
            conn.close()

        return report
```

**归档元数据表（记录每个 chunk 的归档状态）：**

```sql
-- 归档元数据表：追踪每个 chunk 的冷存储归档状态
CREATE TABLE archive_metadata (
    id              BIGSERIAL PRIMARY KEY,
    table_name      VARCHAR(100) NOT NULL,
    chunk_name      VARCHAR(100) NOT NULL UNIQUE,
    s3_path         TEXT NOT NULL,
    range_start     TIMESTAMPTZ NOT NULL,
    range_end       TIMESTAMPTZ NOT NULL,
    archived_at     TIMESTAMPTZ,
    restored_at     TIMESTAMPTZ,
    deleted_at      TIMESTAMPTZ,
    status          VARCHAR(20) DEFAULT 'pending',
                    -- pending / archived / restored / deleted
    row_count       BIGINT,         -- chunk 行数
    compressed_size BIGINT,         -- 压缩后字节数
    checksum        VARCHAR(64),    -- 数据完整性校验
    metadata        JSONB           -- 扩展信息
);

CREATE INDEX idx_archive_table_status
    ON archive_metadata(table_name, status);
CREATE INDEX idx_archive_range
    ON archive_metadata(range_start, range_end);
CREATE INDEX idx_archive_status_time
    ON archive_metadata(status, archived_at)
    WHERE status = 'archived';
```

**生命周期监控指标：**

```python
class LifecycleMonitor:
    """生命周期监控指标收集器"""

    def collect_metrics(self) -> Dict[str, float]:
        """收集生命周期相关的监控指标"""
        conn = self.get_conn()
        cur = conn.cursor()
        metrics = {}

        try:
            # 各层数据量
            cur.execute("""
                SELECT
                    CASE
                        WHEN NOT is_compressed THEN 'hot'
                        ELSE 'warm'
                    END AS tier,
                    count(*) AS chunk_count,
                    sum(range_end - range_start) AS total_duration
                FROM timescaledb_information.chunks
                WHERE hypertable_name = 'device_metrics'
                GROUP BY tier
            """)
            for row in cur.fetchall():
                tier, chunk_count, duration = row
                metrics[f"chunks_{tier}"] = chunk_count

            # Cold 层归档量
            cur.execute("""
                SELECT count(*), sum(compressed_size)
                FROM archive_metadata
                WHERE status = 'archived'
            """)
            row = cur.fetchone()
            metrics["cold_archive_count"] = row[0]
            metrics["cold_archive_bytes"] = float(row[1] or 0)

            # Job 执行状态
            cur.execute("""
                SELECT proc_name, count(*) FILTER (
                    WHERE last_run_status = 'Success'
                ), count(*) FILTER (
                    WHERE last_run_status != 'Success'
                )
                FROM timescaledb_information.job_stats
                WHERE proc_name IN (
                    'policy_compression', 'policy_retention'
                )
                GROUP BY proc_name
            """)
            for row in cur.fetchall():
                proc, success, fail = row
                metrics[f"job_{proc}_success"] = success
                metrics[f"job_{proc}_fail"] = fail

        finally:
            conn.close()

        return metrics
```

### 设备离线补写与聚合重算

**问题：** 设备离线 2 小时后重新上线，补报 7200 条数据。这些数据落入已经计算过 1 分钟聚合的时间段。

**TimescaleDB 的处理：**

Continuous Aggregates 有两种刷新策略：
1. `timescaledb.materialized_only = true`：只查询物化结果，不自动重算
2. `timescaledb.materialized_only = false`：查询时实时计算未物化的部分

**补写场景下的配置：**

```sql
-- 设置为非仅物化模式（允许实时计算）
ALTER MATERIALIZED VIEW device_metrics_1min SET (
    timescaledb.materialized_only = false
);
```

**查询时的行为：**

```sql
-- 查询最近 1 小时的 1 分钟聚合
-- TimescaleDB 会：
--   1. 读取已物化的数据（到最近一次刷新时间）
--   2. 实时计算未物化的部分（从原始数据表）
--   3. 合并两部分结果
SELECT * FROM device_metrics_1min
WHERE device_id = 'PLC-A001'
  AND bucket > NOW() - INTERVAL '1 hour'
ORDER BY bucket;
```

**手动刷新受影响的聚合时间段：**

```sql
-- 补写数据后，手动刷新受影响的 1 分钟聚合
-- 例如：补写了 2 小时的数据，刷新 2:00-4:00 的聚合
CALL refresh_continuous_aggregate('device_metrics_1min', 
    '2024-05-07 02:00:00', '2024-05-07 04:00:00');
```

**自动化流程：**

```python
class BackfillHandler:
    def on_device_backfill(self, device_id, start_time, end_time):
        # 1. 写入补报数据到 device_metrics
        self.write_backfill_data(device_id, start_time, end_time)

        # 2. 刷新受影响时间段的 1 分钟聚合
        self.refresh_continuous_aggregate(
            'device_metrics_1min', start_time, end_time
        )

        # 3. 刷新受影响时间段的 1 小时聚合
        self.refresh_continuous_aggregate(
            'device_metrics_1hour', start_time, end_time
        )

        # 4. 重新评估受影响时间段的告警
        self.reevaluate_alerts(device_id, start_time, end_time)
```

### 告警系统设计

**告警规则示例：**

```python
class AlertRule:
    """告警规则引擎"""

    rules = [
        {
            "name": "temperature_high",
            "device_type": "PLC",
            "condition": "temperature > 90 AND duration > 30s",
            "severity": "critical",
            "action": "notify_and_shutdown"
        },
        {
            "name": "vibration_anomaly",
            "device_type": "Motor",
            "condition": "vibration > 0.1 OR (vibration > 0.05 AND duration > 60s)",
            "severity": "warning",
            "action": "notify"
        },
        {
            "name": "current_spike",
            "device_type": "Motor",
            "condition": "current > max_current * 1.5",
            "severity": "critical",
            "action": "emergency_shutdown"
        }
    ]
```

### 实时告警管道：指标摄入 → 阈值评估 → 告警 → 通知（含去重）

完整的实时告警管道从数据摄入到最终通知，涉及四个阶段。以下是端到端完整实现。

**告警管道架构：**

```
┌──────────────┐   ┌───────────────┐   ┌──────────────┐   ┌──────────────┐
│  指标摄入     │──▶│  阈值评估      │──▶│  告警去重     │──▶│  通知分发     │
│  Kafka消费    │   │  Flink CEP    │   │  状态机管理   │   │  多渠道推送   │
│  反序列化     │   │  规则匹配     │   │  告警收敛     │   │  升级策略     │
└──────────────┘   └───────────────┘   └──────────────┘   └──────────────┘
       │                   │                   │                   │
       ▼                   ▼                   ▼                   ▼
   写入TimescaleDB    产生告警事件       合并重复告警         短信/邮件/Webhook
```

**阶段 1：指标摄入与预处理（Flink Source）：**

```java
/**
 * 指标摄入模块：从 Kafka 消费设备指标数据
 */
public class MetricIngestionFunction
        extends RichSourceFunction<DeviceMetric> {

    private final String bootstrapServers;
    private final String topicPattern;
    private final String groupId;
    private transient FlinkKafkaConsumer<DeviceMetric> kafkaConsumer;

    public MetricIngestionFunction(String bootstrapServers,
                                    String topicPattern,
                                    String groupId) {
        this.bootstrapServers = bootstrapServers;
        this.topicPattern = topicPattern;
        this.groupId = groupId;
    }

    @Override
    public void open(Configuration parameters) {
        Properties props = new Properties();
        props.setProperty("bootstrap.servers", bootstrapServers);
        props.setProperty("group.id", groupId);
        props.setProperty("auto.offset.reset", "latest");
        props.setProperty("max.poll.records", "5000");

        kafkaConsumer = new FlinkKafkaConsumer<>(
            topicPattern,
            new DeviceMetricDeserializer(),
            props
        );
    }

    @Override
    public void run(SourceContext<DeviceMetric> ctx) {
        // 由 Flink 框架管理消费循环
    }

    @Override
    public void cancel() {
        if (kafkaConsumer != null) {
            kafkaConsumer.close();
        }
    }
}

/**
 * 设备指标反序列化器
 */
public class DeviceMetricDeserializer
        implements KafkaDeserializationSchema<DeviceMetric> {

    private final ObjectMapper mapper = new ObjectMapper();

    @Override
    public DeviceMetric deserialize(
            ConsumerRecord<byte[], byte[]> record) {
        try {
            return mapper.readValue(record.value(),
                                    DeviceMetric.class);
        } catch (IOException e) {
            throw new RuntimeException(
                "Failed to deserialize: " + e.getMessage(), e
            );
        }
    }

    @Override
    public boolean isEndOfStream(DeviceMetric nextElement) {
        return false;
    }

    @Override
    public TypeInformation<DeviceMetric> getProducedType() {
        return TypeInformation.of(DeviceMetric.class);
    }
}
```

**阶段 2：阈值评估引擎（Flink CEP）：**

```java
/**
 * 阈值评估引擎：基于 Flink CEP 实现多条件组合告警
 */
public class ThresholdEvaluationEngine {

    private final StreamExecutionEnvironment env;
    private final AlertRuleRepository ruleRepo;

    public ThresholdEvaluationEngine(
            StreamExecutionEnvironment env,
            AlertRuleRepository ruleRepo) {
        this.env = env;
        this.ruleRepo = ruleRepo;
    }

    /**
     * 构建告警模式并注册到流处理
     */
    public DataStream<AlertEvent> buildAlertPipeline(
            DataStream<DeviceMetric> metricStream) {

        DataStream<AlertEvent> allAlerts = null;
        List<AlertRuleDefinition> rules = ruleRepo.loadActiveRules();

        for (AlertRuleDefinition rule : rules) {
            DataStream<AlertEvent> ruleAlerts =
                buildRulePattern(metricStream, rule);

            if (allAlerts == null) {
                allAlerts = ruleAlerts;
            } else {
                allAlerts = allAlerts.union(ruleAlerts);
            }
        }

        return allAlerts;
    }

    /**
     * 为单条规则构建 CEP 模式
     */
    private DataStream<AlertEvent> buildRulePattern(
            DataStream<DeviceMetric> stream,
            AlertRuleDefinition rule) {

        Pattern<DeviceMetric, ?> pattern;

        switch (rule.getPatternType()) {
            case THRESHOLD:
                // 简单阈值：值超过阈值持续N秒
                pattern = Pattern.<DeviceMetric>begin("trigger")
                    .where(m -> evaluateThreshold(m, rule))
                    .timesOrMore(rule.getConsecutiveCount())
                    .within(Time.seconds(rule.getWithinSeconds()));
                break;

            case RATE_OF_CHANGE:
                // 变化率：值在短时间内急剧变化
                pattern = Pattern.<DeviceMetric>begin("start")
                    .where(m -> matchDevice(m, rule))
                    .followedBy("spike")
                    .where(new RateOfChangeCondition(
                        rule.getMetricField(),
                        rule.getRateThreshold(),
                        rule.getRateWindowSeconds()
                    ))
                    .within(Time.seconds(rule.getWithinSeconds()));
                break;

            case COMPOSITE:
                // 组合模式：多个指标同时异常
                pattern = Pattern.<DeviceMetric>begin("temp_trigger")
                    .where(m -> evaluateCompositeTrigger(m, rule))
                    .followedBy("vibration_trigger")
                    .where(m -> evaluateCompositeSecondary(m, rule))
                    .within(Time.seconds(rule.getWithinSeconds()));
                break;

            default:
                throw new IllegalArgumentException(
                    "Unknown pattern type: " + rule.getPatternType()
                );
        }

        return CEP.pattern(
            stream.keyBy(DeviceMetric::getDeviceId), pattern
        ).select(matches -> {
            DeviceMetric triggerMetric = matches.get("trigger") != null
                ? matches.get("trigger").get(0)
                : matches.get("start").get(0);

            return AlertEvent.builder()
                .alertId(UUID.randomUUID().toString())
                .ruleId(rule.getRuleId())
                .ruleName(rule.getRuleName())
                .deviceId(triggerMetric.getDeviceId())
                .severity(rule.getSeverity())
                .firedAt(Instant.now())
                .metricField(rule.getMetricField())
                .actualValue(extractValue(triggerMetric, rule))
                .thresholdValue(rule.getThreshold())
                .message(buildAlertMessage(triggerMetric, rule))
                .build();
        });
    }

    private boolean evaluateThreshold(DeviceMetric metric,
                                       AlertRuleDefinition rule) {
        if (!matchDevice(metric, rule)) return false;
        double value = extractValue(metric, rule);
        switch (rule.getOperator()) {
            case GT:  return value > rule.getThreshold();
            case GTE: return value >= rule.getThreshold();
            case LT:  return value < rule.getThreshold();
            case LTE: return value <= rule.getThreshold();
            default:  return false;
        }
    }

    private double extractValue(DeviceMetric metric,
                                 AlertRuleDefinition rule) {
        switch (rule.getMetricField()) {
            case "temperature": return metric.getTemperature();
            case "pressure":    return metric.getPressure();
            case "rpm":         return metric.getRpm();
            case "current":     return metric.getCurrent();
            case "voltage":     return metric.getVoltage();
            case "vibration":   return metric.getVibration();
            default:
                throw new IllegalArgumentException(
                    "Unknown metric: " + rule.getMetricField()
                );
        }
    }
}

/**
 * 变化率条件：检测指标在短时间内的急剧变化
 */
public class RateOfChangeCondition
        extends SimpleCondition<DeviceMetric> {

    private final String metricField;
    private final double rateThreshold;
    private final long windowSeconds;
    private final Map<String, Double> lastValues = new ConcurrentHashMap<>();
    private final Map<String, Long> lastTimestamps = new ConcurrentHashMap<>();

    public RateOfChangeCondition(String metricField,
                                  double rateThreshold,
                                  long windowSeconds) {
        this.metricField = metricField;
        this.rateThreshold = rateThreshold;
        this.windowSeconds = windowSeconds;
    }

    @Override
    public boolean filter(DeviceMetric metric) {
        String key = metric.getDeviceId();
        double currentValue = getFieldValue(metric);
        long currentTs = metric.getTimestamp();

        Double lastValue = lastValues.get(key);
        Long lastTs = lastTimestamps.get(key);

        lastValues.put(key, currentValue);
        lastTimestamps.put(key, currentTs);

        if (lastValue == null || lastTs == null) return false;
        if (currentTs - lastTs > windowSeconds) return false;

        double rate = Math.abs(currentValue - lastValue) /
                      Math.abs(lastValue);
        return rate > rateThreshold;
    }
}
```

**阶段 3：告警去重与状态机管理：**

```python
"""
告警去重与状态机管理器
负责告警生命周期：firing → acknowledged → resolved
以及告警去重（同设备同规则在告警窗口内不重复触发）
"""
import logging
import threading
from datetime import datetime, timedelta
from typing import Dict, List, Optional
from dataclasses import dataclass, field
from enum import Enum
import psycopg2

logger = logging.getLogger(__name__)


class AlertStatus(Enum):
    FIRING = "firing"
    ACKNOWLEDGED = "acknowledged"
    RESOLVED = "resolved"
    SUPPRESSED = "suppressed"


@dataclass
class AlertEvent:
    alert_id: str
    rule_id: str
    rule_name: str
    device_id: str
    severity: str
    fired_at: datetime
    metric_field: str
    actual_value: float
    threshold_value: float
    message: str


@dataclass
class ActiveAlert:
    alert_id: str
    rule_id: str
    device_id: str
    severity: str
    status: AlertStatus
    first_fired_at: datetime
    last_fired_at: datetime
    fire_count: int = 1
    dedup_key: str = ""
    suppress_until: Optional[datetime] = None
    notification_count: int = 0
    escalation_level: int = 0


class AlertDeduplicator:
    """告警去重器"""

    DEDUP_WINDOW = timedelta(minutes=5)
    NOTIFICATION_INTERVAL = timedelta(minutes=15)
    ESCALATION_INTERVAL = timedelta(minutes=30)

    def __init__(self, dsn: str):
        self.dsn = dsn
        self._active_alerts: Dict[str, ActiveAlert] = {}
        self._lock = threading.RLock()

    def _make_dedup_key(self, device_id: str, rule_id: str) -> str:
        return f"{device_id}:{rule_id}"

    def process_alert(self, event: AlertEvent) -> Optional[ActiveAlert]:
        """
        处理告警事件，返回需要通知的告警（去重后）
        同设备同规则在去重窗口内不重复触发通知
        """
        dedup_key = self._make_dedup_key(event.device_id, event.rule_id)

        with self._lock:
            existing = self._active_alerts.get(dedup_key)

            if existing and existing.status == AlertStatus.FIRING:
                # 告警已存在且仍在触发中 → 更新计数
                existing.fire_count += 1
                existing.last_fired_at = event.fired_at

                # 检查是否需要重复通知
                if self._should_renotify(existing):
                    existing.notification_count += 1
                    return existing
                else:
                    self._log_suppressed_alert(existing, event)
                    return None

            elif existing and existing.status == AlertStatus.ACKNOWLEDGED:
                existing.fire_count += 1
                existing.last_fired_at = event.fired_at
                return None

            else:
                # 新告警
                alert = ActiveAlert(
                    alert_id=event.alert_id,
                    rule_id=event.rule_id,
                    device_id=event.device_id,
                    severity=event.severity,
                    status=AlertStatus.FIRING,
                    first_fired_at=event.fired_at,
                    last_fired_at=event.fired_at,
                    fire_count=1,
                    dedup_key=dedup_key,
                )
                self._active_alerts[dedup_key] = alert
                alert.notification_count = 1
                return alert

    def resolve_alert(self, device_id: str, rule_id: str,
                      resolved_at: datetime = None):
        """标记告警为已恢复"""
        dedup_key = self._make_dedup_key(device_id, rule_id)
        with self._lock:
            existing = self._active_alerts.get(dedup_key)
            if existing and existing.status in (
                AlertStatus.FIRING, AlertStatus.ACKNOWLEDGED
            ):
                existing.status = AlertStatus.RESOLVED
                self._persist_alert_resolution(existing, resolved_at)

    def _should_renotify(self, alert: ActiveAlert) -> bool:
        """判断是否需要重复通知"""
        if alert.severity == "critical":
            interval = timedelta(minutes=5)
        elif alert.severity == "warning":
            interval = timedelta(minutes=15)
        else:
            interval = timedelta(minutes=60)
        elapsed = datetime.utcnow() - alert.last_fired_at
        return elapsed >= interval

    def check_escalation(self) -> List[ActiveAlert]:
        """
        检查需要升级的告警：
        - critical 15分钟未确认 → 升级到值班经理
        - warning 30分钟未确认 → 升级到技术负责人
        """
        escalated = []
        with self._lock:
            now = datetime.utcnow()
            for alert in self._active_alerts.values():
                if alert.status != AlertStatus.FIRING:
                    continue
                duration = now - alert.first_fired_at

                if (alert.severity == "critical"
                        and duration > timedelta(minutes=15)
                        and alert.escalation_level == 0):
                    alert.escalation_level = 1
                    escalated.append(alert)
                elif (alert.severity == "warning"
                      and duration > timedelta(minutes=30)
                      and alert.escalation_level == 0):
                    alert.escalation_level = 1
                    escalated.append(alert)
                elif (alert.escalation_level == 1
                      and duration > timedelta(hours=1)):
                    alert.escalation_level = 2
                    escalated.append(alert)
        return escalated

    def _log_suppressed_alert(self, alert: ActiveAlert,
                               event: AlertEvent):
        conn = psycopg2.connect(self.dsn)
        try:
            cur = conn.cursor()
            cur.execute("""
                INSERT INTO alert_suppression_log
                    (original_alert_id, device_id, rule_id,
                     suppressed_at, actual_value, threshold_value)
                VALUES (%s, %s, %s, %s, %s, %s)
            """, [alert.alert_id, alert.device_id, alert.rule_id,
                  datetime.utcnow(), event.actual_value,
                  event.threshold_value])
            conn.commit()
        finally:
            conn.close()

    def _persist_alert_resolution(self, alert: ActiveAlert,
                                   resolved_at: datetime = None):
        conn = psycopg2.connect(self.dsn)
        try:
            cur = conn.cursor()
            cur.execute("""
                UPDATE alerts
                SET status = 'resolved', resolved_at = %s
                WHERE id = (
                    SELECT id FROM alerts
                    WHERE device_id = %s
                      AND alert_type = %s
                      AND status = 'firing'
                    ORDER BY fired_at DESC LIMIT 1
                )
            """, [resolved_at or datetime.utcnow(),
                  alert.device_id, alert.rule_id])
            conn.commit()
        finally:
            conn.close()

    def cleanup_resolved(self, max_age: timedelta = timedelta(hours=24)):
        """清理已解决的告警缓存"""
        with self._lock:
            now = datetime.utcnow()
            expired = [
                k for k, v in self._active_alerts.items()
                if (v.status == AlertStatus.RESOLVED
                    and (now - v.last_fired_at) > max_age)
            ]
            for k in expired:
                del self._active_alerts[k]
            logger.info(f"Cleaned {len(expired)} resolved caches")
```

**阶段 4：通知分发（多渠道推送与升级策略）：**

```python
"""
通知分发器
支持多渠道推送（短信、邮件、Webhook、钉钉/企业微信）
以及升级策略（未确认自动升级到上级）
"""
import logging
import smtplib
from datetime import datetime, timedelta
from typing import Dict, List
from dataclasses import dataclass
from enum import Enum
from email.mime.text import MIMEText
import requests

logger = logging.getLogger(__name__)


class NotificationChannel(Enum):
    SMS = "sms"
    EMAIL = "email"
    WEBHOOK = "webhook"
    DINGTALK = "dingtalk"
    WECHAT = "wechat"


@dataclass
class NotificationRule:
    severity: str
    channel: NotificationChannel
    recipients: List[str]
    escalation_after: int          # 多少秒后升级
    escalation_recipients: List[str]


@dataclass
class NotificationResult:
    success: bool
    channel: NotificationChannel
    recipient: str
    message_id: str = ""
    error: str = ""


class NotificationDispatcher:
    """通知分发器"""

    DEFAULT_RULES = [
        NotificationRule(
            severity="critical",
            channel=NotificationChannel.SMS,
            recipients=["13800138001", "13800138002"],
            escalation_after=15,
            escalation_recipients=["13900139001"],
        ),
        NotificationRule(
            severity="critical",
            channel=NotificationChannel.DINGTALK,
            recipients=["group:ops-critical"],
            escalation_after=15,
            escalation_recipients=["group:management"],
        ),
        NotificationRule(
            severity="warning",
            channel=NotificationChannel.EMAIL,
            recipients=["ops-team@company.com"],
            escalation_after=30,
            escalation_recipients=["cto@company.com"],
        ),
        NotificationRule(
            severity="warning",
            channel=NotificationChannel.WEBHOOK,
            recipients=["https://hooks.internal/alert"],
            escalation_after=30,
            escalation_recipients=[],
        ),
    ]

    def __init__(self):
        self._senders = {
            NotificationChannel.SMS: self._send_sms,
            NotificationChannel.EMAIL: self._send_email,
            NotificationChannel.WEBHOOK: self._send_webhook,
            NotificationChannel.DINGTALK: self._send_dingtalk,
            NotificationChannel.WECHAT: self._send_wechat,
        }

    def dispatch(self, alert, escalation: bool = False
                ) -> List[NotificationResult]:
        """分发告警通知，escalation=True 时使用升级接收人"""
        results = []
        for rule in self.DEFAULT_RULES:
            if rule.severity != alert.severity:
                continue
            recipients = (rule.escalation_recipients if escalation
                          else rule.recipients)
            sender = self._senders.get(rule.channel)
            if not sender:
                continue
            for recipient in recipients:
                try:
                    result = sender(recipient, alert, escalation)
                    results.append(result)
                except Exception as e:
                    logger.error(
                        f"Failed {rule.channel.value} to {recipient}: {e}"
                    )
                    results.append(NotificationResult(
                        success=False, channel=rule.channel,
                        recipient=recipient, error=str(e),
                    ))
        return results

    def _send_sms(self, phone: str, alert, escalation: bool
                 ) -> NotificationResult:
        prefix = "[升级]" if escalation else "[告警]"
        message = (
            f"{prefix}{alert.severity.upper()} - "
            f"设备{alert.device_id} {alert.rule_name} "
            f"当前值{alert.actual_value} 阈值{alert.threshold_value}"
        )
        resp = requests.post(
            "https://sms-gateway.internal/api/send",
            json={"phone": phone, "message": message}, timeout=10,
        )
        return NotificationResult(
            success=resp.status_code == 200,
            channel=NotificationChannel.SMS, recipient=phone,
            message_id=resp.json().get("msg_id", ""),
        )

    def _send_email(self, email: str, alert, escalation: bool
                   ) -> NotificationResult:
        subject = (
            f"{'[升级]' if escalation else '[告警]'} "
            f"{alert.severity.upper()}: "
            f"设备 {alert.device_id} - {alert.rule_name}"
        )
        body = (
            f"告警详情：\n"
            f"  设备ID: {alert.device_id}\n"
            f"  告警规则: {alert.rule_name}\n"
            f"  严重程度: {alert.severity}\n"
            f"  当前值: {alert.actual_value}\n"
            f"  阈值: {alert.threshold_value}\n"
            f"  触发时间: {alert.first_fired_at}\n"
            f"  重复次数: {alert.fire_count}\n"
        )
        try:
            msg = MIMEText(body)
            msg["Subject"] = subject
            msg["From"] = "alert-system@company.com"
            msg["To"] = email
            with smtplib.SMTP("smtp.internal", 25) as smtp:
                smtp.send_message(msg)
            return NotificationResult(
                success=True, channel=NotificationChannel.EMAIL,
                recipient=email,
            )
        except Exception as e:
            return NotificationResult(
                success=False, channel=NotificationChannel.EMAIL,
                recipient=email, error=str(e),
            )

    def _send_webhook(self, url: str, alert, escalation: bool
                     ) -> NotificationResult:
        payload = {
            "alert_id": alert.alert_id,
            "device_id": alert.device_id,
            "rule_name": alert.rule_name,
            "severity": alert.severity,
            "actual_value": alert.actual_value,
            "threshold_value": alert.threshold_value,
            "fire_count": alert.fire_count,
            "escalated": escalation,
            "fired_at": alert.first_fired_at.isoformat(),
        }
        resp = requests.post(url, json=payload, timeout=10)
        return NotificationResult(
            success=resp.status_code == 200,
            channel=NotificationChannel.WEBHOOK, recipient=url,
        )

    def _send_dingtalk(self, group: str, alert, escalation: bool
                      ) -> NotificationResult:
        prefix = "[升级]" if escalation else "[告警]"
        payload = {
            "msgtype": "markdown",
            "markdown": {
                "title": f"{prefix}{alert.severity.upper()} 告警",
                "text": (
                    f"### {prefix}{alert.severity.upper()} 告警\n"
                    f"- 设备: {alert.device_id}\n"
                    f"- 规则: {alert.rule_name}\n"
                    f"- 当前值: {alert.actual_value}\n"
                    f"- 阈值: {alert.threshold_value}\n"
                    f"- 重复次数: {alert.fire_count}\n"
                ),
            },
        }
        resp = requests.post(
            f"https://oapi.dingtalk.com/robot/send"
            f"?access_token={self._get_token(group)}",
            json=payload, timeout=10,
        )
        return NotificationResult(
            success=resp.status_code == 200,
            channel=NotificationChannel.DINGTALK, recipient=group,
        )

    def _send_wechat(self, user: str, alert, escalation: bool
                    ) -> NotificationResult:
        prefix = "[升级]" if escalation else "[告警]"
        payload = {
            "touser": user,
            "msgtype": "text",
            "text": {"content": (
                f"{prefix}{alert.severity.upper()}\n"
                f"设备: {alert.device_id}\n"
                f"规则: {alert.rule_name}\n"
                f"值: {alert.actual_value} / 阈值: {alert.threshold_value}"
            )},
        }
        resp = requests.post(
            "https://qyapi.weixin.qq.com/cgi-bin/message/send",
            json=payload, timeout=10,
        )
        return NotificationResult(
            success=resp.status_code == 200,
            channel=NotificationChannel.WECHAT, recipient=user,
        )

    def _get_token(self, group: str) -> str:
        token_map = {
            "group:ops-critical": "token_ops_001",
            "group:management": "token_mgmt_001",
        }
        return token_map.get(group, "")
```

**告警抑制日志与通知日志表：**

```sql
-- 告警抑制日志：记录被去重抑制的重复告警
CREATE TABLE alert_suppression_log (
    id              BIGSERIAL PRIMARY KEY,
    original_alert_id VARCHAR(100) NOT NULL,
    device_id       VARCHAR(50) NOT NULL,
    rule_id         VARCHAR(50) NOT NULL,
    suppressed_at   TIMESTAMPTZ NOT NULL,
    actual_value    FLOAT8,
    threshold_value FLOAT8,
    reason          VARCHAR(50) DEFAULT 'dedup'
                    -- dedup / maintenance_window
);

CREATE INDEX idx_suppression_alert
    ON alert_suppression_log(original_alert_id);
CREATE INDEX idx_suppression_device_time
    ON alert_suppression_log(device_id, suppressed_at DESC);

-- 通知发送日志
CREATE TABLE notification_log (
    id              BIGSERIAL PRIMARY KEY,
    alert_id        VARCHAR(100) NOT NULL,
    device_id       VARCHAR(50) NOT NULL,
    channel         VARCHAR(20) NOT NULL,
                    -- sms / email / webhook / dingtalk / wechat
    recipient       VARCHAR(200) NOT NULL,
    success         BOOLEAN NOT NULL,
    error_message   TEXT,
    sent_at         TIMESTAMPTZ DEFAULT NOW(),
    message_id      VARCHAR(100)
);

CREATE INDEX idx_notification_alert
    ON notification_log(alert_id);
CREATE INDEX idx_notification_sent_at
    ON notification_log(sent_at DESC);
```

**端到端告警管道主程序：**

```python
"""
告警管道主程序：串联 摄入 → 评估 → 去重 → 通知
"""
import logging
import threading
import time
from datetime import datetime, timedelta
from typing import List

logger = logging.getLogger(__name__)


class AlertPipeline:
    """端到端告警管道"""

    def __init__(self, dsn: str, kafka_config: dict):
        self.deduplicator = AlertDeduplicator(dsn)
        self.dispatcher = NotificationDispatcher()
        self.db_sink = TimescaleDBSink(dsn)
        self._running = False

    def start(self):
        self._running = True
        self._start_flink_job()

        threading.Thread(target=self._consume_alerts,
                         daemon=True, name="alert-consumer").start()
        threading.Thread(target=self._check_escalations,
                         daemon=True, name="escalation-checker").start()
        threading.Thread(target=self._check_resolutions,
                         daemon=True, name="resolve-checker").start()
        logger.info("Alert pipeline started")

    def stop(self):
        self._running = False
        logger.info("Alert pipeline stopped")

    def _start_flink_job(self):
        logger.info("Starting Flink CEP job...")

    def _consume_alerts(self):
        """从告警队列消费告警事件并处理"""
        while self._running:
            try:
                alerts = self._poll_alert_events()
                for event in alerts:
                    self._persist_alert(event)
                    active_alert = self.deduplicator.process_alert(event)
                    if active_alert:
                        results = self.dispatcher.dispatch(active_alert)
                        self._log_dispatch_results(event, results)
                time.sleep(0.1)
            except Exception as e:
                logger.error(f"Alert consumption error: {e}")
                time.sleep(1)

    def _check_escalations(self):
        """定期检查告警升级"""
        while self._running:
            try:
                escalated = self.deduplicator.check_escalation()
                for alert in escalated:
                    logger.warning(
                        f"Escalating alert: {alert.alert_id} "
                        f"device={alert.device_id} "
                        f"level={alert.escalation_level}"
                    )
                    self.dispatcher.dispatch(alert, escalation=True)
                time.sleep(30)
            except Exception as e:
                logger.error(f"Escalation check error: {e}")
                time.sleep(5)

    def _check_resolutions(self):
        """定期检查告警恢复条件"""
        while self._running:
            try:
                firing_alerts = self._get_firing_alerts()
                for alert in firing_alerts:
                    current = self._get_current_metric(
                        alert[1], alert[2]  # device_id, alert_type
                    )
                    threshold = float(alert[5])  # threshold_value
                    if current < threshold * 0.9:
                        self.deduplicator.resolve_alert(
                            alert[1], alert[2]
                        )
                        logger.info(
                            f"Alert resolved: device={alert[1]}"
                        )
                time.sleep(10)
            except Exception as e:
                logger.error(f"Resolution check error: {e}")
                time.sleep(5)

    def _poll_alert_events(self) -> List:
        return []

    def _persist_alert(self, event):
        import psycopg2
        conn = psycopg2.connect(self.dsn)
        try:
            cur = conn.cursor()
            cur.execute("""
                INSERT INTO alerts
                    (device_id, alert_type, severity,
                     threshold_value, actual_value,
                     message, status, fired_at, metadata)
                VALUES (%s, %s, %s, %s, %s, %s, 'firing', %s, %s)
            """, [event.device_id, event.rule_name, event.severity,
                  event.threshold_value, event.actual_value,
                  event.message, event.fired_at,
                  {"rule_id": event.rule_id}])
            conn.commit()
        finally:
            conn.close()

    def _log_dispatch_results(self, event, results):
        for r in results:
            status = "OK" if r.success else f"FAIL: {r.error}"
            logger.info(
                f"Notification [{r.channel.value}] → "
                f"{r.recipient}: {status}"
            )

    def _get_firing_alerts(self):
        import psycopg2
        conn = psycopg2.connect(self.dsn)
        try:
            cur = conn.cursor()
            cur.execute("""
                SELECT id, device_id, alert_type, severity,
                       threshold_value, actual_value, fired_at
                FROM alerts WHERE status = 'firing'
                ORDER BY fired_at ASC
            """)
            return cur.fetchall()
        finally:
            conn.close()

    def _get_current_metric(self, device_id, metric_field):
        import psycopg2
        conn = psycopg2.connect(self.dsn)
        try:
            cur = conn.cursor()
            cur.execute(f"""
                SELECT {metric_field}
                FROM device_metrics
                WHERE device_id = %s
                ORDER BY time DESC LIMIT 1
            """, [device_id])
            row = cur.fetchone()
            return float(row[0]) if row else 0.0
        finally:
            conn.close()
```

**Flink CEP 告警检测（补充完整实现）：**

```java
// 使用 Flink CEP（Complex Event Processing）检测复杂告警模式
Pattern<DeviceMetric, ?> tempHighPattern = Pattern
    .<DeviceMetric>begin("start")
    .where(m -> m.temperature > 90)
    .timesOrMore(6)  // 连续 6 次（6秒）温度超过 90°
    .within(Time.seconds(10));

CEP.pattern(metricStream.keyBy(m -> m.deviceId), tempHighPattern)
    .select(matches -> {
        DeviceMetric first = matches.get("start").get(0);
        return new Alert(
            first.deviceId, 
            "temperature_high", 
            "CRITICAL",
            "连续6秒温度超过90°C"
        );
    })
    .addSink(alertSink);
```

### TimescaleDB 连续聚合完整实现：三级降采样（1min → 1hour → 1day）

**1 分钟聚合（从原始数据降采样）：**

```sql
-- 1 分钟连续聚合视图
CREATE MATERIALIZED VIEW device_metrics_1min
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 minute', time) AS bucket,
    device_id,
    avg(temperature)   AS avg_temp,
    max(temperature)   AS max_temp,
    min(temperature)   AS min_temp,
    count(temperature) AS count_temp,
    avg(pressure)      AS avg_pressure,
    max(pressure)      AS max_pressure,
    min(pressure)      AS min_pressure,
    count(pressure)    AS count_pressure,
    avg(rpm)           AS avg_rpm,
    max(rpm)           AS max_rpm,
    min(rpm)           AS min_rpm,
    avg(current)       AS avg_current,
    max(current)       AS max_current,
    min(current)       AS min_current,
    count(current)     AS count_current,
    avg(voltage)       AS avg_voltage,
    min(voltage)       AS min_voltage,
    max(voltage)       AS max_voltage,
    avg(vibration)     AS avg_vibration,
    max(vibration)     AS max_vibration,
    min(vibration)     AS min_vibration
FROM device_metrics
GROUP BY bucket, device_id
WITH NO DATA;

-- 添加自动刷新策略：每 5 分钟刷新一次，延迟 5 分钟确保数据到齐
SELECT add_continuous_aggregate_policy('device_metrics_1min',
    start_offset => INTERVAL '5 minutes',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '5 minutes');

-- 1 年后删除 1 分钟聚合数据
SELECT add_retention_policy('device_metrics_1min', INTERVAL '1 year');

-- 1 分钟聚合也启用压缩（30天后压缩）
ALTER MATERIALIZED VIEW device_metrics_1min SET (
    timescaledb.compress_segmentby = 'device_id',
    timescaledb.compress_orderby = 'bucket DESC'
);
SELECT add_compression_policy('device_metrics_1min', INTERVAL '30 days');
```

**1 小时聚合（从 1 分钟聚合进一步降采样）：**

```sql
-- 1 小时连续聚合视图
CREATE MATERIALIZED VIEW device_metrics_1hour
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', bucket) AS bucket,
    device_id,
    avg(avg_temp)    AS avg_temp,
    max(max_temp)    AS max_temp,
    min(min_temp)    AS min_temp,
    sum(count_temp)  AS total_samples_temp,
    avg(avg_pressure) AS avg_pressure,
    max(max_pressure) AS max_pressure,
    min(min_pressure) AS min_pressure,
    avg(avg_rpm)     AS avg_rpm,
    max(max_rpm)     AS max_rpm,
    min(min_rpm)     AS min_rpm,
    avg(avg_current) AS avg_current,
    max(max_current) AS max_current,
    min(min_current) AS min_current,
    avg(avg_voltage) AS avg_voltage,
    min(min_voltage) AS min_voltage,
    max(max_voltage) AS max_voltage,
    avg(avg_vibration) AS avg_vibration,
    max(max_vibration) AS max_vibration,
    min(min_vibration) AS min_vibration
FROM device_metrics_1min
GROUP BY bucket, device_id
WITH NO DATA;

-- 添加自动刷新策略：每 1 小时刷新一次
SELECT add_continuous_aggregate_policy('device_metrics_1hour',
    start_offset => INTERVAL '1 hour',
    end_offset => INTERVAL '4 hours',
    schedule_interval => INTERVAL '1 hour');

-- 1 小时聚合永久保留（不设置 retention policy）
-- 但启用压缩以节省空间（90天后压缩）
ALTER MATERIALIZED VIEW device_metrics_1hour SET (
    timescaledb.compress_segmentby = 'device_id',
    timescaledb.compress_orderby = 'bucket DESC'
);
SELECT add_compression_policy('device_metrics_1hour', INTERVAL '90 days');
```

**1 天聚合（从 1 小时聚合进一步降采样——新增层级）：**

```sql
-- 1 天连续聚合视图（用于长期趋势分析和年度报表）
CREATE MATERIALIZED VIEW device_metrics_1day
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', bucket) AS bucket,
    device_id,
    avg(avg_temp)    AS avg_temp,
    max(max_temp)    AS max_temp,
    min(min_temp)    AS min_temp,
    sum(total_samples_temp) AS total_samples_temp,
    avg(avg_pressure) AS avg_pressure,
    max(max_pressure) AS max_pressure,
    min(min_pressure) AS min_pressure,
    avg(avg_rpm)     AS avg_rpm,
    max(max_rpm)     AS max_rpm,
    min(min_rpm)     AS min_rpm,
    avg(avg_current) AS avg_current,
    max(max_current) AS max_current,
    min(min_current) AS min_current,
    avg(avg_voltage) AS avg_voltage,
    min(min_voltage) AS min_voltage,
    max(max_voltage) AS max_voltage,
    avg(avg_vibration) AS avg_vibration,
    max(max_vibration) AS max_vibration,
    min(min_vibration) AS min_vibration
FROM device_metrics_1hour
GROUP BY bucket, device_id
WITH NO DATA;

-- 添加自动刷新策略：每天刷新一次
SELECT add_continuous_aggregate_policy('device_metrics_1day',
    start_offset => INTERVAL '1 day',
    end_offset => INTERVAL '2 days',
    schedule_interval => INTERVAL '1 day');

-- 1 天聚合永久保留，启用压缩（180天后压缩）
ALTER MATERIALIZED VIEW device_metrics_1day SET (
    timescaledb.compress_segmentby = 'device_id',
    timescaledb.compress_orderby = 'bucket DESC'
);
SELECT add_compression_policy('device_metrics_1day', INTERVAL '180 days');
```

**连续聚合刷新策略详解：**

```
┌──────────────────────────────────────────────────────────────────────┐
│ 聚合层级    刷新间隔    start_offset    end_offset    数据保留       │
├──────────────────────────────────────────────────────────────────────┤
│ 1min        5分钟      5 minutes       1 hour        1 year         │
│ 1hour       1小时      1 hour          4 hours       永久           │
│ 1day        1天        1 day           2 days        永久           │
└──────────────────────────────────────────────────────────────────────┘

start_offset：聚合计算的最远起始点（距当前时间的偏移量）
end_offset：聚合计算的截止点（距当前时间的偏移量，留出缓冲确保数据到齐）
schedule_interval：自动刷新的调度间隔
```

**查询自动重写示例（TimescaleDB 透明优化）：**

```sql
-- 用户查询 1 分钟聚合，TimescaleDB 自动合并物化数据 + 实时计算
-- 模式1：物化数据覆盖范围内 → 直接读物化表（最快）
SELECT avg_temp, max_temp FROM device_metrics_1min
WHERE device_id = 'PLC-A001'
  AND bucket > NOW() - INTERVAL '1 day';

-- 模式2：部分超出物化范围 → 物化数据 + 实时从原始表计算未物化部分
SELECT avg_temp, max_temp FROM device_metrics_1min
WHERE device_id = 'PLC-A001'
  AND bucket > NOW() - INTERVAL '10 minutes';
-- TimescaleDB 内部执行计划：
--   1) 从物化表读取已刷新的 bucket
--   2) 从原始表实时计算未刷新的 bucket
--   3) UNION ALL 合并返回

-- 模式3：设置 materialized_only = true 时，跳过实时计算
ALTER MATERIALIZED VIEW device_metrics_1min SET (
    timescaledb.materialized_only = true
);
-- 此时查询只返回物化数据，性能更稳定但可能有延迟
```

**手动刷新与补写联动：**

```sql
-- 场景：设备补写 2:00-4:00 的数据后，手动刷新三级聚合
-- 第一级：1 分钟聚合
CALL refresh_continuous_aggregate('device_metrics_1min',
    '2024-05-07 02:00:00', '2024-05-07 04:00:00');

-- 第二级：1 小时聚合（依赖 1 分钟聚合的结果）
CALL refresh_continuous_aggregate('device_metrics_1hour',
    '2024-05-07 02:00:00', '2024-05-07 04:00:00');

-- 第三级：1 天聚合（依赖 1 小时聚合的结果）
CALL refresh_continuous_aggregate('device_metrics_1day',
    '2024-05-07 00:00:00', '2024-05-07 00:00:00' + INTERVAL '1 day');
```

### 查询性能

| 查询类型 | 数据源 | 预估延迟 |
|---------|--------|---------|
| 实时仪表盘（最近1小时） | device_metrics（Hot/SSD） | < 50ms |
| 1天趋势图 | device_metrics_1min | < 100ms |
| 1个月趋势图 | device_metrics_1min（Warm） | < 300ms |
| 1年趋势图 | device_metrics_1hour | < 200ms |
| 单设备历史明细（30天内） | device_metrics（Hot/Warm） | < 200ms |

**查询示例：**

```sql
-- 实时仪表盘：最近1小时所有设备的最新温度
SELECT DISTINCT ON (device_id)
    device_id, time, temperature
FROM device_metrics
WHERE time > NOW() - INTERVAL '1 hour'
ORDER BY device_id, time DESC;

-- 1天温度趋势图
SELECT bucket, device_id, avg_temp, max_temp, min_temp
FROM device_metrics_1min
WHERE device_id = 'PLC-A001'
  AND bucket > NOW() - INTERVAL '1 day'
ORDER BY bucket;
```

## 常见陷阱（深度分析）

### 陷阱 1：只保留平均值

**具体的故障漏检场景：**

某电机轴承温度在 1 分钟内出现 3 次 120°C 的尖峰（每次持续 1 秒），其余时间正常 45°C。

- 平均值：45.78°C → 低于告警阈值 90°C → 无告警
- 最大值：120°C → 触发严重告警 → 及时停机检查

**后果：** 轴承过热导致电机烧毁，维修费用 20 万元，停产 2 天。

### 陷阱 2：原始数据永久保留

**存储增长：**
- 1 年原始数据：190TB
- 3 年：570TB
- 按年增长，永不减少

**正确策略：** 原始数据 30 天后删除，只保留聚合数据。聚合数据已包含 avg/max/min，满足绝大多数分析需求。

### 陷阱 3：补写数据不触发聚合重算

**具体问题：** 设备 2:00-4:00 离线，4:00 补报数据。但 1 分钟聚合在 2:00 就已经计算完成（结果为 NULL 或缺失）。如果不刷新，查询 2:00-4:00 的聚合数据仍然为空。

**解决方案：** 补写后调用 `refresh_continuous_aggregate`，TimescaleDB 会重新计算受影响时间段的聚合。

### 陷阱 4：单节点扛全量写入

**200 万点/秒 = 10 万行/秒。** TimescaleDB 单节点写入上限约 10 万行/秒（批量写入）。刚好在临界值，没有余量。

**解决方案：** 3 节点分布式 Hypertable，每节点承担约 3.3 万行/秒，有 3 倍余量。

## 延伸思考

- **设备数从 10 万增长到 100 万**：写入量增长 10 倍到 2000 万点/秒。需要更多 TimescaleDB 节点（约 30 个），同时 Kafka 分区数需要扩展到 200+。
- **设备健康度评分**：基于历史数据计算设备的退化趋势（如振动值逐月增加 → 轴承磨损 → 预测性维护）。需要在 1 小时聚合数据上运行机器学习模型。
- **边缘计算**：在工厂本地部署边缘节点，做本地聚合和告警判断，只上传聚合数据和异常数据到云端。减少 95% 的上行带宽。
## 时序数据降采样完整实现

```python
class TimeseriesDownsampler:
    """时序数据降采样：保留趋势特征，减少存储"""

    def downsample(self, device_id, raw_data, target_interval_seconds=60):
        """将原始数据降采样到目标间隔"""
        downsampled = []
        window_start = raw_data[0]["timestamp"]
        window_values = []

        for point in raw_data:
            if (point["timestamp"] - window_start).total_seconds() < target_interval_seconds:
                window_values.append(point["value"])
            else:
                # 计算窗口统计值
                downsampled.append({
                    "device_id": device_id,
                    "timestamp": window_start,
                    "interval_seconds": target_interval_seconds,
                    "min": min(window_values),
                    "max": max(window_values),
                    "avg": statistics.mean(window_values),
                    "count": len(window_values),
                    "sum": sum(window_values),
                    "p50": sorted(window_values)[len(window_values) // 2],
                    "p95": sorted(window_values)[int(len(window_values) * 0.95)],
                    "p99": sorted(window_values)[int(len(window_values) * 0.99)],
                })
                # 开始新窗口
                window_start = point["timestamp"]
                window_values = [point["value"]]

        return downsampled

    def multi_level_downsample(self, device_id, raw_data):
        """多级降采样：1分钟 → 5分钟 → 1小时 → 1天"""
        levels = [
            {"interval": 60, "retention": "30d", "table": "ts_1min"},
            {"interval": 300, "retention": "90d", "table": "ts_5min"},
            {"interval": 3600, "retention": "365d", "table": "ts_1hour"},
            {"interval": 86400, "retention": "forever", "table": "ts_1day"},
        ]
        current = raw_data
        for level in levels:
            current = self.downsample(device_id, current, level["interval"])
            self.db.batch_insert(level["table"], current)
        return len(levels)
```

## 异常检测引擎

```python
class TimeseriesAnomalyDetector:
    """时序异常检测"""

    def detect(self, device_id, current_value, metric_name):
        """检测当前值是否异常"""
        # 获取最近 7 天基线
        baseline = self.db.query(
            f"SELECT avg(value) as avg_val, stddev(value) as std_val "
            f"FROM ts_5min WHERE device_id = %s AND metric_name = %s "
            f"AND timestamp > NOW() - INTERVAL 7 DAY",
            device_id, metric_name)[0]

        if not baseline["std_val"]:
            return {"is_anomaly": False}

        # Z-score 检测
        z_score = abs(current_value - baseline["avg_val"]) / baseline["std_val"]
        is_anomaly = z_score > 3  # 3-sigma 规则

        if is_anomaly:
            severity = "CRITICAL" if z_score > 5 else "WARNING"
            self.alert(f"设备 {device_id} {metric_name} 异常: "
                       f"值={current_value}, 基线={baseline['avg_val']:.2f}, "
                       f"Z-score={z_score:.1f}, 严重程度={severity}")

        return {"is_anomaly": is_anomaly, "z_score": z_score,
                "baseline_avg": baseline["avg_val"], "severity": severity if is_anomaly else "NORMAL"}
```

## 数据保留策略

```sql
-- TimescaleDB 连续聚合 + 保留策略
-- 原始数据保留 30 天
SELECT add_retention_policy('device_readings', INTERVAL '30 days');

-- 1 分钟聚合保留 90 天
CREATE MATERIALIZED VIEW device_1min
WITH (timescaledb.continuous) AS
SELECT device_id, metric_name,
    time_bucket('1 minute', timestamp) as bucket,
    avg(value) as avg_val, min(value) as min_val,
    max(value) as max_val, count(*) as sample_count
FROM device_readings GROUP BY device_id, metric_name, bucket;

SELECT add_retention_policy('device_1min', INTERVAL '90 days');

-- 1 小时聚合永久保留
CREATE MATERIALIZED VIEW device_1hour
WITH (timescaledb.continuous) AS
SELECT device_id, metric_name,
    time_bucket('1 hour', timestamp) as bucket,
    avg(value) as avg_val, min(value) as min_val,
    max(value) as max_val, count(*) as sample_count
FROM device_readings GROUP BY device_id, metric_name, bucket;
```

## 性能分析详细数据

**写入吞吐量：**

| 数据库 | 设备数 | 每设备/秒 | 总写入 TPS | 压缩比 |
|--------|--------|----------|-----------|--------|
| TimescaleDB | 10 万 | 1 | 10 万 | 10:1 |
| InfluxDB | 10 万 | 1 | 10 万 | 8:1 |
| ClickHouse | 100 万 | 1 | 100 万 | 15:1 |

**月度成本：**

| 组件 | 规格 | 月成本 |
|------|------|-------|
| TimescaleDB | 8c64G + 2TB SSD × 3 | ¥5 万 |
| Kafka (数据摄入) | 6节点 | ¥3 万 |
| Redis (最新值缓存) | 3节点 | ¥1.5 万 |
| Flink (流处理) | 4节点 | ¥2 万 |
| **合计** | | **¥11.5 万** |

## 设备管理完整实现

```python
class IoTDeviceManager:
    """IoT 设备管理：注册、配置、健康监控"""

    def register_device(self, device_info):
        """注册新设备"""
        device_id = str(uuid4())
        # 1. 验证设备证书
        if not self.certificate_validator.validate(device_info["certificate"]):
            raise DeviceCertificateError("设备证书无效")

        # 2. 分配设备到租户
        self.db.insert("devices", {
            "device_id": device_id,
            "device_type": device_info["type"],
            "model": device_info["model"],
            "firmware_version": device_info["firmware_version"],
            "tenant_id": device_info["tenant_id"],
            "status": "registered",
            "last_seen": now(),
            "registered_at": now()
        })

        # 3. 创建设备配置
        config = self._get_default_config(device_info["type"])
        self.db.insert("device_configs", {
            "device_id": device_id,
            "report_interval_seconds": config["report_interval"],
            "sampling_rate_hz": config["sampling_rate"],
            "alert_thresholds": json.dumps(config["thresholds"])
        })

        # 4. 推送配置到设备
        self.mqtt_client.publish(f"config/{device_id}", json.dumps(config), qos=1)

        return device_id

    def check_device_health(self, device_id):
        """设备健康检查"""
        device = self.db.get_device(device_id)

        # 1. 连接状态：最后心跳时间
        heartbeat_age = (now() - device["last_seen"]).total_seconds()
        if heartbeat_age > 300:  # 5分钟无心跳
            return {"status": "offline", "last_seen": device["last_seen"]}

        # 2. 数据新鲜度：最新数据时间
        latest_data = self.db.query_one(
            "SELECT MAX(timestamp) as latest FROM device_readings "
            "WHERE device_id = %s", device_id)
        if latest_data and (now() - latest_data["latest"]).total_seconds() > 120:
            return {"status": "stale_data", "latest_data": latest_data["latest"]}

        # 3. 错误率检查
        error_count = self.redis.get(f"device_errors:{device_id}")
        if error_count and int(error_count) > 10:
            return {"status": "high_errors", "error_count": int(error_count)}

        return {"status": "healthy"}
```

## 异常场景补充

### 场景：设备固件升级失败

```python
class DeviceFirmwareManager:
    """设备固件升级管理"""
    def upgrade(self, device_id, target_version):
        firmware = self.db.get_firmware(target_version)
        # 推送固件
        self.mqtt_client.publish(f"ota/{device_id}", json.dumps({
            "version": target_version,
            "url": firmware["download_url"],
            "checksum": firmware["sha256"],
            "size": firmware["size_bytes"]
        }), qos=2)
        # 等待设备确认
        result = self.wait_for_ota_confirmation(device_id, timeout=120)
        if not result["success"]:
            # 升级失败 → 设备自动回滚
            self.alert(f"设备 {device_id} 升级到 {target_version} 失败")
            return {"status": "failed", "current_version": result["rolled_back_version"]}
        return {"status": "success", "new_version": target_version}
```

### 场景：设备数据上报延迟

```
触发：设备数据上报间隔从 10 秒变为 60 秒 → 降采样间隔内数据不足
检测：
  1. 上报间隔 > 预期间隔 3 倍 → 告警
  2. 降采样窗口内数据点 < 5 → 数据稀疏
处理：
  1. 降低降采样精度：从统计值改为最后值
  2. 延长降采样窗口（1 分钟 → 5 分钟）
  3. 检查设备原因：电量低、信号弱
预防：上报间隔监控 + 自适应降采样窗口
```

### 场景：Redis 缓存与数据库不一致

```
触发：Redis 中最新值与数据库最新值不同 → 查询返回过时数据
检测：
  1. 定期对账：Redis vs TimescaleDB 最新值
  2. 差异 > 1% → 告警
处理：
  1. 以数据库为准 → 修正 Redis
  2. 增加双写一致性：写入时同时写 Redis + DB
预防：写入时双写 + 定期对账修正
```

## IoT 规则引擎完整实现

```python
class IoTRuleEngine:
    """IoT 规则引擎：条件评估 + 动作执行 + 冷却期"""

    def evaluate_rules(self, device_id, metric_name, value):
        """评估设备触发的规则"""
        rules = self.db.query(
            "SELECT * FROM iot_rules WHERE device_type = "
            "(SELECT device_type FROM devices WHERE id = %s) "
            "AND metric_name = %s AND enabled = 1",
            device_id, metric_name)

        for rule in rules:
            condition = json.loads(rule["condition"])
            if self._evaluate_condition(value, condition):
                # 检查冷却期
                cooldown_key = f"rule_cooldown:{rule['id']}:{device_id}"
                if self.redis.exists(cooldown_key):
                    continue  # 冷却期内，跳过

                # 执行动作
                self._execute_action(device_id, rule)
                # 设置冷却期
                self.redis.setex(cooldown_key, rule["cooldown_seconds"], "1")

    def _evaluate_condition(self, value, condition):
        """评估条件"""
        op = condition["operator"]
        threshold = condition["threshold"]

        if op == "gt":
            return value > threshold
        elif op == "lt":
            return value < threshold
        elif op == "gte":
            return value >= threshold
        elif op == "lte":
            return value <= threshold
        elif op == "between":
            return condition["min"] <= value <= condition["max"]
        elif op == "rate_of_change":
            # 变化率检测：5 分钟内变化超过阈值
            return abs(value - condition["previous_value"]) / max(condition["previous_value"], 0.01) > threshold
        return False

    def _execute_action(self, device_id, rule):
        """执行规则动作"""
        action = json.loads(rule["action"])
        if action["type"] == "alert":
            self.alert(f"设备 {device_id} 规则触发: {rule['name']}")
        elif action["type"] == "command":
            self.device_shadow.update_desired(device_id, action["params"])
        elif action["type"] == "webhook":
            self.http_client.post(action["url"], {
                "device_id": device_id, "rule_id": rule["id"],
                "triggered_at": now().isoformat()
            })
```

## 设备影子服务

```python
class DeviceShadowService:
    """设备影子：期望/报告/差异状态管理"""

    def update_desired(self, device_id, desired_state):
        """更新期望状态（App/云端侧）"""
        shadow = self._get_shadow(device_id)
        # 合并期望状态
        shadow["desired"].update(desired_state)
        shadow["desired_version"] += 1
        shadow["delta"] = self._calc_delta(shadow["desired"], shadow["reported"])

        # 存储
        self.redis.set(f"shadow:{device_id}", json.dumps(shadow))

        # 推送差异到设备
        if shadow["delta"]:
            self.mqtt_client.publish(f"shadow/delta/{device_id}",
                json.dumps(shadow["delta"]), qos=1)

    def update_reported(self, device_id, reported_state):
        """更新报告状态（设备侧）"""
        shadow = self._get_shadow(device_id)
        shadow["reported"].update(reported_state)
        shadow["reported_version"] += 1
        shadow["delta"] = self._calc_delta(shadow["desired"], shadow["reported"])

        self.redis.set(f"shadow:{device_id}", json.dumps(shadow))

    def _calc_delta(self, desired, reported):
        """计算差异（desired 有但 reported 不同）"""
        delta = {}
        for key, value in desired.items():
            if key not in reported or reported[key] != value:
                delta[key] = value
        return delta

    def _get_shadow(self, device_id):
        """获取设备影子"""
        cached = self.redis.get(f"shadow:{device_id}")
        if cached:
            return json.loads(cached)
        return {
            "desired": {}, "desired_version": 0,
            "reported": {}, "reported_version": 0,
            "delta": {}
        }
```

## 异常场景补充

### 场景：规则引擎无限循环检测

```
触发：规则 A → 控制设备 → 规则 B 监听变化 → 触发规则 A → 循环
检测：
  1. 同一规则 10 分钟内触发 > 5 次 → 振荡检测
  2. 规则执行链深度 > 5 → 循环检测
处理：
  1. 自动暂停触发振荡的规则
  2. 通知管理员规则冲突
  3. 增加规则执行冷却期
预防：规则 DAG 检测循环 + 执行深度限制 + 振荡检测
```

### 场景：设备影子版本冲突

```
触发：App 和设备同时更新影子 → 版本冲突
检测：
  1. desired_version 与设备端携带版本不匹配 → 冲突
处理：
  1. 使用乐观并发控制：客户端携带版本号
  2. 版本不匹配 → 返回 409 Conflict + 最新影子
  3. 客户端重新获取最新影子后重试
预防：版本号乐观锁 + 冲突时返回最新状态
```

## IoT 规则引擎深度实现

### 条件评估（阈值/范围/变化率）

```python
class IoTRuleEngineV2:
    """IoT规则引擎V2：条件评估 + 动作执行 + 规则链 + 冷却期"""

    def __init__(self, db, redis, mqtt_client, device_shadow):
        self.db = db
        self.redis = redis
        self.mqtt_client = mqtt_client
        self.device_shadow = device_shadow

    def evaluate_rules(self, device_id, metric_name, value, timestamp=None):
        """评估设备触发的规则"""
        # 1. 获取设备类型
        device = self.db.get("devices", device_id)
        if not device:
            return []

        # 2. 加载适用规则（设备类型 + 指标名）
        rules = self.db.query(
            "SELECT * FROM iot_rules WHERE device_type = %s "
            "AND metric_name = %s AND enabled = 1",
            device["device_type"], metric_name)

        triggered = []
        for rule in rules:
            condition = json.loads(rule["condition"])
            if self._evaluate_condition(device_id, value, condition):
                # 冷却期检查
                cooldown_key = f"rule_cooldown:{rule['id']}:{device_id}"
                if self.redis.exists(cooldown_key):
                    continue

                # 振荡检测
                if self._is_oscillating(rule["id"], device_id):
                    self._pause_rule(rule["id"], "oscillation_detected")
                    continue

                # 执行动作
                result = self._execute_action(device_id, rule, value)
                # 设置冷却期
                self.redis.setex(cooldown_key, rule["cooldown_seconds"], "1")
                # 记录触发
                self.redis.incr(f"rule_trigger_count:{rule['id']}:{device_id}")
                self.redis.expire(f"rule_trigger_count:{rule['id']}:{device_id}", 600)

                triggered.append({"rule_id": rule["id"], "action_result": result})

        return triggered

    def _evaluate_condition(self, device_id, value, condition):
        """评估条件（支持阈值/范围/变化率/持续时长）"""
        op = condition["operator"]

        if op == "gt":
            return value > condition["threshold"]
        elif op == "lt":
            return value < condition["threshold"]
        elif op == "gte":
            return value >= condition["threshold"]
        elif op == "lte":
            return value <= condition["threshold"]
        elif op == "between":
            return condition["min"] <= value <= condition["max"]
        elif op == "not_between":
            return not (condition["min"] <= value <= condition["max"])
        elif op == "eq":
            return value == condition["threshold"]
        elif op == "neq":
            return value != condition["threshold"]
        elif op == "rate_of_change":
            # 变化率检测：与上次值比较
            prev_key = f"metric_prev:{device_id}:{condition.get('metric', '')}"
            prev_value = self.redis.get(prev_key)
            if prev_value is None:
                self.redis.set(prev_key, str(value))
                return False
            prev = float(prev_value)
            rate = abs(value - prev) / max(abs(prev), 0.01)
            self.redis.set(prev_key, str(value))
            return rate > condition["threshold"]
        elif op == "sustained":
            # 持续时长检测：连续N秒超阈值
            sustain_key = f"sustain:{device_id}:{condition.get('metric', '')}"
            if value > condition["threshold"]:
                count = self.redis.incr(sustain_key)
                self.redis.expire(sustain_key, condition["duration_seconds"])
                return count >= condition["duration_seconds"]
            else:
                self.redis.delete(sustain_key)
                return False

        return False

    def _is_oscillating(self, rule_id, device_id):
        """振荡检测：同一规则10分钟内触发>5次"""
        count = self.redis.get(f"rule_trigger_count:{rule_id}:{device_id}")
        return count is not None and int(count) > 5

    def _pause_rule(self, rule_id, reason):
        """暂停振荡规则"""
        self.db.update("iot_rules",
            {"enabled": 0, "pause_reason": reason},
            {"id": rule_id})
        self.alert(f"规则 {rule_id} 因{reason}被自动暂停")
```

### 动作执行（告警/命令/Webhook）

```python
class RuleActionExecutor:
    """规则动作执行器：告警、设备命令、Webhook"""

    def __init__(self, db, redis, mqtt_client, device_shadow, http_client):
        self.db = db
        self.redis = redis
        self.mqtt_client = mqtt_client
        self.device_shadow = device_shadow
        self.http_client = http_client

    def execute_action(self, device_id, rule, current_value):
        """执行规则动作"""
        action = json.loads(rule["action"])
        action_type = action["type"]

        # 记录执行日志
        log_id = self.db.insert("rule_execution_log", {
            "rule_id": rule["id"],
            "device_id": device_id,
            "action_type": action_type,
            "trigger_value": current_value,
            "status": "executing",
            "started_at": now()
        })

        try:
            if action_type == "alert":
                result = self._send_alert(device_id, rule, current_value, action)
            elif action_type == "command":
                result = self._send_device_command(device_id, action)
            elif action_type == "webhook":
                result = self._call_webhook(device_id, rule, current_value, action)
            elif action_type == "chain":
                result = self._execute_chain(device_id, action, current_value)
            else:
                result = {"status": "unknown_action"}

            self.db.update("rule_execution_log",
                {"status": "success", "result": json.dumps(result)},
                {"id": log_id})
            return result

        except Exception as e:
            self.db.update("rule_execution_log",
                {"status": "failed", "error": str(e)},
                {"id": log_id})
            return {"status": "failed", "error": str(e)}

    def _send_alert(self, device_id, rule, value, action):
        """发送告警通知"""
        severity = action.get("severity", "warning")
        channels = action.get("channels", ["sms", "email"])

        alert_data = {
            "device_id": device_id,
            "rule_name": rule["name"],
            "rule_id": rule["id"],
            "trigger_value": value,
            "severity": severity,
            "timestamp": now().isoformat()
        }

        # 写入告警表
        self.db.insert("iot_alerts", {
            "device_id": device_id,
            "rule_id": rule["id"],
            "severity": severity,
            "message": f"设备{device_id}触发规则{rule['name']}，当前值{value}",
            "status": "active",
            "created_at": now()
        })

        # 推送通知
        for channel in channels:
            if channel == "sms":
                self._send_sms_alert(alert_data)
            elif channel == "email":
                self._send_email_alert(alert_data)
            elif channel == "mqtt":
                self.mqtt_client.publish(f"alerts/{device_id}",
                    json.dumps(alert_data), qos=1)

        return {"status": "alerted", "severity": severity}

    def _send_device_command(self, device_id, action):
        """发送设备控制命令"""
        command = action["command"]
        params = action.get("params", {})

        # 通过设备影子下发
        self.device_shadow.update_desired(device_id, params)

        # 同时通过MQTT直接发送命令（实时性更高）
        self.mqtt_client.publish(f"cmd/{device_id}", json.dumps({
            "command": command,
            "params": params,
            "timestamp": now().isoformat()
        }), qos=1)

        return {"status": "command_sent", "command": command}

    def _call_webhook(self, device_id, rule, value, action):
        """调用外部Webhook"""
        url = action["url"]
        method = action.get("method", "POST")
        headers = action.get("headers", {"Content-Type": "application/json"})

        payload = {
            "device_id": device_id,
            "rule_id": rule["id"],
            "rule_name": rule["name"],
            "trigger_value": value,
            "triggered_at": now().isoformat(),
            "custom_data": action.get("custom_data", {})
        }

        # 超时保护
        response = self.http_client.request(method, url,
            json=payload, headers=headers, timeout=5)

        return {"status": "webhook_called", "status_code": response.status_code}

    def _execute_chain(self, device_id, action, current_value):
        """执行规则链（串联多个动作）"""
        chain_actions = action["actions"]
        results = []
        for chain_action in chain_actions:
            # 每个链动作可以有自己的条件
            if "condition" in chain_action:
                if not self._evaluate_condition(device_id, current_value,
                                                  chain_action["condition"]):
                    results.append({"action": chain_action["type"], "status": "skipped"})
                    continue

            if chain_action["type"] == "alert":
                r = self._send_alert(device_id, {"name": "chain", "id": "chain"},
                                      current_value, chain_action)
            elif chain_action["type"] == "command":
                r = self._send_device_command(device_id, chain_action)
            elif chain_action["type"] == "webhook":
                r = self._call_webhook(device_id, {"name": "chain", "id": "chain"},
                                        current_value, chain_action)
            elif chain_action["type"] == "delay":
                # 延迟执行：下一个动作延迟N秒
                time.sleep(chain_action["seconds"])
                r = {"status": "delayed", "seconds": chain_action["seconds"]}
            else:
                r = {"status": "unknown_action"}

            results.append({"action": chain_action["type"], "result": r})

            # 链式执行失败时决定是否继续
            if r.get("status") == "failed" and chain_action.get("abort_on_fail", False):
                break

        return {"status": "chain_executed", "results": results}
```

### 规则链与冷却期管理

```python
class RuleChainManager:
    """规则链与冷却期管理"""

    def __init__(self, db, redis):
        self.db = db
        self.redis = redis

    def create_rule_chain(self, chain_name, rules_config):
        """创建规则链（规则串联执行）"""
        chain_id = str(uuid4())

        # 验证规则链无循环
        if self._has_cycle(rules_config):
            raise ValueError("规则链存在循环依赖")

        self.db.insert("iot_rule_chains", {
            "chain_id": chain_id,
            "name": chain_name,
            "rules": json.dumps(rules_config),
            "enabled": 1,
            "created_at": now()
        })

        return {"chain_id": chain_id, "name": chain_name}

    def _has_cycle(self, rules_config):
        """检测规则链中是否存在循环"""
        # 构建依赖图
        graph = defaultdict(list)
        for rule in rules_config:
            for dep in rule.get("depends_on", []):
                graph[dep].append(rule["id"])

        # DFS检测环
        visited = set()
        rec_stack = set()

        def dfs(node):
            visited.add(node)
            rec_stack.add(node)
            for neighbor in graph.get(node, []):
                if neighbor not in visited:
                    if dfs(neighbor):
                        return True
                elif neighbor in rec_stack:
                    return True
            rec_stack.remove(node)
            return False

        for rule in rules_config:
            if rule["id"] not in visited:
                if dfs(rule["id"]):
                    return True
        return False

    def set_cooldown(self, rule_id, device_id, cooldown_seconds):
        """设置冷却期"""
        key = f"rule_cooldown:{rule_id}:{device_id}"
        self.redis.setex(key, cooldown_seconds, str(now().isoformat()))

    def get_cooldown_remaining(self, rule_id, device_id):
        """获取冷却期剩余时间"""
        key = f"rule_cooldown:{rule_id}:{device_id}"
        ttl = self.redis.ttl(key)
        return max(ttl, 0)

    def reset_cooldown(self, rule_id, device_id):
        """重置冷却期（紧急场景）"""
        key = f"rule_cooldown:{rule_id}:{device_id}"
        self.redis.delete(key)
```

## 设备影子服务深度实现

### 期望/报告/差异状态管理

```python
class DeviceShadowServiceV2:
    """设备影子V2：期望/报告/差异/版本冲突/离线队列"""

    def __init__(self, db, redis, mqtt_client):
        self.db = db
        self.redis = redis
        self.mqtt_client = mqtt_client

    def get_shadow(self, device_id):
        """获取设备影子完整状态"""
        cached = self.redis.get(f"shadow:{device_id}")
        if cached:
            return json.loads(cached)
        # 从DB加载
        shadow = self.db.get("device_shadows", device_id)
        if shadow:
            result = {
                "device_id": device_id,
                "desired": json.loads(shadow["desired"]),
                "desired_version": shadow["desired_version"],
                "reported": json.loads(shadow["reported"]),
                "reported_version": shadow["reported_version"],
                "delta": json.loads(shadow.get("delta", "{}")),
                "metadata": json.loads(shadow.get("metadata", "{}")),
                "last_updated": shadow["updated_at"]
            }
            self.redis.set(f"shadow:{device_id}", json.dumps(result))
            return result
        return self._create_empty_shadow(device_id)

    def update_desired(self, device_id, desired_state, client_version=None):
        """更新期望状态（App/云端侧）"""
        shadow = self.get_shadow(device_id)

        # 版本冲突检测（乐观并发控制）
        if client_version is not None and shadow["desired_version"] != client_version:
            return {
                "status": "version_conflict",
                "current_version": shadow["desired_version"],
                "current_shadow": shadow
            }

        # 合并期望状态
        shadow["desired"].update(desired_state)
        shadow["desired_version"] += 1
        shadow["delta"] = self._calc_delta(shadow["desired"], shadow["reported"])
        shadow["metadata"] = self._update_metadata(shadow.get("metadata", {}),
                                                      desired_state)
        shadow["last_updated"] = now().isoformat()

        # 持久化
        self._save_shadow(device_id, shadow)

        # 推送差异到设备
        if shadow["delta"]:
            self.mqtt_client.publish(f"shadow/delta/{device_id}",
                json.dumps({
                    "state": shadow["delta"],
                    "version": shadow["desired_version"]
                }), qos=1)

        return {
            "status": "updated",
            "desired_version": shadow["desired_version"],
            "delta": shadow["delta"]
        }

    def update_reported(self, device_id, reported_state, client_version=None):
        """更新报告状态（设备侧）"""
        shadow = self.get_shadow(device_id)

        # 版本冲突检测
        if client_version is not None and shadow["reported_version"] != client_version:
            return {
                "status": "version_conflict",
                "current_version": shadow["reported_version"],
                "current_shadow": shadow
            }

        shadow["reported"].update(reported_state)
        shadow["reported_version"] += 1
        shadow["delta"] = self._calc_delta(shadow["desired"], shadow["reported"])
        shadow["last_updated"] = now().isoformat()

        self._save_shadow(device_id, shadow)

        # 如果delta为空，说明设备已同步到期望状态
        if not shadow["delta"]:
            self.mqtt_client.publish(f"shadow/ack/{device_id}",
                json.dumps({"status": "synced"}), qos=0)

        return {
            "status": "updated",
            "reported_version": shadow["reported_version"],
            "delta": shadow["delta"]
        }

    def _calc_delta(self, desired, reported):
        """计算差异（desired有但reported不同）"""
        delta = {}
        for key, value in desired.items():
            if key not in reported or reported[key] != value:
                delta[key] = value
        return delta

    def _update_metadata(self, metadata, updates):
        """更新元数据（每个属性的更新时间）"""
        ts = now().isoformat()
        for key in updates:
            metadata[key] = {"timestamp": ts}
        return metadata

    def _save_shadow(self, device_id, shadow):
        """持久化设备影子"""
        # Redis缓存
        self.redis.set(f"shadow:{device_id}", json.dumps(shadow))

        # MySQL持久化
        self.db.upsert("device_shadows", {
            "device_id": device_id,
            "desired": json.dumps(shadow["desired"]),
            "desired_version": shadow["desired_version"],
            "reported": json.dumps(shadow["reported"]),
            "reported_version": shadow["reported_version"],
            "delta": json.dumps(shadow["delta"]),
            "metadata": json.dumps(shadow.get("metadata", {})),
            "updated_at": now()
        }, conflict_key="device_id")

    def _create_empty_shadow(self, device_id):
        """创建空影子"""
        shadow = {
            "device_id": device_id,
            "desired": {}, "desired_version": 0,
            "reported": {}, "reported_version": 0,
            "delta": {}, "metadata": {},
            "last_updated": now().isoformat()
        }
        self._save_shadow(device_id, shadow)
        return shadow
```

### 离线命令队列与重连同步

```python
class OfflineCommandQueue:
    """离线命令队列：设备离线期间缓存命令，重连后推送"""

    def __init__(self, db, redis, mqtt_client, device_shadow):
        self.db = db
        self.redis = redis
        self.mqtt_client = mqtt_client
        self.device_shadow = device_shadow

    def queue_command(self, device_id, command, params, priority="normal"):
        """缓存离线命令"""
        queue_key = f"offline_cmd_queue:{device_id}"

        cmd_entry = {
            "command": command,
            "params": params,
            "priority": priority,
            "queued_at": now().isoformat()
        }

        # 按优先级插入队列
        if priority == "high":
            self.redis.lpush(queue_key, json.dumps(cmd_entry))
        else:
            self.redis.rpush(queue_key, json.dumps(cmd_entry))

        # 记录到DB（持久化保障）
        self.db.insert("offline_commands", {
            "device_id": device_id,
            "command": command,
            "params": json.dumps(params),
            "priority": priority,
            "status": "queued",
            "created_at": now()
        })

        return {"status": "queued", "device_id": device_id}

    def on_device_reconnect(self, device_id):
        """设备重连后同步离线命令"""
        # 1. 先同步设备影子差异
        shadow = self.device_shadow.get_shadow(device_id)
        if shadow["delta"]:
            self.mqtt_client.publish(f"shadow/delta/{device_id}",
                json.dumps({
                    "state": shadow["delta"],
                    "version": shadow["desired_version"]
                }), qos=1)

        # 2. 推送离线命令队列
        queue_key = f"offline_cmd_queue:{device_id}"
        queue_size = self.redis.llen(queue_key)

        if queue_size > 0:
            # 批量取出命令（去重 + 合并）
            commands = []
            while True:
                cmd_json = self.redis.lpop(queue_key)
                if not cmd_json:
                    break
                commands.append(json.loads(cmd_json))

            # 去重：相同command保留最新的
            deduped = self._dedup_commands(commands)

            # 合并：相同属性的设置命令合并
            merged = self._merge_commands(deduped)

            # 按优先级排序
            merged.sort(key=lambda x: 0 if x["priority"] == "high" else 1)

            # 逐条推送
            for cmd in merged:
                self.mqtt_client.publish(f"cmd/{device_id}",
                    json.dumps(cmd), qos=1)
                # 更新DB状态
                self.db.update("offline_commands",
                    {"status": "delivered", "delivered_at": now()},
                    {"device_id": device_id, "command": cmd["command"],
                     "status": "queued"})

            return {"synced": True, "commands_delivered": len(merged)}

        return {"synced": True, "commands_delivered": 0}

    def _dedup_commands(self, commands):
        """去重：相同command保留最新的"""
        seen = {}
        for cmd in commands:
            key = cmd["command"]
            seen[key] = cmd  # 后出现的覆盖前面的
        return list(seen.values())

    def _merge_commands(self, commands):
        """合并：相同属性的设置命令合并params"""
        set_commands = {}
        other_commands = []

        for cmd in commands:
            if cmd["command"] == "set_properties":
                # 合并属性设置
                if "set_properties" in set_commands:
                    set_commands["set_properties"]["params"].update(cmd["params"])
                else:
                    set_commands["set_properties"] = dict(cmd)
            else:
                other_commands.append(cmd)

        return list(set_commands.values()) + other_commands

    def get_queue_status(self, device_id):
        """获取离线命令队列状态"""
        queue_key = f"offline_cmd_queue:{device_id}"
        queue_size = self.redis.llen(queue_key)
        pending_db = self.db.query(
            "SELECT COUNT(*) as cnt FROM offline_commands "
            "WHERE device_id = %s AND status = 'queued'", device_id)
        return {
            "redis_queue_size": queue_size,
            "db_pending_count": pending_db[0]["cnt"] if pending_db else 0
        }
```

## IoT 网关管理

### 网关注册与下游设备代理

```python
class IoTGatewayManager:
    """IoT网关管理：注册、代理、协议转换、健康监控"""

    def __init__(self, db, redis, mqtt_client):
        self.db = db
        self.redis = redis
        self.mqtt_client = mqtt_client

    def register_gateway(self, gateway_id, config):
        """注册网关"""
        gateway = {
            "gateway_id": gateway_id,
            "name": config["name"],
            "location": config.get("location", ""),
            "protocols_supported": json.dumps(config.get("protocols", ["mqtt"])),
            "max_downstream_devices": config.get("max_devices", 100),
            "status": "online",
            "last_heartbeat": now(),
            "created_at": now()
        }
        self.db.insert("iot_gateways", gateway)
        self.redis.hset(f"gateway:{gateway_id}", mapping={
            "status": "online",
            "last_heartbeat": now().isoformat(),
            "connected_devices": 0
        })
        return {"gateway_id": gateway_id, "status": "registered"}

    def register_downstream_device(self, gateway_id, device_id, protocol):
        """注册下游设备到网关"""
        gateway = self.redis.hgetall(f"gateway:{gateway_id}")
        if not gateway:
            raise ValueError(f"网关 {gateway_id} 不存在")

        current_devices = int(gateway.get("connected_devices", 0))
        max_devices = self.db.get("iot_gateways", gateway_id)["max_downstream_devices"]
        if current_devices >= max_devices:
            raise ValueError(f"网关 {gateway_id} 已达最大设备数 {max_devices}")

        # 绑定设备到网关
        self.db.insert("gateway_devices", {
            "gateway_id": gateway_id,
            "device_id": device_id,
            "protocol": protocol,
            "status": "online",
            "registered_at": now()
        })

        # 更新网关设备计数
        self.redis.hincrby(f"gateway:{gateway_id}", "connected_devices", 1)
        # 设备→网关映射（快速查找设备所属网关）
        self.redis.set(f"device_gateway:{device_id}", gateway_id)

        return {"device_id": device_id, "gateway_id": gateway_id, "protocol": protocol}

    def proxy_message_to_device(self, device_id, message):
        """通过网关代理消息到下游设备"""
        gateway_id = self.redis.get(f"device_gateway:{device_id}")
        if not gateway_id:
            raise ValueError(f"设备 {device_id} 未绑定网关")

        # 获取设备协议
        device_binding = self.db.query(
            "SELECT protocol FROM gateway_devices "
            "WHERE gateway_id = %s AND device_id = %s",
            gateway_id, device_id)
        if not device_binding:
            raise ValueError("设备未注册到网关")

        protocol = device_binding[0]["protocol"]

        # 协议转换
        translated = self._translate_protocol(message, "mqtt", protocol)

        # 通过网关转发
        self.mqtt_client.publish(
            f"gateway/{gateway_id}/downstream/{device_id}",
            json.dumps(translated), qos=1)

        return {"status": "sent", "gateway_id": gateway_id, "protocol": protocol}

    def _translate_protocol(self, message, from_proto, to_proto):
        """协议转换（MQTT ↔ CoAP ↔ HTTP）"""
        if from_proto == to_proto:
            return message

        if from_proto == "mqtt" and to_proto == "coap":
            # MQTT → CoAP：简化消息格式
            return {
                "method": "POST" if message.get("command") else "GET",
                "uri": f"/devices/{message.get('device_id', '')}",
                "payload": message.get("params", {}),
                "content_format": "json"
            }
        elif from_proto == "mqtt" and to_proto == "http":
            # MQTT → HTTP：RESTful 格式
            return {
                "method": "POST",
                "url": f"http://device/api/{message.get('device_id', '')}",
                "headers": {"Content-Type": "application/json"},
                "body": message.get("params", {})
            }
        elif from_proto == "coap" and to_proto == "mqtt":
            # CoAP → MQTT：封装为MQTT topic消息
            return {
                "topic": f"device/{message.get('uri', '').split('/')[-1]}",
                "payload": message.get("payload", {}),
                "qos": 1
            }
        elif from_proto == "http" and to_proto == "mqtt":
            # HTTP → MQTT：提取body作为payload
            return {
                "topic": f"device/{message.get('url', '').split('/')[-1]}",
                "payload": message.get("body", {}),
                "qos": 1
            }

        return message  # 不支持的转换，原样返回

    def update_heartbeat(self, gateway_id):
        """更新网关心跳"""
        self.redis.hset(f"gateway:{gateway_id}", mapping={
            "last_heartbeat": now().isoformat(),
            "status": "online"
        })
        self.redis.expire(f"gateway:{gateway_id}", 120)  # 120秒无心跳标记离线

    def check_gateway_health(self):
        """检查所有网关健康状态"""
        gateways = self.db.query(
            "SELECT gateway_id FROM iot_gateways WHERE status = 'online'")
        unhealthy = []

        for gw in gateways:
            gw_data = self.redis.hgetall(f"gateway:{gw['gateway_id']}")
            if not gw_data:
                continue
            last_hb = datetime.fromisoformat(gw_data["last_heartbeat"])
            if (now() - last_hb).total_seconds() > 120:
                unhealthy.append(gw["gateway_id"])
                # 标记网关离线
                self.db.update("iot_gateways",
                    {"status": "offline"}, {"gateway_id": gw["gateway_id"]})
                self.redis.hset(f"gateway:{gw['gateway_id']}", "status", "offline")

                # 通知下游设备离线
                self._notify_downstream_offline(gw["gateway_id"])

        return {"total": len(gateways), "unhealthy": unhealthy}

    def _notify_downstream_offline(self, gateway_id):
        """通知下游设备网关离线"""
        devices = self.db.query(
            "SELECT device_id FROM gateway_devices "
            "WHERE gateway_id = %s AND status = 'online'",
            gateway_id)
        for d in devices:
            self.redis.set(f"device_offline_via_gateway:{d['device_id']}", gateway_id)
            # 触发设备离线事件
            self.mqtt_client.publish(f"device/status/{d['device_id']}",
                json.dumps({"status": "gateway_offline",
                             "gateway_id": gateway_id}), qos=1)
```

## 异常场景：规则引擎无限循环检测（深度分析）

```
触发：规则A → 控制设备 → 规则B监听变化 → 触发规则A → 形成无限循环
典型场景：
  规则A：温度 > 30° → 开启空调（设定温度25°）
  规则B：温度 < 26° → 关闭空调
  结果：空调反复开关 → 设备损坏
检测机制：
  1. 同一规则10分钟内触发 > 5次 → 振荡检测
  2. 规则执行链深度 > 5 → 循环检测
  3. 规则DAG图静态分析 → 检测潜在循环
  4. 实时执行追踪：记录 rule_id → device_id → metric 变更链
处理流程：
  1. 自动暂停触发振荡的规则（标记pause_reason）
  2. 通知管理员规则冲突（包含冲突链路详情）
  3. 增加规则执行冷却期（5分钟内不重复触发）
  4. 管理员确认修复后手动重新启用规则
预防措施：
  1. 规则创建时静态检测循环（DAG拓扑排序）
  2. 执行深度限制（链深度 > 5 自动终止）
  3. 振荡检测（短时间高频触发自动暂停）
  4. 规则间互斥声明（显式标注哪些规则不能共存）
  5. 规则沙箱模拟执行（上线前模拟验证）
```

## 异常场景：设备影子版本冲突解决（深度分析）

```
触发：App端和设备端同时更新设备影子 → 版本号冲突
典型场景：
  1. App设置温度25°（version=5），同时设备上报当前温度22°（version=4）
  2. App读取影子时version=4，提交更新时version已变为5 → 冲突
  3. 多个App同时控制同一设备 → 并发写冲突
检测机制：
  1. 客户端携带version字段，服务端比对当前version
  2. version不匹配 → 返回409 Conflict + 最新影子状态
  3. 毫秒级时间窗口内的并发写 → 检测到冲突
处理流程：
  1. 乐观并发控制：客户端必须携带version
  2. 版本不匹配 → 返回409 + 最新desired + reported + delta
  3. 客户端重新获取最新影子后合并变更重试
  4. 合并策略：
     - 属性级合并：不同属性变更互不冲突
     - 同属性冲突：后写入覆盖（Last-Write-Wins）
     - 业务语义合并：如温度设置取最新
  5. 冲突日志记录（用于分析和优化）
预防措施：
  1. 强制客户端携带version（API层校验）
  2. 属性级细粒度版本控制（每个属性独立version）
  3. 设备端只上报reported，App端只写desired → 减少冲突面
  4. 操作锁：关键设备同一时间只允许一个控制端
  5. 冲突自动重试（客户端自动重新获取+合并+提交）
```

## IoT 数据管道编排完整实现

```python
class IoTPipelineOrchestrator:
    """IoT 数据管道编排：MQTT → Kafka → Flink → TimescaleDB"""

    def process_message(self, topic, payload):
        """处理 MQTT 消息"""
        # 1. Schema 验证
        device_type = topic.split("/")[1]
        schema = self.schema_registry.get_schema(device_type)
        if not schema:
            self._send_to_dlq(topic, payload, "unknown_device_type")
            return

        try:
            validated = schema.validate(json.loads(payload))
        except ValidationError as e:
            self._send_to_dlq(topic, payload, f"schema_validation: {e}")
            return

        # 2. 值范围检查
        for field, limits in schema.get("range_limits", {}).items():
            value = validated.get(field)
            if value is not None and (value < limits["min"] or value > limits["max"]):
                self._send_to_dlq(topic, payload,
                    f"out_of_range: {field}={value}")
                return

        # 3. 写入 Kafka
        self.kafka.produce("iot_validated", json.dumps({
            "device_id": validated["device_id"],
            "device_type": device_type,
            "timestamp": validated["timestamp"],
            "measurements": validated["measurements"],
            "received_at": now().isoformat()
        }))

    def run_flink_aggregation(self):
        """Flink SQL 窗口聚合"""
        # 1 分钟聚合
        self.flink.execute("""
            INSERT INTO iot_1min_agg
            SELECT device_id,
                   TUMBLE_START(event_time, INTERVAL '1' MINUTE) as window_start,
                   AVG(temperature) as temp_avg,
                   MIN(temperature) as temp_min,
                   MAX(temperature) as temp_max,
                   COUNT(*) as sample_count
            FROM iot_validated_stream
            GROUP BY device_id, TUMBLE(event_time, INTERVAL '1' MINUTE)
        """)

        # 5 分钟聚合
        self.flink.execute("""
            INSERT INTO iot_5min_agg
            SELECT device_id,
                   TUMBLE_START(event_time, INTERVAL '5' MINUTE) as window_start,
                   AVG(temperature) as temp_avg,
                   STDDEV(temperature) as temp_stddev,
                   COUNT(*) as sample_count
            FROM iot_validated_stream
            GROUP BY device_id, TUMBLE(event_time, INTERVAL '5' MINUTE)
        """)

        # 1 小时降采样
        self.flink.execute("""
            INSERT INTO iot_1hour_agg
            SELECT device_id,
                   TUMBLE_START(event_time, INTERVAL '1' HOUR) as window_start,
                   AVG(temperature) as temp_avg,
                   MIN(temperature) as temp_min,
                   MAX(temperature) as temp_max,
                   PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY temperature) as temp_p95,
                   COUNT(*) as sample_count
            FROM iot_validated_stream
            GROUP BY device_id, TUMBLE(event_time, INTERVAL '1' HOUR)
        """)

    def monitor_pipeline_health(self):
        """管道健康监控"""
        stages = {
            "mqtt_to_kafka": self._check_mqtt_kafka_latency(),
            "kafka_to_flink": self._check_kafka_flink_lag(),
            "flink_to_db": self._check_flink_db_latency(),
        }

        # 端到端延迟
        e2e_latency = sum(s["latency_ms"] for s in stages.values())

        # 背压检测
        kafka_lag = self._get_kafka_consumer_lag()
        if kafka_lag > 100000:
            self._throttle_mqtt_broker(rate_limit=1000)
            self.alert(f"Kafka lag {kafka_lag}，已限流 MQTT 接入")

        return {"stages": stages, "e2e_latency_ms": e2e_latency,
                "kafka_lag": kafka_lag}

    def _send_to_dlq(self, topic, payload, reason):
        """发送到死信队列"""
        self.kafka.produce("iot_dlq", json.dumps({
            "original_topic": topic,
            "payload": payload,
            "error_reason": reason,
            "failed_at": now().isoformat()
        }))
```

## 设备 OTA 固件管理

```python
class DeviceOTAManager:
    """设备 OTA 固件管理：Delta 更新 + 灰度 + 双分区"""

    def generate_delta_update(self, old_version, new_version):
        """生成差量更新（bsdiff）"""
        old_firmware = self.storage.get(f"firmware/{old_version}.bin")
        new_firmware = self.storage.get(f"firmware/{new_version}.bin")

        # bsdiff 生成差量
        delta = bsdiff.diff(old_firmware, new_firmware)

        delta_path = f"firmware/delta/{old_version}_to_{new_version}.delta"
        self.storage.upload(delta_path, delta)

        # 计算节省比例
        savings = (1 - len(delta) / len(new_firmware)) * 100

        return {"delta_path": delta_path,
                "delta_size_bytes": len(delta),
                "full_size_bytes": len(new_firmware),
                "bandwidth_savings_pct": round(savings, 1)}

    def staged_rollout(self, firmware_id, device_model, stages=None):
        """灰度发布：1%→5%→25%→100%"""
        stages = stages or [
            {"percentage": 1, "hold_minutes": 30, "max_offline_rate": 0.02},
            {"percentage": 5, "hold_minutes": 60, "max_offline_rate": 0.02},
            {"percentage": 25, "hold_minutes": 120, "max_offline_rate": 0.01},
            {"percentage": 100, "hold_minutes": 0, "max_offline_rate": 0.01},
        ]

        rollout_id = str(uuid4())
        for stage in stages:
            # 选择目标设备
            targets = self._select_devices(device_model, stage["percentage"])
            for device_id in targets:
                self._push_update(device_id, firmware_id)

            # 等待观察期
            time.sleep(stage["hold_minutes"] * 60)

            # 检查结果
            offline_rate = self._check_offline_rate(targets)
            if offline_rate > stage["max_offline_rate"]:
                # 自动回滚
                self._rollback(rollout_id, firmware_id, targets)
                return {"status": "rolled_back", "offline_rate": offline_rate}

        return {"status": "completed"}

    def _push_update(self, device_id, firmware_id):
        """推送固件更新（双分区 A/B）"""
        firmware = self.db.get_firmware(firmware_id)

        # 签名验证（ECDSA P-256）
        signature = self.signer.sign(firmware["binary_url"])

        # 推送到设备（写入备用分区）
        self.mqtt_client.publish(f"device/{device_id}/ota", json.dumps({
            "action": "update",
            "firmware_id": firmware_id,
            "version": firmware["version"],
            "delta_url": firmware.get("delta_url"),
            "full_url": firmware["binary_url"],
            "signature": signature,
            "target_partition": "B",  # 写入备用分区
            "size_bytes": firmware["size_bytes"],
            "sha256": firmware["sha256"]
        }))
```

## 异常场景补充

### 场景：Flink Checkpoint 失败

```
触发：Flink checkpoint 超时 → Job 重启 → 数据重复处理
检测：
  1. Checkpoint 失败告警
  2. Job 重启 → 数据可能重复
处理：
  1. 下游幂等消费（TimescaleDB INSERT ON CONFLICT）
  2. 恢复 checkpoint 后重新消费
  3. 数据去重：基于 device_id + timestamp 唯一约束
预防：幂等消费 + checkpoint 定期验证 + 去重约束
```

### 场景：Delta 更新损坏

```
触发：Delta 二进制差量文件损坏 → 设备应用差量失败 → OTA 失败
检测：
  1. 设备报告 delta apply 失败 → 损坏
  2. SHA256 校验失败 → 文件损坏
处理：
  1. 自动回退到全量更新
  2. 重新生成 delta 文件
  3. 全量更新成功后标记 delta 为无效
预防：SHA256 校验 + 全量回退 + 差量文件验证
```

## IoT 设备生命周期管理完整实现

```python
class DeviceLifecycleManager:
    """设备生命周期：注册 → 配置 → 激活 → 退役"""

    def register_device(self, device_info):
        """注册设备"""
        device_id = str(uuid4())
        # 生成设备证书
        cert = self.pkI.issue_device_certificate(
            device_id, validity_days=365)

        self.db.insert("devices", {
            "device_id": device_id,
            "device_type": device_info["type"],
            "model": device_info["model"],
            "firmware_version": device_info.get("firmware_version"),
            "location": device_info.get("location"),
            "certificate_pem": cert["certificate"],
            "certificate_expires_at": cert["expires_at"],
            "status": "registered",
            "registered_at": now()
        })

        # 配置 MQTT 主题权限
        self.mqtt_auth.grant(device_id, [
            f"telemetry/{device_id}/publish",
            f"command/{device_id}/subscribe",
            f"status/{device_id}/publish"
        ])

        return {"device_id": device_id, "certificate": cert["certificate"]}

    def activate_device(self, device_id):
        """激活设备"""
        self.db.update("devices",
            {"status": "active", "activated_at": now()},
            {"device_id": device_id})

    def decommission_device(self, device_id, reason):
        """退役设备"""
        # 1. 撤销证书
        self.pki.revoke_certificate(device_id)

        # 2. 撤销 MQTT 权限
        self.mqtt_auth.revoke_all(device_id)

        # 3. 归档历史数据
        self._archive_device_data(device_id)

        # 4. 更新状态
        self.db.update("devices",
            {"status": "decommissioned",
             "decommissioned_reason": reason,
             "decommissioned_at": now()},
            {"device_id": device_id})

    def rotate_certificates(self):
        """批量轮换设备证书（到期前 7 天）"""
        expiring = self.db.query(
            "SELECT * FROM devices WHERE status = 'active' "
            "AND certificate_expires_at < NOW() + INTERVAL 7 DAY "
            "AND certificate_expires_at > NOW()")

        for device in expiring:
            try:
                new_cert = self.pki.issue_device_certificate(
                    device["device_id"], validity_days=365)

                # 推送新证书到设备
                self.mqtt_client.publish(
                    f"command/{device['device_id']}/cert_update",
                    json.dumps({
                        "action": "rotate_cert",
                        "new_certificate": new_cert["certificate"],
                        "expires_at": new_cert["expires_at"].isoformat()
                    }))

                self.db.update("devices", {
                    "certificate_pem": new_cert["certificate"],
                    "certificate_expires_at": new_cert["expires_at"]
                }, {"device_id": device["device_id"]})

            except Exception as e:
                self.alert(f"设备 {device['device_id']} 证书轮换失败: {e}")
```

## 时序数据压缩与降采样

```python
class TimeseriesCompressionManager:
    """时序数据压缩：Gorilla 编码 + 降采样 + 分层存储"""

    RETENTION_POLICIES = {
        "raw": {"retention_days": 30, "compression_after_days": 7},
        "1min_agg": {"retention_days": 90},
        "1hour_agg": {"retention_days": 365},
        "daily_agg": {"retention_days": None},  # 永久保留
    }

    def setup_continuous_aggregates(self, table_name):
        """配置 TimescaleDB 连续聚合"""
        # 1 分钟聚合
        self.db.execute(f"""
            CREATE MATERIALIZED VIEW {table_name}_1min
            WITH (timescaledb.continuous) AS
            SELECT time_bucket('1 minute', timestamp) as bucket,
                   device_id,
                   AVG(value) as avg_value,
                   MIN(value) as min_value,
                   MAX(value) as max_value,
                   COUNT(*) as sample_count
            FROM {table_name}
            GROUP BY bucket, device_id
        """)

        # 1 小时聚合
        self.db.execute(f"""
            CREATE MATERIALIZED VIEW {table_name}_1hour
            WITH (timescaledb.continuous) AS
            SELECT time_bucket('1 hour', bucket) as bucket,
                   device_id,
                   AVG(avg_value) as avg_value,
                   MIN(min_value) as min_value,
                   MAX(max_value) as max_value,
                   SUM(sample_count) as sample_count
            FROM {table_name}_1min
            GROUP BY bucket, device_id
        """)

        # 每日聚合
        self.db.execute(f"""
            CREATE MATERIALIZED VIEW {table_name}_daily
            WITH (timescaledb.continuous) AS
            SELECT time_bucket('1 day', bucket) as bucket,
                   device_id,
                   AVG(avg_value) as avg_value,
                   MIN(min_value) as min_value,
                   MAX(max_value) as max_value,
                   SUM(sample_count) as sample_count
            FROM {table_name}_1hour
            GROUP BY bucket, device_id
        """)

    def setup_compression_policy(self, table_name):
        """配置压缩策略（7 天后压缩，90-95% 压缩率）"""
        self.db.execute(f"""
            ALTER TABLE {table_name} SET (
                timescaledb.compress,
                timescaledb.compress_segmentby = 'device_id',
                timescaledb.compress_orderby = 'timestamp DESC'
            )
        """)

        # 自动压缩 7 天以上的数据块
        self.db.execute(f"""
            SELECT add_compression_policy('{table_name}',
                INTERVAL '7 days')
        """)

    def verify_compression(self, table_name):
        """验证压缩效果"""
        stats = self.db.query_one(f"""
            SELECT
                SUM(before_compression_total_bytes) as uncompressed,
                SUM(after_compression_total_bytes) as compressed,
                ROUND((1 - SUM(after_compression_total_bytes)::float /
                    NULLIF(SUM(before_compression_total_bytes), 0)) * 100, 1) as ratio
            FROM timescaledb_information.compressed_chunk_stats
            WHERE hypertable_name = '{table_name}'
        """)
        return {"uncompressed_gb": round(stats["uncompressed"] / 1073741824, 1),
                "compressed_gb": round(stats["compressed"] / 1073741824, 1),
                "compression_ratio_pct": stats["ratio"]}
```

## 异常场景补充

### 场景：证书轮换导致设备批量断连

```
触发：新证书推送后旧证书立即失效 → 设备未加载新证书 → 批量断连
检测：
  1. 设备离线率突增 → 证书问题
  2. MQTT 连接失败日志 → 认证错误
处理：
  1. 临时恢复旧证书有效期（24 小时宽限）
  2. 检查设备证书加载逻辑
  3. 改为双证书过渡期（新旧都有效）
预防：双证书过渡 + 推送后确认加载 + 渐进式轮换
```

### 场景：压缩策略损坏历史数据

```
触发：压缩后解压数据不完整 → 历史查询返回错误结果
检测：
  1. 压缩前后数据校验和对比 → 不一致
  2. 查询结果异常 → 压缩损坏
处理：
  1. 从备份恢复受影响的压缩块
  2. 禁用自动压缩策略
  3. 修复后逐步重新压缩
预防：压缩后校验 + 备份保留 + 灰度压缩
```

## IoT 设备生命周期管理深度实现

```python
class DeviceProvisioningService:
    """设备配置服务：证书生成 → MQTT 配置 → 初始状态 → 激活"""

    def register_device(self, device_info):
        """注册设备：分配 device_id、凭证、元数据"""
        device_id = f"dev-{device_info['type'][:4]}-{str(uuid4())[:8]}"

        # 验证设备元数据完整性
        required_fields = ["type", "model", "firmware_version", "location", "owner"]
        missing = [f for f in required_fields if f not in device_info]
        if missing:
            return {"status": "rejected", "reason": f"缺少必填字段: {missing}"}

        # 生成 X.509 设备证书
        cert_result = self._generate_device_certificate(device_id)

        # 存储设备记录
        self.db.insert("devices", {
            "device_id": device_id,
            "device_type": device_info["type"],
            "model": device_info["model"],
            "firmware_version": device_info["firmware_version"],
            "location": device_info["location"],
            "owner": device_info["owner"],
            "metadata": json.dumps(device_info.get("metadata", {})),
            "certificate_serial": cert_result["serial"],
            "certificate_pem": cert_result["certificate"],
            "certificate_expires_at": cert_result["expires_at"],
            "status": "registered",
            "registered_at": now(),
        })

        return {
            "status": "registered",
            "device_id": device_id,
            "certificate": cert_result["certificate"],
            "ca_certificate": cert_result["ca_certificate"],
            "next_step": "provision",
        }

    def provision_device(self, device_id):
        """配置设备：生成证书 → 配置 MQTT → 设置初始状态 → 激活"""
        device = self.db.get("devices", device_id)
        if device["status"] != "registered":
            return {"status": "error",
                    "reason": f"设备状态为 {device['status']}，无法配置"}

        # 1. 生成设备证书（如果尚未生成）
        if not device.get("certificate_pem"):
            cert = self._generate_device_certificate(device_id)
            self.db.update("devices", {
                "certificate_pem": cert["certificate"],
                "certificate_expires_at": cert["expires_at"],
            }, {"device_id": device_id})

        # 2. 配置 MQTT 主题权限
        mqtt_topics = self._generate_mqtt_topics(device_id, device["device_type"])
        self.mqtt_auth.configure_permissions(device_id, {
            "publish": mqtt_topics["publish"],
            "subscribe": mqtt_topics["subscribe"],
        })

        # 3. 设置初始状态
        initial_state = self._get_initial_state(device["device_type"])
        self.db.insert("device_states", {
            "device_id": device_id,
            "state": json.dumps(initial_state),
            "state_version": 1,
            "updated_at": now(),
        })

        # 4. 推送配置到设备
        self.mqtt_client.publish(f"command/{device_id}/provision", json.dumps({
            "action": "provision",
            "device_id": device_id,
            "mqtt_topics": mqtt_topics,
            "initial_state": initial_state,
            "certificate": device.get("certificate_pem"),
            "telemetry_interval_seconds": 1,
            "heartbeat_interval_seconds": 30,
        }))

        # 5. 激活设备
        self.db.update("devices",
            {"status": "active", "activated_at": now(), "provisioned_at": now()},
            {"device_id": device_id})

        return {"status": "provisioned", "device_id": device_id}

    def _generate_device_certificate(self, device_id):
        """生成 X.509 设备证书（90 天有效期）"""
        cert = self.pki.issue_certificate(
            common_name=device_id,
            organization="IoT Platform",
            validity_days=90,
            key_type="ECDSA",
            key_size=256,  # P-256 曲线
            extensions={
                "subject_alt_name": [f"URI:urn:iot:device:{device_id}"],
                "key_usage": ["digital_signature", "key_encipherment"],
                "extended_key_usage": ["client_auth"],
            },
        )
        return cert

    def _generate_mqtt_topics(self, device_id, device_type):
        """生成设备 MQTT 主题"""
        return {
            "publish": [
                f"telemetry/{device_type}/{device_id}",       # 遥测数据上报
                f"status/{device_id}",                          # 设备状态上报
                f"event/{device_type}/{device_id}",            # 事件上报
                f"heartbeat/{device_id}",                       # 心跳
            ],
            "subscribe": [
                f"command/{device_id}",                         # 命令下发
                f"config/{device_id}",                          # 配置下发
                f"ota/{device_type}/{device_id}",              # OTA 更新
                f"cert/{device_id}",                            # 证书更新
            ],
        }


class DeviceCertificateRotationService:
    """设备证书轮换服务：90 天有效期，到期前 7 天自动续期"""

    CERT_CONFIG = {
        "validity_days": 90,
        "renewal_before_expiry_days": 7,
        "grace_period_days": 3,  # 新旧证书重叠期
        "batch_size": 100,       # 每批轮换数量
        "rotation_timeout_minutes": 30,
    }

    def rotate_expiring_certificates(self):
        """轮换即将过期的设备证书"""
        renewal_threshold = now() + timedelta(
            days=self.CERT_CONFIG["renewal_before_expiry_days"])

        expiring = self.db.query(
            "SELECT * FROM devices WHERE status = 'active' "
            "AND certificate_expires_at < %s "
            "AND certificate_expires_at > NOW() "
            "ORDER BY certificate_expires_at ASC "
            "LIMIT %s",
            renewal_threshold, self.CERT_CONFIG["batch_size"])

        results = {"success": 0, "failed": 0, "pending": 0}

        for device in expiring:
            try:
                result = self._rotate_single_certificate(device)
                results[result] += 1
            except Exception as e:
                results["failed"] += 1
                self.alert(
                    f"设备 {device['device_id']} 证书轮换异常: {e}")

        # 如果有失败的 → 告警
        if results["failed"] > 0:
            self.alerting.send(
                severity="warning",
                title=f"证书轮换: {results['failed']} 台设备失败",
                message=f"成功: {results['success']}, 失败: {results['failed']}, "
                        f"待确认: {results['pending']}",
            )

        return results

    def _rotate_single_certificate(self, device):
        """轮换单台设备证书（双证书过渡期）"""
        device_id = device["device_id"]

        # 1. 生成新证书
        new_cert = self._generate_device_certificate(device_id)

        # 2. 推送新证书到设备（设备应同时接受新旧证书）
        self.mqtt_client.publish(f"cert/{device_id}", json.dumps({
            "action": "rotate_certificate",
            "new_certificate": new_cert["certificate"],
            "old_certificate_serial": device["certificate_serial"],
            "expires_at": new_cert["expires_at"].isoformat(),
            "grace_period_until": (new_cert["expires_at"] - timedelta(
                days=self.CERT_CONFIG["validity_days"] -
                self.CERT_CONFIG["grace_period_days"])).isoformat(),
        }))

        # 3. 等待设备确认收到新证书
        confirmation = self._wait_for_cert_confirmation(
            device_id, timeout_minutes=self.CERT_CONFIG["rotation_timeout_minutes"])

        if confirmation and confirmation.get("loaded"):
            # 设备已加载新证书 → 更新数据库
            self.db.update("devices", {
                "certificate_pem": new_cert["certificate"],
                "certificate_serial": new_cert["serial"],
                "certificate_expires_at": new_cert["expires_at"],
                "cert_rotated_at": now(),
            }, {"device_id": device_id})
            return "success"
        else:
            # 设备未确认 → 标记待处理
            self.db.insert("cert_rotation_pending", {
                "device_id": device_id,
                "new_cert_serial": new_cert["serial"],
                "new_certificate": new_cert["certificate"],
                "new_expires_at": new_cert["expires_at"],
                "attempts": 1,
                "last_attempt_at": now(),
            })
            return "pending"

    def _wait_for_cert_confirmation(self, device_id, timeout_minutes):
        """等待设备确认证书加载"""
        deadline = now() + timedelta(minutes=timeout_minutes)
        while now() < deadline:
            msg = self.mqtt_client.get_last_message(f"status/{device_id}")
            if msg and msg.get("cert_status") == "new_cert_loaded":
                return {"loaded": True}
            time.sleep(5)
        return None


class DeviceGroupManager:
    """设备分组管理：按位置/类型/固件分组，支持批量操作"""

    def create_group(self, group_name, criteria):
        """创建设备组"""
        group_id = generate_id()
        self.db.insert("device_groups", {
            "group_id": group_id,
            "name": group_name,
            "criteria": json.dumps(criteria),
            "created_at": now(),
        })

        # 计算组内设备
        devices = self._find_matching_devices(criteria)
        for device in devices:
            self.db.insert("device_group_members", {
                "group_id": group_id,
                "device_id": device["device_id"],
                "added_at": now(),
            })

        return {"group_id": group_id, "device_count": len(devices)}

    def bulk_operation(self, group_id, operation, params):
        """对设备组执行批量操作"""
        devices = self.db.query(
            "SELECT device_id FROM device_group_members "
            "WHERE group_id = %s", group_id)

        results = {"success": 0, "failed": 0}

        for device in devices:
            device_id = device["device_id"]

            if operation == "firmware_update":
                result = self.ota_service.push_update(device_id, params["firmware_id"])
            elif operation == "config_update":
                result = self._push_config(device_id, params["config"])
            elif operation == "cert_rotate":
                result = self.cert_service._rotate_single_certificate(
                    self.db.get("devices", device_id))
            elif operation == "reboot":
                self.mqtt_client.publish(f"command/{device_id}", json.dumps({
                    "action": "reboot", "reason": params.get("reason", "bulk_operation"),
                }))
                result = "success"
            else:
                result = "unknown_operation"

            if result in ("success", True):
                results["success"] += 1
            else:
                results["failed"] += 1

        return results

    def _find_matching_devices(self, criteria):
        """按条件查找设备"""
        query = "SELECT * FROM devices WHERE status = 'active'"
        params = []

        if "device_type" in criteria:
            query += " AND device_type = %s"
            params.append(criteria["device_type"])
        if "location" in criteria:
            query += " AND location = %s"
            params.append(criteria["location"])
        if "firmware_version" in criteria:
            query += " AND firmware_version = %s"
            params.append(criteria["firmware_version"])

        return self.db.query(query, *params)


class DeviceDecommissioningService:
    """设备退役服务：撤销证书 → 归档数据 → 注销"""

    def decommission_device(self, device_id, reason, requested_by):
        """退役设备完整流程"""
        device = self.db.get("devices", device_id)
        if device["status"] == "decommissioned":
            return {"status": "already_decommissioned"}

        # 1. 撤销设备证书
        self.pki.revoke_certificate(
            serial=device["certificate_serial"],
            reason="superseded",
            comment=f"设备退役: {reason}",
        )

        # 2. 撤销 MQTT 权限
        self.mqtt_auth.revoke_all(device_id)

        # 3. 归档历史时序数据
        self._archive_device_timeseries(device_id)

        # 4. 清除设备缓存和会话
        self._clear_device_sessions(device_id)

        # 5. 更新状态
        self.db.update("devices", {
            "status": "decommissioned",
            "decommissioned_reason": reason,
            "decommissioned_by": requested_by,
            "decommissioned_at": now(),
        }, {"device_id": device_id})

        # 6. 审计记录
        self.audit_log.record({
            "action": "device_decommissioned",
            "device_id": device_id,
            "reason": reason,
            "performed_by": requested_by,
            "timestamp": now(),
        })

        return {"status": "decommissioned", "device_id": device_id}

    def _archive_device_timeseries(self, device_id):
        """归档设备历史时序数据"""
        # 导出设备所有时序数据到归档存储
        data_stats = self.db.query_one(
            "SELECT COUNT(*) as row_count, "
            "MIN(timestamp) as first_record, "
            "MAX(timestamp) as last_record "
            "FROM device_telemetry "
            "WHERE device_id = %s", device_id)

        if data_stats["row_count"] > 0:
            # 导出到 S3 归档
            export_path = f"archive/devices/{device_id}/telemetry.parquet"
            self.db.export_to_s3(
                query=f"SELECT * FROM device_telemetry WHERE device_id = '{device_id}'",
                s3_path=export_path,
                format="parquet",
            )

            # 保留归档记录（不删除原始数据，仅标记已归档）
            self.db.insert("device_data_archives", {
                "device_id": device_id,
                "archive_path": export_path,
                "record_count": data_stats["row_count"],
                "date_range": f"{data_stats['first_record']} ~ {data_stats['last_record']}",
                "archived_at": now(),
            })
```

## 时序数据压缩与降采样深度实现

```python
class GorillaCompressionEngine:
    """Gorilla 风格浮点压缩：Delta-of-Delta 时间戳 + XOR 浮点值编码"""

    def compress_block(self, data_points):
        """压缩一个数据块（同设备同指标）"""
        if not data_points:
            return b""

        # 按时间排序
        data_points.sort(key=lambda p: p["timestamp"])

        compressed = bytearray()

        # === 时间戳压缩：Delta-of-Delta 编码 ===
        first_ts = int(data_points[0]["timestamp"].timestamp() * 1000)
        compressed.extend(self._encode_varint(first_ts))

        prev_ts = first_ts
        prev_delta = 0  # 第一个 delta 就是 first_ts 本身

        # === 浮点值压缩：XOR 编码 ===
        first_val_bits = self._float_to_bits(data_points[0]["value"])
        compressed.extend(self._encode_varint(first_val_bits))
        prev_val_bits = first_val_bits
        prev_leading_zeros = None
        prev_trailing_zeros = None

        for i in range(1, len(data_points)):
            # --- 时间戳编码 ---
            curr_ts = int(data_points[i]["timestamp"].timestamp() * 1000)
            delta = curr_ts - prev_ts
            delta_of_delta = delta - prev_delta

            if delta_of_delta == 0:
                # 写入 0 bit（1 位）
                compressed.append(0)  # 标记位
            else:
                # 写入 1 bit + 编码值
                compressed.append(1)  # 标记位
                encoded_dod = self._encode_delta_of_delta(delta_of_delta)
                compressed.extend(encoded_dod)

            prev_delta = delta
            prev_ts = curr_ts

            # --- 浮点值 XOR 编码 ---
            curr_val_bits = self._float_to_bits(data_points[i]["value"])
            xor = curr_val_bits ^ prev_val_bits

            if xor == 0:
                # 写入 0 bit（值与前一个相同）
                compressed.append(0)
            else:
                compressed.append(1)
                leading_zeros = self._count_leading_zeros(xor)
                trailing_zeros = self._count_trailing_zeros(xor)

                # 判断是否可以复用前一个的 leading/trailing zeros
                if (prev_leading_zeros is not None and
                    leading_zeros >= prev_leading_zeros and
                    trailing_zeros >= prev_trailing_zeros):
                    # 复用前一个的有效位范围 → 写入 1 bit + meaningful bits
                    compressed.append(1)
                    meaningful_bits = xor >> prev_trailing_zeros
                    meaningful_len = 64 - prev_leading_zeros - prev_trailing_zeros
                    compressed.extend(self._encode_bits(meaningful_bits, meaningful_len))
                else:
                    # 写入新的 leading zeros + meaningful len + meaningful bits
                    compressed.append(0)
                    compressed.extend(self._encode_bits(leading_zeros, 6))  # 6 bits for leading
                    meaningful_len = 64 - leading_zeros - trailing_zeros
                    compressed.extend(self._encode_bits(meaningful_len - 1, 6))  # 6 bits for len
                    meaningful_bits = xor >> trailing_zeros
                    compressed.extend(self._encode_bits(meaningful_bits, meaningful_len))

                prev_leading_zeros = leading_zeros
                prev_trailing_zeros = trailing_zeros

            prev_val_bits = curr_val_bits

        return bytes(compressed)

    def decompress_block(self, compressed_data):
        """解压数据块"""
        reader = BitReader(compressed_data)
        data_points = []

        # 读取第一个时间戳和值
        first_ts = reader.read_varint()
        first_val_bits = reader.read_varint()
        data_points.append({
            "timestamp": datetime.fromtimestamp(first_ts / 1000, tz=timezone.utc),
            "value": self._bits_to_float(first_val_bits),
        })

        prev_ts = first_ts
        prev_delta = 0
        prev_val_bits = first_val_bits
        prev_leading_zeros = None
        prev_trailing_zeros = None

        while reader.has_more():
            # --- 时间戳解码 ---
            marker = reader.read_bit()
            if marker == 0:
                # delta_of_delta = 0
                curr_ts = prev_ts + prev_delta
            else:
                delta_of_delta = reader.read_delta_of_delta()
                curr_delta = prev_delta + delta_of_delta
                curr_ts = prev_ts + curr_delta
                prev_delta = curr_delta

            # --- 浮点值解码 ---
            val_marker = reader.read_bit()
            if val_marker == 0:
                # XOR = 0，值相同
                curr_val_bits = prev_val_bits
            else:
                reuse_marker = reader.read_bit()
                if reuse_marker == 1:
                    # 复用前一个的 leading/trailing zeros
                    meaningful_bits = reader.read_bits(
                        64 - prev_leading_zeros - prev_trailing_zeros)
                    xor = (meaningful_bits << prev_trailing_zeros)
                else:
                    leading_zeros = reader.read_bits(6)
                    meaningful_len = reader.read_bits(6) + 1
                    meaningful_bits = reader.read_bits(meaningful_len)
                    trailing_zeros = 64 - leading_zeros - meaningful_len
                    xor = (meaningful_bits << trailing_zeros)
                    prev_leading_zeros = leading_zeros
                    prev_trailing_zeros = trailing_zeros

                curr_val_bits = prev_val_bits ^ xor

            data_points.append({
                "timestamp": datetime.fromtimestamp(curr_ts / 1000, tz=timezone.utc),
                "value": self._bits_to_float(curr_val_bits),
            })

            prev_ts = curr_ts
            prev_val_bits = curr_val_bits

        return data_points

    def _float_to_bits(self, value):
        """浮点数转 64 位整数表示"""
        import struct
        return struct.unpack("!Q", struct.pack("!d", value))[0]

    def _bits_to_float(self, bits):
        """64 位整数转浮点数"""
        import struct
        return struct.unpack("!d", struct.pack("!Q", bits))[0]

    def _count_leading_zeros(self, value):
        """计算前导零数量"""
        if value == 0:
            return 64
        count = 0
        while (value & (1 << 63)) == 0:
            count += 1
            value <<= 1
        return count

    def _count_trailing_zeros(self, value):
        """计算后缀零数量"""
        if value == 0:
            return 64
        count = 0
        while (value & 1) == 0:
            count += 1
            value >>= 1
        return count


class TimeseriesRetentionManager:
    """时序数据保留策略管理：原始→聚合→压缩→归档"""

    RETENTION_POLICIES = {
        "raw": {
            "retention_days": 30,
            "compression_after_days": 7,
            "target_compression_ratio": 0.95,  # 95% 压缩率
            "description": "原始数据保留 30 天，7 天后压缩",
        },
        "1min_agg": {
            "retention_days": 90,
            "compression_after_days": 30,
            "target_compression_ratio": 0.90,
            "description": "1 分钟聚合保留 90 天",
        },
        "1hour_agg": {
            "retention_days": 365,
            "compression_after_days": 90,
            "target_compression_ratio": 0.90,
            "description": "1 小时聚合保留 1 年",
        },
        "daily_agg": {
            "retention_days": None,  # 永久保留
            "compression_after_days": 365,
            "target_compression_ratio": 0.85,
            "description": "每日聚合永久保留",
        },
    }

    def setup_retention_policies(self, hypertable_name):
        """配置 TimescaleDB 保留与压缩策略"""

        # 1. 原始数据保留 30 天
        self.db.execute(f"""
            SELECT add_retention_policy('{hypertable_name}',
                INTERVAL '30 days')
        """)

        # 2. 7 天后自动压缩
        self.db.execute(f"""
            ALTER TABLE {hypertable_name} SET (
                timescaledb.compress,
                timescaledb.compress_segmentby = 'device_id',
                timescaledb.compress_orderby = 'timestamp DESC'
            )
        """)
        self.db.execute(f"""
            SELECT add_compression_policy('{hypertable_name}',
                INTERVAL '7 days')
        """)

        # 3. 连续聚合：1 分钟
        self.db.execute(f"""
            CREATE MATERIALIZED VIEW {hypertable_name}_1min
            WITH (timescaledb.continuous) AS
            SELECT time_bucket('1 minute', timestamp) as bucket,
                   device_id,
                   AVG(value) as avg_value,
                   MIN(value) as min_value,
                   MAX(value) as max_value,
                   STDDEV(value) as stddev_value,
                   PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY value) as p95_value,
                   COUNT(*) as sample_count
            FROM {hypertable_name}
            GROUP BY bucket, device_id
        """)

        # 1 分钟聚合保留 90 天
        self.db.execute(f"""
            SELECT add_retention_policy('{hypertable_name}_1min',
                INTERVAL '90 days')
        """)

        # 4. 连续聚合：1 小时
        self.db.execute(f"""
            CREATE MATERIALIZED VIEW {hypertable_name}_1hour
            WITH (timescaledb.continuous) AS
            SELECT time_bucket('1 hour', bucket) as bucket,
                   device_id,
                   AVG(avg_value) as avg_value,
                   MIN(min_value) as min_value,
                   MAX(max_value) as max_value,
                   AVG(stddev_value) as avg_stddev,
                   SUM(sample_count) as sample_count
            FROM {hypertable_name}_1min
            GROUP BY bucket, device_id
        """)

        # 1 小时聚合保留 1 年
        self.db.execute(f"""
            SELECT add_retention_policy('{hypertable_name}_1hour',
                INTERVAL '365 days')
        """)

        # 5. 连续聚合：每日
        self.db.execute(f"""
            CREATE MATERIALIZED VIEW {hypertable_name}_daily
            WITH (timescaledb.continuous) AS
            SELECT time_bucket('1 day', bucket) as bucket,
                   device_id,
                   AVG(avg_value) as avg_value,
                   MIN(min_value) as min_value,
                   MAX(max_value) as max_value,
                   SUM(sample_count) as sample_count
            FROM {hypertable_name}_1hour
            GROUP BY bucket, device_id
        """)

        # 每日聚合永久保留，365 天后压缩
        self.db.execute(f"""
            SELECT add_compression_policy('{hypertable_name}_daily',
                INTERVAL '365 days')
        """)

    def verify_compression_integrity(self, hypertable_name):
        """验证压缩数据完整性（解压后与原始数据对比）"""
        # 获取最近压缩的数据块
        chunks = self.db.query(f"""
            SELECT chunk_name, before_compression_total_bytes,
                   after_compression_total_bytes,
                   compression_status
            FROM timescaledb_information.chunks
            WHERE hypertable_name = '{hypertable_name}'
              AND compression_status = 'Compressed'
            ORDER BY range_start DESC LIMIT 10
        """)

        results = []
        for chunk in chunks:
            # 采样验证：解压部分数据与预期对比
            sample_data = self.db.query(f"""
                SELECT device_id, timestamp, value
                FROM {chunk['chunk_name']}
                ORDER BY timestamp LIMIT 100
            """)

            # 验证数据可正常读取（TimescaleDB 自动透明解压）
            if len(sample_data) > 0:
                results.append({
                    "chunk": chunk["chunk_name"],
                    "status": "ok",
                    "sample_count": len(sample_data),
                    "compression_ratio": round(
                        1 - chunk["after_compression_total_bytes"] /
                        max(chunk["before_compression_total_bytes"], 1), 3),
                })
            else:
                results.append({
                    "chunk": chunk["chunk_name"],
                    "status": "error",
                    "reason": "解压后无数据",
                })

        return results

    def get_storage_summary(self):
        """获取存储概览"""
        summary = {}
        for policy_name, policy in self.RETENTION_POLICIES.items():
            table_name = f"device_telemetry_{policy_name}" if policy_name != "raw" else "device_telemetry"

            try:
                stats = self.db.query_one(f"""
                    SELECT pg_size_pretty(pg_total_relation_size('{table_name}')) as total_size,
                           (SELECT count(*) FROM {table_name}) as row_count
                """)
                summary[policy_name] = {
                    "size": stats["total_size"],
                    "rows": stats["row_count"],
                    "retention_days": policy["retention_days"] or "永久",
                    "compression_after_days": policy["compression_after_days"],
                }
            except Exception:
                summary[policy_name] = {"error": "表不存在或查询失败"}

        return summary
```

### 场景：设备证书轮换失败导致批量断连

```
触发：证书轮换服务推送新证书后，旧证书立即被 PKI 吊销
      → 设备固件 bug：收到新证书后未能正确加载 → 仍使用旧证书
      → MQTT 认证失败 → 设备批量断连（影响 500+ 台设备）
检测：
  1. 设备离线率从 2% 飙升到 15% → 异常
  2. MQTT broker 认证失败日志激增 → 证书问题
  3. 证书轮换操作记录 → 时间吻合
  4. 受影响设备集中在同一固件版本 → 固件 bug
处理：
  1. 【紧急】恢复旧证书有效性（PKI 取消吊销，24 小时宽限期）
  2. 【止血】暂停所有证书轮换操作
  3. 【定位】确认设备固件 bug：证书加载后未重置 TLS 连接
  4. 【修复】推送固件热补丁修复证书加载逻辑
  5. 【恢复】修复后逐步恢复证书轮换（每批 50 台，间隔 5 分钟）
  6. 【验证】确认所有设备重新上线且使用新证书
预防：双证书过渡期（新旧证书重叠 3 天）+ 推送后等待设备确认加载
      + 渐进式轮换（小批量验证后全量）+ 轮换前固件兼容性检查
      + 证书轮换操作窗口限制（非高峰时段）
```

### 场景：压缩策略损坏历史数据块

```
触发：TimescaleDB 压缩策略在压缩某个数据块时遭遇磁盘空间不足
      → 压缩过程被中断 → 数据块处于半压缩状态
      → 后续查询该数据块时返回错误或截断数据 → 历史趋势图异常
检测：
  1. 查询返回 "compressed chunk decompression error" → 压缩损坏
  2. 仪表盘历史趋势图出现数据缺口 → 数据不可读
  3. TimescaleDB 日志记录压缩失败 → 但数据块已被标记为"已压缩"
  4. 压缩前后数据校验和不匹配 → 数据完整性受损
处理：
  1. 【紧急】禁用自动压缩策略（防止更多数据块损坏）
     SELECT remove_compression_policy('device_telemetry');
  2. 【定位】查询所有处于异常状态的压缩块
     SELECT * FROM timescaledb_information.chunks
     WHERE compression_status = 'Compressed'
       AND after_compression_total_bytes = 0;
  3. 【恢复】从最近的备份恢复损坏的数据块
     - TimescaleDB 快照备份（每日全量 + WAL 增量）
     - 恢复受影响时间段的原始数据
  4. 【验证】恢复后验证数据完整性
     - 对比聚合查询结果与预期
     - 采样验证原始数据点
  5. 【改进】压缩策略增加前置检查：
     - 压缩前验证磁盘剩余空间 > 数据块大小的 2 倍
     - 压缩后校验和验证
     - 灰度压缩：先压缩最老的数据块，逐步推进
  6. 【监控】增加压缩健康度监控指标：
     - 压缩成功率、压缩耗时、解压错误率
预防：压缩前磁盘空间检查 + 压缩后校验和验证
      + 灰度压缩（从最老块开始）+ 每日备份 + 解压错误率监控

## IoT 数据查询优化完整实现

```python
class IoTQueryOptimizer:
    """IoT 数据查询优化：分区裁剪 + 预聚合 + 查询缓存"""

    def optimize_query(self, query_params):
        """优化 IoT 时间序列查询"""
        # 1. 分区裁剪（只扫描相关时间分区）
        start_time = query_params.get("start_time", now() - timedelta(hours=1))
        end_time = query_params.get("end_time", now())

        # 2. 选择最佳聚合层级
        time_range_seconds = (end_time - start_time).total_seconds()
        if time_range_seconds <= 3600:  # ≤1 小时 → 查原始数据
            table = "device_measurements"
        elif time_range_seconds <= 86400 * 90:  # ≤90 天 → 查 1 分钟聚合
            table = "device_measurements_1min"
        else:  # >90 天 → 查小时聚合
            table = "device_measurements_1hour"

        # 3. 构建优化查询
        sql = f"""
            SELECT device_id, time_bucket('{self._bucket_size(time_range_seconds)}', timestamp) as bucket,
                   AVG(value) as avg_value,
                   MIN(value) as min_value,
                   MAX(value) as max_value,
                   COUNT(*) as sample_count
            FROM {table}
            WHERE device_id IN ({','.join(['%s'] * len(query_params['device_ids']))})
            AND timestamp BETWEEN %s AND %s
            GROUP BY device_id, bucket
            ORDER BY bucket ASC
        """

        # 4. 查询缓存检查
        cache_key = f"iot_query:{hashlib.md5(json.dumps(query_params, sort_keys=True).encode()).hexdigest()}"
        cached = self.redis.get(cache_key)
        if cached:
            return json.loads(cached)

        # 5. 执行查询
        result = self.db.query(sql,
            *query_params["device_ids"], start_time, end_time)

        # 6. 缓存结果（TTL 基于时间范围）
        ttl = min(3600, time_range_seconds / 10)
        self.redis.setex(cache_key, int(ttl), json.dumps(result))

        return result

    def _bucket_size(self, seconds):
        """自动选择时间桶大小"""
        if seconds <= 3600:
            return "1 minute"
        elif seconds <= 86400:
            return "5 minutes"
        elif seconds <= 86400 * 30:
            return "1 hour"
        else:
            return "1 day"
```

## 异常场景补充

### 场景：查询缓存返回过期数据

```
触发：缓存 TTL 过长 → 设备已离线但缓存仍返回旧数据 → 误导运维
检测：
  1. 缓存数据时间戳 > 30 分钟 → 可能过期
  2. 设备已离线但查询返回正常值 → 缓存过期
处理：
  1. 缩短缓存 TTL（设备在线: 5 分钟, 离线: 不缓存）
  2. 缓存标记数据新鲜度
  3. 关键查询绕过缓存
预防：动态 TTL + 数据新鲜度标记 + 关键查询不缓存
```

### 场景：分区裁剪失效导致全表扫描

```
触发：时间分区索引损坏 → 查询扫描所有分区 → 响应超时
检测：
  1. 查询执行时间 > 30 秒 → 可能全表扫描
  2. EXPLAIN 显示 Seq Scan → 分区裁剪失效
处理：
  1. 重建时间分区索引
  2. 临时使用预聚合表替代
  3. 修复后验证分区裁剪生效
预防：定期 EXPLAIN 验证 + 索引健康检查 + 预聚合兜底
```

## IoT 设备异常检测与根因分析完整实现

```python
class IoTAnomalyDetectionService:
    """IoT 异常检测：统计阈值 + 趋势检测 + 根因分析"""

    ANOMALY_TYPES = {
        "point": "点异常（单值偏离）",
        "contextual": "上下文异常（特定条件下偏离）",
        "collective": "集体异常（多个设备同时异常）",
    }

    def detect_anomalies(self, device_id, metric_name, window_minutes=60):
        """检测设备指标异常"""
        # 1. 获取历史基线
        baseline = self._get_metric_baseline(device_id, metric_name)

        if not baseline:
            return {"status": "insufficient_data"}

        mean = baseline["mean"]
        std = baseline["std"]
        seasonal_pattern = baseline.get("seasonal_pattern")

        # 2. 获取当前窗口数据
        current_data = self.timeseries.query(
            device_id=device_id, metric=metric_name,
            start=now()-timedelta(minutes=window_minutes), end=now())

        anomalies = []

        for point in current_data:
            value = Decimal(str(point["value"]))
            timestamp = point["timestamp"]

            # 2a. 点异常检测（3-sigma 规则）
            if std > 0:
                z_score = abs(float(value) - float(mean)) / float(std)

                if z_score > 3:
                    # 考虑季节性调整
                    expected = mean
                    if seasonal_pattern:
                        hour = timestamp.hour
                        expected = Decimal(str(seasonal_pattern.get(hour, float(mean))))

                    adjusted_z = abs(float(value) - float(expected)) / float(std)

                    if adjusted_z > 3:
                        anomalies.append({
                            "type": "point",
                            "timestamp": timestamp.isoformat(),
                            "value": str(value),
                            "expected": str(expected),
                            "z_score": round(adjusted_z, 2),
                            "severity": "critical" if adjusted_z > 5 else "warning"
                        })

        # 3. 集体异常检测（同类型设备是否同时异常）
        if anomalies:
            device = self.db.get_device(device_id)
            device_type = device.get("type")

            similar_devices = self.db.query(
                "SELECT device_id FROM devices WHERE type = %s "
                "AND location_id = %s AND status = 'active'",
                device_type, device.get("location_id"))

            collective_count = 0
            for sd in similar_devices[:10]:
                sd_data = self.timeseries.query(
                    device_id=sd["device_id"], metric=metric_name,
                    start=now()-timedelta(minutes=5), end=now())

                for point in sd_data:
                    if std > 0 and abs(float(point["value"]) - float(mean)) / float(std) > 3:
                        collective_count += 1
                        break

            if collective_count > len(similar_devices) * 0.3:
                anomalies.append({
                    "type": "collective",
                    "affected_devices": collective_count,
                    "total_devices": len(similar_devices),
                    "severity": "critical",
                    "message": f"{collective_count}/{len(similar_devices)} 同类型设备同时异常"
                })

        # 4. 记录异常
        for anomaly in anomalies:
            self.db.insert("iot_anomalies", {
                "anomaly_id": str(uuid4()),
                "device_id": device_id,
                "metric_name": metric_name,
                "anomaly_type": anomaly["type"],
                "details": json.dumps(anomaly),
                "severity": anomaly.get("severity", "warning"),
                "detected_at": now()
            })

        return {"device_id": device_id, "metric": metric_name,
                "anomaly_count": len(anomalies), "anomalies": anomalies}

    def root_cause_analysis(self, device_id, anomaly_id):
        """根因分析"""
        anomaly = self.db.get_anomaly(anomaly_id)

        # 1. 关联事件分析
        related_events = self.db.query(
            "SELECT * FROM iot_events "
            "WHERE device_id = %s "
            "AND timestamp BETWEEN %s AND %s "
            "ORDER BY timestamp",
            device_id,
            anomaly["detected_at"] - timedelta(minutes=30),
            anomaly["detected_at"])

        # 2. 关联指标分析（其他指标是否也异常）
        device = self.db.get_device(device_id)
        device_metrics = self.db.query(
            "SELECT DISTINCT metric_name FROM device_metrics "
            "WHERE device_id = %s", device_id)

        correlated_anomalies = []
        for m in device_metrics:
            if m["metric_name"] == anomaly.get("metric_name"):
                continue
            other = self.detect_anomalies(device_id, m["metric_name"],
                                          window_minutes=30)
            if other["anomaly_count"] > 0:
                correlated_anomalies.append({
                    "metric": m["metric_name"],
                    "anomaly_count": other["anomaly_count"]
                })

        # 3. 推断根因
        causes = self._infer_causes(device, anomaly, related_events, correlated_anomalies)

        return {
            "device_id": device_id,
            "anomaly_id": anomaly_id,
            "related_events": len(related_events),
            "correlated_metrics": correlated_anomalies,
            "probable_causes": causes
        }

    def _infer_causes(self, device, anomaly, events, correlated):
        """推断可能原因"""
        causes = []

        # 硬件故障模式
        if any(e["type"] == "error" for e in events):
            causes.append({
                "cause": "hardware_failure",
                "confidence": 0.7,
                "evidence": "异常前有错误事件"
            })

        # 环境变化
        env_metrics = [c for c in correlated if c["metric"] in ["temperature", "humidity"]]
        if env_metrics:
            causes.append({
                "cause": "environmental_change",
                "confidence": 0.6,
                "evidence": f"环境指标异常: {[c['metric'] for c in env_metrics]}"
            })

        # 网络问题
        if any(c["metric"] == "network_latency" for c in correlated):
            causes.append({
                "cause": "network_issue",
                "confidence": 0.5,
                "evidence": "网络延迟指标同时异常"
            })

        # 电源问题
        if any(c["metric"] == "voltage" for c in correlated):
            causes.append({
                "cause": "power_issue",
                "confidence": 0.8,
                "evidence": "电压指标异常"
            })

        causes.sort(key=lambda c: c["confidence"], reverse=True)
        return causes

    def _get_metric_baseline(self, device_id, metric_name):
        """获取指标基线"""
        # 最近 7 天数据计算统计基线
        data = self.timeseries.query(
            device_id=device_id, metric=metric_name,
            start=now()-timedelta(days=7), end=now(),
            aggregation="1h")  # 1 小时粒度

        if len(data) < 24:
            return None

        values = [float(d["value"]) for d in data]
        mean = statistics.mean(values)
        std = statistics.stdev(values) if len(values) > 1 else 0

        # 计算按小时的季节性模式
        hourly_values = {}
        for d in data:
            hour = d["timestamp"].hour
            hourly_values.setdefault(hour, []).append(float(d["value"]))

        seasonal_pattern = {h: statistics.mean(vs) for h, vs in hourly_values.items()}

        return {"mean": Decimal(str(round(mean, 4))),
                "std": Decimal(str(round(std, 4))),
                "seasonal_pattern": seasonal_pattern}
```

## 异常场景补充

### 场景：异常检测误报导致设备误停

```
触发：设备温度正常波动 → 被判定为异常 → 自动停机 → 生产损失
检测：
  1. 异常检测后设备停机但实际无故障 → 误报
  2. 误报率 > 10% → 阈值过严
处理：
  1. 异常检测只告警不自动停机（人工确认后停机）
  2. 提高检测阈值（3-sigma → 4-sigma）
  3. 加入季节性调整
预防：告警不自动停机 + 阈值调整 + 季节性
```

### 场景：基线数据被污染

```
触发：设备故障 3 天 → 故障数据纳入基线 → 基线偏高 → 正常值反被判定为异常
检测：
  1. 基线均值突然偏移 → 可能被污染
  2. 正常数据被频繁判定异常 → 基线不准
处理：
  1. 基线计算排除异常值（IQR 过滤）
  2. 设备故障期间数据不纳入基线
  3. 定期重新计算基线
预防：异常值过滤 + 故障期排除 + 定期重算
```

## IoT 设备固件 OTA 升级完整实现

```python
class OTAUpgradeService:
    """OTA 升级：版本管理 → 灰度推送 → 升级执行 → 回滚"""

    UPGRADE_STRATEGIES = {
        "full_push": "全量推送",
        "canary": "灰度推送（先 5% → 25% → 50% → 100%）",
        "staged": "分批推送（按设备组分批）",
        "scheduled": "定时推送（指定时间窗口）",
    }

    UPGRADE_STATUSES = {
        "pending": "待升级",
        "downloading": "下载中",
        "verifying": "校验中",
        "installing": "安装中",
        "rebooting": "重启中",
        "completed": "已完成",
        "failed": "失败",
        "rolled_back": "已回滚",
    }

    def create_firmware_version(self, version_data):
        """创建固件版本"""
        version_id = str(uuid4())

        # 1. 验证固件文件
        firmware_url = version_data["firmware_url"]
        checksum = version_data.get("checksum")

        if not self.object_storage.exists(firmware_url):
            raise ValueError(f"固件文件不存在: {firmware_url}")

        # 2. 验证 checksum
        if checksum:
            actual = self._calculate_firmware_checksum(firmware_url)
            if actual != checksum:
                raise ValueError(f"固件校验失败: 期望 {checksum}, 实际 {actual}")

        # 3. 注册版本
        self.db.insert("firmware_versions", {
            "version_id": version_id,
            "device_type": version_data["device_type"],
            "version_number": version_data["version_number"],
            "firmware_url": firmware_url,
            "checksum": checksum or self._calculate_firmware_checksum(firmware_url),
            "file_size_bytes": self.object_storage.get_size(firmware_url),
            "release_notes": version_data.get("release_notes", ""),
            "is_mandatory": version_data.get("is_mandatory", False),
            "min_current_version": version_data.get("min_current_version"),
            "status": "active",
            "created_at": now()
        })

        return {"version_id": version_id,
                "version_number": version_data["version_number"]}

    def push_upgrade(self, device_type, target_version_id, strategy="canary"):
        """推送升级"""
        target_version = self.db.get_firmware_version(target_version_id)

        # 1. 获取目标设备列表
        devices = self.db.query(
            "SELECT * FROM devices "
            "WHERE type = %s AND status = 'online' "
            "AND firmware_version != %s",
            device_type, target_version["version_number"])

        if not devices:
            return {"status": "no_devices_to_upgrade"}

        # 2. 创建升级任务
        task_id = str(uuid4())
        self.db.insert("ota_upgrade_tasks", {
            "task_id": task_id,
            "device_type": device_type,
            "target_version_id": target_version_id,
            "target_version_number": target_version["version_number"],
            "strategy": strategy,
            "total_devices": len(devices),
            "status": "in_progress",
            "created_at": now()
        })

        # 3. 按策略分配设备
        if strategy == "canary":
            self._push_canary(task_id, devices, target_version)
        elif strategy == "staged":
            self._push_staged(task_id, devices, target_version)
        elif strategy == "full_push":
            self._push_full(task_id, devices, target_version)
        elif strategy == "scheduled":
            self._push_scheduled(task_id, devices, target_version,
                version_data.get("scheduled_time"))

        return {"task_id": task_id, "total_devices": len(devices),
                "strategy": strategy}

    def _push_canary(self, task_id, devices, target_version):
        """灰度推送"""
        total = len(devices)

        # 第一批：5%
        batch1_size = max(1, int(total * 0.05))
        batch1 = devices[:batch1_size]

        for device in batch1:
            self._send_upgrade_command(device["id"], target_version, task_id)

        # 记录后续批次计划
        batches = [
            {"pct": 5, "devices": [d["id"] for d in devices[:batch1_size]]},
            {"pct": 25, "devices": [d["id"] for d in devices[batch1_size:int(total*0.25)]]},
            {"pct": 50, "devices": [d["id"] for d in devices[int(total*0.25):int(total*0.5)]]},
            {"pct": 100, "devices": [d["id"] for d in devices[int(total*0.5):]]},
        ]

        self.db.update("ota_upgrade_tasks",
            {"batches": json.dumps(batches), "current_batch": 0},
            {"task_id": task_id})

    def _push_full(self, task_id, devices, target_version):
        """全量推送"""
        for device in devices:
            self._send_upgrade_command(device["id"], target_version, task_id)

    def _push_staged(self, task_id, devices, target_version):
        """分批推送（每批 100 台）"""
        batch_size = 100
        batches = []

        for i in range(0, len(devices), batch_size):
            batch = devices[i:i+batch_size]
            batches.append({
                "batch_number": i // batch_size,
                "device_ids": [d["id"] for d in batch]
            })

        # 推送第一批
        for device in batches[0]["device_ids"]:
            self._send_upgrade_command(device, target_version, task_id)

        self.db.update("ota_upgrade_tasks",
            {"batches": json.dumps(batches), "current_batch": 0},
            {"task_id": task_id})

    def _push_scheduled(self, task_id, devices, target_version, scheduled_time):
        """定时推送"""
        self.task_queue.schedule(
            self._push_full, task_id, devices, target_version,
            run_at=scheduled_time)

    def _send_upgrade_command(self, device_id, target_version, task_id):
        """发送升级指令"""
        self.db.insert("ota_device_upgrades", {
            "upgrade_id": str(uuid4()),
            "task_id": task_id,
            "device_id": device_id,
            "target_version": target_version["version_number"],
            "firmware_url": target_version["firmware_url"],
            "checksum": target_version["checksum"],
            "status": "pending",
            "created_at": now()
        })

        self.mqtt.publish(f"device/{device_id}/ota", json.dumps({
            "action": "upgrade",
            "version": target_version["version_number"],
            "firmware_url": target_version["firmware_url"],
            "checksum": target_version["checksum"],
            "is_mandatory": target_version["is_mandatory"]
        }))

    def handle_upgrade_result(self, device_id, result):
        """处理升级结果"""
        upgrade = self.db.query_one(
            "SELECT * FROM ota_device_upgrades "
            "WHERE device_id = %s AND status IN ('pending', 'downloading', "
            "'verifying', 'installing', 'rebooting') "
            "ORDER BY created_at DESC LIMIT 1",
            device_id)

        if not upgrade:
            return {"status": "no_active_upgrade"}

        # 更新状态
        self.db.update("ota_device_upgrades",
            {"status": result["status"],
             "error_message": result.get("error"),
             "completed_at": now() if result["status"] == "completed" else None},
            {"upgrade_id": upgrade["upgrade_id"]})

        if result["status"] == "completed":
            # 更新设备固件版本
            self.db.update("devices",
                {"firmware_version": upgrade["target_version"]},
                {"id": device_id})

            # 检查是否需要推进灰度
            self._check_canary_progress(upgrade["task_id"])

        elif result["status"] == "failed":
            # 升级失败 → 记录并检查失败率
            self._check_failure_rate(upgrade["task_id"])

        return {"device_id": device_id, "status": result["status"]}

    def _check_canary_progress(self, task_id):
        """检查灰度进度"""
        task = self.db.get_upgrade_task(task_id)

        if task["strategy"] != "canary":
            return

        # 统计当前批次成功率
        completed = self.db.count("ota_device_upgrades",
            task_id=task_id, status="completed")
        failed = self.db.count("ota_device_upgrades",
            task_id=task_id, status="failed")

        total = completed + failed
        if total == 0:
            return

        success_rate = completed / total

        # 成功率 > 95% → 推进下一批次
        if success_rate > 0.95 and total >= 5:
            batches = json.loads(task["batches"])
            current = task.get("current_batch", 0)

            if current + 1 < len(batches):
                next_batch = batches[current + 1]
                target_version = self.db.get_firmware_version(
                    task["target_version_id"])

                for device_id in next_batch["devices"]:
                    self._send_upgrade_command(device_id, target_version, task_id)

                self.db.update("ota_upgrade_tasks",
                    {"current_batch": current + 1},
                    {"task_id": task_id})

        # 成功率 < 80% → 暂停灰度
        elif success_rate < 0.80:
            self.alert(f"OTA 升级成功率过低: {success_rate:.0%}，已暂停灰度")
            self.db.update("ota_upgrade_tasks",
                {"status": "paused"},
                {"task_id": task_id})

    def _check_failure_rate(self, task_id):
        """检查失败率"""
        completed = self.db.count("ota_device_upgrades",
            task_id=task_id, status="completed")
        failed = self.db.count("ota_device_upgrades",
            task_id=task_id, status="failed")

        total = completed + failed
        if total < 5:
            return

        failure_rate = failed / total

        if failure_rate > 0.2:
            self.alert(f"OTA 升级失败率 {failure_rate:.0%} 超过阈值，已暂停")
            self.db.update("ota_upgrade_tasks",
                {"status": "paused"},
                {"task_id": task_id})

    def rollback_upgrade(self, device_id, target_version):
        """回滚设备固件"""
        device = self.db.get_device(device_id)
        previous_version = device.get("previous_firmware_version")

        if not previous_version:
            return {"status": "no_previous_version"}

        # 发送回滚指令
        self.mqtt.publish(f"device/{device_id}/ota", json.dumps({
            "action": "rollback",
            "version": previous_version,
        }))

        return {"device_id": device_id, "rolling_back_to": previous_version}

    def _calculate_firmware_checksum(self, firmware_url):
        """计算固件校验和"""
        import hashlib
        data = self.object_storage.read(firmware_url)
        return hashlib.sha256(data).hexdigest()
```

## 异常场景补充

### 场景：OTA 升级导致设备变砖

```
触发：固件有严重 bug → 升级后设备无法启动 → 无法再接收指令 → 变砖
检测：
  1. 升级后设备离线超时 → 可能变砖
  2. 大量设备升级后同时离线 → 固件问题
处理：
  1. 立即暂停升级任务
  2. 设备双分区（A/B 系统）→ 自动回滚到旧分区
  3. 无双分区 → 需人工恢复（串口刷机）
预防：双分区 + 升级前验证 + 暂停机制
```

### 场景：大量设备同时下载固件导致 CDN 过载

```
触发：10 万台设备同时收到升级指令 → 同时下载固件 → CDN 带宽打满 → 下载失败
检测：
  1. CDN 带宽使用率 > 90% → 过载
  2. 固件下载失败率 > 10% → 下载问题
处理：
  1. 分批推送（每批 1000 台）
  2. 设备随机延迟下载（0-30 分钟内随机）
  3. P2P 分发（设备间共享固件包）
预防：分批推送 + 随机延迟 + P2P 分发
```
