---
name: gateway-acl-request-translation
description: Spring Cloud Gateway Anti-Corruption Layer 请求翻译过滤器模式，将所有请求统一为 POST 后翻译为实际 HTTP 方法
tags: [spring-cloud, gateway, filter, acl, anti-corruption, request-decorator]
---

## 概述

Anti-Corruption Layer (ACL) 是一种微服务防损坏层模式，在 Gateway 层将客户端请求统一为特定格式（如全部 POST），再由 Gateway 翻译为下游服务期望的实际 HTTP 方法和参数格式。这种模式确保内部端点不被直接暴露，所有通信走请求体，便于后续加密/签名。

## 架构流程

```
Client ──► POST (always) ──────► Gateway + RequestTranslationFilter
           {                         │
             "targetMethod":"GET",    │ 1. 提取请求体中的 GatewayRequest
             "queryParams":{...},    │ 2. 根据 targetMethod 创建对应的请求装饰器
             "body":{...}            │ 3. 转发到下游微服务
           }                         │
                                    ▼
                              GET /api/supplies?name=...
                              POST /api/orders {body}
```

## 核心实现

### 1. 请求模型

```java
public class GatewayRequest {
    private HttpMethod targetMethod;           // 下游真实的 HTTP 方法
    private LinkedMultiValueMap<String, String> queryParams;  // GET 参数
    private Object body;                       // POST 请求体
}
```

### 2. 全局过滤器

```java
@Component
@Order(-1)
public class RequestTranslationFilter implements GlobalFilter {

    private final RequestBodyExtractor bodyExtractor;
    private final RequestDecoratorFactory decoratorFactory;

    public RequestTranslationFilter(RequestBodyExtractor bodyExtractor,
                                     RequestDecoratorFactory decoratorFactory) {
        this.bodyExtractor = bodyExtractor;
        this.decoratorFactory = decoratorFactory;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();

        // 只接受 POST + JSON
        if (request.getMethod() != HttpMethod.POST) {
            return Mono.error(new ResponseStatusException(
                HttpStatus.BAD_REQUEST, "Only POST requests are allowed"));
        }

        return bodyExtractor.extract(exchange)
            .flatMap(gatewayRequest -> {
                ServerHttpRequest decorated = decoratorFactory.create(gatewayRequest, request);
                return chain.filter(exchange.mutate().request(decorated).build());
            });
    }
}
```

### 3. 请求体提取器

```java
@Component
public class RequestBodyExtractor {

    private final ObjectMapper objectMapper;

    public Mono<GatewayRequest> extract(ServerWebExchange exchange) {
        return exchange.getRequest().getBody()
            .reduce(DataBuffer::write)
            .map(dataBuffer -> {
                byte[] bytes = new byte[dataBuffer.readableByteCount()];
                dataBuffer.read(bytes);
                DataBufferUtils.release(dataBuffer);
                try {
                    return objectMapper.readValue(bytes, GatewayRequest.class);
                } catch (IOException e) {
                    throw new ResponseStatusException(
                        HttpStatus.BAD_REQUEST, "Invalid request body", e);
                }
            });
    }
}
```

### 4. 请求装饰器工厂

```java
@Component
public class RequestDecoratorFactory {

    private final ObjectMapper objectMapper;

    public ServerHttpRequest create(GatewayRequest gatewayRequest,
                                     ServerHttpRequest original) {
        return switch (gatewayRequest.getTargetMethod()) {
            case GET -> new GetRequestDecorator(original, gatewayRequest);
            case POST -> new PostRequestDecorator(original, gatewayRequest, objectMapper);
            default -> throw new IllegalArgumentException(
                "Unsupported target method: " + gatewayRequest.getTargetMethod());
        };
    }
}
```

### 5. GET 请求装饰器

```java
public class GetRequestDecorator extends ServerHttpRequestDecorator {

    private final URI newUri;

    public GetRequestDecorator(ServerHttpRequest original, GatewayRequest req) {
        super(original);
        this.newUri = appendQueryParams(original.getURI(), req.getQueryParams());
    }

    @Override
    public HttpMethod getMethod() {
        return HttpMethod.GET;
    }

    @Override
    public URI getURI() {
        return newUri;
    }

    @Override
    public Flux<DataBuffer> getBody() {
        return Flux.empty();  // GET 请求没有 body
    }

    private URI appendQueryParams(URI uri, MultiValueMap<String, String> params) {
        if (params == null || params.isEmpty()) return uri;
        String query = params.entrySet().stream()
            .flatMap(e -> e.getValue().stream()
                .map(v -> e.getKey() + "=" + UriUtils.encode(v, "UTF-8")))
            .collect(Collectors.joining("&"));
        return URI.create(uri.toString() + (uri.getQuery() == null ? "?" : "&") + query);
    }
}
```

### 6. POST 请求装饰器

```java
public class PostRequestDecorator extends ServerHttpRequestDecorator {

    private final Flux<DataBuffer> body;

    public PostRequestDecorator(ServerHttpRequest original,
                                 GatewayRequest req,
                                 ObjectMapper objectMapper) {
        super(original);
        try {
            byte[] bytes = objectMapper.writeValueAsBytes(req.getBody());
            this.body = Flux.just(DefaultDataBufferFactory.sharedInstance.wrap(bytes));
        } catch (Exception e) {
            throw new RuntimeException("Failed to serialize request body", e);
        }
    }

    @Override
    public HttpMethod getMethod() {
        return HttpMethod.POST;
    }

    @Override
    public Flux<DataBuffer> getBody() {
        return body;
    }
}
```

## 客户端请求格式

### GET 请求翻译

```json
// 客户端 POST 到 Gateway
POST /api/supplies
Content-Type: application/json

{
  "targetMethod": "GET",
  "queryParams": {
    "name": ["paper"],
    "type": ["Stationery"]
  },
  "body": null
}

// Gateway 转发为
GET /api/supplies?name=paper&type=Stationery
```

### POST 请求翻译

```json
// 客户端 POST 到 Gateway
POST /api/orders
Content-Type: application/json

{
  "targetMethod": "POST",
  "queryParams": null,
  "body": {
    "productId": 123,
    "quantity": 2
  }
}

// Gateway 转发为
POST /api/orders
Content-Type: application/json

{"productId": 123, "quantity": 2}
```

## 注意事项

### 1. Header 处理

请求体提取后需要调整 headers：

```java
// 移除 Content-Length（body 已变化），设置 chunked 编码
public ServerHttpRequest mutateHeaders(ServerHttpRequest original) {
    return original.mutate()
        .header("Content-Length", "" )
        .header("Transfer-Encoding", "chunked")
        .build();
}
```

### 2. 安全性增强

- 可以在 Gateway 层统一做 JWT 鉴权后再转发
- 支持对请求体做字段级加密/签名
- 结合 `RequestRateLimiter` 做全局限流

### 3. 异常处理

```java
@ExceptionHandler
public ResponseEntity<Map<String, Object>> handleTranslationError(Exception e) {
    return ResponseEntity.badRequest().body(Map.of(
        "error", "REQUEST_TRANSLATION_FAILED",
        "message", e.getMessage()
    ));
}
```

### 4. 适用范围

- ✅ 内部服务间通信需要隐藏真实 API 结构
- ✅ 需要对所有请求进行统一加密/签名
- ✅ 客户端无法直接调用 RESTful 接口的场景
- ❌ 对性能有极致要求的场景（每次转发需解析 body）
- ❌ 已有稳定 API 契约的对外公开服务
