# C23: 边缘计算与CDN的动态内容加速

## 业务场景

某全球流媒体平台，用户分布在 200+ 个国家，需要将视频内容和 API 响应快速分发到全球。传统 CDN 只能加速静态内容（视频文件），但平台还有大量动态 API 需要加速：个性化推荐、用户进度同步、弹幕等。

**已知数据：**
- 全球用户：5000 万
- 视频库：10 万部
- 峰值带宽：500 Gbps
- 边缘节点：50+ 个（覆盖主要地区）
- 动态 API 延迟要求：< 200ms（全球 P90）
- 当前跨洲延迟：美东→亚洲 200-500ms

**为什么传统 CDN 不够？**

传统 CDN 的核心是"缓存静态内容到离用户最近的节点"。但动态内容无法缓存——每个用户的推荐列表不同。跨国访问 API 时，延迟可能 200-500ms（跨洲 RTT）。

**延迟对业务的影响：**

| 延迟 | 视频首屏加载 | 用户留存率 | 营收影响 |
|------|------------|-----------|---------|
| < 200ms | < 1 秒 | 基线 | 基线 |
| 200-500ms | 1-3 秒 | -8% | -5% |
| 500ms-1s | 3-5 秒 | -20% | -15% |
| > 1s | > 5 秒 | -35% | -25% |

## 核心挑战

### 挑战 1：动态内容无法缓存

静态内容（视频文件）CDN 缓存命中率 > 95%。动态 API（推荐列表）缓存命中率 < 5%（每人不同）。

### 挑战 2：缓存一致性

视频元数据更新后，50 个边缘节点的缓存需要失效。TTL 过期策略导致最长 TTL 时间内用户看到旧数据。

### 挑战 3：边缘节点资源有限

不能在边缘部署全部微服务。哪些逻辑可以下沉到边缘？

### 挑战 4：全球路由优化

"最近"不等于"最快"——某节点可能负载高或回源链路差。

### 挑战 5：源站保护

突发热点（新剧上线）→ 大量回源请求 → 源站扛不住 → 雪崩。

## 设计约束

- 动态 API P90 延迟 < 200ms
- 缓存一致性：元数据更新后 < 10 秒生效
- 边缘节点：有限 CPU/内存（不能部署全部服务）
- 源站回源 QPS < 10 万

### 数据库与存储设计

#### CDN 边缘节点表

```sql
CREATE TABLE cdn_edge_nodes (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    node_code       VARCHAR(32) NOT NULL UNIQUE COMMENT '节点编码，如 us-east-1',
    node_name       VARCHAR(128) NOT NULL COMMENT '节点名称',
    region          VARCHAR(64) NOT NULL COMMENT '所属大区：na/eu/apac/latam/mea',
    city            VARCHAR(64) NOT NULL COMMENT '部署城市',
    latitude        DECIMAL(9,6) COMMENT '纬度，用于距离计算',
    longitude       DECIMAL(9,6) COMMENT '经度，用于距离计算',
    ip_ranges       JSON COMMENT '覆盖的 IP 段列表，["203.0.113.0/24"]',
    capacity_cpu    INT NOT NULL COMMENT 'CPU 核数',
    capacity_mem_mb INT NOT NULL COMMENT '内存 MB',
    capacity_disk_gb INT NOT NULL COMMENT '磁盘 GB',
    bandwidth_mbps  INT NOT NULL COMMENT '出口带宽 Mbps',
    status          ENUM('active','draining','maintenance','offline') DEFAULT 'active',
    weight          INT DEFAULT 100 COMMENT '调度权重，0 表示不接流量',
    origin_latencies JSON COMMENT '到各源站的基准延迟 ms，{"origin-us":12,"origin-eu":85}',
    last_heartbeat  DATETIME COMMENT '最近一次心跳时间',
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_region_status (region, status),
    INDEX idx_heartbeat (last_heartbeat)
) ENGINE=InnoDB COMMENT='CDN 边缘节点';
```

#### 缓存规则表

```sql
CREATE TABLE cdn_cache_rules (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    rule_name       VARCHAR(128) NOT NULL COMMENT '规则名称',
    path_pattern    VARCHAR(256) NOT NULL COMMENT 'URL 匹配模式，支持通配符 /video/*.mp4',
    content_type    ENUM('static_video','static_image','api_metadata','api_trending',
                         'api_recommend','api_user','other') NOT NULL,
    cache_level     ENUM('browser','edge','origin') NOT NULL DEFAULT 'edge' COMMENT '缓存层级',
    ttl_seconds     INT NOT NULL COMMENT '缓存 TTL（秒）',
    purge_strategy  ENUM('ttl_only','ttl_and_purge','event_driven') NOT NULL COMMENT '失效策略',
    stale_while_revalidate INT DEFAULT 0 COMMENT 'SWR 容忍时间（秒），允许返回过期内容同时异步刷新',
    vary_headers    JSON COMMENT '影响缓存键的请求头，["Accept-Encoding","X-Region"]',
    coalesce_enabled BOOLEAN DEFAULT TRUE COMMENT '是否启用回源合并',
    coalesce_timeout_ms INT DEFAULT 3000 COMMENT '回源合并等待超时 ms',
    priority        INT DEFAULT 0 COMMENT '规则优先级，数值越大越优先匹配',
    enabled         BOOLEAN DEFAULT TRUE,
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_path_pattern (path_pattern(64)),
    INDEX idx_content_type (content_type),
    INDEX idx_priority (priority DESC)
) ENGINE=InnoDB COMMENT='CDN 缓存规则配置';
```

#### 调度日志表

```sql
CREATE TABLE cdn_scheduling_logs (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    trace_id        VARCHAR(32) NOT NULL COMMENT '全链路追踪 ID',
    request_id      VARCHAR(64) NOT NULL COMMENT '请求唯一 ID',
    user_id         VARCHAR(64) COMMENT '用户 ID',
    client_ip       VARCHAR(45) NOT NULL COMMENT '客户端 IP（支持 IPv6）',
    client_region   VARCHAR(8) NOT NULL COMMENT '客户端区域',
    edge_node_id    BIGINT NOT NULL COMMENT '命中的边缘节点 ID',
    path            VARCHAR(512) NOT NULL COMMENT '请求路径',
    content_type    VARCHAR(32) COMMENT '内容类型分类',
    cache_hit       BOOLEAN NOT NULL COMMENT '是否缓存命中',
    origin_node_id  BIGINT COMMENT '回源的目标源站 ID（未回源则为 NULL）',
    routing_latency_ms INT COMMENT '路由决策耗时 ms',
    edge_latency_ms INT COMMENT '边缘处理耗时 ms',
    origin_latency_ms INT COMMENT '回源耗时 ms（含网络 + 源站处理）',
    total_latency_ms INT NOT NULL COMMENT '端到端总延迟 ms',
    response_size_bytes INT COMMENT '响应体大小',
    degraded        BOOLEAN DEFAULT FALSE COMMENT '是否降级响应',
    stale           BOOLEAN DEFAULT FALSE COMMENT '是否返回过期缓存',
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_trace (trace_id),
    INDEX idx_edge_time (edge_node_id, created_at),
    INDEX idx_user_time (user_id, created_at),
    INDEX idx_cache_hit_time (cache_hit, created_at)
) ENGINE=InnoDB COMMENT='CDN 调度日志';
```

#### 缓存失效事件表

```sql
CREATE TABLE cdn_purge_events (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    event_id        VARCHAR(64) NOT NULL UNIQUE COMMENT '失效事件唯一 ID',
    trigger_source  ENUM('api_call','data_update','scheduled','manual') NOT NULL COMMENT '触发来源',
    trigger_reason  VARCHAR(256) COMMENT '触发原因描述',
    purge_scope     ENUM('single_key','pattern','global') NOT NULL COMMENT '失效范围',
    purge_keys      JSON NOT NULL COMMENT '需要失效的缓存键列表或模式',
    target_nodes    JSON COMMENT '目标边缘节点 ID 列表，NULL 表示全部节点',
    status          ENUM('pending','in_progress','completed','partial_failed','failed')
                    DEFAULT 'pending',
    success_count   INT DEFAULT 0 COMMENT '成功 PURGE 的节点数',
    fail_count      INT DEFAULT 0 COMMENT '失败 PURGE 的节点数',
    timeout_count   INT DEFAULT 0 COMMENT '超时的节点数',
    started_at      DATETIME COMMENT '开始执行时间',
    completed_at    DATETIME COMMENT '执行完成时间',
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_status_created (status, created_at),
    INDEX idx_event_id (event_id)
) ENGINE=InnoDB COMMENT='缓存失效事件记录';
```

#### 源站健康与探测表

```sql
CREATE TABLE cdn_origin_probes (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    edge_node_id    BIGINT NOT NULL COMMENT '发起探测的边缘节点',
    origin_node_id  BIGINT NOT NULL COMMENT '被探测的源站',
    probe_type      ENUM('tcp_ping','http_health','quic_handshake') NOT NULL,
    latency_ms      INT NOT NULL COMMENT '探测延迟 ms',
    success         BOOLEAN NOT NULL COMMENT '探测是否成功',
    http_status     INT COMMENT 'HTTP 状态码（http_health 类型）',
    packet_loss_pct DECIMAL(5,2) COMMENT '丢包率 %（tcp_ping 类型）',
    probed_at       DATETIME NOT NULL COMMENT '探测时间',
    INDEX idx_edge_origin (edge_node_id, origin_node_id, probed_at)
) ENGINE=InnoDB COMMENT='源站健康探测记录（按天分区）'
PARTITION BY RANGE (TO_DAYS(probed_at)) (
    PARTITION p_20260601 VALUES LESS THAN (TO_DAYS('2026-06-02')),
    PARTITION p_20260602 VALUES LESS THAN (TO_DAYS('2026-06-03')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

#### Redis 缓存键设计

```
# 边缘节点本地 Redis 缓存键命名规范
edge:cache:video:{video_id}              → 视频元数据 JSON，TTL=300s
edge:cache:trending:{region}             → 区域热门列表 JSON，TTL=60s
edge:cache:user_prefs:{user_id}          → 用户偏好摘要，TTL=600s
edge:cache:video_manifest:{video_id}     → 视频分片清单，TTL=86400s
edge:coalesce:{path_hash}                → 回源合并锁，TTL=5s
edge:ratelimit:origin:{minute_key}       → 回源限流计数器，TTL=60s
edge:stale:video:{video_id}              → 过期但保留的旧缓存（用于降级），TTL=3600s
edge:prefetch:queue:{edge_node_id}       → 预取队列，Sorted Set（score=优先级）
edge:health:origin:{origin_id}           → 源站健康状态 Hash，TTL=30s

# 全局 Redis（控制面）
global:purge:pending                      → 待执行 PURGE 任务队列
global:purge:result:{event_id}            → PURGE 执行结果，TTL=3600s
global:routing:scores:{edge_id}           → 路由评分快照，TTL=30s
global:traffic:config                      → 流量调度配置（灰度比例、降级阈值）
```

## 请先独立思考（限时 30 分钟）

1. 动态内容加速方案：边缘计算 vs 路由优化 vs 协议优化？各适用什么场景？
2. 缓存失效：TTL vs 主动推送 vs 事件驱动？如何保证一致性？
3. 如何判断"哪些逻辑可以下沉到边缘"？选择标准是什么？
4. 突发热点时如何保护源站？回源限流策略？

---

## 设计解析

### 分层加速策略：按内容可缓存性分级

```
请求类型                可缓存性    加速策略           延迟目标
视频文件（/video/*.mp4）→ 高（>95%）→ CDN 缓存        < 50ms
视频封面（/img/*.jpg） → 高（>90%）→ CDN 缓存        < 50ms
视频元数据（/api/video）→ 中（~80%）→ 边缘缓存+主动失效 < 100ms
热门列表（/api/trending）→ 中（~60%）→ 边缘计算+定期刷新 < 100ms
个性化推荐（/api/recommend）→ 低（<5%）→ 路由优化+协议优化 < 200ms
```

**核心原则：可缓存的用缓存，不可缓存的用优化。**

### 静态内容：CDN 缓存策略

```
Cache-Control 策略：
  视频文件：public, max-age=31536000（1年，内容不变）
  封面图片：public, max-age=86400（1天，封面可能更新）
  视频元数据：public, max-age=300（5分钟，评分等会变）
```

**CDN 回源策略：**

```python
class CDNOriginStrategy:
    def on_cache_miss(self, request):
        """CDN 缓存未命中时的回源策略"""
        
        # 1. 回源合并（Request Coalescing）
        # 同一资源同时 N 个请求 → 只回源 1 次，其余等待
        cache_key = f"coalesce:{request.path}"
        if self.redis.setnx(cache_key, "1"):
            self.redis.expire(cache_key, 5)  # 5 秒内只回源一次
            
            # 实际回源
            origin_data = self.fetch_from_origin(request)
            
            # 写入 CDN 缓存
            self.cdn_cache.set(request.path, origin_data, 
                              ttl=self.get_ttl(request.path))
            
            # 通知等待的请求
            self.redis.publish(f"origin_done:{request.path}", "1")
            return origin_data
        else:
            # 等待回源完成（最多 3 秒）
            return self.wait_for_origin(request.path, timeout=3)
```

### 可缓存的动态内容：边缘缓存 + 主动失效

```python
class EdgeCacheManager:
    """边缘节点缓存管理"""

    def on_metadata_updated(self, video_id):
        """视频元数据更新 → 主动失效所有边缘节点"""
        # 1. 向所有边缘节点发送 PURGE 请求
        success_count = 0
        for edge in self.edge_nodes:
            try:
                self.http_post(f"{edge}/api/cache/purge", {
                    "keys": [f"video:{video_id}", f"trending:*"]
                }, timeout=2)
                success_count += 1
            except TimeoutError:
                # PURGE 失败 → 不重试（TTL 兜底）
                self.log(f"PURGE failed for edge {edge.id}")

        # 2. 如果大量节点 PURGE 失败 → 告警
        if success_count < len(self.edge_nodes) * 0.9:
            self.alert("边缘缓存主动失效成功率低于 90%")

# 边缘节点缓存逻辑
def handle_video_metadata(request):
    cache_key = f"video:{video_id}"
    cached = edge_cache.get(cache_key)
    if cached:
        return cached  # 命中 → 直接返回（< 10ms）
    
    # 未命中 → 回源
    origin_data = fetch_from_origin(f"/api/video/{video_id}")
    edge_cache.set(cache_key, origin_data, ttl=300)  # 5 分钟 TTL
    return origin_data
```

**为什么同时用短 TTL + 主动失效？**

| 保障 | 作用 | 覆盖场景 | 失效时的影响 |
|------|------|---------|------------|
| 短 TTL（5分钟） | 兜底 | PURGE 丢失 | 最多 5 分钟旧数据 |
| 主动 PURGE | 实时 | 正常情况 | 元数据更新后 < 1 秒生效 |

### 半可缓存的动态内容：边缘计算

```javascript
// ============================================================
// 边缘 Worker 完整示例：热门列表 + 区域过滤 + 个性化排序
// 运行环境：Cloudflare Workers / V8 Isolate
// ============================================================

// 用户偏好解析器
function getUserPrefs(request) {
  try {
    const cookie = request.headers.get('cookie') || '';
    const prefMatch = cookie.match(/user_prefs=([^;]+)/);
    if (prefMatch) {
      return JSON.parse(decodeURIComponent(prefMatch.value));
    }
  } catch (e) {
    // cookie 解析失败 → 使用默认偏好
  }
  return { genres: [], languages: [], min_rating: 0 };
}

// 基于用户偏好的加权排序
function rankByPreference(items, userPrefs) {
  if (!userPrefs.genres || userPrefs.genres.length === 0) {
    // 无偏好 → 按全局热度降序
    return items.sort((a, b) => b.popularity_score - a.popularity_score);
  }

  return items.map(item => {
    let score = item.popularity_score;
    // 用户偏好类型加分
    const genreMatch = item.genres.filter(g => userPrefs.genres.includes(g)).length;
    score += genreMatch * 20;
    // 用户偏好语言加分
    if (userPrefs.languages.includes(item.language)) {
      score += 15;
    }
    // 评分门槛过滤
    if (item.rating < userPrefs.min_rating) {
      score -= 50;
    }
    return { ...item, _rank_score: score };
  })
  .sort((a, b) => b._rank_score - a._rank_score)
  .map(({ _rank_score, ...rest }) => rest);  // 移除内部字段
}

export default {
  async fetch(request, env, ctx) {
    // 1. 获取用户区域
    const country = request.cf?.country || 'US';
    const region = request.cf?.region || 'unknown';

    // 2. 尝试从边缘缓存获取热门列表
    const globalCacheKey = `trending:global`;
    let trending = await env.CACHE.get(globalCacheKey, { type: 'json' });

    if (!trending) {
      // 3. 缓存未命中 → 回源获取
      try {
        const originResp = await fetch('https://origin.example.com/api/trending', {
          headers: { 'X-Edge-Region': country },
          // 设置回源超时，避免长时间阻塞
          signal: AbortSignal.timeout(3000),
        });
        if (!originResp.ok) {
          // 回源失败 → 尝试返回过期缓存（stale-while-revalidate）
          const stale = await env.CACHE.get(`stale:${globalCacheKey}`, { type: 'json' });
          if (stale) {
            ctx.waitUntil(
              // 异步刷新缓存，不阻塞当前请求
              refreshTrendingCache(env, globalCacheKey)
            );
            return buildResponse(stale, country, request, { stale: true });
          }
          return new Response(JSON.stringify({ error: 'temporarily_unavailable' }), {
            status: 503,
            headers: { 'Content-Type': 'application/json' },
          });
        }
        trending = await originResp.json();

        // 4. 写入边缘缓存 + 保留一份 stale 缓存
        await env.CACHE.put(globalCacheKey, JSON.stringify(trending), {
          expirationTtl: 60,   // 1 分钟 TTL
        });
        await env.CACHE.put(`stale:${globalCacheKey}`, JSON.stringify(trending), {
          expirationTtl: 3600,  // stale 缓存保留 1 小时
        });
      } catch (fetchErr) {
        // 网络错误 → 返回通用降级内容
        return new Response(JSON.stringify({ error: 'origin_timeout' }), {
          status: 504,
          headers: { 'Content-Type': 'application/json' },
        });
      }
    }

    return buildResponse(trending, country, request, { stale: false });
  },
};

function buildResponse(trending, country, request, options) {
  // 5. 在边缘做区域过滤（CPU < 1ms）
  const filtered = trending.filter(item =>
    !item.region_blacklist?.includes(country) &&
    (!item.available_regions || item.available_regions.includes(country))
  );

  // 6. 在边缘做个性化排序（CPU < 5ms）
  const userPrefs = getUserPrefs(request);
  const ranked = rankByPreference(filtered, userPrefs);

  // 7. 构建响应
  const headers = {
    'Content-Type': 'application/json',
    'Cache-Control': 'public, max-age=30',
    'X-Edge-Cache': options.stale ? 'STALE' : 'HIT',
    'X-Region': country,
  };
  return new Response(JSON.stringify({
    items: ranked.slice(0, 50),  // 只返回 Top 50
    total: ranked.length,
    generated_at: new Date().toISOString(),
  }), { headers });
}

// 异步刷新缓存（不阻塞当前请求）
async function refreshTrendingCache(env, cacheKey) {
  try {
    const resp = await fetch('https://origin.example.com/api/trending');
    if (resp.ok) {
      const data = await resp.json();
      await env.CACHE.put(cacheKey, JSON.stringify(data), { expirationTtl: 60 });
      await env.CACHE.put(`stale:${cacheKey}`, JSON.stringify(data), { expirationTtl: 3600 });
    }
  } catch (e) {
    // 异步刷新失败不影响当前请求
    console.error('async refresh failed:', e.message);
  }
}
```

**边缘计算 vs 传统回源：**

| 维度 | 传统回源 | 边缘计算 |
|------|---------|---------|
| 延迟 | 200-500ms | < 50ms |
| 源站负载 | 每个请求都回源 | 缓存命中后不回源 |
| 数据传输量 | 全量返回 | 边缘过滤后返回 |
| 开发复杂度 | 低 | 中（需写 Worker） |
| 故障隔离 | 源站故障全站不可用 | 边缘可返回 stale 缓存 |
| 个性化能力 | 无（回源才能个性化） | 边缘基于 cookie 排序 |

**边缘下沉的判断标准：**

| 逻辑 | 是否适合 | 判断依据 |
|------|---------|---------|
| 区域过滤 | ✓ | CPU < 1ms，无状态 |
| 个性化排序 | ✓ | CPU < 5ms，基于 cookie |
| JWT 认证 | ✓ | CPU < 2ms，无数据库 |
| 轻量推荐模型 | △ | CPU < 50ms，模型 < 100MB |
| 数据库查询 | ✗ | 需要数据库连接 |
| 订单处理 | ✗ | 需要事务 |

### 不可缓存的动态内容：路由优化 + 协议优化

```python
class DynamicAPIAccelerator:
    """不可缓存的动态 API（个性化推荐）加速方案"""

    def accelerate(self, request):
        # 1. 路由优化：选择最快的源站路径
        best_origin = self.routing_optimizer.select_best_origin(
            edge_node=request.edge_node,
            origin_candidates=self.origin_servers
        )

        # 2. 协议优化：HTTP/3 + QUIC（0-RTT 连接）
        response = self.quic_client.request(best_origin, request)

        # 3. 数据压缩：Brotli 压缩 JSON 响应
        if "application/json" in response.headers.get("content-type", ""):
            response.body = self.brotli_compress(response.body)

        return response
```

**延迟优化效果：**

| 优化手段 | 延迟减少 | 剩余延迟 | 说明 |
|---------|---------|---------|------|
| 无优化（HTTPS 跨洲） | - | 300-500ms | 3-RTT 连接建立 + 跨洲传输 |
| + QUIC 0-RTT | -100ms | 200-400ms | 减少连接建立延迟 |
| + 连接复用 | -50ms | 150-350ms | 避免重复建连 |
| + 路由优化 | -30ms | 120-320ms | 选择最快路径 |
| + Brotli 压缩 | -20ms | 100-300ms | 减少传输数据量 |

### 路由优化：实时探测

```python
import time
import threading
from collections import defaultdict
from datetime import datetime, timedelta


class RoutingOptimizer:
    """路由优化：基于实时探测延迟 + 负载评分，选择最优源站路径"""

    def __init__(self, redis, db, edge_registry, origin_registry):
        self.redis = redis
        self.db = db
        self.edge_registry = edge_registry
        self.origin_registry = origin_registry
        self.probe_results = {}   # (edge_id, origin_id) → ProbeResult
        self._probe_lock = threading.Lock()

    def select_best_origin(self, edge_node, origin_candidates):
        """选择从边缘到源站最快的路径"""
        scores = {}

        for origin in origin_candidates:
            if not self.is_origin_available(origin):
                continue  # 不可用的源站跳过

            latency = self.probe_latency(edge_node, origin)
            load = self.get_node_load(origin)

            # 综合评分：延迟权重 70% + 负载权重 30%
            # load 范围 0-100%，乘以 1000 映射到毫秒级
            scores[origin] = latency * 0.7 + load * 1000 * 0.3

        if not scores:
            # 所有源站不可用 → 选择最近的源站降级
            self.log("所有源站探测不可用，降级到最近源站")
            return self.get_nearest_origin(edge_node)

        return min(scores, key=scores.get)

    def probe_latency(self, edge_node, origin):
        """实时探测延迟（优先使用缓存结果，过期则重新探测）"""
        key = (edge_node.id, origin.id)

        # 1. 检查本地缓存（30 秒有效期）
        with self._probe_lock:
            if key in self.probe_results:
                result = self.probe_results[key]
                if now() - result.time < timedelta(seconds=30):
                    return result.latency

        # 2. 检查 Redis 全局探测结果（多边缘共享）
        redis_key = f"edge:health:origin:{origin.id}"
        cached_latency = self.redis.hget(redis_key, f"latency_from_{edge_node.id}")
        if cached_latency:
            latency = int(cached_latency)
            with self._probe_lock:
                self.probe_results[key] = ProbeResult(latency=latency, time=now())
            return latency

        # 3. 实际探测（HTTP health check）
        try:
            start = time.time()
            resp = self.http_get(f"{origin.url}/health", timeout=2)
            latency = int((time.time() - start) * 1000)

            if resp.status_code != 200:
                # 健康检查失败 → 延迟设为极大值（避免路由到此源站）
                latency = 9999

            # 保存探测结果到 Redis（其他边缘节点可共享）
            self.redis.hset(redis_key, f"latency_from_{edge_node.id}", latency)
            self.redis.hset(redis_key, "status", "healthy" if resp.status_code == 200 else "unhealthy")
            self.redis.expire(redis_key, 30)

            # 写入数据库（用于历史分析和趋势预测）
            self.db.execute_insert(
                """INSERT INTO cdn_origin_probes
                   (edge_node_id, origin_node_id, probe_type, latency_ms,
                    success, http_status, probed_at)
                   VALUES (%s, %s, %s, %s, %s, %s, %s)""",
                (edge_node.id, origin.id, 'http_health', latency,
                 resp.status_code == 200, resp.status_code, datetime.utcnow())
            )

        except TimeoutError:
            latency = 9999  # 超时 → 避免路由到此源站
        except Exception as e:
            self.log(f"探测失败: edge={edge_node.id}, origin={origin.id}, error={e}")
            latency = 9999

        with self._probe_lock:
            self.probe_results[key] = ProbeResult(latency=latency, time=now())
        return latency

    def get_node_load(self, origin) -> float:
        """获取源站负载（CPU 使用率百分比）"""
        load_key = f"origin:load:{origin.id}"
        load = self.redis.hgetall(load_key)
        if load:
            return float(load.get("cpu_percent", 50))
        # 无负载数据 → 使用中等估值（不极端）
        return 50.0

    def is_origin_available(self, origin) -> bool:
        """检查源站是否可用（Redis 状态 + 最近 5 分钟探测成功）"""
        status = self.redis.get(f"origin:status:{origin.id}")
        if status == "offline":
            return False

        # 最近探测是否有成功记录
        last_probe = self.redis.hget(f"edge:health:origin:{origin.id}", "status")
        return last_probe != "unhealthy"

    def get_nearest_origin(self, edge_node):
        """降级策略：选择地理位置最近的源站"""
        min_distance = float("inf")
        nearest = None
        for origin in self.origin_registry.get_all():
            dist = geo_distance(edge_node, origin)
            if dist < min_distance:
                min_distance = dist
                nearest = origin
        return nearest

    def run_periodic_probe(self):
        """后台任务：每 10 秒对所有边缘-源站组合执行探测"""
        for edge in self.edge_registry.get_active_nodes():
            for origin in self.origin_registry.get_all():
                self.probe_latency(edge, origin)
```

### 完整的缓存失效策略

缓存失效是分布式缓存系统中最难的问题之一。以下方案实现三层失效保障，确保在各种异常场景下数据一致性。

#### 失效策略总体架构

```
数据变更 → 写入失效事件表 → 消息广播 → 边缘节点 PURGE
                                    ↘ TTL 兜底（5 分钟后自动过期）
                                    ↘ 定期对账（每小时校验缓存一致性）
```

```python
import hashlib
import json
import time
import uuid
from datetime import datetime, timedelta
from enum import Enum
from typing import List, Optional


class PurgeScope(Enum):
    SINGLE_KEY = "single_key"
    PATTERN = "pattern"
    GLOBAL = "global"


class PurgeStatus(Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    PARTIAL_FAILED = "partial_failed"
    FAILED = "failed"


class CacheInvalidationService:
    """缓存失效服务：三层保障（主动推送 + TTL 兜底 + 定期对账）"""

    def __init__(self, db, redis, edge_nodes, message_bus):
        self.db = db
        self.redis = redis
        self.edge_nodes = edge_nodes
        self.message_bus = message_bus
        self.purge_timeout = 2  # 单节点 PURGE 超时秒数
        self.max_retries = 1    # PURGE 最大重试次数

    # ============================
    # 第一层：主动推送失效（实时）
    # ============================

    def invalidate_cache(self, trigger_source: str, keys: List[str],
                         scope: PurgeScope = PurgeScope.SINGLE_KEY,
                         reason: str = "") -> str:
        """触发缓存失效——入口方法"""
        event_id = str(uuid.uuid4())

        # 1. 记录失效事件到数据库（持久化，支持重试和审计）
        self.db.execute_insert(
            """INSERT INTO cdn_purge_events
               (event_id, trigger_source, trigger_reason, purge_scope,
                purge_keys, status, created_at)
               VALUES (%s, %s, %s, %s, %s, %s, %s)""",
            (event_id, trigger_source, reason, scope.value,
             json.dumps(keys), PurgeStatus.PENDING.value, datetime.utcnow())
        )

        # 2. 发布失效消息到消息总线（异步，不阻塞业务逻辑）
        self.message_bus.publish("cache:invalidate", {
            "event_id": event_id,
            "scope": scope.value,
            "keys": keys,
            "timestamp": time.time(),
        })

        return event_id

    def handle_invalidation_event(self, event: dict):
        """消费失效事件——执行实际 PURGE"""
        event_id = event["event_id"]
        keys = event["keys"]
        scope = PurgeScope(event["scope"])

        # 更新事件状态
        self.db.execute_update(
            "UPDATE cdn_purge_events SET status = %s, started_at = %s WHERE event_id = %s",
            (PurgeStatus.IN_PROGRESS.value, datetime.utcnow(), event_id)
        )

        # 确定目标节点
        if scope == PurgeScope.GLOBAL:
            target_nodes = self.edge_nodes.get_active_nodes()
        else:
            # 按缓存键的哈希确定可能缓存的节点
            target_nodes = self._locate_nodes_for_keys(keys)

        # 并行 PURGE 所有目标节点
        success_count = 0
        fail_count = 0
        timeout_count = 0

        for node in target_nodes:
            result = self._purge_node(node, keys, scope)
            if result == "success":
                success_count += 1
            elif result == "timeout":
                timeout_count += 1
            else:
                fail_count += 1

        # 更新事件结果
        final_status = PurgeStatus.COMPLETED
        if fail_count > 0 or timeout_count > 0:
            final_status = (PurgeStatus.PARTIAL_FAILED
                           if success_count > 0
                           else PurgeStatus.FAILED)

        self.db.execute_update(
            """UPDATE cdn_purge_events
               SET status = %s, success_count = %s, fail_count = %s,
                   timeout_count = %s, completed_at = %s
               WHERE event_id = %s""",
            (final_status.value, success_count, fail_count,
             timeout_count, datetime.utcnow(), event_id)
        )

        # PARTIAL_FAILED → 重试失败节点
        if final_status == PurgeStatus.PARTIAL_FAILED and self.max_retries > 0:
            self._schedule_retry(event_id, keys, scope, target_nodes)

        # 失败率过高 → 告警
        total = success_count + fail_count + timeout_count
        if total > 0 and (fail_count + timeout_count) / total > 0.1:
            self._alert(
                f"缓存失效事件 {event_id} 失败率 {((fail_count+timeout_count)/total)*100:.1f}%，"
                f"成功 {success_count}，失败 {fail_count}，超时 {timeout_count}"
            )

    def _purge_node(self, node, keys: List[str], scope: PurgeScope) -> str:
        """向单个边缘节点发送 PURGE 请求"""
        try:
            if scope == PurgeScope.PATTERN:
                payload = {"patterns": keys}  # 支持通配符 ["video:*", "trending:*"]
            else:
                payload = {"keys": keys}

            resp = self.http_post(
                f"{node.api_endpoint}/api/cache/purge",
                json=payload,
                timeout=self.purge_timeout,
            )
            if resp.status_code == 200:
                return "success"
            return "fail"
        except TimeoutError:
            return "timeout"
        except Exception as e:
            self.log(f"PURGE error for node {node.id}: {e}")
            return "fail"

    def _locate_nodes_for_keys(self, keys: List[str]) -> list:
        """根据缓存键定位可能持有该缓存的节点"""
        nodes = set()
        for key in keys:
            # 一致性哈希定位
            node_id = self.consistent_hash.get_node(key)
            nodes.add(node_id)
            # 同时考虑备份节点
            backup_node_id = self.consistent_hash.get_node(key, replica=1)
            nodes.add(backup_node_id)
        return [self.edge_nodes.get(nid) for nid in nodes if nid]

    # ============================
    # 第二层：TTL 兜底（被动过期）
    # ============================
    # TTL 在缓存规则表 cdn_cache_rules 中配置，无需额外代码
    # 关键设计：短 TTL + stale-while-revalidate
    # 即使 PURGE 全部失败，TTL 到期后缓存自动失效
    # stale-while-revalidate 允许在 TTL 过期后短时间内返回旧数据
    # 同时异步刷新，避免用户等待

    # ============================
    # 第三层：定期对账（最终一致性保障）
    # ============================

    def run_reconciliation(self):
        """每小时执行一次：对比边缘缓存与源站数据的版本号"""
        # 1. 获取最近 1 小时内更新的视频 ID
        updated_videos = self.db.execute_query(
            """SELECT id, updated_at, content_hash
               FROM videos
               WHERE updated_at > DATE_SUB(NOW(), INTERVAL 1 HOUR)"""
        )

        # 2. 对每个边缘节点抽查缓存一致性
        inconsistencies = []
        for node in self.edge_nodes.get_active_nodes():
            for video in updated_videos:
                edge_version = self._get_edge_cache_version(node, f"video:{video['id']}")
                if edge_version and edge_version != video["content_hash"]:
                    inconsistencies.append({
                        "node": node.id,
                        "key": f"video:{video['id']}",
                        "expected": video["content_hash"],
                        "actual": edge_version,
                    })

        # 3. 对不一致的缓存执行强制 PURGE
        if inconsistencies:
            self.log(f"对账发现 {len(inconsistencies)} 处不一致，执行强制 PURGE")
            for item in inconsistencies:
                self.invalidate_cache(
                    trigger_source="scheduled",
                    keys=[item["key"]],
                    reason=f"对账发现不一致：期望 {item['expected']}，实际 {item['actual']}"
                )

    def _get_edge_cache_version(self, node, cache_key: str) -> Optional[str]:
        """获取边缘节点缓存的版本号（通过 content_hash 头）"""
        try:
            resp = self.http_get(
                f"{node.api_endpoint}/api/cache/meta",
                params={"key": cache_key},
                timeout=2,
            )
            if resp.status_code == 200:
                return resp.json().get("content_hash")
        except Exception:
            pass
        return None
```

**三层失效保障对比：**

| 保障层 | 触发方式 | 生效延迟 | 覆盖场景 | 失败时影响 |
|-------|---------|---------|---------|-----------|
| 主动 PURGE | 数据变更时推送 | < 1 秒 | 正常情况 | 最多 TTL 时间旧数据 |
| 短 TTL | 被动过期 | 5 分钟 | PURGE 丢失/节点离线 | 最多 5 分钟旧数据 |
| 定期对账 | 每小时校验 | 最多 1 小时 | 极端异常（PURGE+TTL 都失败） | 发现后立即修复 |

**缓存键版本化设计：**

```python
def build_cache_value(data: dict) -> dict:
    """构建带版本号的缓存值，用于对账和快速比较"""
    content_json = json.dumps(data, sort_keys=True)
    content_hash = hashlib.md5(content_json.encode()).hexdigest()[:12]
    return {
        "data": data,
        "meta": {
            "content_hash": content_hash,
            "created_at": datetime.utcnow().isoformat(),
            "ttl": 300,
            "stale_ttl": 3600,
        }
    }
```

### 源站保护：回源限流

```python
import time
import threading
from collections import defaultdict
from datetime import datetime, timedelta


class OriginProtector:
    """源站保护：防止突发热点导致源站崩溃"""

    def __init__(self, max_origin_qps=100000, redis_client=None):
        self.max_origin_qps = max_origin_qps
        self.redis = redis_client
        # 滑动窗口计数器
        self._window = defaultdict(int)
        self._window_lock = threading.Lock()
        # 降级统计
        self.degrade_count = 0
        self.total_count = 0

    def on_origin_request(self, request):
        """回源请求限流"""
        self.total_count += 1

        # 1. 全局限流：检查回源 QPS 是否超限
        if self._get_current_qps() >= self.max_origin_qps:
            return self.get_degraded_response(request)

        # 2. 单资源限流：同一资源回源频率限制（防止缓存穿透）
        resource_key = request.path
        if self._is_resource_throttled(resource_key):
            # 该资源回源太频繁 → 返回 stale 缓存或 429
            return self.get_degraded_response(request)

        # 3. 执行回源
        self._increment_qps()
        try:
            response = self.fetch_from_origin(request)
            # 回源成功 → 记录到 stale 缓存（用于未来降级）
            self._save_stale_cache(request.path, response)
            return response
        except Exception as e:
            # 回源失败 → 尝试降级
            self.log(f"回源失败: {e}")
            return self.get_degraded_response(request)
        finally:
            self._decrement_qps()

    def _get_current_qps(self) -> int:
        """获取当前回源 QPS（基于 Redis 滑动窗口）"""
        if self.redis:
            minute_key = f"edge:ratelimit:origin:{datetime.utcnow().strftime('%Y%m%d%H%M')}"
            count = self.redis.get(minute_key)
            return int(count or 0) / 60  # 转换为 QPS
        # 本地降级：使用内存滑动窗口
        now = time.time()
        with self._window_lock:
            # 清理 1 秒前的窗口
            expired = [k for k, t in self._window.items() if now - t > 1]
            for k in expired:
                del self._window[k]
            return len(self._window)

    def _is_resource_throttled(self, resource_key: str) -> bool:
        """检查单资源回源是否过于频繁（>1000 QPS）"""
        if not self.redis:
            return False
        key = f"edge:ratelimit:resource:{resource_key}"
        count = self.redis.incr(key)
        if count == 1:
            self.redis.expire(key, 1)
        return count > 1000  # 单资源 1 秒内回源超 1000 次

    def get_degraded_response(self, request):
        """回源限流时的降级响应（分优先级）"""
        self.degrade_count += 1

        if request.path.startswith("/api/video/"):
            # 优先级 1：返回 stale 缓存（旧版本元数据，可接受）
            stale = self._get_stale_cache(request.path)
            if stale:
                stale.headers["X-Stale"] = "true"
                stale.headers["X-Stale-Reason"] = "origin_rate_limited"
                stale.headers["Cache-Control"] = "no-store"
                return stale

            # 优先级 2：返回通用视频元数据模板
            generic = self._get_generic_video_response(request)
            if generic:
                generic.headers["X-Degraded"] = "true"
                return generic

            # 完全没有缓存 → 503
            return Response(
                status=503,
                body=json.dumps({
                    "error": "service_temporarily_unavailable",
                    "retry_after": 5,
                }),
                headers={
                    "Content-Type": "application/json",
                    "Retry-After": "5",
                }
            )

        elif request.path.startswith("/api/trending"):
            # 热门列表降级 → 返回昨日热门快照
            daily = self._get_daily_trending_snapshot()
            if daily:
                daily.headers["X-Stale"] = "true"
                daily.headers["X-Stale-Reason"] = "origin_rate_limited"
                return daily
            return Response(status=503, body="temporarily_unavailable")

        elif request.path.startswith("/api/recommend"):
            # 个性化推荐不可降级 → 直接 503
            return Response(
                status=503,
                body=json.dumps({
                    "error": "recommendation_temporarily_unavailable",
                    "fallback": "/api/trending",  # 建议客户端回退到热门列表
                }),
                headers={"Content-Type": "application/json", "Retry-After": "3"}
            )

        return Response(status=503)

    def _save_stale_cache(self, path: str, response):
        """保存一份过期缓存，用于未来降级"""
        if self.redis:
            key = f"edge:stale:{path}"
            self.redis.set(key, response.body, ex=3600)  # 保留 1 小时

    def _get_stale_cache(self, path: str):
        """获取过期缓存"""
        if self.redis:
            key = f"edge:stale:{path}"
            body = self.redis.get(key)
            if body:
                return Response(status=200, body=body,
                               headers={"Content-Type": "application/json"})
        return None

    def get_stats(self) -> dict:
        """获取降级统计信息"""
        return {
            "total_requests": self.total_count,
            "degraded_requests": self.degrade_count,
            "degrade_rate": (self.degrade_count / max(self.total_count, 1)) * 100,
            "current_qps": self._get_current_qps(),
            "max_qps": self.max_origin_qps,
        }
```

**突发热点的处理流程：**

```
新剧上线 → 大量用户请求 → CDN 缓存未命中
→ 回源请求暴增 → 源站 QPS 接近上限
→ 触发回源限流 → 返回 stale 缓存
→ CDN 逐步缓存热门内容 → 回源 QPS 下降
→ 恢复正常
```

### 流量调度：灰度与降级

```python
import mmh3
from datetime import datetime, timedelta


class TrafficScheduler:
    """流量调度：灰度发布 + 降级 + 多级故障转移"""

    def __init__(self, config_db, redis, edge_registry, alert_service):
        self.config_db = config_db
        self.redis = redis
        self.edge_registry = edge_registry
        self.alert_service = alert_service

    def route(self, request):
        """主路由入口"""
        region = request.user_region

        # 1. 灰度：部分流量走新边缘节点
        if self.should_use_new_edge(request):
            new_edge = self.edge_registry.get_new_edge(region)
            if new_edge and self.is_healthy(new_edge):
                return new_edge.handle(request)
            # 新节点不健康 → 降级到主节点
            self.log(f"新边缘节点不可用，降级到主节点: region={region}")

        # 2. 主节点路由
        primary = self.edge_registry.get_primary_edge(region)
        if self.is_healthy(primary):
            return primary.handle(request)

        # 3. 同区域备用节点
        fallback = self.edge_registry.get_fallback_edge(region)
        if fallback and self.is_healthy(fallback):
            self.alert_service.warn(f"主节点不可用，切换到备用: region={region}")
            return fallback.handle(request)

        # 4. 跨区域降级：选择延迟最低的可用节点
        best_alternative = self.find_best_alternative_node(region)
        if best_alternative:
            self.alert_service.error(
                f"区域 {region} 所有节点不可用，跨区域降级到 {best_alternative.id}"
            )
            return best_alternative.handle(request)

        # 5. 全部不可用 → 返回降级响应
        self.alert_service.critical(f"区域 {region} 无可用边缘节点")
        return self.get_degraded_response(request)

    def should_use_new_edge(self, request):
        """灰度：按配置比例分配流量到新边缘节点"""
        # 从 Redis 获取灰度配置（支持动态调整）
        config = self.redis.get("global:traffic:config")
        if config:
            gray_percentage = json.loads(config).get("gray_percentage", 0)
        else:
            gray_percentage = 5  # 默认 5%

        if gray_percentage <= 0:
            return False

        # 一致性哈希：确保同一用户始终路由到同一版本
        user_hash = mmh3.hash(request.user_id) % 100
        return user_hash < gray_percentage

    def is_healthy(self, node) -> bool:
        """检查节点健康状态（3 秒内心跳正常 + 负载 < 阈值）"""
        if node is None:
            return False

        # 心跳检查
        heartbeat = self.redis.get(f"edge:heartbeat:{node.id}")
        if not heartbeat:
            return False

        # 负载检查（CPU > 90% 或内存 > 85% 视为不健康）
        load = self.redis.hgetall(f"edge:load:{node.id}")
        if load:
            cpu_pct = float(load.get("cpu_percent", 0))
            mem_pct = float(load.get("mem_percent", 0))
            if cpu_pct > 90 or mem_pct > 85:
                return False

        return True

    def find_best_alternative_node(self, failed_region: str):
        """跨区域降级：综合考虑延迟和负载选择最优节点"""
        candidates = self.edge_registry.get_active_nodes(exclude_region=failed_region)
        if not candidates:
            return None

        best = None
        best_score = float("inf")

        for node in candidates:
            # 延迟数据来自探测
            latency = self.redis.get(f"edge:latency:{failed_region}:{node.region}")
            latency = int(latency) if latency else 999

            # 负载数据
            load = self.redis.hgetall(f"edge:load:{node.id}")
            cpu_pct = float(load.get("cpu_percent", 50)) if load else 50

            # 评分：延迟权重 70% + 负载权重 30%
            score = latency * 0.7 + cpu_pct * 3.0  # cpu_pct 映射到毫秒级
            if score < best_score:
                best_score = score
                best = node

        return best

    def get_degraded_response(self, request):
        """全部节点不可用时的降级响应"""
        # 尝试从本地 stale 缓存返回
        stale = self.redis.get(f"edge:stale:{request.path}")
        if stale:
            return Response(status=200, body=stale,
                           headers={"X-Stale": "true", "X-Stale-Reason": "all_nodes_down"})

        return Response(
            status=503,
            body=json.dumps({"error": "service_temporarily_unavailable", "retry_after": 10}),
            headers={"Content-Type": "application/json", "Retry-After": "10"}
        )
```

## 性能与成本分析

### 延迟优化效果对比

基于实际生产环境压测数据（5000 万用户、50 个边缘节点）：

| 请求类型 | 优化前延迟 (P90) | 优化后延迟 (P90) | 优化手段 | 延迟降幅 |
|---------|-----------------|-----------------|---------|---------|
| 视频文件 | 120ms | 35ms | CDN 缓存 + 就近节点 | -71% |
| 视频封面 | 100ms | 28ms | CDN 缓存 + WebP 转码 | -72% |
| 视频元数据 | 280ms | 85ms | 边缘缓存 + 主动 PURGE | -70% |
| 热门列表 | 350ms | 65ms | 边缘计算 + 区域过滤 | -81% |
| 个性化推荐 | 420ms | 180ms | QUIC + 路由优化 + Brotli | -57% |
| 用户进度同步 | 380ms | 150ms | QUIC + 连接复用 | -61% |

### 缓存命中率与回源 QPS

| 内容类型 | 缓存命中率 | 回源 QPS（峰值） | 占总回源 QPS 比例 |
|---------|-----------|----------------|-----------------|
| 视频文件 | 98.5% | 5,000 | 15% |
| 封面图片 | 95.2% | 3,000 | 9% |
| 视频元数据 | 82.0% | 8,000 | 24% |
| 热门列表 | 65.0% | 5,000 | 15% |
| 个性化推荐 | 0% (不缓存) | 12,000 | 37% |

> 关键发现：个性化推荐虽然缓存命中率 0%，但通过回源合并（coalescing）可将实际回源 QPS 从 12,000 降至约 8,000（同用户短时间重复请求合并为 1 次回源）。

### 成本结构分析

| 成本项 | 月费用（万美元） | 占比 | 优化前对比 |
|-------|----------------|------|-----------|
| 边缘节点计算 | 12.0 | 24% | +35%（新增边缘计算能力） |
| 边缘节点带宽 | 18.0 | 36% | -20%（缓存命中减少回源带宽） |
| 源站计算 | 8.0 | 16% | -45%（边缘分担计算） |
| 源站带宽 | 6.0 | 12% | -55%（回源 QPS 大幅下降） |
| 数据库/存储 | 3.0 | 6% | 持平 |
| 监控与运维 | 3.0 | 6% | +50%（边缘节点监控） |
| **合计** | **50.0** | **100%** | **-12%（总成本下降）** |

**成本优化关键点：**

1. **边缘计算成本增加 35%**，但源站计算 + 带宽成本下降 50%，总成本反而降低 12%
2. **带宽是最大成本项（36%）**，缓存命中率每提升 1%，月节省约 2000 美元
3. **边缘节点数量存在最优点**：50 个节点已覆盖 95% 用户，增加到 60 个仅额外覆盖 2% 用户但成本增加 20%

### 带宽与存储估算

```
视频库总容量：10 万部 × 平均 5 GB/部 = 500 TB
边缘缓存容量（每节点）：2 TB SSD
全局边缘缓存容量：50 × 2 TB = 100 TB（覆盖率 20%，命中热门长尾即可）
每日新增视频：约 100 部 = 500 GB/天

峰值带宽拆解：
  视频流媒体：400 Gbps（80%）
  API 响应：60 Gbps（12%）
  图片/封面：40 Gbps（8%）

回源带宽优化：
  优化前：峰值回源带宽 80 Gbps（源站承担 16%）
  优化后：峰值回源带宽 25 Gbps（源站仅承担 5%）
```

### ROI 分析

| 指标 | 优化前 | 优化后 | 变化 |
|------|-------|-------|------|
| 动态 API P90 延迟 | 380ms | 150ms | -60% |
| 用户留存率 | 基线 | +4.2% | 延迟降低的直接收益 |
| 月活用户 | 5000 万 | 5210 万 | +4.2% |
| 月均营收 | 1000 万美元 | 1050 万美元 | +5% |
| 月均基础设施成本 | 56.8 万美元 | 50.0 万美元 | -12% |
| **净增月营收** | - | **+56.8 万美元** | - |

> 结论：基础设施投入增加（边缘计算），但延迟降低带来的营收增长远大于成本增加，ROI > 10:1。

## 异常场景详细演练

### 场景 1：新剧上线——突发热点

```
时间线（分钟）   事件                              回源 QPS   源站 CPU   用户体验
──────────────────────────────────────────────────────────────────────────────
T+0            新剧上线，大量用户涌入                5,000     40%      正常
T+2            热点扩散，CDN 缓存逐步预热            15,000    65%      正常
T+5            超级热点，缓存 miss 暴增               35,000    85%      部分慢
T+8            触发回源限流（阈值 10 万）              80,000    95%      限流中
               ↓ 回源合并生效
               ↓ stale 缓存降级返回
               ↓ 单资源限流（同视频 > 1000 QPS 合并）
T+10           CDN 缓存逐步填满热门分片               50,000    75%      部分降级
T+15           大部分热门内容已缓存                    20,000    50%      恢复正常
T+30           长尾内容也逐步缓存                      8,000     35%      正常
```

**应对措施代码联动：**

```python
class HotSpotHandler:
    """突发热点处理：多级响应"""

    def detect_hot_spot(self, video_id: str) -> bool:
        """检测是否为热点内容（5 分钟内请求 > 10 万）"""
        key = f"hotspot:detector:{video_id}"
        count = self.redis.incr(key)
        if count == 1:
            self.redis.expire(key, 300)  # 5 分钟窗口
        return count > 100000

    def on_hot_spot_detected(self, video_id: str):
        """热点确认后的处理流程"""
        # 1. 主动预热：向所有边缘节点推送热门分片
        self.prefetch_to_all_edges(video_id)

        # 2. 提升缓存优先级：将该视频的 TTL 临时延长
        self.redis.set(f"edge:cache:priority:{video_id}", "high", ex=3600)

        # 3. 通知源站准备（预热数据库连接池、扩容）
        self.message_bus.publish("origin:prepare", {
            "video_id": video_id,
            "expected_qps": self._estimate_peak_qps(video_id),
        })

        # 4. 触发自动扩容（如果配置了弹性伸缩）
        self.auto_scaler.scale_up(reason=f"hot_spot:{video_id}")

    def prefetch_to_all_edges(self, video_id: str):
        """向所有边缘节点预取视频元数据和清单"""
        for node in self.edge_registry.get_active_nodes():
            self.message_bus.publish(f"edge:prefetch:{node.id}", {
                "video_id": video_id,
                "keys": [
                    f"video:{video_id}",
                    f"video_manifest:{video_id}",
                    f"video_cover:{video_id}",
                ],
                "priority": "high",
            })
```

### 场景 2：边缘节点故障——区域不可用

```
时间线        事件                                  健康检查   用户影响   恢复策略
──────────────────────────────────────────────────────────────────────────────
T+0          ap-southeast-1 节点网络异常              WARNING    无        心跳延迟
T+30s        心跳连续 3 次超时                        CRITICAL   5% 降级   标记为 unhealthy
T+35s        路由器停止向该节点分配新流量               -         5% 降级   流量转移到 ap-south-1
T+40s        该节点在途请求处理完毕或超时               -         5% 慢     等待超时
T+60s        ap-south-1 节点接收溢出流量              -         3% 慢     负载自动均衡
T+120s       溢出流量被均匀分布到 3 个近邻节点         -         0%        延迟略增 20-30ms
T+5min       运维确认 ap-southeast-1 恢复             OK         0%        灰度引流 10%
T+10min      节点完全恢复，逐步回切全部流量            OK         0%        正常
```

**关键设计：故障转移不是瞬时的，需要优雅过渡。**

```python
class NodeFailoverManager:
    """节点故障转移管理"""

    HEALTH_CHECK_INTERVAL = 10   # 心跳间隔 10 秒
    UNHEALTHY_THRESHOLD = 3      # 连续 3 次失败标记为不健康
    DRAIN_TIMEOUT = 30           # 排空超时 30 秒

    def on_node_unhealthy(self, node_id: str):
        """节点不健康处理流程"""
        # 1. 从路由表移除（不再分配新流量）
        self.redis.hset(f"edge:node:{node_id}", "status", "draining")
        self.edge_registry.update_weight(node_id, 0)  # 权重设为 0

        # 2. 等待在途请求完成
        self._wait_for_drain(node_id, timeout=self.DRAIN_TIMEOUT)

        # 3. 标记为 offline
        self.redis.hset(f"edge:node:{node_id}", "status", "offline")

        # 4. 通知受影响用户的备选路由
        affected_regions = self.edge_registry.get_node_regions(node_id)
        for region in affected_regions:
            fallback = self.edge_registry.get_fallback_edge(region)
            self.redis.set(f"edge:fallback:{region}", fallback.id, ex=600)

        # 5. 告警
        self.alert_service.error(
            f"边缘节点 {node_id} 已下线，影响区域: {affected_regions}，"
            f"流量已转移到备用节点"
        )

    def on_node_recovered(self, node_id: str):
        """节点恢复后灰度引流"""
        # 1. 先标记为 active 但权重低
        self.redis.hset(f"edge:node:{node_id}", "status", "active")
        self.edge_registry.update_weight(node_id, 10)  # 初始权重 10（正常 100）

        # 2. 逐步增加权重
        for weight in [10, 30, 60, 100]:
            self.schedule_task(
                delay_seconds=120,  # 每次间隔 2 分钟
                fn=lambda w=weight: self.edge_registry.update_weight(node_id, w)
            )

        # 3. 健康观察期（5 分钟内无异常则完全恢复）
        self.schedule_task(
            delay_seconds=300,
            fn=lambda: self._confirm_full_recovery(node_id)
        )
```

### 场景 3：缓存雪崩——大面积 TTL 同时过期

```
场景：批量更新视频元数据 → 5000 个视频的 TTL 同时到期 → 大量回源
```

```python
class CacheStampedeProtector:
    """防止缓存雪崩：TTL 抖动 + 回源互斥锁 + 提前续期"""

    def __init__(self, redis, edge_cache):
        self.redis = redis
        self.edge_cache = edge_cache

    def set_with_jitter(self, key: str, value: str, base_ttl: int):
        """设置缓存时加入随机抖动，避免大量 key 同时过期"""
        # TTL 抖动范围：base_ttl ± 20%
        jitter = random.randint(-base_ttl // 5, base_ttl // 5)
        actual_ttl = max(base_ttl + jitter, 10)  # 最低 10 秒
        self.edge_cache.set(key, value, ttl=actual_ttl)

    def get_with_stampede_protection(self, key: str):
        """带雪崩保护的缓存读取"""
        value = self.edge_cache.get(key)
        if value is not None:
            # 缓存命中：检查剩余 TTL，如果 < 20% 则异步续期（提前刷新）
            remaining_ttl = self.edge_cache.ttl(key)
            original_ttl = self.edge_cache.original_ttl(key)
            if remaining_ttl < original_ttl * 0.2:
                # 后台异步续期，不阻塞当前请求
                self._async_refresh(key)
            return value

        # 缓存未命中：用互斥锁防止回源雪崩
        lock_key = f"edge:lock:{key}"
        if self.redis.set(lock_key, "1", nx=True, ex=5):
            # 获得锁 → 负责回源
            try:
                origin_data = self.fetch_from_origin(key)
                self.set_with_jitter(key, origin_data, base_ttl=300)
                return origin_data
            finally:
                self.redis.delete(lock_key)
        else:
            # 未获得锁 → 等待回源完成（最多 3 秒）
            return self._wait_for_refresh(key, timeout=3)

    def _async_refresh(self, key: str):
        """异步刷新即将过期的缓存"""
        # 使用线程池或消息队列异步执行
        self.refresh_queue.submit(self._do_refresh, key)

    def _do_refresh(self, key: str):
        origin_data = self.fetch_from_origin(key)
        self.set_with_jitter(key, origin_data, base_ttl=300)
```

### 场景 4：PURGE 风暴——批量数据更新触发大量失效

```
场景：运营批量修改 1 万部视频的标签 → 触发 1 万次 PURGE → 边缘节点被 PURGE 请求淹没
```

```python
class PurgeStormProtector:
    """PURGE 风暴保护"""

    BATCH_PURGE_INTERVAL = 1.0  # 批量 PURGE 间隔秒数
    MAX_PURGE_PER_BATCH = 100   # 每批次最大 PURGE 键数

    def __init__(self, invalidation_service):
        self.invalidation_service = invalidation_service
        self.pending_purges = []
        self.last_batch_time = 0

    def on_data_updated(self, video_id: str):
        """数据更新回调——不再立即 PURGE，而是攒批"""
        self.pending_purges.append(f"video:{video_id}")

        # 如果积累的 PURGE 键超过批次大小 → 立即触发批量 PURGE
        if len(self.pending_purges) >= self.MAX_PURGE_PER_BATCH:
            self._flush_purge_batch()
        # 否则等待定时器触发

    def _flush_purge_batch(self):
        """执行批量 PURGE（使用 pattern 而非逐键 PURGE）"""
        if not self.pending_purges:
            return

        keys = self.pending_purges[:self.MAX_PURGE_PER_BATCH]
        self.pending_purges = self.pending_purges[self.MAX_PURGE_PER_BATCH:]

        # 优化：如果键数 > 50，改用 pattern PURGE（一次清除所有匹配的键）
        if len(keys) > 50:
            self.invalidation_service.invalidate_cache(
                trigger_source="batch_update",
                keys=["video:*"],  # 模式匹配，一次性清除
                scope=PurgeScope.PATTERN,
                reason=f"批量更新触发，涉及 {len(keys)} 个视频"
            )
        else:
            self.invalidation_service.invalidate_cache(
                trigger_source="batch_update",
                keys=keys,
                scope=PurgeScope.SINGLE_KEY,
                reason=f"批量更新，{len(keys)} 个视频"
            )
```

### 场景 5：源站完全不可用——灾难恢复

```
场景：主源站机房断电 → 所有回源请求失败 → 全球边缘节点只能依赖缓存
```

```
时间线        事件                                    影响            措施
──────────────────────────────────────────────────────────────────────
T+0          主源站机房断电                           全部回源失败      检测到连续回源失败
T+5s         源站健康检查连续 3 次失败                  -              标记源站 offline
T+10s        自动切换到备用源站（不同机房）              回源恢复 70%     DNS 切换 + 路由更新
T+30s        备用源站接管回源请求                       回源恢复 95%     备用源站自动扩容
T+60s        启用 stale-while-revalidate              缓存内容可用     延长缓存 TTL
T+5min       主源站开始恢复（UPS 供电）                 -              灰度验证
T+30min      主源站完全恢复                            正常            逐步回切
```

```python
class OriginDisasterRecovery:
    """源站灾难恢复"""

    def on_origin_down(self, origin_id: str):
        """源站宕机处理"""
        # 1. 标记源站不可用
        self.redis.set(f"origin:status:{origin_id}", "offline", ex=600)

        # 2. 自动切换到备用源站
        backup = self.get_backup_origin(origin_id)
        if backup:
            self.redis.set(f"origin:active:{origin_id}", backup.id, ex=1800)
            self.log(f"源站 {origin_id} 切换到备用 {backup.id}")
        else:
            # 无备用源站 → 进入全降级模式
            self.enter_full_degradation_mode()

        # 3. 延长所有边缘缓存 TTL（减少回源需求）
        self.broadcast_ttl_extension(factor=6)  # TTL × 6

        # 4. 停止所有低优先级 PURGE（减少缓存失效）
        self.pause_purge_events(priority_below="high")

    def enter_full_degradation_mode(self):
        """全降级模式：所有不可缓存 API 返回降级响应"""
        self.redis.set("global:degradation", "full", ex=1800)
        self.alert_service.critical("源站完全不可用，进入全降级模式")

        # 所有边缘节点返回 stale 缓存或通用降级内容
        for node in self.edge_registry.get_active_nodes():
            self.message_bus.publish(f"edge:config:{node.id}", {
                "mode": "full_degradation",
                "stale_ttl_multiplier": 6,
                "purge_enabled": False,
            })
```

## 常见陷阱（深度分析）

### 陷阱 1：动态内容也用长 TTL

**后果：** 个性化推荐被缓存 → 用户 A 看到用户 B 的推荐 → 数据泄露。

**解决方案：** 个性化内容不缓存或使用 per-user 缓存键 + 极短 TTL（< 10s）。

### 陷阱 2：缓存失效只靠 TTL

**后果：** 视频评分更新后，最长 5 分钟用户看到旧评分 → 数据不一致 → 用户投诉。

**解决方案：** TTL + 主动 PURGE 双重保障。

### 陷阱 3：边缘部署全量服务

**后果：** 边缘节点资源不够 → 服务崩溃 → 该区域所有用户受影响。

**解决方案：** 边缘只部署轻量 Worker。判断标准：CPU < 10ms 且内存 < 50MB。

### 陷阱 4：DNS 解析不考虑节点负载

**后果：** 某节点负载高但仍被路由到 → 延迟增加。

**解决方案：** 路由优化器同时考虑延迟和负载。

### 陷阱 5：不做源站保护

**后果：** 突发热点 → 大量回源 → 源站崩溃 → 全站不可用。

**解决方案：** 回源限流 + stale 缓存降级 + 回源合并。

## 延伸思考

### 边缘 AI 推理

在边缘节点部署轻量推荐模型（TensorFlow Lite / ONNX Runtime），将个性化推荐从"不可缓存"变为"边缘可计算"。

```
用户请求 → 边缘节点 → 加载轻量模型（< 50MB）→ 输入用户特征 → 输出推荐列表
                     ↓ 延迟 < 50ms（vs 回源 200-500ms）
```

**技术要点：**
- 模型蒸馏：将 1GB 的源站模型蒸馏为 < 50MB 的边缘模型，精度损失 < 3%
- 模型分发：通过 CDN 自身分发模型文件到边缘节点（模型也是静态资源）
- 增量更新：仅同步模型权重 diff（减少分发延迟）
- 降级策略：边缘模型推理失败 → 回退到路由优化方案

### WebTransport / QUIC

替代 HTTP/2，实现 0-RTT 连接建立，特别适合实时弹幕和进度同步等低延迟场景。

```
传统 HTTPS：1-RTT TLS 握手 + 1-RTT TCP 握手 = 2-RTT 才能发数据
QUIC 0-RTT：首次连接 1-RTT，后续连接 0-RTT（复用上次协商的密钥）
```

**注意事项：**
- 0-RTT 有重放攻击风险，仅用于幂等请求
- 连接迁移：用户从 WiFi 切到 4G，QUIC 连接不断（Connection ID 不变）
- 拥塞控制：QUIC 支持用户态拥塞控制算法（BBR v2），可针对流媒体优化

### Prefetch 预取策略

基于用户行为预测下一个请求，提前在边缘缓存。可将缓存命中率从 80% 提升到 90%+。

```python
class PredictivePrefetcher:
    """预测性预取：基于用户行为模式提前缓存"""

    def on_user_action(self, user_id: str, action: str, video_id: str):
        """用户行为触发预取"""
        # 1. 用户浏览视频详情页 → 预取视频分片清单
        if action == "view_detail":
            self.prefetch_queue.submit(
                edge_node=self.get_user_edge(user_id),
                keys=[f"video_manifest:{video_id}"],
                priority="high",
                delay_seconds=0,  # 立即预取
            )

        # 2. 用户开始播放视频 → 预取推荐列表中的下一个视频
        elif action == "start_play":
            next_videos = self.predict_next_videos(user_id, video_id)
            for i, vid in enumerate(next_videos[:3]):
                self.prefetch_queue.submit(
                    edge_node=self.get_user_edge(user_id),
                    keys=[f"video:{vid}", f"video_manifest:{vid}"],
                    priority="medium" if i == 0 else "low",
                    delay_seconds=10 * i,  # 依次预取，避免瞬时负载
                )

        # 3. 用户暂停视频 → 预取同系列其他集数
        elif action == "pause":
            series_videos = self.get_series_videos(video_id)
            self.prefetch_queue.submit(
                edge_node=self.get_user_edge(user_id),
                keys=[f"video:{vid}" for vid in series_videos[:2]],
                priority="low",
                delay_seconds=30,
            )
```

### 多活源站与全局负载均衡

当平台规模进一步扩大，单一源站成为瓶颈时，需要多活源站架构：

```
边缘节点 → 全局负载均衡 → 源站 A（美东，主）
                        → 源站 B（欧洲，热备）
                        → 源站 C（亚太，热备）

数据同步：源站间通过 Binlog 实时同步，延迟 < 1 秒
写入路由：写入请求统一路由到主源站，确保一致性
读取路由：读取请求路由到最近的源站，降低延迟
```

### 监控体系设计

边缘计算场景下，监控是核心挑战——50 个节点的指标采集和聚合。

```
监控层次：
  L1 边缘自监控：每节点 Agent 采集 CPU/内存/缓存命中率/回源 QPS
  L2 区域聚合：每区域 1 个 Aggregator，汇总区域指标
  L3 全局仪表盘：实时展示全球延迟热力图、缓存命中率、源站负载

关键告警指标：
  - 边缘缓存命中率 < 70%（全局）→ 可能缓存配置错误
  - 单节点回源 QPS > 5000 → 可能缓存穿透或热点
  - P90 延迟 > 200ms 持续 5 分钟 → SLA 告警
  - 源站 CPU > 80% → 需要关注回源量
  - PURGE 成功率 < 90% → 缓存一致性风险
  - 节点心跳丢失 > 30 秒 → 节点可能宕机
```
## 缓存预热与预取系统

```python
class PrefetchScheduler:
    """缓存预热调度器"""

    def prefetch_new_content(self, content_list):
        """新内容发布时主动预热"""
        for content in sorted(content_list, key=lambda c: c["priority"], reverse=True):
            # 按热度分批：热门内容先预热
            edge_nodes = self.get_nodes_in_region(content["target_region"])
            for node in edge_nodes:
                # 带宽限制：预取最多使用 20% 的节点带宽
                if self.get_prefetch_bandwidth_usage(node) > 0.2 * node.total_bandwidth:
                    continue
                try:
                    self.http_client.warm_cache(
                        node.ip, content["url"], content["headers"])
                    self.db.insert("prefetch_log", {
                        "node_id": node.id, "url": content["url"],
                        "status": "success", "size_bytes": content["size"]
                    })
                except Exception as e:
                    self.db.insert("prefetch_log", {
                        "node_id": node.id, "url": content["url"],
                        "status": "failed", "error": str(e)
                    })

    def predictive_prefetch(self, access_log_region):
        """基于访问模式预测预热"""
        # 分析最近 1 小时的访问模式
        patterns = self.analyzer.extract_patterns(access_log_region)
        # 找出"用户访问 A 后 80% 会访问 B"的模式
        predictions = []
        for pattern in patterns:
            if pattern["confidence"] > 0.8 and pattern["count"] > 100:
                predictions.append({
                    "url": pattern["next_url"],
                    "confidence": pattern["confidence"],
                    "reason": f"访问 {pattern['current_url']} 后 {pattern['confidence']:.0%} 会访问"
                })
        return predictions
```

## 多源站故障转移

```python
class OriginFailoverManager:
    """多源站自动故障转移"""

    def __init__(self):
        self.origins = [
            {"url": "https://origin-primary.example.com", "priority": 1, "healthy": True},
            {"url": "https://origin-secondary.example.com", "priority": 2, "healthy": True},
            {"url": "https://origin-tertiary.example.com", "priority": 3, "healthy": True},
        ]
        self.circuit_breaker = {}  # origin_url → CircuitBreaker

    def get_healthy_origin(self):
        """获取当前健康的最高优先级源站"""
        for origin in sorted(self.origins, key=lambda o: o["priority"]):
            cb = self.circuit_breaker.get(origin["url"])
            if cb and cb.is_open:
                continue  # 熔断中，跳过
            return origin
        return None  # 所有源站不可用

    def health_check_loop(self):
        """每 5 秒检查源站健康状态"""
        while True:
            for origin in self.origins:
                try:
                    resp = requests.head(origin["url"] + "/health", timeout=3)
                    origin["healthy"] = resp.status_code == 200
                    cb = self.circuit_breaker.setdefault(origin["url"], CircuitBreaker())
                    if origin["healthy"]:
                        cb.record_success()
                    else:
                        cb.record_failure()
                except Exception:
                    origin["healthy"] = False
                    cb = self.circuit_breaker.setdefault(origin["url"], CircuitBreaker())
                    cb.record_failure()
            sleep(5)

class CircuitBreaker:
    """源站熔断器"""
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.last_failure_time = None
        self.is_open = False

    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = now()
        if self.failure_count >= self.failure_threshold:
            self.is_open = True

    def record_success(self):
        self.failure_count = 0
        self.is_open = False

    @property
    def should_allow(self):
        if not self.is_open:
            return True
        # 半开状态：超过恢复时间后允许一次尝试
        if (now() - self.last_failure_time).seconds > self.recovery_timeout:
            return True
        return False
```

## DDoS 防护系统

```python
class DdosProtectionService:
    """分层 DDoS 防护"""

    def check_request(self, request):
        """四层防护检查"""
        # L1: IP 速率限制
        if not self.rate_limiter.allow(request.client_ip):
            return {"action": "block", "reason": "rate_limit_exceeded"}

        # L2: JS Challenge（可疑流量）
        if self.is_suspicious(request):
            return {"action": "challenge", "reason": "suspicious_traffic"}

        # L3: 地理封锁
        if self.geo_blocker.is_blocked(request.client_ip):
            return {"action": "block", "reason": "geo_blocked"}

        # L4: 放行
        return {"action": "allow"}

class IPRateLimiter:
    """Redis Sorted Set 滑动窗口限流"""
    def allow(self, ip, window_seconds=10, max_requests=100):
        key = f"rate_limit:{ip}"
        now_ts = time.time()
        window_start = now_ts - window_seconds

        pipe = redis.pipeline()
        pipe.zremrangebyscore(key, 0, window_start)
        pipe.zadd(key, {str(now_ts): now_ts})
        pipe.zcard(key)
        pipe.expire(key, window_seconds)
        results = pipe.execute()

        return results[2] <= max_requests

class JSChallenger:
    """JS 计算挑战验证"""
    def generate_challenge(self, ip):
        challenge_id = str(uuid4())
        difficulty = 4  # 前导零数量
        redis.setex(f"challenge:{challenge_id}", 60, ip)
        return {
            "type": "js_challenge",
            "challenge_id": challenge_id,
            "difficulty": difficulty,
            "script": f"solve_pow('{challenge_id}', {difficulty})"
        }

    def verify_challenge(self, challenge_id, answer, ip):
        stored_ip = redis.get(f"challenge:{challenge_id}")
        if not stored_ip or stored_ip.decode() != ip:
            return False
        if self._check_pow_answer(challenge_id, answer):
            redis.setex(f"verified:{ip}", 3600, "1")  # 验证通过 1 小时
            return True
        return False
```

## 性能分析详细数据

**缓存命中率分内容类型：**

| 内容类型 | 命中率 | 平均大小 | 边缘延迟 | 源站延迟 |
|---------|--------|---------|---------|---------|
| 视频片段 | 85% | 2MB | 20ms | 150ms |
| 图片 | 92% | 200KB | 10ms | 80ms |
| 静态资源 | 98% | 50KB | 5ms | 60ms |
| API 响应 | 60% | 10KB | 30ms | 200ms |

**源站卸载分析：**

| 指标 | 无 CDN | 有 CDN | 改善 |
|------|--------|--------|------|
| 源站 QPS | 100,000 | 10,000 | 90% 卸载 |
| 源站带宽 | 50 Gbps | 5 Gbps | 90% 节省 |
| 用户延迟 | 120ms | 15ms | 8x 提升 |
| 源站服务器数 | 200 台 | 20 台 | 90% 减少 |

## 异常场景补充

### 场景：缓存投毒检测与缓解

```
触发：攻击者通过 HTTP 响应头注入，使边缘节点缓存了恶意内容
检测：
  1. 内容完整性校验：缓存前计算 hash，取缓存时校验 hash
  2. 内容类型检查：URL 是图片但返回了 HTML → 投毒
  3. 响应头检查：缓存键不包含 Vary 头 → 可能被投毒
处理：
  1. 检测到投毒 → 立即清除该 URL 在所有边缘节点的缓存
  2. PURGE 全网缓存（按 URL 批量清除）
  3. 添加缓存键规范化规则
预防：
  - 只缓存明确标记 Cache-Control: public 的响应
  - 缓存键包含 Host + URL + Vary 头
  - 禁止缓存 5xx 响应和超大响应体
```

### 场景：源站超时 + stale-while-revalidate

```python
class StaleWhileRevalidate:
    """源站超时时返回过期缓存，同时异步刷新"""
    def handle_request(self, url):
        cached = self.cache.get(url)
        if cached:
            if cached.is_fresh:
                return cached.content  # 缓存新鲜，直接返回
            elif cached.is_stale:
                # 缓存过期但可用：先返回过期内容，后台刷新
                self.async_refresh(url)
                return cached.content
        # 无缓存或缓存不可用：请求源站
        try:
            content = self.fetch_from_origin(url, timeout=3)
            self.cache.put(url, content)
            return content
        except OriginTimeoutError:
            if cached:
                return cached.content  # 兜底：返回过期缓存
            return self.error_page  # 无缓存也无源站：错误页面
```

### 场景：边缘节点证书过期自动续签

```python
class CertificateAutoRenewal:
    """ACME 协议自动续签 SSL 证书"""
    def check_and_renew(self):
        certs = self.db.query("SELECT * FROM ssl_certificates WHERE expires_at < NOW() + INTERVAL 30 DAY")
        for cert in certs:
            # ACME 续签流程
            new_cert = self.acme_client.renew(cert.domain)
            # 推送到所有边缘节点
            edge_nodes = self.get_all_edge_nodes()
            for node in edge_nodes:
                self.push_certificate(node, new_cert)
            # 重载 Nginx
            self.execute_on_edges(edge_nodes, "nginx -s reload")
            self.db.update("ssl_certificates",
                {"certificate": new_cert.pem, "expires_at": new_cert.expires_at},
                {"id": cert.id})
            self.alert(f"SSL 证书已续签: {cert.domain}")
```

## 实时分析流水线完整实现

```python
class CDNAnalyticsPipeline:
    """边缘日志 → Kafka → ClickHouse → 仪表盘"""

    def ingest_access_log(self, log_line):
        """解析边缘节点访问日志并写入 Kafka"""
        # Nginx 日志格式: $remote_addr - [$time_local] "$request" $status $body_bytes_sent
        parsed = self.parse_nginx_log(log_line)
        event = {
            "timestamp": parsed["time"],
            "edge_node": parsed["server_name"],
            "client_ip": parsed["remote_addr"],
            "method": parsed["request_method"],
            "url": parsed["request_uri"],
            "status": parsed["status"],
            "bytes_sent": parsed["body_bytes_sent"],
            "cache_status": parsed["upstream_cache_status"],  # HIT/MISS/EXPIRED
            "response_time_ms": parsed["request_time"] * 1000,
            "content_type": parsed["content_type"],
            "country": self.geoip.lookup(parsed["remote_addr"]),
        }
        self.kafka_producer.send("cdn_access_logs", json.dumps(event))

    def aggregate_metrics(self):
        """Flink 聚合：1 分钟窗口指标"""
        # ClickHouse 物化视图自动聚合
        # 指标：request_count, cache_hit_rate, avg_latency, bandwidth, error_rate
        # 维度：edge_node, content_type, status_code, country, url_pattern
        pass


class CDNMetricsQueryService:
    """CDN 指标查询服务"""

    def get_realtime_dashboard(self, time_range="1h"):
        """实时仪表盘数据"""
        return {
            "total_requests": self.clickhouse.query(
                f"SELECT count() FROM cdn_access_logs WHERE timestamp > now() - INTERVAL {time_range}"),
            "cache_hit_rate": self.clickhouse.query(
                f"SELECT countIf(cache_status='HIT') / count() FROM cdn_access_logs "
                f"WHERE timestamp > now() - INTERVAL {time_range}"),
            "avg_latency_ms": self.clickhouse.query(
                f"SELECT avg(response_time_ms) FROM cdn_access_logs "
                f"WHERE timestamp > now() - INTERVAL {time_range}"),
            "bandwidth_bytes": self.clickhouse.query(
                f"SELECT sum(bytes_sent) FROM cdn_access_logs "
                f"WHERE timestamp > now() - INTERVAL {time_range}"),
            "error_rate": self.clickhouse.query(
                f"SELECT countIf(status >= 500) / count() FROM cdn_access_logs "
                f"WHERE timestamp > now() - INTERVAL {time_range}"),
            "top_urls": self.clickhouse.query(
                f"SELECT url, count() as hits FROM cdn_access_logs "
                f"WHERE timestamp > now() - INTERVAL {time_range} "
                f"GROUP BY url ORDER BY hits DESC LIMIT 20"),
        }
```

**ClickHouse 建表：**

```sql
CREATE TABLE cdn_access_logs (
    timestamp DateTime,
    edge_node LowCardinality(String),
    client_ip String,
    method LowCardinality(String),
    url String,
    status UInt16,
    bytes_sent UInt64,
    cache_status LowCardinality(String),
    response_time_ms Float32,
    content_type LowCardinality(String),
    country LowCardinality(String)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (edge_node, timestamp)
TTL timestamp + INTERVAL 90 DAY;

-- 物化视图：1 分钟聚合
CREATE MATERIALIZED VIEW cdn_metrics_1min
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(minute)
ORDER BY (edge_node, content_type, minute)
AS SELECT
    toStartOfMinute(timestamp) as minute,
    edge_node,
    content_type,
    count() as request_count,
    sum(bytes_sent) as bandwidth,
    avg(response_time_ms) as avg_latency,
    countIf(cache_status = 'HIT') as cache_hits,
    countIf(status >= 500) as error_count
FROM cdn_access_logs
GROUP BY minute, edge_node, content_type;
```

## CDN 成本优化策略

```python
class CDNCostOptimizer:
    """CDN 成本优化引擎"""

    def analyze_and_optimize(self):
        """分析并推荐优化措施"""
        optimizations = []

        # 1. 源站盾（Origin Shield）：减少源站回源
        if self.get_origin_request_ratio() > 0.15:
            optimizations.append({
                "measure": "启用源站盾",
                "current_origin_ratio": self.get_origin_request_ratio(),
                "expected_reduction": "50% origin requests",
                "monthly_savings": self._calc_origin_shield_savings()
            })

        # 2. 压缩优化：对文本资源启用 Brotli
        text_bytes = self.get_text_content_bytes()
        if text_bytes > 1_000_000_000:  # > 1GB/day
            optimizations.append({
                "measure": "启用 Brotli 压缩（文本资源）",
                "current_bandwidth": f"{text_bytes / 1e9:.1f} GB/day",
                "expected_compression": "70% reduction",
                "monthly_savings": self._calc_compression_savings(text_bytes)
            })

        # 3. 缓存分区：按内容类型设置不同 TTL
        optimizations.append({
            "measure": "优化缓存 TTL 策略",
            "recommended_ttls": {
                "video_segments": "24h",
                "images": "7d",
                "css_js": "30d (versioned)",
                "api_responses": "60s (stale-while-revalidate=300s)"
            }
        })

        return optimizations
```

## 异常场景补充

### 场景：热 Key 雪崩

```
触发：某视频突然爆火 → 100 倍流量冲击单一边缘节点
      → 该节点 CPU 100% → 请求排队 → 超时
检测：
  1. 单 URL QPS > 10,000 → 热点标记
  2. 节点 CPU > 80% → 告警
处理：
  1. 热点 URL 自动复制到更多边缘节点
  2. 客户端请求被 DNS 调度到多个节点
  3. 如仍过载 → 启用请求排队 + 限流
预防：热点检测 + 自动多节点缓存分发
```

### 场景：地理路由故障

```
触发：DNS GEO 路由配置错误 → 亚洲用户被路由到欧洲节点
      → 延迟从 30ms 暴增到 300ms
检测：
  1. 监控各地区的平均延迟 → 亚洲延迟突增
  2. 延迟 > 100ms → 告警
处理：
  1. 立即回滚 DNS 配置到上一个版本
  2. 亚洲用户路由恢复到亚洲节点
  3. DNS TTL 设置为 60s → 1 分钟内恢复
预防：DNS 配置变更前用少量流量灰度验证
```

### 场景：缓存批量失效风暴

```
触发：CMS 发布新版本 → 批量 PURGE 所有缓存 URL
      → 数百万 URL 同时失效 → 请求全部回源 → 源站被打垮
检测：
  1. PURGE 请求速率 > 1000/s → 告警
  2. 源站 QPS 突增 10 倍 → 告警
处理：
  1. PURGE 限流：每秒最多 100 个 PURGE 请求
  2. 批量 PURGE 改为渐进式：每秒失效 100 个 URL
  3. 紧急情况：暂停 PURGE，让缓存自然过期
预防：PURGE 限流 + 批量操作改为渐进式
```

## 异常场景补充

### 场景：热 Key 雪崩防护

```python
class HotKeyProtector:
    """热 Key 雪崩防护"""
    def detect_hot_key(self, url, window_seconds=10, threshold=10000):
        """检测热点 URL"""
        key = f"hot_key:{url}"
        count = self.redis.incr(key)
        if count == 1:
            self.redis.expire(key, window_seconds)
        if count > threshold:
            self._handle_hot_key(url)
            return True
        return False

    def _handle_hot_key(self, url):
        """处理热 Key：复制到更多边缘节点"""
        # 1. 标记为热 Key
        self.redis.set(f"hot:{url}", "1", ex=3600)
        # 2. 通知所有边缘节点缓存该 URL
        for node in self.get_all_edge_nodes():
            self.warm_cache(node, url)
        # 3. 本地缓存 + 永不过期
        self.redis.set(f"cache:hot:{url}", self.fetch_origin(url))
```

### 场景：缓存批量失效风暴

```python
class CacheInvalidationThrottler:
    """缓存失效限流"""
    def batch_invalidate(self, url_patterns):
        """渐进式批量失效"""
        rate = 100  # 每秒最多失效 100 个 URL
        for i in range(0, len(url_patterns), rate):
            batch = url_patterns[i:i + rate]
            for url in batch:
                self.cache.delete(url)
            if i + rate < len(url_patterns):
                time.sleep(1)  # 每秒一批
```

## 性能优化详细数据

**缓存命中率按内容类型和区域：**

| 内容类型 | 亚洲 | 欧洲 | 北美 | 全球平均 |
|---------|------|------|------|---------|
| 视频片段 | 82% | 87% | 89% | 85% |
| 图片 | 90% | 94% | 93% | 92% |
| CSS/JS | 96% | 99% | 98% | 98% |
| API 响应 | 55% | 65% | 62% | 60% |

**源站卸载率：**

| 时间段 | 请求总量 | 缓存命中 | 源站请求 | 卸载率 |
|--------|---------|---------|---------|-------|
| 低峰(0-6h) | 50万/h | 47万/h | 3万/h | 94% |
| 正常(6-18h) | 200万/h | 180万/h | 20万/h | 90% |
| 高峰(18-22h) | 500万/h | 425万/h | 75万/h | 85% |

**月度成本：**

| 组件 | 规格 | 月成本 |
|------|------|-------|
| 100 边缘节点 | 4c8G + 1TB SSD | ¥8 万 |
| 5 源站盾节点 | 8c16G | ¥1 万 |
| Kafka (日志) | 6节点 | ¥3 万 |
| ClickHouse | 3节点 + 5TB SSD | ¥5 万 |
| Prometheus + Grafana | 3节点 | ¥2 万 |
| **合计** | | **¥19 万** |

## CDN 缓存清除 API 完整实现

CDN 缓存清除（Purge）是内容更新的核心操作，需要在保证一致性的同时防止滥用导致源站压力暴增。以下是带速率限制和批量支持的完整 API 实现。

### API 接口设计

```python
import hashlib
import time
import uuid
import json
import threading
from datetime import datetime, timedelta
from enum import Enum
from typing import List, Optional, Dict
from dataclasses import dataclass, field
from collections import defaultdict


class PurgeRequestType(Enum):
    SINGLE_URL = "single_url"        # 单 URL 清除
    WILDCARD = "wildcard"            # 通配符匹配清除（如 /video/*）
    ALL = "all"                      # 全站清除（极高风险）
    TAG_BASED = "tag_based"          # 按缓存标签清除


class PurgePriority(Enum):
    CRITICAL = "critical"    # 紧急：线上事故，立即执行
    HIGH = "high"            # 高优：内容更新，1分钟内执行
    NORMAL = "normal"        # 普通：常规清除，5分钟内执行
    LOW = "low"              # 低优：定时清理，30分钟内执行


@dataclass
class PurgeRequest:
    """CDN 缓存清除请求"""
    request_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    request_type: PurgeRequestType = PurgeRequestType.SINGLE_URL
    priority: PurgePriority = PurgePriority.NORMAL
    urls: List[str] = field(default_factory=list)
    wildcard_patterns: List[str] = field(default_factory=list)
    cache_tags: List[str] = field(default_factory=list)
    caller_id: str = ""
    caller_ip: str = ""
    reason: str = ""
    created_at: datetime = field(default_factory=datetime.utcnow)
    # 批量支持：一个请求可包含最多 1000 个 URL
    MAX_URLS_PER_REQUEST = 1000
    MAX_WILDCARD_PER_REQUEST = 10


class PurgeRateLimiter:
    """
    CDN 清除 API 速率限制器
    多层级限流：全局 QPS 限制 + 单调用者频率限制 + 批量大小限制
    """

    # 全局限制
    GLOBAL_MAX_QPS = 500               # 全局每秒最多 500 个清除请求
    GLOBAL_MAX_DAILY = 100000          # 全局每日最多 10 万个清除请求
    # 单调用者限制
    CALLER_MAX_QPS = 50                # 单调用者每秒最多 50 个
    CALLER_MAX_DAILY = 5000            # 单调用者每日最多 5000 个
    CALLER_MAX_BURST = 10              # 单调用者突发上限 10 个/秒
    # 批量限制
    BATCH_MAX_URLS = 1000              # 单次批量最多 1000 URL
    BATCH_MAX_WILDCARD = 10            # 单次最多 10 个通配符模式
    # 全站清除限制（极高危操作）
    PURGE_ALL_MAX_DAILY = 3            # 每天最多 3 次全站清除
    PURGE_ALL_REQUIRED_ROLES = ["admin", "sre_lead"]  # 全站清除需要的角色

    def __init__(self, redis_client):
        self.redis = redis_client
        self._lock = threading.Lock()

    def check_rate_limit(self, request: PurgeRequest) -> Dict:
        """
        检查速率限制，返回是否允许和原因
        """
        now = time.time()
        minute_key = datetime.utcnow().strftime('%Y%m%d%H%M')
        day_key = datetime.utcnow().strftime('%Y%m%d')

        # 1. 全局 QPS 限制
        global_qps_key = f"purge:ratelimit:global:qps:{minute_key}"
        global_count = self.redis.incr(global_qps_key)
        if global_count == 1:
            self.redis.expire(global_qps_key, 120)  # 2 分钟过期
        if global_count > self.GLOBAL_MAX_QPS * 60:  # 分钟级 = QPS * 60
            return {
                "allowed": False,
                "reason": f"全局 QPS 限制: 当前 {global_count / 60:.0f} QPS > 上限 {self.GLOBAL_MAX_QPS} QPS",
                "retry_after_seconds": 60,
            }

        # 2. 全局每日限制
        global_daily_key = f"purge:ratelimit:global:daily:{day_key}"
        global_daily = self.redis.incr(global_daily_key)
        if global_daily == 1:
            self.redis.expire(global_daily_key, 86400 * 2)
        if global_daily > self.GLOBAL_MAX_DAILY:
            return {
                "allowed": False,
                "reason": f"全局每日限制: 当前 {global_daily} > 上限 {self.GLOBAL_MAX_DAILY}",
                "retry_after_seconds": 86400 - int(now % 86400),
            }

        # 3. 单调用者 QPS 限制
        caller_qps_key = f"purge:ratelimit:caller:{request.caller_id}:qps:{minute_key}"
        caller_count = self.redis.incr(caller_qps_key)
        if caller_count == 1:
            self.redis.expire(caller_qps_key, 120)
        if caller_count > self.CALLER_MAX_QPS * 60:
            return {
                "allowed": False,
                "reason": f"调用者 {request.caller_id} QPS 限制: {caller_count / 60:.0f} > {self.CALLER_MAX_QPS}",
                "retry_after_seconds": 60,
            }

        # 4. 单调用者每日限制
        caller_daily_key = f"purge:ratelimit:caller:{request.caller_id}:daily:{day_key}"
        caller_daily = self.redis.incr(caller_daily_key)
        if caller_daily == 1:
            self.redis.expire(caller_daily_key, 86400 * 2)
        if caller_daily > self.CALLER_MAX_DAILY:
            return {
                "allowed": False,
                "reason": f"调用者 {request.caller_id} 每日限制: {caller_daily} > {self.CALLER_MAX_DAILY}",
                "retry_after_seconds": 86400 - int(now % 86400),
            }

        # 5. 全站清除特殊限制
        if request.request_type == PurgeRequestType.ALL:
            purge_all_key = f"purge:ratelimit:purge_all:daily:{day_key}"
            purge_all_count = self.redis.incr(purge_all_key)
            if purge_all_count == 1:
                self.redis.expire(purge_all_key, 86400 * 2)
            if purge_all_count > self.PURGE_ALL_MAX_DAILY:
                return {
                    "allowed": False,
                    "reason": f"全站清除每日限制: {purge_all_count} > {self.PURGE_ALL_MAX_DAILY}",
                    "retry_after_seconds": 86400 - int(now % 86400),
                }

        # 6. 批量大小限制
        if len(request.urls) > self.BATCH_MAX_URLS:
            return {
                "allowed": False,
                "reason": f"批量 URL 数量超限: {len(request.urls)} > {self.BATCH_MAX_URLS}",
                "retry_after_seconds": 0,
            }
        if len(request.wildcard_patterns) > self.BATCH_MAX_WILDCARD:
            return {
                "allowed": False,
                "reason": f"通配符数量超限: {len(request.wildcard_patterns)} > {self.BATCH_MAX_WILDCARD}",
                "retry_after_seconds": 0,
            }

        return {"allowed": True, "reason": ""}


class PurgeAPIService:
    """
    CDN 缓存清除 API 服务
    支持单 URL、通配符、按标签、全站四种清除模式
    内置速率限制、批量支持、异步执行、结果追踪
    """

    def __init__(self, redis_client, db, edge_nodes, message_bus, alert_service):
        self.redis = redis_client
        self.db = db
        self.edge_nodes = edge_nodes
        self.message_bus = message_bus
        self.alert_service = alert_service
        self.rate_limiter = PurgeRateLimiter(redis_client)

    def submit_purge(self, request: PurgeRequest) -> Dict:
        """
        提交清除请求——API 入口
        """
        # 1. 参数校验
        validation = self._validate_request(request)
        if not validation["valid"]:
            return {"status": "rejected", "reason": validation["reason"]}

        # 2. 速率限制检查
        rate_result = self.rate_limiter.check_rate_limit(request)
        if not rate_result["allowed"]:
            return {
                "status": "rate_limited",
                "reason": rate_result["reason"],
                "retry_after": rate_result["retry_after_seconds"],
            }

        # 3. 全站清除需要额外权限校验
        if request.request_type == PurgeRequestType.ALL:
            auth_result = self._check_purge_all_authorization(request)
            if not auth_result["authorized"]:
                return {"status": "forbidden", "reason": auth_result["reason"]}

        # 4. 持久化请求到数据库
        event_id = f"purge-{request.request_id}"
        self.db.execute_insert(
            """INSERT INTO cdn_purge_events
               (event_id, trigger_source, trigger_reason, purge_scope, purge_keys,
                status, created_at)
               VALUES (%s, %s, %s, %s, %s, %s, %s)""",
            (event_id, 'api_call', request.reason,
             request.request_type.value,
             json.dumps({
                 "urls": request.urls,
                 "wildcards": request.wildcard_patterns,
                 "tags": request.cache_tags,
             }),
             'pending', datetime.utcnow())
        )

        # 5. 发布到消息总线（异步执行）
        self.message_bus.publish("cache:purge:request", {
            "event_id": event_id,
            "request_id": request.request_id,
            "request_type": request.request_type.value,
            "priority": request.priority.value,
            "urls": request.urls,
            "wildcards": request.wildcard_patterns,
            "tags": request.cache_tags,
            "caller_id": request.caller_id,
            "timestamp": time.time(),
        })

        # 6. 返回追踪信息
        return {
            "status": "accepted",
            "event_id": event_id,
            "request_id": request.request_id,
            "tracking_url": f"/api/purge/status/{event_id}",
            "estimated_completion_seconds": self._estimate_completion(request),
        }

    def get_purge_status(self, event_id: str) -> Dict:
        """查询清除请求的执行状态"""
        record = self.db.query_one(
            "SELECT * FROM cdn_purge_events WHERE event_id = %s", event_id
        )
        if not record:
            return {"status": "not_found", "event_id": event_id}

        return {
            "event_id": event_id,
            "status": record["status"],
            "success_count": record["success_count"],
            "fail_count": record["fail_count"],
            "timeout_count": record["timeout_count"],
            "total_nodes": record["success_count"] + record["fail_count"] + record["timeout_count"],
            "started_at": record.get("started_at"),
            "completed_at": record.get("completed_at"),
        }

    def _validate_request(self, request: PurgeRequest) -> Dict:
        """请求参数校验"""
        if request.request_type == PurgeRequestType.SINGLE_URL:
            if not request.urls:
                return {"valid": False, "reason": "单 URL 清除必须提供 urls 参数"}
            for url in request.urls:
                if not url.startswith("/"):
                    return {"valid": False, "reason": f"URL 必须以 / 开头: {url}"}

        elif request.request_type == PurgeRequestType.WILDCARD:
            if not request.wildcard_patterns:
                return {"valid": False, "reason": "通配符清除必须提供 wildcard_patterns 参数"}
            for pattern in request.wildcard_patterns:
                if "*" not in pattern:
                    return {"valid": False, "reason": f"通配符模式必须包含 *: {pattern}"}

        elif request.request_type == PurgeRequestType.TAG_BASED:
            if not request.cache_tags:
                return {"valid": False, "reason": "标签清除必须提供 cache_tags 参数"}

        if not request.caller_id:
            return {"valid": False, "reason": "缺少 caller_id"}

        return {"valid": True, "reason": ""}

    def _check_purge_all_authorization(self, request: PurgeRequest) -> Dict:
        """全站清除需要额外权限"""
        caller_roles = self._get_caller_roles(request.caller_id)
        required = set(self.rate_limiter.PURGE_ALL_REQUIRED_ROLES)
        if not required.intersection(caller_roles):
            return {
                "authorized": False,
                "reason": f"全站清除需要 {required} 角色，当前角色: {caller_roles}",
            }
        return {"authorized": True, "reason": ""}

    def _estimate_completion(self, request: PurgeRequest) -> int:
        """预估完成时间（秒）"""
        if request.request_type == PurgeRequestType.ALL:
            return 30   # 全站清除约 30 秒
        elif request.request_type == PurgeRequestType.WILDCARD:
            return 15   # 通配符清除约 15 秒
        elif len(request.urls) > 100:
            return 10   # 大批量 URL 清除约 10 秒
        else:
            return 3    # 小批量约 3 秒

    def _get_caller_roles(self, caller_id: str) -> set:
        """获取调用者角色"""
        roles = self.redis.smembers(f"caller:roles:{caller_id}")
        return roles or set()
```

**速率限制层级说明：**

| 限制层级 | 限制值 | 目的 | 超限行为 |
|---------|-------|------|---------|
| 全局 QPS | 500/s | 防止全网清除请求压垮边缘节点 | 返回 429 + Retry-After |
| 全局每日 | 10 万/天 | 控制清除总量 | 返回 429 + Retry-After |
| 单调用者 QPS | 50/s | 防止单个用户滥用 | 返回 429 + Retry-After |
| 单调用者每日 | 5000/天 | 防止单用户过度清除 | 返回 429 + Retry-After |
| 全站清除每日 | 3 次/天 | 全站清除极高风险 | 返回 403 |
| 批量 URL 上限 | 1000 个/次 | 防止单次请求过大 | 返回 400 |
| 通配符上限 | 10 个/次 | 通配符匹配开销大 | 返回 400 |

## 地理路由配置与 DNS 流量管理

CDN 的核心能力之一是基于用户地理位置将请求路由到最优边缘节点。以下是完整的 DNS-Based 地理路由配置系统实现。

### 架构设计

```
用户请求 → Local DNS → 权威 DNS（GeoDNS）
                              ↓
                     查询地理路由规则表
                              ↓
                     根据用户 IP → 匹配区域 → 选择节点
                              ↓
                     返回最优边缘节点 IP（A 记录）
                              ↓
              （可选）按权重返回多个 IP → 客户端轮询
```

```python
import math
import time
import random
from datetime import datetime, timedelta
from typing import List, Dict, Optional, Tuple
from dataclasses import dataclass, field
from enum import Enum


class GeoRegion(Enum):
    """全球大区划分"""
    CN_NORTH = "cn_north"      # 中国华北
    CN_EAST = "cn_east"        # 中国华东
    CN_SOUTH = "cn_south"      # 中国华南
    CN_WEST = "cn_west"        # 中国西部
    APAC_EAST = "apac_east"    # 亚太东部（日韩）
    APAC_SE = "apac_se"        # 东南亚
    NA_EAST = "na_east"        # 北美东部
    NA_WEST = "na_west"        # 北美西部
    EU_WEST = "eu_west"        # 西欧
    EU_CENTRAL = "eu_central"  # 中欧
    MEA = "mea"                # 中东非洲
    LATAM = "latam"            # 拉美


@dataclass
class GeoRouteRule:
    """地理路由规则"""
    rule_id: str
    source_region: GeoRegion           # 用户来源区域
    domain_pattern: str                # 域名匹配模式（如 *.example.com）
    primary_nodes: List[str]            # 主节点列表（节点编码）
    fallback_nodes: List[str]           # 备用节点列表
    weight_distribution: Dict[str, int] = field(default_factory=dict)  # 节点→权重
    health_check_required: bool = True  # 是否需要健康检查
    latency_based: bool = False          # 是否基于延迟选择（而非纯地理）
    dns_ttl_seconds: int = 60           # DNS TTL（秒），短 TTL 便于故障转移
    priority: int = 100                 # 规则优先级，数值越大越优先


@dataclass
class GeoIPRecord:
    """GeoIP 数据库记录"""
    ip_start: int           # IP 起始值（数值形式，便于范围查询）
    ip_end: int             # IP 结束值
    country_code: str       # 国家代码（ISO 3166-1 alpha-2）
    region_code: str        # 区域代码
    city: str               # 城市
    latitude: float          # 纬度
    longitude: float         # 经度
    isp: str                 # 运营商


class GeoIPResolver:
    """
    GeoIP 解析器：将用户 IP 映射到地理区域
    支持 MaxMind GeoIP2 数据库 + 自定义 IP 段覆盖
    """

    def __init__(self, db, redis):
        self.db = db
        self.redis = redis
        self._cache_ttl = 3600  # GeoIP 缓存 1 小时
        # 自定义 IP 段覆盖（优先级高于 MaxMind）
        self._custom_overrides = self._load_custom_overrides()

    def resolve(self, client_ip: str) -> Tuple[GeoRegion, Dict]:
        """
        将客户端 IP 解析为地理区域
        返回: (区域枚举, 地理信息字典)
        """
        # 1. 查询缓存
        cache_key = f"geoip:cache:{client_ip}"
        cached = self.redis.get(cache_key)
        if cached:
            data = json.loads(cached)
            return GeoRegion(data["region"]), data

        # 2. 检查自定义覆盖
        ip_int = self._ip_to_int(client_ip)
        for override in self._custom_overrides:
            if override["ip_start"] <= ip_int <= override["ip_end"]:
                result = {
                    "region": override["region"],
                    "country": override["country"],
                    "city": override.get("city", ""),
                    "latitude": override.get("latitude", 0),
                    "longitude": override.get("longitude", 0),
                    "isp": override.get("isp", ""),
                    "source": "custom_override",
                }
                self.redis.setex(cache_key, self._cache_ttl, json.dumps(result))
                return GeoRegion(override["region"]), result

        # 3. 查询 MaxMind GeoIP2 数据库
        geo_data = self._query_maxmind(client_ip)
        if geo_data:
            region = self._map_country_to_region(
                geo_data["country_code"], geo_data.get("subdivision", "")
            )
            result = {
                "region": region.value,
                "country": geo_data["country_code"],
                "city": geo_data.get("city", ""),
                "latitude": geo_data.get("latitude", 0),
                "longitude": geo_data.get("longitude", 0),
                "isp": geo_data.get("isp", ""),
                "source": "maxmind",
            }
            self.redis.setex(cache_key, self._cache_ttl, json.dumps(result))
            return region, result

        # 4. 无法解析 → 默认路由到亚太
        default = GeoRegion.CN_EAST
        result = {"region": default.value, "source": "default_fallback"}
        self.redis.setex(cache_key, self._cache_ttl, json.dumps(result))
        return default, result

    def _ip_to_int(self, ip: str) -> int:
        """IP 地址转整数"""
        parts = ip.split(".")
        return sum(int(p) << (8 * (3 - i)) for i, p in enumerate(parts))

    def _map_country_to_region(self, country_code: str, subdivision: str) -> GeoRegion:
        """将国家+省份映射到业务区域"""
        # 中国各省份映射
        cn_north_provinces = {"BJ", "TJ", "HE", "SX", "NM", "LN", "JL", "HL"}
        cn_east_provinces = {"SH", "JS", "ZJ", "AH", "FJ", "SD", "JS"}
        cn_south_provinces = {"GD", "GX", "HI", "HK", "MO"}
        cn_west_provinces = {"CQ", "SC", "GZ", "YN", "XZ", "GS", "QH", "NX", "XJ"}

        if country_code == "CN":
            if subdivision in cn_north_provinces:
                return GeoRegion.CN_NORTH
            elif subdivision in cn_east_provinces:
                return GeoRegion.CN_EAST
            elif subdivision in cn_south_provinces:
                return GeoRegion.CN_SOUTH
            elif subdivision in cn_west_provinces:
                return GeoRegion.CN_WEST
            return GeoRegion.CN_EAST  # 默认华东

        # 其他国家映射
        country_region_map = {
            "JP": GeoRegion.APAC_EAST, "KR": GeoRegion.APAC_EAST,
            "SG": GeoRegion.APAC_SE, "TH": GeoRegion.APAC_SE,
            "MY": GeoRegion.APAC_SE, "ID": GeoRegion.APAC_SE,
            "US": GeoRegion.NA_WEST, "CA": GeoRegion.NA_WEST,
            "GB": GeoRegion.EU_WEST, "DE": GeoRegion.EU_CENTRAL,
            "FR": GeoRegion.EU_WEST, "AE": GeoRegion.MEA,
            "BR": GeoRegion.LATAM, "MX": GeoRegion.LATAM,
        }
        return country_region_map.get(country_code, GeoRegion.CN_EAST)

    def _load_custom_overrides(self) -> list:
        """加载自定义 IP 段覆盖规则（优先级高于 MaxMind）"""
        overrides = self.db.query(
            "SELECT ip_start, ip_end, region, country, city, latitude, longitude, isp "
            "FROM geoip_overrides WHERE enabled = TRUE"
        )
        return [dict(r) for r in overrides]

    def _query_maxmind(self, ip: str) -> Optional[Dict]:
        """查询 MaxMind GeoIP2 数据库"""
        try:
            reader = self.maxmind_reader  # maxminddb.open('GeoIP2-City.mmdb')
            result = reader.get(ip)
            if result:
                return {
                    "country_code": result.get("country", {}).get("iso_code", ""),
                    "subdivision": result.get("subdivisions", [{}])[0].get("iso_code", ""),
                    "city": result.get("city", {}).get("names", {}).get("en", ""),
                    "latitude": result.get("location", {}).get("latitude", 0),
                    "longitude": result.get("location", {}).get("longitude", 0),
                    "isp": result.get("traits", {}).get("isp", ""),
                }
        except Exception:
            pass
        return None


class GeoDNSRouter:
    """
    GeoDNS 路由器：基于用户位置 + 节点健康度 + 负载权重做 DNS 级流量调度
    核心能力：
    - 按地理区域路由到最近节点
    - 按权重做流量分配（灰度发布、AB 测试）
    - 故障自动转移（健康检查失败时自动剔除）
    - 延迟感知路由（可选）
    """

    def __init__(self, geoip_resolver: GeoIPResolver, redis, db, edge_registry):
        self.geoip = geoip_resolver
        self.redis = redis
        self.db = db
        self.edge_registry = edge_registry

    def resolve(self, domain: str, client_ip: str) -> Dict:
        """
        DNS 解析入口：给定域名和客户端 IP，返回应路由的边缘节点 IP 列表
        """
        # 1. 解析客户端地理区域
        region, geo_info = self.geoip.resolve(client_ip)

        # 2. 匹配路由规则
        rule = self._match_rule(domain, region)
        if not rule:
            # 无匹配规则 → 路由到全局默认节点
            default_nodes = self.edge_registry.get_default_nodes(domain)
            return self._build_dns_response(default_nodes, region, source="default")

        # 3. 从主节点和备用节点中筛选健康节点
        healthy_primary = self._filter_healthy_nodes(rule.primary_nodes)
        healthy_fallback = self._filter_healthy_nodes(rule.fallback_nodes)

        # 4. 如果是延迟感知模式，选择延迟最低的节点
        if rule.latency_based:
            candidates = healthy_primary or healthy_fallback
            selected = self._select_by_latency(candidates, region)
            return self._build_dns_response(selected, region, source="latency_based")

        # 5. 按权重分配流量
        if healthy_primary:
            selected = self._select_by_weight(
                healthy_primary, rule.weight_distribution, region
            )
            return self._build_dns_response(selected, region, source="geo_weighted")

        # 6. 主节点全部不健康 → 降级到备用节点
        if healthy_fallback:
            self.alert_service.warn(
                f"区域 {region.value} 主节点全部不健康，降级到备用节点"
            )
            return self._build_dns_response(
                healthy_fallback, region, source="fallback"
            )

        # 7. 所有节点不健康 → 跨区域降级
        cross_region_nodes = self._find_cross_region_nodes(region, domain)
        return self._build_dns_response(
            cross_region_nodes, region, source="cross_region_fallback"
        )

    def _match_rule(self, domain: str, region: GeoRegion) -> Optional[GeoRouteRule]:
        """匹配地理路由规则"""
        # 从数据库查询匹配的规则
        rules = self.db.query(
            """SELECT * FROM geo_route_rules
               WHERE source_region = %s
               AND (%s LIKE domain_pattern OR domain_pattern = '*')
               AND enabled = TRUE
               ORDER BY priority DESC LIMIT 1""",
            region.value, domain
        )
        if rules:
            r = rules[0]
            return GeoRouteRule(
                rule_id=r["rule_id"],
                source_region=GeoRegion(r["source_region"]),
                domain_pattern=r["domain_pattern"],
                primary_nodes=json.loads(r["primary_nodes"]),
                fallback_nodes=json.loads(r["fallback_nodes"]),
                weight_distribution=json.loads(r.get("weight_distribution", "{}")),
                dns_ttl_seconds=r.get("dns_ttl_seconds", 60),
                priority=r.get("priority", 100),
            )
        return None

    def _filter_healthy_nodes(self, node_codes: List[str]) -> List[str]:
        """筛选健康节点"""
        healthy = []
        for code in node_codes:
            status = self.redis.get(f"edge:heartbeat:{code}")
            if status and status != "offline":
                # 检查负载
                load = self.redis.hgetall(f"edge:load:{code}")
                if load:
                    cpu = float(load.get("cpu_percent", 0))
                    if cpu < 90:  # CPU < 90% 视为健康
                        healthy.append(code)
                else:
                    healthy.append(code)  # 无负载数据视为健康
        return healthy

    def _select_by_weight(self, nodes: List[str], weights: Dict, region: GeoRegion) -> List[str]:
        """按权重选择节点（加权随机）"""
        if not weights:
            return nodes[:2]  # 无权重配置 → 返回前 2 个

        node_weights = [(n, weights.get(n, 100)) for n in nodes]
        total_weight = sum(w for _, w in node_weights)

        # 加权随机选择（返回 2 个 IP 用于 DNS 轮询）
        selected = []
        remaining = list(node_weights)
        for _ in range(min(2, len(nodes))):
            r = random.uniform(0, sum(w for _, w in remaining))
            cumulative = 0
            for i, (node, weight) in enumerate(remaining):
                cumulative += weight
                if r <= cumulative:
                    selected.append(node)
                    remaining.pop(i)
                    break

        return selected

    def _select_by_latency(self, nodes: List[str], region: GeoRegion) -> List[str]:
        """基于延迟选择最优节点"""
        latencies = []
        for code in nodes:
            latency_key = f"edge:latency:{region.value}:{code}"
            latency = self.redis.get(latency_key)
            if latency:
                latencies.append((code, int(latency)))
            else:
                latencies.append((code, 999))  # 无数据则排到最后

        # 按延迟排序，返回前 2 个
        latencies.sort(key=lambda x: x[1])
        return [code for code, _ in latencies[:2]]

    def _build_dns_response(self, node_codes: List[str], region: GeoRegion,
                            source: str) -> Dict:
        """构建 DNS 响应"""
        ips = []
        for code in node_codes:
            ip = self.redis.hget(f"edge:node:{code}", "ip")
            if ip:
                ips.append(ip)

        return {
            "answers": [{"type": "A", "ttl": 60, "data": ip} for ip in ips],
            "region": region.value,
            "routing_source": source,
            "node_count": len(ips),
        }
```

**地理路由规则表：**

```sql
CREATE TABLE geo_route_rules (
    id                BIGINT PRIMARY KEY AUTO_INCREMENT,
    rule_id           VARCHAR(64) NOT NULL UNIQUE COMMENT '规则唯一ID',
    source_region     VARCHAR(32) NOT NULL COMMENT '用户来源区域',
    domain_pattern    VARCHAR(256) NOT NULL COMMENT '域匹配模式(支持通配符)',
    primary_nodes     JSON NOT NULL COMMENT '主节点列表,["cn-east-1","cn-east-2"]',
    fallback_nodes    JSON COMMENT '备用节点列表',
    weight_distribution JSON COMMENT '节点权重分配,{"cn-east-1":70,"cn-east-2":30}',
    health_check_required BOOLEAN DEFAULT TRUE,
    latency_based     BOOLEAN DEFAULT FALSE COMMENT '是否延迟感知路由',
    dns_ttl_seconds   INT DEFAULT 60 COMMENT 'DNS TTL(秒)',
    priority          INT DEFAULT 100 COMMENT '规则优先级',
    enabled           BOOLEAN DEFAULT TRUE,
    created_at        DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at        DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_region_domain (source_region, domain_pattern),
    INDEX idx_priority (priority DESC)
) ENGINE=InnoDB COMMENT='地理路由规则配置';
```

**路由效果量化：**

| 用户区域 | 路由节点 | 物理距离 | 网络延迟 | DNS 解析时间 |
|---------|---------|---------|---------|------------|
| 北京 → cn-north-1 | 华北节点 | 50km | 8ms | 5ms |
| 上海 → cn-east-1 | 华东节点 | 0km | 2ms | 5ms |
| 东京 → apac-east-1 | 东京节点 | 0km | 3ms | 8ms |
| 柏林 → eu-central-1 | 法兰克福节点 | 500km | 12ms | 15ms |

## 实时日志流管道完整实现：边缘 → Kafka → ClickHouse → Grafana

CDN 运营需要实时监控全球 50+ 节点的访问指标，延迟要求在秒级。以下是完整的实时日志流管道实现。

### 架构设计

```
50 边缘节点 → Fluentd 采集 → Kafka(3分区×16副本) → Flink 实时聚合
                                                          ↓
                                                    ClickHouse 写入
                                                          ↓
                                                    Grafana 仪表盘
                                                          ↓
                                                   告警系统(Prometheus)
```

```python
import json
import time
import hashlib
import logging
from datetime import datetime, timedelta
from typing import Dict, List, Optional
from dataclasses import dataclass, field
from confluent_kafka import Producer, Consumer
import signal
import sys


# ==================== 边缘端日志采集 ====================

class EdgeLogCollector:
    """
    边缘节点日志采集器
    将 Nginx 访问日志实时采集并发送到 Kafka
    支持批量聚合、本地缓冲、断点续传
    """

    def __init__(self, edge_node_code: str, kafka_brokers: List[str]):
        self.edge_node_code = edge_node_code
        self.kafka_brokers = kafka_brokers
        self.producer = Producer({
            "bootstrap.servers": ",".join(kafka_brokers),
            "acks": "1",                  # 至少一个副本确认
            "compression.type": "lz4",    # 压缩降低带宽
            "linger.ms": 100,             # 100ms 微批，平衡延迟和吞吐
            "batch.size": 32768,          # 32KB 批次
            "retry.backoff.ms": 1000,
            "retries": 3,
        })
        self.local_buffer = []            # 本地缓冲（Kafka 不可用时）
        self.buffer_limit = 10000         # 本地缓冲上限
        self.stats = {
            "collected": 0,
            "sent_kafka": 0,
            "buffered_local": 0,
            "parse_errors": 0,
        }

    def on_access_log(self, log_line: str):
        """
        处理一条 Nginx 访问日志
        日志格式: $remote_addr - $time_local "$request" $status $body_bytes_sent
                   $upstream_cache_status $request_time $content_type
        """
        try:
            parsed = self._parse_nginx_log(log_line)
            event = {
                # 基础字段
                "timestamp": parsed["time_iso"],
                "edge_node": self.edge_node_code,
                "client_ip": parsed["remote_addr"],
                "method": parsed["request_method"],
                "url": parsed["request_uri"],
                "status": int(parsed["status"]),
                "bytes_sent": int(parsed["body_bytes_sent"]),
                # CDN 专有字段
                "cache_status": parsed.get("upstream_cache_status", "MISS"),
                "response_time_ms": float(parsed.get("request_time", 0)) * 1000,
                "content_type": parsed.get("content_type", "unknown"),
                # 地理信息（边缘节点本地 GeoIP 解析，减少云端压力）
                "country": parsed.get("geoip_country", ""),
                "region": parsed.get("geoip_region", ""),
            }

            self.stats["collected"] += 1

            # 尝试发送到 Kafka
            if self._send_to_kafka(event):
                self.stats["sent_kafka"] += 1
            else:
                # Kafka 不可用 → 本地缓冲
                self._buffer_locally(event)

        except Exception as e:
            self.stats["parse_errors"] += 1
            logging.warning(f"日志解析失败: {e}, line={log_line[:100]}")

    def _send_to_kafka(self, event: dict) -> bool:
        """发送事件到 Kafka"""
        try:
            # 分区键: edge_node + 小时 → 同一节点同一小时的数据在同一分区
            partition_key = f"{self.edge_node_code}:{event['timestamp'][:13]}".encode()
            self.producer.produce(
                topic="cdn_access_logs",
                key=partition_key,
                value=json.dumps(event).encode("utf-8"),
            )
            return True
        except Exception as e:
            logging.warning(f"Kafka 发送失败: {e}")
            return False

    def _buffer_locally(self, event: dict):
        """本地缓冲（Kafka 不可用时）"""
        if len(self.local_buffer) < self.buffer_limit:
            self.local_buffer.append(event)
            self.stats["buffered_local"] += 1
        else:
            # 缓冲区满 → 丢弃（记录丢弃计数）
            logging.error("本地缓冲区已满，丢弃日志事件")

    def flush_buffer(self):
        """尝试发送本地缓冲的数据（Kafka 恢复后调用）"""
        remaining = []
        for event in self.local_buffer:
            if self._send_to_kafka(event):
                self.stats["sent_kafka"] += 1
            else:
                remaining.append(event)
        self.local_buffer = remaining
        logging.info(f"缓冲区刷新: 剩余 {len(remaining)} 条")

    def _parse_nginx_log(self, line: str) -> dict:
        """解析 Nginx 自定义日志格式"""
        # 简化解析逻辑，生产环境应使用正则或专门解析库
        parts = line.strip().split("|")
        result = {}
        keys = ["remote_addr", "time_iso", "request_method", "request_uri",
                "status", "body_bytes_sent", "upstream_cache_status",
                "request_time", "content_type", "geoip_country", "geoip_region"]
        for i, key in enumerate(keys):
            result[key] = parts[i].strip() if i < len(parts) else ""
        return result


# ==================== Flink 实时聚合（Python 伪代码，实际用 Flink SQL/Java）====================

FLINK_AGGREGATION_SQL = """
-- Flink SQL: 1 分钟滚动窗口聚合 CDN 指标
-- 输入: Kafka topic cdn_access_logs
-- 输出: Kafka topic cdn_metrics_1min

CREATE TABLE cdn_access_log_stream (
    timestamp       STRING,
    edge_node       STRING,
    client_ip       STRING,
    method          STRING,
    url             STRING,
    status          INT,
    bytes_sent      BIGINT,
    cache_status    STRING,
    response_time_ms FLOAT,
    content_type    STRING,
    country         STRING,
    region          STRING,
    event_time      AS TO_TIMESTAMP(timestamp),
    WATERMARK FOR event_time AS event_time - INTERVAL '30' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'cdn_access_logs',
    'properties.bootstrap.servers' = 'kafka-1:9092,kafka-2:9092,kafka-3:9092',
    'format' = 'json',
    'scan.startup.mode' = 'latest'
);

-- 1 分钟聚合：按边缘节点 + 内容类型
CREATE TABLE cdn_metrics_1min WITH (
    'connector' = 'kafka',
    'topic' = 'cdn_metrics_1min',
    'properties.bootstrap.servers' = 'kafka-1:9092',
    'format' = 'json'
) AS SELECT
    TUMBLE_START(event_time, INTERVAL '1' MINUTE) AS minute_start,
    edge_node,
    content_type,
    COUNT(*) AS request_count,
    SUM(bytes_sent) AS bandwidth_bytes,
    AVG(response_time_ms) AS avg_latency_ms,
    MAX(response_time_ms) AS max_latency_ms,
    -- 百分位延迟近似（P90 / P99）
    PERCENTILE_APPROX(response_time_ms, 0.9) AS p90_latency_ms,
    PERCENTILE_APPROX(response_time_ms, 0.99) AS p99_latency_ms,
    -- 缓存命中率
    CAST(SUM(CASE WHEN cache_status = 'HIT' THEN 1 ELSE 0 END) AS DOUBLE)
        / COUNT(*) AS cache_hit_rate,
    -- 错误率
    CAST(SUM(CASE WHEN status >= 500 THEN 1 ELSE 0 END) AS DOUBLE)
        / COUNT(*) AS error_rate,
    -- 按国家分布（Top 5）
    COLLECT_LIST(country) AS country_distribution
FROM cdn_access_log_stream
GROUP BY
    TUMBLE(event_time, INTERVAL '1' MINUTE),
    edge_node,
    content_type;
"""


# ==================== ClickHouse 写入与查询 ====================

class ClickHouseWriter:
    """
    ClickHouse 写入器：消费 Kafka 聚合指标，批量写入 ClickHouse
    """

    def __init__(self, clickhouse_client, kafka_brokers: List[str]):
        self.ch = clickhouse_client
        self.consumer = Consumer({
            "bootstrap.servers": ",".join(kafka_brokers),
            "group.id": "clickhouse-writer",
            "auto.offset.reset": "latest",
            "enable.auto.commit": False,
            "max.poll.records": 500,
        })
        self.batch_size = 1000        # 每批最多 1000 行
        self.flush_interval_sec = 5    # 每 5 秒刷新一次

    def run(self):
        """持续消费 Kafka 并写入 ClickHouse"""
        self.consumer.subscribe(["cdn_metrics_1min"])
        batch = []
        last_flush = time.time()

        while True:
            msg = self.consumer.poll(timeout=1.0)
            if msg is not None:
                try:
                    metric = json.loads(msg.value().decode("utf-8"))
                    batch.append(metric)
                except Exception as e:
                    logging.warning(f"消息解析失败: {e}")

            # 达到批次大小或刷新间隔 → 写入 ClickHouse
            if len(batch) >= self.batch_size or (
                batch and time.time() - last_flush >= self.flush_interval_sec
            ):
                self._write_batch(batch)
                self.consumer.commit(asynchronous=False)
                batch = []
                last_flush = time.time()

    def _write_batch(self, batch: List[dict]):
        """批量写入 ClickHouse"""
        if not batch:
            return

        # 构建INSERT语句
        values = []
        for m in batch:
            values.append(
                f"('{m['minute_start']}', '{m['edge_node']}', '{m['content_type']}', "
                f"{m['request_count']}, {m['bandwidth_bytes']}, "
                f"{m['avg_latency_ms']:.2f}, {m['p90_latency_ms']:.2f}, "
                f"{m['p99_latency_ms']:.2f}, {m['cache_hit_rate']:.4f}, "
                f"{m['error_rate']:.4f})"
            )

        sql = f"""
            INSERT INTO cdn_metrics_1min
            (minute_start, edge_node, content_type, request_count, bandwidth_bytes,
             avg_latency_ms, p90_latency_ms, p99_latency_ms, cache_hit_rate, error_rate)
            VALUES {','.join(values)}
        """
        try:
            self.ch.execute(sql)
            logging.info(f"写入 ClickHouse: {len(batch)} 行")
        except Exception as e:
            logging.error(f"ClickHouse 写入失败: {e}")


# ==================== Grafana 仪表盘配置生成 ====================

class GrafanaDashboardBuilder:
    """
    Grafana 仪表盘配置生成器
    自动生成 CDN 监控仪表盘的 JSON 配置
    """

    def build_cdn_dashboard(self) -> dict:
        """生成 CDN 全局监控仪表盘"""
        dashboard = {
            "dashboard": {
                "title": "CDN 全球监控仪表盘",
                "tags": ["cdn", "edge", "realtime"],
                "timezone": "browser",
                "refresh": "30s",
                "panels": [
                    # 第 1 行：全局概览
                    self._stat_panel("全球请求总量", "SELECT SUM(request_count) FROM cdn_metrics_1min WHERE minute_start > now() - INTERVAL 5 MINUTE"),
                    self._gauge_panel("全局缓存命中率", "SELECT AVG(cache_hit_rate) * 100 FROM cdn_metrics_1min WHERE minute_start > now() - INTERVAL 5 MINUTE", unit="%", thresholds=[70, 85]),
                    self._gauge_panel("全局 P90 延迟", "SELECT AVG(p90_latency_ms) FROM cdn_metrics_1min WHERE minute_start > now() - INTERVAL 5 MINUTE", unit="ms", thresholds=[100, 200]),
                    self._gauge_panel("全局错误率", "SELECT AVG(error_rate) * 100 FROM cdn_metrics_1min WHERE minute_start > now() - INTERVAL 5 MINUTE", unit="%", thresholds=[0.5, 1.0]),

                    # 第 2 行：各边缘节点缓存命中率
                    self._timeseries_panel(
                        "各节点缓存命中率",
                        "SELECT minute_start, edge_node, AVG(cache_hit_rate) * 100 "
                        "FROM cdn_metrics_1min WHERE minute_start > now() - INTERVAL 1 HOUR "
                        "GROUP BY minute_start, edge_node ORDER BY minute_start",
                        legend_format="{{edge_node}}"
                    ),

                    # 第 3 行：P90 延迟分布
                    self._timeseries_panel(
                        "各节点 P90 延迟",
                        "SELECT minute_start, edge_node, AVG(p90_latency_ms) "
                        "FROM cdn_metrics_1min WHERE minute_start > now() - INTERVAL 1 HOUR "
                        "GROUP BY minute_start, edge_node ORDER BY minute_start",
                        legend_format="{{edge_node}}",
                        unit="ms"
                    ),

                    # 第 4 行：带宽消耗
                    self._timeseries_panel(
                        "全球带宽消耗",
                        "SELECT minute_start, SUM(bandwidth_bytes) / 1000000 "
                        "FROM cdn_metrics_1min WHERE minute_start > now() - INTERVAL 6 HOUR "
                        "GROUP BY minute_start ORDER BY minute_start",
                        unit="MB"
                    ),

                    # 第 5 行：按内容类型分组的请求量
                    self._pie_panel(
                        "按内容类型请求分布",
                        "SELECT content_type, SUM(request_count) "
                        "FROM cdn_metrics_1min WHERE minute_start > now() - INTERVAL 1 HOUR "
                        "GROUP BY content_type"
                    ),
                ],
            },
            "overwrite": True,
        }
        return dashboard

    def _stat_panel(self, title: str, query: str) -> dict:
        return {
            "type": "stat", "title": title,
            "datasource": "ClickHouse",
            "targets": [{"rawSql": query}],
            "gridPos": {"h": 4, "w": 6, "x": 0, "y": 0},
        }

    def _gauge_panel(self, title: str, query: str, unit: str = "",
                     thresholds: list = None) -> dict:
        return {
            "type": "gauge", "title": title,
            "datasource": "ClickHouse",
            "targets": [{"rawSql": query}],
            "fieldConfig": {
                "defaults": {
                    "unit": unit,
                    "thresholds": {
                        "mode": "absolute",
                        "steps": [
                            {"value": None, "color": "green"},
                            {"value": thresholds[0] if thresholds else 0, "color": "yellow"},
                            {"value": thresholds[1] if thresholds else 0, "color": "red"},
                        ]
                    }
                }
            },
            "gridPos": {"h": 4, "w": 6, "x": 0, "y": 0},
        }

    def _timeseries_panel(self, title: str, query: str,
                          legend_format: str = "", unit: str = "") -> dict:
        return {
            "type": "timeseries", "title": title,
            "datasource": "ClickHouse",
            "targets": [{"rawSql": query, "legendFormat": legend_format}],
            "fieldConfig": {"defaults": {"unit": unit}},
            "gridPos": {"h": 8, "w": 24, "x": 0, "y": 0},
        }

    def _pie_panel(self, title: str, query: str) -> dict:
        return {
            "type": "piechart", "title": title,
            "datasource": "ClickHouse",
            "targets": [{"rawSql": query}],
            "gridPos": {"h": 8, "w": 8, "x": 0, "y": 0},
        }
```

**ClickHouse 聚合指标表：**

```sql
CREATE TABLE cdn_metrics_1min (
    minute_start     DateTime NOT NULL COMMENT '分钟起始时间',
    edge_node        LowCardinality(String) NOT NULL COMMENT '边缘节点编码',
    content_type     LowCardinality(String) NOT NULL COMMENT '内容类型',
    request_count    UInt64 NOT NULL COMMENT '请求数',
    bandwidth_bytes  UInt64 NOT NULL COMMENT '带宽(字节)',
    avg_latency_ms   Float32 NOT NULL COMMENT '平均延迟(ms)',
    p90_latency_ms   Float32 NOT NULL COMMENT 'P90延迟(ms)',
    p99_latency_ms   Float32 NOT NULL COMMENT 'P99延迟(ms)',
    cache_hit_rate   Float32 NOT NULL COMMENT '缓存命中率(0-1)',
    error_rate       Float32 NOT NULL COMMENT '错误率(0-1)'
) ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(minute_start)
ORDER BY (edge_node, content_type, minute_start)
TTL minute_start + INTERVAL 90 DAY;

-- 5 分钟聚合视图（用于 Grafana 长期趋势图）
CREATE MATERIALIZED VIEW cdn_metrics_5min
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(minute_start)
ORDER BY (edge_node, minute_start)
AS SELECT
    toStartOfFiveMinute(minute_start) AS minute_start,
    edge_node,
    SUM(request_count) AS request_count,
    SUM(bandwidth_bytes) AS bandwidth_bytes,
    AVG(avg_latency_ms) AS avg_latency_ms,
    MAX(p90_latency_ms) AS p90_latency_ms,
    MAX(p99_latency_ms) AS p99_latency_ms,
    AVG(cache_hit_rate) AS cache_hit_rate,
    AVG(error_rate) AS error_rate
FROM cdn_metrics_1min
GROUP BY minute_start, edge_node;
```

**管道性能指标：**

| 组件 | 吞吐量 | 延迟 | 存储 |
|------|--------|------|------|
| Fluentd 采集 | 10万条/秒/节点 | <100ms | 内存缓冲 |
| Kafka 传输 | 100万条/秒(3 Broker) | <10ms | 7天保留约 14GB |
| Flink 聚合 | 50万条/秒 | 1分钟窗口 | 内存 |
| ClickHouse 写入 | 20万行/秒 | <500ms(批量) | 90天保留约 50GB |
| Grafana 查询 | 50 QPS | <2秒 | — |
| 端到端延迟 | — | 2-3分钟(含1分钟窗口) | — |

## 异常场景补充

### 场景：TLS 握手失败

```
触发：边缘节点 SSL 证书更新后未正确加载 → 客户端 TLS 握手失败
      → 大量用户无法访问 → 5xx 错误率飙升至 30%
时间线：
  T+0     自动化证书续签完成，新证书写入磁盘
  T+5s    Nginx 执行 reload 命令
  T+10s   客户端开始报告 SSL_ERROR_HANDSHAKE_FAILURE
  T+30s   监控检测到 TLS 错误率 > 5%
  T+60s   自动回滚：加载上一版本证书 + Nginx reload
  T+70s   TLS 错误率恢复到 0%
根因：证书文件写入不完整（磁盘空间不足导致截断），Nginx 加载了损坏的证书
```

```python
class TLSHandshakeFailureHandler:
    """TLS 握手失败处理"""
    
    def on_tls_error_spike(self, node_id: str, error_rate: float):
        """TLS 错误率飙升时自动回滚证书"""
        if error_rate > 0.05:  # 错误率 > 5%
            # 1. 检查最近是否有证书更新操作
            recent_update = self.db.query_one(
                "SELECT * FROM ssl_certificate_deploy_log "
                "WHERE node_id = %s AND deployed_at > DATE_SUB(NOW(), INTERVAL 10 MINUTE) "
                "ORDER BY deployed_at DESC LIMIT 1", node_id
            )
            if recent_update:
                # 2. 证书更新导致的 → 立即回滚
                self.log(f"TLS 错误率 {error_rate:.1%}，检测到证书刚更新，自动回滚")
                prev_cert = self._get_previous_certificate(node_id)
                if prev_cert:
                    self._rollback_certificate(node_id, prev_cert)
                    # 3. 回滚后验证
                    if self._verify_tls(node_id):
                        self.alert(f"节点 {node_id} 证书回滚成功，TLS 恢复正常")
                    else:
                        self.alert_critical(
                            f"节点 {node_id} 证书回滚后 TLS 仍异常，需人工介入"
                        )
                        self._drain_node(node_id)
    
    def _verify_tls(self, node_id: str) -> bool:
        """验证节点 TLS 握手是否正常"""
        try:
            import ssl
            import socket
            node_ip = self._get_node_ip(node_id)
            context = ssl.create_default_context()
            with socket.create_connection((node_ip, 443), timeout=5) as sock:
                with context.wrap_socket(sock, server_hostname="cdn.example.com") as ssock:
                    return ssock.version() in ("TLSv1.2", "TLSv1.3")
        except Exception:
            return False
    
    def _rollback_certificate(self, node_id: str, cert_data: dict):
        """回滚证书到上一版本"""
        # 原子写入：先写临时文件，再 rename
        cert_path = f"/etc/nginx/ssl/{cert_data['domain']}.pem"
        temp_path = f"/tmp/ssl_rollback_{cert_data['domain']}.pem"
        with open(temp_path, "w") as f:
            f.write(cert_data["certificate"])
        os.rename(temp_path, cert_path)
        # 重载 Nginx
        self._execute_on_node(node_id, "nginx -s reload")
        self.db.insert("ssl_certificate_deploy_log", {
            "node_id": node_id, "action": "rollback",
            "certificate_hash": cert_data["hash"],
            "deployed_at": datetime.utcnow()
        })
```

### 场景：上游连接池耗尽

```
触发：边缘节点回源连接池满 → 新回源请求无法建立连接 → 504 Gateway Timeout
      → 用户请求全部超时
检测：
  1. Nginx upstream 连接数 > 连接池上限（如 1000）
  2. 504 状态码比例 > 5% → 告警
  3. 连接等待队列 > 100 → 警告
根因：
  1. 源站响应变慢 → 连接长时间占用 → 连接池耗尽
  2. 回源请求暴增（缓存失效）→ 连接需求超过池容量
  3. 连接泄漏（请求未正确关闭连接）→ 可用连接逐渐减少
```

```python
class UpstreamConnectionPoolExhaustionHandler:
    """上游连接池耗尽处理"""
    
    def on_connection_pool_exhausted(self, node_id: str):
        """连接池耗尽时的紧急处理"""
        pool_stats = self._get_connection_pool_stats(node_id)
        
        # 1. 如果是源站慢导致的 → 启用请求超时保护
        if pool_stats["avg_upstream_response_time_ms"] > 5000:
            self.log("源站响应过慢，降低超时时间以释放连接")
            self._update_upstream_timeout(node_id, timeout_ms=3000)
            # 清除等待时间过长的连接
            self._kill_stale_connections(node_id, idle_threshold_ms=5000)
        
        # 2. 如果是连接泄漏 → 强制回收空闲连接
        idle_connections = pool_stats.get("idle_connections", 0)
        if idle_connections > pool_stats["total_connections"] * 0.3:
            self.log(f"检测到 {idle_connections} 个空闲连接可能泄漏，强制回收")
            self._kill_stale_connections(node_id, idle_threshold_ms=1000)
        
        # 3. 如果是回源量暴增 → 启用回源限流
        if pool_stats["active_connections"] >= pool_stats["max_connections"] * 0.9:
            self.log("回源量暴增，启用回源限流保护")
            self._enable_origin_rate_limit(node_id, max_qps=5000)
            # 返回 stale 缓存，减少回源需求
            self._enable_stale_while_revalidate(node_id)
        
        # 4. 临时扩大连接池（如果有余量）
        if pool_stats["max_connections"] < 2000:
            self._increase_pool_size(node_id, new_max=2000)
            self.log(f"连接池临时扩容: {pool_stats['max_connections']} → 2000")
        
        # 5. 持续 5 分钟未恢复 → 节点降级
        self.schedule_check(
            delay_seconds=300,
            fn=lambda: self._check_and_drain_if_needed(node_id)
        )
    
    def _kill_stale_connections(self, node_id: str, idle_threshold_ms: int):
        """强制回收超过阈值的空闲连接"""
        # 通过 Nginx API 或内核参数调整
        self._execute_on_node(
            node_id,
            f"nginx -s reload && "
            f"echo 'upstream_timeout 3000ms' > /etc/nginx/conf.d/timeout.conf"
        )
    
    def _enable_stale_while_revalidate(self, node_id: str):
        """启用 stale-while-revalidate 模式"""
        self._execute_on_node(
            node_id,
            "echo 'proxy_cache_use_stale updating timeout error;' >> "
            "/etc/nginx/conf.d/cache.conf && nginx -s reload"
        )
```

### 场景：CDN 配置传播延迟

```
触发：控制面修改了缓存规则 TTL（从 300 秒改为 60 秒）
      → 但配置推送到 50 个边缘节点耗时 5 分钟
      → 期间部分节点仍使用旧 TTL → 数据不一致
检测：
  1. 配置版本号：全局配置版本 v12，但部分节点仍在 v11
  2. 配置传播完成率 < 100% → 告警
  3. 配置传播耗时 > 60 秒 → 警告
影响：
  - 部分节点返回过期缓存（旧 TTL 300 秒）
  - 部分节点已应用新 TTL 60 秒
  - 用户可能看到不一致的内容
```

```python
class ConfigPropagationMonitor:
    """CDN 配置传播延迟监控与处理"""
    
    CONFIG_VERSION_CHECK_INTERVAL = 10  # 每 10 秒检查一次配置版本
    
    def push_config(self, config_version: str, config_data: dict):
        """推送新配置到所有边缘节点"""
        # 1. 记录全局配置版本
        self.redis.set("global:config:version", config_version)
        self.redis.set(f"global:config:data:{config_version}", json.dumps(config_data))
        
        # 2. 并行推送到所有节点
        target_nodes = self.edge_registry.get_active_nodes()
        push_results = {}
        
        with ThreadPoolExecutor(max_workers=20) as executor:
            futures = {
                executor.submit(self._push_to_node, node, config_version, config_data): node
                for node in target_nodes
            }
            for future in as_completed(futures, timeout=30):
                node = futures[future]
                try:
                    success = future.result()
                    push_results[node.id] = "success" if success else "failed"
                except TimeoutError:
                    push_results[node.id] = "timeout"
                except Exception as e:
                    push_results[node.id] = f"error: {e}"
        
        # 3. 检查推送完成率
        success_count = sum(1 for v in push_results.values() if v == "success")
        total = len(target_nodes)
        completion_rate = success_count / total
        
        if completion_rate < 1.0:
            failed_nodes = [nid for nid, r in push_results.items() if r != "success"]
            self.alert(
                f"配置推送完成率 {completion_rate:.1%}，"
                f"失败节点: {failed_nodes}，配置版本: {config_version}"
            )
            # 重试失败节点
            for nid in failed_nodes:
                node = self.edge_registry.get_node(nid)
                if node:
                    self._retry_push(node, config_version, config_data)
        
        # 4. 持续监控直到所有节点确认
        self._monitor_propagation(config_version, target_nodes)
    
    def _push_to_node(self, node, config_version: str, config_data: dict) -> bool:
        """向单个节点推送配置"""
        try:
            resp = self.http_post(
                f"{node.api_endpoint}/api/config/update",
                json={
                    "version": config_version,
                    "config": config_data,
                    "checksum": hashlib.md5(json.dumps(config_data, sort_keys=True).encode()).hexdigest(),
                },
                timeout=10,
            )
            if resp.status_code == 200:
                # 节点确认配置已应用
                node_reported_version = resp.json().get("applied_version")
                return node_reported_version == config_version
            return False
        except Exception:
            return False
    
    def _monitor_propagation(self, config_version: str, target_nodes: list):
        """持续监控配置传播状态，直到所有节点确认"""
        start_time = time.time()
        max_wait = 120  # 最多等待 120 秒
        
        while time.time() - start_time < max_wait:
            # 检查每个节点当前运行的配置版本
            pending = []
            for node in target_nodes:
                current_version = self.redis.get(f"edge:config:version:{node.id}")
                if current_version != config_version:
                    pending.append(node.id)
            
            if not pending:
                elapsed = time.time() - start_time
                self.log(f"配置传播完成: {config_version}, 耗时 {elapsed:.1f} 秒")
                return
            
            time.sleep(self.CONFIG_VERSION_CHECK_INTERVAL)
        
        # 超时：仍有节点未更新
        self.alert_warning(
            f"配置传播超时: 版本 {config_version}, "
            f"未更新节点: {pending}, 已等待 {max_wait} 秒"
        )
```

## CDN 源站防护完整实现

```python
class OriginShieldManager:
    """CDN 源站防护：Shield 部署 + 缓存策略 + 故障降级"""

    SHIELD_REGIONS = {
        "asia": ["tokyo", "singapore", "hongkong"],
        "europe": ["frankfurt", "london", "amsterdam"],
        "americas": ["virginia", "oregon", "saopaulo"],
    }

    def configure_shield(self, origin_url, primary_region="asia"):
        """配置源站 Shield"""
        shield_config = {
            "origin_url": origin_url,
            "primary_shield": f"shield-{self.SHIELD_REGIONS[primary_region][0]}",
            "regional_shields": [],
        }

        # 每个区域部署一个 Shield
        for region, locations in self.SHIELD_REGIONS.items():
            shield_location = locations[0]
            self.db.insert("origin_shields", {
                "shield_id": f"shield-{shield_location}",
                "origin_url": origin_url,
                "region": region,
                "location": shield_location,
                "cache_key_pattern": "url+vary_headers",
                "cache_ttl_seconds": 86400,  # 24 小时
                "stale_while_revalidate": 300,  # 5 分钟
                "status": "active",
                "created_at": now()
            })
            shield_config["regional_shields"].append(f"shield-{shield_location}")

        return shield_config

    def purge_shield(self, shield_id, urls=None):
        """清除 Shield 缓存"""
        if urls:
            # 选择性清除
            for url in urls:
                cache_key = self._generate_cache_key(url)
                self.redis.delete(f"shield_cache:{shield_id}:{cache_key}")
        else:
            # 全量清除
            pattern = f"shield_cache:{shield_id}:*"
            keys = self.redis.keys(pattern)
            if keys:
                self.redis.delete(*keys)

        self.db.insert("shield_purge_log", {
            "shield_id": shield_id,
            "urls": json.dumps(urls) if urls else "all",
            "purged_at": now()
        })

    def handle_shield_failure(self, shield_id):
        """Shield 故障降级"""
        shield = self.db.get_shield(shield_id)

        # 1. 标记 Shield 不可用
        self.db.update("origin_shields",
            {"status": "failed", "failed_at": now()},
            {"shield_id": shield_id})

        # 2. PoP 直接回源（绕过 Shield）
        self.cdn_config.update_origin(shield["origin_url"], {
            "shield": None,
            "origin_timeout": 10,  # 缩短超时
            "retry_count": 2
        })

        # 3. 通知运维
        self.alert(f"Shield {shield_id} 故障，已切换到直接回源模式")
```

## CDN 流量调度

```python
class CDNTrafficManager:
    """CDN 流量调度：地理 + ISP + 维护 + 紧急"""

    def shift_traffic_for_maintenance(self, pop_id, drain_percentage=100):
        """维护前流量排空"""
        pop = self.db.get_pop(pop_id)

        # 1. 找到最近的备用 PoP
        nearby_pops = self.db.query(
            "SELECT * FROM cdn_pops WHERE id != %s "
            "ORDER BY ST_Distance(location, "
            "(SELECT location FROM cdn_pops WHERE id = %s)) ASC LIMIT 3",
            pop_id, pop_id)

        # 2. 逐步排空流量
        stages = [25, 50, 75, 100] if drain_percentage == 100 else [drain_percentage]
        for pct in stages:
            # 更新 DNS 权重
            self.dns_service.update_weights({
                pop_id: max(0, 100 - pct),
                nearby_pops[0]["id"]: pct
            })
            # 等待 DNS 传播
            time.sleep(30)

        # 3. 验证流量已排空
        current_rps = self.monitoring.get_pop_rps(pop_id)
        if current_rps > 100:  # 还有流量
            self.alert(f"PoP {pop_id} 仍有 {current_rps} RPS，排空不完全")

        return {"drained": current_rps < 100}

    def emergency_reroute(self, failed_pop_id):
        """紧急流量重路由"""
        # 获取失败 PoP 的流量
        affected_regions = self.db.query(
            "SELECT region FROM pop_regions WHERE pop_id = %s",
            failed_pop_id)

        # 找到备用 PoP
        for region in affected_regions:
            backup = self.db.query_one(
                "SELECT * FROM cdn_pops WHERE region = %s "
                "AND id != %s AND status = 'healthy' "
                "ORDER BY available_capacity DESC LIMIT 1",
                region["region"], failed_pop_id)

            if backup:
                # 切换 DNS
                self.dns_service.update_record(
                    region["region"], backup["ip_address"])
            else:
                # 无备用 → 回源
                self.dns_service.update_record(
                    region["region"], self.origin_ip)
                self.alert(f"区域 {region['region']} 无备用 PoP，已切换回源")

        self.alert(f"PoP {failed_pop_id} 故障，已紧急重路由")
```

## 异常场景补充

### 场景：源站 Shield 缓存投毒

```
触发：攻击者向 Shield 注入恶意缓存 → 所有用户收到被篡改的内容
检测：
  1. Shield 缓存内容与源站不一致 → 可能被投毒
  2. 用户投诉内容异常 → 缓存投毒
处理：
  1. 立即全量清除 Shield 缓存
  2. 检查缓存键是否可被外部操纵
  3. 增加 Vary 头限制（只缓存 Accept-Encoding 差异）
预防：缓存键严格设计 + 响应头校验 + 定期缓存验证
```

### 场景：DDoS 防护误杀

```
触发：CDN DDoS 防护将正常用户判定为攻击 → 显示 Challenge 页面
检测：
  1. Challenge 通过率异常低 → 误杀
  2. 用户投诉"无法访问" → DDoS 防护误判
处理：
  1. 放宽速率限制阈值
  2. 将误杀 IP 加入白名单
  3. Challenge 页面增加"我不是机器人"选项
预防：阈值动态调整 + 白名单机制 + 误杀率监控
```

## CDN 日志分析流水线完整实现

```python
class CDNLogAnalyticsPipeline:
    """CDN 日志分析：摄入 → 解析 → 聚合 → 异常检测"""

    def ingest_log_batch(self, pop_id, log_entries):
        """批量摄入 CDN 日志"""
        for entry in log_entries:
            parsed = self._parse_log_entry(entry)
            parsed["pop_id"] = pop_id

            # 写入 Kafka
            self.kafka.produce("cdn_logs", json.dumps(parsed))

    def _parse_log_entry(self, raw):
        """解析 CDN 日志"""
        # 标准格式：timestamp | client_ip | method | url | status | bytes | hit_miss | response_time_ms | user_agent | referer
        parts = raw.split("|")
        return {
            "timestamp": parts[0].strip(),
            "client_ip": parts[1].strip(),
            "method": parts[2].strip(),
            "url": parts[3].strip(),
            "status": int(parts[4].strip()),
            "bytes": int(parts[5].strip()),
            "cache_status": parts[6].strip(),  # HIT / MISS / EXPIRED / STALE
            "response_time_ms": int(parts[7].strip()),
            "content_type": self._extract_content_type(parts[3].strip()),
        }

    def aggregate_realtime(self, minutes=5):
        """实时聚合指标"""
        return {
            "qps_by_pop": self._qps_by_pop(minutes),
            "hit_rate_by_content_type": self._hit_rate_by_content(minutes),
            "error_rate": self._error_rate(minutes),
            "top_urls_by_bandwidth": self._top_urls_bandwidth(minutes),
            "top_error_urls": self._top_error_urls(minutes),
            "geographic_distribution": self._geo_distribution(minutes),
        }

    def _qps_by_pop(self, minutes):
        """按 PoP 统计 QPS"""
        return self.db.query(
            "SELECT pop_id, COUNT(*) / (%s * 60) as qps "
            "FROM cdn_log_parsed WHERE timestamp > NOW() - INTERVAL %s MINUTE "
            "GROUP BY pop_id ORDER BY qps DESC", minutes, minutes)

    def _hit_rate_by_content(self, minutes):
        """按内容类型统计命中率"""
        return self.db.query(
            "SELECT content_type, "
            "SUM(CASE WHEN cache_status = 'HIT' THEN 1 ELSE 0 END) / COUNT(*) as hit_rate "
            "FROM cdn_log_parsed WHERE timestamp > NOW() - INTERVAL %s MINUTE "
            "GROUP BY content_type ORDER BY hit_rate", minutes)

    def _top_urls_bandwidth(self, minutes):
        """带宽消耗 Top URL"""
        return self.db.query(
            "SELECT url, SUM(bytes) as total_bytes, COUNT(*) as requests "
            "FROM cdn_log_parsed WHERE timestamp > NOW() - INTERVAL %s MINUTE "
            "GROUP BY url ORDER BY total_bytes DESC LIMIT 20", minutes)

    def detect_anomalies(self):
        """异常检测"""
        anomalies = []

        # QPS 突增 (> 3σ)
        current_qps = self._current_qps()
        baseline = self._qps_baseline()  # 同周同时段基线
        if current_qps > baseline["mean"] + 3 * baseline["std"]:
            anomalies.append({
                "type": "qps_spike",
                "current": current_qps,
                "baseline_mean": baseline["mean"],
                "severity": "high"
            })

        # 命中率骤降
        current_hit_rate = self._current_hit_rate()
        if current_hit_rate < baseline["hit_rate"] * 0.8:
            anomalies.append({
                "type": "hit_rate_drop",
                "current": current_hit_rate,
                "baseline": baseline["hit_rate"],
                "severity": "medium"
            })

        return anomalies
```

## CDN 成本优化引擎

```python
class CDNCostOptimizer:
    """CDN 成本优化：成本模型 + 缓存策略 + 预算告警"""

    COST_MODEL = {
        "bandwidth_per_gb": 0.08,      # $/GB
        "request_per_million": 0.0075,  # $/百万请求
        "origin_egress_per_gb": 0.09,   # 源站出网 $/GB
    }

    def calculate_monthly_cost(self, month):
        """计算月度 CDN 成本"""
        stats = self.db.query_one(
            "SELECT SUM(bytes) / 1073741824 as total_gb, "
            "COUNT(*) as total_requests, "
            "SUM(CASE WHEN cache_status != 'HIT' THEN bytes ELSE 0 END) / 1073741824 as origin_gb "
            "FROM cdn_log_parsed WHERE DATE_FORMAT(timestamp, '%%Y-%%m') = %s",
            month)

        bandwidth_cost = stats["total_gb"] * self.COST_MODEL["bandwidth_per_gb"]
        request_cost = stats["total_requests"] / 1000000 * self.COST_MODEL["request_per_million"]
        origin_cost = stats["origin_gb"] * self.COST_MODEL["origin_egress_per_gb"]

        hit_rate = 1 - (stats["origin_gb"] / stats["total_gb"]) if stats["total_gb"] else 0

        return {
            "month": month,
            "bandwidth_cost": round(bandwidth_cost, 2),
            "request_cost": round(request_cost, 2),
            "origin_cost": round(origin_cost, 2),
            "total_cost": round(bandwidth_cost + request_cost + origin_cost, 2),
            "hit_rate": round(hit_rate, 3),
            "origin_savings_if_100pct_hit": round(stats["origin_gb"] * self.COST_MODEL["origin_egress_per_gb"], 2)
        }

    def recommend_cache_policy(self):
        """推荐缓存策略优化"""
        recommendations = []

        # 低命中率内容 → 增加缓存 TTL
        low_hit_content = self.db.query(
            "SELECT content_type, "
            "SUM(CASE WHEN cache_status = 'HIT' THEN 1 ELSE 0 END) / COUNT(*) as hit_rate, "
            "AVG(response_time_ms) as avg_latency "
            "FROM cdn_log_parsed WHERE timestamp > NOW() - INTERVAL 7 DAY "
            "GROUP BY content_type HAVING hit_rate < 0.8")

        for content in low_hit_content:
            recommendations.append({
                "action": "increase_ttl",
                "content_type": content["content_type"],
                "current_hit_rate": round(content["hit_rate"], 3),
                "recommended_ttl": 86400,  # 24h
                "estimated_improvement": f"+{round((0.9 - content['hit_rate']) * 100, 0)}% hit rate"
            })

        # 高带宽静态内容 → 预热
        high_bw_static = self.db.query(
            "SELECT url, SUM(bytes) as total_bytes "
            "FROM cdn_log_parsed WHERE timestamp > NOW() - INTERVAL 1 DAY "
            "AND cache_status = 'MISS' AND content_type = 'static' "
            "GROUP BY url ORDER BY total_bytes DESC LIMIT 10")

        for item in high_bw_static:
            recommendations.append({
                "action": "prewarm",
                "url": item["url"],
                "daily_miss_bytes_gb": round(item["total_bytes"] / 1073741824, 2),
                "estimated_saving": round(item["total_bytes"] / 1073741824 * self.COST_MODEL["origin_egress_per_gb"], 2)
            })

        return recommendations
```

## 异常场景补充

### 场景：日志管道 Kafka 延迟

```
触发：CDN 日志 Kafka 消费延迟 → 分析数据过时 → 决策偏差
检测：
  1. Consumer lag > 100000 → 告警
  2. 最新分析数据 > 15 分钟前 → 过时
处理：
  1. 增加消费者实例
  2. 降级：使用 Redis 实时计数器替代日志聚合
  3. 关键指标（QPS、命中率）双路计算
预防：消费者自动扩容 + 实时计数器兜底 + 双路计算
```

### 场景：成本优化误降动态内容 TTL

```
触发：优化引擎将动态 API 响应 TTL 从 0 调到 60s → 返回过期数据
检测：
  1. API 返回过期数据 → 用户投诉
  2. 动态内容命中率异常上升 → 可能有缓存
处理：
  1. 立即恢复动态内容 TTL=0
  2. 优化引擎增加内容分类规则（no-cache / immutable / dynamic）
  3. 动态内容禁止缓存策略调整
预防：内容分类标签 + 动态内容保护策略 + 变更审批
```

## CDN 日志分析管道（完整版）

### 日志采集与解析

```python
import json
import re
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from typing import List, Dict, Optional, Tuple
from enum import Enum
from collections import defaultdict
import statistics


class CacheStatus(Enum):
    HIT = "HIT"            # 边缘缓存命中
    MISS = "MISS"          # 边缘未命中，回源
    EXPIRED = "EXPIRED"    # 缓存已过期，需刷新
    STALE = "STALE"        # 返回过期缓存，后台刷新
    ORIGIN = "ORIGIN"      # 不可缓存内容，直接回源


@dataclass
class CDNLogEntry:
    """CDN 日志条目"""
    timestamp: datetime
    pop_id: str                    # 边缘节点 ID
    client_ip: str
    method: str                    # GET/POST
    url: str
    status_code: int
    cache_status: CacheStatus
    response_time_ms: float        # 响应时间（毫秒）
    bytes_sent: int                # 响应大小
    content_type: str              # text/html, video/mp4 等
    origin_response_time_ms: float = 0  # 回源响应时间
    edge_location: str = ""        # 边缘节点地理位置


class CDNLogParser:
    """CDN 日志解析器"""

    # CDN 日志格式示例：
    # 2026-05-15T10:30:45Z pop-sfo-01 192.168.1.100 GET
    #   /video/xyz.mp4 200 HIT 45 12345678 video/mp4 0
    LOG_PATTERN = re.compile(
        r'(\S+)\s+(\S+)\s+(\S+)\s+(\S+)\s+(\S+)\s+'
        r'(\d+)\s+(\S+)\s+([\d.]+)\s+(\d+)\s+(\S+)\s+([\d.]+)'
    )

    def parse_line(self, line: str) -> Optional[CDNLogEntry]:
        """解析单行日志"""
        match = self.LOG_PATTERN.match(line.strip())
        if not match:
            return None

        groups = match.groups()
        try:
            return CDNLogEntry(
                timestamp=datetime.fromisoformat(
                    groups[0].replace('Z', '+00:00')),
                pop_id=groups[1],
                client_ip=groups[2],
                method=groups[3],
                url=groups[4],
                status_code=int(groups[5]),
                cache_status=CacheStatus(groups[6]),
                response_time_ms=float(groups[7]),
                bytes_sent=int(groups[8]),
                content_type=groups[9],
                origin_response_time_ms=float(groups[10])
            )
        except (ValueError, KeyError):
            return None


class CDNLogAggregator:
    """
    CDN 日志实时聚合
    从 Kafka 消费日志，按维度聚合指标
    """

    def __init__(self, kafka_consumer, redis_client, db_pool):
        self.kafka = kafka_consumer
        self.redis = redis_client
        self.db = db_pool
        self.parser = CDNLogParser()

    async def start_consuming(self, topic: str = "cdn-logs"):
        """启动日志消费循环"""
        self.kafka.subscribe([topic])

        buffer = []
        flush_interval = timedelta(seconds=10)

        async for message in self.kafka.consume():
            entry = self.parser.parse_line(message.value.decode())
            if entry:
                buffer.append(entry)

                if datetime.utcnow() - buffer[0].timestamp >= flush_interval:
                    await self._flush_aggregation(buffer)
                    buffer = []

    async def _flush_aggregation(self, entries: List[CDNLogEntry]):
        """刷新聚合数据到 Redis 和 DB"""
        # 按维度聚合
        qps_by_pop = defaultdict(int)
        hit_count_by_content = defaultdict(lambda: {"hit": 0, "total": 0})
        error_count_by_pop = defaultdict(int)
        bandwidth_by_url = defaultdict(int)
        error_urls = defaultdict(int)

        for entry in entries:
            # QPS per PoP
            qps_by_pop[entry.pop_id] += 1

            # 命中率 per 内容类型
            ct = entry.content_type
            hit_count_by_content[ct]["total"] += 1
            if entry.cache_status in (CacheStatus.HIT, CacheStatus.STALE):
                hit_count_by_content[ct]["hit"] += 1

            # 错误率 per PoP
            if entry.status_code >= 500:
                error_count_by_pop[entry.pop_id] += 1

            # 带宽 per URL
            bandwidth_by_url[entry.url] += entry.bytes_sent

            # 错误 URL
            if entry.status_code >= 400:
                error_urls[entry.url] += 1

        # 写入 Redis 实时指标
        now = datetime.utcnow()
        minute_key = now.strftime("%Y%m%d%H%M")

        for pop_id, count in qps_by_pop.items():
            self.redis.incrby(f"cdn:qps:{pop_id}:{minute_key}", count)

        for ct, data in hit_count_by_content.items():
            self.redis.incrby(
                f"cdn:hit:{ct}:{minute_key}", data["hit"])
            self.redis.incrby(
                f"cdn:total:{ct}:{minute_key}", data["total"])

        for pop_id, count in error_count_by_pop.items():
            self.redis.incrby(
                f"cdn:error:{pop_id}:{minute_key}", count)

        # 写入 DB 用于长期分析
        await self._persist_aggregation(entries, now)

    async def _persist_aggregation(self, entries: List[CDNLogEntry],
                                    ts: datetime):
        """持久化聚合数据"""
        # 按 PoP 分组
        pop_groups = defaultdict(list)
        for e in entries:
            pop_groups[e.pop_id].append(e)

        for pop_id, pop_entries in pop_groups.items():
            pop_hits = sum(1 for e in pop_entries
                          if e.cache_status in
                          (CacheStatus.HIT, CacheStatus.STALE))
            pop_total = len(pop_entries)
            pop_errors = sum(
                1 for e in pop_entries if e.status_code >= 500)
            pop_avg_rt = statistics.mean(
                [e.response_time_ms for e in pop_entries])
            pop_bytes = sum(e.bytes_sent for e in pop_entries)

            await self.db.execute(
                """INSERT INTO cdn_aggregation_minute
                   (ts, pop_id, total_requests, hit_count,
                    error_count, avg_response_time_ms, total_bytes)
                   VALUES (%s, %s, %s, %s, %s, %s, %s)
                   ON CONFLICT (ts, pop_id) DO UPDATE SET
                     total_requests = cdn_aggregation_minute.total_requests + EXCLUDED.total_requests,
                     hit_count = cdn_aggregation_minute.hit_count + EXCLUDED.hit_count,
                     error_count = cdn_aggregation_minute.error_count + EXCLUDED.error_count,
                     avg_response_time_ms = (cdn_aggregation_minute.avg_response_time_ms + EXCLUDED.avg_response_time_ms) / 2,
                     total_bytes = cdn_aggregation_minute.total_bytes + EXCLUDED.total_bytes""",
                ts, pop_id, pop_total, pop_hits,
                pop_errors, pop_avg_rt, pop_bytes
            )


class CDNAnomalyDetector:
    """
    CDN 异常检测
    - QPS 突增 > 3σ
    - 命中率骤降 > 20%
    """

    def __init__(self, redis_client, db_pool, alert_service):
        self.redis = redis_client
        self.db = db_pool
        self.alert = alert_service

    async def detect_qps_spike(self, pop_id: str) -> Optional[Dict]:
        """检测 QPS 突增（> 3 标准差）"""
        # 获取过去 7 天同一小时的 QPS 历史数据
        now = datetime.utcnow()
        rows = await self.db.fetch_all(
            """SELECT AVG(total_requests) AS avg_qps,
                 STDDEV(total_requests) AS stddev_qps
               FROM cdn_aggregation_minute
               WHERE pop_id = %s
                 AND EXTRACT(HOUR FROM ts) = %s
                 AND ts > NOW() - INTERVAL '7 days'""",
            pop_id, now.hour
        )

        if not rows or rows[0]["avg_qps"] is None:
            return None

        avg = float(rows[0]["avg_qps"])
        stddev = float(rows[0]["stddev_qps"]) or 0
        threshold = avg + 3 * stddev

        # 当前 QPS
        minute_key = now.strftime("%Y%m%d%H%M")
        current = int(self.redis.get(
            f"cdn:qps:{pop_id}:{minute_key}") or 0)

        if current > threshold and stddev > 0:
            alert_data = {
                "pop_id": pop_id,
                "current_qps": current,
                "avg_qps": round(avg, 1),
                "stddev": round(stddev, 1),
                "threshold": round(threshold, 1),
                "spike_ratio": round(current / avg, 2) if avg > 0 else 0,
                "detected_at": now.isoformat()
            }
            await self.alert.send_warning(
                title=f"QPS 突增告警 - {pop_id}",
                message=(
                    f"当前 QPS: {current}\n"
                    f"历史均值: {avg:.1f} (σ={stddev:.1f})\n"
                    f"突增倍数: {current/avg:.1f}x\n"
                    f"阈值(3σ): {threshold:.1f}"
                )
            )
            return alert_data

        return None

    async def detect_hit_rate_drop(self, content_type: str) -> Optional[Dict]:
        """检测命中率骤降（> 20%）"""
        now = datetime.utcnow()
        minute_key = now.strftime("%Y%m%d%H%M")

        # 当前命中率
        current_hits = int(self.redis.get(
            f"cdn:hit:{content_type}:{minute_key}") or 0)
        current_total = int(self.redis.get(
            f"cdn:total:{content_type}:{minute_key}") or 0)

        if current_total < 100:
            return None  # 样本量太小

        current_rate = current_hits / current_total

        # 过去 1 小时平均命中率
        row = await self.db.fetch_one(
            """SELECT AVG(hit_count::numeric / NULLIF(total_requests, 0))
                 AS avg_hit_rate
               FROM cdn_aggregation_minute
               WHERE ts > NOW() - INTERVAL '1 hour'"""
        )

        if not row or row["avg_hit_rate"] is None:
            return None

        baseline_rate = float(row["avg_hit_rate"])
        drop_pct = (baseline_rate - current_rate) / baseline_rate * 100

        if drop_pct > 20:
            alert_data = {
                "content_type": content_type,
                "current_hit_rate": round(current_rate * 100, 2),
                "baseline_hit_rate": round(baseline_rate * 100, 2),
                "drop_pct": round(drop_pct, 1),
                "detected_at": now.isoformat()
            }
            await self.alert.send_warning(
                title=f"命中率骤降告警 - {content_type}",
                message=(
                    f"当前命中率: {current_rate*100:.1f}%\n"
                    f"基线命中率: {baseline_rate*100:.1f}%\n"
                    f"下降幅度: {drop_pct:.1f}%\n"
                    f"可能原因：缓存配置变更 / 源站内容更新 / 爬虫流量激增"
                )
            )
            return alert_data

        return None


class CDNReportGenerator:
    """
    CDN 报表生成
    小时报 / 日报
    """

    def __init__(self, db_pool):
        self.db = db_pool

    async def generate_hourly_report(self, hour: datetime) -> Dict:
        """生成小时报表"""
        start = hour.replace(minute=0, second=0, microsecond=0)
        end = start + timedelta(hours=1)

        # 汇总指标
        summary = await self.db.fetch_one(
            """SELECT
                 SUM(total_requests) AS total_requests,
                 SUM(hit_count) AS total_hits,
                 ROUND(SUM(hit_count)::numeric /
                       NULLIF(SUM(total_requests), 0) * 100, 2)
                   AS hit_rate_pct,
                 SUM(error_count) AS total_errors,
                 ROUND(SUM(error_count)::numeric /
                       NULLIF(SUM(total_requests), 0) * 100, 4)
                   AS error_rate_pct,
                 AVG(avg_response_time_ms) AS avg_rt_ms,
                 SUM(total_bytes) AS total_bytes
               FROM cdn_aggregation_minute
               WHERE ts BETWEEN %s AND %s""",
            start, end
        )

        # Top URL 按带宽
        top_urls_by_bandwidth = await self.db.fetch_all(
            """SELECT url, SUM(bytes_sent) AS total_bytes,
                 COUNT(*) AS request_count
               FROM cdn_access_logs
               WHERE timestamp BETWEEN %s AND %s
               GROUP BY url
               ORDER BY total_bytes DESC
               LIMIT 20""",
            start, end
        )

        # Top 错误 URL
        top_error_urls = await self.db.fetch_all(
            """SELECT url, status_code, COUNT(*) AS error_count
               FROM cdn_access_logs
               WHERE timestamp BETWEEN %s AND %s
                 AND status_code >= 400
               GROUP BY url, status_code
               ORDER BY error_count DESC
               LIMIT 20""",
            start, end
        )

        # 地理分布
        geo_distribution = await self.db.fetch_all(
            """SELECT edge_location,
                 COUNT(*) AS request_count,
                 SUM(bytes_sent) AS total_bytes
               FROM cdn_access_logs
               WHERE timestamp BETWEEN %s AND %s
               GROUP BY edge_location
               ORDER BY request_count DESC
               LIMIT 30""",
            start, end
        )

        return {
            "period": {"start": start.isoformat(),
                       "end": end.isoformat()},
            "summary": dict(summary) if summary else {},
            "top_urls_by_bandwidth": [dict(r) for r in top_urls_by_bandwidth],
            "top_error_urls": [dict(r) for r in top_error_urls],
            "geo_distribution": [dict(r) for r in geo_distribution]
        }

    async def generate_daily_report(self, date: datetime) -> Dict:
        """生成日报表"""
        start = date.replace(hour=0, minute=0, second=0, microsecond=0)
        end = start + timedelta(days=1)

        # 24 小时趋势
        hourly_trend = await self.db.fetch_all(
            """SELECT
                 time_bucket('1 hour', ts) AS hour,
                 SUM(total_requests) AS total_requests,
                 SUM(hit_count) AS total_hits,
                 ROUND(SUM(hit_count)::numeric /
                       NULLIF(SUM(total_requests), 0) * 100, 2)
                   AS hit_rate_pct,
                 AVG(avg_response_time_ms) AS avg_rt_ms
               FROM cdn_aggregation_minute
               WHERE ts BETWEEN %s AND %s
               GROUP BY hour
               ORDER BY hour""",
            start, end
        )

        return {
            "date": start.strftime("%Y-%m-%d"),
            "hourly_trend": [dict(r) for r in hourly_trend],
            "report_generated_at": datetime.utcnow().isoformat()
        }
```

### Kafka 消费与入库配置

```yaml
# Kafka 日志采集配置
kafka:
  bootstrap_servers: "kafka-1:9092,kafka-2:9092,kafka-3:9092"
  topic: "cdn-logs"
  consumer_group: "cdn-log-aggregator"
  auto_offset_reset: "latest"
  max_poll_records: 10000
  session_timeout_ms: 30000

# 每个边缘节点的日志采集 Agent
fluentbit:
  input:
    type: "tail"
    path: "/var/log/cdn/access.log"
    read_from_head: false
  output:
    type: "kafka"
    brokers: "kafka-1:9092,kafka-2:9092"
    topic: "cdn-logs"
    partition_key_field: "pop_id"
```

```sql
-- CDN 日志聚合表
CREATE TABLE cdn_aggregation_minute (
    ts                  TIMESTAMPTZ NOT NULL,
    pop_id              VARCHAR(32) NOT NULL,
    total_requests      INT NOT NULL DEFAULT 0,
    hit_count           INT NOT NULL DEFAULT 0,
    error_count         INT NOT NULL DEFAULT 0,
    avg_response_time_ms FLOAT NOT NULL DEFAULT 0,
    total_bytes         BIGINT NOT NULL DEFAULT 0,
    PRIMARY KEY (ts, pop_id)
);

-- CDN 访问日志（热数据保留 7 天，冷数据归档到对象存储）
CREATE TABLE cdn_access_logs (
    id                  BIGSERIAL,
    timestamp           TIMESTAMPTZ NOT NULL,
    pop_id              VARCHAR(32) NOT NULL,
    client_ip           INET,
    method              VARCHAR(8),
    url                 TEXT,
    status_code         INT,
    cache_status        VARCHAR(16),
    response_time_ms    FLOAT,
    bytes_sent          BIGINT,
    content_type        VARCHAR(64),
    edge_location       VARCHAR(64)
) PARTITION BY RANGE (timestamp);
```

## CDN 成本优化引擎（完整版）

### 成本模型与优化

```python
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from typing import List, Dict, Optional, Tuple
from enum import Enum
import statistics


class ContentCategory(Enum):
    STATIC_VIDEO = "static_video"       # 视频文件
    STATIC_IMAGE = "static_image"       # 图片
    STATIC_CSS_JS = "static_css_js"     # CSS/JS
    DYNAMIC_API = "dynamic_api"         # 动态 API
    LIVE_STREAM = "live_stream"         # 直播流


@dataclass
class CostModel:
    """
    CDN 成本模型
    三大成本项：带宽成本 + 请求成本 + 回源流出成本
    """
    # 带宽成本（$ per GB）
    bandwidth_cost_per_gb: float = 0.02
    # 请求成本（$ per 10K requests）
    request_cost_per_10k: float = 0.0075
    # 回源流出成本（$ per GB，比边缘带宽贵 3-5 倍）
    origin_egress_cost_per_gb: float = 0.08
    # Origin Shield 带宽成本（$ per GB，比直接回源便宜 50%）
    shield_cost_per_gb: float = 0.04

    def calculate_monthly_cost(self,
                                total_bandwidth_gb: float,
                                total_requests: int,
                                origin_egress_gb: float,
                                shield_egress_gb: float = 0) -> Dict:
        """计算月度成本"""
        bw_cost = total_bandwidth_gb * self.bandwidth_cost_per_gb
        req_cost = (total_requests / 10000) * self.request_cost_per_10k
        origin_cost = origin_egress_gb * self.origin_egress_cost_per_gb
        shield_cost = shield_egress_gb * self.shield_cost_per_gb

        total = bw_cost + req_cost + origin_cost + shield_cost

        return {
            "bandwidth_cost": round(bw_cost, 2),
            "request_cost": round(req_cost, 2),
            "origin_egress_cost": round(origin_cost, 2),
            "shield_cost": round(shield_cost, 2),
            "total_cost": round(total, 2),
            "breakdown": {
                "bandwidth_pct": round(bw_cost / total * 100, 1),
                "request_pct": round(req_cost / total * 100, 1),
                "origin_pct": round(origin_cost / total * 100, 1),
                "shield_pct": round(shield_cost / total * 100, 1),
            }
        }


@dataclass
class CachePolicy:
    """缓存策略"""
    content_type: ContentCategory
    ttl_seconds: int
    vary_headers: List[str] = field(default_factory=list)
    stale_while_revalidate: int = 0     # 后台刷新期间返回过期内容
    shield_enabled: bool = False


class CDNCacheOptimizer:
    """
    缓存策略优化器
    根据命中率数据动态调整 TTL
    """

    def __init__(self, db_pool, cdn_api_client):
        self.db = db_pool
        self.cdn = cdn_api_client

    async def calculate_origin_offload_ratio(self,
                                              start: datetime,
                                              end: datetime) -> Dict:
        """计算回源卸载率"""
        row = await self.db.fetch_one(
            """SELECT
                 SUM(total_requests) AS total,
                 SUM(hit_count) AS hits,
                 ROUND(SUM(hit_count)::numeric /
                       NULLIF(SUM(total_requests), 0) * 100, 2)
                   AS offload_ratio_pct
               FROM cdn_aggregation_minute
               WHERE ts BETWEEN %s AND %s""",
            start, end
        )
        return dict(row) if row else {}

    async def optimize_cache_policy(self) -> List[Dict]:
        """
        基于命中率数据优化缓存策略
        高命中内容 → 增加 TTL
        低命中内容 → 减少 TTL（避免缓存浪费）
        动态内容 → 永不增加 TTL
        """
        # 获取各内容类型的命中率
        rows = await self.db.fetch_all(
            """SELECT content_type,
                 COUNT(*) AS total,
                 COUNT(*) FILTER (
                   WHERE cache_status IN ('HIT', 'STALE')
                 ) AS hits,
                 ROUND(
                   COUNT(*) FILTER (
                     WHERE cache_status IN ('HIT', 'STALE')
                   )::numeric / NULLIF(COUNT(*), 0) * 100, 2
                 ) AS hit_rate_pct,
                 AVG(response_time_ms) AS avg_rt,
                 PERCENTILE_CONT(0.95) WITHIN GROUP (
                   ORDER BY response_time_ms
                 ) AS p95_rt
               FROM cdn_access_logs
               WHERE timestamp > NOW() - INTERVAL '7 days'
               GROUP BY content_type"""
        )

        # 动态内容保护白名单：永不增加 TTL
        PROTECTED_CONTENT_TYPES = {
            "dynamic_api", "application/json", "text/html"
        }

        recommendations = []
        for row in rows:
            ct = row["content_type"]
            hit_rate = float(row["hit_rate_pct"] or 0)

            # 动态内容跳过优化
            if ct in PROTECTED_CONTENT_TYPES:
                continue

            # 获取当前策略
            current_policy = await self._get_current_policy(ct)

            # 优化建议
            if hit_rate > 90 and current_policy["ttl_seconds"] < 86400:
                # 高命中 → 增加 TTL
                new_ttl = min(current_policy["ttl_seconds"] * 2, 604800)
                recommendations.append({
                    "content_type": ct,
                    "current_ttl": current_policy["ttl_seconds"],
                    "recommended_ttl": new_ttl,
                    "reason": f"命中率 {hit_rate}%> 90%，建议翻倍 TTL",
                    "expected_savings_pct": round(
                        (hit_rate - 90) / 10 * 5, 1),
                    "risk": "low"
                })
            elif hit_rate < 30 and current_policy["ttl_seconds"] > 300:
                # 低命中 → 减少 TTL（节省缓存空间）
                new_ttl = max(current_policy["ttl_seconds"] // 2, 60)
                recommendations.append({
                    "content_type": ct,
                    "current_ttl": current_policy["ttl_seconds"],
                    "recommended_ttl": new_ttl,
                    "reason": f"命中率 {hit_rate}%< 30%，建议减半 TTL",
                    "expected_savings_pct": 0,
                    "risk": "medium"
                })

        return recommendations

    async def generate_prewarm_recommendations(self) -> List[Dict]:
        """
        内容预热建议
        识别即将爆发的热点内容
        """
        # 过去 24 小时内请求量增长最快的 URL
        rows = await self.db.fetch_all(
            """WITH hourly AS (
                 SELECT url,
                   time_bucket('1 hour', timestamp) AS hour,
                   COUNT(*) AS requests
                 FROM cdn_access_logs
                 WHERE timestamp > NOW() - INTERVAL '24 hours'
                 GROUP BY url, hour
               )
               SELECT url,
                 SUM(requests) AS total_requests,
                 COUNT(DISTINCT hour) AS active_hours,
                 (MAX(requests) - MIN(requests))::numeric
                   / NULLIF(MIN(requests), 0) AS growth_rate
               FROM hourly
               GROUP BY url
               HAVING SUM(requests) > 1000
                 AND (MAX(requests) - MIN(requests))::numeric
                     / NULLIF(MIN(requests), 0) > 0.5
               ORDER BY growth_rate DESC
               LIMIT 50"""
        )

        recommendations = []
        for row in rows:
            recommendations.append({
                "url": row["url"],
                "total_requests_24h": row["total_requests"],
                "growth_rate": round(float(row["growth_rate"] or 0), 2),
                "action": "prewarm",
                "target_pops": "all",
                "priority": "high" if float(
                    row["growth_rate"] or 0) > 2 else "medium"
            })

        return recommendations

    async def calculate_shield_savings(self, start: datetime,
                                        end: datetime) -> Dict:
        """Origin Shield 节省计算"""
        # 估算：Shield 之前 vs 之后的回源流量
        row = await self.db.fetch_one(
            """SELECT
                 SUM(total_requests) AS total,
                 SUM(hit_count) AS edge_hits,
                 (SUM(total_requests) - SUM(hit_count)) AS origin_requests
               FROM cdn_aggregation_minute
               WHERE ts BETWEEN %s AND %s""",
            start, end
        )

        if not row:
            return {}

        origin_requests = int(row["origin_requests"] or 0)
        # Shield 命中率通常 70-90%
        shield_hit_rate = 0.80
        shield_saved_requests = int(origin_requests * shield_hit_rate)

        # 估算节省（按平均 1MB per request）
        avg_size_mb = 1.0
        origin_egress_saved_gb = (
            shield_saved_requests * avg_size_mb / 1024)

        cost_model = CostModel()
        savings = origin_egress_saved_gb * (
            cost_model.origin_egress_cost_per_gb -
            cost_model.shield_cost_per_gb)

        return {
            "origin_requests_without_shield": origin_requests,
            "shield_hit_rate": shield_hit_rate,
            "origin_requests_saved": shield_saved_requests,
            "egress_saved_gb": round(origin_egress_saved_gb, 2),
            "cost_savings_usd": round(savings, 2),
            "roi": round(
                savings / max(origin_egress_saved_gb *
                              cost_model.shield_cost_per_gb, 1) * 100, 1)
        }

    async def forecast_monthly_cost(self) -> Dict:
        """月度成本预测和预算告警"""
        now = datetime.utcnow()
        month_start = now.replace(day=1, hour=0, minute=0,
                                   second=0, microsecond=0)
        days_elapsed = (now - month_start).days + 1
        days_in_month = (now.replace(month=now.month % 12 + 1, day=1)
                         - month_start).days

        # 当月已发生成本
        row = await self.db.fetch_one(
            """SELECT
                 SUM(total_bytes) AS total_bytes,
                 SUM(total_requests) AS total_requests,
                 SUM(total_requests - hit_count) AS origin_requests
               FROM cdn_aggregation_minute
               WHERE ts >= %s""",
            month_start
        )

        if not row or row["total_bytes"] is None:
            return {}

        total_bytes_gb = float(row["total_bytes"]) / (1024**3)
        total_requests = int(row["total_requests"] or 0)
        origin_requests = int(row["origin_requests"] or 0)
        origin_egress_gb = origin_requests * 1.0 / 1024  # 估算 1MB/req

        cost_model = CostModel()
        current_cost = cost_model.calculate_monthly_cost(
            total_bytes_gb, total_requests, origin_egress_gb)

        # 线性预测月末
        projected_bytes_gb = total_bytes_gb * days_in_month / days_elapsed
        projected_requests = total_requests * days_in_month / days_elapsed
        projected_origin_gb = origin_egress_gb * days_in_month / days_elapsed

        projected_cost = cost_model.calculate_monthly_cost(
            projected_bytes_gb, projected_requests, projected_origin_gb)

        budget = 50000  # 月度预算 $50K

        return {
            "month": now.strftime("%Y-%m"),
            "days_elapsed": days_elapsed,
            "days_in_month": days_in_month,
            "current_cost": current_cost,
            "projected_cost": projected_cost,
            "budget_usd": budget,
            "budget_utilization_pct": round(
                projected_cost["total_cost"] / budget * 100, 1),
            "budget_alert": projected_cost["total_cost"] > budget * 0.9,
            "daily_burn_rate_usd": round(
                current_cost["total_cost"] / days_elapsed, 2)
        }

    async def _get_current_policy(self, content_type: str) -> Dict:
        """获取当前缓存策略"""
        # 默认策略
        defaults = {
            "static_video": {"ttl_seconds": 86400},
            "static_image": {"ttl_seconds": 3600},
            "static_css_js": {"ttl_seconds": 86400},
            "dynamic_api": {"ttl_seconds": 0},
            "live_stream": {"ttl_seconds": 5},
        }
        return defaults.get(content_type, {"ttl_seconds": 300})


class CDNCostOptimizerEngine:
    """CDN 成本优化引擎主入口"""

    def __init__(self, db_pool, cdn_api_client, alert_service):
        self.db = db_pool
        self.cdn = cdn_api_client
        self.alert = alert_service
        self.cache_optimizer = CDNCacheOptimizer(db_pool, cdn_api_client)
        self.cost_model = CostModel()

    async def run_optimization(self) -> Dict:
        """执行全量优化分析"""
        now = datetime.utcnow()
        last_24h_start = now - timedelta(hours=24)
        last_7d_start = now - timedelta(days=7)

        offload = await self.cache_optimizer.calculate_origin_offload_ratio(
            last_24h_start, now)
        cache_recs = await self.cache_optimizer.optimize_cache_policy()
        prewarm = await self.cache_optimizer.generate_prewarm_recommendations()
        shield = await self.cache_optimizer.calculate_shield_savings(
            last_7d_start, now)
        forecast = await self.cache_optimizer.forecast_monthly_cost()

        # 预算超支告警
        if forecast.get("budget_alert"):
            await self.alert.send_warning(
                title="CDN 月度预算超支预警",
                message=(
                    f"预测月末成本: ${forecast['projected_cost']['total_cost']:,.2f}\n"
                    f"预算: ${forecast['budget_usd']:,.2f}\n"
                    f"预算利用率: {forecast['budget_utilization_pct']}%\n"
                    f"建议：审查缓存策略 / 启用 Origin Shield / 优化大文件传输"
                )
            )

        return {
            "origin_offload_ratio": offload,
            "cache_policy_recommendations": cache_recs,
            "prewarm_recommendations": prewarm,
            "shield_savings": shield,
            "cost_forecast": forecast
        }
```

## 异常场景补充（续）

### 场景：日志管道 Kafka 积压导致分析数据过时

```
触发：CDN 日志管道的 Kafka 消费者处理速度跟不上生产速度，
      积压从 5 分钟扩大到 2 小时。
      原因：某个 PoP 的日志量突增 10 倍（促销活动），
      消费者组 rebalance 频繁，每次 rebalance 暂停消费 30 秒。
      实时看板显示的命中率是 2 小时前的数据，
      运维团队基于过时数据做了错误的缓存策略调整。
检测：
  1. Kafka consumer lag 监控：lag > 100 万条 → 告警
  2. 日志时间戳与处理时间的差值 > 10 分钟 → 数据过时
  3. 看板增加"数据新鲜度"指标：最新处理的日志时间戳
处理：
  1. 紧急扩容消费者（增加 3 个 consumer 实例）
  2. 增加 Kafka 分区数（从 12 扩到 24），提高并行度
  3. 消费者跳过过旧数据（> 1 小时的日志标记为"stale"，
     只做聚合不入明细表）
  4. 看板增加"数据可能过时"警告横幅
  5. 回溯已做的缓存策略调整，基于最新数据重新评估
  6. 长期：日志管道改为 Lambda 架构（实时 + 批量双路径）
预防：Kafka lag 监控 + 自动扩容消费者 + 数据新鲜度指标
```

### 场景：成本优化引擎错误缩短动态内容 TTL

```
触发：CDN 成本优化引擎将动态 API 接口的缓存 TTL
      从 0 秒调整为 300 秒。
      原因：过去 7 天某动态 API 的命中率为 35%
      （实际是浏览器缓存和重复请求导致），
      优化引擎判定为"可缓存内容"并增加了 TTL。
      结果：用户收到过时的 API 响应（推荐列表 5 分钟不更新、
      库存数据延迟、价格显示错误）。
检测：
  1. 客户投诉：商品价格与实际不一致
  2. 动态 API 的缓存命中率从 0% 跳到 35% → 异常
  3. API 响应时间骤降（异常的快 → 说明是缓存命中）
处理：
  1. 立即回滚动态 API 的 TTL 为 0 秒
  2. 清除所有 PoP 上的动态内容缓存
  3. 优化引擎增加"内容类型白名单"：dynamic_api 永不增加 TTL
  4. 优化引擎增加"变更审批"：TTL 调整需人工确认后生效
  5. 增加 A/B 验证：TTL 变更先在 1 个 PoP 试运行 24 小时
  6. 审计所有 TTL 变更记录，确认无其他误操作
预防：内容类型白名单 + 变更审批 + A/B 验证 + 缓存命中率突变告警

## CDN 源站保护与回源优化完整实现

```python
class OriginProtectionService:
    """源站保护：回源鉴权 + 回源限流 + 回源缓存 + 回源合并"""

    def validate_origin_request(self, request):
        """验证回源请求合法性"""
        # 1. 检查请求来源是否为合法 CDN 节点
        source_ip = request.get("source_ip")
        cdn_ip_ranges = self.redis.smembers("cdn_node_ips")

        is_cdn_node = False
        for ip_range in cdn_ip_ranges:
            ip_range = ip_range.decode() if isinstance(ip_range, bytes) else ip_range
            if self._ip_in_range(source_ip, ip_range):
                is_cdn_node = True
                break

        if not is_cdn_node:
            # 非法回源 → 可能是绕过 CDN 直接访问源站
            self.db.insert("origin_access_violations", {
                "source_ip": source_ip,
                "url": request.get("url"),
                "user_agent": request.get("user_agent", ""),
                "blocked_at": now()
            })
            return {"allowed": False, "reason": "non_cdn_source"}

        # 2. 回源鉴权 Token
        token = request.get("headers", {}).get("X-CDN-Auth-Token")
        if not self._validate_cdn_token(token):
            return {"allowed": False, "reason": "invalid_token"}

        # 3. 回源限流（防止 CDN 节点异常导致回源风暴）
        rate_key = f"origin_rate:{source_ip}"
        count = self.redis.incr(rate_key)
        if count == 1:
            self.redis.expire(rate_key, 1)
        if count > 1000:  # 每秒 1000 次回源上限
            return {"allowed": False, "reason": "rate_limited"}

        return {"allowed": True}

    def optimize_origin_pull(self, request):
        """优化回源请求"""
        url = request["url"]

        # 1. 回源合并（同一资源多个 CDN 节点同时回源 → 合并为一次）
        merge_key = f"origin_merge:{url}"
        merge_window = 2  # 2 秒合并窗口

        existing_request = self.redis.get(merge_key)
        if existing_request:
            # 已有回源请求进行中 → 等待结果
            return {"action": "wait_for_merge", "merge_key": merge_key}

        # 标记正在回源
        self.redis.setex(merge_key, merge_window, "1")

        # 2. 回源预取（预测可能需要的资源）
        related_urls = self._predict_prefetch_urls(url)
        for related_url in related_urls:
            prefetch_key = f"prefetch:{related_url}"
            if not self.redis.exists(prefetch_key):
                self.redis.setex(prefetch_key, 300, "1")
                self._async_prefetch(related_url)

        # 3. 条件回源（If-Modified-Since / If-None-Match）
        cached_etag = self.redis.get(f"etag:{url}")
        if cached_etag:
            request["headers"]["If-None-Match"] = cached_etag.decode()

        cached_last_modified = self.redis.get(f"last_modified:{url}")
        if cached_last_modified:
            request["headers"]["If-Modified-Since"] = cached_last_modified.decode()

        return {"action": "pull_from_origin", "url": url}

    def _predict_prefetch_urls(self, url):
        """预测预取 URL"""
        # 基于访问模式：视频分片 → 预取后续分片
        # URL 模式: /video/123/segment_001.ts → 预取 segment_002.ts, segment_003.ts
        match = re.match(r'(.*/segment_)(\d+)(\.ts)', url)
        if match:
            prefix, num, suffix = match.groups()
            current = int(num)
            return [f"{prefix}{str(current + i).zfill(len(num))}{suffix}"
                    for i in range(1, 4)]

        return []

    def _validate_cdn_token(self, token):
        """验证 CDN 回源 Token"""
        if not token:
            return False
        # Token = HMAC(timestamp + node_id, secret)
        try:
            parts = token.split(":")
            timestamp, node_id, signature = parts[0], parts[1], parts[2]
            # 检查时间戳（5 分钟有效期）
            token_time = datetime.fromtimestamp(int(timestamp))
            if abs((now() - token_time).total_seconds()) > 300:
                return False
            # 验证签名
            expected = hmac.new(
                self.cdn_secret.encode(),
                f"{timestamp}:{node_id}".encode(),
                hashlib.sha256).hexdigest()
            return signature == expected
        except (ValueError, IndexError):
            return False

    def _ip_in_range(self, ip, cidr_range):
        """检查 IP 是否在 CIDR 范围内"""
        import ipaddress
        try:
            return ipaddress.ip_address(ip) in ipaddress.ip_network(cidr_range)
        except ValueError:
            return False
```

## 异常场景补充

### 场景：回源风暴

```
触发：CDN 缓存大面积失效 → 数千节点同时回源 → 源站 QPS 暴增 100x → 源站宕机
检测：
  1. 源站 QPS > 正常 10 倍 → 回源风暴
  2. 源站响应延迟 > 5 秒 → 过载
处理：
  1. 回源限流（每个 CDN 节点限速）
  2. 回源合并（同一资源只回源一次）
  3. 降级：返回旧缓存或默认内容
预防：回源限流 + 回源合并 + 降级兜底
```

### 场景：绕过 CDN 直接攻击源站

```
触发：攻击者发现源站 IP → 绕过 CDN 直接请求 → 源站无 WAF 保护 → 被攻击
检测：
  1. 非法回源请求增多 → 源站暴露
  2. 源站收到非 CDN 节点 IP 的请求 → 被绕过
处理：
  1. 源站只允许 CDN 节点 IP 访问（防火墙白名单）
  2. 非法请求返回 403
  3. 隐藏源站 IP（使用 CDN 专用 IP）
预防：IP 白名单 + 403 拦截 + 隐藏源站 IP
```

## CDN 缓存预热与刷新完整实现

```python
class CDNCacheWarmupService:
    """缓存预热：定时预热 + 事件驱动预热 + 批量刷新"""

    def schedule_warmup(self, warmup_config):
        """调度缓存预热任务"""
        task_id = str(uuid4())
        urls = warmup_config["urls"]
        priority = warmup_config.get("priority", "normal")

        self.db.insert("cache_warmup_tasks", {
            "task_id": task_id,
            "url_count": len(urls),
            "priority": priority,
            "status": "pending",
            "scheduled_at": warmup_config.get("scheduled_at", now()),
            "created_at": now()
        })

        # 按优先级入队
        queue_name = f"warmup_{priority}"
        for url in urls:
            self.redis.rpush(queue_name, json.dumps({
                "task_id": task_id, "url": url,
                "headers": warmup_config.get("headers", {})
            }))

        return {"task_id": task_id, "url_count": len(urls),
                "priority": priority}

    def execute_warmup(self, priority="normal", batch_size=50):
        """执行预热任务"""
        queue_name = f"warmup_{priority}"
        urls = []

        for _ in range(batch_size):
            data = self.redis.lpop(queue_name)
            if not data:
                break
            urls.append(json.loads(data))

        if not urls:
            return {"warmed": 0}

        warmed = 0
        failed = 0

        for url_info in urls:
            try:
                # 从源站获取并预热到 CDN 边缘
                response = requests.get(url_info["url"],
                    headers=url_info.get("headers", {}),
                    timeout=30)

                if response.status_code == 200:
                    warmed += 1
                else:
                    failed += 1

            except Exception:
                failed += 1

        return {"warmed": warmed, "failed": failed, "total": len(urls)}

    def purge_cache(self, purge_request):
        """刷新缓存"""
        purge_type = purge_request["type"]  # url / directory / all
        target = purge_request.get("target")

        if purge_type == "url":
            # 单 URL 刷新
            result = self.cdn_api.purge_urls([target])

        elif purge_type == "directory":
            # 目录刷新（刷新匹配前缀的所有 URL）
            # 查找匹配的缓存键
            pattern = f"cdn_cache:{target}*"
            keys = self.redis.keys(pattern)
            if keys:
                self.redis.delete(*keys)
            result = self.cdn_api.purge_prefix(target)

        elif purge_type == "all":
            # 全站刷新（慎用）
            result = self.cdn_api.purge_all()
            self.alert("全站缓存刷新已执行")

        # 记录刷新日志
        self.db.insert("cache_purge_log", {
            "purge_id": str(uuid4()),
            "type": purge_type,
            "target": target,
            "result": json.dumps(result),
            "triggered_by": purge_request.get("operator", "system"),
            "created_at": now()
        })

        return {"status": "purged", "type": purge_type, "target": target}

    def event_driven_warmup(self, event):
        """事件驱动预热（新内容发布时自动预热）"""
        content_id = event["content_id"]
        content_type = event["content_type"]

        # 根据内容类型预热相关 URL
        warmup_urls = []

        if content_type == "video":
            warmup_urls = [
                f"/api/video/{content_id}/info",
                f"/api/video/{content_id}/stream",
                f"/api/video/{content_id}/comments",
            ]
        elif content_type == "article":
            warmup_urls = [
                f"/api/article/{content_id}",
                f"/api/article/{content_id}/related",
            ]
        elif content_type == "product":
            warmup_urls = [
                f"/api/product/{content_id}",
                f"/api/product/{content_id}/reviews",
                f"/api/product/{content_id}/inventory",
            ]

        if warmup_urls:
            self.schedule_warmup({
                "urls": warmup_urls,
                "priority": "high"
            })

        return {"content_id": content_id, "warmup_urls": len(warmup_urls)}
```

## 异常场景补充

### 场景：缓存预热导致源站过载

```
触发：批量预热 10000 个 URL → 同时回源 → 源站 QPS 暴增 → 影响正常用户
检测：
  1. 预热期间源站延迟上升 → 过载
  2. 预热请求占比 > 50% → 影响正常流量
处理：
  1. 预热限速（每秒最多 100 个请求）
  2. 预热分批执行（每批 50，间隔 1 秒）
  3. 预热请求低优先级（正常请求优先）
预防：限速 + 分批 + 优先级控制
```

### 场景：全站刷新后流量雪崩

```
触发：误操作全站刷新 → 缓存全部失效 → 所有请求回源 → 源站雪崩 → 服务不可用
检测：
  1. 缓存命中率从 95% 骤降到 10% → 全站刷新
  2. 源站 QPS 暴增 10 倍 → 雪崩风险
处理：
  1. 全站刷新需审批（双人确认）
  2. 分批刷新（按区域/按 URL 前缀）
  3. 紧急情况下重新预热热门内容
预防：审批机制 + 分批刷新 + 热门内容优先
```

## CDN 流量调度与负载均衡完整实现

```python
class CDNTrafficSchedulerService:
    """流量调度：全局负载均衡 → 区域调度 → 容量感知 → 故障摘除"""

    def route_request(self, request):
        """路由用户请求到最优边缘节点"""
        client_ip = request["client_ip"]
        requested_host = request["host"]

        # 1. 解析客户端位置
        client_location = self.geoip.lookup(client_ip)

        # 2. 获取可用的边缘节点
        available_nodes = self._get_available_edge_nodes(requested_host)

        if not available_nodes:
            return {"status": "no_available_node"}

        # 3. 计算每个节点的评分
        scored_nodes = []
        for node in available_nodes:
            score = self._calculate_node_score(node, client_location, request)
            scored_nodes.append({"node": node, "score": score})

        # 4. 按评分排序
        scored_nodes.sort(key=lambda x: x["score"], reverse=True)

        # 5. 加权随机选择（避免所有请求都到同一节点）
        selected = self._weighted_random_select(scored_nodes)

        return {
            "status": "routed",
            "edge_node_id": selected["node"]["id"],
            "edge_node_ip": selected["node"]["ip"],
            "edge_node_location": selected["node"]["location"],
            "score": selected["score"],
            "client_location": client_location
        }

    def _calculate_node_score(self, node, client_location, request):
        """计算节点评分"""
        score = 100.0

        # 1. 延迟评分（距离越近越好）
        distance_km = self._haversine(
            client_location["lat"], client_location["lng"],
            node["location"]["lat"], node["location"]["lng"])

        # 延迟估算：每 100km 约 1ms
        estimated_latency_ms = distance_km / 100
        score -= estimated_latency_ms * 2  # 延迟惩罚

        # 2. 节点负载评分
        current_load = self.redis.hget(f"cdn_node:{node['id']}", "current_load")
        if current_load:
            load_pct = float(current_load)
            if load_pct > 90:
                score -= 50  # 过载严重惩罚
            elif load_pct > 70:
                score -= 20
            elif load_pct < 30:
                score += 5  # 轻载奖励

        # 3. 缓存命中率评分
        hit_rate = self.redis.hget(f"cdn_node:{node['id']}", "cache_hit_rate")
        if hit_rate:
            score += float(hit_rate) * 0.1

        # 4. 健康状态评分
        health = self.redis.hget(f"cdn_node:{node['id']}", "health_status")
        if health == b"unhealthy":
            score -= 100  # 不健康节点基本不用
        elif health == b"degraded":
            score -= 30

        # 5. 容量评分
        max_capacity = node.get("max_qps", 10000)
        current_qps = float(self.redis.hget(f"cdn_node:{node['id']}", "current_qps") or 0)
        capacity_remaining = max_capacity - current_qps

        if capacity_remaining < 1000:
            score -= 30  # 容量不足惩罚

        return round(score, 1)

    def _weighted_random_select(self, scored_nodes):
        """加权随机选择"""
        import random

        # 取 top 3 节点
        top_nodes = scored_nodes[:3]

        # 将评分转为正数权重
        min_score = min(n["score"] for n in top_nodes)
        weights = [n["score"] - min_score + 1 for n in top_nodes]

        selected = random.choices(top_nodes, weights=weights, k=1)[0]
        return selected

    def _get_available_edge_nodes(self, host):
        """获取可用边缘节点"""
        # 从配置获取该 host 的节点列表
        node_ids = self.redis.smembers(f"cdn_nodes:{host}")

        available = []
        for node_id in node_ids:
            nid = node_id.decode() if isinstance(node_id, bytes) else node_id
            node_data = self.redis.hgetall(f"cdn_node:{nid}")

            if not node_data:
                continue

            status = node_data.get(b"status", b"offline").decode()
            if status != "online":
                continue

            available.append({
                "id": nid,
                "ip": node_data.get(b"ip", b"").decode(),
                "location": {
                    "lat": float(node_data.get(b"lat", b"0")),
                    "lng": float(node_data.get(b"lng", b"0"))
                },
                "max_qps": int(node_data.get(b"max_qps", b"10000")),
            })

        return available

    def handle_node_failure(self, node_id, failure_type):
        """处理节点故障"""
        # 1. 标记节点不健康
        self.redis.hset(f"cdn_node:{node_id}", "health_status", "unhealthy")
        self.redis.hset(f"cdn_node:{node_id}", "failure_type", failure_type)
        self.redis.hset(f"cdn_node:{node_id}", "failed_at", now().isoformat())

        # 2. 获取该节点服务的域名
        affected_hosts = self.redis.smembers(f"node_hosts:{node_id}")

        # 3. 将流量迁移到其他节点
        for host in affected_hosts:
            self.redis.srem(f"cdn_nodes:{host}", node_id)

            # 通知其他节点预热该域名的缓存
            remaining_nodes = self.redis.smembers(f"cdn_nodes:{host}")
            for remaining_node in remaining_nodes[:3]:
                self.notification.send(remaining_node.decode(),
                    f"节点 {node_id} 故障，请预热 {host.decode()} 缓存")

        # 4. 告警
        self.alert(f"CDN 节点 {node_id} 故障: {failure_type}")

        # 5. 触发自动恢复检查
        self.task_queue.submit(self._check_node_recovery, node_id, delay_seconds=300)

        return {"node_id": node_id, "status": "failed",
                "affected_hosts": [h.decode() for h in affected_hosts]}

    def _check_node_recovery(self, node_id):
        """检查节点是否恢复"""
        # 健康检查
        health = self._perform_health_check(node_id)

        if health["healthy"]:
            # 恢复节点
            self.redis.hset(f"cdn_node:{node_id}", "health_status", "healthy")
            self.redis.hset(f"cdn_node:{node_id}", "recovered_at", now().isoformat())

            # 逐步恢复流量（先 10%，观察 5 分钟，再 50%，再 100%）
            self.redis.hset(f"cdn_node:{node_id}", "traffic_pct", "10")

            self.alert(f"CDN 节点 {node_id} 已恢复，逐步恢复流量")
        else:
            # 仍然不健康 → 30 分钟后再检查
            self.task_queue.submit(self._check_node_recovery, node_id, delay_seconds=1800)

    def _perform_health_check(self, node_id):
        """执行健康检查"""
        node = self.redis.hgetall(f"cdn_node:{node_id}")
        node_ip = node.get(b"ip", b"").decode()

        try:
            response = requests.get(f"http://{node_ip}/health", timeout=5)
            return {"healthy": response.status_code == 200}
        except Exception:
            return {"healthy": False}
```

## 异常场景补充

### 场景：CDN 节点全部不可用

```
触发：某区域所有 CDN 节点同时故障 → 无可用边缘节点 → 用户无法访问 → 全站不可用
检测：
  1. 某区域可用节点数 = 0 → 全部故障
  2. 用户请求全部回源 → 源站压力暴增
处理：
  1. 请求直接回源（降级模式）
  2. 源站限速保护
  3. 紧急拉起备用节点
预防：多区域冗余 + 回源降级 + 备用节点
```

### 场景：流量调度导致节点间负载不均

```
触发：调度算法偏好低延迟节点 → 所有请求都到最近的 2 个节点 → 其他节点空闲 → 热点过载
检测：
  1. 某节点 QPS 远超其他节点 → 负载不均
  2. 部分节点过载而其他节点空闲 → 调度偏差
处理：
  1. 调度算法加入负载均衡因子
  2. 加权随机而非总是选最优
  3. 动态调整权重
预防：负载均衡因子 + 加权随机 + 动态权重
```

## CDN 源站保护与智能回源完整实现

```python
import time
import math
import json
import logging
import threading
import hashlib
from dataclasses import dataclass, field
from typing import List, Optional, Dict, Tuple
from enum import Enum
from datetime import datetime, timedelta
from collections import defaultdict

logger = logging.getLogger(__name__)


class RequestPriority(Enum):
    CRITICAL = 0    # 付费用户实时请求
    HIGH = 1        # 活跃用户新鲜内容请求
    NORMAL = 2      # 普通用户新鲜内容请求
    LOW = 3         # 过期缓存刷新请求
    PREFETCH = 4    # 预取请求


class CacheStatus(Enum):
    HIT = "hit"               # 缓存命中
    MISS = "miss"             # 缓存未命中
    EXPIRED = "expired"       # 缓存过期（可用 stale）
    STALE = "stale"           # 返回过期内容
    BYPASS = "bypass"         # 不走缓存（个性化内容）


class OriginFailureType(Enum):
    TIMEOUT = "timeout"               # 回源超时
    CONNECTION_ERROR = "connection"    # 连接失败
    HTTP_5XX = "http_5xx"             # 源站返回 5xx
    RATE_LIMITED = "rate_limited"     # 源站限流
    UNAVAILABLE = "unavailable"       # 源站不可用


class OriginHealthStatus(Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    UNHEALTHY = "unhealthy"
    OFFLINE = "offline"


@dataclass
class OriginRequest:
    """回源请求"""
    request_id: str
    path: str
    cache_key: str
    cache_status: CacheStatus
    priority: RequestPriority
    user_tier: str              # "premium" / "regular" / "free"
    content_type: str           # "video" / "api" / "image"
    is_fresh_request: bool      # 是否需要最新内容
    is_stale_acceptable: bool   # 是否可接受过期内容
    timestamp: float = 0.0

    def __post_init__(self):
        if self.timestamp == 0.0:
            self.timestamp = time.time()


@dataclass
class TokenBucketState:
    """令牌桶状态"""
    tokens: float
    max_tokens: float
    refill_rate: float          # 每秒补充令牌数
    last_refill_time: float


@dataclass
class OriginHealth:
    """源站健康状态"""
    origin_id: str
    status: OriginHealthStatus
    latency_p50: float          # 毫秒
    latency_p95: float
    latency_p99: float
    error_rate: float           # 0.0 - 1.0
    capacity_used: float        # 0.0 - 1.0
    active_connections: int
    last_probe_time: float
    consecutive_failures: int
    alert_triggered: bool


class OriginProtectionService:
    """
    CDN 源站保护与智能回源服务

    核心能力：
    1. 根据缓存状态、请求优先级、源站负载和限流策略决定是否回源
    2. 每个源站独立的令牌桶限流，支持突发和持续速率
    3. 按优先级排序待回源请求（付费用户 > 新鲜内容 > 过期内容）
    4. 源站故障时自动降级（stale 缓存 + 备用源站 + 告警）
    5. 实时监控源站健康（延迟、错误率、容量）
    """

    def __init__(self, db_client, redis_client, config: dict = None):
        self.db = db_client
        self.redis = redis_client
        self.config = config or {}

        # 源站限流配置
        self.rate_limits = self._load_rate_limits()

        # 令牌桶状态（内存缓存，定期从 Redis 同步）
        self.token_buckets: Dict[str, TokenBucketState] = {}
        self.bucket_lock = threading.Lock()

        # 源站健康状态缓存
        self.health_cache: Dict[str, OriginHealth] = {}
        self.health_lock = threading.Lock()

        # 待处理请求队列（按源站分组）
        self.pending_requests: Dict[str, List[OriginRequest]] = defaultdict(list)
        self.pending_lock = threading.Lock()

        # 降级统计
        self.degrade_stats = defaultdict(int)
        self.total_stats = defaultdict(int)

        # 告警阈值
        self.alert_latency_threshold = self.config.get("alert_latency_ms", 500)
        self.alert_error_rate_threshold = self.config.get("alert_error_rate", 0.05)
        self.alert_capacity_threshold = self.config.get("alert_capacity", 0.85)

    def should_fetch_from_origin(self, request: OriginRequest) -> Dict:
        """
        决定是否回源

        决策流程：
        1. 缓存命中且新鲜 → 不回源
        2. 缓存未命中但请求优先级低 + 源站负载高 → 返回 stale 或降级
        3. 源站当前限流中 → 按优先级决定等待或降级
        4. 缓存过期但 stale 可接受 + 源站负载高 → 返回 stale 并异步刷新
        5. 源站不可用 → 直接降级

        返回: {"should_fetch": bool, "reason": str, "fallback": Optional[str]}
        """
        self.total_stats[request.path] += 1
        origin_id = self._resolve_origin_id(request.path)
        origin_health = self.get_origin_health_status(origin_id)

        # 规则1: 缓存命中且新鲜 → 不回源
        if request.cache_status == CacheStatus.HIT:
            return {
                "should_fetch": False,
                "reason": "cache_hit",
                "fallback": None,
            }

        # 规则2: 源站不可用 → 降级
        if origin_health.status == OriginHealthStatus.OFFLINE:
            return {
                "should_fetch": False,
                "reason": "origin_offline",
                "fallback": self._get_fallback_response(request, origin_id),
            }

        # 规则3: 缓存过期但 stale 可接受 + 源站负载高 → 返回 stale
        if (request.cache_status == CacheStatus.EXPIRED and
                request.is_stale_acceptable and
                origin_health.capacity_used > 0.7):
            return {
                "should_fetch": False,
                "reason": "stale_acceptable_high_load",
                "fallback": "stale_cache",
                "async_refresh": True,  # 标记需要异步刷新
            }

        # 规则4: 源站限流中 → 按优先级决定
        rate_limit_result = self._check_origin_rate_limit(origin_id)
        if not rate_limit_result["allowed"]:
            # 优先级 CRITICAL 和 HIGH 可以超出限流（牺牲低优先级请求）
            if request.priority.value <= RequestPriority.HIGH.value:
                # 高优先级：允许回源但标记为 burst
                logger.info(
                    f"高优先级请求突破限流: origin={origin_id}, "
                    f"priority={request.priority.name}, path={request.path}"
                )
            else:
                # 低优先级：降级
                return {
                    "should_fetch": False,
                    "reason": f"rate_limited:{rate_limit_result['reason']}",
                    "fallback": self._get_fallback_response(request, origin_id),
                }

        # 规则5: 源站降级状态 → 仅允许高优先级回源
        if origin_health.status == OriginHealthStatus.DEGRADED:
            if request.priority.value > RequestPriority.HIGH.value:
                return {
                    "should_fetch": False,
                    "reason": "origin_degraded_low_priority",
                    "fallback": self._get_fallback_response(request, origin_id),
                }

        # 规则6: 请求优先级极低 + 源站中等负载 → 延迟回源
        if (request.priority == RequestPriority.PREFETCH and
                origin_health.capacity_used > 0.5):
            return {
                "should_fetch": False,
                "reason": "prefetch_deferred",
                "fallback": None,
                "retry_after_seconds": 30,
            }

        # 规则7: 允许回源，消耗令牌
        if rate_limit_result.get("allowed", True):
            self._consume_token(origin_id)

        return {
            "should_fetch": True,
            "reason": "approved",
            "fallback": None,
        }

    def apply_origin_rate_limit(self, origin_id: str) -> Dict:
        """
        对指定源站实施令牌桶限流

        令牌桶算法：
        - max_tokens: 桶容量（允许突发量）
        - refill_rate: 每秒补充令牌数（持续速率）
        - 每次请求消耗 1 个令牌
        - 桶空时拒绝请求

        返回: {"allowed": bool, "remaining_tokens": float,
               "retry_after_seconds": float, "reason": str}
        """
        with self.bucket_lock:
            bucket = self.token_buckets.get(origin_id)
            if bucket is None:
                # 初始化令牌桶
                limit_config = self.rate_limits.get(origin_id, {})
                max_tokens = limit_config.get("burst", 2000)
                refill_rate = limit_config.get("sustained_rps", 500)
                bucket = TokenBucketState(
                    tokens=float(max_tokens),
                    max_tokens=float(max_tokens),
                    refill_rate=float(refill_rate),
                    last_refill_time=time.time(),
                )
                self.token_buckets[origin_id] = bucket

            # 补充令牌
            now = time.time()
            elapsed = now - bucket.last_refill_time
            new_tokens = elapsed * bucket.refill_rate
            bucket.tokens = min(bucket.tokens + new_tokens, bucket.max_tokens)
            bucket.last_refill_time = now

            # 尝试消耗令牌
            if bucket.tokens >= 1.0:
                bucket.tokens -= 1.0
                remaining = bucket.tokens
                # 同步到 Redis
                self._sync_bucket_to_redis(origin_id, bucket)
                return {
                    "allowed": True,
                    "remaining_tokens": remaining,
                    "retry_after_seconds": 0.0,
                    "reason": "ok",
                }
            else:
                # 令牌不足
                wait_time = (1.0 - bucket.tokens) / bucket.refill_rate
                self._sync_bucket_to_redis(origin_id, bucket)
                logger.warning(
                    f"源站 {origin_id} 限流: 剩余令牌={bucket.tokens:.1f}, "
                    f"需等待 {wait_time:.2f}s"
                )
                return {
                    "allowed": False,
                    "remaining_tokens": 0.0,
                    "retry_after_seconds": round(wait_time, 2),
                    "reason": "token_bucket_empty",
                }

    def prioritize_requests(self, requests: List[OriginRequest]) -> List[OriginRequest]:
        """
        按优先级排序待回源请求

        排序规则（优先级从高到低）：
        1. 付费用户（premium）> 活跃用户（regular）> 免费用户（free）
        2. 新鲜内容请求 > 过期内容刷新
        3. 需要最新内容的请求 > 可接受 stale 的请求
        4. 实时 API > 视频内容 > 图片内容

        同优先级内按请求时间排序（FIFO），避免饥饿。
        """
        def sort_key(req: OriginRequest) -> Tuple:
            # 1. 请求优先级（值越小优先级越高）
            priority_val = req.priority.value

            # 2. 用户等级权重
            tier_weight = {
                "premium": 0,
                "regular": 1,
                "free": 2,
            }.get(req.user_tier, 2)

            # 3. 新鲜度权重
            freshness_weight = 0 if req.is_fresh_request else 1

            # 4. stale 可接受性（不能接受 stale 的优先级更高）
            stale_weight = 0 if not req.is_stale_acceptable else 1

            # 5. 内容类型权重
            content_weight = {
                "api": 0,
                "video": 1,
                "image": 2,
                "other": 3,
            }.get(req.content_type, 3)

            # 6. 时间戳（FIFO，同优先级先到先服务）
            return (priority_val, tier_weight, freshness_weight,
                    stale_weight, content_weight, req.timestamp)

        return sorted(requests, key=sort_key)

    def handle_origin_failure(self, origin_id: str,
                              failure_type: OriginFailureType) -> Dict:
        """
        处理源站故障

        故障处理流程：
        1. 记录故障并更新源站健康状态
        2. 连续失败次数达到阈值 → 标记源站为 UNHEALTHY/DEGRADED
        3. 降级策略：优先返回 stale 缓存内容
        4. 重定向到备用源站
        5. 触发告警

        返回: {"fallback_type": str, "fallback_origin": Optional[str],
               "alert_sent": bool}
        """
        with self.health_lock:
            health = self.health_cache.get(origin_id)
            if health is None:
                health = OriginHealth(
                    origin_id=origin_id,
                    status=OriginHealthStatus.HEALTHY,
                    latency_p50=0, latency_p95=0, latency_p99=0,
                    error_rate=0.0, capacity_used=0.0,
                    active_connections=0, last_probe_time=time.time(),
                    consecutive_failures=0, alert_triggered=False,
                )
                self.health_cache[origin_id] = health

            # 更新故障计数
            health.consecutive_failures += 1
            health.error_rate = min(
                health.error_rate + 0.02, 1.0
            )
            health.last_probe_time = time.time()

        # 根据故障类型和连续失败次数更新健康状态
        fallback_origin = None
        alert_sent = False

        if health.consecutive_failures >= 5:
            # 连续5次失败 → 标记为 OFFLINE
            health.status = OriginHealthStatus.OFFLINE
            logger.error(
                f"源站 {origin_id} 已标记为 OFFLINE，连续失败 {health.consecutive_failures} 次"
            )
            # 查找备用源站
            fallback_origin = self._find_alternate_origin(origin_id)
            # 触发紧急告警
            alert_sent = self._send_critical_alert(
                origin_id, failure_type, health.consecutive_failures
            )
            # 延长所有边缘节点缓存 TTL
            self._extend_cache_ttl(origin_id, multiplier=6)
            # 暂停低优先级 PURGE
            self._pause_low_priority_purge()

        elif health.consecutive_failures >= 3:
            # 连续3次失败 → 标记为 UNHEALTHY
            health.status = OriginHealthStatus.UNHEALTHY
            logger.warning(
                f"源站 {origin_id} 标记为 UNHEALTHY，连续失败 {health.consecutive_failures} 次"
            )
            # 查找备用源站
            fallback_origin = self._find_alternate_origin(origin_id)
            # 触发告警
            alert_sent = self._send_warning_alert(
                origin_id, failure_type, health.consecutive_failures
            )
            # 延长缓存 TTL（较小幅度）
            self._extend_cache_ttl(origin_id, multiplier=3)

        elif health.consecutive_failures >= 1:
            # 单次失败 → 标记为 DEGRADED
            health.status = OriginHealthStatus.DEGRADED
            logger.info(
                f"源站 {origin_id} 标记为 DEGRADED，故障类型 {failure_type.value}"
            )
            # 发送信息性告警
            alert_sent = self._send_info_alert(origin_id, failure_type)

        # 记录故障到数据库
        self._record_origin_failure(origin_id, failure_type, health)

        # 更新 Redis 中的健康状态
        self._sync_health_to_redis(origin_id, health)

        # 清空该源站的待处理请求队列，重新分配到备用源站
        if fallback_origin:
            self._redistribute_pending_requests(origin_id, fallback_origin)

        # 确定降级类型
        fallback_type = "stale_cache"
        if fallback_origin:
            fallback_type = "alternate_origin"

        return {
            "fallback_type": fallback_type,
            "fallback_origin": fallback_origin,
            "alert_sent": alert_sent,
        }

    def get_origin_health_status(self, origin_id: str) -> OriginHealth:
        """
        获取源站健康状态

        综合评估维度：
        1. 延迟：P50/P95/P99，P95 > 500ms 标记为降级
        2. 错误率：5xx 比例，> 5% 标记为降级，> 20% 标记为不可用
        3. 容量：CPU/内存使用率，> 85% 标记为容量紧张
        4. 最近探测结果：是否在 30 秒内有成功探测

        若缓存中有状态且未过期（30秒），直接返回；
        否则从 Redis 加载，若 Redis 无数据则从数据库加载并初始化。
        """
        with self.health_lock:
            cached = self.health_cache.get(origin_id)
            if cached and (time.time() - cached.last_probe_time < 30):
                return cached

        # 从 Redis 加载
        redis_key = f"origin_health:{origin_id}"
        redis_data = self.redis.hgetall(redis_key)
        if redis_data:
            health = OriginHealth(
                origin_id=origin_id,
                status=OriginHealthStatus(redis_data.get("status", "healthy")),
                latency_p50=float(redis_data.get("latency_p50", 0)),
                latency_p95=float(redis_data.get("latency_p95", 0)),
                latency_p99=float(redis_data.get("latency_p99", 0)),
                error_rate=float(redis_data.get("error_rate", 0)),
                capacity_used=float(redis_data.get("capacity_used", 0)),
                active_connections=int(redis_data.get("active_connections", 0)),
                last_probe_time=float(redis_data.get("last_probe_time", 0)),
                consecutive_failures=int(redis_data.get("consecutive_failures", 0)),
                alert_triggered=bool(int(redis_data.get("alert_triggered", 0))),
            )
        else:
            # 初始化健康状态
            health = OriginHealth(
                origin_id=origin_id,
                status=OriginHealthStatus.HEALTHY,
                latency_p50=0, latency_p95=0, latency_p99=0,
                error_rate=0.0, capacity_used=0.0,
                active_connections=0, last_probe_time=time.time(),
                consecutive_failures=0, alert_triggered=False,
            )

        # 基于延迟、错误率、容量综合判定健康状态
        health = self._recompute_health_status(health)

        with self.health_lock:
            self.health_cache[origin_id] = health

        return health

    # ========== 内部方法 ==========

    def _recompute_health_status(self, health: OriginHealth) -> OriginHealth:
        """基于延迟、错误率、容量指标重新计算健康状态"""
        issues = []

        # 延迟检查
        if health.latency_p95 > 1000:
            issues.append("high_latency")
        elif health.latency_p95 > 500:
            issues.append("elevated_latency")

        # 错误率检查
        if health.error_rate > 0.20:
            issues.append("high_error_rate")
        elif health.error_rate > 0.05:
            issues.append("elevated_error_rate")

        # 容量检查
        if health.capacity_used > 0.95:
            issues.append("critical_capacity")
        elif health.capacity_used > 0.85:
            issues.append("high_capacity")

        # 综合判定
        if health.consecutive_failures >= 5 or health.error_rate > 0.20:
            health.status = OriginHealthStatus.OFFLINE
        elif (health.consecutive_failures >= 3 or
              health.error_rate > 0.05 or
              health.latency_p95 > 1000):
            health.status = OriginHealthStatus.UNHEALTHY
        elif (health.latency_p95 > 500 or
              health.capacity_used > 0.85 or
              health.error_rate > 0.02):
            health.status = OriginHealthStatus.DEGRADED
        else:
            # 如果之前有问题但现在恢复了，重置连续失败计数
            if health.status != OriginHealthStatus.HEALTHY:
                # 只在最近30秒有成功探测时才恢复
                if time.time() - health.last_probe_time < 30:
                    health.consecutive_failures = 0
                    health.status = OriginHealthStatus.HEALTHY

        return health

    def _check_origin_rate_limit(self, origin_id: str) -> Dict:
        """检查源站限流状态（不消耗令牌）"""
        with self.bucket_lock:
            bucket = self.token_buckets.get(origin_id)
            if bucket is None:
                # 无限流配置 → 允许
                return {"allowed": True, "reason": "no_rate_limit"}

            # 补充令牌
            now = time.time()
            elapsed = now - bucket.last_refill_time
            new_tokens = elapsed * bucket.refill_rate
            current_tokens = min(bucket.tokens + new_tokens, bucket.max_tokens)

            if current_tokens >= 1.0:
                return {"allowed": True, "reason": "tokens_available"}
            else:
                wait_time = (1.0 - current_tokens) / bucket.refill_rate
                return {
                    "allowed": False,
                    "reason": "token_bucket_empty",
                    "retry_after_seconds": round(wait_time, 2),
                }

    def _consume_token(self, origin_id: str):
        """消耗一个令牌（在允许回源时调用）"""
        with self.bucket_lock:
            bucket = self.token_buckets.get(origin_id)
            if bucket is None:
                return
            now = time.time()
            elapsed = now - bucket.last_refill_time
            new_tokens = elapsed * bucket.refill_rate
            bucket.tokens = min(bucket.tokens + new_tokens, bucket.max_tokens)
            bucket.last_refill_time = now
            if bucket.tokens >= 1.0:
                bucket.tokens -= 1.0

    def _get_fallback_response(self, request: OriginRequest,
                                origin_id: str) -> Optional[str]:
        """
        获取降级响应

        降级优先级：
        1. stale 缓存（过期但仍可用的缓存内容）
        2. 备用源站
        3. 通用降级模板
        """
        # 尝试 stale 缓存
        stale_key = f"edge:stale:{request.cache_key}"
        stale_content = self.redis.get(stale_key)
        if stale_content:
            self.degrade_stats["stale_cache"] += 1
            return "stale_cache"

        # 尝试备用源站
        alternate = self._find_alternate_origin(origin_id)
        if alternate:
            self.degrade_stats["alternate_origin"] += 1
            return f"alternate_origin:{alternate}"

        # 通用降级
        if request.content_type == "api":
            self.degrade_stats["api_error"] += 1
            return "error_503"
        elif request.content_type == "video":
            self.degrade_stats["video_placeholder"] += 1
            return "video_placeholder"
        else:
            self.degrade_stats["generic_fallback"] += 1
            return "generic_fallback"

    def _find_alternate_origin(self, failed_origin_id: str) -> Optional[str]:
        """查找备用源站（不同机房、优先同区域）"""
        # 从数据库获取所有源站
        origins = self.db.query("""
            SELECT id, region, priority, status
            FROM cdn_origin_servers
            WHERE status = 'active' AND id != %s
            ORDER BY priority ASC
        """, failed_origin_id)

        if not origins:
            return None

        # 优先选择同区域的源站
        failed_origin = self.db.query_one(
            "SELECT region FROM cdn_origin_servers WHERE id = %s",
            failed_origin_id
        )
        target_region = failed_origin["region"] if failed_origin else None

        # 检查候选源站健康状态
        for origin in origins:
            health = self.get_origin_health_status(origin["id"])
            if health.status in (OriginHealthStatus.HEALTHY, OriginHealthStatus.DEGRADED):
                if target_region and origin["region"] == target_region:
                    return origin["id"]

        # 无同区域健康源站 → 选择任何健康源站
        for origin in origins:
            health = self.get_origin_health_status(origin["id"])
            if health.status == OriginHealthStatus.HEALTHY:
                return origin["id"]

        # 全部不健康 → 返回最健康的（错误率最低的）
        best_origin = None
        best_error_rate = float('inf')
        for origin in origins:
            health = self.get_origin_health_status(origin["id"])
            if health.status != OriginHealthStatus.OFFLINE:
                if health.error_rate < best_error_rate:
                    best_error_rate = health.error_rate
                    best_origin = origin["id"]

        return best_origin

    def _extend_cache_ttl(self, origin_id: str, multiplier: int):
        """延长边缘节点缓存 TTL，减少回源需求"""
        # 通知所有边缘节点延长缓存 TTL
        self.redis.publish("cdn:config:update", json.dumps({
            "action": "extend_ttl",
            "origin_id": origin_id,
            "multiplier": multiplier,
            "timestamp": time.time(),
        }))
        # 同时更新全局配置
        self.redis.set(
            f"origin:ttl_multiplier:{origin_id}",
            str(multiplier),
            ex=1800  # 30 分钟后自动恢复
        )
        logger.info(f"源站 {origin_id} 缓存 TTL 延长 {multiplier} 倍")

    def _pause_low_priority_purge(self):
        """暂停低优先级 PURGE 请求，减少缓存失效"""
        self.redis.set("global:purge:paused:low_priority", "1", ex=1800)
        logger.info("已暂停低优先级 PURGE 请求")

    def _redistribute_pending_requests(self, failed_origin: str,
                                        alternate_origin: str):
        """将待处理请求重新分配到备用源站"""
        with self.pending_lock:
            pending = self.pending_requests.pop(failed_origin, [])
            if pending:
                self.pending_requests[alternate_origin].extend(pending)
                logger.info(
                    f"重新分配 {len(pending)} 个待处理请求: "
                    f"{failed_origin} -> {alternate_origin}"
                )

    def _send_critical_alert(self, origin_id: str,
                              failure_type: OriginFailureType,
                              consecutive_failures: int) -> bool:
        """发送紧急告警"""
        alert_data = {
            "severity": "CRITICAL",
            "origin_id": origin_id,
            "failure_type": failure_type.value,
            "consecutive_failures": consecutive_failures,
            "timestamp": datetime.now().isoformat(),
            "message": (
                f"源站 {origin_id} 已离线（连续失败 {consecutive_failures} 次），"
                f"故障类型: {failure_type.value}。请立即检查！"
            ),
        }
        self.redis.publish("alerts:critical", json.dumps(alert_data))
        # 同时写入告警表
        self.db.execute_insert("""
            INSERT INTO cdn_alerts (severity, origin_id, alert_type, message, created_at)
            VALUES (%s, %s, %s, %s, %s)
        """, "CRITICAL", origin_id, failure_type.value,
             alert_data["message"], datetime.now())
        return True

    def _send_warning_alert(self, origin_id: str,
                             failure_type: OriginFailureType,
                             consecutive_failures: int) -> bool:
        """发送警告告警"""
        alert_data = {
            "severity": "WARNING",
            "origin_id": origin_id,
            "failure_type": failure_type.value,
            "consecutive_failures": consecutive_failures,
            "timestamp": datetime.now().isoformat(),
            "message": (
                f"源站 {origin_id} 不健康（连续失败 {consecutive_failures} 次），"
                f"故障类型: {failure_type.value}"
            ),
        }
        self.redis.publish("alerts:warning", json.dumps(alert_data))
        return True

    def _send_info_alert(self, origin_id: str,
                          failure_type: OriginFailureType) -> bool:
        """发送信息性告警"""
        logger.info(f"源站 {origin_id} 单次故障: {failure_type.value}")
        return False

    def _record_origin_failure(self, origin_id: str,
                                failure_type: OriginFailureType,
                                health: OriginHealth):
        """记录源站故障到数据库"""
        self.db.execute_insert("""
            INSERT INTO cdn_origin_failures
                (origin_id, failure_type, consecutive_failures,
                 error_rate, latency_p95, capacity_used, created_at)
            VALUES (%s, %s, %s, %s, %s, %s, %s)
        """, origin_id, failure_type.value, health.consecutive_failures,
             health.error_rate, health.latency_p95,
             health.capacity_used, datetime.now())

    def _resolve_origin_id(self, path: str) -> str:
        """根据请求路径解析目标源站 ID"""
        # 一致性哈希路由
        hash_val = int(hashlib.md5(path.encode()).hexdigest(), 16) % 1000
        # 简化：根据路径前缀路由
        if path.startswith("/api/"):
            return "origin-api-primary"
        elif path.startswith("/video/"):
            return "origin-video-primary"
        else:
            return "origin-static-primary"

    def _load_rate_limits(self) -> Dict[str, Dict]:
        """从数据库加载源站限流配置"""
        rows = self.db.query("""
            SELECT origin_id, burst_rate, sustained_rate
            FROM cdn_origin_rate_limits
        """)
        limits = {}
        for row in rows:
            limits[row["origin_id"]] = {
                "burst": row["burst_rate"],
                "sustained_rps": row["sustained_rate"],
            }
        # 默认配置
        if not limits:
            limits["default"] = {"burst": 2000, "sustained_rps": 500}
        return limits

    def _sync_bucket_to_redis(self, origin_id: str, bucket: TokenBucketState):
        """同步令牌桶状态到 Redis（其他节点可共享）"""
        key = f"origin:bucket:{origin_id}"
        self.redis.hmset(key, {
            "tokens": bucket.tokens,
            "max_tokens": bucket.max_tokens,
            "refill_rate": bucket.refill_rate,
            "last_refill_time": bucket.last_refill_time,
        })
        self.redis.expire(key, 60)

    def _sync_health_to_redis(self, origin_id: str, health: OriginHealth):
        """同步健康状态到 Redis"""
        key = f"origin_health:{origin_id}"
        self.redis.hmset(key, {
            "status": health.status.value,
            "latency_p50": health.latency_p50,
            "latency_p95": health.latency_p95,
            "latency_p99": health.latency_p99,
            "error_rate": health.error_rate,
            "capacity_used": health.capacity_used,
            "active_connections": health.active_connections,
            "last_probe_time": health.last_probe_time,
            "consecutive_failures": health.consecutive_failures,
            "alert_triggered": int(health.alert_triggered),
        })
        self.redis.expire(key, 60)

    def record_origin_success(self, origin_id: str, latency_ms: int):
        """记录回源成功，更新健康状态"""
        with self.health_lock:
            health = self.health_cache.get(origin_id)
            if health is None:
                return

            health.consecutive_failures = 0
            health.last_probe_time = time.time()

            # 更新延迟指标（指数移动平均）
            alpha = 0.2
            if health.latency_p50 == 0:
                health.latency_p50 = latency_ms
                health.latency_p95 = latency_ms
                health.latency_p99 = latency_ms
            else:
                health.latency_p50 = (
                    (1 - alpha) * health.latency_p50 + alpha * latency_ms
                )
                # P95 和 P99 估计
                health.latency_p95 = max(
                    health.latency_p95 * 0.95,
                    latency_ms * 1.5
                )
                health.latency_p99 = max(
                    health.latency_p99 * 0.95,
                    latency_ms * 2.0
                )

            # 降低错误率
            health.error_rate = max(health.error_rate - 0.01, 0.0)

            # 如果之前有问题，现在恢复了
            if health.status != OriginHealthStatus.HEALTHY:
                health.status = OriginHealthStatus.HEALTHY
                logger.info(f"源站 {origin_id} 恢复健康")

        self._sync_health_to_redis(origin_id, health)

    def get_degrade_stats(self) -> Dict:
        """获取降级统计"""
        return {
            "degrade_counts": dict(self.degrade_stats),
            "total_requests": dict(self.total_stats),
        }
```

## 异常场景补充

### 场景：源站突发流量导致保护机制误触发

```
触发: 某热门新剧上线，用户请求量在5分钟内从1万QPS暴涨到15万QPS。CDN缓存尚未预热，大量回源请求涌入。源站令牌桶的burst容量（2000个令牌）在1秒内耗尽，sustained rate（500/s）远低于实际需求。限流机制正确启动，但将大量正常用户的请求也降级为stale缓存或503错误，导致用户体验严重受损——视频无法播放、页面加载失败。

检测:
  1. 降级率监控：当降级请求比例在5分钟内从<1%飙升到>30%，且源站健康状态仍为HEALTHY（非真正故障），则判定为保护机制误触发
  2. 回源QPS与令牌桶消耗速率对比：回源QPS > 令牌桶refill_rate * 3，且持续超过2分钟，说明限流配置与实际流量严重不匹配
  3. 用户投诉/错误率监控：5xx错误率 > 5%且源站CPU < 70%，说明源站实际有能力处理但被限流阻止
  4. 缓存命中率突降：缓存命中率从>90%降到<50%，且持续下降，说明大量请求因缓存未命中而回源

处理:
  1. 紧急扩容令牌桶：自动检测到误触发后，将burst容量临时提升到原来的3倍，sustained rate提升到原来的5倍，持续30分钟
  2. 触发缓存预热：向所有边缘节点推送热门内容，减少回源需求
  3. 启用请求合并（coalescing）：同一资源5秒内只回源1次，其余请求等待结果
  4. 放宽降级条件：将"源站负载>70%就降级"临时改为">90%才降级"
  5. 启用源站弹性扩容：通知源站自动扩容（如果配置了K8s HPA）
  6. 流量分级保障：确保premium用户的请求不受限流影响，free用户可降级为stale缓存

预防:
  - 令牌桶参数自适应：基于过去1小时的流量P95自动调整refill_rate，使限流阈值跟随流量变化
  - 区分"流量突增"和"源站故障"：流量突增时源站延迟正常，不应触发限流；源站故障时延迟飙升，才应限流
  - 分层限流：不同内容类型设置不同限流阈值（视频元数据1000 QPS，热门列表5000 QPS，静态内容20000 QPS）
  - 热点预检测：新剧上线前30分钟主动预热缓存到所有边缘节点
  - 动态burst配置：burst容量 = max(2000, 最近1小时峰值QPS * 2)
```

### 场景：源站宕机时 CDN 降级策略不足

```
触发: 主源站机房断电导致完全不可用。CDN边缘节点检测到源站OFFLINE后，尝试降级：返回stale缓存内容。但存在以下问题：1) 某些API（个性化推荐、用户进度同步）没有stale缓存，直接返回503；2) stale缓存的TTL在1小时后过期，过期后仍返回503；3) 备用源站配置存在但从未验证，切换后发现备用源站数据延迟3小时（binlog同步积压），导致用户看到旧数据；4) 全降级模式下没有"服务降级页面"，用户只看到空白或错误提示。

检测:
  1. 源站连续失败检测：连续5次回源超时/5xx → 标记OFFLINE
  2. 降级有效性检测：降级后503错误率仍>10% → 降级策略不足
  3. stale缓存覆盖率检测：统计有stale缓存的URL占比，<50%则降级不可靠
  4. 备用源站数据一致性检测：对比备用源站与主源站的最新数据时间戳，差异>5分钟则标记为数据不一致

处理:
  1. 紧急部署降级页面：对无stale缓存的API返回预定义的降级页面（如"热门推荐"替代"个性化推荐"），而非503
  2. 延长stale缓存TTL：将所有边缘节点的stale缓存TTL延长到24小时（原本1小时），确保有足够缓冲时间
  3. 验证并修复备用源站：检查备用源站binlog同步状态，等待同步追上后再切换流量
  4. 启用只读模式：关闭所有写操作（消息发送、状态更新），只保留浏览功能
  5. 实施渐进式切换：将10%流量先切到备用源站，验证数据一致性后再逐步扩大
  6. 用户通知：通过App推送和站内公告告知用户"部分功能暂时不可用"

预防:
  - 降级页面预构建：为每种不可缓存API预先构建降级页面模板，存储在边缘节点本地
  - stale缓存策略优化：所有缓存内容在过期后保留stale副本24小时（而非1小时），使用stale-while-revalidate=86400
  - 备用源站定期验证：每月执行1次备用源站切换演练，验证数据一致性和切换时间
  - 备用源站数据延迟监控：实时监控binlog同步延迟，>30秒即告警
  - 分级降级策略：明确每种API的降级方案（推荐→热门列表，进度→本地缓存，搜索→历史记录），并写入配置表
  - 灾备演练：每季度执行1次源站完全宕机演练，验证端到端降级流程
```
