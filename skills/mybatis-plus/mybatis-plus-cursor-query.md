---
name: mybatis-plus-cursor-query
description: MyBatis-Plus 游标查询 selectWithCursor 处理大数据集避免 OOM
tags: [mybatis-plus, orm, cursor, query, large-data]
---

## 概述

MyBatis-Plus 3.5.16+ 新增 `selectWithCursor` 游标查询方法，用于**分批流式读取**大数据集，避免一次性加载全部数据导致内存溢出（OOM）。适用于数据迁移、批量导出、ETL 等场景。

> 传统 `selectList` 会将所有结果加载到内存，百万级数据时极易 OOM。游标查询通过数据库游标机制逐批拉取，内存占用恒定。

## 游标查询 API

### 1. BaseMapper 接口

```java
public interface BaseMapper<T> {

    /**
     * 游标查询 — 流式返回数据
     * @param queryWrapper 查询条件
     * @param consumer     逐条处理回调（游标内执行）
     */
    void selectWithCursor(@Param("ew") Wrapper<T> queryWrapper,
                          Consumer<T> consumer);
}
```

### 2. IService 链式调用

```java
// 通过 lambdaQuery 链式触发游标查询
List<Result> results = new ArrayList<>(1000);
userService.lambdaQuery()
    .ge(User::getCreateTime, LocalDate.now().minusDays(7))
    .cursor(item -> {
        // 逐条处理，不会撑爆内存
        Result r = transform(item);
        results.add(r);
        if (results.size() >= 1000) {
            batchSave(results);  // 满一批写入
            results.clear();
        }
    });
```

## 使用场景与示例

### 场景1：全量数据导出 CSV

```java
@Service
public class DataExportService {

    @Autowired
    private UserMapper userMapper;

    public void exportUsers(OutputStream outputStream) {
        try (CSVWriter writer = new CSVWriter(
                new OutputStreamWriter(outputStream, StandardCharsets.UTF_8))) {

            // 写入表头
            writer.writeNext(new String[]{"ID", "姓名", "邮箱", "注册时间"});

            // 游标查询 — 百万级用户也不会 OOM
            userMapper.selectWithCursor(
                new LambdaQueryWrapper<User>()
                    .select(User::getId, User::getName,
                            User::getEmail, User::getCreateTime)
                    .orderByAsc(User::getId),
                user -> writer.writeNext(new String[]{
                    String.valueOf(user.getId()),
                    user.getName(),
                    user.getEmail(),
                    user.getCreateTime().toString()
                })
            );
        } catch (IOException e) {
            throw new RuntimeException("导出失败", e);
        }
    }
}
```

### 场景2：数据迁移/清洗

```java
@Service
public class DataMigrationService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private OrderArchiveMapper archiveMapper;

    @Transactional
    public void migrateOldOrders(LocalDate cutoff) {
        List<OrderArchive> batch = new ArrayList<>(500);

        orderMapper.selectWithCursor(
            new LambdaQueryWrapper<Order>()
                .lt(Order::getCreateTime, cutoff)
                .orderByAsc(Order::getId),
            order -> {
                OrderArchive archive = convert(order);
                batch.add(archive);

                if (batch.size() >= 500) {
                    archiveMapper.insertBatchSomeColumn(batch);
                    batch.clear();
                }
            }
        );

        // 处理剩余批次
        if (!batch.isEmpty()) {
            archiveMapper.insertBatchSomeColumn(batch);
        }
    }

    private OrderArchive convert(Order order) {
        // 清洗逻辑
        return OrderArchive.builder()
            .orderId(order.getId())
            .userId(order.getUserId())
            .amount(order.getAmount())
            .archivedAt(LocalDateTime.now())
            .build();
    }
}
```

### 场景3：大数据量计算

```java
@Service
public class StatisticsService {

    public Map<String, Long> calculateTagDistribution() {
        Map<String, Long> tagCounts = new ConcurrentHashMap<>();

        productMapper.selectWithCursor(
            new LambdaQueryWrapper<Product>()
                .select(Product::getTags)
                .isNotNull(Product::getTags),
            product -> {
                // 每个产品多个标签，累加统计
                String[] tags = product.getTags().split(",");
                for (String tag : tags) {
                    tagCounts.merge(tag.trim(), 1L, Long::sum);
                }
            }
        );

        return tagCounts;
    }
}
```

## 实现原理

```text
执行流程：
1. MyBatis 开启游标（Cursor）模式
2. 数据库创建游标（ResultSet.TYPE_FORWARD_ONLY + fetchSize=Integer.MIN_VALUE）
3. MyBatis 逐条从游标读取，调用 Consumer
4. 读取完毕自动关闭游标和 Statement

关键点：
- fetchSize=Integer.MIN_VALUE 触发 MySQL 流式读取
- PostgreSQL 游标模式自动启用
- Oracle 使用 fetchSize 控制批次大小
```

## 与 Cursor 接口的对比

| 特性 | `selectWithCursor` | MyBatis `Cursor<T>` |
|------|-------------------|---------------------|
| 返回方式 | Consumer 回调 | 返回 Cursor 对象 |
| 事务管理 | 自动管理 | 需手动保证事务 |
| 资源释放 | 自动关闭 | 需 try-with-resources |
| 中断控制 | 无（可抛异常） | isOpen() / close() |
| 适合场景 | 数据导出/迁移 | 需要手动控制迭代 |

## 注意事项

1. **必须在事务内执行**：游标查询需要保持数据库连接，`selectWithCursor` 已自动处理，但调用方应在 `@Transactional` 范围内
2. **Consumer 内不要持有数据库连接**：Consumer 回调已占用一条 JDBC 连接，内部不要再调用数据库操作，否则可能耗尽连接池
3. **避免在 Consumer 内执行复杂操作**：Consumer 在游标的迭代上下文中执行，长时间阻塞会影响数据库游标超时
4. **批量操作使用批处理**：如上例所示，每 N 条批量写入一次，不要逐条 insert
5. **MySQL fetchSize 配置**：MySQL 需设置 `fetchSize=Integer.MIN_VALUE` 触发流式读取，MyBatis-Plus 已自动配置
6. **分页 vs 游标**：分页查询在深分页时性能急剧下降（OFFSET 越大越慢），游标查询始终保持 O(1) 内存

```properties
# application.yml — MySQL 游标配置
spring:
  datasource:
    hikari:
      # 游标查询需要较长的连接保活时间
      maximum-pool-size: 20
      idle-timeout: 600000
      max-lifetime: 1800000
```
