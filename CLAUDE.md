# CLAUDE.md

此檔案為 `private-domain-rfm-studio/` 專屬說明，供 Claude Code 在此子資料夾工作時參考。根目錄 `C:\Users\mark_\AI Test\CLAUDE.md` 是整個工作區的總覽，此檔補充本專案細節。

## 專案定位

「私域流量與RFM顧客價值分析工作站」單檔前端工具，兩大分析模組合一、各自獨立可用，用頂部分頁切換：

- **💰 私域流量效益試算**：比較「一般行銷（單年、一次性）」vs「私域流量（訂閱制，依續訂率逐年遞減）」的長期效益。核心變數：年訂閱費／新訂戶數／續訂率／平均招攬成本／成本率，可設定「樂觀／一般／悲觀」三種情境。含：**行銷預算編列（招攬成本雙向試算，2026-09-30 新增，見下）**、單批新訂戶 5 年生命週期衰退表、持續獲客的多世代疊加成長模型（6年）、續訂率情境比較（50/70/90%）、敏感度分析熱力表（新訂戶數 × 招攬成本 → 5年總淨利）。
- **👥 RFM顧客分群分析**：客戶清單（手動增刪列或上傳 CSV／Excel）依「基準日」即時計算 R／F／M 十分位評分（1-10 分，tie-aware percentile rank），再以 6 分為門檻組合成 8 種客群（此 8 客群分類是本工具額外設計補上的業界常見通用框架，**非原始課程試算表既有邏輯**——原始試算表只算到 R/F/M 三個分數為止）。

兩模組共用：頁面下方「⚙️ AI 設定」（BYOK，Claude/OpenAI/Gemini/OpenRouter 四選一，兩分頁共用同一組設定但各自獨立 prompt/結果）、各自的「📄 匯出試算報告（PDF）」（獨立 print-only 區塊 `#printReportRoot` + `#pdfWatermark`，非直接列印可見畫面）。**浮水印機制從一開始建置時就已存在**（`watermark-source.png` 已去背馬克老師吉祥物圖＋`#pdfWatermark`／`#wmImg`＋`@media print` CSS，比照 `new-product-strategy-studio` 等姊妹專案的做法），2026-09-30 用 `page.emulateMedia({media:'print'})`＋截圖實測過確認正常顯示，並非本次新增；使用者反映「要加浮水印」時純屬先前沒有實際驗證過，不是機制真的缺漏。

**2026-09-30 為 RFM 報告加上客群分布圖表截圖**：`buildPrintReportRfm()` 原本只有文字表格，現在額外用 `chartToImageWithBg("segmentChart","#161A2C")`／`chartToImageWithBg("scatterChart","#161A2C")`（定義在 `renderChart()` 附近）把畫面上的長條圖／散佈圖轉成內嵌 `<img>`。**這裡刻意不是簡單呼叫 Chart.js 內建的 `toBase64Image()`**——那樣匯出的 PNG 背景透明，圖表的座標軸/圖例文字用的是淺色（`CHART_TEXT_COLOR="#A6ACC9"`，給深色底配色），直接疊在報告的白底版面上會看不清楚。`chartToImageWithBg()` 用 `ctx.globalCompositeOperation="destination-over"` 在畫布內容「後面」補畫一塊純色矩形再輸出 dataURL，輸出後立刻用事先存好的 `ImageData` 復原成原本透明的畫布（不影響畫面上即時互動的圖表本身）。深色底刻意畫進 PNG 像素、不是用 CSS `background-color`——後者預設會被瀏覽器「列印背景圖形」選項擋掉（使用者沒勾選就會消失），`<img>` 內容不受這個選項影響，跟浮水印圖片是同一個道理。

## 行銷預算編列（招攬成本試算，2026-09-30 新增）

使用者要求把「招攬每一位新客的成本」也納入行銷預算編列的考量。實作為雙向連動計算機（`budgetCard`）：「行銷預算總額」與「目標招攬新客數」兩個 input 互相換算，換算基準是目前選定方案的 `acqCost`（平均招攬成本/位）。

- `budgetState = {mode, amount, customers}`（`pdrfmBudgetState` localStorage 記住），`mode` 記錄「最後修改的是哪一欄」——這個設計是刻意的：切換方案／範例導致 `acqCost` 改變時，要保留使用者原本的意圖（以預算為主還是以人數為主）重新換算另一欄，而不是兩欄都保持不變或都被覆寫。`computeBudgetInfo()` 依 `mode` 決定算哪一個：`mode==="amount"` 時 `customers = Math.floor(amount/acqCost)`（無條件捨去，不會有半個客戶）；`mode==="customers"` 時 `amount = Math.round(customers*acqCost)`。
- 下方 KPI 用算出的 `customers` 覆寫 `newSubs` 跑一次既有的 `singleCohortModel()`（5年），跟「單批新訂戶」卡片同一套模型，只是新訂戶數換成這裡設定的目標人數——刻意重用既有函式而非另寫一套計算邏輯。
- `renderBudgetSection()` 掛在 `recalcTraffic()` 開頭，任何會觸發 `recalcTraffic()` 的操作（切換方案/範例、編輯方案表格）都會連帶重算這個區塊。
- `buildPrintReportTraffic()` 已同步加入這個區塊的內容（預算/人數/5年營收/5年淨利），使用者原話「另外也要增加PDF下載功能」實際上指的是「把這個新功能也納入既有的匯出試算報告（PDF）」，**不是另外做一個新的下載按鈕**——這個工具本來就有 `pdfBtnTraffic`／`pdfBtnRfm` 兩個既有匯出按鈕，理解需求時要留意這點，不要重複造一個功能相同的按鈕。

## 已知的模型設計決策（皆已寫入 manual.html，避免重複踩坑）

- **多年衰退模型的公式修正**：manual.html 明確記載本工具修正了原始課程用試算表的一處公式瑕疵——原始試算表「多年衰退模型」只有第一年正確套用所選方案的續訂率，之後年度會誤用固定方案的續訂率；本工具的 `singleCohortModel()` 已修正為全程一致套用同一個續訂率（`cohortDecay()`）。**修改這段邏輯前務必先讀 manual.html 對應段落**，避免不小心改回原始試算表的錯誤行為。
- **RFM 評分法**：`scoreFromValue()` 用 tie-aware percentile rank（`tieAwareRank()`：同值同分，取名次中點）換算成 1-10 分，等同常見的 PERCENTRANK 百分等級評分法，不是簡單的等寬分箱。
- **8 客群分類是本工具原創補充**，非對應任何官方或原始試算表規範，僅供參考。

## 資料流與儲存

- 純前端，無後端；`localStorage` 兩個獨立 key：`pdrfmTrafficState`（方案設定）、`pdrfmRfmState`（客戶清單＋基準日）。AI 設定另存 `pdrfmApiConfig`。
- CSV／Excel 匯入用 PapaParse + SheetJS（CDN）。**2026-09-30 改為「自動辨識＋確認視窗」流程**（不再是猜不到就直接失敗）：`mapHeaders()` 先用 `CSV_FIELD_ALIASES` 精確比對表頭別名，比對不到的欄位再用 `FIELD_HEURISTICS` 關鍵字子字串猜測。**`FIELD_HEURISTICS.id` 刻意不含單獨的「代號」「編號」**——這兩個字太通用，「訂單編號」「商品編號」這類非客戶欄位也會誤中（實測踩過這個坑：「客戶」＋「訂單編號」兩欄並存時，「訂單編號」曾被誤判成客戶代號，因為它先出現且包含「編號」子字串）；只保留「客戶／顧客／會員／customer／member／name／姓名」這類明確指向客戶身分的關鍵字。
- **同日再追加「逐筆交易紀錄」彙總模式**：使用者反映實務上很多匯出檔是「每筆訂單一列」而非「每位客戶一列」，看不出哪欄是「購買次數」或「最近購買日期」（因為這兩欄根本不存在，需要從交易列彙總算出）。`#csvMapModal` 因此分兩組欄位對應（`MAP_FIELD_SELECTS_CUSTOMER` 4 欄 vs `MAP_FIELD_SELECTS_TXN` 3 欄：客戶代號／交易日期／交易金額），用 radio 切換；`rowsFromTransactions()` 依客戶代號分組，購買次數＝列數、最近購買日期＝交易日期最大值、累計消費金額＝交易金額加總。`openMapModal()` 猜不到 `frequency` 欄位但猜得到 `id` 時，預設自動切換到交易模式（`defaultTxnMode`），減少使用者自己發現要切換模式的摩擦。
- 不論哪種模式，一律跳出確認視窗讓使用者用下拉選單確認或手動調整，即時重繪前 3 位客戶的 `renderMapPreview()` 彙總後預覽，按「確認匯入」才真正寫入 `rfmState.customers`。過長的儲存格內容用 `truncateCellText()`（24字+刪節號）截斷顯示並加 `title` 屬性供 hover 查看完整內容，避免預覽表格/下拉選單被撐爆版面。
- 兩個分頁各自 5 組快速範例（虛構情境／虛構客戶），互不影響。
- **2026-09-30 加上搜尋／縮放／客群篩選**（使用者反映真實 Excel 匯入後筆數太多影響觀看）：客戶清單表格與客戶明細表格皆有搜尋框（`applyCustomerSearchFilter()`／`getFilteredDetailRows()`），客戶清單用「隱藏不比對的列」而非重新渲染，避免打亂 `data-idx` 對應到 `rfmState.customers` 陣列索引；顯示縮放用 CSS `zoom` 屬性（`--rfm-zoom` CSS variable，套在 `.rfm-zoomable` 的 `.table-scroll` 容器上，70%-130%，`pdrfmTableZoom` 記住設定）。**客戶明細的篩選只影響顯示，`recalcRfm()` 仍對全體 `rfmState.customers` 呼叫 `computeRfm()`**——R/F/M 十分位評分本來就要看全體客戶的相對分布才有意義，KPI／長條圖／散佈圖固定吃未篩選的 `lastRfmRows`，只有 `renderRfmDetailTable()` 吃 `getFilteredDetailRows(lastRfmRows)`；改這段時務必維持這個界線，不要把篩選後的子集拿去重算分數。
- **同日再加上「客戶清單與基準日」「客戶明細」兩張卡片各自獨立收合**（使用者要求，這兩張卡片內容長，筆數多時把其他區塊擠到很下面）：`#customerListCard`／`#detailCard` 各自用 `.collapsed` class 切換，收合時隱藏 `#customerListBody`／`#detailBody`（說明文字/工具列/篩選列/表格全部收在這個容器內）、改顯示各自的 `#customerListCollapsedHint`／`#detailCollapsedHint` 摘要列；`pdrfmCustomerListCollapsed`／`pdrfmDetailCollapsed` 兩個獨立 localStorage key 各自記住開合狀態。`loadCustomerListCollapse()`／`loadDetailCollapse()` 需在 `loadRfmState()` 之後呼叫才能拿到正確的客戶筆數摘要文字；`updateDetailCollapsedHint()` 掛在 `renderRfmDetailTable()` 尾端，每次篩選/重算都會更新摘要裡的總筆數（用 `totalCount` 參數而非篩選後的 `rows.length`，收合摘要要顯示全體筆數不是篩選後的子集筆數）。
- **同日再加上 F/M 範圍篩選＋欄位顯示/隱藏**（使用者反映客戶明細欄位太多太寬）：`getFilteredDetailRows()` 擴充 `detailFreqMin/Max`／`detailMonMin/Max` 四個 number input 做範圍篩選（留空即不限，用 `parseFloat` + `isNaN` 判斷，不是用 truthy 判斷 0 這種合法邊界值）。欄位顯示/隱藏用 `DETAIL_COLUMNS` 陣列（`{key,label,render(r)}`，客戶代號／客群固定不在此陣列內、永遠顯示）＋ `detailColVisible` 物件（`pdrfmDetailColumns` localStorage 記住設定）＋原生 `<details><summary>` 摺疊面板（免寫開闔 JS），`renderRfmDetailTable()` 依 `detailColVisible` 動態組表頭與每列儲存格，不是寫死 9 個固定欄位。

## 共用元件（跟隨工作區既有慣例，非本專案獨創）

- 頂部跑馬燈：獨立 `<script>` 區塊，串工作區共用的 Google Apps Script 公告端點，與主程式邏輯無關（改動主程式不會影響它，反之亦然）。
- PWA 加入主畫面：`manifest.json` + `service-worker.js` + 獨立 `<script>` 區塊，iOS/Mac Safari 走 fallback 提示訊息。
- 訪客計數器：`visitor-badge.laobi.icu`，`page_id=m255525.privatedomainrfmstudio`。

## 執行與測試

- 無建置步驟。本機預覽：`.claude/launch.json` 已註冊 port 8822（`private-domain-rfm-studio` 設定）。
- 測試優先用 Playwright 直接操作瀏覽器（切分頁、改變數、上傳/下載 CSV、無金鑰時點 AI 診斷應 toast 提示），不要真的點「📄 匯出試算報告（PDF）」按鈕做端對端驗證——會觸發 `window.print()`，曾在其他專案凍結 Playwright CDP 連線；PDF 版面正確性用程式碼審查 `buildPrintReportTraffic()`/`buildPrintReportRfm()` 或改用 `page.pdf()` 驗證，不要真點列印鈕。

## 授權與使用限制

manual.html 明確聲明：僅供教學、課程及個人使用，禁止未經授權公開發布、販售或商業化使用。**2026-09-30 已加上序號授權，鎖定整個工具**（`#licenseGate` 全螢幕遮罩，比照 `amazon-listing-generator`／`new-product-strategy-studio` 的做法；12個月效期，即時重驗不快取，背景每20分鐘重驗一次）：

- 綁定的 Google Sheet：使用者指定的既有任務追蹤表 <https://docs.google.com/spreadsheets/d/1sK0-LecMHkv628Zn2xCpqLBsL7k7htyVLeuZsj9E8jQ/edit>，固定操作獨立分頁 **「PrivateDomainRFM序號」**（`Code.gs` 的 `SHEET_NAME` 常數），不掃描該試算表裡其他分頁；分頁不存在時第一次驗證會自動建立並寫入表頭。
- 部署方式：`clasp create --parentId <SheetID>`（不加 `--type`）→ 推送 `Code.gs` → `appsscript.json` 加 `webapp:{executeAs:"USER_DEPLOYING",access:"ANYONE_ANONYMOUS"}` → `clasp push --force` → `clasp deploy`，全程在 `.gas-deploy/`（已加入 `.gitignore`）內操作，跳過瀏覽器複製貼上。
- `LICENSE_CHECK_URL = https://script.google.com/macros/s/AKfycby94UpWjA6XK26wQVOgm_oz0bay5r0n7TF1qWPzn3ElggtJi2ioY0eXWYtyyQLVboM-DA/exec`；Apps Script 編輯器：<https://script.google.com/d/1O_edw4CLtP-GljoZr4O7BaVRlr3mXyqyggJSREBwtcajzy1uO1qkSOXA/edit>。
- **⚠️ 部署後尚待使用者完成一次性 OAuth 授權**（`clasp deploy` 用 API 建立部署會跳過瀏覽器部署精靈附帶的授權流程，目前開啟部署網址會看到 Google「存取遭拒」頁面，已用 curl 實測確認）——步驟見 `SETUP-授權伺服器設定.md`。完成授權前，工具首頁會永遠停留在鎖定畫面（fail-closed，符合設計預期，不是 bug）。
- `localStorage` key：`pdrfmSerial`（不與 `pdrfmTrafficState`／`pdrfmRfmState`／`pdrfmApiConfig` 衝突）。
- 是否要推上公開 GitHub Pages 仍是刻意保留的待確認決策（2026-09-30 使用者明確選擇「先不要，維持純本機」），跟序號授權是兩件獨立的事——序號授權已完成，公開部署與否之後再問。
