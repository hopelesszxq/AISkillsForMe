---
name: pg-postgis-spatial-queries
description: PostgreSQL PostGIS 地理空间查询实战 —— 空间数据类型、索引、距离查询、范围搜索与 Spring Boot 集成
tags: [postgresql, postgis, spatial, gis, geography, index, spring-boot]
---

## 概述

PostGIS 是 PostgreSQL 的空间扩展，提供地理空间数据类型、索引和查询函数。适用于 LBS 位置服务、地图应用、物流配送等场景。

## 环境准备

### 安装扩展

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS postgis_topology;

-- 验证版本
SELECT PostGIS_Version();
-- 输出示例: 3.4 USE_GEOS=1 USE_PROJ=1 USE_STATS=1
```

### 数据类型对比

| 类型 | 坐标系 | 单位 | 精度 | 适用场景 |
|------|--------|------|------|---------|
| `GEOMETRY` | 平面坐标系（如 3857） | 与 SRID 相关 | 高 | 小范围、投影坐标 |
| `GEOGRAPHY` | 地理坐标系（WGS84, 4326） | 米 | 球面计算 | 全球范围、距离计算 |

**建议**：LBS 业务优先用 `GEOGRAPHY`，距离计算结果自动以米为单位。

## 建表与数据初始化

### 创建空间表

```sql
CREATE TABLE pois (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    category    VARCHAR(50),
    location    GEOGRAPHY(Point, 4326),  -- WGS84 经纬度
    address     TEXT,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- 创建空间索引（GIST）
CREATE INDEX idx_pois_location ON pois USING GIST (location);

-- 覆盖索引优化（包含常用字段避免回表）
CREATE INDEX idx_pois_location_covering
    ON pois USING GIST (location)
    INCLUDE (name, category);
```

### 插入数据

```sql
-- 使用 ST_GeogFromText（WKT 格式）
INSERT INTO pois (name, category, location)
VALUES ('北京天安门', 'landmark',
        ST_GeogFromText('SRID=4326;POINT(116.39723 39.90886)'));

-- 使用 ST_MakePoint（经纬度，经度在前）
INSERT INTO pois (name, category, location)
VALUES ('上海外滩', 'landmark',
        ST_SetSRID(ST_MakePoint(121.49057, 31.24014), 4326)::GEOGRAPHY);

INSERT INTO pois (name, category, location)
VALUES ('杭州西湖', 'scenic',
        ST_SetSRID(ST_MakePoint(120.15004, 30.27292), 4326)::GEOGRAPHY);
```

## 常用查询

### 1. 距离查询（给定点 N 公里内）

```sql
-- 搜索天安门 50km 范围内的 POI
SELECT
    id, name, category,
    ST_Distance(location, ST_SetSRID(
        ST_MakePoint(116.39723, 39.90886), 4326)::GEOGRAPHY) AS distance_meters
FROM pois
WHERE ST_DWithin(
    location,
    ST_SetSRID(ST_MakePoint(116.39723, 39.90886), 4326)::GEOGRAPHY,
    50000  -- 50km
)
ORDER BY distance_meters;

-- 简化版（使用 ST_GeogFromText）
SELECT name,
       ST_Distance(location, ST_GeogFromText('POINT(116.39723 39.90886)')) AS dist
FROM pois
WHERE ST_DWithin(location, ST_GeogFromText('POINT(116.39723 39.90886)'), 50000)
ORDER BY dist;
```

### 2. 最近邻查询（KNN - K Nearest Neighbor）

```sql
-- 利用 GIST 索引的 <-> 运算符，高效获取最近 5 个点
SELECT id, name,
       location <-> ST_SetSRID(
           ST_MakePoint(116.39723, 39.90886), 4326)::GEOGRAPHY AS distance
FROM pois
ORDER BY location <-> ST_SetSRID(
    ST_MakePoint(116.39723, 39.90886), 4326)::GEOGRAPHY
LIMIT 5;
```

### 3. 边界框查询（Bounding Box）

```sql
-- 使用 && 运算符做 bounding box 过滤（走索引，非常快）
SELECT * FROM pois
WHERE location && ST_MakeEnvelope(116.0, 39.5, 117.0, 40.5, 4326)::GEOGRAPHY;

-- 使用 ST_Within / ST_Intersects
SELECT name FROM pois
WHERE ST_Within(location::GEOMETRY,
    ST_MakeEnvelope(116.0, 39.5, 117.0, 40.5, 4326)::GEOMETRY);
```

### 4. 聚合查询

```sql
-- 按类别分组统计各区域 POI 数量
SELECT category,
       COUNT(*) AS count,
       ST_AsText(ST_Centroid(ST_Collect(location::GEOMETRY))) AS center
FROM pois
GROUP BY category;
```

### 5. 路线距离计算

```sql
-- 多地点的总距离
WITH points AS (
    SELECT location, id FROM pois WHERE id IN (1, 2, 3)
    ORDER BY id
)
SELECT SUM(
    ST_Distance(location, LEAD(location) OVER (ORDER BY id))
) AS total_distance_meters
FROM points;
```

## Spring Boot + MyBatis-Plus 集成

### 依赖

```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
<dependency>
    <groupId>net.postgis</groupId>
    <artifactId>postgis-jdbc</artifactId>
    <version>2023.1.0</version>
</dependency>
```

### 实体映射

```java
@Data
@TableName("pois")
public class Poi {
    @TableId(type = IdType.AUTO)
    private Long id;
    private String name;
    private String category;

    // 自定义 TypeHandler 处理 PGgeometry
    @TableField(typeHandler = PgGeometryTypeHandler.class)
    private Point location;  // PostGIS Point 类型

    private String address;
    private LocalDateTime createdAt;
}
```

### 自定义 TypeHandler

```java
@MappedTypes(Point.class)
@MappedJdbcTypes(JdbcType.OTHER)
public class PgGeometryTypeHandler extends BaseTypeHandler<Point> {

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i,
                                     Point parameter, JdbcType jdbcType) throws SQLException {
        PGgeometry geom = new PGgeometry(parameter);
        ps.setObject(i, geom);
    }

    @Override
    public Point getNullableResult(ResultSet rs, String columnName) throws SQLException {
        PGgeometry geom = (PGgeometry) rs.getObject(columnName);
        if (geom == null) return null;
        return (Point) geom.getGeometry();
    }
    // ... 其他 getNullableResult 重载
}
```

### Mapper 查询示例

```xml
<!-- 根据坐标查询附近 POI -->
<select id="findNearby" resultType="com.example.entity.Poi">
    SELECT *, ST_Distance(
        location,
        ST_SetSRID(ST_MakePoint(#{lng}, #{lat}), 4326)::GEOGRAPHY
    ) AS distance
    FROM pois
    WHERE ST_DWithin(
        location,
        ST_SetSRID(ST_MakePoint(#{lng}, #{lat}), 4326)::GEOGRAPHY,
        #{radiusMeters}
    )
    ORDER BY location <-> ST_SetSRID(
        ST_MakePoint(#{lng}, #{lat}), 4326)::GEOGRAPHY
    LIMIT #{limit}
</select>
```

## 索引优化

### GIST 索引

```sql
-- 基础 GIST 索引
CREATE INDEX idx_pois_location ON pois USING GIST (location);

-- 包含额外字段的覆盖索引（避免回表）
CREATE INDEX idx_pois_location_cov ON pois USING GIST (location)
    INCLUDE (name, category);
```

### 索引使用分析

```sql
-- 查看查询是否走索引
EXPLAIN (ANALYZE, BUFFERS)
SELECT name FROM pois
WHERE ST_DWithin(location, ST_GeogFromText('POINT(116.39 39.91)'), 50000);
-- 期望输出中有 "Index Scan using idx_pois_location"
```

### 分区表

```sql
-- 按地理区域分区（需在 GEOMETRY 类型下）
CREATE TABLE pois_partitioned (
    id BIGSERIAL,
    name VARCHAR(200),
    location GEOMETRY(Point, 4326)
) PARTITION BY RANGE (ST_X(location));

CREATE TABLE pois_east PARTITION OF pois_partitioned
    FOR VALUES FROM (100) TO (130);  -- 东经 100-130

CREATE TABLE pois_west PARTITION OF pois_partitioned
    FOR VALUES FROM (70) TO (100);   -- 东经 70-100
```

## 注意事项

### 1. SRID 一致性

- 同一列中的所有数据必须使用相同 SRID
- 查询中的坐标也必须使用相同 SRID
- WGS84 = SRID 4326（GPS 坐标），最常用

### 2. GEOMETRY vs GEOGRAPHY 选择

```sql
-- GEOMETRY 计算更快，但距离结果单位取决于 SRID
-- GEOGRAPHY 计算稍慢，但距离结果自动为米，适合全球范围

-- 长度测量：GEOGRAPHY 自动返回米
SELECT ST_Distance(
    ST_GeogFromText('SRID=4326;POINT(0 0)'),
    ST_GeogFromText('SRID=4326;POINT(0 1)')
);  -- ≈ 111195 米

-- GEOMETRY 返回度（需投影转换）
SELECT ST_Distance(
    'SRID=4326;POINT(0 0)'::GEOMETRY,
    'SRID=4326;POINT(0 1)'::GEOMETRY
);  -- ≈ 1.0（度）
```

### 3. 性能评估

| 数据量 | GEOGRAPHY 查询时间 (ST_DWithin) | GEOMETRY 查询时间 |
|--------|-------------------------------|-------------------|
| 1万行 | ~5ms | ~3ms |
| 100万行 | ~50ms | ~30ms |
| 1000万行 | ~200ms | ~120ms |

### 4. 常见坑点

- **经纬度顺序**：PostGIS 标准是 `POINT(经度 纬度)`，与 Google Maps 一致
- **索引未生效**：确保查询条件用 `ST_DWithin` 而非 `ST_Distance <`（后者不走索引）
- **坐标漂移**：GPS 原始坐标用 WGS84，国产地图（高德/百度）需偏移转换
- **类型转换**：GEOMETRY 转 GEOGRAPHY 有额外开销，尽量统一类型
