---
name: vite6-rolldown
description: Vite 6 / Rolldown 构建工具新特性：Rolldown 打包器迁移、极速 HMR、微前端支持
tags: [vue3, vite, rolldown, build, bundler, frontend]
---

## 概述

Vite 6（预计 2025 年中发布）最重大的变化是使用 **Rolldown** 替代 esbuild/Rollup 作为底层打包器。Rolldown 是 Rust 编写的打包工具，兼容 Rollup 插件生态，速度提升 5-10 倍。

## Rolldown 核心优势

| 对比项 | Vite 5 (esbuild + Rollup) | Vite 6 (Rolldown) |
|--------|--------------------------|-------------------|
| 开发模式 | esbuild（Go） | Rolldown（Rust） |
| 生产构建 | Rollup（JS） | Rolldown（Rust） |
| HMR 热更新 | ~50ms | ~10ms |
| 冷启动 | ~2s | ~500ms |
| 插件生态 | Rollup 插件 | 兼容 Rollup 插件（95%+） |

## 升级到 Vite 6

### 1. 更新依赖

```bash
npm install vite@^6.0.0 @vitejs/plugin-vue@^5.0.0
```

### 2. vite.config.ts 兼容性

```ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import path from 'path'

// Vite 6 配置基本兼容，但部分 esbuild 特定配置需迁移
export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src'),
    },
  },
  build: {
    target: 'es2022',
    // Vite 6 不再支持 esbuild.minify
    // ✅ 使用自带的 napi-rs minifier
    minify: 'esbuild', // TODO: 迁移为 'rolldown' 以获得最佳性能
    
    // ⚠️ Vite 6 变化：rollupOptions → build.rollupOptions 仍兼容
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['vue', 'vue-router', 'pinia'],
        },
      },
    },
  },
  // Vite 6 新增：优化依赖预构建
  optimizeDeps: {
    include: ['vue', 'vue-router', 'pinia'],
  },
})
```

### 3. 迁移清单

```ts
// ❌ Vite 5（esbuild 特有配置）
{
  esbuild: {
    drop: ['console', 'debugger'],
    pure: ['console.log'],
  },
}

// ✅ Vite 6（使用 Rolldown 方式）
{
  build: {
    drop: ['console', 'debugger'],  // 原生支持
    pure: ['console.log'],          // 原生支持
  },
}
```

## 极速 HMR 实践

### 1. 组件级热更新优化

```vue
<script setup lang="ts">
// Vite 6 HMR 不再需要 accept 声明
// 以下写法在 Vite 6 中默认支持
const count = ref(0)

// 状态持久化 HMR（Vite 6 新增策略）
// 更新组件时保留 ref 状态，无需手动 preserve
</script>

<template>
  <div>{{ count }}</div>
</template>
```

### 2. CSS HMR 改进

```scss
// Vite 6 支持 CSS 模块的热替换不丢失 JS 状态
// 修改 .scss/.less 文件时只会热替换样式模块

// _variables.scss
$primary-color: #409eff;
```

### 3. 依赖预构建

```ts
// vite.config.ts
export default defineConfig({
  // Vite 6 自动检测依赖中使用的 ESM/CJS 并进行预构建
  // 新增 exclude 优化
  optimizeDeps: {
    exclude: ['your-slow-dep'], // 排除不兼容 Rolldown 的依赖
    // Vite 6 新增：禁用预构建调试
    disabled: false,
  },
})
```

## 微前端支持增强

### Module Federation 原生支持

```ts
// vite.config.ts - 使用 @originjs/vite-plugin-federation
import federation from '@originjs/vite-plugin-federation'

export default defineConfig({
  plugins: [
    vue(),
    federation({
      name: 'my-app',
      filename: 'remoteEntry.js',
      exposes: {
        './HelloWorld': './src/components/HelloWorld.vue',
      },
      shared: ['vue', 'vue-router'],
    }),
  ],
  // Vite 6 对 Module Federation 有更好的兼容性
  build: {
    target: 'es2022',
    // 确保 federation 正常运行
    cssCodeSplit: false,
  },
})
```

### 消费远程模块

```ts
// 主应用
import { defineAsyncComponent } from 'vue'

// Vite 6 中远程组件加载更快
const RemoteHello = defineAsyncComponent(() =>
  import('remote_app/HelloWorld')
)
```

## 性能优化技巧

### 1. 构建分析

```bash
# Vite 6 内置更强大的分析工具
npx vite build --analyze

# 或使用
npx vite optimize --analyze
```

### 2. 代码分割策略

```ts
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        // Vite 6 + Rolldown 支持更细粒度的分割
        manualChunks(id: string) {
          if (id.includes('node_modules')) {
            // 根据包大小智能分割
            if (id.includes('echarts') || id.includes('antv')) {
              return 'charts'
            }
            if (id.includes('lodash') || id.includes('dayjs')) {
              return 'utils'
            }
            if (id.includes('monaco-editor')) {
              return 'editor'
            }
            return 'vendor'
          }
        },
      },
    },
  },
})
```

### 3. CSS 优化

```ts
// vite.config.ts
export default defineConfig({
  css: {
    // Vite 6 支持 Lightning CSS 替代 PostCSS
    transformer: 'lightningcss',
    lightningcss: {
      // 自动添加浏览器前缀 + 压缩
      targets: {
        chrome: 90,
        firefox: 90,
        safari: 15,
      },
    },
  },
})
```

## 常见迁移问题

### 1. 插件不兼容

```bash
# 检查插件是否支持 Rolldown
npm ls vite-plugin-

# 已知不兼容的插件需要替换：
# ❌ vite-plugin-imp        → ✅ 使用 unplugin-auto-import
# ❌ @vitejs/plugin-legacy  → ✅ 需要更新版本
# ❌ vite-tsconfig-paths    → ✅ 内置支持 paths
```

### 2. CJS 依赖处理

```ts
// vite.config.ts
export default defineConfig({
  // Vite 6 默认预构建 CJS 依赖
  // 如果遇到模块格式问题，显式指定
  ssr: {
    format: 'esm', // 推荐 ESM
  },
})
```

### 3. 路径别名

```ts
/// <reference types="vite/client" />

// 在 tsconfig.json 中配置 paths
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

## Rolldown 配置进阶

```ts
// rolldown.config.ts（可选，替代 vite.config.ts 进行深度定制）
import { defineConfig } from 'rolldown'

export default defineConfig({
  input: 'src/main.ts',
  output: {
    format: 'esm',
    dir: 'dist',
    // Rolldown 原生支持
    minify: true,
  },
  // 兼容大部分 Rollup 插件
  plugins: [],
})
```

## 关键注意事项

1. **Vite 6 是破坏性升级**：升级前运行 `npx vite build --debug` 检查兼容性
2. **Rolldown 不支持所有 Rollup 插件**：检查插件是否使用 `this.parse`、`this.resolve` 等内部 API
3. **迁移时先锁定 Vite 5.x**：等生态成熟后再升级
4. **开发环境已几乎无构建延迟**：10万行项目冷启动 < 1s
