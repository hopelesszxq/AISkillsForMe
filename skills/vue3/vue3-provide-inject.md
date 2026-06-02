---
name: vue3-provide-inject
description: Vue3 provide/inject 深度模式 — 跨层级状态共享与依赖注入
tags: [vue3, provide, inject, composition-api, frontend]
---

## provide/inject 适用场景

| 场景 | 推荐方案 | 不推荐方案 |
|---|---|---|
| 主题/语言设置（全局） | provide/inject + reactive | Pinia（大材小用） |
| 表单上下文（嵌套表单） | provide/inject + readonly | Props 逐层透传 |
| 组件库根组件传递配置 | provide/inject + Symbol | EventBus |
| 多级路由懒加载共享状态 | provide/inject + shallowRef | Vuex/Pinia |
| 全局数据（多个页面） | Pinia | provide/inject |

## 基础用法

### 祖先组件提供

```typescript
// App.vue — 全局主题提供者
<script setup lang="ts">
import { ref, provide, readonly } from 'vue';

const theme = ref<'light' | 'dark'>('light');
const locale = ref('zh-CN');

function toggleTheme() {
  theme.value = theme.value === 'light' ? 'dark' : 'light';
}

// 使用 readonly 防止子组件直接修改
provide('theme', readonly(theme));
provide('locale', readonly(locale));
// 提供修改方法
provide('toggleTheme', toggleTheme);
</script>
```

### 后代组件注入

```typescript
<script setup lang="ts">
import { inject } from 'vue';
import type { Ref } from 'vue';

// 注入时提供默认值（子组件脱离父组件时有用）
const theme = inject<Ref<'light' | 'dark'>>('theme', ref('light'));
const locale = inject<Ref<string>>('locale', ref('zh-CN'));
const toggleTheme = inject<() => void>('toggleTheme', () => {});
</script>

<template>
  <div :class="`app-${theme}`">
    <p>当前语言: {{ locale }}</p>
    <button @click="toggleTheme">切换主题</button>
  </div>
</template>
```

## 高级模式

### 模式 1：Symbol Injection Key（类型安全）

```typescript
// symbols/injection-keys.ts
import type { InjectionKey, Ref } from 'vue';

// 泛型 InjectionKey 提供完整的类型推导
export interface ThemeContext {
  theme: Ref<'light' | 'dark'>;
  toggleTheme: () => void;
}

export const THEME_KEY: InjectionKey<ThemeContext> = Symbol('theme');

export interface FormContext {
  model: Record<string, any>;
  errors: Ref<Record<string, string>>;
  validate: (field: string) => boolean;
}

export const FORM_KEY: InjectionKey<FormContext> = Symbol('form');
```

```typescript
// 提供者
import { provide } from 'vue';
import { THEME_KEY } from '@/symbols/injection-keys';

provide(THEME_KEY, {
  theme,
  toggleTheme,
});
```

```typescript
// 消费者 — 自动类型推导
import { inject } from 'vue';
import { THEME_KEY } from '@/symbols/injection-keys';

const ctx = inject(THEME_KEY);
// ctx 自动推导为 ThemeContext | undefined

const ctx2 = inject(THEME_KEY)!; // 非空断言
// ctx2.theme 类型为 Ref<'light' | 'dark'>
```

### 模式 2：Form 表单上下文（实际案例）

```typescript
// 表单提供者 — 封装验证逻辑
<!-- FormRoot.vue -->
<script setup lang="ts">
import { ref, provide, reactive } from 'vue';
import { FORM_KEY } from '@/symbols/injection-keys';
import type { FormContext } from '@/symbols/injection-keys';

const props = defineProps<{
  model: Record<string, any>;
}>();

const errors = reactive<Record<string, string>>({});

function validate(field: string): boolean {
  // 简单验证示例
  const value = props.model[field];
  if (value === undefined || value === '') {
    errors[field] = `${field} 不能为空`;
    return false;
  }
  delete errors[field];
  return true;
}

function validateAll(): boolean {
  return Object.keys(props.model).every(field => validate(field));
}

provide(FORM_KEY, {
  model: props.model,
  errors,
  validate,
});

// 暴露提交方法给父组件
defineExpose({ validateAll });
</script>

<template>
  <form class="form-root">
    <slot />
  </form>
</template>
```

```typescript
// 表单项组件 — 注入表单上下文
<!-- FormItem.vue -->
<script setup lang="ts">
import { inject, computed } from 'vue';
import { FORM_KEY } from '@/symbols/injection-keys';

const props = defineProps<{
  field: string;
  label: string;
}>();

const form = inject(FORM_KEY);
const error = computed(() => form?.errors[props.field] || '');
</script>

<template>
  <div class="form-item" :class="{ 'has-error': error }">
    <label>{{ label }}</label>
    <slot :model="form?.model" :field="props.field" />
    <p v-if="error" class="error-msg">{{ error }}</p>
  </div>
</template>
```

### 模式 3：响应式注入 + readonly 保护

```typescript
// 使用 readonly + provide 防止子组件非法修改
import { readonly, shallowRef, provide } from 'vue';

// ❌ 错误：直接提供响应式对象，子组件可修改
provide('config', config); // 子组件 inject('config').value = xxx

// ✅ 正确：提供 readonly 包装
provide('config', readonly(config)); // 子组件不可修改

// 提供修改方法给子组件调用
function updateConfig(key: string, value: any) {
  config.value[key] = value;
}
provide('updateConfig', updateConfig);
```

### 模式 4：provide 配合 shallowRef 优化

```typescript
import { shallowRef, provide } from 'vue';

// 大数据场景使用 shallowRef（不深度追踪）
const largeDataset = shallowRef({
  users: [] as User[],
  page: 1,
  total: 0,
});

// 整体替换而非深度修改
function updateUsers(users: User[]) {
  largeDataset.value = { ...largeDataset.value, users };
}

provide('dataset', readonly(largeDataset));
provide('updateUsers', updateUsers);
```

## provide 层级覆盖

子组件可以覆盖上层 provide，不影响兄弟组件：

```typescript
// Layout.vue（提供者）
provide('title', '主标题');

// AdminLayout.vue（中间层，覆盖提供）
provide('title', '管理后台');

// Page.vue（注入者）
const title = inject('title'); // 得到 '管理后台'
```

## 替代方案对比

| 方案 | 优点 | 缺点 |
|---|---|---|
| **provide/inject** | 轻量，无需额外库，类型安全（Symbol） | 不适合全局状态，无 DevTools |
| **Pinia** | DevTools 支持，模块化，持久化 | 引入额外依赖，简单场景过度设计 |
| **Vuex** | 成熟 | 已不推荐，Pinia 是官方替代 |
| **Props 透传** | 显式声明，调试容易 | 深层嵌套时繁琐 |
| **EventBus** | 灵活 | 调试困难，不推荐 |

## 注意事项

- **命名冲突**：Injection key 使用 Symbol 替代字符串字面量，避免全局命名冲突
- **undefined 处理**：`inject()` 可能返回 `undefined`（当提供者不存在时），始终提供默认值或使用可选链
- **响应式丢失**：`inject()` 拿到的如果是基本类型（string/number），默认非响应式。需包装为 `ref()` 或 `computed()`
- **readonly 保护**：提供方使用 `readonly()` 包裹后再 provide，防止子组件意外修改父状态
- **不要滥用**：同页面内跨 3 层以上才考虑 provide/inject，否则 props 更清晰
- **不可 tree-shake**：provide/inject 是运行时绑定，不会像 Pinia 那样有 DevTools 时间旅行调试
- **SSR 注意**：服务端渲染时 provide 在 setup 中执行，确保所有 inject 都有默认值兜底
- **组件库开发**：组件库内部推荐 provide/inject + Symbol 传递内部状态（如 el-form 的 validate 方法），对外暴露 props
