---
name: gateway-filter
description: Spring Cloud Gateway 自定义过滤器、全局过滤器与限流过滤链实战
tags: [spring-cloud, gateway, filter, rate-limit, circuit-breaker]
---

## 过滤器体系

Spring Cloud Gateway 的过滤器分为三类：

| 类型 | 接口 | 作用范围 | 典型用途 |
|------|------|---------|---------|
| **GatewayFilter** | 单一/条件 | 绑定到特定路由 | 鉴权、请求改写、限流 |
| **GlobalFilter** | 全局 | 所有路由 | 日志、安全、请求追踪 |
| **Ordered** | 排序 | 执行顺序 | 定义过滤器链顺序 |

## GatewayFilter（路由级过滤器）

### 自定义鉴权过滤器

```java
@Component
public class AuthGatewayFilterFactory
        extends AbstractGatewayFilterFactory<AuthGatewayFilterFactory.Config> {

    public AuthGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();

            // 1. 白名单路径跳过鉴权
            String path = request.getURI().getPath();
            for (String exclude : config.getExcludePaths()) {
                if (path.startsWith(exclude)) {
                    return chain.filter(exchange);
                }
            }

            // 2. 检查 token
            String token = request.getHeaders()
                .getFirst(config.getTokenHeader());

            if (token == null || token.isBlank()) {
                return unauthorized(exchange, "缺少认证 Token");
            }

            // 3. 验证并解析 token（伪代码）
            try {
                Claims claims = JwtUtil.parseToken(token);
                // 将用户信息注入请求头，传递给下游服务
                exchange = exchange.mutate()
                    .request(r -> r.header("X-User-Id", claims.getSubject())
                                    .header("X-User-Name",
                                        claims.get("name", String.class)))
                    .build();
            } catch (Exception e) {
                return unauthorized(exchange, "Token 无效或已过期");
            }

            return chain.filter(exchange);
        };
    }

    private Mono<Void> unauthorized(ServerWebExchange exchange, String msg) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.UNAUTHORIZED);
        response.getHeaders().setContentType(MediaType.APPLICATION_JSON);
        byte[] body = ("{\"code\":401,\"msg\":\"" + msg + "\"}").getBytes();
        DataBuffer buffer = response.bufferFactory().wrap(body);
        return response.writeWith(Mono.just(buffer));
    }

    @Data
    public static class Config {
        private String tokenHeader = "Authorization";
        private List<String> excludePaths = new ArrayList<>();
    }
}
```

```yaml
# application.yml 使用
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - name: Auth
              args:
                tokenHeader: Authorization
                excludePaths: /api/user/login, /api/user/register, /api/user/refresh
```

### 请求/响应改写过滤器

```java
@Component
public class RequestRewriteGatewayFilterFactory
        extends AbstractGatewayFilterFactory<RequestRewriteGatewayFilterFactory.Config> {

    public RequestRewriteGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();

            // 1. 添加公共请求头
            ServerHttpRequest mutatedRequest = request.mutate()
                .header("X-Gateway-Version", config.getVersion())
                .header("X-Request-Id", UUID.randomUUID().toString())
                .header("X-Forwarded-For", Objects.requireNonNullElse(
                    request.getRemoteAddress() != null
                        ? request.getRemoteAddress().getAddress().getHostAddress()
                        : "unknown", "unknown"))
                .build();

            // 2. 改写请求路径前缀
            if (config.getStripPrefix() > 0) {
                String path = mutatedRequest.getURI().getRawPath();
                String newPath = "/" + Arrays.stream(path.split("/"))
                    .skip(config.getStripPrefix() + 1)
                    .collect(Collectors.joining("/"));
                mutatedRequest = mutatedRequest.mutate()
                    .path(newPath)
                    .build();
            }

            return chain.filter(exchange.mutate().request(mutatedRequest).build());
        };
    }

    @Data
    public static class Config {
        private String version = "v1";
        private int stripPrefix = 0;
    }
}
```

## GlobalFilter（全局过滤器）

### 请求耗时日志过滤器

```java
@Component
@Slf4j
public class RequestTimingGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();
        String method = exchange.getRequest().getMethod().name();
        long startTime = System.currentTimeMillis();

        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            long elapsed = System.currentTimeMillis() - startTime;
            HttpStatus status = exchange.getResponse().getStatusCode();

            log.info("[GATEWAY] {} {} → {} ({}ms)",
                method, path, status != null ? status.value() : "??", elapsed);

            // 慢请求告警（> 3s）
            if (elapsed > 3000) {
                log.warn("[SLOW] {} {} took {}ms, status={}",
                    method, path, elapsed, status);
            }
        }));
    }

    @Override
    public int getOrder() {
        return -1;  // 高优先级
    }
}
```

### IP 黑白名单过滤器

```java
@Component
public class IpFilterGlobalFilter implements GlobalFilter, Ordered {

    private final Set<String> blacklist = Set.of(
        "10.0.0.100", "192.168.1.200"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String ip = Objects.requireNonNullElse(
            exchange.getRequest().getRemoteAddress(),
            new InetSocketAddress(0)).getAddress().getHostAddress();

        if (blacklist.contains(ip)) {
            ServerHttpResponse response = exchange.getResponse();
            response.setStatusCode(HttpStatus.FORBIDDEN);
            return response.setComplete();
        }

        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -100;  // 最先执行
    }
}
```

### 统一响应包装过滤器

```java
@Component
public class ResponseWrapperGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpResponse originalResponse = exchange.getResponse();
        DataBufferFactory bufferFactory = originalResponse.bufferFactory();

        // 只包装 JSON 响应
        if (!exchange.getRequest().getURI().getPath().startsWith("/api/")) {
            return chain.filter(exchange);
        }

        ServerHttpResponseDecorator decoratedResponse = new ServerHttpResponseDecorator(originalResponse) {
            @Override
            public Mono<Void> writeWith(Publisher<? extends DataBuffer> body) {
                if (body instanceof Flux) {
                    Flux<? extends DataBuffer> fluxBody = Flux.from(body);
                    return super.writeWith(
                        fluxBody.buffer().map(dataBuffers -> {
                            // 将原始响应包装成统一格式
                            byte[] originalContent = new byte[0];
                            for (DataBuffer db : dataBuffers) {
                                originalContent = concatBytes(originalContent,
                                    readBytes(db.readableByteCount(), db));
                            }
                            String originalJson = new String(originalContent, StandardCharsets.UTF_8);
                            String wrappedJson = "{\"code\":200,\"msg\":\"success\",\"data\":"
                                + originalJson + "}";
                            return bufferFactory.wrap(wrappedJson.getBytes());
                        })
                    );
                }
                return super.writeWith(body);
            }
        };

        return chain.filter(exchange.mutate().response(decoratedResponse).build());
    }

    private byte[] concatBytes(byte[] a, byte[] b) {
        byte[] result = new byte[a.length + b.length];
        System.arraycopy(a, 0, result, 0, a.length);
        System.arraycopy(b, 0, result, a.length, b.length);
        return result;
    }

    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;  // 最后执行
    }
}
```

## 限流过滤器实战

### 基于 Redis 的令牌桶限流

```java
@Component
public class RateLimitGatewayFilterFactory
        extends AbstractGatewayFilterFactory<RateLimitGatewayFilterFactory.Config> {

    @Autowired
    private StringRedisTemplate redisTemplate;

    public RateLimitGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            String key = resolveKey(exchange, config);

            // Lua 脚本实现令牌桶
            String luaScript = """
                local key = KEYS[1]
                local capacity = tonumber(ARGV[1])     -- 桶容量
                local rate = tonumber(ARGV[2])         -- 每秒补充速率
                local now = tonumber(ARGV[3])
                local requested = tonumber(ARGV[4])    -- 请求消耗的令牌数

                local ttl = math.floor(rate)
                local lastBucket = redis.call('HMGET', key, 'last_bucket', 'last_time')
                local lastBucketCount = tonumber(lastBucket[1] or capacity)
                local lastTime = tonumber(lastBucket[2] or now)

                -- 计算时间间隔内补充的令牌
                local elapsed = math.max(now - lastTime, 0)
                local tokensToAdd = math.floor(elapsed * rate)
                local currentBucket = math.min(lastBucketCount + tokensToAdd, capacity)

                if currentBucket >= requested then
                    redis.call('HSET', key, 'last_bucket', currentBucket - requested, 'last_time', now)
                    redis.call('EXPIRE', key, ttl * 2)
                    return 1  -- 允许通过
                else
                    return 0  -- 限流
                end
                """;

            DefaultRedisScript<Long> script = new DefaultRedisScript<>(luaScript, Long.class);

            Long result = redisTemplate.execute(script, List.of(key),
                String.valueOf(config.getCapacity()),
                String.valueOf(config.getRate()),
                String.valueOf(System.currentTimeMillis() / 1000),
                "1");

            if (result == null || result == 0) {
                ServerHttpResponse response = exchange.getResponse();
                response.setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
                response.getHeaders().add("Retry-After", "1");
                return response.writeWith(Mono.just(
                    response.bufferFactory().wrap(
                        "{\"code\":429,\"msg\":\"请求过于频繁，请稍后再试\"}".getBytes()
                    )
                ));
            }

            return chain.filter(exchange);
        };
    }

    private String resolveKey(ServerWebExchange exchange, Config config) {
        return switch (config.getKeyType()) {
            case "ip" -> Objects.requireNonNull(
                exchange.getRequest().getRemoteAddress()).getAddress().getHostAddress();
            case "user" -> exchange.getRequest().getHeaders()
                .getFirst("X-User-Id");
            case "path" -> exchange.getRequest().getURI().getPath();
            default -> "global";
        };
    }

    @Data
    public static class Config {
        private String keyType = "ip";     // ip / user / path / global
        private int capacity = 10;         // 桶容量
        private int rate = 1;              // 每秒补充速率
    }
}
```

```yaml
# 路由级限流配置
spring:
  cloud:
    gateway:
      routes:
        - id: api-rate-limit
          uri: lb://backend-service
          predicates:
            - Path=/api/**
          filters:
            - name: RateLimit
              args:
                keyType: ip
                capacity: 100
                rate: 20
```

### Sentinel Gateway 限流集成

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    sentinel:
      filter:
        enabled: true
      transport:
        dashboard: localhost:8080
      scg:
        fallback:
          mode: response
          response-status: 429
          response-body: '{"code":429,"msg":"系统繁忙，请稍后重试"}'
```

## 过滤器执行顺序

```
Order 值越小 → 越先执行

- Ordered.HIGHEST_PRECEDENCE  (~ -2147483648)
  ├── IP 黑白名单 (order = -100)
  ├── 请求追踪 / MDC (order = -50)
  ├── 鉴权 (order = -10)
  ├── 限流 (order = 0)
  │
  ├── 路由级过滤器 (由路由配置顺序决定)
  │
  ├── 请求改写 (order = 10)
  ├── 响应包装 (order = 100)
  │
  └── Ordered.LOWEST_PRECEDENCE (~ 2147483647)
      └── 请求耗时日志 (order = Int.MAX_VALUE - 1)
```

## 注意事项

1. **过滤器是单例的**：自定义 `GatewayFilterFactory` 是 Spring Bean（单例），内部不能持有请求级别的状态
2. **Lua 脚本与 Redis 限流**：确保 Redis 可用性，限流降级可在 Lua 调用失败时放行（fail-open），而不是拒绝所有请求
3. **响应包装性能**：`ServerHttpResponseDecorator` 会对响应体做缓冲区合并，大量响应会消耗内存。建议只对下游聚合 API 启用
4. **过滤器执行顺序**：`GlobalFilter` 通过 `Ordered` 接口排序，相同 Order 值的执行顺序不确定
5. **GatewayFilter 与 GlobalFilter 的互相影响**：GlobalFilter 先执行，通过 `exchange.getAttributes()` 传递上下文给 GatewayFilter
6. **跨过滤器传递数据**：使用 `exchange.getAttributes().put(key, value)` 而不是 ThreadLocal（Gateway 是响应式模型，不绑定线程）
7. **异常处理**：全局异常捕获通过 `@RestControllerAdvice` 无效。需实现 `ErrorWebExceptionHandler` 或在 filter 中 try-catch
