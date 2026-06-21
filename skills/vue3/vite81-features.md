---
name: vite81-features
description: Vite 8.1 beta 新特性：WASM ESM 集成、Importmap 构建分块、Vite Task 零配置缓存、server.hmr→ws 重命名
tags: [vue3, vite, wasm, importmap, rolldown, build-tools]
---

## 概述

Vite 8.1.0-beta.0（2026-06-15 发布）在 Rolldown 1.0 稳定版基础上引入了多项新特性和改进。主要亮点：**WASM ESM 集成**（直接导入 `.wasm` 文件）、**Importmap 构建分块**、**Vite Task 零配置构建缓存**、以及配置属性的破坏性变更。

> Vite 8.0.x 的迁移指南请参见 `vite8-migration.md`，本文聚焦 8.1 beta 的新增内容。

## 主要新特性

### 1. WASM ESM 集成（WASM ESM Integration）

Vite 8.1 支持直接导入 `.wasm` 文件作为 ES Module，无需手动调用 `WebAssembly.instantiate()`：

```ts
// ✅ Vite 8.1 — 直接导入
import { add, multiply } from './math.wasm'

console.log(add(2, 3)) // 5
```

底层使用 WASM ESM Integration 规范，Rolldown 会自动处理 WASM 模块的编译和实例化流程。支持：

- 默认导出和命名导出（取决于 WASM 模块的导出）
- 同步调用（WebAssembly 在 Vite 开发模式下保持同步）
- 浏览器和生产构建自动兼容

```ts
// vite.config.ts
export default defineConfig({
  build: {
    target: 'es2022' // 需要支持 WASM ESM 的 target
  }
})
```

### 2. Importmap 构建分块（build chunk importmap）

Vite 8.1 新增 `build.rollupOptions.output.importmap` 配置，支持将构建产物中的共享依赖自动提取为 Import Map：

```ts
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        importmap: {
          // 自动提取匹配的模块为 importmap entries
          entries: ['react', 'react-dom', 'vue']
        }
      }
    }
  }
})
```

构建输出示例（`index.html`）：

```html
<script type="importmap">
{
  "imports": {
    "vue": "/assets/vue-3afe4d8e.js",
    "react": "/assets/react-18d5e2f.js"
  }
}
</script>
```

**适用场景**：
- 微前端架构（多子应用共享依赖）
- CDN 分发优化（依赖单独缓存）
- 多页面应用共享依赖

### 3. Vite Task 集成（零配置构建缓存）

Vite 8.1 集成了 **Vite Task**，一个零配置的构建缓存系统，自动缓存和复用构建结果：

```ts
// vite.config.ts
export default defineConfig({
  // 默认启用 Vite Task 缓存
  task: {
    enabled: true,             // 默认 true
    cacheDir: '.vite-task',    // 缓存目录
    strategy: 'content-hash'   // content-hash | timestamp
  }
})
```

在 CI 环境中配合 `VITE_TASK_CACHE_KEY` 环境变量可实现跨构建缓存共享：

```bash
# CI 中使用
VITE_TASK_CACHE_KEY=$CI_COMMIT_SHA npx vite build
```

Vite Task 自动追踪所有构建输入（源码、配置、依赖）的变化，仅重新构建发生变化的部分。

### 4. server.hmr → server.ws 重命名

Vite 8.1 将 `server.hmr` 配置重命名为 `server.ws`，更准确地反映 WebSocket 连接的职责：

```ts
// vite.config.ts
export default defineConfig({
  server: {
    // ✅ Vite 8.1 新风格
    ws: {
      protocol: 'wss',
      host: 'example.com',
      port: 443
    }
  }
})
```

> `server.hmr` 仍然向后兼容，但会触发弃用警告。建议新项目直接使用 `server.ws`。

### 5. 其他改进

| 特性 | 说明 |
|------|------|
| `import.meta.glob` `caseSensitive` | glob 匹配支持大小写敏感选项 |
| LightningCSS 插件依赖 | CSS 支持 lightningcss plugin dependency |
| HTML `additionalAssetSources` | 自定义 HTML 中额外资源来源 |
| 多 hosts 支持 | `__VITE_ADDITIONAL_SERVER_ALLOWED_HOSTS` 支持多个 host |
| Rolldown 1.1.1 | 底层 Rolldown 升级到 1.1.1 |
| 原生配置加载跟踪 | 使用原生工具加载配置时追踪依赖变化 |

#### import.meta.glob caseSensitive

```ts
// 大小写敏感匹配
const modules = import.meta.glob('./**/*.ts', { caseSensitive: true })
```

#### 多 hosts 配置

```bash
# 环境变量中支持多个 host
__VITE_ADDITIONAL_SERVER_ALLOWED_HOSTS=example.com,api.example.com
```

## 注意事项

1. **WASM ESM 需要合适的 `build.target`**：`es2022` 或更高版本，旧浏览器需要 polyfill
2. **server.hmr 已废弃**：新代码使用 `server.ws`，旧配置仍兼容
3. **Vite Task 默认启用**：如果遇到缓存问题，可设置 `task.enabled: false` 临时关闭
4. **Importmap 与旧浏览器兼容性**：Import Map 需要 Chromium 89+/Firefox 108+/Safari 16.4+
5. **Vite 8.1 仍为 beta 版本**：不建议直接用于生产环境，等待正式版发布后再升级
