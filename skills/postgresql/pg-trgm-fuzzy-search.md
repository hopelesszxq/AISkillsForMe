---
name: pg-trgm-fuzzy-search
description: PostgreSQL pg_trgm 模糊搜索与文本相似度查询：三元组索引、相似度排序、拼写纠错、中文支持
tags: [postgresql, pg_trgm, fuzzy-search, similarity, full-text-search, gin, indexing]
---

## 概述

pg_trgm 是 PostgreSQL 的官方扩展，通过将文本分解为连续的三元组（trigram）来实现模糊匹配和相似度计算。适合搜索引擎的**拼写纠错**、**自动补全**、**模糊查询**和**去重归并**场景，与 FTS（全文搜索）互补使用。

## 一、安装与启用

```sql
-- 安装扩展（每个数据库单独启用）
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- 查看版本
SELECT * FROM pg_available_extensions WHERE name = 'pg_trgm';
-- 输出: pg_trgm | 1.6 | text similarity measurement and index searching based on trigrams

-- 查看提供的函数和运算符
\dx+ pg_trgm
```

## 二、核心函数与运算符

### 相似度函数

```sql
-- 相似度(0~1)，1 表示完全相同
SELECT similarity('hello', 'hallo');       -- 0.333...
SELECT similarity('hello', 'hello');       -- 1.0
SELECT similarity('hello', 'world');       -- 0.0

-- 显示三元组（调试用）
SELECT show_trgm('hello');
-- {"  h"," he",ell,hel,"lo ",llo}

-- 单词相似度（忽略顺序）
SELECT word_similarity('hello', 'say hello world');  -- 1.0

-- 严格单词相似度（边界敏感）
SELECT strict_word_similarity('hello', 'say hello world');  -- 1.0
SELECT strict_word_similarity('hello', 'sayhelloworld');    -- 较低或0
```

### 距离/差异度

```sql
-- 差异度 = 1 - similarity(差异小→相似度高)
SELECT 1 - similarity('hello', 'hallo');    -- 0.666...

-- 差异度运算符 <->（用于排序）
SELECT 'hello' <-> 'hallo';                 -- 0.666...
```

### 匹配运算符

```sql
-- 包含相似语义（任意三元组匹配即可）
SELECT 'hello' % 'hallo';           -- true （相似度 > pg_trgm.similarity_threshold）

-- 严格单词匹配
SELECT 'hello' <% 'say hello world';  -- true
SELECT 'hello' %> 'say hello world';  -- true

-- 差异度计算
SELECT 'hello' <<-> 'hallo';        -- 严格单词距离
```

## 三、索引

pg_trgm 支持 GIN 和 GiST 索引，GIN 查询更快但更新慢，GiST 适合频繁写入场景。

### GIN 索引（推荐查询密集场景）

```sql
-- 标准表达式索引（使用 similarity 运算符）
CREATE INDEX idx_users_name_trgm ON users USING GIN (name gin_trgm_ops);

-- 多列复合索引（与普通列组合）
CREATE INDEX idx_users_name_email_trgm ON users USING GIN (name gin_trgm_ops, email gin_trgm_ops);
```

### GiST 索引（写入密集场景）

```sql
-- GiST 索引适合频繁 INSERT/UPDATE
CREATE INDEX idx_users_name_gist ON users USING GIST (name gist_trgm_ops);
```

> GIN 查询比 GiST 快约 3~5 倍，但索引构建和写入略慢。生产环境推荐 GIN。

### 阈值控制

```sql
-- 查看当前阈值（默认 0.3）
SHOW pg_trgm.similarity_threshold;    -- 0.3

-- 会话级修改
SET pg_trgm.similarity_threshold = 0.4;

-- 修改后，低于 0.4 的匹配被过滤
SELECT 'hello' % 'hallo';
```

## 四、常见查询模式

### 1. 模糊搜索（Top-N 相似度）

```sql
-- 搜索与 "johnsonn" 最相似的前 10 个名字
SELECT name, similarity(name, 'johnsonn') AS sim
FROM users
WHERE name % 'johnsonn'
ORDER BY sim DESC
LIMIT 10;

-- 使用差异度运算符更简洁（<-> 越小越相似）
SELECT name, name <-> 'johnsonn' AS dist
FROM users
WHERE name % 'johnsonn'
ORDER BY name <-> 'johnsonn'
LIMIT 10;
```

### 2. 拼写纠错（自动补全/建议）

```sql
-- 用户输入 "teh"，推荐最接近的 5 个词
SELECT word, similarity(word, 'teh') AS sim
FROM (
  SELECT unnest(string_to_array(lower(content), ' ')) AS word
  FROM articles
) t
WHERE word % 'teh'
ORDER BY sim DESC
LIMIT 5;
```

### 3. 近似去重（Duplicate Detection）

```sql
-- 查找相似度 > 0.7 的重复记录
SELECT a.id AS id_a, b.id AS id_b,
       a.title, b.title,
       similarity(a.title, b.title) AS sim
FROM articles a
JOIN articles b ON a.id < b.id
               AND a.title % b.title
WHERE similarity(a.title, b.title) > 0.7
ORDER BY sim DESC;
```

### 4. 自动补全（Prefix Search）

结合 `LIKE` 或 `ILIKE` 实现搜索建议：

```sql
-- 用户输入 "joh"，查询匹配前缀的候选词
SELECT DISTINCT name
FROM users
WHERE name ILIKE 'joh%'
LIMIT 10;

-- pg_trgm 可以加速这种 LIKE 查询（自动利用 gin_trgm_ops 索引）
-- 注意：ILIKE 模式以常量开头时 pg_trgm 索引生效
EXPLAIN ANALYZE
SELECT * FROM users WHERE name ILIKE 'joh%' LIMIT 10;
```

### 5. 结合全文搜索

```sql
-- 先全文搜索过滤相关文档，再对搜索结果做拼写纠错
WITH matched AS (
  SELECT id, title, body,
         ts_rank(to_tsvector('simple', body), plainto_tsquery('simple', 'postgresql performanc')) AS rank
  FROM articles
  WHERE to_tsvector('simple', body) @@ plainto_tsquery('simple', 'postgresql performanc')
  ORDER BY rank DESC
  LIMIT 50
)
SELECT id, title, body,
       similarity(body, 'postgresql performanc') AS fuzzy_sim
FROM matched
ORDER BY fuzzy_sim DESC
LIMIT 10;
```

## 五、性能调优

### 索引大小估算

```sql
-- 查看索引大小
SELECT pg_size_pretty(pg_relation_size('idx_users_name_trgm'));
```

### 批量构建

```sql
-- 对已有大表创建索引时控制并行度和维护开销
SET maintenance_work_mem = '1GB';
SET max_parallel_maintenance_workers = 4;

CREATE INDEX CONCURRENTLY idx_users_name_trgm
ON users USING GIN (name gin_trgm_ops);
```

### 查询计划检查

```sql
-- 确认是否走索引（Bitmap Index Scan）
EXPLAIN (ANALYZE, BUFFERS)
SELECT name
FROM users
WHERE name % 'johnsonn'
ORDER BY name <-> 'johnsonn'
LIMIT 10;
```

## 六、中文支持

PostgreSQL 的 pg_trgm 按字节分割三元组，对中文按 UTF-8 字节切分，效果有限。推荐以下方案：

### 方案 A：拼音辅助字段

```sql
-- 增加拼音字段
ALTER TABLE users ADD COLUMN name_pinyin TEXT;

-- 使用 pypinyin 等工具在应用层生成拼音后存储
-- "张三" → "zhang san"

-- 在拼音字段上建索引
CREATE INDEX idx_users_name_pinyin_trgm
ON users USING GIN (name_pinyin gin_trgm_ops);

-- 查询时也转拼音
SELECT *
FROM users
WHERE name_pinyin % 'zhang san'
ORDER BY name_pinyin <-> 'zhang san'
LIMIT 10;
```

### 方案 B：分词 + pg_trgm

```sql
-- 使用 jieba 等分词工具在应用层分词后存储
-- "我是一个程序员" → "我 是 一个 程序员"

-- 在分词字段上创建索引
CREATE INDEX idx_users_desc_segmented ON users USING GIN (description_segmented gin_trgm_ops);
```

## 注意事项

1. **最小长度限制** — pg_trgm 是基于 3 字符的，短文本（<=2 个字符）无法有效计算相似度。短文本场景考虑 `pg_trgm.word_similarity` 或应用层处理。
2. **阈值设置** — 默认 `similarity_threshold = 0.3`，对中文场景偏低（中文三元组匹配更稀疏），建议调整到 0.2~0.25。在线调整需 `SET` 语句，重启失效。
3. **索引更新开销** — GIN 索引更新延迟较高，高频写入场景（每秒数百次 INSERT/UPDATE）建议使用 GiST 或批量写入后重建索引。
4. **NULL 处理** — `similarity(NULL, 'hello')` 返回 NULL，`'hello' % NULL` 返回 NULL。查询时注意 COALESCE。
5. **大小写敏感** — `similarity` 是大小写敏感的！建议在索引和查询中统一 `lower()`。
   ```sql
   CREATE INDEX idx_users_name_lower_trgm ON users USING GIN (lower(name) gin_trgm_ops);
   SELECT * FROM users WHERE lower(name) % 'johnson';
   ```
6. **pg_trgm vs 全文搜索** — FTS（`to_tsvector` / `tsquery`）适合语义搜索和词干匹配；pg_trgm 适合拼写纠错和模糊匹配。两者互补，同一个字段可以同时建 `gin_trgm_ops` 和 `gin(to_tsvector(...))` 两种索引。
7. **资源消耗** — 大文本（>1000 字符）的 pg_trgm 索引较大，每个文本生成约 `len(text) - 2` 个三元组。`text` 字段建议限制长度或使用分段索引。
