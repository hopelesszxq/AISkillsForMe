---
name: feign-best-practices
description: OpenFeign 声明式调用最佳实践：超时配置、连接池、日志、多版本与测试
tags: [spring-cloud, feign, rpc, microservice]
---

## 核心配置

```yaml
feign:
  sentinel:
    enabled: true                  # 开启 Sentinel 熔断（必须）
  okhttp:
    enabled: true                  # 使用 OkHttp 替代默认 URLConnection
  httpclient:
    hc5:
      enabled: false               # OkHttp 和 HC5 二选一
  compression:
    request:
      enabled: true
      mime-types: application/json
      min-request-size: 2048
    response:
      enabled: true
  client:
    config:
      default:                     # 全局默认配置
        connect-timeout: 5000      # 连接超时（毫秒）
        read-timeout: 10000        # 读取超时（毫秒）
        logger-level: BASIC        # 日志级别：NONE/BASIC/HEADERS/FULL
      user-service:                # 按服务单独配置
        connect-timeout: 2000
        read-timeout: 3000
        logger-level: FULL
```

## 连接池配置（OkHttp）

```yaml
feign:
  okhttp:
    enabled: true
  client:
    okhttp:
      max-connections: 200            # 最大连接数
      max-connections-per-route: 50   # 每个路由最大连接数
      time-to-live: 900               # 空闲连接存活时间（秒）
      connect-timeout: 5000           # 连接超时
      read-timeout: 10000             # 读取超时
      write-timeout: 10000            # 写入超时
      follow-redirects: false         # 不跟随重定向
```

```xml
<!-- 依赖 -->
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-okhttp</artifactId>
</dependency>
```

## FeignClient 声明规范

### 基础用法

```java
// 1. 对外提供 jar 包，定义 Feign 接口
@FeignClient(
    name = "user-service",
    url = "${user.service.url:}",           // 直连模式（非注册中心）
    path = "/api/users",
    primary = false,                         // 避免多个实现冲突
    qualifiers = "userClient"                // 注入时的限定符
)
public interface UserClient {

    @GetMapping("/{id}")
    Result<UserVO> getById(@PathVariable("id") Long id);

    @PostMapping("/batch")
    Result<List<UserVO>> getByIds(@RequestBody List<Long> ids);

    @PutMapping("/{id}")
    Result<Void> update(@PathVariable("id") Long id, @RequestBody UserUpdateDTO dto);
}
```

### 服务间调用

```yaml
# 使用 Nacos 注册中心时，name 就是服务名
# 不需要配置 url，自动负载均衡
feign:
  client:
    config:
      user-service:
        connect-timeout: 3000
  # 开启懒加载（减少启动时间）
  hystrix:
    enabled: false  # 如果用了 Sentinel 要关掉 Hystrix
```

### Feign 接口复用（公共 jar）

```java
// 方案一：定义在 common 模块的 API 包中
// user-api 模块（provider + consumer 都依赖）
// └── src/main/java/com/example/user/api/
//     ├── UserClient.java          # Feign 接口
//     └── UserVO.java              # DTO

// Provider 实现
@RestController
@RequestMapping("/api/users")
public class UserController implements UserClient {
    // 直接实现 Feign 接口
}

// Consumer 注入
@Service
public class OrderService {
    @Autowired
    private UserClient userClient;  // 复用同一个接口
}
```

## 降级处理

```java
@FeignClient(
    name = "user-service",
    fallbackFactory = UserClientFallbackFactory.class,
    fallback = UserClientFallback.class  // 二选一，优先 fallbackFactory
)
public interface UserClient {
    @GetMapping("/{id}")
    Result<UserVO> getById(@PathVariable("id") Long id);
}

// 推荐 FallbackFactory（可以拿到异常原因）
@Slf4j
@Component
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {

    @Override
    public UserClient create(Throwable cause) {
        log.error("UserClient 调用失败", cause);

        // 根据异常类型返回不同的兜底数据
        if (cause instanceof RateLimitException) {
            return new UserClient() {
                @Override
                public Result<UserVO> getById(Long id) {
                    return Result.error(429, "服务限流，请稍后重试");
                }
            };
        }

        return new UserClient() {
            @Override
            public Result<UserVO> getById(Long id) {
                // 从本地缓存获取兜底数据
                UserVO cached = LocalCache.get("user:" + id);
                if (cached != null) {
                    return Result.success(cached);
                }
                return Result.error(503, "用户服务暂不可用");
            }
        };
    }
}
```

## 多版本 API 支持

```yaml
# 通过 headers 传递版本号
feign:
  client:
    config:
      user-service:
        request-interceptors:
          - com.example.common.feign.VersionInterceptor
```

```java
@Component
public class VersionInterceptor implements RequestInterceptor {
    @Value("${feign.api.version:v1}")
    private String apiVersion;

    @Override
    public void apply(RequestTemplate template) {
        template.header("X-API-Version", apiVersion);
    }
}
```

## 请求/响应日志

```java
// 生产日志配置
// application.yml
logging:
  level:
    com.example.user.client.UserClient: DEBUG   # Feign 接口包名

// 自定义日志级别
@Configuration
public class FeignLogConfig {
    @Bean
    Logger.Level feignLoggerLevel() {
        // NONE: 不记录（默认）
        // BASIC: 只记录请求方法、URL、响应状态码、执行时间
        // HEADERS: BASIC + 请求响应头
        // FULL: 记录所有（请求/响应头、体、元数据）
        return Logger.Level.BASIC;
    }
}
```

## 请求/响应拦截

### 1. 链路追踪透传

```java
@Component
public class TraceIdInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        // 透传 TraceId 用于链路追踪
        String traceId = MDC.get("traceId");
        if (StringUtils.hasText(traceId)) {
            template.header("X-Trace-Id", traceId);
        }
        // 透传 SpanId
        String spanId = MDC.get("spanId");
        if (StringUtils.hasText(spanId)) {
            template.header("X-Span-Id", spanId);
        }
    }
}
```

### 2. 统一处理请求编码

```java
@Component
public class FeignEncoderConfig {
    @Bean
    public Encoder feignEncoder() {
        // 默认使用 SpringEncoder，支持 @SpringQueryMap
        return new SpringEncoder(() -> new Jackson2ObjectMapperBuilder()
            .dateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"))
            .timeZone(TimeZone.getTimeZone("Asia/Shanghai"))
            .build());
    }
}
```

## Feign 调用测试

```java
// 单元测试：Mock Feign 客户端
@SpringBootTest
@AutoConfigureMockMvc
class OrderServiceTest {

    @MockitoBean
    private UserClient userClient;

    @Autowired
    private OrderService orderService;

    @Test
    void testCreateOrder() {
        // Mock Feign 调用
        UserVO mockUser = new UserVO();
        mockUser.setId(1L);
        mockUser.setName("测试用户");
        Mockito.when(userClient.getById(1L)).thenReturn(Result.success(mockUser));

        // 执行业务
        Result<String> result = orderService.createOrder(/*...*/);

        // 验证
        assertThat(result.isSuccess()).isTrue();
        Mockito.verify(userClient).getById(1L);
    }
}
```

```java
// 集成测试：MockFeignClient
@SpringBootTest(properties = {
    "user.service.url=http://localhost:${wiremock.server.port}"
})
@AutoConfigureWireMock(port = 0)
class UserClientIntegrationTest {

    @Autowired
    private UserClient userClient;

    @Test
    void testGetUser() {
        // WireMock 模拟响应
        stubFor(get(urlEqualTo("/api/users/1"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {"code":200,"data":{"id":1,"name":"测试用户"}}
                """)
            )
        );

        Result<UserVO> result = userClient.getById(1L);
        assertThat(result.isSuccess()).isTrue();
        assertThat(result.getData().getName()).isEqualTo("测试用户");
    }
}
```

```xml
<!-- WireMock 依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
    <scope>test</scope>
</dependency>
```

## 常见坑点

### 1. GET 请求传对象

```java
// ❌ 错误：Feign GET 不支持 @RequestBody
// ✅ 正确：使用 @SpringQueryMap 或拆解为多个 @RequestParam

@FeignClient(name = "user-service")
public interface UserClient {

    // ✅ 方式一：@SpringQueryMap
    @GetMapping("/search")
    Result<List<UserVO>> search(@SpringQueryMap UserQuery query);

    // ✅ 方式二：@RequestParam 展开
    @GetMapping("/search")
    Result<List<UserVO>> search(
        @RequestParam("name") String name,
        @RequestParam("age") Integer age,
        @RequestParam("page") Integer page
    );
}
```

### 2. 路径变量含特殊字符

```java
// ❌ 错误：路径中直接拼接 userId，可能导致 URL 编码问题
// ✅ 正确：使用 @PathVariable 自动编码

@GetMapping("/{userId}")
Result<UserVO> getByUserId(@PathVariable("userId") String userId);
```

### 3. 超时配置不生效

```yaml
# ❌ 错误：放在 feign 外层
feign:
  connect-timeout: 5000  # 不生效

# ✅ 正确：放在 feign.client.config 下
feign:
  client:
    config:
      default:
        connect-timeout: 5000
      specific-service:
        connect-timeout: 3000
```

### 4. 循环依赖

```java
// 避免服务间循环调用：
// A → B → A 会形成循环
// 解决方案：引入消息队列解耦，或 API 合并

// 如果必须在启动时注入 FeignClient，使用 @Lazy
@Lazy
@Autowired
private UserClient userClient;
```

### 5. 请求头丢失

```java
// 配置 Hystrix 线程池隔离时请求头会丢失
// 解决方案：使用 HystrixRequestVariableDefault 或信号量隔离
hystrix:
  command:
    default:
      execution:
        isolation:
          strategy: SEMAPHORE  # 使用信号量隔离

// 或者自定义 Feign 请求头传递
@Component
public class RequestHeaderInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        RequestAttributes attributes = RequestContextHolder.getRequestAttributes();
        if (attributes instanceof ServletRequestAttributes sra) {
            HttpServletRequest request = sra.getRequest();
            // 透传特定请求头
            template.header("Authorization", request.getHeader("Authorization"));
            template.header("X-Request-Id", request.getHeader("X-Request-Id"));
        }
    }
}
```

### 6. 404 错误处理

```java
// Spring Boot 3.x 中 Feign 404 默认抛异常
// 如果下游服务返回空数据用 404，需要自定义 ErrorDecoder 处理
@Component
public class Feign404ErrorDecoder implements ErrorDecoder {
    @Override
    public Exception decode(String methodKey, Response response) {
        if (response.status() == 404) {
            // 返回 null 或空结果，不抛异常
            return new FeignNotFoundException("资源不存在");
        }
        return new Default().decode(methodKey, response);
    }
}
```

## 性能对比

| HTTP 客户端 | 连接池 | 性能 | 推荐场景 |
|-------------|--------|------|----------|
| URLConnection（默认） | 无 | 低 | 不推荐 |
| Apache HC5 | 有 | 中 | 需要连接池 + HTTP/2 |
| **OkHttp** | 有 | **高** | **通用推荐** |
| Reactor Netty | 有 | 高（异步） | WebFlux 项目 |

## 总结清单

- [✓] 开启 `feign.sentinel.enabled=true`
- [✓] 使用 OkHttp 替代默认 HTTP 客户端
- [✓] 配置合理的 connect-timeout 和 read-timeout（按服务）
- [✓] 生产环境日志级别 BASIC
- [✓] FallbackFactory 处理降级（带异常日志）
- [✓] ErrorDecoder 统一处理 HTTP 错误
- [✓] 请求拦截器透传 TraceId、Token
- [✓] GET 请求使用 @SpringQueryMap
- [✓] @PathVariable 路径变量自动编码
- [✓] 公共 Feign 接口定义在独立 jar 包
- [✓] 单元测试用 @MockitoBean，集成测试用 WireMock
