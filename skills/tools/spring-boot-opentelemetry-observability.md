---
name: spring-boot-opentelemetry-observability
description: Spring Boot OpenTelemetry 可观测性实战：自动配置、环境变量、OTLP 导出、K8s 集成与最佳实践
tags: [tools, spring-boot, opentelemetry, observability, tracing, metrics, kubernetes]
---

## 概述

Spring Boot 3.4+（以及 4.0/4.1）内置了对 OpenTelemetry 的原生支持，无需额外引入 OpenTelemetry SDK。通过 `spring-boot-starter-actuator` 结合 Micrometer 即可实现**分布式追踪**、**指标收集**和**日志关联**。

Spring Boot 4.1 进一步增强了 **OTel 环境变量支持**（`OTEL_RESOURCE_ATTRIBUTES`、`OTEL_SERVICE_NAME`、`OTEL_EXPORTER_OTLP_ENDPOINT` 等），使得在 Kubernetes 中通过环境变量配置可观测性更加便捷。

## 快速开始

### 1. 添加依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

### 2. 基础配置

```yaml
# application.yml
management:
  tracing:
    enabled: true
    sampling:
      probability: 0.1  # 采样率 10%，生产推荐 0.01~0.1
  opentelemetry:
    resource-attributes:
      environment: production
      region: cn-east-1
  endpoints:
    web:
      exposure:
        include: health,metrics,info
  metrics:
    tags:
      application: ${spring.application.name}
```

### 3. OTLP 导出配置（Spring Boot 4.1+）

```yaml
# 方式一：Spring Boot 属性（4.1+）
management:
  opentelemetry:
    exporter:
      otlp:
        endpoint: http://otel-collector:4318/v1/traces
        headers:
          authorization: Bearer ${OTEL_AUTH_TOKEN}
        compression: gzip
        timeout: 10s
```

```bash
# 方式二：OpenTelemetry 标准环境变量（4.1+ 原生支持）
export OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318
export OTEL_EXPORTER_OTLP_HEADERS=authorization=Bearer%20mytoken
export OTEL_RESOURCE_ATTRIBUTES=service.version=1.0.0,deployment.environment=production
export OTEL_SERVICE_NAME=order-service
```

**环境变量优先级**：Spring Boot 属性配置优先于 OTel 标准环境变量。

### 4. Spring Boot 4.0（旧方式，无 OTel 环境变量支持）

```yaml
# Spring Boot 4.0 需要手动配置 OpenTelemetry bean 才能支持完整环境变量
@Configuration
public class OpenTelemetryConfig {

    @Bean
    public OpenTelemetry openTelemetry() {
        // 4.0 不支持 OTEL_* 环境变量自动识别
        // 需手动从环境变量读取
        String endpoint = System.getenv("OTEL_EXPORTER_OTLP_ENDPOINT");
        // ...
        return OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .build();
    }
}
```

> **Spring Boot 4.1 改进**：支持 `OTEL_RESOURCE_ATTRIBUTES`、`OTEL_SERVICE_NAME`、`OTEL_EXPORTER_OTLP_ENDPOINT`、`OTEL_EXPORTER_OTLP_HEADERS` 等标准环境变量，自动合并到 Micrometer 资源属性中。

## 分布式追踪实战

### 自动追踪 HTTP 请求

Spring Boot 自动为所有 `@RestController` 和 WebFlux 端点创建 Span：

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping("/{id}")
    public Order getOrder(@PathVariable Long id) {
        // 方法自动生成 Span，名称: GET /api/orders/{id}
        return orderService.findOrder(id);
    }
}
```

### 自定义 Span

```java
import io.micrometer.observation.annotation.Observed;

@Service
public class OrderService {

    @Observed(name = "order.create",
              contextualName = "create-order",
              lowCardinalityKeyValues = {"db", "postgresql"})
    public Order createOrder(OrderRequest request) {
        // 这个方法会被自动追踪为 order.create Span
        // 标签: db=postgresql, class=OrderService, method=createOrder
        return orderRepository.save(request.toEntity());
    }
}
```

### 跨服务传播 Trace

Spring Boot 自动处理 W3C Trace Context 传播：

```yaml
# 默认已启用 W3C traceparent/tracestate 头传播
management:
  tracing:
    propagation:
      type: w3c  # 默认值，推荐
```

```java
// RestClient 自动携带 trace context（Spring Boot 4.1+）
@Autowired
private RestClient restClient;

public User getUser(Long userId) {
    // RestClient 自动插入 traceparent 头
    return restClient.get()
        .uri("http://user-service/users/{id}", userId)
        .retrieve()
        .body(User.class);
}
```

### 数据库查询追踪

```yaml
# 启用 JDBC 追踪（需要 spring-boot-starter-jdbc 或 spring-boot-starter-data-jpa）
management:
  tracing:
    jdbc:
      enabled: true  # 记录每个 SQL 查询的 Span
```

Spring Data JPA 查询会自动生成 Span：
```
Span: "UserRepository.findById" (class.method)
  ├── Span: "SELECT * FROM users WHERE id = ?" (JDBC query)
  └── Span: "redis cache lookup" (若启用缓存)
```

## 指标与监控

### 自动暴露的指标

```bash
# 查看应用暴露的所有指标
curl http://localhost:8080/actuator/metrics

# 查看 JVM 内存指标
curl http://localhost:8080/actuator/metrics/jvm.memory.used
```

### 自定义指标

```java
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;

@Service
public class PaymentService {

    private final Timer paymentTimer;

    public PaymentService(MeterRegistry registry) {
        this.paymentTimer = Timer.builder("payment.process.duration")
            .description("支付处理耗时")
            .tag("service", "order-service")
            .register(registry);
    }

    public PaymentResult process(PaymentRequest req) {
        return paymentTimer.record(() -> {
            // 实际支付处理逻辑
            return paymentClient.charge(req);
        });
    }
}
```

## Kubernetes 部署集成

### Deployment 环境变量注入

```yaml
# k8s-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    spec:
      containers:
        - name: order-service
          image: order-service:1.0.0
          env:
            - name: OTEL_SERVICE_NAME
              value: "order-service"
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector.monitoring:4318"
            - name: OTEL_RESOURCE_ATTRIBUTES
              value: "k8s.namespace=$(POD_NAMESPACE),k8s.pod=$(POD_NAME)"
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            # Spring Boot 4.1+ 自动识别以上 OTEL_* 环境变量
```

### 配合 OpenTelemetry Collector

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
  memory_limiter:
    check_interval: 1s
    limit_mib: 512

exporters:
  prometheus:
    endpoint: 0.0.0.0:8889
  otlp:
    endpoint: "jaeger:4317"
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
```

## 日志关联 Trace ID

```yaml
# application.yml - 将 traceId/spanId 注入日志
logging:
  pattern:
    level: "%5p [%X{traceId:-},%X{spanId:-}]"
```

日志输出示例：
```
2026-06-04 10:30:00.123  INFO [59f3a2b1e8c7d,9a8b7c6d5e4f] 12345 --- [nio-8080-exec-1] c.e.OrderService : Creating order #12345
```

## 与 Prometheus/Grafana 集成

```yaml
# application.yml - 暴露 Prometheus 指标端点
management:
  endpoints:
    web:
      exposure:
        include: prometheus,health,metrics
  prometheus:
    metrics:
      export:
        enabled: true
```

```yaml
# prometheus.yml - 采集配置
scrape_configs:
  - job_name: 'spring-boot-apps'
    metrics_path: '/actuator/prometheus'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

## OpenTelemetry SDK 环境变量速查

| 环境变量 | 作用 | Spring Boot 支持版本 |
|----------|------|---------------------|
| `OTEL_SERVICE_NAME` | 服务名称 | 3.4+ / 4.1+ 自动识别 |
| `OTEL_RESOURCE_ATTRIBUTES` | 资源属性（k=v,k2=v2） | 3.4+ / 4.1+ 自动识别 |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP 导出端点 | 4.1+ 自动识别 |
| `OTEL_EXPORTER_OTLP_HEADERS` | OTLP 请求头 | 4.1+ 自动识别 |
| `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` | 指标 OTLP 端点 | 4.1+（需自定义 Bean） |
| `OTEL_EXPORTER_OTLP_METRICS_HEADERS` | 指标请求头 | 4.1+（需自定义 Bean） |
| `OTEL_TRACES_SAMPLER` | 采样器类型 | 需自定义 Bean |
| `OTEL_TRACES_SAMPLER_ARG` | 采样器参数 | 需自定义 Bean |

> **注意**：非列表中的 OTel SDK 环境变量（如 `OTEL_TRACES_SAMPLER`）在 Spring Boot 中**不会自动生效**，需要自行提供 `OpenTelemetry` Bean。

## 注意事项

1. **采样率控制**：生产环境建议采样率 `0.01`~`0.1`（1%-10%），高流量服务可设置更低
2. **OTel Collector 必用**：不建议直接导出到后端（Jaeger/Zipkin），应通过 Collector 做批量和过滤
3. **自定义 Span 不要过度**：只在关键业务路径（创建订单、支付、核心 RPC）添加 `@Observed`
4. **4.0 vs 4.1 差异**：如果使用 Spring Boot 4.0，`OTEL_*` 环境变量需要自定义 Bean 才能完整支持
5. **日志关联**：`logging.pattern.level` 配置需与 `logback.xml` 配合，确保 MDC 上下文正确传递
6. **RestClient vs RestTemplate**：Spring Boot 4.1 的 RestClient 自动携带 trace context，旧版 RestTemplate 也支持但需要额外配置
