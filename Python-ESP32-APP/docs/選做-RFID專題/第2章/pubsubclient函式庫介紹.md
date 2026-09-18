# PubSubClient：MQTT 函式庫介紹

PubSubClient 是 Arduino 常用的 MQTT Client 函式庫；第 2 章用它讓 ESP32 發布固定假事件，並訂閱相符回覆。

## 你會學到什麼

- 說明 PubSubClient 與 Wi-Fi、TLS Client、MQTT Broker 的分工。
- 看懂建立連線、訂閱、發布、接收與持續處理的順序。
- 認識第 2 章用到的主要函式與回傳結果。
- 知道本函式庫的訊息大小、QoS 與維護狀態限制。

## 開始前

- 已完成[第 2 章主頁](index.md)的固定假資料測試。
- 本頁是函式庫介紹，不需要新增帳密、topic、硬體或公開連線。

## 它在第 2 章的位置

```text
WiFiClientSecure ── 提供 TLS 網路連線 ──→ PubSubClient ──→ MQTT Broker
                                               │
                                               ├→ publish()：送出 card_read
                                               └→ callback：收到回覆後處理
```

PubSubClient 不負責讓 ESP32 連上 Wi-Fi，也不會自己檢查 RFID 卡片。它使用已建立的網路 Client，依 MQTT 規則和 Broker 交換訊息。

第 2 章使用 `WiFiClientSecure`，所以 MQTT 資料走 TLS 連線；是否驗證伺服器身分，則由 `WiFiClientSecure` 的設定決定，不是 PubSubClient 自己決定。

## 先看整體使用順序

```cpp
WiFiClientSecure secure_client;
PubSubClient mqtt_client(secure_client);

secure_client.setInsecure();
mqtt_client.setServer(MQTT_HOST, MQTT_PORT);
mqtt_client.setCallback(on_mqtt_message);

if (mqtt_client.connect(MQTT_CLIENT_ID, MQTT_USERNAME, MQTT_PASSWORD)) {
  mqtt_client.subscribe(MQTT_TOPIC, 0);
  mqtt_client.publish(MQTT_TOPIC, TEST_EVENT, false);
}

void loop() {
  mqtt_client.loop();
}
```

可以記成六個動作：**準備網路 Client → 建立 PubSubClient → 指定 Broker → 指定收到訊息的處理函式 → 連線、訂閱與發布 → 持續呼叫 `loop()`**。

如果少了最後的 `loop()`，ESP32 即使已訂閱，也不會把收到的訊息交給 `on_mqtt_message()`。

## 第 2 章主要函式

| 函式 | 最常見用法 | 傳入什麼 | 回傳或效果 |
| --- | --- | --- | --- |
| `PubSubClient(network_client)` | 建立 MQTT 物件 | 已可使用的網路 Client，例如 `WiFiClientSecure` | 建立物件；尚未連線。 |
| `setServer(host, port)` | 指定 Broker 位置 | 主機名稱與連接埠 | 儲存連線目標，尚未登入。 |
| `setCallback(function)` | 指定收訊處理方式 | 一個函式名稱 | 收到訂閱訊息後，函式庫會呼叫該函式。 |
| `connect(id, user, password)` | 登入 Broker | 裝置代號、帳號、密碼 | 成功回傳 `true`；失敗回傳 `false`。 |
| `subscribe(topic, qos)` | 訂閱一個 topic | 完整 topic 與 QoS | 成功回傳 `true`。 |
| `publish(topic, payload, retained)` | 送出一筆訊息 | 完整 topic、文字、是否保留 | 成功加入傳送流程時回傳 `true`。 |
| `loop()` | 處理網路收發 | 不需傳入資料 | 必須在 ESP32 `loop()` 持續呼叫。 |
| `connected()` | 查看目前連線 | 不需傳入資料 | 連線中為 `true`。 |
| `state()` | 讀取失敗代號 | 不需傳入資料 | 回傳整數，協助判斷上次連線失敗原因。 |
| `setBufferSize(size)` | 調整訊息緩衝區 | 位元組數 | 成功回傳 `true`；第 2 章設為 `512`。 |

`publish()` 回傳 `true` 不等於另一端已經看到資料。第 2 章以另一端送回、且 `event_id` 相符的回覆作為雙向流程完成的確認。

## 收到訊息時的 callback

```cpp
void on_mqtt_message(char* topic, byte* payload, unsigned int length) {
  String message;
  message.reserve(length);

  for (unsigned int index = 0; index < length; index++) {
    message += static_cast<char>(payload[index]);
  }

  // 再檢查 event_id 和 status
}
```

callback 的三個資料是：

| 資料 | 意思 |
| --- | --- |
| `topic` | 訊息送到哪一個 topic。第 2 章只允許自己的完整 topic。 |
| `payload` | 收到的原始位元組資料，不保證最後有文字結尾符號。 |
| `length` | `payload` 的實際長度，因此程式用迴圈逐一讀取。 |

第 2 章先略過帶有 `event_type` 的自身事件，再確認回覆帶有本次 `TEST_EVENT_ID` 和 `status`。只檢查 `status` 不夠，因為其他事件的回覆也可能在同一個 topic 出現。

## `state()` 可怎麼看

```cpp
Serial.printf("MQTT 連線失敗，狀態 %d\n", mqtt_client.state());
```

常見代號如下；它們只協助縮小問題範圍，不能取代確認帳密和權限。

| 代號 | 可先理解為 |
| --- | --- |
| `0` | 目前已連線。 |
| `-4` | 連線等待逾時。 |
| `-3` | 原本的連線已中斷。 |
| `-2` | 無法建立網路連線。 |
| `4` | 帳號或密碼不正確。 |
| `5` | 帳號沒有被允許連線。 |

第 2 章只嘗試有限次。看到失敗時，先檢查受控 Wi-Fi、主機名稱、TLS 連接埠、專屬帳密與完整 topic 權限；不要用無限重連掩蓋問題。

## 函式庫限制與本章做法

PubSubClient 的官方 README 說明：它只能發布 QoS `0`，可訂閱 QoS `0` 或 `1`；預設最大封包大小是 `256` bytes，也可用 `setBufferSize()` 調整。第 2 章只傳很短的固定 JSON，使用 QoS `0` 與 `retain: false`，並將緩衝區設為 `512`。這能讓你觀察資料流，但不代表每筆訊息都保證送達或保存。 [官方 README](https://github.com/knolleary/pubsubclient)

!!! warning "目前維護狀態"

    2026 年的官方 README 已標示 PubSubClient 不再維護，並建議新的專案選擇仍在維護的替代函式庫。本課第 2 章保留它，是因為既有的 NodeMCU-32S 固定假資料流程已用 `2.8.0` 實測；不要把這頁當成新產品的長期技術選型建議。 [官方 README](https://github.com/knolleary/pubsubclient)

## 何時使用、何時不要使用

| 情境 | 建議 |
| --- | --- |
| 想理解小型 ESP32 如何發布與訂閱固定假資料 | 可依第 2 章使用已驗證版本。 |
| 訊息可能超過預設緩衝區 | 先量測訊息大小，再理解 `setBufferSize()`；不要盲目調大。 |
| 需要保證送達、離線保存或完整重連策略 | 不要只靠本章範例；需要另外設計與驗證。 |
| 要開始新的長期專案 | 先評估仍維護的 MQTT 函式庫與你的安全、更新需求。 |

## 完成時，應能確認

- 能按順序說出 `setServer()`、`setCallback()`、`connect()`、`subscribe()`、`publish()` 與 `loop()` 的角色。
- 能說出 callback 為何需要 `payload` 與 `length`。
- 能解釋為什麼回覆必須比對本次 `event_id`。
- 能知道 PubSubClient 的 QoS、訊息大小與維護狀態限制。

## 常見問題

### 為什麼已經 `subscribe()`，卻沒有收到回覆？

確認 `mqtt_client.loop()` 持續在 ESP32 的 `loop()` 執行，也確認回覆使用同一個完整 topic、當前事件代號與 `status` 欄位。

### 為什麼直接增加 `setBufferSize()` 不是第一個修正方式？

緩衝區只處理訊息大小問題，不能修正帳密、topic 權限、Wi-Fi、TLS 或回覆格式。先看序列輸出的連線狀態與訊息內容是否符合本章固定格式。

## 重點整理

- PubSubClient 讓 Arduino 裝置以發布／訂閱方式交換 MQTT 訊息。
- `loop()` 是接收訊息的必要工作；callback 是訊息到達後的處理位置。
- 第 2 章以固定資料、單一專屬 topic、QoS `0` 與短期帳密縮小練習範圍。
- 函式庫目前不再維護；本頁介紹已驗證課程用法，不等於推薦新專案採用。

## 官方來源

- [PubSubClient GitHub README 與限制](https://github.com/knolleary/pubsubclient)
- [官方 `mqtt_esp8266` 範例](https://github.com/knolleary/pubsubclient/tree/master/examples/mqtt_esp8266)
- [官方 API 文件](https://pubsubclient.knolleary.net/)

## 下一步

回到[第 2 章主頁](index.md)，或閱讀[ESP32 MQTT 程式導讀](mqtt程式導讀.md)。
