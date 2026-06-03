---
name: spring-mvc-performance-tuning
description: Spring MVC 性能优化实战 —— 虚拟线程、@Transactional 优化、Spring Framework 7 内建优化
tags: [tools, spring-mvc, spring-boot, performance, optimization, virtual-threads]
---

## 概述

Spring 官方博客发表了 Spring MVC 性能基准分析（*"Optimizations in Spring MVC"*, Dave Syer, 2026-02-25），通过实际测试数据揭示了几项关键优化措施的效果。本文提炼其中可落地的性能优化策略。

## 性能基线分析

基准测试场景：获取并渲染完整数据集（水果-商店-价格 JSON 接口）。

测试指标：
- 吞吐量（TPS）：每秒请求数
- 周期时间（Cycle Time）：`1 / throughput = A + B*N`（A=框架固定开销，B=每项数据处理时间，N=数据项数）

关键发现：对于小数据集，框架固定开销占比大；对于大数据集，业务处理时间占主导。

## 优化策略

### 1. 为只读事务使用 SUPPORTS + readOnly

**问题**：事务管理器对每次 HTTP 请求都创建事务，即使操作是只读的，也存在显著开销。

**优化前**（默认）：
```java
@GetMapping("/fruits")
public List<Fruit> getFruits() {
    return fruitService.findAll();  // 隐式创建只读事务，仍有事务管理开销
}
```

**优化后**：
```java
@GetMapping("/fruits")
@Transactional(propagation = Propagation.SUPPORTS, readOnly = true)
public List<Fruit> getFruits() {
    return fruitService.findAll();  // 仅在已有事务中运行，无事务则直接执行
}
```

**效果**：对小数据集的场景吞吐量提升显著（高达 50-100%），因为消除了事务管理的固定开销。

### 2. 启用虚拟线程（Virtual Threads）

Spring Boot 应用启用虚拟线程极其简单：

```properties
# application.properties
spring.threads.virtual.enabled=true
```

```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true
```

**效果**：虚拟线程显著提升并发吞吐量，特别是在 I/O 密集型场景下。

### 3. 升级到 Spring Framework 7.0.6+

Spring Framework 7.0.6 引入了内建 HTTP 协议优化：
- 更高效的 HTTP 头部处理
- 适配底层容器的优化实现
- 减少不必要的 HTTP 协议开销

**版本要求**：Spring Boot 4.x + Spring Framework 7.x

## 综合性能数据

| N（数据项数） | 基准 TPS | + @Transactional 优化 | + 虚拟线程 | + Spring Framework 7.0.6 |
|---|---|---|---|---|
| 10 | 22000 | 34000 (+55%) | 38000 (+73%) | 41000 (+86%) |
| 100 | 6500 | 7500 (+15%) | 8800 (+35%) | 9200 (+42%) |
| 1000 | 800 | 830 (+4%) | 950 (+19%) | 990 (+24%) |

> 注：数据基于 Spring 官方基准测试，实际提升幅度因应用特性而异。数据集越大，业务处理时间占比越高，框架优化的边际收益递减。

## 最佳实践组合

```java
@RestController
@RequestMapping("/api")
public class OptimizedController {

    @GetMapping("/products")
    @Transactional(propagation = Propagation.SUPPORTS, readOnly = true)
    public List<Product> getProducts() {
        return productService.findAll();
    }
    
    @PostMapping("/products")
    @Transactional(rollbackFor = Exception.class)
    public Product createProduct(@RequestBody Product product) {
        return productService.save(product);
    }
}
```

```yaml
# application.yml - 完整优化配置
spring:
  threads:
    virtual:
      enabled: true
  jpa:
    open-in-view: false  # 避免 OSIV 带来的连接泄漏
```

## 注意事项

1. **@Transactional SUPPORTS 的适用场景**：仅对**纯只读查询**端点使用；写入操作仍需 `REQUIRED`（默认传播行为）
2. **虚拟线程池**：虚拟线程适用于 I/O 密集型场景，对 CPU 密集型计算任务提升有限
3. **OSIV（Open Session In View）**：Spring Boot 默认开启，建议在性能敏感场景下关闭（`spring.jpa.open-in-view: false`）
4. **测试验证**：基准测试数据仅供参考，实际效果需结合业务代码进行压测验证
5. **渐进式升级**：优先应用 @Transactional 优化（零依赖升级），再考虑虚拟线程和框架升级
