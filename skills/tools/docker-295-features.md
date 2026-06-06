---
name: docker-295-features
description: Docker Engine 29.5 新特性：gvisor-tap-vsock 根less默认驱动、私有时间命名空间、HealthStatus占位符
tags: [tools, docker, rootless, security, container, networking]
---

## 概述

Docker Engine 29.5.0（2026-05-14 发布）引入了多项重要特性更新。后续 29.5.1 修复了 2 个高危 CVE（docker cp 权限提升），29.5.2 修复了 docker cp 符号链接遍历问题，29.5.3 更新了依赖并修复了 Rootless 模式。

> 当前最新：**Docker v29.5.3**（2026-06-03 发布）

## 主要新特性

### 1. Rootless: gvisor-tap-vsock 成为默认网络驱动

**变化**：rootless 模式下，`gvisor-tap-vsock` 取代 `slirp4netns` 成为**默认网络驱动**。

```bash
# 无需任何配置，rootless 安装后自动使用新驱动
dockerd-rootless-setuptool.sh install

# 验证当前使用的网络驱动
docker info | grep -A5 "Rootless"
```

**影响**：
- `slirp4netns` 不再默认安装，需手动安装才能使用
- `gvisor-tap-vsock` 性能更好，延迟更低
- 如果有依赖 `slirp4netns` 的自定义配置，需要显式指定

```bash
# 如需回退到 slirp4netns
dockerd-rootless-setuptool.sh install --driver slirp4netns
```

### 2. 私有时间命名空间默认启用

从 29.5.0 开始，Docker 在支持的内核上**默认启用容器的私有时间命名空间**（time namespace）。

```bash
# 容器内看到的时间是容器启动时的 monotonic clock
# 而非宿主机的系统时间

# 如需禁用
docker run --security-opt=no-new-privileges --security-opt=time-namespace=disabled ...
```

**优势**：
- 防止容器内进程感知宿主机运行时长
- 增强容器隔离性
- 可通过 `"time-namespaces"` feature flag 关闭

### 3. `docker ps --format` 支持 `.HealthStatus`

新的格式化占位符，可直接打印容器健康状态：

```bash
# 查看所有容器的健康状态
docker ps --format "table {{.Names}}\t{{.HealthStatus}}"

# 仅列出健康检查失败的容器
docker ps --filter "health=unhealthy" --format "{{.Names}}: {{.HealthStatus}}"
```

**输出值**：`starting` / `healthy` / `unhealthy` / `none`（未设置健康检查）

### 4. local 日志驱动支持自定义属性

`local` 日志驱动现在支持 `label`、`label-regex`、`env`、`env-regex`、`tag` 等选项：

```yaml
# docker-compose.yml
services:
  app:
    logging:
      driver: local
      options:
        labels: "env,app"
        env: "SERVICE_VERSION"
        tag: "{{.Name}}/{{.ID}}"
```

### 5. Windows: Unix Socket 支持

Windows 版 Docker 守护进程现在支持监听 Unix socket：

```powershell
# Windows 上启动 Docker 监听 Unix socket
dockerd -H unix:///var/run/docker.sock --group docker-users
```

## 安全修复

### CVE-2026-32288 — 恶意镜像拒绝服务（29.5.0）

拉取特制镜像时，守护进程在处理镜像层时可能**分配无限内存**导致 DoS。

```bash
# 升级后即可防护
docker version | grep "Version: 29.5"
```

### CVE-2026-41567 — docker cp 路径遍历权限提升（29.5.1）

`docker cp` 的归档解压二进制（如 `xz`、`unpigz`）通过容器文件系统的 `PATH` 查找并以宿主 root 权限执行，恶意容器可借此**以宿主 root 权限执行任意代码**。

```bash
# 危害等级：CRITICAL
# 攻击路径：恶意容器 → 在容器内放置特制 xz 二进制 → docker cp 触发解压 → 宿主 root 执行
```

### CVE-2026-41568 — docker cp TOCTOU 任意文件写入（29.5.1）

`docker cp` 存在 Time-of-Check/Time-of-Use 漏洞，容器内进程可在检查与使用的时间窗口内创建**宿主机任意路径的文件或目录**。

```bash
# 修复版本：Docker Engine 29.5.1+
# 升级命令
sudo apt-get install --only-upgrade docker-ce=5:29.5.3-1~ubuntu.24.04~noble
```

## 其他增强

| 功能 | 说明 |
|------|------|
| `docker system df -v` | 修复共享 blob 的 SIZE 计算，现在包含共享内容块 |
| Swarm Raft 快照 | 防止大状态下的快照损坏 |
| CDI 扩展 | 修复需要额外 group ID 的 CDI 规范支持 |
| 卷子路径挂载 | 修复文件覆盖已存在文件时的 "not a directory" 错误 |
| auth token | 修复 containerd 集成中忽略 per-host TLS 设置的问题 |
| 用户代理代理诊断 | `docker info` 可查看用户代理代理的诊断数据 |
| docker cp symlink（29.5.2） | 修复 bind mount 目标穿越容器内符号链接时 "mkdirat: file exists" 错误 |
| docker system df（29.5.3） | 减少 containerd 镜像存储下并发镜像清理时的错误 |
| Rootless 插件安装（29.5.3） | 修复需要 host 网络的插件安装失败问题 |
| Rootless AWS IMDS（29.5.3） | 修复 gvisor-tap-vsock + UDP 端口转发中非 loopback 客户端访问问题 |

## 升级注意事项

```bash
# Ubuntu/Debian
sudo apt-get update && sudo apt-get install docker-ce=5:29.5.3-1~ubuntu.24.04~noble

# CentOS/RHEL
sudo yum install docker-ce-29.5.3

# 验证
docker version --format '{{.Server.Version}}'
# 29.5.3
```

## Docker Compose 5.1.4 注意事项

Docker Compose v5.1.4（2026-05-20）配合 Docker 29.5.x 使用，新增：

- **Stop lifecycle hook**：外部 Provider 支持停止生命周期钩子
- **OCI artifact proxy**：修复 OCI 拉取通过 Docker Desktop HTTP 代理的路由问题

```bash
# 升级 Docker Compose
docker compose version
# Docker Compose version v5.1.4
```

## 参考

- [Docker 29.5.0 Release Notes](https://github.com/moby/moby/releases/tag/docker-v29.5.0)
- [Docker 29.5.1 Security Release](https://github.com/moby/moby/releases/tag/docker-v29.5.1)
- [Docker 29.5.3 Release Notes](https://github.com/moby/moby/releases/tag/docker-v29.5.3)
- [GHSA-x86f-5xw2-fm2r (CVE-2026-41567)](https://github.com/moby/moby/security/advisories/GHSA-x86f-5xw2-fm2r)
- [Rootless mode docs](https://docs.docker.com/engine/security/rootless/)
