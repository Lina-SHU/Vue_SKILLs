---
description: 此文件適用於處理 Vue.js 的相關任務。我們強烈建議使用 Composition API 搭配 `<script setup>` 作為標準實踐。內容涵蓋 Vue 3、SSR、Volar，適用於任何涉及 Vue、.vue 檔案、Vue Router、Pinia 或 Vite with Vue 的工作。除非專案明確要求使用 Options API，否則應一律採用 Composition API。
---

# Vue.js 最佳實踐工作流程

請將此技能文件作為一份指導手冊。除非使用者明確要求，否則請遵循以下工作流程。

## 核心原則
- **保持狀態的可預測性：** 將狀態視為唯一的「真實來源」(Source of Truth)，並由此衍生出所有其他狀態。
- **確保資料流的明確性：** 在多數情況下，資料流應遵循單向原則：Props 由上而下傳遞，Events 由下而上觸發。
- **傾向於構建小而專注的元件：** 這類元件更易於測試、重用與維護。
- **避免不必要的重複渲染：** 善用 `computed` 屬性與 `watch` 監聽器。
- **可讀性至關重要：** 編寫清晰、易於理解的程式碼。

## 1) 動手前，先確認架構（必要步驟）

- 預設技術棧：Vue 3 + Composition API + `<script setup>`。
- 如果專案明確使用 Options API，請載入 `vue-options-api-best-practices` 技能。
- 如果專案明確使用 JSX，請載入 `vue-jsx-best-practices` 技能。

### 1.1 必讀的核心參考文件（必要步驟）

- 在執行任何 Vue 相關任務前，請務必閱讀並應用以下核心參考文件的指引：
  - `references/reactivity.md`
  - `references/sfc.md`
  - `references/component-data-flow.md`
  - `references/composables.md`
- 在整個任務過程中，應隨時參考這些文件，而不僅是在遇到特定問題時才查閱。

### 1.2 編寫程式前，先規劃元件邊界（必要步驟）

在實作任何稍具複雜度的功能前，請先建立一份簡易的元件結構圖。

- 用一句話定義每個元件的單一職責。
- 預設將入口頁面 (Entry/Root) 和路由層級的視圖元件 (Route-level View) 當作「組合層」(Composition Facade)。
- 將功能性的 UI 和商業邏輯移出這些組合層元件，除非任務目標本身就是一個極微小的單檔案範例。
- 為結構圖中的每個子元件定義其 Props 和 Emits 的契約 (contract)。
- 當需要新增一個以上的元件時，優先採用功能性的資料夾結構，例如 `components/<feature>/...`、`composables/use<Feature>.js`。

## 2) 應用 Vue 的基礎實踐（必要步驟）

這些是基礎且必須掌握的實踐。請借助在 `1.1` 節已載入的核心參考文件，在每個 Vue 任務中應用所有這些原則。

### 反應性 (Reactivity)

- 必讀參考文件 (來自 `1.1` 節): [reactivity](references/reactivity.md)
- 保持原始狀態 (raw state, 即 `ref` / `reactive`) 的最小化，並盡可能使用 `computed` 來衍生狀態。
- 必要時，使用 `watch` 來處理副作用 (side effects)。
- 避免在模板 (template) 中進行耗費大量運算資源的邏輯。

### SFC 結構與模板安全

- 必讀參考文件 (來自 `1.1` 節): [sfc](references/sfc.md)
- 請遵循 `<script>` → `<template>` → `<style>` 的順序來組織單一檔案元件 (SFC) 的區塊。
- 保持 SFC 的職責單一；將過於龐大的元件進行拆分。
- 保持模板的宣告性，將邏輯判斷與衍生狀態移至 `<script>` 中。
- 遵循 Vue 模板的安全規則 (例如：謹慎使用 `v-html`、為列表渲染指定 `key`、選擇合適的條件渲染方式)。

### 保持元件的專注性

當一個元件承擔了**一個以上的明確職責**時（例如：同時處理資料流程與 UI 呈現，或包含多個獨立的 UI 區塊），就應該進行拆分。

- 優先選擇**更小的元件 + Composables** 的組合，而不是單一的「巨無霸元件」。
- 將**特定的 UI 區塊**移至子元件中（Props 傳入，Events 發出）。
- 將**狀態管理或副作用邏輯**移至 Composables 中（例如 `useXxx()`）。

遵循以下客觀原則來拆分元件。若**任何一項**條件成立，就應該進行拆分：

- 當元件同時負責資料處理（狀態管理、流程控制）以及複雜的畫面呈現邏輯時。
- 當元件包含了 3 個或更多個不同的 UI 區塊時（例如：表單、篩選器、列表、頁腳狀態列）。
- 當模板中的某個區塊重複出現，或可以被抽象化成一個可重用的部分時（例如：表格的每一行、卡片、列表項目）。

入口頁面 (Entry/Root) 與路由層級視圖 (Route-level View) 的規則：

- 保持入口頁面與路由層級視圖的簡潔：它們應主要用於應用層級的版面配置 (layout)、啟用全域 Provider，以及組合各個功能模組。
- 當這些視圖需要整合獨立的功能區塊時，不應將完整的實作邏輯直接放在視圖元件中。
- 對於 CRUD 或列表類型功能（例如：待辦事項、表格、商品目錄、收件匣），至少應拆分為：
  - 功能容器元件 (Feature Container Component)
  - 輸入/表單元件 (Input/Form Component)
  - 列表（或列表項目）元件 (List/Item Component)
  - 頁腳/操作列或篩選/狀態元件 (Footer/Actions or Filter/Status Component)
- 只有在開發極小規模、一次性的示範範例時，才允許將所有邏輯放在單一檔案中。如果選擇此作法，請明確說明為何不需要拆分。

### 元件的資料流

- 必讀參考文件 (來自 `1.1` 節): [component-data-flow](references/component-data-flow.md)
- 使用「Props 向下傳遞，Events 向上觸發」作為主要的資料流模型。
- 僅在真正需要雙向綁定的元件契約時才使用 `v-model`。
- 僅在處理深層跨元件依賴或共享上下文時，才使用 `provide/inject`。
- 保持 Props 與 Events 契約的明確性與易讀性。

### Composables

- 必讀參考文件 (來自 `1.1` 節): [composables](references/composables.md)
- 當一段邏輯需要被重用、具有狀態，或包含複雜的副作用時，應將其提取到 Composables 中。
- 保持 Composable 的 API 小巧、清晰且可預測。
- 將功能邏輯與負責畫面呈現的元件分離。

## 3) 功能驗證後，再進行效能優化

效能優化應在主要功能完成後才進行。在核心功能被完整實作並通過驗證之前，請不要進行過早的優化。

- 大型列表的渲染瓶頸 -> [perf-virtualize-large-lists](references/perf-virtualize-large-lists.md)
- 靜態內容的子樹發生不必要的重新渲染 -> [perf-v-once-v-memo-directives](references/perf-v-once-v-memo-directives.md)
- 在頻繁更新的列表渲染路徑中過度抽象 -> [perf-avoid-component-abstraction-in-lists](references/perf-avoid-component-abstraction-in-lists.md)
- 耗費資源的 `update` 鉤子觸發過於頻繁 -> [updated-hook-performance](references/updated-hook-performance.md)

## 4) 完成前的最終檢核清單

- 核心功能運作正常且符合需求。
- 所有必讀的參考文件都已被閱讀並應用。
- 反應性狀態模型保持最小化且可預測。
- SFC 的結構與模板規則得到遵循。
- 元件保持職責單一，並在必要時進行了良好的拆分。
- 入口頁面與路由層級視圖維持其作為「組合層」的角色，除非是明確的小型示範範例。
- 元件的拆分決策清晰且有合理的依據（明確的職責邊界）。
- 資料流的契約明確且易於理解。
- 在邏輯重用或複雜度證明其合理性時，使用了 Composables。
- 狀態管理與副作用邏輯已移至 Composables (如果適用)。
- 選擇性的功能僅在需求明確要求時才使用。
- 效能相關的調整僅在功能開發完成後才應用。
