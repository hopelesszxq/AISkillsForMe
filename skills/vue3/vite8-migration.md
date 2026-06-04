---
name: vite8-migration
description: Vite 8 完整升级指南——从 Vite 6/7 迁移、Rolldown 1.0 稳定版、v2 原生插件、DevTools 集成与破坏性变更
tags: [vue3, vite, rolldown, migration, bundler, frontend, build-tools]
---

## 概述

Vite 从 6 到 8 经历了两次重大版本升级：

| 版本 | 发布日期 | 关键变化 |
|------|---------|---------|
| Vite 8.0.x | 2026-03-12 | Rolldown 1.0 合并、v2 原生插件默认启用、DevTools 集成 |
| Vite 7.0.x | 2025-06-24 | 移除 Node 18/ESM 仅支持、Sass 编译器 API 统一、buildApp 钩子 |
| Vite 6.0.x | 2024-11-26 | 首次引入 Rolldown 作为底层打包器 |

当前最新稳定版：**Vite 8.0.16**（2026-06-01），Rolldown 1.0.3 稳定版。

> 已有技能 `vite6-rolldown.md` 覆盖 Vite 6→Rolldown 初始迁移，本文补充 Vite 7→8 的完整变化。

## 一、Vite 7：ESM 唯一、Sass 统一

### 破坏性变更

1. **Node.js 最低版本提升**：要求 Node 20.19+ / 22.12+，移除 Node 18 支持
2. **移除 CJS 构建**：Vite CLI 和核心库仅提供 ESM 格式
3. **Sass 编译器统一**：`css.preprocessorOptions.sass` 始终使用 Sass Compiler API，移除 Legacy JS API
4. **废弃移除**：
   - `splitVendorChunkPlugin` 已移除，使用 `manualChunks` 替代
   - `experimental.skipSsrTransform` 已移除
   - `HotBroadcaster` 类型已移除
5. **构建目标提升**：默认 target 变为 `baseline-widely-available`（ES2022+）

### 新特性

```ts
// vite.config.ts
export default defineConfig({
  // glob 支持 base 选项
  resolve: {},
  build: {
    // buildApp 钩子——在应用构建完成后执行
    buildApp: (app, config) => {
      console.log('Build complete!')
    }
  },
  css: {
    // preprocessorMaxWorkers 已稳定化，默认启用
    preprocessorMaxWorkers: true
  },
  optimizeDeps: {
    // noDiscovery 已稳定化
    noDiscovery: true
  }
})
```

## 二、Vite 8：Rolldown 1.0 完全合并

### 破坏性变更

1. **Rolldown 成为默认打包器**（"the epic rolldown-vite merge"，#21189）
   - 开发模式和生产构建统一使用 Rolldown（Rust）
   - esbuild 仅作为可选优化依赖保留
2. **移除 `import.meta.hot.accept` 解析回退**：必须显式声明模块路径
3. **默认浏览器目标更新**：移除了对非常老旧浏览器的支持
4. **v2 原生插件默认启用**：不再需要 `experimental.enableNativePlugin` 配置

### 迁移步骤

#### 1. 更新依赖

```bash
npm install vite@^8.0.0 @vitejs/plugin-vue@^5.0.0
# 如果使用 React
npm install @vitejs/plugin-react@^5.0.0
```

> **Node 版本要求**：Node 20.19+ / 22.12+（与 Vite 7 相同）

#### 2. 配置适配

```ts
// vite.config.ts — Vite 8 完全兼容
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  
  // ✅ v2 原生插件默认启用，无需额外配置
  
  // ⚠️ 注意：如果之前使用了自定义 esbuild 配置项，
  //    Vite 8 会发出警告建议迁移到 Rolldown 等价配置
  build: {
    // Rolldown 是唯一打包器
    minify: true,         // 使用内置 napi-rs minifier
    target: 'es2022',     // 默认已是最新
    
    // 已废弃：rolldownOptions 会被合并到顶层 build 配置
    // ❌ 之前 Vite 6 的配置项可能需要调整
  },
  
  // HMR 路径必须显式声明（移除 import.meta.hot.accept 回退）
})
```

#### 3. 插件兼容性

Vite 8 引入了 **v2 原生插件系统**，默认启用：

```ts
// 旧版插件（v1）仍然兼容，但会收到迁移建议
// 新版 v2 原生插件使用 Rolldown 原生能力，性能更好

// 如果检测到 vite-tsconfig-paths 插件，Vite 8 会发出警告
// → 因为它被 v2 原生插件的路径解析功能替代
```

### 重要新特性

#### 1. DevTools 集成

```ts
// vite.config.ts
export default defineConfig({
  // Vite 8 内置了 DevTools 集成
  // 开发模式下可通过快捷键打开
  // 默认快捷键：Ctrl+Shift+I（同浏览器 DevTools）
})
```

Vite 内置的 DevTools 支持：
- 查看模块依赖图
- 实时模块热更新状态
- 性能分析
- 浏览器的 console 日志同步到终端

#### 2. 浏览器日志转发到终端

```ts
// 开发模式下，浏览器中的 console.log/error/warn 会自动
// 转发到 Vite 终端，无需手动查看浏览器控制台
// 配置：forwardBrowserLogs（默认 true）
```

#### 3. 实验性 Full Bundle Mode

```ts
// vite.config.ts
export default defineConfig({
  experimental: {
    // 实验性：完整打包模式
    // 将应用打包为单个 JS 产物（类似传统打包工具）
    // 适合部署到某些限制性环境
    fullBundle: true
  }
})
```

#### 4. WASM SSR 支持

```ts
// 在 SSR 中加载 WASM 模块
import wasmModule from './module.wasm?init'

// Vite 8 支持 SSR + WASM 组合
// 需要显式使用 ?init 后缀
```

#### 5. 忽略过时优化请求

```ts
export default defineConfig({
  optimizeDeps: {
    // 当依赖变化时，忽略正在处理的过期请求
    ignoreOutdatedRequests: true
  }
})
```

#### 6. CSS 构建目标新选项

```ts
export default defineConfig({
  build: {
    cssTarget: 'es2025'  // 如使用 lightningcss
    // 或通过 css.lightningcss 配置
  },
  css: {
    lightningcss: {}  // 启用 lightningcss 支持 ES2024/2025 target
  }
})
```

## 三、Vite 8 关键变化总览

| 变化 | Vite 6 | Vite 7 | Vite 8 |
|------|--------|--------|--------|
| 开发打包器 | Rolldown（实验） | Rolldown | Rolldown（默认） |
| 生产打包器 | Rolldown/Rollup | Rolldown | Rolldown（唯一） |
| Node 最低版本 | 18+ | 20.19+ | 20.19+ |
| 构建格式 | CJS + ESM | ESM 仅 | ESM 仅 |
| 原生插件 | 实验性 | 实验性 | 默认启用 |
| DevTools | ❌ | ❌ | ✅ 内置 |
| WASM SSR | ❌ | ❌ | ✅ |

## 四、从 Vite 6 升级到 8 的常见问题

### 1. esbuild 配置被忽略

```ts
// ❌ Vite 6 写法（Vite 8 会警告）
{
  optimizeDeps: {
    esbuild: {
      // 自定义 esbuild 配置不再生效
    }
  }
}

// ✅ Vite 8 写法
{
  // 使用 Rolldown 等价配置
  build: {
    // Rolldown 原生配置项
  }
}
```

### 2. CSS 预处理器配置迁移

```ts
// Vite 6 兼容写法（Vite 7+ 强制使用 Compiler API）
{
  css: {
    preprocessorOptions: {
      sass: {
        // ❌ Legacy JS API 不再支持
        // 使用 modern Compiler API
        silenceDeprecations: ['legacy-js-api'],
        api: 'modern-compiler'
      }
    }
  }
}
```

### 3. splitVendorChunkPlugin 替代方案

```ts
// ❌ Vite 6 方案
import { splitVendorChunkPlugin } from 'vite'
plugins: [splitVendorChunkPlugin()]

// ✅ Vite 7+ 方案
{
  build: {
    rollupOptions: {
      output: {
        manualChunks(id) {
          if (id.includes('node_modules')) {
            if (id.includes('vue')) return 'vendor-vue'
            return 'vendor'
          }
        }
      }
    }
  }
}
```

### 4. import.meta.hot.accept 必须显式声明路径

```ts
// ❌ Vite 6 允许未指定路径的回退
import.meta.hot.accept(() => {})

// ✅ Vite 8 必须声明模块路径
import.meta.hot.accept('./module.js', () => {})
```

## 五、Rolldown 1.0 稳定版特性

Vite 8.0.12+ 使用 Rolldown 1.0.0 正式版，具有以下优势：

- **速度**：相比 Vite 5 (esbuild+Rollup) 构建速度提升 5-10x
- **兼容性**：95%+ Rollup 插件兼容
- **内存**：Rust 编译，内存占用更低
- **HMR**：~10ms 热更新（Vite 5 约 50ms）
- **类型支持**：从 `rolldown/utils` 导出 Visitor、ESTree 类型

## 注意事项

1. **升级链路**：如果当前使用 Vite 5，建议先升级到 Vite 6/7 过渡，再升级到 8
2. **插件测试**：检查使用的所有 Rollup 插件是否兼容 Rolldown（[兼容性列表](https://rolldown.dev/compat)）
3. **SSR 项目**：检查 SSR 构建配置——Vite 8 的 WASM SSR 支持是新增能力
4. **Node 版本**：务必确认 CI/CD 环境使用 Node 20.19+ 或 22.12+
5. **vite-tsconfig-paths**：Vite 8 内置了路径解析替代能力，建议移除该插件
6. **旧版浏览器兼容**：Vite 8 默认 target 提升，如需兼容旧浏览器需显式配置 `build.target`
