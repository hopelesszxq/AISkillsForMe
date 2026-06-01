---
name: pg-jsonb-fulltext
description: PostgreSQL JSONB 高级实战与全文搜索：JSON 数据操作、索引优化、全文检索、混合查询
tags: [postgresql, jsonb, full-text-search, indexing, performance, gin]
---

## 概述

PostgreSQL 的 JSONB 数据类型和全文搜索（Full-Text Search, FTS）的组合使用，可以在关系数据库中实现高效的半结构化数据存储和全文检索能力，避免引入额外的搜索中间件。

## 1. JSONB 数据类型基础

### 1.1 JSON vs JSONB 选择

| 特性 | JSON | JSONB |
|------|------|-------|
| 存储格式 | 文本（原样保存） | 二进制（分解后的结构） |
| 写入速度 | 快（无解析开销） | 慢（需解析+分解） |
| 查询性能 | 慢（每次查询需重解析） | 快（按结构直接访问） |
| 索引支持 | 不支持 | 支持 GIN/B-tree 索引 |
| 重复键处理 | 保留所有键 | 保留最后一个值 |
| 空格/缩进 | 保留 | 不保留（标准化） |

**结论**：几乎总是使用 JSONB，除非只需要日志存储（只写不查）。

### 1.2 建表与基本操作

```sql
-- 创建含 JSONB 字段的表
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    details JSONB NOT NULL DEFAULT '{}',
    attributes JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 插入 JSONB 数据
INSERT INTO products (name, details, attributes) VALUES (
    'MacBook Pro 16"',
    '{
        "brand": "Apple",
        "model": "MacBook Pro",
        "specs": {
            "cpu": "M4 Pro",
            "ram": "32GB",
            "storage": "512GB SSD",
            "screen": "16.2 inch Liquid Retina XDR"
        },
        "price": 19999,
        "inStock": true,
        "colors": ["Silver", "Space Gray", "Space Black"]
    }',
    '{"weight": "2.14kg", "warranty": "2 years"}'
);

INSERT INTO products (name, details) VALUES (
    'ThinkPad X1 Carbon',
    '{
        "brand": "Lenovo",
        "model": "ThinkPad X1 Carbon Gen 13",
        "specs": {"cpu": "Intel Core Ultra 7", "ram": "32GB", "storage": "1TB SSD"},
        "price": 12999,
        "inStock": true,
        "colors": ["Black"]
    }'
);
```

### 1.3 查询操作符

```sql
-- -> 返回 JSON（对象/数组）
SELECT details -> 'brand' AS brand FROM products;
-- 结果: "Apple"

-- ->> 返回文本（text 类型）
SELECT details ->> 'brand' AS brand FROM products;
-- 结果: Apple

-- #> 路径获取 JSON
SELECT details #> '{specs, cpu}' AS cpu FROM products;
-- 结果: "M4 Pro"

-- #>> 路径获取文本
SELECT details #>> '{specs, ram}' AS ram FROM products;

-- ? 是否存在键
SELECT * FROM products WHERE details ? 'colors';

-- ?| 是否存在任一键
SELECT * FROM products WHERE details ?| ARRAY['colors', 'tags'];

-- ?& 是否存在所有键
SELECT * FROM products WHERE details ?& ARRAY['brand', 'specs'];

-- @> 是否包含（匹配子集）
SELECT * FROM products 
WHERE details @> '{"brand": "Apple", "inStock": true}';

-- <@ 是否被包含
SELECT * FROM products 
WHERE '{"brand": "Apple"}' <@ details;

-- || JSONB 拼接/合并
UPDATE products 
SET details = details || '{"discount": 0.1}'::JSONB
WHERE id = 1;

-- - 删除键
UPDATE products 
SET details = details - 'discount'
WHERE id = 1;

-- jsonb_set 更新嵌套字段
UPDATE products 
SET details = jsonb_set(
    details, 
    '{specs, ram}', 
    '"64GB"'::JSONB, 
    true
)
WHERE id = 1;

-- jsonb_array_elements 展开数组
SELECT id, 
       jsonb_array_elements_text(details -> 'colors') AS color
FROM products WHERE details ? 'colors';

-- jsonb_each 展开键值对
SELECT id, key, value 
FROM products, 
     jsonb_each(details -> 'specs')
WHERE id = 1;
```

## 2. JSONB 索引优化

### 2.1 GIN 索引（通用）

```sql
-- 通用 GIN 索引（支持 ?、?|、?&、@> 操作符）
CREATE INDEX idx_products_details_gin 
ON products USING GIN (details);

-- 仅 @> 操作符的 GIN 索引（更小更快，但不支持键存在查询）
CREATE INDEX idx_products_details_gin_path 
ON products USING GIN (details jsonb_path_ops);
```

**jsonb_path_ops 与默认对比**：

```sql
-- 默认 GIN：支持 ? ?| ?& @>
-- 大小：约 2.5MB（10 万行）
-- 适用：需要检测键的存在性

-- jsonb_path_ops GIN：只支持 @> 但更快
-- 大小：约 1.2MB（10 万行），缩小 50%
-- 适用：只有包含查询的场景
```

### 2.2 表达式索引（特定路径）

当查询针对特定 JSON 路径时，表达式索引更高效：

```sql
-- 只对 brand 字段建立索引
CREATE INDEX idx_products_brand 
ON products ((details ->> 'brand'));

-- 复合索引：品牌 + 价格
CREATE INDEX idx_products_brand_price 
ON products ((details ->> 'brand'), ((details -> 'price')::NUMERIC));

-- 使用表达式索引的查询
SELECT * FROM products 
WHERE details ->> 'brand' = 'Apple' 
  AND (details -> 'price')::NUMERIC < 20000;
```

### 2.3 索引选择指南

```sql
-- 场景一：按品牌和价格范围查询 → 表达式索引
CREATE INDEX idx_prod_brand_price 
ON products ((details ->> 'brand'), ((details -> 'price')::NUMERIC));

-- 场景二：复杂的 JSON 包含查询 → GIN 索引
CREATE INDEX idx_prod_gin ON products USING GIN (details);

-- 场景三：混合查询 → 表达式 + GIN 联合
CREATE INDEX idx_prod_gin ON products USING GIN (details jsonb_path_ops);
CREATE INDEX idx_prod_category ON products ((details ->> 'category'));
```

## 3. 全文搜索（Full-Text Search）

### 3.1 基础概念

```sql
-- 词素分析（tsvector）
SELECT to_tsvector('english', 'PostgreSQL is a powerful, open-source database');
-- 结果: 'databas':6 'open-sourc':5 'postgresql':1 'power':4

-- 查询解析（tsquery）
SELECT to_tsquery('english', 'powerful & database');
-- 结果: 'power' & 'databas'

-- 匹配查询
SELECT to_tsvector('english', 'PostgreSQL is powerful') 
       @@ to_tsquery('english', 'powerful & database');
-- 结果: false
```

### 3.2 为产品表启用全文搜索

```sql
-- 添加 tsvector 列
ALTER TABLE products ADD COLUMN search_vector TSVECTOR;

-- 建立搜索内容：名称 + JSONB 中的品牌和型号
UPDATE products SET search_vector = 
    to_tsvector('simple', 
        COALESCE(name, '') || ' ' || 
        COALESCE(details ->> 'brand', '') || ' ' ||
        COALESCE(details ->> 'model', '') || ' ' ||
        COALESCE(details #>> '{specs, cpu}', '') || ' ' ||
        COALESCE(details #>> '{specs, ram}', '')
    );

-- 创建 GIN 索引
CREATE INDEX idx_products_search ON products USING GIN (search_vector);
```

### 3.3 自动更新触发器

```sql
-- 创建自动更新 search_vector 的触发器函数
CREATE OR REPLACE FUNCTION products_search_update()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector := 
        to_tsvector('simple',
            COALESCE(NEW.name, '') || ' ' ||
            COALESCE(NEW.details ->> 'brand', '') || ' ' ||
            COALESCE(NEW.details ->> 'model', '') || ' ' ||
            COALESCE(NEW.details #>> '{specs, cpu}', '') || ' ' ||
            COALESCE(NEW.details #>> '{specs, ram}', '') || ' ' ||
            COALESCE(NEW.details #>> '{specs, storage}', '')
        );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 创建触发器
CREATE TRIGGER trg_products_search 
BEFORE INSERT OR UPDATE OF name, details
ON products
FOR EACH ROW
EXECUTE FUNCTION products_search_update();
```

### 3.4 全文搜索查询

```sql
-- 基础搜索
SELECT id, name, details ->> 'brand' AS brand,
       ts_rank(search_vector, query) AS rank
FROM products, 
     to_tsquery('simple', 'm4 & pro') AS query
WHERE search_vector @@ query
ORDER BY rank DESC;

-- 使用 ts_headline 高亮匹配词
SELECT id, name,
       ts_headline('simple', name, query, 'MaxWords=20, MinWords=5') AS headline
FROM products,
     to_tsquery('simple', 'thinkpad & carbon') AS query
WHERE search_vector @@ query;

-- 带过滤的混合查询（全文搜索 + JSONB 过滤）
SELECT id, name, details ->> 'price' AS price
FROM products,
     to_tsquery('simple', 'laptop') AS query
WHERE search_vector @@ query
  AND details @> '{"inStock": true}'
  AND (details -> 'price')::NUMERIC BETWEEN 5000 AND 20000
ORDER BY (details -> 'price')::NUMERIC ASC;

-- 分页搜索
SELECT id, name, details,
       ts_rank(search_vector, query) AS rank
FROM products,
     to_tsquery('simple', $1) AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20 OFFSET 0;
```

## 4. 高级应用模式

### 4.1 动态属性存储（EAV 替代）

```sql
-- 电商商品：属性差异极大的场景
CREATE TABLE products_eav_style (
    id BIGSERIAL PRIMARY KEY,
    category VARCHAR(50) NOT NULL,
    name VARCHAR(200),
    attributes JSONB NOT NULL,  -- 动态属性
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 电子产品：CPU、RAM、存储
INSERT INTO products_eav_style (category, name, attributes) VALUES
('electronics', 'MacBook Pro', 
 '{"cpu": "M4 Pro", "ram": "32GB", "storage": "512GB", "usb_ports": 3}');

-- 服装：尺码、颜色、材质
INSERT INTO products_eav_style (category, name, attributes) VALUES
('clothing', 'Winter Jacket',
 '{"size": "L", "color": "Black", "material": "Gore-Tex", "waterproof": true}');

-- 查询：所有防水服装
SELECT * FROM products_eav_style 
WHERE category = 'clothing' 
  AND attributes @> '{"waterproof": true}';

-- 查询：存储 >= 512GB 的电子产品
SELECT * FROM products_eav_style 
WHERE category = 'electronics' 
  AND (attributes ->> 'storage') ~ '^\d+GB$' 
  AND REPLACE(attributes ->> 'storage', 'GB', '')::INT >= 512;
```

### 4.2 JSONB 和 全文搜索混合索引

```sql
-- 综合索引：全文搜索 + JSONB GIN
-- 支持搜索文本 + JSON 过滤
CREATE INDEX idx_products_hybrid ON products USING GIN (search_vector);
CREATE INDEX idx_products_details_gin ON products USING GIN (details jsonb_path_ops);

-- 复合查询示例（高性能）
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name, details ->> 'brand' AS brand,
       (details -> 'price')::NUMERIC AS price
FROM products
WHERE search_vector @@ plainto_tsquery('simple', 'm4 pro laptop')
  AND details @> '{"inStock": true}'
ORDER BY ts_rank(search_vector, plainto_tsquery('simple', 'm4 pro laptop')) DESC
LIMIT 10;
```

### 4.3 使用 JSONB 存储配置中心数据

```sql
-- 应用配置存储
CREATE TABLE app_configs (
    app_name VARCHAR(100) PRIMARY KEY,
    config JSONB NOT NULL DEFAULT '{}',
    version INT NOT NULL DEFAULT 1,
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 插入配置
INSERT INTO app_configs (app_name, config) VALUES (
    'order-service',
    '{
        "timeout": 5000,
        "retry": {"count": 3, "backoff": "exponential"},
        "features": {
            "payment": {"enabled": true, "providers": ["alipay", "wechat"]},
            "notification": {"enabled": true, "channels": ["email", "sms"]}
        },
        "rateLimit": {"global": 1000, "perUser": 10}
    }'
);

-- 局部更新配置（只修改 retry.count，不覆盖其他字段）
UPDATE app_configs 
SET config = jsonb_set(
    config, 
    '{retry, count}', 
    '5'::JSONB
),
    version = version + 1,
    updated_at = NOW()
WHERE app_name = 'order-service';

-- 嵌套配置查询
SELECT 
    config #>> '{features, payment, enabled}' AS payment_enabled,
    config #>> '{rateLimit, perUser}' AS per_user_limit
FROM app_configs WHERE app_name = 'order-service';
```

### 4.4 JSONB 树形结构（层级分类）

```sql
-- 分类树
CREATE TABLE category_tree (
    id SERIAL PRIMARY KEY,
    path LTREE NOT NULL,              -- 路径枚举（需启用 ltree 扩展）
    metadata JSONB NOT NULL DEFAULT '{}'
);

-- 需要启用 ltree 扩展
CREATE EXTENSION IF NOT EXISTS ltree;

-- 插入分类树
INSERT INTO category_tree (path, metadata) VALUES
('Electronics', '{"name": "电子产品", "level": 1, "visible": true}'),
('Electronics.Computers', '{"name": "电脑", "level": 2, "visible": true}'),
('Electronics.Computers.Laptops', '{"name": "笔记本", "level": 3, "visible": true}'),
('Electronics.Computers.Desktops', '{"name": "台式机", "level": 3, "visible": true}'),
('Electronics.Phones', '{"name": "手机", "level": 2, "visible": true}');

-- 查询所有子分类
SELECT subpath(path, 0, -1) AS parent,
       path::TEXT AS full_path,
       metadata ->> 'name' AS name
FROM category_tree
WHERE path <@ 'Electronics.Computers';

-- 查询所有顶层分类
SELECT * FROM category_tree WHERE nlevel(path) = 1;
```

## 5. 性能对比与最佳实践

### 5.1 JSONB vs 关系表

| 场景 | JSONB | 关系表（规范化） |
|------|-------|-----------------|
| 属性固定、查询模式简单 | 不推荐 | 推荐（类型安全、约束完整） |
| 属性动态变化 | 推荐 | 不推荐（频繁 DDL） |
| 复杂嵌套查询 | 推荐 | 不推荐（多表 JOIN） |
| 需要外键约束 | 不推荐 | 推荐 |
| 全文搜索 | 推荐（结合 FTS） | 需要额外实现 |
| 统计分析 | 较复杂 | 简单 |

### 5.2 查询性能优化

```sql
-- ❌ 低效：函数包裹导致无法使用索引
SELECT * FROM products 
WHERE LOWER(details ->> 'brand') = 'apple';

-- ✅ 高效：直接在表达式索引上查询（索引已存储处理后的值）
-- 先建立 LOWER 表达式索引
CREATE INDEX idx_products_brand_lower 
ON products (LOWER(details ->> 'brand'));
-- 然后查询
SELECT * FROM products 
WHERE details ->> 'brand' = 'Apple';  -- 匹配大小写一致
-- 或
SELECT * FROM products 
WHERE LOWER(details ->> 'brand') = 'apple';

-- ❌ 低效：类型转换在 WHERE 条件中
SELECT * FROM products 
WHERE (details -> 'price')::NUMERIC > 10000;

-- ✅ 高效：使用表达式索引
CREATE INDEX idx_products_price 
ON products (((details -> 'price')::NUMERIC));
```

### 5.3 JSONB 数据验证

```sql
-- 使用 CHECK 约束验证 JSONB 结构
CREATE TABLE products_with_validation (
    id BIGSERIAL PRIMARY KEY,
    details JSONB NOT NULL,
    CONSTRAINT valid_product_details CHECK (
        details ? 'brand' 
        AND details ? 'price'
        AND (details -> 'price')::NUMERIC > 0
        AND jsonb_typeof(details -> 'price') = 'number'
    )
);

-- 自定义函数做更复杂的验证
CREATE OR REPLACE FUNCTION validate_product_jsonb(details JSONB)
RETURNS BOOLEAN AS $$
BEGIN
    IF NOT (details ? 'brand') THEN
        RAISE EXCEPTION '缺少 brand 字段';
    END IF;
    IF NOT (details ? 'price') THEN
        RAISE EXCEPTION '缺少 price 字段';
    END IF;
    IF (details -> 'price')::NUMERIC <= 0 THEN
        RAISE EXCEPTION 'price 必须大于 0';
    END IF;
    IF details ? 'specs' AND NOT (details -> 'specs') ? 'cpu' THEN
        RAISE EXCEPTION 'specs 中缺少 cpu 字段';
    END IF;
    RETURN true;
END;
$$ LANGUAGE plpgsql;

ALTER TABLE products 
ADD CONSTRAINT chk_product_details 
CHECK (validate_product_jsonb(details));
```

## 6. 注意事项

| 要点 | 说明 |
|------|------|
| **JSONB 更新成本** | 更新 JSONB 字段的任意部分都会重写整行数据，高频更新字段应拆分到单独列 |
| **GIN 索引维护** | 频繁 DML 的表上 GIN 索引维护成本高，需要考虑 autovacuum 频率 |
| **全文搜索词库** | 中文搜索需要 `zhparser` 或 `jieba` 等扩展，PG 原生不支持中文分词 |
| **JSONB 大小限制** | 单个 JSONB 文档最大约 255MB，但建议单行 < 10KB |
| **NULL vs 缺失键** | JSONB 中 `"key": null` 和键不存在是不同的，`@>` 操作符对二者处理不同 |
| **并行查询** | JSONB 的 GIN 索引扫描不支持并行，大表查询需注意 | 
| **to_tsvector 配置** | `simple` 配置不做词干提取适合专有名词搜索；`english` 做词干提取适合英文文本 |
