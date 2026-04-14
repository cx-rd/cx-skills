---
name: generate-settings-page
description: 產生或更新使用 `@cx-rd/ui-kit` `SettingsPageComponent` 的設定頁。用於建立 shell 內 settings route、提供 `SETTINGS_TABS` 與 `SETTINGS_SECTIONS`、同步 `:tabId` 之類的 route state，並保留 UI-kit 擁有的 header、tabs、sidebar、section scroll container 與 section 導覽行為。
---

# Generate Settings Page Skill

用於建立或更新設定頁。

## 核心規則

`SettingsPageComponent` 是功能完整的設定內容頁元件。

它已經擁有：

- page header
- save button
- top tabs
- 左側 section sidebar
- 右側 content area
- section scrolling 行為

consumer 端只負責設定與 route wiring。
它可以存在於既有 `MainLayout` app shell 之內，但不可因此重做 settings 內部布局。

## 必要輸出

建立或更新：

- `src/app/settings/settings.component.ts`
- `src/app/settings/settings.component.html`
- `src/app/settings/settings.component.scss`

選擇性：

- `src/app/settings/sections/*`，只有在應用程式真的需要自訂 section component 時才建立

但為了讓首次接手專案的人明確知道 app-side section extension point 的位置，至少必須建立：

- `src/app/settings/sections/` 資料夾

必須使用檔案分離。禁止 inline `template` 或 `styles`。

## 實作契約

1. 從 `@cx-rd/ui-kit` 匯入 `SettingsPageComponent`。
2. 透過 `multi: true` 提供 `SETTINGS_TABS` 與 `SETTINGS_SECTIONS`。
3. 需要 deep link 時，使用像 `/settings/:tabId` 這樣的 route-driven tab state。
4. 將 `title`、`activeTabId`，必要時 `activeSectionId` 傳給 `<lib-settings-page>`。
5. `tabChange`、`sectionScroll`、`save` 只用來做 app state integration，不可重做整個 settings 內部布局。
6. settings route 預設註冊在既有 app shell 之下，除非 orchestrator 合約另有要求。
7. component 名稱固定為 `settings`，對應 Angular class 必須為 `SettingsComponent`。
8. `title` 必須使用 Title Case。
9. 即使目前沿用 UI-kit 內建 sections，也必須保留 `src/app/settings/sections/` 作為後續自訂 section 的明確掛點。

## Ownership Rules

禁止額外新增：

- `<lib-settings-page>` 外層 page header
- 第二個 sidebar
- 第二組 tabs
- 另一層包住 settings content 的 card shell

app shell 可以擁有整體應用外框，但 settings 內部布局由設定頁元件自己擁有。
settings 應視為 shell 內內容頁，而不是 shell 外獨立頁。

## Scroll Rules

UI-kit settings page 擁有 section 內容區的 scrolling 行為。

- 不可再為 settings 內部建立競爭性的外層 scroll container。
- 不可在 consumer 端用 `scrollIntoView()` 去推動整個 app shell。
- 如果 scroll 行為要修，優先修 UI-kit `SettingsPageComponent` 本身，而不是在 app 頁面外掛 workaround wrapper。

## Fail If

- 缺少 `SETTINGS_TABS` 或 `SETTINGS_SECTIONS`
- consumer 在 `<lib-settings-page>` 外又包了一層 page header 或 card shell
- consumer 重複建立 tabs 或 section navigation
- scroll 行為掛在錯誤的 container 上
- 使用 inline metadata

## 參考模板

```html
<lib-settings-page
  [title]="title"
  [activeTabId]="activeTabId()"
  [activeSectionId]="activeSectionId()"
  (tabChange)="handleTabChange($event)"
  (sectionScroll)="handleSectionScroll($event)"
  (save)="onSave()"
>
</lib-settings-page>
```
