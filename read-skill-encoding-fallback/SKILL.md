---
name: read-skill-encoding-fallback
description: 當讀取 skill 文件出現亂碼時使用。改用其他編碼重新讀取，優先確認 UTF-8，避免在亂碼狀態下誤判 skill contract。
---

# Read Skill Encoding Fallback

## 使用時機

- 讀取 `SKILL.md` 時出現亂碼、問號、破碎中文。
- 同一份 skill 在其他工具可正常顯示，但目前讀法異常。

## 規則

1. 不要直接根據亂碼內容做判斷或生成。
2. 先改用不同編碼重讀，優先檢查 `UTF8`。
3. 若 `UTF8` 正常，後續一律以 `UTF8` 內容為準。
4. 若仍異常，再嘗試系統預設編碼或其他常見編碼。
5. 只有在讀到可辨識內容後，才能繼續套用 skill contract。

## 成功條件

- 能清楚讀到 `name`、`description`、步驟與 fail 條件。
- 不再使用亂碼版本內容作為依據。
