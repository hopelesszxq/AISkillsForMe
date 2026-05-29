---
name: gateway-performance-tuning
description: Spring Cloud Gateway 性能调优实战：Netty 参数调优、连接池、线程模型、吞吐量优化与监控
tags: [spring-cloud, gateway, performance, netty, throughput, tuning, reactive]
---

## 概述

Spring Cloud Gateway 基于 WebFlux（Reactor + Netty），性能调优重点是 Netty 的 I/O 线程模型、HTTP 客户端连接池和响应式背压管理。本文覆盖从 JVM 到 Netty 层的全链路调优。

## 1. JVM 参数优化

```bash
# 推荐 JVM 参数（8C16G 网关实例）
JAVA_OPTS="
  -Xms8g -Xmx8g
  -XX:+UseZGC                    # Java 21+ 推荐 ZGC，低延迟
  -XX:ConcGCThreads=2            # 并发 GC 线程数
  -XX:ParallelGCThreads=4        # 并行 GC 线程数
  
  # 或者 Java 17 使用 G1GC
  # -XX:+UseG1GC
  # -XX:MaxGCPauseMillis=50
  
  -Xlog:gc*:file=/var/log/gateway-gc.log:time,level,tags:filecount=10,filesize=50m
  
  -Dreactor.netty.pool.maxConnections=1000
  -Dreactor.netty.ioWorkerCount=4
  -Dreactor.netty.workerCount=8
  
  -Djava.net.preferIPv4Stack=true
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/var/log/gateway-heapdump.hprof
"
```

## 2. Netty 线程模型调优

Spring Cloud Gateway 底层的 Netty 分为两组线程：**Boss Group**（接受连接）和 **Worker Group**（处理 I/O）。

### 配置文件

```yaml
server:
  # 绑定端口
  port: 8080
  
  netty:
    # 连接超时（毫秒）
    connection-timeout: 5000
    
    # Boss 线程数（接受连接，通常 1 即可，高并发可设为 2-4）
    # 默认值：1
    # 设置方式：系统属性 reactor.netty.bossCount
    # 或通过 NettyRuntime.availableProcessors() 自动配置
    
    # Worker 线程数（处理 I/O，默认 = CPU 核数 * 2）
    # 设置方式：系统属性 reactor.netty.ioWorkerCount
    # 公式：cpuCores * 2 是安全起始点
    
    # 事件循环组配置
    reactor:
      netty:
        # Worker 线程数（覆盖默认值）
        ioWorkerCount: 8
        # Boss 线程数
        bossCount: 2
        
        # 最大连接数
        maxConnections: 8192
        
        # 连接池获取超时
        poolAcquireTimeout: 5000
        
        # 连接空闲超时
        poolIdleTimeout: 30000
        
        # 连接最大生命时间
        poolMaxLifeTime: 120000
```

### 代码方式调优

```java
import io.netty.channel.EventLoopGroup;
import io.netty.channel.epoll.EpollEventLoopGroup;
import io.netty.channel.kqueue.KqueueEventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import org.springframework.boot.web.embedded.netty.NettyReactiveWebServerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class NettyTuningConfig {

    @Bean
    public NettyReactiveWebServerFactory nettyFactory() {
        NettyReactiveWebServerFactory factory = new NettyReactiveWebServerFactory();

        // Linux 下使用 Epoll（比 NIO 更高效）
        if (isLinux()) {
            factory.addServerCustomizers(server -> {
                // 设置 Boss 和 Worker 线程组
                EventLoopGroup bossGroup = new EpollEventLoopGroup(2);
                EventLoopGroup workerGroup = new EpollEventLoopGroup(
                    Runtime.getRuntime().availableProcessors() * 2
                );
                server.setBossGroup(bossGroup);
                server.setWorkerGroup(workerGroup);
            });
        }

        // TCP 参数优化
        factory.addServerCustomizers(server -> {
            server.tcpConfiguration(tcp -> {
                tcp.bootstrap(bootstrap -> {
                    // 开启 TCP_NODELAY（禁用 Nagle 算法）
                    bootstrap.option(ChannelOption.TCP_NODELAY, true);
                    // 开启 TCP KeepAlive
                    bootstrap.option(ChannelOption.SO_KEEPALIVE, true);
                    // 降低 TIME_WAIT 数
                    bootstrap.option(ChannelOption.SO_REUSEADDR, true);
                    // 接收缓冲区
                    bootstrap.option(ChannelOption.SO_RCVBUF, 65536);
                    // 发送缓冲区
                    bootstrap.option(ChannelOption.SO_SNDBUF, 65536);
                    // 等待队列长度
                    bootstrap.option(ChannelOption.SO_BACKLOG, 1024);
                    return bootstrap;
                });
                return tcp;
            });
        });

        return factory;
    }

    private boolean isLinux() {
        String os = System.getProperty("os.name").toLowerCase();
        return os.contains("linux");
    }
}
```

```xml
<!-- 添加 Epoll 依赖（Linux 下自动使用） -->
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-transport-native-epoll</artifactId>
    <classifier>linux-x86_64</classifier>
</dependency>
```

## 3. HTTP 客户端连接池调优

Gateway 转发请求使用 Reactor Netty HTTP 客户端，连接池是关键瓶颈。

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        # 连接池配置
        pool:
          # 连接池类型：ELASTIC / FIXED / DISABLED
          # ELASTIC：动态扩展（默认），连接数无上限，适合突发流量
          # FIXED：固定连接数，适合稳定流量，推荐生产使用
          type: FIXED
          
          # 最大连接数（FIXED 模式生效）
          max-connections: 500
          
          # 每个远程主机的最大连接数
          max-connections-per-host: 100
          
          # 连接获取超时（毫秒，默认 45000）
          acquire-timeout: 5000
          
          # 连接最大空闲时间（毫秒，默认 null 不限制）
          max-idle-time: 30000
          
          # 连接最大生命时间（毫秒，默认 null 不限制）
          max-life-time: 120000
        
        # SSL 配置
        ssl:
          use-insecure-trust-manager: false
          handshake-timeout: 10000
          close-notify-flush-timeout: 3000
        
        # 连接超时（毫秒）
        connect-timeout: 3000
        # 响应超时（毫秒）
        response-timeout: 30s
        
        # HTTP 压缩
        compression: true
        
        # 代理协议支持
        # protocol: HTTP/1.1
        # protocol: HTTP/2 (需要额外配置)
```

### 连接池类型对比

| 类型 | 行为 | 适用场景 |
|------|------|---------|
| **ELASTIC**（默认） | 按需创建，无上限，空闲 60s 回收 | 低流量、测试环境 |
| **FIXED** | 固定 max-connections，超时等待或失败 | **生产推荐**，防止下游被打爆 |
| **DISABLED** | 每次新建连接，用完即关闭 | 特殊场景（长连接不可用） |

## 4. 路由级别超时控制

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: critical-service
          uri: lb://critical-service
          predicates:
            - Path=/api/order/**
          metadata:
            # 路由级响应超时（覆盖全局值）
            response-timeout: 5000
            
        - id: slow-service
          uri: lb://slow-service
          predicates:
            - Path=/api/report/**
          metadata:
            # 允许更长的超时
            response-timeout: 60000
```

```java
// 编程方式设置路由超时
@Bean
public RouteLocator customRoutes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("order-service", r -> r
            .path("/api/order/**")
            .filters(f -> f
                .requestRateLimiter(config -> config
                    .setRateLimiter(redisRateLimiter)
                    .setKeyResolver(userKeyResolver)
                )
                .retry(retryConfig -> retryConfig
                    .setRetries(3)
                    .setBackoff(Duration.ofMillis(200), Duration.ofSeconds(2), 2, true)
                ))
            .uri("lb://order-service")
            .metadata("response-timeout", 5000))
        .build();
}
```

## 5. 请求体大小与缓冲优化

```yaml
spring:
  codec:
    max-in-memory-size: 512KB      # 请求体最大缓冲区（默认 256KB）
  
  cloud:
    gateway:
      # 请求体缓冲区大小
      request-body-max-in-memory-size: 512KB
      
      # 是否缓存请求体（用于需要重读请求体的过滤器）
      # 开启会额外消耗内存，仅在需要时才开启
      # request-body-cache: true
```

## 6. 过滤器链性能优化

### 过滤器执行顺序

```java
// 高优先级过滤器（靠近 I/O 层）- 应该轻量快速
@Component
public class RequestTimingFilter implements GatewayFilter, Ordered {
    @Override
    public int getOrder() {
        return -100;  // 高优先级，最先执行
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        long start = System.nanoTime();
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            long elapsed = (System.nanoTime() - start) / 1_000_000;
            log.info("Request {} {} completed in {}ms",
                exchange.getRequest().getMethod(),
                exchange.getRequest().getURI().getPath(),
                elapsed);
        }));
    }
}
```

### 过滤器执行顺序建议

| 优先级 | 过滤器类型 | 说明 |
|--------|-----------|------|
| **HIGHEST** (> 0) | 安全、鉴权、限流 | 尽早拒绝无效请求 |
| **NORMAL** (0) | 请求改写、Header 处理 | 无阻塞操作 |
| **LOWEST** (< 0) | 日志、指标、Trace | 后置处理 |

### 避免阻塞操作

```java
// ❌ 错误：在过滤器中使用阻塞调用
@Component
public class BadFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 阻塞当前 Netty 线程！
        User user = userService.findById(blockingCall(exchange));
        return chain.filter(exchange);
    }
}

// ✅ 正确：使用响应式链
@Component
public class GoodFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return reactiveUserService.findById(getUserId(exchange))
            .flatMap(user -> {
                exchange.getAttributes().put("user", user);
                return chain.filter(exchange);
            });
    }
}
```

## 7. 性能监控与压测

### Actuator 监控

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,gateway
  metrics:
    tags:
      application: api-gateway
    export:
      prometheus:
        enabled: true
```

```java
// 自定义 Gateway 指标
@Bean
public MeterRegistryCustomizer<MeterRegistry> gatewayMetrics() {
    return registry -> {
        // 路由请求计数
        Counter.builder("gateway.requests.total")
            .description("Total gateway requests")
            .register(registry);
        // 路由延迟直方图
        Timer.builder("gateway.request.duration")
            .description("Request duration")
            .publishPercentiles(0.5, 0.9, 0.95, 0.99)
            .register(registry);
    };
}
```

### wrk 压测

```bash
# 网关基准压测
wrk -t8 -c200 -d60s --latency http://gateway:8080/api/test

# 保持长连接
wrk -t8 -c200 -d120s -H "Connection: keep-alive" \
  --latency http://gateway:8080/api/health

# 压测报告关注指标：
# - Requests/sec: 吞吐量
# - Latency (p99): 尾延迟（应 < 100ms）
# - Socket errors: 连接错误数
```

### 关键监控指标

| 指标 | 预警阈值 | 说明 |
|------|---------|------|
| gateway.requests.total | — | 请求总量 |
| gateway.request.duration.p99 | > 500ms | 尾延迟过高 |
| reactor.netty.connection.provider.pending.acquire | > 0 | 连接池耗尽，请求等待连接 |
| reactor.netty.bytebuf.allocator.memory.heap.used | > 80% | 堆内存过高 |
| jvm.gc.pause | > 200ms | GC 暂停时间过长 |
| system.load.average | > CPU核数 * 0.7 | 系统负载过高 |

## 8. 常见性能问题排查

### 问题：高并发下延迟急剧上升

```bash
# 检查连接池状态
curl http://localhost:8080/actuator/metrics/reactor.netty.connection.provider.pending.acquire

# 检查 GC 情况
jstat -gcutil <pid> 1000

# 抓取线程栈
jstack <pid> | grep -E "reactor-http|epoll" | head -30
```

### 问题：连接被重置（Connection Reset）

```yaml
# 尝试降低连接池超时，快速释放异常连接
spring:
  cloud:
    gateway:
      httpclient:
        pool:
          max-idle-time: 15000     # 缩短空闲超时
          max-life-time: 60000     # 缩短生命周期
        connect-timeout: 2000
        response-timeout: 10s
```

### 问题：内存持续增长

```yaml
# 限制请求体大小
spring:
  codec:
    max-in-memory-size: 256KB

# 启用请求体回收
spring:
  cloud:
    gateway:
      request-body:
        cache: false  # 不缓存请求体
```

## 注意事项

- **不要**在过滤器中使用阻塞调用（如 `Thread.sleep()`、`Future.get()`），会阻塞 Netty 线程池
- **建议**生产环境使用 `FIXED` 连接池类型，避免 ELASTIC 无限制创建连接打爆下游
- **Epoll/NIO 选择**：Linux 下优先使用 `netty-transport-native-epoll`，性能提升 10-30%
- **HTTP/2**：开启 HTTP/2 可减少连接数，但需确认下游均支持
- **预热**：网关上线前进行流量预热，让 JIT 编译和连接池充分初始化
- **版本选择**：Spring Boot 3.4+ 对响应式性能有显著优化，建议使用最新稳定版
- **不要使用 `spring-boot-starter-web`**（Tomcat）+ Gateway 共用，会导致冲突
