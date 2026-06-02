---
name: spring-boot-41-grpc-support
description: Spring Boot 4.1 原生 gRPC 支持：@GrpcAdvice、gRPC Starter、服务端/客户端自动配置
tags: [tools, spring-boot, grpc, microservice, rpc, protobuf]
---

## 概述

Spring Boot 4.1.0（RC1 于 2026-04-23 发布）引入了 **原生 gRPC 支持**，不再需要第三方 starter（如 grpc-spring-boot-starter）。内建了服务端和客户端的自动配置、`@GrpcAdvice` 异常处理、以及完整的测试支持。

> 本文聚焦 Spring Boot 4.1 的新增 gRPC 原生集成，通用 gRPC/Protobuf 实践请参考 `grpc-protobuf-practice.md`。

## 一、Maven 依赖

### 服务端

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-grpc-server</artifactId>
</dependency>
```

### 客户端

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-grpc-client</artifactId>
</dependency>
```

### 测试

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-grpc-client-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-grpc-server-test</artifactId>
    <scope>test</scope>
</dependency>
```

## 二、服务端配置

### application.properties

```properties
# gRPC 服务端端口（默认 9090）
spring.grpc.server.port=9090

# 安全配置
spring.grpc.server.security.csrf.enabled=true
```

### 定义 gRPC 服务

```java
import org.springframework.grpc.server.service.GrpcService;

@GrpcService
public class OrderGrpcService extends OrderServiceGrpc.OrderServiceImplBase {

    private final OrderRepository orderRepository;

    public OrderGrpcService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Override
    public void createOrder(CreateOrderRequest request,
                            StreamObserver<CreateOrderResponse> responseObserver) {
        Order order = orderRepository.save(new Order(request.getUserId(),
                                                      request.getProductId(),
                                                      request.getQuantity()));
        CreateOrderResponse response = CreateOrderResponse.newBuilder()
            .setOrderId(order.getId())
            .setStatus("CREATED")
            .build();
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
}
```

### 全局异常处理（`@GrpcAdvice`）

```java
import org.springframework.grpc.server.exception.GrpcAdvice;
import org.springframework.grpc.server.exception.GrpcExceptionHandler;
import io.grpc.Status;

@GrpcAdvice
public class GrpcExceptionAdvice {

    @GrpcExceptionHandler(IllegalArgumentException.class)
    public Status handleInvalidArgument(IllegalArgumentException e) {
        return Status.INVALID_ARGUMENT
            .withDescription(e.getMessage())
            .withCause(e);
    }

    @GrpcExceptionHandler(ResourceNotFoundException.class)
    public Status handleNotFound(ResourceNotFoundException e) {
        return Status.NOT_FOUND
            .withDescription(e.getMessage());
    }

    @GrpcExceptionHandler(Exception.class)
    public Status handleGeneral(Exception e) {
        return Status.INTERNAL
            .withDescription("Internal server error");
    }
}
```

## 三、客户端配置

### application.properties

```properties
# 目标服务地址
spring.grpc.client.default.target=localhost:9090

# 连接超时
spring.grpc.client.default.negotiation-type=PLAINTEXT
spring.grpc.client.default.default-load-balancing-policy=round_robin
```

### 使用 gRPC 客户端

```java
import org.springframework.grpc.client.service.GrpcClient;

@Service
public class OrderClientService {

    @GrpcClient("order-service")
    private OrderServiceGrpc.OrderServiceBlockingStub orderStub;

    public String createOrder(String userId, String productId, int qty) {
        CreateOrderRequest request = CreateOrderRequest.newBuilder()
            .setUserId(userId)
            .setProductId(productId)
            .setQuantity(qty)
            .build();

        CreateOrderResponse response = orderStub.createOrder(request);
        return response.getOrderId();
    }
}
```

### 响应式/异步客户端

```java
@GrpcClient("order-service")
private OrderServiceGrpc.OrderServiceFutureStub futureStub;

@GrpcClient("order-service")
private OrderServiceGrpc.OrderServiceStub asyncStub;
```

## 四、测试支持

### 服务端测试

```java
@SpringBootTest
@ImportAutoConfiguration(GrpcServerTestAutoConfiguration.class)
class OrderGrpcServiceTest {

    @Autowired
    private GrpcServerTestClient testClient;

    @Test
    void shouldCreateOrder() {
        CreateOrderRequest request = CreateOrderRequest.newBuilder()
            .setUserId("user-1")
            .setProductId("prod-1")
            .setQuantity(2)
            .build();

        CreateOrderResponse response = testClient
            .send(request, OrderServiceGrpc.getCreateOrderMethod());

        assertThat(response.getOrderId()).isNotBlank();
        assertThat(response.getStatus()).isEqualTo("CREATED");
    }
}
```

### 客户端测试

```java
@SpringBootTest
@ImportAutoConfiguration(GrpcClientTestAutoConfiguration.class)
class OrderClientServiceTest {

    @MockBean
    private OrderServiceGrpc.OrderServiceImplBase mockService;

    @Autowired
    private OrderClientService clientService;

    @Test
    void shouldCallGrpcService() {
        // Mock gRPC 响应
        doAnswer(invocation -> {
            StreamObserver<CreateOrderResponse> observer = invocation.getArgument(1);
            observer.onNext(CreateOrderResponse.newBuilder()
                .setOrderId("mock-123")
                .setStatus("CREATED")
                .build());
            observer.onCompleted();
            return null;
        }).when(mockService).createOrder(any(), any());

        String orderId = clientService.createOrder("user-1", "prod-1", 2);
        assertThat(orderId).isEqualTo("mock-123");
    }
}
```

## 五、与第三方客户端的区别

| 方面 | 三方 starter（grpc-spring-boot-starter） | Spring Boot 4.1 原生 |
|------|-----------------------------------------|---------------------|
| 维护方 | 社区（Lognet/Google） | Spring 官方 |
| 版本兼容 | 需匹配 Boot 版本 | 随 Boot 版本自动适配 |
| @GrpcAdvice | 不支持 | 原生支持 |
| 测试支持 | 需手动配置 | 内置 Test Starter |
| 安全集成 | 需额外配置 | CSRF 内置支持 |
| Spring Security 集成 | 手动 | 自动集成 |

## 注意事项

1. **JDK 要求**：Spring Boot 4.x 最低 JDK 21（推荐 JDK 25）
2. **Protobuf 编译**：仍需使用 `protobuf-maven-plugin` 或 `protobuf-gradle-plugin` 生成 Java 代码
3. **端口冲突**：默认 gRPC 端口 9090，与 HTTP 端口（8080）不同，注意防火墙配置
4. **负载均衡**：客户端侧负载均衡使用 `round_robin`，生产环境建议配合注册中心（Nacos/Consul）使用
5. **安全性**：生产环境务必启用 TLS（`negotiation-type=TLS`）并配置证书
6. **方法名冲突**：`@GrpcAdvice` 的方法不能有同名重载，每个异常类型只能对应一个处理方法
