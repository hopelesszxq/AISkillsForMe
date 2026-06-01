---
name: seata-at-tuning
description: Seata AT 模式深度调优：undo log 清理、全局锁争用、SQL 解析优化与 Spring Boot 3 集成
tags: [spring-cloud, seata, at, performance, tuning, undo-log, global-lock]
---

## 概述

Seata AT 模式虽然自动化程度高（自动生成回滚 SQL），但生产环境中常遇到**全局锁超时、undo log 膨胀、SQL 解析失败**等问题。本文深入 AT 模式的性能调优与避坑。

## 1. Undo Log 管理

### 表结构与清理策略

AT 模式的 undo_log 表是核心，生产环境必须定期清理：

```sql
-- undo_log 表结构（需手动建表）
CREATE TABLE IF NOT EXISTS `undo_log` (
    `id`             BIGINT(20)   NOT NULL AUTO_INCREMENT,
    `branch_id`      BIGINT(20)   NOT NULL,
    `xid`            VARCHAR(128) NOT NULL,
    `context`        VARCHAR(128) NOT NULL,
    `rollback_info`  LONGBLOB     NOT NULL,
    `log_status`     TINYINT(4)   NOT NULL,
    `log_created`    DATETIME     NOT NULL,
    `log_modified`   DATETIME     NOT NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_undo_log` (`xid`, `branch_id`)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4;

-- 定时清理：删除已处理的 undo_log（log_status=1 表示已回滚/已提交）
DELETE FROM undo_log
WHERE log_created < NOW() - INTERVAL 7 DAY
  AND log_status = 1;
```

### Undo Log 保留时间配置

```yaml
seata:
  client:
    undo:
      log-table: undo_log           # 自定义表名
      data-validation: true         # SQL 回滚前校验镜像
      log-serialization: jackson    # 序列化方式：jackson / protostuff
      only-care-update-columns: true # 仅关心更新列（减少 undo log 大小）
```

### 大事务 Undo Log 优化

```java
// ❌ 问题：一个事务更新 10000 行 → undo_log 巨大
@GlobalTransactional
public void batchUpdate(List<Long> ids) {
    ids.forEach(id -> userMapper.update(id, newData));
}

// ✅ 优化：拆分为批量 SQL，每条 SQL 只产生一条 undo log
@GlobalTransactional
public void batchUpdate(List<Long> ids) {
    userMapper.batchUpdateByIds(ids, newData);
}
```

批量更新 SQL 示例：

```xml
<update id="batchUpdateByIds">
    UPDATE users SET status = #{status}, updated_at = now()
    WHERE id IN
    <foreach collection="ids" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</update>
```

## 2. 全局锁调优

### 锁超时配置

```yaml
seata:
  client:
    lock:
      retry-interval: 10            # 重试间隔（ms），默认 10
      retry-times: 30               # 重试次数，默认 30
      retry-policy-branch-rollback-on-conflict: true  # 分支冲突时等待后重试
```

### 减少锁冲突的设计

```java
// ❌ 问题：大事务锁定多张表，高并发下锁冲突剧烈
@GlobalTransactional
public void createOrder(OrderDTO dto) {
    orderMapper.insert(dto.getOrder());        // 锁订单表
    inventoryMapper.deduct(dto.getSku(), 1);   // 锁库存表
    accountMapper.debit(dto.getUserId(), 100); // 锁账户表
}

// ✅ 优化：按锁冲突概率排序资源
// 原则：将争用最激烈的资源放在最后锁定，持有时间最短
@GlobalTransactional
public void createOrder(OrderDTO dto) {
    // 1. 先操作争用低的资源
    orderMapper.insert(dto.getOrder());

    // 2. 再操作争用中的资源
    accountMapper.debit(dto.getUserId(), 100);

    // 3. 最后操作争用最激烈的资源（持有锁最短）
    inventoryMapper.deduct(dto.getSku(), 1);
}
```

### 查看当前锁等待

```sql
-- MySQL 查看 Seata 全局锁等待
SELECT * FROM INFORMATION_SCHEMA.INNODB_TRX
WHERE trx_query LIKE '%undo_log%'
   OR trx_query LIKE '%lock_table%';

-- Seata Server 查询活跃锁
-- GET http://seata-server:7091/api/v1/transaction/lock/query
```

## 3. SQL 解析常见问题

AT 模式依赖 SQL 解析器自动生成镜像查询，以下场景需要特别注意：

### INSERT 无主键回滚

```java
// ❌ 无主键 → Seata 无法生成精确的回滚镜像
INSERT INTO log (message, level) VALUES ('test', 'INFO');

// ✅ 必须有主键或唯一索引
@Entity
@Table(name = "log")
public class LogEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;          // 必须有主键，Seata 用主键定位行
    private String message;
    private String level;
}
```

### 不支持 SQL 类型

| SQL 类型 | 支持情况 | 替代方案 |
|----------|----------|---------|
| `INSERT ... ON DUPLICATE KEY UPDATE` | ❌ 不支持 | 拆分为 SELECT + INSERT/UPDATE |
| `REPLACE INTO` | ❌ 不支持 | 同上 |
| `UPDATE ... LIMIT` | ❌ 不支持 | 精确 WHERE 条件 |
| `DELETE ... JOIN` | ❌ 不支持 | 拆分为两条 SQL |
| `ALTER TABLE` | ❌ 不支持 | 人工管控 |
| **多表 JOIN UPDATE** | ⚠️ 部分支持 | 确认主表有主键 |

### SQL 解析日志排查

```yaml
# 开启 SQL 解析日志（用于排查解析失败）
logging:
  level:
    io.seata: DEBUG
    io.seata.sqlparser: TRACE    # SQL 解析详细日志
```

## 4. Spring Boot 3 + Virtual Threads 适配

```yaml
# Spring Boot 3.2+ 虚拟线程配置
spring:
  threads:
    virtual:
      enabled: true

seata:
  enabled: true
  application-id: ${spring.application.name}
  tx-service-group: my_tx_group
  client:
    rm:
      # 虚拟线程下建议增加重试次数
      report-retry-count: 10
```

### 数据源代理关键配置

```java
@Configuration
public class SeataDataSourceConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DataSource dataSource() {
        return new DruidDataSource();
    }

    @Bean
    public DataSource seataDataSource(DataSource dataSource) {
        // 必须使用 Seata 代理数据源
        return new DataSourceProxy(dataSource);
    }
}
```

## 5. 监控与诊断

### Prometheus 指标

`seata-server` 暴露以下关键指标：

| 指标 | 含义 | 告警阈值 |
|------|------|---------|
| `seata_global_lock_count` | 当前全局锁数量 | > 1000 |
| `seata_global_lock_wait_time_ms` | 全局锁平均等待时间 | > 500ms |
| `seata_branch_rollback_total` | 分支回滚总数 | 持续增长需排查 |
| `seata_global_session_active` | 活跃全局事务数 | > 200 |

### 常见问题排查

1. **Undo log 太大**：开启 `only-care-update-columns: true`，仅记录变更列
2. **全局锁频繁超时**：检查事务内是否有远程调用或长时间计算
3. **回滚失败**：检查 `log_status=0`（未处理）的 undo_log，手动清理
4. **性能骤降**：检查 `SELECT FOR UPDATE` 导致的全局锁堆积

## 注意事项

- **不要在 @GlobalTransactional 内做远程 RPC 调用**——会长时间持有全局锁
- **XA 模式 vs AT 模式**：XA 强一致性但性能低，AT 适合>99%的场景
- **Seata Server 高可用**：至少部署 2 个 Server 节点使用 Nacos 或 DB 模式存储
- **Spring Boot 4.x**：Seata 需升级至 2.6.x+ 以兼容 Jakarta EE 11
