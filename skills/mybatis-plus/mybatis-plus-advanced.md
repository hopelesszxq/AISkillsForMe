---
name: mybatis-plus-advanced
description: MyBatis-Plus 高阶用法：数据权限、动态表名、批量操作与插件扩展
tags: [mybatis-plus, orm, database, permissions]
---

## 数据权限拦截器（DataPermissionInterceptor）

MyBatis-Plus 提供数据权限插件，通过拦截 SQL 自动追加权限过滤条件，无需在每个 Mapper 写重复的 WHERE 子句。

### 自定义数据权限处理器

```java
@Component
public class CustomDataPermissionHandler implements DataPermissionHandler {

    /**
     * 给 SQL 追加权限过滤条件
     * @param where    是否在 WHERE 后追加（false 表示整个 SQL 没有 WHERE）
     * @param mappedStatementId Mapper 方法全限定名
     */
    @Override
    public SqlSegment getSqlSegment(TableInfo tableInfo, Entity where, String mappedStatementId, BoundSql boundSql) {
        // 获取当前用户（从 SecurityContext / ThreadLocal）
        UserInfo user = SecurityContextHolder.getCurrentUser();
        if (user == null || user.isAdmin()) {
            return null; // 管理员不过滤
        }

        // 仅对指定表生效
        String tableName = tableInfo.getTableName();
        switch (tableName) {
            case "sys_order":
                // 业务员只看自己的订单，经理看部门订单
                if ("SALESMAN".equals(user.getRole())) {
                    // WHERE user_id = ?
                    return new SqlSegment(" AND %s.user_id = ", tableName, user.getId());
                } else if ("MANAGER".equals(user.getRole())) {
                    // WHERE dept_id = ?
                    return new SqlSegment(" AND %s.dept_id = ", tableName, user.getDeptId());
                }
                break;

            case "sys_customer":
                // 客户数据按租户隔离：WHERE tenant_id = ?
                return new SqlSegment(" AND %s.tenant_id = ", tableName, user.getTenantId());

            default:
                return null; // 其他表不做数据权限
        }
        return null;
    }
}
```

### 配置拦截器

```java
@Configuration
public class MyBatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 1. 数据权限（需排在分页前面）
        DataPermissionInterceptor dataPermission = new DataPermissionInterceptor();
        dataPermission.setDataPermissionHandler(new CustomDataPermissionHandler());
        interceptor.addInnerInterceptor(dataPermission);

        // 2. 分页
        PaginationInnerInterceptor pagination = new PaginationInnerInterceptor(DbType.POSTGRE_SQL);
        pagination.setMaxLimit(500L); // 防止全表查询
        interceptor.addInnerInterceptor(pagination);

        // 3. 乐观锁
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());

        return interceptor;
    }
}
```

> **注意**：数据权限拦截器要排在分页之前，否则先分页再过滤会导致数据错误。

## 动态表名

适用于分表场景（按年月分表、按租户分表）。

```java
@Component
public class DynamicTableNameHandler implements TableNameHandler {

    private static final ThreadLocal<String> TABLE_SUFFIX = new ThreadLocal<>();

    // 设置表名后缀（在调用 Mapper 前设置）
    public static void setSuffix(String suffix) {
        TABLE_SUFFIX.set(suffix);
    }

    public static void clear() {
        TABLE_SUFFIX.remove();
    }

    @Override
    public String dynamicTableName(String sql, String tableName) {
        String suffix = TABLE_SUFFIX.get();
        if (suffix != null) {
            return tableName + "_" + suffix;  // order_202507
        }
        return tableName;  // 默认不走分表
    }
}

// 配置
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

    DynamicTableNameInnerInterceptor dynamicTable = new DynamicTableNameInnerInterceptor();
    dynamicTable.setTableNameHandler(new DynamicTableNameHandler());
    interceptor.addInnerInterceptor(dynamicTable);

    interceptor.addInnerInterceptor(new PaginationInnerInterceptor());
    return interceptor;
}
```

### 使用示例

```java
@Service
public class OrderServiceImpl extends ServiceImpl<OrderMapper, Order> implements OrderService {

    public Page<Order> queryOrdersByMonth(int page, int size, String yearMonth) {
        // 动态切换到 order_202507 表
        DynamicTableNameHandler.setSuffix(yearMonth);
        try {
            return baseMapper.selectPage(new Page<>(page, size),
                Wrappers.lambdaQuery(Order.class).eq(Order::getUserId, 1001));
        } finally {
            DynamicTableNameHandler.clear(); // 必须清理，防止影响后续请求
        }
    }
}
```

## 批量操作

MyBatis-Plus 3.5.5+ 提供了一键批量写入功能。

### 批量插入（JDBC Batch）

```java
@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {

    /**
     * 批量插入（SQL 级别 batch，性能远优于逐条 insert）
     * 需要在 yaml 配置：rewriteBatchedStatements=true（MySQL）/ 默认支持（PostgreSQL）
     */
    @Transactional(rollbackFor = Exception.class)
    public void batchInsertUsers(List<User> userList) {
        // MyBatis-Plus 3.5.5+ 原生批量插入
        saveBatch(userList, 1000);  // 每批 1000 条

        // 底层实际执行 JDBC Batch，而非逐条 INSERT
    }

    /**
     * 批量新增或更新（按主键判断）
     */
    @Transactional(rollbackFor = Exception.class)
    public void batchSaveOrUpdate(List<User> userList) {
        saveOrUpdateBatch(userList, 500);
    }
}
```

### 流式查询（大数据量导出）

```java
@Mapper
public interface OrderMapper extends BaseMapper<Order> {

    /**
     * 流式查询：避免大数据量 OOM
     * resultType 必须指定，fetchSize 设为 Integer.MIN_VALUE 驱动才真正流式
     */
    @Select("SELECT * FROM sys_order WHERE created_at >= #{start} AND created_at < #{end}")
    @Options(resultSetType = ResultSetType.FORWARD_ONLY, fetchSize = Integer.MIN_VALUE)
    void streamQueryOrders(@Param("start") LocalDateTime start,
                           @Param("end") LocalDateTime end,
                           ResultHandler<Order> handler);
}

// 使用
@Autowired
private SqlSessionFactory sqlSessionFactory;

public void exportOrders(LocalDateTime start, LocalDateTime end, OutputStream os) {
    // 需要在单独的 SqlSession 中执行，避免流式连接长时间不释放
    try (SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.SIMPLE, false)) {
        OrderMapper mapper = sqlSession.getMapper(OrderMapper.class);
        CSVWriter writer = new CSVWriter(new OutputStreamWriter(os, StandardCharsets.UTF_8));

        mapper.streamQueryOrders(start, end, context -> {
            Order order = context.getResultObject();
            writer.writeNext(new String[]{
                String.valueOf(order.getId()),
                order.getOrderNo(),
                order.getStatus(),
                order.getCreatedAt().toString()
            });
        });

        writer.flush();
    }
}
```

## 自定义类型处理器（TypeHandler）

处理 JSONB、数组等数据库特殊类型。

### JSONB TypeHandler

```java
@MappedTypes(List.class)
@MappedJdbcTypes(JdbcType.OTHER)  // PostgreSQL JSONB 通过 OTHER 映射
public class JsonbListTypeHandler extends BaseTypeHandler<List<Object>> {

    private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i,
                                    List<Object> parameter, JdbcType jdbcType) throws SQLException {
        try {
            String json = OBJECT_MAPPER.writeValueAsString(parameter);
            ps.setObject(i, json, Types.OTHER);
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public List<Object> getNullableResult(ResultSet rs, String columnName) throws SQLException {
        String json = rs.getString(columnName);
        return parseJsonList(json);
    }

    @Override
    public List<Object> getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        String json = rs.getString(columnIndex);
        return parseJsonList(json);
    }

    @Override
    public List<Object> getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        String json = cs.getString(columnIndex);
        return parseJsonList(json);
    }

    private List<Object> parseJsonList(String json) {
        if (json == null || json.isEmpty()) return Collections.emptyList();
        try {
            return OBJECT_MAPPER.readValue(json, new TypeReference<List<Object>>() {});
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
    }
}
```

### 实体中使用

```java
@TableName(value = "sys_product", autoResultMap = true)
public class Product {

    @TableId
    private Long id;

    private String name;

    @TableField(typeHandler = JsonbListTypeHandler.class)
    private List<String> tags;  // PostgreSQL JSONB 列

    @TableField(typeHandler = JsonbListTypeHandler.class)
    private List<ProductSku> skus;  // 复杂对象列表
}
```

## 注意事项

1. **数据权限性能影响**：每个查询额外执行一次数据权限解析，复杂规则建议缓存 TableInfo
2. **分页总数优化**：百万级以上大表 `PaginationInnerInterceptor` 的 COUNT 查询可能很慢，可手动 `page.setOptimizeCountSql(false)` 或自定义 count 查询
3. **逻辑删除 + 唯一索引**：逻辑删除的表建唯一索引时，MySQL 需要降级（删除记录+时间戳），PostgreSQL 可用部分索引 `WHERE deleted_at IS NULL`
4. **`saveBatch` 的注意事项**：默认 batchSize=1000，事务必须手动开启（`@Transactional`），否则逐条提交
5. **Lambda 查询类型安全**：`Wrappers.lambdaQuery()` 在编译期校验字段名，避免 `Wrappers.query()` 的字符串拼写错误
6. **填充策略**：`MetaObjectHandler` 的 `insertFill` / `updateFill` 方法中，用 `strictInsertFill` 避免覆盖已有值
7. **代码生成器**：MyBatis-Plus Generator 3.5.5+ 支持输出到指定目录，配合模板可生成 Controller/Service/Mapper/Entity 全套代码
