---
description: 合約感知型調度者：負責分析需求、管理技能映射並記錄委派鏈的全局管理器。
---

# Generate App Orchestrator Skill (V5.2 - Ownership Conscious)

本 Skill 不參與具體功能開發，為 **Delegation-First** 架構的全局管理器。其核心任務是確保應用程式結構與 UI-kit 的 Ownership 分配完全對齊。

## 1. 核心守則
1. **Orchestrator is contract-aware, not feature-implementing.**
2. 僅管理 Routing、Layout Shell 與 Feature Delegation。
3. **Ownership Guard**: 必須明確區分「全頁模板 (Full-Page Template)」與「頁內內容 (In-Shell Content)」。
4. **Shell Height Guard**: `MainLayout` 必須作為唯一的 app shell 高度基準，sidebar 與內容區必須吃滿 shell 可用高度。
5. **UI-kit Adopt-First Guard**: 在委派任何 feature 前，必須先確認 `@cx-rd/ui-kit` 是否已提供可直接承接需求的 component / template。

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
4. **Setup Shell**: 建立 `MainContainer` 或 Auth Route 等全局佈局合約。
   - shell 必須提供一致的 viewport / remaining-height sizing baseline。
   - sidebar 不可只依賴子頁高度撐開。
   - 子頁不可用額外 `100vh` wrapper 破壞 shell 高度歸屬。
5. **Execute Delegation**: 調用子技能，並傳遞 Ownership 判定結果與 UI-kit adopt-first 前提。
6. **Log Delegation Record**:
   - `featureId`: {name}
   - `featureType`: {type}
   - `ownership`: {Full-Page | In-Shell}
   - `generatingSkill`: {skill}
   - `outputPath`: {path}
   - `verificationTarget`: {type} contract

## 4. 全域驗證委派
Orchestrator 在所有子技能生成結束後，**必須**調用 `verify-app-generation` 並執行：
- **Mechanical Audit** (三件套與 Metadata 檢查)
- **UI-kit Adoption Audit** (檢查是否忽略已存在的 UI-kit export)
- **Ownership Audit** (檢查有無重複包裝 Shell 或 Header)
- **Scroll Audit** (檢查滾動歸屬是否正確)
