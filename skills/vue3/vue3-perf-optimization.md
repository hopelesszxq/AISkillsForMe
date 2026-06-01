---
name: vue3-perf-optimization
description: Vue 3.5.x 性能优化实战：响应式缓存、事件分发优化、SSR 渲染列表、Teleport 挂载策略
tags: [vue3, performance, reactivity, ssr, teleport, v-memo]
---

## 概述

Vue 3.5.35（2026-05-27 发布）包含多项性能优化和关键 Bug 修复。本文将这些经验提炼为实用的性能优化模式，适用于 Vue 3.5+ 项目。

> 已有技能 `vue3-advanced.md` 覆盖了 Vue 3 通用高级用法，本文专注于 3.5.x 版本的性能优化实践。

## 一、响应式系统优化

### 1. 跳过已缓存代理的类型检查（3.5.35）

React 3.5.35 优化了响应式代理的缓存逻辑：对于已经缓存的 Proxy 对象，跳过重复的类型检查，减少不必要的开销。

```js
import { reactive, shallowReactive } from 'vue'

// ✅ 优化后：同一对象多次 reactive 调用不会重复类型检查
const raw = { count: 0 }
const p1 = reactive(raw)  // 创建代理，类型检查
const p2 = reactive(raw)  // ✅ 直接返回缓存代理，跳过类型检查
console.log(p1 === p2)    // true

// ❌ 避免混用 reactive 和 shallowReactive
// 不同类型代理不能共享缓存
const s = shallowReactive(raw)  // 创建新的 shallow 代理
```

### 2. 防止在停止作用域中创建孤立 Effect（3.5.34）

```js
import { effectScope, watchEffect } from 'vue'

const scope = effectScope()

scope.run(() => {
  const stop = watchEffect(() => {
    // 一些副作用逻辑
  })

  // ❌ 错误：在 scope 停止后创建 effect
  setTimeout(() => {
    scope.stop()
    // scope 已停止，但下面仍创建 effect
    watchEffect(() => { /* 这会产生孤立 effect */ })  // ⚠️ 3.5.34 修复：现在会自动抑制
  }, 100)
})
```

**最佳实践**：使用 `scope.onDispose()` 确保 scope 停止时清理所有副作用。

```js
const scope = effectScope()

scope.run(() => {
  const interval = setInterval(() => {
    // 定时任务
  }, 1000)

  scope.onDispose(() => {
    clearInterval(interval)
  })
})

// 停止 scope 时自动清理 interval
scope.stop()
```

## 二、事件分发优化

### 1. 数组事件处理器的批量分发（3.5.35）

Vue 3.5.35 优化了数组形式的事件处理器分发性能，减少遍历开销。

```vue
<template>
  <!-- ✅ 优化后：数组事件处理器在大列表中性能更好 -->
  <button @click="[handler1, handler2, handler3]">
    点击
  </button>
</template>

<script setup>
const handler1 = () => console.log('handler 1')
const handler2 = () => console.log('handler 2')
const handler3 = () => console.log('handler 3')
</script>
```

### 2. 避免 Symbol 类型 Props 验证时的类型强制转换（3.5.34）

```vue
<script setup>
// ✅ Symbol 类型的 props 验证现在更安全
const props = defineProps({
  status: {
    type: Symbol,
    default: () => Symbol('default')
  }
})

// ❌ 3.5.34 之前：Symbol 可能被强制转为字符串导致验证失败
// 3.5.34 修复：避免 Symbol 在 props 验证过程中被强制类型转换
</script>
```

## 三、SSR 渲染优化

### 1. 避免在 SSR 中物化迭代器（3.5.35）

服务端渲染时，`v-for` 遍历迭代器会产生额外内存开销。Vue 3.5.35 优化了 `ssrRenderList`，避免将迭代器完全物化为数组。

```vue
<template>
  <!-- ✅ 优化后：生成器/迭代器在 SSR 中更高效 -->
  <div v-for="item in generateItems()" :key="item.id">
    {{ item.name }}
  </div>
</template>

<script setup>
// 使用生成器函数
function* generateItems() {
  for (let i = 0; i < 1000; i++) {
    yield { id: i, name: `Item ${i}` }
  }
}
</script>
```

### 2. 传播 Suspense SSR 渲染中的同步错误（3.5.35）

```vue
<template>
  <Suspense>
    <AsyncComponent />
  </Suspense>
</template>

<script setup>
// ✅ 同步错误现在能正确传播到 Suspense 的错误处理
// 3.5.35 修复：ssrRenderSuspense 传播同步错误
</script>
```

## 四、Teleport 挂载优化

### 1. 延迟挂载时跳过子组件卸载（3.5.35）

当 Teleport 的目标元素尚未就绪时，Vue 会延迟挂载。3.5.35 修复了在此过程中未挂载的 Teleport 子组件被错误卸载的问题。

```vue
<template>
  <!-- Teleport 到动态目标 -->
  <Teleport :to="target" v-if="showTeleport">
    <ExpensiveComponent />
  </Teleport>
</template>

<script setup>
import { ref, onMounted } from 'vue'

const target = ref()
const showTeleport = ref(true)

onMounted(() => {
  // 模拟目标容器延迟出现
  setTimeout(() => {
    target.value = document.querySelector('#portal-target')
  }, 100)
})
</script>
```

### 2. Teleport 更新在延迟挂载前的正确处理（3.5.32）

```vue
<script setup>
import { ref, nextTick } from 'vue'

const portalTarget = ref()
const count = ref(0)

// ✅ 3.5.32 修复：Teleport 在延迟挂载前接收更新不会导致异常
async function updateBeforeMount() {
  count.value++
  await nextTick()
  // Teleport 尚未挂载，但响应式更新已触发
  // 修复前会导致"子组件未挂载但已更新"的警告
}

// 后续触发挂载
onMounted(() => {
  setTimeout(() => {
    portalTarget.value = document.getElementById('app-footer')
  }, 1000)
})
</script>
```

## 五、V-Memo 与 V-For 共存优化

### 1. 避免 V-Memo + V-For Keys 的双重处理（3.5.35）

```vue
<template>
  <!-- ✅ 3.5.35 修复：v-memo 与相同 key 的 v-for 不再重复处理 -->
  <div
    v-for="item in items"
    :key="item.id"
    v-memo="[item.id, item.version]"
  >
    {{ item.name }}
  </div>
</template>

<script setup>
import { ref } from 'vue'

const items = ref([
  { id: 1, name: 'A', version: 1 },
  { id: 2, name: 'B', version: 1 },
])

// 只更新 version 变化的部分
function updateItem(id) {
  const item = items.value.find(i => i.id === id)
  if (item) {
    item.version++
    item.name = `Updated ${id}`
  }
}
</script>
```

### 2. 卸载后使已分离的 V-Memo VNodes 失效（3.5.31）

```vue
<template>
  <div v-if="visible">
    <div
      v-for="item in items"
      :key="item.id"
      v-memo="[item.data]"
    >
      {{ item.data }}
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const visible = ref(true)
const items = ref([...])

// 组件卸载后，v-memo 的缓存会被正确清理
// 3.5.31 修复：分离后的 v-memo vnode 不再在 unmount 后访问
</script>
```

## 六、Transition 与 TransitionGroup 优化

### 1. HMR 更新时跳过 Transition Enter Guard（3.5.31）

```vue
<template>
  <Transition name="fade">
    <!-- HMR 热更新时，transition 的 enter 动画被正确跳过 -->
    <div v-if="show" key="content">
      <h1>{{ title }}</h1>
    </div>
  </Transition>
</template>
```

### 2. Keep-Alive 中空闲 Transition Hooks 跳过（3.5.35）

```vue
<template>
  <KeepAlive>
    <component :is="currentView" />
  </KeepAlive>
</template>

<script setup>
import { shallowRef } from 'vue'

const currentView = shallowRef(ComponentA)

// 3.5.35 优化：Keep-Alive 移动组件时跳过空闲的 persisted transition hooks
// 减少不必要的 hook 执行
</script>
```

## 七、TypeScript 类型优化

### 1. `customRef` 支持不同 getter/setter 类型（3.5.32）

```typescript
// ✅ 3.5.32+：customRef 的 getter 和 setter 可以有不同的类型
function myCustomRef<T, U>(): Ref<T> {
  return customRef<T, U>((track, trigger) => ({
    get() {
      track()
      return value as T
    },
    set(newValue: U) {
      value = transform(newValue) as T
      trigger()
    }
  }))
}
```

### 2. `shallowReactive` 使用私有 branding 防止类型泄漏（3.5.32）

```typescript
import { shallowReactive } from 'vue'

// 3.5.32 修复：shallowReactive 的 branding 标记不会泄漏到联合类型中
type User = { name: string }
const user = shallowReactive({ name: 'Alice' })

// ✅ 修复后，user 的类型就是 User，不会包含额外的内部标记
```

## 注意事项

| 要点 | 说明 |
|------|------|
| **Vue 版本** | 以上优化适用于 Vue 3.5.31+，建议升级到最新的 3.5.35 |
| **React 性能优化** | `reactive()` 缓存优化不需要修改代码，升级即受益 |
| **SSR 相关** | `ssrRenderList` 迭代器优化仅影响服务端渲染 |
| **Teleport 问题** | 涉及动态目标 Teleport 的应用应至少升级到 3.5.32 |
| **V-Memo 缓存** | 确保 v-memo 的依赖数组正确包含所有可能变化的属性 |
| **Transition** | HMR 相关修复开发体验提升，不影响生产环境 |
