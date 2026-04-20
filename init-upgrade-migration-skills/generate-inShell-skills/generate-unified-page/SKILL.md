---
name: generate-unified-page
description: 產生一般標準應用頁，這些頁面應該渲染在既有 app shell 內容區內，並重用 `@cx-rd/ui-kit` 的共用 layout primitive，但不可重建全域 shell。適用於 dashboard、CRUD 列表頁、搜尋 / 篩選頁、表單頁，以及只用自然語言描述「做一個畫面，裡面有按鈕、下拉選單、表格、panel、pagination」這類標準後台畫面；生成前仍必須先審計 UI-kit 是否已有可直接承接的 component / primitive。
---

# Generate Unified Page Skill

用於建立一般 in-shell 應用頁。

若使用者沒有指定 skill，只是用自然語言描述一個標準後台畫面，例如：

- 做一個有按鈕和下拉選單的畫面
- 做一個帶 filter、table、pagination 的頁面
- 重建一個搜尋與列表頁

只要它本質上屬於標準 in-shell page，就應納入本 skill 的判定範圍，但仍必須先走 `UI-kit adopt-first`。

## 核心規則

這個 skill 是給標準內容頁使用，不是給 UI-kit 已經完整擁有整頁 chrome 的 page template 使用。

例如：

- dashboard
- all-notifications
- settings

以下這些 full-page UI-kit 元件不可用本 skill 當成普通 section 或普通頁再包一次：

- `LoginPageComponent`

這些必須使用各自的 specialized skill。

## Layout Ownership

`MainLayoutComponent` 是全域 app shell owner。

凡是已經渲染在 shell 之內的標準頁：

- 不可再在頁面內渲染 `<lib-main-layout>`
- 不可再建立第二層 toolbar 或 sidebar
- 除非頁面本身確實需要，否則不要再引入競爭性的 scroll container

## UI-kit Adopt-First

在建立任何本地 HTML / SCSS 前，必須先審計 `@cx-rd/ui-kit` 的公開 exports 與 `.d.ts`。

規則如下：

- 若 UI-kit 已有可語義承接需求的 component / primitive，必須優先使用
- 最多只允許建立 thin adapter layer 來橋接資料、事件或路由
- 不得先自寫近似 UI，再用 CSS / wrapper 模仿 UI-kit
- 只有在 UI-kit 沒有合適 export，或現有 extension point 明確不足時，才允許 custom composition

這條規則同樣適用於未來新增的 form control、button、dialog、datatable、pagination、panel、stepper 等元件。

## 命名合約

- sidebar 第一個 section 的 component 名稱固定為 `dashboard`
- 對應 Angular class 必須為 `DashboardComponent`
- 頁面標題必須使用 Title Case

## 必要輸出

建立或更新頁面本地：

- `.component.ts`
- `.component.html`
- `.component.scss`

並在需要時同步更新 `app.routes.ts` 與 navigation config。

## Fail If

- 標準頁重建了全域 shell
- full-page UI-kit component 被當成普通頁 section 包裝
- 頁面重複建立 app-shell navigation、toolbar 或 layout chrome
- UI-kit 已有可 direct adopt 的 component / primitive，卻仍自建近似 UI
- 以 app-local wrapper 或自寫 DOM 骨架模仿既有 UI-kit component
- 使用 inline metadata
