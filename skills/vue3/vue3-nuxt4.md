---
name: vue3-nuxt4
description: Nuxt 4 新特性与迁移指南：Nitro 引擎、Route Rules、Hybrid Rendering、模块系统升级
tags: [vue3, nuxt, ssr, nitro, fullstack]
---

## 概述

Nuxt 4（v4.0.0 2025-10 GA，当前 v4.4.6 2026-05）是 Nuxt 3 的重大升级版本。核心变化：全新的 Nitro v2 引擎、Hybrid Rendering（混合渲染）、增强的 Route Rules、模块系统重写。Nuxt 3 将在 2026-07-31 停止维护。

## 一、重大变化概览

| 特性 | Nuxt 3 | Nuxt 4 |
|------|--------|--------|
| 服务端引擎 | Nitro v1 | Nitro v2（更快构建+更小包体） |
| 渲染模式 | 全局 SSR/SSG | 按路由 Hybrid（Route Rules） |
| 目录结构 | `pages/` 扁平 | 支持多层 layouts 嵌套 |
| 模块系统 | 手动注册 | 自动发现 + Hooks 增强 |
| WebSocket | 需第三方 | 内置 WebSocket 支持 |
| 构建工具 | Vite 5 | Vite 6（默认）|

## 二、Route Rules 与 Hybrid Rendering

Nuxt 4 的核心特性：在同一个应用中为不同路由指定不同渲染策略。

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    // 静态页面（构建时生成）
    '/': { prerender: true },
    '/about': { prerender: true },

    // SSR 动态渲染
    '/products/**': { ssr: true },

    // 客户端渲染（SPA）
    '/dashboard/**': { ssr: false },

    // ISR 增量静态生成
    '/blog/**': { isr: 60 },         // 60s 后重新验证
    '/docs/**': { isr: 3600 },       // 1h 后重新验证

    // API 路由不需要 SSR
    '/api/**': { ssr: false },

    // 重定向
    '/old-page': { redirect: '/new-page' },
  }
})
```

### 路由级 Layout 控制（4.3+）

```ts
export default defineNuxtConfig({
  routeRules: {
    '/admin/**': { appLayout: 'admin' },
    '/public/**': { appLayout: 'public' },
  }
})
```

## 三、Nitro v2 引擎

```ts
// server/api/products.ts
export default defineEventHandler(async (event) => {
  // Nitro 自动处理 CORS、缓存、压缩
  const query = getQuery(event)
  const products = await $fetch('http://internal-api/products', {
    params: query
  })
  return products
})

// WebSocket 处理
export default defineWebSocketHandler({
  async open(peer) {
    peer.send({ type: 'connected', id: peer.id })
  },
  async message(peer, message) {
    const data = JSON.parse(message.text())
    await handleMessage(peer, data)
  },
  async close(peer) {
    console.log('disconnected:', peer.id)
  }
})
```

```ts
// server/middleware/auth.ts — 共享中间件
export default defineEventHandler(async (event) => {
  const token = getHeader(event, 'authorization')
  if (!token) {
    throw createError({ statusCode: 401, message: 'Unauthorized' })
  }
  event.context.user = await verifyToken(token)
})
```

## 四、模块系统增强

### 自动模块发现

Nuxt 4 自动扫描 `modules/` 目录，无需手动注册。

```ts
// modules/my-module/index.ts — 自动发现
export default defineNuxtModule({
  meta: {
    name: 'my-module',
    version: '1.0.0'
  },
  setup(options, nuxt) {
    // 注册 hooks
    nuxt.hook('app:resolve', (app) => {
      console.log('App resolved:', app)
    })

    // 添加插件
    this.addPlugin({
      src: resolve(__dirname, './templates/plugin.ts'),
      mode: 'server'
    })
  }
})
```

### Hooks 增强

```ts
// 在模块中监听
nuxt.hook('build:before', () => { /* ... */ })
nuxt.hook('nitro:build:public-assets', (assets) => { /* ... */ })
nuxt.hook('pages:extend', (pages) => { /* 添加/修改页面 */ })
```

## 五、从 Nuxt 3 迁移

### 1. 检查兼容性

```bash
npx nuxi upgrade --force
npx nuxi prepare  # 检查配置问题
```

### 2. 更新配置

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  // Nuxt 4 默认使用 Vite 6
  compatibilityVersion: 4,

  // 迁移完成前可保持 Nuxt 3 行为
  future: {
    compatibilityVersion: 3  // 过渡期使用
  }
})
```

### 3. nitro.config 调整

```ts
// nuxt.config.ts — Nitro v2 配置
nitro: {
  preset: 'node-server',      // 部署目标
  storage: {
    'redis': {
      driver: 'redis',
      host: process.env.REDIS_HOST,
      port: 6379
    }
  },
  experimental: {
    openAPI: true  // 自动生成 API 文档
  }
}
```

## 六、部署配置

```bash
# 构建静态站点
npm run generate

# 构建 SSR 应用
npm run build
node .output/server/index.mjs

# Docker 部署
# Dockerfile
FROM node:22-alpine
COPY .output /app
EXPOSE 3000
CMD ["node", "/app/server/index.mjs"]
```

```yaml
# Docker Compose 全栈部署
services:
  nuxt:
    build: .
    ports:
      - "3000:3000"
    environment:
      NITRO_PRESET: node-server
      DB_URL: postgresql://postgres:password@db:5432/app
      REDIS_URL: redis://redis:6379
    depends_on:
      - db
      - redis

  db:
    image: postgres:18
  redis:
    image: redis:7-alpine
```

## 七、性能优化

```ts
export default defineNuxtConfig({
  // 代码分割优化
  experimental: {
    asyncEntry: true,     // 异步入口，减少首屏 JS
    sharePrerenderCache: true  // 共享预渲染缓存
  },

  // 资源内联
  vite: {
    inlineStylesheets: 4096,  // 内联小于 4KB 的 CSS
    include: ['src/**/*.vue']
  },

  // 图片优化
  image: {
    provider: 'ipx',
    screens: {
      xs: 320,
      sm: 640,
      md: 768,
      lg: 1024,
      xl: 1280
    }
  }
})
```

## 注意事项

| 要点 | 说明 |
|------|------|
| **compatibilityVersion** | 迁移时不要急于设置 `compatibilityVersion: 4`，先用 `future: { compatibilityVersion: 3 }` 过渡 |
| **Nitro 预设** | 不同部署平台需要匹配 Nitro preset（`node-server` / `vercel` / `cloudflare-pages`）|
| **ISR 限制** | ISR 需要 Node.js 20+，且部署平台需支持（Vercel / Netlify 原生支持）|
| **WebSocket** | Nitro v2 WebSocket 在生产环境需确保反向代理配置 WebSocket 升级 |
| **模块兼容** | 社区模块可能尚未迁移到 Nuxt 4，检查 modules 文档的 Nuxt 4 兼容性 |
| **Nuxi CLI** | Nuxt 4 使用 `nuxi` 命令（不再是 `nuxt`），`npx nuxi dev` 启动开发服务器 |
| **SSL/TLS** | 生产环境反向代理（Nginx/Caddy）在前端终止 SSL，确保 Nitro 内部使用 HTTP |
