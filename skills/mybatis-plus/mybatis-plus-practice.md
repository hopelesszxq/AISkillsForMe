---
name: mybatis-plus-practice
description: MyBatis-Plus 使用规范
tags: [mybatis-plus, orm, database]
---

## 核心功能
- LambdaQueryWrapper 链式查询
- 分页插件 PaginationInnerInterceptor
- 自动填充（MetaObjectHandler）
- 逻辑删除 @TableLogic

## 规范
- 表名约定：下划线命名
- Service 层继承 IService
- 复杂 SQL 用 XML 或自定义 Mapper
