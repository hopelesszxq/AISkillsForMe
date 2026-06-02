---
name: pg184-security-update
description: PostgreSQL 18.4 安全更新：10+ CVE 修复、内存溢出、逻辑复制注入、启动包递归拒绝服务
tags: [postgresql, pg18, security, cve, upgrade, bugfix]
---

## 概述

PostgreSQL 18.4（2026-05-14 发布）是 18.x 系列的安全维护版本，**修复了 10+ 个 CVE 漏洞**，涵盖内存溢出、SQL 注入、拒绝服务等高风险问题。同时发布了 17.10、16.14、15.18、14.23。

> **升级建议**：所有 PostgreSQL 18.x 生产环境**应立即升级**到 18.4。
> 从 18.x 升级无需 dump/restore。

## 高危 CVE 明细

### CVE-2026-6479 — 启动包递归拒绝服务（高危）

恶意客户端通过交替发送被拒绝的 SSL 和 GSS 加密请求，可导致后端进程**无限递归**而崩溃。

```python
# 攻击原理：客户端交替请求 SSL / GSS 加密
# 服务端反复拒绝，触发 unbounded recursion
```

### CVE-2026-6473 — 内存分配整数溢出（多路径，高危）

多个代码路径在计算内存分配大小时未检查整数溢出，导致分配过小的缓冲区，后续写入越界。可能触发**服务器崩溃或任意代码执行**。

**影响的代码路径包括：**
- 超长 `session_authorization` 字符串（超过 32KB 未检查）
- 各类 `palloc`/`palloc0` 调用中的大小计算
- 32-bit 构建风险更高

### CVE-2026-6476 — 订阅名 SQL 注入

订阅名在拼接 SQL 命令时**未加引号**，若订阅名来自不可信来源可导致注入攻击。

```sql
-- ❌ 危险：subscription_name 未引号转义
-- ALTER SUBSCRIPTION ... REFRESH PUBLICATION
```

### CVE-2026-6638 — 逻辑复制对象名 SQL 注入

`ALTER SUBSCRIPTION ... REFRESH PUBLICATION` 将 schema 和 relation 名未加引号地插入 SQL 命令，允许在 Publisher 上**执行任意 SQL**。

### CVE-2026-6575 — MCV 统计信息损坏导致计划器崩溃

统计信息恢复函数未充分校验 most-common-value 统计数据的有效性，可接受导致计划器后续崩溃的坏值。

### 其他 CVE

| CVE | 风险 | 描述 |
|-----|------|------|
| CVE-2026-6472 | 中 | 32KB session_authorization 长度未检查 |
| CVE-2026-6474 | 高 | 输出缓冲区大小溢出 |
| CVE-2026-6475 | 中 | 多内存分配点整数溢出 |
| CVE-2026-6477 | 中 | PQinitOpenSSL 客户端 API 弃用 |
| CVE-2026-6478 | 中 | 内部缓冲区大小传递问题 |
| CVE-2026-6637 | 低 | 时区名注入防护 |

## 其他重要修复

### 逻辑复制

- **Walsender 关闭修复**：集群关闭时，发布逻辑复制的 walsender 未能正确请求刷出所有待处理 WAL，可能导致**无限等待**
- **订阅清理修复**：修复停止订阅时可能的数据丢失

### 性能与稳定性

- **分区键匹配改进**：优化计划器对分区键列与子查询输出的匹配
- **并行 B-tree 索引**：修复共享内存分配不足问题
- **ICU 字符串处理**：修复内存泄漏
- **最旧 multixact 数组**：修复共享内存中索引错误，可能导致行锁不可见
- **tar/pg_dump**：修复 LZ4 压缩数据边界情况损坏、内存泄漏等问题

### 安全性增强

- **信任认证日志**：`trust` 认证现在会在服务器日志中记录认证事件
- **客户端 PQinitOpenSSL 弃用**：标记为废弃，因无法在不改 API 前提下修复缓冲区溢出

## 升级注意事项

```bash
# 1. 下载并安装
wget https://ftp.postgresql.org/pub/source/v18.4/postgresql-18.4.tar.gz
tar xzf postgresql-18.4.tar.gz
cd postgresql-18.4
./configure --prefix=/usr/local/pgsql/18.4
make -j$(nproc)
sudo make install

# 2. 重启集群（无需 dump/restore）
sudo systemctl restart postgresql-18

# 3. 验证版本
psql -c "SELECT version();"
# PostgreSQL 18.4 on x86_64-pc-linux-gnu ...
```

## 参考

- [PG 18.4 Release Notes](https://www.postgresql.org/docs/18/release-18-4.html)
- [PG 安全公告页](https://www.postgresql.org/support/security/)
