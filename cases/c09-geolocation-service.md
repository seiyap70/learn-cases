# C09: O2O平台的位置服务架构

## 业务场景

某 O2O 平台（外卖+到店+出行），需要为全球 400 城市的用户提供基于位置的服务。核心功能：附近商家搜索、骑手实时位置追踪、配送范围判定、热力图。

**一个真实的性能事故：** 午高峰期间"附近商家"搜索 QPS 从 2 万飙到 8 万 → Elasticsearch 集群 P99 延迟从 50ms 飙升到 2 秒 → 用户搜索超时 → 订单量下降 30% → 午高峰营收损失约 200 万元。根因：geo_distance 排序在大结果集上极其消耗 CPU。

**已知数据：**
- 商家数：500 万
- 用户数：2 亿
- 在线骑手：50 万（高峰期）
- 骑手位置更新：每 3 秒一次
- "附近"搜索 QPS：10 万
- 搜索延迟要求：< 100ms
- 配送范围判定：< 50ms

**为什么不能用 MySQL 直接做空间查询？**

```sql
SELECT * FROM merchants
WHERE ST_Distance_Sphere(location, ST_MakePoint(116.4, 39.9)) <= 3000
ORDER BY ST_Distance_Sphere(location, ST_MakePoint(116.4, 39.9))
LIMIT 20;
```

500 万商家全表扫描 → 即使有空间索引也需要 1-5 秒 → 违反 100ms 要求。

## 核心挑战

### 挑战 1：空间索引的效率

500 万商家中毫秒级查出 3km 内的商家。B+ 树索引无法高效处理二维空间查询。

**各种方案的性能对比：**

| 方案 | 写入 QPS | 查询延迟 | 距离排序 | 多条件过滤 |
|------|---------|---------|---------|-----------|
| MySQL + 空间索引 | 5000 | 500ms-2s | 支持 | 支持 |
| Redis GEO | 10万/节点 | < 5ms | 支持 | 不支持 |
| Elasticsearch | 5万/节点 | 30-50ms | 支持 | 支持 |
| PostGIS | 3000 | 200ms-1s | 支持 | 完整GIS |

### 挑战 2：骑手位置的实时更新

50 万骑手 × 每 3 秒更新 = 17 万次/秒写入。每次更新需要写入新位置、更新空间索引、支持实时查询。

### 挑战 3：配送范围的复杂形状

商家配送范围不是简单的"3km 圆形"，而是不规则多边形（避开河流、铁路、高速路）。500 万商家 × 平均 20 个顶点的多边形 → 存储 1 亿个坐标点 → 如何高效判定"用户是否在配送范围内"？

### 挑战 4：综合排序的复杂度

用户搜"附近美食"——不只是按距离排序。100 米外的 2 星苍蝇馆子 vs 1 公里外的 4.8 星网红餐厅 → 需要综合排序（距离+评分+配送时长+起送价）。

## 设计约束

- 搜索延迟 < 100ms（P99）
- 骑手位置查询延迟 < 200ms
- 配送范围判定 < 50ms
- 骑手位置写入 QPS > 20 万/秒
- 数据最终一致性（骑手位置允许 1-2 秒延迟）

## 请先独立思考（限时 30 分钟）

1. GeoHash 的原理是什么？为什么它能把二维空间查询转化为一维范围查询？边界问题如何解决？
2. 500 万商家的空间数据用 Redis GEO 还是 Elasticsearch？各自的优劣？
3. 50 万骑手的位置数据如何高效存储和查询？按城市分片的关键是什么？
4. 不规则配送范围如何存储和判断"点是否在多边形内"？

---

## 设计解析

### 分层架构：不同数据用不同引擎

```
商家搜索 → Elasticsearch（距离+评分+分类+营业状态综合查询）
骑手位置 → Redis GEO（高频写入+简单距离查询）
配送范围 → Elasticsearch geo_shape（不规则多边形判定）
热力图   → Redis + Flink 聚合
```

### 商家搜索：Elasticsearch + geo_distance

**索引设计：**

```json
PUT /merchants
{
  "mappings": {
    "properties": {
      "name": { "type": "text", "analyzer": "ik_max_word" },
      "category": { "type": "keyword" },
      "location": { "type": "geo_point" },
      "rating": { "type": "float" },
      "avg_delivery_minutes": { "type": "integer" },
      "min_order_amount": { "type": "float" },
      "is_open": { "type": "boolean" },
      "delivery_area": { "type": "geo_shape" },
      "monthly_orders": { "type": "integer" }
    }
  },
  "settings": {
    "number_of_shards": 20,
    "number_of_replicas": 1,
    "index.sort.field": "monthly_orders",
    "index.sort.order": "desc"
  }
}
```

**搜索查询（完整）：**

```json
GET /merchants/_search
{
  "query": {
    "bool": {
      "must": [
        { "term": { "category": "chinese_food" } },
        { "term": { "is_open": true } },
        { "range": { "min_order_amount": { "lte": 30 } } }
      ],
      "filter": {
        "geo_distance": {
          "distance": "3km",
          "location": { "lat": 39.9165, "lon": 116.3971 }
        }
      }
    }
  },
  "sort": [
    { "_geo_distance": { "location": { "lat": 39.9165, "lon": 116.3971 }, "order": "asc", "unit": "km" } },
    { "rating": { "order": "desc" } }
  ],
  "size": 20
}
```

**性能优化：避免全量 geo_distance 排序**

午高峰事故的根因：`geo_distance` 排序对大量结果计算距离 → CPU 打满。解决方案：先用 `geo_bounding_box` 粗筛，再精确排序。

```json
GET /merchants/_search
{
  "query": {
    "bool": {
      "must": [
        { "term": { "is_open": true } }
      ],
      "filter": [
        {
          "geo_bounding_box": {
            "location": {
              "top_left": { "lat": 39.9435, "lon": 116.3671 },
              "bottom_right": { "lat": 39.8895, "lon": 116.4271 }
            }
          }
        }
      ]
    }
  },
  "sort": [
    { "_geo_distance": { "location": { "lat": 39.9165, "lon": 116.3971 }, "order": "asc", "unit": "km" } }
  ],
  "size": 20
}
```

`geo_bounding_box` 用 BKD 树做矩形过滤 → O(log N) → 快速缩小候选集 → 再对候选集做 geo_distance 排序 → CPU 消耗降低 80%。

**综合排序（距离 + 评分 + 配送时长）：**

```python
class MerchantSorter:
    def sort(self, merchants, user_location):
        for m in merchants:
            distance = geo_distance(user_location, m.location)
            distance_score = max(0, 1 - distance / 3000)    # 0-1，越近越高
            rating_score = m.rating / 5.0                    # 0-1
            delivery_score = max(0, 1 - m.avg_delivery_minutes / 60)  # 0-1

            m.composite_score = (
                0.35 * distance_score +
                0.35 * rating_score +
                0.30 * delivery_score
            )

        return sorted(merchants, key=lambda m: -m.composite_score)
```

### 骑手位置：Redis GEO + 按城市分片

```python
class RiderLocationService:
    def update_location(self, rider_id, lat, lng):
        """骑手位置更新（每 3 秒调用一次）"""
        city_id = self.get_city_id(lat, lng)

        # 1. 更新骑手 GEO 位置
        self.redis.geoadd(f"riders:geo:{city_id}", lng, lat, rider_id)

        # 2. 更新骑手状态（带 TTL，30 秒过期 = 10 次心跳未更新则视为离线）
        self.redis.hset(f"rider:status:{rider_id}", mapping={
            "lat": lat, "lng": lng,
            "timestamp": int(time.time()),
            "status": "delivering",
            "city_id": city_id
        })
        self.redis.expire(f"rider:status:{rider_id}", 30)

        # 3. 轨迹写入 Kafka（异步，不影响位置更新延迟）
        self.kafka.produce("rider_track", {
            "rider_id": rider_id,
            "lat": lat, "lng": lng,
            "timestamp": now_ms()
        })

    def get_nearby_riders(self, lat, lng, radius_km=3, count=20):
        """查询附近空闲骑手"""
        city_id = self.get_city_id(lat, lng)

        results = self.redis.georadius(
            f"riders:geo:{city_id}", lng, lat, radius_km,
            unit="km", withdist=True, withcoord=True,
            count=count * 3,  # 多取 3 倍（部分可能不空闲）
            sort="ASC"
        )

        riders = []
        for rider_id, distance, coord in results:
            status = self.redis.hget(f"rider:status:{rider_id}", "status")
            if status == "idle":
                riders.append({"rider_id": rider_id, "distance": distance})
            if len(riders) >= count:
                break

        return riders
```

**按城市分片的原因：** `GEORADIUS` 在大 Sorted Set 上性能下降。全国 50 万骑手在一个 key 中 → 查询北京 3km 仍需扫描大量无关数据。按城市分片后，单城市 1-5 万骑手 → 查询 < 5ms。

**离线骑手清理：**

```python
class RiderCleanupJob:
    """每分钟清理已离线骑手的 GEO 数据"""
    def cleanup(self, city_id):
        all_riders = self.redis.zrange(f"riders:geo:{city_id}", 0, -1)
        removed = 0
        for rider_id in all_riders:
            if not self.redis.exists(f"rider:status:{rider_id}"):
                self.redis.zrem(f"riders:geo:{city_id}", rider_id)
                removed += 1
        return removed
```

### 不规则配送范围：geo_shape 多边形

**存储：**

```json
{
  "merchant_id": "M-001",
  "delivery_area": {
    "type": "Polygon",
    "coordinates": [[
      [116.397, 39.916], [116.405, 39.918], [116.410, 39.914],
      [116.408, 39.908], [116.400, 39.905], [116.393, 39.910],
      [116.397, 39.916]
    ]]
  }
}
```

**判定用户是否在配送范围内：**

```json
GET /merchants/_search
{
  "query": {
    "bool": {
      "filter": {
        "geo_shape": {
          "delivery_area": {
            "shape": {
              "type": "point",
              "coordinates": [116.401, 39.912]
            },
            "relation": "contains"
          }
        }
      }
    }
  }
}
```

**为什么不用圆形范围？**

| 维度 | 3km 圆形 | 不规则多边形 |
|------|---------|------------|
| 精确性 | 包含河对岸等不可达区域 | 精确排除不可达区域 |
| 配送时间准确性 | 误差大（过河绕行8km vs 直线3km） | 准确 |
| 实现复杂度 | 低（只算距离） | 中（需绘制多边形） |
| 用户投诉率 | 高（预计15分钟实际40分钟） | 低 |

### GeoHash 原理与边界问题

**编码原理：**

```
经度 116.397 → 二进制 110100101011001...
纬度  39.916 → 二进制 101110001100011...

交错合并：11100111000010...（奇数位经度，偶数位纬度）
每 5 位编码为一个 Base32 字符 → wx4g0s...
```

**精度：**

| GeoHash 长度 | 精度 | 适用场景 |
|-------------|------|---------|
| 4 位 | 20km | 省级范围 |
| 5 位 | 2.4km | 区级范围 |
| 6 位 | 610m | 街道范围 |
| 7 位 | 76m | 楼栋范围 |

**边界问题的解决方案：**

```
两个地理位置很近的点，如果正好在 GeoHash 网格的边界两侧：
  点 A: wx4g0s (右侧)    点 B: wx4g0e (左侧)
前缀不同 → 直接按前缀范围查询会漏掉。

解决方案：查询当前格子 + 8 个相邻格子 = 9 宫格，合并查询。
```

```python
class GeoHashSearch:
    def search_nearby(self, lat, lng, radius_km):
        current_hash = geohash.encode(lat, lng, precision=6)
        neighbors = geohash.neighbors(current_hash)
        all_hashes = [current_hash] + neighbors

        results = []
        for h in all_hashes:
            members = self.redis.zrangebyscore(
                f"merchants:geohash:{h[:4]}",
                geohash.decode(h)[0] - radius_km / 111,
                geohash.decode(h)[0] + radius_km / 111
            )
            results.extend(members)

        # 精确过滤：计算真实距离
        filtered = [
            m for m in results
            if geo_distance((lat, lng), m.location) <= radius_km * 1000
        ]
        return sorted(filtered, key=lambda m: geo_distance((lat, lng), m.location))
```

### 热力图：GeoHash 聚合 + Flink

```python
class HeatmapService:
    """实时订单热力图"""

    def get_heatmap(self, city_id, zoom_level=5):
        """获取城市热力图数据"""
        # GeoHash 精度对应缩放级别
        precision_map = {4: 20, 5: 2.4, 6: 0.6}  # km
        precision = precision_map.get(zoom_level, 5)

        # 从 Redis 读取各 GeoHash 格子的订单数
        keys = self.redis.keys(f"heatmap:{city_id}:*")
        heatmap = []
        for key in keys:
            count = self.redis.get(key)
            geohash_str = key.split(":")[-1]
            lat, lng = geohash.decode(geohash_str)
            heatmap.append({
                "geohash": geohash_str,
                "lat": lat, "lng": lng,
                "count": int(count)
            })

        return heatmap

    def on_order_created(self, order):
        """订单创建 → 更新热力图"""
        geohash_str = geohash.encode(order.lat, order.lng, precision=5)
        key = f"heatmap:{order.city_id}:{geohash_str}"
        self.redis.incr(key)
        self.redis.expire(key, 3600)  # 1 小时过期
```

### 写入性能分析

| 数据类型 | 写入 QPS | 引擎 | 节点数 | 存储 |
|---------|---------|------|-------|------|
| 商家数据 | ~1/秒 | ES 批量写入 | 6 节点 | 50GB |
| 骑手位置 | 17 万/秒 | Redis GEO | 4 节点 | 2GB |
| 配送范围 | ~1/秒 | ES geo_shape | 6 节点 | 含在商家索引中 |
| 骑手轨迹 | 17 万/秒 | Kafka → HDFS | 6 broker | 5TB/天 |

## 常见陷阱（深度分析）

### 陷阱 1：geo_distance 排序打满 CPU

**后果：** 午高峰 QPS 8 万 → ES P99 延迟从 50ms 飙升到 2 秒 → 用户搜索超时 → 订单量下降 30%。根因：geo_distance 排序对全量候选集计算距离 → CPU 打满。

**解决方案：** 先用 geo_bounding_box 粗筛（BKD 树，O(log N)）→ 再对候选集排序 → CPU 消耗降低 80%。

### 陷阱 2：GeoHash 不处理边界

**后果：** 用户在格子边缘 → 漏掉相邻格子中的最近商家 → 搜索结果不完整 → 用户看到"附近无商家"但实际 50 米外有商家。

**解决方案：** 查询 9 宫格（当前格子 + 8 邻居）→ 精确过滤 → 不遗漏边界商家。

### 陷阱 3：骑手位置不清理

**后果：** 骑手下班后 GEO 数据仍在 Redis → 查询返回已下线骑手 → 派单失败 → 订单超时 → 用户投诉。

**解决方案：** 骑手状态 key 设 30 秒 TTL + 定时清理 GEO 数据中的离线骑手。

### 陷阱 4：配送范围用圆形

**后果：** 3km 圆形包含河对岸 → 无法配送 → 预计 15 分钟实际 40 分钟 → 用户投诉率升高。

**解决方案：** 不规则多边形（geo_shape），运营在地图上绘制精确配送范围。

### 陷阱 5：骑手轨迹存 Redis

**后果：** 50 万骑手 × 每 3 秒 1 条 × 24 小时 = 14 亿条/天 → Redis 内存不够（约 140GB/天）→ 成本过高。

**解决方案：** 实时位置存 Redis（只存当前），轨迹存 Kafka + HDFS（冷存储，用于骑手行为分析和争议仲裁）。

## 延伸思考

- **热力图渲染**：某区域的订单密度如何实时计算？用 GeoHash 聚合（5 位精度，2.4km 网格）+ Flink 实时统计 → 前端用 Mapbox 渲染。
- **室内定位**：商场内部的商家定位（GPS 精度 10-50m 不够），需要蓝牙信标或 WiFi 指纹定位 → 架构需增加信标数据采集层。
- **跨城配送**：城际物流的距离计算不是直线距离而是路线距离，需要接入地图 API（高德/百度）获取实际行驶距离和 ETA。