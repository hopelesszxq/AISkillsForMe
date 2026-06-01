---
name: sidecar-pattern
description: Sidecar 模式实战指南：服务网格代理、配置同步、日志收集、安全代理的 Sidecar 容器设计
tags: [patterns, microservice, sidecar, kubernetes, service-mesh, istio]
---

## 概述

**Sidecar 模式**是一种将应用容器的辅助功能（日志、监控、配置、安全等）剥离到独立容器中的架构模式。Sidecar 容器与主容器共享同一 Pod 生命周期，但职责分离，互不侵入。

> 本技能补充 `microservice-patterns.md` 通用微服务模式，聚焦 Sidecar 模式的实战部署。

## 一、Sidecar 模式 vs Ambassador 模式 vs 服务网格

| 模式 | 职责 | 典型实现 |
|------|------|----------|
| **Sidecar** | 辅助主容器（日志、监控、代理） | Filebeat Sidecar、Consul Connect |
| **Ambassador** | 代表主容器访问外部服务 | Redis Sentinel Sidecar、Envoy |
| **Service Mesh** | 基础设施层通信治理（多 Sidecar） | Istio Envoy、Linkerd |

Sidecar 和 Ambassador 本质上都是 Sidecar 模式的不同变体。服务网格则是 Sidecar 模式在微服务通信层面的标准化实现。

## 二、Kubernetes Sidecar 容器（K8s 1.29+）

Kubernetes 从 1.29 开始支持原生 Sidecar 容器的生命周期管理。

### 基础定义

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-service
  template:
    metadata:
      labels:
        app: my-service
    spec:
      containers:
        # 主应用容器
        - name: app
          image: my-app:latest
          ports:
            - containerPort: 8080

        # ✅ K8s 1.29+ 原生 Sidecar 容器
        - name: log-sidecar
          image: fluent/fluent-bit:3.2
          # restartPolicy: Always 表明这是 Sidecar 容器
          # 与主容器同时启动，主容器停止后 Sidecar 也自动停止
          restartPolicy: Always
          volumeMounts:
            - name: logs
              mountPath: /var/log/app
          env:
            - name: FLUENT_BIT_CONFIG
              value: /etc/fluent-bit/conf/fluent-bit.conf

        # ❌ 旧方式（initContainers + sidecar 模拟）
        # 现在应使用原生 Sidecar 声明
      volumes:
        - name: logs
          emptyDir: {}
```

### Sidecar 启动与终止顺序

```yaml
# K8s 1.29+ 原生 Sidecar 的生命周期保证：
# 1. 启动：主容器与 Sidecar 同时启动（不再依赖 initContainers hack）
# 2. 终止：主容器终止后，Sidecar 才收到 SIGTERM
# 3. 保证：Sidecar 先于主容器完成终止

# ❌ 旧方式（initContainers + sidecar）
# initContainers:
#   - name: wait-for-envoy
#     image: envoyproxy/envoy:v1.30
#     # 必须在主容器前先跑起来

# ✅ 新方式（原生 Sidecar）
containers:
  - name: envoy-sidecar
    image: envoyproxy/envoy:v1.33
    restartPolicy: Always  # ✅ 关键标记
    ports:
      - containerPort: 9901  # Envoy admin
```

## 三、典型 Sidecar 场景

### 1. 日志采集 Sidecar

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: app-with-filebeat
spec:
  template:
    spec:
      containers:
        - name: app
          image: my-spring-app:latest
          volumeMounts:
            - name: app-logs
              mountPath: /app/logs

        - name: filebeat
          image: docker.elastic.co/beats/filebeat:8.18.0
          restartPolicy: Always
          volumeMounts:
            - name: app-logs
              mountPath: /app/logs
            - name: filebeat-config
              mountPath: /usr/share/filebeat/filebeat.yml
              subPath: filebeat.yml
      volumes:
        - name: app-logs
          emptyDir: {}
        - name: filebeat-config
          configMap:
            name: filebeat-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
data:
  filebeat.yml: |
    filebeat.inputs:
    - type: log
      paths:
        - /app/logs/*.log
    output.elasticsearch:
      hosts: ['${ELASTICSEARCH_HOST:elasticsearch:9200}']
```

### 2. 配置热更新 Sidecar

用于从配置中心（Nacos、Consul、Spring Cloud Config）拉取配置并写入本地文件，主应用监听文件变化。

```java
// Sidecar 主逻辑（Spring Boot 应用）
@Component
public class ConfigSyncSidecar implements CommandLineRunner {

    private final ConfigService configService;

    @Override
    public void run(String... args) {
        // 监听配置变更
        configService.addListener("app", "DEFAULT_GROUP", new Listener() {
            @Override
            public void receiveConfigInfo(String configInfo) {
                try {
                    // 写入共享卷的配置文件
                    Files.writeString(
                        Path.of("/shared-config/application.yml"),
                        configInfo
                    );
                    log.info("Config synced to shared volume");
                } catch (IOException e) {
                    log.error("Failed to write config", e);
                }
            }

            @Override
            public Executor getExecutor() {
                return Executors.newSingleThreadExecutor();
            }
        });
    }
}
```

### 3. 流量代理 Sidecar（Envoy）

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-config
data:
  envoy.yaml: |
    static_resources:
      listeners:
      - name: ingress_listener
        address: { socket_address: { address: 0.0.0.0, port_value: 8080 } }
        filter_chains:
        - filters:
          - name: envoy.filters.network.http_connection_manager
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              codec_type: AUTO
              stat_prefix: ingress_http
              route_config:
                name: local_route
                virtual_hosts:
                - name: backend
                  domains: ["*"]
                  routes:
                  - match: { prefix: "/" }
                    route:
                      cluster: backend_service
              http_filters:
              - name: envoy.filters.http.router
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
      clusters:
      - name: backend_service
        type: STRICT_DNS
        lb_policy: ROUND_ROBIN
        load_assignment:
          cluster_name: backend_service
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: 127.0.0.1
                    port_value: 9090
```

## 四、Spring Boot 集成示例

### Nacos 配置同步 Sidecar

```java
@SpringBootApplication
public class NacosConfigSidecarApplication {

    public static void main(String[] args) {
        SpringApplication.run(NacosConfigSidecarApplication.class, args);
    }
}

@Service
public class NacosConfigSyncService {

    @Value("${nacos.server-addr}")
    private String serverAddr;

    private static final String SHARED_CONFIG_PATH = "/shared-config/";

    @PostConstruct
    public void init() throws NacosException {
        ConfigService configService = NacosFactory.createConfigService(serverAddr);

        String content = configService.getConfig("application.yml", "DEFAULT_GROUP", 5000);
        writeConfigFile("application.yml", content);

        configService.addListener("application.yml", "DEFAULT_GROUP", new Listener() {
            @Override
            public void receiveConfigInfo(String config) {
                writeConfigFile("application.yml", config);
            }

            @Override
            public Executor getExecutor() {
                return Executors.newSingleThreadExecutor();
            }
        });
    }

    private void writeConfigFile(String fileName, String content) {
        try {
            Path path = Path.of(SHARED_CONFIG_PATH + fileName);
            Files.createDirectories(path.getParent());
            Files.writeString(path, content);
            log.info("Config synced: {}", fileName);
        } catch (IOException e) {
            log.error("Failed to write config file", e);
        }
    }
}
```

### Dockerfile 优化

```dockerfile
# Sidecar 容器应尽可能小
FROM eclipse-temurin:21-jre-alpine AS sidecar-base
RUN addgroup -S sidecar && adduser -S sidecar -G sidecar

FROM sidecar-base
WORKDIR /app
COPY target/config-sidecar.jar app.jar
USER sidecar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 五、常见 Sidecar 实现对比

| 方案 | 协议支持 | 配置复杂度 | 性能开销 | 适用场景 |
|------|---------|-----------|---------|---------|
| **Istio Envoy** | HTTP/gRPC/TCP | 高（CRD 管理） | 中等（5-10ms） | 全面服务网格 |
| **Linkerd** | HTTP/gRPC/TCP | 中（注解驱动） | 低（<2ms） | 轻量服务网格 |
| **Consul Connect** | HTTP/gRPC | 中 | 低 | HashiCorp 生态 |
| **Envoy 独立** | 任意 | 中（YAML 配置） | 中等 | 自定义代理 |
| **Filebeat** | 日志 | 低 | 极低 | 日志采集 |
| **自定义代理** | 自定义 | 高（需开发） | 取决于实现 | 特定需求 |

## 注意事项

| 要点 | 说明 |
|------|------|
| **K8s 版本** | 原生 Sidecar 需 Kubernetes 1.29+，低版本需 initContainers hack |
| **资源配额** | Sidecar 容器也要配置 `resources.requests/limits`，避免资源争抢 |
| **端口冲突** | 确保 Sidecar 与主容器端口不冲突，使用不同端口范围 |
| **共享卷权限** | 多个容器读写共享卷时注意文件权限和锁竞争 |
| **健康检查** | 为 Sidecar 配置独立的 `livenessProbe` 和 `readinessProbe` |
| **日志** | Sidecar 日志应与主容器日志区分（不同文件或不同标签） |
| **滚动更新** | 原生 Sidecar 在主容器更新时保持运行，避免短时间中断 |
| **调试** | `kubectl exec -c <sidecar-name>` 进入特定 Sidecar 容器 |
