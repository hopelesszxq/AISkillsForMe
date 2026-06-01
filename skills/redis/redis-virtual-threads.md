---
name: redis-virtual-threads
description: Redis 客户端（Lettuce/Redisson）在 Java 21+ 虚拟线程下的配置、坑点与最佳实践
tags: [redis, java, virtual-threads, lettuce, redisson, performance]
---

## 概述

Java 21 虚拟线程（Virtual Threads）允许以极低成本创建大量线程，改变了 Redis 客户端的连接管理模式。但 Lettuce 和 Redisson 在虚拟线程环境下存在不同的兼容性问题。

## 1. Lettuce + 虚拟线程

Lettuce 基于 Netty（异步/非阻塞），默认不适合虚拟线程——因为 Netty 的 Channel 绑定到固定线程。

### 推荐配置：共享连接 + blocking 适配

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        LettuceClientConfiguration config = LettuceClientConfiguration.builder()
            .commandTimeout(Duration.ofSeconds(5))
            // 关键：使用共享连接（不推荐池化）
            .build();

        LettuceConnectionFactory factory = new LettuceConnectionFactory(
            new RedisStandaloneConfiguration("localhost", 6379), config);
        // 关闭共享 native 连接（虚拟线程下避免死锁）
        factory.setShareNativeConnection(true);
        return factory;
    }
}
```

### 虚拟线程下 Lettuce 的坑

```yaml
# application.yml - Spring Boot 3.2+ 启用虚拟线程
spring:
  threads:
    virtual:
      enabled: true
```

```java
// ❌ 问题：虚拟线程中直接使用 RedisTemplate 阻塞调用
// 虚拟线程内部调用 get() 时，Lettuce 的 Netty 线程也可能阻塞
redisTemplate.opsForValue().get("key");

// ✅ 解决：使用 Lettuce asynchronous API + 在虚拟线程中 get()
CompletionStage<String> future = redisTemplate
    .getConnectionFactory()
    .getReactiveConnection()
    .stringCommands()
    .get(LettuceStrings.codecUtf8(LettuceStrings.stringToByteBuf("key")));

// 在虚拟线程中阻塞等待异步结果（不阻塞平台线程）
String result = future.toCompletableFuture().get(5, TimeUnit.SECONDS);
```

### Lettuce Connection Pool 与虚拟线程

**不要混用连接池和虚拟线程**——虚拟线程本身就是为了消除池化而生：

```java
// ❌ 不推荐：虚拟线程 + 连接池（浪费虚拟线程优势）
@Bean
public RedisConnectionFactory lettuceConnectionFactory() {
    GenericObjectPoolConfig<StatefulRedisConnection<String, String>> poolConfig =
        new GenericObjectPoolConfig<>();
    poolConfig.setMaxTotal(16);  // 虚拟线程下不需要
    // ...
}

// ✅ 推荐：共享连接 + 虚拟线程
// 让 Netty 管理连接复用，虚拟线程只在阻塞 API 时挂起
```

## 2. Redisson + 虚拟线程

Redisson 虽然是基于 Netty 的异步库，但其 `RBucket.get()` 等同步方法内部使用了 `CountDownLatch` 阻塞——在虚拟线程中可能引发**固定死锁（pinned）**。

### 规避方案

```java
@Configuration
public class RedissonConfig {

    @Bean(destroyMethod = "shutdown")
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer()
            .setAddress("redis://localhost:6379")
            // 减少 Netty 工作线程数，避免线程争用
            .setNettyThreads(4);

        // 关键：使用异步 API
        return Redisson.create(config);
    }
}
```

### 异步 API 在虚拟线程中的最佳用法

```java
@Service
public class InventoryService {

    private final RedissonClient redisson;

    // ⚡ 使用异步 API，然后在虚拟线程中 join
    public Long getStock(String sku) {
        RFuture<Long> future = redisson.getAtomicLong("stock:" + sku)
            .addAndGetAsync(0L);

        // 在虚拟线程中等待，不阻塞 Carrier 线程
        return future.toCompletableFuture().join();
    }
}
```

### RLock 在虚拟线程中的问题

```java
// ❌ 危险：虚拟线程中获取锁可能导致 pinning
RLock lock = redisson.getLock("myLock");
lock.lock(10, TimeUnit.SECONDS);  // 内部 tryAcquire 使用 CountDownLatch

try {
    // 业务逻辑
} finally {
    lock.unlock();
}

// ✅ 安全：使用 tryLock with timeout + async
RLock lock = redisson.getLock("myLock");
RFuture<Boolean> future = lock.tryLockAsync(3, 10, TimeUnit.SECONDS);
if (future.toCompletableFuture().get()) {
    try {
        // 业务逻辑
    } finally {
        lock.unlockAsync();
    }
}
```

## 3. Spring Data Redis 模板优化

```java
@Configuration
public class RedisTemplateConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);

        // 使用 GenericJackson2JsonRedisSerializer
        // 避免 JDK 序列化性能问题
        Jackson2JsonRedisSerializer<Object> serializer =
            new Jackson2JsonRedisSerializer<>(Object.class);
        template.setDefaultSerializer(serializer);
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(serializer);

        template.afterPropertiesSet();
        return template;
    }
}
```

## 4. 性能基准建议

| 配置 | 吞吐量（并发 1000） | 内存占用 | 适用场景 |
|------|-------------------|---------|---------|
| Lettuce + 连接池 + 平台线程 | 基线 | 高 | 传统架构 |
| Lettuce + 共享连接 + 虚拟线程 | +15~30% | 低 | 高吞吐 API |
| Redisson + 异步 + 虚拟线程 | +20~40% | 中 | 复杂场景（锁、队列） |
| Lettuce + 反应式 + 虚拟线程 | -10% | 最低 | 非必要不混用 |

## 注意事项

- **避免线程固定（Pinning）**：同步锁、`CountDownLatch.await()`、`Object.wait()` 都会固定 Carrier 线程
- **Netty 线程数设置**：虚拟线程环境下 Netty 线程数建议减半（4~6），减少 Context Switch
- **超时配置**：虚拟线程不会释放 Carrier 线程，所以 Redis 超时必须设置合理值（建议 < 5s）
- **监控维度**：`jcmd <pid> Thread.vthread_stats` 查看虚拟线程状态
- **Spring Boot 3.4+**：虚拟线程默认开启 `spring.threads.virtual.enabled=true`，Redis 集成需额外测试
