---
title: 避免在大型列表中過度進行元件抽象化
impact: MEDIUM
impactDescription: 每個元件實例都有記憶體和渲染開銷 - 抽象化在列表中乘法這個開銷
type: efficiency
tags: [vue3, performance, components, abstraction, lists, optimization]
---

# 避免在大型列表中過度進行元件抽象化

**影響：MEDIUM** - 元件實例比純 DOM 節點更昂貴。雖然抽象化改善代碼組織，但不必要的嵌套會建立開銷。在大型列表中，此開銷乘法 - 100 個有 3 層抽象化的項目意味著 300+ 個元件實例而不是 100 個。

不要避免完全抽象化，但要注意頻繁渲染的元素（如列表項目）中的元件深度。

## 任務清單

- 審查列表項目元件中是否有不必要的包裝元件
- 考慮在熱路徑中壓平元件層級
- 當元件不增加價值時使用原生元素
- 使用 Vue DevTools 進行元件計數分析
- 將優化工作集中在最常渲染的元件上

**不好的：**
```vue
<!-- 不好：列表項目中的深層抽象化 -->
<template>
  <div class="user-list">
    <!-- 對於 100 個使用者：建立 400 個元件實例 -->
    <UserCard v-for="user in users" :key="user.id" :user="user" />
  </div>
</template>

<!-- UserCard.vue -->
<template>
  <Card>  <!-- 包裝元件 #1 -->
    <CardHeader>  <!-- 包裝元件 #2 -->
      <UserAvatar :src="user.avatar" />  <!-- 包裝元件 #3 -->
    </CardHeader>
    <CardBody>  <!-- 包裝元件 #4 -->
      <Text>{{ user.name }}</Text>
    </CardBody>
  </Card>
</template>

<!-- 每個 UserCard 建立：Card + CardHeader + CardBody + UserAvatar + Text
     100 個使用者 = 500+ 個元件實例 -->
```

**好的：**
```vue
<!-- 好的：列表項目中壓平的結構 -->
<template>
  <div class="user-list">
    <!-- 對於 100 個使用者：建立 100 個元件實例 -->
    <UserCard v-for="user in users" :key="user.id" :user="user" />
  </div>
</template>

<!-- UserCard.vue - 壓平、使用原生元素 -->
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
/* 會在 Card、CardHeader 等中的樣式 */
.card { /* ... */ }
.card-header { /* ... */ }
.card-body { /* ... */ }
.avatar { /* ... */ }
</style>
```

## 何時抽象化仍值得

```vue
<!-- 元件抽象化有價值的情況： -->

<!-- 1. 複雜行為被封裝 -->
<UserStatusIndicator :user="user" />  <!-- 有邏輯、工具提示等 -->

<!-- 2. 在熱路徑之外重複使用 -->
<Card>  <!-- 在一次性地方使用可以，但不在 100 個項目列表中 -->

<!-- 3. 列表本身很小 -->
<template v-if="items.length < 20">
  <FancyItem v-for="item in items" :key="item.id" />
</template>

<!-- 4. 虛擬化已使用（一次只渲染 ~20 個項目） -->
<RecycleScroller :items="items">
  <template #default="{ item }">
    <ComplexItem :item="item" />  <!-- 可以 - 只有 20 個實例存在 -->
  </template>
</RecycleScroller>
```

## 測量元件開銷

```javascript
// 在開發中，進行元件計數分析
import { onMounted, getCurrentInstance } from 'vue'

onMounted(() => {
  const instance = getCurrentInstance()
  let count = 0

  function countComponents(vnode) {
    if (vnode.component) count++
    if (vnode.children) {
      vnode.children.forEach(child => {
        if (child.component || child.children) countComponents(child)
      })
    }
  }

  // 改為使用 Vue DevTools 進行準確計數
  console.log('在 Vue DevTools 元件標籤中檢查實例計數')
})
```

## 包裝元件的替代方案

```vue
<!-- 不用 <Button> 元件進行樣式化： -->
<button class="btn btn-primary">點擊</button>

<!-- 不用 <Text> 元件： -->
<span class="text-body">{{ content }}</span>

<!-- 不用列表中的佈局包裝元件： -->
<div class="flex items-center gap-2">
  <!-- content -->
</div>

<!-- 改用 CSS 類別或 Tailwind 而不是元件抽象化進行樣式化 -->
```

## 影響計算

| 列表大小 | 每個項目的元件 | 總實例 | 記憶體影響 |
|-----------|---------------------|-----------------|---------------|
| 100 個項目 | 1 個（壓平） | 100 個 | 基線 |
| 100 個項目 | 3 個（嵌套） | 300 個 | ~3 倍記憶體 |
| 100 個項目 | 5 個（深層嵌套） | 500 個 | ~5 倍記憶體 |
| 1000 個項目 | 1 個（壓平） | 1000 個 | 高 |
| 1000 個項目 | 5 個（深層嵌套） | 5000 個 | 非常高 |
