---
name: springdoc-openapi
description: SpringDoc OpenAPI 3.1 最佳实践：RESTful API 文档自动生成、分组聚合、安全配置、OAS 3.1 兼容
tags: [tools, springdoc, openapi, swagger, api-documentation, spring-boot]
---

## 概述

SpringDoc 是 Spring Boot 社区最主流的 OpenAPI 文档生成工具。最新版本已支持 **OpenAPI 3.1**（OAS 3.1），与 JSON Schema 全面兼容，支持 `@RestController` 注解自动推断 API 文档，无需额外配置。

## 一、快速集成

```xml
<!-- Maven -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.0</version>
</dependency>

<!-- WebFlux 使用 webflux-ui -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webflux-ui</artifactId>
    <version>2.8.0</version>
</dependency>
```

```yaml
# application.yml
springdoc:
  api-docs:
    path: /api-docs           # OpenAPI JSON 端点（默认 /v3/api-docs）
  swagger-ui:
    path: /swagger-ui.html    # Swagger UI 路径
    enabled: true
    operations-sorter: method  # 按 HTTP 方法排序
    tags-sorter: alpha         # 按标签名排序
  show-actuator: false        # 屏蔽 Actuator 端点
  default-produces-media-type: application/json
  cache:
    disabled: true            # 开发环境禁用缓存
```

访问 `http://localhost:8080/swagger-ui.html` 即可看到文档界面。

## 二、分组文档（多模块微服务）

```java
@Configuration
public class OpenApiConfig {

    @Bean
    public GroupedOpenApi userApi() {
        return GroupedOpenApi.builder()
                .group("用户服务")
                .displayName("用户管理 API")
                .pathsToMatch("/api/user/**")
                .packagesToScan("com.example.user.controller")
                .addOpenApiCustomizer(openApi -> {
                    openApi.info(new Info()
                            .title("用户服务 API")
                            .version("1.0")
                            .description("用户注册、登录、信息管理"));
                })
                .build();
    }

    @Bean
    public GroupedOpenApi orderApi() {
        return GroupedOpenApi.builder()
                .group("订单服务")
                .displayName("订单管理 API")
                .pathsToMatch("/api/order/**")
                .addOpenApiCustomizer(openApi -> {
                    openApi.info(new Info()
                            .title("订单服务 API")
                            .version("1.0")
                            .description("订单创建、查询、退款"));
                })
                .build();
    }

    @Bean
    public GroupedOpenApi adminApi() {
        return GroupedOpenApi.builder()
                .group("管理后台")
                .pathsToMatch("/api/admin/**")
                .addOpenApiCustomizer(openApi -> {
                    openApi.info(new Info().title("管理后台 API"));
                    openApi.addSecurityItem(new SecurityRequirement()
                            .addList("admin-token"));
                    openApi.schemaRequirement("admin-token", new SecurityScheme()
                            .type(SecurityScheme.Type.APIKEY)
                            .in(SecurityScheme.In.HEADER)
                            .name("X-Admin-Token"));
                })
                .build();
    }
}
```

## 三、注解使用规范

### 3.1 控制器层注解

```java
@RestController
@RequestMapping("/api/user")
@Tag(name = "用户管理", description = "用户注册、登录、信息维护")
public class UserController {

    @Operation(summary = "用户注册", description = "创建新用户账号，需要提供邮箱和密码")
    @ApiResponse(responseCode = "201", description = "注册成功")
    @ApiResponse(responseCode = "400", description = "参数校验失败", 
                 content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
    @ApiResponse(responseCode = "409", description = "邮箱已被注册")
    @PostMapping("/register")
    public ResponseEntity<UserVO> register(@Valid @RequestBody RegisterRequest request) {
        // ...
    }

    @Operation(summary = "获取用户信息", description = "根据用户 ID 获取用户详细信息")
    @Parameter(name = "id", description = "用户 ID", required = true, example = "10001")
    @GetMapping("/{id}")
    public UserVO getUser(@PathVariable Long id) {
        // ...
    }

    @Operation(summary = "搜索用户", description = "按用户名模糊搜索")
    @Parameters({
        @Parameter(name = "keyword", description = "搜索关键词", example = "张三"),
        @Parameter(name = "page", description = "页码", example = "0"),
        @Parameter(name = "size", description = "每页条数", example = "20")
    })
    @GetMapping("/search")
    public PageResult<UserVO> searchUsers(@RequestParam String keyword,
                                          @PageableDefault Pageable pageable) {
        // ...
    }
}
```

### 3.2 DTO 层注解

```java
public record RegisterRequest(
    @Schema(description = "邮箱地址", example = "user@example.com", maxLength = 100)
    @Email @NotBlank String email,

    @Schema(description = "密码（6-20位字母数字组合）", example = "Abc123456", minLength = 6, maxLength = 20)
    @NotBlank @Size(min = 6, max = 20) String password,

    @Schema(description = "用户昵称", example = "小明", maxLength = 50)
    @NotBlank @Size(max = 50) String nickname
) {}

public record UserVO(
    @Schema(description = "用户 ID", example = "10001")
    Long id,
    
    @Schema(description = "邮箱", example = "user@example.com")
    String email,
    
    @Schema(description = "昵称", example = "小明")
    String nickname,
    
    @Schema(description = "注册时间", example = "2026-06-10T12:00:00")
    LocalDateTime createdAt
) {}
```

> 使用 Java Record 作为 DTO 可以大幅减少样板代码，SpringDoc 能自动推断字段类型和名称。

## 四、安全配置（JWT / OAuth2）

```java
@Configuration
public class OpenApiSecurityConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        // 定义 JWT Bearer Token 安全方案
        SecurityScheme securityScheme = new SecurityScheme()
                .type(SecurityScheme.Type.HTTP)
                .scheme("bearer")
                .bearerFormat("JWT")
                .description("输入 JWT Token（不含 Bearer 前缀）");

        // 全局安全要求
        SecurityRequirement securityRequirement = new SecurityRequirement()
                .addList("bearer-jwt");

        return new OpenAPI()
                .info(new Info()
                        .title("商城 API")
                        .version("2.0.0")
                        .description("在线商城微服务 API 文档")
                        .contact(new Contact()
                                .name("技术团队")
                                .email("dev@example.com"))
                        .license(new License()
                                .name("Apache 2.0")
                                .url("https://www.apache.org/licenses/LICENSE-2.0")))
                .addSecurityItem(securityRequirement)
                .schemaRequirement("bearer-jwt", securityScheme)
                // 全局服务器信息
                .servers(List.of(
                        new Server().url("https://api.example.com").description("生产环境"),
                        new Server().url("https://staging-api.example.com").description("预发布环境"),
                        new Server().url("http://localhost:8080").description("本地开发")
                ));
    }
}
```

### 按接口控制安全策略

```java
@RestController
@RequestMapping("/api/public")
@Tag(name = "公开接口", description = "无需认证的公开 API")
@SecurityRequirements()  // 覆盖全局安全要求，移除此接口的 JWT 要求
public class PublicController {
    // ...
}
```

## 五、OpenAPI 3.1 新特性应用

SpringDoc 2.7+ 支持 OpenAPI 3.1，核心变化：

```java
@Schema(
    description = "用户状态",
    // OpenAPI 3.1 支持 JSON Schema 2020-12 的 const/oneOf/anyOf
    oneOf = { ActiveUser.class, InactiveUser.class },
    discriminatorProperty = "status"
)
public class UserStatus {}

// 使用 @Schema examples 替代旧的 example（3.1 支持多示例）
@Schema(description = "订单状态枚举", examples = {
    "PENDING", "PAID", "SHIPPED", "COMPLETED", "CANCELLED"
})
public enum OrderStatus {
    PENDING, PAID, SHIPPED, COMPLETED, CANCELLED
}

// 3.1 支持 nullable 直接替代 required
public record UpdateRequest(
    @Schema(nullable = true, description = "可为空的字段")
    String optionalField
) {}
```

## 六、SpringDoc 与 Spring Security 集成

```java
// 自动显示当前用户信息
@Operation(summary = "获取当前登录用户信息")
@GetMapping("/me")
public UserVO getCurrentUser(
    @AuthenticationPrincipal UserDetails userDetails,
    @Parameter(hidden = true)  // 不在文档中显示
    HttpServletRequest request
) {
    // ...
    return userService.findByUsername(userDetails.getUsername());
}

// 隐藏内部端点
@Operation(hidden = true)  // 不在 Swagger UI 显示
@GetMapping("/internal/health/check")
public String internalHealth() {
    return "OK";
}
```

## 七、生产环境配置

```yaml
# application-prod.yml
springdoc:
  api-docs:
    enabled: true              # 仍然启用，供网关/测试工具使用
    path: /internal/api-docs
  swagger-ui:
    enabled: false             # 生产环境禁用 Swagger UI
  packages-to-scan: com.example.app.controller
  paths-to-match: /api/**
```

### 安全控制（IP 白名单 + Spring Security）

```java
@Configuration
@Profile("prod")
public class ProdSecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.securityMatcher("/swagger-ui/**", "/v3/api-docs/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(RequestMatchers.ipAddress("10.0.0.0/8"))
                .permitAll()
                .anyRequest().denyAll()
            );
        return http.build();
    }
}
```

## 八、Gateway 统一聚合文档

在 Spring Cloud Gateway 中聚合所有微服务的 OpenAPI 文档：

```java
// Gateway 路由配置
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("user-service", r -> r
            .path("/api/user/**")
            .uri("lb://user-service"))
        .route("order-service", r -> r
            .path("/api/order/**")
            .uri("lb://order-service"))
        .build();
}

// 聚合文档端点
@Bean
public GroupedOpenApi aggregatedApi() {
    return GroupedOpenApi.builder()
            .group("all-services")
            .displayName("全服务聚合")
            .pathsToMatch("/api/**")
            .build();
}
```

## 注意事项

1. **生产环境禁用 UI** — 永远不要在公网暴露 Swagger UI。可使用 `/internal/api-docs` 内网路径 + IP 白名单。SpringDoc 本身不提供认证，依赖 Spring Security 做访问控制。
2. **Record vs 传统类** — Java Record 默认无 setter，SpringDoc 仍能正确解析字段。但需注意：若使用 `@Schema(hidden = true)` 隐藏字段，Record 需配合 `@JsonIgnore` 或自定义序列化。
3. **循环引用** — 双向关联的 JPA 实体（如 `User` ↔ `Order`）会导致文档生成 StackOverflow。使用 `@JsonIgnoreProperties` 或 `@Schema(implementation = ...)` 打破循环。
4. **泛型支持** — 泛型返回类型（如 `Page<UserVO>`、`Result<T>`）需要在 `@Schema` 中显式声明或使用 SpringDoc 的 `ModelConverter` 自定义解析。
5. **枚举序列化** — 默认 `@Schema(implementation = String.class)` 可将枚举显示为字符串。若枚举实现了 `JsonSerializer` 或使用了 `@JsonValue`，需通过 `@Schema(type = "string", allowableValues = {"A", "B"})` 手动声明。
6. **OpenAPI 3.1 vs 3.0** — 如果 API 网关或聚合工具只支持 3.0，在配置中降级：`springdoc.api-docs.version=openapi_3_0`。3.1 的 JSON Schema 2020-12 在部分旧工具中不兼容。
7. **大型项目性能** — 50+ Controller 的项目首次加载文档可能较慢。配置 `springdoc.cache.disabled=false` 启用缓存，或使用 `springdoc.paths-to-match` 限制扫描范围。
8. **Kotlin 支持** — SpringDoc 原生支持 Kotlin，但 nullable 类型需配置 `jackson-datatype-jsr310` 和 `jackson-module-kotlin`。Kotlin data class 的默认值不会被识别为 optional。
