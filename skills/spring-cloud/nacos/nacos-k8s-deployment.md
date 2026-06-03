---
name: nacos-k8s-deployment
description: Nacos 集群在 Kubernetes 上的生产部署最佳实践：StatefulSet 配置、Helm Chart、服务发现、持久化存储与健康检查
tags: [nacos, kubernetes, k8s, helm, statefulset, deployment, infrastructure]
---

## 概述

Nacos 是带状态的中间件（依赖 MySQL + 内部 Raft），在 K8s 上部署需要特殊考虑：**StatefulSet 管理实例标识**、**Headless Service 实现稳定网络标识**、**PVC 持久化日志/数据**。本文覆盖从 Helm 一键部署到手动生产调优的完整方案。

## 一、Helm Chart 部署（推荐）

### 添加仓库并安装

```bash
# 添加 Nacos Helm 仓库
helm repo add nacos https://nacos-group.github.io/nacos-k8s
helm repo update

# 部署 Nacos 集群（3 节点）
helm upgrade --install nacos nacos/nacos \
  --namespace nacos --create-namespace \
  --set mode=cluster \
  --set replicaCount=3 \
  --set mysql.host=my-mysql.nacos.svc.cluster.local \
  --set mysql.db.name=nacos \
  --set mysql.port=3306 \
  --set mysql.user=nacos \
  --set mysql.password=SecureP@ssw0rd \
  --set persistence.enabled=true \
  --set persistence.storageClass=nfs-client \
  --set persistence.accessMode=ReadWriteOnce \
  --set service.type=ClusterIP \
  --set service.port=8848 \
  --set service.grpcPort=9848
```

### 自定义 values.yaml

```yaml
# values-custom.yaml
mode: cluster
replicaCount: 3

# 镜像
image:
  repository: nacos/nacos-server
  tag: v2.5.0
  pullPolicy: IfNotPresent

# MySQL 配置
mysql:
  host: "mysql-0.mysql-headless.nacos.svc.cluster.local"
  port: 3306
  db:
    name: "nacos"
  user: "nacos"
  password: "SecureP@ssw0rd"
  param: "characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useSSL=false"

# 持久化
persistence:
  enabled: true
  storageClass: "nfs-client"
  accessMode: ReadWriteOnce
  size: 10Gi

# 资源配置
resources:
  requests:
    cpu: 500m
    memory: 2Gi
  limits:
    cpu: 2
    memory: 4Gi

# 启动探活
startupProbe:
  enabled: true
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 30

# 服务
service:
  type: ClusterIP
  port: 8848
  grpcPort: 9848
  dnsPolicy: ClusterFirst

# JVM 调优
jvm:
  xms: "2g"
  xmx: "2g"
  xmn: "1g"
  custom: "-Dnacos.security.ssl.enabled=false -Dnacos.functionMode=all"

# 认证
auth:
  enabled: true
  token: "SecretKey012345678901234567890123456789012345678901234567890123456789"
  identityKey: "nacosServer"
  identityValue: "MyNacosServer"
```

```bash
helm upgrade --install nacos nacos/nacos \
  --namespace nacos \
  --values values-custom.yaml
```

## 二、手动 StatefulSet 部署（生产可控）

### 1. 命名空间

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nacos
```

### 2. Headless Service

```yaml
# service-headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: nacos-headless
  namespace: nacos
  labels:
    app: nacos
spec:
  clusterIP: None  # Headless，支持 DNS 解析到每个 Pod IP
  ports:
    - port: 8848
      name: http
    - port: 9848
      name: grpc
    - port: 9849
      name: grpc-cluster
  selector:
    app: nacos
```

### 3. MySQL 初始化 ConfigMap

```yaml
# configmap-nacos-mysql.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nacos-mysql-config
  namespace: nacos
data:
  mysql.host: "mysql-0.mysql-headless.nacos.svc.cluster.local"
  mysql.db.name: "nacos"
  mysql.port: "3306"
  mysql.user: "nacos"
  mysql.db.param: "characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useSSL=false"
```

### 4. StatefulSet

```yaml
# statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nacos
  namespace: nacos
spec:
  serviceName: nacos-headless
  replicas: 3
  podManagementPolicy: Parallel  # 并行启动，加快就绪速度
  selector:
    matchLabels:
      app: nacos
  template:
    metadata:
      labels:
        app: nacos
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: nacos
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: nacos
                topologyKey: kubernetes.io/hostname
      initContainers:
        - name: init-mysql
          image: busybox:latest
          command:
            - sh
            - -c
            - |
              echo "Waiting for MySQL..."
              until nc -z mysql-0.mysql-headless.nacos.svc.cluster.local 3306; do
                sleep 2
              done
              echo "MySQL ready."
      containers:
        - name: nacos
          image: nacos/nacos-server:v2.5.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8848
              name: http
            - containerPort: 9848
              name: grpc
            - containerPort: 9849
              name: grpc-cluster
          env:
            - name: MODE
              value: "cluster"
            - name: NACOS_SERVERS
              value: "nacos-0.nacos-headless.nacos.svc.cluster.local:8848 nacos-1.nacos-headless.nacos.svc.cluster.local:8848 nacos-2.nacos-headless.nacos.svc.cluster.local:8848"
            - name: MYSQL_SERVICE_HOST
              valueFrom:
                configMapKeyRef:
                  name: nacos-mysql-config
                  key: mysql.host
            - name: MYSQL_SERVICE_DB_NAME
              valueFrom:
                configMapKeyRef:
                  name: nacos-mysql-config
                  key: mysql.db.name
            - name: MYSQL_SERVICE_PORT
              valueFrom:
                configMapKeyRef:
                  name: nacos-mysql-config
                  key: mysql.port
            - name: MYSQL_SERVICE_USER
              valueFrom:
                configMapKeyRef:
                  name: nacos-mysql-config
                  key: mysql.user
            - name: MYSQL_SERVICE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: nacos-mysql-secret
                  key: password
            - name: MYSQL_SERVICE_DB_PARAM
              valueFrom:
                configMapKeyRef:
                  name: nacos-mysql-config
                  key: mysql.db.param
            - name: JVM_XMS
              value: "2g"
            - name: JVM_XMX
              value: "2g"
            - name: JVM_XMN
              value: "1g"
            - name: NACOS_AUTH_ENABLE
              value: "true"
            - name: NACOS_AUTH_TOKEN
              valueFrom:
                secretKeyRef:
                  name: nacos-auth-secret
                  key: token
            - name: NACOS_AUTH_IDENTITY_KEY
              value: "nacosServer"
            - name: NACOS_AUTH_IDENTITY_VALUE
              valueFrom:
                secretKeyRef:
                  name: nacos-auth-secret
                  key: identityValue
          readinessProbe:
            httpGet:
              path: /nacos/v1/console/health/readiness
              port: 8848
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
          livenessProbe:
            httpGet:
              path: /nacos/v1/console/health/liveness
              port: 8848
            initialDelaySeconds: 60
            periodSeconds: 15
            timeoutSeconds: 5
          resources:
            requests:
              cpu: 500m
              memory: 2Gi
            limits:
              cpu: 2
              memory: 4Gi
          volumeMounts:
            - name: nacos-data
              mountPath: /home/nacos/data
      terminationGracePeriodSeconds: 60
  volumeClaimTemplates:
    - metadata:
        name: nacos-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "nfs-client"
        resources:
          requests:
            storage: 10Gi
```

### 5. Service（对外暴露）

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nacos-service
  namespace: nacos
spec:
  type: ClusterIP  # 内部访问；外部用 Ingress/LB 暴露 8848
  ports:
    - port: 8848
      targetPort: 8848
      name: http
    - port: 9848
      targetPort: 9848
      name: grpc
  selector:
    app: nacos
```

### 6. Ingress

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nacos-ingress
  namespace: nacos
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: 10m
spec:
  ingressClassName: nginx
  rules:
    - host: nacos.your-company.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nacos-service
                port:
                  number: 8848
```

## 三、2.x 版本 gRPC 端口说明

Nacos 2.x 需要开放额外的 gRPC 端口：

| 端口 | 协议 | 用途 |
|---|---|---|
| 8848 | HTTP | 客户端 API + 控制台 |
| 9848 | gRPC | 客户端 gRPC 请求 |
| 9849 | gRPC | 集群节点间 gRPC 通信 |

K8s Service 必须同时暴露 8848 和 9848：

```yaml
ports:
  - port: 8848
    targetPort: 8848
    name: http
    protocol: TCP
  - port: 9848
    targetPort: 9848
    name: grpc-client
    protocol: TCP
```

## 四、Spring Cloud 客户端配置

```yaml
spring:
  cloud:
    nacos:
      server-addr: nacos-service.nacos.svc.cluster.local:8848
      discovery:
        namespace: dev
        group: DEFAULT_GROUP
      config:
        namespace: dev
        group: DEFAULT_GROUP
        file-extension: yaml
        refresh-enabled: true
```

## 五、生产 Checklist

| 检查项 | 说明 |
|---|---|
| ✅ MySQL 高可用 | 使用 MySQL 主从或 Galera 集群，避免单点 |
| ✅ 资源限制 | `resources.limits` 必须设置，防止 GC 打满宿主机 |
| ✅ Pod 反亲和 | 3 个 Nacos Pod 分布在不同 Node，防止节点宕机导致集群不可用 |
| ✅ PVC 存储类 | 使用 Ceph/NFS/Longhorn 等共享存储，Pod 漂移后数据不丢失 |
| ✅ 就绪探针 | 基于 `/nacos/v1/console/health/readiness` 确保流量不发给未就绪 Pod |
| ✅ 认证开启 | `NACOS_AUTH_ENABLE=true` + 强密码 |
| ✅ 日志收集 | stdout 日志 + 挂载持久卷，配合 EFK/Loki 采集 |
| ✅ 配置备份 | 定期备份 MySQL 数据库（nacos 库） |
| ✅ 网络策略 | 限制 Nacos 可被哪些 Namespace 访问 |

## 注意事项

1. **NACOS_SERVERS 硬编码**：StatefulSet 中 `NACOS_SERVERS` 需列出所有 Pod 的 DNS 名称，扩缩容时需要同步更新
2. **启动顺序**：Nacos 集群启动时不需要严格顺序，但第一个启动的节点会尝试选主，如果没有 MySQL 则进入保护模式
3. **Pod 重启**：Nacos Pod 重启后数据从 MySQL 恢复，本地日志文件丢失不影响运行
4. **存储性能**：MySQL 的磁盘 IOPS 是 Nacos 性能的主要瓶颈，推荐使用 SSD 云盘
5. **版本兼容**：Nacos 2.5.x 需要 JDK 17+，Spring Boot 应用客户端兼容 nacos-client 2.3.x+
6. **升级策略**：采用 RollingUpdate 逐个升级，观察集群健康状态后再更新下一个
