---
name: docker-bake-practice
description: Docker Bake 多平台构建实战——使用 HCL/JSON 定义构建矩阵、多架构镜像、Spring Boot 项目 CI/CD 集成
tags: [tools, docker, bake, buildx, ci-cd, multi-arch, spring-boot]
---

## 概述

Docker Bake 是 Docker Buildx 内置的高级构建编排工具（`docker buildx bake`），使用 **HCL** 或 **JSON** 配置文件定义多个构建目标。在 Docker Compose v5 中，构建被委托给 Bake，使其成为 Compose 生态的标准构建方式。

**适用场景**：
- 多架构镜像构建（x86 + arm64）
- 同一项目的多个变体构建（dev/staging/prod）
- CI/CD 流水线中的并行构建
- 微服务项目的批量镜像构建

## 与 Compose v5 的关系

Docker Compose v5 内置 BuildKit 构建器已移除，构建委托给 Bake：

```yaml
# docker-compose.yml v5
services:
  order-service:
    # build 指令实际由 Bake 处理
    build:
      context: ./order-service
      tags:
        - registry.example.com/order-service:latest
  payment-service:
    build:
      context: ./payment-service
```

```bash
# Compose v5 构建时自动调用 Bake
docker compose build

# 相当于直接执行
docker buildx bake -f docker-compose.yml
```

## 基础用法

### 安装与验证

```bash
# Docker Bake 内置于 docker buildx，无需单独安装
docker buildx bake --version

# 或指定 bake 文件
docker buildx bake -f bake.hcl
```

### 文件格式对比

Bake 支持三种配置文件格式：

| 格式 | 扩展名 | 适用场景 |
|------|--------|---------|
| HCL | `docker-bake.hcl` | 最灵活，支持变量/函数/动态目标 |
| JSON | `docker-bake.json` | CI/CD 动态生成配置 |
| Compose | `docker-compose.yml` | Compose 项目直接使用 |

### HCL 基础配置

```hcl
# docker-bake.hcl
variable "TAG" {
  default = "latest"
}

variable "REGISTRY" {
  default = "registry.example.com"
}

group "default" {
  targets = ["order-service", "payment-service"]
}

target "order-service" {
  dockerfile = "Dockerfile"
  context    = "./order-service"
  tags       = ["${REGISTRY}/order-service:${TAG}"]
  args = {
    APP_VERSION = "${TAG}"
  }
}

target "payment-service" {
  dockerfile = "Dockerfile"
  context    = "./payment-service"
  tags       = ["${REGISTRY}/payment-service:${TAG}"]
}
```

```bash
# 构建所有服务
docker buildx bake

# 指定 tag
docker buildx bake --set TAG=v1.2.0

# 只构建特定服务
docker buildx bake payment-service
```

## 多架构构建

### 创建构建器实例

```bash
# 创建支持多架构的构建器
docker buildx create --name multiarch --driver docker-container --bootstrap

# 使用该构建器
docker buildx use multiarch

# 查看支持的平台
docker buildx inspect --bootstrap
# 输出: linux/amd64, linux/arm64, linux/arm/v7
```

### Bake 多架构配置

```hcl
# docker-bake.hcl - 多架构构建
variable "REGISTRY" {
  default = "registry.example.com"
}

variable "TAG" {
  default = "latest"
}

# 定义平台矩阵
target "platforms" {
  platforms = ["linux/amd64", "linux/arm64"]
}

target "spring-app" {
  inherits  = ["platforms"]
  dockerfile = "Dockerfile"
  context    = "."
  tags       = [
    "${REGISTRY}/spring-app:${TAG}",
    "${REGISTRY}/spring-app:${TAG}-amd64",
    "${REGISTRY}/spring-app:${TAG}-arm64"
  ]
  cache-from = ["type=gha"]
  cache-to   = ["type=gha,mode=max"]
}
```

```bash
# 同时构建两个架构
docker buildx bake spring-app

# 等同于分别构建 amd64 和 arm64，然后合并为 manifest
```

### Spring Boot Dockerfile 多阶段构建

```dockerfile
# Dockerfile - 适合多架构的 Spring Boot 构建
# Stage 1: 构建
FROM --platform=$BUILDPLATFORM eclipse-temurin:21-jdk-alpine AS builder
ARG TARGETPLATFORM
ARG BUILDPLATFORM
WORKDIR /build
COPY . .
RUN --mount=type=cache,target=/root/.gradle \
    ./gradlew bootJar -x test

# Stage 2: 运行
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
COPY --from=builder /build/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## CI/CD 集成

### GitHub Actions

```yaml
# .github/workflows/docker-build.yml
name: Docker Multi-Arch Build

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  bake:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ secrets.REGISTRY }}
          username: ${{ secrets.REGISTRY_USER }}
          password: ${{ secrets.REGISTRY_PASS }}

      - name: Extract metadata
        id: meta
        run: |
          echo "TAG=${GITHUB_REF_NAME#v}" >> $GITHUB_ENV

      - name: Build and Push
        uses: docker/bake-action@v6
        with:
          files: docker-bake.hcl
          set: |
            *.tags=${{ env.REGISTRY }}/app:${TAG}
            *.cache-from=type=gha
            *.cache-to=type=gha,mode=max
          push: true
```

### GitLab CI

```yaml
# .gitlab-ci.yml
docker-build:
  image: docker:27
  services:
    - docker:27-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker buildx create --use --driver docker-container
  script:
    - docker buildx bake -f docker-bake.hcl
        --set *.tags=$CI_REGISTRY_IMAGE:$CI_COMMIT_TAG
        --push
  only:
    - tags
```

## 高级模式

### 构建缓存优化

```hcl
# docker-bake.hcl - 缓存配置
target "_cache" {
  cache-from = [
    "type=gha",          # GitHub Actions 缓存
    "type=registry,ref=registry.example.com/app:cache"  # registry 缓存
  ]
  cache-to = [
    "type=gha,mode=max",
    "type=registry,ref=registry.example.com/app:cache,mode=max"
  ]
}

target "app" {
  inherits = ["_cache"]
  # ...
}
```

### 条件构建（不同环境）

```hcl
# docker-bake.hcl - 环境矩阵
variable "ENV" {
  default = "dev"
}

target "base" {
  dockerfile = "Dockerfile"
  context    = "."
  args = {
    SPRING_PROFILES_ACTIVE = "${ENV}"
  }
}

target "dev" {
  inherits = ["base"]
  tags = ["app:dev"]
}

target "prod" {
  inherits = ["base"]
  tags = [
    "registry.example.com/app:${ENV}",
    "registry.example.com/app:${ENV}-${version}"
  ]
}

# 选择目标
group "default" {
  targets = [ENV == "prod" ? "prod" : "dev"]
}
```

### Spring Boot 微服务矩阵

```hcl
# docker-bake.hcl - 微服务批量构建
variable "REGISTRY" { default = "registry.example.com" }
variable "TAG" { default = "latest" }

# 服务列表
services = ["gateway", "order-service", "payment-service", "user-service"]

# 动态生成目标
target "microservices" {
  matrix = {
    service = services
  }
  name    = "${matrix.service}"
  dockerfile = "Dockerfile"
  context    = "./${matrix.service}"
  tags       = ["${REGISTRY}/${matrix.service}:${TAG}"]
  platforms  = ["linux/amd64", "linux/arm64"]
}
```

```bash
# 构建所有微服务（并行4个目标 x 2架构 = 8个构建）
docker buildx bake microservices

# 查看要执行的任务
docker buildx bake microservices --print
# 输出 JSON 格式的构建计划
```

## 与 Compose v5 配合

```yaml
# docker-compose.yml (v5 格式)
services:
  order-service:
    build:
      context: ./order-service
      tags:
        - order-service:latest
    ports:
      - "8080:8080"

  payment-service:
    build:
      context: ./payment-service
      tags:
        - payment-service:latest

# Bake 配置可以内嵌在 Compose 文件的 x-bake 扩展字段
x-bake:
  targets:
    order-service:
      platforms: ["linux/amd64", "linux/arm64"]
      cache-from: ["type=gha"]
    payment-service:
      platforms: ["linux/amd64", "linux/arm64"]
```

```bash
# 使用 Compose v5 构建（自动调用 Bake）
docker compose build

# 或直接调用 Bake
docker buildx bake -f docker-compose.yml
```

## 性能对比

| 构建方式 | 单架构 | 双架构 | 缓存命中 | CI 耗时 |
|---------|--------|--------|---------|--------|
| `docker build` | 45s | N/A | 是 | 2min |
| `docker buildx build` | 50s | 90s | 是 | 3min |
| `docker buildx bake` (矩阵) | 45s | 55s* | 是（共享缓存） | 1.5min |

> *Bake 在多架构构建时并行执行，总时间约等于最慢架构的构建时间，而非累加。

## 注意事项

1. **Bake 需要 BuildKit**：确保 `DOCKER_BUILDKIT=1` 环境变量已设置（Docker 23+ 默认启用）
2. **多架构需 Container Driver**：`docker buildx create --driver docker-container`，默认的 `docker` 驱动不支持跨平台构建
3. **HCL 变量作用域**：`variable` 块定义的变量可通过 `-set` 或环境变量覆盖，优先级：CLI set > 环境变量 > HCL 默认值
4. **缓存后端选择**：
   - GitHub Actions: `type=gha`
   - GitLab CI: `type=registry`
   - 本地开发: `type=local`
5. **`--print` 调试**：使用 `docker buildx bake --print` 可查看解析后的完整 JSON 构建计划，便于调试
6. **与 Compose v5 共存**：如果同时定义了 `docker-compose.yml` 和 `docker-bake.hcl`，优先使用 HCL 文件
