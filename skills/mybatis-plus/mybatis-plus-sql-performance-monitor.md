---
name: mybatis-plus-sql-performance-monitor
description: MyBatis-Plus SQL 性能监控：慢 SQL 拦截器、执行耗时跟踪与审计日志
tags: [mybatis-plus, orm, performance, sql, monitoring, interceptor]
---

## 概述

MyBatis-Plus 提供了强大的拦截器机制（Interceptor），通过实现 `InnerInterceptor` 接口可以在 SQL 执行前后插入自定义逻辑。本文介绍如何构建 SQL 性能监控系统，包括慢 SQL 检测、执行耗时统计、SQL 审计日志。

## 方案一：基于 PerformanceInterceptor（内置）

MyBatis-Plus 3.5.x 内置了性能分析插件：

```java
@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        // 慢 SQL 阈值：超过 1000ms 输出警告日志
        PerformanceInterceptor performance = new PerformanceInterceptor();
        performance.setMaxTime(1000);
        performance.setFormat(true);
        interceptor.addInnerInterceptor(performance);
        return interceptor;
    }
}
```

配置后会在日志输出：
```
==>  Preparing: SELECT * FROM user WHERE id = ?
==> Parameters: 1(Integer)
<==    Time: 1203ms - Slow SQL detected!
```

## 方案二：自定义慢 SQL 监控拦截器

更适合生产环境的方案，支持报警和聚合统计：

```java
@Slf4j
public class SlowSqlInterceptor implements InnerInterceptor {

    private final long slowThresholdMs;
    private final MeterRegistry meterRegistry;

    public SlowSqlInterceptor(long slowThresholdMs, MeterRegistry meterRegistry) {
        this.slowThresholdMs = slowThresholdMs;
        this.meterRegistry = meterRegistry;
    }

    @Override
    public void beforeQuery(Executor executor, MappedStatement ms, Object parameter,
                            RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
            throws SQLException {
        // 在 ThreadLocal 中记录开始时间
        SqlTimerHolder.start(ms.getId());
    }

    @Override
    public void beforeUpdate(Executor executor, MappedStatement ms, Object parameter)
            throws SQLException {
        SqlTimerHolder.start(ms.getId());
    }

    @Override
    public void afterQuery(Executor executor, MappedStatement ms, Object parameter,
                           RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
            throws SQLException {
        recordExecution(ms, boundSql);
    }

    @Override
    public void afterUpdate(Executor executor, MappedStatement ms, Object parameter)
            throws SQLException {
        recordExecution(ms, null);
    }

    private void recordExecution(MappedStatement ms, BoundSql boundSql) {
        long elapsed = SqlTimerHolder.stop();
        String mapperId = ms.getId();

        // Micrometer 指标记录
        meterRegistry.timer("mybatis.sql.execution", "mapper", mapperId)
                .record(Duration.ofMillis(elapsed));

        log.debug("SQL [{}] executed in {}ms", mapperId, elapsed);

        if (elapsed > slowThresholdMs) {
            String sql = boundSql != null ? boundSql.getSql() : "N/A";
            log.warn("⚠️ 慢SQL告警! mapper={}, 耗时={}ms, SQL={}",
                    mapperId, elapsed, sql);

            // 可集成 Sentry / 阿里云告警
            // AlertManager.notifySlowSql(mapperId, elapsed, sql);
        }
    }
}

class SqlTimerHolder {
    private static final ThreadLocal<Map<String, Long>> TIMER_MAP =
            ThreadLocal.withInitial(HashMap::new);

    static void start(String key) {
        TIMER_MAP.get().put(key, System.currentTimeMillis());
    }

    static long stop() {
        // 实际生产需要更精确的 key 匹配
        long start = TIMER_MAP.get().values().iterator().next();
        TIMER_MAP.remove();
        return System.currentTimeMillis() - start;
    }
}
```

## 方案三：SQL 审计日志拦截器

记录完整的 SQL 执行信息用于安全审计：

```java
@Component
public class SqlAuditInterceptor implements InnerInterceptor {

    private final SqlAuditRepository auditRepository;

    @Override
    public void beforeQuery(Executor executor, MappedStatement ms, Object parameter,
                            RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
        // 转换 SQL 为可读格式
        String sql = SqlUtils.formatSql(boundSql.getSql(), parameter);
        String operator = SecurityContextHolder.getContext().getAuthentication().getName();

        auditRepository.save(new SqlAuditLog(
                operator,
                ms.getId(),
                "SELECT",
                sql,
                Thread.currentThread().getName()
        ));
    }
}
```

## 方案四：使用 MyBatis-Plus 的 SQL 打印 + 第三方 APM

生产环境推荐组合方案：

| 组件 | 作用 |
|------|------|
| MyBatis-Plus 配置 `log-impl: org.apache.ibatis.logging.stdout.StdOutImpl` | 开发调试 |
| SkyWalking / Pinpoint APM agent | 生产链路追踪 |
| P6Spy | SQL 参数打印、耗时统计 |
| 自定义 Interceptor | 慢 SQL 告警 + 指标上报 |

### P6Spy 集成

```xml
<dependency>
    <groupId>p6spy</groupId>
    <artifactId>p6spy</artifactId>
    <version>3.9.1</version>
</dependency>
```

```yaml
spring:
  datasource:
    driver-class-name: com.p6spy.engine.spy.P6SpyDriver
    url: jdbc:p6spy:mysql://localhost:3306/db
```

spy.properties:
```properties
appender=com.p6spy.engine.spy.appender.Slf4jLogger
logMessageFormat=com.p6spy.engine.spy.appender.CustomLineFormat
customLogMessageFormat=%(currentTime)|%(executionTime)|%(sql)
executionThreshold=500
```

## Spring Boot 集成配置

```yaml
mybatis-plus:
  configuration:
    # 生产环境建议关闭 SQL 打印，使用拦截器替代
    # log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
  global-config:
    sql-banner: true
```

配合 Actuator 暴露指标：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: metrics
  metrics:
    tags:
      application: ${spring.application.name}
```

## 注意事项

- **性能开销**：自定义拦截器会轻微增加 SQL 执行耗时，生产环境建议将慢 SQL 阈值设为 500-1000ms
- **ThreadLocal 内存泄漏**：确保在 finally 块或 afterQuery/afterUpdate 中清理 ThreadLocal
- **敏感 SQL 脱敏**：审计日志中避免记录完整的 SQL 参数值（尤其是密码、身份证号）
- **不要在生产用 PerformanceInterceptor**：该插件仅用于开发/测试环境，生产推荐自定义实现
- **分页与拦截器顺序**：`PaginationInnerInterceptor` 必须是最后一个添加的拦截器
- **批量操作**：`beforeUpdate`/`afterUpdate` 可能会因为批量 Executor.BATCH 模式被多次调用
