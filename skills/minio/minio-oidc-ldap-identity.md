---
name: minio-oidc-ldap-identity
description: MinIO 外部身份提供商（OIDC/LDAP）集成：Keycloak、Azure AD、OpenLDAP 配置与 STS 临时凭证
tags: [minio, oidc, ldap, identity, sso, keycloak, azure-ad, sts, iam]
---

## 概述

MinIO 支持集成外部身份提供商（IdP）实现**单点登录（SSO）** 和**集中式身份管理**，替代内置的静态访问密钥。支持两种主流方案：

| 方案 | 协议 | 适用场景 | 用户管理 |
|------|------|---------|---------|
| **OpenID Connect (OIDC)** | OAuth 2.0 + OIDC | 已有 Keycloak / Azure AD / Okta | IdP 管理 |
| **LDAP/AD** | LDAP | 企业 AD / OpenLDAP 环境 | IdP 管理 |

通过 IdP 集成，用户无需创建 MinIO 访问密钥即可通过 Web Console 或 `mc` 登录，并自动继承 IdP 中的角色/组映射。

## 一、OIDC 集成（Keycloak 示例）

### 1. Keycloak 配置

```bash
# 创建 Client
# Client ID: minio-console
# Client Protocol: openid-connect
# Access Type: confidential
# Valid Redirect URIs: https://minio-console.example.com/oauth/callback
# Web Origins: https://minio-console.example.com
```

### 2. MinIO 服务端配置

```bash
# 方式一：环境变量
export MINIO_IDENTITY_OPENID_CONFIG_URL="https://keycloak.example.com/realms/myrealm/.well-known/openid-configuration"
export MINIO_IDENTITY_OPENID_CLIENT_ID="minio-console"
export MINIO_IDENTITY_OPENID_CLIENT_SECRET="your-client-secret"
export MINIO_IDENTITY_OPENID_SCOPES="openid,profile,email,groups"
export MINIO_IDENTITY_OPENID_ROLE_POLICY="readonly"  # 默认策略

# 方式二：mc 配置
mc admin config set myminio identity_openid \
  config_url="https://keycloak.example.com/realms/myrealm/.well-known/openid-configuration" \
  client_id="minio-console" \
  client_secret="your-client-secret" \
  scopes="openid,profile,email,groups" \
  role_policy="consoleAdmin"
mc admin service restart myminio
```

### 3. 用户组到策略的映射

```bash
# 将 Keycloak 中的 "admin-group" 映射为 MinIO 的 "consoleAdmin" 策略
mc admin policy attach myminio consoleAdmin --group="admin-group"

# 将 "dev-group" 映射为自定义策略
mc admin policy create myminio dev-policy /path/to/dev-policy.json
mc admin policy attach myminio dev-policy --group="dev-group"
```

### 4. 自定义 Claim 映射（OIDC）

```json
// 在 MinIO 配置中，通过 claim_userinfo 控制是否从 UserInfo 端点获取额外属性
// 通过 claim_name / claim_groups 指定自定义字段名（非标准 OIDC 时）
export MINIO_IDENTITY_OPENID_CLAIM_USERINFO="on"
export MINIO_IDENTITY_OPENID_CLAIM_NAME="preferred_username"
export MINIO_IDENTITY_OPENID_CLAIM_GROUPS="cognito:groups"  // 非标准字段
```

## 二、LDAP 集成

### 1. OpenLDAP 配置示例

```bash
# MinIO 服务端配置
export MINIO_IDENTITY_LDAP_SERVER_ADDR="ldap.example.com:389"
export MINIO_IDENTITY_LDAP_SERVER_INSECURE="on"   # LDAP（非 LDAPS）
export MINIO_IDENTITY_LDAP_LOOKUP_BIND_DN="cn=admin,dc=example,dc=com"
export MINIO_IDENTITY_LDAP_LOOKUP_BIND_PASSWORD="admin-password"
export MINIO_IDENTITY_LDAP_USER_DN_SEARCH_BASE="dc=example,dc=com"
export MINIO_IDENTITY_LDAP_USER_DN_SEARCH_FILTER="(uid=%s)"
export MINIO_IDENTITY_LDAP_GROUP_SEARCH_BASE_DN="dc=example,dc=com"
export MINIO_IDENTITY_LDAP_GROUP_SEARCH_FILTER="(memberUid=%s)"
mc admin service restart myminio
```

### 2. AD（Active Directory）配置

```bash
export MINIO_IDENTITY_LDAP_SERVER_ADDR="ad.example.com:389"
export MINIO_IDENTITY_LDAP_LOOKUP_BIND_DN="CN=ServiceAccount,CN=Users,DC=example,DC=com"
export MINIO_IDENTITY_LDAP_USER_DN_SEARCH_BASE="DC=example,DC=com"
export MINIO_IDENTITY_LDAP_USER_DN_SEARCH_FILTER="(sAMAccountName=%s)"
export MINIO_IDENTITY_LDAP_GROUP_SEARCH_BASE_DN="DC=example,DC=com"
export MINIO_IDENTITY_LDAP_GROUP_SEARCH_FILTER="(member=%s)"
export MINIO_IDENTITY_LDAP_TLS_SKIP_VERIFY="on"     # 仅测试环境
```

### 3. LDAP 用户登录测试

```bash
# 使用 mc 以 LDAP 用户身份登录
mc alias set myminio-ldap https://minio.example.com --ldap-username "john" --ldap-password
Enter LDAP password: ********

# 验证身份
mc admin info myminio-ldap
```

## 三、STS 临时凭证（Assume Role）

MinIO 的 STS API 允许外部身份通过 AssumeRoleWithWebIdentity 获取临时访问凭证。

```bash
# 从 OIDC IdP 获取 ID Token
TOKEN=$(curl -s -X POST https://keycloak.example.com/realms/myrealm/protocol/openid-connect/token \
  -d "client_id=minio-console" \
  -d "client_secret=your-client-secret" \
  -d "grant_type=password" \
  -d "username=user" \
  -d "password=pass" | jq -r '.id_token')

# 通过 MinIO STS 换取临时 AK/SK
STS_CREDS=$(curl -s -X POST https://minio.example.com \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "Action=AssumeRoleWithWebIdentity" \
  -d "WebIdentityToken=$TOKEN" \
  -d "Version=2011-06-15" \
  -d "DurationSeconds=3600")

echo "$STS_CREDS" | grep -oP '(?<=<AccessKeyId>).*?(?=</AccessKeyId>)'
echo "$STS_CREDS" | grep -oP '(?<=<SecretAccessKey>).*?(?=</SecretAccessKey>)'
```

### Java SDK STS 示例

```java
import io.minio.S3Client;
import io.minio.credentials.WebIdentityProvider;
import io.minio.credentials.WebIdentityProviderConfig;

// 使用 OIDC ID Token 获取临时凭证
WebIdentityProvider provider = new WebIdentityProvider(
    WebIdentityProviderConfig.builder()
        .stsEndpoint("https://minio.example.com")
        .roleArn("arn:xxx:iam::123456789:role/admin-role")
        .webIdentityToken(tokenSupplier)
        .durationSeconds(3600)
        .build()
);

S3Client client = S3Client.builder()
    .endpoint("https://minio.example.com")
    .credentialsProvider(provider)
    .build();
```

## 四、混合模式：IdP + 本地用户

```bash
# MinIO 支持同时启用本地 IAM 和外部 IdP
# 本地用户使用常规 Access Key
# IdP 用户通过 SSO 登录

# 配置 IdP（不影响现有本地用户）
export MINIO_IDENTITY_OPENID_CONFIG_URL="..."
export MINIO_IDENTITY_LDAP_SERVER_ADDR="..."

# 本地用户继续使用 mc alias set
mc alias set myminio-local https://minio.example.com AKIAxxx SECRETxxx
```

## 五、生产配置要点

### 1. JWT 验证缓存

```bash
# OIDC JWT 公钥缓存时间（默认 15 分钟），降低对 IdP 的请求频率
export MINIO_IDENTITY_OPENID_PUBLIC_KEY_CACHE_TTL="15m"
```

### 2. 证书与 TLS

```bash
# LDAPS 使用系统 CA 证书或自定义 CA
export MINIO_IDENTITY_LDAP_SERVER_ADDR="ldaps://ldap.example.com:636"
export MINIO_IDENTITY_LDAP_TLS_SKIP_VERIFY="off"  # 生产不要跳过
export SSL_CERT_FILE="/etc/ssl/certs/ca-certificates.crt"
```

### 3. Session Policy 限制

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::my-bucket/*"]
    }
  ]
}
```

```bash
# 限制 IdP 用户的默认权限范围
export MINIO_IDENTITY_OPENID_ROLE_POLICY="restricted-user"
mc admin policy create myminio restricted-user restrict-policy.json
```

## 六、注意事项

1. **OIDC Role Policy 安全**：`role_policy` 是所有 OIDC 用户的默认权限，应设为最小权限策略。如需更细粒度，使用组映射。
2. **LDAP TLS**：生产环境必须使用 LDAPS（636端口），避免明文传输密码。`server_insecure=on` 仅用于开发测试。
3. **组同步延迟**：IdP 中的组变更不会立即反映到 MinIO，需等待 JWT Token 刷新（通常按 Token 有效期，默认 1 小时）。
4. **Service Account 与 IdP 共存**：IdP 用户创建的 Service Account 会继承父账号的 IdP 身份限制。
5. **CVE-2025-62506 影响**：启用了 IdP + Session Policy 的部署，需升级到 MinIO ≥ 2025-06-15T00-00-00Z 修复版本。
6. **OIDC 多个 IdP**：MinIO 支持同时配置多个 OIDC 提供者（不同 `role_policy` 和 `claim_groups`），按 `config_url` 区分。
