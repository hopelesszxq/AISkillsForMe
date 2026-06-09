---
name: vue3-suspense-patterns
description: Vue 3 <Suspense> 异步数据处理深度实践 — Suspense 边界划分、嵌套 Suspense、错误处理、Transition 配合与 SSR 注意事项
tags: [vue3, suspense, async, ssr, error-handling, frontend]
---

## 概述

`<Suspense>` 是 Vue 3 内置的实验性组件，用于协调组件树中的异步依赖（异步 setup、异步组件），在等待异步内容时展示 fallback 内容。Vue 3.6+ 中 Suspense 稳定性大幅提升，可作为生产可用特性。

### 核心概念

| 概念 | 说明 |
|------|------|
| **Pending** | 等待异步依赖完成的状态，显示 fallback |
| **Resolved** | 所有异步依赖已就绪，显示实际内容 |
| **Rejected** | 异步依赖抛出错误，需配合 `onErrorCaptured` 或 `ErrorBoundary` 处理 |

## 1. Suspense 基础用法

### 异步组件 + Suspense

```vue
<!-- App.vue -->
<template>
  <Suspense>
    <!-- 默认插槽：异步内容就绪后显示 -->
    <Dashboard />
    <!-- fallback 插槽：等待期间显示 -->
    <template #fallback>
      <div class="loading-spinner">
        <span>加载中...</span>
      </div>
    </template>
  </Suspense>
</template>

<script setup lang="ts">
import { defineAsyncComponent } from 'vue'

const Dashboard = defineAsyncComponent(() => import('./Dashboard.vue'))
</script>
```

### 异步 setup() 函数（推荐方式）

比起 defineAsyncComponent，更推荐直接在 setup 中使用 `await`：

```vue
<!-- UserProfile.vue -->
<template>
  <div>
    <h2>{{ user.name }}</h2>
    <p>{{ user.email }}</p>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'

// ✨ setup 中可直接 await，Suspense 会等待
const response = await fetch('/api/user')
const user = await response.json()
// user 就在 template 中直接可用，无需额外解包
</script>
```

## 2. Suspense 边界划分策略

Suspense 边界的设计直接影响用户体验。错误的边界划分会导致"全屏 loading"或"内容跳跃"。

### 粒度原则

```
❌ 错误：整个页面一个 Suspense
<template>
  <Suspense>
    <WholePage />         ← 任何子组件异步未完成，整个页面白屏
    <template #fallback> <FullPageSpinner /> </template>
  </Suspense>
</template>

✅ 正确：按区域拆分 Suspense
<template>
  <PageHeader />           ← 同步内容立即显示
  <Suspense>
    <AsyncMainContent />   ← 主体内容区域 loading
    <template #fallback> <ContentSkeleton /> </template>
  </Suspense>
  <Suspense>
    <AsyncSidebar />       ← 侧边栏独立 loading
    <template #fallback> <SidebarSkeleton /> </template>
  </Suspense>
</template>
```

### 三种边界模式

| 模式 | 适用场景 | 示例 |
|------|----------|------|
| **细粒度** | 每个独立模块一个 Suspense | 仪表盘的每个 widget 独立 loading |
| **合并边界** | 相关依赖需同时就绪 | 列表 + 分页器必须同时可用 |
| **延迟展示** | 避免闪烁（结合 `@delay`） | 256ms 以内的请求直接跳过 fallback |

### 避免 Fallback 闪烁

```vue
<Suspense :timeout="300">
  <!-- timeout=300ms：在 300ms 内完成则跳过 fallback，避免闪烁 -->
  <QuickAsyncComponent />
  <template #fallback>
    <InlineSkeleton />
  </template>
</Suspense>
```

> 在 Vue 3.6+ 中，可以通过 CSS 动画配合 `@pending` 事件实现更精细的延迟控制。

## 3. 嵌套 Suspense 与依赖协调

当存在多级异步依赖时，Vue 会递归处理 Suspense 边界：

```vue
<template>
  <Suspense>
    <UserDashboard>           ← 外层 Suspense 等待所有嵌套边界
      <Suspense>
        <UserProfile />       ← 内层 Suspense 1
        <template #fallback> <ProfileSkeleton /> </template>
      </Suspense>
      <Suspense>
        <UserOrders />        ← 内层 Suspense 2
        <template #fallback> <OrdersSkeleton /> </template>
      </Suspense>
    </UserDashboard>
    <template #fallback>
      <FullPageSpinner />     ← 仅最外层 fallback
    </template>
  </Suspense>
</template>
```

**行为规则**：
- 外层 Suspense 等待**所有**内层 Suspense 的异步依赖都解析
- 内层 Suspense 的 fallback**不会显示**——外层会统一显示自己的 fallback
- 内层异步完成后，逐层向上通知，外层替换 fallback

> ⚠️ **陷阱**：如果希望内层 Suspense 独立展示 fallback（例如部分模块独立加载），保持它们**平级放置在外层 Suspense 之外**，而非嵌套在内。

## 4. 错误处理

Suspense 本身不提供错误处理，必须与错误边界配合：

```vue
<!-- ErrorBoundary.vue — 通用错误边界组件 -->
<template>
  <slot v-if="!error" />
  <div v-else role="alert">
    <h3>加载失败</h3>
    <p>{{ error.message }}</p>
    <button @click="retry">重试</button>
  </div>
</template>

<script setup lang="ts">
import { ref, onErrorCaptured } from 'vue'

const error = ref<Error | null>(null)

onErrorCaptured((err: Error) => {
  error.value = err
  return false // 阻止向上传播
})

function retry() {
  error.value = null
  // 触发上层 Suspense 重新渲染子组件
}
</script>
```

### Suspense + ErrorBoundary 组合模式

```vue
<template>
  <ErrorBoundary @retry="key++">
    <Suspense>
      <AsyncComponent :key="key" />
      <template #fallback>加载中...</template>
    </Suspense>
  </ErrorBoundary>
</template>
```

`key` 变化会强制重建 `AsyncComponent`，触发重新 await，实现重试效果。

## 5. Suspense + Vue Router 集成

```typescript
// router/index.ts
import { defineAsyncComponent } from 'vue'
import { createRouter, createWebHistory } from 'vue-router'

const routes = [
  {
    path: '/dashboard',
    component: {
      // 用异步组件包装，让 Suspense 能捕获
      render() {
        return h(Suspense, null, {
          default: () => h(DashboardPage),
          fallback: () => h(LoadingSpinner),
        })
      },
    },
  },
]
```

### 推荐方式：在 Router 中使用异步组件 + Suspense 布局

```vue
<!-- layouts/MainLayout.vue -->
<template>
  <AppHeader />
  <main>
    <Suspense>
      <RouterView />            ← 页面级异步加载
      <template #fallback>
        <PageSkeleton />
      </template>
    </Suspense>
  </main>
</template>
```

## 6. SSR 注意事项

| 环境 | Suspense 行为 | 注意事项 |
|------|--------------|---------|
| **SSR（服务器端）** | 自动解析所有异步依赖 | 确保异步操作在服务器有完整数据 |
| **SSR + 水合（Client）** | 客户端跳过初始 Suspense | 避免客户端重复请求 |
| **SSG（静态生成）** | 在构建时解析 | 异步数据需稳定可用 |

```typescript
// 在 SSR 环境下确保数据预取
// 使用 useAsyncData (Nuxt) 或 onServerPrefetch
import { onServerPrefetch } from 'vue'

const data = ref(null)

onServerPrefetch(async () => {
  data.value = await fetchData() // 服务端预取
})
```

## 7. Suspense + Pinia

```typescript
// stores/userStore.ts
export const useUserStore = defineStore('user', () => {
  const user = ref(null)
  const loading = ref(false)

  async function fetchUser(id: string) {
    loading.value = true
    try {
      // 不要在 Pinia action 里 await 自身
      // 让 Suspense 在组件层处理异步
      user.value = await api.getUser(id)
    } finally {
      loading.value = false
    }
  }

  return { user, loading, fetchUser }
})
```

```vue
<!-- component 中：调用 action 但不 await，Suspense 不会感知 Pinia 的异步 -->
<script setup lang="ts">
const store = useUserStore()
// 方案1: 手动管理 loading
await store.fetchUser('123')  // ❌ Suspense 不感知 Pinia action

// 方案2: 数据获取放在 setup 中，Suspense 拦截
const response = await fetch('/api/user')
const user = await response.json()
// ✅ Suspense 等待直到数据就绪
</script>
```

> **关键**: Suspense 只等待组件 setup 函数内的异步操作，不会等待 Pinia action 或外部 store 调用。如需 Suspense 感知 store 数据，将数据获取逻辑放在组件 setup 中。

## 注意事项

1. **稳定性标记**：Suspense 在 Vue 3.6+ 中仍标记为实验性，但已被大量生产项目采用。监控 Vue 版本的更新日志。
2. **不要滥用 Suspense 替代状态管理**：Suspense 适合一次性异步依赖（页面初始化数据），不适合交互触发的异步操作。
3. **微任务 vs 宏任务**：Suspense 基于微任务调度，fallback 切换是同步渲染，不会出现中间态闪烁问题。
4. **Teleport 与 Suspense 的交互**：Teleport 出的内容在 Suspense 等待期间仍可显示，适用于全局 toast/loading 场景。
5. **KeepAlive 不兼容**：`<KeepAlive>` 与 `<Suspense>` 不兼容，不要在 Suspense 外部包裹 KeepAlive（可用 `v-if` 控制缓存替代）。
6. **测试 Suspense 组件**：使用 `@vue/test-utils` 时，需用 `flushPromises()` 等待异步完成，或使用 `global.stubs` 模拟。
7. **Vite HMR 影响**：开发环境下 Suspense 边界可能在 HMR 更新时异常，不影响生产构建。
