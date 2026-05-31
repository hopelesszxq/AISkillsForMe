---
name: onlyoffice-deployment-scaling
description: OnlyOffice Document Server 生产环境高可用部署、性能调优与规模化管理
tags: [onlyoffice, deployment, ha, scaling, performance, production]
---

## 生产部署架构

```
          ┌──────────────┐
          │  Nginx/HAP   │  ← SSL termination + 负载均衡
          │  LoadBalancer│
          └──────┬───────┘
          ┌──────┴───────┐
          │  Redis Cluster│  ← 会话共享、协同编辑状态
          └──────┬───────┘
     ┌──────┬────┴────┬──────┐
 ┌───┴───┐┌───┴───┐┌───┴───┐
 │ DS #1 ││ DS #2 ││ DS #3 │  ← Document Server 无状态集群
 └───┬───┘└───┬───┘└───┬───┘
     └────────┴────────┘
          ┌──────┴───────┐
          │  PostgreSQL    │  ← 文档元数据、变更历史
          └──────────────┘
          ┌──────┴───────┐
          │  MinIO/S3     │  ← 文档文件存储（跨可用区共享）
          └──────────────┘
```

## 一、Document Server 集群方案

### 1. Docker Compose 多实例

```yaml
version: '3.8'
services:
  onlyoffice-ds1:
    image: onlyoffice/documentserver:8.2
    environment:
      - JWT_ENABLED=true
      - JWT_SECRET=${JWT_SECRET}
      - AMQP_URI=amqp://rabbitmq:5672  # 协同编辑消息通道
      - REDIS_SERVER_HOST=redis
      - REDIS_SERVER_PORT=6379
      - DB_TYPE=postgres
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=onlyoffice
      - DB_USER=onlyoffice
      - DB_PWD=${DB_PASSWORD}
      - WOPI_ENABLED=true               # WOPI 协议支持
    volumes:
      - ds1-data:/var/www/onlyoffice/Data
      - ds1-logs:/var/log/onlyoffice
    networks:
      - onlyoffice-net

  onlyoffice-ds2:
    image: onlyoffice/documentserver:8.2
    environment:
      - JWT_ENABLED=true
      - JWT_SECRET=${JWT_SECRET}
      - AMQP_URI=amqp://rabbitmq:5672
      - REDIS_SERVER_HOST=redis
      - REDIS_SERVER_PORT=6379
      - DB_TYPE=postgres
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=onlyoffice
      - DB_USER=onlyoffice
      - DB_PWD=${DB_PASSWORD}
    volumes:
      - ds2-data:/var/www/onlyoffice/Data
      - ds2-logs:/var/log/onlyoffice
    networks:
      - onlyoffice-net

  nginx:
    image: nginx:1.27-alpine
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    ports:
      - "443:443"
    depends_on:
      - onlyoffice-ds1
      - onlyoffice-ds2
    networks:
      - onlyoffice-net
```

### 2. Nginx 负载均衡配置

```nginx
upstream onlyoffice_cluster {
    least_conn;                        # 最少连接算法
    server onlyoffice-ds1:80 max_fails=3 fail_timeout=30s;
    server onlyoffice-ds2:80 max_fails=3 fail_timeout=30s;
    keepalive 32;                      # 连接池
}

server {
    listen 443 ssl http2;
    server_name doc.example.com;

    ssl_certificate     /etc/nginx/certs/cert.pem;
    ssl_certificate_key /etc/nginx/certs/key.pem;

    # WebSocket 支持（协同编辑核心）
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    # 大文件上传支持
    client_max_body_size 100M;
    proxy_request_buffering off;
    proxy_buffering off;

    location / {
        proxy_pass http://onlyoffice_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket 路径
        location /ws/ {
            proxy_pass http://onlyoffice_cluster;
            proxy_read_timeout 86400s;  # 协同编辑长连接
        }
    }
}
```

## 二、性能调优

### 1. 内核参数（/etc/sysctl.conf）

```bash
# 单进程最大文件数
fs.file-max = 1000000

# 网络优化
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.ip_local_port_range = 1024 65000

# 大文件处理
vm.dirty_ratio = 30
vm.dirty_background_ratio = 5
```

### 2. Document Server 配置优化

```bash
# 修改 /etc/onlyoffice/documentserver/default.json

# 转换超时（大文档）
{
  "services": {
    "CoAuthoring": {
      "server": {
        "timeout": "300000"    # 5 分钟（默认 120s）
      }
    },
    "DocService": {
      "conversion": {
        "timeout": "180s"      # 文档转换超时
      }
    }
  }
}
```

### 3. 资源建议

| 场景 | 并发用户 | CPU | 内存 | 磁盘 |
|------|---------|-----|------|------|
| 小型团队 | 50 | 4 核 | 8 GB | SSD 50GB |
| 中型企业 | 200 | 8 核 | 16 GB | SSD 200GB |
| 大型部署 | 1000+ | 16 核 × 3 节点 | 32 GB × 3 | SSD 500GB + NAS |

## 三、关键配置项

### 1. Redis 配置（协同编辑性能核心）

```yaml
# docker-compose 中的 Redis 配置
redis:
  image: redis:7.4-alpine
  command: redis-server --appendonly yes --maxmemory 4gb --maxmemory-policy allkeys-lru
  volumes:
    - redis-data:/data
  networks:
    - onlyoffice-net
```

### 2. PostgreSQL 配置

```ini
# postgresql.conf 优化
max_connections = 200
shared_buffers = 4GB
effective_cache_size = 12GB
work_mem = 256MB
maintenance_work_mem = 1GB
random_page_cost = 1.1
effective_io_concurrency = 200
wal_buffers = 16MB
checkpoint_completion_target = 0.9
```

### 3. RabbitMQ 配置（协同编辑消息队列）

```yaml
rabbitmq:
  image: rabbitmq:4.0-management-alpine
  environment:
    RABBITMQ_DEFAULT_USER: onlyoffice
    RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD}
  volumes:
    - rabbitmq-data:/var/lib/rabbitmq
```

## 四、健康检查与监控

### 健康检查端点

```bash
# Document Server 健康检查
curl https://doc.example.com/healthcheck
# 返回: {"status":0}

# 转换服务状态
curl https://doc.example.com/coauthoring/HealthCheck
```

### Prometheus 集成

```yaml
# docker-compose 添加 exporter
prometheus:
  image: prom/prometheus:v2.55
  volumes:
    - ./prometheus.yml:/etc/prometheus/prometheus.yml

grafana:
  image: grafana/grafana:11.3
  ports:
    - "3000:3000"
```

## 注意事项

1. **文件存储必须共享**：所有 DS 实例必须访问同一存储（NFS/MinIO/S3），否则协同编辑的文件版本不一致
2. **JWT 密钥一致**：所有实例使用相同的 JWT_SECRET，否则回调验证失败
3. **会话亲和性**：Nginx 建议 least_conn + ip_hash，确保同一用户的请求落到同一 DS 实例（非必须，Redis 共享会话后无状态）
4. **协同编辑依赖 Redis**：Redis 不可用时协同编辑降级为只读模式，务必部署 Redis Sentinel 或 Cluster
5. **文档转换瓶颈**：文档首次打开时的格式转换是 CPU 密集型，高峰时可通过预转换缓解
6. **备份策略**：备份 PostgreSQL + 文件存储 + 配置，Document Server 实例本身无状态
7. **版本升级**：滚动升级时建议挂维护页，避免编辑中保存失败
