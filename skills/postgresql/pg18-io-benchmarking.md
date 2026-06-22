---
name: pg18-io-benchmarking
description: PostgreSQL 18 I/O 性能基准测试：io_method (sync/worker/io_uring) 对比 PG17 的实际数据
tags: [postgresql, pg18, performance, io, benchmarking, sysbench]
---

## 概述

PostgreSQL 18 引入 `io_method` 配置项，提供三种 I/O 模式：
- **sync** — 传统同步 I/O（PG17 及之前的默认行为）
- **worker**（默认）— 使用专用后台工作进程处理所有 I/O 操作
- **io_uring** — 使用 Linux io_uring 接口实现异步磁盘读取

PlanetScale 团队使用 sysbench `oltp_read_only` 对 PG17 vs PG18 三种模式进行了 96 组基准测试。

## 测试环境

| 实例类型 | vCPU | 内存 | 磁盘类型 | IOPS | 吞吐量 | 月成本 |
|---------|------|------|---------|------|--------|-------|
| r7i.2xlarge | 8 | 64GB | gp3 3K | 3,000 | 125MB/s | $442 |
| r7i.2xlarge | 8 | 64GB | gp3 10K | 10,000 | 500MB/s | $492 |
| r7i.2xlarge | 8 | 64GB | io2 16K | 16,000 | - | $1,514 |
| i7i.2xlarge | 8 | 64GB | NVMe本地盘 | 300,000 | - | $551 |

## 核心结论

### 1. PG18 整体比 PG17 更快

在几乎所有场景下，PG18 的 `sync` 和 `worker` 模式都显著优于 PG17，尤其是在短查询（`--range_size=100`）场景：

```
短查询场景 (range_size=100, 单连接):
  PG17                → 基准线
  PG18 + sync/worker  → 明显更快 (+15~30%)
  PG18 + io_uring     → 与 PG17 持平或略低
```

### 2. io_uring 并非全能冠军

很多人期待 io_uring 带来飞跃，但实际测试表明：

- **网络附加存储 (EBS gp3/io2)** 上，io_uring 表现最差 — 延迟是关键瓶颈
- **本地 NVMe 高并发** 场景下，io_uring 略有优势
- 原因：**索引扫描尚未使用 AIO**，且 io_uring 的 checksum/memcpy 仍可能成为瓶颈

> Workers 模式在单进程视角下提供了更好的 I/O 并行度

### 3. 本地 NVMe 性价比最高

| 配置 | 月成本 | 相对性能 |
|------|--------|---------|
| i7i + NVMe (1.8TB) | **$551** | 最高（~2x gp3） |
| r7i + gp3 3K (700GB) | $442 | 基准 |
| r7i + io2 16K (700GB) | $1,514 | 接近 NVMe 但贵 3x |

### 4. 高并发下 I/O 差异缩小

50 连接并发时，IOPS/吞吐量成为瓶颈，不同 I/O 模式差异变小。这是好现象 — PG18 在大压力下没有退化。

## 配置建议

```ini
# postgresql.conf — 常规场景（推荐默认 worker）
io_method = 'worker'          # PG18 默认值，大多数场景最优

# 高性能本地 NVMe + 大量并发读取
io_method = 'io_uring'        # 高 I/O 并发时略微胜出
effective_io_concurrency = 200

# 网络存储 (EBS/NFS) — 不要用 io_uring
io_method = 'sync'            # 或 'worker'，两者在 EBS 上表现接近
```

## 关键观察

### 为什么 io_uring 没赢？

1. **索引扫描尚未使用 AIO** — PG18 的 io_uring 实现还不覆盖索引 I/O
2. **网络存储延迟** — gp3/io2 的延迟掩盖了 io_uring 的异步优势
3. **Checksum 开销** — 即使 I/O 异步，checksum 验证 + memcpy 仍是串行瓶颈
4. **低并发下无优势** — io_uring 在高 I/O 并发时才显现优势

### 最适合 io_uring 的场景

- 本地 NVMe 存储（低延迟 + 高 IOPS）
- 大量顺序/位图扫描
- 高并发读取（50+ 连接）

## 参考

- 基准测试原文: [Benchmarking Postgres 17 vs 18](https://planetscale.com/blog/benchmarking-postgres-17-vs-18) (PlanetScale, 2025-10-14)
- 测试工具: sysbench oltp_read_only
- 已有相关技能: `pg18-optimizer-monitoring.md`（AIO 配置详解）
