---
name: microservice-observability-tracing
description: 微服务可观测性模式：OpenTelemetry 分布式追踪、结构化日志聚合、Metrics 与告警的最佳实践
tags: [patterns, observability, distributed-tracing, opentelemetry, metrics, logging, monitoring]
---

## 概述

微服务可观测性（Observability）三支柱：**分布式追踪（Tracing）**、**结构化日志（Logging）**、**指标（Metrics）**。OpenTelemetry（OTel）已成为业界标准的数据采集层，统一了 Trace、Metrics、Logs 的生成和导出格式。

## 一、核心架构

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Service A│  │ Service B│  │ Service C│
│  OTel SDK│→│  OTel SDK│→│  OTel SDK│
└─────┬────┘  └─────┬────┘  └─────┬────┘
      │             │             │
      └─────────────┼─────────────┘
                    ▼
          ┌─────────────────┐
          │  OTel Collector │ ← 核心节点：批处理、过滤、采样、重试
          └────────┬────────┘
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
   Jaeger/Tempo  Prometheus  Loki/Elastic
    (Tracing)    (Metrics)    (Logs)
```

## 二、分布式追踪（Tracing）

### 2.1 Spring Boot 集成 OpenTelemetry

```xml
<!-- build.gradle / pom.xml -->
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
    <version>2.12.0</version>
</dependency>
```

```yaml
# application.yml
otel:
  service:
    name: order-service
    instance:
      id: ${HOSTNAME:unknown}
  exporter:
    otlp:
      endpoint: http://otel-collector:4318
      protocol: http/protobuf
  traces:
    sampler: parentbased_always_on   # 生产环境用 parentbased_traceidratio(0.1)
  propagators: tracecontext, baggage, b3
```

### 2.2 自定义 Span 标注

```java
@Service
public class OrderService {
    
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    @Autowired
    private Tracer tracer;

    @Autowired
    private InventoryClient inventoryClient;

    @WithSpan("createOrder")
    public OrderResult createOrder(@SpanAttribute("order.userId") Long userId,
                                   @SpanAttribute("order.itemCount") int itemCount) {
        
        // 创建子 Span
        Span inventorySpan = tracer.spanBuilder("inventory.check")
                .setAttribute("user.id", userId)
                .startSpan();
        
        try (Scope scope = inventorySpan.makeCurrent()) {
            boolean available = inventoryClient.checkStock(itemCount);
            inventorySpan.setAttribute("inventory.available", available);
            
            if (!available) {
                inventorySpan.setStatus(StatusCode.ERROR, "库存不足");
                throw new InsufficientStockException("库存不足");
            }
            // ... 业务逻辑
            return new OrderResult(true, "下单成功");
        } catch (Exception e) {
            inventorySpan.recordException(e);
            inventorySpan.setStatus(StatusCode.ERROR);
            throw e;
        } finally {
            inventorySpan.end();
        }
    }
}
```

### 2.3 异步方法追踪

```java
@Async
@WithSpan("sendNotification")
public CompletableFuture<Void> sendNotification(@SpanAttribute("orderId") Long orderId) {
    // 自动传播 TraceContext 到异步线程
    log.info("Sending notification for order: {}", orderId);
    return CompletableFuture.completedFuture(null);
}
```

### 2.4 消息队列传播 TraceContext

```java
// RabbitMQ 生产者
@Autowired
private Tracer tracer;

public void sendOrderMessage(Order order) {
    Span span = tracer.spanBuilder("order.publish").startSpan();
    try (Scope scope = span.makeCurrent()) {
        rabbitTemplate.convertAndSend("order.exchange", "order.created", order, message -> {
            // 注入 TraceContext 到消息头
            TextMapSetter<Message> setter = (carrier, key, value) ->
                carrier.getMessageProperties().setHeader(key, value);
            tracer.inject(span.getSpanContext(), TextMapPropagation.INSTANCE, message, setter);
            return message;
        });
    } finally {
        span.end();
    }
}

// RabbitMQ 消费者
@Component
public class OrderConsumer {
    
    @RabbitListener(queues = "order.queue")
    public void handleOrder(Order order, Message message) {
        // 从消息头提取 TraceContext
        TextMapGetter<Message> getter = new TextMapGetter<>() {
            @Override
            public Iterable<String> keys(Message carrier) {
                return carrier.getMessageProperties().getHeaders().keySet();
            }
            @Override
            public String get(Message carrier, String key) {
                return (String) carrier.getMessageProperties().getHeaders().get(key);
            }
        };
        
        SpanContext parentCtx = tracer.extract(TextMapPropagation.INSTANCE, message, getter);
        
        Span span = tracer.spanBuilder("order.process")
                .setParent(Context.current().with(Span.wrap(parentCtx)))
                .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            processOrder(order);
        } finally {
            span.end();
        }
    }
}
```

## 三、结构化日志（Structured Logging）

### 3.1 Logback + MDC 集成 TraceID

```xml
<!-- logback-spring.xml -->
<appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <!-- 自动包含 MDC 中的 trace_id, span_id, trace_flags -->
        <includeMdcKeyName>trace_id</includeMdcKeyName>
        <includeMdcKeyName>span_id</includeMdcKeyName>
        <includeMdcKeyName>trace_flags</includeMdcKeyName>
        <fieldNames>
            <timestamp>@timestamp</timestamp>
            <version>[ignore]</version>
            <logger>logger</logger>
            <levelValue>[ignore]</levelValue>
        </fieldNames>
    </encoder>
</appender>
```

```yaml
# application.yml 配置异步日志
logging:
  level:
    root: INFO
    com.yourcompany: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] [%X{trace_id:-}] %-5level %logger{36} - %msg%n"
```

### 3.2 关键业务事件日志

```java
@Service
public class PaymentService {
    
    private static final Logger auditLog = LoggerFactory.getLogger("AUDIT");
    private static final Logger bizLog = LoggerFactory.getLogger("BIZ");
    
    @WithSpan("payment.process")
    public PaymentResult processPayment(PaymentRequest request) {
        MDC.put("paymentId", request.getPaymentId().toString());
        MDC.put("userId", request.getUserId().toString());
        
        try {
            bizLog.info("开始支付处理, amount={}", request.getAmount());
            
            PaymentResult result = doPayment(request);
            
            // 审计日志：不可篡改的业务记录
            auditLog.info("PAYMENT_COMPLETED|userId={}|paymentId={}|amount={}|status={}",
                    request.getUserId(), request.getPaymentId(),
                    request.getAmount(), result.getStatus());
            
            return result;
        } catch (Exception e) {
            auditLog.error("PAYMENT_FAILED|userId={}|paymentId={}|error={}",
                    request.getUserId(), request.getPaymentId(), e.getMessage());
            throw e;
        } finally {
            MDC.clear();
        }
    }
}
```

## 四、指标（Metrics）

### 4.1 Micrometer + Prometheus

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  metrics:
    tags:
      application: ${spring.application.name}
    export:
      prometheus:
        enabled: true
        step: 15s
```

### 4.2 自定义业务指标

```java
@Component
public class OrderMetrics {
    
    private final Counter orderCreatedCounter;
    private final Timer orderProcessTimer;
    private final DistributionSummary orderAmountSummary;
    private final Gauge pendingOrdersGauge;

    public OrderMetrics(MeterRegistry registry) {
        this.orderCreatedCounter = Counter.builder("order.created.total")
                .description("总下单数")
                .tag("type", "all")
                .register(registry);

        this.orderProcessTimer = Timer.builder("order.process.duration")
                .description("订单处理耗时")
                .publishPercentiles(0.5, 0.95, 0.99)
                .register(registry);

        this.orderAmountSummary = DistributionSummary.builder("order.amount")
                .description("订单金额分布")
                .baseUnit("CNY")
                .register(registry);

        this.pendingOrdersGauge = Gauge.builder("order.pending.count",
                this, OrderMetrics::countPendingOrders)
                .description("待处理订单数")
                .register(registry);
    }

    public void recordOrderCreated() {
        orderCreatedCounter.increment();
    }

    public <T> T recordOrderProcess(Supplier<T> supplier) {
        return orderProcessTimer.record(supplier);
    }

    public void recordAmount(double amount) {
        orderAmountSummary.record(amount);
    }

    private int countPendingOrders() {
        // 从数据库计数
        return 0;
    }
}
```

## 五、Sampling 策略

生产环境全量采样开销过大，需要合理采样策略：

| 策略 | 场景 | 采样率 |
|------|------|--------|
| Head-based | 默认方案，请求入口决定 | 10% 即可 |
| Tail-based | 只保留慢请求/错误请求 | 动态 |
| Probabilistic | 均匀丢弃 | 1%~10% |
| Rate Limiting | 每秒固定 Span 数 | 100 span/s |

```yaml
# OTel Collector 配置 tail-based 采样
processors:
  tail_sampling:
    decision_wait: 30s
    num_traces: 1000
    expected_new_traces_per_sec: 200
    policies:
      - name: error-policy
        type: status_code
        properties:
          status_codes: [ERROR]
      - name: slow-policy
        type: latency
        properties:
          threshold_ms: 1000  # 保留超过 1s 的请求
      - name: default-policy
        type: probabilistic
        properties:
          sampling_percentage: 10
```

## 六、告警规则（Prometheus + AlertManager）

```yaml
# alert-rules.yml
groups:
  - name: microservice-alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) / rate(http_server_requests_seconds_count[5m]) > 0.05
        for: 3m
        labels:
          severity: critical
        annotations:
          summary: "服务 {{ $labels.service }} 错误率超过 5%"

      - alert: SlowResponseTime
        expr: histogram_quantile(0.99, rate(http_server_requests_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "服务 {{ $labels.service }} P99 响应时间超过 2s"

      - alert: HighPendingOrders
        expr: order_pending_count > 1000
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "待处理订单超过 1000"

      - alert: TraceDroppedRate
        expr: rate(otelcol_processor_dropped_spans[5m]) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "OTel Collector 丢弃 Span 速率异常"
```

## 七、服务拓扑自动发现

OpenTelemetry 通过 Span 中的 `peer.service` 和 `net.peer.name` 属性自动生成服务依赖图：

```java
// 自动记录下游依赖
Span.current().setAttribute("peer.service", "inventory-service");
Span.current().setAttribute("net.peer.name", "inventory-svc.ns.svc.cluster.local");
```

配合 Jaeger/Grafana Tempo 的 Service Graph 功能，可以自动生成服务调用拓扑图，识别：
- 扇出（fan-out）过多的服务
- 调用深度异常的链路
- 环形依赖
- 热点服务

## 注意事项

1. **TraceID Logging 隔离** — 确保异步线程（@Async、CompletableFuture）中 MDC 能自动传播。Spring Boot 4.1+ 的 `TaskExecutorCustomizer` 自动处理，旧版本需要手动配置 `ThreadPoolTaskExecutor` 的 task decorator。
2. **OTel Collector 高可用** — Collector 作为数据管道核心节点，必须多副本部署（至少 2 副本），并使用 `batch` 和 `memory_limiter` 处理器防止 OOM。
3. **过度采集** — 不要采样所有请求的入参/出参（Payload）。只在关键业务 Span 设置少量标签，大对象（>1KB）不应作为 Span 属性。
4. **存储成本** — 分布式追踪存储量巨大（每个 Span ~200B+标签）。建议：Trace 保留 7 天，Metrics 保留 30 天，Logs 保留 15 天（热数据）→ 归档到冷存储。
5. **采样率调整** — 生产环境从 `parentbased_always_on` 切换到 `parentbased_traceidratio(0.1)` 后，需要同时调整 Collector 的 tail-based 采样策略，确保错误链路不丢失。
6. **监控告警风暴** — 设置告警依赖关系（依赖服务故障时抑制下游告警），避免一个服务宕机引发 50+ 告警。
7. **隐私合规** — Span 属性和日志中不能包含用户密码、身份证号、银行卡号等敏感信息。使用 `Span.setAttribute("user.id", hash(userId))` 脱敏。
