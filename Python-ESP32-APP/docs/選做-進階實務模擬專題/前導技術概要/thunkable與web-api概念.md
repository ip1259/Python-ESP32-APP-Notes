# Thunkable：App 與 Web API 的概念

Thunkable 是以畫面元件和積木安排 App 行為的工具。Gateway 是集中檢查資料並回覆結果的 Python 服務；Web API 則是 App 透過網路與它交換資料的窗口，通常使用 HTTP 傳送要求與回應。

## 你會學到什麼

- 理解 Thunkable 在本教材中的用途，以及 App 如何透過 Gateway Web API 取得資料。
- 看懂 App 發出 HTTP 要求並取得回應、狀態與錯誤的資料流。
- 理解 JSON 回應如何轉成 Thunkable `Object`（物件），再依屬性名稱取得資料。
- 知道 App 不應直接讀卡、保存 UID 或連線 MQTT Broker（負責轉送 MQTT 訊息的服務）。

## 開始前

- 本頁是工具與資料流概念，不需要建立 Thunkable 帳號或 App 專案。
- Thunkable 的帳號條件、介面與手機測試方式可能調整。本頁只說明資料流，不包含帳號或操作步驟。
- 本頁不提供登入、畫面製作、API 網址或手機權限步驟。

## Thunkable 在本教材中的位置

本教材使用 Thunkable，是因為它能以畫面元件與積木建立小型 App 介面，並透過 Web API 元件直接使用 Gateway 提供的 HTTPS API。這讓本頁可以聚焦在 HTTP 要求、JSON 資料、錯誤處理與畫面更新，不需要先學習完整的手機 App 開發框架。

App 在這裡只作為 Gateway API 的用戶端，不改變 ESP32 與 Gateway 原有的通訊方式，也不直接使用 MQTT、Broker 設定或硬體中的秘密資料。這項分工能縮小更換 App 工具時需要修改的範圍，但不表示只要使用 Thunkable 就會自動降低系統耦合。

若想了解產品團隊選擇代管 IoT 平台或自建 Gateway 時，會如何衡量開發速度、現成功能、維運責任與替換成本，可閱讀[延伸選讀：Blynk App 與 Gateway 的產品架構取捨](blynk-app與gateway產品架構取捨.md)。

Thunkable 減少的是輸入程式語法與設定開發環境的負擔；資料欄位、要求狀態、錯誤處理、權限與秘密資料仍要另外規劃。帳號方案、手機測試與 HTTPS API 整合方式可能隨版本改變；本頁只介紹資料流概念，不包含帳號或操作步驟。

## App 畫面與行為是兩個部分

| 部分 | 白話用途 | 例子 |
| --- | --- | --- |
| 畫面元件 | 讓使用者看見或操作 App。 | 按鈕、標籤、輸入欄位。 |
| 事件積木 | 決定某件事發生時要做什麼。 | 按下按鈕後讀取狀態。 |
| 變數 | 暫時保存 App 目前需要的值。 | 最近一次顯示文字。 |
| Web API 元件 | 經 HTTP 向 API 送出要求並接收結果。 | 對 Gateway 執行 `GET`。 |

積木式工具降低了撰寫介面程式的門檻，但不會自動解決資料格式、網路錯誤或權限問題。

## Web API 資料流

以下是 App 讀取 Gateway API 的資料流示意：

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

Web API 的 `response` 可能是一段 JSON 格式文字。JSON 用屬性名稱和值表示資料，例如 `"status": "waiting"` 中，`status` 是屬性名稱，`waiting` 是它的值。App 要先確認要求成功，再把 JSON 轉成 Thunkable 可以讀取的 `Object`。

## Thunkable 的 Object 是什麼？

`Object`（物件）可以把多個「屬性名稱與值」放在一起。它的用途類似 Python 的 `dict`：知道屬性名稱後，就能取出對應的值。

以上方 JSON 為例，轉成 `Object` 後可以這樣理解：

| 屬性名稱 | 值 | 值的種類 |
| --- | --- | --- |
| `event_id` | `evt-demo-001` | 文字 |
| `status` | `waiting` | 文字 |
| `display_text` | `尚無新事件` | 文字 |

在 Thunkable 中，讀取 API 資料的概念順序如下：

```text
取得 response
    ↓
確認 status 與 error
    ↓
把 JSON 文字轉成 Object
    ↓
依屬性名稱取得值
    ↓
更新標籤或其他畫面元件
```

JSON 是交換資料時使用的文字格式；`Object` 是 App 轉換後方便查找資料的結構。兩者內容可以對應，但不能因為 `response` 看起來像 JSON，就跳過轉換與檢查。

### 巢狀 Object 與清單

一個 `Object` 的值也可以是另一個 `Object`，這稱為巢狀物件。值也可能是一組有順序的資料，也就是清單。以下是資料形狀示意，不是目前正式 API 回應：

```json
{
  "device": {
    "name": "demo-board",
    "online": true
  },
  "readings": [24.5, 24.8]
}
```

若要取得 `demo-board`，要先取得 `device` 屬性的物件，再從裡面取得 `name`。`readings` 則是清單，應依清單位置取得其中一筆，不能把它當成具有屬性名稱的物件。Thunkable 清單的第一筆位置是 `1`，和 Python 清單從 `0` 開始不同。

### 取值前要先確認什麼？

- API 要求是否完成，而且 HTTP 狀態符合預期。
- `response` 是否為可轉換的 JSON。
- 需要的屬性是否存在，例如是否真的有 `display_text`。
- 取出的值是否為預期種類，例如文字、數字、布林值、物件或清單。

如果任何一項不符合預期，畫面應顯示可理解的錯誤，也不要繼續顯示上一次成功取得的舊資料。

## 在本專題的位置

以下是 App 與 Gateway 的責任分工示意：

```text
App ── HTTPS API ──> Gateway
App <── 最小摘要或錯誤 ── Gateway
```

App 是 Gateway API 的用戶端，不是規則中心。Gateway 才負責檢查輸入、套用名單規則、保存最小紀錄與決定是否發布家庭情境事件。

後續章節即使加入註冊或事件設定，App 也只能選擇 Gateway 已提供的有限功能；不能輸入任意 MQTT topic、直接修改資料檔或取得秘密設定。

## 為什麼不讓 App 直接連 MQTT

如果 App 直接保存 Broker 帳密、發布讀卡要求或自行判斷結果，名單規則與秘密資料會分散到更多地方。改由 App 只使用 Gateway API，可以把檢查與紀錄集中在同一位置，也能隔開手機介面與 ESP32 的通訊細節。日後更換 App 工具時，只要仍遵守 Gateway API 的資料契約，較不需要連帶改動 ESP32。

## 常見誤解

### 積木式 App 不需要處理錯誤嗎？

仍然需要。無網路、逾時、HTTP 錯誤和 JSON 格式不符都可能發生；積木只是另一種表達程式邏輯的方式。

### JSON 和 Object 是同一個東西嗎？

不是。JSON 是 API 傳送資料時常用的文字格式；`Object` 是 Thunkable 將 JSON 轉換後，用來依屬性名稱取得資料的結構。App 應先檢查回應，再進行轉換與取值。

### App 收到 `whitelisted` 就可以直接控制門鎖嗎？

不可以。本專題只顯示模擬狀態與家庭情境效果，不控制真實門鎖或其他高風險設備。

### 可以把 API token 放在文字積木裡嗎？

不要直接這樣做。API token（用來辨認並授權使用者或程式的秘密字串）若寫在 App 的文字積木中，可能隨專案或匯出檔被他人取得。正式使用時，必須另外設計並確認身分與秘密保存方式。

## 理解確認

- 能理解 Thunkable 在本教材中用來建立小型 App 介面，並讓 App 直接使用 Gateway API。
- 能分辨畫面元件、事件積木、變數與 Web API 元件。
- 知道 API 要求可能得到回應、HTTP 狀態或錯誤。
- 能分辨 JSON 文字、`Object`、屬性與清單的基本差異。
- 知道 JSON 屬性不存在或資料種類不符時不能沿用舊結果。
- 知道 App 只透過 Gateway API 使用有限功能，不直接使用 MQTT 或 UID。

## 重點整理

- Thunkable 適合本教材目前用積木理解小型 App 介面與 Web API 資料流的範圍。
- App 經 Gateway API 使用明確約定的資料格式，不直接介入 ESP32 與 Gateway 原有的通訊。
- Thunkable 用畫面元件與積木建立 App 行為。
- Web API 元件可送出 HTTP 要求，App 仍須檢查狀態、錯誤與 JSON 欄位。
- JSON 回應要先轉成 `Object`，才能依屬性名稱取得值；巢狀物件與清單需要分層讀取。
- Gateway 是規則與紀錄的集中位置；App 只是受限的 API 用戶端。
- 實際使用前，仍須確認帳號、手機、API 與錯誤流程符合當時環境。

## 參考資料

- [Thunkable 官方文件：Getting Started](https://docs.thunkable.com/getting-started)
- [Thunkable 官方文件：Web APIs Blocks](https://docs.thunkable.com/blocks/advanced-app-features/web-api)
- [Thunkable 官方文件：Bluetooth Low Energy Blocks](https://docs.thunkable.com/blocks/advanced-app-features/bluetooth-low-energy)
- [Thunkable 官方文件：Objects Blocks](https://docs.thunkable.com/blocks/blocks/objects)

## 下一步

接著可進行[選做實作：Thunkable 瀏覽政府開放資料清單](thunkable專題資訊卡介面.md)。你會調整 `page`、`size` 查詢參數，保存 API 回傳的 JSON 清單，並在 Thunkable Live 中選擇顯示其中一筆公開資料；不會連接 BLE、ESP32 或其他服務。
