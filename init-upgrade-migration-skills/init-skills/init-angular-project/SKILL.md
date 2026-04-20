---
name: init-angular-project
description: 在使用者已建立好的專案資料夾內，初始化 Angular 21 應用程式並整合以 build-package 形式安裝的 @cx-rd/ui-kit，建立可立即接受頁面生成任務的標準工作環境。
---

# Init Angular Project Skill

## 適用情境 (Precondition)

使用前提：
1. ✅ 專案資料夾已存在（例如 `C:\PJ\my-project\`）。
2. ✅ `.agent/skills/` 目錄已建立，且本 Skill 與其他所需 Skill 已放入。
3. ✅ `@cx-rd/ui-kit` 的來源（本地路徑或 npm 位址）已知。
4. ✅ 初始化完成後，後續 specialized page skill 會先讀取 `.agent/skills/AI_Skill_Generation_Spec.md` 再開始委派與生成。

核心硬規則：
- 只在目標專案路徑與其 `node_modules` 內操作。
- 禁止遞迴搜尋父目錄或其他平行專案資料夾。
- 初始化流程從本 Skill 第一步開始接管。
- 本 Skill 不負責建立 `.agent/` 目錄結構。

---

## 執行步驟

### 步驟 1：初始化 Angular 21 應用程式

在**當前所在的專案資料夾**內執行以下指令（若資料夾已有 `package.json`，跳過此步）：

```bash
npx -y @angular/cli@21 new . --style=scss --routing=true --standalone=true --ssr=false --skip-git
```

固定使用 `--style=scss` 與 `--standalone=true`。

---

### 步驟 2：偵測接入模式並安裝 @cx-rd/ui-kit

先判定接入模式，再安裝 `@cx-rd/ui-kit`。
**嚴禁**使用指向其他專案 `node_modules` 的 `file:` 路徑；任何 `file:` 都必須指向真實的 source/dist 目錄。

| 模式 | 判定條件 | 執行動作 |
| :--- | :--- | :--- |
| **Monorepo / Workspace** | 根目錄有 `tsconfig.json` path mapping 或 `projects/` 資料夾 | 1. 透過 `tsconfig.json` mappings 連結。<br>2. 審計並確保 `peerDependencies` 滿足。 |
| **External Local Package** | 提供之路徑位於當前專案外 | 1. **Peer Dependency Audit**: 審計 UI Kit 的 `peerDependencies`。<br>2. **Host Alignment**: Host 專案必須精準備齊所有 UI Kit 要求的 Peer Dependencies (例如 `@angular/cdk`, `rxjs` 等)。<br>3. `npm install @cx-rd/ui-kit@file:<path-to-dist>`。<br>4. **Single Instance Guard**: 確認沒有發生 Nested Angular 核心的狀況，且**僅能**在此模式下開啟 `preserveSymlinks` 以避免解析錯亂。 |
| **Published Package** | 無提供路徑，僅有名稱 | 1. `npm install @cx-rd/ui-kit`。 |

---

### 步驟 3：庫審計與條件式配置 angular.json

寫入 `angular.json` 前，先審計兩個路徑：
- 資產路徑：`node_modules/@cx-rd/ui-kit/src/assets`
- 樣式路徑：`node_modules/@cx-rd/ui-kit/lib/core/styles/index.scss`

**不得**預設其他 fallback 路徑存在；若找不到合法樣式入口，任務必須直接中止。

**根據審計結果配置 `angular.json`**：

```json
"assets": [
  { "glob": "**/*", "input": "public" },
  {
    "glob": "**/*",
    "input": "node_modules/@cx-rd/ui-kit/[AUDITED_ASSETS_PATH]",
    "output": "assets"
  }
]
```

配置規則：
- 僅有在 **External Local Package** 模式下，才需要在 `angular.json` 的 `options` 層級額外加入 `"preserveSymlinks": true`。
- 如果 `assets` 陣列已存在其他項目，**合併加入**，不要覆蓋掉 `public` 的設定。
- **嚴禁**參考其他專案的 `angular.json` 來解決路徑問題；應完全遵循本 Skill 的映射邏輯。

v21 build-package 的預期審計結果通常會是：
- `AUDITED_ASSETS_PATH = src/assets`
- `AUDITED_STYLE_MODULE_PATH = lib/core/styles`

---

### 步驟 4：排除常見建置錯誤 (Troubleshooting Knowledge)

如果在整合過程中遇到以下錯誤，**嚴禁**檢索外部參考專案，直接依照本表進行修復：

| 錯誤代碼 / 症狀 | 原因 | 修復方案 |
| :--- | :--- | :--- |
| `NG8002: Can't bind to 'X' since it isn't a known property` | 組件選用錯誤或未導入 | 1. 檢查 `imports` 是否包含該組件。<br>2. 檢查屬性名是否拼錯（例如應為 `route` 而非 `path`）。<br>3. 若使用 `lib-settings-page`，**禁用** `[tabs]` 綁定，改用 DI Provider。 |
| 樣式丟失 (Style missing) | `styles.scss` 未正確載入，或仍使用舊版 `@import` 入口 | 檢查 `src/styles.scss` 是否用 `@use '@cx-rd/ui-kit/[AUDITED_STYLE_MODULE_PATH]'` 載入正確模組。 |
| Sidebar / App Shell 高度沒有吃滿 | root height baseline 不完整，導致下游 `height: 100%` 無法成立 | 1. 檢查 `html`, `body`, `app-root` 是否都有明確的 `height: 100%`。<br>2. 檢查 `src/app/app.component.scss` 是否有 `:host { display: block; height: 100%; }`。<br>3. 禁止用 `100vh` wrapper 當成初始化階段的補丁。 |
| Toast 沒有顯示或跨 route 消失 | `lib-toast-viewport` 沒有掛在 app root，或被錯誤放在 shell / page 內 | 1. 檢查 `src/app/app.component.html` 是否存在且只存在一個 `<lib-toast-viewport>`。<br>2. 檢查 `src/app/app.component.ts` 是否有匯入 `ToastViewportComponent`。<br>3. 禁止在 `MainLayout`、page component 或 feature route host 內重複渲染 toast viewport。 |
| 找不到 `@cx-rd/ui-kit` | 套件未正確安裝，或本地 `file:` 路徑失效 | Published Package：先檢查套件是否已安裝且版本與 Angular 21 對齊。<br>External Local Package：再檢查 `package.json` 的 `file:` 路徑與必要時的 `preserveSymlinks: true`。 |

---

### 步驟 5：設置全域樣式（src/styles.scss）

確認專案使用 `src/styles.scss`（若仍是 `src/styles.css` 才需要更名），並寫入以下內容：

```scss
@use '@cx-rd/ui-kit/[AUDITED_STYLE_MODULE_PATH]';

/* 全域重置 */
*,
*::before,
*::after {
  box-sizing: border-box;
}

html,
body,
app-root {
  height: 100%;
}

body {
  margin: 0;
}

app-root {
  display: block;
  min-height: 100%;
}
```

> [!IMPORTANT]
> **嚴禁**使用舊版 `@import '@cx-rd/ui-kit/styles/ui-kit'` 之類已失效入口。
> 所有視覺設計**完全交給 `@cx-rd/ui-kit`**，不要手動覆蓋 `body` 背景、主題色或 page chrome。

---

### 步驟 6：更新 src/index.html

在 `<head>` 中加入 Material Icons（這是 UI Kit 圖示渲染的必要依賴）：

```html
<link href="https://fonts.googleapis.com/icon?family=Material+Icons" rel="stylesheet">
```

---

### 步驟 7：清理 AppComponent（建立 App Root Infrastructure Host）

同步建立或更新以下三個檔案：

1. `src/app/app.component.html`

```html
<router-outlet></router-outlet>
<lib-toast-viewport></lib-toast-viewport>
```

2. `src/app/app.component.ts`

```typescript
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';
import { ToastViewportComponent } from '@cx-rd/ui-kit';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet, ToastViewportComponent],
  templateUrl: './app.component.html',
  styleUrl: './app.component.scss'
})
export class AppComponent {}

```

3. `src/app/app.component.scss`

```scss
:host {
  display: block;
  height: 100%;
}
```

> [!CAUTION]
> **嚴禁**在 `app.component.html` 中加入任何 `<nav>`、`<header>`、`<footer>`、Landing Page 內容或自訂佈局。
> `AppComponent` 是 app root infrastructure host，只允許放置路由容器與全域基礎設施，例如 `<router-outlet>` 與唯一的 `<lib-toast-viewport>`。

---

### 步驟 8：設置 tsconfig.json 路徑映射（若使用本地庫）

若 `@cx-rd/ui-kit` 是以 `file:` 方式安裝的本地包，更新 `tsconfig.json` 加入路徑映射以便 IDE 支援：

```json
"paths": {
  "@cx-rd/ui-kit": ["./node_modules/@cx-rd/ui-kit"]
}
```

---

## 完成標準 (Definition of Done)

以下所有條件都必須滿足，才算初始化成功：

- [ ] `ng serve` 可以正常啟動（無編譯錯誤）。
- [ ] `app.component.html` 包含且只包含一個 `<router-outlet>` 與一個 `<lib-toast-viewport>`，且沒有任何 visual layout chrome。
- [ ] `src/app/app.component.ts` 使用 `templateUrl` 與 `styleUrl`，且 `imports` 包含 `RouterOutlet` 與 `ToastViewportComponent`。
- [ ] `src/app/app.component.scss` 存在，且 `:host` 具有 `display: block; height: 100%`。
- [ ] `angular.json` 包含 UI Kit 的 `assets` 映射。
- [ ] `src/styles.scss` 沒有自創顏色或主題，且已透過 `@use '@cx-rd/ui-kit/[AUDITED_STYLE_MODULE_PATH]'` 載入 v21 樣式入口。
- [ ] `html`, `body`, `app-root` 具有明確高度基準，避免 app shell / sidebar 高度失效。
- [ ] `src/index.html` 包含 Material Icons 連結。
- [ ] 專案已具備全域 toast 能力，且 `lib-toast-viewport` 的 owner 為 `AppComponent` 而不是 `MainLayout` 或任何 page component。

初始化完成後，可立即使用 `generate-unified-page` 或 `generate-app-orchestrator` Skill 開始生成業務頁面。

> [!IMPORTANT]
> 初始化完成後，後續所有頁面生成與委派都應以 `.agent/skills/AI_Skill_Generation_Spec.md` 為母規格，再交由 specialized skills 實作。
