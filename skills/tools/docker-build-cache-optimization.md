---
name: docker-build-cache-optimization
description: Docker BuildKit 构建缓存优化策略——缓存挂载、远程缓存、多阶段缓存、CI/CD 缓存加速
tags: [tools, docker, buildkit, cache, ci-cd, performance]
---

## 概述

Docker 构建速度是 CI/CD 效率的关键瓶颈。BuildKit（Docker 23.0+ 默认）提供了多层缓存机制，合理配置可减少 80%+ 的重复构建时间。本文覆盖本地缓存、远程缓存、缓存挂载等实战优化策略。

## 一、构建缓存基本原理

### 1.1 Docker 层缓存（Layer Cache）

每一条 `RUN`/`COPY`/`ADD` 指令生成一个只读层。当 Dockerfile 未变化且上游层未变化时，该层从缓存复用：

```dockerfile
# 如果 pom.xml 未变化，依赖下载层直接命中缓存
COPY pom.xml ./
RUN mvn dependency:go-offline    # 这层被缓存

COPY src ./src
RUN mvn package                  # 这层因 src 变化而重跑
```

**递减原则**：将变化频率低的操作放在 Dockerfile 前面。

### 1.2 BuildKit 新增特性

BuildKit 相比传统 Builder 的关键增强：

- `RUN --mount=type=cache`：持久化缓存目录
- `RUN --mount=type=bind`：只读绑定源码
- `--cache-from` / `--cache-to`：远程缓存导入导出
- 并行构建、SSH 转发、Secret 注入

## 二、缓存挂载（RUN --mount=type=cache）

### 2.1 Maven/Gradle 依赖缓存

```dockerfile
# ===== Maven 构建缓存 =====
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app

# 挂载 Maven 本地仓库缓存（持续跨构建复用）
RUN --mount=type=cache,target=/root/.m2,sharing=locked \
    --mount=type=bind,source=pom.xml,target=pom.xml \
    [ ! -f mvnw ] || ./mvnw dependency:go-offline -B

COPY src src
RUN --mount=type=cache,target=/root/.m2,sharing=locked \
    ./mvnw package -DskipTests -B
```

```dockerfile
# ===== Gradle 构建缓存 =====
FROM gradle:8-jdk21 AS builder
WORKDIR /app

# 缓存 Gradle 依赖
RUN --mount=type=cache,target=/root/.gradle,sharing=locked \
    --mount=type=bind,source=build.gradle.kts,target=build.gradle.kts \
    --mount=type=bind,source=settings.gradle.kts,target=settings.gradle.kts \
    gradle dependencies --no-daemon

COPY src src
RUN --mount=type=cache,target=/root/.gradle,sharing=locked \
    gradle build -x test --no-daemon
```

### 2.2 npm/pnpm 依赖缓存

```dockerfile
FROM node:22-alpine AS builder
WORKDIR /app

# 缓存 node_modules 和 npm 缓存
RUN --mount=type=cache,target=/root/.npm,sharing=locked \
    --mount=type=bind,source=package.json,target=package.json \
    --mount=type=bind,source=package-lock.json,target=package-lock.json \
    npm ci --prefer-offline

COPY . .
RUN npm run build
```

```dockerfile
# ===== pnpm 更快 =====
FROM node:22-alpine
WORKDIR /app
RUN corepack enable && corepack prepare pnpm@latest --activate

RUN --mount=type=cache,target=/root/.local/share/pnpm/store,sharing=locked \
    --mount=type=bind,source=pnpm-lock.yaml,target=pnpm-lock.yaml \
    --mount=type=bind,source=package.json,target=package.json \
    pnpm install --frozen-lockfile

COPY . .
RUN pnpm build
```

### 2.3 APT 包缓存

```dockerfile
FROM ubuntu:24.04 AS base

# 缓存 apt 包下载，避免重复下载
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt/lists,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        ca-certificates \
        tzdata && \
    rm -rf /var/cache/apt/archives
```

### 2.4 Python pip 缓存

```dockerfile
FROM python:3.12-slim AS builder

RUN --mount=type=cache,target=/root/.cache/pip,sharing=locked \
    --mount=type=bind,source=requirements.txt,target=requirements.txt \
    pip install -r requirements.txt

COPY . .
```

## 三、远程缓存（CI/CD 共享缓存）

### 3.1 Registry 缓存（--cache-from / --cache-to）

```bash
# 构建时导出缓存到镜像仓库
docker build \
  --cache-from ghcr.io/myorg/myapp:cache \
  --cache-to type=registry,ref=ghcr.io/myorg/myapp:cache,mode=max \
  -t ghcr.io/myorg/myapp:latest \
  .

# mode=max 模式会缓存所有层（包括中间层）
# mode=min 模式仅缓存已打标签的层（体积更小，但命中率略低）
```

### 3.2 S3/Blob 存储缓存

```bash
# 使用 S3 缓存 (需 docker buildx)
docker buildx build \
  --cache-to type=s3,bucket=my-build-cache,region=us-east-1,name=myapp-cache,mode=max \
  --cache-from type=s3,bucket=my-build-cache,region=us-east-1,name=myapp-cache \
  -t myapp:latest .
```

```bash
# 使用本地文件系统缓存（适合自建 CI Runner）
docker build \
  --cache-to type=local,dest=/tmp/docker-cache,mode=max \
  --cache-from type=local,src=/tmp/docker-cache \
  -t myapp:latest .
```

### 3.3 GitHub Actions 缓存示例

```yaml
# .github/workflows/docker-build.yml
name: Docker Build with Cache
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Cache Docker layers
        uses: actions/cache@v4
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-buildx-

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
          cache-from: type=local,src=/tmp/.buildx-cache
          cache-to: type=local,dest=/tmp/.buildx-cache,mode=max
```

## 四、多阶段构建缓存策略

### 4.1 分离构建与运行阶段

```dockerfile
# ===== 阶段1：构建 =====
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app

# 先复制构建配置（低频变化）
COPY mvnw pom.xml ./
COPY .mvn .mvn
RUN --mount=type=cache,target=/root/.m2,sharing=locked \
    ./mvnw dependency:go-offline -B

# 再复制源码（高频变化）
COPY src src
RUN --mount=type=cache,target=/root/.m2,sharing=locked \
    ./mvnw package -DskipTests -B

# ===== 阶段2：运行 =====
FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
USER appuser
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 4.2 基础镜像更新策略

```bash
# 定期拉取基础镜像以确保缓存最新，避免安全漏洞
docker pull eclipse-temurin:21-jdk-alpine --platform linux/amd64
docker build --pull --no-cache-filter=builder ...  # 仅强制重构 builder 阶段
```

## 五、缓存失效分析与调试

### 5.1 检查缓存命中

```bash
# 构建时查看缓存详情
docker build --progress=plain -t myapp:latest . 2>&1 | grep -E "CACHED|ERROR"

# 输出示例
# #8 [builder 3/5] COPY pom.xml ./            CACHED
# #9 [builder 4/5] RUN mvn dependency:go-offline  CACHED
# #10 [builder 5/5] COPY src ./src            0.5s  ← 非缓存
```

### 5.2 常见缓存失效原因

| 原因 | 影响 | 解决 |
|------|------|------|
| 基础镜像 tag 变化 | 全部缓存失效 | 固定 digest 或使用 `--pull` 显式控制 |
| COPY 文件变化 | 该层及后续层失效 | 优化 COPY 顺序，低频先 COPY |
| ARG/ENV 变化 | 该 RUN 层失效 | 稳定 ARG 值，不变的不写入 |
| `ADD` 指令 | 总是破坏缓存 | 用 COPY 替代 ADD（除非需要 tar 自动解压） |
| `--no-cache` 构建 | 无缓存 | 去掉 `--no-cache` |
| RUN 内文件系统变化 | 缓存副本无效 | 使用 `--mount=type=cache` 分离缓存目录 |

### 5.3 分阶段调试

```bash
# 使用 docker history 查看每层大小
docker history myapp:latest

# 使用 dive 工具图形化分析（需安装 dive）
dive myapp:latest
```

## 六、不同语言组合的 Dockerfile 模版

### 6.1 Spring Boot + Maven

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY mvnw pom.xml ./
COPY .mvn .mvn
RUN --mount=type=cache,target=/root/.m2,sharing=locked \
    ./mvnw dependency:go-offline -B
COPY src src
RUN --mount=type=cache,target=/root/.m2,sharing=locked \
    ./mvnw package -DskipTests -B

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 6.2 Go

```dockerfile
FROM golang:1.23-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    CGO_ENABLED=0 go build -o /app/server .

FROM alpine:3.20
COPY --from=builder /app/server /server
EXPOSE 8080
CMD ["/server"]
```

### 6.3 Rust

```dockerfile
FROM rust:1.81-slim-bookworm AS builder
WORKDIR /app
# 创建虚拟入口以缓存依赖
RUN mkdir src && echo 'fn main() {}' > src/main.rs
COPY Cargo.toml Cargo.lock ./
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    cargo build --release && rm -rf src
COPY src src
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    cargo build --release -j $(nproc)

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/myapp /usr/local/bin/
CMD ["myapp"]
```

## 注意事项

1. **缓存目录不用清理**：`--mount=type=cache` 的缓存目录由 BuildKit 管理，自动清理 LRU。不要手动 `rm -rf` 缓存目录。
2. **sharing 策略**：并行构建时 `sharing=locked` 可防止并发写入冲突，但会阻塞其他构建。单构建时用 `shared` 更高效。
3. **CI 缓存持久化**：GitHub Actions / GitLab CI 每次 Runner 都是全新的，必须用远程缓存（Registry/S3）或 actions/cache 保持跨构建缓存。
4. **缓存膨胀**：长时间运行的 CI 缓存会持续增长，建议设置缓存大小上限或定期清理。
5. **构建上下文（.dockerignore）**：`.dockerignore` 遗漏大文件会显著增加发送到 daemon 的上下文体积，破坏缓存效率。
6. **max vs min 模式**：`mode=max` 缓存所有层（加快构建但缓存大）；`mode=min` 只缓存最终镜像层（缓存小但命中率低）。推荐 CI 用 `mode=max`。
7. **多平台构建**：`docker buildx build --platform linux/amd64,linux/arm64` 时，缓存自动按平台隔离。
