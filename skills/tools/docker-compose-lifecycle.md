---
name: docker-compose-lifecycle
description: Docker Compose v5.1 生命周期管理：停止钩子、依赖顺序、健康检查与优雅关闭实战
tags: [tools, docker, docker-compose, lifecycle, orchestration, healthcheck]
---

## 概述

Docker Compose v5.1.4（2026-05 发布）引入了 **Stop Lifecycle Hook**（停止生命周期钩子），这是 Compose 在容器生命周期管理方面的重要增强。结合已有的健康检查、依赖顺序和重启策略，构建完整的生命周期管理体系。

> 已有技能 `docker-compose-production.md`、`docker-practice.md` 和 `docker-compose-patterns.md` 覆盖了基础使用，本文聚焦生命周期高级管理。

## 一、Stop Lifecycle Hook（Docker Compose v5.1+）

### 功能说明

Stop Lifecycle Hook 允许在**容器停止时**执行清理操作，特别适用于：

- 优雅关闭 Sidecar/Proxy 代理
- 通知外部服务（服务注册中心）实例下线
- 刷新持久化缓冲数据
- 清理临时资源

### 配置方式

```yaml
services:
  app:
    image: my-app:latest
    stop_grace_period: 30s
    # v5.1+ 停止钩子
    post_stop:
      - command: ["/bin/sh", "-c", "curl -X POST http://registry:8500/v1/agent/service/deregister/my-app"]
        timeout: 10s
      - command: ["/bin/sh", "-c", "sync && echo 'Flushed buffers'"]
        timeout: 5s

  proxy:
    image: envoyproxy/envoy:v1.33
    stop_grace_period: 60s
    # Sidecar 需要更多时间完成连接排空
    post_stop:
      - command: ["envoy", "--mode", "post-shutdown"]
        timeout: 30s
```

### 完整生命周期事件流

```yaml
services:
  db:
    image: postgres:17
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 30s

  app:
    image: my-app:latest
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 3
    stop_grace_period: 30s
    post_stop:
      # 通知 Nacos 下线路由实例
      - command:
          [
            "sh", "-c",
            "curl -s -X PUT 'http://nacos:8848/nacos/v1/ns/instance?serviceName=my-app&ip=$$(hostname -i)&port=8080&enabled=false'"
          ]
        timeout: 10s
      # 刷新日志缓冲区
      - command: ["sh", "-c", "kill -USR1 1 && sleep 1"]
        timeout: 5s

  nginx:
    image: nginx:1.27
    ports:
      - "80:80"
    depends_on:
      app:
        condition: service_started
```

## 二、高级依赖顺序管理

### 多阶段依赖

```yaml
services:
  init-db:
    image: my-migration:latest
    depends_on:
      postgres:
        condition: service_healthy
    profiles:
      - dont-start  # 仅用于依赖，不自动启动

  postgres:
    image: postgres:17
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 3s
      timeout: 2s
      retries: 10
      start_period: 15s
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7.4
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 3s
      timeout: 2s
      retries: 5

  # ✅ 协调所有依赖就绪后再启动主应用
  backend:
    image: my-backend:latest
    depends_on:
      init-db:
        condition: service_completed_successfully
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health/liveness"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 60s  # Spring Boot 启动较慢
```

### 条件重启依赖

```yaml
services:
  config-server:
    image: spring-cloud-config-server:latest
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8888/actuator/health"]
      interval: 5s
      timeout: 3s
      retries: 10
    restart: always

  # ✅ service_healthy 确保配置中心就绪后再启动
  gateway:
    image: spring-cloud-gateway:latest
    depends_on:
      config-server:
        condition: service_healthy
    environment:
      - SPRING_CLOUD_CONFIG_URI=http://config-server:8888
    restart: on-failure
```

## 三、Restart Policies 深度配置

### 不同服务的重启策略

```yaml
services:
  # 核心服务：宕机必须立即恢复
  core-api:
    image: core-api:latest
    restart: always                          # 任何情况都重启
    deploy:
      replicas: 3
      restart_policy:
        condition: any
        delay: 5s
        max_attempts: 5
        window: 120s

  # 批量任务：失败不重启（让编排器处理）
  batch-job:
    image: batch-processor:latest
    restart: "no"                            # 不自动重启
    profiles: ["batch"]

  # 辅助服务：可接受短暂不可用
  cache-sidecar:
    image: redis:7.4
    restart: unless-stopped                  # 手动停止的不重启
    stop_grace_period: 5s

  # 一次性初始化
  db-migration:
    image: flyway/flyway:latest
    restart: "no"
    profiles: ["migration"]
```

## 四、优雅关闭最佳实践

### Spring Boot 应用

```yaml
services:
  app:
    image: my-spring-app:latest
    # 确保优雅关闭
    stop_grace_period: 45s
    environment:
      - SPRING_LIFECYCLE_TIMEOUT_PER_SHUTDOWN_PHASE=30s
      - SERVER_SHUTDOWN=graceful
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 60s
    # v5.1+：关闭前执行清理
    post_stop:
      - command:
          [
            "sh", "-c",
            "echo 'App stopped at $$(date)' >> /var/log/lifecycle.log"
          ]
        timeout: 5s
```

### 组合多个 Stop Hook

```yaml
services:
  message-worker:
    image: message-worker:latest
    stop_grace_period: 120s  # 给消息处理足够时间
    environment:
      - GRACEFUL_SHUTDOWN_WAIT=60
    post_stop:
      # 1. 从注册中心注销
      - command: ["sh", "-c", "curl -X PUT 'http://nacos:8848/nacos/v1/ns/instance?serviceName=worker&ip=$$(hostname -i)&port=8080&enabled=false'"]
        timeout: 10s
      # 2. 等待正在处理的消息完成
      - command: ["sh", "-c", "while [ $(curl -s http://localhost:8080/actuator/health | jq -r '.details.inProgress') -gt 0 ]; do sleep 1; done"]
        timeout: 60s
      # 3. 持久化偏移量
      - command: ["sh", "-c", "curl -X POST http://localhost:8080/actuator/offset/commit"]
        timeout: 15s
```

## 五、监控生命周期事件

```yaml
services:
  lifecycle-monitor:
    image: alpine:3.21
    restart: "no"
    profiles: ["monitor"]
    command:
      - sh
      - -c
      - |
        # 简单监控：监听 docker events
        docker events --filter 'type=container' --filter 'event=stop' --filter 'event=die'
```
```yaml
# 使用 docker compose events 监控
# docker compose events --json
# 输出示例：
# {"time":"2026-06-01T12:00:00Z","type":"container","action":"health_status","service":"app"}
# {"time":"2026-06-01T12:00:05Z","type":"container","action":"stop","service":"app"}
```

## 注意事项

| 要点 | 说明 |
|------|------|
| **Stop Hook 版本** | `post_stop` 钩子需要 Docker Compose v5.1+ |
| **超时设置** | `stop_grace_period` 应大于 post_stop 最大超时之和 |
| **SIGTERM 处理** | 应用必须正确处理 SIGTERM 信号实现优雅关闭 |
| **环境变量** | post_stop 命令中可使用 `$$HOSTNAME` 等 Compose 模板变量 |
| **幂等性** | post_stop 命令应幂等，防止重复执行导致错误 |
| **Docker 版本** | Docker Engine 27+ 推荐用于 Compose v5.x |
| **健康检查** | 健康检查失败不会自动停止容器，需结合 `depends_on` 控制依赖 |
| **调试 post_stop** | 使用 `docker compose logs` 查看 post_stop 命令的输出 |
