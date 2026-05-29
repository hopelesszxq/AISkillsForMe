---
name: redis-cluster-sentinel
description: Redis 高可用架构：Sentinel 与 Cluster 拓扑部署、故障切换、集群管理与最佳实践
tags: [redis, cluster, sentinel, ha, high-availability, topology]
---

## 概述

Redis 提供两种高可用方案：**Sentinel**（主从 + 哨兵）和 **Cluster**（分片 + 自动故障转移）。Sentinel 适用于数据量较小的单主场景，Cluster 适用于海量数据的分片场景。本文覆盖两种方案的部署配置、运维监控和实践坑点。

## 方案选型

| 对比项 | Sentinel | Cluster |
|--------|----------|---------|
| 数据分片 | 不支持（单主库） | 支持（16384 个槽位） |
| 最大数据量 | 单机内存（~64GB） | 可水平扩展（PB 级） |
| 故障切换 | Sentinel 选主（~5-30s） | 自动选主（~2-10s） |
| 读写分离 | 支持（配置 slave-read-only） | 支持（从节点只读） |
| 多键操作 | 支持 | 仅限同一个 slot |
| 客户端 | 普通 Redis 客户端 | Cluster 客户端（需支持 MOVED 重定向） |
| 适用场景 | 缓存、小数据量高可用 | 大数据量、海量 QPS |

## 1. Redis Sentinel 部署

### 架构

```
          +-----------+
          |  Client   |
          +-----+-----+
                |
        +-------+-------+
        |               |
  +-----+-----+   +-----+-----+
  | Sentinel 1|   | Sentinel 2|   | Sentinel 3|   (奇数节点，3 或 5)
  +-----------+   +-----------+   +-----------+
        |               |               |
  +-----+-----+   +-----+-----+
  |  Redis M   |   |  Redis S   |   |  Redis S   |
  |  (master)  |←--|  (replica) |   |  (replica) |
  +-----------+   +-----------+   +-----------+
```

### docker-compose 部署

```yaml
version: "3.8"

services:
  redis-master:
    image: redis:7-alpine
    container_name: redis-master
    command: redis-server --requirepass redispass --masterauth redispass
    ports:
      - "6379:6379"
    volumes:
      - redis-master-data:/data
    restart: unless-stopped

  redis-replica-1:
    image: redis:7-alpine
    container_name: redis-replica-1
    command: >
      redis-server --requirepass redispass --masterauth redispass
      --replicaof redis-master 6379
    depends_on:
      - redis-master
    volumes:
      - redis-replica-1-data:/data
    restart: unless-stopped

  redis-replica-2:
    image: redis:7-alpine
    container_name: redis-replica-2
    command: >
      redis-server --requirepass redispass --masterauth redispass
      --replicaof redis-master 6379
    depends_on:
      - redis-master
    volumes:
      - redis-replica-2-data:/data
    restart: unless-stopped

  sentinel-1:
    image: redis:7-alpine
    container_name: sentinel-1
    command: >
      redis-sentinel /etc/sentinel/sentinel.conf
    volumes:
      - ./sentinel-1.conf:/etc/sentinel/sentinel.conf
    depends_on:
      - redis-master
      - redis-replica-1
      - redis-replica-2
    restart: unless-stopped

  sentinel-2:
    image: redis:7-alpine
    container_name: sentinel-2
    command: >
      redis-sentinel /etc/sentinel/sentinel.conf
    volumes:
      - ./sentinel-2.conf:/etc/sentinel/sentinel.conf
    depends_on:
      - redis-master
      - redis-replica-1
      - redis-replica-2
    restart: unless-stopped

  sentinel-3:
    image: redis:7-alpine
    container_name: sentinel-3
    command: >
      redis-sentinel /etc/sentinel/sentinel.conf
    volumes:
      - ./sentinel-3.conf:/etc/sentinel/sentinel.conf
    depends_on:
      - redis-master
      - redis-replica-1
      - redis-replica-2
    restart: unless-stopped

volumes:
  redis-master-data:
  redis-replica-1-data:
  redis-replica-2-data:
```

### Sentinel 配置文件

```conf
# sentinel.conf（每个 Sentinel 节点相同，除 port）
port 26379
daemonize no
pidfile /var/run/redis-sentinel.pid
logfile ""

# 监控 mymaster 集群（2：至少 2 个 Sentinel 同意才切换）
sentinel monitor mymaster redis-master 6379 2
sentinel auth-pass mymaster redispass

# 主观下线时间（30 秒无响应即认为主观下线）
sentinel down-after-milliseconds mymaster 30000

# 故障转移超时（180s）
sentinel failover-timeout mymaster 180000

# 新主库同时最多同步的从库数量（建议 1，避免复制风暴）
sentinel parallel-syncs mymaster 1

# 决议超时
sentinel resolve-hostnames yes
```

### Spring Boot 集成 Sentinel

```yaml
spring:
  data:
    redis:
      sentinel:
        master: mymaster
        nodes:
          - sentinel-1:26379
          - sentinel-2:26379
          - sentinel-3:26379
      password: redispass
      lettuce:
        pool:
          max-active: 16
          max-idle: 8
          min-idle: 4
```

## 2. Redis Cluster 部署

### 架构

```
       +-----------+
       |  Client   |
       |(Cluster   |
       |  Mode)    |
       +-----+-----+
             |
    +--------+--------+
    |        |        |
+---+----+ +---+----+ +---+----+
|Node 1  | |Node 2  | |Node 3  |
|M: 7001 | |M: 7002 | |M: 7003 |
|S: 7004 | |S: 7005 | |S: 7006 |
|slots   | |slots   | |slots   |
|0-5460  | |5461-   | |10923-  |
|        | |10922   | |16383   |
+--------+ +--------+ +--------+
  |    |      |    |      |    |
  |    +------+----+------+    |
  +-----------------------------+
  (Gossip 协议，每个节点全互联)
```

### docker-compose 部署 6 节点 Cluster

```yaml
version: "3.8"

services:
  redis-cluster-1:
    image: redis:7-alpine
    container_name: redis-cluster-1
    command: redis-cluster-entry.sh
    environment:
      - REDIS_PORT=7001
    volumes:
      - ./cluster-entry.sh:/usr/local/bin/redis-cluster-entry.sh
      - cluster-1-data:/data
    network_mode: host

  # ... 重复 2-6，端口 7002-7006

  redis-cluster-init:
    image: redis:7-alpine
    container_name: redis-cluster-init
    depends_on:
      - redis-cluster-1
      - redis-cluster-2
      - redis-cluster-3
      - redis-cluster-4
      - redis-cluster-5
      - redis-cluster-6
    entrypoint: >
      sh -c "
      sleep 10 &&
      redis-cli --cluster create
      127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003
      127.0.0.1:7004 127.0.0.1:7005 127.0.0.1:7006
      --cluster-replicas 1
      --cluster-yes
      "
```

```bash
# cluster-entry.sh
#!/bin/sh
cat > /etc/redis.conf <<EOF
port ${REDIS_PORT:-7001}
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
appendfilename "appendonly.aof"
protected-mode no
bind 0.0.0.0
EOF
redis-server /etc/redis.conf
```

### 手动创建 Cluster

```bash
# 1. 启动 6 个 Redis 实例（3 master + 3 replica）
for port in 7001 7002 7003 7004 7005 7006; do
  mkdir -p /data/redis-cluster/$port
  cat > /data/redis-cluster/$port/redis.conf <<EOF
port $port
cluster-enabled yes
cluster-config-file nodes-$port.conf
cluster-node-timeout 5000
appendonly yes
protected-mode no
daemonize yes
EOF
  redis-server /data/redis-cluster/$port/redis.conf
done

# 2. 创建集群（3 master + 1 副本 per master）
redis-cli --cluster create \
  192.168.1.10:7001 192.168.1.10:7002 192.168.1.10:7003 \
  192.168.1.10:7004 192.168.1.10:7005 192.168.1.10:7006 \
  --cluster-replicas 1

# 3. 验证集群
redis-cli -c -p 7001 cluster info
redis-cli -c -p 7001 cluster nodes
```

### Spring Boot 集成 Cluster

```yaml
spring:
  data:
    redis:
      cluster:
        nodes:
          - 192.168.1.10:7001
          - 192.168.1.10:7002
          - 192.168.1.10:7003
          - 192.168.1.10:7004
          - 192.168.1.10:7005
          - 192.168.1.10:7006
        max-redirects: 3
      password: redispass
      lettuce:
        pool:
          max-active: 32
          max-idle: 16
          min-idle: 8
        cluster:
          refresh:
            period: 5000ms         # 定期刷新集群拓扑
            adaptive: true          # 自适应刷新
```

## 3. Cluster 运维命令

```bash
# 查看集群状态
redis-cli -c -p 7001 cluster info
redis-cli -c -p 7001 cluster nodes

# 检查槽位分配
redis-cli --cluster check 127.0.0.1:7001

# 重新平衡槽位
redis-cli --cluster rebalance 127.0.0.1:7001 \
  --cluster-use-empty-masters

# 添加新节点（先加为 master，再迁移槽位）
redis-cli --cluster add-node 192.168.1.11:7007 192.168.1.10:7001
redis-cli --cluster reshard 192.168.1.10:7001

# 添加副本节点
redis-cli --cluster add-node \
  192.168.1.11:7008 192.168.1.10:7001 \
  --cluster-slave --cluster-master-id <master-node-id>

# 删除节点（先迁走槽位，再删除）
redis-cli --cluster del-node 127.0.0.1:7001 <node-id>

# 手动故障转移
redis-cli -p 7004 cluster failover

# 查看槽位分布
redis-cli -p 7001 cluster slots
```

## 4. Cluster 模式注意事项

### Multi-key 操作限制

```java
// ❌ 报错：CROSSSLOT Keys in request don't hash to the same slot
redisTemplate.opsForValue().multiGet(Arrays.asList("user:1", "order:1"));

// ✅ 使用 Hash Tag 强制将相关 key 分配到同一 slot
redisTemplate.opsForValue().multiGet(Arrays.asList("{user}:1", "{user}:order:1"));

// ✅ 或使用 Redis 8.0 JUMBLE 跨槽命令
// redis-cli> JUMBLE MGET user:1 order:1
```

### Pipeline 和 Lua 脚本

```java
// ❌ Pipeline 中的 key 跨 slot 会导致 MOVED 重定向
// ✅ Pipeline 只操作同一 slot 的 key

// ❌ Lua 脚本中的 key 跨 slot 会报错
// ✅ Lua 脚本中所有 key 使用 Hash Tag 保证在同一 slot
String lua = """
    local key1 = KEYS[1]  -- 使用 {user}:1 方式
    local key2 = KEYS[2]  -- 使用 {user}:2 方式
    return redis.call('MGET', key1, key2)
    """;
```

### 数据迁移与扩容

```bash
# 在线扩容步骤
# 1. 启动新节点
redis-server /data/redis-cluster/7007/redis.conf

# 2. 加入集群
redis-cli --cluster add-node 192.168.1.10:7007 192.168.1.10:7001

# 3. 重新分配槽位（从各节点迁部分槽位到新节点）
redis-cli --cluster reshard 192.168.1.10:7007
# 根据提示输入要迁移的槽位数（如 4096，即 16384/4）
# 输入接收节点 ID（新节点）
# 输入源节点 ID（all 或指定节点）

# 4. 添加副本
redis-cli --cluster add-node \
  192.168.1.10:7008 192.168.1.10:7007 \
  --cluster-slave
```

### 客户端需知

| 特性 | 说明 |
|------|------|
| MOVED 重定向 | 客户端需要处理 MOVED 错误并更新路由缓存 |
| ASK 重定向 | 槽位迁移中的临时重定向 |
| 集群拓扑刷新 | 定期从任意节点获取集群拓扑信息 |
| 最大重试数 | 设置合理的 max-redirects（建议 3-5） |
| 批量操作 | 使用 Hash Tag 保证 key 在同一 slot |
| SCAN 命令 | `SCAN 0 COUNT 100` 返回的 key 可能不在当前节点 |

## 注意事项

1. **Sentinel 集群节点数**：必须是奇数（3 或 5），避免脑裂无法达成共识
2. **Cluster 最小节点**：至少 3 个 master + 3 个 replica（共 6 节点）
3. **网络延迟敏感**：Cluster 各节点之间通过 Gossip 协议通信，网络延迟 > 50ms 可能导致不稳定
4. **Cluster 槽位迁移期间**：部分 key 可能短暂不可用（ASK 重定向），业务层需有重试机制
5. **持久化选择**：Cluster 环境建议 AOF + RDB 双开，RDB 做快照恢复，AOF 保证最小数据丢失
6. **内存碎片**：Cluster 节点较多时内存碎片率高，定期执行 `MEMORY PURGE` 或重启维护
7. **Cluster 下的事务**：MULTI/EXEC 事务中的 key 必须在同一 slot，不可跨节点
8. **备份策略**：Cluster 模式下在每个 slave 节点执行 BGSAVE 做 RDB 备份，避免影响 master
9. **PASSWORD 统一**：Cluster 所有节点必须使用相同的 requirepass 和 masterauth
10. **故障恢复**：Cluster 自动故障转移后，旧 master 恢复时会以 slave 身份加入，不会自动切回
