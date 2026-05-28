---
name: seata-practice
description: Seata 分布式事务最佳实践：AT、TCC、Saga 与 XA 模式实战
tags: [spring-cloud, seata, distributed-transaction, tcc, saga, at]
---

## 概述

Seata 是阿里巴巴开源的分布式事务解决方案，提供 AT、TCC、Saga、XA 四种模式。生产环境中 AT 模式使用最广，TCC 适用于高性能场景，Saga 适用于长事务。

## 四种模式选型

| 模式 | 原理 | 适用场景 | 性能 | 侵入性 |
|------|------|----------|------|--------|
| **AT** | 自动生成反向 SQL 回滚 | CRUD 为主的业务 | 中 | 低（加注解即可） |
| **TCC** | Try-Confirm-Cancel 三段式 | 资源预留型业务（库存、账户） | 高 | 高（需手写三方法） |
| **Saga** | 正向补偿 + 逆向补偿 | 长流程、编排型业务 | 高 | 中 |
| **XA** | 数据库 XA 协议 | 强一致性要求、短事务 | 低 | 低 |

## AT 模式最佳实践

### 1. 基础配置

```yaml
seata:
  enabled: true
  application-id: ${spring.application.name}
  tx-service-group: my_tx_group
  service:
    vgroup-mapping:
      my_tx_group: default
    grouplist:
      default: 127.0.0.1:8091
  config:
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      namespace: seata
      group: SEATA_GROUP
  registry:
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      namespace: seata
      group: SEATA_GROUP
```

### 2. 数据源代理（关键）

Seata AT 模式要求数据源被代理：

```java
@Configuration
public class SeataConfig {
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DataSource dataSource() {
        return new DruidDataSource();
    }

    @Bean
    public DataSourceProxy dataSourceProxy(DataSource dataSource) {
        return new DataSourceProxy(dataSource);
    }

    @Bean
    public SqlSessionFactory sqlSessionFactory(DataSourceProxy dataSourceProxy) throws Exception {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
        factory.setDataSource(dataSourceProxy);
        return factory.getObject();
    }
}
```

### 3. 全局事务注解

```java
@Service
public class OrderService {
    @GlobalTransactional(name = "create-order", rollbackFor = Exception.class)
    public void createOrder(OrderDTO order) {
        // 1. 扣减库存（远程调用）
        storageService.deduct(order.getProductId(), order.getCount());
        // 2. 创建订单（本地事务）
        orderMapper.insert(order);
        // 3. 扣减账户余额（远程调用）
        accountService.debit(order.getUserId(), order.getAmount());
    }
}
```

### 4. undo_log 表（AT 模式必须）

```sql
-- 每个业务库都需要
CREATE TABLE IF NOT EXISTS `undo_log` (
  `id` BIGINT(20) NOT NULL AUTO_INCREMENT,
  `branch_id` BIGINT(20) NOT NULL,
  `xid` VARCHAR(128) NOT NULL,
  `context` VARCHAR(128) NOT NULL,
  `rollback_info` LONGBLOB NOT NULL,
  `log_status` INT(11) NOT NULL,
  `log_created` DATETIME NOT NULL,
  `log_modified` DATETIME NOT NULL,
  `ext` VARCHAR(100) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_unionkey` (`xid`,`branch_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## TCC 模式实战

### 1. TCC 接口定义

```java
@LocalTCC
public interface StorageTccAction {
    @TwoPhaseBusinessAction(
        name = "deduct",
        commitMethod = "confirm",
        rollbackMethod = "cancel"
    )
    boolean prepare(BusinessActionContext ctx,
                    @BusinessActionContextParameter(paramName = "productId") Long productId,
                    @BusinessActionContextParameter(paramName = "count") Integer count);

    boolean confirm(BusinessActionContext ctx);
    boolean cancel(BusinessActionContext ctx);
}
```

### 2. TCC 实现

```java
@Service
public class StorageTccActionImpl implements StorageTccAction {
    @Autowired
    private StorageMapper storageMapper;

    @Override
    public boolean prepare(BusinessActionContext ctx, Long productId, Integer count) {
        // Try：预扣库存（冻结库存）
        int updated = storageMapper.freezeStock(productId, count);
        if (updated == 0) {
            throw new RuntimeException("库存不足");
        }
        return true;
    }

    @Override
    public boolean confirm(BusinessActionContext ctx) {
        // Confirm：确认扣减（将冻结库存扣减）
        Long productId = Long.parseLong(ctx.getActionContext("productId").toString());
        Integer count = Integer.parseInt(ctx.getActionContext("count").toString());
        storageMapper.confirmDeduct(productId, count);
        return true;
    }

    @Override
    public boolean cancel(BusinessActionContext ctx) {
        // Cancel：释放冻结库存
        Long productId = Long.parseLong(ctx.getActionContext("productId").toString());
        Integer count = Integer.parseInt(ctx.getActionContext("count").toString());
        storageMapper.unfreezeStock(productId, count);
        return true;
    }
}
```

### 3. 幂等控制

```java
// confirm/cancel 必须幂等
@Transactional
public boolean confirm(BusinessActionContext ctx) {
    // 使用唯一键防重
    int inserted = confirmLogMapper.insertIgnore(ctx.getXid());
    if (inserted == 0) {
        return true; // 已执行过，直接返回成功
    }
    // 实际业务操作...
}
```

## 避坑指南

### 1. 本地事务与全局事务混用

```java
@GlobalTransactional
public void createOrder(OrderDTO order) {
    // ❌ 错误：内部方法有自己的 @Transactional，容易导致锁未释放
    // ✅ 正确：去掉内部方法的 @Transactional，让 Seata 管理
    doCreateOrder(order);
}
```

### 2. 隔离级别与脏读

AT 模式默认**读未提交**（Read Uncommitted），因为 Seata 使用**写隔离**（全局锁）而非快照读。解决方式：

```java
// 方案一：使用 SELECT ... FOR UPDATE 加全局锁
@Select("SELECT * FROM storage WHERE product_id = #{productId} FOR UPDATE")
Storage selectForUpdate(Long productId);

// 方案二：依赖 Seata 全局锁（写操作自动加锁，读不加锁时可能读到未提交数据）
// 业务可接受时无需处理
```

### 3. 超时与重试

```yaml
seata:
  client:
    tm:
      default-global-transaction-timeout: 60000  # 默认60s，根据业务调整
    rm:
      retry:
        interval: 1000    # 重试间隔（毫秒）
        times: 5          # 重试次数
```

### 4. 日志清理

```sql
-- undo_log 只会保留到全局事务结束，但如果 TC 异常会残留
-- 建议定时清理：
DELETE FROM undo_log WHERE log_created < DATE_SUB(NOW(), INTERVAL 7 DAY);
```

### 5. 性能调优

```yaml
seata:
  client:
    rm:
      lock:
        retry-interval: 10     # 获取全局锁重试间隔（毫秒），默认10
        retry-times: 30        # 重试次数，默认30
      report-success-enable: false  # 关闭成功上报可提升性能
```

## 关键注意事项

1. **AT 模式不支持跨数据库 JOIN**：AT 模式的回滚基于快照，多个数据源无法保证全局一致性读
2. **TCC 的 Cancel 必须幂等**：TC 会重试 Cancel 直到成功
3. **Saga 模式无隔离性**：Saga 不做隔离保护，需业务层自行处理
4. **不要用 Seata 做短事务**：< 50ms 的事务用 Seata 会增加 3-5ms 延迟，不如用本地事务
5. **表必须有主键**：AT 模式依赖主键生成反向 SQL
6. **字段不能为 BLOB/TEXT 以外的 LOB 类型**：回滚 SQL 生成可能失败
