---
name: api-versioning-pattern
description: 微服务 API 版本管理策略：URI 路径、请求头、查询参数、Content Negotiation 四方案对比与最佳实践
tags: [patterns, api, versioning, microservice, rest, backward-compatibility]
---

## 概述

API 版本管理是微服务架构中不可回避的问题。服务端的接口变更（新增字段、修改参数类型、删除接口）需要确保**向后兼容**，同时允许客户端逐步迁移到新版本。

## 版本策略对比

| 策略 | 实现方式 | 示例 | 优点 | 缺点 |
|------|---------|------|------|------|
| **URI 路径** | URL 中包含版本号 | `/api/v1/orders` | 最直观，缓存友好 | URL 不幂等，语义不纯粹 |
| **请求头** | 自定义 Header | `Accept-Version: v1` | URL 干净 | 客户端实现复杂，浏览器调试不便 |
| **查询参数** | Query String | `/api/orders?version=1` | 简单灵活 | URL 污染，缓存困难 |
| **Content Negotiation** | Accept Header | `Accept: application/vnd.company.v1+json` | RESTful 纯粹 | 配置复杂，工具支持差 |

## 1. URI 路径版本（推荐）

### 原理

版本号直接放在 URL 路径中，Spring Cloud Gateway 或 API 网关统一路由。

```yaml
# Spring Cloud Gateway 路由
spring:
  cloud:
    gateway:
      routes:
        - id: order-service-v1
          uri: lb://order-service
          predicates:
            - Path=/api/v1/orders/**
          filters:
            - StripPrefix=1

        - id: order-service-v2
          uri: lb://order-service-v2
          predicates:
            - Path=/api/v2/orders/**
          filters:
            - StripPrefix=1
```

### Spring Boot Controller 实现

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderControllerV1 {

    @GetMapping
    public List<OrderV1> list() {
        // V1 实现
    }
}

@RestController
@RequestMapping("/api/v2/orders")
public class OrderControllerV2 {

    @GetMapping
    public List<OrderV2> list() {
        // V2 实现（新增分页参数）
    }
}
```

### Nginx 路由

```nginx
# nginx.conf - 按版本分发到不同后端
upstream order-v1 {
    server order-service-v1:8080;
}

upstream order-v2 {
    server order-service-v2:8080;
}

server {
    listen 80;

    location /api/v1/ {
        proxy_pass http://order-v1/;
    }

    location /api/v2/ {
        proxy_pass http://order-v2/;
    }
}
```

## 2. 请求头版本

### 实现（Spring Boot）

```java
// 自定义版本解析器
public class ApiVersionHandler {

    private static final ThreadLocal<String> VERSION_HOLDER = new ThreadLocal<>();

    public static void setVersion(String version) {
        VERSION_HOLDER.set(version);
    }

    public static String getVersion() {
        return VERSION_HOLDER.get();
    }

    public static void clear() {
        VERSION_HOLDER.remove();
    }
}

// 拦截器解析版本
@Component
public class ApiVersionInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) {
        String version = request.getHeader("X-API-Version");
        if (version == null) {
            version = "v1";  // 默认版本
        }
        ApiVersionHandler.setVersion(version);
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                HttpServletResponse response,
                                Object handler, Exception ex) {
        ApiVersionHandler.clear();
    }
}

// 注册拦截器
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new ApiVersionInterceptor())
                .addPathPatterns("/api/**");
    }
}
```

### 服务端版本路由

```java
@Service
public class OrderService {

    public OrderResult getOrder(Long id) {
        String version = ApiVersionHandler.getVersion();
        return switch (version) {
            case "v1" -> getOrderV1(id);
            case "v2" -> getOrderV2(id);
            default -> throw new UnsupportedVersionException(version);
        };
    }
}
```

## 3. Content Negotiation（媒体类型版本）

### 原理

利用 HTTP `Accept` 头指定 API 版本：

```bash
# V1 请求
GET /api/orders/123
Accept: application/vnd.company.order-v1+json

# V2 请求
GET /api/orders/123
Accept: application/vnd.company.order-v2+json
```

### Spring Boot 配置

```java
@Configuration
public class ContentNegotiationConfig
        implements WebMvcConfigurer {

    @Override
    public void configureContentNegotiation(
            ContentNegotiationConfigurer configurer) {
        configurer
            .favorParameter(false)
            .ignoreAcceptHeader(false)
            .defaultContentType(
                MediaType.valueOf("application/vnd.company.order-v1+json"));
    }
}

// 注册自定义媒体类型
@Configuration
public class MediaTypeConfig {

    @PostConstruct
    public void init() {
        MediaType.registerMediaType(
            "application/vnd.company.order-v1+json",
            MediaType.valueOf("application/vnd.company.order-v1+json"));
        MediaType.registerMediaType(
            "application/vnd.company.order-v2+json",
            MediaType.valueOf("application/vnd.company.order-v2+json"));
    }
}
```

```java
// V1 接口
@RestController
@RequestMapping("/api/orders")
public class OrderControllerV1 {

    @GetMapping(produces = "application/vnd.company.order-v1+json")
    public OrderV1 get(@PathVariable Long id) {
        return orderService.getV1(id);
    }
}

// V2 接口
@RestController
@RequestMapping("/api/orders")
public class OrderControllerV2 {

    @GetMapping(produces = "application/vnd.company.order-v2+json")
    public OrderV2 get(@PathVariable Long id) {
        return orderService.getV2(id);
    }
}
```

## 4. gRPC 版本管理

gRPC 使用 Protobuf，版本管理策略不同：

```protobuf
// 方式一：Package 命名空间隔离
package order.v1;
service OrderService { ... }

package order.v2;
service OrderService { ... }

// 方式二：字段兼容（推荐）
// 永远不删除字段，使用 reserved 标记弃用字段
message Order {
  reserved 2;          // 废弃的 price 字段
  reserved "price";
  
  int64 id = 1;
  string name = 3;
  int64 total_amount = 4;  // 替代 price
}
```

## 5. 向后兼容最佳实践

### 字段变更规则

```json
// 原则：只做加法，不做减法

// V1 响应
{
  "id": 123,
  "name": "张三",
  "email": "zhangsan@example.com"
}

// V2 响应（V1 字段不动，新增字段）
{
  "id": 123,
  "name": "张三",
  "email": "zhangsan@example.com",
  "phone": "13800138000",       // 新增
  "profile": {                   // 新增嵌套对象
    "avatar": "/img/avatar.png",
    "bio": "Hello"
  }
}
```

### 默认值策略

```java
public class OrderV1toV2Converter {

    // V1 请求 → V2 请求（老客户端调用新 API）
    public OrderV2 convertV1toV2(OrderV1 v1) {
        return OrderV2.builder()
            .id(v1.getId())
            .productName(v1.getProductName())
            .quantity(v1.getQuantity())
            .couponCode("")          // V1 没有，默认空
            .deliveryMethod("standard") // V1 没有，默认普通配送
            .build();
    }

    // V2 响应 → V1 响应（新 API 兼容老客户端）
    public OrderV1 convertV2toV1(OrderV2 v2) {
        return OrderV1.builder()
            .id(v2.getId())
            .productName(v2.getProductName())
            .quantity(v2.getQuantity())
            .build();
            // V2 新增字段自动忽略
    }
}
```

### Deprecation 通知

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderControllerV1 {

    @GetMapping
    @Deprecated
    @Operation(
        summary = "获取订单列表（已废弃）",
        deprecated = true,
        description = "请使用 /api/v2/orders"
    )
    public Response<List<OrderV1>> list(
        HttpServletResponse response
    ) {
        // 设置废弃响应头
        response.setHeader("Sunset", "Sat, 31 Dec 2026 23:59:59 GMT");
        response.setHeader("Deprecation", "true");
        response.setHeader("Link",
            "</api/v2/orders>; rel=\"successor-version\"");

        return Response.success(orderService.listV1());
    }
}
```

## 6. 版本生命周期管理

```yaml
# API 版本生命周期表
# +-----------+------------+--------------+------------------+
# | 版本      | 状态        | 发布日期     | 废弃日期        |
# +-----------+------------+--------------+------------------+
# | v1        | 已废弃     | 2024-01-01   | 2026-12-31       |
# | v2        | 当前活跃   | 2025-06-01   | -                |
# | v3        | Beta       | 2026-06-01   | -                |
# +-----------+------------+--------------+------------------+
```

```java
// 版本过期间-返回 410 Gone
@Component
public class DeprecatedVersionFilter implements Filter {

    private static final Set<String> DEPRECATED_VERSIONS =
        Set.of("v0", "v0.5");

    @Override
    public void doFilter(ServletRequest request,
                         ServletResponse response,
                         FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest req = (HttpServletRequest) request;
        String path = req.getRequestURI();

        for (String deprecated : DEPRECATED_VERSIONS) {
            if (path.contains("/api/" + deprecated + "/")) {
                HttpServletResponse resp = (HttpServletResponse) response;
                resp.setStatus(410);
                resp.setContentType("application/json");
                resp.getWriter().write(
                    "{\"error\":\"version_gone\"," +
                    "\"message\":\"API version " + deprecated +
                    " is no longer supported\"}");
                return;
            }
        }
        chain.doFilter(request, response);
    }
}
```

## 注意事项

1. **URI 版本最实用**：虽然 REST 纯化论者不认同，但 URI 版本在生产中管理成本最低
2. **服务端版本 + 网关路由**：不同版本的接口应部署为独立服务（或不同 Deployment），避免同一个 Jar 包维护多版本
3. **版本号格式**：使用 `v1`、`v2` 整数版本号，避免语义化版本（`v1.2.3`）增加复杂度
4. **废弃窗口期**：至少保留老版本 12 个月，给客户端充分迁移时间
5. **监控版本使用率**：通过日志或指标追踪各版本调用量，决定何时彻底下架
6. **OpenAPI 文档**：每个版本生成独立的 OpenAPI/Swagger 文档，避免混淆
7. **内部服务版本**：内部服务间调用建议用 gRPC Protobuf 的字段兼容方案，而非多版本部署

## 参考链接

- [Microsoft REST API Guidelines - Versioning](https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md#12-versioning)
- [Zalando RESTful API Guidelines](https://opensource.zalando.com/restful-api-guidelines/)
- [Spring REST API Versioning](https://spring.io/guides/tutorials/rest/5-versioning/)
