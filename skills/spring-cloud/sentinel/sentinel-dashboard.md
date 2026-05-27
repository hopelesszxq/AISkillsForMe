---
name: sentinel-dashboard
description: Sentinel Dashboard 部署配置与生产运维最佳实践
tags: [spring-cloud, sentinel, dashboard, monitoring]
---

## Sentinel Dashboard 部署

### Docker 快速部署

```bash
docker run -d --name sentinel-dashboard \
  -p 8858:8858 \
  -p 8719:8719 \
  -e AUTH_USERNAME=sentinel \
  -e AUTH_PASSWORD=sentinel@2024 \
  bladex/sentinel-dashboard:1.8.8
```

### 生产环境部署

```bash
# 下载并定制启动
wget https://github.com/alibaba/Sentinel/releases/download/1.8.8/sentinel-dashboard-1.8.8.jar

java -Dserver.port=8858 \
     -Dcsp.sentinel.dashboard.server=localhost:8858 \
     -Dproject.name=sentinel-dashboard \
     -Dsentinel.dashboard.auth.username=sentinel \
     -Dsentinel.dashboard.auth.password=your-strong-password \
     -Dserver.servlet.session.timeout=7200 \
     -Xms512m -Xmx512m \
     -jar sentinel-dashboard-1.8.8.jar
```

### Docker Compose 编排

```yaml
# docker-compose.yml
version: '3.8'
services:
  sentinel-dashboard:
    image: bladex/sentinel-dashboard:1.8.8
    container_name: sentinel-dashboard
    ports:
      - "8858:8858"
      - "8719:8719"
    environment:
      - AUTH_USERNAME=sentinel
      - AUTH_PASSWORD=${SENTINEL_PASSWORD}
      # JVM 参数
      - JAVA_OPTS=-Dserver.servlet.session.timeout=7200
    volumes:
      - sentinel-logs:/root/logs/csp
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 1G
        reservations:
          memory: 512M

volumes:
  sentinel-logs:
```

## 客户端接入

### Spring Boot 配置

```yaml
spring:
  application:
    name: order-service
  cloud:
    sentinel:
      transport:
        dashboard: sentinel-dashboard:8858    # Dashboard 地址
        port: 8719                            # 客户端暴露端口（用于心跳 + 规则推送）
        heartbeat-interval-ms: 10000          # 心跳间隔
      eager: true                             # 启动即注册到 Dashboard
      log:
        dir: /var/log/sentinel                # 日志独立目录
        switch-file: /var/log/sentinel/switch # 开关文件
```

### 依赖配置

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-actuator</artifactId>
</dependency>
```

## 规则持久化（生产必配）

Dashboard 默认规则存储在内存中，重启即丢失，生产环境必须持久化。

### Nacos 数据源（推荐）

```yaml
spring:
  cloud:
    sentinel:
      datasource:
        # 流控规则
        flow-rules:
          nacos:
            server-addr: ${NACOS_ADDR:localhost:8848}
            namespace: ${NACOS_NAMESPACE:dev}
            data-id: ${spring.application.name}-sentinel-flow
            group-id: SENTINEL
            rule-type: flow
        # 熔断规则
        degrade-rules:
          nacos:
            server-addr: ${NACOS_ADDR:localhost:8848}
            namespace: ${NACOS_NAMESPACE:dev}
            data-id: ${spring.application.name}-sentinel-degrade
            group-id: SENTINEL
            rule-type: degrade
        # 热点规则
        param-flow-rules:
          nacos:
            server-addr: ${NACOS_ADDR:localhost:8848}
            data-id: ${spring.application.name}-sentinel-param-flow
            group-id: SENTINEL
            rule-type: param-flow
```

### Nacos 配置示例

```json
// dataId: order-service-sentinel-flow, group: SENTINEL
[
    {
        "resource": "GET:/api/order/create",
        "limitApp": "default",
        "grade": 1,
        "count": 100,
        "strategy": 0,
        "controlBehavior": 0,
        "clusterMode": false
    }
]
```

### Apollo 数据源

```yaml
spring:
  cloud:
    sentinel:
      datasource:
        flow-rules:
          apollo:
            namespace: application
            flowRulesKey: sentinel.flow.rules
            rule-type: flow
```

## Dashboard 运维功能

### 机器列表管理

```
Dashboard 页面 → 机器列表
- 查看所有已接入的客户端
- 健康状态检查（心跳检测）
- 手动断开/下线
```

### 实时监控

```
Dashboard 页面 → 实时监控
- QPS 曲线图（5 分钟内实时数据）
- 每分钟聚合统计
- 异常流量快速定位
```

### 集群流控（Cluster Flow Control）

```yaml
# Token Server 配置（在某个客户端上启动）
spring:
  cloud:
    sentinel:
      cluster:
        server:
          enable: true
          port: 18730
          # 命名空间配置
        client:
          connect-timeout: 3000
```

```yaml
# Token Client 配置
spring:
  cloud:
    sentinel:
      cluster:
        client:
          server-host: ${sentinel.token-server:192.168.1.100}
          server-port: 18730
          request-timeout: 200
```

## 生产配置调优

### 客户端参数优化

```yaml
spring:
  cloud:
    sentinel:
      # 预热的 QPS 统计窗口
      metric:
        charset: UTF-8
        file-single-size: 52428800        # 单个日志文件 50MB
        file-total-count: 6               # 日志文件总数
        # 每秒滑动窗口统计
      stat:
        max-single-rt: 60000              # 最大响应时间（毫秒）
        # 冷启动因子
      flow:
        cold-factor: 3
```

### JVM 参数

```bash
java -Xms256m -Xmx512m \
     -Dcsp.sentinel.statistic.max.rt=60000 \
     -Dcsp.sentinel.app.type=1 \
     -Dcsp.sentinel.log.output.type=console \
     -jar app.jar
```

## 与 Gateway 集成

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - name: SentinelGatewayFilter
              args:
                fallbackUri: "forward:/fallback"
            - name: RequestRateLimiter
              args:
                key-resolver: '#{@ipKeyResolver}'
                redis-rate-limiter:
                  replenishRate: 100
                  burstCapacity: 200
```

```java
// 网关自定义 API 分组
@Configuration
public class GatewayConfig {

    @PostConstruct
    public void initGatewayRules() {
        // 定义 API 分组
        Set<ApiDefinition> definitions = new HashSet<>();
        ApiDefinition userApi = new ApiDefinition("user_api")
            .setPredicateItems(new HashSet<>(Arrays.asList(
                new ApiPathPredicateItem().setPattern("/api/user/**")
                    .setMatchStrategy(SentinelConstants.URL_MATCH_STRATEGY_PREFIX)
            )));
        definitions.add(userApi);
        GatewayApiDefinitionManager.loadApiDefinitions(definitions);
    }
}
```

## 注意事项

1. **Dashboard 安全加固**：生产环境务必修改默认密码，建议前置 Nginx 做 HTTPS + 认证
2. **日志清理**：Sentinel 日志默认在 `~/logs/csp/`，长时间运行会占用大量磁盘，配置日志轮转
3. **Dashboard 高可用**：Dashboard 本身是单点，生产可以用 Nginx 负载均衡，但需注意客户端配置
4. **规则推送一致性**：使用 Nacos 数据源后，Dashboard 上的修改不会自动同步到 Nacos，需通过 Nacos API/控制台修改
5. **集群流控网络要求**：Token Server 与 Client 延迟 < 10ms，否则影响性能
6. **版本兼容性**：Sentinel 1.8.x 对应 spring-cloud-alibaba 2021.x/2022.x，注意版本匹配
7. **内存占用**：Dashboard 本身轻量（256MB 足够），但客户端越多，Dashboard 内存越高
8. **关闭 sentinel 的方式**：`spring.cloud.sentinel.enabled=false`（测试/本地开发时可关闭）
