# Thunkable：App 與 Web API 的概念

Thunkable 是以畫面元件和積木安排 App 行為的工具。Gateway 是集中檢查資料並回覆結果的 Python 服務；Web API 則是 App 透過網路與它交換資料的窗口，通常使用 HTTP 傳送要求與回應。

## 你會學到什麼

- 理解畫面元件、事件積木、變數與 Web API 元件的分工。
- 看懂 App 發出 HTTP 要求並取得回應、狀態與錯誤的資料流。
- 理解 JSON 物件如何讓 App 取得指定欄位。
- 知道 App 不應直接讀卡、保存 UID 或連線 MQTT Broker（負責轉送 MQTT 訊息的服務）。

## 開始前

- 本頁是工具與資料流概念，不需要建立 Thunkable 帳號或 App 專案。
- Thunkable 的帳號條件、介面與手機測試方式可能調整。本頁只說明資料流，不包含帳號或操作步驟。
- 本頁不提供登入、畫面製作、API 網址或手機權限步驟。

## App 畫面與行為是兩個部分

| 部分 | 白話用途 | 例子 |
| --- | --- | --- |
| 畫面元件 | 讓使用者看見或操作 App。 | 按鈕、標籤、輸入欄位。 |
| 事件積木 | 決定某件事發生時要做什麼。 | 按下按鈕後讀取狀態。 |
| 變數 | 暫時保存 App 目前需要的值。 | 最近一次顯示文字。 |
| Web API 元件 | 經 HTTP 向 API 送出要求並接收結果。 | 對 Gateway 執行 `GET`。 |

積木式工具降低了撰寫介面程式的門檻，但不會自動解決資料格式、網路錯誤或權限問題。

## Web API 資料流

```text
使用者按下按鈕
    ↓
Thunkable Web API 元件送出 GET 要求
    ↓
Gateway 回傳 JSON、HTTP 狀態或錯誤
    ↓
App 檢查結果後更新畫面
```

Thunkable 官方 Web API 文件說明，`Get` 積木會提供 `response`、`status` 與 `error`。App 不能只讀 `response` 就假設成功；應先分辨要求是否完成、狀態是否符合預期，以及 JSON 是否具有必要欄位。

## JSON 在 App 中的角色

以下是用途示意，不是目前正式 API 回應：

```json
{
  "event_id": "evt-demo-001",
  "status": "waiting",
  "display_text": "尚無新事件"
}
```

App 可將 JSON 轉成物件，再依欄位名稱取得 `status` 或 `display_text`。如果回應不是 JSON、欄位不存在或資料型別不同，畫面應顯示可理解的錯誤，不保留舊的成功內容。

## 在本專題的位置

```text
App ── HTTPS API ──> Gateway
App <── 最小摘要或錯誤 ── Gateway
```

App 是 Gateway API 的用戶端，不是規則中心。Gateway 才負責檢查輸入、套用名單規則、保存最小紀錄與決定是否發布家庭情境事件。

後續章節即使加入註冊或事件設定，App 也只能選擇 Gateway 已提供的有限功能；不能輸入任意 MQTT topic、直接修改資料檔或取得秘密設定。

## 為什麼不讓 App 直接連 MQTT

如果 App 直接保存 Broker 帳密、發布讀卡要求或自行判斷結果，名單規則與秘密資料會分散到更多地方。改由 App 只使用 Gateway API，可以把檢查與紀錄集中在同一位置，也讓日後更換 App 工具時較不需要改動 ESP32。

## 常見誤解

### 積木式 App 不需要處理錯誤嗎？

仍然需要。無網路、逾時、HTTP 錯誤和 JSON 格式不符都可能發生；積木只是另一種表達程式邏輯的方式。

### App 收到 `whitelisted` 就可以直接控制門鎖嗎？

不可以。本專題只顯示模擬狀態與家庭情境效果，不控制真實門鎖或其他高風險設備。

### 可以把 token 放在文字積木裡嗎？

不要。App 專案、畫面與匯出檔都有可能被他人取得。正式使用時，必須另外設計並確認身分與秘密保存方式。

## 理解確認

- 能分辨畫面元件、事件積木、變數與 Web API 元件。
- 知道 API 要求可能得到回應、HTTP 狀態或錯誤。
- 知道 JSON 欄位不存在或型別錯誤時不能沿用舊結果。
- 知道 App 只透過 Gateway API 使用有限功能，不直接使用 MQTT 或 UID。

## 重點整理

- Thunkable 用畫面元件與積木建立 App 行為。
- Web API 元件可送出 HTTP 要求，App 仍須檢查狀態、錯誤與 JSON 欄位。
- Gateway 是規則與紀錄的集中位置；App 只是受限的 API 用戶端。
- 實際使用前，仍須確認帳號、手機、API 與錯誤流程符合當時環境。

## 參考資料

- [Thunkable 官方文件：Web APIs Blocks](https://docs.thunkable.com/blocks/advanced-app-features/web-api)
- [Thunkable 官方文件：Objects Blocks](https://docs.thunkable.com/blocks/core-features/objects)

## 下一步

查看[Thunkable 專題資訊卡介面](thunkable專題資訊卡介面.md)。這項實作只建立靜態畫面，不連接 API、BLE 或其他服務。
