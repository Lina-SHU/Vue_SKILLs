---
title: 反應性核心模式 (ref, reactive, shallowRef, computed, watch)
impact: MEDIUM
impactDescription: 選用清晰的反應性 API 能讓狀態變得可預測，並減少 Vue 3 應用程式中不必要的更新
tags: [vue3, reactivity, ref, reactive, shallowRef, computed, watch, watchEffect, external-state, best-practice]
---

# 反應性核心模式 (ref, reactive, shallowRef, computed, watch)

**影響：MEDIUM** - 起手式就該選擇正確的反應性 API，使用 `computed` 來衍伸狀態，並只在需要處理副作用時才使用 `watch`。

本篇內容涵蓋了在處理本地狀態、外部資料、衍生值和副作用時，所需要做的核心反應性決策。

## 任務清單

- 正確地宣告反應性狀態
  - 針對原始值 (primitives)，使用 `ref()`
  - 為物件、陣列、Map 或 Set 選擇合適的反應性 API
- 遵循 `reactive` 的最佳實踐
  - 避免直接對 `reactive()` 的回傳值進行解構
  - 正確地監聽 `reactive` 物件
- 遵循 `computed` 最佳實踐
  - 優先使用 `computed`，而不是透過 `watch` 來改變 `ref` 來達成衍生狀態
  - 將過濾或排序等衍生狀態的邏輯，保留在模板 (template) 之外
  - 針對可重用的 class 或 style 邏輯，使用 `computed`
  - 保持 `computed` getter 的純粹性 (無副作用)，並將副作用交給 `watch` 處理
- 遵循 `watch` 的最佳實踐
  - 使用 `immediate: true` 選項，來取代在 `watch` 之外重複執行初始呼叫
  - 在 `watch` 中處理非同步副作用時，記得進行清理 (cleanup)

## 正確宣告反應性狀態

### 針對原始值 (字串、數字、布林值、null 等)，使用 `ref()`。

**正確的：**
```js
import { ref } from 'vue'
const count = ref(0)
```

### 為物件、陣列、Map、Set 選擇合適的反應性 API

當您需要深層的反應性，又經常需要**替換整個物件** (`state.value = newObj`) 時，請使用 `ref()`。常見情境：

- 頻繁被重新賦值的狀態 (例如替換整個 fetch 回來的物件/列表、重設為預設值)。
- Composable 的回傳值，其更新主要透過 `.value` 的重新賦值來完成。

當您主要是**改變物件的屬性**，而很少替換整個物件時，請使用 `reactive()`。常見情境：

- 「單一狀態物件」的模式 (例如 store 或表單狀態)：`state.count++`、`state.items.push(...)`、`state.user.name = ...`。
- 想避免使用 `.value`，且需要直接更新巢狀屬性的情境。

```js
import { reactive } from 'vue'

const state = reactive({
  count: 0,
  user: { name: 'Alice', age: 30 }
})

state.count++ // ✅ reactive
state.user.age = 31 // ✅ reactive
// ❌ 應避免替換整個 reactive 物件的參考：
// state = reactive({ count: 1 })
```

當值是**不透明的、或不該被代理 (proxy)** (例如類別的實例、外部套件的物件、或非常大的巢狀資料)，且您只希望在**替換 `.value`** 時觸發更新 (不做深層追蹤) 時，請使用 `shallowRef()`。常見情境：

- 存放不需要 Vue 來代理其內部的外部實例或句柄 (例如 SDK client、類別實例)。
- 透過替換根參考來更新的龐大資料 (亦即 immutable 風格的更新)。

```js
import { shallowRef } from 'vue'

const user = shallowRef({ name: 'Alice', age: 30 })

user.value.age = 31 // ❌ 不會觸發反應
user.value = { name: 'Bob', age: 25 } // ✅ 會觸發更新
```

當您只希望**頂層屬性**具有反應性，而巢狀物件保持原樣時，請使用 `shallowReactive()`。常見情境：

- 某個容器物件，只有頂層的 key 會變動，而巢狀的內容應保持不受 Vue 管理或代理。
- 混合結構，其中 Vue 追蹤外層的包裝物件，但不追蹤深層的巢狀或外部物件。

```js
import { shallowReactive } from 'vue'

const state = shallowReactive({
  count: 0,
  user: { name: 'Alice', age: 30 }
})

state.count++ // ✅ reactive
state.user.age = 31 // ❌ 不會觸發反應
```

## `reactive` 最佳實踐

### 避免直接對 reactive 物件進行解構

**不好的：**

```js
import { reactive } from 'vue'

const state = reactive({ count: 0 })
const { count } = state // ❌ 解構後會失去反應性
```

### 正確地監聽 reactive 物件

**不好的：**

直接將一個普通值傳給 `watch()`

```js
import { reactive, watch } from 'vue'

const state = reactive({ count: 0 })

// ❌ watch() 的第一個參數需要是 getter 函式、ref、reactive 物件，或由以上類型組成的陣列
watch(state.count, () => { /* ... */ })
```

**好的：**

使用 `toRefs()` 來保持反應性，或為 `watch()` 傳入 getter 函式

```js
import { reactive, toRefs, watch } from 'vue'

const state = reactive({ count: 0 })
const { count } = toRefs(state) // ✅ count 現在是一個 ref

watch(count, () => { /* ... */ }) // ✅
watch(() => state.count, () => { /* ... */ }) // ✅
```

## `computed` 最佳實踐

### 優先使用 computed 來衍生狀態，而非 watch

**不好的：**
```js
import { ref, watchEffect } from 'vue'

const items = ref([{ price: 10 }, { price: 20 }])
const total = ref(0)

watchEffect(() => {
  total.value = items.value.reduce((sum, item) => sum + item.price, 0)
})
```

**好的：**
```js
import { ref, computed } from 'vue'

const items = ref([{ price: 10 }, { price: 20 }])
const total = computed(() =>
  items.value.reduce((sum, item) => sum + item.price, 0)
)
```

### 將過濾/排序等衍生邏輯保留在模板之外

**不好的：**
```vue
<template>
  <li v-for="item in items.filter(item => item.active)" :key="item.id">
    {{ item.name }}
  </li>

  <li v-for="item in getSortedItems()" :key="item.id">
    {{ item.name }}
  </li>
</template>

<script setup>
import { ref } from 'vue'

const items = ref([
  { id: 1, name: 'B', active: true },
  { id: 2, name: 'A', active: false }
])

function getSortedItems() {
  return [...items.value].sort((a, b) => a.name.localeCompare(b.name))
}
</script>
```

**好的：**
```vue
<script setup>
import { ref, computed } from 'vue'

const items = ref([
  { id: 1, name: 'B', active: true },
  { id: 2, name: 'A', active: false }
])

const visibleItems = computed(() =>
  items.value
    .filter(item => item.active)
    .sort((a, b) => a.name.localeCompare(b.name))
)
</script>

<template>
  <li v-for="item in visibleItems" :key="item.id">
    {{ item.name }}
  </li>
</template>
```

### 針對可重用的 class 或 style 邏輯使用 computed

**不好的：**
```vue
<template>
  <button :class="{ btn: true, 'btn-primary': type === 'primary' && !disabled, 'btn-disabled': disabled }">
    {{ label }}
  </button>
</template>
```

**好的：**
```vue
<script setup>
import { computed } from 'vue'

const props = defineProps({
  type: { type: String, default: 'primary' },
  disabled: Boolean,
  label: String
})

const buttonClasses = computed(() => ({
  btn: true,
  [`btn-${props.type}`]: !props.disabled,
  'btn-disabled': props.disabled
}))
</script>

<template>
  <button :class="buttonClasses">
    {{ label }}
  </button>
</template>
```

### 保持 computed getter 的純粹性，並將副作用交給 watch 處理

`computed` getter 應該只做衍生值的計算。不要在裡面改變其他狀態、呼叫 API、寫入儲存區或發送事件。
([參考資料](https://vuejs.org/guide/essentials/computed.html#best-practices))

**不好的：**

計算內的副作用

```js
const count = ref(0)

const doubled = computed(() => {
  // ❌ 副作用
  if (count.value > 10) console.warn('Too big!')
  return count.value * 2
})
```

**好的：**

純粹的 computed + 用 `watch()` 處理副作用

```js
const count = ref(0)
const doubled = computed(() => count.value * 2)

watch(count, (value) => {
  if (value > 10) console.warn('Too big!')
})
```

## `watch` 的最佳實踐

### 使用 `immediate: true` 來取代重複的初始呼叫

**不好的：**
```js
import { ref, watch, onMounted } from 'vue'

const userId = ref(1)

function loadUser(id) {
  // ...
}

onMounted(() => loadUser(userId.value))
watch(userId, (id) => loadUser(id))
```

**好的：**
```js
import { ref, watch } from 'vue'

const userId = ref(1)

watch(
  userId,
  (id) => loadUser(id),
  { immediate: true }
)
```

### 在 `watch` 中處理非同步副作用時記得進行清理

當監聽的來源變化很快時 (例如：搜尋框、篩選器)，應該要能取消前一次未完成的請求。

**好的：**

```js
const query = ref('')
const results = ref([])

watch(query, async (q, _prev, onCleanup) => {
  const controller = new AbortController()
  onCleanup(() => controller.abort())

  const res = await fetch(`/api/search?q=${encodeURIComponent(q)}`, {
    signal: controller.signal,
  })

  results.value = await res.json()
})
```
