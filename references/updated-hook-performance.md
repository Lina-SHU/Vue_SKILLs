---
title: 避免在 Updated 鉤子中進行昂貴的操作
impact: MEDIUM
impactDescription: 在 updated 鉤子中的繁重計算會導致性能瓶頸和潛在的無限迴圈
type: capability
tags: [vue3, vue2, lifecycle, updated, performance, optimization, reactivity]
---

# 避免在 Updated 鉤子中進行昂貴的操作

**影響：MEDIUM** - `updated` 鉤子在導致重新渲染的每次反應性狀態改變後運行。在此處放置昂貴的操作、API 呼叫或狀態變更可能導致嚴重性能下降、無限迴圈和掉幀至最佳 60fps 閾值以下。

謹慎使用 `updated`/`onUpdated` 處理無法由監視器或計算屬性處理的 DOM 後更新操作。對於大多數反應性數據處理，偏好監視器（`watch`/`watchEffect`），它們對觸發回調的內容提供更多控制。

## 任務清單

- 永遠不要在 updated 鉤子中進行 API 呼叫
- 永遠不要在 updated 內部變更反應性狀態（會導致無限迴圈）
- 在操作之前使用條件檢查來驗證更新是否相關
- 對於回應特定數據改變，偏好 `watch` 或 `watchEffect`
- 如果 updated 操作很昂貴，使用節流/防波
- 保留 updated 用於低階 DOM 同步任務

**不好的：**
```javascript
// 不好：updated 中的 API 呼叫 - 在每次重新渲染時觸發
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
// 不好：updated 中的狀態變更 - 無限迴圈
export default {
  data() {
    return { renderCount: 0 }
  },
  updated() {
    // 此導致另一次更新，這觸發 updated 再次！
    this.renderCount++ // 無限迴圈
  }
}
```

```javascript
// 不好：每次更新時的繁重計算
export default {
  updated() {
    // 昂貴的操作在每次按鍵、每次狀態改變時運行
    this.processedData = this.heavyComputation(this.rawData)
    this.analytics = this.calculateMetrics(this.allData)
  }
}
```

**好的：**
```javascript
import debounce from 'lodash-es/debounce'

// 好的：為特定數據改變使用監視器
export default {
  data() {
    return { items: [] }
  },
  watch: {
    // 僅當 items 實際改變時觸發
    items: {
      handler(newItems) {
        this.syncToServer(newItems)
      },
      deep: true
    }
  },
  methods: {
    syncToServer: debounce(function(items) {
      fetch('/api/sync', {
        method: 'POST',
        body: JSON.stringify(items)
      })
    }, 500)
  }
}
```

```vue
<!-- 好的：搭配目標監視器的 Composition API -->
<script setup>
import { ref, watch, onUpdated } from 'vue'
import { useDebounceFn } from '@vueuse/core'

const items = ref([])
const scrollContainer = ref(null)

// 監視特定數據 - 不是所有更新
watch(items, (newItems) => {
  syncToServer(newItems)
}, { deep: true })

const syncToServer = useDebounceFn((items) => {
  fetch('/api/sync', { method: 'POST', body: JSON.stringify(items) })
}, 500)

// 僅為 DOM 同步使用 onUpdated
onUpdated(() => {
  // 如果內容改變高度，僅滾動到底部
  if (scrollContainer.value) {
    scrollContainer.value.scrollTop = scrollContainer.value.scrollHeight
  }
})
</script>
```

```javascript
// 好的：updated 鉤子中的條件檢查
export default {
  data() {
    return {
      content: '',
      lastSyncedContent: ''
    }
  },
  updated() {
    // 僅在滿足特定條件時操作
    if (this.content !== this.lastSyncedContent) {
      this.syncContent()
      this.lastSyncedContent = this.content
    }
  },
  methods: {
    syncContent: debounce(function() {
      // 同步邏輯
    }, 300)
  }
}
```

## Updated 鉤子的有效使用情況

```javascript
// 好的：低階 DOM 同步
export default {
  updated() {
    // 將第三方庫與 Vue 的 DOM 同步
    this.thirdPartyWidget.refresh()

    // 內容改變後更新滾動位置
    this.$nextTick(() => {
      this.maintainScrollPosition()
    })
  }
}
```

## 對推導數據偏好計算屬性

```javascript
// 不好：在 updated 中計算推導數據
export default {
  data() {
    return { numbers: [1, 2, 3, 4, 5] }
  },
  updated() {
    this.sum = this.numbers.reduce((a, b) => a + b, 0) // 導致另一次更新！
  }
}

// 好的：改用計算屬性
export default {
  data() {
    return { numbers: [1, 2, 3, 4, 5] }
  },
  computed: {
    sum() {
      return this.numbers.reduce((a, b) => a + b, 0)
    }
  }
}
```
