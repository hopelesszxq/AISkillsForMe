---
name: pg-best-practices
description: PostgreSQL 使用最佳实践
tags: [postgresql, database]
---

## 设计规范
- 主键推荐 UUID v7 或雪花 ID，或自增 BIGSERIAL
- 合理使用索引：B-tree, GIN, GiST
- 避免 N+1 查询

## 性能
- EXPLAIN ANALYZE 分析慢查询
- 连接池推荐 HikariCP 配合 maximum-pool-size 配置
- 分页用 Keyset Pagination 替代 OFFSET
