---
name: pg-brin-indexing
description: PostgreSQL BRIN 索引原理、适用场景与 PG17/PG18 新特性
tags: [postgresql, indexing, performance]
---

## BRIN 索引原理

BRIN（Block Range INdex）对数据物理顺序敏感，将连续的数据页划分为块范围，记录每个块范围内列的最小值和最大值。

```
Page 0-31:  min=100, max=500
Page 32-63: min=501, max=900
Page 64-95: min=901, max=1500
```

查询时只扫描可能包含目标数据的块范围，跳过无关块。

## 适用场景

| 场景 | 是否适合 | 原因 |
|---|---|---|
| 时序数据（时间戳排序插入） | ✅ 非常适合 | 数据天然有序 |
| IoT 传感器数据 | ✅ 非常适合 | 追加写入，物理顺序与业务顺序一致 |
| 日志表 | ✅ 非常适合 | 按时间追加，极少更新 |
| UUID 主键无序插入 | ❌ 不适合 | 数据分散，BRIN 退化 |
| 频繁 UPDATE 的列 | ❌ 不适合 | 索引维护成本高 |

## 与 B-Tree 对比

```
B-Tree 大小: 100% (基准)
BRIN 大小:  约 0.1% ~ 1%

B-Tree 写入开销: 高（每行更新索引）
BRIN 写入开销: 极低（每块范围更新摘要）

B-Tree 精确匹配: O(log n)
BRIN 精确匹配: O(log m + 部分扫描)  m = 块范围数
```

## PG17 多值 MIN/MAX 聚合

PG17 优化了 BRIN 索引的创建速度——使用了多值 MIN/MAX 聚合（Multi-Value MIN/MAX Aggregate），创建索引时扫描更快。创建语法无变化，后台自动受益：

```sql
-- 创建 BRIN 索引（推荐指定 pages_per_range）
CREATE INDEX idx_orders_created_at ON orders USING brin(created_at)
  WITH (pages_per_range = 32);

-- 创建一个自动汇总的 BRIN 索引
CREATE INDEX idx_orders_status ON orders USING brin(status)
  WITH (autosummarize = on);
```

## PG18 BRIN 多范围索引（Multi-Range）

PG18 引入了 Multi-Range BRIN，允许一个索引列包含不连续的块范围，解决了 UPDATE/DELETE 后索引空洞问题：

```sql
-- PG18+ 创建多范围 BRIN 索引
CREATE INDEX idx_orders_multi ON orders USING brin(created_at)
  WITH (multi_range = on);
```

传统 BRIN 假设数据是连续的块范围，UPDATE 或 DELETE 后中间空洞导致精度下降。Multi-Range 允许索引跟踪分散的块组，保持查询精度。

## 最佳实践

### 1. pages_per_range 调优

```sql
-- 小表（< 100MB）用较小值
CREATE INDEX idx_log_time ON access_log USING brin(created_at)
  WITH (pages_per_range = 4);

-- 大表（> 10GB）用较大值
CREATE INDEX idx_log_time ON access_log USING brin(created_at)
  WITH (pages_per_range = 64);
```

经验法则：pages_per_range = 目标每个索引条目覆盖约 1MB 数据。

### 2. 复合 BRIN 索引

```sql
-- 对经常一起查询的列创建复合 BRIN
CREATE INDEX idx_sensor_ts_loc ON sensor_data USING brin(ts, location_id)
  WITH (pages_per_range = 32);
```

### 3. autosummarize（自动汇总）

```sql
-- 对于频繁插入但不重建索引的表，开启自动汇总
CREATE INDEX idx_events_time ON events USING brin(created_at)
  WITH (autosummarize = on);
```

不加 autosummarize 时，新插入的数据不会立即被索引覆盖，直到块范围被手动汇总或索引膨胀。

### 4. 手动维护

```sql
-- 手动汇总最近的块范围
SELECT brin_summarize_range('idx_events_time', last_consistent_range);

-- 或指定具体页范围
SELECT brin_summarize_range('idx_events_time', 1000);
```

## 监控 BRIN 效率

```sql
-- 查看索引统计
SELECT * FROM pg_stat_all_indexes 
WHERE indexrelname = 'idx_orders_created_at';

-- 检查索引大小
SELECT pg_size_pretty(pg_relation_size('idx_orders_created_at'));
```

## 注意事项

- **BRIN 不是银弹**：如果查询总是扫描大部分数据（>10%），B-Tree 可能更好
- **数据必须有序写入**：插入顺序与业务查询列不一致时，BRIN 效率急剧下降
- **UPDATE 破坏顺序**：随机 UPDATE 会导致索引退化，需定期重建
- **NULL 处理**：BRIN 也会记录 NULL，但空值过多时浪费页范围描述
- **PG18 multi_range 目前是实验性**：生产环境谨慎使用
- **组合使用**：时序表通常用 BRIN(时间) + B-Tree(实体ID) 的组合索引方案
