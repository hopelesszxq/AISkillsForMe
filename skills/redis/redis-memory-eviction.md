---
name: redis-memory-eviction
description: Redis 内存管理、淘汰策略、TTL 设计与内存优化实战
tags: [redis, memory, eviction, ttl, maxmemory, optimization]
---

## 内存淘汰策略

### 八种淘汰策略对比

| 策略 | 描述 | 适用场景 |
|------|------|---------|
| `noeviction` | 内存满时写入报错 | 数据不可丢失（队列、分布式锁） |
| `allkeys-lru` | 所有 key 按 LRU 淘汰 | **通用缓存（推荐）** |
| `allkeys-lfu` | 所有 key 按 LFU 淘汰 | 热点数据差异大的缓存 |
| `allkeys-random` | 随机淘汰 | 均匀访问的缓存 |
| `volatile-lru` | 仅对设了 TTL 的 key 按 LRU 淘汰 | 混合场景：部分缓存 + 部分持久 |
| `volatile-lfu` | 仅对设了 TTL 的 key 按 LFU 淘汰 | 同上，带热度区分 |
| `volatile-random` | 仅对有 TTL 的 key 随机淘汰 | 少用 |
| `volatile-ttl` | 淘汰 TTL 最短的 key | 少用 |

### 配置方式

```bash
# redis.conf
# 设置最大内存（建议物理内存的 60-70%）
maxmemory 4gb

# 淘汰策略
maxmemory-policy allkeys-lru

# 每次淘汰的 key 数量（越大越精确，但阻塞时间越长）
maxmemory-samples 10

# 启用 LFU 时的对数计数器衰减因子
lfu-log-factor 10
lfu-decay-time 1
```

```java
// RedisTemplate 动态配置
@Component
public class RedisConfigManager {

    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * 动态修改淘汰策略（生产不建议频繁更改）
     */
    public void setEvictionPolicy(String policy) {
        redisTemplate.execute(
            (RedisCallback<Void>) conn -> {
                conn.setConfig("maxmemory-policy", policy);
                return null;
            }
        );
    }

    /**
     * 查看当前淘汰策略与内存使用
     */
    public Map<String, String> getMemoryInfo() {
        Properties info = redisTemplate.execute(
            (RedisCallback<Properties>) conn -> conn.info("memory")
        );
        Map<String, String> result = new HashMap<>();
        if (info != null) {
            result.put("used_memory_human", info.getProperty("used_memory_human"));
            result.put("used_memory_peak_human", info.getProperty("used_memory_peak_human"));
            result.put("maxmemory_human", info.getProperty("maxmemory_human"));
            result.put("maxmemory_policy", info.getProperty("maxmemory_policy"));
            result.put("evicted_keys", info.getProperty("evicted_keys"));
        }
        return result;
    }
}
```

## TTL 设计模式

### 合理的过期时间设置

```java
@Service
public class CacheService {

    // 1. 基础过期 + 随机偏移（防雪崩）
    public void setWithRandomExpiry(String key, String value, long baseTtl) {
        // 在基础 TTL 上 ±20% 随机偏移
        long jitter = (long) (baseTtl * 0.2 * (Math.random() * 2 - 1));
        long ttl = baseTtl + jitter;
        redisTemplate.opsForValue().set(key, value, ttl, TimeUnit.SECONDS);
    }

    // 2. 热点数据永不淘汰（但需要主动失效）
    public void setHotKey(String key, String value) {
        // 不设 TTL，依赖 allkeys-lru 淘汰
        redisTemplate.opsForValue().set(key, value);
    }

    // 3. 渐进式缓存（读时延长 TTL）
    public String getAndRefresh(String key, long refreshTtl) {
        String value = redisTemplate.opsForValue().get(key);
        if (value != null) {
            // 每次读取重新设置过期时间（"滑动窗口"模式）
            redisTemplate.expire(key, refreshTtl, TimeUnit.SECONDS);
        }
        return value;
    }
}
```

### 缓存穿透/击穿/雪崩完整方案

```java
@Service
public class CacheAvalancheService {

    @Autowired
    private StringRedisTemplate redisTemplate;

    @Autowired
    private RedissonClient redissonClient;

    /**
     * 解决缓存穿透：布隆过滤器前置过滤
     */
    public String getWithBloomFilter(String key, String bloomKey,
                                      Function<String, String> loader) {
        RBloomFilter<String> bloom = redissonClient.getBloomFilter(bloomKey);
        bloom.tryInit(1000000L, 0.01);

        // 1. 布隆过滤器判断不存在 → 直接返回（避免查 DB）
        if (!bloom.contains(key)) {
            return null;
        }

        // 2. 查缓存
        String value = redisTemplate.opsForValue().get(key);
        if (value != null) return value;

        // 3. 缓存未命中 → 查 DB
        // 注意：这里仍然有极小概率（1%）布隆误判导致穿透
        value = loader.apply(key);
        if (value != null) {
            redisTemplate.opsForValue().set(key, value, 300, TimeUnit.SECONDS);
        } else {
            // 缓存空值（防穿透）
            redisTemplate.opsForValue().set(key, "", 30, TimeUnit.SECONDS);
        }
        return value;
    }

    /**
     * 解决缓存击穿：分布式锁 + 单线程回源
     */
    public String getWithLock(String key, Function<String, String> loader) {
        // 1. 查缓存
        String value = redisTemplate.opsForValue().get(key);
        if (value != null) return value;

        // 2. 分布式锁（只让一个线程回源 DB）
        RLock lock = redissonClient.getLock("lock:" + key);
        try {
            if (lock.tryLock(3, 10, TimeUnit.SECONDS)) {
                // 双重检查：锁获取后可能已经有线程回源了
                value = redisTemplate.opsForValue().get(key);
                if (value != null) return value;

                // 回源 DB
                value = loader.apply(key);
                if (value != null) {
                    // 过期时间带随机偏移（防雪崩）
                    long ttl = 300 + (long)(Math.random() * 60);
                    redisTemplate.opsForValue().set(key, value, ttl, TimeUnit.SECONDS);
                }
                return value;
            } else {
                // 没拿到锁，短暂等待后重试缓存
                Thread.sleep(50);
                return redisTemplate.opsForValue().get(key);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return loader.apply(key);
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }

    /**
     * 解决缓存雪崩：多级缓存 + 过期时间打散
     */
    public String getWithMultiLevel(String key, Function<String, String> loader) {
        // 查一级缓存（本地 Caffeine）
        String value = localCache.getIfPresent(key);
        if (value != null) return value;

        // 查二级缓存（Redis）
        value = redisTemplate.opsForValue().get(key);
        if (value != null) {
            localCache.put(key, value);  // 回填本地缓存
            return value;
        }

        // 查三级（数据库）
        value = loader.apply(key);
        if (value != null) {
            // Redis TTL 加随机偏移
            long baseTtl = 300;
            long jitter = (long) (baseTtl * 0.3 * (Math.random() * 2 - 1));
            redisTemplate.opsForValue().set(key, value,
                baseTtl + jitter, TimeUnit.SECONDS);
            localCache.put(key, value);
        }
        return value;
    }
}
```

## 内存优化实战

### Big Key 处理

```java
/**
 * 查找 Big Key（扫描 Redis，仅用于诊断）
 */
public void scanBigKeys() {
    Set<String> bigKeys = new HashSet<>();
    ScanOptions options = ScanOptions.scanOptions()
        .count(1000)
        .build();

    try (Cursor<String> cursor = redisTemplate.scan(options)) {
        while (cursor.hasNext()) {
            String key = cursor.next();
            DataType type = redisTemplate.type(key);
            long size = 0;

            switch (type) {
                case STRING:
                    size = redisTemplate.opsForValue().size(key);
                    break;
                case LIST:
                    size = redisTemplate.opsForList().size(key);
                    break;
                case SET:
                    size = redisTemplate.opsForSet().size(key);
                    break;
                case ZSET:
                    size = redisTemplate.opsForZSet().size(key);
                    break;
                case HASH:
                    size = redisTemplate.opsForHash().size(key);
                    break;
            }

            if (size > 10000) {  // 超过 1万 个元素 = 大 key
                log.warn("Big Key 发现: key={}, type={}, size={}", key, type, size);
                bigKeys.add(key);
            }
        }
    }
}

/**
 * Big Key 拆分：Hash 分片
 * 场景：用户粉丝列表（最多千万级）
 */
public class HashShardingUtil {

    private static final int SHARD_SIZE = 1000;

    /**
     * 将大 Set 拆分为多个小 Hash（按 ID 哈希分片）
     */
    public String getShardKey(String baseKey, long memberId) {
        int shard = (int) (memberId % SHARD_SIZE);
        return baseKey + ":shard:" + shard;
    }

    // 写入：添加到对应分片
    public void addFollower(String baseKey, long userId, long followerId) {
        String shardKey = getShardKey(baseKey, userId);
        redisTemplate.opsForSet().add(shardKey, String.valueOf(followerId));
    }

    // 读取：查询是否关注
    public boolean isFollower(String baseKey, long userId, long followerId) {
        String shardKey = getShardKey(baseKey, userId);
        return Boolean.TRUE.equals(
            redisTemplate.opsForSet().isMember(shardKey, String.valueOf(followerId)));
    }
}
```

### 内存压缩

```java
// 1. 使用更小的 key 命名
// ❌ user:profile:1001:personal:information
// ✅ u:1001:profile

// 2. 使用整数代替字符串（intset 编码更紧凑）
// Redis 内部会用 intset 编码存储纯整数 Set

// 3. 使用 Hash 的 ziplist 编码（小 Hash 节省大量内存）
// 配置 redis.conf:
// hash-max-ziplist-entries 512
// hash-max-ziplist-value 64

// 4. 短结构优化
redisTemplate.opsForHash().putAll("user:1001", Map.of(
    "n", "张三",    // 短字段名
    "a", "28",
    "c", "北京"
));
// 替代长字段名: name, age, city
```

### 监控内存使用

```bash
# 查看内存使用详情
redis-cli> INFO memory
# used_memory_human: 2.50G
# used_memory_peak_human: 3.80G
# maxmemory_human: 4.00G
# mem_fragmentation_ratio: 1.02        # 碎片率 > 1.5 需重启
# evicted_keys: 0                      # 被淘汰的 key 数

# 查看最大使用内存的 key（需安装 redis-rdb-tools）
# rdb -c memory dump.rdb | head -20

# 实时监控 key 数量
redis-cli> DBSIZE

# 按类型统计 key 数量
redis-cli> INFO keyspace

# 查看内存碎片
redis-cli> MEMORY PURGE                # 主动整理碎片（阻塞）
redis-cli> MEMORY STAT
```

## 注意事项

1. **淘汰策略选择**：绝大多数缓存场景用 `allkeys-lru`，不要用 `volatile-*` 系列（因为 TTL 可能被忘记设置，导致内存不释放）
2. **Big Key 是万恶之源**：单个 key 超过 10MB 会导致集群 slot 迁移缓慢、网络 IO 阻塞、写操作延迟飙升。务必拆分
3. **Hot Key 处理**：单个 key QPS > 10万的场景，需用本地缓存（Caffeine）做二级缓存 + 读写分离
4. **maxmemory 不要设满物理内存**：留 20-30% 给操作系统页缓存和 fork 子进程（RDB/AOF 重写时需要）
5. **内存碎片**：频繁更新删除会导致内存碎片率上升（>1.5）。建议定时重启或使用 `MEMORY PURGE`
6. **`KEYS *` 是生产禁忌**：阻塞 Redis 所有操作。用 `SCAN` 替代
7. **`FLUSHALL`/`FLUSHDB` 不回内存**：只是标记 key 为过期，内存仍在。需重启或等 lazy-free 逐步释放
8. **启用 lazy-free 机制**（Redis 6.0+）：
   ```bash
   # redis.conf
   lazyfree-lazy-eviction yes     # 异步淘汰
   lazyfree-lazy-expire yes       # 异步过期
   lazyfree-lazy-server-del yes   # 异步删除
   replica-lazy-flush yes         # 从库异步清空
   ```
