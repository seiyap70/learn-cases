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

## 数据库设计

### 商家表（merchants）

```sql
CREATE TABLE merchants (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    merchant_no     VARCHAR(32) NOT NULL UNIQUE COMMENT '商家编号 M-XXXXX',
    name            VARCHAR(128) NOT NULL COMMENT '商家名称',
    category        VARCHAR(32) NOT NULL COMMENT '分类: chinese_food/japanese/fast_food/...',
    brand_id        BIGINT COMMENT '品牌ID，连锁店共用',
    lat             DECIMAL(10,7) NOT NULL COMMENT '纬度',
    lng             DECIMAL(10,7) NOT NULL COMMENT '经度',
    location        POINT NOT NULL SRID 4326 COMMENT '空间点，用于空间索引',
    address         VARCHAR(256) NOT NULL COMMENT '详细地址',
    city_id         INT NOT NULL COMMENT '城市ID',
    district_id     INT NOT NULL COMMENT '区域ID',
    rating          DECIMAL(2,1) NOT NULL DEFAULT 0.0 COMMENT '评分 0-5',
    monthly_orders  INT NOT NULL DEFAULT 0 COMMENT '月订单量',
    avg_delivery_minutes INT NOT NULL DEFAULT 0 COMMENT '平均配送时长(分钟)',
    min_order_amount DECIMAL(8,2) NOT NULL DEFAULT 0.00 COMMENT '起送价',
    delivery_fee    DECIMAL(8,2) NOT NULL DEFAULT 0.00 COMMENT '配送费',
    is_open         TINYINT(1) NOT NULL DEFAULT 1 COMMENT '是否营业',
    open_time       TIME COMMENT '营业开始时间',
    close_time      TIME COMMENT '营业结束时间',
    phone           VARCHAR(20) COMMENT '联系电话',
    status          TINYINT NOT NULL DEFAULT 1 COMMENT '1-正常 2-休息 3-下线',
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    SPATIAL INDEX idx_location (location),
    INDEX idx_city_category (city_id, category, is_open),
    INDEX idx_brand (brand_id),
    INDEX idx_rating (rating DESC),
    INDEX idx_monthly_orders (monthly_orders DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商家表';
```

### 骑手表（riders）

```sql
CREATE TABLE riders (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    rider_no        VARCHAR(32) NOT NULL UNIQUE COMMENT '骑手编号 R-XXXXX',
    name            VARCHAR(64) NOT NULL COMMENT '姓名',
    phone           VARCHAR(20) NOT NULL UNIQUE COMMENT '手机号',
    city_id         INT NOT NULL COMMENT '所属城市',
    work_status     TINYINT NOT NULL DEFAULT 0 COMMENT '0-离线 1-空闲 2-配送中 3-休息',
    vehicle_type    TINYINT NOT NULL DEFAULT 1 COMMENT '1-电动车 2-自行车 3-步行',
    current_lat     DECIMAL(10,7) COMMENT '当前纬度(缓存)',
    current_lng     DECIMAL(10,7) COMMENT '当前经度(缓存)',
    last_heartbeat  DATETIME COMMENT '最后心跳时间',
    total_orders    INT NOT NULL DEFAULT 0 COMMENT '累计完成订单',
    today_orders    INT NOT NULL DEFAULT 0 COMMENT '今日完成订单',
    rating          DECIMAL(2,1) NOT NULL DEFAULT 5.0 COMMENT '骑手评分',
    status          TINYINT NOT NULL DEFAULT 1 COMMENT '1-正常 2-冻结 3-离职',
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_city_status (city_id, work_status),
    INDEX idx_heartbeat (last_heartbeat)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='骑手表';
```

### 配送范围表（delivery_areas）

```sql
CREATE TABLE delivery_areas (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    merchant_id     BIGINT NOT NULL COMMENT '商家ID',
    area_type       TINYINT NOT NULL DEFAULT 1 COMMENT '1-主配送区 2-扩展配送区',
    polygon         POLYGON NOT NULL SRID 4326 COMMENT '配送范围多边形',
    center_lat      DECIMAL(10,7) NOT NULL COMMENT '范围中心纬度',
    center_lng      DECIMAL(10,7) NOT NULL COMMENT '范围中心经度',
    max_distance_m  INT NOT NULL COMMENT '最远配送距离(米)',
    avg_delivery_min INT NOT NULL COMMENT '该范围平均配送时长(分钟)',
    vertex_count    INT NOT NULL COMMENT '多边形顶点数',
    is_active       TINYINT(1) NOT NULL DEFAULT 1 COMMENT '是否生效',
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    SPATIAL INDEX idx_polygon (polygon),
    INDEX idx_merchant (merchant_id, is_active),
    INDEX idx_center (center_lat, center_lng),

    CONSTRAINT fk_delivery_area_merchant FOREIGN KEY (merchant_id) REFERENCES merchants(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='配送范围表';
```

### 骑手轨迹表（rider_tracks）

```sql
-- 按月分表: rider_tracks_202601, rider_tracks_202602, ...
CREATE TABLE rider_tracks_202606 (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    rider_id        BIGINT NOT NULL COMMENT '骑手ID',
    order_id        BIGINT COMMENT '关联订单ID(配送中时有值)',
    lat             DECIMAL(10,7) NOT NULL COMMENT '纬度',
    lng             DECIMAL(10,7) NOT NULL COMMENT '经度',
    accuracy        SMALLINT NOT NULL DEFAULT 0 COMMENT 'GPS精度(米)',
    speed           SMALLINT NOT NULL DEFAULT 0 COMMENT '速度(km/h)',
    bearing         SMALLINT COMMENT '方向角(0-359度)',
    altitude        SMALLINT COMMENT '海拔(米)',
    source          TINYINT NOT NULL DEFAULT 1 COMMENT '1-GPS 2-WiFi 3-基站 4-蓝牙',
    battery_level   TINYINT COMMENT '电量百分比',
    created_at      DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3) COMMENT '毫秒精度时间戳',

    INDEX idx_rider_time (rider_id, created_at),
    INDEX idx_order (order_id),
    INDEX idx_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='骑手轨迹表-按月分表';

-- 归档策略: 3个月以上数据迁移至 HDFS/OSS 冷存储
-- 查询策略: 近3个月查MySQL，历史数据查HDFS/Hive
```

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

**完整索引设计（含所有字段映射与调优参数）：**

```json
PUT /merchants
{
  "mappings": {
    "properties": {
      "merchant_no": { "type": "keyword", "doc_values": true },
      "name": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 128 },
          "pinyin": { "type": "text", "analyzer": "pinyin_analyzer" }
        }
      },
      "category": { "type": "keyword" },
      "sub_category": { "type": "keyword" },
      "brand_id": { "type": "long" },
      "city_id": { "type": "integer" },
      "district_id": { "type": "integer" },
      "location": {
        "type": "geo_point",
        "ignore_malformed": true
      },
      "address": { "type": "text", "analyzer": "ik_max_word" },
      "rating": { "type": "float", "coerce": true },
      "monthly_orders": { "type": "integer", "coerce": true },
      "avg_delivery_minutes": { "type": "integer" },
      "min_order_amount": { "type": "float" },
      "delivery_fee": { "type": "float" },
      "is_open": { "type": "boolean" },
      "open_time": { "type": "keyword" },
      "close_time": { "type": "keyword" },
      "tags": { "type": "keyword" },
      "promo_tags": {
        "type": "keyword",
        "eager_global_ordinals": true
      },
      "delivery_area": {
        "type": "geo_shape",
        "strategy": "recursive",
        "tree": "quadtree",
        "tree_levels": 12,
        "distance_error_pct": 0.025
      },
      "priority_weight": { "type": "float", "default": 1.0 },
      "status": { "type": "integer" },
      "updated_at": { "type": "date", "format": "yyyy-MM-dd HH:mm:ss||epoch_millis" }
    }
  },
  "settings": {
    "number_of_shards": 20,
    "number_of_replicas": 1,
    "index.sort.field": ["monthly_orders", "rating"],
    "index.sort.order": ["desc", "desc"],
    "refresh_interval": "5s",
    "translog.durability": "async",
    "translog.sync_interval": "5s",
    "translog.flush_threshold_size": "512mb",
    "merge.scheduler.max_thread_count": 1,
    "analysis": {
      "analyzer": {
        "pinyin_analyzer": {
          "tokenizer": "pinyin_tokenizer"
        }
      },
      "tokenizer": {
        "pinyin_tokenizer": {
          "type": "pinyin",
          "keep_first_letter": true,
          "keep_separate_first_letter": false,
          "keep_full_pinyin": true,
          "keep_original": true,
          "limit_first_letter_length": 16,
          "lowercased": true
        }
      }
    },
    "index.max_result_window": 5000
  }
}
```

**索引设计要点解析：**

| 参数 | 值 | 原因 |
|------|---|------|
| `number_of_shards` | 20 | 500万商家，每 shard 约25万文档，保证单 shard 查询 < 50ms |
| `number_of_replicas` | 1 | 高可用，允许1节点故障；午高峰可临时调整为0提升写入吞吐 |
| `index.sort.field` | monthly_orders + rating | 早终止优化：查询只需前20条时，ES可在找到足够高分文档后立即停止扫描 |
| `refresh_interval` | 5s | 商家数据不要求毫秒级实时性，降低刷新频率减少 segment merge 开销 |
| `translog.durability` | async | 商家数据变更频率低，允许5秒内丢失（可接受），提升写入性能 |
| `geo_shape.tree_levels` | 12 | quadtree 12层 → 精度约 76m，足够判定配送范围边界 |
| `eager_global_ordinals` | promo_tags | 午高峰频繁按促销标签过滤，预构建全局序数避免每次查询重建 |
| `ignore_malformed` | true (geo_point) | 防止个别商家坐标异常导致整条文档写入失败 |
| `name.fields.pinyin` | | 支持拼音搜索"ml"匹配"麦当劳"，提升用户体验 |

**骑手索引设计（用于骑手搜索与行为分析）：**

```json
PUT /riders
{
  "mappings": {
    "properties": {
      "rider_no": { "type": "keyword" },
      "city_id": { "type": "integer" },
      "name": { "type": "keyword" },
      "location": { "type": "geo_point" },
      "work_status": { "type": "integer" },
      "current_order_id": { "type": "keyword" },
      "rating": { "type": "float" },
      "total_orders": { "type": "integer" },
      "vehicle_type": { "type": "integer" },
      "updated_at": { "type": "date" }
    }
  },
  "settings": {
    "number_of_shards": 10,
    "number_of_replicas": 0,
    "refresh_interval": "1s",
    "translog.durability": "async",
    "translog.sync_interval": "1s"
  }
}
```

骑手索引设计考虑：副本设为 0（骑手位置的主数据在 Redis，ES 只做备份查询），refresh_interval 1s（比商家更频繁，骑手位置需要更高的实时性）。

**中文地址搜索自定义分析器：**

商家搜索不仅需要按名称搜索，还需要支持用户输入中文地址（如"望京SOHO"）或地址片段（如"望京"）。需要定制分析器解决中文分词、地址切分、同义词扩展等问题。

```json
PUT /merchants
{
  "settings": {
    "analysis": {
      "char_filter": {
        "address_char_filter": {
          "type": "mapping",
          "mappings": [
            "号楼 => 栋",
            "幢 => 栋",
            "号院 => 小区",
            "路 => 街道",
            "大道 => 街道"
          ]
        }
      },
      "tokenizer": {
        "pinyin_tokenizer": {
          "type": "pinyin",
          "keep_first_letter": true,
          "keep_separate_first_letter": false,
          "keep_full_pinyin": true,
          "keep_original": true,
          "limit_first_letter_length": 16,
          "lowercased": true
        },
        "address_ngram_tokenizer": {
          "type": "ngram",
          "min_gram": 2,
          "max_gram": 4,
          "token_chars": ["letter", "digit"]
        }
      },
      "analyzer": {
        "ik_pinyin_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word",
          "filter": ["pinyin_filter", "lowercase"]
        },
        "address_search_analyzer": {
          "type": "custom",
          "tokenizer": "ik_smart",
          "char_filter": ["address_char_filter"],
          "filter": ["synonym_filter", "lowercase"]
        },
        "address_index_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word",
          "char_filter": ["address_char_filter"],
          "filter": ["synonym_filter", "lowercase"]
        },
        "pinyin_analyzer": {
          "type": "custom",
          "tokenizer": "pinyin_tokenizer"
        }
      },
      "filter": {
        "pinyin_filter": {
          "type": "pinyin",
          "keep_first_letter": true,
          "keep_full_pinyin": false,
          "keep_original": false,
          "min_item_length": 2
        },
        "synonym_filter": {
          "type": "synonym",
          "synonyms_path": "analysis/synonyms.txt",
          "updateable": true
        }
      }
    }
  }
}
```

同义词配置文件 `analysis/synonyms.txt` 内容示例：

```
望京SOHO,望京soho,望京大厦 => 望京SOHO
三里屯,三里屯太古里,三里屯village => 三里屯
国贸,国贸中心,CBD => 国贸CBD
中关村,中关村科技园,硅谷 => 中关村
美食,餐饮,吃饭,小吃 => 美食
快餐,便当,简餐 => 快餐
奶茶,饮品,饮料,茶饮 => 奶茶
```

**地址字段映射（使用自定义分析器）：**

```json
{
  "address": {
    "type": "text",
    "analyzer": "address_index_analyzer",
    "search_analyzer": "address_search_analyzer",
    "fields": {
      "keyword": { "type": "keyword", "ignore_above": 256 },
      "pinyin": {
        "type": "text",
        "analyzer": "ik_pinyin_analyzer",
        "search_analyzer": "pinyin_analyzer"
      },
      "ngram": {
        "type": "text",
        "analyzer": "address_ngram_tokenizer"
      }
    }
  }
}
```

**索引模板与别名策略（零停机重建索引）：**

```json
PUT _index_template/merchants_template
{
  "index_patterns": ["merchants_*"],
  "priority": 100,
  "template": {
    "mappings": {
      "properties": {
        "merchant_no": { "type": "keyword", "doc_values": true },
        "name": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_smart",
          "fields": {
            "keyword": { "type": "keyword", "ignore_above": 128 },
            "pinyin": { "type": "text", "analyzer": "pinyin_analyzer" }
          }
        },
        "location": { "type": "geo_point", "ignore_malformed": true },
        "delivery_area": {
          "type": "geo_shape",
          "strategy": "recursive",
          "tree": "quadtree",
          "tree_levels": 12,
          "distance_error_pct": 0.025
        }
      }
    },
    "settings": {
      "number_of_shards": 20,
      "number_of_replicas": 1,
      "index.sort.field": ["monthly_orders", "rating"],
      "index.sort.order": ["desc", "desc"],
      "refresh_interval": "5s",
      "translog.durability": "async",
      "translog.sync_interval": "5s"
    }
  }
}
```

零停机重建索引流程：

```python
class ZeroDowntimeReindex:
    """ES 零停机重建索引（修改 mapping 时必须重建）"""

    def reindex_merchants(self):
        # 1. 创建新索引（自动匹配模板）
        new_index = f"merchants_{int(time.time())}"
        es_client.indices.create(index=new_index)

        # 2. 原地重建（使用 _reindex API，内部滚动方式避免OOM）
        es_client.reindex(
            body={
                "source": {"index": "merchants"},  # 当前别名指向的索引
                "dest": {"index": new_index},
                "conflicts": "proceed"  # 版本冲突跳过（避免被并发写入阻塞）
            },
            wait_for_completion=False,  # 异步执行
            request_timeout=10
        )
        # 等待重建完成（通过 _tasks API 轮询）

        # 3. 切换别名（原子操作）
        es_client.indices.update_aliases(body={
            "actions": [
                {"remove": {"index": "merchants_*", "alias": "merchants"}},
                {"add": {"index": new_index, "alias": "merchants"}}
            ]
        })

        # 4. 删除旧索引
        # 确认新索引正常后再删除
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

### 骑手位置：Redis GEO + 按城市分片 + Kafka 流式架构

**完整骑手位置更新流程架构：**

```
骑手APP → API网关 → 位置写入服务 → Redis GEO（实时查询）
                                  → Kafka（轨迹流 + 下游消费）
                                       ├→ Flink → 实时聚合（热力图/骑手分布统计）
                                       ├→ Flink → 偏离检测（配送路线偏移告警）
                                       ├→ Flink → ES 同步（骑手位置备份索引）
                                       └→ HDFS/Hive → 轨迹归档（争议仲裁/行为分析）
```

**Kafka Topic 设计：**

```
Topic: rider_location
  Partitions: 60（按 rider_id % 60 分区，保证同一骑手数据有序）
  Replication: 3
  Retention: 72h（3天后自动清理，归档由Flink写入HDFS）
  Compression: lz4（位置数据重复字段多，压缩比约 4:1）
  max.message.bytes: 1MB
  Producer: acks=1（允许1副本确认即可，优先吞吐量）

Topic: rider_location_compact
  Partitions: 60
  Replication: 3
  Cleanup Policy: compact（保留每个 rider_id 最新一条）
  Purpose: 消费者可获取骑手最新位置快照，无需遍历全部消息
```

**Kafka Producer 配置与写入逻辑：**

```python
from confluent_kafka import Producer
import json

class RiderLocationProducer:
    def __init__(self):
        self.producer = Producer({
            'bootstrap.servers': 'kafka1:9092,kafka2:9092,kafka3:9092',
            'acks': 1,                    # 1副本确认，优先吞吐
            'compression.type': 'lz4',    # 压缩比好，CPU消耗低
            'linger.ms': 20,              # 等待20ms批量发送
            'batch.size': 65536,          # 64KB批量
            'max.in.flight.requests.per.connection': 5,
            'retries': 3,
            'retry.backoff.ms': 100,
            'enable.idempotence': True    # 精确一次语义，防止重复
        })
        self.topic = 'rider_location'
        self.compact_topic = 'rider_location_compact'

    def publish(self, rider_id, lat, lng, accuracy, speed, source):
        """发布骑手位置到 Kafka"""
        msg = {
            "rider_id": rider_id,
            "lat": lat,
            "lng": lng,
            "accuracy": accuracy,        # GPS精度(米)
            "speed": speed,              # 速度(km/h)
            "source": source,            # GPS/WiFi/基站
            "timestamp": int(time.time() * 1000)
        }
        # 按rider_id分区，保证同一骑手消息有序
        payload = json.dumps(msg).encode('utf-8')
        self.producer.produce(
            self.topic,
            key=str(rider_id).encode('utf-8'),
            value=payload
        )
        # 写入compact topic用于最新位置快照
        self.producer.produce(
            self.compact_topic,
            key=str(rider_id).encode('utf-8'),
            value=payload
        )
        self.producer.poll(0)  # 非阻塞触发回调
```

**Flink 消费 Kafka 实现实时处理：**

```python
"""
Flink Job: 骑手位置实时处理
功能：1) 同步ES  2) 偏离检测  3) 聚合统计
"""
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.connectors import FlinkKafkaConsumer
from pyflink.common import SimpleStringSchema

class RiderLocationFlinkJob:
    def run(self):
        env = StreamExecutionEnvironment.get_execution_environment()
        env.set_parallelism(60)  # 与Kafka分区数一致

        # 消费Kafka
        kafka_source = FlinkKafkaConsumer(
            topics='rider_location',
            deserialization_schema=SimpleStringSchema(),
            properties={
                'bootstrap.servers': 'kafka1:9092,kafka2:9092',
                'group.id': 'rider-location-processor',
                'auto.offset.reset': 'latest'
            }
        )
        stream = env.add_source(kafka_source)

        # 处理1: 同步骑手位置到ES（低频，每5秒批量）
        (stream
         .map(self.parse_location)
         .key_by(lambda x: x['city_id'])
         .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
         .process(self.batch_sync_to_es))

        # 处理2: 检测骑手偏离配送路线
        (stream
         .key_by(lambda x: x['rider_id'])
         .map(self.detect_route_deviation))

        # 处理3: GeoHash聚合统计 → 写Redis热力图
        (stream
         .map(self.to_geohash_aggregation)
         .key_by(lambda x: x['geohash_prefix'])
         .window(TumblingProcessingTimeWindows.of(Time.seconds(10)))
         .reduce(self.aggregate_count)
         .add_sink(self.write_heatmap_to_redis))

        env.execute('rider-location-processor')

    def batch_sync_to_es(self, key, window, locations):
        """批量同步到ES，避免逐条写入"""
        bulk_body = []
        for loc in locations:
            action = {"index": {"_index": "riders", "_id": loc['rider_id']}}
            doc = {
                "rider_no": loc['rider_id'],
                "location": {"lat": loc['lat'], "lon": loc['lng']},
                "city_id": loc['city_id'],
                "work_status": loc['status'],
                "updated_at": loc['timestamp']
            }
            bulk_body.extend([action, doc])
        es_client.bulk(body=bulk_body, refresh=False)

    def detect_route_deviation(self, location):
        """检测骑手是否偏离预计配送路线（超过500米触发告警）"""
        rider_id = location['rider_id']
        route = self.get_expected_route(rider_id)  # 从Redis获取预计路线
        if route is None:
            return location

        # 找到路线上距离最近的点
        min_dist = float('inf')
        for point in route['waypoints']:
            d = geo_distance(
                (location['lat'], location['lng']),
                (point['lat'], point['lng'])
            )
            min_dist = min(min_dist, d)

        if min_dist > 500:  # 偏离超过500米
            self.alert_deviation(rider_id, min_dist, location)
        return location
```

**完整骑手位置流式管线（Kafka → Flink → Redis GEO → ES）：**

```
骑手APP ──(3s心跳)──→ API网关 ──→ 位置写入服务
                                      │
                                      ├─→ Redis GEO（实时查询，P99 < 5ms）
                                      │     └─ riders:geo:{city_id}
                                      │     └─ rider:status:{rider_id}
                                      │
                                      └─→ Kafka topic: rider_location
                                              │
                                              ├→ Flink Job 1: Redis GEO 同步（核心路径）
                                              │     └─ 消费 → 去重 → 按 city_id 分组
                                              │        → 批量 GEORADIUS 更新 → Redis
                                              │
                                              ├→ Flink Job 2: ES 索引同步
                                              │     └─ 消费 → 5s 窗口聚合 → Bulk API
                                              │
                                              ├→ Flink Job 3: 热力图聚合
                                              │     └─ 消费 → GeoHash 聚合 → Redis INCR
                                              │
                                              └→ Flink Job 4: 轨迹归档
                                                    └─ 消费 → Parquet 格式 → HDFS/OSS
```

**Flink Job 1：核心 Redis GEO 同步（完整 Java 实现）：**

```java
/**
 * Flink Job: 骑手位置 Redis GEO 同步
 * 功能：消费 Kafka 位置数据，批量写入 Redis GEO，保证实时查询数据新鲜度
 */
public class RiderLocationRedisSyncJob {

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(60);  // 与 Kafka 分区数一致
        env.enableCheckpointing(30000);  // 30秒 Checkpoint，Exactly-Once 语义

        // 1. Kafka Source
        KafkaSource<String> source = KafkaSource.<String>builder()
            .setBootstrapServers("kafka1:9092,kafka2:9092,kafka3:9092")
            .setTopics("rider_location")
            .setGroupId("redis-sync-group")
            .setStartingOffsets(OffsetsInitializer.latest())
            .setValueOnlyDeserializer(new SimpleStringSchema())
            .build();

        DataStream<String> rawStream = env.fromSource(
            source, WatermarkStrategy.noWatermarks(), "RiderLocationSource"
        );

        // 2. 解析 & 去重（同一骑手3秒内可能有重复上报）
        DataStream<RiderLocation> locationStream = rawStream
            .map(new LocationParser())
            .keyBy(loc -> loc.riderId)
            .process(new DeduplicateFunction());  // 按 riderId 去重，只保留最新

        // 3. 按城市分组，1秒窗口批量写入 Redis
        locationStream
            .keyBy(loc -> loc.cityId)
            .window(TumblingProcessingTimeWindows.of(Time.seconds(1)))
            .process(new RedisGeoBatchSink());

        env.execute("RiderLocation-RedisSync");
    }

    /** 位置数据解析器 */
    static class LocationParser implements MapFunction<String, RiderLocation> {
        private static final ObjectMapper mapper = new ObjectMapper();

        @Override
        public RiderLocation map(String value) throws Exception {
            JsonNode node = mapper.readTree(value);
            return new RiderLocation(
                node.get("rider_id").asLong(),
                node.get("lat").asDouble(),
                node.get("lng").asDouble(),
                node.get("accuracy").asInt(),
                node.get("speed").asInt(),
                node.get("source").asText(),
                node.get("timestamp").asLong(),
                CityBoundaryService.getCityId(
                    node.get("lat").asDouble(),
                    node.get("lng").asDouble()
                )
            );
        }
    }

    /** 去重函数：同一骑手在窗口内只保留最新位置 */
    static class DeduplicateFunction extends KeyedProcessFunction<Long, RiderLocation, RiderLocation> {
        private ValueState<RiderLocation> latestState;

        @Override
        public void open(Configuration parameters) {
            latestState = getRuntimeContext().getState(
                new ValueStateDescriptor<>("latest", RiderLocation.class)
            );
        }

        @Override
        public void processElement(RiderLocation loc, Context ctx, Collector<RiderLocation> out)
                throws Exception {
            RiderLocation prev = latestState.value();
            if (prev == null || loc.timestamp > prev.timestamp) {
                latestState.update(loc);
                out.collect(loc);
            }
            // 旧数据丢弃，不输出
        }
    }

    /** Redis GEO 批量写入 Sink */
    static class RedisGeoBatchSink extends ProcessWindowFunction<RiderLocation, String, Integer, TimeWindow> {
        private transient JedisPool jedisPool;

        @Override
        public void open(Configuration parameters) {
            jedisPool = new JedisPool(new JedisPoolConfig(),
                "redis-cluster-endpoint", 6379, 2000, null);
        }

        @Override
        public void process(Integer cityId, Context ctx,
                            Iterable<RiderLocation> locations, Collector<String> out) {
            try (Jedis jedis = jedisPool.getResource()) {
                Pipeline pipeline = jedis.pipelined();
                String geoKey = "riders:geo:" + cityId;

                int count = 0;
                for (RiderLocation loc : locations) {
                    // GEORADIUS 更新：加入 Sorted Set（经度作为score）
                    pipeline.geoadd(geoKey, loc.lng, loc.lat, String.valueOf(loc.riderId));

                    // 同时更新骑手状态 Hash
                    String statusKey = "rider:status:" + loc.riderId;
                    Map<String, String> statusMap = new HashMap<>();
                    statusMap.put("lat", String.valueOf(loc.lat));
                    statusMap.put("lng", String.valueOf(loc.lng));
                    statusMap.put("accuracy", String.valueOf(loc.accuracy));
                    statusMap.put("speed", String.valueOf(loc.speed));
                    statusMap.put("timestamp", String.valueOf(loc.timestamp));
                    statusMap.put("city_id", String.valueOf(cityId));
                    pipeline.hset(statusKey, statusMap);
                    pipeline.expire(statusKey, 30);

                    count++;
                }

                pipeline.sync();  // 一次性提交
                out.collect("City " + cityId + ": synced " + count + " riders");
            }
        }
    }

    /** 位置数据 POJO */
    static class RiderLocation implements Serializable {
        public long riderId;
        public double lat, lng;
        public int accuracy, speed;
        public String source;
        public long timestamp;
        public int cityId;

        public RiderLocation() {}  // Flink 反序列化需要

        public RiderLocation(long riderId, double lat, double lng, int accuracy,
                             int speed, String source, long timestamp, int cityId) {
            this.riderId = riderId;
            this.lat = lat;
            this.lng = lng;
            this.accuracy = accuracy;
            this.speed = speed;
            this.source = source;
            this.timestamp = timestamp;
            this.cityId = cityId;
        }
    }
}
```

**Flink Job 2：ES 索引同步（Java 实现）：**

```java
/**
 * Flink Job: 骑手位置 ES 索引同步
 * 功能：5秒窗口批量 Bulk 写入 ES，用于骑手搜索和后台管理
 */
public class RiderLocationESSyncJob {

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(20);
        env.enableCheckpointing(30000);

        KafkaSource<String> source = KafkaSource.<String>builder()
            .setBootstrapServers("kafka1:9092,kafka2:9092,kafka3:9092")
            .setTopics("rider_location_compact")  // 使用 compact topic，数据量更小
            .setGroupId("es-sync-group")
            .setStartingOffsets(OffsetsInitializer.latest())
            .setValueOnlyDeserializer(new SimpleStringSchema())
            .build();

        DataStream<String> rawStream = env.fromSource(
            source, WatermarkStrategy.noWatermarks(), "RiderLocationCompactSource"
        );

        rawStream
            .map(new LocationParser())
            .keyBy(loc -> loc.cityId)
            .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
            .process(new ESBatchSink());

        env.execute("RiderLocation-ESSync");
    }

    static class ESBatchSink extends ProcessWindowFunction<RiderLocation, String, Integer, TimeWindow> {
        private transient RestHighLevelClient esClient;

        @Override
        public void open(Configuration parameters) {
            esClient = new RestHighLevelClient(
                RestClient.builder(
                    new HttpHost("es-node1", 9200, "http"),
                    new HttpHost("es-node2", 9200, "http"),
                    new HttpHost("es-node3", 9200, "http")
                )
            );
        }

        @Override
        public void process(Integer cityId, Context ctx,
                            Iterable<RiderLocation> locations, Collector<String> out) throws Exception {
            BulkRequest bulkRequest = new BulkRequest();
            int count = 0;

            for (RiderLocation loc : locations) {
                // ES 文档 ID = rider_id，实现 upsert 语义
                Map<String, Object> doc = new HashMap<>();
                doc.put("rider_no", "R-" + loc.riderId);
                doc.put("city_id", loc.cityId);
                doc.put("location", new double[]{loc.lng, loc.lat});  // geo_point: [lon, lat]
                doc.put("work_status", loc.workStatus);
                doc.put("updated_at", Instant.ofEpochMilli(loc.timestamp).toString());

                IndexRequest request = new IndexRequest("riders")
                    .id(String.valueOf(loc.riderId))
                    .source(doc);
                bulkRequest.add(request);
                count++;
            }

            if (count > 0) {
                bulkRequest.setRefreshPolicy(WriteRequest.RefreshPolicy.NONE);  // 不立即刷新
                BulkResponse response = esClient.bulk(bulkRequest, RequestOptions.DEFAULT);
                if (response.hasFailures()) {
                    log.error("ES bulk sync failures: {}", response.buildFailureMessage());
                }
                out.collect("Synced " + count + " riders to ES");
            }
        }
    }
}
```

**Kafka 消费者组管理（Python 运维工具）：**

```python
class KafkaConsumerManager:
    """Kafka 消费者组运维工具"""

    def __init__(self, bootstrap_servers):
        self.admin_client = KafkaAdminClient(bootstrap_servers=bootstrap_servers)
        self.consumer = KafkaConsumer(bootstrap_servers=bootstrap_servers)

    def get_consumer_lag(self, group_id, topic):
        """检查消费者组延迟（关键监控指标）"""
        partitions = self.consumer.partitions_for_topic(topic)
        lag_info = {}

        for partition in partitions:
            tp = TopicPartition(topic, partition)
            # 消费者组的当前位移
            committed = self.consumer.committed(tp)
            # 分区的最新位移
            end_offset = self.consumer.end_offsets([tp])[tp]

            if committed is not None:
                lag = end_offset - committed
                lag_info[partition] = {
                    "committed_offset": committed,
                    "end_offset": end_offset,
                    "lag": lag,
                    "lag_seconds": lag / 56000  # 大约每秒56000条（17万/3秒）
                }

        total_lag = sum(v["lag"] for v in lag_info.values())
        max_lag_seconds = max(v["lag_seconds"] for v in lag_info.values())

        return {
            "group_id": group_id,
            "topic": topic,
            "total_lag": total_lag,
            "max_lag_seconds": max_lag_seconds,
            "partitions": lag_info,
            "healthy": max_lag_seconds < 5  # 延迟超过5秒告警
        }

    def rebalance_consumer_group(self, group_id):
        """强制消费者组再平衡（处理消费者假死）"""
        # 删除消费者组的位移提交，触发再平衡
        try:
            self.admin_client.delete_consumer_groups([group_id])
            log.info(f"Consumer group {group_id} rebalanced")
        except Exception as e:
            log.error(f"Failed to rebalance group {group_id}: {e}")
```

**位置写入服务完整实现：**

```python
class RiderLocationService:
    """骑手位置服务 - 完整实现"""

    def __init__(self, redis, kafka_producer):
        self.redis = redis
        self.kafka = kafka_producer
        self.city_boundary = CityBoundaryService()  # 城市边界服务

    def update_location(self, rider_id, lat, lng, accuracy=0, speed=0, source='gps'):
        """骑手位置更新（每 3 秒调用一次）"""

        # 0. GPS漂移检测与修正
        lat, lng = self._filter_gps_drift(rider_id, lat, lng, accuracy, speed)

        city_id = self._get_city_id(lat, lng)

        # 1. 更新骑手 GEO 位置
        self.redis.geoadd(f"riders:geo:{city_id}", lng, lat, rider_id)

        # 2. 更新骑手状态（带 TTL，30 秒过期 = 10 次心跳未更新则视为离线）
        self.redis.hset(f"rider:status:{rider_id}", mapping={
            "lat": lat, "lng": lng,
            "accuracy": accuracy,
            "speed": speed,
            "timestamp": int(time.time()),
            "status": "delivering",
            "city_id": city_id
        })
        self.redis.expire(f"rider:status:{rider_id}", 30)

        # 3. 轨迹写入 Kafka（异步，不影响位置更新延迟）
        self.kafka.publish(rider_id, lat, lng, accuracy, speed, source)

    def _filter_gps_drift(self, rider_id, lat, lng, accuracy, speed):
        """GPS漂移检测与修正

        漂移场景：
        1. 高楼遮挡 → GPS信号反射 → 位置跳变几百米
        2. 地下通道 → GPS丢失 → 突然定位到几公里外
        3. WiFi定位切换 → 位置跳变
        """
        prev = self.redis.hgetall(f"rider:status:{rider_id}")
        if not prev:
            return lat, lng

        prev_lat, prev_lng = float(prev[b'lat']), float(prev[b'lng'])
        prev_time = float(prev[b'timestamp'])

        # 计算位移距离
        distance = geo_distance((prev_lat, prev_lng), (lat, lng))
        time_delta = time.time() - prev_time

        # 物理可行性判断：骑手最高速度约60km/h = 16.7m/s
        max_possible_speed = 16.7  # m/s
        max_possible_distance = max_possible_speed * time_delta * 1.5  # 1.5倍容差

        if distance > max_possible_distance and distance > 100:
            # 超出物理可能 → GPS漂移
            # 策略：使用前一个位置做线性插值，而非直接采信新位置
            if time_delta > 0:
                drift_ratio = min(max_possible_distance / distance, 1.0)
                corrected_lat = prev_lat + (lat - prev_lat) * drift_ratio
                corrected_lng = prev_lng + (lng - prev_lng) * drift_ratio
                return corrected_lat, corrected_lng

        # GPS精度差（>100米）时降低权重：取加权平均
        if accuracy > 100 and distance < 200:
            weight = max(0.3, 1.0 - accuracy / 300)  # 精度越差权重越低
            lat = prev_lat + (lat - prev_lat) * weight
            lng = prev_lng + (lng - prev_lng) * weight

        return lat, lng

    def _get_city_id(self, lat, lng):
        """根据坐标确定城市ID（使用城市边界多边形判定）"""
        return self.city_boundary.get_city_id(lat, lng)

    def get_nearby_riders(self, lat, lng, radius_km=3, count=20, status='idle'):
        """查询附近骑手（支持状态过滤）"""
        city_id = self._get_city_id(lat, lng)

        results = self.redis.georadius(
            f"riders:geo:{city_id}", lng, lat, radius_km,
            unit="km", withdist=True, withcoord=True,
            count=count * 3,  # 多取 3 倍（部分可能不空闲）
            sort="ASC"
        )

        riders = []
        for rider_id, distance, coord in results:
            rider_status = self.redis.hget(f"rider:status:{rider_id}", "status")
            if rider_status == status:
                riders.append({
                    "rider_id": rider_id,
                    "distance": round(distance, 2),
                    "lat": coord[1],
                    "lng": coord[0]
                })
            if len(riders) >= count:
                break

        return riders

    def batch_update_locations(self, updates):
        """批量位置更新（API网关聚合后批量写入，减少Redis往返）

        updates: [(rider_id, lat, lng, accuracy, speed, source), ...]
        """
        pipeline = self.redis.pipeline()

        for rider_id, lat, lng, accuracy, speed, source in updates:
            lat, lng = self._filter_gps_drift(rider_id, lat, lng, accuracy, speed)
            city_id = self._get_city_id(lat, lng)

            pipeline.geoadd(f"riders:geo:{city_id}", lng, lat, rider_id)
            pipeline.hset(f"rider:status:{rider_id}", mapping={
                "lat": lat, "lng": lng,
                "accuracy": accuracy,
                "speed": speed,
                "timestamp": int(time.time()),
                "status": "delivering",
                "city_id": city_id
            })
            pipeline.expire(f"rider:status:{rider_id}", 30)

        pipeline.execute()  # 一次性提交所有命令

        # Kafka异步写入
        for rider_id, lat, lng, accuracy, speed, source in updates:
            self.kafka.publish(rider_id, lat, lng, accuracy, speed, source)
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

### 配送范围多边形判定：点在多边形内算法

**核心算法：射线法（Ray Casting）**

原理：从待判定点向右发射一条水平射线，统计与多边形边的交点数量。如果交点数为奇数，点在多边形内；偶数则在多边形外。

**完整实现（含边界点、凹多边形等边界情况处理）：**

```python
import math
from typing import List, Tuple, Optional

class PointInPolygon:
    """点在多边形内判定 - 完整实现

    支持：
    1. 凸多边形和凹多边形
    2. 点在边界上的判定（可配置包含/排除）
    3. 带"洞"的多边形（环状配送区，如排除小区内部不可达区域）
    4. GeoJSON 格式输入
    """

    def __init__(self, include_boundary=True):
        """
        Args:
            include_boundary: 点在边界上是否视为在配送区内
                              True = 包含（推荐，对用户友好）
                              False = 不包含（严格模式）
        """
        self.include_boundary = include_boundary

    def contains(self, point: Tuple[float, float],
                 polygon: List[Tuple[float, float]]) -> bool:
        """判定点是否在多边形内

        Args:
            point: (lng, lat) 待判定点坐标
            polygon: [(lng, lat), ...] 多边形顶点列表（逆时针或顺时针均可）

        Returns:
            True = 在多边形内或边界上（include_boundary=True时）
        """
        n = len(polygon)
        if n < 3:
            return False

        # 先做包围盒（Bounding Box）快速排除
        if not self._in_bounding_box(point, polygon):
            return False

        inside = False
        px, py = point

        j = n - 1  # 前一个顶点
        for i in range(n):
            xi, yi = polygon[i]
            xj, yj = polygon[j]

            # 检查点是否在当前边上
            if self._on_segment(point, (xi, yi), (xj, yj)):
                return self.include_boundary

            # 射线法核心逻辑
            # 条件1: 点的 Y 坐标在边的 Y 范围内（严格不等）
            # 条件2: 射线与边的交点在点的右侧
            if ((yi > py) != (yj > py)) and \
               (px < (xj - xi) * (py - yi) / (yj - yi) + xi):
                inside = not inside

            j = i

        return inside

    def contains_with_holes(self, point: Tuple[float, float],
                            outer: List[Tuple[float, float]],
                            holes: List[List[Tuple[float, float]]]) -> bool:
        """判定点是否在带"洞"的多边形内

        典型场景：配送范围覆盖某区域，但排除中间的湖泊/军事区。

        Args:
            point: (lng, lat) 待判定点
            outer: 外部多边形顶点
            holes: 内部洞的多边形列表（每个洞也是一个多边形）

        Returns:
            在外部多边形内且不在任何洞内 → True
        """
        # 必须在外部多边形内
        if not self.contains(point, outer):
            return False

        # 不能在任何洞内
        for hole in holes:
            if self.contains(point, hole):
                return False

        return True

    def _in_bounding_box(self, point, polygon):
        """包围盒快速排除：点不在包围盒内 → 一定不在多边形内"""
        px, py = point
        min_x = min(p[0] for p in polygon)
        max_x = max(p[0] for p in polygon)
        min_y = min(p[1] for p in polygon)
        max_y = max(p[1] for p in polygon)
        return min_x <= px <= max_x and min_y <= py <= max_y

    def _on_segment(self, point, seg_start, seg_end):
        """判断点是否在线段上（考虑浮点精度）

        使用向量叉积判断共线 + 点积判断在线段范围内
        """
        px, py = point
        x1, y1 = seg_start
        x2, y2 = seg_end

        # 向量叉积 = 0 表示共线
        cross = (px - x1) * (y2 - y1) - (py - y1) * (x2 - x1)
        if abs(cross) > 1e-10:  # 浮点容差
            return False

        # 共线后，判断是否在线段范围内
        dot = (px - x1) * (x2 - x1) + (py - y1) * (y2 - y1)
        squared_length = (x2 - x1) ** 2 + (y2 - y1) ** 2

        if squared_length < 1e-12:  # 退化线段
            return abs(px - x1) < 1e-10 and abs(py - y1) < 1e-10

        t = dot / squared_length
        return -1e-10 <= t <= 1.0 + 1e-10

    @staticmethod
    def from_geojson(geojson: dict) -> 'PointInPolygon.Checker':
        """从 GeoJSON 创建判定器

        支持 Polygon 和 MultiPolygon 两种类型。
        MultiPolygon 用于商家有多个不连续配送区域（如两岸都有配送）。
        """
        geometry = geojson.get('geometry', geojson)
        geo_type = geometry['type']

        if geo_type == 'Polygon':
            return PointInPolygon.Checker(
                polygons=[geometry['coordinates'][0]],
                holes=[geometry['coordinates'][1:]] if len(geometry['coordinates']) > 1 else []
            )
        elif geo_type == 'MultiPolygon':
            all_polygons = []
            all_holes = []
            for polygon_coords in geometry['coordinates']:
                all_polygons.append(polygon_coords[0])
                if len(polygon_coords) > 1:
                    all_holes.extend(polygon_coords[1:])
            return PointInPolygon.Checker(polygons=all_polygons, holes=all_holes)
        else:
            raise ValueError(f"Unsupported GeoJSON type: {geo_type}")

    class Checker:
        """预编译的多边形判定器（从 GeoJSON 创建，支持多配送区）"""

        def __init__(self, polygons, holes):
            self.pip = PointInPolygon(include_boundary=True)
            self.polygons = polygons  # 可能多个不连续区域
            self.holes = holes        # 洞（排除区域）

        def contains(self, point):
            """判定点是否在任一配送区域内（排除洞）"""
            in_any = False
            for polygon in self.polygons:
                if self.pip.contains(point, polygon):
                    in_any = True
                    break

            if not in_any:
                return False

            # 检查是否在洞内
            for hole in self.holes:
                if self.pip.contains(point, hole):
                    return False

            return True
```

**配送范围服务（整合 ES + 本地算法）：**

```python
class DeliveryAreaService:
    """配送范围判定服务

    两级判定策略：
    1. 快速路径：Redis 缓存 + 本地算法（P99 < 5ms）
    2. 兜底路径：Elasticsearch geo_shape 查询（P99 < 50ms）
    """

    def __init__(self, redis, es_client):
        self.redis = redis
        self.es = es_client
        self.pip = PointInPolygon(include_boundary=True)
        # LRU 缓存：商家ID → 多边形数据（减少 Redis 查询）
        self.polygon_cache = LRUCache(maxsize=100000)

    def check_delivery_area(self, merchant_id, user_lng, user_lat):
        """判定用户是否在商家配送范围内

        Args:
            merchant_id: 商家ID
            user_lng: 用户经度
            user_lat: 用户纬度

        Returns:
            {
                "deliverable": True/False,
                "area_type": "main"/"extended"/None,
                "estimated_minutes": 30,
                "delivery_fee": 5.0,
                "source": "cache"/"es"/"local"
            }
        """
        # 第一级：本地缓存判定
        cached = self.polygon_cache.get(merchant_id)
        if cached is not None:
            result = self._check_local(cached, user_lng, user_lat)
            if result is not None:
                result["source"] = "local"
                return result

        # 第二级：Redis 缓存判定
        polygon_data = self.redis.get(f"delivery_area:{merchant_id}")
        if polygon_data:
            areas = json.loads(polygon_data)
            self.polygon_cache[merchant_id] = areas
            result = self._check_local(areas, user_lng, user_lat)
            if result is not None:
                result["source"] = "cache"
                return result

        # 第三级：ES 查询兜底
        return self._check_es(merchant_id, user_lng, user_lat)

    def _check_local(self, areas, user_lng, user_lat):
        """本地算法判定（最快，P99 < 2ms）"""
        point = (user_lng, user_lat)

        for area in areas:
            polygon = [(p[0], p[1]) for p in area['coordinates']]
            if self.pip.contains(point, polygon):
                holes = [
                    [(h[0], h[1]) for h in hole]
                    for hole in area.get('holes', [])
                ]
                if holes and any(self.pip.contains(point, h) for h in holes):
                    continue  # 在洞内，跳过此区域

                return {
                    "deliverable": True,
                    "area_type": area.get('area_type', 'main'),
                    "estimated_minutes": area.get('avg_delivery_min', 30),
                    "delivery_fee": area.get('delivery_fee', 5.0)
                }

        return {"deliverable": False, "area_type": None}

    def _check_es(self, merchant_id, user_lng, user_lat):
        """ES geo_shape 查询兜底（P99 < 50ms）"""
        result = self.es.search(index="merchants", body={
            "query": {
                "bool": {
                    "filter": [
                        {"term": {"_id": merchant_id}},
                        {
                            "geo_shape": {
                                "delivery_area": {
                                    "shape": {
                                        "type": "point",
                                        "coordinates": [user_lng, user_lat]
                                    },
                                    "relation": "contains"
                                }
                            }
                        }
                    ]
                }
            },
            "_source": ["delivery_area", "avg_delivery_minutes", "delivery_fee"]
        })

        if result['hits']['total']['value'] > 0:
            return {
                "deliverable": True,
                "area_type": "main",
                "source": "es"
            }

        return {"deliverable": False, "area_type": None, "source": "es"}

    def batch_check(self, merchant_ids, user_lng, user_lat):
        """批量判定：用户同时查看多个商家是否可配送

        优化：本地缓存命中后并行判定，避免逐个查 Redis/ES
        """
        results = {}
        cache_miss = []

        for mid in merchant_ids:
            cached = self.polygon_cache.get(mid)
            if cached:
                result = self._check_local(cached, user_lng, user_lat)
                if result:
                    result["source"] = "local"
                    results[mid] = result
                    continue
            cache_miss.append(mid)

        if cache_miss:
            # 批量从 Redis 获取
            pipe = self.redis.pipeline()
            for mid in cache_miss:
                pipe.get(f"delivery_area:{mid}")
            polygon_list = pipe.execute()

            for mid, polygon_data in zip(cache_miss, polygon_list):
                if polygon_data:
                    areas = json.loads(polygon_data)
                    self.polygon_cache[mid] = areas
                    result = self._check_local(areas, user_lng, user_lat)
                    if result:
                        result["source"] = "cache"
                        results[mid] = result
                    else:
                        results[mid] = {"deliverable": False, "source": "cache"}
                else:
                    results[mid] = {"deliverable": False, "source": "miss"}

        return results
```

**GeoJSON 配送范围数据示例（含洞和 MultiPolygon）：**

```json
{
  "type": "Feature",
  "properties": {
    "merchant_id": "M-001",
    "merchant_name": "朝阳大悦城店"
  },
  "geometry": {
    "type": "Polygon",
    "coordinates": [
      [
        [116.397, 39.916], [116.405, 39.918], [116.410, 39.914],
        [116.408, 39.908], [116.400, 39.905], [116.393, 39.910],
        [116.397, 39.916]
      ],
      [
        [116.400, 39.912], [116.403, 39.912], [116.403, 39.910],
        [116.400, 39.910], [116.400, 39.912]
      ]
    ]
  }
}
```

```json
{
  "type": "Feature",
  "properties": {
    "merchant_id": "M-002",
    "merchant_name": "黄浦江两岸配送店"
  },
  "geometry": {
    "type": "MultiPolygon",
    "coordinates": [
      [[[121.47, 31.23], [121.49, 31.24], [121.50, 31.22], [121.47, 31.23]]],
      [[[121.52, 31.24], [121.54, 31.25], [121.55, 31.23], [121.52, 31.24]]]
    ]
  }
}
```

**凹多边形与凸多边形的边界情况对比：**

```
凸多边形（餐厅A）：任意内角 < 180°
  ┌─────┐
  │     │
  │  ★  │  ← 用户点在内部，射线交2次（偶数）→ 内部
  │     │
  └─────┘

凹多边形（餐厅B，配送区避开了铁路）：存在内角 > 180°
  ┌─────┐
  │     │
  │  ★──┼──┐  ← 射线穿过凹角区域，交3次（奇数）→ 内部
  │     │  │
  └─────┘  │
     └─────┘

边界上的点：
  ┌──★──┐    ← 点正好在边上
  │     │       include_boundary=True → 在配送区内（推荐）
  │     │       include_boundary=False → 不在配送区内
  └─────┘

凹多边形内角处的点：
  ┌─────┐
  │  ★  │     ← 凹角处射线可能交2次或4次
  │╲    │        射线法仍然正确（奇数次=内部）
  │ ╲   │
  └──╲──┘
```

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

### 热力图：GeoHash 聚合 + Flink + 瓦片渲染

**热力图整体架构：**

```
骑手/订单位置数据 → Kafka → Flink 实时聚合
                                  │
                                  ├→ Redis（实时热力图查询，P99 < 10ms）
                                  │    key: heatmap:{city_id}:{geohash_prefix}
                                  │    value: 计数（订单密度/骑手密度）
                                  │
                                  ├→ Redis（瓦片缓存，前端直接取）
                                  │    key: heatmap:tile:{z}:{x}:{y}
                                  │    value: PNG 二进制
                                  │
                                  └→ MySQL（历史热力图，按小时归档）
                                       用于运营分析（高峰时段、区域对比）
```

**热力图精度与缩放级别对应关系：**

| 地图缩放级别 z | GeoHash 精度 | 格子边长 | 用途 |
|:---:|:---:|:---:|:---|
| 4-6 | 3位 | 156km×156km | 全国概览 |
| 7-9 | 4位 | 20km×20km | 城市级概览 |
| 10-12 | 5位 | 2.4km×2.4km | 区级密度 |
| 13-15 | 6位 | 610m×610m | 街道级热力 |
| 16-18 | 7位 | 76m×76m | 楼栋级精度 |

**Flink 热力图聚合作业（完整 Java 实现）：**

```java
/**
 * Flink Job: 实时热力图聚合
 * 输入: Kafka rider_location / order_created 事件
 * 输出: Redis 热力图数据 + 瓦片缓存
 */
public class HeatmapAggregationJob {

    // 地图缩放级别 → GeoHash 精度映射
    private static final Map<Integer, Integer> ZOOM_TO_PRECISION = Map.of(
        4, 3, 5, 3, 6, 3,   // 全国概览
        7, 4, 8, 4, 9, 4,   // 城市级
        10, 5, 11, 5, 12, 5, // 区级
        13, 6, 14, 6, 15, 6, // 街道级
        16, 7, 17, 7, 18, 7  // 楼栋级
    );

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(30);
        env.enableCheckpointing(60000);  // 1分钟 Checkpoint

        // 消费骑手位置和订单创建两个 Topic
        KafkaSource<String> riderSource = KafkaSource.<String>builder()
            .setBootstrapServers("kafka1:9092,kafka2:9092,kafka3:9092")
            .setTopics("rider_location")
            .setGroupId("heatmap-rider-group")
            .setStartingOffsets(OffsetsInitializer.latest())
            .setValueOnlyDeserializer(new SimpleStringSchema())
            .build();

        KafkaSource<String> orderSource = KafkaSource.<String>builder()
            .setBootstrapServers("kafka1:9092,kafka2:9092,kafka3:9092")
            .setTopics("order_created")
            .setGroupId("heatmap-order-group")
            .setStartingOffsets(OffsetsInitializer.latest())
            .setValueOnlyDeserializer(new SimpleStringSchema())
            .build();

        DataStream<HeatmapEvent> riderEvents = env
            .fromSource(riderSource, WatermarkStrategy.noWatermarks(), "RiderSource")
            .map(new RiderLocationMapper());

        DataStream<HeatmapEvent> orderEvents = env
            .fromSource(orderSource, WatermarkStrategy.noWatermarks(), "OrderSource")
            .map(new OrderCreatedMapper());

        // 合并两个流
        DataStream<HeatmapEvent> allEvents = riderEvents.union(orderEvents);

        // 多级精度聚合（同时维护4个精度级别）
        for (Map.Entry<Integer, Integer> entry : List.of(
            Map.entry(3, 4), Map.entry(4, 7), Map.entry(5, 10), Map.entry(6, 13)
        )) {
            int precision = entry.getKey();
            int windowSec = entry.getValue();

            allEvents
                .map(event -> toGeoHashBucket(event, precision))
                .keyBy(bucket -> bucket.key)
                .window(TumblingProcessingTimeWindows.of(Time.seconds(windowSec)))
                .aggregate(new CountAggregator())
                .addSink(new RedisHeatmapSink(precision));
        }

        env.execute("HeatmapAggregation");
    }

    /** 将事件映射到 GeoHash 桶 */
    static GeoHashBucket toGeoHashBucket(HeatmapEvent event, int precision) {
        String hash = GeoHash.geoHashStringWithCharacterPrecision(
            event.lat, event.lng, precision
        );
        return new GeoHashBucket(
            String.format("heatmap:%d:%s", event.cityId, hash),
            hash, event.cityId, event.eventType
        );
    }

    /** 计数聚合器 */
    static class CountAggregator implements AggregateFunction<GeoHashBucket, Long, Long> {
        @Override public Long createAccumulator() { return 0L; }
        @Override public Long add(GeoHashBucket bucket, Long acc) { return acc + 1; }
        @Override public Long getResult(Long acc) { return acc; }
        @Override public Long merge(Long a, Long b) { return a + b; }
    }

    /** Redis 热力图 Sink */
    static class RedisHeatmapSink extends RichSinkFunction<Tuple2<GeoHashBucket, Long>> {
        private final int precision;
        private transient JedisPool jedisPool;

        RedisHeatmapSink(int precision) { this.precision = precision; }

        @Override
        public void open(Configuration parameters) {
            jedisPool = new JedisPool(new JedisPoolConfig(),
                "redis-cluster-endpoint", 6379, 2000);
        }

        @Override
        public void invoke(Tuple2<GeoHashBucket, Long> value, Context context) {
            try (Jedis jedis = jedisPool.getResource()) {
                String key = value.f0.key;
                long count = value.f1;

                // 更新计数值（用 SET 而非 INCR，因为 Flink 窗口已聚合）
                jedis.setex(key, 3600, String.valueOf(count));  // 1小时过期

                // 同时更新瓦片缓存（预渲染，减少前端计算）
                updateTileCache(jedis, value.f0.cityId, value.f0.geohash, count);
            }
        }

        /** 瓦片缓存更新：将 GeoHash 聚合结果映射到地图瓦片坐标 */
        private void updateTileCache(Jedis jedis, int cityId, String geohash, long count) {
            GeoPoint center = GeoHash.decodeGeohashAsLongs(geohash);
            double lat = center.getLat() / 1e6;
            double lng = center.getLon() / 1e6;

            // 对每个缩放级别计算瓦片坐标
            for (int z = 10; z <= 15; z++) {
                int[] tileXY = latLngToTile(lat, lng, z);
                String tileKey = String.format("heatmap:tile:%d:%d:%d", z, tileXY[0], tileXY[1]);
                // 将计数追加到瓦片的 Hash 中（瓦片内可能有多个 GeoHash 格子）
                jedis.hset(tileKey, geohash, String.valueOf(count));
                jedis.expire(tileKey, 1800);  // 30分钟过期
            }
        }

        /** 经纬度 → 瓦片坐标（Web Mercator 投影） */
        static int[] latLngToTile(double lat, double lng, int zoom) {
            int n = (1 << zoom);
            int x = (int) ((lng + 180.0) / 360.0 * n);
            double latRad = Math.toRadians(lat);
            int y = (int) ((1.0 - Math.log(Math.tan(latRad) + 1/Math.cos(latRad)) / Math.PI) / 2.0 * n);
            return new int[]{x, y};
        }
    }

    static class GeoHashBucket {
        public String key;       // Redis key
        public String geohash;   // GeoHash 字符串
        public int cityId;
        public String eventType;
    }

    static class HeatmapEvent {
        public double lat, lng;
        public int cityId;
        public String eventType;  // "rider" or "order"
    }
}
```

**热力图查询服务（Python 实现）：**

```python
class HeatmapService:
    """实时热力图服务 - 完整实现"""

    def __init__(self, redis_client):
        self.redis = redis_client
        self.precision_map = {
            (4, 6): 3,   # 全国概览
            (7, 9): 4,   # 城市级
            (10, 12): 5,  # 区级
            (13, 15): 6,  # 街道级
            (16, 18): 7,  # 楼栋级
        }

    def _get_precision(self, zoom_level):
        """根据缩放级别确定 GeoHash 精度"""
        for (lo, hi), precision in self.precision_map.items():
            if lo <= zoom_level <= hi:
                return precision
        return 5  # 默认

    def get_heatmap(self, city_id, zoom_level=12, bounds=None):
        """获取城市热力图数据

        Args:
            city_id: 城市ID
            zoom_level: 地图缩放级别（4-18）
            bounds: 可选，地图可视区域 (south, west, north, east)，减少返回数据量

        Returns:
            热力图数据列表 [{"lat": ..., "lng": ..., "count": ...}, ...]
        """
        precision = self._get_precision(zoom_level)

        if bounds:
            # 只返回可视区域内的热力数据，大幅减少数据量
            return self._get_heatmap_in_bounds(city_id, precision, bounds)

        # 全城市热力数据
        pattern = f"heatmap:{city_id}:*"
        keys = self.redis.keys(pattern)

        # 使用 Pipeline 批量读取，避免逐个 GET
        pipe = self.redis.pipeline()
        for key in keys:
            pipe.get(key)
        values = pipe.execute()

        heatmap = []
        for key, count in zip(keys, values):
            if count is None:
                continue
            geohash_str = key.decode().split(":")[-1]
            lat, lng = self._decode_geohash_center(geohash_str)
            heatmap.append({
                "geohash": geohash_str,
                "lat": lat,
                "lng": lng,
                "count": int(count),
                "intensity": self._calc_intensity(int(count), precision)
            })

        return heatmap

    def _get_heatmap_in_bounds(self, city_id, precision, bounds):
        """只获取可视区域内的热力数据

        计算可视区域四个角对应的 GeoHash 前缀，
        然后只查询这些前缀覆盖的格子。
        """
        south, west, north, east = bounds

        # 生成覆盖可视区域的所有 GeoHash 前缀
        covered_hashes = set()
        # 沿边界采样点，计算每个点所属的 GeoHash
        sample_points = []
        for lat in [south, north]:
            for lng in [west, east]:
                sample_points.append((lat, lng))
        # 在中间也采样，避免遗漏
        sample_points.append(((south + north) / 2, (west + east) / 2))

        for lat, lng in sample_points:
            h = geohash.encode(lat, lng, precision=precision)
            covered_hashes.add(h)
            # 加上相邻格子（边界安全）
            for neighbor in geohash.neighbors(h):
                covered_hashes.add(neighbor)

        # 批量查询
        pipe = self.redis.pipeline()
        keys_map = {}
        for h in covered_hashes:
            key = f"heatmap:{city_id}:{h}"
            pipe.get(key)
            keys_map[key] = h

        values = pipe.execute()

        heatmap = []
        for (key, h), count in zip(keys_map.items(), values):
            if count is None:
                continue
            lat, lng = self._decode_geohash_center(h)
            heatmap.append({
                "geohash": h,
                "lat": lat,
                "lng": lng,
                "count": int(count),
                "intensity": self._calc_intensity(int(count), precision)
            })

        return heatmap

    def get_tile(self, z, x, y):
        """获取预渲染瓦片（前端直接使用，避免实时计算）

        前端请求: /api/heatmap/tile/{z}/{x}/{y}.png
        返回: 瓦片内的热力点数据
        """
        tile_key = f"heatmap:tile:{z}:{x}:{y}"
        tile_data = self.redis.hgetall(tile_key)

        if not tile_data:
            return None

        result = []
        for geohash_str, count in tile_data.items():
            lat, lng = self._decode_geohash_center(geohash_str.decode())
            result.append({
                "geohash": geohash_str.decode(),
                "lat": lat, "lng": lng,
                "count": int(count)
            })

        return result

    def on_order_created(self, order):
        """订单创建 → 实时更新热力图"""
        precision = 5  # 默认区级精度
        geohash_str = geohash.encode(order.lat, order.lng, precision=precision)

        # 多级精度同时更新
        for prec in [3, 4, 5, 6]:
            h = geohash.encode(order.lat, order.lng, precision=prec)
            key = f"heatmap:{order.city_id}:{h}"
            self.redis.incr(key)
            self.redis.expire(key, 3600)  # 1小时过期

    def _calc_intensity(self, count, precision):
        """计算热力强度（0-1），用于前端颜色渲染

        不同精度下同一区域计数差异大，需要归一化。
        精度3位：一个格子可能包含数千订单
        精度6位：一个格子可能只有个位数订单
        """
        thresholds = {3: 5000, 4: 500, 5: 50, 6: 10, 7: 3}
        threshold = thresholds.get(precision, 50)
        return min(1.0, count / threshold)

    def _decode_geohash_center(self, geohash_str):
        """解码 GeoHash 中心点坐标"""
        lat, lng = geohash.decode(geohash_str)
        return lat, lng
```

**热力图 API 端点（Flask 实现）：**

```python
from flask import Flask, request, jsonify

app = Flask(__name__)
heatmap_service = HeatmapService(redis_client)

@app.route('/api/heatmap', methods=['GET'])
def get_heatmap():
    """热力图数据接口

    GET /api/heatmap?city_id=1&zoom=12&south=39.8&west=116.3&north=39.95&east=116.45
    """
    city_id = int(request.args.get('city_id'))
    zoom = int(request.args.get('zoom', 12))

    bounds = None
    if all(k in request.args for k in ['south', 'west', 'north', 'east']):
        bounds = (
            float(request.args['south']),
            float(request.args['west']),
            float(request.args['north']),
            float(request.args['east'])
        )

    data = heatmap_service.get_heatmap(city_id, zoom, bounds)
    return jsonify({"heatmap": data, "count": len(data)})

@app.route('/api/heatmap/tile/<int:z>/<int:x>/<int:y>', methods=['GET'])
def get_tile(z, x, y):
    """瓦片数据接口（前端地图组件使用）"""
    data = heatmap_service.get_tile(z, x, y)
    if data is None:
        return jsonify({"heatmap": [], "count": 0})
    return jsonify({"heatmap": data, "count": len(data)})
```

### 写入性能分析

| 数据类型 | 写入 QPS | 引擎 | 节点数 | 存储 |
|---------|---------|------|-------|------|
| 商家数据 | ~1/秒 | ES 批量写入 | 6 节点 | 50GB |
| 骑手位置 | 17 万/秒 | Redis GEO | 4 节点 | 2GB |
| 配送范围 | ~1/秒 | ES geo_shape | 6 节点 | 含在商家索引中 |
| 骑手轨迹 | 17 万/秒 | Kafka → HDFS | 6 broker | 5TB/天 |

### 详细性能分析

#### Redis GEO 内存分析：50 万骑手

```
Redis GEO 底层实现 = Sorted Set (ZSET)
每个成员存储: member(骑手ID) + score(GeoHash编码的52位整数)

单个骑手 GEO 条目内存:
  member (rider_id):    ~20 bytes  (字符串 "R-1234567890")
  score:                8 bytes    (double)
  ZSET 节点开销:        ~32 bytes  (指针、层级等)
  ─────────────────────────────
  单条总计:             ~60 bytes

50万骑手:
  GEO 数据:  500,000 × 60 bytes ≈ 30 MB / 城市
  状态 Hash: 500,000 × 200 bytes ≈ 100 MB (含 TTL 和所有字段)

400 城市（假设平均每城市 1,250 骑手，高峰期集中约 50 个大城市）:
  大城市(50个): 50 × 30MB = 1.5 GB (GEO) + 50 × 100MB = 5 GB (Hash)
  小城市(350个): 350 × 0.75MB = 263 MB (GEO)
  ─────────────────────────────
  总内存: ~7 GB

4 节点 Redis Cluster，每节点: ~2 GB（含副本则 ~4 GB/节点）
推荐配置: 4 主 4 从，每节点 8GB 内存 → 充足
```

**GEORADIUS 查询延迟在不同规模下的表现：**

| 城市骑手数 | 3km 半径结果数 | P50 延迟 | P99 延迟 | CPU 占用 |
|:---:|:---:|:---:|:---:|:---:|
| 1,000 | ~30 | 0.5ms | 2ms | < 1% |
| 5,000 | ~150 | 1ms | 4ms | < 2% |
| 20,000 | ~600 | 3ms | 8ms | ~5% |
| 50,000 | ~1,500 | 5ms | 15ms | ~10% |
| 100,000 | ~3,000 | 12ms | 35ms | ~20% |

> 注意：超过 5 万骑手的超大城市（如北京），应进一步按区域（区/商圈）分片，避免单 key 过大。

#### Elasticsearch 集群容量分析：500 万商家

```
单文档大小估算:
  merchant_no:  20B
  name:         ~50B (含分词倒排索引)
  location:     16B (geo_point)
  delivery_area: ~500B (geo_shape, 平均20顶点多边形)
  其他字段:     ~100B
  倒排索引开销: ~200B (各字段的倒排)
  doc_values:   ~80B
  ─────────────────────────────
  单文档总计:   ~1 KB

500万商家:
  原始数据: 500万 × 1KB = 5 GB
  副本(1份): +5 GB
  段合并开销(50%): +5 GB
  操作系统页缓存: +5 GB
  ─────────────────────────────
  总存储需求: ~20 GB

20 分片 × 6 节点:
  每节点: ~3-4 分片
  每节点存储: ~4 GB
  推荐: 6 节点 × 32GB 内存 (16GB ES堆 + 16GB OS页缓存)
```

**ES 查询延迟在不同 QPS 下的表现：**

| 查询 QPS | P50 延迟 | P99 延迟 | CPU 平均 | CPU 峰值 |
|:---:|:---:|:---:|:---:|:---:|
| 10,000 | 15ms | 50ms | 30% | 50% |
| 30,000 | 20ms | 80ms | 50% | 70% |
| 50,000 | 30ms | 150ms | 65% | 85% |
| 80,000 | 80ms | 500ms | 80% | 95% |
| 100,000 | 200ms | 2000ms | 95% | 100% (打满) |

> 优化后（geo_bounding_box 粗筛 + index sorting 早终止），80K QPS 时 P99 可从 500ms 降到 120ms。

#### Kafka 吞吐量分析：17 万次/秒位置更新

```
单条消息大小:
  rider_id:    8B
  lat/lng:     16B
  accuracy:    2B
  speed:       2B
  source:      4B
  timestamp:   8B
  JSON 开销:   ~40B
  ─────────────────────────────
  单条总计:    ~80 bytes

17万条/秒:
  原始吞吐: 170,000 × 80B = 13.6 MB/s
  LZ4 压缩后(4:1): ~3.4 MB/s
  ─────────────────────────────
  每天数据量: 170,000 × 86400 × 80B ≈ 1.1 TB (未压缩)
  压缩后:     ~280 GB/天

Kafka 集群配置:
  6 Broker, 60 分区, 3 副本
  每分区吞吐: 170,000 / 60 ≈ 2,800 条/秒 = 0.22 MB/s
  Kafka 单 Broker 可支撑 100+ MB/s → 远未达瓶颈
```

**Kafka 消费延迟在不同消费者数量下的表现：**

| 消费者组 | 并行度 | 处理延迟 | 消费 Lag | 资源消耗 |
|:---:|:---:|:---:|:---:|:---:|
| Redis GEO 同步 | 60 | < 1s | < 5,000 条 | 4 CPU + 8GB |
| ES 索引同步 | 20 | < 5s | < 50,000 条 | 4 CPU + 8GB |
| 热力图聚合 | 30 | < 10s | < 100,000 条 | 8 CPU + 16GB |
| 轨迹归档(HDFS) | 20 | < 30s | < 300,000 条 | 4 CPU + 8GB |

#### 配送范围判定性能分析

| 判定方式 | 单次延迟 | 批量(100商家) | 适用场景 |
|:---:|:---:|:---:|:---:|
| 本地算法(内存缓存) | 0.01ms | 1ms | 缓存命中时 |
| Redis + 本地算法 | 2ms | 20ms | 首次查询 |
| ES geo_shape 查询 | 30ms | 300ms | 兜底路径 |
| MySQL ST_Contains | 500ms | 50s | 不推荐 |

**端到端查询延迟（"附近商家"搜索完整链路）：**

```
用户点击搜索 → API 网关 → 搜索服务
  │
  ├→ ES 查询（geo_bounding_box + 业务过滤 + 排序）: 30-80ms
  │
  ├→ 配送范围批量判定（本地缓存命中）: 1-5ms
  │
  ├→ 骑手位置查询（Redis GEORADIUS）: 2-5ms
  │
  ├→ 结果聚合 + 排序: 1-3ms
  │
  └→ 返回用户: 总计 35-95ms（满足 < 100ms P99 要求）
```

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
## 地理围栏引擎完整实现

```python
class GeofenceEngine:
    """地理围栏引擎：判断点是否在围栏内"""

    def check_geofences(self, point, tenant_id):
        """检查点是否在任何围栏内"""
        fences = self.get_applicable_fences(point, tenant_id)
        triggered = []
        for fence in fences:
            if self._is_inside(point, fence):
                triggered.append(fence)
        return triggered

    def _is_inside(self, point, fence):
        """判断点是否在围栏内"""
        if fence["type"] == "circle":
            return self._point_in_circle(point, fence["center"], fence["radius_meters"])
        elif fence["type"] == "polygon":
            return self._point_in_polygon(point, fence["vertices"])
        return False

    def _point_in_circle(self, point, center, radius):
        """判断点是否在圆形围栏内"""
        distance = self.haversine(point.lat, point.lng, center["lat"], center["lng"])
        return distance <= radius

    def _point_in_polygon(self, point, vertices):
        """射线法判断点是否在多边形内"""
        n = len(vertices)
        inside = False
        j = n - 1
        for i in range(n):
            vi, vj = vertices[i], vertices[j]
            if ((vi["lat"] > point.lat) != (vj["lat"] > point.lat) and
                point.lng < (vj["lng"] - vi["lng"]) * (point.lat - vi["lat"]) /
                (vj["lat"] - vi["lat"]) + vi["lng"]):
                inside = not inside
            j = i
        return inside

    def haversine(self, lat1, lon1, lat2, lon2):
        """Haversine 公式计算两点间距离（米）"""
        R = 6371000
        dlat = math.radians(lat2 - lat1)
        dlon = math.radians(lon2 - lon1)
        a = (math.sin(dlat / 2) ** 2 +
             math.cos(math.radians(lat1)) * math.cos(math.radians(lat2)) *
             math.sin(dlon / 2) ** 2)
        return R * 2 * math.atan2(math.sqrt(a), math.sqrt(1 - a))
```

## 轨迹分析完整实现

```python
class TrajectoryAnalyzer:
    """轨迹分析：停留点检测、路线匹配"""

    def detect_stay_points(self, points, time_threshold_minutes=30, distance_threshold_meters=50):
        """检测停留点：在某区域停留超过阈值时间"""
        stay_points = []
        i = 0
        while i < len(points):
            j = i + 1
            while j < len(points):
                dist = self.haversine(points[i].lat, points[i].lng,
                                      points[j].lat, points[j].lng)
                if dist > distance_threshold_meters:
                    break
                j += 1

            # 检查停留时间
            if j > i + 1:
                duration = (points[j-1].timestamp - points[i].timestamp).total_seconds() / 60
                if duration >= time_threshold_minutes:
                    stay_points.append({
                        "center_lat": statistics.mean(p.lat for p in points[i:j]),
                        "center_lng": statistics.mean(p.lng for p in points[i:j]),
                        "arrival_time": points[i].timestamp,
                        "departure_time": points[j-1].timestamp,
                        "duration_minutes": duration,
                        "point_count": j - i
                    })
            i = j
        return stay_points
```

## 异常场景补充

### 场景：GPS 信号漂移

```
触发：城市高楼区 GPS 信号反射 → 定位点跳跃 200-500 米
      → 用户位置突然从 A 点跳到 B 点（不可能的速度）
检测：
  1. 速度检测：两点间距离 / 时间差 > 200km/h → 漂移
  2. 精度检测：GPS accuracy > 100 米 → 可信度低
处理：
  1. 漂移点过滤：丢弃不可能的跳跃点
  2. 卡尔曼滤波：平滑轨迹，消除噪声
  3. 可信度降级：标记该位置为 "低精度"
预防：位置上报时携带 accuracy 字段 + 速度合理性校验
```

### 场景：围栏误触发

```
触发：用户在围栏边界附近 → GPS 精度 50 米 → 反复进出围栏
      → 产生大量无意义的进出事件
检测：
  1. 同一围栏 5 分钟内进出 > 3 次 → 边界抖动
处理：
  1. 防抖：进出围栏后 30 秒内不再触发
  2. 滞后阈值：离开围栏需超过 100 米才触发 exit 事件
  3. 合并事件：5 分钟内同一围栏只保留最终状态
预防：围栏边界加缓冲区（50-100 米灰色地带）
```

## 附近商家搜索完整实现（Redis GEO 命令）

```python
class ProximitySearchService:
    """基于 Redis GEO 的附近商家搜索服务

    利用 Redis GEOADD 存储商家位置，GEORADIUS 查找范围内商家，
    结合业务过滤（分类、评分、营业状态）和分页排序。
    """

    # Redis Key 前缀
    GEO_KEY_PREFIX = "merchants:geo"
    STATUS_KEY_PREFIX = "merchants:status"
    DETAIL_KEY_PREFIX = "merchants:detail"

    def __init__(self, redis_client, es_client):
        self.redis = redis_client
        self.es = es_client

    # ========== 商家位置存储（GEOADD）==========

    def add_merchant_location(self, merchant_id, lat, lng, city_id):
        """存储商家地理位置到 Redis GEO

        使用 GEOADD 将商家经纬度写入 Sorted Set，
        底层用 GeoHash 编码为 52 位整数作为 score。

        Args:
            merchant_id: 商家ID
            lat: 纬度
            lng: 经度
            city_id: 城市ID（用于按城市分片）
        """
        geo_key = f"{self.GEO_KEY_PREFIX}:{city_id}"
        # GEOADD key longitude latitude member
        self.redis.geoadd(geo_key, lng, lat, str(merchant_id))

        # 同时存储商家简要状态到 Hash（用于快速过滤）
        status_key = f"{self.STATUS_KEY_PREFIX}:{merchant_id}"
        self.redis.hset(status_key, mapping={
            "lat": str(lat),
            "lng": str(lng),
            "city_id": str(city_id),
        })
        self.redis.expire(status_key, 86400)  # 24小时过期

    def batch_add_merchant_locations(self, merchants, city_id):
        """批量存储商家位置（Pipeline 优化）

        Args:
            merchants: [(merchant_id, lat, lng), ...]
            city_id: 城市ID
        """
        geo_key = f"{self.GEO_KEY_PREFIX}:{city_id}"
        pipe = self.redis.pipeline()

        for merchant_id, lat, lng in merchants:
            pipe.geoadd(geo_key, lng, lat, str(merchant_id))
            status_key = f"{self.STATUS_KEY_PREFIX}:{merchant_id}"
            pipe.hset(status_key, mapping={
                "lat": str(lat),
                "lng": str(lng),
                "city_id": str(city_id),
            })
            pipe.expire(status_key, 86400)

        pipe.execute()

    # ========== 附近商家搜索（GEORADIUS + 过滤）==========

    def search_nearby(self, user_lat, user_lng, city_id,
                      radius_m=3000, category=None,
                      min_rating=0.0, only_open=True,
                      sort_by="distance", page=1, page_size=20):
        """查找附近商家，支持多条件过滤和分页

        流程：
        1. GEORADIUS 获取范围内所有商家ID+距离
        2. 根据过滤条件筛选（分类、评分、营业状态）
        3. 按指定方式排序（距离/评分）
        4. 分页返回

        Args:
            user_lat: 用户纬度
            user_lng: 用户经度
            city_id: 城市ID
            radius_m: 搜索半径（米），默认 3000 米
            category: 分类过滤（如 chinese_food）
            min_rating: 最低评分过滤
            only_open: 是否只返回营业中的商家
            sort_by: 排序方式 - distance / rating
            page: 页码（从1开始）
            page_size: 每页数量

        Returns:
            {
                "total": 匹配总数,
                "page": 当前页码,
                "merchants": [
                    {"merchant_id": ..., "distance": ..., "rating": ..., ...},
                    ...
                ]
            }
        """
        geo_key = f"{self.GEO_KEY_PREFIX}:{city_id}"

        # 1. GEORADIUS 获取范围内所有商家（多取，因为需要过滤）
        # 实际需要数量 = page_size * 5（预估过滤后剩余约 20%）
        fetch_count = page * page_size * 5
        raw_results = self.redis.georadius(
            geo_key, user_lng, user_lat, radius_m,
            unit="m", withdist=True, withcoord=True,
            count=fetch_count, sort="ASC"
        )

        if not raw_results:
            return {"total": 0, "page": page, "merchants": []}

        # 2. 批量获取商家详情并过滤
        filtered = []
        pipe = self.redis.pipeline()
        merchant_ids = [str(r[0]) for r in raw_results]

        for mid in merchant_ids:
            detail_key = f"{self.DETAIL_KEY_PREFIX}:{mid}"
            pipe.hgetall(detail_key)

        details = pipe.execute()

        for (mid_bytes, distance, coord), detail in zip(raw_results, details):
            if not detail:
                continue

            # 解析详情
            merchant_detail = self._parse_detail(detail)

            # 过滤条件
            if category and merchant_detail.get("category") != category:
                continue
            if min_rating > 0 and float(merchant_detail.get("rating", 0)) < min_rating:
                continue
            if only_open and merchant_detail.get("is_open") != "1":
                continue

            filtered.append({
                "merchant_id": mid_bytes.decode() if isinstance(mid_bytes, bytes) else mid_bytes,
                "distance_m": round(distance, 1),
                "lat": coord[1],
                "lng": coord[0],
                "name": merchant_detail.get("name", ""),
                "category": merchant_detail.get("category", ""),
                "rating": float(merchant_detail.get("rating", 0)),
                "is_open": merchant_detail.get("is_open") == "1",
                "monthly_orders": int(merchant_detail.get("monthly_orders", 0)),
                "avg_delivery_minutes": int(merchant_detail.get("avg_delivery_minutes", 0)),
            })

        # 3. 排序
        if sort_by == "rating":
            filtered.sort(key=lambda x: (-x["rating"], x["distance_m"]))
        elif sort_by == "composite":
            # 综合排序：距离权重 35% + 评分权重 35% + 配送时长权重 30%
            for m in filtered:
                distance_score = max(0, 1 - m["distance_m"] / radius_m)
                rating_score = m["rating"] / 5.0
                delivery_score = max(0, 1 - m["avg_delivery_minutes"] / 60)
                m["_composite"] = (
                    0.35 * distance_score +
                    0.35 * rating_score +
                    0.30 * delivery_score
                )
            filtered.sort(key=lambda x: -x["_composite"])
        else:  # distance
            filtered.sort(key=lambda x: x["distance_m"])

        # 4. 分页
        total = len(filtered)
        start = (page - 1) * page_size
        end = start + page_size
        page_results = filtered[start:end]

        return {
            "total": total,
            "page": page,
            "page_size": page_size,
            "total_pages": (total + page_size - 1) // page_size,
            "merchants": page_results
        }

    def _parse_detail(self, detail):
        """解析 Redis Hash 中的商家详情"""
        if isinstance(detail, dict):
            return {k.decode() if isinstance(k, bytes) else k:
                    v.decode() if isinstance(v, bytes) else v
                    for k, v in detail.items()}
        return {}

    # ========== 商家详情缓存同步 ==========

    def sync_merchant_detail_cache(self, merchant_id, detail_dict):
        """将商家详情同步到 Redis 缓存（由 MySQL binlog 消费触发）

        Args:
            merchant_id: 商家ID
            detail_dict: 商家详情字典（name, category, rating, is_open 等）
        """
        detail_key = f"{self.DETAIL_KEY_PREFIX}:{merchant_id}"
        mapping = {k: str(v) for k, v in detail_dict.items()}
        self.redis.hset(detail_key, mapping=mapping)
        self.redis.expire(detail_key, 86400)
```

**Redis GEO 搜索性能分析：**

| 操作 | 数据量 | QPS | 延迟 P99 |
|------|--------|-----|---------|
| GEOADD（单条） | 50 万/集合 | 10 万 | < 1ms |
| GEOADD（Pipeline 1000条） | 50 万/集合 | 50 万 | < 5ms |
| GEORADIUS 3km | 50 万/集合 | 5 万 | 3-8ms |
| GEORADIUS + Pipeline 过滤 | 50 万/集合 | 2 万 | 10-20ms |

## 逆地理编码完整实现（层级多边形匹配）

```python
import json
from typing import List, Tuple, Optional, Dict

class ReverseGeocodingService:
    """逆地理编码服务：根据经纬度返回行政区划信息

    核心思路：
    1. 加载行政区划边界数据（国家/省/市/区）为多边形
    2. 从最大范围到最小范围逐级匹配（国家 → 省 → 市 → 区）
    3. 结果按 1km x 1km 网格缓存（减少重复计算）
    """

    # 缓存 Key 前缀
    CACHE_PREFIX = "reverse_geo"
    # 缓存网格精度：1km x 1km ≈ 0.01 度
    GRID_PRECISION = 2  # 小数点后 2 位 → 约 1.1km

    def __init__(self, redis_client, db_client):
        self.redis = redis_client
        self.db = db_client
        # 内存缓存：层级多边形数据（启动时加载，定期刷新）
        self.admin_polygons = {
            "country": [],   # [{name, polygon, ...}]
            "province": [],
            "city": [],
            "district": []
        }
        self.pip = PointInPolygon(include_boundary=True)
        self._load_admin_boundaries()

    # ========== 行政区划加载 ==========

    def _load_admin_boundaries(self):
        """从数据库加载行政区划边界多边形

        按层级加载：国家 → 省 → 市 → 区
        每个层级按面积降序排列（大区域优先匹配）
        """
        for level in ["country", "province", "city", "district"]:
            records = self.db.query(f"""
                SELECT id, name, parent_id, level, ST_AsGeoJSON(boundary) as geojson
                FROM admin_boundaries
                WHERE level = %s AND is_active = 1
                ORDER BY ST_Area(boundary) DESC
            """, level)

            polygons = []
            for rec in records:
                geojson = json.loads(rec["geojson"])
                coordinates = geojson["coordinates"]
                # 外部多边形
                outer = [(p[0], p[1]) for p in coordinates[0]]
                # 洞（如有）
                holes = [
                    [(p[0], p[1]) for p in hole]
                    for hole in coordinates[1:]
                ] if len(coordinates) > 1 else []

                polygons.append({
                    "id": rec["id"],
                    "name": rec["name"],
                    "parent_id": rec["parent_id"],
                    "level": rec["level"],
                    "outer": outer,
                    "holes": holes,
                    # 预计算包围盒（加速排除）
                    "bbox": (
                        min(p[0] for p in outer),
                        min(p[1] for p in outer),
                        max(p[0] for p in outer),
                        max(p[1] for p in outer)
                    )
                })

            self.admin_polygons[level] = polygons

    # ========== 逆地理编码核心 ==========

    def reverse_geocode(self, lat: float, lng: float) -> dict:
        """逆地理编码：经纬度 → 行政区划

        从最大范围到最小范围逐级匹配：
        国家 → 省 → 市 → 区

        Args:
            lat: 纬度
            lng: 经度

        Returns:
            {
                "country": "中国",
                "province": "北京市",
                "city": "北京市",
                "district": "朝阳区",
                "address": "中国 北京市 北京市 朝阳区",
                "source": "cache" / "compute"
            }
        """
        # 1. 检查缓存
        cache_key = self._get_cache_key(lat, lng)
        cached = self.redis.get(cache_key)
        if cached:
            result = json.loads(cached)
            result["source"] = "cache"
            return result

        # 2. 逐级匹配
        point = (lng, lat)  # 注意：GeoJSON 格式为 (lng, lat)
        result = {}

        # 国家级
        country = self._find_containing_polygon(point, "country")
        if country:
            result["country"] = country["name"]
            result["country_id"] = country["id"]

            # 省级（在匹配的国家下搜索）
            province = self._find_containing_polygon(
                point, "province", parent_id=country["id"]
            )
            if province:
                result["province"] = province["name"]
                result["province_id"] = province["id"]

                # 市级
                city = self._find_containing_polygon(
                    point, "city", parent_id=province["id"]
                )
                if city:
                    result["city"] = city["name"]
                    result["city_id"] = city["id"]

                    # 区级
                    district = self._find_containing_polygon(
                        point, "district", parent_id=city["id"]
                    )
                    if district:
                        result["district"] = district["name"]
                        result["district_id"] = district["id"]

        # 3. 组装地址
        result["address"] = " ".join(filter(None, [
            result.get("country", ""),
            result.get("province", ""),
            result.get("city", ""),
            result.get("district", "")
        ]))
        result["source"] = "compute"

        # 4. 写入缓存（1km x 1km 网格，同一网格内所有点返回相同结果）
        self.redis.setex(cache_key, 86400 * 7, json.dumps(result, ensure_ascii=False))

        return result

    def _find_containing_polygon(self, point: Tuple[float, float],
                                  level: str,
                                  parent_id: int = None) -> Optional[dict]:
        """在指定层级中查找包含给定点的多边形

        优化策略：
        1. 包围盒快速排除（O(1)）
        2. 如果指定 parent_id，只搜索该父级下的子区域
        """
        candidates = self.admin_polygons[level]

        for polygon_data in candidates:
            # 父级过滤
            if parent_id is not None and polygon_data["parent_id"] != parent_id:
                continue

            # 包围盒快速排除
            bbox = polygon_data["bbox"]
            if not (bbox[0] <= point[0] <= bbox[2] and
                    bbox[1] <= point[1] <= bbox[3]):
                continue

            # 精确点在多边形内判定
            if self.pip.contains_with_holes(
                point, polygon_data["outer"], polygon_data["holes"]
            ):
                return polygon_data

        return None

    def _get_cache_key(self, lat: float, lng: float) -> str:
        """计算缓存 Key：按 1km x 1km 网格聚合

        小数点后 2 位 ≈ 1.1km，同一网格内的所有点共享缓存
        """
        grid_lat = round(lat, self.GRID_PRECISION)
        grid_lng = round(lng, self.GRID_PRECISION)
        return f"{self.CACHE_PREFIX}:{grid_lat}:{grid_lng}"

    # ========== 批量逆地理编码 ==========

    def batch_reverse_geocode(self, points: List[Tuple[float, float]]) -> List[dict]:
        """批量逆地理编码

        Args:
            points: [(lat, lng), ...]

        Returns:
            编码结果列表
        """
        # 1. 先批量查缓存
        pipe = self.redis.pipeline()
        cache_keys = []
        for lat, lng in points:
            key = self._get_cache_key(lat, lng)
            cache_keys.append(key)
            pipe.get(key)
        cached_results = pipe.execute()

        # 2. 对缓存未命中的执行计算
        results = [None] * len(points)
        miss_indices = []

        for i, cached in enumerate(cached_results):
            if cached:
                results[i] = json.loads(cached)
                results[i]["source"] = "cache"
            else:
                miss_indices.append(i)

        # 3. 计算缓存未命中的点
        for i in miss_indices:
            lat, lng = points[i]
            results[i] = self.reverse_geocode(lat, lng)

        return results
```

**逆地理编码性能分析：**

| 操作 | 单次延迟 | 批量(100点) | 缓存命中率 |
|------|---------|-----------|-----------|
| 全量计算（4级匹配） | 2-5ms | 50-200ms | 0% |
| 缓存命中 | 0.5ms | 5ms | > 90% |
| 包围盒排除率 | 95%+ | - | - |

## 位置推荐引擎完整实现

```python
from datetime import datetime, time
from typing import List, Dict, Optional

class LocationRecommendationEngine:
    """基于位置的个性化推荐引擎

    三维推荐模型：
    1. 个性化维度：用户历史访问偏好
    2. 上下文维度：时间、天气、季节
    3. 社交维度：好友最近访问

    最终得分 = w1 * 个性化得分 + w2 * 上下文得分 + w3 * 社交得分
    """

    # 权重配置
    WEIGHT_PERSONAL = 0.40    # 个性化权重
    WEIGHT_CONTEXTUAL = 0.35  # 上下文权重
    WEIGHT_SOCIAL = 0.25      # 社交权重

    # 时间段映射
    TIME_PERIODS = {
        "breakfast":  (time(6, 0), time(9, 30)),
        "lunch":      (time(11, 0), time(13, 30)),
        "afternoon_tea": (time(14, 0), time(16, 30)),
        "dinner":     (time(17, 0), time(20, 30)),
        "midnight_snack": (time(21, 0), time(2, 0)),
    }

    # 天气 → 推荐偏好
    WEATHER_PREFERENCES = {
        "sunny":    {"outdoor": 1.2, "ice_cream": 1.5, "cold_drink": 1.3},
        "rainy":    {"indoor": 1.5, "hot_pot": 1.4, "delivery": 1.3},
        "cold":     {"hot_pot": 1.5, "soup": 1.4, "bakery": 1.2},
        "hot":      {"ice_cream": 1.5, "cold_drink": 1.4, "salad": 1.2},
        "snowy":    {"hot_pot": 1.6, "indoor": 1.4, "delivery": 1.3},
    }

    # 季节 → 推荐偏好
    SEASON_PREFERENCES = {
        "spring":   {"outdoor": 1.2, "salad": 1.1, "tea": 1.2},
        "summer":   {"ice_cream": 1.5, "cold_drink": 1.4, "bbq": 1.2},
        "autumn":   {"hot_pot": 1.1, "seafood": 1.2, "coffee": 1.1},
        "winter":   {"hot_pot": 1.6, "soup": 1.5, "bakery": 1.3},
    }

    def __init__(self, redis_client, db_client, proximity_service):
        self.redis = redis_client
        self.db = db_client
        self.proximity = proximity_service

    def recommend(self, user_id: str, lat: float, lng: float,
                  city_id: int, context: dict = None,
                  count: int = 20, radius_m: int = 5000) -> List[dict]:
        """生成位置推荐

        Args:
            user_id: 用户ID
            lat: 用户纬度
            lng: 用户经度
            city_id: 城市ID
            context: 上下文信息（weather, season 等）
            count: 推荐数量
            radius_m: 搜索半径（米）

        Returns:
            推荐结果列表，按综合得分降序排列
        """
        if context is None:
            context = {}

        # 1. 获取附近商家（基数集合）
        nearby = self.proximity.search_nearby(
            user_lat=lat, user_lng=lng, city_id=city_id,
            radius_m=radius_m, page=1, page_size=count * 3
        )
        candidates = nearby["merchants"]

        if not candidates:
            return []

        # 2. 计算各维度得分
        personal_scores = self._compute_personal_scores(user_id, candidates)
        contextual_scores = self._compute_contextual_scores(candidates, context)
        social_scores = self._compute_social_scores(user_id, candidates)

        # 3. 综合加权
        for m in candidates:
            mid = m["merchant_id"]
            m["_personal_score"] = personal_scores.get(mid, 0.0)
            m["_contextual_score"] = contextual_scores.get(mid, 0.0)
            m["_social_score"] = social_scores.get(mid, 0.0)

            m["recommendation_score"] = (
                self.WEIGHT_PERSONAL * m["_personal_score"] +
                self.WEIGHT_CONTEXTUAL * m["_contextual_score"] +
                self.WEIGHT_SOCIAL * m["_social_score"]
            )

        # 4. 按综合得分排序
        candidates.sort(key=lambda x: -x["recommendation_score"])

        # 5. 清理内部字段，返回结果
        results = []
        for m in candidates[:count]:
            results.append({
                "merchant_id": m["merchant_id"],
                "name": m["name"],
                "category": m["category"],
                "rating": m["rating"],
                "distance_m": m["distance_m"],
                "recommendation_score": round(m["recommendation_score"], 4),
                "reason": self._generate_reason(m),
            })

        return results

    # ========== 个性化维度 ==========

    def _compute_personal_scores(self, user_id: str,
                                 candidates: List[dict]) -> Dict[str, float]:
        """计算个性化得分：基于用户历史访问偏好

        逻辑：
        1. 获取用户最近 30 天的访问记录
        2. 统计各类别/标签的偏好权重
        3. 对每个候选商家计算匹配度
        """
        # 从 Redis 获取用户偏好向量（由离线任务定期更新）
        pref_key = f"user:pref:{user_id}"
        preferences = self.redis.hgetall(pref_key)

        if not preferences:
            # 新用户：无历史偏好，使用热门偏好
            return {m["merchant_id"]: 0.5 for m in candidates}

        # 解析偏好：{category: weight}
        pref_weights = {}
        for k, v in preferences.items():
            key = k.decode() if isinstance(k, bytes) else k
            val = float(v.decode() if isinstance(v, bytes) else v)
            pref_weights[key] = val

        scores = {}
        for m in candidates:
            mid = m["merchant_id"]
            category = m.get("category", "")
            # 基础分 = 类别偏好权重
            score = pref_weights.get(category, 0.1)

            # 评分加成：高评分商家额外加分
            rating = m.get("rating", 0)
            score *= (0.5 + rating / 10.0)

            # 距离衰减：距离越远得分越低
            distance = m.get("distance_m", 5000)
            distance_factor = max(0.1, 1 - distance / 5000)
            score *= distance_factor

            scores[mid] = min(1.0, score)

        return scores

    # ========== 上下文维度 ==========

    def _compute_contextual_scores(self, candidates: List[dict],
                                    context: dict) -> Dict[str, float]:
        """计算上下文得分：时间/天气/季节

        逻辑：
        1. 判断当前时间段（早餐/午餐/下午茶/晚餐/夜宵）
        2. 结合天气信息调整偏好
        3. 结合季节信息调整偏好
        """
        now = datetime.now().time()
        weather = context.get("weather", "sunny")
        season = context.get("season", self._detect_season())

        # 判断当前时间段
        current_period = self._get_time_period(now)

        scores = {}
        for m in candidates:
            mid = m["merchant_id"]
            category = m.get("category", "")
            tags = m.get("tags", [])

            score = 0.5  # 基础分

            # 时间段偏好
            period_preferences = {
                "breakfast": ["breakfast", "bakery", "coffee", "milk_tea"],
                "lunch": ["chinese_food", "fast_food", "japanese", "noodle"],
                "afternoon_tea": ["coffee", "milk_tea", "bakery", "dessert"],
                "dinner": ["chinese_food", "hot_pot", "bbq", "japanese"],
                "midnight_snack": ["bbq", "hot_pot", "snack", "noodle"],
            }
            if current_period:
                preferred = period_preferences.get(current_period, [])
                if category in preferred or any(t in preferred for t in tags):
                    score += 0.3

            # 天气偏好
            weather_prefs = self.WEATHER_PREFERENCES.get(weather, {})
            for pref_key, multiplier in weather_prefs.items():
                if category == pref_key or pref_key in tags:
                    score += 0.2 * (multiplier - 1.0)  # 超额部分作为加分

            # 季节偏好
            season_prefs = self.SEASON_PREFERENCES.get(season, {})
            for pref_key, multiplier in season_prefs.items():
                if category == pref_key or pref_key in tags:
                    score += 0.15 * (multiplier - 1.0)

            scores[mid] = min(1.0, max(0.0, score))

        return scores

    # ========== 社交维度 ==========

    def _compute_social_scores(self, user_id: str,
                                candidates: List[dict]) -> Dict[str, float]:
        """计算社交得分：好友最近访问

        逻辑：
        1. 获取用户好友列表
        2. 获取好友最近 7 天的访问记录
        3. 统计每个候选商家的好友访问次数
        4. 好友访问越多 → 社交得分越高
        """
        # 获取好友最近访问的商家
        friends_key = f"user:friends:{user_id}"
        friends = self.redis.smembers(friends_key)

        if not friends:
            return {m["merchant_id"]: 0.0 for m in candidates}

        # 批量获取好友最近访问
        recent_visits = {}  # merchant_id → visit_count
        pipe = self.redis.pipeline()
        friend_ids = [f.decode() if isinstance(f, bytes) else f for f in friends]

        for fid in friend_ids[:50]:  # 最多查 50 个好友
            visits_key = f"user:recent_visits:{fid}"
            pipe.zrange(visits_key, 0, -1, withscores=True)

        visit_results = pipe.execute()

        for result in visit_results:
            for item in result:
                merchant_id = item[0].decode() if isinstance(item[0], bytes) else item[0]
                recent_visits[merchant_id] = recent_visits.get(merchant_id, 0) + 1

        # 计算社交得分
        max_visits = max(recent_visits.values()) if recent_visits else 1
        scores = {}
        for m in candidates:
            mid = m["merchant_id"]
            visit_count = recent_visits.get(mid, 0)
            scores[mid] = min(1.0, visit_count / max_visits) if max_visits > 0 else 0.0

        return scores

    # ========== 辅助方法 ==========

    def _get_time_period(self, current_time: time) -> Optional[str]:
        """判断当前时间段"""
        for period, (start, end) in self.TIME_PERIODS.items():
            if start <= end:
                if start <= current_time <= end:
                    return period
            else:  # 跨午夜（如夜宵 21:00-02:00）
                if current_time >= start or current_time <= end:
                    return period
        return None

    def _detect_season(self) -> str:
        """根据月份判断季节"""
        month = datetime.now().month
        if month in (3, 4, 5):
            return "spring"
        elif month in (6, 7, 8):
            return "summer"
        elif month in (9, 10, 11):
            return "autumn"
        else:
            return "winter"

    def _generate_reason(self, merchant: dict) -> str:
        """生成推荐理由（可解释性）"""
        reasons = []
        if merchant["_personal_score"] > 0.6:
            reasons.append("符合你的口味偏好")
        if merchant["_contextual_score"] > 0.7:
            reasons.append("适合当前时段")
        if merchant["_social_score"] > 0.5:
            reasons.append("好友最近去过")
        if merchant["distance_m"] < 500:
            reasons.append("距离你很近")

        if not reasons:
            reasons.append("综合推荐")

        return "，".join(reasons)
```

**推荐引擎效果评估：**

| 指标 | 仅距离排序 | 距离+评分 | 三维推荐引擎 |
|------|-----------|----------|------------|
| 点击率（CTR） | 8% | 12% | 22% |
| 下单转化率 | 3% | 5% | 9% |
| 用户满意度评分 | 3.2/5 | 3.8/5 | 4.3/5 |
| 推荐多样性 | 低（附近优先） | 中 | 高（个性化+场景化） |

## 异常场景补充

### 场景：Redis GEO 命令失败降级

```
触发：Redis 集群主节点故障切换期间 → GEOADD/GEORADIUS 命令超时
      → 附近商家搜索返回空结果 → 用户看到"附近无商家"
检测：
  1. Redis 命令超时 > 500ms → 标记为降级状态
  2. 连续 3 次 GEOADD 失败 → 触发告警
  3. GEORADIUS 返回空但预期非空 → 疑似故障
处理：
  1. 降级到 Elasticsearch geo_distance 查询（延迟 30-80ms，可接受）
  2. GEOADD 写入失败 → 写入本地缓冲队列 → Redis 恢复后重试写入
  3. 降级状态持续 > 5 分钟 → 通知 SRE 值班
  4. 降级期间前端显示"搜索结果可能不完整"提示
预防：
  - Redis Cluster 多主多从部署（3 主 3 从）
  - GEOADD 写入使用 Pipeline + 重试（3 次）
  - 定期健康检查：每 10 秒 PING Redis 集群
```

### 场景：逆地理编码缓存过期

```
触发：行政区划变更（如撤县设区）→ 缓存中仍返回旧名称
      → 用户看到"XX 县"但实际已更名为"XX 区"
      → 影响地址展示和区域统计
检测：
  1. 行政区划变更时发布 Kafka 事件 → 消费者清除相关缓存
  2. 缓存 TTL = 7 天 → 最坏情况下 7 天后自动刷新
  3. 定期校验：每日抽样 1000 个网格 → 与数据库最新数据比对
处理：
  1. 收到行政区划变更事件 → 按区域批量清除缓存
     清除策略：受影响区域 1km 网格的缓存全部删除
  2. 下次查询时重新计算并写入新缓存
  3. 变更通知：向下游系统广播"XX 区划已变更"消息
预防：
  - 行政区划变更走审批流程 → 审批通过后自动触发缓存清理
  - 缓存版本号：admin_boundary_version → 版本变更时全量清除
  - 监控：缓存命中率突降 → 疑似区划变更，触发校验
```

### 场景：附近搜索性能劣化

```
触发：某城市商家数从 5 万暴增到 30 万（如城市合并或平台扩张）
      → GEORADIUS 单次查询从 5ms 劣化到 35ms → P99 超过 100ms
      → 用户搜索延迟明显 → 部分请求超时
检测：
  1. GEORADIUS 延迟监控：P99 > 50ms → 告警
  2. Sorted Set 大小监控：ZCARD > 10 万 → 预警
  3. Redis CPU 使用率 > 70% → 资源瓶颈
处理：
  1. 短期：按区域进一步分片
     原Key: merchants:geo:{city_id}
     新Key: merchants:geo:{city_id}:{district_id}
     查询时先判断用户所在区 → 只查对应区的 GEO 集合
  2. 中期：减少 GEORADIUS 返回数量
     count 参数从 200 降到 50 → 减少排序开销
     前端分页加载（每页 20 条）
  3. 长期：引入二级索引
     GeoHash 前缀 → Redis Hash 存储该网格内的商家列表
     查询时先按 GeoHash 网格定位 → 再在小范围内 GEORADIUS
预防：
  - 每个 GEO Key 的成员数上限：10 万
  - 超过上限自动触发分片迁移
  - 压测：每月模拟 30 万商家 × 10 万 QPS 验证性能
```

## 位置围栏事件处理完整实现

```python
class GeofenceEventHandler:
    """地理围栏事件处理：进入/离开/停留"""

    def process_location_update(self, user_id, lat, lng):
        """处理位置更新 → 检测围栏事件"""
        # 1. 查找用户当前所在围栏
        current_fences = self._find_containing_fences(lat, lng)

        # 2. 获取用户之前所在的围栏
        previous_fences = self.redis.smembers(f"user_fences:{user_id}")

        # 3. 计算进入/离开
        entered = current_fences - previous_fences
        exited = previous_fences - current_fences

        # 4. 防抖处理（短时间内反复进出 → 只触发一次）
        for fence_id in entered:
            debounce_key = f"fence_debounce:{user_id}:{fence_id}"
            if self.redis.exists(debounce_key):
                continue  # 防抖期内，忽略
            self.redis.setex(debounce_key, 60, "1")  # 60秒防抖
            self._on_enter_fence(user_id, fence_id, lat, lng)

        for fence_id in exited:
            debounce_key = f"fence_debounce:{user_id}:{fence_id}"
            if self.redis.exists(debounce_key):
                continue
            self.redis.setex(debounce_key, 60, "1")
            self._on_exit_fence(user_id, fence_id, lat, lng)

        # 5. 更新用户当前围栏
        self.redis.delete(f"user_fences:{user_id}")
        if current_fences:
            self.redis.sadd(f"user_fences:{user_id}", *current_fences)

    def _find_containing_fences(self, lat, lng):
        """查找包含该点的所有围栏（Redis GEO + 精确判断）"""
        # 1. Redis GEO 粗筛：5km 内的围栏中心点
        nearby = self.redis.georadius("fence_centers", lng, lat, 5, unit="km")

        containing = set()
        for fence_data in nearby:
            fence = json.loads(fence_data)
            fence_id = fence["id"]
            # 2. 精确判断：点是否在多边形内
            if self._point_in_polygon(lat, lng, fence["polygon"]):
                containing.add(fence_id)

        return containing

    def _point_in_polygon(self, lat, lng, polygon):
        """射线法判断点是否在多边形内"""
        n = len(polygon)
        inside = False
        j = n - 1
        for i in range(n):
            yi, xi = polygon[i]["lat"], polygon[i]["lng"]
            yj, xj = polygon[j]["lat"], polygon[j]["lng"]
            if ((yi > lat) != (yj > lat)) and \
               (lng < (xj - xi) * (lat - yi) / (yj - yi) + xi):
                inside = not inside
            j = i
        return inside

    def _on_enter_fence(self, user_id, fence_id, lat, lng):
        """围栏进入事件"""
        fence = self.db.get_fence(fence_id)
        self.db.insert("fence_events", {
            "user_id": user_id, "fence_id": fence_id,
            "event_type": "enter", "lat": lat, "lng": lng,
            "timestamp": now()
        })
        # 触发关联动作
        for action in fence.get("actions", []):
            self.action_executor.execute(action, user_id, fence_id)
```

## 轨迹分析服务

```python
class TrajectoryAnalyzer:
    """轨迹分析：停留点检测 + 通勤路线识别"""

    def detect_stay_points(self, user_id, hours=24):
        """检测停留点（在某位置停留超过 10 分钟）"""
        points = self.db.query(
            "SELECT * FROM user_locations WHERE user_id = %s "
            "AND timestamp > NOW() - INTERVAL %s HOUR "
            "ORDER BY timestamp", user_id, hours)

        stay_points = []
        i = 0
        while i < len(points):
            j = i + 1
            while j < len(points):
                dist = self.haversine(
                    points[i]["lat"], points[i]["lng"],
                    points[j]["lat"], points[j]["lng"])
                if dist > 50:  # 超过 50 米 → 离开停留区
                    break
                j += 1

            duration = (points[j-1]["timestamp"] - points[i]["timestamp"]).total_seconds() / 60
            if duration >= 10:  # 停留超过 10 分钟
                stay_points.append({
                    "lat": statistics.mean(p["lat"] for p in points[i:j]),
                    "lng": statistics.mean(p["lng"] for p in points[i:j]),
                    "arrive_time": points[i]["timestamp"],
                    "leave_time": points[j-1]["timestamp"],
                    "duration_minutes": duration
                })
            i = j

        return stay_points
```

## 异常场景补充

### 场景：围栏误触发（GPS 漂移）

```
触发：用户在围栏边界附近 → GPS 漂移导致反复进出围栏
检测：
  1. 同一围栏 5 分钟内进出 > 2 次 → 漂移
  2. 防抖机制过滤
处理：
  1. 防抖期 60 秒内忽略重复事件
  2. 漂移检测 → 暂停该用户该围栏事件 5 分钟
  3. 位置稳定后恢复
预防：防抖机制 + 漂移检测 + 围栏边界缓冲区
```

### 场景：轨迹数据量过大

```
触发：高频位置上报 → 存储和查询性能下降
检测：
  1. 单用户日位置点 > 10000 → 数据量过大
  2. 查询延迟 > 2 秒 → 性能问题
处理：
  1. 降低上报频率（移动中 5 秒 → 30 秒）
  2. 静止时不上报（加速度 < 阈值）
  3. 历史轨迹降采样（保留关键点：方向变化 > 15°）
预防：自适应上报频率 + 轨迹压缩 + 分区存储
```

## POI 搜索与推荐完整实现

```python
class POISearchService:
    """POI 搜索：附近搜索 + 分类筛选 + 排序"""

    def search_nearby(self, lat, lng, category=None, radius_km=5,
                      keyword=None, sort_by="distance", page=1, size=20):
        """附近 POI 搜索"""
        # 1. Redis GEO 粗筛
        geo_results = self.redis.georadius(
            "poi_locations", lng, lat, radius_km, unit="km",
            withdist=True, withcoord=True, sort="ASC", count=size * 3)

        # 2. 过滤条件
        filtered = []
        for poi_id, distance, coord in geo_results:
            poi = self.redis.hgetall(f"poi:{poi_id}")
            if not poi:
                continue
            if category and poi.get("category") != category:
                continue
            if keyword and keyword.lower() not in poi.get("name", "").lower():
                continue
            poi["distance_km"] = round(distance, 2)
            filtered.append(poi)

        # 3. 排序
        if sort_by == "rating":
            filtered.sort(key=lambda x: float(x.get("rating", 0)), reverse=True)
        elif sort_by == "popularity":
            filtered.sort(key=lambda x: int(x.get("visit_count", 0)), reverse=True)
        # distance already sorted by Redis

        # 4. 分页
        start = (page - 1) * size
        return filtered[start:start + size]

    def get_poi_detail(self, poi_id):
        """获取 POI 详情"""
        poi = self.db.get_poi(poi_id)
        # 补充实时的排队/等候信息
        if poi["category"] == "restaurant":
            wait_info = self.redis.hgetall(f"restaurant_wait:{poi_id}")
            poi["current_wait_minutes"] = int(wait_info.get("minutes", 0))

        return poi
```

## 导航路径规划

```python
class RoutePlanningService:
    """路径规划：最短路径 + 多交通方式"""

    def plan_route(self, origin, destination, mode="driving"):
        """规划路径"""
        if mode == "driving":
            return self._plan_driving(origin, destination)
        elif mode == "walking":
            return self._plan_walking(origin, destination)
        elif mode == "transit":
            return self._plan_transit(origin, destination)

    def _plan_driving(self, origin, destination):
        """驾车路径规划"""
        # 1. 获取路网数据
        # 2. A* 算法寻路
        # 3. 实时路况调整
        route = self.graph_router.find_path(
            origin, destination,
            weight="time_with_traffic")

        # 4. 分步导航指令
        steps = []
        for i, segment in enumerate(route["segments"]):
            steps.append({
                "instruction": self._generate_instruction(segment, i),
                "distance_m": segment["distance"],
                "duration_s": segment["duration"],
                "start_location": segment["start"],
                "end_location": segment["end"],
            })

        return {
            "total_distance_m": route["total_distance"],
            "total_duration_s": route["total_duration"],
            "steps": steps,
            "polyline": route["polyline"],
            "traffic_info": route.get("traffic_segments", [])
        }

    def _generate_instruction(self, segment, index):
        """生成导航指令"""
        turn_type = segment.get("turn_type", "straight")
        road_name = segment.get("road_name", "未知道路")
        instructions = {
            "straight": f"沿{road_name}直行",
            "left": f"左转进入{road_name}",
            "right": f"右转进入{road_name}",
            "slight_left": f"稍向左转，进入{road_name}",
            "slight_right": f"稍向右转，进入{road_name}",
            "uturn": f"掉头进入{road_name}",
            "arrive": "到达目的地",
        }
        return instructions.get(turn_type, f"沿{road_name}行驶")
```

## 异常场景补充

### 场景：POI 数据过期

```
触发：餐厅已搬迁但 POI 仍显示旧地址 → 用户导航到错误位置
检测：
  1. 用户到达后 5 分钟内再次搜索同类 → 可能地址错误
  2. 用户反馈"已关闭"或"地址不对" → 数据过期
处理：
  1. 标记 POI 为"待验证"
  2. 通知数据团队核实
  3. 核实期间 POI 标注"信息可能不准确"
预防：POI 数据定期验证 + 用户反馈机制 + 数据过期告警
```

### 场景：导航路径规划超时

```
触发：路网数据量大 → A* 算法计算超时 → 无法返回路径
检测：
  1. 路径规划耗时 > 3 秒 → 超时
  2. 超时 → 降级到直线距离估算
处理：
  1. 降级：返回直线距离 + 预估时间
  2. 后台继续计算最优路径
  3. 计算完成后推送更新
预防：路径缓存 + 降级策略 + 后台异步计算
```

## 热力图生成服务完整实现

```python
class HeatmapGenerator:
    """热力图生成：网格聚合 + 高斯核平滑 + 瓦片渲染"""

    GRID_SIZE = 500  # 500 米网格

    def generate_heatmap(self, region, time_range="hourly"):
        """生成热力图数据"""
        # 1. 网格聚合
        grid = self._aggregate_to_grid(region, time_range)

        # 2. 高斯核平滑
        smoothed = self._gaussian_smooth(grid, sigma=1.5)

        # 3. 热冷区检测
        zones = self._detect_zones(smoothed)

        return {
            "grid": smoothed,
            "zones": zones,
            "time_range": time_range,
            "generated_at": now().isoformat()
        }

    def _aggregate_to_grid(self, region, time_range):
        """聚合位置数据到网格"""
        time_filter = {
            "hourly": "AND timestamp > NOW() - INTERVAL 1 HOUR",
            "daily": "AND DATE(timestamp) = CURRENT_DATE",
            "weekly": "AND timestamp > NOW() - INTERVAL 7 DAY",
        }

        # 查询时间范围内的位置点
        points = self.db.query(
            "SELECT lat, lng FROM user_locations "
            "WHERE lat BETWEEN %s AND %s AND lng BETWEEN %s AND %s "
            + time_filter.get(time_range, ""),
            region["min_lat"], region["max_lat"],
            region["min_lng"], region["max_lng"])

        # 聚合到网格
        grid = {}
        for p in points:
            grid_lat = round(p["lat"] * 1000 / (self.GRID_SIZE / 11100)) / (1000 / (self.GRID_SIZE / 11100))
            grid_lng = round(p["lng"] * 1000 / (self.GRID_SIZE / (11100 * math.cos(math.radians(p["lat"]))))) / (1000 / (self.GRID_SIZE / (11100 * math.cos(math.radians(p["lat"])))))
            key = f"{grid_lat:.4f},{grid_lng:.4f}"
            grid[key] = grid.get(key, 0) + 1

        return grid

    def _gaussian_smooth(self, grid, sigma=1.5):
        """高斯核平滑"""
        smoothed = {}
        kernel_size = int(sigma * 3)

        for key, count in grid.items():
            lat, lng = map(float, key.split(","))
            total = count  # 自身
            weight_sum = 1.0  # 自身权重

            for dlat in range(-kernel_size, kernel_size + 1):
                for dlng in range(-kernel_size, kernel_size + 1):
                    if dlat == 0 and dlng == 0:
                        continue
                    neighbor_key = f"{lat + dlat * 0.005:.4f},{lng + dlng * 0.005:.4f}"
                    if neighbor_key in grid:
                        dist = math.sqrt(dlat**2 + dlng**2)
                        weight = math.exp(-dist**2 / (2 * sigma**2))
                        total += grid[neighbor_key] * weight
                        weight_sum += weight

            smoothed[key] = round(total / weight_sum, 1)

        return smoothed

    def _detect_zones(self, smoothed):
        """热冷区检测"""
        if not smoothed:
            return {"hot": [], "cold": []}

        values = list(smoothed.values())
        mean = statistics.mean(values)
        std = statistics.stdev(values) if len(values) > 1 else 0

        hot_threshold = mean + 2 * std
        cold_threshold = max(0, mean - 1 * std)

        hot = []
        cold = []
        for key, value in smoothed.items():
            lat, lng = map(float, key.split(","))
            if value >= hot_threshold:
                hot.append({"lat": lat, "lng": lng, "intensity": value})
            elif value <= cold_threshold:
                cold.append({"lat": lat, "lng": lng, "intensity": value})

        # 热区告警
        if len(hot) > 0 and max(h["intensity"] for h in hot) > mean * 5:
            self.alert(f"检测到异常热区: 最大密度 {max(h['intensity'] for h in hot):.0f}")

        return {"hot": sorted(hot, key=lambda x: x["intensity"], reverse=True)[:10],
                "cold": cold[:10]}

    def generate_tile(self, region, zoom_level, x, y):
        """生成热力图瓦片（用于地图渲染）"""
        # 根据缩放级别计算瓦片范围
        tile_bounds = self._tile_to_bounds(zoom_level, x, y)

        heatmap = self.generate_heatmap(tile_bounds, time_range="hourly")

        # 渲染为 PNG 瓦片
        return self._render_tile_png(heatmap["grid"], tile_bounds, zoom_level)
```

## 位置隐私保护

```python
class LocationPrivacyService:
    """位置隐私保护：混淆 + 聚合 + 差分隐私"""

    PRIVACY_LEVELS = {
        "precise": {"offset_m": 0, "desc": "精确位置"},
        "coarse": {"offset_m": 500, "desc": "约 500 米精度"},
        "city": {"offset_m": 5000, "desc": "城市级精度"},
        "off": {"offset_m": None, "desc": "不共享位置"},
    }

    def obfuscate_location(self, user_id, lat, lng, privacy_level="coarse"):
        """位置混淆"""
        if privacy_level == "off":
            return None
        if privacy_level == "precise":
            return {"lat": lat, "lng": lng}

        offset_m = self.PRIVACY_LEVELS[privacy_level]["offset_m"]
        # 随机偏移（均匀分布在圆内）
        angle = random.uniform(0, 2 * math.pi)
        radius = random.uniform(0, offset_m)
        offset_lat = radius / 111000 * math.cos(angle)
        offset_lng = radius / (111000 * math.cos(math.radians(lat))) * math.sin(angle)

        return {
            "lat": round(lat + offset_lat, 6),
            "lng": round(lng + offset_lng, 6),
            "privacy_level": privacy_level
        }

    def apply_differential_privacy(self, aggregate_count, epsilon=1.0):
        """差分隐私：添加拉普拉斯噪声"""
        sensitivity = 1  # 每个用户最多贡献 1 个位置点
        scale = sensitivity / epsilon
        noise = random.gauss(0, scale)  # 拉普拉斯噪声近似
        return max(0, int(aggregate_count + noise))

    def check_user_consent(self, user_id, purpose):
        """检查用户隐私同意"""
        consent = self.db.query_one(
            "SELECT * FROM privacy_consents WHERE user_id = %s "
            "AND purpose = %s AND revoked_at IS NULL",
            user_id, purpose)
        if not consent:
            return {"allowed": False, "reason": "用户未授权"}
        if consent["privacy_level"] == "off":
            return {"allowed": False, "reason": "用户已关闭位置共享"}
        return {"allowed": True, "privacy_level": consent["privacy_level"]}

    def enforce_retention(self):
        """执行数据保留策略（30 天后删除原始位置）"""
        deleted = self.db.execute(
            "DELETE FROM user_locations "
            "WHERE timestamp < NOW() - INTERVAL 30 DAY")
        # 保留聚合数据（热力图、统计）
        return {"raw_deleted": deleted}
```

## 异常场景补充

### 场景：热力图计算 OOM

```
触发：一线城市早高峰 → 位置点数百万 → 网格聚合内存不足
检测：
  1. 热力图生成耗时 > 30 秒 → 可能 OOM
  2. 内存使用 > 80% → 告警
处理：
  1. 分区域计算（按 10km×10km 分片）
  2. 使用预聚合表（每小时增量计算）
  3. 降低网格精度（500m → 1km）
预防：增量预聚合 + 分区域计算 + 内存限制
```

### 场景：位置去匿名化攻击

```
触发：攻击者通过交叉引用（家庭地址+工作地址）识别匿名用户
检测：
  1. 大量位置查询来自同一 IP → 爬虫攻击
  2. 查询模式：先查区域热力图 → 再查具体 POI → 去匿名化
处理：
  1. API 限流：同一 IP 每分钟最多 10 次查询
  2. 聚合数据添加差分隐私噪声
  3. 禁止精确位置批量查询
预防：差分隐私 + API 限流 + 最小数据暴露原则
```

## 完整热力图生成服务（增强版）

```python
import numpy as np
import math
from collections import defaultdict
from datetime import datetime, timedelta

class HeatmapGenerationServiceEnhanced:
    """
    完整热力图生成服务（增强版）
    功能：
      - 500m×500m 网格聚合用户位置
      - 时间维度热力图（每小时/每天/每周）
      - 高斯核密度平滑
      - 多缩放级别瓦片生成
      - 热区/冷区检测与阈值告警
    """

    GRID_CELL_SIZE = 500          # 500m × 500m 网格单元
    GAUSSIAN_SIGMA = 1.5          # 高斯核平滑标准差（网格单元数）
    DENSITY_THRESHOLD_HOT = 1000  # 热区阈值：每网格单元人数
    DENSITY_THRESHOLD_COLD = 10   # 冷区阈值

    def __init__(self, redis, db, tile_storage):
        self.redis = redis
        self.db = db
        self.tile_storage = tile_storage  # S3/OSS 瓦片存储

    def aggregate_positions_to_grid(self, positions, bounds):
        """
        将用户位置聚合到 500m×500m 网格单元
        positions: [{"user_id": "u1", "lat": 39.9, "lng": 116.4, "timestamp": "..."}]
        bounds: {"min_lat": 39.5, "max_lat": 40.0, "min_lng": 116.0, "max_lng": 117.0}
        返回: (grid数组, 行数, 列数)
        """
        # 1. 计算网格维度
        lat_span = bounds["max_lat"] - bounds["min_lat"]
        lng_span = bounds["max_lng"] - bounds["min_lng"]
        # 1度纬度约111km，1度经度约111km×cos(lat)
        meters_per_deg_lat = 111000
        meters_per_deg_lng = 111000 * math.cos(math.radians(bounds["min_lat"]))

        rows = max(1, int(lat_span * meters_per_deg_lat / self.GRID_CELL_SIZE))
        cols = max(1, int(lng_span * meters_per_deg_lng / self.GRID_CELL_SIZE))

        # 2. 聚合位置到网格
        grid = np.zeros((rows, cols), dtype=np.int32)
        for pos in positions:
            row = int((pos["lat"] - bounds["min_lat"]) * meters_per_deg_lat / self.GRID_CELL_SIZE)
            col = int((pos["lng"] - bounds["min_lng"]) * meters_per_deg_lng / self.GRID_CELL_SIZE)
            if 0 <= row < rows and 0 <= col < cols:
                grid[row][col] += 1

        return grid, rows, cols

    def generate_time_based_heatmap(self, region, period="hourly"):
        """
        基于时间维度的热力图生成
        period: hourly（每小时）/ daily（每天）/ weekly（每周）
        返回: 热力图时间序列数据
        """
        now = datetime.utcnow()
        time_ranges = {
            "hourly": [(now - timedelta(hours=i), now - timedelta(hours=i-1)) for i in range(24, 0, -1)],
            "daily": [(now - timedelta(days=i), now - timedelta(days=i-1)) for i in range(7, 0, -1)],
            "weekly": [(now - timedelta(weeks=i), now - timedelta(weeks=i-1)) for i in range(4, 0, -1)],
        }

        bounds = self._get_region_bounds(region)
        heatmap_series = []

        for start, end in time_ranges[period]:
            # 从数据库获取时间范围内的位置数据
            positions = self.db.query(
                "SELECT user_id, lat, lng, timestamp "
                "FROM user_positions "
                "WHERE region = %s AND timestamp BETWEEN %s AND %s "
                "AND precision_level >= 'coarse'",
                region, start, end
            )

            if not positions:
                continue

            grid, rows, cols = self.aggregate_positions_to_grid(positions, bounds)

            # 高斯核平滑
            smoothed = self._gaussian_kernel_smoothing(grid)

            heatmap_series.append({
                "time_start": start.isoformat(),
                "time_end": end.isoformat(),
                "grid_data": smoothed.tolist(),
                "rows": rows,
                "cols": cols,
                "total_users": len(positions),
                "max_density": float(np.max(smoothed)),
                "avg_density": float(np.mean(smoothed[smoothed > 0])) if np.any(smoothed > 0) else 0,
            })

        # 缓存结果
        cache_key = f"heatmap:{region}:{period}:{now.strftime('%Y%m%d%H')}"
        self.redis.setex(cache_key, 3600, json.dumps(heatmap_series))

        return heatmap_series

    def _gaussian_kernel_smoothing(self, grid):
        """
        高斯核平滑：消除噪声，使热力图更自然
        使用 scipy 的 gaussian_filter 进行二维高斯卷积
        """
        from scipy.ndimage import gaussian_filter
        smoothed = gaussian_filter(grid.astype(np.float64), sigma=self.GAUSSIAN_SIGMA)
        return np.maximum(smoothed, 0)  # 确保非负

    def generate_heatmap_tiles(self, region, zoom_levels=[8, 10, 12, 14]):
        """
        生成不同缩放级别的热力图瓦片（用于地图渲染）
        瓦片坐标系：z/x/y（Google 瓦片方案）
        每个瓦片为 256×256 RGBA PNG 图片
        """
        bounds = self._get_region_bounds(region)

        # 获取最新位置数据
        positions = self.db.query(
            "SELECT lat, lng FROM user_positions "
            "WHERE region = %s AND timestamp > NOW() - INTERVAL 1 HOUR "
            "AND precision_level >= 'coarse'",
            region
        )

        grid, rows, cols = self.aggregate_positions_to_grid(positions, bounds)
        smoothed = self._gaussian_kernel_smoothing(grid)

        tiles_generated = []
        for zoom in zoom_levels:
            # 计算该缩放级别下的瓦片范围
            tile_range = self._get_tile_range(bounds, zoom)

            for tx in range(tile_range["min_x"], tile_range["max_x"] + 1):
                for ty in range(tile_range["min_y"], tile_range["max_y"] + 1):
                    # 生成 256×256 RGBA 热力图瓦片
                    tile_image = self._render_tile(smoothed, bounds, zoom, tx, ty)
                    tile_key = f"heatmap/{region}/{zoom}/{tx}/{ty}.png"
                    self.tile_storage.upload(tile_key, tile_image, content_type="image/png")
                    tiles_generated.append(tile_key)

        return {
            "region": region,
            "zoom_levels": zoom_levels,
            "tiles_count": len(tiles_generated),
            "generated_at": datetime.utcnow().isoformat(),
        }

    def _render_tile(self, grid, bounds, zoom, tx, ty):
        """渲染单个热力图瓦片为 PNG 图片"""
        from PIL import Image
        TILE_SIZE = 256

        img = Image.new("RGBA", (TILE_SIZE, TILE_SIZE), (0, 0, 0, 0))

        # 计算瓦片对应的地理范围
        tile_bounds = self._tile_to_bounds(zoom, tx, ty)

        # 将网格数据映射到瓦片像素
        max_val = np.max(grid) if np.max(grid) > 0 else 1
        for py in range(TILE_SIZE):
            for px in range(TILE_SIZE):
                # 像素 → 地理坐标 → 网格坐标
                lat = tile_bounds["max_lat"] - (py / TILE_SIZE) * (tile_bounds["max_lat"] - tile_bounds["min_lat"])
                lng = tile_bounds["min_lng"] + (px / TILE_SIZE) * (tile_bounds["max_lng"] - tile_bounds["min_lng"])

                row, col = self._geo_to_grid(lat, lng, bounds, grid.shape)
                if 0 <= row < grid.shape[0] and 0 <= col < grid.shape[1]:
                    intensity = grid[row][col] / max_val
                    rgba = self._intensity_to_color(intensity)
                    img.putpixel((px, py), rgba)

        import io
        buf = io.BytesIO()
        img.save(buf, format="PNG")
        return buf.getvalue()

    def _intensity_to_color(self, intensity):
        """密度值 → 颜色映射（蓝→绿→黄→红）"""
        intensity = min(max(intensity, 0), 1)
        if intensity < 0.25:
            r, g, b = 0, int(intensity * 4 * 255), 255
        elif intensity < 0.5:
            r, g, b = 0, 255, int((1 - (intensity - 0.25) * 4) * 255)
        elif intensity < 0.75:
            r, g, b = int((intensity - 0.5) * 4 * 255), 255, 0
        else:
            r, g, b = 255, int((1 - (intensity - 0.75) * 4) * 255), 0
        alpha = int(intensity * 200)  # 半透明
        return (r, g, b, alpha)

    def detect_hot_cold_zones(self, region):
        """
        热区/冷区检测，超过阈值触发告警
        热区：密度超过 DENSITY_THRESHOLD_HOT → 拥挤预警
        冷区：密度低于 DENSITY_THRESHOLD_COLD → 流量异常低
        """
        bounds = self._get_region_bounds(region)
        positions = self.db.query(
            "SELECT lat, lng FROM user_positions "
            "WHERE region = %s AND timestamp > NOW() - INTERVAL 30 MINUTE "
            "AND precision_level >= 'coarse'",
            region
        )

        grid, rows, cols = self.aggregate_positions_to_grid(positions, bounds)
        smoothed = self._gaussian_kernel_smoothing(grid)

        hot_zones = []
        cold_zones = []

        for row in range(rows):
            for col in range(cols):
                density = smoothed[row][col]
                lat, lng = self._grid_to_geo(row, col, bounds, rows, cols)

                if density >= self.DENSITY_THRESHOLD_HOT:
                    hot_zones.append({
                        "grid_row": row, "grid_col": col,
                        "lat": lat, "lng": lng,
                        "density": float(density),
                        "severity": "critical" if density >= self.DENSITY_THRESHOLD_HOT * 2 else "warning",
                    })
                elif 0 < density <= self.DENSITY_THRESHOLD_COLD:
                    cold_zones.append({
                        "grid_row": row, "grid_col": col,
                        "lat": lat, "lng": lng,
                        "density": float(density),
                    })

        # 热区告警
        for zone in hot_zones:
            if zone["severity"] == "critical":
                self.alert(
                    f"热区告警 [{region}]: 位置({zone['lat']:.4f}, {zone['lng']:.4f}) "
                    f"密度 {zone['density']:.0f} 超过阈值 {self.DENSITY_THRESHOLD_HOT}，"
                    f"存在拥挤风险"
                )

        return {
            "region": region,
            "hot_zones": hot_zones,
            "cold_zones": cold_zones,
            "hot_zone_count": len(hot_zones),
            "cold_zone_count": len(cold_zones),
            "detected_at": datetime.utcnow().isoformat(),
        }

    def _get_region_bounds(self, region):
        """获取区域边界"""
        bounds_map = {
            "beijing": {"min_lat": 39.4, "max_lat": 41.1, "min_lng": 115.4, "max_lng": 117.5},
            "shanghai": {"min_lat": 30.7, "max_lat": 31.9, "min_lng": 120.8, "max_lng": 122.2},
            "guangzhou": {"min_lat": 22.5, "max_lat": 23.9, "min_lng": 112.9, "max_lng": 114.5},
        }
        return bounds_map.get(region, bounds_map["beijing"])

    def _grid_to_geo(self, row, col, bounds, rows, cols):
        """网格坐标 → 地理坐标"""
        lat_span = bounds["max_lat"] - bounds["min_lat"]
        lng_span = bounds["max_lng"] - bounds["min_lng"]
        lat = bounds["min_lat"] + (row + 0.5) * lat_span / rows
        lng = bounds["min_lng"] + (col + 0.5) * lng_span / cols
        return lat, lng

    def _geo_to_grid(self, lat, lng, bounds, shape):
        """地理坐标 → 网格坐标"""
        meters_per_deg_lat = 111000
        meters_per_deg_lng = 111000 * math.cos(math.radians(bounds["min_lat"]))
        row = int((lat - bounds["min_lat"]) * meters_per_deg_lat / self.GRID_CELL_SIZE)
        col = int((lng - bounds["min_lng"]) * meters_per_deg_lng / self.GRID_CELL_SIZE)
        return row, col

    def _get_tile_range(self, bounds, zoom):
        """计算地理范围对应的瓦片坐标范围"""
        n = 2 ** zoom
        min_x = int((bounds["min_lng"] + 180) / 360 * n)
        max_x = int((bounds["max_lng"] + 180) / 360 * n)
        lat_rad_min = math.radians(bounds["max_lat"])
        lat_rad_max = math.radians(bounds["min_lat"])
        min_y = int((1 - math.log(math.tan(lat_rad_min) + 1/math.cos(lat_rad_min)) / math.pi) / 2 * n)
        max_y = int((1 - math.log(math.tan(lat_rad_max) + 1/math.cos(lat_rad_max)) / math.pi) / 2 * n)
        return {"min_x": min_x, "max_x": max_x, "min_y": min_y, "max_y": max_y}

    def _tile_to_bounds(self, zoom, tx, ty):
        """瓦片坐标 → 地理范围"""
        n = 2 ** zoom
        min_lng = tx / n * 360 - 180
        max_lng = (tx + 1) / n * 360 - 180
        lat_rad_min = math.atan(math.sinh(math.pi * (1 - 2 * ty / n)))
        lat_rad_max = math.atan(math.sinh(math.pi * (1 - 2 * (ty + 1) / n)))
        min_lat = math.degrees(lat_rad_max)
        max_lat = math.degrees(lat_rad_min)
        return {"min_lat": min_lat, "max_lat": max_lat, "min_lng": min_lng, "max_lng": max_lng}

    def alert(self, message):
        """发送告警"""
        print(f"[ALERT] {message}")
```

## 基于位置的推送通知

```python
class LocationBasedPushService:
    """基于位置的推送通知：围栏触发营销 + 近场推荐 + 频次限制 + A/B测试"""

    MAX_PUSH_PER_DAY = 3  # 每用户每天最多推送次数

    def __init__(self, redis, db, push_client, geofence_service):
        self.redis = redis
        self.db = db
        self.push_client = push_client
        self.geofence_service = geofence_service

    def on_geofence_enter(self, user_id, geofence_id):
        """
        围栏进入触发：营销推送
        示例：用户进入商场围栏 → "欢迎光临！全场商品9折优惠"
        """
        # 1. 检查用户是否允许位置推送
        consent = self._get_user_consent(user_id)
        if not consent or not consent.get("marketing_push", False):
            self.log(f"用户 {user_id} 未授权营销推送，跳过")
            return None

        # 2. 频次限制检查
        if not self._check_push_frequency(user_id):
            self.log(f"用户 {user_id} 今日推送已达上限")
            return None

        # 3. 获取围栏关联的推送模板
        geofence = self.geofence_service.get_geofence(geofence_id)
        push_template = self._get_push_template(geofence)
        if not push_template:
            return None

        # 4. A/B 测试变体选择
        variant = self._select_ab_variant(user_id, push_template["campaign_id"])

        # 5. 组装推送内容
        push_content = self._render_push(push_template, variant, geofence)

        # 6. 发送推送
        result = self.push_client.send(
            user_id=user_id,
            title=push_content["title"],
            body=push_content["body"],
            data={
                "type": "geofence_marketing",
                "geofence_id": geofence_id,
                "campaign_id": push_template["campaign_id"],
                "variant": variant,
                "poi_id": geofence.get("poi_id"),
            }
        )

        # 7. 记录推送日志
        self._log_push(user_id, push_template["campaign_id"], variant, geofence_id)

        return result

    def proximity_recommendation(self, user_id, lat, lng, radius_m=500):
        """
        近场推荐：用户附近的 POI + 优惠信息
        示例：用户附近500米有咖啡店 → "星巴克就在你旁边，美式买一送一"
        """
        # 1. 检查用户偏好和授权
        consent = self._get_user_consent(user_id)
        if not consent or not consent.get("proximity_recommendation", False):
            return []

        # 2. 频次限制
        if not self._check_push_frequency(user_id):
            return []

        # 3. 搜索附近有优惠的 POI
        nearby_pois = self.db.query(
            "SELECT p.*, d.title as deal_title, d.discount, d.deal_id "
            "FROM pois p "
            "JOIN deals d ON p.poi_id = d.poi_id "
            "WHERE d.status = 'active' "
            "AND ST_Distance_Sphere(p.location, ST_MakePoint(%s, %s)) <= %s "
            "ORDER BY ST_Distance_Sphere(p.location, ST_MakePoint(%s, %s)) "
            "LIMIT 5",
            lng, lat, radius_m, lng, lat
        )

        if not nearby_pois:
            return []

        # 4. 根据用户偏好排序
        user_prefs = self._get_user_preferences(user_id)
        ranked = self._rank_by_preference(nearby_pois, user_prefs)

        # 5. 推送最近的一个推荐
        best_match = ranked[0]
        distance = self._calculate_distance(lat, lng, best_match["lat"], best_match["lng"])

        push_content = {
            "title": f"附近推荐：{best_match['name']}",
            "body": f"距离你{distance:.0f}米 · {best_match['deal_title']}",
        }

        # A/B 测试
        variant = self._select_ab_variant(user_id, "proximity_rec")

        result = self.push_client.send(
            user_id=user_id,
            title=push_content["title"],
            body=push_content["body"],
            data={
                "type": "proximity_recommendation",
                "poi_id": best_match["poi_id"],
                "deal_id": best_match.get("deal_id"),
                "distance_m": distance,
                "variant": variant,
            }
        )

        self._log_push(user_id, "proximity_rec", variant, best_match["poi_id"])
        return result

    def _check_push_frequency(self, user_id):
        """推送频次限制：每天最多 MAX_PUSH_PER_DAY 次"""
        today = datetime.utcnow().strftime("%Y%m%d")
        key = f"push:freq:{user_id}:{today}"
        count = self.redis.incr(key)
        if count == 1:
            self.redis.expire(key, 86400)  # 1天过期
        return count <= self.MAX_PUSH_PER_DAY

    def _select_ab_variant(self, user_id, campaign_id):
        """A/B 测试变体选择（基于用户ID哈希分桶）"""
        experiment = self.db.get(
            "SELECT * FROM ab_experiments "
            "WHERE campaign_id = %s AND status = 'running'",
            campaign_id
        )
        if not experiment:
            return "control"

        # 哈希分桶
        bucket = hash(f"{user_id}:{campaign_id}") % 100
        variants = json.loads(experiment["variant_distribution"])
        # variants: [{"name": "control", "weight": 50}, {"name": "variant_a", "weight": 50}]
        cumulative = 0
        for v in variants:
            cumulative += v["weight"]
            if bucket < cumulative:
                return v["name"]
        return "control"

    def manage_user_opt(self, user_id, settings):
        """
        用户推送偏好管理（opt-in/out）
        settings: {
            "marketing_push": True,            # 围栏营销推送
            "proximity_recommendation": True,   # 近场推荐
            "location_precision": "coarse",     # precise/coarse/city-level
            "quiet_hours": {"start": "22:00", "end": "08:00"},
        }
        """
        self.db.upsert("user_push_settings", {
            "user_id": user_id,
            "marketing_push": settings.get("marketing_push", False),
            "proximity_recommendation": settings.get("proximity_recommendation", False),
            "location_precision": settings.get("location_precision", "coarse"),
            "quiet_hours_start": settings.get("quiet_hours", {}).get("start", "22:00"),
            "quiet_hours_end": settings.get("quiet_hours", {}).get("end", "08:00"),
            "updated_at": datetime.utcnow(),
        }, key="user_id")

        # 清除缓存
        self.redis.delete(f"push:consent:{user_id}")
        return {"status": "updated", "user_id": user_id}

    def _get_user_consent(self, user_id):
        """获取用户推送授权"""
        cached = self.redis.get(f"push:consent:{user_id}")
        if cached:
            return json.loads(cached)

        consent = self.db.get(
            "SELECT * FROM user_push_settings WHERE user_id = %s", user_id)
        if consent:
            self.redis.setex(f"push:consent:{user_id}", 300, json.dumps(consent))
        return consent

    def _get_push_template(self, geofence):
        """获取围栏关联的推送模板"""
        return self.db.get(
            "SELECT * FROM push_templates "
            "WHERE geofence_id = %s AND status = 'active' "
            "ORDER BY priority DESC LIMIT 1",
            geofence["id"]
        )

    def _render_push(self, template, variant, geofence):
        """渲染推送内容（支持变体）"""
        if variant == "control":
            return {
                "title": template["title"].format(name=geofence.get("name", "")),
                "body": template["body"].format(discount=template.get("discount", "10%")),
            }
        elif variant == "variant_a":
            return {
                "title": template.get("variant_a_title", template["title"]).format(name=geofence.get("name", "")),
                "body": template.get("variant_a_body", template["body"]).format(discount=template.get("discount", "10%")),
            }
        return {"title": template["title"], "body": template["body"]}

    def _log_push(self, user_id, campaign_id, variant, ref_id):
        """记录推送日志"""
        self.db.insert("push_logs", {
            "user_id": user_id,
            "campaign_id": campaign_id,
            "variant": variant,
            "ref_id": ref_id,
            "pushed_at": datetime.utcnow(),
        })

    def _get_user_preferences(self, user_id):
        """获取用户偏好"""
        prefs = self.redis.get(f"user:prefs:{user_id}")
        return json.loads(prefs) if prefs else {}

    def _rank_by_preference(self, pois, prefs):
        """根据偏好排序 POI"""
        favorite_categories = prefs.get("favorite_categories", [])
        for poi in pois:
            poi["score"] = 0
            if poi["category"] in favorite_categories:
                poi["score"] += 10
            poi["score"] += float(poi.get("discount", 0)) * 0.5
        return sorted(pois, key=lambda x: x["score"], reverse=True)

    def _calculate_distance(self, lat1, lng1, lat2, lng2):
        """计算两点间距离（米）"""
        lat1, lng1, lat2, lng2 = map(math.radians, [lat1, lng1, lat2, lng2])
        dlat = lat2 - lat1
        dlng = lng2 - lng1
        a = math.sin(dlat/2)**2 + math.cos(lat1) * math.cos(lat2) * math.sin(dlng/2)**2
        return 6371000 * 2 * math.asin(math.sqrt(a))

    def log(self, message):
        """日志输出"""
        print(f"[LocationPush] {message}")
```

## 位置隐私保护（增强版）

```python
import random
import hashlib
from datetime import datetime, timedelta

class LocationPrivacyServiceEnhanced:
    """
    位置隐私保护（增强版）
    功能：
      - 位置模糊化（100-500m随机偏移）
      - 时间聚合（每N分钟上报一次而非实时）
      - 差分隐私（校准拉普拉斯噪声）
      - 用户同意管理（细粒度：precise/coarse/city_level/off）
      - 数据保留策略（原始位置30天后删除）
    """

    # 数据保留策略
    RAW_RETENTION_DAYS = 30          # 原始位置数据保留30天
    OBFUSCATED_RETENTION_DAYS = 180  # 模糊化数据保留180天
    AGGREGATED_RETENTION_DAYS = 365  # 聚合统计数据保留365天

    def __init__(self, db, redis, consent_client):
        self.db = db
        self.redis = redis
        self.consent_client = consent_client

    def obfuscate_location(self, lat, lng, precision_level="coarse"):
        """
        位置模糊化：根据精度等级添加随机偏移
        precision_level:
          - precise: 无偏移（仅限必要场景如导航）
          - coarse: 100-500m 随机偏移
          - city_level: 替换为城市中心点
          - off: 不上报位置
        """
        if precision_level == "precise":
            return {"lat": lat, "lng": lng, "obfuscated": False}

        elif precision_level == "coarse":
            # 添加 100-500m 随机偏移
            offset_meters = random.uniform(100, 500)
            angle = random.uniform(0, 2 * math.pi)
            dlat = offset_meters * math.cos(angle) / 111000
            dlng = offset_meters * math.sin(angle) / (111000 * math.cos(math.radians(lat)))
            return {
                "lat": lat + dlat,
                "lng": lng + dlng,
                "obfuscated": True,
                "offset_meters": offset_meters,
            }

        elif precision_level == "city_level":
            # 替换为城市中心点
            city_center = self._get_city_center(lat, lng)
            return {
                "lat": city_center["lat"],
                "lng": city_center["lng"],
                "obfuscated": True,
                "precision": "city",
            }

        elif precision_level == "off":
            return None

    def temporal_aggregation(self, user_id, positions, report_interval_minutes=10):
        """
        时间聚合：不是实时上报位置，而是每隔N分钟上报一次
        将时间窗口内的多个位置点聚合为中心点
        positions: 该用户最近的原始位置序列
        """
        if not positions:
            return None

        # 按时间间隔分组
        interval = timedelta(minutes=report_interval_minutes)
        groups = defaultdict(list)

        for pos in positions:
            # 计算所属时间窗口
            window_start = pos["timestamp"].replace(
                minute=(pos["timestamp"].minute // report_interval_minutes) * report_interval_minutes,
                second=0, microsecond=0
            )
            groups[window_start].append(pos)

        # 每个窗口只保留一个聚合位置（中心点）
        aggregated = []
        for window_start, window_positions in groups.items():
            avg_lat = sum(p["lat"] for p in window_positions) / len(window_positions)
            avg_lng = sum(p["lng"] for p in window_positions) / len(window_positions)

            aggregated.append({
                "user_id": user_id,
                "lat": avg_lat,
                "lng": avg_lng,
                "timestamp": window_start,
                "sample_count": len(window_positions),
                "aggregation_type": "temporal",
            })

        return aggregated

    def apply_differential_privacy(self, grid_data, epsilon=1.0):
        """
        差分隐私：向密度数据添加校准的拉普拉斯噪声
        epsilon: 隐私预算，越小隐私保护越强（推荐 0.1-1.0）
        """
        sensitivity = 1  # 单个用户对网格计数的最大贡献
        scale = sensitivity / epsilon

        noisy_grid = np.copy(grid_data).astype(np.float64)
        rows, cols = noisy_grid.shape

        for i in range(rows):
            for j in range(cols):
                # 添加拉普拉斯噪声
                noise = np.random.laplace(0, scale)
                noisy_grid[i][j] = max(0, noisy_grid[i][j] + noise)  # 非负约束

        return noisy_grid

    def manage_user_consent(self, user_id, consent_settings):
        """
        用户同意管理（细粒度权限控制）
        consent_settings: {
            "location_precision": "coarse",     # precise/coarse/city_level/off
            "marketing_push": True,
            "proximity_recommendation": False,
            "navigation": True,                 # 导航需要精确位置
            "analytics": "coarse",              # 分析用途的精度
            "third_party_sharing": False,        # 是否允许第三方共享
            "consent_version": "2.0",            # 同意条款版本
        }
        """
        # 1. 验证同意条款版本
        current_version = self.consent_client.get_latest_version()
        if consent_settings.get("consent_version") != current_version:
            return {"error": "consent_version_outdated", "latest": current_version}

        # 2. 存储同意设置
        self.db.upsert("user_location_consent", {
            "user_id": user_id,
            "location_precision": consent_settings.get("location_precision", "off"),
            "marketing_push": consent_settings.get("marketing_push", False),
            "proximity_recommendation": consent_settings.get("proximity_recommendation", False),
            "navigation": consent_settings.get("navigation", False),
            "analytics_precision": consent_settings.get("analytics", "off"),
            "third_party_sharing": consent_settings.get("third_party_sharing", False),
            "consent_version": consent_settings["consent_version"],
            "consented_at": datetime.utcnow(),
        }, key="user_id")

        # 3. 更新缓存
        self.redis.setex(
            f"consent:location:{user_id}",
            3600,
            json.dumps(consent_settings)
        )

        # 4. 如果精度降级，立即模糊化现有位置数据
        if consent_settings.get("location_precision") in ["coarse", "city_level", "off"]:
            self._reobfuscate_existing_data(user_id, consent_settings["location_precision"])

        return {"status": "updated", "user_id": user_id}

    def _reobfuscate_existing_data(self, user_id, precision_level):
        """对现有位置数据重新模糊化"""
        recent_positions = self.db.query(
            "SELECT id, lat, lng FROM user_positions "
            "WHERE user_id = %s AND timestamp > NOW() - INTERVAL 24 HOUR",
            user_id
        )

        for pos in recent_positions:
            obfuscated = self.obfuscate_location(pos["lat"], pos["lng"], precision_level)
            if obfuscated:
                self.db.update("user_positions", {
                    "lat": obfuscated["lat"],
                    "lng": obfuscated["lng"],
                    "precision_level": precision_level,
                    "obfuscated": obfuscated.get("obfuscated", False),
                }, {"id": pos["id"]})

    def enforce_data_retention_policy(self):
        """
        数据保留策略执行
        - 原始位置数据：30天后删除
        - 模糊化数据：180天后删除
        - 聚合统计：365天后归档
        """
        now = datetime.utcnow()

        # 1. 删除过期的原始位置数据
        raw_cutoff = now - timedelta(days=self.RAW_RETENTION_DAYS)
        deleted_raw = self.db.execute(
            "DELETE FROM user_positions_raw "
            "WHERE timestamp < %s AND precision_level = 'precise'",
            raw_cutoff
        )
        self.log(f"删除过期原始位置数据: {deleted_raw} 条")

        # 2. 将30天前的精确位置数据模糊化后迁移
        migrated = self.db.execute(
            "UPDATE user_positions "
            "SET lat = lat + (RANDOM() * 0.01 - 0.005), "
            "    lng = lng + (RANDOM() * 0.01 - 0.005), "
            "    precision_level = 'coarse', "
            "    obfuscated = TRUE "
            "WHERE timestamp < %s AND precision_level = 'precise'",
            raw_cutoff
        )
        self.log(f"模糊化迁移精确位置数据: {migrated} 条")

        # 3. 删除过期的模糊化数据
        obf_cutoff = now - timedelta(days=self.OBFUSCATED_RETENTION_DAYS)
        deleted_obf = self.db.execute(
            "DELETE FROM user_positions "
            "WHERE timestamp < %s AND precision_level IN ('coarse', 'city_level')",
            obf_cutoff
        )
        self.log(f"删除过期模糊化位置数据: {deleted_obf} 条")

        # 4. 归档过期聚合数据
        agg_cutoff = now - timedelta(days=self.AGGREGATED_RETENTION_DAYS)
        self.db.execute(
            "INSERT INTO location_stats_archive "
            "SELECT * FROM location_stats WHERE period_end < %s "
            "ON CONFLICT DO NOTHING",
            agg_cutoff
        )
        self.db.execute("DELETE FROM location_stats WHERE period_end < %s", agg_cutoff)

        return {
            "deleted_raw": deleted_raw,
            "migrated_to_coarse": migrated,
            "deleted_obfuscated": deleted_obf,
            "executed_at": now.isoformat(),
        }

    def _get_city_center(self, lat, lng):
        """获取城市中心点（用于 city_level 精度）"""
        city = self.db.get(
            "SELECT name, lat, lng FROM cities "
            "ORDER BY ST_Distance_Sphere(location, ST_MakePoint(%s, %s)) "
            "LIMIT 1",
            lng, lat
        )
        if city:
            return {"name": city["name"], "lat": city["lat"], "lng": city["lng"]}
        return {"name": "unknown", "lat": lat, "lng": lng}

    def log(self, message):
        """日志输出"""
        print(f"[LocationPrivacy] {message}")
```

## 异常场景补充（增强版）

### 场景：热力图计算 OOM（高密度城市区域）

```
触发：北京核心区域（三环内）高峰时段用户位置数 > 500万
      → 聚合到 500m×500m 网格 → 矩阵 100×100 仍可接受
      → 但高斯核平滑时 scipy 的 gaussian_filter 对超大矩阵内存溢出
      → 或生成缩放级别14的瓦片时，瓦片数量爆炸（>10万张）
检测：
  1. 热力图生成进程内存使用 > 80% → 告警
  2. 瓦片生成任务超时（> 5分钟）→ 可能 OOM
  3. 进程被 OOM Killer 终止 → 确认 OOM
处理：
  1. 分区域计算：将大区域拆分为子区域，分别计算后合并
  2. 降采样：高密度区域使用更大的网格单元（1km×1km）
  3. 限制瓦片缩放级别：高密度区域最多生成到 zoom=12
  4. 流式瓦片生成：不一次性生成所有瓦片，按需生成并缓存
  5. 内存限制：单次计算内存上限 2GB，超过则分批处理
预防：
  - 根据区域用户密度自适应调整网格大小
  - 瓦片生成任务设置内存限制和超时
  - 高密度区域使用预计算 + 增量更新策略
  - 监控热力图生成任务的资源消耗
```

### 场景：位置隐私泄露（去匿名化攻击）

```
触发：攻击者通过公开的差分隐私热力图数据
      + 外部知识（用户社交媒体签到、工作时间表）
      → 交叉关联分析 → 去匿名化 → 识别出特定用户的位置轨迹
示例：
  1. 热力图显示某区域早上9点有1人（差分隐私噪声后仍可辨识低密度区域）
  2. 攻击者知道用户A每天9点在该区域（通过社交媒体公开的工作地点）
  3. 交叉关联 → 确认热力图中该数据点就是用户A
检测：
  1. 低密度网格单元（< 5人）在差分隐私后仍可辨识 → 隐私风险
  2. 差分隐私 epsilon 值过大（> 5）→ 保护不足
  3. 数据查询异常：同一用户频繁请求不同区域的热力图 → 可能是攻击者
处理：
  1. 提高差分隐私保护强度：降低 epsilon（推荐 0.1-1.0）
  2. 低密度区域抑制：网格单元人数 < k（k=5）时置零不输出
  3. 合并小区域：将低密度网格与邻近网格合并
  4. 热力图查询频率限制：同一用户每小时最多查询 10 次
  5. 已泄露用户：强制重置其位置数据精度为 city_level
预防：
  - 差分隐私 epsilon 严格控制在 1.0 以下
  - 实施k-匿名：任何输出区域至少包含 k 个用户
  - 定期隐私审计：模拟去匿名化攻击测试保护强度
  - 限制热力图数据的公开粒度和查询频率
  - 敏感区域（医院、军事基地）默认不出现在热力图中
```

## 地理围栏服务完整实现

```python
class GeofenceService:
    """地理围栏：创建 + 判断 + 事件触发"""

    def create_geofence(self, name, geometry, rules):
        """创建地理围栏"""
        fence_id = str(uuid4())

        self.db.insert("geofences", {
            "fence_id": fence_id,
            "name": name,
            "geometry": json.dumps(geometry),  # {"type": "circle", "center": [lat, lng], "radius_m": 500}
            "rules": json.dumps(rules),  # {"on_enter": "notify", "on_exit": "alert", "dwell_threshold_min": 30}
            "status": "active",
            "created_at": now()
        })

        # 如果是圆形围栏 → 写入 Redis GEO 用于快速过滤
        if geometry["type"] == "circle":
            center_lat, center_lng = geometry["center"]
            self.redis.geoadd("geofence_centers", center_lng, center_lat, fence_id)
            self.redis.set(f"geofence_radius:{fence_id}", geometry["radius_m"])

        return {"fence_id": fence_id, "name": name}

    def check_position(self, lat, lng, entity_id=None):
        """检查位置是否在任何围栏内"""
        triggered = []

        # 1. Redis 粗筛（查找 5km 内的围栏中心点）
        nearby = self.redis.geosearch("geofence_centers", lng, lat,
            radius=5000, unit="m")

        for fence_id in nearby:
            fence_id = fence_id if isinstance(fence_id, str) else fence_id.decode()
            geometry = self._get_fence_geometry(fence_id)

            # 2. 精确判断
            inside = self._point_in_geometry(lat, lng, geometry)

            if inside:
                # 3. 检查是否新进入
                was_inside = self.redis.sismember(f"entity_in_fence:{fence_id}", entity_id) if entity_id else False

                if not was_inside and entity_id:
                    # 新进入
                    self.redis.sadd(f"entity_in_fence:{fence_id}", entity_id)
                    rules = self._get_fence_rules(fence_id)
                    if rules.get("on_enter") == "notify":
                        triggered.append({"fence_id": fence_id, "event": "enter",
                            "action": "notify"})
                    elif rules.get("on_enter") == "alert":
                        triggered.append({"fence_id": fence_id, "event": "enter",
                            "action": "alert"})

                # 4. 检查停留时间
                rules = self._get_fence_rules(fence_id)
                if rules.get("dwell_threshold_min") and entity_id:
                    enter_time = self.redis.get(f"dwell:{fence_id}:{entity_id}")
                    if enter_time:
                        dwell_minutes = (now() - datetime.fromisoformat(enter_time.decode())).total_seconds() / 60
                        if dwell_minutes >= rules["dwell_threshold_min"]:
                            triggered.append({"fence_id": fence_id, "event": "dwell",
                                "duration_minutes": round(dwell_minutes, 0)})
                    else:
                        self.redis.setex(f"dwell:{fence_id}:{entity_id}",
                            rules["dwell_threshold_min"] * 120, now().isoformat())
            else:
                # 离开围栏
                if entity_id:
                    was_inside = self.redis.sismember(f"entity_in_fence:{fence_id}", entity_id)
                    if was_inside:
                        self.redis.srem(f"entity_in_fence:{fence_id}", entity_id)
                        self.redis.delete(f"dwell:{fence_id}:{entity_id}")
                        rules = self._get_fence_rules(fence_id)
                        if rules.get("on_exit"):
                            triggered.append({"fence_id": fence_id, "event": "exit",
                                "action": rules["on_exit"]})

        return {"position": {"lat": lat, "lng": lng}, "triggered": triggered}

    def _point_in_geometry(self, lat, lng, geometry):
        """判断点是否在几何图形内"""
        if geometry["type"] == "circle":
            center_lat, center_lng = geometry["center"]
            radius_m = geometry["radius_m"]
            distance = self._haversine(lat, lng, center_lat, center_lng)
            return distance <= radius_m

        elif geometry["type"] == "polygon":
            return self._point_in_polygon(lat, lng, geometry["vertices"])

        return False

    def _point_in_polygon(self, lat, lng, vertices):
        """射线法判断点是否在多边形内"""
        n = len(vertices)
        inside = False
        j = n - 1
        for i in range(n):
            yi, xi = vertices[i]
            yj, xj = vertices[j]
            if ((yi > lng) != (yj > lng)) and (lat < (xj - xi) * (lng - yi) / (yj - yi) + xi):
                inside = not inside
            j = i
        return inside
```

## 异常场景补充

### 场景：围栏判断精度不足

```
触发：GPS 精度 ±50m → 围栏半径 100m → 位置漂移导致频繁进出围栏 → 误触发
检测：
  1. 同一实体进出围栏频率 > 5 次/分钟 → 漂移
  2. GPS 精度 < 围栏半径的 50% → 精度不足
处理：
  1. 增加防抖（进出围栏需持续 30 秒以上）
  2. 最小围栏半径限制（> 200m）
  3. 使用 Wi-Fi 指纹辅助定位（室内）
预防：防抖机制 + 最小半径 + 多源定位
```

### 场景：围栏数量过多导致查询慢

```
触发：10 万个围栏 → 每次位置更新需检查所有围栏 → 延迟 > 1 秒
检测：
  1. 位置检查延迟 > 200ms → 围栏数量过多
  2. Redis GEO 搜索结果 > 100 → 候选过多
处理：
  1. 分级过滤：Redis GEO 粗筛 → 几何精判
  2. 围栏分片（按区域分组）
  3. 增量检查（只检查上次附近围栏）
预防：分级过滤 + 围栏分片 + 增量检查
```

## 地理位置轨迹平滑与预测完整实现

```python
class TrajectorySmoothingService:
    """轨迹平滑：卡尔曼滤波 + 异常点剔除 + 目的地预测"""

    def smooth_trajectory(self, raw_points):
        """平滑原始轨迹点"""
        if len(raw_points) < 3:
            return raw_points

        # 1. 异常点剔除（速度阈值）
        cleaned = self._remove_outliers(raw_points)

        # 2. 卡尔曼滤波平滑
        smoothed = self._kalman_filter(cleaned)

        # 3. 道路吸附（将点吸附到最近的道路上）
        snapped = self._road_snap(smoothed)

        return snapped

    def _remove_outliers(self, points):
        """剔除异常点"""
        if len(points) < 3:
            return points

        cleaned = [points[0]]
        for i in range(1, len(points)):
            prev = points[i - 1]
            curr = points[i]

            # 计算速度
            distance = self._haversine(prev["lat"], prev["lng"],
                                       curr["lat"], curr["lng"])
            time_diff = (curr["timestamp"] - prev["timestamp"]).total_seconds()
            speed_mps = distance / max(time_diff, 1)

            # 超速阈值（300 km/h → 异常）
            if speed_mps > 83.3:
                continue  # 跳过异常点

            cleaned.append(curr)

        return cleaned

    def _kalman_filter(self, points):
        """卡尔曼滤波"""
        # 状态: [lat, lng, v_lat, v_lng]
        # 初始化
        dt = 1.0  # 时间步长

        # 状态转移矩阵
        F = np.array([
            [1, 0, dt, 0],
            [0, 1, 0, dt],
            [0, 0, 1, 0],
            [0, 0, 0, 1]
        ])

        # 观测矩阵（只观测位置）
        H = np.array([
            [1, 0, 0, 0],
            [0, 1, 0, 0]
        ])

        # 过程噪声
        Q = np.eye(4) * 0.01
        # 观测噪声
        R = np.eye(2) * 0.0001

        # 初始状态
        x = np.array([points[0]["lat"], points[0]["lng"], 0, 0])
        P = np.eye(4) * 1.0

        smoothed = []

        for point in points:
            # 预测
            x_pred = F @ x
            P_pred = F @ P @ F.T + Q

            # 更新
            z = np.array([point["lat"], point["lng"]])
            y = z - H @ x_pred
            S = H @ P_pred @ H.T + R
            K = P_pred @ H.T @ np.linalg.inv(S)
            x = x_pred + K @ y
            P = (np.eye(4) - K @ H) @ P_pred

            smoothed.append({
                "lat": round(x[0], 6),
                "lng": round(x[1], 6),
                "timestamp": point["timestamp"],
                "confidence": round(1 - np.trace(P) / 4, 3)
            })

        return smoothed

    def predict_destination(self, trajectory_points):
        """预测目的地"""
        if len(trajectory_points) < 5:
            return {"confidence": 0, "destination": None}

        # 获取最近 5 个点的方向
        recent = trajectory_points[-5:]
        total_dx = sum(recent[i]["lng"] - recent[i-1]["lng"] for i in range(1, len(recent)))
        total_dy = sum(recent[i]["lat"] - recent[i-1]["lat"] for i in range(1, len(recent)))
        avg_dx = total_dx / (len(recent) - 1)
        avg_dy = total_dy / (len(recent) - 1)

        # 外推 30 分钟
        last = recent[-1]
        speed = math.sqrt(avg_dx**2 + avg_dy**2)
        if speed < 0.00001:
            return {"confidence": 0, "destination": None}

        extrapolation_steps = 30  # 30 步
        predicted_lat = last["lat"] + avg_dy * extrapolation_steps
        predicted_lng = last["lng"] + avg_dx * extrapolation_steps

        # 匹配到已知目的地（家/公司）
        known_locations = self._get_known_locations(trajectory_points[0].get("user_id"))
        best_match = None
        best_distance = float('inf')

        for loc in known_locations:
            dist = self._haversine(predicted_lat, predicted_lng,
                                   loc["lat"], loc["lng"])
            if dist < best_distance:
                best_distance = dist
                best_match = loc

        if best_match and best_distance < 2000:  # 2km 内匹配
            return {
                "destination": best_match,
                "confidence": round(max(0, 1 - best_distance / 2000), 2),
                "predicted_arrival_minutes": round(best_distance / (speed * 111000) * 60, 0)
            }

        return {
            "destination": {"lat": round(predicted_lat, 4), "lng": round(predicted_lng, 4),
                           "name": "预测位置"},
            "confidence": 0.3,
            "predicted_arrival_minutes": None
        }
```

## 异常场景补充

### 场景：卡尔曼滤波平滑过度

```
触发：卡尔曼滤波过度平滑 → 转弯处轨迹被拉直 → 偏离实际道路 → 导航错误
检测：
  1. 平滑后轨迹与实际道路偏差 > 50m → 过度平滑
  2. 转弯点被平滑掉 → 道路吸附失败
处理：
  1. 调整过程噪声 Q（增大 → 更信任观测）
  2. 转弯检测：转弯处不平滑
  3. 道路吸附修正
预防：自适应 Q 矩阵 + 转弯检测 + 道路吸附
```

### 场景：目的地预测频繁变更

```
触发：用户在城市中行驶 → 预测目的地从 A 变 B 变 C → 频繁变更 → 用户体验差
检测：
  1. 预测目的地 5 分钟内变更 > 3 次 → 不稳定
  2. 预测置信度低 → 不可靠
处理：
  1. 低置信度不展示预测
  2. 预测结果稳定后才展示（连续 3 次一致）
  3. 只对已知地点预测（家/公司）
预防：置信度阈值 + 稳定性检查 + 已知地点优先
```

## 地理位置热点分析完整实现

```python
class GeoHotspotAnalysisService:
    """地理热点分析：DBSCAN 聚类 + 热力图 + 时变分析"""

    def detect_hotspots(self, points, eps_meters=500, min_points=10):
        """DBSCAN 检测地理热点"""
        n = len(points)
        visited = [False] * n
        cluster_id = 0
        clusters = {}

        for i in range(n):
            if visited[i]:
                continue
            visited[i] = True

            # 找到邻域内的所有点
            neighbors = self._range_query(points, i, eps_meters)

            if len(neighbors) < min_points:
                # 噪声点
                points[i]["cluster"] = -1
                continue

            # 扩展聚类
            cluster_id += 1
            clusters[cluster_id] = [i]
            points[i]["cluster"] = cluster_id

            seed_set = list(neighbors)
            j = 0
            while j < len(seed_set):
                q = seed_set[j]
                if not visited[q]:
                    visited[q] = True
                    q_neighbors = self._range_query(points, q, eps_meters)
                    if len(q_neighbors) >= min_points:
                        for n_idx in q_neighbors:
                            if n_idx not in seed_set:
                                seed_set.append(n_idx)

                if points[q].get("cluster", -1) == -1:
                    points[q]["cluster"] = cluster_id
                    if q not in clusters[cluster_id]:
                        clusters[cluster_id].append(q)

                j += 1

        # 计算每个聚类的中心点和统计信息
        result_clusters = []
        for cid, member_indices in clusters.items():
            members = [points[i] for i in member_indices]

            # 加权中心点（按权重/数量加权）
            total_weight = sum(m.get("weight", 1) for m in members)
            center_lat = sum(m["lat"] * m.get("weight", 1) for m in members) / total_weight
            center_lng = sum(m["lng"] * m.get("weight", 1) for m in members) / total_weight

            # 计算覆盖半径
            max_dist = max(self._haversine(center_lat, center_lng,
                m["lat"], m["lng"]) for m in members)

            result_clusters.append({
                "cluster_id": cid,
                "center_lat": round(center_lat, 6),
                "center_lng": round(center_lng, 6),
                "member_count": len(members),
                "total_weight": round(total_weight, 1),
                "radius_meters": round(max_dist, 0),
                "density": round(len(members) / max(math.pi * (max_dist / 1000) ** 2, 0.001), 1)
            })

        # 按密度排序
        result_clusters.sort(key=lambda c: c["density"], reverse=True)

        return {
            "total_points": n,
            "cluster_count": len(result_clusters),
            "noise_count": sum(1 for p in points if p.get("cluster") == -1),
            "clusters": result_clusters
        }

    def _range_query(self, points, center_idx, eps_meters):
        """查找 eps 范围内的邻居"""
        center = points[center_idx]
        neighbors = []
        for i, p in enumerate(points):
            if self._haversine(center["lat"], center["lng"],
                              p["lat"], p["lng"]) <= eps_meters:
                neighbors.append(i)
        return neighbors

    def analyze_temporal_hotspots(self, region_lat, region_lng, radius_km=10,
                                 time_granularity="hour"):
        """时变热点分析（不同时段的热点分布变化）"""
        # 获取区域内所有位置事件
        events = self.db.query(
            "SELECT lat, lng, timestamp, weight FROM location_events "
            "WHERE ST_DWithin(location, ST_MakePoint(%s, %s), %s) "
            "AND timestamp > NOW() - INTERVAL 7 DAY "
            "ORDER BY timestamp",
            region_lng, region_lat, radius_km * 1000)

        if not events:
            return {"status": "no_data"}

        # 按时段分组
        time_slots = {}
        for event in events:
            if time_granularity == "hour":
                slot_key = event["timestamp"].strftime("%Y%m%d%H")
            elif time_granularity == "day_of_week":
                slot_key = str(event["timestamp"].weekday())
            else:
                slot_key = event["timestamp"].strftime("%Y%m%d")

            time_slots.setdefault(slot_key, []).append({
                "lat": event["lat"], "lng": event["lng"],
                "weight": event.get("weight", 1)
            })

        # 每个时段独立做热点检测
        temporal_hotspots = {}
        for slot, points in time_slots.items():
            if len(points) >= 5:
                hotspots = self.detect_hotspots(points, eps_meters=300, min_points=5)
                temporal_hotspots[slot] = hotspots["clusters"]

        return {
            "region": {"lat": region_lat, "lng": region_lng, "radius_km": radius_km},
            "time_granularity": time_granularity,
            "time_slots_analyzed": len(temporal_hotspots),
            "temporal_hotspots": temporal_hotspots
        }
```

## 异常场景补充

### 场景：DBSCAN 参数选择不当

```
触发：eps=500m 对城市合适 → 对郊区太大 → 郊区多个热点被合并为一个 → 分析失真
检测：
  1. 聚类半径 > 2km → 可能 eps 过大
  2. 噪声点比例 > 50% → 可能 eps 过小
处理：
  1. 自适应 eps（根据点密度调整）
  2. 多尺度分析（不同 eps 分别检测）
  3. 人工调参
预防：自适应 eps + 多尺度 + 参数验证
```

### 场景：时变分析数据量过大

```
触发：7 天 × 24 小时 = 168 个时段 → 每个时段独立 DBSCAN → 计算耗时 > 10 分钟
检测：
  1. 时变分析耗时 > 5 分钟 → 数据量过大
  2. 内存使用过高 → 需要优化
处理：
  1. 合并相似时段（如工作日高峰合并）
  2. 采样分析（每小时取 10% 数据）
  3. 预计算 + 缓存
预防：时段合并 + 采样 + 预计算缓存
```

## 地理围栏触发与通知完整实现

```python
class GeofenceTriggerService:
    """地理围栏：围栏定义 → 进出检测 → 事件触发 → 通知推送"""

    FENCE_TYPES = {
        "circle": "圆形围栏",
        "polygon": "多边形围栏",
        "corridor": "走廊围栏（线段缓冲区）",
    }

    def create_geofence(self, fence_name, fence_type, geometry, triggers, metadata=None):
        """创建地理围栏"""
        fence_id = str(uuid4())

        # 验证几何数据
        if fence_type == "circle":
            if not all(k in geometry for k in ["center_lat", "center_lng", "radius_m"]):
                raise ValueError("圆形围栏需要 center_lat, center_lng, radius_m")
        elif fence_type == "polygon":
            if len(geometry.get("vertices", [])) < 3:
                raise ValueError("多边形围栏至少需要 3 个顶点")

        self.db.insert("geofences", {
            "fence_id": fence_id,
            "name": fence_name,
            "type": fence_type,
            "geometry": json.dumps(geometry),
            "triggers": json.dumps(triggers),  # ["enter", "exit", "dwell"]
            "dwell_duration_seconds": triggers.get("dwell_duration", 300) if "dwell" in triggers else None,
            "metadata": json.dumps(metadata or {}),
            "status": "active",
            "created_at": now()
        })

        # 索引到 Redis Geo（用于快速围栏查询）
        if fence_type == "circle":
            self.redis.geoadd("geofence_index",
                geometry["center_lng"], geometry["center_lat"], fence_id)
            self.redis.set(f"geofence:{fence_id}", json.dumps({
                "type": "circle",
                "center_lat": geometry["center_lat"],
                "center_lng": geometry["center_lng"],
                "radius_m": geometry["radius_m"]
            }))

        return {"fence_id": fence_id, "name": fence_name, "type": fence_type}

    def check_position(self, device_id, lat, lng, timestamp=None):
        """检查设备位置是否触发围栏"""
        timestamp = timestamp or now()

        # 1. 查找附近的围栏
        nearby_fences = self.redis.georadius(
            "geofence_index", lng, lat, 10, unit="km", withdist=True)

        if not nearby_fences:
            return {"device_id": device_id, "triggered_fences": []}

        triggered = []
        previous_state = self._get_device_fence_state(device_id)

        for fence_data in nearby_fences:
            fence_id = fence_data[0].decode() if isinstance(fence_data[0], bytes) else fence_data[0]
            distance = fence_data[1]

            fence_info = json.loads(self.redis.get(f"geofence:{fence_id}"))
            was_inside = fence_id in previous_state.get("inside", [])

            # 2. 判断是否在围栏内
            is_inside = self._is_inside_fence(lat, lng, fence_info)

            # 3. 检测触发事件
            if is_inside and not was_inside:
                # 进入围栏
                triggered.append({
                    "fence_id": fence_id,
                    "event": "enter",
                    "timestamp": timestamp.isoformat()
                })
                self._fire_trigger(device_id, fence_id, "enter", lat, lng, timestamp)

            elif not is_inside and was_inside:
                # 离开围栏
                triggered.append({
                    "fence_id": fence_id,
                    "event": "exit",
                    "timestamp": timestamp.isoformat()
                })
                self._fire_trigger(device_id, fence_id, "exit", lat, lng, timestamp)

            elif is_inside and was_inside:
                # 围栏内停留 → 检查驻留触发
                entered_at = previous_state["inside"][fence_id].get("entered_at")
                if entered_at:
                    dwell_seconds = (timestamp - datetime.fromisoformat(entered_at)).total_seconds()
                    fence = self.db.get_geofence(fence_id)
                    dwell_target = fence.get("dwell_duration_seconds", 300)

                    if dwell_seconds >= dwell_target and not previous_state["inside"][fence_id].get("dwell_triggered"):
                        triggered.append({
                            "fence_id": fence_id,
                            "event": "dwell",
                            "dwell_seconds": dwell_seconds,
                            "timestamp": timestamp.isoformat()
                        })
                        self._fire_trigger(device_id, fence_id, "dwell", lat, lng, timestamp)

        # 4. 更新设备围栏状态
        self._update_device_fence_state(device_id, lat, lng, triggered, timestamp)

        return {"device_id": device_id, "lat": lat, "lng": lng,
                "triggered_fences": triggered}

    def _is_inside_fence(self, lat, lng, fence_info):
        """判断点是否在围栏内"""
        if fence_info["type"] == "circle":
            distance = self._haversine(lat, lng,
                fence_info["center_lat"], fence_info["center_lng"])
            return distance <= fence_info["radius_m"]

        elif fence_info["type"] == "polygon":
            return self._point_in_polygon(lat, lng, fence_info["vertices"])

        return False

    def _point_in_polygon(self, lat, lng, vertices):
        """射线法判断点是否在多边形内"""
        n = len(vertices)
        inside = False

        j = n - 1
        for i in range(n):
            yi, xi = vertices[i]["lat"], vertices[i]["lng"]
            yj, xj = vertices[j]["lat"], vertices[j]["lng"]

            if ((yi > lat) != (yj > lat)) and \
               (lng < (xj - xi) * (lat - yi) / (yj - yi) + xi):
                inside = not inside

            j = i

        return inside

    def _fire_trigger(self, device_id, fence_id, event_type, lat, lng, timestamp):
        """触发围栏事件"""
        fence = self.db.get_geofence(fence_id)

        # 1. 记录事件
        self.db.insert("geofence_events", {
            "event_id": str(uuid4()),
            "device_id": device_id,
            "fence_id": fence_id,
            "event_type": event_type,
            "lat": lat, "lng": lng,
            "fence_name": fence["name"],
            "triggered_at": timestamp
        })

        # 2. 发送通知
        device = self.db.get_device(device_id)
        if device:
            if event_type == "enter":
                self.notification.send(device.get("user_id"),
                    f"您已进入 {fence['name']} 区域")
            elif event_type == "exit":
                self.notification.send(device.get("user_id"),
                    f"您已离开 {fence['name']} 区域")

        # 3. 发布事件（供其他系统消费）
        self.event_bus.publish("geofence.triggered", {
            "device_id": device_id, "fence_id": fence_id,
            "event_type": event_type, "fence_name": fence["name"],
            "lat": lat, "lng": lng, "timestamp": timestamp.isoformat()
        })

    def _get_device_fence_state(self, device_id):
        """获取设备围栏状态"""
        state = self.redis.get(f"device_fence_state:{device_id}")
        return json.loads(state) if state else {"inside": {}}

    def _update_device_fence_state(self, device_id, lat, lng, triggered, timestamp):
        """更新设备围栏状态"""
        state = self._get_device_fence_state(device_id)

        for trigger in triggered:
            fence_id = trigger["fence_id"]
            if trigger["event"] == "enter":
                state["inside"][fence_id] = {
                    "entered_at": timestamp.isoformat(),
                    "entry_lat": lat, "entry_lng": lng
                }
            elif trigger["event"] == "exit":
                state["inside"].pop(fence_id, None)
            elif trigger["event"] == "dwell":
                if fence_id in state["inside"]:
                    state["inside"][fence_id]["dwell_triggered"] = True

        state["last_lat"] = lat
        state["last_lng"] = lng
        state["last_check"] = timestamp.isoformat()

        self.redis.setex(f"device_fence_state:{device_id}", 86400,
            json.dumps(state))
```

## 异常场景补充

### 场景：围栏边界抖动

```
触发：设备在围栏边界附近 → GPS 精度波动 → 反复进出围栏 → 触发风暴
检测：
  1. 短时间内同一围栏反复进出 → 边界抖动
  2. 同一设备 1 分钟内触发 > 3 次 enter/exit → 抖动
处理：
  1. 增加边界缓冲区（围栏半径 +50m 才算进入，-50m 才算离开）
  2. 进出去抖（进入后 30 秒内不再触发离开）
  3. 触发频率限制
预防：缓冲区 + 去抖 + 频率限制
```

### 场景：大量设备同时触发围栏

```
触发：演唱会结束 → 1 万人同时离开场馆围栏 → 大量通知同时发送 → 通知系统过载
检测：
  1. 围栏触发事件突增 → 批量触发
  2. 通知发送队列堆积 → 过载
处理：
  1. 围栏触发批量处理（合并通知）
  2. 通知发送限速（每秒最多 100 条）
  3. 非紧急通知延迟发送
预防：批量处理 + 限速 + 延迟发送
```

## 地理围栏实时追踪与批量检测完整实现

```python
import math
import time
from datetime import datetime, timedelta
from collections import defaultdict
from typing import List, Dict, Tuple, Optional, Set

class GeoTrackingService:
    """地理围栏实时追踪与批量检测服务，提供设备轨迹追踪、围栏检测、热力图生成和异常路线检测"""

    EARTH_RADIUS_KM = 6371.0
    STOP_VELOCITY_THRESHOLD = 0.5  # m/s
    STOP_DURATION_THRESHOLD = 180  # 3 minutes in seconds
    DEVIATION_THRESHOLD = 0.3  # 30% deviation
    KALMAN_PROCESS_NOISE = 0.01
    KALMAN_MEASUREMENT_NOISE = 0.001

    def __init__(self):
        self.device_tracks: Dict[str, List[Dict]] = {}
        self.geofence_index: Dict[str, List[Dict]] = {}
        self.geofence_events: List[Dict] = []
        self.device_states: Dict[str, Dict] = {}
        self.position_history: List[Dict] = []
        self.geohash_prefix_map: Dict[str, List[str]] = defaultdict(list)

    def _haversine_distance(self, lat1: float, lon1: float, lat2: float, lon2: float) -> float:
        """计算两点之间的Haversine距离，返回米"""
        lat1_rad = math.radians(lat1)
        lat2_rad = math.radians(lat2)
        delta_lat = math.radians(lat2 - lat1)
        delta_lon = math.radians(lon2 - lon1)
        a = (math.sin(delta_lat / 2) ** 2 +
             math.cos(lat1_rad) * math.cos(lat2_rad) * math.sin(delta_lon / 2) ** 2)
        c = 2 * math.atan2(math.sqrt(a), math.sqrt(1 - a))
        return self.EARTH_RADIUS_KM * c * 1000

    def _encode_geohash(self, lat: float, lon: float, precision: int = 8) -> str:
        """将经纬度编码为geohash字符串"""
        base32_chars = "0123456789bcdefghjkmnpqrstuvwxyz"
        lat_range = [-90.0, 90.0]
        lon_range = [-180.0, 180.0]
        geohash = ""
        bits = 0
        bit_count = 0
        is_lon = True
        while len(geohash) < precision:
            mid = (lon_range[0] + lon_range[1]) / 2 if is_lon else (lat_range[0] + lat_range[1]) / 2
            if (lon if is_lon else lat) >= mid:
                if is_lon:
                    lon_range[0] = mid
                else:
                    lat_range[0] = mid
                bits = bits * 2 + 1
            else:
                if is_lon:
                    lon_range[1] = mid
                else:
                    lat_range[1] = mid
                bits = bits * 2
            bit_count += 1
            is_lon = not is_lon
            if bit_count == 5:
                geohash += base32_chars[bits]
                bits = 0
                bit_count = 0
        return geohash

    def _query_spatial_index(self, lat: float, lon: float, radius_km: float) -> List[Dict]:
        """使用geohash前缀模拟R-tree空间索引查询"""
        precision = max(1, min(8, int(radius_km / 0.5)))
        geohash_prefix = self._encode_geohash(lat, lon, precision)[:max(1, precision - 1)]
        nearby_geofences = []
        for prefix_len in range(max(1, precision - 2), precision + 1):
            prefix = self._encode_geohash(lat, lon, prefix_len)[:prefix_len]
            if prefix in self.geohash_prefix_map:
                for fence_id in self.geohash_prefix_map[prefix]:
                    fence = self.geofence_index[fence_id]
                    center_lat = fence.get("center_lat", 0)
                    center_lon = fence.get("center_lon", 0)
                    dist = self._haversine_distance(lat, lon, center_lat, center_lon)
                    if dist <= radius_km * 1000:
                        nearby_geofences.append(fence)
        return nearby_geofences

    def _point_in_circle(self, lat: float, lon: float, circle: Dict) -> bool:
        """判断点是否在圆形围栏内"""
        center_lat = circle["center_lat"]
        center_lon = circle["center_lon"]
        radius = circle["radius_m"]
        distance = self._haversine_distance(lat, lon, center_lat, center_lon)
        return distance <= radius

    def _point_in_polygon(self, lat: float, lon: float, polygon: List[Tuple[float, float]]) -> bool:
        """使用射线法判断点是否在多边形内"""
        n = len(polygon)
        if n < 3:
            return False
        inside = False
        j = n - 1
        for i in range(n):
            yi, xi = polygon[i]
            yj, xj = polygon[j]
            if ((yi > lat) != (yj > lat)) and (lon < (xj - xi) * (lat - yi) / (yj - yi) + xi):
                inside = not inside
            j = i
        return inside

    def _kalman_predict(self, state: Dict) -> Dict:
        """Kalman滤波预测步骤"""
        dt = state.get("dt", 1.0)
        predicted_lat = state["lat"] + state["velocity_lat"] * dt
        predicted_lon = state["lon"] + state["velocity_lon"] * dt
        predicted_cov = state["covariance"] + self.KALMAN_PROCESS_NOISE * dt
        return {
            "lat": predicted_lat,
            "lon": predicted_lon,
            "velocity_lat": state["velocity_lat"],
            "velocity_lon": state["velocity_lon"],
            "covariance": predicted_cov
        }

    def _kalman_update(self, predicted: Dict, measurement: Dict) -> Dict:
        """Kalman滤波更新步骤，使用GPS测量值和精度权重"""
        accuracy = measurement.get("accuracy_m", 10.0)
        measurement_noise = self.KALMAN_MEASUREMENT_NOISE * (accuracy / 10.0)
        kalman_gain = predicted["covariance"] / (predicted["covariance"] + measurement_noise)
        updated_lat = predicted["lat"] + kalman_gain * (measurement["lat"] - predicted["lat"])
        updated_lon = predicted["lon"] + kalman_gain * (measurement["lon"] - predicted["lon"])
        updated_cov = (1 - kalman_gain) * predicted["covariance"]
        dt = measurement.get("timestamp", 0) - predicted.get("prev_timestamp", measurement.get("timestamp", 0))
        if dt > 0:
            velocity_lat = (measurement["lat"] - predicted.get("prev_lat", measurement["lat"])) / dt
            velocity_lon = (measurement["lon"] - predicted.get("prev_lon", measurement["lon"])) / dt
        else:
            velocity_lat = predicted["velocity_lat"]
            velocity_lon = predicted["velocity_lon"]
        return {
            "lat": updated_lat,
            "lon": updated_lon,
            "velocity_lat": velocity_lat,
            "velocity_lon": velocity_lon,
            "covariance": updated_cov,
            "prev_lat": measurement["lat"],
            "prev_lon": measurement["lon"],
            "prev_timestamp": measurement.get("timestamp", 0)
        }

    def _detect_stop_points(self, positions: List[Dict]) -> List[Dict]:
        """检测停留点：速度<0.5m/s持续>3分钟"""
        stop_points = []
        if len(positions) < 2:
            return stop_points
        slow_start_idx = None
        for i in range(len(positions)):
            pos = positions[i]
            velocity = pos.get("velocity_mps", self._estimate_velocity(positions, i))
            if velocity < self.STOP_VELOCITY_THRESHOLD:
                if slow_start_idx is None:
                    slow_start_idx = i
                duration = pos["timestamp"] - positions[slow_start_idx]["timestamp"]
                if duration >= self.STOP_DURATION_THRESHOLD:
                    avg_lat = sum(p["lat"] for p in positions[slow_start_idx:i + 1]) / (i - slow_start_idx + 1)
                    avg_lon = sum(p["lon"] for p in positions[slow_start_idx:i + 1]) / (i - slow_start_idx + 1)
                    stop_points.append({
                        "lat": avg_lat,
                        "lon": avg_lon,
                        "start_time": positions[slow_start_idx]["timestamp"],
                        "end_time": pos["timestamp"],
                        "duration_seconds": duration,
                        "type": "stop_point"
                    })
            else:
                slow_start_idx = None
        return stop_points

    def _estimate_velocity(self, positions: List[Dict], idx: int) -> float:
        """根据相邻位置估算速度"""
        if idx == 0:
            if len(positions) < 2:
                return 0.0
            next_pos = positions[1]
            dt = next_pos["timestamp"] - positions[0]["timestamp"]
            if dt <= 0:
                return 0.0
            dist = self._haversine_distance(positions[0]["lat"], positions[0]["lon"],
                                            next_pos["lat"], next_pos["lon"])
            return dist / dt
        prev_pos = positions[idx - 1]
        curr_pos = positions[idx]
        dt = curr_pos["timestamp"] - prev_pos["timestamp"]
        if dt <= 0:
            return 0.0
        dist = self._haversine_distance(prev_pos["lat"], prev_pos["lon"],
                                        curr_pos["lat"], curr_pos["lon"])
        return dist / dt

    def _split_into_trips(self, positions: List[Dict], stop_points: List[Dict]) -> List[List[Dict]]:
        """根据停留点将轨迹分割为行程段"""
        if not stop_points:
            return [positions] if positions else []
        trips = []
        current_trip = []
        stop_ranges = [(sp["start_time"], sp["end_time"]) for sp in stop_points]
        stop_idx = 0
        for pos in positions:
            is_in_stop = False
            while stop_idx < len(stop_ranges) and pos["timestamp"] > stop_ranges[stop_idx][1]:
                stop_idx += 1
            if stop_idx < len(stop_ranges):
                if stop_ranges[stop_idx][0] <= pos["timestamp"] <= stop_ranges[stop_idx][1]:
                    is_in_stop = True
            if is_in_stop:
                if current_trip:
                    trips.append(current_trip)
                    current_trip = []
            else:
                current_trip.append(pos)
        if current_trip:
            trips.append(current_trip)
        return trips

    def track_device_movement(self, device_id: str, positions: List[Dict]) -> Dict:
        """处理批量位置更新，应用Kalman滤波平滑，检测停留点，分割行程段，并存储处理后的轨迹"""
        if not positions:
            return {"device_id": device_id, "status": "no_data", "trip_count": 0}
        sorted_positions = sorted(positions, key=lambda p: p["timestamp"])
        smoothed_positions = []
        state = self.device_states.get(device_id, {
            "lat": sorted_positions[0]["lat"],
            "lon": sorted_positions[0]["lon"],
            "velocity_lat": 0.0,
            "velocity_lon": 0.0,
            "covariance": 1.0,
            "prev_lat": sorted_positions[0]["lat"],
            "prev_lon": sorted_positions[0]["lon"],
            "prev_timestamp": sorted_positions[0].get("timestamp", 0)
        })
        for pos in sorted_positions:
            dt = pos["timestamp"] - state.get("prev_timestamp", pos["timestamp"])
            state["dt"] = max(dt, 0.1)
            predicted = self._kalman_predict(state)
            updated = self._kalman_update(predicted, pos)
            smoothed_pos = {
                "lat": updated["lat"],
                "lon": updated["lon"],
                "timestamp": pos["timestamp"],
                "accuracy_m": pos.get("accuracy_m", 10.0),
                "velocity_mps": self._estimate_velocity(sorted_positions, sorted_positions.index(pos))
            }
            smoothed_positions.append(smoothed_pos)
            state = updated
        self.device_states[device_id] = state
        for sp in smoothed_positions:
            self.position_history.append({
                "device_id": device_id,
                "lat": sp["lat"],
                "lon": sp["lon"],
                "timestamp": sp["timestamp"]
            })
        stop_points = self._detect_stop_points(smoothed_positions)
        trips = self._split_into_trips(smoothed_positions, stop_points)
        total_distance = 0.0
        for i in range(1, len(smoothed_positions)):
            total_distance += self._haversine_distance(
                smoothed_positions[i - 1]["lat"], smoothed_positions[i - 1]["lon"],
                smoothed_positions[i]["lat"], smoothed_positions[i]["lon"]
            )
        processed_track = {
            "device_id": device_id,
            "positions": smoothed_positions,
            "stop_points": stop_points,
            "trips": trips,
            "total_distance_m": total_distance,
            "total_duration_s": smoothed_positions[-1]["timestamp"] - smoothed_positions[0]["timestamp"],
            "processed_at": time.time()
        }
        if device_id not in self.device_tracks:
            self.device_tracks[device_id] = []
        self.device_tracks[device_id].append(processed_track)
        return {
            "device_id": device_id,
            "status": "processed",
            "position_count": len(smoothed_positions),
            "stop_point_count": len(stop_points),
            "trip_count": len(trips),
            "total_distance_m": total_distance,
            "total_duration_s": processed_track["total_duration_s"]
        }

    def batch_check_geofences(self, positions: List[Dict]) -> List[Dict]:
        """批量检测位置是否触发围栏，使用空间索引和射线法"""
        if not positions:
            return []
        all_events = []
        for pos in positions:
            lat = pos["lat"]
            lon = pos["lon"]
            timestamp = pos.get("timestamp", time.time())
            device_id = pos.get("device_id", "unknown")
            search_radius_km = 1.0
            nearby_fences = self._query_spatial_index(lat, lon, search_radius_km)
            triggered_fences = []
            for fence in nearby_fences:
                is_inside = False
                if fence["shape"] == "circle":
                    is_inside = self._point_in_circle(lat, lon, fence)
                elif fence["shape"] == "polygon":
                    is_inside = self._point_in_polygon(lat, lon, fence["vertices"])
                if is_inside:
                    triggered_fences.append(fence)
            for fence in triggered_fences:
                prev_inside = self._was_device_inside(device_id, fence["id"])
                event_type = self._classify_geofence_event(
                    device_id, fence["id"], is_inside=True, prev_inside=prev_inside, timestamp=timestamp
                )
                dwell_duration = self._calculate_dwell_duration(
                    device_id, fence["id"], timestamp, is_inside=True
                )
                event = {
                    "device_id": device_id,
                    "geofence_id": fence["id"],
                    "geofence_name": fence.get("name", ""),
                    "event_type": event_type,
                    "lat": lat,
                    "lon": lon,
                    "timestamp": timestamp,
                    "dwell_duration_s": dwell_duration,
                    "fence_shape": fence["shape"],
                    "fence_metadata": fence.get("metadata", {})
                }
                all_events.append(event)
                self.geofence_events.append(event)
            checked_fence_ids = {f["id"] for f in nearby_fences}
            for fence_id in checked_fence_ids - {f["id"] for f in triggered_fences}:
                prev_inside = self._was_device_inside(device_id, fence_id)
                if prev_inside:
                    fence_data = self.geofence_index.get(fence_id, {})
                    dwell_duration = self._calculate_dwell_duration(
                        device_id, fence_id, timestamp, is_inside=False
                    )
                    event = {
                        "device_id": device_id,
                        "geofence_id": fence_id,
                        "geofence_name": fence_data.get("name", ""),
                        "event_type": "exit",
                        "lat": lat,
                        "lon": lon,
                        "timestamp": timestamp,
                        "dwell_duration_s": dwell_duration,
                        "fence_shape": fence_data.get("shape", "unknown"),
                        "fence_metadata": fence_data.get("metadata", {})
                    }
                    all_events.append(event)
                    self.geofence_events.append(event)
        return all_events

    def _was_device_inside(self, device_id: str, fence_id: str) -> bool:
        """检查设备之前是否在围栏内"""
        device_events = [e for e in self.geofence_events
                         if e["device_id"] == device_id and e["geofence_id"] == fence_id]
        if not device_events:
            return False
        latest = max(device_events, key=lambda e: e["timestamp"])
        return latest["event_type"] in ("enter", "dwell")

    def _classify_geofence_event(self, device_id: str, fence_id: str,
                                  is_inside: bool, prev_inside: bool, timestamp: float) -> str:
        """分类围栏事件为enter/exit/dwell"""
        if is_inside and not prev_inside:
            return "enter"
        if is_inside and prev_inside:
            return "dwell"
        if not is_inside and prev_inside:
            return "exit"
        return "none"

    def _calculate_dwell_duration(self, device_id: str, fence_id: str,
                                   timestamp: float, is_inside: bool) -> float:
        """计算在围栏内的停留时长"""
        device_events = [e for e in self.geofence_events
                         if e["device_id"] == device_id and e["geofence_id"] == fence_id]
        enter_events = [e for e in device_events if e["event_type"] == "enter"]
        if not enter_events:
            return 0.0 if is_inside else 0.0
        latest_enter = max(enter_events, key=lambda e: e["timestamp"])
        if is_inside:
            return timestamp - latest_enter["timestamp"]
        return timestamp - latest_enter["timestamp"]

    def calculate_heatmap(self, area_bounds: Dict, time_range: Tuple[float, float],
                          resolution: float = 0.01) -> Dict:
        """计算热力图：网格划分、设备计数、高斯核平滑、归一化"""
        min_lat = area_bounds["min_lat"]
        max_lat = area_bounds["max_lat"]
        min_lon = area_bounds["min_lon"]
        max_lon = area_bounds["max_lon"]
        start_time, end_time = time_range
        lat_steps = max(1, int((max_lat - min_lat) / resolution))
        lon_steps = max(1, int((max_lon - min_lon) / resolution))
        grid = [[0.0 for _ in range(lon_steps)] for _ in range(lat_steps)]
        time_bucket_duration = (end_time - start_time) / 24
        time_buckets = [
            (start_time + i * time_bucket_duration, start_time + (i + 1) * time_bucket_duration)
            for i in range(24)
        ]
        grid_per_bucket = [
            [[set() for _ in range(lon_steps)] for _ in range(lat_steps)]
            for _ in range(24)
        ]
        filtered_positions = [
            p for p in self.position_history
            if start_time <= p["timestamp"] <= end_time
            and min_lat <= p["lat"] <= max_lat
            and min_lon <= p["lon"] <= max_lon
        ]
        for pos in filtered_positions:
            lat_idx = min(lat_steps - 1, max(0, int((pos["lat"] - min_lat) / resolution)))
            lon_idx = min(lon_steps - 1, max(0, int((pos["lon"] - min_lon) / resolution)))
            bucket_idx = min(23, max(0, int((pos["timestamp"] - start_time) / time_bucket_duration)))
            grid_per_bucket[bucket_idx][lat_idx][lon_idx].add(pos["device_id"])
        for bucket_idx in range(24):
            for lat_idx in range(lat_steps):
                for lon_idx in range(lon_steps):
                    count = len(grid_per_bucket[bucket_idx][lat_idx][lon_idx])
                    grid[lat_idx][lon_idx] += count
        smoothed_grid = self._apply_gaussian_smoothing(grid, lat_steps, lon_steps, sigma=1.0)
        max_val = 0.0
        for row in smoothed_grid:
            for val in row:
                if val > max_val:
                    max_val = val
        normalized_grid = []
        if max_val > 0:
            for row in smoothed_grid:
                normalized_row = [val / max_val for val in row]
                normalized_grid.append(normalized_row)
        else:
            normalized_grid = smoothed_grid
        return {
            "grid": normalized_grid,
            "lat_steps": lat_steps,
            "lon_steps": lon_steps,
            "resolution": resolution,
            "time_range": time_range,
            "time_buckets": 24,
            "max_density": max_val,
            "total_positions": len(filtered_positions)
        }

    def _apply_gaussian_smoothing(self, grid: List[List[float]], rows: int, cols: int,
                                   sigma: float) -> List[List[float]]:
        """应用高斯核平滑"""
        kernel_size = int(math.ceil(sigma * 3)) * 2 + 1
        kernel_radius = kernel_size // 2
        kernel = []
        kernel_sum = 0.0
        for i in range(kernel_size):
            kernel_row = []
            for j in range(kernel_size):
                x = i - kernel_radius
                y = j - kernel_radius
                val = math.exp(-(x * x + y * y) / (2 * sigma * sigma))
                kernel_row.append(val)
                kernel_sum += val
            kernel.append(kernel_row)
        for i in range(kernel_size):
            for j in range(kernel_size):
                kernel[i][j] /= kernel_sum
        smoothed = [[0.0 for _ in range(cols)] for _ in range(rows)]
        for i in range(rows):
            for j in range(cols):
                acc = 0.0
                weight_sum = 0.0
                for ki in range(kernel_size):
                    for kj in range(kernel_size):
                        ni = i + ki - kernel_radius
                        nj = j + kj - kernel_radius
                        if 0 <= ni < rows and 0 <= nj < cols:
                            acc += grid[ni][nj] * kernel[ki][kj]
                            weight_sum += kernel[ki][kj]
                if weight_sum > 0:
                    smoothed[i][j] = acc / weight_sum
                else:
                    smoothed[i][j] = grid[i][j]
        return smoothed

    def detect_anomalous_route(self, device_id: str, expected_waypoints: List[Dict]) -> Dict:
        """检测异常路线：计算Hausdorff距离，识别偏移段，标记偏离路线"""
        if device_id not in self.device_tracks or not self.device_tracks[device_id]:
            return {"device_id": device_id, "status": "no_track_data", "anomaly_score": 0.0}
        latest_track = self.device_tracks[device_id][-1]
        actual_points = latest_track["positions"]
        if not actual_points or not expected_waypoints:
            return {"device_id": device_id, "status": "insufficient_data", "anomaly_score": 0.0}
        actual_coords = [(p["lat"], p["lon"]) for p in actual_points]
        expected_coords = [(wp["lat"], wp["lon"]) for wp in expected_waypoints]
        hausdorff_dist = self._hausdorff_distance(actual_coords, expected_coords)
        total_route_length = self._route_length(expected_coords)
        deviation_ratio = hausdorff_dist / total_route_length if total_route_length > 0 else 0.0
        detour_segments = self._identify_detour_segments(actual_coords, expected_coords)
        anomaly_score = self._calculate_anomaly_score(deviation_ratio, len(detour_segments),
                                                       latest_track.get("total_distance_m", 0),
                                                       total_route_length)
        is_anomalous = deviation_ratio > self.DEVIATION_THRESHOLD or anomaly_score > 0.7
        result = {
            "device_id": device_id,
            "status": "anomalous" if is_anomalous else "normal",
            "hausdorff_distance_m": hausdorff_dist,
            "deviation_ratio": deviation_ratio,
            "detour_segments": detour_segments,
            "anomaly_score": anomaly_score,
            "is_anomalous": is_anomalous,
            "actual_distance_m": latest_track.get("total_distance_m", 0),
            "expected_distance_m": total_route_length,
            "detour_count": len(detour_segments),
            "threshold_used": self.DEVIATION_THRESHOLD
        }
        return result

    def _hausdorff_distance(self, set_a: List[Tuple], set_b: List[Tuple]) -> float:
        """计算两个点集之间的Hausdorff距离"""
        if not set_a or not set_b:
            return float("inf")
        max_min_dist_a_to_b = 0.0
        for a in set_a:
            min_dist = float("inf")
            for b in set_b:
                dist = self._haversine_distance(a[0], a[1], b[0], b[1])
                if dist < min_dist:
                    min_dist = dist
            if min_dist > max_min_dist_a_to_b:
                max_min_dist_a_to_b = min_dist
        max_min_dist_b_to_a = 0.0
        for b in set_b:
            min_dist = float("inf")
            for a in set_a:
                dist = self._haversine_distance(b[0], b[1], a[0], a[1])
                if dist < min_dist:
                    min_dist = dist
            if min_dist > max_min_dist_b_to_a:
                max_min_dist_b_to_a = min_dist
        return max(max_min_dist_a_to_b, max_min_dist_b_to_a)

    def _route_length(self, coords: List[Tuple]) -> float:
        """计算路线总长度"""
        total = 0.0
        for i in range(1, len(coords)):
            total += self._haversine_distance(coords[i - 1][0], coords[i - 1][1],
                                               coords[i][0], coords[i][1])
        return total

    def _identify_detour_segments(self, actual: List[Tuple],
                                   expected: List[Tuple]) -> List[Dict]:
        """识别偏移超过30%的绕行段"""
        if len(actual) < 2:
            return []
        detour_segments = []
        segment_start = None
        expected_total = self._route_length(expected)
        if expected_total == 0:
            return []
        for i, point in enumerate(actual):
            min_dist = float("inf")
            nearest_expected_idx = 0
            for j, exp_point in enumerate(expected):
                dist = self._haversine_distance(point[0], point[1], exp_point[0], exp_point[1])
                if dist < min_dist:
                    min_dist = dist
                    nearest_expected_idx = j
            local_deviation = min_dist / expected_total if expected_total > 0 else 0.0
            is_detour = local_deviation > self.DEVIATION_THRESHOLD or min_dist > 500
            if is_detour:
                if segment_start is None:
                    segment_start = i
            else:
                if segment_start is not None:
                    segment_points = actual[segment_start:i]
                    if segment_points:
                        avg_dev = sum(
                            self._haversine_distance(p[0], p[1], ep[0], ep[1])
                            for p_idx, p in enumerate(segment_points)
                            for ep in [self._find_nearest(p, expected)]
                        ) / len(segment_points)
                        detour_segments.append({
                            "start_idx": segment_start,
                            "end_idx": i - 1,
                            "start_lat": segment_points[0][0],
                            "start_lon": segment_points[0][1],
                            "end_lat": segment_points[-1][0],
                            "end_lon": segment_points[-1][1],
                            "point_count": len(segment_points),
                            "avg_deviation_m": avg_dev,
                            "max_deviation_m": max(
                                self._haversine_distance(p[0], p[1], ep[0], ep[1])
                                for p in segment_points
                                for ep in [self._find_nearest(p, expected)]
                            )
                        })
                    segment_start = None
        if segment_start is not None:
            segment_points = actual[segment_start:]
            if segment_points:
                detour_segments.append({
                    "start_idx": segment_start,
                    "end_idx": len(actual) - 1,
                    "start_lat": segment_points[0][0],
                    "start_lon": segment_points[0][1],
                    "end_lat": segment_points[-1][0],
                    "end_lon": segment_points[-1][1],
                    "point_count": len(segment_points),
                    "avg_deviation_m": sum(
                        self._haversine_distance(p[0], p[1], ep[0], ep[1])
                        for p in segment_points
                        for ep in [self._find_nearest(p, expected)]
                    ) / len(segment_points),
                    "max_deviation_m": max(
                        self._haversine_distance(p[0], p[1], ep[0], ep[1])
                        for p in segment_points
                        for ep in [self._find_nearest(p, expected)]
                    )
                })
        return detour_segments

    def _find_nearest(self, point: Tuple, coords: List[Tuple]) -> Tuple:
        """找到最近的坐标点"""
        min_dist = float("inf")
        nearest = coords[0] if coords else (0, 0)
        for c in coords:
            dist = self._haversine_distance(point[0], point[1], c[0], c[1])
            if dist < min_dist:
                min_dist = dist
                nearest = c
        return nearest

    def _calculate_anomaly_score(self, deviation_ratio: float, detour_count: int,
                                  actual_distance: float, expected_distance: float) -> float:
        """计算异常分数"""
        deviation_component = min(1.0, deviation_ratio * 2)
        detour_component = min(1.0, detour_count * 0.2)
        distance_ratio = actual_distance / expected_distance if expected_distance > 0 else 1.0
        distance_component = min(1.0, max(0.0, (distance_ratio - 1.0) * 2))
        score = deviation_component * 0.4 + detour_component * 0.3 + distance_component * 0.3
        return round(min(1.0, score), 4)

    def add_geofence(self, fence_id: str, shape: str, **kwargs) -> None:
        """添加地理围栏到空间索引"""
        fence = {"id": fence_id, "shape": shape, **kwargs}
        self.geofence_index[fence_id] = fence
        if shape == "circle":
            center_lat = kwargs.get("center_lat", 0)
            center_lon = kwargs.get("center_lon", 0)
        elif shape == "polygon" and kwargs.get("vertices"):
            lats = [v[0] for v in kwargs["vertices"]]
            lons = [v[1] for v in kwargs["vertices"]]
            center_lat = sum(lats) / len(lats)
            center_lon = sum(lons) / len(lons)
        else:
            return
        for prefix_len in range(1, 7):
            geohash_prefix = self._encode_geohash(center_lat, center_lon, prefix_len)[:prefix_len]
            self.geohash_prefix_map[geohash_prefix].append(fence_id)
```

## 异常场景补充

### 场景：GPS 信号突然丢失
```
trigger: 设备进入隧道、地下停车场或城市峡谷区域，GPS信号被遮挡，连续3个以上位置更新失败或精度骤降至500米以上
detection: 1) 监控位置更新频率，当更新间隔超过预设阈值（如30秒）时触发告警；2) 检测位置精度字段accuracy_m，当持续超过500米时判定为信号丢失；3) 对比Kalman滤波预测位置与实际GPS位置的偏差，当偏差突增且协方差持续增大时确认信号异常
handling: 1) 自动切换到Kalman滤波纯预测模式，使用最后的有效速度和方向进行航位推算；2) 尝试利用WiFi指纹定位和基站三角定位作为备用定位源；3) 在追踪界面上标注信号丢失区间，使用虚线显示推算轨迹；4) 当信号恢复后，用实际GPS位置重新校准Kalman滤波器状态；5) 对信号丢失期间的停留点检测暂停，避免误判；6) 若信号丢失超过10分钟，通知监控人员
prevention: 1) 在设备端实现多源融合定位（GPS+WiFi+基站+IMU惯性导航）；2) 预加载地图中的已知信号盲区标记；3) 当接近已知盲区时提前增大Kalman滤波的过程噪声以适应更大不确定性；4) 在进入盲区前缓存关键路径点；5) 部署低功耗蓝牙信标在关键盲区补充定位
```

### 场景：批量围栏检测内存溢出
```
trigger: 一次性批量检测超过10万个位置点，或单个查询匹配的围栏数量超过1万个，导致内存使用超过可用限制
detection: 1) 在批量检测入口处检查输入位置列表长度，超过阈值时提前告警；2) 实时监控进程内存使用率，当使用率超过85%时触发熔断；3) 监控单次查询返回的围栏匹配数量，超过5000时标记为高风险；4) 检测geohash空间索引的桶倾斜，某些热门区域桶过大
handling: 1) 立即对批量请求进行分片处理，将位置列表按geohash区域分组，每组不超过5000个点顺序处理；2) 对超大围栏集使用流式处理，每次只加载当前geohash桶中的围栏子集；3) 启用位置采样策略，对同一设备的高频位置进行降采样（保留关键转折点）；4) 释放已完成处理的中间结果内存；5) 返回部分结果并标记为截断响应，告知客户端需要分页获取
prevention: 1) 在API层设置批量请求大小上限（默认5000个位置/次）；2) 优化空间索引结构，使用更细粒度的geohash前缀减少桶内围栏数量；3) 对围栏数据实现LRU缓存淘汰策略；4) 使用生成器而非列表存储中间结果，减少内存峰值；5) 定期清理过期的围栏事件记录，避免历史数据无限增长；6) 部署内存监控和自动熔断机制
```

### 场景：设备轨迹数据被篡改
```
trigger: 恶意客户端伪造GPS坐标绕过地理围栏，或设备被劫持发送虚假位置数据，导致围栏事件误触发
detection: 1) 物理一致性检测：计算相邻位置点之间的速度，若超过物理可能值（如汽车>300km/h、步行>20km/h）则标记异常；2) 信号特征检测：检查GPS报文中的卫星数量、信噪比、HDOP值是否在合理范围；3) 位置跳跃检测：同一设备在极短时间内出现在地理上不可能的两个位置；4) 统计异常检测：设备在某区域的停留模式与历史基线严重偏离；5) 设备指纹校验：验证位置上报的设备ID与注册信息是否匹配
handling: 1) 立即隔离可疑设备的轨迹数据，标记为"待验证"状态，不触发围栏告警；2) 启动多源交叉验证，对比WiFi定位、基站定位结果与GPS位置的一致性；3) 通知安全团队进行人工审核；4) 对已触发的围栏事件进行回溯审计，撤销基于篡改数据产生的告警；5) 记录完整的篡改证据链（原始数据、异常指标、验证结果）用于法律追责
prevention: 1) 实现端到端的位置数据加密签名，服务端验证签名完整性；2) 在设备端部署可信执行环境（TEE），保护定位模块输出不被篡改；3) 引入设备行为基线模型，实时检测偏离基线的异常行为；4) 对关键围栏事件增加二次验证机制（如要求视频或照片确认）；5) 建立设备信任评分体系，对低信任设备的位置数据降权处理；6) 定期轮换通信加密密钥
```
