---
name: rabbitmq-patterns
description: RabbitMQ 消息队列最佳实践
tags: [rabbitmq, mq, message]
---

## 核心模式
- Direct Exchange：点对点
- Topic Exchange：通配符路由
- Fanout Exchange：广播

## 可靠性
- 生产者确认（Publisher Confirm）
- 消费者手动 ACK
- 死信队列（DLQ）处理失败消息
