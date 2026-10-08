# 行動祕書 App — 專案說明（給 Claude Code 的交接筆記）

## 這是什麼
整合「工作＋個人」的行動祕書網頁 App（PWA），繁體中文介面，給個人使用（安規認證主任的日常業務＋個人生活），
部署在 GitHub Pages。純前端、無後端、無打包流程：所有資料存在使用者瀏覽器的 localStorage。
做法比照同一位使用者的「Healthy Care」（~/Foodie-control）。

## 檔案結構
```
index.html        整個 App：CSS、畫面、程式都在這一個檔案（vanilla JS，無框架、無 build）
manifest.json     PWA 設定（名稱、圖示、theme color #F5F1E8 生成色）
sw.js             Service Worker：同源 network-first、Google Fonts cache-first；改版時把 CACHE 名稱 +1
icon-180.png / icon-192.png / icon-512.png   App 圖示：AI 仿真雪納瑞「開心」照片裁成正方形（System.Drawing）
mascot/*.jpg      吉祥物貼紙照片（256px）：happy／worried／sleepy；原圖在 _private/mascot-src/
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
  **主資料是使用者每月審查的 Excel《TWINHEAD_安規認證到期管理_YYYYMMDD.xlsx》**（存在她的 Google Drive，每月一份新檔）。
  證書效期頁「從效期表更新」用 SheetJS（cdnjs xlsx 0.18.5，點按鈕才載入）讀「到期提醒總覽」工作表：
  標題列含「機型／認證‧憑證項目／到期日／備註」，到期日是公式（讀存檔時的計算值）。以「機型＋認證項目」比對（`certKey`），
  先顯示差異（新增／到期日變更／不變／表中沒有）再套用；新證書 id = `hashId(certKey)`（tw 開頭，跨裝置一致）。
  單向：不寫回 Excel（Excel 的到期日是公式，寫回會弄壞）。App 的續證狀態／證書編號／備註保留；到期日往後延時「續證中」自動變回「追蹤中」；
  表中沒有的不自動刪，讓使用者勾選「不續證／停用」。Excel 檔本身不要放進版控。
- `quotes`：每日一句（{id, text}）。內容是使用者自己提供的《這時候，深呼吸》段落，**有版權，絕對不要寫進程式碼或 commit**；
  只透過 App 載入：今日卡片／設定「載入 Word 檔」直接讀她的 `這時候深呼吸.docx`（JSZip，點按鈕才載入；每段以「-」開頭、{}→「」），
  或匯入 `_private/每日一句_這時候深呼吸.json`、或設定頁貼上。使用者要的是「App 每天自動貼一篇」，載入一次後就全自動，
  不要改成要她每天手動操作；也不要為了省掉載入步驟把內容寫進公開程式碼。
  `quoteOf(date)` 以日期為種子洗牌，每天 0:00 換、各裝置同一段、一輪不重複；當天第一次開 App 跳 `#dailyPop`，關掉後才接時段提醒。
  顯示格式（使用者指定）：第一行「與真如繼主同在，這時候，深呼吸」，下面是段落。
- 所有資料集合列在 `COLS`；新增集合時要一起加進 COLS、readRemote、snapshot、payload、backup、import。
- 範例資料帶 `sample: true`，首次開啟自動放入，可一鍵清除。
- 每筆資料都有 `updatedAt`；刪除記在 `deleted["<col>:<id>"] = 時間`（墓碑，保留 120 天）。
  **任何寫入都要經過 Store.set/patch/del 或自行補上 updatedAt／tomb()，否則雲端同步會被舊資料蓋回去。**

## 雲端同步（電腦・手機連動）
- 儲存在使用者 GitHub 帳號的**私密 Gist**（描述：`行動祕書 Mobile Secretary 同步資料（請勿刪除）`），
  檔案 `msec-data.json`（完整資料）與 `msec-calendar.ics`（給 iPhone 行事曆訂閱）。
- 金鑰：使用者自建的 classic token，只勾 `gist`；存在各裝置 localStorage `msec.sync.v1`，不進備份檔、不進程式碼。
- 合併：逐筆比 updatedAt，新的贏；墓碑時間比資料新就刪除。時機：存檔後 1.5 秒、開 App、切回 App、每 60 秒、恢復連線。
- 配對：`MSEC1.` + base64url({t: token, g: gistId})；QR Code 是 `<網址>#pair=<配對碼>`。
  iPhone 主畫面 App 與 Safari 的儲存空間是分開的，所以從 Safari 開的配對頁會提供「複製配對碼」給主畫面 App 貼上。
- 行事曆訂閱網址：`https://gist.githubusercontent.com/<login>/<gistId>/raw/msec-calendar.ics`（webcal://）。
  內容只放標題、時間、地點、屬性・類別，不放電話／對象／備註（使用者隱私偏好）。單向：iPhone 端修改不會回 App。

## 視覺風格（使用者指定）
日系手帳風、有質感、很多可愛小貼圖；**不要粉紅色系、不要粉色外框**。
- 配色：生成色方格紙底（--bg／--grid）、墨色文字、主色「藍」#3F5B72，輔色抹茶／山吹／朱／藤／空色；紙膠帶為低彩度點點和紙。
- 字體：標題 LXGW WenKai TC（手寫感）、內文 Noto Sans TC、數字 Quicksand、印章 Noto Serif TC。
- 小貼圖：`doodle(name)`／區塊標題 `ph(name)`（pencil、bell、plane、seal、hourglass、leaf、fuji、clip、onigiri、tea、paw、bone、sparkle、cloud、sun、moon、flower、memo）。
- 「完成」是朱色方形印章 `stamp()`；月曆週六藍、週日朱（日本慣例）。

## 吉祥物
使用者自家的白色雪納瑞。使用者用 Gemini 依她的照片產生的 AI 仿真照片（她不喜歡手繪卡通版）。
`mascot(mood, color, prop)` 回傳圓角照片貼紙：mood `happy|calm|worried|shock|sleepy` 對應 `PHOTO` 表
（目前 calm→happy、shock→worried，等使用者補「平靜」「驚訝」照片後新增 mascot/calm.jpg、shock.jpg 並改 PHOTO）；
color 是貼紙外框顏色（pink/mint/sun/coral/lilac/sky）；prop 是右下角 SVG 徽章（check/bell/cert/hourglass/sun/moon/coffee/star/doc/heart/phone，zzz 在右上）。
她原本的手機照片不放進版控。

## 提醒規則（使用者指定，不要改）
- **推播：只在前一天提醒一次**（2026/10/08 使用者要求，取消「開始前 2 小時」）：
  有時間的事在前一天同一時間；整日或沒填時間在前一天 09:00；重複事項每次各提醒一次。
  App 內由 `noticeTimes()`／`checkNotices()`（tick 每 20 秒，過時 6 小時內補發，`msec.ntf.v1` 記已發送）；
  行事曆（訂閱 feed 與一次性匯出）由 `alarmsFor()` 產生相同的 VALARM。改規則時把 `FEED_VER` +1，訂閱的行事曆才會立刻更新。
- 2026/10 起**取消**每日 07:00／12:00／17:00／21:00 時段提醒（使用者要求），不要再加回來。
- 今日頁「即將到來」仍列出 2 天內到期的事（只是清單，不是推播）。
- 證書：180 天內追蹤、90 天內與過期列入提醒；行事曆鬧鐘為到期前 90／30／1 天。

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
