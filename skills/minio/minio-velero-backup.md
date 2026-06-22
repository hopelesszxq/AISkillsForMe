---
name: minio-velero-backup
description: 使用 Velero + MinIO 构建裸金属 Kubernetes 集群备份方案
tags: [minio, velero, kubernetes, backup, disaster-recovery]
---

## 概述

在裸金属 Kubernetes 环境中，没有托管的 S3 服务可用。使用 **Velero + MinIO** 构建自托管备份方案，可以实现完整的集群备份与恢复。核心要点：**MinIO 必须部署在集群外部**，避免循环依赖。

## 架构设计

```
┌─────────────────────────────┐     ┌──────────────────────┐
│   K8s 集群                  │     │   裸金属节点           │
│                             │     │                      │
│  Velero → AWS S3 Plugin ────┼─────┼─→ MinIO (外部存储)   │
│                             │     │   └─ k8s-backups 桶  │
│  Longhorn CSI 快照          │     │                      │
│  ETCD 原始快照 (systemd) ────┼─────┼─→ rsync → .db 文件  │
└─────────────────────────────┘     └──────────────────────┘
```

### 关键原则

1. **MinIO 独立部署**：与 K8s 集群隔离，集群全挂时备份不受影响
2. **三层备份**：Velero（资源+PV）+ Longhorn 快照（CSI）+ ETCD 快照（控制平面）
3. **CSI 快照兼容性**：NFS 等不支持 CSI 快照的卷需单独处理

## 部署步骤

### 1. 部署 MinIO（外部节点）

```bash
# 在独立服务器上启动 MinIO
# 创建专用桶和服务账号
mc mb myminio/k8s-backups
mc admin user add myminio velero-backup <secret-key>
mc admin policy attach myminio readwrite --user velero-backup
```

### 2. 安装 Velero

关键点：使用 AWS S3 插件但覆盖 endpoint 指向本地 MinIO。

```bash
# 准备 credentials 文件（AWS 兼容格式）
cat > credentials-velero <<EOF
[default]
aws_access_key_id=velero-backup
aws_secret_access_key=<secret-key>
EOF

# 安装 Velero
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.14.0 \
  --bucket k8s-backups \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=true \
  --backup-destination-type=s3 \
  --s3-url http://minio.example.com:9000 \
  --namespace velero
```

### 3. 处理不同的存储类型

裸金属环境常见混合存储（Longhorn + NFS + HostPath），Velero 默认对所有 PV 尝试 CSI 快照，NFS 卷会失败导致 `PartiallyFailed`。

#### 方案 A：禁用不兼容卷的快照

```bash
# 创建备份调度时关闭卷快照
velero schedule create daily-cluster-backup \
  --schedule="0 2 * * *" \
  --snapshot-volumes=false \
  --ttl 168h
```

```bash
# 或修改已有 schedule
kubectl patch schedule daily-cluster-backup -n velero \
  --type=merge \
  -p '{"spec":{"template":{"snapshotVolumes":false}}}'
```

#### 方案 B：Restic/Kopia 文件级备份

```yaml
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: default
  namespace: velero
spec:
  provider: aws
  objectStorage:
    bucket: k8s-backups
  config:
    region: us-east-1
    s3Url: http://minio.example.com:9000
---
apiVersion: velero.io/v1
kind: VolumeSnapshotLocation
metadata:
  name: fs-backup
  namespace: velero
spec:
  provider: velero.io/restic
  config:
    # 使用 Restic 进行文件级备份，不依赖 CSI
```

### 4. ETCD 控制平面安全网

Velero 不备份 ETCD 本身，需要独立机制保护控制平面：

```ini
# /etc/systemd/system/etcd-snapshot.service
[Unit]
Description=ETCD Snapshot Backup
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/bin/etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/backups/etcd/etcd-snapshot-$(date +%Y%m%d).db
ExecStartPost=/bin/sh -c '/usr/bin/find /var/backups/etcd -type f -name "etcd-snapshot-*.db" -mtime +7 -exec rm -f {} \;'
```

```ini
# /etc/systemd/system/etcd-snapshot.timer
[Unit]
Description=Daily ETCD Snapshot
[Timer]
OnCalendar=daily
[Install]
WantedBy=timers.target
```

```bash
systemctl enable etcd-snapshot.timer --now
# 使用 cron 将 .db 文件同步到 MinIO
0 3 * * * rsync -avz /var/backups/etcd/ velero@minio-server:/mnt/backups/etcd/
```

## 备份恢复验证

```bash
# 查看备份状态
velero backup get
velero backup describe daily-cluster-backup --details

# 从备份恢复
velero restore create --from-backup daily-cluster-backup \
  --namespace-mappings myapp:myapp-restored

# ETCD 恢复
etcdctl snapshot restore /var/backups/etcd/etcd-snapshot-20260601.db \
  --data-dir /var/lib/etcd-restored
```

## 注意事项

1. **MinIO 不可与集群共存**：备份目标不能是集群内服务，否则集群故障=备份丢失
2. **CSI 快照兼容性**：NFS、HostPath、部分 CSI 驱动不支持快照，需用 Restic/Kopia
3. **Helm 部署时指定 s3Url**：GitOps 场景务必覆盖 `configuration.s3Url` 值
4. **SealedSecrets 管理凭证**：credentials 文件不应提交到 Git，用 SealedSecrets 加密
5. **定期恢复演练**：至少每季度做一次完整恢复测试，验证备份可用性
6. **裸金属网络规划**：Velero → MinIO 之间建议独立网络或 VPN，避免备份流量占用业务带宽
