---
name: seata-260-features
description: Seata v2.6.0 新特性：HTTP/2 支持、PostgreSQL 数组类型、Oracle 批量插入、事务组管理
tags: [spring-cloud, seata, distributed-transaction, http2, postgresql, oracle]
---

## Seata v2.6.0 新特性

Apache Seata(incubating) v2.6.0（2026-01-28 发布）引入了多项重要功能和兼容性增强。

### 1. HTTP/2 支持

Seata Server 和 Client 均支持 HTTP/2 协议，提升通信效率。

```yaml
# seata-server application.yml
server:
  # HTTP/2 配置
  http2:
    enabled: true
  ssl:
    enabled: true
    key-store: classpath:seata-server.jks
    key-store-password: changeit
    key-alias: seata-server
```

```java
// Client 端配置 HTTP/2
// upgrade HTTP client in common module to support HTTP/2
// 客户端自动适配，无需额外配置
@Bean
public SeataRestTemplate seataRestTemplate() {
    return new SeataRestTemplate(); // 自动使用 HTTP/2
}
```

### 2. 事务组管理

新增事务组（Transaction Group）管理支持，可以在 Seata Console 中查看集群信息。

```yaml
# 事务组配置
seata:
  tx-service-group: my_tx_group
  service:
    # 支持事务组管理
    vgroup-mapping:
      my_tx_group: default
    # 集群信息（新增控制台展示）
    grouplist:
      default: 
        - 192.168.1.10:8091
        - 192.168.1.11:8091
        - 192.168.1.12:8091
```

控制台新增功能：
- 显示集群节点信息
- 事务组状态可视化
- Raft 模式下 Watch API 支持 HTTP/2 响应处理

### 3. PostgreSQL 数组类型支持

新增 Jackson 序列化/反序列化对 PostgreSQL 数组类型的支持。

```java
// 在 undo_log 和业务表中处理 PostgreSQL 数组字段
@TableName("product")
public class Product {
    @TableId
    private Long id;
    
    // PostgreSQL 数组字段 — Seata 2.6.0 以下版本会序列化失败
    private String[] tags;
    
    private Integer[] categoryIds;
}

// 需在配置中启用 Jackson 支持
seata:
  serializer: jackson  # 使用 Jackson 序列化以支持数组类型
  config:
    type: nacos
```

### 4. Oracle 批量插入支持

AT 模式下支持 Oracle 数据库的 Batch Insert 操作，提升大批量数据导入性能。

```java
// 批量插入示例（Oracle）
@GlobalTransactional
public void batchImport(List<Product> products) {
    // Seata 2.6.0 支持 Oracle 批量插入的 SQL 解析
    productMapper.insertBatch(products); // 底层使用 JDBC batch
}
```

### 5. 其他增强

| 功能 | 说明 |
|------|------|
| HTTP 请求过滤器 | Seata Server 新增 HTTP 请求过滤器 |
| 连接复用 | 复用连接合并分支事务，减少资源消耗 |
| Fury 序列化/反序列化 | 支持 Fury 序列化格式和 Fury UndoLog 解析器 |
| DM 数据库 XA | XAUtils 新增达梦数据库支持 |
| 神通数据库 XA | 支持神通数据库 XA 模式 |
| Java 25 支持 | CI 配置新增 Java 25 支持 |
| NamingServer 心跳修复 | 修复 NamingServer 与其他注册中心混合使用时心跳异常 |

### 6. 安全增强（2.5.0 起）

```yaml
# 强制账号初始化（2.5.0 新增）
seata:
  server:
    security:
      # 2.5.0 起强制初始化管理员账号
      enable-auth: true
      # 禁用默认凭证，必须修改默认密码
      force-init-account: true
```

## 升级注意事项

1. **从 2.5.x 升级到 2.6.0**：配置兼容，SQL 无需改动
2. **HTTP/2 需手动开启**：默认仍使用 HTTP/1.1
3. **PostgreSQL 数组支持**：需使用 Jackson 序列化器（默认是 Kryo）
4. **事务组管理**：控制台需升级到对应版本
5. **Seata 已进入 Apache 孵化器**：注意 Maven GroupId 变更

```xml
<!-- Apache Seata(incubating) Maven 坐标 -->
<dependency>
    <groupId>org.apache.seata</groupId>
    <artifactId>seata-spring-boot-starter</artifactId>
    <version>2.6.0</version>
</dependency>
```

## 参考链接

- [Seata v2.6.0 Release](https://github.com/apache/incubator-seata/releases/tag/v2.6.0)
- [Seata 官方文档](https://seata.apache.org/)
