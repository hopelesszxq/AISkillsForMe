---
name: vue3-composition
description: Vue3 Composition API 最佳实践
tags: [vue3, composition-api, frontend]
---

## 核心规范
- 使用 `<script setup>` 语法
- 组合式函数（Composables）抽离逻辑复用
- TypeScript 优先，避免 any

## 性能
- v-for 配 key，避免使用 index
- 计算属性（computed）优于方法（methods）
- 组件懒加载 defineAsyncComponent
