---
name: redis-keyspace-notifications
description: Redis 键空间通知实战：过期事件监听、延迟任务、缓存失效通知、分布式调度与 Spring Boot 集成
tags: [redis, keyspace, notification, pub-sub, event, delayed-task, spring-boot]
---

## 概述

Redis 键空间通知（Keyspace Notifications）允许客户端**订阅 key 的事件**（设置、过期、删除、修改等），实现事件驱动的编程模式。常见应用场景：

- **延迟任务**：监听 key 过期事件实现毫秒级延迟队列
- **缓存失效通知**：收到缓存过期通知后刷新或预热
- **分布式调度**：基于 key 事件协调多个节点
- **数据变更追踪**：监听特定 key 的修改

## 1. 配置与启用

### 启用通知（默认关闭）

```bash
# 方式一：运行时启用（重启失效）
CONFIG SET notify-keyspace-events KEA

# 方式二：持久化配置（redis.conf）
notify-keyspace-events "KEA"
```

### 事件类型标志位

| 标志 | 含义 | 说明 |
|------|------|------|
| `K` | Keyspace events | `__keyspace@<db>__:<key>` 格式 |
| `E` | Keyevent events | `__keyevent@<db>__:<op>` 格式 |
| `g` | 通用命令 | DEL, EXPIRE, RENAME 等 |
| `$` | 字符串命令 | SET, GETSET 等 |
| `l` | 列表命令 | LPUSH, RPOP 等 |
| `s` | 集合命令 | SADD, SREM 等 |
| `h` | 哈希命令 | HSET, HDEL 等 |
| `z` | 有序集合命令 | ZADD, ZREM 等 |
| `x` | 过期事件 | key 过期时触发（独立通道） |
| `e` | 驱逐事件 | key 因 maxmemory 被驱逐 |
| `A` | 等同于 `g$lshzxe` | 所有事件别名 |

### 常见配置组合

```bash
# 仅监听过期事件（最常用）
notify-keyspace-events "Ex"

# 监听所有修改事件
notify-keyspace-events "K$lshzxe"

# 监听所有事件（生产环境慎用，性能开销大）
notify-keyspace-events "KEA"
```

## 2. 订阅与消费

### 命令行测试

```bash
# 终端 1：订阅所有过期事件
PSUBSCRIBE __keyevent@0__:expired

# 终端 2：设置一个 5 秒过期的 key
SETEX order:123 5 "paid"

# 5 秒后终端 1 收到：
# 1) "pmessage"
# 2) "__keyevent@0__:expired"
# 3) "__keyevent@0__:expired"
# 4) "order:123"
```

### 订阅特定 key

```bash
# 监听特定 key 的所有事件
SUBSCRIBE __keyspace@0__:order:123

# key 被修改时收到通知：
# 1) "message"
# 2) "__keyspace@0__:order:123"
# 3) "set"          ← 操作类型
```

## 3. 延迟任务实现

利用 key 过期事件实现精确到秒级的延迟队列：

```java
@Component
public class DelayTaskListener {

    private static final Logger log = LoggerFactory.getLogger(DelayTaskListener.class);

    private final StringRedisTemplate redisTemplate;
    private final TaskHandler taskHandler;

    // 使用独立连接（避免阻塞业务连接）
    private final RedisMessageListenerContainer container;

    public DelayTaskListener(StringRedisTemplate redisTemplate,
                             TaskHandler taskHandler,
                             RedisMessageListenerContainer container) {
        this.redisTemplate = redisTemplate;
        this.taskHandler = taskHandler;
        this.container = container;
    }

    @PostConstruct
    public void init() {
        // 订阅 DB 0 的所有过期事件
        container.addMessageListener(
            (message, pattern) -> {
                String expiredKey = new String(message.getBody());
                handleExpiredKey(expiredKey);
            },
            new PatternTopic("__keyevent@0__:expired")
        );
    }

    private void handleExpiredKey(String key) {
        // key 格式：delay:order:{orderId}
        if (key.startsWith("delay:order:")) {
            String orderId = key.substring("delay:order:".length());
            log.info("订单超时未支付，执行取消: {}", orderId);
            taskHandler.cancelOrder(orderId);
        } else if (key.startsWith("delay:notification:")) {
            // 延迟通知
            String userId = key.substring("delay:notification:".length());
            taskHandler.sendDelayedNotification(userId);
        }
    }
}

// 发送延迟任务
@Service
public class DelayTaskProducer {

    private final StringRedisTemplate redisTemplate;

    // 创建延迟任务
    public void createDelayOrder(String orderId, Duration delay) {
        String key = "delay:order:" + orderId;
        // 设置 value + 过期时间（过期时触发通知）
        redisTemplate.opsForValue().set(key, "pending", delay);
    }

    // 取消延迟任务
    public void cancelDelayOrder(String orderId) {
        redisTemplate.delete("delay:order:" + orderId);
        log.info("订单已支付，取消超时任务: {}", orderId);
    }
}
```

## 4. Spring Boot 集成

### 配置

```yaml
spring:
  redis:
    host: localhost
    port: 6379
    # 启用 keyspace 通知（需 Redis 端配置）
```

### 监听配置

```java
@Configuration
public class RedisKeyspaceConfig {

    @Bean
    RedisMessageListenerContainer redisMessageListenerContainer(
            RedisConnectionFactory connectionFactory) {
        RedisMessageListenerContainer container =
            new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        // 设置线程池大小
        container.setTaskExecutor(
            Executors.newFixedThreadPool(4,
                new ThreadFactoryBuilder()
                    .setNameFormat("redis-listener-%d")
                    .build()));
        return container;
    }
}
```

### @EventListener 注解方式

```java
@Component
public class CacheEvictionListener {

    private final CacheManager cacheManager;

    @Bean
    public MessageListenerAdapter cacheEvictionAdapter() {
        return new MessageListenerAdapter(this, "handleCacheEviction");
    }

    // 必须与 MessageListenerAdapter 中指定的方法名一致
    public void handleCacheEviction(String message) {
        log.info("缓存过期: {}", message);
        // 可在此处执行缓存预热
        if (message.startsWith("cache:hot:")) {
            String key = message.substring("cache:hot:".length());
            refreshHotCache(key);
        }
    }

    private void refreshHotCache(String key) {
        // 从数据库重新加载并设置缓存
        // ...
    }

    @Bean
    public RedisMessageListenerContainer container(
            RedisConnectionFactory factory,
            MessageListenerAdapter adapter) {
        RedisMessageListenerContainer container =
            new RedisMessageListenerContainer();
        container.setConnectionFactory(factory);
        container.addMessageListener(
            adapter,
            new PatternTopic("__keyevent@0__:expired")
        );
        return container;
    }
}
```

## 5. 缓存一致性实现

使用 keyspace 通知实现**缓存一致性**：一个节点更新缓存后，通知其他节点失效本地缓存。

```java
@Component
public class CacheInvalidationCoordinator {

    private final RedisTemplate<String, Object> redisTemplate;
    private final Map<String, Object> localCache = new ConcurrentHashMap<>();

    // 监听所有 key 的 set 操作
    @PostConstruct
    public void init() {
        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        // ... 配置容器

        container.addMessageListener(
            (message, pattern) -> {
                String channel = new String(message.getChannel());
                String key = new String(message.getBody());
                // channel: __keyspace@0__:config:system
                // body: set ← 操作类型
                if (channel.contains(":config:")) {
                    log.info("配置发生变化，清除本地缓存: {}", channel);
                    localCache.remove(key);
                }
            },
            new PatternTopic("__keyspace@0__:config:*")
        );
    }

    // 更新配置通知所有节点
    public void updateConfig(String key, Object value) {
        // 1. 更新 Redis
        redisTemplate.opsForValue().set(key, value);
        // 2. 其他节点通过 keyspace 通知自动清除本地缓存
    }
}
```

## 6. 分布式锁自动续期

监听锁 key 的过期来自动续期：

```java
@Service
public class LockAutoRenewService {

    private final StringRedisTemplate redisTemplate;
    private final RedisMessageListenerContainer container;

    private final Map<String, ScheduledFuture<?>> renewTasks =
        new ConcurrentHashMap<>();

    public void acquireWithAutoRenew(String lockKey,
                                     String lockValue,
                                     Duration ttl) {
        Boolean locked = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, lockValue, ttl);

        if (Boolean.TRUE.equals(locked)) {
            // 启动自动续期线程
            startRenewTask(lockKey, lockValue, ttl);
        }
    }

    private void startRenewTask(String lockKey,
                                String value,
                                Duration ttl) {
        ScheduledFuture<?> task = Executors
            .newSingleThreadScheduledExecutor()
            .scheduleAtFixedRate(() -> {
                // 每 1/3 TTL 续期一次
                redisTemplate.expire(lockKey, ttl);
            }, ttl.toMillis() / 3, ttl.toMillis() / 3,
               TimeUnit.MILLISECONDS);

        renewTasks.put(lockKey, task);

        // 监听锁 key 的 del 事件（其它节点释放锁时停止续期）
        container.addMessageListener(
            (message, pattern) -> {
                String channel = new String(message.getChannel());
                String operation = new String(message.getBody());
                String key = channel.substring(
                    channel.lastIndexOf(':') + 1);

                if ("del".equals(operation)
                        && renewTasks.containsKey(key)) {
                    log.info("锁被释放，停止续期: {}", key);
                    cancelRenew(key);
                }
            },
            new ChannelTopic("__keyspace@0__:" + lockKey)
        );
    }

    private void cancelRenew(String lockKey) {
        ScheduledFuture<?> task = renewTasks.remove(lockKey);
        if (task != null) {
            task.cancel(false);
        }
    }
}
```

## 7. Redis 8.8 字段级通知

Redis 8.8 增强了对 Hash 字段级别的变更通知：

```bash
# 订阅 hash 字段级别的变更
PSUBSCRIBE __keyevent@0__:hset:*   # 任意 hash 字段设置
PSUBSCRIBE __keyevent@0__:hdel:*   # 任意 hash 字段删除
```

```java
// Redis 8.8+ 监听 Hash 字段级变更
container.addMessageListener(
    (message, pattern) -> {
        String channel = new String(message.getChannel());
        String operation = new String(message.getBody());
        // channel: __keyevent@0__:hset:user:1001
        // operation: hset:name  ← 修改了 name 字段
        log.info("Hash 字段变更: {} => {}", channel, operation);
    },
    new PatternTopic("__keyevent@0__:hset:*")
);
```

## 8. 生产注意事项

### 性能影响

| 事件量 | 影响 | 建议 |
|-------|------|------|
| < 100/s | 可忽略 | 直接使用 |
| 1000/s | 轻微 CPU 增加 | 独立 Redis 实例监听 |
| 10000/s+ | 可能影响主线程 | 使用专门 Sentinel/Slave 实例 |

### 安全与稳定性

1. **订阅端故障**：如果订阅客户端断开连接，会丢失中间的通知。重要场景需结合轮询补偿
2. **Redis 主线程**：事件在 key 操作同线程发布，大量通知可能影响 Redis 性能
3. **单播特性**：keyspace 通知是"发后即忘"，不保证投递可靠性
4. **连接隔离**：监听通知使用独立的 Redis 连接，避免阻塞业务操作
5. **至少一次语义**：通知不保证"至少一次/恰好一次"语义，关键业务不能仅依赖通知

### 补偿机制

```java
@Component
public class CompensatingChecker {

    private final StringRedisTemplate redisTemplate;

    // 定期扫描补偿未处理的任务
    @Scheduled(fixedDelay = 60000)  // 每分钟
    public void compensate() {
        // 扫描所有延迟任务 key（带特定前缀）
        Set<String> pendingKeys = redisTemplate.keys("delay:order:*");

        for (String key : pendingKeys) {
            Long ttl = redisTemplate.getExpire(key);
            if (ttl != null && ttl <= 0) {
                // key 过期但可能错过了通知
                log.warn("补偿处理未消费的延迟任务: {}", key);
                handleDelayedKey(key);
            }
        }
    }

    private void handleDelayedKey(String key) {
        // 从 key 提取业务信息
        String orderId = key.replace("delay:order:", "");
        log.info("补偿取消订单: {}", orderId);
    }
}
```

## 注意事项

1. **notify-keyspace-events 必须开启**：默认关闭，所有发布订阅都不生效
2. **事件丢失可能性**：订阅客户端断开后重新连接，中间事件无法补发
3. **不同 DB 隔离**：事件按 DB 分隔，订阅 `__keyevent@0__:expired` 只收 DB 0 的事件
4. **Redis 8.8+ 字段级通知**：需要 Redis 版本 ≥ 8.8
5. **性能监控**：通过 `INFO keyspace` 查看当前订阅数量
6. **集群模式**：Redis Cluster 中，keyspace 通知仅在 key 所在的节点发布，订阅需连接所有节点

## 参考链接

- [Redis Keyspace Notifications 官方文档](https://redis.io/docs/manual/keyspace-notifications/)
- [Redis 8.8 字段级 Hash 通知](https://redis.io/docs/about/releases/8.8/)
- [Spring Data Redis - Message Listeners](https://docs.spring.io/spring-data/redis/reference/redis/pubsub.html)
