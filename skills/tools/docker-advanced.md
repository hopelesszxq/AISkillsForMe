---
name: docker-advanced
description: Docker 高阶特性：BuildKit 缓存优化、Compose Watch 热重载、Scout 镜像分析、多平台构建
tags: [tools, docker, buildkit, compose, devops, ci-cd]
---

## BuildKit 高级构建

Docker BuildKit 是下一代构建引擎（Docker 23.0+ 默认启用），大幅提升构建速度和缓存命中率。

### 启用 BuildKit

```bash
# 默认已启用（Docker 23.0+）
export DOCKER_BUILDKIT=1
export COMPOSE_DOCKER_CLI_BUILD=1

# 查看是否启用
docker info | grep BuildKit
# BuildKit: true
```

### 缓存挂载（RUN --mount=type=cache）

```dockerfile
# ===== Java/Maven 构建缓存优化 =====
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app

# 缓存 Maven 依赖下载（大幅加速重复构建）
RUN --mount=type=cache,target=/root/.m2,sharing=locked \
    --mount=type=bind,source=pom.xml,target=pom.xml \
    --mount=type=bind,source=.mvn,target=.mvn \
    [ ! -f mvnw ] || ./mvnw dependency:go-offline

COPY src src
RUN --mount=type=cache,target=/root/.m2,sharing=locked \
    ./mvnw package -DskipTests -B -q

# ===== 运行阶段 =====
FROM eclipse-temurin:21-jre-alpine
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```dockerfile
# ===== Node.js 构建缓存 =====
FROM node:22-alpine AS builder
WORKDIR /app

# 缓存 node_modules 和 npm cache
RUN --mount=type=cache,target=/root/.npm,sharing=locked \
    --mount=type=bind,source=package.json,target=package.json \
    --mount=type=bind,source=package-lock.json,target=package-lock.json \
    npm ci --only=production

COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

### 秘密挂载（避免敏感信息留在镜像中）

```dockerfile
# 构建时使用密钥，但不写入镜像层
RUN --mount=type=secret,id=nexus_pass \
    --mount=type=cache,target=/root/.m2 \
    mvn deploy -Dnexus.password=$(cat /run/secrets/nexus_pass)
```

```bash
# 构建时传递密钥（不通过 build args，不留在镜像历史中）
docker build --secret id=nexus_pass,env=NEXUS_PASSWORD .
```

### SSH 转发（私库依赖）

```dockerfile
# 构建时使用 SSH 密钥访问私有 Git 仓库
RUN --mount=type=ssh \
    git clone git@github.com:myorg/private-lib.git
```

```bash
# 使用宿主机的 SSH agent
docker build --ssh default .
```

## Docker Compose Watch（开发热重载）

Docker Compose 2.23+ 支持 `watch` 指令，实现开发环境**文件变更自动同步**，无需手动重启容器。

### 配置示例

```yaml
# docker-compose.yml
services:
  frontend:
    build:
      context: ./frontend
      target: dev
    ports:
      - "5173:5173"
    develop:
      watch:
        # 1. 同步代码变更（重启容器）
        - action: sync
          path: ./frontend/src
          target: /app/src
          ignore:
            - node_modules/
            - .git/
        
        # 2. 配置文件变更 → 重建镜像
        - action: rebuild
          path: ./frontend/package.json
        
        # 3. 监听到新文件 → 同步
        - action: sync+restart
          path: ./frontend/vite.config.ts
          target: /app/vite.config.ts

  backend:
    build:
      context: ./backend
    ports:
      - "8080:8080"
    develop:
      watch:
        # Java/Gradle 项目：源码变更后自动同步 + 重启
        - action: sync+restart
          path: ./backend/src
          target: /app/src
        - action: rebuild
          path: ./backend/build.gradle
```

### 使用方式

```bash
# 启动开发模式（热重载）
docker compose watch

# 也可以先启动再进入 watch 模式
docker compose up -d
docker compose watch
```

### Spring Boot 开发热重载示例

```dockerfile
# backend/Dockerfile
FROM eclipse-temurin:21-jdk-alpine AS dev
WORKDIR /app

# 安装 spring-boot-devtools（用于自动重启）
COPY gradlew build.gradle settings.gradle ./
COPY gradle gradle
RUN ./gradlew dependencies

# dev 模式下通过 docker compose watch 同步代码
# 无需 COPY src

FROM dev AS prod
COPY src src
RUN ./gradlew bootJar
CMD ["java", "-jar", "build/libs/app.jar"]
```

```yaml
# docker-compose.dev.yml
services:
  app:
    build:
      context: ./backend
      target: dev
    command: ["./gradlew", "bootRun"]
    ports:
      - "8080:8080"
    volumes:
      # Gradle 缓存（持久化，避免重复下载）
      - gradle-cache:/root/.gradle
    develop:
      watch:
        - action: sync+restart
          path: ./backend/src
          target: /app/src
        - action: rebuild
          path: ./backend/build.gradle

volumes:
  gradle-cache:
```

## Docker Scout（镜像安全分析）

Docker Scout 是内置的镜像安全分析工具（Docker Desktop 4.17+ / CLI 插件）。

### 安装与使用

```bash
# 安装 Scout CLI 插件
docker scout install

# 分析本地镜像
docker scout quickview myapp:latest

# 生成 CVE 修复建议
docker scout recommendations myapp:latest

# 比较两个版本的差异
docker scout compare myapp:1.0 myapp:2.0

# CI/CD 中集成（失败门槛）
docker scout cves myapp:latest --only-severity critical --exit-code
```

### GitHub Actions 集成

```yaml
# .github/workflows/docker-scout.yml
name: Docker Scout

on:
  push:
    branches: [main]

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .
      
      - name: Analyze with Scout
        uses: docker/scout-action@v1
        with:
          image: myapp:${{ github.sha }}
          command: cves
          only-severities: critical,high
          exit-code: true  # 发现高危漏洞则 CI 失败
          org: ${{ vars.DOCKER_ORG }}
```

### Dockerfile 安全最佳实践

```dockerfile
# 安全构建示例
FROM eclipse-temurin:21-jre-alpine AS final

# 添加非 root 用户
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app
COPY --from=builder /app/target/app.jar app.jar

# 移除 setuid/setgid 权限
RUN find / -perm /6000 -type f -exec chmod a-s {} \; || true

# 非 root 运行
USER appuser

# 只读根文件系统
LABEL "docker.security.read-only-rootfs"="true"

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 多平台构建（Multi-arch）

使用 Docker Buildx 构建多架构镜像（amd64 + arm64）。

### 配置与构建

```bash
# 1. 创建 Buildx 构建器
docker buildx create --name multiarch --driver docker-container --use
docker buildx inspect --bootstrap

# 2. 构建并推送多架构镜像
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    --tag myapp:latest \
    --push .

# 3. 本地构建单一架构
docker buildx build --platform linux/arm64 --load -t myapp:arm64 .
```

### GitHub Actions 多架构构建

```yaml
# .github/workflows/docker-build.yml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: |
            ${{ secrets.DOCKER_USERNAME }}/myapp:latest
            ${{ secrets.DOCKER_USERNAME }}/myapp:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## Docker Init（快速生成）

```bash
# Docker 4.30+ 支持：为项目自动生成 Dockerfile + .dockerignore + compose.yaml
docker init

# 交互式选择语言/框架
# 支持：Java/Spring Boot, Node, Python/Flask, Go, Rust, .NET, PHP
```

## 关键注意事项

1. **BuildKit 缓存共享**：`sharing=locked` 防止并发构建时缓存损坏（CI 环境必设）
2. **Compose Watch 限制**：仅用于开发环境，不要在生产使用
3. **Scout 配额**：免费版每月有限额，团队用需付费订阅
4. **多平台构建性能**：arm64 模拟构建比本地慢 2-3 倍，建议用 native arm runner
5. **Buildx 构建器持久化**：构建器默认临时的，`docker buildx create --driver docker-container` 创建的命名构建器会持久化缓存
6. **LAYER CACHE 失效**：`COPY` 指令前的 `RUN` 层要合理安排，不常变的放前面
