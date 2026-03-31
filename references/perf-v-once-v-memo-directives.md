---
title: 使用 v-once 和 v-memo 略過不必要的更新
impact: MEDIUM
impactDescription: v-once 略過所有靜態內容的未來更新；v-memo 有條件地記憶子樹
type: efficiency
tags: [vue3, performance, v-once, v-memo, optimization, directives]
---

# 使用 v-once 和 v-memo 略過不必要的更新

**影響：MEDIUM** - Vue 在每次反應性變化時重新評估模板。對於永不改變或很少改變的內容，`v-once` 和 `v-memo` 告訴 Vue 略過更新，減少渲染工作。

對於永遠不需要更新的真正靜態內容使用 `v-once`，對於應僅在特定條件改變時更新的列表項目使用 `v-memo`。

## 任務清單

- 將 `v-once` 應用於使用執行時數據但永遠不需要更新的元素
- 將 `v-memo` 應用於應僅在特定條件改變時更新的列表項目
- 驗證記憶內容無需回應其他狀態變化
- 使用 Vue DevTools 進行分析以確認更新略過

## v-once：渲染一次，永不更新

**不好的：**
```vue
<template>
  <!-- 不好：在每次父級重新渲染時重新評估 -->
  <div class="terms-content">
    <h1>服務條款</h1>
    <p>版本：{{ termsVersion }}</p>
    <div v-html="termsContent"></div>
  </div>

  <!-- 此內容永不改變，但 Vue 在每次渲染時檢查它 -->
  <footer>
    <p>版權 {{ copyrightYear }} {{ companyName }}</p>
  </footer>
</template>
```

**好的：**
```vue
<template>
  <!-- 好的：渲染一次，在所有未來更新時略過 -->
  <div class="terms-content" v-once>
    <h1>服務條款</h1>
    <p>版本：{{ termsVersion }}</p>
    <div v-html="termsContent"></div>
  </div>

  <!-- v-once 告訴 Vue 永遠不需要更新此內容 -->
  <footer v-once>
    <p>版權 {{ copyrightYear }} {{ companyName }}</p>
  </footer>
</template>

<script setup>
// 這些值在元件建立時設置一次
const termsVersion = '2.1'
const termsContent = fetchedTermsHTML
const copyrightYear = 2024
const companyName = 'Acme Corp'
</script>
```

## v-memo：列表的條件記憶化

**不好的：**
```vue
<template>
  <!-- 不好：當 selectedId 改變時所有項目重新渲染 -->
  <div v-for="item in list" :key="item.id">
    <div :class="{ selected: item.id === selectedId }">
      <ExpensiveComponent :data="item" />
    </div>
  </div>
</template>
```

**好的：**
```vue
<template>
  <!-- 好的：項目僅在其選擇狀態改變時重新渲染 -->
  <div
    v-for="item in list"
    :key="item.id"
    v-memo="[item.id === selectedId]"
  >
    <div :class="{ selected: item.id === selectedId }">
      <ExpensiveComponent :data="item" />
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const list = ref([/* 許多項目 */])
const selectedId = ref(null)

// 當 selectedId 改變時：
// - 僅之前選中的項目重新渲染（selected: true -> false）
// - 僅新選中的項目重新渲染（selected: false -> true）
// - 所有其他項目被略過（v-memo 值未改變）
</script>
```

## v-memo 搭配多個依賴項

```vue
<template>
  <!-- 僅當項目的選擇或編輯狀態改變時重新渲染 -->
  <div
    v-for="item in items"
    :key="item.id"
    v-memo="[item.id === selectedId, item.id === editingId]"
  >
    <ItemCard
      :item="item"
      :selected="item.id === selectedId"
      :editing="item.id === editingId"
    />
  </div>
</template>

<script setup>
const selectedId = ref(null)
const editingId = ref(null)
const items = ref([/* ... */])
</script>
```

## v-memo 搭配空陣列 = v-once

```vue
<template>
  <!-- v-memo="[]" 相當於 v-once -->
  <div v-for="item in staticList" :key="item.id" v-memo="[]">
    {{ item.name }}
  </div>
</template>
```

## 何時不應使用這些指令

```vue
<template>
  <!-- 不要：確實需要更新的內容 -->
  <div v-once>
    <span>計數：{{ count }}</span>  <!-- count 不會更新！ -->
  </div>

  <!-- 不要：當子元件有其自己的反應性狀態時 -->
  <div v-memo="[selected]">
    <InputField v-model="item.name" />  <!-- v-model 無法正常工作 -->
  </div>

  <!-- 不要：當記憶化好處最小時 -->
  <span v-once>{{ simpleText }}</span>  <!-- 開銷不值得 -->
</template>
```

## 性能比較

| 情景 | 沒有指令 | 搭配 v-once/v-memo |
|----------|-------------------|-------------------|
| 靜態頁首、父級重新渲染 100 次 | 重新評估 100 次 | 評估 1 次 |
| 1000 個項目、選擇改變 | 1000 個項目重新渲染 | 2 個項目重新渲染 |
| 複雜子元件 | 完整重新渲染 | 如果記憶化則略過 |

## 對記憶化元件進行偵錯

```vue
<script setup>
import { onUpdated } from 'vue'

// 如果 v-memo 防止更新，此事件不會觸發
onUpdated(() => {
  console.log('元件已更新')
})
</script>
```
