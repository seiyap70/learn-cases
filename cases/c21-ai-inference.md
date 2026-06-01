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

### 数据库设计：模型注册与推理任务管理

推理服务需要持久化模型元数据、推理任务状态和 GPU 节点信息，以支持调度、监控和故障恢复。

```sql
-- 模型注册表：管理所有可用的模型版本与配置
CREATE TABLE model_registry (
    model_id         BIGINT PRIMARY KEY AUTO_INCREMENT,
    model_name       VARCHAR(128) NOT NULL COMMENT '模型名称，如 llama-70b',
    version          VARCHAR(32)  NOT NULL COMMENT '语义化版本号，如 v2.1.0',
    quantization     ENUM('fp16','int8','int4_awq','int4_gptq') NOT NULL DEFAULT 'fp16',
    weights_path     VARCHAR(512) NOT NULL COMMENT '模型权重文件路径（S3/OSS）',
    weights_size_gb  DECIMAL(8,2) NOT NULL COMMENT '权重文件大小',
    num_params_b     DECIMAL(6,1) NOT NULL COMMENT '参数量（十亿）',
    num_layers       INT          NOT NULL COMMENT 'Transformer 层数',
    hidden_dim       INT          NOT NULL COMMENT '隐藏层维度',
    max_seq_len      INT          NOT NULL COMMENT '最大序列长度',
    min_gpu_count    INT          NOT NULL COMMENT '最少 GPU 数量（模型并行）',
    loading_time_sec INT          NOT NULL COMMENT '模型加载耗时（秒）',
    perplexity_delta DECIMAL(6,4) DEFAULT 0 COMMENT '相对 FP16 的困惑度增量',
    status           ENUM('active','deprecated','testing') NOT NULL DEFAULT 'testing',
    created_at       DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_model_version (model_name, version, quantization)
) COMMENT '模型注册表';

-- GPU 节点表：记录集群中每台 GPU 机器的状态
CREATE TABLE gpu_node (
    node_id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    instance_id      VARCHAR(64)  NOT NULL COMMENT '云厂商实例 ID',
    gpu_type         ENUM('A100_80G','A100_40G','L4_24G','H100_80G') NOT NULL,
    gpu_count        INT          NOT NULL COMMENT '单机 GPU 卡数',
    total_vram_gb    DECIMAL(8,2) NOT NULL COMMENT '总显存（GB）',
    available_vram_gb DECIMAL(8,2) NOT NULL COMMENT '可用显存（GB）',
    loaded_models    JSON         NOT NULL COMMENT '已加载模型列表 [{model_id, vram_used_gb}]',
    running_requests INT          NOT NULL DEFAULT 0 COMMENT '当前运行中的请求数',
    max_requests     INT          NOT NULL DEFAULT 160 COMMENT '最大并发请求数（= gpu_count × 20）',
    status           ENUM('idle','serving','draining','loading','failed') NOT NULL DEFAULT 'idle',
    cpu_usage_pct    DECIMAL(5,2) DEFAULT 0,
    gpu_usage_pct    DECIMAL(5,2) DEFAULT 0 COMMENT 'GPU 计算利用率',
    last_heartbeat   DATETIME     NOT NULL COMMENT '最后心跳时间',
    launched_at      DATETIME     NOT NULL COMMENT '实例启动时间',
    hourly_cost_usd  DECIMAL(8,4) NOT NULL COMMENT '每小时费用',
    INDEX idx_status (status),
    INDEX idx_heartbeat (last_heartbeat)
) COMMENT 'GPU 节点表';

-- 推理任务表：跟踪每个推理请求的全生命周期
CREATE TABLE inference_job (
    job_id           BIGINT PRIMARY KEY AUTO_INCREMENT,
    request_id       VARCHAR(64)  NOT NULL COMMENT '外部请求 ID',
    user_id          VARCHAR(64)  NOT NULL,
    user_tier        ENUM('premium','standard','free') NOT NULL DEFAULT 'free',
    model_id         BIGINT       NOT NULL COMMENT '使用的模型 ID',
    node_id          BIGINT       NULL COMMENT '分配的 GPU 节点',
    input_tokens     INT          NOT NULL COMMENT '输入 token 数',
    max_output_tokens INT         NOT NULL COMMENT '最大输出 token 数',
    actual_output_tokens INT      NULL COMMENT '实际输出 token 数',
    status           ENUM('queued','prefilling','decoding','completed','failed','cancelled','preempted') NOT NULL DEFAULT 'queued',
    priority         INT          NOT NULL DEFAULT 0 COMMENT '调度优先级，越高越优先',
    prefill_latency_ms INT        NULL COMMENT '首 token 延迟（ms）',
    total_latency_ms INT          NULL COMMENT '总延迟（ms）',
    tokens_per_second DECIMAL(8,2) NULL COMMENT '生成速度',
    kv_cache_pages   INT          NULL COMMENT '使用的 KV Cache 页数',
    error_code       VARCHAR(32)  NULL COMMENT '错误码：OOM/CRASH/TIMEOUT/PREEMPTED',
    error_message    TEXT         NULL,
    cost_usd         DECIMAL(10,6) NULL COMMENT '本次请求成本',
    created_at       DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    started_at       DATETIME     NULL,
    completed_at     DATETIME     NULL,
    INDEX idx_user_status (user_id, status),
    INDEX idx_node_status (node_id, status),
    INDEX idx_created (created_at)
) COMMENT '推理任务表';

-- GPU 利用率时序表：用于容量规划和成本分析
CREATE TABLE gpu_metrics (
    id               BIGINT PRIMARY KEY AUTO_INCREMENT,
    node_id          BIGINT       NOT NULL,
    timestamp        DATETIME     NOT NULL COMMENT '采集时间',
    gpu_util_pct     DECIMAL(5,2) NOT NULL COMMENT 'GPU 计算利用率',
    vram_used_pct    DECIMAL(5,2) NOT NULL COMMENT '显存使用率',
    running_requests INT          NOT NULL,
    tokens_per_sec   DECIMAL(8,2) NOT NULL COMMENT '实际吞吐量',
    power_draw_w     DECIMAL(6,1) NOT NULL COMMENT '功耗（瓦）',
    INDEX idx_node_time (node_id, timestamp)
) COMMENT 'GPU 利用率时序表';
```

**设计要点说明：**

- `model_registry` 支持同一模型的多种量化版本共存，调度时根据用户场景选择合适的量化版本
- `gpu_node.loaded_models` 使用 JSON 存储已加载模型列表，支持单机加载多个小模型或单一模型并行
- `inference_job` 记录完整的请求生命周期，`error_code` 字段区分 OOM、崩溃、超时、抢占等不同失败原因
- `gpu_metrics` 按 10 秒粒度采集，用于趋势分析、容量规划和异常检测

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

    def preempt_lowest_priority(self):
        """抢占最低优先级请求的 KV Cache 以释放显存"""
        if not self.request_pages:
            raise OOMError("No KV Cache to preempt — all GPU memory exhausted")

        # 找到优先级最低的请求（最早加入的免费用户请求）
        lowest_priority_id = min(
            self.request_pages.keys(),
            key=lambda rid: self._get_request_priority(rid)
        )
        # 抢占：释放 KV Cache，将请求放回等待队列
        self.free(lowest_priority_id)
        self.requeue_request(lowest_priority_id)  # 重新入队，稍后重试

    def get_memory_usage(self):
        """返回当前内存使用情况，用于监控"""
        used = self.total_pages - len(self.free_pages)
        return {
            "total_pages": self.total_pages,
            "used_pages": used,
            "free_pages": len(self.free_pages),
            "utilization_pct": round(used / self.total_pages * 100, 2),
            "active_requests": len(self.request_pages),
        }
```

**PagedAttention 内存碎片分析：**

```
场景：8×A100 640GB，INT8 量化模型 70GB 权重

可用 KV Cache 显存 = 640 - 70 - 10(临时) = 560GB
单页 KV Cache 大小 = 16 tokens × 2 × 8192 × 2 bytes × 80 层 / 1e9 ≈ 0.042 GB/页
总页数 = 560 / 0.042 ≈ 13,333 页

对比内存浪费：
  传统预分配（max_seq_len=2048）：
    单请求分配 2048/16=128 页，实际平均用 500/16≈32 页
    浪费率 = (128-32)/128 = 75%
    最大并发 = 13333 / 128 ≈ 104 请求

  PagedAttention：
    单请求按需分配 ≈ 32 页（平均）
    碎片率 < 4%（页级管理，仅最后 1 页可能不满）
    最大并发 = 13333 / 32 ≈ 416 请求（提升 4 倍！）
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

    def forward_step(self, requests):
        """对所有运行中的请求执行一次前向传播"""
        if not requests:
            return

        # 合并所有请求的输入到同一个 batch 张量
        batch_input_ids = torch.tensor([r.get_next_input_id() for r in requests])

        # 执行前向传播（所有请求并行计算 1 个 token）
        with torch.no_grad():
            logits = self.model(batch_input_ids)

        # 为每个请求采样下一个 token
        for i, r in enumerate(requests):
            next_token = self.sample_token(logits[i], r.sampling_params)
            r.append_token(next_token)
            if next_token == self.eos_token_id or len(r.generated_tokens) >= r.max_tokens:
                r.is_done = True
```

**性能对比：**

| 指标 | 静态 Batch | 连续 Batch |
|------|-----------|-----------|
| GPU 利用率 | 40-50% | 80-90% |
| 平均等待时间 | 峰值时 30s | 峰值时 5s |
| 吞吐量 (tokens/s) | 250 | 450 |
| 有效并发 | 20（受限于最慢请求） | 40（动态加入/移出） |

### 完整批处理引擎实现

以下是一个可运行的连续批处理引擎核心实现，包含请求管理、调度循环和流式输出。

```python
import time
import threading
import math
from collections import deque
from dataclasses import dataclass, field
from typing import Optional
from enum import Enum


class RequestStatus(Enum):
    QUEUED = "queued"
    PREFILLING = "prefilling"
    DECODING = "decoding"
    COMPLETED = "completed"
    FAILED = "failed"
    PREEMPTED = "preempted"


@dataclass
class SamplingParams:
    temperature: float = 1.0
    top_p: float = 0.9
    top_k: int = 50
    max_tokens: int = 512
    stop_tokens: list = field(default_factory=lambda: ["</s>"])


@dataclass
class InferenceRequest:
    request_id: str
    prompt_token_ids: list  # 输入 token IDs
    sampling_params: SamplingParams
    user_tier: str = "standard"
    priority: int = 0  # 越高越优先
    status: RequestStatus = RequestStatus.QUEUED
    generated_tokens: list = field(default_factory=list)
    kv_cache_pages: list = field(default_factory=list)
    enqueue_time: float = field(default_factory=time.time)
    start_time: Optional[float] = None
    end_time: Optional[float] = None
    first_token_time: Optional[float] = None
    error: Optional[str] = None

    @property
    def is_done(self):
        return self.status in (
            RequestStatus.COMPLETED,
            RequestStatus.FAILED,
            RequestStatus.PREEMPTED,
        )

    @property
    def current_token(self):
        if self.generated_tokens:
            return self.generated_tokens[-1]
        return None


class ContinuousBatchEngine:
    """连续批处理推理引擎——完整实现"""

    def __init__(
        self,
        model,
        kv_cache_manager,
        max_batch_size=20,
        max_waiting_queue=1000,
        prefill_chunk_size=512,
    ):
        self.model = model
        self.kv_cache_manager = kv_cache_manager
        self.max_batch_size = max_batch_size
        self.max_waiting_queue = max_waiting_queue
        self.prefill_chunk_size = prefill_chunk_size  # Chunked Prefill 块大小

        self.waiting_queue = deque()
        self.running_requests: list[InferenceRequest] = []
        self.token_streams: dict[str, deque] = {}  # request_id → token 流

        self._lock = threading.Lock()
        self._stop_event = threading.Event()
        self._engine_thread = None

        # 统计指标
        self.stats = {
            "total_requests": 0,
            "completed_requests": 0,
            "preempted_requests": 0,
            "total_tokens_generated": 0,
            "avg_first_token_latency_ms": 0,
            "avg_total_latency_ms": 0,
        }

    def start(self):
        """启动引擎调度线程"""
        self._engine_thread = threading.Thread(target=self._run_loop, daemon=True)
        self._engine_thread.start()

    def stop(self):
        """停止引擎"""
        self._stop_event.set()
        if self._engine_thread:
            self._engine_thread.join(timeout=10)

    def submit(self, request: InferenceRequest) -> deque:
        """提交推理请求，返回 token 流（消费者可迭代读取）"""
        with self._lock:
            if len(self.waiting_queue) >= self.max_waiting_queue:
                raise QueueOverflowError(
                    f"Waiting queue full ({self.max_waiting_queue}), "
                    f"request rejected"
                )
            request.status = RequestStatus.QUEUED
            request.enqueue_time = time.time()
            self.waiting_queue.append(request)
            self.token_streams[request.request_id] = deque()
            self.stats["total_requests"] += 1

        return self.token_streams[request.request_id]

    def _run_loop(self):
        """引擎主循环：每次迭代处理 1 个 token"""
        while not self._stop_event.is_set():
            # 1. 移除已完成的请求
            self._evict_completed()

            # 2. 从等待队列加入新请求（直到 batch 满）
            self._admit_new_requests()

            if not self.running_requests:
                time.sleep(0.001)  # 无请求时短暂休眠，避免 CPU 空转
                continue

            # 3. 执行一次前向传播（生成 1 token）
            self._forward_step()

    def _evict_completed(self):
        """移除已完成的请求，释放资源"""
        completed = [r for r in self.running_requests if r.is_done]
        for r in completed:
            self.running_requests.remove(r)
            self.kv_cache_manager.free(r.request_id)
            r.end_time = time.time()

            # 在 token 流末尾放入终止标记
            self.token_streams[r.request_id].append(None)

            # 更新统计
            if r.status == RequestStatus.COMPLETED:
                self.stats["completed_requests"] += 1
                self.stats["total_tokens_generated"] += len(r.generated_tokens)
                total_ms = (r.end_time - r.start_time) * 1000
                first_ms = (r.first_token_time - r.start_time) * 1000
                self._update_avg_latency(total_ms, first_ms)
            elif r.status == RequestStatus.PREEMPTED:
                self.stats["preempted_requests"] += 1

    def _admit_new_requests(self):
        """从等待队列中选取新请求加入 batch"""
        while (
            len(self.running_requests) < self.max_batch_size
            and self.waiting_queue
        ):
            # 按优先级选取（优先级高的先处理）
            request = self._select_next_request()
            if request is None:
                break

            # 分配 KV Cache（含 Chunked Prefill 策略）
            try:
                # 初始只为 prefill chunk 分配页，解码阶段按需追加
                initial_tokens = min(
                    len(request.prompt_token_ids), self.prefill_chunk_size
                )
                pages = self.kv_cache_manager.allocate(
                    request.request_id,
                    initial_tokens,
                )
                request.kv_cache_pages = pages
                request.status = RequestStatus.PREFILLING
                request.start_time = time.time()
                self.running_requests.append(request)
            except OOMError:
                # 显存不足：抢占低优先级请求
                if self._try_preempt_for(request):
                    continue  # 抢占成功，重试分配
                else:
                    # 无法抢占，将请求放回队列头部
                    self.waiting_queue.appendleft(request)
                    break

    def _select_next_request(self) -> Optional[InferenceRequest]:
        """从等待队列中按优先级选取下一个请求"""
        if not self.waiting_queue:
            return None

        # 简单策略：遍历队列找最高优先级（生产环境用堆优化）
        best_idx = 0
        best_priority = -1
        for i, r in enumerate(self.waiting_queue):
            # 等待超时提升优先级（防止饥饿）
            wait_time = time.time() - r.enqueue_time
            effective_priority = r.priority + int(wait_time / 5)
            if effective_priority > best_priority:
                best_priority = effective_priority
                best_idx = i

        request = self.waiting_queue[best_idx]
        del self.waiting_queue[best_idx]
        return request

    def _try_preempt_for(self, incoming_request) -> bool:
        """尝试抢占低优先级请求为高优先级请求腾出空间"""
        candidates = [
            r for r in self.running_requests
            if r.priority < incoming_request.priority
        ]
        if not candidates:
            return False

        # 抢占优先级最低的请求
        victim = min(candidates, key=lambda r: r.priority)
        victim.status = RequestStatus.PREEMPTED
        # 将被抢占的请求重新入队
        self.waiting_queue.appendleft(victim)
        return True

    def _forward_step(self):
        """对所有运行中的请求执行一次前向传播"""
        for r in self.running_requests:
            # Chunked Prefill：长输入分块处理，避免阻塞解码请求
            if r.status == RequestStatus.PREFILLING:
                remaining = len(r.prompt_token_ids) - len(r.kv_cache_pages) * 16
                if remaining <= 0:
                    r.status = RequestStatus.DECODING
                # Prefill 阶段：每次处理 prefill_chunk_size 个 token

            elif r.status == RequestStatus.DECODING:
                # 解码阶段：生成 1 个 token
                pass

        # 批量前向传播
        with torch.no_grad():
            outputs = self.model.forward_batch(self.running_requests)

        # 处理每个请求的输出
        for r in self.running_requests:
            token = outputs[r.request_id]
            if token is None:
                r.status = RequestStatus.COMPLETED
                continue

            r.generated_tokens.append(token)
            if r.first_token_time is None:
                r.first_token_time = time.time()

            # 推送到 token 流
            self.token_streams[r.request_id].append(token)

            # 检查停止条件
            if (token in r.sampling_params.stop_tokens
                    or len(r.generated_tokens) >= r.sampling_params.max_tokens):
                r.status = RequestStatus.COMPLETED

    def _update_avg_latency(self, total_ms, first_ms):
        """滑动平均更新延迟指标"""
        alpha = 0.1  # 平滑因子
        if self.stats["avg_total_latency_ms"] == 0:
            self.stats["avg_total_latency_ms"] = total_ms
            self.stats["avg_first_token_latency_ms"] = first_ms
        else:
            self.stats["avg_total_latency_ms"] = (
                (1 - alpha) * self.stats["avg_total_latency_ms"]
                + alpha * total_ms
            )
            self.stats["avg_first_token_latency_ms"] = (
                (1 - alpha) * self.stats["avg_first_token_latency_ms"]
                + alpha * first_ms
            )

    def get_stats(self):
        """返回引擎统计信息"""
        return {
            **self.stats,
            "running_requests": len(self.running_requests),
            "waiting_requests": len(self.waiting_queue),
            "kv_cache_usage": self.kv_cache_manager.get_memory_usage(),
        }


class OOMError(Exception):
    pass


class QueueOverflowError(Exception):
    pass
```

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

### 模型版本管理与金丝雀部署

推理服务的模型更新是一个高风险操作——新版本可能存在精度退化、延迟劣化甚至生成乱码等问题。金丝雀部署（Canary Deployment）通过逐步切流的方式，在真实流量中验证新模型的输出质量，一旦检测到质量退化就自动回滚。

#### 模型版本注册与追踪

```python
import hashlib
import json
from datetime import datetime
from enum import Enum


class ModelVersionStatus(Enum):
    DRAFT = "draft"           # 开发中，未发布
    CANARY = "canary"         # 金丝雀发布中
    PROMOTING = "promoting"   # 正在逐步放量
    STABLE = "stable"         # 稳定版本，全量服务
    DEPRECATED = "deprecated" # 已废弃，等待下线
    ROLLED_BACK = "rolled_back"  # 已回滚


@dataclass
class ModelVersion:
    """模型版本记录"""
    version_id: str           # 版本唯一标识，如 "llama-70b-v2.3.1"
    model_name: str           # 模型名称
    semver: str               # 语义化版本号，如 "2.3.1"
    quantization: str         # 量化方案
    weights_path: str         # 权重文件路径
    weights_checksum: str     # 权重文件 SHA256 校验和
    weights_size_gb: float    # 权重文件大小
    config_hash: str          # 模型配置哈希（确保配置与权重匹配）
    parent_version: str       # 父版本（基于哪个版本训练/微调的）
    status: ModelVersionStatus = ModelVersionStatus.DRAFT
    canary_percentage: float = 0.0  # 金丝雀流量百分比
    created_at: datetime = None
    promoted_at: datetime = None     # 升级为稳定版的时间
    rolled_back_at: datetime = None  # 回滚时间
    quality_metrics: dict = field(default_factory=dict)  # 质量指标快照

    def compute_checksum(self, weights_file_path: str) -> str:
        """计算权重文件的 SHA256 校验和，用于检测权重损坏"""
        sha256 = hashlib.sha256()
        with open(weights_file_path, "rb") as f:
            for chunk in iter(lambda: f.read(8192), b""):
                sha256.update(chunk)
        return sha256.hexdigest()


class ModelVersionRegistry:
    """模型版本注册中心——管理模型的全生命周期版本"""

    def __init__(self, db_session):
        self.db = db_session
        self._version_cache = {}  # model_name → {semver → ModelVersion}

    def register_version(self, version: ModelVersion) -> str:
        """注册新模型版本"""
        # 1. 校验权重文件完整性
        actual_checksum = version.compute_checksum(version.weights_path)
        if actual_checksum != version.weights_checksum:
            raise ValueError(
                f"权重文件校验失败：期望 {version.weights_checksum}，"
                f"实际 {actual_checksum}"
            )

        # 2. 同一模型不能同时存在多个金丝雀版本
        existing_canary = self._find_canary(version.model_name)
        if existing_canary:
            raise ValueError(
                f"模型 {version.model_name} 已有金丝雀版本 "
                f"{existing_canary.version_id}，请先处理"
            )

        # 3. 持久化到数据库
        self.db.execute("""
            INSERT INTO model_version_history
            (version_id, model_name, semver, quantization, weights_path,
             weights_checksum, status, parent_version, canary_percentage, created_at)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (version.version_id, version.model_name, version.semver,
              version.quantization, version.weights_path, version.weights_checksum,
              version.status.value, version.parent_version, 0.0, datetime.utcnow()))
        self.db.commit()

        # 4. 更新缓存
        self._version_cache.setdefault(version.model_name, {})[version.semver] = version
        return version.version_id

    def get_stable_version(self, model_name: str) -> Optional[ModelVersion]:
        """获取当前稳定版本"""
        versions = self._version_cache.get(model_name, {})
        for v in versions.values():
            if v.status == ModelVersionStatus.STABLE:
                return v
        return None

    def get_canary_version(self, model_name: str) -> Optional[ModelVersion]:
        """获取当前金丝雀版本"""
        return self._find_canary(model_name)

    def _find_canary(self, model_name: str) -> Optional[ModelVersion]:
        versions = self._version_cache.get(model_name, {})
        for v in versions.values():
            if v.status in (ModelVersionStatus.CANARY, ModelVersionStatus.PROMOTING):
                return v
        return None

    def list_versions(self, model_name: str) -> list[ModelVersion]:
        """列出模型的所有版本（按创建时间倒序）"""
        versions = self._version_cache.get(model_name, {}).values()
        return sorted(versions, key=lambda v: v.created_at, reverse=True)

    def deprecate_version(self, version_id: str):
        """废弃指定版本"""
        version = self._find_version(version_id)
        if version and version.status == ModelVersionStatus.STABLE:
            raise ValueError("不能直接废弃稳定版本，请先发布新稳定版")
        if version:
            version.status = ModelVersionStatus.DEPRECATED
            version.canary_percentage = 0.0
            self._persist_status_change(version)
```

#### 金丝雀部署控制器

金丝雀部署的核心是逐步放量：5% → 25% → 100%。每一阶段都需要监控质量指标，只有指标达标才能进入下一阶段。

```python
class CanaryDeploymentController:
    """金丝雀部署控制器——逐步放量并自动质量保障"""

    # 金丝雀阶段配置：每个阶段的流量百分比和观察时长
    CANARY_STAGES = [
        {"percentage": 0.05, "duration_sec": 600, "label": "5% 金丝雀验证"},    # 5% 流量，观察 10 分钟
        {"percentage": 0.25, "duration_sec": 1800, "label": "25% 扩大验证"},     # 25% 流量，观察 30 分钟
        {"percentage": 1.00, "duration_sec": 0, "label": "100% 全量发布"},       # 全量
    ]

    # 质量阈值——任一指标超标即触发回滚
    QUALITY_THRESHOLDS = {
        "perplexity_increase_pct": 5.0,    # 困惑度增加不超过 5%
        "error_rate_pct": 1.0,             # 错误率不超过 1%
        "p99_latency_increase_pct": 20.0,  # P99 延迟增加不超过 20%
        "empty_output_rate_pct": 0.5,      # 空输出率不超过 0.5%
        "repetition_rate_pct": 2.0,        # 重复生成率不超过 2%
    }

    def __init__(
        self,
        registry: ModelVersionRegistry,
        metrics_collector: "QualityMetricsCollector",
        gpu_scheduler: "GPUScheduler",
    ):
        self.registry = registry
        self.metrics = metrics_collector
        self.scheduler = gpu_scheduler
        self._active_deployments = {}  # model_name → CanaryDeploymentState

    def start_canary(
        self, model_name: str, new_version_id: str, initial_percentage: float = 0.05
    ):
        """启动金丝雀部署"""
        stable = self.registry.get_stable_version(model_name)
        new_version = self.registry._find_version(new_version_id)

        if not stable:
            raise ValueError(f"模型 {model_name} 没有稳定版本，无法金丝雀部署")

        # 基线指标：记录稳定版本当前的质量指标作为对比基准
        baseline_metrics = self.metrics.get_baseline_metrics(model_name)

        state = CanaryDeploymentState(
            model_name=model_name,
            stable_version_id=stable.version_id,
            canary_version_id=new_version_id,
            current_stage_idx=0,
            current_percentage=initial_percentage,
            baseline_metrics=baseline_metrics,
            stage_started_at=datetime.utcnow(),
        )
        self._active_deployments[model_name] = state

        # 更新版本状态
        new_version.status = ModelVersionStatus.CANARY
        new_version.canary_percentage = initial_percentage
        self.registry._persist_status_change(new_version)

        # 为金丝雀版本加载模型到 GPU（至少 1 个节点）
        self._ensure_canary_nodes(model_name, new_version, min_nodes=1)

        self._log_canary_event(model_name, "CANARY_STARTED", {
            "stable": stable.version_id,
            "canary": new_version_id,
            "percentage": initial_percentage,
        })

    def check_and_advance(self, model_name: str):
        """检查金丝雀质量并决定是否推进到下一阶段

        此方法由定时任务每分钟调用一次。
        """
        state = self._active_deployments.get(model_name)
        if not state:
            return

        stage = self.CANARY_STAGES[state.current_stage_idx]
        elapsed = (datetime.utcnow() - state.stage_started_at).total_seconds()

        # 1. 收集金丝雀版本的质量指标
        canary_metrics = self.metrics.collect_canary_metrics(
            model_name, state.canary_version_id
        )

        # 2. 与基线对比，检查质量是否达标
        quality_result = self._evaluate_quality(
            state.baseline_metrics, canary_metrics
        )

        if not quality_result.passed:
            # 质量不达标 → 自动回滚
            self._rollback(model_name, state, quality_result)
            return

        # 3. 观察时长不足 → 继续观察
        if elapsed < stage["duration_sec"]:
            self._log_canary_event(model_name, "OBSERVING", {
                "stage": stage["label"],
                "elapsed_sec": elapsed,
                "remaining_sec": stage["duration_sec"] - elapsed,
                "quality": quality_result.details,
            })
            return

        # 4. 观察期满且质量达标 → 推进到下一阶段
        next_stage_idx = state.current_stage_idx + 1
        if next_stage_idx >= len(self.CANARY_STAGES):
            # 已是最后阶段 → 全量发布
            self._promote_to_stable(model_name, state)
        else:
            # 进入下一阶段
            self._advance_stage(model_name, state, next_stage_idx)

    def _evaluate_quality(self, baseline: dict, canary: dict) -> "QualityResult":
        """评估金丝雀版本的质量指标"""
        violations = []

        # 困惑度对比
        if baseline.get("perplexity") and canary.get("perplexity"):
            increase_pct = (
                (canary["perplexity"] - baseline["perplexity"])
                / baseline["perplexity"] * 100
            )
            if increase_pct > self.QUALITY_THRESHOLDS["perplexity_increase_pct"]:
                violations.append(
                    f"困惑度增加 {increase_pct:.1f}%，超过阈值 "
                    f"{self.QUALITY_THRESHOLDS['perplexity_increase_pct']}%"
                )

        # 错误率
        if canary.get("error_rate_pct", 0) > self.QUALITY_THRESHOLDS["error_rate_pct"]:
            violations.append(
                f"错误率 {canary['error_rate_pct']:.2f}%，超过阈值 "
                f"{self.QUALITY_THRESHOLDS['error_rate_pct']}%"
            )

        # P99 延迟
        if baseline.get("p99_latency_ms") and canary.get("p99_latency_ms"):
            latency_increase_pct = (
                (canary["p99_latency_ms"] - baseline["p99_latency_ms"])
                / baseline["p99_latency_ms"] * 100
            )
            if latency_increase_pct > self.QUALITY_THRESHOLDS["p99_latency_increase_pct"]:
                violations.append(
                    f"P99 延迟增加 {latency_increase_pct:.1f}%，超过阈值 "
                    f"{self.QUALITY_THRESHOLDS['p99_latency_increase_pct']}%"
                )

        # 空输出率
        if canary.get("empty_output_rate_pct", 0) > self.QUALITY_THRESHOLDS["empty_output_rate_pct"]:
            violations.append(
                f"空输出率 {canary['empty_output_rate_pct']:.2f}%，超过阈值"
            )

        # 重复生成率
        if canary.get("repetition_rate_pct", 0) > self.QUALITY_THRESHOLDS["repetition_rate_pct"]:
            violations.append(
                f"重复生成率 {canary['repetition_rate_pct']:.2f}%，超过阈值"
            )

        return QualityResult(
            passed=len(violations) == 0,
            violations=violations,
            details={
                "baseline": baseline,
                "canary": canary,
                "violations": violations,
            },
        )

    def _advance_stage(self, model_name: str, state: "CanaryDeploymentState",
                       next_stage_idx: int):
        """推进到下一金丝雀阶段"""
        next_stage = self.CANARY_STAGES[next_stage_idx]
        state.current_stage_idx = next_stage_idx
        state.current_percentage = next_stage["percentage"]
        state.stage_started_at = datetime.utcnow()

        # 更新版本的流量百分比
        canary_version = self.registry._find_version(state.canary_version_id)
        canary_version.status = ModelVersionStatus.PROMOTING
        canary_version.canary_percentage = next_stage["percentage"]
        self.registry._persist_status_change(canary_version)

        # 确保有足够的 GPU 节点运行金丝雀版本
        self._ensure_canary_nodes(model_name, canary_version,
                                  min_nodes=max(1, int(next_stage["percentage"] * 10)))

        self._log_canary_event(model_name, "STAGE_ADVANCED", {
            "new_stage": next_stage["label"],
            "percentage": next_stage["percentage"],
        })

    def _promote_to_stable(self, model_name: str, state: "CanaryDeploymentState"):
        """金丝雀验证通过 → 升级为稳定版本"""
        # 1. 将旧稳定版标记为废弃
        old_stable = self.registry._find_version(state.stable_version_id)
        old_stable.status = ModelVersionStatus.DEPRECATED
        self.registry._persist_status_change(old_stable)

        # 2. 将金丝雀版本升级为稳定版
        new_stable = self.registry._find_version(state.canary_version_id)
        new_stable.status = ModelVersionStatus.STABLE
        new_stable.canary_percentage = 1.0
        new_stable.promoted_at = datetime.utcnow()
        self.registry._persist_status_change(new_stable)

        # 3. 清理旧版本占用的 GPU 节点
        self._drain_old_version_nodes(model_name, state.stable_version_id)

        # 4. 移除部署状态
        del self._active_deployments[model_name]

        self._log_canary_event(model_name, "PROMOTED_TO_STABLE", {
            "old_stable": state.stable_version_id,
            "new_stable": state.canary_version_id,
        })

    def _rollback(self, model_name: str, state: "CanaryDeploymentState",
                  quality_result: "QualityResult"):
        """质量不达标 → 自动回滚到稳定版本"""
        # 1. 立即将金丝雀版本流量归零
        canary_version = self.registry._find_version(state.canary_version_id)
        canary_version.status = ModelVersionStatus.ROLLED_BACK
        canary_version.canary_percentage = 0.0
        canary_version.rolled_back_at = datetime.utcnow()
        self.registry._persist_status_change(canary_version)

        # 2. 将金丝雀版本的 GPU 节点标记为 draining
        self._drain_old_version_nodes(model_name, state.canary_version_id)

        # 3. 移除部署状态
        del self._active_deployments[model_name]

        self._log_canary_event(model_name, "ROLLED_BACK", {
            "canary_version": state.canary_version_id,
            "violations": quality_result.violations,
            "baseline_metrics": state.baseline_metrics,
        })

        # 4. 发送告警
        alert_ops_team(
            f"金丝雀部署回滚：{model_name} {state.canary_version_id}，"
            f"原因：{'; '.join(quality_result.violations)}"
        )

    def _ensure_canary_nodes(self, model_name: str, version: ModelVersion,
                             min_nodes: int):
        """确保有足够的 GPU 节点加载金丝雀版本"""
        current_nodes = self.scheduler.count_nodes_with_model(version.version_id)
        if current_nodes < min_nodes:
            for _ in range(min_nodes - current_nodes):
                node = self.scheduler.get_idle_node()
                if node:
                    node.load_model(version)

    def _drain_old_version_nodes(self, model_name: str, version_id: str):
        """排空指定版本的 GPU 节点"""
        nodes = self.scheduler.get_nodes_with_model(version_id)
        for node in nodes:
            node.drain()  # 不接受新请求，等现有请求完成后卸载模型

    def _log_canary_event(self, model_name: str, event_type: str, details: dict):
        """记录金丝雀部署事件"""
        self.db.execute("""
            INSERT INTO canary_deployment_log
            (model_name, event_type, details, timestamp)
            VALUES (?, ?, ?, ?)
        """, (model_name, event_type, json.dumps(details), datetime.utcnow()))
        self.db.commit()


@dataclass
class CanaryDeploymentState:
    """金丝雀部署状态"""
    model_name: str
    stable_version_id: str
    canary_version_id: str
    current_stage_idx: int
    current_percentage: float
    baseline_metrics: dict
    stage_started_at: datetime


@dataclass
class QualityResult:
    """质量评估结果"""
    passed: bool
    violations: list[str]
    details: dict
```

**金丝雀部署流程图：**

```
新版本注册 → 加载到 1 个 GPU 节点 → 5% 流量 → 观察 10 分钟
                                                    ↓
                                          质量达标？───否──→ 自动回滚（流量归零 + 排空节点）
                                            │
                                           是
                                            ↓
                                    25% 流量 → 观察 30 分钟
                                                    ↓
                                          质量达标？───否──→ 自动回滚
                                            │
                                           是
                                            ↓
                                    100% 全量发布 → 升级为稳定版本
                                                    ↓
                                          排空旧版本节点 → 部署完成
```

**金丝雀流量路由实现：**

```python
class CanaryTrafficRouter:
    """金丝雀流量路由器——根据流量百分比将请求分发到不同模型版本"""

    def route_request(self, request: InferenceRequest, model_name: str) -> str:
        """决定请求路由到哪个模型版本，返回 version_id"""
        state = self.canary_controller._active_deployments.get(model_name)

        if not state:
            # 无金丝雀部署 → 路由到稳定版本
            return self.registry.get_stable_version(model_name).version_id

        # 基于请求 ID 的哈希决定路由（确保同一用户/会话路由一致）
        route_key = self._compute_route_key(request)
        if route_key < state.current_percentage:
            # 路由到金丝雀版本
            return state.canary_version_id
        else:
            # 路由到稳定版本
            return state.stable_version_id

    def _compute_route_key(self, request: InferenceRequest) -> float:
        """计算请求的路由键（0~1 之间的均匀分布）

        使用用户 ID + 会话 ID 的哈希，确保：
        1. 同一用户的请求路由一致（避免同一对话中模型版本切换）
        2. 分布均匀（避免流量倾斜）
        """
        key = f"{request.user_id}:{request.session_id}"
        hash_val = int(hashlib.md5(key.encode()).hexdigest(), 16)
        return (hash_val % 10000) / 10000.0  # 归一化到 [0, 1)
```

### GPU 调度算法：多维度最优分配

GPU 调度不只是"有空闲就分配"，还需要考虑模型亲和性（已加载相同模型的节点优先）、显存碎片、GPU 利用率均衡等因素。

```python
class GPUScheduler:
    """多维度 GPU 调度器：综合考虑模型亲和性、负载均衡和显存碎片"""

    def __init__(self, node_repository, kv_cache_manager):
        self.nodes = node_repository          # GPU 节点仓库（DB 查询封装）
        self.kv_cache = kv_cache_manager
        self.scheduling_stats = {
            "affinity_hits": 0,    # 模型亲和命中次数
            "affinity_misses": 0,  # 需要加载模型的次数
            "preemptions": 0,      # 抢占次数
        }

    def select_node(self, request) -> Optional["GPUNode"]:
        """为推理请求选择最优 GPU 节点

        调度策略（按优先级）：
        1. 模型亲和性——已加载目标模型的节点优先（省去 20s 加载时间）
        2. 负载均衡——选择运行请求数最少的节点
        3. 显存充足——确保节点有足够显存分配 KV Cache
        """
        model_id = request.model_id
        required_vram = self._estimate_kv_cache_vram(request)

        # 第一轮：在已加载目标模型的节点中找
        affinity_nodes = self.nodes.find_by_loaded_model(model_id)
        for node in sorted(affinity_nodes, key=lambda n: n.running_requests):
            if (node.running_requests < node.max_requests
                    and node.available_vram_gb >= required_vram):
                self.scheduling_stats["affinity_hits"] += 1
                return node

        # 第二轮：所有有空闲容量的节点中找（需要加载模型）
        all_nodes = self.nodes.find_by_status("serving")
        for node in sorted(all_nodes, key=lambda n: n.running_requests):
            if (node.running_requests < node.max_requests
                    and node.available_vram_gb >= model_vram(model_id) + required_vram):
                self.scheduling_stats["affinity_misses"] += 1
                return node

        # 第三轮：所有节点都满——尝试抢占低优先级请求
        preempted = self._preempt_for_request(request)
        if preempted:
            self.scheduling_stats["preemptions"] += 1
            return preempted.node

        # 无法调度——触发扩容
        self._trigger_scale_up()
        return None

    def _estimate_kv_cache_vram(self, request) -> float:
        """估算请求所需的 KV Cache 显存（GB）"""
        # KV Cache 大小 = 2 × seq_len × hidden_dim × 2 bytes × num_layers
        total_tokens = request.input_tokens + request.max_output_tokens
        kv_cache_gb = 2 * total_tokens * 8192 * 2 * 80 / 1e9
        # 加 10% 安全余量
        return kv_cache_gb * 1.1

    def _preempt_for_request(self, incoming_request):
        """抢占策略：逐个抢占最低优先级的运行中请求，直到腾出足够空间"""
        # 按优先级升序排列当前运行的请求
        running = sorted(
            self._get_all_running_requests(),
            key=lambda r: (r.priority, r.enqueue_time)
        )
        freed_vram = 0.0
        needed_vram = self._estimate_kv_cache_vram(incoming_request)

        for victim in running:
            if victim.priority >= incoming_request.priority:
                break  # 不抢占同优先级或更高优先级的请求
            victim_vram = self._estimate_kv_cache_vram(victim)
            victim.status = RequestStatus.PREEMPTED
            self.kv_cache.free(victim.request_id)
            freed_vram += victim_vram
            if freed_vram >= needed_vram:
                return victim

        return None  # 抢占后仍不够空间

    def _trigger_scale_up(self):
        """触发自动扩容"""
        # 通知 Autoscaler 立即扩容 1 台
        self.autoscaler.emergency_scale_up(count=1)

    def rebalance(self):
        """定期重平衡：将请求从过载节点迁移到空闲节点

        迁移条件：
        - 源节点 GPU 利用率 > 90%
        - 目标节点 GPU 利用率 < 50%
        - 请求可安全迁移（正在 decoding 且 KV Cache 可重建）
        """
        overloaded = [n for n in self.nodes.get_all()
                      if n.running_requests > n.max_requests * 0.9]
        underloaded = [n for n in self.nodes.get_all()
                       if n.running_requests < n.max_requests * 0.5]

        for src in overloaded:
            for dst in underloaded:
                if src.running_requests <= src.max_requests * 0.8:
                    break
                # 迁移一个请求（需要重建 KV Cache，代价较高）
                candidate = self._find_migratable_request(src)
                if candidate and dst.available_vram_gb >= self._estimate_kv_cache_vram(candidate):
                    self._migrate_request(candidate, src, dst)
                    # 迁移后 dst 利用率更新
                    if dst.running_requests >= dst.max_requests * 0.5:
                        underloaded.remove(dst)
```

### 流式输出架构

推理结果需要实时推送给客户端，否则用户需要等待全部 token 生成完毕才能看到结果——对于 500 tokens 的输出，这意味着 25 秒的空白等待。流式输出通过逐 token 推送，将首 token 延迟控制在 500ms 以内，用户体验显著提升。

**SSE（Server-Sent Events）与 WebSocket 的选择：**

| 特性 | SSE | WebSocket |
|------|-----|-----------|
| 协议 | HTTP/1.1+ | 独立协议 |
| 方向 | 服务端→客户端单向 | 双向通信 |
| 断线重连 | 内置（自动重连+Last-Event-ID） | 需手动实现 |
| 代理兼容 | 普通 HTTP，CDN/代理友好 | 需特殊配置 |
| 适用场景 | 单向流式输出（LLM 生成） | 双向交互（对话、实时编辑） |
| 复杂度 | 低 | 中 |

**推荐策略：** 对于大多数 LLM 推理场景，SSE 是首选——简单、CDN 兼容、内置断线重连。只有在需要双向交互（如用户在生成过程中发送中止信号）时才用 WebSocket。

#### SSE 流式输出完整实现

```python
import asyncio
import json
import time
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from sse_starlette.sse import EventSourceResponse


app = FastAPI()


class SSETokenStreamer:
    """SSE 流式输出管理器——将推理引擎的 token 流转换为 SSE 事件"""

    def __init__(self, engine: ContinuousBatchEngine):
        self.engine = engine
        self.active_streams: dict[str, asyncio.Queue] = {}
        self._cleanup_task = None

    async def create_stream(self, request_id: str) -> asyncio.Queue:
        """创建一个异步队列作为 SSE 事件源"""
        queue = asyncio.Queue()
        self.active_streams[request_id] = queue
        return queue

    async def stream_to_sse(self, request_id: str, queue: asyncio.Queue):
        """从队列中读取 token 并生成 SSE 事件"""
        token_count = 0
        start_time = time.time()

        try:
            while True:
                # 从引擎的 token 流中读取（同步→异步桥接）
                token = await asyncio.wait_for(queue.get(), timeout=60.0)

                if token is None:
                    # 生成完成
                    elapsed = time.time() - start_time
                    yield {
                        "event": "done",
                        "data": json.dumps({
                            "request_id": request_id,
                            "total_tokens": token_count,
                            "total_time_sec": round(elapsed, 2),
                            "tokens_per_second": round(
                                token_count / elapsed, 2
                            ) if elapsed > 0 else 0,
                        }),
                    }
                    break

                token_count += 1
                yield {
                    "event": "token",
                    "data": json.dumps({
                        "request_id": request_id,
                        "token": token,
                        "index": token_count,
                    }),
                }

        except asyncio.TimeoutError:
            yield {
                "event": "error",
                "data": json.dumps({
                    "request_id": request_id,
                    "error": "timeout",
                    "message": "Token stream timeout after 60 seconds",
                }),
            }
        finally:
            self.active_streams.pop(request_id, None)

    def bridge_token_stream(self, request_id: str, token_deque: deque):
        """将引擎的同步 deque 转换为异步 Queue（后台线程运行）"""
        async_queue = self.active_streams.get(request_id)
        if async_queue is None:
            return

        while True:
            if token_deque:
                token = token_deque.popleft()
                asyncio.run_coroutine_threadsafe(
                    async_queue.put(token), asyncio.get_event_loop()
                )
                if token is None:
                    break
            else:
                time.sleep(0.001)  # 无 token 时短暂等待


# ---- FastAPI SSE 端点 ----

streamer = SSETokenStreamer(engine)


@app.post("/v1/completions/stream")
async def stream_completion(request: Request):
    """SSE 流式推理端点"""
    body = await request.json()
    user_tier = request.headers.get("X-User-Tier", "free")
    user_id = request.headers.get("X-User-Id", "anonymous")

    # 构造推理请求
    inference_req = InferenceRequest(
        request_id=f"req-{time.time_ns()}",
        prompt_token_ids=tokenize(body["prompt"]),
        sampling_params=SamplingParams(
            temperature=body.get("temperature", 1.0),
            top_p=body.get("top_p", 0.9),
            max_tokens=body.get("max_tokens", 512),
        ),
        user_tier=user_tier,
        priority={"premium": 10, "standard": 5, "free": 0}[user_tier],
    )

    # 提交到引擎并获取同步 token 流
    token_deque = engine.submit(inference_req)

    # 创建异步队列并启动桥接线程
    async_queue = await streamer.create_stream(inference_req.request_id)
    threading.Thread(
        target=streamer.bridge_token_stream,
        args=(inference_req.request_id, token_deque),
        daemon=True,
    ).start()

    # 返回 SSE 流式响应
    return EventSourceResponse(
        streamer.stream_to_sse(inference_req.request_id, async_queue),
        ping=15,  # 每 15 秒发送心跳 ping，防止连接超时
    )


@app.post("/v1/completions")
async def non_stream_completion(request: Request):
    """非流式推理端点——等待全部 token 生成后一次性返回"""
    body = await request.json()
    # ...（构造请求同上）
    token_deque = engine.submit(inference_req)

    # 等待所有 token 生成完毕
    result_tokens = []
    while True:
        if token_deque:
            token = token_deque.popleft()
            if token is None:
                break
            result_tokens.append(token)
        else:
            time.sleep(0.01)

    return {
        "request_id": inference_req.request_id,
        "text": detokenize(result_tokens),
        "tokens": len(result_tokens),
    }
```

#### WebSocket 流式输出完整实现（支持双向交互）

```python
from fastapi import WebSocket, WebSocketDisconnect
import uuid


class WebSocketSessionManager:
    """WebSocket 会话管理器——支持用户在生成过程中发送中止信号"""

    def __init__(self):
        self.sessions: dict[str, WebSocket] = {}

    async def connect(self, ws: WebSocket, session_id: str):
        await ws.accept()
        self.sessions[session_id] = ws

    def disconnect(self, session_id: str):
        self.sessions.pop(session_id, None)

    async def send_token(self, session_id: str, token_data: dict):
        ws = self.sessions.get(session_id)
        if ws:
            await ws.send_json(token_data)

    async def send_error(self, session_id: str, error: dict):
        ws = self.sessions.get(session_id)
        if ws:
            await ws.send_json({"type": "error", **error})


ws_manager = WebSocketSessionManager()


@app.websocket("/ws/v1/completions")
async def ws_stream_completion(ws: WebSocket):
    """WebSocket 流式推理端点"""
    session_id = str(uuid.uuid4())
    await ws_manager.connect(ws, session_id)

    try:
        # 1. 接收用户请求
        init_msg = await ws.receive_json()
        user_tier = init_msg.get("tier", "free")

        inference_req = InferenceRequest(
            request_id=f"ws-{session_id}",
            prompt_token_ids=tokenize(init_msg["prompt"]),
            sampling_params=SamplingParams(
                temperature=init_msg.get("temperature", 1.0),
                max_tokens=init_msg.get("max_tokens", 512),
            ),
            user_tier=user_tier,
            priority={"premium": 10, "standard": 5, "free": 0}[user_tier],
        )

        token_deque = engine.submit(inference_req)

        # 2. 并行：监听用户中止信号 + 推送生成 token
        async def listen_for_stop():
            """监听用户发送的中止信号"""
            while True:
                try:
                    msg = await ws.receive_json()
                    if msg.get("action") == "stop":
                        inference_req.status = RequestStatus.COMPLETED
                        await ws.send_json({
                            "type": "stopped",
                            "request_id": inference_req.request_id,
                            "tokens_so_far": len(inference_req.generated_tokens),
                        })
                        break
                except WebSocketDisconnect:
                    inference_req.status = RequestStatus.COMPLETED
                    break

        async def stream_tokens():
            """将引擎生成的 token 推送给客户端"""
            while not inference_req.is_done:
                if token_deque:
                    token = token_deque.popleft()
                    if token is None:
                        await ws.send_json({
                            "type": "done",
                            "request_id": inference_req.request_id,
                            "total_tokens": len(inference_req.generated_tokens),
                        })
                        break
                    await ws.send_json({
                        "type": "token",
                        "token": token,
                        "index": len(inference_req.generated_tokens),
                    })
                else:
                    await asyncio.sleep(0.001)

        # 同时运行两个任务
        await asyncio.gather(
            listen_for_stop(),
            stream_tokens(),
            return_exceptions=True,
        )

    except WebSocketDisconnect:
        # 客户端断线 → 清理请求
        ws_manager.disconnect(session_id)
    except Exception as e:
        await ws_manager.send_error(session_id, {
            "error": str(e),
            "request_id": inference_req.request_id,
        })
        ws_manager.disconnect(session_id)
```

**SSE 断线重连处理：**

```python
# SSE 支持 Last-Event-ID，断线重连后从断点继续
@app.get("/v1/completions/stream/{request_id}")
async def reconnect_stream(request_id: str, last_event_id: Optional[str] = None):
    """SSE 断线重连——从上次中断的位置继续"""
    if last_event_id:
        # 从 DB 或缓存中获取该请求已生成的 token
        cached_tokens = get_cached_tokens(request_id, after_index=int(last_event_id))
        # 先补发断线期间丢失的 token
        for i, token in enumerate(cached_tokens):
            yield {
                "event": "token",
                "id": str(int(last_event_id) + i + 1),
                "data": json.dumps({"token": token, "index": int(last_event_id) + i + 1}),
            }

    # 继续从引擎流式读取
    async_queue = await streamer.create_stream(request_id)
    return EventSourceResponse(
        streamer.stream_to_sse(request_id, async_queue),
        ping=15,
    )
```

### 请求排队与优先级调度

推理服务的请求调度远比普通 Web 服务复杂——不同用户等级的请求需要差异化处理，同时要防止低优先级请求饥饿。以下是完整的优先级队列实现，包含饥饿防止、抢占机制和超时处理。

#### 带饥饿防止的优先级队列

```python
import heapq
import time
from dataclasses import dataclass, field
from typing import Optional
from collections import defaultdict


@dataclass(order=False)
class PriorityTask:
    """优先级队列中的任务包装器"""
    request: InferenceRequest
    effective_priority: float = 0.0  # 实际调度优先级（含老化补偿）
    enqueue_time: float = field(default_factory=time.time)
    _heap_key: tuple = (0, 0)  # (负优先级, 入队时间)——堆按此排序

    def __lt__(self, other):
        return self._heap_key < other._heap_key


class PriorityRequestQueue:
    """带饥饿防止的优先级请求队列

    设计要点：
    1. 三级队列：premium > standard > free，付费用户绝对优先
    2. 饥饿防止：等待时间越长，有效优先级越高（aging 机制）
    3. 抢占：premium 请求可以抢占 free 请求的 GPU 执行槽
    4. 超时清理：排队超过阈值的请求自动过期并通知用户
    """

    # 各等级配置
    TIER_CONFIG = {
        "premium": {
            "base_priority": 100,       # 基硎优先级
            "max_queue_size": 500,      # 最大排队数
            "max_wait_sec": 60,         # 最大等待时间
            "aging_rate": 2.0,          # 优先级老化速率（每秒增加）
            "preempt_allowed": True,    # 允许抢占低优先级
            "timeout_action": "preempt",# 超时动作：抢占其他请求
        },
        "standard": {
            "base_priority": 50,
            "max_queue_size": 300,
            "max_wait_sec": 120,
            "aging_rate": 1.0,
            "preempt_allowed": False,
            "timeout_action": "reject",
        },
        "free": {
            "base_priority": 10,
            "max_queue_size": 200,
            "max_wait_sec": 180,
            "aging_rate": 0.5,
            "preempt_allowed": False,
            "timeout_action": "reject",
        },
    }

    def __init__(self, redis_client=None):
        self._heap = []  # 最小堆（用负优先级实现最大堆）
        self._request_map = {}  # request_id → PriorityTask
        self._tier_counts = defaultdict(int)  # 各等级当前排队数
        self._total_enqueued = 0
        self._total_expired = 0
        self.redis = redis_client  # 用于分布式限流

    def enqueue(self, request: InferenceRequest) -> dict:
        """将请求加入优先级队列"""
        tier = request.user_tier
        config = self.TIER_CONFIG[tier]

        # 1. 检查队列容量
        if self._tier_counts[tier] >= config["max_queue_size"]:
            return {
                "status": "rejected",
                "error": "QUEUE_FULL",
                "message": f"{tier} tier queue at capacity ({config['max_queue_size']})",
                "retry_after": 30,
            }

        # 2. 计算有效优先级（基础优先级 + 老化补偿）
        base_priority = config["base_priority"]
        task = PriorityTask(
            request=request,
            effective_priority=base_priority,
            enqueue_time=time.time(),
        )
        task._heap_key = (-base_priority, task.enqueue_time)

        # 3. 加入堆
        heapq.heappush(self._heap, task)
        self._request_map[request.request_id] = task
        self._tier_counts[tier] += 1
        self._total_enqueued += 1

        return {
            "status": "queued",
            "request_id": request.request_id,
            "position": self._tier_counts[tier],
            "estimated_wait_sec": self._estimate_wait(tier),
        }

    def dequeue(self, count: int = 1) -> list[InferenceRequest]:
        """从队列中取出优先级最高的 N 个请求

        每次调用时重新计算有效优先级（含老化补偿），确保等待时间长的请求
        不会被永远饿死。
        """
        # 1. 重新计算所有任务的有效优先级（老化补偿）
        self._apply_aging()

        # 2. 重新堆化（因为优先级已更新）
        heapq.heapify(self._heap)

        # 3. 取出前 N 个
        result = []
        while len(result) < count and self._heap:
            task = heapq.heappop(self._heap)
            request = task.request
            del self._request_map[request.request_id]
            self._tier_counts[request.user_tier] -= 1
            result.append(request)

        return result

    def _apply_aging(self):
        """老化补偿：等待时间越长，有效优先级越高

        防止低优先级请求永远得不到处理（饥饿问题）。
        老化速率按等级递减：free 的 aging_rate 低，但基础优先级也低，
        所以需要等待更长时间才能提升到与 standard 相当的优先级。
        """
        now = time.time()
        for task in self._heap:
            tier = task.request.user_tier
            config = self.TIER_CONFIG[tier]
            wait_time = now - task.enqueue_time

            # 有效优先级 = 基础优先级 + 等待时间 × 老化速率
            aging_bonus = wait_time * config["aging_rate"]
            task.effective_priority = config["base_priority"] + aging_bonus
            task._heap_key = (-task.effective_priority, task.enqueue_time)

    def _estimate_wait(self, tier: str) -> float:
        """估算排队等待时间"""
        # 简化模型：假设 GPU 处理速率恒定
        config = self.TIER_CONFIG[tier]
        # 排队位置 × 平均请求处理时间
        avg_processing_sec = 25.0  # 平均 25 秒/请求
        concurrent_capacity = 20   # 单 GPU 20 并发
        position = self._tier_counts[tier]
        return position * avg_processing_sec / concurrent_capacity

    def try_preempt(self, incoming: InferenceRequest,
                    running_requests: list[InferenceRequest]) -> Optional[InferenceRequest]:
        """尝试为高优先级请求抢占低优先级运行中请求

        仅 premium 用户可以抢占 free 用户的运行中请求。
        被抢占的请求状态变为 PREEMPTED，重新入队。
        """
        incoming_config = self.TIER_CONFIG.get(incoming.user_tier, {})
        if not incoming_config.get("preempt_allowed", False):
            return None

        # 找到可抢占的候选：优先级低于 incoming 的运行中请求
        candidates = [
            r for r in running_requests
            if self.TIER_CONFIG[r.user_tier]["base_priority"]
               < self.TIER_CONFIG[incoming.user_tier]["base_priority"]
        ]

        if not candidates:
            return None

        # 选择优先级最低且最早入队的请求作为牺牲者
        victim = min(
            candidates,
            key=lambda r: (
                self.TIER_CONFIG[r.user_tier]["base_priority"],
                r.enqueue_time,
            ),
        )
        victim.status = RequestStatus.PREEMPTED
        # 被抢占的请求重新入队（优先级提升，防止再次被抢占）
        victim.priority += 5
        self.enqueue(victim)

        return victim

    def cleanup_expired(self) -> int:
        """清理排队超时的请求，返回清理数量"""
        now = time.time()
        expired_ids = []

        for request_id, task in self._request_map.items():
            tier = task.request.user_tier
            config = self.TIER_CONFIG[tier]
            wait_time = now - task.enqueue_time

            if wait_time > config["max_wait_sec"]:
                expired_ids.append(request_id)
                task.request.status = RequestStatus.FAILED
                task.request.error = "QUEUE_TIMEOUT"

        # 从堆中移除过期请求
        for rid in expired_ids:
            task = self._request_map.pop(rid)
            self._heap.remove(task)
            self._tier_counts[task.request.user_tier] -= 1
            self._total_expired += 1

        if expired_ids:
            heapq.heapify(self._heap)  # 移除元素后重新堆化

        return len(expired_ids)

    def get_queue_stats(self) -> dict:
        """返回队列统计信息"""
        return {
            "total_queued": len(self._heap),
            "tier_counts": dict(self._tier_counts),
            "total_enqueued": self._total_enqueued,
            "total_expired": self._total_expired,
            "estimated_waits": {
                tier: self._estimate_wait(tier)
                for tier in self.TIER_CONFIG
            },
        }
```

**饥饿防止效果分析：**

```
场景：premium 请求持续到达，free 请求排队等待

无老化补偿：
  free 请求基础优先级 10，premium 基础优先级 100
  → free 请求永远排不到 → 饥饿

有老化补偿（aging_rate: free=0.5, premium=2.0）：
  free 请求等待 180 秒后：有效优先级 = 10 + 180×0.5 = 100
  → 与新到达的 premium 请求（优先级 100）相当
  → free 请求终于可以被调度

  但 free 请求最多等待 180 秒（max_wait_sec=180）
  → 超时后自动过期 → 不会无限等待
```

### Token 计量与限流

推理服务的成本与 token 数量直接相关——输入 1000 tokens + 输出 500 tokens 的请求成本是短请求的 10 倍。简单的请求次数限流无法准确控制成本，必须实现基于 token 的精细限流。

#### 基于 Token 的配额与滑动窗口限流

```python
import time
from collections import defaultdict


class TokenQuotaManager:
    """Token 配额管理器——按用户维度管理 token 消耗配额

    配额模型：
    - 每个用户有日/月 token 配额（不同等级配额不同）
    - 每次请求消耗 input_tokens + output_tokens
    - 配额耗尽后请求被拒绝，直到配额重置
    """

    # 各等级配额配置
    TIER_QUOTAS = {
        "premium": {
            "daily_token_limit": 10_000_000,    # 1000 万 tokens/天
            "monthly_token_limit": 200_000_000,  # 2 亿 tokens/月
            "per_request_max_tokens": 8192,      # 单次请求最大 8K tokens
        },
        "standard": {
            "daily_token_limit": 1_000_000,      # 100 万 tokens/天
            "monthly_token_limit": 20_000_000,   # 2000 万 tokens/月
            "per_request_max_tokens": 4096,      # 单次请求最大 4K tokens
        },
        "free": {
            "daily_token_limit": 100_000,        # 10 万 tokens/天
            "monthly_token_limit": 1_000_000,    # 100 万 tokens/月
            "per_request_max_tokens": 2048,      # 单次请求最大 2K tokens
        },
    }

    def __init__(self, redis_client):
        self.redis = redis_client

    def check_and_reserve(
        self, user_id: str, user_tier: str, input_tokens: int, max_output_tokens: int
    ) -> dict:
        """检查配额并预留 token 用量

        返回：
          - allowed: 是否允许
          - remaining_daily: 日配额剩余
          - remaining_monthly: 月配额剩余
        """
        config = self.TIER_QUOTAS[user_tier]
        total_tokens = input_tokens + max_output_tokens

        # 1. 单次请求 token 上限检查
        if total_tokens > config["per_request_max_tokens"]:
            return {
                "allowed": False,
                "error": "TOKEN_LIMIT_EXCEEDED",
                "message": (
                    f"Request tokens ({total_tokens}) exceed per-request limit "
                    f"({config['per_request_max_tokens']}) for {user_tier} tier"
                ),
            }

        # 2. 日配额检查（Redis 原子操作）
        daily_key = f"token_quota:daily:{user_id}:{self._today()}"
        daily_used = int(self.redis.get(daily_key) or 0)
        daily_remaining = config["daily_token_limit"] - daily_used

        if total_tokens > daily_remaining:
            return {
                "allowed": False,
                "error": "DAILY_QUOTA_EXCEEDED",
                "message": f"Daily token quota exceeded ({daily_remaining} remaining)",
                "remaining_daily": daily_remaining,
                "reset_at": "end_of_day",
            }

        # 3. 月配额检查
        monthly_key = f"token_quota:monthly:{user_id}:{self._current_month()}"
        monthly_used = int(self.redis.get(monthly_key) or 0)
        monthly_remaining = config["monthly_token_limit"] - monthly_used

        if total_tokens > monthly_remaining:
            return {
                "allowed": False,
                "error": "MONTHLY_QUOTA_EXCEEDED",
                "message": f"Monthly token quota exceeded ({monthly_remaining} remaining)",
                "remaining_monthly": monthly_remaining,
                "reset_at": "end_of_month",
            }

        # 4. 预留配额（原子递增）
        pipe = self.redis.pipeline()
        pipe.incrby(daily_key, total_tokens)
        pipe.incrby(monthly_key, total_tokens)
        # 设置过期时间（日配额 48 小时过期，月配额 32 天过期）
        pipe.expire(daily_key, 48 * 3600)
        pipe.expire(monthly_key, 32 * 24 * 3600)
        pipe.execute()

        return {
            "allowed": True,
            "reserved_tokens": total_tokens,
            "remaining_daily": daily_remaining - total_tokens,
            "remaining_monthly": monthly_remaining - total_tokens,
        }

    def record_actual_usage(self, user_id: str, input_tokens: int, actual_output_tokens: int):
        """记录实际 token 用量（输出可能少于 max_output_tokens）

        如果实际用量少于预留量，退还差额。
        """
        # 实际用量已在 check_and_reserve 时预留，此处仅记录明细
        self.redis.hset(
            f"token_usage:detail:{user_id}",
            mapping={
                "last_request_time": time.time(),
                "last_input_tokens": input_tokens,
                "last_output_tokens": actual_output_tokens,
            },
        )

    def _today(self) -> str:
        return time.strftime("%Y-%m-%d")

    def _current_month(self) -> str:
        return time.strftime("%Y-%m")


class SlidingWindowRateLimiter:
    """滑动窗口限流器——精确控制请求速率

    相比固定窗口限流，滑动窗口避免了窗口边界处的突发流量问题。

    示例：限制 20 次/分钟
    - 固定窗口：[0:00-1:00) 内 20 次，[1:00-2:00) 内 20 次
      → 边界处 0:59 和 1:01 两秒内可以 40 次请求
    - 滑动窗口：任意 60 秒内不超过 20 次 → 无突发问题
    """

    # 各等级限流配置
    TIER_RATE_LIMITS = {
        "premium": {
            "requests_per_minute": 100,
            "tokens_per_minute": 500_000,     # 50 万 tokens/分钟
            "burst_allowance": 20,             # 允许 20 次突发
        },
        "standard": {
            "requests_per_minute": 20,
            "tokens_per_minute": 50_000,
            "burst_allowance": 5,
        },
        "free": {
            "requests_per_minute": 5,
            "tokens_per_minute": 10_000,
            "burst_allowance": 2,
        },
    }

    def __init__(self, redis_client):
        self.redis = redis_client

    def check_rate_limit(
        self, user_id: str, user_tier: str, token_count: int = 0
    ) -> dict:
        """滑动窗口限流检查

        使用 Redis Sorted Set 实现滑动窗口：
        - 每个请求的时间戳作为 score 存入 sorted set
        - 检查时统计窗口内的请求数/token数
        - 允许突发流量（burst allowance）
        """
        config = self.TIER_RATE_LIMITS[user_tier]
        now = time.time()
        window_sec = 60  # 1 分钟窗口

        # --- 请求数限流 ---
        req_key = f"rate_limit:req:{user_id}"
        # 移除窗口外的旧记录
        self.redis.zremrangebyscore(req_key, 0, now - window_sec)
        # 统计当前窗口内的请求数
        current_requests = self.redis.zcard(req_key)

        if current_requests >= config["requests_per_minute"] + config["burst_allowance"]:
            # 超过限流阈值 + 突发余量 → 拒绝
            oldest = self.redis.zrange(req_key, 0, 0, withscores=True)
            retry_after = int(oldest[0][1] + window_sec - now) + 1 if oldest else 60
            return {
                "allowed": False,
                "error": "RATE_LIMITED",
                "message": f"Request rate exceeded ({current_requests}/{config['requests_per_minute']} per min)",
                "retry_after_sec": max(1, retry_after),
            }

        # --- Token 数限流 ---
        if token_count > 0:
            token_key = f"rate_limit:tokens:{user_id}"
            self.redis.zremrangebyscore(token_key, 0, now - window_sec)
            # 计算窗口内已消耗的 token 总量
            window_entries = self.redis.zrange(token_key, 0, -1, withscores=True)
            current_tokens = sum(int(e[0]) for e in window_entries)

            if current_tokens + token_count > config["tokens_per_minute"]:
                return {
                    "allowed": False,
                    "error": "TOKEN_RATE_LIMITED",
                    "message": (
                        f"Token rate exceeded "
                        f"({current_tokens + token_count}/{config['tokens_per_minute']} per min)"
                    ),
                    "retry_after_sec": 60,
                }

            # 记录 token 消耗
            self.redis.zadd(token_key, {str(token_count): now})
            self.redis.expire(token_key, window_sec + 10)

        # 记录本次请求
        self.redis.zadd(req_key, {str(now): now})
        self.redis.expire(req_key, window_sec + 10)

        # 检查是否在使用突发余量
        using_burst = current_requests >= config["requests_per_minute"]

        return {
            "allowed": True,
            "remaining_requests": config["requests_per_minute"] - current_requests,
            "burst_remaining": (
                config["burst_allowance"] - (current_requests - config["requests_per_minute"])
                if using_burst else config["burst_allowance"]
            ),
            "using_burst": using_burst,
        }
```

**滑动窗口 vs 固定窗口对比：**

```
场景：限流 20 次/分钟，用户在 0:59 发送 20 次，1:01 发送 20 次

固定窗口：
  [0:00-1:00) 20 次 ✓
  [1:00-2:00) 20 次 ✓
  → 两秒内 40 次请求通过 → 突发流量未被控制

滑动窗口：
  任意 60 秒窗口内请求数 ≤ 20 + burst_allowance
  → 0:59-1:01 这 2 秒内有 40 次请求
  → 但滑动窗口 [0:01-1:01] 内已有 20 次 → 后续 20 次被拒绝
  → 突发流量被精确控制

突发余量（Burst Allowance）：
  premium 用户：100 次/分 + 20 次突发余量
  → 正常情况 100 次/分
  → 偶尔突发可达 120 次/分（但不持续，因为突发余量不会立即恢复）
  → 突发余量恢复速率 = 窗口滑动速率（1 分钟后最早的请求滑出窗口）
```

### 模型 A/B 测试

模型上线前除了金丝雀部署的质量保障，还需要系统化的 A/B 测试来量化比较不同模型版本的效果差异。与金丝雀部署不同，A/B 测试关注的是模型输出质量的统计显著性，而不仅仅是"是否出问题"。

#### 影子模式对比与质量指标收集

```python
import hashlib
from dataclasses import dataclass
from typing import Optional
from scipy import stats as scipy_stats


@dataclass
class ABTestConfig:
    """A/B 测试配置"""
    test_id: str                          # 测试唯一标识
    model_name: str                       # 模型名称
    control_version_id: str               # 对照组版本（当前稳定版）
    treatment_version_id: str             # 实验组版本（候选版本）
    traffic_percentage: float             # 实验组流量占比，如 0.1
    min_sample_size: int = 1000           # 最小样本量（统计显著性要求）
    max_duration_hours: int = 72          # 最大测试时长
    metrics: list[str] = field(default_factory=lambda: [
        "quality_score",       # 人工/自动质量评分（1-5 分）
        "relevance_score",     # 相关性评分
        "helpfulness_score",   # 有用性评分
        "safety_score",        # 安全性评分
        "latency_ms",         # 推理延迟
        "perplexity",          # 困惑度
        "repetition_rate",     # 重复率
        "empty_output_rate",   # 空输出率
    ])


class ShadowModeEvaluator:
    """影子模式评估器——同时运行两个模型版本，对比输出质量

    影子模式（Shadow Mode）：
    - 用户请求仍然由稳定版本处理（结果返回给用户）
    - 同时将请求发送到候选版本（结果不返回用户，仅用于质量对比）
    - 这样可以在不影响用户体验的前提下收集候选版本的真实表现数据
    """

    def __init__(self, engine: ContinuousBatchEngine, metrics_store):
        self.engine = engine
        self.metrics_store = metrics_store
        self._shadow_results = {}  # request_id → ShadowResult

    async def process_with_shadow(
        self, request: InferenceRequest, config: ABTestConfig
    ) -> InferenceRequest:
        """影子模式处理：同时发送到对照组和实验组"""
        # 1. 正常请求走对照组（稳定版本）
        control_request = request
        control_request.model_version = config.control_version_id

        # 2. 复制请求到实验组（影子请求）
        shadow_request = InferenceRequest(
            request_id=f"shadow-{request.request_id}",
            prompt_token_ids=request.prompt_token_ids.copy(),
            sampling_params=request.sampling_params,
            user_tier=request.user_tier,
            priority=-1,  # 影子请求最低优先级，不影响正常请求
        )
        shadow_request.model_version = config.treatment_version_id

        # 3. 提交两个请求
        control_stream = self.engine.submit(control_request)
        shadow_stream = self.engine.submit(shadow_request)

        # 4. 异步收集影子结果（不阻塞用户响应）
        self._collect_shadow_result(shadow_request, control_request, config)

        return control_request

    def _collect_shadow_result(
        self, shadow: InferenceRequest, control: InferenceRequest, config: ABTestConfig
    ):
        """异步收集影子请求结果，与对照组对比"""
        # 等待两个请求都完成
        # ...（实际实现用 asyncio 并发等待）

        result = ShadowResult(
            request_id=control.request_id,
            test_id=config.test_id,
            control_version=config.control_version_id,
            treatment_version=config.treatment_version_id,
            control_output=control.generated_tokens,
            treatment_output=shadow.generated_tokens,
            control_latency_ms=(control.end_time - control.start_time) * 1000,
            treatment_latency_ms=(shadow.end_time - shadow.start_time) * 1000,
        )

        # 自动质量评估
        result.quality_scores = self._auto_evaluate(result)

        # 存储到指标仓库
        self.metrics_store.save_shadow_result(result)

    def _auto_evaluate(self, result: "ShadowResult") -> dict:
        """自动评估两个版本的输出质量

        评估维度：
        1. 文本相似度（与参考答案的 ROUGE/BLEU 分数）
        2. 重复率（连续重复 n-gram 的比例）
        3. 长度合理性（输出是否过长或过短）
        4. 安全性（是否包含有害内容）
        """
        scores = {}

        # 重复率检测
        scores["control_repetition_rate"] = self._calc_repetition_rate(
            result.control_output
        )
        scores["treatment_repetition_rate"] = self._calc_repetition_rate(
            result.treatment_output
        )

        # 输出长度
        scores["control_length"] = len(result.control_output)
        scores["treatment_length"] = len(result.treatment_output)

        # 延迟
        scores["control_latency_ms"] = result.control_latency_ms
        scores["treatment_latency_ms"] = result.treatment_latency_ms

        return scores

    def _calc_repetition_rate(self, tokens: list) -> float:
        """计算重复率：连续 3-gram 重复的比例"""
        if len(tokens) < 3:
            return 0.0
        trigrams = [tuple(tokens[i:i+3]) for i in range(len(tokens) - 2)]
        unique_trigrams = set(trigrams)
        return 1.0 - len(unique_trigrams) / len(trigrams)


@dataclass
class ShadowResult:
    """影子模式对比结果"""
    request_id: str
    test_id: str
    control_version: str
    treatment_version: str
    control_output: list
    treatment_output: list
    control_latency_ms: float
    treatment_latency_ms: float
    quality_scores: dict = field(default_factory=dict)


class ABTestStatisticalEvaluator:
    """A/B 测试统计评估器——判断两个版本的差异是否具有统计显著性"""

    def evaluate_test(self, test_id: str, metrics_store) -> "ABTestReport":
        """评估 A/B 测试结果，生成统计报告"""
        # 1. 获取所有影子结果
        results = metrics_store.get_shadow_results(test_id)

        control_scores = [r.quality_scores.get("quality_score", 0) for r in results
                         if r.quality_scores.get("quality_score") is not None]
        treatment_scores = [r.quality_scores.get("treatment_quality_score", 0) for r in results
                           if r.quality_scores.get("treatment_quality_score") is not None]

        # 2. 样本量检查
        min_sample = min(len(control_scores), len(treatment_scores))
        if min_sample < 100:
            return ABTestReport(
                test_id=test_id,
                conclusion="INSUFFICIENT_DATA",
                message=f"样本量不足（控制组 {len(control_scores)}，实验组 {len(treatment_scores)}），至少需要 100",
            )

        # 3. 统计显著性检验（Welch t-test，不假设等方差）
        t_stat, p_value = scipy_stats.ttest_ind(
            control_scores, treatment_scores, equal_var=False
        )

        # 4. 效应量计算（Cohen's d）
        mean_c = sum(control_scores) / len(control_scores)
        mean_t = sum(treatment_scores) / len(treatment_scores)
        var_c = sum((x - mean_c) ** 2 for x in control_scores) / len(control_scores)
        var_t = sum((x - mean_t) ** 2 for x in treatment_scores) / len(treatment_scores)
        pooled_std = ((var_c + var_t) / 2) ** 0.5
        cohens_d = (mean_t - mean_c) / pooled_std if pooled_std > 0 else 0

        # 5. 判定结论
        alpha = 0.05  # 显著性水平
        is_significant = p_value < alpha

        if not is_significant:
            conclusion = "NO_SIGNIFICANT_DIFFERENCE"
            message = "两组无统计显著差异，实验组效果与对照相当"
        elif mean_t > mean_c and cohens_d > 0.2:
            conclusion = "TREATMENT_BETTER"
            message = f"实验组显著优于对照组（p={p_value:.4f}, Cohen's d={cohens_d:.2f}）"
        elif mean_t < mean_c and cohens_d < -0.2:
            conclusion = "TREATMENT_WORSE"
            message = f"实验组显著劣于对照组（p={p_value:.4f}, Cohen's d={cohens_d:.2f}）"
        else:
            conclusion = "INSIGNIFICANT_EFFECT"
            message = f"差异显著但效应量极小（Cohen's d={cohens_d:.2f}），实际意义不大"

        return ABTestReport(
            test_id=test_id,
            conclusion=conclusion,
            message=message,
            control_mean=mean_c,
            treatment_mean=mean_t,
            p_value=p_value,
            cohens_d=cohens_d,
            is_significant=is_significant,
            sample_size=min_sample,
        )


@dataclass
class ABTestReport:
    """A/B 测试统计报告"""
    test_id: str
    conclusion: str  # TREATMENT_BETTER / TREATMENT_WORSE / NO_SIGNIFICANT_DIFFERENCE / INSIGNIFICANT_EFFECT / INSUFFICIENT_DATA
    message: str
    control_mean: float = 0.0
    treatment_mean: float = 0.0
    p_value: float = 1.0
    cohens_d: float = 0.0
    is_significant: bool = False
    sample_size: int = 0
```

**A/B 测试流程图：**

```
创建 A/B 测试配置
       ↓
影子模式运行（对照+实验同时推理）
       ↓
收集质量指标（自动评估 + 人工抽样）
       ↓
统计显著性检验（Welch t-test）
       ↓
  ┌────┴────┐
  ↓         ↓
显著差异   无显著差异
  ↓         ↓
效应量     延长测试/终止
评估       （样本不足时）
  ↓
Cohen's d > 0.2？
  ↓    ↓
 是    否
  ↓    ↓
实验组  实际意义
更优    不大
  ↓
建议金丝雀
全量部署
```

### 异常场景处理：GPU OOM、模型崩溃与队列溢出

推理服务的异常处理比普通 Web 服务复杂得多——GPU OOM 不是简单的"内存不足"，它可能导致整台机器上的所有请求全部失败。以下是关键异常场景的完整处理方案。

#### 场景 1：GPU OOM（显存溢出）

```
触发原因：
  - 并发请求过多，KV Cache 总量超出可用显存
  - 单个请求输入过长（8000+ tokens），KV Cache 占用超预期
  - 模型权重 + KV Cache + 临时缓冲区三者之和超限

影响范围：
  - CUDA OOM 会触发异常，导致当前 batch 中所有请求失败
  - 不处理 → GPU 进程崩溃 → 该节点上所有请求丢失 → 雪崩
```

```python
class OOMRecoveryHandler:
    """GPU OOM 恢复处理器"""

    def __init__(self, engine: ContinuousBatchEngine, kv_cache: PagedKVCacheManager):
        self.engine = engine
        self.kv_cache = kv_cache
        self.oom_count = 0  # 连续 OOM 计数
        self.last_oom_time = 0

    def handle_oom(self, error: RuntimeError, failed_batch: list):
        """处理 CUDA OOM 异常

        策略：
        1. 立即释放所有 KV Cache（腾出显存）
        2. 将失败请求按优先级重新入队
        3. 降低 max_batch_size（防止再次 OOM）
        4. 如果连续 OOM 3 次 → 重启 GPU 进程
        """
        if "out of memory" not in str(error).lower():
            raise error  # 非 OOM 错误，向上抛出

        self.oom_count += 1
        self.last_oom_time = time.time()

        # 1. 释放所有 KV Cache
        for request in failed_batch:
            self.kv_cache.free(request.request_id)
            request.status = RequestStatus.FAILED
            request.error = "GPU_OOM"

        # 2. 高优先级请求重新入队（低优先级直接丢弃）
        for request in sorted(failed_batch, key=lambda r: r.priority, reverse=True):
            if request.priority >= 5:  # standard 及以上
                request.status = RequestStatus.QUEUED
                self.engine.waiting_queue.appendleft(request)
            else:
                # 对免费用户返回错误
                self.engine.token_streams[request.request_id].append(None)

        # 3. 降低 batch size（防止立即再次 OOM）
        new_batch_size = max(1, self.engine.max_batch_size - 5)
        self.engine.max_batch_size = new_batch_size

        # 4. 连续 OOM 检测
        if self.oom_count >= 3 and (time.time() - self.last_oom_time) < 60:
            self._restart_gpu_process()

        # 5. 清空 CUDA 缓存
        torch.cuda.empty_cache()

        # 记录 OOM 事件到数据库
        log_oom_event(
            oom_count=self.oom_count,
            batch_size=len(failed_batch),
            vram_usage=self.kv_cache.get_memory_usage(),
            new_batch_size=new_batch_size,
        )

    def _restart_gpu_process(self):
        """重启 GPU 进程（最后手段）"""
        # 1. 将所有运行中请求标记为失败
        for request in self.engine.running_requests:
            request.status = RequestStatus.FAILED
            request.error = "GPU_PROCESS_RESTART"

        # 2. 通知调度器本节点不可用
        self.engine.node.status = "failed"

        # 3. 触发新进程启动
        subprocess.Popen(["python", "-m", "inference_server", "--recover"])

    def maybe_restore_batch_size(self):
        """OOM 后逐渐恢复 batch size"""
        if (self.oom_count > 0
                and time.time() - self.last_oom_time > 300  # 5 分钟无 OOM
                and self.engine.max_batch_size < 20):
            self.engine.max_batch_size = min(
                self.engine.max_batch_size + 2, 20
            )
            if self.engine.max_batch_size == 20:
                self.oom_count = 0  # 完全恢复
```

#### 场景 2：模型推理崩溃（NaN / 权重损坏）

```
触发原因：
  - GPU 硬件故障（ECC 错误导致权重损坏）
  - 量化精度溢出（极端输入导致 INT4 计算 NaN）
  - 驱动/内核错误

影响范围：
  - 生成乱码或 NaN → 输出无意义 → 用户体验极差
  - 权重损坏 → 后续所有请求结果都错误
```

```python
class ModelCrashDetector:
    """模型崩溃检测器——检测异常输出并触发恢复"""

    def __init__(self):
        self.consecutive_error_count = 0
        self.max_consecutive_errors = 5  # 连续 5 次异常 → 判定为模型崩溃

    def check_output(self, request: InferenceRequest, logits: torch.Tensor):
        """检测推理输出是否异常"""
        # 检查 1：logits 中是否包含 NaN 或 Inf
        if torch.isnan(logits).any() or torch.isinf(logits).any():
            self._handle_anomaly(request, "NaN_IN_LOGITS")
            return False

        # 检查 2：输出概率分布是否异常（所有概率集中在一个 token）
        probs = torch.softmax(logits, dim=-1)
        max_prob = probs.max().item()
        if max_prob > 0.99:  # 单个 token 概率 > 99%
            self.consecutive_error_count += 1
            if self.consecutive_error_count >= self.max_consecutive_errors:
                self._handle_model_crash()
            return True  # 仍输出，但记录异常

        # 检查 3：重复生成相同 token（死循环）
        if len(request.generated_tokens) >= 10:
            last_10 = request.generated_tokens[-10:]
            if len(set(last_10)) == 1:
                self._handle_anomaly(request, "REPETITION_LOOP")
                request.status = RequestStatus.COMPLETED  # 强制终止
                return False

        # 输出正常，重置计数器
        self.consecutive_error_count = 0
        return True

    def _handle_anomaly(self, request: InferenceRequest, error_type: str):
        """处理单次异常"""
        request.error = error_type
        self.consecutive_error_count += 1
        log_anomaly_event(request.request_id, error_type)

        if self.consecutive_error_count >= self.max_consecutive_errors:
            self._handle_model_crash()

    def _handle_model_crash(self):
        """模型崩溃——需要重新加载模型权重"""
        # 1. 将节点标记为不可用
        self.engine.node.status = "loading"

        # 2. 清空 CUDA 缓存
        torch.cuda.empty_cache()

        # 3. 重新加载模型权重（20 秒）
        self.model.load_weights(self.model.weights_path)

        # 4. 验证模型输出（运行一次健康检查）
        test_output = self.model.forward(self._health_check_input())
        if torch.isnan(test_output).any():
            # 重新加载仍然异常 → 硬件故障，需要更换 GPU
            self.engine.node.status = "failed"
            alert_ops_team("GPU hardware failure detected")
        else:
            self.engine.node.status = "serving"
            self.consecutive_error_count = 0
```

#### 场景 3：请求队列溢出

```
触发原因：
  - 流量突增（如热搜导致 API 调用量暴增 10 倍）
  - GPU 扩容速度跟不上请求增长（冷启动需 50 秒）
  - 部分节点故障导致可用容量下降

影响范围：
  - 队列满 → 新请求被拒绝 → 用户收到 503 错误
  - 排队时间过长 → 用户放弃 → 请求超时
```

```python
class QueueOverflowHandler:
    """请求队列溢出处理器"""

    def __init__(self, max_queue_size=1000, max_wait_time_sec=30):
        self.max_queue_size = max_queue_size
        self.max_wait_time_sec = max_wait_time_sec

    def handle_new_request(self, request: InferenceRequest):
        """处理新到达的请求"""
        queue_size = len(self.engine.waiting_queue)

        # 策略 1：免费用户——队列超过 50% 容量时直接拒绝
        if request.user_tier == "free" and queue_size > self.max_queue_size * 0.5:
            return {
                "status": "rejected",
                "error": "QUEUE_OVERFLOW",
                "message": "Service temporarily at capacity for free tier",
                "retry_after": 60,
            }

        # 策略 2：标准用户——队列超过 80% 容量时限流
        if request.user_tier == "standard" and queue_size > self.max_queue_size * 0.8:
            return {
                "status": "rejected",
                "error": "RATE_LIMITED",
                "message": "High traffic, please retry",
                "retry_after": 10,
            }

        # 策略 3：付费用户——队列满时抢占免费用户位置
        if request.user_tier == "premium" and queue_size >= self.max_queue_size:
            # 移除队列中最后一个免费用户请求
            removed = self._evict_free_user_request()
            if removed:
                self.engine.waiting_queue.append(request)
                return {"status": "queued", "request_id": request.request_id}

        # 策略 4：队列真的满了 → 返回 503
        if queue_size >= self.max_queue_size:
            return {
                "status": "rejected",
                "error": "SERVICE_OVERLOADED",
                "message": "All queues full, please retry later",
                "retry_after": 30,
            }

        # 正常入队
        self.engine.waiting_queue.append(request)
        return {"status": "queued", "request_id": request.request_id}

    def cleanup_stale_requests(self):
        """清理排队超时的请求（定期执行）"""
        now = time.time()
        stale = [
            r for r in self.engine.waiting_queue
            if now - r.enqueue_time > self.max_wait_time_sec
        ]
        for r in stale:
            self.engine.waiting_queue.remove(r)
            r.status = RequestStatus.FAILED
            r.error = "QUEUE_TIMEOUT"
            self.engine.token_streams[r.request_id].append(None)

        if stale:
            log_queue_timeout(len(stale), self.max_wait_time_sec)
```

#### 场景 4：GPU 节点心跳丢失

```python
class NodeHealthMonitor:
    """GPU 节点健康监控——检测心跳丢失并自动故障转移"""

    def __init__(self, heartbeat_timeout_sec=30, check_interval_sec=10):
        self.heartbeat_timeout = heartbeat_timeout_sec
        self.check_interval = check_interval_sec

    def check_heartbeats(self):
        """定期检查所有节点的心跳"""
        now = datetime.utcnow()
        nodes = db.query(GPUNode).filter(GPUNode.status.in_(["serving", "idle"]))

        for node in nodes:
            elapsed = (now - node.last_heartbeat).total_seconds()
            if elapsed > self.heartbeat_timeout:
                self._handle_node_failure(node)

    def _handle_node_failure(self, node: GPUNode):
        """处理节点故障"""
        # 1. 标记节点为故障
        node.status = "failed"

        # 2. 将该节点上的所有运行中请求重新入队
        failed_jobs = db.query(InferenceJob).filter(
            InferenceJob.node_id == node.node_id,
            InferenceJob.status.in_(["prefilling", "decoding"]),
        )
        requeued = 0
        for job in failed_jobs:
            job.status = "queued"
            job.error_code = "NODE_FAILURE"
            job.node_id = None
            requeued += 1

        # 3. 通知调度器排除故障节点
        scheduler.remove_node(node.node_id)

        # 4. 触发自动扩容补偿
        autoscaler.emergency_scale_up(count=1)

        # 5. 告警
        alert_ops_team(
            f"GPU node {node.instance_id} heartbeat lost. "
            f"Requeued {requeued} requests."
        )

    def start_monitoring(self):
        """启动监控线程"""
        def monitor_loop():
            while True:
                self.check_heartbeats()
                time.sleep(self.check_interval)
        threading.Thread(target=monitor_loop, daemon=True).start()
```

**异常处理全景图：**

```
异常类型          检测方式               恢复策略                      恢复时间
─────────────────────────────────────────────────────────────────────────────────
GPU OOM          CUDA RuntimeError      释放KV Cache + 降batch size   < 1秒
模型权重损坏      NaN/Inf检测           重新加载权重                   ~20秒
模型输出异常      重复token检测          强制终止请求                   < 0.1秒
队列溢出          队列长度检查           分级拒绝/抢占                  < 0.01秒
节点心跳丢失      心跳超时检测           故障转移+扩容                  ~60秒
GPU硬件故障       重载后仍NaN            标记故障+人工介入               ~30分钟
请求超时          排队时间检查           清理超时请求                   < 0.01秒
网络断连          SSE/WebSocket断开      自动重连+续传                  < 5秒
权重文件损坏      SHA256校验和不匹配     自动重新下载+重载              ~120秒
KV Cache OOM     显存使用率>95%         降batch+抢占低优先级请求        < 1秒
推理超时          生成时间超限           返回部分结果+标记不完整         < 0.1秒
```

#### 场景 5：模型权重文件损坏检测与自动重载

```
触发原因：
  - GPU 显存 ECC 错误导致加载到显存中的权重位翻转
  - 磁盘 I/O 错误导致权重文件部分损坏
  - 网络传输中断导致下载的权重文件不完整
  - 多进程并发写权重文件导致文件内容不一致

影响范围：
  - 权重损坏 → 模型输出完全不可靠 → 所有后续请求结果错误
  - 与"模型输出异常"不同，权重损坏是持久性的，不会自行恢复
  - 不检测 → 持续产生错误结果 → 严重损害产品信誉
```

```python
class ModelWeightIntegrityChecker:
    """模型权重完整性检测器——检测权重损坏并触发自动重载"""

    def __init__(self, model, registry: ModelVersionRegistry, check_interval_sec=300):
        self.model = model
        self.registry = registry
        self.check_interval = check_interval_sec
        self._known_checksums = {}   # version_id → SHA256 checksum
        self._last_check_time = 0
        self._corruption_count = 0   # 连续损坏检测次数

        # 初始化：记录所有已加载模型的权重校验和
        for version in registry.list_versions(model.model_name):
            if version.status in (ModelVersionStatus.STABLE, ModelVersionStatus.CANARY):
                self._known_checksums[version.version_id] = version.weights_checksum

    def check_integrity(self) -> dict:
        """执行完整性检查

        检查三个层级：
        1. 磁盘权重文件校验（SHA256）——检测磁盘/传输损坏
        2. 显存权重抽样校验（比对关键层）——检测 ECC 位翻转
        3. 推理输出一致性校验（固定输入的输出是否与基线一致）——检测隐性损坏
        """
        results = {
            "disk_check": "unknown",
            "vram_check": "unknown",
            "inference_check": "unknown",
            "action_taken": "none",
        }

        # --- 第 1 层：磁盘文件完整性 ---
        disk_result = self._check_disk_integrity()
        results["disk_check"] = disk_result["status"]

        if disk_result["status"] == "CORRUPTED":
            # 磁盘文件已损坏 → 重新下载
            self._handle_disk_corruption(disk_result)
            results["action_taken"] = "redownloading_weights"
            return results

        # --- 第 2 层：显存权重抽样校验 ---
        vram_result = self._check_vram_integrity()
        results["vram_check"] = vram_result["status"]

        if vram_result["status"] == "CORRUPTED":
            # 显存权重损坏（ECC 位翻转）→ 重新加载权重到显存
            self._handle_vram_corruption(vram_result)
            results["action_taken"] = "reloading_weights_to_vram"
            return results

        # --- 第 3 层：推理输出一致性 ---
        inference_result = self._check_inference_consistency()
        results["inference_check"] = inference_result["status"]

        if inference_result["status"] == "INCONSISTENT":
            # 输出与基线不一致（可能是量化导致的隐性损坏）
            self._corruption_count += 1
            if self._corruption_count >= 3:
                # 连续 3 次不一致 → 判定为权重问题，重新加载
                self._handle_inference_corruption(inference_result)
                results["action_taken"] = "force_reload_after_inconsistency"
            else:
                results["action_taken"] = "monitoring_trend"
        else:
            self._corruption_count = 0  # 重置连续计数

        return results

    def _check_disk_integrity(self) -> dict:
        """检查磁盘上的权重文件是否完整"""
        current_version = self.model.current_version_id
        expected_checksum = self._known_checksums.get(current_version)

        if not expected_checksum:
            return {"status": "UNKNOWN", "message": "No checksum on record"}

        # 计算当前文件的 SHA256
        version_info = self.registry._find_version(current_version)
        actual_checksum = version_info.compute_checksum(version_info.weights_path)

        if actual_checksum != expected_checksum:
            return {
                "status": "CORRUPTED",
                "expected": expected_checksum,
                "actual": actual_checksum,
                "message": "Weight file checksum mismatch — file corrupted on disk",
            }

        return {"status": "OK", "checksum": actual_checksum}

    def _check_vram_integrity(self) -> dict:
        """检查显存中的权重是否与磁盘一致（抽样关键层）

        抽样策略：检查 Transformer 的第 0 层、中间层、最后一层的权重，
        因为 ECC 错误通常只影响部分显存区域。
        """
        # 抽样检查关键层
        num_layers = self.model.config.num_hidden_layers
        sample_layers = [0, num_layers // 2, num_layers - 1]

        for layer_idx in sample_layers:
            # 从磁盘读取该层权重的哈希
            disk_hash = self._get_disk_layer_hash(layer_idx)
            # 从显存读取该层权重并计算哈希
            vram_hash = self._get_vram_layer_hash(layer_idx)

            if disk_hash != vram_hash:
                return {
                    "status": "CORRUPTED",
                    "corrupted_layer": layer_idx,
                    "expected_hash": disk_hash,
                    "actual_hash": vram_hash,
                    "message": f"Layer {layer_idx} weights in VRAM don't match disk",
                }

        return {"status": "OK", "checked_layers": sample_layers}

    def _check_inference_consistency(self) -> dict:
        """推理输出一致性检查——使用固定输入测试

        原理：用一组预定义的测试输入，对比模型输出与历史基线。
        如果输出差异超出阈值，说明模型可能存在隐性权重损坏
        （磁盘和显存哈希都正确，但量化后行为异常）。
        """
        test_inputs = self._get_health_check_inputs()  # 预定义测试用例
        baseline_outputs = self._get_baseline_outputs()  # 历史基线输出

        for test_input, baseline in zip(test_inputs, baseline_outputs):
            current_output = self.model.forward(test_input)

            # 计算与基线的余弦相似度
            similarity = self._cosine_similarity(current_output, baseline)
            if similarity < 0.95:  # 相似度低于 95% → 异常
                return {
                    "status": "INCONSISTENT",
                    "similarity": similarity,
                    "threshold": 0.95,
                    "message": f"Output similarity {similarity:.3f} below threshold 0.95",
                }

        return {"status": "OK", "similarity": "all_above_threshold"}

    def _handle_disk_corruption(self, result: dict):
        """处理磁盘文件损坏——重新下载权重文件"""
        # 1. 标记节点为不可用
        self.model.node.status = "loading"

        # 2. 从 S3/OSS 重新下载权重文件
        version_info = self.registry._find_version(self.model.current_version_id)
        self._download_weights(version_info.weights_path, force=True)

        # 3. 重新校验
        new_checksum = version_info.compute_checksum(version_info.weights_path)
        if new_checksum != version_info.weights_checksum:
            # 重新下载后仍然校验失败 → 存储系统可能有问题
            alert_ops_team(
                f"权重文件重新下载后校验仍失败: {version_info.version_id}，"
                f"请检查存储系统"
            )
            self.model.node.status = "failed"
            return

        # 4. 重新加载到显存
        self.model.load_weights(version_info.weights_path)
        self.model.node.status = "serving"

        log_weight_corruption(
            version_id=self.model.current_version_id,
            corruption_type="disk",
            action="redownloaded_and_reloaded",
            details=result,
        )

    def _handle_vram_corruption(self, result: dict):
        """处理显存权重损坏（ECC 位翻转）——重新加载权重到显存"""
        # 不需要重新下载，只需要从磁盘重新加载到显存
        self.model.node.status = "loading"

        # 清空当前显存
        torch.cuda.empty_cache()

        # 从磁盘重新加载
        version_info = self.registry._find_version(self.model.current_version_id)
        self.model.load_weights(version_info.weights_path)

        # 验证
        recheck = self._check_vram_integrity()
        if recheck["status"] == "CORRUPTED":
            # 重新加载后仍然损坏 → 可能是 GPU 硬件故障
            alert_ops_team(
                f"GPU {self.model.node.instance_id}: 显存权重重载后仍损坏，"
                f"疑似硬件故障（ECC 错误累积），建议更换 GPU"
            )
            self.model.node.status = "failed"
        else:
            self.model.node.status = "serving"
            log_weight_corruption(
                version_id=self.model.current_version_id,
                corruption_type="vram_ecc",
                action="reloaded_to_vram",
                details=result,
            )

    def _handle_inference_corruption(self, result: dict):
        """处理推理输出不一致——强制重新加载"""
        self.model.node.status = "loading"
        torch.cuda.empty_cache()

        version_info = self.registry._find_version(self.model.current_version_id)
        self.model.load_weights(version_info.weights_path)

        # 重置计数器
        self._corruption_count = 0
        self.model.node.status = "serving"

        log_weight_corruption(
            version_id=self.model.current_version_id,
            corruption_type="inference_inconsistency",
            action="force_reload",
            details=result,
        )

    def start_periodic_check(self):
        """启动定期完整性检查（后台线程）"""
        def check_loop():
            while True:
                self.check_integrity()
                time.sleep(self.check_interval)
        threading.Thread(target=check_loop, daemon=True).start()

    # 辅助方法（省略实现细节）
    def _get_disk_layer_hash(self, layer_idx: int) -> str:
        """从磁盘读取指定层的权重并计算哈希"""
        pass

    def _get_vram_layer_hash(self, layer_idx: int) -> str:
        """从显存读取指定层的权重并计算哈希"""
        pass

    def _get_health_check_inputs(self) -> list:
        """获取预定义的健康检查输入"""
        return self._health_check_inputs

    def _get_baseline_outputs(self) -> list:
        """获取历史基线输出（模型首次加载时计算并缓存）"""
        return self._baseline_outputs

    def _cosine_similarity(self, a, b) -> float:
        """计算两个向量的余弦相似度"""
        pass

    def _download_weights(self, path: str, force: bool = False):
        """从远程存储下载权重文件"""
        pass
```

#### 场景 6：KV Cache OOM 与优雅降级

```
触发原因：
  - 多个长上下文请求同时运行，KV Cache 总量超出可用显存
  - 与 GPU OOM 不同，KV Cache OOM 是显存不足但尚未触发 CUDA 崩溃的"临界状态"
  - 如果不及时处理 → 进化为 GPU OOM → 所有请求失败

关键区别：
  - GPU OOM：已经崩溃，需要紧急恢复
  - KV Cache OOM：尚未崩溃，可以通过降级策略优雅处理
```

```python
class KVCacheOOMHandler:
    """KV Cache OOM 优雅降级处理器

    策略优先级（从影响最小到影响最大）：
    1. 降低 batch size → 减少并发请求数 → 降低 KV Cache 总量
    2. 抢占低优先级请求 → 释放其 KV Cache → 为高优先级请求腾出空间
    3. 压缩已有请求的 KV Cache（量化 KV Cache：FP16→INT8）
    4. 拒绝新的长上下文请求（限制 max_seq_len）
    """

    def __init__(
        self,
        engine: ContinuousBatchEngine,
        kv_cache: PagedKVCacheManager,
        node_repo,
    ):
        self.engine = engine
        self.kv_cache = kv_cache
        self.node_repo = node_repo
        self._degraded = False  # 是否处于降级模式
        self._original_batch_size = engine.max_batch_size
        self._original_max_seq_len = 8192  # 原始最大序列长度

    def check_and_handle(self):
        """定期检查 KV Cache 使用率并执行降级策略"""
        usage = self.kv_cache.get_memory_usage()
        utilization_pct = usage["utilization_pct"]

        if utilization_pct > 95:
            # 临界状态 → 紧急降级
            self._emergency_degradation()
        elif utilization_pct > 85:
            # 高负载 → 温和降级
            self._gentle_degradation()
        elif utilization_pct < 60 and self._degraded:
            # 负载下降 → 逐步恢复
            self._gradual_recovery()

    def _emergency_degradation(self):
        """紧急降级：KV Cache 使用率 > 95%"""
        self._degraded = True

        # 策略 1：大幅降低 batch size（立即生效）
        new_batch_size = max(5, self.engine.max_batch_size - 10)
        self.engine.max_batch_size = new_batch_size

        # 策略 2：抢占最低优先级的请求（释放 KV Cache）
        freed_count = self._preempt_low_priority_requests(target_freed_pages=500)

        # 策略 3：限制新请求的最大序列长度
        self._original_max_seq_len = getattr(self.engine, 'max_seq_len', 8192)
        self.engine.max_seq_len = 2048  # 从 8K 降到 2K

        # 策略 4：对现有 KV Cache 做 INT8 量化（如果支持）
        if hasattr(self.kv_cache, 'quantize_active_pages'):
            self.kv_cache.quantize_active_pages()  # FP16 KV Cache → INT8，节省 50% 空间

        log_kv_cache_oom(
            action="emergency_degradation",
            batch_size=new_batch_size,
            freed_requests=freed_count,
            max_seq_len=self.engine.max_seq_len,
            vram_usage=self.kv_cache.get_memory_usage(),
        )

    def _gentle_degradation(self):
        """温和降级：KV Cache 使用率 > 85%"""
        self._degraded = True

        # 策略 1：小幅降低 batch size
        if self.engine.max_batch_size > 10:
            self.engine.max_batch_size -= 2

        # 策略 2：限制新请求的最大序列长度（从 8K 降到 4K）
        self.engine.max_seq_len = 4096

        log_kv_cache_oom(
            action="gentle_degradation",
            batch_size=self.engine.max_batch_size,
            max_seq_len=self.engine.max_seq_len,
        )

    def _preempt_low_priority_requests(self, target_freed_pages: int) -> int:
        """抢占低优先级请求以释放 KV Cache 页

        按优先级从低到高逐个抢占，直到释放足够的页数。
        被抢占的请求标记为 PREEMPTED 并重新入队。
        """
        running = sorted(
            self.engine.running_requests,
            key=lambda r: (r.priority, r.enqueue_time)
        )

        freed_pages = 0
        freed_count = 0

        for victim in running:
            if freed_pages >= target_freed_pages:
                break
            # 不抢占 premium 用户的请求
            if victim.user_tier == "premium":
                continue

            # 释放该请求的 KV Cache
            if victim.request_id in self.kv_cache.request_pages:
                pages_held = len(self.kv_cache.request_pages[victim.request_id])
                self.kv_cache.free(victim.request_id)
                freed_pages += pages_held
                freed_count += 1

                # 标记为被抢占，重新入队
                victim.status = RequestStatus.PREEMPTED
                self.engine.waiting_queue.appendleft(victim)

        return freed_count

    def _gradual_recovery(self):
        """逐步恢复：KV Cache 使用率下降到安全水平后"""
        # 恢复 batch size（每次 +2，避免过快导致再次 OOM）
        if self.engine.max_batch_size < self._original_batch_size:
            self.engine.max_batch_size = min(
                self.engine.max_batch_size + 2,
                self._original_batch_size,
            )

        # 恢复最大序列长度（每次 +1024）
        if self.engine.max_seq_len < self._original_max_seq_len:
            self.engine.max_seq_len = min(
                self.engine.max_seq_len + 1024,
                self._original_max_seq_len,
            )

        # 恢复 KV Cache 量化（INT8 → FP16，提升精度）
        if hasattr(self.kv_cache, 'dequantize_active_pages'):
            self.kv_cache.dequantize_active_pages()

        # 检查是否完全恢复
        if (self.engine.max_batch_size == self._original_batch_size
                and self.engine.max_seq_len == self._original_max_seq_len):
            self._degraded = False

        log_kv_cache_oom(
            action="gradual_recovery",
            batch_size=self.engine.max_batch_size,
            max_seq_len=self.engine.max_seq_len,
            fully_recovered=not self._degraded,
        )

    def should_accept_request(self, request: InferenceRequest) -> dict:
        """降级模式下对新请求的准入检查"""
        if not self._degraded:
            return {"allowed": True}

        # 长上下文请求：检查是否超出降级后的 max_seq_len
        total_tokens = len(request.prompt_token_ids) + request.sampling_params.max_tokens
        if total_tokens > self.engine.max_seq_len:
            return {
                "allowed": False,
                "error": "CONTEXT_LENGTH_EXCEEDED",
                "message": (
                    f"Service under load, max context length reduced to "
                    f"{self.engine.max_seq_len} tokens (your request: {total_tokens})"
                ),
                "current_max_seq_len": self.engine.max_seq_len,
                "retry_after_sec": 60,
            }

        # free 用户在降级模式下直接拒绝
        if request.user_tier == "free":
            return {
                "allowed": False,
                "error": "SERVICE_DEGRADED",
                "message": "Service temporarily under high load, free tier unavailable",
                "retry_after_sec": 30,
            }

        return {"allowed": True}

    @property
    def is_degraded(self) -> bool:
        return self._degraded

    def get_status(self) -> dict:
        """返回当前降级状态"""
        return {
            "is_degraded": self._degraded,
            "current_batch_size": self.engine.max_batch_size,
            "original_batch_size": self._original_batch_size,
            "current_max_seq_len": self.engine.max_seq_len,
            "original_max_seq_len": self._original_max_seq_len,
            "kv_cache_usage": self.kv_cache.get_memory_usage(),
        }
```

**KV Cache OOM 降级过程可视化：**

```
KV Cache 使用率变化：
100% ├────────────────────╲  紧急降级              ╱─── 恢复
 95% │                     ╲ batch-10, seq 2K     ╱
 85% │                      ╲───────────────╲──╱
 60% │                                      ╲╱
     └─────────────────────────────────────────── 时间
        ↑                    ↑              ↑
    正常运行             温和降级         逐步恢复

各阶段参数：
  正常：    batch=20, max_seq=8192,  接受所有等级请求
  温和降级： batch=18, max_seq=4096,  接受所有等级请求
  紧急降级： batch=10, max_seq=2048,  拒绝 free 用户 + 抢占低优先级
  恢复：    batch +2/次, max_seq +1024/次, 逐步回到正常
```

#### 场景 7：推理超时与部分结果返回

```
触发原因：
  - 模型生成死循环（重复 token 未被检测到）→ 永远不会结束
  - 请求 max_tokens 设置过大（如 32000）→ 生成时间超长
  - GPU 节点负载过高 → 每 token 生成延迟剧增 → 总时间超预期
  - 解码过程中遇到罕见 token → 采样效率下降

影响范围：
  - 用户无限等待 → 体验极差
  - GPU 资源被长时间占用 → 其他请求排队
  - 流式输出中断在某个 token → 用户看到不完整内容
```

```python
class InferenceTimeoutHandler:
    """推理超时处理器——超时后返回已生成的部分结果

    设计原则：
    1. 优先返回部分结果而非空结果——用户宁愿看到半截回答也不要白等
    2. 部分结果必须标记为"不完整"——避免用户误以为是完整答案
    3. 超时请求必须释放 GPU 资源——防止死循环请求永远占用
    """

    # 各等级超时配置
    TIER_TIMEOUT = {
        "premium": {
            "max_total_sec": 120,       # 最大总时间 120 秒
            "max_first_token_sec": 5,   # 首 token 最大等待 5 秒
            "max_decode_per_token_sec": 0.5,  # 每 token 最大解码时间 0.5 秒
        },
        "standard": {
            "max_total_sec": 90,
            "max_first_token_sec": 8,
            "max_decode_per_token_sec": 0.8,
        },
        "free": {
            "max_total_sec": 60,
            "max_first_token_sec": 15,
            "max_decode_per_token_sec": 1.0,
        },
    }

    def __init__(self, engine: ContinuousBatchEngine):
        self.engine = engine
        self._timeout_checks_per_sec = 2  # 每秒检查 2 次

    def check_timeouts(self):
        """检查所有运行中请求的超时状态（定期调用）"""
        now = time.time()
        timed_out = []

        for request in self.engine.running_requests:
            if request.start_time is None:
                continue

            tier = request.user_tier
            config = self.TIER_TIMEOUT[tier]
            elapsed = now - request.start_time

            # 检查 1：首 token 超时
            if request.first_token_time is None:
                if elapsed > config["max_first_token_sec"]:
                    timed_out.append((request, "FIRST_TOKEN_TIMEOUT"))
                    continue

            # 检查 2：总时间超时
            if elapsed > config["max_total_sec"]:
                timed_out.append((request, "TOTAL_TIMEOUT"))
                continue

            # 检查 3：单 token 解码超时
            if request.first_token_time is not None and len(request.generated_tokens) > 0:
                # 计算最近 5 个 token 的平均解码时间
                if len(request.generated_tokens) >= 5:
                    recent_start = now - 5 * config["max_decode_per_token_sec"]
                    tokens_in_window = sum(
                        1 for t in request.generated_tokens[-5:]
                        if True  # 简化：实际需记录每个 token 的时间戳
                    )
                    # 如果最近 5 个 token 的解码时间超过预期 2 倍
                    avg_decode_time = elapsed / max(len(request.generated_tokens), 1)
                    if avg_decode_time > config["max_decode_per_token_sec"] * 2:
                        timed_out.append((request, "SLOW_DECODE_TIMEOUT"))

        # 处理超时请求
        for request, reason in timed_out:
            self._handle_timeout(request, reason)

    def _handle_timeout(self, request: InferenceRequest, reason: str):
        """处理超时请求——返回部分结果并释放资源"""
        has_partial_result = len(request.generated_tokens) > 0

        if has_partial_result:
            # 有部分结果 → 返回部分结果 + 不完整标记
            request.status = RequestStatus.COMPLETED  # 标记为完成（虽然是部分完成）
            request.error = f"PARTIAL_RESULT:{reason}"

            # 在 token 流中附加部分结果元信息
            partial_metadata = {
                "is_partial": True,
                "reason": reason,
                "tokens_generated": len(request.generated_tokens),
                "tokens_requested": request.sampling_params.max_tokens,
                "completion_pct": round(
                    len(request.generated_tokens) / request.sampling_params.max_tokens * 100, 1
                ),
            }
            # 将元信息推送到 token 流（在 None 终止标记之前）
            self.engine.token_streams[request.request_id].append(
                ("partial_metadata", partial_metadata)
            )
            # 终止标记
            self.engine.token_streams[request.request_id].append(None)

        else:
            # 无部分结果 → 返回超时错误
            request.status = RequestStatus.FAILED
            request.error = reason

            # 推送超时错误到 token 流
            self.engine.token_streams[request.request_id].append(
                ("error", {
                    "error": "INFERENCE_TIMEOUT",
                    "reason": reason,
                    "message": self._get_timeout_message(reason),
                })
            )
            self.engine.token_streams[request.request_id].append(None)

        # 释放 GPU 资源
        self.engine.kv_cache_manager.free(request.request_id)
        if request in self.engine.running_requests:
            self.engine.running_requests.remove(request)

        # 记录超时事件
        log_inference_timeout(
            request_id=request.request_id,
            user_tier=request.user_tier,
            reason=reason,
            tokens_generated=len(request.generated_tokens),
            elapsed_sec=time.time() - request.start_time if request.start_time else 0,
        )

    def _get_timeout_message(self, reason: str) -> str:
        """返回用户友好的超时消息"""
        messages = {
            "FIRST_TOKEN_TIMEOUT": "Model is taking too long to start generating. Please try again.",
            "TOTAL_TIMEOUT": "Generation exceeded maximum allowed time. A partial result may be available.",
            "SLOW_DECODE_TIMEOUT": "Generation speed is unusually slow. This may be due to high server load.",
        }
        return messages.get(reason, "Inference timed out.")

    def start_monitoring(self):
        """启动超时监控线程"""
        def monitor_loop():
            while True:
                self.check_timeouts()
                time.sleep(1.0 / self._timeout_checks_per_sec)
        threading.Thread(target=monitor_loop, daemon=True).start()
```

**SSE 端点部分结果处理：**

```python
# 在 SSE streamer 中处理部分结果
async def stream_to_sse_with_partial(self, request_id: str, queue: asyncio.Queue):
    """SSE 流式输出——支持部分结果标记"""
    token_count = 0
    start_time = time.time()

    try:
        while True:
            item = await asyncio.wait_for(queue.get(), timeout=60.0)

            if item is None:
                # 正常完成
                yield {"event": "done", "data": json.dumps({
                    "request_id": request_id,
                    "total_tokens": token_count,
                    "is_partial": False,
                })}
                break

            if isinstance(item, tuple) and item[0] == "partial_metadata":
                # 部分结果元信息——标记输出不完整
                metadata = item[1]
                yield {"event": "partial", "data": json.dumps({
                    "request_id": request_id,
                    "is_partial": True,
                    "reason": metadata["reason"],
                    "tokens_generated": metadata["tokens_generated"],
                    "tokens_requested": metadata["tokens_requested"],
                    "completion_pct": metadata["completion_pct"],
                    "message": "Generation was interrupted. The output above is partial.",
                })}
                break

            if isinstance(item, tuple) and item[0] == "error":
                # 超时错误——无部分结果
                error_info = item[1]
                yield {"event": "error", "data": json.dumps({
                    "request_id": request_id,
                    **error_info,
                })}
                break

            # 正常 token
            token_count += 1
            yield {"event": "token", "data": json.dumps({
                "request_id": request_id,
                "token": item,
                "index": token_count,
            })}

    except asyncio.TimeoutError:
        yield {"event": "error", "data": json.dumps({
            "request_id": request_id,
            "error": "stream_timeout",
            "message": "Token stream timeout",
        })}
    finally:
        self.active_streams.pop(request_id, None)
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

**Chunked Prefill 原理详解：**

```
传统 Prefill：8000 tokens 一次性计算 → 占用 GPU 5 秒 → 其他请求等待

Chunked Prefill：将 8000 tokens 切分为 16 个 512-token 的 chunk
  每个 chunk 计算约 300ms
  chunk 之间可以插入 decode 步骤（为其他请求生成 1 个 token）
  → 长请求的 Prefill 延迟分摊到 16 次迭代中
  → 短请求的首 token 延迟从 5 秒降到 500ms
```

```python
class ChunkedPrefillScheduler:
    """Chunked Prefill 调度器——长输入分块处理，避免阻塞解码"""

    def __init__(self, chunk_size=512):
        self.chunk_size = chunk_size

    def schedule_iteration(self, running_requests, waiting_queue):
        """每次迭代：优先处理 decode，空闲时间处理 prefill chunk"""
        decode_requests = [r for r in running_requests if r.status == "decoding"]
        prefill_requests = [r for r in running_requests if r.status == "prefilling"]

        # 1. 所有 decode 请求并行前向传播（生成 1 token）
        if decode_requests:
            self.forward_decode(decode_requests)

        # 2. 利用 decode 之间的间隙处理 prefill chunk
        for r in prefill_requests:
            chunk = self.get_next_prefill_chunk(r, self.chunk_size)
            self.forward_prefill_chunk(r, chunk)
            if self.is_prefill_complete(r):
                r.status = "decoding"  # Prefill 完成，转入解码阶段

    def get_next_prefill_chunk(self, request, chunk_size):
        """获取下一个 prefill chunk"""
        start = request.prefill_progress  # 已处理的 token 数
        end = min(start + chunk_size, len(request.prompt_token_ids))
        chunk = request.prompt_token_ids[start:end]
        request.prefill_progress = end
        return chunk
```

### 陷阱 6：不做限流

**具体数据：** 免费用户突发 1000 并发 → 占满所有 GPU → 付费用户请求排队 30 秒 → 付费用户流失。

**解决方案：** 按用户等级限流 + 优先级队列。付费用户优先处理，免费用户限流。

### 陷阱 7：模型切换导致请求全部中断

**具体数据：** 从 FP16 模型切换到 INT8 版本 → 需要卸载旧模型（5秒）+ 加载新模型（20秒） → 25 秒内该节点无法服务 → 排队请求暴增。

**解决方案：** 滚动升级——新节点加载新模型，旧节点 drain 后下线，始终保持可用容量。或双模型共存（大显存节点同时加载两个版本）。

### 陷阱 8：流式输出中途断连丢失 token

**具体数据：** 用户网络抖动导致 SSE 连接断开 → 已生成 300 个 token 丢失 → 用户重连后从头开始生成 → 浪费 GPU 算力 + 用户体验差。

**解决方案：** SSE 的 `Last-Event-ID` 机制 + 服务端缓存已生成 token。断线重连后从断点续传，无需重新推理。

## 性能与成本分析

### GPU 利用率优化路径

```
优化阶段          GPU利用率    吞吐量(tokens/s)    月成本(8×A100)   优化手段
────────────────────────────────────────────────────────────────────────────
基线(FP16+静态)    40%         250                $16,000         无
+连续Batch         65%         400                $9,800          Continuous Batching
+PagedAttention    75%         450                $8,500          KV Cache按需分配
+INT8量化          80%         550                $4,800          4卡即可运行
+弹性扩缩容        78%         520                $3,200          峰值30台/均值6台
+Speculative Dec.  88%         850                $2,100          小模型草稿+大模型验证
────────────────────────────────────────────────────────────────────────────
总计：利用率从40%→88%，成本从$16K→$2.1K，降低87%
```

### 推理延迟分解

```
单次请求延迟组成（INT8 量化，500 tokens 输出）：

  Tokenize（CPU）      20ms    ┐
  排队等待            0-3000ms  │ 受 QPS 和 batch 调度影响
  Prefill（GPU）      200ms   ┘── 首 Token 延迟 = 220ms~3220ms
  Decode×500（GPU）   25ms/token × 500 = 12,500ms
  Detokenize（CPU）    15ms
  网络传输             10ms/token × 500 = 5,000ms（SSE逐token推送）

  总延迟 ≈ 首 Token 延迟 + 12,500ms + 15ms
         = 12.7s ~ 15.7s（取决于排队时间）

  P99 首 Token 延迟优化：
    - 无优化：3000ms（高峰排队）
    - +优先级队列：800ms（付费用户优先调度）
    - +Chunked Prefill：400ms（长请求不阻塞短请求）
    - +Speculative Decoding：250ms（小模型预生成首 token）
```

### 成本模型详解

```
月成本 = GPU 租金 + 网络费用 + 存储费用 + 运维人力

方案 A：固定集群（8×A100 FP16）
  GPU 租金：8 × $2,000/月 = $16,000
  网络费用：100万请求 × 500 tokens × 4 bytes/token × $0.09/GB ≈ $180
  存储：500GB 模型权重 × $0.023/GB ≈ $12
  利用率：均值 QPS 200 → 需要 8 台 → 利用率 40%
  有效成本/请求 = $16,192 / 3,000,000 ≈ $0.0054

方案 B：弹性集群（INT8 量化 + 连续 Batch）
  GPU 租金：均值 4 台 × $2,000 + 峰值 30 台 × $2,000 × 20% 时间
          = $8,000 + $12,000 × 0.2 = $10,400
  量化：INT8 → 4 卡可运行，单台处理能力翻倍
  利用率：均值 78%
  有效成本/请求 = $10,592 / 3,000,000 ≈ $0.0035

方案 C：极致优化（INT4 AWQ + Speculative Decoding + 弹性）
  GPU 租金：均值 3 台 × $2,000 + 峰值 15 台 × $2,000 × 15%
          = $6,000 + $4,500 = $10,500 → 弹性调度后 $2,800
  INT4 AWQ：2 卡可运行 70B → 成本再降 50%
  Speculative Decoding：吞吐量提升 2.5 倍 → 需要 GPU 数量减半
  利用率：88%
  有效成本/请求 ≈ $0.0010

对比：
  方案 A → 方案 C：每请求成本从 $0.0054 降到 $0.0010，降低 81%
```

### 每 1K Tokens 成本分析

精确到每 1K tokens 的成本分析是定价和利润计算的基础。不同量化方案、GPU 型号和负载水平下的成本差异显著。

```
每 1K tokens 成本计算公式：
  cost_per_1k_tokens = (GPU 小时成本 × 1000) / (吞吐量 tokens/s × 3600)

各方案每 1K tokens 成本（70B 模型）：
────────────────────────────────────────────────────────────────────────────
方案                    GPU配置            吞吐量      成本/1K tokens
────────────────────────────────────────────────────────────────────────────
FP16 固定集群           8×A100 固定8台     250 t/s     $0.0178
FP16 + 连续Batch        8×A100 固定8台     400 t/s     $0.0111
INT8 + 连续Batch        4×A100 固定4台     550 t/s     $0.0040
INT8 + 弹性             4×A100 弹性        520 t/s     $0.0027
INT4 AWQ + 弹性         2×A100 弹性        650 t/s     $0.0017
INT4 AWQ + Spec.Dec.    2×A100 + 1×L4      850 t/s     $0.0010
────────────────────────────────────────────────────────────────────────────

按请求类型细分成本（INT4 AWQ + 弹性方案）：
  短对话（input 200 + output 100 = 300 tokens）：$0.0003/请求
  中等请求（input 1000 + output 500 = 1500 tokens）：$0.0015/请求
  长文档（input 4000 + output 2000 = 6000 tokens）：$0.0060/请求
  代码生成（input 2000 + output 1500 = 3500 tokens）：$0.0035/请求

定价策略建议（保持 50% 毛利率）：
  input tokens：$0.002/1K tokens
  output tokens：$0.004/1K tokens（输出成本是输入的 2 倍，因为自回归生成更慢）
```

### 延迟百分位分析（不同 Batch Size）

不同 batch size 下的延迟分布差异巨大。以下是 70B INT8 模型在不同 batch size 下的延迟百分位实测数据。

```
延迟百分位数据（INT8 量化，输入 1000 tokens，输出 500 tokens）：
─────────────────────────────────────────────────────────────────────────────────
Batch   P50首token  P95首token  P99首token  P50总延迟   P99总延迟   吞吐量
Size    (ms)        (ms)        (ms)        (ms)        (ms)        (tokens/s)
─────────────────────────────────────────────────────────────────────────────────
1       150         180         200         12,600      12,800      20
5       200         350         500         13,200      15,000      85
10      250         500         800         14,000      18,000      150
20      350         700         1,200       15,500      22,000      250
40      500         1,200       2,000       18,000      30,000      350
60      700         1,800       3,500       22,000      45,000      400
80      1,000       2,500       5,000       28,000      60,000      420
─────────────────────────────────────────────────────────────────────────────────

关键洞察：
  1. Batch 1→10：吞吐量 7.5 倍提升，P99 首 token 延迟仅 4 倍增加
     → 性价比最高的区间，推荐默认 batch size = 10~20

  2. Batch 20→40：吞吐量 1.4 倍提升，但 P99 首 token 延迟 1.7 倍增加
     → 边际收益递减，仅在吞吐量压力大时使用

  3. Batch 40→80：吞吐量仅 1.2 倍提升，P99 首 token 延迟 2.5 倍增加
     → 不推荐，延迟恶化严重而吞吐量几乎不再增长

  4. P50 vs P99 差距随 batch size 增大而急剧扩大：
     Batch 1：  P99/P50 = 1.3x（可预测）
     Batch 40： P99/P50 = 4.0x（长尾严重）
     Batch 80： P99/P50 = 5.0x（极不可预测）
     → 原因：大 batch 中长请求 Prefill 阻塞短请求的解码
```

```
不同输入长度下的 Prefill 延迟（单 GPU，INT8）：
───────────────────────────────────────────────
输入 tokens   Prefill 时间(ms)   首 token 延迟
───────────────────────────────────────────────
128           30                  50ms
256           55                  80ms
512           100                 130ms
1,024         200                 240ms
2,048         420                 470ms
4,096         900                 960ms
8,192         1,900               1,980ms
───────────────────────────────────────────────
注意：Prefill 时间近似线性增长（O(n)），因为可以并行计算所有输入 token。
但 8K+ tokens 的请求 Prefill 时间接近 2 秒，会严重阻塞同 batch 的短请求。
→ 使用 Chunked Prefill 将长输入分块处理，每块 512 tokens（~100ms）。
```

### GPU 利用率优化深度路径

从 40% 利用率到 88% 利用率的每一步优化，其原理、实现复杂度和收益如下：

```
优化步骤详解：
───────────────────────────────────────────────────────────────────────────────
步骤              原理                          实现复杂度   利用率提升   成本节省
───────────────────────────────────────────────────────────────────────────────
1.连续Batch        消除请求间隙，                中等         40%→65%     39%
                   请求完成即加入新请求

2.PagedAttention   KV Cache按需分配，            高           65%→75%     13%
                   消除75%内存浪费，
                   支持更多并发

3.INT8量化         权重+KV Cache减半，           低           75%→80%     44%
                   同样显存支持2倍并发，
                   精度损失<1%

4.弹性扩缩容       按需分配GPU，                 中等         80%→78%     33%
                   低谷缩容（利用率略降
                   但绝对成本大幅下降）

5.Speculative      小模型生成候选，              高           78%→88%     34%
  Decoding         大模型批量验证，
                   接受率70%时等效
                   速度3.5倍

6.KV Cache量化     KV Cache FP16→INT8，         低           88%→91%     10%
  (INT8)           显存再省50%，
                   支持更多并发

7.Prefix Caching   共享system prompt的           中等         91%→93%     8%
                   KV Cache跨请求复用，
                   跳过重复计算
───────────────────────────────────────────────────────────────────────────────
总计：利用率 40%→93%，成本降低 92%

复杂度-收益矩阵：
  高收益 + 低复杂度 → 优先做：INT8 量化、KV Cache 量化
  高收益 + 高复杂度 → 值得做：连续 Batch、Speculative Decoding
  低收益 + 低复杂度 → 顺手做：弹性扩缩容参数调优
  低收益 + 高复杂度 → 暂缓做：Prefix Caching（需改动推理框架）
```

### 不同场景下的成本最优配置

```
场景              日均请求   峰值QPS  推荐配置                      月成本
────────────────────────────────────────────────────────────────────────
企业内部工具       10,000    50      2×A100 INT8, 固定             $4,000
中型SaaS产品      100,000   500     6×A100 INT8, 弹性             $8,500
大型API平台      1,000,000  5,000   15×A100 INT4, 弹性+预测扩容   $3,200
对话式AI产品     500,000    2,000   10×A100 INT8, SSE流式         $6,000
代码补全         2,000,000  10,000  20×L4 INT4, Speculative      $4,500
```

## 延伸思考

- **Speculative Decoding**：用小模型（7B）快速生成草稿 token，大模型批量验证 → 吞吐量提升 2-3 倍，精度无损。具体原理：小模型以 100 tokens/s 速度生成 5 个候选 token，大模型一次前向传播验证 5 个 token → 接受率约 70% → 等效速度 = 100 × 0.7 × 5 = 350 tokens/s（vs 原始 50 tokens/s）。关键约束：小模型和大模型的 tokenizer 必须一致。

- **MoE（Mixture of Experts）**：Mixtral 8×7B 只有 2 个 Expert 激活 → 推理成本 = 14B 模型 → 成本降 80%。但 MoE 的显存需求仍需加载全部 8 个 Expert（56B 权重）→ 需要更大显存或 Expert Offloading（将不活跃 Expert 放到 CPU 内存，按需加载到 GPU）。

- **Serverless GPU**：按请求计费（$0.0001/token）→ 无需管理集群 → 适合低流量场景。但冷启动延迟 30-50 秒 → 不适合实时交互。适用场景：批量离线推理、低频 API 调用、开发测试环境。

- **Prefill/Decode 分离**：Prefill（计算密集）在 A100 上做，Decode（内存密集）在 L4 上做 → 成本降低 50%。架构：Prefill 节点处理输入 → 将 KV Cache 传输到 Decode 节点 → Decode 节点持续生成 token。KV Cache 传输延迟约 10ms（同机房 RDMA），跨机房约 50ms（需权衡）。

- **KV Cache 跨请求复用（Prefix Caching）**：相同 system prompt 的请求共享 KV Cache → 重复输入不重复计算。例如所有请求共享 500 token 的 system prompt → 每个请求节省 200ms Prefill 时间。实现：对 prompt 前缀做哈希，命中缓存则跳过已计算的 token。

- **多租户隔离**：不同客户运行在同一 GPU 集群 → 需要确保：1）KV Cache 隔离（不同租户的 KV Cache 页不混用）；2）性能隔离（一个租户的长请求不影响另一个租户的延迟）；3）计费隔离（精确计量每个租户的 GPU 使用量）。实现方案：虚拟 GPU 切分（MPS）+ 优先级队列 + 令牌桶限流。

- **模型蒸馏替代大模型推理**：对于高频低精度场景（如客服自动回复），用大模型蒸馏出的小模型（7B）替代 70B → 推理成本降 90%，延迟降 80%。关键：定期用大模型生成标注数据，持续蒸馏更新小模型，保持输出质量。
## 模型 A/B 测试与金丝雀部署完整实现

```python
class ModelCanaryService:
    """模型 A/B 测试与金丝雀部署"""

    def deploy_canary(self, model_name, new_version, canary_percentage=5):
        """部署金丝雀模型"""
        # 1. 注册新模型版本
        self.model_registry.register(model_name, new_version)

        # 2. 预热新模型（加载权重 + 预热推理）
        self.inference_service.warmup(model_name, new_version)

        # 3. 配置流量分割
        self.router.set_weights(model_name, {
            "baseline": 100 - canary_percentage,
            "canary": canary_percentage
        })

        # 4. 启动金丝雀监控
        canary_id = str(uuid4())
        self.db.insert("model_canaries", {
            "canary_id": canary_id,
            "model_name": model_name,
            "baseline_version": self._get_current_version(model_name),
            "canary_version": new_version,
            "canary_percentage": canary_percentage,
            "status": "running",
            "started_at": now()
        })

        return canary_id

    def evaluate_canary(self, canary_id):
        """评估金丝雀结果"""
        canary = self.db.get_canary(canary_id)

        # 收集指标
        baseline_metrics = self._collect_model_metrics(
            canary["model_name"], canary["baseline_version"])
        canary_metrics = self._collect_model_metrics(
            canary["model_name"], canary["canary_version"])

        # 对比判定
        decision = "promote"
        reasons = []

        # 延迟对比
        if canary_metrics["p99_latency_ms"] > baseline_metrics["p99_latency_ms"] * 1.2:
            decision = "rollback"
            reasons.append(f"P99 延迟增加 {(canary_metrics['p99_latency_ms'] / baseline_metrics['p99_latency_ms'] - 1) * 100:.0f}%")

        # 准确率对比
        if canary_metrics["accuracy"] < baseline_metrics["accuracy"] - 0.02:
            decision = "rollback"
            reasons.append(f"准确率下降 {(baseline_metrics['accuracy'] - canary_metrics['accuracy']) * 100:.1f}%")

        # 错误率对比
        if canary_metrics["error_rate"] > baseline_metrics["error_rate"] * 2:
            decision = "rollback"
            reasons.append(f"错误率增加 {canary_metrics['error_rate'] / baseline_metrics['error_rate'] - 1:.0%}")

        return {
            "canary_id": canary_id,
            "decision": decision,
            "baseline": baseline_metrics,
            "canary": canary_metrics,
            "reasons": reasons
        }

    def _collect_model_metrics(self, model_name, version):
        """收集模型指标"""
        return {
            "p50_latency_ms": float(self.redis.get(f"model:{model_name}:{version}:p50") or 0),
            "p99_latency_ms": float(self.redis.get(f"model:{model_name}:{version}:p99") or 0),
            "accuracy": float(self.redis.get(f"model:{model_name}:{version}:accuracy") or 0),
            "error_rate": float(self.redis.get(f"model:{model_name}:{version}:error_rate") or 0),
            "qps": float(self.redis.get(f"model:{model_name}:{version}:qps") or 0),
            "cost_per_1k_inferences": float(self.redis.get(f"model:{model_name}:{version}:cost") or 0),
        }
```

## 推理请求路由

```python
class InferenceRequestRouter:
    """推理请求路由：策略 + GPU 亲和 + 优先级"""

    ROUTING_STRATEGIES = {
        "round_robin": "轮询",
        "least_latency": "最低延迟优先",
        "gpu_affinity": "GPU 亲和（KV 缓存复用）",
        "capability_based": "模型能力匹配",
    }

    def route(self, request):
        """路由推理请求"""
        model_name = request["model"]
        strategy = self._get_strategy(model_name)

        if strategy == "gpu_affinity":
            # 尝试路由到有 KV 缓存的 GPU
            target = self._find_cached_gpu(request)
            if not target:
                target = self._find_least_loaded_gpu(model_name)
        elif strategy == "least_latency":
            target = self._find_least_latency_endpoint(model_name)
        elif strategy == "capability_based":
            target = self._find_capable_endpoint(request)
        else:
            target = self._round_robin(model_name)

        # 熔断检查
        if self._is_circuit_open(target):
            target = self._find_fallback(model_name, exclude=target)

        return target

    def _find_cached_gpu(self, request):
        """查找有 KV 缓存的 GPU"""
        session_id = request.get("session_id")
        if session_id:
            gpu_id = self.redis.get(f"kv_cache_gpu:{session_id}")
            if gpu_id and self._is_healthy(gpu_id):
                return gpu_id
        return None

    def _is_circuit_open(self, endpoint):
        """熔断检查"""
        error_count = int(self.redis.get(f"circuit:{endpoint}:errors") or 0)
        return error_count > 10  # 10 次错误 → 熔断

    def enqueue_batch(self, requests, max_batch_size=32, max_wait_ms=50):
        """请求合并（Batch 推理）"""
        batch_key = f"batch:{requests[0]['model']}"
        for req in requests:
            self.redis.rpush(batch_key, json.dumps(req))

        # 等待凑够 batch 或超时
        batch = []
        deadline = now() + timedelta(milliseconds=max_wait_ms)
        while len(batch) < max_batch_size and now() < deadline:
            item = self.redis.lpop(batch_key)
            if item:
                batch.append(json.loads(item))
            else:
                time.sleep(0.005)

        if batch:
            return self.inference_service.batch_infer(batch)
        return []
```

## 异常场景补充

### 场景：金丝雀部署 KV 缓存抖动

```
触发：金丝雀模型与基线模型使用不同 GPU → KV 缓存全部失效 → 延迟飙升
检测：
  1. 金丝雀模型 P99 延迟 > 基线 × 3 → KV 缓存未命中
  2. GPU 显存使用率下降 → 缓存被清除
处理：
  1. 金丝雀模型与基线模型共享 GPU 集群
  2. 流量分割时保持 session 亲和（同一用户路由到同一模型）
  3. 预热金丝雀模型 KV 缓存
预防：session 亲和 + 共享 GPU + 缓存预热
```

### 场景：路由不均衡导致 GPU 热点

```
触发：least_latency 策略 → 所有请求集中到最快的 GPU → 该 GPU 过载
检测：
  1. 单 GPU 利用率 > 95% → 热点
  2. 其他 GPU 利用率 < 30% → 不均衡
处理：
  1. 切换到加权轮询策略
  2. 热点 GPU 设置最大并发限制
  3. 溢出请求路由到其他 GPU
预防：最大并发限制 + 加权轮询 + 负载均衡监控
```

## 模型 A/B 测试与金丝雀部署（完整版）

### 模型版本注册中心

```python
import hashlib
import uuid
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from typing import List, Dict, Optional, Tuple
from enum import Enum
import statistics
import json


class ModelStatus(Enum):
    REGISTERED = "registered"       # 已注册
    WARMING_UP = "warming_up"       # 预热中
    READY = "ready"                 # 就绪
    CANARY = "canary"               # 金丝雀部署中
    PRODUCTION = "production"       # 生产流量
    DEPRECATED = "deprecated"       # 已弃用
    RETIRED = "retired"             # 已下线


@dataclass
class ModelVersion:
    """模型版本"""
    model_name: str
    version: str
    status: ModelStatus
    artifact_path: str             # 模型文件路径
    config: Dict                   # 模型配置
    registered_at: datetime
    warmed_up_at: Optional[datetime] = None
    promoted_at: Optional[datetime] = None
    retired_at: Optional[datetime] = None


class ModelVersionRegistry:
    """
    模型版本注册中心
    管理模型版本的生命周期
    """

    def __init__(self, db_pool, storage_client):
        self.db = db_pool
        self.storage = storage_client

    async def register(self, model_name: str, version: str,
                        artifact_path: str, config: Dict) -> ModelVersion:
        """注册新模型版本"""
        # 校验模型文件完整性
        checksum = await self._verify_artifact(artifact_path)

        model_version = ModelVersion(
            model_name=model_name,
            version=version,
            status=ModelStatus.REGISTERED,
            artifact_path=artifact_path,
            config=config,
            registered_at=datetime.utcnow()
        )

        await self.db.execute(
            """INSERT INTO model_versions
               (model_name, version, status, artifact_path, config,
                checksum, registered_at)
               VALUES (%s, %s, %s, %s, %s, %s, %s)""",
            model_name, version, model_version.status.value,
            artifact_path, json.dumps(config),
            checksum, model_version.registered_at
        )

        return model_version

    async def promote_to_production(self, model_name: str,
                                     version: str) -> None:
        """将模型版本提升为生产版本"""
        # 先将当前生产版本降级
        await self.db.execute(
            """UPDATE model_versions
               SET status = 'deprecated', deprecated_at = NOW()
               WHERE model_name = %s AND status = 'production'""",
            model_name
        )

        # 提升新版本
        await self.db.execute(
            """UPDATE model_versions
               SET status = 'production', promoted_at = NOW()
               WHERE model_name = %s AND version = %s""",
            model_name, version
        )

    async def _verify_artifact(self, path: str) -> str:
        """校验模型文件完整性"""
        h = hashlib.sha256()
        async for chunk in self.storage.read_chunks(path):
            h.update(chunk)
        return h.hexdigest()


class ModelABTestService:
    """
    模型 A/B 测试服务
    流量分割 + 指标收集 + 统计显著性检验 + 自动优胜选择
    """

    def __init__(self, db_pool, redis_client, inference_service,
                 alert_service):
        self.db = db_pool
        self.redis = redis_client
        self.inference = inference_service
        self.alert = alert_service
        self.registry = ModelVersionRegistry(db_pool, None)

    async def split_traffic(self, model_name: str,
                             weights: Dict[str, float]) -> None:
        """
        配置流量分割
        weights: {"v1.0": 80, "v2.0": 20} → 80%/20% 分割
        使用一致性哈希保证同一 request_id 始终路由到同一版本
        """
        # 归一化权重
        total = sum(weights.values())
        normalized = {k: v / total for k, v in weights.items()}

        # 存储到 Redis
        self.redis.set(
            f"model_traffic:{model_name}",
            json.dumps(normalized)
        )

        # 更新路由表
        await self.db.execute(
            """UPDATE model_traffic_split
               SET weights = %s, updated_at = NOW()
               WHERE model_name = %s""",
            json.dumps(normalized), model_name
        )

    def route_request(self, model_name: str,
                       request_id: str) -> str:
        """
        根据一致性哈希路由请求到模型版本
        相同 request_id 始终路由到同一版本
        """
        weights_json = self.redis.get(f"model_traffic:{model_name}")
        if not weights_json:
            return "default"

        weights = json.loads(weights_json)

        # 一致性哈希：对 request_id 哈希，映射到 [0, 1) 区间
        hash_val = int(hashlib.md5(
            request_id.encode()).hexdigest(), 16) / (2**128)

        cumulative = 0.0
        for version, weight in sorted(weights.items()):
            cumulative += weight
            if hash_val < cumulative:
                return version

        # 兜底：返回最大权重版本
        return max(weights, key=weights.get)

    async def collect_metrics(self, model_name: str,
                               version: str) -> Dict:
        """收集指定模型版本的指标"""
        key_prefix = f"model_metrics:{model_name}:{version}"

        # 从 Redis 读取实时指标
        metrics = {
            "p50_latency_ms": float(
                self.redis.get(f"{key_prefix}:p50") or 0),
            "p99_latency_ms": float(
                self.redis.get(f"{key_prefix}:p99") or 0),
            "accuracy": float(
                self.redis.get(f"{key_prefix}:accuracy") or 0),
            "error_rate": float(
                self.redis.get(f"{key_prefix}:error_rate") or 0),
            "qps": float(
                self.redis.get(f"{key_prefix}:qps") or 0),
            "cost_per_1k_inferences": float(
                self.redis.get(f"{key_prefix}:cost") or 0),
            "total_requests": int(
                self.redis.get(f"{key_prefix}:total") or 0),
            "kv_cache_hit_rate": float(
                self.redis.get(f"{key_prefix}:kv_hit_rate") or 0),
        }

        return metrics


class SequentialABTest:
    """
    序贯 A/B 检验（Sequential Test）
    不需要预先确定样本量，达到统计显著性即可提前结束
    """

    def __init__(self, significance_level: float = 0.05,
                 power: float = 0.8,
                 min_effect_size: float = 0.02):
        self.alpha = significance_level
        self.power = power
        self.min_effect = min_effect_size

    def test(self, baseline_metrics: Dict,
             candidate_metrics: Dict) -> Dict:
        """
        执行序贯检验
        返回：是否显著、效果大小、建议
        """
        n_baseline = baseline_metrics.get("total_requests", 0)
        n_candidate = candidate_metrics.get("total_requests", 0)

        if n_baseline < 100 or n_candidate < 100:
            return {
                "significant": False,
                "reason": "样本量不足（需要至少 100 个请求）",
                "recommendation": "continue"
            }

        # 延迟对比（双样本 t 检验近似）
        baseline_p99 = baseline_metrics.get("p99_latency_ms", 0)
        candidate_p99 = candidate_metrics.get("p99_latency_ms", 0)
        latency_change = ((candidate_p99 - baseline_p99)
                          / baseline_p99 * 100) if baseline_p99 > 0 else 0

        # 准确率对比（比例检验近似）
        baseline_acc = baseline_metrics.get("accuracy", 0)
        candidate_acc = candidate_metrics.get("accuracy", 0)
        acc_diff = candidate_acc - baseline_acc

        # 错误率对比
        baseline_err = baseline_metrics.get("error_rate", 0)
        candidate_err = candidate_metrics.get("error_rate", 0)
        err_ratio = (candidate_err / baseline_err
                     if baseline_err > 0 else float('inf'))

        # 成本对比
        baseline_cost = baseline_metrics.get("cost_per_1k_inferences", 0)
        candidate_cost = candidate_metrics.get("cost_per_1k_inferences", 0)
        cost_change = ((candidate_cost - baseline_cost)
                       / baseline_cost * 100) if baseline_cost > 0 else 0

        # 综合判定
        is_significant = n_candidate >= 500  # 简化：至少 500 样本
        recommendation = "continue"

        # 优胜条件：延迟不增加 > 20% + 准确率不降低 > 2% + 错误率不翻倍
        if (latency_change > 20 or acc_diff < -0.02
                or err_ratio > 2):
            recommendation = "rollback"
        elif (latency_change <= 5 and acc_diff >= 0
              and err_ratio <= 1.2 and cost_change <= 10):
            if is_significant:
                recommendation = "promote"

        return {
            "significant": is_significant,
            "latency_change_pct": round(latency_change, 1),
            "accuracy_diff": round(acc_diff, 4),
            "error_rate_ratio": round(err_ratio, 2),
            "cost_change_pct": round(cost_change, 1),
            "recommendation": recommendation,
            "sample_sizes": {
                "baseline": n_baseline,
                "candidate": n_candidate
            }
        }


class ModelCanaryDeployment:
    """
    模型金丝雀部署控制器
    全流程：注册 → 预热 → 流量切换 → 指标监控 → 自动提升/回滚
    """

    CANARY_PERCENTAGES = [1, 5, 10, 25, 50, 100]  # 渐进式流量

    def __init__(self, db_pool, redis_client, inference_service,
                 alert_service):
        self.db = db_pool
        self.redis = redis_client
        self.inference = inference_service
        self.alert = alert_service
        self.ab_test = SequentialABTest()
        self.traffic = ModelABTestService(
            db_pool, redis_client, inference_service, alert_service)

    async def start_canary(self, model_name: str,
                            new_version: str,
                            initial_percentage: float = 1.0) -> str:
        """启动金丝雀部署"""
        # 1. 获取当前生产版本
        current = await self._get_production_version(model_name)

        # 2. 预热新模型
        await self.inference.warmup(model_name, new_version)

        # 3. 注册金丝雀部署
        canary_id = str(uuid.uuid4())
        await self.db.execute(
            """INSERT INTO model_canaries
               (canary_id, model_name, baseline_version,
                canary_version, current_percentage, status,
                started_at)
               VALUES (%s, %s, %s, %s, %s, 'running', NOW())""",
            canary_id, model_name, current,
            new_version, initial_percentage
        )

        # 4. 配置初始流量分割
        await self.traffic.split_traffic(model_name, {
            current: 100 - initial_percentage,
            new_version: initial_percentage
        })

        return canary_id

    async def evaluate_and_advance(self, canary_id: str) -> Dict:
        """评估金丝雀结果，决定提升、继续或回滚"""
        canary = await self.db.fetch_one(
            """SELECT * FROM model_canaries
               WHERE canary_id = %s""", canary_id
        )

        if not canary:
            return {"error": "canary not found"}

        # 收集双版本指标
        baseline = await self.traffic.collect_metrics(
            canary["model_name"], canary["baseline_version"])
        candidate = await self.traffic.collect_metrics(
            canary["model_name"], canary["canary_version"])

        # 执行统计检验
        test_result = self.ab_test.test(baseline, candidate)

        # 自动回滚：指标退化
        if test_result["recommendation"] == "rollback":
            await self._rollback_canary(canary_id, canary, test_result)
            return {
                "action": "rollback",
                "reason": test_result,
                "canary_id": canary_id
            }

        # 自动提升：指标优于基线
        if test_result["recommendation"] == "promote":
            await self._promote_canary(canary_id, canary)
            return {
                "action": "promote",
                "reason": test_result,
                "canary_id": canary_id
            }

        # 继续观察：提升流量比例
        current_pct = float(canary["current_percentage"])
        next_pct = self._get_next_percentage(current_pct)
        if next_pct > current_pct:
            await self._advance_traffic(canary, next_pct)
            return {
                "action": "advance",
                "new_percentage": next_pct,
                "reason": test_result,
                "canary_id": canary_id
            }

        return {
            "action": "continue",
            "reason": test_result,
            "canary_id": canary_id
        }

    async def _rollback_canary(self, canary_id: str,
                                canary: Dict, reason: Dict) -> None:
        """回滚金丝雀部署"""
        # 流量全部切回基线版本
        await self.traffic.split_traffic(canary["model_name"], {
            canary["baseline_version"]: 100,
            canary["canary_version"]: 0
        })

        # 更新状态
        await self.db.execute(
            """UPDATE model_canaries
               SET status = 'rolled_back',
                   rollback_reason = %s,
                   rolled_back_at = NOW()
               WHERE canary_id = %s""",
            json.dumps(reason), canary_id
        )

        # 告警
        await self.alert.send_warning(
            title=f"金丝雀回滚 - {canary['model_name']}",
            message=(
                f"金丝雀版本: {canary['canary_version']}\n"
                f"基线版本: {canary['baseline_version']}\n"
                f"回滚原因: {reason}"
            )
        )

    async def _promote_canary(self, canary_id: str,
                               canary: Dict) -> None:
        """提升金丝雀版本为生产版本"""
        # 流量全部切到新版本
        await self.traffic.split_traffic(canary["model_name"], {
            canary["canary_version"]: 100
        })

        # 更新注册中心
        registry = ModelVersionRegistry(self.db, None)
        await registry.promote_to_production(
            canary["model_name"], canary["canary_version"])

        # 更新金丝雀状态
        await self.db.execute(
            """UPDATE model_canaries
               SET status = 'promoted',
                   promoted_at = NOW(),
                   current_percentage = 100
               WHERE canary_id = %s""",
            canary_id
        )

    def _get_next_percentage(self, current: float) -> float:
        """获取下一个流量比例"""
        for pct in self.CANARY_PERCENTAGES:
            if pct > current:
                return pct
        return 100.0

    async def _advance_traffic(self, canary: Dict,
                                new_pct: float) -> None:
        """逐步提升金丝雀流量"""
        await self.traffic.split_traffic(canary["model_name"], {
            canary["baseline_version"]: 100 - new_pct,
            canary["canary_version"]: new_pct
        })

        await self.db.execute(
            """UPDATE model_canaries
               SET current_percentage = %s
               WHERE canary_id = %s""",
            new_pct, canary["canary_id"]
        )

    async def _get_production_version(self,
                                       model_name: str) -> str:
        """获取当前生产版本"""
        row = await self.db.fetch_one(
            """SELECT version FROM model_versions
               WHERE model_name = %s AND status = 'production'""",
            model_name
        )
        return row["version"] if row else "default"
```

## 推理请求路由（完整版）

### 路由策略与 GPU 亲和

```python
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from typing import List, Dict, Optional, Set
from enum import Enum
import hashlib
import json
import time
import threading


class RoutingStrategy(Enum):
    ROUND_ROBIN = "round_robin"             # 轮询
    LEAST_LATENCY = "least_latency"         # 最低延迟优先
    GPU_AFFINITY = "gpu_affinity"           # GPU 亲和（KV 缓存复用）
    CAPABILITY_BASED = "capability_based"   # 模型能力匹配
    WEIGHTED_ROUND_ROBIN = "weighted_rr"    # 加权轮询


class RequestPriority(Enum):
    REALTIME = "realtime"     # 实时请求（延迟敏感）
    BATCH = "batch"           # 批量请求（吞吐优先）
    BACKGROUND = "background" # 后台任务（最低优先级)


@dataclass
class GPUEndpoint:
    """GPU 推理端点"""
    endpoint_id: str
    gpu_id: str
    model_name: str
    model_version: str
    capacity: int               # 最大并发
    current_load: int           # 当前并发
    avg_latency_ms: float       # 平均延迟
    is_healthy: bool            # 健康状态
    kv_cache_sessions: Set[str] = field(default_factory=set)  # 已缓存 session
    capabilities: List[str] = field(default_factory=list)     # 模型能力标签


@dataclass
class InferenceRequest:
    """推理请求"""
    request_id: str
    model_name: str
    session_id: Optional[str]     # 会话 ID（用于 KV 缓存亲和）
    priority: RequestPriority
    input_tokens: int
    max_output_tokens: int
    capability_tags: List[str] = field(default_factory=list)
    timeout_ms: int = 30000


class CircuitBreaker:
    """
    熔断器（per model endpoint）
    状态：CLOSED → OPEN → HALF_OPEN → CLOSED
    """

    def __init__(self, failure_threshold: int = 10,
                 recovery_timeout: int = 30,
                 half_open_max_requests: int = 3):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.half_open_max = half_open_max_requests

        self.state = "closed"           # closed / open / half_open
        self.failure_count = 0
        self.last_failure_time = None
        self.half_open_requests = 0
        self._lock = threading.Lock()

    def allow_request(self) -> bool:
        """是否允许请求通过"""
        with self._lock:
            if self.state == "closed":
                return True
            elif self.state == "open":
                # 检查是否超过恢复时间
                if (self.last_failure_time and
                    time.time() - self.last_failure_time >
                        self.recovery_timeout):
                    self.state = "half_open"
                    self.half_open_requests = 0
                    return True
                return False
            else:  # half_open
                if self.half_open_requests < self.half_open_max:
                    self.half_open_requests += 1
                    return True
                return False

    def record_success(self):
        """记录成功"""
        with self._lock:
            if self.state == "half_open":
                self.state = "closed"
            self.failure_count = 0

    def record_failure(self):
        """记录失败"""
        with self._lock:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "open"


class InferenceRouter:
    """
    推理请求路由器
    支持：轮询、最低延迟、GPU 亲和、能力匹配
    特性：优先级队列、请求合并、熔断器、回退路由
    """

    def __init__(self, redis_client, db_pool, inference_service):
        self.redis = redis_client
        self.db = db_pool
        self.inference = inference_service
        self.circuit_breakers: Dict[str, CircuitBreaker] = {}
        self._rr_counters: Dict[str, int] = {}

    async def route(self, request: InferenceRequest) -> Dict:
        """
        路由推理请求
        返回选中的端点信息
        """
        strategy = self._get_strategy(request.model_name)

        # 获取所有可用端点
        endpoints = await self._get_endpoints(request.model_name)

        # 过滤掉熔断的端点
        healthy = [ep for ep in endpoints
                   if self._get_breaker(ep.endpoint_id).allow_request()]

        if not healthy:
            # 所有端点熔断 → 回退到基线模型
            fallback = await self._find_fallback(request)
            if fallback:
                return {
                    "endpoint": fallback,
                    "strategy": "fallback",
                    "original_model": request.model_name
                }
            raise Exception(
                f"无可用端点: model={request.model_name}")

        # 按策略选择端点
        if strategy == RoutingStrategy.GPU_AFFINITY:
            target = self._gpu_affinity_route(healthy, request)
        elif strategy == RoutingStrategy.LEAST_LATENCY:
            target = self._least_latency_route(healthy, request)
        elif strategy == RoutingStrategy.CAPABILITY_BASED:
            target = self._capability_route(healthy, request)
        elif strategy == RoutingStrategy.WEIGHTED_ROUND_ROBIN:
            target = self._weighted_rr_route(healthy, request)
        else:
            target = self._round_robin_route(healthy, request)

        # 更新负载
        target.current_load += 1

        # 记录 KV 缓存亲和
        if request.session_id:
            self.redis.setex(
                f"kv_cache_gpu:{request.session_id}",
                1800,  # 30 分钟 TTL
                target.endpoint_id
            )

        return {
            "endpoint": target,
            "strategy": strategy.value,
            "fallback": False
        }

    def _gpu_affinity_route(self, endpoints: List[GPUEndpoint],
                             request: InferenceRequest) -> GPUEndpoint:
        """
        GPU 亲和路由：优先路由到有 KV 缓存的 GPU
        同一会话的请求路由到同一 GPU → KV Cache 复用
        """
        if request.session_id:
            # 查找缓存了该 session 的 GPU
            cached_gpu = self.redis.get(
                f"kv_cache_gpu:{request.session_id}")
            if cached_gpu:
                for ep in endpoints:
                    if (ep.endpoint_id == cached_gpu.decode()
                            and ep.is_healthy
                            and ep.current_load < ep.capacity):
                        return ep

        # 没有缓存 → 选择负载最低的 GPU
        return min(endpoints,
                   key=lambda ep: ep.current_load / ep.capacity)

    def _least_latency_route(self, endpoints: List[GPUEndpoint],
                              request: InferenceRequest) -> GPUEndpoint:
        """最低延迟路由：选择 P99 延迟最低的端点"""
        # 加入随机扰动防止所有请求集中到同一端点
        import random
        jitter = random.uniform(0.9, 1.1)
        return min(endpoints,
                   key=lambda ep: ep.avg_latency_ms * jitter)

    def _capability_route(self, endpoints: List[GPUEndpoint],
                           request: InferenceRequest) -> GPUEndpoint:
        """能力匹配路由：根据请求标签匹配模型能力"""
        required = set(request.capability_tags)

        # 优先完全匹配
        for ep in endpoints:
            if required.issubset(set(ep.capabilities)):
                return ep

        # 降级：最大交集
        return max(endpoints,
                   key=lambda ep: len(required & set(ep.capabilities)))

    def _round_robin_route(self, endpoints: List[GPUEndpoint],
                            request: InferenceRequest) -> GPUEndpoint:
        """轮询路由"""
        key = request.model_name
        idx = self._rr_counters.get(key, 0)
        selected = endpoints[idx % len(endpoints)]
        self._rr_counters[key] = idx + 1
        return selected

    def _weighted_rr_route(self, endpoints: List[GPUEndpoint],
                            request: InferenceRequest) -> GPUEndpoint:
        """加权轮询路由：按剩余容量加权"""
        weights = [max(0, ep.capacity - ep.current_load)
                   for ep in endpoints]
        total = sum(weights)
        if total == 0:
            return endpoints[0]

        import random
        r = random.uniform(0, total)
        cumulative = 0
        for ep, w in zip(endpoints, weights):
            cumulative += w
            if r <= cumulative:
                return ep
        return endpoints[-1]

    async def enqueue_priority(self, request: InferenceRequest) -> Dict:
        """
        优先级队列
        实时请求优先处理，批量请求凑批执行
        """
        if request.priority == RequestPriority.REALTIME:
            # 实时请求：直接路由，不等待凑批
            return await self.route(request)

        elif request.priority == RequestPriority.BATCH:
            # 批量请求：凑批执行
            return await self._coalesce_batch(request)

        else:
            # 后台任务：低优先级队列
            return await self._enqueue_background(request)

    async def _coalesce_batch(self,
                               request: InferenceRequest,
                               max_batch_size: int = 32,
                               max_wait_ms: int = 100) -> Dict:
        """
        请求合并（Batch Coalescing）
        将多个批量请求合并到一次 GPU 推理
        """
        batch_key = f"batch:{request.model_name}"

        # 加入待处理队列
        self.redis.rpush(
            batch_key,
            json.dumps({
                "request_id": request.request_id,
                "input_tokens": request.input_tokens,
                "max_output_tokens": request.max_output_tokens
            })
        )

        # 等待凑够 batch 或超时
        deadline = time.time() + max_wait_ms / 1000
        batch = []

        while len(batch) < max_batch_size and time.time() < deadline:
            item = self.redis.lpop(batch_key)
            if item:
                batch.append(json.loads(item))
            else:
                time.sleep(0.01)

        if batch:
            # 执行批量推理
            results = await self.inference.batch_infer(
                request.model_name, batch)
            return {
                "batch_size": len(batch),
                "results": results
            }

        return {"batch_size": 0}

    async def _enqueue_background(self,
                                   request: InferenceRequest) -> Dict:
        """后台任务入队"""
        queue_key = f"bg_queue:{request.model_name}"
        self.redis.rpush(
            queue_key,
            json.dumps({
                "request_id": request.request_id,
                "model_name": request.model_name,
                "input_tokens": request.input_tokens,
                "max_output_tokens": request.max_output_tokens,
                "enqueued_at": datetime.utcnow().isoformat()
            })
        )
        return {"status": "enqueued", "queue": queue_key}

    async def _find_fallback(self,
                              request: InferenceRequest) -> Optional[GPUEndpoint]:
        """回退路由：主模型过载时使用备选模型"""
        fallback_models = {
            "llm-70b": "llm-13b",          # 70B → 13B 降级
            "llm-13b": "llm-7b",           # 13B → 7B 降级
            "code-70b": "code-13b",         # 代码模型降级
        }

        fallback_model = fallback_models.get(request.model_name)
        if not fallback_model:
            return None

        endpoints = await self._get_endpoints(fallback_model)
        healthy = [ep for ep in endpoints
                   if self._get_breaker(ep.endpoint_id).allow_request()]
        if healthy:
            return min(healthy,
                       key=lambda ep: ep.current_load / ep.capacity)

        return None

    def _get_breaker(self, endpoint_id: str) -> CircuitBreaker:
        """获取端点的熔断器"""
        if endpoint_id not in self.circuit_breakers:
            self.circuit_breakers[endpoint_id] = CircuitBreaker()
        return self.circuit_breakers[endpoint_id]

    async def _get_endpoints(self,
                              model_name: str) -> List[GPUEndpoint]:
        """获取模型的所有端点"""
        rows = await self.db.fetch_all(
            """SELECT endpoint_id, gpu_id, model_name, model_version,
                 max_concurrency AS capacity,
                 current_concurrency AS current_load,
                 avg_latency_ms, is_healthy,
                 kv_cache_sessions, capabilities
               FROM gpu_endpoints
               WHERE model_name = %s AND is_healthy = TRUE""",
            model_name
        )

        return [
            GPUEndpoint(
                endpoint_id=r["endpoint_id"],
                gpu_id=r["gpu_id"],
                model_name=r["model_name"],
                model_version=r["model_version"],
                capacity=r["capacity"],
                current_load=r["current_load"],
                avg_latency_ms=float(r["avg_latency_ms"] or 0),
                is_healthy=r["is_healthy"],
                kv_cache_sessions=set(
                    r.get("kv_cache_sessions", []) or []),
                capabilities=r.get("capabilities", []) or []
            )
            for r in rows
        ]

    def _get_strategy(self, model_name: str) -> RoutingStrategy:
        """获取模型的默认路由策略"""
        strategy_map = {
            "llm-70b": RoutingStrategy.GPU_AFFINITY,
            "llm-13b": RoutingStrategy.LEAST_LATENCY,
            "code-70b": RoutingStrategy.CAPABILITY_BASED,
        }
        return strategy_map.get(model_name, RoutingStrategy.ROUND_ROBIN)
```

## 异常场景补充（续）

### 场景：模型金丝雀部署导致 KV 缓存抖动

```
触发：金丝雀模型 v2.0 与基线模型 v1.0 部署在不同 GPU 集群。
      流量按 5%/95% 分割后，5% 的请求被路由到新 GPU。
      但这些请求之前在 v1.0 的 GPU 上有 KV 缓存 → 路由到 v2.0 后
      KV 缓存全部失效 → 需要重新 Prefill → P99 延迟从 200ms 飙到 800ms。
      更严重的是：某些请求在 v1.0 和 v2.0 之间反复跳转
      （流量分割的一致性哈希不够稳定），导致 KV 缓存不断失效重建。
检测：
  1. 金丝雀版本 P99 延迟 > 基线 × 3 → KV 缓存未命中
  2. GPU 显存使用率下降（缓存被清除后新缓存尚未建立）
  3. Prefill 阶段耗时占比从 20% 上升到 60% → 大量缓存未命中
  4. 同一 session_id 在不同 GPU 之间跳转 → 亲和性失效
处理：
  1. 修改流量分割策略：基于 session_id 哈希而非 request_id
     → 保证同一会话始终路由到同一模型版本
  2. 金丝雀模型预热：提前用历史请求填充 KV 缓存
  3. 金丝雀模型与基线模型共享 GPU 集群（同机部署不同版本）
  4. 增加 KV 缓存迁移：模型版本切换时，将活跃 session 的
     KV 缓存预加载到新版本 GPU
  5. 降低金丝雀流量比例（5% → 1%），减少抖动影响范围
预防：session 级别流量亲和 + KV 缓存预热 + 共享 GPU 集群 + 渐进式流量切换
```

### 场景：路由不均衡导致 GPU 热点

```
触发：推理路由器使用 least_latency 策略。
      某个 GPU（gpu-07）因为请求队列短、响应快，
      被路由器选为所有请求的目标 → gpu-07 利用率飙到 98%，
      而其他 GPU 利用率仅 20-30%。
      结果：gpu-07 的请求排队时间增加 → 延迟反而升高
      → 路由器仍然选择它（因为历史 P99 仍低）→ 恶性循环。
      更严重时：gpu-07 OOM → 所有路由到它的请求失败。
检测：
  1. 单 GPU 利用率 > 95% 而其他 < 40% → 热点不均衡
  2. gpu-07 请求队列深度 > 100 → 过载
  3. gpu-07 的 OOM 错误 → 严重过载
  4. 路由决策日志：连续 100 次路由到同一 GPU → 不均衡
处理：
  1. 紧急切换到加权轮询策略（按 GPU 容量加权）
  2. 给每个 GPU 设置最大并发限制（capacity * 0.8）
  3. 超出最大并发的请求溢出到其他 GPU
  4. least_latency 策略增加随机扰动（±10% 延迟抖动）
  5. 增加"热点检测"：某 GPU 被连续选择超过 N 次 → 暂时降低权重
  6. gpu-07 OOM → 熔断该端点，流量重分配
预防：最大并发限制 + 加权轮询兜底 + 热点检测 + 熔断器 + 负载均衡监控

## AI 推理模型版本管理完整实现

```python
class ModelVersionManagementService:
    """模型版本管理：版本注册 → 灰度 → 全量 → 回退"""

    def register_model_version(self, model_name, version, artifact_path,
                               metrics, description=None):
        """注册模型版本"""
        # 1. 验证模型指标
        required_metrics = ["accuracy", "latency_p95_ms", "model_size_mb"]
        for m in required_metrics:
            if m not in metrics:
                raise InvalidModelError(f"缺少必需指标: {m}")

        # 2. 与当前线上版本对比
        current = self._get_production_version(model_name)
        if current:
            comparison = self._compare_versions(current["metrics"], metrics)
            if comparison["regression"]:
                # 存在回退 → 需要审批
                self.db.insert("model_version_reviews", {
                    "review_id": str(uuid4()),
                    "model_name": model_name,
                    "new_version": version,
                    "current_version": current["version"],
                    "regressions": json.dumps(comparison["regressions"]),
                    "status": "pending_review",
                    "created_at": now()
                })
                return {"status": "needs_review", "regressions": comparison["regressions"]}

        # 3. 注册版本
        version_id = str(uuid4())
        self.db.insert("model_versions", {
            "version_id": version_id,
            "model_name": model_name,
            "version": version,
            "artifact_path": artifact_path,
            "metrics": json.dumps(metrics),
            "description": description,
            "status": "registered",
            "registered_at": now()
        })

        return {"version_id": version_id, "model_name": model_name,
                "version": version, "status": "registered"}

    def deploy_version(self, model_name, version, strategy="canary",
                      canary_pct=5):
        """部署模型版本"""
        model_version = self.db.query_one(
            "SELECT * FROM model_versions "
            "WHERE model_name = %s AND version = %s",
            model_name, version)

        if not model_version:
            return {"status": "version_not_found"}

        if strategy == "canary":
            # 灰度部署
            self.db.update("model_versions",
                {"status": "canary", "canary_pct": canary_pct,
                 "canary_started_at": now()},
                {"version_id": model_version["version_id"]})

            # 更新路由规则
            self.redis.set(f"model_route:{model_name}", json.dumps({
                "production": self._get_production_version(model_name)["version"],
                "canary": version,
                "canary_pct": canary_pct
            }))

            return {"status": "canary_deployed", "canary_pct": canary_pct}

        elif strategy == "blue_green":
            # 蓝绿部署
            self.db.update("model_versions",
                {"status": "staging"}, {"version_id": model_version["version_id"]})

            self.redis.set(f"model_staging:{model_name}", json.dumps({
                "staging_version": version,
                "production_version": self._get_production_version(model_name)["version"],
                "switch_ready": False
            }))

            return {"status": "staging_ready"}

    def switch_blue_green(self, model_name, operator_id):
        """蓝绿切换"""
        staging = self.redis.get(f"model_staging:{model_name}")
        if not staging:
            return {"status": "no_staging"}

        staging_info = json.loads(staging)

        # 1. 验证 staging 版本健康
        health = self._check_model_health(model_name, staging_info["staging_version"])
        if not health["healthy"]:
            return {"status": "unhealthy", "issues": health["issues"]}

        # 2. 切换
        old_version = staging_info["production_version"]
        new_version = staging_info["staging_version"]

        self.redis.set(f"model_route:{model_name}", json.dumps({
            "production": new_version
        }))

        # 3. 更新版本状态
        self.db.update("model_versions",
            {"status": "production", "promoted_at": now()},
            {"model_name": model_name, "version": new_version})

        self.db.update("model_versions",
            {"status": "retired", "retired_at": now()},
            {"model_name": model_name, "version": old_version})

        # 4. 清理 staging
        self.redis.delete(f"model_staging:{model_name}")

        # 5. 记录
        self.db.insert("model_deployment_history", {
            "id": str(uuid4()), "model_name": model_name,
            "from_version": old_version, "to_version": new_version,
            "operator": operator_id, "strategy": "blue_green",
            "deployed_at": now()
        })

        return {"status": "switched", "from": old_version, "to": new_version}

    def rollback(self, model_name, operator_id, reason=None):
        """回退到上一版本"""
        # 获取当前版本和上一版本
        current = self._get_production_version(model_name)
        previous = self.db.query_one(
            "SELECT * FROM model_versions "
            "WHERE model_name = %s AND status = 'retired' "
            "ORDER BY retired_at DESC LIMIT 1", model_name)

        if not previous:
            return {"status": "no_previous_version"}

        # 切换回上一版本
        self.redis.set(f"model_route:{model_name}", json.dumps({
            "production": previous["version"]
        }))

        self.db.update("model_versions",
            {"status": "production"}, {"version_id": previous["version_id"]})
        self.db.update("model_versions",
            {"status": "rolled_back", "rollback_reason": reason},
            {"version_id": current["version_id"]})

        self.db.insert("model_deployment_history", {
            "id": str(uuid4()), "model_name": model_name,
            "from_version": current["version"], "to_version": previous["version"],
            "operator": operator_id, "strategy": "rollback",
            "reason": reason, "deployed_at": now()
        })

        return {"status": "rolled_back", "from": current["version"],
                "to": previous["version"]}

    def _compare_versions(self, current_metrics, new_metrics):
        """对比版本指标"""
        regressions = []
        # 准确率不能下降 > 1%
        if new_metrics.get("accuracy", 1) < current_metrics.get("accuracy", 1) - 0.01:
            regressions.append({
                "metric": "accuracy",
                "current": current_metrics["accuracy"],
                "new": new_metrics["accuracy"],
                "delta": round(new_metrics["accuracy"] - current_metrics["accuracy"], 4)
            })
        # 延迟不能增加 > 20%
        if new_metrics.get("latency_p95_ms", 0) > current_metrics.get("latency_p95_ms", 0) * 1.2:
            regressions.append({
                "metric": "latency_p95_ms",
                "current": current_metrics["latency_p95_ms"],
                "new": new_metrics["latency_p95_ms"]
            })

        return {"regression": regressions if regressions else None}
```

## 异常场景补充

### 场景：模型灰度后指标漂移

```
触发：模型灰度 5% 时指标正常 → 扩大到 50% 时准确率下降 → 负载相关退化
检测：
  1. 灰度流量扩大后指标下降 → 负载相关
  2. 不同流量比例下指标不一致 → 非线性退化
处理：
  1. 回退到灰度前版本
  2. 分析负载相关退化原因
  3. 修复后重新灰度
预防：渐进扩大 + 负载测试 + 指标监控
```

### 场景：蓝绿切换失败

```
触发：蓝绿切换执行中 → 新版本启动失败 → 旧版本已被标记 retired → 服务中断
检测：
  1. 新版本健康检查失败 → 切换失败
  2. 服务无可用版本 → 中断
处理：
  1. 切换失败 → 自动回退到旧版本
  2. 旧版本标记 retired 延迟到切换成功
  3. 切换过程保持旧版本可用
预防：自动回退 + 延迟标记 + 保持可用
```

## AI 推理模型 A/B 测试完整实现

```python
import hashlib
import time
import math
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from typing import Optional, Dict, List, Tuple
from enum import Enum
from scipy import stats as scipy_stats


class TestStatus(Enum):
    CREATED = "created"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"


class ModelEndpoint(Enum):
    MODEL_A = "model_a"
    MODEL_B = "model_b"


@dataclass
class ABTestConfig:
    """A/B 测试配置"""
    test_id: str
    model_a_version: str
    model_b_version: str
    traffic_split: float  # model A 的流量比例, 0.0~1.0
    min_samples: int = 1000
    significance_level: float = 0.05
    max_duration_hours: int = 72
    created_at: datetime = field(default_factory=datetime.utcnow)


@dataclass
class InferenceResult:
    """单次推理结果"""
    request_id: str
    model_endpoint: ModelEndpoint
    latency_ms: float
    is_correct: bool
    timestamp: datetime = field(default_factory=datetime.utcnow)
    user_id: Optional[str] = None


@dataclass
class ModelMetrics:
    """模型指标累积器"""
    model_endpoint: ModelEndpoint
    latencies: List[float] = field(default_factory=list)
    correctness: List[bool] = field(default_factory=list)
    sample_count: int = 0

    def add_result(self, result: InferenceResult):
        self.latencies.append(result.latency_ms)
        self.correctness.append(result.is_correct)
        self.sample_count += 1

    def avg_latency(self) -> float:
        if not self.latencies:
            return 0.0
        return sum(self.latencies) / len(self.latencies)

    def accuracy(self) -> float:
        if not self.correctness:
            return 0.0
        return sum(self.correctness) / len(self.correctness)


@dataclass
class ABTestResult:
    """A/B 测试结果"""
    test_id: str
    status: TestStatus
    winner: Optional[ModelEndpoint] = None
    latency_p_value: Optional[float] = None
    accuracy_p_value: Optional[float] = None
    latency_significant: bool = False
    accuracy_significant: bool = False
    model_a_avg_latency: float = 0.0
    model_b_avg_latency: float = 0.0
    model_a_accuracy: float = 0.0
    model_b_accuracy: float = 0.0
    reason: str = ""


class ModelABTestService:
    """AI 推理模型 A/B 测试服务
    
    在两个模型版本之间进行 A/B 测试, 基于用户哈希分流,
    收集延迟和准确率指标, 并通过统计显著性检验确定获胜模型。
    """

    def __init__(self):
        self._tests: Dict[str, ABTestConfig] = {}
        self._metrics: Dict[str, Dict[ModelEndpoint, ModelMetrics]] = {}
        self._results: Dict[str, ABTestResult] = {}
        self._model_health: Dict[str, Dict[ModelEndpoint, bool]] = {}

    def create_model_test(
        self,
        test_id: str,
        model_a_version: str,
        model_b_version: str,
        traffic_split: float = 0.5,
        min_samples: int = 1000,
        significance_level: float = 0.05,
        max_duration_hours: int = 72,
    ) -> ABTestConfig:
        """创建模型 A/B 测试
        
        Args:
            test_id: 测试唯一标识
            model_a_version: 模型 A 版本号
            model_b_version: 模型 B 版本号
            traffic_split: 模型 A 的流量比例 (0.0~1.0)
            min_samples: 最小样本量
            significance_level: 统计显著性水平
            max_duration_hours: 最大测试时长 (小时)
            
        Returns:
            ABTestConfig: 测试配置
            
        Raises:
            ValueError: traffic_split 不在 [0.1, 0.9] 范围内
            KeyError: test_id 已存在
        """
        if not (0.1 <= traffic_split <= 0.9):
            raise ValueError(
                f"traffic_split must be between 0.1 and 0.9, got {traffic_split}"
            )
        if test_id in self._tests:
            raise KeyError(f"Test {test_id} already exists")

        config = ABTestConfig(
            test_id=test_id,
            model_a_version=model_a_version,
            model_b_version=model_b_version,
            traffic_split=traffic_split,
            min_samples=min_samples,
            significance_level=significance_level,
            max_duration_hours=max_duration_hours,
        )
        self._tests[test_id] = config
        self._metrics[test_id] = {
            ModelEndpoint.MODEL_A: ModelMetrics(model_endpoint=ModelEndpoint.MODEL_A),
            ModelEndpoint.MODEL_B: ModelMetrics(model_endpoint=ModelEndpoint.MODEL_B),
        }
        self._model_health[test_id] = {
            ModelEndpoint.MODEL_A: True,
            ModelEndpoint.MODEL_B: True,
        }
        return config

    def _hash_user_to_bucket(self, user_id: str, traffic_split: float) -> ModelEndpoint:
        """基于用户 ID 哈希将请求路由到模型 A 或 B
        
        使用 MD5 哈希保证同一用户始终路由到同一模型,
        避免用户看到不一致的模型行为。
        """
        hash_hex = hashlib.md5(user_id.encode("utf-8")).hexdigest()
        hash_int = int(hash_hex[:8], 16)  # 取前 8 位十六进制
        bucket_value = hash_int / 0xFFFFFFFF  # 归一化到 [0, 1)
        if bucket_value < traffic_split:
            return ModelEndpoint.MODEL_A
        return ModelEndpoint.MODEL_B

    def route_request(
        self, test_id: str, user_id: str, request_id: str
    ) -> Tuple[ModelEndpoint, str]:
        """路由推理请求到对应模型
        
        Args:
            test_id: 测试 ID
            user_id: 用户 ID, 用于一致性哈希分流
            request_id: 请求 ID
            
        Returns:
            (ModelEndpoint, model_version): 目标模型端点和版本号
            
        Raises:
            KeyError: test_id 不存在
            RuntimeError: 测试已结束或目标模型不可用
        """
        if test_id not in self._tests:
            raise KeyError(f"Test {test_id} does not exist")

        config = self._tests[test_id]
        result = self._results.get(test_id)
        if result and result.status == TestStatus.COMPLETED:
            raise RuntimeError(f"Test {test_id} has already completed")

        # 检查测试是否超时
        elapsed = datetime.utcnow() - config.created_at
        if elapsed > timedelta(hours=config.max_duration_hours):
            self.evaluate_winner(test_id)
            raise RuntimeError(f"Test {test_id} has exceeded max duration")

        target = self._hash_user_to_bucket(user_id, config.traffic_split)

        # 如果目标模型不健康, 切换到另一个模型
        if not self._model_health[test_id][target]:
            fallback = (
                ModelEndpoint.MODEL_B if target == ModelEndpoint.MODEL_A
                else ModelEndpoint.MODEL_A
            )
            if not self._model_health[test_id][fallback]:
                raise RuntimeError(
                    f"Both models are unhealthy for test {test_id}"
                )
            target = fallback

        version = (
            config.model_a_version if target == ModelEndpoint.MODEL_A
            else config.model_b_version
        )
        return target, version

    def collect_metrics(
        self,
        test_id: str,
        request_id: str,
        model_endpoint: ModelEndpoint,
        latency_ms: float,
        is_correct: bool,
        user_id: Optional[str] = None,
    ) -> ModelMetrics:
        """收集单次推理指标
        
        Args:
            test_id: 测试 ID
            request_id: 请求 ID
            model_endpoint: 推理使用的模型端点
            latency_ms: 推理延迟 (毫秒)
            is_correct: 推理结果是否正确
            user_id: 用户 ID
            
        Returns:
            更新后的 ModelMetrics
            
        Raises:
            KeyError: test_id 不存在
        """
        if test_id not in self._metrics:
            raise KeyError(f"Test {test_id} does not exist")

        result = InferenceResult(
            request_id=request_id,
            model_endpoint=model_endpoint,
            latency_ms=latency_ms,
            is_correct=is_correct,
            user_id=user_id,
        )
        metrics = self._metrics[test_id][model_endpoint]
        metrics.add_result(result)

        # 自动检查是否达到最小样本量
        total_samples = sum(
            m.sample_count for m in self._metrics[test_id].values()
        )
        config = self._tests[test_id]
        if total_samples >= config.min_samples * 2:
            self.evaluate_winner(test_id)

        return metrics

    def evaluate_winner(self, test_id: str) -> ABTestResult:
        """评估 A/B 测试获胜者
        
        使用 Welch t-test 比较两组延迟, 使用卡方检验比较两组准确率。
        只有当统计显著性 p 值小于预设阈值时才确定获胜者。
        
        Args:
            test_id: 测试 ID
            
        Returns:
            ABTestResult: 包含统计检验结果的测试结论
            
        Raises:
            KeyError: test_id 不存在
            ValueError: 样本量不足以进行统计检验
        """
        if test_id not in self._tests:
            raise KeyError(f"Test {test_id} does not exist")

        config = self._tests[test_id]
        metrics_a = self._metrics[test_id][ModelEndpoint.MODEL_A]
        metrics_b = self._metrics[test_id][ModelEndpoint.MODEL_B]

        # 样本量检查
        if metrics_a.sample_count < 30 or metrics_b.sample_count < 30:
            result = ABTestResult(
                test_id=test_id,
                status=TestStatus.FAILED,
                reason="Insufficient samples for statistical test",
            )
            self._results[test_id] = result
            return result

        # --- 延迟比较: Welch t-test ---
        latency_p_value = 1.0
        latency_significant = False
        try:
            t_stat, latency_p_value = scipy_stats.ttest_ind(
                metrics_a.latencies, metrics_b.latencies, equal_var=False
            )
            latency_significant = latency_p_value < config.significance_level
        except Exception:
            latency_p_value = 1.0

        # --- 准确率比较: 卡方检验 ---
        accuracy_p_value = 1.0
        accuracy_significant = False
        try:
            a_correct = sum(metrics_a.correctness)
            a_incorrect = metrics_a.sample_count - a_correct
            b_correct = sum(metrics_b.correctness)
            b_incorrect = metrics_b.sample_count - b_correct
            contingency = [[a_correct, a_incorrect], [b_correct, b_incorrect]]
            chi2, accuracy_p_value, _, _ = scipy_stats.chi2_contingency(
                contingency, correction=False
            )
            accuracy_significant = accuracy_p_value < config.significance_level
        except Exception:
            accuracy_p_value = 1.0

        # 确定获胜者: 延迟更低且准确率更高的模型获胜
        # 如果只有一项显著, 则只看该项; 都显著则综合判断
        winner = None
        reason_parts = []

        a_latency = metrics_a.avg_latency()
        b_latency = metrics_b.avg_latency()
        a_accuracy = metrics_a.accuracy()
        b_accuracy = metrics_b.accuracy()

        if latency_significant:
            if a_latency < b_latency:
                reason_parts.append(
                    f"Model A faster ({a_latency:.1f}ms vs {b_latency:.1f}ms, p={latency_p_value:.4f})"
                )
            else:
                reason_parts.append(
                    f"Model B faster ({b_latency:.1f}ms vs {a_latency:.1f}ms, p={latency_p_value:.4f})"
                )

        if accuracy_significant:
            if a_accuracy > b_accuracy:
                reason_parts.append(
                    f"Model A more accurate ({a_accuracy:.2%} vs {b_accuracy:.2%}, p={accuracy_p_value:.4f})"
                )
            else:
                reason_parts.append(
                    f"Model B more accurate ({b_accuracy:.2%} vs {a_accuracy:.2%}, p={accuracy_p_value:.4f})"
                )

        # 综合判定: 准确率优先, 延迟次之
        if latency_significant or accuracy_significant:
            a_score = 0
            b_score = 0
            if accuracy_significant:
                if a_accuracy > b_accuracy:
                    a_score += 2
                else:
                    b_score += 2
            if latency_significant:
                if a_latency < b_latency:
                    a_score += 1
                else:
                    b_score += 1
            if a_score > b_score:
                winner = ModelEndpoint.MODEL_A
            elif b_score > a_score:
                winner = ModelEndpoint.MODEL_B
            # 平局时无获胜者

        result = ABTestResult(
            test_id=test_id,
            status=TestStatus.COMPLETED,
            winner=winner,
            latency_p_value=latency_p_value,
            accuracy_p_value=accuracy_p_value,
            latency_significant=latency_significant,
            accuracy_significant=accuracy_significant,
            model_a_avg_latency=a_latency,
            model_b_avg_latency=b_latency,
            model_a_accuracy=a_accuracy,
            model_b_accuracy=b_accuracy,
            reason="; ".join(reason_parts) if reason_parts else "No significant difference found",
        )
        self._results[test_id] = result
        return result

    def mark_model_unhealthy(self, test_id: str, model: ModelEndpoint):
        """标记模型为不健康状态, 后续请求将路由到另一个模型"""
        if test_id in self._model_health:
            self._model_health[test_id][model] = False

    def get_test_status(self, test_id: str) -> Optional[ABTestResult]:
        """获取测试结果"""
        return self._results.get(test_id)

    def get_metrics_summary(self, test_id: str) -> Dict[str, dict]:
        """获取指标摘要"""
        if test_id not in self._metrics:
            return {}
        return {
            ep.value: {
                "sample_count": m.sample_count,
                "avg_latency_ms": m.avg_latency(),
                "accuracy": m.accuracy(),
            }
            for ep, m in self._metrics[test_id].items()
        }
```

## 异常场景补充

### 场景：A/B 测试流量分配不均
```
trigger: 哈希函数在某些 user_id 分布下产生偏斜, 导致 80% 请求路由到模型 A, 而配置的 traffic_split 为 50%

detection:
  1. 在 collect_metrics 中比较两个模型的实际样本量比率与配置的 traffic_split
  2. 计算实际比率 |actual_ratio - configured_split|, 若偏差超过阈值 (如 0.1) 则告警
  3. 定期执行流量均衡检查任务, 每 5 分钟校验一次实际分流比率

handling:
  1. 记录告警日志, 包含实际分流比率和预期分流比率
  2. 若偏差在 10%~20%, 在结果评估时对样本量加权校正, 避免统计偏差
  3. 若偏差超过 20%, 暂停测试, 切换为一致性哈希加盐策略 (在 user_id 后追加 salt 重新哈希)
  4. 通知测试负责人, 等待人工确认后恢复测试

prevention:
  1. 创建测试时对流量分流算法进行预校验: 使用模拟 user_id 集合验证分流均匀性
  2. 采用双重哈希策略: 使用两个不同哈希函数取异或, 减少分布偏斜概率
  3. 设置最小样本量门槛: 两个模型各自至少积累 min_samples 后才进行统计检验
  4. 监控分流偏斜度并设置自动熔断: 偏差超过阈值时自动暂停测试
```

### 场景：测试期间模型 A 崩溃
```
trigger: 模型 A 所在服务节点发生 OOM 或 GPU 故障, 导致所有路由到模型 A 的推理请求超时失败

detection:
  1. 在 collect_metrics 中检测模型 A 的连续失败率: 若最近 100 个请求中失败率 > 50%, 判定为模型异常
  2. 推理请求超时率监控: 若模型 A 的 P99 延迟超过正常值的 5 倍, 触发异常告警
  3. 健康检查探针: 定期对模型 A 发送轻量级推理请求 (如空输入), 连续 3 次失败则标记不健康

handling:
  1. 立即调用 mark_model_unhealthy 将模型 A 标记为不健康
  2. route_request 中检测到模型 A 不健康后, 将所有请求自动路由到模型 B (降级模式)
  3. 保留模型 A 已收集的指标数据, 但标记为 "partial_data"
  4. 若模型 A 在 30 分钟内恢复, 重新标记为健康, 恢复正常分流
  5. 若模型 A 超过 2 小时未恢复, 自动结束测试, 基于已收集数据给出有限结论, 并在结果中标注 "Model A experienced downtime"

prevention:
  1. 创建测试前对两个模型进行健康检查, 确认均处于可用状态
  2. 为每个模型配置自动重试和熔断机制: 连续失败 5 次后熔断 30 秒
  3. 设计模型故障时的自动降级策略: 单模型故障时 100% 流量切换到健康模型
  4. 设置最大可容忍停机时间: 若任一模型停机超过阈值, 自动终止测试以避免数据偏差
  5. 在测试开始前与模型部署团队确认容量规划和故障恢复 SLA
```

## AI 推理模型版本管理完整实现

```python
class ModelVersionManagementService:
    """模型版本管理：版本注册 → 灰度部署 → 性能对比 → 回滚"""

    VERSION_STATUSES = {
        "development": "开发中",
        "staging": "预发布",
        "canary": "灰度中",
        "production": "生产",
        "deprecated": "已下线",
    }

    def register_model_version(self, model_name, version, artifact_path,
                               metrics=None, config=None):
        """注册模型版本"""
        version_id = str(uuid4())

        # 1. 验证模型产物
        if not self.object_storage.exists(artifact_path):
            raise ValueError(f"模型产物不存在: {artifact_path}")

        # 2. 检查版本号唯一性
        existing = self.db.query_one(
            "SELECT * FROM model_versions "
            "WHERE model_name = %s AND version = %s",
            model_name, version)
        if existing:
            return {"status": "version_exists", "version_id": existing["version_id"]}

        # 3. 注册
        self.db.insert("model_versions", {
            "version_id": version_id,
            "model_name": model_name,
            "version": version,
            "artifact_path": artifact_path,
            "config": json.dumps(config or {}),
            "baseline_metrics": json.dumps(metrics or {}),
            "status": "development",
            "created_at": now()
        })

        return {"version_id": version_id, "model_name": model_name,
                "version": version, "status": "development"}

    def promote_to_canary(self, version_id, canary_pct=5):
        """提升到灰度"""
        version = self.db.get_model_version(version_id)

        if version["status"] != "staging":
            return {"status": "invalid_transition",
                    "current": version["status"], "required": "staging"}

        # 1. 检查是否已有灰度版本
        current_canary = self.db.query_one(
            "SELECT * FROM model_versions "
            "WHERE model_name = %s AND status = 'canary'",
            version["model_name"])

        if current_canary:
            return {"status": "canary_exists",
                    "current_canary_version": current_canary["version"]}

        # 2. 获取当前生产版本
        production = self.db.query_one(
            "SELECT * FROM model_versions "
            "WHERE model_name = %s AND status = 'production'",
            version["model_name"])

        # 3. 创建灰度规则
        self.db.insert("model_canary_rules", {
            "rule_id": str(uuid4()),
            "model_name": version["model_name"],
            "canary_version_id": version_id,
            "production_version_id": production["version_id"] if production else None,
            "canary_pct": canary_pct,
            "status": "active",
            "started_at": now()
        })

        # 4. 更新状态
        self.db.update("model_versions",
            {"status": "canary", "canary_started_at": now()},
            {"version_id": version_id})

        return {"version_id": version_id, "status": "canary",
                "canary_pct": canary_pct}

    def get_canary_metrics(self, model_name):
        """获取灰度指标对比"""
        rule = self.db.query_one(
            "SELECT * FROM model_canary_rules "
            "WHERE model_name = %s AND status = 'active'",
            model_name)

        if not rule:
            return {"status": "no_active_canary"}

        # 获取两个版本的指标
        canary_metrics = self._collect_version_metrics(rule["canary_version_id"])
        production_metrics = self._collect_version_metrics(rule["production_version_id"])

        # 对比
        comparison = {}
        for metric_name in set(list(canary_metrics.keys()) + list(production_metrics.keys())):
            canary_val = canary_metrics.get(metric_name, 0)
            prod_val = production_metrics.get(metric_name, 0)
            diff_pct = ((canary_val - prod_val) / max(prod_val, 0.001)) * 100

            comparison[metric_name] = {
                "canary": round(canary_val, 4),
                "production": round(prod_val, 4),
                "diff_pct": round(diff_pct, 2),
                "improved": diff_pct > 0 if "accuracy" in metric_name or "f1" in metric_name
                           else diff_pct < 0  # latency 越低越好
            }

        return {
            "model_name": model_name,
            "canary_version": rule["canary_version_id"],
            "production_version": rule["production_version_id"],
            "canary_pct": rule["canary_pct"],
            "comparison": comparison
        }

    def promote_to_production(self, version_id):
        """灰度 → 全量生产"""
        version = self.db.get_model_version(version_id)

        if version["status"] != "canary":
            return {"status": "invalid_transition"}

        # 1. 检查灰度指标
        metrics = self.get_canary_metrics(version["model_name"])

        # 2. 检查关键指标是否恶化
        for metric_name, comparison in metrics.get("comparison", {}).items():
            if not comparison.get("improved", True) and abs(comparison["diff_pct"]) > 5:
                return {"status": "metrics_regression",
                        "metric": metric_name,
                        "diff_pct": comparison["diff_pct"]}

        # 3. 旧版本下线
        old_prod = self.db.query_one(
            "SELECT * FROM model_versions "
            "WHERE model_name = %s AND status = 'production'",
            version["model_name"])

        if old_prod:
            self.db.update("model_versions",
                {"status": "deprecated", "deprecated_at": now()},
                {"version_id": old_prod["version_id"]})

        # 4. 新版本上线
        self.db.update("model_versions",
            {"status": "production", "promoted_at": now()},
            {"version_id": version_id})

        # 5. 关闭灰度规则
        self.db.update("model_canary_rules",
            {"status": "completed", "completed_at": now()},
            {"canary_version_id": version_id, "status": "active"})

        return {"version_id": version_id, "status": "production"}

    def rollback_model(self, model_name, target_version=None):
        """回滚模型"""
        # 1. 获取当前生产版本
        current = self.db.query_one(
            "SELECT * FROM model_versions "
            "WHERE model_name = %s AND status = 'production'",
            model_name)

        if not current:
            return {"status": "no_production_version"}

        # 2. 确定回滚目标
        if target_version:
            target = self.db.query_one(
                "SELECT * FROM model_versions "
                "WHERE model_name = %s AND version = %s AND status = 'deprecated'",
                model_name, target_version)
        else:
            # 回滚到上一个生产版本
            target = self.db.query_one(
                "SELECT * FROM model_versions "
                "WHERE model_name = %s AND status = 'deprecated' "
                "ORDER BY deprecated_at DESC LIMIT 1",
                model_name)

        if not target:
            return {"status": "no_rollback_target"}

        # 3. 执行回滚
        self.db.update("model_versions",
            {"status": "deprecated"},
            {"version_id": current["version_id"]})

        self.db.update("model_versions",
            {"status": "production", "promoted_at": now()},
            {"version_id": target["version_id"]})

        # 4. 记录回滚
        self.db.insert("model_rollbacks", {
            "rollback_id": str(uuid4()),
            "model_name": model_name,
            "from_version_id": current["version_id"],
            "to_version_id": target["version_id"],
            "rolled_back_at": now()
        })

        return {"model_name": model_name,
                "from_version": current["version"],
                "to_version": target["version"],
                "status": "rolled_back"}

    def _collect_version_metrics(self, version_id):
        """收集版本指标"""
        metrics = self.db.query(
            "SELECT metric_name, AVG(value) as avg_value "
            "FROM model_inference_metrics "
            "WHERE version_id = %s AND timestamp > NOW() - INTERVAL 1 HOUR "
            "GROUP BY metric_name", version_id)

        return {m["metric_name"]: float(m["avg_value"]) for m in metrics}
```

## 异常场景补充

### 场景：模型灰度期间指标波动

```
触发：灰度 5% 流量 → 指标正常 → 扩大到 50% → 指标突然恶化 → 灰度比例影响结果
检测：
  1. 灰度扩大后指标恶化 → 流量比例效应
  2. 小流量指标与大流量指标不一致 → 采样偏差
处理：
  1. 灰度逐步扩大（5% → 10% → 25% → 50% → 100%）
  2. 每个阶段停留足够时间收集指标
  3. 指标恶化自动回退到上一阶段
预防：逐步扩大 + 充分观察 + 自动回退
```

### 场景：模型版本 artifact 被误删

```
触发：清理旧存储时误删生产模型 artifact → 推理服务报错 → 模型不可用
检测：
  1. 推理服务加载模型失败 → artifact 缺失
  2. 模型文件不存在 → 被删除
处理：
  1. 从备份恢复 artifact
  2. 回滚到可用的旧版本
  3. 推理服务自动降级到上一版本
预防：artifact 只读保护 + 自动备份 + 服务降级
```
