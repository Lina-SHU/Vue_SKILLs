---
title: 單檔案元件結構、樣式和模板模式
impact: MEDIUM
impactDescription: 一致的 SFC 結構和樣式選用，能提升程式碼的可維護性、工具支援及渲染效能
tags: [vue3, sfc, scoped-css, styles, build-tools, performance, template, v-html, v-for, computed, v-if, v-show]
---

# 單檔案元件結構、樣式和模板模式

**影響：MEDIUM** - 採用結構一致且樣式高效的 SFC，能讓元件更容易維護，並避免不必要的渲染開銷。

## 任務清單

- 開發元件時，使用 .vue 的 SFC 格式，而非將 .js/.ts 和 .css 拆分成獨立檔案。
- 預設情況下，將模板 (template)、腳本 (script) 和樣式 (style) 都放在同一個 SFC 檔案中。
- 在模板與檔案命名時，對元件名稱使用 PascalCase (大駝峰式命名)。
- 優先採用限定作用域在元件內的樣式 (component-scoped styles)。
- 為了效能考量，在限定作用域的 CSS 中，優先使用類別選擇器 (class selector)，而不是元素選擇器 (element selector)。
- 在 Vue 3.5+ 環境中，使用 `useTemplateRef()` 來存取 DOM 或元件的 ref。
- 在 `:style` 綁定中，使用 camelCase (小駝峰式命名) 的屬性，以維持一致性並獲得 IDE 更好的支援。
- 正確使用 `v-for` 和 `v-if`
- 絕對不要對不受信任或使用者提供的內容使用 `v-html`。
- 根據切換的頻繁程度與初次渲染的成本，來決定使用 `v-if` 或是 `v-show`。

## 將模板、腳本與樣式寫在同一檔案

**不好的：**
```
components/
├── UserCard.vue
├── UserCard.js
└── UserCard.css
```

**好的：**
```vue
<!-- components/UserCard.vue -->
<script setup>
import { computed } from 'vue'

const props = defineProps({
  user: { type: Object, required: true }
})

const displayName = computed(() =>
  `${props.user.firstName} ${props.user.lastName}`
)
</script>

<template>
  <div class="user-card">
    <h3 class="name">{{ displayName }}</h3>
  </div>
</template>

<style scoped>
.user-card {
  padding: 1rem;
}

.name {
  margin: 0;
}
</style>
```

## 元件命名採用 PascalCase

**不好的：**
```vue
<script setup>
import userProfile from './user-profile.vue'
</script>

<template>
  <user-profile :user="currentUser" />
</template>
```

**好的：**
```vue
<script setup>
import UserProfile from './UserProfile.vue'
</script>

<template>
  <UserProfile :user="currentUser" />
</template>
```

## SFC 中 `<style>` 區塊的最佳實踐

### 優先採用限定作用域的樣式

- 對屬於元件的樣式使用 `<style scoped>`。
- 將**全域 CSS** 存放在獨立的檔案中 (例如 `src/assets/main.css`)，用來定義重設 (reset)、排版 (typography)、設計變數 (tokens) 等。
- 謹慎使用 `:deep()`，只在特殊情境下使用。

**不好的：**

```vue
<style>
/* ❌ 樣式會洩漏至全域 */
button { border-radius: 999px; }
</style>
```

**好的：**

```vue
<style scoped>
.button { border-radius: 999px; }
</style>
```

**好的：**

```css
/* src/assets/main.css */
/* ✅ 用於定義重設、設計變數、排版與全域規則 */
:root { --radius: 999px; }
```

### 在限定作用域的 CSS 中使用類別選擇器

**不好的：**
```vue
<template>
  <article>
    <h1>{{ title }}</h1>
    <p>{{ subtitle }}</p>
  </article>
</template>

<style scoped>
article { max-width: 800px; }
h1 { font-size: 2rem; }
p { line-height: 1.6; }
</style>
```

**好的：**
```vue
<template>
  <article class="article">
    <h1 class="article-title">{{ title }}</h1>
    <p class="article-subtitle">{{ subtitle }}</p>
  </article>
</template>

<style scoped>
.article { max-width: 800px; }
.article-title { font-size: 2rem; }
.article-subtitle { line-height: 1.6; }
</style>
```

## 使用 `useTemplateRef()` 來存取 DOM / 元件 ref

對於 Vue 3.5+ 環境中：使用 `useTemplateRef()` 來存取模板 ref。

```vue
<script setup>
import { onMounted, useTemplateRef } from 'vue'

const inputRef = useTemplateRef('input')

onMounted(() => {
  inputRef.value?.focus()
})
</script>

<template>
  <input ref="input" />
</template>
```

## 在 `:style` 綁定中使用 camelCase

**不好的：**
```vue
<template>
  <div :style="{ 'font-size': fontSize + 'px', 'background-color': bg }">
    Content
  </div>
</template>
```

**好的：**
```vue
<template>
  <div :style="{ fontSize: fontSize + 'px', backgroundColor: bg }">
    Content
  </div>
</template>
```

## 正確使用 `v-for` 和 `v-if`

### 始終提供穩定的 `:key`

- 優先使用原始型別 (`string | number`) 作為 key。
- 避免使用物件作為鍵。

**好的：**

```vue
<li v-for="item in items" :key="item.id">
  <input v-model="item.text" />
</li>
```

### 避免在同一元素上使用 `v-if` 和 `v-for`

這種寫法會讓意圖變得模糊，並導致不必要的運算。
([參考資料](https://vuejs.org/guide/essentials/list.html#v-for-with-v-if))

**篩選列表項目**
**不好的：**

```vue
<li v-for="user in users" v-if="user.active" :key="user.id">
  {{ user.name }}
</li>
```

**好的：**

```vue
<script setup>
import { computed } from 'vue'

const activeUsers = computed(() => users.value.filter(u => u.active))
</script>

<template>
  <li v-for="user in activeUsers" :key="user.id">
    {{ user.name }}
  </li>
</template>
```

**依條件顯示/隱藏整個列表**
**好的：**

```vue
<ul v-if="shouldShowUsers">
  <li v-for="user in users" :key="user.id">
    {{ user.name }}
  </li>
</ul>
```

## 絕對不要用 `v-html` 渲染不受信任的 HTML

**不好的：**
```vue
<template>
  <!-- 風險：不受信任的內容可能會注入惡意腳本 -->
  <article v-html="userProvidedContent"></article>
</template>
```

**好的：**
```vue
<script setup>
import { computed } from 'vue'
import DOMPurify from 'dompurify'

const props = defineProps({
  trustedHtml: String,
  plainText: { type: String, required: true }
})

const safeHtml = computed(() => DOMPurify.sanitize(props.trustedHtml ?? ''))
</script>

<template>
  <!-- 建議作法：使用會自動跳脫 (escape) 的插值 -->
  <p>{{ props.plainText }}</p>

  <!-- 僅限於渲染可信任或已淨化 (sanitized) 的 HTML -->
  <article v-html="safeHtml"></article>
</template>
```

## 根據切換行為來選擇 `v-if` 或 `v-show`

**不好的：**
```vue
<template>
  <!-- 頻繁切換的場景若使用 v-if，會導致不斷地掛載/卸載元件，成本較高 -->
  <ComplexPanel v-if="isPanelOpen" />

  <!-- 很少顯示的內容若使用 v-show，即使不顯示，仍在初次渲染時支付了成本 -->
  <AdminPanel v-show="isAdmin" />
</template>
```
