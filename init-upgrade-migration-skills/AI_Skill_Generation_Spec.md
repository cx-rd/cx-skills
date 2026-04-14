# AI Skill 生成決策與行為規格書 (V5.1 - Hard Gate)

本文件詳述了針對 `@cx-rd/ui-kit` 與應用程式 (Application) 之間的 AI 生成決策邏輯。旨在建立一套「委派優先 (Delegation-First)」與「交付級硬阻斷 (Hard Gate)」的架構，確保系統一致性與模組化開發。

---

## 1. 核心設計原則 (Core Mantras)

1. **Orchestrator is contract-aware, not feature-implementing.**
2. **Specialized skills own their generation contract and verification logic.**
3. **Global verification validates both system-wide and delegated contracts.**
4. **Pre-build Contract Failure is the highest priority audit.** (合約稽核失敗即為終止性錯誤)

---

## 2. 專項技能的五大支柱 (Mandatory 5-Pillar Structure)

每個負責生成特定功能頁面 (Specialized Page Skill) 的技能**必須**完整定義以下五根支柱。

1. **Scope (範圍)**: 明確定義技能涵蓋的業務邊界。
2. **Generation Contract (生成合約)**: 規則、邏輯與禁止模式。
3. **Output Structure (產出結構)**: 明確的目錄架構與檔案命名規範。
4. **Mock Data Requirements (資料需求)**: 規定資料的正當性與密度。
5. **Verification Block (Fail-Fast 驗證塊)**:
   - **Check**: 包含檔案存在、結構組成、**Step 0 檔案拆分稽核**。
   - **Fail if**: 只要任一 Check 失敗，則立即宣告驗證失敗。

---

## 3. 運行時模型 (Runtime Model)

1. **Identify**: 識別 Features。
2. **Resolve**: 解析對應子技能。
3. **Delegate**: 調用子技能進行生成。
4. **Record**: 在 **Delegation Record** 中記錄細節。
5. **Step 0: Global Mechanical Audit**: 執行代碼級別的硬性稽核（見下節）。
6. **Recursive Verify**: 根據 Delegation Record 執行子技能 Verification Block。

---

## 4. 全域檔案拆分合約與硬阻斷條件 (Hard Gates)

針對專案源碼中的組件檔案 (`*.component.ts`)，AI 與驗證程序必須執行以下機械稽核，違反者觸發 **Pre-build Contract Failure**：

### A. 組件元數據誠信 (Component Metadata Integrity)
- **禁止 (Forbidden)**: 嚴禁在 `@Component` 裝飾器中使用 `template:` 或 `styles:`。
- **必須 (Mandatory)**: 必須使用 `templateUrl` 與 `styleUrl` (或 `styleUrls`)。
- **有效性 (Validity)**: 指向的 `.html` 與 `.scss` 檔案必須在實體路徑上確實存在。

### B. 檔案三件套規範 (File Triad Standard)
- 每個組件目錄必須完整包含 `.ts`, `.html`, `.scss` 三個檔案，除非該組件確實不需樣式（須有顯式理由，但 V5 預設強制三件套）。

---

## 5. 委派紀錄欄位 (Delegation Record Fields)
- `featureId`, `featureType`, `route`, `generatingSkill`, `outputPath`, `verificationTarget`。

---

## 6. 通用工程化原則 (Global Engineering Rules)
- **工作區隔離**: 嚴禁遞迴外部路徑。
- **合約先行**: 優先建立 `navigation.config.ts`。
- **庫審計優先**: 優先讀取 `.d.ts` 確認 API。

## 6.1 全域命名合約 (Global Naming Contract)

所有子技能若未宣告例外，必須遵循以下命名規則：

- component class 名稱必須由技能合約指定的 component 名稱轉為 PascalCase 後再加上 `Component`
- selector 必須使用 `app-{component-name-kebab}`
- page title / title 字串必須使用 Title Case

Title Case 規則：

- 第一個字母必須大寫
- 空格後的第一個字母必須大寫
- `-` 後的第一個字母必須大寫

若某個 specialized skill 需要使用不同的 title 欄位名稱或不同的 component 名稱，必須在該 skill 內明確宣告。

---
*文件更新日期：2026-03-10 (V5.1 - Hard Gate)*

---

# V5.2 增補：Ownership First Addendum

以下內容為 V5.1 的增補規格，目的是補強 AI 在 `@cx-rd/ui-kit` 與 application page 之間的 ownership 判斷能力。此增補不取代 V5.1，而是建立更清楚的 layout、shell 與 scroll 邊界。

---

## 7. Ownership Model

### A. Full-Page Template Ownership

若某個 `@cx-rd/ui-kit` component 已經提供完整頁面 chrome，則它視為 `full-page owner`。

這類元件通常已經內建其中多項：

- page header
- tabs
- filters
- sticky controls
- sidebar
- content panel
- section navigation
- page-level spacing 或 card shell

對這類元件，consumer 只能負責：

- route wiring
- state wiring
- data wiring
- DI / token configuration

consumer 不可再重複建立：

- header
- tabs
- filters
- 外層 page shell
- 額外 card shell
- 競爭性的 sticky 結構

說明：
本總規格不需要維護龐大元件白名單。只需要求各 specialized skill 明確標示：自己使用的 ui-kit component 是否屬於 full-page owner。

### B. App Shell Ownership

若頁面已經渲染在全域 shell 內，則該頁面不可再重建 shell。

全域 shell 通常擁有：

- toolbar
- sidebar
- main content slot
- app-level responsive behavior
- viewport / remaining-height sizing baseline

因此，standard in-shell page 不可重建：

- 第二個 shell
- 第二個 toolbar
- 第二個 sidebar
- 第二套與 shell 高度模型競爭的 `100vh` / fixed-height wrapper

### C. Standard Page Ownership

一般標準頁面只擁有自身內容區，例如：

- dashboard body
- customers table
- orders list
- analytics cards

這類頁面可自由定義：

- local header
- local cards
- local charts
- local table layout

前提是不可與上層 shell 或 full-page template 發生 ownership 衝突。

---

## 8. Scroll Ownership Model

scroll ownership 必須以「誰是主要 scroll container」為準，而不是誰先寫 CSS。

### 核心原則

如果 UI-kit component 已經內建主要 scroll container，consumer 不可再建立競爭性的外層 scroll 行為。

### 禁止行為

- 用外層 wrapper 改變 full-page template 的滾動模型
- 在 consumer 端用 `scrollIntoView()` 推動整個 app shell
- 為已經擁有 internal scroll 的元件再套一層 competing scroll container
- 額外加入 sticky header 或 tabs，導致原本 sticky 與 scroll container 失效

### 允許行為

- app shell 提供唯一的外層內容滾動區
- full-page template 在自己內部管理 section scroll
- standard page 自己管理 local content overflow，但不可破壞 shell 模型

### 修正原則

若 scroll 行為錯誤，應優先修 ownership 真正所屬的元件，而不是在 app 層加 workaround wrapper。

---

## 9. Ownership Hard Gates

以下任一條件成立時，應視為 `Pre-build Contract Failure`：

### A. Full-Page Ownership Violation

- full-page template 被 consumer 再包一層 page chrome
- consumer 重複建立該元件已擁有的 header、tabs、filters、card shell

### B. App Shell Ownership Violation

- in-shell page 重建 `MainLayoutComponent`
- route 已在 shell 內，又產生第二層 toolbar / sidebar

### C. Scroll Ownership Violation

- 點擊 section 導致外層 shell 滾動，而不是目標 content panel 滾動
- sticky 行為掛在錯誤 container
- wrapper 破壞既有元件的 scroll 模型

---

## 10. Verifier 增補責任

Verifier 不只驗證結構，還必須驗證 ownership。

### Step 1: Shell Audit

檢查：

- 是否重複建立 shell
- 是否在 shell 內又生成第二層 toolbar / sidebar

### Step 2: Full-Page Ownership Audit

檢查：

- 是否對 full-page template 重複建立 header / tabs / filters / card shell

### Step 3: Scroll Ownership Audit

檢查：

- 主要滾動容器是否合理
- scroll 行為是否落在正確層級
- sticky 區塊是否與 scroll container 一致

---

## 11. 本文件與 Specialized Skills 的分工

### 本文件應定義

- ownership 原則
- shell 原則
- scroll 原則
- hard gate 類型
- verifier 應驗證的抽象類型

### specialized skill 應定義

- 具體頁面使用哪個 ui-kit component
- 該 component 是否是 full-page owner
- 該頁必要的 route / DI / data wiring
- 該頁自己的 fail-fast 規則

換言之：

- 總規格定義「怎麼判斷」
- specialized skill 定義「實際套用在哪個頁面」

---

## 12. Practical Decision Rules

當 AI 不確定某個 UI-kit component 是否屬於 full-page owner 時，應依下列方式判斷：

1. 若該 component 同時帶有多個 page-level 結構特徵，例如 header、tabs、sidebar、sticky filter、page-level spacing，則預設視為 `full-page owner`。
2. 若某頁已經位於 route shell 之下，則預設它是 standard in-shell page，除非 contract 明確要求它自己擁有完整頁面結構。
3. 若某元件已定義 `overflow`、`sticky`、section navigation 或內部 scroll 區塊，則預設 scroll ownership 屬於該元件，不可由 consumer 任意覆蓋。

### Practical Override Rule

若 specialized skill 已明確宣告某個功能完整的 UI-kit 頁面元件在當前 application contract 中屬於 `In-Shell Content`，則應以 specialized skill 為準。

也就是說：

- 該元件仍可保有自己內部的 header、tabs、filters、section navigation 與 scroll ownership。
- 但它的 route 歸屬與 app-level shell ownership 仍屬於 `MainLayout`。
- verifier 應檢查的是是否重複建立 shell，而不是因元件本身結構完整就自動判成 shell 外 full-page route。

---

## 13. Success Criteria Addendum

除了 V5.1 原有要求外，還需額外滿足：

- ownership 沒有衝突
- scroll container 層級正確
- full-page template 沒有被重複包裝

---
*增補日期：2026-03-11 (V5.2 - Ownership First Addendum)*

---

# V5.3 增補：Legacy Migration & Orchestration Addendum

以下內容為 V5.3 的增補規格，旨在規範 AI 在面對高度完成、具有複雜商業邏輯的舊有專案 (Legacy Project) 進行 UI-kit 遷移時的決策流程與行為合規性。引入 `orchestrate-legacy-migration` 與 `migrate-legacy-to-uikit` 兩套核心技能，以確保遷移過程的安全可控與業務邏輯的完整保留。

---

## 14. 遷移決策模型 (Migration Decision Model)

針對現有功能的改造與遷移，必須遵循 **絞殺者模式 (Strangler Fig Pattern)** 與 **依賴圖優先 (Dependency Graph First)** 的原則，嚴禁用「一鍵重建全站」或「大面積覆寫」的方式改寫。

### A. 宏觀調度與邊界
1. **全新生成 (New Generation)**: 適用 V5.1 規範的 5-Pillar Structure，產生全新的合規頁面。
2. **遺留功能遷移 (Legacy Migration)**: 必須調用 `migrate-legacy-to-uikit` 執行。**嚴禁**刪除既有業務邏輯 (Business Logic)、API Contract 與 Domain Model。僅限於安全拆除 UI 外部佈局 (Shell-only Teardown) 與新 UI-kit wrapper 的組裝。
3. **宏觀遷移調度 (Macro Orchestration)**: 必須調用 `orchestrate-legacy-migration`。系統會將全站切割為具有拓樸關係的依賴圖 (DAG)，以 Feature 為單位精準且逐一遷移。**請注意，遷移節點不得只包含 route feature，必須同時涵蓋 shell surface 介面能力**。

### B. 範本分類與能力盤點 (Template Classification & Capability Matrix)
遷移任務執行前，必須完成能力盤點並將 UI-kit 範本進行分析歸類：
1. **Capability Matrix First**: 執行最終 cutover 之前，必須盤點 legacy shell 提供的能力 (如 login, toolbar, notifications) 並分配給未來的 owner。若有 `unmapped` (未分配) 能力，**絕對禁止進行最終退役 (Final Teardown)**。
2. **Scope Decomposition (範圍拆解)**: 面對寬語意能力或路由 (例如 settings, profile, notifications)，不得以名稱生硬對位，必須先拆分為具體的範圍 (例如 `user-scope`, `system-scope`, `domain-scope`, `shared-shell-scope`) 再進行對應 mapping。
3. **Template Classification (範本分類依賴)**:
   - **Class A (Pure Presentational)**: 不帶既定產品邏輯的純視覺外殼，預設必須採用不允許跳過。
   - **Class B (Interaction Skeleton)**: 帶標準互動骨架 (如 toolbar, sidebar)，必須建立 adapter layer 以保留真實互動的資料綁定。
   - **Class C (Opinionated Product)**: 帶內建產品語義，必須明確地在 `adopt` (採納), `adapt` (轉接), 與 `defer-with-record` (標記且推遲) 間做出決策，**禁止隱性遺漏**。

### C. 雙軌過渡路由 (Dual-Track Shell Transition)
在全站遷移計畫徹底完成之前，**不允許一刀切斷**舊有的全局佈局外殼 (`app.component.html` 內原本的 Legacy Shell)。
- 新舊路由應採齊行策略 (Hybrid Routing)。將已通過驗證 (Verified) 的完成路由逐步掛載至新 UI-kit 的 `MainLayoutComponent` 或路由層次之下。
- 只有當所有 DAG 節點驗證完成、無路由依賴舊款全域佈局，且最終包含 product smoke test 在內的稽核通過後，才能移除舊 shell。

---

## 15. 安全屏障與防呆合約 (Migration Hard Gates)

遷移操作時若觸發以下條件，即視為 **Terminal Failure (終止性失敗)**，系統應立刻中斷作業並匯報，嚴禁強行推進：

1. **Parity Gate (對等性阻斷)**: `verified` 不等於單純編譯成功 (compile success)。除取得 `ng build` 成功退場外，**必須同時滿足**對等性指標 (`routeWired`, `templateApplied`, `behaviorMapped`)，以及針對 shell-surface 或 full-page template 的最小產品可用性煙霧測試 (`smokeChecked`)。
2. **Business Space Violation**: 絕對禁止擅自更動 API interface、資料流向或本質邏輯。若套用佈局導致原始 Template 的事件偵聽或錯誤資料驗證狀態丟失，即破壞依賴合約。
3. **Batch Regex Override**: 觸及大規模元件改名或引用更改時，**嚴禁**使用廣域正則表達式 (Regex) 盲目批次改寫；應當使用 AST 解析技術或逐檔精細修正為標準。
4. **State Machine Inconsistency**: 宏觀遷移程序必須確保專案目錄內的 `migration-state.json` 時刻追蹤各節點的狀況。不可跳躍執行 (`pending` -> `verified`)；若有正當理由無法處理的節點應使用 `deferred` 狀態，且必須清楚註明 `deferredReason` 以防隱性遺漏。

---

## 16. 子技能職責與協作劃界 (Orchestration vs Specialized Skills)

遷移系統不允許「越俎代庖」，應清楚權責劃分：

- **Macro Orchestrator (`orchestrate-legacy-migration`)**: 負責通盤掃描、判讀技術棧，產出狀態機與拓樸依賴圖 (DAG) 作為全時調度總管；處理 Capability Matrix 建立、執行 Template Classification 決策，並把關最終 Parity Gate。**不親自修改單個 Feature 源碼或畫面設計**。
- **Refactor Skill (`migrate-legacy-to-uikit`)**: 重構的純「執行端」，針對單一節點獨立作業，在保證業務邏輯無損之下，執行舊殼拆解與新的 UI-kit 殼綁定。**不准干預全站路徑與其它節點決策**。

---
*增補日期：2026-04-02 (V5.3 - Legacy Migration Addendum - Capbility & Parity Gates Update)*
