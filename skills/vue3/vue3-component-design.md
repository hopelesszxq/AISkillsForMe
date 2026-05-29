---
name: vue3-component-design
description: Vue3 组件设计模式：v-model 自定义、插槽组合、函数式组件、渲染函数与高阶组件
tags: [vue3, component, design-pattern, v-model, slots, composable, frontend]
---

## 概述

Vue3 的组合式 API 和 Composition API 为组件设计提供了更强的灵活性。本文覆盖 Vue3 中从基础到进阶的组件设计模式，包括自定义 v-model、插槽组合技巧、受控/非受控组件、渲染函数、Teleport、动态组件等实战模式。

## 1. 自定义 v-model（双向绑定）

### 基础用法（Vue3 多 v-model）

```vue
<script setup lang="ts">
// ChildComponent.vue
const props = defineProps<{
  modelValue: string
  visible?: boolean
}>()

const emit = defineEmits<{
  'update:modelValue': [value: string]
  'update:visible': [value: boolean]
}>()
</script>

<template>
  <div v-if="visible">
    <input
      :value="modelValue"
      @input="emit('update:modelValue', ($event.target as HTMLInputElement).value)"
    />
    <button @click="emit('update:visible', false)">关闭</button>
  </div>
</template>
```

```vue
<!-- ParentComponent.vue -->
<script setup lang="ts">
const searchText = ref('')
const dialogVisible = ref(true)
</script>

<template>
  <ChildComponent
    v-model="searchText"
    v-model:visible="dialogVisible"
  />
</template>
```

### 自定义修饰符

```vue
<script setup lang="ts">
// InputWithTrim.vue
const props = defineProps<{
  modelValue: string
  modelModifiers?: { trim?: boolean; uppercase?: boolean }
}>()

const emit = defineEmits<{
  'update:modelValue': [value: string]
}>()

function onInput(e: Event) {
  let value = (e.target as HTMLInputElement).value
  if (props.modelModifiers?.trim) value = value.trim()
  if (props.modelModifiers?.uppercase) value = value.toUpperCase()
  emit('update:modelValue', value)
}
</script>

<template>
  <input :value="modelValue" @input="onInput" />
</template>
```

```vue
<template>
  <InputWithTrim v-model.trim.uppercase="username" />
</template>
```

## 2. 插槽（Slots）组合模式

### 作用域插槽：提供数据给父组件渲染

```vue
<!-- DataTable.vue -->
<script setup lang="ts">
defineProps<{
  data: any[]
  columns: { key: string; title: string }[]
}>()
</script>

<template>
  <table>
    <thead>
      <tr>
        <th v-for="col in columns" :key="col.key">{{ col.title }}</th>
        <th v-if="$slots.actions">操作</th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="(row, index) in data" :key="index">
        <td v-for="col in columns" :key="col.key">
          <!-- 如果父组件提供了该列的插槽，用插槽渲染，否则显示原始值 -->
          <slot :name="`cell-${col.key}`" :row="row" :value="row[col.key]">
            {{ row[col.key] }}
          </slot>
        </td>
        <td v-if="$slots.actions">
          <slot name="actions" :row="row" />
        </td>
      </tr>
    </tbody>
  </table>
</template>
```

```vue
<!-- 使用 DataTable -->
<script setup lang="ts">
const users = ref([
  { id: 1, name: '张三', email: 'zhangsan@example.com', role: 'admin' },
  { id: 2, name: '李四', email: 'lisi@example.com', role: 'user' },
])

const columns = [
  { key: 'name', title: '姓名' },
  { key: 'email', title: '邮箱' },
  { key: 'role', title: '角色' },
]

function deleteUser(user: any) {
  console.log('删除用户:', user.id)
}
</script>

<template>
  <DataTable :data="users" :columns="columns">
    <template #cell-email="{ value }">
      <a :href="`mailto:${value}`">{{ value }}</a>
    </template>
    <template #cell-role="{ value }">
      <el-tag :type="value === 'admin' ? 'danger' : 'info'">{{ value }}</el-tag>
    </template>
    <template #actions="{ row }">
      <button @click="deleteUser(row)">删除</button>
    </template>
  </DataTable>
</template>
```

### 插槽命名规范与类型安全

```vue
<script setup lang="ts">
// 声明插槽类型（Vue 3.3+）

defineSlots<{
  // 具名插槽
  default(props: { item: any }): any
  header(): any
  footer(props: { total: number }): any
  // 动态插槽
  [key: `cell-${string}`](props: { row: any; value: any }): any
}>()
</script>
```

## 3. 受控 vs 非受控组件模式

### 非受控（内部状态）

```vue
<script setup lang="ts">
// Counter.vue — 非受控（内部 state 驱动）
import { ref } from 'vue'

const props = withDefaults(defineProps<{
  initialValue?: number
}>(), { initialValue: 0 })

const count = ref(props.initialValue)
const emit = defineEmits<{
  change: [value: number]
}>()

function increment() {
  count.value++
  emit('change', count.value)
}
</script>

<template>
  <button @click="increment">计数: {{ count }}</button>
</template>
```

### 受控（外部 state 驱动）

```vue
<script setup lang="ts">
// ControlledCounter.vue — 受控
const props = defineProps<{
  modelValue: number
}>()

const emit = defineEmits<{
  'update:modelValue': [value: number]
}>()

function increment() {
  emit('update:modelValue', props.modelValue + 1)
}
</script>

<template>
  <button @click="increment">计数: {{ modelValue }}</button>
</template>
```

### 两者兼顾（推荐模式）

```vue
<script setup lang="ts">
// FlexibleCounter.vue
import { ref, computed } from 'vue'

const props = withDefaults(defineProps<{
  modelValue?: number
  defaultValue?: number
}>(), { defaultValue: 0 })

const emit = defineEmits<{
  'update:modelValue': [value: number]
}>()

// 使用 useVModel 简化（VueUse）
import { useVModel } from '@vueuse/core'
const count = useVModel(props, 'modelValue', emit, {
  passive: true,        // 不主动 emit（性能优化）
  defaultValue: props.defaultValue,
})

// 或者手动实现
// const internalCount = ref(props.modelValue ?? props.defaultValue)
// const count = computed({
//   get: () => props.modelValue !== undefined ? props.modelValue : internalCount.value,
//   set: (val) => {
//     if (props.modelValue !== undefined) {
//       emit('update:modelValue', val)
//     } else {
//       internalCount.value = val
//     }
//   },
// })
</script>

<template>
  <button @click="count++">计数: {{ count }}</button>
</template>
```

## 4. 组合式函数（Composables）复用模式

### 异步数据获取

```typescript
// composables/useAsyncData.ts
import { ref, watchEffect, type Ref } from 'vue'

interface UseAsyncDataOptions<T> {
  immediate?: boolean
  onSuccess?: (data: T) => void
  onError?: (error: Error) => void
}

export function useAsyncData<T>(
  fetcher: () => Promise<T>,
  options: UseAsyncDataOptions<T> = {}
) {
  const data = ref<T | null>(null) as Ref<T | null>
  const loading = ref(false)
  const error = ref<Error | null>(null)

  async function execute() {
    loading.value = true
    error.value = null
    try {
      data.value = await fetcher()
      options.onSuccess?.(data.value)
    } catch (e) {
      error.value = e as Error
      options.onError?.(e as Error)
    } finally {
      loading.value = false
    }
  }

  if (options.immediate !== false) {
    watchEffect(() => { execute() })
  }

  return { data, loading, error, execute, refresh: execute }
}

// 使用
// const { data: users, loading, error } = useAsyncData(() => api.getUsers())
```

### 浏览器 API 封装

```typescript
// composables/useEventListener.ts
import { onMounted, onUnmounted, type Ref } from 'vue'

export function useEventListener<K extends keyof WindowEventMap>(
  target: EventTarget | Ref<EventTarget | null>,
  event: K,
  handler: (e: WindowEventMap[K]) => void
) {
  onMounted(() => {
    const el = target instanceof EventTarget ? target : target.value
    el?.addEventListener(event, handler as EventListener)
  })
  onUnmounted(() => {
    const el = target instanceof EventTarget ? target : target.value
    el?.removeEventListener(event, handler as EventListener)
  })
}

// 使用
// useEventListener(window, 'resize', () => { width.value = window.innerWidth })
```

## 5. 渲染函数（Render Function）与 JSX

```tsx
// render-table.tsx
import { defineComponent, h } from 'vue'

interface Column {
  key: string
  title: string
  width?: number
}

interface Props {
  data: Record<string, any>[]
  columns: Column[]
  striped?: boolean
}

export default defineComponent({
  props: {
    data: { type: Array, required: true },
    columns: { type: Array, required: true },
    striped: Boolean,
  },
  emits: ['row-click'],
  setup(props: Props, { emit, slots }) {
    return () => (
      <table class={['data-table', { striped: props.striped }]}>
        <thead>
          <tr>
            {props.columns.map(col => (
              <th key={col.key} style={{ width: col.width + 'px' }}>
                {col.title}
              </th>
            ))}
          </tr>
        </thead>
        <tbody>
          {props.data.map((row, rowIndex) => (
            <tr
              key={rowIndex}
              onClick={() => emit('row-click', row)}
              class={{ 'row-hover': true }}
            >
              {props.columns.map(col => (
                <td key={col.key}>
                  {/* 支持作用域插槽 */}
                  {slots[`cell-${col.key}`]
                    ? slots[`cell-${col.key}`]({ row, value: row[col.key] })
                    : row[col.key]
                  }
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
    )
  },
})
```

## 6. Teleport 与 Portal 模式

```vue
<!-- ConfirmDialog.vue -->
<script setup lang="ts">
const props = withDefaults(defineProps<{
  visible: boolean
  title?: string
  /** 挂载目标，默认 body */
  to?: string | HTMLElement
}>(), {
  to: 'body',
})

const emit = defineEmits<{
  confirm: []
  cancel: []
}>()
</script>

<template>
  <Teleport :to="to">
    <Transition name="fade">
      <div v-if="visible" class="overlay" @click.self="emit('cancel')">
        <div class="dialog">
          <h3>{{ title || '确认' }}</h3>
          <slot />
          <div class="actions">
            <button class="btn-cancel" @click="emit('cancel')">取消</button>
            <button class="btn-confirm" @click="emit('confirm')">确定</button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>
```

## 7. 动态组件与 KeepAlive

```vue
<script setup lang="ts">
import { shallowRef, type Component } from 'vue'

import UserList from './UserList.vue'
import UserForm from './UserForm.vue'
import UserDetail from './UserDetail.vue'

const TABS: Record<string, Component> = {
  list: UserList,
  form: UserForm,
  detail: UserDetail,
}

const currentTab = ref('list')

// 缓存标签页状态
</script>

<template>
  <div class="tabs">
    <button
      v-for="(_, key) in TABS"
      :key="key"
      :class="{ active: currentTab === key }"
      @click="currentTab = key"
    >
      {{ key }}
    </button>
  </div>

  <KeepAlive :include="['UserList', 'UserForm']">
    <component :is="TABS[currentTab]" />
  </KeepAlive>
</template>
```

## 8. 错误边界组件

```vue
<!-- ErrorBoundary.vue -->
<script setup lang="ts">
import { ref, onErrorCaptured, type Component } from 'vue'

const error = ref<Error | null>(null)
const errorInfo = ref<string>('')

onErrorCaptured((err, instance, info) => {
  error.value = err as Error
  errorInfo.value = info
  // 阻止错误继续传播
  return false
})

function reset() {
  error.value = null
  errorInfo.value = ''
}
</script>

<template>
  <slot v-if="!error" />
  <div v-else class="error-boundary">
    <h3>组件渲染异常</h3>
    <p>{{ error.message }}</p>
    <pre v-if="errorInfo">{{ errorInfo }}</pre>
    <button @click="reset">重试</button>
  </div>
</template>
```

## 注意事项

1. **v-model 参数名**：Vue3 中 `v-model` 相当于 `:modelValue` + `@update:modelValue`，允许多个 v-model
2. **v-model 修饰符**：通过 `props.modelModifiers` 获取，自定义修饰符在组件内处理
3. **组件设计原则**：单一职责、接口清晰、Props 扁平化、避免深层嵌套传递
4. **性能优化**：`v-once` 静态内容标记、`shallowRef` 大对象、`defineAsyncComponent` 懒加载、`computed` 缓存
5. **类型安全**：Vue 3.3+ 推荐使用 `defineSlots`、`defineEmits` 声明类型，使用 `generics` 属性支持泛型组件
6. **泛型组件**（Vue 3.3+）：
   ```vue
   <script setup lang="ts" generics="T extends { id: number; name: string }">
   defineProps<{ data: T[] }>()
   </script>
   ```
7. **Provide/Inject 模式**：适用跨多层组件传递，但不要滥用（业务状态推荐 Pinia）
8. **受控 vs 非受控**：优先提供两种模式（通过 `modelValue` 是否传入判断），提高组件通用性
9. **Teleport 陷阱**：Teleport 内容不受父组件样式作用域限制，CSS 需要使用 `:deep()` 穿透
10. **组件命名**：遵循 Vue 风格指南——多单词组件名（避免与 HTML 标签冲突）、PascalCase
