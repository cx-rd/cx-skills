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
