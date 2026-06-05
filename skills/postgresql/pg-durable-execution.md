---
name: pg-durable-execution
description: Microsoft pg_durable — PostgreSQL 内数据库级 Durable Execution 扩展，SQL 工作流持久化执行
tags: [postgresql, durable-execution, workflow, microsoft, extension, pgrx]
---

## 概述

**pg_durable** 是微软开源（2026-06）的 PostgreSQL 扩展，将 **Durable Execution（持久化执行）** 模式引入数据库内部。它允许开发人员用纯 SQL 定义可容错、可恢复的工作流函数，在崩溃、重启或步骤失败后从最后一个检查点自动恢复执行，无需外部编排服务（Temporal、Airflow、Step Functions）。

- **仓库**: [microsoft/pg_durable](https://github.com/microsoft/pg_durable)
- **Stars**: 510+（快速上升中）
- **语言**: Rust（基于 pgrx 框架）
- **许可证**: PostgreSQL License
- **支持**: PostgreSQL 17 & 18
- **状态**: Preview

### 解决了什么问题

传统上，在 PostgreSQL 中做后台任务需要拼凑 `pg_cron` + 状态表 + 重试计数器 + 轮询 Worker，或者引入外部编排器。pg_durable 让工作流定义、状态追踪和检查点全部在数据库内完成：

| 场景 | 传统方案 | pg_durable |
|---|---|---|
| 向量嵌入管道 | 外部脚本 + 状态表 | 纯 SQL 定义，内建检查点 |
| 数据导入清洗 | Airflow DAG | SQL 图 + 自动重试 |
| 定时运维 | pg_cron + 手动状态管理 | 声明式工作流，崩溃恢复 |
| 扇出聚合 | 应用层并行 + 补偿逻辑 | `df.join()` 原生并行 |

### 适用场景

- AI 向量嵌入管道：分块 → 调用 Embedding API → upsert 到 pgvector
- 数据摄入管道：暂存 → 去重 → 转换 → 发布
- 定时维护：检测膨胀 → 通知 → 等待审批 → 执行
- 扇出聚合：并行执行独立查询，合并结果
- 外部 API 工作流：从 SQL 中调用 enrichment、classification 或 webhook

## 架构

```
┌──────────────────────────────────────────────────────────────┐
│                       PostgreSQL                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │             pg_durable 扩展 (pgrx/Rust)                │  │
│  │                                                        │  │
│  │  SQL DSL:  'sql' |=> 'name' ~> 'sql2'                 │  │
│  │           df.if() | df.join() | df.loop()              │  │
│  │                                                        │  │
│  │  后台 Worker（内嵌 duroxide 运行时）                    │  │
│  │  ┌────────────────────────────────────────────────┐    │  │
│  │  │  duroxide      （编排运行时）                    │    │  │
│  │  │  duroxide-pg   （PostgreSQL 状态提供者）        │    │  │
│  │  └────────────────────────────────────────────────┘    │  │
│  └────────────────────────────────────────────────────────┘  │
│  Schema: df.*（图、实例、变量）, duroxide.*（运行时状态）    │
└──────────────────────────────────────────────────────────────┘
```

**核心依赖**：
- **[duroxide](https://github.com/microsoft/duroxide)** — 通用 Durable Task 框架，提供确定性重放、检查点、子编排、定时器
- **[duroxide-pg](https://github.com/microsoft/duroxide-pg)** — PostgreSQL 状态存储后端

## 快速开始

### 安装

```bash
# Debian 包安装（PG17/18）
wget https://github.com/microsoft/pg_durable/releases/download/v0.1.0/pg-durable-postgresql-17_0.1.0-1_amd64.deb
dpkg -i pg-durable-postgresql-17_0.1.0-1_amd64.deb

# 配置 shared_preload_libraries
echo "shared_preload_libraries = 'pg_durable'" >> /etc/postgresql/17/main/postgresql.conf

# 重启 PostgreSQL
systemctl restart postgresql

# 创建扩展
psql -d mydb -c "CREATE EXTENSION pg_durable;"
```

### 定义并启动第一个工作流

```sql
-- 定义一个分步处理工作流
SELECT df.start(
    'SELECT id FROM documents WHERE processed = false LIMIT 100' |=> 'batch'
    ~> 'UPDATE documents SET processed = true WHERE id = ANY($batch)'
);
```

### 完整示例：扇出-聚合

```sql
-- 并行统计，然后聚合结果
SELECT df.start(
    df.join(
        'SELECT count(*) FROM users'   |=> 'user_count',
        'SELECT count(*) FROM orders'  |=> 'order_count',
        'SELECT sum(amount) FROM orders' |=> 'total_revenue'
    )
    ~> 'SELECT $user_count AS users, $order_count AS orders, $total_revenue AS revenue'
    |=> 'dashboard'
);
```

### 条件分支与循环

```sql
-- 条件分支
SELECT df.start(
    'SELECT count(*) AS cnt FROM documents WHERE status = ''pending''' |=> 'check'
    ~> df.if('$check > 0',
        'UPDATE documents SET status = ''processing'' WHERE status = ''pending''' |=> 'process',
        'SELECT ''no work to do''' |=> 'skip'
    )
);

-- 循环处理
SELECT df.start(
    df.loop(
        'SELECT id FROM queue WHERE status = ''pending'' LIMIT 10' |=> 'items',
        'SELECT count(*) AS remaining FROM queue WHERE status = ''pending''' |=> 'remaining',
        df.if('$remaining = 0', df.break(), df.noop())
    )
);
```

## 核心 DSL 操作符

| 操作符 | 含义 | 示例 |
|---|---|---|
| `\|=> 'name'` | 将 SQL 结果绑定到命名变量 | `'SELECT ...' \|=> 'my_var'` |
| `~>` | 顺序连接步骤 | `step1 ~> step2` |
| `df.join(...)` | 并行执行多个步骤 | `df.join(step1, step2)` |
| `df.if(cond, then, else)` | 条件分支 | `df.if('$x > 0', step_a, step_b)` |
| `df.loop(body)` | 循环执行 | `df.loop(process_step)` |
| `df.break()` | 跳出循环 | 配合 `df.if()` 使用 |
| `df.noop()` | 空操作 | 占位符 |
| `df.http(url, options)` | HTTP 调用外部 API | `df.http('https://api.example.com')` |

## 监控与运维

工作流实例和节点状态存储在标准 PostgreSQL 表中，可用 SQL 直接查询：

```sql
-- 查看所有工作流实例
SELECT * FROM df.instances ORDER BY created_at DESC LIMIT 10;

-- 查看某个工作流的步骤节点
SELECT * FROM df.nodes WHERE instance_id = '...' ORDER BY created_at;

-- 查看变量
SELECT * FROM df.vars WHERE instance_id = '...';

-- 取消运行中的工作流
SELECT df.cancel('instance_id');
```

## 多用户与权限

扩展安装后不自动授予 PUBLIC 权限，需显式授权：

```sql
-- 授予应用角色
SELECT df.grant_usage('app_role');

-- 或通过中间角色
CREATE ROLE pg_durable_user NOLOGIN;
SELECT df.grant_usage('pg_durable_user');
GRANT pg_durable_user TO app_backend, etl_service;
```

**安全要点**：
- 后台 Worker 角色（`pg_durable.worker_role` GUC）**必须是超级用户**
- RLS 确保用户只能操作自己的实例
- 升级扩展后需重新执行 `df.grant_usage()`

## 注意事项

1. **Preview 阶段** — 不建议用于核心生产业务，适合评估和实验
2. **SQL 限制** — 工作流逻辑必须是 SQL 形态；复杂应用逻辑需包装为 SQL 函数或通过 HTTP 暴露
3. **性能边界** — 不适合亚毫秒级同步请求处理
4. **PostgreSQL 版本** — 当前仅支持 PG17 和 PG18
5. **Rust 构建** — 从源码构建需要 Rust nightly + cargo-pgrx 0.16.1
6. **升级注意** — 扩展升级后需重新授权，新函数可能不会自动对现有角色可见
7. **无遥测** — pg_durable **不会**向微软发送遥测数据
8. **Azure HorizonDB** — 可在微软的 PostgreSQL 云服务 HorizonDB 中直接体验 pg_durable
