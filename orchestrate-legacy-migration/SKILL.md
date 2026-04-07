---
name: orchestrate-legacy-migration
description: 一鍵全站遷移調度器。用於將成熟的 legacy Angular 專案遷移到 UI-kit 架構，建立 DAG 與狀態機，逐節點推進 route、shell surface 與 strict UI-kit parity，強制通過 build gate、capability mapping、UI conformance gate 與 product smoke gate 後才允許 cutover。
---

# orchestrate-legacy-migration

## 一、技能定位

本技能是巨集遷移任務的全局調度器。

它的工作不是直接完成單一畫面的細節重構，而是負責：

- 建立 route feature 與 shell surface 的遷移 DAG
- 維護可續跑的 migration state machine
- 逐節點委派遷移與驗證
- 在 final cutover 前確保 strict UI-kit parity，而不只是 build success

本技能的完成標準是：

- 結構完成遷移
- 共享 shell surface 已被明確建模
- UI-kit template 已依分類完成 direct adopt，或明確 blocked/deferred-with-record
- 最終 cutover 通過 build gate、UI conformance gate 與最小 product smoke gate

## 二、核心原則

1. **Dependency Graph over Linear Queue**
   真實專案必須建立 DAG，不得無腦線性遷移。
2. **Route + Shell Surface over Route Only**
   遷移節點不得只包含 route feature，必須包含 shell-level capabilities。
3. **Template Parity over Build Only**
   `verified` 不等於 compile success；必須同時滿足 template、behavior 與 capability mapping。
4. **UI-kit Conformance over Visual Approximation**
   「看起來像 UI-kit」不算完成；核心頁面必須直接服從 UI-kit 的 DOM 結構、視覺語言與互動契約。
5. **Dual-Track Shell Transition**
   Root shell 必須雙軌過渡，直到 legacy 依賴清空。
6. **State Machine & Resumability**
   必須維護 `migration-state.json`，支援 resume、rollback 與 deferred record。
7. **Capability Matrix First**
   shell cutover 前必須先盤點 legacy shell 提供的能力與未來 owner。
8. **AST over Regex**
   跨檔案 rename、import rewrite、symbol replacement 不得使用盲目 regex。
9. **Scope Decomposition before Mapping**
   遇到寬語意 capability 或 template，不得直接以名稱硬對位，必須先拆分 scope 再做 mapping。

## 三、禁止事項

1. **禁止模擬代跑**
   不得在未逐節點完成前宣稱整站完成。
2. **禁止只改 Router 不改 Capability**
   不得只改 `app.routes.ts`、children routes 或 shell wrapper，卻未處理對應 shell behavior。
3. **禁止以 build pass 取代 parity 驗收**
   build 通過只能證明可編譯，不能直接標記 `verified`。
4. **禁止隱性遺漏 template**
   若 UI-kit template 尚未落地，不得假裝完成；必須 direct adopt，否則明確 `deferred-with-record` 或 `failed`。
5. **禁止無 owner 的 shell cutover**
   若 capability matrix 中仍有 `unmapped` 能力，不得做 final teardown。
6. **禁止假性 checkpoint**
   若 build 或 parity gate 未通過，不得更新 `lastStableCheckpoint`。
7. **禁止以自製 UI 模仿 UI-kit 取代 direct adopt**
   若 UI-kit 已提供對應 component/template，不得以自寫 HTML/SCSS 近似實作後宣稱完成。
8. **禁止在未揭露偏離時標記 verified**
   若 DOM、layout、spacing、color、interaction contract 未直接遵守 UI-kit，最多只能標記 `migrating`、`failed` 或 `deferred`。

## 四、Template Classification Layer

所有 UI-kit template 在進入 Phase B 前，必須先分類，但分類不代表可以任意選擇策略。除非明確 blocked，預設目標一律是 direct adopt。

### Class A: Pure Presentational Template

特徵：

- 只提供 UI style、layout、slots、wrapper
- 不帶既定產品 workflow

處理規則：

- 預設必須套用
- 以 wrapper、projection 或既有 component logic 接入
- 不得因為既有邏輯存在就跳過 template migration
- 不得重寫主要 DOM 骨架來模仿 UI-kit

典型例：

- login style shell
- page wrapper
- simple dashboard wrapper

### Class B: Interaction Skeleton Template

特徵：

- 帶標準互動骨架
- 需要綁定 app-specific data source 或 action

處理規則：

- 必須優先直接使用 UI-kit component，僅允許建立 behavior adapter layer
- adapter 只能橋接資料與事件，不得重寫 UI-kit 的主要結構、樣式與互動骨架
- 驗收標準必須包含真實 input/output、資料綁定與 UI conformance

典型例：

- toolbar
- sidebar
- notification bell / popover
- user menu

### Class C: Opinionated Product Template

特徵：

- 帶內建產品語意
- 與既有產品概念不一定一一對應

處理規則：

必須先做 mapping decision，且預設順序必須是：

- `adopt`
- `defer-with-record`

只有在以下條件全部成立時才允許 `adapt`：

- UI-kit 沒有可直接承接的 template 或 extension point
- 已明確記錄缺口與無法 adopt 的證據
- 使用者已接受此節點只能暫時採 `adapt`
- 該節點不得標記 `verified`

典型例：

- settings-page
- all-notifications
- profile/security/notifications sections

## 五、Scope Decomposition Rule

對任何寬語意 capability、route、template 或 shell action，orchestrator 在做 mapping 前都必須先做 scope decomposition。

最少要拆成以下 scope：

- `user-scope`
  只影響單一使用者
- `system-scope`
  影響整個系統或所有使用者
- `domain-scope`
  影響特定模組、專案、workspace 或 tenant
- `shared-shell-scope`
  屬於全域殼層互動，不是單一路由業務頁

規則：

1. 若同名 capability 橫跨多個 scope，不得合併驗收。
2. template mapping 必須明確聲明它作用於哪個 scope。
3. 若 UI-kit template 的預設語意與現有 scope 不一致，必須先嘗試重新切分 owner 與 adopt；仍無法成立時才可 `defer-with-record`，不得默認 `adapt`。
4. 不得把 `user-scope` template 直接套到 `system-scope` 或 `domain-scope`。

這條規則適用於所有寬語意概念，不只 `settings`，也包括：

- notifications
- profile
- preferences
- security
- workspace
- billing
- admin
- dashboard

## 六、狀態機合約

Orchestrator 必須在專案根目錄維護 `migration-state.json`，至少支援以下欄位：

```typescript
interface MigrationNode {
  name: string;
  routePath: string | null;
  kind: 'route-feature' | 'shell-surface' | 'shared';
  status: 'pending' | 'migrating' | 'verified' | 'failed' | 'deferred';
  dependencies: string[];
  isSharedLogic: boolean;
  templateType?: 'presentational' | 'interaction' | 'opinionated' | 'none';
  strategy?: 'adopt' | 'adapt' | 'defer-with-record' | 'none';
  routeWired?: boolean;
  templateApplied?: boolean;
  behaviorMapped?: boolean;
  uiConformanceChecked?: boolean;
  uiConformanceStatus?: 'unknown' | 'direct-adopt' | 'adapted' | 'deferred';
  smokeChecked?: boolean;
  lastBuildCommand?: string;
  lastBuildPassed?: boolean;
  lastBuildAt?: string;
  lastErrorSummary?: string;
  deferredReason?: string;
}

interface CapabilityMatrixItem {
  capability: string;
  scope: 'user-scope' | 'system-scope' | 'domain-scope' | 'shared-shell-scope' | 'unknown';
  legacySource: string;
  futureOwner: string;
  migrationStrategy: string;
  doneCriteria: string;
  status: 'mapped' | 'unmapped' | 'deferred';
}

interface MigrationState {
  features: MigrationNode[];
  capabilities: CapabilityMatrixItem[];
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

- 不得從 `pending` 直接跳到 `verified`
- `verified` 至少要求：
  - `lastBuildPassed = true`
  - `routeWired = true` 或不適用
  - `templateApplied = true`
  - `behaviorMapped = true`
  - `uiConformanceChecked = true`
  - `uiConformanceStatus = direct-adopt`
  - `smokeChecked = true`，若此節點屬於 shell-surface 或 full-page template
- 若 `strategy = adapt`，該節點不得標記 `verified`
- `deferred` 只能在 `strategy = defer-with-record` 且 `deferredReason` 已寫明時使用

## 七、Phase A: Discovery And Graph Building

### 1. Dependency Contract Detection

先讀取 `package.json`，判定 Angular 主版本與 `@cx-rd/ui-kit` 來源。

- Angular 19：允許 registry semver dependency
- Angular 21：必須使用 Angular 21 專用來源，例如 git dependency
- 不支援版本：立即停止

必須驗證：

- `@cx-rd/ui-kit` 是否存在
- 來源是否符合 Angular 主版本線
- `@angular/cdk` 是否與 Angular major 對齊

### 2. Environment Awareness Checklist

若 UI-kit 來自 git 或 file source，必須額外檢查：

- `tsconfig` path 是否指向 `src/public-api`
- public symbols 是否可解析
- styles / assets / builder 設定是否可共存
- build command 是否可在目前環境實際執行

若未備妥，不得進入 Phase B。

### 3. Capability Matrix Discovery

必須先盤點 legacy shell 提供的能力，至少包含：

- login entry
- fullscreen routes
- sidebar navigation
- toolbar actions
- user menu
- settings entry
- notification bell
- notification center
- logout flow

對於寬語意 capability，必須先做 scope decomposition，再寫入 capability matrix。

例如：

- `settings`
  可能拆成 `user-scope`、`system-scope`
- `notifications`
  可能拆成 `user-scope`、`domain-scope`、`shared-shell-scope`
- `profile`
  可能拆成 `user-scope`、`system-scope`

每一項都必須有：

- `scope`
- `legacySource`
- `futureOwner`
- `migrationStrategy`
- `doneCriteria`

若 capability 無 owner，不得宣稱 discovery 完成。

### 4. DAG Construction

除了 route feature，也必須建立 shell-surface nodes。最少應考慮：

- `AuthEntry`
- `ShellNavigation`
- `ShellUserMenu`
- `ShellNotifications`
- `UserSettings`
- `FullscreenPages`

若專案中不存在某個 capability，必須顯式標記為不適用，而不是忽略。

## 八、Phase B: DAG Execution

### 1. 單節點推進

每次只挑選一個 `pending` 且 dependencies 皆已完成的節點。

### 2. 委派重構

對單一節點調用子技能或進行精準修改。

### 3. 一致性檢查

至少核對：

- route import 存在
- component / export 存在
- provider / standalone / module 引用一致
- shell action 與 destination 一致
- scope decomposition 已完成
- template classification 與 strategy 已落地
- UI 是否直接使用 UI-kit component/template，而非自製近似版
- DOM / class / layout / spacing / interaction contract 是否與 UI-kit 一致

### 4. Build Gate

必須執行真實 build command，exit code = 0 才可繼續。

### 5. Parity Gate

若節點屬於 shell-surface 或 full-page template，build pass 後還必須額外檢查：

- template 是否已套用
- 必要 behavior adapter 是否已存在
- capability matrix 的 done criteria 是否滿足
- UI conformance 是否達到 `direct-adopt`

### 5.1 UI Conformance Gate

以下任一項不成立，該節點不得 `verified`：

- 對應頁面直接使用 UI-kit component/template
- 沒有以自製 HTML/SCSS 重寫主要 UI 骨架
- 視覺階層、結構、間距、按鈕與主要互動遵守 UI-kit
- 若有 adapter，其責任僅限資料/事件橋接，不涉及重做視覺

若 UI-kit 本身無法滿足需求，必須：

- 記錄缺口
- 將節點標記為 `deferred` 或 `failed`
- 等待 UI-kit 補足或使用者明確接受偏離

### 6. Deferred Gate

若節點無法立即完成，只有在以下條件同時成立時才可 `deferred`：

- 已做出 `adopt/adapt/defer-with-record` 決策
- `deferredReason` 已清楚寫明
- 不影響目前 cutover 的完整性

### 7. Checkpoint & Rollback

- Pass：build 與 parity gate 皆成功後才可 checkpoint
- Fail：標記 `failed`，寫入 `lastErrorSummary`，回到 `lastStableCheckpoint`，停止流程

## 九、Phase C: Dual-Track Shell Cutover

### 1. Hybrid Shell Injection

Root shell 必須先採雙軌過渡，未完成節點留在 legacy shell，已完成節點逐步掛進 UI-kit shell。

### 2. Gradual Transition

只能導流已完成 parity 的節點。

### 3. Final Teardown

只有在以下條件全部成立時，才允許移除 legacy shell：

- 所有必要節點皆為 `verified` 或合法 `deferred`
- capability matrix 沒有 `unmapped`
- login、notifications、settings、logout、user menu 等 shell surface 已有明確 owner
- 最後一次 build exit code = 0
- 所有必要節點 `uiConformanceStatus = direct-adopt`
- 最小 smoke checklist 通過

之後才可：

- 安全刪除舊有 global header/sidebar/topbar/root layout
- 將 `global.shell` 切為 `uikit`

## 十、最小 Product Smoke Checklist

以下項目至少要在 final cutover 前驗證一次：

- `/login` 可進
- login submit flow 正常
- `returnUrl` 正常
- sidebar 可渲染且導航正常
- toolbar 可渲染
- user menu 可開啟
- settings action 有明確 destination
- notification bell 已接線，或已合法 deferred
- `/notifications` 可進，或已合法 deferred
- logout 正常

若任一項不成立，不得宣稱整站 migration 完成。

## 十一、執行策略補充

1. **Class A template 預設必須落地**
   若 UI-kit 只提供 style shell，應優先保留既有邏輯並替換視覺層。
2. **Class B template 只允許 behavior adapter**
   不要重寫 domain service；僅可將既有資料與事件橋接到 UI-kit，不可重寫 UI 骨架。
3. **Class C template 先做產品語意映射，再優先 adopt**
   未做 mapping decision 前，不得直接宣稱完成。
4. **對寬語意 capability 先做 scope decomposition**
   不要把名稱相同的功能混成同一節點；必須先拆成 user、system、domain、shared-shell 等 scope。
5. **無法完成時要顯式記錄**
   `deferred` 是顯式狀態，不是遺漏；若原因是 UI-kit 無法 direct adopt，必須明寫。
6. **驗證比推進重要**
   無法 build 或無法通過 parity gate，就停止，不得續跑下一節點。
7. **核心畫面不得以 adapt 冒充完成**
   login、notifications、settings、shell surfaces 若不是 direct adopt，不得宣稱 migration 完成。