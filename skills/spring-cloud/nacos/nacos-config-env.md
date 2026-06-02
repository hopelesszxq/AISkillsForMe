---
name: nacos-config-env
description: Nacos 多环境配置管理与环境隔离最佳实践
tags: [spring-cloud, nacos, config, environment]
---

## 环境隔离方案

Nacos 提供三种环境隔离方式：

| 方案 | 粒度 | 适用场景 |
|---|---|---|
| **Namespace** | 租户级隔离 | 开发/测试/生产环境完全隔离 |
| **Group** | 逻辑分组 | 同一环境内按业务模块分组 |
| **Data ID 前缀** | 文件级命名 | 按应用名+环境后缀命名 |

### 推荐方案：Namespace + Group 组合

```
nacos-server
├── Namespace: dev（开发环境）
│   ├── Group: DEFAULT_GROUP
│   │   ├── order-service-dev.yml
│   │   └── user-service-dev.yml
│   └── Group: SHARED
│       ├── datasource-dev.yml
│       └── redis-dev.yml
├── Namespace: staging（预发布）
│   ├── Group: DEFAULT_GROUP
│   │   ├── order-service-staging.yml
│   │   └── user-service-staging.yml
│   └── Group: SHARED
│       ├── datasource-staging.yml
│       └── redis-staging.yml
└── Namespace: prod（生产）
    ├── Group: DEFAULT_GROUP
    ├── Group: SHARED
    └── Group: GRAY（灰度机器专用）
        └── order-service-gray.yml
```

## Spring Boot 配置

### application.yml 多环境配置

```yaml
spring:
  application:
    name: order-service
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}
  cloud:
    nacos:
      config:
        server-addr: ${NACOS_ADDR:127.0.0.1:8848}
        namespace: ${NACOS_NAMESPACE:dev}
        group: DEFAULT_GROUP
        file-extension: yml
        # 共享配置
        shared-configs:
          - data-id: datasource-${spring.profiles.active}.yml
            group: SHARED
            refresh: true
          - data-id: redis-${spring.profiles.active}.yml
            group: SHARED
            refresh: true
          - data-id: common-${spring.profiles.active}.yml
            group: SHARED
            refresh: true
        # 扩展配置
        extension-configs:
          - data-id: order-service-${spring.profiles.active}.yml
            group: DEFAULT_GROUP
            refresh: true
```

### Spring Cloud Alibaba 2025.x 配置（推荐）

```yaml
spring:
  cloud:
    nacos:
      config:
        server-addr: ${NACOS_ADDR}
        namespace: ${NACOS_NAMESPACE}
        # 使用 import 方式（代替 shared-configs）
        import:
          - nacos:datasource-${spring.profiles.active}.yml?group=SHARED&refreshEnabled=true
          - nacos:redis-${spring.profiles.active}.yml?group=SHARED&refreshEnabled=true
          - nacos:${spring.application.name}-${spring.profiles.active}.yml?group=DEFAULT_GROUP&refreshEnabled=true
```

## CI/CD 环境变量注入

### Kubernetes 部署示例

```yaml
# k8s-deployment.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: order-service
          image: registry.example.com/order-service:${BUILD_TAG}
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "staging"
            - name: NACOS_ADDR
              value: "nacos-headless.nacos.svc.cluster.local:8848"
            - name: NACOS_NAMESPACE
              value: "staging-namespace-id"
```

### Docker Compose 本地多环境

```yaml
# docker-compose.yml
services:
  order-service:
    image: order-service:latest
    environment:
      - NACOS_ADDR=nacos:8848
      - NACOS_NAMESPACE=${NACOS_NS:-dev}
      - SPRING_PROFILES_ACTIVE=${ENV:-dev}
```

## 配置管理操作

### Nacos Open API 批量导入

```bash
# 导入配置到指定 Namespace
curl -X POST 'http://nacos-server:8848/nacos/v1/cs/configs?import=true&namespace=prod-ns-id' \
  -F 'file=@configs.zip'

# 导出配置
curl -X GET 'http://nacos-server:8848/nacos/v1/cs/configs?export=true&namespace=dev-ns-id&group=DEFAULT_GROUP,SHARED' \
  -o nacos-configs.zip
```

### 配置同步脚本（环境间复制）

```bash
#!/bin/bash
# sync-configs.sh：从 dev 同步配置到 staging

NACOS_ADDR="nacos-server:8848"
SRC_NS="dev-ns-id"
DST_NS="staging-ns-id"

# 1. 导出源环境配置
curl -s "http://${NACOS_ADDR}/nacos/v1/cs/configs?export=true&namespace=${SRC_NS}&group=DEFAULT_GROUP,SHARED" \
  -o /tmp/nacos-configs.zip

# 2. 导入目标环境（自动覆盖）
curl -s -X POST "http://${NACOS_ADDR}/nacos/v1/cs/configs?import=true&namespace=${DST_NS}&policy=OVERWRITE" \
  -F "file=@/tmp/nacos-configs.zip"

echo "Config synced from ${SRC_NS} to ${DST_NS}"
```

## 动态配置刷新

### @RefreshScope 使用

```java
@Component
@RefreshScope
public class DynamicConfigHolder {

    @Value("${order.timeout:5000}")
    private int orderTimeout;

    @Value("${order.max-retry:3}")
    private int maxRetry;

    public int getOrderTimeout() {
        return orderTimeout;
    }

    public int getMaxRetry() {
        return maxRetry;
    }
}
```

### Nacos 监听器（编程式监听）

```java
@Component
public class NacosConfigWatcher implements InitializingBean {

    @Autowired
    private NacosConfigManager configManager;

    @Override
    public void afterPropertiesSet() throws Exception {
        ConfigService configService = configManager.getConfigService();

        configService.addListener("order-service-dev.yml", "DEFAULT_GROUP", new Listener() {
            @Override
            public Executor getExecutor() {
                return Executors.newSingleThreadExecutor();
            }

            @Override
            public void receiveConfigInfo(String configInfo) {
                // 配置变更回调
                log.info("Config changed: {}", configInfo);
                // 执行自定义逻辑
                reloadBizConfig(configInfo);
            }
        });
    }
}
```

## 灰度配置

利用 Nacos 的 Beta Publish 实现灰度发布：

```yaml
# 场景：只对特定 IP 的机器推送新配置
# Nacos 控制台 -> 配置管理 -> Beta 发布
# 配置：order.payment.switch=v2
# Beta IP：10.0.1.100,10.0.1.101
```

```java
// 代码中读取，无需特殊处理
@Value("${order.payment.switch:v1}")
private String paymentSwitch;

public void processPayment(Order order) {
    if ("v2".equals(paymentSwitch)) {
        // 新支付流程
    } else {
        // 旧支付流程
    }
}
```

## 注意事项

- **Namespace ID 不可变**：创建后不能修改，建议规划好命名规范（dev/staging/prod）
- **配置漂移**：禁止手动修改生产环境 Nacos 控制台配置，所有变更通过 CI/CD Pipeline + 审批流
- **敏感信息加密**：密码、密钥等使用 Nacos 配置加密插件或 Spring Cloud Config 的 `{cipher}` 前缀加密
- **共享配置刷新**：`shared-configs` 的 `refresh: true` 可能影响所有引用该配置的服务，谨慎修改
- **配置版本回溯**：Nacos 控制台支持历史版本查看和回滚，运维时优先使用回滚而非手动编辑
- **Namespaces 数量**：不要为每个微服务单独创建 Namespace，按环境切分即可（一般 3-5 个 Namespace）
- **Group 规划**：`DEFAULT_GROUP` 放应用级配置，`SHARED` 放公共配置（数据库、Redis），`PRIVATE` 放保密配置
