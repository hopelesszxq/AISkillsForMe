---
name: docker-multi-scenario-pattern
description: Docker 多场景模式 — 单一运行时 + 多部署场景，解决开发/测试/生产环境漂移问题
tags: [tools, docker, docker-compose, devops, devcontainer]
---

## 要点

Docker 本身不能保证环境一致性——即使 `FROM` 层不固定平台或镜像版本，也会出现差异。项目演化过程中，Dockerfile 会从 1 个发展到 3-4 个（开发、测试、生产），最终无人知道哪个是"正确"的。

**多场景模式**的核心思想：**一个运行时 + 多个部署场景**。Dockerfile 和基础环境必须是单一的，差异只允许在场景层存在。

| 传统做法 | 多场景模式 |
|---------|-----------|
| 多个 Dockerfile 逐渐漂移 | 单一 Dockerfile 作为运行时 |
| docker-compose.override.yml 层层叠加 | 每个场景是独立目录（物理隔离） |
| 环境差异隐藏在配置中 | 差异显式可见 |
| 启动命令复杂（-f 多个文件） | `cd scenario-xxx && make up` |

## 项目结构

```text
my-app/
├── .devcontainer/
│   ├── _configs/              # 共享运行时配置
│   ├── _scripts/              # 共享入口脚本
│   ├── _data/                 # 辅助二进制依赖
│   ├── scenario-mapped/       # 本地开发（bind mount）
│   │   ├── docker-compose.yml
│   │   ├── devcontainer.json
│   │   ├── Makefile
│   │   └── .env
│   ├── scenario-embedded/     # 部署（代码内置镜像中）
│   │   ├── docker-compose.yml
│   │   ├── devcontainer.json
│   │   ├── Makefile
│   │   └── .env
│   ├── Dockerfile.app
│   ├── Dockerfile.database
│   └── Dockerfile.*
├── .env                       # 基础应用配置
└── ...
```

## 两种场景说明

### scenario-mapped（本地开发）
通过 bind mount 挂载代码，修改实时生效，适合调试。

```bash
cd .devcontainer/scenario-mapped
make up
```

### scenario-embedded（生产 / CI）
代码复制到镜像内部，不依赖宿主机，可重复构建。

```bash
cd .devcontainer/scenario-embedded
make deploy
```

## 为什么不直接用 Docker Compose Profiles？

Compose Profiles 只解决了服务启用/禁用的开关问题，而多场景是一个更广泛的概念：

- 包含 `.env`、`Makefile`、`devcontainer.json`、额外脚本
- 包含环境启动规则和部署流程
- **场景是完整的运行环境，不仅仅是服务切换**

## 为什么不覆盖 docker-compose.override.yml？

传统方式：

```bash
docker compose -f docker-compose.yml -f docker-compose.override.yml -f docker-compose.staging.yml up
```

随文件增多，最终结果难以预测。多场景模式下：

```bash
cd .devcontainer/scenario-embedded && make deploy
```

场景成为仓库中的物理对象，而不是多个标志和文件的组合。

## CI 也是场景

CI 不再是独立世界。CI 管道使用与其他环境相同的场景启动模型。开发、CI、生产使用相同的启动方式，环境差异越少，部署后的意外越少。

```makefile
deploy:
	@$(MAKE) build
	@$(MAKE) migrations-check
	@$(MAKE) up
	@$(MAKE) migrations-run
	@$(MAKE) healthcheck
```

## Podman 兼容

该模式不绑定 Docker Engine。场景描述的是启动方式而非特定容器引擎，同一套场景可同时用于 Docker 和 Podman。

## 模式保证

- 所有场景共享单一运行时
- 开发和生产之间无隐藏差异
- 部署场景显式分离
- 开发工具与生产环境隔离
- 通过共享层实现集中配置

## 注意事项

- **不要在共享运行时中添加场景特定逻辑**
- **不要在 `_configs` 之外复制配置**
- **不要在 Dockerfile 中混入开发和生产的差异行为**
- 该模式不消除对架构纪律的需求，只是让漂移变得可见和可控
- 如果项目很简单（仅一个环境），不需要引入此模式

## 参考

- [Multi-Scenario Docker Pattern 完整示例](https://github.com/outcomer/multi-scenario-docker-pattern)
