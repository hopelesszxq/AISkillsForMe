---
name: pg-repack
description: PostgreSQL pg_repack 在线表维护——无锁整理表空间、回收膨胀空间、重建索引
tags: [postgresql, maintenance, repack, vacuum, bloat, index, online-ddl]
---

## 概述

`pg_repack` 是一个 PostgreSQL 扩展，能够在**不持有排他锁**的情况下重建表和索引，回收膨胀（bloat）空间、整理碎片。与 `VACUUM FULL` 不同，`pg_repack` 在重组期间**不会阻塞并发读写**。

### 解决的核心问题

- 长时间运行的表产生大量死元组和膨胀
- `VACUUM` 只能标记空间可复用，不能收缩表文件
- `VACUUM FULL` 需要 `ACCESS EXCLUSIVE` 锁，阻塞所有读写
- 需要在线整理 7×24 小时运行的表

## 工作原理

```
┌─────────────┐    1. 创建影子表（与原表结构相同）
│  原表       │    2. 在影子表上创建索引
│  (旧文件)   │    3. 将原表数据批量复制到影子表
└──────┬──────┘    4. 记录复制期间的增量变更（触发器）
       │           5. EXCHANGE 阶段：在短暂锁下切换表名
       ▼           6. 删除原表文件（释放磁盘空间）
┌─────────────┐
│  影子表     │    ← 新表（无膨胀）
│  (新文件)   │
└─────────────┘
```

**锁时间**：仅最终 `EXCHANGE` 阶段需要持有 `ACCESS EXCLUSIVE` 锁，通常仅需**几毫秒到几秒**。

## 安装

```bash
# 编译安装
git clone https://github.com/reorg/pg_repack.git
cd pg_repack
make && sudo make install

# 创建扩展
psql -d mydb -c "CREATE EXTENSION pg_repack;"
```

### 依赖

- PostgreSQL 9.4+
- `pg_repack` 版本需与 PostgreSQL 大版本匹配
- 需要 `superuser` 或 `pg_repack` 角色权限

## 命令行使用

### 基本命令

```bash
# 在线整理指定表
pg_repack -h localhost -p 5432 -U postgres -d mydb -t orders

# 整理所有数据库
pg_repack -h localhost -U postgres -d mydb --all

# 只整理索引（不移除表膨胀）
pg_repack -h localhost -d mydb --only-indexes

# 只整理指定 schema 的表
pg_repack -h localhost -d mydb --schema public

# 整理多个表
pg_repack -h localhost -d mydb -t orders -t order_items -t payments
```

### 常用选项

| 选项 | 说明 |
|------|------|
| `-t, --table` | 指定要整理的表 |
| `-T, --wait-timeout` | 等待锁的超时时间（秒，默认不限制） |
| `-D, --no-kill-backend` | 不终止阻塞进程 |
| `-N, --exclude-schema` | 排除指定 schema |
| `--only-indexes` | 只重建索引 |
| `--dry-run` | 仅检查，不执行 |
| `-j, --jobs` | 并行工作线程数 |

### 空闲超时设置

```bash
# 如果无法获取锁，等待 30 秒后退出（避免长时间阻塞）
pg_repack -h localhost -d mydb -t busy_table --wait-timeout 30
```

## 编程方式调用

### Java 调用

```java
/**
 * 通过 JDBC 执行 pg_repack 管理表
 */
@Service
public class TableMaintenanceService {

    @Autowired
    private JdbcTemplate jdbc;

    /**
     * 执行在线表整理
     */
    public void repackTable(String tableName) {
        // 方式一：使用 pg_repack 扩展函数
        jdbc.execute("SELECT repack.repack_table('" + tableName + "')");
    }

    /**
     * 只重建索引（更快，锁时间更短）
     */
    public void rebuildIndexes(String tableName) {
        jdbc.execute("SELECT repack.repack_indexes('" + tableName + "')");
    }

    /**
     * 获取表膨胀情况
     */
    public TableBloatInfo getTableBloat(String tableName) {
        return jdbc.queryForObject("""
            SELECT
                schemaname || '.' || tablename AS full_name,
                pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
                ROUND(100 * (CASE WHEN avg_chain_len > 0 THEN 1 - (n_live_tup::numeric / n_dead_tup) ELSE 0 END), 2) AS bloat_pct
            FROM pg_stat_user_tables
            WHERE tablename = ?
            """, (rs, n) -> new TableBloatInfo(
                rs.getString("full_name"),
                rs.getString("total_size"),
                rs.getDouble("bloat_pct")
            ), tableName);
    }

    record TableBloatInfo(String name, String size, double bloatPercent) {}
}
```

### Spring 定时任务

```java
@Component
public class ScheduledTableMaintenance {

    @Autowired
    private JdbcTemplate jdbc;

    /**
     * 每周日凌晨 2 点整理高频写入的表
     */
    @Scheduled(cron = "0 0 2 * * SUN")
    public void weeklyRepack() {
        List<String> tables = List.of("orders", "order_items", "audit_logs");

        for (String table : tables) {
            try {
                log.info("开始在线整理: {}", table);
                jdbc.execute("SELECT repack.repack_table('" + table + "')");
                log.info("整理完成: {}", table);
            } catch (Exception e) {
                log.error("整理失败 {}: {}", table, e.getMessage());
                // 通知运维
            }
        }
    }
}
```

## 实战场景

### 1. 高频写入表的周期性整理

```bash
# crontab：每天凌晨 3 点整理 orders 表
0 3 * * * /usr/bin/pg_repack -h localhost -d mydb -t orders -T 60
```

### 2. 只重建索引（表膨胀不严重时）

```bash
# 索引碎片化严重时，只重建索引比全表 repack 更快
pg_repack -h localhost -d mydb -t orders --only-indexes -j 4
```

### 3. 监控膨胀 → 自动触发整理

```sql
-- 找出膨胀率超过 30% 的表
SELECT
    schemaname || '.' || tablename AS full_name,
    ROUND(100 * pg_table_bloat_ratio(schemaname || '.' || tablename), 1) AS bloat_pct
FROM pg_stat_user_tables
WHERE pg_table_bloat_ratio(schemaname || '.' || tablename) > 30
ORDER BY bloat_pct DESC;
```

## 注意事项

### 1. 磁盘空间要求

```
整理期间所需空间 ≈ 1.5× 目标表（含索引）的大小
```

- `pg_repack` 需要额外磁盘空间存放影子表
- 确保磁盘有足够的剩余空间（建议 2 倍）
- 整理完成后，空间会自动释放

### 2. 锁等待与超时

```bash
# 如果表上有长时间运行的事务，pg_repack 会等待
# 设置超时避免无限等待
pg_repack -d mydb -t orders --wait-timeout 30

# 超时后返回错误，可结合监控告警
```

### 3. 不支持的场景

| 场景 | 原因 | 替代方案 |
|------|------|---------|
| 部分索引（WHERE 子句） | pg_repack 不创建部分索引 | 手动重建 |
| GiST/GIN 索引包含 NULL | 排序问题 | 先 DROP 再 CREATE INDEX CONCURRENTLY |
| 外键引用的表 | 复制阶段可能违反约束 | 暂停外键检查（需排他锁） |
| 物化视图 | 不支持 | DROP + CREATE 或 REFRESH |
| 系统表、临时表 | 内部限制 | 无需整理 |

### 4. 与 VACUUM 的比较

| 操作 | 锁类型 | 阻塞读写 | 释放磁盘 | 碎片整理 | 建议频率 |
|------|--------|---------|---------|---------|---------|
| `VACUUM` | SHARE UPDATE EXCLUSIVE | ❌ 不阻塞 | ❌ 不释放 | ❌ 不整理 | 高频（按需） |
| `VACUUM FULL` | ACCESS EXCLUSIVE | ✅ 阻塞 | ✅ 释放 | ✅ 整理 | 低频（维护窗口） |
| `pg_repack` | 短暂 ACCESS EXCLUSIVE | 仅几毫秒 | ✅ 释放 | ✅ 整理 | 按膨胀率 |

### 5. 生产最佳实践

```bash
# Step 1: 监控膨胀率
# 使用 pg_stat_user_tables + pgstattuple 定期检查

# Step 2: 制定整理策略
# 小表 (< 10GB) → VACUUM FULL（维护窗口）
# 大表 (> 10GB) → pg_repack（在线）
# 超大表 (> 100GB) → 分批 pg_repack + 分区表

# Step 3: 定期执行
# 高频写入的表：每周
# 中等频率的表：每月
# 低频表：按需

# Step 4: 验证结果
# 整理前后对比：
SELECT pg_size_pretty(pg_total_relation_size('orders'));
```

### 6. 安全建议

- **测试环境验证**：先在 staging 环境运行 `--dry-run`
- **监控磁盘空间**：整理期间需额外 1.5× 表大小
- **设置超时**：`--wait-timeout` 避免长时间锁等待
- **避开高峰期**：选择业务低峰期执行
- **保留回滚计划**：`pg_repack` 失败时原表不受影响
