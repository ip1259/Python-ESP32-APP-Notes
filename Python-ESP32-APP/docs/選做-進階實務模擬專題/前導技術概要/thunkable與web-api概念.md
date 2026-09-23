# Thunkable：App 與 Web API 的概念

Thunkable 是以畫面元件和積木安排 App 行為的工具。Gateway 是集中檢查資料並回覆結果的 Python 服務；Web API 則是 App 透過網路與它交換資料的窗口，通常使用 HTTP 傳送要求與回應。

## 你會學到什麼

- 理解本教材在此階段選擇 Thunkable 的原因，並能辨認 Flutter + Dart、Blynk 與 MIT App Inventor 的不同定位。
- 看懂 App 發出 HTTP 要求並取得回應、狀態與錯誤的資料流。
- 理解 JSON 物件如何讓 App 取得指定欄位。
- 知道 App 不應直接讀卡、保存 UID 或連線 MQTT Broker（負責轉送 MQTT 訊息的服務）。

## 開始前

- 本頁是工具與資料流概念，不需要建立 Thunkable 帳號或 App 專案。
- Thunkable 的帳號條件、介面與手機測試方式可能調整。本頁只說明資料流，不包含帳號或操作步驟。
- 本頁不提供登入、畫面製作、API 網址或手機權限步驟。

## 為什麼本教材選擇 Thunkable

本教材選擇 Thunkable，是為了先用積木建立跨平台畫面，練習 App 透過 HTTPS API 取得 JSON 資料，並把注意力放在資料流、畫面與基本錯誤處理。本教材暫時不以完成可上架的產品、管理大量裝置或建立完整 App 架構為目標。

Thunkable 目前主要推廣 AI Vibe Coding（用文字描述需求，讓 AI 協助產生 App）。免費方案提供的 AI token（使用 AI 功能時消耗的額度）有限，有興趣可以自行試用，但不影響本教材的進度。為了看清楚畫面元件與事件邏輯如何組成 App，也避免把 AI 額度當成必要條件，本教材仍採用較傳統的視覺化方式，讓你透過拖放元件與積木式程式理解操作過程。

這不是在比較哪一個工具最好；不同專案會有不同選擇。本專題的 App 規劃為 Gateway API 的用戶端，不改變 ESP32 與 Gateway 的通訊架構，也不讓 App 直接使用 MQTT、Broker 設定或硬體中的秘密資料。

下表不是所有 App 工具的排名，而是比較它們在目前教材範圍中的位置：

| 方案 | 特性與本專題的取捨 |
| --- | --- |
| Flutter + Dart | 是以程式碼建立完整跨平台 App 的常見方式，適合日後想深入 App 開發的人。不過需要額外學習 Dart、Flutter SDK（讓電腦建立 Flutter App 的工具組）、畫面元件與 App 狀態處理；對目前只建立資訊卡與認識 API 的階段，學習範圍較大。 |
| Blynk | 可快速建立 IoT 儀表板與控制畫面。常見的 Blynk Library 流程會讓 ESP32 以 Blynk 自家定義的二進位串流協議（裝置與雲端用二進位資料持續交換內容的規則）連到 Blynk.Cloud（Blynk 的雲端服務），並以 Virtual Pin（平台使用的非實體資料編號）、Datastream（平台中的資料通道）與 Blynk 函式收發資料。這些封裝能加快原型製作；但如果裝置核心邏輯直接依賴它們，日後更換 App、Gateway 或雲端服務時，可能要一起改寫資料模型與連線方式。 |
| MIT App Inventor | 同樣能以積木方式開始製作 App，也有許多入門資源。不過本專題後段可能需要 BLE；iOS 的藍牙與 extension（擴充元件）支援情況必須依目標手機和當時版本實測確認，所以不把它作為本教材的預定工具。這不代表 MIT App Inventor 較差。 |
| Thunkable | 可用畫面元件與積木建立跨平台介面，並以 Web API 元件處理 HTTPS API 與 JSON。它不要求 ESP32 改用特定 App 平台的函式庫；日後若轉往 Flutter 或原生 App，ESP32 與 Gateway 的責任分工仍可維持。 |

這些 Blynk 平台封裝不是實體 GPIO，也不是本專題採用的 HTTP 網址／JSON 欄位或 MQTT topic／payload 資料契約（雙方約定的欄位名稱、格式與意思）。如果學習只停在單一平台的函式，換到自建 API、一般 MQTT Broker 或另一家雲端服務時，仍要補上如何設計要求、回應、topic、payload 與錯誤處理。本教材優先練習這些較容易跨工具使用的概念，再把 App 接到 Gateway API；這能降低 App 工具直接綁住 ESP32 程式的程度。

這裡的「綁得較緊」就是系統耦合性：更換其中一部分時，其他部分也必須跟著修改的程度。若想進一步理解判斷方式，可閱讀[資訊補充：什麼是系統耦合性？](../../資訊補充-系統耦合性與替換彈性.md)。

### 為什麼本教材不採用 Blynk？

在使用 Blynk.Cloud 的常見 Blynk Library 流程中，ESP32 會使用 Blynk 自家的通訊方式連接雲端。即使改用 Blynk 提供的 HTTPS／MQTT API，資料仍須先送到 Blynk.Cloud，再由 Blynk App 取得：

```text
ESP32 → Blynk.Cloud → Blynk App
```

因此，採用 Blynk 時，Blynk.Cloud 是資料流的一部分，無法直接抽換掉。本教材採用的是另一種架構：以 Gateway 模擬實務開發中由團隊自行建置與控制的自家伺服器，再讓 App 直接使用 Gateway 提供的 HTTP／JSON API。這讓服務與資料格式由專題自行控制，也保留日後更換 App 或雲端服務的彈性，所以不以 Blynk 作為實作方案。

基於這項取捨，本教材以 Thunkable 作為 App 概念與靜態介面的工具。它的用途是理解「App 透過 Gateway API 使用有限功能」，不是替代 Gateway、MQTT 或 ESP32 的既有角色。

Thunkable 減少的是輸入程式語法與設定開發環境的負擔；資料欄位、要求狀態、錯誤處理、權限與秘密資料仍要由人仔細規劃。帳號方案、手機測試、HTTPS API 整合與 BLE 功能可能隨版本與裝置改變；本頁只介紹概念，實作時以對應選做頁與官方文件列出的條件為準。

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

- 能理解本教材選擇 Thunkable 是為了先聚焦小型介面與 HTTP／JSON 資料流，而不是全面比較工具好壞。
- 能理解本專題為何避免讓 App 工具的選擇直接綁住 ESP32 程式。
- 能分辨畫面元件、事件積木、變數與 Web API 元件。
- 知道 API 要求可能得到回應、HTTP 狀態或錯誤。
- 知道 JSON 欄位不存在或型別錯誤時不能沿用舊結果。
- 知道 App 只透過 Gateway API 使用有限功能，不直接使用 MQTT 或 UID。

## 重點整理

- Thunkable 適合本教材目前用積木理解小型 App 介面與 Web API 資料流的範圍；其他工具也有各自適合的情境。
- App 經 Gateway API 使用固定資料格式，可降低更換 App 工具時連帶修改 ESP32 程式的機會。
- Thunkable 用畫面元件與積木建立 App 行為。
- Web API 元件可送出 HTTP 要求，App 仍須檢查狀態、錯誤與 JSON 欄位。
- Gateway 是規則與紀錄的集中位置；App 只是受限的 API 用戶端。
- 實際使用前，仍須確認帳號、手機、API 與錯誤流程符合當時環境。

## 參考資料

- [Thunkable 官方文件：Getting Started](https://docs.thunkable.com/getting-started)
- [Thunkable 官方網站：AI Builder](https://thunkable.com/ai)
- [Thunkable 官方網站：方案與 AI token 額度](https://thunkable.com/pricing)
- [Thunkable 官方文件：Web APIs Blocks](https://docs.thunkable.com/blocks/advanced-app-features/web-api)
- [Thunkable 官方文件：Bluetooth Low Energy Blocks](https://docs.thunkable.com/blocks/advanced-app-features/bluetooth-low-energy)
- [Thunkable 官方文件：Objects Blocks](https://docs.thunkable.com/blocks/core-features/objects)
- [Flutter 官方文件：Build for multiple platforms](https://docs.flutter.dev/platform-integration)
- [Blynk 官方文件：Introduction](https://docs.blynk.io/en)
- [Blynk 官方文件：Datastreams](https://docs.blynk.io/en/blynk.console/templates/datastreams)
- [Blynk 官方文件：Supported Hardware](https://docs.blynk.io/en/getting-started/supported-boards)
- [Blynk 官方文件：Blynk Protocol](https://docs.blynk.io/en/blynk-library-firmware-api/blynk-protocol)
- [Blynk 官方文件：Virtual Pins](https://docs.blynk.io/en/blynk-library-firmware-api/virtual-pins)
- [Blynk 官方文件：Device HTTPS API](https://docs.blynk.io/en/blynk.cloud/device-https-api)
- [Blynk 官方文件：Device MQTT API](https://docs.blynk.io/en/blynk.cloud-mqtt-api/device-mqtt-api)
- [MIT App Inventor：Get Started](https://appinventor.mit.edu/explore/get-started)
- [MIT App Inventor：iOS 現況與仍待完成項目](https://appinventor.mit.edu/ios_tips)

## 下一步

接著可進行[選做實作：Thunkable 瀏覽政府開放資料清單](thunkable專題資訊卡介面.md)。你會調整 `page`、`size` 查詢參數，保存 API 回傳的 JSON 清單，並在 Thunkable Live 中選擇顯示其中一筆公開資料；不會連接 BLE、ESP32 或其他服務。
