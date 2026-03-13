---
name: generate-unified-page
description: 產生一般標準應用頁，這些頁面應該渲染在既有 app shell 內容區內，並重用 `@cx-rd/ui-kit` 的共用 layout primitive，但不可重建全域 shell。適用於 dashboard、all-notifications、Settings 這類標準後台頁。
---

# Generate Unified Page Skill

用於建立一般 in-shell 應用頁。

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
- 使用 inline metadata
