---
name: java-25-features
description: Java 25 核心特性：Scoped Values 转正 (JEP 506)、JFR CPU-Time Profiling (JEP 509)
tags: [tools, java, jdk25, profiling, scoped-values, performance]
---

## 概述

Java 25（2025-09 正式发布）标志着 Java 并发和性能工具链的进一步成熟。两大关键变化：

- **JEP 506: Scoped Values 转正** — 作用域值 API 在三次预览后正式定稿，替代 ThreadLocal 的新方案
- **JEP 509: JFR CPU-Time Profiling（实验性）** — 利用 Linux CPU 定时器实现更精确的 CPU 时间采样

## 一、Scoped Values 转正 (JEP 506)

### 核心变化

Scoped Values 从 JDK 21 的孵化（JEP 429）→ JDK 22/23/24 三次预览 → **JDK 25 正式转正**。

唯一 API 变更：移除了 `ScopedValue.callWhere` 方法，统一使用 `where(...).run(...)` / `where(...).get(...)` 模式。

### 为什么替代 ThreadLocal？

| 维度 | ThreadLocal | ScopedValue |
|------|------------|-------------|
| 可变性 | 任意修改 | 不可变（无 `set` 方法） |
| 子线程继承 | 复制（内存开销） | 自动继承（零拷贝） |
| 虚拟线程 | 频繁创建销毁时泄漏 | 设计匹配（轻量） |
| 可重入 | 需手工处理 | 内置支持 `rebind` |

### 典型用法

```java
// 声明
private static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

// 绑定（在请求入口处）
ScopedValue.where(CURRENT_USER, authenticatedUser)
    .run(() -> handleRequest(request));

// 读取（任意深处）
User user = CURRENT_USER.get(); // 自动获取
```

### 子线程继承

```java
// 自动继承——子线程可直接读取父线程的 ScopedValue
ScopedValue.where(REQUEST_ID, "req-123")
    .run(() -> {
        // 虚拟线程自动继承
        try (var scope = new StructuredTaskScope<>()) {
            scope.fork(() -> processSubTask()); // 可读取 REQUEST_ID
            scope.join();
        }
    });
```

### 多值绑定

```java
// 同时绑定多个 ScopedValue
ScopedValue.where(USER, u)
    .where(TENANT, t)
    .where(REQUEST_ID, id)
    .run(() -> process());
```

## 二、JFR CPU-Time Profiling (JEP 509)

### 问题背景

传统 JFR 执行时间采样（`jdk.ExecutionSample`）存在三大缺陷：

1. **仅采样 Java 代码**，不采样 native 代码执行
2. **子采样偏差** — 大机器上有效采样率仅 19%（32核时 20ms → 53ms）
3. **不报告失败样本** — 最多 1/3 的采样被静默丢弃

### CPU-Time Profiling 原理

利用 Linux 内核 2.6.12+ 的 **CPU 定时器**，以固定 CPU 时间间隔（而非墙上时间间隔）采样每个线程：

```
墙上时钟采样（旧）:  每 20ms 采 5 个线程 → 偏差大
CPU 时间采样（新）:  每线程每 n ms CPU 时间采一次 → 准确
```

### 启用方式

```bash
# 启动时启用 CPU-Time Profiling
java -XX:StartFlightRecording=jdk.CPUTimeSample#enabled=true,filename=profile.jfr \
     -jar myapp.jar

# 使用 jfr 查看
jfr view methods profile.jfr
```

### 新事件对比

| 事件 | 旧 | 新 |
|------|-----|-----|
| 事件名称 | `jdk.ExecutionSample` | `jdk.CPUTimeSample` |
| 采样基准 | 墙上时间 | CPU 时间 |
| 覆盖范围 | Java 代码 | Java + native 代码 |
| 子采样 | 有（最多 5 线程/次） | 无（所有 CPU 活跃线程） |
| 失败报告 | 静默忽略 | 报告失败计数 |

### 实际案例

```
┌──────────────────────────────────────────────┐
│ 场景：HTTP 服务，两个线程做请求              │
│ Thread A: makeTenRequests()  → 10次×10ms I/O │
│ Thread B: makeOneRequest()   → 1次×100ms I/O │
├──────────────────────────────────────────────┤
│ 执行时间采样：两者相近（主机都是 I/O 等待）  │
│ CPU时间采样：makeTenRequests 占用更多 CPU    │
│              （10次请求处理 vs 1次）           │
│ 结论：makeTenRequests 是优化目标              │
└──────────────────────────────────────────────┘
```

## 三、迁移建议

### 从 ThreadLocal → ScopedValue

```java
// ❌ 旧
private static final ThreadLocal<User> tlUser = new ThreadLocal<>();

// 设置
tlUser.set(user);

// 读取
User u = tlUser.get();

// ✅ 新
private static final ScopedValue<User> svUser = ScopedValue.newInstance();

// 绑定（在作用域入口）
ScopedValue.where(svUser, user).run(() -> {
    // 读取
    User u = svUser.get();
});
```

### 性能提示

- **ScopedValue 读取比 ThreadLocal 快** — 无哈希查找，直接指针访问
- **虚拟线程 + ScopedValue** — 黄金组合，避免线程池泄漏
- **CPU-Time Profiling 是实验功能** — 仅 Linux 可用，后续可能扩展其他平台

## 参考

- [JEP 506: Scoped Values](https://openjdk.org/jeps/506)
- [JEP 509: JFR CPU-Time Profiling (Experimental)](https://openjdk.org/jeps/509)
- [Java 25's new CPU-Time Profiler (1)](https://mostlynerdless.de/blog/2025/06/11/java-25s-new-cpu-time-profiler-1/)
