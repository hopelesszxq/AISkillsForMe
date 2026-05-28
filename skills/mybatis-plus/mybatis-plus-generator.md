---
name: mybatis-plus-generator
description: MyBatis-Plus 代码生成器实战：自定义模板、分模块生成、数据库逆向工程
tags: [mybatis-plus, generator, codegen, spring-boot, scaffold]
---

## 概述

MyBatis-Plus 代码生成器（Code Generator）可基于数据库表结构自动生成 Entity、Mapper、Service、Controller 等全套代码。3.5.x 版本引入了全新的生成器 API（不再依赖旧版 AutoGenerator），支持自定义模板、分模块输出。

## 1. 快速开始（3.5.x 新版 API）

### 依赖

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-generator</artifactId>
    <version>3.5.9</version>
</dependency>
<dependency>
    <groupId>org.apache.velocity</groupId>
    <artifactId>velocity-engine-core</artifactId>
    <version>2.3</version>
</dependency>
```

### 代码生成器主类

```java
public class MyBatisPlusGenerator {

    public static void main(String[] args) {
        // 数据源配置
        FastAutoGenerator.create("jdbc:mysql://localhost:3306/db_order?useUnicode=true&characterEncoding=utf-8",
                                 "root", "root")
            // 全局配置
            .globalConfig(builder -> builder
                .author("developer")
                .outputDir(System.getProperty("user.dir") + "/src/main/java")
                .commentDate("yyyy-MM-dd")
                .disableOpenDir()       // 不自动打开输出目录
            )
            // 包配置
            .packageConfig(builder -> builder
                .parent("com.example.order")
                .moduleName("order")
                .entity("entity")
                .mapper("mapper")
                .service("service")
                .controller("controller")
            )
            // 策略配置
            .strategyConfig(builder -> builder
                .addInclude("t_order", "t_order_item")  // 表名
                .addTablePrefix("t_")                    // 表前缀过滤
                .entityBuilder()
                    .enableLombok()                      // @Data
                    .enableTableFieldAnnotation()        // @TableField 注解
                    .versionColumnName("version")        // 乐观锁字段
                    .logicDeleteColumnName("deleted")    // 逻辑删除字段
                .serviceBuilder()
                    .formatServiceFileName("%sService")
                    .formatServiceImplFileName("%sServiceImpl")
                .controllerBuilder()
                    .enableRestStyle()                   // @RestController
                    .enableHyphenStyle()                 // /order-item 命名
            )
            .templateConfig(builder -> builder
                .controller("/templates/controller.java.vm")
            )
            .execute();
    }
}
```

## 2. 分模块生成配置

### 多模块项目结构

```text
order-parent/
├── order-api/          # 对外接口 DTO
├── order-service/      # 业务实现
│   └── src/main/java/
│       ├── entity/
│       ├── mapper/
│       └── service/
└── order-web/          # 控制器层
    └── src/main/java/
        └── controller/
```

### 分模块生成配置

```java
FastAutoGenerator.create(url, username, password)
    // Entity + Mapper → order-service 模块
    .packageConfig(builder -> builder
        .parent("com.example.order.service")
        .entity("entity")
        .mapper("mapper")
        .service("service")
        .pathInfo(Collections.singletonMap(
            OutputFile.entity,
            "/path/to/order-service/src/main/java/com/example/order/service/entity"
        ))
        .pathInfo(Collections.singletonMap(
            OutputFile.mapper,
            "/path/to/order-service/src/main/java/com/example/order/service/mapper"
        ))
    )
    // Controller → order-web 模块
    .strategyConfig(builder -> builder
        .controllerBuilder()
            .outputDir("/path/to/order-web/src/main/java/com/example/order/web/controller")
    )
    .execute();
```

## 3. 自定义模板

### 模板引擎选择

| 引擎 | 依赖 |
|---|---|
| Velocity（默认） | `velocity-engine-core` |
| Freemarker | `freemarker` |
| Mustache | `mustache` |

### 自定义 Service 模板

在 `resources/templates/service.java.vm` 中：

```java
package ${package.Service};

import ${package.Entity}.${entity};
import ${superServiceClassPackage};
import java.util.List;

/**
 * $!{table.comment} 服务接口
 *
 * @author ${author}
 * @since ${date}
 */
#if(${kotlin})
interface ${table.serviceName} : ${superServiceClass}<${entity}> {
#else
public interface ${table.serviceName} extends ${superServiceClass}<${entity}> {

    /**
     * 批量插入（自定义方法）
     */
    boolean saveBatch(List<${entity}> list, int batchSize);

    /**
     * 按条件删除并返回删除数量
     */
    int deleteByCondition(${entity} condition);
}
#end
```

### 自定义 Controller 模板

```java
package ${package.Controller};

import ${package.Entity}.${entity};
import ${package.Service}.${table.serviceName};
import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import org.springframework.web.bind.annotation.*;
import lombok.RequiredArgsConstructor;

/**
 * $!{table.comment} REST API
 *
 * @author ${author}
 * @since ${date}
 */
@RestController
@RequestMapping("/api/${table.entityPath}")
@RequiredArgsConstructor
public class ${table.controllerName} {

    private final ${table.serviceName} ${table.entityPath}Service;

    @GetMapping("/page")
    public Result<Page<${entity}>> page(
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "10") int size) {
        return Result.success(${table.entityPath}Service.page(new Page<>(page, size)));
    }

    @GetMapping("/{id}")
    public Result<${entity}> get(@PathVariable Long id) {
        return Result.success(${table.entityPath}Service.getById(id));
    }

    @PostMapping
    public Result<Boolean> create(@RequestBody ${entity} entity) {
        return Result.success(${table.entityPath}Service.save(entity));
    }

    @DeleteMapping("/{id}")
    public Result<Boolean> delete(@PathVariable Long id) {
        return Result.success(${table.entityPath}Service.removeById(id));
    }
}
```

## 4. 数据库逆向工程

### 从多个数据源生成

```java
// Oracle 数据源
FastAutoGenerator.create(
    "jdbc:oracle:thin:@localhost:1521:xe",
    "scott", "tiger"
)
.strategyConfig(builder -> builder
    .addInclude("EMP", "DEPT")
    .addTablePrefix("T_")
).execute();
```

### 排除特定表和字段

```java
.strategyConfig(builder -> builder
    .addInclude("t_order", "t_order_item")  // 只生成这些表
    .addExclude("flyway_schema_history", "sys_config")  // 排除非业务表
    .entityBuilder()
        .addTableFills(
            new Column("create_time", FieldFill.INSERT),
            new Column("update_time", FieldFill.INSERT_UPDATE)
        )
        .addIgnoreColumns("id", "version", "deleted")  // 忽略这些字段
)
```

## 5. 生成后自动格式化

```java
.globalConfig(builder -> builder
    .outputDir("/tmp/gen-code")
)
.templateConfig(builder -> builder
    .disable()  // 禁用默认模板
)
.execute();

// 生成后自动格式化（调用 Spotless 或本地工具）
public static void formatCode(String projectPath) {
    try {
        ProcessBuilder pb = new ProcessBuilder(
            "mvn", "spotless:apply", "-f", projectPath + "/pom.xml"
        );
        pb.inheritIO().start().waitFor();
    } catch (Exception e) {
        System.err.println("格式化失败: " + e.getMessage());
    }
}
```

## 注意事项

1. **版本匹配**：MyBatis-Plus Generator 3.5.x 必须对应 mybatis-plus-boot-starter 3.5.x，否则生成代码编译报错
2. **模板引擎冲突**：同时引入 Velocity 和 Freemarker 会导致错误，只保留一个
3. **表前缀处理**：`addTablePrefix("t_")` 后生成的 Entity 名称为 `Order`（POJO）、表名为 `t_order`（SQL）
4. **覆盖策略**：生成器默认**覆盖**已有文件，建议先备份或使用 Git 管理后再重新生成
5. **增量生成**：如果表结构变更，再次运行生成器会覆盖已有改动。推荐用 `@TableField(exist = false)` 标记不需要持久化的自定义字段，避免被覆盖
6. **自定义模板路径**：模板文件放在 `resources/templates/` 下，命名格式为 `xxx.java.vm`（Velocity）、`xxx.ftl`（Freemarker）
7. **Controller 风格**：`enableRestStyle()` 生成 `@RestController`，不开则生成 `@Controller`
