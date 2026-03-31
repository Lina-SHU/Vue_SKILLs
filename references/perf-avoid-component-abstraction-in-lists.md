---
title: 避免在列表中過度抽象化元件
impact: MEDIUM
impactDescription: 每個元件實例都有其記憶體和初始化開銷。在大型列表中，不必要的元件抽象化會層層疊加，顯著放大這些開銷。
type: efficiency
tags: [vue3, performance, components, abstraction, lists, optimization]
---

# 避免在列表中過度抽象化元件

**影響：中等 (MEDIUM)**

每個元件的實例化都需要佔用記憶體並耗費初始化時間，因此會比純粹的 DOM 節點昂貴。雖然抽象化有助於改善程式碼的組織結構，但在大型列表中，不必要的巢狀元件會導致效能開銷被大幅放大。

例如，一個包含 100 個項目的列表，如果每個項目都由 3 層抽象化的元件所組成，那麼最終將會產生 300 多個元件實例，而非僅僅 100 個。

這並非建議完全避免抽象化，而是提醒我們在處理頻繁渲染的元素（例如列表項目）時，應謹慎評估元件的嵌套深度。

## 檢查清單

-   審查列表項目元件，移除不必要的包裝 (Wrapper) 元件。
-   在效能熱點路徑 (hot path) 中，盡可能地「壓平」元件層級。
-   當子元件沒有提供額外的邏輯或狀態時，優先使用原生 HTML 元素。
-   使用 Vue DevTools 的效能工具來分析和計算頁面中的元件數量。
-   將優化心力集中在最常被渲染的元件上。

---

**不好的範例：**

在列表項目中進行了深度的抽象化。

```vue
<!-- 不好：列表項目中的深層抽象化 -->
<template>
  <div class="user-list">
    <!-- 100 個使用者將建立約 600 個元件實例 -->
    <UserCard v-for="user in users" :key="user.id" :user="user" />
  </div>
</template>

<!-- UserCard.vue -->
<template>
  <Card>  <!-- 抽象 #1 -->
    <CardHeader>  <!-- 抽象 #2 -->
      <UserAvatar :src="user.avatar" />  <!-- 抽象 #3 -->
    </CardHeader>
    <CardBody>  <!-- 抽象 #4 -->
      <Text>{{ user.name }}</Text> <!-- 抽象 #5 -->
    </CardBody>
  </Card>
</template>

<!-- 
  計算：
  每個 UserCard 自身是 1 個實例。
  其內部又包含了 Card, CardHeader, UserAvatar, CardBody, Text 這 5 個子元件。
  總計：100 * (1 + 5) = 600 個元件實例，效能開銷龐大。
-->
```

**好的範例：**

將列表項目的結構「壓平」，並多使用原生元素。

```vue
<!-- 好：列表項目採用扁平化結構 -->
<template>
  <div class="user-list">
    <!-- 100 個使用者只會建立 100 個元件實例 -->
    <UserCard v-for="user in users" :key="user.id" :user="user" />
  </div>
</template>

<!-- UserCard.vue - 扁平化，並使用原生元素 -->
<template>
  <div class="card">
    <div class="card-header">
      <img :src="user.avatar" :alt="user.name" class="avatar" />
    </div>
    <div class="card-body">
      <span class="user-name">{{ user.name }}</span>
    </div>
  </div>
</template>

<script setup>
defineProps({
  user: Object
})
</script>

<style scoped>
/* 原本分散在 Card、CardHeader 等元件的樣式可以集中於此 */
.card { /* ... */ }
.card-header { /* ... */ }
.card-body { /* ... */ }
.avatar { /* ... */ }
</style>
```

## 何時仍值得使用元件抽象化

在以下情況，元件抽象化所帶來的好處依然大於其效能成本：

```vue
<!-- 1. 當元件封裝了複雜的行為邏輯 -->
<!-- 這個元件可能包含了 API 請求、狀態管理、互動提示等 -->
<UserStatusIndicator :user="user" />

<!-- 2. 在效能熱點路徑之外的地方重複使用 -->
<!-- 在單次顯示的頁面區塊使用 Card 元件是完全可以接受的 -->
<Card>
  <!-- ... -->
</Card>

<!-- 3. 列表本身的尺寸非常小 -->
<!-- 如果列表長度可預期地很短，抽象化的開銷可以忽略不計 -->
<template v-if="items.length < 20">
  <FancyItem v-for="item in items" :key="item.id" />
</template>

<!-- 4. 當列表已經採用了虛擬化技術 -->
<!-- 虛擬滾動器一次只渲染可見範圍內的項目（約 20 個），因此元件實例總數可控 -->
<RecycleScroller :items="items">
  <template #default="{ item }">
    <ComplexItem :item="item" />  <!-- 沒問題，因為同時存在的實例數量很少 -->
  </template>
</RecycleScroller>
```

## 如何測量元件開銷

最直接且準確的方式是使用 **Vue DevTools**。

1.  在瀏覽器中打開開發者工具。
2.  切換到 "Vue" 分頁。
3.  使用元件檢測器選取您想分析的列表或區塊。
4.  觀察 "Component Count" 或類似的統計數據，了解實例化的確切數量。

這能幫助您直觀地識別出哪些部分產生了過多的元件實例。

## 元件抽象化的替代方案

在追求樣式複用或版面配置時，不一定需要透過元件。考慮以下替代方案：

```vue
<!-- 使用 CSS class 來複用樣式，而非 <Button> 元件 -->
<button class="btn btn-primary">點擊</button>

<!-- 直接使用帶有 class 的原生元素，而非 <Text> 元件 -->
<span class="text-body">{{ content }}</span>

<!-- 在列表中使用 CSS (如 Flexbox/Grid) 進行佈局，而非 <Layout> 元件 -->
<div class="flex items-center gap-2">
  <!-- content -->
</div>

<!-- 優先考慮使用如 Tailwind CSS 這類的 Utility-First CSS 框架，
     或定義可複用的 CSS class，以此來取代僅用於樣式的元件抽象化。 -->
```

## 影響估算

下表展示了元件嵌套深度對總實例數量和記憶體佔用的影響：

| 列表大小  | 每個項目的元件數 | 總實例數 | 記憶體影響 |
|-----------|--------------------|----------|------------|
| 100 項目  | 1 (扁平)           | 100      | 基準       |
| 100 項目  | 3 (輕度嵌套)       | 300      | ~3 倍開銷  |
| 100 項目  | 5 (深度嵌套)       | 500      | ~5 倍開銷  |
| 1000 項目 | 1 (扁平)           | 1000     | 高         |
| 1000 項目 | 5 (深度嵌套)       | 5000     | 非常高     |
