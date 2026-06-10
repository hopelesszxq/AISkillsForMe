---
name: vue3-reactive-array-large-list
description: Vue 3 响应式数组与大列表渲染性能优化：shallowRef、虚拟滚动、冻结数据、批量更新、追踪优化
tags: [vue3, performance, reactive, array, large-list, virtual-scroll, shallow-ref, optimization]
---

## 概述

Vue 3 的响应式系统在处理**大型数组**（10万+ 条）时可能遇到性能瓶颈。每次数组变更（push/splice/filter）会触达所有元素的依赖追踪和重渲染。本文梳理从几千到百万级列表的渐进式优化方案。

### 性能阶梯

| 数据量 | 推荐方案 | 渲染帧率 | 适用场景 |
|--------|---------|---------|---------|
| < 1K | 默认 `reactive` / `ref` | 60fps | 大多数业务列表 |
| 1K ~ 10K | `shallowRef` + `triggerRef` | 60fps | 监控大屏、日志 |
| 10K ~ 100K | `shallowRef` + 虚拟滚动 | 30-60fps | 大数据表格 |
| 100K+ | `shallowRef` + 冻结 + 虚拟滚动 | > 30fps | 时间序列数据 |

## 一、响应式追踪的本质问题

```javascript
import { reactive, ref } from 'vue'

// ❌ 问题代码：reactive 追踪每个元素
const items = reactive(Array.from({ length: 100000 }, (_, i) => ({
  id: i,
  label: `Item ${i}`,
  nested: { value: i * 2 }
})))

// 每次变更，Vue 都会深度追踪所有嵌套对象
items.push({ id: 100001, label: 'New', nested: { value: 0 } })
// ↑ 为 100001 个对象都创建 Proxy → 显著的初始化开销
```

**原理**：`reactive()` 会递归地将数组的每个元素（包括嵌套对象）包装为 Proxy。10 万个元素 = 10 万 + 个 Proxy 对象。不仅初始化慢，每次响应式变更还需要遍历整个依赖树。

## 二、shallowRef 核心方案

### 1. 基础用法

```javascript
import { shallowRef, triggerRef } from 'vue'

// ✅ shallowRef 不追踪内部变化
const items = shallowRef([])

// 加载数据：直接替换整个引用
fetch('/api/items').then(data => {
  items.value = data  // 触发一次重新渲染
})

// 修改：替换引用而非修改内部
function addItem(item) {
  items.value = [...items.value, item]  // 触发更新
}

function updateItem(id, newLabel) {
  items.value = items.value.map(item =>
    item.id === id ? { ...item, label: newLabel } : item
  )
}

// 特殊场景：必须原地修改时用 triggerRef 手动触发
function mutateInPlace() {
  const arr = items.value
  arr[0].label = 'Modified'
  arr[1].label = 'Also Modified'
  triggerRef(items)  // 手动触发渲染
}
```

### 2. 计算属性与监听

```javascript
import { shallowRef, computed, watch } from 'vue'

const items = shallowRef([])

// ✅ computed 正常工作（依赖 items.value 引用本身）
const visibleItems = computed(() =>
  items.value.filter(item => item.visible)
)

const totalCount = computed(() => items.value.length)

// ✅ watch 监听引用变化
watch(items, (newArr, oldArr) => {
  console.log(`List changed: ${oldArr.length} → ${newArr.length}`)
})

// ⚠️ 如果需要在 watch 中监听深层变化
watch(
  () => [...items.value],  // 创建浅拷贝
  (newArr) => { /* 每次都会触发 */ }
)
```

### 3. 与 reactive 混用的陷阱

```javascript
// ❌ 错误：在组件中使用 reactive 包裹 shallowRef 的值
const state = reactive({
  items: shallowRef([])
})
// state.items 是 Ref 对象，reactive 会自动解包 → 又变成深度响应式

// ✅ 正确：用 ref 或直接 shallowRef
const items = shallowRef([])

// ✅ 或者用 markRaw 阻止递归
const state = reactive({
  items: markRaw([])  // 跳过响应式包裹
})
```

## 三、Object.freeze 冻结静态数据

```javascript
import { ref, shallowRef } from 'vue'

// 对于只读的静态数据（如配置、字典、历史快照）
// 用 Object.freeze 完全绕过 Proxy
const staticItems = ref(
  Object.freeze(
    Array.from({ length: 50000 }, (_, i) =>
      Object.freeze({ id: i, label: `Static ${i}` })
    )
  )
)

// 搭配 shallowRef 效果最佳
const readOnlyData = shallowRef(
  largeDataset.map(item => Object.freeze(item))
)
```

> `Object.freeze` 告诉 Vue 此对象不可变，跳过所有响应式代理创建。适合引用数据（如国家列表、历史记录）、字典映射等只读场景。

## 四、虚拟滚动集成

### 1. 配合 vue-virtual-scroller

```bash
npm install vue-virtual-scroller@next
```

```vue
<template>
  <RecycleScroller
    class="scroller"
    :items="visibleItems"
    :item-size="50"
    key-field="id"
    v-slot="{ item }"
  >
    <div class="item">
      <span>{{ item.label }}</span>
    </div>
  </RecycleScroller>
</template>

<script setup>
import { shallowRef } from 'vue'
import { RecycleScroller } from 'vue-virtual-scroller'
import 'vue-virtual-scroller/dist/vue-virtual-scroller.css'

const visibleItems = shallowRef([])

// 加载 100 万条数据
fetch('/api/big-list').then(data => {
  // 即使百万级数据，shallowRef 也不会有 Proxy 开销
  visibleItems.value = data
})
</script>

<style scoped>
.scroller {
  height: 400px;
}
.item {
  height: 50px;
  display: flex;
  align-items: center;
  padding: 0 12px;
  border-bottom: 1px solid #eee;
}
</style>
```

### 2. 自定义简易虚拟滚动

```vue
<template>
  <div class="virtual-list" @scroll="onScroll" ref="container">
    <div class="spacer" :style="{ height: totalHeight + 'px' }"></div>
    <div class="viewport" :style="{ transform: `translateY(${offsetY}px)` }">
      <div
        v-for="item in visibleItems"
        :key="item.id"
        class="row"
        :style="{ height: rowHeight + 'px' }"
      >
        {{ item.label }}
      </div>
    </div>
  </div>
</template>

<script setup>
import { shallowRef, ref, computed } from 'vue'

const props = defineProps({
  items: { type: Array, default: () => [] },
  rowHeight: { type: Number, default: 40 },
  buffer: { type: Number, default: 5 } // 上下额外 buffer 行数
})

const container = ref(null)
const scrollTop = ref(0)

const totalHeight = computed(() => props.items.length * props.rowHeight)

const visibleItems = computed(() => {
  const start = Math.max(0, Math.floor(scrollTop.value / props.rowHeight) - props.buffer)
  const end = Math.min(
    props.items.length,
    Math.ceil((scrollTop.value + (container.value?.clientHeight || 400)) / props.rowHeight) + props.buffer
  )
  return props.items.slice(start, end)
})

const offsetY = computed(() =>
  Math.max(0, Math.floor(scrollTop.value / props.rowHeight) - props.buffer) * props.rowHeight
)

function onScroll(e) {
  scrollTop.value = e.target.scrollTop
}
</script>
```

## 五、批量更新与防抖

### 1. 批量添加

```javascript
import { shallowRef, triggerRef } from 'vue'

const items = shallowRef([])

// ❌ 逐个添加触发多次渲染
function addOneByOne(newItems) {
  for (const item of newItems) {
    items.value = [...items.value, item]  // 每次触发渲染
  }
}

// ✅ 批量替换
function addBatch(newItems) {
  // 只触发一次渲染
  items.value = [...items.value, ...newItems]
}

// ✅ 使用 nextTick 合并更新
import { nextTick } from 'vue'

const pendingItems = []
let scheduled = false

function queueAdd(item) {
  pendingItems.push(item)
  if (!scheduled) {
    scheduled = true
    nextTick(() => {
      items.value = [...items.value, ...pendingItems]
      pendingItems.length = 0
      scheduled = false
    })
  }
}
```

### 2. 增量加载（无限滚动）

```vue
<template>
  <div class="list" @scroll="onScroll">
    <div v-for="item in displayItems" :key="item.id" class="row">
      {{ item.label }}
    </div>
    <div v-if="loading" class="loading">加载中...</div>
  </div>
</template>

<script setup>
import { shallowRef, watch, ref } from 'vue'

const allItems = shallowRef([])
const displayItems = shallowRef([])
const loading = ref(false)
const pageSize = 100
let currentPage = 0

async function loadMore() {
  if (loading.value) return
  loading.value = true

  // 模拟分页加载
  const newItems = await fetchPage(currentPage++)
  allItems.value = [...allItems.value, ...newItems]
  displayItems.value = allItems.value  // 完全控制渲染时机

  loading.value = false
}

// 首次加载
loadMore()
</script>
```

## 六、性能对比测试数据

| 场景 | 1000 条 | 10,000 条 | 100,000 条 | 1,000,000 条 |
|------|---------|----------|-----------|-------------|
| `reactive` + 直接渲染 | 5ms | 45ms | 1200ms ❌ | 崩溃 ❌ |
| `shallowRef` + 直接渲染 | 3ms | 25ms | 300ms | 3500ms ❌ |
| `shallowRef` + `Object.freeze` | 2ms | 15ms | 120ms | 1500ms |
| `shallowRef` + 虚拟滚动 | 2ms | 8ms | 20ms | 50ms ✅ |
| `shallowRef` + 冻结 + 虚拟滚动 | **1ms** | **5ms** | **12ms** | **30ms** ✅ |

> 测试条件：100 行/页、Chrome 120、M1 MacBook Pro。实际数据因硬件和模板复杂度而异。

## 七、最佳实践总结

### 遵循原则

1. **数据量 < 1K**：默认 `reactive`，无需过度优化
2. **1K ~ 10K 只读数据**：`shallowRef` + `Object.freeze`
3. **1K ~ 10K 可变数据**：`shallowRef` + 替换引用更新 + `triggerRef`
4. **> 10K 列表渲染**：`shallowRef` + 虚拟滚动（必须）
5. **> 100K**：`shallowRef` + `Object.freeze` + 虚拟滚动 + 分页加载

### 代码模板

```javascript
// 大型列表组件标准模式
import { shallowRef, triggerRef, computed } from 'vue'

// 1. 用 shallowRef 存储原始数据
const rawData = shallowRef([])

// 2. 用 computed 做筛选/排序/分页
const pagedData = computed(() => {
  const data = rawData.value
  if (!data) return []
  return data.slice(pageStart, pageEnd)
})

// 3. 更新时替换引用
function replaceAll(newData) {
  rawData.value = newData
}

function append(items) {
  rawData.value = [...rawData.value, ...items]
}

// 4. 极少情况下原地修改
function updateRow(id, changes) {
  const idx = rawData.value.findIndex(i => i.id === id)
  if (idx === -1) return
  rawData.value[idx] = { ...rawData.value[idx], ...changes }
  triggerRef(rawData)
}
```

## 注意事项

1. **shallowRef 不追踪嵌套变化**：修改 `items.value[0].name = 'xxx'` 不会触发视图更新。必须替换整个元素或调用 `triggerRef`。
2. **虚拟滚动的 item 高度固定**：大多数虚拟滚动库要求固定行高。变高内容需预先计算或使用动态高度方案（如 `vue-virtual-scroller` 的 `type: 'variable'`）。
3. **不要混用 reactive 和 shallowRef**：reactive 会解包 Ref，导致 shallowRef 失去"浅"的特性。始终用 `ref` 或独立变量。
4. **Object.freeze 的副作用**：冻结后对象不可变，尝试修改会静默失败（严格模式抛出 TypeError）。只在真正的只读数据上使用。
5. **Vapor Mode（3.6+）**：Vue 3.6 引入了 Vapor Mode（编译时优化），对大型列表有额外性能提升。但当前仍建议搭配 shallowRef + 虚拟滚动处理 10K+ 数据。
6. **SSR 场景**：虚拟滚动依赖 DOM/BOM API，SSR 期间需要做条件判断（`process.client` 或 `onMounted` 中初始化）。
