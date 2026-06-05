---
name: redis-86-critical-fixes
description: Redis 8.6.4 / 8.4.4 高危 Bug 修复：AArch64 启动失败、集群崩溃、MULTI 内存泄漏、XREADGROUP 副本不一致
tags: [redis, bugfix, critical, production, aarch64, cluster, sentinel]
---

## 概述

Redis 8.6.4 / 8.4.4（2026-06-04 发布）是面向 8.6.x 和 8.4.x 系列的高危维护版本。**更新紧迫度：HIGH**（可能影响部分用户的生产环境）。

> 8.2.7 也同步发布，修复了相同的高危问题。**所有 8.6.x / 8.4.x / 8.2.x 用户应尽快升级。**

## 一、关键 Bug 修复（8.6.4）

### 1. AArch64（ARM64）启动失败 ⚠️

```text
Issue: #15175, RediSearch#9262
影响：Redis 在 ARM64 架构机器上完全无法启动
```

**根因**：特定 ARM64 CPU 指令兼容性问题导致 Redis 进程在初始化阶段崩溃。

**受影响的部署**：
- AWS Graviton、Ampere Altra 等 ARM 实例
- Apple Silicon（M1/M2/M3/M4）Mac 上的 Docker Redis
- 树莓派等 ARM 嵌入式设备

### 2. 集群崩溃：CLIENT KILL + SSUBSCRIBE + EXEC

```text
Issue: #15094
影响：集群模式下，在 EXEC 内部对 SSUBSCRIBE 客户端执行 CLIENT KILL 导致整个集群崩溃
```

**触发条件**：
```bash
# 在 Lua 脚本或 MULTI/EXEC 事务中执行
MULTI
SSUBSCRIBE channel
CLIENT KILL <client-id>
EXEC  # 💥 集群崩溃
```

**解决方案**：升级到 8.6.4+，修复了在事务/脚本中对订阅客户端执行 KILL 的竞争条件。

### 3. MULTI 队列内存泄漏

```text
Issue: #15163
影响：MULTI 队列的内存计数不准确，导致内存限制判断失误
```

**根因**：`MULTI` 命令队列在跟踪内存使用时，未正确计算部分命令的内存开销，可能导致：
- 实际内存使用超出 `maxmemory` 限制
- 触发错误的 OOM（内存不足）判断

### 4. XREADGROUP 消费者复制不一致

```text
Issue: #14963
影响：Stream 消费者组的消费者在副本（Replica）上的状态与主节点不一致
```

**触发场景**：频繁的消费者组创建/删除 + XREADGROUP 读取时，副本上的消费者信息可能滞后或遗漏。

**影响范围**：使用 Redis Stream 做消息队列且依赖读写分离（主从复制）的场景。

### 5. TCP 死锁/Stall

```text
Issue: #14667, #14886
影响：罕见的网络层死锁导致连接完全卡住，无响应
```

在特定网络条件下（高并发 + 大 payload），TCP 发送缓冲区处理逻辑进入死锁状态，导致：
- 客户端连接一直挂起
- 连接不超时也不断开
- 无法处理新请求

### 6. Sentinel 配置注入

```text
Issue: #14970
影响：通过 SENTINEL SET 命令注入恶意配置
```

Sentinel 模式下，如果攻击者能访问 Sentinel 端口，可通过 `SENTINEL SET` 注入恶意配置项。

**缓解措施**：
```bash
# 除了升级外，确保 Sentinel 端口受防火墙保护
# Sentinel 默认端口 26379 不应暴露到公网
# 启用 ACL 限制 SENTINEL SET 命令权限
```

### 7. HEXPIRE 字段计数溢出

```text
Issue: #15021
影响：HEXPIRE 处理大量字段时整数溢出
```

当对一个包含大量字段（>2^31）的 Hash 执行 `HEXPIRE` 时，内部计数器溢出，可能导致：
- 错误的过期行为
- Redis 进程崩溃

## 二、RediSearch 重要修复

### 1. JSON + 向量索引 + Active-Active 分片崩溃

```text
Issue: RediSearch#9484
影响：在 Active-Active（CRDT）数据库上，后台扫描包含向量字段的 JSON 文档时，分片崩溃
```

**触发条件**：
- 使用 RediSearch 模块
- 启用 Active-Active（CRDT）复制
- 索引包含 JSON 文档中的向量字段
- 后台索引扫描任务触发

### 2. 大索引 EXPIRE/EXPIREAT 并发导致严重延迟

```text
Issue: RediSearch#9635
影响：当 EXPIRE 或 EXPIREAT 操作与对大索引的查询并发执行时，出现严重延迟尖峰和分片无响应
```

**表现**：
- 延迟从毫秒级飙升到数秒
- 分片响应超时
- 集群客户端触发熔断

**解决方案**：升级到 8.6.4+，优化了过期操作与索引查询的锁竞争。

## 三、8.6.4 完整 Bug 清单

| Issue | 描述 | 严重程度 |
|-------|------|---------|
| #15175 | AArch64 启动失败 | 🔴 致命 |
| #15163 | MULTI 队列内存错误计数 | 🟡 高 |
| #15094 | CLIENT KILL + SSUBSCRIBE + EXEC 集群崩溃 | 🔴 致命 |
| #14963 | XREADGROUP 消费者复制不一致 | 🟡 高 |
| #14667/14886 | TCP 死锁/Stall | 🟡 高 |
| #14970 | Sentinel 配置注入 | 🟡 高 |
| #15115 | Lua 调试器内存拷贝不足 | 🟠 中 |
| #14934 | 客户端输出缓冲区内存跟踪 | 🟠 中 |
| #14982 | SCAN COUNT 整数溢出 | 🟠 中 |
| #15073 | CLIENT TRACKING 循环索引 | 🟠 中 |
| #15059 | Use-after-free 漏洞 | 🔴 致命 |
| #15037 | XINFO STREAM 槽位内存错误 | 🟠 中 |
| #15034/15081 | 损坏 RDB 数据处理 | 🟡 高 |
| #15021 | HEXPIRE 字段计数溢出 | 🟠 中 |
| #15188 | cluster-announce-ip 拒绝主机名回归 | 🟡 高 |
| RediSearch#9484 | JSON+向量+CRDT 分片崩溃 | 🔴 致命 |
| RediSearch#9635 | EXPIRE+大索引查询延迟 | 🟡 高 |

## 四、升级建议

### 版本选择

| 当前版本 | 目标版本 | 升级风险 |
|---------|---------|---------|
| 8.6.x | **8.6.4** | 低（补丁版本，无 Breaking Change） |
| 8.4.x | **8.4.4** | 低（补丁版本） |
| 8.2.x | **8.2.7** | 低（补丁版本） |
| 7.4.x | 8.2.7+ 或保持 7.4（不受影响） | 需考虑大版本升级 |

### 升级步骤

```bash
# 1. 下载新版本
wget https://github.com/redis/redis/archive/refs/tags/8.6.4.tar.gz

# 2. 编译安装
tar xzf 8.6.4.tar.gz
cd redis-8.6.4
make -j$(nproc)
make install

# 3. 滚动重启（集群模式）
redis-cli -p 6379 DEBUG SLEEP 1
systemctl restart redis-server

# 4. 确认版本
redis-cli INFO SERVER | grep redis_version
```

### Docker 升级

```bash
docker pull redis:8.6.4
# 或者使用 Alpine 版本
docker pull redis:8.6.4-alpine

# 滚动更新 Swarm/Compose 服务
docker service update --image redis:8.6.4 my-redis-service
```

## 注意事项

| 要点 | 说明 |
|------|------|
| **AArch64 用户优先升级** | 8.6.4 修复了 ARM64 启动失败问题，所有 ARM 部署应优先升级 |
| **Stream 用户注意** | 使用 Redis Stream + 主从复制的场景，XREADGROUP 不一致问题影响较大 |
| **RediSearch 用户** | Active-Active + JSON + 向量索引的组合使用场景务必升级 |
| **集群模式** | 集群环境滚动升级即可，注意先升级 Slave 再切换 Master |
| **8.2.x LTS 用户** | 8.2.7 同步修复了相同的高危问题，LTS 用户可继续留在 8.2 线 |
