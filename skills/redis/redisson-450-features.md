---
name: redisson-450-features
description: Redisson 4.5.0 新特性：Array 对象、BitVector Store、readMode 配置、RMap.keysAsync 与 RVectorSet 增强
tags: [redis, redisson, array, bitvector, map, collection]
---

## 概述

Redisson 4.5.0（2026-06-05 发布）在 4.4.0 基础上新增了 **Array 对象**、**BitVector Store** 等数据结构，并增强了 Map、原子计数器和向量集合的 API。

> 当前最新版本：**4.5.0** | Maven：`org.redisson:redisson:4.5.0`

## 核心新特性

### 1. Array 对象（与 Redis 8.8 ARRAY 类型对齐）

Redisson 4.5.0 新增 `RArray` 接口，对应 Redis 8.8 引入的原生 Array 数据结构，提供类型化、紧凑内存的数组操作：

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>4.5.0</version>
</dependency>
```

```java
RedissonClient redisson = Redisson.create(config);

// 创建 FLOAT 类型数组
RArray<Double> array = redisson.getArray("sensor:readings");

// 初始化数组（指定容量）
array.trySet(new double[]{1.0, 2.0, 3.0, 4.0, 5.0});

// 追加元素
array.add(6.0);

// 按索引获取
double val = array.get(0);

// 范围查询
List<Double> range = array.range(0, 3);  // [1.0, 2.0, 3.0, 4.0]

// 获取长度
int size = array.size();

// 迭代
for (Double d : array) {
    System.out.println(d);
}
```

**支持的类型**：

| 类型常量 | Redis 类型 | 说明 |
|---------|-----------|------|
| `ArrayType.INT8` | INT8 | 8位有符号整数 |
| `ArrayType.INT16` | INT16 | 16位有符号整数 |
| `ArrayType.INT32` | INT32 | 32位有符号整数 |
| `ArrayType.INT64` | INT64 | 64位有符号整数 |
| `ArrayType.FLOAT` | FLOAT | 32位浮点数 |
| `ArrayType.DOUBLE` | DOUBLE | 64位浮点数 |
| `ArrayType.BFLOAT16` | BFLOAT16 | 16位脑浮点（ML 场景） |

```java
// 指定类型创建
RArray<Integer> intArray = redisson.getArray("scores", ArrayType.INT32);
intArray.trySet(new Integer[]{100, 200, 300});
```

### 2. BitVector Store

新增 `RBitVector` 接口，提供高效的位向量存储操作，适合布隆过滤器之外的大规模位图场景：

```java
RBitVector bitVector = redisson.getBitVector("feature:flags");

// 设置位
bitVector.set(100, true);

// 批量设置
bitVector.setAll(50, 100, true);  // 设置 50-100 位为 true

// 获取位
boolean flag = bitVector.get(100);

// 统计 true 的数量
long count = bitVector.cardinality();

// 范围查询
BitSet bitset = bitVector.getRange(0, 1000);

// 位运算
bitVector.or("other:vector");
bitVector.and("other:vector");
bitVector.xor("other:vector");
bitVector.not();
```

**适用场景**：
- 用户每日签到记录（位图压缩存储）
- 功能开关大规模管理
- 在线状态追踪

### 3. readMode 配置支持

4.5.0 为 `MapOptions`、`PlainOptions`、`LocalCachedMapOptions` 新增了 `readMode` 设置，控制读取时对副本的访问策略：

```java
// 优先从 SLAVE 读取（适合读写分离场景）
MapOptions<String, Object> options = MapOptions.<String, Object>defaults()
    .readMode(MapOptions.ReadMode.SLAVE);

RMap<String, Object> map = redisson.getMap("config-cache", options);

// 其他 readMode：
// MapOptions.ReadMode.MASTER  - 始终读 MASTER（默认，保证强一致性）
// MapOptions.ReadMode.SLAVE   - 优先读 SLAVE（牺牲一致性换取性能）
// MapOptions.ReadMode.ANY     - 任意可用节点
```

```yaml
# YAML 配置方式
codec: !<org.redisson.codec.JsonJacksonCodec> {}
readMode: "SLAVE"  # 全局 readMode 配置
```

### 4. RMap.keysAsync()

新增异步批量获取键的方法，适合大 Map 场景下避免阻塞：

```java
RMap<String, Object> map = redisson.getMap("large-map");

// 异步获取所有键
CompletionStage<Set<String>> keysFuture = map.keysAsync();

keysFuture.thenAccept(keys -> {
    log.info("Map 包含 {} 个键", keys.size());
    // 对键进行批量处理
});
```

### 5. RVectorSet 增强

`RVectorSet` 新增以下方法：

```java
RVectorSet<String> vectorSet = redisson.getVectorSet("vectors");

// 判断是否包含元素
boolean exists = vectorSet.contains("embedding-1");

// 范围迭代
vectorSet.range(0, 100).forEach(System.out::println);

// 迭代器
Iterator<String> it = vectorSet.iterator();
while (it.hasNext()) {
    process(it.next());
}
```

`RVectorSet` 现在也支持 `RBatch` 批量操作：

```java
RBatch batch = redisson.createBatch();
RVectorSetAsync<String> asyncSet = batch.getVectorSet("vectors");
asyncSet.addAsync("new-vector");
asyncSet.containsAsync("existing-vector");
batch.execute();
```

### 6. 扩展的原子计数器方法

`RAtomicLong` 和 `RAtomicDouble` 新增了扩展的 `incrementAndGet()` 方法：

```java
RAtomicLong counter = redisson.getAtomicLong("visitor:count");

// 增量自增并返回（支持自定义增量值）
long newVal = counter.incrementAndGet(5);  // +5
long newVal2 = counter.incrementAndGet(-2); // -2
```

## Breaking Change

Map 监听器（`RMapCacheListener`、`RMapListener` 等）的签名已变更，新增了 `fieldName` 参数：

```java
// 4.4.0 旧版签名
void onEntryCreated(String key, V value);

// 4.5.0 新版签名
void onEntryCreated(String key, V value, String fieldName);
//                                      ^^^^^^^^^^ 新增参数
```

升级时所有自定义 `MapListener` 实现都需要调整。

## Bug 修复亮点

| 问题 | 说明 |
|------|------|
| `RScoredSortedSet` Rx/Reactive | 空结果时返回 `absent` 而非空 Optional |
| `PingConnectionHandler` | 修复竞态条件导致连接误判问题 |
| UUID 类型元数据泄漏 | `TypedJsonJacksonCodec` 修复 |
| `LZ4CodecV2` | 缓冲区截断修复 |
| 集群 TLS `WRONGPASS` | 修复 4.4.0 引入的集群从节点 TLS 密码设置回归 |
| 事务 `RMap.fastRemove()` | 释放不存在键的锁 |

## 注意事项

1. **Array 类型不兼容旧客户端**：`RArray` 依赖 Redis 8.8+ 的 `ARRAY.*` 命令，需要服务端 ≥ 8.8
2. **Map 监听器签名破坏性变更**：升级后需重新编译所有监听器实现
3. **readMode=SLAVE** 需要集群/哨兵模式下配置从节点，单节点模式等同于 MASTER
4. **BitVector vs RBloomFilter**：`RBitVector` 是精确位图，`RBloomFilter` 是概率性去重，场景不同
5. **IncrementAndGet 负值**：支持负值，可同时作为 decrement 的替代
