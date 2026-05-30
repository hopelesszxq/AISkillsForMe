---
name: vue36-vapor-mode
description: Vue 3.6 Vapor Mode 新编译策略——无虚拟 DOM、更高性能、更小体积
tags: [vue3, vapor, compiler, performance, rendering]
---

## 概述

Vue 3.6 正在 Beta 阶段（当前最新 v3.6.0-beta.13，2026-05-28），引入了革命性的 **Vapor Mode**（汽化模式）——一种**无虚拟 DOM（Virtual DOM-less）** 的编译策略，旨在为高性能场景提供极致优化。

> 当前稳定版为 Vue 3.5.35（2026-05-27），Vapor Mode 仍处于 Beta 阶段，预计随 3.6 正式版发布。

## Vapor Mode 原理

### 传统 Vue 渲染流程

```
模板 → 编译 → Virtual DOM → Diff → DOM 操作
```

### Vapor Mode 渲染流程

```
模板 → 编译 → 直接命令式 DOM 操作（跳过 VDOM）
```

Vapor Mode 在**编译阶段**就分析模板结构，直接生成精细化的 DOM 操作代码，完全跳过 Virtual DOM 的创建和 Diff 过程。

### 关键区别

| 维度 | 传统模式 | Vapor Mode |
|------|---------|------------|
| 虚拟 DOM | ✅ 有 | ❌ 无 |
| 运行时开销 | 中等（patch + diff） | 极低（直接 DOM 操作） |
| 包体积 | 较大（含完整运行时） | 更小（仅保留响应式系统） |
| 初次渲染 | VDOM 创建 + diff | 直接生成 DOM |
| 更新粒度 | 组件级 diff | 精确的绑定级更新 |
| 兼容性 | 全面 | 有限（部分动态语法受限） |

## 使用方式

### 按组件启用

Vapor Mode 可以**逐组件启用**，无需全量切换：

```vue
<!-- 在 SFC 中通过 vapor 标记启用 -->
<template vapor>
  <div>
    <h1>{{ title }}</h1>
    <p v-for="item in items" :key="item.id">
      {{ item.name }}
    </p>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const title = ref('Hello Vapor')
const items = ref([
  { id: 1, name: 'Item A' },
  { id: 2, name: 'Item B' }
])
</script>
```

### 全局配置

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [
    vue({
      vapor: true,  // 全局启用 Vapor Mode
      // 或选择性启用
      vapor: {
        include: ['**/vapor/**/*.vue'],
        exclude: ['**/legacy/**/*.vue']
      }
    })
  ]
})
```

## Vapor 运行时特性

### 1. 编译器优化

Vapor Mode 在编译阶段进行大量优化：

```ts
// 编译前（模板）
// <div :class="active ? 'on' : 'off'" @click="handleClick">{{ text }}</div>

// 编译后（Vapor）— 直接操作 DOM，无 VDOM
import { template, on, effect, className, child } from 'vue/vapor'

const t = template('<div></div>')
export function render(ctx) {
  const div = t()
  on(div, 'click', ctx.handleClick)
  effect(() => className(div, ctx.active ? 'on' : 'off'))
  effect(() => child(div, ctx.text))
  return div
}
```

### 2. 响应式绑定优化

- **v-once**：编译阶段直接静态化，完全跳过响应式追踪
- **v-memo**：更高效的记忆化缓存
- **v-model**：编译为直接事件处理器 + 属性设置

### 3. 条件/循环渲染

```vue
<template vapor>
  <!-- v-if 编译为条件 DOM 创建/销毁 -->
  <div v-if="visible">可见</div>
  
  <!-- v-for 编译为有序的 DOM 插入/移动/移除 -->
  <li v-for="item in items" :key="item.id">{{ item }}</li>
  
  <!-- v-show 变更为 display 属性切换 -->
  <div v-show="visible">显示切换</div>
</template>
```

## 性能基准

根据 Vue 团队公布的测试数据：

| 基准测试 | 传统 VDOM | Vapor Mode | 提升 |
|---------|-----------|------------|------|
| 首次渲染 | 基准 | -40%~-60% | ~2x |
| 组件更新 | 基准 | -50%~-70% | ~2-3x |
| 内存占用 | 基准 | -30%~-50% | ~2x |
| 包体积 (gzip) | ~16KB | ~8KB | ~50% |

## 与现有模式的共存

### 组件间互操作

Vapor 组件和传统 VDOM 组件可以在同一个应用中**混合使用**：

```vue
<!-- 传统组件可以包含 Vapor 子组件 -->
<template>
  <div>
    <VaporChild />  <!-- Vapor 子组件 -->
    <LegacyChild /> <!-- 传统 VDOM 子组件 -->
  </div>
</template>
```

Vue 运行时自动处理两者之间的**插槽、props、emit** 互操作。

### 限制

Vapor Mode 目前不支持的特性：

- ❌ 动态组件 `<component :is="...">`
- ❌ `<KeepAlive>`（有部分支持，但功能受限）
- ❌ `<Teleport>` 内使用 Vapor 组件（Beta 阶段限制）
- ❌ `<Suspense>` 与 Vapor 组件互操作（Beta 阶段限制）
- ❌ `this.$refs` 在选项式 API 中（需使用 `<script setup>`）
- ⚠️ `Transition` / `TransitionGroup` — Beta 阶段持续改进中

## 迁移策略

### 推荐路径

```
1. 全面测试 → 2. 组件级逐步迁移 → 3. 性能热点优先 → 4. 全量切换
```

### 性能热点识别

```ts
// 使用 Vue DevTools 分析组件渲染性能
// 对渲染次数多、更新频繁的组件优先迁移到 Vapor Mode

// 示例：高性能列表组件
<template vapor>
  <div class="virtual-list">
    <div v-for="item in visibleItems" :key="item.id"
         :style="{ transform: `translateY(${item.offset}px)` }">
      {{ item.label }}
    </div>
  </div>
</template>
```

## 注意事项

1. **Beta 阶段不建议用于生产**：Vue 3.6 仍处于 Beta，Vapor Mode 的边界情况尚未完全覆盖
2. **Compiler-vapor 非默认**：需要在 Vite 插件中显式启用 `vapor: true`
3. **不能全部迁移**：复杂动态组件、高阶 HOC 模式可能无法使用 Vapor Mode
4. **调试体验差异**：Vapor 生成的代码是命令式 DOM 操作，Vue DevTools 的支持正在完善
5. **TSX 支持有限**：Vapor Mode 主要针对 SFC 模板优化，TSX/JSX 渲染函数的收益较小
6. **搭配 Pinia 无额外配置**：Vapor 组件直接使用 `useStore()`，完全兼容 Pinia
7. **关注 3.6 正式版**：预计 2026 Q3 发布正式版，届时生态工具（Vite DevTools、ESLint 插件）将同步更新
