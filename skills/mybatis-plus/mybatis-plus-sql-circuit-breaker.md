---
name: mybatis-plus-sql-circuit-breaker
description: MyBatis-Plus SQL 超时熔断器——基于拦截器的慢 SQL 快速失败、保护连接池防雪崩
tags: [mybatis-plus, circuit-breaker, sql, performance, connection-pool, interceptor]
---

## 概述

数据库连接池耗尽导致**雪崩**是微服务中常见的故障模式。当某个慢 SQL 长时间占用连接，其他请求排队等待，最终所有线程阻塞、服务不可用。

**SQL 超时熔断器**通过 MyBatis 拦截器拦截 SQL 执行，设置超时阈值，超过阈值的慢 SQL **快速失败**而非继续等待，从而保护连接池。实现按 SQL 指纹维度、双态状态机、多数据源隔离。

## 架构设计

```
请求 → MyBatis Interceptor → 检查 SQL 指纹状态
  ├── CLOSED（正常） → 执行 SQL → 成功（重置计数）/ 超时（增加计数）
  ├── HALF_OPEN（半开）→ 放行一条 → 成功（关闭熔断）/ 失败（重新熔断）
  └── OPEN（熔断）  → 直接抛出 CircuitBreakerException
```

## 完整实现

### 1. 熔断状态枚举

```java
public enum CircuitBreakerState {
    CLOSED,    // 关闭（正常）
    OPEN,      // 打开（熔断）
    HALF_OPEN  // 半开（试探恢复）
}
```

### 2. SQL 指纹提取

```java
public class SqlFingerprint {
    
    /**
     * 提取 SQL 指纹：去参数、统一大小写、标准化空格
     * 不同参数值的同一条 SQL 视为同一个指纹
     */
    public static String fingerprint(String sql) {
        if (sql == null) return "";
        // 1. 统一大小写
        String normalized = sql.toUpperCase();
        // 2. 去除字符串和数字参数
        normalized = normalized.replaceAll("'[^']*'", "?");
        normalized = normalized.replaceAll("\\b\\d+\\b", "?");
        // 3. 标准化空白
        normalized = normalized.replaceAll("\\s+", " ");
        // 4. IN 子句归一化
        normalized = normalized.replaceAll("\\(\\?(,\\?)*\\)", "(?)");
        return normalized.trim();
    }
}
```

### 3. 熔断器核心

```java
@Slf4j
public class SqlCircuitBreaker {
    
    // 每个 SQL 指纹对应一个熔断器实例
    private final ConcurrentHashMap<String, BreakerState> breakers = new ConcurrentHashMap<>();
    
    // 配置
    private final int failureThreshold;       // 触发熔断的连续超时次数（默认 5）
    private final long timeoutMs;             // SQL 超时阈值（默认 3000ms）
    private final long openTimeoutMs;         // 熔断持续时间（默认 30000ms）
    private final int halfOpenMaxRequests;    // 半开状态最大试探请求数（默认 1）
    
    public SqlCircuitBreaker() {
        this(5, 3000, 30000, 1);
    }
    
    public SqlCircuitBreaker(int failureThreshold, long timeoutMs, 
                             long openTimeoutMs, int halfOpenMaxRequests) {
        this.failureThreshold = failureThreshold;
        this.timeoutMs = timeoutMs;
        this.openTimeoutMs = openTimeoutMs;
        this.halfOpenMaxRequests = halfOpenMaxRequests;
    }
    
    /**
     * 检查是否允许执行 SQL
     */
    public boolean tryAcquire(String sql) {
        String fingerprint = SqlFingerprint.fingerprint(sql);
        BreakerState state = breakers.computeIfAbsent(
            fingerprint, k -> new BreakerState());
        return state.tryAcquire();
    }
    
    /**
     * 记录执行结果（成功或超时）
     */
    public void onSuccess(String sql) {
        String fingerprint = SqlFingerprint.fingerprint(sql);
        BreakerState state = breakers.get(fingerprint);
        if (state != null) state.onSuccess();
    }
    
    public void onTimeout(String sql) {
        String fingerprint = SqlFingerprint.fingerprint(sql);
        BreakerState state = breakers.computeIfAbsent(
            fingerprint, k -> new BreakerState());
        state.onFailure();
    }
    
    /**
     * 单个 SQL 指纹的状态机
     */
    class BreakerState {
        private final AtomicReference<CircuitBreakerState> state = 
            new AtomicReference<>(CircuitBreakerState.CLOSED);
        private final AtomicInteger failureCount = new AtomicInteger(0);
        private volatile long lastFailureTime = 0;
        private final AtomicInteger halfOpenRequests = new AtomicInteger(0);
        
        boolean tryAcquire() {
            CircuitBreakerState current = state.get();
            switch (current) {
                case CLOSED:
                    return true;
                case OPEN:
                    // 检查是否超过熔断持续时间
                    if (System.currentTimeMillis() - lastFailureTime > openTimeoutMs) {
                        if (state.compareAndSet(CircuitBreakerState.OPEN, 
                                                CircuitBreakerState.HALF_OPEN)) {
                            halfOpenRequests.set(0);
                            log.warn("熔断器半开，允许试探请求");
                            return tryAcquire(); // 重试
                        }
                    }
                    return false;
                case HALF_OPEN:
                    // 半开状态限制试探请求数量
                    return halfOpenRequests.incrementAndGet() <= halfOpenMaxRequests;
                default:
                    return true;
            }
        }
        
        void onSuccess() {
            CircuitBreakerState current = state.get();
            if (current == CircuitBreakerState.HALF_OPEN) {
                // 半开状态下执行成功 → 关闭熔断
                if (state.compareAndSet(CircuitBreakerState.HALF_OPEN, 
                                        CircuitBreakerState.CLOSED)) {
                    failureCount.set(0);
                    log.info("熔断器关闭，服务已恢复");
                }
            } else if (current == CircuitBreakerState.CLOSED) {
                // 正常状态下成功 → 重置失败计数
                failureCount.set(0);
            }
        }
        
        void onFailure() {
            CircuitBreakerState current = state.get();
            if (current == CircuitBreakerState.CLOSED) {
                int count = failureCount.incrementAndGet();
                if (count >= failureThreshold) {
                    // 连续失败达到阈值 → 打开熔断
                    state.set(CircuitBreakerState.OPEN);
                    lastFailureTime = System.currentTimeMillis();
                    log.error("熔断器打开：SQL 指纹 {} 连续超时 {} 次", 
                        SqlFingerprint.fingerprint("***"), count);
                }
            } else if (current == CircuitBreakerState.HALF_OPEN) {
                // 半开状态下失败 → 重新熔断
                state.set(CircuitBreakerState.OPEN);
                lastFailureTime = System.currentTimeMillis();
                log.error("熔断器重新打开：半开试探请求失败");
            }
        }
    }
}
```

### 4. MyBatis 拦截器集成

```java
@Component
@Intercepts({
    @Signature(type = StatementHandler.class, method = "query", args = {Statement.class, ResultHandler.class}),
    @Signature(type = StatementHandler.class, method = "update", args = {Statement.class}),
    @Signature(type = StatementHandler.class, method = "batch", args = {Statement.class})
})
public class SqlCircuitBreakerInterceptor implements Interceptor {
    
    private final SqlCircuitBreaker breaker;
    private final Set<String> bypassPatterns; // 白名单 SQL 不熔断
    
    public SqlCircuitBreakerInterceptor() {
        this.breaker = new SqlCircuitBreaker(5, 3000, 30000, 1);
        this.bypassPatterns = new HashSet<>(List.of("SELECT 1", "SHOW", "COMMIT", "ROLLBACK"));
    }
    
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        StatementHandler handler = (StatementHandler) invocation.getTarget();
        BoundSql boundSql = handler.getBoundSql();
        String sql = boundSql.getSql();
        
        // 白名单跳过
        if (bypassPatterns.stream().anyMatch(p -> sql.toUpperCase().startsWith(p))) {
            return invocation.proceed();
        }
        
        // 检查熔断状态
        if (!breaker.tryAcquire(sql)) {
            throw new CircuitBreakerOpenException(
                "SQL 熔断: " + SqlFingerprint.fingerprint(sql) 
                + " - 服务暂时不可用，请稍后重试");
        }
        
        long start = System.currentTimeMillis();
        try {
            Object result = invocation.proceed();
            long elapsed = System.currentTimeMillis() - start;
            if (elapsed > breaker.getTimeoutMs()) {
                breaker.onTimeout(sql);
            } else {
                breaker.onSuccess(sql);
            }
            return result;
        } catch (Exception e) {
            // SQL 执行异常也视为失败
            breaker.onTimeout(sql);
            throw e;
        }
    }
    
    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }
    
    @Override
    public void setProperties(Properties properties) {
        // 可以从配置读取参数
    }
}
```

### 5. 配置启动熔断器

```java
@Configuration
public class MybatisPlusConfig {

    @Bean
    public SqlCircuitBreakerInterceptor sqlCircuitBreakerInterceptor() {
        return new SqlCircuitBreakerInterceptor();
    }
    
    @Bean
    @ConditionalOnProperty(prefix = "mybatis-plus.circuit-breaker", name = "enabled", havingValue = "true")
    public ConfigurationCustomizer mybatisPlusCustomizer(
            SqlCircuitBreakerInterceptor interceptor) {
        return configuration -> {
            configuration.addInterceptor(interceptor);
        };
    }
}
```

```yaml
# application.yml
mybatis-plus:
  circuit-breaker:
    enabled: true
    failure-threshold: 5          # 连续超时 5 次触发熔断
    timeout-ms: 3000              # SQL 执行超时阈值
    open-timeout-ms: 30000        # 熔断持续时间 30 秒
    half-open-max-requests: 1     # 半开试探请求数
```

### 6. 异常处理

```java
public class CircuitBreakerOpenException extends RuntimeException {
    public CircuitBreakerOpenException(String message) {
        super(message);
    }
}

@RestControllerAdvice
public class CircuitBreakerExceptionHandler {

    @ExceptionHandler(CircuitBreakerOpenException.class)
    public ResponseEntity<Map<String, Object>> handleCircuitBreaker(CircuitBreakerOpenException e) {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of(
                "code", 503,
                "message", e.getMessage(),
                "hint", "该 SQL 当前被熔断，请稍后重试或联系管理员"
            ));
    }
}
```

## 多数据源隔离

```java
// 每个数据源独立熔断器
public class DataSourceAwareCircuitBreaker {
    
    private final Map<String, SqlCircuitBreaker> breakers = new ConcurrentHashMap<>();
    
    public SqlCircuitBreaker getBreaker(String dataSourceName) {
        return breakers.computeIfAbsent(dataSourceName, 
            k -> new SqlCircuitBreaker(5, getTimeoutForDataSource(k), 30000, 1));
    }
    
    private long getTimeoutForDataSource(String name) {
        // 读库可以给更长超时，写库更严格
        return name.contains("read") ? 5000 : 2000;
    }
}
```

## Metrics 监控

```java
// 暴露 Prometheus 指标
@Component
public class CircuitBreakerMetrics {
    
    private final MeterRegistry registry;
    private final SqlCircuitBreaker breaker;
    
    public CircuitBreakerMetrics(MeterRegistry registry, SqlCircuitBreaker breaker) {
        this.registry = registry;
        this.breaker = breaker;
        
        // 定期收集每个 SQL 指纹的状态
        Gauge.builder("mybatis.circuitbreaker.open.count", breaker,
            b -> b.getOpenCount())
            .description("当前熔断的 SQL 指纹数量")
            .register(registry);
    }
}
```

## 注意事项

1. **超时时间设置**：不要设置太短（误杀正常 SQL）或太长（失去保护意义）。建议从 3s 开始，根据 P99 延迟调整
2. **熔断粒度**：按 SQL 指纹（相同 SQL 不同参数）而非按表或数据源，避免一条慢 SQL 影响整个业务
3. **白名单机制**：健康检查、心跳 SQL 必须加入白名单跳过熔断
4. **与数据库超时配合**：熔断器超时应 ≤ `mysql-connector` 的 `socketTimeout`，否则连接池还是会被占满
5. **熔断恢复**：半开状态的试探请求数不宜过多（1-2 个即可），防止恢复瞬间压垮数据库
6. **缓存配合**：熔断的查询如果有缓存，应从缓存返回而非直接抛异常
7. **记录熔断日志**：每次熔断切换（OPEN/CLOSED/HALF_OPEN）必须记录 WARN/ERROR 日志，便于排查
