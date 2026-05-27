---
name: feign-best-practices
description: OpenFeign 声明式调用最佳实践
tags: [spring-cloud, feign, rpc]
---

## 配置要点
- 开启 `feign.sentinel.enabled=true` 集成 Sentinel 熔断
- 配置连接超时和读取超时
- 使用 `@FeignClient` 的 fallbackFactory 处理降级

## 日志
- feign 日志级别：NONE, BASIC, HEADERS, FULL
- 生产环境建议 BASIC，调试用 FULL
