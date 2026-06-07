---
name: redis-client-side-caching
description: Redis RESP3 服务端辅助客户端缓存（Tracking）—— 零延迟缓存读取、自动失效与生产实践
tags: [redis, caching, performance, resp3, tracking, latency]
---

## 概述

Redis 6.0+ 引入的服务端辅助客户端缓存（Server-Assisted Client Caching，又称 Tracking）是**降低读取延迟的关键技术**。在 RESP3 协议下，客户端可以本地缓存热点数据，Redis 服务端在 key 发生变化时主动推送失效通知，实现 **近乎零网络开销的缓存读取**，同时保证缓存一致性。

**核心优势：**
- 读取延迟从 ~0.5ms（网络 RTT）降至 ~0.01ms（本地内存）
- 减少 Redis 服务端 QPS 压力（热点 key 无需反复查询）
- 自动失效，无需 TTL 或手动清理
- 与 Redis 8.x 完全兼容，性能更优

## 工作原理

```
┌──────────────┐          ┌──────────────┐
│   Redis 客户端 │          │  Redis 服务器 │
│  (本地缓存)    │          │              │
├──────────────┤          ├──────────────┤
│  1. GET key   │ ──────→  │  返回 value  │
│  2. 存入本地   │ ←────── │  + 注册跟踪   │
│  3. GET key   │ ──────→  │  (本地命中    │
│     (本地命中) │          │   不请求)     │
│  4. SET key X │ ──────→  │  更新数据     │
│  5. ←─────────│ ←────── │  推送 INVALID │
│  6. 清除本地   │          │  清除跟踪     │
└──────────────┘          └──────────────┘
```

### 两种模式

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| **默认模式** | 服务端记录每个客户端跟踪的 key，修改时推送失效 | 通用场景，但有内存开销 |
| **广播模式**（Redis 8.0+ 优化） | 客户端订阅特定前缀的失效通知，服务端无需维护跟踪表 | 大量客户端、大 key 空间场景 |

## 客户端配置

### Lettuce 6.x+（推荐）

```java
@Configuration
public class RedisClientSideCacheConfig {

    @Bean
    public RedisClient redisClient() {
        // 启用 RESP3 协议（Lettuce 6.0+ 默认协商 RESP3）
        RedisURI uri = RedisURI.builder()
            .withHost("localhost")
            .withPort(6379)
            .build();

        RedisClient client = RedisClient.create(uri);
        // 开启客户端缓存
        client.setOptions(ClientOptions.builder()
            .clientCacheOptions(ClientCacheOptions.builder()
                .enabled(true)                    // 启用客户端缓存
                .maxSize(10_000)                  // 本地缓存上限
                .evictionPolicy(EvictionPolicy.LRU) // 淘汰策略
                .build())
            .build());
        return client;
    }

    @Bean
    public StatefulRedisConnection<String, String> connection(RedisClient client) {
        // 需要同步连接使用
        return client.connect();
    }

    @Bean
    public RedisCommands<String, String> syncCommands(
            StatefulRedisConnection<String, String> connection) {
        return connection.sync();
    }
}
```

### 使用示例

```java
@Service
public class CachedProductService {

    @Autowired
    private RedisCommands<String, String> redis;

    public Product getProduct(String id) {
        // 第一次：走网络请求
        // 后续：本地缓存命中（零网络）直到 key 被修改
        String json = redis.get("product:" + id);

        // redis.get() 会自动注册跟踪
        // 如果其他客户端修改了 "product:" + id，服务端推送失效
        // 下次 get 将重新从网络获取并注册新跟踪

        return json != null ? JSON.parseObject(json, Product.class) : null;
    }
}
```

### Spring Data Redis (Lettuce) 集成

```yaml
# application.yml
spring:
  data:
    redis:
      client-name: my-app
      lettuce:
        client-cache:
          enabled: true
          max-size: 10000
          eviction: LRU
```

```java
@Configuration
public class LettuceClientCacheConfig {

    @Bean
    public LettuceClientResources lettuceClientResources() {
        return LettuceClientResources.builder()
            .clientCacheOptions(ClientCacheOptions.builder()
                .enabled(true)
                .maxSize(5000)
                .evictionPolicy(EvictionPolicy.LFU) // LFU 更适合热点数据
                .build())
            .build();
    }

    @Bean
    public RedisConnectionFactory redisConnectionFactory(
            LettuceClientResources clientResources) {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration("localhost", 6379);
        LettuceConnectionFactory factory = new LettuceConnectionFactory(config);
        factory.setClientResources(clientResources);
        return factory;
    }
}
```

### Jedis 5.x+

```java
// Jedis 5.x 支持 RESP3 和客户端缓存
HostAndPort host = new HostAndPort("localhost", 6379);
JedisClientConfig config = JedisClientConfig.builder()
    .resp3()                       // 启用 RESP3
    .clientCache(new ClientCache() // 开启客户端缓存
        .maxEntries(10000))
    .build();

try (Jedis jedis = new Jedis(host, config)) {
    // 第一次走网络
    String val = jedis.get("mykey");
    // 后续访问本地缓存
    val = jedis.get("mykey"); // 零网络
}
```

## 广播模式（Subscription Mode）

适用于大量客户端共享相同热点 key 集合的场景，服务端无需跟踪每个客户端。

```java
// 客户端订阅特定 key 前缀的失效通知
// 使用 broadcast 模式
StatefulRedisConnection<String, String> conn = client.connect();
conn.setOptions(ClientOptions.builder()
    .clientCacheOptions(ClientCacheOptions.builder()
        .enabled(true)
        .broadcast(true)  // 广播模式
        .prefixes("product:", "user:") // 仅跟踪这些前缀
        .build())
    .build());
```

## 监控客户端缓存命中率

```bash
# Redis 服务端查看跟踪信息
redis-cli CLIENT TRACKING INFO
# 输出: flags=on, redirect=0, prefixes=2

# 客户端缓存统计
redis-cli INFO stats
# tracking_total_keys: 3421         # 已跟踪的 key 数
# tracking_total_invalidations: 156 # 累计失效推送次数

# 客户端侧可通过 Lettuce Metrics 监控
```

## 注意事项

### 缓存容量控制
- **maxSize 必须设置**，防止内存泄漏
- 根据 JVM 堆大小合理设置：通常 5,000~50,000 条
- 使用 LRU/LFU 淘汰策略，避免 OOM

### 不适用场景
- **写密集场景**：key 频繁变更会导致大量失效通知，性能反而下降
- **超大 value**：本地缓存堆外或序列化存储，大 value 占用过多内存
- **一次性 key**：用完即弃的 key 不具备缓存价值

### 连接池注意事项
```java
// ❌ 错误：连接池中的连接重建后丢失跟踪关系
// 每次 redisTemplate 操作可能使用不同连接

// ✅ 正确：使用专用连接进行跟踪
@Bean
public StatefulRedisConnection<String, String> trackingConnection(
        RedisClient client) {
    StatefulRedisConnection<String, String> conn = client.connect();
    conn.setAutoFlush(true);
    return conn;
}
```

### Redis 8.x 增强
- Redis 8.0 优化了广播模式的内部数据结构，减少内存开销
- Redis 8.6+ 修复了跟踪表在集群模式下的边缘崩溃问题
- Redis 8.8 的 Hash 字段级通知可与客户端缓存配合使用

### 安全建议
- 生产环境建议配合 TLS 使用，防止缓存内容泄露
- 敏感数据（密码、令牌）不应进入客户端缓存
- 通过 `CLIENT NO-EVICT` 控制跟踪行为
