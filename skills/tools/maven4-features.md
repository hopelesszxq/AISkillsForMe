---
name: maven4-features
description: Maven 4 新特性详解：POM-less 构建、Config Relocation、Consumer/Producer POM、.mvn 扩展目录
tags: [tools, maven, build, java, maven4]
---

## 概述

Maven 4（2024 年 7 月发布 GA）是 Maven 3 之后的最大版本升级，核心变化：**POM-less 简化配置**、**Consumer POM / Producer POM 分离**、**Config Relocation 配置迁移**、**原生 .mvn 扩展支持**。

## 1. POM-less 构建（Convention Over Configuration）

Maven 4 允许在**没有 pom.xml** 的情况下构建标准项目。

### 自动推断配置

```xml
<!-- 最简单 pom.xml，只需定义 GAV -->
<project>
    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <version>1.0.0</version>
</project>
```

Maven 4 自动推断：

- 源码目录：`src/main/java/`、`src/test/java/`
- 资源目录：`src/main/resources/`、`src/test/resources/`
- 输出目录：`target/classes/`、`target/test-classes/`
- 打包方式：jar（如果存在 `src/main/java/Main.java` 则自动执行 `maven-jar-plugin`）
- 编译插件：自动绑定 `maven-compiler-plugin`（JDK 21）

### 完全无 pom.xml

```bash
# 在标准项目结构中直接构建
mkdir -p src/main/java/com/example
cat > src/main/java/com/example/App.java << 'EOF'
package com.example;
public class App {
    public static void main(String[] args) {
        System.out.println("Maven 4 POM-less!");
    }
}
EOF

# 直接构建，无需 pom.xml
mvn compile        # 自动推断一切
mvn package        # 自动打包为 my-app-1.0.0.jar
```

## 2. Consumer POM / Producer POM

核心概念：**构建产物（Producer）的 POM 和 依赖消费者（Consumer）看到的 POM 分离**。

### Producer POM（构建时）

```xml
<!-- pom.xml：构建时完整 POM -->
<project>
    <groupId>com.example</groupId>
    <artifactId>my-lib</artifactId>
    <version>1.0.0</version>

    <dependencies>
        <!-- 编译依赖（不会泄漏给消费者） -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <scope>provided</scope>
        </dependency>

        <!-- 测试依赖 -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-source-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### Consumer POM（自动生成）

```xml
<!-- 自动生成到 META-INF/maven/consumer-pom.xml -->
<!-- 只包含消费者需要的信息 -->
<project>
    <groupId>com.example</groupId>
    <artifactId>my-lib</artifactId>
    <version>1.0.0</version>

    <!-- provided/test scope 依赖被自动排除 -->
    <!-- build 元素被自动排除 -->
    <!-- 只保留 compile/runtime 依赖 -->
    <dependencies>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
    </dependencies>
</project>
```

### 手动控制 Consumer POM

```xml
<project>
    <!-- 使用 consumer-pom 配置 -->
    <consumer-pom>
        <dependencies>
            <!-- 显式声明需要暴露给消费者的依赖 -->
        </dependencies>
        <exclusions>
            <!-- 隐藏内部依赖 -->
            <exclude>com.example:internal-utils</exclude>
        </exclusions>
    </consumer-pom>
</project>
```

## 3. Config Relocation（配置迁移）

解决在 `~/.m2/settings.xml` 中配置仓库和镜像的问题，支持按项目局部化配置。

### 项目级配置：.mvn/maven.config

```properties
# .mvn/maven.config
# 不再需要在 settings.xml 配置仓库
-Drepositories=https://repo.example.com/releases,https://repo.example.com/snapshots
-DpluginRepositories=https://repo.example.com/plugins
-Dmirror.default.url=https://nexus.internal.com/repository/maven-public/
```

### 项目级 settings：.mvn/settings.xml

```xml
<!-- .mvn/settings.xml — 项目专属配置，不污染全局 ~/.m2 -->
<settings>
    <servers>
        <server>
            <id>private-repo</id>
            <username>${env.REPO_USER}</username>
            <password>${env.REPO_PASS}</password>
        </server>
    </servers>

    <profiles>
        <profile>
            <id>ci</id>
            <activation>
                <property><name>env.CI</name></property>
            </activation>
            <repositories>
                <repository>
                    <id>private-snapshot</id>
                    <url>https://repo.internal.com/snapshots</url>
                </repository>
            </repositories>
        </profile>
    </profiles>
</settings>
```

## 4. .mvn 扩展目录

Maven 4 正式支持 `.mvn/extensions/` 目录放置扩展 JAR，无需修改 POM。

```bash
# 目录结构
.mvn/
├── extensions/
│   └── maven-enforcer-extension.jar   # 自动加载
├── maven.config                       # CLI 默认参数
└── settings.xml                       # 项目级 settings
```

## 5. 性能改进

### 并行解析依赖

```bash
# Maven 4 默认并行解析依赖树（Maven 3 单线程）
# 大型多模块项目加速 2-3 倍

# 显式设置线程数
mvn clean install -T 4
```

### 增量编译支持

```bash
# 只编译变更的文件（需 maven-compiler-plugin 4.x）
mvn compile -Dmaven.compiler.incremental=true
```

### Build Cache 预览

```xml
<!-- pom.xml — 开启构建缓存 -->
<project>
    <properties>
        <!-- 缓存编译输出减少重复构建 -->
        <maven.build.cache.enabled>true</maven.build.cache.enabled>
        <maven.build.cache.dir>${user.home}/.m2/build-cache</maven.build.cache.dir>
    </properties>
</project>
```

## 6. 升级注意事项

### 从 Maven 3 升级

```bash
# 安装 Maven 4（独立安装，与 Maven 3 不冲突）
wget https://dlcdn.apache.org/maven/maven-4/4.0.0/binaries/apache-maven-4.0.0-bin.tar.gz
tar -xzf apache-maven-4.0.0-bin.tar.gz -C /opt/
export PATH=/opt/apache-maven-4.0.0/bin:$PATH
```

### 兼容性

| 功能 | Maven 3 | Maven 4 | 备注 |
|------|---------|---------|------|
| pom.xml schema | 4.0.0 | 4.1.0 | Maven 4 同时兼容 4.0.0 |
| Consumer POM | ❌ 无 | ✅ 自动 | 无需改动 POM |
| .mvn 扩展 | 实验性 | ✅ 正式 | 路径不变 |
| Local Repository | `~/.m2/repository` | 同左 | 可共享 |
| JVM 要求 | JDK 8+ | **JDK 17+** | 推荐 JDK 21 |

## 注意事项

- **JDK 要求**：Maven 4 最低 JDK 17，推荐 JDK 21+
- **插件兼容性**：部分老旧插件（如 `maven-war-plugin` < 4.0）需要升级
- **Consumer POM 覆盖**：如果消费者需要特定依赖版本，仍需在 consumer-pom 中显式声明
- **CI/CD 适配**：Jenkins/GitLab CI 需安装 Maven 4 版本
- **混合版本**：Maven 3 和 Maven 4 可共存，分别使用 `mvn3` 和 `mvn4` 命令
