---
name: nacos-config-gray
description: Nacos 3.2.2 配置灰度发布实战：标签匹配规则、Beta 隔离与回滚
tags: [spring-cloud, nacos, config, grayscale, canary]
---

## Nacos 3.2.2 配置灰度增强

Nacos 3.2.2（2026-05-29 发布）在配置灰度发布方面引入了 **Label 标签匹配机制**，使灰度规则不再限于 IP 白名单，支持更灵活的多维度灰度路由。

## 一、标签匹配灰度规则（3.2.2 新特性）

### 传统 Beta 灰度 vs 标签灰度

| 维度 | Beta IP 灰度 | 标签灰度（3.2.2+） |
|------|-------------|-------------------|
| 匹配方式 | IP 白名单 | 客户端 metadata 标签 |
| 灵活性 | 按 IP | 按版本/集群/自定义标签 |
| 运维成本 | 每次发布手动改 IP 列表 | 标签自动匹配 |
| 云原生适配 | 差（POD IP 不固定） | 好（标签可继承自 Pod Label） |

### 标签灰度配置

```yaml
# Nacos 服务端配置灰度规则（通过 API 或控制台）
# POST /nacos/v1/cs/config/gray
{
  "dataId": "application.yaml",
  "group": "DEFAULT_GROUP",
  "content": "db.url: jdbc:mysql://gray-db.example.com:3306/test",
  "grayLabels": {
    "version": "v2.0",           # 匹配客户端 version=v2.0 的实例
    "region": "cn-shanghai"      # 多条件 AND 逻辑
  },
  "grayMatchType": "LABEL",      # 标签匹配模式
  "grayPriority": 10             # 灰度优先级（数字越大优先级越高）
}
```

### 客户端设置灰度标签

```yaml
# bootstrap.yml — 客户端配置
spring:
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        namespace: prod
        group: DEFAULT_GROUP
        file-extension: yaml
        # 灰度标签（与 Nacos 服务端 grayLabels 匹配）
        tags:
          version: v2.0
          region: cn-shanghai
```

```java
// 程序化设置
@Configuration
public class NacosGrayTagConfig {

    @Bean
    public NacosConfigProperties nacosConfigProperties() {
        NacosConfigProperties props = new NacosConfigProperties();
        props.setServerAddr("127.0.0.1:8848");
        props.setNamespace("prod");

        // 设置灰度标签
        Map<String, String> tags = new HashMap<>();
        tags.put("version", System.getenv("SERVICE_VERSION"));
        tags.put("region", System.getenv("REGION"));
        props.setTags(tags);

        return props;
    }
}
```

## 二、基于 Spring Cloud Bus 的配置刷新

```yaml
# 依赖
# spring-cloud-starter-bus-amqp 或 spring-cloud-starter-bus-kafka
spring:
  cloud:
    bus:
      enabled: true
      refresh:
        enabled: true

# Nacos 配置变更 → Bus 广播刷新 → 所有节点生效
# 无需每个服务单独调用 /actuator/refresh
```

```java
// 配置监听
@Component
@RefreshScope
public class DynamicConfig {

    @Value("${db.url}")
    private String dbUrl;

    @EventListener
    public void onRefresh(RefreshScopeRefreshedEvent event) {
        log.info("配置已刷新，当前 db.url={}", dbUrl);
    }
}
```

## 三、配置灰度发布完整流程

```bash
# 1. 创建灰度配置（标签匹配）
curl -X POST "http://nacos:8848/nacos/v1/cs/config/gray" \
  -H "Content-Type: application/json" \
  -d '{
    "dataId": "application.yaml",
    "group": "DEFAULT_GROUP",
    "content": "feature: new-payment-flow",
    "grayLabels": {"version": "v2.0-golden"},
    "grayMatchType": "LABEL",
    "grayPriority": 10
  }'

# 2. 验证灰度节点配置生效
curl http://gray-service:8080/actuator/env/feature

# 3. 发布到全局（复制灰度配置为正式配置）
# Nacos 控制台 → 配置管理 → 灰度配置 → "发布为正式配置"

# 4. 删除灰度配置（防止残留影响下次发布）
curl -X DELETE "http://nacos:8848/nacos/v1/cs/config/gray?dataId=application.yaml&group=DEFAULT_GROUP"
```

## 四、配置回滚策略

```bash
# 查看配置历史（保留最近 30 天）
curl "http://nacos:8848/nacos/v1/cs/history?dataId=application.yaml&group=DEFAULT_GROUP"

# 回滚到指定历史版本
curl -X POST "http://nacos:8848/nacos/v1/cs/config/rollback" \
  -d "dataId=application.yaml&group=DEFAULT_GROUP&nid=12345"
```

## 五、灰度发布注意事项

### 1. 标签优先级

标签灰度会覆盖同名配置，优先级规则如下：

```
Beta IP 灰度 > 标签灰度(高priority) > 标签灰度(低priority) > 正式配置
```

### 2. 灰度清理

灰度发布完成转为正式后，**务必删除灰度配置**。遗留的灰度配置会在下次配置变更时产生干扰。

### 3. 配置灰度 vs 服务灰度

| 场景 | 推荐方式 |
|------|---------|
| 数据库地址/连接串变更 | 配置灰度（通过标签隔离灰度节点） |
| 功能开关/Feature Flag | 配置灰度（按实例标签） |
| 服务全链路灰度 | 配置灰度 + 服务灰度路由（Nacos Discovery metadata）同时使用 |

### 4. Nacos 3.2.2 升级建议

```yaml
# nacos-server application.properties 新增配置
# 启用标签灰度功能（默认开启）
nacos.config.gray.label-match-enabled=true

# AI mode 配置（如不使用 AI Registry 可关闭）
nacos.functionMode=microservice   # microservice | ai | all
```

### 5. 已知坑点

- **标签值变更不生效**：客户端启动时读取 tags 配置，运行中修改 `spring.cloud.nacos.config.tags` 需要重启生效
- **灰度覆盖冲突**：多个灰度配置匹配同一实例时，按 `grayPriority` 降序生效，相同优先级按更新时间
- **K8s 环境标签传递**：推荐通过 K8s Downward API 将 Pod Label 注入环境变量，再映射到 Nacos tags
