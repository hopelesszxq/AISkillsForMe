---
name: redis-llm-cache-two-layer
description: Redis 分层缓存策略——精确匹配缓存（L1 Redis）+语义缓存（L2 向量数据库）为 AI API 调用降本增效
tags: [redis, llm, cache, semantic-cache, vector-database, ai, performance]
---

## 概述

生产级 AI API 缓存需要**两层**，而非一层：

| 层级 | 用途 | 命中率 | 延迟 | 适合技术 |
|------|------|--------|------|---------|
| L1 精确匹配 | SHA-256 哈希，字节精确匹配 | 5-15% | 亚毫秒级 | **Redis** |
| L2 语义缓存 | Embedding 相似度搜索 | 25-50% | 20-40ms | 向量数据库（pgvector、Pinecone 等） |

两层叠加，总命中率可达 **30-65%**，显著降低 LLM API 调用成本。

> 核心原则：**不是 "Redis 还是向量数据库？"，而是 "哪一层缓存用哪个后端？"**
> Redis 服务 L1 精确匹配，向量数据库服务 L2 语义缓存——两者不可互相替代。

## 一、L1 精确匹配缓存（Redis）

### 工作原理

用确定性指纹（请求的 SHA-256 哈希）作为键，存储完整响应：

```
请求 → 哈希(fingerprint) → Redis GET → 命中则直接返回
                                   → 未命中 → 调用 LLM → Redis SET → 返回
```

### 代码示例

```python
import hashlib
import json
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def get_cached_response(messages, model="gpt-4"):
    """两层缓存：先 Redis 精确匹配，再向量语义匹配"""
    # L1: Redis 精确匹配
    fingerprint = hashlib.sha256(
        json.dumps({"messages": messages, "model": model}, sort_keys=True).encode()
    ).hexdigest()

    cached = r.get(f"llm:exact:{fingerprint}")
    if cached:
        r.incr(f"stats:exact:hits")
        return json.loads(cached)

    # L2: 语义缓存（调用向量数据库）
    semantic = semantic_cache_lookup(messages)
    if semantic:
        r.incr(f"stats:semantic:hits")
        return semantic

    # 未命中，调用 LLM
    response = call_llm(messages, model)

    # 写入缓存
    r.setex(f"llm:exact:{fingerprint}", 3600, json.dumps(response))
    r.incr(f"stats:exact:misses")
    return response
```

### Redis 配置选择

| 方案 | 优势 | 适用场景 |
|------|------|---------|
| Redis Cloud (Redis Inc.) | 托管、99.9% SLA | 生产首选 |
| Upstash Redis | Serverless、REST API | 中低流量、避免连接池管理 |
| ElastiCache / Memorystore | 云原生、大规模便宜 | 单云部署 |
| 自建 Redis | 成本最低 | 有运维能力的小团队 |
| KeyDB / DragonflyDB | 多线程、高吞吐 | 高 QPS 场景 |

### 容量估算

- 每条 LLM 响应约 **500-2000 bytes**
- 10 万条缓存 ≈ 100MB，100 万条 ≈ 1GB
- 10 万请求/天 ≈ 1.2 ops/sec（Redis 轻松处理）
- 成本：~$10-30/月（5GB 托管 Redis）

### TTL 与失效策略

```python
# 推荐：TTL 驱动失效，不主动清理
r.setex(f"llm:exact:{fingerprint}", 3600, json.dumps(response))

# LRU eviction（内存满时）
r.config_set("maxmemory-policy", "allkeys-lru")

# 统计跟踪
r.incr(f"stats:{metric}")
r.expire(f"stats:{metric}", 86400)  # 24h 过期自动清理
```

## 二、L2 语义缓存（向量数据库）

### 工作原理

将用户 prompt 用句子嵌入模型（如 `text-embedding-3-small`）转成向量，在向量索引中搜索最接近的已缓存 prompt：

```
请求 → Embedding(model) → 向量搜索 → 相似度 > 阈值 → 返回缓存响应
                                     → 相似度 < 阈值 → 调用 LLM → 存入向量库
```

### 关键参数

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| Embedding 模型 | `text-embedding-3-small` | 低延迟、高性价比 |
| 相似度阈值 | 0.85-0.95 | 根据业务精确度要求调整 |
| 索引类型 | HNSW | 延迟与召回率的最佳平衡 |
| Top-K | 1-3 | 取最相似结果 |

### 向量数据库选择

| 方案 | 延迟 | 成本特征 | 适用场景 |
|------|------|---------|---------|
| **pgvector** | ~30ms | 零额外基础设施 | 已有 PostgreSQL、中等规模 |
| Pinecone | ~25ms | 按存储量计费 | 高可靠性要求 |
| Weaviate / Qdrant | ~20-40ms | 自建/托管可选 | 定制需求 |

### 代码示例（pgvector）

```sql
-- 建表
CREATE TABLE llm_cache (
    id BIGSERIAL PRIMARY KEY,
    prompt_hash TEXT NOT NULL,
    prompt_text TEXT NOT NULL,
    embedding vector(1536),  -- text-embedding-3-small 维度
    response JSONB NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- HNSW 索引
CREATE INDEX idx_llm_cache_embedding ON llm_cache
    USING hnsw (embedding vector_cosine_ops);
```

```python
import numpy as np
from openai import OpenAI

client = OpenAI()

def get_embedding(text: str) -> list[float]:
    resp = client.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return resp.data[0].embedding

def semantic_cache_lookup(messages, threshold=0.92):
    """语义缓存查询"""
    prompt = extract_user_prompt(messages)
    embedding = get_embedding(prompt)

    # pgvector 查询（余弦相似度）
    result = db.query("""
        SELECT response, 1 - (embedding <=> :embedding) AS similarity
        FROM llm_cache
        WHERE 1 - (embedding <=> :embedding) > :threshold
        ORDER BY similarity DESC
        LIMIT 1
    """, {"embedding": embedding, "threshold": threshold})

    return result[0].response if result else None
```

## 三、延迟与成本模型

### 延迟对比

| 场景 | p95 延迟 |
|------|---------|
| Redis 精确匹配命中 | **< 8ms** |
| 向量语义缓存命中 | **20-40ms**（含 Embedding） |
| LLM 原始调用 | **500ms-5s** |

### 成本节省公式

```
每月节省 = (命中数 × 每次 LLM 调用成本) - 缓存基础设施成本
```

| 流量级别 | 无缓存成本 | 两层缓存成本 | 节省 |
|---------|-----------|-------------|------|
| 10万次/月 | $1,500 | ~$50 | **97%** |
| 100万次/月 | $15,000 | ~$500 | **97%** |
| 1000万次/月 | $150,000 | ~$5,000 | **97%** |

> 假设：GPT-4 $0.03/1K input + $0.06/1K output，平均每次 ~$0.015

## 四、决策框架

```
需要缓存 AI API 调用？
├── 字节精确匹配就够了？→ 只用 Redis（Layer 1）
├── 用户会改写同一意图的问题？
│   ├── 已有 PostgreSQL？→ pgvector（Layer 2）
│   └── 没有 PostgreSQL？→ Pinecone / Upstash Vector
└── 两者都需要？→ Redis + 向量数据库双层缓存
```

## 五、注意事项

### 坑点与避坑

1. **不要只做语义缓存不做精确匹配**：精确匹配命中虽低但零 Embedding 延迟，应先短路
2. **TTL 驱动失效优于主动清除**：AI 缓存天然可容忍过期数据，不主动做精确的逐出逻辑
3. **Embedding 延迟不可忽略**：每次语义查询需 ~10-20ms Embedding 推理，用异步批处理优化
4. **相似度阈值要调优**：阈值太低 → 幻觉（返回错误答案），阈值太高 → 命中率下降
5. **Redis 大 Key 问题**：LLM 响应可能超过 10KB，避免在 Redis 中存储超大 value，考虑压缩

### 生产级监控指标

```python
# 需要监控的关键指标
METRICS = [
    "exact:hit_rate",      # = exact:hits / (exact:hits + exact:misses)
    "semantic:hit_rate",   # = semantic:hits / total_requests
    "cache_savings_usd",   # 成本节省金额
    "avg_latency_ms",      # 平均查询延迟
    "cache_size_mb",       # Redis 内存用量
]
```

### 不适合的场景

- **实时性极高**（秒级数据更新）：缓存带来的延迟节省不如数据新鲜度重要
- **每个请求都不同**（个性化推荐）：命中率极低，缓存收益小
- **小流量**（< 1000次/天）：缓存基础设施成本 > LLM 调用成本
