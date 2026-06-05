---
name: docker-layers-xml-classpath
description: Spring Boot 4.1 自定义 layers.xml 从 classpath 加载 — Docker 镜像分层优化
tags: [tools, spring-boot, docker, layers, image-optimization]
---

## 要点

Spring Boot 4.1 (#32466) 支持从 **classpath 读取自定义 `layers.xml`**，无需将配置文件放在文件系统特定路径。这简化了 Docker 镜像分层配置的分发和管理。

**背景**：Spring Boot 从 2.3 起支持 `spring-boot-jarmode-layertools` 将 fat JAR 拆分为 Docker 分层（dependencies、snapshot-dependencies、resources、application），大幅提升镜像缓存利用率。但自定义分层配置（`layers.xml`）之前只能从文件系统加载。4.1 允许将其放在 JAR 内的 `BOOT-INF/classes/` 或类路径中，简化了交付。

## 使用方法

### 1. 创建自定义 layers.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<layers xmlns="http://www.springframework.org/schema/boot/layers"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.springframework.org/schema/boot/layers
                            https://www.springframework.org/schema/boot/layers/layers-4.1.xsd">
    <!-- 应用依赖：第三方库 -->
    <layer name="dependencies" order="0">
        <includes>
            <groupId>org.springframework.boot</groupId>
            <groupId>org.springframework</groupId>
            <groupId>io.netty</groupId>
            <groupId>com.fasterxml.jackson</groupId>
        </includes>
    </layer>

    <!-- AI/ML 模型文件：单独一层便于缓存 -->
    <layer name="models" order="1">
        <includes>
            <path>/models/**</path>
        </includes>
    </layer>

    <!-- 配置文件：变动频繁 -->
    <layer name="resources" order="2">
        <includes>
            <path>/config/**</path>
            <path>/i18n/**</path>
        </includes>
    </layer>

    <!-- 快照依赖：相对更稳定 -->
    <layer name="snapshot-dependencies" order="3">
        <includes>
            <version>*SNAPSHOT*</version>
        </includes>
    </layer>

    <!-- 应用代码：最常变动 -->
    <layer name="application" order="4">
        <includes>
            <path>/**</path>
        </includes>
    </layer>
</layers>
```

### 2. 将 layers.xml 放入 classpath

```text
src/main/resources/layers.xml
# 构建后会在 BOOT-INF/classes/layers.xml
```

### 3. 构建 Docker 镜像

```dockerfile
# 使用 bootBuildImage（Spring Boot Maven/Gradle 插件）
# 或命令行指定
FROM eclipse-temurin:21-jre AS builder
WORKDIR /app
COPY target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract --layers-config classpath:layers.xml

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=builder app/dependencies/ ./
COPY --from=builder app/models/ ./
COPY --from=builder app/snapshot-dependencies/ ./
COPY --from=builder app/resources/ ./
COPY --from=builder app/application/ ./
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

使用 `--layers-config classpath:layers.xml` 从 classpath 加载自定义分层配置。

### 4. Maven 插件配置

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <version>4.1.0</version>
    <configuration>
        <layers>
            <enabled>true</enabled>
            <!-- 引用资源文件夹中的 layers.xml -->
            <configuration>${project.basedir}/src/main/resources/layers.xml</configuration>
        </layers>
    </configuration>
</plugin>
```

## 场景示例

### 场景一：AI 模型服务单独分层

对于 Spring AI 或 LLM 推理项目，模型文件通常很大（几百 MB）但不常变动：

```xml
<layer name="models" order="1">
    <includes>
        <path>/models/**</path>
    </includes>
</layer>
```

模型层只在模型更新时重新构建，应用代码变更不影响该层缓存。

### 场景二：多环境配置分离

```xml
<layer name="config-prod" order="2">
    <includes>
        <path>/config/prod/**</path>
    </includes>
</layer>
<layer name="config-default" order="3">
    <includes>
        <path>/config/**</path>
    </includes>
</layer>
```

## 注意事项

1. **classpath 优先**：4.1 中 `--layers-config classpath:layers.xml` 会优先查找 classpath 中的文件，再回退到文件系统路径
2. **兼容性**：`layers.xml` Schema 在 4.1 中保持不变，3.x 的配置无需修改即可使用
3. **Dockerfile 优化**：利用自定义分层，应用代码变更只需重新构建 application 层，大幅减少镜像拉取时间
4. **CI/CD 集成**：将 `layers.xml` 作为项目源码管理，多项目共享分层配置
5. **默认分层**：如果不提供自定义配置，Spring Boot 使用与之前版本相同的默认分层策略
