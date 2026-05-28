---
name: redis8-features
description: Redis 8.0 新特性：Vector Set 向量相似搜索、内置查询引擎、JSON 原生支持、新 Hash 命令
tags: [redis, vector, search, json, performance, ai]
---

## 概述

Redis 8.0（2025年正式发布）是近年来最大的一次版本升级。核心变化是将 RediSearch、RedisJSON、RedisTimeSeries、RedisBloom 等模块**原生内置到 Redis 内核**，无需额外加载模块。新增 Vector Set 数据结构，使 Redis 可直接用于 AI 向量检索场景。

## 架构变化

### 模块内置化

```
Redis 7.x                    Redis 8.0
─────────────                ─────────────
Redis Core                   Redis Core (一体化)
  ├ RediSearch (模块)    →     ├ Query Engine (内置)
  ├ RedisJSON (模块)     →     ├ JSON 数据类型 (内置)
  ├ RedisTimeSeries (模块)→    ├ Time Series (内置)
  ├ RedisBloom (模块)    →     ├ Bloom/Cuckoo/Count-Min/TopK (内置)
  └ 各自独立版本号            └ 统一版本号 8.0.x
```

### 新 ACL 分类

```bash
# Redis 8.0 新增 ACL 类别
@search      # 搜索命令 (FT.SEARCH, FT.AGGREGATE 等)
@json        # JSON 命令 (JSON.SET, JSON.GET 等)
@timeseries  # 时序命令 (TS.ADD, TS.RANGE 等)
@bloom       # 布隆过滤器 (BF.ADD, BF.EXISTS 等)
@cuckoo      # 布谷鸟过滤器 (CF.ADD 等)
@cms         # Count-Min Sketch (CMS.INCRBY 等)
@topk        # Top-K (TOPK.ADD 等)
@tdigest     # T-Digest 分位数 (TDIGEST.ADD 等)
```

```bash
# 示例：只允许执行搜索和 JSON 命令
ACL SETUSER myapp on >password ~* +@search +@json -@all
```

## Vector Set（向量集合）[Beta]

### 核心概念

Vector Set 是 Redis 8.0 新增的核心数据结构，基于 Sorted Set 的设计理念扩展而来，用于存储和查询**高维向量嵌入**（embeddings）。

```bash
# 创建向量集合
VS.CREATE myvectors TYPE FLOAT64 DIM 768 DISTANCE_METRIC COSINE

# 添加向量元素
VS.ADD myvectors "doc:001" 0.12 0.34 0.56 ...  # 768 维向量

# 向量相似搜索（返回最近邻）
VSIM myvectors 0.11 0.33 0.55 ... COUNT 10 WITHSCORES
```

### 主要命令

| 命令 | 功能 |
|------|------|
| `VS.CREATE` | 创建 Vector Set，指定维度、距离度量 |
| `VS.ADD` | 添加向量元素（可同时指定 score） |
| `VS.REM` | 删除向量元素 |
| `VS.SIM` (VSIM) | 向量相似搜索，支持 EPSILON 距离阈值 |
| `VS.CARD` | 返回集合基数 |
| `VS.RANGE` | 按 score 范围查询 |

### Java 客户端使用（Jedis / Lettuce）

```java
// Jedis 8.x 操作 Vector Set
try (Jedis jedis = new Jedis("localhost", 6379)) {
    // 创建向量集合
    jedis.sendCommand(Protocol.Command.valueOf("VS.CREATE"),
        "myvectors", "TYPE", "FLOAT64", "DIM", "4", 
        "DISTANCE_METRIC", "COSINE");
    
    // 添加向量
    jedis.sendCommand(Protocol.Command.valueOf("VS.ADD"),
        "myvectors", "vec:1", "0.1", "0.2", "0.3", "0.4");
    
    // 相似搜索
    List<String> results = jedis.sendCommand(
        Protocol.Command.valueOf("VSIM"),
        "myvectors", "0.1", "0.2", "0.3", "0.4",
        "COUNT", "10", "WITHSCORES");
}
```

### 使用场景

```java
// AI 语义搜索：将 OpenAI/通义千问 的 Embedding 存入 Redis
public class SemanticSearchService {
    
    private final JedisPool jedisPool;
    
    // 存入文档向量
    public void indexDocument(String docId, float[] embedding) {
        try (Jedis jedis = jedisPool.getResource()) {
            String[] args = new String[2 + embedding.length];
            args[0] = "myvectors";
            args[1] = docId;
            for (int i = 0; i < embedding.length; i++) {
                args[2 + i] = String.valueOf(embedding[i]);
            }
            jedis.sendCommand(Protocol.Command.valueOf("VS.ADD"), args);
        }
    }
    
    // 语义搜索 topK
    public List<String> search(float[] queryVector, int topK) {
        try (Jedis jedis = jedisPool.getResource()) {
            String[] args = new String[3 + queryVector.length];
            args[0] = "myvectors";
            for (int i = 0; i < queryVector.length; i++) {
                args[1 + i] = String.valueOf(queryVector[i]);
            }
            args[1 + queryVector.length] = "COUNT";
            args[2 + queryVector.length] = String.valueOf(topK);
            args[3 + queryVector.length] = "WITHSCORES";
            
            return jedis.sendCommand(
                Protocol.Command.valueOf("VSIM"), args);
        }
    }
}
```

## 内置查询引擎（Query Engine）

Redis 8.0 将 RediSearch 全面内置，支持全文搜索、向量搜索、聚合查询。

### 全文搜索

```bash
# 创建索引（内置，无需 FT.CREATE 加载模块）
FT.CREATE idx:products ON HASH PREFIX 1 "product:" SCHEMA
    name TEXT WEIGHT 5.0
    description TEXT
    price NUMERIC SORTABLE
    category TAG

# 搜索
FT.SEARCH idx:products "@name:手机 @price:[2000 5000]"

# 聚合：按分类统计
FT.AGGREGATE idx:products "*"
    GROUPBY 1 @category
    REDUCE COUNT 0 AS count
    SORTBY 2 @count DESC
```

### 向量搜索（Query Engine 方式）

```bash
# 在索引中定义向量字段
FT.CREATE idx:embeds ON HASH PREFIX 1 "embed:" SCHEMA
    content TEXT
    embedding VECTOR FLAT 6 TYPE FLOAT32 DIM 768 DISTANCE_METRIC COSINE

# 向量搜索（KNN）
FT.SEARCH idx:embeds "*=>[KNN 10 @embedding $vec AS score]"
    PARAMS 2 vec "0.12,0.34,...,0.56"
    SORTBY score
    RETURN 3 content score
```

## 新的 Hash 命令

Redis 8.0 新增 3 个 Hash 操作命令，减少网络往返。

```bash
# HGETDEL — 获取并删除字段（原子操作）
HGETDEL session:user:1001 last_access
# 原来需要: HGET + HDEL（2次网络往返）

# HGETEX — 获取字段并设置过期时间（TTL）
HGETEX cache:config:1001 theme EX 3600
# 获取 theme 值，同时刷新 TTL = 3600s

# HSETEX — 设置字段值并带过期时间
HSETEX cache:otp:18812345678 code 123456 EX 300
# 设置验证码，5分钟自动过期（替代 SET + EXPIRE）
```

### Java 中使用新 Hash 命令

```java
// Spring Data Redis — 通过 RedisTemplate.execute 调用
redisTemplate.execute((RedisCallback<String>) conn -> {
    // HGETEX: 获取并设置过期时间
    byte[] result = conn.execute("HGETEX".getBytes(),
        "cache:user:1001".getBytes(),
        "profile".getBytes(),
        "EX".getBytes(),
        "3600".getBytes());
    
    // HGETDEL: 获取并删除
    byte[] delResult = conn.execute("HGETDEL".getBytes(),
        "session:user:1001".getBytes(),
        "token".getBytes());
    
    return result != null ? new String(result) : null;
});
```

## 集群兼容性检测

```bash
# Redis 8.0 新增：检测应用是否兼容集群模式
# 在单机模式下，跟踪所有使用了跨 slot 操作的命令

# 查看不兼容命令统计
INFO stats
# cluster_incompatible_ops: 42
# cluster_incompatible_commands: "MGET,MSET,EVAL"

# 设置采样率（默认 10%）
CONFIG SET cluster-compatibility-sample-ratio 50
```

```java
// 在迁移到集群前，先验证兼容性
// 单机模式运行一段时间，查看 cluster_incompatible_ops
// 如果为 0，放心切换到集群模式
```

## 性能与运维改进

### RDB 通道复制

```bash
# Redis 8.0 支持 RDB 传输与复制通道分离
# 复制过程中 replica 可更快上线
replica-priority 100
rdb-channel-replication yes  # 8.0 新增，默认开启
```

### 内存碎片整理增强

```bash
# 8.0 增强了 defrag 对复杂类型的处理
# 支持对 Hash TTL 过期字段的碎片整理
ACTIVE DEFRAG THRESHOLD 15  # 碎片率超过 15% 开始整理
```

## 升级注意事项

1. **模块兼容性**：如果之前使用了 RediSearch/RedisJSON 等模块，升级到 8.0 后**无需再加载模块**，但需检查客户端版本是否支持新命令
2. **客户端升级**：Jedis 5.x+ / Lettuce 6.4+ 开始支持 Redis 8.0 新命令
3. **RDB 兼容性**：8.0 可读取 7.x 的 RDB 文件，但 8.0 写入的 RDB 无法降级读取
4. **Vector Set Beta**：当前为 Beta 状态，生产环境谨慎使用
5. **ACL 迁移**：如果之前用 ACL，升级后需要为 `@search`, `@json` 等新分类设置权限
6. **内存开销**：内置模块会增加基础内存占用（约 30-50MB），小内存实例注意评估
7. **Spring Data Redis 兼容**：截至 2026年5月，Spring Data Redis 3.4+ 已支持 Redis 8.0 新命令
