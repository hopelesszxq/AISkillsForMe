---
name: gateway-reactive-spring6
description: Spring Cloud Gateway 与 Spring Boot 3.2+ 虚拟线程、gRPC 代理、Spring 6.1 HTTP Interface 集成
tags: [spring-cloud, gateway, virtual-threads, grpc, spring-boot-3, reactive]
---

## 概述

Spring Cloud Gateway 基于 WebFlux（Reactor/Netty），是响应式网关。Spring Boot 3.2+ 引入**虚拟线程**（Virtual Threads），虽然 Gateway 本身是响应式的，但其下游服务调用、过滤器等场景可以与虚拟线程配合优化。

## 虚拟线程与 Gateway 配合

### 核心原则

Gateway 的核心处理链路（Netty → Reactor）已经是异步非阻塞的，**不应直接启用虚拟线程做请求处理**。虚拟线程的用武之地在于：

1. **阻塞型过滤器**：调用数据库、远程 HTTP（非 WebClient 场景）
2. **后备 Fallback 处理**：熔断后的同步降级逻辑
3. **自定义 GatewayFilter**：包含阻塞 I/O

### 启用虚拟线程

```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true    # Spring Boot 3.2+ 启用虚拟线程
  cloud:
    gateway:
      # 保持 WebFlux 的 worker 线程为平台线程
      httpserver:
        worker-threads: 4    # 减少 Netty worker（虚拟线程接管阻塞）
```

### 自定义阻塞过滤器中用虚拟线程

```java
@Component
public class BlockingAuthFilter implements GatewayFilter, Ordered {

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            String token = exchange.getRequest().getHeaders()
                .getFirst("Authorization");
            
            // 使用虚拟线程处理阻塞 I/O
            return Mono.fromCallable(() -> {
                    // 这里会在虚拟线程中执行
                    return authService.validateToken(token);
                })
                .subscribeOn(Schedulers.boundedElastic())  // 虚拟线程调度器
                .flatMap(valid -> {
                    if (!valid) {
                        return unauthorized(exchange);
                    }
                    return chain.filter(exchange);
                });
        };
    }
}
```

### JDK 21 Virtual Threads + WebClient 优化

```java
// 不推荐：直接在虚拟线程中使用阻塞式 RestTemplate
// 推荐：继续使用 WebClient（响应式），由虚拟线程管理线程池

@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient(WebClient.Builder builder) {
        return builder
            .baseUrl("http://backend-service")
            .defaultHeader("Content-Type", "application/json")
            .build();
    }
}

// 在 Virtual Thread 环境下的同步包装（适配旧代码）
@Service
public class UserServiceClient {

    @Autowired
    private WebClient webClient;

    // 将响应式调用包装为同步，在虚拟线程中阻塞
    public User getUser(Long id) {
        return webClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .bodyToMono(User.class)
            .block();  // ✅ 虚拟线程中 block() 不会浪费平台线程
    }
}
```

## gRPC 代理在 Gateway

Spring Cloud Gateway 可以转发 gRPC 请求（基于 gRPC-Web 或 HTTP/2）。

### 方案一：gRPC-Web 代理

```yaml
# gateway 配置
spring:
  cloud:
    gateway:
      routes:
        - id: grpc-service
          uri: lb://grpc-backend
          predicates:
            - Path=/myapp.GrpcService/**
          # gRPC-Web 通过 HTTP/1.1 传输，可使用标准路由
          filters:
            - StripPrefix=0
            - name: CircuitBreaker
              args:
                name: grpcCB
                fallbackUri: forward:/fallback/grpc
```

### 方案二：HTTP/2 gRPC 透明代理

```java
// 自定义 gRPC 代理过滤器（基于 Netty HTTP/2）
@Component
public class GrpcProxyFilter implements GlobalFilter, Ordered {

    private final HttpClient httpClient;
    
    public GrpcProxyFilter() {
        this.httpClient = HttpClient.create()
            .protocol(HttpProtocol.H2)  // 强制 HTTP/2
            .wiretap("grpc-proxy", LogLevel.DEBUG, 
                AdvancedByteBufFormat.TEXTUAL);
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String path = request.getURI().getPath();
        
        // 识别 gRPC 请求（Content-Type: application/grpc）
        String contentType = request.getHeaders()
            .getFirst(HttpHeaders.CONTENT_TYPE);
        if (contentType == null || !contentType.startsWith("application/grpc")) {
            return chain.filter(exchange);  // 非 gRPC 请求，正常处理
        }

        // gRPC 请求代理
        String serviceName = extractServiceName(path);
        URI backendUri = lookupBackend(serviceName);
        
        return proxyGrpcRequest(exchange, backendUri);
    }

    private Mono<Void> proxyGrpcRequest(ServerWebExchange exchange, URI backendUri) {
        ServerHttpRequest request = exchange.getRequest();
        
        // 构建转发请求
        Flux<DataBuffer> body = request.getBody();
        
        return httpClient
            .post()
            .uri(backendUri.toString() + request.getURI().getRawPath())
            .headers(headers -> {
                headers.add(HttpHeaders.CONTENT_TYPE, "application/grpc");
                headers.add(HttpHeaders.TE, "trailers");
                // 复制必要的请求头
                request.getHeaders().forEach((key, values) -> {
                    if (!key.equalsIgnoreCase(HttpHeaders.HOST)) {
                        values.forEach(v -> headers.add(key, v));
                    }
                });
            })
            .send((req, response) -> {
                // 转发请求体
                return response.sendByteArray(
                    request.getBody().map(buf -> {
                        byte[] bytes = new byte[buf.readableByteCount()];
                        buf.read(bytes);
                        return bytes;
                    }));
            })
            .responseSingle((resp, body) -> {
                // 写入响应
                ServerHttpResponse gatewayResponse = exchange.getResponse();
                gatewayResponse.getHeaders()
                    .set(HttpHeaders.CONTENT_TYPE, "application/grpc");
                return body.asByteArray()
                    .flatMap(bytes -> gatewayResponse
                        .writeWith(Mono.just(gatewayResponse
                            .bufferFactory().wrap(bytes))));
            });
    }

    private String extractServiceName(String path) {
        // /myapp.GrpcService/MethodName → myapp.GrpcService
        return path.substring(1, path.lastIndexOf('/'));
    }

    private URI lookupBackend(String serviceName) {
        // 从注册中心获取 gRPC 服务地址
        return URI.create("grpc://backend-service:9090");
    }
}
```

## Spring 6.1 HTTP Interface

Spring 6.1 / Spring Boot 3.2 引入了声明式 HTTP Interface，可以替代 Feign/WebClient 实现**类型安全的 HTTP 客户端**，Gateway 过滤器中使用尤其方便。

### 定义 HTTP Interface

```java
// 用户服务接口
@HttpExchange("/api/users")
public interface UserClient {

    @GetExchange("/{id}")
    User getUser(@PathVariable Long id);

    @PostExchange
    User createUser(@RequestBody CreateUserRequest request);

    @GetExchange("/search")
    Flux<User> searchUsers(@RequestParam String name);
}

// 订单服务接口
@HttpExchange("/api/orders")
public interface OrderClient {

    @GetExchange("/{id}")
    Mono<Order> getOrder(@PathVariable Long id);

    @GetExchange("/user/{userId}")
    Flux<Order> getUserOrders(@PathVariable Long userId);
}
```

### 在 Gateway 过滤器中使用

```java
@Component
public class AggregationWithHttpInterfaceFilter implements GlobalFilter, Ordered {

    @Autowired
    private UserClient userClient;   // HTTP Interface
    
    @Autowired
    private OrderClient orderClient; // HTTP Interface

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();
        
        // 命中聚合接口
        if (path.equals("/api/aggregate/user-dashboard")) {
            Long userId = extractUserId(exchange);
            
            return Mono.zip(
                userClient.getUser(userId),
                orderClient.getUserOrders(userId).collectList()
            ).flatMap(tuple -> {
                Map<String, Object> result = new LinkedHashMap<>();
                result.put("user", tuple.getT1());
                result.put("orders", tuple.getT2());
                return writeJsonResponse(exchange, result);
            });
        }
        
        return chain.filter(exchange);
    }
}
```

### 注册 HTTP Interface 客户端

```java
@Configuration
public class HttpClientConfig {

    @Bean
    public UserClient userClient(WebClient.Builder builder) {
        return HttpServiceProxyFactory
            .builderFor(WebClientAdapter.create(builder.build()))
            .build()
            .createClient(UserClient.class);
    }

    @Bean
    public OrderClient orderClient(WebClient.Builder builder) {
        return HttpServiceProxyFactory
            .builderFor(WebClientAdapter.create(builder.baseUrl(
                "http://order-service").build()))
            .build()
            .createClient(OrderClient.class);
    }
}
```

### HTTP Interface vs OpenFeign 对比

| 特性 | OpenFeign | HTTP Interface |
|------|-----------|----------------|
| 运行时 | Spring Cloud | Spring 6.1+ 内置 |
| 响应式支持 | ❌ 不支持 | ✅ 支持 Mono/Flux |
| Gateway 兼容 | ❌ WebFlux 下不可用 | ✅ 基于 WebClient 可原生兼容 |
| 虚拟线程 | ❌ Feign 阻塞 | ✅ 可配合虚拟线程 |
| 服务发现 | Nacos/Eureka 自动集成 | 需手动配置 LoadBalancer |
| 复杂度 | 低（社区成熟） | 低（声明式） |

## 注意事项

1. **Gateway 不要启用全局虚拟线程**：WebFlux 的响应式模型不需要虚拟线程，强行启用反而引入 context 切换开销
2. **gRPC 代理性能**：HTTP/2 代理会消耗较多连接资源，建议启用连接池复用
3. **HTTP Interface 限流**：Gateway 中使用 HTTP Interface 时，同样需要配合 `RequestRateLimiter` 等限流措施
4. **负载均衡**：HTTP Interface 配合 `@LoadBalanced` WebClient 实现服务发现
   ```java
   @Bean
   @LoadBalanced
   public WebClient.Builder loadBalancedWebClientBuilder() {
       return WebClient.builder();
   }
   ```
5. **WebFlux 异常处理**：使用 `@ControllerAdvice` 时需实现 `ErrorWebExceptionHandler` 而非传统模式
6. **JDK 版本要求**：虚拟线程需要 JDK 21+，HTTP Interface 需要 Spring Boot 3.2+
