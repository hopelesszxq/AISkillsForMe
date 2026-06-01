---
name: minio-k8s-health
description: MinIO Kubernetes 部署健康检查与监控实战：存活探针、就绪探针、Prometheus 指标采集、告警规则
tags: [minio, kubernetes, monitoring, healthcheck, prometheus, alerting]
---

## 概述

在生产环境中运行 MinIO，健康检查和监控是保障可用性的关键。MinIO 提供了丰富的健康检查端点（`/minio/health/live`、`/minio/health/ready` 等）和 Prometheus 兼容指标，用于 Kubernetes 探针配置和集群监控。

> 已有技能 `minio-ops.md` 覆盖了 MinIO 运维基础，本文专注于 K8s 环境下的健康检查与监控体系。

## 一、MinIO 健康检查端点

MinIO 提供以下标准健康检查端点：

| 端点 | 类型 | 返回码 | 说明 |
|------|------|--------|------|
| `/minio/health/live` | 存活探针 | 200 OK | 进程存活，无响应及时重启 |
| `/minio/health/ready` | 就绪探针 | 200 OK | 磁盘 I/O 正常，集群可读写 |
| `/minio/health/cluster` | 集群探针 | 200 OK | 集群所有节点/磁盘正常 |

### 集群模式检查行为

- **独立模式**：`/minio/health/ready` 只检查本地磁盘状态
- **分布式模式**：`/minio/health/ready` 检查所有节点和磁盘状态
- **Write Quorum**：当写入仲裁丢失时，就绪探针返回非 200 状态

## 二、Kubernetes 探针配置

### 基础存活与就绪探针

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: minio
spec:
  serviceName: minio
  replicas: 4
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
        - name: minio
          image: minio/minio:RELEASE.2025-10-15T17-29-55Z
          args:
            - server
            - /data
            - --console-address
            - ":9001"
          env:
            - name: MINIO_ROOT_USER
              valueFrom:
                secretKeyRef:
                  name: minio-secret
                  key: root-user
            - name: MINIO_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: minio-secret
                  key: root-password
          ports:
            - containerPort: 9000  # API
            - containerPort: 9001  # Console
          # ✅ 存活探针：进程是否存活
          livenessProbe:
            httpGet:
              path: /minio/health/live
              port: 9000
            initialDelaySeconds: 10
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3
          # ✅ 就绪探针：磁盘/集群是否可用
          readinessProbe:
            httpGet:
              path: /minio/health/ready
              port: 9000
            initialDelaySeconds: 15
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3
            successThreshold: 1
          resources:
            requests:
              memory: "4Gi"
              cpu: "1"
            limits:
              memory: "8Gi"
              cpu: "4"
          volumeMounts:
            - name: data
              mountPath: /data
```

### 启动探针（Startup Probe）

对于大集群部署，MinIO 启动时间较长，使用启动探针避免存活检查过早触发重启：

```yaml
spec:
  containers:
    - name: minio
      # ...
      startupProbe:
        httpGet:
          path: /minio/health/live
          port: 9000
        initialDelaySeconds: 10
        periodSeconds: 10
        timeoutSeconds: 5
        failureThreshold: 30  # 允许最多 300 秒启动时间
      livenessProbe:
        httpGet:
          path: /minio/health/live
          port: 9000
        periodSeconds: 30
      readinessProbe:
        httpGet:
          path: /minio/health/ready
          port: 9000
        periodSeconds: 15
```

### 使用 mc 客户端作为探针

对于更复杂的检查，可以使用 MinIO Client (`mc`):

```yaml
spec:
  containers:
    - name: minio
      # ...
      livenessProbe:
        exec:
          command:
            - mc
            - ready
            - local
        initialDelaySeconds: 30
        periodSeconds: 30
        timeoutSeconds: 10

      readinessProbe:
        exec:
          command:
            - sh
            - -c
            - |
              mc ready local && \
              mc admin info local --json | \
              grep -q '"status" : "online"'
        initialDelaySeconds: 30
        periodSeconds: 15
        timeoutSeconds: 10
```

## 三、Prometheus 监控集成

### 启用 Prometheus 端点

```bash
# 启动时启用指标采集
export MINIO_PROMETHEUS_AUTH_TYPE="public"
# 或使用 JWT 认证
# export MINIO_PROMETHEUS_AUTH_TYPE="token"
# export MINIO_PROMETHEUS_BEARER_TOKEN="your-jwt-token"

minio server /data --console-address ":9001"
```

### Kubernetes ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: minio-monitor
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: minio
  namespaceSelector:
    matchNames:
      - minio-system
  endpoints:
    - port: api
      path: /minio/v2/metrics/cluster
      interval: 30s
      scheme: http
      # 如果启用了 JWT 认证，配置 Bearer Token
      # bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
    - port: api
      path: /minio/v2/metrics/bucket
      interval: 60s
      scheme: http
    - port: api
      path: /minio/v2/metrics/resource
      interval: 60s
      scheme: http
```

### Grafana Dashboard 关键指标

| 指标 | 说明 | 告警阈值 |
|------|------|---------|
| `minio_cluster_disk_offline_total` | 离线磁盘数 | > 0 |
| `minio_cluster_nodes_offline_total` | 离线节点数 | > 0 |
| `minio_s3_requests_total` | S3 请求总数 | 用于容量规划 |
| `minio_s3_errors_total` | S3 请求错误数 | 持续上升告警 |
| `minio_heal_objects_total` | 自愈对象数 | > 100/分钟 |
| `minio_disk_storage_used_bytes` | 磁盘使用量 | > 80% 告警 |
| `minio_disk_storage_free_bytes` | 磁盘剩余空间 | < 20% 告警 |

## 四、Prometheus 告警规则

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: minio-alerts
  namespace: minio-system
  labels:
    release: prometheus
spec:
  groups:
    - name: minio
      rules:
        # 磁盘离线告警
        - alert: MinioDiskOffline
          expr: minio_cluster_disk_offline_total > 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "MinIO 磁盘离线"
            description: "MinIO {{ $labels.instance }} 有 {{ $value }} 个磁盘离线"

        # 节点离线告警
        - alert: MinioNodeOffline
          expr: minio_cluster_nodes_offline_total > 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "MinIO 节点离线"
            description: "MinIO 集群有 {{ $value }} 个节点离线"

        # 磁盘空间告警
        - alert: MinioDiskSpaceLow
          expr: |
            (minio_disk_storage_used_bytes / minio_disk_storage_total_bytes) * 100 > 85
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "MinIO 磁盘空间不足"
            description: "磁盘使用率 {{ $value | humanizePercentage }}"

        - alert: MinioDiskSpaceCritical
          expr: |
            (minio_disk_storage_used_bytes / minio_disk_storage_total_bytes) * 100 > 95
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "MinIO 磁盘空间严重不足"
            description: "磁盘使用率 {{ $value | humanizePercentage }}，立即扩容"

        # 错误率告警
        - alert: MinioErrorRateHigh
          expr: |
            rate(minio_s3_errors_total[5m]) / rate(minio_s3_requests_total[5m]) * 100 > 5
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "MinIO 错误率过高"
            description: "错误率 {{ $value | humanizePercentage }}"

        # 自愈活动告警
        - alert: MinioHealingActive
          expr: rate(minio_heal_objects_total[5m]) > 0
          for: 10m
          labels:
            severity: info
          annotations:
            summary: "MinIO 自愈进行中"
            description: "对象自愈速率 {{ $value | humanize }} 对象/秒"
```

## 五、日志监控与审计

### 结构化日志配置

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: minio-log-config
data:
  # MinIO 启动参数中加入日志配置
  # 推荐 MinIO 2025+ 版本使用 JSON 格式日志
  json.log: |
    {"json": true, "anonymous": false, "quiet": false}
```

```bash
# 启用 JSON 格式日志输出
export MINIO_JSON_LOGS=on
# 捕获 JSON 格式日志用于 ELK/Loki 采集
```

### 审计日志（Audit Log）

```yaml
# MinIO 支持 Webhook 审计日志
# 配置将操作日志发送到外部端点
apiVersion: v1
kind: Secret
metadata:
  name: minio-audit-webhook
stringData:
  config.env: |
    export MINIO_AUDIT_WEBHOOK_ENABLE_audit=on
    export MINIO_AUDIT_WEBHOOK_ENDPOINT_audit=http://audit-collector:8080/minio-audit
    export MINIO_AUDIT_WEBHOOK_AUTH_TOKEN_audit=your-bearer-token
```

## 六、Kubernetes 事件监控

```yaml
apiVersion: v1
kind: Service
metadata:
  name: minio
  annotations:
    # Prometheus 自动发现
    prometheus.io/scrape: "true"
    prometheus.io/port: "9000"
    prometheus.io/path: "/minio/v2/metrics/cluster"
spec:
  ports:
    - name: api
      port: 9000
      targetPort: 9000
    - name: console
      port: 9001
      targetPort: 9001
  selector:
    app: minio
---
# 弹性伸缩：基于指标自动扩缩
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: minio-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: minio
  minReplicas: 4
  maxReplicas: 16
  metrics:
    - type: Object
      object:
        metric:
          name: minio_disk_storage_used_bytes
        describedObject:
          apiVersion: v1
          kind: Service
          name: minio
        target:
          type: AverageValue
          averageValue: "800Gi"  # 平均超过 800G 触发扩容
```

## 注意事项

| 要点 | 说明 |
|------|------|
| **探针路径** | `ready` 探针在分布式模式下会检查仲裁——部分节点离线时可能误判 |
| **启动探针** | 4 节点以上分布式 MinIO 必须配置 `startupProbe`，防止启动超时被误杀 |
| **资源预留** | `resources.requests` 应基于数据量预估，不足会导致 OOM Kill |
| **持久卷** | StatefulSet 必须使用 PVC，避免 Pod 重建后数据丢失 |
| **Prometheus 认证** | 生产环境建议使用 `MINIO_PROMETHEUS_AUTH_TYPE=token` 保护指标端点 |
| **指标频率** | `/minio/v2/metrics/bucket` 指标数随 Bucket 数增长，采集间隔不宜过短 |
| **磁盘满处理** | 磁盘使用率超过 95% 时 MinIO 进入只读模式，提前监控告警 |
| **版本兼容** | 检查端点 `/minio/health/live` 在 MinIO RELEASE.2023+ 中稳定，旧版本路径可能不同 |
