---
name: nacos-grpc-config
description: Nacos 2.x gRPC 长连接机制、配置灰度发布与多环境治理
tags: [spring-cloud, nacos, grpc, config, grayscale]
---

## Nacos 2.x gRPC 架构变化

Nacos 2.0+ 从 HTTP 轮询升级为 **gRPC 长连接**，带来以下改进：

- **配置变更实时推送**：客户端与服务端建立双向流 gRPC 连接，配置变更无需轮询，延迟降至毫秒级
- **减少网络开销**：不再因轮询产生大量 HTTP 请求
- **连接复用**：同一个连接可以处理配置监听、服务心跳、服务查询等多种请求

### 连接生命周期

```
客户端启动 → gRPC 握手 (SDK) → 注册配置/服务监听 → 长连接保活
                                                      ↓
                                                30s 心跳检测
                                                      ↓
                                                断开自动重连
```

## 配置灰度发布

Nacos 2.2+ 支持基于 Beta 标签的配置灰度：

```yaml
# application.yml — 客户端配置
spring:
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        namespace: dev
        group: DEFAULT_GROUP
        file-extension: yaml
        # 灰度标签匹配
        config-long-poll-timeout: 30000
```

### 灰度发布流程

```bash
# 1. 通过 Nacos 控制台发布灰度配置
#    指定 Beta 匹配规则，如 metadata 标签

# 2. 配置 JSON 示例：
# {
#   "dataId": "application.yaml",
#   "group": "DEFAULT_GROUP",
#   "betaIps": "192.168.1.100,192.168.1.101",
#   "content": "feature: new-value\n"
# }
```

### 客户端配置灰度标签

```java
@Configuration
public class NacosGrayConfig {
    @Bean
    public NacosConfigProperties nacosConfigProperties() {
        NacosConfigProperties props = new NacosConfigProperties();
        // 设置灰度标记，匹配服务端 Beta 规则
        props.getTags().put("version", "gray-v2");
        return props;
    }
}
```

## Nacos 集群部署最佳实践

### 生产推荐架构

```
         ┌───────────────────┐
         │   SLB (NGINX/LVS) │  ← gRPC 端口 9848 转发
         └────────┬──────────┘
                   │
    ┌──────────────┼──────────────┐
    │              │              │
┌───▼────┐   ┌───▼────┐   ┌───▼────┐
│Nacos 01│   │Nacos 02│   │Nacos 03│
│8848    │   │8848    │   │8848    │
│9848 gRPC│  │9848 gRPC│  │9848 gRPC│
└───┬────┘   └───┬────┘   └───┬────┘
    │              │              │
    └──────────────┼──────────────┘
                   │
            ┌──────▼──────┐
            │   MySQL 8.0  │  ← 共享存储
            │ + Derby(备)  │
            └─────────────┘
```

### Nacos 集群配置文件（cluster.conf）

```bash
# ${NACOS_HOME}/conf/cluster.conf
# 节点 1
192.168.1.10:8848
# 节点 2
192.168.1.11:8848
# 节点 3
192.168.1.12:8848
```

### application.properties 关键配置

```properties
# 使用 MySQL 作为数据源（生产必配）
spring.datasource.platform=mysql
db.url.0=jdbc:mysql://mysql-host:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true
db.user.0=nacos
db.password.0=your_password

# gRPC 端口（默认偏移量 ±1000）
# 9848 = 8848 + 1000 → gRPC 客户端通信
# 9849 = 8848 + 1001 → gRPC 节点间通信
nacos.remote.server.grpc.port.offset=1000

# 开启鉴权（生产必配）
nacos.core.auth.enabled=true
nacos.core.auth.server.identity.key=nacos_server
nacos.core.auth.server.identity.value=your_secure_key
nacos.core.auth.plugin.nacos.token.secret.key=Base64EncodedSecretKey
```

### Docker Compose 部署

```yaml
version: '3.8'
services:
  nacos1:
    image: nacos/nacos-server:v2.4.0
    ports:
      - "8848:8848"
      - "9848:9848"
    environment:
      - MODE=cluster
      - NACOS_SERVERS=nacos2:8848 nacos3:8848
      - NACOS_SERVER_IP=nacos1
      - MYSQL_SERVICE_HOST=mysql
      - MYSQL_SERVICE_DB_NAME=nacos
      - MYSQL_SERVICE_USER=nacos
      - MYSQL_SERVICE_PASSWORD=nacos
      - JVM_XMS=512m
      - JVM_XMX=512m
```

## 配置多环境治理

```yaml
# bootstrap.yml
spring:
  cloud:
    nacos:
      config:
        server-addr: ${NACOS_ADDR:127.0.0.1:8848}
        namespace: ${NACOS_NAMESPACE:dev}
        ext-config:
          - data-id: common.yaml
            group: GLOBAL
            refresh: true
          - data-id: datasource.yaml
            group: ${spring.profiles.active}
            refresh: false
```

### Nacos Namespace 规划

| 环境 | Namespace ID | 说明 |
|------|-------------|------|
| dev | dev | 开发环境，允许随意修改 |
| test | test | 测试环境，配置变更需审批 |
| staging | staging | 预发环境，灰度验证 |
| prod | prod | 生产环境，变更走审批流 |

## 注意事项

1. **gRPC 端口映射**：使用 SLB 时需同时转发 HTTP(8848) 和 gRPC(9848) 端口
2. **连接超时配置**：`nacos.remote.client.grpc.timeout=10000`，避免网络抖动导致频繁重连
3. **配置刷新范围**：`@RefreshScope` 只对标注的 Bean 生效，配置类需注入
4. **灰度标签冲突**：Beta 配置优先级高于常规配置，灰度结束后务必删除 Beta 版本
5. **Nacos 3.0 预告**：将原生支持 Kubernetes（Nacos-Controller），支持 CRD 管理配置
