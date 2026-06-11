---
name: redisson-spring-boot-starter
description: Redisson Spring Boot Starter 实战：配置、分布式锁、集合、PubSub、Stream 与健康检查
tags: [redis, redisson, spring-boot, distributed-lock, pubsub, stream]
---

## 概述

Redisson 官方提供了 `redisson-spring-boot-starter`，零配置即可在 Spring Boot 中集成 Redisson 客户端。相比手动创建 `RedissonClient` Bean，Starter 方式支持自动装配、健康检查和外部化配置。

> 版本：Redisson 3.45.1+ | Spring Boot 3.4.x | JDK 21

## 1. 依赖配置

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.45.1</version>
</dependency>
```

自动引入 `redisson`、`redisson-spring-data` 等核心依赖，无需额外手动创建 Bean。

## 2. Redisson 配置（YAML）

在 `src/main/resources/redisson.yaml` 中配置，Starter 会自动加载：

### 单机模式

```yaml
singleServerConfig:
  address: "redis://localhost:6379"
  password: null
  clientName: redis-app
  connectionPoolSize: 64
  connectionMinimumIdleSize: 24
  idleConnectionTimeout: 10000
  connectTimeout: 10000
  timeout: 3000
  retryAttempts: 3
  retryInterval: 1500
  subscriptionsPerConnection: 5
  subscriptionConnectionMinimumIdleSize: 1
  subscriptionConnectionPoolSize: 50
  dnsMonitoring: false
  dnsMonitoringInterval: 5000
threads: 16
nettyThreads: 32
codec: !<org.redisson.codec.JsonJacksonCodec> {}
transportMode: NIO
```

### 集群模式

```yaml
clusterServersConfig:
  clientName: redis-cluster
  nodeAddresses:
    - "redis://localhost:6379"
    - "redis://localhost:6380"
    - "redis://localhost:6381"
    - "redis://localhost:6382"
    - "redis://localhost:6383"
    - "redis://localhost:6384"
  scanInterval: 1000
  slaveConnectionPoolSize: 64
  masterConnectionPoolSize: 64
  failedSlaveCheckInterval: 180000
  failedSlaveReconnectionInterval: 3000
  readMode: SLAVE
  subscriptionMode: MASTER
  loadBalancer: !<org.redisson.connection.balancer.RoundRobinLoadBalancer> {}
threads: 16
nettyThreads: 32
codec: !<org.redisson.codec.JsonJacksonCodec> {}
```

### 哨兵模式

```yaml
sentinelServersConfig:
  clientName: redis-sentinel
  masterName: mymaster
  sentinelAddresses:
    - "redis://sentinel1:26379"
    - "redis://sentinel2:26379"
    - "redis://sentinel3:26379"
  database: 0
```

### application.yml 配置（备选）

```yaml
spring:
  redis:
    redisson:
      # 指定 redisson 配置文件位置
      config: classpath:redisson.yaml
      # 或直接内联配置（file: 路径）
      # config: file:/etc/redisson.yaml
```

## 3. 集合操作（Collections）

Redisson 提供了丰富的分布式集合，与 Java 原生集合接口兼容。

```java
@Service
@RequiredArgsConstructor
public class CollectionService {
    private final RedissonClient redissonClient;

    // Bucket — K/V 存储（类似 String）
    public void bucketExample() {
        RBucket<String> bucket = redissonClient.getBucket("myKey");
        bucket.set("Hello, Redisson!");
        System.out.println(bucket.get());

        // 带 TTL 写入（自动过期）
        bucket.set("Hello with TTL!", Duration.ofSeconds(5));
    }

    // List — 有序列表
    public void listExample() {
        RList<String> list = redissonClient.getList("myList");
        list.add("Apple");
        list.add("Banana");
        System.out.println(list.get(0)); // Apple
    }

    // Set — 无序去重
    public void setExample() {
        RSet<String> set = redissonClient.getSet("mySet");
        set.add("A");
        set.add("B");
        set.add("A"); // 重复，不会添加
        System.out.println(set.contains("B")); // true
        // 设置 TTL
        redissonClient.getKeys().expire("mySet", 30, TimeUnit.SECONDS);
    }

    // Map — 分布式 Map
    public void mapExample() {
        RMap<String, Integer> map = redissonClient.getMap("myMap");
        map.put("views", 10);
        System.out.println(map.get("views")); // 10
    }

    // ScoredSortedSet — 带权重的有序集合（类似 ZSET）
    public void scoredSortedSetExample() {
        RScoredSortedSet<String> set = redissonClient.getScoredSortedSet("ranking");
        set.add(9.5, "Alice");
        set.add(8.0, "Bob");
        System.out.println(set.first()); // Alice（分数最高）
    }

    // Geo — 地理位置
    public void geoExample() {
        RGeo<String> geo = redissonClient.getGeo("myGeo");
        geo.add(13.361389, 38.115556, "Palermo");
        geo.add(15.087269, 37.502669, "Catania");
        double dist = geo.dist("Palermo", "Catania", GeoUnit.KILOMETERS);
        System.out.println("Distance: " + dist + " km");
    }

    // BitSet — 位图
    public void bitSetExample() {
        RBitSet bitSet = redissonClient.getBitSet("myBitSet");
        bitSet.set(0, true);
        bitSet.set(1, false);
        System.out.println(bitSet.get(0)); // true
    }

    // MapCache — 支持逐出策略的 Map
    public void mapCacheExample() {
        // 基于 Redis TTL
        RMapCache<String, String> mapCache = redissonClient.getMapCache("myMapCache");
        mapCache.put("hello", "world", 30, TimeUnit.SECONDS);

        // 本地缓存加速版（MapCacheNative）
        RMapCacheNative<String, String> nativeCache =
            redissonClient.getMapCacheNative("myNativeMapCache");
        nativeCache.put("hello", "world", Duration.ofSeconds(10));
    }
}
```

## 4. 分布式锁（Distributed Lock）

Redisson 分布式锁支持自动续期（看门狗 Watchdog），可重入，无需手动处理锁超时。

```java
@Service
@RequiredArgsConstructor
public class LockService {
    private final RedissonClient redissonClient;

    // 自动阻塞直到获取锁（看门狗自动续期，默认 30s）
    public void lock() {
        RLock lock = redissonClient.getLock("LOCK_ID");
        lock.lock();  // 阻塞直到成功
        try {
            // 业务逻辑
            System.out.println("Lock acquired");
        } finally {
            lock.unlock(); // 必须释放
        }
    }

    // 尝试获取锁（非阻塞），支持超时
    public void tryLock() {
        RLock lock = redissonClient.getLock("TRYLOCK_ID");
        try {
            // 等待 5 秒，持有 10 秒后自动释放
            if (lock.tryLock(5, 10, TimeUnit.SECONDS)) {
                try {
                    System.out.println("Lock acquired with tryLock");
                } finally {
                    lock.unlock();
                }
            } else {
                System.out.println("Failed to acquire lock");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### 看门狗（Watchdog）机制

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `lockWatchdogTimeout` | 锁超时时间（毫秒） | 30000 |
| 续期间隔 | watchdog 每 10s 续期一次 | watchdogTimeout / 3 |

- `lock.lock()` 默认开启 watchdog：业务未完成时自动续期，防止锁意外释放
- `lock.lock(10, TimeUnit.SECONDS)` 指定超时：**不开启 watchdog**，到期自动释放

## 5. PubSub 消息发布订阅

Redisson 提供 `RTopic`（标准）和 `RReliableTopic`（可靠）两种 PubSub 实现。

```java
@Service
@RequiredArgsConstructor
public class PubSubService {
    private final RedissonClient redissonClient;

    public void pubSubExample() {
        // 标准 Topic（即发即弃，订阅者离线丢失消息）
        RTopic topic = redissonClient.getTopic("myTopic");
        topic.addListener(Integer.class, (channel, msg) ->
            System.out.println("Received: " + msg));
        topic.publish(1);
        topic.publish(2);

        // 可靠 Topic（基于 List/Set 持久化，断线重连后恢复）
        RTopic reliableTopic = redissonClient.getReliableTopic("myReliableTopic");
        reliableTopic.addListener(Integer.class, (channel, msg) ->
            System.out.println("Reliable received: " + msg));
        reliableTopic.publish(1);
        reliableTopic.publish(2);
    }
}
```

**RTopic vs RReliableTopic 对比**

| 特性 | RTopic | RReliableTopic |
|------|--------|----------------|
| 消息持久化 | 否 | 是（基于 Redis List） |
| 断线重连恢复 | 否 | 是 |
| 性能 | 更高 | 稍低 |
| 适用场景 | 实时通知、缓存失效广播 | 重要事件、离线消费 |

## 6. Stream 消息队列

Redis Stream 支持消费组，类似 Kafka 的消费者组模型。

```java
@Service
@RequiredArgsConstructor
public class StreamService {
    private final RedissonClient redissonClient;

    public void streamExample() {
        RStream<String, String> stream = redissonClient.getStream("usersEvents");

        // 创建消费组（幂等）
        try {
            stream.createGroup(StreamCreateGroupArgs.name("usersGroup").makeStream());
        } catch (RedisBusyException e) {
            // 消费组已存在，忽略
        }

        // 发送消息
        Map<String, String> event = Map.of(
            "userId", "123",
            "timestamp", String.valueOf(System.currentTimeMillis())
        );
        stream.add(StreamAddArgs.entries(event));

        // 消费消息（读取未投递消息）
        Map<StreamMessageId, Map<String, String>> messages =
            stream.readGroup("usersGroup", "consumer-A",
                StreamReadGroupArgs.neverDelivered());

        messages.forEach((id, body) -> {
            System.out.printf("Processing [%s]: %s%n", id, body);
            stream.ack("usersGroup", id); // 手动 ACK
        });
    }
}
```

### 消费组关键参数

| 参数 | 说明 |
|------|------|
| `StreamCreateGroupArgs.name("group").makeStream()` | 创建消费组，若 Stream 不存在则自动创建 |
| `StreamReadGroupArgs.neverDelivered()` | 读取未投递的消息 |
| `StreamReadGroupArgs.entries(StreamId.MAX)` | 读取所有消息（含已投递未 ACK） |
| `stream.ack(group, id)` | 确认消息处理完成 |

## 7. 自定义健康检查（Health Indicator）

针对 Redis 集群环境，可以自定义健康检查指标，区分 Master 和 Slave。

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class RedisClusterHealthIndicator implements HealthIndicator {
    private final RedissonClient redissonClient;

    @Override
    public Health health() {
        RedisCluster cluster = redissonClient.getRedisNodes(RedisNodes.CLUSTER);

        // 检查所有 Master 节点
        Collection<RedisClusterMaster> masters = cluster.getMasters();
        for (RedisClusterMaster master : masters) {
            try {
                if (!master.ping()) {
                    return Health.down()
                        .withDetail("failedMaster", master.toString())
                        .build();
                }
            } catch (Exception e) {
                return Health.down()
                    .withDetail("failedMaster", master.toString())
                    .withDetail("error", e.getMessage())
                    .build();
            }
        }
        return Health.up()
            .withDetail("mastersChecked", masters.size())
            .build();
    }
}
```

## 8. 注意事项

1. **配置文件位置**：`redisson.yaml` 必须在 classpath 根目录，或在 `application.yml` 中通过 `spring.redis.redisson.config` 指定
2. **编解码器**：默认使用 Jackson，若使用 `RMapCache` 等需要序列化的结构，确保对象有默认构造函数
3. **Watchdog 与自定义超时互斥**：`lock.lock(time, unit)` 指定超时后 Watchdog 不会启动，到期自动释放锁，务必确保业务在超时前完成
4. **Stream 消费组幂等创建**：`createGroup` 会抛 `RedisBusyException` 如果组已存在，需要捕获忽略
5. **ReliableTopic 与 Stream 的区别**：ReliableTopic 适合一对多广播，Stream 适合消费者组分摊消费
6. **连接池调优**：`connectionPoolSize` 和 `connectionMinimumIdleSize` 根据并发量调整，避免连接不足或资源浪费
7. **集群 readMode**：`readMode: SLAVE` 读操作走从节点；`readMode: MASTER` 保证强一致性
