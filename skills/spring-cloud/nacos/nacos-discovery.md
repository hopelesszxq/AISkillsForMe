---
name: nacos-discovery
description: Nacos 服务注册发现与健康检查实战
tags: [spring-cloud, nacos, discovery, load-balance]
---

## 服务注册与发现配置

### 基础配置

```yaml
spring:
  application:
    name: order-service

  cloud:
    nacos:
      discovery:
        server-addr: 192.168.1.100:8848    # Nacos 服务器地址
        namespace: prod                     # 命名空间（环境隔离）
        group: DEFAULT_GROUP               # 分组
        cluster-name: HZ                   # 集群名（同城多机房：HZ/SH/BJ）
        register-enabled: true             # 是否注册到 Nacos
        ip: 192.168.1.10                   # 注册 IP（多网卡时必配）
        port: 8080                         # 注册端口
        weight: 1.0                        # 权重（负载均衡比重）
        metadata:
          version: 1.0.0                   # 版本号（灰度发布用）
          region: cn-hangzhou
  # 健康检查端口（用于 Nacos 主动探测）
  server:
    port: 8080
  # actuator 提供健康检查端点
  management:
    endpoints:
      web:
        exposure:
          include: health,info
```

### 多环境隔离

```yaml
# application-dev.yml → namespace: dev
# application-prod.yml → namespace: prod
```

```yaml
spring:
  cloud:
    nacos:
      discovery:
        namespace: ${NACOS_NAMESPACE:public}  # 通过环境变量控制
```

## 健康检查机制

Nacos 通过两种方式判断服务实例健康状态：

| 类型 | 说明 | 推荐场景 |
|---|---|---|
| **主动探测** | Nacos 服务端向客户端 `/actuator/health` 发送 HTTP 请求 | 推荐，更准确 |
| **心跳上报** | 客户端每 5 秒向 Nacos 发送心跳包 | 默认机制 |

### 启用主动探测

```yaml
spring:
  cloud:
    nacos:
      discovery:
        heart-beat-enabled: false           # 关闭心跳（开启主动探测时）
        healthy-check-enabled: true         # 开启主动健康检查（Nacos 服务端配置）
```

### 元数据 + 自定义健康检查

```java
@Component
@Slf4j
public class CustomHealthIndicator implements HealthIndicator {

    private final DatabaseChecker databaseChecker;
    private final RedisChecker redisChecker;

    @Override
    public Health health() {
        boolean dbOk = databaseChecker.isConnected();
        boolean redisOk = redisChecker.isConnected();

        if (dbOk && redisOk) {
            return Health.up()
                .withDetail("database", "connected")
                .withDetail("redis", "connected")
                .build();
        }

        // 数据库不可用 → 标记为 DOWN，Nacos 会自动摘除该实例
        return Health.down()
            .withDetail("database", dbOk ? "ok" : "disconnected")
            .withDetail("redis", redisOk ? "ok" : "disconnected")
            .build();
    }
}
```

## 负载均衡与权重

### Nacos 权重负载

```yaml
# 服务 A 配置 weight: 2.0 → 承接 2 倍流量
# 服务 B 配置 weight: 1.0 → 承接正常流量
# 权重为 0 → 不接收流量（优雅上下线）
```

```java
// Spring Cloud LoadBalancer + Nacos 权重
@Bean
public ReactorLoadBalancer<ServiceInstance> nacosLoadBalancer(
        Environment environment, NacosDiscoveryProperties nacosProperties) {
    String name = environment.getProperty("spring.application.name");
    return new NacosLoadBalancer(
        nacosProperties.getNamingService(), name,
        nacosProperties.getGroup());
}
```

### 集群优先调用

```yaml
spring:
  cloud:
    nacos:
      discovery:
        cluster-name: HZ
```

开启集群优先后，消费者优先调用同集群的服务提供者，同集群无可用实例时才跨集群调用（减少跨机房延迟）。

## 灰度发布（金丝雀部署）

利用 Nacos 元数据实现灰度路由。

### 服务提供方标记版本

```yaml
spring:
  cloud:
    nacos:
      discovery:
        metadata:
          version: ${SERVICE_VERSION:1.0.0}     # 稳定版
          gray-version: ${GRAY_VERSION:false}   # 灰度标志
```

### 消费方灰度路由

```java
@Component
@Slf4j
public class GrayLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    private final NacosNamingService namingService;
    private final String serviceId;

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        // 从请求头获取灰度标识
        String grayTag = getGrayTagFromRequest(request);

        return Mono.fromSupplier(() -> {
            try {
                List<Instance> instances;
                if (grayTag != null) {
                    // 灰度请求 → 只路由到灰度实例
                    instances = namingService.selectInstances(serviceId, true, true)
                        .stream()
                        .filter(inst -> "true".equals(inst.getMetadata().get("gray-version")))
                        .collect(Collectors.toList());

                    log.info("灰度路由: service={}, instance={}", serviceId,
                        instances.stream().map(Instance::toInetAddr).collect(Collectors.toList()));
                } else {
                    // 正常请求 → 只路由到稳定版
                    instances = namingService.selectInstances(serviceId, true, true)
                        .stream()
                        .filter(inst -> !"true".equals(inst.getMetadata().get("gray-version")))
                        .collect(Collectors.toList());
                }

                if (instances.isEmpty()) {
                    instances = namingService.selectInstances(serviceId, true, true);
                }

                return new Response<>(new NacosServiceInstance(
                    instances.get(ThreadLocalRandom.current().nextInt(instances.size()))));
            } catch (NacosException e) {
                log.error("灰度路由失败", e);
                return new Response<>(null);
            }
        });
    }
}
```

## 优雅上下线

### 服务下线

```bash
# 方式一：API 调用（推荐，用于 CI/CD 流程）
curl -X PUT "http://nacos:8848/nacos/v1/ns/instance?serviceName=order-service&ip=192.168.1.10&port=8080&enabled=false"

# 方式二：权重置零（流量逐渐降为零，但注册信息保留）
# 在 Nacos 控制台修改 weight=0
```

### Spring 优雅停机关闭服务发现

```yaml
# 先摘除流量，再关闭服务
spring:
  cloud:
    nacos:
      discovery:
        deregister-enabled: false   # 关闭自动反注册，由外部脚本处理
```

PreStop 钩子（K8s 场景）：

```yaml
# deployment.yaml
lifecycle:
  preStop:
    exec:
      command:
        - /bin/sh
        - -c
        - |
          curl -X PUT "http://nacos:8848/nacos/v1/ns/instance?serviceName=order-service&ip=$POD_IP&port=8080&enabled=false"
          sleep 30  # 等待流量排空
```

## 注意事项

1. **Nacos 2.x 使用 gRPC 端口**：客户端连接 Nacos 除了 8848（HTTP）还需要 9848（gRPC），防火墙必须放开
2. **多网卡问题**：Docker/K8s 部署时必须通过 `spring.cloud.nacos.discovery.ip` 明确指定注册 IP，否则会注册容器内网 IP
3. **健康检查频率**：Nacos 2.x 心跳间隔 5s，健康检查超时 15s，实例摘除需要约 30s
4. **临时/持久实例**：微服务场景用临时实例（临时=true），Ephemeral 实例通过心跳管理；持久实例需手动注销
5. **保护阈值**：设置 `protectThreshold=0.8` 表示健康实例比例低于 80% 时触发保护，全部实例均参与流量转发
6. **Nacos 集群部署**：至少 3 节点，搭配 Nginx 做反向代理，AP 模式下保证可用性
7. **Spring Cloud 版本兼容性**：Spring Cloud 2023.0.x 对应 Nacos 2.3.x，注意版本对应关系
