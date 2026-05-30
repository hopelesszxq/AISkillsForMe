---
name: minio-monitoring
description: MinIO 监控与告警：Prometheus 指标、Grafana 面板、Bucket 通知与审计日志
tags: [minio, monitoring, prometheus, grafana, alerting, observe]
---

## 概述

MinIO 提供原生 Prometheus 指标暴露端点，支持 bucket 通知（Webhook/AMQP/Kafka）、审计日志和健康检查。生产环境建议将监控与告警作为部署标配。

---

## 一、Prometheus 指标

### 1.1 启用指标端点

MinIO 默认在 `:9000` 暴露公共指标，`:9001` 暴露控制台指标。

```bash
# 方式一：环境变量
export MINIO_PROMETHEUS_AUTH_TYPE="public"
minio server /data

# 方式二：启动参数
minio server /data --address ":9000" --console-address ":9001"
```

```bash
# 访问指标
curl http://localhost:9000/minio/v2/metrics/cluster     # 集群级别的指标
curl http://localhost:9000/minio/v2/metrics/node         # 节点级别的指标
curl http://localhost:9000/minio/v2/metrics/bucket       # Bucket 级别的指标
curl http://localhost:9000/minio/v2/metrics/cluster?labels=*  # 带 Label 维度
```

### 1.2 认证保护

```bash
# 设置 Bearer Token 认证
export MINIO_PROMETHEUS_AUTH_TYPE="token"
export MINIO_PROMETHEAS_BEARER_TOKEN="your-secure-token"

# 访问
curl -H "Authorization: Bearer your-secure-token" \
    http://localhost:9000/minio/v2/metrics/cluster
```

### 1.3 Prometheus 配置

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'minio-cluster'
    metrics_path: /minio/v2/metrics/cluster
    scheme: http
    static_configs:
      - targets: ['minio-1:9000', 'minio-2:9000', 'minio-3:9000', 'minio-4:9000']

  - job_name: 'minio-bucket'
    metrics_path: /minio/v2/metrics/bucket
    scheme: http
    static_configs:
      - targets: ['minio-1:9000']
```

---

## 二、关键指标说明

### 2.1 存储指标

| 指标名 | 类型 | 说明 | 告警阈值 |
|--------|------|------|----------|
| `minio_cluster_capacity_total_bytes` | Gauge | 集群总容量 | - |
| `minio_cluster_capacity_used_bytes` | Gauge | 已用容量 | > 80% |
| `minio_cluster_capacity_usable_total_bytes` | Gauge | 可用总容量 | - |
| `minio_cluster_disk_total_bytes` | Gauge | 磁盘总容量 | - |
| `minio_cluster_disk_used_bytes` | Gauge | 磁盘已用容量 | > 85% |
| `minio_cluster_disk_free_bytes` | Gauge | 磁盘可用容量 | < 20% |
| `minio_cluster_bucket_total` | Gauge | Bucket 总数 | - |
| `minio_cluster_objects_total` | Gauge | 对象总数 | - |
| `minio_cluster_usage_total_bytes` | Gauge | 对象总使用量 | - |

### 2.2 性能指标

| 指标名 | 类型 | 说明 | 关注点 |
|--------|------|------|--------|
| `minio_cluster_traffic_received_bytes` | Counter | 接收流量 | - |
| `minio_cluster_traffic_sent_bytes` | Counter | 发送流量 | - |
| `minio_cluster_s3_requests_total` | Counter | S3 请求总数 | - |
| `minio_cluster_s3_errors_total` | Counter | S3 请求错误数 | > 1% |
| `minio_cluster_s3_requests_4xx_errors` | Counter | 4xx 错误 | - |
| `minio_cluster_s3_requests_5xx_errors` | Counter | 5xx 错误 | > 0 |
| `minio_cluster_heal_drive_total` | Counter | 磁盘修复事件 | > 0 |
| `minio_cluster_heal_disk_total` | Counter | 磁盘修复次数 | > 0 |

### 2.3 内部指标

| 指标名 | 说明 |
|--------|------|
| `minio_cluster_drive_offline_total` | 离线磁盘数（应始终为 0） |
| `minio_cluster_heal_objects_total` | 修复中的对象数 |
| `minio_cluster_heal_objects_failed_total` | 修复失败的对象数 |
| `minio_cluster_nodes_online_total` | 在线节点数 |
| `minio_cluster_nodes_offline_total` | 离线节点数（应为 0） |
| `minio_cluster_audit_failed_messages` | 审计日志发送失败数 |

---

## 三、Grafana 面板

### 3.1 官方面板

MinIO 官方提供 Grafana Dashboard：

```bash
# ID: 15406 — MinIO Cluster Dashboard（推荐）
# 导入地址：https://grafana.com/grafana/dashboards/15406
```

### 3.2 告警规则

```yaml
# prometheus-alerts.yml
groups:
  - name: minio
    rules:
      # 磁盘容量告警
      - alert: MinioDiskUsageHigh
        expr: |
          (minio_cluster_disk_used_bytes / minio_cluster_disk_total_bytes) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MinIO 磁盘使用率超过 85%"
          description: "节点 {{ $labels.instance }} 磁盘使用率 {{ $value }}%"

      - alert: MinioDiskUsageCritical
        expr: |
          (minio_cluster_disk_used_bytes / minio_cluster_disk_total_bytes) * 100 > 95
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MinIO 磁盘使用率超过 95%"

      # 离线磁盘告警
      - alert: MinioDriveOffline
        expr: minio_cluster_drive_offline_total > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MinIO 磁盘离线"
          description: "有 {{ $value }} 块磁盘离线"

      # 离线节点告警
      - alert: MinioNodeOffline
        expr: minio_cluster_nodes_offline_total > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MinIO 节点离线"
          description: "有 {{ $value }} 个节点离线"

      # 5xx 错误告警
      - alert: Minio5xxErrors
        expr: |
          rate(minio_cluster_s3_requests_5xx_errors[5m]) > 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "MinIO 5xx 错误"
          description: "5xx 错误率为 {{ $value }} req/s"

      # 修复失败告警
      - alert: MinioHealFailed
        expr: minio_cluster_heal_objects_failed_total > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MinIO 修复异常"
          description: "有 {{ $value }} 个对象修复失败"
```

---

## 四、Bucket 通知

MinIO 支持将 Bucket 事件发送到外部系统。

### 4.1 支持的 Target

| 类型 | 配置前缀 | 用途 |
|------|----------|------|
| Webhook | `WEBHOOK` | 通用 HTTP 回调 |
| AMQP | `AMQP` | RabbitMQ |
| Kafka | `KAFKA` | Apache Kafka |
| MySQL | `MYSQL` | 数据库持久化 |
| PostgreSQL | `POSTGRESQL` | 数据库持久化 |
| Elasticsearch | `ELASTICSEARCH` | 全文检索 |
| Redis | `REDIS` | 缓存/队列 |
| MQTT | `MQTT` | IoT 场景 |
| NSQ | `NSQ` | 分布式消息 |

### 4.2 Kafka 通知配置

```bash
# 环境变量方式（所有节点一致）
export MINIO_KAFKA_BROKERS=kafka-1:9092,kafka-2:9092
export MINIO_KAFKA_TOPIC=minio-events
export MINIO_KAFKA_TLS_ENABLE=off
export MINIO_KAFKA_SASL_ENABLE=off

# mc 命令配置
mc admin config set myminio notify_kafka:main \
  brokers="kafka-1:9092,kafka-2:9092" \
  topic="minio-events" \
  queue_limit="1000"

# 重启 MinIO 使配置生效
mc admin config set myminio notify_kafka:main
mc admin service restart myminio

# 配置 Bucket 事件监听
mc event add myminio/my-bucket arn:minio:sqs::main:kafka \
  --event put,delete,get
```

### 4.3 通知事件类型

| 事件 | 触发条件 |
|------|----------|
| `put` | 对象创建/上传 |
| `delete` | 对象删除 |
| `get` | 对象读取 |
| `post` | POST 请求 |
| `put_multipart` | 分片上传完成 |
| `delete_marker` | 版本控制下的删除标记创建 |

---

## 五、健康检查与审计

### 5.1 存活检测

```bash
# MinIO 集群健康检查端点
curl -s http://localhost:9000/minio/health/live     # 存活检测
curl -s http://localhost:9000/minio/health/ready     # 就绪检测（集群模式）

# 返回格式
{"status": "ready"}  # 健康
# 非 200 状态码表示异常
```

### 5.2 Kubernetes 探针配置

```yaml
# deployment.yaml
livenessProbe:
  httpGet:
    path: /minio/health/live
    port: 9000
  initialDelaySeconds: 10
  periodSeconds: 15

readinessProbe:
  httpGet:
    path: /minio/health/ready
    port: 9000
  initialDelaySeconds: 5
  periodSeconds: 10
```

### 5.3 审计日志

```bash
# 启用审计 Webhook
export MINIO_AUDIT_WEBHOOK_ENABLE=main
export MINIO_AUDIT_WEBHOOK_ENDPOINT=http://log-collector:8080/minio-audit

# 审计日志字段示例
# {
#   "version": "1",
#   "time": "2026-05-30T10:00:00Z",
#   "deploymentid": "abc123",
#   "request": {
#     "bucket": "my-bucket",
#     "object": "file.pdf",
#     "accessKey": "minioadmin",
#     "userAgent": "MinIOJavaSDK/9.0.1",
#     "host": "192.168.1.100:9000",
#     "method": "PUT",
#     "path": "/my-bucket/file.pdf",
#     "query": ""
#   }
# }
```

---

## 六、docker-compose 监控部署

```yaml
version: "3.9"
services:
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_PROMETHEUS_AUTH_TYPE: "public"
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - minio_data:/data
    ports:
      - "9000:9000"
      - "9001:9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 5s
      retries: 3

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  minio_data:
  grafana_data:
```

## 注意事项

1. **指标延迟**：MinIO 指标刷新周期约 10-15 秒，不适合秒级实时告警
2. **指标端点认证**：生产环境一定要设置 `MINIO_PROMETHEUS_AUTH_TYPE=token`，避免指标端点暴露
3. **分布式指标**：在分布式模式下，`/minio/v2/metrics/cluster` 返回的是聚合后的集群指标（访问任意节点均可）
4. **审计日志性能**：审计日志会记录每次 S3 请求，高并发场景建议使用单独的日志收集管道避免影响主 IO
5. **Bucket 通知可靠性**：通知投递为"至多一次"语义，关键数据变更建议使用数据库双写或 Outbox 模式确保一致性
6. **Grafana 面板版本**：确认 Grafana 版本 >= 9.x 以兼容 MinIO 官方面板 15406 的最新版本
