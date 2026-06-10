---
name: vue3-i18n-internationalization
description: Vue 3 国际化 (vue-i18n 9.x/10.x) 最佳实践：多语言切换、Lazy-loading、TypeScript 类型安全、SSR 兼容
tags: [vue3, i18n, internationalization, vue-i18n, localization, composition-api]
---

## 概述

vue-i18n 是 Vue 生态中最成熟的国际化方案。Vue 3 对应 vue-i18n 9.x（Vue I18n IETF），最新版本已进入 10.x，新增了 Lazy Message Loading、TypeScript 类型推导、Message Format 改进等特性。

## 一、基础安装与配置

```bash
npm install vue-i18n@latest
```

### Vite 项目配置

```typescript
// src/i18n/index.ts
import { createI18n } from 'vue-i18n'
import type { I18nOptions } from 'vue-i18n'

// 语言列表定义
export type LocaleType = 'zh-CN' | 'en-US' | 'ja-JP'

export const SUPPORTED_LOCALES: LocaleType[] = ['zh-CN', 'en-US', 'ja-JP']

const i18n = createI18n<I18nOptions, LocaleType, false>({
  legacy: false,       // 使用 Composition API 模式
  locale: 'zh-CN',     // 默认语言
  fallbackLocale: 'en-US',
  globalInjection: true,
  messages: {
    'zh-CN': {
      hello: '你好',
      welcome: '欢迎来到 {name}',
    },
    'en-US': {
      hello: 'Hello',
      welcome: 'Welcome to {name}',
    },
  },
})

export default i18n
```

```typescript
// main.ts
import { createApp } from 'vue'
import i18n from './i18n'

const app = createApp(App)
app.use(i18n)
app.mount('#app')
```

## 二、Composition API 使用模式

### 模板中使用

```vue
<template>
  <p>{{ t('hello') }}</p>
  <p>{{ t('welcome', { name: 'Alice' }) }}</p>
  <!-- 复数形式 -->
  <p>{{ n(10, 'currency') }}</p>
  <p>{{ d(new Date(), 'short') }}</p>
</template>

<script setup lang="ts">
import { useI18n } from 'vue-i18n'

const { t, n, d, locale, availableLocales } = useI18n()
</script>
```

### 非模板/JS 中使用

```typescript
import { useI18n } from 'vue-i18n'

function useLocaleHelper() {
  const { t, locale } = useI18n()

  function switchLocale(newLocale: LocaleType) {
    locale.value = newLocale
    document.documentElement.lang = newLocale
    // 可选：持久化到 localStorage
    localStorage.setItem('locale', newLocale)
  }

  function validateMessage(key: string): boolean {
    return t(key) !== key // 未找到时返回 key 本身
  }

  return { switchLocale, validateMessage }
}
```

## 三、TypeScript 类型安全

vue-i18n 10.x 支持基于资源文件的类型推导，编译时校验 key 有效性。

```typescript
// src/i18n/resource.ts
export default {
  'zh-CN': {
    nav: {
      home: '首页',
      about: '关于我们',
      contact: '联系方式',
    },
    error: {
      notFound: '页面未找到',
      serverError: '服务器错误',
    },
  },
  'en-US': {
    nav: {
      home: 'Home',
      about: 'About Us',
      contact: 'Contact',
    },
    error: {
      notFound: 'Page Not Found',
      serverError: 'Server Error',
    },
  },
} as const

// src/i18n/typed-i18n.ts
import { defineI18n } from 'vue-i18n'
import messages from './resource'

// 自动推导 key 类型：'nav.home' | 'nav.about' | 'nav.contact' | 'error.notFound' | 'error.serverError'
type MessageKeys = {
  [K in keyof typeof messages['zh-CN']]: 
    typeof messages['zh-CN'][K] extends Record<string, string>
      ? `${K & string}.${keyof typeof messages['zh-CN'][K] & string}`
      : K
}[keyof typeof messages['zh-CN']]

// defineI18n 提供完整类型推导
export default defineI18n({
  legacy: false,
  locale: 'zh-CN',
  messages,
})
```

## 四、异步 Lazy Loading 消息

大型应用按需加载语言包，避免首屏加载所有翻译资源。

```typescript
// src/i18n/lazy.ts
import { nextTick } from 'vue'
import { createI18n } from 'vue-i18n'

export type LocaleModule = Record<string, () => Promise<{ default: Record<string, unknown> }>>

const localeModules: LocaleModule = {
  'zh-CN': () => import('./locales/zh-CN.json'),
  'en-US': () => import('./locales/en-US.json'),
  'ja-JP': () => import('./locales/ja-JP.json'),
}

const i18n = createI18n({
  legacy: false,
  locale: 'zh-CN',
  fallbackLocale: 'en-US',
  messages: {}, // 初始为空
})

// 动态加载语言包
export async function loadLocaleMessages(locale: string): Promise<void> {
  if (!i18n.global.availableLocales.includes(locale)) {
    const messages = await localeModules[locale]?.()
    if (messages) {
      i18n.global.setLocaleMessage(locale, messages.default)
    }
  }

  // 切换语言后等待 DOM 更新
  i18n.global.locale.value = locale as any
  await nextTick()
  document.querySelector('html')?.setAttribute('lang', locale)
}

export default i18n
```

## 五、组件级本地化

大型组件库中，每个组件可携带自己的翻译资源，通过 `$t` 或 `useI18n({ messages })` 注入。

```vue
<script setup lang="ts">
import { useI18n } from 'vue-i18n'

const componentMessages = {
  'zh-CN': { submit: '提交', cancel: '取消', loading: '加载中...' },
  'en-US': { submit: 'Submit', cancel: 'Cancel', loading: 'Loading...' },
}

const { t } = useI18n({ messages: componentMessages, useScope: 'local' })
</script>

<template>
  <button>{{ t('submit') }}</button>
  <button>{{ t('cancel') }}</button>
</template>
```

> 注意: `useScope: 'local'` 使消息仅在该组件及子组件中可见，不会污染全局消息空间。

## 六、ICU MessageFormat 高级用法

vue-i18n 支持 ICU 消息格式（需安装 `@intlify/message-compiler` 或 `@intlify/vue-i18n/message-resolver`）。

```typescript
messages: {
  'en-US': {
    items_count: '{count} {count, plural, =0 {items} one {item} other {items}}',
    gender: '{gender, select, male {He} female {She} other {They}} joined.',
    nested: 'Value: {nested, number, ::currency/USD}',
  },
}
```

```vue
<template>
  <p>{{ t('items_count', { count: 0 }) }}</p>   <!-- 0 items -->
  <p>{{ t('items_count', { count: 1 }) }}</p>   <!-- 1 item -->
  <p>{{ t('items_count', { count: 5 }) }}</p>   <!-- 5 items -->
  <p>{{ t('gender', { gender: 'male' }) }}</p>  <!-- He joined. -->
</template>
```

## 七、SSR（Nuxt）兼容

### Nuxt 3/4 中配置

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxtjs/i18n'],
  i18n: {
    locales: [
      { code: 'zh-CN', file: 'zh-CN.json', name: '中文' },
      { code: 'en-US', file: 'en-US.json', name: 'English' },
    ],
    lazy: true,
    langDir: 'locales/',
    defaultLocale: 'zh-CN',
    detectBrowserLanguage: {
      useCookie: true,
      cookieKey: 'i18n_redirected',
      redirectOn: 'root',
    },
    vueI18n: './i18n.config.ts',
  },
})
```

```typescript
// i18n.config.ts
export default defineI18nConfig(() => ({
  legacy: false,
  fallbackLocale: 'en-US',
  fallbackWarn: false,
  missingWarn: false,
}))
```

### SEO 多语言

```vue
<script setup lang="ts">
const { locale, t } = useI18n()
const { locale: nuxtLocale, setLocale } = useI18n()
const head = useHead({
  htmlAttrs: { lang: locale },
  link: [
    { rel: 'alternate', hreflang: 'zh-CN', href: '/zh-CN' },
    { rel: 'alternate', hreflang: 'en-US', href: '/en-US' },
  ],
})
</script>
```

## 八、性能优化

### Message Compiler 预编译

Vite 插件 `@intlify/unplugin-vue-i18n` 可以在构建时预编译消息模板，减少运行时开销。

```typescript
// vite.config.ts
import VueI18nPlugin from '@intlify/unplugin-vue-i18n/vite'
import { resolve } from 'path'

export default defineConfig({
  plugins: [
    VueI18nPlugin({
      include: resolve(__dirname, './src/locales/**'),
      runtimeOnly: true, // 跳过消息编译器，减少包体积
    }),
  ],
})
```

### 摇树优化（Tree Shaking）

```typescript
// vue-i18n 10.x 支持只有 Message Compiler 的模式
// 在 vite.config.ts 中设置
export default defineConfig({
  resolve: {
    alias: {
      // 强制使用 runtime-only 包
      'vue-i18n': 'vue-i18n/dist/vue-i18n.runtime.esm-bundler.js',
    },
  },
})
```

## 注意事项

1. **legacy: false 必须设置** — Vue 3 必须使用 Composition API 模式，否则 `useI18n()` 无法正常工作。
2. **Fallback 策略** — 设置 `fallbackLocale` 避免翻译缺失时显示空值；配合 `fallbackWarn: false` 避免开发环境刷屏警告。
3. **语言持久化** — 用户切换语言后应写 Cookie / localStorage，应用启动时读取并初始化，避免每次刷新回到默认语言。
4. **SSR 双端一致** — 服务端渲染时 `locale` 需要在请求上下文中确定（Cookie / Accept-Language），防止客户端水合（hydration）不匹配。
5. **日期/数字格式** — 使用 `d()` 和 `n()` 代替手动 format，它们会根据当前 locale 自动选择格式。在 `createI18n` 中通过 `datetimeFormats` 和 `numberFormats` 自定义格式。
6. **Key 冲突** — 不同组件使用不同命名空间前缀（如 `nav.`、`form.`、`error.`），避免 key 冲突导致覆盖。
7. **动态语言名** — 若语言名称也需要国际化，额外维护一份 `localeNames` 映射：`{ 'zh-CN': { zh: '中文', en: 'Chinese' } }`。
8. **繁体中文本土化** — 推荐使用 OpenCC 工具将简体中文自动转为繁体中文，避免维护两套中文翻译。
