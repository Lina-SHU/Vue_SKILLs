---
title: 透過「列表虛擬化」避免 DOM 負擔過重
impact: HIGH
impactDescription: 當一次性渲染成千上萬個列表項目時，會產生過多的 DOM 節點，導致渲染速度變慢、記憶體佔用過高，甚至可能使頁面卡頓。
tags: [vue3, performance, virtual-list, large-data, dom, optimization]
---

# 透過「列表虛擬化」避免 DOM 負擔過重

**影響：高 (HIGH)**

當一個列表包含數百甚至數千個項目時，若使用傳統的 `v-for` 進行渲染，會一次性地將所有項目都生成為 DOM 節點。每一個 DOM 節點都會消耗記憶體、拖慢頁面的初始渲染速度，並使得後續的任何更新都變得代價高昂。

「列表虛擬化」 (List Virtualization) 是一種僅渲染當前可視範圍內項目的技術，它能極大地提升處理長列表時的應用程式效能。

**核心建議：** 當列表的項目數量可能超過 50-100 項時，特別是當每個項目都包含複雜內容時，就應該考慮使用列表虛擬化技術。

## 檢查清單

-   找出應用中所有可能渲染超過 50-100 個項目的列表。
-   選擇並安裝一個合適的列表虛擬化函式庫 (例如 `vue-virtual-scroller` 或 `@tanstack/vue-virtual`)。
-   使用虛擬化元件來取代標準的 `v-for` 迴圈。
-   確保列表中的每個項目都具有一致或可預估的高度 (這是虛擬化計算的基礎)。
-   在開發過程中，使用接近真實場景的數據量來進行測試和驗證。

## 推薦的函式庫

| 函式庫 | 最適用場景 | 備註 |
|---|---|---|
| `vue-virtual-scroller` | 通用，設定簡單 | 社群最流行，預設功能完善，上手快。 |
| `@tanstack/vue-virtual` | 複雜佈局，無預設樣式 | 又稱為 "Headless" 函式庫，與框架解耦，提供極高靈活性。 |
| `vue-virtual-scroll-grid` | 網格 (Grid) 佈局 | 專為二維滾動場景設計。 |
| `vueuc/VVirtualList` | Naive UI 專案 | 若您已在使用 Naive UI，這是最自然的選擇。 |

---

**不好的範例：**

一次性渲染全部 10,000 個項目，會產生大量 DOM 節點，極易導致瀏覽器回應緩慢或凍結。

```vue
<template>
  <!-- 不好：立即渲染所有 10,000 個項目 -->
  <div class="user-list">
    <UserCard
      v-for="user in users"
      :key="user.id"
      :user="user"
    />
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'

const users = ref([])

onMounted(async () => {
  // 這一步將會一次性建立 10,000 個 DOM 節點，對瀏覽器造成巨大壓力
  users.value = await fetchAllUsers() // 假設返回 10,000 筆資料
})
</script>
```

**好的範例：**

使用 `vue-virtual-scroller`，無論總項目有多少，DOM 中始終只維持 20 個左右的節點。

```vue
<template>
  <!-- 好：一次只渲染約 20 個可見項目 -->
  <RecycleScroller
    class="user-list"
    :items="users"
    :item-size="80"
    key-field="id"
    v-slot="{ item }"
  >
    <UserCard :user="item" />
  </RecycleScroller>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { RecycleScroller } from 'vue-virtual-scroller'
import 'vue-virtual-scroller/dist/vue-virtual-scroller.css'

const users = ref([])

onMounted(async () => {
  // 10,000 筆資料存在於記憶體中，但 DOM 中只會建立約 20 個節點
  users.value = await fetchAllUsers()
})
</script>

<style scoped>
.user-list {
  /* 關鍵：虛擬滾動的容器必須具有一個明確的高度 */
  height: 600px;
}
</style>
```

## 範例：使用 `@tanstack/vue-virtual`

這個函式庫提供了更底層的控制，讓您可以自訂滾動的容器和項目結構。

```vue
<template>
  <div ref="parentRef" class="list-container">
    <!-- 總高度由虛擬器計算得出，用於產生正確的滾動條 -->
    <div :style="{ height: `${rowVirtualizer.getTotalSize()}px`, position: 'relative' }">
      <!-- 僅渲染可見的項目 -->
      <div
        v-for="virtualRow in rowVirtualizer.getVirtualItems()"
        :key="virtualRow.key"
        :style="{
          position: 'absolute',
          top: 0,
          left: 0,
          width: '100%',
          height: `${virtualRow.size}px`,
          transform: `translateY(${virtualRow.start}px)`
        }"
      >
        <UserCard :user="users[virtualRow.index]" />
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { useVirtualizer } from '@tanstack/vue-virtual'

const users = ref([/*...10,000 個使用者...*/])
const parentRef = ref(null)

const rowVirtualizer = useVirtualizer({
  count: users.value.length,
  getScrollElement: () => parentRef.value,
  estimateSize: () => 80,  // 提供一個預估的行高
  overscan: 5  // 在可視範圍的上方和下方額外渲染 5 個項目，以減少滾動時的白屏
})
</script>

<style scoped>
.list-container {
  height: 600px;
  overflow: auto; /* 容器自身負責滾動 */
}
</style>
```

## 處理動態高度的項目

如果列表項目的高度不固定（例如聊天訊息），`vue-virtual-scroller` 提供了 `DynamicScroller` 元件。

```vue
<template>
  <!-- 對於高度可變的項目，使用 DynamicScroller -->
  <DynamicScroller
    :items="messages"
    :min-item-size="54"
    key-field="id"
  >
    <template #default="{ item, index, active }">
      <DynamicScrollerItem
        :item="item"
        :active="active"
        :data-index="index"
      >
        <ChatMessage :message="item" />
      </DynamicScrollerItem>
    </template>
  </DynamicScroller>
</template>
```

## 效能對比

| 渲染方式 | 100 項目 | 1,000 項目 | 10,000 項目 |
|---|---|---|---|
| 一般 `v-for` | ~100 DOM 節點 | ~1,000 DOM 節點 | ~10,000 DOM 節點 |
| **虛擬化** | **~20 DOM 節點** | **~20 DOM 節點** | **~20 DOM 節點** |
| 初始渲染時間 (v-for) | 快 | 慢 | 非常慢 / 可能崩潰 |
| 初始渲染時間 (虛擬化) | **快** | **快** | **快** |

## 何時不該使用虛擬化

列表虛擬化並非萬靈丹，在以下場景中應避免使用：

-   **短列表**：當列表項目少於 50 個且內容簡單時，引入虛擬化的複雜性得不償失。
-   **可訪問性 (Accessibility)**：如果所有項目必須同時對螢幕閱讀器等輔助技術可見。
-   **列印佈局**：當頁面需要被列印時，所有內容都必須完整渲染出來。
-   **SEO 關鍵內容**：如果列表內容對於搜尋引擎優化至關重要，需要確保它們存在於初始的 HTML 中。
