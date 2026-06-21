---
name: istio-130-agent-gateway
description: Istio 1.30 Agent Gateway（AI Agent/MCP 流量代理）、TLSRoute 增强、Ambient 模式改进、TrafficExtension API 统一扩展
tags: [istio, service-mesh, ambient, agent-gateway, ai, kubernetes, pattern]
---

## 概述

Istio 1.30.0（2026-06-04 发布）是服务网格领域的重要更新，引入了面向 AI Agent 场景的 **Agent Gateway**（实验性）、Gateway API **TLSRoute** 终止/混合模式支持、Ambient 模式多项增强，以及统一的 **TrafficExtension** API。支持的 Kubernetes 版本为 1.32~1.36。

## 1. Agent Gateway（实验性）

面向 AI Agent 和 MCP（Model Context Protocol）服务器流量的新数据面代理。在 Gateway Pod 上用专用代理替换 Envoy。

### 启用方式

```yaml
# 在 istiod 上启用 Agent Gateway
# 设置环境变量
PILOT_ENABLE_AGENTGATEWAY=true
```

```yaml
# 使用 Agent Gateway
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: istio-agentgateway
spec:
  controllerName: istio.io/agent-gateway-controller
```

### 当前限制

| 项目 | 状态 |
|------|------|
| Gateway API 实现 | ✅ 单 GatewayClass (`istio-agentgateway`) |
| Sidecar 模式 | ❌ 不支持 |
| Waypoint 代理 | ❌ 不支持 |
| 生产环境 | ❌ 实验性，早期功能 |

### 适用场景

- AI Agent 之间的消息通信
- MCP 服务器流量路由
- LLM API Gateway 场景

## 2. Gateway API TLSRoute 增强

### TLSRoute 终止与混合模式

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: TLSRoute
metadata:
  name: tls-passthrough
spec:
  parentRefs:
  - name: eastwest-gateway
  rules:
  - backendRefs:
    - name: my-service
      port: 443
```

新增支持：
- **TLS 终止模式**：Gateway 终止 TLS 连接，后端接收明文
- **TLS 混合模式**：部分路由终止、部分透传
- **东西向 Gateway TLS 透传**：east-west Gateway 支持 TLS passthrough 监听器
- **ListenerSet 状态报告**：Gateway 状态中报告已关联的 ListenerSet 和路由

## 3. Ambient 模式增强

### 3.1 ServiceEntry CIDR 地址支持

```yaml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: external-cidr
spec:
  hosts:
  - "*.external-service.com"
  addresses:
  - 10.0.0.0/24    # CIDR 范围，无需逐个枚举 IP
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
```

### 3.2 Waypoint XFCC 合成

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: waypoint
  annotations:
    ambient.istio.io/xfcc-include-client-identity: "true"  # 合成 X-Forwarded-Client-Cert
spec:
  gatewayClassName: istio-waypoint
  # ...
```

Waypoint 从 ztunnel 提供的来源工作负载 SPIFFE 身份合成 `x-forwarded-client-cert` 头，上游应用可识别原始客户端身份。

### 3.3 HBONE 窗口大小可配置

```yaml
# istiod 环境变量
PILOT_HBONE_INITIAL_STREAM_WINDOW_SIZE: 65536      # 流窗口（默认）
PILOT_HBONE_INITIAL_CONNECTION_WINDOW_SIZE: 1048576  # 连接窗口（默认）
```

用于调优高吞吐 Ambient 工作负载的 HBONE CONNECT 集群性能。

### 3.4 ztunnel Tokio 运行时指标

ztunnel 暴露 Tokio 运行时指标，提升每实例资源可见性：

```yaml
# ztunnel 配置
adminAddress: "127.0.0.1:15000"
# 访问: curl http://127.0.0.1:15000/metrics
# 可查看 Tokio 线程池使用率、任务队列等
```

### 3.5 Sidecar→Ambient 迁移指南

新增官方逐步迁移指南，覆盖：
1. Ambient 组件安装
2. 策略迁移
3. 按命名空间启用
4. 渐进式/可逆迁移设计——Sidecar 和 Ambient 工作负载可在迁移期间共存

## 4. TrafficExtension API（统一可扩展性）

统一的 Wasm/Lua 扩展 API，取代 `WasmPlugin` 成为主要代理扩展机制，适用于：

- Envoy Sidecar
- Gateway
- Waypoint

```yaml
apiVersion: extensions.istio.io/v1
kind: TrafficExtension
metadata:
  name: auth-filter
spec:
  workloadSelector:
    matchLabels:
      app: my-service
  phase: AUTHN
  priorities:
  - name: oauth
    wasm:
      url: oci://registry.example.com/oauth-filter:v1
  - name: rate-limit
    lua: |
      function envoy_on_request(request_handle)
        -- 限流逻辑
      end
```

## 5. 流量管理增强

### 5.1 命名空间级流量分布注解

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  annotations:
    traffic.istio.io/distribution: "round_robin"  # 命名空间默认策略
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    traffic.istio.io/connect-strategy: "RACE_FIRST_TCP_CONNECT"  # TCP 竞速策略
```

- 服务未显式设置流量分布时，继承命名空间注解——减少每个服务的重复配置
- `connect-strategy` 支持 `RACE_FIRST_TCP_CONNECT`：DNS 返回多个 A 记录时，选择第一个成功完成 TCP 连接的后端

### 5.2 DNS 超时配置

```yaml
# istiod 环境变量
DNS_FORWARD_TIMEOUT: 5s  # DNS 上游超时（默认 5s，可自定义）
```

### 5.3 多 CUSTOM 认证提供者

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: multi-auth
spec:
  rules:
  - to:
    - operation:
        paths: ["/api/oauth"]
        ports: ["443"]
    when:
    - key: request.auth.authentications
      values: ["oauth-provider"]
  - to:
    - operation:
        paths: ["/api/ldap"]
    when:
    - key: request.auth.authentications
      values: ["ldap-provider"]
```

同一工作负载可配置多个 CUSTOM 认证提供者，不同 API 路径使用不同认证方案（OAuth、LDAP、API Key）。

## 6. Helm v4 支持

```bash
# Istio 1.30 支持 Helm v4（server-side apply）
helm install istio-base manifests/charts/base --server-side
helm install istiod manifests/charts/istio-control/istio-discovery --server-side
# Webhook failurePolicy 字段所有权问题已修复，无需额外 workaround
```

## 7. 安全增强

| 项目 | 描述 |
|------|------|
| Debug 端点认证 | XDS debug 端点 (`syncz`, `config_dump`) 需认证（`ENABLE_DEBUG_ENDPOINT_AUTH=true`） |
| 允许命名空间 | `DEBUG_ENDPOINT_AUTH_ALLOWED_NAMESPACES` 可放行特定命名空间 |
| TLS 最低版本 | `pilot-discovery --tls-min-version` 提升控制面 TLS 安全基线 |
| WaypointBound 状态 | `WorkloadEntry` 新增 WaypointBound 状态条件 |

## 注意事项

1. **Agent Gateway 实验性**：仅作为 Gateway API Gateway 使用，不支持 Sidecar 或 Waypoint，存在粗糙边缘
2. **Helm v4 升级**：从 Helm v3 升级到 v4 需注意 CRD 管理方式差异
3. **TLSRoute 终止**：TLS 终止模式下 Gateway 需要配置证书，后端接收明文
4. **Debu 端点认证**：升级后 debug 端点默认需认证，可能影响现有运维脚本
5. **Ambient 迁移**：建议先在非关键命名空间试点，确认稳定后再推广
6. **Kubernetes 版本**：1.30 仅支持 K8s 1.32-1.36，旧集群需先升级 K8s
