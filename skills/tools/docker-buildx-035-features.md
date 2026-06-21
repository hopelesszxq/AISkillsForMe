---
name: docker-buildx-035-features
description: Docker BuildX v0.35.0 新特性：local output mode=delete、exec proxy source policy、BuildKit 0.31.0
tags: [tools, docker, buildx, buildkit, source-policy, security]
---

## 概述

Docker BuildX v0.35.0（2026-06-17 发布）在 v0.34 的基础上引入了**本地输出 mode=delete**、**exec proxy 源策略支持**以及 **BuildKit v0.31.0** 集成。需要 BuildKit v0.31.0+ 才能使用完整新特性。

## 1. local output mode=delete

新增 `mode=delete` 属性，构建结果会**替换**目标目录而不是合并到其中——与 rsync 的 `--delete` 标志类似。

```bash
# 不指定 mode（默认 merge）：构建产物与目录内容合并
docker buildx build --output type=local,dest=./dist .

# mode=delete：先清空目标目录再写入构建结果
docker buildx build --output type=local,mode=delete,dest=./dist .
```

### 安全限制

- 只允许在工作目录的子目录中使用（防止误删系统目录）
- 导出到其他目标目录时需显式授权：`--allow=buildx.local.delete`
- 多平台构建结果需要 BuildKit v0.31.0+

```bash
# 导出到工作目录之外的路径需要授权
docker buildx build \
  --allow=buildx.local.delete \
  --output type=local,mode=delete,dest=/tmp/artifacts .
```

**适用场景**：CI/CD 中每次构建前需要干净的输出目录，无需手动 `rm -rf`。

## 2. Exec Proxy 源策略

Source Policy 现在支持 BuildKit v0.31.0+ 的 **exec proxy** 功能，可捕获构建步骤的网络流量并在策略层面进行管控。

```rego
# Dockerfile.rego - 允许特定请求，阻止其余
package source.policy

import future.keywords.if

default allow = false

# 允许从官方源拉取
allow if {
  input.caps["exec.proxy.network"] == true
  input.request.url == "https://registry.docker.io/v2/"
}

# 阻止指向内部网络的请求
deny["blocked internal request"] {
  input.caps["exec.proxy.network"] == true
  startswith(input.request.url, "http://10.")
}
```

```bash
# 构建时启用 exec proxy
docker buildx build \
  --source-policy Dockerfile.rego \
  --allow=buildx.source-policy.exec-proxy \
  .
```

**用途**：构建过程中自动审计依赖下载、阻止已知恶意源、记录所有外部请求。

## 3. BuildKit v0.31.0 集成

BuildX v0.35.0 要求 BuildKit v0.31.0+ 以支持 exec proxy 和 mode=delete 的完整功能。详见 [`docker-296-features.md`](docker-296-features.md) 中 BuildKit v0.31.0 部分。

## 4. 其他改进

- 构建器实例支持 `--driver` 参数默认值改进
- 多平台构建的稳定性修复

## 注意事项

1. **mode=delete 有数据安全风险**：确保目标目录不含重要数据，或在 CI 临时目录中使用。
2. Exec Proxy 启用后所有容器流量都会被代理，可能影响构建性能（尤其是大量 `apt-get` 或 `npm install` 的场景）。
3. Source Policy 需要编写 [Rego](https://www.openpolicyagent.org/docs/latest/policy-language/) 策略文件，学习曲线存在。
4. v0.35.0 与旧版 BuildKit 后端不兼容部分新特性，建议同步升级 BuildKit。
