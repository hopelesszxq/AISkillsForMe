---
name: vueuse-143-composables
description: VueUse 14.3 新特性与高级 Composable 模式——useElementVisibility 增强、useTextareaAutosize、createReusableTemplate、createInjectionState 改进
tags: [vue3, vueuse, composables, hooks, frontend]
---

## 概述

VueUse 是 Vue 3 生态中最重要的组合式工具库，截至 2026 年 5 月已发布 **v14.3.0**。本文介绍 VueUse 14.2~14.3 新增/增强的重要 composable 及其实际用法。

## 新增 Composable

### 1. useCssSupports（14.2 新增）

检测浏览器 CSS 特性支持，SSR 安全：

```typescript
import { useCssSupports } from '@vueuse/core'

const supportsGrid = useCssSupports('display', 'grid')
const supportsHas = useCssSupports('selector(:has(> *))')

// 在模板中条件渲染
// <div v-if="supportsGrid">Grid Layout</div>
// <div v-else>Fallback Layout</div>
```

**注意事项**：未 mount 前返回 SSR 默认值（`false`），可通过 `ssrValue` 选项自定义。

### 2. useElementVisibility 新增 controls 选项（14.3）

增强版可见性检测，支持暂停/恢复：

```typescript
import { useElementVisibility } from '@vueuse/core'

const target = ref<HTMLElement | null>(null)
const { isVisible, pause, resume, stop } = useElementVisibility(
  target,
  { controls: true }
)

// 只在元素可见时加载数据
watch(isVisible, (visible) => {
  if (visible) resume()
  else pause()  // 不可见时暂停轮询
})
```

### 3. useIntersectionObserver reactive rootMargin（14.2）

`rootMargin` 支持响应式，动态调整观测区域：

```typescript
import { useIntersectionObserver } from '@vueuse/core'

const margin = ref('0px')
const target = ref<HTMLElement | null>(null)
const { isIntersecting } = useIntersectionObserver(
  target,
  ([entry]) => console.log(entry.isIntersecting),
  { rootMargin: margin }
)

// 动态扩大观测区域
margin.value = '100px'
```

### 4. useTextareaAutosize 新增 maxHeight（14.3）

限制自动高度增长的上限：

```vue
<script setup lang="ts">
import { useTextareaAutosize } from '@vueuse/core'

const { textareaRef, input } = useTextareaAutosize({ maxHeight: 300 })
</script>

<template>
  <textarea
    ref="textareaRef"
    v-model="input"
    placeholder="自动增长高度，最多 300px..."
  />
</template>
```

不再需要手动监听 `input` 事件+计算高度。

### 5. onLongPress 暴露 PointerEvent（14.3）

长按事件回调现在可以获取原始的 PointerEvent：

```typescript
import { onLongPress } from '@vueuse/core'

onLongPress(
  targetRef,
  (event: PointerEvent) => {
    console.log('长按位置:', event.clientX, event.clientY)
    // 可获取 pressure、pointerType 等指针信息
  },
  { modifiers: ['prevent'] }
)
```

## 高级 Composable 改进

### 1. createInjectionState 支持非 undefined 默认值（14.3）

注入状态不再强制要求消费者处理 `undefined`：

```typescript
// store.ts
import { createInjectionState } from '@vueuse/core'

const [useProvideCounter, useCounter] = createInjectionState((initial: number = 0) => {
  const count = ref(initial)
  const increment = () => count.value++
  return { count, increment }
})

// ✅ 14.3 起可以提供默认值，返回类型不再包含 undefined
// 消费者可以直接解构，无需空值检查
```

### 2. createReusableTemplate 支持组件命名（14.3）

复用模板现在可指定组件名称，便于 DevTools 调试：

```vue
<script setup lang="ts">
import { createReusableTemplate } from '@vueuse/core'

const [DefineHeader, ReuseHeader] = createReusableTemplate({
  name: 'SectionHeader'  // Vue DevTools 中显示为 <SectionHeader>
})
</script>

<template>
  <DefineHeader>
    <h1 class="title">
      <slot name="icon" />
      <slot />
    </h1>
  </DefineHeader>

  <ReuseHeader>首页</ReuseHeader>
  <ReuseHeader>
    <template #icon>⭐</template>
    收藏
  </ReuseHeader>
</template>
```

### 3. useDraggable 容器内拖拽自动滚动（14.2）

在受限容器内拖拽时支持自动滚动：

```typescript
import { useDraggable } from '@vueuse/core'

const el = ref<HTMLElement | null>(null)
const container = ref<HTMLElement | null>(null)

const { x, y, isDragging } = useDraggable(el, {
  containerElement: container,
  // 拖拽到边界时自动滚动容器
  preventDefault: true
})
```

### 4. useSortable watchElement 选项（14.2）

元素变化时自动重新初始化排序：

```vue
<script setup lang="ts">
import { useSortable } from '@vueuse/core'

const items = ref([...])  
const el = ref<HTMLElement | null>(null)

useSortable(el, items, {
  watchElement: true,  // el 变化时自动重初始化
  handle: '.drag-handle'
})
</script>
```

### 5. 可配置调度器（14.2）

定时 composable（`useIntervalFn`、`useTimeoutFn` 等）支持自定义调度器：

```typescript
import { useIntervalFn } from '@vueuse/core'

// 使用 requestAnimationFrame 替代 setInterval
const { pause, resume } = useIntervalFn(
  () => updateAnimation(),
  16,  // ~60fps
  { scheduler: 'requestAnimationFrame' }
)

// 或使用自定义调度器
const { pause, resume } = useIntervalFn(
  () => syncData(),
  5000,
  {
    scheduler: (callback, interval) => {
      // 自定义调度逻辑
      const id = setInterval(callback, interval)
      return () => clearInterval(id)
    }
  }
)
```

## Nuxt 集成增强

VueUse 14.3 增强了 Nuxt 集成，composable 变体现在支持自动导入：

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@vueuse/nuxt'],
  vueuse: {
    // 自动导入所有 composable（tree-shaking）
    autoImports: true,
    // 排除不需要的 composable
    excludes: ['useAsyncState']
  }
})
```

## 注意事项

- `useCssSupports` 在 SSR 环境 mount 前返回 `false`，注意水合不匹配问题
- `createInjectionState` 的默认值功能需要 VueUse >= 14.3.0
- `useDraggable` 的 `containerElement` 选项在 14.2 以上才支持自动滚动
- VueUse 14.3 添加了 `./package.json` 导出字段，确保与最新打包工具兼容
- Nuxt 模块 `@vueuse/nuxt` 自动导入是 tree-shaking 安全的，无需担心 bundle 体积
- VueUse 支持 `vue-router` 5 作为 peer dependency（14.2+）
