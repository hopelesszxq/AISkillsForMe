---
name: gradle-95-task-provenance
description: Gradle 9.5 新特性：Task Provenance（任务来源追踪）与 Kotlin Settings 插件类型安全访问器
tags: [tools, gradle, build, build-tool, kotlin]
---

## 概述

Gradle 9.5.1（2026-05-12 发布）引入了两个重要特性：**Task Provenance**（任务来源追踪）和 **Type-safe Accessors for Precompiled Kotlin Settings Plugins**。这些特性显著增强了构建的可调试性和类型安全性。

## 一、Task Provenance（任务来源追踪）

### 1.1 什么是 Task Provenance

Task Provenance 记录了每个任务的**来源信息**：由哪个插件、在哪个构建文件中、通过哪个 DSL 方法创建的。当构建失败或行为异常时，可以快速定位任务来源。

### 1.2 在构建报告中查看

```bash
# 生成 HTML 构建报告（包含 Task Provenance）
./gradlew build --build-report

# 查看任务来源（命令行输出）
./gradlew build --info | grep "Task provenance"
```

生成的 HTML 报告中，每个任务会显示：

```
Task: compileJava
  Provenance:
    Plugin: java (org.gradle.api.plugins.JavaPlugin)
    File: build.gradle.kts:15
    Method: tasks.register("compileJava")
    Owner: project ':app'
```

### 1.3 在失败信息中追踪

```bash
# 构建失败时自动显示任务来源
./gradlew build
```

如果某个任务失败，错误信息会包含该任务的来源信息，便于快速定位是哪个插件或哪个脚本引入了该任务。

```
* What went wrong:
Execution failed for task ':app:compileJava'.
> Compilation failed

* Task provenance:
  Plugin: org.gradle.api.plugins.JavaBasePlugin
  Declared in: build.gradle.kts at line 23
```

### 1.4 自定义插件中标记任务来源

```kotlin
// 自定义插件中可以显式标记任务来源
abstract class MyPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        val myTask = project.tasks.register("myCustomTask") {
            description = "Custom task with provenance"
            // Gradle 9.5 自动追踪来源，无需额外配置
        }
    }
}
```

## 二、预编译 Kotlin Settings 插件的类型安全访问器

### 2.1 背景

在 Gradle 9.5 之前，`settings.gradle.kts` 中使用的预编译 Kotlin Settings 插件（实现了 `SettingsPlugin` 接口）没有类型安全访问器，开发者需要手动通过字符串键名访问扩展。

### 2.2 使用示例

**定义 Settings 插件**（在 `buildSrc` 或复合构建中）：

```kotlin
// buildSrc/src/main/kotlin/my-settings-plugin.gradle.kts
abstract class MySettingsPlugin : SettingsPlugin<MySettingsExtension> {
    override fun apply(settings: Settings) {
        settings.extensions.create("mySettings", MySettingsExtension::class.java)
    }
}

abstract class MySettingsExtension {
    abstract val projectGroup: Property<String>
    abstract val defaultVersion: Property<String>
    abstract val includeModules: ListProperty<String>
}
```

**使用类型安全访问器**：

```kotlin
// settings.gradle.kts
plugins {
    id("my-settings-plugin")
}

// Gradle 9.5+ 类型安全访问器
mySettings {
    projectGroup.set("com.example")
    defaultVersion.set("1.0.0")
    includeModules.set(listOf(":shared", ":app"))
}
```

### 2.3 对比

```kotlin
// Gradle 9.4 及之前（字符串方式）
configure<MySettingsExtension> {
    projectGroup.set("com.example")
}

// 或
extensions.configure("mySettings") {
    // 需要自行转换类型
}

// Gradle 9.5+（类型安全，无需手动 configure）
mySettings {
    projectGroup.set("com.example")
}
```

## 三、其他值得注意的变更

### 3.1 升级命令

```bash
# 升级 Wrapper 到 9.5.1
./gradlew wrapper --gradle-version=9.5.1
```

### 3.2 兼容性

- Java: JDK 17+（推荐 JDK 21+）
- Kotlin: 2.1.x
- Android: AGP 8.7+

## 注意事项

- Task Provenance 在所有 Gradle 任务中默认启用，无需额外配置
- 构建报告（`--build-report`）功能在 9.5 中有格式更新，CI 中若解析报告需同步更新解析逻辑
- Kotlin Settings 插件的类型安全访问器仅在 `settings.gradle.kts` 中生效，不适用于 Groovy DSL
- 如果使用 composite build 且 settings 插件定义在外部项目中，需确保 Gradle 版本一致
- 从 Gradle 8.x 升级到 9.x 的用户，请先查阅 [Gradle 9.x Upgrade Guide](https://docs.gradle.org/9.5.1/userguide/upgrading_version_9.html)
