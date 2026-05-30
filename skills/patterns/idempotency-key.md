---
name: idempotency-key
description: Spring Boot HTTP Idempotency-Key 头实现：幂等键注解、AOP 切面、并发控制与结果重放
tags: [patterns, spring-boot, idempotency, rest-api]
---

## 要点

HTTP `Idempotency-Key` 用于确保 POST/PATCH 等非幂等请求在重试时只执行一次：

- 客户端发送 `Idempotency-Key` 请求头（如 `7f3b2c1c-4f4f-4c8d-9b7e-0f18d4d0b2aa`）
- 首次请求正常执行并存储结果
- 相同 key 的后续请求直接返回已存储结果（重放）
- 并发请求（相同 key 正在处理中）返回 `409 Conflict`

> 不要跟分布式事务 / 消息去重的幂等混淆，这是**HTTP API 层面的幂等键实现**。

## 方案设计

### 存储模型

```sql
CREATE TABLE idempotency_record (
    idempotency_key VARCHAR(200) PRIMARY KEY,
    status          VARCHAR(20)  NOT NULL,     -- IN_PROGRESS / COMPLETED
    response_status INT          NULL,
    content_type    VARCHAR(100) NULL,
    response_body   TEXT         NULL,
    created_at      TIMESTAMP    NOT NULL,
    updated_at      TIMESTAMP    NOT NULL,
    expires_at      TIMESTAMP    NOT NULL
);
```

### 实现架构

1. 注解 `@Idempotent` 标记需要幂等保护的端点
2. Spring AOP `@Around` 切面拦截：
   - 读取 `Idempotency-Key` 请求头
   - 构建内部 key（method + path + client key）
   - 尝试插入 `IN_PROGRESS` 记录（并发锁）
   - 如果已 `COMPLETED` → 重放结果
   - 如果 `IN_PROGRESS` → 返回 `409`
   - 执行后存储返回结果

## 代码示例

### 1. @Idempotent 注解

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Idempotent {
    /** TTL 秒数（默认 24h） */
    long ttlSeconds() default 24 * 60 * 60;
}
```

### 2. JPA Entity + Repository

```java
@Entity
@Table(name = "idempotency_record")
public class IdempotencyRecord {

    @Id
    @Column(name = "idempotency_key", length = 200)
    private String idempotencyKey;

    @Column(nullable = false, length = 20)
    private String status; // IN_PROGRESS / COMPLETED

    @Column(name = "response_status")
    private Integer responseStatus;

    @Column(name = "content_type", length = 100)
    private String contentType;

    @Lob
    @Column(name = "response_body")
    private String responseBody;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    @Column(name = "expires_at", nullable = false)
    private Instant expiresAt;

    public static IdempotencyRecord inProgress(String key, Instant now, Instant expiresAt) {
        IdempotencyRecord r = new IdempotencyRecord();
        r.idempotencyKey = key;
        r.status = "IN_PROGRESS";
        r.createdAt = now;
        r.updatedAt = now;
        r.expiresAt = expiresAt;
        return r;
    }

    public void complete(int status, String contentType, String body, Instant now) {
        this.status = "COMPLETED";
        this.responseStatus = status;
        this.contentType = contentType;
        this.responseBody = body;
        this.updatedAt = now;
    }
}

public interface IdempotencyRecordRepository extends JpaRepository<IdempotencyRecord, String> {
    long deleteByExpiresAtBefore(Instant now);
}
```

### 3. 核心 Service（并发控制 + 结果重放）

```java
@Service
public class IdempotencyService {

    public enum AcquireState { ACQUIRED, IN_PROGRESS, COMPLETED }

    public static class AcquireResult {
        private final AcquireState state;
        private final ResponseEntity<String> replay;

        private AcquireResult(AcquireState state, ResponseEntity<String> replay) {
            this.state = state;
            this.replay = replay;
        }

        public static AcquireResult acquired() { return new AcquireResult(AcquireState.ACQUIRED, null); }
        public static AcquireResult inProgress() { return new AcquireResult(AcquireState.IN_PROGRESS, null); }
        public static AcquireResult completed(ResponseEntity<String> replay) {
            return new AcquireResult(AcquireState.COMPLETED, replay);
        }

        public AcquireState state() { return state; }
        public ResponseEntity<String> replay() { return replay; }
    }

    private final IdempotencyRecordRepository repo;
    private final ObjectMapper objectMapper;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public AcquireResult acquire(String internalKey, long ttlSeconds) {
        Instant now = Instant.now();
        Instant expiresAt = now.plus(ttlSeconds, ChronoUnit.SECONDS);
        try {
            repo.saveAndFlush(IdempotencyRecord.inProgress(internalKey, now, expiresAt));
            return AcquireResult.acquired();
        } catch (DataIntegrityViolationException dup) {
            Optional<IdempotencyRecord> existing = repo.findById(internalKey);
            if (existing.isEmpty()) return AcquireResult.inProgress();
            IdempotencyRecord r = existing.get();
            if ("COMPLETED".equals(r.getStatus())) {
                return AcquireResult.completed(replay(r));
            }
            return AcquireResult.inProgress();
        }
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void completeSuccess(String internalKey, ResponseEntity<?> response) {
        repo.findById(internalKey).ifPresent(r -> {
            if (!response.getStatusCode().is2xxSuccessful()) return;
            try {
                String bodyJson = response.getBody() == null
                    ? "" : objectMapper.writeValueAsString(response.getBody());
                String contentType = response.getHeaders().getContentType() == null
                    ? MediaType.APPLICATION_JSON_VALUE
                    : response.getHeaders().getContentType().toString();
                r.complete(response.getStatusCodeValue(), contentType, bodyJson, Instant.now());
                repo.save(r);
            } catch (Exception ignore) { /* 序列化失败不缓存 */ }
        });
    }

    private ResponseEntity<String> replay(IdempotencyRecord r) {
        MediaType ct = r.getContentType() == null
            ? MediaType.APPLICATION_JSON
            : MediaType.parseMediaType(r.getContentType());
        int status = (r.getResponseStatus() == null) ? 200 : r.getResponseStatus();
        String body = (r.getResponseBody() == null) ? "" : r.getResponseBody();
        return ResponseEntity.status(status).contentType(ct).body(body);
    }
}
```

> `REQUIRES_NEW` 确保幂等记录的事务与业务事务隔离，避免业务事务回滚时幂等记录也被撤销。

### 4. AOP 切面

```java
@Aspect
@Component
public class IdempotencyAspect {

    private final IdempotencyService service;

    @Around("@annotation(idempotent)")
    public Object around(ProceedingJoinPoint pjp, Idempotent idempotent) throws Throwable {
        HttpServletRequest req = currentRequest();
        String key = req.getHeader("Idempotency-Key");
        if (key == null || key.isBlank()) {
            return ResponseEntity.badRequest()
                .body(new SimpleError("idempotency.key.required", "Missing Idempotency-Key"));
        }
        String internalKey = buildInternalKey(req, key);
        IdempotencyService.AcquireResult ar = service.acquire(internalKey, idempotent.ttlSeconds());
        if (ar.state() == IdempotencyService.AcquireState.COMPLETED) {
            return ar.replay();
        }
        if (ar.state() == IdempotencyService.AcquireState.IN_PROGRESS) {
            return ResponseEntity.status(409)
                .body(new SimpleError("idempotency.in_progress", "Request with the same key is in progress"));
        }
        Object result = pjp.proceed();
        if (result instanceof ResponseEntity<?> re) {
            service.completeSuccess(internalKey, re);
        }
        return result;
    }

    private static HttpServletRequest currentRequest() {
        RequestAttributes attrs = RequestContextHolder.currentRequestAttributes();
        return ((ServletRequestAttributes) attrs).getRequest();
    }

    private static String buildInternalKey(HttpServletRequest req, String clientKey) {
        return req.getMethod() + ":" + req.getRequestURI() + ":" + clientKey.trim();
    }

    public record SimpleError(String error, String message) {}
}
```

### 5. 在 Controller 中使用

```java
@RestController
@RequestMapping("/api/payments")
public class PaymentController {

    @PostMapping
    @Idempotent(ttlSeconds = 24 * 60 * 60)
    public ResponseEntity<PaymentResponse> create(@RequestBody CreatePaymentRequest req) {
        String paymentId = "pay_" + System.currentTimeMillis();
        return ResponseEntity.ok(new PaymentResponse(paymentId, "CREATED"));
    }

    public record CreatePaymentRequest(@NotBlank String orderId) {}
    public record PaymentResponse(String paymentId, String status) {}
}
```

### 6. 定时清理任务

```java
@Component
public class IdempotencyCleanupJob {

    private final IdempotencyRecordRepository repo;

    @Scheduled(cron = "0 0 * * * *") // 每小时
    public void cleanup() {
        repo.deleteByExpiresAtBefore(Instant.now());
    }
}
```

### 7. 测试（MockMvc 验证重放）

```java
@SpringBootTest
@AutoConfigureMockMvc
class IdempotencyKeyTest {

    @Autowired MockMvc mvc;

    @Test
    void sameKey_replaysSameResponse() throws Exception {
        String body = "{\"orderId\":\"o_1\"}";

        String r1 = mvc.perform(post("/api/payments")
                .contentType(MediaType.APPLICATION_JSON)
                .header("Idempotency-Key", "abc-123")
                .content(body))
            .andExpect(status().isOk())
            .andReturn().getResponse().getContentAsString();

        String r2 = mvc.perform(post("/api/payments")
                .contentType(MediaType.APPLICATION_JSON)
                .header("Idempotency-Key", "abc-123")
                .content(body))
            .andExpect(status().isOk())
            .andReturn().getResponse().getContentAsString();

        assertEquals(r1, r2);
    }
}
```

## 注意事项

| 问题 | 建议 |
|------|------|
| **客户端复用 key 但换 body** | 存储 `request_hash`，匹配到 key 但 hash 不同时返回 409 |
| **只缓存 2xx 吗** | 支付/订单场景建议只缓存成功响应，避免缓存敏感错误信息 |
| **ACQUIRE 后崩溃** | 留下 `IN_PROGRESS` 记录，可加"过期检测"（updated_at > N 分钟）允许重试 |
| **别用 traceId 当幂等键** | Trace ID 每次请求不同，幂等键必须跨重试稳定 |
| **key 唯一性** | 内部 key = `METHOD:path:client_key`，跨用户碰撞时追加 userId |
| **性能注意** | `REQUIRES_NEW` 会挂起外层事务，高频场景建议使用 Redis 替代数据库存储 |
