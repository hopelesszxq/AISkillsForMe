---
name: pgvector-vector-search
description: PostgreSQL pgvector 扩展实战：向量相似度搜索、HNSW/IVFFlat 索引、混合检索、量化与性能调优
tags: [postgresql, pgvector, vector-search, ai, embeddings, similarity-search, hnsw, ivfflat]
---

## 概述

pgvector 是 PostgreSQL 的开源向量相似度搜索扩展（最新 v0.8.2），支持**精确搜索**和**近似最近邻（ANN）搜索**。与 PostgreSQL 的 ACID、PITR、JOIN 等能力深度融合，是 AI/LLM 应用中存储和检索 Embedding 向量的首选方案。

### 核心能力

| 功能 | 说明 |
|------|------|
| 向量类型 | `vector`（2000维）、`halfvec`（半精度/4000维）、`bit`（二进制/64000维）、`sparsevec`（稀疏/1000非零元素） |
| 距离函数 | L2(`<->`)、内积(`<#>`)、余弦(`<=>`)、L1(`<+>`)、汉明(`<~>`)、Jaccard(`<%>`) |
| 索引 | HNSW（高性能）、IVFFlat（快速构建） |
| 缩放 | 二进制量化、半精度索引、子向量索引 |
| 过滤 | B-tree + HNSW 联合、迭代索引扫描、部分索引、分区 |

## 1. 安装与启用

```bash
# Linux/Mac
git clone --branch v0.8.2 https://github.com/pgvector/pgvector.git
cd pgvector
make && make install  # 需要 sudo

# PostgreSQL >= 13
psql -U postgres -d mydb -c "CREATE EXTENSION vector;"
```

## 2. 基础操作

### 建表与插入

```sql
-- 创建向量列（3维）
CREATE TABLE items (
    id bigserial PRIMARY KEY,
    embedding vector(3),
    category_id int,
    metadata jsonb
);

-- 插入向量
INSERT INTO items (embedding, category_id) VALUES
    ('[1,2,3]', 1),
    ('[4,5,6]', 2),
    ('[7,8,9]', 1);

-- COPY 批量加载（推荐）
COPY items (embedding) FROM STDIN WITH (FORMAT BINARY);
```

### 查询

```sql
-- 最近邻搜索（L2距离）
SELECT * FROM items ORDER BY embedding <-> '[3,1,2]' LIMIT 5;

-- 余弦距离（归一化向量推荐）
SELECT * FROM items ORDER BY embedding <=> '[3,1,2]' LIMIT 5;

-- 内积（OpenAI Embedding 推荐，需归一化）
SELECT * FROM items ORDER BY embedding <#> '[3,1,2]' LIMIT 5;

-- 距离阈值过滤
SELECT * FROM items WHERE embedding <-> '[3,1,2]' < 5;

-- 获取相似度分数
SELECT id, 1 - (embedding <=> '[3,1,2]') AS cosine_similarity
FROM items ORDER BY cosine_similarity DESC LIMIT 5;

-- 向量聚合
SELECT category_id, AVG(embedding) FROM items GROUP BY category_id;
```

## 3. 索引策略

### HNSW（推荐，高性能）

HNSW 是多层图索引，查询性能优于 IVFFlat（速度-召回率平衡更好），但构建更慢、内存消耗更大。

```sql
-- L2距离索引
CREATE INDEX ON items USING hnsw (embedding vector_l2_ops)
    WITH (m = 16, ef_construction = 64);

-- 内积索引
CREATE INDEX ON items USING hnsw (embedding vector_ip_ops);

-- 余弦距离索引
CREATE INDEX ON items USING hnsw (embedding vector_cosine_ops);

-- HNSW 查询参数
SET hnsw.ef_search = 100;  -- 越大召回率越高，默认为40
```

### IVFFlat（快速构建）

IVFFlat 将向量分桶，搜索时只查最近似的子集。构建快、内存低，但查询性能逊于 HNSW。

```sql
-- 创建索引（先有数据，再建索引）
CREATE INDEX ON items USING ivfflat (embedding vector_l2_ops)
    WITH (lists = 100);  -- rows/1000（百万行以下）或 sqrt(rows)（百万行以上）

-- IVFFlat 查询参数
SET ivfflat.probes = 10;  -- 越大召回率越高，默认为1
```

### 索引调优黄金法则

| 场景 | 推荐索引 | 理由 |
|------|---------|------|
| 小数据集（<10万行） | 无索引/精确搜索 | 全表扫描足够快 |
| 中等数据（10万~100万） | IVFFlat | 构建快，可接受 |
| 大数据（100万+） | HNSW | 速度-召回率最佳 |
| 高召回率要求 | HNSW + 大 ef_search | 接近精确搜索 |
| 频繁写入 | IVFFlat | 构建开销小 |
| 少量写入、大量查询 | HNSW | 查询性能更好 |

## 4. 过滤查询优化

### 方案一：B-tree 索引过滤（精确）

```sql
-- 在过滤列上建 B-tree 索引
CREATE INDEX ON items (category_id);

-- 精确最近邻搜索（适合筛选率低的场景）
SELECT * FROM items WHERE category_id = 123
ORDER BY embedding <-> '[3,1,2]' LIMIT 5;
```

### 方案二：HNSW + 迭代索引扫描（0.8.0+）

当过滤后仍有足够多匹配时，自动扫描更多索引条目：

```sql
SET hnsw.iterative_scan = strict_order;    -- 严格排序
SET hnsw.iterative_scan = relaxed_order;   -- 宽松排序（更高召回率）
SET hnsw.max_scan_tuples = 20000;          -- 最大扫描条目数
```

### 方案三：部分索引（适合少量固定值）

```sql
CREATE INDEX ON items USING hnsw (embedding vector_l2_ops)
    WHERE (category_id = 123);
```

### 方案四：分区表（适合大量不同值）

```sql
CREATE TABLE items (embedding vector(3), category_id int)
    PARTITION BY LIST(category_id);
```

## 5. 量化与缩放

### 二进制量化（空间节省 32x）

```sql
-- 建二进制量化索引
CREATE INDEX ON items USING hnsw ((binary_quantize(embedding)::bit(3)) bit_hamming_ops);

-- 查询 + 重排序（先粗筛后精排）
SELECT * FROM (
    SELECT * FROM items
    ORDER BY binary_quantize(embedding)::bit(3) <~>
             binary_quantize('[1,-2,3]') LIMIT 20
) AS candidates
ORDER BY embedding <=> '[1,-2,3]' LIMIT 5;
```

### 半精度向量

```sql
-- halfvec 类型（维度翻倍）
CREATE TABLE items (id bigserial, embedding halfvec(3));

-- 半精度索引（索引空间减半）
CREATE INDEX ON items USING hnsw ((embedding::halfvec(3)) halfvec_l2_ops);
```

### 稀疏向量

```sql
-- sparsevec 类型（适合词袋/BOW 特征）
CREATE TABLE items (id bigserial, embedding sparsevec(5));
INSERT INTO items (embedding) VALUES ('{1:1,3:2,5:3}/5');
-- 格式：{index1:value1,index2:value2}/dimensions
```

## 6. 混合搜索（向量 + 全文检索）

```sql
-- 结合 PostgreSQL 全文搜索
CREATE TABLE documents (
    id bigserial,
    content text,
    embedding vector(384),
    textsearch tsvector
);

CREATE INDEX textsearch_idx ON documents USING gin(textsearch);

-- 混合搜索（RRF 融合或 Cross-Encoder 重排序）
SELECT id, content FROM documents, plainto_tsquery('hello search') query
    WHERE textsearch @@ query
    ORDER BY ts_rank_cd(textsearch, query) DESC LIMIT 10;
```

## 7. 性能调优

```sql
-- 1. 增加维护内存加速建索引
SET maintenance_work_mem = '8GB';
SET max_parallel_maintenance_workers = 7;

-- 2. 批量加载后再建索引
-- COPY 导入数据 → CREATE INDEX → 开始查询

-- 3. 生产环境并发建索引
CREATE INDEX CONCURRENTLY ON items USING hnsw (embedding vector_l2_ops);

-- 4. Vacuum 前先 REINDEX 加速
REINDEX INDEX CONCURRENTLY items_embedding_idx;
VACUUM items;

-- 5. 监控索引构建进度
SELECT phase, round(100.0 * blocks_done / nullif(blocks_total, 0), 1) AS "%"
FROM pg_stat_progress_create_index;

-- 6. 使用 EXPLAIN 分析查询
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM items ORDER BY embedding <-> '[3,1,2]' LIMIT 5;

-- 7. 对比精确/近似搜索召回率
BEGIN;
SET LOCAL enable_indexscan = off;  -- 强制精确搜索
SELECT ...;
COMMIT;
```

## 注意事项

1. **索引前提**：HNSW/IVFFlat 索引只能在 ORDER BY + LIMIT + 距离算子的组合上生效，不能在表达式上使用
2. **维度限制**：`vector` 最多 2000 维，`halfvec` 最多 4000 维，`bit` 最多 64000 维
3. **数据预处理**：OpenAI Embedding 输出已归一化，推荐使用内积（`<#>`）；否则使用余弦距离
4. **索引顺序**：先导入数据再创建索引，速度提升显著
5. **HNSW 内存**：索引不强制全部驻留内存，但内存中性能更好。可用二进制量化缩小索引
6. **IVFFlat 召回率**：建索引时表内必须有数据，否则 K-means 聚类效果差
7. **WAL 复制**：pgvector 使用 WAL，支持流复制和 PITR
8. **Spring Boot 集成**：使用 `pgvector-java` 库或 JDBC 直接操作，配合 MyBatis-Plus 使用时注意向量类型的自定义 TypeHandler
