---
name: onlyoffice-integration
description: OnlyOffice 文档在线编辑集成
tags: [onlyoffice, office, document]
---

## 集成要点
- Document Server 独立部署
- 回调（Callback）处理文档保存
- JWT 加密配置保证安全

## 配置
- 文档存储用 MinIO 或本地
- 支持格式：docx, xlsx, pptx
- 协同编辑需配置 WebSocket
