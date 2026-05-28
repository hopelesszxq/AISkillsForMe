---
name: maven-practice
description: Maven 构建工具最佳实践：多模块、依赖管理、性能优化与 CI/CD 配置
tags: [tools, maven, build, java, dependency]
---

## 多模块项目结构

### 推荐结构

```
my-project/
├── pom.xml                      # 父 POM（packaging: pom）
├── my-project-common/           # 通用工具模块
│   └── pom.xml
├── my-project-dal/              # 数据访问模块
│   └── pom.xml
├── my-project-service/          # 业务服务模块
│   └── pom.xml
└── my-project-web/              # Web 接口模块
    └── pom.xml
```

### 父 POM 最佳实践

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>my-project</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <modules>
        <module>my-project-common</module>
        <module>my-project-dal</module>
        <module>my-project-service</module>
        <module>my-project-web</module>
    </modules>

    <!-- 统一版本管理 -->
    <properties>
        <java.version>17</java.version>
        <maven.compiler.source>${java.version}</maven.compiler.source>
        <maven.compiler.target>${java.version}</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

        <!-- 依赖版本 -->
        <spring-boot.version>3.2.5</spring-boot.version>
        <spring-cloud.version>2023.0.1</spring-cloud.version>
        <mybatis-plus.version>3.5.7</mybatis-plus.version>
        <hutool.version>5.8.28</hutool.version>
        <guava.version>33.2.0-jre</guava.version>
    </properties>

    <!-- 依赖版本管理（只声明版本，不引入依赖） -->
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>com.baomidou</groupId>
                <artifactId>mybatis-plus-boot-starter</artifactId>
                <version>${mybatis-plus.version}</version>
            </dependency>
            <dependency>
                <groupId>cn.hutool</groupId>
                <artifactId>hutool-all</artifactId>
                <version>${hutool.version}</version>
            </dependency>
            <dependency>
                <groupId>com.google.guava</groupId>
                <artifactId>guava</artifactId>
                <version>${guava.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <!-- 所有子模块都继承的依赖 -->
    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>

    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <version>${spring-boot.version}</version>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
```

## 依赖管理原则

### 1. 三层依赖规则

| 层级 | POM 角色 | 声明方式 |
|------|----------|----------|
| 父 POM | 版本仲裁 | `dependencyManagement` |
| BOM 模块 | 集中管理外部依赖版本 | `dependencyManagement` + `import scope` |
| 业务模块 | 按需引用，不写版本 | 只写 `groupId` + `artifactId` |

### 2. 依赖收敛

```bash
# 查看依赖树，检查冲突
mvn dependency:tree -Dverbose

# 查看未使用的依赖
mvn dependency:analyze
```

### 3. 排除不必要的传递依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<!-- 替换为 Undertow（性能更好） -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

## 构建性能优化

### 1. 多线程构建

```bash
# 使用 CPU 核心数构建
mvn clean install -T 4

# 每个模块独立线程
mvn clean install -T 1C
```

### 2. 跳过测试（CI 中单独运行）

```bash
# 跳过测试编译和执行
mvn clean package -DskipTests

# 只跳过执行，保留编译（用于检查测试编译是否正确）
mvn clean package -Dmaven.test.skip=true
```

### 3. 增量编译

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <source>17</source>
        <target>17</target>
        <parameters>true</parameters>
        <!-- 启用增量编译 -->
        <useIncrementalCompilation>true</useIncrementalCompilation>
    </configuration>
</plugin>
```

### 4. Maven Daemon（mvnd）

```bash
# 安装 mvnd（比 mvn 快 3-8 倍）
# macOS
brew install mvnd

# 使用方式完全兼容
mvnd clean install -T 4
```

## CI/CD 配置示例

### GitHub Actions

```yaml
name: Java CI with Maven
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven  # 缓存 Maven 依赖
      - name: Build with Maven
        run: mvn -B clean verify --file pom.xml
      - name: Upload coverage
        uses: codecov/codecov-action@v4
```

## 常用插件配置

### 1. 打包源码

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-source-plugin</artifactId>
    <executions>
        <execution>
            <id>attach-sources</id>
            <goals><goal>jar-no-fork</goal></goals>
        </execution>
    </executions>
</plugin>
```

### 2. 代码检查

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-checkstyle-plugin</artifactId>
    <version>3.3.1</version>
    <configuration>
        <configLocation>checkstyle.xml</configLocation>
        <failOnViolation>true</failOnViolation>
    </configuration>
</plugin>
```

### 3. 生成文档

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-javadoc-plugin</artifactId>
    <configuration>
        <show>protected</show>
        <failOnWarning>false</failOnWarning>
    </configuration>
</plugin>
```

## 常见坑点

### 1. SNAPSHOT 依赖更新不及时

```xml
<!-- settings.xml 中配置快照更新策略 -->
<profile>
    <id>nexus</id>
    <repositories>
        <repository>
            <id>nexus-snapshots</id>
            <url>https://nexus.example.com/repository/maven-snapshots/</url>
            <snapshots>
                <enabled>true</enabled>
                <updatePolicy>always</updatePolicy>  <!-- always / daily / interval:60 -->
            </snapshots>
        </repository>
    </repositories>
</profile>
```

### 2. 子模块版本统一

```xml
<!-- 父 POM 设置版本属性，子模块引用 -->
<properties>
    <revision>1.0.0-SNAPSHOT</revision>
</properties>

<!-- 子模块引用 -->
<version>${revision}</version>
```

### 3. 不要提交 target/ 目录

```gitignore
target/
*.class
*.jar
*.war
!**/src/main/**/target/
```

### 4. settings.xml 安全配置

```xml
<!-- credentials 不要明文写在 pom.xml 中 -->
<servers>
    <server>
        <id>nexus-releases</id>
        <username>${env.NEXUS_USERNAME}</username>
        <password>${env.NEXUS_PASSWORD}</password>
    </server>
</servers>
```
