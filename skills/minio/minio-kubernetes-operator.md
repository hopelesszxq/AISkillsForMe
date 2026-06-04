---
name: minio-kubernetes-operator
description: MinIO Operator 在 Kubernetes 上的部署管理：Operator 安装、Tenant 配置、持久卷、自动扩缩容与监控
tags: [minio, kubernetes, operator, storage, s3, k8s]
---

## 概述

MinIO Operator 是 Kubernetes 原生的 MinIO 部署管理方案，通过 CRD（Custom Resource Definition）管理 MinIO 集群的生命周期。相比手动部署 StatefulSet，Operator 提供了**自动扩缩容、故障自动修复、证书管理、统一监控**等能力。

> 已有技能 `minio-k8s-health.md` 覆盖了健康检查配置，本文专注于 Operator 模式部署。

## 1. 安装 MinIO Operator

### 使用 Kustomize 安装

```bash
# 最新稳定版
kubectl apply -k github.com/minio/operator

# 指定版本
kubectl apply -k "github.com/minio/operator?ref=v6.0.3"
```

### 使用 Helm 安装

```bash
helm repo add minio-operator https://operator.min.io
helm repo update

helm install minio-operator minio-operator/operator \
  --namespace minio-operator \
  --create-namespace \
  --set operator.replicaCount=2
```

### 验证安装

```bash
# 查看 Operator Pod
kubectl get pods -n minio-operator

# 查看 CRD
kubectl get crd | grep minio
# 应看到：tenants.minio.min.io, tenants.minio.cnrm.cloud.google.com 等
```

## 2. 创建 MinIO Tenant

MinIO Tenant = 一个独立的 MinIO 集群实例，包含多个 Pod（Pool）。Pool 是 MinIO 的扩缩容单元。

### 基础 Tenant 配置

```yaml
# tenant.yaml
apiVersion: minio.min.io/v2
kind: Tenant
metadata:
  name: my-minio-tenant
  namespace: minio-tenant
spec:
  # MinIO 镜像版本
  image: quay.io/minio/minio:RELEASE.2025-12-01T22-57-13Z
  
  # 服务配置
  serviceMetadata:
    minioService: minio
    consoleService: console
  
  # 存储桶（自动创建）
  buckets:
    - name: uploads
    - name: backups
      # 可指定存储类或标签
  
  # 证书配置（可选，默认自动生成）
  certConfig:
    commonName: minio.example.com
    dnsNames:
      - minio.example.com
      - "*.minio-tenant.svc.cluster.local"
  
  # 配置信息
  configuration:
    name: minio-config
  
  # Pool 配置（核心）
  pools:
    - name: pool-0
      servers: 4                    # Pod 数量
      volumesPerServer: 4           # 每个 Pod 的 PV 数量
      volumeClaimTemplate:
        metadata:
          name: data
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 500Gi        # 每个 PV 大小
          storageClassName: standard
      resources:
        requests:
          memory: 4Gi
          cpu: "2"
        limits:
          memory: 8Gi
          cpu: "4"
  
  # 控制台配置
  console:
    image: quay.io/minio/console:v1.8.0
    replicas: 2
    resources:
      requests:
        memory: 256Mi
        cpu: "0.5"
```

### 创建并访问

```bash
# 创建命名空间
kubectl create namespace minio-tenant

# 应用 Tenant 配置
kubectl apply -f tenant.yaml

# 查看 Tenant 状态
kubectl describe tenant my-minio-tenant -n minio-tenant

# 查看 MinIO Pod
kubectl get pods -n minio-tenant -l app=minio

# 获取 MinIO 终端地址
kubectl get svc -n minio-tenant my-minio-tenant-hl  # 集群内地址
kubectl get svc -n minio-tenant my-minio-tenant      # 对外地址（ClusterIP/NodePort）

# 获取访问凭证
kubectl get secret my-minio-tenant-env-configuration \
  -n minio-tenant -o jsonpath="{.data.config\.env}" | base64 -d
```

## 3. 多 Pool 与扩缩容

### 添加 Pool（水平扩展）

```yaml
# tenant-expand.yaml
spec:
  pools:
    - name: pool-0
      servers: 4
      volumesPerServer: 4
      # ... 原有配置
    - name: pool-1          # 新增 Pool
      servers: 2
      volumesPerServer: 4
      volumeClaimTemplate:
        metadata:
          name: data
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 500Gi
          storageClassName: standard-fast  # 可用不同存储类
```

```bash
# 更新 Tenant
kubectl apply -f tenant-expand.yaml

# 查看 Pool 状态（新 Pool 自动加入集群）
kubectl get pods -n minio-tenant -l app=minio
```

### 自动扩缩容（HPA + 存储指标）

MinIO Operator 支持基于存储使用率的自动扩缩容：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: minio-tenant-autoscaler
  namespace: minio-tenant
spec:
  scaleTargetRef:
    apiVersion: minio.min.io/v2
    kind: Tenant
    name: my-minio-tenant
  minReplicas: 4
  maxReplicas: 16
  metrics:
    - type: Pods
      pods:
        metric:
          name: minio_cluster_capacity_usable_total_bytes
        target:
          type: Utilization
          averageUtilization: 80
```

## 4. 持久卷策略

### 使用本地 PV 提升性能

```yaml
# 推荐：生产环境使用本地 SSD
apiVersion: v1
kind: PersistentVolume
metadata:
  name: minio-local-pv-1
spec:
  capacity:
    storage: 500Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-ssd
  local:
    path: /mnt/ssd/minio-data-1
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - k8s-node-1
```

### 使用 Topology Spread Constraints

```yaml
# 确保 Pod 分散在不同节点
spec:
  pools:
    - name: pool-0
      servers: 4
      topology:
        # 最小化跨 AZ 网络开销
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - us-east-1a
                  - us-east-1b
                  - us-east-1c
                  - us-east-1d
```

## 5. 监控与告警

### Prometheus 集成

MinIO Operator 自动暴露 Prometheus 指标：

```yaml
# tenant 中启用监控
spec:
  metrics:
    image: quay.io/minio/minio:latest
    resources:
      requests:
        memory: 256Mi
        cpu: "0.5"
    # Prometheus 自动发现 annotation
    prometheus:
      enabled: true
      port: 9000
      path: /minio/v2/metrics/cluster
```

```yaml
# Prometheus ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: minio-tenant-monitor
  namespace: minio-tenant
spec:
  selector:
    matchLabels:
      app: minio
  endpoints:
    - port: http
      path: /minio/v2/metrics/cluster
      interval: 15s
  namespaceSelector:
    matchNames:
      - minio-tenant
```

### 关键告警规则

```yaml
# prometheus-alerts.yaml
groups:
  - name: minio-alerts
    rules:
      - alert: MinioDiskOffline
        expr: minio_disk_offline_total > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "MinIO 磁盘离线"
          description: "Tenant {{ $labels.tenant }} 有 {{ $value }} 个磁盘离线"

      - alert: MinioStorageUsageHigh
        expr: |
          (minio_cluster_capacity_usable_total_bytes
           - minio_cluster_capacity_usable_free_bytes)
          / minio_cluster_capacity_usable_total_bytes * 100 > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "MinIO 存储使用率 > 85%"

      - alert: MinioHealActivity
        expr: minio_heal_objects_error_count_total > 0
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "MinIO 自愈产生错误"
```

## 6. 备份与恢复

### 备份 Tenant 声明

```bash
# 导出 Tenant YAML（含凭证）
kubectl get tenant my-minio-tenant -n minio-tenant -o yaml > tenant-backup.yaml
kubectl get secret my-minio-tenant-env-configuration \
  -n minio-tenant -o yaml >> tenant-backup.yaml

# 备份 PersistentVolumeClaim 声明
kubectl get pvc -n minio-tenant -o yaml > pvc-backup.yaml
```

### 迁移到新集群

```bash
# 1. 备份 PVC 中的数据（使用 mc 或 velero）
velero backup create minio-backup --include-namespaces minio-tenant

# 2. 在新集群安装 Operator
# 3. 创建相同配置的 Tenant
kubectl apply -f tenant-backup.yaml

# 4. 恢复数据
velero restore create --from-backup minio-backup

# 5. 验证数据完整性
kubectl exec -it my-minio-tenant-pool-0-0 -n minio-tenant -- mc ls local/
```

## 7. 常见问题

### Operator 升级

```bash
# 升级 Operator
kubectl apply -k github.com/minio/operator

# 注意：Operator 升级不影响已有 Tenant
# 建议先升级 Operator，再滚动更新 Tenant 镜像版本
```

### Tenant 版本升级

```bash
# 修改 Tenant spec.image 为新版本
kubectl edit tenant my-minio-tenant -n minio-tenant

# 或直接 apply 新配置
kubectl apply -f tenant-upgrade.yaml

# MinIO Operator 会自动执行滚动更新
# 每个 Pod 逐个升级，确保服务不中断
kubectl rollout status statefulset my-minio-tenant-pool-0 -n minio-tenant
```

### 常见坑点

1. **PV 数量必须 ≥ 4**：MinIO 分布式模式至少需要 4 块盘（erasure coding）
2. **Pool 不可缩减**：MinIO 不支持移除 Pool，添加前确认容量规划
3. **存储类选择**：MinIO 推荐使用本地存储或低延迟网络存储，避免 NFS
4. **Operator 命名空间隔离**：Operator 和管理面放在 `minio-operator` 命名空间，Tenant 放在独立命名空间
5. **TLS 证书轮换**：Operator 默认使用 Let's Encrypt 自动签发证书，需确保 DNS 解析正确
6. **大规模集群**：单 Tenant 超过 32 个 Pod 时建议拆分为多个 Tenant

## 参考链接

- [MinIO Operator 官方文档](https://min.io/docs/minio/kubernetes/upstream/index.html)
- [MinIO Operator GitHub](https://github.com/minio/operator)
- [MinIO Tenant CRD 参考](https://min.io/docs/minio/kubernetes/upstream/operations/deploy-manage-minio-operator.html)
