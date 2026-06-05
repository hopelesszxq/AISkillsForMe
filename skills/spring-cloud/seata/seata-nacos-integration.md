---
name: seata-nacos-integration
description: Seata 与 Nacos 集成实战：注册中心、配置中心、Seata-Server 集群部署、Spring Boot 集成与高可用配置
tags: [seata, nacos, distributed-transaction, spring-cloud, microservice, integration]
---

## 概述

Seata 与 Nacos 的集成是 Spring Cloud Alibaba 微服务体系中最常见的分布式事务方案。Nacos 同时承担 Seata 的**注册中心**和**配置中心**角色，实现 Seata-Server 的高可用集群和动态配置管理。

## 1. 架构总览

```
                    ┌──────────────────────┐
                    │    Nacos Cluster      │
                    │  (注册中心+配置中心)    │
                    └──────┬──────────┬────┘
                           │          │
              ┌────────────┤          ├────────────┐
              ▼            ▼          ▼            ▼
        ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
        │Seata-Svr│  │Seata-Svr│  │Seata-Svr│  │Seata-Svr│
        │Node-1   │  │Node-2   │  │Node-3   │  │Node-4   │
        └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘
             │            │            │            │
             └────────────┴────────────┴────────────┘
                    │ 注册到 Nacos
                    ▼
        ┌──────────────────────┐
        │  Microservice App    │  ← 通过 Nacos 发现 Seata-Server
        │ (TM/RM)              │
        └──────────────────────┘
```

## 2. Seata-Server 使用 Nacos 做注册中心

### 下载与配置

```bash
# 1. 下载 Seata-Server（推荐 2.x 最新版）
wget https://github.com/seata/seata/releases/download/v2.6.0/seata-server-2.6.0.zip
unzip seata-server-2.6.0.zip -d /opt/seata-server

# 2. 修改 registry.conf（使用 Nacos 作为注册中心）
```

### registry.conf

```properties
registry {
  type = "nacos"
  nacos {
    application = "seata-server"
    serverAddr = "192.168.1.100:8848"
    group = "SEATA_GROUP"
    namespace = "public"
    cluster = "default"
    username = "nacos"
    password = "nacos"
  }
}

config {
  type = "nacos"
  nacos {
    serverAddr = "192.168.1.100:8848"
    group = "SEATA_GROUP"
    namespace = "public"
    dataId = "seata-server.properties"
    username = "nacos"
    password = "nacos"
  }
}
```

### seata-server.properties（上传到 Nacos 配置中心）

```properties
# 存储模式（支持 file/db/redis）
store.mode=db

# 数据库存储配置（推荐 MySQL 8.0+）
store.db.datasource=druid
store.db.dbType=mysql
store.db.driverClassName=com.mysql.cj.jdbc.Driver
store.db.url=jdbc:mysql://192.168.1.200:3306/seata?useSSL=false&rewriteBatchedStatements=true&characterEncoding=utf8mb4
store.db.user=seata
store.db.password=seata@2026
store.db.minConn=5
store.db.maxConn=30
store.db.globalTable=global_table
store.db.branchTable=branch_table
store.db.lockTable=lock_table
store.db.distributedLockTable=distributed_lock_table
store.db.queryLimit=100
store.db.maxWait=5000

# 事务日志相关
store.db.logTable=log_table

# 事务会话存储
server.session.enableBranchAsyncRemove=true  # 异步清理分支事务
server.recovery.committingRetryPeriod=1000
server.recovery.asynCommittingRetryPeriod=1000
server.recovery.rollbackingRetryPeriod=1000
server.recovery.timeoutRetryPeriod=1000

# 事务超时
server.maxCommitRetryTimeout=-1
server.maxRollbackRetryTimeout=-1
server.undo.logSaveDays=7

# 线程池
server.service.threadPoolWorkSize=100
server.service.maxBranchRegisterSize=100
```

### 初始化数据库

```sql
-- Seata Server 数据库（存储 global_table, branch_table, lock_table）
CREATE DATABASE IF NOT EXISTS seata DEFAULT CHARSET utf8mb4;

-- 建表脚本：https://github.com/seata/seata/blob/develop/script/server/db/mysql.sql
-- 确保 global_table / branch_table / lock_table / distributed_lock_table 已创建
```

### 启动 Seata-Server

```bash
# 单节点启动
cd /opt/seata-server
sh bin/seata-server.sh -p 8091 -h 192.168.1.10

# 多节点集群（各节点使用不同 IP/端口）
# Node 1: sh bin/seata-server.sh -p 8091 -h 192.168.1.10
# Node 2: sh bin/seata-server.sh -p 8091 -h 192.168.1.11
# Node 3: sh bin/seata-server.sh -p 8091 -h 192.168.1.12
```

### Docker Compose 集群部署

```yaml
# docker-compose-seata-cluster.yml
version: '3.8'

services:
  seata-server-1:
    image: seataio/seata-server:2.6.0
    container_name: seata-server-1
    ports:
      - "8091:8091"
    environment:
      - SEATA_PORT=8091
      - REGISTRY_TYPE=nacos
      - REGISTRY_NACOS_SERVER_ADDR=nacos:8848
      - CONFIG_TYPE=nacos
      - CONFIG_NACOS_SERVER_ADDR=nacos:8848
      - STORE_MODE=db
      - STORE_DB_URL=jdbc:mysql://mysql:3306/seata
      - STORE_DB_USER=seata
      - STORE_DB_PASSWORD=seata@2026
    volumes:
      - ./seata/resources:/seata-server/resources
    depends_on:
      - nacos
      - mysql
    restart: unless-stopped

  seata-server-2:
    image: seataio/seata-server:2.6.0
    container_name: seata-server-2
    ports:
      - "8092:8091"
    environment:
      - SEATA_PORT=8092
      - REGISTRY_TYPE=nacos
      - REGISTRY_NACOS_SERVER_ADDR=nacos:8848
      - CONFIG_TYPE=nacos
      - CONFIG_NACOS_SERVER_ADDR=nacos:8848
      - STORE_MODE=db
      - STORE_DB_URL=jdbc:mysql://mysql:3306/seata
      - STORE_DB_USER=seata
      - STORE_DB_PASSWORD=seata@2026
    depends_on:
      - nacos
      - mysql
    restart: unless-stopped
```

## 3. 微服务客户端集成

### 依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
<dependency>
    <groupId>io.seata</groupId>
    <artifactId>seata-spring-boot-starter</artifactId>
    <version>2.6.0</version>
</dependency>
```

### application.yml

```yaml
seata:
  enabled: true
  application-id: ${spring.application.name}
  tx-service-group: my_tx_group           # 事务分组
  # 事务分组映射关系：my_tx_group → Nacos 上的 cluster（需在 Nacos 中配置）
  service:
    vgroup-mapping:
      my_tx_group: default                # 映射到 Seata-Server 的 cluster 名
    grouplist:                            # 不使用 grouplist，通过 Nacos 自动发现
    #  disable-global-transaction: false
  registry:
    type: nacos
    nacos:
      application: seata-server
      server-addr: ${spring.cloud.nacos.discovery.server-addr}
      group: SEATA_GROUP
      namespace: public
      cluster: default
      username: nacos
      password: nacos
  config:
    type: nacos
    nacos:
      server-addr: ${spring.cloud.nacos.discovery.server-addr}
      group: SEATA_GROUP
      data-id: seata-server.properties
      username: nacos
      password: nacos
```

### 事务分组映射说明

```
客户端配置:
  seata.tx-service-group = my_tx_group
  seata.service.vgroup-mapping.my_tx_group = default

Nacos 配置中心:
  在 Nacos 中配置 my_tx_group 到 Seata 集群的映射:
  Data ID: service.vgroupMapping.my_tx_group
  Group: SEATA_GROUP
  内容: default

  客户端通过 Nacos 获取到映射后，再根据 cluster=default 去发现 Seata-Server 实例
```

### 使用 @GlobalTransactional

```java
@Service
public class OrderService {

    @GlobalTransactional(name = "create-order", rollbackFor = Exception.class)
    public void createOrder(OrderDTO order) {
        // 1. 创建订单（本地事务）
        orderMapper.insert(order);

        // 2. 扣减库存（远程调用）
        inventoryService.deduct(order.getProductId(), order.getQuantity());

        // 3. 扣减余额（远程调用）
        accountService.debit(order.getUserId(), order.getAmount());

        // 4. 如果任意步骤失败，Seata 自动回滚所有已提交的本地事务
    }
}
```

## 4. 配置中心管理（Nacos）

### Nacos 中推荐的配置项

通过 Nacos 配置中心动态管理 Seata 参数，无需重启 Server：

| Data ID | Group | 说明 |
|---------|-------|------|
| `seata-server.properties` | `SEATA_GROUP` | Seata-Server 全局配置 |
| `service.vgroupMapping.my_tx_group` | `SEATA_GROUP` | TX 分组 → 集群映射 |
| `client.tm.degradeCheck` | `SEATA_GROUP` | TM 降级检查开关 |
| `client.rm.reportSuccessEnable` | `SEATA_GROUP` | RM 成功上报开关 |

### 事务分组与集群容灾

```properties
# 方案：主集群 + 备用集群
# 在 Nacos 配置中维护分组映射

# 主集群映射
Data ID: service.vgroupMapping.my_tx_group
Content: default

# 备用集群映射（切换时修改内容）
Data ID: service.vgroupMapping.my_tx_group
Content: backup_cluster

# 客户端自动感知配置变更，切换 Seata-Server 集群
```

## 5. 客户端 Undo_log 表

```sql
-- 每个微服务的业务数据库都要创建 undo_log 表
CREATE TABLE IF NOT EXISTS `undo_log` (
  `branch_id` bigint NOT NULL COMMENT '分支事务 ID',
  `xid` varchar(128) NOT NULL COMMENT '全局事务 ID',
  `context` varchar(128) NOT NULL COMMENT '上下文信息',
  `rollback_info` longblob NOT NULL COMMENT '回滚信息',
  `log_status` int NOT NULL COMMENT '状态：0正常 1已回滚',
  `log_created` datetime NOT NULL COMMENT '创建时间',
  `log_modified` datetime NOT NULL COMMENT '修改时间',
  `ext` varchar(100) DEFAULT NULL COMMENT '扩展字段',
  PRIMARY KEY (`branch_id`),
  KEY `idx_xid` (`xid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Seata AT 模式回滚日志表';
```

> **重要**：每个参与分布式事务的微服务数据库**必须**有 `undo_log` 表，否则 AT 模式无法工作。

## 6. 生产环境注意事项

| 问题 | 解决方案 |
|------|----------|
| Seata-Server 单点故障 | 部署 3+ 节点 Nacos 注册集群 |
| 事务日志磁盘增长 | 设置 `server.undo.logSaveDays=7`，定期清理 `undo_log` |
| 高并发全局锁争用 | 调大 `store.db.maxConn=30`，使用 `SELECT ... FOR UPDATE` 优化 |
| 网络分区导致悬挂 | 开启 `server.session.enableBranchAsyncRemove=true` |
| TC 节点切换事务丢失 | 使用 `store.mode=db`（数据库共享状态）而非 `file` |
| 客户端连接超时 | 设置 `seata.client.rm.connectTimeout=5000` |
| 大事务超时 | `@GlobalTransactional(timeoutMills=600000)` 设置超时 |

## 注意事项

- **Nacos 必须先于 Seata-Server 启动**，否则 Seata 无法注册和拉取配置
- Seata-Server 集群必须使用 **db 存储模式**（`store.mode=db`），`file` 模式不支持多节点共享状态
- 事务分组（`tx-service-group`）的映射关系通过 Nacos 配置中心管理，不允许写死在配置文件中
- 跨服务调用时，Feign 调用链自动传播 `xid`（基于 `SeataRestTemplateInterceptor`），**无需手动传递**
- AT 模式依赖数据库本地锁，**不要在 `@GlobalTransactional` 中嵌套调用自身服务**（导致行锁死等）
