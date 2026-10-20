# 行動祕書 App — 專案說明（給 Claude Code 的交接筆記）

## 這是什麼
整合「工作＋個人」的行動祕書網頁 App（PWA），繁體中文介面，給個人使用（安規認證主任的日常業務＋個人生活），
部署在 GitHub Pages。純前端、無後端、無打包流程：所有資料存在使用者瀏覽器的 localStorage。
做法比照同一位使用者的「Healthy Care」（~/Foodie-control）。

## 檔案結構
```
index.html        整個 App：CSS、畫面、程式都在這一個檔案（vanilla JS，無框架、無 build）
manifest.json     PWA 設定（名稱、圖示、theme color #F4F2EE 素色紙）
sw.js             Service Worker：同源 network-first、Google Fonts cache-first；改版時把 CACHE 名稱 +1
icon-180.png / icon-192.png / icon-512.png   App 圖示：AI 仿真雪納瑞「開心」照片裁成正方形（System.Drawing）
mascot/*.jpg      吉祥物貼紙照片（256px）：happy／worried／sleepy；原圖在 _private/mascot-src/
tools/serve.js    本機預覽用的小型靜態伺服器：node tools/serve.js . 8765
.claude/launch.json  預覽設定（名稱 mobile-secretary，port 8765；本機絕對路徑，不進版控）
_private/         ★不進版控（.gitignore）。放使用者的證書清單匯入檔、舊版 claude.ai 網頁原始檔
```

## 資料模型（localStorage key: `msec.data.v1`，偏好設定：`msec.prefs.v1`）
- `items`：事項。scope `work|personal|shinnyo`（屬性與類別定義在 `SCOPES`／`CATS`）、cat、date、
  allDay（整日）、time（開始）、endTime（約略結束，選填）、title、target、company、location、phone、link、note、
  pri（優先：`''|important|urgent`＝一般／重要／急件，2026/10/14 新增；舊資料的 `star: true` 視為重要，`priOf()`；存檔時 star 跟著 pri 寫）。
  急件／重要：事項前面有標籤、今日待辦與逾期排最前面、問候列顯示急件數、行程頁可「只看急件／重要」、推播與報表文字加【急件】。
  重複：repeat `none|daily|weekdays|weekly|monthly|yearly`、until；重複事項的完成記在 `doneDates[YYYY-MM-DD]`，
  一般事項用 `done`/`doneAt`。從 Google 主日曆帶進來的事項另有 `gsrc: 'primary'`、`gev`（Google 事件 id），見下方「Google 日曆連動」。
- `apps`：認證申請。applyDate → certDate（月曆紙膠帶色帶，color 1–6）、status `active|obtained|paused|cancelled`、
  steps[{id,date,time,title,done}] 提交節點、actualCertDate、certNo、expiry。已取證且有 expiry 會自動建立 certs 的 `fa<appId>`。
  `bill`（2026/10/15 新增，報價與請款）：{ quote 收到報價單, sign 回簽報價單, invoice 收到發票, claim 發票請款：勾選的日期或 ''；amount 報價金額 }，
  步驟定義在 `BILL`；卡片上可直接勾（記今天，再按取消），表單「報價與請款」可改日期與金額。
  認證頁第三個分頁「報表」（`vCertReport`）：依配合單位（`company`）統計案件、節點進度、報價合計、帳務完成；案件進度表；
  可「複製報表文字」與匯出 CSV（含報價與請款各步驟日期）。
- `certs`：證書效期。project、cert、due、no、state `normal|renewing|retired`、note。
  **主資料是唯一來源 Excel `Z:\交接區\Joyce 記錄區\TWINHEAD_Safety_application_and_standard_revision_Joyce.xlsx`**
  的「系統效期管理」「Battery 效期管理」「Adapter效期管理」三個分頁（2026-10-15 使用者指定；取代先前每月一份的
  《TWINHEAD_安規認證到期管理_YYYYMMDD.xlsx》）。
  進料方式：在 `C:\Users\joyce_fan\OneDrive\安規報價試算器\` 跑
  `.\update-cert-board.ps1 -AppJson "C:\Users\joyce_fan\Mobile-secretary\_private\行動祕書_匯入_證書效期.json"`，
  再到 App「設定 → 匯入備份／證書資料」選該檔「合併匯入」。同一支腳本也更新安規認證效期看板，兩邊數字一定一致。
  匯入檔的 id 用 App 自己的 `hashId(certKey)`（tw 開頭、跨裝置一致），所以重複匯入是更新同一筆、不會長出重複證書。
  note 會寫入各認證的提前通知月數（例「提前 3 個月通知（自 2025/03/18 起）｜65W｜適用 S14、Z14」）；
  `state` 一律 `normal`，180／90 天的追蹤與提醒門檻仍由 App 自己算。
  ⚠ 合併匯入是整筆取代同 id 的證書，**App 裡手動填的續證狀態／證書編號／備註會被蓋掉**；
  ⚠ 新表沒有的舊證書不會被刪，要在 App 裡勾「不續證／停用」。
  證書效期頁的「從效期表更新」（SheetJS 讀「到期提醒總覽」工作表）**自 2026-10-15 起停用不再使用** ——
  那條路讀的是另一份月檔，會變成第二份真相；程式碼保留但不要再叫使用者用它，也不要據此修改資料流。
  單向：不寫回 Excel。Excel 檔本身不要放進版控。
- `quotes`：每日一句（{id, text}）。內容是使用者自己提供的《這時候，深呼吸》段落，**有版權，絕對不要寫進程式碼或 commit**；
  只透過 App 載入：今日卡片／設定「載入 Word 檔」直接讀她的 `這時候深呼吸.docx`（JSZip，點按鈕才載入；每段以「-」開頭、{}→「」），
  或匯入 `_private/每日一句_這時候深呼吸.json`、或設定頁貼上。使用者要的是「App 每天自動貼一篇」，載入一次後就全自動，
  不要改成要她每天手動操作；也不要為了省掉載入步驟把內容寫進公開程式碼。
  `quoteOf(date)` 以日期為種子洗牌，每天 0:00 換、各裝置同一段、一輪不重複；當天第一次開 App 跳 `#dailyPop`，關掉後才接時段提醒。
  顯示格式（使用者指定）：第一行「與真如繼主同在，這時候，深呼吸」，下面是段落。
- `journal`：每日心情結語（2026/10/13 新增）。{id: `j`+日期, date, text}，一天一則、100 字內（`MOOD_MAX`）。
  今日頁最下方輸入（停筆 1.2 秒或離開欄位就存；只存資料不重畫，草稿放 `UI.moodDraft`，避免中文輸入被打斷）；
  月曆點選今天以前的日子可以補寫；報表頁「心情回顧」列出期間內的結語（屬性選「工作」時不顯示）。
  私人內容：不進 iPhone 訂閱、不進 Google 日曆、不進「複製報表文字」與 CSV。
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

## 資安（2026/10/20 新增，使用者要求）
- **App 鎖**（每台裝置各自設定，`msec.lock.v1`，不同步、不進備份）：PIN（4～8 碼）或密碼，只存 PBKDF2-SHA256（15 萬次）雜湊與 salt。
  上鎖時 `body.app-locked` 只顯示 `#lockScreen`；自動上鎖：離開 App 超過 `delay` 分鐘（0＝每次進入都要解鎖，**預設**，使用者要求「一進入就要密碼」）、
  或畫面開著沒操作超過 max(delay,5) 分鐘。可用 WebAuthn 平台驗證（Face ID／指紋／Windows Hello，`lockCfg.wa` 存 credential id）解鎖。
  錯 5 次開始冷卻。「忘記密碼」＝清除這台的資料與連線設定（不能只清鎖，否則有同步的人一清就能把資料抓回來），再重新配對取回。
  頁首鎖頭按鈕可立即上鎖。這是防「拿到沒鎖的裝置」的鎖，不是磁碟加密。
- **雲端同步加密**：secret Gist 不是私密的（知道網址就能讀），所以 `msec-data.json` 可加密成
  `{ enc:1, kdf:{alg,iter,salt}, iv, ct }`（AES-GCM，金鑰＝PBKDF2(同步密語, salt, 25 萬次)）。金鑰存在各裝置 `syncCfg.ek/esalt`，不存密語原文。
  沒有金鑰或密語錯 → `sync.err='needpass'`，**絕不上傳**，設定頁出現輸入框；配對碼的 `e` 欄位會帶金鑰，掃 QR 的新裝置不用再輸入。
  換密語：舊金鑰暫存 `prevKeys`，讀舊檔、寫新檔後清掉；其他裝置會被要求輸入新密語。雲端檔的加密狀態與本機不一致（例如舊版裝置寫回明文）就重新上傳加密版。
  沒有「取消加密」（避免各裝置互相改回）。`msec-calendar.ics` 為了 iPhone 訂閱維持明文（只有標題、時間、地點）。

## Google 日曆連動（雙向，2026/10 新增；使用者的個人 Gmail）
- **不用 Google Cloud 專案**：使用者把 index.html 裡的 `GAS_CODE` 貼到自己帳號的 Apps Script（script.new），在「服務」加入
  Google Calendar API（進階服務），部署成網頁應用程式（執行身分：我／誰可以存取：所有人）。App 用 POST（text/plain，免 CORS 預檢）送
  `{key, tz, ops}`，ops：hello／list／get／upsert／patch／remove。
- `KEY` 是 App 產生的隨機密碼，只寫在她自己的 Apps Script 裡；`msec.gcal.v1`（url、key、me、scope、lastSync）只存在裝置，
  不進 Gist、不進備份、不進程式碼。配對碼 MSEC1 的 `s` 欄位會帶上 Google 連動；單獨的連動碼是 `MSECG1.` + base64url({u, k})。
- **改 GAS_CODE 時要把裡面的 VER +1，並請使用者在 Apps Script 貼上新版 →「部署 → 管理部署作業 → 編輯 → 版本：新版本」**（網址不變）。
- App → Google：寫到專用日曆「行動祕書」（第一次自動建立，id 記在 Script Properties）。事件 id 由資料 id 推得：
  `ms`+base32hex(事項 id)、`st`(認證 id/節點 id)、`cd`(認證 id，取證日)、`ce`(證書 id)；多台裝置重複寫入也只有一筆
  （insert 遇到 already exists 就改成更新並恢復）。事項雙向；節點、取證日、證書單向（以 App 為準）。只新寫 30 天前之後的資料。
  描述只放「屬性・類別」，不放對象、電話、備註（和 iPhone 訂閱同樣的隱私原則）。顏色：工作 9、個人 2、Shinnyo 3、認證 5、證書 11。
  提醒同 App 規則（前一天）；證書為 28 天前＋前一天 09:00（Google 最多只能提前 4 週）。
- Google → App：主日曆「過去 7 天～未來 90 天」（singleEvents 展開，重複行程每次一筆）帶進來成為事項：id `g_<事件 id>`（太長用 `gh_<hash>`），
  新帶進來的 updatedAt 用 Google 的 updated（別台已補的屬性／備註合併時不會被蓋掉）。婉拒的、workingLocation 不帶；說明只在第一次帶進備註。
  在 Google「行動祕書」日曆直接新增的行程（沒有邀請對象、重複規則簡單）會收編成一般事項（`ga<hash>`）。
  gsrc 事項不放進 iPhone 訂閱與 .ics 匯出（Google 帳號加到 iPhone 就看得到，避免重複），日期過了自動視為完成（不算逾期）。
- 比對欄位：標題、日期、整日、開始／結束時間（剛好一小時視同沒填）、地點；行動祕書日曆另比重複規則（Google 上改成 App 表達不了的規則就不比）。
  兩邊不同時以較晚修改的為準（Google `updated` vs App `updatedAt`）。別人邀請的會議改不了（403）→ 還原成 Google 的內容並提示。
- **只有明確的刪除才刪**：Google 狀態 cancelled → 刪 App 事項；App 墓碑 → 刪 Google 事件。這台不認得的事件一律不動（可能是別台較新的資料）。
  一次刪超過 15 筆且超過 30% → 暫停，在設定頁讓使用者選「一起刪除／保留」。
- 開了 GitHub 同步時，這次開 App 要先 Gist 同步成功一次（`sync.okOnce`）才跑 Google 同步，避免用舊資料改 Google。
  時機：存檔後 3 秒、Gist 同步帶來變更後、畫面開著每 2 分鐘、切回 App。
- 測試：沒有可用的 Google 帳號時，用本機假的 Apps Script（直接執行 GAS_CODE ＋ 假 Calendar 服務）測；`gUrlOk` 在 localhost 允許 `http://localhost:<port>/exec`。

## 新增事項的方式（2026/10/14，使用者要求「參考 iPhone 行事曆」）
- 「＋」（頁首按鈕／手機右下圓鈕）直接開新增表單：月曆頁用選取的那天、認證頁開認證申請或證書、其他頁用今天。沒有下拉選單了。
- 新增表單像 iPhone「新增行程」：上方「取消／加入」、最上面可切換「事項／認證申請／證書效期」（`typeSeg`）、
  一句話輸入（`parseQuick`，像 Apple 行事曆的快速事件）會即時帶入下面欄位；內容與地點在前，整日開關、日期（今天／明天／後天／下週一捷徑）、
  開始／結束、重複（「重複到」只在有重複時出現）、屬性、類別、優先；編輯時下方有「已完成」與「刪除事項」。
- 月曆：點日期＝選取，下方（電腦是右側）當天清單上有一行快速新增（`qaBox`，Enter 加入、可選屬性、「完整表單 ›」帶著內容開表單）。
- `parseQuick` 認得：今天／明天／後天／大後天、N天後、（下／下下）週X、M/D、YYYY/M/D、N號；14:00、下午2點半、2pm、範圍「下午2點到4點」；
  `@地點`、`急件`／`!!`、`重要`／`!`（要用空白隔開）、`#工作／#個人／#shinnyo`、整天。
  也認得中文數字（`zhDigits`：兩點半、十月二十號、三天後；「早一點」「一點點」「第三點」不轉），不用切英數。
- 時間欄（`timeInp`、認證節點的 `.se-time`）：電腦（`pointer: fine`）上是唯讀＋`keyTime()` 自己接數字鍵——
  唯讀欄位不會啟動注音，使用者不用切英數（她的電腦是微軟注音）。進入欄位後第一個數字會取代原值；手機維持一般輸入框＋數字鍵盤。

## 視覺風格（使用者指定：簡約、像無印良品，不要太可愛；不要粉紅色）
2026/10/13 起由「日系手帳＋可愛小貼圖」改為 MUJI 風：
- 配色：素色生成紙底 --bg #F4F2EE（沒有方格）、卡片 --card、細線 --line、墨色文字 --ink；主要按鈕與選取狀態用炭灰 --accent；
  深紅 --brand #8B1E2B 只少量點綴（英文小標、每日一句標題、月曆「今天」、完成標記）。狀態色低彩度：完成 --mint（橄欖灰綠）、即將 --sun、逾期 --red。
  --tape-1～6 是認證申請的代表色（平塗，不再有紙膠帶點點）。
- 平面：沒有陰影（--shadow:none）、細實線（不用虛線）、小圓角（卡片與按鈕 6px、標籤 3px）。
- 字體：標題與內文 Noto Sans TC、數字用系統無襯線（--f-num）、閱讀文字（每日一句、心情結語）用 Noto Serif TC（--f-read）。
- **不要再加回**手繪小貼圖、紙膠帶、彩紙慶祝、跳動／搖鈴動畫、手寫字體。圖示用 `IC` 的細線圖示（stroke 1.6）。
- 「完成」是細線方框直排兩字（`stamp()`，深紅；「取證」灰色）；月曆週六藍、週日紅。
- `toast(msg, mood)`：沒有圖，只有左邊一條色線（happy 綠、worried／shock 紅、calm／sleepy 灰）。`emptyBox(title, sub)` 只有文字。

## 吉祥物
使用者自家的白色雪納瑞（Gemini 依她的照片產生的 AI 仿真照片；她不喜歡手繪卡通版）。
MUJI 風之後**只用在左上標誌與今日頁問候**：圓形照片、細邊框，不加徽章或膠帶。
`mascot(mood, cls)`：mood `happy|calm|worried|shock|sleepy` 對應 `PHOTO` 表
（目前 calm→happy、shock→worried，等使用者補「平靜」「驚訝」照片後新增 mascot/calm.jpg、shock.jpg 並改 PHOTO）。
App 圖示 icon-*.png 仍是「開心」照片。她原本的手機照片不放進版控。

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
改版後記得把 `sw.js` 的 `CACHE` 版本號 +1，手機才會拿到新版；**同時把 index.html 的 `APP_VERSION`（同一個數字）與 `APP_DATE` 一起改**——
使用者靠今日頁最下方、設定頁「目前版本 vNN」確認有沒有更新到；設定頁「檢查更新」會讀伺服器上的 sw.js 版本號來比對。
