---
name: redis-search-query-tuning
description: Redis 8.0 内置查询引擎 FT.SEARCH 性能调优：索引设计、查询优化、Profile 分析、集群部署策略
tags: [redis, redis8, search, performance, tuning, ft-search]
---

## 概述

Redis 8.0 将 RediSearch 的搜索能力原生内置到内核中。FT.SEARCH、FT.AGGREGATE 等命令不再依赖外部模块，但索引设计和查询编写不当会导致严重的性能问题。本文覆盖搜索查询的全链路调优。

> 背景知识请参考 `redis8-features.md`—本文聚焦性能调优实践。

## 一、索引设计最佳实践

### 1.1 字段类型选择

```bash
# ❌ 低效：所有字段都索引
FT.CREATE idx:products ON HASH PREFIX 1 product: SCHEMA
  name TEXT WEIGHT 5.0        # 全文搜索（最重）
  description TEXT WEIGHT 1.0
  price NUMERIC               # 数值范围查询
  category TAG SEPARATOR ,    # 标签精确匹配
  created_at NUMERIC SORTABLE # 排序用

# ✅ 高效：仅对搜索字段建索引
FT.CREATE idx:products ON HASH PREFIX 1 product: SCHEMA
  name TEXT WEIGHT 5.0
  category TAG SEPARATOR ,    # 只索引必要的
  price NUMERIC               # 按需索引
  # description 不索引（仅展示，不搜索）
  # created_at 不索引（不搜索不排序）
```

**字段索引性能对比**（从重到轻）：

| 类型 | 存储开销 | 查询速度 | 适用场景 |
|------|---------|---------|---------|
| TEXT | 最高 | 中 | 全文搜索（title, content） |
| TAG | 低 | 快 | 精确匹配（category, status） |
| NUMERIC | 中 | 快 | 范围查询（price, rating） |
| GEO | 中 | 中 | 地理空间查询 |
| VECTOR | 高 | 取决维度 | 向量相似搜索 |
| SORTABLE | 额外开销 | - | 需要排序的字段（慎用） |

### 1.2 索引前缀与过滤

```bash
# 使用 PREFIX 限制数据范围（避免全量扫描）
FT.CREATE idx:orders ON HASH PREFIX 2 order:active: order:archived: SCHEMA ...

# 多前缀 vs 通配符：多前缀更高效
# ✅ 推荐
PREFIX 2 order:active: order:paid:
# ❌ 避免
PREFIX 1 order:
```

### 1.3 停用词与同义词

```bash
# 自定义停用词，减少索引大小
FT.CREATE idx:articles ON HASH PREFIX 1 article:
  STOPWORDS 10 the a an of to in for is on at
  SCHEMA title TEXT content TEXT
```

## 二、查询优化

### 2.1 使用 LIMIT 限制结果集

```bash
# ❌ 低效：查询全部结果
FT.SEARCH idx:products "@name:(laptop)" SORTBY price DESC

# ✅ 高效：分页查询
FT.SEARCH idx:products "@name:(laptop)" SORTBY price DESC LIMIT 0 20
```

### 2.2 查询语法优化

```bash
# ❌ 模糊查询（全文扫描）
FT.SEARCH idx:products "laptop computer"  # 默认 OR，结果多

# ✅ 精确限定
FT.SEARCH idx:products "@name:laptop @category:electronics"

# ✅ 使用前缀匹配代替模糊
FT.SEARCH idx:products "@name:lap*"         # 前缀索引更快

# ✅ 使用 TAG 字段替代 TEXT
FT.SEARCH idx:products "@category:{electronics}"  # TAG 同等条件下更快
```

### 2.3 聚合查询优化

```bash
# ❌ 全量聚合
FT.AGGREGATE idx:orders "*"
  GROUPBY 1 @category
  REDUCE COUNT 0 AS count

# ✅ 先过滤再聚合
FT.AGGREGATE idx:orders "@price:[100 +inf]"
  GROUPBY 1 @category
  REDUCE COUNT 0 AS count
  REDUCE AVG 1 @price AS avg_price
  SORTBY 2 @count DESC
  LIMIT 0 10
```

## 三、性能诊断命令

### 3.1 FT.PROFILE 查询分析

```bash
# 查看查询执行计划（定位慢查询）
FT.PROFILE idx:products SEARCH
  QUERY "@name:(laptop) @price:[500 2000]"
  LIMITED  # 只返回前 N 步，减少开销

# 输出示例：
# 1) 1) "Total profile time"      → 总耗时
#    2) "6.6540000000000004"
# 2) 1) "Parsing time"            → 解析时间
#    2) "0.11200000000000001"
# 3) 1) "Pipeline creation time"  → 执行计划构建
#    2) "0.128"
# 4) 1) "Iterators profile"       → 各迭代器耗时
#    2) 1) "Query type: UNION"
#       2) "Query type: INTERSECT"
#       3) "Query type: TAG"
#       4) "Query type: NUMERIC"
```

### 3.2 索引统计

```bash
# 查看索引状态
FT.INFO idx:products

# 关键字段解读：
# - num_docs：文档数
# - num_terms：词汇数（越大越需要更多内存）
# - max_doc_id：最大文档 ID
# - indexing_failures：索引失败数（应始终为 0）
# - inverted_sz_mb：倒排索引大小（主内存消耗）
# - offset_vectors_sz_mb：偏移向量大小
# - skip_index_size_mb：跳表索引大小
# - total_indexing_time：索引构建总耗时
```

## 四、内存与资源调优

### 4.1 关键配置参数

```ini
# redis.conf — 搜索相关配置

# 索引构建并发性
search-max-search-results 1000000       # 最大搜索结果数（默认 1000000）

# 查询超时保护（防止慢查询耗尽 CPU）
search-query-timeout 500                # 查询超时（ms）

# 并发限制
search-max-pending-queries 1000         # 最大待处理查询数
search-max-filter-expansions 100        # 如查询扩展过多则裁剪

# 游标查询
search-cursor-read-size 1000            # 游标每次读取量
```

### 4.2 内存监控

```bash
# 全局搜索内存占用
INFO search

# 输出包含：
# search_active_index_count         有效索引数
# search_total_indexing_time        索引总耗时
# search_total_index_failures       索引失败次数
# search_bytes_indexed              索引字节数
# search_bytes_unindexed            未索引字节数
```

## 五、集群部署策略

### 5.1 索引分片

```bash
# Redis 8.0 集群模式下，索引支持分片
# 当集群有 6 个分片时，索引自动分布在所有分片
FT.CREATE idx:global ON HASH PREFIX 1 global: SCHEMA ...
# 查询时自动合并结果

# ⚠️ 注意：分片数会影响聚合查询性能
# 建议：分片数 ≤ 集群总 CPU 核心数
```

### 5.2 跨分片聚合

```bash
# FT.AGGREGATE 在集群模式下会自动跨分片聚合
# 优化策略：

# 1. 在查询层面过滤，减少跨分片数据量
FT.AGGREGATE idx:global "@price:[1000 +inf]"
  GROUPBY 1 @region
  REDUCE SUM 1 @sales AS total_sales

# 2. 使用 LOAD 字段仅加载必要字段
FT.SEARCH idx:global "@name:laptop" RETURN 3 name price stock
```

## 六、常见坑点与避坑指南

### ⚠️ 坑点 1：TEXT 字段不加权重
```bash
# 默认 WEIGHT 1.0，不加区分导致搜索结果不精准
# 改为：核心字段 WEIGHT 5.0，次要字段 WEIGHT 1.0
```

### ⚠️ 坑点 2：大文本全文索引
- TEXT 字段索引整个字符串，超大文本（>10KB）导致倒排索引膨胀
- 方案：只索引标题/摘要，正文用 `RETURN` 加载

### ⚠️ 坑点 3：SORTABLE 滥用
- 每个 SORTABLE 字段在内存中维护一个 skiplist
- 仅对排序字段加 `SORTABLE`，避免所有 NUMERIC 字段都加

### ⚠️ 坑点 4：未设置查询超时
```bash
# 生产环境必须设置超时，防止恶意/错误查询拖垮 CPU
search-query-timeout 500
```

### ⚠️ 坑点 5：集群模式下的 CURSOR
- 集群模式不支持 `FT.CURSOR`
- 替代方案：`LIMIT` + `OFFSET` 或应用层分页

### ⚠️ 坑点 6：中文分词
```bash
# Redis 8.0 内置查询引擎默认使用分词器
# 中文搜索需明确指定：
FT.CREATE idx:cn ON HASH PREFIX 1 doc:
  SCHEMA title TEXT WEIGHT 5.0
# 使用 FT.SEARCH 时支持中文分词（基于 ICU）
# 确保 Redis 编译时启用了 icu 支持
```
