---
name: docker-compose-v5-migration
description: Docker Compose v5.x 重大变更：SDK 化、构建委派 Docker Bake、OCI/Git 远程资源、版本跳跃 v2→v5
tags: [tools, docker, docker-compose, docker-bake, migration, devops]
---

## 概述

Docker Compose v5.0.0（2026-05 发布）是 Compose 的一个**重大版本跳跃**（v2 → v5，跳过了 v3/v4）。主要变更包括：

- **版本编号跳跃**：跳过 3.x 和 4.x，避免与已废弃的旧 `docker-compose.yml` 文件版本号（2.x/3.x）混淆
- **内置 BuildKit 构建器移除**：构建委托给 Docker Bake（即 `docker build`）
- **Compose SDK 化**：可作为 Go SDK 被第三方集成
- **OCI/Git 远程资源**：直接引用 OCI 制品和 Git 仓库
- **重启钩子**：支持服务重启生命周期钩子

> ⚠️ v5.x 仅支持从 v2.40.x 升级，不支持从 Docker Compose v1 直接升级。

## 一、版本跳跃说明

### 版本号对应关系

| Compose CLI 版本 | 说明 |
|-----------------|------|
| **v1.x** | 旧版 Python 实现（已淘汰） |
| **v2.x** | 新版 Go 实现（v2.0~v2.40） |
| **v5.0+** | 当前主线，跳过了 3.x/4.x |

> 注意：`docker-compose.yml` 文件格式的 `version: "3.x"` 字段**已于 2023 年淘汰**，与 Compose CLI 版本号无关。v5.x 继续使用无版本声明的 Compose Specification 格式。

```yaml
# ✅ v5.x 正确写法（无需 version 字段）
services:
  app:
    image: nginx:alpine

# ❌ 不要加过时的 version: "3.8"
```

### 升级检查

```bash
# 检查当前版本
docker compose version  # v2.40.2 → 可升级到 v5

# 确认 compose 文件格式合规（无 version 字段、无已弃用配置）
docker compose config
```

## 二、构建委派给 Docker Bake

### 核心变化

v5.x 移除了内置的 BuildKit 构建器，所有构建操作统一委托给 **Docker Bake**（即 `docker buildx bake` 或 `docker build`）。

**影响**：
- 现有 `build` 配置**无需改动**，自动兼容
- 构建行为与 `docker build` 完全一致
- 之前通过 Compose 特有构建参数获得的某些行为可能变化

```yaml
# ✅ 原有 build 配置完全兼容
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        BUILD_VERSION: "1.0.0"
      labels:
        org.opencontainers.image.source: "https://github.com/example/app"
```

### 构建缓存控制

```yaml
services:
  app:
    build:
      context: .
      # 跳过特定阶段的缓存（v5.0+）
      no_cache_filter:
        - "copy-packages"
        - "install-deps"
```

### 使用 Docker Bake 直接构建

```bash
# v5.x 内部等同于：
docker buildx bake -f docker-compose.yml

# 或直接使用 bake 文件
docker buildx bake -f docker-bake.hcl
```

## 三、OCI / Git 远程资源

### 从 OCI 制品引用 Compose 文件

v5.x 支持直接从 **OCI 容器镜像仓库**拉取 Compose 文件：

```bash
# 从 Docker Hub / Harbor / ECR 引用 OCI 制品
docker compose -f oci://registry.example.com/myapp/compose:v1.0.0 up

# 简写格式（默认 Docker Hub）
docker compose -f oci://myorg/myapp-compose:v1.0.0 up
```

### 从 Git 仓库引用 Compose 文件

直接引用 Git 仓库中的 Compose 文件：

```bash
# 语法：git://<repo-url>[#<ref>]:<path>
docker compose -f git://github.com/myorg/myapp.git#main:deploy/compose.yml up

# 也支持 SSH 协议
docker compose -f git+ssh://git@github.com/myorg/myapp.git#v1.0:docker-compose.yml up
```

**适用场景**：
- CI/CD 管道中直接引用指定版本的部署配置
- 多团队共享部署模板
- 不克隆仓库即可启动远程服务

## 四、Compose SDK

### 概述

v5.x 将 Compose 的核心能力抽取为 **Go SDK**，允许第三方工具直接集成 Compose 的服务编排能力，无需调用 Docker CLI。

### SDK 使用示例（Go）

```go
package main

import (
    "context"
    "github.com/docker/compose/v5/pkg/compose"
    "github.com/docker/compose/v5/pkg/api"
)

func main() {
    ctx := context.Background()

    // 创建 Compose 服务实例
    svc, err := compose.New(ctx, compose.WithProjectName("myapp"))
    if err != nil {
        panic(err)
    }

    // 加载 Compose 文件
    project, err := svc.Load(ctx, []string{"docker-compose.yml"}, nil)
    if err != nil {
        panic(err)
    }

    // 启动服务
    _, err = svc.Up(ctx, project, api.UpOptions{})
    if err != nil {
        panic(err)
    }
}
```

**适用场景**：
- CI/CD 工具集成容器编排
- IDE/Debug 工具一键启动开发环境
- 监控工具集成 Compose 服务生命周期

## 五、重启钩子（Restart Hooks）

v5.x 支持在服务重启时执行自定义钩子：

```yaml
services:
  app:
    image: myapp:latest
    develop:
      hooks:
        # 重启前执行
        pre_restart:
          - command: echo "Service about to restart"
        # 重启后执行
        post_restart:
          - command: curl -X POST http://healthcheck/notify
```

> 重启钩子目前主要用于开发场景（watch 模式下的重启），生产环境建议使用容器健康检查和 orchestration 级别的生命周期管理。

## 六、其他新特性

### `docker compose start --wait`

```bash
# 启动服务并等待所有服务通过健康检查
docker compose start --wait

# 设置超时（默认 60s）
docker compose start --wait --wait-timeout 120
```

### `build.no_cache_filter`

```yaml
services:
  app:
    build:
      context: .
      # 只对特定构建阶段禁用缓存，其他阶段使用缓存
      no_cache_filter:
        - "download-deps"  # 这个阶段总是重新执行
```

### `--insecure-registry`

```bash
# 测试用：允许使用自签名证书的镜像仓库
docker compose pull --insecure-registry registry.internal:5000
```

## 升级指南

### 从 v2.40.x 升级

```bash
# 1. 下载 v5.x
sudo curl -L "https://github.com/docker/compose/releases/download/v5.0.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# 2. 检查 compose 文件兼容性
docker compose config

# 3. 常见不兼容项
#    - 移除已弃用的 depends_on 旧语法（推荐使用 healthcheck 条件）
#    - 确保没有使用已移除的 --compatibility 标志
#    - 检查自定义构建脚本（构建行为统一到 docker build）
```

### 兼容性检查清单

```bash
# ❌ 以下内容在 v5.x 中可能有问题

# 1. 使用了 --compatibility 标志
docker compose --compatibility up  # v5.x 不再支持

# 2. 依赖了旧版 BuildKit 构建器特有的行为
# 解决方案：使用 docker-bake.hcl 替代

# 3. 直接调用内部构建 API
# 解决方案：改用 docker build / Docker Bake
```

## 注意事项

1. **v5.x 不可降级**：升级后配置文件可能被更新，v2.x 无法读取
2. **构建行为变化**：如果使用复杂的多阶段构建或自定义 BuildKit 配置，建议在测试环境验证
3. **CI/CD 更新**：CI 管道的 Docker Compose 安装脚本需要更新到 v5.x
4. **仅支持 v2.40+ 升级**：如果仍在 v2.23 以下，需先升级到 v2.40
5. **OCI/Git 远程资源需要网络**：离线环境无法使用此功能

## 参考链接

- [Docker Compose v5.0.0 Release Notes](https://github.com/docker/compose/releases/tag/v5.0.0)
- [Docker Compose Specification](https://compose-spec.io/)
- [Docker Bake Documentation](https://docs.docker.com/build/bake/)
