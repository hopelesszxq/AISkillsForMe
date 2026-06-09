---
name: docker-debug-ephemeral
description: Docker Debug 与临时容器调试：docker debug 命令、ephemeral containers、非侵入式容器排障
tags: [tools, docker, debug, ephemeral, container, troubleshooting, kubectl-debug]
---

## 概述

传统容器调试需要 `docker exec -it` 进入容器，但生产环境容器通常**没有 shell 和调试工具**（镜像最小化原则）。Docker Debug 和 Kubernetes Ephemeral Containers 提供了**非侵入式调试**方案：向运行中的容器注入独立的调试容器，共享目标容器的命名空间。

## 一、Docker Debug 命令

`docker debug` 是 Docker CLI 内置的调试命令（Docker 4.x+），自动在目标容器旁启动一个包含调试工具的临时容器。

### 1.1 基本用法

```bash
# 调试运行中的容器
docker debug my-container

# 指定调试镜像（默认使用 busybox）
docker debug --image=nicolaka/netshoot my-container

# 不自动附加，只创建调试容器
docker debug --detach my-container

# 指定 shell
docker debug --shell=/bin/bash my-container
```

### 1.2 实操示例

```bash
# 启动一个最小化的 Nginx 容器（无 curl、无 ping、无 bash）
docker run -d --name web nginx:alpine

# 用 docker debug 注入调试容器
docker debug web

# 在调试容器中可以：
/# curl http://localhost:80/health    # 检查 HTTP 端点
/# ping upstream-service               # 网络连通性
/# ss -tlnp                             # 检查端口监听
/# nslookup db.internal                # DNS 解析
/# cat /proc/1/root/etc/nginx/conf.d/default.conf  # 查看宿容器文件

# 退出调试容器（自动清理）
/# exit
```

### 1.3 调试容器的特性

| 特性 | 说明 |
|------|------|
| **共享命名空间** | PID、Network、Mount、IPC、UTS 均共享 |
| **独立文件系统** | 调试工具不会污染目标容器 |
| **自动清理** | 退出调试会话后自动删除 |
| **root 权限** | 默认以 root 运行，可访问宿容器所有进程 |
| **网络工具** | 内置 curl、wget、ping、nslookup、dig 等 |

## 二、Kubernetes Ephemeral Containers

K8s 1.25+ 稳定支持 Ephemeral Containers（临时容器），用于调试运行中的 Pod。

### 2.1 基本用法

```bash
# 调试 Pod — 注入临时容器
kubectl debug my-pod -it --image=nicolaka/netshoot

# 调试特定容器（多容器 Pod）
kubectl debug my-pod -c my-container -it --image=nicolaka/netshoot

# 复制 Pod 并调试副本（不会影响原 Pod）
kubectl debug my-pod -it --image=nicolaka/netshoot --copy-to=my-pod-debug

# 指定节点调试（节点级别排障）
kubectl debug node/my-node -it --image=nicolaka/netshoot
```

### 2.2 生产环境安全调试

```bash
# 场景：Pod 处于 CrashLoopBackOff，无法 exec
# 方案：复制 Pod 并启动调试容器
kubectl debug my-pod --copy-to=my-pod-debug \
    --image=nicolaka/netshoot \
    --share-processes \
    --set-image=my-app=my-image:latest

# 调试启动失败的容器
kubectl debug my-pod -it \
    --image=busybox:latest \
    --target=my-container  # 只共享目标容器的命名空间
```

## 三、常见的容器调试场景

### 3.1 网络排障

```bash
# 场景：服务间调用超时
docker debug --image=nicolaka/netshoot my-service

# 网络诊断命令
/# curl -v http://upstream:8080/api/health    # HTTP 探测
/# ping -c 5 upstream                          # ICMP 连通性
/# mtr upstream                                # 路由追踪
/# dig +short upstream                         # DNS 解析
/# ss -tlnp                                    # 本地端口监听
/# tcpdump -i eth0 port 8080                   # 抓包分析
```

### 3.2 文件系统排障

```bash
# 场景：容器内文件丢失或权限错误
docker debug my-app

/# ls -la /app/config/                          # 检查文件是否存在
/# cat /proc/1/root/app/config/application.yml  # 查看容器文件
/# stat /app/data                               # 检查挂载
/# mount                                        # 查看挂载点
/# df -h                                        # 磁盘空间
/# du -sh /app/logs                             # 日志大小
```

### 3.3 进程和资源排障

```bash
# 场景：容器内存泄漏或 CPU 高
docker debug --image=nicolaka/netshoot my-app

/# top -p 1                                     # 目标进程资源
/# ps aux                                       # 所有进程
/# cat /proc/1/status                           # 进程状态
/# ls /proc/1/fd/                               # 文件描述符
/# strace -p 1                                  # 系统调用追踪
/# lsof -p 1                                    # 打开文件
```

## 四、推荐的调试镜像

| 镜像 | 大小 | 内置工具 | 适用场景 |
|------|------|----------|----------|
| `nicolaka/netshoot` | ~400MB | curl, wget, dig, nslookup, ping, tcpdump, mtr, iperf, ss | 网络排障首选 |
| `busybox:latest` | ~5MB | sh, ls, cat, ps, ping, wget | 最小化调试 |
| `alpine:latest` | ~30MB | apk 可安装任意工具 | 通用调试 |
| `ubuntu:latest` | ~80MB | apt 安装，兼容性最好 | 复杂排障 |
| `docker:cli` | ~200MB | docker CLI 本身 | 容器内 docker 操作 |

## 五、CI/CD 中的调试技巧

```yaml
# Docker Compose — 调试模式
services:
  app:
    image: my-app:latest
    # 生产环境不暴露调试端口
    
  debug:
    image: nicolaka/netshoot
    network_mode: "service:app"  # 共享网络
    pid: "service:app"           # 共享 PID
    depends_on:
      - app
```

```yaml
# GitHub Actions — 调试工作流
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        
    steps:
      - name: Debug session
        uses: docker://nicolaka/netshoot
        with:
          args: >
            sh -c "curl -v http://postgres:5432 && nslookup postgres"
```

## 六、注意事项

1. **安全**：调试容器以 root 运行，生产环境应限制谁有 `docker debug` / `kubectl debug` 权限
2. **资源**：调试容器会消耗额外资源，不要长时间保持调试会话
3. **存储驱动**：overlay2 下的 `/proc/1/root` 路径可能受 mount namespace 影响
4. **K8s 版本**：kubectl debug 需要 K8s 1.25+，旧集群可用 `kubectl run --rm -it --image=busybox --restart=Never debug -- sh`
5. **非 root 容器**：如果容器以非 root 运行，`docker debug` 默认使用 root，可加 `--user` 参数
6. **Windows 容器**：`docker debug` 目前仅支持 Linux 容器
