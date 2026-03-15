## 社群打卡自動化模組

這個資料夾提供一套「LINE OA＋LIFF＋Google Sheet＋Apps Script」的打卡自動化範本，你可以直接複製到 Google Apps Script 與試算表中使用。

### 整體架構

- **LINE OA / 社群**: 提供使用者入口（主選單按鈕或關鍵字訊息）。
- **LIFF**: 在 LINE 內嵌一個簡單網頁打卡畫面。
- **Apps Script Web App**: 同時負責產出打卡網頁（HTML Service）與處理打卡請求（寫入 Google Sheet）。
- **Google Sheet**: 儲存原始打卡紀錄，並透過 Apps Script 或公式產生統計結果。

資料流大致如下：

- 使用者在 LINE OA 中點選「每日打卡」按鈕（或輸入關鍵字）。
- LINE 透過 LIFF 開啟 Apps Script Web App 網址。
- LIFF 於前端呼叫 `liff.getProfile()` 取得 `userId` 與 `displayName`。
- 前端使用 `google.script.run` 呼叫 Apps Script 後端：
  - 先呼叫 `getStatus(userId)`，顯示今日是否已打卡與目前累積點數。
  - 按下「打卡」後呼叫 `checkIn(userId, displayName)`，寫入 `Checkins` 工作表。
- 設定時間驅動觸發器定期執行 `recalcStats()`，將每位成員的總點數、打卡次數等寫入 `Stats` 工作表。
- （選配）打卡成功時，後端呼叫 LINE Messaging API 推播「打卡成功＋目前點數」訊息。

### Google Sheet 結構建議

建立一個 Google 試算表，內含兩個工作表：

- **`Checkins` 工作表（原始打卡紀錄）**
  - A 欄 `timestamp`：打卡時間（由 Apps Script 自動填入）。
  - B 欄 `date`：打卡日期（文字，例如 `2026-03-05`）。
  - C 欄 `lineUserId`：LINE 使用者唯一 ID。
  - D 欄 `displayName`：顯示名稱（方便辨識）。
  - E 欄 `groupId`：群組或房間 ID（如有需要可用，沒用到可留白）。
  - F 欄 `point`：該次打卡給予點數（通常為 1）。

- **`Stats` 工作表（彙總統計）**
  - A 欄 `lineUserId`
  - B 欄 `displayName`
  - C 欄 `totalPoints`：總點數。
  - D 欄 `checkinCount`：打卡次數。
  - E 欄 `lastCheckinDate`：最後打卡日期。

- **`Products` 工作表（選用，供 OCR 辨識商品名稱）**
  - A 欄 **品名**（或 `productName`）：正式名稱，會寫入 Checkins 的 `productName`，例如「珍珠奶茶」。
  - B 欄 **別名**（或 `aliases`）：包裝上可能出現的名稱，**逗號分隔**，例如「珍奶,珍珠奶茶,pearl milk tea」。
  - 第一列為標題列，第 2 列起一列一種商品。若未建立此工作表或無資料，程式會使用內建範例關鍵字（珍奶/黑咖啡/拿鐵）。上傳打卡圖片且未手動填品名時，會先以 Vision API OCR 取得文字，再依此表比對出商品與數量。

> 若你偏好用公式完成統計，也可以在 `Stats` 中用 `UNIQUE`, `COUNTIFS`, `SUMIFS` 等公式；本範例則示範用 Apps Script 的 `recalcStats()` 直接重算整張表。

### LINE OA 與 LIFF 設定指引（對應模組 1）

- 在 LINE 官方帳號後台：
  - 將「主選單」中的某一個按鈕命名為「每日打卡」，URL 指向稍後部署好的 Apps Script Web App 網址（同時也是 LIFF Endpoint）。
  - 或設定關鍵字（例如「打卡」），回覆一則含有 LIFF 連結的訊息按鈕。
- 在 LINE Developers 後台：
  - 使用你目前社群所屬的 Channel，新增一個 LIFF。
  - **LIFF Endpoint URL**：必須與 Google Apps Script「部署後」的 Web App URL **完全相同**（例如 `https://script.google.com/macros/s/xxxxx/exec`，結尾為 `/exec`）。若之後重新部署 GAS 且網址變更，必須回到 LINE Developers 的 LIFF 設定頁更新 Endpoint URL，否則會出現 LIFF 初始化超時。
  - Scope 至少勾選 `profile`，讓前端可以取得 `userId` 與 `displayName`。
  - 記下 `LIFF ID`，待會會填入 `Index.html` 內的常數。

### 部署步驟（高層指引）

1. 在 Google Drive 建立一份新的試算表，新增 `Checkins` 與 `Stats` 工作表並設定欄位。
2. 在同一份試算表內，開啟「擴充功能 → Apps Script」，建立 GAS 專案。
3. 將本資料夾 `apps-script/Code.gs` 與 `apps-script/Index.html` 的內容，分別貼到 Apps Script 專案中的檔案。
4. 在 `Index.html` 中將 `const LIFF_ID = 'YOUR_LIFF_ID';` 替換成實際的 LIFF ID。
5. 在 GAS 專案中設定 Script Properties：新增 `LINE_CHANNEL_ACCESS_TOKEN`（若暫時不需要推播，可略過）；**自動辨識打卡品名**需新增 **VISION_API_KEY**（Cloud Vision API 金鑰），否則上傳圖片不會跑 OCR、品名會留空。
6. 在 Apps Script 中：
   - 部署為「網路應用程式」，權限建議：任何知道連結的人皆可存取。
   - 把部署後的 URL 回填到 LINE Developers 的 LIFF Endpoint。
7. 在 Apps Script 的「觸發條件」中，新增一個時間驅動觸發器，定期呼叫 `recalcStats()`。

### 檔案說明

- `apps-script/Code.gs`：GAS 後端程式，處理打卡寫入與統計，以及（選用）呼叫 LINE Messaging API 推播訊息。
- `apps-script/Index.html`：HTML Service 前端頁面，透過 LIFF 取得使用者資訊，並用 `google.script.run` 呼叫 GAS 後端函式。

