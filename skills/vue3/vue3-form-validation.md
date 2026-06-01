---
name: vue3-form-validation
description: Vue3 表单验证最佳实践：VeeValidate 集成、Zod 模式校验、组合式 API 封装
tags: [vue3, form, validation, veevalidate, zod, typescript]
---

## 概述

Vue3 表单验证推荐的方案是 **VeeValidate**（表单状态管理）结合 **Zod**（模式校验引擎），利用 Composition API 实现类型安全、可复用的验证逻辑。

## 1. VeeValidate + Zod 基础配置

```bash
npm install vee-validate zod @vee-validate/zod
```

### 全局注册

```typescript
// main.ts
import { createApp } from 'vue'
import { configure } from 'vee-validate'
import { zodResolver } from '@vee-validate/zod'
import { z } from 'zod'
import App from './App.vue'

// 设置全局 Zod Resolver（可选，也可以在组件内按需设置）
configure({
  generateMessage: (ctx) => {
    // 自定义错误消息格式化
    return ctx.rule?.message || `${ctx.field} 校验失败`
  }
})

createApp(App).mount('#app')
```

## 2. 基础表单验证

### 使用 `<Form>` + `<Field>` 组件

```vue
<template>
  <Form
    :validation-schema="loginSchema"
    @submit="onSubmit"
    v-slot="{ errors, isSubmitting }"
  >
    <div class="field">
      <label>邮箱</label>
      <Field
        name="email"
        type="email"
        placeholder="请输入邮箱"
        class="input"
        :class="{ 'input-error': errors.email }"
      />
      <ErrorMessage name="email" class="error-msg" />
    </div>

    <div class="field">
      <label>密码</label>
      <Field
        name="password"
        type="password"
        placeholder="请输入密码"
        class="input"
        :class="{ 'input-error': errors.password }"
      />
      <ErrorMessage name="password" class="error-msg" />
    </div>

    <button type="submit" :disabled="isSubmitting">
      {{ isSubmitting ? '提交中...' : '登录' }}
    </button>
  </Form>
</template>

<script setup lang="ts">
import { Form, Field, ErrorMessage } from 'vee-validate'
import { toTypedSchema } from '@vee-validate/zod'
import { z } from 'zod'

// Zod Schema 定义
const loginSchema = toTypedSchema(
  z.object({
    email: z
      .string()
      .min(1, '请输入邮箱')
      .email('邮箱格式不正确'),
    password: z
      .string()
      .min(6, '密码至少6位')
      .max(20, '密码最多20位'),
  })
)

// 提交处理
const onSubmit = (values: { email: string; password: string }) => {
  // values 已通过 Zod 校验，类型安全
  console.log('登录:', values)
}
</script>
```

## 3. 使用 Composition API（`useForm`）

适合复杂表单/动态表单场景。

```typescript
// composables/useLoginForm.ts
import { useForm, useField } from 'vee-validate'
import { toTypedSchema } from '@vee-validate/zod'
import { z } from 'zod'
import type { LoginRequest } from '@/types'

export function useLoginForm() {
  const schema = toTypedSchema(
    z.object({
      email: z.string().email('邮箱格式不正确'),
      password: z.string().min(6, '密码至少6位'),
      rememberMe: z.boolean().default(false),
    })
  )

  const { handleSubmit, isSubmitting, errors, resetForm } = useForm({
    validationSchema: schema,
    initialValues: {
      email: '',
      password: '',
      rememberMe: false,
    },
  })

  const { value: email } = useField<string>('email')
  const { value: password } = useField<string>('password')
  const { value: rememberMe } = useField<boolean>('rememberMe')

  const onSubmit = handleSubmit((values: LoginRequest) => {
    // 自动校验通过后触发
    return loginApi(values)
  })

  return {
    email,
    password,
    rememberMe,
    errors,
    isSubmitting,
    onSubmit,
    resetForm,
  }
}
```

```vue
<!-- 在组件中使用 -->
<script setup lang="ts">
import { useLoginForm } from '@/composables/useLoginForm'

const { email, password, rememberMe, errors, isSubmitting, onSubmit } = useLoginForm()
</script>

<template>
  <form @submit="onSubmit">
    <input v-model="email" placeholder="邮箱" />
    <span class="error">{{ errors.email }}</span>

    <input v-model="password" type="password" placeholder="密码" />
    <span class="error">{{ errors.password }}</span>

    <label><input v-model="rememberMe" type="checkbox" /> 记住我</label>

    <button type="submit" :disabled="isSubmitting">提交</button>
  </form>
</template>
```

## 4. 复杂验证模式

### 交叉字段校验

```typescript
const registerSchema = toTypedSchema(
  z.object({
    password: z.string().min(8, '密码至少8位'),
    confirmPassword: z.string(),
  }).refine(
    (data) => data.password === data.confirmPassword,
    { message: '两次密码不一致', path: ['confirmPassword'] }
  )
)
```

### 动态表单（数组字段）

```typescript
interface Item {
  name: string
  quantity: number
}

const orderSchema = toTypedSchema(
  z.object({
    items: z.array(z.object({
      name: z.string().min(1, '商品名称不能为空'),
      quantity: z.number().min(1, '数量至少为1').max(99),
    })).min(1, '至少添加一个商品'),
  })
)

// 动态添加字段
const { values, setFieldValue, push} = useForm({
  validationSchema: orderSchema,
})

const addItem = () => {
  // VeeValidate 4.x 使用 setFieldValue 操作数组
  const currentItems = values.items || []
  setFieldValue('items', [...currentItems, { name: '', quantity: 1 }])
}

const removeItem = (index: number) => {
  const items = [...(values.items || [])]
  items.splice(index, 1)
  setFieldValue('items', items)
}
```

### 异步校验（检查用户名重复）

```typescript
const schema = toTypedSchema(
  z.object({
    username: z
      .string()
      .min(3, '用户名至少3位')
      // Zod 管道 —— 先做基础校验，再做异步校验
      .pipe(
        z.string().refine(
          async (val) => {
            if (!val) return true
            const exists = await checkUsernameExists(val)
            return !exists
          },
          { message: '用户名已被占用' }
        )
      ),
  })
)
```

## 5. 自定义组件集成

```vue
<!-- components/AppInput.vue -->
<template>
  <div class="app-input">
    <label v-if="label">{{ label }}</label>
    <input
      :value="value"
      @input="handleInput"
      @blur="handleBlur"
      v-bind="$attrs"
    />
    <span v-if="error" class="error">{{ error }}</span>
  </div>
</template>

<script setup lang="ts">
import { useField } from 'vee-validate'

const props = defineProps<{
  name: string
  label?: string
}>()

// useField 自动关联 Form 的 validation-schema
const { value, errorMessage, handleBlur, handleInput } = useField(() => props.name)
</script>
```

```vue
<!-- 使用自定义组件 -->
<Form :validation-schema="mySchema">
  <AppInput name="email" label="邮箱" type="email" />
  <AppInput name="password" label="密码" type="password" />
</Form>
```

## 6. 常用 Zod Schema 模板

```typescript
// 手机号
export const phoneSchema = z.string().regex(
  /^1[3-9]\d{9}$/,
  '手机号格式不正确'
)

// 身份证号（简单校验）
export const idCardSchema = z.string().regex(
  /^\d{17}[\dXx]$/,
  '身份证号格式不正确'
)

// 分页查询参数
export const pageSchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  pageSize: z.coerce.number().int().min(1).max(100).default(20),
  keyword: z.string().optional(),
  status: z.enum(['active', 'inactive', 'all']).default('all'),
})

// 枚举值校验
export const RoleEnum = z.enum(['admin', 'editor', 'viewer'])
export type Role = z.infer<typeof RoleEnum>
```

## 7. VeeValidate 手动触发校验

```typescript
import { useForm } from 'vee-validate'

const { validate, setErrors, setFieldError, setValues } = useForm({
  validationSchema: mySchema,
})

// 手动触发全表单校验
const isValid = await validate()

// 手动设置服务端返回的错误
setErrors({
  email: '该邮箱已被注册',
  password: '密码强度不足',
})

// 设置单个字段错误
setFieldError('username', '用户名不合法')

// 回填表单值
setValues({
  name: '张三',
  age: 25,
})
```

## 注意事项

1. **不要混用 VeeValidate 和原生 HTML5 验证**：设置 `<form novalidate>` 关闭浏览器默认验证，避免冲突
2. **Zod 版本兼容性**：`@vee-validate/zod` 目前仅支持 Zod 3.x，升级 Zod 大版本前确认兼容
3. **异步校验性能**：异步 `refine` 在每次字段变化时都会触发，建议配合防抖使用
4. **类型推断**：使用 `z.infer<typeof schema>` 获取表单类型，避免手动定义重复类型
5. **I18n 集成**：Zod 错误消息通过 `z.string().min(1, { message: t('email.required') })` 与 Vue I18n 联动
6. **性能优化**：大型表单建议 `useField` 按需监听，避免在 `Form` 层面监听所有字段变化
