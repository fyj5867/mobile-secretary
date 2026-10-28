# 行動祕書 — 獨立網頁 App（PWA）

整合「工作＋個人」的行動祕書：行程與待辦、認證申請（申請日→取證日色帶＋提交節點）、
證書效期、前一天推播提醒、每日一句、每日心情結語、週報與月報。

不需要登入任何帳號。資料存在使用者自己裝置的瀏覽器裡（localStorage）；開啟「雲端同步」或「Google 日曆連動」後，
才會同步到使用者自己的 GitHub 私密 Gist／Google 日曆。

## 資料夾內容

```
index.html     整個 App（畫面、樣式、程式都在這一個檔案）
manifest.json  PWA 設定：加入主畫面後的名稱、圖示、全螢幕
sw.js          離線快取（App 外殼可離線開啟）
icon.*         App 圖示
tools/serve.js 本機預覽用：node tools/serve.js . 8765
_private/      私人資料（證書匯入檔），不會上傳到 GitHub
```

不需要打包，改 `index.html` 存檔、推上 GitHub 就是新版。改版時請把 `sw.js` 的 `CACHE` 版本號加 1。

## 上架（GitHub Pages）

1. 把檔案推到 GitHub repository 的 `main` 分支。
2. 到 repository 的 **Settings → Pages**，Source 選「Deploy from a branch」，Branch 選 `main`、資料夾 `/(root)`，按 Save。
3. 約 1 分鐘後會出現網址 `https://<帳號>.github.io/<repository>/`。

> `_private/` 已經在 `.gitignore` 裡，證書資料不會被上傳。網址是公開的，資料一律用 App 的「匯入」放進自己的手機。

## 第一次使用

1. 手機用 Safari（iPhone）或 Chrome（Android）打開網址 →「加入主畫面」。
2. App 會先放一些標示【範例】的示範資料，看完可在「今日」或「設定」一鍵清除。
3. 把 `_private/行動祕書_匯入_證書效期.json` 傳到手機（例如存到 iCloud／OneDrive），
   在 App「設定 → 匯入備份／證書資料」選這個檔案 →「合併匯入」。
   這個檔由安規效期來源表產生，要更新就在 `OneDrive\安規報價試算器\` 跑：
   `.\update-cert-board.ps1 -AppJson "C:\Users\joyce_fan\Mobile-secretary\_private\行動祕書_匯入_證書效期.json"`

## 提醒怎麼運作

| 方式 | 何時有效 |
|---|---|
| App 通知＋提示音 | 在「設定」開啟系統通知後，App 開著或在背景時；剛錯過（6 小時內）會補發 |
| iPhone 行事曆推播 | 「雲端同步 → 訂閱到 iPhone 行事曆」（自動更新）或「匯出未來 180 天行程」，**App 關著也會準時推播** |

規則：事項、認證節點、預計取證日只在**前一天**推播一次（有時間的事在前一天同一時間；整日事項在前一天 09:00）。
證書 180 天內追蹤、90 天內列入提醒（行事曆在到期前 90／30／1 天提醒）。
重複事項（每天／每週／每月／每年）每次各自打勾。

## 雲端同步（電腦・手機連動）

「設定 → 雲端同步」開啟後，所有裝置共用同一份資料（存在自己 GitHub 帳號的私密 Gist）：

1. 電腦：按「建立 GitHub 金鑰」→ 只勾 `gist` → Generate token → 複製 `ghp_` 開頭的金鑰 → 貼上 → 連線。
2. 手機：電腦按「配對其他裝置」→ 用 iPhone 相機掃 QR Code →（Safari）「複製配對碼」→
   打開主畫面的行動祕書 App → 設定 → 雲端同步 → 貼上 → 連線。
3. iPhone 行事曆：設定 → 雲端同步 →「訂閱到 iPhone 行事曆」→ 訂閱（若有「移除提示」請關閉）。

## Google 日曆連動（雙向）

「設定 → Google 日曆連動 → 開始設定」照著做一次（請用電腦，約 5 分鐘，不用申請開發者帳號）：

1. 登入要連動的 Google 帳號 → 開 https://script.new → 專案改名「行動祕書連動」。
2. 刪掉原本的程式，貼上 App 給的程式碼（含專屬連動密碼）→ 儲存。
3. 左側「服務」＋ → Google Calendar API → 新增。
4. 部署 → 新增部署作業 → 網頁應用程式；執行身分「我」、誰可以存取「所有人」→ 部署 → 授權。
5. 複製網頁應用程式網址（…/exec）貼回 App → 連線。

之後：App 的事項、認證節點、證書到期會出現在 Google 日曆的「行動祕書」日曆；Google 主日曆過去 7 天～未來 90 天的行程會帶進 App。
兩邊修改標題、日期、時間、地點都會互相同步。其他裝置用「配對其他裝置」掃 QR Code 就會一起連動。

> iPhone 若已把 Google 帳號加進內建行事曆，就不需要再「訂閱到 iPhone 行事曆」，否則同一件事會出現兩次。

## 備份與換機

開了雲端同步就不用手動搬資料；仍建議每週「設定 → 匯出備份檔」存一份到雲端硬碟。
