# PubSubClient：MQTT 函式庫介紹

`PubSubClient` 是讓 Arduino 裝置使用 MQTT 收發訊息的函式庫；本頁用通用方式說明它負責什麼、怎麼依序呼叫，以及讀取訊息時要注意的資料界線。

## 你會學到什麼

- 分辨 network Client（提供網路連線的物件）、`PubSubClient` 與 MQTT Broker（訊息中介服務）各自的工作。
- 看懂從建立物件到持續接收訊息的通用呼叫順序。
- 知道常用函式要傳入什麼資料，以及回傳值或效果代表什麼。
- 理解 callback（回呼函式）的 `topic`（訊息分類名稱）、`payload`（訊息內容）、`length`（內容長度），以及這個函式庫的使用限制。

## 開始前

- 建議先閱讀 [MQTT 基礎：發布、訂閱與 Broker](mqtt基礎.md)，認識發布者、訂閱者、Broker、topic（訊息分類名稱）與 payload（訊息內容）。
- 本頁是函式庫介紹，不需要帳號、網路、Broker、ESP32、額外硬體或安裝函式庫。
- 這是進階實務模擬專題的前導閱讀，不是主線必備條件；不閱讀本頁仍可回到主線教材。

## 它是什麼

`PubSubClient` 不負責讓裝置連上 Wi-Fi。它會透過已準備好的 network Client（網路連線物件）建立與 Broker 的底層連線，再按照 MQTT 規則收發訊息。

```text
network Client ── 提供網路連線 ──> PubSubClient ── 收發 MQTT 訊息 ──> Broker
                                       │
                                       ├─ publish()：送出訊息
                                       └─ callback：收到訂閱訊息後處理
```

network Client 會依連線方式而不同，例如一般網路連線或加密連線；選擇哪一種、如何讓裝置連上網路，並不是 `PubSubClient` 的責任。Broker 負責依 topic 轉送訊息，也不是 `PubSubClient` 內建的一部分。

## 從設定到持續收訊

可以先把一般使用順序記成六個動作：**準備網路連線 → 建立 MQTT 物件 → 指定 Broker → 指定 callback → 連線後訂閱或發布 → 持續呼叫 `loop()`**。

執行位置：Arduino IDE／ESP32  
用途：一般用法節錄，不可獨立執行。
來源：依 PubSubClient 官方 `README.md` 與 `src/PubSubClient.h` 整理的通用示意，非專案原始碼。

```cpp
NetworkClient network_client;
PubSubClient mqtt_client(network_client);

mqtt_client.setServer("broker.example", MQTT_PORT);
mqtt_client.setCallback(on_mqtt_message);

if (mqtt_client.connect("device-name")) {
  mqtt_client.subscribe("example/messages", 0);
  mqtt_client.publish("example/messages", "hello", false);
}

void loop() {
  mqtt_client.loop();
}
```

這段只用來辨認呼叫順序；`NetworkClient`、Broker 位址、裝置名稱與 topic 都是示意名稱，`MQTT_PORT` 則代表程式自行宣告的 Broker 連接埠常數。這些內容不能直接編譯或連線；實際程式還需要先完成網路連線，並依使用環境處理設定與失敗情況。

`loop()` 必須在 Arduino 的 `loop()` 中持續呼叫。否則即使已訂閱 topic，收到的訊息也不會被交給 callback 處理；keepalive（維持連線的定期確認）也無法正常進行。

## 主要函式

| 函式 | 它做什麼 | 常見輸入 | 回傳或效果 |
| --- | --- | --- | --- |
| `PubSubClient(network_client)` | 建立 MQTT 用戶端物件。 | 已建立的 network Client。 | 建立物件，尚未連線。 |
| `setServer(host, port)` | 指定要連線的 Broker 位置。 | 主機名稱與連接埠。 | 記住連線目標，尚未登入。 |
| `setCallback(function)` | 指定收到訂閱訊息時要呼叫的函式。 | callback 函式名稱。 | 之後收到訊息會交給該函式處理。 |
| `connect(id)` 或 `connect(id, user, password)` | 嘗試連上 Broker。 | MQTT Client ID（Broker 用來識別此連線的名稱）；有需要時另加帳號與密碼。 | 成功回傳 `true`，失敗回傳 `false`。 |
| `subscribe(topic, qos)` | 表示想接收某個 topic 的訊息。 | topic 與訂閱 QoS（訊息傳遞保證等級）。 | 成功送出訂閱要求時回傳 `true`。 |
| `publish(topic, payload, retained)` | 送出一筆 MQTT 訊息。 | topic、訊息內容、是否保留最後一筆。 | 成功交給連線傳送流程時回傳 `true`。 |
| `loop()` | 處理網路收發、keepalive 與收到的訊息。 | 不需傳入資料。 | 要在 Arduino 的 `loop()` 持續呼叫。 |
| `connected()` | 查看目前是否仍連線。 | 不需傳入資料。 | 連線中回傳 `true`。 |
| `state()` | 讀取最近一次連線的狀態代號。 | 不需傳入資料。 | 回傳整數，協助縮小連線問題範圍。 |
| `setBufferSize(size)` | 調整可處理的 MQTT 封包緩衝區。 | 位元組數。 | 成功調整時回傳 `true`。 |

`publish()` 回傳 `true`，只表示函式庫已接受這次傳送並交給目前連線處理；不表示 Broker、訂閱者或另一端的程式一定已收到或完成處理。需要確認結果時，應由接收端另外設計可觀察的回覆或狀態。

## 收到訊息時的 callback

當已訂閱的訊息到達時，函式庫會呼叫你用 `setCallback()` 指定的函式。它提供三項資料：

| 資料 | 用途 |
| --- | --- |
| `topic` | 這筆訊息送到的 topic，可用來分辨訊息分類。 |
| `payload` | 收到的原始位元組資料。 |
| `length` | `payload` 實際有多少個位元組。 |

執行位置：Arduino IDE／ESP32  
用途：一般用法節錄，不可獨立執行。
來源：依 PubSubClient 官方 `README.md` 與 `src/PubSubClient.h` 整理的通用示意，非專案原始碼。

```cpp
void on_mqtt_message(char* topic, byte* payload, unsigned int length) {
  for (unsigned int index = 0; index < length; index++) {
    char character = static_cast<char>(payload[index]);
    // 依序處理 character。
  }
}
```

`payload` 是一段原始資料，不保證最後自動加上 C++ 字串使用的結尾字元 `\0`。因此不能直接假設它是可安全列印或比較的文字字串；先依 `length` 決定可讀取的範圍，再轉換、複製或逐一處理資料。若要比較固定文字，也要同時確認長度，避免把前幾個字元相同但後面還有資料的內容誤判為相同訊息。

## 限制與使用範圍

PubSubClient 的功能適合用來理解小型 Arduino MQTT 程式，但使用前要先知道它的界線。

| 項目 | 需要知道的事 |
| --- | --- |
| 維護狀態 | 官方 README 表示此函式庫已停止維護，並建議新專案評估仍在維護的替代方案。它適合理解既有程式，不應直接當成新產品的長期技術選擇。 |
| QoS | 發布只能使用 QoS `0`；訂閱可要求 QoS `0` 或 `1`。QoS `0` 的訊息可能遺失，QoS `1` 的訊息可能重複。 |
| MQTT 版本 | 預設使用 MQTT `3.1.1`。若 Broker 或其他程式要求不同版本，不能只靠修改 topic 解決。 |
| keepalive | 預設為 15 秒，用來確認連線是否仍存在。持續呼叫 `loop()` 是它能運作的必要條件。 |
| 封包大小 | 預設最大 MQTT 封包大小（含標頭）為 `256` bytes。topic、payload 與協定資料都會占用這個空間。 |
| 緩衝區 | 可用 `setBufferSize()` 調大緩衝區，但會使用更多記憶體，也不能解決網路、帳密、權限或資料格式問題。 |

收到的資料接近或超過緩衝區範圍時，先縮小可控制的測試資料並確認 topic 與 payload 的實際大小。不要只是不斷調大緩衝區，也不要把未檢查的外部資料當成安全字串使用。

## 常見誤解

### 已經呼叫 `subscribe()`，為什麼還要 `loop()`？

`subscribe()` 是送出訂閱要求；`loop()` 才會持續處理連線上收到的資料，並呼叫 callback。少了 `loop()`，程式沒有持續取回與處理訊息的機會。

### `publish()` 回傳成功，代表另一端一定收到嗎？

不代表。它沒有證明 Broker 已轉送、另一端仍在線上，或另一端已完成動作。MQTT 的 QoS 與接收端程式處理方式都會影響結果。

### 訊息太長時，只要呼叫 `setBufferSize()` 就好了嗎？

不一定。它只調整函式庫可用的記憶體空間。先確認資料是否真的必須那麼長、記憶體是否足夠，以及接收程式是否依 `length` 安全讀取；若不確定，先停止擴大資料量。

## 理解確認

- 能分辨 network Client 提供連線、`PubSubClient` 收發 MQTT、Broker 轉送訊息的工作。
- 能依序辨認 `setServer()`、`setCallback()`、`connect()`、`subscribe()`、`publish()` 與 `loop()` 的角色。
- 能理解 callback 為什麼同時提供 `payload` 與 `length`。
- 知道 `publish()` 的成功回傳不等於另一端已完成處理。
- 知道 PubSubClient 的 QoS、封包大小、keepalive 與停止維護等限制。

## 重點整理

- PubSubClient 透過既有的 network Client，讓 Arduino 裝置使用 MQTT 發布與訂閱訊息。
- `loop()` 與 callback 是持續接收、處理訊息的關鍵；不能只完成連線與訂閱。
- `payload` 不是保證結尾的字串，應依 `length` 決定安全的讀取範圍。
- 函式庫有 QoS、緩衝區與維護狀態的限制；了解這些限制後，才能判斷它是否適合目前用途。

## 官方來源

- [PubSubClient README](https://github.com/knolleary/pubsubclient/blob/master/README.md)
- [PubSubClient API 文件](https://pubsubclient.knolleary.net/)
- [PubSubClient 原始碼](https://github.com/knolleary/pubsubclient/tree/master/src)

## 下一步

- 想建立受限的雲端 MQTT 測試環境，可閱讀[HiveMQ Cloud 基本設定](hivemq-cloud-基本設定.md)。
- 想看通用函式如何放進一個已完成的選做情境，可閱讀[PubSubClient：兩個 Topic 的控制與回傳實作](pubsubclient雙topic控制與回傳.md)。
- 或回到[前導技術概要](index.md)，選擇其他工具閱讀。
