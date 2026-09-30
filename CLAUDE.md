# CLAUDE.md

此檔案為 `private-domain-rfm-studio/` 專屬說明，供 Claude Code 在此子資料夾工作時參考。根目錄 `C:\Users\mark_\AI Test\CLAUDE.md` 是整個工作區的總覽，此檔補充本專案細節。

## 專案定位

「私域流量與RFM顧客價值分析工作站」單檔前端工具，兩大分析模組合一、各自獨立可用，用頂部分頁切換：

- **💰 私域流量效益試算**：比較「一般行銷（單年、一次性）」vs「私域流量（訂閱制，依續訂率逐年遞減）」的長期效益。核心變數：年訂閱費／新訂戶數／續訂率／平均招攬成本／成本率，可設定「樂觀／一般／悲觀」三種情境。含：單批新訂戶 5 年生命週期衰退表、持續獲客的多世代疊加成長模型（6年）、續訂率情境比較（50/70/90%）、敏感度分析熱力表（新訂戶數 × 招攬成本 → 5年總淨利）。
- **👥 RFM顧客分群分析**：客戶清單（手動增刪列或上傳 CSV／Excel）依「基準日」即時計算 R／F／M 十分位評分（1-10 分，tie-aware percentile rank），再以 6 分為門檻組合成 8 種客群（此 8 客群分類是本工具額外設計補上的業界常見通用框架，**非原始課程試算表既有邏輯**——原始試算表只算到 R/F/M 三個分數為止）。

兩模組共用：頁面下方「⚙️ AI 設定」（BYOK，Claude/OpenAI/Gemini/OpenRouter 四選一，兩分頁共用同一組設定但各自獨立 prompt/結果）、各自的「📄 匯出試算報告（PDF）」（獨立 print-only 區塊 `#printReportRoot` + `#pdfWatermark`，非直接列印可見畫面）。

## 已知的模型設計決策（皆已寫入 manual.html，避免重複踩坑）

- **多年衰退模型的公式修正**：manual.html 明確記載本工具修正了原始課程用試算表的一處公式瑕疵——原始試算表「多年衰退模型」只有第一年正確套用所選方案的續訂率，之後年度會誤用固定方案的續訂率；本工具的 `singleCohortModel()` 已修正為全程一致套用同一個續訂率（`cohortDecay()`）。**修改這段邏輯前務必先讀 manual.html 對應段落**，避免不小心改回原始試算表的錯誤行為。
- **RFM 評分法**：`scoreFromValue()` 用 tie-aware percentile rank（`tieAwareRank()`：同值同分，取名次中點）換算成 1-10 分，等同常見的 PERCENTRANK 百分等級評分法，不是簡單的等寬分箱。
- **8 客群分類是本工具原創補充**，非對應任何官方或原始試算表規範，僅供參考。

## 資料流與儲存

- 純前端，無後端；`localStorage` 兩個獨立 key：`pdrfmTrafficState`（方案設定）、`pdrfmRfmState`（客戶清單＋基準日）。AI 設定另存 `pdrfmApiConfig`。
- CSV／Excel 匯入用 PapaParse + SheetJS（CDN），欄位標題比對支援中英文別名（見 `CSV_FIELD_ALIASES`）。
- 兩個分頁各自 5 組快速範例（虛構情境／虛構客戶），互不影響。

## 共用元件（跟隨工作區既有慣例，非本專案獨創）

- 頂部跑馬燈：獨立 `<script>` 區塊，串工作區共用的 Google Apps Script 公告端點，與主程式邏輯無關（改動主程式不會影響它，反之亦然）。
- PWA 加入主畫面：`manifest.json` + `service-worker.js` + 獨立 `<script>` 區塊，iOS/Mac Safari 走 fallback 提示訊息。
- 訪客計數器：`visitor-badge.laobi.icu`，`page_id=m255525.privatedomainrfmstudio`。

## 執行與測試

- 無建置步驟。本機預覽：`.claude/launch.json` 已註冊 port 8822（`private-domain-rfm-studio` 設定）。
- 測試優先用 Playwright 直接操作瀏覽器（切分頁、改變數、上傳/下載 CSV、無金鑰時點 AI 診斷應 toast 提示），不要真的點「📄 匯出試算報告（PDF）」按鈕做端對端驗證——會觸發 `window.print()`，曾在其他專案凍結 Playwright CDP 連線；PDF 版面正確性用程式碼審查 `buildPrintReportTraffic()`/`buildPrintReportRfm()` 或改用 `page.pdf()` 驗證，不要真點列印鈕。

## 授權與使用限制

manual.html 明確聲明：僅供教學、課程及個人使用，禁止未經授權公開發布、販售或商業化使用。**目前無序號授權機制**（跟 `mandala-thinking`／`scamper-thinking-generator`／`where-what-how-strategy-studio` 一樣走 manual-first 無鎖定路線），是否要加序號授權或推上公開 GitHub Pages 屬於商業/發布範圍決策，改動前請先跟使用者確認。
