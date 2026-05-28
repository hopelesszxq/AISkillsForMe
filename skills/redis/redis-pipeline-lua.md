---
name: redis-pipeline-lua
description: Redis Pipeline（管道）与 Lua 脚本深度优化：批量操作、原子性、性能压榨
tags: [redis, pipeline, lua, performance, optimization]
---

## Pipeline 管道技术

Pipeline 将多个命令打包一次发送，**减少网络 RTT（Round Trip Time）**，适合批量操作场景。

### 性能对比

| 场景 | 逐条发送 | Pipeline (50条/批) | 提升倍数 |
|------|---------|-------------------|---------|
| 本地网络 | ~0.3ms/条 | ~5ms/批 | ~3x |
| 跨机房 10ms 延迟 | ~10ms/条 | ~15ms/批 | ~33x |
| 写入 10 万条数据 | ~30s | ~2s | ~15x |

### Spring Boot + Redis Pipeline

```java
@Service
public class UserCacheService {

    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * Pipeline 批量查询
     */
    public List<User> batchGetUsers(List<Long> userIds) {
        List<Object> results = redisTemplate.executePipelined(
            (RedisCallback<Object>) connection -> {
                userIds.forEach(id -> {
                    String key = "user:" + id;
                    // 只添加命令到 pipeline，不立即执行
                    connection.get(key.getBytes());
                });
                return null; // 返回 null 告诉 Spring 使用 pipeline
            }
        );

        return results.stream()
            .map(r -> r == null ? null : JSON.parseObject((String) r, User.class))
            .collect(Collectors.toList());
    }

    /**
     * Pipeline 批量写入（带 TTL）
     */
    public void batchSetUsers(Map<Long, User> userMap) {
        redisTemplate.executePipelined(
            (RedisCallback<Object>) connection -> {
                userMap.forEach((id, user) -> {
                    String key = "user:" + id;
                    byte[] value = JSON.toJSONBytes(user);
                    connection.setEx(key.getBytes(), 3600, value);
                });
                return null;
            }
        );
    }

    /**
     * Pipeline + 事务（MULTI/EXEC）
     */
    public void batchTransfer(String fromKey, String toKey, int amount) {
        redisTemplate.executePipelined(
            (RedisCallback<Object>) connection -> {
                // 开启事务
                connection.multi();
                // 多个操作
                connection.decrBy(fromKey.getBytes(), amount);
                connection.incrBy(toKey.getBytes(), amount);
                // 提交事务
                connection.exec();
                return null;
            }
        );
    }
}
```

### 原生 Jedis Pipeline

```java
public class PipelineBenchmark {

    @Test
    public void comparePipelineVsSingle() {
        try (Jedis jedis = new Jedis("localhost", 6379)) {
            long start = System.currentTimeMillis();

            // ❌ 逐条发送（10000 次 RTT）
            for (int i = 0; i < 10000; i++) {
                jedis.set("key:" + i, String.valueOf(i));
            }
            long singleCost = System.currentTimeMillis() - start;
            System.out.println("逐条发送耗时: " + singleCost + "ms");

            start = System.currentTimeMillis();

            // ✅ Pipeline 批量发送（~1 次 RTT）
            Pipeline pipeline = jedis.pipelined();
            for (int i = 10000; i < 20000; i++) {
                pipeline.set("key:" + i, String.valueOf(i));
            }
            pipeline.sync();                   // 同步发送并获取响应
            long pipelineCost = System.currentTimeMillis() - start;
            System.out.println("Pipeline 耗时: " + pipelineCost + "ms");
        }
    }

    /**
     * Pipeline 批量读取
     */
    public Map<String, String> batchGet(List<String> keys) {
        try (Jedis jedis = new Jedis("localhost", 6379)) {
            Pipeline pipeline = jedis.pipelined();
            List<Response<String>> responses = new ArrayList<>();

            for (String key : keys) {
                responses.add(pipeline.get(key));  // 不阻塞，只记录响应位置
            }
            pipeline.sync();  // 一起发送并接收所有响应

            Map<String, String> result = new HashMap<>(keys.size());
            for (int i = 0; i < keys.size(); i++) {
                String value = responses.get(i).get();  // 从缓冲区读取，不涉及网络
                if (value != null) {
                    result.put(keys.get(i), value);
                }
            }
            return result;
        }
    }
}
```

### Pipeline 使用原则

```java
/**
 * ✅ 适合 Pipeline 的场景
 */
// 1. 大量无关的 SET/GET 操作（批量缓存预热）
List<String> allKeys = scanAllUserKeys();
batchGet(allKeys);

// 2. 批量写入 + 统一 TTL
batchSetUsers(allUsers);

// 3. 计数器批量递增
pipeline.incrBy("counter:day:20260528", 100);

/**
 * ❌ 不适合 Pipeline 的场景
 */
// 1. 后一个命令依赖前一个结果
String val = jedis.get("key1");   // 必须先知道 key1 的值
pipeline.set("key2", val + "_suffix");  // 才能决定 key2 写入什么

// 2. Pipeline 中混合不同数据类型的命令（虽然可以，但增加复杂度）

// 3. 超大批量（一次 Pipeline > 10000 条），建议分页
```

## Lua 脚本

Lua 脚本保证 **原子性**（整个脚本作为一个原子操作执行），且支持复杂逻辑。

### 基础用法

```lua
-- eval.lua
-- 原子地获取并删除
local val = redis.call('GET', KEYS[1])
if val then
    redis.call('DEL', KEYS[1])
end
return val
```

```java
public class LuaScriptService {

    @Autowired
    private StringRedisTemplate redisTemplate;

    private static final DefaultRedisScript<String> GET_AND_DEL_SCRIPT;

    static {
        GET_AND_DEL_SCRIPT = new DefaultRedisScript<>();
        GET_AND_DEL_SCRIPT.setScriptText(
            "local val = redis.call('GET', KEYS[1])\n" +
            "if val then\n" +
            "    redis.call('DEL', KEYS[1])\n" +
            "end\n" +
            "return val"
        );
        GET_AND_DEL_SCRIPT.setResultType(String.class);
    }

    public String getAndDel(String key) {
        return redisTemplate.execute(GET_AND_DEL_SCRIPT,
            Collections.singletonList(key));
    }
}
```

### 限流脚本（滑动窗口）

```lua
-- rate_limit.lua
-- KEYS[1]: 限流 key
-- ARGV[1]: 窗口大小（毫秒）
-- ARGV[2]: 窗口内最大请求数
-- 返回: 1=允许, 0=拒绝

local key = KEYS[1]
local window = tonumber(ARGV[1])
local maxRequests = tonumber(ARGV[2])
local now = redis.call('TIME')[1]  -- Redis 服务器时间（秒级）

-- 计算窗口起始时间
local windowStart = now * 1000 - window

-- 移除窗口外的请求记录
redis.call('ZREMRANGEBYSCORE', key, 0, windowStart)

-- 统计当前窗口内的请求数
local currentCount = redis.call('ZCARD', key)

if currentCount < maxRequests then
    -- 加入当前请求（精确到毫秒）
    local timestamp = now * 1000
    redis.call('ZADD', key, timestamp, timestamp .. ':' .. math.random())
    redis.call('EXPIRE', key, math.ceil(window / 1000) + 1)
    return 1
else
    return 0
end
```

```java
@Component
public class RateLimiter {

    @Autowired
    private StringRedisTemplate redisTemplate;

    private static final DefaultRedisScript<Long> RATE_LIMIT_SCRIPT;

    static {
        RATE_LIMIT_SCRIPT = new DefaultRedisScript<>();
        RATE_LIMIT_SCRIPT.setScriptText(loadScript("rate_limit.lua"));
        RATE_LIMIT_SCRIPT.setResultType(Long.class);
    }

    /**
     * 滑动窗口限流
     * @param key        限流 key（如 "rate_limit:api:/order:user_123"）
     * @param windowMs   窗口大小（毫秒）
     * @param maxRequests 最大请求数
     * @return true=允许请求
     */
    public boolean tryAcquire(String key, long windowMs, int maxRequests) {
        Long result = redisTemplate.execute(
            RATE_LIMIT_SCRIPT,
            Collections.singletonList(key),
            String.valueOf(windowMs),
            String.valueOf(maxRequests)
        );
        return Long.valueOf(1).equals(result);
    }
}
```

### 分布式锁增强版（可重入 + 自动续期）

```lua
-- redis_lock.lua
-- KEYS[1]: 锁 key
-- ARGV[1]: 锁持有者标识
-- ARGV[2]: 锁过期时间（毫秒）
-- 返回: 1=获取成功, 0=获取失败

local key = KEYS[1]
local owner = ARGV[1]
local ttl = tonumber(ARGV[2])

-- 检查当前持有者
local currentOwner = redis.call('GET', key)
if currentOwner == owner then
    -- 可重入：续期锁
    redis.call('PEXPIRE', key, ttl)
    return 1
end

-- 尝试获取锁（SET NX PX）
local result = redis.call('SET', key, owner, 'NX', 'PX', ttl)
if result then
    return 1
end

return 0
```

```lua
-- redis_unlock.lua
-- KEYS[1]: 锁 key
-- ARGV[1]: 锁持有者标识
-- 返回: 1=释放成功, 0=不是持有者（释放失败）

if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
end
return 0
```

### 批量 Hash 操作脚本

```lua
-- batch_hash_update.lua
-- KEYS[1]: Hash key
-- ARGV: 交替的 field1, value1, field2, value2, ...
-- 返回: 成功写入的字段数

local key = KEYS[1]
local count = 0

for i = 1, #ARGV, 2 do
    local field = ARGV[i]
    local value = ARGV[i + 1]
    if field and value then
        redis.call('HSET', key, field, value)
        count = count + 1
    end
end

return count
```

## Pipeline vs Lua 选型

| 对比维度 | Pipeline | Lua 脚本 |
|---------|---------|---------|
| **原子性** | 不保证（除非包装在 MULTI/EXEC 中） | 绝对保证 |
| **网络开销** | 1 次 RTT（无论多少命令） | 1 次 RTT |
| **复杂逻辑** | 不支持条件判断 | 支持 if/else/for 等 |
| **响应处理** | 可独立获取每条命令的响应 | 只能得到脚本最终返回值 |
| **调试难度** | 低（熟悉 Redis 命令即可） | 中（需调试 Lua 语法） |
| **适用场景** | 批量读写无依赖操作 | 需要原子性的复杂操作 |

## 性能压测数据

```bash
# 使用 redis-benchmark 测试 Pipeline 效果

# 逐条 SET（基线）
redis-benchmark -t set -n 100000 -q
# SET: 85000 requests/sec

# Pipeline 批量 SET（每次 50 条）
redis-benchmark -t set -n 100000 -P 50 -q
# SET (Pipeline): 1200000 requests/sec  (14x)
```

## 注意事项

1. **Pipeline 非原子**：Pipeline 只是批量发送，中间某条命令失败不会影响其他命令执行。需要原子性时用 Lua 或 MULTI/EXEC
2. **Pipeline 结果缓存**：`Response.get()` 在 `sync()` 后从本地缓冲区读取，不涉及网络
3. **Lua 脚本超时**：默认脚本最长执行 5 秒（`lua-time-limit` 配置），超时后只阻塞其他命令，不终止脚本
4. **Lua 脚本注册**：生产环境用 `SCRIPT LOAD` + `EVALSHA` 避免每次传输完整脚本
5. **Pipeline 大小**：单次 Pipeline 建议 50-200 条，过大导致 Redis 输出缓冲区压力增大
6. **Lua 随机性**：`redis.call('TIME')` 等随机命令在复制/集群场景行为不同，注意使用 `redis.replicate_commands()`
7. **EVAL vs EVALSHA**：`EVALSHA` 用 SHA1 缓存脚本，避免带宽浪费。但需处理 `NOSCRIPT` 错误回退到 EVAL
