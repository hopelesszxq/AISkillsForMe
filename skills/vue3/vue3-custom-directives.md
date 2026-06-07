---
name: vue3-custom-directives
description: Vue3 自定义指令深度实践：权限控制、水印、懒加载、防抖节流、点击外部、拖拽排序
tags: [vue3, directives, composable, permission, lazy-load, debounce]
---

## 概述

Vue3 自定义指令（Custom Directives）是**直接操作 DOM** 的首选方式。相比 Vue2，Vue3 指令生命周期更贴近组件生命周期（mounted → updated → unmounted），同时支持在 `<script setup>` 中以 `vXxx` 格式局部注册。

> 应用场景：权限控制、防抖节流、图片懒加载、点击外部、拖拽、水印、自动聚焦、滚动加载更多等。

## 基础结构与生命周期

```typescript
// 完整的指令生命周期
const vMyDirective: Directive = {
  // 属性绑定或事件监听器被应用到元素之前调用
  created(el, binding, vnode, prevVnode) {},
  // 元素被插入到 DOM 中前调用
  beforeMount(el, binding, vnode, prevVnode) {},
  // 元素被插入到 DOM（父组件挂载）后调用 —— 最常用
  mounted(el, binding, vnode, prevVnode) {},
  // 组件的 VNode 更新前调用
  beforeUpdate(el, binding, vnode, prevVnode) {},
  // 组件的 VNode 更新后调用
  updated(el, binding, vnode, prevVnode) {},
  // 元素从 DOM 移除前调用 —— 清理资源
  beforeUnmount(el, binding, vnode, prevVnode) {},
  // 元素从 DOM 移除后调用
  unmounted(el, binding, vnode, prevVnode) {},
};

// binding 对象详解
// binding.value   — 指令的值（如 v-permission="'admin'"）
// binding.oldValue — 更新前的值
// binding.arg     — 指令参数（如 v-tooltip:top 的 'top'）
// binding.modifiers — 修饰符对象（如 v-lazy.once 的 { once: true }）
```

## 1. 权限控制指令（实战最多）

```typescript
// directives/permission.ts
import type { Directive } from 'vue';
import { usePermissionStore } from '@/stores/permission';

export const vPermission: Directive<HTMLElement, string | string[]> = {
  mounted(el, binding) {
    const store = usePermissionStore();
    const permissions = store.permissions; // 当前用户权限列表

    const required = Array.isArray(binding.value)
      ? binding.value
      : [binding.value];

    const hasPermission = required.some(p => permissions.includes(p));

    if (!hasPermission) {
      // 策略1：移除元素
      el.parentNode?.removeChild(el);

      // 策略2：禁用（适用于按钮）
      // el.setAttribute('disabled', 'disabled');
      // el.classList.add('disabled');

      // 策略3：隐藏
      // el.style.display = 'none';
    }
  },
  updated(el, binding) {
    // 权限变化时重新检查
    // 实现同 mounted（略）
  },
};
```

```vue
<!-- 使用 -->
<template>
  <button v-permission="'system:user:create'">新增用户</button>
  <button v-permission="['admin', 'superadmin']">管理控制台</button>
</template>

<script setup lang="ts">
import { vPermission } from '@/directives/permission';
</script>
```

## 2. 点击外部指令（Dropdown/Modal 关闭）

```typescript
// directives/clickOutside.ts
import type { Directive } from 'vue';

type DocumentHandler = (e: MouseEvent | TouchEvent) => void;

export const vClickOutside: Directive<HTMLElement, () => void> = {
  mounted(el, binding) {
    const handler: DocumentHandler = (e) => {
      if (!el.contains(e.target as Node) && el !== e.target) {
        binding.value?.(); // 执行回调
      }
    };
    // 存储到 el 上便于 unmounted 时清理
    (el as any)._clickOutsideHandler = handler;
    document.addEventListener('click', handler, false);
  },
  unmounted(el) {
    const handler = (el as any)._clickOutsideHandler;
    if (handler) {
      document.removeEventListener('click', handler, false);
    }
  },
};
```

```vue
<template>
  <div v-click-outside="closeDropdown" class="dropdown">
    <button @click="open = !open">菜单</button>
    <div v-if="open" class="menu">下拉内容</div>
  </div>
</template>
```

## 3. 防抖节流指令

```typescript
// directives/debounce.ts
import type { Directive } from 'vue';

export const vDebounce: Directive<HTMLElement, { fn: Function; delay?: number }> = {
  mounted(el, binding) {
    const { fn, delay = 300 } = binding.value;
    let timer: ReturnType<typeof setTimeout>;

    el.addEventListener('input', (e: Event) => {
      clearTimeout(timer);
      timer = setTimeout(() => fn(e), delay);
    });
  },
};
```

```vue
<template>
  <input
    v-debounce="{ fn: search, delay: 500 }"
    placeholder="搜索..."
  />
</template>
```

## 4. 图片懒加载（IntersectionObserver）

```typescript
// directives/lazyLoad.ts
import type { Directive } from 'vue';

export const vLazyLoad: Directive<HTMLImageElement, string> = {
  mounted(el, binding) {
    // 默认占位图
    el.src = 'data:image/svg+xml,...';

    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting) {
          el.src = binding.value; // 真实图片地址
          observer.unobserve(el);
        }
      },
      { rootMargin: '200px' } // 提前 200px 加载
    );

    observer.observe(el);
    (el as any)._lazyObserver = observer;
  },
  unmounted(el) {
    (el as any)._lazyObserver?.disconnect();
  },
};
```

```vue
<template>
  <img v-lazy-load="'https://example.com/huge-image.jpg'" alt="懒加载图片" />
</template>
```

## 5. 水印指令

```typescript
// directives/watermark.ts
import type { Directive } from 'vue';

interface WatermarkOptions {
  text: string;
  color?: string;
  fontSize?: number;
  opacity?: number;
  rotate?: number;
}

export const vWatermark: Directive<HTMLElement, WatermarkOptions> = {
  mounted(el, binding) {
    const {
      text = '内部资料',
      color = '#000',
      fontSize = 14,
      opacity = 0.05,
      rotate = -25,
    } = binding.value || {};

    const canvas = document.createElement('canvas');
    canvas.width = 200;
    canvas.height = 150;
    const ctx = canvas.getContext('2d')!;

    ctx.clearRect(0, 0, 200, 150);
    ctx.fillStyle = color;
    ctx.globalAlpha = opacity;
    ctx.font = `${fontSize}px serif`;
    ctx.rotate((rotate * Math.PI) / 180);
    ctx.fillText(text, 10, 80);

    const bg = document.createElement('div');
    bg.style.position = 'absolute';
    bg.style.inset = '0';
    bg.style.pointerEvents = 'none';
    bg.style.zIndex = '9999';
    bg.style.backgroundImage = `url(${canvas.toDataURL()})`;
    bg.style.backgroundRepeat = 'repeat';

    el.style.position = 'relative';
    el.appendChild(bg);
  },
};
```

```vue
<template>
  <div v-watermark="{ text: '张三 - admin@example.com', opacity: 0.03 }">
    <!-- 受保护的内部内容 -->
  </div>
</template>
```

## 6. 拖拽排序指令

```typescript
// directives/draggable.ts
import type { Directive } from 'vue';

export const vDraggable: Directive<HTMLElement, (oldIndex: number, newIndex: number) => void> = {
  mounted(el, binding) {
    let dragIndex: number;

    el.querySelectorAll('.drag-item').forEach((item, index) => {
      item.setAttribute('draggable', 'true');
      item.addEventListener('dragstart', () => { dragIndex = index; });
      item.addEventListener('dragover', (e) => e.preventDefault());
      item.addEventListener('drop', () => {
        binding.value?.(dragIndex, index);
      });
    });
  },
};
```

## 7. 全局注册

```typescript
// directives/index.ts
import type { App } from 'vue';
import { vPermission } from './permission';
import { vClickOutside } from './clickOutside';
import { vDebounce } from './debounce';
import { vLazyLoad } from './lazyLoad';
import { vWatermark } from './watermark';
import { vDraggable } from './draggable';

export function setupDirectives(app: App) {
  app.directive('permission', vPermission);
  app.directive('click-outside', vClickOutside);
  app.directive('debounce', vDebounce);
  app.directive('lazy-load', vLazyLoad);
  app.directive('watermark', vWatermark);
  app.directive('draggable', vDraggable);
}

// main.ts
// import { setupDirectives } from '@/directives';
// setupDirectives(app);
```

## 注意事项

### 与 Composition API 的取舍
- **指令**：适合单纯 DOM 操作（focus、style、attribute、event）
- **Composable**：适合需要响应式状态的逻辑（如 useMouse、useFetch）
- **混合使用**：指令内部也可以调用 composable：

```typescript
mounted(el, binding) {
  const { width, height } = useWindowSize(); // ✅ 指令内使用 composable
  // ...
}
```

### Vue 3.5+ 指令增强
- `mounted` 和 `updated` 生命周期优化，减少不必要的调用
- 更稳定的 `binding.oldValue` 追踪
- `<Teleport>` 内的指令更可靠地触发 `unmounted`

### 常见陷阱
1. **清理资源**：addEventListener/observer 必须在 `unmounted` 中移除，否则内存泄漏
2. **服务端渲染（SSR）**：指令只在客户端运行，SSR 会跳过 `mounted` 阶段
3. **避免过度使用**：能用模板语法解决的就不要用指令
4. **类型安全**：始终为 `Directive<T>` 指定泛型类型
