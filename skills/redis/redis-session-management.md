---
name: redis-session-management
description: Redis 实现分布式会话管理：Spring Session 集成、高可用、性能优化
tags: [redis, session, spring-session, distributed, security]
---

## 概述

在微服务架构中，HTTP Session 共享是常见的需求。Redis 作为高性能内存数据库，是分布式 Session 存储的首选方案。Spring Session + Redis 的组合提供了开箱即用的分布式会话管理能力。

## 1. Spring Session + Redis 集成

### 依赖配置

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```

### 配置

```yaml
spring:
  session:
    store-type: redis
    timeout: 1800              # Session 超时时间（秒），默认 30 分钟
    redis:
      namespace: myapp:session  # Redis key 前缀
      flush-mode: on-save       # 保存模式：on-save(默认) / immediate
      cleanup-cron: "0 0/5 * * * ?"  # 清理过期 Session 的 cron
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}
      database: 0
      timeout: 2000ms
      lettuce:
        pool:
          max-active: 16         # 最大连接数
          max-idle: 8            # 最大空闲连接
          min-idle: 4            # 最小空闲连接
          max-wait: 500ms        # 获取连接最大等待时间
```

### 启动 Session

```java
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
@Configuration
public class SessionConfig {

    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName("localhost");
        config.setPort(6379);
        config.setPassword(RedisPassword.of("password"));
        return new LettuceConnectionFactory(config);
    }

    // 自定义 Session 序列化（推荐 JSON 替代 JDK）
    @Bean
    public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
        return new GenericJackson2JsonRedisSerializer();
    }
}
```

## 2. Session 数据结构

Spring Session 在 Redis 中的存储结构：

```
Redis Keys:
  myapp:session:sessions:<session-id>         # Hash - Session 数据
  myapp:session:expirations:<expiration-time>  # Set - 过期索引
  myapp:session:sessions:expires:<session-id>  # String - 过期标记
```

### Session Hash 内部结构

```
Key: myapp:session:sessions:abc123
Hash:
  creationTime         → 1748937600000
  lastAccessedTime     → 1748937700000
  maxInactiveInterval  → 1800
  sessionAttr:username → "admin"
  sessionAttr:role     → "ROLE_USER"
```

### Session 数据自动管理

Redis 使用以下机制自动处理 Session 过期：

1. **`TTR` on `expires:<session-id>`**：Redis 会自动删除此 Key，触发 `keyspace` 通知
2. **Spring Session 后台任务**：定期扫描 `expirations` Set，清理过期 Session
3. **惰性删除**：读取 Session 时检查是否过期

## 3. 基本使用

```java
@RestController
@RequestMapping("/api")
public class SessionController {

    // 读取 Session 属性
    @GetMapping("/session")
    public Map<String, Object> getSession(HttpSession session) {
        Map<String, Object> attrs = new HashMap<>();
        session.getAttributeNames()
            .asIterator()
            .forEachRemaining(name -> 
                attrs.put(name, session.getAttribute(name)));
        attrs.put("sessionId", session.getId());
        attrs.put("creationTime", session.getCreationTime());
        attrs.put("lastAccessedTime", session.getLastAccessedTime());
        return attrs;
    }

    // 设置 Session 属性
    @PostMapping("/session")
    public String setSession(HttpSession session,
                              @RequestParam String key,
                              @RequestParam String value) {
        session.setAttribute(key, value);
        return "OK";
    }

    // 销毁 Session
    @DeleteMapping("/session")
    public String invalidateSession(HttpSession session) {
        session.invalidate();
        return "Session invalidated";
    }
}
```

## 4. 多微服务 Session 共享

### 配置统一 Redis

```yaml
# 所有微服务配置同一个 Redis 实例/集群
spring:
  session:
    redis:
      namespace: app:session  # 所有服务使用相同 namespace
```

### Cookie 配置（统一域名）

```yaml
server:
  servlet:
    session:
      cookie:
        name: APPSESSION       # 统一 Cookie 名
        domain: .example.com   # 共享二级域名
        path: /
        http-only: true
        secure: true
        max-age: 1800
```

### 服务间认证校验

```java
@Component
public class SessionAuthFilter implements WebFilter {

    @Autowired
    private SessionRepository<RedisSession> sessionRepository;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String sessionId = exchange.getRequest()
            .getCookies()
            .getFirst("APPSESSION")
            .getValue();

        RedisSession session = sessionRepository.findById(sessionId);

        if (session == null || session.isExpired()) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        // 将用户信息放入请求上下文
        String userId = session.getAttribute("userId");
        ServerHttpRequest mutatedRequest = exchange.getRequest()
            .mutate()
            .header("X-User-Id", userId)
            .build();

        return chain.filter(exchange.mutate().request(mutatedRequest).build());
    }
}
```

## 5. 安全性增强

### Session Fixation 防护

```java
// 登录成功后重新生成 Session ID（默认已开启）
@PostMapping("/login")
public String login(HttpServletRequest request,
                    @RequestParam String username) {
    HttpSession oldSession = request.getSession(false);
    if (oldSession != null) {
        oldSession.invalidate(); // 使旧 Session 失效
    }
    HttpSession newSession = request.getSession(true);
    newSession.setAttribute("userId", username);
    return "Login success";
}

// 或者使用 Spring Security 自动处理
// http.sessionManagement()
//     .sessionFixation().migrateSession() // 默认：迁移属性到新 Session
//     .sessionFixation().newSession()      // 创建新 Session（不迁移属性）
//     .sessionFixation().changeSessionId() // 仅改 ID
```

### Session 并发控制

```yaml
spring:
  session:
    security:
      maximum-sessions: 1           # 最大并发 Session 数
      max-sessions-prevents-login: false  # true=超出时禁止新登录, false=踢掉旧会话
```

```java
@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                .maximumSessions(1)
                .maxSessionsPreventsLogin(false)
                .expiredUrl("/login?expired")
            );
        return http.build();
    }
}
```

## 6. Session 性能优化

### 减少 Session 存储大小

```java
// ❌ 避免存储大对象到 Session
session.setAttribute("userDetails", hugeUserObject);  // 每次都序列化/反序列化

// ✅ 只存储必要信息
session.setAttribute("userId", user.getId());         // 小而简单
session.setAttribute("role", user.getRole());
```

### 配置连接池

```yaml
spring:
  data:
    redis:
      lettuce:
        pool:
          max-active: 32          # 根据并发量调整
          max-idle: 16
          min-idle: 8
          time-between-eviction-runs: 30s  # 空闲连接检查
```

### 使用集群模式

```java
@Bean
public RedisConnectionFactory redisConnectionFactory() {
    RedisClusterConfiguration clusterConfig = new RedisClusterConfiguration()
        .clusterNode("redis-node1", 6379)
        .clusterNode("redis-node2", 6379)
        .clusterNode("redis-node3", 6379);
    clusterConfig.setMaxRedirects(3);
    return new LettuceConnectionFactory(clusterConfig);
}
```

### 合理设置 Flush Mode

```yaml
spring:
  session:
    redis:
      # on-save: 在请求完成时一次性写入 Redis（默认，推荐）
      # immediate: 每次 setAttribute 都立即写入（性能差）
      flush-mode: on-save
```

## 7. Session 失效场景与处理

```java
@Component
public class SessionEventListener {

    // 监听 Session 创建
    @EventListener
    public void handleSessionCreated(SessionCreatedEvent event) {
        String sessionId = event.getSessionId();
        log.info("Session created: {}", sessionId);
    }

    // 监听 Session 销毁
    @EventListener
    public void handleSessionDestroyed(SessionDestroyedEvent event) {
        String sessionId = event.getSessionId();
        log.info("Session destroyed: {}", sessionId);
        // 清理与该 Session 关联的资源
        cleanupUserResources(sessionId);
    }

    // 监听 Session 过期
    @EventListener
    public void handleSessionExpired(SessionExpiredEvent event) {
        String sessionId = event.getSessionId();
        log.info("Session expired: {}", sessionId);
        cleanupSessionResources(sessionId);
    }
}
```

## 8. 自定义 Session 属性限制

```java
// 限制 Session 可存储的属性类型
@EnableRedisHttpSession
@Configuration
public class SecureSessionConfig extends AbstractHttpSessionApplicationInitializer {

    @Bean
    public FindByIndexNameSessionRepository<?> sessionRepository(
            RedisIndexedSessionRepository sessionRepository) {

        // 设置最大属性数，防止恶意填充
        sessionRepository.setDefaultMaxInactiveInterval(1800);

        return sessionRepository;
    }
}
```

## 注意事项

1. **Session 序列化推荐 JSON**：默认的 JDK 序列化体积大、兼容性差，切换到 `GenericJackson2JsonRedisSerializer` 可减少 60%+ 存储空间
2. **Cookie Domain 要精确**：`.example.com` 会匹配所有子域名（包括 `a.b.example.com`），仅需一级子域名用 `.example.com`
3. **Session 超时不要过长**：建议 15-30 分钟，过长浪费 Redis 内存。安全敏感场景建议 5 分钟
4. **避免存储敏感信息**：Session 中不要存密码、Token 等敏感数据，如必须存储则加密后再放入
5. **Session 失效不即时**：Redis TTL 结合 Spring Session 的后台清理可能有数分钟延迟，如需即时失效配合 `Invalidate` 手动调用
6. **内存预估**：每个 Session 大约 300-500 字节（含 Redis 开销），100 万在线用户约需 500MB 内存
7. **断线重连**：Lettuce 默认自动重连，但断开期间 Session 读写会失败，建议配合本地缓存或降级策略
8. **混合架构**：`spring-session` 不支持 JDBC 和 Redis 同时作为 Session 存储，只能用一种
