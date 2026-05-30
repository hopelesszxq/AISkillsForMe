---
name: redis-80-security
description: Redis 8.0.x 安全补丁与 CVE 汇总：Lua RCE、HyperLogLog OOB、Bloom 过滤器漏洞
tags: [redis, security, cve, patch, upgrade]
---

## 概述

Redis 8.0.x 系列修复了多个高危安全漏洞。以下汇总了 8.0.3 → 8.0.6 版本的安全补丁和关键 Bug 修复，建议生产环境及时升级。

> 当前最新稳定版：**Redis 8.0.6**（2026-02-23）

---

## 版本升级路径

| 版本 | 发布时间 | 升级紧急度 | 关键修复 |
|------|----------|-----------|----------|
| 8.0.6 | 2026-02-23 | LOW | `\r\n` 注入安全修复 |
| 8.0.5 | 2025-11-02 | **HIGH** | HyperLogLog/Bloom/Cuckoo 崩溃修复 |
| 8.0.4 | 2025-10-03 | **SECURITY** | 4 个 Lua 相关 CVE |
| 8.0.3 | 2025-07-06 | **SECURITY** | HyperLogLog OOB Write |
| 8.0.2-8.0.0 | 2025-05-14 | - | 初始发布 |

**建议**：如果当前使用的是 8.0.0-8.0.3，**立即升级到 8.0.6**。

---

## CVE 详情

### 8.0.4 修复的 Lua 相关 CVE

```bash
# CVE-2025-49844 — Lua 脚本导致远程代码执行
# 影响：攻击者可通过精心构造的 Lua 脚本在 Redis 服务器上执行任意代码
# 评分：CRITICAL
# 修复版本：>= 8.0.4

# CVE-2025-46817 — Lua 整数溢出导致 RCE
# 影响：Lua 脚本中的整数溢出可导致远程代码执行
# 评分：CRITICAL
# 修复版本：>= 8.0.4

# CVE-2025-46818 — Lua 脚本跨用户执行
# 影响：Lua 脚本可在其他用户上下文环境下执行
# 评分：HIGH
# 修复版本：>= 8.0.4

# CVE-2025-46819 — Lua 越界读取
# 影响：Lua 脚本可读取越界内存
# 评分：HIGH
# 修复版本：>= 8.0.4
```

### 8.0.3 修复的 CVE

```bash
# CVE-2025-32023 — HyperLogLog 越界写入
# 影响：攻击者可利用 HLL 命令导致 Redis 崩溃或代码执行
# 评分：HIGH
# 修复版本：>= 8.0.3
```

### 8.0.6 修复的安全问题

```bash
# 安全检查：禁止连接注入 \r\n 序列到错误回复中
# 影响：恶意连接可通过精心构造的数据操纵其他连接的读取结果
# 评分：MEDIUM
# 修复版本：>= 8.0.6
```

---

## 关键 Bug 修复（非 CVE）

### 8.0.5（HIGH 紧急度）

```bash
# HGETEX 修复
# 问题：FIELDS 子句缺少 numfields 参数时导致 Redis 崩溃
# 影响：使用 HGETEX 命令的查询可能导致服务中断

# HyperLogLog 溢出
# 问题：超过 2GB 条目的 HLL 操作可能导致崩溃

# Bloom/Cuckoo 过滤器安全修复
# - Cuckoo 过滤器除零错误
# - Cuckoo 过滤器计数器溢出
# - Bloom 过滤器任意内存读写
# - Bloom 过滤器空链越界访问
# - Top-K 越界访问
```

### 8.0.4 非安全修复

```bash
# VSIM 命令新增 EPSILON 参数
VSIM myvectors 0.1 0.2 0.3 ... COUNT 10 EPSILON 0.5

# 潜在 use-after-free 修复
# - PubSub + Lua defrag 场景修复
# - Lua 脚本 defrag 潜在崩溃修复

# 复制相关修复
# - HINCRBYFLOAT 在 replica 上错误移除字段过期时间
# - 带 TTL 的哈希在同步时的短读错误处理
```

### 8.0.6 Bug 修复

```bash
# 稳定性修复
# - 配置重写时日志刷新问题修复
# - 防御性编程改进
```

---

## 升级检查清单

```bash
# 1. 检查当前版本
redis-cli INFO SERVER | grep redis_version

# 2. 查看 ACL 是否需要更新（8.0 新增 @search/@json 等分类）
redis-cli ACL LIST

# 3. 检查 Lua 脚本兼容性
# 如果使用了 EVAL/EVALSHA，确保升级后脚本正常运行

# 4. 模块兼容性检查
# 如果之前使用 RedisStack 模块，8.0 已内置无需额外加载

# 5. 升级操作（基于 Docker）
docker pull redis:8.0.6
docker stop redis-8.0.3
docker run --name redis-8.0.6 \
    --restart=always \
    -p 6379:6379 \
    -v /data/redis:/data \
    redis:8.0.6 \
    redis-server --appendonly yes

# 6. 验证升级
redis-cli INFO SERVER | grep redis_version
# redis_version:8.0.6
```

## 注意事项

1. **Lua 脚本用户**：如果应用大量使用 Lua 脚本，建议先在测试环境验证 8.0.4+ 的兼容性
2. **Bloom/Cuckoo 过滤器用户**：8.0.5 修复了过滤器相关内存读写漏洞，使用 BF/CF 命令必须升级
3. **VSIM WITHATTRIBS**：8.0.3 新增 `WITHATTRIBS` 选项可返回 JSON 属性，8.0.4 新增 `EPSILON` 参数
4. **ACL 审计**：升级后审查 ACL 规则，确保 `@search`, `@json` 等新分类的权限正确
5. **RDB 格式**：8.0.x 的 RDB 文件格式一致，小版本间可直接升级降级（仅限 8.0.x 系列内）
