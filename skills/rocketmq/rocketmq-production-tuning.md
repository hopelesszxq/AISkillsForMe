---
name: rocketmq-production-tuning
description: RocketMQ 生产环境部署与性能调优：JVM 参数、OS 内核、磁盘 RAID、Broker 配置、网络优化与容量规划
tags: [rocketmq, mq, performance, tuning, production, deployment, jvm, linux]
---

## 概述

RocketMQ 生产环境性能取决于 Broker 的 JVM 参数、操作系统内核参数、磁盘 I/O 策略和 Broker 配置的合理调优。本文从底层操作系统到上层 Broker 配置，提供完整的性能调优指南。

## 1. 操作系统层调优

### 内核参数

```bash
# /etc/sysctl.conf — 高性能消息队列内核配置

# 内核调度：deadline/noop 对 SSD 友好
# 查看: cat /sys/block/sdX/queue/scheduler

# 网络优化
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.rmem_default = 16777216
net.core.wmem_default = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_syncookies = 1
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535

# 文件系统
vm.swappiness = 1                # 尽量不使用 swap
vm.max_map_count = 655360        # 映射数限制
fs.file-max = 2097152            # 最大文件句柄
vm.dirty_background_ratio = 5    # 脏页后台回写阈值
vm.dirty_ratio = 15              # 脏页强制回写阈值
```

### 磁盘 RAID 策略

| 场景 | 推荐 RAID | 说明 |
|------|-----------|------|
| 高吞吐（顺序写） | RAID 10 | 读写性能均衡，有冗余 |
| 极致性能（SSD） | 单盘 no RAID / JBOD | 应用层冗余（主从） |
| 成本敏感 | RAID 5（仅限写少读多） | 不推荐写密集型 |

> **关键**：RocketMQ 所有消息先写到 `commitlog`（顺序写），磁盘顺序 I/O 性能至关重要。**建议使用 SSD（NVMe）**，强制开启 `Write Back` 缓存（有 BBU 保护）。

### 文件系统选择

```bash
# XFS 优先（比 ext4 在大文件顺序读写场景好 5-10%）
mkfs.xfs -f -n size=8192 -d agcount=4 /dev/sdb

# 挂载参数
mount -o noatime,nodiratime,nobarrier,inode64 /dev/sdb /data/rocketmq

# 预读优化（SSD 建议设为 0）
blockdev --setra 0 /dev/sdb
```

## 2. JVM 调优

### 推荐 JVM 参数（8C16G Broker）

```bash
JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g -Xmn4g"
JAVA_OPT="${JAVA_OPT} -XX:+UseG1GC -XX:G1HeapRegionSize=16m"
JAVA_OPT="${JAVA_OPT} -XX:G1NewSizePercent=30 -XX:G1MaxNewSizePercent=40"
JAVA_OPT="${JAVA_OPT} -XX:+DisableExplicitGC"
JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow"
JAVA_OPT="${JAVA_OPT} -XX:+AlwaysPreTouch"                    # 启动时预分配内存
JAVA_OPT="${JAVA_OPT} -XX:-UseBiasedLocking"                   # JDK 11+ 默认关闭
JAVA_OPT="${JAVA_OPT} -Dfile.encoding=UTF-8"

# GC 日志（JDK 11+ 统一格式）
JAVA_OPT="${JAVA_OPT} -Xlog:gc*:file=/opt/rocketmq/logs/gc.log:time,level,tags:filecount=5,filesize=100m"
```

### 堆外内存（TransientStorePool）

```properties
# broker.conf
# 启用堆外内存池（显著提升写入性能）
transientStorePoolEnable=true
# 堆外内存池大小（建议 = 5~10 秒的写入量）
# 每 1GB 大约支撑 5~10 万 TPS（视消息大小而定）
maxTransientStorePoolSize=2048    # MB
```

> **原理**：写入时先写到 `DirectByteBuffer`（堆外），由后台线程批量刷盘。减少 GC 停顿对写入路径的影响。

### 内存分配示意图

```
JVM Heap (8G)
├── Young Gen (4G)  ← 消息写入主要在这里分配
│   ├── Eden (60%)  ← 大部分消息对象短暂存在
│   └── Survivor (40%)
└── Old Gen (4G)    ← MappedFile、Consumer 订阅信息
    └── G1 自动管理

堆外 (2G)
└── TransientStorePool (2G)  ← 消息写入缓冲池
```

## 3. Broker 核心配置调优

### 高吞吐配置

```properties
# broker.conf

# === 存储 ===
# 消息存储路径（建议使用独立 SSD 挂载点）
storePathRootDir=/data/rocketmq/store
storePathCommitLog=/data/rocketmq/store/commitlog
# 异步刷盘（性能优先）
flushDiskType=ASYNC_FLUSH
# CommitLog 文件大小（默认 1G，一般不需要改）
mapedFileSizeCommitLog=1073741824
# ConsumeQueue 文件大小
mapedFileSizeConsumeQueue=6000000

# === 线程池 ===
# 发送消息线程数（建议 = CPU 核心数 * 2）
sendMessageThreadPoolNums=16
# 拉取消息线程数
pullMessageThreadPoolNums=16
# 处理回复的线程数
processReplyThreadPoolNums=16

# === 网络 ===
# Netty 参数
serverSocketRcvBufSize=131072
serverSocketSndBufSize=131072
clientSocketRcvBufSize=131072
clientSocketSndBufSize=131072
# 工作线程数（建议 = CPU * 2）
serverWorkerThreads=16
# 异步回调线程数
serverCallbackExecutorThreads=16

# === 消息保留 ===
# 消息保留 72 小时
fileReservedTime=72
# 磁盘空间达到 75% 开始清理
diskMaxUsedSpaceRatio=75
# 清理过期文件间隔（默认 10 秒）
cleanResourceInterval=10000
```

### 高可靠配置（性能 vs 可靠性权衡）

```properties
# 同步刷盘（保证不丢失任何消息，TPS 降 50-70%）
flushDiskType=SYNC_FLUSH

# 同步复制（Master 等待 Slave 写入完毕才返回）
brokerRole=SYNC_MASTER

# 同步双写（最可靠但最慢）
# 结合 SYNC_FLUSH + SYNC_MASTER
```

## 4. 网络与连接优化

### Producer 端

```java
// 批量发送（提升吞吐）
producer.setBatchMaxSize(1024 * 1024);         // 单批最大 1MB
producer.setMaxMessageSize(1024 * 1024 * 4);    // 单条消息最大 4MB
producer.setCompressMsgBodyOverHowmuch(4096);   // 超过 4KB 自动压缩

// 异步发送（主流生产用法）
producer.sendAsync(message, new SendCallback() {
    @Override
    public void onSuccess(SendResult result) {
        // 成功回调
    }
    @Override
    public void onException(Throwable e) {
        // 失败处理 + 重试逻辑
    }
});
```

### Consumer 端

```java
// Push 消费者线程数（建议 20-64，取决于业务处理耗时）
consumer.setConsumeThreadMin(20);
consumer.setConsumeThreadMax(64);
// 批量消费（减少网络往返）
consumer.setConsumeMessageBatchMaxSize(32);
// 拉取批量大小
consumer.setPullBatchSize(32);
// 消费超时（默认 15 分钟，长耗时业务需调大）
consumer.setConsumeTimeout(15);
```

## 5. 容量规划

| 指标 | 估算公式 | 示例 |
|------|----------|------|
| 单机 TPS | 磁盘顺序写带宽 / 平均消息大小 | 500MB/s ÷ 1KB = 500,000 TPS |
| 内存需求 | 堆 8G + 堆外 2G + PageCache | 建议 32G+ 物理内存 |
| 磁盘需求 | 日均消息量 × 保留天数 × (1 + 副本数) | 1TB/天 × 3天 × 2 = 6TB |
| 网络带宽 | TPS × 平均消息大小 × 8 | 10万 × 1KB × 8 = 800Mbps |

> **经验值**：单台 8C32G Broker + SSD 可以稳定支撑 **10~20 万 TPS**（1KB 消息，异步刷盘）。

## 6. 常见性能瓶颈排查

| 现象 | 排查命令 | 常见原因 |
|------|----------|----------|
| 写入延迟高 | `iostat -x 1` | 磁盘 `await > 5ms`、队列 `avgqu-sz > 1` |
| GC 频繁 | `jstat -gcutil <pid> 1s` | Young GC > 5次/秒、Old GC 频发 |
| 网络瓶颈 | `sar -n DEV 1` | 带宽利用率 > 80% |
| 消息积压 | `mqadmin consumerProgress` | 消费能力不足或 Broker IO 瓶颈 |
| PageCache 不足 | `cat /proc/meminfo \| grep -E "Dirty|Writeback"` | 脏页比例过高 |

## 注意事项

- **不要在生产环境使用跨网络（公网）Broker 同步**，延迟抖动会导致 Raft 频繁选举
- `transientStorePoolEnable=true` 时，若进程 Crash 会丢失秒级数据（在同步复制+多副本下可接受）
- **SYNC_FLUSH + SYNC_MASTER** 组合下 TPS 下降明显，优先用多副本保证可靠性而非同步刷盘
- 大消息（>4MB）建议走对象存储（OSS/MinIO），MQ 只传递引用路径
- 监控指标：`jstat -gcutil`、`iostat -x`、`netstat -s` 应纳入生产巡检脚本
