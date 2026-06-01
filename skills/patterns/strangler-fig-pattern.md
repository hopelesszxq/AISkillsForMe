---
name: strangler-fig-pattern
description: Strangler Fig（绞杀者）模式：渐进式单体应用向微服务迁移的实战策略与路由控制
tags: [patterns, migration, microservice, refactoring, strangler-fig, architecture]
---

## 概述

Strangler Fig（绞杀者模式）源自植物学中的绞杀榕——在新宿主树上生长，逐渐包裹并替代原宿主。对应到软件架构，就是**在旧系统外围构建新服务，通过路由逐步将功能切到新系统，最终淘汰旧系统**。

> 核心原则：**不重写、不替换，而是渐进式替换，每一步都可回滚。**

## 适用场景

| 场景 | 说明 |
|------|------|
| 单体 → 微服务 | 最经典场景，逐步拆解大单体 |
| 技术栈升级 | 如 Struts → Spring Boot，PHP → Java |
| 数据库迁移 | 同时服务新旧两个数据库，逐步切换读/写流量 |
| API 版本替换 | V1 → V2 接口平滑过渡 |

## 1. 核心组件

```
┌─────────────────────────────────────┐
│           反向代理/网关               │
│  (Nginx / Spring Cloud Gateway)      │
└──────┬─────────────────────┬────────┘
       │                     │
       ▼                     ▼
┌──────────────┐    ┌──────────────┐
│  旧系统(单体) │    │  新系统(微服务)│
│  routes-v1/* │    │  routes-v2/* │
└──────────────┘    └──────────────┘
```

### 关键要素

- **路由层（Routing Layer）**：决定请求走旧还是新系统
- **集成点（Integration Points）**：新旧系统共享的数据通道
- **替换策略（Migration Strategy）**：功能的切分粒度与顺序

## 2. 路由策略实现

### 基于 Nginx 的路由分流

```nginx
# 步骤1：新增功能走新服务
upstream new-service {
    server new-app:8080 weight=10;
}

upstream old-service {
    server legacy-app:8080 weight=90;
}

# 按 URL 前缀路由
location /api/ {
    # 新功能路径
    location /api/v2/orders {
        proxy_pass http://new-service;
    }
    location /api/v2/products {
        proxy_pass http://new-service;
    }
    # 其余仍走旧系统
    try_files $uri @legacy;
}

location @legacy {
    proxy_pass http://old-service;
}

# 按 Header 灰度路由（用于测试验证）
map $http_x_migration $backend {
    default "old-service";
    "v2"    "new-service";
}

server {
    location /api/ {
        proxy_pass http://$backend;
    }
}
```

### 基于 Spring Cloud Gateway 的动态路由

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
        # 新系统路由
        - id: order-service-v2
          uri: lb://order-service-v2
          predicates:
            - Path=/api/v2/orders/**
          filters:
            - StripPrefix=1
        # 旧系统路由（兜底）
        - id: legacy-app
          uri: lb://legacy-app
          predicates:
            - Path=/api/**
```

### 基于 Cookie/Header 的灰度控制

```java
// 使用 Cookie 判断是否走新系统
@Bean
public RouteLocator customRoutes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("order-v2-gray", r -> r
            .path("/api/orders/**")
            .and().header("X-Migration", "v2")
            .uri("lb://new-order-service"))
        .route("order-legacy", r -> r
            .path("/api/orders/**")
            .uri("lb://legacy-app"))
        .build();
}
```

## 3. 数据库迁移策略

### 双写模式（Dual Write）

```java
@Service
public class OrderService {
    
    @Transactional
    public void createOrder(Order order) {
        // 先写旧库（保证兼容）
        legacyOrderRepo.save(order);
        
        // 再写新库（异步，失败不影响主流程）
        try {
            newOrderRepo.save(order);
        } catch (Exception e) {
            log.warn("新库写入失败，由数据同步任务补偿", e);
        }
    }
}
```

### 数据同步方案

```bash
# 使用 Canal + Debezium 监听 binlog 同步
# docker-compose 部署 Canal Server
docker run -d --name canal-server \
  -e canal.instance.master.address=legacy-db:3306 \
  -e canal.instance.dbUsername=canal \
  -e canal.instance.dbPassword=canal \
  canal/canal-server:latest
```

### 读流量切换步骤

```yaml
# 1. 第一阶段：新库只读，老库读写
# 2. 第二阶段：重要读切到新库，写全量双写
# 3. 第三阶段：新库读写，老库只读
# 4. 第四阶段：完全切到新库，下线老库
```

## 4. 切分步骤实战

### 示例：电商单体拆分为订单服务

```
Phase 1 — 创建新服务，只处理新订单
  ├─ 路由：新订单路径 -> order-service-v2
  ├─ 数据：新建订单数据库
  └─ 回滚：改路由即可

Phase 2 — 迁移「查询已发货订单」功能
  ├─ 路由：查询接口分流
  ├─ 数据：双写同步历史订单
  └─ 验证：对比新旧接口结果

Phase 3 — 迁移「创建订单」功能
  ├─ 路由：POST /api/orders -> 新服务
  ├─ 数据：新服务写新库，同步旧库
  └─ 验证：业务闭环测试

Phase 4 — 下线旧订单模块
  ├─ 确认所有调用方已迁移
  ├─ 清理旧代码
  └─ 回收旧数据库资源
```

## 5. 集成测试策略

### 新旧结果对比验证

```java
@Component
public class MigrationVerifier {
    
    private final LegacyClient legacyClient;
    private final NewClient newClient;
    
    @Scheduled(fixedDelay = 5000)
    public void compareResults() {
        // 模拟请求，同时发给新老系统
        // 对比 JSON 响应差异
        String legacyResp = legacyClient.queryOrders("test-user");
        String newResp = newClient.queryOrders("test-user");
        
        double diffRatio = compareJsonDiff(legacyResp, newResp);
        if (diffRatio > 0.01) {
            alertService.sendWarning("差异超过1%: " + diffRatio);
        }
    }
}
```

### 流量回放测试

```bash
# 录制生产流量
# 使用 goreplay 或 tcpcopy
gor --input-raw :8080 --output-file requests.gor

# 回放到新系统
gor --input-file requests.gor --output-http http://new-service:8080
```

## 注意事项

### ⚠️ 事务一致性问题
- 双写阶段可能产生数据不一致，必须实现**补偿机制**
- 建议使用消息队列确保最终一致性：`旧库写入 → 发 MQ → 新库消费写入`
- 定期运行数据对账任务，修复差异数据

### ⚠️ 调用链追踪
- 新旧系统之间的调用需要传递 Trace ID，方便问题排查
- 在 Header 中添加 `X-Migration-Trace-Id` 串联完整链路

### ⚠️ 回滚策略
- **每次发布必须包含回滚方案**（改路由或切 DNS）
- 保持旧系统代码不被删除至少 2 个迭代周期
- 记录每个功能的「迁移状态」元数据

### ⚠️ 渐进式测试
- 先用内部管理后台试用新系统（5% 流量）
- 扩展到特定用户群（按商户/地域灰度）
- 全量切换前需压测到 150% 的预期峰值

### ⚠️ 不要同时拆太多
- 一个迭代周期只拆 1-2 个功能模块
- 每完成一次切分就「稳定运行 1-2 周」再继续
- 拆得太碎会导致分布式事务爆炸
