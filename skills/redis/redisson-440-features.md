---
name: redisson-440-features
description: Redisson 4.4.0 新特性：GCRA 限流器、非可重入锁、Hibernate 7.3 支持与 API 增强
tags: [redis, redisson, distributed-lock, rate-limiter, hibernate]
---

## 概述

Redisson 4.4.0（2026-05-12 发布）是一次重要的功能更新，引入了 **GCRA 限流器**、**非可重入锁**、**Hibernate 7.3.x 支持** 等关键特性。4.4.0 对原有分布式锁、布隆过滤器、搜索和流式处理 API 做了大量增强。

> 当前最新版本：**4.4.0** | Maven：`org.redisson:redisson:4.4.0`

## 核心新特性

### 1. GCRA Rate Limiter（通用信元速率算法）

新增基于 GCRA（Generic Cell Rate Algorithm）的限流器，比传统令牌桶更平滑、内存开销更小：

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>4.4.0</version>
</dependency>
```

```java
RedissonClient redisson = Redisson.create(config);

// 创建 GCRA 限流器
RRateLimiter limiter = redisson.getRateLimiter("api:rate:gcra");

// 初始化：每秒 100 个请求，突发 200
limiter.trySetRate(RateType.OVERALL, 100, 1, RateIntervalUnit.SECONDS);
limiter.setRateLimiterArgs(RateLimiterArgs.builder()
    .maxBurst(200)
    .build());

// 非阻塞获取许可
if (limiter.tryAcquire()) {
    // 处理请求
} else {
    // 限流拒绝
}

// 阻塞获取许可（最多等 5 秒）
boolean acquired = limiter.tryAcquire(5, TimeUnit.SECONDS);
```

**动态调整**（4.4.0 新增）：

```java
// 运行时更新限流参数
limiter.set(RateLimiterArgs.builder()
    .maxBurst(500)
    .build());

// 增量更新（不重置已有计数器）
limiter.update(RateLimiterArgs.builder()
    .maxBurst(300)
    .build());
```

### 2. 非可重入锁（Non-Reentrant Lock）

传统 Redisson 锁默认是可重入的（同一线程可重复加锁）。4.4.0 新增了**非可重入**变体，防止意外的重入导致锁语义错误：

```java
// 非可重入锁 — 同一线程再次加锁会阻塞
RLock lock = redisson.getLock("order:lock:non-reentrant");
lock.lock();    // 第一次加锁成功
lock.lock();    // 阻塞！因为是非可重入的

// 非可重入公平锁
RFairLock fairLock = redisson.getFairLock("order:fair:non-reentrant");
fairLock.lock();
```

**适用场景**：
- 同一方法内既调同步代码又调异步分支，避免误重入
- 严格的锁层次结构（不允许嵌套）
- 排查重入导致的不一致性 bug

### 3. Hibernate 7.3.x 支持

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-hibernate-73</artifactId>
    <version>4.4.0</version>
</dependency>
```

Redisson 现已支持 Hibernate 7.3.x 二级缓存和查询缓存，配置方式与之前版本一致：

```properties
# Hibernate 7.3 + Redisson 4.4.0
hibernate.cache.region.factory_class=org.redisson.hibernate.RedissonRegionFactory
hibernate.redisson.config=/redisson-config.yaml
```

## API 增强

### 4. RSearch 增强

```java
// 分析搜索查询的分布
RSearch search = redisson.getSearch("idx:products");
Map<String, Object> profile = search.profileSearch(
    "(@price:[100 500])",
    QueryOptions.create()
        .limit(0, 10)
);

// 聚合分析
Map<String, Object> aggProfile = search.profileAggregate(
    "FT.AGGREGATE idx:products @price:[0 +inf] GROUPBY 1 @category SORTBY 2 @price DESC"
);
```

### 5. RBloomFilter 批量检查

```java
RBloomFilter<String> bloom = redisson.getBloomFilter("users:bloom");
bloom.tryInit(1000000, 0.01);

// 批量判断是否存在（4.4.0 新增）
List<Boolean> results = bloom.exists(Arrays.asList("user1", "user2", "user3"));
// 返回 [true, false, true]
```

### 6. RStream.nack()

```java
RStream<String, String> stream = redisson.getStream("orders:stream");
StreamMessageId msgId = ...; // 已读取的消息

// 手动标记消息为未确认（4.4.0 新增）
// 适用于消费者的业务逻辑失败后重新入队
stream.nack("group-1", "consumer-1", msgId);
```

### 7. RJsonBucket 增强

```java
RJsonBucket bucket = redisson.getJsonBucket("sensor:data");

// 支持浮点数同构数组精度类型
bucket.set(jsonObject, 
    JsonBucketOptions.builder()
        .precision(JsonBucketOptions.Precision.HOMOGENEOUS_ARRAY)
        .build());
```

### 8. RedissonClient.shutdownAsync()

```java
// 异步关闭（4.4.0 新增）
CompletionStage<Void> future = redisson.shutdownAsync();
future.thenRun(() -> log.info("Redisson 已优雅关闭"));
```

### 9. 新的监听器接口

```java
// Map 增量监听
RMapCache<String, Integer> map = redisson.getMapCache("cache");
map.addListener(new MapIncrListener() {
    @Override
    public void onIncr(String key, Number delta, Number newValue) {
        log.info("Key {} 增加 {}，当前值 {}", key, delta, newValue);
    }
});

// Deque 操作监听
RDeque<String> deque = redisson.getDeque("tasks");
deque.addListener(new DequeAddFirstListener() {
    @Override
    public void onAddFirst(String element) {
        log.info("元素 {} 插入队列头部", element);
    }
});
```

### 10. 向量搜索改进

```java
// 距离函数和分片 K 值设置
VectorSimilarityNearestNeighbors.Params params = VectorSimilarityNearestNeighbors.Params.create()
    .yieldDistanceAs("distance")
    .shardKRatio(0.8)  // 分片 K 值比例
    .topK(10);

RScoredSortedSet<String> set = redisson.getScoredSortedSet("vectors");
// COUNT 聚合选项
RScoredSortedSet.Aggregate aggregate = RScoredSortedSet.Aggregate.COUNT;
```

## 配置增强

```yaml
# redisson-config.yaml
singleServerConfig:
  address: "redis://127.0.0.1:6379"
  # DNS 监控间隔（4.4.0 新增）
  dnsMonitoringTimes: 5000
  # 主节点加载回退（4.4.0 新增）
  fallbackLoadingToMaster: true
```

## 注意事项

1. **GCRA vs 令牌桶**：GCRA 限流器在突发后恢复更平滑，适合 API 网关和微服务限流场景；令牌桶适合简单的固定速率限流
2. **非可重入锁风险**：如果现有代码有隐式重入逻辑（如 AOP 切面中已有锁再调远程服务），使用非可重入锁会导致死锁，务必先审查调用链
3. **Hibernate 7.3**：仅 `redisson-hibernate-73` 模块兼容，旧项目仍需使用 `redisson-hibernate-6` 或 `redisson-hibernate-62`
4. **RStream.nack()**：仅在消费者组模式下有效，独立消费者不支持 nack
5. **shutdownAsync()**：推荐在 Spring `@PreDestroy` 或 ApplicationListener 中使用异步关闭，避免阻塞主线程
6. **JSON 精度**：`HOMOGENEOUS_ARRAY` 精度模式仅对同构浮点数组生效，异构数组或非浮点类型不适用
