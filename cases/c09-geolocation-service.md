# C09: 全球用户的位置服务

## 业务场景

某 O2O 平台（外卖+到店+出行），需要为全球 400 城市的用户提供基于位置的服务：

- "附近"搜索：用户查找 3km 内的商家
- 外卖配送范围：商家只配送到 5km 范围内
- 实时骑手位置：用户追踪骑手到哪了
- 热力图：运营查看某区域的订单密度

**已知数据：**
- 商家数：500 万
- 用户数：2 亿
- 在线骑手：50 万（高峰期）
- 骑手位置更新：每 3 秒
- "附近"搜索 QPS：10 万
- 搜索延迟要求：< 100ms

**为什么不能用 MySQL 直接做空间查询？**

```sql
-- 计算所有商家与用户的距离，返回 3km 内的
SELECT * FROM merchants
WHERE ST_Distance_Sphere(location, ST_MakePoint(116.4, 39.9)) <= 3000
ORDER BY ST_Distance_Sphere(location, ST_MakePoint(116.4, 39.9))
LIMIT 20;
```

这条查询需要计算 500 万商家与用户的距离——全表扫描，即使有空间索引，500 万行的扫描也需要 1-5 秒。

## 核心挑战

### 挑战 1：空间索引的效率

如何在 500 万商家中毫秒级查出 3km 内的商家？B+ 树索引无法高效处理二维空间查询。

### 挑战 2：骑手位置的实时更新

50 万骑手 × 每 3 秒更新位置 = 约 17 万次/秒的位置写入。每次更新需要：
- 写入新位置
- 更新空间索引
- 支持实时查询

### 挑战 3：配送范围的复杂形状

商家配送范围不是简单的"3km 圆形"，而是：
- 不规则多边形（避开河流、铁路、高速路）
- 不同方向的距离不同（向东 5km，向西 3km）

### 挑战 4：全球多时区

400 城市分布在不同时区，商家的营业时间需要按当地时区判断。

## 设计约束

- Redis 集群可用（GEO 模块）
- MySQL 支持 PostGIS 或使用 Elasticsearch 地理查询
- 搜索结果需按距离 + 评分综合排序
- 骑手位置查询延迟 < 200ms

## 请先独立思考（限时 30 分钟）

1. GeoHash 的原理是什么？为什么它能把二维空间查询转化为一维范围查询？精度如何控制？
2. 500 万商家的空间数据用 Redis GEO 还是 Elasticsearch？各自的优劣？
3. 骑手位置每 3 秒更新一次，50 万骑手的位置数据如何高效存储和查询？
4. 不规则配送范围如何存储和判断"点是否在多边形内"？

---

## 设计解析

### 空间索引方案选型

| 方案 | 写入性能 | 查询性能 | 复杂查询支持 | 适用场景 |
|------|---------|---------|------------|---------|
| Redis GEO | 10万QPS/节点 | < 5ms | 仅距离+排序 | 骑手位置、简单附近搜索 |
| Elasticsearch | 5万QPS/节点 | < 50ms | 距离+多边形+过滤 | 商家搜索（含筛选条件） |
| MySQL PostGIS | 5000QPS | 100-500ms | 完整GIS | 复杂空间分析（离线） |

**分工：**
- **商家搜索**：Elasticsearch（需要距离+评分+分类+营业状态综合查询）
- **骑手位置**：Redis GEO（高频写入+简单距离查询）
- **配送范围**：PostgreSQL PostGIS（不规则多边形存储和判断）

### 商家搜索：Elasticsearch

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
      "tags": { "type": "keyword" }
    }
  }
}
```

**搜索查询：**

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

**性能：** 500 万商家，geo_distance 过滤 + 排序 → 约 30-50ms

**综合排序（距离 + 评分）：**

```python
class MerchantSorter:
    def sort(self, merchants, user_location):
        """
        综合排序：距离 + 评分 + 配送时长
        不只按距离排序——1km 外的5星餐厅可能比100m内的3星餐厅更好
        """
        for m in merchants:
            distance = geo_distance(user_location, m.location)
            
            # 距离分数：0-1，越近越高，3km 以上为 0
            distance_score = max(0, 1 - distance / 3000)
            
            # 评分分数：0-1
            rating_score = m.rating / 5.0
            
            # 配送时长分数：越快越高，60分钟以上为0
            delivery_score = max(0, 1 - m.avg_delivery_minutes / 60)
            
            # 综合分
            m.composite_score = (
                0.35 * distance_score +
                0.35 * rating_score +
                0.30 * delivery_score
            )
        
        return sorted(merchants, key=lambda m: -m.composite_score)
```

### 骑手位置：Redis GEO

**写入：**

```python
class RiderLocationService:
    def update_location(self, rider_id, lat, lng):
        """骑手位置更新（每 3 秒调用一次）"""
        city_id = self.get_city_id(lat, lng)
        
        # 更新骑手当前位置
        self.redis.geoadd(f"riders:geo:{city_id}", lng, lat, rider_id)
        
        # 更新骑手状态
        self.redis.hset(f"rider:status:{rider_id}", mapping={
            "lat": lat,
            "lng": lng,
            "timestamp": int(time.time()),
            "status": "delivering"  # idle/delivering/offline
        })
```

**查询附近骑手：**

```python
def get_nearby_riders(self, lat, lng, radius_km=3, count=20):
    city_id = self.get_city_id(lat, lng)
    
    # GEORADIUS 查询 3km 内的骑手
    results = self.redis.georadius(
        f"riders:geo:{city_id}", lng, lat, radius_km,
        unit="km", withdist=True, withcoord=True,
        count=count, sort="ASC"
    )
    
    # 过滤：只返回空闲骑手
    riders = []
    for rider_id, distance, coord in results:
        status = self.redis.hget(f"rider:status:{rider_id}", "status")
        if status == "idle":
            riders.append({
                "rider_id": rider_id,
                "distance": distance,
                "location": coord
            })
    
    return riders
```

**按城市分片的原因：** `GEORADIUS` 在大 Sorted Set 上性能下降。如果全国 50 万骑手在一个 key 中，查询北京 3km 范围仍需扫描大量无关数据。按城市分片后，单城市约 1-5 万骑手，查询极快。

**位置过期清理：** 骑手下线后位置数据应清除，避免查询到已离线的骑手。

```python
# 骑手心跳：每 3 秒更新位置时刷新 TTL
self.redis.expire(f"rider:status:{rider_id}", 30)  # 30 秒过期

# 定时任务：清理过期骑手的 GEO 数据
def cleanup_offline_riders(self, city_id):
    all_riders = self.redis.zrange(f"riders:geo:{city_id}", 0, -1)
    for rider_id in all_riders:
        if not self.redis.exists(f"rider:status:{rider_id}"):
            self.redis.zrem(f"riders:geo:{city_id}", rider_id)
```

### 不规则配送范围：GeoShape

**商家配送范围的存储：**

```json
// Elasticsearch geo_shape 类型
{
  "merchant_id": "M-001",
  "delivery_area": {
    "type": "Polygon",
    "coordinates": [[
      [116.397, 39.916],
      [116.405, 39.918],
      [116.410, 39.914],
      [116.408, 39.908],
      [116.400, 39.905],
      [116.393, 39.910],
      [116.397, 39.916]
    ]]
  }
}
```

**判断用户是否在配送范围内：**

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

**为什么不用简单的圆形范围？**

```
北京某商家：
  3km 圆形范围 → 包含了河对岸的区域（骑手无法过河）
  不规则多边形 → 精确排除河对岸，沿桥梁可到达的区域适当扩展
```

**配送范围的编辑界面：** 运营在地图上绘制多边形，自动简化为 GeoJSON（减少顶点数，提高查询性能）。

### GeoHash 原理与精度

**编码原理：**

```
将经纬度交替编码为二进制：
  经度 116.397 → 二进制 110100101011001...
  纬度  39.916 → 二进制 101110001100011...

交错合并：11100111000010...（奇数位经度，偶数位纬度）

每 5 位编码为一个 Base32 字符：
  wx4g0s... → 这就是 GeoHash
```

**GeoHash 精度与距离：**

| GeoHash 长度 | 精度（约） | 适用场景 |
|-------------|-----------|---------|
| 4 位 | 20km | 省级范围 |
| 5 位 | 2.4km | 区级范围 |
| 6 位 | 610m | 街道范围 |
| 7 位 | 76m | 楼栋范围 |
| 8 位 | 19m | 精确定位 |

**GeoHash 的边界问题：**

```
两个地理位置很近的点，如果正好在 GeoHash 网格的边界两侧：
  点 A: wx4g0s (右侧)
  点 B: wx4g0e (左侧)

它们的 GeoHash 前缀不同，直接按前缀范围查询会漏掉。

解决方案：查询时获取当前 GeoHash 及其 8 个相邻格子的范围，
         合并查询确保不遗漏边界附近的商家。
```

```python
class GeoHashSearch:
    def search_nearby(self, lat, lng, radius_km):
        # 1. 计算当前点的 GeoHash（6位，精度约 610m）
        current_hash = geohash.encode(lat, lng, precision=6)
        
        # 2. 获取相邻 8 个格子的 GeoHash
        neighbors = geohash.neighbors(current_hash)
        all_hashes = [current_hash] + neighbors
        
        # 3. 在 Redis 中按 GeoHash 前缀范围查询
        results = []
        for h in all_hashes:
            # 商家按 GeoHash 前缀索引
            members = self.redis.zrangebyscore(
                f"merchants:geohash:{h[:4]}",  # 4位前缀分区
                geohash.decode(h)[0] - radius_km / 111,  # 纬度范围
                geohash.decode(h)[0] + radius_km / 111
            )
            results.extend(members)
        
        # 4. 精确过滤：计算真实距离，排除超范围的
        filtered = [
            m for m in results
            if geo_distance((lat, lng), m.location) <= radius_km * 1000
        ]
        
        return sorted(filtered, key=lambda m: geo_distance((lat, lng), m.location))
```

### 写入性能计算

**商家数据（相对静态）：**
- 500 万商家，更新频率低（每天约 10 万次更新）
- Elasticsearch 批量写入，10 万次/天 → 约 1 次/秒 → 无压力

**骑手数据（高频动态）：**
- 50 万骑手 × 每 3 秒更新 = 约 17 万次/秒
- Redis GEO `GEOADD`：单节点约 10 万 QPS → 2 节点即可
- 按城市分片后，单城市约 1-5 万骑手，单 key 操作无压力

## 常见陷阱（深度分析）

### 陷阱 1：只用距离排序

**具体问题：** 用户搜"附近美食"，100 米内有一家 2 星苍蝇馆子，1km 外有一家 4.8 星网红餐厅。纯距离排序会把苍蝇馆子排第一。

**解决方案：** 综合排序（距离 + 评分 + 配送时长），权重可动态调整。

### 陷阱 2：GeoHash 不处理边界

**漏查的具体场景：**
- 用户在 wx4g0s 网格的左边缘
- 最近的一家餐厅在 wx4g0e 网格的右边缘（物理距离 < 100m）
- 只查 wx4g0s 前缀 → 漏掉这家餐厅

**必须查询 9 个格子（当前 + 8 邻居）。**

### 陷阱 3：骑手位置不清理

**问题：** 骑手下班后，GEO 数据仍在 Redis 中。查询附近骑手时返回了已下线的骑手，导致派单失败。

**解决方案：** 骑手状态 key 设置 30 秒 TTL，GEO 定时清理。

### 陷阱 4：配送范围用圆形

**问题场景：**
```
商家在河东侧，3km 圆形范围包含河西侧 → 无法配送
骑手过桥需要绕行 8km → 实际配送时间 40 分钟（而非预计的 15 分钟）
```

**解决方案：** 用不规则多边形（geo_shape）精确描述配送范围。

## 延伸思考

- **热力图渲染**：某区域的订单密度如何实时计算？用 GeoHash 聚合（6 位精度，610m 网格）+ Flink 实时统计。
- **室内定位**：商场内部的商家定位（GPS 精度 10-50m 不够），需要蓝牙信标或 WiFi 指纹定位，架构如何扩展？
- **跨城配送**：城际物流的距离计算不是直线距离而是路线距离，需要接入地图 API（高德/百度）获取实际行驶距离。