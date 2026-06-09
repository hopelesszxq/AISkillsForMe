---
name: java-structured-concurrency-scoped-values
description: Java 结构化并发与作用域值模式：StructuredTaskScope 替代 CompletableFuture、ScopedValue 替代 ThreadLocal 的最佳实践
tags: [patterns, java, structured-concurrency, scoped-values, virtual-threads, concurrency]
---

## 概述

Java 21+（通过预览阶段，Java 25 已趋于稳定）引入了两大并发编程新模式：**结构化并发（Structured Concurrency, JEP 428）** 和**作用域值（Scoped Values, JEP 429）**。它们与虚拟线程（Virtual Threads）一起构成了 Java 新一代并发编程的"三驾马车"。

## 一、结构化并发（StructuredTaskScope）

结构化并发将并发任务的生命周期与代码块绑定，确保任务要么**全部成功**并收集结果，要么**统一取消**并处理异常。

### 1.1 基本用法 vs 传统模式

```java
// ❌ 传统模式：CompletableFuture 容易导致线程泄漏
List<CompletableFuture<Result>> futures = services.stream()
    .map(s -> CompletableFuture.supplyAsync(() -> s.fetch()))
    .toList();
// 如果某个 future 超时，其他 future 仍然在后台运行
// 一旦主线程退出，这些"遗落"的任务无法被管理

// ✅ 结构化并发模式
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    List<Subtask<Result>> subtasks = services.stream()
        .map(s -> scope.fork(() -> s.fetch()))
        .toList();
    
    scope.join();                // 等待所有子任务
    scope.throwIfFailed();       // 任一失败则抛出异常并取消其余任务
    
    List<Result> results = subtasks.stream()
        .map(Subtask::get)
        .toList();
}
// scope.close() 自动取消未完成的任务
```

### 1.2 三种内置关闭策略

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| `ShutdownOnFailure` | 任一子任务失败 → 取消其余 | 并行调用依赖链 |
| `ShutdownOnSuccess` | 任一子任务成功 → 取消其余 | 多数据源冗余调用（先到先得） |
| 自定义 | 扩展 `StructuredTaskScope` 自定义策略 | 特殊业务逻辑 |

### 1.3 经典模式：冗余调用（先到先得）

```java
// 同时查询多个缓存/数据库，取最先返回的结果
try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {
    scope.fork(() -> redisCache.get(key));
    scope.fork(() -> localCache.get(key));
    scope.fork(() -> database.query(key));
    
    String result = scope.join().result();  // 最先成功的结果
    // 其余任务自动被取消
    
    return result;
}
```

### 1.4 自定义策略

```java
// 自定义策略：收集成功结果，等所有任务结束或超时
class CollectSuccessScope extends StructuredTaskScope<String> {
    private final List<String> results = new CopyOnWriteArrayList<>();
    private final List<Throwable> errors = new CopyOnWriteArrayList<>();
    
    @Override
    protected void handleComplete(Subtask<? extends String> subtask) {
        switch (subtask.state()) {
            case SUCCESS -> results.add(subtask.get());
            case FAILED  -> errors.add(subtask.exception());
            case UNAVAILABLE -> {} // task cancelled
        }
    }
    
    public List<String> results() { return results; }
    public List<Throwable> errors() { return errors; }
}

// 使用
try (var scope = new CollectSuccessScope()) {
    scope.fork(() -> callService("A"));
    scope.fork(() -> callService("B"));
    scope.fork(() -> callService("C"));
    scope.join();  // 等待所有任务完成（或取消）
    
    return scope.results();  // 返回所有成功结果
}
```

## 二、作用域值（ScopedValue）

`ScopedValue` 替代 `ThreadLocal`，解决 ThreadLocal 的三大问题：
1. **不可变性**：值一旦绑定不可变（除 rebind）
2. **父子继承**：虚拟线程中自动继承到子任务
3. **资源泄漏**：作用域结束时自动清理，无需 remove()

### 2.1 基本用法

```java
// 定义作用域值
private static final ScopedValue<String> REQUEST_ID = ScopedValue.newInstance();
private static final ScopedValue<String> USER_ID = ScopedValue.newInstance();

// 绑定值 — 仅在 lambda 范围内有效
ScopedValue.where(REQUEST_ID, "req-123")
           .where(USER_ID, "user-456")
           .run(() -> {
               // 在此作用域内可以读取
               handleRequest();
           });
// 离开作用域后自动清理

void handleRequest() {
    // 读取作用域值
    String requestId = REQUEST_ID.get();
    String userId = USER_ID.get();
    
    // 在虚拟线程中 fork 子任务 — 自动继承
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        scope.fork(() -> processA());  // 自动继承 REQUEST_ID, USER_ID
        scope.fork(() -> processB());
        scope.join();
    }
}
```

### 2.2 替代 ThreadLocal 的迁移

```java
// ❌ 旧模式：ThreadLocal
private static final ThreadLocal<String> currentUser = new ThreadLocal<>();

// 在 filter 中设置
currentUser.set(userId);
try {
    // 处理请求
} finally {
    currentUser.remove();  // 容易忘记
}

// ✅ 新模式：ScopedValue
private static final ScopedValue<String> CURRENT_USER = ScopedValue.newInstance();

// 在 filter 中绑定
ScopedValue.where(CURRENT_USER, userId).run(() -> {
    processRequest();  // 无需手动清理
});
```

### 2.3 与虚拟线程结合

```java
// ScopedValue 在虚拟线程中的自动传递
private static final ScopedValue<TraceContext> TRACE = ScopedValue.newInstance();

ScopedValue.where(TRACE, TraceContext.start()).run(() -> {
    // 主处理
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        // 子任务自动继承 TRACE
        scope.fork(() -> {
            // TRACE.get() 可以正常访问
            TraceContext ctx = TRACE.get();
            return serviceCall(ctx);
        });
        scope.fork(() -> anotherService());
        scope.join();
    }
});
```

## 三、结构化并发 + 超时控制

```java
// 组合超时与结构化并发
Duration timeout = Duration.ofSeconds(5);

try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    scope.fork(() -> longRunningTask());
    
    try {
        scope.join(timeout);  // 等待指定时长
    } catch (TimeoutException e) {
        // scope.close() 自动取消未完成的任务
        throw new ServiceException("Request timed out after 5s");
    }
    
    scope.throwIfFailed();
}
```

## 四、注意事项

### 性能与兼容性

| 对比项 | ThreadLocal | ScopedValue | 差异 |
|--------|-------------|-------------|------|
| 可变性 | 可读可写 | 只读（需 rebind） | ScopedValue 更安全 |
| 继承性 | 需手动 InheritableThreadLocal | 自动继承 | 更简洁 |
| 清理 | finally{remove()} | 自动 | 防泄漏 |
| 性能 | 较快 | 略慢（但比 ITL 快） | 可接受 |

### 坑点

1. **ScopedValue 不可变**：`where()` 返回新绑定，原作用域不受影响
2. **rebind 谨慎使用**：`ScopedValue.where(...).run()` 嵌套时，内层覆盖外层
3. **结构化并发必须 try-with-resources**：忘记 close() 会导致任务泄露（编译器会警告）
4. **无法取消阻塞 I/O**：`scope.join()` 超时后，正在阻塞的 I/O 操作不会自动中断
5. **Spring 兼容性**：Spring Boot 4.x+ 对 ScopedValue 有原生支持；老版本可能仍使用 ThreadLocal

### Java 版本要求

```xml
<!-- JEP 428 & 429 支持版本 -->
<!-- Java 21: 预览（需 --enable-preview） -->
<!-- Java 23: 第二预览 -->
<!-- Java 25: 趋于稳定 -->
```

```bash
# 编译和运行需要
javac --release 21 --enable-preview App.java
java --enable-preview App
```
