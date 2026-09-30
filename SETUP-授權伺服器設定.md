# 授權伺服器設定指南

這份工具（私域流量與RFM顧客價值分析工作站）在頁面一開啟就會顯示「授權序號」的鎖定畫面，**必須輸入序號並按「確認」驗證通過，才能使用整個工具**（不是只鎖某個功能）。序號需連到你的 Google Sheet 確認是否還在 12 個月使用期限內。這個檢查是透過 **Google Apps Script**（Google Sheet 內建、免費）架設的一支小型 API 完成的。

## 你的 Google Sheet

<https://docs.google.com/spreadsheets/d/1sK0-LecMHkv628Zn2xCpqLBsL7k7htyVLeuZsj9E8jQ/edit>

這份是你指定沿用的既有任務追蹤表。程式固定操作一個**獨立分頁「PrivateDomainRFM序號」**（不會動到這份表裡其他分頁或其他用途的資料），分頁目前還不存在——**第一次有人驗證序號時會自動建立**，並自動寫入表頭「序號／開始日期／結束日期」。你也可以現在就手動去該試算表新增這個分頁並預先貼一筆測試序號，兩種方式都可以。

## 部署狀態：已用 clasp 完成部署

**已用 `clasp` 完成部署，你不需要手動操作 Apps Script 編輯器貼程式碼。** 過程：`clasp create --parentId <此Sheet的檔案ID>`（不加 `--type`，才能正確綁定到這份既有 Sheet）建立綁定腳本專案 → 推送 `Code.gs` → `appsscript.json` 加上 `webapp:{executeAs:"USER_DEPLOYING", access:"ANYONE_ANONYMOUS"}` → `clasp push --force` → `clasp deploy` 一次成功，跳過瀏覽器複製貼上與手動部署設定畫面。

部署網址已回填到 `index.html` 的 `LICENSE_CHECK_URL`：
```
https://script.google.com/macros/s/AKfycby94UpWjA6XK26wQVOgm_oz0bay5r0n7TF1qWPzn3ElggtJi2ioY0eXWYtyyQLVboM-DA/exec
```
Apps Script 編輯器（若之後要手動改程式碼或管理部署）：<https://script.google.com/d/1O_edw4CLtP-GljoZr4O7BaVRlr3mXyqyggJSREBwtcajzy1uO1qkSOXA/edit>

### ⚠️ 還差最後一步：你需要手動授權一次

`clasp deploy` 用 API 建立部署會跳過瀏覽器的「部署」精靈畫面，**但也因此跳過了 Google 要求的一次性 OAuth 授權**（讓這支腳本有權限讀寫你的 Sheet）。目前直接開啟部署網址會看到 Google 的「存取遭拒」頁面（已用 curl 實測確認）。修法很簡單，只需要你做一次：

1. 開啟上面的 Apps Script 編輯器連結。
2. 上方工具列函式下拉選單選 `doGet`，點「執行（Run）」▷ 按鈕。
3. 會跳出「需要授權」→ 選你的帳號 → 若出現「Google 尚未驗證這個應用程式」，點左下角「進階」→「前往...(不安全)」→ 允許。這是正常現象（因為這是你自己寫的私人腳本，沒有送 Google 審查），不是真的有安全疑慮。
4. 執行完成後（下方「執行紀錄」顯示成功），部署網址就會正常運作，不需要重新部署。

授權完成後，去該試算表新增分頁「PrivateDomainRFM序號」並填一筆測試序號（序號欄填任意字串，開始/結束日期留空），或直接在工具首頁的鎖定畫面試填一組序號按「確認」——第一次驗證會自動建立分頁＋自動開卡計時。

## 驗證部署是否成功

完成上面的手動授權步驟後，把部署網址直接貼到瀏覽器網址列開啟（GET 請求），應該會看到：

```json
{"ok":true,"message":"授權伺服器運作中。請用 POST 傳送 JSON body，例如 {\"serial\":\"your-serial-here\"}"}
```

看到這個就代表部署成功可用。**注意：不要用 `curl` 測試實際的驗證（POST 請求）**，Apps Script 的轉址機制會讓 curl 出現誤導性錯誤（不代表真的壞了）；請直接開啟工具測試，填入一組序號並按「確認」，確認鎖定畫面消失、狀態欄顯示「✓ 剩餘 N 天可用」。

## 之後每次要發新的序號要做什麼

**不需要重新部署 Apps Script。** 只要：

1. 打開 Google Sheet 的「PrivateDomainRFM序號」分頁，新增一列。
2. 「序號」欄填一組你要發出去的序號（例如用工作區的 `SN-maker` 序號產生器批次產生）。
3. 「開始日期」「結束日期」兩欄**留空**——第一次有人驗證這組序號時，系統會自動把「開始日期」寫成當下時間，「結束日期」自動算成開始日期 + 12 個月。
4. 把這組序號發給該使用者。

## 修改使用期限長度

在 `Code.gs` 開頭的 `const VALID_AMOUNT = 12;` 改掉這個數字，改完要用 `clasp push --force` 推送新版程式碼，再回到 Apps Script 編輯器「部署 → 管理部署作業 → 編輯 → 部署」一次（**部署網址不會變**，不需要再改前端檔案）。**注意：已經驗證啟用過、結束日期已寫入的序號不會回溯套用新期限**，只有尚未啟用（開始/結束日期都是空白）的序號才會套用新的天數。

## 常見問題

- **使用者按確認一直顯示「無法連線授權伺服器」或畫面顯示存取阻擋**：多半是還沒完成上面「還差最後一步：你需要手動授權一次」的動作。
- **改了 Code.gs 之後網址失效或行為沒變**：Apps Script 修改程式碼後，必須到「部署 → 管理部署作業 → 編輯（鉛筆圖示）→ 版本選「新版本」→ 部署」才會生效，只存檔不會自動更新已部署的網址。
- **想收回某組序號的使用權**：把該列的「結束日期」改成一個過去的日期即可，之後驗證都會回傳逾期，畫面會在最多 20 分鐘內自動重新鎖定。
