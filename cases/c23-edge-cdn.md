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

## 核心挑战

### 挑战 1：动态内容无法缓存

静态内容（视频文件）CDN 缓存命中率 > 95%。动态 API（推荐列表）缓存命中率 < 5%（每人不同）。

### 挑战 2：缓存一致性

视频元数据更新后，50 个边缘节点的缓存需要失效。TTL 过期策略导致最长 TTL 时间内用户看到旧数据。

### 挑战 3：边缘节点资源有限

不能在边缘部署全部微服务。哪些逻辑可以下沉到边缘？

### 挑战 4：全球路由优化

"最近"不等于"最快"——某节点可能负载高或回源链路差。

## 设计约束

- 动态 API P90 延迟 < 200ms
- 缓存一致性：元数据更新后 < 10 秒生效
- 边缘节点：有限 CPU/内存（不能部署全部服务）

## 请先独立思考（限时 30 分钟）

1. 动态内容加速方案：边缘计算 vs 路由优化 vs 协议优化？各适用什么场景？
2. 缓存失效：TTL vs 主动推送 vs 事件驱动？如何保证一致性？
3. 如何判断"哪些逻辑可以下沉到边缘"？选择标准是什么？

---

## 设计解析

### 分层加速策略：按内容可缓存性分级

```
请求类型                可缓存性    加速策略
视频文件（/video/*.mp4）→ 高（>95%）→ CDN 缓存
视频封面（/img/*.jpg） → 高（>90%）→ CDN 缓存
视频元数据（/api/video）→ 中（~80%）→ 边缘缓存 + 短TTL + 主动失效
热门列表（/api/trending）→ 中（~60%）→ 边缘计算 + 定期刷新
个性化推荐（/api/recommend）→ 低（<5%）→ 路由优化 + 协议优化
```

**核心原则：可缓存的用缓存，不可缓存的用优化。**

### 静态内容：CDN 缓存策略

```
Cache-Control 策略：
  视频文件：public, max-age=31536000（1年，内容不变）
  封面图片：public, max-age=86400（1天，封面可能更新）
  视频元数据：public, max-age=300（5分钟，评分等会变）
```

### 可缓存的动态内容：边缘缓存 + 主动失效

```python
class EdgeCacheManager:
    """边缘节点缓存管理"""

    def on_metadata_updated(self, video_id):
        """视频元数据更新 → 主动失效所有边缘节点"""
        # 向所有边缘节点发送 PURGE 请求
        for edge in self.edge_nodes:
            self.http_post(f"{edge}/api/cache/purge", {
                "keys": [f"video:{video_id}", f"trending:*"]
            })

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

| 保障 | 作用 | 覆盖场景 |
|------|------|---------|
| 短 TTL（5分钟） | 兜底 | 主动失效推送丢失时，最多 5 分钟旧数据 |
| 主动 PURGE | 实时 | 元数据更新后 < 1 秒生效 |

双重保障：失效推送可能因网络问题丢失 → TTL 兜底。

### 半可缓存的动态内容：边缘计算

```javascript
// 边缘 Worker（类似 Cloudflare Workers）
// 热门列表：全球数据相同，可缓存；个性化过滤在边缘做

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

    // 2. 在边缘做区域过滤（不需要回源）
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

**边缘计算的优势：**

| 维度 | 传统回源 | 边缘计算 |
|------|---------|---------|
| 延迟 | 200-500ms（跨洲 RTT） | < 50ms（边缘处理） |
| 源站负载 | 每个请求都回源 | 缓存命中后不回源 |
| 数据传输量 | 全量返回 | 边缘过滤后返回 |

**哪些逻辑适合下沉到边缘？**

| 逻辑 | 是否适合边缘 | 原因 |
|------|------------|------|
| 区域过滤 | ✓ | 只需国家信息，无状态 |
| 个性化排序 | ✓ | 基于用户 cookie，轻量 |
| 推荐模型推理 | △ | 轻量模型可以，深度模型不行 |
| 用户认证 | △ | JWT 验证可以，数据库查询不行 |
| 订单处理 | ✗ | 需要数据库事务，不能在边缘 |

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
            # 实时探测延迟（每 30 秒一次）
            latency = self.probe_latency(edge_node, origin)
            # 节点负载（0-1）
            load = self.get_node_load(origin)

            # 综合评分：延迟权重 70%，负载权重 30%
            scores[origin] = latency * 0.7 + load * 1000 * 0.3

        return min(scores, key=scores.get)

    def probe_latency(self, edge_node, origin):
        """实时探测延迟"""
        # 使用 ICMP ping + HTTP health check
        key = (edge_node.id, origin.id)
        if key in self.probe_results and \
           now() - self.probe_results[key].time < timedelta(seconds=30):
            return self.probe_results[key].latency

        # 执行探测
        latency = self.http_ping(f"{origin.url}/health")
        self.probe_results[key] = ProbeResult(latency=latency, time=now())
        return latency
```

## 常见陷阱（深度分析）

### 陷阱 1：动态内容也用长 TTL

**后果：** 个性化推荐被缓存 → 用户 A 看到用户 B 的推荐 → 数据泄露。

**解决方案：** 个性化内容不缓存或使用 per-user 缓存键 + 极短 TTL（< 10s）。

### 陷阱 2：缓存失效只靠 TTL

**后果：** 视频评分更新后，最长 5 分钟（TTL）用户看到旧评分 → 数据不一致 → 用户投诉。

**解决方案：** TTL + 主动 PURGE 双重保障。

### 陷阱 3：边缘部署全量服务

**后果：** 边缘节点资源不够（CPU/内存有限）→ 服务崩溃 → 该区域所有用户受影响。

**解决方案：** 边缘只部署轻量 Worker，重计算仍在源站。判断标准：CPU < 10ms 且内存 < 50MB 的逻辑可以下沉。

### 陷阱 4：DNS 解析不考虑节点负载

**后果：** 某节点负载高但仍被路由到 → 延迟增加 → 用户体验差。

**解决方案：** 路由优化器同时考虑延迟和负载，加权评分选最优节点。

## 延伸思考

- **边缘 AI 推理**：在边缘节点部署轻量推荐模型（如 MobileBERT），减少回源延迟。模型大小 < 100MB，推理 < 50ms。
- **WebTransport/QUIC**：替代 HTTP/2，0-RTT 连接建立，减少握手延迟。
- **Prefetch**：基于用户行为预测下一个请求（如用户浏览视频列表 → 预取热门视频元数据），提前在边缘缓存。