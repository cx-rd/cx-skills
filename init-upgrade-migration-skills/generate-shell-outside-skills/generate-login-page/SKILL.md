---
name: generate-login-page
description: 產生或更新使用 `@cx-rd/ui-kit` `LoginPageComponent` 的登入頁。用於建立 `src/app/login/` 路由頁面，並處理登入後導頁或驗證流程；此元件是完整頁面模板，不可再外包 hero、標題區、額外卡片或第二層全頁殼。
---

# Generate Login Page Skill

用於建立或更新應用程式登入頁。

## 核心規則

`LoginPageComponent` 是完整頁面模板。

- 不可在 `<lib-login>` 外再包一層 page header、hero、marketing 文案區或外層卡片。
- 不可再建立第二層全螢幕布局來包住 UI-kit 登入頁。
- consumer 端只負責路由與登入成功後的應用邏輯。

## 必要輸出

建立或更新：

- `src/app/login/login.component.ts`
- `src/app/login/login.component.html`
- `src/app/login/login.component.scss`

必須使用檔案分離。禁止 inline `template` 或 `styles`。

## 實作契約

1. 從 `@cx-rd/ui-kit` 匯入 `LoginPageComponent`。
2. 在 template 中直接渲染 `<lib-login>`。
3. 讓 route host 保持適合 full-page login 的高度。
4. 需要時在 consumer component 處理登入成功後的導頁或 auth 狀態。
5. 在 `app.routes.ts` 中註冊登入路由。

## Fail If

- `<lib-login>` 被另一層 page chrome 或 full-page card 包住。
- consumer 重複建立 UI-kit 已擁有的標題、副標、CTA 或裝飾性整頁布局。
- 使用 inline metadata。

## 參考模板

```html
<lib-login></lib-login>
```
