---
name: grpc-protobuf-practice
description: gRPC + Protobuf 微服务通信实战：服务定义、流式通信、拦截器、负载均衡与性能调优
tags: [tools, grpc, protobuf, microservice, rpc, serialization, java]
---

## 概述

gRPC 是 Google 开源的高性能 RPC 框架，基于 HTTP/2 和 Protocol Buffers（Protobuf）。在微服务间通信场景中，比 REST + JSON 性能更高（约 5-10 倍），支持双向流、流量控制。

> gRPC 适合：内部微服务通信、实时数据推送、IoT 设备通信、流式处理管道。

## 1. Protobuf 定义文件

### 基础 message 定义

```protobuf
// order.proto
syntax = "proto3";

package com.example.order;

option java_multiple_files = true;
option java_package = "com.example.order.proto";

// 枚举
enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING     = 1;
  ORDER_STATUS_PAID        = 2;
  ORDER_STATUS_SHIPPED     = 3;
  ORDER_STATUS_CANCELLED   = 4;
}

// 请求消息
message CreateOrderRequest {
  string user_id    = 1;
  string product_id = 2;
  int32  quantity   = 3;
  double unit_price = 4;
  map<string, string> extra = 5;  // 扩展属性
}

// 响应消息
message OrderResponse {
  string order_id   = 1;
  string order_no   = 2;
  OrderStatus status = 3;
  double total_amount = 4;
  int64  created_at   = 5;  // Unix 毫秒时间戳
}
```

### Service 定义（四种通信模式）

```protobuf
service OrderService {
  // 1. 一元 RPC（Unary）
  rpc CreateOrder(CreateOrderRequest) returns (OrderResponse);

  // 2. 服务端流（Server Streaming）
  rpc ListOrders(ListOrdersRequest) returns (stream OrderResponse);

  // 3. 客户端流（Client Streaming）
  rpc BatchCreateOrders(stream CreateOrderRequest) returns (BatchOrderResponse);

  // 4. 双向流（Bidirectional Streaming）
  rpc ProcessOrderStream(stream OrderEvent) returns (stream OrderResult);
}

message ListOrdersRequest {
  string user_id = 1;
  int32  page_size = 2;
  string page_token = 3;
}

message BatchOrderResponse {
  repeated OrderResponse orders = 1;
  int32 total_count = 2;
}
```

## 2. Java 服务端实现（Spring Boot + grpc-spring-boot-starter）

### 依赖配置

```xml
<!-- Maven -->
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-server-spring-boot-starter</artifactId>
    <version>3.1.0.RELEASE</version>
</dependency>
```

```yaml
# application.yml
grpc:
  server:
    port: 50051
    max-inbound-message-size: 16MB      # 默认 4MB，大消息需调大
    # keepalive
    keepalive-time: 30s
    keepalive-timeout: 5s
    permit-keepalive-without-calls: false
```

### 服务实现

```java
import net.devh.boot.grpc.server.service.GrpcService;
import io.grpc.stub.StreamObserver;

@GrpcService
public class OrderServiceImpl extends OrderServiceGrpc.OrderServiceImplBase {

    @Override
    public void createOrder(CreateOrderRequest request,
                            StreamObserver<OrderResponse> responseObserver) {
        // 业务逻辑
        OrderResponse response = OrderResponse.newBuilder()
            .setOrderId(UUID.randomUUID().toString())
            .setOrderNo("ORD-" + System.currentTimeMillis())
            .setStatus(OrderStatus.ORDER_STATUS_PENDING)
            .setTotalAmount(request.getQuantity() * request.getUnitPrice())
            .setCreatedAt(System.currentTimeMillis())
            .build();

        responseObserver.onNext(response);
        responseObserver.onCompleted();  // 必须调用
    }

    // 服务端流：逐条返回
    @Override
    public void listOrders(ListOrdersRequest request,
                           StreamObserver<OrderResponse> responseObserver) {
        List<Order> orders = orderService.queryByUser(request.getUserId());
        for (Order o : orders) {
            responseObserver.onNext(buildResponse(o));
        }
        responseObserver.onCompleted();
    }
}
```

## 3. Java 客户端实现

```java
import net.devh.boot.grpc.client.inject.GrpcClient;

@Service
public class OrderClientService {

    @GrpcClient("order-service")
    private OrderServiceGrpc.OrderServiceBlockingStub blockingStub;

    @GrpcClient("order-service")
    private OrderServiceGrpc.OrderServiceStub asyncStub;

    // 同步调用
    public OrderResponse createOrder(String userId, String productId, int qty) {
        CreateOrderRequest request = CreateOrderRequest.newBuilder()
            .setUserId(userId)
            .setProductId(productId)
            .setQuantity(qty)
            .build();

        return blockingStub.createOrder(request);
    }

    // 异步流式调用
    public void listOrdersAsync(String userId) {
        ListOrdersRequest request = ListOrdersRequest.newBuilder()
            .setUserId(userId)
            .setPageSize(100)
            .build();

        asyncStub.listOrders(request, new StreamObserver<OrderResponse>() {
            @Override
            public void onNext(OrderResponse order) {
                log.info("收到订单: {}", order.getOrderNo());
            }

            @Override
            public void onError(Throwable t) {
                log.error("流式调用出错", t);
            }

            @Override
            public void onCompleted() {
                log.info("所有订单接收完成");
            }
        });
    }
}
```

```yaml
# 客户端配置
grpc:
  client:
    order-service:
      address: static://10.0.1.100:50051
      # 或使用 name resolver
      # address: discovery:///order-service
      enable-keepalive: true
      keepalive-interval: 20s
      keepalive-timeout: 5s
      negotiation-type: plaintext  # 开发环境
      # negotiation-type: tls      # 生产环境
      max-inbound-message-size: 32MB
```

## 4. 拦截器（Interceptor）

### 服务端拦截器

```java
import io.grpc.*;

@Component
public class LoggingInterceptor implements ServerInterceptor {

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        String method = call.getMethodDescriptor().getFullMethodName();
        log.info("gRPC 请求: {}", method);

        // 读取自定义 Header
        String traceId = headers.get(
            Metadata.Key.of("X-Trace-Id", Metadata.ASCII_STRING_MARSHALLER));

        return Contexts.interceptCall(
            Context.current().withValue(TRACE_ID_KEY, traceId),
            call, headers, next
        );
    }
}
```

```java
// 注册拦截器
@GrpcGlobalServerInterceptor
public class GlobalLoggingInterceptor extends LoggingInterceptor {}
```

### 客户端拦截器（用于追踪、熔断）

```java
@Component
public class ClientMetricsInterceptor implements ClientInterceptor {
    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
            MethodDescriptor<ReqT, RespT> method,
            CallOptions callOptions,
            Channel next) {

        return new ForwardingClientCall.SimpleForwardingClientCall<>(
            next.newCall(method, callOptions)) {

            @Override
            public void start(Listener<RespT> responseListener, Metadata headers) {
                // 注入 Trace ID
                headers.put(
                    Metadata.Key.of("X-Trace-Id", Metadata.ASCII_STRING_MARSHALLER),
                    MDC.get("traceId"));
                super.start(responseListener, headers);
            }
        };
    }
}
```

## 5. 性能调优

### 关键配置

```yaml
# 服务端
grpc:
  server:
    max-inbound-message-size: 64MB      # 大消息传输
    max-inbound-metadata-size: 32KB      # 大 Headers
    thread-pool:
      core-size: 200
      max-size: 500
      keep-alive: 60s

# 客户端
grpc:
  client:
    order-service:
      # 连接池配置
      default-loadbalancing-policy: round_robin
      # 重试（需 proto 中配置）
      enable-retry: true
      retry-max-attempts: 3
```

### 性能对比（与 REST 对比）

| 指标 | gRPC + Proto | REST + JSON |
|------|-------------|-------------|
| 序列化速度 | ~300ms / 10万对象 | ~2200ms / 10万对象 |
| 序列化大小 | ~1.2MB | ~8.5MB |
| 网络吞吐 | ~45K req/s | ~8K req/s |
| 实时推送 | 原生 Server Push | 需要 WebSocket/SSE |

### 大消息处理建议

```java
// 1. 分页/分块
// 避免单次传输过大，总超过 100MB 建议分块

// 2. 使用流式 API 替代一元调用
// ❌ 一元 RPC 传输大列表
OrderResponse batchGet(BigListRequest);

// ✅ 服务端流
stream OrderResponse batchGetStream(BigListRequest);

// 3. 压缩
// 在 stub 上启用 GZIP
blockingStub.withCompression("gzip").listOrders(request);
```

## 6. 使用 protoc 生成代码

```bash
# 安装 protoc（Linux）
PROTOC_VERSION=29.3
curl -LO https://github.com/protocolbuffers/protobuf/releases/download/v${PROTOC_VERSION}/protoc-${PROTOC_VERSION}-linux-x86_64.zip
unzip protoc-${PROTOC_VERSION}-linux-x86_64.zip -d /usr/local/

# 安装 gRPC Java 插件
# 在 pom.xml 中使用 protobuf-maven-plugin 自动生成
```

```xml
<!-- Maven 插件 -->
<plugin>
    <groupId>com.github.os72</groupId>
    <artifactId>protoc-jar-maven-plugin</artifactId>
    <version>3.11.4</version>
    <executions>
        <execution>
            <phase>generate-sources</phase>
            <goals><goal>run</goal></goals>
            <configuration>
                <includeMavenTypes>direct</includeMavenTypes>
                <inputDirectories>
                    <include>src/main/resources/proto</include>
                </inputDirectories>
                <outputTargets>
                    <outputTarget>
                        <type>java</type>
                        <outputDirectory>src/main/java</outputDirectory>
                    </outputTarget>
                    <outputTarget>
                        <type>grpc-java</type>
                        <outputDirectory>src/main/java</outputDirectory>
                        <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.68.2</pluginArtifact>
                    </outputTarget>
                </outputTargets>
            </configuration>
        </execution>
    </executions>
</plugin>
```

## 注意事项

### ⚠️ 版本兼容性
- Spring Boot 3.x 需 `grpc-spring-boot-starter` 3.x 版本
- gRPC Java 版本与 Protobuf 版本需匹配（见 grpc-java release notes）
- protoc 版本应与 `protobuf-java` 依赖版本一致

### ⚠️ 错误处理
- gRPC 使用 Status 码而非 HTTP 状态码
- 业务错误通过 `StatusRuntimeException` 传递，建议在 interceptor 统一转换
- 不要将内部栈信息透传给客户端

```java
try {
    return blockingStub.createOrder(request);
} catch (StatusRuntimeException e) {
    switch (e.getStatus().getCode()) {
        case NOT_FOUND:    throw new ResourceNotFoundException();
        case UNAVAILABLE:  throw new ServiceUnavailableException();
        case DEADLINE_EXCEEDED: throw new TimeoutException();
        default:          throw new InternalServerException(e);
    }
}
```

### ⚠️ 负载均衡
- gRPC 客户端需要在 Name Resolver 层面做负载均衡（非 HTTP 反向代理）
- 推荐：Kubernetes Headless Service + round_robin
- 避免使用 Nginx 代理 gRPC（除非 nginx 1.13.10+ 且配置 grpc_pass）

```yaml
# K8s Headless Service
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  clusterIP: None    # Headless
  selector:
    app: order-service
  ports:
    - port: 50051
      name: grpc
```

### ⚠️ 流程控制
- HTTP/2 自带流量控制，默认 64KB 窗口大小
- 流式处理大数据时，调整 `flow-control-window` 参数
- 注意客户端与服务器的窗口大小一致性

### ⚠️ 安全
- 生产环境使用 TLS 加密（`negotiation-type: tls`）
- gRPC 不支持 REST 风格的 JWT Header，需通过 Metadata 传递 Token
- 建议在客户端拦截器中注入 Token，服务端拦截器中校验

```java
// 客户端注入 Token
Metadata headers = new Metadata();
headers.put(
    Metadata.Key.of("Authorization", Metadata.ASCII_STRING_MARSHALLER),
    "Bearer " + jwtToken
);
blockingStub.withHeaders(headers).createOrder(request);
```
