# AISkillsForMe

> 开发技能资料库 —— 按技术栈分类整理的 AI Skills / Rules 合集

## 📂 目录结构

```
skills/
├── spring-cloud/       # Spring Cloud 全家桶
│   ├── nacos/          # 服务注册与配置中心
│   ├── gateway/        # 网关
│   ├── sentinel/       # 限流熔断
│   └── feign/          # 声明式调用
├── vue3/               # Vue3 生态
├── postgresql/         # PostgreSQL 数据库
├── redis/              # Redis 缓存
├── minio/              # MinIO 对象存储
├── onlyoffice/         # OnlyOffice 文档服务
├── rabbitmq/           # RabbitMQ 消息队列
├── rocketmq/           # RocketMQ 消息队列
├── mybatis-plus/       # MyBatis-Plus ORM
├── tools/              # 通用开发工具
└── patterns/           # 架构/设计模式
```

## 🎯 用途

每天从网络上收集高质量的技术资料，按类别整理到这里。AI 编码时可以通过 AGENTS.md 加载对应技能的 rules，让 AI 遵循最佳实践。

## 📝 Skill 格式

每个技能文件遵循以下格式：

```markdown
---
name: 技能名称（英文小写）
description: 一句话描述
tags: [分类标签]
---

## 要点

核心知识点和最佳实践

## 使用场景

何时使用这个技能

## 注意事项

常见的坑和避坑指南

## 参考来源

收集自何处（可选）
```

## 🚀 使用方式

在 Hermes Agent 中告诉 AI：

> "加载 spring-cloud/gateway 的 skills"

或者让 AI 自动根据项目需要加载对应技能。
