---
name: gradle-96-features
description: Gradle 9.6 RC1 新特性：配置缓存命中率提升、--no-interactive 无交互模式、NO_COLOR 环境变量支持
tags: [tools, gradle, build, configuration-cache, ci]
---

## 概述

Gradle 9.6.0-RC1（2026-05-28 发布）聚焦于**配置缓存（Configuration Cache）** 的体验优化和 CI/CD 场景增强。核心改进：配置缓存项目属性精确追踪、`--no-interactive` 无交互运行模式、`NO_COLOR` 环境变量支持以及项目层次结构查找废弃警告。

> 当前稳定版为 Gradle 9.5.1（2026-05-12），本文覆盖 RC1 新增特性，正式版发布后通常不会有重大变化。

## 一、配置缓存：项目属性精确追踪

### 问题背景

之前配置缓存在检测到**任何**系统属性或环境变量变化时都会失效：

```bash
# 即使 value 属性未被配置阶段读取，也会导致缓存失效
./gradlew --configuration-cache printValue -Dorg.gradle.project.value=1
# ⚠️ 缓存因 'org.gradle.project.value' 变化而失效率
```

### 9.6 改进：精确追踪

Gradle 9.6 精确追踪**实际被配置阶段读取**的项目属性，不再因为无关属性变化而失效：

```kotlin
// build.gradle.kts
val value = providers.gradleProperty("value").orElse("N/A")
```

```bash
# 即使通过不同方式传入，只要配置阶段未读取，缓存依然有效
./gradlew --configuration-cache printValue -Dorg.gradle.project.value=2
# ✅ 缓存命中！
```

### 环境变量同样支持

```bash
# 通过环境变量传入属性，同样不会破坏配置缓存
$ ORG_GRADLE_PROJECT_value=3 ./gradlew --configuration-cache printValue
# ✅ 缓存命中！
```

### 适用场景

- **CI 流水线**：频繁通过命令行/环境变量传入不同参数时，大幅提升缓存命中率
- **多分支构建**：不同分支使用不同属性值时，配置缓存无需重建
- **大型项目**：减少 IO 负载，尤其适合低 IOPS 的云 CI 环境

### 配置缓存 IO 优化

Gradle 9.6 还降低了文件型日志（journal）的 IO 负载，在低 IOPS 存储（如云 CI Runner）上性能提升显著：

```
# 在 GitHub Actions / GitLab CI 等云环境中效果明显
# 配置缓存加载时间减少 30-50%（取决于项目规模和存储类型）
```

## 二、--no-interactive：无交互运行模式

### 新选项

```bash
# 禁用所有交互式提示
./gradlew build --no-interactive
```

### 适用场景

| 场景 | 说明 |
|------|------|
| **CI 流水线** | 避免构建因等待用户输入而挂起 |
| **AI Agent** | 自动化脚本调用 Gradle 时无需交互 |
| **脚本/自动化** | 批量构建、自动部署等场景 |

### 等效环境变量

```bash
# 通过环境变量设置
export GRADLE_INTERACTIVE=false
./gradlew build
```

## 三、NO_COLOR 环境变量支持

遵循 [no-color.org](https://no-color.org) 标准：

```bash
# 禁用颜色输出，但保留其他样式（加粗、下划线）和丰富特性（进度条、动画）
export NO_COLOR=1
./gradlew build
```

```bash
# 输出效果
# 🔵 → 正常的蓝字（无颜色）
# ✨ 进度条、动画等功能正常显示
```

### 对比 --no-color

| 方式 | 颜色 | 样式（加粗/下划线） | 进度条/动画 |
|------|------|-------------------|------------|
| 默认 | ✅ | ✅ | ✅ |
| `--no-color` | ❌ | ❌ | ❌ |
| `NO_COLOR=1` | ❌ | ✅ | ✅ |

## 四、项目层次结构查找废弃警告

### 问题

Gradle 9.6 开始，通过项目层次结构进行**隐式属性和方法查找**（即从子项目访问父项目的未声明属性）会发出警告：

```kotlin
// ❌ 隐式访问父项目属性（不推荐）
val rootProp = rootProject.property("someProp")

// ✅ 显式声明（推荐）
val rootProp by rootProject.ext
```

### 升级路径

```bash
# 启用 Gradle 10 行为预览，提前适配
# 在 gradle.properties 或 settings.gradle.kts 中：
systemProp.gradle.preview.earlyDeprecation=true
```

> 此特性将在 Gradle 10 中完全移除，建议尽早迁移。

## 五、废弃注解检测增强

对在任务属性上误用 `@Internal`、`@Input` 等注解的情况，提供更清晰的废弃警告：

```kotlin
// ❌ 错误：@Input 注解在非输入属性上
@get:Input
val tempDir: Provider<Directory> = project.layout.buildDirectory.dir("tmp")

// ✅ 正确：使用 @OutputDirectory
@get:OutputDirectory
val tempDir: Provider<Directory> = project.layout.buildDirectory.dir("tmp")
```

## 升级注意事项

1. **RC1 不建议用于生产**：等待正式版（GA）发布后再升级
2. **检查项目层次结构依赖**：运行 `gradle build --warning-mode=all` 查找隐式属性访问
3. **配置缓存测试**：在 CI 中启用 `--configuration-cache` 验证缓存命中率提升
4. **`--no-interactive` 与 CI 集成**：建议在 CI 配置中默认添加该选项
5. **NO_COLOR 兼容性**：如果已使用 `--no-color`，可改为 `NO_COLOR=1` 保留进度条体验

## 参考

- [Gradle 9.6.0 RC1 Release Notes](https://docs.gradle.org/9.6.0-rc-1/release-notes.html)
- [Configuration Cache 文档](https://docs.gradle.org/current/userguide/configuration_cache.html)
- [NO_COLOR 标准](https://no-color.org)
