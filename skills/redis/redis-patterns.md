---
name: redis-patterns
description: Redis 缓存与分布式锁实践
tags: [redis, cache, lock]
---

## 缓存策略
- Redis + 本地缓存（Caffeine）多级缓存
- 缓存穿透：布隆过滤器 + 缓存空值
- 缓存雪崩：过期时间加随机值

## 分布式锁
- Redisson 实现可重入锁
- 看门狗机制自动续期
- Lua 脚本保证原子性
