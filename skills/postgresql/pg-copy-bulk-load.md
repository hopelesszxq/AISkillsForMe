---
name: pg-copy-bulk-load
description: PostgreSQL COPY 批量数据加载性能优化：Binary 格式、并行 COPY、错误处理、pg_bulkload 与生产实战
tags: [postgresql, copy, bulk-load, performance, etl, data-import, binary, parallel]
---

## 概述

PostgreSQL 的 `COPY` 命令是最高效的数据批量导入方式，比逐行 `INSERT` 快 **10~50 倍**。本文覆盖从单表 COPY 到 TB 级数据加载的完整优化方案。

### COPY 与 INSERT 对比

| 方式 | 10 万行耗时 | 100 万行耗时 | 特点 |
|------|-----------|------------|------|
| 逐行 INSERT | ~3-5s | ~30-60s | 每行一次事务 |
| 批量 INSERT (1000/batch) | ~0.5-1s | ~5-10s | 减少事务开销 |
| **COPY** | **~0.1-0.3s** | **~1-3s** | 批量协议，最优路径 |
| **COPY Binary** | **~0.08-0.2s** | **~0.8-2s** | 避免文本解析 |

## 一、基础用法与性能差异

### 1. Text 格式

```sql
-- 标准 COPY，文本格式默认逗号分隔
COPY orders FROM '/data/orders.csv' WITH (FORMAT CSV, HEADER true, DELIMITER ',');

-- 带 NULL 处理
COPY products FROM '/data/products.csv' WITH (
    FORMAT CSV,
    HEADER true,
    NULL 'NULL',           -- CSV 中的 NULL 字符串
    FORCE_NOT_NULL (price) -- 空字符串不转换为 NULL
);
```

### 2. Binary 格式（推荐大批量）

```sql
-- Binary 格式，减少解析开销，速度提升 20-40%
COPY orders FROM '/data/orders.bin' WITH (FORMAT BINARY);

-- 导出 Binary
COPY orders TO '/data/orders.bin' WITH (FORMAT BINARY);
```

> **Binary 格式要求**：字段类型与表定义完全匹配，无法指定分隔符和 NULL 字符串。适合程序化导出的 ETL 场景。

## 二、并行 COPY 策略

### 1. 多路并行（手动分区）

```bash
#!/bin/bash
# 按 ID 范围拆分数据文件，并行 COPY
# 假设 orders_2024.csv 有 2 亿行

split -l 5000000 orders_2024.csv orders_chunk_  # 40 个文件

for f in orders_chunk_*; do
    psql -c "\COPY orders FROM '$f' WITH (FORMAT CSV)" &
    
    # 控制并发数
    while [ "$(jobs -r | wc -l)" -ge 4 ]; do
        sleep 0.5
    done
done
wait
```

### 2. pg_bulkload（高性能扩展）

```bash
# 安装
# apt install postgresql-17-pg-bulkload

# pg_bulkload 直接绕过 shared_buffers 写数据文件，比 COPY 快 2-3 倍
pg_bulkload -d mydb -i /data/orders.csv \
  -O orders -l /tmp/bulkload.log \
  -o "DELIMITER=," -o "SKIP=1" \
  -o "WRITE=DIRECT"      # 直接写入数据文件
```

### 3. 分区并行加载

```sql
-- 对分区表，每组 COPY 直接写入对应分区
\COPY orders_2024_q1 FROM '/data/q1.csv' WITH (FORMAT CSV)
\COPY orders_2024_q2 FROM '/data/q2.csv' WITH (FORMAT CSV)
\COPY orders_2024_q3 FROM '/data/q3.csv' WITH (FORMAT CSV)
\COPY orders_2024_q4 FROM '/data/q4.csv' WITH (FORMAT CSV)

-- 推荐：多连接并行执行，每个分区一个连接
```

## 三、错误处理与恢复

### 1. LOG_ERRORS（PostgreSQL 15+）

```sql
-- 记录错误行到错误表，不中断导入
COPY orders FROM '/data/orders.csv' WITH (
    FORMAT CSV,
    HEADER true,
    LOG_ERRORS 'orders_load_errors',
    REJECT_LIMIT '1000'   -- 允许最多 1000 行错误
);

-- 查询错误详情
SELECT * FROM orders_load_errors;
```

### 2. 自定义错误处理

```sql
-- 方式一：先 COPY 到 staging 临时表
CREATE TEMP TABLE staging_orders (LIKE orders);

\COPY staging_orders FROM '/data/orders.csv' WITH (FORMAT CSV)

-- 验证并清洗
INSERT INTO orders (id, product, amount, created_at)
SELECT id, product, amount, created_at
FROM staging_orders
WHERE amount > 0;  -- 过滤无效数据

-- 方式二：使用 NOT VALID + 后续校验
SET session_replication_role = replica;  -- 临时禁用触发器
\COPY orders FROM '/data/orders.csv' WITH (FORMAT CSV)
SET session_replication_role = origin;

-- 事后检查
SELECT id FROM orders WHERE amount IS NULL;
```

## 四、大事务策略

### 1. 分批提交（避免膨胀）

```bash
# 使用脚本分批导入，每批单独提交
for i in $(seq 0 9); do
    start=$((i * 1000000 + 1))
    end=$(( (i + 1) * 1000000 ))
    psql -c "
        BEGIN;
        COPY orders FROM PROGRAM 'sed -n ''${start},${end}p'' /data/orders.csv' WITH (FORMAT CSV);
        COMMIT;
    "
done
```

### 2. 临时禁用索引和约束

```sql
-- 大批量加载前：删除索引 → 加载 → 重建
DROP INDEX IF EXISTS idx_orders_product;
DROP INDEX IF EXISTS idx_orders_created;

\COPY orders FROM '/data/orders.csv' WITH (FORMAT CSV)

-- 加载完成后重建（比维持索引更新快得多）
CREATE INDEX CONCURRENTLY idx_orders_product ON orders(product);
CREATE INDEX CONCURRENTLY idx_orders_created ON orders(created_at);

-- 同样适用于外键约束
ALTER TABLE orders DISABLE TRIGGER ALL;
\COPY orders FROM '/data/orders.csv' WITH (FORMAT CSV)
ALTER TABLE orders ENABLE TRIGGER ALL;
```

### 3. 调整 maintenance_work_mem

```sql
-- 重建索引时分配更多内存
SET maintenance_work_mem = '2GB';

-- COPY 过程本身的 WAL 控制
-- 注意：仅限可以丢失的批量导入场景
BEGIN;
SET local wal_level = 'minimal';
SET local archive_mode = 'off';
-- COPY 操作
COMMIT;
```

## 五、COPY FROM PROGRAM

```bash
# 直接从外部命令管道输入
# 解压流式导入，无需中间文件
COPY orders FROM PROGRAM 'zcat /data/orders.csv.gz' WITH (FORMAT CSV);

# 远程文件通过 curl
COPY orders FROM PROGRAM 'curl -s https://data.example.com/export/orders.csv' WITH (FORMAT CSV);

# 流式变换
COPY orders FROM PROGRAM 'sed "s/\\r//g" /data/orders_dirty.csv' WITH (FORMAT CSV);
```

## 六、pg17+ 新特性

### 1. COPY TO 分区表直接导出 (pg17+)

```sql
-- pg17+：直接 COPY 分区表，无需 SELECT 子查询
COPY partitioned_orders TO '/data/all_orders.csv' WITH (FORMAT CSV, HEADER true);
```

### 2. COPY 并行读取 (pg18+)

```sql
-- pg18+：COPY 支持并行读取输入文件（多块同时读取）
COPY orders FROM '/data/orders.csv' WITH (
    FORMAT CSV,
    HEADER true,
    PARALLEL 4       -- 4 个 worker 并行读取
);
```

### 3. 带校验的 COPY (pg19+)

```sql
-- pg19+：支持在 COPY 过程中执行简单的数据校验
COPY orders FROM '/data/orders.csv' WITH (
    FORMAT CSV,
    CHECK (amount > 0 AND created_at IS NOT NULL)
);
```

## 七、生产级完整示例

```bash
#!/bin/bash
# 生产环境大数据量 ETL 脚本
# 适用于 1 亿+ 行级别的批量导入

DB="analytics"
TABLE="fact_orders"
FILE="/data/orders_export.csv"
THREADS=8

# 1. 准备：清空临时区
psql -d "$DB" -c "TRUNCATE ${TABLE}_staging;"

# 2. 拆分大文件
echo "Splitting file..."
split -l 5000000 "$FILE" chunk_

# 3. 并行 COPY
echo "Starting parallel COPY..."
for chunk in chunk_*; do
    (
        psql -d "$DB" -c "\COPY ${TABLE}_staging FROM '$(pwd)/${chunk}' WITH (FORMAT CSV, HEADER false, LOG_ERRORS '${TABLE}_errors')"
    ) &
    
    while [ "$(jobs -r | wc -l)" -ge "$THREADS" ]; do
        sleep 1
    done
done
wait

# 4. 数据验证
echo "Validating..."
psql -d "$DB" -c "
    SELECT COUNT(*) AS total_rows,
           COUNT(*) FILTER (WHERE amount <= 0) AS invalid_amount,
           COUNT(*) FILTER (WHERE order_date IS NULL) AS null_date
    FROM ${TABLE}_staging;
"

# 5. 交换分区 / 插入正式表
psql -d "$DB" <<SQL
    BEGIN;
    TRUNCATE ${TABLE};
    INSERT INTO ${TABLE} SELECT DISTINCT * FROM ${TABLE}_staging;
    COMMIT;
SQL

# 6. 重建索引
echo "Rebuilding indexes..."
psql -d "$DB" -c "REINDEX TABLE ${TABLE};"

# 7. 分析统计信息
psql -d "$DB" -c "ANALYZE ${TABLE};"

# 8. 清理临时文件
rm -f chunk_*
echo "Import complete!"
```

## 八、性能调优检查清单

| 配置 | 建议值 | 说明 |
|------|--------|------|
| `max_wal_size` | 32GB+ | 大事务避免 WAL 频繁触发 checkpoint |
| `checkpoint_timeout` | 30-60min | 减少 COPY 期间的 checkpoint 频率 |
| `maintenance_work_mem` | 1-4GB | COPY + 索引重建时使用 |
| `wal_buffers` | 64MB | 写密集时可增大 |
| `autovacuum` | OFF（导入期间） | 避免 autovacuum 与 COPY 争抢 I/O |

## 注意事项

1. **WAL 膨胀**：超大事务（>10GB 数据变更）会导致 WAL 急剧膨胀。使用分批提交（每 500 万行一个事务）控制 checkpoint 间隔。
2. **VACUUM 陷阱**：一个月级大事务后若回滚，需要等量的 VACUUM 清理死元组。建议使用 staging 表模式，失败后直接 TRUNCATE。
3. **COPY Binary 兼容性**：不同大版本 PostgreSQL 间 Binary 格式不兼容（因系统表结构变化）。跨版本迁移用 CSV 或 pg_dump。
4. **错误记录表清理**：`LOG_ERRORS` 创建的错误记录表不会自动清理，大量错误会占用空间。导入成功后及时 DROP。
5. **权限要求**：`COPY FROM` 需要 `pg_read_server_files` 权限；`COPY FROM PROGRAM` 需要超级用户权限。
6. **编码问题**：CSV 文件含 BOM 时需预处理（`sed '1s/^\xEF\xBB\xBF//' file.csv`），否则首列名可能异常。
