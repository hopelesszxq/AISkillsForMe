---
name: sentinel-rule-persistence
description: Sentinel 规则持久化：Nacos/Apollo 动态数据源与生产运维
tags: [spring-cloud, sentinel, rule-persistence, nacos, apollo]
---

## 规则持久化方案

| 方案 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| **Nacos 数据源** | Sentinel 监听 Nacos 配置变更 | 与 Spring Cloud 生态集成好 | 依赖 Nacos 可用性 |
| **Apollo 数据源** | Sentinel 监听 Apollo 配置变更 | 配置管理成熟、灰度发布 | 多一个组件依赖 |
| **文件数据源** | 定期从文件加载规则 | 不依赖外部组件 | 不方便动态变更 |
| **Push 模式（最佳）** | Dashboard 推送到 Nacos → Sentinel 监听 | 规则持久化 + 实时生效 | 需额外开发 Dashboard |

## Nacos 动态数据源（推荐）

### 1. 依赖引入

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
</dependency>
```

### 2. 配置数据源

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
      datasource:
        # 流控规则
        ds-flow:
          nacos:
            server-addr: ${nacos.addr:localhost:8848}
            namespace: ${nacos.namespace:sentinel}
            group-id: SENTINEL_GROUP
            data-id: sentinel-flow-rules.json
            rule-type: flow
            data-type: json
        # 熔断降级规则
        ds-degrade:
          nacos:
            server-addr: ${nacos.addr:localhost:8848}
            namespace: ${nacos.namespace:sentinel}
            group-id: SENTINEL_GROUP
            data-id: sentinel-degrade-rules.json
            rule-type: degrade
            data-type: json
        # 热点规则
        ds-param-flow:
          nacos:
            server-addr: ${nacos.addr:localhost:8848}
            namespace: ${nacos.namespace:sentinel}
            group-id: SENTINEL_GROUP
            data-id: sentinel-param-flow-rules.json
            rule-type: param-flow
            data-type: json
        # 系统规则
        ds-system:
          nacos:
            server-addr: ${nacos.addr:localhost:8848}
            namespace: ${nacos.namespace:sentinel}
            group-id: SENTINEL_GROUP
            data-id: sentinel-system-rules.json
            rule-type: system
            data-type: json
        # 授权规则（黑白名单）
        ds-authority:
          nacos:
            server-addr: ${nacos.addr:localhost:8848}
            namespace: ${nacos.namespace:sentinel}
            group-id: SENTINEL_GROUP
            data-id: sentinel-authority-rules.json
            rule-type: authority
            data-type: json
```

### 3. Nacos 规则配置示例

```json
// sentinel-flow-rules.json（Nacos 中配置）
[
  {
    "resource": "getOrder",
    "count": 100.0,
    "grade": 1,
    "limitApp": "default",
    "strategy": 0,
    "controlBehavior": 0,
    "clusterMode": false
  },
  {
    "resource": "createOrder",
    "count": 50.0,
    "grade": 1,
    "limitApp": "default",
    "strategy": 0,
    "controlBehavior": 2,
    "warmUpPeriodSec": 10,
    "maxQueueingTimeMs": 500,
    "clusterMode": false
  }
]

// sentinel-degrade-rules.json
[
  {
    "resource": "getOrder",
    "grade": 0,
    "count": 500.0,
    "timeWindow": 10,
    "minRequestAmount": 5,
    "statIntervalMs": 1000,
    "slowRatioThreshold": 0.2
  }
]
```

### 4. 规则参数说明

```java
// FlowRule 字段
count                // 限流阈值
grade                // 阈值类型：0=线程数, 1=QPS
limitApp             // 来源应用控制
strategy             // 调用关系限流策略
controlBehavior      // 流控效果：0=直接拒绝, 1=Warm Up, 2=匀速排队
warmUpPeriodSec      // Warm Up 预热秒数（controlBehavior=1）
maxQueueingTimeMs    // 最大排队等待时间（controlBehavior=2）

// DegradeRule 字段
grade                // 熔断策略：0=RT, 1=异常比例, 2=异常数
count                // 阈值（RT=最大响应时间, 异常比例=0.0~1.0）
timeWindow           // 熔断时长（秒）
minRequestAmount     // 触发熔断的最小请求数
statIntervalMs       // 统计时长（毫秒）
slowRatioThreshold   // 慢调用比例阈值（grade=0 时生效）
```

## Apollo 数据源

### 1. 依赖引入

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-apollo</artifactId>
</dependency>
```

### 2. 配置数据源

```yaml
spring:
  cloud:
    sentinel:
      datasource:
        ds-flow:
          apollo:
            namespace-name: application
            flow-rules-key: sentinel.flow.rules
            rule-type: flow
            data-type: json
```

### 3. Apollo 配置

```properties
# Apollo 中配置
sentinel.flow.rules = [
  {
    "resource": "getOrder",
    "count": 100,
    "grade": 1,
    "controlBehavior": 0
  }
]
```

## Push 模式（Dashboard + Nacos）

### 1. 改造 Sentinel Dashboard

```bash
# 1. 从 GitHub 下载 Sentinel Dashboard 源码
git clone https://github.com/alibaba/Sentinel.git
cd Sentinel/sentinel-dashboard

# 2. 修改 pom.xml 添加 Nacos 依赖
# 3. 修改 application.properties 配置 Nacos 地址
```

```properties
# sentinel-dashboard application.properties
sentinel.dashboard.auth.enabled=false

# 动态规则存储到 Nacos
sentinel.datasource.nacos.server-addr=localhost:8848
sentinel.datasource.nacos.namespace=sentinel
sentinel.datasource.nacos.group-id=SENTINEL_GROUP
```

### 2. 业务应用配置

```yaml
# 应用端不再配置 datasource，由 Dashboard 推送到 Nacos
# 应用只需配置 transport 连接 Dashboard
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
        client-ip: ${spring.cloud.client.ip-address}
        port: 8719
      eager: true  # 提前初始化，避免首次请求时才注册
```

### 3. 实现数据源动态持久化（核心代码）

```java
// 在业务应用中注入 Sentinel Nacos 数据源（用于接收 Dashboard 推送的规则）
@Configuration
public class SentinelNacosDataSourceConfig {

    @Value("${nacos.addr:localhost:8848}")
    private String serverAddr;

    @PostConstruct
    public void init() {
        // 流控规则
        ReadableDataSource<String, List<FlowRule>> flowDataSource =
            new NacosDataSource<>(serverAddr, "SENTINEL_GROUP",
                "sentinel-flow-rules.json",
                source -> JSON.parseObject(source, new TypeReference<List<FlowRule>>() {}));
        FlowRuleManager.register2Property(flowDataSource.getProperty());

        // 熔断规则
        ReadableDataSource<String, List<DegradeRule>> degradeDataSource =
            new NacosDataSource<>(serverAddr, "SENTINEL_GROUP",
                "sentinel-degrade-rules.json",
                source -> JSON.parseObject(source, new TypeReference<List<DegradeRule>>() {}));
        DegradeRuleManager.register2Property(degradeDataSource.getProperty());

        // 热点规则
        ReadableDataSource<String, List<ParamFlowRule>> paramFlowDataSource =
            new NacosDataSource<>(serverAddr, "SENTINEL_GROUP",
                "sentinel-param-flow-rules.json",
                source -> JSON.parseObject(source, new TypeReference<List<ParamFlowRule>>() {}));
        ParamFlowRuleManager.register2Property(paramFlowDataSource.getProperty());

        // 系统规则
        ReadableDataSource<String, List<SystemRule>> systemDataSource =
            new NacosDataSource<>(serverAddr, "SENTINEL_GROUP",
                "sentinel-system-rules.json",
                source -> JSON.parseObject(source, new TypeReference<List<SystemRule>>() {}));
        SystemRuleManager.register2Property(systemDataSource.getProperty());
    }
}
```

## 规则管理 REST API

```bash
# 查看当前生效的规则
curl http://localhost:8719/api/getRules?type=flow
curl http://localhost:8719/api/getRules?type=degrade
curl http://localhost:8719/api/getRules?type=param-flow
curl http://localhost:8719/api/getRules?type=system

# 查看资源实时监控
curl http://localhost:8719/api/metric?identity=resourceName

# 查看集群信息
curl http://localhost:8719/api/clusterState
```

## Spring Cloud Alibaba 完整配置

```yaml
spring:
  cloud:
    sentinel:
      enabled: true
      eager: true
      transport:
        dashboard: ${sentinel.dashboard:localhost:8080}
        client-ip: ${spring.cloud.client.ip-address}
        port: 8719
        heartbeat-interval-ms: 10000
      datasource:
        ds-flow:
          nacos:
            server-addr: ${nacos.addr:localhost:8848}
            namespace: ${nacos.namespace:public}
            group-id: SENTINEL_GROUP
            data-id: ${spring.application.name}-flow-rules.json
            rule-type: flow
            data-type: json
      filter:
        enabled: true
        order: -2147483648
        url-patterns: /**
        # 排除健康检查等路径
        exclude-urls:
          - /actuator/**
          - /health
      # 需要记录所有资源的 metrics，而不是只记录限流资源
      metric:
        file-single-size: 104857600    # 单文件 100MB
        file-total-count: 6            # 总文件数
        charset: UTF-8
```

## 生产运维

### 1. 规则初始化

```java
@Component
public class SentinelRuleInitializer implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) {
        // 启动时设置默认规则（兜底）
        List<FlowRule> rules = new ArrayList<>();
        FlowRule rule = new FlowRule();
        rule.setResource("default");
        rule.setCount(500);
        rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        rule.setLimitApp("default");
        rules.add(rule);
        FlowRuleManager.loadRules(rules);
        log.info("Sentinel 默认规则已加载");
    }
}
```

### 2. 监控报警

```properties
# sentinel 日志配置
# logging.level.com.alibaba.csp.sentinel=INFO
# sentinel 日志文件（默认 ${user.home}/logs/csp/）
csp.sentinel.log.dir=/var/log/sentinel
csp.sentinel.log.use.pid=true
```

```yaml
# Prometheus + Grafana 监控（需引入 Actuator + Micrometer）
management:
  endpoints:
    web:
      exposure:
        include: sentinel,metrics
  endpoint:
    sentinel:
      enabled: true
  metrics:
    tags:
      application: ${spring.application.name}
      instance: ${spring.cloud.client.ip-address}:${server.port}
```

### 3. 规则备份与恢复

```bash
#!/bin/bash
# 定时备份 Sentinel 规则
curl -s http://localhost:8719/api/getRules?type=flow > /backup/sentinel/flow-rules-$(date +%Y%m%d).json
curl -s http://localhost:8719/api/getRules?type=degrade > /backup/sentinel/degrade-rules-$(date +%Y%m%d).json
```

## 注意事项

1. **规则优先级**：代码规则 < 文件规则 < Nacos/Apollo 规则（后加载覆盖前加载）
2. **首次规则丢失**：Nacos 配置不存在时 Sentinel 用默认空规则，需要先创建 Nacos 配置
3. **Client IP 配置**：`client-ip` 必须配置为可路由 IP，否则 Dashboard 无法连接
4. **端口冲突**：sentinel transport port（默认 8719）如果被占用会自动尝试 8720、8721...
5. **Dashboard 不持久化**：默认 Dashboard 规则存在内存中，重启丢失 → 必须用 Push 模式
6. **规则热更新无感**：Sentinel 监听 Nacos 配置变更后立即生效，不需要重启应用
7. **多环境中规则隔离**：使用 Nacos namespace 隔离 dev/test/prod 的规则
