---
name: docker-296-features
description: Docker Engine 29.6.0 新特性：镜像 SLSA/SPDX 证明 API、per-device blkio、nftables 无 nft 命令、BuildKit 0.31.0
tags: [tools, docker, security, attestation, buildkit, slsa, sbom, nftables]
---

## 概述

Docker Engine 29.6.0（2026-06-18 发布）带来了多项重要更新，包括**镜像证明（Attestation）API**、**per-device blkio 资源控制**、**nftables 无 `nft` 命令支持**以及内置 **BuildKit v0.31.0**。

## 1. 镜像 Attestation API

新增 `GET /images/{name}/attestations` 端点，用于获取镜像的 **in-toto attestation** 证明（如 SLSA provenance 和 SPDX SBOM）。

```bash
# 获取镜像的 attestation 清单
curl --unix-socket /var/run/docker.sock \
  http://localhost/images/myapp:latest/attestations

# 支持平台选择与 predicate 类型过滤
curl --unix-socket /var/run/docker.sock \
  "http://localhost/images/myapp:latest/attestations?platform=linux/amd64&predicateType=https://spdx.dev/Document"
```

**用途**：CI/CD 流水线中可在镜像推送后直接通过 API 验证镜像的供应链安全元数据，无需额外工具。

## 2. per-device blkio

`POST /containers/{id}/update` 现在支持按设备设置 blkio 限制：

```bash
# 只限制 /dev/sda 的 IO
docker update --blkio-weight-device /dev/sda:300 mycontainer

# API 层面也支持
curl -X POST --unix-socket /var/run/docker.sock \
  http://localhost/containers/mycontainer/update \
  -H "Content-Type: application/json" \
  -d '{"BlkioWeightDevice":[{"Path":"/dev/sda","Weight":300}]}'
```

**适用场景**：多磁盘环境中对特定数据盘进行 IO 隔离，避免一个容器的 IO 风暴影响其他容器。

## 3. nftables 改进

当守护进程链接了 `libnftables` 库时，**无需系统安装 `nft` 命令**即可使用 nftables 防火墙模式。

```json
{
  "iptables": false,
  "nftables": true
}
```

将上述配置添加到 `/etc/docker/daemon.json` 即可启用。适用于容器运行时环境最小化、不安装 `nft` 命令的场景。

## 4. docker login --password STDIN 增强

`--password` 标志现在可通过 `-` 从 STDIN 读取密码，作为 `--password-stdin` 的替代方案：

```bash
# 两种等效方式
echo "mytoken" | docker login --password-stdin registry.example.com
echo "mytoken" | docker login --password - registry.example.com
```

## 5. 其他 Bug 修复

| 修复项 | 说明 |
|--------|------|
| `COPY --chmod` 权限修复 | 显式文件模式不再被 daemon umask 过滤 |
| `docker system prune` | containerd 镜像存储模式正确报告回收空间 |
| `docker system df` | 镜像大小只计算直接使用的快照 |
| 注册表认证失败提示 | 修复误导性的 "No such image" 错误信息 |
| BuildKit GC 策略 | 修复可重现缓存类型的默认 GC 未清理问题 |
| 端口分配 | 不将容器端口发布到 `ip_local_reserved_ports` 范围 |
| 网络 DNS | 修复 overlay 网络 bulk sync 的竞态条件（~30s DNS 异常）|

## 6. 内置 BuildKit v0.31.0

Docker 29.6.0 捆绑了 **BuildKit v0.31.0**（2026-06-17 发布），核心变化：

### 网络代理（Exec Proxy）
构建步骤的所有容器流量可通过 HTTP 代理服务器路由，用于网络流量捕获、审计和策略执行。Source Policy 可定义允许/禁止的请求。

### per-step 资源限制
LLB API 支持对每个构建步骤设置 CPU 和内存限制：

```dockerfile
# 通过 BuildKit 配置限制单个构建步骤资源
# 需要在 BuildKit 配置中启用
```

### OCI 媒体类型默认
所有镜像结果默认使用 OCI 媒体类型（不再仅在有 annotation/attestation 时使用）。向后兼容可通过 `oci-mediatypes=false` 回退。

### 其他
- Dockerfile 前端更新至 v1.25.0
- 本地缓存导出支持 `reset` 选项清除未引用缓存
- 构建指标（次数、时长）通过 OTEL provider 上报
- QEMU 升级至 v10.2.3（嵌入式 binfmt 模拟器）
- runc 升级至 v1.3.6
- attestation 使用 in-toto v1 statement 格式

## 注意事项

1. **Attestation API** 需要镜像使用 BuildKit 构建且在构建时生成了 attestation（`--attest type=provenance,type=sbom`），旧镜像不会自动有 attestation 数据。
2. **nftables 无 nft 模式**需要 Docker daemon 编译时链接 libnftables（静态构建的二进制可能不带此支持）。
3. **OCI 媒体类型默认**可能影响与旧版镜像仓库的兼容性——如果仓库不支持 OCI manifest，需显式设置 `oci-mediatypes=false`。
4. BuildKit v0.31.0 的兼容性版本提升至 30，旧版 BuildKit 客户端可能无法使用新特性。
