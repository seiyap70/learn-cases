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
    min(min_pressure) AS min_pressure
FROM device_metrics_1min
GROUP BY bucket, device_id
WITH NO DATA;
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

**Flink 中的实时告警检测：**

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