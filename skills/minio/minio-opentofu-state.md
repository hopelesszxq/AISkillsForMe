---
name: minio-opentofu-state-backend
description: 使用 MinIO 作为 OpenTofu/Terraform S3 后端存储 IaC 状态文件
tags: [minio, opentofu, terraform, s3-backend, iac, state-management]
---

## 概述

使用 MinIO 作为 OpenTofu/Terraform 的 S3 后端，将 `.tfstate` 存储在自托管的对象存储中，无需 Terraform Cloud 或 AWS S3。关键挑战：S3 兼容性在实际使用中存在不少"坑"。

## 后端配置（坑点详解）

### 错误示范（不要这样写）

```hcl
terraform {
  backend "s3" {
    bucket   = "opentofu-state"
    key      = "proxmox/terraform.tfstate"
    region   = "us-east-1"
    endpoint = "https://minio.example.com"
  }
}
```

**问题**：AWS SDK 会尝试访问 `opentofu-state.minio.example.com`（虚拟主机风格），MinIO 不支持这种 DNS 解析，导致 TLS/DNS 错误。

### 正确配置

```hcl
terraform {
  backend "s3" {
    bucket = "opentofu-state"
    key    = "proxmox/terraform.tfstate"
    region = "us-east-1"  # 必须填写，SDK 需要

    endpoints = {
      s3 = "https://minio.example.com"
    }

    use_path_style              = true  # 关键：路径风格，而非主机名风格
    use_lockfile                = true  # S3 原生锁定，无需 DynamoDB
    skip_credentials_validation = true  # 跳过 AWS 凭证校验
    skip_requesting_account_id  = true  # 跳过 AWS 账户 ID 查询
    skip_metadata_api_check     = true  # 跳过 AWS 元数据服务
    skip_region_validation      = true  # 跳过区域校验
  }
}
```

## 配置项演变（版本兼容性注意）

| 旧版属性 | 新版属性 | 说明 |
|---------|---------|------|
| `endpoint` | `endpoints.s3` | OpenTofu 1.10+ / Terraform 1.10+ |
| `force_path_style` | `use_path_style` | 重命名 |
| DynamoDB 表锁定 | `use_lockfile = true` | 新增，无需额外数据库 |

> ⚠️ 检查你所用的 OpenTofu/Terraform 版本文档，键名在不同的版本中可能不同。

## 凭证管理

凭证不能写在 `.tf` 文件中。通过环境变量注入：

```bash
# 使用 1Password CLI 或其他密钥管理器
export AWS_ACCESS_KEY_ID="$(op read 'op://Infra/minio-opentofu/access-key')"
export AWS_SECRET_ACCESS_KEY="$(op read 'op://Infra/minio-opentofu/secret-key')"
tofu init
```

### MinIO 最小权限策略

创建专用服务账号，权限范围仅限于目标桶：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket", "s3:GetBucketLocation"],
      "Resource": ["arn:aws:s3:::opentofu-state"]
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": ["arn:aws:s3:::opentofu-state/*"]
    }
  ]
}
```

> `DeleteObject` 是必需的——锁定文件在操作前后会写入和删除。没有 delete 权限会导致无法释放锁。

```bash
mc admin policy create local opentofu-state ./opentofu-state-policy.json
mc admin user add local opentofu-ci "$(op read 'op://Infra/minio-opentofu/secret-key')"
mc admin policy attach local opentofu-state --user opentofu-ci
```

## CI/CD 集成（GitHub Actions）

```yaml
- name: OpenTofu Init
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.MINIO_ACCESS_KEY }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.MINIO_SECRET_KEY }}
  run: tofu init

- name: OpenTofu Plan
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.MINIO_ACCESS_KEY }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.MINIO_SECRET_KEY }}
  run: tofu plan -detailed-exitcode -out=tfplan
```

> `-detailed-exitcode`：0=无变更，2=有变更需审批，1=错误——让自动 apply 需要人工审批。

## 防止意外销毁

在管理真实机器（如 Proxmox）时，自动 apply 的风险很高。添加生命周期保护：

```hcl
resource "proxmox_virtual_environment_container" "app" {
  # ... 容器配置 ...

  lifecycle {
    prevent_destroy = true
  }
}
```

这样任何涉及删除/重建该资源的 plan 都会在 plan 阶段报错，不会实际执行。

## 为什么这样工作

### 路径风格 vs 主机名风格

- **虚拟主机风格**（AWS 默认）：`bucket.s3.amazonaws.com/key`
- **路径风格**（MinIO 默认）：`minio.example.com/bucket/key`
- `use_path_style = true` 告诉 SDK 使用路径风格，避免 MinIO 不支持的 DNS 解析

### S3 原生锁定

- OpenTofu 1.10+ 的 `use_lockfile = true` 使用 S3 条件写入（`PutObject` with `If-None-Match`）实现原子锁
- MinIO 支持条件写入，所以无需额外数据库
- 同时运行的第二个 apply 会因锁冲突被拒绝

## 注意事项

1. **版本兼容性**：不同版本的 OpenTofu/Terraform 后端配置键名不同，请查阅你的版本文档
2. **凭证最小权限**：MinIO 密钥应仅限一个桶，并使用密钥管理器注入环境变量
3. **`prevent_destroy` 的局限性**：如果从配置中删除了资源块，`prevent_destroy` 也会消失——删除资源配置要像执行销毁一样谨慎审查
4. **开启桶版本控制**：MinIO 支持桶版本控制，可追溯状态文件历史
5. **网络安全**：如果 CI Runner 无法直接访问 MinIO，可配合 Tailscale 等 VPN 方案
