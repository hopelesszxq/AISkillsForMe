---
name: docker-compose-production
description: Docker Compose 生产部署最佳实践：健康检查、资源限制、日志轮转、Secrets 管理、滚动更新
tags: [tools, docker, docker-compose, production, devops]
---

## 概述

Docker Compose 不仅适用于本地开发，通过合理配置也可以承载小规模生产负载。本文覆盖生产级 Compose 的核心配置：健康检查、资源限制、Secrets 管理、日志轮转和滚动更新。

## 工具选型参考

| 规模 | 推荐工具 | 理由 |
|------|----------|------|
| 1-3 服务，单主机 | **Docker Compose** | 配置最简单，单文件，无编排开销 |
| 3-10 服务，2-5 主机 | **Docker Swarm** | Compose 兼容语法，内置 LB 和 Secrets |
| 10+ 服务，多区域，自动扩缩 | **Kubernetes** | 完整编排，Service Mesh，生态丰富 |

## 生产级 Compose 文件模板

```yaml
# docker-compose.prod.yml
services:
  app:
    image: myapp:latest
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: '512M'
        reservations:
          cpus: '0.5'
          memory: '256M'
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    restart: unless-stopped
    env_file:
      - .env.production
    secrets:
      - db_password
      - jwt_secret
    volumes:
      - app_uploads:/app/uploads

  postgres:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - pgdata:/var/lib/postgresql/data
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
  jwt_secret:
    file: ./secrets/jwt_secret.txt

volumes:
  pgdata:
  app_uploads:
```

## 关键生产配置说明

### 1. 健康检查（Healthcheck）

健康检查是 **服务依赖顺序** 的基础：

```yaml
services:
  app:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s      # 每30s检查一次
      timeout: 10s        # 单次检查超时
      retries: 3          # 连续失败3次标记为unhealthy
      start_period: 40s   # 启动后40s内不检查（留给应用初始化）
```

在依赖服务中使用 `depends_on` + `condition` 等待依赖就绪：

```yaml
services:
  app:
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
```

### 2. 资源限制

| 配置项 | 作用 | 推荐值 |
|--------|------|--------|
| `deploy.resources.limits` | **硬上限**——容器不能超过此限制 | 根据应用 profile 设定 |
| `deploy.resources.reservations` | **软下限**——调度器保证分配 | limits 的 50-75% |

如果未设置 `memory` 限制，单容器可能耗尽主机所有内存导致 OOM。

### 3. 日志轮转

```yaml
logging:
  driver: "json-file"      # 默认驱动
  options:
    max-size: "10m"        # 每个日志文件最大 10MB
    max-file: "3"          # 保留最近 3 个文件
```

> 每个服务约占用 30MB 日志空间。不设置轮转会导致磁盘写满。

### 4. Secrets 管理

```yaml
# 定义 secrets
secrets:
  db_password:
    file: ./secrets/db_password.txt   # 文件方式（慎用生产）
    # 或使用外部 secrets（Swarm 模式更安全）
    # external: true

# 挂载到服务
services:
  app:
    secrets:
      - db_password

# 在容器内：/run/secrets/db_password
```

> **Swarm 模式**下，Secrets 加密存储在 Raft 日志中，以 tmpfs 挂载到容器，比环境变量更安全。

### 5. 重启策略

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| `no` | 不自动重启 | 一次性任务 |
| `always` | 总是重启 | 极少数特殊用途 |
| `unless-stopped` | 手动停止则不重启 | **生产推荐** |
| `on-failure[:max-retries]` | 仅退出码非0时重启 | 调试时使用 |

## 零停机滚动更新（Swarm 模式）

Swarm 模式下 Compose 文件获得滚动更新能力：

```yaml
services:
  app:
    image: myapp:latest
    deploy:
      replicas: 3                    # 3 个副本
      update_config:
        parallelism: 1               # 每次更新 1 个副本
        delay: 10s                   # 副本间等待 10s
        order: start-first           # 先启动新容器再停旧容器
        failure_action: rollback     # 失败则回滚
      rollback_config:
        parallelism: 1
        delay: 5s
```

更新命令：
```bash
docker stack deploy -c docker-compose.prod.yml myapp
```

## 生产 Checklist

- [ ] 所有服务配置 healthcheck
- [ ] 关键服务配置 CPU/内存 limits
- [ ] 日志配置 max-size + max-file
- [ ] Secrets 使用 Docker secrets 或外部 vault
- [ ] 容器间依赖使用 `condition: service_healthy`
- [ ] 敏感信息不硬编码在 yaml 中（使用 `.env` 或 secrets）
- [ ] 使用 `restart: unless-stopped`
- [ ] 持久数据使用 named volumes（非 bind mount）
- [ ] 仅暴露必要端口到 host

## 注意事项

### 1. `depends_on` 的局限性

```yaml
# ❌ 仅保证启动顺序，不保证服务就绪
depends_on:
  - postgres

# ✅ 等待健康检查通过
depends_on:
  postgres:
    condition: service_healthy
```

### 2. 环境变量 vs Secrets

```yaml
# ❌ 密码明文暴露（docker inspect 可见）
environment:
  - DB_PASSWORD=supersecret

# ✅ 使用 secrets 挂载
secrets:
  - db_password
```

### 3. Compose watch（开发）不用于生产

`develop.watch` 是 Docker Compose 2.x 的开发特性，用于热重载，不应在生产 Compose 文件中使用。

### 4. Compose 版本兼容性

使用 `version: "3.8"` 或更高版本以支持所有生产特性。Compose V2（`docker compose` 命令）不需要 version 字段。
