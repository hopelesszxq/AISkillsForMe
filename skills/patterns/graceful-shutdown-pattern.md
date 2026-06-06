---
name: graceful-shutdown-pattern
description: 微服务优雅关闭模式：Spring Boot Graceful Shutdown、K8s PreStop Hook、Readiness/Liveness 探针配合与流量摘除
tags: [patterns, microservice, graceful-shutdown, kubernetes, health-check, spring-boot]
---

## 概述

微服务优雅关闭（Graceful Shutdown）是生产环境的基础要求。当 Pod 被销毁或服务下线时，应确保**正在处理的请求完成**、**新流量不再路由**、**资源正确释放**。Spring Boot + Kubernetes 组合需要多方配合才能实现真正的零停机滚动更新。

## 一、优雅关闭的核心时序

```text
K8s 滚动更新时序：
1. K8s 发送 SIGTERM 信号
2. Readiness 探针变为 NotReady → 从 Service Endpoint 摘除
3. PreStop Hook 执行（等待已有请求完成）
4. Spring Boot Graceful Shutdown（等待活跃请求）
5. 超时后 SIGKILL 强制终止
```

**目标**：4 个环节配合，确保零丢请求。

## 二、Spring Boot 配置

### 启用 Graceful Shutdown（Spring Boot 2.3+）

```yaml
# application.yml
server:
  shutdown: graceful         # 启用优雅关闭（默认 immediate）
  netty:
    shutdown-timeout: 30s    # Netty 关闭超时

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # 每个关闭阶段超时

# 配合 Undertow/Tomcat
# server.tomcat.connection-timeout: 5s
# server.undertow.eager-filter-init: false
```

### 自定义关闭逻辑

```java
@Component
public class GracefulShutdownHandler {

    @EventListener
    public void onShutdown(ContextClosedEvent event) {
        log.info("=== 开始优雅关闭 ===");

        // 1. 标记服务不可用（自定义健康检查）
        AppHealthIndicator.setAcceptingTraffic(false);

        // 2. 等待正在处理的请求完成
        RequestTracker.drain(30, TimeUnit.SECONDS);

        // 3. 关闭消息消费者
        messageConsumer.stop();

        // 4. 关闭线程池
        executorService.shutdown();
        try {
            executorService.awaitTermination(30, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        // 5. 关闭数据库连接池
        HikariDataSource ds = (HikariDataSource) dataSource;
        if (ds != null) ds.close();

        log.info("=== 优雅关闭完成 ===");
    }
}
```

### 请求追踪器

```java
@Component
public class RequestTracker {

    private final AtomicInteger activeRequests = new AtomicInteger(0);
    private final CountDownLatch drainLatch = new CountDownLatch(1);

    public void increment() {
        activeRequests.incrementAndGet();
    }

    public void decrement() {
        if (activeRequests.decrementAndGet() <= 0) {
            drainLatch.countDown();
        }
    }

    public void drain(long timeout, TimeUnit unit) {
        log.info("等待 {} 个活跃请求完成...", activeRequests.get());
        try {
            if (!drainLatch.await(timeout, unit)) {
                log.warn("等待超时，仍有 {} 个未完成请求", activeRequests.get());
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    // 使用示例：过滤器或拦截器
    // @Bean
    // public Filter requestTrackingFilter() {
    //     return (request, response, chain) -> {
    //         increment();
    //         try {
    //             chain.doFilter(request, response);
    //         } finally {
    //             decrement();
    //         }
    //     };
    // }
}
```

## 三、Kubernetes 探针配合

### Readiness + Liveness + Startup 三探针

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # 最多多启动 1 个新 Pod
      maxUnavailable: 0    # 保证最少 N-1 个 Pod 可用（零停机）
  template:
    spec:
      terminationGracePeriodSeconds: 60  # 总关闭超时（必须 > 各阶段总和）
      containers:
        - name: app
          image: order-service:latest
          ports:
            - containerPort: 8080
          startupProbe:           # 启动探针（慢启动时很有用）
            httpGet:
              path: /actuator/health/startup
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 30   # 最多等 150 秒
          readinessProbe:          # 就绪探针（控制流量接入/摘除）
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 2     # 探针失败后快速摘除流量
          livenessProbe:           # 存活探针（重启异常 Pod）
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            periodSeconds: 15
            failureThreshold: 3     # 45 秒无响应则重启
          lifecycle:
            preStop:
              exec:
                command:
                  - sh
                  - -c
                  - |-
                    # 1. 立即标记 Readiness 为 NotReady（摘除流量）
                    # 2. 等待 30 秒让已有请求完成
                    # 注意：SIGTERM 在 preStop 完成后才发送
                    sleep 5 && echo "PreStop done"
```

### 探针响应实现

```java
@RestController
public class HealthController {

    private volatile boolean ready = true;

    // 启动探针 — 应用初始化完毕即可
    @GetMapping("/actuator/health/startup")
    public ResponseEntity<Map<String, String>> startup() {
        return ResponseEntity.ok(Map.of("status", "UP"));
    }

    // 就绪探针 — 摘除流量时返回 DOWN
    @GetMapping("/actuator/health/readiness")
    public ResponseEntity<Map<String, String>> readiness() {
        if (ready) {
            return ResponseEntity.ok(Map.of("status", "UP"));
        }
        // 返回 503 让 K8s 摘除流量
        return ResponseEntity.status(503).body(Map.of("status", "DOWN"));
    }

    // 存活探针 — 死锁/内存溢出时返回 DOWN
    @GetMapping("/actuator/health/liveness")
    public ResponseEntity<Map<String, String>> liveness() {
        // 检查关键依赖
        if (dataSourceIsHealthy() && redisIsHealthy()) {
            return ResponseEntity.ok(Map.of("status", "UP"));
        }
        return ResponseEntity.status(503).body(Map.of("status", "DOWN"));
    }

    // 关闭时自动摘除流量
    @EventListener
    public void onShutdown(ContextClosedEvent event) {
        this.ready = false;
    }
}
```

## 四、Spring Boot Actuator 三探针自动配置

Spring Boot 4.x 内置了三个健康端点：

```yaml
# application.yml
management:
  endpoint:
    health:
      probes:
        enabled: true          # 启用 /health/startup 和 /health/readiness
      show-details: always
  health:
    readinessstate:
      enabled: true
    livenessstate:
      enabled: true
```

```yaml
# K8s 探针配置（对应 Actuator 端点）
readinessProbe:
  httpGet:
    path: /actuator/health/readiness  # 返回 {"status":"UP"} 或 503
livenessProbe:
  httpGet:
    path: /actuator/health/liveness   # 返回 {"status":"UP"} 或 503
startupProbe:
  httpGet:
    path: /actuator/health/startup    # 应用启动完毕前返回 503
```

## 五、消息队列优雅关闭

### RabbitMQ 消费者

```java
@Component
public class GracefulRabbitConsumer {

    @EventListener
    public void onShutdown(ContextClosedEvent event) {
        // 停止接受新消息
        rabbitListenerEndpointRegistry.stop();
        // 等待正在处理的消息完成
        messageProcessor.drain(30, TimeUnit.SECONDS);
    }
}
```

### Kafka 消费者

```java
@Component
public class GracefulKafkaConsumer {

    @EventListener
    public void onShutdown(ContextClosedEvent event) {
        // Kafka Consumer 会自动触发 rebalance
        // 确保处理中的记录完成即可
        pendingRecords.drain(30, TimeUnit.SECONDS);
    }
}
```

## 六、Spring Cloud Gateway 优雅关闭

Gateway 作为入口网关，关闭时需要先停止接受新连接：

```yaml
# Gateway 专用
spring:
  cloud:
    gateway:
      httpclient:
        pool:
          max-connections: 1000
          acquire-timeout: 5000
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

## 七、完整配置模板

```yaml
# application.yml — 微服务优雅关闭完整配置
server:
  shutdown: graceful
  netty:
    shutdown-timeout: 30s

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
  task:
    execution:
      shutdown:
        await-termination: true
        await-termination-period: 30s

management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    readinessstate:
      enabled: true
    livenessstate:
      enabled: true
```

```yaml
# K8s Deployment — 优雅关闭配置
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: app
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]
          startupProbe:
            httpGet: { path: /actuator/health/startup, port: 8080 }
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 30
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            periodSeconds: 15
            failureThreshold: 3
```

## 注意事项

1. **时间设置配合**：`terminationGracePeriodSeconds` 必须大于 `PreStop Sleep + 关闭超时`，否则 SIGKILL 提前触发
2. **maxUnavailable=0**：滚动更新时确保至少 N-1 个 Pod 在线，但这会增加更新耗时
3. **Readiness 先于 Liveness**：Readiness 摘除流量后，Liveness 失败不会影响用户体验（因为已无流量）
4. **PreStop vs Shutdown Hook**：preStop 在 SIGTERM 之前执行，用于摘除流量；Spring 的 `@PreDestroy` / `ContextClosedEvent` 在 SIGTERM 后执行，用于释放资源
5. **长连接处理**：WebSocket / gRPC Stream 连接需要特殊处理—在 preStop 中通知客户端主动重连，或等待超时断开
6. **DB 连接池**：HikariCP 在 `shutdown()` 时会等待活跃连接完成，但建议配合 `ContextClosedEvent` 手动管理
7. **测试方法**：使用 `kubectl delete pod --wait=false` 手动关闭 Pod，观察日志确认请求是否完整处理
