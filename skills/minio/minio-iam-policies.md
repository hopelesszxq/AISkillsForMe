---
name: minio-iam-policies
description: MinIO IAM 用户管理、Bucket Policy、Service Account 实战指南
tags: [minio, iam, policy, security, access-control]
---

## 概述

MinIO 提供完整的 IAM（Identity and Access Management）体系，兼容 AWS IAM 策略语法。合理配置 IAM 策略是实现最小权限原则、保障对象存储安全的关键。

## 核心概念

### 用户体系层级

```
MinIO 集群
 ├── 管理员用户 (root)
 ├── 普通用户 (User)
 │    └── 可关联多个策略
 ├── 组 (Group)
 │    └── 组成员继承组策略
 └── Service Account (服务账号)
      └── 关联到某个用户，继承用户权限 + 自定义策略
```

### 策略类型

| 类型 | 说明 | 适用范围 |
|------|------|----------|
| **User Policy** | 直接绑定用户 | 单个用户 |
| **Group Policy** | 绑定到组，所有成员继承 | 一组用户 |
| **Resource Policy (Bucket Policy)** | 绑定到 Bucket，匿名/公开访问 | 匿名/未认证用户 |
| **Service Account Policy** | 限制 Service Account 操作 | 单个服务账号 |

## 1. 策略语法（兼容 AWS IAM）

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::my-bucket/*", "arn:aws:s3:::my-bucket"]
    },
    {
      "Effect": "Deny",
      "Action": ["s3:DeleteObject"],
      "Resource": ["arn:aws:s3:::my-bucket/*"]
    }
  ]
}
```

### 常用 Action

| Action | 说明 |
|--------|------|
| `s3:GetObject` | 下载对象 |
| `s3:PutObject` | 上传对象 |
| `s3:DeleteObject` | 删除对象 |
| `s3:ListBucket` | 列出 Bucket 对象 |
| `s3:GetBucketPolicy` | 查看 Bucket 策略 |
| `s3:PutBucketPolicy` | 设置 Bucket 策略 |
| `s3:GetBucketEncryption` | 查看加密配置 |
| `s3:GetBucketVersioning` | 查看版本控制 |
| `s3:GetBucketLifecycleConfiguration` | 查看生命周期规则 |
| `s3:*` | 所有 S3 操作 |
| `admin:*` | MinIO 管理操作 |

### 条件（Condition）示例

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "StringEquals": {
      "s3:prefix": ["public/"]
    },
    "IpAddress": {
      "aws:SourceIp": ["192.168.1.0/24"]
    }
  }
}
```

## 2. 常用场景策略模板

### 只读用户（单个 Bucket）

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": ["arn:aws:s3:::app-files"]
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::app-files/*"]
    }
  ]
}
```

### 上传专用用户（只能写入不能读取）

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject"],
      "Resource": ["arn:aws:s3:::upload-bucket/*"]
    }
  ]
}
```

### 全功能 Bucket 管理员

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:*"],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    }
  ]
}
```

### 跨 Bucket 隔离（微服务场景）

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": ["arn:aws:s3:::service-a-*"]
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": ["arn:aws:s3:::service-a-uploads/*"]
    }
  ]
}
```

## 3. 使用 `mc` 管理 IAM

```bash
# --- 用户管理 ---

# 创建用户
mc admin user add myminio uploader mySecretKey123

# 列出用户
mc admin user list myminio

# 启用/禁用用户
mc admin user enable myminio uploader
mc admin user disable myminio uploader

# 删除用户
mc admin user remove myminio uploader

# --- 组管理 ---

# 创建组
mc admin group add myminio developers

# 添加组成员
mc admin group add myminio developers user1 user2 user3

# 列出组成员
mc admin group list myminio

# --- 策略管理 ---

# 创建策略（从 JSON 文件）
mc admin policy create myminio readonly-policy ./readonly-policy.json

# 列出所有策略
mc admin policy list myminio

# 绑定策略到用户
mc admin policy set myminio readonly-policy user=uploader

# 绑定策略到组
mc admin policy set myminio readonly-policy group=developers

# 解除策略
mc admin policy unset myminio readonly-policy user=uploader

# 查看用户权限
mc admin user info myminio uploader

# 删除策略
mc admin policy remove myminio readonly-policy
```

## 4. Service Account（服务账号）

Service Account 是生产环境的推荐方式，每个微服务应有独立的 Service Account。

```bash
# 创建 Service Account（附加到某个用户）
mc admin user svcacct add myminio uploader \
  --access-key "service-uploader-key" \
  --secret-key "service-uploader-secret" \
  --policy ./svc-policy.json

# 列出用户的 Service Account
mc admin user svcacct list myminio uploader

# 编辑 Service Account 策略
mc admin user svcacct edit myminio service-uploader-key \
  --policy ./new-policy.json

# 删除 Service Account
mc admin user svcacct remove myminio service-uploader-key

# 查看 Service Account 信息
mc admin user svcacct info myminio service-uploader-key
```

### Service Account 策略覆盖规则

```
创建时指定 policy → 使用该策略（优先）
创建时未指定 policy → 继承父用户权限
编辑时更新 policy → 新的策略完全覆盖旧策略（非合并）
```

## 5. Bucket Policy（公开/匿名访问）

```bash
# 创建公开只读的 Bucket 策略文件
cat > public-read-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": "*"},
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::public-assets/*"]
    }
  ]
}
EOF

# 应用 Bucket Policy
mc anonymous set-json public-read-policy.json myminio/public-assets

# 查看 Bucket 当前策略
mc anonymous get myminio/public-assets

# 取消 Bucket 策略
mc anonymous set myminio/public-assets

# 设置公开下载（无需 JSON）
mc anonymous set download myminio/public-assets

# 设置公开上传（用于文件收集场景）
mc anonymous set upload myminio/upload-bucket
```

## 6. Java SDK 编程管理 IAM

```xml
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>8.5.17</version>
</dependency>
```

```java
import io.minio.*;
import io.minio.admin.*;
import io.minio.messages.*;

MinioClient minioClient = MinioClient.builder()
    .endpoint("https://play.min.io")
    .credentials("admin", "password")
    .build();

MinioAdminClient adminClient = new MinioAdminClient(minioClient);

// 创建用户
CreateUserResponse userResp = adminClient.createUser(
    "app-user", "app-secret-key-123"
);

// 创建策略 JSON
String policyJson = """
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": ["arn:aws:s3:::my-bucket/*"]
  }]
}
""";

adminClient.addPolicy("app-policy", policyJson);
adminClient.setPolicy("app-policy", false, "app-user", false);

// 创建 Service Account
CreateServiceAccountResponse svcResp = adminClient.createServiceAccount(
    "app-user",
    "svc-app-key",
    "svc-app-secret",
    policyJson  // 限制 Service Account 权限
);

// 列出用户的数据
List<String> users = adminClient.listUsers();
for (String user : users) {
    UserInfo userInfo = adminClient.getUserInfo(user);
    System.out.println("User: " + user + " Status: " + userInfo.status());
}
```

## 7. 用 Spring Boot 集成 IAM 策略验证

```java
@Service
public class MinioAuthorizationService {

    private final MinioClient minioClient;

    // 验证用户是否有权访问某对象
    public boolean canAccessObject(String accessKey, String bucket, String object, String action) {
        try {
            // 通过尝试执行操作来验证权限
            switch (action) {
                case "READ":
                    minioClient.statObject(
                        StatObjectArgs.builder()
                            .bucket(bucket)
                            .object(object)
                            .build()
                    );
                    return true;
                case "WRITE":
                    // 无需实际写入，检查 PutObject 权限较复杂
                    // 建议预创建 Service Account 时配置好策略
                    return true;
                default:
                    return false;
            }
        } catch (Exception e) {
            return false;
        }
    }
}
```

## 注意事项

1. **策略生效延迟**：MinIO IAM 策略更新后可能有短暂延迟（通常 < 1s），不能依赖即时生效
2. **策略大小限制**：单个策略 JSON 不超过 10KB，超出则拒绝保存
3. **Service Account 数量限制**：默认每个用户最多 100 个 Service Account，可通过 `_MINIO_SVC_ACCT_LIMIT` 调整
4. **Root 用户限制**：生产环境不应使用 Root 用户操作，创建不同权限的普通用户代替
5. **LDAP/SSO 集成**：企业环境可配置 OpenID Connect 或 LDAP 进行外部身份认证
6. **策略非继承**：Service Account 自带的策略会覆盖父用户的权限（非合并），需要显式列出所有允许的操作
7. **Bucket Policy 优先级**：匿名策略 < 用户策略 < Service Account 策略，Deny 始终优先于 Allow
8. **审计日志**：结合 MinIO 审计日志（console/HTTP 审计），可追踪每次操作的用户身份
