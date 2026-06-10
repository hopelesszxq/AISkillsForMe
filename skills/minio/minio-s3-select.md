---
name: minio-s3-select
description: MinIO S3 Select — 对 CSV/JSON/Parquet 对象执行 SQL 查询，无需下载全量数据
tags: [minio, s3-select, sql, query, parquet, csv, json, big-data]
---

## 概述

S3 Select 是 MinIO 提供的一项功能，允许通过 SQL 语句直接查询存储在对象中的结构化数据（CSV、JSON、Parquet），**只返回需要的行和列**，无需下载整个文件。适合大数据筛选、报表预处理、ETL 管道等场景。

> MinIO 从 RELEASE.2022-03-26T06-28-27Z 起支持 S3 Select。当前版本对 CSV 和 JSON 支持最成熟，Parquet 需 MinIO 企业版。

## 基本用法

### Java SDK（SelectObjectContentRequest）

```xml
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>8.5.14</version>
</dependency>
```

```java
import io.minio.*;
import io.minio.messages.*;

MinioClient client = MinioClient.builder()
    .endpoint("https://play.min.io")
    .credentials("accessKey", "secretKey")
    .build();

// 构建 S3 Select 请求
SelectObjectContentArgs args = SelectObjectContentArgs.builder()
    .bucket("my-bucket")
    .object("sales-2026-01.csv")
    .sqlExpression("SELECT region, SUM(amount) AS total FROM S3Object "
                 + "WHERE date >= '2026-01-01' AND date < '2026-02-01' "
                 + "GROUP BY region")
    .inputSerialization(b -> b.csvInput(
        new CsvInput().fileHeaderInfo(FileHeaderInfo.USE)))
    .outputSerialization(b -> b.csvOutput(new CsvOutput()))
    .requestProgress(b -> b.enable(true))
    .build();

try (SelectResponseStream response = client.selectObjectContent(args)) {
    BufferedReader reader = new BufferedReader(
        new InputStreamReader(response.getStream(), StandardCharsets.UTF_8));
    String line;
    while ((line = reader.readLine()) != null) {
        System.out.println(line); // 每行一个 CSV 记录
    }
}
```

### CSV 输入配置

```java
// CSV 输入详细配置
InputSerialization input = InputSerialization.builder()
    .csvInput(new CsvInput()
        .fileHeaderInfo(FileHeaderInfo.USE)       // 使用首行作为列名
        .allowQuotedRecordDelimiter(true)          // 允许引号内包含换行
        .recordDelimiter("\n")                     // 记录分隔符
        .fieldDelimiter(",")                       // 字段分隔符
        .quoteEscapeCharacter("\\")                // 转义符
        .comments("#")                             // 注释行前缀
    )
    .build();
```

| `FileHeaderInfo` 选项 | 说明 |
|----------------------|------|
| `USE` | 首行作为列名，可在 SQL 中按列名引用 |
| `IGNORE` | 忽略首行，SQL 中使用 `_1, _2, _3` 引用列 |
| `NONE` | 无表头，SQL 中使用 `_1, _2, _3` 引用列 |

### JSON 输入配置

```java
// JSON 输入 — 文档模式（每行一个 JSON 对象）
SelectObjectContentArgs args = SelectObjectContentArgs.builder()
    .bucket("logs")
    .object("app-logs-2026.json")
    .sqlExpression("SELECT level, COUNT(*) AS cnt FROM S3Object "
                 + "WHERE timestamp >= '2026-01-01' "
                 + "GROUP BY level")
    .inputSerialization(b -> b.jsonInput(
        new JsonInput().type(JsonType.DOCUMENT)))
    .outputSerialization(b -> b.jsonOutput(new JsonOutput()))
    .build();
```

**JSON 两种输入类型**：

| 类型 | 描述 | 示例文件 |
|------|------|---------|
| `DOCUMENT` | 每行一个独立的 JSON 对象（JSON Lines） | `{"id":1,"val":"a"}\n{"id":2,"val":"b"}` |
| `LINES` | 数组格式，整个文件一个 JSON 数组 | `[{"id":1},{"id":2}]` |

### Parquet 输入（MinIO 企业版）

```java
// Parquet 输入 — 需要 MinIO Enterprise
SelectObjectContentArgs args = SelectObjectContentArgs.builder()
    .bucket("analytics")
    .object("events.parquet")
    .sqlExpression("SELECT event_type, COUNT(*) FROM S3Object "
                 + "WHERE ts >= CAST('2026-01-01' AS TIMESTAMP) "
                 + "GROUP BY event_type")
    .inputSerialization(b -> b.parquetInput(new ParquetInput()))
    .outputSerialization(b -> b.csvOutput(new CsvOutput()))
    .build();
```

## 高级查询模式

### 使用聚合函数

```sql
-- 支持的聚合函数
SELECT COUNT(*) FROM S3Object
SELECT COUNT(column_name) FROM S3Object
SELECT SUM(amount) FROM S3Object WHERE region = '华东'
SELECT AVG(price) FROM S3Object
SELECT MIN(price), MAX(price) FROM S3Object
```

### 条件过滤 + 投影

```sql
-- 只返回指定列，减少网络传输
SELECT order_id, customer_name, total
FROM S3Object
WHERE total > 1000 AND created_at >= '2026-01-01'
ORDER BY total DESC
LIMIT 100
```

### 嵌套 JSON 查询

```json
{"user": {"name": "张三", "tags": ["vip", "active"]}, "score": 95}
```

```sql
-- 访问嵌套字段（使用点号语法）
SELECT user.name, user.tags[0] AS primary_tag
FROM S3Object
WHERE score > 80
```

### 字符串与日期函数

```sql
-- 字符串操作
SELECT UPPER(name), SUBSTRING(description, 1, 50), TRIM(code)
FROM S3Object

-- 类型转换
SELECT CAST(price AS FLOAT), CAST(created_at AS TIMESTAMP)
FROM S3Object

-- 日期提取
SELECT EXTRACT(YEAR FROM created_at) AS year,
       EXTRACT(MONTH FROM created_at) AS month
FROM S3Object
```

## 性能优化

### 1. 列裁剪

```java
// ❌ 不推荐：查询所有列
String sql = "SELECT * FROM S3Object WHERE date >= '2026-01-01'";

// ✅ 推荐：只选择需要的列
String sql = "SELECT id, name, amount, region FROM S3Object WHERE ...";
```

### 2. 压缩对象

```java
// 对 GZIP 压缩的 CSV 启用解压缩
SelectObjectContentArgs args = SelectObjectContentArgs.builder()
    .bucket("data")
    .object("sales.csv.gz")
    .sqlExpression("SELECT * FROM S3Object")
    .inputSerialization(b -> b
        .csvInput(new CsvInput().fileHeaderInfo(FileHeaderInfo.USE))
        .compressionType(CompressionType.GZIP))
    // .compressionType(CompressionType.BZIP2)  // 也支持 BZIP2
    // .compressionType(CompressionType.NONE)    // 无压缩（默认）
    .build();
```

### 3. 扫描范围限定

```java
// 只扫描文件的前 N 行（适合抽样预览）
SelectObjectContentArgs args = SelectObjectContentArgs.builder()
    .bucket("data")
    .object("large-file.csv")
    .sqlExpression("SELECT * FROM S3Object LIMIT 1000")
    .inputSerialization(b -> b.csvInput(
        new CsvInput().fileHeaderInfo(FileHeaderInfo.USE)))
    .build();
```

## 错误处理

```java
try (SelectResponseStream response = client.selectObjectContent(args)) {
    // 处理数据流
} catch (MinioException e) {
    // S3 Select 常见错误
    switch (e.errorResponse().code()) {
        case "InvalidTextEncoding":
            // CSV/JSON 编码不正确
            break;
        case "MissingRequestBody":
            // SQL 表达式为空
            break;
        case "InvalidExpressionType":
            // SQL 语法错误
            break;
        case "UnsupportedOperation":
            // 当前版本不支持此操作（如 Parquet 在社区版）
            break;
    }
}
```

## 与传统方式对比

| 场景 | 传统方式 | S3 Select 方式 |
|------|---------|---------------|
| 筛选 10GB CSV 中符合条件的数据 | 下载整文件 → 本地解析 → 过滤 | 发送 SQL → 只返回结果行 |
| 提取 JSON 中特定字段 | 下载全量 → 反序列化 → 提取 | SQL 投影列 → 直接返回 |
| 数据预览 | 下载 10GB 文件查看前100行 | `SELECT * LIMIT 100` |
| 跨文件聚合 | 需要外部引擎（Spark/Presto） | 当前不支持单查询跨文件 |

## 注意事项

1. **单文件查询**：S3 Select 一次只能查询一个对象，不支持跨文件 JOIN 或 UNION
2. **Parquet 限制**：社区版 MinIO 不支持 Parquet 输入，需要 MinIO Enterprise 或 SUBNET 授权
3. **SQL 语法子集**：不支持 `HAVING`、子查询、窗口函数等高级 SQL 功能
4. **数据类型推断**：CSV 模式下列所有值以字符串处理，需要在 SQL 中用 `CAST()` 转换
5. **扫描计费**：MinIO 按扫描的数据量计费（虽然常用于本地部署），大量扫描可能影响性能
6. **大文件性能**：对于 > 100MB 的文件，S3 Select 比下载全量快 2-5 倍（仅返回需要的数据）
7. **请求进度**：开启 `requestProgress` 后可监控扫描进度，但会增加少量开销
