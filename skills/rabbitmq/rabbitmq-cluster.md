---
name: rabbitmq-cluster
description: RabbitMQ 集群部署、高可用配置、网络分区处理与运维最佳实践
tags: [rabbitmq, mq, cluster, high-availability, ops]
---

## 集群架构

RabbitMQ 集群通过 Erlang 分布式协议互联，支持以下模式：

| 模式 | 特点 | 适用场景 |
|------|------|---------|
| **普通集群** | 队列元数据同步，消息数据只存一个节点 | 高吞吐，容忍少量节点故障 |
| **仲裁队列（Quorum）** | 数据多副本 Raft 共识（推荐） | 高可用、数据安全性要求高 |
| **镜像队列（已废弃）** | 主从复制（RabbitMQ 4.0 移除） | 迁移到 Quorum |

## 集群部署（Docker Compose）

```yaml
# docker-compose.yml — 3 节点 RabbitMQ 集群
version: '3.8'

networks:
  rabbitmq-cluster:
    driver: bridge

services:
  rabbitmq-1:
    image: rabbitmq:4.0-management
    hostname: rabbitmq-1
    container_name: rabbitmq-1
    networks:
      - rabbitmq-cluster
    ports:
      - "5672:5672"      # AMQP
      - "15672:15672"    # Management UI
      - "5552:5552"      # Stream 协议
    environment:
      RABBITMQ_ERLANG_COOKIE: "SECRET_COOKIE_VALUE_CHANGE_ME"
      RABBITMQ_NODENAME: "rabbit@rabbitmq-1"
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
      RABBITMQ_CONFIG_FILE: /etc/rabbitmq/rabbitmq
    volumes:
      - rabbitmq-1-data:/var/lib/rabbitmq
      - ./rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro
      - ./definitions.json:/etc/rabbitmq/definitions.json:ro
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 15s
      timeout: 5s
      retries: 5

  rabbitmq-2:
    image: rabbitmq:4.0-management
    hostname: rabbitmq-2
    container_name: rabbitmq-2
    networks:
      - rabbitmq-cluster
    ports:
      - "5673:5672"
      - "15673:15672"
    environment:
      RABBITMQ_ERLANG_COOKIE: "SECRET_COOKIE_VALUE_CHANGE_ME"
      RABBITMQ_NODENAME: "rabbit@rabbitmq-2"
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
      RABBITMQ_CONFIG_FILE: /etc/rabbitmq/rabbitmq
    volumes:
      - rabbitmq-2-data:/var/lib/rabbitmq
      - ./rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro
    depends_on:
      rabbitmq-1:
        condition: service_healthy
    # 加入集群（启动后自动执行）
    command: >
      sh -c "
        rabbitmq-server -detached 2>/dev/null;
        sleep 5;
        rabbitmqctl stop_app;
        rabbitmqctl reset;
        rabbitmqctl join_cluster rabbit@rabbitmq-1;
        rabbitmqctl start_app;
        tail -f /var/log/rabbitmq/rabbit@$${HOSTNAME}.log
      "

  rabbitmq-3:
    image: rabbitmq:4.0-management
    hostname: rabbitmq-3
    container_name: rabbitmq-3
    networks:
      - rabbitmq-cluster
    ports:
      - "5674:5672"
      - "15674:15672"
    environment:
      RABBITMQ_ERLANG_COOKIE: "SECRET_COOKIE_VALUE_CHANGE_ME"
      RABBITMQ_NODENAME: "rabbit@rabbitmq-3"
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
    volumes:
      - rabbitmq-3-data:/var/lib/rabbitmq
      - ./rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro
    depends_on:
      rabbitmq-1:
        condition: service_healthy
    command: >
      sh -c "
        rabbitmq-server -detached 2>/dev/null;
        sleep 8;
        rabbitmqctl stop_app;
        rabbitmqctl reset;
        rabbitmqctl join_cluster rabbit@rabbitmq-1;
        rabbitmqctl start_app;
        tail -f /var/log/rabbitmq/rabbit@$${HOSTNAME}.log
      "

volumes:
  rabbitmq-1-data:
  rabbitmq-2-data:
  rabbitmq-3-data:
```

### RabbitMQ 配置文件

```ini
# rabbitmq.conf — 生产环境推荐配置
# ===== 集群基础 =====
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config
cluster_formation.classic_config.nodes.1 = rabbit@rabbitmq-1
cluster_formation.classic_config.nodes.2 = rabbit@rabbitmq-2
cluster_formation.classic_config.nodes.3 = rabbit@rabbitmq-3

# ===== 默认队列类型 =====
default_queue_type = quorum                 # RabbitMQ 4.0 默认

# ===== 内存与磁盘 =====
vm_memory_high_watermark.relative = 0.7     # 内存使用超 70% 触发流控
vm_memory_high_watermark_paging_ratio = 0.8 # 80% 时开始换页
disk_free_limit.relative = 2.0              # 磁盘剩余 2GB 以下触发流控
disk_free_limit.absolute = 2GB

# ===== 网络 =====
collect_statistics_interval = 10000         # 统计采集间隔 10s

# ===== 心跳 =====
heartbeat = 60                              # TCP 心跳 60s

# ===== 网络分区处理 =====
cluster_partition_handling = autoheol       # 自动修复（推荐）

# ===== Quorum 队列 =====
quorum_cluster_size = 3                     # 仲裁队列副本数
quorum_commands_soft_limit = 64

# ===== Stream =====
stream.max_segment_size_bytes = 536870912   # 512MB
stream.initial_cluster_size = 3

# ===== 管理插件 =====
management.tcp.port = 15672
management.rates_mode = basic

# ===== 日志 =====
log.file.level = info

# ===== 监听器 =====
listeners.tcp.default = 5672
```

## 使用 Ansible 部署集群

```yaml
# ansible/playbooks/rabbitmq-cluster.yml
---
- name: 部署 RabbitMQ 集群
  hosts: rabbitmq_nodes
  become: yes
  vars:
    erlang_cookie: "{{ vault_erlang_cookie }}"
    cluster_nodes: "{{ groups['rabbitmq_nodes'] }}"

  tasks:
    - name: 安装 Erlang + RabbitMQ
      apt:
        name:
          - erlang-base
          - erlang-nox
          - rabbitmq-server
        state: present
        update_cache: yes

    - name: 设置 Erlang Cookie
      copy:
        content: "{{ erlang_cookie }}"
        dest: /var/lib/rabbitmq/.erlang.cookie
        owner: rabbitmq
        group: rabbitmq
        mode: '0400'

    - name: 启动 RabbitMQ
      systemd:
        name: rabbitmq-server
        state: started
        enabled: yes

    - name: 启用管理插件
      command: rabbitmq-plugins enable rabbitmq_management
      args:
        creates: /etc/rabbitmq/enabled_plugins

    - name: 停止应用（非第一个节点）
      command: rabbitmqctl stop_app
      when: inventory_hostname != cluster_nodes[0]

    - name: 重置节点（非第一个节点）
      command: rabbitmqctl reset
      when: inventory_hostname != cluster_nodes[0]

    - name: 加入集群（非第一个节点）
      command: rabbitmqctl join_cluster rabbit@{{ cluster_nodes[0] }}
      when: inventory_hostname != cluster_nodes[0]

    - name: 启动应用（非第一个节点）
      command: rabbitmqctl start_app
      when: inventory_hostname != cluster_nodes[0]

    - name: 设置镜像策略（如果使用经典队列）
      command: >
        rabbitmqctl set_policy ha-all "^ha\." '{"ha-mode":"all","ha-sync-mode":"automatic"}'
      when: inventory_hostname == cluster_nodes[0]
```

## 运维命令

```bash
# ===== 集群管理 =====
# 查看集群状态
rabbitmqctl cluster_status

# 查看节点
rabbitmqctl status

# 加入集群
rabbitmqctl stop_app
rabbitmqctl reset                  # 注意：会清除所有数据！
rabbitmqctl join_cluster rabbit@rabbitmq-1
rabbitmqctl start_app

# 离开集群
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl start_app

# 强制离开集群（节点宕机时在其他节点执行）
rabbitmqctl forget_cluster_node rabbit@rabbitmq-3

# 更改节点类型（disc/ram）
rabbitmqctl change_cluster_node_type disc

# ===== 队列管理 =====
# 查看所有队列
rabbitmqctl list_queues name type messages consumers

# 查看仲裁队列状态
rabbitmqctl list_queues name type quorum_online quorum_ready

# 迁移队列到 Quorum（RabbitMQ 4.0 需要）
rabbitmq-queues quorum_migrate <queue_name>

# ===== 用户管理 =====
rabbitmqctl add_user admin strongPassword!
rabbitmqctl set_user_tags admin administrator
rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"

# ===== 监控 =====
# 检查节点是否健康
rabbitmq-diagnostics check_port_connectivity
rabbitmq-diagnostics check_virtual_hosts
rabbitmq-diagnostics check_if_nodes_are_quorum_critical

# 查看连接
rabbitmqctl list_connections name user state channels

# 查看通道
rabbitmqctl list_channels name connection confirm messages_unacknowledged
```

## 网络分区处理

### 分区检测

```bash
# 查看是否分区
rabbitmqctl cluster_status
# 输出中看到 {partitions,{rabbit@node2,[rabbit@node1]}} 表示分区

# 自动修复策略（推荐）
# rabbitmq.conf 配置：
cluster_partition_handling = autoheol
# autoheol 策略：少数派节点自动重启并重新加入集群
```

### 手动恢复

```bash
# 步骤1：确定哪个节点是多数派
# 在多数派节点上查看状态
rabbitmqctl cluster_status

# 步骤2：在少数派节点上
rabbitmqctl stop_app
rabbitmqctl reset                     # 注意：会丢弃该节点数据
rabbitmqctl start_app                  # 自动重新加入集群

# 步骤3：修复队列（如果数据丢失）
# 副本自动重建（Quorum 队列）或手动同步（经典队列）
```

### 分区预防

```ini
# rabbitmq.conf — 分区预防
# 1. 自动修复
cluster_partition_handling = autoheol

# 2. 网络超时
cluster_keepalive_interval = 10000     # 集群保活间隔 10s

# 3. 节点间通讯
kernel.net.ipv4.tcp_keepalive_time = 300  # Linux 内核 keepalive

# 4. 避免大消息导致 GC 暂停
max_message_size = 134217728           # 最大消息 128MB
```

## 性能调优

```ini
# rabbitmq.conf — 性能调优

# 1. 内存阈值
vm_memory_high_watermark.relative = 0.8   # 8GB 机器可用 6.4GB

# 2. Erlang VM 调优
# 通过 RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS 环境变量
# RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS="+P 1000000 +A 64 +S 4:4"

# 3. TCP 连接
tcp_listen_options.backlog = 4096
tcp_listen_options.nodelay = true

# 4. 消息确认
# 生产者使用 Publisher Confirm，不要使用事务
# 消费者手动 ACK，prefetch=100（根据业务调整）
```

## 监控集成

### Prometheus + Grafana

```yaml
# prometheus.yml scrape config
scrape_configs:
  - job_name: 'rabbitmq'
    static_configs:
      - targets: ['rabbitmq-1:15692', 'rabbitmq-2:15692', 'rabbitmq-3:15692']
```

```bash
# 启用 Prometheus 插件
rabbitmq-plugins enable rabbitmq_prometheus
# 默认端口 15692，Prometheus 格式指标

# 查看指标
curl http://localhost:15692/metrics | grep rabbitmq_queue_messages
```

### 关键监控指标

| 指标 | 告警阈值 | 说明 |
|------|---------|------|
| `rabbitmq_queue_messages_ready` | > 10000 | 未消费消息堆积 |
| `rabbitmq_queue_messages_unacknowledged` | > 5000 | 消费者处理缓慢 |
| `rabbitmq_disk_space_available_bytes` | < 2GB | 磁盘空间不足 |
| `rabbitmq_resident_memory_bytes` | > 70% watermark | 内存压力 |
| `rabbitmq_node_memory_used` | > 1GB | 节点内存使用 |
| 连接数 | > 80% max | 连接耗尽风险 |
| 通道数 | > 80% max | 通道泄漏 |
| 节点状态 | down | 节点宕机 |

## 注意事项

1. **Cookie 必须一致**：所有节点的 `.erlang.cookie` 必须完全相同，否则无法组集群
2. **hostname 解析**：节点间必须通过 hostname 通信，确保 `/etc/hosts` 或 DNS 正确解析
3. **防火墙端口**：需要开放 5672（AMQP）、15672（管理）、4369（EPMD）、25672（集群）、5552（Stream）
4. **Quorum 队列推荐节点数**：3 或 5 节点，奇数个节点保证选举成功
5. **避免 RAM 节点**：RabbitMQ 4.0 已移除 RAM 节点类型，统一使用 disc 节点
6. **升级顺序**：先升级非主节点，逐个滚动升级，最后升级主节点
7. **数据备份**：定期备份 `definitions.json`（通过 Management UI 导出），集群元数据可通过 `rabbitmqadmin export` 导出
8. **不要使用 `reset` 恢复集群**：`reset` 会清除所有数据，只在新节点加入或故障接管场景使用
9. **分区自动修复**：设置 `autoheol` 后，分区自动恢复但可能丢失少量消息，对一致性要求高的场景考虑 `pause_minority` 模式
10. **延迟敏感的 Quorum**：Quorum 队列写入需要多数节点确认，跨机房部署会增加延迟，建议同一机房部署
