---
name: vue3-35-features
description: Vue 3.5+ 新特性详解：响应式优化、useId、watch 改进、Props 解构与开发体验提升
tags: [vue3, vue35, reactivity, performance, composition-api]
---

## 概述

Vue 3.5（代号 "Sentinel"，2025 年发布）带来了响应式系统重构和大量 DX 改进。当前最新版本 3.5.35（2026 年 5 月）。以下是核心变化。

## 1. 响应式系统重构（Reactivity 6.0）

Vue 3.5 对 `@vue/reactivity` 进行了重写，核心改进：

### 细粒度响应式依赖追踪

```typescript
import { ref, computed, watchEffect } from 'vue'

// 3.5 之前：所有依赖在 effect 运行时一次性收集
// 3.5 之后：按需收集，条件分支中的未使用依赖不会被追踪

const show = ref(true)
const name = ref('Alice')
const age = ref(30)

// 3.5+ 更智能的依赖追踪
watchEffect(() => {
  if (show.value) {
    console.log(name.value)  // 只追踪 show 和 name
  } else {
    console.log(age.value)   // 只追踪 show 和 age
  }
  // 3.5 之前：name 和 age 都会被追踪
  // 3.5 之后：只追踪实际执行路径中的依赖
})
```

### 优化的数组响应式

```typescript
import { reactive } from 'vue'

const list = reactive([1, 2, 3, 4, 5])

// 3.5 显著提升数组操作的性能
list.push(6)            // ✅ 更快
list.splice(0, 1)       // ✅ 更快
list[0] = 100           // ✅ 更快

// 长数组遍历性能提升 2-3 倍
list.forEach(item => console.log(item))
list.map(item => item * 2)
```

### Proxy 优化

```typescript
// 3.5 减少了不必要的 Proxy 包装
// 深层嵌套对象不再全部包装为 Proxy，只在访问时惰性包装

const deep = reactive({
  level1: {
    level2: {
      level3: 'deep value'
    }
  }
})

// 3.5 之前：创建时递归包装所有层级 → 起始慢
// 3.5 之后：访问时才包装 → 起始快 40%
const val = deep.level1.level2.level3
```

## 2. Props 解构（Props Destructure）

Vue 3.5 正式支持在 `<script setup>` 中解构 props，且保持响应性：

```vue
<script setup lang="ts">
// 3.5+ props 解构（不再需要 toRefs）
const { title, count = 0, items = [] } = defineProps<{
  title: string
  count?: number
  items?: string[]
}>()

// 解构后的变量保持响应性
watchEffect(() => {
  console.log(title.value)  // 自动 .value 访问
  console.log(count.value)
})

// ❌ 注意：不能重新赋值
// title = 'new title'  ← 编译错误！
</script>

<template>
  <h1>{{ title }}</h1>
  <p>{{ count }}</p>
</template>
```

### 默认值处理

```vue
<script setup lang="ts">
// 3.5+ 支持解构默认值
const {
  name,
  // withDefaults 方式依然可用，但解构默认值更简洁
  pageSize = 20,
  immediate = false,
  tags = () => []  // 对象/数组用函数返回
} = defineProps<{
  name: string
  pageSize?: number
  immediate?: boolean
  tags?: string[]
}>()
</script>
```

## 3. useId() — 唯一 ID 生成

```vue
<script setup lang="ts">
import { useId } from 'vue'

// 在服务端渲染（SSR）中生成稳定唯一的 ID
// 同一组件在客户端和服务端生成的 ID 一致
const inputId = useId()
const descId = useId()
</script>

<template>
  <div>
    <label :for="inputId">用户名</label>
    <input
      :id="inputId"
      type="text"
      :aria-describedby="descId"
    />
    <p :id="descId">请输入您的用户名，至少 3 个字符</p>
  </div>
</template>
```

## 4. watch / watchEffect 改进

### 一次性监听（watchOnce）

```typescript
import { ref, watchOnce } from 'vue'

const count = ref(0)

// watchOnce：只监听第一次变化后自动停止
watchOnce(count, (newVal) => {
  console.log('count 首次变化:', newVal)
  // 执行一次后自动取消监听
})
```

### watch 支持 Promise

```typescript
import { ref, watch } from 'vue'

const userId = ref(1)

// watch 回调支持 async/await
watch(userId, async (newId, oldId) => {
  // 自动处理竞态条件
  const data = await fetchUser(newId)
  // 如果再次变化，前一个请求的后续操作被忽略
  if (userId.value === newId) {  // 确保数据是最新的
    userData.value = data
  }
})
```

### watchDepth 选项

```typescript
import { reactive, watch } from 'vue'

const state = reactive({
  user: {
    profile: {
      name: 'Alice',
      settings: { theme: 'dark' }
    }
  }
})

// 3.5+ deep 优化：支持指定深度层级
watch(
  () => state.user,
  (newVal) => {
    console.log('user 变化', newVal)
  },
  { deep: 2 }  // 只监听 2 层深度
  // 之前只能 deep: true（全量深监听）
)

// 只监听特定路径
watch(
  () => state.user.profile.settings.theme,
  (newTheme) => {
    console.log('主题变化:', newTheme)
  }
  // 无需 deep，自动精确追踪
)
```

## 5. onWatcherCleanup — 清理回调

```vue
<script setup lang="ts">
import { ref, watch, onWatcherCleanup } from 'vue'

const keyword = ref('')

watch(keyword, (newKeyword) => {
  const controller = new AbortController()

  // 注册清理函数（3.5+）
  // 在 watch 重新执行或组件卸载时自动调用
  onWatcherCleanup(() => {
    controller.abort()
    console.log('请求已取消')
  })

  fetch(`/api/search?q=${newKeyword}`, {
    signal: controller.signal
  }).then(res => res.json())
    .then(data => results.value = data)
})
</script>
```

## 6. 模板编译优化

### 惰性解析（Lazy Hydration）

```vue
<script setup lang="ts">
import { defineAsyncComponent, lazyHydrate } from 'vue'

// 3.5+ 组件支持懒激活
// 仅在组件进入可视区域时才激活（hydration）
const HeavyComponent = lazyHydrate(() => import('./HeavyComponent.vue'))

// 配置选项
const LazyChart = lazyHydrate(() => import('./Chart.vue'), {
  hydrateOn: {
    idle: true,      // 浏览器空闲时激活
    visible: true,   // 进入视口时激活
    interaction: ['click', 'focus']  // 用户交互时激活
  }
})
</script>

<template>
  <!-- 仅在需要时 hydrate，减少首屏 JS 执行 -->
  <LazyChart />
</template>
```

### 静态节点提升优化

```vue
<template>
  <!-- 3.5+ 提升更多场景的静态节点 -->
  <div class="static-wrapper">
    <h1 class="title">{{ dynamicTitle }}</h1>
    <!-- 静态部分完全不参与更新 -->
    <p class="static-description">
      这是一段纯静态文本，3.5 会将其彻底提升。
    </p>
  </div>
</template>
```

## 7. 编译宏增强

### defineModel 改进

```vue
<script setup lang="ts">
// 3.5+ defineModel 支持多个 v-model
const model = defineModel<string>()        // v-model
const visible = defineModel<boolean>('visible')  // v-model:visible
const label = defineModel<string>('label', { defaultValue: '默认标签' })

// 3.5+ 支持类型转换
const count = defineModel<number>('count', {
  type: Number,
  // 自动处理数字输入框的 string → number 转换
  converter: (val: string) => Number(val)
})

// 3.5+ 支持修饰符
const [modelValue, modelModifiers] = defineModel<string>({
  // 自定义修饰符逻辑
  get(val: string) { return val.trim().toUpperCase() },
  set(val: string) { return val.trim() }
})
</script>
```

### useTemplateRef

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

// 3.5+ 类型安全的模板引用（替代 ref(null)）
const inputRef = useTemplateRef<HTMLInputElement>('input')

onMounted(() => {
  // 类型安全，无需断言
  inputRef.value?.focus()
})
</script>

<template>
  <input ref="inputRef" type="text" />
</template>
```

## 8. 更新建议

| 功能 | 推荐等级 | 迁移说明 |
|------|---------|---------|
| Props 解构 | ⭐⭐⭐ 强烈推荐 | 替代 `toRefs`，注意不能重新赋值 |
| useId() | ⭐⭐⭐ 强烈推荐 | SSR 无障碍必备 |
| watchOnce | ⭐⭐ 推荐 | 简化一次性监听逻辑 |
| onWatcherCleanup | ⭐⭐ 推荐 | 替代 `watchHandle()` 清理模式 |
| lazyHydrate | ⭐⭐ 推荐 | 大型 SSR 项目性能提升 |
| useTemplateRef | ⭐⭐⭐ 强烈推荐 | 类型安全的模板引用 |
| defineModel 改进 | ⭐⭐⭐ 强烈推荐 | 组件双向绑定更简洁 |

## 注意事项

1. **Props 解构不可重新赋值**：解构后的 props 是只读的，尝试赋值会触发编译警告
2. **useTemplateRef 仅在 3.5+**：3.4 及以下版本仍用 `ref<HTMLElement | null>(null)`
3. **响应式重构向后兼容**：绝大多数 3.4 代码无需修改即可升级
4. **watchOnce 需要显式导入**：不属于全局 API，从 `vue` 包中导入
5. **lazyHydrate 仅 SSR 模式有效**：纯客户端渲染（SPA）无效果
6. **升级路径**：`npm update vue@latest`，同时升级 `@vue/compiler-sfc` 等配套包
