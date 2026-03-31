---
title: 避免在 updated 生命週期鉤子中執行昂貴操作
impact: MEDIUM
impactDescription: 在 updated 鉤子中執行耗時的計算或 I/O 操作，不僅會拖慢效能，還極易引發無限更新迴圈。
tags: [vue3, vue2, lifecycle, updated, performance, optimization, reactivity]
---

# 避免在 updated 生命週期鉤子中執行昂貴操作

**影響：中等 (MEDIUM)**

`updated` (或 Composition API 中的 `onUpdated`) 鉤子會在每一次因響應式狀態變更而觸發重新渲染後執行。這意味著它是一個非常敏感且執行頻繁的函式。

在此鉤子中放置昂貴的操作 (如 API 請求、複雜計算) 或會再次修改狀態的程式碼，極易導致以下問題：
- **效能下降**：每次更新都觸發耗時操作，可能導致頁面掉幀，無法達到流暢的 60fps。
- **無限迴圈**：如果在 `updated` 中修改了響應式狀態，會觸發另一次更新，從而再次呼叫 `updated`，形成死循環。

`updated` 鉤子的主要用途是處理那些**必須在 DOM 更新後**才能執行的操作，且這些操作無法透過 `watch` 或 `computed` 來完成。對於絕大多數的數據監聽與響應，**應優先使用 `watch`**，因為它能提供更精確、更可控的觸發條件。

## 檢查清單

-   **絕對不要**在 `updated` 鉤子中直接發起 API 請求。
-   **絕對不要**在 `updated` 鉤子中直接修改響應式狀態，以防無限迴圈。
-   如果必須在 `updated` 中執行操作，請務必加入條件檢查，確保只在必要時執行。
-   優先使用 `watch` 或 `watchEffect` 來響應**特定**數據的變化。
-   如果 `updated` 中的操作確實無法避免且較為昂貴，考慮使用 `debounce` (防抖) 或 `throttle` (節流) 進行效能控制。
-   將 `updated` 的使用場景限制在與 DOM 同步相關的底層任務上。

---

**不好的範例：**

```javascript
// 錯誤 1：在 updated 中發起 API 請求
// 這會在每一次重新渲染後都觸發 API 請求，造成資源浪費和不必要的後端壓力。
export default {
  data() {
    return { items: [], lastUpdate: null }
  },
  updated() {
    // 此動作在每次單一狀態改變後運行！
    fetch('/api/sync', {
      method: 'POST',
      body: JSON.stringify(this.items)
    })
  }
}
```

```javascript
// 錯誤 2：在 updated 中修改狀態，導致無限迴圈
export default {
  data() {
    return { renderCount: 0 };
  },
  updated() {
    // 1. 狀態改變，觸發 updated
    // 2. this.renderCount++ 再次改變狀態
    // 3. 觸發另一次更新 -> 再次呼叫 updated -> 無限迴圈...
    this.renderCount++;
  }
}
```

```javascript
// 錯誤 3：在 updated 中執行昂貴的計算
// 任何微小的狀態變化（例如使用者在輸入框中打一個字）都會觸發此昂貴計算。
export default {
  updated() {
    this.processedData = this.heavyComputation(this.rawData);
  }
}
```

---

**好的範例：**

**1. 使用 `watch` 監聽特定數據的變化**

這是最推薦的替代方案。`watch` 只在被監聽的數據源發生變化時才會執行。

```javascript
// Options API 寫法
export default {
  watch: {
    // 僅在 `items` 實際改變時才觸發
    items: {
      handler(newItems) {
        this.syncToServer(newItems);
      },
      deep: true // 深度監聽，適用於物件或陣列
    }
  },
  methods: {
    // 使用 debounce (防抖) 來避免過於頻繁地觸發
    syncToServer: debounce(function(items) {
      fetch('/api/sync', { /* ... */ });
    }, 500)
  }
}
```

**2. Composition API 的寫法**

Composition API 提供了更靈活的 `watch` 和 `onUpdated` 組合。

```vue
<script setup>
import { ref, watch, onUpdated } from 'vue';
import { useDebounceFn } from '@vueuse/core';

const items = ref([]);
const scrollContainer = ref(null);

// 方案一：監聽特定數據變化，執行邏輯
watch(items, (newItems) => {
  syncToServer(newItems);
}, { deep: true });

const syncToServer = useDebounceFn((items) => {
  fetch('/api/sync', { /* ... */ });
}, 500);

// 方案二：僅在必要時，為 DOM 同步操作使用 onUpdated
onUpdated(() => {
  // 例如：當聊天室內容更新後，自動滾動到底部
  if (scrollContainer.value) {
    scrollContainer.value.scrollTop = scrollContainer.value.scrollHeight;
  }
});
</script>
```

**3. 在 `updated` 中加入條件檢查**

如果實在無法避免，務必加入條件判斷，以防止不必要的執行。

```javascript
export default {
  updated() {
    // 僅在 content 的值確實發生了變化時才執行同步
    if (this.content !== this.lastSyncedContent) {
      this.syncContent();
      this.lastSyncedContent = this.content;
    }
  },
  methods: {
    syncContent: debounce(function() { /* ... */ }, 300)
  }
}
```

## `updated` 鉤子的合理使用場景

-   **與第三方函式庫的 DOM 同步**：當您使用一個不基於 Vue 的函式庫（例如圖表庫、地圖庫）時，可能需要在 Vue 更新 DOM 後，手動呼叫該函式庫的 refresh 或 update 方法。
-   **手動的 DOM 操作**：例如，在 DOM 更新後計算元素尺寸、位置，或手動調整滾動條位置。

```javascript
export default {
  updated() {
    // 在 Vue 更新 DOM 後，通知第三方小工具進行刷新
    this.thirdPartyWidget.refresh();

    // 在下次 DOM 更新循環結束後，執行滾動位置維護
    this.$nextTick(() => {
      this.maintainScrollPosition();
    });
  }
}
```

## 衍生數據應使用 `computed`

**絕對不要**在 `updated` 鉤子中計算衍生數據，這不僅效能低下，還可能觸發無限迴圈。請使用 `computed` 屬性。

```javascript
// 不好的示範
export default {
  updated() {
    // 錯誤！這會修改 `this.sum`，從而觸發另一次更新！
    this.sum = this.numbers.reduce((a, b) => a + b, 0);
  }
}

// 正確的示範：使用 computed 屬性
export default {
  computed: {
    // `sum` 會在 `numbers` 改變時自動且高效地重新計算
    sum() {
      return this.numbers.reduce((a, b) => a + b, 0);
    }
  }
}
```
