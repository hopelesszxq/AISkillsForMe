---
name: feign-cache-async
description: OpenFeign 响应缓存与异步调用模式：Spring Cache 集成、Caffeine 本地缓存、CompletableFuture 异步调用、批量合并请求
tags: [spring-cloud, feign, cache, async, performance, microservice]
---

## 概述

微服务间 Feign 调用的性能优化两大方向：**响应缓存**（减少重复请求）和 **异步调用**（释放调用线程）。本文覆盖 Feign 与 Spring Cache 集成、本地缓存策略、CompletableFuture 异步调用的最佳实践。

## 1. Feign + Spring Cache 集成

### 1.1 基础配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

```yaml
spring:
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=1000,expireAfterWrite=30s,recordStats
```

### 1.2 Feign 接口缓存

```java
@FeignClient(name = "user-service", path = "/users")
public interface UserClient {

    @GetMapping("/{id}")
    @Cacheable(value = "users", key = "#id", unless = "#result == null")
    UserDTO getUser(@PathVariable Long id);

    @GetMapping("/batch")
    @Cacheable(value = "users", key = "#ids.hashCode()")
    List<UserDTO> getUsers(@RequestParam("ids") List<Long> ids);

    @GetMapping("/{id}/profile")
    @Cacheable(value = "userProfiles", key = "#id", unless = "#result == null")
    UserProfileDTO getUserProfile(@PathVariable Long id);
}

// 缓存失效——数据变更时主动清除缓存
@FeignClient(name = "user-service", path = "/users")
public interface UserAdminClient {

    @PutMapping("/{id}")
    @CacheEvict(value = "users", key = "#id")
    UserDTO updateUser(@PathVariable Long id, @RequestBody UserUpdateReq req);

    @DeleteMapping("/{id}")
    @CacheEvict(value = "users", key = "#id")
    void deleteUser(@PathVariable Long id);

    @PostMapping("/batch")
    @CacheEvict(value = "users", allEntries = true)  // 批量操作清空整个缓存
    List<UserDTO> batchCreate(@RequestBody List<UserCreateReq> reqs);
}
```

### 1.3 缓存 TTL 策略

```java
@Configuration
public class FeignCacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        
        // 不同 Cache 配置不同 TTL
        Map<String, Caffeine<Object, Object>> cacheConfigs = Map.of(
            "users", Caffeine.newBuilder()
                .maximumSize(5000)
                .expireAfterWrite(30, TimeUnit.SECONDS)
                .recordStats(),
            "userProfiles", Caffeine.newBuilder()
                .maximumSize(2000)
                .expireAfterWrite(5, TimeUnit.MINUTES)  // 用户画像更新不频繁
                .recordStats(),
            "orders", Caffeine.newBuilder()
                .maximumSize(2000)
                .expireAfterWrite(10, TimeUnit.SECONDS)  // 订单状态变化快，短 TTL
                .recordStats()
        );

        cacheConfigs.forEach((name, caffeine) -> {
            CaffeineCache cache = new CaffeineCache(name, caffeine.build());
            manager.registerCustomCache(name, cache.getNativeCache());
        });

        return manager;
    }
}
```

### 1.4 缓存统计与监控

```java
@Component
public class CacheMonitor {

    private final CacheManager cacheManager;

    public CacheMonitor(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    @Scheduled(fixedRate = 60000)  // 每分钟输出缓存统计
    public void reportCacheStats() {
        for (String name : cacheManager.getCacheNames()) {
            CaffeineCache cache = (CaffeineCache) cacheManager.getCache(name);
            if (cache != null) {
                CacheStats stats = cache.getNativeCache().stats();
                double hitRate = stats.hitRate() * 100;
                log.info("Cache[{}] hitRate={}%, hitCount={}, missCount={}, evictions={}",
                    name, String.format("%.1f", hitRate),
                    stats.hitCount(), stats.missCount(), stats.evictionCount());
            }
        }
    }
}

// Actuator 端点暴露缓存信息
management:
  endpoints:
    web:
      exposure:
        include: cache
  cache:
    stats: true
```

## 2. Feign 装饰器模式——缓存拦截器

### 2.1 自定义缓存拦截器（不依赖 Spring Cache）

适合不需要全局 CacheManager 的轻量场景：

```java
public class CachingFeignInterceptor implements RequestInterceptor {

    private final Cache<String, String> cache;
    private final Set<String> cacheableMethods;

    public CachingFeignInterceptor() {
        this.cache = Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(30, TimeUnit.SECONDS)
            .build();

        // 只对 GET 请求缓存
        this.cacheableMethods = Set.of("GET");
    }

    @Override
    public void apply(RequestTemplate template) {
        if (!cacheableMethods.contains(template.method())) {
            return;
        }

        String cacheKey = buildCacheKey(template);
        String cachedResponse = cache.getIfPresent(cacheKey);

        if (cachedResponse != null) {
            // 将缓存结果注入到请求上下文中
            template.header("X-Feign-Cache", "HIT");
            template.header("X-Feign-Cache-Key", cacheKey);
        }
    }

    private String buildCacheKey(RequestTemplate template) {
        return template.method() + ":" + template.url() 
             + ":" + template.queries();
    }
}
```

### 2.2 响应缓存装饰器

```java
@Component
public class CachingFeignDecorator {

    private final Cache<String, Object> responseCache;

    public CachingFeignDecorator() {
        this.responseCache = Caffeine.newBuilder()
            .maximumSize(5000)
            .expireAfterWrite(30, TimeUnit.SECONDS)
            .recordStats()
            .build();
    }

    /**
     * 使用缓存包装 Feign 调用
     * 适用：getUser、getOrder 等读接口
     */
    @SuppressWarnings("unchecked")
    public <T> T cacheable(String cacheKey, Class<T> type, Supplier<T> feignCall) {
        Object cached = responseCache.getIfPresent(cacheKey);
        if (cached != null) {
            log.debug("Feign cache HIT: key={}", cacheKey);
            return (T) cached;
        }

        T result = feignCall.get();
        if (result != null) {
            responseCache.put(cacheKey, result);
        }
        return result;
    }

    /** 手动失效缓存 */
    public void evict(String cacheKey) {
        responseCache.invalidate(cacheKey);
    }

    /** 按前缀批量失效（如 evictByPrefix("users:")） */
    public void evictByPrefix(String keyPrefix) {
        responseCache.asMap().keySet().removeIf(k -> k.startsWith(keyPrefix));
    }
}

// 使用示例
@Service
public class UserService {

    private final UserClient userClient;
    private final CachingFeignDecorator cacheDecorator;

    public UserDTO getUser(Long id) {
        String cacheKey = "users:" + id;
        return cacheDecorator.cacheable(cacheKey, UserDTO.class, 
            () -> userClient.getUser(id));
    }
}
```

## 3. Feign 异步调用模式

### 3.1 异步 Feign 客户端

```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-reactive-wrappers</artifactId>
    <version>13.5</version>
</dependency>
```

```java
// 使用 CompletableFuture 包装 Feign 调用
@FeignClient(name = "order-service")
public interface OrderAsyncClient {

    @GetMapping("/orders/{id}")
    CompletableFuture<OrderDTO> getOrder(@PathVariable Long id);

    @GetMapping("/orders/user/{userId}")
    CompletableFuture<List<OrderDTO>> getUserOrders(@PathVariable Long userId);

    @PostMapping("/orders")
    CompletableFuture<OrderDTO> createOrder(@RequestBody CreateOrderReq req);
}

// 如果使用 Spring Cloud 2024.0+ 原生异步支持
@FeignClient(name = "inventory-service", async = true)
public interface InventoryAsyncClient {

    @GetMapping("/stock/{sku}")
    CompletableFuture<StockVO> getStock(@PathVariable String sku);

    @PutMapping("/stock/deduct")
    CompletableFuture<Void> deductStock(@RequestBody DeductReq req);
}
```

### 3.2 并行调用——聚合多个服务

```java
@Service
public class OrderAggregationService {

    private final OrderAsyncClient orderClient;
    private final UserClient userClient;           // 同步
    private final InventoryAsyncClient inventoryClient;
    private final PaymentClient paymentClient;       // 同步

    // 自定义线程池
    @Bean("feignAsyncPool")
    public Executor feignAsyncPool() {
        return new ThreadPoolExecutor(
            8, 20, 60L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(200),
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }

    @Async("feignAsyncPool")
    public CompletableFuture<OrderDetailVO> getOrderDetail(Long orderId) {
        // 并行调用多个服务
        CompletableFuture<OrderDTO> orderFuture = 
            orderClient.getOrder(orderId);
        CompletableFuture<List<OrderItemVO>> itemsFuture = 
            CompletableFuture.supplyAsync(() -> 
                orderItemClient.getItems(orderId), feignAsyncPool());

        return orderFuture.thenCombine(itemsFuture, (order, items) -> {
            // 串行依赖：拿到 order 后才能查询 user 和 payment
            CompletableFuture<UserDTO> userFuture = 
                CompletableFuture.supplyAsync(() -> 
                    userClient.getUser(order.getUserId()), feignAsyncPool());
            CompletableFuture<PaymentDTO> paymentFuture = 
                CompletableFuture.supplyAsync(() -> 
                    paymentClient.getPayment(order.getPaymentId()), feignAsyncPool());

            // 等待所有并行任务完成
            return CompletableFuture.allOf(userFuture, paymentFuture)
                .thenApply(v -> {
                    UserDTO user = userFuture.join();
                    PaymentDTO payment = paymentFuture.join();
                    return new OrderDetailVO(order, user, payment, items);
                });
        }).thenCompose(f -> f);
    }
}
```

### 3.3 批量请求合并（Batch Collapsing）

```java
@Component
public class BatchUserClient {

    private final UserClient userClient;
    private final Cache<Long, UserDTO> userCache;

    // 待合并的请求队列
    private final ScheduledExecutorService scheduler = 
        Executors.newSingleThreadScheduledExecutor();

    // 批量合并配置
    @Value("${feign.batch.max-size:50}")
    private int maxBatchSize;

    @Value("${feign.batch.window-ms:10}")
    private int windowMs;

    public BatchUserClient(UserClient userClient) {
        this.userClient = userClient;
        this.userCache = Caffeine.newBuilder()
            .maximumSize(5000)
            .expireAfterWrite(30, TimeUnit.SECONDS)
            .build();

        // 定时清理解放待合并队列
        scheduler.scheduleAtFixedRate(this::flushBatch, 
            windowMs, windowMs, TimeUnit.MILLISECONDS);
    }

    private final Queue<PendingRequest<UserDTO>> pendingQueue = 
        new ConcurrentLinkedQueue<>();

    public UserDTO getUser(Long id) {
        // 先查缓存
        UserDTO cached = userCache.getIfPresent(id);
        if (cached != null) {
            return cached;
        }

        // 加入批量队列
        CompletableFuture<UserDTO> future = new CompletableFuture<>();
        pendingQueue.add(new PendingRequest<>(id, future));

        // 队列达到阈值时立即发送
        if (pendingQueue.size() >= maxBatchSize) {
            flushBatch();
        }

        try {
            return future.get(500, TimeUnit.MILLISECONDS);
        } catch (Exception e) {
            // 超时时降级为单查
            UserDTO user = userClient.getUser(id);
            if (user != null) userCache.put(id, user);
            return user;
        }
    }

    private void flushBatch() {
        List<PendingRequest<UserDTO>> batch = new ArrayList<>();
        PendingRequest<UserDTO> req;
        while ((req = pendingQueue.poll()) != null && batch.size() < maxBatchSize) {
            batch.add(req);
        }

        if (batch.isEmpty()) return;

        List<Long> ids = batch.stream()
            .map(PendingRequest::getId)
            .distinct()
            .collect(Collectors.toList());

        try {
            // 批量调用
            List<UserDTO> users = userClient.getUsers(ids);
            Map<Long, UserDTO> userMap = users.stream()
                .collect(Collectors.toMap(UserDTO::getId, u -> u));

            // 分发结果
            for (PendingRequest<UserDTO> reqItem : batch) {
                UserDTO user = userMap.get(reqItem.getId());
                if (user != null) {
                    userCache.put(reqItem.getId(), user);
                    reqItem.getFuture().complete(user);
                } else {
                    reqItem.getFuture().completeExceptionally(
                        new RuntimeException("User not found: " + reqItem.getId()));
                }
            }
        } catch (Exception e) {
            // 批量调用失败，降级为逐个调用
            for (PendingRequest<UserDTO> reqItem : batch) {
                try {
                    UserDTO user = userClient.getUser(reqItem.getId());
                    userCache.put(reqItem.getId(), user);
                    reqItem.getFuture().complete(user);
                } catch (Exception ex) {
                    reqItem.getFuture().completeExceptionally(ex);
                }
            }
        }
    }

    @Data
    @AllArgsConstructor
    private static class PendingRequest<T> {
        private Long id;
        private CompletableFuture<T> future;
    }
}
```

### 3.4 虚拟线程 + Feign（Java 21+）

```java
// Spring Boot 3.x + Java 21 虚拟线程
spring:
  threads:
    virtual:
      enabled: true  # 启用虚拟线程

// Feign 调用自动运行在虚拟线程上
// 无需显式 async 配置，每个请求自动使用虚拟线程
@FeignClient(name = "user-service")
public interface UserClient {
    
    @GetMapping("/users/{id}")
    UserDTO getUser(@PathVariable Long id);

    @GetMapping("/users/batch")
    List<UserDTO> getUsers(@RequestParam("ids") List<Long> ids);
}

// 虚拟线程下并行调用多个 Feign
@Service
public class VirtualThreadAggregationService {

    public OrderDetailVO getOrderDetail(Long orderId) {
        // 虚拟线程下直接用 sync API，每个调用不阻塞系统线程
        OrderDTO order = orderClient.getOrder(orderId);
        
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            Supplier<UserDTO> user = scope.fork(() -> userClient.getUser(order.getUserId()));
            Supplier<List<OrderItemVO>> items = scope.fork(() -> itemClient.getItems(orderId));
            Supplier<PaymentDTO> payment = scope.fork(() -> paymentClient.getPayment(order.getPaymentId()));

            scope.join();
            scope.throwIfFailed();

            return new OrderDetailVO(order, user.get(), items.get(), payment.get());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
    }
}
```

## 4. 生产实践与注意事项

### 4.1 缓存一致性保证

```java
// 更新数据时清除缓存 + 主动刷新
@Service
public class UserServiceWithCache {

    @CacheEvict(value = "users", key = "#id")
    public UserDTO updateUser(Long id, UserUpdateReq req) {
        UserDTO updated = userAdminClient.updateUser(id, req);
        return updated;
    }

    // 写后立即读——使用 CachePut 更新缓存而非清除
    @Caching(
        put = { @CachePut(value = "users", key = "#result.id") },
        evict = { @CacheEvict(value = "userProfiles", key = "#id") }
    )
    public UserDTO updateUserWithRefresh(Long id, UserUpdateReq req) {
        return userAdminClient.updateUser(id, req);
    }
}
```

### 4.2 缓存穿透防护

```java
// 空值缓存（防止缓存穿透）
@Cacheable(value = "users", key = "#id", unless = "#result == null")
public UserDTO getUser(Long id) {
    UserDTO user = userClient.getUser(id);
    if (user == null) {
        // 缓存空对象，设置短 TTL
        return new UserDTO();  // 空对象
    }
    return user;
}

// 布隆过滤器（拒绝不存在的 ID）
@Component
public class BloomFilterGuard {
    
    private final BloomFilter<Long> bloomFilter;

    public BloomFilterGuard() {
        this.bloomFilter = BloomFilter.create(Funnels.longFunnel(), 100000, 0.01);
    }

    public boolean mightContain(Long id) {
        return bloomFilter.mightContain(id);
    }

    public void put(Long id) {
        bloomFilter.put(id);
    }
}
```

### 4.3 超时与熔断

```yaml
# Feign 超时配置
feign:
  client:
    config:
      default:
        connect-timeout: 2000
        read-timeout: 3000
      user-service:
        connect-timeout: 1000
        read-timeout: 2000

# Sentinel 熔断保护
spring:
  cloud:
    sentinel:
      enabled: true
      datasource:
        degrade:
          nacos:
            server-addr: ${nacos.addr}
            data-id: sentinel-degrade-rules
            rule-type: degrade
```

### 4.4 缓存最佳实践对照

| 模式 | 适用场景 | TTL 建议 | 注意事项 |
|------|----------|----------|----------|
| 基础数据缓存 | 用户信息、商品详情 | 30s-5min | 配合 CacheEvict 保证最终一致性 |
| 配置数据缓存 | 系统配置、字典 | 10-30min | 被动失效 + 主动刷新 |
| 统计数据缓存 | 聚合报表、计数 | 1-5min | 容忍暂时不一致 |
| 空值缓存 | 防穿透 | 5-10s | TTL 不宜过长 |
| 批量请求合并 | 高 QPS 读多写少 | 窗口 5-20ms | 注意超时熔断 |

### 4.5 坑点与避坑

```java
// ❌ 错误：Feign 接口上 @Cacheable 与 @PathVariable 混用导致缓存键异常
@FeignClient(name = "user-service")
public interface UserClient {
    @GetMapping("/users/{id}")
    @Cacheable("users")  // 默认 key 是 SimpleKey，不包含 {id}
    UserDTO getUser(@PathVariable("id") Long id);
}
// ✅ 正确：显式指定 key
@Cacheable(value = "users", key = "#id")

// ❌ 错误：异步 Feign + 事务混用
@Transactional
public void createOrder(CreateOrderReq req) {
    CompletableFuture<InventoryDTO> stock = 
        inventoryClient.checkStock(req.getSkuId());  // 异步可能在其他线程执行
    // Transactional 只绑定调用线程
}
// ✅ 正确：同步调用异步方法
@Transactional
public void createOrder(CreateOrderReq req) {
    InventoryDTO stock = inventoryClient.checkStock(req.getSkuId()).get();

// ❌ 注意：批量合并请求的线程安全
// PendingRequest 队列中的 CompletableFuture 必须在同一线程完成
// 或确保 complete() 线程安全
}
```
