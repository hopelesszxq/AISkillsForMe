---
name: vue3-define-model-patterns
description: Vue3 defineModel 双向绑定高阶模式：多 v-model、修饰符、类型安全、组件组合
tags: [vue3, define-model, v-model, component, typescript]
---

## 概述

Vue 3.4+ 引入的 `defineModel()` 彻底简化了自定义组件的双向绑定。相比 Vue 3.3 的 `defineProps` + `defineEmits` 手动模式，`defineModel` 大幅减少了样板代码，并原生支持 TypeScript 类型推导。

> 最低版本要求：Vue 3.4+（推荐 3.5+ 以获得修饰符支持）

## 1. 基础用法

### 组件中定义

```vue
<!-- MyInput.vue -->
<script setup lang="ts">
// 自动生成 props.modelValue + emit('update:modelValue')
const model = defineModel<string>()
</script>

<template>
  <input :value="model" @input="model = ($event.target as HTMLInputElement).value" />
</template>
```

### 父组件使用

```vue
<template>
  <MyInput v-model="username" />
</template>

<script setup lang="ts">
import { ref } from 'vue'
const username = ref('')
</script>
```

等效于 Vue 3.3 写法：
```vue
<script setup lang="ts">
const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()
const model = computed({
  get: () => props.modelValue,
  set: (val) => emit('update:modelValue', val)
})
</script>
```

**减少约 60% 的样板代码。**

## 2. 多 v-model 绑定

一个组件支持多个双向绑定参数，通过别名区分。

```vue
<!-- UserForm.vue -->
<script setup lang="ts">
const name = defineModel<string>('name', { required: true })
const age = defineModel<number>('age', { default: 0 })
const email = defineModel<string>('email')
</script>

<template>
  <input :value="name" @input="name = ($event.target as HTMLInputElement).value" />
  <input type="number" :value="age" @input="age = Number(($event.target as HTMLInputElement).value)" />
  <input :value="email" @input="email = ($event.target as HTMLInputElement).value" />
</template>
```

### 父组件

```vue
<UserForm v-model:name="userName" v-model:age="userAge" v-model:email="userEmail" />
```

## 3. 修饰符（Modifiers）支持

Vue 3.5+ 增强了对 `defineModel` 修饰符的完整支持。

### 内置修饰符

```vue
<!-- 组件内部无需特殊处理 -->
<InputTrim v-model.trim="text" />
```

### 自定义修饰符

```vue
<!-- PriceInput.vue — 将数字格式化为保留两位小数 -->
<script setup lang="ts">
import { computed } from 'vue'

// 第二个参数是修饰符对象
const [model, modifiers] = defineModel<string>('price', {
  set(val: string) {
    // 自定义 setter，在写入前处理
    if (modifiers.precise) {
      return Number(val).toFixed(2)
    }
    return val
  }
})

// 读取修饰符
console.log(modifiers.precise) // true/false
</script>

<template>
  <input :value="model" @input="model = ($event.target as HTMLInputElement).value" />
</template>
```

### 父组件使用修饰符

```vue
<PriceInput v-model:price.precise="price" />
```

## 4. 类型安全的泛型组件

利用 TypeScript 泛型实现类型安全的 `defineModel`。

```vue
<!-- SelectableList.vue -->
<script setup lang="ts" generic="T extends { id: string | number; label: string }">
const selected = defineModel<T | null>('selected', { default: null })

const items = defineProps<{
  options: T[]
  multiple?: boolean
}>()

function select(item: T) {
  selected.value = item
}
</script>

<template>
  <div class="list">
    <div
      v-for="item in options"
      :key="item.id"
      :class="{ active: selected?.id === item.id }"
      @click="select(item)"
    >
      {{ item.label }}
    </div>
  </div>
</template>
```

### 使用

```vue
<script setup lang="ts">
interface User { id: string; name: string; email: string }

const options = ref<User[]>([
  { id: '1', name: 'Alice', email: 'alice@example.com' },
  { id: '2', name: 'Bob', email: 'bob@example.com' }
])

const currentUser = ref<User | null>(null)
</script>

<template>
  <SelectableList :options="options" v-model:selected="currentUser" />
</template>
```

## 5. 与 Pinia 联动

```vue
<!-- UserProfile.vue -->
<script setup lang="ts">
import { useUserStore } from '@/stores/user'

const userStore = useUserStore()

// defineModel 可以直接绑定到 store 状态
const username = defineModel<string>('username', {
  // 从 store 初始化
  get() { return userStore.username },
  // 更新时同步到 store
  set(val) { userStore.updateUsername(val) }
})
</script>
```

## 6. 嵌套组件传递（层层透传）

```vue
<!-- GrandParent.vue -->
<Parent v-model="value" />

<!-- Parent.vue — 直接透传给子组件 -->
<script setup lang="ts">
const model = defineModel<string>()
</script>

<template>
  <Child v-model="model" />
</template>

<!-- Child.vue — 最终使用 -->
<script setup lang="ts">
const model = defineModel<string>()
</script>
```

## 7. 与 v-bind.sync 对比（Vue 2 迁移）

| 特性 | Vue 2 `.sync` | Vue 3 `defineModel` |
|------|--------------|---------------------|
| 声明方式 | `update:xxx` emit | `defineModel('xxx')` |
| 多值绑定 | 多个 `.sync` 修饰符 | 多个 `v-model:xxx` |
| TypeScript | 无类型推导 | 完整类型推导 |
| 修饰符 | 不支持 | 支持（Vue 3.5+） |
| 计算属性衍生 | 需手动 computed | getter/setter 原生支持 |

## 8. 注意事项

1. **必需属性**：`defineModel('name', { required: true })` 当父组件未传对应 v-model 时会警告
2. **避免直接引用 props**：`defineModel` 返回的可写 ref 经过特殊处理，不要通过 `defineProps` 重复声明同名属性
3. **修饰符与 setter 冲突**：当同时使用 `modifiers` 和自定义 `set` 时，set 中接收的是原始值（修饰符尚未应用），需要在 setter 内自行处理
4. **深层响应式**：`defineModel` 默认是深层响应式，对于大型对象的双向绑定，考虑使用 `shallowRef` + 手动触发
5. **组合式函数内使用**：`defineModel` 只能在 `<script setup>` 中使用，不能在普通 Composable 内调用。如需 Composable 封装，使用 `computed` + `emit` 模式替代
6. **过渡动画冲突**：v-model 的更新是同步的，如果配合 `<Transition>` 使用，注意用 `nextTick` 确保 DOM 更新完成
