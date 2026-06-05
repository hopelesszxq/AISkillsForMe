---
name: spring-boot-http-smuggling-splitting
description: Spring Boot HTTP Request Smuggling 与 Request Splitting 攻击原理、区别与防御方案
tags: [tools, spring-boot, security, tomcat, http, web-security]
---

## 概述

Request Smuggling（请求走私）和 Request Splitting（请求分割/CRLF 注入）是两种常见但容易被混淆的 HTTP 攻击。两者都涉及 CRLF，都滥用 HTTP 解析机制，但攻击面、前置条件和防御方式完全不同。

| 维度 | Request Smuggling | Request Splitting |
|------|-------------------|-------------------|
| 攻击面 | 代理与后端之间的共享连接 | 任何由用户输入构建的响应头或转发请求 |
| 前置条件 | 代理和后端对分帧头不一致 | 应用将未清洗的用户输入写入响应头 |
| 协议层 | 传输/分帧（HTTP/1.1 连接语义） | 应用层（头部构建） |
| 典型影响 | 缓存投毒、请求队列失同步、越权、反射 XSS | 头部注入、伪造重定向、会话固定、响应分割 |
| 检测难度 | 高；需了解代理/后端拓扑 | 较低；直接 grep 未清洗的 header 写入 |
| 影响 HTTP 版本 | 主要是 HTTP/1.1；HTTP/2 端到端不适用 | HTTP/1.x 和 HTTP/2（头部值仍有注入风险） |
| Spring Boot 防御 | 升级 Tomcat、强制严格分帧、使用 HTTP/2 | 清洗输入、使用白名单、依靠容器级头部编码 |

## Request Smuggling 详解

### 原理

利用反向代理（nginx、HAProxy、ALB）和后端（Tomcat）对 HTTP/1.1 请求边界判断不一致的漏洞。代理用 `Content-Length` 判断请求结束位置，后端用 `Transfer-Encoding: chunked`，导致代理认为属于第一个请求的字节，被后端当成第二个请求处理（CL.TE 变种）。

### CL.TE 攻击示例

```http
POST / HTTP/1.1
Host: internal.example.com
Content-Length: 13
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
Host: internal.example.com
```

### TE.CL 变种

后端读 `Content-Length`，代理读 `Transfer-Encoding: chunked` — 角色互换。

## Request Splitting（CRLF 注入）详解

### 原理

应用层漏洞。应用将用户可控的值直接写入 HTTP 响应头或转发的请求中，未剥离 CR/LF 字符。注入的 `\r\n` 提前结束当前头部，攻击者可伪造新头部。

### 不安全的控制器示例

```java
@GetMapping("/redirect")
public void redirect(
        @RequestParam String redirectUrl,
        HttpServletResponse response) throws IOException {
    // 不安全！用户输入直接传给 sendRedirect
    // 攻击者提供: redirectUrl = "https://safe.example.com\r\nSet-Cookie: session=attacker"
    response.sendRedirect(redirectUrl);
}
```

## Spring Boot 防御方案

### 1. 修复 CRLF / Request Splitting

在用户可控值到达响应头之前必须剥离或拒绝 CR 和 LF：

```java
import org.springframework.util.StringUtils;
import jakarta.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.net.URI;
import java.net.URISyntaxException;

@GetMapping("/redirect")
public void safeRedirect(
        @RequestParam String redirectUrl,
        HttpServletResponse response) throws IOException {

    String sanitized = sanitizeHeaderValue(redirectUrl);

    // 白名单检查：只允许重定向到已知安全来源
    if (!isAllowedRedirectTarget(sanitized)) {
        response.sendError(HttpServletResponse.SC_BAD_REQUEST, "Invalid redirect target");
        return;
    }
    response.sendRedirect(sanitized);
}

private String sanitizeHeaderValue(String value) {
    if (value == null) return "";
    // 剥离 CR、LF 和空字节
    return value.replaceAll("[\\r\\n\\x00]", "");
}

private boolean isAllowedRedirectTarget(String url) {
    try {
        URI uri = new URI(url);
        String host = uri.getHost();
        return host != null && host.endsWith(".example.com");
    } catch (URISyntaxException e) {
        return false;
    }
}
```

### 2. 修复 Smuggling — Tomcat 配置

```properties
# application.properties
# 拒绝有冲突分帧头的请求
server.tomcat.reject-illegal-header=true

# 连接超时设置
server.tomcat.connection-timeout=20000
```

### 3. nginx 代理加固

```nginx
# 强制 HTTP/1.1 + 清理连接
proxy_http_version 1.1;
proxy_set_header Connection "";

# 或者端到端使用 HTTP/2 彻底消除分帧歧义
```

## 代码审查检测

### 需要关注的高危模式

```java
response.addHeader(name, <tainted>)
response.setHeader(name, <tainted>)
response.sendRedirect(<tainted>)
httpHeaders.set(name, <tainted>)           // Spring's HttpHeaders
restTemplate.exchange 使用用户输入的 UriComponentsBuilder
```

### Semgrep 检测规则

```yaml
rules:
  - id: crlf-injection-spring-response
    patterns:
      - pattern: |
          $RESP.sendRedirect($URL);
      - pattern-not: |
          $RESP.sendRedirect(sanitize($URL));
      - pattern-not: |
          $RESP.sendRedirect($CONSTANT);
    message: >
      sendRedirect 用可能受污染的参数调用 — 可能 CRLF 注入。
      请先清洗 $URL。
    languages: [java]
    severity: ERROR
    metadata:
      cwe: "CWE-113"
```

## 测试复现

### 复现 CRLF 分割

```bash
# %0d%0a 是 URL 编码的 CRLF
curl -v "http://localhost:8080/redirect?redirectUrl=https%3A%2F%2Fsafe.example.com%0d%0aSet-Cookie%3A%20session%3Dattacker_controlled"

# 响应头中应看到:
# Location: https://safe.example.com
# Set-Cookie: session=attacker_controlled
```

### 复现 CL.TE Smuggling

```bash
# 需要 nginx 1.19（宽松分帧）+ Spring Boot 2.5 的测试环境
curl -v --http1.1 -s \
  -H "Host: localhost" \
  -H "Transfer-Encoding: chunked" \
  -H "Content-Length: 4" \
  --data-binary $'1\r\nZ\r\n0\r\n\r\nGET /admin HTTP/1.1\r\nHost: localhost\r\n\r\n' \
  http://localhost:80/api/data
```

## 常见误解

| 误解 | 真相 |
|------|------|
| 两种都是 CRLF 注入 | Splitting 是 CRLF 注入。Smuggling 利用分帧不一致而非字面 CRLF 注入 |
| Tomcat 9+ 同时解决两者 | 解决了 Tomcat 侧的问题，但如果 nginx/HAProxy 在转发前规范化了头部，后端永远看不到歧义 |
| HTTP/2 完全消灭 Smuggling | 仅客户端到代理是 HTTP/2 不够，代理到后端通常降级为 HTTP/1.1 |
| Splitting 已死 | 现代容器拒绝 `sendRedirect` 中的 CRLF，但 Spring 的 `HttpHeaders.set()`、`RestTemplate` 转发不走相同校验路径 |
| 仅影响公网服务 | 内网服务在共享代理后的 Smuggling 危害更大（内网管理 API 认证较弱） |

## 团队检查清单

- **确认嵌入式服务器版本**：Tomcat 9.0.31+（Spring Boot 2.x）或 10.1+（Boot 3.x）。检查 `mvn dependency:tree | grep tomcat-embed-core`
- **拒绝歧义分帧**：设 `server.tomcat.reject-illegal-header=true`，发双头请求确认返回 400
- **代理配置审计**：审查 nginx/HAProxy/ALB 的 `proxy_http_version`、头部规范化设置
- **HTTP/2 端到端**：条件允许时，将 HTTP/2 终止到 Tomcat 层面
- **响应头值清洗**：添加共用工具，剥离 `\r`、`\n`、`\x00`
- **重定向目标白名单**：用户提供的重定向 URL 必须通过白名单验证
- **头部转发审查**：审计所有将传入请求头部转发到上游的 `RestTemplate`/`WebClient` 用法，建立显式白名单
- **回归测试**：添加发送 `%0d%0a` 在重定向参数中的集成测试，确认为 400 响应
