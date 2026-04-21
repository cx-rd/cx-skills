---
name: generate-notification-page
description: 產生或更新使用 `@cx-rd/ui-kit` `AllNotificationsPageComponent` 的通知頁。用於建立 shell 內 notifications route、提供 `NotificationItem[]` 資料來源，並避免在 UI-kit 通知頁外再包重複的 header、filter、tabs、sticky 區塊或額外 page shell。
---

# Generate Notification Page Skill

用於建立或更新通知頁。

## 核心規則

`AllNotificationsPageComponent` 是功能完整的通知頁內容元件。

它可以渲染在既有 `MainLayout` app shell 之內，但通知頁本身的 header、filter 與 sticky 行為仍由 UI-kit 元件擁有。

- 不可在它上方再加一層 page header。
- 不可在外部重複建立 filter tabs、settings buttons 或 sticky controls。
- 除非既有設計系統明確要求，否則不可再加一層外部 padding page shell。

## 必要輸出

建立或更新：

- `src/app/notifications/notifications.component.ts`
- `src/app/notifications/notifications.component.html`
- `src/app/notifications/notifications.component.scss`

必須使用檔案分離。禁止 inline `template` 或 `styles`。

## 實作契約

1. 從 `@cx-rd/ui-kit` 匯入 `AllNotificationsPageComponent` 與 `NotificationItem`。
2. 在 consumer component 準備 `NotificationItem[]` 資料來源。
3. 直接渲染 `<lib-all-notifications [notifications]="notifications()">`。
4. 若 `@cx-rd/ui-kit` 已 export `ALL_NOTIFICATIONS_ROUTE` 或等效 route contract，註冊 route 與任何 bridge 時必須優先使用該 contract，不可自行猜測 path。
5. notifications route 預設註冊在既有 app shell 之下，除非 orchestrator 合約另有要求。
6. 讓 route host 高度與既有 app shell 相容。
7. 在 `app.routes.ts` 中註冊 notifications route。
8. 若 app 使用 UI-kit notification bell / popover，則 popover 內的 `View All Notifications` 動作必須橋接到 all-notifications route。
9. 若當前 UI-kit notification popover 尚未提供 `viewAll` 類型的正式 output / extension point，必須先修補 UI-kit contract 或明確標記 blocked；禁止用 DOM query、字串選 footer、手動事件委派等脆弱 workaround。
10. component 名稱固定為 `all-notification`，對應 Angular class 必須為 `AllNotificationComponent`。
11. 若需要可見標題字串，必須使用 Title Case。

## Shell Bridge Scope

此 skill 不只生成 notifications page consumer，也負責定義 notifications feature 與 app shell 的 route bridge。

- 必須更新 route host，例如 `app.routes.ts`。
- 若專案已有 `MainLayout` shell，必須更新擁有 `notificationAction` 或等效 shell event bridge 的 host 檔案。
- `View All Notifications` 的 destination 應以 notifications feature route contract 為單一真值來源，不可同時存在多組互相競爭的目的地。

## Scroll Ownership

通知頁的 page chrome、sticky header、filter 行為由 UI-kit 元件自己擁有，但它的 route 歸屬於 app shell。

主要 scroll container 應屬於 `lib-all-notifications` 內部的內容區，而不是外層 shell。

- 不可額外增加會與它衝突的外部 sticky header 或 top padding。
- 除非 app shell 本身已經規定，否則不可再建立另一個包住它的 scroll container。
- 不可把 notifications route 當成 shell 外獨立 landing page 重新建立第二層應用外框。
- 若通知內容滾動時帶動整個 shell 一起滾動，視為 scroll ownership 錯誤。
- 若修正 scroll 問題需要調整容器，應優先修正 notifications page host 與 UI-kit content panel 的高度閉合，不可先在 shell 外再包一層 workaround wrapper。
- 不可為了接 `View All Notifications` 而在 UI-kit popover 外再包一層自製 popover 或重畫 footer。

## Fail If

- `<lib-all-notifications>` 被外部重複 page header 包住。
- 頁面又額外建立 tabs 或 filters。
- 頁面引入與 UI-kit 衝突的 sticky 或 scroll 行為。
- UI-kit 已 export notifications route contract，卻仍硬寫另一個 path 或讓 agent 自行命名 route。
- 專案存在 UI-kit notification popover，但 `View All Notifications` 沒有被橋接到 notifications route。
- 以 DOM hack、手動 query footer 文字或其他非 contract 方式攔截 `View All Notifications`。
- component 名稱未遵守 `all-notification -> AllNotificationComponent`。
- 使用 inline metadata。

## 參考模板

```html
<lib-all-notifications [notifications]="notifications()"></lib-all-notifications>
```
