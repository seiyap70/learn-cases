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

**完整的 OpenTelemetry 集成：上下文传播实现**

以上代码依赖 SDK 自动传播，下面展示完整的上下文传播机制，包括 HTTP、gRPC、Kafka 三种协议的显式传播实现，以及 Baggage 传递业务上下文。

```java
// ===== 1. OpenTelemetry SDK 初始化（应用启动时） =====
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.api.trace.propagation.W3CTraceContextPropagator;
import io.opentelemetry.context.propagation.ContextPropagators;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;
import io.opentelemetry.sdk.trace.samplers.Sampler;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;

public class TelemetryInitializer {

    private static OpenTelemetry openTelemetry;

    public static synchronized OpenTelemetry init(String serviceName) {
        if (openTelemetry != null) return openTelemetry;

        // 配置 OTLP Exporter（发送到 Jaeger/Collector）
        OtlpGrpcSpanExporter spanExporter = OtlpGrpcSpanExporter.builder()
                .setEndpoint("http://otel-collector:4317")
                .setTimeout(10, TimeUnit.SECONDS)
                .build();

        // 配置采样策略：正常 1%，错误 100%
        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
                .setSampler(Sampler.parentBased(
                    Sampler.traceIdRatioBased(0.01)  // 基础采样率 1%
                ))
                .addSpanProcessor(BatchSpanProcessor.builder(spanExporter)
                        .setMaxQueueSize(2048)
                        .setScheduleDelay(5, TimeUnit.SECONDS)
                        .setExporterTimeout(30, TimeUnit.SECONDS)
                        .build())
                .setResource(Resource.create(Attributes.of(
                    ResourceAttributes.SERVICE_NAME, serviceName,
                    ResourceAttributes.DEPLOYMENT_ENVIRONMENT, "production"
                )))
                .build();

        openTelemetry = OpenTelemetrySdk.builder()
                .setTracerProvider(tracerProvider)
                .setPropagators(ContextPropagators.create(
                    W3CTraceContextPropagator.getInstance()  // W3C 标准传播格式
                ))
                .buildAndRegisterGlobal();

        return openTelemetry;
    }
}
```

```java
// ===== 2. HTTP 调用的上下文传播（手动传播示例） =====
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.context.Context;
import io.opentelemetry.context.propagation.TextMapSetter;

public class HttpClientPropagation {

    // 定义如何将上下文注入 HTTP Headers
    private static final TextMapSetter<HttpHeaders> HTTP_SETTER = (carrier, key, value) -> {
        if (carrier != null && value != null) {
            carrier.put(key, value);
        }
    };

    public HttpResponse callDownstream(String url, String payload) {
        // 获取当前 Span 的上下文
        Span currentSpan = Span.current();
        Context currentContext = Context.current();

        // 创建子 Span
        Tracer tracer = GlobalOpenTelemetry.getTracer("order-service");
        Span childSpan = tracer.spanBuilder("HTTP POST " + url)
                .setParent(currentContext)
                .startSpan();

        try (Scope scope = childSpan.makeCurrent()) {
            // 构造 HTTP 请求
            HttpRequest request = new HttpRequest(url, payload);

            // 关键：将 Trace 上下文注入 HTTP Headers
            // W3C 格式会写入 traceparent 和 tracestate 两个 Header
            GlobalOpenTelemetry.getPropagators().getTextMapPropagator()
                    .inject(Context.current(), request.getHeaders(), HTTP_SETTER);

            // request.getHeaders() 此时包含:
            // traceparent: 00-abc123def456-567890-01
            // tracestate: key=value

            childSpan.setAttribute("http.method", "POST");
            childSpan.setAttribute("http.url", url);

            return httpClient.execute(request);
        } catch (Exception e) {
            childSpan.recordException(e);
            childSpan.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            childSpan.end();
        }
    }
}
```

```java
// ===== 3. gRPC 调用的上下文传播 =====
import io.grpc.Metadata;
import io.opentelemetry.context.propagation.TextMapSetter;
import io.opentelemetry.context.propagation.TextMapGetter;

public class GrpcPropagation {

    // gRPC Metadata Setter
    private static final TextMapSetter<Metadata> GRPC_SETTER = (metadata, key, value) -> {
        if (metadata != null && value != null) {
            metadata.put(Metadata.Key.of(key, Metadata.ASCII_STRING_MARSHALLER), value);
        }
    };

    // gRPC Metadata Getter（服务端接收时用）
    private static final TextMapGetter<Metadata> GRPC_GETTER = new TextMapGetter<>() {
        @Override
        public Iterable<String> keys(Metadata metadata) {
            return metadata.keys();
        }
        @Override
        public String get(Metadata metadata, String key) {
            return metadata.get(Metadata.Key.of(key, Metadata.ASCII_STRING_MARSHALLER));
        }
    };

    // 客户端：发送 gRPC 请求时注入上下文
    public void callPaymentService(PaymentRequest request) {
        Metadata metadata = new Metadata();

        // 注入 Trace 上下文到 gRPC Metadata
        GlobalOpenTelemetry.getPropagators().getTextMapPropagator()
                .inject(Context.current(), metadata, GRPC_SETTER);

        // 使用带有 metadata 的 stub 发送请求
        paymentServiceStub
                .withInterceptors(MetadataUtils.newAttachMetadataInterceptor(metadata))
                .charge(request);
    }

    // 服务端：从 gRPC Metadata 中提取上下文
    public void handleRequest(PaymentRequest request, Metadata metadata) {
        // 从 Metadata 提取上游上下文
        Context extractedContext = GlobalOpenTelemetry.getPropagators()
                .getTextMapPropagator()
                .extract(Context.current(), metadata, GRPC_GETTER);

        // 在提取的上下文中创建子 Span
        try (Scope scope = extractedContext.makeCurrent()) {
            Span serverSpan = tracer.spanBuilder("PaymentService.charge")
                    .setParent(extractedContext)
                    .startSpan();
            try (Scope spanScope = serverSpan.makeCurrent()) {
                // 业务逻辑处理
                processPayment(request);
            } finally {
                serverSpan.end();
            }
        }
    }
}
```

```java
// ===== 4. Kafka 消息的上下文传播 =====
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import io.opentelemetry.context.propagation.TextMapSetter;
import io.opentelemetry.context.propagation.TextMapGetter;

public class KafkaPropagation {

    // Kafka Header Setter
    private static final TextMapSetter<ProducerRecord<String, String>> KAFKA_SETTER =
        (record, key, value) -> {
            if (record != null && value != null) {
                record.headers().add(key, value.getBytes(StandardCharsets.UTF_8));
            }
        };

    // Kafka Header Getter
    private static final TextMapGetter<ConsumerRecord<String, String>> KAFKA_GETTER =
        new TextMapGetter<>() {
            @Override
            public Iterable<String> keys(ConsumerRecord<String, String> record) {
                List<String> keys = new ArrayList<>();
                record.headers().forEach(header -> keys.add(header.key()));
                return keys;
            }
            @Override
            public String get(ConsumerRecord<String, String> record, String key) {
                Header header = record.headers().lastHeader(key);
                return header != null ? new String(header.value(), StandardCharsets.UTF_8) : null;
            }
        };

    // 生产者：发送消息时注入上下文
    public void sendOrderEvent(OrderEvent event) {
        Tracer tracer = GlobalOpenTelemetry.getTracer("order-service");
        Span span = tracer.spanBuilder("kafka.produce order-events")
                .setSpanKind(SpanKind.PRODUCER)
                .startSpan();

        try (Scope scope = span.makeCurrent()) {
            String payload = JsonSerializer.serialize(event);
            ProducerRecord<String, String> record =
                    new ProducerRecord<>("order-events", event.getOrderId(), payload);

            // 关键：将 Trace 上下文注入 Kafka 消息头
            GlobalOpenTelemetry.getPropagators().getTextMapPropagator()
                    .inject(Context.current(), record, KAFKA_SETTER);

            // record.headers() 此时包含:
            // traceparent: 00-abc123def456-567890-01

            span.setAttribute("messaging.system", "kafka");
            span.setAttribute("messaging.destination", "order-events");
            span.setAttribute("messaging.message_type", "publish");

            kafkaProducer.send(record);
        } finally {
            span.end();
        }
    }

    // 消费者：消费消息时提取上下文
    public void consumeOrderEvent(ConsumerRecord<String, String> record) {
        // 从 Kafka 消息头提取上游 Trace 上下文
        Context extractedContext = GlobalOpenTelemetry.getPropagators()
                .getTextMapPropagator()
                .extract(Context.current(), record, KAFKA_GETTER);

        // 在提取的上下文中创建消费者 Span
        try (Scope scope = extractedContext.makeCurrent()) {
            Span consumerSpan = tracer.spanBuilder("kafka.consume order-events")
                    .setParent(extractedContext)
                    .setSpanKind(SpanKind.CONSUMER)
                    .startSpan();

            try (Scope spanScope = consumerSpan.makeCurrent()) {
                consumerSpan.setAttribute("messaging.system", "kafka");
                consumerSpan.setAttribute("messaging.destination", "order-events");
                consumerSpan.setAttribute("messaging.message_type", "receive");

                // 业务逻辑：处理订单事件
                processOrderEvent(JsonDeserializer.deserialize(record.value(), OrderEvent.class));
            } finally {
                consumerSpan.end();
            }
        }
    }
}
```

```java
// ===== 5. Baggage 传递业务上下文 =====
// Baggage 是跨服务传递业务键值对的机制，与 Trace 上下文一起自动传播
import io.opentelemetry.api.baggage.Baggage;

public class BaggagePropagation {

    public void handleOrder(Order order) {
        // 在入口服务设置 Baggage（会随 Trace 上下文一起传播到下游）
        Baggage baggage = Baggage.builder()
                .put("user_id", order.getUserId())
                .put("order_id", order.getOrderId())
                .put("tenant_id", order.getTenantId())
                .put("env", "production")
                .build();

        try (Scope scope = baggage.makeCurrent()) {
            // 下游服务可自动获取这些值
            paymentService.charge(order);
        }
    }

    // 下游服务获取 Baggage
    public void charge(Order order) {
        Baggage currentBaggage = Baggage.current();
        String userId = currentBaggage.getEntryValue("user_id");
        String tenantId = currentBaggage.getEntryValue("tenant_id");

        // 用于日志和业务逻辑
        logger.info("Processing payment for user={}, tenant={}", userId, tenantId);
    }
}
```

```java
// ===== 6. Span Linking：跨异步边界关联链路 =====
// Span Link 用于关联不属于同一 Trace 树的 Span，典型场景：
// - 消息队列：生产者 Span 和消费者 Span 分属不同 Trace
// - 批处理任务：触发者和执行者分属不同 Trace
// - 手动重试：原请求和新请求分属不同 Trace

import io.opentelemetry.api.trace.Link;

public class SpanLinkingExample {

    // 场景 1：Kafka 生产者-消费者 Span Linking
    // 消费者收到消息时，创建一个新 Trace（因为跨进程边界），
    // 但通过 Link 关联到生产者的 Span
    public void consumeWithLinking(ConsumerRecord<String, String> record) {
        // 从消息头提取上游上下文
        Context extractedContext = GlobalOpenTelemetry.getPropagators()
                .getTextMapPropagator()
                .extract(Context.current(), record, KAFKA_GETTER);

        // 获取上游 SpanContext 用于 Link
        SpanContext upstreamSpanContext = Span.fromContext(extractedContext).getSpanContext();

        // 创建新的消费者 Span，并通过 Link 关联上游生产者 Span
        Span consumerSpan = tracer.spanBuilder("kafka.consume order-events")
                .setSpanKind(SpanKind.CONSUMER)
                // 关键：不设置 parent，而是添加 Link
                // 这意味着消费者 Span 是新 Trace 的根，
                // 但通过 Link 可以追溯到生产者的 Span
                .addLink(upstreamSpanContext)
                // 可以为 Link 添加属性说明关联原因
                .startSpan();

        try (Scope scope = consumerSpan.makeCurrent()) {
            consumerSpan.setAttribute("messaging.system", "kafka");
            consumerSpan.setAttribute("messaging.destination", "order-events");
            consumerSpan.setAttribute("link.reason", "async_message_boundary");

            // 处理消息
            processOrderEvent(JsonDeserializer.deserialize(record.value(), OrderEvent.class));
        } finally {
            consumerSpan.end();
        }
    }

    // 场景 2：批处理任务 Span Linking
    // 定时任务扫描数据库，为每条记录创建独立的处理 Trace，
    // 通过 Link 关联到触发者（定时任务）的 Span
    public void processBatchJob(List<Order> pendingOrders) {
        Span triggerSpan = tracer.spanBuilder("batch.order-scan")
                .setSpanKind(SpanKind.INTERNAL)
                .startSpan();

        try (Scope scope = triggerSpan.makeCurrent()) {
            for (Order order : pendingOrders) {
                // 每条订单创建独立的处理 Span，通过 Link 关联到触发者
                Span orderProcessSpan = tracer.spanBuilder("batch.process-order")
                        .addLink(triggerSpan.getSpanContext())
                        .setAttribute("order.id", order.getOrderId())
                        .startSpan();

                try (Scope orderScope = orderProcessSpan.makeCurrent()) {
                    processOrder(order);
                } catch (Exception e) {
                    orderProcessSpan.recordException(e);
                    orderProcessSpan.setStatus(StatusCode.ERROR, e.getMessage());
                } finally {
                    orderProcessSpan.end();
                }
            }
        } finally {
            triggerSpan.end();
        }
    }
}
```

**W3C TraceContext 传播格式详解**

W3C TraceContext 是跨服务传播 Trace 上下文的标准协议，由两个 HTTP Header 组成：

```
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
              │  │                      │                     │  │
              │  │                      │                     │  └─ trace-flags (01=sampled)
              │  │                      │                     └─ 1 hex digit
              │  │                      └─ parent-id (span-id): 16 hex digits
              │  └─ trace-id: 32 hex digits (128 bit, 全局唯一)
              └─ version: 00 (当前固定)

tracestate: vendor=value,vendor2=value2   ← 可选，供应商特定信息
```

```java
// ===== 7. W3C TraceContext 手动解析与构造 =====
// 某些场景需要手动处理 traceparent（如与遗留系统对接）

public class W3CTraceContextManual {

    /**
     * 手动解析 traceparent Header
     * 格式: version-traceid-parentid-traceflags
     */
    public static TraceContext parse(String traceparentHeader) {
        if (traceparentHeader == null || traceparentHeader.isEmpty()) {
            return null;
        }

        String[] parts = traceparentHeader.split("-");
        if (parts.length != 4) {
            return null;
        }

        String version = parts[0];
        if (!"00".equals(version)) {
            return null; // 仅支持 version 00
        }

        String traceId = parts[1];
        if (traceId.length() != 32 || traceId.matches("0{32}")) {
            return null; // trace-id 必须为 32 hex 且不全为 0
        }

        String parentId = parts[2];
        if (parentId.length() != 16 || parentId.matches("0{16}")) {
            return null; // parent-id 必须为 16 hex 且不全为 0
        }

        String traceFlags = parts[3];
        boolean sampled = "01".equals(traceFlags);

        return new TraceContext(traceId, parentId, sampled);
    }

    /**
     * 手动构造 traceparent Header
     */
    public static String format(String traceId, String parentId, boolean sampled) {
        return String.format("00-%s-%s-%s",
                traceId, parentId, sampled ? "01" : "00");
    }

    /**
     * 与遗留系统对接：将旧版 X-B3-TraceId 转为 W3C traceparent
     * Spring Cloud Sleuth / Zipkin 使用 B3 传播格式
     */
    public static String convertB3ToW3C(String b3TraceId, String b3SpanId, boolean sampled) {
        // B3 trace-id 可能为 16 或 32 hex，W3C 要求 32 hex
        String paddedTraceId = b3TraceId.length() == 16
                ? "0000000000000000" + b3TraceId
                : b3TraceId;
        return format(paddedTraceId, b3SpanId, sampled);
    }

    public record TraceContext(String traceId, String parentId, boolean sampled) {}
}
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

**链路数据存储 Schema 设计**

```sql
-- ===== Trace 存储 Schema（Elasticsearch 索引映射，Jaeger 使用） =====
-- Jaeger 默认使用 ES 存储，以下为关键索引结构设计

-- 索引 1: jaeger-span（核心 Span 数据）
-- 对应 ES Mapping:
-- {
--   "mappings": {
--     "properties": {
--       "traceID":         { "type": "keyword" },
--       "spanID":          { "type": "keyword" },
--       "operationName":   { "type": "keyword" },
--       "startTimeMillis": { "type": "date", "format": "epoch_millis" },
--       "durationMillis":  { "type": "long" },
--       "flags":           { "type": "integer" },
--       "process": {
--         "properties": {
--           "serviceName": { "type": "keyword" },
--           "tags":        { "type": "object", "enabled": false }
--         }
--       },
--       "references":      { "type": "nested" },
--       "tags":            { "type": "object", "enabled": false },
--       "tag_q": {
--         "type": "keyword",
--         "index": true
--       },
--       "logs":            { "type": "object", "enabled": false }
--     }
--   }
-- }

-- 索引 2: jaeger-service（服务名索引，加速服务维度查询）
-- {
--   "mappings": {
--     "properties": {
--       "serviceName":    { "type": "keyword" },
--       "operationName":  { "type": "keyword" }
--     }
--   }
-- }

-- ===== 关系型数据库补充：服务依赖关系表 =====
-- 用于告警关联和拓扑图渲染，定期从 Trace 数据中聚合

CREATE TABLE service_dependency (
    id              BIGSERIAL PRIMARY KEY,
    source_service  VARCHAR(64) NOT NULL COMMENT '上游服务名',
    target_service  VARCHAR(64) NOT NULL COMMENT '下游服务名',
    protocol        VARCHAR(16) NOT NULL COMMENT '调用协议: HTTP/gRPC/Kafka',
    operation       VARCHAR(128) NOT NULL COMMENT '调用操作名',
    avg_latency_ms  DOUBLE PRECISION DEFAULT 0 COMMENT '平均延迟(ms)',
    p99_latency_ms  DOUBLE PRECISION DEFAULT 0 COMMENT 'P99延迟(ms)',
    error_rate      DOUBLE PRECISION DEFAULT 0 COMMENT '错误率(0-1)',
    call_count      BIGINT DEFAULT 0 COMMENT '统计周期内调用次数',
    stat_period     VARCHAR(16) NOT NULL COMMENT '统计周期: 1m/5m/1h',
    start_time      TIMESTAMP NOT NULL COMMENT '统计开始时间',
    end_time        TIMESTAMP NOT NULL COMMENT '统计结束时间',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(source_service, target_service, protocol, operation, start_time)
);

CREATE INDEX idx_dep_source ON service_dependency(source_service);
CREATE INDEX idx_dep_target ON service_dependency(target_service);
CREATE INDEX idx_dep_time ON service_dependency(start_time, end_time);

-- ===== 告警规则表 =====
CREATE TABLE alert_rule (
    id              BIGSERIAL PRIMARY KEY,
    rule_name       VARCHAR(128) NOT NULL COMMENT '规则名称',
    rule_type       VARCHAR(32) NOT NULL COMMENT '规则类型: metric_trace/log_trace/composite',
    service_name    VARCHAR(64) COMMENT '关联服务(为空表示全局规则)',
    promql_expr     TEXT COMMENT 'PromQL 表达式(metric_trace类型)',
    threshold       DOUBLE PRECISION COMMENT '阈值',
    comparison_op   VARCHAR(8) COMMENT '比较运算符: gt/lt/gte/lte/eq',
    duration_sec    INT DEFAULT 120 COMMENT '持续时间(秒)',
    severity        VARCHAR(16) NOT NULL COMMENT '严重级别: P0/P1/P2/P3',
    notify_channel  VARCHAR(64) NOT NULL COMMENT '通知渠道: phone/slack/email',
    notify_group    VARCHAR(64) COMMENT '通知分组(值班组)',
    enabled         BOOLEAN DEFAULT TRUE,
    runbook_url     VARCHAR(256) COMMENT 'Runbook 文档链接',
    suppress_rules  JSONB COMMENT '降噪规则: {"suppress_after_sec": 300, "group_by": ["service"]}',
    created_by      VARCHAR(64),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_alert_rule_service ON alert_rule(service_name);
CREATE INDEX idx_alert_rule_severity ON alert_rule(severity);
CREATE INDEX idx_alert_rule_enabled ON alert_rule(enabled) WHERE enabled = TRUE;

-- ===== 告警事件表 =====
CREATE TABLE alert_event (
    id              BIGSERIAL PRIMARY KEY,
    fingerprint     VARCHAR(64) NOT NULL COMMENT '告警指纹(去重键)',
    rule_id         BIGINT REFERENCES alert_rule(id),
    service_name    VARCHAR(64) NOT NULL,
    severity        VARCHAR(16) NOT NULL,
    status          VARCHAR(16) NOT NULL COMMENT 'firing/resolved/suppressed',
    value           DOUBLE PRECISION COMMENT '触发时的指标值',
    threshold       DOUBLE PRECISION COMMENT '阈值',
    correlation_id  VARCHAR(64) COMMENT '关联ID(根因推断聚合后共享同一ID)',
    is_root_cause   BOOLEAN DEFAULT FALSE COMMENT '是否为根因告警',
    affected_services JSONB COMMENT '受影响的服务列表',
    trace_id        VARCHAR(64) COMMENT '关联的 TraceID',
    labels          JSONB COMMENT '告警标签',
    annotations     JSONB COMMENT '告警注解',
    fired_at        TIMESTAMP NOT NULL,
    resolved_at     TIMESTAMP,
    notified_at     TIMESTAMP,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_alert_fp ON alert_event(fingerprint);
CREATE INDEX idx_alert_svc ON alert_event(service_name);
CREATE INDEX idx_alert_corr ON alert_event(correlation_id);
CREATE INDEX idx_alert_fired ON alert_event(fired_at);
CREATE INDEX idx_alert_status ON alert_event(status) WHERE status = 'firing';

-- ===== 告警关联组表 =====
CREATE TABLE alert_correlation_group (
    id              BIGSERIAL PRIMARY KEY,
    correlation_id  VARCHAR(64) NOT NULL UNIQUE,
    root_cause_service VARCHAR(64) NOT NULL COMMENT '根因服务',
    root_cause_event_id BIGINT REFERENCES alert_event(id),
    affected_services JSONB NOT NULL COMMENT '受影响服务及告警ID映射',
    correlation_score DOUBLE PRECISION DEFAULT 0 COMMENT '关联置信度(0-1)',
    correlation_method VARCHAR(32) NOT NULL COMMENT '关联方法: topology/time_series/bayesian',
    total_alerts    INT DEFAULT 0 COMMENT '聚合的告警总数',
    suppressed_alerts INT DEFAULT 0 COMMENT '被抑制的告警数',
    status          VARCHAR(16) NOT NULL COMMENT 'active/resolved',
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    resolved_at     TIMESTAMP
);

CREATE INDEX idx_corr_group_root ON alert_correlation_group(root_cause_service);
CREATE INDEX idx_corr_group_status ON alert_correlation_group(status) WHERE status = 'active';
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

**完整的告警关联引擎实现**

上面的 `AlertCorrelator` 是概念性示例，下面给出完整的生产级告警关联引擎，包含：基于 trace_id / 服务依赖拓扑 / 时间窗口的三重关联，去重逻辑，以及根因推断算法。

```python
# ===== 告警关联引擎：完整实现 =====
import hashlib
import time
from datetime import datetime, timedelta
from dataclasses import dataclass, field
from typing import List, Optional, Dict, Set, Tuple
from collections import defaultdict
import json
import logging

logger = logging.getLogger("alert_correlation_engine")


@dataclass
class Alert:
    """告警事件"""
    alert_id: str
    fingerprint: str              # 告警指纹（用于去重）
    service_name: str
    alert_name: str               # 告警规则名称
    severity: str                 # P0/P1/P2/P3
    status: str = "firing"        # firing / resolved / suppressed
    value: float = 0.0
    threshold: float = 0.0
    trace_id: Optional[str] = None
    labels: Dict[str, str] = field(default_factory=dict)
    annotations: Dict[str, str] = field(default_factory=dict)
    fired_at: datetime = field(default_factory=datetime.utcnow)
    resolved_at: Optional[datetime] = None
    correlation_id: Optional[str] = None
    is_root_cause: bool = False


@dataclass
class CorrelationGroup:
    """告警关联组：将同一根因的告警聚合在一起"""
    correlation_id: str
    root_cause_alert: Optional[Alert] = None
    root_cause_service: str = ""
    affected_alerts: List[Alert] = field(default_factory=list)
    affected_services: Set[str] = field(default_factory=set)
    correlation_score: float = 0.0
    correlation_method: str = ""     # topology / trace / time_series
    created_at: datetime = field(default_factory=datetime.utcnow)
    resolved_at: Optional[datetime] = None
    status: str = "active"


class ServiceDependencyGraph:
    """服务依赖拓扑图"""

    def __init__(self):
        # 存储服务间的依赖关系
        # dependencies[A] = {B, C} 表示 A 依赖 B 和 C（A 调用 B、C）
        self.dependencies: Dict[str, Set[str]] = defaultdict(set)
        # 反向依赖：dependents[B] = {A} 表示 B 被 A 依赖
        self.dependents: Dict[str, Set[str]] = defaultdict(set)

    def add_dependency(self, caller: str, callee: str):
        """添加依赖关系：caller 调用 callee"""
        self.dependencies[caller].add(callee)
        self.dependents[callee].add(caller)

    def get_dependencies(self, service: str) -> Set[str]:
        """获取服务的直接下游依赖"""
        return self.dependencies.get(service, set())

    def get_dependents(self, service: str) -> Set[str]:
        """获取依赖此服务的上游服务"""
        return self.dependents.get(service, set())

    def get_all_downstream(self, service: str) -> Set[str]:
        """获取所有下游服务（递归，含直接和间接依赖）"""
        visited = set()
        stack = [service]
        while stack:
            current = stack.pop()
            for dep in self.dependencies.get(current, set()):
                if dep not in visited:
                    visited.add(dep)
                    stack.append(dep)
        return visited

    def is_upstream_of(self, candidate_cause: str, affected: str) -> bool:
        """判断 candidate_cause 是否是 affected 的上游（被 affected 依赖）"""
        return candidate_cause in self.get_all_downstream(affected)

    def find_common_root(self, services: List[str]) -> Optional[str]:
        """找出多个服务共同的下游根因"""
        if not services:
            return None
        downstream_sets = [self.get_all_downstream(s) | {s} for s in services]
        common = downstream_sets[0]
        for s in downstream_sets[1:]:
            common = common & s
        # 在共同下游中，找最深（离所有服务最远）的节点作为根因
        # 简化：按拓扑排序返回最深的
        for candidate in common:
            # 如果 candidate 的所有下游都不在 common 中，它就是最深节点
            candidate_downstream = self.get_all_downstream(candidate)
            if not (candidate_downstream & common):
                return candidate
        return min(common) if common else None


class AlertCorrelationEngine:
    """
    生产级告警关联引擎
    三重关联策略：
    1. Trace ID 关联：同一 Trace 中的告警必然相关
    2. 拓扑关联：依赖链上的告警可能因果相关
    3. 时间窗口关联：同一时间窗内的告警可能相关
    """

    def __init__(self, dependency_graph: ServiceDependencyGraph,
                 time_window_minutes: int = 5,
                 dedup_window_minutes: int = 10):
        self.dependency_graph = dependency_graph
        self.time_window = timedelta(minutes=time_window_minutes)
        self.dedup_window = timedelta(minutes=dedup_window_minutes)

        # 活跃告警存储
        self.active_alerts: Dict[str, Alert] = {}       # fingerprint -> Alert
        # 关联组存储
        self.correlation_groups: Dict[str, CorrelationGroup] = {}  # correlation_id -> Group
        # trace_id 到告警的映射
        self.trace_alert_map: Dict[str, List[Alert]] = defaultdict(list)
        # 服务到告警的映射
        self.service_alert_map: Dict[str, List[Alert]] = defaultdict(list)

    def process_alert(self, alert: Alert) -> Tuple[Optional[CorrelationGroup], bool]:
        """
        处理新告警：去重 -> 关联 -> 根因推断
        返回: (关联组, 是否为新告警)
        """
        # 第一步：去重
        if self._is_duplicate(alert):
            logger.info(f"Alert deduplicated: {alert.fingerprint}")
            return None, False

        # 第二步：尝试关联到现有关联组
        correlated_group = self._correlate(alert)

        if correlated_group:
            # 第三步：加入现有关联组并更新根因推断
            self._add_to_group(correlated_group, alert)
            # 重新推断根因
            self._recalculate_root_cause(correlated_group)
        else:
            # 新建关联组
            correlated_group = self._create_new_group(alert)

        # 存储告警
        self.active_alerts[alert.fingerprint] = alert
        if alert.trace_id:
            self.trace_alert_map[alert.trace_id].append(alert)
        self.service_alert_map[alert.service_name].append(alert)

        return correlated_group, True

    def _is_duplicate(self, alert: Alert) -> bool:
        """基于指纹的去重：同一服务同一告警规则在窗口内不重复"""
        existing = self.active_alerts.get(alert.fingerprint)
        if existing and existing.status == "firing":
            # 检查是否在去重窗口内
            if existing.fired_at + self.dedup_window > alert.fired_at:
                return True
        return False

    def _correlate(self, alert: Alert) -> Optional[CorrelationGroup]:
        """三重关联策略"""
        # 策略 1：Trace ID 关联（最强关联信号）
        if alert.trace_id:
            trace_alerts = self.trace_alert_map.get(alert.trace_id, [])
            for existing in trace_alerts:
                if existing.correlation_id:
                    group = self.correlation_groups.get(existing.correlation_id)
                    if group and group.status == "active":
                        return group

        # 策略 2：拓扑关联（基于服务依赖图）
        for group in self.correlation_groups.values():
            if group.status != "active":
                continue
            # 新告警的服务是否依赖关联组中已有服务，或被其依赖
            for group_service in group.affected_services | {group.root_cause_service}:
                if (self.dependency_graph.is_upstream_of(group_service, alert.service_name) or
                    self.dependency_graph.is_upstream_of(alert.service_name, group_service)):
                    # 拓扑关联，计算置信度
                    score = self._calculate_topology_score(alert, group)
                    if score >= 0.6:  # 置信度阈值
                        return group

        # 策略 3：时间窗口关联（最弱信号，作为兜底）
        for group in self.correlation_groups.values():
            if group.status != "active":
                continue
            time_diff = abs((alert.fired_at - group.created_at).total_seconds())
            if time_diff <= self.time_window.total_seconds():
                # 同一时间窗内的告警，检查服务间是否有间接关系
                if self._has_indirect_relation(alert.service_name, group):
                    return group

        return None

    def _calculate_topology_score(self, alert: Alert, group: CorrelationGroup) -> float:
        """计算拓扑关联的置信度分数"""
        score = 0.0

        # 直接依赖关系 +0.4
        for group_service in group.affected_services | {group.root_cause_service}:
            if alert.service_name in self.dependency_graph.get_dependencies(group_service):
                score += 0.4
                break
            if group_service in self.dependency_graph.get_dependencies(alert.service_name):
                score += 0.4
                break

        # 时间接近度 +0~0.3（越接近分数越高）
        if group.affected_alerts:
            time_diff = abs((alert.fired_at - group.created_at).total_seconds())
            time_score = max(0, 1 - time_diff / self.time_window.total_seconds()) * 0.3
            score += time_score

        # 告警类型相关性 +0.3（如都是错误率告警）
        for existing_alert in group.affected_alerts:
            if existing_alert.alert_name == alert.alert_name:
                score += 0.3
                break

        return min(score, 1.0)

    def _has_indirect_relation(self, service: str, group: CorrelationGroup) -> bool:
        """检查服务与关联组是否有间接关系（共同的上游/下游）"""
        group_services = group.affected_services | {group.root_cause_service}
        service_deps = self.dependency_graph.get_all_downstream(service)
        for gs in group_services:
            gs_deps = self.dependency_graph.get_all_downstream(gs)
            # 有共同下游
            if service_deps & gs_deps:
                return True
        return False

    def _add_to_group(self, group: CorrelationGroup, alert: Alert):
        """将告警加入关联组"""
        group.affected_alerts.append(alert)
        group.affected_services.add(alert.service_name)
        alert.correlation_id = group.correlation_id

        # 如果新告警更严重，可能需要重新评估
        severity_order = {"P0": 0, "P1": 1, "P2": 2, "P3": 3}
        if severity_order.get(alert.severity, 3) < severity_order.get(
                group.root_cause_alert.severity if group.root_cause_alert else "P3", 3):
            # 新告警优先级更高，不自动变根因，但标记需重新评估
            pass

    def _create_new_group(self, alert: Alert) -> CorrelationGroup:
        """创建新的关联组"""
        correlation_id = f"corr-{alert.fingerprint}-{int(time.time())}"
        group = CorrelationGroup(
            correlation_id=correlation_id,
            root_cause_alert=alert,
            root_cause_service=alert.service_name,
            affected_services={alert.service_name},
            correlation_method="single_alert",
            status="active"
        )
        alert.correlation_id = correlation_id
        alert.is_root_cause = True
        self.correlation_groups[correlation_id] = group
        return group

    def _recalculate_root_cause(self, group: CorrelationGroup):
        """
        重新推断根因：基于拓扑排序 + 告警严重度 + 时间顺序
        核心算法：
        1. 在依赖图中，根因服务应该是"最深"的下游服务
        2. 如果多个服务在同一层级，选告警级别最高的
        3. 如果告警级别相同，选最早触发的
        """
        services = list(group.affected_services)
        if len(services) <= 1:
            return

        # 找出共同的下游根因
        common_root = self.dependency_graph.find_common_root(services)

        if common_root:
            # 找到该服务的告警
            root_alert = None
            for alert in group.affected_alerts:
                if alert.service_name == common_root:
                    root_alert = alert
                    break

            if root_alert:
                # 更新根因
                if group.root_cause_alert:
                    group.root_cause_alert.is_root_cause = False
                root_alert.is_root_cause = True
                group.root_cause_alert = root_alert
                group.root_cause_service = common_root
                group.correlation_method = "topology"

                # 抑制非根因告警的通知
                for alert in group.affected_alerts:
                    if not alert.is_root_cause:
                        alert.status = "suppressed"
        else:
            # 无法通过拓扑确定根因，按严重度和时间排序
            severity_order = {"P0": 0, "P1": 1, "P2": 2, "P3": 3}
            sorted_alerts = sorted(
                group.affected_alerts,
                key=lambda a: (severity_order.get(a.severity, 3), a.fired_at)
            )
            if sorted_alerts:
                if group.root_cause_alert:
                    group.root_cause_alert.is_root_cause = False
                sorted_alerts[0].is_root_cause = True
                group.root_cause_alert = sorted_alerts[0]
                group.root_cause_service = sorted_alerts[0].service_name
                group.correlation_method = "severity_time"

    def resolve_group(self, correlation_id: str):
        """解决关联组：标记所有告警为 resolved"""
        group = self.correlation_groups.get(correlation_id)
        if not group:
            return
        group.status = "resolved"
        group.resolved_at = datetime.utcnow()
        for alert in group.affected_alerts:
            alert.status = "resolved"
            alert.resolved_at = datetime.utcnow()

    def get_notification_summary(self, correlation_id: str) -> str:
        """生成告警通知摘要（只通知关键信息）"""
        group = self.correlation_groups.get(correlation_id)
        if not group:
            return ""

        root = group.root_cause_alert
        affected = [a for a in group.affected_alerts if not a.is_root_cause]

        summary = f"[根因告警] {root.service_name} - {root.alert_name}\n"
        summary += f"  严重级别: {root.severity}\n"
        summary += f"  当前值: {root.value}, 阈值: {root.threshold}\n"
        if root.trace_id:
            summary += f"  关联 Trace: {root.trace_id}\n"

        if affected:
            summary += f"\n[受影响服务] {len(affected)} 个\n"
            for a in affected:
                summary += f"  - {a.service_name}: {a.alert_name} ({a.severity})\n"
            summary += f"\n建议：优先排查 {root.service_name}，其异常可能是其他服务告警的根因。"

        return summary


# ===== 告警关联引擎使用示例 =====
def demo_correlation_engine():
    # 1. 初始化服务依赖图
    graph = ServiceDependencyGraph()
    graph.add_dependency("order-service", "payment-service")
    graph.add_dependency("order-service", "inventory-service")
    graph.add_dependency("payment-service", "risk-control-service")
    graph.add_dependency("risk-control-service", "compliance-api")

    # 2. 初始化关联引擎
    engine = AlertCorrelationEngine(
        dependency_graph=graph,
        time_window_minutes=5,
        dedup_window_minutes=10
    )

    # 3. 模拟告警风暴
    base_time = datetime.utcnow()

    # 3.1 风控服务首先告警（根因）
    alert1 = Alert(
        alert_id="a1",
        fingerprint=hashlib.md5(b"risk-control:HighErrorRate").hexdigest()[:16],
        service_name="risk-control-service",
        alert_name="HighErrorRate",
        severity="P0",
        value=0.12,
        threshold=0.05,
        trace_id="abc123",
        fired_at=base_time
    )

    # 3.2 支付服务随后告警（受影响）
    alert2 = Alert(
        alert_id="a2",
        fingerprint=hashlib.md5(b"payment:HighErrorRate").hexdigest()[:16],
        service_name="payment-service",
        alert_name="HighErrorRate",
        severity="P0",
        value=0.082,
        threshold=0.05,
        trace_id="abc123",
        fired_at=base_time + timedelta(seconds=30)
    )

    # 3.3 订单服务告警（受影响）
    alert3 = Alert(
        alert_id="a3",
        fingerprint=hashlib.md5(b"order:HighErrorRate").hexdigest()[:16],
        service_name="order-service",
        alert_name="HighErrorRate",
        severity="P1",
        value=0.051,
        threshold=0.05,
        trace_id="abc123",
        fired_at=base_time + timedelta(seconds=45)
    )

    # 3.4 风控服务的延迟告警（与 alert1 同服务，触发去重）
    alert4 = Alert(
        alert_id="a4",
        fingerprint=hashlib.md5(b"risk-control:HighErrorRate").hexdigest()[:16],
        service_name="risk-control-service",
        alert_name="HighErrorRate",
        severity="P0",
        value=0.15,
        threshold=0.05,
        fired_at=base_time + timedelta(seconds=60)
    )

    # 4. 处理告警
    group1, is_new1 = engine.process_alert(alert1)
    group2, is_new2 = engine.process_alert(alert2)
    group3, is_new3 = engine.process_alert(alert3)
    group4, is_new4 = engine.process_alert(alert4)  # 去重，不会创建新告警

    # 5. 输出结果
    print(f"Alert1 new={is_new1}, group={group1.correlation_id}")
    print(f"Alert2 new={is_new2}, correlated to group={group2.correlation_id}")
    print(f"Alert3 new={is_new3}, correlated to group={group3.correlation_id}")
    print(f"Alert4 new={is_new4}, deduplicated")

    # 6. 通知摘要
    print(engine.get_notification_summary(group1.correlation_id))
    # 输出:
    # [根因告警] risk-control-service - HighErrorRate
    #   严重级别: P0
    #   当前值: 0.12, 阈值: 0.05
    #   关联 Trace: abc123
    #
    # [受影响服务] 2 个
    #   - payment-service: HighErrorRate (P0)
    #   - order-service: HighErrorRate (P1)
    #
    # 建议：优先排查 risk-control-service，其异常可能是其他服务告警的根因。
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
## 性能与成本分析

**Trace 存储规模估算：**

| 规模 | 日均 Span 数 | 月存储量(原始) | ClickHouse 压缩后 | 月成本 |
|------|------------|--------------|------------------|-------|
| 小型 | 1 亿 | 200GB | 20GB | ¥0.5 万 |
| 中型 | 10 亿 | 2TB | 200GB | ¥3 万 |
| 大型 | 100 亿 | 20TB | 2TB | ¥15 万 |

**采样率对可观测性的影响：**

| 采样率 | 月存储 | 捕获率 | 延迟异常发现 | 适用场景 |
|--------|-------|-------|------------|---------|
| 100% | 2TB | 全部 | 最佳 | 核心服务 |
| 10% | 200GB | 10% | 良好 | 一般服务 |
| 1% | 20GB | 1% | 可接受 | 边缘服务 |
| 自适应 | 动态 | 动态 | 最优 | 推荐方案 |

**查询延迟基准：**

| 查询类型 | ClickHouse | Elasticsearch | Prometheus |
|---------|-----------|--------------|-----------|
| 单 trace 查询 | <100ms | <200ms | N/A |
| 服务拓扑图 | 2-5s | 5-10s | N/A |
| P99 延迟趋势 | <1s | 2-3s | <500ms |
| 错误率聚合 | <1s | 1-2s | <500ms |

**月度基础设施成本：**

| 组件 | 规格 | 数量 | 月成本 |
|------|------|------|-------|
| OTEL Collector | 4c8G | 5 台 | ¥2 万 |
| Kafka（Trace 数据） | 8c32G + 2TB | 6 节点 | ¥3 万 |
| ClickHouse | 8c64G + 5TB SSD | 3 节点 | ¥5 万 |
| Prometheus | 8c32G + 2TB SSD | 3 节点 | ¥3 万 |
| Grafana | 4c8G | 2 台 | ¥0.5 万 |
| **合计** | | | **¥13.5 万** |

## 异常场景完整演练

### 场景 1：断路器误触发导致级联熔断

```
触发：服务 B 响应慢（P99 从 200ms 升至 800ms）但未宕机
      → 断路器统计失败率超 50% → 触发熔断
      → 服务 A 调用 B 全部走降级 → B 实际可服务但被拒绝
级联：
  A 的断路器打开 → A 所有请求降级 → 用户看到错误页面
  → 但 B 只是慢，不是挂 → 大部分请求其实可以成功
处理：
  1. 断路器参数调优：失败率阈值从 50% 调至 70%
  2. 慢调用不算失败：只统计 5xx 错误，超时不算
  3. 半开状态探测：每 10s 放行 1 个请求测试
  4. 增加 "慢但可用" 状态：不是全有或全无
预防：断路器区分"慢"和"挂"，不同故障不同策略
```

### 场景 2：Kafka 消费者 Rebalance 导致 Trace 丢失

```
触发：OTEL Collector 节点扩容 → Kafka 消费者 Rebalance
      → Rebalance 期间（10-30秒）Trace 数据无法消费
      → 这段时间的 Span 丢失 → 链路不完整
检测：
  1. 监控 consumer_lag 指标 → Rebalance 时 lag 突增
  2. Trace 完整性检查：入口 Span 有但出口 Span 缺失
处理：
  1. 使用 CooperativeStickyAssignor 减少 Rebalance 影响
  2. Rebalance 期间数据暂存 Kafka → 恢复后继续消费
  3. 设置 session.timeout.ms=30s 减少 Rebalance 频率
预防：避免频繁扩缩容 + 使用 sticky 分配策略
```

### 场景 3：Metric 基数爆炸

```
触发：开发者添加 user_id 作为 Prometheus label
      → 1000 万用户 × 10 个 metric = 1 亿 time series
      → Prometheus 内存从 4GB 暴增到 100GB → OOM 崩溃
检测：
  1. Prometheus cardinality 分析：检查 label 值数量
  2. 自动告警：single metric > 10K series → 通知
处理：
  1. 立即移除 user_id label → 重启 Prometheus
  2. 将 user_id 维度移到 ClickHouse（支持高基数查询）
  3. Prometheus 只保留 service/method/status_code 等低基数 label
预防：
  1. CI 检查：新 metric 的 label 候选值 < 100
  2. 自定义 metric 必须经过审批
  3. 高基数维度（user_id, request_id）→ Trace 系统，不进 Metric
```

### OpenTelemetry 上下文传播实现

```python
class TraceContextInterceptor:
    """W3C TraceContext HTTP 传播"""

    def inject_trace_context(self, headers, span):
        """注入 traceparent 和 tracestate 到 HTTP 请求头"""
        # traceparent 格式: version-trace_id-span_id-trace_flags
        # 例: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
        traceparent = f"00-{span.context.trace_id:032x}-{span.context.span_id:016x}-01"
        headers["traceparent"] = traceparent

        # tracestate: 业务自定义键值对
        # 例: acme=env:prod,rookie=true
        tracestate = f"acme=env:{self.env},tenant={span.attributes.get('tenant_id', '')}"
        headers["tracestate"] = tracestate

    def extract_trace_context(self, headers):
        """从 HTTP 响应头提取上下文，恢复为 SpanContext"""
        traceparent = headers.get("traceparent")
        if not traceparent:
            return None  # 无上游 trace → 创建新 trace

        # 解析 traceparent: version-trace_id-span_id-flags
        parts = traceparent.split("-")
        version, trace_id, span_id, flags = parts[0], parts[1], parts[2], parts[3]

        # 创建链接关系（不是父子，是因果关联）
        parent_context = SpanContext(
            trace_id=int(trace_id, 16),
            span_id=int(span_id, 16),
            trace_flags=int(flags, 16),
            is_remote=True
        )
        return parent_context

    def propagate_to_kafka(self, topic, message, span):
        """Kafka 消息传播：将 trace context 放入 Kafka Header"""
        headers = {
            "traceparent": f"00-{span.context.trace_id:032x}-{span.context.span_id:016x}-01",
            "tracestate": f"source={self.service_name}",
        }
        # 消费端读取这些 Header → 创建 link 而非 child span
        self.kafka_produce(topic, message, kafka_headers=headers)
```

### 告警关联引擎

```python
class AlertCorrelationEngine:
    """告警关联：将相关告警分组，定位根因"""

    def correlate(self, alerts):
        """将告警按 trace_id / service / time_window 分组"""
        groups = defaultdict(list)

        for alert in alerts:
            # 分组维度 1: 同 trace_id → 同一次请求的所有告警
            if alert.trace_id:
                groups[("trace", alert.trace_id)].append(alert)
            # 分组维度 2: 同 service + 5分钟窗口
            else:
                window = alert.timestamp.replace(minute=alert.timestamp.minute // 5 * 5)
                groups[("service_time", alert.service, window)].append(alert)

        correlated_results = []
        for key, group_alerts in groups.items():
            # 去重：同一告警可能来自 Prometheus + Grafana + PagerDuty
            deduped = self._deduplicate(group_alerts)

            # 定位根因：在 trace 中找 error_rate 最高的 service
            root_cause = self._find_root_cause(deduped)

            correlated_results.append({
                "group_key": key,
                "alerts": deduped,
                "alert_count": len(deduped),
                "root_cause_service": root_cause.service,
                "root_cause_reason": root_cause.reason,
                "suggested_action": self._suggest_action(root_cause),
                "severity": max(a.severity for a in deduped)
            })

        return sorted(correlated_results, key=lambda r: r["severity"], reverse=True)

    def _find_root_cause(self, alerts):
        """根因定位：error_rate 最高的 service 是根因"""
        service_errors = defaultdict(int)
        for alert in alerts:
            service_errors[alert.service] += 1

        # 错误数最多的 service = 根因
        root_service = max(service_errors, key=service_errors.get)
        root_alert = next(a for a in alerts if a.service == root_service)
        return root_alert

    def _deduplicate(self, alerts):
        """去重：不同告警系统发送的相同告警"""
        seen = set()
        deduped = []
        for alert in alerts:
            key = (alert.service, alert.alert_name, alert.severity)
            if key not in seen:
                seen.add(key)
                deduped.append(alert)
        return deduped

    def _suggest_action(self, root_cause):
        """根据根因类型建议处理动作"""
        action_map = {
            "high_error_rate": "检查该服务近期部署变更",
            "high_latency": "检查数据库慢查询和缓存命中率",
            "service_down": "检查健康检查日志和资源使用",
            "connection_pool_exhausted": "检查连接泄漏和池大小配置",
        }
        return action_map.get(root_cause.alert_type, "人工排查")
```

### Metric 基数管理

```python
class CardinalityManager:
    """管理 Prometheus metric 基数，防止爆炸"""

    def scan_high_cardinality(self):
        """扫描所有 metric，找出高基数 label"""
        results = []
        for metric in self.prometheus.get_all_metrics():
            for label_name in metric.label_names:
                unique_values = self.prometheus.count_unique_values(metric.name, label_name)
                if unique_values > 1000:
                    results.append({
                        "metric": metric.name,
                        "label": label_name,
                        "unique_values": unique_values,
                        "risk": "HIGH" if unique_values > 10000 else "MEDIUM",
                        "memory_impact_mb": unique_values * 3 / 1024  # 每个series ~3KB
                    })

        return sorted(results, key=lambda r: r["unique_values"], reverse=True)

    def auto_aggregate(self, metric_name, label_name, strategy):
        """自动聚合高基数 label"""
        if strategy == "drop":
            # 直接丢弃该 label
            return f"metric_without_{label_name}"
        elif strategy == "top_n":
            # 只保留前 N 个最常见的值，其余归为 "other"
            return f"keep top 10 values, rest → 'other'"
        elif strategy == "bucket":
            # 将值分桶：user_id → user_tier (free/pro/enterprise)
            return f"map {label_name} to tier-based label"

### OpenTelemetry 完整集成代码

```python
from opentelemetry import trace, context, baggage
from opentelemetry.propagate import inject, extract

class TraceContextPropagator:
    """W3C TraceContext 传播器"""

    def inject_to_http(self, headers, span):
        """HTTP 请求注入 trace context"""
        ctx = trace.set_span_in_context(span)
        # 自动注入 traceparent + tracestate
        inject(headers, context=ctx)
        # 注入 baggage（业务上下文）
        ctx = baggage.set_baggage("tenant_id", span.attributes.get("tenant_id"))
        ctx = baggage.set_baggage("user_tier", span.attributes.get("user_tier", "free"))
        inject(headers, context=ctx)

    def extract_from_http(self, headers):
        """HTTP 响应提取 trace context"""
        ctx = extract(headers)
        return ctx

    def inject_to_kafka(self, topic, message, span):
        """Kafka 消息注入 trace context"""
        headers = {}
        ctx = trace.set_span_in_context(span)
        inject(headers, context=ctx)
        # Kafka Header 格式: list of (key, value) tuples
        kafka_headers = [(k, v.encode()) for k, v in headers.items()]
        self.producer.send(topic, message, headers=kafka_headers)

    def extract_from_kafka(self, kafka_record):
        """Kafka 消息提取 trace context，创建 link"""
        headers = {k: v.decode() for k, v in kafka_record.headers()}
        parent_ctx = extract(headers)
        # 消费端创建 link（关联生产者 span），而非 child
        parent_span_ctx = trace.get_current_span(parent_ctx).get_span_context()
        return parent_span_ctx
```

### 告警关联引擎完整实现

```python
class AlertCorrelationEngine:
    """告警关联引擎：分组、去重、定位根因"""

    def correlate(self, alerts):
        """将告警按 trace_id / service / time_window 分组"""
        groups = defaultdict(list)
        for alert in alerts:
            if alert.trace_id:
                groups[("trace", alert.trace_id)].append(alert)
            else:
                window = alert.timestamp.replace(
                    minute=alert.timestamp.minute // 5 * 5, second=0)
                groups[("service_time", alert.service, window)].append(alert)

        results = []
        for key, group_alerts in groups.items():
            deduped = self._deduplicate(group_alerts)
            root_cause = self._find_root_cause(deduped)
            results.append({
                "group_key": key,
                "alert_count": len(deduped),
                "root_cause_service": root_cause.service,
                "root_cause_reason": root_cause.reason,
                "suggested_action": self._suggest_action(root_cause),
                "severity": max(a.severity for a in deduped)
            })

        return sorted(results, key=lambda r: r["severity"], reverse=True)

    def _find_root_cause(self, alerts):
        """根因定位：错误率最高的 service"""
        error_counts = defaultdict(int)
        for alert in alerts:
            error_counts[alert.service] += 1
        root_service = max(error_counts, key=error_counts.get)
        return next(a for a in alerts if a.service == root_service)

    def _deduplicate(self, alerts):
        """去重：不同告警系统发送的相同告警"""
        seen = set()
        deduped = []
        for alert in alerts:
            key = (alert.service, alert.alert_name, alert.severity)
            if key not in seen:
                seen.add(key)
                deduped.append(alert)
        return deduped

    def _suggest_action(self, root_cause):
        action_map = {
            "high_error_rate": "检查该服务近期部署变更",
            "high_latency": "检查数据库慢查询和缓存命中率",
            "service_down": "检查健康检查日志和资源使用",
            "connection_pool_exhausted": "检查连接泄漏和池大小配置",
        }
        return action_map.get(root_cause.alert_type, "人工排查")
```

### Metric 基数管理完整实现

```python
class CardinalityManager:
    """Prometheus Metric 基数管理"""

    def scan_high_cardinality(self):
        """扫描高基数 label"""
        results = []
        for metric in self.prometheus.get_all_metrics():
            for label_name in metric.label_names:
                unique_values = self.prometheus.count_unique_values(
                    metric.name, label_name)
                if unique_values > 1000:
                    results.append({
                        "metric": metric.name,
                        "label": label_name,
                        "unique_values": unique_values,
                        "risk": "HIGH" if unique_values > 10000 else "MEDIUM",
                        "memory_impact_mb": unique_values * 3 / 1024,
                        "suggestion": self._suggest_aggregation(label_name, unique_values)
                    })
        return sorted(results, key=lambda r: r["unique_values"], reverse=True)

    def _suggest_aggregation(self, label_name, unique_values):
        """建议聚合策略"""
        if label_name in ("user_id", "request_id", "session_id"):
            return "DROP: 高基数唯一标识，不应作为 metric label，移至 Trace 系统"
        elif label_name in ("ip_address",):
            return "BUCKET: 按 IP 段聚合 (10.0.x.x → 10.0.0.0/16)"
        elif unique_values < 100:
            return "KEEP: 基数可控，保留"
        else:
            return "TOP_N: 只保留前 20 个值，其余归入 'other'"

    def auto_aggregate(self, metric_name, label_name, strategy="drop"):
        """自动聚合高基数 label"""
        if strategy == "drop":
            # 修改 metric 定义，移除该 label
            self.prometheus.relabel_config(metric_name, action="drop", label=label_name)
        elif strategy == "top_n":
            self.prometheus.relabel_config(metric_name, action="keep_top_n",
                label=label_name, n=20, replacement="other")
        elif strategy == "bucket":
            # 按 IP 段聚合
            self.prometheus.relabel_config(metric_name, action="replace",
                label=label_name, regex=r"(\d+\.\d+)\..*", replacement=r"\1.0.0/16")
```

## SLO/SLI 框架完整实现

```python
class SLOManager:
    """SLO 管理：定义 SLI、计算错误预算、告警"""

    SLI_DEFINITIONS = {
        "availability": {
            "query": "sum(rate(http_requests_total{status=~'2xx'}[5m])) / sum(rate(http_requests_total[5m]))",
            "target": 0.999,  # 99.9% 可用性
            "window_days": 30
        },
        "latency_p99": {
            "query": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "target": 0.5,  # P99 < 500ms
            "window_days": 30
        },
        "error_rate": {
            "query": "sum(rate(http_requests_total{status=~'5xx'}[5m])) / sum(rate(http_requests_total[5m]))",
            "target": 0.001,  # 错误率 < 0.1%
            "window_days": 30
        }
    }

    def calculate_error_budget(self, sli_name):
        """计算错误预算"""
        sli = self.SLI_DEFINITIONS[sli_name]
        target = sli["target"]
        window_days = sli["window_days"]

        # 30 天总请求量
        total_requests = self.prometheus.query(
            f"sum(increase(http_requests_total[{window_days}d]))")

        # 允许的错误数 = 总请求 × (1 - target)
        if sli_name == "availability":
            allowed_errors = total_requests * (1 - target)
            actual_errors = self.prometheus.query(
                f"sum(increase(http_requests_total{{status=~'5xx'}}[{window_days}d]))")
            budget_remaining = allowed_errors - actual_errors
            budget_consumed_pct = (actual_errors / allowed_errors) * 100
        elif sli_name == "latency_p99":
            allowed_slow = total_requests * (1 - target)
            actual_slow = self.prometheus.query(
                f"sum(increase(http_request_duration_seconds_count{{le>0.5}}[{window_days}d]))")
            budget_remaining = allowed_slow - actual_slow
            budget_consumed_pct = (actual_slow / allowed_slow) * 100

        return {
            "sli": sli_name,
            "target": target,
            "total_requests": total_requests,
            "budget_remaining": max(0, budget_remaining),
            "budget_consumed_pct": min(100, budget_consumed_pct),
            "is_healthy": budget_consumed_pct < 100
        }

    def check_burn_rate(self, sli_name):
        """检查错误预算消耗速率"""
        # Fast burn: 2% budget in 1 hour → 立即告警
        # Slow burn: 5% budget in 6 hours → 工单告警
        fast_burn_rate = self.prometheus.query(
            f"sum(rate(http_requests_total{{status=~'5xx'}}[1h])) / "
            f"sum(rate(http_requests_total[1h]))")
        slow_burn_rate = self.prometheus.query(
            f"sum(rate(http_requests_total{{status=~'5xx'}}[6h])) / "
            f"sum(rate(http_requests_total[6h]))")

        sli = self.SLI_DEFINITIONS[sli_name]
        error_budget_per_hour = (1 - sli["target"]) / (sli["window_days"] * 24)

        if fast_burn_rate > error_budget_per_hour * 14.4:
            return {"alert_level": "PAGE", "reason": "Fast burn: 2% budget in 1h",
                    "burn_rate": fast_burn_rate}
        elif slow_burn_rate > error_budget_per_hour * 6:
            return {"alert_level": "TICKET", "reason": "Slow burn: 5% budget in 6h",
                    "burn_rate": slow_burn_rate}
        return {"alert_level": "OK", "burn_rate": slow_burn_rate}
```

## 自适应采样实现

```python
class AdaptiveSampler:
    """自适应采样：根据流量特征动态调整采样率"""

    def __init__(self):
        self.base_rate = 0.01  # 基础采样率 1%
        self.service_rates = {}  # per-service 采样率

    def should_sample(self, span):
        """决定是否采样该 span"""
        service = span.attributes.get("service.name")
        rate = self.service_rates.get(service, self.base_rate)

        # 优先采样规则：始终采样错误和慢请求
        if span.status.code == StatusCode.ERROR:
            return True  # 100% 采样错误
        if span.attributes.get("http.status_code", 200) >= 500:
            return True  # 100% 采样 5xx
        if span.duration_ms > 3000:
            return True  # 100% 采样超慢请求（>3s）

        # 随机采样
        return random.random() < rate

    def adjust_rates(self):
        """每分钟根据指标调整采样率"""
        for service in self.get_all_services():
            error_rate = self.prometheus.query(
                f"sum(rate(http_requests_total{{service='{service}',status=~'5xx'}}[5m])) / "
                f"sum(rate(http_requests_total{{service='{service}'}}[5m]))")
            p99_latency = self.prometheus.query(
                f"histogram_quantile(0.99, "
                f"sum(rate(http_request_duration_seconds_bucket{{service='{service}'}}[5m])) by (le))")

            if error_rate > 0.01:  # 错误率 > 1%
                self.service_rates[service] = 1.0  # 100% 采样
            elif error_rate > 0.001:  # 错误率 > 0.1%
                self.service_rates[service] = 0.5  # 50% 采样
            elif p99_latency > 2.0:  # P99 > 2s
                self.service_rates[service] = 0.1  # 10% 采样
            else:
                self.service_rates[service] = self.base_rate  # 正常：1% 采样
```

## 分布式日志聚合

```python
class StructuredLogger:
    """结构化日志：每条日志携带 trace_id"""

    def log(self, level, message, **context):
        span = trace.get_current_span()
        log_entry = {
            "timestamp": now().isoformat(),
            "level": level,
            "service": self.service_name,
            "trace_id": format(span.context.trace_id, "032x") if span else None,
            "span_id": format(span.context.span_id, "016x") if span else None,
            "message": message,
            "context": context
        }
        self.kafka_producer.send("logs", json.dumps(log_entry))

class LogQueryService:
    """日志查询服务：按 trace_id 聚合全链路日志"""

    def query_by_trace(self, trace_id):
        """查询某次请求的全链路日志"""
        logs = self.clickhouse.query(
            "SELECT * FROM logs WHERE trace_id = %s ORDER BY timestamp", trace_id)
        return {
            "trace_id": trace_id,
            "log_count": len(logs),
            "services": list(set(l["service"] for l in logs)),
            "timeline": [{
                "timestamp": l["timestamp"],
                "service": l["service"],
                "level": l["level"],
                "message": l["message"]
            } for l in logs]
        }
```

## 异常场景补充

### 场景：指标管道延迟导致过期仪表盘

```
触发：Flink 聚合延迟 30 分钟 → Grafana 显示的数据是 30 分钟前的
检测：
  1. 数据新鲜度检查：最新数据时间戳 vs 当前时间 > 10 分钟 → 告警
  2. 仪表盘显示"数据新鲜度：30 分钟前"警告横幅
处理：
  1. 仪表盘自动显示数据新鲜度时间戳
  2. 延迟 > 10 分钟 → 显示黄色警告
  3. 延迟 > 30 分钟 → 显示红色警告 + 禁止基于此数据做决策
  4. Flink 恢复后 → 自动移除警告
预防：所有仪表盘都显示数据新鲜度指标
```

### 场景：采样丢失关键错误 span

```
触发：服务 A 错误率 0.5% → 自适应采样 50% → 关键错误 span 被丢弃
      → 排查时看不到完整的错误链路
检测：
  1. 自适应采样优先保证错误 span 100% 采样
  2. 如果错误 span 仍然丢失 → 通过日志中的 trace_id 补全
  3. 补全逻辑：从 ClickHouse 日志中提取 trace_id → 查询 span → 关联
处理：
  1. 自适应采样规则：ERROR span → 始终采样
  2. 5xx 响应 → 始终采样
  3. 超慢请求（>3s）→ 始终采样
预防：错误和慢请求永远不被采样丢弃
```

### 场景：日志量暴增压垮存储

```
触发：某服务 DEBUG 日志未关闭 → 日志量 10x 暴增
      → ClickHouse 写入跟不上 → 查询变慢 → 存储成本暴增
检测：
  1. 日志量监控：日增量 > 昨日 3 倍 → 告警
  2. ClickHouse 写入延迟 > 5 分钟 → 告警
处理：
  1. 自动降级：将 DEBUG 日志级别过滤掉
  2. 压缩历史日志：将 7 天前的日志从 LZ4 压缩为 ZSTD
  3. 缩短保留期：从 30 天缩短到 14 天
  4. 通知相关服务负责人检查日志级别
预防：生产环境默认 INFO 级别 + 日志量监控
```

## Grafana 仪表盘 JSON 配置

```json
{
  "dashboard": {
    "title": "Service Overview",
    "templating": {
      "list": [
        {"name": "service", "type": "query", "query": "label_values(http_requests_total, service)"},
        {"name": "environment", "type": "query", "query": "label_values(http_requests_total, environment)"}
      ]
    },
    "panels": [
      {
        "title": "Request Rate",
        "type": "timeseries",
        "targets": [{"expr": "sum(rate(http_requests_total{service=\"$service\",environment=\"$environment\"}[5m])) by (status_code)"}],
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0}
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "targets": [{"expr": "sum(rate(http_requests_total{service=\"$service\",status_code=~\"5..\"}[5m])) / sum(rate(http_requests_total{service=\"$service\"}[5m]))"}],
        "thresholds": [{"value": 0.001, "color": "green"}, {"value": 0.01, "color": "yellow"}, {"value": 0.05, "color": "red"}],
        "gridPos": {"h": 4, "w": 6, "x": 12, "y": 0}
      },
      {
        "title": "P50/P95/P99 Latency",
        "type": "timeseries",
        "targets": [
          {"expr": "histogram_quantile(0.5, sum(rate(http_request_duration_seconds_bucket{service=\"$service\"}[5m])) by (le))", "legendFormat": "P50"},
          {"expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{service=\"$service\"}[5m])) by (le))", "legendFormat": "P95"},
          {"expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{service=\"$service\"}[5m])) by (le))", "legendFormat": "P99"}
        ],
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 4}
      },
      {
        "title": "SLO Budget Remaining",
        "type": "gauge",
        "targets": [{"expr": "100 - (sum(rate(http_requests_total{service=\"$service\",status_code=~\"5..\"}[30d])) / sum(rate(http_requests_total{service=\"$service\"}[30d])) / 0.001 * 100)"}],
        "thresholds": [{"value": 20, "color": "red"}, {"value": 50, "color": "yellow"}, {"value": 100, "color": "green"}],
        "gridPos": {"h": 4, "w": 6, "x": 18, "y": 0}
      }
    ]
  }
}
```

## 分布式日志聚合完整实现

```python
class StructuredLogger:
    """结构化日志：每条日志携带 trace_id"""
    def log(self, level, message, **context):
        span = trace.get_current_span()
        log_entry = {
            "timestamp": now().isoformat(),
            "level": level,
            "service": self.service_name,
            "trace_id": format(span.context.trace_id, "032x") if span else None,
            "span_id": format(span.context.span_id, "016x") if span else None,
            "message": message,
            "context": context
        }
        self.kafka_producer.send("structured_logs", json.dumps(log_entry))

class LogQueryService:
    """按 trace_id 聚合全链路日志"""
    def query_by_trace(self, trace_id):
        logs = self.clickhouse.query(
            "SELECT * FROM structured_logs WHERE trace_id = %s ORDER BY timestamp", trace_id)
        return {
            "trace_id": trace_id,
            "log_count": len(logs),
            "services": list(set(l["service"] for l in logs)),
            "timeline": [{"timestamp": l["timestamp"], "service": l["service"],
                          "level": l["level"], "message": l["message"]} for l in logs]
        }
```

## 异常场景补充

### 场景：指标管道延迟

```
触发：Flink 聚合延迟 30 分钟 → 仪表盘显示过时数据
检测：数据新鲜度 = 最新数据时间戳 vs 当前时间 > 10 分钟 → 告警
处理：
  1. 仪表盘显示"数据延迟中"警告横幅
  2. 延迟 > 30 分钟 → 禁止基于此数据做决策
  3. Flink 恢复后 → 自动移除警告
预防：所有仪表盘都显示数据新鲜度时间戳
```

### 场景：采样丢失关键错误 span

```
触发：1% 采样率恰好丢弃了导致 500 错误的 span → 无法排查
检测：日志中有错误 trace_id 但 ClickHouse 无对应 span
处理：
  1. 自适应采样：错误 span → 100% 采样
  2. 如果仍缺失 → 通过日志 trace_id 补全链路
预防：错误、5xx、超慢请求始终 100% 采样
```

### 场景：日志量暴增

```
触发：某服务 DEBUG 日志未关 → 日志量 10x 暴增 → ClickHouse 写入跟不上
检测：日增量 > 昨日 3 倍 → 告警
处理：
  1. 自动降级：过滤 DEBUG 级别日志
  2. 压缩历史日志（LZ4 → ZSTD）
  3. 缩短保留期（30 天 → 14 天）
预防：生产环境默认 INFO 级别 + 日志量监控
```

## 指标管道完整实现

### 场景：采样丢失关键错误 span 的补救

```python
class TraceRescueService:
    """从日志中补救丢失的 trace span"""
    def rescue_missing_spans(self, trace_id):
        """通过日志 trace_id 重建缺失的 span"""
        logs = self.clickhouse.query(
            "SELECT * FROM structured_logs WHERE trace_id = %s ORDER BY timestamp", trace_id)
        if not logs:
            return None

        # 从日志重建 span
        reconstructed_spans = []
        for log in logs:
            span = {
                "trace_id": trace_id,
                "span_id": hashlib.md5(f"{trace_id}:{log['timestamp']}".encode()).hexdigest()[:16],
                "operation_name": f"{log['service']}.{log.get('operation', 'unknown')}",
                "start_time": log["timestamp"],
                "tags": {"source": "log_rescue", "level": log["level"]},
                "logs": [{"timestamp": log["timestamp"], "message": log["message"]}]
            }
            reconstructed_spans.append(span)

        # 写入 ClickHouse trace 表
        for span in reconstructed_spans:
            self.clickhouse.insert("traces_rescued", span)

        return {"trace_id": trace_id, "rescued_spans": len(reconstructed_spans)}
```

## 性能分析详细数据

**Trace 存储规模：**

| 规模 | 日均 Span | 月存储(原始) | ClickHouse 压缩 | 月成本 |
|------|----------|-------------|-----------------|-------|
| 小型 | 1亿 | 200GB | 20GB | ¥0.5 万 |
| 中型 | 10亿 | 2TB | 200GB | ¥3 万 |
| 大型 | 100亿 | 20TB | 2TB | ¥15 万 |

**采样率对成本影响：**

| 采样率 | 月存储 | 发现问题能力 | 推荐场景 |
|--------|-------|------------|---------|
| 100% | 200GB | 最佳 | 核心支付服务 |
| 10% | 20GB | 良好 | 一般业务服务 |
| 1% | 2GB | 可接受 | 边缘服务 |
| 自适应 | 动态 | 最优 | **推荐方案** |

**查询延迟基准：**

| 查询类型 | ClickHouse | Elasticsearch |
|---------|-----------|--------------|
| 单 trace 查询 | <100ms | <200ms |
| 服务拓扑图 | 2-5s | 5-10s |
| P99 延迟趋势 | <1s | 2-3s |

**月度成本：**

| 组件 | 规格 | 月成本 |
|------|------|-------|
| OTEL Collector | 4c8G × 5台 | ¥2 万 |
| Kafka (Trace) | 8c32G × 6节点 | ¥3 万 |
| ClickHouse | 8c64G + SSD × 3节点 | ¥5 万 |
| Prometheus | 8c32G + SSD × 3节点 | ¥3 万 |
| Grafana | 4c8G × 2台 | ¥0.5 万 |
| **合计** | | **¥13.5 万** |

## 服务网格可观测性集成

```python
class ServiceMeshObserver:
    """Istio/Envoy 服务网格可观测性"""

    def generate_dependency_map(self):
        """从 Trace 数据自动生成服务依赖图"""
        spans = self.clickhouse.query(
            "SELECT service_name, peer_service, count() as call_count, "
            "avg(duration_ms) as avg_latency, "
            "countIf(status_code >= 500) as error_count "
            "FROM traces WHERE timestamp > now() - INTERVAL 1 HOUR "
            "GROUP BY service_name, peer_service")

        nodes = set()
        edges = []
        for span in spans:
            nodes.add(span["service_name"])
            nodes.add(span["peer_service"])
            edges.append({
                "source": span["service_name"],
                "target": span["peer_service"],
                "call_count": span["call_count"],
                "avg_latency_ms": span["avg_latency"],
                "error_rate": span["error_count"] / span["call_count"]
            })

        return {"nodes": list(nodes), "edges": edges}
```

## 告警规则即代码

```yaml
# Prometheus 告警规则完整定义
groups:
  - name: slo_alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          / sum(rate(http_requests_total[5m])) by (service) > 0.01
        for: 2m
        labels:
          severity: page
          team: "{{ $labels.service }}"
        annotations:
          summary: "{{ $labels.service }} error rate > 1%"
          runbook: "https://wiki/runbook/high-error-rate"

      - alert: SLOBurnRateFast
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[1h])) by (service)
          / sum(rate(http_requests_total[1h])) by (service)
          > (1 - 0.999) * 14.4
        for: 5m
        labels:
          severity: page
        annotations:
          summary: "{{ $labels.service }} SLO fast burn detected"

      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))
          > 2.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.service }} P99 latency > 2s"

      - alert: ConnectionPoolExhausted
        expr: |
          db_connection_pool_active / db_connection_pool_max > 0.9
        for: 3m
        labels:
          severity: page
```

## 可观测性成本优化

```python
class ObservabilityCostOptimizer:
    """可观测性成本优化"""

    def optimize_trace_storage(self):
        """优化 Trace 存储"""
        savings = []

        # 1. 自适应采样：错误 100%，正常 1%
        current_rate = self.get_current_sampling_rate()
        if current_rate == 1.0:
            savings.append({
                "measure": "启用自适应采样",
                "current_storage": "2TB/month",
                "optimized_storage": "200GB/month",
                "savings": "90%"
            })

        # 2. Trace TTL：30 天 → 7 天热 + 90 天冷
        savings.append({
            "measure": "分层存储",
            "hot_storage": "7 天 (SSD)",
            "cold_storage": "90 天 (HDD/S3)",
            "savings": "60% 存储成本"
        })

        # 3. 尾部采样：保留所有错误 trace，只采样正常 trace
        savings.append({
            "measure": "尾部采样（保留错误）",
            "kept_traces": "100% 错误 + 1% 正常",
            "savings": "85% 存储"
        })

        return savings
```

## 异常场景补充

### 场景：Metric Label 爆炸

```python
class LabelExplosionDetector:
    """检测 Metric Label 值爆炸"""
    def check(self):
        metrics = self.prometheus.get_all_metrics()
        alerts = []
        for metric in metrics:
            for label in metric.label_names:
                cardinality = self.prometheus.cardinality(metric.name, label)
                if cardinality > 10000:
                    alerts.append({
                        "metric": metric.name, "label": label,
                        "cardinality": cardinality,
                        "memory_mb": cardinality * 3 / 1024,
                        "action": "DROP 或聚合该 label"
                    })
        return alerts
```

### 场景：Trace Collector 瓶颈

```
触发：OTEL Collector 处理能力不足 → Span 丢失
      → 链路不完整 → 排查困难
检测：
  1. Collector 队列深度 > 10000 → 告警
  2. Span 丢弃率 > 0 → 告警
  3. Collector CPU > 80% → 扩容
处理：
  1. 自动扩容 Collector（从 5 台扩展到 10 台）
  2. 启用背压：降级采样率从 10% → 1%
  3. 紧急时启用文件缓冲（Span 先写磁盘，后异步发送）
预防：Collector 水平扩展 + 队列深度监控
```

## Tail-Based 采样完整实现

```python
class TailBasedSampler:
    """尾部采样：先收集完整 trace，事后决定是否保留"""

    def should_keep_trace(self, trace):
        """根据 trace 结果决定是否保留"""
        # 1. 始终保留错误 trace
        if any(span.status_code == StatusCode.ERROR for span in trace.spans):
            return True

        # 2. 始终保留慢 trace（P99 以上）
        max_duration = max(span.duration_ms for span in trace.spans)
        p99_threshold = self.get_p99_threshold(trace.service)
        if max_duration > p99_threshold:
            return True

        # 3. 始终保留 5xx 响应
        if any(span.http_status >= 500 for span in trace.spans):
            return True

        # 4. 正常 trace → 按概率采样
        return random.random() < self.base_sample_rate

    def get_p99_threshold(self, service):
        """动态获取 P99 阈值"""
        cached = self.redis.get(f"p99_threshold:{service}")
        if cached:
            return float(cached)
        # 从 Prometheus 查询
        p99 = self.prometheus.query(
            f"histogram_quantile(0.99, "
            f"sum(rate(http_request_duration_seconds_bucket{{service='{service}'}}[5m])) by (le))")
        self.redis.setex(f"p99_threshold:{service}", 300, p99)
        return p99
```

## 可观测性成熟度模型

| 级别 | 名称 | Trace | Metric | Log | 告警 | 覆盖率 |
|------|------|-------|--------|-----|------|--------|
| L0 | 无 | 无 | 无 | 文件 | 无 | 0% |
| L1 | 基础 | 部分 | 系统指标 | 集中 | 手动 | 30% |
| L2 | 标准 | 全链路 | 业务指标 | 结构化 | 规则化 | 60% |
| L3 | 高级 | 采样+关联 | SLI/SLO | 关联 | SLO驱动 | 80% |
| L4 | 智能 | 自适应 | 预测 | AI分析 | 自愈 | 95% |

**当前目标：L3（SLO 驱动告警 + 自适应采样 + Trace-Log-Metric 关联）**

## 性能分析详细数据

**月度基础设施成本：**

| 组件 | 规格 | 数量 | 月成本 |
|------|------|------|-------|
| OTEL Collector | 4c8G | 5 台 | ¥2 万 |
| Kafka (Trace) | 8c32G + 2TB | 6 节点 | ¥3 万 |
| ClickHouse | 8c64G + 5TB SSD | 3 节点 | ¥5 万 |
| Prometheus | 8c32G + 2TB SSD | 3 节点 | ¥3 万 |
| Grafana | 4c8G | 2 台 | ¥0.5 万 |
| Alertmanager | 2c4G | 3 台 | ¥0.3 万 |
| **合计** | | | **¥13.8 万** |

**采样率对成本的影响：**

| 采样策略 | Trace 存储/月 | 发现问题能力 | 月成本 |
|---------|-------------|------------|-------|
| 100% 全量 | 2TB | 最佳 | ¥15 万 |
| 10% 固定 | 200GB | 良好 | ¥8 万 |
| 1% 固定 | 20GB | 可接受 | ¥5 万 |
| 自适应 | 动态(50-200GB) | 最优 | ¥6 万 |
| 尾部采样 | 动态(100-300GB) | 优秀 | ¥7 万 |

## OTEL Collector 完整配置

```yaml
# OpenTelemetry Collector 完整配置
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
        max_receive_message_size: 4MB
      http:
        endpoint: 0.0.0.0:4318
  prometheus:
    config:
      scrape_configs:
        - job_name: 'services'
          scrape_interval: 15s
          static_configs:
            - targets: ['service-a:9090', 'service-b:9090']
  kafka:
    brokers: ['kafka:9092']
    topics: ['structured-logs']

processors:
  batch:
    send_batch_size: 1024
    timeout: 5s
  filter:
    error_mode: ignore
    traces:
      span:
        - 'attributes["http.route"] == "/health"'
        - 'attributes["http.route"] == "/metrics"'
  tail_sampling:
    decision_wait: 10s
    sampling_table:
      - name: "errors"
        policy: "status_code == ERROR"
        sampling_rate: 1.0
      - name: "slow_requests"
        policy: "duration_ms > 3000"
        sampling_rate: 1.0
      - name: "normal"
        policy: "default"
        sampling_rate: 0.01
  transform:
    trace_statements:
      - context: span
        statements:
          - 'set(attributes["environment"], "${ENV}")'
          - 'set(attributes["deployment.version"], "${VERSION}")'

exporters:
  clickhouse:
    endpoint: tcp://ch:9000
    database: observability
    timeout: 5s
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: "otel"
  elasticsearch:
    endpoint: http://es:9200
    logs_index: "otel-logs"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, filter, tail_sampling, transform]
      exporters: [clickhouse]
    metrics:
      receivers: [otlp, prometheus]
      processors: [batch, transform]
      exporters: [prometheus]
    logs:
      receivers: [otlp, kafka]
      processors: [batch]
      exporters: [elasticsearch]
```

## SLO 仪表盘完整规格

```python
class SLODashboard:
    """SLO 仪表盘规格"""

    PANELS = [
        {
            "title": "Error Budget Remaining",
            "type": "gauge",
            "query": "100 - (sum(rate(http_requests_total{status=~'5..'}[30d])) "
                     "/ sum(rate(http_requests_total[30d])) / 0.001 * 100)",
            "thresholds": {"red": 20, "yellow": 50, "green": 100}
        },
        {
            "title": "Burn Rate (1h window)",
            "type": "stat",
            "query": "sum(rate(http_requests_total{status=~'5..'}[1h])) "
                     "/ sum(rate(http_requests_total[1h])) "
                     "/ (1 - 0.999) * 24",
            "thresholds": {"page": 14.4, "ticket": 6}
        },
        {
            "title": "Burn Rate (6h window)",
            "type": "stat",
            "query": "sum(rate(http_requests_total{status=~'5..'}[6h])) "
                     "/ sum(rate(http_requests_total[6h])) "
                     "/ (1 - 0.999) * 24",
            "thresholds": {"page": 6, "ticket": 3}
        },
        {
            "title": "SLO Compliance Timeline",
            "type": "timeseries",
            "query": "sum(rate(http_requests_total{status!~'5..'}[1d])) "
                     "/ sum(rate(http_requests_total[1d]))",
            "legend": "30d rolling availability"
        },
        {
            "title": "P99 Latency vs SLO Target",
            "type": "timeseries",
            "query": "histogram_quantile(0.99, "
                     "sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "threshold_line": 0.5  # SLO target: 500ms
        }
    ]
```

## 异常场景补充

### 场景：Collector 管道死锁

```
触发：OTEL Collector 处理管道中 batch processor 累积大量数据
      → 内存溢出 → Collector 崩溃 → 所有 Trace/Metric 断流
检测：
  1. Collector 内存 > 80% → 告警
  2. Collector 进程退出 → 严重告警
  3. Trace 接收速率归零 → 服务降级告警
处理：
  1. 自动重启 Collector
  2. 减小 batch size（1024 → 256）
  3. 启用背压：降低接收速率
  4. 临时切换到直接写入（绕过 Collector）
预防：内存监控 + batch size 合理配置 + 水平扩展
```

### 场景：SLO 预算耗尽流程

```
触发：30 天错误预算已耗尽 → 所有变更需审批
检测：
  1. Error Budget Remaining < 0 → 告警
  2. 自动进入"变更冻结"模式
处理：
  1. 所有非紧急部署暂停
  2. 只允许修复错误的变更
  3. 错误预算重置：下一个 30 天周期开始时恢复
  4. 紧急变更需 CTO 审批
预防：提前设置错误预算告警（剩余 < 20%）
```

### 场景：告警关联误报

```
触发：3 个服务同时告警 → 关联引擎判定为同一根因
      → 但实际上是 3 个独立问题
检测：
  1. 关联后修复只解决了 1 个问题 → 其他仍然告警
  2. 关联准确率统计：< 70% → 引擎需要改进
处理：
  1. 人工标记误关联 → 更新关联规则
  2. 增加关联维度：不只是 trace_id，还需看 error_type
  3. 降低自动关联的置信度阈值
预防：关联引擎准确率监控 + 人工校准
```

## OpenTelemetry Collector 完整配置与路由

OpenTelemetry Collector 是可观测性数据管道的核心枢纽，负责接收、处理、导出三种信号（Trace/Metric/Log）。以下配置覆盖了生产环境的完整需求：多协议接收、智能采样、批量处理、数据路由和多云导出。

```yaml
# ==================== OpenTelemetry Collector 完整配置 ====================
# 部署模式：DaemonSet（每个 K8s 节点一个 Collector 实例）
# 版本：OTEL Collector Contrib v0.96.0

extensions:
  health_check:
    endpoint: 0.0.0.0:13133
  pprof:
    endpoint: 0.0.0.0:1777
  zpages:
    endpoint: 0.0.0.0:55679

receivers:
  # ---- OTLP 接收器：接收 SDK 发送的 Trace 和 Metric ----
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
        max_receive_message_size: 4MiB
        max_concurrent_streams: 1000
        read_buffer_size: 512KiB
        write_buffer_size: 32KiB
        tls_settings:
          cert_file: /etc/otel/certs/server.crt
          key_file: /etc/otel/certs/server.key
        auth:
          authenticator: basicauth
      http:
        endpoint: 0.0.0.0:4318
        max_request_body_size: 4MiB
        cors:
          allowed_origins: ["https://grafana.internal", "https://jaeger.internal"]
          allowed_headers: ["Content-Type", "Authorization"]

  # ---- Prometheus 接收器：拉取 K8s 基础设施指标 ----
  prometheus:
    config:
      global:
        scrape_interval: 15s
        evaluation_interval: 15s
        scrape_timeout: 10s
      scrape_configs:
        - job_name: 'kubernetes-pods'
          kubernetes_sd_configs:
            - role: pod
          relabel_configs:
            - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
              action: keep
              regex: true
            - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
              action: replace
              target_label: __metrics_path__
              regex: (.+)
            - source_labels: [__meta_kubernetes_namespace]
              action: replace
              target_label: namespace
            - source_labels: [__meta_kubernetes_pod_name]
              action: replace
              target_label: pod
        - job_name: 'kubernetes-nodes'
          kubernetes_sd_configs:
            - role: node
          relabel_configs:
            - source_labels: [__meta_kubernetes_node_name]
              action: replace
              target_label: node

  # ---- Kafka 接收器：接收结构化日志 ----
  kafka:
    brokers: ['kafka-1:9092', 'kafka-2:9092', 'kafka-3:9092']
    topics: ['structured-logs', 'app-events']
    consumer_group: 'otel-collector-logs'
    auth:
      tls:
        ca_file: /etc/otel/certs/kafka-ca.crt
        cert_file: /etc/otel/certs/kafka-client.crt
        key_file: /etc/otel/certs/kafka-client.key
    metadata:
      retry_max: 3
      retry_backoff: 250ms
    encoding: otlp_json

processors:
  # ---- 批量处理器：减少网络请求次数 ----
  batch:
    send_batch_size: 1024
    send_batch_max_size: 2048
    timeout: 5s

  # ---- 过滤处理器：丢弃健康检查等无用 Span ----
  filter:
    error_mode: ignore
    traces:
      span:
        - 'attributes["http.route"] == "/health"'
        - 'attributes["http.route"] == "/metrics"'
        - 'attributes["http.route"] == "/ready"'
        - 'attributes["http.route"] == "/live"'
        - 'name == "probe"'

  # ---- 尾部采样：基于 Trace 完整结果决定是否保留 ----
  tail_sampling:
    decision_wait: 10s
    num_traces: 100000
    expected_new_traces_per_sec: 100
    sampling_policies:
      - name: "error-traces"
        type: status_code
        status_code:
          status_codes:
            - ERROR
      - name: "slow-traces"
        type: latency
        latency:
          threshold_ms: 3000
      - name: "critical-endpoints"
        type: string_attribute
        string_attribute:
          key: "http.route"
          values:
            - "/api/v1/payments"
            - "/api/v1/orders"
            - "/api/v1/refunds"
      - name: "sample-normal"
        type: probabilistic
        probabilistic:
          sampling_percentage: 1

  # ---- 变换处理器：统一添加环境和版本标签 ----
  transform:
    error_mode: ignore
    trace_statements:
      - context: resource
        statements:
          - 'set(attributes["deployment.environment"], "${DEPLOY_ENV}")'
          - 'set(attributes["service.version"], "${SERVICE_VERSION}")'
          - 'set(attributes["collector.version"], "0.96.0")'
      - context: span
        statements:
          - 'set(attributes["environment"], "${DEPLOY_ENV}")'
          - 'set(attributes["deployment.version"], "${SERVICE_VERSION}")'
          - 'delete_key(attributes, "http.request.header.authorization")'
          - 'delete_key(attributes, "http.request.header.cookie")'
    metric_statements:
      - context: resource
        statements:
          - 'set(attributes["deployment.environment"], "${DEPLOY_ENV}")'
    log_statements:
      - context: resource
        statements:
          - 'set(attributes["deployment.environment"], "${DEPLOY_ENV}")'

  # ---- 内存限制处理器 ----
  memory_limiter:
    check_interval: 1s
    limit_mib: 1024
    spike_limit_mib: 256

  # ---- 属性处理器 ----
  attributes:
    actions:
      - key: http.route
        value: "${http.route}"
        action: upsert

exporters:
  # ---- ClickHouse 导出器：Trace 数据持久化 ----
  clickhouse:
    endpoint: tcp://clickhouse-01:9000
    database: observability
    username: otel_writer
    password: ${CLICKHOUSE_PASSWORD}
    ttl_days: 30
    timeout: 5s
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s
      max_elapsed_time: 300s
    sending_queue:
      enabled: true
      num_consumers: 10
      queue_size: 5000
    traces_table_name: otel_traces
    spans_table_name: otel_spans

  # ---- Prometheus 导出器：Metric 数据暴露 ----
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: "otel"
    const_labels:
      collector: daemonset
    metric_expiration: 5m
    resource_to_telemetry_conversion:
      enabled: true

  # ---- Elasticsearch 导出器：日志数据持久化 ----
  elasticsearch:
    endpoint: http://elasticsearch-01:9200
    logs_index: "otel-logs-%{yyyy.MM.dd}"
    traces_index: "otel-traces-%{yyyy.MM.dd}"
    num_workers: 4
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s
      max_elapsed_time: 300s
    sending_queue:
      enabled: true
      num_consumers: 10
      queue_size: 5000
    ilm:
      policy_name: "otel-logs-policy"
      rollover_alias: "otel-logs"
      min_size: "50gb"
      min_index_age: "1d"

  # ---- 调试导出器（开发环境）----
  debug:
    verbosity: basic
    sampling_initial: 5
    sampling_thereafter: 200

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, filter, tail_sampling, transform, batch]
      exporters: [clickhouse, debug]
    metrics:
      receivers: [otlp, prometheus]
      processors: [memory_limiter, transform, batch]
      exporters: [prometheus]
    logs:
      receivers: [otlp, kafka]
      processors: [memory_limiter, transform, batch]
      exporters: [elasticsearch]

  extensions: [health_check, pprof, zpages]

  telemetry:
    logs:
      level: info
      development: false
      encoding: json
      output_paths: ["stdout"]
      error_output_paths: ["stderr"]
    metrics:
      level: detailed
      address: 0.0.0.0:8888
```

**Collector 性能基线：**

| 配置项 | 值 | 说明 |
|-------|---|------|
| 单实例接收吞吐 | 5000 span/s | OTLP gRPC |
| 内存使用 | 500-1024 MiB | 含批量缓冲 |
| CPU 使用 | 0.5-1.5 核 | 处理+序列化 |
| 端到端延迟 | < 2s | SDK 到 Collector 到后端 |
| 批量发送间隔 | 5s | 平衡延迟与吞吐 |

## SLI/SLO 仪表盘完整规格

SLO 仪表盘是可观测性的"北极星指标"——它不关注单个服务的状态，而是关注用户感受到的整体服务质量。以下规格定义了完整的 SLO 仪表盘面板布局、查询表达式和告警阈值映射。

```python
class SLODashboardSpec:
    """
    SLI/SLO 仪表盘完整规格。
    包含错误预算、燃烧率、合规时间线三大核心面板，
    以及告警阈值到仪表盘注释的映射。
    """

    DASHBOARD_TITLE = "SLO 合规监控"
    REFRESH_INTERVAL = "30s"
    TIME_RANGE = "now-30d to now"

    SLO_DEFINITIONS = {
        "availability": {
            "name": "服务可用性",
            "target": 0.999,
            "window_days": 30,
            "error_budget_pct": 0.1,
            "sli_query": (
                "sum(rate(http_requests_total{status=~'2xx',environment='production'}[5m])) "
                "/ sum(rate(http_requests_total{environment='production'}[5m]))"
            ),
        },
        "latency_p99": {
            "name": "P99 延迟",
            "target": 0.5,
            "window_days": 30,
            "error_budget_pct": 0.1,
            "sli_query": (
                "histogram_quantile(0.99, "
                "sum(rate(http_request_duration_seconds_bucket{environment='production'}[5m])) by (le))"
            ),
        },
        "error_rate": {
            "name": "错误率",
            "target": 0.001,
            "window_days": 30,
            "error_budget_pct": 0.1,
            "sli_query": (
                "sum(rate(http_requests_total{status=~'5..',environment='production'}[5m])) "
                "/ sum(rate(http_requests_total{environment='production'}[5m]))"
            ),
        },
    }

    PANELS = [
        # 面板 1：错误预算剩余量（Gauge）
        {
            "id": "error_budget_remaining",
            "title": "错误预算剩余量",
            "type": "gauge",
            "gridPos": {"h": 8, "w": 6, "x": 0, "y": 0},
            "targets": [
                {
                    "expr": (
                        "100 * (1 - "
                        "sum(increase(http_requests_total{status=~'5..',environment='production'}[30d])) "
                        "/ sum(increase(http_requests_total{environment='production'}[30d])) "
                        "/ 0.001)"
                    ),
                    "legendFormat": "可用性预算剩余%",
                },
            ],
            "thresholds": {
                "steps": [
                    {"value": None, "color": "green"},
                    {"value": 50, "color": "yellow"},
                    {"value": 20, "color": "red"},
                ],
            },
        },

        # 面板 2：燃烧率图表（Time Series，1h/6h/3d 三条线）
        {
            "id": "burn_rate",
            "title": "错误预算燃烧率",
            "type": "timeseries",
            "gridPos": {"h": 8, "w": 12, "x": 6, "y": 0},
            "targets": [
                {
                    "expr": (
                        "sum(rate(http_requests_total{status=~'5..',environment='production'}[1h])) "
                        "/ sum(rate(http_requests_total{environment='production'}[1h])) "
                        "/ (1 - 0.999) * 24"
                    ),
                    "legendFormat": "1h 燃烧率",
                },
                {
                    "expr": (
                        "sum(rate(http_requests_total{status=~'5..',environment='production'}[6h])) "
                        "/ sum(rate(http_requests_total{environment='production'}[6h])) "
                        "/ (1 - 0.999) * 24"
                    ),
                    "legendFormat": "6h 燃烧率",
                },
                {
                    "expr": (
                        "sum(rate(http_requests_total{status=~'5..',environment='production'}[3d])) "
                        "/ sum(rate(http_requests_total{environment='production'}[3d])) "
                        "/ (1 - 0.999) * 24"
                    ),
                    "legendFormat": "3d 燃烧率",
                },
            ],
            "annotations": [
                {"value": 14.4, "label": "PAGE 阈值（1h窗口）", "color": "red"},
                {"value": 6.0, "label": "TICKET 阈值（6h窗口）", "color": "orange"},
                {"value": 1.0, "label": "正常燃烧速率", "color": "green"},
            ],
        },

        # 面板 3：SLO 合规时间线（30 天滚动）
        {
            "id": "slo_compliance_timeline",
            "title": "SLO 合规时间线（30 天滚动）",
            "type": "timeseries",
            "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0},
            "targets": [
                {
                    "expr": (
                        "sum(rate(http_requests_total{status!~'5..',environment='production'}[1d])) "
                        "/ sum(rate(http_requests_total{environment='production'}[1d]))"
                    ),
                    "legendFormat": "可用性（30d 滚动）",
                },
            ],
            "annotations": [
                {"value": 0.999, "label": "可用性目标 99.9%", "color": "green"},
                {"value": 0.99, "label": "可用性警告 99%", "color": "yellow"},
            ],
        },

        # 面板 4：告警阈值映射表
        {
            "id": "alert_threshold_table",
            "title": "告警阈值与仪表盘映射",
            "type": "table",
            "gridPos": {"h": 8, "w": 12, "x": 0, "y": 8},
            "data": [
                {"slo": "可用性 99.9%", "window": "1h",
                 "burn_rate_threshold": 14.4, "alert_level": "PAGE",
                 "dashboard_panel": "燃烧率 -> 红色线", "action": "立即响应"},
                {"slo": "可用性 99.9%", "window": "6h",
                 "burn_rate_threshold": 6.0, "alert_level": "TICKET",
                 "dashboard_panel": "燃烧率 -> 橙色线", "action": "4小时内处理"},
                {"slo": "可用性 99.9%", "window": "3d",
                 "burn_rate_threshold": 1.0, "alert_level": "INFO",
                 "dashboard_panel": "燃烧率 -> 绿色线", "action": "关注"},
                {"slo": "P99 < 500ms", "window": "1h",
                 "burn_rate_threshold": 14.4, "alert_level": "PAGE",
                 "dashboard_panel": "延迟燃烧率", "action": "立即响应"},
            ],
        },

        # 面板 5：每日错误预算消耗量（Bar Chart）
        {
            "id": "budget_consumption_trend",
            "title": "每日错误预算消耗量",
            "type": "barchart",
            "gridPos": {"h": 8, "w": 12, "x": 12, "y": 8},
            "targets": [
                {
                    "expr": (
                        "sum(increase(http_requests_total{status=~'5..',environment='production'}[1d])) "
                        "/ (sum(increase(http_requests_total{environment='production'}[30d])) * 0.001)"
                        " * 100"
                    ),
                    "legendFormat": "每日预算消耗%",
                },
            ],
            "thresholds": {
                "steps": [
                    {"value": None, "color": "green"},
                    {"value": 3.3, "color": "yellow"},
                    {"value": 10, "color": "red"},
                ],
            },
        },
    ]

    ALERT_DASHBOARD_MAPPING = {
        "SLOBurnRateFast": {
            "promql": (
                "sum(rate(http_requests_total{status=~'5..'}[1h])) "
                "/ sum(rate(http_requests_total[1h])) "
                "> (1 - 0.999) * 14.4"
            ),
            "dashboard_link": "/d/slo-dashboard?viewPanel=burn_rate",
            "annotation_text": "1h 燃烧率超过 14.4x，预算将在 2.5 天内耗尽",
            "severity": "PAGE",
        },
        "SLOBurnRateSlow": {
            "promql": (
                "sum(rate(http_requests_total{status=~'5..'}[6h])) "
                "/ sum(rate(http_requests_total[6h])) "
                "> (1 - 0.999) * 6"
            ),
            "dashboard_link": "/d/slo-dashboard?viewPanel=burn_rate",
            "annotation_text": "6h 燃烧率超过 6x，预算将在 5 天内耗尽",
            "severity": "TICKET",
        },
        "SLOBudgetExhausted": {
            "promql": (
                "100 - (sum(rate(http_requests_total{status!~'5..'}[30d])) "
                "/ sum(rate(http_requests_total[30d])) / 0.999 * 100) < 0"
            ),
            "dashboard_link": "/d/slo-dashboard?viewPanel=error_budget_remaining",
            "annotation_text": "错误预算已耗尽，进入变更冻结模式",
            "severity": "PAGE",
        },
    }
```

## 事故关联工作流完整实现

从告警触发到根因定位的完整自动化工作流，包括：告警到 Trace 到 Log 到根因的关联链路，自动事故时间线生成，以及历史相似事故搜索。

```python
class IncidentCorrelationWorkflow:
    """
    事故关联工作流：从告警自动推导到根因。
    核心路径：告警 → Trace 定位 → Log 细节 → 根因分析 → 时间线生成。
    """

    def __init__(self, trace_store, log_store, metric_store, db):
        self.trace_store = trace_store
        self.log_store = log_store
        self.metric_store = metric_store
        self.db = db

    def correlate_alert_to_root_cause(self, alert: dict) -> dict:
        """
        从告警出发，自动关联 Trace 和 Log，定位根因。
        步骤：告警 → 查错误Trace → 定位最深错误Span → 查ERROR日志 → 推断根因
        """
        service_name = alert["service_name"]
        fired_at = alert["fired_at"]
        window_start = fired_at - timedelta(minutes=5)
        window_end = fired_at + timedelta(minutes=5)

        # Step 1：查找错误 Trace
        error_traces = self.trace_store.query("""
            SELECT trace_id, span_id, operation_name, duration_ms,
                   status_code, parent_span_id, service_name
            FROM otel_traces
            WHERE service_name = %(service)s
              AND status_code = 'ERROR'
              AND start_time BETWEEN %(start)s AND %(end)s
            ORDER BY duration_ms DESC LIMIT 10
        """, {"service": service_name, "start": window_start, "end": window_end})

        if not error_traces:
            return {
                "status": "no_error_traces_found",
                "service": service_name,
                "suggestion": "检查是否为指标误报或非 Trace 覆盖的组件故障",
            }

        # Step 2：定位最深层的错误 Span
        root_spans = []
        for trace in error_traces:
            full_trace = self.trace_store.get_trace(trace["trace_id"])
            deepest_error = self._find_deepest_error_span(full_trace)
            if deepest_error:
                root_spans.append({
                    "trace_id": trace["trace_id"],
                    "root_cause_service": deepest_error["service_name"],
                    "root_cause_operation": deepest_error["operation_name"],
                    "duration_ms": deepest_error["duration_ms"],
                    "error_message": deepest_error.get("status_message", ""),
                })

        # Step 3：用 TraceID 查询 ERROR 日志
        trace_ids = [s["trace_id"] for s in root_spans]
        error_logs = self.log_store.search({
            "query": {
                "bool": {
                    "must": [
                        {"terms": {"trace_id": trace_ids}},
                        {"term": {"level": "ERROR"}},
                    ]
                }
            },
            "size": 50,
            "sort": [{"timestamp": "asc"}],
        })

        # Step 4：从日志中提取根因信息
        root_causes = []
        for log in error_logs:
            cause = self._extract_root_cause_from_log(log)
            if cause:
                root_causes.append(cause)

        root_cause_summary = self._summarize_root_causes(root_causes)

        # Step 5：生成事故时间线
        timeline = self._generate_timeline(
            alert, error_traces, root_spans, error_logs
        )

        # Step 6：搜索历史相似事故
        similar_incidents = self._search_similar_incidents(
            service_name, root_cause_summary
        )

        # Step 7：组装完整结果
        incident = {
            "alert_id": alert.get("alert_id"),
            "service_name": service_name,
            "fired_at": fired_at.isoformat(),
            "root_cause": root_cause_summary,
            "error_trace_count": len(error_traces),
            "error_log_count": len(error_logs),
            "affected_traces": trace_ids[:5],
            "timeline": timeline,
            "similar_incidents": similar_incidents,
            "recommended_actions": self._recommend_actions(root_cause_summary),
            "correlation_confidence": self._calculate_confidence(
                error_traces, error_logs, root_causes
            ),
        }

        self.db.insert("incidents", {
            "alert_id": alert.get("alert_id"),
            "service_name": service_name,
            "root_cause_summary": json.dumps(root_cause_summary, ensure_ascii=False),
            "timeline": json.dumps(timeline, ensure_ascii=False),
            "correlation_confidence": incident["correlation_confidence"],
            "created_at": now(),
        })

        return incident

    def _find_deepest_error_span(self, trace: dict) -> dict:
        """在 Trace 树中找到最深层的错误 Span（根因 Span）"""
        spans = trace.get("spans", [])
        error_spans = [s for s in spans if s.get("status_code") == "ERROR"]
        if not error_spans:
            return None

        def span_depth(span):
            depth = 0
            current = span
            while current.get("parent_span_id"):
                parent = next(
                    (s for s in spans if s["span_id"] == current["parent_span_id"]),
                    None
                )
                if not parent:
                    break
                depth += 1
                current = parent
            return depth

        return max(error_spans, key=span_depth)

    def _extract_root_cause_from_log(self, log: dict) -> dict:
        """从错误日志中提取根因信息，基于模式匹配"""
        message = log.get("message", "")
        cause = {
            "trace_id": log.get("trace_id"),
            "service": log.get("service"),
            "timestamp": log.get("timestamp"),
            "message": message[:500],
        }

        patterns = {
            "timeout": r"(?i)(timeout|timed?\s*out|deadline\s*exceeded)",
            "connection_refused": r"(?i)(connection\s*refused|ECONNREFUSED)",
            "dns_failure": r"(?i)(dns|ENOTFOUND|getaddrinfo)",
            "oom": r"(?i)(out\s*of\s*memory|OOM|Cannot allocate memory)",
            "disk_full": r"(?i)(no\s*space\s*left|disk\s*full|ENOSPC)",
            "auth_failure": r"(?i)(unauthorized|authentication\s*failed|401|403)",
            "rate_limit": r"(?i)(rate\s*limit|too\s*many\s*requests|429)",
            "dependency_error": r"(?i)(upstream|downstream|dependency|external)",
        }

        import re
        for cause_type, pattern in patterns.items():
            if re.search(pattern, message):
                cause["type"] = cause_type
                break
        else:
            cause["type"] = "unknown"

        return cause

    def _summarize_root_causes(self, causes: list) -> dict:
        """汇总根因：按类型分组统计"""
        by_type = defaultdict(list)
        for c in causes:
            by_type[c["type"]].append(c)

        primary_type = max(by_type, key=lambda k: len(by_type[k])) if by_type else "unknown"
        primary_causes = by_type.get(primary_type, [])

        return {
            "primary_type": primary_type,
            "primary_count": len(primary_causes),
            "affected_services": list(set(c["service"] for c in primary_causes)),
            "sample_message": primary_causes[0]["message"] if primary_causes else "无日志",
            "all_types": {k: len(v) for k, v in by_type.items()},
        }

    def _generate_timeline(self, alert, error_traces, root_spans, error_logs) -> list:
        """自动生成事故时间线"""
        events = []

        events.append({
            "timestamp": alert["fired_at"].isoformat(),
            "type": "alert_fired",
            "description": f"告警触发: {alert.get('alert_name', 'unknown')}",
            "service": alert["service_name"],
        })

        if error_traces:
            first_error = min(error_traces, key=lambda t: t.get("start_time", now()))
            events.append({
                "timestamp": str(first_error.get("start_time", "")),
                "type": "first_error_trace",
                "description": f"首个错误 Trace: {first_error.get('operation_name', '')}",
                "trace_id": first_error["trace_id"],
            })

        for span in root_spans:
            events.append({
                "timestamp": span.get("timestamp", ""),
                "type": "root_cause_span",
                "description": f"根因: {span['root_cause_service']}.{span['root_cause_operation']}",
                "trace_id": span["trace_id"],
            })

        for log in error_logs[:5]:
            events.append({
                "timestamp": log.get("timestamp", ""),
                "type": "error_log",
                "description": f"[{log.get('service', '')}] {log.get('message', '')[:100]}",
                "trace_id": log.get("trace_id"),
            })

        events.sort(key=lambda e: e.get("timestamp", ""))
        return events

    def _search_similar_incidents(self, service_name: str, root_cause: dict) -> list:
        """从历史事故中搜索相似事故"""
        cause_type = root_cause.get("primary_type", "unknown")
        similar = self.db.query(
            "SELECT * FROM incidents "
            "WHERE service_name = %s "
            "AND root_cause_summary LIKE %s "
            "AND created_at > NOW() - INTERVAL 90 DAY "
            "ORDER BY created_at DESC LIMIT 5",
            [service_name, f"%{cause_type}%"]
        )
        return [
            {
                "incident_id": inc["id"],
                "created_at": inc["created_at"].isoformat(),
                "root_cause_summary": inc["root_cause_summary"],
                "resolution": inc.get("resolution", "未记录"),
            }
            for inc in similar
        ]

    def _recommend_actions(self, root_cause: dict) -> list:
        """根据根因类型推荐处理动作"""
        action_map = {
            "timeout": ["检查依赖服务健康状态", "查看连接池是否耗尽", "考虑增加超时或重试"],
            "connection_refused": ["检查目标服务是否运行", "检查网络策略和防火墙", "检查DNS解析"],
            "oom": ["检查内存泄漏", "增加Pod内存限制", "考虑添加自动扩缩容"],
            "rate_limit": ["联系API供应商提高限额", "添加请求限流", "实现退避策略"],
            "dependency_error": ["检查外部依赖状态", "启用断路器", "考虑降级方案"],
        }
        return action_map.get(
            root_cause.get("primary_type", "unknown"),
            ["人工排查：查看Trace和日志详情"]
        )

    def _calculate_confidence(self, error_traces, error_logs, root_causes) -> float:
        """计算关联置信度（0-1）"""
        score = 0.0
        if error_traces:
            score += 0.3
        if error_logs:
            score += 0.3
        trace_ids = set(t["trace_id"] for t in error_traces)
        log_trace_ids = set(
            l.get("trace_id") for l in error_logs if l.get("trace_id")
        )
        if trace_ids & log_trace_ids:
            score += 0.2
        if root_causes and any(c.get("type") != "unknown" for c in root_causes):
            score += 0.2
        return min(score, 1.0)
```

**事故关联示例（35 秒完成根因定位）：**

```
14:00:00  告警触发: 订单服务 5xx 率 > 5%
14:00:05  查询错误 Trace → 发现 3 条错误 Trace
14:00:10  定位根因 Span: RiskControlService → ComplianceAPI.verify
14:00:15  查询 ERROR 日志: "Compliance API timeout after 10s"
14:00:20  根因类型: timeout (依赖服务超时)
14:00:25  生成时间线: 告警 → Trace → Log → 根因
14:00:30  搜索相似事故: 发现 30 天前类似事故（外部API超时）
14:00:35  推荐动作: 启用断路器 / 降级方案 / 联系API供应商

总耗时: 35 秒（对比之前 40 分钟人工排查）
```

### 更多异常场景

#### 场景：Collector 管道死锁

```
触发：OTEL Collector 处理管道中 batch processor 累积大量数据，
      内存持续增长直到 OOM，Collector 崩溃后重启又再次 OOM，
      形成死循环。所有 Trace/Metric/Log 断流。
检测：
  1. Collector 内存使用率 > 80% → 告警
  2. Collector 进程 OOMKilled → K8s 自动重启 → 严重告警
  3. Trace 接收速率归零 → 服务降级告警
  4. batch processor 队列深度 > 5000 → 告警
根因分析：
  - 通常是因为下游 Exporter（如 ClickHouse）写入变慢
  - batch processor 持续累积未发送的数据
  - memory_limiter 处理器未及时拒绝新数据（配置阈值过高）
处理：
  1. 紧急：减小 batch size（1024 → 256），缩短超时（5s → 2s）
  2. 临时：启用背压机制，降低接收速率（在 LB 层限流）
  3. 降级：临时切换到直接写入（绕过 Collector，SDK 直连后端）
  4. 排查：检查 ClickHouse 写入延迟，是否需要扩容
  5. 恢复：ClickHouse 恢复后，逐步恢复 Collector 正常配置
预防：
  - memory_limiter 设置为系统内存的 50%
  - batch size 根据下游写入能力设置
  - 水平扩展 Collector 实例数
  - 监控 Collector 自身指标（队列深度、发送延迟）
```

#### 场景：SLO 预算耗尽流程

```
触发：30 天错误预算已耗尽（剩余 < 0%），意味着 SLO 违规。
      此时所有部署变更必须经过额外审批。
检测：
  1. Error Budget Remaining < 20% → 警告告警
  2. Error Budget Remaining < 0% → PAGE 告警
  3. 仪表盘错误预算 Gauge 变红
  4. 燃烧率持续 > 1x（预算消耗速度快于恢复速度）
处理：
  1. 自动进入"变更冻结"模式：
     - 所有非紧急部署暂停
     - CI/CD 流水线增加审批门禁
     - 只允许修复错误的变更通过
  2. SRE 团队优先处理：
     - 分析预算消耗原因（是突发事件还是持续退化？）
     - 如为突发事件 → 修复后预算自然恢复
     - 如为持续退化 → 需要系统性改进
  3. 错误预算重置：
     - 30 天滚动窗口自动重置（不依赖人工）
     - 预算恢复后自动解除变更冻结
  4. 紧急变更流程：
     - 安全修复：SRE 负责人审批即可
     - 功能变更：需 CTO 审批
预防：
  - 提前设置多级告警（剩余 < 50% / 20% / 0%）
  - 燃烧率监控（1h/6h/3d 多窗口）
  - 每周 SLO 审查会议
```

#### 场景：告警关联误报（假阳性）

```
触发：3 个服务同时告警 → 关联引擎判定为同一根因（拓扑关联）
      → 合并为一条通知 → 但实际上 3 个问题是独立发生的：
      - 订单服务：数据库慢查询
      - 支付服务：外部 API 超时
      - 风控服务：内存泄漏导致 OOM
检测：
  1. 关联后修复只解决了 1 个问题 → 其他 2 个仍然告警
  2. 关联引擎准确率统计：< 70% → 引擎需要调优
  3. 运维反馈：标记"误关联"事件 → 积累数据改进模型
根因分析：
  - 拓扑关联只考虑了服务依赖，未考虑错误类型
  - 时间窗口关联过于宽松（5 分钟内所有告警都被关联）
  - 缺少"错误类型正交性"检查
处理：
  1. 短期：人工标记误关联 → 拆分为独立告警通知
  2. 中期：增加关联维度：不仅看 trace_id，还需看 error_type
  3. 长期：降低自动关联的置信度阈值
     - 置信度 < 0.8 时不自动合并，只建议关联
     - 人工确认后才真正合并
预防：
  - 关联引擎准确率持续监控
  - 定期用历史事故做回测
  - 人工校准关联规则（每周 review）
```

## 指标告警收敛完整实现

```python
class AlertCorrelationService:
    """告警收敛：避免告警风暴"""

    def process_alert(self, alert):
        """处理告警：去重、分组、抑制"""
        # 1. 去重：相同告警 5 分钟内只发一次
        dedup_key = f"alert_dedup:{alert['service']}:{alert['alert_name']}"
        if self.redis.exists(dedup_key):
            return {"action": "suppressed", "reason": "duplicate"}
        self.redis.setex(dedup_key, 300, "1")

        # 2. 分组：相同服务的告警合并
        group_key = f"alert_group:{alert['service']}"
        self.redis.lpush(group_key, json.dumps(alert))
        self.redis.expire(group_key, 600)

        # 3. 抑制：低优先级告警被高优先级抑制
        if self._is_suppressed(alert):
            return {"action": "suppressed", "reason": "higher_severity_exists"}

        # 4. 发送告警
        self.notification.send(alert)
        return {"action": "sent", "alert_id": alert["id"]}

    def _is_suppressed(self, alert):
        """检查告警是否应被抑制"""
        # 如果已有同服务 CRITICAL 告警 → 抑制 WARNING
        if alert["severity"] == "WARNING":
            critical_exists = self.redis.exists(
                f"alert_active:{alert['service']}:CRITICAL")
            return bool(critical_exists)
        return False
```

## 容量规划模型

```python
class ObservabilityCapacityPlanner:
    """可观测性容量规划"""

    def plan(self, current_metrics, growth_rate_monthly=0.1):
        """根据增长率为未来 6 个月做容量规划"""
        months = 6
        plan = []
        for m in range(1, months + 1):
            scale = (1 + growth_rate_monthly) ** m
            plan.append({
                "month": m,
                "trace_volume_gb": current_metrics["trace_gb_month"] * scale,
                "metric_series": int(current_metrics["metric_series"] * scale),
                "log_volume_gb": current_metrics["log_gb_month"] * scale,
                "clickhouse_nodes": max(3, int(current_metrics["ch_nodes"] * scale)),
                "kafka_partitions": max(6, int(current_metrics["kafka_partitions"] * scale)),
                "estimated_cost": current_metrics["monthly_cost"] * scale,
            })
        return plan
```

## 异常场景补充

### 场景：ClickHouse 写入延迟暴增

```
触发：ClickHouse 写入延迟从 100ms 暴增到 5s
检测：
  1. 写入延迟 > 1s → 告警
  2. 消费者 lag 持续增长 → 严重告警
处理：
  1. 检查是否有大查询阻塞写入
  2. KILL 阻塞查询
  3. 增加写入批次大小（减少写入次数）
  4. 临时增加 ClickHouse 分片
预防：写入/查询资源隔离 + 大查询自动限流
```

### 场景：Prometheus 目标频繁 UP/DOWN

```
触发：服务频繁重启 → Prometheus 目标状态反复变化
      → 产生大量 false alert
检测：
  1. 同一目标 10 分钟内 UP/DOWN > 3 次 → 抖动
  2. 抖动期间暂停该目标的告警
处理：
  1. 告警规则增加 for: 2m 等待时间
  2. 抖动目标标记为 flapping → 告警降级
  3. 通知 SRE 检查服务稳定性
预防：告警增加 for 持续时间 + flapping 检测
```

## 分布式链路追踪埋点指南

### OpenTelemetry SDK 配置：Java

```java
// ============================================
// Java OpenTelemetry SDK 初始化
// ============================================
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.api.trace.propagation.W3CTraceContextPropagator;
import io.opentelemetry.context.propagation.ContextPropagators;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcTraceExporter;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;
import io.opentelemetry.sdk.trace.samplers.Sampler;
import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.semconv.ServiceAttributes;

public class OtelInitializer {

    public static OpenTelemetry initialize(String serviceName, String otelEndpoint) {
        // 资源定义：服务名 + 版本 + 环境
        Resource resource = Resource.getDefault()
            .merge(Resource.builder()
                .put(ServiceAttributes.SERVICE_NAME, serviceName)
                .put(ServiceAttributes.SERVICE_VERSION, getVersion())
                .put("deployment.environment", getEnv())
                .put("service.namespace", getNamespace())
                .build());

        // Trace Exporter：OTLP gRPC 发送到 Collector
        OtlpGrpcTraceExporter traceExporter = OtlpGrpcTraceExporter.builder()
            .setEndpoint(otelEndpoint)
            .setTimeout(5, TimeUnit.SECONDS)
            .build();

        // Tracer Provider：采样策略 + 批量导出
        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
            .setResource(resource)
            .setSampler(Sampler.parentBased(
                // 根采样：错误 100%、正常 10%
                Sampler.traceIdRatioBased(0.1)))
            .addSpanProcessor(BatchSpanProcessor.builder(traceExporter)
                .setMaxExportBatchSize(512)
                .setScheduleDelay(5, TimeUnit.SECONDS)
                .setExporterTimeout(30, TimeUnit.SECONDS)
                .build())
            // Span 限制：避免超大 Span
            .setSpanLimits(spans -> spans
                .setMaxNumberOfAttributes(128)
                .setMaxNumberOfEvents(128)
                .setMaxNumberOfLinks(32)
                .setMaxAttributeValueLength(4096))
            .build();

        return OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .setPropagators(ContextPropagators.create(
                W3CTraceContextPropagator.getInstance()))
            .buildAndRegisterGlobal();
    }
}
```

### OpenTelemetry SDK 配置：Python

```python
# ============================================
# Python OpenTelemetry SDK 初始化
# ============================================
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.sampling import ParentBasedTraceIdRatio
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource, SERVICE_NAME, SERVICE_VERSION
from opentelemetry.propagate import set_global_textmap
from opentelemetry.propagators.w3c_trace_context import W3CTraceContextPropagator

def init_telemetry(service_name: str, otel_endpoint: str):
    """初始化 OpenTelemetry"""
    resource = Resource.create({
        SERVICE_NAME: service_name,
        SERVICE_VERSION: get_version(),
        "deployment.environment": get_env(),
        "service.namespace": get_namespace(),
    })

    provider = TracerProvider(
        resource=resource,
        sampler=ParentBasedTraceIdRatio(0.1),  # 10% 采样
    )

    exporter = OTLPSpanExporter(endpoint=otel_endpoint, timeout=5)
    processor = BatchSpanProcessor(
        exporter,
        max_export_batch_size=512,
        schedule_delay_millis=5000,
        export_timeout_millis=30000,
    )
    provider.add_span_processor(processor)

    trace.set_tracer_provider(provider)
    set_global_textmap(W3CTraceContextPropagator())

    # 自动埋点：HTTP / Redis / PostgreSQL / Kafka
    from opentelemetry.instrumentation.auto_instrumentation import sitecustomize
    from opentelemetry.instrumentation.requests import RequestsInstrumentor
    from opentelemetry.instrumentation.flask import FlaskInstrumentor
    from opentelemetry.instrumentation.redis import RedisInstrumentor
    from opentelemetry.instrumentation.psycopg2 import Psycopg2Instrumentor

    RequestsInstrumentor().instrument()
    FlaskInstrumentor().instrument()
    RedisInstrumentor().instrument()
    Psycopg2Instrumentor().instrument()
```

### OpenTelemetry SDK 配置：Node.js

```javascript
// ============================================
// Node.js OpenTelemetry SDK 初始化
// ============================================
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');
const { Resource } = require('@opentelemetry/resources');
const { ATTR_SERVICE_NAME, ATTR_SERVICE_VERSION } = require('@opentelemetry/semantic-conventions');
const { ParentBasedTraceIdRatioSampler } = require('@opentelemetry/sdk-trace-base');
const { W3CTraceContextPropagator } = require('@opentelemetry/api');

const sdk = new NodeSDK({
    resource: new Resource({
        [ATTR_SERVICE_NAME]: process.env.SERVICE_NAME || 'unknown',
        [ATTR_SERVICE_VERSION]: process.env.SERVICE_VERSION || '0.0.0',
        'deployment.environment': process.env.DEPLOY_ENV || 'dev',
    }),
    traceExporter: new OTLPTraceExporter({
        url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT,
        timeoutMillis: 5000,
    }),
    sampler: new ParentBasedTraceIdRatioSampler(0.1),
    textMapPropagator: new W3CTraceContextPropagator(),
    instrumentations: [getNodeAutoInstrumentations({
        '@opentelemetry/instrumentation-fs': { enabled: false },
        '@opentelemetry/instrumentation-http': {
            ignoreIncomingPaths: ['/health', '/ready', '/metrics'],
        },
    })],
});

sdk.start();
process.on('SIGTERM', () => sdk.shutdown());
```

### 手动埋点 vs 自动埋点选择

| 维度 | 自动埋点 | 手动埋点 |
|------|---------|---------|
| 覆盖范围 | HTTP/gRPC/DB/Cache/MQ | 自定义业务逻辑 |
| 接入成本 | 低（零代码修改） | 高（需逐处添加） |
| Span 粒度 | 框架级（粗） | 方法级（细） |
| 业务属性 | 无 | 可添加自定义属性 |
| 适用阶段 | 初期快速接入 | 精细化排查阶段 |

**推荐策略：自动埋点打底 + 手动埋点补关键业务**

```
自动埋点（基础层）：
  - HTTP 请求/响应 → Span: HTTP GET /api/orders
  - gRPC 调用 → Span: OrderService/CreateOrder
  - DB 查询 → Span: SELECT * FROM orders
  - Redis 调用 → Span: GET cache:order:123
  - Kafka 消息 → Span: CONSUME topic.order-events

手动埋点（业务层）：
  - 下单流程 → Span: create_order
    - 属性: order_id, user_tier, payment_method
    - 事件: risk_check_passed, inventory_reserved
  - 支付流程 → Span: process_payment
    - 属性: payment_channel, amount, currency
    - 事件: fraud_check_started, payment_gateway_responded
```

### Context Propagation 与 Baggage

```
Context Propagation 传递链：

用户请求 → API Gateway
  │ HTTP Header: traceparent=00-abc123-xyz456-01
  │ HTTP Header: tracestate=congo=t61rcWkgMzE
  ↓
Order Service
  │ gRPC Metadata: traceparent=00-abc123-xyz456-01
  │ Baggage: user_tier=vip, region=cn-east
  ↓
Payment Service
  │ HTTP Header: traceparent=00-abc123-def789-01
  │ Baggage: user_tier=vip, region=cn-east
  ↓
Kafka Message (Header)
  │ Key: traceparent → 00-abc123-ghi012-01
  │ Key: baggage → user_tier=vip,region=cn-east
  ↓
Notification Service

关键规则：
  1. traceparent 必须在每个跨进程调用中传递（W3C 标准）
  2. Baggage 仅传必要字段（全链路可见，有性能开销）
  3. Kafka 消息通过 Header 传递 Trace Context
  4. 异步场景（MQ/定时任务）需要显式注入/提取 Context
  5. 跨线程传递：使用 Context.current().wrap(runnable)
```

## 告警 Runbook 自动化

### 告警 → Runbook 自动关联

```python
class AlertRunbookAutomation:
    """告警 Runbook 自动化：告警触发 → 自动关联 Runbook → 自动/半自动修复"""

    def __init__(self, runbook_registry, remediation_engine, escalation_policy):
        self.runbooks = runbook_registry    # Runbook 注册表
        self.engine = remediation_engine     # 自动修复引擎
        self.escalation = escalation_policy  # 升级策略

    def handle_alert(self, alert):
        """处理告警：关联 Runbook → 尝试自动修复 → 失败则升级"""
        # 1. 匹配 Runbook
        runbook = self._match_runbook(alert)
        if not runbook:
            self.escalation.notify(alert, reason="无匹配 Runbook")
            return {"action": "escalated", "reason": "no_runbook"}

        # 2. 记录关联
        self._log_runbook_association(alert["id"], runbook["id"])

        # 3. 尝试自动修复
        if runbook.get("auto_remediation"):
            result = self._execute_auto_remediation(alert, runbook)
            if result["success"]:
                return {"action": "auto_fixed", "runbook": runbook["id"]}
            else:
                # 自动修复失败 → 升级到人工
                self.escalation.notify(alert,
                    reason=f"自动修复失败: {result['error']}",
                    runbook=runbook)
                return {"action": "escalated", "reason": "auto_fix_failed"}

        # 4. 仅提供 Runbook 链接（半自动）
        self._send_runbook_link(alert, runbook)
        return {"action": "runbook_sent", "runbook": runbook["id"]}

    def _match_runbook(self, alert):
        """根据告警标签匹配 Runbook"""
        # 精确匹配：alertname + service
        key = f"{alert['labels'].get('alertname')}_{alert['labels'].get('service')}"
        runbook = self.runbooks.get(key)

        # 模糊匹配：alertname only
        if not runbook:
            key = alert['labels'].get('alertname')
            runbook = self.runbooks.get(key)

        # 通用匹配：severity 级别
        if not runbook:
            key = f"generic_{alert['severity'].lower()}"
            runbook = self.runbooks.get(key)

        return runbook

    def _execute_auto_remediation(self, alert, runbook):
        """执行自动修复动作"""
        actions = runbook["auto_remediation"]
        results = []
        for action in actions:
            try:
                result = self.engine.execute(action, context={
                    "alert": alert,
                    "service": alert["labels"].get("service"),
                    "namespace": alert["labels"].get("namespace"),
                })
                results.append({"action": action["type"], "success": True})
            except Exception as e:
                results.append({"action": action["type"], "success": False, "error": str(e)})
                # 任一动作失败 → 停止后续动作
                return {"success": False, "results": results, "error": str(e)}

        return {"success": True, "results": results}

    def _send_runbook_link(self, alert, runbook):
        """发送 Runbook 链接到告警通知渠道"""
        message = (
            f"告警: {alert['labels']['alertname']}\n"
            f"服务: {alert['labels'].get('service', 'unknown')}\n"
            f"严重度: {alert['severity']}\n"
            f"Runbook: {runbook['url']}\n"
            f"预估修复时间: {runbook.get('estimated_time', 'unknown')}\n"
            f"自动修复: {'可用' if runbook.get('auto_remediation') else '不可用'}"
        )
        self._notify(alert, message)
```

### 自动修复动作注册表

```yaml
# Runbook 示例：Pod CrashLoopBackOff
runbook:
  id: POD_CRASHLOOPBACKOFF
  alertname: PodCrashLoopBackOff
  auto_remediation:
    - type: kubectl_logs
      description: "获取 Pod 最近日志"
      command: "kubectl logs {pod_name} -n {namespace} --tail=100"
    - type: kubectl_describe
      description: "获取 Pod 事件"
      command: "kubectl describe pod {pod_name} -n {namespace}"
    - type: restart_deployment
      description: "重启 Deployment"
      command: "kubectl rollout restart deployment/{service} -n {namespace}"
      condition: "restart_count < 3"  # 防止无限重启
    - type: scale_up
      description: "扩容实例数"
      command: "kubectl scale deployment/{service} -n {namespace} --replicas={current_replicas}+1"
      condition: "current_replicas < max_replicas"
  escalation:
    - level: l1
      after: "5m"
      notify: "#sre-oncall"
    - level: l2
      after: "15m"
      notify: "#sre-managers"
  estimated_time: "5-10 分钟"
```

### 升级策略

```
告警升级流程：

告警触发
  │
  ├─ 有匹配 Runbook？
  │   ├─ 是 → 有自动修复？
  │   │   ├─ 是 → 执行自动修复
  │   │   │       ├─ 成功 → 关闭告警 + 记录
  │   │   │       └─ 失败 → 升级到 L1
  │   │   └─ 否 → 发送 Runbook 链接 → 等待人工处理
  │   │                ├─ 5 分钟内确认 → 进入处理流程
  │   │                └─ 5 分钟未确认 → 升级到 L1
  │   └─ 否 → 升级到 L1
  │
  └─ 升级路径：
      L1（5 分钟）→ SRE 值班
        ↓ 未解决
      L2（15 分钟）→ SRE 主管
        ↓ 未解决
      L3（30 分钟）→ 技术总监
        ↓ 未解决
      L4（1 小时）→ CTO + 事故指挥官

升级规则：
  - P0 告警跳过 L1，直接 L2
  - 同一服务 1 小时内重复告警 → 自动升级一级
  - 凌晨 0-6 点 → 通知方式升级（短信 → 电话）
```

## 可观测性治理

### 指标/标签/日志命名规范

```yaml
# ============================================
# 命名规范
# ============================================
metrics:
  naming_pattern: "<namespace>_<subsystem>_<unit>"
  examples:
    - "http_server_request_duration_seconds"      # 正确
    - "order_service_payment_latency_ms"          # 正确
    - "metric1"                                    # 错误：无意义
    - "orderServicePaymentLatency"                 # 错误：驼峰命名

  required_labels:
    - service          # 服务名
    - namespace        # K8s 命名空间
    - environment      # 环境标识
    - team             # 归属团队

  forbidden_labels:
    - user_id          # 高基数 → 基数爆炸
    - email            # PII 数据
    - ip_address       # 高基数 + PII
    - request_id       # 高基数

  label_value_rules:
    - rule: "标签值必须为有限枚举集"
    - rule: "枚举值数量 < 50"
    - rule: "禁止动态生成的标签值"

logs:
  structured_format: JSON
  required_fields:
    - timestamp        # ISO 8601 格式
    - level            # DEBUG/INFO/WARN/ERROR/FATAL
    - service          # 服务名
    - trace_id         # 链路追踪 ID
    - span_id          # Span ID
    - message          # 日志消息

  forbidden_content:
    - 密码、Token
    - 信用卡号
    - 身份证号
    - 完整手机号（可脱敏后记录）
```

### 标签标准与归属元数据

```python
class ObservabilityGovernance:
    """可观测性治理：命名检查、归属管理、废弃流程"""

    def __init__(self, registry_db):
        self.db = registry_db  # 指标/日志注册表

    def register_metric(self, metric_name, labels, owner_team, description):
        """注册新指标，执行命名规范检查"""
        # 1. 命名规范检查
        violations = self._check_naming_convention(metric_name, labels)
        if violations:
            return {"approved": False, "violations": violations}

        # 2. 基数检查
        cardinality = self._estimate_cardinality(labels)
        if cardinality > 10000:
            return {"approved": False,
                    "reason": f"预估基数 {cardinality} 超过阈值 10000",
                    "suggestion": "移除高基数标签或使用聚合"}

        # 3. 注册到中心注册表
        self.db.insert("metric_registry", {
            "name": metric_name,
            "labels": labels,
            "owner_team": owner_team,
            "description": description,
            "status": "active",
            "created_at": datetime.utcnow(),
            "last_used_at": datetime.utcnow(),
        })
        return {"approved": True, "metric_id": metric_name}

    def check_cardinality(self, metric_name):
        """检查指标实际基数"""
        series_count = self.db.query(
            "SELECT COUNT(DISTINCT label_hash) FROM metric_series "
            "WHERE metric_name = %s", metric_name)
        if series_count > 50000:
            self._alert_high_cardinality(metric_name, series_count)
        return {"metric": metric_name, "cardinality": series_count}

    def deprecate_stale_metrics(self, stale_threshold_days=90):
        """废弃过期指标"""
        stale = self.db.query(
            "SELECT name, owner_team, last_used_at FROM metric_registry "
            "WHERE status = 'active' AND last_used_at < %s",
            datetime.utcnow() - timedelta(days=stale_threshold_days))

        deprecation_plan = []
        for metric in stale:
            # 分阶段废弃
            deprecation_plan.append({
                "metric": metric["name"],
                "owner": metric["owner_team"],
                "phase1": "标记为 deprecated（仍可查询，仪表盘提示）",
                "phase1_duration": "30 天",
                "phase2": "停止采集（Prometheus relabel_rules drop）",
                "phase2_duration": "14 天",
                "phase3": "从注册表删除",
            })
            self.db.update("metric_registry",
                {"status": "deprecated", "deprecate_at": datetime.utcnow()},
                {"name": metric["name"]})

        return deprecation_plan

    def _check_naming_convention(self, metric_name, labels):
        """检查命名规范"""
        violations = []
        # 指标名必须为 snake_case
        if not re.match(r'^[a-z][a-z0-9_]+[a-z0-9]$', metric_name):
            violations.append(f"指标名 '{metric_name}' 不符合 snake_case 规范")
        # 必须包含单位后缀
        unit_suffixes = ("_seconds", "_bytes", "_total", "_count", "_ratio")
        if not metric_name.endswith(unit_suffixes):
            violations.append(f"指标名 '{metric_name}' 缺少单位后缀")
        # 禁止标签检查
        forbidden = {"user_id", "email", "ip_address", "request_id"}
        for label in labels:
            if label in forbidden:
                violations.append(f"标签 '{label}' 禁止使用（高基数/PII）")
        return violations

    def _estimate_cardinality(self, labels):
        """估算指标基数"""
        total = 1
        for label in labels:
            distinct_values = self.db.query(
                "SELECT COUNT(DISTINCT value) FROM label_values "
                "WHERE label_name = %s", label)
            total *= max(distinct_values, 10)  # 最小估算 10
        return total
```

## 异常场景补充

### 场景：Trace Context 跨消息队列丢失

```
触发：服务 A 通过 Kafka 发送消息，服务 B 消费后无法关联到上游 Trace
      → 链路断裂 → 无法追踪跨服务异步调用
      → 故障排查时只能看到半条链路
根因分析：
  1. Kafka Producer 未将 traceparent 注入消息 Header
  2. Kafka Consumer 未从消息 Header 提取 traceparent
  3. 消息序列化框架（如 Protobuf）丢失了非业务字段
  4. Consumer 端创建了新的 Span 但未 link 到 Producer Span
检测：
  1. 链路追踪面板中，跨 MQ 的链路出现断裂（无跨服务关联）
  2. Span 的 parent_id 为空但应有关联
  3. 统计"孤儿 Span"数量 → > 5% 则告警
处理：
  1. 立即：确认消息 Header 中 traceparent 字段是否存在
  2. 排查 Producer 端：
     - 检查 OpenTelemetry Kafka Instrumentation 是否启用
     - 确认消息发送时 Context 已注入 Header
  3. 排查 Consumer 端：
     - 检查消费时是否从 Header 提取 Context
     - 确认消费者 Span 是否 link 到生产者 Span
  4. 修复代码：确保 Producer/Consumer 均配置 OTel 自动埋点
     - Java: kafka-clients Instrumentation
     - Python: opentelemetry-instrumentation-kafka-python
     - Node: @opentelemetry/instrumentation-kafkajs
  5. 验证：发送测试消息，确认链路完整
预防：
  - 强制所有消息队列接入 OpenTelemetry Instrumentation
  - CI 流水线增加链路完整性测试（端到端 Trace 验证）
  - 监控"孤儿 Span"比例 → 链路完整性 SLI > 95%
  - 消息 Schema 强制包含 trace_context 字段
```

### 场景：Metric Cardinality 爆炸（user_id 标签）

```
触发：某服务将 user_id 作为 Prometheus 指标标签
      → 指标基数 = 服务数 × 状态码数 × 用户数(1000万) = 数十亿
      → Prometheus 内存暴增 → OOM → 监控系统不可用
      → 连锁反应：所有告警失效 → 故障无法发现
根因分析：
  1. 开发者在 http_request_duration_seconds 中添加了 user_id 标签
  2. 每个用户每次请求产生唯一的时序 → 基数爆炸
  3. Prometheus TSDB 无法高效存储超高基数指标
  4. 内存持续增长直到 OOM Killer 杀掉 Prometheus 进程
检测：
  1. Prometheus TSDB cardinality 分析：某个指标系列数 > 10 万 → 告警
  2. Prometheus 内存使用 > 80% → 严重告警
  3. Prometheus 目标频繁 UP/DOWN → OOM 重启迹象
  4. 指标查询超时 → TSDB 过载
处理：
  1. 紧急：定位高基数指标
     - 使用 Prometheus API: /api/v1/cardinality/analyze
     - 识别标签值最多的指标
  2. 止血：立即 drop 高基数标签
     - Prometheus relabel_configs: action: drop, regex: user_id
     - 重启 Prometheus 加载新配置
  3. 恢复：重启后验证内存和查询性能
  4. 根治：修改业务代码，移除 user_id 标签
     - 替代方案：用 user_tier（VIP/普通/免费）代替 user_id
     - 需要用户维度分析 → 使用 ClickHouse 日志查询
  5. 清理：删除 Prometheus 历史高基数数据（TSDB tombstone）
预防：
  - 指标注册流程强制基数评估（见可观测性治理）
  - Prometheus 配置 cardinality_limits（> 5 万自动 drop）
  - CI 流水线增加指标命名和标签检查
  - 高基数字段（user_id/order_id）禁止作为指标标签
  - 需要用户维度分析 → 使用日志/Trace 查询，不用 Metric
```

## 可观测性告警降噪完整实现

```python
class AlertNoiseReducer:
    """告警降噪：去重 + 聚合 + 抑制 + 根因分析"""

    def process_alert(self, alert):
        """处理告警（降噪管道）"""
        # 1. 去重（相同告警 5 分钟内不重复发送）
        dedup_key = f"{alert['service']}:{alert['metric']}:{alert['severity']}"
        if self.redis.exists(f"alert_dedup:{dedup_key}"):
            return {"status": "deduped"}

        # 2. 聚合（同服务多个告警合并为一条）
        related = self._find_related_alerts(alert)
        if related:
            alert["group_id"] = related[0]["group_id"]
            alert["related_count"] = len(related) + 1
        else:
            alert["group_id"] = str(uuid4())[:8]

        # 3. 抑制规则（低优先级告警在高优先级存在时抑制）
        if self._is_suppressed(alert):
            return {"status": "suppressed"}

        # 4. 频率限制（同服务每小时最多 5 条告警）
        rate_key = f"alert_rate:{alert['service']}:{now().strftime('%Y%m%d%H')}"
        rate = int(self.redis.get(rate_key) or 0)
        if rate >= 5:
            return {"status": "rate_limited"}
        self.redis.incr(rate_key)
        self.redis.expire(rate_key, 3600)

        # 5. 根因分析（尝试找到根本原因）
        root_cause = self._analyze_root_cause(alert)

        # 6. 发送告警
        self._send_alert(alert, root_cause)

        self.redis.setex(f"alert_dedup:{dedup_key}", 300, "1")
        return {"status": "sent", "group_id": alert["group_id"]}

    def _analyze_root_cause(self, alert):
        """根因分析（基于拓扑推断）"""
        # 检查上游服务是否也有告警
        upstream = self.topology.get_upstream(alert["service"])
        for svc in upstream:
            upstream_alert = self.redis.get(f"active_alert:{svc}")
            if upstream_alert:
                return {
                    "likely_root_cause": svc,
                    "reason": f"上游服务 {svc} 也有告警，可能是根因",
                    "downstream_impact": alert["service"]
                }

        # 检查基础设施层
        infra_alerts = self._check_infra_alerts(alert["service"])
        if infra_alerts:
            return {
                "likely_root_cause": "infrastructure",
                "reason": f"基础设施告警: {infra_alerts[0]['description']}"
            }

        return {"likely_root_cause": "unknown"}

    def _is_suppressed(self, alert):
        """检查抑制规则"""
        # P1 告警存在时抑制同服务的 P3/P4
        if alert["severity"] in ["P3", "P4"]:
            active_p1 = self.redis.get(
                f"active_alert:{alert['service']}:P1")
            if active_p1:
                return True
        return False
```

## SLO 管理与错误预算

```python
class SLOManagementService:
    """SLO 管理：定义 + 追踪 + 错误预算 + 燃尽率"""

    def track_slo(self, slo_id):
        """追踪 SLO 达成情况"""
        slo = self.db.get_slo(slo_id)

        # 计算当前周期内的 SLI
        period_start = self._get_period_start(slo["period"])
        sli = self._calculate_sli(slo, period_start)

        # 错误预算
        error_budget = 1 - slo["target"]
        consumed = max(0, slo["target"] - sli)
        remaining = error_budget - consumed
        burn_rate = consumed / max((now() - period_start).total_seconds() / (slo["period_days"] * 86400), 0.01)

        # 预测是否能达成
        elapsed_pct = (now() - period_start).total_seconds() / (slo["period_days"] * 86400)
        projected_sli = sli  # 假设未来与过去表现一致
        projected_achievement = projected_sli >= slo["target"]

        return {
            "slo_name": slo["name"],
            "target": slo["target"],
            "current_sli": round(sli, 4),
            "error_budget_remaining": round(remaining, 4),
            "error_budget_remaining_pct": round(remaining / error_budget * 100, 1) if error_budget > 0 else 0,
            "burn_rate": round(burn_rate, 2),
            "projected_achievement": projected_achievement,
            "status": "healthy" if remaining > 0.5 * error_budget else
                     "at_risk" if remaining > 0 else "exhausted"
        }

    def _calculate_sli(self, slo, period_start):
        """计算 SLI"""
        if slo["sli_type"] == "availability":
            total = self.db.count("requests", service=slo["service"],
                created_at__gte=period_start)
            success = self.db.count("requests", service=slo["service"],
                created_at__gte=period_start, status__in=["200", "201", "204"])
            return success / max(total, 1)

        elif slo["sli_type"] == "latency":
            p99 = self.db.query_one(
                "SELECT PERCENTILE_CONT(0.99) WITHIN GROUP "
                "(ORDER BY latency_ms) as p99 FROM requests "
                "WHERE service = %s AND created_at >= %s",
                slo["service"], period_start)["p99"] or 0
            threshold = slo["latency_threshold_ms"]
            under = self.db.count("requests", service=slo["service"],
                created_at__gte=period_start, latency_ms__lte=threshold)
            total = self.db.count("requests", service=slo["service"],
                created_at__gte=period_start)
            return under / max(total, 1)

        return 0
```

## 异常场景补充

### 场景：告警风暴

```
触发：核心服务故障 → 下游 20 个服务告警 → 100+ 条告警 → 告警风暴
检测：
  1. 1 分钟内 > 20 条告警 → 告警风暴
  2. 值班工程师被淹没 → 无法定位根因
处理：
  1. 自动聚合：基于拓扑将相关告警合并
  2. 只发送根因告警 + 影响范围摘要
  3. 抑制下游衍生告警
预防：告警聚合 + 拓扑关联 + 频率限制 + 根因标注
```

### 场景：SLO 错误预算耗尽

```
触发：月中错误预算已耗尽 → 所有变更被阻止 → 业务无法发布
检测：
  1. 错误预算剩余 < 10% → 高风险
  2. 变更冻结影响业务 → 需要权衡
处理：
  1. 紧急 SLO 审查：目标是否合理？
  2. 不可靠变更可绕过（需 SRE 审批）
  3. 修复 SLO 违规的根因 → 释放预算
预防：错误预算按比例分配 + 早期预警 + 灵活的变更策略
```

## 分布式追踪采样策略完整实现

```python
class TraceSamplingService:
    """分布式追踪采样：头部采样 + 尾部采样 + 优先采样"""

    SAMPLING_STRATEGIES = {
        "probabilistic": "概率采样（固定比率）",
        "rate_limiting": "速率限制（每秒最多 N 条）",
        "adaptive": "自适应采样（基于流量动态调整）",
        "priority": "优先采样（错误/慢请求必采）",
    }

    def should_sample(self, trace_context):
        """决定是否采样此追踪"""
        # 1. 优先采样：错误请求必采
        if trace_context.get("error"):
            return True

        # 2. 优先采样：慢请求必采（P99 阈值）
        if trace_context.get("duration_ms") > 500:
            return True

        # 3. 优先采样：关键路径必采
        critical_paths = ["payment", "order_create", "user_login"]
        if trace_context.get("operation") in critical_paths:
            return True

        # 4. 自适应概率采样
        current_qps = self._get_current_qps()
        target_samples_per_second = 10  # 目标：每秒 10 条采样

        sample_rate = min(1.0, target_samples_per_second / max(current_qps, 1))
        self.redis.set("trace:sample_rate", round(sample_rate, 4))

        # W3C TraceContext 采样标志
        trace_id = trace_context.get("trace_id", "")
        sample_val = int(trace_id[:8], 16) % 10000 / 10000
        return sample_val < sample_rate

    def adaptive_sample_rate(self):
        """自适应采样率计算"""
        # 根据流量大小动态调整
        current_qps = self._get_current_qps()

        # 目标：每秒采样 10 条 + 所有错误 + 所有慢请求
        if current_qps < 20:
            return 1.0  # 低流量 → 全量采样
        elif current_qps < 100:
            return 0.5  # 中流量 → 50%
        elif current_qps < 500:
            return 0.1  # 高流量 → 10%
        else:
            return 0.02  # 超高流量 → 2%

    def tail_sampling(self, trace_spans):
        """尾部采样（基于完整追踪结果决定是否保留）"""
        # 收集完整追踪后再决定
        has_error = any(s.get("error") for s in trace_spans)
        max_duration = max(s.get("duration_ms", 0) for s in trace_spans)

        # 保留条件
        if has_error or max_duration > 500:
            return True  # 错误或慢请求 → 保留

        # 正常请求 → 概率保留
        return random.random() < 0.05  # 5% 采样
```

## 可观测性数据管道

```python
class ObservabilityDataPipeline:
    """可观测性数据管道：收集 → 处理 → 存储 → 可视化"""

    PIPELINE_STAGES = [
        {"name": "collect", "sources": ["metrics", "logs", "traces"]},
        {"name": "process", "operations": ["parse", "enrich", "aggregate", "sample"]},
        {"name": "store", "destinations": ["prometheus", "elasticsearch", "tempo"]},
        {"name": "visualize", "tools": ["grafana", "kibana"]},
    ]

    def process_log_entry(self, raw_log):
        """处理日志条目"""
        # 1. 解析
        parsed = self._parse_log(raw_log)

        # 2. 富化（添加元数据）
        enriched = self._enrich(parsed)

        # 3. 提取指标
        metrics = self._extract_metrics(enriched)

        # 4. 写入存储
        self.elasticsearch.index("logs", enriched)
        self.prometheus.push_metrics(metrics)

        return {"parsed": parsed, "enriched": enriched, "metrics": metrics}

    def _enrich(self, parsed):
        """富化日志（添加上下文）"""
        # 服务拓扑信息
        service = parsed.get("service")
        service_meta = self.service_registry.get(service)
        if service_meta:
            parsed["team"] = service_meta["team"]
            parsed["env"] = service_meta["environment"]
            parsed["version"] = service_meta["version"]

        # 关联追踪 ID
        if parsed.get("trace_id"):
            trace = self.tracing.get_trace(parsed["trace_id"])
            if trace:
                parsed["span_count"] = trace["span_count"]
                parsed["root_service"] = trace["root_service"]

        return parsed

    def generate_dashboard(self, service_name):
        """自动生成 Grafana 看板"""
        panels = [
            {"title": "请求速率", "query": f"rate(http_requests_total{{service='{service_name}'}}[5m])"},
            {"title": "错误率", "query": f"rate(http_errors_total{{service='{service_name}'}}[5m]) / rate(http_requests_total{{service='{service_name}'}}[5m])"},
            {"title": "P50 延迟", "query": f"histogram_quantile(0.5, rate(http_latency_bucket{{service='{service_name}'}}[5m]))"},
            {"title": "P99 延迟", "query": f"histogram_quantile(0.99, rate(http_latency_bucket{{service='{service_name}'}}[5m]))"},
            {"title": "CPU 使用率", "query": f"process_cpu_seconds_total{{service='{service_name}'}}"},
            {"title": "内存使用", "query": f"process_resident_memory_bytes{{service='{service_name}'}}"},
        ]

        return {
            "dashboard": {
                "title": f"{service_name} 监控看板",
                "panels": panels,
                "refresh": "10s",
                "time_range": "last 1 hour"
            }
        }
```

## 异常场景补充

### 场景：采样策略漏采关键错误

```
触发：概率采样 5% → 关键错误请求未被采样 → 无法排查
检测：
  1. 错误率告警但追踪系统中找不到错误追踪 → 漏采
  2. 采样率过低导致关键事件丢失
处理：
  1. 优先采样：错误请求和慢请求必采（不经过概率采样）
  2. 调整采样策略：优先采样 + 概率采样
预防：错误/慢请求优先必采 + 采样策略定期审查
```

### 场景：日志管道 Elasticsearch 过载

```
触发：日志量突增 10x → ES 写入延迟 → 日志丢失 → 无法排查
检测：
  1. ES 写入队列深度 > 100000 → 过载
  2. 日志搜索延迟 > 10 秒 → 性能问题
处理：
  1. 日志采样（降低写入量）
  2. 分流：非关键日志写入冷存储
  3. ES 紭急扩容
预防：日志采样 + 冷热分流 + ES 自动扩容
```

## 可观测性数据关联完整实现

```python
class ObservabilityCorrelationService:
    """可观测性三支柱关联：Metrics ↔ Logs ↔ Traces"""

    def correlate_incident(self, alert_id):
        """关联告警的三支柱数据"""
        alert = self.db.get_alert(alert_id)

        # 1. 时间窗口
        start = alert["triggered_at"] - timedelta(minutes=5)
        end = alert["triggered_at"] + timedelta(minutes=5)

        # 2. 关联 Metrics
        metrics = self.prometheus.query_range(
            alert["service"], alert["metric_name"],
            start=start, end=end)

        # 3. 关联 Logs
        logs = self.elasticsearch.search({
            "query": {
                "bool": {
                    "must": [
                        {"term": {"service": alert["service"]}},
                        {"range": {"timestamp": {"gte": start.isoformat(),
                                                  "lte": end.isoformat()}}},
                        {"term": {"level": "ERROR"}}
                    ]
                }
            },
            "size": 50,
            "sort": [{"timestamp": "desc"}]
        })

        # 4. 关联 Traces
        traces = self.tempo.search({
            "service": alert["service"],
            "min_duration_ms": alert.get("duration_threshold_ms", 500),
            "start": start,
            "end": end,
            "limit": 20
        })

        # 5. 构建关联图
        correlation = {
            "alert_id": alert_id,
            "time_window": f"{start.isoformat()} → {end.isoformat()}",
            "metrics_spike": self._detect_metric_spike(metrics),
            "error_log_count": len(logs.get("hits", {}).get("hits", [])),
            "slow_trace_count": len(traces),
            "root_trace": self._find_root_trace(traces) if traces else None,
            "correlated_logs": self._extract_log_patterns(logs),
        }

        return correlation

    def _find_root_trace(self, traces):
        """找到根因追踪（最慢的 span 链）"""
        if not traces:
            return None

        # 找持续时间最长的 trace
        longest = max(traces, key=lambda t: t.get("duration_ms", 0))

        # 提取最慢的 span 链
        slow_spans = []
        for span in longest.get("spans", []):
            if span["duration_ms"] > longest["duration_ms"] * 0.3:
                slow_spans.append({
                    "operation": span["operation_name"],
                    "service": span["service_name"],
                    "duration_ms": span["duration_ms"],
                    "tags": span.get("tags", {})
                })

        return {"trace_id": longest["trace_id"],
                "duration_ms": longest["duration_ms"],
                "slow_spans": slow_spans}

    def _extract_log_patterns(self, logs):
        """提取日志中的重复模式"""
        hits = logs.get("hits", {}).get("hits", [])
        if not hits:
            return []

        # 提取错误消息模板（去除变量部分）
        patterns = {}
        for hit in hits:
            msg = hit["_source"].get("message", "")
            template = self._extract_template(msg)
            patterns[template] = patterns.get(template, 0) + 1

        # 按频率排序
        sorted_patterns = sorted(patterns.items(), key=lambda x: x[1], reverse=True)
        return [{"pattern": p[0], "count": p[1]} for p in sorted_patterns[:5]]

    def _extract_template(self, msg):
        """提取消息模板（将数字和 ID 替换为占位符）"""
        template = re.sub(r'\d+', '{N}', msg)
        template = re.sub(r'[a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12}', '{UUID}', template)
        return template
```

## 异常场景补充

### 场景：三支柱数据时间不对齐

```
触发：Metrics 时间戳是 UTC，Logs 是本地时间，Traces 是另一个时区 → 关联失败
检测：
  1. 关联查询结果为空 → 时间不对齐
  2. 时间戳格式不一致 → 时区问题
处理：
  1. 统一所有数据源时间戳为 UTC
  2. 查询时自动转换时区
  3. 增大时间窗口容忍偏移
预防：统一 UTC 时间戳 + 自动时区转换 + 宽容时间窗口
```

### 场景：日志量过大导致 ES 搜索延迟

```
触发：服务故障 → 每秒 10 万条 ERROR 日志 → ES 搜索延迟 30 秒 → 排查受阻
检测：
  1. ES 搜索延迟 > 5 秒 → 过载
  2. 告警关联耗时 > 10 秒 → 无法快速排查
处理：
  1. 日志降采样（ERROR > 1万/秒时只保留 10%）
  2. 关联查询限制结果数（50 条）
  3. 使用实时流处理代替 ES 搜索
预防：日志降采样 + 结果数限制 + 流式处理
```

## 告警聚合与降噪完整实现

```python
class AlertAggregationService:
    """告警聚合与降噪：重复告警聚合 + 关联告警合并 + 静默期"""

    def process_alert(self, alert):
        """处理告警（聚合 + 降噪）"""
        # 1. 生成告警指纹（用于去重）
        fingerprint = self._generate_fingerprint(alert)

        # 2. 检查是否为重复告警
        existing = self.redis.get(f"alert_fingerprint:{fingerprint}")
        if existing:
            # 重复告警 → 增加计数，不重复通知
            count = self.redis.incr(f"alert_count:{fingerprint}")
            self.db.update("alerts",
                {"repeat_count": count, "last_seen_at": now()},
                {"fingerprint": fingerprint})
            return {"action": "deduplicated", "fingerprint": fingerprint, "count": count}

        # 3. 检查静默期
        if self._is_silenced(alert):
            return {"action": "silenced", "reason": "维护窗口或手动静默"}

        # 4. 关联聚合（同一服务的多个告警合并）
        related = self._find_related_alerts(alert)
        if related:
            # 合并为一条告警
            group_id = related[0].get("group_id") or str(uuid4())
            self.db.update("alerts",
                {"group_id": group_id}, {"id": related[0]["id"]})

            self.db.insert("alerts", {
                "id": str(uuid4()),
                "fingerprint": fingerprint,
                "group_id": group_id,
                "service": alert["service"],
                "severity": max(a["severity"] for a in related + [alert]),
                "title": f"[聚合] {alert['service']} {len(related)+1} 条告警",
                "description": json.dumps([a["title"] for a in related] + [alert["title"]]),
                "status": "firing",
                "repeat_count": 0,
                "first_seen_at": now(),
                "last_seen_at": now()
            })
            return {"action": "aggregated", "group_id": group_id,
                    "alert_count": len(related) + 1}

        # 5. 新告警 → 正常通知
        self.db.insert("alerts", {
            "id": str(uuid4()),
            "fingerprint": fingerprint,
            "service": alert["service"],
            "severity": alert["severity"],
            "title": alert["title"],
            "description": alert.get("description", ""),
            "status": "firing",
            "repeat_count": 0,
            "first_seen_at": now(),
            "last_seen_at": now()
        })

        # 记录指纹（TTL = 告警间隔）
        self.redis.setex(f"alert_fingerprint:{fingerprint}", 3600, "1")
        self.redis.set(f"alert_count:{fingerprint}", 1)

        # 发送通知
        self._notify(alert)

        return {"action": "notified", "fingerprint": fingerprint}

    def _generate_fingerprint(self, alert):
        """生成告警指纹（用于去重）"""
        # 基于服务 + 指标 + 标签生成唯一指纹
        key_parts = [
            alert.get("service", ""),
            alert.get("metric", ""),
            alert.get("severity", ""),
            json.dumps(alert.get("labels", {}), sort_keys=True)
        ]
        return hashlib.md5("|".join(key_parts).encode()).hexdigest()

    def _find_related_alerts(self, alert):
        """查找关联告警"""
        # 同一服务 5 分钟内的其他告警
        return self.db.query(
            "SELECT * FROM alerts "
            "WHERE service = %s AND status = 'firing' "
            "AND last_seen_at > NOW() - INTERVAL 5 MINUTE "
            "AND fingerprint != %s",
            alert["service"],
            self._generate_fingerprint(alert))

    def _is_silenced(self, alert):
        """检查是否在静默期"""
        # 1. 维护窗口
        maintenance = self.db.query_one(
            "SELECT * FROM maintenance_windows "
            "WHERE service = %s AND NOW() BETWEEN start_time AND end_time",
            alert["service"])
        if maintenance:
            return True

        # 2. 手动静默
        silence = self.redis.get(f"silence:{alert['service']}:{alert.get('metric', '')}")
        if silence:
            return True

        return False

    def set_silence(self, service, metric, duration_minutes, reason, set_by):
        """设置静默"""
        silence_id = str(uuid4())
        self.redis.setex(
            f"silence:{service}:{metric}",
            duration_minutes * 60, reason)

        self.db.insert("alert_silences", {
            "silence_id": silence_id,
            "service": service, "metric": metric,
            "duration_minutes": duration_minutes,
            "reason": reason, "set_by": set_by,
            "expires_at": now() + timedelta(minutes=duration_minutes),
            "created_at": now()
        })

        return {"silence_id": silence_id, "expires_at": (now() + timedelta(minutes=duration_minutes)).isoformat()}
```

## 异常场景补充

### 场景：告警聚合导致关键告警被淹没

```
触发：低级别告警先到 → 高级别告警被聚合到同组 → 高级别告警未单独通知 → 延迟响应
检测：
  1. 聚合组中最高级别告警响应时间 > SLA → 聚合问题
  2. 高级别告警被聚合后通知标题不够醒目
处理：
  1. 高级别告警不参与聚合，单独通知
  2. 聚合组中有高级别 → 升级通知
  3. 聚合通知中突出最高级别
预防：高级别不聚合 + 聚合升级 + 突出最高级别
```

### 场景：静默期被滥用

```
触发：运维人员设置 24 小时静默 → 忘记取消 → 真正告警被静默 → 故障未发现
检测：
  1. 静默期 > 4 小时 → 可能滥用
  2. 静默期间服务故障 → 静默问题
处理：
  1. 限制静默时长（最长 4 小时，需升级审批更长）
  2. 静默期到期自动恢复
  3. 高级别告警不受静默影响
预防：静默时长限制 + 自动恢复 + 高级别例外
```

## 容量规划与预测完整实现

```python
class CapacityPlanningService:
    """容量规划：资源预测 + 容量预警 + 扩容建议"""

    RESOURCE_TYPES = {
        "cpu": {"unit": "cores", "utilization_threshold": 0.7},
        "memory": {"unit": "GB", "utilization_threshold": 0.8},
        "disk": {"unit": "TB", "utilization_threshold": 0.75},
        "network": {"unit": "Gbps", "utilization_threshold": 0.6},
        "qps": {"unit": "requests/s", "utilization_threshold": 0.7},
    }

    def predict_capacity(self, service_name, resource_type, days_ahead=30):
        """预测容量需求"""
        config = self.RESOURCE_TYPES[resource_type]

        # 1. 获取历史数据（90 天）
        history = self.prometheus.query_range(
            service=service_name,
            metric=f"{resource_type}_utilization",
            start=now() - timedelta(days=90),
            end=now(),
            step="1h"
        )

        if len(history) < 168:  # 至少 7 天数据
            return {"status": "insufficient_data", "data_points": len(history)}

        # 2. 提取时间序列
        timestamps = [h["timestamp"] for h in history]
        values = [h["value"] for h in history]

        # 3. 线性回归预测
        x = np.array([(t - timestamps[0]) / 86400 for t in timestamps])  # 天数
        y = np.array(values)

        # 加入周期性（日/周）
        hours = np.array([(t % 86400) / 3600 for t in timestamps])
        day_of_week = np.array([(t // 86400) % 7 for t in timestamps])

        # 简化模型：趋势 + 日周期
        coefficients = np.polyfit(x, y, 1)  # 线性趋势
        trend = np.poly1d(coefficients)

        # 残差 → 日周期模式
        residuals = y - trend(x)
        hourly_pattern = np.zeros(24)
        hourly_counts = np.zeros(24)
        for i, h in enumerate(hours):
            h_int = int(h) % 24
            hourly_pattern[h_int] += residuals[i]
            hourly_counts[h_int] += 1
        hourly_pattern = np.where(hourly_counts > 0, hourly_pattern / hourly_counts, 0)

        # 4. 预测未来
        future_days = np.arange(0, days_ahead, 1/24)  # 每小时
        predicted_trend = trend(len(x) + future_days)
        predicted_cycle = np.array([hourly_pattern[int(d * 24) % 24] for d in future_days])
        predicted = predicted_trend + predicted_cycle

        # 5. 找到首次超阈值时间
        threshold = config["utilization_threshold"]
        breach_idx = None
        for i, p in enumerate(predicted):
            if p > threshold:
                breach_idx = i
                break

        breach_date = None
        if breach_idx is not None:
            breach_date = (now() + timedelta(hours=breach_idx)).isoformat()

        # 6. 扩容建议
        current_capacity = self._get_current_capacity(service_name, resource_type)
        peak_predicted = max(predicted) if len(predicted) > 0 else 0
        required_capacity = current_capacity * (peak_predicted / threshold) * 1.1  # 10% 余量

        return {
            "service": service_name,
            "resource_type": resource_type,
            "current_utilization": round(values[-1], 3),
            "current_capacity": current_capacity,
            "predicted_peak": round(peak_predicted, 3),
            "threshold": threshold,
            "breach_date": breach_date,
            "days_until_breach": breach_idx / 24 if breach_idx else None,
            "recommended_capacity": round(required_capacity, 1),
            "scale_up_by": round(required_capacity - current_capacity, 1),
            "confidence": round(1 - np.std(residuals) / max(np.mean(y), 0.01), 2)
        }

    def _get_current_capacity(self, service_name, resource_type):
        """获取当前容量"""
        return self.db.query_one(
            "SELECT capacity FROM service_resources "
            "WHERE service_name = %s AND resource_type = %s",
            service_name, resource_type)["capacity"]
```

## 异常场景补充

### 场景：容量预测模型不适应突变

```
触发：业务量因营销活动突然增长 5x → 线性预测无法覆盖 → 容量不足 → 服务降级
检测：
  1. 实际使用率超过预测上界 → 突变
  2. 营销活动未纳入预测 → 遗漏
处理：
  1. 营销活动提前通知容量规划
  2. 弹性扩容（自动伸缩）
  3. 突变检测 → 紧急扩容
预防：活动预报 + 弹性伸缩 + 突变检测
```

### 场景：容量预测置信度过低

```
触发：服务使用量波动大 → 线性回归 R² < 0.3 → 预测不可靠 → 扩容决策犹豫
检测：
  1. 预测置信度 < 0.5 → 不可靠
  2. 预测值与实际偏差 > 30% → 模型不准
处理：
  1. 使用更复杂的模型（Prophet / ARIMA）
  2. 增加业务特征（促销/节假日）
  3. 保守策略：按 2 倍预测值扩容
预防：复杂模型 + 业务特征 + 保守策略
```

## 可观测性 SLO 管理完整实现

```python
class SLOManagementService:
    """SLO 管理：定义 → 监控 → 错误预算 → 燃尽图"""

    def create_slo(self, service_name, slo_name, target_pct,
                   window_days=30, category="availability"):
        """创建 SLO"""
        slo_id = str(uuid4())

        # 错误预算 = 100% - SLO 目标
        error_budget_pct = 100 - target_pct

        self.db.insert("service_slos", {
            "slo_id": slo_id,
            "service_name": service_name,
            "slo_name": slo_name,
            "category": category,  # availability / latency / correctness
            "target_pct": target_pct,
            "error_budget_pct": error_budget_pct,
            "window_days": window_days,
            "status": "active",
            "created_at": now()
        })

        return {"slo_id": slo_id, "target_pct": target_pct,
                "error_budget_pct": error_budget_pct}

    def calculate_slo_status(self, slo_id):
        """计算 SLO 当前状态"""
        slo = self.db.get_slo(slo_id)
        window_days = slo["window_days"]
        window_start = now() - timedelta(days=window_days)

        if slo["category"] == "availability":
            # 可用性：成功请求数 / 总请求数
            total = self.prometheus.query_range(
                slo["service_name"], "http_requests_total",
                start=window_start, end=now())

            errors = self.prometheus.query_range(
                slo["service_name"], "http_requests_errors",
                start=window_start, end=now())

            good_events = sum(t["value"] for t in total) - sum(e["value"] for e in errors)
            total_events = sum(t["value"] for t in total)

            current_pct = (good_events / max(total_events, 1)) * 100

        elif slo["category"] == "latency":
            # 延迟：P99 < 阈值 的请求比例
            threshold_ms = slo.get("latency_threshold_ms", 200)
            total = self.prometheus.query_range(
                slo["service_name"], "http_requests_total",
                start=window_start, end=now())

            fast = self.prometheus.query_range(
                slo["service_name"], f"http_requests_duration_bucket_le_{threshold_ms}",
                start=window_start, end=now())

            good_events = sum(f["value"] for f in fast)
            total_events = sum(t["value"] for t in total)

            current_pct = (good_events / max(total_events, 1)) * 100

        # 错误预算消耗
        error_budget_total = slo["error_budget_pct"] * window_days  # 总预算（百分比×天数）
        error_budget_consumed = max(0, (slo["target_pct"] - current_pct) * window_days)
        error_budget_remaining = error_budget_total - error_budget_consumed
        error_budget_pct_remaining = (error_budget_remaining / max(error_budget_total, 0.01)) * 100

        # 燃尽率
        days_elapsed = (now() - window_start).days
        burn_rate = error_budget_consumed / max(days_elapsed, 1)

        # 预计耗尽时间
        if burn_rate > 0:
            days_until_exhausted = error_budget_remaining / burn_rate
        else:
            days_until_exhausted = float('inf')

        status = "healthy" if current_pct >= slo["target_pct"] else \
                "at_risk" if error_budget_pct_remaining > 10 else "violated"

        return {
            "slo_id": slo_id,
            "service_name": slo["service_name"],
            "slo_name": slo["slo_name"],
            "target_pct": slo["target_pct"],
            "current_pct": round(current_pct, 4),
            "window_days": window_days,
            "error_budget": {
                "total_pct": slo["error_budget_pct"],
                "consumed_pct": round(100 - error_budget_pct_remaining, 2),
                "remaining_pct": round(error_budget_pct_remaining, 2),
            },
            "burn_rate": round(burn_rate, 4),
            "days_until_exhausted": round(days_until_exhausted, 1) if days_until_exhausted != float('inf') else None,
            "status": status
        }

    def get_slo_burn_rate_alerts(self, slo_id):
        """获取 SLO 燃尽率告警"""
        status = self.calculate_slo_status(slo_id)

        alerts = []

        # 快速燃尽（1 小时内消耗 14.4 天的预算 → 14.4x 燃尽率）
        if status["burn_rate"] > 14.4:
            alerts.append({"severity": "critical", "message": "SLO 快速燃尽（1h 窗口）"})

        # 慢速燃尽（6 小时内消耗 6 天的预算 → 6x 燃尽率）
        elif status["burn_rate"] > 6:
            alerts.append({"severity": "high", "message": "SLO 慢速燃尽（6h 窗口）"})

        # 长期燃尽（3 天内消耗 1 天的预算 → 1x 燃尽率）
        elif status["burn_rate"] > 1:
            alerts.append({"severity": "warning", "message": "SLO 长期燃尽（3d 窗口）"})

        return {"slo_id": slo_id, "alerts": alerts, "status": status["status"]}
```

## 异常场景补充

### 场景：SLO 目标设定不合理

```
触发：SLO 设为 99.999%（5 个 9）→ 错误预算仅 0.001% → 任何一次故障都耗尽预算 → 告警风暴
检测：
  1. 错误预算 < 0.01% → 目标过严
  2. SLO 告警过于频繁 → 目标不合理
处理：
  1. 重新评估 SLO 目标（参考行业标准和历史数据）
  2. 分级 SLO（核心接口 99.99%，非核心 99.9%）
  3. 基于窗口调整（30 天窗口比 7 天更稳定）
预防：合理目标 + 分级 SLO + 窗口调整
```

### 场景：SLO 数据源不可靠

```
触发：Prometheus 采集延迟 → SLO 计算基于不完整数据 → 误判 SLO 违规 → 不必要的告警
检测：
  1. SLO 计算时段内数据点缺失 > 10% → 数据不可靠
  2. SLO 状态频繁切换 → 数据不稳定
处理：
  1. SLO 计算前检查数据完整性
  2. 数据不完整 → 降级为"未知"状态
  3. 使用多个数据源交叉验证
预防：数据完整性检查 + 降级状态 + 多源验证
```

## 可观测性告警聚合与去重完整实现

```python
import re
import time
import hashlib
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from typing import Optional, Dict, List, Set, Tuple
from enum import Enum
from collections import defaultdict


class AlertSeverity(Enum):
    CRITICAL = "critical"
    HIGH = "high"
    MEDIUM = "medium"
    LOW = "low"
    INFO = "info"

    @property
    def weight(self) -> int:
        return {
            AlertSeverity.CRITICAL: 5,
            AlertSeverity.HIGH: 4,
            AlertSeverity.MEDIUM: 3,
            AlertSeverity.LOW: 2,
            AlertSeverity.INFO: 1,
        }[self]


class IncidentStatus(Enum):
    ACTIVE = "active"
    RESOLVED = "resolved"
    ACKNOWLEDGED = "acknowledged"


@dataclass
class RawAlert:
    """原始告警"""
    alert_id: str
    service_name: str
    message: str
    severity: AlertSeverity
    timestamp: datetime = field(default_factory=datetime.utcnow)
    labels: Dict[str, str] = field(default_factory=dict)
    source: str = ""


@dataclass
class AggregatedAlert:
    """聚合后的告警"""
    group_key: str
    service_name: str
    message_pattern: str
    severity: AlertSeverity
    count: int = 1
    first_seen: datetime = field(default_factory=datetime.utcnow)
    last_seen: datetime = field(default_factory=datetime.utcnow)
    alert_ids: List[str] = field(default_factory=list)
    labels: Dict[str, str] = field(default_factory=dict)


@dataclass
class Incident:
    """聚合生成的运维事件"""
    incident_id: str
    title: str
    severity: AlertSeverity
    status: IncidentStatus = IncidentStatus.ACTIVE
    affected_services: List[str] = field(default_factory=list)
    alert_count: int = 0
    created_at: datetime = field(default_factory=datetime.utcnow)
    updated_at: datetime = field(default_factory=datetime.utcnow)
    resolved_at: Optional[datetime] = None
    aggregated_alerts: List[AggregatedAlert] = field(default_factory=list)
    dedup_count: int = 0


class AlertAggregationService:
    """可观测性告警聚合与去重服务
    
    接收原始告警, 按服务名和相似消息在时间窗口内聚合,
    去重相同告警, 根据告警密度计算严重等级, 生成聚合事件。
    """

    AGGREGATION_WINDOW = timedelta(minutes=5)
    SIMILARITY_THRESHOLD = 0.7
    MAX_ALERTS_PER_WINDOW = 500  # 防止告警风暴

    def __init__(self):
        self._raw_alerts: List[RawAlert] = []
        self._dedup_set: Set[str] = set()
        self._aggregated: Dict[str, AggregatedAlert] = {}
        self._incidents: Dict[str, Incident] = {}
        self._service_alert_counts: Dict[str, List[Tuple[datetime, str]]] = defaultdict(list)
        self._alert_rate_limit: Dict[str, int] = defaultdict(int)

    def _compute_dedup_key(self, alert: RawAlert) -> str:
        """计算告警去重键: 相同服务 + 相同消息 + 相同标签 = 重复告警"""
        normalized_msg = re.sub(r'\d+', 'N', alert.message.strip().lower())
        label_str = ",".join(f"{k}={v}" for k, v in sorted(alert.labels.items()))
        raw = f"{alert.service_name}:{normalized_msg}:{label_str}"
        return hashlib.sha256(raw.encode("utf-8")).hexdigest()[:16]

    def _compute_group_key(self, alert: RawAlert) -> str:
        """计算告警分组键: 相同服务 + 消息模式 (忽略具体数值)"""
        pattern = re.sub(r'0x[0-9a-fA-F]+', 'HEX', alert.message)
        pattern = re.sub(r'\d+', 'N', pattern)
        pattern = re.sub(r'\s+', ' ', pattern.strip().lower())
        raw = f"{alert.service_name}:{pattern}"
        return hashlib.sha256(raw.encode("utf-8")).hexdigest()[:16]

    def _message_similarity(self, msg1: str, msg2: str) -> float:
        """基于 Jaccard 相似度计算两条消息的相似性"""
        words1 = set(re.findall(r'\w+', msg1.lower()))
        words2 = set(re.findall(r'\w+', msg2.lower()))
        if not words1 or not words2:
            return 0.0
        intersection = words1 & words2
        union = words1 | words2
        return len(intersection) / len(union)

    def _calculate_severity_by_density(
        self, service_name: str, now: datetime
    ) -> AlertSeverity:
        """根据告警密度 (时间窗口内同服务告警数) 计算严重等级"""
        window_start = now - self.AGGREGATION_WINDOW
        counts = self._service_alert_counts.get(service_name, [])
        recent = [ts for ts, _ in counts if ts >= window_start]
        density = len(recent)

        if density >= 50:
            return AlertSeverity.CRITICAL
        elif density >= 20:
            return AlertSeverity.HIGH
        elif density >= 10:
            return AlertSeverity.MEDIUM
        elif density >= 3:
            return AlertSeverity.LOW
        return AlertSeverity.INFO

    def ingest_alert(self, alert: RawAlert) -> Optional[AggregatedAlert]:
        """接收原始告警并进行去重和聚合
        
        Args:
            alert: 原始告警对象
            
        Returns:
            聚合后的告警 (如果是新告警), 或 None (如果是重复告警)
            
        Raises:
            ValueError: 告警缺少必要字段
            RuntimeError: 告警速率超过限流阈值
        """
        if not alert.alert_id or not alert.service_name or not alert.message:
            raise ValueError("Alert must have alert_id, service_name, and message")

        now = datetime.utcnow()

        # 限流检查: 防止告警风暴
        self._alert_rate_limit[alert.service_name] += 1
        window_key = f"{alert.service_name}:{now.strftime('%Y%m%d%H%M')}"
        if self._alert_rate_limit.get(window_key, 0) > self.MAX_ALERTS_PER_WINDOW:
            raise RuntimeError(
                f"Alert rate limit exceeded for service {alert.service_name}: "
                f">{self.MAX_ALERTS_PER_WINDOW} alerts/min"
            )

        # 去重检查
        dedup_key = self._compute_dedup_key(alert)
        if dedup_key in self._dedup_set:
            # 更新已有聚合告警的计数
            group_key = self._compute_group_key(alert)
            if group_key in self._aggregated:
                self._aggregated[group_key].count += 1
                self._aggregated[group_key].last_seen = now
                self._aggregated[group_key].alert_ids.append(alert.alert_id)
            return None

        self._dedup_set.add(dedup_key)
        self._raw_alerts.append(alert)

        # 记录服务告警计数 (用于密度计算)
        self._service_alert_counts[alert.service_name].append(
            (now, alert.alert_id)
        )

        # 聚合
        return self.aggregate_alerts(alert)

    def aggregate_alerts(self, alert: RawAlert) -> AggregatedAlert:
        """将告警聚合到对应分组
        
        同一服务 + 相似消息 + 时间窗口内的告警聚合为一组。
        
        Args:
            alert: 原始告警
            
        Returns:
            聚合后的告警
        """
        now = datetime.utcnow()
        group_key = self._compute_group_key(alert)

        if group_key in self._aggregated:
            existing = self._aggregated[group_key]
            # 检查是否在聚合窗口内
            if now - existing.last_seen <= self.AGGREGATION_WINDOW:
                similarity = self._message_similarity(
                    existing.message_pattern, alert.message
                )
                if similarity >= self.SIMILARITY_THRESHOLD:
                    existing.count += 1
                    existing.last_seen = now
                    existing.alert_ids.append(alert.alert_id)
                    # 保留最高严重等级
                    if alert.severity.weight > existing.severity.weight:
                        existing.severity = alert.severity
                    return existing

        # 创建新的聚合告警
        aggregated = AggregatedAlert(
            group_key=group_key,
            service_name=alert.service_name,
            message_pattern=alert.message,
            severity=alert.severity,
            count=1,
            first_seen=now,
            last_seen=now,
            alert_ids=[alert.alert_id],
            labels=alert.labels,
        )
        self._aggregated[group_key] = aggregated
        return aggregated

    def deduplicate(self, alerts: List[RawAlert]) -> List[RawAlert]:
        """批量去重告警列表
        
        Args:
            alerts: 原始告警列表
            
        Returns:
            去重后的告警列表
        """
        seen_keys: Set[str] = set()
        unique_alerts: List[RawAlert] = []
        for alert in alerts:
            dedup_key = self._compute_dedup_key(alert)
            if dedup_key not in seen_keys:
                seen_keys.add(dedup_key)
                unique_alerts.append(alert)
        return unique_alerts

    def create_incident(
        self, aggregated_alerts: List[AggregatedAlert]
    ) -> Incident:
        """从聚合告警创建运维事件
        
        当聚合告警的严重等级达到 HIGH 或 CRITICAL 时,
        自动创建 Incident 通知运维团队。
        
        Args:
            aggregated_alerts: 触发事件的聚合告警列表
            
        Returns:
            创建的 Incident
        """
        now = datetime.utcnow()

        # 计算事件严重等级: 取所有告警中的最高等级
        max_severity = AlertSeverity.INFO
        for agg in aggregated_alerts:
            # 同时考虑密度调整后的等级
            density_severity = self._calculate_severity_by_density(
                agg.service_name, now
            )
            effective = (
                density_severity
                if density_severity.weight > agg.severity.weight
                else agg.severity
            )
            if effective.weight > max_severity.weight:
                max_severity = effective

        # 生成事件 ID 和标题
        services = list({a.service_name for a in aggregated_alerts})
        total_count = sum(a.count for a in aggregated_alerts)
        incident_id = f"INC-{now.strftime('%Y%m%d%H%M%S')}-{hashlib.md5(services[0].encode()).hexdigest()[:6]}"
        title = f"Alert storm on {', '.join(services)}: {total_count} alerts aggregated"

        incident = Incident(
            incident_id=incident_id,
            title=title,
            severity=max_severity,
            status=IncidentStatus.ACTIVE,
            affected_services=services,
            alert_count=total_count,
            created_at=now,
            updated_at=now,
            aggregated_alerts=aggregated_alerts,
            dedup_count=sum(
                a.count - 1 for a in aggregated_alerts if a.count > 1
            ),
        )
        self._incidents[incident_id] = incident
        return incident

    def get_active_incidents(
        self,
        service_name: Optional[str] = None,
        min_severity: Optional[AlertSeverity] = None,
    ) -> List[Incident]:
        """获取活跃事件列表
        
        Args:
            service_name: 按服务名过滤 (可选)
            min_severity: 最低严重等级过滤 (可选)
            
        Returns:
            活跃事件列表, 按严重等级降序排列
        """
        active = [
            inc for inc in self._incidents.values()
            if inc.status == IncidentStatus.ACTIVE
        ]
        if service_name:
            active = [
                inc for inc in active
                if service_name in inc.affected_services
            ]
        if min_severity:
            active = [
                inc for inc in active
                if inc.severity.weight >= min_severity.weight
            ]
        active.sort(key=lambda inc: inc.severity.weight, reverse=True)
        return active

    def resolve_incident(self, incident_id: str) -> Optional[Incident]:
        """解决事件"""
        if incident_id not in self._incidents:
            return None
        incident = self._incidents[incident_id]
        incident.status = IncidentStatus.RESOLVED
        incident.resolved_at = datetime.utcnow()
        incident.updated_at = datetime.utcnow()
        return incident

    def cleanup_expired(self, max_age: timedelta = timedelta(hours=24)):
        """清理过期的聚合告警和去重记录"""
        now = datetime.utcnow()
        expired_keys = [
            key for key, agg in self._aggregated.items()
            if now - agg.last_seen > max_age
        ]
        for key in expired_keys:
            del self._aggregated[key]

        # 清理服务告警计数中的过期记录
        for service in self._service_alert_counts:
            self._service_alert_counts[service] = [
                (ts, aid) for ts, aid in self._service_alert_counts[service]
                if now - ts <= max_age
            ]
```

## 异常场景补充

### 场景：告警风暴导致系统过载
```
trigger: 某服务大规模故障 (如数据库主库宕机) 在 1 分钟内产生 5000+ 条告警, 超过 MAX_ALERTS_PER_WINDOW 限流阈值

detection:
  1. ingest_alert 中限流检查触发 RuntimeError, 记录被丢弃的告警数量
  2. 监控告警摄入速率: 当每分钟告警数超过阈值的 80% 时发出预警
  3. 检测 _raw_alerts 列表增长速率: 若 1 分钟内增长超过 1000 条, 判定为告警风暴

handling:
  1. 启动告警风暴模式: 将所有新告警暂存到溢出队列 (磁盘或消息队列), 不做实时聚合
  2. 对溢出告警进行采样: 每 N 条取 1 条 (N = 当前速率 / 正常速率), 采样告警正常聚合
  3. 保留 CRITICAL 级别告警的完整处理, 不做采样
  4. 风暴结束后 (速率降到正常水平 2 倍以下), 从溢出队列回放告警进行批量聚合
  5. 在 create_incident 中标注 "alert storm mode", 提醒运维告警可能不完整

prevention:
  1. 在告警源端配置限流: 每个服务每分钟最多发送 N 条告警, 超出部分在源端聚合
  2. 使用消息队列 (如 Kafka) 作为告警摄入缓冲, 解耦告警产生和消费速率
  3. 设置 ingest_alert 的背压机制: 当内部队列超过阈值时, 返回 429/503 让告警源重试
  4. 实现分层聚合: 先在服务实例本地聚合, 再发送到中心聚合服务, 减少中心节点压力
  5. 定期清理过期数据 (cleanup_expired), 防止内存无限增长
```

### 场景：关键告警被误聚合为低优先级
```
trigger: 一条 CRITICAL 级别的数据库连接失败告警, 因其消息与同一服务下大量 LOW 级别的连接超时告警相似度 > 0.7, 被聚合到已有的 LOW 级别聚合组中

detection:
  1. 在 aggregate_alerts 中, 当新告警的 severity 高于已有聚合组的 severity 时, 记录升级事件日志
  2. 检查聚合组内告警的严重等级方差: 若方差 > 2 (如组内同时有 CRITICAL 和 LOW), 标记为 "severity_mismatch"
  3. 在 create_incident 中校验: 若事件包含 severity_mismatch 的聚合组, 发出人工审核通知

handling:
  1. 聚合时始终升级: 新告警 severity 高于聚合组时, 将聚合组 severity 升级为最高值
  2. 对 CRITICAL 和 HIGH 级别告警实行独立聚合: 不与 MEDIUM/LOW/INFO 级别告警混合
  3. 当检测到 severity_mismatch 时, 将高严重等级告警从原聚合组拆分出来, 创建独立聚合组
  4. 拆分后立即为高严重等级聚合组调用 create_incident, 确保关键事件不被延误
  5. 通知运维团队: 发送 "告警聚合拆分" 通知, 包含原始聚合组 ID 和新拆分组 ID

prevention:
  1. 聚合分组时加入严重等级维度: group_key 计算包含 severity 等级, 不同等级不聚合到同一组
  2. 设置聚合升级规则: 聚合组内任一告警为 CRITICAL 时, 整组自动升级为 CRITICAL
  3. 对高严重等级告警设置更短的聚合窗口 (如 1 分钟), 确保快速生成事件
  4. 实现 "保底通道": CRITICAL 告警在正常聚合流程之外, 额外走一条直达事件创建的快速通道
  5. 定期审计聚合规则: 检查是否存在因消息相似度误判导致的高等级告警被低等级聚合吞没的情况
```

## 可观测性 SLO 管理与错误预算完整实现

```python
class SLOManagementService:
    """SLO 管理：SLO 定义 → 错误预算计算 → 预警消耗 → 预算耗尽处理"""

    def define_slo(self, service_name, slo_name, target_pct, window_days,
                   indicator_type, indicator_config):
        """定义 SLO"""
        slo_id = str(uuid4())

        # 验证目标值
        target = Decimal(str(target_pct))
        if target < Decimal("90") or target > Decimal("99.99"):
            raise ValueError("SLO 目标需在 90%-99.99% 之间")

        # 计算错误预算
        error_budget_pct = Decimal("100") - target
        error_budget_ratio = error_budget_pct / Decimal("100")

        self.db.insert("slo_definitions", {
            "slo_id": slo_id,
            "service_name": service_name,
            "slo_name": slo_name,
            "target_pct": str(target),
            "window_days": window_days,
            "indicator_type": indicator_type,  # availability / latency / correctness
            "indicator_config": json.dumps(indicator_config),
            "error_budget_pct": str(error_budget_pct),
            "error_budget_ratio": str(error_budget_ratio),
            "status": "active",
            "created_at": now()
        })

        return {"slo_id": slo_id, "service_name": service_name,
                "slo_name": slo_name, "target_pct": str(target),
                "error_budget_pct": str(error_budget_pct)}

    def calculate_error_budget(self, slo_id):
        """计算错误预算消耗"""
        slo = self.db.get_slo_definition(slo_id)
        window_days = slo["window_days"]
        target = Decimal(str(slo["target_pct"]))
        indicator_type = slo["indicator_type"]
        indicator_config = json.loads(slo["indicator_config"])

        # 1. 计算窗口内总事件数
        start_time = now() - timedelta(days=window_days)
        total_events = self.db.count("service_metrics",
            service_name=slo["service_name"],
            metric_name=indicator_config["metric"],
            timestamp__gte=start_time)

        # 2. 计算不良事件数
        if indicator_type == "availability":
            bad_events = self.db.count("service_metrics",
                service_name=slo["service_name"],
                metric_name=indicator_config["metric"],
                value__lt=indicator_config["threshold"],
                timestamp__gte=start_time)

        elif indicator_type == "latency":
            bad_events = self.db.count("service_metrics",
                service_name=slo["service_name"],
                metric_name=indicator_config["metric"],
                value__gt=indicator_config["threshold_ms"],
                timestamp__gte=start_time)

        elif indicator_type == "correctness":
            bad_events = self.db.count("service_metrics",
                service_name=slo["service_name"],
                metric_name=indicator_config["metric"],
                value__ne=indicator_config["expected_value"],
                timestamp__gte=start_time)

        # 3. 计算当前 SLO 达成率
        current_pct = (total_events - bad_events) / max(total_events, 1) * 100

        # 4. 计算错误预算
        total_budget = (Decimal("100") - target) / Decimal("100") * total_events
        consumed = bad_events
        remaining = total_budget - consumed

        budget_consumed_pct = consumed / max(total_budget, 1) * 100

        # 5. 预算状态
        if budget_consumed_pct >= 100:
            budget_status = "exhausted"
        elif budget_consumed_pct >= 80:
            budget_status = "critical"
        elif budget_consumed_pct >= 60:
            budget_status = "warning"
        else:
            budget_status = "healthy"

        # 6. 记录并告警
        self.db.insert("slo_budget_snapshots", {
            "snapshot_id": str(uuid4()),
            "slo_id": slo_id,
            "total_events": total_events,
            "bad_events": bad_events,
            "current_pct": round(current_pct, 2),
            "total_budget": int(total_budget),
            "consumed_budget": consumed,
            "remaining_budget": int(remaining),
            "budget_consumed_pct": round(budget_consumed_pct, 1),
            "budget_status": budget_status,
            "snapshot_at": now()
        })

        if budget_status in ["critical", "exhausted"]:
            self.alert(
                f"SLO {slo['slo_name']} 错误预算 {budget_status}: "
                f"已消耗 {budget_consumed_pct:.1f}%, "
                f"剩余 {int(remaining)} 个事件")

        return {
            "slo_id": slo_id,
            "service_name": slo["service_name"],
            "slo_name": slo["slo_name"],
            "current_pct": round(current_pct, 2),
            "target_pct": str(target),
            "total_events": total_events,
            "bad_events": bad_events,
            "total_budget": int(total_budget),
            "consumed_budget": consumed,
            "remaining_budget": int(remaining),
            "budget_consumed_pct": round(budget_consumed_pct, 1),
            "budget_status": budget_status
        }

    def handle_budget_exhaustion(self, slo_id):
        """处理预算耗尽"""
        slo = self.db.get_slo_definition(slo_id)

        # 1. 标记 SLO 为违反状态
        self.db.update("slo_definitions",
            {"status": "violated"},
            {"slo_id": slo_id})

        # 2. 冻结非紧急变更
        self.change_management.freeze_deployments(
            slo["service_name"],
            reason=f"SLO {slo['slo_name']} 错误预算耗尽")

        # 3. 创建事故响应
        incident_id = self.incident_service.create_incident({
            "service_name": slo["service_name"],
            "title": f"SLO 违反: {slo['slo_name']}",
            "severity": "high",
            "slo_id": slo_id
        })

        # 4. 通知
        self.notification.send("sre_team",
            f"SLO 违反: {slo['slo_name']} ({slo['service_name']}), "
            f"错误预算耗尽，已冻结部署，事故 {incident_id}")

        return {"slo_id": slo_id, "status": "violated",
                "incident_id": incident_id}

    def generate_slo_report(self, service_name, period_days=30):
        """生成 SLO 报告"""
        slos = self.db.query(
            "SELECT * FROM slo_definitions "
            "WHERE service_name = %s AND status != 'deleted'",
            service_name)

        report = {
            "service_name": service_name,
            "period_days": period_days,
            "slos": []
        }

        for slo in slos:
            budget = self.calculate_error_budget(slo["slo_id"])

            # 获取历史趋势
            history = self.db.query(
                "SELECT snapshot_at, current_pct, budget_consumed_pct "
                "FROM slo_budget_snapshots "
                "WHERE slo_id = %s "
                "AND snapshot_at > NOW() - INTERVAL %s DAY "
                "ORDER BY snapshot_at",
                slo["slo_id"], period_days)

            trend = {
                "dates": [h["snapshot_at"].strftime("%Y-%m-%d") for h in history],
                "current_pct": [h["current_pct"] for h in history],
                "budget_consumed_pct": [h["budget_consumed_pct"] for h in history]
            }

            report["slos"].append({
                "slo_name": slo["slo_name"],
                "target_pct": slo["target_pct"],
                "current_pct": budget["current_pct"],
                "budget_status": budget["budget_status"],
                "budget_consumed_pct": budget["budget_consumed_pct"],
                "trend": trend,
                "is_met": budget["current_pct"] >= float(slo["target_pct"])
            })

        return report
```

## 异常场景补充

### 场景：SLO 目标设置不合理导致错误预算总是耗尽

```
触发：设置 99.99% SLO → 实际达成率 99.5% → 每月错误预算都耗尽 → 持续冻结部署 → 开发停滞
检测：
  1. SLO 连续 3 个月违反 → 目标可能过高
  2. 错误预算总是在月初就耗尽 → SLI 与 SLO 不匹配
处理：
  1. 根据实际达成率调整 SLO（99.99% → 99.9%）
  2. 设置阶梯 SLO（99.9% 基础 + 99.95% 挑战）
  3. 定期审查 SLO 合理性
预防：阶梯 SLO + 定期审查 + 数据驱动调整
```

### 场景：错误预算计算指标数据缺失

```
触发：监控系统维护 → 1 小时数据缺失 → 错误预算计算不准 → 错误判断预算耗尽
检测：
  1. SLI 数据连续缺失 > 5 分钟 → 数据空洞
  2. 错误预算突然跳变 → 数据缺失导致
处理：
  1. 数据缺失期间不纳入错误预算计算
  2. 使用最近有效数据填补空洞
  3. 数据质量检查前置
预防：排除空洞 + 数据填补 + 质量检查
```

### 场景：多个 SLO 同时违反导致部署永久冻结

```
触发：服务有 3 个 SLO → 2 个预算耗尽 → 部署冻结 → 第 3 个 SLO 也因无修复而违反 → 全部冻结 → 死锁
检测：
  1. 所有 SLO 同时 violated → 全冻结
  2. 冻结持续时间 > 7 天 → 部署死锁
处理：
  1. 允许修复性部署（标记为 SLO 修复）
  2. 冻结策略分级（仅冻结高风险变更）
  3. SLO 修复变更自动审批
预防：修复部署 + 分级冻结 + 自动审批
```
