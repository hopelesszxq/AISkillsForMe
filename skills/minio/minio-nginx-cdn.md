---
name: minio-nginx-cdn
description: MinIO 对象存储 CDN 搭建：Nginx 反向代理、SSL 终止、大文件流式传输最佳实践
tags: [minio, nginx, cdn, reverse-proxy, ssl, docker]
---

## 概述

生产环境中，MinIO 不应直接暴露给公网。通过 Nginx 反向代理实现 SSL 终止、域名绑定、大文件流式传输，构建自托管 S3 兼容 CDN。

## 架构

```
Client → HTTPS (cdn.example.com:443) → Nginx (SSL 终止) → MinIO (127.0.0.1:9000)
```

- MinIO 绑定 localhost，只允许 Nginx 访问
- Nginx 对外暴露 443 端口，Let's Encrypt 免费证书
- 三个关键配置点：Host 头、body 大小限制、缓冲关闭

## Docker Compose 部署 MinIO

```yaml
# /opt/minio/docker-compose.yml
services:
  minio:
    image: minio/minio:latest
    container_name: minio
    restart: unless-stopped
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
      MINIO_SERVER_URL: https://cdn.example.com
      MINIO_BROWSER_REDIRECT_URL: https://cdn.example.com/console
    ports:
      - "127.0.0.1:9000:9000"   # 仅本地监听
      - "127.0.0.1:9001:9001"   # Console 仅本地
    volumes:
      - ./data:/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### 关键环境变量

| 变量 | 说明 |
|------|------|
| `MINIO_SERVER_URL` | 设为公网访问地址（如 `https://cdn.example.com`），否则预签名 URL 会返回 `127.0.0.1` |
| `MINIO_BROWSER_REDIRECT_URL` | Console 重定向地址 |
| 端口绑定 | 强制 `127.0.0.1:` 前缀，防止绕过 Nginx 直接访问 |

## Nginx 配置（最易踩坑的部分）

```nginx
# /etc/nginx/sites-available/cdn.example.com

server {
    listen 80;
    listen [::]:80;
    server_name cdn.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name cdn.example.com;

    # Let's Encrypt 证书（Certbot 自动填充）
    # ssl_certificate /etc/letsencrypt/live/cdn.example.com/fullchain.pem;
    # ssl_certificate_key /etc/letsencrypt/live/cdn.example.com/privkey.pem;

    # 允许大文件上传
    client_max_body_size 5G;

    # 关闭缓冲——MinIO 需要流式传输
    proxy_buffering off;
    proxy_request_buffering off;

    # 防止 Nginx 二次 chunked 编码
    chunked_transfer_encoding off;

    # 超时配置（大文件场景）
    proxy_connect_timeout 300;
    proxy_send_timeout 300;
    proxy_read_timeout 300;
    send_timeout 300;

    location / {
        proxy_pass http://127.0.0.1:9000;

        # ⚠️ Host 头必须透传——否则 S3 签名验证失败
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

启用站点并验证：
```bash
sudo ln -s /etc/nginx/sites-available/cdn.example.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## SSL 证书（Certbot + Let's Encrypt）

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx

# 自动签发并配置 Nginx SSL
sudo certbot --nginx -d cdn.example.com

# 验证自动续期
sudo certbot renew --dry-run
```

## 踩坑记录

### 1. `SignatureDoesNotMatch` 错误

**现象**：预签名 URL 访问报签名不匹配
**原因**：Nginx 转发时将 `Host` 头改为了 `127.0.0.1:9000`，S3 签名计算依赖 Host
**解决**：`proxy_set_header Host $host;`

### 2. 大文件上传失败（413 Request Entity Too Large）

**现象**：上传超过 1MB 的文件被 Nginx 拒绝
**原因**：Nginx 默认 `client_max_body_size` 为 1MB
**解决**：设为足够大的值（如 5G）

### 3. 上传/下载速度慢或内存溢出

**现象**：大文件传输极慢，磁盘空间爆满
**原因**：Nginx 默认缓冲整个请求/响应体到临时目录
**解决**：`proxy_buffering off; proxy_request_buffering off;`

### 4. 预签名 URL 返回 localhost

**现象**：生成的预签名 URL 域名是 `127.0.0.1`
**原因**：MinIO 不知道公网地址
**解决**：设置环境变量 `MINIO_SERVER_URL=https://cdn.example.com`

## 验证

```bash
# 测试上传
curl -k -X PUT \
  -H "Host: cdn.example.com" \
  --data-binary @test.jpg \
  "https://cdn.example.com/test-bucket/test.jpg"

# 测试下载
curl -k -O "https://cdn.example.com/test-bucket/test.jpg"

# 生成预签名 URL 测试
mc alias set myminio https://cdn.example.com $MINIO_ROOT_USER $MINIO_ROOT_PASSWORD
mc share download myminio/test-bucket/test.jpg
```
