---
name: valkey-91-features
description: Valkey 9.1 新特性：数据库级 ACL、Lua 模块化、新 I/O 线程模型、HGETDEL/MSETEX/CLUSTERSCAN 命令
tags: [redis, valkey, security, performance, acl, migration]
---

## 概述

Valkey 9.1.0（2026-05-19 发布）是 Redis 分叉项目 Valkey 的首个次要版本。在 Linux 基金会治理下的 80+ 贡献者在安全、可观测性、性能、效率方面做出了重要增强。核心特性：

- **数据库级 ACL**：`db=` 选择器实现按数据库粒度的多租户隔离
- **Lua 脚本引擎模块化**：`libvalkeylua.so` 独立模块，纯缓存集群可卸载
- **新 I/O 线程模型**：单机可达 **2.1M RPS**（512B payload, 9 IO 线程, pipeline depth 10）
- **新命令**：`HGETDEL` / `MSETEX` / `CLUSTERSCAN`
- **JSON 日志**：`log-format json` 直接输出结构化日志
- **TLS 自动重载 + SAN-URI mTLS**：零停机证书轮换

| 日期 | 版本 | 意义 |
|------|------|------|
| 2024.03 | Redis license 变更 → Valkey 分叉 (Redis 7.4) | BSD-3-Clause 保留, Linux Foundation 治理 |
| 2025.10 | Valkey 9.0 GA | Atomic Slot Migration, Hash Field Expiration, 集群编号 DBs |
| 2026.05.19 | **Valkey 9.1.0** | DB-level ACLs, Lua 模块化, 新 I/O 线程 2.1M RPS |

## 数据库级 ACL — 多租户隔离新标准

传统 ACL 的 `~*` 模式匹配对所有数据库生效，无法隔离不同 DB。9.1 新增 `db=` 选择器：

```bash
# 允许 app-user 仅访问 db 0 和 1
> ACL SETUSER app-user on >secretpass +@all ~* db=0,1
OK

> SELECT 0
OK
> SET mykey "hello"
OK

# db 2 被阻止
> SELECT 2
(error) NOPERM No permissions to access database
```

**效果**：原来"一个实例/集群一个租户"的模式可变为"一个集群内按 DB 隔离 + 逐 DB ACL"，大幅减少实例数。结合 9.0 的集群模式编号 DB 效果更佳。

**注意**：编号 DB 是逻辑隔离，不是物理隔离。对强监管数据（PII、支付）仍需实例隔离。

## Lua 脚本引擎模块化 — 缩小攻击面

Lua 引擎被提取为独立模块 `libvalkeylua.so`：

```bash
# 检查已加载的脚本引擎
> INFO scripting_engines
# Scripting Engines
engine_lua:loaded=1,libname=libvalkeylua.so
```

纯缓存负载不需要 Lua 脚本，可直接不加载该模块，将攻击面降到零。

## 新命令详解

### HGETDEL — 原子获取并删除 Hash 字段

适用于队列模式（读取后立即删除），避免 `HGET` + `HDEL` 包装在 `MULTI` 中的开销：

```bash
> HSET job:42 status "pending" payload '{"action":"send_email"}' retries "3"
> HGETDEL job:42 FIELDS 2 status payload
1) "pending"
2) "{\"action\":\"send_email\"}"
> HGETALL job:42
1) "retries"
2) "3"
```

### MSETEX — 批量设置带共享 TTL 的 Key

替代 `SET` + `EXPIRE` 流水线，支持 `NX` 幂等性设置：

```bash
# 设置 3 个 session key，均在 3600s 后过期
> MSETEX 3 session:abc "user:1" session:def "user:2" session:ghi "user:3" EX 3600
OK

# NX: 仅设置不存在的 key
> MSETEX 2 session:abc "user:99" session:xyz "user:4" NX EX 3600
OK
> GET session:abc
"user:1"   # 未被覆盖
```

### CLUSTERSCAN — 集群范围内扫描 Key

替代客户端分别 `SCAN` 每个节点再合并结果：

```bash
> CLUSTERSCAN 0 MATCH "session:*"
> CLUSTERSCAN 0 TYPE hash
> CLUSTERSCAN 0 SLOT 7638
```

## I/O 线程模型 — 单机 2.1M RPS

| 优化项 | 效果 |
|--------|------|
| 新 I/O 线程通信模型 | 吞吐量提升 **17%** |
| 流操作加速 (`XRANGE`/`XREVRANGE`) | 最多 **30%** 更快 |
| String GET 吞吐提升 | 嵌入字符串大小阈值上调，最多 **30%** 更高 |
| 有序集合查询加速 | `ZRANGEBYSCORE`/`ZRANGEBYLEX` 更快 |
| 硬件时钟默认启用 | 整体 GET/SET 改善 **3%** |

## 内存效率

| 优化项 | 目标 | 效果 |
|--------|------|------|
| 字符串内存缩减 | <128B 小字符串 | 最多 **20%** 更少 |
| 有序集合内存缩减 | Skiplist 优化 | 最多 **10%** 更少 |
| 批量删除优化 | `SREM`/`ZREM`/`HDEL` 时暂停重哈希 | 避免不必要的重哈希 |

小字符串节省 20% 对 session token、标志位、短缓存值等场景有实质性的成本优化。

## 可观测性 — JSON 日志

```bash
# valkey.conf
log-format json
```

输出示例（可直接被 Loki/Elastic/CloudWatch 解析）：

```json
{"pid":14500,"role":"primary","timestamp":"14 May 2026 14:13:02.921","level":"notice","message":"oO0OoO0OoO0Oo Valkey is starting oO0oO0OoO0OoO"}
{"pid":14500,"role":"primary","timestamp":"14 May 2026 14:13:02.928","level":"warning","message":"WARNING: The TCP backlog setting of 511 cannot be enforced..."}
```

新增主/IO 线程使用率指标（`INFO` 可查），弥补之前 CPU 接近 100% 但无法区分忙/闲的缺陷。

## 升级决策流程

```
当前存储引擎
├── Redis 7.2 或更早 OSS → 评估直接迁移到 BSD-3 Valkey 9.1
├── Valkey 8.x → 9.1 次版本升级
└── Redis 7.4+ SSPL → 根据许可证政策决定
    └── 是否使用脚本？
        ├── 否 → 不加载 Lua 模块，缩小攻击面
        └── 是 → 保留 Lua 模块
            └── 多租户？
                ├── 是 → 通过编号 DB + 逐 DB ACL 合并实例
                └── 否 → 单 DB 操作
```

## 升级检查清单

1. 盘点所有集群的引擎版本
2. 审计 Lua 脚本使用情况 — 追踪 `EVAL`/`EVALSHA` 调用
3. 开发集群升级到 9.1 → 验证客户端兼容性
4. 设计编号 DB + 逐 DB ACL 映射文档（多租户候选集群）
5. 无脚本集群卸载 Lua 模块
6. 启用 JSON 日志 `log-format json` → 接入日志管道
7. 将主/IO 线程指标暴露到 Prometheus + 设置告警
8. 验证 TLS 自动重载 + 证书过期时间指标
9. 灰度升级（replica → primary），零停机

## 注意事项

- Lua 审计是第一步：贸然卸载会破坏依赖 `EVAL` 的功能
- 编号 DB 是逻辑隔离，监管数据仍需物理隔离
- 硬件时钟默认启用，虚拟化环境需验证单调时钟行为
- 次版本也不可跳过 staging 验证 — 全局行为变更（硬件时钟、新 I/O 线程）
