---
title: 使用 v-once 與 v-memo 指令跳過不必要的更新
impact: MEDIUM
impactDescription: v-once 能跳過靜態內容的所有後續更新；v-memo 則能根據條件快取 DOM 子樹，避免不必要的重新渲染。
tags: [vue3, performance, v-once, v-memo, optimization, directives]
---

# 使用 v-once 與 v-memo 指令跳過不必要的更新

**影響：中等 (MEDIUM)**

當 Vue 應用中的響應式資料變更時，相關的模板會被重新評估並渲染。然而，對於那些內容永不改變、或僅在極少數情況下才需要更新的區塊，`v-once` 和 `v-memo` 這兩個指令可以告訴 Vue 直接跳過更新程序，從而減少不必要的渲染開銷。

-   `v-once`：適用於那些初始化後就**永遠不需要**再更新的靜態內容。
-   `v-memo`：適用於那些**僅在特定條件**滿足時才需要更新的元素或元件，尤其適合用在列表渲染中。

## 檢查清單

-   對於僅在初始化時載入、後續永不變更的區塊，應用 `v-once` 指令。
-   對於大型列表中的項目，如果該項目僅需在特定資料變更時才重新渲染，應使用 `v-memo` 進行優化。
-   在使用前，務必確認被快取的內容不需要對其他狀態的變化做出響應。
-   使用 Vue DevTools 的效能分析工具，來驗證更新是否確實被跳過。

## v-once：僅渲染一次，永不更新

**不好的範例：**

即使內容是靜態的，在沒有指令的情況下，每次父元件更新時，Vue 仍會檢查這塊 DOM。

```vue
<template>
  <!-- 不好：每當父層重新渲染時，這個區塊也會被重新評估 -->
  <div class="terms-content">
    <h1>服務條款</h1>
    <p>版本：{{ termsVersion }}</p>
    <div v-html="termsContent"></div>
  </div>

  <!-- 這個頁腳內容永不改變，但 Vue 仍然會在每次更新時檢查它 -->
  <footer>
    <p>版權所有 {{ copyrightYear }} {{ companyName }}</p>
  </footer>
</template>
```

**好的範例：**

使用 `v-once` 後，這部分 DOM 在初次渲染後就會被視為靜態內容，完全跳過所有後續的更新檢查。

```vue
<template>
  <!-- 好：僅渲染一次，在所有未來的更新中都會被跳過 -->
  <div class="terms-content" v-once>
    <h1>服務條款</h1>
    <p>版本：{{ termsVersion }}</p>
    <div v-html="termsContent"></div>
  </div>

  <!-- v-once 告訴 Vue 這塊內容永遠不需要更新 -->
  <footer v-once>
    <p>版權所有 {{ copyrightYear }} {{ companyName }}</p>
  </footer>
</template>

<script setup>
// 這些值在元件建立時僅被賦值一次
const termsVersion = '2.1'
const termsContent = fetchedTermsHTML
const copyrightYear = 2024
const companyName = 'Acme Corp'
</script>
```

## v-memo：列表的條件式快取

**不好的範例：**

當 `selectedId` 改變時，即使絕大多數項目都沒有變化，整個列表的所有項目依然會被重新渲染。

```vue
<template>
  <!-- 不好：當 selectedId 改變時，所有項目都會重新渲染 -->
  <div v-for="item in list" :key="item.id">
    <div :class="{ selected: item.id === selectedId }">
      <ExpensiveComponent :data="item" />
    </div>
  </div>
</template>
```

**好的範例：**

使用 `v-memo` 後，只有當 `v-memo` 陣列中的依賴項發生變化時，對應的項目才會重新渲染。

```vue
<template>
  <!-- 好：項目僅在其「被選中」的狀態改變時才重新渲染 -->
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

const list = ref([/*...一個包含大量項目的陣列...*/])
const selectedId = ref(null)

// 當 selectedId 改變時：
// - 只有「之前被選中」的項目會重新渲染 (selected 狀態從 true 變為 false)
// - 只有「新被選中」的項目會重新渲染 (selected 狀態從 false 變為 true)
// - 所有其他項目的 v-memo 依賴項 [false] 並未改變，因此會被完全跳過。
</script>
```

## v-memo 的多重依賴

`v-memo` 的依賴陣列可以包含多個值。只有當陣列中任一值發生變化時，更新才會被觸發。

```vue
<template>
  <!-- 僅當項目的「被選中」或「編輯中」狀態改變時，才重新渲染 -->
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
```

## 使用 v-memo 模擬 v-once

如果傳遞給 `v-memo` 的是一個空陣列 `[]`，其效果等同於 `v-once`。

```vue
<template>
  <!-- v-memo="[]" 的效果與 v-once 相同 -->
  <div v-for="item in staticList" :key="item.id" v-memo="[]">
    {{ item.name }}
  </div>
</template>
```

## 何時不該使用

不當的使用會導致應有的更新被阻止，產生 Bug。

```vue
<template>
  <!-- 錯誤：內容需要隨 count 更新，但 v-once 會阻止它 -->
  <div v-once>
    <span>計數器：{{ count }}</span>  <!-- 這個數字將永遠不會更新！ -->
  </div>

  <!-- 錯誤：當子元件自身包含需要更新的狀態時 (例如 v-model) -->
  <div v-memo="[isSelected]">
    <!-- v-memo 會阻止這個元件更新，導致 v-model 行為不正常 -->
    <InputField v-model="item.name" />
  </div>

  <!-- 不建議：當優化的效益微乎其微時 -->
  <!-- 對一個簡單的文本節點使用 v-once，其帶來的開銷可能比效益還大 -->
  <span v-once>{{ simpleText }}</span>
</template>
```

## 效能比較

| 場景 | 無指令 | 使用 v-once / v-memo |
|------------------------------------|--------------------|----------------------------|
| 靜態頁首，父層重新渲染 100 次      | 重新評估 100 次    | 僅評估 1 次                |
| 1000 個項目的列表，僅一個選項改變  | 1000 個項目重新渲染| 僅 2 個項目重新渲染         |
| 包含複雜子元件的列表項            | 總是完整重新渲染   | 若 v-memo 條件未變則跳過     |

## 偵錯與驗證

要確認 `v-memo` 是否成功阻止了更新，可以使用 `onUpdated` 生命週期鉤子進行測試。

```vue
<script setup>
import { onUpdated } from 'vue'

// 如果 v-memo 成功阻止了更新，這個 onUpdated 鉤子將不會被觸發
onUpdated(() => {
  console.log('元件已更新')
})
</script>
```
