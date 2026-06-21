---
name: redisson-460-features
description: Redisson 4.6.0 新特性：T-digest 分位数估计、Top-K 高频元素统计、RMapCache 租约方法、SB 4.1 集成
tags: [redis, redisson, t-digest, top-k, probabilistic, mapcache, spring-boot-41]
---

## 概述

Redisson 4.6.0（2026-06-15 发布）在 4.5.0 基础上新增了 **T-digest** 和 **Top-K** 两种概率性数据结构，支持 Spring Boot 4.1.0 和 Spring Data Redis 4.1.0 集成，并增强了 RArray 和 RMapCache 的 API。

> 4.6.1（2026-06-18）是 Bugfix 版本，修复了 AsyncSemaphore 竞态条件和连接池初始化问题。

## 一、T-digest 分位数估计

T-digest 是一种**在线分位数估计算法**，可以在不保存全部数据的情况下近似计算 P50/P95/P99 等百分位值，适合延迟监控、性能分析等场景。

### 基本用法

```java
// 创建 T-digest（指定压缩比，精度与内存的平衡）
RTDigest digest = redisson.getTDigest("latency-p95");

// 添加观测值
digest.add(1.5, 2.3, 1.8, 5.0, 3.2);

// 批量添加
List<Double> values = Arrays.asList(1.1, 2.2, 3.3, 4.4, 5.5);
digest.addAll(values);

// 查询分位数
double p50 = digest.quantile(0.5);   // 中位数
double p95 = digest.quantile(0.95);  // P95
double p99 = digest.quantile(0.99);  // P99

// 查询累计分布
double percentileRank = digest.cdf(3.0);  // 值 <= 3.0 的比例

// 合并多个 T-digest
digest.union(otherDigest);

// 导出/导入（跨实例共享）
byte[] state = digest.toArray();
RTDigest restored = redisson.getTDigest("latency-p95-restored");
restored.fromArray(state);
```

### 应用场景

| 场景 | 说明 |
|------|------|
| 接口响应延迟监控 | 实时计算 P50/P95/P99，无需存所有原始值 |
| 业务指标聚合 | 多节点上报观测值，合并计算全局分位数 |
| 滑动窗口分析 | 定时重置，统计最近 N 分钟的分位数 |

### 配置参数

```java
// 创建时指定压缩比（默认 100）
// 数值越大精度越高，内存占用也越大
RTDigest digest = redisson.getTDigest("latency", 200);
```

## 二、Top-K 高频元素统计

Top-K 用于实时统计出现频率最高的 K 个元素，适合热点数据发现、趋势分析等场景。

### 基本用法

```java
// 创建 Top-K（统计 Top 10）
RTopK topK = redisson.getTopK("hot-keys", 10);

// 添加元素（记录出现次数）
topK.add("user:1001", "user:1002", "user:1001", "user:1003");
topK.addAll(Arrays.asList("user:1001", "user:1004", "user:1005"));

// 获取 Top-K 列表
Collection<String> topItems = topK.getAll();  // 按频率降序

// 获取元素频率
Integer count = topK.count("user:1001");

// 合并多个 Top-K
topK.union(otherTopK);
```

### 应用场景

```java
// 热点商品实时统计
RTopK hotItems = redisson.getTopK("hot-items", 20);

// 订单完成后记录商品 ID
orderService.onOrderPlaced(order -> {
    order.getItems().forEach(item ->
        hotItems.add(String.valueOf(item.getProductId()))
    );
});

// 定时获取 Top 20 热点商品（→ 缓存预热、推荐排序）
List<String> hotProducts = new ArrayList<>(hotItems.getAll());
```

### 注意事项

- Top-K 是**近似算法**（基于 Count-Min Sketch），高频元素准确，低频可能有偏差
- K 值越大内存开销越高，建议控制在 1000 以内
- 支持跨 JVM 共享数据（基于 Redis），适合分布式统计

## 三、Spring Boot 4.1.0 集成

Redisson 4.6.0 支持 Spring Boot 4.1.0 和 Spring Data Redis 4.1.0：

```yaml
# application.yml (Spring Boot 4.1 + Redisson)
spring:
  data:
    redis:
      host: localhost
      port: 6379
      redisson:
        config: classpath:redisson.yaml
```

```yaml
# redisson.yaml
singleServerConfig:
  address: "redis://localhost:6379"
  connectionPoolSize: 32
  connectionMinimumIdleSize: 8
codec: !<org.redisson.codec.Kryo5Codec> {}
```

```java
@Configuration
public class RedissonConfig {

    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer()
              .setAddress("redis://localhost:6379")
              .setConnectionPoolSize(32);
        return Redisson.create(config);
    }

    @Bean
    public RTDigest latencyDigest(RedissonClient client) {
        return client.getTDigest("api-latency");
    }

    @Bean
    public RTopK hotProducts(RedissonClient client) {
        return client.getTopK("hot-products", 20);
    }
}
```

## 四、RMapCache 租约方法增强

新增带**租约时间**的存取方法，元素自动过期后清理：

```java
RMapCache<String, String> cache = redisson.getMapCache("session-cache");

// 写入带 TTL
cache.putWithLease("token:abc", "user:1001", Duration.ofMinutes(30));

// 读取并获取剩余 TTL
MapCache.Entry<String, String> entry = cache.getWithLease("token:abc");
String value = entry.getValue();
long ttl = entry.getTtl();  // 剩余毫秒

// 删除并获取值和 TTL
MapCache.Entry<String, String> removed = cache.removeWithLease("token:abc");
```

## 五、RArray 新方法

```java
RArray<Double> arr = redisson.getArray("scores");

// 获取数组全量信息和最后 N 个元素
RArray.FullInfo<Double> info = arr.getFullInfo();
info.getSize();             // 数组大小
info.getCapacity();         // 容量
info.getType();             // 元素类型

// 逆序获取最后 N 个元素
List<Double> last10 = arr.lastItemsReversed(10);
```

## 注意事项

1. **T-digest vs 精确分位数**：T-digest 是近似算法，对极端尾部（P99.9+）精度较低，如果要求精确值请自行排序
2. **Top-K 内存管理**：K 值越大占用内存越多，建议按 `maxBytes` 监控实际使用量
3. **Kryo5Codec 安全性**：如果使用 yaml 配置 Kryo5Codec，务必配置 `allowedClasses` 限制反序列化白名单
4. **Redisson 4.6.1 修复**：如果遇到 AsyncSemaphore 阻塞或连接池卡住，建议升到 4.6.1
5. **版本兼容**：Redisson 4.6.x 需要 Redis 7.4+（推荐 8.0+），不支持旧版 Redis
- T-digest 和 Top-K 数据存储在 Redis 中，不同客户端共享同一实例，注意 key 命名冲突
