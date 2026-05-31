---
name: vue3-lottie-animation
description: Vue 3 集成 Lottie 动画的完整指南：dotlottie-vue 和 lottie-web 两种方案，含交互触发和播放控制
tags: [vue3, animation, lottie, frontend, ui]
---

## 要点

Lottie 是向 Vue 3 应用添加轻量级、高性能动画的首选方案。JSON/DotLottie 格式体积小，支持矢量缩放，不会像 GIF 一样模糊。

Vue 3 有两种主流的 Lottie 集成方案：

| 方案 | 包体积 | 格式支持 | 推荐场景 |
|------|--------|---------|----------|
| `@lottiefiles/dotlottie-vue` | ~100KB | .lottie, .json | ✅ 通用推荐 |
| `lottie-web` | ~500KB | .json | 需精细控制帧/片段/事件 |

## 方案一：@lottiefiles/dotlottie-vue（推荐）

### 安装

```bash
npm install @lottiefiles/dotlottie-vue
```

### 基础用法

```vue
<template>
  <DotLottieVue
    src="/animations/loader.lottie"
    :loop="true"
    :autoplay="true"
    style="width: 200px; height: 200px"
  />
</template>

<script setup>
import { DotLottieVue } from '@lottiefiles/dotlottie-vue'
</script>
```

支持 `.lottie`（压缩格式）和 `.json` 文件。对于 `.json` 文件，使用同样的 `src` prop 指定路径。

### 程序化播放控制

```vue
<template>
  <div class="animation-wrapper">
    <DotLottieVue
      ref="lottieRef"
      src="/animations/check.lottie"
      :loop="false"
      :autoplay="false"
    />
    <button @click="play">▶ 播放</button>
    <button @click="pause">⏸ 暂停</button>
    <button @click="stop">⏹ 停止</button>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { DotLottieVue } from '@lottiefiles/dotlottie-vue'

const lottieRef = ref(null)

function play()   { lottieRef.value?.play() }
function pause()  { lottieRef.value?.pause() }
function stop()   { lottieRef.value?.stop() }
</script>
```

## 方案二：lottie-web（精细控制）

需要控制帧位置、片段播放或监听复杂事件时使用。

### 安装

```bash
npm install lottie-web
```

### 基础用法（记得销毁！）

```vue
<template>
  <div ref="container" class="lottie-container" />
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import lottie from 'lottie-web'

const container = ref(null)
let anim = null

onMounted(() => {
  anim = lottie.loadAnimation({
    container: container.value,
    renderer: 'svg',          // 'svg' | 'canvas' | 'html'
    loop: true,
    autoplay: true,
    path: '/animations/loader.json',
  })
})

// 必须销毁，否则内存泄漏！
onBeforeUnmount(() => {
  anim?.destroy()
})
</script>

<style scoped>
.lottie-container {
  width: 200px;
  height: 200px;
}
</style>
```

### 用户交互触发动画

```vue
<template>
  <div>
    <div ref="container" class="lottie-icon" />
    <button @click="handleSubmit" :disabled="submitting">
      {{ submitting ? '提交中...' : '提交' }}
    </button>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import lottie from 'lottie-web'
import successData from '@/assets/success.json'  // 直接 import JSON

const container = ref(null)
const submitting = ref(false)
let anim = null

onMounted(() => {
  anim = lottie.loadAnimation({
    container: container.value,
    renderer: 'svg',
    loop: false,
    autoplay: false,
    animationData: successData,
  })
})

function handleSubmit() {
  submitting.value = true
  // 模拟请求
  setTimeout(() => {
    submitting.value = false
    anim?.goToAndPlay(0)  // 从头播放一次
  }, 1500)
}

onBeforeUnmount(() => anim?.destroy())
</script>
```

### 全局注册（可选）

如果多处使用，可全局注册组件：

```ts
// main.ts
import { createApp } from 'vue'
import { DotLottieVue } from '@lottiefiles/dotlottie-vue'

const app = createApp(App)
app.component('DotLottieVue', DotLottieVue)
app.mount('#app')
```

## 免费动画资源

| 来源 | 格式 | 特点 |
|------|------|------|
| [LottieFiles](https://lottiefiles.com) | .json, .lottie | 最大社区，免费+付费 |
| [IconScout](https://iconscout.com) | .json, .lottie | 图标为主 |
| [Lordicon](https://lordicon.com) | .json | 高质量图标动画 |

## 注意事项

- **内存泄漏**：`lottie-web` 使用时必须在 `onBeforeUnmount` 中执行 `anim.destroy()`，否则组件卸载后动画仍在运行
- **包体积**：`lottie-web` 约 500KB（min+gzip ~150KB），`dotlottie-vue` ~100KB。移动端优先选后者
- **SSR 兼容**：两个库都依赖浏览器 DOM API，在 Nuxt 中需客户端渲染
  ```vue
  <ClientOnly>
    <DotLottieVue src="/animations/loader.lottie" />
  </ClientOnly>
  ```
- **性能**：大量动画同时播放时（如列表每项都有动画），使用 `renderer: 'canvas'` 替代默认的 `'svg'`
- **JSON 导入**：使用 Vite 时可以直接 `import data from '@/assets/anim.json'`，Vite 自动处理
