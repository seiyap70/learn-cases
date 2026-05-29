# C21: AI模型服务的在线推理架构

## 业务场景

某 AI 公司提供大语言模型（LLM）的在线推理服务，用户通过 API 调用模型生成文本。推理服务的性能和成本直接决定了产品的竞争力。

**一个真实的成本困境：** 8×A100 GPU 集群月租 $16K，但日均请求 100 万次 → 每次 GPU 时间仅 25 秒 × 利用率 40% → 大量 GPU 时间浪费 → 月亏损 $10K。

**已知数据：**
- 模型：70B 参数 LLM（约 140GB 权重，FP16）
- GPU：A100 80GB × 8 张（单机 640GB 总显存）
- 推理延迟：首 token < 500ms，每 token 生成 ~50ms（20 tokens/秒）
- 并发请求上限：单 GPU 约 20 并发（batch 推理）
- 日均请求量：100 万次
- 峰值 QPS：5000
- 平均输出长度：500 tokens（约 25 秒/请求）
- 平均输入长度：1000 tokens

**为什么这是难题？**

1. **推理是串行的**——每个 token 必须依赖前一个 token 的输出
2. **GPU 内存有限**——140GB 权重 + KV Cache + 并发上下文 = 容易 OOM
3. **成本与利用率矛盾**——GPU 按小时计费，但请求有峰谷 → 利用率低
4. **长尾延迟**——某些请求输入 8000 tokens → Prefill 阶段极慢 → 拖慢其他请求

## 核心挑战

### 挑战 1：GPU 显存分配

```
总显存 640GB = 140GB 权重 + KV Cache + 临时缓冲区

KV Cache 计算：
  每层 KV Cache = 2 × batch_size × seq_len × hidden_dim × 2 bytes
  70B 模型有 80 层，hidden_dim = 8192
  
  单请求 500 tokens KV Cache ≈ 2 × 1 × 500 × 8192 × 2 × 80 = 1.3GB
  20 并发 KV Cache ≈ 26GB
  140GB 权重 + 26GB KV Cache + 10GB 临时 = 176GB → 单机可以

  但如果输入 8000 tokens + 输出 2000 tokens：
  单请求 KV Cache ≈ 2 × 1 × 10000 × 8192 × 2 × 80 = 26GB
  20 并发 KV Cache ≈ 520GB → 超出 640GB - 140GB = 500GB 可用空间！
```

### 挑战 2：吞吐量与成本

```
单机吞吐量：
  20 并发 × 20 tokens/秒 = 400 tokens/秒
  平均请求 500 tokens → 吞吐量 = 0.8 请求/秒

  峰值 QPS 5000 → 需要 6250 台机器 → 月成本 $100M → 不可行

  实际方案：batch 推理 + 量化 + 弹性扩缩容
```

### 挑战 3：延迟与吞吐的权衡

| Batch Size | 吞吐量 (tokens/s) | 单请求延迟 | GPU 利用率 |
|-----------|-------------------|-----------|-----------|
| 1 | 20 | 25s | 5% |
| 10 | 150 | 33s | 40% |
| 20 | 250 | 40s | 65% |
| 40 | 350 | 57s | 85% |

Batch 越大吞吐量越高，但延迟也越高。如何平衡？

## 设计约束

- P99 首 token 延迟 < 1 秒
- P99 每 token 生成延迟 < 100ms
- GPU 利用率 > 70%（非峰值时段 > 50%）
- 单次请求成本 < $0.02
- 弹性扩缩容延迟 < 60 秒

## 请先独立思考（限时 35 分钟）

1. KV Cache 管理方案：PagedAttention（vLLM）vs 静态预分配？各有什么内存浪费？
2. Batch 调度：静态 batch vs 连续 batch（continuous batching）？延迟和吞吐如何权衡？
3. 量化方案：FP16 vs INT8 vs INT4？精度损失和吞吐量提升的量化数据？
4. 弹性扩缩容：GPU 冷启动需要 30 秒加载模型 → 如何应对突发流量？

---

## 设计解析

### 端到端推理架构

```
用户请求 → API Gateway → 请求队列（优先级排序）
                              ↓
                     调度器（continuous batching）
                              ↓
                  GPU 集群（多机多卡，模型并行）
                              ↓
                  流式输出 → SSE → 用户
```

### KV Cache 管理：PagedAttention（vLLM 方案）

**问题：** 传统方案为每个请求预分配最大 seq_len 的 KV Cache → 大量浪费。

```
传统方案：
  为每个请求预分配 2048 tokens 的 KV Cache
  但平均只用 500 tokens → 浪费 75% 内存
  20 并发 × 2048 tokens × 1.3GB/1000tokens × 2 = 53GB 浪费

PagedAttention 方案：
  KV Cache 按页分配（每页 16 tokens）
  请求需要多少就分配多少页
  内存碎片率 < 4%（vs 传统方案的 75% 浪费）
```

```python
class PagedKVCacheManager:
    def __init__(self, total_gpu_memory_gb, model_weights_gb, page_size=16):
        self.page_size = page_size
        available = total_gpu_memory_gb - model_weights_gb  # 640-140=500GB
        self.pages_per_gb = 1000 / (2 * page_size * 8192 * 2 * 80 / 1e9)  # 约 5 pages/GB
        self.total_pages = int(available * self.pages_per_gb)
        self.free_pages = set(range(self.total_pages))
        self.request_pages = {}  # request_id → list of page_ids

    def allocate(self, request_id, num_tokens):
        """为请求分配 KV Cache 页"""
        pages_needed = math.ceil(num_tokens / self.page_size)
        
        if len(self.free_pages) < pages_needed:
            # 内存不足 → 抢占最低优先级请求的 KV Cache
            self.preempt_lowest_priority()
        
        allocated = []
        for _ in range(pages_needed):
            page = self.free_pages.pop()
            allocated.append(page)
        
        self.request_pages[request_id] = allocated
        return allocated

    def free(self, request_id):
        """释放请求的 KV Cache 页"""
        if request_id in self.request_pages:
            for page in self.request_pages[request_id]:
                self.free_pages.add(page)
            del self.request_pages[request_id]
```

### 连续批处理（Continuous Batching）

**传统静态 Batch 的问题：**

```
Batch = [请求A(500 tokens), 请求B(200 tokens), 请求C(1000 tokens)]

静态 Batch：等最慢的请求C完成（1000/20=50秒）才能处理下一批
  → 请求B 在 10 秒就完成了，但 GPU 仍等 C → 浪费 40 秒

连续 Batch：请求B 完成后立即加入新请求D
  → GPU 利用率从 40% → 85%
```

```python
class ContinuousBatchScheduler:
    def __init__(self, max_batch_size=20):
        self.max_batch_size = max_batch_size
        self.running_requests = []

    def schedule_iteration(self, waiting_queue):
        """每次迭代（生成 1 token）时调度"""
        # 1. 移除已完成的请求
        completed = [r for r in self.running_requests if r.is_done]
        for r in completed:
            self.kv_cache_manager.free(r.request_id)
            self.running_requests.remove(r)

        # 2. 从等待队列加入新请求（直到 batch 满）
        while len(self.running_requests) < self.max_batch_size and waiting_queue:
            new_request = waiting_queue.popleft()
            self.kv_cache_manager.allocate(new_request.request_id, 
                                           new_request.max_tokens)
            self.running_requests.append(new_request)

        # 3. 生成一个 token（所有请求并行前向传播）
        self.forward_step(self.running_requests)

        # 4. 流式输出已生成的 token
        for r in self.running_requests:
            if r.current_token is not None:
                self.stream_token(r.request_id, r.current_token)
```

**性能对比：**

| 指标 | 静态 Batch | 连续 Batch |
|------|-----------|-----------|
| GPU 利用率 | 40-50% | 80-90% |
| 平均等待时间 | 峰值时 30s | 峰值时 5s |
| 吞吐量 (tokens/s) | 250 | 450 |
| 有效并发 | 20（受限于最慢请求） | 40（动态加入/移出） |

### 模型量化：精度 vs 吞吐量 vs 成本

| 量化方案 | 显存占用 | 吞吐量 | 精度损失（困惑度增加） | 单次请求成本 |
|---------|---------|-------|-------------------|-----------|
| FP16（基线） | 140GB | 250 tokens/s | 0% | $0.04 |
| INT8 | 70GB | 400 tokens/s | < 1% | $0.025 |
| INT4 (GPTQ) | 35GB | 600 tokens/s | 2-5% | $0.015 |
| INT4 (AWQ) | 35GB | 650 tokens/s | 1-3% | $0.014 |

**选择建议：**

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| 代码生成（精度敏感） | INT8 | 精度损失 < 1%，可接受 |
| 聊天/创意写作 | INT4 (AWQ) | 精度损失 1-3% 用户无感，成本减半 |
| 金融/医疗（不允许精度损失） | FP16 | 精度第一 |

### GPU 弹性扩缩容

```python
class GPUAutoscaler:
    def __init__(self, min_replicas=2, max_replicas=50):
        self.min_replicas = min_replicas
        self.max_replicas = max_replicas
        self.current_replicas = 2

    def check_scaling(self, current_qps, avg_latency_ms):
        """每 30 秒检查一次扩缩容"""
        # 目标：每台机器处理 25 QPS，P99 延迟 < 1s
        target_qps_per_replica = 25
        needed = max(self.min_replicas, 
                    math.ceil(current_qps / target_qps_per_replica))
        
        # 延迟过高 → 紧急扩容
        if avg_latency_ms > 2000:
            needed = min(needed * 2, self.max_replicas)
        
        needed = min(needed, self.max_replicas)

        if needed > self.current_replicas:
            self.scale_up(needed - self.current_replicas)
        elif needed < self.current_replicas:
            self.scale_down(self.current_replicas - needed)

    def scale_up(self, count):
        """扩容：启动新 GPU 实例 + 加载模型"""
        for i in range(count):
            # GPU 冷启动：启动实例 30s + 加载模型 20s = 50s
            instance = self.launch_gpu_instance()
            instance.load_model()  # 20s
            self.add_to_pool(instance)

    def scale_down(self, count):
        """缩容：等待请求完成后移除"""
        for i in range(count):
            # 标记为 draining → 不接受新请求
            instance = self.get_least_loaded()
            instance.drain()
            # 等 30 秒让现有请求完成
            self.schedule_terminate(instance, delay=30)
```

**预测性扩缩容：**

```python
class PredictiveScaler:
    """基于历史流量的预测性扩缩容"""
    
    def predict_qps(self, minutes_ahead=5):
        """预测未来 5 分钟的 QPS"""
        # 用过去 7 天同一时段的均值 + 趋势
        historical = self.get_historical_qps(
            same_weekday=True, same_hour=True, last_n_weeks=7
        )
        trend = self.get_recent_trend()  # 最近 1 小时的趋势
        
        predicted = historical.mean() * (1 + trend)
        return max(predicted, self.current_qps * 0.8)  # 下限保护

    def pre_scale(self):
        """提前扩容（在流量到来之前）"""
        predicted = self.predict_qps(minutes_ahead=5)
        needed = math.ceil(predicted / 25)  # 25 QPS/机器
        
        if needed > self.current_replicas:
            # 提前 5 分钟扩容（冷启动需 50 秒）
            self.scale_up(needed - self.current_replicas)
```

### 流式输出架构

```python
class StreamingInferenceService:
    def infer_stream(self, request):
        """流式推理：每生成 1 个 token 就推送给用户"""
        # 1. 加入调度队列
        request_id = self.scheduler.enqueue(request)
        
        # 2. 创建 SSE 流
        stream = SSEStream(request_id)
        
        # 3. 异步生成 token 并推送
        def generate_and_stream():
            while True:
                token = self.scheduler.get_next_token(request_id)
                if token is None:  # 生成完成
                    stream.send({"type": "done", "request_id": request_id})
                    break
                stream.send({"type": "token", "content": token})
        
        threading.Thread(target=generate_and_stream).start()
        return stream
```

### 限流与优先级队列

```python
class RequestPriorityQueue:
    def __init__(self):
        self.queues = {
            "premium": deque(),   # 付费用户：优先处理
            "standard": deque(),  # 普通用户
            "free": deque(),      # 免费用户：最低优先级
        }
        self.rate_limits = {
            "premium": (100, 60),    # 100 次/分钟
            "standard": (20, 60),    # 20 次/分钟
            "free": (5, 60),         # 5 次/分钟
        }

    def enqueue(self, request, user_tier):
        # 1. 限流检查
        key = f"rate_limit:{user_tier}:{request.user_id}"
        count = self.redis.incr(key)
        if count == 1:
            self.redis.expire(key, self.rate_limits[user_tier][1])
        if count > self.rate_limits[user_tier][0]:
            return {"status": "rate_limited", "retry_after": 60}

        # 2. 加入对应优先级队列
        self.queues[user_tier].append(request)
        return {"status": "queued", "request_id": request.id}

    def dequeue(self):
        """调度器取请求：优先取高优先级"""
        for tier in ["premium", "standard", "free"]:
            if self.queues[tier]:
                return self.queues[tier].popleft()
        return None
```

## 常见陷阱（深度分析）

### 陷阱 1：静态 Batch 导致 GPU 利用率低

**具体数据：** 静态 Batch 下 GPU 利用率 40-50% → 8×A100 月租 $16K → 浪费 $8K/月。改用连续 Batch → 利用率提升到 85% → 节省 $6K/月。

**解决方案：** 连续 Batch（Continuous Batching），请求完成后立即加入新请求。

### 陷阱 2：KV Cache 静态预分配

**具体数据：** 为每个请求预分配 2048 tokens 的 KV Cache → 平均只用 500 tokens → 浪费 75% 内存 → 可支持并发数从 80 降到 20。

**解决方案：** PagedAttention（vLLM），按需分配 KV Cache 页，内存碎片率 < 4%。

### 陷阱 3：不量化，全用 FP16

**具体数据：** FP16 70B 模型需要 140GB → 必须 8×A100 → 月租 $16K。INT8 量化后 70GB → 4×A100 → 月租 $8K。精度损失 < 1%。

**解决方案：** 根据场景选择量化方案。聊天场景用 INT4 AWQ（成本减半，用户无感），代码生成用 INT8。

### 陷阱 4：GPU 不做弹性扩缩容

**具体数据：** 固定 20 台 GPU → 峰值 QPS 5000 需 200 台 → 但均值 QPS 只有 200 → 需要 8 台 → 12 台浪费 → $24K/月。

**解决方案：** 弹性扩缩容 + 预测性扩容。均值 8 台，峰值 30 台，均值成本 $16K/月。

### 陷阱 5：长请求拖慢短请求

**具体数据：** 一个 8000 token 输入的请求 Prefill 阶段需 5 秒 → 在同一 batch 中拖慢其他请求 → 其他用户的首 token 延迟从 500ms 变成 5 秒。

**解决方案：** 请求按输入长度分桶调度——长请求和长请求组 batch，短请求和短请求组 batch。或使用 Chunked Prefill（将长输入切分为小块逐块处理）。

### 陷阱 6：不做限流

**具体数据：** 免费用户突发 1000 并发 → 占满所有 GPU → 付费用户请求排队 30 秒 → 付费用户流失。

**解决方案：** 按用户等级限流 + 优先级队列。付费用户优先处理，免费用户限流。

## 延伸思考

- **Speculative Decoding**：用小模型（7B）快速生成草稿 token，大模型批量验证 → 吞吐量提升 2-3 倍，精度无损。
- **MoE（Mixture of Experts）**：Mixtral 8×7B 只有 2 个 Expert 激活 → 推理成本 = 14B 模型 → 成本降 80%。
- **Serverless GPU**：按请求计费（$0.0001/token）→ 无需管理集群 → 适合低流量场景。
- **Prefill/Decode 分离**：Prefill（计算密集）在 A100 上做，Decode（内存密集）在 L4 上做 → 成本降低 50%。