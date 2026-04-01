---
name: migrate-legacy-to-uikit
description: 將尚未導入 UI-kit 的高複雜度 legacy feature，在保留既有 business logic 的前提下，分階段遷移至 UI-kit contract。此技能採用 strangler-style staged migration，不允許一步到位重建，不允許覆蓋 domain logic。
---

# migrate-legacy-to-uikit

## 一、技能定位
本技能是一個 **Legacy Refactor Skill**，專門用於處理：
- 已存在且高度完善的 Angular feature
- 擁有複雜 business logic、表單流程或 API flow
- 尚未導入 UI-kit contract
- 需要在不重建業務邏輯前提下，逐步且安全地遷移至 UI-kit shell / wrapper / page contract 的情境

本技能不是初始化器，不是頁面生成器，也不是升版同步器。

## 二、核心原則
1. **Logic First, UI Later**: *This skill must preserve the original feature behavior before preserving visual similarity.* (先保住功能，再談對齊 UI-kit，最後才是畫面整理)
2. **Shell-only Teardown**: 只拆除外部 layout 殼，嚴禁刪除與資料綁定、核心邏輯掛鉤的 template 區塊。
3. **No Business Logic Rewrite**: 嚴禁修改 API contract、欄位語意、domain model 與資料流方向。
4. **Stage Completion Before Next Stage**: 必須嚴格按照階段執行，前一階段未滿足 completion criteria 前，不得進入下一階段。
5. **Fail Loudly on Unsafe Refactor**: 無法確認拆解安全性時，必須中止並向使用者回報。
6. **AST over Regex**: 需要改名、補後綴、重寫 import 時，必須使用 AST 或逐檔精準修正，不得使用廣域 regex 批次替換。
7. **Build-Proven Completion**: 每一階段的完成，尤其是最終完成，必須建立在真實編譯成功之上。

---

## 三、禁止事項

1. **禁止整頁砍掉重練**
   不得把整個 feature TS/HTML 大面積刪除後，以新 UI-kit 頁面直接覆蓋。
2. **禁止直接改 business contract**
   不得擅改 API interface、DTO、service contract、domain model。
3. **禁止 Router-only 修補**
   不得只改 route 註冊與 import，卻不核對實際 component class / export 是否存在。
4. **禁止 Regex 暴衝式改名**
   不得用字串取代一次性替所有 `Service`、`Guard`、`DTO`、`Interface`、`Component` 類別做後綴改寫。
5. **禁止跳過 build 驗證**
   未完成真實 build 前，不得宣稱 feature 已完成遷移。

---

## 四、執行階段 (5 Phases)

### Phase 0 — Pre-Migration Audit
**「先盤點，不動刀」**

**目的**：找出舊 feature 的邏輯與 UI 耦合點，明確界定重構的紅線。

**必查內容**：
- `component.ts` 中的 API call、submit flow、computed state、form init。
- `template` 中的條件渲染、事件綁定、slot-like 區塊。
- `scss` 中的 layout 定位、z-index、height、overflow、header/sidebar 相關樣式。
- routing 與 navigation 接線。
- 是否存在自製 global shell、toolbar、modal、drawer 等全域違建。
- 是否存在跨檔案 rename 需求，例如 route import 與 component export 名稱不一致。

**本階段輸出標準**（必須吐出具體的結構化結果，才可推進）：
```text
Audit Result
- Business Logic:
  - loadProfile()
  - saveSettings()
  - buildForm()
- Layout Shell:
  - legacy page header
  - local sidebar
  - content wrapper
- High Risk:
  - template contains mixed layout + form logic
  - scss uses 100vh and fixed header offsets
  - submit button disabled state depends on derived flags
- Rename Risk:
  - route imports CasesComponent but feature exports CasesPage
```

### Phase 1 — Abstract & Isolate
**「先救邏輯，不改視覺骨架」**

**目的**：將 `component.ts` 裡的 business logic 拆出來，優先抽成 facade / feature store / presenter service，讓 component 只保留最薄的 UI adapter。

**絕對允許**：
- 抽取 observable / signal。
- 抽取 load / save / validate / submit。
- 抽取 form build 與 state mapping。

**絕對禁止**：
- 改 API contract。
- 改欄位語意與 Domain model。
- 改變原有的資料流方向。

**本階段完成標準 (Hard Gate)**：
- 抽離後專案可正常編譯。
- 原有頁面仍以舊邏輯可完美運作（視覺骨架尚未拆除）。
- business logic 被純化，可隨時被新的 UI 乾淨重用。

### Phase 2 — UI Teardown & Scaffold
**「拆殼，但只拆殼」**

**目的**：移除與 UI-kit shell 重疊的外層布局結構，保留所有與資料綁定、互動事件、驗證顯示有關的核心內容區塊。

**可拆的 (Layout Shell)**：
- legacy header / legacy sidebar
- local page wrapper
- 定位與撐開高度的 layout positioning scss
- 與 UI-kit shell 在結構上直接重疊的區塊

**不可拆的 (Business View Core)**：
- 與表單欄位綁定直接相關的 template 區塊。
- 與錯誤提示、驗證狀態、條件渲染 (`@if` / `@for`) 直接綁定的節點。
- 與資料呈現邏輯強耦合的清單或重複區塊。

**Scaffold (重建基石)**：
在此階段引入 UI-kit page wrapper、準備對應的 section container、補齊 required provider / route / style entry。

### Phase 3 — Wire-up & Audit
**「把舊腦接到新殼」**

在組裝新元件時，必須進行三種深度的綁定：

1. **State Wire-up**
   將 Facade 暴露的 signals / observables 接到 UI-kit 的 inputs 上；將 loading / empty / error state 精準對應到 UI-kit 的專屬 slot。
2. **Event Wire-up**
   將 save / cancel / retry / tab switch / modal open 等 UI 事件重新對接到 facade。
3. **Contract Wire-up**
   正確註冊 route、provider (e.g. `SETTINGS_SECTIONS`)、navigation bridge 與 style。
4. **Cross-file Symbol Audit**
   逐一核對 route import、component export、standalone import、provider reference 是否一致。

**階段驗證重點**：
- 是否依然保有舊邏輯的行為模式？
- 是否符合 UI-kit 合約規範？
- 是否還殘留 legacy shell，或者出現了 layout 樣式衝突？
- 是否存在 import/export 脫鉤？

### Phase 4 — Hard Gate Verification
**「高危險操作的全域防護網」**

為確保這支風險極高的技能順利落地，完成前必須由系統自檢以下項目：
- [ ] Business logic 是否完整存在且未被優化掉？
- [ ] Facade / Store 是否真的有被 UI 層調用綁定？
- [ ] Template 原有的事件與資料綁定是否有任何無意間的丟失？
- [ ] Legacy shell 是否已經確實移除？
- [ ] UI-kit 所需的 wrapper / provider / route / style 是否配置完整？
- [ ] route import 與實際 component export 是否完全對齊？
- [ ] **絕對沒有**「整頁砍掉重練」的作弊痕跡，無大規模覆蓋原本的 TS/HTML。

### Phase 5 — Build Gate
**「沒有編譯成功，就不算完成」**

完成前必須執行專案指定 build 指令，例如：

```bash
npm run build
```

或：

```bash
ng build
```

只有在 exit code = 0 時，才允許將此 feature 回報給 Orchestrator 作為可標記 `verified` 的節點。

若 build 失敗，必須：
- 停止回報成功
- 輸出錯誤摘要
- 明確指出是 symbol rename、route wiring、provider、template binding，或 style 衝突所造成
