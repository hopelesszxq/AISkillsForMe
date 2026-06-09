---
name: redis-cache-invalidation-pubsub
description: Redis Pub/Sub 分布式缓存失效广播 —— 多实例环境下缓存一致性的订阅发布模式
tags: [redis, cache, pub-sub, invalidation, distributed, spring-boot]
---

## 概述

在分布式多实例部署下，每个实例连接同一个 Redis，但缓存数据各自独立（本地缓存）或共享 Redis 缓存。当某个实例更新/删除数据时，其他实例不会自动感知缓存变化。通过 Redis Pub/Sub 广播失效事件，可以在所有实例间同步缓存状态，保证最终一致性。

## 架构

```
Instance A                    Redis                     Instance B
    │                           │                           │
    ├── Update Data ──────────► │                           │
    │                           │                           │
    ├── Publish: "cache:inv" ──►│ ──► SUBSCRIBE "cache:inv"─┤
    │   {key: "product:1"}     │     └── Evict local cache  │
    │                           │                           │
    │ ◄──── UNSUBSCRIBE ────────┤                           │
```

## 核心实现

### 1. Redis 配置（JSON 序列化 + Pub/Sub）

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisMessageListenerContainer redisMessageListenerContainer(
            RedisConnectionFactory connectionFactory,
            CacheInvalidationListener listener) {

        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);

        // 订阅失效通道
        container.addMessageListener(listener,
            new ChannelTopic("cache:invalidation"));
        return container;
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());

        // JSON 序列化 —— 可读性好，跨版本兼容
        GenericJackson2JsonRedisSerializer serializer =
            new GenericJackson2JsonRedisSerializer();
        template.setValueSerializer(serializer);
        template.setHashValueSerializer(serializer);
        return template;
    }
}
```

### 2. 失效消息发布器

```java
@Component
public class CacheInvalidationPublisher {

    private final RedisTemplate<String, Object> redisTemplate;
    private static final String CHANNEL = "cache:invalidation";

    public void publish(String namespace, String key) {
        Map<String, String> message = new HashMap<>();
        message.put("namespace", namespace);
        message.put("key", key);
        message.put("timestamp", Instant.now().toString());
        redisTemplate.convertAndSend(CHANNEL, message);
    }

    public void publishNamespace(String namespace) {
        Map<String, String> message = new HashMap<>();
        message.put("namespace", namespace);
        message.put("key", "*");
        message.put("action", "flush");
        redisTemplate.convertAndSend(CHANNEL, message);
    }
}
```

### 3. 失效消息监听器

```java
@Component
public class CacheInvalidationListener implements MessageListener {

    private static final Logger log = LoggerFactory.getLogger(CacheInvalidationListener.class);
    private final StringRedisTemplate stringRedisTemplate;

    public CacheInvalidationListener(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    @Override
    public void onMessage(Message message, byte[] pattern) {
        try {
            String body = new String(message.getBody());
            // 简化解析，生产环境建议用 JSON 库
            Map<String, String> data = parseSimpleJson(body);
            String namespace = data.get("namespace");
            String key = data.get("key");
            String action = data.getOrDefault("action", "evict");

            if ("flush".equals(action)) {
                // 清空整个命名空间
                Set<String> keys = stringRedisTemplate.keys(namespace + ":*");
                if (keys != null && !keys.isEmpty()) {
                    stringRedisTemplate.delete(keys);
                    log.info("Flushed namespace: {} ({} keys)", namespace, keys.size());
                }
            } else {
                String redisKey = namespace + ":" + key;
                stringRedisTemplate.delete(redisKey);
                log.info("Evicted cache key: {} (from pub/sub)", redisKey);
            }
        } catch (Exception e) {
            log.error("Failed to process cache invalidation message", e);
        }
    }

    private Map<String, String> parseSimpleJson(String json) {
        // 生产环境建议用 Jackson / Gson
        Map<String, String> result = new HashMap<>();
        json = json.replaceAll("[{}\"]", "");
        for (String pair : json.split(",")) {
            String[] kv = pair.split(":", 2);
            if (kv.length == 2) result.put(kv[0].trim(), kv[1].trim());
        }
        return result;
    }
}
```

### 4. 缓存服务（带失效通知）

```java
@Service
public class CacheService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final CacheInvalidationPublisher publisher;

    // 写入缓存（带 TTL）
    public void put(String namespace, String key, Object value, long ttlSeconds) {
        String redisKey = namespace + ":" + key;
        redisTemplate.opsForValue().set(redisKey, value,
            Duration.ofSeconds(ttlSeconds));
    }

    // 读取缓存（Cache-Aside）
    @SuppressWarnings("unchecked")
    public <T> T get(String namespace, String key, Class<T> type) {
        String redisKey = namespace + ":" + key;
        Object value = redisTemplate.opsForValue().get(redisKey);
        return value != null ? (T) value : null;
    }

    // 失效缓存（发布广播）
    public void evict(String namespace, String key) {
        String redisKey = namespace + ":" + key;
        redisTemplate.delete(redisKey);
        publisher.publish(namespace, key);  // 通知其他实例
    }

    // 失效整个命名空间
    public void evictNamespace(String namespace) {
        Set<String> keys = redisTemplate.keys(namespace + ":*");
        if (keys != null && !keys.isEmpty()) {
            redisTemplate.delete(keys);
        }
        publisher.publishNamespace(namespace);
    }
}
```

### 5. 写操作集成

```java
@Service
public class ProductService {

    private final ProductRepository productRepository;
    private final CacheService cacheService;
    private static final String NAMESPACE = "products";
    private static final long TTL = 3600;  // 1 hour

    @Transactional
    public Product update(Long id, ProductUpdateRequest request) {
        // 1. 先写数据库（source of truth）
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException(id));
        product.setName(request.getName());
        product.setPrice(request.getPrice());
        productRepository.save(product);

        // 2. 更新缓存
        cacheService.put(NAMESPACE, String.valueOf(id), product, TTL);

        return product;
    }

    @Transactional
    public void delete(Long id) {
        // 1. 删除数据库
        productRepository.deleteById(id);

        // 2. 失效缓存（触发广播通知其他实例）
        cacheService.evict(NAMESPACE, String.valueOf(id));
    }
}
```

## 三种失效策略对比

| 策略 | 实现 | 场景 | 注意事项 |
|------|------|------|---------|
| **TTL 过期** | Redis 原生 EXPIRE | 读多写少、容忍短时不一致 | 过期时间内读到的可能是旧数据 |
| **手动驱逐** | DEL 当前实例的 key | 写操作后立即失效 | 只影响当前实例，多实例需配合广播 |
| **Pub/Sub 广播** | `convertAndSend` + `MessageListener` | 多实例需要同步失效 | 依赖 Redis 连接，网络抖动可能导致消息丢失 |

## 注意事项

### 1. 消息可靠性

Redis Pub/Sub 是"发后即忘"模型，不保证消息可靠送达：
- 订阅者离线会丢失消息
- Redis 重启后订阅者需要重新订阅
- **解决方案**：配合 TTL 兜底，或在写操作同时更新共享缓存

### 2. 循环通知

当监听器也通过同一 `RedisTemplate` 操作 Redis 时，可能触发循环：
```yaml
# 建议使用单独的 StringRedisTemplate 给监听器
# 与业务操作的 template 隔离开
```

### 3. 序列化一致

```yaml
# 发布者和订阅者必须使用相同的序列化配置
# 推荐使用 GenericJackson2JsonRedisSerializer
# 避免使用 JdkSerializationRedisSerializer（跨版本耦合）
```

### 4. 生产级优化

```java
// 异步发布，避免阻塞主流程
@Async
public void publishAsync(String namespace, String key) {
    publisher.publish(namespace, key);
}

// 批量失效合并
public void evictBatch(String namespace, List<String> keys) {
    List<String> redisKeys = keys.stream()
        .map(k -> namespace + ":" + k)
        .collect(Collectors.toList());
    redisTemplate.delete(redisKeys);
    publisher.publish(namespace, String.join(",", keys));
}
```

### 5. 替代方案

- **Redis Keyspace Notifications**：监听 key 过期事件，但对主动 DEL 不友好
- **Redisson 分布式缓存**：内置了缓存失效机制
- **Canal + MQ**：订阅 MySQL binlog 做缓存同步，解耦更强但架构更重
