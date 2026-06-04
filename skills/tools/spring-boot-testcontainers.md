---
name: spring-boot-testcontainers
description: Spring Boot + Testcontainers 集成测试：数据库、消息队列与外部服务容器化管理
tags: [tools, testing, testcontainers, spring-boot, integration-test, docker]
---

## 概述

Testcontainers 是 Java 生态中最重要的集成测试库之一，它为测试提供一次性 Docker 容器（数据库、消息队列、缓存等），测试结束后自动销毁。Spring Boot 3.1+ 原生支持 Testcontainers 的 `@ServiceConnection` 自动配置，无需手动设置连接属性。

> 本文基于 Testcontainers 1.20.x + Spring Boot 3.x/4.x。

## 快速开始

### 依赖配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <version>1.20.4</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>rabbitmq</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

### 方式一：静态容器（推荐，重用容器减少测试时间）

```java
@SpringBootTest
@Testcontainers
class OrderServiceTest {

    // 静态容器：所有测试共用，一次启动
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private OrderService orderService;

    @Test
    void shouldCreateOrder() {
        Order order = orderService.createOrder(new CreateOrderRequest("item-1", 2));
        assertThat(order.getId()).isNotNull();
        assertThat(order.getStatus()).isEqualTo(OrderStatus.PENDING);
    }
}
```

### 方式二：使用 @ServiceConnection（Spring Boot 3.1+）

```java
@SpringBootTest
@Testcontainers
class OrderServiceTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    // @ServiceConnection 自动注册 DataSource 的 DynamicPropertySource
    // 无需手动写 @DynamicPropertySource 方法
    @Autowired
    private OrderService orderService;

    @Test
    void shouldCreateOrder() {
        // ...
    }
}
```

## 常用容器配置

### 多数据源 + Redis + RabbitMQ

```java
@SpringBootTest
@Testcontainers
class FullIntegrationTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Container
    @ServiceConnection
    static RedisContainer redis = new RedisContainer(
            DockerImageName.parse("redis:7-alpine"));

    @Container
    @ServiceConnection
    static RabbitMQContainer rabbitmq = new RabbitMQContainer(
            DockerImageName.parse("rabbitmq:3.13-management"));

    @Test
    void fullIntegration() {
        // 数据库、Redis、RabbitMQ 全部已连接
    }
}
```

### 自定义容器配置

```java
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test")
        .withInitScript("sql/init-test-data.sql")  // 初始化 DDL/DML
        .withReuse(true);  // 启用容器复用（需要 .testcontainers.properties 配置）
```

## 分层测试策略

### Repository 层测试

```java
@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class OrderRepositoryTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void shouldFindByStatus() {
        orderRepository.save(new Order(null, "item-1", OrderStatus.PENDING));
        var result = orderRepository.findByStatus(OrderStatus.PENDING);
        assertThat(result).hasSize(1);
    }
}
```

### Service 层测试（含事务回滚）

```java
@SpringBootTest
@Testcontainers
@Transactional  // 每个测试自动回滚
class OrderServiceTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired
    private OrderService orderService;

    @Test
    void shouldHandleConcurrentOrders() {
        // 测试并发情况下的数据库行为
        ExecutorService executor = Executors.newFixedThreadPool(10);
        List<Future<Order>> futures = new ArrayList<>();

        for (int i = 0; i < 10; i++) {
            int finalI = i;
            futures.add(executor.submit(() ->
                orderService.createOrder(new CreateOrderRequest("item-" + finalI, 1))
            ));
        }

        futures.forEach(f -> assertDoesNotThrow(() -> f.get()));
    }
}
```

## 高级模式

### 1. 容器复用

`.testcontainers.properties`（放在用户 home 目录）：

```properties
testcontainers.reuse.enable=true
```

容器设置 `withReuse(true)` 后，首次启动的容器在测试结束后不会销毁，下次运行直接复用，**启动时间从 30s 降到 1s**。

### 2. 共享容器（抽象基类）

```java
public abstract class AbstractIntegrationTest {

    private static final PostgreSQLContainer<?> POSTGRES;

    static {
        POSTGRES = new PostgreSQLContainer<>("postgres:16-alpine")
                .withReuse(true);
        POSTGRES.start();  // 在静态初始化块中启动
    }

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
    }
}

// 子测试类
class OrderServiceTest extends AbstractIntegrationTest {
    // 继承容器配置
}
```

### 3. MinIO 容器集成测试

```java
@Container
static GenericContainer<?> minio = new GenericContainer<>("minio/minio:latest")
        .withCommand("server /data --console-address :9001")
        .withExposedPorts(9000)
        .withEnv("MINIO_ROOT_USER", "minioadmin")
        .withEnv("MINIO_ROOT_PASSWORD", "minioadmin");

@DynamicPropertySource
static void minioProperties(DynamicPropertyRegistry registry) {
    registry.add("minio.endpoint", () ->
            String.format("http://%s:%d", minio.getHost(), minio.getMappedPort(9000)));
    registry.add("minio.access-key", () -> "minioadmin");
    registry.add("minio.secret-key", () -> "minioadmin");
}
```

### 4. 测试 Kafka（无 Testcontainers 官方模块时）

```java
@Container
static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.7.0"));
```

## CI/CD 集成

### GitHub Actions

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      docker:
        options: --privileged
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - run: |
          # Testcontainers 在 Docker-in-Docker 环境正常工作
          ./mvnw verify -Pintegration-test
```

### Docker Socket 权限

如果 CI Runner 不支持 DinD，挂载 Docker Socket：

```yaml
steps:
  - name: Verify Docker
    run: docker ps  # Testcontainers 需要通过 /var/run/docker.sock 访问
  - run: ./gradlew test
```

## 注意事项

- **启动慢**：首次启动需要拉取 Docker 镜像，建议使用 `withReuse(true)` 或预拉镜像
- **资源消耗**：每个测试类启动一个容器 → 使用静态容器（static @Container）避免每类启动一个新实例
- **端口冲突**：Testcontainers 自动分配随机端口，不要硬编码端口号
- **Docker 环境要求**：CI 需要 Docker 运行环境（Docker-in-Docker 或 Docker Socket 挂载）
- **不要用 H2 替代 PostgreSQL**：H2 与 PostgreSQL 的 SQL 方言差异会导致测试环境不一致，始终使用 Testcontainers PostgreSQL 容器
- **内存限制**：多容器测试建议设置 JVM 内存限制或使用 `@Container` + `@DirtiesContext` 精准控制容器生命周期
- **不要在 @BeforeEach 中启动容器**：容器应该用 `@Container` 注解的静态字段，在测试类的生命周期内仅启动一次
- **测试类应放在 *Test.java 中**：Maven Surefire/Failsafe 插件默认只运行特定命名模式的测试类
