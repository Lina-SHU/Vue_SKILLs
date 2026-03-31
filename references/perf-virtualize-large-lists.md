---
title: 虛擬化大型列表以避免 DOM 過載
impact: HIGH
impactDescription: 渲染數千個列表項目會建立過多 DOM 節點，導致緩慢渲染和高記憶體使用
type: efficiency
tags: [vue3, performance, virtual-list, large-data, dom, optimization]
---

# 虛擬化大型列表以避免 DOM 過載

**影響：HIGH** - 在大型列表中渲染所有項目（數百或數千個）會建立大量 DOM 節點。每個節點消耗記憶體、減慢初始渲染，並使更新代價高昂。列表虛擬化只渲染可見項目，大幅改善性能。

處理可能超過 50-100 個項目的列表時使用虛擬化庫，特別是當項目有複雜內容時。

## 任務清單

- 辨識渲染超過 50-100 個項目的列表
- 安裝虛擬化庫（vue-virtual-scroller、@tanstack/vue-virtual）
- 用虛擬化元件替換標準 `v-for`
- 確保列表項目有一致或可估計的高度
- 在開發過程中使用實際數據量進行測試

## 推薦的庫

| 庫 | 最適合 | 備註 |
|------|----------|-------|
| `vue-virtual-scroller` | 通用、易於設置 | 最受歡迎、預設值良好 |
| `@tanstack/vue-virtual` | 複雜佈局、無界面庫 | 與框架無關、靈活 |
| `vue-virtual-scroll-grid` | 網格佈局 | 2D 虛擬化 |
| `vueuc/VVirtualList` | Naive UI 專案 | Naive UI 生態系統的一部分 |

**不好的：**
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
import UserCard from './UserCard.vue'

const users = ref([])

onMounted(async () => {
  // 建立 10,000 個 DOM 節點，瀏覽器掙扎
  users.value = await fetchAllUsers()
})
</script>
```

**好的：**
```vue
<template>
  <!-- 好的：一次只渲染 ~20 個可見項目 -->
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
import UserCard from './UserCard.vue'

const users = ref([])

onMounted(async () => {
  // 10,000 個項目在記憶體中，但僅 ~20 個 DOM 節點
  users.value = await fetchAllUsers()
})
</script>

<style scoped>
.user-list {
  height: 600px; /* 容器必須有固定高度 */
}
</style>
```

## 使用 @tanstack/vue-virtual

```vue
<template>
  <div ref="parentRef" class="list-container">
    <div
      :style="{
        height: `${rowVirtualizer.getTotalSize()}px`,
        position: 'relative'
      }"
    >
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

const users = ref([/* 10,000 個使用者 */])
const parentRef = ref(null)

const rowVirtualizer = useVirtualizer({
  count: users.value.length,
  getScrollElement: () => parentRef.value,
  estimateSize: () => 80,  // 估計行高
  overscan: 5  // 在視口上方/下方渲染 5 個額外項目
})
</script>

<style scoped>
.list-container {
  height: 600px;
  overflow: auto;
}
</style>
```

## 使用 vue-virtual-scroller 處理動態高度

```vue
<template>
  <!-- 對於可變高度項目，使用 DynamicScroller -->
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

<script setup>
import { DynamicScroller, DynamicScrollerItem } from 'vue-virtual-scroller'
</script>
```

## 性能比較

| 方式 | 100 個項目 | 1,000 個項目 | 10,000 個項目 |
|----------|-----------|-------------|--------------|
| 常規 v-for | ~100 個 DOM 節點 | ~1,000 個 DOM 節點 | ~10,000 個 DOM 節點 |
| 虛擬化 | ~20 個 DOM 節點 | ~20 個 DOM 節點 | ~20 個 DOM 節點 |
| 初始渲染 | 快速 | 緩慢 | 非常緩慢 / 當機 |
| 虛擬化渲染 | 快速 | 快速 | 快速 |

## 何時不應虛擬化

- 包含簡單內容的少於 50 個項目的列表
- 列表中所有項目必須同時可供螢幕閱讀器存取的列表
- 所有內容必須渲染的列印佈局
- 必須在初始 HTML 中包含的 SEO 關鍵內容
