---
name: docker-practice
description: Docker 容器化最佳实践与多阶段构建
tags: [tools, docker, container, devops]
---

## 多阶段构建

```dockerfile
# === 构建阶段 ===
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY mvnw pom.xml ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline -B
COPY src src
RUN ./mvnw package -DskipTests -B

# === 运行阶段 ===
FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
USER appuser
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 镜像瘦身技巧

```dockerfile
# 合并 RUN 减少层数
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# 使用 distroless 基础镜像（仅运行时无 shell）
FROM gcr.io/distroless/java21-debian12
COPY --from=builder /app/target/app.jar app.jar
CMD ["app.jar"]
```

| 策略 | 效果 |
|---|---|
| 多阶段构建 | 移除构建工具，Java 镜像从 500MB → 150MB |
| Alpine 镜像 | 最小基础镜像，但 musl libc 可能有兼容问题 |
| Distroless | 极安全，无 shell/su，但有调试困难 |
| JLink 定制 JRE | 只包含需要的模块，可缩至 30-50MB |

## Docker Compose 配置

```yaml
version: "3.8"
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - DB_URL=jdbc:postgresql://db:5432/app
      - REDIS_HOST=redis
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: app
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}

volumes:
  pgdata:
  redis-data:
```

## Docker 网络注意事项

```bash
# 创建自定义网络（推荐替代默认 bridge）
docker network create --driver bridge --subnet 172.20.0.0/16 app-network

# 容器间通信用服务名而不是 IP
# app → db, app → redis

# 暴露多端口，但只开放必要端口
# 内部端口: DB 5432, Redis 6379 不需要对外暴露
```

## 资源限制

```yaml
# docker-compose 中的资源限制
services:
  app:
    deploy:
      resources:
        limits:
          cpus: "2"
          memory: 2G
        reservations:
          cpus: "1"
          memory: 1G
  # JVM 感知容器内存限制（JDK 10+ 自动识别）
  # 手动指定：-XX:ActiveProcessorCount=2 -XX:MaxRAMPercentage=70.0
```

## 注意事项

1. **不要在镜像中存敏感信息**：build args 会被保留在镜像历史中，用运行时环境变量
2. **时区问题**：Alpine 镜像默认 UTC，安装 `tzdata` 或挂载 `/etc/localtime`
3. **PID 1 问题**：Java 默认不是 init 进程，无法处理 SIGTERM → 使用 `dumb-init` 或 `tini`
4. **日志处理**：应用日志输出到 stdout/stderr，由 Docker 日志驱动收集
5. **Layer 缓存**：经常变化的文件放在 Dockerfile 末尾，利用缓存加速构建
6. **安全扫描**：`docker scout` 或 `trivy` 定期扫描镜像漏洞
