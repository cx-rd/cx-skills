---
name: verify-app-generation
description: 驗證 Angular 應用頁面是否符合結構、routing、shell ownership 與 UI-kit contract。用於檢查檔案分離、delegation record、route/layout 正確性、`@cx-rd/ui-kit` 使用方式、UI-kit adopt-first 規則、scroll container ownership、shell height baseline，以及 UI-kit 頁面元件是否在錯誤 ownership 模式下被使用。
---

# Verify App Generation Skill

用於頁面生成後的 fail-fast 驗證。

## 驗證順序

1. 先跑 mechanical audit。
2. 再讀 `delegation-record.ts`，確認 feature route 與 ownership 宣告。
3. 驗 UI-kit adopt-first 與 component 選型。
4. 驗 app-shell ownership 與 shell height baseline。
5. 接著依 feature specialized contract 驗每一頁。
6. 對 duplicated page chrome、錯誤 route placement、忽略 UI-kit 既有 export 或錯誤 scroll ownership 直接 fail。

## Mechanical Audit

若任一生成 component：

- 使用 inline `template` 或 `styles`
- 缺少應有的 `.ts`、`.html`、`.scss`
- `templateUrl` 或 `styleUrl` 指向錯誤

則直接 fail。

## Delegation Record Audit

必須先讀 `src/app/core/delegation-record.ts`。

每個 feature 至少必須有：

- `featureId`
- `featureType`
- `route`
- `ownership`
- `shellSurfaceOwner`
- `generatingSkill`
- `outputPath`
- `verificationTarget`

若 delegation record 缺少欄位，或 route / ownership 與實際 route wiring 不一致，直接 fail。

`ownership` 只允許以下值：

- `Full-Page`
- `In-Shell`

## UI Ownership Audit

Verifier 不可只靠 UI-kit 元件名字硬編碼判定 ownership。

必須以：

1. `delegation-record.ts` 的 `ownership`
2. specialized skill contract
3. route 是否位於 `MainLayout` shell 之下

三者綜合判定。

以下元件預設是功能完整頁面元件：

- `LoginPageComponent`
- `AllNotificationsPageComponent`
- `SettingsPageComponent`

但若 specialized skill 已明確宣告它在當前 application contract 中屬於 `In-Shell Content`，則 verifier 必須接受它渲染在 shell 內。

若 consumer page 又額外包出以下任一項，直接 fail：

- page header
- tabs
- filters
- settings shell
- card shell
- 會改變 sticky / scroll 模型的外層 spacing container

額外規則：

- `Full-Page` feature 不可註冊在 `MainLayout` shell children 之下
- `In-Shell` feature 不可在自己的 consumer component 中再渲染 `MainLayoutComponent`
- `In-Shell` feature 若使用功能完整的 UI-kit page component，允許其保有內部 header / tabs / sidebar，但不可再重建第二層 app shell

## UI-kit Adoption Audit

Verifier 必須確認生成流程有遵守「先用 UI-kit，沒有才自建」。

至少檢查：

- 是否先審計 `@cx-rd/ui-kit` 公開 exports / `.d.ts`
- 若 UI-kit 已有可承接需求的 export，是否直接採用該 component / template
- 若 UI-kit 已 export route constant 或 shell action contract，route wiring 是否採用該 contract
- 若最終採 custom UI，是否有明確說明為何 UI-kit 無法 direct adopt

以下任一情況直接 fail：

- UI-kit 已有可直接承接的 component / template，卻仍建立近似的 app-local UI
- UI-kit 已 export route constant 或既有 shell contract，卻仍自行猜 route path 或 action destination
- 以自寫 HTML / SCSS / wrapper 模仿 UI-kit 的主要 DOM 骨架或互動骨架
- 未先審計 UI-kit API，就直接宣稱需要自建

### Login Contract Audit

若 login feature 使用 `LoginPageComponent` 或直接渲染 `<lib-login>`：

- consumer template 必須綁定 `(submitted)`，將 UI-kit 的登入資料事件接回 app handler
- 不可把 `LoginPageComponent` 當成純展示頁而完全忽略登入事件
- 若 `<lib-login>` 存在但沒有 `(submitted)` 綁定，直接 fail
- 不可在 consumer 端再額外重建 account/password form 與 UI-kit 登入頁並存

## App Shell Audit

`MainLayoutComponent` 視為全域 shell owner。

如果 route 已經渲染在 shell 裡，又再建立以下任一項，直接 fail：

- 第二個 `MainLayoutComponent`
- 第二個 toolbar
- 第二個 sidebar

另外必須驗證 shell 高度模型：

- `MainContainer` / shell host 必須提供明確的 viewport 或 remaining-height baseline
- sidebar 不可依賴某個 child page 的內容高度才被撐開
- child page 不可用競爭性的 `100vh` / fixed-height wrapper 覆蓋 shell 高度歸屬

若 shell 高度基準不明確，或 child page 反向成為 shell 高度 owner，直接 fail。

## Shell Surface Audit

若 app 使用 `MainLayoutComponent` 或等效 UI-kit shell，必須驗證 shell surface 與 feature route bridge。

- 若存在 settings feature，且 app 使用 UI-kit user menu，必須存在 `settings` action 到 settings route 的 bridge。
- 若 UI-kit 已 export `SETTINGS_PAGE_ROUTE` 或等效 route contract，settings bridge 不可改指向其他自行命名 path。
- 若存在 notifications feature，且 app 使用 UI-kit notification bell / popover，必須存在 `View All Notifications` 到 notifications route 的 bridge。
- 若 UI-kit 已 export `ALL_NOTIFICATIONS_ROUTE` 或等效 route contract，notification bridge 不可改指向其他自行命名 path。
- 若使用 DOM query、footer 文字比對、手動事件委派等非 contract 方式攔截 UI-kit popover 行為，直接 fail。
- 若未經使用者明確要求，卻因為有 settings page 而自行新增 competing sidebar settings entry，應視為 shell surface ownership 漂移。

## Scroll Ownership Audit

若 scroll 行為掛在錯誤 container，直接 fail。

例如：

- settings section click 造成外層 app shell 移動，而不是 settings content panel 自己滾動
- notifications page 外層 wrapper 破壞元件自己的 sticky header 與 tabs 行為
- child page 以額外 wrapper 改變 UI-kit 內容元件的主要 scroll container
- shell 內容區本應固定，而 sidebar 或整個 app shell 跟著 feature internal scroll 一起移動

### Notifications Scroll Checks

若 notifications feature 使用 `AllNotificationsPageComponent`：

- 主要 scroll container 應屬於 notifications page 內部內容區
- 不可由外層 shell wrapper 改寫其 sticky header / tabs / filter 行為
- 若 notifications 捲動時造成整個 shell 異常位移，直接 fail
- `View All Notifications` 的 bridge 應導向 notifications feature route，而非只關閉 popover 或停留在目前頁

### Settings Scroll Checks

若 settings feature 使用 `SettingsPageComponent`：

- section navigation 應驅動 settings content panel，而非整個 app shell
- 不可在 consumer 端用 workaround wrapper 改寫 settings 內部 scroll container
- 若 section click 導致 app shell 被推動而非 settings content panel 移動，直接 fail
- 若 shell 存在 settings action，其 destination 應導向 settings feature route，而非其他臨時頁或 placeholder route

## Naming Contract Audit

Verifier 必須依母規格與 specialized skill 檢查命名。

至少包含：

- component class 是否符合 skill 指定名稱
- selector 是否符合 `app-{component-kebab}`
- `pageTitle` / `title` 是否符合 Title Case contract

若 skill 已明確指定 component 名稱，驗證時不得自行猜測。

## Success Criteria

只有在以下條件全部成立時才可通過：

- 檔案分離正確
- delegation record 完整且與 route wiring 一致
- route wiring 正確
- shell ownership 沒有重複
- shell height baseline 正確
- shell surface 與 route bridge 正確
- UI-kit adopt-first 規則有被遵守
- UI-kit 功能完整頁面元件的 ownership 使用方式正確
- login 頁的 `submitted` contract 已正確接線
- full-page UI-kit component 沒有被錯誤重包
- naming contract 正確
- scroll ownership 位於正確 container
