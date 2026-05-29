---
name: pg-vector-search
description: PostgreSQL + pgvector 向量检索实战：索引选择、性能调优与混合搜索
tags: [postgresql, pgvector, vector, ai, embedding, search, hsnw]
---

## 概述

pgvector 是 PostgreSQL 最流行的向量搜索扩展，适用于 RAG（检索增强生成）、语义搜索、推荐系统等 AI 场景。本文聚焦 pgvector 在生产环境中的索引选型、查询优化和高级用法。

## 一、安装与基础

```sql
-- 安装扩展
CREATE EXTENSION vector;

-- 创建向量表
CREATE TABLE documents (
    id        BIGSERIAL PRIMARY KEY,
    content   TEXT,
    embedding VECTOR(1536)  -- OpenAI text-embedding-3-small 维度
);

-- 插入数据
INSERT INTO documents (content, embedding)
VALUES ('...', '[0.001, 0.002, ...]'::vector);
```

## 二、索引策略对比

pgvector 1.12+ 支持三种索引类型：

| 索引类型 | 搜索模式 | 适合场景 | 构建速度 | 精度 |
|---------|---------|---------|---------|------|
| IVFFlat | 近似搜索 (ANN) | 快速构建、小到中型数据集 | ⚡ 快 | 中 |
| HNSW | 近似搜索 (ANN) | 高精度、大数据集 | 🐢 慢 | 高 |
| 全扫描 (无索引) | 精确搜索 (KNN) | 小数据集、精度要求极高 | - | 100% |

### HNSW 索引（推荐，pgvector 0.7.0+）

```sql
-- HNSW: 高召回率，适合大部分生产场景
CREATE INDEX ON documents 
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 200);

-- 参数说明
--   m: 每层最大连接数（默认 16，越大精度越高，构建越慢）
--   ef_construction: 动态列表大小（默认 64，越大构建越慢但精度越高）
```

### IVFFlat 索引（适合快速实验）

```sql
-- IVFFlat: 构建快，但精度低于 HNSW
CREATE INDEX ON documents 
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
-- lists ≈ sqrt(rows)
-- 100 万行 → lists = 1000
```

## 三、距离度量选择

```sql
-- L2 距离（欧氏距离）
SELECT * FROM documents
ORDER BY embedding <-> '[0.001, ...]'::vector
LIMIT 10;

-- 余弦距离（文本语义搜索最常用）
SELECT * FROM documents
ORDER BY embedding <=> '[0.001, ...]'::vector
LIMIT 10;

-- 内积距离
SELECT * FROM documents
ORDER BY embedding <#> '[0.001, ...]'::vector
LIMIT 10;
```

### 索引操作符对应关系

| 距离函数 | 距离越小越... | HNSW/IVFFlat 操作符 |
|---------|-------------|-------------------|
| L2 (`<->`) | 越近 | `vector_l2_ops` |
| 余弦 (`<=>`) | 越相似 | `vector_cosine_ops` |
| 内积 (`<#>`) | 越相关 | `vector_ip_ops` |

**最佳实践**：文本语义搜索用 `<=>`（余弦距离），图像/结构数据用 `<->`（L2 距离）。

## 四、混合搜索（向量 + 关键词）

结合 PostgreSQL 全文搜索和向量搜索实现双路召回：

```sql
-- 创建全文搜索索引
ALTER TABLE documents ADD COLUMN tsv tsvector
GENERATED ALWAYS AS (to_tsvector('english', content)) STORED;

CREATE INDEX documents_tsv_idx ON documents USING GIN (tsv);

-- 混合搜索：向量相似度 + 关键词匹配 + 加权排序
SELECT 
    id,
    content,
    -- 向量相似度分数 (0~1, 1=最相似)
    1 - (embedding <=> '[0.001, ...]'::vector) AS vector_score,
    -- 关键词匹配分数
    ts_rank(tsv, plainto_tsquery('english', 'search term')) AS keyword_score
FROM documents
WHERE 
    -- 关键词过滤（可选）
    tsv @@ plainto_tsquery('english', 'search term')
ORDER BY 
    -- 加权合并排序
    (0.7 * (1 - (embedding <=> '[0.001, ...]'::vector))
     + 0.3 * ts_rank(tsv, plainto_tsquery('english', 'search term'))) DESC
LIMIT 20;
```

### RRF（倒数排序融合）实现

```sql
-- 不使用搜索关键词时纯向量检索
-- 使用搜索关键词时 RRF 合并
WITH vector_results AS (
    SELECT id, ROW_NUMBER() OVER (ORDER BY embedding <=> '[0.001,...]'::vector) AS rank
    FROM documents
    LIMIT 100
),
keyword_results AS (
    SELECT id, ROW_NUMBER() OVER (ORDER BY ts_rank(tsv, query) DESC) AS rank
    FROM documents, plainto_tsquery('english', 'search term') AS query
    WHERE tsv @@ query
    LIMIT 100
)
SELECT 
    COALESCE(v.id, k.id) AS id,
    COALESCE(1.0 / (60 + v.rank), 0.0) + COALESCE(1.0 / (60 + k.rank), 0.0) AS rrf_score
FROM vector_results v
FULL OUTER JOIN keyword_results k ON v.id = k.id
ORDER BY rrf_score DESC
LIMIT 20;
```

## 五、性能调优

### 1. 索引构建优化

```sql
-- HNSW 索引构建调优
SET maintenance_work_mem = '4GB';  -- 增大索引构建内存
SET hnsw.ef_search = 200;         -- 搜索时的动态列表大小
SET hnsw.ef_construction = 400;   -- 构建时的动态列表大小（默认 200）

CREATE INDEX CONCURRENTLY ON documents 
USING hnsw (embedding vector_cosine_ops)
WITH (m = 32, ef_construction = 400);
```

### 2. 分区表向量索引

```sql
-- 按日期分区
CREATE TABLE embeddings (
    id        BIGSERIAL,
    content   TEXT,
    embedding VECTOR(768),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    tenant_id INT NOT NULL
) PARTITION BY LIST (tenant_id);

-- 每个分区独立建 HNSW 索引
CREATE INDEX ON embeddings_part1 USING hnsw (embedding vector_cosine_ops);
CREATE INDEX ON embeddings_part2 USING hnsw (embedding vector_cosine_ops);
-- 查询时自动分区裁剪
```

### 3. Streaming 读取优化

```sql
-- 使用 CURSOR 避免大结果集内存 OOM
BEGIN;
DECLARE vec_cursor CURSOR FOR
    SELECT id, content, 1 - (embedding <=> '[0.001,...]'::vector) AS score
    FROM documents
    ORDER BY embedding <=> '[0.001,...]'::vector
    LIMIT 1000;
FETCH 100 FROM vec_cursor;
-- ... 分批处理
COMMIT;
```

### 4. 半精度量化（pgvector 0.8.0+）

```sql
-- 使用 halfvec 类型，内存占用减半
ALTER TABLE documents ADD COLUMN embedding_hf halfvec(1536);

-- 适用于内存受限场景，精度损失 ~1-3%
INSERT INTO documents (embedding_hf)
SELECT embedding::halfvec(1536) FROM documents;
```

## 六、生产部署注意

```yaml
# postgresql.conf 调优（向量密集场景）
maintenance_work_mem = 4GB        # 索引构建
work_mem = 256MB                   # 排序操作
shared_buffers = 8GB               # 缓存向量数据
effective_cache_size = 24GB        # OS 缓存估计
random_page_cost = 1.1             # SSD 优化
effective_io_concurrency = 200     # 并行 IO
```

## 注意事项

| 要点 | 说明 |
|------|------|
| **维度限制** | pgvector 默认支持最 2000 维，如需更高用编译参数 `--with-max-dim=4000` |
| **索引内存** | HNSW 索引占用约 `dim × 4 × (1 + m) × rows` 字节，1536 维 100 万行约占用 10GB |
| **IVFFlat lists 调优** | `lists = sqrt(rows)` 为经验值，实际需根据数据分布测试 |
| **精确搜索 vs 近似搜索** | HNSW 召回率可达 99%+，精确搜索仅用于小数据集（<1万行）验证 |
| **事务影响** | HNSW 索引对 INSERT 性能有影响（每插入一条需更新索引图），批量导入时建议先删索引再重建 |
| **pgvector 版本** | HNSW 需要 pgvector ≥ 0.7.0，halfvec 需要 ≥ 0.8.0；最新版 0.9.x 持续优化性能 |
| **不适用于** | 高维稀疏向量（推荐用倒排索引）；实时更新频繁的场景（HNSW 增量维护成本高） |
