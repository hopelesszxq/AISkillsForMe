---
name: pg-one-to-many-index
description: PostgreSQL 一对多关系复合索引策略：按子表过滤、按父表排序的性能优化方案
tags: [postgresql, indexing, performance, optimization, one-to-many]
---

## 要点

在关系型数据库（PostgreSQL）中处理「按子表条件过滤、按父表字段排序」的查询是一大经典挑战。本文对比 MongoDB 多键索引，给出 PostgreSQL 的几种解决方案。

### 问题场景

```sql
-- 需求：查询至少有一个子记录满足 child_value = 0.9 的父记录，
-- 按父表的 parent_value 排序，取 Top 10
SELECT DISTINCT p.parent_id, p.parent_value
FROM parent p
JOIN child c ON c.parent_id = p.parent_id
WHERE c.child_value = 0.9
ORDER BY p.parent_value
LIMIT 10;
```

MongoDB 通过多键索引（文档内嵌数组 + 复合索引）可以高效实现此查询。但在 PostgreSQL 中，由于范式化的父子表分离，需要特殊策略。

## 方案对比

| 策略 | 实现方式 | 读性能 | 写开销 | 复杂度 |
|------|---------|--------|--------|--------|
| **方案A**：子表冗余父字段 | 子表加 `parent_value` 列 + 复合索引 | ⭐⭐⭐ 最优 | 中等（级联约束） | 低 |
| **方案B**：父表存子值数组 | 父表加 `float[]` + GIN 索引 | ⭐⭐ 良好 | 高（触发器维护） | 中 |
| **方案C**：标准半连接 | JOIN + 子查询索引扫描 | ⭐ 较差 | 无 | 低 |

## 代码示例

### 方案A（推荐）：子表冗余父表字段

将父表排序字段反范式化到子表，创建复合索引。

```sql
-- 1. 子表增加 parent_value 列
ALTER TABLE child ADD COLUMN parent_value FLOAT;
UPDATE child SET parent_value = parent.parent_value
FROM parent WHERE child.parent_id = parent.parent_id;
ALTER TABLE child ALTER parent_value SET NOT NULL;

-- 2. 创建复合索引（同时覆盖过滤和排序）
CREATE INDEX ON child (child_value, parent_value);

-- 3. 建外键约束保证数据一致性（级联更新）
ALTER TABLE parent ADD UNIQUE (parent_id, parent_value);
ALTER TABLE child ADD CONSTRAINT fk_child_parent
  FOREIGN KEY (parent_id, parent_value)
  REFERENCES parent (parent_id, parent_value)
  ON UPDATE CASCADE;
```

**查询性能对比**：

```sql
-- 使用复合索引，单次 Index Scan 即可完成
SELECT DISTINCT parent_id, parent_value
FROM child
WHERE child_value = 0.9999
ORDER BY parent_value
LIMIT 10;

-- 执行计划：Index Scan + Incremental Sort
-- Buffers: shared hit=36（仅扫描33行即返回结果）
```

对比无冗余的标准半连接方案：

```sql
-- 标准 JOIN 方案：需要扫描 169 行 + 子查询物化
-- Buffers: shared hit=245（6倍IO开销）
```

**级联更新性能**：
```sql
-- 更新单个父记录（影响1000个子记录）
UPDATE parent SET parent_value = parent_value - 0.000000000000000042
WHERE parent_id = 789;
-- 执行时间：44ms（约束触发级联更新）

-- 批量更新所有父记录（1000条）
UPDATE parent SET parent_value = parent_value - 0.000000000000000042;
-- 执行时间：46s（约束逐行验证）
-- ⚠️ 适用于少量更新场景，不适用于频繁批量更新的字段
```

### 方案B：父表存子值数组 + GIN 索引

更接近 MongoDB 多键索引的内存模型。

```sql
-- 1. 安装扩展
CREATE EXTENSION IF NOT EXISTS btree_gin;

-- 2. 父表添加子值数组列
ALTER TABLE parent ADD COLUMN child_values FLOAT[];

UPDATE parent p SET child_values = (
  SELECT array_agg(DISTINCT child_value)
  FROM child WHERE parent_id = p.parent_id
);

-- 3. 创建 GIN 复合索引
CREATE INDEX ON parent USING GIN (parent_value, child_values);

-- 4. 维护触发器
CREATE OR REPLACE FUNCTION sync_child_values_array() RETURNS TRIGGER AS $$
BEGIN
  UPDATE parent SET child_values = (
    SELECT array_agg(DISTINCT child_value)
    FROM child WHERE parent_id = COALESCE(NEW.parent_id, OLD.parent_id)
  ) WHERE parent_id = COALESCE(NEW.parent_id, OLD.parent_id);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_child_values_array
AFTER INSERT OR UPDATE OR DELETE ON child
FOR EACH ROW EXECUTE FUNCTION sync_child_values_array();
```

**查询**：
```sql
SELECT parent_id, parent_value
FROM parent
WHERE child_values @> ARRAY[0.9999::FLOAT]
ORDER BY parent_value
LIMIT 10;

-- 执行计划：Bitmap Index Scan（GIN）+ Sort
-- Buffers: shared hit=14
-- ⚠️ GIN 索引无法复用排序顺序，需要额外 Sort 步骤
```

### 方案C（不推荐）：标准半连接

```sql
-- 无冗余，纯 JOIN
SELECT DISTINCT p.parent_id, p.parent_value
FROM parent p
JOIN child c ON c.parent_id = p.parent_id
WHERE c.child_value = 0.9999
ORDER BY p.parent_value
LIMIT 10;

-- 执行计划：两个索引各自扫描 → Nested Loop Semi Join
-- 需要扫描 parent_value 索引 169 行 + child 表 99 行
-- Buffers: shared hit=245
```

## 性能总结

| 方案 | 查询IO | 单条更新(1父) | 批量更新(1000父) |
|------|--------|---------------|------------------|
| A: 子表冗余 | 36 buffers | 44ms | 46s ⚠️ |
| B: GIN 数组 | 14 buffers | 高（触发器） | 极高 |
| C: 标准 JOIN | 245 buffers | 无开销 | 无开销 |

## 选型决策

```
父表更新不频繁（参考数据、SCD）→ 方案A（推荐）
父表频繁更新               → 方案C（接受性能损失）或重新设计查询
需要更接近 MongoDB 语义     → 方案B（注意写放大问题）
子表行数极大（>1亿）        → 方案A + 分区索引
```

## 注意事项

1. **级联约束不是免费的**：`ON UPDATE CASCADE` 通过触发器实现，批量更新时放大写开销
2. **触发器替代约束不推荐**：自定义触发器虽然验证开销略低，但无法保证数据一致性（子表可能被直接修改）
3. **GIN 索引的排序问题**：GIN 是倒排索引，不支持前向扫描排序，需要额外的 Sort 步骤
4. **数组维护成本高**：子表每次 DML 都要更新父表数组，写密集型场景不适用
5. **`btree_gin` 限制**：GIN 复合索引虽能支持等值过滤 + 数组包含，但第一列必须是标量
6. **`DISTINCT` 的处理**：方案A 的结果可能包含重复 parent 记录（多个子记录满足条件），需用 `DISTINCT` 或 `UNIQUE` 去重
