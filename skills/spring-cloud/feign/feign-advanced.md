---
name: feign-advanced
description: OpenFeign 高级用法（拦截器、错误解码、重试、OAuth2）
tags: [spring-cloud, feign, rpc, microservice]
---

## 自定义拦截器（Interceptor）

### 统一添加请求头

```java
// 1. 实现 RequestInterceptor 接口
@Component
public class FeignAuthInterceptor implements RequestInterceptor {

    @Autowired
    private HttpServletRequest request;

    @Override
    public void apply(RequestTemplate template) {
        // 透传 Token
        String token = request.getHeader("Authorization");
        if (StringUtils.hasText(token)) {
            template.header("Authorization", token);
        }

        // 透传 Trace ID（分布式链路追踪）
        String traceId = request.getHeader("X-Trace-Id");
        if (StringUtils.hasText(traceId)) {
            template.header("X-Trace-Id", traceId);
        }

        // 内部服务认证
        template.header("X-Internal-Call", "true");
    }
}
```

### 请求/响应日志拦截

```java
@Component
public class FeignLogInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        MDC.put("feign.url", template.url());
        MDC.put("feign.method", template.method());
    }
}
```

## 自定义错误解码器（ErrorDecoder）

### 统一异常处理

```java
@Slf4j
@Component
public class FeignErrorDecoder implements ErrorDecoder {

    @Autowired
    private ObjectMapper objectMapper;

    @Override
    public Exception decode(String methodKey, Response response) {
        log.error("Feign 调用失败: method={}, status={}, url={}",
            methodKey, response.status(), response.request().url());

        // 读取响应体
        String body = null;
        try {
            if (response.body() != null) {
                body = Util.toString(response.body().asReader(StandardCharsets.UTF_8));
            }
        } catch (IOException e) {
            // ignore
        }

        return switch (response.status()) {
            case 400 -> {
                ErrorResponse error = parseError(body);
                yield new BadRequestException(error.getMessage());
            }
            case 401 -> new UnauthorizedException("服务认证失败");
            case 403 -> new ForbiddenException("服务调用无权限");
            case 404 -> new ResourceNotFoundException("请求资源不存在");
            case 429 -> new RateLimitException("服务限流，请稍后重试");
            case 500, 502, 503 -> {
                yield new ServiceUnavailableException("下游服务异常: " + response.status());
            }
            default -> new GenericFeignException("Feign 调用异常: " + response.reason());
        };
    }

    private ErrorResponse parseError(String body) {
        try {
            return objectMapper.readValue(body, ErrorResponse.class);
        } catch (Exception e) {
            return new ErrorResponse("unknown_error", body);
        }
    }
}
```

### 注册 ErrorDecoder

```java
// 方式一：全局配置
@Bean
public ErrorDecoder feignErrorDecoder() {
    return new FeignErrorDecoder();
}

// 方式二：按 FeignClient 单独配置（@Configuration 类中）
@Bean
public ErrorDecoder userServiceErrorDecoder() {
    return new RetryableErrorDecoder(); // 针对特定服务
}
```

## 重试机制（Retryer）

```java
// 1. 默认重试策略（不重试）
// Feign 默认不重试，需要手动配置

// 2. 自定义重试器
@Component
public class FeignRetryer extends Retryer.Default {

    public FeignRetryer() {
        // period=100ms(初始间隔), maxPeriod=1000ms(最大间隔), maxAttempts=3(最大尝试)
        super(100, 1000, 3);
    }

    @Override
    public void continueOrPropagate(RetryableException e) {
        if (e.status() == 400 || e.status() == 401 || e.status() == 403) {
            // 这些状态不需要重试，直接抛出
            throw e;
        }
        // 只有 5xx 和网络异常才重试
        super.continueOrPropagate(e);
    }
}

// 3. 配置重试
# application.yml
feign:
  client:
    config:
      default:
        retryer: com.example.common.feign.FeignRetryer
      # 按服务单独配置
      user-service:
        retryer: com.example.common.feign.UserServiceRetryer
```

## Sentinel 集成

```yaml
# 1. 开启 Sentinel 对 Feign 的支持
feign:
  sentinel:
    enabled: true

# 2. 配置 Sentinel 规则（可以通过 Nacos 动态下发）
spring:
  cloud:
    sentinel:
      datasource:
        ds1:
          nacos:
            server-addr: ${nacos.addr}
            data-id: sentinel-flow-rules.json
            rule-type: flow
```

```java
// 3. FeignClient + fallbackFactory
@FeignClient(
    name = "user-service",
    url = "${user.service.url}",
    fallbackFactory = UserClientFallbackFactory.class
)
public interface UserClient {

    @GetMapping("/api/users/{id}")
    Result<UserDTO> getUser(@PathVariable("id") Long id);

    @PostMapping("/api/users/batch")
    Result<List<UserDTO>> getUsers(@RequestBody List<Long> ids);
}

@Slf4j
@Component
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {

    @Override
    public UserClient create(Throwable cause) {
        log.error("用户服务 Feign 调用降级", cause);
        return new UserClient() {
            @Override
            public Result<UserDTO> getUser(Long id) {
                // 返回兜底数据
                return Result.error("用户服务暂不可用");
            }

            @Override
            public Result<List<UserDTO>> getUsers(List<Long> ids) {
                return Result.error("用户服务暂不可用");
            }
        };
    }
}
```

## OAuth2 集成

```java
// 自动添加 OAuth2 令牌
@Configuration
public class FeignOAuth2Config {

    @Bean
    @ConditionalOnBean(OAuth2AuthorizedClientManager.class)
    public RequestInterceptor oauth2FeignRequestInterceptor(
            OAuth2AuthorizedClientManager manager,
            @Value("${feign.client.registration-id}") String registrationId) {

        return template -> {
            OAuth2AuthorizedClient client = manager.authorize(
                OAuth2AuthorizeRequest
                    .withClientRegistrationId(registrationId)
                    .principal("internal_service")
                    .build()
            );
            if (client != null) {
                template.header("Authorization",
                    "Bearer " + client.getAccessToken().getTokenValue());
            }
        };
    }
}
```

## 性能优化配置

```yaml
feign:
  client:
    config:
      default:
        # 连接超时
        connect-timeout: 5000
        # 读取超时
        read-timeout: 10000
        # 日志级别
        logger-level: BASIC
  # 开启 HTTP/2 支持（需要引入 feign-hc5 或 feign-okhttp）
  httpclient:
    hc5:
      enabled: true
      # 连接池大小
      max-connections: 200
      max-connections-per-route: 50
      # 空闲连接存活时间
      time-to-live: 900
    # 连接超时和重试
    connection-timeout: 5000
    connection-timer-repeat: 3000
  # 启用压缩
  compression:
    request:
      enabled: true
      mime-types: application/json
      min-request-size: 2048
    response:
      enabled: true
```

## 最佳实践总结

| 场景 | 推荐做法 |
|---|---|
| 统一认证 | `RequestInterceptor` 透传 Token |
| 错误处理 | 自定义 `ErrorDecoder` 将 HTTP 状态码转为业务异常 |
| 降级熔断 | 配合 Sentinel，`fallbackFactory` 返回兜底数据 |
| 接口重试 | `Retryer` 配置，5xx 重试，4xx 不重试 |
| 性能 | 使用 OkHttp 或 HC5 替代默认的 URLConnection |
| 日志 | 生产用 BASIC，调试用 FULL |
| 超时 | 区分连接超时和读取超时，根据服务 SLA 单独配置 |
