---
name: nuxt-security-hardening
description: Nuxt 安全防护最佳实践——路径遍历、XSS、SSRF、开放重定向防御（含 v4.4.7 热修复要点）
tags: [vue3, nuxt, security, ssr, xss, ssrf]
---

## 概述

Nuxt v4.4.7（2026-06-04）是一次**安全热修复**版本，修补了 20+ 安全漏洞，涵盖路径遍历、XSS 注入、SSRF 和开放重定向等攻击面。

> 以下模式适用于所有 Nuxt 4.x 版本，部分配置在 4.4.7 中成为默认行为。

## 一、路径遍历防护（Path Traversal）

### 问题场景

攻击者可通过构造特殊路径访问服务器上的敏感文件。

```text
# 攻击示例
/reloadNuxtApp?path=../../../etc/passwd
/navigateTo?path=//evil.com/steal
```

### 防护措施（4.4.7 已内置）

```typescript
// nuxt.config.ts - 额外路径验证
export default defineNuxtConfig({
  nitro: {
    // 限制可访问的文件系统路径
    serveStatic: true,
    // 4.4.7+ 自动验证协议，但建议额外配置
  },
  vite: {
    // 限制 Vite Dev 允许的目录
    devBundler: {
      // 4.4.7: 避免过滤共享前缀的目录
      allowDirs: ['src', 'server']
    }
  }
})
```

### 自定义路由守卫

```typescript
// middleware/security.ts
export default defineNuxtRouteMiddleware((to) => {
  // 检查 navigateTo 的目标路径
  if (to.path && to.path.includes('..')) {
    return abortNavigation('Invalid path')
  }

  // 阻止 script: 协议跳转（v4.4.7 已修复 NuxtLink）
  if (to.path && /^javascript:|^data:|^vbscript:/i.test(to.path)) {
    return abortNavigation('Blocked protocol')
  }
})
```

## 二、XSS 防护（跨站脚本）

### 服务端渲染（SSR）输出转义

```vue
<!-- 危险写法：直接渲染用户输入 -->
<span v-html="userInput" />

<!-- 安全写法：默认转义 -->
<span>{{ userInput }}</span>

<!-- 必须用 HTML 时，使用 sanitize -->
<span v-html="sanitizeHtml(userInput)" />
```

### NuxtLink 安全配置

```typescript
// vue 组件中的 NuxtLink
// v4.4.7 修复：拒绝 script-capable 协议的 href
<NuxtLink :to="userProvidedUrl">链接</NuxtLink>

// ✅ 安全：自动验证协议
// ❌ 不安全：直接使用 dangerously 选项
```

### NoScript 插槽转义

```vue
<!-- v4.4.7 修复：NoScript 插槽内容逃逸 -->
<!-- 之前：<Noscript><script>alert(1)</script></Noscript> 可能绕过 -->
<Noscript>
  <!-- 4.4.7+ 自动转义插槽内容 -->
</Noscript>
```

## 三、SSRF 与开放重定向防护

### 配置限制

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    // 限制 DevTools 端点仅限本地请求（v4.4.7 新增）
    devTools: {
      // 默认 gate to localhost
    }
  },
  runtimeConfig: {
    // 对外部 API 调用添加白名单
    allowedOrigins: ['https://api.example.com']
  }
})
```

### 安全 reloadNuxtApp

```typescript
// v4.4.7 修复：reloadNuxtApp 路径验证
// ✅ 安全用法
await reloadNuxtApp({ path: '/dashboard' })

// ❌ 会被拦截的用法
await reloadNuxtApp({ path: '//attacker.com' })
await reloadNuxtApp({ path: '../../../etc/passwd' })
await reloadNuxtApp({ path: 'javascript:alert(1)' })
```

## 四、构建安全配置清单

```typescript
// nuxt.config.ts - 完整安全配置
export default defineNuxtConfig({
  // 1. CSP 头
  nitro: {
    routeRules: {
      '/**': {
        headers: {
          'Content-Security-Policy':
            "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'"
        }
      }
    }
  },

  // 2. 避免依赖注入（v4.4.7 修复 island registry 原型链注入）
  experimental: {
    // 确保不启用危险实验性功能
  },

  // 3. 安全响应头
  security: {
    // 使用 @nuxt/security 模块（推荐）
    headers: {
      xFrameOptions: 'SAMEORIGIN',
      xXSSProtection: '1; mode=block',
      strictTransportSecurity: 'max-age=31536000'
    }
  }
})
```

## 五、安全依赖管理

```bash
# 1. 锁定 Nuxt 版本到安全版本
npm install nuxt@4.4.7

# 2. 使用 npm audit 检查依赖
npm audit --production

# 3. 更新 lockfile
npm update
```

## 六、v4.4.7 关键修复汇总

| 修复类型 | 组件 | 描述 |
|---------|------|------|
| 路径遍历 | `reloadNuxtApp` | 验证 reload 路径协议 |
| 路径遍历 | `navigateTo` | 阻止路径标准化开放重定向 |
| 路径遍历 | `NuxtLink` | 拒绝 script-capable 协议 |
| XSS | `NuxtClientFallback` | SSR 输出属性转义 |
| XSS | `NoScript` | 插槽内容转义 |
| 原型链注入 | Island Registry | 拒绝 prototype-chain 键 |
| SSRF | Vite IPC | 绑定到权限化文件系统 socket |
| 开放重定向 | route rules | 不区分大小写匹配 |
| 目录遍历 | kit/tsconfig | 排除 node-context 文件 |

## 注意事项

- **升级到 4.4.7 后建议全量回归测试**，部分安全修复涉及 SSR 渲染逻辑变更
- **不要关闭默认安全行为**：v4.4.7 默认要求 localhost 访问 DevTools 端点
- **@nuxt/security 模块**是 Nuxt 官方安全方案，建议与内置防护配合使用
- **layers 和 modules** 中的安全边界同样重要，确保所有依赖模块也升级到兼容版本
