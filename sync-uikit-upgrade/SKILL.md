---
name: sync-uikit-upgrade
description: 將一個已初始化且已套用過 UI-kit 模板或 contract 的 Angular 專案，同步升級至最新版 @cx-rd/ui-kit 的整合合約。此技能專注於 contract-aware migration、最小侵入式 patch、缺失整合點補齊、既有 component API 對齊，以及升版後的 Hard Gate 驗證。不得重建整個 feature，不得覆蓋 business logic。
---

# sync-uikit-upgrade

## 一、技能定位
本技能是一個 **Contract-Aware Migration Skill**，用於處理：
- 已初始化的 Angular 專案
- 已套用過 UI-kit 模板、wrapper、token 或 contract 的專案
- 在 UI-kit 升版後，需要將專案重新同步到最新版整合合約的情境

本技能不是頁面生成器，也不是專案初始化器。
本技能的主要責任是：
1. 偵測目前專案如何使用 UI-kit
2. 與最新版 UI-kit contract 比對差異
3. 以最小侵入方式套用必要 patch
4. 補齊缺失整合點
5. 修正變更後的 component / token / provider / route / style 接法
6. 保留 app 端既有 business logic
7. 防止 agent 因升版而重建整個 feature
8. 在 patch 後執行 Hard Gate 驗證

---

## 二、技能目標
將一個已初始化且已使用過 @cx-rd/ui-kit 模板或 contract 的 Angular 專案，同步到最新版 UI-kit 整合合約，並達成以下目標：
- 補上新版新增的必要 integration points
- 修正舊版 component API、provider、token、route、style entry 的不相容問題
- 保留使用者原有的商業邏輯與 feature 資料流
- 對既有專案執行最小侵入式升版修補
- 嚴格防止整頁重建、局部業務邏輯覆寫、與 contract 漂移

---

## 三、核心原則

### 1. Patch-First 原則
本技能必須優先執行 **Patch**，而非 **Regenerate**。
允許：
- 加 import
- 補 provider
- 修正 selector
- 補 route
- 補 style entry
- 修正 input/output
- 修正 wrapper 與 contract 接線

禁止：
- 直接刪掉既有 feature 後整頁重做
- 直接用新版模板整頁覆蓋既有頁面

---

### 2. Contract-Aware 原則
本技能的核心判斷依據不是畫面是否長得像，而是：
- 是否正確使用官方 UI-kit shell / wrapper
- 是否註冊必要 token / provider
- 是否符合 feature contract
- 是否存在 contract 漂移
- 是否仍保有必要 integration point

---

### 3. Preserve Business Logic 原則
本技能不得重寫或覆蓋以下內容，除非使用者明確要求：
- service 業務邏輯
- API 呼叫流程
- domain model
- feature data transformation
- section 中既有 business rules
- 由 app 端自行維護的資料綁定與流程控制

---

### 4. Fail-Loudly 原則
若技能無法安全推斷升級方式，必須：
- 明確標記為 manual review required
- 說明衝突原因
- 停止危險 patch
- 不可自由發揮或自行猜測 UI-kit contract

---

## 四、支援範圍

### 支援的升版範圍
本技能 v1 支援以下 feature：
- Main Layout
- Login
- Notifications
- Settings
- Navigation Bridge
- Shared Style Integration
- UI-kit 全域整合入口修正

### 暫不支援的範圍
以下內容不屬於本技能責任範圍：
- Backend API migration
- Database migration
- State management 重構
- Router 全量重設計
- Dashboard 全新生成
- 大量自訂 feature 模板化回收
- 大規模資料模型重構

---

## 五、技能模式
本技能支援兩種模式，必須先判定模式再執行。

### Mode A — Safe Sync
用途：
- 快速升版
- 風險最小化
- 只補必要 contract integration
- 不深入改動 custom structure

規則：
- 只做必要 patch
- 不重建 wrapper
- 不移除自訂 shell
- 發現高度客製結構時優先標記 manual review
- 嚴格禁止 full feature regeneration

### Mode B — Contract Repair
用途：
- 將已偏離 contract 的專案拉回標準整合路徑
- 修復既有的錯誤接線
- 修正之前 agent 未依技能規範生成的產物

規則：
- 允許修正 local substitute shell
- 允許補建必要 wrapper / sections
- 允許移除 deprecated integration pattern
- 仍不得覆蓋 business logic

---

## 六、適用前提
本技能只適用於以下情境：
1. 專案已存在且可讀取原始碼
2. 專案已經是 Angular 專案
3. 專案已安裝或曾使用 @cx-rd/ui-kit
4. 專案至少存在一種 UI-kit 使用痕跡，例如：
   - import from @cx-rd/ui-kit
   - 使用過 UI-kit selector
   - 使用過 UI-kit token
   - 使用過 UI-kit wrapper / shell
   - 使用過與 UI-kit 綁定的 route / config / styles

若專案完全沒有 UI-kit 使用痕跡，本技能不得假設該專案屬於 template-derived app，必須停止並回報不適用。

---

## 七、輸入契約
本技能應接收以下輸入，或從上下文中明確取得。

### 必要輸入
#### 1. Target Project Root
目標 Angular 專案根目錄。
#### 2. Upgrade Scope
可接受值：
- layout
- login
- notifications
- settings
- all
#### 3. Target UI-kit Version
目標升版版本，或等效的 contract release 說明。
#### 4. Patch Mode
可接受值：
- safe-sync
- contract-repair

### 選填輸入
#### 1. Breaking Change Notes
例如：
- NotificationPopover input renamed
- SETTINGS_SECTIONS is now required
- ui-kit.scss entrypoint replaces legacy style imports
- All Notifications route bridge is required
- local shell is no longer allowed for settings
#### 2. Customization Zones
指定哪些檔案或區域屬於 user-owned customization，應謹慎 patch。
#### 3. Known Template Usage
例如：
- project derived from login template
- notifications template previously applied
- settings wrapper already exists
- layout shell already exists

---

## 八、總體執行流程
本技能必須按照以下步驟執行，且不得跳過 Mechanical Audit 與 Hard Gate。

---

## Step 0 — Mechanical Discovery Audit

### 目的
在任何 patch 前，先機械式掃描專案，建立目前專案的 UI-kit 使用地圖。

### 必掃描檔案
- src/app/**/*.ts
- src/app/**/*.html
- src/app/**/*.scss
- src/styles.scss
- package.json
- angular.json
- src/app/app.routes.ts
- src/app/**/app.routes.ts
- src/app/**/navigation.config.ts
- 任何與 feature integration 有關的 routes / configs / wrappers / shells

### 必須偵測的資訊
1. 目前有哪些 @cx-rd/ui-kit imports
2. 目前有哪些 UI-kit selectors 被使用
3. 目前有哪些 UI-kit tokens 被註冊
4. 目前有哪些 feature routes 指向 UI-kit feature wrapper
5. 目前哪些 feature 已存在 template-derived wrapper
6. 目前是否有 triad 破壞
7. 目前是否存在 inline template: / styles:
8. 目前是否有 local substitute shell
9. 目前 styles 入口是否仍為 legacy import
10. 目前 navigation bridge 是否存在

### 本步驟輸出
必須先產出：
#### Detected Usage Record
```text
Detected UI-kit usage:
- MainLayoutComponent imported from @cx-rd/ui-kit
- NotificationPopoverComponent used in toolbar
- AllNotificationsPageComponent not found
- SETTINGS_TABS found
- SETTINGS_SECTIONS missing
- navigation.config.ts present
- ui-kit.scss not imported
- local settings shell detected
```
---

## Step 1 — Contract Delta Analysis

### 目的
比對目前專案與目標 UI-kit contract 的差異。

### 差異分類
每個差異必須分類為以下類型：
#### Missing Required Integration
缺少新版必要整合點。
#### Deprecated Usage
專案仍使用舊版 API 或 contract。
#### API Mismatch
component input、output、token 或 provider 不相容。
#### Structural Deviation
專案結構偏離官方模板 contract。
#### Manual Review Required
技能無法安全推斷自動修補方式。

### 本步驟輸出
#### Upgrade Delta Record
```text
Upgrade Deltas:
1. Missing SETTINGS_SECTIONS provider
2. NotificationPopover input outdated
3. ui-kit.scss entrypoint missing
4. All Notifications route bridge missing
```
