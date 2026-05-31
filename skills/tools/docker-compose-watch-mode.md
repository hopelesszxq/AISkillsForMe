---
name: docker-compose-watch-mode
description: Docker Compose watch 模式实现开发热重载：代码变更自动同步、免手动重启
tags: [tools, docker, docker-compose, development, hot-reload, devops]
---

## Docker Compose Watch 模式（v2.23+）

Docker Compose 从 **v2.23.0** 开始支持 `watch` 指令，用于开发时的自动文件同步和容器热重载。相比传统 `volumes + nodemon` 方案，`watch` 模式更高效且与 Compose 原生集成。

## 一、与传统方案对比

| 方案 | 同步方式 | 触发重启 | 性能 | 平台支持 |
|------|---------|---------|------|---------|
| **volumes bind mount** | 实时文件映射 | 需要外部工具（nodemon） | 文件系统事件量大时开销高 | 所有平台 |
| **docker compose watch** | 按规则监听+增量同步 | 内置 action 机制重启 | 按路径过滤，效率更高 | Docker Desktop / Linux |
| **Tilt / Skaffold** | 全方位同步+构建 | 自动 | 功能最强 | 需要额外安装 |

## 二、基础配置

### docker-compose.yml

```yaml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    develop:
      watch:
        # 规则 1：同步源代码（不重启）
        - action: sync
          path: ./src
          target: /app/src
          ignore:
            - node_modules/
            - "*.test.ts"

        # 规则 2：监听配置文件变更 → 重建容器
        - action: rebuild
          path: ./pom.xml

        # 规则 3：监听静态资源变更 → 同步到容器
        - action: sync
          path: ./static
          target: /app/static

  redis:
    image: redis:7-alpine
```

### 启动

```bash
# 以 watch 模式启动（阻塞，监听文件变更）
docker compose watch

# 如果想在后台启动服务，再加 watch
docker compose up -d
docker compose watch
```

## 三、Java/Spring Boot 最佳实践

### Spring Boot Devtools 配合 Watch 模式

```yaml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_DEVTOOLS_RESTART_ENABLED=true
      - SPRING_DEVTOOLS_LIVERELOAD_ENABLED=true
    develop:
      watch:
        # 监听编译产出（IDE 或 mvn compile 写入 target/classes）
        - action: sync
          path: ./target/classes
          target: /app/target/classes
        # pom.xml 变更时需要完整重建
        - action: rebuild
          path: ./pom.xml
        # 新增依赖时重建
        - action: rebuild
          path: ./pom.xml.lock
```

> **注意**：Spring Boot Devtools 使用**双类加载器机制**，`sync` 到 `target/classes` 后 Devtools 会自动检测类变更并触发重启，比容器级别 `rebuild` 快得多（秒级 vs 分钟级）。

### Maven 编译 + Watch 联动

```bash
# 终端 1：持续编译
mvn compile -o

# 终端 2：Docker Compose watch 同步编译结果
docker compose watch
```

## 四、Python / Node.js 应用

```yaml
services:
  api:
    build: .
    develop:
      watch:
        # Python：代码变更时同步（框架自带 reload）
        - action: sync
          path: ./app
          target: /app
          ignore:
            - __pycache__/
            - "*.pyc"
        # requirements 变更时重建
        - action: rebuild
          path: ./requirements.txt
```

```yaml
services:
  nextjs:
    build: .
    develop:
      watch:
        # Node.js：同步 src 目录（next dev 热更新）
        - action: sync
          path: ./src
          target: /app/src
        # 依赖变更时重建
        - action: rebuild
          path: ./package.json
```

## 五、action 类型详解

### sync（同步）

将本地 path 下的文件增量复制到容器内的 target 路径，**不重启容器**。

```yaml
- action: sync
  path: ./src                # 本地路径（相对 compose 文件目录）
  target: /app/src          # 容器内目标路径
  ignore:                   # 忽略模式（可选）
    - "*.log"
    - .git/
```

### rebuild（重建）

当 path 的文件变更时，Docker Compose 会终止当前容器，重新执行 `build`，然后启动新容器。

```yaml
- action: rebuild
  path: ./Dockerfile
```

### sync-restart（同步后重启）

先同步文件，再重启容器（不重建镜像）。适合需要重启进程但不需要重新构建的场景。

```yaml
- action: sync-restart
  path: ./config
  target: /app/config
```

## 六、高级用法

### 多服务同时 watch

```yaml
services:
  frontend:
    build: ./frontend
    develop:
      watch:
        - action: sync
          path: ./frontend/src
          target: /app/src

  backend:
    build: ./backend
    develop:
      watch:
        - action: sync
          path: ./backend/src
          target: /app/src
        - action: rebuild
          path: ./backend/pom.xml
```

### 排除 node_modules 提升性能

```yaml
develop:
  watch:
    - action: sync
      path: .
      target: /app
      ignore:
        - node_modules/          # 排除前端依赖
        - .git/
        - target/                # Maven 构建产出
        - build/                 # Gradle 构建产出
        - __pycache__/
        - .nyc_output/
```

## 注意事项

1. **Docker Compose 版本要求**：v2.23.0+。检查版本：
   ```bash
   docker compose version  # 需要 v2.23+
   ```

2. **watch 模式是阻塞命令**：`docker compose watch` 会一直运行在前台，按 Ctrl+C 退出后不会停止服务。

3. **不适用于 Production**：watch 模式是开发专用，生产环境请使用构建后的镜像。

4. **文件监听性能**：在 WSL2 或 macOS 上，如果项目中 node_modules 数量巨大，务必使用 `ignore` 排除。

5. **与 volumes 共存**：watch 和 volumes bind mount 可以同时使用，但 watch 的同步优先级更高。

## 参考

- [Docker Compose Watch官方文档](https://docs.docker.com/compose/file-watch/)
- [Docker Compose CLI Reference](https://docs.docker.com/compose/reference/)
