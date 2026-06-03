---
name: redis-local-cached-map
description: Redisson RLocalCachedMap 本地缓存（Near Cache）深度实践：缓存策略、一致性保证、Spring Boot 集成与性能优化
tags: [redis, redisson, near-cache, local-cache, rlocalcachedmap, cache-strategy]
---

## 概述

**RLocalCachedMap** 是 Redisson 提供的分布式 Map 变体，在每个客户端 JVM 中维护一份本地缓存副本。读操作优先从本地缓存获取（零网络开销），Redis 作为分布式一致性与变更通知中枢。

相比"Redis + Caffeine 手动双写"方案，RLocalCachedMap 的优势在于：
- **自动失效通知**：任一客户端写入 → Redis 发布变更 → 其他客户端自动失效本地条目
- **零代码同步**：不需要自己实现消息队列或 Redis Pub/Sub 手动清理
- **多种淘汰策略**：LRU、LFU、SOFT、WEAK 等 JVM 缓存策略
- **事务一致性**：支持在 Redisson 事务中操作

## 适用场景

| 场景 | 推荐度 | 原因 |
|---|---|---|
| **配置/字典数据** | ⭐⭐⭐⭐⭐ | 读多写极少，对一致性要求不苛刻 |
| **用户 Session 缓存** | ⭐⭐⭐⭐ | 每个节点只读自己的用户数据，跨节点失效可接受 |
| **热点商品/文章详情** | ⭐⭐⭐⭐ | Hot Key QPS 高，本地缓存大幅降低 Redis 压力 |
| **全局计数器/秒杀库存** | ❌ | 写频繁，本地缓存无意义，直接用 Redis 原子操作 |
| **金融交易数据** | ⚠️ | 最终一致性，不能接受脏读则需配合 Redis 直读兜底 |

## Maven 依赖

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>4.4.0</version>
</dependency>
```

## Spring Boot 配置

```yaml
spring:
  data:
    redis:
      host: 192.168.1.100
      port: 6379
      password: ${REDIS_PASSWORD}
      timeout: 3000ms
      lettuce:
        pool:
          max-active: 32
          max-idle: 16
          min-idle: 8
```

```java
@Configuration
public class RedissonConfig {

    @Value("${spring.data.redis.host}")
    private String redisHost;

    @Value("${spring.data.redis.port}")
    private int redisPort;

    @Value("${spring.data.redis.password:}")
    private String password;

    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        SingleServerConfig serverConfig = config.useSingleServer()
            .setAddress("redis://" + redisHost + ":" + redisPort);

        if (StringUtils.hasText(password)) {
            serverConfig.setPassword(password);
        }

        // 连接池配置
        serverConfig.setConnectionPoolSize(32);
        serverConfig.setConnectionMinimumIdleSize(8);
        serverConfig.setDnsMonitoringInterval(5000);

        return Redisson.create(config);
    }

    @Bean
    public RLocalCachedMap<String, Object> configCache(RedissonClient redisson) {
        // 本地缓存配置：LFU 淘汰，最大 1000 条目，存活 30 分钟
        LocalCachedMapOptions<String, Object> options = LocalCachedMapOptions
            .<String, Object>defaults()
            .cacheSize(1000)                    // 本地缓存大小
            .evictionPolicy(LocalCachedMapOptions.EvictionPolicy.LFU)  // 淘汰策略
            .timeToLive(30, TimeUnit.MINUTES)   // 本地缓存 TTL
            .syncStrategy(LocalCachedMapOptions.SyncStrategy.UPDATE)   // 同步策略
            .reconnectionStrategy(LocalCachedMapOptions.ReconnectionStrategy.LOAD);  // 重连策略

        return redisson.getLocalCachedMap("config:cache", options);
    }
}
```

## 五种本地缓存策略

### 1. LRU（最近最少使用）

```java
LocalCachedMapOptions<String, Object> options = LocalCachedMapOptions
    .<String, Object>defaults()
    .cacheSize(500)
    .evictionPolicy(LocalCachedMapOptions.EvictionPolicy.LRU);
```

最近最少访问的条目被淘汰。适用于**访问模式有局部性**的场景。

### 2. LFU（最不经常使用）

```java
LocalCachedMapOptions<String, Object> options = LocalCachedMapOptions
    .<String, Object>defaults()
    .cacheSize(1000)
    .evictionPolicy(LocalCachedMapOptions.EvictionPolicy.LFU);
```

访问频率最低的条目被淘汰。适用于**热点数据分布不均匀**的场景（配置中心、字典表）。

### 3. SOFT（软引用）

```java
LocalCachedMapOptions<String, Object> options = LocalCachedMapOptions
    .<String, Object>defaults()
    .cacheSize(0)  // 0 = 无上限
    .evictionPolicy(LocalCachedMapOptions.EvictionPolicy.SOFT);
```

JVM GC 内存不足时回收。适用于**内存充裕但愿意在压力下降级**的场景。

### 4. WEAK（弱引用）

```java
LocalCachedMapOptions<String, Object> options = LocalCachedMapOptions
    .<String, Object>defaults()
    .cacheSize(0)
    .evictionPolicy(LocalCachedMapOptions.EvictionPolicy.WEAK);
```

GC 扫描到即回收，不等到内存不足。适用于**临时缓存且敏感于内存泄漏**的场景。

### 5. NONE（无淘汰）

```java
LocalCachedMapOptions<String, Object> options = LocalCachedMapOptions
    .<String, Object>defaults()
    .cacheSize(0)
    .evictionPolicy(LocalCachedMapOptions.EvictionPolicy.NONE);
```

禁用本地缓存。**等同于普通 RMap**，一般不使用。

## 同步策略（SyncStrategy）

| 策略 | 含义 | 使用场景 |
|---|---|---|
| `UPDATE` | 本地被修改/写入后，立即通知其他节点失效缓存 | 写较少的配置类场景（默认推荐） |
| `INVALIDATE` | 本地修改后，仅通知其他节点失效（本地也不保留） | 写频繁但读不能读到旧值 |
| `NONE` | 不发送任何失效通知 | 纯本地缓存、无需跨节点一致性 |

## 重连策略（ReconnectionStrategy）

| 策略 | 含义 |
|---|---|
| `LOAD` | 重连后从 Redis 重新加载所有本地缓存（推荐） |
| `CLEAR` | 重连后清空本地缓存（后续按需加载更安全但首次性能差） |

## 完整实战：配置中心缓存

```java
@Service
public class ConfigCacheService {

    private final RLocalCachedMap<String, Object> configCache;

    public ConfigCacheService(RedissonClient redissonClient) {
        LocalCachedMapOptions<String, Object> options = LocalCachedMapOptions
            .<String, Object>defaults()
            .cacheSize(2000)
            .evictionPolicy(LocalCachedMapOptions.EvictionPolicy.LFU)
            .timeToLive(1, TimeUnit.HOURS)
            .syncStrategy(SyncStrategy.UPDATE)
            .reconnectionStrategy(ReconnectionStrategy.LOAD);
        this.configCache = redissonClient.getLocalCachedMap("app:config", options);
    }

    /**
     * 获取配置（优先本地缓存）
     */
    @SuppressWarnings("unchecked")
    public <T> T getConfig(String key, Class<T> type) {
        Object value = configCache.get(key);
        if (value == null) {
            // 兜底：从数据库加载
            value = loadFromDb(key);
            if (value != null) {
                configCache.put(key, value);
            }
        }
        return (T) value;
    }

    /**
     * 更新配置（自动通知其他节点失效本地缓存）
     */
    public void updateConfig(String key, Object value) {
        saveToDb(key, value);          // 先持久化
        configCache.put(key, value);   // 更新本地 + Redis + 广播失效通知
    }

    /**
     * 批量预热
     */
    @PostConstruct
    public void warmUp() {
        Map<String, Object> allConfigs = loadAllFromDb();
        configCache.putAll(allConfigs);
        log.info("Config cache warmed up with {} entries", allConfigs.size());
    }

    @EventListener(ApplicationReadyEvent.class)
    public void onReady() {
        log.info("RLocalCachedMap store: {}", configCache.getStore()); // 检查存储类型
    }
}
```

## 与 Caffeine 手动实现方案的对比

| 维度 | RLocalCachedMap | Caffeine + Redis 手动 |
|---|---|---|
| **代码量** | 5 行配置 | 50+ 行（缓存管理 + 失效监听） |
| **自动失效** | ✅ Redis Pub/Sub 原生支持 | ❌ 需要自建 MQ 或 Redis Pub/Sub |
| **淘汰策略** | LRU/LFU/SOFT/WEAK/NONE | 丰富（Weighted, Timed, Reference） |
| **序列化控制** | Redisson 统一管理 | 需自己配置 |
| **原子操作** | fastPut/fastRemove 等 | 需自己加锁 |
| **Spring Cache 集成** | 需额外适配 | 可直接用 @Cacheable |

## 注意事项

1. **内存泄漏风险**：`cacheSize=0` 且 `EvictionPolicy.NONE` 时，本地缓存无限增长，生产务必设置上限
2. **大对象序列化**：RLocalCachedMap 存储大对象会占用双倍内存（Redis + JVM），建议只缓存精简后的 DTO
3. **一致性窗口期**：`SyncStrategy.UPDATE` 模式下，从写入到通知其他节点存在短暂窗口（毫秒级），严格一致场景用 `INVALIDATE` + 读时回源
4. **网络分区**：断连期间本地缓存还是旧值，重连后 `reconnectionStrategy=LOAD` 可恢复一致
5. **Hot Key 瓶颈不在 Redis 而在序列化**：JSON 序列化/反序列化是 CPU 主要开销，改用 FST/Kryo 或 protocol buffer 可提升 3-5 倍
6. **不支持 RMap 的部分操作**：如 `putIfAbsent`、`replace` 等原子 CAS 方法在本地缓存模式下行为与 RMap 不同，测试覆盖
7. **调试**：开启 Redisson 日志级别 DEBUG 可观察缓存命中/失效事件
