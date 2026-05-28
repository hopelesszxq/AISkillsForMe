---
name: vue3-testing
description: Vue3 组件测试与工程化配置（Vitest + Testing Library + Auto-Import）
tags: [vue3, vitest, testing, frontend, engineering]
---

## Vitest 单元测试

Vue3 官方推荐 Vitest 替代 Jest，原生支持 ESM、TypeScript 和 Vite 插件。

### 安装配置

```bash
npm install -D vitest @vue/test-utils @vitejs/plugin-vue jsdom
```

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov'],
      include: ['src/**/*.{vue,ts}'],
      exclude: ['src/**/*.d.ts', 'src/test/**'],
    },
  },
})
```

```typescript
// src/test/setup.ts — 全局测试配置
import { vi } from 'vitest'
import { config } from '@vue/test-utils'

// 全局 Stub 第三方组件
config.global.stubs = {
  'router-link': true,
  'router-view': true,
}

// Mock ResizeObserver（某些 UI 库需要）
vi.stubGlobal('ResizeObserver', vi.fn(() => ({
  observe: vi.fn(),
  unobserve: vi.fn(),
  disconnect: vi.fn(),
})))
```

### 组件测试示例

```typescript
// src/components/UserCard.vue
<script setup lang="ts">
defineProps<{
  user: { name: string; email: string; avatar?: string }
}>()
const emit = defineEmits<{
  click: [id: string]
}>()
</script>

<template>
  <div class="user-card" @click="emit('click', user.email)">
    <img v-if="user.avatar" :src="user.avatar" alt="avatar" />
    <div class="info">
      <h3>{{ user.name }}</h3>
      <p>{{ user.email }}</p>
    </div>
  </div>
</template>
```

```typescript
// src/components/__tests__/UserCard.spec.ts
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import UserCard from '../UserCard.vue'

describe('UserCard', () => {
  const mockUser = {
    name: '张三',
    email: 'zhangsan@example.com',
    avatar: 'https://example.com/avatar.jpg',
  }

  it('渲染用户信息', () => {
    const wrapper = mount(UserCard, { props: { user: mockUser } })
    expect(wrapper.find('h3').text()).toBe('张三')
    expect(wrapper.find('p').text()).toBe('zhangsan@example.com')
  })

  it('无头像时不渲染 img 标签', () => {
    const userWithoutAvatar = { name: '李四', email: 'lisi@example.com' }
    const wrapper = mount(UserCard, { props: { user: userWithoutAvatar } })
    expect(wrapper.find('img').exists()).toBe(false)
  })

  it('点击触发 click 事件', async () => {
    const wrapper = mount(UserCard, { props: { user: mockUser } })
    await wrapper.find('.user-card').trigger('click')
    expect(wrapper.emitted('click')).toBeTruthy()
    expect(wrapper.emitted('click')![0]).toEqual(['zhangsan@example.com'])
  })
})
```

### 组合式函数测试

```typescript
// composables/useCounter.ts
import { ref } from 'vue'
export function useCounter(initial = 0) {
  const count = ref(initial)
  const increment = () => count.value++
  const decrement = () => count.value--
  const reset = () => { count.value = initial }
  return { count, increment, decrement, reset }
}
```

```typescript
// composables/__tests__/useCounter.spec.ts
import { describe, it, expect } from 'vitest'
import { useCounter } from '../useCounter'

describe('useCounter', () => {
  it('初始值为 0', () => {
    const { count } = useCounter()
    expect(count.value).toBe(0)
  })

  it('increment 增加计数', () => {
    const { count, increment } = useCounter()
    increment()
    expect(count.value).toBe(1)
  })

  it('可以自定义初始值', () => {
    const { count, reset } = useCounter(10)
    expect(count.value).toBe(10)
    reset()
    expect(count.value).toBe(10)
  })
})
```

### Pinia Store 测试

```typescript
// stores/__tests__/user.spec.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'
import { useUserStore } from '../user'

describe('UserStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia()) // 每个测试重置 Pinia
  })

  it('登录成功更新状态', async () => {
    const store = useUserStore()
    // Mock api.login
    vi.mock('@/api/user', () => ({
      login: vi.fn().mockResolvedValue({
        token: 'fake-token',
        name: '测试用户',
        roles: ['user'],
      }),
    }))
    await store.login('test', '123456')
    expect(store.token).toBe('fake-token')
    expect(store.name).toBe('测试用户')
  })

  it('logout 重置所有状态', () => {
    const store = useUserStore()
    store.name = '张三'
    store.token = 'xxx'
    store.logout()
    expect(store.name).toBe('')
    expect(store.token).toBe('')
  })
})
```

## unplugin-auto-import 工程化配置

Vue3 项目中推荐用自动导入代替手动 import，减少样板代码。

### 安装配置

```bash
npm install -D unplugin-auto-import unplugin-vue-components
```

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import AutoImport from 'unplugin-auto-import/vite'
import Components from 'unplugin-vue-components/vite'
import { ElementPlusResolver } from 'unplugin-vue-components/resolvers'

export default defineConfig({
  plugins: [
    vue(),
    AutoImport({
      imports: [
        'vue',           // ref, reactive, computed, watch 等
        'vue-router',    // useRouter, useRoute
        'pinia',         // defineStore, storeToRefs
        '@vueuse/core',  // useDebounce, useLocalStorage 等
      ],
      resolvers: [ElementPlusResolver()],
      dts: 'src/auto-imports.d.ts',  // 生成类型声明
      eslintrc: {
        enabled: true,   // 生成 .eslintrc-auto-import.json 供 ESLint 识别
      },
    }),
    Components({
      resolvers: [ElementPlusResolver()],
      dts: 'src/components.d.ts',
    }),
  ],
})
```

配置后，项目中无需手动 import：

```vue
<script setup lang="ts">
// 无需 import { ref, computed, watch } from 'vue'
// 无需 import { useRouter } from 'vue-router'
// 无需 import { useDebounce } from '@vueuse/core'

const count = ref(0)
const doubled = computed(() => count.value * 2)
const router = useRouter()
const debouncedSearch = useDebounce(searchQuery, 300)
</script>
```

### ESLint 适配

```javascript
// .eslintrc.cjs
module.exports = {
  extends: [
    './.eslintrc-auto-import.json', // 自动生成的配置
  ],
}
```

## Vite 高级配置

### 环境变量管理

```typescript
// .env.development
VITE_API_BASE_URL=http://localhost:8080/api
VITE_APP_TITLE=开发环境

// .env.production
VITE_API_BASE_URL=https://api.example.com
VITE_APP_TITLE=生产环境
```

```typescript
// 类型安全的环境变量
// src/env.d.ts
/// <reference types="vite/client" />
interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string
  readonly VITE_APP_TITLE: string
}

// 使用
const baseUrl = import.meta.env.VITE_API_BASE_URL
```

### 代理配置（解决跨域）

```typescript
// vite.config.ts
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
      '/ws': {
        target: 'ws://localhost:8080',
        ws: true,
      },
    },
  },
})
```

## 注意事项

1. **jsdom vs happy-dom**：jsdom 更稳定兼容性好，happy-dom 更快但对 canvas/WebGL 支持差
2. **异步组件测试**：使用 `flushPromises()` 等待异步渲染完成，或 `nextTick()` 等待 DOM 更新
3. **测试不可依赖全局状态**：每个测试前 `setActivePinia(createPinia())` 重置，避免测试间污染
4. **Auto-Import 的 ESLint 警告**：启用 `eslintrc.enabled` 生成配置文件，否则 ESLint 报 `no-undef`
5. **路径别名**：`vite.config.ts` 中配置 `resolve.alias`，vitest 自动继承无需重复配置
6. **快照测试**：`toMatchSnapshot()` 适用于稳定的 UI 组件，频繁变更的组件不建议用快照
7. **coverage 阈值**：CI 中可设置覆盖率门槛，低于阈值构建失败
