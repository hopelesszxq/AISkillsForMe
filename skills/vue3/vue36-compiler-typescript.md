---
name: vue36-compiler-typescript
description: Vue 3.6 Compiler Macros 与 TypeScript 增强：$infer()、useTemplateRef()、defineModel 修饰符、具名插槽类型推导
tags: [vue3, vue36, typescript, compiler-macros, template-ref, define-model]
---

## 概述

Vue 3.6（当前 Beta 阶段）在 Vapor Mode 之外，还引入了大量 **Compiler Macros 和 TypeScript 类型系统的增强**。这些改进对数百万行 SFC 应用的开发体验有直接影响，且无需等待 Vapor Mode 正式发布即可使用。

> 关于 Vapor Mode 详情请参考 `vue36-vapor-mode.md`，本文聚焦非 Vapor 的增强功能。

## 一、`$infer()` — 运行时类型推导宏

### 功能说明

`$infer()` 是 Vue 3.6 新增的编译时宏，用于从组件的运行时声明推导出 TypeScript 类型，避免重复维护类型定义。

```vue
<script setup lang="ts">
import { $infer } from 'vue'

// 定义组件 Props（运行时声明）
const props = defineProps({
  title: { type: String, required: true },
  count: { type: Number, default: 0 },
  items: { type: Array as PropType<string[]>, default: () => [] }
})

// 从运行时声明推导出 Props 类型
type Props = $infer<typeof defineProps>
// 推导结果: { title: string; count?: number; items?: string[] }

// 推导 Emits 类型
type Emits = $infer<typeof defineEmits<{
  (e: 'update:count', val: number): void
  (e: 'delete', id: string): void
}>>
// 推导结果: { 'update:count': [val: number]; 'delete': [id: string] }

// 推导 slots 类型
type Slots = $infer<typeof defineSlots<{
  default(props: { item: string; index: number }): any
  header(): any
}>>
</script>
```

### 应用场景

```vue
<script setup lang="ts">
import { $infer } from 'vue'

const props = defineProps({
  results: { type: Array as PropType<{ id: number; name: string }[]>, required: true }
})

// 推导出单条记录类型 —— 避免手动写两次泛型
type ResultItem = $infer<typeof defineProps>['results'][number]
// 等价于: { id: number; name: string }

function processItem(item: ResultItem) {
  console.log(item.id, item.name)
}
</script>
```

## 二、`useTemplateRef()` — 类型安全的模板引用

### 功能说明

Vue 3.6 引入 `useTemplateRef`（从 VueUse 移植并改进），替代旧的 `ref(null) as Ref<HTMLElement | null>` 模式，**自动推导 ref 类型**。

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

// ❌ Vue 3.5 及之前
const inputRef = ref<HTMLInputElement | null>(null)

// ✅ Vue 3.6 — 自动类型推导
const inputRef = useTemplateRef('inputRef')
const modalRef = useTemplateRef('modalRef')
const chartRef = useTemplateRef('chartRef')
</script>

<template>
  <input ref="inputRef" type="text" />
  <Modal ref="modalRef" />
  <Chart ref="chartRef" />
</template>
```

### 与 defineExpose 配合

```vue
<!-- Child.vue -->
<script setup lang="ts">
function validate(): boolean { /* ... */ }
function reset(): void { /* ... */ }

defineExpose({ validate, reset })
</script>

<!-- Parent.vue -->
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import Child from './Child.vue'

// 类型自动推导为 Child 的 expose 类型
const childRef = useTemplateRef<InstanceType<typeof Child>>('childRef')

function handleSubmit() {
  if (childRef.value?.validate()) {
    // childRef.value 类型安全
  }
}
</script>

<template>
  <Child ref="childRef" />
</template>
```

## 三、`defineModel()` 增强

### 3.1 修饰符支持（正式稳定）

Vue 3.5 引入了 defineModel 修饰符实验支持，3.6 正式稳定并增强：

```vue
<!-- 父组件 -->
<MyInput v-model:value.trim="name" />

<!-- 子组件 -->
<script setup lang="ts">
// 自动处理 trim 修饰符
const [value, modifiers] = defineModel<string>('value', {
  set(val: string) {
    // modifiers.trim 为 true
    if (modifiers.trim) {
      return val.trim()
    }
    return val
  }
})
</script>
```

### 3.2 多个 v-model 的增强

```vue
<script setup lang="ts">
// Vue 3.6 支持一次性声明多个 v-model
const [search, setSearch] = useVModel(props, 'search')  // 组合式 API 模式

// 或使用 defineModel 一次声明
const search = defineModel<string>('search')
const page = defineModel<number>('page')
const pageSize = defineModel<number>('pageSize', { default: 20 })
</script>

<template>
  <input v-model="search.value" />
  <button @click="page.value++">Next</button>
</template>
```

### 3.3 类型安全的复杂模型

```vue
<script setup lang="ts">
interface Filters {
  category: string
  minPrice: number
  maxPrice: number
  inStock: boolean
}

// defineModel 完整支持泛型
const filters = defineModel<Filters>('filters', {
  required: true,
  deep: true  // 深度响应式
})

// 直接修改对象属性
filters.value.inStock = true
</script>
```

## 四、具名插槽类型改进

### 4.1 编译时插槽类型检查

Vue 3.6 改进了具名插槽的类型推导：

```vue
<!-- List.vue -->
<script setup lang="ts">
defineSlots<{
  default(props: { item: string; index: number }): any
  empty(): any           // 没有参数的插槽
  header(props: { title: string }): any
  footer(props: { total: number }): any
}>()
</script>

<template>
  <slot name="header" :title="title" />
  <slot v-for="..." :item="item" :index="index" />
  <slot name="empty" v-if="!items.length" />
  <slot name="footer" :total="items.length" />
</template>
```

### 4.2 父组件类型安全

```vue
<template>
  <List :items="items">
    <!-- 类型错误！'item' 是 string，但 .toFixed 是 number 方法 -->
    <template #default="{ item }">
      {{ item.toFixed(2) }}
    </template>

    <!-- ✅ 正确：item 类型为 string -->
    <template #default="{ item }">
      {{ item.toUpperCase() }}
    </template>
  </List>
</template>
```

## 五、其他编译器宏改进

### 5.1 `defineProp()` 替代 `defineProps`（实验）

Vue 3.6 引入了实验性的 `defineProp()`，用于声明单个 prop：

```vue
<script setup lang="ts">
const title = defineProp<string>('title', { required: true })
const count = defineProp<number>('count', { default: 0 })

// 等价于：
// const props = defineProps({ title: { type: String, required: true }, count: { type: Number, default: 0 } })
// 但类型推导更直接
</script>
```

### 5.2 `defineEmits()` 回调模式增强

```vue
<script setup lang="ts">
// 回调模式（推荐）— 类型更简洁
const emit = defineEmits<{
  'update:search': [value: string]
  'select': [id: number, item: { name: string }]
}>()

// 调用时完全类型安全
emit('update:search', 'vue')    // ✅
emit('select', 1, { name: 'test' })  // ✅
emit('select', '1')  // ❌ 类型错误
</script>
```

## 注意事项

1. **`$infer()` 是编译时宏**：仅在 `<script setup>` 中可用，编译后消失，不增加运行时体积
2. **`useTemplateRef()` 需要 Vue 3.6+**：3.5 及以下版本不可用，可用 VueUse 的 `useTemplateRef` 兼容
3. **`defineProp()` 目前实验性**：3.6-beta 阶段，API 可能微调，生产环境建议继续使用 `defineProps`
4. **插槽类型检测依赖 IDE 支持**：需要 VSCode Volar 插件 v2.4+ 才能生效
5. **Vapor Mode 与编译器宏的关系**：Vapor Mode 使用相同的编译器宏系统，迁移时无需改动模板语法
