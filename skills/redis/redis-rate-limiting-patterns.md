---
name: redis-rate-limiting-patterns
description: Redis 分布式限流模式对比——INCREX、Lua 脚本、Redisson RRateLimiter、滑动窗口、令牌桶
tags: [redis, rate-limit, pattern, incres, redisson, sliding-window, token-bucket]
---

## 概述

Redis 是分布式限流最常用的基础设施。本文对比 5 种主流限流模式的实现原理、适用场景和注意事项，帮助你在不同业务场景中选择合适的方案。

## 限流模式对比

| 模式 | 原子性 | 精度 | 复杂度 | 适用场景 |
|------|--------|------|--------|---------|
| **INCREX**（Redis 8.8+）| ⭐⭐⭐⭐⭐ | 窗口 | ⭐ | API 限流、简单频控 |
| **Lua 脚本** | ⭐⭐⭐⭐⭐ | 精确 | ⭐⭐⭐ | 滑动窗口、复杂逻辑 |
| **Redisson RRateLimiter** | ⭐⭐⭐⭐⭐ | 令牌桶 | ⭐⭐ | 平滑限流、突发控制 |
| **滑动窗口 ZSET** | ⭐⭐⭐⭐ | 精确 | ⭐⭐⭐ | 严格速率控制 |
| **Redis Cell (GCRA)** | ⭐⭐⭐⭐⭐ | 精确 | ⭐⭐ | 通用限速（需要模块） |

## 一、INCREX 模式（Redis 8.8+）

Redis 8.8 引入的 `INCREX` 是**最简单高效的限流方案**，将计数、过期、边界检查合并为一个原子命令。

### 固定窗口限流

```bash
# 每分钟最多 100 次请求
INCREX rate:api:user_42 NX EX 60
```

### Java 实现

```java
@Service
public class IncrexRateLimiter {

    @Autowired
    private StringRedisTemplate redis;

    /**
     * 固定窗口限流
     * @param key    限流 key，如 "rate:api:user_42"
     * @param limit  窗口内最大请求数
     * @param window 窗口大小（秒）
     * @return true=允许, false=已限流
     */
    public boolean allow(String key, int limit, long window) {
        String result = (String) redis.execute(
            (RedisCallback<Object>) conn -> conn.execute(
                "INCREX", key.getBytes(),
                "NX".getBytes(),                             // 首次创建
                "EX".getBytes(), String.valueOf(window).getBytes()
            )
        );
        if (result == null) return false;
        return Long.parseLong(result) <= limit;
    }
}
```

### 优点

- **一行命令完成**：无需 Lua、无需事务
- **完全原子**：无竞态条件
- **零客户端逻辑**：服务端自包含

### 缺点

- **固定窗口问题**：窗口边界可能产生突发流量（如 59s-01s 之间翻倍）
- **需要 Redis 8.8+**：旧版本无法使用

## 二、Lua 脚本模式（滑动窗口）

Lua 脚本允许实现**精确的滑动窗口限流**，不受固定窗口边界问题影响。

### 滑动窗口 Lua 脚本

```lua
-- slide_window_rate_limit.lua
-- KEYS[1]: 限流 key
-- ARGV[1]: 窗口大小（毫秒）
-- ARGV[2]: 最大请求数
-- ARGV[3]: 当前时间戳（毫秒）

local window = tonumber(ARGV[1])
local limit = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

-- 清理窗口外的过期数据
redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, now - window)

-- 获取当前窗口请求数
local count = redis.call('ZCARD', KEYS[1])

if count >= limit then
    return 0  -- 拒绝
end

-- 记录本次请求
redis.call('ZADD', KEYS[1], now, now .. ':' .. math.random())
redis.call('EXPIRE', KEYS[1], window / 1000 + 1)

return 1  -- 允许
```

### Java 调用 Lua 脚本

```java
@Component
public class SlidingWindowRateLimiter {

    @Autowired
    private RedisTemplate<String, String> redis;

    private static final String SCRIPT = """
        local window = tonumber(ARGV[1])
        local limit = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])
        redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, now - window)
        local count = redis.call('ZCARD', KEYS[1])
        if count >= limit then return 0 end
        redis.call('ZADD', KEYS[1], now, now .. ':' .. math.random())
        redis.call('EXPIRE', KEYS[1], window / 1000 + 1)
        return 1
    """;

    private final RedisScript<Long> slideScript = new DefaultRedisScript<>(SCRIPT, Long.class);

    public boolean allow(String key, int limit, long windowMs) {
        Long result = redis.execute(slideScript,
            List.of(key),
            String.valueOf(windowMs),
            String.valueOf(limit),
            String.valueOf(System.currentTimeMillis())
        );
        return result != null && result == 1;
    }
}
```

### 优点

- **精确滑动窗口**：无边界突发问题
- **完全原子**：Lua 在 Redis 单线程中执行
- **兼容 Redis 2.6+**

### 缺点

- **内存开销大**：每个请求需要在 ZSET 中存一条记录
- **高并发下 ZREMRANGEBYSCORE 有 O(logN) 成本**

## 三、Redisson RRateLimiter（令牌桶）

Redisson 提供了基于令牌桶的 `RRateLimiter`，支持平滑限流 + 突发处理。

### Maven 依赖

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.44.0</version>
</dependency>
```

### 基本使用

```java
@Configuration
public class RateLimitConfig {

    @Autowired
    private RedissonClient redisson;

    @Bean
    public RRateLimiter apiRateLimiter() {
        RRateLimiter limiter = redisson.getRateLimiter("rate:api:global");
        // 速率：每秒 100 个令牌，突发容量 200
        limiter.trySetRate(RateType.OVERALL, 100, 1, RateIntervalUnit.SECONDS);
        return limiter;
    }
}

@Service
public class TokenBucketService {

    @Autowired
    private RRateLimiter apiRateLimiter;

    public boolean tryAcquire() {
        // 尝试获取 1 个令牌
        return apiRateLimiter.tryAcquire();
    }

    public boolean tryAcquire(int permits, long timeout, TimeUnit unit) {
        // 尝试在超时内获取指定数量的令牌
        return apiRateLimiter.tryAcquire(permits, timeout, unit);
    }
}
```

### 每客户端独立限流

```java
// 用户级别限流（按 userId 分桶）
public boolean allowUser(String userId) {
    RRateLimiter userLimiter = redisson.getRateLimiter("rate:user:" + userId);
    userLimiter.trySetRate(RateType.PER_CLIENT, 5, 1, RateIntervalUnit.SECONDS);
    return userLimiter.tryAcquire();
}
```

### 优点

- **支持突发**：令牌桶可积累令牌应对突发流量
- **两种模式**：`OVERALL`（全局）和 `PER_CLIENT`（每个客户端）
- **内置持久化**：重启后限流状态不丢失

### 缺点

- **强依赖 Redisson**：非标准 Redis 命令，客户端绑定
- **配置复杂**：`trySetRate` 只在初始化时有效，需提前规划

## 四、Redis Cell (GCRA) 模式

需要加载 `redis-cell` 模块（或使用 Redis 8.x 内置），实现**通用细胞速率算法（GCRA）**。

```bash
# 安装模块
git clone https://github.com/brandur/redis-cell.git
# 加载到 Redis
redis-server --loadmodule /path/to/libredis_cell.so
```

### 命令使用

```bash
# CL.THROTTLE <key> <max_burst> <tokens_per_interval> <interval> [capacity]
# 每分钟最多 30 个请求，突发 10
CL.THROTTLE user:api:42 10 30 60

# 返回数组（允许数，总限制，剩余配额，重试秒数，重试毫秒数）
# 1) (integer) 0        # 0=允许, 1=限流
# 2) (integer) 11       # 总限制
# 3) (integer) 10       # 剩余
# 4) (integer) -1       # 重试时间（秒，-1表示无需等待）
# 5) (integer) 0        # 重试时间（毫秒）
```

### Spring Boot 集成

```java
@Service
public class GcraRateLimiter {

    @Autowired
    private StringRedisTemplate redis;

    public boolean allow(String key, int maxBurst, int count, int period) {
        List<Object> results = redis.execute(
            (RedisCallback<List<Object>>) conn -> {
                List<Object> raw = conn.execute(
                    "CL.THROTTLE", key.getBytes(),
                    String.valueOf(maxBurst).getBytes(),
                    String.valueOf(count).getBytes(),
                    String.valueOf(period).getBytes()
                );
                // 第一个返回值：0=允许，1=拒绝
                return List.of(raw.get(0));
            }
        );
        return results != null && "0".equals(results.get(0).toString());
    }
}
```

### 优点

- **最精确的通用限速算法**
- **一行命令完成**
- **支持突发和稳定速率分离**

### 缺点

- **需要额外模块**：Redis 8.x 内置，旧版本需手动加载
- **社区活跃度波动**：redis-cell 维护不如其他方案活跃

## 五、场景选型指南

| 业务场景 | 推荐方案 | 原因 |
|---------|---------|------|
| API 网关统一限流 | **INCREX** | 简单高效，无需额外逻辑 |
| 用户登录频控（精确） | **Lua 滑动窗口** | 窗口精确，防止边界绕过 |
| 短信发送限流 | **Redisson RRateLimiter** | 令牌桶 + 突发控制 |
| 文件上传限速（根据文件大小） | **Lua 自定义** | 灵活控制每次消耗的配额 |
| 分布式全局限流 | **Redis Cell / INCREX** | 全局协同，低延迟 |
| 多级限流（用户 + IP + 全局） | **组合使用** | 不同层级用不同方案 |

## 注意事项

### 1. 固定窗口 vs 滑动窗口

```
固定窗口: [00:00 - 00:01] 100次 → [00:01 - 00:02] 100次
          如果在 00:00:59 时用了 100 次，00:01:01 又用 100 次
          实际上 2 秒内通过了 200 次请求！

滑动窗口: 任何时候的过去 1 分钟内都不会超过 100 次
```

### 2. 限流 key 的设计

```java
// ✅ 好的设计：包含用户/IP/接口等维度
String key = "rate:" + apiName + ":" + userId;

// ❌ 避免：粒度太粗或太细
String key = "rate:api";               // 太粗：所有用户共享
String key = "rate:" + UUID.random();  // 太细：每个请求不同，限流失效
```

### 3. 异常降级

```java
public boolean allowWithFallback(String key, int limit, long window) {
    try {
        return allow(key, limit, window);
    } catch (Exception e) {
        // Redis 不可用时，降级为本地限流或放行
        log.warn("Rate limiter unavailable, falling back: {}", e.getMessage());
        return true; // 或使用 Guava RateLimiter 本地限流
    }
}
```

### 4. 测试要点

- 用 `concurrent` 模拟高并发测试原子性
- 验证窗口边界是否严格
- 测试 Redis 重启后限流状态恢复
- 测试 Redis 宕机时的降级策略
