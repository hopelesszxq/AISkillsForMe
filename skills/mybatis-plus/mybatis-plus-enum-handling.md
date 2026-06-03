---
name: mybatis-plus-enum-handling
description: MyBatis-Plus 枚举类型处理最佳实践：IEnum 接口、@EnumValue 注解、EnumTypeHandler、自动序列化与JSON互转
tags: [mybatis-plus, enum, orm, typehandler, serialization]
---

## 概述

MyBatis-Plus 对枚举类型提供了丰富的支持方案，从简单的 `IEnum` 接口到通用的 `@EnumValue` 注解，再到自定义 TypeHandler。合理使用可大幅减少枚举相关的 Boilerplate 代码。

## 推荐方案对比

| 方案 | 代码量 | 灵活性 | 推荐场景 |
|---|---|---|---|
| `IEnum` 接口 | ⭐⭐⭐ | ⭐⭐ | 简单枚举，仅存值不存描述 |
| `@EnumValue` + `@JsonValue` | ⭐⭐⭐⭐ | ⭐⭐⭐ | 枚举值 ≠ 数据库值（最常见） |
| `@EnumValue` + Jackson 自动序列化 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 前后端直通，自动序列化为对象 |
| 自定义 TypeHandler | ⭐⭐ | ⭐⭐⭐⭐⭐ | 复杂映射逻辑（位掩码、复合枚举） |

## 方案一：IEnum 接口（最简单）

### 枚举定义

```java
public enum OrderStatus implements IEnum<Integer> {
    PENDING(0, "待支付"),
    PAID(1, "已支付"),
    SHIPPED(2, "已发货"),
    COMPLETED(3, "已完成"),
    CANCELLED(-1, "已取消");

    private final int value;
    private final String desc;

    OrderStatus(int value, String desc) {
        this.value = value;
        this.desc = desc;
    }

    @Override
    public Integer getValue() {
        return this.value;
    }
}
```

### 实体类

```java
@Data
@TableName("t_order")
public class Order {
    private Long id;
    private String orderNo;

    // 无需任何额外注解，MP 自动识别 IEnum
    private OrderStatus status;
}
```

### 数据库映射

`order_status` 列存储 `0`, `1`, `2` 等整数值，MyBatis-Plus 自动调用 `getValue()` 读写。

### 全局配置（application.yml）

```yaml
mybatis-plus:
  configuration:
    # 默认使用 EnumTypeHandler，也可以全局指定
    default-enum-type-handler: com.baomidou.mybatisplus.core.handlers.MybatisEnumTypeHandler
```

## 方案二：@EnumValue + @JsonValue（最常用）

当枚举的数据库存储值与 `ordinal()` 或 `name()` 不符时，使用 `@EnumValue` 显式指定存储值。

### 枚举定义

```java
public enum Gender {
    MALE(1, "男"),
    FEMALE(0, "女"),
    UNKNOWN(-1, "未知");

    @EnumValue   // 告诉 MP 存入数据库时用此字段
    @JsonValue   // 告诉 Jackson 序列化时用此字段（前后端统一）
    private final int code;

    @JsonIgnore  // 不序列化描述到前端
    private final String desc;

    Gender(int code, String desc) {
        this.code = code;
        this.desc = desc;
    }
}
```

### 效果

```json
// 前端收到的 JSON
{ "id": 1, "name": "张三", "gender": 1 }

// 数据库存储的值
gender = 1
```

### 前端回传枚举值

Spring MVC 自动将 `1` 反序列化为 `Gender.MALE`：

```java
@PostMapping("/user")
public User createUser(@RequestBody User user) {
    // gender 字段从 JSON 的 1 自动转为 Gender.MALE
    return userService.save(user);
}
```

需要加配置（Spring Boot 自动配置已默认支持）：

```java
@Bean
public Jackson2ObjectMapperBuilderCustomizer enumCustomizer() {
    return builder -> builder
        .featuresToEnable(SerializationFeature.WRITE_ENUMS_USING_TO_STRING)
        .serializerByType(Enum.class, new EnumSerializer());
}
```

## 方案三：枚举序列化为完整对象（前后端枚举字典）

当需要前端展示枚举的可选值和描述时，用 Jackson 的 `@JsonFormat(shape = Shape.OBJECT)`。

```java
@JsonFormat(shape = JsonFormat.Shape.OBJECT)
public enum OrderType {
    NORMAL(0, "普通订单", "普通购买流程"),
    PRESALE(1, "预售订单", "预先付款等待发货"),
    GROUP(2, "拼团订单", "多人拼团优惠");

    @EnumValue
    private final int code;
    private final String name;
    private final String desc;

    OrderType(int code, String name, String desc) {
        this.code = code;
        this.name = name;
        this.desc = desc;
    }
}
```

### 序列化结果

```json
// 单值字段
{ "orderType": { "code": 1, "name": "预售订单", "desc": "预先付款等待发货" } }

// 前端枚举字典接口
[
  { "code": 0, "name": "普通订单" },
  { "code": 1, "name": "预售订单" },
  { "code": 2, "name": "拼团订单" }
]
```

## 方案四：MybatisEnumTypeHandler + 按需配置

全局统一使用 MyBatis-Plus 内置的枚举处理器：

```yaml
mybatis-plus:
  configuration:
    default-enum-type-handler: com.baomidou.mybatisplus.core.handlers.MybatisEnumTypeHandler
```

也可以按字段指定：

```java
@TableName("t_user")
public class User {
    private Long id;
    private String name;

    @TableField(typeHandler = MybatisEnumTypeHandler.class)
    private Gender gender;
}
```

## 枚举 Feign 序列化（跨服务调用）

Feign 默认使用 Spring 的 Jackson 配置，枚举序列化一致。但需要确保：

```java
@FeignClient(name = "user-service", configuration = UserFeignConfig.class)
public interface UserFeignClient {
    @GetMapping("/user/{id}")
    User getUser(@PathVariable Long id);
}
```

如果返回的 JSON 中枚举值是整型（`"gender": 1`），而接收端枚举用 `@JsonValue` 标注，需注意反序列化冲突：

```java
// 问题：JSON 中 gender=1，但 Jackson 默认按 ordinal 反序列化
// 解决：使用 @JsonCreator 自定义反序列化
public enum Gender {
    MALE(1), FEMALE(0), UNKNOWN(-1);

    private final int code;

    Gender(int code) { this.code = code; }

    @EnumValue
    public int getCode() { return code; }

    @JsonCreator
    public static Gender fromCode(int code) {
        for (Gender g : values()) {
            if (g.code == code) return g;
        }
        return UNKNOWN;
    }
}
```

## 方案五：自定义 TypeHandler（高级场景）

适用于位掩码枚举、Set<Enum> 存逗号分隔串等特殊需求。

### 位掩码枚举 TypeHandler

```java
// 枚举定义
public enum Permission {
    READ(1), WRITE(2), DELETE(4), ADMIN(8);

    private final int bit;

    Permission(int bit) { this.bit = bit; }

    public int getBit() { return bit; }
}

// TypeHandler：权限列表 ↔ 整型位掩码
@MappedTypes(Set.class)
public class PermissionSetTypeHandler extends BaseTypeHandler<Set<Permission>> {

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i,
                                    Set<Permission> params, JdbcType jdbcType) throws SQLException {
        int mask = params.stream().mapToInt(Permission::getBit).reduce(0, (a, b) -> a | b);
        ps.setInt(i, mask);
    }

    @Override
    public Set<Permission> getNullableResult(ResultSet rs, String columnName) throws SQLException {
        return decode(rs.getInt(columnName));
    }

    @Override
    public Set<Permission> getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        return decode(rs.getInt(columnIndex));
    }

    @Override
    public Set<Permission> getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        return decode(cs.getInt(columnIndex));
    }

    private Set<Permission> decode(int mask) {
        return Arrays.stream(Permission.values())
            .filter(p -> (mask & p.getBit()) != 0)
            .collect(Collectors.toSet());
    }
}
```

### 使用

```java
@Data
@TableName("t_role")
public class Role {
    private Long id;
    private String name;

    @TableField(typeHandler = PermissionSetTypeHandler.class)
    private Set<Permission> permissions;  // 数据库存整型，如 6 → {WRITE, DELETE}
}
```

## 枚举校验（Spring Validation）

```java
import jakarta.validation.constraints.NotNull;

public class OrderCreateRequest {

    @NotNull(message = "订单类型不能为空")
    private OrderType orderType;
}

// 自定义校验：检查枚举值是否存在
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = EnumValueValidator.class)
public @interface ValidEnum {
    String message() default "无效的枚举值";
    Class<?>[] groups() default {};
    Class<?>[] payload() default {};
    Class<? extends Enum<?>> enumClass();
}

public class EnumValueValidator implements ConstraintValidator<ValidEnum, Object> {
    private Object[] enumValues;

    @Override
    public void initialize(ValidEnum constraint) {
        enumValues = constraint.enumClass().getEnumConstants();
    }

    @Override
    public boolean isValid(Object value, ConstraintValidatorContext context) {
        if (value == null) return true;
        for (Object e : enumValues) {
            if (e.equals(value)) return true;
            // 兼容 Integer 类型枚举值
            if (e instanceof IEnum && ((IEnum<?>) e).getValue().equals(value)) return true;
        }
        return false;
    }
}
```

## 注意事项

1. **枚举值不可变**：一旦生产使用过的枚举值，**绝不可删除或修改 code**，只能新增且标记 `@Deprecated`
2. **序列化循环**：`@JsonFormat(Shape.OBJECT)` 时，注意枚举内不要互相引用导致死循环
3. **MybatisEnumTypeHandler vs EnumTypeHandler**：后者使用 `name()`（枚举名），前者使用 `@EnumValue`。务必使用 MP 提供的处理器
4. **枚举长度**：数据库列类型与枚举 code 类型一致，`int` → `INT`，`String` → `VARCHAR(32)`
5. **全局默认处理器配置**在 `application.yml` 即可，无需在每个字段加 `@TableField(typeHandler=...)`
6. **枚举缓存**：频繁使用的枚举列表建议用 Redis 缓存做前端下拉字典，避免每次查数据库
