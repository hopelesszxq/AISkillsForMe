---
name: mybatis-plus-sql-injector
description: MyBatis-Plus SQL 注入器、自定义 ID 生成器与多租户插件实战
tags: [mybatis-plus, orm, database, plugin, injector]
---

## SQL 注入器（SQL Injector）

MyBatis-Plus 内置了常用的 CRUD 方法。当需要自定义全局通用方法（如批量更新、批量删除、自定义 INSERT）时，使用 SQL 注入器，无需在每个 Mapper 重复编写 SQL。

### 自定义 SQL 注入器步骤

```java
// 1. 定义自定义方法
public class InsertBatchMethod extends AbstractMethod {

    @Override
    public MappedStatement injectMappedStatement(Class<?> mapperClass,
                                                  Class<?> modelClass,
                                                  TableInfo tableInfo) {
        // SQL 模板：INSERT INTO 表名(字段...) VALUES (值...)
        String sql = "<script>INSERT INTO %s %s VALUES %s</script>";

        String fieldSql = prepareFieldSql(tableInfo);
        String valueSql = prepareValueSql(tableInfo);

        // 构建 MappedStatement
        String sqlResult = String.format(sql, tableInfo.getTableName(),
            fieldSql, valueSql);

        return addInsertMappedStatement(mapperClass, modelClass,
            "insertBatch", sqlResult, new InsertHandler(), null, null);
    }

    private String prepareFieldSql(TableInfo tableInfo) {
        StringBuilder fieldSql = new StringBuilder();
        fieldSql.append("(");
        // 主键
        fieldSql.append(tableInfo.getKeyColumn()).append(",");
        // 普通字段
        tableInfo.getFieldList().forEach(f ->
            fieldSql.append(f.getColumn()).append(","));
        fieldSql.deleteCharAt(fieldSql.length() - 1);
        fieldSql.append(")");
        return fieldSql.toString();
    }

    private String prepareValueSql(TableInfo tableInfo) {
        StringBuilder valueSql = new StringBuilder();
        valueSql.append("<foreach collection=\"list\" item=\"item\" separator=\",\">");
        valueSql.append("(");
        // #{item.属性名}，从实体类获取
        valueSql.append("#{item.").append(tableInfo.getKeyProperty()).append("},");
        tableInfo.getFieldList().forEach(f ->
            valueSql.append("#{item.").append(f.getProperty()).append("},"));
        valueSql.deleteCharAt(valueSql.length() - 1);
        valueSql.append(")");
        valueSql.append("</foreach>");
        return valueSql.toString();
    }

    // INSERT 处理器
    public static class InsertHandler implements SqlHandler {
        // 处理 INSERT 逻辑
    }
}
```

```java
// 2. 注册到 SQL 注入器
@Component
public class CustomSqlInjector extends DefaultSqlInjector {

    @Override
    public List<AbstractMethod> getMethodList(Class<?> mapperClass) {
        // 获取父类所有内置方法
        List<AbstractMethod> methodList = super.getMethodList(mapperClass);
        // 追加自定义方法
        methodList.add(new InsertBatchMethod());
        // methodList.add(new DeleteAllMethod());
        // methodList.add(new TruncateMethod());
        return methodList;
    }
}
```

```java
// 3. 配置注入器
@Configuration
public class MyBatisPlusConfig {

    @Bean
    public CustomSqlInjector customSqlInjector() {
        return new CustomSqlInjector();
    }
}
```

```java
// 4. Mapper 继承自定义接口
public interface UserMapper extends BaseMapper<User> {

    // 来自 SQL 注入器的方法，无需在 XML 中手写
    int insertBatch(@Param("list") List<User> list);

    // 也可以同时定义自己的方法
    @Select("SELECT * FROM sys_user WHERE dept_id = #{deptId}")
    List<User> selectByDeptId(@Param("deptId") Long deptId);
}
```

### 常用内置 AbstractMethod 重用

| 方法类 | 描述 |
|---|---|
| `Insert` | 插入一条数据 |
| `Delete` | 根据 entity 条件删除 |
| `Update` | 根据 whereWrapper 更新 |
| `SelectCount` | 查询总数 |
| `SelectMapsPage` | 分页查询返回 Map |
| `SelectObjs` | 只查第一列 |

## 自定义 ID 生成器

MyBatis-Plus 3.5.0+ 支持自定义 ID 生成器（雪花算法、UUID、自增 ID 以外的策略）。

### 雪花算法增强版（解决时钟回拨）

```java
@Component
public class CustomIdGenerator implements IdentifierGenerator {

    private final SnowflakeIdWorker worker = new SnowflakeIdWorker(1, 1);

    @Override
    public Long nextId(Object entity) {
        return worker.nextId();
    }

    /**
     * 解决时钟回拨的雪花算法
     */
    static class SnowflakeIdWorker {
        private final long workerId;
        private final long datacenterId;
        private long sequence = 0L;
        private long lastTimestamp = -1L;

        // 起始时间戳（2024-01-01）
        private static final long EPOCH = 1704067200000L;
        // 机器 ID 占位
        private static final long WORKER_ID_BITS = 5L;
        private static final long DATACENTER_ID_BITS = 5L;
        private static final long SEQUENCE_BITS = 12L;

        private static final long MAX_WORKER_ID = ~(-1L << WORKER_ID_BITS);
        private static final long MAX_DATACENTER_ID = ~(-1L << DATACENTER_ID_BITS);

        private static final long WORKER_ID_SHIFT = SEQUENCE_BITS;
        private static final long DATACENTER_ID_SHIFT = SEQUENCE_BITS + WORKER_ID_BITS;
        private static final long TIMESTAMP_LEFT_SHIFT = SEQUENCE_BITS + WORKER_ID_BITS + DATACENTER_ID_BITS;
        private static final long SEQUENCE_MASK = ~(-1L << SEQUENCE_BITS);

        public SnowflakeIdWorker(long workerId, long datacenterId) {
            if (workerId > MAX_WORKER_ID || workerId < 0)
                throw new IllegalArgumentException("workerId 非法");
            if (datacenterId > MAX_DATACENTER_ID || datacenterId < 0)
                throw new IllegalArgumentException("datacenterId 非法");
            this.workerId = workerId;
            this.datacenterId = datacenterId;
        }

        public synchronized long nextId() {
            long timestamp = System.currentTimeMillis();

            // 时钟回拨处理：等待到下一毫秒或抛出异常
            if (timestamp < lastTimestamp) {
                long offset = lastTimestamp - timestamp;
                // 回拨 ≤ 5ms 时等待
                if (offset <= 5) {
                    try {
                        wait(offset << 1);
                        timestamp = System.currentTimeMillis();
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        throw new RuntimeException("ID 生成中断");
                    }
                } else {
                    throw new RuntimeException("时钟回拨超过 5ms: " + offset);
                }
            }

            if (lastTimestamp == timestamp) {
                sequence = (sequence + 1) & SEQUENCE_MASK;
                if (sequence == 0) {
                    timestamp = tilNextMillis(lastTimestamp);
                }
            } else {
                sequence = 0L;
            }

            lastTimestamp = timestamp;

            return ((timestamp - EPOCH) << TIMESTAMP_LEFT_SHIFT)
                | (datacenterId << DATACENTER_ID_SHIFT)
                | (workerId << WORKER_ID_SHIFT)
                | sequence;
        }

        private long tilNextMillis(long lastTimestamp) {
            long timestamp = System.currentTimeMillis();
            while (timestamp <= lastTimestamp) {
                timestamp = System.currentTimeMillis();
            }
            return timestamp;
        }
    }
}
```

### 在实体中使用

```java
@Data
@TableName("sys_user")
public class User {

    @TableId(type = IdType.ASSIGN_ID)  // 使用自定义 ID 生成器
    private Long id;

    private String name;
}
```

注意：需要在配置中指定生成器 Bean：

```yaml
mybatis-plus:
  global-config:
    db-config:
      id-type: assign_id  # 全局 ID 策略
```

## 多租户插件（TenantLineInnerInterceptor）

MyBatis-Plus 内置多租户插件，自动在 SQL 追加租户过滤条件。

### 基础配置

```java
@Configuration
public class MyBatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 多租户插件（需排在数据权限之前）
        TenantLineInnerInterceptor tenant = new TenantLineInnerInterceptor();
        tenant.setTenantLineHandler(new CustomTenantHandler());
        interceptor.addInnerInterceptor(tenant);

        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.POSTGRE_SQL));
        return interceptor;
    }
}
```

```java
@Component
public class CustomTenantHandler implements TenantLineHandler {

    /**
     * 获取租户 ID 列名，默认 tenant_id
     */
    @Override
    public String getTenantIdColumn() {
        return "tenant_id";
    }

    /**
     * 获取当前租户 ID（从请求上下文或 Token 中解析）
     */
    @Override
    public Expression getTenantId() {
        Long tenantId = TenantContextHolder.getCurrentTenantId();
        if (tenantId == null) {
            throw new RuntimeException("无法获取租户 ID");
        }
        return new LongValue(tenantId);
    }

    /**
     * 忽略租户过滤的表（基础数据表不需要租户隔离）
     */
    @Override
    public boolean ignoreTable(String tableName) {
        return "sys_config".equals(tableName)
            || "sys_dict".equals(tableName)
            || "sys_tenant".equals(tableName);
    }
}
```

```java
// TenantContextHolder — 存储当前请求的租户信息
public class TenantContextHolder {

    private static final ThreadLocal<Long> CURRENT_TENANT = new ThreadLocal<>();

    public static void setCurrentTenantId(Long tenantId) {
        CURRENT_TENANT.set(tenantId);
    }

    public static Long getCurrentTenantId() {
        return CURRENT_TENANT.get();
    }

    public static void clear() {
        CURRENT_TENANT.remove();
    }
}
```

### 忽略特定方法的租户过滤

```java
@Mapper
public interface ConfigMapper extends BaseMapper<Config> {

    /**
     * 查询全局配置（不需要租户过滤）
     * 使用 @InterceptorIgnore(tenantLine = "true") 忽略
     */
    @InterceptorIgnore(tenantLine = "true")
    @Select("SELECT * FROM sys_config WHERE config_key = #{key}")
    Config findByKey(@Param("key") String key);

    /**
     * 查询租户配置（需要租户过滤，这是默认行为）
     */
    @Select("SELECT * FROM sys_config WHERE config_key = #{key}")
    Config findTenantConfig(@Param("key") String key);
}
```

也支持在 Service 层使用：

```java
@Service
public class ConfigService {

    // 忽略整个方法内的所有 SQL 租户过滤
    @InterceptorIgnore(tenantLine = "true")
    public List<Config> getAllGlobalConfigs() {
        return configMapper.selectList(Wrappers.emptyWrapper());
    }
}
```

## 注意事项

1. **SQL 注入器与 XML 冲突**：自定义 AbstractMethod 生成的 SQL 和 XML 中定义的同名方法冲突，注意方法名唯一
2. **ID 生成器性能**：雪花算法在高并发（万级/秒）场景下仍表现良好，但需注意机器 ID 的分配策略（可通过 Zookeeper/Redis 动态分配）
3. **多租户性能**：每个 SQL 追加 AND tenant_id=? 条件，确保此列有索引，否则全表扫描
4. **ignoreTable 配置**：缓存表、字典表、配置表等全局表需要忽略租户过滤，否则查不到数据
5. **@InterceptorIgnore 的局限性**：注解只对当前 Mapper 方法生效，如果 Mapper 方法内部调用了其他 Mapper，不会级联忽略
6. **多租户 + 逻辑删除**：逻辑删除的 SQL（UPDATE SET deleted=1）也会自动追加租户条件，确保不会误删其他租户数据
7. **租户 ID 传递**：在 Gateway/Filler 中解析 Token 后通过 TenantContextHolder 设置租户 ID，需在请求完成后 clear() 防止内存泄漏
