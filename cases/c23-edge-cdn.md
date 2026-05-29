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
// 边缘 Worker（类似 Cloudflare Workers）
export default {
  async fetch(request) {
    const country = request.cf.country;
    const cacheKey = `trending:global`;

    // 1. 获取全球热门列表（可缓存 1 分钟）
    let trending = await CACHE.get(cacheKey);
    if (!trending) {
      trending = await fetch('https://origin/api/trending');
      await CACHE.put(cacheKey, trending, { expirationTtl: 60 });
    }

    // 2. 在边缘做区域过滤
    const data = JSON.parse(trending);
    const filtered = data.filter(item => 
      item.available_regions.includes(country)
    );

    // 3. 在边缘做个性化排序（基于用户偏好 cookie）
    const userPrefs = getUserPrefs(request);
    const ranked = rankByPreference(filtered, userPrefs);

    return new Response(JSON.stringify(ranked));
  }
};
```

**边缘计算 vs 传统回源：**

| 维度 | 传统回源 | 边缘计算 |
|------|---------|---------|
| 延迟 | 200-500ms | < 50ms |
| 源站负载 | 每个请求都回源 | 缓存命中后不回源 |
| 数据传输量 | 全量返回 | 边缘过滤后返回 |
| 开发复杂度 | 低 | 中（需写 Worker） |

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
class RoutingOptimizer:
    def __init__(self):
        self.probe_results = {}  # (edge, origin) → latency

    def select_best_origin(self, edge_node, origin_candidates):
        """选择从边缘到源站最快的路径"""
        scores = {}

        for origin in origin_candidates:
            latency = self.probe_latency(edge_node, origin)
            load = self.get_node_load(origin)
            scores[origin] = latency * 0.7 + load * 1000 * 0.3

        return min(scores, key=scores.get)

    def probe_latency(self, edge_node, origin):
        """实时探测延迟"""
        key = (edge_node.id, origin.id)
        if key in self.probe_results and \
           now() - self.probe_results[key].time < timedelta(seconds=30):
            return self.probe_results[key].latency

        latency = self.http_ping(f"{origin.url}/health")
        self.probe_results[key] = ProbeResult(latency=latency, time=now())
        return latency
```

### 源站保护：回源限流

```python
class OriginProtector:
    """源站保护：防止突发热点导致源站崩溃"""

    def __init__(self, max_origin_qps=100000):
        self.max_origin_qps = max_origin_qps
        self.current_qps = 0

    def on_origin_request(self, request):
        """回源请求限流"""
        if self.current_qps >= self.max_origin_qps:
            # 回源限流 → 返回降级内容
            return self.get_degraded_response(request)

        self.current_qps += 1
        try:
            return self.fetch_from_origin(request)
        finally:
            self.current_qps -= 1

    def get_degraded_response(self, request):
        """回源限流时的降级响应"""
        if request.path.startswith("/api/video/"):
            # 返回缓存中的旧版本元数据（延迟 T 分钟可接受）
            stale = self.get_stale_cache(request.path)
            if stale:
                stale.headers["X-Stale"] = "true"
                return stale
            # 完全没有缓存 → 返回通用降级页面
            return Response(status=503, body="Service temporarily unavailable")

        return Response(status=503)
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
class TrafficScheduler:
    """流量调度：灰度发布 + 降级"""

    def route(self, request):
        # 灰度：5% 流量走新边缘节点
        if self.should_use_new_edge(request):
            return self.new_edge_cluster.handle(request)

        # 降级：边缘节点故障 → 切换到备用节点
        primary = self.get_primary_edge(request.user_region)
        if self.is_healthy(primary):
            return primary.handle(request)
        
        fallback = self.get_fallback_edge(request.user_region)
        return fallback.handle(request)

    def should_use_new_edge(self, request):
        """灰度：5% 流量走新边缘节点"""
        user_hash = mmh3.hash(request.user_id) % 100
        return user_hash < 5  # 5% 流量
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

- **边缘 AI 推理**：在边缘节点部署轻量推荐模型，减少回源延迟。
- **WebTransport/QUIC**：替代 HTTP/2，0-RTT 连接建立。
- **Prefetch**：基于用户行为预测下一个请求，提前在边缘缓存。