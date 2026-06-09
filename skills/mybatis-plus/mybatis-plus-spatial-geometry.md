---
name: mybatis-plus-spatial-geometry
description: MyBatis-Plus 空间数据类型处理——JTS/PostGIS 几何类型集成、空间查询与索引优化
tags: [mybatis-plus, spatial, geometry, postgis, jts, gis]
---

## 概述

地理空间数据在业务系统中越来越常见（门店定位、配送范围、轨迹查询等）。MyBatis-Plus 结合 JTS (Java Topology Suite) 和 PostGIS/MySQL Spatial，可以优雅地处理 Point、LineString、Polygon 等空间数据类型。

本文介绍两种集成方案：自定义 TypeHandler（通用方案）和 mybatis-plus-geometry 组件（即插即用方案）。

## 方案一：自定义 TypeHandler（通用）

### 1. 依赖

```xml
<!-- JTS 核心库 -->
<dependency>
    <groupId>org.locationtech.jts</groupId>
    <artifactId>jts-core</artifactId>
    <version>1.20.0</version>
</dependency>

<!-- PostGIS JDBC -->
<dependency>
    <groupId>net.postgis</groupId>
    <artifactId>postgis-jdbc</artifactId>
    <version>2024.1.0</version>
</dependency>
```

### 2. Geometry TypeHandler

```java
import org.apache.ibatis.type.BaseTypeHandler;
import org.apache.ibatis.type.JdbcType;
import org.apache.ibatis.type.MappedTypes;
import org.locationtech.jts.geom.*;
import org.locationtech.jts.io.WKBReader;
import org.locationtech.jts.io.WKBWriter;
import org.locationtech.jts.io.WKTReader;
import org.locationtech.jts.io.WKTWriter;

import java.sql.*;

/**
 * JTS Geometry <-> PostgreSQL WKB (bytea) 互转
 */
@MappedTypes(Geometry.class)
public class GeometryTypeHandler extends BaseTypeHandler<Geometry> {

    private static final GeometryFactory GEOMETRY_FACTORY = new GeometryFactory();
    private static final WKBReader WKB_READER = new WKBReader(GEOMETRY_FACTORY);
    private static final WKBWriter WKB_WRITER = new WKBWriter(2, true); // EWKB

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, Geometry parameter, JdbcType jdbcType) throws SQLException {
        // 写入：Geometry -> EWKB bytes
        byte[] bytes = WKB_WRITER.write(parameter);
        ps.setBytes(i, bytes);
    }

    @Override
    public Geometry getNullableResult(ResultSet rs, String columnName) throws SQLException {
        byte[] bytes = rs.getBytes(columnName);
        return bytes != null ? fromWKB(bytes) : null;
    }

    @Override
    public Geometry getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        byte[] bytes = rs.getBytes(columnIndex);
        return bytes != null ? fromWKB(bytes) : null;
    }

    @Override
    public Geometry getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        byte[] bytes = cs.getBytes(columnIndex);
        return bytes != null ? fromWKB(bytes) : null;
    }

    private Geometry fromWKB(byte[] bytes) {
        try {
            return WKB_READER.read(bytes);
        } catch (Exception e) {
            throw new RuntimeException("WKB 解析失败", e);
        }
    }
    
    /**
     * 从 WKT 字符串构建几何对象
     */
    public static Geometry fromWKT(String wkt) {
        try {
            return new WKTReader(GEOMETRY_FACTORY).read(wkt);
        } catch (Exception e) {
            throw new RuntimeException("WKT 解析失败: " + wkt, e);
        }
    }
    
    /**
     * 几何对象转 WKT 
     */
    public static String toWKT(Geometry geometry) {
        return new WKTWriter().write(geometry);
    }
}
```

### 3. 实体映射

```java
@Data
@TableName("store")
public class Store {
    
    @TableId
    private Long id;
    
    private String name;
    
    /**
     * 门店位置坐标（PostGIS: geometry(Point, 4326)）
     * 经纬度：longitude, latitude
     */
    @TableField(value = "location", typeHandler = GeometryTypeHandler.class)
    private Point location;
    
    /**
     * 配送范围多边形（PostGIS: geometry(Polygon, 4326)）
     */
    @TableField(value = "delivery_zone", typeHandler = GeometryTypeHandler.class)
    private Polygon deliveryZone;
    
    /**
     * 营业时间轨迹（PostGIS: geometry(LineString, 4326)）
     */
    @TableField(value = "route", typeHandler = GeometryTypeHandler.class)
    private LineString route;
}
```

### 4. 建表 SQL

```sql
-- PostgreSQL + PostGIS
CREATE EXTENSION IF NOT EXISTS postgis;

CREATE TABLE store (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    location    GEOGRAPHY(Point, 4326),     -- WGS84 经纬度
    delivery_zone GEOGRAPHY(Polygon, 4326), -- 配送范围
    route       GEOGRAPHY(LineString, 4326) -- 路线轨迹
);

-- 空间索引（GIST）
CREATE INDEX idx_store_location ON store USING GIST (location);
CREATE INDEX idx_store_delivery_zone ON store USING GIST (delivery_zone);
```

### 5. 空间查询 Mapper

```java
@Mapper
public interface StoreMapper extends BaseMapper<Store> {

    /**
     * 查询指定范围内的门店（ST_DWithin）
     */
    @Select("SELECT * FROM store " +
            "WHERE ST_DWithin(location, ST_SetSRID(ST_MakePoint(#{lng}, #{lat}), 4326), #{radiusMeters}) " +
            "ORDER BY location <-> ST_SetSRID(ST_MakePoint(#{lng}, #{lat}), 4326) " +
            "LIMIT #{limit}")
    List<Store> findNearbyStores(@Param("lng") double lng, 
                                  @Param("lat") double lat,
                                  @Param("radiusMeters") double radiusMeters,
                                  @Param("limit") int limit);
    
    /**
     * 查询某点是否在配送范围内
     */
    @Select("SELECT COUNT(1) FROM store " +
            "WHERE id = #{storeId} " +
            "AND ST_Contains(delivery_zone, ST_SetSRID(ST_MakePoint(#{lng}, #{lat}), 4326))")
    boolean isInDeliveryZone(@Param("storeId") Long storeId,
                              @Param("lng") double lng,
                              @Param("lat") double lat);

    /**
     * 计算两点距离（米）
     */
    @Select("SELECT ST_Distance(" +
            "  ST_SetSRID(ST_MakePoint(#{lng1}, #{lat1}), 4326)::geography, " +
            "  ST_SetSRID(ST_MakePoint(#{lng2}, #{lat2}), 4326)::geography" +
            ")")
    double calculateDistance(@Param("lng1") double lng1, @Param("lat1") double lat1,
                              @Param("lng2") double lng2, @Param("lat2") double lat2);
}
```

## 方案二：mybatis-plus-geometry 组件

开源组件 `mybatis-plus-geometry` 提供自动化的 JTS 集成，自动检测数据库类型（PostGIS / MySQL Spatial）并注册对应的 TypeHandler。

### 依赖

```xml
<dependency>
    <groupId>io.github.yoy0o</groupId>
    <artifactId>mybatis-plus-geometry-spring-boot-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

### 使用方式

添加依赖后自动生效，无需手写 TypeHandler：

```java
@Data
@TableName("store")
public class Store {
    
    @TableId
    private Long id;
    
    private String name;
    
    // 自动处理 Geometry 类型
    private Point location;
    
    private Polygon deliveryZone;
}
```

### 支持的 JTS 类型

| JTS 类型 | 数据库类型 | 说明 |
|---------|-----------|------|
| `Point` | `geometry(Point, 4326)` | 点（经纬度） |
| `LineString` | `geometry(LineString, 4326)` | 线（路径、轨迹） |
| `Polygon` | `geometry(Polygon, 4326)` | 多边形（区域、范围） |
| `MultiPoint` | `geometry(MultiPoint, 4326)` | 多点集合 |
| `MultiLineString` | `geometry(MultiLineString, 4326)` | 多线集合 |
| `MultiPolygon` | `geometry(MultiPolygon, 4326)` | 多多边形集合 |
| `GeometryCollection` | `geometry(GeometryCollection, 4326)` | 几何集合 |

## 空间查询性能优化

### 1. GIST 索引

```sql
-- 务必创建 GIST 索引，否则空间查询会全表扫描
CREATE INDEX idx_geo_location ON store USING GIST (location);

-- 复合索引（索引条件下推）
CREATE INDEX idx_geo_type_status ON store USING GIST (location) 
    WHERE status = 'ACTIVE';
```

### 2. 使用 `ST_DWithin` 替代 `ST_Distance` + `<=`

```sql
-- ❌ 慢：先算所有距离再过滤（无法使用索引）
WHERE ST_Distance(location, target) <= 1000

-- ✅ 快：空间索引过滤后再精确计算（GIST 索引可选）  
WHERE ST_DWithin(location::geography, target::geography, 1000)
```

### 3. 避免 SRID 隐式转换

```sql
-- ❌ 无 SRID，PostGIS 需猜测，可能走错索引
WHERE ST_DWithin(location, ST_MakePoint(116.4, 39.9), 1000)

-- ✅ 显式指定 SRID（4326 = WGS84）
WHERE ST_DWithin(location, ST_SetSRID(ST_MakePoint(116.4, 39.9), 4326), 1000)
```

### 4. 批量写入优化

```java
public void batchInsertStores(List<Store> stores) {
    // 使用 JDBC batch，Geometry 处理在 TypeHandler 中完成
    saveBatch(stores, 500); // MyBatis-Plus batch
}
```

## 注意事项

1. **SRID 一致性**：所有几何数据必须使用相同 SRID（WGS84 = 4326），混合使用会导致计算结果错误
2. **坐标顺序**：PostGIS 接受 `ST_MakePoint(longitude, latitude)` = `(经度, 纬度)`，不要搞反
3. **Geography vs Geometry**：
   - `GEOGRAPHY`：基于球面计算，距离结果单位为**米**，支持 `ST_DWithin`
   - `GEOMETRY`：基于平面计算，性能更好，但距离结果单位为**度**
   - 经纬度数据推荐用 `GEOGRAPHY`，避免单位换算错误
4. **索引覆盖**：GIST 索引占用空间较大（约为 BTREE 的 2-3 倍），需评估磁盘
5. **精度控制**：Point 建议使用 `FLOAT8`（双精度），避免经纬度精度丢失
6. **Lambda 查询限制**：MyBatis-Plus 的 LambdaQueryWrapper 目前不直接支持空间函数，复杂空间查询仍需手写 `@Select` / XML Mapper
7. **MySQL 兼容**：MySQL 也支持 Spatial 类型但函数名不同（`ST_Distance_Sphere`），迁移时注意差异
