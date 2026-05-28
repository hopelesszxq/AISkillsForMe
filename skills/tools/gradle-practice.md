---
name: gradle-practice
description: Gradle 构建工具最佳实践：Kotlin DSL、多模块、依赖管理、构建优化
tags: [tools, gradle, build, java, kotlin]
---

## 项目结构

### 推荐目录布局

```
my-project/
├── settings.gradle.kts           # 项目设置（模块声明）
├── build.gradle.kts              # 根项目构建脚本
├── gradle.properties             # JVM 参数和项目属性
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew                       # Gradle Wrapper（Unix）
├── gradlew.bat                   # Gradle Wrapper（Windows）
├── sub-project-a/
│   ├── build.gradle.kts
│   └── src/
├── sub-project-b/
│   ├── build.gradle.kts
│   └── src/
└── buildSrc/                     # 构建脚本公共逻辑
    ├── build.gradle.kts
    └── src/main/kotlin/
        └── convention/
            ├── java-convention.gradle.kts
            └── spring-convention.gradle.kts
```

## Kotlin DSL 最佳实践

### 1. 根项目配置

```kotlin
// settings.gradle.kts
rootProject.name = "my-project"

pluginManagement {
    repositories {
        maven { url = uri("https://maven.aliyun.com/repository/public") }
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositories {
        maven { url = uri("https://maven.aliyun.com/repository/public") }
        mavenCentral()
    }
}

include("sub-project-a", "sub-project-b")
```

### 2. 多模块依赖管理

```kotlin
// build.gradle.kts（根项目）
plugins {
    id("java") apply false
    id("org.springframework.boot") version "3.2.5" apply false
    id("io.spring.dependency-management") version "1.1.4" apply false
    kotlin("jvm") version "1.9.24" apply false
    kotlin("plugin.spring") version "1.9.24" apply false
}

// 子模块通用配置
subprojects {
    apply(plugin = "java")
    apply(plugin = "io.spring.dependency-management")

    group = "com.example"
    version = "1.0.0-SNAPSHOT"

    java {
        toolchain {
            languageVersion = JavaLanguageVersion.of(21)
        }
    }

    tasks.withType<Test> {
        useJUnitPlatform()
    }

    tasks.withType<JavaCompile> {
        options.encoding = "UTF-8"
        options.compilerArgs.add("-parameters")
    }

    // 统一依赖版本管理（替代 Spring Boot BOM）
    the<io.spring.gradle.dependencymanagement.DependencyManagementExtension>().apply {
        imports {
            mavenBom("org.springframework.boot:spring-boot-dependencies:3.2.5")
            mavenBom("org.springframework.cloud:spring-cloud-dependencies:2023.0.1")
        }
    }
}
```

### 3. 子模块构建配置

```kotlin
// sub-project-a/build.gradle.kts
plugins {
    id("java")
    id("org.springframework.boot")
    id("io.spring.dependency-management")
}

dependencies {
    // 项目内依赖
    implementation(project(":sub-project-b"))

    // Spring Boot
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")

    // 数据库
    implementation("com.baomidou:mybatis-plus-boot-starter:3.5.7")
    runtimeOnly("org.postgresql:postgresql")

    // 工具
    implementation("cn.hutool:hutool-all:5.8.28")
    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")

    // 测试
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}
```

## buildSrc 公共约定（Convention Plugin）

### 创建约定插件

```kotlin
// buildSrc/build.gradle.kts
plugins {
    `kotlin-dsl`
}

repositories {
    mavenCentral()
    gradlePluginPortal()
}

// buildSrc/src/main/kotlin/convention/java-convention.gradle.kts
plugins {
    id("java")
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

tasks.withType<Test> {
    useJUnitPlatform()
}

// buildSrc/src/main/kotlin/convention/spring-convention.gradle.kts
plugins {
    id("java")
    id("org.springframework.boot")
    id("io.spring.dependency-management")
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

### 使用约定插件

```kotlin
// sub-project/build.gradle.kts
plugins {
    id("convention.spring-convention")
    id("convention.java-convention")
}
```

## 构建性能优化

### 1. 构建缓存

```kotlin
// gradle.properties
org.gradle.caching=true
org.gradle.caching.debug=false

// 使用 HTTP 构建缓存（CI 共享）
// settings.gradle.kts
buildCache {
    remote<HttpBuildCache> {
        url = uri("https://build-cache.example.com/cache/")
        isPush = System.getenv("CI") != null
        credentials {
            username = System.getenv("CACHE_USERNAME")
            password = System.getenv("CACHE_PASSWORD")
        }
    }
}
```

### 2. 并行执行与守护进程

```properties
# gradle.properties
org.gradle.parallel=true
org.gradle.workers.max=4
org.gradle.daemon=true
org.gradle.jvmargs=-Xmx2048m -XX:MaxMetaspaceSize=512m -XX:+HeapDumpOnOutOfMemoryError
```

### 3. 配置缓存

```kotlin
// settings.gradle.kts
enableFeaturePreview("STABLE_CONFIGURATION_CACHE")

// 避免使用 project 相关配置，否则配置缓存失效
// ✅ 正确：使用 providers 延迟计算
val myProp = providers.gradleProperty("myProp")
// ❌ 错误：project 访问会在配置时触发
// val myProp = project.properties["myProp"]
```

### 4. 增量编译

```kotlin
// build.gradle.kts
tasks.withType<JavaCompile> {
    options.isIncremental = true
    options.isFork = true
    options.forkOptions.memoryMaximumSize = "1g"
}
```

## 依赖管理技巧

### 版本目录（Version Catalog）

```toml
# gradle/libs.versions.toml
[versions]
spring-boot = "3.2.5"
spring-cloud = "2023.0.1"
mybatis-plus = "3.5.7"
hutool = "5.8.28"

[libraries]
spring-boot-starter-web = { module = "org.springframework.boot:spring-boot-starter-web", version.ref = "spring-boot" }
spring-boot-starter-data-jpa = { module = "org.springframework.boot:spring-boot-starter-data-jpa", version.ref = "spring-boot" }
mybatis-plus = { module = "com.baomidou:mybatis-plus-boot-starter", version.ref = "mybatis-plus" }
hutool-all = { module = "cn.hutool:hutool-all", version.ref = "hutool" }

[bundles]
spring-web = ["spring-boot-starter-web", "spring-boot-starter-data-jpa"]

[plugins]
spring-boot = { id = "org.springframework.boot", version.ref = "spring-boot" }
```

```kotlin
// build.gradle.kts
dependencies {
    implementation(libs.bundles.spring.web)
    implementation(libs.mybatis.plus)
    implementation(libs.hutool.all)
}
```

## 常用任务配置

### 1. Fat JAR 排除

```kotlin
tasks.bootJar {
    archiveFileName.set("app.jar")
    // 排除特定依赖
    excludes.add("**/tomcat-embed-*")
}

// 生成可执行 JAR + 普通 JAR
tasks.jar {
    enabled = true
    archiveClassifier.set("plain")
}
```

### 2. 自动化版本管理

```kotlin
// 使用 Git 提交数作为版本号
val gitCount = providers.exec {
    commandLine("git", "rev-list", "--count", "HEAD")
}.standardOutput.asText.map { it.trim() }

version = "1.0.${gitCount.get()}"
```

### 3. 多环境构建

```kotlin
// build.gradle.kts
val env = (project.findProperty("env") as? String) ?: "dev"

sourceSets {
    main {
        resources {
            srcDir("src/main/resources")
            srcDir("src/main/resources-${env}")
        }
    }
}

tasks.processResources {
    duplicatesStrategy = DuplicatesStrategy.EXCLUDE
}
```

```bash
# 构建不同环境
./gradlew clean build -Penv=dev
./gradlew clean build -Penv=prod
```

## Maven → Gradle 迁移对照

| Maven | Gradle |
|-------|--------|
| `pom.xml` | `build.gradle.kts` |
| `dependencyManagement` | `dependencyResolutionManagement` |
| `mvn clean install` | `./gradlew clean build` |
| `mvn test` | `./gradlew test` |
| `mvn dependency:tree` | `./gradlew dependencies` |
| `-DskipTests` | `-x test` |
| 多模块 `<modules>` | `include(":module")` |
| `<parent>` | 根项目 + subprojects {} |
| `mvn deploy` | `./gradlew publish` |
| `mvn -T 4` | `org.gradle.parallel=true`（默认） |

## 常见坑点

1. **依赖缓存不更新**：删除 `~/.gradle/caches/` 或运行 `./gradlew clean --refresh-dependencies`
2. **Kotlin DSL 编译慢**：第一次运行慢，后续守护进程加速。升级 Gradle 8.x+ 改善很大
3. **Lombok 与 Gradle**：使用 `compileOnly` + `annotationProcessor`，不要用 `implementation`
4. **Windows 路径过长**：Gradle 8.x 已修复，升级即可
5. **配置缓存失效**：避免在 `build.gradle.kts` 顶层调用 `file()`、`project` 等方法，使用 `providers`
6. **Spring Boot DevTools**：默认重启策略可能和 Gradle 增量编译冲突，要配置 `trigger-file`
7. **不要提交 build 目录**：`.gitignore` 中添加 `build/`、`.gradle/`、`local.properties`
8. **Wrapper 版本管理**：`./gradlew wrapper --gradle-version=8.7` 升级 Wrapper
