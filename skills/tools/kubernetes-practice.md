---
name: kubernetes-practice
description: Spring Boot 微服务 Kubernetes 部署实战：Dockerfile、Deployment、Service、Ingress、ConfigMap
tags: [tools, kubernetes, k8s, docker, deployment, spring-boot]
---

## 概述

将 Spring Boot / Spring Cloud 微服务部署到 Kubernetes（K8s）是现代微服务架构的标配。本文覆盖从 Docker 镜像构建到生产级 K8s 部署的全流程最佳实践。

## 1. Spring Boot 容器化（Dockerfile）

### 分层构建（推荐）

```dockerfile
# 1. 构建阶段
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /build
COPY mvnw pom.xml ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline -B
COPY src src
RUN ./mvnw package -DskipTests -B

# 2. 运行阶段（使用 jlink 裁剪 JRE）
FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar
EXPOSE 8080
USER appuser
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 启动脚本/健康检查

```dockerfile
# 优化 JVM 参数，适配容器内存限制
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-XX:+ExitOnOutOfMemoryError", \
  "-jar", "app.jar"]

# Spring Boot 2.3+ 自带 Actuator 健康检查
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1
```

## 2. Deployment 配置

### 基础部署模板

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: production
  labels:
    app: user-service
    version: v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # 最大额外 Pod 数
      maxUnavailable: 0    # 滚动期间保持全部可用
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
        version: v1
    spec:
      terminationGracePeriodSeconds: 60   # 优雅关闭等待时间
      containers:
        - name: user-service
          image: registry.example.com/user-service:1.0.0
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
              name: http
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "k8s"
            - name: JAVA_OPTS
              value: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75"
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secrets
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi
          livenessProbe:         # 存活探针（失败则重启 Pod）
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:        # 就绪探针（失败则从 Service 摘除）
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
            timeoutSeconds: 3
          startupProbe:          # 启动探针（防止慢启动容器被误杀）
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 30  # 允许最长 150s 启动
          lifecycle:
            preStop:             # 优雅关闭：先停止接收新请求
              exec:
                command: ["sh", "-c", "sleep 10"]
```

### Spring Boot 优雅关闭配置

```yaml
# application-k8s.yml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s

# Nacos 等注册中心主动注销
spring.cloud.nacos.discovery:
  deregister-on-stop: true
```

## 3. Service 与 Ingress

### Service 配置

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: production
  labels:
    app: user-service
spec:
  type: ClusterIP
  ports:
    - port: 8080
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app: user-service
```

### Ingress（Traefik / Nginx Ingress）

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: 10m
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: tls-secret
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /api/user
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 8080
          - path: /api/order
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 8080
```

## 4. ConfigMap 与 Secret 管理

### ConfigMap（非敏感配置）

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  application.yml: |
    spring:
      datasource:
        url: jdbc:postgresql://postgres-service:5432/mydb
        hikari:
          maximum-pool-size: 10
          minimum-idle: 2
      redis:
        host: redis-service
        port: 6379
    logging:
      level:
        com.example: DEBUG
```

### Secret（敏感配置，用 base64 或 External Secrets Operator）

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
data:
  DB_PASSWORD: bXlzcWxwYXNzCg==       # base64("mysqlpass")
  REDIS_PASSWORD: cmVkaXNwYXNzCg==    # base64("redispass")
  JWT_SECRET: c3VwZXJzZWNyZXQK        # base64("supersecret")
```

> **生产建议**：使用 External Secrets Operator 或 Sealed Secrets 管理密钥，避免明文 base64 提交到 Git。

## 5. 使用 Helm 管理部署

### 项目结构

```
helm-charts/
  user-service/
    Chart.yaml           # 元数据
    values.yaml          # 默认值
    templates/
      deployment.yaml
      service.yaml
      ingress.yaml
      configmap.yaml
      hpa.yaml
      _helpers.tpl       # 模板辅助函数
```

### 模板示例

```yaml
{{- /* deployment.yaml */ -}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.fullname" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: {{ .Values.spring.profiles }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

```yaml
# values.yaml
replicaCount: 3
image:
  repository: registry.example.com/user-service
  tag: "1.0.0"
service:
  port: 8080
spring:
  profiles: k8s
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi
```

## 6. 自动伸缩（HPA）

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

## 7. Service Mesh 集成（Istio）

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: user-service
  namespace: production
spec:
  hosts:
    - user-service
  http:
    - match:
        - headers:
            version:
              exact: v2
      route:
        - destination:
            host: user-service
            subset: v2
    - route:
        - destination:
            host: user-service
            subset: v1
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: user-service
  namespace: production
spec:
  host: user-service
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
    connectionPool:
      tcp:
        maxConnections: 100
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 60s
```

## 注意事项

1. **容器镜像安全**：使用非 root 用户运行（Dockerfile 中 `USER appuser`），避免用 latest 标签
2. **资源限制必设**：不设 limits 的 Pod 可能耗尽节点资源导致其他服务受影响
3. **三探针配合**：liveness + readiness + startup 三管齐下，避免慢启动服务和滚动更新中断
4. **优雅关闭不可少**：K8s 发送 SIGTERM → Spring Boot 停止接受新请求 → 处理完存量请求 → 注销注册中心 → 退出
5. **日志集中收集**：Pod 日志输出到 stdout/stderr，用 EFK/Loki 集中收集，不要写入文件
6. **配置外置化**：配置文件通过 ConfigMap 挂载，不要打包在镜像内，便于环境切换
7. **健康检查端点**：Spring Boot Actuator 需暴露 `/actuator/health/liveness` 和 `/actuator/health/readiness`（management.endpoints.web.exposure.include=health,info）
8. **镜像拉取策略**：生产环境用 `imagePullPolicy: Always` 或固定版本标签，避免缓存旧镜像
9. **Ingress 统一入口**：所有微服务通过一个 Ingress 暴露，用 path 路由到不同 Service
10. **命名空间隔离**：不同环境（dev/staging/prod）使用不同 Namespace，通过 NetworkPolicy 限制跨 Namespace 访问
