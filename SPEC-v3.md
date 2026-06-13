# You're On — v3 Spec（增量，疊在 v2 上）

> v2 把追蹤變成節目。v3 讓節目可以「現場直播」，並把停播 / 回歸 / 月底這些
> 既有狀態補上該有的電影語言。
> v3 不推翻 v2 — 只加東西。這份只寫差異，**v2（SPEC-v2.md）的一切預設仍然成立**。
> 已實作上線：2026-06-13，commit `b8bbe3f`，storage key 不升版（仍 `reality_v6`）。

## 一句話

v2 的記錄是事後補登，v3 多了「邊做邊播」：按下計時器 = 你真的在直播這一集。

## 設計原則（延續 v2，補充三條）

- **即時性服務儀式感，不是監控**：計時器是讓你「想開」的工具，不是逼你盯的 KPI 碼表
- **LIVE 不能變成新的擺爛途徑**：無觀眾數加成 + >12h 防呆 —— 自我回報悖論的延伸防線
- **視覺要狀態化**：直播中 / 停播 / 回歸 / 月底，每個狀態都要一眼可辨的畫面，不靠文字解釋

## 與 v2 的差異總表

| | v2 | v3 |
|---|---|---|
| 記錄方式 | 事後填時數 | 事後填 **＋ LIVE 即時計時** |
| 「播出」 | 比喻（送出記錄＝播出） | **可真的現場直播**（計時中＝ON AIR） |
| 停播畫面 | 銀幕一行小字 | **NO SIGNAL 死台**（彩條＋雜訊） |
| 回歸 | 評論＋一行 airTxt | **全螢幕片頭卡＋回流 count-up** |
| 月底 | 只有月會觸發 | 月會＋**平時 dashboard 倒數** |
| 手機懸浮窗 | 無 | **Android PiP 浮動視窗** |

## 1. LIVE 計時器（核心新增）

- Record 頁頂新增計時卡。選一個 category → **START** → 過程可離開 app
- 停止 → 自動把經過時間換算成時數，**預填**進該 category 的輸入框（加到既有值，可手改）→ 再走原本的「播出本集」
- 兩條路並存：用 app 計時器、或自己手動輸入，都行

### 1a. 為什麼能跨 app／鎖屏／被殺存活
- 只存一個 timestamp（`startTs`），不存「累計秒數」。回來時 `現在 - startTs` 重算
- 切去別的 app、鎖屏、瀏覽器被系統回收都不影響——這是設計的重點，不是附帶效果

### 1b. LIVE = 真的直播（節目語言）
- 計時中：dashboard 的 ON AIR 燈變**紅色閃爍 LIVE**，銀幕顯示直播秒錶（這才是「正在播」，v2 的燈只代表「今天播過了」）
- 用 LIVE 錄的集數掛 **● LIVE 徽章**，並進 airTxt 小註（「本集 LIVE 直播」）

### 1c. 防擺爛（重要）
- **無觀眾數加成**：airToday 只認「當天首播」的 credit，LIVE 與手動同源、不額外給。掛著計時器去睡不會漲觀眾
- **>12h 防呆**：停止時若超過 12 小時，跳 Modal 問「你不是掛著計時器去睡吧」。確認照記、取消不填（停止照樣停）

## 2. Android 浮動視窗（PiP）

- 計時中出現「📱 浮動視窗」按鈕：canvas 畫秒錶 → `captureStream` → `<video>` → `requestPictureInPicture`
- 浮在所有 app 最上層，位置由 OS 管，**自動避開瀏海/系統列**，可拖
- **限制（誠實）**：
  - 只有 **Android Chrome** 真的能浮。`PiP.supported()` feature-detect，不支援就**不顯示按鈕**
  - **iPhone Safari、Firefox 不支援** → 退回「計時照算、但無浮窗」。這是 web 的天花板，iPhone 要浮窗只能包 native app（不在 v3 scope）
- 背景分頁 rAF 會被節流，所以用 `setInterval(500)` 重畫（秒級解析度足夠），秒錶在背景照跑

## 3. 視覺狀態三件套（補完 v2 §10 的電影感）

不是新機制，是給既有狀態「該有的畫面」。

### 3a. 停播訊號卡
- 停播（`offDays>=1`）時，dashboard 銀幕從暖光螢幕切成 **死台**：SMPTE 彩條 + 雜訊掃描線 + `NO SIGNAL`／訊號中斷
- 觀眾數仍在，但變**冷白**，配字「仍守著雜訊的觀眾」——數字是主角，但這時它在受苦

### 3b. 回歸特輯片頭卡
- 停播 ≥3 天後回來記錄 → 進評論前先放**全螢幕片頭卡**：金字「回歸特輯／SEASON RETURN」+ EP 字卡 + 觀眾回流 count-up
- 點擊才進評論。回歸是全 app 最該爽的時刻，值得這兩秒
- `refund=0`（沒流失量可回補）時，回流區塊不顯示——不假裝有

### 3c. 改版會議倒數
- dashboard 在每月最後 **≤10 天**顯示「SEASON FINALE 倒數 N 天」；當天顯示「今晚・改版會議」
- 用 season finale 語言補回 month 維度在平時的存在感（v2 的 month 只活在月會那一刻）

## 4. 技術

- 維持單檔 index.html（React CDN + babel standalone），**storage key 不升版**（`reality_v6`）
- 新增資料欄位：`liveSession`（`{startTs, catId}` 或 `null`）。load 補 default，無需 migration
- 新 helper：`fmtElapsed`（ms→`H:MM:SS`）、`msToHours`（ms→取 0.1h）
- `PiP` 模組（canvas/video/timer 控制器）；`liveSession` 變 `null` 時 App 的 effect 自動 `exitPictureInPicture`
- record 物件多一個可選 `live:true`
- dashboard 銀幕用 `screenMode = isLive ? 'live' : offDays>=1 ? 'dead' : 'lit'` 分流
- CSS 新增：`.live-pulse`、`.screen-dead`（含彩條/scanline）、`.flick`、`.cb-in`

## 5. 保留／不變（v2 全部成立）

- 觀眾數機制、播出/停播/連播/回歸特輯、開場宣言、月底改版會議、評論系統 v2（角色記憶/錯位/事件/試播）
- 三種評論格式與視覺、影廳/觀眾席、總表、深色主題與字體
- 補登過去日期只算時數、不觸發加分（policy 不變）

## 6. 已知限制（誠實備註）

- **iPhone 無浮窗** —— web 平台限制，非 bug。基本計時器全平台可用
- LIVE 的自我回報悖論同 v2 無解也不用解（誠實是前提）
- 觀眾數機制數值仍是猜的，待實測調整（v2 就標記的風險，v3 沒動到）

## 7. 還沒做／backlog

- **評論文案審稿**：`COMMENT-REVIEW.md` 384 條初稿待 user 挑選修改（線上目前跑的全是初稿）
- 觀眾數增減數值實際用幾天後微調
- LIVE 直播時數變成正式評論素材（目前只有 airTxt 小註，沒進評論池——要做得擴模板變數 + 加幾條 LINE/PTT）
