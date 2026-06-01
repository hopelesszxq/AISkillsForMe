---
name: rabbitmq-observability
description: RabbitMQ 4.x 可观测性最佳实践：Prometheus 指标采集、Grafana 监控面板、告警规则与健康检查
tags: [rabbitmq, monitoring, prometheus, grafana, observability, alerting]
---

## 概述

RabbitMQ 4.x 提供了丰富的可观测性能力，包括 **Prometheus 指标暴露**（通过 `rabbitmq_prometheus` 插件）、**详细的健康检查端点**、**分布式追踪支持**，以及 **Erlang VM 监控**。本文涵盖从指标采集到告警的全链路实践。

## 1. Prometheus 指标采集

### 启用 Prometheus 插件

```bash
# 启用插件（不需要重启）
rabbitmq-plugins enable rabbitmq_prometheus

# 验证指标端点
curl http://localhost:15692/metrics | head -20
```

RabbitMQ 4.x 默认在 **15692** 端口暴露 Prometheus 指标（4.0+ 新增的专用指标端口，与之前的 15672 管理端口分离）。

### Prometheus 采集配置

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'rabbitmq'
    scrape_interval: 15s
    scrape_timeout: 10s

    # RabbitMQ 4.x 使用专用指标端口 15692
    static_configs:
      - targets:
          - 'rabbitmq-node1:15692'
          - 'rabbitmq-node2:15692'
          - 'rabbitmq-node3:15692'

    # 集群模式下按节点维度聚合
    metric_relabel_configs:
      - source_labels: [rabbitmq_node]
        target_label: node
```

### 4.x 新指标 vs 旧指标

| 指标类别 | 4.x 新指标 | 旧指标 (3.x) |
|---------|-----------|------------|
| 队列深度 | `rabbitmq_quorum_queue_messages_ready` | `rabbitmq_queue_messages_ready` |
| 消费者 | `rabbitmq_queue_consumer_count` | `rabbitmq_queue_consumers` |
| 集群网络 | `rabbitmq_cluster_link_status` | 无独立指标 |
| 磁盘空间 | `rabbitmq_disk_space_available_bytes` | `rabbitmq_disk_free` |
| 内存 | `rabbitmq_memory_used_bytes` | `rabbitmq_memory` |
| Raft 状态 | `rabbitmq_raft_term` | 无 |
| Khepri 状态 | `rabbitmq_khepri_raft_term` | 无 |

## 2. Grafana 监控面板配置

### 核心监控指标

以下 PromQL 查询覆盖 RabbitMQ 4.x 核心监控维度：

```promql
# ========== 系统级 ==========

# Erlang 进程数
erlang_vm_process_count{job="rabbitmq"}

# 内存使用率（%）
rabbitmq_memory_used_bytes{job="rabbitmq"}
  / rabbitmq_memory_limit_bytes{job="rabbitmq"} * 100

# 磁盘剩余空间
rabbitmq_disk_space_available_bytes{job="rabbitmq"}

# FD 使用率
rabbitmq_file_descriptors_used{job="rabbitmq"}
  / rabbitmq_file_descriptors_limit{job="rabbitmq"} * 100

# ========== 消息级 ==========

# 全局消息速率（最近5分钟平均）
rate(rabbitmq_global_messages_published_total{job="rabbitmq"}[5m])
rate(rabbitmq_global_messages_delivered_total{job="rabbitmq"}[5m])

# 每个 vhost 的消息速率
sum by (vhost) (
  rate(rabbitmq_global_messages_published_total{job="rabbitmq"}[5m])
)

# ========== 队列级 ==========

# Ready 消息数（待消费）
rabbitmq_quorum_queue_messages_ready{job="rabbitmq"}

# Unacked 消息数（已投递未确认）
rabbitmq_quorum_queue_messages_unacked{job="rabbitmq"}

# 消费者连接数
rabbitmq_queue_consumer_count{job="rabbitmq"}

# ========== Raft/Khepri ==========

# Raft Leader 变更
changes(rabbitmq_raft_leader{job="rabbitmq"}[15m])

# Khepri Raft Term 变化
changes(rabbitmq_khepri_raft_term{job="rabbitmq"}[5m])
```

### Grafana Dashboard JSON 关键部分

```json
{
  "title": "RabbitMQ 4.x 集群监控",
  "panels": [
    {
      "title": "集群概览",
      "type": "stat",
      "targets": [
        {
          "expr": "count(rabbitmq_build_info{job=\"rabbitmq\"})",
          "legendFormat": "节点数"
        },
        {
          "expr": "rabbitmq_rabbitmq_version{job=\"rabbitmq\"}",
          "legendFormat": "版本"
        }
      ]
    },
    {
      "title": "消息吞吐 (5m)",
      "type": "graph",
      "targets": [
        {
          "expr": "rate(rabbitmq_global_messages_published_total[5m])",
          "legendFormat": "发布"
        },
        {
          "expr": "rate(rabbitmq_global_messages_delivered_total[5m])",
          "legendFormat": "投递"
        },
        {
          "expr": "rate(rabbitmq_global_messages_acked_total[5m])",
          "legendFormat": "确认"
        }
      ]
    },
    {
      "title": "队列深度 Top 10",
      "type": "table",
      "targets": [
        {
          "expr": "topk(10, rabbitmq_quorum_queue_messages_ready)"
        }
      ]
    }
  ]
}
```

## 3. 告警规则

### Prometheus Alerting Rules

```yaml
# rabbitmq_alerts.yml
groups:
  - name: rabbitmq
    interval: 30s
    rules:

      # --- 系统资源告警 ---
      - alert: RabbitMQHighMemoryUsage
        expr: |
          rabbitmq_memory_used_bytes / rabbitmq_memory_limit_bytes * 100 > 80
        for: 2m
        labels: { severity: warning }
        annotations:
          summary: "RabbitMQ 内存使用率超过 80% ({{ $value | humanizePercentage }})"
          description: "节点 {{ $labels.rabbitmq_node }} 内存使用率过高"

      - alert: RabbitMQCriticalMemoryUsage
        expr: |
          rabbitmq_memory_used_bytes / rabbitmq_memory_limit_bytes * 100 > 95
        for: 1m
        labels: { severity: critical }
        annotations:
          summary: "RabbitMQ 内存使用率超过 95%，可能触发流控"

      - alert: RabbitMQLowDiskSpace
        expr: |
          rabbitmq_disk_space_available_bytes < 5e9  # < 5GB
        for: 1m
        labels: { severity: critical }
        annotations:
          summary: "RabbitMQ 磁盘空间不足 (< 5GB)"

      - alert: RabbitMQFileDescriptorsHigh
        expr: |
          rabbitmq_file_descriptors_used / rabbitmq_file_descriptors_limit * 100 > 80
        for: 5m
        labels: { severity: warning }
        annotations:
          summary: "文件描述符使用率超过 80%"

      # --- 队列告警 ---
      - alert: RabbitMQQueueDepthHigh
        expr: |
          rabbitmq_quorum_queue_messages_ready > 10000
        for: 5m
        labels: { severity: warning }
        annotations:
          summary: "队列 {{ $labels.queue }} 深度超过 10000"

      - alert: RabbitMQQueueDepthCritical
        expr: |
          rabbitmq_quorum_queue_messages_ready > 100000
        for: 5m
        labels: { severity: critical }
        annotations:
          summary: "队列 {{ $labels.queue }} 深度超过 10万，消费严重滞后"

      - alert: RabbitMQNoConsumer
        expr: |
          rabbitmq_queue_consumer_count == 0
            and rabbitmq_quorum_queue_messages_ready > 0
        for: 2m
        labels: { severity: critical }
        annotations:
          summary: "队列 {{ $labels.queue }} 有消息但没有消费者"

      # --- 集群告警 ---
      - alert: RabbitMQClusterNodeDown
        expr: |
          up{job="rabbitmq"} == 0
        for: 1m
        labels: { severity: critical }
        annotations:
          summary: "RabbitMQ 节点 {{ $labels.instance }} 离线"

      - alert: RabbitMQClusterPartition
        expr: |
          rabbitmq_cluster_link_status{status="down"} > 0
        for: 30s
        labels: { severity: critical }
        annotations:
          summary: "检测到集群网络分区"

      - alert: RabbitMQLeaderChanges
        expr: |
          changes(rabbitmq_raft_leader[5m]) > 2
        labels: { severity: warning }
        annotations:
          summary: "Raft Leader 频繁变更 (> 2次/5分钟)，可能网络不稳定"
```

## 4. 健康检查端点

### 基础健康检查

RabbitMQ 4.x 提供多个健康检查端点：

```bash
# 1. 节点健康（返回简单 ALIVE/DEAD）
GET /api/health/checks/node
# 响应: {"status":"ok"}

# 2. 集群健康（检查所有节点连通性）
GET /api/health/checks/cluster
# 响应: {"status":"ok","reason":"集群正常"}

# 3. 自定义健康检查（4.x 新增）
GET /api/health/checks/alarms
# 检查是否有内存/磁盘告警触发
```

### Spring Boot Actuator 健康检查

```java
@Component
public class RabbitMQClusterHealthIndicator implements HealthIndicator {

    private final RabbitTemplate rabbitTemplate;

    @Override
    public Health health() {
        try {
            // 发送测试消息到内部队列
            ConnectionFactory cf = rabbitTemplate.getConnectionFactory();
            Connection conn = cf.createConnection();
            boolean isOpen = conn.isOpen();
            conn.close();

            if (isOpen) {
                return Health.up()
                    .withDetail("cluster", rabbitTemplate.execute(
                        channel -> channel.queueDeclarePassive("health.check.queue")))
                    .build();
            }
            return Health.down()
                .withDetail("error", "连接失败")
                .build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,rabbitmq
  health:
    rabbit:
      enabled: true  # 启用 RabbitMQ 健康检查
    readinessstate:
      enabled: true
```

## 5. 分布式追踪

### OpenTelemetry 集成

RabbitMQ 4.x 支持通过插件集成 OpenTelemetry：

```bash
# 启用 OpenTelemetry 插件
rabbitmq-plugins enable rabbitmq_amqp_client_otel

# 配置 OTLP 导出端点
# rabbitmq.conf
otel.traces.exporter = otlp
otel.exporter.otlp.endpoint = http://otel-collector:4318
otel.service.name = rabbitmq-cluster
```

### 消息追踪头传递

```java
// Spring Boot 生产者传递 TraceContext
@Autowired
private Tracer tracer;

public void sendWithTrace(Order order) {
    Span span = tracer.nextSpan().name("send-order").start();
    try (Scope scope = tracer.withSpan(span)) {
        rabbitTemplate.convertAndSend("order.exchange", "order.created",
            order, message -> {
                // 注入 trace header
                message.getMessageProperties().setHeader("trace_id",
                    span.context().traceIdString());
                message.getMessageProperties().setHeader("span_id",
                    span.context().spanIdString());
                return message;
            });
    } finally {
        span.finish();
    }
}

// 消费者提取 Trace
@RabbitListener(queues = "order.queue")
public void handleOrder(Order order, Message message) {
    String traceId = message.getMessageProperties()
        .getHeader("trace_id");
    String spanId = message.getMessageProperties()
        .getHeader("span_id");
    // 恢复 TraceContext 进行后续追踪
}
```

## 6. 运维命令脚本

```bash
#!/bin/bash
# rabbitmq_health_check.sh

NODES=("rabbitmq-node1:15672" "rabbitmq-node2:15672" "rabbitmq-node3:15672")

for node in "${NODES[@]}"; do
    echo "=== 检查节点: $node ==="

    # 1. Prometheus 指标可达性
    curl -sf "http://${node/15672/15692}/metrics" > /dev/null \
        && echo "  ✅ Metrics 端点正常" \
        || echo "  ❌ Metrics 端点异常"

    # 2. API 健康检查
    HEALTH=$(curl -sf -u guest:guest \
        "http://$node/api/health/checks/node" 2>/dev/null)
    echo "  健康状态: $HEALTH"

    # 3. 关键指标快照
    echo "  队列深度:"
    curl -sf -u guest:guest \
        "http://$node/api/queues" 2>/dev/null \
        | jq -r '.[] | "    \(.name): ready=\(.messages_ready) unacked=\(.messages_unacknowledged)"' \
        | sort -t '=' -k2 -rn | head -10
done
```

## 注意事项

1. **指标端口区分**：RabbitMQ 4.x 的 Prometheus 指标端口为 15692（独立端口），不要从 15672（管理端口）抓取指标，避免管理插件负载过重
2. **Khepri 监控**：4.x 集群务必监控 Khepri Raft Term 的变更频率，频繁变更表明集群网络不稳定
3. **告警风暴防护**：RabbitMQ 集群节点同时离线会产生大量告警，建议设置 `for: 2m` 等持续时间条件
4. **指标粒度**：
   - `rabbitmq_quorum_queue_messages_ready`：队列级别，按 queue + vhost 标签区分
   - `rabbitmq_global_messages_published_total`：全局级别，按 vhost 区分
   - 需要按具体队列告警时，使用队列级指标并配置聚合规则
5. **历史数据保留**：RabbitMQ 的 Prometheus 指标建议保留 7-30 天，旧数据可归档用于容量规划
6. **Grafana 变量**：在面板中使用 Grafana 模板变量（vhost、node、queue）实现动态切换
7. **JMX 监控（替代方案）**：如果已使用 JMX 监控栈，可通过 `rabbitmq_jmx_metrics` 插件暴露 JMX 指标
8. **4.x 新指标命名**：4.3+ 中一些指标前缀从 `rabbitmq_` 改为 `rabbitmq_quorum_queue_` / `rabbitmq_stream_queue_`，更新旧告警规则时注意

### 推荐 Grafana Dashboard ID

- **RabbitMQ 4.x 官方面板**：RabbitMQ 官方仓库提供开箱即用的 Grafana 面板模板
- **社区面板**: `rabbitmq-prometheus` Grafana Dashboard ID: 10991（需适配 4.x 指标）
- 建议根据实际集群规模自定义面板，避免加载过多队列级指标导致 Grafana 卡顿
