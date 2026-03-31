---
title: Composable 組織模式
impact: MEDIUM
impactDescription: 良好結構的 composable 能提升程式碼的可維護性、可重用性及更新效能
tags: [vue3, composables, composition-api, code-organization, api-design, readonly, utilities]
---

# Composable 組織模式

**影響：MEDIUM** - 將 composable 視為可重複使用、具備狀態的建構區塊，並根據其功能邏輯來組織程式碼。這麼做能讓大型元件更容易維護，同時避免產生難以除錯的變更和 API 設計上的問題。

## 任務清單

- 透過組合小巧、功能單一的 composable 來建構複雜的行為
- 當 composable 含有多個可選參數時，使用選項物件模式
- 回傳唯讀狀態，並透過明確的函式來進行更新
- 將純粹的工具函式保持原樣，不要轉換成 composable
- 根據功能邏輯來組織 composable 與元件的程式碼，並在元件規模變大時將邏輯提取至 composable 中

## 透過組合更小的單元來建構 Composable

**不好的：**
```vue
<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

const x = ref(0)
const y = ref(0)
const inside = ref(false)
const el = ref(null)

function onMove(e) {
  x.value = e.pageX
  y.value = e.pageY
  if (!el.value) return
  const r = el.value.getBoundingClientRect()
  inside.value = x.value >= r.left && x.value <= r.right &&
    y.value >= r.top && y.value <= r.bottom
}

onMounted(() => window.addEventListener('mousemove', onMove))
onUnmounted(() => window.removeEventListener('mousemove', onMove))
</script>
```

**好的：**
```javascript
// composables/useEventListener.js
import { onMounted, onUnmounted, toValue } from 'vue'

export function useEventListener(target, event, callback) {
  onMounted(() => toValue(target).addEventListener(event, callback))
  onUnmounted(() => toValue(target).removeEventListener(event, callback))
}
```

```javascript
// composables/useMouse.js
import { ref } from 'vue'
import { useEventListener } from './useEventListener'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)

  useEventListener(window, 'mousemove', (e) => {
    x.value = e.pageX
    y.value = e.pageY
  })

  return { x, y }
}
```

```javascript
// composables/useMouseInElement.js
import { computed } from 'vue'
import { useMouse } from './useMouse'

export function useMouseInElement(elementRef) {
  const { x, y } = useMouse()

  const isOutside = computed(() => {
    if (!elementRef.value) return true
    const rect = elementRef.value.getBoundingClientRect()
    return x.value < rect.left || x.value > rect.right ||
      y.value < rect.top || y.value > rect.bottom
  })

  return { x, y, isOutside }
}
```

## 為 Composable 參數使用選項物件模式

**不好的：**
```javascript
export function useFetch(url, method, headers, timeout, retries, immediate) {
  // 參數順序容易混淆，不易閱讀
}

useFetch('/api/users', 'GET', null, 5000, 3, true)
```

**好的：**
```javascript
export function useFetch(url, options = {}) {
  const {
    method = 'GET',
    headers = {},
    timeout = 30000,
    retries = 0,
    immediate = true
  } = options

  // implementation
  return { method, headers, timeout, retries, immediate }
}

useFetch('/api/users', {
  method: 'POST',
  timeout: 5000,
  retries: 3
})
```

```javascript
export function useCounter(options = {}) {
  const { initial = 0, min = -Infinity, max = Infinity, step = 1 } = options
  // implementation
}
```

## 回傳唯讀狀態並提供明確的更新函式

**不好的：**
```javascript
export function useCart() {
  const items = ref([])
  const total = computed(() => items.value.reduce((sum, item) => sum + item.price, 0))
  return { items, total } // state 可被外部任意修改
}

const { items } = useCart()
items.value.push({ id: 1, price: 10 })
```

**好的：**
```javascript
import { ref, computed, readonly } from 'vue'

export function useCart() {
  const _items = ref([])

  const total = computed(() =>
    _items.value.reduce((sum, item) => sum + item.price * item.quantity, 0)
  )

  function addItem(product, quantity = 1) {
    const existing = _items.value.find(item => item.id === product.id)
    if (existing) {
      existing.quantity += quantity
      return
    }
    _items.value.push({ ...product, quantity })
  }

  function removeItem(productId) {
    _items.value = _items.value.filter(item => item.id !== productId)
  }

  return {
    items: readonly(_items),
    total,
    addItem,
    removeItem
  }
}
```

## 讓工具函式保持純粹

**不好的：**
```javascript
export function useFormatters() {
  const formatDate = (date) => new Intl.DateTimeFormat('en-US').format(date)
  const formatCurrency = (amount) =>
    new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(amount)
  return { formatDate, formatCurrency }
}

const { formatDate } = useFormatters()
```

**好的：**
```javascript
// utils/formatters.js
export function formatDate(date) {
  return new Intl.DateTimeFormat('en-US').format(date)
}

export function formatCurrency(amount) {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD'
  }).format(amount)
}
```

```javascript
// composables/useInvoiceSummary.js
import { computed } from 'vue'
import { formatCurrency } from '@/utils/formatters'

export function useInvoiceSummary(invoiceRef) {
  const totalLabel = computed(() => formatCurrency(invoiceRef.value.total))
  return { totalLabel }
}
```

## 根據功能邏輯組織 Composable 與元件

**不好的：**
```vue
<script setup>
import { ref, computed, watch, onMounted } from 'vue'

const searchQuery = ref('')
const items = ref([])
const selected = ref(null)
const showModal = ref(false)
const sortBy = ref('name')
const filter = ref('all')
const loading = ref(false)

const filtered = computed(() => items.value.filter(i => i.category === filter.value))
function openModal() { showModal.value = true }
const sorted = computed(() => [...filtered.value].sort(/* ... */))
watch(searchQuery, () => { /* ... */ })
onMounted(() => { /* ... */ })
</script>
```

**好的：**
```vue
<script setup>
import { useItems } from '@/composables/useItems'
import { useSearch } from '@/composables/useSearch'
import { useSelectionModal } from '@/composables/useSelectionModal'

// 數據
const { items, loading, fetchItems } = useItems()

// 搜索/篩選/排序
const { query, visibleItems } = useSearch(items)

// 選擇 + 模態
const { selectedItem, isModalOpen, selectItem, closeModal } = useSelectionModal()
</script>
```

```javascript
// composables/useItems.js
import { ref, onMounted } from 'vue'

export function useItems() {
  const items = ref([])
  const loading = ref(false)

  async function fetchItems() {
    loading.value = true
    try {
      items.value = await api.getItems()
    } finally {
      loading.value = false
    }
  }

  onMounted(fetchItems)
  return { items, loading, fetchItems }
}
```
