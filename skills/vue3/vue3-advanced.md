---
name: vue3-advanced
description: Vue3 Pinia 状态管理、Vue Router 进阶、Nuxt 与 VueUse 工具集
tags: [vue3, pinia, router, nuxt, vueuse, frontend]
---

## Pinia 状态管理

### 选项式 Store vs 组合式 Store

```typescript
// 选项式 (Option Store)
export const useUserStore = defineStore('user', {
  state: () => ({
    name: '',
    token: '',
    roles: [] as string[],
  }),
  getters: {
    isAdmin: (state) => state.roles.includes('admin'),
    displayName: (state) => state.name || '未登录',
  },
  actions: {
    async login(username: string, password: string) {
      const res = await api.login(username, password);
      this.token = res.token;
      this.name = res.name;
      this.roles = res.roles;
    },
    logout() {
      this.$reset(); // 一键重置所有状态
    },
  },
});

// 组合式 (Setup Store) — 更灵活，推荐复杂逻辑
export const useCartStore = defineStore('cart', () => {
  const items = ref<CartItem[]>([]);
  const totalPrice = computed(() =>
    items.value.reduce((sum, i) => sum + i.price * i.qty, 0)
  );

  function addItem(item: CartItem) {
    const existing = items.value.find(i => i.id === item.id);
    existing ? existing.qty++ : items.value.push({ ...item, qty: 1 });
  }

  async function checkout() {
    const res = await api.checkout(items.value);
    items.value = [];
    return res;
  }

  return { items, totalPrice, addItem, checkout };
});
```

### Pinia 插件（持久化缓存）

```typescript
// pinia-plugin-persistedstate
// npm i pinia-plugin-persistedstate

// main.ts
import { createPinia } from 'pinia';
import piniaPluginPersistedstate from 'pinia-plugin-persistedstate';
const pinia = createPinia();
pinia.use(piniaPluginPersistedstate);

// Store 启用持久化
export const useUserStore = defineStore('user', {
  state: () => ({ token: '', theme: 'light' }),
  persist: {
    key: 'my-app-user',       // localStorage key
    storage: localStorage,    // 默认 localStorage
    paths: ['token'],         // 只持久化 token
  },
});
```

## Vue Router 进阶

### 路由权限控制

```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router';

const routes = [
  { path: '/login', component: () => import('@/views/Login.vue') },
  {
    path: '/admin',
    component: () => import('@/layouts/AdminLayout.vue'),
    meta: { roles: ['admin'] },
    children: [
      { path: 'dashboard', component: () => import('@/views/Dashboard.vue') },
    ],
  },
];

const router = createRouter({ history: createWebHistory(), routes });

router.beforeEach(async (to, from, next) => {
  const userStore = useUserStore();
  
  if (to.meta.roles && !to.meta.roles.some(r => userStore.roles.includes(r))) {
    next('/403');
  } else {
    next();
  }
});
```

### 路由过渡动画

```vue
<router-view v-slot="{ Component, route }">
  <transition :name="route.meta.transition || 'fade'" mode="out-in">
    <component :is="Component" :key="route.path" />
  </transition>
</router-view>

<style>
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s ease;
}
.fade-enter-from, .fade-leave-to {
  opacity: 0;
}
.slide-enter-active, .slide-leave-active {
  transition: transform 0.3s ease;
}
.slide-enter-from { transform: translateX(20px); }
.slide-leave-to { transform: translateX(-20px); }
</style>
```

## VueUse 实用组合式

```typescript
// npm i @vueuse/core

// 1. 防抖/节流
const searchQuery = ref('');
const debouncedQuery = useDebounce(searchQuery, 300);
watch(debouncedQuery, (val) => api.search(val));

// 2. 在线状态检测
const online = useOnline();

// 3. 页面可见性（切后台暂停轮询）
const visibility = useDocumentVisibility();
watch(visibility, (val) => {
  if (val === 'visible') refreshData();
});

// 4. 拖拽
const { x, y, isDragging } = useDragg(elRef);

// 5. 全屏
const { isFullscreen, toggle } = useFullscreen();

// 6. 存储响应式绑定
const token = useLocalStorage('token', '');
const theme = useSessionStorage('theme', 'light');
```

## Nuxt 3 基础套路

```typescript
// 服务端数据获取（避免瀑布流）
// 推荐并行获取
const [users, posts] = await Promise.all([
  useFetch('/api/users'),
  useFetch('/api/posts'),
]);

// 自动导入 composables
// composables/useCounter.ts
export const useCounter = () => {
  const count = ref(0);
  const increment = () => count.value++;
  return { count, increment };
};
// 在任意页面中直接调用，无需 import

// 中间件
// middleware/auth.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const token = useCookie('token');
  if (!token.value && to.path !== '/login') return navigateTo('/login');
});
```

## 注意事项

- **Pinia vs Vuex**：Pinia 是官方推荐，完全替代 Vuex，更好的 TypeScript 支持和去掉了 mutations
- **持久化插件**：`pinia-plugin-persistedstate` 需注意敏感信息（token）不要明文存入 localStorage，可配合加密
- **组件命名**：使用多单词命名（`UserProfileCard`），避免与 HTML 原生标签冲突
- **服务器端渲染**：Nuxt 中使用 `<ClientOnly>` 包裹浏览器特有 API（如 window, localStorage）
- **KeepAlive**：动态组件 + `<KeepAlive>` 时注意 `onActivated()`/`onDeactivated()` 生命周期
- **VueUse 按需导入**：tree-shaking 友好，不用担心 bundle 大小
