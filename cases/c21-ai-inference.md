# C21: AI模型服务的在线推理架构

## 业务场景

某 AI 公司提供大语言模型（LLM）的在线推理服务，用户通过 API 调用模型生成文本。推理服务的性能和成本直接决定了产品的竞争力。

**已知数据：**
- 模型：70B 参数 LLM（约 140GB 权重）
- GPU：A100 80GB × 8 张（单机）
- 推理延迟：首 token < 500ms，每 token 生成 ~50ms（20 tokens/秒）
- 并发请求上限：单 GPU 约 20 并发（batch 推理）
- 日均请求量：100 万次
- 峰值 QPS：5000
- 平均输出长度：500 tokens（约 25 秒/请求）

**为什么这是难题？**

LLM 推理有几个独特约束：
1. **推理是串行的**——每个 token 必须依赖前一个 token 的输出，不能并行生成（除首 token 外）
2. **GPU 内存有限**——140GB 权重 + KV Cache + 20 个并发请求的上下文 = 可能超出 8×80GB = 640GB
3. **长请求占用资源久**——25 秒的请求占据 GPU 20 秒以上，低吞吐
4. **流式输出**——用户期望逐 token 看到（像 ChatGPT），不是等 25 秒后一次性返回

## 核心挑战

### 挑战 1：GPU 利用率与吞吐的权衡

单请求推理：GPU 利用率极低（大部分时间在等待内存读取），吞吐约 1 req/s。
批量推理：20 个请求一起推理，GPU 利用率高，吞吐约 20 req/s，但每个请求的延迟增加（等待 batch 组装）。

### 挑战 2：KV Cache 的内存管理

LLM 推理需要为每个请求维护 KV Cache（注意力机制的中间状态），长度约 2MB/token × 500 tokens = 1GB/请求。20 个并发 = 20GB KV Cache。加上 140GB 权重 = 160GB，看似可承受。

但如果有长上下文请求（8K tokens）= 16GB KV Cache/请求 → 2 个请求就占 32GB → 并发大幅下降。

### 挑战 3：流式输出的实现

模型每生成一个 token 就要发送给客户端（SSE/WebSocket）。但 GPU 推理是批量的——一个 batch 中 20 个请求的 token 在同一时刻生成，如何逐个推送给对应的客户端？

### 挑战 4：成本控制

8 张 A100 约 ¥40 万/月。峰值 5000 QPS × 25 秒/请求 = 125000 并发 → 需要 6250 / 20 = 313 台 8-GPU 服务器 → ¥1.25 亿/月。

这不可行。必须通过优化降低 GPU 数量。

## 设计约束

- 首 token 延迟 < 500ms
- 流式输出（逐 token 推送）
- GPU 集群：8×A100 服务器
- 最大上下文长度：4K tokens
- 成本预算：< ¥50 万/月

## 请先独立思考（限时 40 分钟）

1. GPU 推理的 batching 策略：static batching vs dynamic batching vs continuous batching？各自对延迟和吞吐的影响？
2. KV Cache 内存如何管理？长短请求混合时如何避免短请求被长请求挤出？
3. 流式输出的技术实现：SSE vs WebSocket？token 级推送如何与 batch 推理配合？
4. 如何将 GPU 数量从 313 台降到可接受范围？列出至少 3 种优化手段并计算效果。

---

## 设计解析

### Continuous Batching：动态组装 batch

**为什么不用 static batching？**

Static batching：等凑够 20 个请求再一起推理。问题：
- 低峰期可能等 5 秒才凑够 20 个请求 → 首 token 延迟 5 秒（违反 500ms 要求）
- batch 中所有请求必须等最慢的请求完成才能释放 GPU → 短请求被长请求拖慢

**Continuous batching（vLLM 方案）：**

每个 token 生成步骤时，动态组装当前在等待的请求：
- 新请求到达 → 加入 batch
- 某请求完成（所有 token 已生成）→ 从 batch 中移除
- batch 大小随请求流量动态变化

```python
class ContinuousBatcher:
    def __init__(self, gpu, max_batch_size=20):
        self.gpu = gpu
        self.active_requests = []  # 当前正在推理的请求
        self.max_batch_size = max_batch_size

    def on_new_request(self, request):
        """新请求加入推理队列"""
        self.active_requests.append(request)

    def inference_step(self):
        """单步推理：为所有活跃请求生成一个 token"""
        if not self.active_requests:
            return  # 无请求，GPU 空闲

        # 限制 batch 大小
        batch = self.active_requests[:self.max_batch_size]

        # 生成一个 token for each request in batch
        new_tokens = self.gpu.generate_one_token(batch)

        # 检查每个请求是否完成
        completed = []
        for i, token in enumerate(new_tokens):
            batch[i].tokens.append(token)

            if token.is_eos or len(batch[i].tokens) >= batch[i].max_tokens:
                completed.append(batch[i])

            # 流式推送 token 给客户端
            self.stream_token(batch[i].stream_id, token)

        # 移除完成的请求
        for req in completed:
            self.active_requests.remove(req)
```

**Continuous batching 的优势：**
- 首 token 延迟 = GPU 空闲时的加入延迟（约 10ms）+ 首 token 推理延迟（约 50ms）= ~60ms
- 请求完成后立即释放 GPU → 短请求不被长请求拖慢
- GPU 利用率高：只要有请求就不空闲

### PagedAttention：KV Cache 的虚拟内存管理

**核心问题：** KV Cache 为每个请求预分配最大长度（4K tokens = 16GB）的内存 → 即使请求只用了 100 tokens，也占了 16GB → 内存浪费严重。

**vLLM 的 PagedAttention 方案：**

借鉴操作系统的虚拟内存管理——将 KV Cache 分成固定大小的"页"（如每页 16 tokens = 64KB），按需分配：

```python
class KVCacheManager:
    PAGE_SIZE = 16  # tokens per page
    PAGE_BYTES = 64 * 1024  # 64KB per page

    def __init__(self, total_memory):
        self.total_pages = total_memory // self.PAGE_BYTES
        self.free_pages = list(range(self.total_pages))
        self.request_pages = {}  # request_id -> [page_numbers]

    def allocate_page(self, request_id):
        """为请求分配一个新页"""
        if not self.free_pages:
            # 内存不足 → 需要驱逐某个请求的 KV Cache
            evicted = self.evict_request()
            self.free_pages.extend(evicted.pages)

        page = self.free_pages.pop(0)
        self.request_pages[request_id].append(page)
        return page

    def free_request_pages(self, request_id):
        """请求完成后释放所有页"""
        pages = self.request_pages.pop(request_id)
        self.free_pages.extend(pages)

    def evict_request(self):
        """驱逐最不活跃的请求（类似操作系统 swap）"""
        # 找最久未活跃的请求
        oldest = min(self.request_pages.keys(), 
                     key=lambda rid: self.last_activity_time[rid])
        pages = self.request_pages.pop(oldest)
        # 通知客户端：请求被中断，请重新发起
        self.notify_eviction(oldest)
        return oldest
```

**PagedAttention 的内存节省：**

| 场景 | 传统预分配 | PagedAttention | 节省 |
|------|-----------|---------------|------|
| 100 tokens 请求 | 16GB | 400KB（25页） | 99.997% |
| 500 tokens 请求 | 16GB | 2MB（125页） | 99.987% |
| 4K tokens 请求 | 16GB | 16MB（1000页） | 99% |

**效果：** 原本只能并发 20 个请求（320GB 预分配），现在可以并发 200+ 个短请求。

### 流式输出：SSE 实现

```python
class StreamHandler:
    """将 GPU 生成的 token 逐个推送给客户端"""

    def stream_token(self, stream_id, token):
        """通过 SSE 推送一个 token"""
        if token.is_eos:
            # 结束标记
            self.sse_send(stream_id, {
                "type": "done",
                "usage": {"prompt_tokens": prompt_len, "completion_tokens": completion_len}
            })
        else:
            # 普通 token
            self.sse_send(stream_id, {
                "type": "token",
                "content": token.text
            })

    def sse_send(self, stream_id, data):
        """SSE 格式推送"""
        # 找到对应的 SSE 连接
        connection = self.connections.get(stream_id)
        if connection:
            connection.write(f"data: {json.dumps(data)}\n\n")
            connection.flush()
```

**客户端：**

```javascript
const eventSource = new EventSource('/api/v1/chat/stream?request_id=xxx');

eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    if (data.type === 'token') {
        appendToOutput(data.content);  // 逐 token 显示
    } else if (data.type === 'done') {
        eventSource.close();
    }
};
```

### GPU 资源优化：3 种手段

**1. 模型量化（INT8/INT4）**

```
原始 FP16：140GB 权重 → 需要 2 张 A100（80GB×2=160GB）
INT8 量化：70GB 权重 → 需要 1 张 A100
INT4 量化：35GB 权重 → 需要 1 张 A100（但精度下降约 5%）

效果：GPU 数量减少 50-75%
```

**2. Speculative Decoding（投机解码）**

```
用小模型（7B）快速生成 5 个 candidate tokens
用大模型（70B）并行验证这 5 个 tokens 是否正确

如果 5 个都正确 → 等效吞吐提升 5 倍
如果只有 2 个正确 → 等效吞吐提升 2 倍

平均效果：吞吐提升 2-3 倍
```

**3. 请求排队 + 动态扩缩容**

```python
class GPUScaler:
    def calculate_required_servers(self, current_qps, avg_duration):
        """
        计算需要的服务器数量
        
        并发数 = QPS × 平均持续时间（Little's Law）
        单机并发 = max_batch_size × GPU_count / batch
        """
        concurrent_requests = current_qps * avg_duration  # 5000 × 25 = 125000
        
        # Continuous batching + PagedAttention
        # 单机并发能力：约 200（短请求）或 20（长请求）
        # 混合场景：约 50 并发/机
        servers_needed = concurrent_requests // 50
        
        # 量化后：INT8 模型，单机并发提升到 100
        servers_needed = concurrent_requests // 100  # 125000/100 = 1250
        
        # Speculative decoding：吞吐提升 2 倍
        servers_needed = servers_needed // 2  # 625
        
        # 实际峰值只持续 2 小时/天 → 动态扩缩容
        # 平峰期 QPS 约 500 → 需要 6250/5000 = 125 并发 → 2 台服务器
        # 峰值时临时扩到 625 台 → 但成本太高
        
        # 真实方案：请求排队，峰值排队 30 秒后处理
        # 排队后实际处理 QPS = 服务器容量 → 峰值排队可接受
```

**最终方案：量化 + 排队 + 分级服务**

| 服务等级 | GPU 数量 | 最大 QPS | 延迟 | 月成本 |
|---------|---------|---------|------|--------|
| Premium（实时） | 20台 | 1000 | <500ms首token | ¥80万 |
| Standard（排队） | 5台 | 250 | <30s排队+500ms | ¥20万 |
| Batch（异步） | 2台 | 不限 | 5min-1h | ¥8万 |

总计：约 ¥28 万/月（< ¥50 万预算）

## 常见陷阱（深度分析）

### 陷阱 1：Static batching 等凑满再推理

**后果：** 低峰期等 5 秒凑够 20 个请求 → 首 token 延迟 5 秒 → 用户看到空白 5 秒后才开始输出 → 体验极差。

**解决方案：** Continuous batching，有请求就推理，不等凑满。

### 陷阱 2：KV Cache 预分配最大长度

**后果：** 100 tokens 的请求也占 16GB → 8 张 A100 只能并发 20 个请求 → GPU 利用率极低。

**解决方案：** PagedAttention 按需分配，100 tokens 只占 400KB → 并发提升到 200+。

### 陷阱 3：非流式输出等全部生成完

**后果：** 用户等 25 秒才看到第一个字 → 体验极差（像下载文件而非对话）。

**解决方案：** SSE 流式输出，每 token 推送。

### 陷阱 4：GPU 不做动态扩缩容

**后果：** 按峰值配置 625 台 GPU → 平峰期利用率 < 5% → ¥1.25 亿/月浪费。

**解决方案：** 分级服务 + 排队 + 云上动态扩缩容（峰值时临时加机器，5 分钟弹性扩容）。

## 延伸思考

- **多 LoRA 服务**：一个 GPU 上同时服务多个定制化模型（不同 LoRA adapter），如何高效切换？
- **分布式推理**：70B 模型切分到 4 台机器上（tensor parallelism），延迟如何？通信开销如何？
- **Prompt Cache**：相同 system prompt 的请求共享 KV Cache prefix，减少重复计算。如何管理共享 prefix 的生命周期？