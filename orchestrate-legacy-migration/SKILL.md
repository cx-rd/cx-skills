---
name: orchestrate-legacy-migration
description: 一鍵全站遷移調度器。負責盤點整個高度成熟的 legacy Angular 專案，將全站建立為 DAG (依賴圖) 遷移模型，維護狀態機並依序調用 migrate 執行絞殺者遷移，最後透過雙軌過渡統一拔除舊全域外殼。
---

# orchestrate-legacy-migration

## 一、技能定位
本技能是**巨集遷移任務的全局調度器 (Macro Migration Orchestrator)**。
它專門設計用來實現「一句話把整個龐大老專案搬上 UI-kit 架構」的高階任務。本身不碰單一 feature 的細部重構，而是負責：
- 切分戰場
- 解析依賴圖 (DAG)
- 維護狀態機 (State Machine)
- 嚴格控管節點切換
- 監測斷點與執行回溯 (Rollback)

這是一套**自動化的絞殺者重構調度系統** (Automated Strangler Refactoring Orchestration System)。

## 二、核心原則
1. **Dependency Graph over Linear Queue**: 真實專案有交叉依賴，嚴禁無腦線性遷移。必須先構建以 Feature 為單位的 DAG (Migration Graph)，無依賴者先跑，有依賴者後跑。
2. **Micro-Batching over Bulk Simulation**: 嚴禁一次宣稱完成多個節點。每次只允許推進一個 Feature Node，完成後必須經過真實驗證才可進入下一個節點。
3. **Dual-Track Shell Transition**: 嚴禁一刀切替換 Root Shell (`app.component.html`)。必須採取雙軌過渡 (Hybrid Routing)，新舊 Layout 並存，直到所有舊路由都被絞殺完畢才移除舊殼。
4. **State Machine & Resumability**: 必須在專案根目錄維護 `migration-state.json`，支援中斷後接續 (Resume) 與 Debug。
5. **Checkpoint & Rollback**: 每個 Feature 遷移成功後必須建立 Checkpoint (e.g. Git commit)。一旦子技能回報任務失敗，允許一鍵 Rollback 回上一個完全穩定的 Checkpoint。
6. **Build Gate over Narrative Success**: 任何 `verified` 都必須建立在實際編譯成功之上，不允許以「看起來改完了」作為成功判定。
7. **AST over Regex**: 任何跨檔案 class rename、symbol rename、import rewrite，不得使用盲目 regex 批次替換，必須使用 AST、schematics、ts-morph 或逐檔精準修改。

---

## 三、禁止事項 (Non-Negotiable Guardrails)

Orchestrator 一律禁止以下行為：

1. **禁止模擬代跑**
   不得在尚未逐節點完成實作與驗證前，直接宣稱多個 Feature 已完成遷移。
2. **禁止只改 Router 不改 Feature**
   不得只更新 `app.routes.ts`、lazy route import、component 名稱引用，卻不核對對應 feature 檔案中的真實 export / class 宣告是否同步完成。
3. **禁止 Regex 暴力改名**
   不得用廣域 regex 直接替所有 `class`、`symbol`、`import`、`identifier` 加上 `Component` 或其他後綴。
4. **禁止跳過實體編譯**
   不得在未執行 `npm run build`、`ng build` 或專案指定 build 指令前，將任何節點狀態改為 `verified`。
5. **禁止假性 Checkpoint**
   若編譯未通過，不得建立 Checkpoint，不得更新 `lastStableCheckpoint`。
6. **禁止硬寫覆蓋式遷移**
   若子技能回報拆解風險或遷移不安全，必須立即中止，不能以大面積覆寫 TS/HTML 方式硬推完成。

---

## 四、狀態機合約 (Migration State Machine)

Orchestrator 在啟動後，必須在專案目錄或 `.agent/` 裡建立並隨時更新以下架構的 JSON/TypeScript 物件：

```typescript
interface MigrationState {
  features: {
    name: string;
    routePath: string;
    status: 'pending' | 'migrating' | 'verified' | 'failed';
    dependencies: string[];
    isSharedLogic: boolean;
    lastBuildCommand?: string;
    lastBuildPassed?: boolean;
    lastBuildAt?: string;
    lastErrorSummary?: string;
  }[];

  global: {
    shell: 'legacy' | 'hybrid' | 'uikit';
    currentPhase: 'A' | 'B' | 'C';
    angularLine: 'ng19' | 'ng21';
    uiKitSource: 'registry' | 'git' | 'file' | 'unknown';
    buildCommand: string;
    environmentReady: boolean;
  };

  lastStableCheckpoint: string;
}
```

狀態更新規則：
- 只有在單一節點完成遷移且 build exit code = 0 後，才允許把 `status` 從 `migrating` 改成 `verified`。
- 若 build 失敗，必須把該節點標記為 `failed`，寫入 `lastErrorSummary`，並中止整體流程。
- 任何時候都不得跳過 `migrating` 直接從 `pending` 變成 `verified`。

---

## 五、執行階段 (3-Phase Engineering Pipeline)

### Phase A: Global Discovery & Graph Building
**「繪製依賴拓樸圖，備妥狀態機」**

1. **依賴環境注射與版本線辨識 (Dependency Contract Detection)**
   Orchestrator 必須先讀取專案 `package.json`，辨識目前專案所屬的 Angular 主版本線，並解析 `@cx-rd/ui-kit` 的依賴來源型態。不得僅以「是否已安裝」作為判斷依據，必須區分 registry / git / file 等來源。

   - **判定規則**
     - **若專案 Angular major version = 19**
       `@cx-rd/ui-kit` 合法來源為 npm registry semver dependency（如 `^0.0.x`）。
       若尚未安裝，執行：
       ```bash
       npm install @cx-rd/ui-kit
       ```
       若 `package.json` 中不存在 `@angular/cdk` 相依，則補齊與 Angular major 相符的版本：
       ```bash
       npm install @angular/cdk@19
       ```
     - **若專案 Angular major version = 21**
       `@cx-rd/ui-kit` 不得自動安裝 npm registry 正式版，必須使用 Angular 21 專用來源，例如 git branch / git tag dependency。
       合法來源示例：
       ```json
       "@cx-rd/ui-kit": "git+https://github.com/cx-rd/cx-ui-kit.git#angular21-application"
       ```
       若專案缺少此依賴，Orchestrator 不得擅自安裝 Angular 19 主線套件，必須中止並回報：
       > 此專案屬於 Angular 21 支線，請先配置 Angular 21 專用 ui-kit 來源後再執行遷移。

       若 `package.json` 中不存在 `@angular/cdk` 相依，則補齊與 Angular major 相符的版本：
       ```bash
       npm install @angular/cdk@21
       ```
     - **若專案 Angular major version 不屬於目前支援範圍**
       Orchestrator 必須停止並明確回報不支援，不得自行猜測安裝策略。

   - **驗證要求**
     Orchestrator 必須驗證以下項目：
     1. `package.json` 是否存在 `@cx-rd/ui-kit`
     2. 該依賴來源是否符合目前 Angular 主版本線
     3. `@angular/cdk` 是否與 Angular major version 對齊
     4. 不得將 Angular 19 的 ui-kit contract 注入 Angular 21 專案
     5. 不得將 Angular 21 的 git dependency contract 注入 Angular 19 專案

2. **環境感知檢查 (Environment Awareness Checklist)**
   若 `@cx-rd/ui-kit` 來自 git 或 file source，必須額外檢查：
   1. `tsconfig.json` 或 `tsconfig.base.json` 的 `paths` 是否正確指向 library 的 `src/public-api`
   2. `MainLayoutComponent`、page wrapper 與相關 public symbols 是否可被專案解析
   3. style entry、builder 設定與 library source import 是否可共存
   4. build 指令是否已知，且可以在目前環境內被實際執行

   若上述任一項不成立，必須先回報環境未備妥，不得進入 Phase B。

3. **全站路由與依賴解析 (Topological Scan)**
   不但要找出 Component 與 Routes，更要掃描 Service 與 Shared Module 的交叉依賴 (Cross-Dependencies)。
   - **Shared Rule**: 如果某個 feature 帶有與其他 feature 共用的 business logic dependency，必須將它標記為 `Shared Migration Unit` 並延後 (Deferred)，直到所有相依的 feature 都被分析完畢。

4. **產出 Migration DAG (狀態機初始化)**
   - 根據無依賴 → 有依賴的順序建立 DAG。
   - 建立並寫入初始的 `MigrationState`。
   - 明確寫入 `global.buildCommand`。
   - 明確寫入 `global.environmentReady`。

### Phase B: DAG Execution & Tracking
**「依賴圖驅動的絞殺者流水線」**

進入嚴格單節點迴圈：

1. **挑選結點**
   從狀態機挑選 `status: 'pending'` 且 `dependencies` 皆已為 `verified` 的單一 Feature。
2. **狀態切換**
   將該 Feature 標記為 `migrating`。
3. **委派重構**
   必須調用 `migrate-legacy-to-uikit` 處理單一 Feature，不得一次派發多個節點。
4. **跨檔案一致性檢查**
   子技能完成後，必須逐一核對：
   - route import 是否存在
   - 對應 component class / export 是否存在
   - feature module / standalone component / provider 引用是否一致
   - 沒有遺留錯誤的 symbol rename
5. **強制實體編譯閘門**
   必須執行 `global.buildCommand`，例如：
   ```bash
   npm run build
   ```
   或：
   ```bash
   ng build
   ```
   只有在終端回傳 exit code = 0 時，才允許推進。
6. **Checkpoint & Rollback**
   - **Pass**: build 成功後，記錄 Git Commit (Checkpoint)，將狀態標記為 `verified`。
   - **Fail**: 若 build 失敗或一致性檢查失敗，必須：
     - 寫入 `lastErrorSummary`
     - 將此節點標記為 `failed`
     - 回復到 `lastStableCheckpoint`
     - 中止整體流程，等待人類介入
7. **人工確認節點節奏**
   若任務屬於高風險大型專案，完成單一節點後應停下並向使用者回報，再決定是否進入下一個 DAG node。

### Phase C: Dual-Track Shell Cutover
**「平滑過渡與無損拆除」**

1. **Hybrid Shell Injection (雙軌共存)**
   在最外層路由中，不要直接刪掉 legacy shell，而是引入 `MainLayoutComponent` 作為並行路由 (Parallel Route Anchor)。
2. **Gradual Routing Transition (漸進導流)**
   將狀態機中標記為 `verified` 的 Route 節點，逐步掛載到 `MainLayout` children 之下；未遷移完畢或 `failed` 的節點，繼續留在 Legacy Shell 的 `<router-outlet>`。
3. **Final Teardown (拔除舊鷹架)**
   只有在以下條件全部成立時，才允許拔除舊殼：
   - `features` 陣列全數為 `verified`
   - 無任何路由仍依賴舊有 Shell
   - 最後一次全站 build exit code = 0
   - 沒有待處理的 `failed` 節點

   之後才可：
   - 安全刪除 `app.component.html` 內殘留的舊有 Global Header、Sidebar、`100vh` 鷹架
   - 將狀態機切換為 `global.shell = 'uikit'`

---

## 六、執行策略補充

1. **每次只處理一個 Feature**
   不允許「Projects, Cases, Dispatch, Ticket 一次全改」這種展示式推進。
2. **優先使用精準程式結構工具**
   遇到跨檔案 rename、import rewrite、decorator 對應調整時，優先使用 AST、schematics、ts-morph，或逐檔人工修正。
3. **驗證比推進更重要**
   若無法在當前環境執行 build，就不能宣稱節點完成。
4. **一旦失敗，立即停機**
   失敗是中止點，不是重試點。未經人類確認，不得自動續跑下一個節點。
