# C10: 微服务下的可观测性体系

## 业务场景

某互联网公司的微服务架构中，20+个微服务通过 gRPC/HTTP 相互调用，50+个实例部署在 K8s 集群中。最近 3 个月发生了 5 次 P0 故障，平均故障定位时间 2 小时。CFO 要求：故障定位时间必须缩短到 10 分钟以内。

**最近一次 P0 故障复盘：**
- 14:00 用户反馈"下单失败"
- 14:05 运维看到"订单服务 5xx 率上升"告警
- 14:10 开始排查：登录服务器看日志，发现订单服务调支付服务超时
- 14:20 查支付服务日志，发现支付服务调风控服务超时
- 14:30 查风控服务日志，发现风控服务调外部合规 API 超时
- 14:35 确认根因：外部合规 API 提供商故障
- 14:40 配置降级：跳过合规检查 → 服务恢复

**问题：40 分钟才定位到根因。** 如果有完整的可观测性体系，这个过程应该在 5 分钟内完成。

**已知数据：**
- 20+ 微服务，50+ 实例
- 日均 API 请求 1 亿次
- 平均单次请求跨越 5-6 个微服务
- 日志量：约 500GB/天
- 指标采集间隔：15 秒
- 当前工具：ELK（日志）、Prometheus（指标）、无链路追踪

## 核心挑战

### 挑战 1：日志量大但信息密度低

500GB/天的日志，但 95% 是 INFO 级别的正常日志。真正有用的 ERROR 日志淹没在噪音中。更关键的是，各服务的日志是割裂的——同一个请求的日志分散在 6 个服务的日志文件中，无法串联。

### 挑战 2：指标告警无法定位根因

"订单服务 5xx 率上升"只告诉你"有问题"，不告诉你"为什么"。可能的原因有：
- 订单服务自身 bug
- 支付服务超时
- 数据库慢查询
- K8s 节点 CPU 打满
- 网络抖动

仅凭指标无法区分，必须关联链路追踪和日志。

### 挑战 3：缺乏分布式链路追踪

一个用户请求经过 6 个服务，每个服务有自己的 request ID。当请求失败时，运维需要在 6 个服务的日志中手动搜索相关请求——这就像在 6 本不同的书中找同一段故事。

### 挑战 4：告警风暴

一次支付服务故障，可能触发：
- 订单服务 5xx 率上升
- 支付服务延迟上升
- 支付服务错误率上升
- 风控服务超时率上升
- 用户服务回调失败率上升

5+ 条告警同时触发，但它们是同一个根因。如何降噪？

## 设计约束

- 日志保留 30 天，指标 1 年
- 全链路追踪开销 < 5%（延迟和 CPU）
- 告警响应时间 < 10 分钟
- 不更换现有 ELK 和 Prometheus

## 请先独立思考（限时 35 分钟）

1. 日志、指标、链路三个维度在故障排查中各自的角色是什么？如何协同？
2. 设计一个 TraceID 跨服务传递的方案：HTTP/gRPC/Kafka 消息中如何传递？
3. 告警降噪方案：如何将同一根因的多条告警聚合？
4. 从"订单 5xx 率上升"到"外部合规 API 故障"，设计一个 5 分钟内的排查路径。

---

## 设计解析

### 三支柱模型

```
指标（Metrics）  → "出了什么问题" → 发现异常
链路（Traces）   → "问题在哪里" → 定位服务
日志（Logs）     → "具体什么原因" → 定位代码行
```

三支柱的协同关系：

| 排查阶段 | 使用什么 | 典型操作 |
|---------|---------|---------|
| 发现异常 | 指标 | Grafana 仪表盘显示 5xx 率飙升 |
| 定位服务 | 链路 | 根据 Trace ID 找到耗时最长的 Span |
| 定位根因 | 日志 | 根据 Trace ID + Span ID 搜索日志，找到 ERROR 行 |
| 确认修复 | 指标 | 观察 5xx 率回落到正常水平 |

### 第一支柱：链路追踪

**为什么链路追踪是关键？** 因为它是串联三大支柱的线索——TraceID 同时出现在指标标签、链路数据和日志中。

**TraceID 跨服务传递方案：**

```
用户请求 → API Gateway（生成 TraceID: abc123）
           → HTTP Header: X-Trace-Id: abc123, X-Span-Id: span1
           → 订单服务
             → gRPC Metadata: trace-id=abc123, span-id=span2, parent-span-id=span1
             → 支付服务
               → Kafka Message Header: trace-id=abc123
               → 风控服务
```

**SDK 自动注入（以 Java 为例）：**

```java
// 使用 OpenTelemetry SDK 自动注入 TraceID
// 无需修改业务代码

// HTTP 调用：自动在 Header 中注入 TraceID
Tracer tracer = GlobalOpenTelemetry.getTracer("order-service");
Span span = tracer.spanBuilder("processOrder").startSpan();

try (Scope scope = span.makeCurrent()) {
    // 下游 HTTP 调用会自动携带 trace-id Header
    paymentService.charge(order);
} finally {
    span.end();
}

// gRPC 调用：自动在 Metadata 中注入
// Kafka 消息：自动在 Message Header 中注入
```

**链路数据的存储：Jaeger**

```yaml
# Jaeger 部署配置
allInOne:
  enabled: true
storage:
  type: elasticsearch  # 与 ELK 共享 ES 集群
  elasticsearch:
    indexPrefix: jaeger
```

**一条完整的链路数据：**

```json
{
  "traceId": "abc123",
  "spans": [
    {"spanId": "s1", "operation": "APIGateway.handle", "duration": "150ms", "status": "OK"},
    {"spanId": "s2", "parentSpanId": "s1", "operation": "OrderService.create", "duration": "140ms", "status": "OK"},
    {"spanId": "s3", "parentSpanId": "s2", "operation": "PaymentService.charge", "duration": "120ms", "status": "ERROR"},
    {"spanId": "s4", "parentSpanId": "s3", "operation": "RiskControlService.check", "duration": "11000ms", "status": "TIMEOUT"},
    {"spanId": "s5", "parentSpanId": "s4", "operation": "ComplianceAPI.verify", "duration": "10500ms", "status": "TIMEOUT"}
  ]
}
```

**5 分钟排查路径的演示：**

```
1. Grafana 告警：订单服务 5xx 率 > 5%
2. 点击告警 → 跳转到 Grafana 面板 → 看到最近的失败请求 Trace ID
3. 输入 Trace ID → Jaeger 展示完整链路：
   APIGateway(5ms) → OrderService(10ms) → PaymentService(10ms) 
   → RiskControlService(11s!) → ComplianceAPI(10.5s!)
   
4. 一眼看出：RiskControlService → ComplianceAPI 耗时 10.5 秒 → 外部 API 超时
5. 搜索日志：trace_id=abc123 → 找到 RiskControlService 的 ERROR 日志
   "Compliance API timeout after 10s, url=https://compliance.example.com/verify"

总耗时：< 3 分钟
```

### 第二支柱：结构化日志 + TraceID 关联

**核心原则：每条日志必须包含 TraceID。**

```python
import logging
import json

class StructuredLogger:
    def __init__(self, service_name):
        self.service = service_name
        self.logger = logging.getLogger(service_name)

    def info(self, msg, **kwargs):
        self._log("INFO", msg, **kwargs)

    def error(self, msg, **kwargs):
        self._log("ERROR", msg, **kwargs)

    def _log(self, level, msg, **kwargs):
        # 从当前上下文获取 TraceID（由 OpenTelemetry SDK 管理）
        from opentelemetry import context
        trace_id = context.get_current().get("trace_id", "")
        span_id = context.get_current().get("span_id", "")

        log_entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "level": level,
            "service": self.service,
            "trace_id": trace_id,    # 关键：关联链路
            "span_id": span_id,
            "message": msg,
            **kwargs
        }

        self.logger.info(json.dumps(log_entry))
```

**日志示例：**

```json
{"timestamp":"2024-05-07T14:05:23Z","level":"ERROR","service":"risk-control","trace_id":"abc123","span_id":"s4","message":"Compliance API timeout","url":"https://compliance.example.com/verify","timeout_ms":10000,"retry_count":3}
```

**在 ELK 中按 TraceID 搜索：**

```
# Kibana 查询
trace_id: "abc123"

# 结果：该请求在所有 6 个服务中的日志，按时间排序
1. [APIGateway] Request received, path=/api/orders
2. [OrderService] Creating order ORD-001
3. [PaymentService] Charging payment for ORD-001
4. [RiskControlService] Checking risk for ORD-001
5. [RiskControlService] ERROR: Compliance API timeout after 10s
6. [PaymentService] Payment failed: risk check timeout
7. [OrderService] Order creation failed: payment error
```

### 第三支柱：指标体系

**RED 指标（每个服务必须暴露）：**

| 指标 | Prometheus 表达式 | 含义 |
|------|------------------|------|
| Rate | `rate(http_requests_total[5m])` | 请求速率 |
| Errors | `rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])` | 错误率 |
| Duration | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` | P99 延迟 |

**USE 指标（每个实例必须暴露）：**

| 指标 | 含义 |
|------|------|
| CPU 使用率 | 实例负载 |
| 内存使用率 | OOM 风险 |
| 连接池使用率 | 资源瓶颈 |
| GC 停顿时间 | Java 服务特有 |

**Grafana 仪表盘设计：**

```
┌──────────────────────────────────────────────────┐
│ 全局概览                                        │
│   [总QPS] [总错误率] [总P99延迟] [活跃告警数]    │
├──────────────────────────────────────────────────┤
│ 服务健康矩阵                                    │
│   订单服务  ✓ 5xx:0.1%  P99:120ms              │
│   支付服务  ✗ 5xx:8.2%  P99:3500ms  ← 红色高亮 │
│   风控服务  ✗ 5xx:12%   P99:11000ms ← 红色高亮 │
│   库存服务  ✓ 5xx:0.0%  P99:50ms               │
├──────────────────────────────────────────────────┤
│ 依赖关系拓扑                                    │
│   订单 → 支付 [8.2% err] → 风控 [12% err]      │
│                      → 合规API [timeout]         │
└──────────────────────────────────────────────────┘
```

### 告警设计：分级 + 降噪 + 根因推断

**告警分级：**

```yaml
groups:
  - name: service-health
    rules:
      # P0：直接影响用户
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          / sum(rate(http_requests_total[5m])) by (service) > 0.05
        for: 2m
        labels:
          severity: critical
          page: true        # 触发电话告警
        annotations:
          summary: "服务 {{ $labels.service }} 错误率超过 5%"

      # P1：可能影响用户
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, 
            sum(rate(http_request_duration_seconds_bucket[5m])) by (service, le)
          ) > 3
        for: 5m
        labels:
          severity: warning
          slack: true        # 触发 Slack 通知
        annotations:
          summary: "服务 {{ $labels.service }} P99 延迟超过 3s"
```

**告警降噪：根因推断**

```python
class AlertCorrelator:
    """将同一根因的多条告警聚合"""

    def correlate(self, new_alert):
        # 查找最近 5 分钟内的活跃告警
        active_alerts = self.get_active_alerts(window_minutes=5)

        # 基于服务依赖图判断因果关系
        # 如果 A 依赖 B，且 B 有告警，则 A 的告警可能是 B 导致的
        for alert in active_alerts:
            if self.is_cause(alert, new_alert):
                # 合并告警：B 是根因，A 是影响
                self.merge_alerts(cause=alert, effect=new_alert)
                return

        # 新的独立告警
        self.create_alert(new_alert)

    def is_cause(self, alert_a, alert_b):
        """判断 alert_a 是否是 alert_b 的根因"""
        # 查服务依赖图
        deps = self.dependency_graph.get_dependencies(alert_b.service)
        return alert_a.service in deps

# 示例：
# 告警1: 风控服务 5xx 率 12%
# 告警2: 支付服务 5xx 率 8.2%（依赖风控服务）
# 告警3: 订单服务 5xx 率 5.1%（依赖支付服务）
#
# 聚合后：
# 根因告警: 风控服务 5xx 率 12%
# 影响告警: 支付服务、订单服务（被合并，不再单独告警）
# 通知内容: "风控服务异常导致支付服务和订单服务受影响"
```

### 完整的故障排查 SOP

```
Step 1: 告警触发 → Grafana 全局概览（5秒）
  → 哪个服务出错？错误率多高？

Step 2: 点击服务 → 跳转到 Jaeger 链路（10秒）
  → 哪个 Span 耗时最长/报错？
  → 根因在哪个服务？

Step 3: 复制 TraceID → Kibana 搜索日志（10秒）
  → 具体的 ERROR 信息
  → 错误原因（超时？拒绝？数据错误？）

Step 4: 确认修复方案 → 执行（1-5分钟）
  → 重启？降级？扩容？配置回滚？

Step 5: 观察指标恢复（1-2分钟）
  → 5xx 率回落？延迟恢复？

总排查时间：< 5 分钟
```

### 性能开销

| 维度 | 开销 | 说明 |
|------|------|------|
| 链路追踪延迟 | +2-3ms | Span 创建+传播 |
| 链路追踪 CPU | < 3% | SDK 开销 |
| 日志量增加 | +20% | TraceID/SpanID 字段 |
| 指标采集 | < 1% | Prometheus pull 模式 |
| 存储成本 | 约 ¥2万/月 | ES 存储日志+链路数据 |

## 常见陷阱（深度分析）

### 陷阱 1：日志不带 TraceID

**后果：** 看到链路中 RiskControlService 超时，但搜索日志时无法关联到具体请求。需要在 6 个服务的日志中按时间范围盲搜，效率极低。

**解决方案：** 使用 OpenTelemetry SDK 自动注入 TraceID 到日志 MDC。

### 陷阱 2：链路追踪采样率 100%

**后果：** 1 亿次请求/天 × 每个 Span 约 1KB = 100GB/天链路数据。ES 存储成本 ¥5 万/月。

**解决方案：** 正常请求采样 1%（足够统计），错误请求 100% 采样（必须全量保留）。

```yaml
# OpenTelemetry 采样配置
sampler:
  type: parent_based_trace_id_ratio
  ratio: 0.01  # 1% 采样率
  error_sampling: 1.0  # 错误请求 100% 采样
```

### 陷阱 3：告警风暴无降噪

**后果：** 一次故障触发 10 条告警 → 运维收到 10 条 Slack 通知 + 5 个电话 → 混乱。

**解决方案：** 告警关联器 + 根因推断 → 只通知根因告警。

### 陷阱 4：链路断裂

**场景：** A → B → C，B 在 Kafka 消息中忘记传递 TraceID → C 没有上游 TraceID → 链路在 B 处断裂。

**解决方案：** 使用 OpenTelemetry SDK 的自动 instrumentation（自动注入 Kafka Header），不要手动传递 TraceID。