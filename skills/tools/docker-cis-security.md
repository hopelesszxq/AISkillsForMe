---
name: docker-cis-security
description: Docker CIS 安全基准 v1.6.0：主机配置、守护进程加固、容器运行时安全
tags: [tools, docker, security, cis-benchmark, container]
---

## 概述

Docker Bench for Security 是基于 CIS Docker Benchmark v1.6.0 的自动化安全审计脚本。覆盖 5 大检查领域：主机配置、Docker 守护进程配置、容器镜像安全、容器运行时安全和 Docker Swarm 配置。

### 运行方式

```sh
# 方式一：直接运行
git clone https://github.com/docker/docker-bench-security.git
cd docker-bench-security
sudo sh docker-bench-security.sh

# 方式二：Docker 运行
docker run --rm --net host --pid host --userns host --cap-add audit_control \
    -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
    -v /etc:/etc:ro \
    -v /usr/bin/containerd:/usr/bin/containerd:ro \
    -v /usr/bin/runc:/usr/bin/runc:ro \
    -v /usr/lib/systemd:/usr/lib/systemd:ro \
    -v /var/lib:/var/lib:ro \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \
    --label docker_bench_security \
    docker-bench-security
```

### 选项说明

```sh
-c CHECK     仅运行指定检查 ID（如 check_2_2）
-e CHECK     排除指定检查 ID
-i INCLUDE   按容器/镜像名称模式过滤
-x EXCLUDE   排除容器/镜像名称模式
-p PRINT     不输出修复建议
```

---

## 一、主机配置（Section 1）

### 1.1 Linux 主机安全配置

| 检查项 | 描述 | 自动化 |
|--------|------|--------|
| 1.1.1 | 为容器创建独立分区（`/var/lib/docker`） | ✅ |
| 1.1.2 | 仅允许受信任用户控制 Docker 守护进程 | ✅ |
| 1.1.3-1.1.18 | 审计关键 Docker 文件/目录（daemon.json、docker.service、containerd.sock 等） | ✅ |

```bash
# 配置 Docker 数据目录独立分区
# /etc/fstab
/dev/sdb1 /var/lib/docker ext4 defaults,noatime 0 2

# 启用审计
auditctl -w /var/lib/docker -k docker
auditctl -w /etc/docker -k docker
auditctl -w /etc/docker/daemon.json -k docker
```

### 1.2 通用配置

| 检查项 | 描述 |
|--------|------|
| 1.2.1 | 使用已加固的容器主机系统（如 Alpine Linux、Flatcar） |
| 1.2.2 | 保持 Docker 版本为最新 |

---

## 二、Docker 守护进程配置（Section 2）

核心加固项（Scored 检查）：

### 2.2 限制默认桥接网络容器间通信

```json
{
  "icc": false
}
```

```bash
# /etc/docker/daemon.json
{
  "icc": false,                    # 禁止默认桥接容器互访
  "log-level": "info",             # 日志级别
  "iptables": true,                # 开放 iptables 管理
  "insecure-registries": [],       # 禁止使用不安全镜像仓库
  "storage-driver": "overlay2",    # 使用 overlay2（禁用 aufs）
  "userland-proxy": false,         # 禁用用户态代理
  "live-restore": true,            # 守护进程重启时保持容器运行
  "no-new-privileges": true,       # 禁止容器获取新权限
  "experimental": false,           # 生产环境禁用实验特性
  "userns-remap": "default",       # 启用用户命名空间重映射
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "default-ulimits": {
    "nofile": {
      "name": "nofile",
      "hard": 64000,
      "soft": 64000
    }
  }
}
```

关键检查清单：

| 检查 | 安全要求 | 风险 |
|------|----------|------|
| 2.2 | `icc: false` | 容器间直接通信绕过防火墙 |
| 2.3 | `log-level: info` | 日志遗漏安全事件 |
| 2.4 | `iptables: true` | 防火墙规则管理失败 |
| 2.5 | `insecure-registries` 为空 | HTTP 仓库可被中间人攻击 |
| 2.6 | 使用 overlay2 而非 aufs | aufs 已弃用且有稳定性问题 |
| 2.7 | 配置 TLS 认证（远程 API） | 未加密 API 可被中间人攻击 |
| 2.9 | 启用用户命名空间（userns-remap） | 容器 root = 主机非特权用户 |
| 2.12 | 启用授权插件 | 客户端命令未授权执行 |
| 2.13 | 集中式远程日志 | 日志丢失无法溯源 |
| 2.14 | `no-new-privileges: true` | 容器可通过 SUID 提权 |
| 2.15 | `live-restore: true` | 守护进程重启导致容器停服 |
| 2.16 | `userland-proxy: false` | 用户态代理增加攻击面 |
| 2.18 | `experimental: false` | 实验特性不稳定 |

### TLS 认证配置（检查 2.7）

```bash
# 生成 CA 证书和服务器证书
# 配置 dockerd 启动参数
ExecStart=/usr/bin/dockerd \
    --tlsverify \
    --tlscacert=/etc/docker/ca.pem \
    --tlscert=/etc/docker/server.pem \
    --tlskey=/etc/docker/server-key.pem \
    -H=0.0.0.0:2376
```

### 用户命名空间重映射（检查 2.9）

```json
{
  "userns-remap": "default"
}
```

```bash
# 创建映射用户
useradd --system dockremap
usermod --add-subuids 100000-165535 dockremap
usermod --add-subgids 100000-165535 dockremap

# 验证
grep dockremap /etc/subuid /etc/subgid
# /etc/subuid: dockremap:100000:65536
# /etc/subgid: dockremap:100000:65536
```

---

## 三、容器镜像与构建安全（Section 4）

### 关键检查

| 检查 | 要求 |
|------|------|
| 4.1 | 创建容器的非 root 用户 |
| 4.2 | 使用可信基础镜像（Docker Content Trust） |
| 4.3 | 确保镜像中不包含敏感信息 |
| 4.5 | 启用 Docker Content Trust |
| 4.6 | 添加 HEALTHCHECK 指令 |
| 4.7 | 不将敏感信息传递给 Dockerfile `ARG` |
| 4.8 | 正确清理 `apt-get`, `yum` 等缓存 |
| 4.9 | 使用 COPY 替代 ADD（避免自动解压远程 URL） |
| 4.10 | 不安装不必要软件包 |
| 4.11 | 镜像标签使用带版本号的固定标签 |

```dockerfile
# Dockerfile 安全最佳实践
FROM eclipse-temurin:21-jre-alpine AS runtime

# 创建非 root 用户
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# 安装必要工具后清理缓存（单层）
RUN apk add --no-cache --virtual .build-deps curl \
    && curl -fsSL https://example.com/checksum -o /tmp/checksum \
    && apk del .build-deps

# 使用 COPY 而非 ADD
COPY --from=builder /app/app.jar /app/app.jar

# 添加健康检查
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

# 切换到非 root 用户
USER appuser:appgroup
```

```bash
# 启用 Docker Content Trust（检查 4.5）
export DOCKER_CONTENT_TRUST=1
export DOCKER_CONTENT_TRUST_SERVER=https://notary.example.com

# 签名并推送镜像
docker trust sign myapp:1.0.0
docker push myapp:1.0.0
```

---

## 四、容器运行时安全（Section 5）

### 关键检查清单

| 检查 | 安全要求 | 违规后果 |
|------|----------|----------|
| 5.1 | 不在不需要时启用 Swarm 模式 | 增加攻击面 |
| 5.2 | 启用 AppArmor 配置文件 | 无限制的容器可执行系统调用 |
| 5.3 | 配置 SELinux 安全选项 | 无 MAC 保护 |
| 5.4 | 限制 Linux Capabilities | 容器拥有过多内核权限 |
| 5.5 | 不使用特权容器 | 容器等价于主机 root |
| 5.6 | 不挂载敏感主机目录 | 容器可修改主机关键文件 |
| 5.7 | 不在容器内运行 sshd | 增加远程入侵入口 |
| 5.8 | 不映射特权端口 (<1024) | 容器可冒充系统服务 |
| 5.10 | 不共享主机网络命名空间 | 容器可监听主机所有网络接口 |
| 5.11 | 限制内存使用 | OOM 可影响主机 |
| 5.12 | 设置 CPU 优先级限制 | CPU 资源争抢 |
| 5.13 | 根文件系统设为只读 | 容器被写恶意脚本后无法持久化 |
| 5.14 | 入站流量绑定到特定主机接口 | 多余暴露面 |
| 5.15 | 重启策略设为 `on-failure:5` | 无限重启导致 DoS |
| 5.16 | 不共享主机 PID 命名空间 | 容器可查看主机所有进程 |
| 5.17 | 不共享主机 IPC 命名空间 | 容器可访问主机共享内存 |
| 5.18 | 不暴露主机设备 | 容器可直接操作主机硬件 |
| 5.20 | 不设置共享挂载传播 | 容器可影响主机挂载点 |
| 5.21 | 不共享主机 UTS 命名空间 | 容器可修改主机名 |
| 5.22 | 不禁用 seccomp 默认配置 | 默认 seccomp 阻止数百系统调用 |
| 5.24 | 不使用 `docker exec --privileged` | 特权 exec 可逃逸 |
| 5.25 | 不使用 `docker exec --user=root` | root 执行可修改容器文件系统 |
| 5.26 | 确认 cgroup 使用 | 资源限制无效 |
| 5.27 | 限制容器获取额外权限 | 容器内 setuid 可提权 |
| 5.28 | 运行时健康检查 | 无法及时发现故障容器 |
| 5.29 | 使用最新镜像版本 | 旧版本含有已知漏洞 |
| 5.30 | 使用 PIDs cgroup 限制 | fork bomb 可耗尽主机 PID |

### 运行时安全示例

```bash
# 安全运行容器（完整加固）
docker run -d \
    --name app \
    --security-opt apparmor=docker-default \
    --security-opt seccomp=/path/to/seccomp-profile.json \
    --cap-drop=ALL \
    --cap-add=NET_BIND_SERVICE \
    --memory=512m \
    --memory-swap=512m \
    --cpus=1.0 \
    --read-only \
    --tmpfs /tmp:rw,noexec,nosuid,size=64m \
    --restart=on-failure:5 \
    --publish 127.0.0.1:8080:8080 \
    --ulimit nofile=1024:1024 \
    --pids-limit=100 \
    --security-opt no-new-privileges:true \
    myapp:1.0.0
```

### Docker Compose 安全配置

```yaml
version: "3.9"
services:
  app:
    image: myapp:1.0.0
    read_only: true
    tmpfs:
      - /tmp:size=64m,noexec,nosuid
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    security_opt:
      - apparmor=docker-default
      - seccomp=/path/to/seccomp.json
      - no-new-privileges:true
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
    restart: on-failure:5
    ports:
      - "127.0.0.1:8080:8080"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

---

## 五、Docker Swarm 配置（Section 6）

| 检查 | 要求 |
|------|------|
| 6.1 | 不在 Swarm 节点上绑定特权端口 |
| 6.2 | 管理节点数限制在 3-7 台 |
| 6.3 | Swarm 服务绑定到特定主机接口 |
| 6.4 | 不共享主机网络命名空间给 Swarm 服务 |
| 6.5 | Swarm 服务的 secret 使用复杂加密数据存储 |
| 6.6 | 定期轮换 Swarm CA 证书 |
| 6.7 | 自动轮换 Swarm 节点证书 |
| 6.8 | Swarm 管理节点操作需要锁定快照 |
| 6.9 | 在 Swarm 中启用 autolock |

---

## 六、常用审计命令

```bash
# 一键运行所有检查
sudo sh docker-bench-security.sh

# 仅运行容器运行时安全检查
sudo sh docker-bench-security.sh -c container_runtime

# 排除守护进程配置检查
sudo sh docker-bench-security.sh -e docker_daemon_configuration

# 输出 JSON 格式日志
sudo sh docker-bench-security.sh -l /tmp/docker-bench-log

# 限制列表输出项数
sudo sh docker-bench-security.sh -n 20
```

## 注意事项

1. **用户命名空间限制**：启用 `userns-remap` 后，宿主机文件系统挂载和端口映射行为会改变，部分需要访问宿主文件系统的镜像可能不兼容
2. **只读根文件系统**：`--read-only` 需要配合 `--tmpfs` 使用，否则部分应用无法写入临时文件
3. **Capabilities 最小化**：删除 ALL 后再按需添加最小集（如 `NET_BIND_SERVICE`），不要反向操作
4. **auditd 规则影响**：大量审计规则可能影响磁盘 I/O 和系统性能，生产环境按需裁剪
5. **Seccomp 兼容性**：默认 seccomp 配置对大多数应用是安全的，只有特定系统调用密集型应用（如某些 JIT 编译器）需要自定义
6. **Docker Content Trust**：启用后所有 pull/push 操作都会验证签名，CI/CD 流程中需要额外配置签名步骤
7. **版本兼容性**：Docker Bench for Security 需要 Docker 1.13.0+，建议使用 Docker 24+ 以获得最好的检查覆盖
