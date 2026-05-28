---
name: redis-cache-consistency
description: Redis 缓存预热、缓存一致性（双写一致）与旁路缓存模式实战
tags: [redis, cache, consistency, write-through, cache-aside]
---

## 缓存一致性（双写一致）

### 核心问题

数据库与 Redis 之间的数据不一致由并发写和异常时序导致。解决方案按严格程度分级：

| 策略 | 一致性 | 复杂度 | 推荐场景 |
|------|--------|--------|----------|
| 先写 DB 后删 Cache（Cache Aside） | 最终一致 | 低 | ✅ 通用推荐 |
| 先写 DB 后写 Cache | 最终一致 | 低 | 读多写少的缓存预热 |
| 延时双删 | 最终一致 | 中 | 主从同步延迟场景 |
| 读写锁 + 版本号 | 强一致 | 高 | 金融级场景 |
| Canal + MQ 订阅 binlog | 最终一致 | 高 | 异构缓存刷新 |

### 推荐方案：Cache Aside Pattern

```
读流程：
  查询 Cache → 命中 → 返回
              ↓ 未命中
  查询 DB → 写入 Cache → 返回

写流程：
  更新 DB → 删除 Cache（不是更新 Cache！）
```

```java
@Service
public class UserService {
    @Autowired
    private StringRedisTemplate redis;
    @Autowired
    private UserMapper userMapper;

    // 读：Cache Aside
    public User getUser(Long id) {
        String key = "user:" + id;
        // 1. 查缓存
        String json = redis.opsForValue().get(key);
        if (json != null) {
            return JSON.parseObject(json, User.class);
        }
        // 2. 缓存未命中，查 DB
        User user = userMapper.selectById(id);
        if (user != null) {
            // 3. 写入缓存（设过期时间）
            redis.opsForValue().set(key, JSON.toJSONString(user), 1, TimeUnit.HOURS);
        }
        return user;
    }

    // 写：先更新 DB，再删除缓存（不是更新缓存）
    @Transactional
    public void updateUser(User user) {
        userMapper.updateById(user);
        // 删除缓存
        redis.delete("user:" + user.getId());
    }
}
```

### 为什么是删除缓存而不是更新缓存？

1. **写频繁场景**：如果每秒更新100次，读只有1次，更新缓存浪费大量资源
2. **并发一致性问题**：A 更新 DB → B 更新 DB → B 更新缓存 → A 更新缓存（旧数据覆盖新数据）
3. **延迟加载**：删除让下次读取时"被动"加载，数据源是 DB 而非旧缓存

## 延时双删策略

适用于 MySQL 主从同步有延迟的场景：

```java
@Transactional
public void updateUserSafe(User user) {
    // 1. 第一次删除缓存
    redis.delete("user:" + user.getId());

    // 2. 更新 DB
    userMapper.updateById(user);

    // 3. 延迟 N 毫秒后再次删除（等待主从同步完成）
    Thread.sleep(500); // 实际用线程池异步执行
    redis.delete("user:" + user.getId());
}
```

## 最终一致性方案：Canal + MQ

```java
// Canal 监听 MySQL binlog → 发送到 RocketMQ
// 消费者：刷新 Redis 缓存

@Component
@RocketMQMessageListener(topic = "canal-user-update", consumerGroup = "cache-refresh")
public class CacheRefreshListener implements RocketMQListener<CanalMessage> {
    @Override
    public void onMessage(CanalMessage msg) {
        String cacheKey = msg.getTable() + ":" + msg.getPrimaryKey();
        // 删除缓存，下次查询自动刷新
        redis.delete(cacheKey);
    }
}
```

## 缓存预热

### 启动时预热关键数据

```java
@Component
public class CachePreloader implements CommandLineRunner {
    @Autowired
    private StringRedisTemplate redis;
    @Autowired
    private HotDataService hotDataService;

    @Override
    public void run(String... args) {
        // 预热 Top 1000 热门商品
        List<HotProduct> hotProducts = hotDataService.getTopHotProducts(1000);
        for (HotProduct p : hotProducts) {
            String key = "product:hot:" + p.getId();
            redis.opsForValue().set(key, JSON.toJSONString(p), 30, TimeUnit.MINUTES);
        }
        log.info("Cache preloaded: {} products", hotProducts.size());
    }
}
```

### 定时刷新热数据

```java
// 每 5 分钟刷新热点数据
@Scheduled(fixedRate = 300_000)
public void refreshHotCache() {
    String lockKey = "cache:preload:lock";
    Boolean locked = redis.opsForValue().setIfAbsent(lockKey, "1", 60, TimeUnit.SECONDS);
    if (Boolean.TRUE.equals(locked)) {
        try {
            List<HotProduct> hotProducts = hotDataService.getTopHotProducts(1000);
            // 批量刷新，避免缓存击穿
            for (HotProduct p : hotProducts) {
                String key = "product:hot:" + p.getId();
                redis.opsForValue().set(key, JSON.toJSONString(p), 30, TimeUnit.MINUTES);
            }
        } finally {
            redis.delete(lockKey);
        }
    }
}
```

## 缓存穿透 / 击穿 / 雪崩进阶处理

### 布隆过滤器精确控制

```java
@Component
public class BloomFilterCacheGuard {
    @Autowired
    private RedissonClient redisson;

    private static final String BLOOM_KEY = "bloom:user-ids";

    @PostConstruct
    public void init() {
        RBloomFilter<Long> bloom = redisson.getBloomFilter(BLOOM_KEY, 10_000_000, 0.01);
        if (!bloom.isExists()) {
            bloom.tryInit(10_000_000, 0.01);
        }
    }

    public boolean mightContain(Long userId) {
        return redisson.getBloomFilter(BLOOM_KEY).contains(userId);
    }
}
```

### 缓存空值（防止穿透）

```java
public User getUser(Long id) {
    String key = "user:" + id;
    String json = redis.opsForValue().get(key);
    if (json != null) {
        // 缓存空值标记
        if ("NULL".equals(json)) return null;
        return JSON.parseObject(json, User.class);
    }
    User user = userMapper.selectById(id);
    // 无论是 null 还是真实数据，都缓存
    redis.opsForValue().set(key,
        user != null ? JSON.toJSONString(user) : "NULL",
        user != null ? 1 : 5, // 空值缓存更短
        TimeUnit.MINUTES);
    return user;
}
```

## 注意事项

1. **删除缓存失败**：使用重试机制或 MQ 异步重试，最终一致即可
2. **缓存过期时间**：添加随机偏移量 `base + random(0, 300)` 秒，防止缓存雪崩
3. **大 Key 删除**：`UNLINK` 替代 `DEL`，异步释放内存
4. **热 Key 检测**：监控 QPS > 阈值时自动本地缓存复制（Caffeine + Redis 多级缓存）
5. **缓存预热时机**：禁止在业务高峰期批量预热，分批淡入
