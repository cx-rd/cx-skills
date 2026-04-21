---
name: generate-app-orchestrator
description: 當使用者要求在 Angular 應用中建立、重建或修改一個完整畫面、頁面或 route 時使用。即使需求只描述畫面包含按鈕、下拉選單、表單、dialog、table、pagination、panel 或 stepper 等 UI 控制，也必須先審計 `@cx-rd/ui-kit` 是否已有可直接承接的 component / template，再決定委派到對應 specialized skill 或 generic page flow。
---

# Generate App Orchestrator Skill (V5.2 - Ownership Conscious)

本 Skill 不參與具體功能開發，為 **Delegation-First** 架構的全局管理器。其核心任務是確保應用程式結構與 UI-kit 的 Ownership 分配完全對齊。

若使用者的描述很寬，例如「做一個畫面」、「重建一個頁面」、「做一個有按鈕和下拉選單的後台畫面」，也應先由本 Skill 判定是否屬於完整 screen / page generation，再進行 UI-kit adopt-first 審計與後續委派。

## 1. 核心守則
1. **Orchestrator is contract-aware, not feature-implementing.**
2. 僅管理 Routing、Layout Shell 與 Feature Delegation。
3. **Ownership Guard**: 必須明確區分「全頁模板 (Full-Page Template)」與「頁內內容 (In-Shell Content)」。
4. **Shell Height Guard**: `MainLayout` 必須作為唯一的 app shell 高度基準，sidebar 與內容區必須吃滿 shell 可用高度。
5. **UI-kit Adopt-First Guard**: 在委派任何 feature 前，必須先確認 `@cx-rd/ui-kit` 是否已提供可直接承接需求的 component / template。
6. **Route Contract Guard**: 若 UI-kit 已 export feature route constant 或 shell action contract，orchestrator 必須優先採用，禁止讓子技能自行猜測 path。
7. **Shell Surface Guard**: feature route 的 destination 與 shell 入口 placement 必須分開處理，不能把頁面存在等同於任意新增入口。

## 2. 特徵與技能映射表 (Skill Registry)

| Feature | FeatureType | Skill | Ownership Model |
| :--- | :--- | :--- | :--- |
| **Login** | login-page | `generate-login-page` | Full-Page Owner |
| **Settings** | settings-page | `generate-settings-page` | In-Shell Content |
| **Notifications** | notification-page | `generate-notification-page` | In-Shell Content |
| **Standard Page** | generic-page | `generate-unified-page` | In-Shell Content |

## 3. 調度流程 (V5.2 Standard)

1. **Identify**: 根據需求列出 Feature 清單。
2. **Audit UI-kit Capability**:
   - 先審計 `@cx-rd/ui-kit` 的公開 exports / `.d.ts`。
   - 先嘗試將需求映射到既有 UI-kit component / template。
   - 只有在沒有合適 export 時，才允許子技能走 custom composition。
3. **Classify (Ownership Check)**:
   - 判定每個 Feature 是否為 `Full-Page Owner`。
   - 判定是否需要渲染在 `MainLayout` (App Shell) 之內。
   - 判定該 feature 的 destination route contract 與 shell surface owner 是否已由 UI-kit 定義。
4. **Setup Shell**: 建立 `MainContainer` 或 Auth Route 等全局佈局合約。
   - shell 必須提供一致的 viewport / remaining-height sizing baseline。
   - sidebar 不可只依賴子頁高度撐開。
   - 子頁不可用額外 `100vh` wrapper 破壞 shell 高度歸屬。
   - 若存在 settings feature，且 app 使用 UI-kit `MainLayout` / user menu，預設應由 user popover 內的 settings action 導向 settings route，除非使用者明確指定其他入口配置。
   - 若存在 notifications feature，且 app 使用 UI-kit notification bell / popover，popover 內的 `View All Notifications` 必須導向 notifications route。
   - 若 UI-kit 已 export `SETTINGS_PAGE_ROUTE`、`ALL_NOTIFICATIONS_ROUTE` 或等效 contract，shell bridge 必須採用該 constant。
   - 若 UI-kit notification popover 尚無正式 `viewAll` output / extension point，必須先修補 UI-kit contract 或標記 blocked；禁止以 DOM hack 補橋。
5. **Execute Delegation**: 調用子技能，並傳遞 Ownership 判定結果與 UI-kit adopt-first 前提。
6. **Log Delegation Record**:
   - `featureId`: {name}
   - `featureType`: {type}
   - `route`: {route contract or resolved path}
   - `ownership`: {Full-Page | In-Shell}
   - `shellSurfaceOwner`: {user-popover | notification-popover | sidebar | none | custom}
   - `generatingSkill`: {skill}
   - `outputPath`: {path}
   - `verificationTarget`: {type} contract

## 4. Shell Surface Defaults

當 feature 與 shell surface 同時存在時，orchestrator 必須先決定誰擁有入口，再決定由哪個 skill 產生內容頁。

- `settings`:
  - 預設 destination 應對應 `SETTINGS_PAGE_ROUTE` 或等效 route contract。
  - 若 app 使用 UI-kit `MainLayout` / `UserMenu`，預設入口 owner 應為 user popover 內建的 settings action。
  - 未經使用者明確要求，不可因為生成了 settings page 就自動再加一個 sidebar settings entry。
- `notifications`:
  - 預設 destination 應對應 `ALL_NOTIFICATIONS_ROUTE` 或等效 route contract。
  - notification bell / popover 的 preview 由 UI-kit 擁有，但 `View All Notifications` 的 route bridge 由 shell host 擁有。
  - 不可在 shell 外再自建第二套 notifications popover 只為了接 route。

## 5. 全域驗證委派
Orchestrator 在所有子技能生成結束後，**必須**調用 `verify-app-generation` 並執行：
- **Mechanical Audit** (三件套與 Metadata 檢查)
- **UI-kit Adoption Audit** (檢查是否忽略已存在的 UI-kit export)
- **Ownership Audit** (檢查有無重複包裝 Shell 或 Header)
- **Shell Surface Audit** (檢查 settings / notifications 入口與 route bridge 是否正確)
- **Scroll Audit** (檢查滾動歸屬是否正確)
