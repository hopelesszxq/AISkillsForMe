---
name: mybatis-plus-data-security
description: MyBatis-Plus 数据安全最佳实践：字段加密、数据脱敏、敏感数据自动处理与审计
tags: [mybatis-plus, security, encryption, data-masking, desensitization]
---

## 概述

MyBatis-Plus 提供了多种实现数据安全的方式，包括**字段加密存储**、**查询结果脱敏**、**敏感数据自动处理**。本文介绍四种主流方案及其适用场景。

## 方案一：TypeHandler 字段加密（推荐）

通过自定义 TypeHandler，在 ORM 层透明加密/解密。

### 1. AES 加密 TypeHandler

```java
import org.apache.ibatis.type.BaseTypeHandler;
import org.apache.ibatis.type.JdbcType;
import javax.crypto.Cipher;
import javax.crypto.spec.SecretKeySpec;
import java.sql.*;

public class AesEncryptHandler extends BaseTypeHandler<String> {

    private static final String AES_KEY = "YourSecretKey123"; // 32位，需外部化配置
    private static final String ALGORITHM = "AES/ECB/PKCS5Padding";

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i,
            String parameter, JdbcType jdbcType) throws SQLException {
        ps.setString(i, encrypt(parameter));
    }

    @Override
    public String getNullableResult(ResultSet rs, String columnName)
            throws SQLException {
        String value = rs.getString(columnName);
        return value == null ? null : decrypt(value);
    }

    @Override
    public String getNullableResult(ResultSet rs, int columnIndex)
            throws SQLException {
        String value = rs.getString(columnIndex);
        return value == null ? null : decrypt(value);
    }

    @Override
    public String getNullableResult(CallableStatement cs, int columnIndex)
            throws SQLException {
        String value = cs.getString(columnIndex);
        return value == null ? null : decrypt(value);
    }

    private String encrypt(String plainText) {
        try {
            SecretKeySpec key = new SecretKeySpec(
                AES_KEY.getBytes(), "AES");
            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.ENCRYPT_MODE, key);
            byte[] encrypted = cipher.doFinal(plainText.getBytes());
            return Base64.getEncoder().encodeToString(encrypted);
        } catch (Exception e) {
            throw new RuntimeException("AES encrypt error", e);
        }
    }

    private String decrypt(String cipherText) {
        try {
            SecretKeySpec key = new SecretKeySpec(
                AES_KEY.getBytes(), "AES");
            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.DECRYPT_MODE, key);
            byte[] decrypted = cipher.doFinal(
                Base64.getDecoder().decode(cipherText));
            return new String(decrypted);
        } catch (Exception e) {
            throw new RuntimeException("AES decrypt error", e);
        }
    }
}
```

### 2. 实体类注解

```java
@Data
@TableName("user")
public class User {
    private Long id;
    private String name;

    @TableField(typeHandler = AesEncryptHandler.class)
    private String phone;           // 加密存储

    @TableField(typeHandler = AesEncryptHandler.class)
    private String idCard;          // 加密存储

    private String email;           // 明文
}
```

### 3. 配置外部化密钥

```yaml
# application.yml
myapp:
  crypto:
    key: ${AES_SECRET_KEY:DefaultDevKeyReplaceInProd!!}
```

```java
@Configuration
public class CryptoConfig {
    @Value("${myapp.crypto.key}")
    private String secretKey;

    @Bean
    public AesEncryptHandler aesEncryptHandler() {
        // 可通过 @PostConstruct 注入密钥
        return new AesEncryptHandler();
    }
}
```

## 方案二：Result 脱敏（Jackson 序列化层）

不在数据库层加密，仅在展示时脱敏。适合日志、API 响应等场景。

### 自定义脱敏注解

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@JacksonAnnotationsInside
@JsonSerialize(using = DesensitizeSerializer.class)
public @interface Desensitize {
    /** 脱敏策略 */
    Strategy strategy() default Strategy.PHONE;

    enum Strategy {
        PHONE,       // 138****1234
        ID_CARD,     // 110**************789
        EMAIL,       // tes***@example.com
        NAME,        // 张*
        ADDRESS,     // 北京市***
        PASSWORD,    // ******
        CUSTOM       // 自定义
    }
}
```

```java
public class DesensitizeSerializer extends JsonSerializer<String> {
    @Override
    public void serialize(String value, JsonGenerator gen,
            SerializerProvider provider) throws IOException {
        gen.writeString(desensitize(value, getStrategy(provider)));
    }

    private String desensitize(String value, Desensitize.Strategy strategy) {
        if (value == null) return null;
        switch (strategy) {
            case PHONE:
                return value.replaceAll("(\\d{3})\\d{4}(\\d{4})", "$1****$2");
            case ID_CARD:
                return value.replaceAll("(\\d{3})\\d{12}(\\w{3})", "$1***********$2");
            case EMAIL:
                int atIdx = value.indexOf("@");
                if (atIdx > 1) return value.charAt(0) + "***" + value.substring(atIdx);
                return "***" + value.substring(atIdx);
            case NAME:
                return value.replaceAll("(.)(.*)", "$1*");
            case PASSWORD:
                return "******";
            default:
                return value;
        }
    }
}
```

### 实体类中使用

```java
@Data
public class UserVO {
    private Long id;
    @Desensitize(strategy = Desensitize.Strategy.NAME)
    private String name;
    @Desensitize(strategy = Desensitize.Strategy.PHONE)
    private String phone;
    @Desensitize(strategy = Desensitize.Strategy.ID_CARD)
    private String idCard;
}
```

## 方案三：拦截器实现自动加密/解密

通过 MyBatis-Plus 的 InnerInterceptor 在 SQL 层拦截，自动处理标记字段。

```java
@Component
public class EncryptInterceptor implements InnerInterceptor {

    private final AesEncryptHandler encryptHandler;

    @Override
    public void beforeInsert(Executor executor, Object parameter) {
        // 遍历参数中 @EncryptedField 注解字段，自动加密
        processEncryptFields(parameter);
    }

    @Override
    public void beforeUpdate(Executor executor, Object parameter) {
        processEncryptFields(parameter);
    }

    @Override
    public void afterQuery(Executor executor, MappedStatement ms,
            RowBounds rowBounds, ResultHandler resultHandler,
            BoundSql boundSql, List<Object> parameterList) {
        // 在 ResultHandler 中解密
    }

    private void processEncryptFields(Object param) {
        // 反射遍历字段，找到 @EncryptedField 注解，调用加密
    }
}
```

## 方案四：MyBatis-Plus 3.5.17+ 内置加密注解

MyBatis-Plus 3.5.17+ 实验性支持 `@EncryptedField` 注解（需确认版本支持情况）：

```java
@Data
@TableName("payment_card")
public class PaymentCard {
    private Long id;

    @EncryptedField                        // 自动加密存储
    private String cardNumber;

    @EncryptedField(type = "SM4")          // 指定加密算法
    private String cvv;

    @DesensitizedField(
        strategy = "PHONE",
        condition = "hasPermission('admin')"  // 有条件脱敏
    )
    private String bindPhone;
}
```

## MyBatis-Plus 配置整合

```java
@Configuration
@MapperScan("com.example.mapper")
public class MyBatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        // 分页等现有插件
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor());
        return interceptor;
    }
}
```

## 注意事项

1. **密钥管理**：永远不要硬编码密钥，使用配置中心（Nacos/Apollo）或 KMS（AWS KMS/阿里云 KMS）管理
2. **模糊查询问题**：加密字段无法直接 LIKE 查询，需要：
   - 使用**哈希索引**（存字段的 SHA-256 哈希用于等值匹配）
   - 或**分词加密**（将身份证分段加密）
   - 或**数据库加密函数**（如 pgcrypto）
3. **性能影响**：加密/解密增加 CPU 开销，QPS 较高的字段建议只在写入时加密，读时不做解密直接返回密文
4. **索引失效**：加密字段上的数据库索引基本失效，考虑使用加密哈希列加索引
5. **日志脱敏**：确保 MyBatis SQL 日志不打印明文，配置 logback 过滤器过滤敏感字段

### 模糊查询替代方案

```java
// 额外存储 SHA-256 哈希用于精确匹配
@TableName("user")
public class User {
    private Long id;

    @TableField(typeHandler = AesEncryptHandler.class)
    private String phone;

    @TableField("phone_hash")       // 额外列
    private String phoneHash;        // SHA-256(phone)

    @TableField(typeHandler = AesEncryptHandler.class)
    private String idCard;

    @TableField("id_card_last4")    // 尾号4位，用于模糊查询
    private String idCardLast4;
}
```

6. **密钥轮换**：设计密钥版本号，支持平滑轮换
7. **测试环境**：开发/测试环境可关闭加密（通过配置开关），降低调试复杂度
