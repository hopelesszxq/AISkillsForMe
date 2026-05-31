---
name: devcontainers
description: Dev Containers 开发环境配置最佳实践：devcontainer.json、Dockerfile、Features、CI 集成
tags: [tools, devcontainer, docker, vscode, devops]
---

## 概述

Dev Containers（开发容器）是 VS Code / GitHub Codespaces / DevPod 使用的标准化开发环境规范。通过 `.devcontainer/` 目录下的配置，团队可以共享一致、可复现的开发环境。DevPod（14.9k stars）提供了开源、不受限于特定平台的 Dev Container 引擎。

## 一、目录结构

```
.devcontainer/
├── devcontainer.json      # 主配置文件
├── Dockerfile             # 自定义 Docker 镜像
├── docker-compose.yml     # 多容器场景（可选）
├── postCreateCommand.sh   # 容器创建后执行脚本
└── .env                   # 环境变量（不提交到 Git）
```

## 二、devcontainer.json 核心配置

### 基础模板

```json
{
    "name": "Spring Boot Dev",
    "image": "mcr.microsoft.com/devcontainers/java:17",
    "features": {
        "ghcr.io/devcontainers/features/java:1": {
            "version": "21",
            "installMaven": true,
            "mavenVersion": "3.9.9",
            "installGradle": true
        },
        "ghcr.io/devcontainers/features/docker-in-docker:2": {}
    },
    "customizations": {
        "vscode": {
            "extensions": [
                "vscjava.vscode-java-pack",
                "vmware.vscode-boot-dev-pack",
                "sonarsource.sonarlint-vscode"
            ],
            "settings": {
                "java.configuration.runtimes": [
                    { "name": "JavaSE-21", "path": "/usr/local/sdkman/candidates/java/21" }
                ],
                "editor.formatOnSave": true
            }
        }
    },
    "postCreateCommand": "mvn dependency:resolve -q",
    "forwardPorts": [8080, 5432, 6379],
    "remoteUser": "vscode"
}
```

### Features 详解

Features 是预打包的开发工具组件，可以直接从 Container Registry 引用：

| Feature | 用途 | 来源 |
|---------|------|------|
| `java` | JDK、Maven、Gradle | devcontainers/features |
| `node` | Node.js 版本管理 | devcontainers/features |
| `docker-in-docker` | 容器内使用 Docker | devcontainers/features |
| `sshd` | SSH 服务端 | devcontainers/features |
| `kubectl-helm-minikube` | Kubernetes 工具链 | devcontainers/features |
| `github-cli` | GitHub CLI | devcontainers/features |

```json
// 多版本 JDK 混合
"features": {
    "ghcr.io/devcontainers/features/java:1": {
        "version": "21",
        "jdkDistros": ["oracle", "graalvm"],
        "installGraalVM": true
    }
}
```

## 三、Dockerfile 定制方案

当内置 image 不能满足需求时，使用自定义 Dockerfile。

```dockerfile
FROM mcr.microsoft.com/devcontainers/java:21

# 安装额外工具
RUN apt-get update && export DEBIAN_FRONTEND=noninteractive \
    && apt-get install -y --no-install-recommends \
        postgresql-client \
        redis-tools \
        httpie \
    && rm -rf /var/lib/apt/lists/*

# 安装 Skaffold（K8s 开发）
RUN curl -Lo /usr/local/bin/skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64 \
    && chmod +x /usr/local/bin/skaffold

# 预装 Spring Boot CLI
RUN sdk install springboot 3.4.0
```

```json
// devcontainer.json 中引用
{
    "build": {
        "dockerfile": "Dockerfile",
        "args": {
            "USER_UID": "1000",
            "USER_GID": "1000"
        }
    }
}
```

## 四、docker-compose 多容器方案

微服务开发通常需要数据库、中间件等依赖。

```yaml
# docker-compose.yml
version: '3.8'
services:
  devcontainer:
    build:
      context: ..
      dockerfile: .devcontainer/Dockerfile
    volumes:
      - ..:/workspace:cached
    command: /bin/sh -c "while sleep 1000; do :; done"
    depends_on:
      - postgres
      - redis
    network_mode: service:postgres

  postgres:
    image: postgres:18
    environment:
      POSTGRES_DB: myapp
      POSTGRES_PASSWORD: devpass
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

volumes:
  pgdata:
```

```json
// devcontainer.json
{
    "dockerComposeFile": "docker-compose.yml",
    "service": "devcontainer",
    "workspaceFolder": "/workspace",
    "shutdownAction": "stopCompose"
}
```

## 五、DevPod 开源方案

DevPod 是开源的 Dev Container 引擎，不依赖特定云平台。

```bash
# 安装 DevPod
curl -L https://github.com/loft-sh/devpod/releases/latest/download/devpod-linux-amd64 -o /usr/local/bin/devpod
chmod +x /usr/local/bin/devpod

# 基于 devcontainer.json 启动
devpod up . --ide vscode

# 列出工作区
devpod list

# 停止工作区
devpod stop my-project

# SSH 进入工作区
devpod ssh my-project
```

## 六、CI 中使用 Dev Containers

在 CI 流水线中复用相同的开发环境配置。

```yaml
# .github/workflows/ci.yml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Dev Container Build
        uses: devcontainers/ci@v0.3
        with:
          runCmd: |
            mvn verify -B
```

```yaml
# GitLab CI
dev-container-test:
  image: mcr.microsoft.com/devcontainers/java:21
  before_script:
    - apt-get update && apt-get install -y maven
  script:
    - mvn test
```

## 七、环境变量管理

```json
{
    "containerEnv": {
        "SPRING_PROFILES_ACTIVE": "dev",
        "DATABASE_URL": "jdbc:postgresql://postgres:5432/myapp",
        "REDIS_HOST": "redis",
        "LOG_LEVEL": "DEBUG"
    },
    "remoteEnv": {
        "GITHUB_TOKEN": "${localEnv:GITHUB_TOKEN}"
    }
}
```

## 注意事项

| 要点 | 说明 |
|------|------|
| **卷挂载性能** | Docker for Mac 上 `/workspace` 挂载性能差，使用 `cached` 标识优化；WSL2 用户推荐将代码放在 Linux 文件系统 |
| **Features 版本固定** | Features 使用 `:1` 标签指向大版本，生产环境应固定到具体 SHA 或 semver 版本 |
| **端口转发** | `forwardPorts` 中的端口在 VS Code 窗口关闭后自动转发停止，如需持久化用 `appPort` |
| **磁盘空间** | Dev Container 镜像可能很大，定期清理：`docker system prune -a` |
| **Git 凭证** | 使用 `gh auth login` 或 `GITHUB_TOKEN` 环境变量处理 Git 操作认证 |
| **多架构支持** | Apple Silicon Mac 用户需确保 Docker image 支持 `linux/arm64`，或使用 `--platform linux/amd64` 模拟（性能损失） |
| **postCreateCommand** | 不要放长时间运行的命令，优先使用 Dev Container Features 预装依赖 |
