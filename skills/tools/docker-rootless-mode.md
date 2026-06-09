---
name: docker-rootless-mode
description: Docker Rootless 模式完整实战 —— 原理、安装、网络架构、生产部署、常见问题与排错
tags: [tools, docker, rootless, security, container, linux, deployment]
---

## 概述

Docker Rootless 模式允许以**非 root 用户**运行 Docker Engine 和所有容器，大幅降低容器逃逸风险。自 Docker 19.03 引入实验支持，到 29.5+ 已成为**生产可用的安全运行方案**。

核心原理：通过 `rootlesskit` + `slirp4netns`/`gvisor-tap-vsock` 在用户命名空间中模拟网络栈，无需任何真实 root 权限。

> Docker 29.5.0+ 已将 `gvisor-tap-vsock` 设为 rootless 模式的**默认网络驱动**，取代了性能较差的 `slirp4netns`。Docker 29.5.3 修复了 rootless 模式下 AWS IMDS 访问和 UDP 端口转发的问题。

---

## 安装

### 1. 前提条件

```bash
# 内核必须支持用户命名空间
uname -r
# >= 3.8（推荐 5.x+）

# 检查是否开启
cat /proc/sys/kernel/unprivileged_userns_clone
# 如果为 0，需要开启
sudo sysctl -w kernel.unprivileged_userns_clone=1
echo 'kernel.unprivileged_userns_clone=1' | sudo tee -a /etc/sysctl.conf
```

### 2. 一键安装

```bash
# 前提：已安装 docker-ce-cli 但未启动 dockerd
dockerd-rootless-setuptool.sh install

# 查看安装状态
dockerd-rootless-setuptool.sh check
```

安装后自动完成：
- 创建 systemd user service
- 配置 socket 路径 `~/.docker/run/docker.sock`
- 设置环境变量脚本 `~/.docker/config`

### 3. 配置环境变量

```bash
# 添加到 ~/.bashrc 或 ~/.zshrc
export PATH=/usr/bin:$PATH
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
export DOCKER_CONTEXT=rootless

source ~/.bashrc
```

### 4. 启动服务

```bash
# 作为用户 systemd 服务
systemctl --user start docker
systemctl --user enable docker

# 验证
docker info | grep -i rootless
# 输出:  Security Options:  rootless
#       Username: youruser
```

---

## 架构对比

| 维度 | Rootful Docker | Rootless Docker |
|---|---|---|
| daemon 权限 | root | 普通用户 |
| 容器进程 | root (限制) | 普通用户 |
| 默认存储驱动 | overlay2 | overlay2（需内核支持）或 fuse-overlayfs |
| 网络驱动 | bridge (iptables) | slirp4netns / gvisor-tap-vsock |
| 端口映射 | iptables DNAT | rootlesskit 代理 |
| cgroups | 完整支持 | 受限（需 cgroup v2 + systemd） |
| 文件所有权 | root | 用户所有 |

---

## 网络详解

### 默认网络驱动：gvisor-tap-vsock（Docker 29.5+）

```
┌─────────────────┐     VSOCK     ┌──────────────┐
│  容器进程          │◄────────────┤  gVisor 网络  │
│  (用户命名空间)    │              │   tap 设备   │
└─────────────────┘              └──────┬───────┘
                                         │
                                    ┌────▼───────┐
                                    │  rootlesskit │
                                    │  端口代理     │
                                    └────┬───────┘
                                         │
                                    ┌────▼───────┐
                                    │ 宿主机网络    │
                                    └────────────┘
```

### 端口映射原理

```bash
# 容器内部 8080 → 宿主机 8080
docker run -p 8080:8080 nginx

# 实际映射过程：
# gvisor-tap-vsock → rootlesskit → 宿主机 :8080
# 由 rootlesskit 进程在用户空间完成端口转发
```

### 常见网络限制

| 功能 | Rootful | Rootless |
|---|---|---|
| `--privileged` | ✅ | ❌（无实际权限） |
| 原始套接字（ICMP ping） | ✅ | ❌ |
| `--cap-add=NET_ADMIN` | ✅ | ❌ |
| 主机网络模式 `--network=host` | ✅ | ❌ |
| iptables 规则管理 | ✅ | ❌ |
| UDP 端口转发 | ✅ | ✅（29.5.3 修复非回环客户端） |
| IPv6 | ✅ | ⚠️ 部分支持 |

---

## 存储

### 推荐配置：fuse-overlayfs

```bash
# 安装 fuse-overlayfs
sudo apt install fuse-overlayfs

# Docker 会自动检测并使用
docker info | grep "Storage Driver"
# 输出: Storage Driver: fuse-overlayfs
```

### 镜像存储位置

```bash
# rootless 镜像存储在用户目录
~/.local/share/docker/

# rootful 镜像位置
/var/lib/docker/

# 查看实际占用
du -sh ~/.local/share/docker/
```

---

## 生产配置

### 1. 系统资源限制

```bash
# 限制 rootlesskit 进程的打开文件数
mkdir -p ~/.config/systemd/user/docker.service.d/
cat > ~/.config/systemd/user/docker.service.d/limits.conf << EOF
[Service]
LimitNOFILE=1048576
LimitNPROC=1048576
EOF

systemctl --user daemon-reload
systemctl --user restart docker
```

### 2. 日志轮转

```yaml
# ~/.docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "rootless": true,
  "experimental": false
}
```

### 3. 文件描述符监控

```bash
# 查看 rootlesskit 进程的 FD 使用量
PID=$(pidof rootlesskit)
ls /proc/$PID/fd | wc -l
```

### 4. 资源清理

```bash
# rootless 模式下 docker system prune 需要手动确认
docker system prune -af --volumes

# 彻底清理构建缓存
docker builder prune -af
```

---

## 常见问题排查

### Q1: `ping` 命令无法使用
```bash
# rootless 模式下 ICMP 需要特殊配置
# 方法1：使用 socat 转发（不推荐）
# 方法2：改用 HTTP/TCP 健康检查
docker run --health-cmd "curl -f http://localhost:8080/health" ...
```

### Q2: 容器无法访问宿主机 localhost
```bash
# rootless 模式下，宿主机 localhost ≠ 容器视角的宿主机
# 改用 host.docker.internal（仅 Mac/Windows）
# Linux rootless 下需要配置：
docker run --add-host host.docker.internal:$(ip -4 addr show docker0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}') ...
```

### Q3: `docker run --restart=always` 不生效
```bash
# rootless 模式依赖 user systemd
# 改用 systemd 管理
systemctl --user enable docker-container@your-container
```

### Q4: 磁盘空间暴涨
```bash
# rootless 模式下 overlay2 使用用户命名空间，每个层都会复制
# 定期清理：
docker system prune -af --volumes
# 或限制存储：
echo '{"storage-opts": ["overlay2.size=10G"]}' > ~/.docker/daemon.json
```

### Q5: `docker-compose up` 无法绑定低端口
```bash
# rootless 模式不能绑定 <1024 端口
# 解决方案：使用 1024+ 端口 + iptables 转发
# 或使用 authbind/nginx 反向代理
# /etc/authbind/byport/80 授权（不推荐）
```

---

## 安全加固

### 1. 限制 rootlesskit 的能力

```bash
# ~/.config/systemd/user/docker.service.d/security.conf
[Service]
# 禁止 rootlesskit 提权
NoNewPrivileges=yes
# 限制系统调用
SystemCallFilter=@docker @basic-io
```

### 2. 只读根文件系统

```yaml
# ~/.docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "rootless": true,
  "group-add": false
}
```

### 3. 审计日志

```bash
# 监控 rootlesskit 行为
sudo auditctl -a exit,always -F uid=$(id -u) -S execve -k docker-rootless
sudo ausearch -k docker-rootless --start today
```

---

## 升级注意事项

| Docker 版本 | Rootless 变化 |
|---|---|
| < 29.5 | 默认使用 `slirp4netns`，UDP 端口转发不稳定 |
| 29.5.0 | `gvisor-tap-vsock` 成为默认网络驱动，性能提升 3-5x |
| 29.5.1-2 | CVE 修复（`docker cp` 提权） |
| **29.5.3** | 修复 rootless 下 AWS IMDS 访问、UDP 端口转发、插件 host networking 安装问题 |

升级到 29.5+ 可显著改善 rootless 网络性能和稳定性。

---

## 参考链接

- [Docker Rootless 官方文档](https://docs.docker.com/engine/security/rootless/)
- [RootlessKit GitHub](https://github.com/rootless-containers/rootlesskit/)
- [gvisor-tap-vsock](https://github.com/containers/gvisor-tap-vsock)
- [Docker 29.5.3 Release](https://github.com/moby/moby/releases/tag/docker-v29.5.3)
