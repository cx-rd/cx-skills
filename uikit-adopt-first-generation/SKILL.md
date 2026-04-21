---
name: uikit-adopt-first-generation
description: 當使用者在使用 `@cx-rd/ui-kit` 的專案中要求建立、重建或修改任何 UI 時使用。適用於整個畫面、頁面、route、局部區塊、表單、filter 區、toolbar 區，以及單一 control 或元件，例如按鈕、下拉選單、select list、dialog、datatable、pagination、panel、stepper。即使需求只說「做一個畫面」或只描述幾個 UI 控制，也必須先審計 UI-kit 公開 exports / `.d.ts` / 專案內 UI-kit 文件，優先 direct adopt 現有 component 或 primitive；只有在 UI-kit 沒有對應能力時才允許 custom UI。
---

# UI-kit Adopt-First Generation Skill

這個 skill 的目的不是直接生成某一種特定頁面，而是作為所有開發者主動發起之 UI 生成任務的前置守門員。

## 何時使用

只要需求屬於以下任一類型，就應先套用本 skill：

- 建立或重建整個頁面 / 畫面 / route
- 建立局部 UI 區塊，例如 filter bar、toolbar、form section、list section
- 建立單一 control 或互動元件，例如 button、下拉選單、select list、dialog、datatable、pagination、panel、stepper

## 核心規則

1. 先審計 `@cx-rd/ui-kit` 公開 exports、`.d.ts`、README 或專案內的 UI-kit generation guide。
2. 若 UI-kit 已有可語義承接需求的 component / primitive，必須優先 direct adopt。
3. 只允許建立 thin adapter layer 來橋接資料、事件、route 或 domain state。
4. 若 UI-kit 已 export 對應的 route constant、token 或 contract constant，必須優先使用，禁止自行猜測或硬寫近似 path 字串。
5. 不得把 custom UI 當成預設路徑。
6. 只有在 UI-kit 沒有對應能力，或現有 extension point 明確不足時，才允許建立 app-local UI。

## 執行流程

1. 先判斷需求是 page-level、section-level 還是 component-level。
2. 先做 UI-kit capability audit。
3. 若需求涉及 route、導航入口、popover action 或 shell action，audit 時必須一併檢查 UI-kit 是否已提供 route constant、output event 或既有 shell surface contract。
4. 若屬於完整畫面 / 頁面，繼續讀 `init-upgrade-migration-skills/AI_Skill_Generation_Spec.md`，並委派給對應 specialized skill 或 `generate-app-orchestrator`。
5. 若屬於局部區塊或單一元件，仍必須維持 `adopt-first`，不得繞過 UI-kit 直接手刻近似版。

## Fail If

- 未先審計 UI-kit，就直接開始生成 UI
- UI-kit 已有可直接承接的 component / primitive，卻仍自建近似 UI
- UI-kit 已 export route constant 或既有 contract，卻仍讓 agent 自行猜測 route path 或 action destination
- 以自寫 HTML / SCSS / wrapper 模仿 UI-kit 既有元件
- 把 custom implementation 視為與 UI-kit direct adopt 等價的預設選項
