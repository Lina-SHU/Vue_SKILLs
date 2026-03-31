---
title: 元件數據流最佳實踐
impact: HIGH
impactDescription: 清晰的元件間數據流防止狀態錯誤、陳舊 UI 和脆弱耦合
tags: [vue3, props, emits, v-model, provide-inject, data-flow]
---

# 元件數據流最佳實踐

**影響：HIGH** - Vue 元件在數據流明確時保持可靠：props 向下、events 向上、`v-model` 處理雙向繫結，provide/inject 支援跨樹依賴項。模糊這些邊界會導致陳舊狀態、隱藏耦合和難以偵錯的 UI。

Vue.js 中數據流的主要原則是 **Props Down / Events Up**。這是最容易維護的預設，並且單向流擴展性良好。

## 任務清單

- 將 props 視為唯讀輸入
- 對元件通訊使用 props/emit；保留 refs 用於命令式動作
- 當 refs 需要用於命令式 API 時，公開必要的 API
- 發送事件而不是直接變更父狀態
- 在現代 Vue (3.4+) 中使用 `defineModel` 用於 v-model
- 在子元件中有意處理 v-model 修飾符
- 為 provide/inject 鍵使用符號以避免 props 鑽孔（超過 ~3 層）
- 在供應商中或公開明確的動作中保留變更

## Props：單向數據向下

Props 是輸入。不要在子元件中變更它們。

**不好的：**
```vue
<script setup>
const props = defineProps({ count: Number })

function increment() {
  props.count++
}
</script>
```

**好的：**

如果狀態需要改變，發送事件、使用 `v-model` 或建立本地副本。

## 偏好 props/emit 而不是元件 refs

**不好的：**
```vue
<script setup>
import { ref } from 'vue'
import UserForm from './UserForm.vue'

const formRef = ref(null)

function submitForm() {
  if (formRef.value.isValid) {
    formRef.value.submit()
  }
}
</script>

<template>
  <UserForm ref="formRef" />
  <button @click="submitForm">Submit</button>
</template>
```

**好的：**
```vue
<script setup>
import UserForm from './UserForm.vue'

function handleSubmit(formData) {
  api.submit(formData)
}
</script>

<template>
  <UserForm @submit="handleSubmit" />
</template>
```

## 當命令式存取是必要時對元件 refs 進行類型化

默認偏好 props/emits。當父級必須呼叫公開的子方法時，明確為 ref 類型化，並從子元件使用 `defineExpose` 公開僅預期的 API。

**不好的：**
```vue
<script setup>
import { ref, onMounted } from 'vue'
import DialogPanel from './DialogPanel.vue'

const panelRef = ref(null)

onMounted(() => {
  panelRef.value.open()
})
</script>

<template>
  <DialogPanel ref="panelRef" />
</template>
```

**好的：**
```vue
<!-- DialogPanel.vue -->
<script setup>
function open() {}

defineExpose({ open })
</script>
```

```vue
<!-- Parent.vue -->
<script setup>
import { onMounted, useTemplateRef } from 'vue'
import DialogPanel from './DialogPanel.vue'

// Vue 3.5+ 搭配 useTemplateRef
const panelRef = useTemplateRef('panelRef')

onMounted(() => {
  panelRef.value?.open()
})
</script>

<template>
  <DialogPanel ref="panelRef" />
</template>
```

## Emits：明確的事件向上

元件事件不會冒泡。如果父級需要了解事件，請明確重新發送它。

**不好的：**
```vue
<!-- Parent 期望來自孫元件的 "saved"，但它不會冒泡 -->
<Child @saved="onSaved" />
```

**好的：**
```vue
<!-- Child.vue -->
<script setup>
const emit = defineEmits(['saved'])

function onGrandchildSaved(payload) {
  emit('saved', payload)
}
</script>

<template>
  <Grandchild @saved="onGrandchildSaved" />
</template>
```

**事件命名：** 在模板中使用 kebab-case，在指令碼中使用 camelCase：
```vue
<script setup>
const emit = defineEmits(['updateUser'])
</script>

<template>
  <ProfileForm @update-user="emit('updateUser', $event)" />
</template>
```

## `v-model`：可預測的雙向繫結

默認使用 `defineModel` 用於元件繫結，在輸入時發送更新。僅當您使用 Vue < 3.4 時才使用 `modelValue` + `update:modelValue` 模式。

**不好的：**
```vue
<script setup>
const props = defineProps({ value: String })
</script>

<template>
  <input :value="props.value" @input="$emit('input', $event.target.value)" />
</template>
```

**好的 (Vue 3.4+)：**
```vue
<script setup>
const model = defineModel({ type: String })
</script>

<template>
  <input v-model="model" />
</template>
```

**好的 (Vue < 3.4)：**
```vue
<script setup>
const props = defineProps({ modelValue: String })
const emit = defineEmits(['update:modelValue'])
</script>

<template>
  <input
    :value="props.modelValue"
    @input="emit('update:modelValue', $event.target.value)"
  />
</template>
```

如果您需要在變更後立即獲得更新的值，請使用輸入事件值或在父級中使用 `nextTick`。

