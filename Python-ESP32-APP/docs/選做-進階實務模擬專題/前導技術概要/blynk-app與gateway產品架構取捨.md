# 延伸選讀：Blynk App 與 Gateway 的產品架構取捨

產品團隊選擇手機 App 與雲端方案時，不只比較畫面怎麼做，也要決定哪些功能交給平台、哪些功能由自己的後端負責。

## 你會了解什麼

- 理解代管 IoT 平台與自建 Gateway／API 各自承擔哪些工作。
- 看懂選擇 Blynk App 後，Blynk.Cloud 為什麼會成為資料流的一部分。
- 分辨平台依賴落在 ESP32 或 Gateway 時，會影響哪些程式與維運工作。
- 知道產品選型要一起衡量開發速度、現成功能、客製需求、維運責任與替換成本。

## 開始前

- 本頁是產品架構的概念比較，不需要建立 Blynk 或 Thunkable 帳號，也不提供雲端設定與程式操作。
- 建議先閱讀[Thunkable：App 與 Web API 的概念](thunkable與web-api概念.md)，知道 App、Gateway 與 ESP32 在資料流中的位置。
- 雲端方案、額度、功能與價格可能調整。實際產品選型時，仍須依當時的官方方案、資料所在地、法規與組織需求重新確認。

## 產品團隊實際在選什麼？

選擇 Blynk 這類代管 IoT 平台，通常不只是取得一個手機畫面。平台也可能提供裝置管理、使用者與權限、儀表板、通知、自動化及裝置資料通道。團隊可以較快組合出可用功能，但裝置、後端與 App 也要配合平台定義的帳號、服務入口和資料模型。

另一種方向是建立自己的 Gateway 或後端 API，再讓自行開發或低程式碼 App 呼叫它。團隊能自行定義 API 與產品流程，也比較容易同時服務手機 App、網頁或其他系統；相對地，登入、權限、監控、備份、錯誤處理與服務可用性都需要投入開發和維運。

這兩種方向沒有固定的優劣。真正的問題是：**哪些能力值得直接採用平台，哪些能力需要由團隊掌握？**

## 選擇 Blynk App 後的資料流

Blynk App 的畫面元件會使用 Blynk.Cloud 中的 Datastream（平台用來保存與交換裝置資料的通道）。因此，若產品選擇 Blynk App，資料必須進入 Blynk.Cloud，不能像一般 Web API 用戶端一樣，直接把本專題的 Gateway HTTP／JSON API 當成資料來源。

### 路徑一：ESP32 直接連到 Blynk.Cloud

以下以裝置資料上傳至 App 為例：

```text
ESP32 → Blynk.Cloud → Blynk App
```

常見做法是讓 ESP32 使用 Blynk Library、Virtual Pin（平台使用的非實體資料編號）與 Datastream。這條路徑能快速把裝置資料接到儀表板與控制元件，但原有 Gateway 不再是手機端資料流的必要部分。

若 Blynk App 向裝置送出控制值，資料仍會反向經過 Blynk.Cloud，再傳給 ESP32。

這時平台依賴主要落在 ESP32：若日後更換雲端或 App 方案，裝置程式中的函式庫、連線設定與資料模型可能需要一起調整。

### 路徑二：保留 Gateway，再串接 Blynk.Cloud

以下同樣以裝置資料上傳至 App 為例：

```text
ESP32 → Gateway → Blynk.Cloud → Blynk App
```

Gateway 可以使用 Blynk 提供的 HTTPS 或 MQTT API，把資料送進 Blynk.Cloud。ESP32 不一定要改用 Blynk Library，但 Gateway 需要處理 Blynk.Cloud 的服務入口、裝置 token（用來辨認並授權裝置的秘密字串）、Datastream 或 MQTT topic。

若 Blynk App 向裝置送出控制值，資料也要反向經過 Blynk.Cloud 與 Gateway，才能到達 ESP32。

這時平台依賴主要落在 Gateway。它能隔開 ESP32 與平台函式庫，卻增加一段資料轉送與平台整合；團隊也要處理同步失敗、重送、監控及兩邊資料格式的對應。

## 使用標準協議不等於沒有平台依賴

HTTPS 與 MQTT 是許多工具都能使用的共通通訊方式，但通訊方式只是整體設計的一部分。例如 Blynk MQTT API 仍使用 Blynk.Cloud 的連線位置、裝置驗證、Datastream 名稱，以及 `ds/...`、`downlink/...` 等 topic 規則。

因此，判斷替換成本時，不能只問「有沒有使用標準協議」，還要檢查：

- 服務網址與連線位置由誰提供。
- 身分驗證與權限由誰管理。
- 欄位、Datastream、topic 與資料型別由誰定義。
- App 的畫面與功能是否只能讀取特定平台中的資料。
- 更換服務時，ESP32、Gateway 與 App 哪些部分要一起修改。

## 業界常見的取捨面向

| 取捨面向 | 採用代管 IoT 平台 | 自建 Gateway／API |
| --- | --- | --- |
| 推出初版的速度 | 可使用既有的 App、裝置管理、儀表板與通知能力，通常較快組合出初版。 | 要先完成 API、帳號、權限、監控與部署，初期工作通常較多。 |
| 客製產品流程 | 需在平台提供的資料模型、元件與方案範圍內設計；是否足夠取決於產品需求。 | 能依自己的使用流程與既有系統設計 API，但每項客製功能都需要開發與測試。 |
| 團隊能力 | 適合希望減少自建 App、裝置平台及雲端維運工作的團隊。 | 需要後端、App、安全、部署與維運能力，或能長期合作的外部團隊。 |
| 維運責任 | 平台負責部分基礎服務；團隊仍要管理裝置程式、資料正確性、權限設定與平台異常時的處理。 | 團隊能掌握更多技術細節，也要自行處理可用性、更新、備份、監控與事故應變。 |
| 整合既有系統 | 需確認平台 API、Webhook 或其他整合方式是否符合企業現有流程。 | 可依企業資料庫、身分系統或內部流程設計介面，但整合成本由團隊承擔。 |
| 成本 | 可能包含方案、使用者、裝置、流量或進階功能費用，須依當時方案估算。 | 不一定比較便宜；人力、主機、監控、維護與資安工作都屬於成本。 |
| 替換彈性 | 使用越多平台專有功能，未來遷移時需要重新處理的範圍通常越大。 | 若 API 邊界清楚，App 與裝置較能分開替換；但自訂設計本身仍會形成需要維護的依賴。 |
| 安全與資料治理 | 要確認平台的權限、資料保存、服務地區及合約是否符合組織要求。 | 能自行設計資料與權限邊界，但必須自行證明並持續維護其安全性。 |

## 哪些情況可能適合代管 IoT 平台？

以下情況可以優先評估 Blynk 或其他代管平台：

- 需要快速確認裝置連線、儀表板與遠端控制是否具有產品價值。
- 需求接近平台已提供的裝置、使用者、通知或自動化功能。
- 團隊目前沒有足夠人力自行建立並維護完整 App 與 IoT 後端。
- 團隊已確認方案限制、資料治理與未來遷移成本可以接受。

這不代表產品永遠不能更換架構。較穩妥的做法是先標出哪些程式和資料依賴平台，並在每個版本重新評估這些依賴是否仍符合產品需求。

## 哪些情況可能適合自建 Gateway／API？

以下情況可以優先評估由自己的 Gateway 或後端提供 API：

- 產品有平台元件難以表達的專屬流程、規則或權限模型。
- 同一份資料需要同時提供給手機 App、網頁、報表或企業內部系統。
- 已有身分驗證、資料庫或營運系統需要整合。
- 組織對資料位置、稽核、合約或服務控制有明確要求。
- 團隊有能力長期負責 API、安全更新、監控、備份與事故處理。

自建不等於完全不依賴外部服務，也不保證較安全或較省錢。雲端主機、通知、登入、監控和套件仍可能來自不同供應商；重點是清楚知道每項依賴由誰負責。

## 本教材為什麼使用 Gateway API 與 Thunkable？

本教材希望你先看懂 ESP32、MQTT、Gateway、HTTP／JSON API 與 App 之間的分工。Thunkable 可以直接使用 Gateway Web API，因此不需要為了手機畫面改變 ESP32 與 Gateway 原有的通訊方式。

這是一項配合學習目標的架構選擇，不表示業界產品都應採用相同方案。真實產品若需要快速取得裝置管理、使用者權限、通知或自動化功能，採用 Blynk 等代管平台可能更符合成本與時程；若需要高度客製與既有系統整合，自建 API 可能更合適。

## 常見誤解

### 平台依賴一定是不好的嗎？

不一定。用可接受的替換成本換取較快上市、較少維運工作或成熟的現成功能，可能是合理的產品決策。問題不是「有沒有依賴」，而是團隊是否知道依賴範圍並願意承擔後續成本。

### 自建 Gateway 就一定比較專業嗎？

不一定。若團隊沒有足夠能力維護帳號、權限、監控、備份與安全更新，自建服務反而可能增加風險。產品方案應配合團隊能力，而不是只看能否自行撰寫程式。

### 先使用平台，之後再搬走會很容易嗎？

不一定。若裝置程式、資料模型、權限、通知與 App 畫面都使用平台專有功能，遷移時可能需要同時修改多個部分。原型階段也應記錄平台依賴，避免把短期方便誤當成沒有替換成本。

## 理解確認

- 能理解 Blynk App 需要透過 Blynk.Cloud 使用 Datastream。
- 能分辨 ESP32 直連 Blynk.Cloud，與 Gateway 再串接 Blynk.Cloud 的依賴落點。
- 能理解代管平台可縮短部分開發時間，但會帶入平台介面、方案與替換成本。
- 能理解自建 Gateway／API 提供較多控制空間，也增加開發與維運責任。
- 能依產品需求、團隊能力與可接受成本判斷方案，而不是把其中一種架構視為固定答案。

## 重點整理

- 選擇手機 App 方案可能連帶改變裝置、Gateway 與雲端之間的資料流。
- 採用 Blynk App 時，Blynk.Cloud 會成為資料流的一部分；平台依賴可以落在 ESP32，也可以移到 Gateway。
- 代管平台的價值在於現成功能與較快整合；代價可能是方案限制與較高的替換成本。
- 自建 Gateway／API 能配合專屬流程與既有系統；團隊也要承擔更多安全、部署與維運工作。
- 產品選型沒有單一正解，應從需求、時程、團隊能力、成本、資料治理與未來變更一起判斷。

## 參考資料

- [Blynk 官方文件：Blynk App Overview](https://docs.blynk.io/en/blynk.apps/overview)
- [Blynk 官方文件：Blynk Console Overview](https://docs.blynk.io/en/blynk.console/console-overview)
- [Blynk 官方文件：Datastreams](https://docs.blynk.io/en/blynk.console/templates/datastreams)
- [Blynk 官方文件：Users](https://docs.blynk.io/en/concepts/users)
- [Blynk 官方文件：Organizations](https://docs.blynk.io/en/concepts/organizations)
- [Blynk 官方文件：Automations](https://docs.blynk.io/en/concepts/automations)
- [Blynk 官方文件：Device HTTPS API](https://docs.blynk.io/en/blynk.cloud/device-https-api)
- [Blynk 官方文件：Device MQTT API](https://docs.blynk.io/en/blynk.cloud-mqtt-api/device-mqtt-api)
- [Thunkable 官方文件：Web APIs Blocks](https://docs.thunkable.com/blocks/advanced-app-features/web-api)

## 下一步

- 回到[Thunkable：App 與 Web API 的概念](thunkable與web-api概念.md)，繼續理解 App 如何讀取 Gateway API。
- 若想用通用方式理解元件之間的依賴，可閱讀[資訊補充：什麼是系統耦合性？](../../資訊補充-系統耦合性與替換彈性.md)。
