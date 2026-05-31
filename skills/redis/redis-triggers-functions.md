---
name: redis-triggers-functions
description: Redis Triggers and Functions（原 RedisGears v2）事件驱动编程与数据处理实战
tags: [redis, triggers, functions, redisgears, stream, processing]
---

## 概述

Redis Triggers and Functions（原名 RedisGears v2）是 Redis Stack 提供的事件驱动数据处理引擎。它允许在 Redis 服务端运行 Lua / JavaScript（作为 Redis Function）来处理数据变更事件、定时任务和流式处理 — 类似数据库的触发器 + 存储过程。

7.2+ 内置，无需额外部署。

## 一、架构模型

```
┌─────────────────────────────────────┐
│         Client Application           │
└──────────┬──────────────────────────┘
           │ SET / PUBLISH / XADD
           ▼
┌─────────────────────────────────────┐
│          Redis Server                │
├─────────────────────────────────────┤
│  Triggers (Key Space / Stream)       │
│  └─→ Functions (Lua / JS)           │
│      └─→ Actions (Write / Notify)   │
└─────────────────────────────────────┘
```

核心概念：
- **Register**：注册触发器，监听特定事件
- **Function**：处理逻辑（Lua 或 JavaScript）
- **Trigger**：事件来源（Key Space / Stream / Scheduled）
- **Library**：函数的打包单元

## 二、快速开始

### 通过 redis-cli 注册

```bash
# 注册 Keyspace 触发器：当 KEY 以 "order:" 开头时触发
redis-cli> RG.TRIGGER register "order-processor" {
    "event_types": ["set", "expired"],
    "key_prefix": "order:",
    "callback": "processOrder"
}
```

### JavaScript 函数定义

```javascript
// 加载 JavaScript Library
#!js name=order_processor

redis.registerFunction("processOrder", function(client, data) {
    // data.keys[0] 是触发 key
    var orderKey = data.keys[0];
    // 获取当前值
    var orderData = client.call("JSON.GET", orderKey);
    // 执行处理逻辑
    client.call("XADD", "order:events", "*",
        "type", "order_created",
        "data", orderData);
    // 返回处理结果
    return orderKey;
});

// 注册触发器
redis.registerKeySpaceTrigger("order-listener", "", {
    "event_types": ["set", "expired"],
    "key_prefix": "order:"
}, function(client, data) {
    var key = data.keys[0];
    return client.call("EXISTS", key);
});
```

## 三、Lua 函数

```lua
--#!lua name=cache_manager

-- 缓存预热函数
redis.registerFunction("prewarmCache", function(client, args)
    local pattern = args[1] or "user:*"
    local cursor = "0"
    local count = 0

    repeat
        local result = client.call("SCAN", cursor, "MATCH", pattern, "COUNT", 100)
        cursor = result[1]
        local keys = result[2]

        for _, key in ipairs(keys) do
            local ttl = client.call("TTL", key)
            if ttl > 0 and ttl < 300 then
                -- 即将过期的 key，重新加载
                local userId = key:match("user:(%d+)")
                local freshData = fetch_from_db(userId)
                client.call("SET", key, freshData, "EX", 3600)
                count = count + 1
            end
        end
    until cursor == "0"

    return count
end)

-- Stream 触发器：自动聚合
redis.registerStreamTrigger("stream-aggregator", "order:events", {
    "duration": 60000,     -- 1 分钟窗口
    "batch": 100
}, function(client, data)
    local totalAmount = 0
    local totalOrders = 0

    for _, msg in ipairs(data.messages) do
        local payload = msg.payload
        totalOrders = totalOrders + 1
        -- 假设 payload 中包含 amount 字段
        totalAmount = totalAmount + (tonumber(payload["amount"]) or 0)
    end

    -- 写入聚合结果
    local now = client.call("TIME")
    client.call("XADD", "order:stats", "*",
        "window_sec", 60,
        "total_orders", totalOrders,
        "total_amount", totalAmount,
        "avg_amount", totalOrders > 0 and totalAmount / totalOrders or 0
    )
end)
```

## 四、Stream Trigger 实战：实时订单统计

```javascript
//#!js name=order_analytics

// 实时订单统计触发器
redis.registerStreamTrigger("order-stats",
    "order:events",     // 监听的 Stream
    {
        duration: 5000,       // 5 秒窗口
        batch: 50,            // 每次最多处理 50 条
        on_trigger_failure: "discard",
        on_registration_failure: "discard"
    },
    function(client, data) {
        var count = data.messages.length;
        var amounts = [];

        for (var i = 0; i < count; i++) {
            var msg = data.messages[i];
            var amount = parseFloat(msg.payload["amount"] || "0");
            amounts.push(amount);
        }

        // 计算统计
        var total = amounts.reduce(function(a, b) { return a + b; }, 0);
        var avg = count > 0 ? total / count : 0;

        // 写入实时统计 Hash
        client.call("HSET", "stats:orders:realtime",
            "count", count,
            "total", total.toString(),
            "average", avg.toString(),
            "timestamp", Date.now().toString()
        );

        // 超过阈值时告警
        if (total > 10000) {
            client.call("PUBLISH", "channel:alert",
                JSON.stringify({
                    type: "high_value_orders",
                    total: total,
                    count: count
                })
            );
        }

        return count;
    }
);
```

## 五、定时触发器（Scheduled）

```javascript
#!js name=scheduled_tasks

// 每 60 秒运行一次
redis.registerFunction("cleanupExpiredSessions", function(client, data) {
    var cursor = "0";
    var cleaned = 0;
    repeat {
        var result = client.call("SCAN", cursor, "MATCH", "session:*", "COUNT", 500);
        cursor = result[1];
        var keys = result[2];
        for (var i = 0; i < keys.length; i++) {
            var ttl = client.call("TTL", keys[i]);
            if (ttl == -1) {
                // 没有 TTL 的 session key，清理
                client.call("DEL", keys[i]);
                cleaned = cleaned + 1;
            }
        }
    } while (cursor != "0");
    return "cleaned " + cleaned + " expired sessions";
});

// 注册定时任务
redis.registerFunction("syncToDatabase", function(client) {
    // 每 30 秒将 Redis 统计数据同步到数据库
    var keys = client.call("KEYS", "stats:*");
    for (var i = 0; i < keys.length; i++) {
        var statData = client.call("HGETALL", keys[i]);
        // 将 statData 写入数据库...
    }
    return "synced " + keys.length + " stats";
});
```

## 六、调试与监控

```bash
# 查看所有注册的库
redis-cli> RG.FUNCTION LIST

# 查看触发器状态
redis-cli> RG.TRIGGER LIST

# 查看函数详细信息
redis-cli> RG.FUNCTION INFO my_library

# 删除库
redis-cli> RG.FUNCTION DELETE my_library

# 手动调用函数测试
redis-cli> RG.FCALL cache_manager prewarmCache 0 "user:*"

# 查看执行统计
redis-cli> RG.FUNCTION STATS
```

## 七、Java 客户端集成（Jedis / Lettuce）

```java
// Lettuce Redis Functions 调用
public class RedisFunctionClient {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    // 注册 JavaScript 库
    public void registerLibrary(String code) {
        var conn = redisTemplate.getConnectionFactory().getConnection();
        conn.execute("RG.FUNCTION", "LOAD", "REPLACE", code);
    }

    // 调用注册的函数
    public Object callFunction(String libName, String funcName, Object... args) {
        var conn = redisTemplate.getConnectionFactory().getConnection();
        List<String> params = new ArrayList<>();
        params.add(libName);
        params.add(funcName);
        params.add("0");  // key 数量
        for (Object arg : args) {
            params.add(arg.toString());
        }
        return conn.execute("RG.FCALL", params.toArray(new String[0]));
    }

    // 获取统计
    public Map<String, Object> getStats() {
        var conn = redisTemplate.getConnectionFactory().getConnection();
        var result = conn.execute("RG.FUNCTION", "STATS");
        // 解析结果...
        return parseResult(result);
    }
}
```

## 注意事项

| 要点 | 说明 |
|------|------|
| **性能影响** | Trigger 回调在 Redis 主线程执行，不要在回调中做耗时操作（如大量外部调用）|
| **阻塞操作** | JavaScript 函数内 `client.call()` 调用其他 Redis 命令会阻塞，避免嵌套调用过深 |
| **原子性** | Function 执行具有原子性，同一时间一个库只有一个实例在执行 |
| **错误处理** | `on_trigger_failure: "discard"` 可避免触发器异常阻塞 Stream |
| **调试** | 优先使用 `RG.FCALL` 直接调用测试函数，确认逻辑正确后再注册触发器 |
| **版本要求** | Triggers and Functions 需要 Redis Stack 7.2+，社区版 Redis 7.2+ 需额外安装 redisgears2 模块 |
| **持久化** | 注册的 Function 存储在 RDB/AOF 中，重启后自动恢复 |
| **内存开销** | 每个 Trigger 都有内存开销，不要注册不必要的触发器 |
