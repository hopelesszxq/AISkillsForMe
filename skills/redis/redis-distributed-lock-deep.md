---
name: redis-distributed-lock-deep
description: Redis 分布式锁深度实战：Redisson 看门狗、可重入、红锁、Spring Integration 集成与高可用方案
tags: [redis, distributed-lock, redisson, concurrency, redlock]
---

## 概述

Redis 分布式锁是微服务中解决并发冲突的常见方案。相比数据库悲观锁，Redis 锁性能更高；相比 Zookeeper 锁，Redis 延迟更低。本文覆盖从原生 SETNX 到 Redisson 高阶用法的完整实践。

## 1. 原生 SET NX 实现（基础版 — 不推荐生产）

```java
/**
 * 纯 Redis 命令实现分布式锁（仅做原理说明）
 * 问题：没有 watchdog 续期，业务超时锁自动释放，其他线程拿到锁
 */
public class SimpleRedisLock {
    private static final String LOCK_PREFIX = "lock:";
    private static final long LOCK_TIMEOUT_MS = 30000;

    private final StringRedisTemplate redisTemplate;

    public boolean tryLock(String key, String requestId) {
        // SET key value NX PX 30000 — 原子操作
        return Boolean.TRUE.equals(
            redisTemplate.opsForValue()
                .setIfAbsent(LOCK_PREFIX + key, requestId,
                             Duration.ofMillis(LOCK_TIMEOUT_MS))
        );
    }

    public boolean unlock(String key, String requestId) {
        // Lua 脚本保证：只有持有者才能释放
        String script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
            """;
        return Boolean.TRUE.equals(
            redisTemplate.execute(
                new DefaultRedisScript<>(script, Long.class),
                List.of(LOCK_PREFIX + key),
                requestId
            )
        );
    }
}
```

## 2. Redisson 集成（推荐生产使用）

### 依赖与配置

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.34.0</version>
</dependency>
```

```yaml
# application.yml
spring:
  data:
    redis:
      host: 192.168.1.100
      port: 6379
      password: ${REDIS_PASSWORD}
      timeout: 3000ms
      lettuce:
        pool:
          max-active: 16
          max-idle: 8
          min-idle: 4

# Redisson 独立配置（可覆盖 Spring Data Redis）
redisson:
  single-server-config:
    address: redis://192.168.1.100:6379
    password: ${REDIS_PASSWORD}
    connection-pool-size: 16
    timeout: 3000
```

### 基础用法

```java
@Service
public class OrderLockService {

    @Autowired
    private RedissonClient redissonClient;

    public void createOrder(Long orderId) {
        RLock lock = redissonClient.getLock("order:" + orderId);

        try {
            // 等待 5 秒，锁过期时间 30 秒（看门狗会自动续期）
            boolean locked = lock.tryLock(5, 30, TimeUnit.SECONDS);
            if (!locked) {
                throw new BusinessException("系统繁忙，请稍后重试");
            }

            // 业务逻辑
            doCreateOrder(orderId);

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new BusinessException("操作被中断");
        } finally {
            // 必须释放锁（Redisson 自动识别持有者）
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

## 3. 看门狗机制（Watch Dog）

```java
@Service
public class WatchDogDemo {

    @Autowired
    private RedissonClient redissonClient;

    /**
     * 看门狗工作原理：
     *
     * 1. lock.tryLock() 默认过期时间 30 秒
     * 2. Redisson 启动一个后台定时任务（每 10 秒执行一次）
     * 3. 如果业务还没执行完，自动将锁过期时间续期到 30 秒
     * 4. 业务完成后 unlock()，后台任务取消
     * 5. 如果应用崩溃，锁最多 30 秒后自动释放（不会死锁）
     *
     * 注意：如果自己指定了 leaseTime（如 tryLock(5, 10, SECONDS)），
     *       看门狗不会生效，锁到期自动释放
     */
    public void longRunningTask(Long taskId) {
        // ❌ 如果业务超过 30 秒，没有看门狗的情况下锁会自动释放
        // RLock lock = redissonClient.getLock("task:" + taskId);
        // lock.lock(30, TimeUnit.SECONDS);  // 不会续期！

        // ✅ 不指定 leaseTime，看门狗生效
        RLock lock = redissonClient.getLock("task:" + taskId);
        lock.lock();  // 默认 30 秒续期

        try {
            // 可能运行 2 分钟的耗时任务
            processLongTask(taskId);
        } finally {
            lock.unlock();
        }
    }

    /**
     * 自定义看门狗续期时间
     */
    public void withCustomTimeout(Long taskId) {
        RLock lock = redissonClient.getLock("task:" + taskId);
        // 锁的超时时间改为 60 秒，看门狗每 20 秒续期一次
        lock.lock(60, TimeUnit.SECONDS);
        // 但这样看门狗就不工作了！必须用下面的方式：
        // Config config = new Config();
        // config.setLockWatchdogTimeout(60_000); // 设置看门狗超时
        try {
            doWork(taskId);
        } finally {
            lock.unlock();
        }
    }
}
```

## 4. 可重入锁与公平锁

```java
@Service
public class LockAdvancedService {

    @Autowired
    private RedissonClient redissonClient;

    /**
     * 可重入锁：同一线程可多次获取同一把锁
     * 内部使用计数器实现，lock() N 次需要 unlock() N 次
     */
    public void reentrantDemo(Long orderId) {
        RLock lock = redissonClient.getLock("order:" + orderId);

        lock.lock();
        try {
            updateOrder(orderId);
            // 再次获取同一把锁（不会阻塞）
            lock.lock();
            try {
                updateOrderItems(orderId);
            } finally {
                lock.unlock();  // 释放第 2 层
            }
        } finally {
            lock.unlock();  // 释放第 1 层
        }
    }

    /**
     * 公平锁：按请求顺序获取锁（FIFO）
     * 适合对公平性有要求的场景（排队处理）
     */
    public void fairLockDemo(Long orderId) {
        RLock fairLock = redissonClient.getFairLock("fair:order:" + orderId);
        fairLock.lock();
        try {
            processOrder(orderId);
        } finally {
            fairLock.unlock();
        }
    }

    /**
     * 读写锁：读读不互斥，读写互斥
     * 适合缓存场景：读多写少
     */
    public void readWriteLockDemo(Long orderId) {
        RReadWriteLock rwLock = redissonClient.getReadWriteLock("cache:order:" + orderId);

        // 读操作
        RLock readLock = rwLock.readLock();
        readLock.lock();
        try {
            Order order = cacheManager.get(orderId);
            if (order == null) {
                // 没有数据，升级为写锁（需先释放读锁）
            }
        } finally {
            readLock.unlock();
        }

        // 写操作
        RLock writeLock = rwLock.writeLock();
        writeLock.lock();
        try {
            cacheManager.put(orderId, updatedOrder);
        } finally {
            writeLock.unlock();
        }
    }
}
```

## 5. 红锁（RedLock）— 跨 Redis 节点

```java
@Configuration
public class RedLockConfig {

    /**
     * RedLock 原理：
     * 在 N 个独立 Redis 节点（N 为奇数，通常 5）上同时获取锁
     * 超过 N/2 + 1 个节点成功才算加锁成功
     *
     * 解决：单个 Redis 节点宕机时锁的安全性问题
     * 注意：建议奇数个节点，如 3 或 5
     */
    @Bean
    public RedissonClient redissonRedLock() {
        Config config = new Config();
        config.useSentinelServers()
            .setMasterName("mymaster")
            .addSentinelAddress(
                "redis://node1:6379",
                "redis://node2:6379",
                "redis://node3:6379"
            )
            .setDatabase(0);
        return Redisson.create(config);
    }
}

@Service
public class RedLockService {

    @Autowired
    private RedissonClient redissonClient;

    /**
     * RedLock 使用场景：对锁安全性要求极高的场景
     * 如：金融交易、跨机房主备切换
     */
    public void transferMoney(Long from, Long to, BigDecimal amount) {
        // 构造红锁（需要多个独立资源）
        RLock lock1 = redissonClient.getLock("account:" + from);
        RLock lock2 = redissonClient.getLock("account:" + to);

        // 同时获取多个锁（红锁模式）
        RedissonRedLock redLock = new RedissonRedLock(lock1, lock2);

        try {
            // 获取锁（等待 5 秒，过期 30 秒）
            redLock.lock(30, TimeUnit.SECONDS);

            // 转账操作
            transfer(from, to, amount);

        } finally {
            redLock.unlock();
        }
    }
}
```

## 6. 基于注解的声明式锁（AOP + Redisson）

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DistributedLock {
    String key();               // SpEL 表达式：如 #orderId
    long waitTime() default 5;  // 等待锁时间（秒）
    long leaseTime() default 30;// 持有锁时间（秒），-1 启用看门狗
}

@Aspect
@Component
public class DistributedLockAspect {

    @Autowired
    private RedissonClient redissonClient;

    @Around("@annotation(lock)")
    public Object around(ProceedingJoinPoint joinPoint, DistributedLock lock) throws Throwable {
        // 解析 SpEL 表达式
        String lockKey = parseKey(lock.key(), joinPoint);
        RLock rLock = redissonClient.getLock(lockKey);

        boolean acquired;
        if (lock.leaseTime() == -1) {
            // 看门狗模式
            acquired = rLock.tryLock(lock.waitTime(), TimeUnit.SECONDS);
        } else {
            acquired = rLock.tryLock(lock.waitTime(), lock.leaseTime(), TimeUnit.SECONDS);
        }

        if (!acquired) {
            throw new BusinessException("获取锁失败: " + lockKey);
        }

        try {
            return joinPoint.proceed();
        } finally {
            if (rLock.isHeldByCurrentThread()) {
                rLock.unlock();
            }
        }
    }

    private String parseKey(String spEl, ProceedingJoinPoint joinPoint) {
        // 使用 Spring 的 ExpressionParser 解析 SpEL
        // 简化实现：直接调用 StandardEvaluationContext
        return "lock:" + spEl;
    }
}

// 使用示例
@Service
public class InventoryService {

    @DistributedLock(key = "'inventory:' + #productId")
    public boolean deductStock(Long productId, int quantity) {
        // 扣减库存（自动加锁）
        return stockMapper.decrement(productId, quantity);
    }
}
```

## 7. Spring Integration 分布式锁

```java
@Configuration
public class RedisLockConfiguration {

    /**
     * 利用 Spring Integration Redis LockRegistry 实现分布式锁
     * 无需引入 Redisson，适合轻量需求
     */
    @Bean
    public LockRegistry lockRegistry(RedisConnectionFactory connectionFactory) {
        return new RedisLockRegistry(connectionFactory, "spring-lock");
    }
}

@Service
public class SpringLockService {

    @Autowired
    private LockRegistry lockRegistry;

    public void processWithSpringLock(Long orderId) {
        Lock lock = lockRegistry.obtain("order:" + orderId);
        try {
            if (lock.tryLock(5, TimeUnit.SECONDS)) {
                try {
                    processOrder(orderId);
                } finally {
                    lock.unlock();
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

## 注意事项

### 选型对比

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| Redisson | watchdog、可重入、读写锁、功能全面 | 依赖较重、有 Redisson 版本兼容问题 | 大多数生产场景 |
| Spring Integration Lock | 轻量、无额外依赖 | 无看门狗、功能简单 | 简单分布式锁场景 |
| Lua + SETNX | 零依赖 | 需手写续期、易踩坑 | 学习/简单工具 |

### 常见坑点
1. **忘记释放锁**：务必用 `try-finally` 包裹 unlock，或用 Redisson 的 `lock()` 方法配合 try-with-resources
2. **看门狗失效**：指定了 `leaseTime` 后 watchdog 不会生效
3. **锁粒度过大**：锁的是业务 ID 而非笼统的 key（如 `order:123` 而非 `order:*`）
4. **主从切换锁丢失**：Redis 主从异步复制下，主节点宕机时锁可能丢失 — RedLock 可缓解
5. **tryLock 超时 ≠ 锁过期**：`tryLock(waitTime, leaseTime)` 的 waitTime 是获取锁的等待时间，不是业务执行时间

### 生产检查清单
- [ ] 所有 unlock 都在 finally 块中
- [ ] 锁 key 必须包含业务标识（如 `order:{id}` 而非 `order`）
- [ ] 设置了合理的锁过期时间（至少覆盖 95% 的业务执行时长）
- [ ] 启用了看门狗（如无特殊需要，不传 leaseTime）
- [ ] 使用了 Lua 脚本确保释放锁的原子性（Redisson 已内置）
- [ ] 单 Redis 实例或 RedLock 根据场景选择
