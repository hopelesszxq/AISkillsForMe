---
name: vue3-composable-patterns
description: Vue3 组合式函数（Composable）高阶设计模式：状态管理、异步逻辑、生命周期组合与类型安全
tags: [vue3, composable, composition-api, typescript, frontend, patterns]
---

## 概述

组合式函数（Composable）是 Vue3 中基于 Composition API 的逻辑复用单元。本文覆盖从基础到进阶的 Composable 设计模式，包括状态工厂、异步状态管理、watch 组合、生命周期组合、泛型 Composable 等实战模式。

## 1. 状态工厂模式（State Factory）

创建可复用的状态管理 Composable，支持重置和订阅。

```typescript
// useCounter.ts — 带 reset 的计数器工厂
import { ref, readonly, type Ref } from 'vue'

interface UseCounterOptions {
  min?: number
  max?: number
  step?: number
}

interface UseCounterReturn {
  count: Ref<number>
  readonly countReadonly: Readonly<Ref<number>>
  increment: () => void
  decrement: () => void
  reset: () => void
  set: (value: number) => void
}

export function useCounter(initial = 0, options: UseCounterOptions = {}): UseCounterReturn {
  const { min = -Infinity, max = Infinity, step = 1 } = options
  const count = ref(initial)
  
  const increment = () => {
    const next = count.value + step
    if (next <= max) count.value = next
  }
  const decrement = () => {
    const next = count.value - step
    if (next >= min) count.value = next
  }
  const reset = () => { count.value = initial }
  const set = (value: number) => {
    if (value >= min && value <= max) count.value = value
  }

  return {
    count,
    countReadonly: readonly(count),
    increment,
    decrement,
    reset,
    set,
  }
}
```

## 2. 异步状态管理（Async Composable）

封装请求的生命周期状态，统一处理 loading / error / data。

```typescript
// useAsyncData.ts
import { ref, type Ref, toValue, watchEffect } from 'vue'
import type { MaybeRefOrGetter } from 'vue'

interface AsyncState<T> {
  data: Ref<T | null>
  error: Ref<Error | null>
  loading: Ref<boolean>
  execute: () => Promise<void>
}

export function useAsyncData<T>(
  fetcher: MaybeRefOrGetter<() => Promise<T>>,
): AsyncState<T> {
  const data = ref<T | null>(null) as Ref<T | null>
  const error = ref<Error | null>(null)
  const loading = ref(false)

  const execute = async () => {
    loading.value = true
    error.value = null
    try {
      data.value = await toValue(fetcher)()
    } catch (e) {
      error.value = e as Error
    } finally {
      loading.value = false
    }
  }

  // 自动执行（响应式触发）
  watchEffect(() => {
    // 使用 toValue 确保 fetcher 是响应式的
    toValue(fetcher)
    execute()
  })

  return { data, error, loading, execute }
}

// 使用示例
// const { data, loading, error } = useAsyncData(() => fetchUserList(route.query.page))
```

## 3. 组合式 Watch（Watch 工厂）

封装 DOM 或第三方库需要的 watch/event 管理，自动清理。

```typescript
// useEventListener.ts
import { onMounted, onUnmounted, type Ref } from 'vue'

export function useEventListener<K extends keyof WindowEventMap>(
  target: EventTarget | Ref<EventTarget | null>,
  event: K,
  handler: (e: WindowEventMap[K]) => void,
  options?: AddEventListenerOptions,
) {
  onMounted(() => {
    const el = target instanceof EventTarget ? target : target?.value
    el?.addEventListener(event, handler as EventListener, options)
  })
  onUnmounted(() => {
    const el = target instanceof EventTarget ? target : target?.value
    el?.removeEventListener(event, handler as EventListener, options)
  })
}

// 使用
// useEventListener(window, 'resize', () => { /* ... */ })
```

## 4. 泛型 Composable（类型安全）

```typescript
// useStorage.ts — 带类型推断的 localStorage 封装
import { ref, watch, type Ref } from 'vue'

export function useStorage<T>(key: string, defaultValue: T): Ref<T> {
  const stored = localStorage.getItem(key)
  const data = ref<T>(stored ? JSON.parse(stored) : defaultValue) as Ref<T>

  watch(data, (newVal) => {
    localStorage.setItem(key, JSON.stringify(newVal))
  }, { deep: true })

  return data
}

// 使用：自动推断类型
// const user = useStorage<User>('current-user', { name: '', age: 0 })
```

## 5. 依赖注入组合（provide/inject 工厂）

```typescript
// useTheme.ts — 跨组件共享主题状态
import { provide, inject, ref, type InjectionKey, type Ref } from 'vue'

type Theme = 'light' | 'dark'

const THEME_KEY: InjectionKey<{
  theme: Ref<Theme>
  toggle: () => void
}> = Symbol('theme')

export function provideTheme() {
  const theme = ref<Theme>('light')
  const toggle = () => {
    theme.value = theme.value === 'light' ? 'dark' : 'light'
  }
  provide(THEME_KEY, { theme, toggle })
}

export function useTheme() {
  const injected = inject(THEME_KEY)
  if (!injected) throw new Error('useTheme() must be used within a component that calls provideTheme()')
  return injected
}
```

## 6. 条件组合（Conditional Composable）

仅在特定条件满足时激活逻辑，避免不必要的 watch/副作用。

```typescript
// usePolling.ts — 条件激活的轮询
import { ref, watch, onUnmounted } from 'vue'

export function usePolling(fn: () => Promise<void>, interval: number, enabled: () => boolean) {
  const timer = ref<ReturnType<typeof setInterval> | null>(null)

  const start = () => {
    if (timer.value) return
    timer.value = setInterval(fn, interval)
  }
  const stop = () => {
    if (timer.value) {
      clearInterval(timer.value)
      timer.value = null
    }
  }

  watch(enabled, (val) => {
    val ? start() : stop()
  }, { immediate: true })

  onUnmounted(stop)

  return { start, stop }
}
```

## 7. 可取消的异步 Composable

```typescript
// useAbortableFetch.ts
import { ref, onUnmounted } from 'vue'

export function useAbortableFetch<T>(url: string) {
  const data = ref<T | null>(null)
  const error = ref<string | null>(null)
  const loading = ref(false)
  const controller = ref<AbortController | null>(null)

  const execute = async () => {
    // 取消上一次未完成的请求
    controller.value?.abort()
    controller.value = new AbortController()

    loading.value = true
    error.value = null
    try {
      const res = await fetch(url, { signal: controller.value.signal })
      data.value = await res.json()
    } catch (e: any) {
      if (e.name !== 'AbortError') {
        error.value = e.message
      }
    } finally {
      loading.value = false
    }
  }

  const abort = () => {
    controller.value?.abort()
  }

  onUnmounted(abort)

  return { data, error, loading, execute, abort }
}
```

## 注意事项

- **Composable 命名**：以 `use` 开头，如 `useUser`, `useAuth`
- **不要在 Composable 中直接修改 DOM**：使用 `onMounted` / `nextTick` 确保模板已渲染
- **清理副作用**：始终在 `onUnmounted` 中清理定时器、事件监听、请求取消
- **输入参数**：使用 `MaybeRefOrGetter<T>` 类型让参数同时支持响应式和静态值
- **避免深层嵌套**：Composable 之间可以互相组合，但避免超过 3 层嵌套
- **测试**：Composable 可以脱离组件测试，用 `withSetup()` 辅助函数
