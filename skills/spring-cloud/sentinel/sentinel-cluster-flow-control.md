---
name: sentinel-cluster-flow-control
description: Sentinel 集群流控实战：Token Server/Client 架构、集群规则配置、生产部署与性能调优
tags: [spring-cloud, sentinel, cluster, flow-control, token-server, rate-limiting]
---

## 概述

Sentinel 集群流控（Cluster Flow Control）解决单机限流无法精确控制集群总流量的痛点，通过 Token Server 集中管理令牌，实现**集群维度的精确限流**。适用于网关层总 QPS 控制、共享资源池保护、多实例统一限流等场景。

## 1. 集群流控架构

### 核心组件

```
┌─────────────────────────────────────┐
│         Token Server (独立/嵌入)       │
│   ┌─────────────────────────────┐   │
│   │  全局规则管理 + 令牌计算       │   │
│   │  ClusterFlowRuleManager     │   │
│   └─────────────────────────────┘   │
└────────────┬────────────────────────┘
             │ TCP/HTTP 通信
    ┌────────┼────────┬────────┐
    ▼        ▼        ▼        ▼
 Token  Token Client  Token  Token
Client  (嵌入应用)   Client  Client
(网关)               (服务A)  (服务B)
```

### 通信模式

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| **独立模式** | Token Server 部署为独立进程/应用 | 生产环境推荐，Server 故障不影响业务 |
| **嵌入模式** | Token Server 嵌入某个应用进程 | 开发/测试环境，节省资源 |
| **动态切换** | 运行时切换 Token Client 连接的 Server | 运维操作，无缝迁移 |

## 2. Token Server 部署

### 2.1 独立模式（生产推荐）

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-cluster-server-default</artifactId>
    <version>1.8.10</version>
</dependency>
```

```java
// Token Server 启动类
public class SentinelTokenServer {

    public static void main(String[] args) throws Exception {
        // 加载集群流控规则
        ClusterServerConfigManager.loadGlobalTransportConfig(
            new ServerTransportConfig()
                .setPort(18730)           // Token Server 监听端口
                .setIdleSeconds(600)      // 连接空闲超时
        );

        // 初始化命名空间
        ClusterServerConfigManager.loadServerNamespaceSet(
            Collections.singleton("default")
        );

        // 启动 Token Server
        ClusterTokenServer tokenServer = new SentinelDefaultTokenServer();
        tokenServer.start();

        // 加载规则（从 Nacos 或其他数据源）
        loadClusterRules();

        System.out.println("Sentinel Token Server started on port 18730");
        Thread.currentThread().join();
    }

    private static void loadClusterRules() {
        List<FlowRule> flowRules = new ArrayList<>();

        FlowRule rule = new FlowRule("order_api")
            .setClusterMode(true)                    // 开启集群模式
            .setClusterConfig(new ClusterFlowConfig()
                .setFlowId(10001L)                   // 集群规则 ID（全局唯一）
                .setThresholdType(1)                 // 0=全局阈值, 1=均摊阈值
                .setFallbackToLocalWhenFail(true)    // 连接失败时回退单机限流
            );

        // thresholdType=0(全局阈值): 所有实例共享总 QPS 阈值
        // thresholdType=1(均摊阈值): 阈值按实例数均摊

        rule.setCount(1000);  // 集群总 QPS=1000 或 每实例 QPS=1000/n
        rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_DEFAULT);

        flowRules.add(rule);
        FlowRuleManager.loadRules(flowRules);
    }
}
```

### 2.2 Nacos 动态配置（推荐）

```json
// Nacos dataId: sentinel-cluster-flow-rules.json
[
    {
        "resource": "order_api",
        "count": 1000.0,
        "grade": 1,
        "limitApp": "default",
        "clusterMode": true,
        "clusterConfig": {
            "flowId": 10001,
            "thresholdType": 0,
            "fallbackToLocalWhenFail": true,
            "strategy": 0,
            "sampleCount": 10,
            "windowIntervalMs": 1000
        }
    },
    {
        "resource": "payment_api",
        "count": 500.0,
        "grade": 1,
        "limitApp": "default",
        "clusterMode": true,
        "clusterConfig": {
            "flowId": 10002,
            "thresholdType": 1,
            "fallbackToLocalWhenFail": true,
            "strategy": 0,
            "sampleCount": 10,
            "windowIntervalMs": 1000
        }
    }
]
```

### 2.3 Docker 部署 Token Server

```dockerfile
# Dockerfile
FROM eclipse-temurin:21-jre
COPY sentinel-token-server.jar /app/
COPY application.yml /app/
WORKDIR /app
EXPOSE 18730
ENTRYPOINT ["java", "-jar", "sentinel-token-server.jar"]
```

```yaml
# docker-compose.yml
services:
  sentinel-token-server:
    build: .
    ports:
      - "18730:18730"
    environment:
      - NACOS_ADDR=nacos:8848
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:18730/health"]
      interval: 30s
      timeout: 3s
      retries: 3
```

## 3. Token Client 配置（嵌入业务应用）

### 3.1 依赖引入

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-cluster-client-default</artifactId>
    <version>1.8.10</version>
</dependency>
```

### 3.2 Spring Boot 配置

```yaml
spring:
  cloud:
    sentinel:
      enabled: true
      transport:
        dashboard: localhost:8080
      # 集群流控客户端配置
      cluster:
        server:
          # Token Server 地址列表
          host: 192.168.1.10
          port: 18730
        # 连接 Token Server 的超时
        connect-timeout: 3000
        # 请求 Token 超时（毫秒）
        request-timeout: 500
        # 底层连接池
        max-connections: 10
```

### 3.3 客户端初始化

```java
@Configuration
public class SentinelClusterClientConfig {

    @PostConstruct
    public void initClusterClient() {
        initPropertySource();

        ClusterClientConfig clientConfig = new ClusterClientConfig();
        clientConfig.setRequestTimeout(500);   // 请求令牌超时 500ms

        // 配置 Token Server 地址
        ClusterClientConfigManager.applyClientConfig(clientConfig);

        // 启动客户端
        ClusterTokenClient tokenClient = new SentinelClusterTokenClient(
            "192.168.1.10",   // Token Server host
            18730              // Token Server port
        );
        tokenClient.start();

        // 注册命名空间
        ClusterClientConfigManager.applyServerNamespaceSet(
            Collections.singleton("default")
        );
    }
}
```

### 3.4 使用 @SentinelResource 配合集群规则

```java
@RestController
public class OrderController {

    @GetMapping("/api/orders")
    @SentinelResource(value = "order_api", blockHandler = "clusterBlockHandler")
    public Result<List<Order>> listOrders(@RequestParam Long userId) {
        return orderService.getOrders(userId);
    }

    public Result<List<Order>> clusterBlockHandler(Long userId, BlockException ex) {
        return Result.error(429, "订单服务当前请求量过大，请稍后重试")
            .putHeader("X-RateLimit-Type", "cluster");
    }
}
```

## 4. 生产部署最佳实践

### 4.1 Token Server 高可用

```java
// Token Client 多 Server 配置（故障切换）
@Configuration
public class ClusterClientHAConfig {

    @Value("${sentinel.cluster.server.list:}")
    private String serverList; // 格式: 192.168.1.10:18730,192.168.1.11:18730

    @PostConstruct
    public void initHA() {
        if (StringUtils.isEmpty(serverList)) {
            return;
        }

        // 解析 Server 地址列表
        String[] servers = serverList.split(",");
        for (String server : servers) {
            String[] parts = server.split(":");
            String host = parts[0];
            int port = Integer.parseInt(parts[1]);

            // 添加为备用 Server
            ClusterClientConfigManager.applyServerNamespaceSet(
                Collections.singleton("default")
            );
        }

        // 客户端自动重连
        ClusterClientConfig clientConfig = new ClusterClientConfig();
        clientConfig.setRequestTimeout(500);
        // 重连间隔（毫秒）
        clientConfig.setReconnectIntervalMs(10000);
        ClusterClientConfigManager.applyClientConfig(clientConfig);
    }
}
```

```yaml
# 多 Token Server 配置
sentinel:
  cluster:
    server:
      list: 192.168.1.10:18730,192.168.1.11:18730,192.168.1.12:18730
    connect-timeout: 3000
    request-timeout: 500
```

### 4.2 本地降级策略

当 Token Server 不可用时自动回退到单机限流：

```yaml
spring:
  cloud:
    sentinel:
      cluster:
        # Token Server 不可用时是否回退单机限流
        fallback-to-local-when-fail: true
        # 本地回退的阈值（QPS）
        local-threshold: 200
```

```java
// 动态调整降级阈值
ClusterFlowRule rule = new ClusterFlowRule("order_api")
    .setCount(1000)                              // 集群总 QPS
    .setClusterConfig(new ClusterFlowConfig()
        .setFlowId(10001L)
        .setThresholdType(0)
        .setFallbackToLocalWhenFail(true)        // 回退到单机
        // 回退单机阈值（默认 1000/实例数）
        .setLocalThresholdForFallback(200)
    );
```

### 4.3 阈值类型选择

```java
// 方案一：全局阈值（thresholdType=0）
// 集群总 QPS = 1000，5 个实例均分约 200 QPS/实例
FlowRule globalRule = new FlowRule("api")
    .setCount(1000)
    .setClusterConfig(new ClusterFlowConfig()
        .setFlowId(1L)
        .setThresholdType(0));

// 方案二：均摊阈值（thresholdType=1）
// 每实例阈值 = 200，实例数变动时总阈值自动调整
FlowRule averageRule = new FlowRule("api")
    .setCount(200)
    .setClusterConfig(new ClusterFlowConfig()
        .setFlowId(2L)
        .setThresholdType(1));

// 方案三：混合模式
// 网关层：全局阈值（严格控流）
// 服务层：均摊阈值（弹性伸缩）
```

### 4.4 预热与冷启动

```java
// 集群流控支持预热 WarmUp，防止流量突增击穿
FlowRule warmUpRule = new FlowRule("cold_start_api")
    .setCount(2000)
    .setClusterMode(true)
    .setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_WARM_UP)
    .setWarmUpPeriodSec(30)       // 预热时间 30 秒
    .setClusterConfig(new ClusterFlowConfig()
        .setFlowId(10003L)
        .setThresholdType(0));
```

## 5. 监控与运维

### 5.1 Token Server 指标

```java
// 获取 Token Server 统计信息
public Map<String, Object> getTokenServerStats() {
    Map<String, Object> stats = new HashMap<>();

    // 活跃连接数
    stats.put("connections", 
        ClusterTokenServerManager.getInstance().getConnectionCount());

    // 请求频率
    stats.put("requestQps", 
        ClusterTokenServerStatManager.getRequestQps());

    // 拒绝请求数
    stats.put("blockedCount", 
        ClusterTokenServerStatManager.getBlockedCount());

    // 本地 fallback 次数
    stats.put("fallbackCount", 
        ClusterTokenServerStatManager.getFallbackCount());

    return stats;
}
```

### 5.2 健康检查端点

```java
@RestController
@RequestMapping("/internal/sentinel")
public class SentinelHealthController {

    @GetMapping("/cluster/health")
    public Result health() {
        ClusterTokenClient client = ClusterTokenClientManager.getInstance();
        return Result.ok()
            .put("serverConnected", client != null && client.isConnected())
            .put("serverHost", ClusterClientConfigManager.getServerHost())
            .put("serverPort", ClusterClientConfigManager.getServerPort())
            .put("mode", ClusterClientConfigManager.getMode().name());
    }

    @GetMapping("/cluster/stats")
    public Result stats() {
        // 查看各集群规则的统计情况
        Map<Long, ClusterMetric> metrics = 
            ClusterTokenServerStatManager.getMetrics();
        return Result.ok(metrics);
    }
}
```

### 5.3 Dashboard 监控

Sentinel Dashboard 1.8.10+ 支持集群流控监控：
- 查看 Token Server 连接状态
- 查看集群 QPS、拒绝数、延迟
- 动态调整集群规则
- 查看各 Client 的 Token 请求分布

## 6. 常见问题与避坑

### 6.1 Token Server 单点故障

```yaml
# ✅ 正确：多 Token Server + 客户端自动切换
sentinel:
  cluster:
    server:
      list: 192.168.1.10:18730,192.168.1.11:18730,192.168.1.12:18730
    fallback-to-local-when-fail: true  # ✅ 开启本地回退

# ❌ 错误：只配置一个 Server 且不允许回退
# fallback-to-local-when-fail: false   # Server 挂了整个集群限流失效
```

### 6.2 流控规则 ID 冲突

```java
// ❌ 不同资源的 FlowId 相同 → 规则互相覆盖
// ✅ 不同资源的 FlowId 必须全局唯一
// 建议：使用资源名称的 hash 作为 flowId
long flowId = Math.abs("order_api".hashCode()) % 100000 + 10000;
long flowId2 = Math.abs("payment_api".hashCode()) % 100000 + 10000;

// 或统一分配 ID 范围
// 流控规则: 10001-19999
// 热点规则: 20001-29999
// 降级规则: 30001-39999
```

### 6.3 网络延迟敏感

```java
// ❌ requestTimeout 过大 → 请求堆积
clientConfig.setRequestTimeout(5000); // 5秒，阻塞业务线程

// ✅ requestTimeout 建议 200-500ms
clientConfig.setRequestTimeout(200);
// 配合 fallbackToLocalWhenFail=true 超时后降级到本地
```

### 6.4 阈值类型选择误区

```
场景：4 个实例，期望集群总 QPS = 4000

❌ thresholdType=1, count=4000
   每个实例独立计算 4000 QPS → 集群总 QPS 可达 16000，超预期 4 倍

✅ thresholdType=0, count=4000
   Token Server 统一控制，集群总 QPS 精确为 4000

✅ thresholdType=1, count=1000
   每实例 1000 QPS，集群总计 4000 QPS
   注意：实例数变为 5 时，集群总 QPS 变为 5000
```

### 6.5 Token Server 资源估算

| 实例数 | 总 QPS | 建议 Token Server 规格 | 网络带宽 |
|--------|--------|----------------------|----------|
| 1-10 | < 5000 | 1C2G | 100Mbps |
| 10-50 | 5000-50000 | 2C4G | 1Gbps |
| 50-200 | 50000-200000 | 4C8G × 2 (HA) | 1Gbps |
| 200+ | 200000+ | 8C16G × 3 (集群) | 10Gbps |
