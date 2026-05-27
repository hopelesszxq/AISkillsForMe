---
name: minio-ops
description: MinIO 对象存储操作指南
tags: [minio, storage, s3]
---

## 核心概念
- Bucket（桶）隔离不同业务
- Presigned URL 用于临时上传/下载
- 生命周期管理：自动过期清理

## 集成
- Java: MinIO Java SDK / AWS S3 SDK
- 预签名 URL 有效时间按需设置
- 防盗链：Bucket Policy + Referer
