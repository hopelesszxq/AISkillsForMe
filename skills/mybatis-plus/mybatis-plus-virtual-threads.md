---
name: mybatis-plus-virtual-threads
description: MyBatis-Plus 虚拟线程（Virtual Threads）配置最佳实践：ThreadLocal 问题、连接池调优、分页兼容
tags: [mybatis-plus, virtual-threads, loom, jdk21, performance, threadpool]
---

## 概述

JDK 21+ 虚拟线程（Virtual Threads / Project Loom）允许以极低成本创建大量线程，但 MyBatis-Plus 内部大量使用 **ThreadLocal**（分页上下文、SqlSession、事务同步等），在虚拟线程场景下需要特殊处理以避免 **线程钉死（Pinning）** 和数据错乱。

## 核心问题

| 问题 | 原因 | 影响 |
|------|------|------|
| 线程钉死 | synchronized 块内使用虚拟线程，导致载体线程被锁定 | 连接池耗尽，吞吐量下降 |
| ThreadLocal 泄漏 | 虚拟线程池不可复用，每次创建新线程，ThreadLocal 用完不被清理 | 内存增长，OOM 风险 |
| 分页数据错乱 | `PaginationInnerInterceptor` 使用 `ThreadLocal` 存储 Page 对象，虚拟线程复用场景下可能交叉污染 | 分页结果不正确 |

## 1. 数据库连接池配置

虚拟线程场景下连接池的配置与平台线程完全不同，核心原则是：**虚拟线程不共享连接池**。

```yaml
spring:
  datasource:
    # HikariCP 虚拟线程调优配置
    hikari:
      maximum-pool-size: 20        # 等于数据库 max_connections 的 1/2
      minimum-idle: 5
      max-lifetime: 600000         # 10 分钟（防防火墙断开）
      keepalive-time: 120000       # 2 分钟（虚拟线程下建议启用）
      connection-timeout: 5000     # 5 秒
      idle-timeout: 300000         # 5 分钟
      pool-name: VirtualPool
      # 关键：虚拟线程场景允许更长的超时
      validation-timeout: 3000
```

> 注意：虚拟线程的数量（百万级）远大于连接池大小（10-50），所以**并发 SQL 请求最终由连接池串行化**。不需要调大连接池，而是需要合理设计业务逻辑。

## 2. MyBatis-Plus 分页兼容配置

分页拦截器使用 `ThreadLocal` 传递 `Page` 对象，在虚拟线程环境下需额外注意：

```java
@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 分页插件
        PaginationInnerInterceptor pagination = new PaginationInnerInterceptor(DbType.MYSQL);
        // 关键配置：超过最大页数时返回空，而不是抛异常
        pagination.setOverflow(true);
        // 单页最大 500 条
        pagination.setMaxLimit(500L);
        // 虚拟线程友好：优化 page count 查询
        pagination.setOptimizeJoin(true);

        interceptor.addInnerInterceptor(pagination);

        // 乐观锁插件
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());

        return interceptor;
    }

    /**
     * 使用 Servlet 虚拟线程执行器
     * Spring Boot 4 + Tomcat + JDK 21+
     */
    @Bean
    public AsyncTaskExecutor virtualThreadTaskExecutor() {
        return new VirtualThreadTaskExecutor("mybatis-async-");
    }
}
```

### 分页的正确使用方式（虚拟线程安全）

```java
@Service
public class UserService {

    /**
     * ✅ 正确：通过方法参数传递 Page，不依赖 ThreadLocal
     */
    public Page<UserVO> queryUsers(Page<User> page, String keyword) {
        // Page 对象作为参数传入 Mapper，拦截器从参数获取而非 ThreadLocal
        return userMapper.selectPage(page,
            Wrappers.<User>lambdaQuery()
                .like(User::getName, keyword)
                .orderByDesc(User::getCreateTime)
        );
    }

    /**
     * ❌ 避免：使用 RequestContextHolder / ThreadLocal 传递分页参数
     * 虚拟线程下可能拿到错误的数据
     */
    // @Deprecated
    // public Page<UserVO> badPattern() { ... }
}
```

## 3. 事务管理（@Transactional 与虚拟线程）

`@Transactional` 依赖 `TransactionSynchronizationManager`（基于 ThreadLocal），虚拟线程下工作正常但需注意：

```java
@Service
public class OrderService {

    /**
     * ✅ 正确：短事务，虚拟线程友好
     */
    @Transactional
    public void createOrder(Order order) {
        orderMapper.insert(order);
        // 少量更新
        inventoryMapper.reduceStock(order.getProductId(), order.getQuantity());
    }

    /**
     * ❌ 避免：长事务，持有数据库连接过久
     * 虚拟线程任务大量排队时，连接池会迅速耗尽
     */
    @Transactional(timeout = 5)  // 必须加超时
    public void slowBatchProcess(List<Order> orders) {
        for (Order order : orders) {
            // 每个循环处理一个订单，可能包含远程调用
            processOne(order);  // 远程调用期间持有数据库连接！
        }
    }

    /**
     * ✅ 推荐：异步 + 独立事务
     */
    @EventListener
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void handleOrderCreated(OrderCreatedEvent event) {
        // 在单独的事务中处理，缩短连接持有时间
    }
}
```

### 虚拟线程环境下 @Transactional 注意事项

```java
@Configuration
public class TransactionConfig {

    /**
     * 虚拟线程事务管理器配置
     * 关键：禁用 "allow-bean-definition-overriding" 避免冲突
     */
    @Bean
    @ConditionalOnThreading(Threading.VIRTUAL)
    public TransactionTemplate virtualThreadTransactionTemplate(
            PlatformTransactionManager transactionManager) {
        TransactionTemplate template = new TransactionTemplate(transactionManager);
        template.setTimeout(10); // 虚拟线程下必须设超时
        template.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
        return template;
    }
}
```

## 4. SQL 会话管理

MyBatis 的 `SqlSession` 默认绑定到 ThreadLocal，虚拟线程没问题，但每次操作都会新建 `SqlSession`。建议启用 **batch mode** 减少开销：

```yaml
mybatis-plus:
  configuration:
    default-executor-type: simple   # simple / reuse / batch
    # 虚拟线程推荐 reuse（复用 PreparedStatement）
    # 批量操作较多时用 batch
    local-cache-scope: session       # session / statement
    # 虚拟线程下建议 statement 级别缓存，避免 ThreadLocal 跨任务泄漏
```

## 5. Spring Boot 4 / MyBatis-Plus 3.5.16+ 虚拟线程支持

```yaml
spring:
  threads:
    virtual:
      enabled: true    # Spring Boot 4 全局启用虚拟线程
  task:
    execution:
      thread-name-prefix: virtual-task-
```

```java
// 虚拟线程 + MyBatis-Plus 兼容性检查
@Component
public class VirtualThreadCompatibilityChecker {

    @PostConstruct
    public void check() {
        // 确认运行在虚拟线程环境
        if (Thread.currentThread().isVirtual()) {
            log.info("✅ Running with Virtual Threads");
            log.info("✅ MyBatis-Plus version: {}", MybatisPlusVersion.getVersion());
        }

        // 验证分页插件兼容性
        try {
            Class.forName("com.baomidou.mybatisplus.extension.plugins.pagination.Page");
            log.info("✅ Page class loadable in virtual thread context");
        } catch (ClassNotFoundException e) {
            log.warn("⚠️  Page class not found");
        }
    }
}
```

## 6. 性能监控（获取虚拟线程统计）

```java
@Component
public class VirtualThreadMetrics {

    /**
     * 监控虚拟线程池 + MyBatis 执行情况
     */
    @Scheduled(fixedRate = 60000)
    public void reportMetrics() {
        // 虚拟线程池状态（JDK 21+）
        Thread.Builder builder = Thread.ofVirtual().name("metrics-checker");
        // 已创建的虚拟线程总数
        long createdCount = ManagementFactory.getThreadMXBean().getThreadCount();

        // HikariCP 连接池状态
        if (dataSource instanceof HikariDataSource hds) {
            HikariPoolMXBean poolBean = hds.getHikariPoolMXBean();
            log.info("Active: {} / Idle: {} / Pending: {} / Total: {}",
                poolBean.getActiveConnections(),
                poolBean.getIdleConnections(),
                poolBean.getThreadsAwaitingConnection(),
                poolBean.getTotalConnections()
            );
        }
    }
}
```

## 注意事项

- **不要调大连接池**：虚拟线程数量远大于连接池，调大连接池不会提升数据库吞吐，反而增加数据库压力
- **必须设置事务超时**：虚拟线程下事务可能被长时间挂起，不设超时会永久占用连接
- **使用 try-with-resources**：确保 InputStream、ResultSet 等资源在虚拟线程中被及时关闭
- **WARN 日志监控**：注意 `HikariPool` 的 `Thread starvation` 警告，说明虚拟线程等待连接超时
- **Page 对象安全**：每个请求创建新的 Page 对象，不要复用或缓存
- **Spring Boot 3.x 兼容**：如果使用 Spring Boot 3.x + Tomcat 10.1 + JDK 21，需额外配置 `spring.threads.virtual.enabled=true`（Spring Boot 3.2+ 支持）
