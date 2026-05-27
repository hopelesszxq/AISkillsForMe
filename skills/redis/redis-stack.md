---
name: redis-stack
description: Redis Stack 模块：JSON、Search、TimeSeries 与高级模式
tags: [redis, stack, json, search, timeseries, redisson]
---

## Redis Stack 概述

Redis Stack 整合了 Redis 核心 + 多个模块，适合现代应用场景：
- **RediSearch**：全文搜索、二级索引、聚合查询
- **RedisJSON**：原生 JSON 文档操作（v2 API）
- **RedisTimeSeries**：时间序列数据，自动降采样
- **RedisBloom**：布隆过滤器、Top-K、Count-Min Sketch
- **RedisGraph**（已归档，推荐迁移到 FalkorDB）

## RedisJSON 使用

```bash
# 存储/读取 JSON
JSON.SET user:1001 $ '{"name":"张三","age":30,"tags":["vip","active"]}'
JSON.GET user:1001 $.name
# "张三"

# 局部更新
JSON.SET user:1001 $.age 31
JSON.ARRAPPEND user:1001 $.tags "premium"
# (integer) 3
```

```java
// Jedis + RedisJSON
Jedis jedis = new Jedis("localhost", 6379);
jedis.jsonSet("user:1001", 
    new JSONObject().put("name", "张三").put("age", 30));
String name = jedis.jsonGet("user:1001", "$.name");
```

## RediSearch 全文搜索

```bash
# 创建索引
FT.CREATE idx:products ON HASH PREFIX 1 "product:" 
  SCHEMA name TEXT WEIGHT 5.0 
         category TAG 
         price NUMERIC SORTABLE

# 写入数据
HSET product:1 name "iPhone 16 Pro" category "phone" price 9999
HSET product:2 name "MacBook Air M4" category "laptop" price 8999

# 搜索
FT.SEARCH idx:products "iPhone" LIMIT 0 10

# 带条件的聚合查询
FT.AGGREGATE idx:products "*" 
  GROUPBY 1 @category 
  REDUCE AVG 1 @price AS avg_price
```

```yaml
# Spring Boot 集成 Redisson + Redis Stack
spring:
  redis:
    host: localhost
    port: 6379
  redisson:
    config: |
      singleServerConfig:
        address: "redis://localhost:6379"
```

## 布隆过滤器防缓存穿透

```bash
BF.RESERVE bloom:users 0.01 1000000
BF.ADD bloom:users "user:1001"
BF.EXISTS bloom:users "user:1001"  # (integer) 1
```

```java
// 使用 Redisson 的 RBloomFilter
RBloomFilter<String> bloom = redissonClient
    .getBloomFilter("bloom:users", 
        new CodecConfiguration(StringCodec.INSTANCE));
bloom.tryInit(1000000L, 0.01);  // 容量 100万，误判率 1%
bloom.add("user:1001");
boolean exists = bloom.contains("user:1001");
```

## 时间序列 (TimeSeries)

```bash
# 创建时间序列，保留 7 天
TS.CREATE ts:cpu:usage RETENTION 604800
TS.ADD ts:cpu:usage * 45.2
TS.ADD ts:cpu:usage * 67.8

# 按分钟聚合查询（降采样）
TS.RANGE ts:cpu:usage - + 
  AGGREGATION AVG 60000  # 每分钟平均值
```

## Redisson 高级用法

```java
// 信号量（跨服务限流）
RSemaphore semaphore = redissonClient.getSemaphore("mySemaphore");
semaphore.trySetPermits(10);
semaphore.acquire(2);   // 获取2个许可
semaphore.release(2);   // 释放

// 计数锁（分布式读写锁）
RReadWriteLock rwLock = redissonClient.getReadWriteLock("data-lock");
rwLock.readLock().lock(10, TimeUnit.SECONDS);
// 读操作...
rwLock.readLock().unlock();

// 延迟队列
RQueue<String> queue = redissonClient.getQueue("delayed-tasks");
// 使用 RDelayedQueue 实现 Redis 版延迟队列
```

## 注意事项

- **Redis Stack 独立安装**：Redis Stack 不是默认安装，需单独下载（redis-stack-server）
- **模块版本兼容**：RedisJSON v1 已废弃，请使用 v2 API（`JSON.SET key $ json` 语法）
- **搜索索引内存**：RediSearch 索引占用内存较大，超出 `maxmemory` 限制可能导致 OOM
- **生产建议**：TimeSeries 需要合理设置 `RETENTION` 和 `LABELS`，避免无限制增长
- **Redisson 版本**：使用 Spring Boot 3.x 时选择 `redisson-spring-boot-starter` 3.x 版本
- **不要用 KEYS**：生产环境用 `SCAN` 代替 `KEYS *`，也可以考虑 Redis Stack 的 `FT.SEARCH`
