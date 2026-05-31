---
name: docker-compose-patterns
description: Docker Compose 微服务架构模式：数据库隔离、健康检查编排、服务发现集成与网络拓扑设计
tags: [tools, docker, docker-compose, microservices, architecture]
---

## 要点

Docker Compose 不仅是本地开发工具，也是单主机微服务部署的实用选择。掌握以下架构模式可以写出更健壮、更易维护的 Compose 配置。

本文涵盖的架构模式：

| 模式 | 目标 | 核心要点 |
|------|------|---------|
| 数据库隔离 | 服务间解耦 | 每服务独立数据库容器 |
| 健康检查编排 | 可靠启动顺序 | depends_on + condition: service_healthy |
| 服务发现集成 | 动态寻址 | Consul/Eureka + DNS 轮询 |
| 网络拓扑 | 安全隔离 | 多网络、内网桥接 |

## 模式一：数据库隔离（Database-Per-Service）

微服务的核心原则——每个服务拥有自己的数据库。Compose 中为每个服务搭配专用的数据库容器：

```yaml
version: '3.8'

services:
  user-service:
    build: ./services/user-service
    environment:
      DB_HOST: user-db
      DB_PORT: 5432
      DB_NAME: users
      DB_USER: user_app
      DB_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    depends_on:
      user-db:
        condition: service_healthy   # 等数据库就绪后再启动
    networks:
      - app-network

  user-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: users
      POSTGRES_USER: user_app
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user_app"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 10s
    volumes:
      - user_db_data:/var/lib/postgresql/data
    networks:
      - app-network

  product-service:
    build: ./services/product-service
    depends_on:
      product-db:
        condition: service_healthy
    networks:
      - app-network

  product-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: products
      POSTGRES_USER: product_app
      POSTGRES_PASSWORD: ${PRODUCT_DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U product_app"]
      interval: 5s
      timeout: 3s
      retries: 5
    volumes:
      - product_db_data:/var/lib/postgresql/data
    networks:
      - app-network

secrets:
  db_password:
    file: ./secrets/db_password.txt

volumes:
  user_db_data:
  product_db_data:

networks:
  app-network:
    driver: bridge
```

### 关键要点

- **`condition: service_healthy`** 比简单的 `depends_on` 更可靠，确保依赖完全就绪
- **`start_period`**：给容器启动留出缓冲期，期间 healthcheck 失败不计入重试次数
- **`POSTGRES_PASSWORD_FILE`**：通过 Docker Secrets 注入密码，避免明文环境变量

## 模式二：服务发现集成

### Consul + DNS 方式

```yaml
services:
  consul:
    image: hashicorp/consul:latest
    ports:
      - "8500:8500"
    command: agent -server -bootstrap -ui -client=0.0.0.0
    networks:
      - microservices-network

  api-gateway:
    build: ./api-gateway
    environment:
      CONSUL_HOST: consul
    depends_on:
      consul:
        condition: service_started
    networks:
      - microservices-network

  user-service:
    build: ./services/user-service
    environment:
      CONSUL_HOST: consul
    depends_on:
      consul:
        condition: service_started
    networks:
      - microservices-network
```

对于 Spring Cloud 应用，通过环境变量注入 Consul 地址：

```yaml
environment:
  - SPRING_CLOUD_CONSUL_HOST=consul
  - SPRING_CLOUD_CONSUL_PORT=8500
```

### Eureka 方式（Spring Cloud）

```yaml
services:
  eureka:
    build: ./eureka-server
    ports:
      - "8761:8761"
    networks:
      - backend

  gateway:
    build: ./gateway
    environment:
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka:8761/eureka/
    depends_on:
      - eureka
    networks:
      - backend
```

## 模式三：网络拓扑与安全隔离

合理利用 Docker 网络可以实现服务间安全隔离：

```yaml
networks:
  frontend:    # API 网关和前端
    driver: bridge
  backend:     # 内部服务
    driver: bridge
  data:        # 数据层
    driver: bridge
    internal: true     # 禁止外部访问数据层网络

services:
  api-gateway:
    networks:
      - frontend
      - backend    # 网关同时在两个网络

  user-service:
    networks:
      - backend
      - data       # 只能通过 backend 通信，数据层网络不可外部访问

  user-db:
    networks:
      - data       # 数据库只在 data 网络，对外完全不可见

  redis:
    networks:
      - data
```

## 模式四：带重试的启动脚本

解决 Compose 启动顺序的不可靠问题，封装等待脚本到入口点：

```bash
#!/bin/sh
# wait-for-service.sh

set -e

host="$1"
shift
cmd="$@"

until nc -z "$host" 5432; do
  >&2 echo "Waiting for $host:5432..."
  sleep 2
done

>&2 echo "$host:5432 is available"
exec $cmd
```

```yaml
user-service:
  build: ./services/user-service
  entrypoint: ["./wait-for-service.sh", "user-db", "java", "-jar", "app.jar"]
```

更推荐的方式：在应用代码中实现重试（使用 Resilience4j 的 Retry），而非在 Compose/脚本层等待。

## 注意事项

- **`depends_on` 只控制启动顺序，不等待服务就绪**，必须配合 healthcheck 或自定义等待脚本
- **`links` 已被弃用**，始终使用 `networks` 自定义网络
- **`internal: true`** 网络中的容器无法访问外部互联网，适合数据库等数据层容器
- **数据库密码建议使用 `secrets` 机制**，避免在 docker-compose.yml 或 .env 文件中明文存储
- **服务发现使用 DNS 轮询**：Docker Compose 默认使用内置 DNS，服务名可解析为容器 IP。但需要应用层实现缓存/重试策略，因为容器重启后 IP 会变
- **生产环境建议升级为 Docker Swarm**：Compose 语法可直接用于 Swarm Stack，获得内置的滚动更新、Secrets 管理和多副本扩缩能力
