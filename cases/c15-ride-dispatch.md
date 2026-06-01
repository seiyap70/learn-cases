# C15: 网约车的派单系统

## 业务场景

某网约车平台，需要实现实时派单系统：乘客发单 → 系统匹配最优司机 → 司机接单 → 服务执行。派单系统是网约车平台的核心——派单质量直接决定了乘客等待时间、司机收入和平台效率。

**已知数据：**
- 同时在线司机：30 万（高峰期），10 万（平峰期）
- 同时等待乘客：5 万（高峰期）
- 派单延迟要求：< 3 秒（从乘客发单到司机收到派单通知）
- 司机位置更新频率：每 5 秒上报一次
- 服务区域：全国 400+ 城市

**派单质量指标：**
- 乘客平均等待时间：< 5 分钟
- 司机接单率：> 85%（派单成功率）
- 司机拒单率：< 10%
- 空驶率：< 30%（司机无客行驶比例）

**为什么这比"找最近司机"复杂得多？**

表面上看，派单就是"找最近的空闲司机"。但实际考虑：

1. **最近的不一定最优** —— 距离 500 米的司机正在反方向行驶，掉头需要 3 分钟；距离 1.5 公里的司机正在同方向行驶，2 分钟就能到
2. **公平性问题** —— 如果总是派给最近的司机，远离热点的司机永远接不到单
3. **全局最优 vs 局部最优** —— 把最近的司机派给 A 乘客，可能意味着 B 乘客（更紧急）要等更久
4. **供需不平衡** —— 高峰期需求远大于供给，如何分配有限的司机资源

## 核心挑战

### 挑战 1：实时空间匹配

30 万在线司机，每 5 秒更新一次位置。乘客发单时，需要在毫秒级内找到 3km 范围内的空闲司机，并计算实际到达时间（考虑道路网络，不是直线距离）。

### 挑战 2：多目标优化

派单不是单一目标的优化问题，而是多目标权衡：
- 最小化乘客等待时间
- 最大化司机收入（减少空驶）
- 保证公平性（司机接单机会均衡）
- 提高拼车效率（如果支持拼车）

这些目标之间有根本性矛盾：公平性往往意味着牺牲效率。

### 挑战 3：供需严重不平衡

早晚高峰时，需求是供给的 3-5 倍。不可能让所有乘客都打到车，必须做资源分配——哪些乘客优先？如何定价调节需求？

### 挑战 4：信息不对称

系统知道司机的实时位置和行驶方向，但不知道司机的主观意愿——司机可能不想接长途单、不想去某个区域、不想接低评分乘客。

## 设计约束

- 司机位置每 5 秒更新一次（存在 5 秒的位置延迟）
- 派单决策需要考虑：距离、方向夹角、司机评分、等待时长、历史接单率
- 拒单率需控制 < 10%（派单准确性指标）
- 3 秒内完成匹配+通知（包括网络传输时间）

## 请先独立思考（限时 45 分钟）

1. 设计一个评分函数，给候选司机打分。明确每个因子的权重和计算方式，然后分析：权重如何影响"效率 vs 公平"的权衡？
2. 派单是"抢单"还是"系统派单"？从司机体验、匹配质量、公平性三个维度对比。
3. 如果某区域有 100 个等待乘客但只有 20 个空闲司机，应该怎么分配？先到先得？出价最高？距离最近？
4. 设计一个动态加价算法，在供需不平衡时自动调节价格。注意：加价不能无上限（舆论风险），也不能不起作用（调节失败）。

---

## 设计解析

### 派单策略选择：为什么选系统派单而非抢单

| 维度 | 抢单 | 系统派单 |
|------|------|---------|
| 匹配质量 | 差——最近的司机不一定手最快，可能 500 米外的司机网速慢抢不到 | 好——算法选最优 |
| 公平性 | 差——好单被手快的人抢，差单无人接 | 可控——算法可加入公平因子 |
| 司机体验 | 焦虑——需要不断盯着手机抢单，可能分心驾驶 | 安心——系统分配，可以专注驾驶 |
| 乘客体验 | 不稳定——等单时间取决于附近有没有"手快"的司机 | 可预期——等单时间由算法优化 |
| 系统复杂度 | 低——只需广播订单 | 高——需要维护司机状态和评分系统 |

**结论：系统派单。** 几乎所有主流网约车平台都已从抢单转向派单。

### 整体架构

```
乘客发单 → 派单服务
              │
              ├─ 1. 空间查询：Redis GEO 找 3km 内空闲司机
              │
              ├─ 2. 方向过滤：排除方向完全相反的司机
              │
              ├─ 3. 状态过滤：排除不合规司机（拒单率过高/休息中等）
              │
              ├─ 4. 综合打分：多因子评分
              │
              ├─ 5. 选择 Top 1 司机
              │
              ├─ 6. 发送派单通知
              │
              └─ 7. 司机接单/拒单 → 接单成功 or 重试 Top 2
```

### 数据库设计

派单系统涉及的核心数据表需要支持高并发写入（司机位置上报）和快速查询（订单匹配），以下是完整的表结构设计。

**订单表（orders）：**

```sql
CREATE TABLE orders (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_no        VARCHAR(32) NOT NULL UNIQUE COMMENT '订单号，如 DD20240601150000001',
    passenger_id    BIGINT NOT NULL COMMENT '乘客用户ID',
    city_id         INT NOT NULL COMMENT '城市ID',
    status          TINYINT NOT NULL DEFAULT 0 COMMENT '0-待派单 1-已派单 2-司机已接单 3-行程中 4-已完成 5-乘客取消 6-司机取消 7-超时取消',
    pickup_lng      DECIMAL(10,6) NOT NULL COMMENT '上车点经度',
    pickup_lat      DECIMAL(10,6) NOT NULL COMMENT '上车点纬度',
    pickup_address  VARCHAR(200) NOT NULL COMMENT '上车点地址文本',
    dest_lng        DECIMAL(10,6) NOT NULL COMMENT '目的地经度',
    dest_lat        DECIMAL(10,6) NOT NULL COMMENT '目的地纬度',
    dest_address    VARCHAR(200) NOT NULL COMMENT '目的地地址文本',
    estimated_distance DECIMAL(8,2) COMMENT '预估距离(米)',
    estimated_duration INT COMMENT '预估时长(秒)',
    estimated_fare  DECIMAL(10,2) COMMENT '预估费用',
    actual_fare     DECIMAL(10,2) COMMENT '实际费用',
    surge_multiplier DECIMAL(3,1) DEFAULT 1.0 COMMENT '加价倍数',
    car_type        TINYINT DEFAULT 1 COMMENT '1-快车 2-专车 3-豪华车',
    is_carpool      TINYINT DEFAULT 0 COMMENT '0-独享 1-拼车',
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    dispatched_at   DATETIME COMMENT '派单时间',
    accepted_at     DATETIME COMMENT '接单时间',
    pickup_at       DATETIME COMMENT '上车时间',
    completed_at    DATETIME COMMENT '完成时间',
    cancel_reason   VARCHAR(200) COMMENT '取消原因',
    INDEX idx_passenger (passenger_id, created_at),
    INDEX idx_status_city (status, city_id, created_at),
    INDEX idx_dispatch_time (dispatched_at, city_id)
) ENGINE=InnoDB COMMENT='订单表';
```

**司机表（drivers）：**

```sql
CREATE TABLE drivers (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    driver_no       VARCHAR(32) NOT NULL UNIQUE COMMENT '司机编号',
    user_id         BIGINT NOT NULL UNIQUE COMMENT '关联用户ID',
    city_id         INT NOT NULL COMMENT '所属城市',
    car_type        TINYINT NOT NULL COMMENT '1-快车 2-专车 3-豪华车',
    rating          DECIMAL(2,1) DEFAULT 4.5 COMMENT '当前评分(1-5)',
    total_trips     INT DEFAULT 0 COMMENT '累计完成订单数',
    reject_rate     DECIMAL(4,3) DEFAULT 0.0 COMMENT '近7天拒单率',
    status          TINYINT DEFAULT 0 COMMENT '0-离线 1-空闲 2-服务中 3-休息中',
    current_lng     DECIMAL(10,6) COMMENT '当前经度(最后上报)',
    current_lat     DECIMAL(10,6) COMMENT '当前纬度(最后上报)',
    bearing         SMALLINT DEFAULT 0 COMMENT '行驶方向角(0-360)',
    speed           DECIMAL(5,1) DEFAULT 0 COMMENT '当前速度(km/h)',
    last_heartbeat  DATETIME COMMENT '最后心跳时间',
    last_trip_at    DATETIME COMMENT '最后一单完成时间',
    online_hours_today DECIMAL(4,1) DEFAULT 0 COMMENT '今日在线时长(小时)',
    trips_today     INT DEFAULT 0 COMMENT '今日完成单数',
    dest_lat        DECIMAL(10,6) COMMENT '顺路单-目的地纬度',
    dest_lng        DECIMAL(10,6) COMMENT '顺路单-目的地经度',
    has_dest        TINYINT DEFAULT 0 COMMENT '0-无顺路设置 1-已设置顺路目的地',
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_city_status (city_id, status, last_heartbeat),
    INDEX idx_rating (rating, total_trips),
    INDEX idx_online (status, last_heartbeat)
) ENGINE=InnoDB COMMENT='司机表';
```

**派单记录表（dispatch_records）：**

```sql
CREATE TABLE dispatch_records (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id        BIGINT NOT NULL COMMENT '订单ID',
    driver_id       BIGINT NOT NULL COMMENT '司机ID',
    dispatch_type   TINYINT NOT NULL COMMENT '1-首次派单 2-拒单重派 3-超时重派 4-取消重派',
    result          TINYINT NOT NULL COMMENT '1-接单 2-拒单 3-超时 4-取消',
    driver_distance INT COMMENT '派单时司机距上车点距离(米)',
    driver_bearing  SMALLINT COMMENT '派单时司机方向角',
    score           DECIMAL(5,3) COMMENT '综合评分',
    reject_reason   VARCHAR(100) COMMENT '拒单原因',
    response_time_ms INT COMMENT '司机响应时间(毫秒)',
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_order (order_id, created_at),
    INDEX idx_driver (driver_id, created_at),
    INDEX idx_result_time (result, created_at)
) ENGINE=InnoDB COMMENT='派单记录表';
```

**司机位置快照表（driver_locations）：**

```sql
CREATE TABLE driver_locations (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    driver_id       BIGINT NOT NULL COMMENT '司机ID',
    city_id         INT NOT NULL COMMENT '城市ID',
    lng             DECIMAL(10,6) NOT NULL COMMENT '经度',
    lat             DECIMAL(10,6) NOT NULL COMMENT '纬度',
    bearing         SMALLINT DEFAULT 0 COMMENT '方向角',
    speed           DECIMAL(5,1) DEFAULT 0 COMMENT '速度(km/h)',
    accuracy        DECIMAL(5,1) COMMENT 'GPS精度(米)',
    recorded_at     DATETIME NOT NULL COMMENT '上报时间',
    INDEX idx_driver_time (driver_id, recorded_at),
    INDEX idx_city_time (city_id, recorded_at)
) ENGINE=InnoDB COMMENT='司机位置快照表(按天分区)';
-- 实际使用按天做分区：PARTITION BY RANGE (TO_DAYS(recorded_at))
```

**动态定价规则表（pricing_rules）：**

```sql
CREATE TABLE pricing_rules (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    city_id         INT NOT NULL COMMENT '城市ID',
    car_type        TINYINT NOT NULL COMMENT '车型',
    rule_type       TINYINT NOT NULL COMMENT '1-基础费率 2-时段加价 3-天气加价 4-供需加价',
    start_time      TIME COMMENT '规则生效开始时间',
    end_time        TIME COMMENT '规则生效结束时间',
    base_fare       DECIMAL(8,2) COMMENT '起步价(元)',
    distance_rate   DECIMAL(8,4) COMMENT '里程费(元/米)',
    duration_rate   DECIMAL(8,4) COMMENT '时长费(元/秒)',
    min_fare        DECIMAL(8,2) COMMENT '最低消费(元)',
    surge_cap       DECIMAL(3,1) DEFAULT 2.5 COMMENT '该规则下最大加价倍数',
    priority        INT DEFAULT 0 COMMENT '规则优先级(越大越优先)',
    is_active       TINYINT DEFAULT 1 COMMENT '0-禁用 1-启用',
    effective_from  DATE COMMENT '生效日期',
    effective_to    DATE COMMENT '失效日期',
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_city_type (city_id, car_type, is_active, priority)
) ENGINE=InnoDB COMMENT='动态定价规则表';
```

**Redis 数据结构设计：**

```
# 司机实时位置（GEO集合，按城市分片）
geo:drivers:{cityId}          → Sorted Set (GeoHash编码的经纬度+司机ID)

# 司机实时状态（Hash，按司机ID分片）
driver:status:{driverId}      → Hash { state, bearing, speed, last_heartbeat, current_order_id }

# 订单等待队列（Sorted Set，按等待时间排序）
order:waiting:{cityId}        → Sorted Set (score=等待时间戳, member=orderId)

# 区域供需统计（用于动态加价，按区域+时间窗口）
surge:stats:{regionId}:{5min_window} → Hash { demand_count, supply_count, multiplier }

# 司机接单统计（用于公平性计算，滑动窗口）
driver:trips:{driverId}:{date} → String (当日接单数)

# 派单锁定（防止同一订单重复派单）
dispatch:lock:{orderId}       → String (driverId, TTL=30s)
```

### 第一步：空间查询

**Redis GEO 存储司机位置：**

```
司机位置上报（每5秒）：
  GEOADD geo:drivers:bj 116.3971 39.9165 driver_12345

3km 范围查询：
  GEORADIUS geo:drivers:bj 116.3971 39.9165 3 km WITHDIST WITHCOORD COUNT 50
```

**按城市分片：** `geo:drivers:{cityId}`，避免跨城市搜索。

**查询耗时：** < 5ms（Redis GEO 基于 Sorted Set + GeoHash，O(log(N)+M)）

**空驶司机标记：**

```
司机状态存储：
  HSET driver:status:12345 state "free"       # free/serving/offline
  HSET driver:status:12345 last_heartbeat 1715000000
```

查询时只返回 `state=free` 的司机（应用层过滤，Redis GEO 不支持条件过滤）。

**完整空间匹配实现：**

```python
import math
import redis
from dataclasses import dataclass
from typing import List, Tuple, Optional

@dataclass
class GeoPoint:
    lng: float
    lat: float

@dataclass
class CandidateDriver:
    driver_id: int
    location: GeoPoint
    bearing: int            # 行驶方向角 0-360
    speed: float            # 当前速度 km/h
    distance_m: float       # 到上车点直线距离(米)
    state: str              # free/serving/offline
    last_heartbeat: int     # 最后心跳时间戳

class SpatialMatcher:
    """空间匹配引擎：基于 Redis GEO + 应用层方向过滤"""

    EARTH_RADIUS_M = 6_371_000  # 地球半径(米)
    DEFAULT_RADIUS_KM = 3.0
    MAX_RADIUS_KM = 8.0         # 最大搜索半径
    MAX_CANDIDATES = 50          # 单次查询最大候选数
    HEARTBEAT_TIMEOUT_S = 30    # 心跳超时阈值

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def find_candidates(
        self,
        pickup: GeoPoint,
        order_bearing: float,
        city_id: int,
        radius_km: float = None
    ) -> List[CandidateDriver]:
        """
        完整的空间匹配流程：
        1. Redis GEO 查询候选
        2. 心跳过滤（排除僵尸司机）
        3. 方向预过滤
        4. 精确距离计算（Haversine 公式）
        """
        radius_km = radius_km or self.DEFAULT_RADIUS_KM
        geo_key = f"geo:drivers:{city_id}"

        # 第一步：Redis GEORADIUS 查询
        raw_results = self.redis.georadius(
            geo_key,
            pickup.lng, pickup.lat,
            radius_km, unit="km",
            withdist=True, withcoord=True,
            count=self.MAX_CANDIDATES,
            sort="ASC"  # 按距离升序
        )

        if not raw_results:
            # 无候选司机，尝试扩大搜索范围
            if radius_km < self.MAX_RADIUS_KM:
                return self.find_candidates(
                    pickup, order_bearing, city_id,
                    radius_km=min(radius_km * 2, self.MAX_RADIUS_KM)
                )
            return []

        # 第二步：批量获取司机状态 + 过滤
        now = int(time.time())
        candidates = []

        for result in raw_results:
            driver_id_str, distance, (lng, lat) = result
            driver_id = int(driver_id_str.replace("driver_", ""))

            # 获取司机实时状态
            status = self.redis.hgetall(f"driver:status:{driver_id}")
            if not status:
                continue

            # 过滤非空闲司机
            state = status.get(b"state", b"").decode()
            if state != "free":
                continue

            # 过滤心跳超时的僵尸司机
            last_hb = int(status.get(b"last_heartbeat", 0))
            if now - last_hb > self.HEARTBEAT_TIMEOUT_S:
                continue

            # 构建候选对象
            candidate = CandidateDriver(
                driver_id=driver_id,
                location=GeoPoint(lng=lng, lat=lat),
                bearing=int(status.get(b"bearing", 0)),
                speed=float(status.get(b"speed", 0)),
                distance_m=distance * 1000,  # km → m
                state=state,
                last_heartbeat=last_hb,
            )
            candidates.append(candidate)

        # 第三步：精确距离计算（Redis 返回的是直线距离，这里用 Haversine 校准）
        for c in candidates:
            c.distance_m = self._haversine_distance(
                pickup.lat, pickup.lng,
                c.location.lat, c.location.lng
            )

        # 第四步：方向预过滤——排除方向夹角 > 126° 的低优先级司机
        # 注意：不完全排除，只是标记，让后续打分环节处理
        for c in candidates:
            c.direction_diff = self._angle_diff(c.bearing, order_bearing)

        # 按距离排序返回
        candidates.sort(key=lambda c: c.distance_m)
        return candidates

    @staticmethod
    def _haversine_distance(lat1, lng1, lat2, lng2) -> float:
        """Haversine 公式计算两点间球面距离(米)"""
        R = 6_371_000
        phi1, phi2 = math.radians(lat1), math.radians(lat2)
        d_phi = math.radians(lat2 - lat1)
        d_lambda = math.radians(lng2 - lng1)

        a = (math.sin(d_phi / 2) ** 2 +
             math.cos(phi1) * math.cos(phi2) *
             math.sin(d_lambda / 2) ** 2)
        c = 2 * math.atan2(math.sqrt(a), math.sqrt(1 - a))

        return R * c

    @staticmethod
    def _angle_diff(bearing1, bearing2) -> float:
        """计算两个方向角的最小夹角(0-180)"""
        diff = abs(bearing1 - bearing2) % 360
        return min(diff, 360 - diff)

    def get_region_density(self, pickup: GeoPoint, city_id: int, radius_km: float = 3.0) -> dict:
        """
        获取区域司机密度统计，用于供需计算
        返回：{ total: 总数, free: 空闲数, avg_distance: 平均距离 }
        """
        geo_key = f"geo:drivers:{city_id}"
        raw = self.redis.georadius(
            geo_key, pickup.lng, pickup.lat,
            radius_km, unit="km",
            withdist=True, count=200, sort="ASC"
        )
        free_count = 0
        total_dist = 0.0
        for item in raw:
            driver_id_str, dist_km = item[0], item[1]
            driver_id = int(driver_id_str.replace("driver_", ""))
            status = self.redis.hget(f"driver:status:{driver_id}", "state")
            if status == b"free":
                free_count += 1
                total_dist += dist_km * 1000

        return {
            "total": len(raw),
            "free": free_count,
            "avg_distance_m": total_dist / max(free_count, 1),
        }
```

**Redis GEO 性能关键参数：**

| 操作 | 命令 | 时间复杂度 | 典型耗时(30万司机) |
|------|------|-----------|-------------------|
| 位置写入 | GEOADD | O(log N) | < 0.1ms |
| 范围查询 | GEORADIUS | O(log N + M) | 2-5ms (M=50) |
| 批量位置更新 | Pipeline GEOADD | O(K * log N) | ~50ms (K=5000) |
| 删除过期位置 | ZREM | O(log N) | < 0.1ms |

**司机位置上报的批量处理：** 司机每 5 秒上报位置，不能每条都直接写 Redis，需要批量聚合：

```python
class DriverLocationUpdater:
    """司机位置批量更新器——将高频上报聚合后批量写入 Redis"""

    BATCH_SIZE = 500
    FLUSH_INTERVAL_S = 0.5  # 每 0.5 秒批量写入一次

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.buffer: Dict[int, GeoPoint] = {}      # driver_id → 最新位置
        self.bearings: Dict[int, int] = {}          # driver_id → 方向角
        self.last_flush = time.time()

    def on_location_report(self, driver_id: int, lng: float, lat: float,
                           bearing: int, speed: float, city_id: int):
        """司机位置上报回调——先写入本地缓冲区"""
        self.buffer[driver_id] = GeoPoint(lng=lng, lat=lat)
        self.bearings[driver_id] = bearing

        # 更新司机状态 Hash
        self.redis.hset(
            f"driver:status:{driver_id}",
            mapping={
                "state": "free",  # 能上报位置说明在线
                "bearing": bearing,
                "speed": speed,
                "last_heartbeat": int(time.time()),
            }
        )

        # 达到批量阈值或超时则刷盘
        if (len(self.buffer) >= self.BATCH_SIZE or
                time.time() - self.last_flush >= self.FLUSH_INTERVAL_S):
            self.flush(city_id)

    def flush(self, city_id: int):
        """批量写入 Redis GEO"""
        if not self.buffer:
            return

        pipe = self.redis.pipeline(transaction=False)
        geo_key = f"geo:drivers:{city_id}"

        for driver_id, point in self.buffer.items():
            pipe.geoadd(geo_key, point.lng, point.lat, f"driver_{driver_id}")

        pipe.execute()
        self.buffer.clear()
        self.bearings.clear()
        self.last_flush = time.time()
```

**渐进式搜索策略：** 当 3km 范围内无空闲司机时，系统需要逐步扩大搜索范围，但不能无限扩大——8km 以外的司机到达时间超过 15 分钟，乘客大概率已经取消。

```python
class ProgressiveSearchStrategy:
    """渐进式空间搜索策略——从近到远，逐步扩大范围"""

    # 搜索阶段配置：每个阶段定义半径、超时、候选数量上限
    SEARCH_STAGES = [
        {"radius_km": 3.0, "timeout_s": 10, "max_candidates": 50, "label": "近距搜索"},
        {"radius_km": 5.0, "timeout_s": 8,  "max_candidates": 30, "label": "中距搜索"},
        {"radius_km": 8.0, "timeout_s": 6,  "max_candidates": 20, "label": "远距搜索"},
    ]

    def __init__(self, spatial_matcher: SpatialMatcher, scorer: DriverScorer):
        self.matcher = spatial_matcher
        self.scorer = scorer

    def search_best_driver(
        self, pickup: GeoPoint, order_bearing: float, city_id: int, order
    ) -> Optional[DispatchCandidate]:
        """
        多阶段渐进搜索：
        1. 先在 3km 内找最优司机
        2. 无候选则扩大到 5km
        3. 再扩大到 8km
        4. 仍无候选则进入排队等待模式
        """
        for stage in self.SEARCH_STAGES:
            candidates = self.matcher.find_candidates(
                pickup, order_bearing, city_id,
                radius_km=stage["radius_km"]
            )

            if not candidates:
                continue  # 本阶段无候选，进入下一阶段

            # 对候选司机进行综合打分
            scored = []
            for c in candidates:
                # 距离超过本阶段上限的跳过（可能来自上一阶段的残余数据）
                if c.distance_m > stage["radius_km"] * 1000:
                    continue
                score = self.scorer.score(c, order)
                # 远距离司机需要额外惩罚——乘客等待时间过长
                if c.distance_m > 3000:
                    distance_penalty = 0.7 if c.distance_m <= 5000 else 0.5
                    score *= distance_penalty
                scored.append(DispatchCandidate(driver=c, score=score, stage=stage))

            if scored:
                scored.sort(key=lambda x: -x.score)
                return scored[0]  # 返回最优候选

        # 所有阶段都无候选——排队等待
        return None
```

**Redis GEO 多城市分片与批量查询优化：** 30 万司机分布在 400+ 城市，需要按城市分片存储和查询。以下是完整的 Redis GEO 集群管理实现。

```python
class RedisGeoClusterManager:
    """Redis GEO 集群管理器——多城市分片 + 批量 Pipeline 优化"""

    # 每个城市的 Redis 分片映射（城市ID → Redis节点编号）
    CITY_SHARD_MAP = {}  # 由配置中心动态加载

    def __init__(self, redis_pool: redis.ConnectionPool):
        self.redis_pool = redis_pool
        self.shard_clients: Dict[int, redis.Redis] = {}

    def get_client_for_city(self, city_id: int) -> redis.Redis:
        """获取城市对应的 Redis 客户端"""
        shard_id = self.CITY_SHARD_MAP.get(city_id, 0)
        if shard_id not in self.shard_clients:
            self.shard_clients[shard_id] = redis.Redis(connection_pool=self.redis_pool)
        return self.shard_clients[shard_id]

    def batch_update_locations(self, city_id: int, updates: List[DriverLocationUpdate]):
        """
        批量更新司机位置——使用 Pipeline 减少网络往返
        
        单次 Pipeline 可处理 5000 个位置更新，耗时约 50ms
        30万司机 × 每5秒更新 = 每秒 6万次更新
        需要约 12 个 Pipeline 批次，总耗时约 600ms（可接受）
        """
        client = self.get_client_for_city(city_id)
        pipe = client.pipeline(transaction=False)
        geo_key = f"geo:drivers:{city_id}"
        now = int(time.time())

        for update in updates:
            # 更新 GEO 位置
            pipe.geoadd(geo_key, update.lng, update.lat, f"driver_{update.driver_id}")
            # 更新状态 Hash
            status_key = f"driver:status:{update.driver_id}"
            pipe.hset(status_key, mapping={
                "state": update.state,
                "bearing": update.bearing,
                "speed": update.speed,
                "last_heartbeat": now,
            })
            # 设置状态过期时间（30秒无心跳则自动清除）
            pipe.expire(status_key, 30)

        results = pipe.execute()
        return len(results)

    def multi_city_geosearch(
        self, pickup: GeoPoint, city_ids: List[int], radius_km: float = 3.0
    ) -> Dict[int, List[CandidateDriver]]:
        """
        跨城市边界搜索——某些订单的上车点在城市边界附近
        需要同时搜索相邻城市的司机
        
        例如：北京和天津交界处，乘客可能匹配到天津的司机
        """
        all_results: Dict[int, List[CandidateDriver]] = {}

        for city_id in city_ids:
            client = self.get_client_for_city(city_id)
            geo_key = f"geo:drivers:{city_id}"

            raw = client.georadius(
                geo_key, pickup.lng, pickup.lat,
                radius_km, unit="km",
                withdist=True, withcoord=True,
                count=50, sort="ASC"
            )

            candidates = []
            now = int(time.time())
            for item in raw:
                driver_id_str, dist_km, (lng, lat) = item
                driver_id = int(driver_id_str.replace("driver_", ""))

                # 批量获取状态（后续可用 Pipeline 优化）
                status = client.hgetall(f"driver:status:{driver_id}")
                if not status or status.get(b"state", b"").decode() != "free":
                    continue

                last_hb = int(status.get(b"last_heartbeat", 0))
                if now - last_hb > 30:
                    continue  # 僵尸司机

                candidates.append(CandidateDriver(
                    driver_id=driver_id,
                    location=GeoPoint(lng=lng, lat=lat),
                    bearing=int(status.get(b"bearing", 0)),
                    speed=float(status.get(b"speed", 0)),
                    distance_m=dist_km * 1000,
                    state="free",
                    last_heartbeat=last_hb,
                ))

            all_results[city_id] = candidates

        return all_results

    def cleanup_expired_drivers(self, city_id: int):
        """
        清理过期司机位置——司机下线但 GEO 数据未删除
        
        策略：扫描状态 Hash 过期的司机，从 GEO 集合中移除
        使用 SCAN 命令避免阻塞 Redis
        """
        client = self.get_client_for_city(city_id)
        geo_key = f"geo:drivers:{city_id}"
        now = int(time.time())

        # 获取 GEO 集合中的所有成员
        all_members = client.zrange(geo_key, 0, -1)
        pipe = client.pipeline(transaction=False)

        removed_count = 0
        for member in all_members:
            driver_id = int(member.replace("driver_", ""))
            status_key = f"driver:status:{driver_id}"

            # 检查状态是否存在（设置了 30s TTL，过期表示司机下线）
            exists = client.exists(status_key)
            if not exists:
                pipe.zrem(geo_key, member)
                removed_count += 1

                # 防止 Pipeline 过大
                if removed_count % 500 == 0:
                    pipe.execute()
                    pipe = client.pipeline(transaction=False)

        if removed_count % 500 != 0:
            pipe.execute()

        return removed_count
```

**司机位置上报的消息队列架构：** 直接将位置上报请求打到 Redis 会造成写入压力过大，需要引入 Kafka 作为缓冲层。

```python
class DriverLocationMessageQueue:
    """司机位置上报的消息队列处理——Kafka缓冲 + 批量消费写入Redis"""

    TOPIC = "driver-location-updates"
    CONSUMER_GROUP = "dispatch-location-consumer"
    BATCH_SIZE = 500
    CONSUME_INTERVAL_MS = 500  # 每 500ms 消费一批

    def __init__(self, kafka_producer, kafka_consumer, geo_cluster: RedisGeoClusterManager):
        self.kafka_producer = kafka_producer
        self.kafka_consumer = kafka_consumer
        self.geo_cluster = geo_cluster

    def publish_location_update(self, update: DriverLocationUpdate):
        """
        生产者：司机位置上报 → Kafka
        
        优势：
        - 解耦：位置上报服务不需要直接连接 Redis
        - 缓冲：高峰期上报量大时，Kafka 作为缓冲池
        - 可靠：Kafka 持久化保证位置更新不丢失
        """
        message = {
            "driver_id": update.driver_id,
            "city_id": update.city_id,
            "lng": update.lng,
            "lat": update.lat,
            "bearing": update.bearing,
            "speed": update.speed,
            "state": update.state,
            "timestamp": int(time.time()),
        }
        self.kafka_producer.send(
            self.TOPIC,
            key=str(update.driver_id).encode(),  # 按 driver_id 分区
            value=json.dumps(message).encode()
        )

    def consume_and_update_redis(self):
        """
        消费者：从 Kafka 批量读取位置更新 → 批量写入 Redis
        
        处理流程：
        1. 从 Kafka 拉取一批消息（最多 500 条）
        2. 按城市分组
        3. 对每个城市执行 Pipeline 批量写入
        4. 提交 Kafka offset
        """
        # 拉取一批消息
        records = self.kafka_consumer.poll(timeout_ms=self.CONSUME_INTERVAL_MS)
        
        # 按城市分组
        city_updates: Dict[int, List[DriverLocationUpdate]] = {}
        for topic_partition, messages in records.items():
            for msg in messages:
                data = json.loads(msg.value.decode())
                city_id = data["city_id"]
                update = DriverLocationUpdate(
                    driver_id=data["driver_id"],
                    city_id=city_id,
                    lng=data["lng"],
                    lat=data["lat"],
                    bearing=data["bearing"],
                    speed=data["speed"],
                    state=data["state"],
                )
                city_updates.setdefault(city_id, []).append(update)

        # 批量写入 Redis
        total_updated = 0
        for city_id, updates in city_updates.items():
            count = self.geo_cluster.batch_update_locations(city_id, updates)
            total_updated += count

        # 提交 offset
        self.kafka_consumer.commit()

        return total_updated
```

**数据流全景：** 司机位置从上报到可查询的完整路径：

```
司机App位置上报(每5秒)
    → Kafka Topic "driver-location-updates"
    → 消费者批量读取(每500ms)
    → Redis Pipeline批量写入
        ├── GEOADD geo:drivers:{cityId}  (位置集合)
        └── HSET driver:status:{driverId} (状态Hash + 30s TTL)
    → 派单服务 GEORADIUS 查询(实时)
```

**关键性能指标：**
- Kafka 生产延迟：< 5ms
- Kafka 消费延迟（批量 500 条）：< 50ms
- Redis Pipeline 写入（500 条）：< 50ms
- 从上报到可查询的总延迟：< 100ms（远低于 5 秒的位置更新间隔）

### 第二步：方向过滤

**问题：** 距离最近的司机可能正在反方向行驶，实际到达时间可能比远处的同向司机更长。

**方向计算：**

```python
def direction_score(driver_bearing, pickup_to_dest_bearing):
    """
    计算司机行驶方向与乘客出行方向的一致性
    
    driver_bearing: 司机当前行驶方向（0-360度，正北为0）
    pickup_to_dest_bearing: 接客点到目的地的方向
    
    返回 0-1 的分数，1 = 完全同向
    """
    diff = abs(driver_bearing - pickup_to_dest_bearing) % 360
    if diff > 180:
        diff = 360 - diff
    
    return 1.0 - (diff / 180.0)

# 示例：
# 司机向北行驶 (0°)，乘客要去北方 (10°) → diff=10°, score=0.94 (同向)
# 司机向北行驶 (0°)，乘客要去南方 (180°) → diff=180°, score=0.00 (反向)
```

**过滤策略：** 方向分数 < 0.3（夹角 > 126°）的司机降低优先级，而非直接排除——因为如果附近没有同向司机，反向司机仍然是选择。

### 第三步：综合打分

**评分函数设计：**

```
score = w1 × distance_score
      + w2 × direction_score
      + w3 × driver_rating_score
      + w4 × idle_time_score
      + w5 × fairness_score
```

**各因子的计算：**

```python
class DriverScorer:
    # 默认权重（可按城市/时段动态调整）
    WEIGHTS = {
        "distance": 0.35,
        "direction": 0.20,
        "rating": 0.15,
        "idle_time": 0.15,
        "fairness": 0.15,
    }

    def score(self, driver, order):
        d = driver
        o = order

        distance_score = self._distance_score(d.distance_to_pickup)
        direction_score = self._direction_score(d.bearing, o.bearing)
        rating_score = d.rating / 5.0
        idle_score = self._idle_score(d.idle_seconds)
        fairness_score = self._fairness_score(d)

        weights = self.get_weights(o.city_id, o.time_of_day)

        return (weights["distance"] * distance_score +
                weights["direction"] * direction_score +
                weights["rating"] * rating_score +
                weights["idle_time"] * idle_score +
                weights["fairness"] * fairness_score)

    def _distance_score(self, distance_m):
        """距离评分：越近分越高，3km 以上为0"""
        max_distance = 3000  # 3km
        if distance_m > max_distance:
            return 0
        return 1.0 - (distance_m / max_distance)

    def _direction_score(self, driver_bearing, order_bearing):
        """方向评分"""
        diff = abs(driver_bearing - order_bearing) % 360
        if diff > 180:
            diff = 360 - diff
        return 1.0 - (diff / 180.0)

    def _idle_score(self, idle_seconds):
        """空闲等待评分：空闲越久优先级越高（10分钟封顶）"""
        max_idle = 600  # 10分钟
        return min(idle_seconds / max_idle, 1.0)

    def _fairness_score(self, driver):
        """公平性评分：近期接单少的司机优先级更高"""
        recent_trips = driver.trips_last_4_hours
        target_trips = driver.online_hours_last_4 * 2  # 目标：每小时2单

        if recent_trips >= target_trips:
            return 0  # 已达目标，不再优先
        return (target_trips - recent_trips) / target_trips
```

**权重的动态调整：**

```python
class WeightManager:
    """根据时段和供需比动态调整权重"""

    PROFILES = {
        "normal": {  # 平峰期：强调公平性
            "distance": 0.30, "direction": 0.15,
            "rating": 0.15, "idle_time": 0.20, "fairness": 0.20,
        },
        "peak": {  # 高峰期：强调效率（快速匹配）
            "distance": 0.45, "direction": 0.25,
            "rating": 0.10, "idle_time": 0.10, "fairness": 0.10,
        },
        "late_night": {  # 深夜：强调安全（评分高优先）
            "distance": 0.30, "direction": 0.15,
            "rating": 0.30, "idle_time": 0.10, "fairness": 0.15,
        },
    }

    def get_weights(self, city_id, time_of_day, supply_demand_ratio):
        # 根据供需比选择配置
        if supply_demand_ratio < 0.5:  # 供不应求
            return self.PROFILES["peak"]
        elif time_of_day.hour < 6:  # 深夜
            return self.PROFILES["late_night"]
        else:
            return self.PROFILES["normal"]
```

**为什么高峰期加重距离权重？**
- 高峰期供需严重不平衡，快速匹配比精准匹配更重要
- 等待时间每多 1 分钟，乘客取消率增加 15%
- 把最近的司机派出去，比花时间找"最合适"的司机更高效

### 第四步：派单执行

```python
class DispatchService:
    def dispatch(self, order):
        """执行派单"""
        # 1. 空间查询
        nearby_drivers = self.geo_search(order.pickup_location, radius_km=3)

        if not nearby_drivers:
            return self.expand_search(order)  # 扩大搜索范围

        # 2. 过滤 + 打分
        candidates = []
        for driver in nearby_drivers:
            if not self.is_eligible(driver, order):
                continue
            score = self.scorer.score(driver, order)
            candidates.append((driver, score))

        if not candidates:
            return self.expand_search(order)

        # 3. 按 score 降序排序
        candidates.sort(key=lambda x: -x[1])

        # 4. 向 Top 1 司机发送派单
        top_driver = candidates[0][0]
        return self.send_dispatch(order, top_driver, candidates[1:])

    def send_dispatch(self, order, driver, remaining_candidates):
        """发送派单通知，等待司机响应"""
        # 发送推送通知
        self.push_service.send(driver.id, {
            "type": "dispatch",
            "order_id": order.id,
            "pickup_location": order.pickup_location,
            "pickup_distance": driver.distance_to_pickup,
            "estimated_fare": order.estimated_fare,
            "timeout_seconds": 10
        })

        # 设置 10 秒超时
        self.set_timeout(order.id, 10, callback=lambda: self.on_driver_timeout(
            order, remaining_candidates
        ))

    def on_driver_timeout(self, order, remaining_candidates):
        """司机超时未响应，派给下一个"""
        if remaining_candidates:
            next_driver = remaining_candidates[0][0]
            self.send_dispatch(order, next_driver, remaining_candidates[1:])
        else:
            # 无更多候选，扩大搜索
            self.expand_search(order)

    def on_driver_accept(self, order_id, driver_id):
        """司机接单"""
        self.cancel_timeout(order_id)
        self.match_service.confirm_match(order_id, driver_id)
        self.notify_passenger(order_id, "司机已接单，正在赶来")

    def on_driver_reject(self, order_id, driver_id, reason):
        """司机拒单"""
        # 记录拒单（影响司机评分和派单优先级）
        self.record_rejection(driver_id, reason)

        # 派给下一个候选
        remaining = self.get_remaining_candidates(order_id)
        if remaining:
            self.send_dispatch(order_id, remaining[0], remaining[1:])
        else:
            self.expand_search(order_id)
```

**为什么串行派单而非并行？**

并行派单（同时发给 3 个司机）的问题：
- 3 个司机同时接单 → 只能 1 个成功，其他 2 个白跑
- 司机看到"订单已被接走"会降低对平台的信任
- 串行派单更公平（综合分最高的优先）

但串行的延迟更高：如果 Top 1 司机 10 秒超时 + Top 2 司机 10 秒超时 = 20 秒。

**折中方案：双发策略**

```python
# 同时发给 Top 1 和 Top 2，先接单者获得订单
def dual_dispatch(self, order, candidates):
    if len(candidates) >= 2:
        driver1, driver2 = candidates[0][0], candidates[1][0]
        self.send_dispatch(order, driver1)
        self.send_dispatch(order, driver2)

        # 第一个接单的获得订单
        # 第二个收到"订单已被接走"
    else:
        self.send_dispatch(order, candidates[0][0])
```

**双发的权衡：** 减少 50% 的等待时间，但增加了 10% 的"白跑"概率。

### 公平性保障：详细机制

**问题：** 如果纯按综合分排序，某些司机总是得到低分——
- 偏远地区的司机：distance_score 总是低
- 新注册的司机：rating_score 初始值低
- 高峰期才上线的司机：idle_time_score 低

**解决方案：公平性配额 + 接单保障**

```python
class FairnessTracker:
    """追踪和保障司机的接单公平性"""

    def get_fairness_boost(self, driver):
        """计算公平性加成"""
        boost = 1.0  # 默认无加成

        # 1. 饥饿保护：在线超过 30 分钟未接单 → 加成
        if driver.minutes_since_last_trip > 30:
            boost *= 1.5  # 综合分乘以 1.5

        # 2. 新司机保护：注册 < 7 天 → 加成
        if driver.days_since_registration < 7:
            boost *= 1.3

        # 3. 偏远区域补偿：所在区域司机少 → 加成
        driver_density = self.get_driver_density(driver.location)
        if driver_density < 5:  # 3km 内 < 5 个司机
            boost *= 1.2

        return boost
```

**应用方式：** `final_score = base_score * fairness_boost`

公平性加成不是单独的评分因子，而是对综合分的乘数——这保证了即使基础分低，饥渴司机也能获得派单机会。

### 动态加价：供需平衡的价格调节

**加价算法：**

```python
class SurgePricing:
    def calculate_multiplier(self, region, current_time):
        """
        计算动态加价倍数
        返回 >= 1.0 的倍数
        """
        # 1. 计算供需比
        demand = self.get_waiting_passengers(region)  # 等待乘客数
        supply = self.get_free_drivers(region)          # 空闲司机数
        ratio = demand / max(supply, 1)                 # 供需比

        # 2. 基础加价倍数
        if ratio < 1.0:
            return 1.0  # 供大于求，不加价

        # 3. 加价计算（弹性系数控制幅度）
        elasticity = 0.5
        base_multiplier = ratio ** elasticity

        # 示例：
        # ratio=1.5 → multiplier = 1.5^0.5 = 1.22 (加价 22%)
        # ratio=3.0 → multiplier = 3.0^0.5 = 1.73 (加价 73%)
        # ratio=5.0 → multiplier = 5.0^0.5 = 2.24 (加价 124%)

        # 4. 上限保护（防止舆论危机）
        MAX_MULTIPLIER = 2.5  # 最高 2.5 倍
        multiplier = min(base_multiplier, MAX_MULTIPLIER)

        # 5. 雨雪天气加成
        weather = self.get_weather(region)
        if weather == "heavy_rain":
            multiplier *= 1.3
        elif weather == "snow":
            multiplier *= 1.5

        # 再次上限保护
        multiplier = min(multiplier, MAX_MULTIPLIER)

        return round(multiplier, 1)

    def get_surge_display(self, region):
        """用户看到的加价信息"""
        multiplier = self.calculate_multiplier(region)

        if multiplier <= 1.0:
            return "正常价格"
        elif multiplier <= 1.5:
            return f"高峰期加价 {multiplier}x"
        else:
            return f"供需紧张，加价 {multiplier}x（建议稍后再试）"
```

**完整的动态加价引擎实现：** 上面的 `SurgePricing` 是核心算法的简化版，实际生产系统需要更完整的加价检测、区域监控、平滑过渡和价格计算机制。

```python
import time
import math
from enum import Enum
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Tuple
from collections import defaultdict

class SurgeLevel(Enum):
    """加价等级——对应不同颜色提示和用户交互"""
    NORMAL = "normal"           # 1.0x 正常价格
    MODERATE = "moderate"       # 1.0-1.5x 黄色提示
    HIGH = "high"              # 1.5-2.0x 橙色提示，需二次确认
    EXTREME = "extreme"        # 2.0-2.5x 红色提示，需输入确认码

@dataclass
class RegionSurgeState:
    """区域加价状态——跟踪供需变化趋势，防止价格剧烈跳变"""
    region_id: str
    current_multiplier: float = 1.0
    previous_multiplier: float = 1.0
    demand_count: int = 0          # 当前等待乘客数
    supply_count: int = 0          # 当前空闲司机数
    avg_wait_time_s: int = 0      # 区域平均等车时间
    weather_factor: float = 1.0    # 天气因子
    event_factor: float = 1.0      # 活动因子（演唱会/比赛散场）
    last_updated: float = 0.0
    trend: str = "stable"          # rising/falling/stable

class SurgeDetectionEngine:
    """
    加价检测引擎——实时监控各区域供需状态，生成加价倍数
    
    核心设计原则：
    1. 加价必须平滑变化（每5分钟最多变化0.3x），避免价格跳变
    2. 供需数据使用滑动窗口（5分钟），避免瞬时抖动
    3. 加价有绝对上限（2.5x），且有紧急熔断机制
    4. 区域划分基于1km×1km网格，支持精细粒度
    """

    # 网格大小：1km ≈ 经度0.01° × 纬度0.009°（中纬度地区）
    GRID_LNG_STEP = 0.01
    GRID_LAT_STEP = 0.009

    # 加价参数
    MAX_MULTIPLIER = 2.5
    MIN_MULTIPLIER = 1.0
    ELASTICITY = 0.5               # 供需比→加价的弹性系数
    MAX_STEP_CHANGE = 0.3          # 单次最大变化幅度
    SURGE_WINDOW_S = 300           # 滑动窗口5分钟
    SMOOTH_FACTOR = 0.7            # 平滑系数（越大越平滑）

    # 区域状态缓存
    region_states: Dict[str, RegionSurgeState] = {}

    def __init__(self, redis_client, db_client):
        self.redis = redis_client
        self.db = db_client

    @staticmethod
    def get_region_id(lng: float, lat: float) -> str:
        """经纬度 → 区域网格ID"""
        grid_x = int(lng / SurgeDetectionEngine.GRID_LNG_STEP)
        grid_y = int(lat / SurgeDetectionEngine.GRID_LAT_STEP)
        return f"r_{grid_x}_{grid_y}"

    def calculate_surge_multiplier(self, lng: float, lat: float) -> Tuple[float, SurgeLevel]:
        """
        计算指定位置的加价倍数（完整流程）
        
        返回：(加价倍数, 加价等级)
        """
        region_id = self.get_region_id(lng, lat)

        # 获取或初始化区域状态
        state = self.region_states.get(region_id)
        if state is None:
            state = RegionSurgeState(region_id=region_id)
            self.region_states[region_id] = state

        # 第一步：获取当前供需数据（滑动窗口）
        demand, supply = self._get_supply_demand(region_id)
        state.demand_count = demand
        state.supply_count = supply

        # 第二步：计算基础供需比
        ratio = demand / max(supply, 1)

        # 第三步：供需比 → 基础加价倍数
        if ratio < 1.0:
            raw_multiplier = 1.0
        else:
            raw_multiplier = ratio ** self.ELASTICITY

        # 第四步：叠加天气因子
        weather_factor = self._get_weather_factor(region_id)
        state.weather_factor = weather_factor
        raw_multiplier *= weather_factor

        # 第五步：叠加活动因子（演唱会散场、比赛结束等）
        event_factor = self._get_event_factor(region_id)
        state.event_factor = event_factor
        raw_multiplier *= event_factor

        # 第六步：叠加等待时间因子——平均等待超过8分钟额外加价
        avg_wait = self._get_avg_wait_time(region_id)
        state.avg_wait_time_s = avg_wait
        if avg_wait > 480:  # 8分钟
            wait_bonus = 1.0 + (avg_wait - 480) / 1200  # 每20分钟额外+1.0
            raw_multiplier *= min(wait_bonus, 1.5)

        # 第七步：绝对上限保护
        raw_multiplier = min(raw_multiplier, self.MAX_MULTIPLIER)

        # 第八步：平滑过渡——与上一次倍数做加权平均
        previous = state.current_multiplier
        smoothed = previous * self.SMOOTH_FACTOR + raw_multiplier * (1 - self.SMOOTH_FACTOR)

        # 第九步：单次变化幅度限制
        delta = smoothed - previous
        if abs(delta) > self.MAX_STEP_CHANGE:
            if delta > 0:
                smoothed = previous + self.MAX_STEP_CHANGE
            else:
                smoothed = previous - self.MAX_STEP_CHANGE

        # 第十步：最终限制
        final_multiplier = round(max(self.MIN_MULTIPLIER, min(smoothed, self.MAX_MULTIPLIER)), 1)

        # 更新趋势判断
        if final_multiplier > previous + 0.05:
            state.trend = "rising"
        elif final_multiplier < previous - 0.05:
            state.trend = "falling"
        else:
            state.trend = "stable"

        # 更新状态
        state.previous_multiplier = previous
        state.current_multiplier = final_multiplier
        state.last_updated = time.time()

        # 确定加价等级
        level = self._get_surge_level(final_multiplier)

        return final_multiplier, level

    def _get_supply_demand(self, region_id: str) -> Tuple[int, int]:
        """
        从 Redis 获取区域供需数据（5分钟滑动窗口）
        
        数据来源：
        - demand: order:waiting:{cityId} 中等待订单数
        - supply: geo:drivers:{cityId} 中空闲司机数
        """
        # 从 Redis Hash 读取区域统计
        stats_key = f"surge:stats:{region_id}"
        stats = self.redis.hgetall(stats_key)

        if stats:
            demand = int(stats.get(b"demand_count", 0))
            supply = int(stats.get(b"supply_count", 0))
        else:
            demand = 0
            supply = 0

        return demand, supply

    def _get_weather_factor(self, region_id: str) -> float:
        """
        获取天气加价因子——从第三方天气API获取，缓存30分钟
        
        天气因子对照表：
        - 晴天/多云：1.0
        - 小雨：1.1
        - 大雨：1.3
        - 暴雨：1.5
        - 小雪：1.2
        - 大雪：1.5
        - 雾霾(能见度<200m)：1.4
        """
        cache_key = f"weather:factor:{region_id}"
        cached = self.redis.get(cache_key)
        if cached:
            return float(cached)

        # 调用天气API（省略具体实现）
        weather_data = self._fetch_weather_api(region_id)
        weather_code = weather_data.get("code", "clear")

        factor_map = {
            "clear": 1.0, "cloudy": 1.0, "light_rain": 1.1,
            "heavy_rain": 1.3, "storm": 1.5, "light_snow": 1.2,
            "heavy_snow": 1.5, "fog": 1.4,
        }
        factor = factor_map.get(weather_code, 1.0)

        # 缓存30分钟
        self.redis.setex(cache_key, 1800, str(factor))
        return factor

    def _get_event_factor(self, region_id: str) -> float:
        """
        获取活动加价因子——演唱会/体育赛事散场时需求暴增
        
        数据来源：运营人员手动配置的活动信息
        """
        cache_key = f"event:factor:{region_id}"
        cached = self.redis.get(cache_key)
        if cached:
            return float(cached)

        # 查询数据库中的活动配置
        # events = self.db.query("SELECT * FROM surge_events WHERE region_id = ? AND active = 1")
        return 1.0

    def _get_avg_wait_time(self, region_id: str) -> int:
        """获取区域平均等车时间（秒）——从最近派单记录计算"""
        # 从 Redis 读取最近5分钟的派单记录统计
        stats_key = f"surge:wait:{region_id}"
        avg = self.redis.get(stats_key)
        return int(avg) if avg else 0

    @staticmethod
    def _get_surge_level(multiplier: float) -> SurgeLevel:
        if multiplier <= 1.0:
            return SurgeLevel.NORMAL
        elif multiplier <= 1.5:
            return SurgeLevel.MODERATE
        elif multiplier <= 2.0:
            return SurgeLevel.HIGH
        else:
            return SurgeLevel.EXTREME

    def emergency_shutdown(self, region_id: str = None):
        """
        紧急熔断——突发事件时手动关闭加价
        例如：自然灾害、恐怖袭击等
        """
        if region_id:
            state = self.region_states.get(region_id)
            if state:
                state.current_multiplier = 1.0
                state.trend = "falling"
        else:
            # 全局熔断
            for state in self.region_states.values():
                state.current_multiplier = 1.0
                state.trend = "falling"


class PriceCalculator:
    """
    价格计算器——将基础费率与加价倍数结合，生成最终价格
    
    价格公式：
    final_fare = (base_fare + distance_rate × distance + duration_rate × duration) × surge_multiplier + tolls
    
    注意：加价倍数作用于全部费用（不含路桥费），这是行业标准做法
    """

    def __init__(self, db_client):
        self.db = db_client

    def calculate_fare(
        self,
        city_id: int,
        car_type: int,
        distance_m: float,
        duration_s: int,
        surge_multiplier: float = 1.0,
        tolls: float = 0.0,
    ) -> dict:
        """
        计算订单费用（完整实现）
        
        返回：{
            base_fare, distance_fee, duration_fee,
            subtotal, surge_amount, total_fare,
            surge_multiplier, is_surge
        }
        """
        # 获取城市×车型的费率规则
        rule = self._get_pricing_rule(city_id, car_type)

        # 各项费用计算
        base_fare = rule["base_fare"]
        distance_fee = round(rule["distance_rate"] * distance_m, 2)
        duration_fee = round(rule["duration_rate"] * duration_s, 2)

        # 小计（不含路桥费）
        subtotal = round(base_fare + distance_fee + duration_fee, 2)

        # 加价金额
        if surge_multiplier > 1.0:
            surge_amount = round(subtotal * (surge_multiplier - 1.0), 2)
        else:
            surge_amount = 0.0

        # 总价
        total_fare = round(subtotal + surge_amount + tolls, 2)

        # 最低消费保护
        if total_fare < rule["min_fare"]:
            total_fare = rule["min_fare"]

        return {
            "base_fare": base_fare,
            "distance_fee": distance_fee,
            "duration_fee": duration_fee,
            "subtotal": subtotal,
            "surge_amount": surge_amount,
            "tolls": tolls,
            "total_fare": total_fare,
            "surge_multiplier": surge_multiplier,
            "is_surge": surge_multiplier > 1.0,
        }

    def _get_pricing_rule(self, city_id: int, car_type: int) -> dict:
        """
        获取当前生效的费率规则——支持时段差异
        例如：北京快车 06:00-22:00 起步13元，22:00-06:00 起步14元
        """
        cache_key = f"pricing:rule:{city_id}:{car_type}"
        cached = self.redis.get(cache_key)
        if cached:
            return json.loads(cached)

        # 数据库查询
        # rule = self.db.query_pricing_rule(city_id, car_type, time.now())
        rule = {
            "base_fare": 13.0,
            "distance_rate": 0.0023,  # 2.3元/km = 0.0023元/m
            "duration_rate": 0.0012,  # 0.8元/min = 0.0012元/s
            "min_fare": 13.0,
        }

        self.redis.setex(cache_key, 3600, json.dumps(rule))
        return rule
```

**加价显示策略——用户端交互设计：**

```python
class SurgeDisplayService:
    """
    加价信息展示策略——不同的加价等级需要不同的用户交互
    
    设计原则：
    - MODERATE(1.0-1.5x)：显示加价标签，无需额外确认
    - HIGH(1.5-2.0x)：弹出确认框，显示预计费用和加价原因
    - EXTREME(2.0-2.5x)：需要输入验证码确认，显示替代方案（拼车/等待）
    """

    def get_display_info(self, multiplier: float, level: SurgeLevel,
                         estimated_fare: float, region_id: str) -> dict:
        """生成用户端展示信息"""
        base_info = {
            "multiplier": multiplier,
            "estimated_fare": estimated_fare,
            "level": level.value,
        }

        if level == SurgeLevel.NORMAL:
            base_info.update({
                "badge_text": "",
                "confirm_required": False,
            })

        elif level == SurgeLevel.MODERATE:
            base_info.update({
                "badge_text": f"高峰加价{multiplier}x",
                "badge_color": "#FFA500",  # 橙色
                "confirm_required": False,
                "surge_reason": "当前时段需求较大",
            })

        elif level == SurgeLevel.HIGH:
            base_info.update({
                "badge_text": f"供需紧张{multiplier}x",
                "badge_color": "#FF6600",  # 深橙色
                "confirm_required": True,
                "confirm_type": "dialog",
                "dialog_title": "确认加价叫车",
                "dialog_message": (
                    f"当前区域车辆紧张，加价{multiplier}x\n"
                    f"预计费用：¥{estimated_fare:.0f}\n"
                    f"建议：稍后再试或选择拼车"
                ),
                "surge_reason": "当前区域车辆紧张",
            })

        elif level == SurgeLevel.EXTREME:
            base_info.update({
                "badge_text": f"严重紧张{multiplier}x",
                "badge_color": "#FF0000",  # 红色
                "confirm_required": True,
                "confirm_type": "code",  # 需要输入验证码
                "dialog_title": "紧急确认",
                "dialog_message": (
                    f"当前区域车辆极度紧张，加价{multiplier}x\n"
                    f"预计费用：¥{estimated_fare:.0f}\n"
                    f"推荐替代方案：\n"
                    f"  · 拼车出行（预计¥{estimated_fare * 0.6:.0f}）\n"
                    f"  · 等待15分钟后重试"
                ),
                "surge_reason": "当前区域车辆极度紧张",
            })

        return base_info
```

**加价的双面效果：**
- 正面：激励更多司机上线（高收入吸引力），减少无效需求（价格敏感的乘客选择公交）
- 负面：舆论风险（"趁火打劫"）、用户体验差、可能违反某些城市的监管要求

**加价的替代方案：拼车推荐**

```
供需紧张时：
  1. 先推荐拼车（1 车服务 2-3 人）→ 供给效率提升 2-3 倍
  2. 拼车价格 = 原价 × 0.6 → 比加价更便宜
  3. 拼车等待时间 = 正常等车 + 绕路 5-10 分钟
```

### 全链路延迟分析

```
乘客点击"叫车"                    0ms
  → 空间查询(GEORADIUS)            5ms
  → 过滤 + 打分(50个候选)          10ms
  → 选择 Top 1                     1ms
  → 推送通知(FCM/APNs)             100ms
  → 司机手机收到通知               300ms (网络延迟)
  → 司机看到并点击接单             2000ms (人为响应)
  → 接单确认推送                   100ms
  → 乘客收到"司机已接单"           200ms
  ────────────────────────────────
  总计                         ~2716ms (< 3s 目标 ✓)
```

**关键发现：** 机器决策部分仅 16ms，超过 95% 的延迟来自网络传输和人为响应。优化算法本身的空间不大，但可以优化推送通道（使用厂商通道而非 FCM）和司机端 UI（大按钮、语音播报）。

## 常见陷阱（深度分析）

### 陷阱 1：只按距离派单

**具体后果：** 司机在高速公路上，距离乘客 500 米但无法掉头（最近出口在 2km 外）。实际到达时间 15 分钟，而 1.5km 外的同向司机只需 5 分钟。

**方向过滤的必要性：** 计算行驶方向与乘客出行方向的夹角，反向司机降低优先级。

### 陷阱 2：并行多派导致冲突

**具体场景：** 同时派给 3 个司机，2 个同时接单。

处理方案：
1. 第一个接单的获得订单（用分布式锁保证原子性）
2. 第二个收到"订单已被接走"通知
3. 司机的信任度下降——"明明我接了单为什么说被接走了？"

**更好的方案：** 串行派单 + 短超时（5 秒而非 10 秒），或双发策略（最多同时派 2 个）。

### 陷阱 3：加价无上限

**Uber 的真实教训：** 2014 年悉尼人质事件期间，Uber 的算法自动将加价提高到 4 倍以上，引发公众强烈抗议。最终 Uber 退还了加价费用并设置了加价上限。

**教训：** 加价必须有上限，且在紧急事件时应手动暂停加价。

### 陷阱 4：忽略拒单率

**问题：** 如果司机频繁拒单却不受惩罚，派单效率会急剧下降。

```
某司机拒单率 50%：
  派给他 10 次 → 拒 5 次 → 5 次超时(10秒) → 50 秒浪费
  期间 5 个乘客的等待时间增加 50 秒
```

**拒单惩罚机制：**
- 拒单率 > 30%：降低派单优先级（fairness_score 乘 0.5）
- 拒单率 > 50%：暂停派单 30 分钟
- 连续拒单 3 次：暂停派单 1 小时

但需要区分"恶意拒单"和"合理拒单"：
- 恶意拒单：只接长途单、只接高价单
- 合理拒单：车辆故障、身体不适、接孩子

**合理拒单的识别：** 司机可以在 App 中选择拒单原因，标记为"合理"的不计入拒单率。

## 延伸思考

- **智能调度**：在乘客发单前，系统预测需求热点，提前将司机引导到需求密集区域。这需要需求预测模型（基于历史数据+天气+事件）和司机激励（引导奖励金）。
- **自动驾驶**：派单系统无需考虑司机意愿和拒单，但需要考虑车辆电量/续航、车队管理、远程监控等新问题。
- **顺路单**：司机设置目的地，系统只派同方向的订单。实现方式：在评分函数中大幅提升 direction_score 的权重（> 0.5），同时放宽距离限制（5km 而非 3km）。
### 完整派单匹配算法（空间代码实现）

前面的 `DriverScorer` 给出了评分框架，但实际生产中需要更精细的空间计算——不仅是直线距离，还要考虑道路网络方向、司机行驶轨迹预测等因素。

```python
import math
from dataclasses import dataclass
from typing import List, Tuple, Optional, Dict

@dataclass
class GeoPoint:
    lng: float
    lat: float

@dataclass
class DispatchCandidate:
    driver_id: int
    location: GeoPoint
    bearing: int              # 行驶方向角 0-360
    speed: float              # 当前速度 km/h
    distance_m: float         # 直线距离
    estimated_eta_s: float    # 预估到达时间（秒）
    direction_score: float    # 方向得分
    distance_score: float     # 距离得分
    rating_score: float       # 评分得分
    idle_score: float         # 空闲得分
    fairness_score: float     # 公平性得分
    total_score: float        # 综合得分
    fairness_boost: float     # 公平性加成

class DispatchMatchingEngine:
    """
    完整派单匹配引擎

    匹配流程：
    1. Redis GEO 查询候选司机
    2. 计算每个候选的空间属性（方向夹角、ETA 预估）
    3. 多因子评分 + 公平性加成
    4. 选择 Top 1 候选
    """

    EARTH_RADIUS_M = 6_371_000

    def __init__(self, redis_client, db, weight_manager, fairness_tracker):
        self.redis = redis_client
        self.db = db
        self.weight_manager = weight_manager
        self.fairness_tracker = fairness_tracker

    def find_best_match(self, order) -> Optional[DispatchCandidate]:
        """为订单找到最优司机"""
        # 第一步：空间查询候选
        candidates = self._geo_search(
            order.pickup_location, order.pickup_bearing,
            order.city_id
        )

        if not candidates:
            return None

        # 第二步：逐个评分
        scored = []
        weights = self.weight_manager.get_weights(
            order.city_id, order.created_at,
            self._get_supply_demand_ratio(order.city_id)
        )

        for driver in candidates:
            # 计算各维度得分
            distance_score = self._distance_score(driver.distance_m)
            direction_score = self._direction_score(driver.bearing, order.pickup_bearing)

            # 从数据库获取司机评分
            driver_info = self._get_driver_info(driver.driver_id)
            if not driver_info:
                continue

            rating_score = driver_info.rating / 5.0
            idle_score = self._idle_score(driver_info.minutes_since_last_trip)
            fairness_score = self._fairness_score(driver_info)

            # 计算公平性加成
            fairness_boost = self.fairness_tracker.get_fairness_boost(driver_info)

            # 加权综合分
            total = (
                weights["distance"] * distance_score +
                weights["direction"] * direction_score +
                weights["rating"] * rating_score +
                weights["idle_time"] * idle_score +
                weights["fairness"] * fairness_score
            ) * fairness_boost

            # 预估 ETA
            eta = self._estimate_eta(driver, order)

            candidate = DispatchCandidate(
                driver_id=driver.driver_id,
                location=driver.location,
                bearing=driver.bearing,
                speed=driver.speed,
                distance_m=driver.distance_m,
                estimated_eta_s=eta,
                direction_score=direction_score,
                distance_score=distance_score,
                rating_score=rating_score,
                idle_score=idle_score,
                fairness_score=fairness_score,
                total_score=total,
                fairness_boost=fairness_boost,
            )
            scored.append(candidate)

        # 第三步：排序选择
        scored.sort(key=lambda c: c.total_score, reverse=True)
        return scored[0] if scored else None

    def _geo_search(self, pickup: GeoPoint, order_bearing: float,
                     city_id: int, radius_km: float = 3.0) -> list:
        """Redis GEO 查询 + 状态过滤 + 方向预筛选"""
        geo_key = f"geo:drivers:{city_id}"
        raw = self.redis.georadius(
            geo_key, pickup.lng, pickup.lat,
            radius_km, unit="km",
            withdist=True, withcoord=True,
            count=50, sort="ASC"
        )

        candidates = []
        now = int(time.time())
        for item in raw:
            driver_id_str = item[0]
            dist_km = item[1]
            lng, lat = item[2]

            driver_id = int(driver_id_str.replace("driver_", ""))
            status = self.redis.hgetall(f"driver:status:{driver_id}")

            if not status or status.get(b"state") != b"free":
                continue

            last_hb = int(status.get(b"last_heartbeat", 0))
            if now - last_hb > 30:
                continue

            bearing = int(status.get(b"bearing", 0))
            speed = float(status.get(b"speed", 0))

            # 精确距离
            distance_m = self._haversine(pickup.lat, pickup.lng, lat, lng)

            candidates.append(type("Driver", (), {
                "driver_id": driver_id,
                "location": GeoPoint(lng=lng, lat=lat),
                "bearing": bearing,
                "speed": speed,
                "distance_m": distance_m,
            }))

        return candidates

    @staticmethod
    def _haversine(lat1, lng1, lat2, lng2) -> float:
        """Haversine 公式计算球面距离（米）"""
        R = 6_371_000
        phi1, phi2 = math.radians(lat1), math.radians(lat2)
        d_phi = math.radians(lat2 - lat1)
        d_lambda = math.radians(lng2 - lng1)
        a = math.sin(d_phi/2)**2 + math.cos(phi1)*math.cos(phi2)*math.sin(d_lambda/2)**2
        return R * 2 * math.atan2(math.sqrt(a), math.sqrt(1-a))

    @staticmethod
    def _direction_score(driver_bearing: int, order_bearing: float) -> float:
        """方向得分：夹角越小分越高"""
        diff = abs(driver_bearing - order_bearing) % 360
        diff = min(diff, 360 - diff)
        return 1.0 - (diff / 180.0)

    @staticmethod
    def _distance_score(distance_m: float) -> float:
        """距离得分：3km 以内线性递减"""
        if distance_m > 3000:
            return 0
        return 1.0 - (distance_m / 3000)

    @staticmethod
    def _idle_score(minutes_idle: float) -> float:
        """空闲等待得分：空闲越久优先级越高"""
        return min(minutes_idle / 600, 1.0)  # 10分钟封顶

    @staticmethod
    def _fairness_score(driver_info) -> float:
        """公平性得分：近期接单少的优先"""
        recent = driver_info.trips_last_4_hours
        target = driver_info.online_hours_last_4 * 2
        if recent >= target:
            return 0
        return (target - recent) / max(target, 1)

    def _estimate_eta(self, driver, order) -> float:
        """预估到达时间（秒）——综合考虑距离、方向、速度"""
        distance_m = driver.distance_m

        # 方向夹角越大，实际路程越长（掉头成本）
        angle_diff = abs(driver.bearing - order.pickup_bearing) % 360
        angle_diff = min(angle_diff, 360 - angle_diff)

        # 方向系数：同向 1.0，反向 2.5（需要掉头）
        direction_factor = 1.0 + (angle_diff / 180.0) * 1.5

        # 实际路程 = 直线距离 × 路网系数 × 方向系数
        ROAD_NETWORK_FACTOR = 1.3  # 城市道路非直线系数
        actual_distance = distance_m * ROAD_NETWORK_FACTOR * direction_factor

        # 速度估计：取司机当前速度和城市平均速度的加权
        CITY_AVG_SPEED_KMH = 25  # 城市平均车速 25km/h
        if driver.speed > 5:  # 正在行驶
            effective_speed_kmh = driver.speed * 0.6 + CITY_AVG_SPEED_KMH * 0.4
        else:  # 静止状态
            effective_speed_kmh = CITY_AVG_SPEED_KMH

        effective_speed_ms = effective_speed_kmh * 1000 / 3600
        eta_seconds = actual_distance / max(effective_speed_ms, 1)

        # 红绿灯和路口延迟：每公里约增加 60 秒
        traffic_delay = (actual_distance / 1000) * 60
        eta_seconds += traffic_delay

        return eta_seconds

    def _get_driver_info(self, driver_id: int) -> Optional[object]:
        """从缓存获取司机信息"""
        cached = self.redis.get(f"driver:info:{driver_id}")
        if cached:
            return json.loads(cached)
        info = self.db.query_one(
            "SELECT * FROM drivers WHERE id = %s", driver_id
        )
        if info:
            self.redis.setex(f"driver:info:{driver_id}", 300, json.dumps(info))
        return info

    def _get_supply_demand_ratio(self, city_id: int) -> float:
        """获取供需比"""
        demand = self.redis.zcard(f"order:waiting:{city_id}")
        supply = self.redis.zcard(f"geo:drivers:{city_id}")
        return demand / max(supply, 1)
```

### 司机 ETA 预测完整实现

ETA（Estimated Time of Arrival）预测是派单质量的关键——距离最近的司机不一定最先到达。以下是考虑道路网络、行驶方向、实时路况的 ETA 预测模型。

```python
import math
from collections import defaultdict
from typing import List, Tuple, Dict, Optional

@dataclass
class ETAPrediction:
    """ETA 预测结果"""
    driver_id: int
    order_id: int
    distance_m: float              # 实际路程距离（米）
    eta_seconds: float             # 预估到达时间（秒）
    confidence: float              # 预测置信度 0-1
    route_type: str                # 路线类型：straight/uturn/complex
    factors: Dict                  # 各因子明细

class DriverETAPredictor:
    """
    司机 ETA 预测器

    预测模型分三层：
    1. 基础层：直线距离 × 路网系数 × 方向系数 → 粗略 ETA
    2. 历史层：同区域同时段的历史平均速度 → 校准 ETA
    3. 实时层：当前交通状态（拥堵/畅通）→ 最终 ETA

    不依赖外部地图 API（延迟太高），全部基于自有数据计算
    """

    # 城市道路非直线系数：实际路程 / 直线距离
    ROAD_NETWORK_FACTORS = {
        "grid_city": 1.25,    # 网格状路网（如北京）：较直
        "radial_city": 1.40,  # 放射状路网（如成都）：绕路多
        "mountain_city": 1.55, # 山城（如重庆）：绕路严重
    }

    # 方向夹角 → 路程放大系数（掉头、绕路）
    BEARING_PENALTY = [
        (0, 30, 1.0),     # 同向：无额外代价
        (30, 60, 1.1),    # 微偏：小幅绕路
        (60, 120, 1.3),   # 斜向：中等绕路
        (120, 150, 1.8),  # 大角度：需要掉头
        (150, 180, 2.5),  # 反向：远距离掉头
    ]

    def __init__(self, redis_client, db):
        self.redis = redis_client
        self.db = db

    def predict_eta(self, driver_location: GeoPoint, driver_bearing: int,
                    driver_speed: float, pickup_location: GeoPoint,
                    pickup_bearing: float, city_id: int,
                    city_type: str = "grid_city") -> ETAPrediction:
        """
        预测司机到达上车点的时间

        参数：
          driver_location: 司机当前位置
          driver_bearing: 司机行驶方向（0-360）
          driver_speed: 司机当前速度（km/h）
          pickup_location: 上车点位置
          pickup_bearing: 乘客出行方向
          city_id: 城市 ID
          city_type: 城市路网类型
        """
        # 第一步：计算直线距离
        straight_distance = self._haversine(
            driver_location.lat, driver_location.lng,
            pickup_location.lat, pickup_location.lng
        )

        # 第二步：计算方向夹角
        angle_diff = self._angle_diff(driver_bearing, pickup_bearing)

        # 第三步：路程放大系数
        road_factor = self.ROAD_NETWORK_FACTORS.get(city_type, 1.35)
        bearing_penalty = self._get_bearing_penalty(angle_diff)

        # 第四步：实际路程估算
        actual_distance = straight_distance * road_factor * bearing_penalty

        # 第五步：速度估算（三层模型）
        base_speed = self._get_base_speed(city_id, city_type)
        historical_speed = self._get_historical_speed(city_id, pickup_location)
        effective_speed = self._combine_speeds(
            driver_speed, base_speed, historical_speed
        )

        # 第六步：行驶时间
        effective_speed_ms = effective_speed * 1000 / 3600
        travel_time = actual_distance / max(effective_speed_ms, 1)

        # 第七步：交通延迟
        traffic_delay = self._estimate_traffic_delay(
            city_id, pickup_location, actual_distance
        )

        # 第八步：路口延迟（每公里约 1.5 个红绿灯，每个 30 秒）
        intersection_delay = (actual_distance / 1000) * 1.5 * 30

        # 第九步：总 ETA
        total_eta = travel_time + traffic_delay + intersection_delay

        # 第十步：置信度评估
        confidence = self._estimate_confidence(
            straight_distance, angle_diff, driver_speed, city_id
        )

        # 路线类型判断
        if angle_diff < 60:
            route_type = "straight"
        elif angle_diff < 120:
            route_type = "complex"
        else:
            route_type = "uturn"

        return ETAPrediction(
            driver_id=0,  # 调用方设置
            order_id=0,
            distance_m=round(actual_distance),
            eta_seconds=round(total_eta),
            confidence=confidence,
            route_type=route_type,
            factors={
                "straight_distance_m": round(straight_distance),
                "road_factor": road_factor,
                "bearing_penalty": bearing_penalty,
                "angle_diff": angle_diff,
                "effective_speed_kmh": round(effective_speed, 1),
                "travel_time_s": round(travel_time),
                "traffic_delay_s": round(traffic_delay),
                "intersection_delay_s": round(intersection_delay),
            }
        )

    def _get_bearing_penalty(self, angle_diff: float) -> float:
        """方向夹角 → 路程放大系数"""
        for low, high, penalty in self.BEARING_PENALTY:
            if low <= angle_diff < high:
                return penalty
        return 2.5  # 默认反向

    def _get_base_speed(self, city_id: int, city_type: str) -> float:
        """基础速度：城市级别的默认行驶速度"""
        # 从 Redis 获取实时城市平均速度
        cached = self.redis.get(f"eta:base_speed:{city_id}")
        if cached:
            return float(cached)

        # 默认值：网格城市 28km/h，放射城市 24km/h，山城 20km/h
        defaults = {"grid_city": 28, "radial_city": 24, "mountain_city": 20}
        return defaults.get(city_type, 25)

    def _get_historical_speed(self, city_id: int, location: GeoPoint) -> float:
        """历史速度：同区域同时段的历史平均"""
        # 从 ClickHouse 查询过去 7 天同一区域同时段的平均速度
        hour = datetime.now().hour
        region_key = self._get_region_key(location)

        result = self.db.query_one("""
            SELECT avg_speed_kmh FROM region_speed_history
            WHERE city_id = %s AND region_key = %s AND hour = %s
            ORDER BY date DESC LIMIT 1
        """, city_id, region_key, hour)

        return result["avg_speed_kmh"] if result else 25.0

    @staticmethod
    def _combine_speeds(current_speed: float, base_speed: float,
                        historical_speed: float) -> float:
        """
        综合速度估算

        权重策略：
        - 司机正在行驶（>5km/h）：当前速度 50% + 历史 30% + 基础 20%
        - 司机静止：历史 60% + 基础 40%
        """
        if current_speed > 5:
            return current_speed * 0.5 + historical_speed * 0.3 + base_speed * 0.2
        else:
            return historical_speed * 0.6 + base_speed * 0.4

    def _estimate_traffic_delay(self, city_id: int, location: GeoPoint,
                                 distance_m: float) -> float:
        """实时交通延迟估算"""
        # 从交通数据服务获取拥堵系数
        congestion = self.redis.get(f"traffic:congestion:{city_id}")
        if congestion:
            congestion_factor = float(congestion)
        else:
            congestion_factor = 1.0  # 默认畅通

        # 拥堵系数 > 1 时，按距离线性增加延迟
        if congestion_factor > 1.0:
            # 每公里额外延迟 = (拥堵系数 - 1) * 60 秒
            extra_delay_per_km = (congestion_factor - 1.0) * 60
            return (distance_m / 1000) * extra_delay_per_km
        return 0

    @staticmethod
    def _estimate_confidence(distance_m: float, angle_diff: float,
                             speed: float, city_id: int) -> float:
        """ETA 预测置信度评估"""
        confidence = 1.0

        # 距离越远，不确定性越大
        if distance_m > 2000:
            confidence *= 0.8
        elif distance_m > 5000:
            confidence *= 0.6

        # 反向行驶，不确定性大
        if angle_diff > 120:
            confidence *= 0.7

        # 速度为 0（司机静止），不确定何时启动
        if speed < 5:
            confidence *= 0.85

        return max(confidence, 0.3)

    @staticmethod
    def _haversine(lat1, lng1, lat2, lng2) -> float:
        R = 6_371_000
        phi1, phi2 = math.radians(lat1), math.radians(lat2)
        d_phi = math.radians(lat2 - lat1)
        d_lambda = math.radians(lng2 - lng1)
        a = math.sin(d_phi/2)**2 + math.cos(phi1)*math.cos(phi2)*math.sin(d_lambda/2)**2
        return R * 2 * math.atan2(math.sqrt(a), math.sqrt(1-a))

    @staticmethod
    def _angle_diff(b1, b2) -> float:
        diff = abs(b1 - b2) % 360
        return min(diff, 360 - diff)

    @staticmethod
    def _get_region_key(location: GeoPoint) -> str:
        """将经纬度映射到区域网格（约 1km×1km）"""
        lat_grid = int(location.lat * 100)  # 0.01度 ≈ 1.1km
        lng_grid = int(location.lng * 100)
        return f"{lat_grid}_{lng_grid}"
```

**ETA 预测各因子对结果的影响：**

| 场景 | 直线距离 | 方向夹角 | 路网系数 | 方向系数 | 实际路程 | 预估速度 | ETA |
|------|---------|---------|---------|---------|---------|---------|------|
| 同向500m | 500m | 10° | 1.25 | 1.0 | 625m | 28km/h | 80s |
| 同向1.5km | 1500m | 20° | 1.25 | 1.0 | 1875m | 28km/h | 240s |
| 斜向800m | 800m | 90° | 1.25 | 1.3 | 1300m | 25km/h | 187s |
| 反向500m | 500m | 170° | 1.25 | 2.5 | 1563m | 22km/h | 256s |
| 反向1km | 1000m | 160° | 1.25 | 2.5 | 3125m | 22km/h | 512s |

**关键洞察：** 500 米反向的司机 ETA（256s）比 1.5km 同向的司机（240s）还长——这就是为什么不能只按距离派单。

### 场景 5：高峰期派单雪崩

```python
# 场景描述：晚高峰 18:00，5 万乘客同时叫车，30 万司机在线但空闲司机仅 3 万
# 大量订单无法匹配到司机 → 乘客反复重试 → 系统负载飙升 → 派单服务雪崩

class PeakHourProtection:
    """高峰期派单保护——防止雪崩"""

    def __init__(self, redis_client, alert_service):
        self.redis = redis_client
        self.alert = alert_service

    def check_and_protect(self, city_id: int) -> dict:
        """检查高峰期状态并采取保护措施"""
        demand = self.redis.zcard(f"order:waiting:{city_id}")
        supply = self.redis.get(f"stats:free_drivers:{city_id}")
        supply = int(supply) if supply else 0
        ratio = demand / max(supply, 1)

        actions_taken = []

        # 一级保护：供需比 > 3:1 → 触发加价
        if ratio > 3.0:
            self.redis.set(f"surge:triggered:{city_id}", "1", ex=1800)
            actions_taken.append("触发动态加价")

        # 二级保护：等待订单 > 5000 → 限流新订单
        if demand > 5000:
            self.redis.set(f"order:throttle:{city_id}", "1", ex=600)
            actions_taken.append(f"限流：每秒最多接收 500 新订单（当前{demand}等待）")

        # 三级保护：等待订单 > 10000 → 排队提示
        if demand > 10000:
            self.redis.set(f"order:queue_notice:{city_id}", json.dumps({
                "message": "当前叫车人数较多，预计等待 20-30 分钟",
                "estimated_wait_minutes": 25,
            }), ex=1800)
            actions_taken.append("显示排队提示，预估等待 25 分钟")

        # 四级保护：暂停非紧急功能
        if demand > 20000:
            # 暂停拼车、代叫等复杂功能
            self.redis.set(f"feature:disable_carpool:{city_id}", "1", ex=3600)
            actions_taken.append("暂停拼车功能，释放派单计算资源")

        if actions_taken:
            self.alert.send_critical(
                title=f"城市 {city_id} 高峰期保护已启动",
                message=f"供需比={ratio:.1f}, 等待={demand}, 空闲={supply}。"
                        f"措施：{', '.join(actions_taken)}"
            )

        return {
            "city_id": city_id,
            "demand": demand,
            "supply": supply,
            "ratio": ratio,
            "actions": actions_taken,
        }

    def estimate_wait_time(self, city_id: int) -> int:
        """估算当前等车时间（分钟）"""
        demand = self.redis.zcard(f"order:waiting:{city_id}")
        supply = int(self.redis.get(f"stats:free_drivers:{city_id}") or 0)

        if supply == 0:
            return 60  # 无空闲司机，等 1 小时+

        # 每个空闲司机每 15 分钟完成一单
        orders_per_15min = supply * 1
        minutes_per_batch = 15

        if demand <= supply:
            return 5  # 供大于求，5 分钟内

        batches = math.ceil((demand - supply) / orders_per_15min)
        return min(batches * minutes_per_batch, 60)
```

### 场景 6：司机端 GPS 漂移导致错误派单

```python
# 场景描述：司机在地下车库等 GPS 信号弱区域 → 位置漂移到 2km 外
# → 被派了 2km 外的订单 → 司机实际需要 10 分钟才能出车库
# → 乘客等待时间远超预期

class GPSDriftDetector:
    """GPS 漂移检测器——识别并过滤位置异常的司机"""

    def __init__(self, redis_client):
        self.redis = redis_client

    def is_likely_drift(self, driver_id: int, current_location: GeoPoint,
                        current_speed: float) -> dict:
        """
        检测 GPS 漂移

        漂移特征：
        1. 位置突变：5 秒内移动距离 > 200m（物理不可能，除非瞬移）
        2. 精度低：GPS accuracy > 100m
        3. 速度矛盾：报告位置快速移动但速度为 0（或相反）
        """
        # 获取上次位置
        last = self.redis.hgetall(f"driver:location_history:{driver_id}")
        if not last:
            return {"is_drift": False, "confidence": 0}

        last_lng = float(last.get(b"lng", 0))
        last_lat = float(last.get(b"lat", 0))
        last_time = float(last.get(b"timestamp", 0))
        last_accuracy = float(last.get(b"accuracy", 100))

        # 计算位移
        distance_m = self._haversine(last_lat, last_lng,
                                      current_location.lat, current_location.lng)
        time_gap = time.time() - last_time

        # 检测1：5秒内位移超过物理可能（最高速度 200km/h = 55m/s）
        max_possible_distance = time_gap * 55  # 55 m/s
        if distance_m > max_possible_distance and time_gap < 30:
            return {
                "is_drift": True,
                "confidence": 0.9,
                "reason": f"位置突变: {distance_m:.0f}m/{time_gap:.0f}s（物理不可能）"
            }

        # 检测2：位置精度差
        current_accuracy = float(self.redis.hget(
            f"driver:status:{driver_id}", "accuracy"
        ) or 100)
        if current_accuracy > 200:
            return {
                "is_drift": True,
                "confidence": 0.7,
                "reason": f"GPS 精度差: {current_accuracy:.0f}m"
            }

        # 检测3：速度矛盾
        implied_speed_kmh = (distance_m / max(time_gap, 1)) * 3.6
        if abs(implied_speed_kmh - current_speed) > 100:
            return {
                "is_drift": True,
                "confidence": 0.6,
                "reason": f"速度矛盾: GPS暗示{implied_speed_kmh:.0f}km/h, 报告{current_speed:.0f}km/h"
            }

        return {"is_drift": False, "confidence": 0}

    @staticmethod
    def _haversine(lat1, lng1, lat2, lng2) -> float:
        R = 6_371_000
        phi1, phi2 = math.radians(lat1), math.radians(lat2)
        d_phi = math.radians(lat2 - lat1)
        d_lambda = math.radians(lng2 - lng1)
        a = math.sin(d_phi/2)**2 + math.cos(phi1)*math.cos(phi2)*math.sin(d_lambda/2)**2
        return R * 2 * math.atan2(math.sqrt(a), math.sqrt(1-a))

# 集成到派单流程：
# 在 SpatialMatcher.find_candidates() 中，对每个候选司机调用 is_likely_drift()
# 如果检测到漂移 → 排除该司机（不参与派单），并标记其位置为"不可靠"
# 司机连续 3 分钟无可靠位置 → 状态改为"GPS信号弱"，不参与派单
```

### 场景 7：大面积司机下线（城市交通管制）

```python
# 场景描述：城市举办大型活动，核心区域 10km 范围内交通管制
# → 3 万司机被迫下线或绕行 → 该区域供需严重失衡

class CityEmergencyHandler:
    """城市紧急事件处理——大面积司机下线时的应急方案"""

    def __init__(self, redis_client, db, alert_service):
        self.redis = redis_client
        self.db = db
        self.alert = alert_service

    def handle_mass_offline(self, city_id: int, affected_area: dict,
                            estimated_duration_hours: float):
        """
        处理大面积司机下线

        affected_area: {center_lat, center_lng, radius_km}
        """
        # 第一步：评估影响范围
        affected_drivers = self._count_affected_drivers(
            city_id, affected_area
        )
        waiting_orders = self.redis.zcard(f"order:waiting:{city_id}")

        # 第二步：设置区域禁行标记
        area_key = f"restricted:area:{city_id}:{int(time.time())}"
        self.redis.set(area_key, json.dumps({
            "center_lat": affected_area["center_lat"],
            "center_lng": affected_area["center_lng"],
            "radius_km": affected_area["radius_km"],
            "expires_at": time.time() + estimated_duration_hours * 3600,
        }))
        # 派单时检查：候选司机是否在禁行区域内

        # 第三步：调整周边区域策略
        # 扩大搜索半径：3km → 8km
        self.redis.set(f"dispatch:search_radius:{city_id}", "8", ex=86400)

        # 第四步：触发高倍加价
        self.redis.set(f"surge:forced:{city_id}", json.dumps({
            "multiplier": 2.0,
            "reason": "交通管制导致大面积司机下线",
            "expires_at": time.time() + estimated_duration_hours * 3600,
        }))

        # 第五步：引导外围司机进入受影响区域
        self._dispatch_incentive(
            city_id, affected_area,
            bonus_amount=20.0,  # 引导奖励 20 元
            radius_km=affected_area["radius_km"] + 5,
        )

        # 第六步：通知乘客
        self.redis.set(f"passenger:notice:{city_id}", json.dumps({
            "message": "受交通管制影响，当前叫车等待时间可能较长，建议选择公共交通",
            "show_duration_hours": estimated_duration_hours,
        }))

        self.alert.send_critical(
            title=f"城市 {city_id} 大面积司机下线",
            message=f"受影响司机: {affected_drivers}, 等待订单: {waiting_orders}, "
                    f"预计持续: {estimated_duration_hours}小时"
        )

    def _count_affected_drivers(self, city_id: int, area: dict) -> int:
        """统计受影响区域内的司机数"""
        geo_key = f"geo:drivers:{city_id}"
        results = self.redis.georadius(
            geo_key, area["center_lng"], area["center_lat"],
            area["radius_km"], unit="km", count=10000
        )
        return len(results)

    def _dispatch_incentive(self, city_id: int, area: dict,
                            bonus_amount: float, radius_km: float):
        """向外围司机发送引导奖励"""
        # 向 radius_km 范围内的司机推送通知
        # "前往 XX 区域可获 20 元额外奖励"
        pass
```

### 完整动态定价引擎

前面的 `SurgeDetectionEngine` 和 `PriceCalculator` 分别实现了加价检测和费用计算，但生产环境需要一个统一的动态定价引擎，将供需加价、时段基础定价、天气加价、区域网格计算整合为一个完整的定价流水线。

**设计目标：**
- 按 1km x 1km 网格实时计算供需比，而非城市级别粗粒度
- 时段基础定价：早晚高峰、深夜、平峰各有不同基础费率
- 天气加价自动叠加，支持暴雨/大雪/雾霾等多级天气
- 加价倍数有绝对上限（2.5x），单次变化幅度有上限（0.3x/5min）
- 支持紧急熔断（自然灾害、公共事件时一键关闭加价）

```python
import time
import math
import json
from enum import Enum
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Tuple
from collections import defaultdict

class TimeSlot(Enum):
    """时段枚举——不同时段有不同的基础费率和加价敏感度"""
    LATE_NIGHT = "late_night"    # 00:00-06:00 深夜
    MORNING_PEAK = "morning_peak"  # 07:00-09:30 早高峰
    FORENOON = "forenoon"        # 09:30-12:00 上午平峰
    LUNCH_PEAK = "lunch_peak"    # 11:30-13:30 午高峰
    AFTERNOON = "afternoon"      # 13:30-17:00 下午平峰
    EVENING_PEAK = "evening_peak" # 17:00-19:30 晚高峰
    EVENING = "evening"          # 19:30-22:00 晚间
    NIGHT = "night"              # 22:00-24:00 夜间

class WeatherCondition(Enum):
    """天气条件枚举"""
    CLEAR = "clear"              # 晴天
    CLOUDY = "cloudy"            # 多云
    LIGHT_RAIN = "light_rain"    # 小雨
    HEAVY_RAIN = "heavy_rain"    # 大雨
    STORM = "storm"              # 暴雨
    LIGHT_SNOW = "light_snow"    # 小雪
    HEAVY_SNOW = "heavy_snow"    # 大雪
    FOG = "fog"                  # 雾霾

@dataclass
class GridSupplyDemand:
    """网格供需数据——1km x 1km 区域的实时供需快照"""
    grid_id: str
    demand_count: int = 0         # 等待乘客数
    supply_count: int = 0         # 空闲司机数
    avg_wait_time_s: float = 0.0  # 区域平均等车时间
    demand_trend: float = 0.0     # 需求变化趋势（正=上升）
    supply_trend: float = 0.0     # 供给变化趋势（正=上升）
    timestamp: float = 0.0

@dataclass
class PricingResult:
    """定价结果——包含完整的费用明细"""
    base_fare: float              # 起步价
    distance_fee: float           # 里程费
    duration_fee: float           # 时长费
    subtotal: float               # 小计（加价前）
    surge_multiplier: float       # 加价倍数
    surge_amount: float           # 加价金额
    weather_multiplier: float     # 天气加价因子
    time_slot_multiplier: float   # 时段加价因子
    tolls: float                  # 路桥费
    total_fare: float             # 最终总价
    is_surge: bool                # 是否加价
    surge_level: str              # 加价等级
    breakdown: Dict               # 费用明细

class DynamicPricingEngine:
    """
    完整动态定价引擎

    定价流水线：
    1. 确定时段 → 获取时段基础费率
    2. 计算网格供需比 → 供需加价倍数
    3. 叠加天气加价因子
    4. 叠加时段加价因子（深夜/高峰额外加价）
    5. 平滑过渡（与上一次倍数做加权平均）
    6. 单次变化幅度限制
    7. 绝对上限保护
    8. 计算最终费用

    关键参数：
    - 网格大小：1km x 1km
    - 供需弹性系数：0.5（ratio^0.5 → 加价倍数）
    - 最大加价倍数：2.5x
    - 单次最大变化：0.3x/5min
    - 平滑系数：0.7（越大越平滑）
    """

    # 网格参数
    GRID_LNG_STEP = 0.01   # 1km ≈ 0.01度经度
    GRID_LAT_STEP = 0.009  # 1km ≈ 0.009度纬度

    # 加价参数
    MAX_SURGE_MULTIPLIER = 2.5
    MIN_SURGE_MULTIPLIER = 1.0
    SUPPLY_DEMAND_ELASTICITY = 0.5
    MAX_STEP_CHANGE = 0.3       # 单次最大变化幅度
    SMOOTH_FACTOR = 0.7         # 平滑系数

    # 时段基础费率倍数（在基础费率之上额外乘的系数）
    TIME_SLOT_MULTIPLIERS = {
        TimeSlot.LATE_NIGHT: 1.0,      # 深夜不加价（起步价已高）
        TimeSlot.MORNING_PEAK: 1.15,   # 早高峰 +15%
        TimeSlot.FORENOON: 1.0,        # 上午平峰
        TimeSlot.LUNCH_PEAK: 1.1,      # 午高峰 +10%
        TimeSlot.AFTERNOON: 1.0,       # 下午平峰
        TimeSlot.EVENING_PEAK: 1.2,    # 晚高峰 +20%
        TimeSlot.EVENING: 1.05,        # 晚间 +5%
        TimeSlot.NIGHT: 1.1,           # 夜间 +10%
    }

    # 天气加价因子映射
    WEATHER_FACTORS = {
        WeatherCondition.CLEAR: 1.0,
        WeatherCondition.CLOUDY: 1.0,
        WeatherCondition.LIGHT_RAIN: 1.1,
        WeatherCondition.HEAVY_RAIN: 1.3,
        WeatherCondition.STORM: 1.5,
        WeatherCondition.LIGHT_SNOW: 1.2,
        WeatherCondition.HEAVY_SNOW: 1.5,
        WeatherCondition.FOG: 1.4,
    }

    # 时段基础费率（元）——不同时段起步价不同
    TIME_SLOT_BASE_FARES = {
        TimeSlot.LATE_NIGHT: 14.0,    # 深夜起步14元
        TimeSlot.MORNING_PEAK: 13.0,  # 早高峰起步13元
        TimeSlot.FORENOON: 13.0,
        TimeSlot.LUNCH_PEAK: 13.0,
        TimeSlot.AFTERNOON: 13.0,
        TimeSlot.EVENING_PEAK: 13.0,
        TimeSlot.EVENING: 13.0,
        TimeSlot.NIGHT: 14.0,         # 夜间起步14元
    }

    def __init__(self, redis_client, db_client, weather_service):
        self.redis = redis_client
        self.db = db_client
        self.weather_service = weather_service
        # 区域加价状态缓存：grid_id → 上次加价倍数
        self._surge_cache: Dict[str, float] = {}

    @staticmethod
    def get_grid_id(lng: float, lat: float) -> str:
        """经纬度 → 网格 ID"""
        grid_x = int(lng / DynamicPricingEngine.GRID_LNG_STEP)
        grid_y = int(lat / DynamicPricingEngine.GRID_LAT_STEP)
        return f"g_{grid_x}_{grid_y}"

    @staticmethod
    def get_time_slot(hour: int, minute: int) -> TimeSlot:
        """当前时间 → 时段枚举"""
        t = hour * 60 + minute  # 转为分钟
        if t < 360:             # 00:00-06:00
            return TimeSlot.LATE_NIGHT
        elif t < 450:          # 06:00-07:30
            return TimeSlot.FORENOON
        elif t < 570:          # 07:30-09:30
            return TimeSlot.MORNING_PEAK
        elif t < 720:          # 09:30-12:00
            return TimeSlot.FORENOON
        elif t < 810:          # 12:00-13:30
            return TimeSlot.LUNCH_PEAK
        elif t < 1020:         # 13:30-17:00
            return TimeSlot.AFTERNOON
        elif t < 1170:         # 17:00-19:30
            return TimeSlot.EVENING_PEAK
        elif t < 1320:         # 19:30-22:00
            return TimeSlot.EVENING
        else:                  # 22:00-24:00
            return TimeSlot.NIGHT

    def get_grid_supply_demand(self, grid_id: str) -> GridSupplyDemand:
        """
        获取网格供需数据

        数据来源：
        - demand: Redis Sorted Set order:waiting:{cityId} 中该网格的等待订单数
        - supply: Redis GEO geo:drivers:{cityId} 中该网格的空闲司机数
        - 使用 5 分钟滑动窗口，避免瞬时抖动
        """
        stats_key = f"surge:grid:{grid_id}"
        stats = self.redis.hgetall(stats_key)

        if stats:
            return GridSupplyDemand(
                grid_id=grid_id,
                demand_count=int(stats.get(b"demand_count", 0)),
                supply_count=int(stats.get(b"supply_count", 0)),
                avg_wait_time_s=float(stats.get(b"avg_wait_s", 0)),
                demand_trend=float(stats.get(b"demand_trend", 0)),
                supply_trend=float(stats.get(b"supply_trend", 0)),
                timestamp=float(stats.get(b"timestamp", 0)),
            )

        return GridSupplyDemand(grid_id=grid_id, timestamp=time.time())

    def calculate_surge_multiplier(
        self,
        lng: float,
        lat: float,
        city_id: int,
        weather: Optional[WeatherCondition] = None,
    ) -> Tuple[float, str]:
        """
        计算动态加价倍数（完整流程）

        返回：(加价倍数, 加价等级描述)
        """
        grid_id = self.get_grid_id(lng, lat)

        # 第一步：获取网格供需数据
        sd = self.get_grid_supply_demand(grid_id)
        ratio = sd.demand_count / max(sd.supply_count, 1)

        # 第二步：供需比 → 基础加价倍数
        if ratio < 1.0:
            raw_surge = 1.0  # 供大于求，不加价
        else:
            raw_surge = ratio ** self.SUPPLY_DEMAND_ELASTICITY
            # ratio=1.5 → 1.22x, ratio=3.0 → 1.73x, ratio=5.0 → 2.24x

        # 第三步：叠加天气加价因子
        if weather is None:
            weather = self.weather_service.get_condition(city_id, lng, lat)
        weather_factor = self.WEATHER_FACTORS.get(weather, 1.0)
        raw_surge *= weather_factor

        # 第四步：叠加时段加价因子
        now = time.localtime()
        time_slot = self.get_time_slot(now.tm_hour, now.tm_min)
        time_slot_factor = self.TIME_SLOT_MULTIPLIERS.get(time_slot, 1.0)
        raw_surge *= time_slot_factor

        # 第五步：等待时间加成——平均等待超过 8 分钟额外加价
        if sd.avg_wait_time_s > 480:
            wait_bonus = 1.0 + (sd.avg_wait_time_s - 480) / 1200
            raw_surge *= min(wait_bonus, 1.5)

        # 第六步：趋势加成——需求快速上升时提前加价
        if sd.demand_trend > 0.3:  # 需求每分钟上升超过 30%
            trend_bonus = 1.0 + sd.demand_trend * 0.2
            raw_surge *= min(trend_bonus, 1.3)

        # 第七步：绝对上限保护
        raw_surge = min(raw_surge, self.MAX_SURGE_MULTIPLIER)

        # 第八步：平滑过渡——与上一次倍数做加权平均
        previous = self._surge_cache.get(grid_id, 1.0)
        smoothed = previous * self.SMOOTH_FACTOR + raw_surge * (1 - self.SMOOTH_FACTOR)

        # 第九步：单次变化幅度限制
        delta = smoothed - previous
        if abs(delta) > self.MAX_STEP_CHANGE:
            smoothed = previous + self.MAX_STEP_CHANGE * (1 if delta > 0 else -1)

        # 第十步：最终限制
        final = round(
            max(self.MIN_SURGE_MULTIPLIER, min(smoothed, self.MAX_SURGE_MULTIPLIER)),
            1
        )

        # 更新缓存
        self._surge_cache[grid_id] = final

        # 确定加价等级描述
        if final <= 1.0:
            level_desc = "正常价格"
        elif final <= 1.5:
            level_desc = "高峰期加价"
        elif final <= 2.0:
            level_desc = "供需紧张加价"
        else:
            level_desc = "严重紧张加价"

        return final, level_desc

    def calculate_full_price(
        self,
        lng: float,
        lat: float,
        city_id: int,
        car_type: int,
        distance_m: float,
        duration_s: int,
        tolls: float = 0.0,
    ) -> PricingResult:
        """
        完整定价计算——整合时段基础费率、供需加价、天气加价

        价格公式：
        total = (base_fare + distance_rate × distance + duration_rate × duration)
                × surge_multiplier + tolls

        其中 surge_multiplier = supply_demand_surge × weather_factor × time_slot_factor
        """
        # 获取时段
        now = time.localtime()
        time_slot = self.get_time_slot(now.tm_hour, now.tm_min)

        # 获取基础费率
        base_fare = self.TIME_SLOT_BASE_FARES.get(time_slot, 13.0)
        distance_rate = 0.0023   # 2.3元/km
        duration_rate = 0.0012   # 0.8元/min
        min_fare = base_fare

        # 获取天气
        weather = self.weather_service.get_condition(city_id, lng, lat)

        # 计算加价倍数
        surge_multiplier, surge_level = self.calculate_surge_multiplier(
            lng, lat, city_id, weather
        )

        # 各项费用
        distance_fee = round(distance_rate * distance_m, 2)
        duration_fee = round(duration_rate * duration_s, 2)
        subtotal = round(base_fare + distance_fee + duration_fee, 2)

        # 加价金额
        surge_amount = round(subtotal * (surge_multiplier - 1.0), 2) if surge_multiplier > 1.0 else 0.0

        # 总价
        total_fare = round(subtotal * surge_multiplier + tolls, 2)
        if total_fare < min_fare:
            total_fare = min_fare

        # 天气因子和时段因子（用于明细展示）
        weather_multiplier = self.WEATHER_FACTORS.get(weather, 1.0)
        time_slot_multiplier = self.TIME_SLOT_MULTIPLIERS.get(time_slot, 1.0)

        return PricingResult(
            base_fare=base_fare,
            distance_fee=distance_fee,
            duration_fee=duration_fee,
            subtotal=subtotal,
            surge_multiplier=surge_multiplier,
            surge_amount=surge_amount,
            weather_multiplier=weather_multiplier,
            time_slot_multiplier=time_slot_multiplier,
            tolls=tolls,
            total_fare=total_fare,
            is_surge=surge_multiplier > 1.0,
            surge_level=surge_level,
            breakdown={
                "time_slot": time_slot.value,
                "weather": weather.value,
                "grid_id": self.get_grid_id(lng, lat),
                "supply_demand_ratio": self.get_grid_supply_demand(
                    self.get_grid_id(lng, lat)
                ).demand_count / max(
                    self.get_grid_supply_demand(self.get_grid_id(lng, lat)).supply_count, 1
                ),
            }
        )

    def emergency_shutdown(self, grid_id: str = None):
        """紧急熔断——一键关闭指定网格或全部网格的加价"""
        if grid_id:
            self._surge_cache[grid_id] = 1.0
        else:
            for

## 派单策略引擎完整实现

```python
class DispatchStrategyEngine:
    """派单策略引擎：多策略动态选择"""

    STRATEGIES = {
        "nearest": {"description": "就近派单", "weight": 0.6},
        "rating": {"description": "好评优先", "weight": 0.2},
        "fairness": {"description": "公平轮转", "weight": 0.2},
    }

    def dispatch(self, order):
        """综合派单"""
        candidates = self._find_nearby_drivers(order, radius_km=3)

        if not candidates:
            # 无可用司机 → 扩大搜索范围
            candidates = self._find_nearby_drivers(order, radius_km=5)
        if not candidates:
            return {"status": "no_driver_available"}

        # 综合评分
        scored = []
        for driver in candidates:
            distance_score = self._score_distance(driver, order)
            rating_score = self._score_rating(driver)
            fairness_score = self._score_fairness(driver)

            total = (distance_score * 0.6 + rating_score * 0.2 + fairness_score * 0.2)
            scored.append({"driver": driver, "score": total})

        # 选择得分最高的司机
        best = max(scored, key=lambda x: x["score"])

        # 发送派单请求
        self.push_driver(best["driver"]["id"], {
            "order_id": order["id"],
            "pickup_location": order["pickup"],
            "estimated_ride_time": order["estimated_time"],
            "timeout_seconds": 15  # 15秒内必须响应
        })

        # 设置超时：司机不响应 → 重新派单
        self.schedule_timeout(order["id"], seconds=15)

        return {"driver_id": best["driver"]["id"], "score": best["score"]}

    def _score_fairness(self, driver):
        """公平性评分：等待时间越长分越高"""
        idle_minutes = (now() - driver["last_ride_completed_at"]).total_seconds() / 60
        # 等待 0 分钟=0分，等待 30 分钟=100分
        return min(100, idle_minutes / 30 * 100)

    def _score_rating(self, driver):
        """评分评分：5星=100分，3星=0分"""
        return (driver["rating"] - 3) / 2 * 100 if driver["rating"] >= 3 else 0

    def _score_distance(self, driver, order):
        """距离评分：0km=100分，3km=0分"""
        distance = self.haversine(driver["lat"], driver["lng"],
            order["pickup"]["lat"], order["pickup"]["lng"])
        return max(0, 100 - distance / 3 * 100)
```

## 订单超时与自动重派

```python
class DispatchTimeoutHandler:
    """派单超时处理"""

    def handle_timeout(self, order_id):
        """司机未响应 → 重新派单"""
        # 1. 取消原派单
        self.db.update("dispatch_records",
            {"status": "timeout"}, {"order_id": order_id})

        # 2. 重新派单（排除超时司机）
        order = self.db.get_order(order_id)
        excluded_drivers = self.db.query(
            "SELECT driver_id FROM dispatch_records "
            "WHERE order_id = %s AND status = 'timeout'", order_id)
        order["excluded_drivers"] = [d["driver_id"] for d in excluded_drivers]

        # 3. 降低超时司机的公平性评分
        for driver_id in order["excluded_drivers"]:
            self.redis.decr(f"driver_fairness:{driver_id}", 10)

        # 4. 重新派单
        result = self.strategy_engine.dispatch(order)
        return result
```

## 异常场景补充

### 场景：高峰期供需失衡

```
触发：暴雨天 → 出行需求暴增 3 倍 → 司机不足
检测：
  1. 等待派单订单 > 500 → 供需失衡告警
  2. 平均等待时间 > 10 分钟 → 严重告警
处理：
  1. 动态加价：价格倍数 = 1 + (等待订单数 / 可用司机数)
  2. 通知更多司机上线（奖励上线补贴）
  3. 推荐用户拼车/等待
预防：天气预警联动 + 动态定价 + 司机补贴池
```

### 场景：司机恶意刷单

```
触发：司机制造虚假订单 → 获取平台补贴
检测：
  1. 同一司机连续接到同一乘客 → 刷单嫌疑
  2. 乘客行程极短（< 1km）且频繁 → 异常
处理：
  1. AI 检测刷单模式 → 标记可疑司机
  2. 可疑司机 → 审查 + 暂停补贴
  3. 确认刷单 → 封号 + 追回补贴
预防：刷单检测模型 + 行程异常监控
```

## 司机评分与激励系统

```python
class DriverScoringService:
    """司机评分与激励系统"""

    def update_score(self, driver_id, trip_id):
        """行程结束后更新司机评分"""
        trip = self.db.get_trip(trip_id)
        rating = trip.get("rider_rating")

        if not rating:
            return

        # 1. 更新评分（指数移动平均）
        current = self.db.get_driver(driver_id)
        alpha = 0.1  # 新评分权重
        new_rating = alpha * rating + (1 - alpha) * current["rating"]
        self.db.update("drivers",
            {"rating": round(new_rating, 2), "total_trips": current["total_trips"] + 1},
            {"id": driver_id})

        # 2. 评分过低 → 警告
        if new_rating < 4.0:
            self.db.insert("driver_warnings", {
                "driver_id": driver_id,
                "type": "low_rating",
                "message": f"当前评分 {new_rating:.1f}，低于 4.0 标准",
                "created_at": now()
            })
        if new_rating < 3.5:
            # 暂停接单
            self.db.update("drivers", {"status": "suspended"}, {"id": driver_id})

    def calculate_incentive(self, driver_id, period="daily"):
        """计算司机激励"""
        if period == "daily":
            trips = self.db.count("trips", driver_id=driver_id,
                created_at__gte=today())
            targets = {8: 50, 12: 100, 15: 180}  # 单数 → 奖励金额
            for target, bonus in sorted(targets.items(), reverse=True):
                if trips >= target:
                    return {"trips": trips, "bonus": bonus, "target": target}
        return {"trips": trips, "bonus": 0}
```

## 乘客安全监控

```python
class RiderSafetyMonitor:
    """乘客安全监控"""

    def check_safety(self, trip_id):
        """行程安全检查"""
        trip = self.db.get_trip(trip_id)

        # 1. 路线偏移检测
        planned_route = json.loads(trip["planned_route"])
        current_location = self.get_current_location(trip["driver_id"])
        deviation = self._calc_route_deviation(current_location, planned_route)
        if deviation > 2000:  # 偏离 2km
            self.alert_safety(trip_id, "route_deviation", deviation)
            self.notify_rider(trip["rider_id"],
                "检测到行程偏离路线，是否安全？")

        # 2. 行程超时检测
        elapsed = (now() - trip["start_time"]).total_seconds() / 60
        estimated = trip["estimated_duration_minutes"]
        if elapsed > estimated * 2:
            self.alert_safety(trip_id, "trip_timeout",
                f"预计{estimated}分钟，已过{elapsed:.0f}分钟")

        # 3. 一键报警
        return {"safe": deviation <= 2000 and elapsed <= estimated * 2}

    def handle_emergency(self, trip_id, trigger="rider"):
        """紧急处理"""
        trip = self.db.get_trip(trip_id)
        # 1. 获取实时位置
        location = self.get_current_location(trip["driver_id"])
        # 2. 通知安全团队
        self.alert_safety(trip_id, "emergency", {
            "trigger": trigger, "location": location
        })
        # 3. 通知紧急联系人
        self.notify_emergency_contacts(trip["rider_id"], location)
        # 4. 启动录音（司机端和乘客端）
        self.start_audio_recording(trip_id)
```

## 异常场景补充

### 场景：派单不公平导致司机流失

```
触发：部分司机长期接不到好单 → 评分下降 → 离开平台
检测：
  1. 司机日均收入 < 城市最低标准 → 流失风险
  2. 司机在线时长 > 接单量 × 3 → 空驶率高
处理：
  1. 调整公平性权重：提高 _score_fairness 权重
  2. 保底收入补贴：收入不足时补差额
  3. 优先派单给低收入司机
预防：司机收入监控 + 公平调度算法 + 保底补贴
```

### 场景：紧急报警误触发

```
触发：乘客误触紧急报警按钮 → 安全团队介入
检测：
  1. 紧急报警触发 → 安全团队 30 秒内回拨
  2. 乘客确认误触 → 取消报警
处理：
  1. 回拨确认：30 秒内联系乘客
  2. 乘客确认安全 → 关闭报警
  3. 乘客未接听 → 升级处理（联系司机+报警）
预防：报警二次确认 + 30 秒回拨确认
```

## 拼车匹配算法完整实现

```python
class CarpoolMatchingService:
    """拼车匹配：顺路度计算 + 拼车定价"""

    def find_carpool(self, order):
        """为订单寻找拼车机会"""
        # 1. 查找路线重叠的进行中行程
        active_trips = self.db.query(
            "SELECT * FROM trips WHERE status = 'in_progress' "
            "AND remaining_seats > 0 AND carpool_enabled = 1")

        candidates = []
        for trip in active_trips:
            # 计算顺路度
            detour = self._calc_detour(trip, order)
            if detour["extra_minutes"] > 10:  # 绕行不超过 10 分钟
                continue

            # 计算拼车折扣
            discount = self._calc_carpool_discount(detour, trip, order)

            candidates.append({
                "trip_id": trip["id"],
                "driver_id": trip["driver_id"],
                "detour_minutes": detour["extra_minutes"],
                "discount": discount,
                "score": detour["extra_minutes"] * -1 + discount * 10  # 绕行少+折扣大=好
            })

        # 按分数排序
        candidates.sort(key=lambda x: x["score"], reverse=True)
        return candidates[:3]

    def _calc_detour(self, trip, order):
        """计算绕行距离和时间"""
        # 原路线距离
        original_distance = self.route_service.get_distance(
            trip["current_location"], trip["destination"])

        # 加入新乘客后的路线距离
        new_route = self.route_service.get_waypoints_route([
            trip["current_location"],
            order["pickup"],
            trip["destination"] if not trip.get("waypoints") else trip["waypoints"][-1],
        ])
        # 额外距离 = 新路线 - 原路线 - 原订单路线
        order_distance = self.route_service.get_distance(
            order["pickup"], order["dropoff"])

        extra_km = new_route["distance_km"] - original_distance - order_distance
        extra_minutes = extra_km / 30 * 60  # 假设平均 30km/h

        return {"extra_km": extra_km, "extra_minutes": extra_minutes}

    def _calc_carpool_discount(self, detour, trip, order):
        """计算拼车折扣：绕行越多折扣越大（补偿原乘客）"""
        base_discount = 0.15  # 基础 85 折
        detour_compensation = min(0.10, detour["extra_minutes"] * 0.01)
        return base_discount + detour_compensation
```

## 动态定价引擎

```python
class DynamicPricingEngine:
    """动态定价：供需 + 天气 + 时段"""

    def calculate_price(self, order):
        """计算动态价格"""
        # 1. 基础价格
        base_price = self._calc_base_price(order)

        # 2. 供需倍率
        supply_demand = self._get_supply_demand_multiplier(
            order["pickup"])
        # 3. 天气倍率
        weather = self._get_weather_multiplier(order["pickup"])
        # 4. 时段倍率
        time_multiplier = self._get_time_multiplier()

        total_multiplier = supply_demand * weather * time_multiplier
        # 封顶 3 倍
        total_multiplier = min(total_multiplier, 3.0)

        final_price = base_price * total_multiplier

        return {
            "base_price": base_price,
            "multiplier": round(total_multiplier, 2),
            "breakdown": {
                "supply_demand": supply_demand,
                "weather": weather,
                "time": time_multiplier
            },
            "final_price": round(final_price, 2)
        }

    def _get_supply_demand_multiplier(self, location):
        """供需倍率"""
        nearby_drivers = self.redis.georadius("driver_locations",
            location["lng"], location["lat"], 3, unit="km")
        waiting_orders = self.redis.georadius("waiting_orders",
            location["lng"], location["lat"], 3, unit="km")

        ratio = len(waiting_orders) / max(len(nearby_drivers), 1)
        if ratio > 5:
            return 2.5
        elif ratio > 3:
            return 2.0
        elif ratio > 2:
            return 1.5
        elif ratio > 1:
            return 1.2
        return 1.0
```

## 异常场景补充

### 场景：拼车乘客目的地变更

```
触发：拼车乘客 A 要求中途更改目的地 → 影响乘客 B 的路线
检测：
  1. 乘客发起目的地变更请求
  2. 系统重新计算路线和费用
处理：
  1. 计算新路线对其他乘客的影响
  2. 影响不超过 5 分钟 → 自动批准
  3. 影响超过 5 分钟 → 通知其他乘客确认
  4. 任何乘客拒绝 → 驳回变更
预防：目的地变更需系统评估 + 其他乘客确认
```

### 场景：动态定价引发用户投诉

```
触发：暴雨天 3 倍加价 → 用户投诉"趁火打劫"
检测：
  1. 加价倍率 > 2 倍 → 投诉风险高
  2. 投诉率 > 5% → 定价策略需调整
处理：
  1. 加价页面显示原因（供需紧张/恶劣天气）
  2. 显示附近等待人数，让用户理解
  3. 提供替代方案：拼车/等待价格回落
预防：透明定价 + 倍率封顶 + 用户教育
```

## 行程计费引擎完整实现

```python
class TripBillingEngine:
    """行程计费：基础费 + 里程费 + 时长费 + 附加费"""

    def calculate_fare(self, trip_id):
        """计算行程费用"""
        trip = self.db.get_trip(trip_id)
        city_config = self.db.get_city_config(trip["city_id"])

        # 1. 基础费（起步价）
        base_fare = city_config["base_fare"]

        # 2. 里程费
        distance_km = trip["distance_km"]
        included_km = city_config["base_included_km"]
        extra_km = max(0, distance_km - included_km)
        distance_fare = extra_km * city_config["per_km_rate"]

        # 3. 时长费（低速行驶补偿）
        duration_minutes = trip["duration_minutes"]
        low_speed_minutes = trip.get("low_speed_minutes", 0)
        duration_fare = low_speed_minutes * city_config["per_minute_rate"]

        # 4. 远途附加费（超过起终点直线距离一定倍数）
        straight_distance = self.haversine(
            trip["pickup_lat"], trip["pickup_lng"],
            trip["dropoff_lat"], trip["dropoff_lng"])
        if distance_km > straight_distance * city_config["detour_threshold"]:
            long_distance_fare = (distance_km - straight_distance *
                city_config["detour_threshold"]) * city_config["long_distance_rate"]
        else:
            long_distance_fare = 0

        # 5. 夜间附加费
        hour = trip["start_time"].hour
        if hour >= 23 or hour < 6:
            night_surcharge = city_config["night_surcharge"]
        else:
            night_surcharge = 0

        # 合计
        total = base_fare + distance_fare + duration_fare + long_distance_fare + night_surcharge
        total = round(total, 2)

        return {
            "base_fare": base_fare,
            "distance_fare": round(distance_fare, 2),
            "duration_fare": round(duration_fare, 2),
            "long_distance_fare": round(long_distance_fare, 2),
            "night_surcharge": night_surcharge,
            "total": total,
            "breakdown_km": round(distance_km, 2),
            "breakdown_minutes": duration_minutes
        }
```

## 司机收入结算

```python
class DriverSettlementService:
    """司机收入结算：日结 + 抽成 + 奖惩"""

    def daily_settle(self, driver_id, date):
        """日结司机收入"""
        trips = self.db.query(
            "SELECT * FROM trips WHERE driver_id = %s "
            "AND DATE(end_time) = %s AND status = 'completed'",
            driver_id, date)

        gross_income = sum(t["fare"] for t in trips)
        # 平台抽成
        commission_rate = self._get_commission_rate(driver_id)
        commission = gross_income * commission_rate

        # 奖励
        incentive = self.scoring.calculate_incentive(driver_id, "daily")

        # 罚款（取消订单）
        penalties = self._calc_penalties(driver_id, date)

        net_income = gross_income - commission + incentive["bonus"] - penalties

        self.db.insert("driver_settlements", {
            "driver_id": driver_id, "date": date,
            "trip_count": len(trips),
            "gross_income": gross_income,
            "commission_rate": commission_rate,
            "commission": round(commission, 2),
            "incentive": incentive["bonus"],
            "penalties": penalties,
            "net_income": round(net_income, 2),
            "settled_at": now()
        })

        return {"net_income": round(net_income, 2), "trip_count": len(trips)}

    def _get_commission_rate(self, driver_id):
        """获取抽成比例（根据等级递减）"""
        driver = self.db.get_driver(driver_id)
        tiers = {"bronze": 0.25, "silver": 0.20, "gold": 0.15, "platinum": 0.10}
        return tiers.get(driver.get("tier", "bronze"), 0.25)
```

## 异常场景补充

### 场景：计费争议处理

```
触发：乘客认为费用不合理 → 发起计费争议
检测：乘客提交争议 → 创建工单
处理：
  1. 调取行程轨迹 + 费用明细
  2. 重新计算费用（使用轨迹数据）
  3. 差异 < 1 元 → 维持原价
  4. 差异 > 1 元 → 退还差额 + 修正计费逻辑
预防：费用明细透明 + 行程轨迹可回溯 + 争议快速处理
```

### 场景：司机端 GPS 漂移影响计费

```
触发：司机 GPS 漂移 → 里程计算偏大 → 费用偏高
检测：
  1. 行程里程 > 预估里程 × 1.5 → 可能漂移
  2. GPS 轨迹出现跳跃点 → 漂移
处理：
  1. 过滤 GPS 漂移点后重新计算里程
  2. 以修正后里程计费
  3. 差额退还乘客
预防：GPS 轨迹滤波 + 里程异常检测 + 自动修正
```

## 司机入驻完整流程

```python
class DriverOnboardingService:
    """司机入驻：文档验证 → 背景审查 → 车辆检验 → 培训 → 等级分配 → 激活"""

    # ---- 1. 文档验证 ----

    DOCUMENT_REQUIREMENTS = {
        "id_card": {"name": "身份证", "sides": ["front", "back"], "ocr_required": True},
        "driver_license": {"name": "驾驶证", "sides": ["front"], "ocr_required": True},
        "vehicle_registration": {"name": "车辆登记证", "sides": ["front", "back"], "ocr_required": True},
    }

    def submit_documents(self, driver_id, doc_type, images):
        """提交司机证件材料"""
        if doc_type not in self.DOCUMENT_REQUIREMENTS:
            raise ValueError(f"不支持的文档类型: {doc_type}")

        requirement = self.DOCUMENT_REQUIREMENTS[doc_type]
        if len(images) < len(requirement["sides"]):
            return {"status": "rejected", "reason": f"{requirement['name']}需要{len(requirement['sides'])}张照片"}

        # OCR 识别
        ocr_results = []
        for img in images:
            ocr_result = self.ocr_service.recognize(img, doc_type)
            ocr_results.append(ocr_result)

        # 交叉验证：身份证姓名 vs 驾驶证姓名
        if doc_type == "driver_license" and self._has_id_card(driver_id):
            id_name = self.db.get_driver_doc(driver_id, "id_card")["ocr_data"]["name"]
            dl_name = ocr_results[0]["name"]
            if id_name != dl_name:
                return {"status": "rejected", "reason": "身份证与驾驶证姓名不一致"}

        # 交叉验证：车辆登记证车主 vs 司机姓名
        if doc_type == "vehicle_registration":
            id_name = self.db.get_driver_doc(driver_id, "id_card")["ocr_data"]["name"]
            vr_name = ocr_results[0]["owner_name"]
            if id_name != vr_name:
                return {"status": "rejected",
                        "reason": "车辆登记证车主与司机姓名不一致，需提供授权证明"}

        # 有效期检查
        if doc_type == "driver_license":
            expiry_date = ocr_results[0].get("expiry_date")
            if expiry_date and expiry_date < now().date() + timedelta(days=90):
                return {"status": "rejected",
                        "reason": f"驾驶证将于{expiry_date}到期，请先更换驾驶证"}

        # 存储文档
        doc_id = self.db.insert("driver_documents", {
            "driver_id": driver_id,
            "doc_type": doc_type,
            "ocr_data": ocr_results,
            "status": "verified",
            "submitted_at": now()
        })

        # 检查是否所有文档都已提交
        all_verified = self._check_all_documents_verified(driver_id)
        if all_verified:
            self._advance_stage(driver_id, "background_check")

        return {"status": "verified", "doc_id": doc_id}

    def _has_id_card(self, driver_id):
        """检查是否已提交身份证"""
        return self.db.exists("driver_documents",
            driver_id=driver_id, doc_type="id_card", status="verified")

    def _check_all_documents_verified(self, driver_id):
        """检查所有必需文档是否已通过验证"""
        for doc_type in self.DOCUMENT_REQUIREMENTS:
            if not self.db.exists("driver_documents",
                driver_id=driver_id, doc_type=doc_type, status="verified"):
                return False
        return True

    # ---- 2. 背景审查 ----

    def run_background_check(self, driver_id):
        """执行背景审查（对接公安/征信系统）"""
        driver_docs = self.db.query(
            "SELECT * FROM driver_documents WHERE driver_id = %s AND status = 'verified'",
            driver_id)

        id_doc = next((d for d in driver_docs if d["doc_type"] == "id_card"), None)
        if not id_doc:
            return {"status": "failed", "reason": "缺少身份证信息"}

        id_number = id_doc["ocr_data"][0]["id_number"]
        name = id_doc["ocr_data"][0]["name"]

        # 并行调用多个审查接口
        results = {}
        with ThreadPoolExecutor(max_workers=3) as executor:
            futures = {
                executor.submit(self._check_criminal_record, id_number): "criminal",
                executor.submit(self._check_credit_record, id_number): "credit",
                executor.submit(self._check_driving_violations, id_number): "traffic",
            }
            for future in as_completed(futures):
                check_type = futures[future]
                try:
                    results[check_type] = future.result(timeout=30)
                except Exception as e:
                    results[check_type] = {"status": "error", "message": str(e)}

        # 审查结果判定
        disqualify_reasons = []
        if results.get("criminal", {}).get("status") == "disqualified":
            disqualify_reasons.append("有刑事犯罪记录")
        if results.get("credit", {}).get("status") == "disqualified":
            disqualify_reasons.append("严重失信记录")
        if results.get("traffic", {}).get("status") == "disqualified":
            disqualify_reasons.append("重大交通违章记录")

        if disqualify_reasons:
            self.db.update("drivers", driver_id, {
                "onboarding_status": "rejected",
                "reject_reason": "; ".join(disqualify_reasons),
                "rejected_at": now()
            })
            return {"status": "rejected", "reasons": disqualify_reasons}

        # 通过审查
        self.db.update("drivers", driver_id, {
            "background_check_status": "passed",
            "background_check_date": now()
        })
        self._advance_stage(driver_id, "vehicle_inspection")
        return {"status": "passed", "checks": results}

    def _check_criminal_record(self, id_number):
        """刑事犯罪记录查询（对接公安接口）"""
        return self.criminal_api.query(id_number)

    def _check_credit_record(self, id_number):
        """征信记录查询（对接人行征信接口）"""
        return self.credit_api.query(id_number)

    def _check_driving_violations(self, id_number):
        """交通违章记录查询（对接交管接口）"""
        return self.traffic_api.query(id_number)

    # ---- 3. 车辆检验 ----

    def schedule_vehicle_inspection(self, driver_id, preferred_date=None):
        """预约车辆检验"""
        driver = self.db.get_driver(driver_id)
        if driver["onboarding_stage"] != "vehicle_inspection":
            return {"status": "error", "reason": "当前阶段不支持预约车辆检验"}

        # 查找最近的检验中心
        driver_location = driver.get("address_geo")
        centers = self.db.query(
            "SELECT * FROM inspection_centers WHERE city = %s AND status = 'active'",
            driver["city"])

        if not centers:
            return {"status": "error", "reason": "所在城市暂无检验中心"}

        # 按距离排序
        centers.sort(key=lambda c: self.haversine(
            driver_location["lat"], driver_location["lng"],
            c["lat"], c["lng"]))

        # 检查可用时段
        target_date = preferred_date or now().date() + timedelta(days=1)
        for center in centers[:3]:
            slots = self.db.query(
                "SELECT * FROM inspection_slots WHERE center_id = %s "
                "AND slot_date = %s AND booked_count < max_count",
                center["center_id"], target_date)
            if slots:
                # 预约第一个可用时段
                slot = slots[0]
                self.db.update("inspection_slots", slot["slot_id"],
                    {"booked_count": slot["booked_count"] + 1})
                self.db.insert("inspection_bookings", {
                    "driver_id": driver_id,
                    "center_id": center["center_id"],
                    "slot_id": slot["slot_id"],
                    "slot_date": target_date,
                    "slot_time": slot["start_time"],
                    "status": "booked",
                    "created_at": now()
                })
                return {
                    "status": "booked",
                    "center": center["name"],
                    "address": center["address"],
                    "date": str(target_date),
                    "time": slot["start_time"]
                }

        return {"status": "error", "reason": "近期无可用检验时段"}

    def complete_vehicle_inspection(self, booking_id, inspection_results):
        """完成车辆检验"""
        booking = self.db.get_inspection_booking(booking_id)
        driver_id = booking["driver_id"]

        # 检验项目：外观、灯光、轮胎、刹车、安全带、灭火器
        required_items = ["exterior", "lights", "tires", "brakes", "seatbelt", "extinguisher"]
        failed_items = [item for item in required_items
                        if inspection_results.get(item) != "pass"]

        if failed_items:
            self.db.update("inspection_bookings", booking_id, {
                "status": "failed",
                "failed_items": failed_items,
                "inspection_results": inspection_results,
                "completed_at": now()
            })
            return {"status": "failed", "failed_items": failed_items}

        # 通过检验
        self.db.update("inspection_bookings", booking_id, {
            "status": "passed",
            "inspection_results": inspection_results,
            "completed_at": now()
        })
        self.db.update("drivers", driver_id, {
            "vehicle_inspection_status": "passed"
        })
        self._advance_stage(driver_id, "training")
        return {"status": "passed"}

    # ---- 4. 培训完成跟踪 ----

    TRAINING_MODULES = [
        {"id": "safety", "name": "安全驾驶培训", "duration_min": 60, "required": True},
        {"id": "service", "name": "服务规范培训", "duration_min": 45, "required": True},
        {"id": "app_usage", "name": "APP 使用培训", "duration_min": 30, "required": True},
        {"id": "emergency", "name": "应急处理培训", "duration_min": 30, "required": True},
        {"id": "local_rules", "name": "当地法规培训", "duration_min": 30, "required": True},
    ]

    def start_training(self, driver_id, module_id):
        """开始培训模块"""
        module = next((m for m in self.TRAINING_MODULES if m["id"] == module_id), None)
        if not module:
            return {"status": "error", "reason": "培训模块不存在"}

        # 检查是否已完成
        existing = self.db.query_one(
            "SELECT * FROM driver_training WHERE driver_id = %s AND module_id = %s",
            driver_id, module_id)
        if existing and existing["status"] == "passed":
            return {"status": "already_completed"}

        self.db.insert("driver_training", {
            "driver_id": driver_id,
            "module_id": module_id,
            "module_name": module["name"],
            "status": "in_progress",
            "started_at": now()
        })
        return {"status": "started", "module": module["name"], "duration_min": module["duration_min"]}

    def complete_training(self, driver_id, module_id, quiz_score):
        """完成培训模块（需通过测验）"""
        passing_score = 80  # 80 分及格

        if quiz_score < passing_score:
            self.db.update("driver_training",
                {"driver_id": driver_id, "module_id": module_id},
                {"status": "quiz_failed", "quiz_score": quiz_score, "updated_at": now()})
            return {"status": "failed", "reason": f"测验得分{quiz_score}，未达{passing_score}分"}

        self.db.update("driver_training",
            {"driver_id": driver_id, "module_id": module_id},
            {"status": "passed", "quiz_score": quiz_score, "completed_at": now()})

        # 检查所有必修模块是否已完成
        all_passed = self._check_all_training_passed(driver_id)
        if all_passed:
            self._advance_stage(driver_id, "tier_assignment")
            return {"status": "all_completed", "next": "tier_assignment"}

        return {"status": "module_completed"}

    def _check_all_training_passed(self, driver_id):
        """检查所有必修培训是否通过"""
        required_ids = [m["id"] for m in self.TRAINING_MODULES if m["required"]]
        completed = self.db.query(
            "SELECT module_id FROM driver_training "
            "WHERE driver_id = %s AND status = 'passed'", driver_id)
        completed_ids = {c["module_id"] for c in completed}
        return required_ids.issubset(completed_ids)

    # ---- 5. 等级分配（基于车辆类型） ----

    VEHICLE_TIER_RULES = {
        "economy": {"tier": "bronze", "service_types": ["economy"]},
        "comfort": {"tier": "silver", "service_types": ["economy", "comfort"]},
        "premium": {"tier": "gold", "service_types": ["economy", "comfort", "premium"]},
        "luxury": {"tier": "platinum", "service_types": ["economy", "comfort", "premium", "luxury"]},
        "suv": {"tier": "gold", "service_types": ["economy", "comfort", "suv"]},
        "van": {"tier": "silver", "service_types": ["economy", "van"]},
    }

    def assign_tier(self, driver_id):
        """根据车辆类型分配司机等级"""
        vehicle = self.db.get_driver_vehicle(driver_id)
        vehicle_type = vehicle["vehicle_type"]  # economy/comfort/premium/luxury/suv/van

        rule = self.VEHICLE_TIER_RULES.get(vehicle_type, self.VEHICLE_TIER_RULES["economy"])

        # 检查车辆年限（超过 5 年降一级）
        vehicle_age = (now().date() - vehicle["registration_date"]).days / 365
        tier = rule["tier"]
        tier_order = ["bronze", "silver", "gold", "platinum"]
        if vehicle_age > 5 and tier != "bronze":
            idx = tier_order.index(tier)
            tier = tier_order[idx - 1]

        self.db.update("drivers", driver_id, {
            "tier": tier,
            "service_types": rule["service_types"],
            "tier_assigned_at": now()
        })

        self._advance_stage(driver_id, "activation")
        return {"tier": tier, "service_types": rule["service_types"]}

    # ---- 6. 激活流程 ----

    def activate_driver(self, driver_id):
        """激活司机（入驻流程最后一步）"""
        driver = self.db.get_driver(driver_id)

        # 最终检查清单
        checks = {
            "documents": self._check_all_documents_verified(driver_id),
            "background": driver.get("background_check_status") == "passed",
            "inspection": driver.get("vehicle_inspection_status") == "passed",
            "training": self._check_all_training_passed(driver_id),
            "tier": driver.get("tier") is not None,
        }

        failed_checks = [k for k, v in checks.items() if not v]
        if failed_checks:
            return {"status": "blocked", "failed_checks": failed_checks}

        # 激活
        self.db.update("drivers", driver_id, {
            "onboarding_status": "active",
            "activated_at": now(),
            "status": "offline"  # 激活后默认离线，司机自行上线
        })

        # 初始化司机画像
        self.db.insert("driver_profiles", {
            "driver_id": driver_id,
            "total_trips": 0,
            "rating": 5.0,
            "acceptance_rate": 1.0,
            "cancellation_rate": 0.0,
            "created_at": now()
        })

        # 发送激活通知
        self.notification_service.send(driver_id, "onboarding_activated", {
            "tier": driver["tier"],
            "service_types": driver["service_types"]
        })

        return {"status": "activated", "tier": driver["tier"]}

    def _advance_stage(self, driver_id, next_stage):
        """推进入驻阶段"""
        self.db.update("drivers", driver_id, {
            "onboarding_stage": next_stage,
            "stage_updated_at": now()
        })
```

## 拼车优化完整实现

```python
class RideSharingOptimizer:
    """拼车优化：匹配算法 + 分段计费 + 乘客体验 + 司机补偿"""

    # ---- 1. 拼车匹配算法 ----

    def find_shared_ride_match(self, order_id):
        """为订单寻找拼车匹配"""
        order = self.db.get_order(order_id)
        pickup = (order["pickup_lat"], order["pickup_lng"])
        dropoff = (order["dropoff_lat"], order["dropoff_lng"])

        # 查找进行中的拼车行程（最多拼 2 人）
        active_trips = self.db.query(
            "SELECT * FROM trips WHERE is_shared = TRUE AND status = 'in_progress' "
            "AND passenger_count < 2 AND city_id = %s", order["city_id"])

        best_match = None
        best_detour_minutes = float('inf')

        for trip in active_trips:
            # 计算新乘客的上下车顺序优化
            sequence = self._optimize_pickup_dropoff_sequence(trip, order)
            if not sequence:
                continue

            # 计算绕行时间（对已有乘客的影响）
            original_eta = self._calculate_route_time(
                [(trip["pickup_lat"], trip["pickup_lng"]),
                 (trip["dropoff_lat"], trip["dropoff_lng"])])
            new_eta = self._calculate_route_time(sequence["waypoints"])

            detour_minutes = (new_eta - original_eta) / 60
            if detour_minutes < best_detour_minutes and detour_minutes <= 10:
                best_detour_minutes = detour_minutes
                best_match = {
                    "trip": trip,
                    "sequence": sequence,
                    "detour_minutes": round(detour_minutes, 1)
                }

        if best_match:
            return {"matched": True, "match": best_match}
        return {"matched": False}

    def _optimize_pickup_dropoff_sequence(self, existing_trip, new_order):
        """优化上下车顺序（穷举所有排列，选最短路径）"""
        # 所有可能的上下车顺序
        # A=已有乘客上车点, B=已有乘客下车点, C=新乘客上车点, D=新乘客下车点
        candidates = [
            ["A", "C", "B", "D"],  # 先接新乘客，先送旧乘客
            ["A", "C", "D", "B"],  # 先接新乘客，先送新乘客
            ["C", "A", "B", "D"],  # 先接新乘客，再接旧乘客
            ["A", "B", "C", "D"],  # 先送旧乘客，再接新乘客
        ]

        points = {
            "A": (existing_trip["pickup_lat"], existing_trip["pickup_lng"]),
            "B": (existing_trip["dropoff_lat"], existing_trip["dropoff_lng"]),
            "C": (new_order["pickup_lat"], new_order["pickup_lng"]),
            "D": (new_order["dropoff_lat"], new_order["dropoff_lng"]),
        }

        # 过滤不合理顺序：上车点必须在其下车点之前
        valid_sequences = []
        for seq in candidates:
            a_idx = seq.index("A")
            b_idx = seq.index("B")
            c_idx = seq.index("C")
            d_idx = seq.index("D")
            if a_idx < b_idx and c_idx < d_idx:
                valid_sequences.append(seq)

        # 选择总行驶时间最短的顺序
        best_seq = None
        best_time = float('inf')
        for seq in valid_sequences:
            waypoints = [points[p] for p in seq]
            route_time = self._calculate_route_time(waypoints)
            if route_time < best_time:
                best_time = route_time
                best_seq = seq

        if best_seq:
            return {"sequence": best_seq, "waypoints": [points[p] for p in best_seq]}
        return None

    def _calculate_route_time(self, waypoints):
        """调用地图API计算路线时间（秒）"""
        return self.map_service.route_time(waypoints)

    # ---- 2. 拼车分段计费 ----

    def calculate_shared_fare(self, trip_id):
        """拼车分段计费：按里程比例分摊"""
        trip = self.db.get_trip(trip_id)
        passengers = self.db.query(
            "SELECT * FROM trip_passengers WHERE trip_id = %s ORDER BY pickup_order",
            trip_id)

        if len(passengers) == 1:
            return {passengers[0]["passenger_id"]: self._solo_fare(trip)}

        # 计算各乘客的独享费用
        solo_fares = {}
        for p in passengers:
            solo_fares[p["passenger_id"]] = self._calculate_solo_fare(
                (p["pickup_lat"], p["pickup_lng"]),
                (p["dropoff_lat"], p["dropoff_lng"]),
                trip["city_id"])

        # 计算总行程费用
        total_route_distance = trip["distance_km"]
        total_solo = sum(solo_fares.values())

        # 按里程比例分摊总费用
        shared_fares = {}
        for p in passengers:
            solo_dist = self.haversine(
                p["pickup_lat"], p["pickup_lng"],
                p["dropoff_lat"], p["dropoff_lng"])
            ratio = solo_dist / sum(
                self.haversine(pp["pickup_lat"], pp["pickup_lng"],
                               pp["dropoff_lat"], pp["dropoff_lng"])
                for pp in passengers)

            # 拼车折扣（共享部分享受 7 折）
            solo_fare = solo_fares[p["passenger_id"]]
            discount = 0.7  # 拼车折扣
            shared_fare = solo_fare * discount

            # 保证不超过独享费用
            shared_fares[p["passenger_id"]] = min(shared_fare, solo_fare)

        return shared_fares

    def _calculate_solo_fare(self, pickup, dropoff, city_id):
        """计算独享费用"""
        distance = self.haversine(pickup[0], pickup[1], dropoff[0], dropoff[1])
        city_config = self.db.get_city_config(city_id)
        base = city_config["base_fare"]
        extra_km = max(0, distance - city_config["base_included_km"])
        return base + extra_km * city_config["per_km_rate"]

    # ---- 3. 乘客体验（ETA 更新 + 隐私保护） ----

    def update_shared_ride_eta(self, trip_id):
        """实时更新拼车乘客的ETA"""
        trip = self.db.get_trip(trip_id)
        passengers = self.db.query(
            "SELECT * FROM trip_passengers WHERE trip_id = %s "
            "AND dropoff_time IS NULL", trip_id)

        current_location = self.gps_tracker.get_location(trip["driver_id"])

        for p in sorted(passengers, key=lambda x: x["pickup_order"]):
            # 计算到该乘客下车点的ETA
            eta_seconds = self.map_service.route_time([
                (current_location["lat"], current_location["lng"]),
                (p["dropoff_lat"], p["dropoff_lng"])
            ])
            eta_minutes = round(eta_seconds / 60, 1)

            # 推送ETA更新
            self.push_service.send(p["passenger_id"], "eta_update", {
                "eta_minutes": eta_minutes,
                "current_status": self._get_passenger_status(p, current_location)
            })

    def _get_passenger_status(self, passenger, driver_location):
        """获取乘客当前状态（上车前/车上/即将到达）"""
        if not passenger.get("pickup_time"):
            dist = self.haversine(
                driver_location["lat"], driver_location["lng"],
                passenger["pickup_lat"], passenger["pickup_lng"])
            return f"司机距您{round(dist, 1)}公里，正在前来接您"
        else:
            dist = self.haversine(
                driver_location["lat"], driver_location["lng"],
                passenger["dropoff_lat"], passenger["dropoff_lng"])
            if dist < 0.5:
                return "即将到达您的目的地"
            return "行程中"

    def get_co_rider_info(self, passenger_id, trip_id):
        """获取同乘乘客信息（隐私保护版）"""
        co_riders = self.db.query(
            "SELECT * FROM trip_passengers WHERE trip_id = %s "
            "AND passenger_id != %s", trip_id, passenger_id)

        # 隐私处理：只显示脱敏信息
        safe_info = []
        for cr in co_riders:
            safe_info.append({
                "pickup_area": self._mask_location(cr["pickup_lat"], cr["pickup_lng"]),
                "dropoff_area": self._mask_location(cr["dropoff_lat"], cr["dropoff_lng"]),
                "passenger_count": cr.get("passenger_count", 1),
                # 不显示：姓名、手机号、精确地址
            })
        return safe_info

    def _mask_location(self, lat, lng):
        """位置脱敏：只保留区域名称，不显示精确地址"""
        return self.geocode_service.reverse_geocode(lat, lng, level="district")

    # ---- 4. 司机拼车补偿 ----

    def calculate_driver_shared_compensation(self, trip_id):
        """计算司机拼车补偿（额外停靠补贴）"""
        trip = self.db.get_trip(trip_id)
        passengers = self.db.query(
            "SELECT * FROM trip_passengers WHERE trip_id = %s", trip_id)

        if len(passengers) <= 1:
            return {"compensation": 0, "reason": "非拼车行程"}

        # 额外停靠次数 = 上下车总次数 - 2（起点和终点各一次）
        total_stops = len(passengers) * 2  # 每个乘客有上车和下车
        extra_stops = total_stops - 2

        # 每次额外停靠补贴
        stop_compensation = extra_stops * 3.0  # 每次额外停靠补 3 元

        # 绕行补贴：如果行程总距离超过最远乘客独享距离的 1.2 倍
        max_solo_distance = max(
            self.haversine(p["pickup_lat"], p["pickup_lng"],
                           p["dropoff_lat"], p["dropoff_lng"])
            for p in passengers)
        if trip["distance_km"] > max_solo_distance * 1.2:
            detour_km = trip["distance_km"] - max_solo_distance * 1.2
            detour_compensation = detour_km * 1.5  # 绕行每公里补 1.5 元
        else:
            detour_compensation = 0

        total_compensation = round(stop_compensation + detour_compensation, 2)

        return {
            "extra_stops": extra_stops,
            "stop_compensation": round(stop_compensation, 2),
            "detour_km": round(max(0, trip["distance_km"] - max_solo_distance * 1.2), 1),
            "detour_compensation": round(detour_compensation, 2),
            "total_compensation": total_compensation
        }
```

## 城市运营仪表盘

```python
class CityOperationsDashboard:
    """城市运营仪表盘：供需热力图 + 司机分布 + 动态定价 + ETA准确率 + 事件追踪"""

    # ---- 1. 实时供需热力图 ----

    def get_supply_demand_heatmap(self, city_id):
        """生成城市实时供需热力图"""
        # 将城市划分为网格（1km x 1km）
        grids = self._get_city_grids(city_id)

        heatmap_data = []
        for grid in grids:
            # 统计网格内司机数
            available_drivers = self.redis.georadius(
                "driver_locations",
                grid["center_lng"], grid["center_lat"],
                0.7, unit="km")  # 0.7km ≈ 1km 网格对角线的一半

            # 统计网格内等待订单数
            waiting_orders = self.redis.georadius(
                "waiting_orders",
                grid["center_lng"], grid["center_lat"],
                0.7, unit="km")

            supply = len(available_drivers)
            demand = len(waiting_orders)
            ratio = demand / max(supply, 1)

            # 热力等级
            if ratio > 5:
                level = "critical"  # 严重缺车
            elif ratio > 3:
                level = "high"      # 供不应求
            elif ratio > 1:
                level = "moderate"  # 轻度紧张
            else:
                level = "balanced"  # 供需平衡

            heatmap_data.append({
                "grid_id": grid["grid_id"],
                "center_lat": grid["center_lat"],
                "center_lng": grid["center_lng"],
                "supply": supply,
                "demand": demand,
                "ratio": round(ratio, 2),
                "level": level
            })

        return heatmap_data

    def _get_city_grids(self, city_id):
        """获取城市网格划分"""
        city = self.db.get_city(city_id)
        bounds = city["bounds"]  # {min_lat, max_lat, min_lng, max_lng}

        grids = []
        step = 0.01  # 约 1km
        lat = bounds["min_lat"]
        grid_id = 0
        while lat < bounds["max_lat"]:
            lng = bounds["min_lng"]
            while lng < bounds["max_lng"]:
                grids.append({
                    "grid_id": f"G{grid_id:06d}",
                    "center_lat": lat + step / 2,
                    "center_lng": lng + step / 2
                })
                lng += step
                grid_id += 1
            lat += step
        return grids

    # ---- 2. 司机分布（按区域） ----

    def get_driver_distribution(self, city_id):
        """按区域统计司机分布"""
        zones = self.db.query(
            "SELECT * FROM city_zones WHERE city_id = %s", city_id)

        distribution = []
        for zone in zones:
            # 统计区域内各状态司机数
            drivers_in_zone = self.redis.georadius(
                "driver_locations",
                zone["center_lng"], zone["center_lat"],
                zone["radius_km"], unit="km")

            # 按状态分组
            online_count = 0
            offline_count = 0
            in_trip_count = 0
            for driver_key in drivers_in_zone:
                driver_id = driver_key.decode() if isinstance(driver_key, bytes) else driver_key
                status = self.redis.hget(f"driver:{driver_id}", "status")
                if status == "online":
                    online_count += 1
                elif status == "in_trip":
                    in_trip_count += 1
                else:
                    offline_count += 1

            distribution.append({
                "zone_id": zone["zone_id"],
                "zone_name": zone["name"],
                "total": len(drivers_in_zone),
                "online": online_count,
                "in_trip": in_trip_count,
                "offline": offline_count,
                "utilization_rate": round(in_trip_count / max(online_count + in_trip_count, 1), 3)
            })

        return sorted(distribution, key=lambda x: x["total"], reverse=True)

    # ---- 3. 动态定价区域 ----

    def get_surge_zones(self, city_id):
        """获取当前动态定价区域"""
        heatmap = self.get_supply_demand_heatmap(city_id)

        surge_zones = []
        for grid in heatmap:
            if grid["level"] in ["high", "critical"]:
                surge_multiplier = self._calculate_surge_multiplier(grid["ratio"])
                surge_zones.append({
                    "grid_id": grid["grid_id"],
                    "center_lat": grid["center_lat"],
                    "center_lng": grid["center_lng"],
                    "supply": grid["supply"],
                    "demand": grid["demand"],
                    "surge_multiplier": surge_multiplier,
                    "estimated_duration_minutes": self._estimate_surge_duration(grid)
                })

        return surge_zones

    def _calculate_surge_multiplier(self, ratio):
        """计算加价倍率"""
        if ratio > 5:
            return 2.5
        elif ratio > 3:
            return 2.0
        elif ratio > 2:
            return 1.5
        elif ratio > 1:
            return 1.2
        return 1.0

    def _estimate_surge_duration(self, grid):
        """预估加价持续时间（分钟）"""
        # 基于历史数据：供需比越高，持续时间越长
        if grid["level"] == "critical":
            return 45
        elif grid["level"] == "high":
            return 30
        return 15

    # ---- 4. ETA 预测准确率 ----

    def get_eta_accuracy(self, city_id, period="daily"):
        """ETA 预测准确率统计"""
        if period == "daily":
            since = now() - timedelta(days=1)
        else:
            since = now() - timedelta(days=7)

        # 获取已完成行程的 ETA 数据
        trips = self.db.query(
            "SELECT * FROM trips WHERE city_id = %s AND status = 'completed' "
            "AND end_time >= %s AND predicted_eta IS NOT NULL "
            "AND actual_arrival_time IS NOT NULL",
            city_id, since)

        if not trips:
            return {"accuracy": None, "sample_size": 0}

        # 计算偏差
        deviations = []
        for trip in trips:
            predicted = trip["predicted_eta"]  # 预计到达分钟
            actual = (trip["actual_arrival_time"] - trip["created_at"]).total_seconds() / 60
            deviation = abs(predicted - actual)
            deviations.append(deviation)

        avg_deviation = sum(deviations) / len(deviations)
        within_3min = sum(1 for d in deviations if d <= 3) / len(deviations)
        within_5min = sum(1 for d in deviations if d <= 5) / len(deviations)

        return {
            "sample_size": len(trips),
            "avg_deviation_minutes": round(avg_deviation, 1),
            "within_3min_rate": round(within_3min, 3),
            "within_5min_rate": round(within_5min, 3),
            "max_deviation_minutes": round(max(deviations), 1)
        }

    # ---- 5. 事件追踪 ----

    def track_incidents(self, city_id, incident_type=None, since=None):
        """事件追踪（事故/投诉/违章）"""
        since = since or now() - timedelta(days=7)

        conditions = ["city_id = %s", "created_at >= %s"]
        params = [city_id, since]

        if incident_type:
            conditions.append("incident_type = %s")
            params.append(incident_type)

        incidents = self.db.query(
            f"SELECT * FROM incidents WHERE {' AND '.join(conditions)} "
            "ORDER BY created_at DESC", *params)

        # 按类型统计
        by_type = {}
        for inc in incidents:
            t = inc["incident_type"]
            by_type.setdefault(t, {"count": 0, "unresolved": 0})
            by_type[t]["count"] += 1
            if inc["status"] != "resolved":
                by_type[t]["unresolved"] += 1

        # 按严重等级统计
        by_severity = {}
        for inc in incidents:
            s = inc["severity"]
            by_severity[s] = by_severity.get(s, 0) + 1

        return {
            "total": len(incidents),
            "by_type": by_type,
            "by_severity": by_severity,
            "recent": incidents[:20]
        }

    # ---- 6. 每日运营汇总 ----

    def get_daily_summary(self, city_id, date=None):
        """每日运营汇总"""
        date = date or now().date()

        # 核心指标
        trips = self.db.query(
            "SELECT * FROM trips WHERE city_id = %s AND DATE(created_at) = %s "
            "AND status = 'completed'", city_id, date)

        total_trips = len(trips)
        total_revenue = sum(t["fare"] for t in trips) if trips else 0
        avg_wait = sum(t["wait_minutes"] for t in trips) / max(total_trips, 1)
        avg_fare = total_revenue / max(total_trips, 1)

        # 按时段统计
        hourly = [0] * 24
        for t in trips:
            hourly[t["created_at"].hour] += 1

        # 峰值时段
        peak_hour = hourly.index(max(hourly))

        return {
            "city_id": city_id,
            "date": str(date),
            "total_trips": total_trips,
            "total_revenue": round(total_revenue, 2),
            "avg_wait_minutes": round(avg_wait, 1),
            "avg_fare": round(avg_fare, 2),
            "peak_hour": f"{peak_hour}:00",
            "hourly_distribution": hourly,
            "completion_rate": round(total_trips / max(
                self.db.query_one(
                    "SELECT COUNT(*) as total FROM trips WHERE city_id = %s "
                    "AND DATE(created_at) = %s", city_id, date)["total"], 1), 3)
        }
```

## 异常场景补充

### 场景：司机证件过期阻断行程

```
触发：司机驾驶证/车辆年检已过期 → 仍在接单
检测：
  1. 每日凌晨批量检查司机证件有效期
  2. 证件到期前 30 天 → 推送提醒
  3. 证件到期前 7 天 → 强制弹窗提醒
  4. 证件已过期 → 自动下线
处理：
  1. 证件过期 → 立即下线司机（无法上线接单）
  2. 进行中的行程 → 允许完成，完成后强制下线
  3. 司机更新证件后 → OCR 验证 → 重新激活
预防：证件到期自动检查 + 分级提醒 + 过期自动下线
```

### 场景：动态定价被司机滥用

```
触发：部分司机联合制造供需紧张假象 → 触发动态加价 → 获取更高收入
检测：
  1. 同一区域多名司机同时下线/上线 → 可疑模式
  2. 加价时段司机拒单率异常升高 → 可疑
  3. 司机群体行为分析：同一区域 > 3 名司机同时拒绝平价单
处理：
  1. 识别到滥用模式 → 冻结相关司机的动态定价收益
  2. 暂停该区域的动态定价 30 分钟
  3. 对参与滥用的司机：警告 → 扣除不当收益 → 封号
  4. 优化定价算法：增加历史基线对比，异常波动需人工确认
预防：司机行为模式分析 + 定价异常检测 + 人工审核机制

## 司机激励与动态定价完整实现

```python
class DynamicPricingService:
    """动态定价：供需失衡 + 时段 + 天气 + 竞品"""

    SURGE_FACTORS = {
        "supply_demand_ratio": {"weight": 0.4, "threshold": 0.7},
        "time_of_day": {"weight": 0.2},
        "weather": {"weight": 0.2},
        "event": {"weight": 0.2},
    }

    def calculate_surge(self, origin_lat, origin_lng, destination=None):
        """计算动态加价倍率"""
        # 1. 供需比
        nearby_drivers = self._count_nearby_drivers(origin_lat, origin_lng, radius_km=3)
        pending_requests = self._count_pending_requests(origin_lat, origin_lng, radius_km=3)
        supply_demand = nearby_drivers / max(pending_requests, 1)

        if supply_demand < 0.5:
            sd_multiplier = 2.5
        elif supply_demand < 0.7:
            sd_multiplier = 1.8
        elif supply_demand < 1.0:
            sd_multiplier = 1.3
        else:
            sd_multiplier = 1.0

        # 2. 时段因子
        hour = now().hour
        if 7 <= hour <= 9 or 17 <= hour <= 19:
            time_multiplier = 1.2  # 高峰
        elif 23 <= hour or hour <= 5:
            time_multiplier = 1.1  # 深夜
        else:
            time_multiplier = 1.0

        # 3. 天气因子
        weather = self.weather_api.get_current(origin_lat, origin_lng)
        weather_multiplier = 1.0
        if weather.get("condition") in ["rain", "snow"]:
            weather_multiplier = 1.3
        elif weather.get("condition") == "heavy_rain":
            weather_multiplier = 1.5

        # 4. 大型活动
        event_multiplier = 1.0
        events = self._check_nearby_events(origin_lat, origin_lng)
        if events:
            event_multiplier = 1.4

        # 综合倍率（加权平均 + 封顶）
        surge = (sd_multiplier * 0.4 +
                 time_multiplier * 0.2 +
                 weather_multiplier * 0.2 +
                 event_multiplier * 0.2)

        surge = min(surge, 3.0)  # 最高 3 倍封顶

        return {
            "surge_multiplier": round(surge, 1),
            "factors": {
                "supply_demand_ratio": round(supply_demand, 2),
                "sd_multiplier": sd_multiplier,
                "time_multiplier": time_multiplier,
                "weather_multiplier": weather_multiplier,
                "event_multiplier": event_multiplier,
            }
        }

    def calculate_driver_incentive(self, driver_id, area):
        """计算司机激励（引导司机前往高需求区域）"""
        target_areas = self._get_high_demand_areas()

        current_location = self._get_driver_location(driver_id)
        nearest_target = self._find_nearest_target(current_location, target_areas)

        if not nearest_target:
            return {"incentive": 0}

        distance = self._calc_distance(current_location, nearest_target)
        incentive = max(5, nearest_target["demand_gap"] * 2 - distance * 0.5)

        return {
            "incentive": round(incentive, 1),
            "target_area": nearest_target["name"],
            "distance_km": round(distance, 1),
            "expires_in_minutes": 30
        }
```

## 司机信用评分系统

```python
class DriverCreditScore:
    """司机信用评分：多维度 + 动态 + 奖惩"""

    DIMENSIONS = {
        "completion_rate": {"weight": 0.25},
        "cancel_rate": {"weight": 0.20},
        "rating": {"weight": 0.20},
        "response_time": {"weight": 0.15},
        "safety_score": {"weight": 0.20},
    }

    def calculate_score(self, driver_id):
        """计算司机信用分（0-100）"""
        # 完单率
        total_orders = self.db.count("orders", driver_id=driver_id)
        completed = self.db.count("orders", driver_id=driver_id, status="completed")
        completion_rate = completed / max(total_orders, 1)
        completion_score = completion_rate * 100

        # 取消率（越低越好）
        cancelled = self.db.count("orders", driver_id=driver_id, status="cancelled_by_driver")
        cancel_rate = cancelled / max(total_orders, 1)
        cancel_score = max(0, 100 - cancel_rate * 200)

        # 乘客评分
        avg_rating = self.db.query_one(
            "SELECT AVG(rating) as avg FROM driver_ratings "
            "WHERE driver_id = %s AND created_at > NOW() - INTERVAL 30 DAY",
            driver_id)["avg"] or 5.0
        rating_score = avg_rating / 5 * 100

        # 响应时间
        avg_response = self.db.query_one(
            "SELECT AVG(TIMESTAMPDIFF(SECOND, dispatched_at, accepted_at)) as avg "
            "FROM orders WHERE driver_id = %s AND accepted_at IS NOT NULL "
            "AND dispatched_at > NOW() - INTERVAL 30 DAY",
            driver_id)["avg"] or 30
        response_score = max(0, 100 - (avg_response - 10) * 2)

        # 安全分（超速、急刹车事件）
        safety_events = self.db.count("safety_events", driver_id=driver_id,
            created_at__gte=now()-timedelta(days=30))
        safety_score = max(0, 100 - safety_events * 10)

        # 加权综合
        scores = {
            "completion_rate": completion_score,
            "cancel_rate": cancel_score,
            "rating": rating_score,
            "response_time": response_score,
            "safety": safety_score,
        }
        overall = sum(scores[k] * self.DIMENSIONS[k]["weight"]
                     for k in ["completion_rate", "cancel_rate", "rating",
                              "response_time", "safety"])

        # 奖惩
        tier = "gold" if overall >= 85 else "silver" if overall >= 70 else "bronze"

        return {"overall": round(overall, 1), "tier": tier, "scores": scores}

    def apply_penalties(self, driver_id, violation_type):
        """应用处罚"""
        penalties = {
            "high_cancel_rate": {"min_score": 60, "action": "reduce_dispatch_priority"},
            "low_rating": {"min_score": 65, "action": "mandatory_training"},
            "safety_violation": {"min_score": 0, "action": "suspend_7_days"},
        }

        penalty = penalties.get(violation_type)
        if penalty:
            score = self.calculate_score(driver_id)
            if score["overall"] < penalty["min_score"]:
                return  # 已低于阈值
            self._execute_penalty(driver_id, penalty["action"])
```

## 异常场景补充

### 场景：动态定价引发公关危机

```
触发：暴雨天加价 3 倍 → 社交媒体负面舆论 → "趁火打劫"
检测：
  1. 社交媒体负面评论激增 → 公关风险
  2. 加价倍率 > 2.5 且持续时间 > 1 小时 → 高风险
处理：
  1. 立即降低加价倍率上限（3x → 2x）
  2. 发布公告说明加价机制
  3. 对受影响用户发放优惠券
预防：加价倍率封顶 + 极端天气特殊策略 + 公关预案
```

### 场景：司机刷信用分

```
触发：司机通过接短途单刷完单率 → 信用分虚高 → 实际服务质量差
检测：
  1. 完单率 99% 但平均订单距离 < 3km → 短途刷单
  2. 高分司机投诉率反而高 → 分数虚高
处理：
  1. 信用分加入订单多样性维度（长途/短途/高峰/非高峰）
  2. 短途单完单率权重降低
  3. 加入乘客投诉维度
预防：多维评分 + 异常模式检测 + 定期审核
```

## 司机与乘客安全系统完整实现

```python
class RideSafetyService:
    """出行安全：行程分享 + 紧急求助 + 行程异常检测"""

    def share_trip(self, user_id, trip_id, contact_phone):
        """行程分享（实时位置）"""
        # 生成分享链接
        share_token = str(uuid4())
        self.db.insert("trip_shares", {
            "share_token": share_token,
            "trip_id": trip_id,
            "sharer_id": user_id,
            "contact_phone": contact_phone,
            "status": "active",
            "created_at": now()
        })

        # 发送短信给联系人
        trip = self.db.get_trip(trip_id)
        self.sms.send(contact_phone,
            f"您的朋友分享了行程：从 {trip['origin']} 到 {trip['destination']}，"
            f"点击查看实时位置: /trip/track/{share_token}")

        return {"share_token": share_token}

    def emergency_sos(self, user_id, trip_id, location):
        """紧急求助"""
        # 1. 记录 SOS
        sos_id = str(uuid4())
        self.db.insert("emergency_sos", {
            "sos_id": sos_id,
            "user_id": user_id,
            "trip_id": trip_id,
            "location_lat": location["lat"],
            "location_lng": location["lng"],
            "status": "active",
            "created_at": now()
        })

        # 2. 通知安全中心
        self.alert(f"SOS: 用户 {user_id} 在行程 {trip_id} 中触发紧急求助")

        # 3. 通知紧急联系人
        contacts = self.db.query(
            "SELECT * FROM emergency_contacts WHERE user_id = %s", user_id)
        for contact in contacts:
            self.sms.send(contact["phone"],
                f"紧急！您的朋友在行程中触发了紧急求助，位置: {location['lat']},{location['lng']}")

        # 4. 自动拨打 110
        # 实际生产中需要与公安系统对接

        return {"sos_id": sos_id, "contacts_notified": len(contacts)}

    def detect_trip_anomaly(self, trip_id):
        """行程异常检测"""
        trip = self.db.get_trip(trip_id)
        anomalies = []

        # 1. 偏离路线检测
        planned_route = self._get_planned_route(trip_id)
        current_location = self._get_current_location(trip_id)
        if planned_route and current_location:
            deviation = self._calc_route_deviation(current_location, planned_route)
            if deviation > 2000:  # 偏离 2km
                anomalies.append({
                    "type": "route_deviation",
                    "deviation_m": round(deviation, 0)
                })

        # 2. 异常停车检测
        last_movement = self.redis.get(f"trip_movement:{trip_id}")
        if last_movement:
            stationary_minutes = (now() - datetime.fromisoformat(last_movement)).total_seconds() / 60
            if stationary_minutes > 10:  # 停留 10 分钟
                anomalies.append({
                    "type": "unusual_stop",
                    "duration_minutes": round(stationary_minutes, 0)
                })

        # 3. 异常行为（争吵检测）
        # 基于车内麦克风的声音分析（需用户授权）

        # 4. 处理异常
        if anomalies:
            severity = "high" if any(a["type"] == "route_deviation" for a in anomalies) else "medium"
            if severity == "high":
                self._trigger_safety_check(trip_id, anomalies)

        return {"anomalies": anomalies, "severity": severity if anomalies else "none"}
```

## 司机乘客互评系统

```python
class RideRatingService:
    """行程互评系统：评分 + 标签 + 画像"""

    RATING_CATEGORIES = {
        "driver": ["驾驶平稳", "服务态度", "车内整洁", "路线合理", "准时到达"],
        "passenger": ["礼貌友好", "准时上车", "无额外要求", "保持车内整洁"],
    }

    def submit_rating(self, rater_id, ratee_id, trip_id, score, tags=None,
                      comment=None):
        """提交评分"""
        if score < 1 or score > 5:
            raise InvalidRatingError("评分必须在 1-5 之间")

        # 检查是否已评价
        existing = self.db.count("ride_ratings",
            rater_id=rater_id, trip_id=trip_id)
        if existing:
            return {"status": "already_rated"}

        self.db.insert("ride_ratings", {
            "rater_id": rater_id,
            "ratee_id": ratee_id,
            "trip_id": trip_id,
            "score": score,
            "tags": json.dumps(tags or []),
            "comment": comment,
            "created_at": now()
        })

        # 更新用户评分缓存
        self._update_rating_cache(ratee_id)

        # 低分预警
        if score <= 2:
            self._flag_low_rating(ratee_id, score, tags)

        return {"status": "rated", "score": score}

    def _update_rating_cache(self, user_id):
        """更新评分缓存"""
        avg = self.db.query_one(
            "SELECT AVG(score) as avg, COUNT(*) as count "
            "FROM ride_ratings WHERE ratee_id = %s "
            "AND created_at > NOW() - INTERVAL 90 DAY", user_id)

        self.redis.hset(f"user_rating:{user_id}", mapping={
            "avg_score": round(avg["avg"] or 5.0, 2),
            "rating_count": avg["count"]
        })

    def _flag_low_rating(self, user_id, score, tags):
        """低分预警"""
        # 检查近期低分频率
        recent_low = self.db.count("ride_ratings",
            ratee_id=user_id, score__lte=2,
            created_at__gte=now()-timedelta(days=7))

        if recent_low >= 3:
            self.alert(f"用户 {user_id} 近 7 天收到 {recent_low} 次低分评价")
            if self._is_driver(user_id):
                self._require_driver_training(user_id)
```

## 异常场景补充

### 场景：SOS 误触发

```
触发：用户误触 SOS 按钮 → 紧急联系人收到恐慌短信 → 不必要的麻烦
检测：
  1. SOS 后 30 秒内用户取消 → 误触发
  2. 误触发频率 > 5% → 按钮设计问题
处理：
  1. SOS 增加 5 秒倒计时确认
  2. 误触发后自动发送"已安全"通知给联系人
  3. 短时间多次误触发 → 禁用 SOS 30 分钟
预防：倒计时确认 + 取消机制 + 频率限制
```

### 场景：评分系统被恶意刷分

```
触发：司机雇佣他人给竞争对手刷低分 → 不公平竞争
检测：
  1. 同一评分者给多人低分 → 可能恶意
  2. 新账号集中给某司机低分 → 刷分
处理：
  1. 过滤新账号评分（注册 < 7 天不计入均分）
  2. 异常评分模式检测
  3. 确认后移除恶意评分
预防：新账号评分权重降低 + 异常检测 + 评分审计
```

## 拼车匹配算法完整实现

```python
class CarpoolMatchingService:
    """拼车匹配：路线重叠 + 时间窗 + 成本分摊"""

    def find_carpool_candidates(self, trip_request):
        """查找拼车候选"""
        origin = (trip_request["origin_lat"], trip_request["origin_lng"])
        destination = (trip_request["dest_lat"], trip_request["dest_lng"])
        pickup_time = trip_request["pickup_time"]
        max_detour_minutes = trip_request.get("max_detour", 10)

        # 1. 空间过滤：起点和终点都在附近
        nearby_origins = self.redis.geosearch(
            "trip_origins", *origin, radius=2000, unit="m")
        nearby_dests = self.redis.geosearch(
            "trip_destinations", *destination, radius=3000, unit="m")

        candidate_trip_ids = set(nearby_origins) & set(nearby_dests)

        # 2. 时间窗过滤
        candidates = []
        for tid in candidate_trip_ids:
            trip = self.db.get_trip(tid)
            if trip["status"] != "seeking_carpool":
                continue

            # 时间差 < 10 分钟
            time_diff = abs((trip["pickup_time"] - pickup_time).total_seconds()) / 60
            if time_diff > 10:
                continue

            # 3. 计算绕行距离
            detour = self._calculate_detour(trip, trip_request)
            if detour["detour_minutes"] > max_detour_minutes:
                continue

            # 4. 计算路线重叠率
            overlap = self._calculate_route_overlap(trip, trip_request)

            # 5. 综合评分
            score = overlap * 0.5 + (1 - detour["detour_minutes"] / max_detour_minutes) * 0.3 + \
                    (1 - time_diff / 10) * 0.2

            candidates.append({
                "trip_id": tid, "score": round(score, 3),
                "detour_minutes": round(detour["detour_minutes"], 1),
                "overlap_pct": round(overlap, 3),
                "time_diff_minutes": round(time_diff, 1)
            })

        candidates.sort(key=lambda x: x["score"], reverse=True)

        return candidates[:5]

    def _calculate_detour(self, existing_trip, new_request):
        """计算绕行距离"""
        # 原路线距离
        original_distance = self._route_distance(
            existing_trip["origin_lat"], existing_trip["origin_lng"],
            existing_trip["dest_lat"], existing_trip["dest_lng"])

        # 拼车后路线：起点A → 起点B → 终点A → 终点B（或 A→B→B→A 等多种组合）
        # 取最优拼车路线
        best_detour = None
        for route in self._generate_carpool_routes(existing_trip, new_request):
            route_distance = self._route_distance_multi(route)
            detour = route_distance - original_distance
            detour_minutes = detour / 500  # 假设 500m/min
            if best_detour is None or detour_minutes < best_detour["detour_minutes"]:
                best_detour = {"detour_km": round(detour / 1000, 1),
                              "detour_minutes": round(detour_minutes, 1),
                              "route": route}

        return best_detour

    def calculate_cost_sharing(self, trip_a, trip_b, detour_info):
        """计算拼车费用分摊"""
        # 原始费用
        cost_a = self._estimate_fare(trip_a)
        cost_b = self._estimate_fare(trip_b)

        # 拼车总费用
        total_distance = self._route_distance_multi(detour_info["route"])
        carpool_cost = self._calculate_fare(total_distance)

        # 分摊方式：按原始距离比例分摊
        total_original = cost_a + cost_b
        share_a = carpool_cost * (cost_a / total_original)
        share_b = carpool_cost * (cost_b / total_original)

        # 双方都省钱
        savings_a = cost_a - share_a
        savings_b = cost_b - share_b

        return {
            "original_cost_a": round(cost_a, 2),
            "original_cost_b": round(cost_b, 2),
            "carpool_total": round(carpool_cost, 2),
            "share_a": round(share_a, 2),
            "share_b": round(share_b, 2),
            "savings_a": round(savings_a, 2),
            "savings_b": round(savings_b, 2),
        }
```

## 异常场景补充

### 场景：拼车绕行时间超出预期

```
触发：系统估算绕行 5 分钟 → 实际绕行 20 分钟 → 乘客投诉
检测：
  1. 实际到达时间 - 预估到达时间 > 10 分钟 → 绕行偏差大
  2. 拼车投诉率上升 → 匹配算法问题
处理：
  1. 修正绕行估算模型（加入实时路况）
  2. 放宽绕行容忍度（10 分钟 → 5 分钟）
  3. 对受影响乘客补偿
预防：实时路况修正 + 保守估算 + 绕行限制
```

### 场景：拼车费用分摊争议

```
触发：乘客认为费用分摊不公 → 投诉 → "我的距离更短但付了一半费用"
检测：
  1. 拼车费用相关投诉增加 → 分摊模型问题
  2. 某类路线组合分摊偏差大 → 算法缺陷
处理：
  1. 改用按实际距离比例分摊（非原始费用比例）
  2. 提供费用预览（拼车前告知预计费用）
  3. 争议时人工调整
预防：实际距离分摊 + 费用预览 + 争议处理
```

## 出行价格动态调整完整实现

```python
class DynamicPricingService:
    """动态定价：供需比 + 时段 + 天气 + 活动影响"""

    SURGE_FACTORS = {
        "supply_demand": {"weight": 0.4, "min_ratio": 0.7, "max_multiplier": 3.0},
        "weather": {"weight": 0.25, "multipliers": {"rain": 1.2, "snow": 1.5, "storm": 2.0}},
        "time_of_day": {"weight": 0.2, "peak_hours": [(7,9), (17,20)]},
        "event": {"weight": 0.15, "max_multiplier": 2.0},
    }

    def calculate_dynamic_price(self, base_price, pickup_lat, pickup_lng,
                                dropoff_lat, dropoff_lng):
        """计算动态价格"""
        multipliers = {}

        # 1. 供需比（核心因素）
        available_drivers = self._count_nearby_drivers(pickup_lat, pickup_lng, radius_km=3)
        waiting_riders = self._count_waiting_riders(pickup_lat, pickup_lng, radius_km=3)
        supply_demand_ratio = available_drivers / max(waiting_riders, 1)

        if supply_demand_ratio < 0.5:
            multipliers["supply_demand"] = 3.0  # 严重缺车
        elif supply_demand_ratio < 0.7:
            multipliers["supply_demand"] = 2.0
        elif supply_demand_ratio < 1.0:
            multipliers["supply_demand"] = 1.3
        elif supply_demand_ratio < 1.5:
            multipliers["supply_demand"] = 1.0
        else:
            multipliers["supply_demand"] = 0.9  # 供过于求 → 小幅降价

        # 2. 天气因素
        weather = self.weather_service.get_current(pickup_lat, pickup_lng)
        weather_multiplier = self.SURGE_FACTORS["weather"]["multipliers"].get(
            weather.get("condition", "clear"), 1.0)
        multipliers["weather"] = weather_multiplier

        # 3. 时段因素
        hour = now().hour
        is_peak = any(start <= hour < end for start, end in self.SURGE_FACTORS["time_of_day"]["peak_hours"])
        multipliers["time_of_day"] = 1.3 if is_peak else 1.0

        # 4. 大型活动
        nearby_events = self._find_nearby_events(pickup_lat, pickup_lng)
        event_multiplier = 1.0
        for event in nearby_events:
            if event.get("expected_attendees", 0) > 10000:
                event_multiplier = max(event_multiplier, 2.0)
            elif event.get("expected_attendees", 0) > 1000:
                event_multiplier = max(event_multiplier, 1.5)
        multipliers["event"] = event_multiplier

        # 5. 加权综合
        total_multiplier = sum(
            multipliers[factor] * config["weight"]
            for factor, config in self.SURGE_FACTORS.items()
        )

        # 6. 限制范围
        total_multiplier = max(0.8, min(4.0, total_multiplier))

        # 7. 计算最终价格
        final_price = round(base_price * total_multiplier, 2)

        return {
            "base_price": base_price,
            "final_price": final_price,
            "multiplier": round(total_multiplier, 2),
            "breakdown": {k: round(v, 2) for k, v in multipliers.items()},
            "supply_demand_ratio": round(supply_demand_ratio, 2),
            "available_drivers": available_drivers,
            "waiting_riders": waiting_riders,
        }

    def _count_nearby_drivers(self, lat, lng, radius_km):
        """统计附近可用司机"""
        return self.redis.geosearch_count("driver_locations", lng, lat,
            radius=radius_km * 1000, unit="m")

    def _count_waiting_riders(self, lat, lng, radius_km):
        """统计附近等待的乘客"""
        return self.redis.geosearch_count("rider_requests", lng, lat,
            radius=radius_km * 1000, unit="m")

    def _find_nearby_events(self, lat, lng):
        """查找附近活动"""
        return self.db.query(
            "SELECT * FROM events "
            "WHERE ST_DWithin(location, ST_MakePoint(%s, %s), 5000) "
            "AND start_time BETWEEN NOW() - INTERVAL 3 HOUR AND NOW() + INTERVAL 3 HOUR",
            lng, lat)
```

## 异常场景补充

### 场景：动态定价被质疑价格歧视

```
触发：同一行程不同用户看到不同价格 → 指控价格歧视 → 监管风险
检测：
  1. 用户投诉价格不公 → 歧视嫌疑
  2. 媒体报道 → 公关危机
处理：
  1. 动态定价基于客观因素（供需/天气/时段）而非用户画像
  2. 定价透明：向用户展示加价原因
  3. 设置加价上限（最高 4x）
预防：基于客观因素 + 定价透明 + 加价上限
```

### 场景：供需数据延迟导致定价不准

```
触发：附近有 100 辆空车但缓存显示只有 10 辆 → 错误加价 → 乘客多付钱
检测：
  1. 供需比与实际观察不符 → 数据延迟
  2. 无加价条件下仍显示加价 → 定价不准
处理：
  1. 使用实时数据（Redis GEO 而非定期同步）
  2. 定价偏差 > 20% → 自动恢复基础价
  3. 差价退还
预防：实时数据 + 偏差自动恢复 + 差价退还
```

## 出行安全监控完整实现

```python
class RideSafetyMonitorService:
    """出行安全：行程分享 + 紧急求助 + 异常检测"""

    def share_trip(self, user_id, order_id, share_to_contacts, share_duration_minutes=60):
        """行程分享"""
        order = self.db.get_order(order_id)
        if order["passenger_id"] != user_id:
            raise PermissionDeniedError("只有乘客可以分享行程")

        share_token = str(uuid4())[:8]
        expires_at = now() + timedelta(minutes=share_duration_minutes)

        self.db.insert("trip_shares", {
            "share_token": share_token,
            "order_id": order_id,
            "shared_by": user_id,
            "expires_at": expires_at,
            "created_at": now()
        })

        # 通知分享对象
        for contact in share_to_contacts:
            self.notification.send(contact,
                f"行程分享: 点击查看实时位置 /trip/share/{share_token}",
                "trip_share")

        return {"share_token": share_token,
                "share_url": f"/trip/share/{share_token}",
                "expires_at": expires_at.isoformat()}

    def get_shared_trip(self, share_token):
        """获取分享的行程信息"""
        share = self.db.query_one(
            "SELECT * FROM trip_shares WHERE share_token = %s",
            share_token)

        if not share:
            return {"status": "invalid_token"}

        if share["expires_at"] < now():
            return {"status": "expired"}

        order = self.db.get_order(share["order_id"])

        # 返回实时位置（不暴露司机/乘客个人信息）
        driver_location = self.redis.geopos("driver_locations", order["driver_id"])

        return {
            "status": "active",
            "order_status": order["status"],
            "pickup_address": order["pickup_address"],
            "dropoff_address": order["dropoff_address"],
            "driver_location": {"lat": driver_location[1], "lng": driver_location[0]} if driver_location else None,
            "started_at": order.get("started_at", {}).isoformat() if order.get("started_at") else None,
            "expires_at": share["expires_at"].isoformat()
        }

    def trigger_sos(self, user_id, order_id, location=None):
        """紧急求助"""
        # 1. 记录 SOS 事件
        sos_id = str(uuid4())
        self.db.insert("ride_sos_events", {
            "sos_id": sos_id,
            "user_id": user_id,
            "order_id": order_id,
            "location_lat": location.get("lat") if location else None,
            "location_lng": location.get("lng") if location else None,
            "status": "active",
            "created_at": now()
        })

        # 2. 通知安全团队
        self.alert(f"SOS 求助: 用户 {user_id}, 订单 {order_id}", priority="critical")

        # 3. 通知紧急联系人
        emergency_contacts = self._get_emergency_contacts(user_id)
        for contact in emergency_contacts:
            self.notification.send(contact["phone"],
                f"紧急求助: 您的联系人 {user_id} 在出行中触发了紧急求助",
                "sos_alert", priority="urgent")

        # 4. 自动报警（如用户开启）
        user_settings = self.db.get_user_safety_settings(user_id)
        if user_settings.get("auto_police"):
            self._notify_police(user_id, order_id, location)

        # 5. 开启全程录音
        order = self.db.get_order(order_id)
        self.audio_recording.start_emergency_recording(order_id)

        return {"sos_id": sos_id, "status": "active",
                "contacts_notified": len(emergency_contacts)}

    def detect_ride_anomaly(self, order_id):
        """检测行程异常"""
        order = self.db.get_order(order_id)
        anomalies = []

        # 1. 路线偏离检测
        if order["status"] == "in_trip":
            current = self.redis.geopos("driver_locations", order["driver_id"])
            if current:
                planned_route = json.loads(order.get("planned_route", "[]"))
                if planned_route:
                    deviation = self._route_deviation(current, planned_route)
                    if deviation > 5000:  # 偏离 5km
                        anomalies.append({"type": "route_deviation",
                            "deviation_m": round(deviation, 0)})

        # 2. 长时间停留检测
        last_movement = self.redis.get(f"driver_last_move:{order['driver_id']}")
        if last_movement:
            idle_minutes = (now() - datetime.fromisoformat(last_movement.decode())).total_seconds() / 60
            if idle_minutes > 10 and order["status"] == "in_trip":
                anomalies.append({"type": "long_stop",
                    "duration_minutes": round(idle_minutes, 0)})

        # 3. 预计到达时间严重超时
        if order.get("eta") and now() > order["eta"] + timedelta(minutes=30):
            anomalies.append({"type": "severely_late",
                "minutes_late": round((now() - order["eta"]).total_seconds() / 60, 0)})

        if anomalies:
            self.db.insert("ride_anomaly_events", {
                "event_id": str(uuid4()),
                "order_id": order_id,
                "anomalies": json.dumps(anomalies),
                "created_at": now()
            })

        return {"order_id": order_id, "anomalies": anomalies}
```

## 异常场景补充

### 场景：SOS 误触发

```
触发：用户误触 SOS → 紧急联系人收到恐慌短信 → 报警 → 资源浪费
检测：
  1. SOS 后 1 分钟内取消 → 误触
  2. SOS 误触率 > 10% → 按钮设计问题
处理：
  1. SOS 需要长按 3 秒或二次确认
  2. 误触后可快速取消（取消通知）
  3. 误触不触发报警（需确认后才报警）
预防：长按确认 + 快速取消 + 延迟报警
```

### 场景：行程分享链接被滥用

```
触发：行程分享链接被转发到公开群 → 陌生人可追踪乘客位置 → 安全风险
检测：
  1. 分享链接访问 IP 多样化 → 可能泄露
  2. 单链接访问次数 > 10 → 可能滥用
处理：
  1. 限制分享链接访问次数
  2. 分享链接隐藏具体地址（只显示大致位置）
  3. 链接有效期缩短
预防：访问次数限制 + 位置模糊 + 短有效期
```

## 出行司机等级与激励机制完整实现

```python
class DriverLevelService:
    """司机等级：等级计算 + 权益分配 + 激励奖惩"""

    LEVEL_CONFIG = {
        "bronze": {"min_score": 0, "max_score": 60, "commission_discount": 0,
                   "priority_dispatch": False, "bonus_per_order": 0},
        "silver": {"min_score": 60, "max_score": 75, "commission_discount": 5,
                   "priority_dispatch": False, "bonus_per_order": 1},
        "gold": {"min_score": 75, "max_score": 90, "commission_discount": 10,
                 "priority_dispatch": True, "bonus_per_order": 3},
        "diamond": {"min_score": 90, "max_score": 100, "commission_discount": 15,
                    "priority_dispatch": True, "bonus_per_order": 5},
    }

    SCORE_COMPONENTS = {
        "service_quality": {"weight": 0.3, "max_score": 100},
        "completion_rate": {"weight": 0.25, "max_score": 100},
        "acceptance_rate": {"weight": 0.2, "max_score": 100},
        "on_time_rate": {"weight": 0.15, "max_score": 100},
        "experience": {"weight": 0.1, "max_score": 100},
    }

    def calculate_driver_score(self, driver_id):
        """计算司机综合评分"""
        scores = {}

        # 1. 服务质量（乘客评分）
        avg_rating = self.db.query_one(
            "SELECT AVG(rating) as avg FROM order_ratings "
            "WHERE driver_id = %s AND created_at > NOW() - INTERVAL 30 DAY",
            driver_id)["avg"] or 0
        scores["service_quality"] = round(avg_rating * 20, 1)  # 5 星 → 100 分

        # 2. 完成率
        total_orders = self.db.count("orders",
            driver_id=driver_id, created_at__gte=now()-timedelta(days=30))
        completed_orders = self.db.count("orders",
            driver_id=driver_id, status="completed",
            created_at__gte=now()-timedelta(days=30))
        scores["completion_rate"] = round(completed_orders / max(total_orders, 1) * 100, 1)

        # 3. 接单率
        total_dispatched = self.db.count("dispatch_records",
            driver_id=driver_id,
            created_at__gte=now()-timedelta(days=30))
        accepted = self.db.count("dispatch_records",
            driver_id=driver_id, accepted=True,
            created_at__gte=now()-timedelta(days=30))
        scores["acceptance_rate"] = round(accepted / max(total_dispatched, 1) * 100, 1)

        # 4. 准时率
        on_time = self.db.count("orders",
            driver_id=driver_id,
            arrival_time__lte=F("promised_arrival_time"),
            created_at__gte=now()-timedelta(days=30))
        scores["on_time_rate"] = round(on_time / max(total_orders, 1) * 100, 1)

        # 5. 经验（累计订单数）
        lifetime_orders = self.db.count("orders",
            driver_id=driver_id, status="completed")
        scores["experience"] = min(100, lifetime_orders / 10)  # 1000 单 = 100 分

        # 加权综合评分
        overall = sum(scores[k] * self.SCORE_COMPONENTS[k]["weight"]
                      for k in scores) / sum(
            self.SCORE_COMPONENTS[k]["weight"] for k in scores)

        # 确定等级
        level = "bronze"
        for level_name, config in self.LEVEL_CONFIG.items():
            if config["min_score"] <= overall < config["max_score"]:
                level = level_name
                break

        # 记录评分
        self.db.upsert("driver_scores", {
            "driver_id": driver_id,
            "overall_score": round(overall, 1),
            "level": level,
            "component_scores": json.dumps(scores),
            "calculated_at": now()
        }, conflict_columns=["driver_id"])

        return {"driver_id": driver_id, "overall_score": round(overall, 1),
                "level": level, "component_scores": scores}

    def apply_level_benefits(self, driver_id, level):
        """应用等级权益"""
        config = self.LEVEL_CONFIG[level]

        # 1. 佣金折扣
        if config["commission_discount"] > 0:
            self.db.update("driver_profiles",
                {"commission_discount_pct": config["commission_discount"]},
                {"id": driver_id})

        # 2. 优先派单
        if config["priority_dispatch"]:
            self.redis.zadd("priority_drivers", {driver_id: self._get_driver_score(driver_id)})

        # 3. 每单奖励
        self.redis.set(f"driver_bonus:{driver_id}",
            config["bonus_per_order"])

        # 4. 通知司机
        self.notification.send(driver_id,
            f"恭喜升级到 {level} 级！佣金折扣 {config['commission_discount']}%，"
            f"每单奖励 {config['bonus_per_order']} 元")

        return {"driver_id": driver_id, "level": level, "benefits": config}

    def handle_level_downgrade(self, driver_id, old_level, new_level):
        """处理等级降级"""
        # 1. 取消高级权益
        old_config = self.LEVEL_CONFIG[old_level]
        if old_config["priority_dispatch"]:
            self.redis.zrem("priority_drivers", driver_id)

        # 2. 降级通知
        self.notification.send(driver_id,
            f"您的等级从 {old_level} 降为 {new_level}，请提升服务质量")

        # 3. 降级保护期（30 天内只降一级）
        self.db.insert("driver_level_changes", {
            "change_id": str(uuid4()),
            "driver_id": driver_id,
            "old_level": old_level,
            "new_level": new_level,
            "change_type": "downgrade",
            "protection_end": now() + timedelta(days=30),
            "changed_at": now()
        })

        return {"driver_id": driver_id, "old_level": old_level, "new_level": new_level}
```

## 异常场景补充

### 场景：等级评分波动导致频繁升降级

```
触发：司机评分在 74-76 之间波动 → 每周升降级一次 → 权益频繁变化 → 体验差
检测：
  1. 月内升降级 > 2 次 → 频繁波动
  2. 评分在等级边界附近 → 易波动
处理：
  1. 降级保护期（降级后 30 天内不再降）
  2. 评分平滑（取 7 天均值而非单日）
  3. 降级缓冲区（评分差 3 分以内不降级）
预防：保护期 + 平滑评分 + 缓冲区
```

### 场景：评分系统被恶意刷分

```
触发：司机与乘客串通 → 每次给 5 星评分 → 评分虚高 → 等级不匹配实际服务质量
检测：
  1. 同一乘客反复给同一司机 5 星 → 串通嫌疑
  2. 评分与投诉率不一致 → 可能刷分
处理：
  1. 过滤异常评分（同一乘客同一司机 > 5 次 → 权重降低）
  2. 评分 + 投诉综合评估
  3. 人工审核异常评分模式
预防：评分权重限制 + 多维度评估 + 异常审核
```

## 出行平台价格动态调整完整实现

```python
class DynamicPricingService:
    """动态定价：供需分析 → 价格系数 → 尖峰定价 → 价格上限"""

    SURGE_CONFIG = {
        "low_demand": {"multiplier_range": [0.8, 1.0], "threshold_ratio": 0.5},
        "normal": {"multiplier_range": [1.0, 1.0], "threshold_ratio": 0.8},
        "moderate_surge": {"multiplier_range": [1.1, 1.5], "threshold_ratio": 1.2},
        "high_surge": {"multiplier_range": [1.5, 2.0], "threshold_ratio": 2.0},
        "extreme_surge": {"multiplier_range": [2.0, 3.0], "threshold_ratio": 3.0},
    }

    MAX_MULTIPLIER = 3.0
    MIN_MULTIPLIER = 0.8

    def calculate_dynamic_price(self, route, base_price):
        """计算动态价格"""
        pickup_lat = route["pickup_lat"]
        pickup_lng = route["pickup_lng"]

        # 1. 获取区域供需比例
        supply_demand_ratio = self._calculate_supply_demand_ratio(
            pickup_lat, pickup_lng)

        # 2. 确定价格系数
        surge_level = self._determine_surge_level(supply_demand_ratio)
        config = self.SURGE_CONFIG[surge_level]

        # 3. 在系数范围内根据供需比选择具体值
        min_m, max_m = config["multiplier_range"]

        if max_m == min_m:
            multiplier = Decimal(str(min_m))
        else:
            # 在范围内线性映射
            ratio_normalized = min(1.0, max(0.0,
                (supply_demand_ratio - config["threshold_ratio"]) / 0.5))
            multiplier = Decimal(str(min_m + (max_m - min_m) * ratio_normalized))

        # 4. 限制系数范围
        multiplier = max(Decimal(str(self.MIN_MULTIPLIER)),
                        min(Decimal(str(self.MAX_MULTIPLIER)), multiplier))

        # 5. 计算动态价格
        base_price = Decimal(str(base_price))
        dynamic_price = (base_price * multiplier).quantize(
            Decimal("0.01"), rounding=ROUND_HALF_UP)

        # 6. 记录定价日志
        self.db.insert("dynamic_pricing_log", {
            "log_id": str(uuid4()),
            "pickup_lat": pickup_lat, "pickup_lng": pickup_lng,
            "supply_demand_ratio": round(supply_demand_ratio, 2),
            "surge_level": surge_level,
            "multiplier": str(multiplier),
            "base_price": str(base_price),
            "dynamic_price": str(dynamic_price),
            "created_at": now()
        })

        return {
            "base_price": str(base_price),
            "multiplier": str(multiplier),
            "dynamic_price": str(dynamic_price),
            "surge_level": surge_level,
            "supply_demand_ratio": round(supply_demand_ratio, 2),
            "price_difference": str(dynamic_price - base_price)
        }

    def _calculate_supply_demand_ratio(self, lat, lng):
        """计算供需比例"""
        # 需求：该区域等待接单的请求数
        demand = self.redis.zcard(f"pending_requests:{self._get_zone(lat, lng)}")

        # 供给：该区域空闲车辆数
        supply = len(self.redis.georadius(
            "fleet_locations", lng, lat, 3, unit="km"))

        ratio = demand / max(supply, 1)
        return ratio

    def _determine_surge_level(self, ratio):
        """确定尖峰等级"""
        if ratio >= 3.0:
            return "extreme_surge"
        elif ratio >= 2.0:
            return "high_surge"
        elif ratio >= 1.2:
            return "moderate_surge"
        elif ratio >= 0.8:
            return "normal"
        else:
            return "low_demand"

    def _get_zone(self, lat, lng):
        """获取区域编码（网格化）"""
        # 将坐标网格化（每 0.01 度约 1km）
        zone_lat = round(lat, 2)
        zone_lng = round(lng, 2)
        return f"{zone_lat}_{zone_lng}"

    def get_surge_map(self, city_center_lat, city_center_lng, radius_km=20):
        """获取城市尖峰定价地图"""
        zones = self.redis.georadius(
            "zone_centers", city_center_lng, city_center_lat,
            radius_km, unit="km", withcoord=True)

        surge_map = []
        for zone in zones:
            zone_id = zone[0].decode() if isinstance(zone[0], bytes) else zone[0]
            coords = zone[1]

            ratio = self._calculate_supply_demand_ratio(coords[1], coords[0])
            level = self._determine_surge_level(ratio)

            surge_map.append({
                "zone_id": zone_id,
                "lat": coords[1], "lng": coords[0],
                "demand_supply_ratio": round(ratio, 2),
                "surge_level": level,
                "multiplier_range": self.SURGE_CONFIG[level]["multiplier_range"]
            })

        return {"surge_map": surge_map}
```

## 异常场景补充

### 场景：动态定价导致用户流失

```
触发：雨天尖峰定价 3 倍 → 用户觉得太贵 → 取消订单 → 流失率上升
检测：
  1. 尖峰定价时取消率 > 30% → 用户反感
  2. 尖峰时段订单量反而下降 → 定价过高
处理：
  1. 设置价格上限（最高 3 倍）
  2. 尖峰定价时提供等候提示（等待降价）
  3. 封顶后用排队制替代加价制
预防：价格上限 + 等候提示 + 排队制
```

### 场景：尖峰定价被恶意操纵

```
触发：多人同时在同一区域大量下单 → 供需比例虚高 → 价格飙升 → 恶意获利
检测：
  1. 短时间同一区域需求突增 → 可疑
  2. 大量下单后取消 → 恶意操纵
处理：
  1. 取消订单不计入需求计算
  2. 需求统计排除短时间内多次下单
  3. 定价系数平滑（不因短时间波动急升）
预防：排除取消 + 过滤异常 + 平滑系数
```

## 出行平台司机管理与激励完整实现

```python
class DriverManagementService:
    """司机管理：注册 → 评分 → 激励 → 违规处理 → 解封"""

    VIOLATION_TYPES = {
        "high_cancellation": {"first": "warning", "second": "24h_suspension",
                             "third": "7d_suspension", "fourth": "deactivation"},
        "low_rating": {"threshold": 4.0, "below": "training_required"},
        "speeding": {"first": "warning", "second": "3d_suspension",
                    "third": "deactivation"},
        "passenger_complaint": {"first": "warning", "second": "7d_suspension",
                               "third": "deactivation"},
    }

    def register_driver(self, driver_data):
        """注册司机"""
        driver_id = str(uuid4())

        # 1. 验证驾照
        license_valid = self._validate_driver_license(driver_data["license_number"])
        if not license_valid:
            return {"status": "license_invalid"}

        # 2. 验证保险
        insurance_valid = self._validate_insurance(driver_data["insurance_policy"])
        if not insurance_valid:
            return {"status": "insurance_invalid"}

        # 3. 背景审查
        background_check = self._run_background_check(
            driver_data["id_number"], driver_data["name"])

        if background_check.get("failed"):
            return {"status": "background_check_failed",
                    "reason": background_check.get("reason")}

        # 4. 创建司机档案
        self.db.insert("drivers", {
            "driver_id": driver_id,
            "name": driver_data["name"],
            "phone": driver_data["phone"],
            "license_number": driver_data["license_number"],
            "vehicle_info": json.dumps(driver_data.get("vehicle_info", {})),
            "service_area": driver_data.get("service_area"),
            "status": "pending_verification",
            "created_at": now()
        })

        # 5. 通知司机
        self.notification.send(driver_data["phone"],
            "注册成功，请等待审核通过后上线接单")

        return {"driver_id": driver_id, "status": "pending_verification"}

    def calculate_driver_score(self, driver_id):
        """计算司机综合评分"""
        # 1. 接单率（20%权重）
        total_requests = self.db.count("ride_requests",
            driver_id=driver_id,
            created_at__gte=now()-timedelta(days=30))
        accepted_requests = self.db.count("ride_requests",
            driver_id=driver_id, status="accepted",
            created_at__gte=now()-timedelta(days=30))
        acceptance_rate = accepted_requests / max(total_requests, 1) * 100

        # 2. 取消率（15%权重，越低越好 → 反转）
        cancelled = self.db.count("ride_requests",
            driver_id=driver_id, status="cancelled_by_driver",
            created_at__gte=now()-timedelta(days=30))
        cancellation_rate = cancelled / max(total_requests, 1) * 100
        cancellation_score = max(0, 100 - cancellation_rate * 10)

        # 3. 乘客评分（25%权重）
        avg_rating = self.db.query_one(
            "SELECT AVG(rating) as avg FROM passenger_ratings "
            "WHERE driver_id = %s AND created_at > NOW() - INTERVAL 30 DAY",
            driver_id)["avg"] or 5.0
        rating_score = float(avg_rating) * 20  # 5星 → 100分

        # 4. 完单率（25%权重）
        completed = self.db.count("ride_requests",
            driver_id=driver_id, status="completed",
            created_at__gte=now()-timedelta(days=30))
        completion_rate = completed / max(accepted_requests, 1) * 100

        # 5. 在线时长（15%权重）
        online_hours = self.db.query_one(
            "SELECT SUM(online_duration_hours) as total "
            "FROM driver_online_sessions "
            "WHERE driver_id = %s AND date > NOW() - INTERVAL 30 DAY",
            driver_id)["total"] or 0
        online_score = min(100, float(online_hours) / 8 * 100)  # 8小时 → 100分

        # 加权总分（近期权重更高）
        total_score = (
            acceptance_rate * 0.20 +
            cancellation_score * 0.15 +
            rating_score * 0.25 +
            completion_rate * 0.25 +
            online_score * 0.15
        )

        # 确定等级
        if total_score >= 85:
            tier = "platinum"
        elif total_score >= 70:
            tier = "gold"
        elif total_score >= 55:
            tier = "silver"
        else:
            tier = "bronze"

        # 保存评分
        self.db.upsert("driver_scores", {
            "driver_id": driver_id,
            "total_score": round(total_score, 1),
            "tier": tier,
            "acceptance_rate": round(acceptance_rate, 1),
            "cancellation_score": round(cancellation_score, 1),
            "rating_score": round(rating_score, 1),
            "completion_rate": round(completion_rate, 1),
            "online_score": round(online_score, 1),
            "calculated_at": now()
        }, conflict_columns=["driver_id"])

        return {"driver_id": driver_id, "total_score": round(total_score, 1),
                "tier": tier, "dimensions": {
                    "acceptance": round(acceptance_rate, 1),
                    "cancellation": round(cancellation_score, 1),
                    "rating": round(rating_score, 1),
                    "completion": round(completion_rate, 1),
                    "online": round(online_score, 1)
                }}

    def apply_incentive(self, driver_id, incentive_type):
        """应用激励"""
        driver = self.db.get_driver(driver_id)
        score = self.db.query_one(
            "SELECT * FROM driver_scores WHERE driver_id = %s", driver_id)

        # 1. 验证资格
        if incentive_type == "guaranteed_hourly":
            # 保底时薪 → 需满足最低在线时长和接单率
            if float(score.get("acceptance_rate", 0)) < 80:
                return {"status": "ineligible", "reason": "接单率低于 80%"}
            if float(score.get("online_score", 0)) < 50:
                return {"status": "ineligible", "reason": "在线时长不足"}

            guaranteed_rate = Decimal("50")  # 50 元/小时保底
            self.db.insert("driver_incentives", {
                "incentive_id": str(uuid4()),
                "driver_id": driver_id,
                "type": "guaranteed_hourly",
                "rate": str(guaranteed_rate),
                "start_time": now(),
                "end_time": now() + timedelta(hours=4),
                "min_acceptance_rate": 80,
                "status": "active",
                "created_at": now()
            })
            return {"status": "active", "type": "guaranteed_hourly",
                    "rate": str(guaranteed_rate)}

        elif incentive_type == "surge_bonus":
            # 尖峰奖金 → 乘数激励
            bonus_multiplier = Decimal("1.5")
            self.db.insert("driver_incentives", {
                "incentive_id": str(uuid4()),
                "driver_id": driver_id,
                "type": "surge_bonus",
                "multiplier": str(bonus_multiplier),
                "start_time": now(),
                "end_time": now() + timedelta(hours=2),
                "status": "active",
                "created_at": now()
            })
            return {"status": "active", "type": "surge_bonus",
                    "multiplier": str(bonus_multiplier)}

        elif incentive_type == "streak_bonus":
            # 连续接单奖励
            consecutive_trips = self.db.count("ride_requests",
                driver_id=driver_id, status="completed",
                created_at__gte=now()-timedelta(hours=2))

            streak_levels = {5: Decimal("20"), 10: Decimal("50"), 15: Decimal("100")}

            bonus_amount = Decimal("0")
            for threshold, amount in sorted(streak_levels.items(), reverse=True):
                if consecutive_trips >= threshold:
                    bonus_amount = amount
                    break

            if bonus_amount > 0:
                self.db.insert("driver_incentives", {
                    "incentive_id": str(uuid4()),
                    "driver_id": driver_id,
                    "type": "streak_bonus",
                    "amount": str(bonus_amount),
                    "consecutive_trips": consecutive_trips,
                    "status": "completed",
                    "created_at": now()
                })

            return {"status": "completed", "type": "streak_bonus",
                    "consecutive_trips": consecutive_trips,
                    "bonus_amount": str(bonus_amount)}

    def handle_driver_violation(self, driver_id, violation_type, evidence=None):
        """处理司机违规"""
        # 1. 查找过往违规记录
        past_violations = self.db.count("driver_violations",
            driver_id=driver_id, type=violation_type)

        # 2. 确定处罚等级
        violation_config = self.VIOLATION_TYPES.get(violation_type)
        if not violation_config:
            return {"status": "unknown_violation"}

        occurrence = past_violations + 1  # 当前是第 N 次

        penalties = {
            "warning": {"action": "warn", "duration": None},
            "training_required": {"action": "training", "duration": None},
            "24h_suspension": {"action": "suspend", "duration": 24},
            "3d_suspension": {"action": "suspend", "duration": 72},
            "7d_suspension": {"action": "suspend", "duration": 168},
            "deactivation": {"action": "deactivate", "duration": None},
        }

        # 按次数递增处罚
        penalty_key = None
        if occurrence == 1:
            penalty_key = violation_config.get("first")
        elif occurrence == 2:
            penalty_key = violation_config.get("second")
        elif occurrence == 3:
            penalty_key = violation_config.get("third")
        else:
            penalty_key = violation_config.get("fourth", "deactivation")

        penalty = penalties.get(penalty_key, {"action": "warn"})

        # 3. 执行处罚
        if penalty["action"] == "suspend":
            suspend_until = now() + timedelta(hours=penalty["duration"])
            self.db.update("drivers",
                {"status": "suspended", "suspended_until": suspend_until,
                 "suspension_reason": violation_type},
                {"driver_id": driver_id})

        elif penalty["action"] == "deactivate":
            self.db.update("drivers",
                {"status": "deactivated", "deactivated_reason": violation_type},
                {"driver_id": driver_id})

        elif penalty["action"] == "training":
            self.db.update("drivers",
                {"status": "training_required", "training_type": violation_type},
                {"driver_id": driver_id})

        # 4. 记录违规
        self.db.insert("driver_violations", {
            "violation_id": str(uuid4()),
            "driver_id": driver_id,
            "type": violation_type,
            "occurrence": occurrence,
            "penalty": penalty_key,
            "evidence": json.dumps(evidence or {}),
            "created_at": now()
        })

        # 5. 通知司机
        self.notification.send(driver_id,
            f"违规处理: {violation_type}，处罚: {penalty_key}")

        return {"driver_id": driver_id, "violation_type": violation_type,
                "occurrence": occurrence, "penalty": penalty_key}

    def _validate_driver_license(self, license_number):
        """验证驾照"""
        # 调用交通部门 API 验证
        result = self.transport_api.verify_license(license_number)
        return result.get("valid", False)

    def _validate_insurance(self, policy_number):
        """验证保险"""
        result = self.insurance_api.verify_policy(policy_number)
        return result.get("valid", False)

    def _run_background_check(self, id_number, name):
        """背景审查"""
        result = self.background_check_service.check(id_number, name)
        return result
```

## 异常场景补充

### 场景：司机恶意刷激励

```
触发：司机短途反复接单完成 → 连续接单数虚高 → 获得连续接单奖励 → 但实际收入低
检测：
  1. 连续完成的订单平均里程 < 2km → 短途刷单
  2. 同一司机在同一区域反复接短途单 → 模式异常
处理：
  1. 连续接单奖励要求最低单均里程 > 5km
  2. 短途订单不计入连续接单统计
  3. 检测到刷单 → 取消奖励 + 违规记录
预防：最低里程门槛 + 短途排除 + 刷单检测
```

### 场景：误封司机导致运力短缺

```
触发：系统误判司机违规 → 封禁 → 该区域运力下降 → 用户等待时间增加 → 投诉
检测：
  1. 封禁后该区域运力下降 > 20% → 运力影响
  2. 封禁司机申诉率高 → 可能误判
处理：
  1. 封禁前增加人工确认环节
  2. 提供快速申诉通道（24 小时内处理）
  3. 申诉成功 → 补偿封禁期间的收入损失
预防：人工确认 + 快速申诉 + 收入补偿
```

## 出行平台乘客安全保护完整实现

```python
class PassengerSafetyService:
    """乘客安全：行程监控 → 异常检测 → 紧急求助 → 安全评分"""

    SAFETY_CHECK_INTERVAL = 30  # 每 30 秒检查一次
    ROUTE_DEVIATION_THRESHOLD = 0.3  # 30% 偏离阈值

    def monitor_ride_safety(self, ride_id):
        """监控行程安全"""
        ride = self.db.get_ride(ride_id)

        if ride["status"] != "in_progress":
            return {"status": "ride_not_active"}

        # 1. 路线偏离检测
        current_pos = self._get_current_position(ride["driver_id"])
        expected_route = json.loads(ride["route"])
        deviation = self._calculate_route_deviation(current_pos, expected_route)

        if deviation > self.ROUTE_DEVIATION_THRESHOLD:
            self._handle_route_deviation(ride_id, deviation)

        # 2. 异常停留检测
        last_update = self.redis.get(f"driver_position_time:{ride['driver_id']}")
        if last_update:
            last_time = datetime.fromisoformat(
                last_update.decode() if isinstance(last_update, bytes) else last_update)
            stagnant_seconds = (now() - last_time).total_seconds()

            if stagnant_seconds > 180:  # 3 分钟无位置更新
                self._handle_abnormal_stop(ride_id, stagnant_seconds)

        # 3. 时间过长检测
        estimated_duration = ride.get("estimated_duration_minutes", 30)
        actual_duration = (now() - ride["started_at"]).total_seconds() / 60

        if actual_duration > estimated_duration * 2:
            self._handle_ride_timeout(ride_id, actual_duration, estimated_duration)

        # 4. 乘客沉默检测（乘客无互动超过 30 分钟）
        last_passenger_interaction = self.db.query_one(
            "SELECT MAX(created_at) as last FROM ride_interactions "
            "WHERE ride_id = %s AND user_type = 'passenger'",
            ride_id)["last"]

        if last_passenger_interaction:
            silence_minutes = (now() - last_passenger_interaction).total_seconds() / 60
            if silence_minutes > 30:
                self._check_passenger_safety(ride_id)

        return {"ride_id": ride_id, "deviation": round(deviation, 2),
                "status": "monitoring"}

    def handle_emergency_request(self, ride_id, passenger_id, emergency_type):
        """处理紧急求助"""
        ride = self.db.get_ride(ride_id)

        # 1. 创建紧急事件
        emergency_id = str(uuid4())
        self.db.insert("ride_emergencies", {
            "emergency_id": emergency_id,
            "ride_id": ride_id,
            "passenger_id": passenger_id,
            "driver_id": ride["driver_id"],
            "emergency_type": emergency_type,
            "status": "active",
            "created_at": now()
        })

        # 2. 根据类型处理
        if emergency_type == "physical_danger":
            # 身体危险 → 最高优先级
            # a. 通知安全中心
            self.alert(f"乘客紧急求助: 身体安全威胁! 行程 {ride_id}")

            # b. 通知司机（让他知道被监控）
            self.notification.send(ride["driver_id"],
                "该行程已触发安全监控，请确保乘客安全")

            # c. 通知紧急联系人
            emergency_contacts = self._get_emergency_contacts(passenger_id)
            for contact in emergency_contacts:
                self.notification.send(contact["phone"],
                    f"您的紧急联系人 {passenger_id} 触发了安全求助，行程号: {ride_id}")

            # d. 自动分享行程位置给紧急联系人
            current_pos = self._get_current_position(ride["driver_id"])
            for contact in emergency_contacts:
                self.sms.send(contact["phone"],
                    f"实时位置: https://maps.example.com/track/{ride_id}")

            # e. 保留录音录像
            self._start_audio_recording(ride_id)

        elif emergency_type == "route_concern":
            # 路线担忧 → 中等优先级
            self.notification.send(ride["driver_id"],
                "乘客对路线有疑虑，请解释路线选择")
            self.notification.send(passenger_id,
                "已通知司机解释路线，如仍有疑虑请再次求助升级")

        elif emergency_type == "feeling_unsafe":
            # 感觉不安全 → 主动联系确认
            self.notification.send(passenger_id,
                "我们已关注您的行程安全，是否需要结束行程？")

            self._schedule_safety_check(ride_id, 60)  # 60 秒后再检查

        # 3. 更新行程安全标记
        self.db.update("rides",
            {"safety_flag": emergency_type},
            {"id": ride_id})

        return {"emergency_id": emergency_id,
                "emergency_type": emergency_type,
                "actions_taken": "notifications_sent"}

    def calculate_driver_safety_score(self, driver_id):
        """计算司机安全评分"""
        # 1. 安全投诉率
        total_rides = self.db.count("rides",
            driver_id=driver_id, status="completed")
        safety_complaints = self.db.count("ride_complaints",
            driver_id=driver_id, type__in=["unsafe_driving", "route_deviation", "harassment"])

        complaint_rate = safety_complaints / max(total_rides, 1)
        complaint_score = max(0, 100 - complaint_rate * 500)  # 1% → 95分

        # 2. 急刹车/急加速检测（来自车辆传感器）
        harsh_events = self.db.count("driving_events",
            driver_id=driver_id, event_type__in=["hard_brake", "rapid_accel"],
            created_at__gte=now()-timedelta(days=30))

        driving_score = max(0, 100 - harsh_events * 5)  # 1 次 → 95分

        # 3. 夜间行车记录
        night_rides = self.db.count("rides",
            driver_id=driver_id, status="completed",
            started_at__hour__gte=22)

        night_score = min(100, night_rides * 2)  # 夜间经验加分

        # 4. 紧急求助触发率
        emergency_triggered = self.db.count("ride_emergencies",
            driver_id=driver_id,
            created_at__gte=now()-timedelta(days=90))

        emergency_score = max(0, 100 - emergency_triggered * 20)

        # 5. 综合评分
        total_score = (
            complaint_score * 0.30 +
            driving_score * 0.30 +
            night_score * 0.10 +
            emergency_score * 0.30
        )

        # 6. 安全等级
        if total_score >= 85:
            safety_level = "excellent"
        elif total_score >= 70:
            safety_level = "good"
        elif total_score >= 55:
            safety_level = "acceptable"
        else:
            safety_level = "poor"

        return {
            "driver_id": driver_id,
            "total_score": round(total_score, 1),
            "safety_level": safety_level,
            "breakdown": {
                "complaint_score": round(complaint_score, 1),
                "driving_score": round(driving_score, 1),
                "night_score": round(night_score, 1),
                "emergency_score": round(emergency_score, 1)
            }
        }

    def _get_current_position(self, driver_id):
        """获取当前位置"""
        pos = self.redis.geopos("driver_positions", driver_id)
        if pos and pos[0]:
            return {"lat": pos[0][1], "lng": pos[0][0]}
        return {"lat": 0, "lng": 0}

    def _calculate_route_deviation(self, current_pos, expected_route):
        """计算路线偏离"""
        # 找最近的路线点
        min_distance = float('inf')
        for waypoint in expected_route:
            distance = self._haversine(
                current_pos["lat"], current_pos["lng"],
                waypoint["lat"], waypoint["lng"])
            min_distance = min(min_distance, distance)

        # 偏离比例 = 实际偏离 / 路线宽度容差
        route_tolerance_km = 0.5  # 500 米容差
        deviation = min_distance / route_tolerance_km

        return deviation

    def _handle_route_deviation(self, ride_id, deviation):
        """处理路线偏离"""
        ride = self.db.get_ride(ride_id)

        self.notification.send(ride["passenger_id"],
            f"行程路线偏离较大（偏离度 {deviation:.1%}），司机是否解释了路线选择？")

        self.db.insert("ride_safety_events", {
            "event_id": str(uuid4()),
            "ride_id": ride_id,
            "type": "route_deviation",
            "deviation": round(deviation, 2),
            "created_at": now()
        })

    def _handle_abnormal_stop(self, ride_id, stagnant_seconds):
        """处理异常停留"""
        ride = self.db.get_ride(ride_id)

        self.notification.send(ride["passenger_id"],
            "行程出现异常停留，如感觉不安全请点击紧急求助")

    def _handle_ride_timeout(self, ride_id, actual, estimated):
        """处理行程超时"""
        ride = self.db.get_ride(ride_id)

        self.notification.send(ride["passenger_id"],
            f"行程时间已超过预估 {actual/estimated:.0%}，请确认是否安全")

    def _get_emergency_contacts(self, passenger_id):
        """获取紧急联系人"""
        return self.db.query(
            "SELECT * FROM emergency_contacts "
            "WHERE user_id = %s AND status = 'active'",
            passenger_id)

    def _start_audio_recording(self, ride_id):
        """开始录音"""
        self.iot_command.send(f"ride:{ride_id}", {
            "action": "start_recording",
            "ride_id": ride_id
        })

    def _check_passenger_safety(self, ride_id):
        """检查乘客安全"""
        ride = self.db.get_ride(ride_id)

        self.notification.send(ride["passenger_id"],
            "行程进行中，是否一切安全？如不需帮助请回复'安全'")

    def _schedule_safety_check(self, ride_id, seconds):
        """安排安全检查"""
        self.task_queue.schedule(
            self.monitor_ride_safety, ride_id,
            delay_seconds=seconds)
```

## 异常场景补充

### 场景：紧急求助被误触发

```
触发：乘客误触紧急求助按钮 → 安全中心介入 → 司机被误标记 → 体验差
检测：
  1. 紧急求助后乘客立即回复"安全" → 误触
  2. 同一乘客频繁触发求助 → 可能误触习惯
处理：
  1. 紧急求助需二次确认（2 秒内再按才触发）
  2. 误触后 30 秒内取消 → 不记录
  3. 司机安全评分不因误触受影响
预防：二次确认 + 取消机制 + 评分排除
```

### 场景：司机故意绕路收取额外费用

```
触发：司机偏离正常路线 → 行程时间延长 → 费用增加 → 乘客被多收费
检测：
  1. 路线偏离 > 30% → 绕路
  2. 实际费用 > 预估费用 50% → 可能绕路
处理：
  1. 自动计算正常路线费用
  2. 绕路部分费用由司机承担
  3. 记录绕路事件 → 影响司机评分
预防：费用上限 + 绕路检测 + 评分影响
```

### 场景：行程中通讯中断无法联系乘客

```
触发：乘客手机信号中断 → 安全检查无法触达 → 无法确认安全 → 可能危险
检测：
  1. 乘客 10 分钟内无任何互动 → 通讯中断
  2. 安全检查消息未送达 → 信号问题
处理：
  1. 通讯中断时自动通知司机停止行程
  2. 尝试联系紧急联系人
  3. 司机需原地等待直到联系成功
预防：司机停车 + 联系紧急联系人 + 等待确认
```
