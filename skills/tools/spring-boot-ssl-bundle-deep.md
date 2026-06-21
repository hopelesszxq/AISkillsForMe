---
name: spring-boot-ssl-bundle-deep
description: Spring Boot 4.x SSL Bundle 深度实践：Let's Encrypt 自动续签与重载、mTLS 双向认证、多证书配置、生产安全加固
tags: [tools, spring-boot, ssl, tls, security, lets-encrypt, mtls]
---

## 概述

Spring Boot 4.x（基于 Spring Framework 7.x）通过 **SSL Bundle** 统一了所有网络组件的证书管理。相比 Boot 3.x 分散的配置，SSL Bundle 支持：**证书自动重载**（Let's Encrypt 场景）、**多证书动态切换**、**mTLS 双向认证**、**PEM/JKS/PKCS12 混合存储**。

## 一、SSL Bundle 基础配置

### 1.1 PEM 格式证书

```yaml
# application.yml
spring:
  ssl:
    bundle:
      pem:
        my-service:
          keystore:
            certificate: "file:/etc/certs/live/example.com/fullchain.pem"
            private-key: "file:/etc/certs/live/example.com/privkey.pem"
          truststore:
            certificate: "file:/etc/certs/ca.crt"
```

### 1.2 JKS/PKCS12 密钥库

```yaml
spring:
  ssl:
    bundle:
      jks:
        my-service:
          keystore:
            location: "file:/etc/certs/keystore.jks"
            password: ${KEYSTORE_PASSWORD}
            type: JKS    # 或 PKCS12
          truststore:
            location: "file:/etc/certs/truststore.jks"
            password: ${TRUSTSTORE_PASSWORD}
```

### 1.3 在连接器中引用

```yaml
server:
  ssl:
    bundle: my-service
  # Tomcat 原生配置会被 SSL Bundle 覆盖
  port: 8443
```

## 二、Let's Encrypt 自动续签与证书重载

### 2.1 原理

Spring Boot 4.x 的 SSL Bundle 支持 **热重载（hot-reload）**——证书文件变化时自动重新加载，无需重启 JVM。配合 Let's Encrypt 的 `certbot renew` 实现零停机自动续签。

### 2.2 配置

```yaml
spring:
  ssl:
    bundle:
      pem:
        letsencrypt:
          keystore:
            certificate: "file:/etc/letsencrypt/live/example.com/fullchain.pem"
            private-key: "file:/etc/letsencrypt/live/example.com/privkey.pem"
          # ⚠️ certbot 更新后文件会原子替换，Bundle 自动检测变化
          reload-on-file-change: true    # 文件变化自动重载（默认 true）
          reload-interval: 30s           # 轮询检查间隔（默认 30s）
```

### 2.3 Certbot 自动化流程

```bash
# 1. 安装 certbot 并获取证书
certbot certonly --webroot -w /var/www/html -d example.com

# 2. 设置自动续签
# crontab 中配置：
# 0 3 * * * certbot renew --quiet --post-hook "touch /etc/letsencrypt/live/example.com/fullchain.pem"
#                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
#                  touch 触发文件 mtime 变化，Spring 检测到后自动重载（无需重启进程）

# 3. 验证自动重载
# 查看日志：
# 2026-06-21 03:05:12.123  INFO 12345 --- [nt-loop-1] o.s.b.s.b.SslBundleFileWatcher
#   : Detected change in SSL bundle 'letsencrypt', reloading...
```

### 2.4 编程式重载

```java
import org.springframework.web.bind.annotation.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.ssl.SslBundles;

@RestController
public class SslReloadController {

    @Autowired
    private SslBundles sslBundles;

    @PostMapping("/admin/ssl/reload")
    public String reloadSsl(@RequestParam String bundleName) {
        // 手动触发 SSL Bundle 重载
        sslBundles.reload(bundleName);
        return "SSL bundle '" + bundleName + "' reloaded";
    }

    @GetMapping("/admin/ssl/status")
    public String sslStatus(@RequestParam String bundleName) {
        var bundle = sslBundles.getBundle(bundleName);
        return "Cert expiry: " + bundle.getManagers()
            .getKeyManager().getCertificateChain(null)[0]
            .getCertificate().toString();
    }
}
```

## 三、mTLS 双向认证

### 3.1 服务端配置（验证客户端证书）

```yaml
spring:
  ssl:
    bundle:
      pem:
        server-mtls:
          keystore:
            certificate: "file:/etc/certs/server.crt"
            private-key: "file:/etc/certs/server.key"
          truststore:
            certificate: "file:/etc/certs/ca.crt"  # 用于验证客户端证书
      # mTLS 需要设置 client-auth
server:
  ssl:
    bundle: server-mtls
    client-auth: need          # need = 强制要求客户端证书（mutual TLS）
                               # want = 可选客户端证书
```

### 3.2 客户端配置（提供客户端证书）

```yaml
spring:
  ssl:
    bundle:
      pem:
        client-cert:
          keystore:
            certificate: "file:/etc/certs/client.crt"
            private-key: "file:/etc/certs/client.key"
          truststore:
            certificate: "file:/etc/certs/ca.crt"

# RestClient 使用 mTLS
spring:
  web:
    client:
      ssl:
        bundle: client-cert
```

### 3.3 多个 RestClient 不同证书

```java
@Configuration
public class ClientConfig {

    @Bean
    @Qualifier("internalClient")
    RestClient internalClient(SslBundles sslBundles) {
        return RestClient.builder()
            .sslBundle(sslBundles.getBundle("internal-cert"))
            .baseUrl("https://internal.service:8443")
            .build();
    }

    @Bean
    @Qualifier("externalClient")
    RestClient externalClient(SslBundles sslBundles) {
        return RestClient.builder()
            .sslBundle(sslBundles.getBundle("letsencrypt"))
            .baseUrl("https://api.example.com")
            .build();
    }
}
```

## 四、生产安全加固

### 4.1 禁用弱加密套件

```yaml
spring:
  ssl:
    bundle:
      pem:
        hardened:
          keystore:
            certificate: "file:/etc/certs/server.crt"
            private-key: "file:/etc/certs/server.key"
      # 全局 TLS 配置
  server:
    tomcat:
      ssl:
        enabled-protocols: TLSv1.3       # 仅 TLS 1.3
        ciphers: TLS_AES_256_GCM_SHA384  # 仅指定强加密套件
```

### 4.2 SSL Bundle 监控（Actuator）

```yaml
# 暴露 SSL 端点
management:
  endpoints:
    web:
      exposure:
        include: ssl
  endpoint:
    ssl:
      enabled: true

# 访问：GET /actuator/ssl
# 响应示例：
# {
#   "bundles": {
#     "letsencrypt": {
#       "keystoreType": "PEM",
#       "keyManagerAlgorithm": "PKIX",
#       "trustManagerAlgorithm": "PKIX",
#       "certificates": [
#         {
#           "subject": "CN=example.com",
#           "issuer": "CN=R3,O=Let's Encrypt,C=US",
#           "notBefore": "2026-03-15T00:00:00Z",
#           "notAfter": "2026-06-13T00:00:00Z",  # <-- 注意过期时间
#           "serialNumber": "04:AB:CD:..."
#         }
#       ]
#     }
#   }
# }
```

### 4.3 证书过期告警（Micrometer + Prometheus）

```java
import io.micrometer.core.instrument.MeterRegistry;
import jakarta.annotation.PostConstruct;
import org.springframework.boot.ssl.SslBundles;
import org.springframework.stereotype.Component;

import java.security.cert.X509Certificate;
import java.time.Duration;
import java.time.Instant;

@Component
public class SslExpiryMetrics {

    private final SslBundles sslBundles;
    private final MeterRegistry meterRegistry;

    public SslExpiryMetrics(SslBundles sslBundles, MeterRegistry meterRegistry) {
        this.sslBundles = sslBundles;
        this.meterRegistry = meterRegistry;
    }

    @PostConstruct
    public void registerMetrics() {
        for (var bundleName : sslBundles.getBundleNames()) {
            var bundle = sslBundles.getBundle(bundleName);
            try {
                var cert = (X509Certificate) bundle.getManagers()
                    .getKeyManager().getCertificateChain(null)[0]
                    .getCertificate();

                Duration remaining = Duration.between(
                    Instant.now(), cert.getNotAfter().toInstant()
                );

                meterRegistry.gauge(
                    "ssl.cert.days_remaining",
                    java.util.List.of(
                        io.micrometer.core.instrument.Tag.of("bundle", bundleName),
                        io.micrometer.core.instrument.Tag.of("cn", 
                            cert.getSubjectX500Principal().getName())
                    ),
                    remaining.toDays()
                );
            } catch (Exception e) {
                // bundle 可能没有密钥对（仅 truststore）
            }
        }
    }
}
```

## 注意事项

### ⚠️ 1. 文件权限安全
```bash
# 证书私钥文件权限必须严格
chmod 600 /etc/certs/*.key
chmod 644 /etc/certs/*.crt
# 运行 Spring Boot 的用户必须是文件 owner 或在同一 group
```

### ⚠️ 2. 不要硬编码密码
```yaml
# ❌ 不安全
password: changeit
# ✅ 使用环境变量或 Vault
password: ${KEYSTORE_PASSWORD}
```

### ⚠️ 3. Let's Encrypt 证书有效期
- Let's Encrypt 证书有效期 90 天
- 必须在到期前完成续签（certbot 通常提前 30 天自动续签）
- 监控 `ssl.cert.days_remaining` 指标，`< 14` 天报警

### ⚠️ 4. Spring Boot 4.x 与 Boot 3.x 差异
| 能力 | Spring Boot 3.x | Spring Boot 4.x |
|------|----------------|----------------|
| SSL Bundle 概念 | 不支持 | 原生支持 |
| 证书热重载 | 需自定义 Watcher | 内置 `reload-on-file-change` |
| 多证书管理 | 手动切换 | 多个 Bundle + Qualifier |
| Actuator SSL 端点 | 无 | `/actuator/ssl` |
| PEM 支持 | 有限 | 完整支持 |

### ⚠️ 5. 性能考量
- SSL 热重载是轻量操作（仅替换 KeyManager/TrustManager），不影响现有连接
- 新连接使用新证书，旧连接继续使用旧证书直到断开
- Let's Encrypt 自动续签建议避开业务高峰期
