# 行動祕書 App — 專案說明（給 Claude Code 的交接筆記）

## 這是什麼
整合「工作＋個人」的行動祕書網頁 App（PWA），繁體中文介面，給個人使用（安規認證主任的日常業務＋個人生活），
部署在 GitHub Pages。純前端、無後端、無打包流程：所有資料存在使用者瀏覽器的 localStorage。
做法比照同一位使用者的「Healthy Care」（~/Foodie-control）。

## 檔案結構
```
index.html        整個 App：CSS、畫面、程式都在這一個檔案（vanilla JS，無框架、無 build）
manifest.json     PWA 設定（名稱、圖示、theme color #E24A82）
sw.js             Service Worker：同源 network-first、Google Fonts cache-first；改版時把 CACHE 名稱 +1
icon.svg / icon-180.png / icon-192.png / icon-512.png   App 圖示（PNG 由 Windows System.Drawing 依 icon.svg 繪製）
tools/serve.js    本機預覽用的小型靜態伺服器：node tools/serve.js . 8765
.claude/launch.json  預覽設定（名稱 mobile-secretary，port 8765；本機絕對路徑，不進版控）
_private/         ★不進版控（.gitignore）。放使用者的證書清單匯入檔、舊版 claude.ai 網頁原始檔
```

## 資料模型（localStorage key: `msec.data.v1`，偏好設定：`msec.prefs.v1`）
- `items`：事項。scope `work|personal|shinnyo`（屬性與類別定義在 `SCOPES`／`CATS`）、cat、date、
  allDay（整日）、time（開始）、endTime（約略結束，選填）、title、target、company、location、phone、link、note、star。
  重複：repeat `none|daily|weekdays|weekly|monthly|yearly`、until；重複事項的完成記在 `doneDates[YYYY-MM-DD]`，
  一般事項用 `done`/`doneAt`。
- `apps`：認證申請。applyDate → certDate（月曆紙膠帶色帶，color 1–6）、status `active|obtained|paused|cancelled`、
  steps[{id,date,time,title,done}] 提交節點、actualCertDate、certNo、expiry。已取證且有 expiry 會自動建立 certs 的 `fa<appId>`。
- `certs`：證書效期。project、cert、due、no、state `normal|renewing|retired`、note。
- 範例資料帶 `sample: true`，首次開啟自動放入，可一鍵清除。

## 提醒規則（使用者指定，不要改）
- 事項／認證節點／預計取證日：到期前 **2 天** 開始提醒。
- 每日時段提醒：**07:00、12:00、17:00、21:00**（App 開著時跳提醒卡＋提示音＋系統通知；開 App 時 3 小時內補提醒）。
- 證書：180 天內追蹤、90 天內與過期列入提醒（2 天對續證太晚）。
- App 關著時的提醒靠 .ics 匯出交給手機行事曆（含 VALARM）。

## 隱私
GitHub Pages 是公開網址，**不要把任何真實業務資料寫進程式碼或 commit**；證書清單只用 App 的「匯入」放進使用者自己的裝置。

## 本機預覽
```bash
node tools/serve.js . 8765
```
然後開 http://localhost:8765 。

## 部署
推到 GitHub repository 的 `main`，Settings → Pages 設定 Branch `main` / `(root)`。
改版後記得把 `sw.js` 的 `CACHE` 版本號 +1，手機才會拿到新版。
