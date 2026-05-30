---
name: helm-spring-boot
description: Helm Chart 最佳实践：Spring Boot 微服务 Kubernetes 部署的 Helm 模板化、配置管理与 CI/CD
tags: [tools, kubernetes, helm, spring-boot, deployment, k8s]
---

## 概述

Helm 是 Kubernetes 的包管理工具。相比裸 YAML（kubectl），Helm 通过 **模板化 + 参数化** 让 Spring Boot 微服务在不同环境间复用部署配置，是生产级 K8s 部署的标配。

## 目录结构规范

```text
helm/
├── Chart.yaml              # 图表元信息
├── values.yaml             # 默认配置（开发环境默认值）
├── values-dev.yaml         # 开发环境覆盖
├── values-test.yaml        # 测试环境覆盖
├── values-prod.yaml        # 生产环境覆盖
├── templates/
│   ├── _helpers.tpl        # 模板辅助函数
│   ├── deployment.yaml     # Deployment
│   ├── service.yaml        # Service
│   ├── ingress.yaml        # Ingress
│   ├── configmap.yaml      # Spring Boot application.yml
│   ├── secret.yaml         # 敏感信息
│   ├── hpa.yaml            # 自动伸缩
│   ├── serviceaccount.yaml # 服务账号
│   └── NOTES.txt           # 安装后提示
└── .helmignore             # 排除文件
```

## Chart.yaml

```yaml
apiVersion: v2
name: order-service
description: A Helm chart for Spring Boot order-service
type: application
version: 0.1.0
appVersion: "1.0.0"
keywords:
  - spring-boot
  - microservice
  - java
maintainers:
  - name: Team Order
    email: order-team@example.com
```

## values.yaml（带环境默认值）

```yaml
# === 全局配置 ===
replicaCount: 1

image:
  repository: registry.example.com/order-service
  tag: latest
  pullPolicy: IfNotPresent
  # 镜像拉取密钥
  pullSecrets: []

# === Spring Boot 应用配置 ===
spring:
  profiles:
    active: dev
  application:
    name: order-service

# Nacos 配置
nacos:
  server-addr: "nacos-headless:8848"
  namespace: "dev"
  config:
    enabled: true
    group: DEFAULT_GROUP

# === 资源限制 ===
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi

# === 健康检查 ===
probes:
  liveness:
    enabled: true
    path: /actuator/health/liveness
    initialDelaySeconds: 30
    periodSeconds: 10
    failureThreshold: 3
  readiness:
    enabled: true
    path: /actuator/health/readiness
    initialDelaySeconds: 20
    periodSeconds: 5

# === 服务配置 ===
service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

# === Ingress ===
ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: order.example.com
      paths:
        - path: /api/order
          pathType: Prefix
  tls:
    - secretName: order-tls
      hosts:
        - order.example.com

# === 自动伸缩 ===
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

# === Java JVM 参数 ===
jvm:
  opts: "-Xmx512m -Xms512m -XX:+UseZGC -XX:MaxRAMPercentage=75.0"
  # ZGC（Z Garbage Collector）是 JDK 21+ Spring Boot 3.x 推荐的低延迟 GC
  # -XX:MaxRAMPercentage 比 -Xmx 更适合容器环境

# === 日志 ===
logging:
  level: INFO
  output: json       # JSON 格式日志，方便 EFK/Loki 收集

# === 环境变量（额外） ===
extraEnv: []
#  - name: MY_VAR
#    value: my-value

# === Pod 安全 Context ===
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000

securityContext:
  capabilities:
    drop: ["ALL"]
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
```

## templates/configmap.yaml（Spring Boot 配置注入）

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "chart.fullname" . }}-config
  labels:
    {{- include "chart.labels" . | nindent 4 }}
data:
  application.yml: |
    spring:
      application:
        name: {{ .Values.spring.application.name }}
      profiles:
        active: {{ .Values.spring.profiles.active }}
      cloud:
        nacos:
          discovery:
            server-addr: {{ .Values.nacos.server-addr }}
            namespace: {{ .Values.nacos.namespace }}
          config:
            server-addr: {{ .Values.nacos.server-addr }}
            namespace: {{ .Values.nacos.namespace }}
            {{- if .Values.nacos.config.enabled }}
            enabled: true
            group: {{ .Values.nacos.config.group }}
            {{- end }}
    logging:
      level:
        root: {{ .Values.logging.level }}
      pattern:
        console: {{ `{"timestamp":"%d","level":"%p","service":"${spring.application.name}","thread":"%t","message":"%m"}` | quote }}
    {{- if .Values.logging.output | eq "json" }}
      # JSON 日志输出
      charset:
        console: UTF-8
    {{- end }}
---
# Bootstrapping 配置（Nacos 优先）
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "chart.fullname" . }}-bootstrap
  labels:
    {{- include "chart.labels" . | nindent 4 }}
data:
  bootstrap.yml: |
    spring:
      cloud:
        nacos:
          config:
            server-addr: {{ .Values.nacos.server-addr }}
            namespace: {{ .Values.nacos.namespace }}
            file-extension: yaml
            refresh-enabled: true
```

## templates/deployment.yaml（核心）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "chart.fullname" . }}
  labels:
    {{- include "chart.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "chart.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.image.pullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "chart.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      volumes:
        - name: config
          configMap:
            name: {{ include "chart.fullname" . }}-config
        - name: bootstrap
          configMap:
            name: {{ include "chart.fullname" . }}-bootstrap
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          # Spring Boot 健康检查
          {{- if .Values.probes.liveness.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
            failureThreshold: {{ .Values.probes.liveness.failureThreshold }}
          {{- end }}
          {{- if .Values.probes.readiness.enabled }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
          {{- end }}
          # JVM 配置
          env:
            - name: JAVA_OPTS
              value: {{ .Values.jvm.opts | quote }}
            - name: SPRING_CONFIG_IMPORT
              value: "optional:configtree:/etc/config/"
            {{- with .Values.extraEnv }}
            {{- toYaml . | nindent 12 }}
            {{- end }}
          volumeMounts:
            - name: config
              mountPath: /etc/config/application.yml
              subPath: application.yml
            - name: bootstrap
              mountPath: /etc/config/bootstrap.yml
              subPath: bootstrap.yml
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

## templates/_helpers.tpl

```yaml
{{- define "chart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "chart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{- define "chart.labels" -}}
helm.sh/chart: {{ include "chart.name" . }}-{{ .Chart.Version | replace "+" "_" }}
{{ include "chart.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "chart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{- define "chart.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "chart.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

## 多环境部署

```bash
# 安装到不同环境
helm install order-dev ./helm -f helm/values-dev.yaml -n dev
helm install order-test ./helm -f helm/values-test.yaml -n test
helm upgrade --install order-prod ./helm -f helm/values-prod.yaml -n production

# 环境差异示例：
# values-dev.yaml
replicaCount: 1
resources:
  requests:
    cpu: 250m
    memory: 256Mi

# values-prod.yaml
replicaCount: 3
resources:
  requests:
    cpu: 1
    memory: 2Gi
autoscaling:
  minReplicas: 3
  maxReplicas: 20
```

## CI/CD 集成（GitHub Actions）

```yaml
# .github/workflows/deploy.yml
name: Deploy to K8s

on:
  push:
    branches: [main]
    paths:
      - 'helm/**'
      - 'src/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Build with Maven
        run: ./mvnw clean package -DskipTests

      - name: Build & Push Docker Image
        run: |
          docker build -t registry.example.com/order-service:${{ github.sha }} .
          docker push registry.example.com/order-service:${{ github.sha }}

      - name: Deploy via Helm
        run: |
          helm upgrade --install order-service ./helm \
            --namespace production \
            -f helm/values-prod.yaml \
            --set image.tag=${{ github.sha }} \
            --set image.repository=registry.example.com/order-service \
            --wait --timeout 5m
```

## Helm 部署注意事项

1. **不要硬编码环境值**：所有环境差异通过 `values-{env}.yaml` 管理，values.yaml 只放默认值
2. **ConfigMap 不可变配置**：application.yml 通过 ConfigMap 注入，敏感信息用 Secret（而非 ConfigMap）
3. **JVM 容器适配**：使用 `-XX:MaxRAMPercentage=75.0` 而非 `-Xmx`，让 JVM 自适应容器内存限制
4. **Spring Boot 3.x Actuator 端点**：使用 `/actuator/health/liveness` 和 `/actuator/health/readiness` 区分存活和就绪探针
5. **镜像拉取策略**：dev 环境用 `Always`，prod 用 `IfNotPresent`，避免每次拉取浪费带宽
6. **`helm lint` 检查**：提交前运行 `helm lint ./helm` 验证模板语法
7. **`helm diff` 预览**：安装 `helm-diff` 插件，每次部署前用 `helm diff upgrade` 预览变更
8. **Rollback 策略**：设置 `--history-max 5` 限制历史版本数量，保留最近 5 个版本用于回滚
