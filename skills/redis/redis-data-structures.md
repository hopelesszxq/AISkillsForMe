---
name: redis-data-structures
description: Redis 高级数据结构与实战应用场景
tags: [redis, data-structures, stream, geospatial]
---

## 高级数据结构总览

| 结构 | Redis 命令前缀 | 核心能力 |
|---|---|---|
| Bitmap | SETBIT / GETBIT / BITCOUNT | 亿级布防、签到统计 |
| HyperLogLog | PFADD / PFCOUNT / PFMERGE | 亿级 UV 去重（误差~0.81%） |
| GEO | GEOADD / GEORADIUS / GEODIST | 地理位置检索 |
| Stream | XADD / XREAD / XGROUP | 消息队列（支持消费者组） |
| Bloom Filter | BF.ADD / BF.EXISTS | 缓存穿透防护（需 Redis Stack） |

## Bitmap 签到系统

```java
// 用户签到：user:sign:2025:07:{userId} → 7月第N天签到
// key: user:sign:2025:07:1001, offset=日期-1, value=1

// 签到（7月3日）
redisTemplate.opsForValue().setBit("user:sign:2025:07:1001", 2, true);

// 查询7月签到情况
BitSet bitSet = new BitSet();
for (int day = 0; day < 31; day++) {
    boolean signed = redisTemplate.opsForValue()
        .getBit("user:sign:2025:07:1001", day);
    bitSet.set(day, signed);
}

// 当月连续签到天数（从今天往前数）
int continuousDays = 0;
for (int day = today - 1; day >= 0; day--) {
    if (redisTemplate.opsForValue()
        .getBit("user:sign:2025:07:1001", day)) {
        continuousDays++;
    } else break;
}

// BITCOUNT 统计月度总签到天数
Long totalDays = redisTemplate.execute(
    (RedisCallback<Long>) conn -> conn.bitCount("user:sign:2025:07:1001".getBytes())
);
```

## HyperLogLog UV 统计

```java
// 百万 UV 去重约 12KB 内存（误差 0.81%）
String key = "page:uv:article:1001";

// 用户访问记录
redisTemplate.opsForHyperLogLog().add(key, "user_1001", "user_1002");

// 获取近似 UV
long uv = redisTemplate.opsForHyperLogLog().size(key);

// 合并多日 PV → 周/月 UV
// PFMERGE page:uv:article:1001:week page:uv:article:1001:20250701 ...
redisTemplate.opsForHyperLogLog().union("weekly_uv", "daily_1", "daily_2");
```

## GEO 地理检索

```java
// 存储门店坐标
redisTemplate.opsForGeo().add("stores", 
    new Point(116.397128, 39.916527), "store_beijing");
redisTemplate.opsForGeo().add("stores",
    new Point(121.473701, 31.230416), "store_shanghai");

// 查询附近门店（半径 5km 内，返回距离）
Circle circle = new Circle(new Point(116.4, 39.92), 
                           new Distance(5, RedisGeoCommands.DistanceUnit.KILOMETERS));
GeoResults<GeoLocation<String>> results = redisTemplate.opsForGeo()
    .radius("stores", circle);

// 两点间距离
Distance dist = redisTemplate.opsForGeo()
    .distance("stores", "store_beijing", "store_shanghai", 
              RedisGeoCommands.DistanceUnit.KILOMETERS);
// dist.getValue() ≈ 1068 km
```

## Stream 消息队列

```java
// 生产者
String eventId = redisTemplate.opsForStream()
    .add(StreamRecords.newRecord()
        .ofMap(Map.of("orderId", "ORD123456", "amount", "99.00"))
        .withStreamKey("orders:stream"));

// 消费者组（支持 ACK 机制）
redisTemplate.opsForStream()
    .createGroup("orders:stream", "order-consumer-group");

// 消费者读取（阻塞读取）
StreamReadOptions readOptions = StreamReadOptions.empty()
    .count(10).block(Duration.ofSeconds(5));

List<MapRecord<String, Object, Object>> messages = redisTemplate.opsForStream()
    .read(Consumer.from("order-consumer-group", "consumer-1"), 
          readOptions, 
          StreamOffset.create("orders:stream", ReadOffset.lastConsumed()));

// 确认消费
for (MapRecord<String, Object, Object> msg : messages) {
    processOrder(msg);
    redisTemplate.opsForStream().acknowledge("orders:stream", 
        "order-consumer-group", msg.getId());
}
```

## Bloom Filter 缓存穿透防护

```shell
# Redis Stack 原生支持
BF.RESERVE user_filter 0.01 1000000  # 初始化，误差1%，容量100万
BF.ADD user_filter user_1001
BF.EXISTS user_filter user_1001  # → 1
BF.EXISTS user_filter user_9999  # → 0（肯定不存在）
```

```java
// Redisson 布隆过滤器
RBloomFilter<String> bloomFilter = redissonClient
    .getBloomFilter("user_filter");
bloomFilter.tryInit(1000000L, 0.01);  // 容量100万，误差1%
bloomFilter.add("user_1001");
bloomFilter.contains("user_1001");  // true
bloomFilter.contains("user_9999");  // false
```

## Lua 脚本保证原子性

```lua
-- 库存扣减脚本
local key = KEYS[1]           -- stock:product:{id}
local quantity = tonumber(ARGV[1])
local stock = redis.call('GET', key)
if not stock or tonumber(stock) < quantity then
    return 0  -- 库存不足
end
redis.call('DECRBY', key, quantity)
return 1     -- 扣减成功
```

```java
// 执行 Lua 脚本
DefaultRedisScript<Long> script = new DefaultRedisScript<>();
script.setScriptText("...");
script.setResultType(Long.class);

Long result = redisTemplate.execute(script, 
    List.of("stock:product:1001"), "1");
```

## 注意事项

1. **Stream vs Pub/Sub**：Pub/Sub 丢消息不持久化，Stream 支持持久化和消费者组
2. **Big Key 问题**：单个 key 超过 10MB 会导致集群读写倾斜，需拆分（Hash 分片）
3. **Hot Key 问题**：单个 key QPS 过高可本地缓存（Caffeine）+ 读写分离
4. **内存淘汰策略**：allkeys-lru（缓存场景）vs noeviction（数据可靠场景）
5. **Pipeline 与 Transaction**：Pipeline 只保证打包发送不保证原子性，MULTI/EXEC 保证原子性但略慢
6. **集群模式限制**：不支持跨 slot 的 MULTI/EXEC 和 Lua 脚本（需 hash tag 处理）
