---
name: redis-88-features
description: Redis 8.8.0 GA 新特性：Array 数据结构、INCREX 限流器、XNACK 流确认、Hash 字段级通知
tags: [redis, array, rate-limiter, streams, pubsub, data-structures]
---

## 概述

Redis 8.8.0（2026-05-25 GA 发布）引入了多项重要新特性，包括全新的 **Array 原生数据结构**、**INCREX 窗口计数限流器**、**XNACK 流消息显式确认** 以及 **Hash 字段级子键通知**。

> 本文补充 `redis8-features.md`（覆盖 Redis 8.0 基础特性），关注 8.8 新增特性。

## 一、Array 数据结构（antirez 实现）

Redis 8.8 引入了全新的原生 **Array（数组）** 数据结构，由 Redis 创始人 antirez 亲自实现。这是一个紧凑的、类型化的数组，支持元素级别的 O(1) 访问。

### 基本命令

```bash
# 创建数组（指定类型和容量）
ARRAY.CREATE myarr FLOAT 1000
ARRAY.CREATE user_scores INT 10000

# 追加元素
ARRAY.APPEND myarr 3.14 2.718 1.618

# 获取指定位置元素
ARRAY.GET myarr 0

# 设置指定位置元素
ARRAY.SET myarr 0 3.14159

# 获取数组长度
ARRAY.LEN myarr

# 获取数组范围
ARRAY.RANGE myarr 0 10

# 删除数组
DEL myarr
```

### 支持的类型

| 类型 | 说明 | 字节/元素 |
|------|------|----------|
| `INT8` | 8位有符号整数 | 1 |
| `INT16` | 16位有符号整数 | 2 |
| `INT32` | 32位有符号整数 | 4 |
| `INT64` | 64位有符号整数 | 8 |
| `FLOAT` | 32位浮点数 | 4 |
| `DOUBLE` | 64位浮点数 | 8 |
| `BFLOAT16` | 16位脑浮点 | 2 |

### Java 示例（Jedis）

```java
// 创建 FLOAT 数组
jedis.sendCommand("ARRAY.CREATE", "sensor_readings", "FLOAT", "86400");

// 批量追加
for (double value : readings) {
    jedis.sendCommand("ARRAY.APPEND", "sensor_readings", String.valueOf(value));
}

// 范围查询
List<String> range = jedis.sendCommand("ARRAY.RANGE", "sensor_readings", "0", "100");
```

### 注意事项

- Array 是**紧凑内存布局**的类型化数组，比 List 或 Sorted Set 更省内存
- Array 不支持**插入/删除中间元素**（类似定长数组）
- 常用于时序数据、ML 特征向量、传感器读数等场景
- 与 Vector Set（Redis 8.0 引入）不同，Array 是纯数据容器，无索引/搜索能力

## 二、INCREX — 窗口计数限流器

`INCREX` 是 Redis 8.8 新增的原子命令，将 `INCR`、`INCRBY`、`INCRBYFLOAT` 与过期时间、上下界检查合并为一个原子操作，**专为限流场景设计**。

### 命令语法

```bash
INCREX key increment [NX] [XX] [GT] [LT] [EX seconds|PX milliseconds|EXAT unix-time|PXAT unix-time|PERSIST]
```

- `NX`：仅在 key 不存在时执行
- `XX`：仅在 key 已存在时执行
- `GT`：仅在自增后值大于给定值时执行
- `LT`：仅在自增后值小于给定值时执行
- `EX/PX/EXAT/PXAT`：设置过期时间
- `PERSIST`：移除过期时间

### 限流器实现

```bash
# API 限流：每分钟最多 100 次
INCREX user:api:123 EX 60
# 返回值 > 100 表示超限，需拒绝请求

# 带上下界检查的限流
INCREX rate:ip:192.168.1.1 NX EX 60
# 首次调用创建 key=1，后续自增
# 结合 GT/LT 判断是否超限
```

### Java 限流器示例

```java
@Service
public class RateLimiter {

    @Autowired
    private StringRedisTemplate redis;

    /**
     * 检查是否允许请求（窗口计数器限流）
     * @param key 限流 key（如 "rate:user:123"）
     * @param limit 窗口内最大请求数
     * @param windowSec 窗口大小（秒）
     * @return true=允许, false=超限
     */
    public boolean allowRequest(String key, int limit, long windowSec) {
        // INCREX 原子操作：自增 + 设置过期时间
        String result = (String) redis.execute(
            (RedisCallback<Object>) conn -> conn.execute(
                "INCREX", key.getBytes(),
                ("NX").getBytes(),           // 首次创建
                ("EX").getBytes(),            // 设置过期
                String.valueOf(windowSec).getBytes()
            )
        );

        if (result == null) return false;

        long count = Long.parseLong(result);
        return count <= limit;
    }
}
```

### 对比传统方案

```bash
# 传统方式（非原子，有竞态）
MULTI
  INCR key
  EXPIRE key 60
  GET key
EXEC

# INCREX 方式（原子，一行命令）
INCREX key NX EX 60
```

## 三、XNACK — 流消息显式确认

Redis 8.8 新增 `XNACK` 命令，允许消费者**显式拒绝/释放**已读取但未处理的消息，将其重新变为待消费状态。

### 命令语法

```bash
XNACK key group consumer id [id ...]
```

### 使用场景

```bash
# 消费者读取消息后，发现无法处理
XREADGROUP GROUP mygroup consumer1 COUNT 1 STREAMS mystream >

# 拒绝该消息，放回 PEL 供其他消费者处理
XNACK mystream mygroup consumer1 1746000000000-0

# 批量拒绝
XNACK mystream mygroup consumer1 1746000000000-0 1746000000001-0 1746000000002-0
```

### 与 XACK/XCLAIM 对比

| 命令 | 行为 | 适用场景 |
|------|------|---------|
| `XACK` | 确认已处理，从 PEL 移除 | 正常消费完成 |
| `XNACK` | 显式拒绝，放回 PEL | 消息处理失败，交给其他消费者 |
| `XCLAIM` | 转移 PEL 消息给其他消费者 | 消费者崩溃后的故障转移 |

### Spring Boot 集成

```java
@StreamListener("mystream")
public void handleMessage(Map<String, Object> message) {
    try {
        // 处理消息
        processMessage(message);
        // ✅ 正常确认
    } catch (BusinessException e) {
        // ❌ 业务拒绝：放回队列
        redisTemplate.opsForStream()
            .nack("mystream", "mygroup", record.getRecordId());
    }
}
```

## 四、Hash 字段级子键通知

Redis 8.8 扩展了 keyspace notification，支持 **字段级别的 Hash 变更通知**，不再只能监听整个 key 的变化。

### 启用配置

```bash
# redis.conf
notify-keyspace-events Kh     # 'K' keyspace, 'h' hash, 新增 'H' hash-field
```

### 订阅字段级通知

```bash
# 订阅特定 hash 字段的变更
PSUBSCRIBE __keyevent@0__:hset    # 任意 hash 字段修改事件
PSUBSCRIBE __keyspace@0__:user:123:hset:name  # 特定字段修改事件
```

### 事件格式

| 事件 | 说明 | 频道 |
|------|------|------|
| `hset:<field>` | 字段被设置 | `__keyspace@<db>__:<key>` |
| `hdel:<field>` | 字段被删除 | `__keyspace@<db>__:<key>` |
| `hincrby:<field>` | 字段自增 | `__keyspace@<db>__:<key>` |

### 实际应用

```java
// 监听用户资料的字段级变更
MessageListener listener = (message, pattern) -> {
    String channel = new String(message.getChannel());
    String body = new String(message.getBody());
    // channel: __keyspace@0__:user:123
    // body: hset:email
    if (body.startsWith("hset:")) {
        String field = body.substring(5);
        String newValue = redis.opsForHash().get("user:123", field).toString();
        // 触发索引更新、缓存刷新等
    }
};

redis.subscribe(listener, "__keyspace@0__:user:123".getBytes());
```

## 五、其他改进

### 1. ZUNION/ZINTER COUNT 聚合器

```bash
# 返回交集的前 10 个元素
ZINTER 2 set1 set2 AGGREGATE MAX COUNT 10
```

### 2. TS.RANGE 多聚合器

```bash
# 单次查询返回多个聚合结果
TS.RANGE temperature:room1 0 -1
  AGGREGATION avg 60000
  AGGREGATION max 60000
  AGGREGATION min 60000
```

### 3. JSON.SET FPHA 参数

```bash
# 为同构浮点数组指定 FP 类型
JSON.SET doc $ FPHA float16 '[1.5, 2.5, 3.5]'
```

### 4. FT.HYBRID KNN 优化

```bash
# 减少每个分片的候选 KNN 数量
FT.HYBRID idx "*=>[KNN 10 @vec $BLOB]" PARAMS 2 BLOB \x00...
  KNN_DOC_LIMIT 100  # 每个分片最多 100 个候选
```

## 升级注意事项

1. **8.x → 8.8 兼容**：向前兼容，直接升级即可
2. **Array 命令需要新客户端**：旧版客户端（如 Jedis < 6.0）不支持 `ARRAY.*` 命令
3. **INCREX 替代方案**：如果不需要原子性，可继续使用 `INCR + EXPIRE` 组合
4. **XNACK 权限**：需要流的写权限才能执行 `XNACK`
5. **安全更新**：8.8.0 修复了多个 CVE（UAF、Lua UAF、RESTORE RCE 等），建议尽快升级
