---
name: vue3-pinia-deep-patterns
description: Vue3 Pinia 深度使用模式：Store 设计、组合式 Store、插件机制、持久化加密、SSR 兼容
tags: [vue3, pinia, state-management, composition-api, frontend, store]
---

## 概述

Pinia 是 Vue3 官方推荐的状态管理方案，相比 Vuex 去除了 Mutation、完整的 TypeScript 支持、天然支持 Composition API。本文覆盖 Pinia 的高阶设计模式：Store 工厂、组合式 Store、自定义插件、加密持久化、SSR 兼容等。

## 1. Store 设计模式

### 1.1 选项式 vs 组合式 Store

```typescript
// 选项式 Store（Option Store）：类似 Vuex 风格
import { defineStore } from 'pinia'

export const useCounterStore = defineStore('counter', {
  state: () => ({ count: 0, name: 'counter' }),
  getters: {
    doubleCount: (state) => state.count * 2,
  },
  actions: {
    increment() {
      this.count++
    },
  },
})
```

```typescript
// 组合式 Store（Setup Store）：类似 Composition API 风格（推荐）
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useCounterStore = defineStore('counter', () => {
  const count = ref(0)
  const name = ref('counter')

  const doubleCount = computed(() => count.value * 2)

  function increment() {
    count.value++
  }

  return { count, name, doubleCount, increment }
})
```

**选择建议**：
- 简单 CRUD 场景：选项式更简洁
- 复杂业务逻辑、需要组合多个 composable：组合式更灵活
- 团队长期维护：组合式 + TypeScript 更易推理

### 1.2 Store 工厂模式（动态创建 Store）

当多个业务实体使用相同的状态模式时，用工厂函数创建同构 Store：

```typescript
// stores/factory.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import type { Ref } from 'vue'

interface EntityState<T> {
  list: T[]
  loading: boolean
  error: string | null
}

export function createEntityStore<T extends { id: number | string }>(
  id: string,
  fetchApi: () => Promise<T[]>
) {
  return defineStore(id, () => {
    const state = ref<EntityState<T>>({
      list: [],
      loading: false,
      error: null,
    })

    const list = computed(() => state.value.list)
    const loading = computed(() => state.value.loading)

    async function fetchList() {
      state.value.loading = true
      state.value.error = null
      try {
        state.value.list = await fetchApi()
      } catch (e: any) {
        state.value.error = e.message
      } finally {
        state.value.loading = false
      }
    }

    return { state, list, loading, fetchList }
  })
}

// 使用
// stores/user.ts
import { createEntityStore } from './factory'
import { fetchUsers } from '@/api/user'

export const useUserStore = createEntityStore<User>('user', fetchUsers)

// stores/order.ts
import { createEntityStore } from './factory'
import { fetchOrders } from '@/api/order'

export const useOrderStore = createEntityStore<Order>('order', fetchOrders)
```

### 1.3 组合式 Store（Store 之间互相引用）

```typescript
// stores/cart.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { useAuthStore } from './auth'

export const useCartStore = defineStore('cart', () => {
  const items = ref<CartItem[]>([])
  const authStore = useAuthStore()

  const totalPrice = computed(() =>
    items.value.reduce((sum, item) => sum + item.price * item.quantity, 0)
  )

  // 依赖其他 Store 的 getter
  const checkoutEnabled = computed(() =>
    authStore.isLoggedIn && items.value.length > 0
  )

  async function checkout() {
    if (!authStore.isLoggedIn) {
      throw new Error('请先登录')
    }
    // ... 结账逻辑
  }

  return { items, totalPrice, checkoutEnabled, checkout }
})
```

## 2. 高级 Getter 模式

### 2.1 Getter 返回函数（传参查询）

```typescript
export const useProductStore = defineStore('product', () => {
  const products = ref<Product[]>([])

  // ✅ Getter 返回函数，支持传参
  const getProductById = computed(() => (id: number) =>
    products.value.find(p => p.id === id)
  )

  const getProductsByCategory = computed(() => (category: string) =>
    products.value.filter(p => p.category === category)
  )

  return { products, getProductById, getProductsByCategory }
})

// 使用
const store = useProductStore()
const product = store.getProductById(42) // ✅ 响应式
```

### 2.2 缓存计算结果（避免重复计算）

```typescript
export const useReportStore = defineStore('report', () => {
  const transactions = ref<Transaction[]>([])

  // 计算结果被缓存，仅在 transactions 变化时重新计算
  const summary = computed(() => ({
    total: transactions.value.reduce((s, t) => s + t.amount, 0),
    count: transactions.value.length,
    avg: transactions.value.length
      ? transactions.value.reduce((s, t) => s + t.amount, 0) / transactions.value.length
      : 0,
  }))

  return { transactions, summary }
})
```

## 3. 插件机制

### 3.1 基本插件结构

```typescript
// plugins/logger.ts
import type { PiniaPluginContext } from 'pinia'

export function loggerPlugin({ store, options }: PiniaPluginContext) {
  // 每次 state 变化时打印日志
  store.$subscribe((mutation, state) => {
    console.log(`[Pinia] ${store.$id} 变更:`, mutation.type, mutation.payload)
  })

  // 记录 Store 创建时间
  store.createdAt = Date.now()
}

// main.ts 注册
import { createPinia } from 'pinia'
const pinia = createPinia()
pinia.use(loggerPlugin)
```

### 3.2 加密持久化插件

替换简单的 `pinia-plugin-persistedstate`，实现 AES 加密持久化：

```typescript
// plugins/encrypted-persist.ts
import type { PiniaPluginContext } from 'pinia'

// 简单加密函数（生产环境使用 Web Crypto API 或 crypto-js）
function encrypt(data: string, key: string): string {
  return btoa(encodeURIComponent(data + ':' + key))
}

function decrypt(data: string, key: string): string | null {
  try {
    const decoded = decodeURIComponent(atob(data))
    const parts = decoded.split(':')
    if (parts.pop() !== key) return null // 校验 key
    return parts.join(':')
  } catch {
    return null
  }
}

interface PersistConfig {
  key?: string
  paths?: string[]
  encryptKey?: string
}

export function encryptedPersistPlugin(config: PersistConfig = {}) {
  return ({ store }: PiniaPluginContext) => {
    const storageKey = config.key ?? store.$id
    const encryptKey = config.encryptKey ?? 'default-key'

    // 恢复
    try {
      const saved = localStorage.getItem(storageKey)
      if (saved) {
        const decrypted = decrypt(saved, encryptKey)
        if (decrypted) {
          store.$patch(JSON.parse(decrypted))
        }
      }
    } catch { /* ignore */ }

    // 持久化
    store.$subscribe((_, state) => {
      const toSave = config.paths
        ? Object.fromEntries(config.paths.map(p => [p, (state as any)[p]]))
        : state
      const encrypted = encrypt(JSON.stringify(toSave), encryptKey)
      localStorage.setItem(storageKey, encrypted)
    })
  }
}

// 使用
const pinia = createPinia()
pinia.use(encryptedPersistPlugin({
  key: 'app-state',
  paths: ['user', 'cart'], // 只持久化部分 state
  encryptKey: import.meta.env.VITE_STORAGE_KEY,
}))
```

### 3.3 埋点/统计插件

```typescript
export function analyticsPlugin({ store }: PiniaPluginContext) {
  // 监听 action 调用
  store.$onAction(({ name, store, args, after, onError }) => {
    const start = performance.now()
    after((result) => {
      const duration = performance.now() - start
      console.log(`[Analytics] ${store.$id}.${name} 耗时 ${duration.toFixed(2)}ms`)
      // 上报埋点
      trackEvent('store_action', {
        store: store.$id,
        action: name,
        duration,
      })
    })
    onError((error) => {
      trackEvent('store_error', {
        store: store.$id,
        action: name,
        error: error.message,
      })
    })
  })
}
```

## 4. SSR 兼容

```typescript
// stores/ssr.ts
export const useSSRSafeStore = defineStore('ssr-safe', () => {
  // 确保只在客户端访问 window/document
  const isClient = ref(false)

  // SSR 时不会执行
  onMounted(() => {
    isClient.value = true
  })

  const localData = computed(() => {
    if (!isClient.value) return null
    return JSON.parse(localStorage.getItem('data') ?? '{}')
  })

  return { isClient, localData }
})
```

## 5. 使用 storeToRefs 保持响应性

```typescript
import { storeToRefs } from 'pinia'

const store = useCounterStore()
// ❌ 解构后丢失响应性
const { count } = store

// ✅ 保持响应性
const { count, doubleCount } = storeToRefs(store)
// actions 可以直接解构（函数没有响应性问题）
const { increment } = store
```

## 6. 注意事项

### 6.1 Store 不要在 setup 外使用

```typescript
// ❌ 错误：组件外使用 store 可能没有活跃的 Pinia 实例
const store = useCounterStore()

// ✅ 正确：在组件 setup 或函数内使用
export function useSomeFeature() {
  const store = useCounterStore()
  return { store }
}
```

### 6.2 避免在 getter 中返回新的引用

```typescript
// ❌ 错误：每次访问 getter 都返回新数组，破坏缓存
const topProducts = computed(() => products.value.slice(0, 10))

// ✅ 正确：如果确实需要派生数组，确保使用方式不变
// 或在组件中用 computed 包裹
```

### 6.3 大型 Store 拆分原则

- 一个页面/功能领域最多一个 Store
- 全局状态（用户、主题、权限）放在共享 Store
- 页面级状态放在页面自己的 Store
- Store 超过 300 行考虑拆分子 Store

### 6.4 DevTools 最佳实践

- 为 action 命名清晰，便于调试追踪
- 避免直接修改 state（始终走 action）
- 使用 `$patch` 批量更新减少 DevTools 记录
