---
name: nacos-config-listener
description: Nacos 配置变更监听与自动刷新高级模式——@RefreshScope、监听器、灰度配置、配置总线
tags: [spring-cloud, nacos, config, listener, refresh, dynamic-config]
---

## 概述

Nacos 配置中心的核心能力是**动态配置管理**——配置变更后自动推送到应用并生效。本文覆盖从基础 `@RefreshScope` 到企业级配置总线的完整监听模式。

## 一、@RefreshScope 自动刷新

最基础的动态配置方式，Spring Cloud 原生支持。

### 配置

```yaml
# application.yml
spring:
  application:
    name: order-service
  config:
    import: nacos:order-service.yml?group=DEFAULT_GROUP&refreshEnabled=true
  cloud:
    nacos:
      config:
        refresh-enabled: true  # 启用自动刷新
```

### Java 使用

```java
@Component
@RefreshScope  // ✔ 标记为可刷新的 Bean
public class DynamicConfig {

    @Value("${order.timeout:30000}")
    private int orderTimeout;

    @Value("${order.max.retry:3}")
    private int maxRetry;

    // 每次访问时都会检查配置是否已变更
    public int getOrderTimeout() {
        return orderTimeout;
    }
}
```

### 注意事项

| 限制 | 说明 | 解决方案 |
|------|------|---------|
| **仅适用于 Spring Bean** | `@Value` 直接在字段上使用 `@RefreshScope` 才刷新 | 注入到 `@RefreshScope` Bean 中 |
| **不刷新方法内嵌值** | 构造方法注入的值不会刷新 | 用 `@Value` 字段 + getter |
| **不刷新已存在的对象** | 集合、Map 等复杂对象需重新获取 | 用 `EnvironmentAware` 或监听器 |
| **性能开销** | 每次访问都检查配置版本号 | 高频访问的场景注意优化 |

## 二、EnvironmentChangeEvent 监听

监听配置变更事件，执行自定义回调逻辑（如重新初始化连接池、清除缓存等）。

```java
@Component
public class ConfigChangeLogger {

    private static final Set<String> WATCHED_KEYS = Set.of(
        "order.timeout", "order.max.retry", "datasource.url"
    );

    @EventListener
    public void onConfigChange(EnvironmentChangeEvent event) {
        Set<String> changedKeys = event.getKeys();
        changedKeys.stream()
            .filter(WATCHED_KEYS::contains)
            .forEach(key -> {
                String newValue = environment.getProperty(key);
                log.info("配置变更: {} = {}", key, newValue);
                handleConfigChange(key, newValue);
            });
    }

    private void handleConfigChange(String key, String value) {
        switch (key) {
            case "datasource.url" -> refreshDataSource(value);      // 重建数据源
            case "order.timeout" -> updateHttpClientTimeout(value); // 更新 HTTP 客户端
            case "order.max.retry" -> updateRetryPolicy(value);     // 更新重试策略
        }
    }

    @Autowired
    private Environment environment;
}
```

## 三、Nacos 原生监听器（无 Spring Cloud）

如果不使用 Spring Cloud，或需要更底层的控制，可以直接使用 Nacos 客户端 API。

```java
@Component
public class NacosNativeListener implements ApplicationRunner {

    @Value("${nacos.server-addr:127.0.0.1:8848}")
    private String serverAddr;

    @Override
    public void run(ApplicationArguments args) {
        ConfigService configService = NacosFactory.createConfigService(serverAddr);

        // 注册监听器：监听指定 DataId + Group
        configService.addListener("order-service.yml", "DEFAULT_GROUP", new Listener() {

            @Override
            public Executor getExecutor() {
                // 建议使用独立线程池，避免阻塞 Nacos 长轮询
                return Executors.newSingleThreadExecutor(r -> {
                    Thread t = new Thread(r, "nacos-config-listener");
                    t.setDaemon(true);
                    return t;
                });
            }

            @Override
            public void receiveConfigInfo(String configInfo) {
                log.info("收到配置变更: {}", configInfo);
                // 解析 YAML/Properties，刷新业务配置
                refreshConfig(configInfo);
            }
        });
    }
}
```

### 监听器线程池注意事项

```
Nacos 长轮询连接 ← 独立线程（Nacos 内部）
                     ↓
                 监听器回调 ← getExecutor() 返回的线程池
                     ↓
                 业务逻辑处理（可阻塞）
```

- **切勿**在监听器中执行长时间阻塞操作（会阻塞 Nacos 长轮询连接）
- **推荐**使用独立的线程池处理配置变更
- **保证幂等**：Nacos 可能重复推送配置变更

## 四、灰度配置监听

Nacos 支持配置的灰度发布（Beta 发布），监听器同样需要感知灰度版本。

```java
@Component
public class GrayConfigService {

    @Autowired
    private NacosConfigManager configManager;

    /**
     * 获取灰度配置（优先读取灰度版本）
     */
    public String getGrayConfig(String dataId, String group, String grayRule) {
        ConfigService configService = configManager.getConfigService();
        try {
            // 获取灰度配置：传入灰度规则表达式
            return configService.getConfig(dataId, group, 5000, grayRule);
        } catch (NacosException e) {
            log.error("获取灰度配置失败", e);
            return null;
        }
    }

    /**
     * 监听灰度配置
     */
    public void listenGrayConfig(String dataId, String group) {
        ConfigService configService = configManager.getConfigService();
        configService.addListener(dataId, group, new Listener() {
            @Override
            public Executor getExecutor() {
                return Executors.newSingleThreadExecutor();
            }

            @Override
            public void receiveConfigInfo(String configInfo) {
                // 配置中可能包含灰度规则信息
                log.info("灰度配置变更: {}", configInfo);
            }
        });
    }
}
```

```yaml
# Nacos 控制台发布灰度配置时指定：
# 灰度规则：users = user_1001, user_1002 或 IP = 192.168.1.*
# 使得只有灰度用户看到新配置
```

## 五、多环境配置变更管理

### 配置变更传播（配置总线模式）

```java
/**
 * 配置变更总线：通过 Redis Pub/Sub 广播配置变更到集群中所有实例
 * 确保配置在各实例间一致生效
 */
@Component
public class ConfigChangeBus {

    @Autowired
    private RedisTemplate<String, String> redis;

    @Autowired
    private ConfigChangeService configChangeService;

    private static final String BUS_CHANNEL = "config:change:bus";

    @PostConstruct
    public void init() {
        // 订阅配置变更广播
        redis.listenToChannel(BUS_CHANNEL, (message, channel) -> {
            ConfigChangeEvent event = JsonUtil.parse(message, ConfigChangeEvent.class);
            log.info("收到配置变更广播: dataId={}, version={}", event.getDataId(), event.getVersion());
            configChangeService.reloadConfig(event.getDataId());
        });
    }

    /**
     * 发布配置变更到全集群
     */
    public void publishChange(String dataId, String group, String content) {
        ConfigChangeEvent event = ConfigChangeEvent.builder()
            .dataId(dataId)
            .group(group)
            .content(content)
            .timestamp(System.currentTimeMillis())
            .version(UUID.randomUUID().toString())  // 去重用
            .build();

        redis.convertAndSend(BUS_CHANNEL, JsonUtil.toJson(event));
    }
}
```

## 六、动态数据源切换（高级场景）

配置变更后动态重建 Druid/HikariCP 连接池。

```java
@Component
public class DynamicDataSourceRefresher {

    @Autowired
    private DataSource dataSource;

    @EventListener
    public void onDataSourceConfigChange(EnvironmentChangeEvent event) {
        if (!event.getKeys().stream().anyMatch(k -> k.startsWith("spring.datasource"))) {
            return;
        }

        // 关闭旧连接池
        if (dataSource instanceof HikariDataSource hikariDS) {
            hikariDS.close();
        }

        // 重新构建数据源（需从 Environment 读取新配置）
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(environment.getProperty("spring.datasource.url"));
        config.setUsername(environment.getProperty("spring.datasource.username"));
        config.setPassword(environment.getProperty("spring.datasource.password"));
        config.setMaximumPoolSize(
            Integer.parseInt(environment.getProperty("spring.datasource.maximum-pool-size", "10"))
        );

        this.dataSource = new HikariDataSource(config);
        log.info("数据源已重建");
    }

    @Autowired
    private Environment environment;
}
```

## 七、配置变更审计（生产推荐）

```java
@Component
public class ConfigChangeAuditListener {

    @EventListener
    public void audit(EnvironmentChangeEvent event) {
        String appName = environment.getProperty("spring.application.name");
        String instanceId = InetAddress.getLocalHost().getHostAddress();

        event.getKeys().forEach(key -> {
            String newValue = environment.getProperty(key);
            // 写入审计日志（数据库或 ELK）
            auditLogRepository.save(ConfigChangeLog.builder()
                .appName(appName)
                .instanceId(instanceId)
                .configKey(key)
                .newValue(maskSensitive(key, newValue))
                .changedAt(Instant.now())
                .build());
        });
    }
}
```

## 注意事项

| 场景 | 问题 | 解决方案 |
|------|------|---------|
| **反复刷新** | 多个配置同时变更触发多次刷新 | 用 `debounce` 合并短时间内的刷新请求 |
| **配置回滚** | 回滚后旧值可能与新值冲突 | 监听器处理 `EnvironmentChangeEvent` 中的旧值 |
| **监听器异常** | 单个监听器抛出异常影响其他监听器 | 每个监听器单独 try-catch |
| **配置与代码兼容** | 新配置需要新代码逻辑配合 | 使用 `@ConditionalOnProperty` 渐进式部署 |
| **长轮询超时** | 大量配置导致长轮询超时 | 拆分配置到多个 DataId，控制单文件大小 |

### 防抖合并示例

```java
@Component
public class DebouncedConfigRefresher {

    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
    private volatile ScheduledFuture<?> pendingRefresh;

    // 2 秒内的多次配置变更合并为一次刷新
    public void scheduleRefresh(String source) {
        if (pendingRefresh != null) {
            pendingRefresh.cancel(false);
        }
        pendingRefresh = scheduler.schedule(() -> {
            log.info("批量刷新配置（来源: {}）", source);
            refreshAll();
        }, 2, TimeUnit.SECONDS);
    }
}
```
