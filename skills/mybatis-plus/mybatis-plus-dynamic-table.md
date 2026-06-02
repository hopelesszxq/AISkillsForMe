---
name: mybatis-plus-dynamic-table
description: MyBatis-Plus 动态表名插件 — 分表、多租户、归档场景实战
tags: [mybatis-plus, dynamic-table, sharding, multi-tenant]
---

## 动态表名插件原理

MyBatis-Plus 3.5.x 提供 `DynamicTableNameInnerInterceptor` 拦截器，可在 SQL 执行前动态替换表名，适用于：

- **分表存储**（按年月分表：`orders_202601`, `orders_202602`）
- **多租户分表**（每个租户独立表：`orders_tenant_001`）
- **数据归档**（`orders_active` vs `orders_archive`）
- **读写分离**（同一结构不同表名）

## 配置方式

### 1. Maven 依赖

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
    <version>3.5.10</version>
</dependency>
```

### 2. 注入拦截器

```java
@Configuration
public class MyBatisPlusConfig {

    @Autowired
    private DynamicTableNameHandler tableNameHandler;

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 动态表名拦截器（必须放在分页拦截器之后）
        DynamicTableNameInnerInterceptor dynamicInterceptor = new DynamicTableNameInnerInterceptor();
        dynamicInterceptor.setTableNameHandler(tableNameHandler);
        interceptor.addInnerInterceptor(dynamicInterceptor);

        // 分页拦截器
        PaginationInnerInterceptor pagination = new PaginationInnerInterceptor(DbType.POSTGRES_SQL);
        interceptor.addInnerInterceptor(pagination);

        return interceptor;
    }
}
```

## 场景 1：按年月分表

### 表名处理器

```java
@Component
public class MonthShardingHandler implements DynamicTableNameHandler {

    /**
     * 按年月分表：orders_202601, orders_202602 ...
     *
     * @param sql       原始 SQL
     * @param tableName 原始表名
     * @return 替换后的表名
     */
    @Override
    public String dynamicTableName(String sql, String tableName) {
        // 通过 ThreadLocal 传入分表日期
        String shardDate = ShardingContext.getShardMonth();
        if (shardDate == null) {
            // 默认当前月份
            shardDate = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMM"));
        }

        // 只处理需要分表的表，其余跳过
        if ("orders".equals(tableName)) {
            return tableName + "_" + shardDate;
        }
        if ("order_items".equals(tableName)) {
            return tableName + "_" + shardDate;
        }
        return tableName;
    }
}
```

### ThreadLocal 上下文

```java
public class ShardingContext {

    private static final ThreadLocal<String> SHARD_MONTH = ThreadLocal.withInitial(() ->
        LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMM"))
    );

    public static void setShardMonth(String month) {
        SHARD_MONTH.set(month);
    }

    public static String getShardMonth() {
        return SHARD_MONTH.get();
    }

    public static void clear() {
        SHARD_MONTH.remove();
    }
}
```

### 自动建表（通过定时任务）

```java
@Component
@Slf4j
public class TableAutoCreator {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    /**
     * 每月1号凌晨检查并创建下月分表
     */
    @Scheduled(cron = "0 0 2 1 * ?")
    public void createNextMonthTable() {
        String nextMonth = LocalDate.now()
            .plusMonths(1)
            .format(DateTimeFormatter.ofPattern("yyyyMM"));

        // 检查表是否存在
        String checkSql = "SELECT to_regclass('orders_" + nextMonth + "')";
        Boolean exists = jdbcTemplate.queryForObject(checkSql, Boolean.class);

        if (Boolean.FALSE.equals(exists)) {
            // 以上月分表为模板创建
            String prevMonth = LocalDate.now()
                .format(DateTimeFormatter.ofPattern("yyyyMM"));

            String createSql = String.format(
                "CREATE TABLE orders_%s (LIKE orders_%s INCLUDING ALL)",
                nextMonth, prevMonth);
            jdbcTemplate.execute(createSql);

            log.info("Created shard table: orders_{}", nextMonth);
        }
    }
}
```

### 使用示例

```java
@Service
public class OrderService {

    @Autowired
    private OrderMapper orderMapper;

    public List<Order> queryByMonth(String userId, String month) {
        // 设置分表月份
        ShardingContext.setShardMonth(month);
        try {
            return orderMapper.selectList(Wrappers.<Order>lambdaQuery()
                .eq(Order::getUserId, userId));
        } finally {
            ShardingContext.clear();
        }
    }
}
```

## 场景 2：多租户独立表

### 租户表名处理器

```java
@Component
public class TenantTableHandler implements DynamicTableNameHandler {

    @Override
    public String dynamicTableName(String sql, String tableName) {
        String tenantId = TenantContext.getTenantId();
        if (tenantId == null) {
            return tableName;
        }

        // 租户专属表：orders -> orders_tenant_001
        return tableName + "_tenant_" + tenantId;
    }
}
```

### 结合 MyBatis-Plus 多租户插件的区别

| 特性 | 多租户插件(TenantLineInnerInterceptor) | 动态表名插件 |
|---|---|---|
| 实现方式 | SQL 追加 `WHERE tenant_id = ?` | 替换表名 |
| 表结构 | 共用同一张表 | 每个租户独立表 |
| 数据隔离 | 逻辑隔离 | 物理隔离 |
| 备份恢复 | 需按 tenant_id 过滤 | 可直接备份单张表 |
| 适用场景 | SaaS 中小规模 | 金融、合规场景 |

## 场景 3：冷热数据分离

```java
@Component
public class HotArchiveHandler implements DynamicTableNameHandler {

    @Override
    public String dynamicTableName(String sql, String tableName) {
        // 只处理归档表
        if (!"orders".equals(tableName)) {
            return tableName;
        }

        // 读取配置：当前查询是热数据还是归档数据
        ArchiveMode mode = ArchiveContext.getMode();
        if (mode == ArchiveMode.ARCHIVE) {
            return "orders_archive";
        }
        return "orders_active";
    }
}
```

## 与分页插件配合

```java
// 动态表名 + 分页必须：先添加分页拦截器，再添加动态表名
// 顺序不能错！

@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

    // 1. 分页（先执行）
    interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.POSTGRES_SQL));
    // 2. 动态表名（后执行，用于替换分页 SQL 中的表名）
    interceptor.addInnerInterceptor(new DynamicTableNameInnerInterceptor()
        .setTableNameHandler(monthShardingHandler));

    return interceptor;
}
```

## 注意事项

- **性能开销**：每次 SQL 执行都会回调 tableNameHandler，handler 中不要做 I/O 操作
- **CRUD 全拦截**：拦截器会拦截 select/insert/update/delete 全部 SQL，注意覆盖所有场景
- **分页兼容**：动态表名拦截器必须在分页拦截器之后添加，否则分页 count 查询中的表名可能未被替换
- **Mapper XML**：如果使用 XML 自定义 SQL，同样会被拦截替换，无需额外处理
- **测试覆盖**：分表场景务必对增删改查 + 分页 + 关联查询做完整单元测试
- **自动建表**：分表策略必须有自动建表机制，避免运行时表不存在报错
- **事务问题**：跨分表的分布式事务需引入 Seata 等方案，单分表内的本地事务不受影响
- **数据迁移**：分表合并或拆分需准备数据迁移脚本，推荐使用 PostgreSQL FDW 或 ETL 工具
