---
name: docker-buildx-034-features
description: Docker Buildx v0.34 新特性：默认Source Policy、K8s StatefulSet驱动、Bake --policy标志
tags: [tools, docker, buildx, buildkit, kubernetes, devops]
---

## 概述

Docker Buildx v0.34.0（2026-05-13 发布）和 v0.34.1（2026-05-19 发布）引入了多项重要功能增强，特别是**默认 Source Policy**、**Kubernetes StatefulSet 驱动支持**以及 **Bake --policy** 标志。

> 当前最新：**Buildx v0.34.1**

## 主要新特性

### 1. 默认 Source Policy

Buildx 现在支持**默认 Source Policy**，用于常见的构建流水线镜像。这些镜像由 Docker Inc. 提供并使用 **Docker GitHub builder** 签名。

```bash
# Source Policy 自动生效，无需额外配置
docker buildx build --push -t myapp:latest .

# 查看当前生效的策略
docker buildx inspect --bootstrap
```

**效果**：
- 构建流水线镜像（如代码仓库镜像）默认使用经过验证的来源
- 防止供应链攻击，确保使用的镜像来自可信源
- 自动签名的构建器增强安全性

### 2. Bake 命令新增 --policy 标志

`bake` 命令新增 `--policy` 标志，用于指定全局策略评估选项：

```bash
# 使用策略文件评估 bake 配置
docker buildx bake --policy policy.json

# 结合默认策略使用
docker buildx bake --policy default
```

**policy.json 示例**：
```json
{
  "targets": {
    "default": {
      "policy": {
        "source-policy": "docker-default",
        "platforms": ["linux/amd64", "linux/arm64"]
      }
    }
  }
}
```

### 3. Kubernetes 驱动 — StatefulSet + PVC 持久化存储支持

K8s 驱动现在支持**持久化存储选项**，部署定义为 **StatefulSet** + **Persistent Volume Claim**。适合需要持久化构建缓存的场景。

```bash
# 创建使用持久化存储的 K8s builder
docker buildx create \
  --name k8s-builder \
  --driver kubernetes \
  --driver-opt namespace=buildkit \
  --driver-opt persistent-storage=true \
  --driver-opt storage-class=standard \
  --driver-opt storage-size=10Gi

# 使用
docker buildx use k8s-builder
docker buildx build --push -t registry/myapp:latest .
```

**配置选项**：

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `persistent-storage` | 启用持久化存储 | `false` |
| `storage-class` | PVC 使用的 StorageClass | 集群默认 |
| `storage-size` | PVC 大小 | `10Gi` |

**工作原理**：
- 构建器部署为 StatefulSet（而非 Deployment），保证稳定的 Pod 身份
- 每个 Pod 挂载独立的 PVC，保留构建缓存
- 适合 GitOps / CI 流水线中的重复构建场景

### 4. 其他修复与增强

- **Windows 路径处理**：修复 OCI layout 定义中的 Windows 路径问题
- **Extra hosts 排序**：修复由于非确定性排序导致的缓存未命中
- **GPU 设备挂载**：修复 WSL 库挂载只在本地 docker-container 端点生效
- **Progress Policy 错误**：修复进度策略错误在输出中丢失的问题
- **`dial-stdio` 修复**：修复构建器连接关闭时的停止命令问题

## 使用示例

### 安装与升级

```bash
# 通过 Docker 内置
docker buildx version  # 内置版本

# 独立安装最新版
# https://github.com/docker/buildx/releases/tag/v0.34.1
```

### 结合 Compose v5 使用 Bake

```bash
# Compose v5 已将构建委托给 Bake
docker compose build --bake

# 结合政策使用
docker compose build --bake --policy production.json
```

## 注意事项

1. **Source Policy 自动生效**：如果依赖自定义或第三方构建器，注意验证策略是否符合预期
2. **K8s StatefulSet**：删除 builder 后 PVC 不会自动清理，需手动删除
3. **Windows 兼容性**：v0.34 修复了 Windows OCI layout 路径问题，Windows 用户建议升级

## 参考

- [Buildx v0.34.0 Release](https://github.com/docker/buildx/releases/tag/v0.34.0)
- [Buildx v0.34.1 Release](https://github.com/docker/buildx/releases/tag/v0.34.1)
- [Source Policy 文档](https://docs.docker.com/build/bake/policy/)
