---
name: vue3-alien-signals-reactivity
description: Vue 3.6 响应式系统重构——基于 alien-signals 的 push-pull 信号算法，性能提升 2-5 倍
tags: [vue3, reactivity, signals, performance, alien-signals, composition-api]
---

## 概述

Vue 3.6（当前最新 v3.6.0-beta.13，2026-05-28）对 `@vue/reactivity` 进行了**重大重构**，采用基于 [alien-signals](https://github.com/stackblitz/alien-signals) 的 **push-pull（推拉）信号算法**。这是 Vue 响应式系统自 3.0 以来的最大底层变更，由尤雨溪本人实现——他也是 alien-signals 的核心作者。

> alien-signals 是一个独立的信号库（npm: `alien-signals`，GitHub Stars: 3K+），它探索了一种 push-pull 混合的信号传播算法，结合了 Vue 3.4 的传播机制、Preact 的双链表信号、Svelte 的效果调度和 Reactively 的图着色策略。

## 算法演进历史

```
Vue 3.0-3.3: 基于 Proxy 的深度响应式，Pull-based（拉模式）检查
Vue 3.4:    ✨ 响应式优化（PR #5912），减少不必要的 re-render
Vue 3.5:    切换到 Pull-based 类似 Preact 信号（PR #10397）
Vue 3.6:    ✨ Push-Pull 混合算法（基于 alien-signals），性能飞跃提升
```

## Push-Pull 信号算法原理

### 传统 Pull 模式（Vue 3.5）

```
组件更新 → 遍历依赖（深度遍历）→ 逐个检查值是否变化 → 判断是否更新
```

每次组件 re-render 时，需要遍历所有依赖并逐个 Pull 值判断是否变化，存在 O(n) 的检查开销。

### Push-Pull 模式（Vue 3.6）

```
值变化 → Push 通知所有依赖（值+脏标记）→ 消费时 Pull 最新值
```

值变化时**主动推送变更信号**（Push），仅在消费时按需拉取最新值（Pull），大幅减少了中间检查次数。

### 核心优势

| 维度 | Vue 3.5 (Pull) | Vue 3.6 (Push-Pull) | 提升 |
|------|----------------|---------------------|------|
| 创建 1M 信号 | ~120ms | ~40ms | **3x** |
| 更新 1M 订阅 | ~200ms | ~40ms | **5x** |
| 内存占用 | 基准 | -50% | **2x** |
| 弱依赖检查 | O(n) | O(1) | **大幅优化** |

> 数据来源：js-reactivity-benchmark（transitive-bullshit/reactivity-benchmark）

## 对开发者的影响

### 完全向后兼容

**这是底层实现重构，API 保持不变**。无需修改现有代码：

```typescript
import { ref, computed, watchEffect } from 'vue'

// 代码无需任何改动
const count = ref(0)
const doubled = computed(() => count.value * 2)

watchEffect(() => {
    console.log(`count: ${count.value}, doubled: ${doubled.value}`)
})

count.value++  // 自动触发更新
```

### 提升场景

以下场景**无需额外优化**即可获得显著性能提升：

```typescript
// 1. 大型表单（数千个响应式字段）
const form = reactive({
    field1: '', field2: '', /* ... 数百个字段 */
})

// 2. 复杂计算属性链
const a = ref(1)
const b = computed(() => a.value * 2)
const c = computed(() => b.value + 1)
const d = computed(() => c.value * 3)
// 更新 a 时，只需 Push 一次，链式推导自动完成

// 3. 高频数据更新（实时仪表盘、股价、游戏状态）
const positions = reactive(new Map<string, Position>())

function batchUpdate(updates: Array<{ id: string, pos: Position }>) {
    // Push 通知高效处理批量更新
    updates.forEach(({ id, pos }) => positions.set(id, pos))
}
```

### 调试体验提升

新算法使得 `effect` 的可视化追踪更加清晰：

```typescript
// Vue DevTools 中 effect 依赖图的展示更加准确
// 因为 Push 通知会携带明确的依赖关系链
```

## alien-signals 独立使用

如果你不在 Vue 中，也可以直接使用 alien-signals 库：

```bash
npm install alien-signals
```

```typescript
import { signal, computed, effect } from 'alien-signals'

const count = signal(0)
const doubled = computed(() => count() * 2)

effect(() => {
    console.log(`count: ${count()}, doubled: ${doubled()}`)
})

count(1)  // 触发 effect
```

## 注意事项

1. **Vue 3.6 仍处于 Beta**（最新 v3.6.0-beta.13），alien-signals 重构已在 beta.1 中合并，建议在测试环境验证
2. **不存在 Breaking Changes**：这是一个纯内部实现重构，对外 API 完全不变
3. **Vapor Mode 搭配效率倍增**：Vapor Mode（非 VDOM 编译）配合 Push-Pull 响应式系统，性能可达 Solid 和 Svelte 5 同等水平
4. **watch 和 computed 行为不变**：所有 `watch`、`watchEffect`、`computed`、`ref`、`reactive`、`shallowRef` 等 API 行为完全一致
5. **TypeScript 类型不变**：`@vue/reactivity` 的 TS 类型导出无变化
6. **第三方响应式库**：在 Vue 3.6 中集成 `vue-reactivity` 的第三方库无需修改代码
7. **基准测试参考**：具体性能提升幅度取决于实际使用场景，极端数据量场景（10K+ 信号）提升最为明显
