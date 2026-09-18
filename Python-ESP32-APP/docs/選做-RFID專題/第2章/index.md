# 選做專題第 2 章：ESP32 MQTT 假信封事件

讓 ESP32 將固定的假信封事件交給 MQTT 中介服務，再接收同一事件的練習狀態回覆。

!!! warning "選做實作與安全限制"

    本章需要可上網的 Wi-Fi、HiveMQ Cloud 帳號與可刪除的測試帳密。程式使用 `setInsecure()`，資料傳輸仍會加密，但 ESP32 不會確認伺服器身分。只可傳送本章固定的假信封與短期測試帳密；真實 UID、個人資料、Wi-Fi 資訊、控制訊息與出入紀錄都不可傳送。

## 你會學到什麼

- 讓 ESP32 使用 MQTT 發布第 0 章約定的假 `uid_envelope` 事件。
- 以 `event_id` 對應事件與練習狀態回覆。
- 知道 MQTT 中介服務只負責轉送，黑白名單判斷要由後續 Gateway（集中處理資料的程式）處理。
- 在帳密、訊息或 Wi-Fi 不符合條件時，看懂序列監控的安全停止訊息。

## 開始前

先完成[第 0 章：準備、安全與資料契約](../第0章/index.md)與[第 1 章：RC522 RFID 與 SPI](../第1章/index.md)，並準備明確獲授權的 Wi-Fi、HiveMQ Cloud Serverless（雲端代管）測試叢集。

在 Arduino IDE 的「開發板管理員」與「程式庫管理員」確認下列版本：

| 項目 | 固定版本／選項 |
| --- | --- |
| ESP32 開發板核心 | `esp32 by Espressif Systems 3.3.11` |
| 開發板 | `NodeMCU-32S` |
| `PubSubClient` | `2.8.0`（Nick O'Leary） |

MQTT 用 topic（訊息地址）區分不同訊息；只有訂閱相同地址的裝置才會收到訊息。本章不接 RC522，只用假信封確認資料路徑；真實 UID 不可放進 MQTT 資料流。

## 成功的樣子

序列監控設定為 `115200` 後，會依序看到類似下列內容：

```text
Wi-Fi 已連線。
MQTT TLS 已連線，但未驗證伺服器身分。
已訂閱測試 topic。
本次事件：evt-XXXXXXXX
已送出固定假信封，請由測試用戶端送回 whitelisted 狀態。
已收到本次事件的 whitelisted 狀態。
```

`evt-XXXXXXXX` 每次重新啟動可能不同；它只是事件取件號碼，不是卡片資料。整個過程不會出現 UID。

## 先看資料流與責任

```text
ESP32 ──固定假信封事件──> HiveMQ Cloud ──> 測試用戶端
ESP32 <──練習狀態回覆──── HiveMQ Cloud <── 測試用戶端

下一章：Gateway（集中處理資料的程式）取代測試用戶端，集中判斷名單與回覆狀態
```

HiveMQ Cloud 像代收轉交訊息的櫃檯：它負責把訊息送到訂閱者，不應被當成可保存真實 UID 的可信位置。ESP32 只傳遞事件，不保存黑白名單，也不把任何狀態轉成開門或解鎖動作。

!!! warning "同一個 topic（訊息地址）不是方向隔離"

    Serverless 測試使用同一個受限 topic 來送出事件與接收回覆。ESP32 會收到自己送出的事件，因此程式會略過帶有 `uid_envelope` 的內容，再確認回覆是否帶有本次 `event_id` 與 `whitelisted`（在白名單）狀態。這是練習用的辨識方式，不是正式安全隔離。

## 步驟 1：建立受限的測試帳密

**目的：** 準備 ESP32 與另一個 MQTT 測試用戶端的短期連線資料。

依[HiveMQ Cloud 測試設定](hivemq-cloud-測試設定.md)建立兩組不同、可刪除的測試帳密，兩組都只允許使用一個不含個人資料的完整 topic，例如：

```text
YOUR_NAMESPACE/devices/demo-esp32-01/messages
```

預期結果：你有主機名稱、TLS 連接埠、兩組不同測試帳密與一個受限 topic。這些資料只留在自己的電腦，不貼到對話、截圖、公開網站或版本控制。

## 步驟 2：建立本機設定檔

**目的：** 讓 Wi-Fi 與 MQTT 測試資料和主程式分開保存。

在第 0 章的 `rfid-project` 資料夾中建立下列 Arduino 草稿位置：

```text
rfid-project/
└── esp32/
    └── rfid_mqtt_test/
        ├── rfid_mqtt_test.ino
        └── secrets.h
```

執行位置：Arduino IDE／ESP32
檔案：`rfid-project/esp32/rfid_mqtt_test/secrets.h`

```cpp
#pragma once

// 這些資料只留在自己的電腦，不要公開、截圖或分享。
const char* WIFI_SSID = "你的 Wi-Fi 名稱";
const char* WIFI_PASSWORD = "你的 Wi-Fi 密碼";

const char* MQTT_HOST = "你的叢集主機名稱";
const uint16_t MQTT_PORT = 8883;
const char* MQTT_USERNAME = "ESP32 測試帳號";
const char* MQTT_PASSWORD = "ESP32 測試密碼";

const char* MQTT_TOPIC = "YOUR_NAMESPACE/devices/demo-esp32-01/messages";
```

預期結果：`secrets.h` 和 `rfid_mqtt_test.ino` 放在同一個草稿資料夾，Arduino IDE 可以一併開啟；檔案內只有你自己的測試資料，且不會公開傳送或發布。

## 步驟 3：上傳固定假信封程式

**目的：** 連上 MQTT、發布假信封事件，並只接受本次事件的 `whitelisted` 練習回覆。

執行位置：Arduino IDE／ESP32
檔案：`rfid-project/esp32/rfid_mqtt_test/rfid_mqtt_test.ino`

```cpp
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <PubSubClient.h>
#include <esp_system.h>

#include "secrets.h"

constexpr char DEVICE_ID[] = "demo-esp32-01";
constexpr uint16_t MQTT_BUFFER_SIZE = 512;

WiFiClientSecure secure_client;
PubSubClient mqtt_client(secure_client);

char current_event_id[20];
char test_event[320];


bool make_test_event() {
  // 每次啟動建立新的事件取件號碼，不使用卡片資料。
  unsigned long random_value = static_cast<unsigned long>(esp_random());
  snprintf(current_event_id, sizeof(current_event_id), "evt-%08lX", random_value);

  // 信封內每個字串都是固定假資料，不是加密後的真實 UID。
  int length = snprintf(
      test_event,
      sizeof(test_event),
      "{\"event_id\":\"%s\",\"device_id\":\"%s\","
      "\"occurred_at\":\"2030-01-01T08:00:00+00:00\","
      "\"uid_envelope\":{\"key_id\":\"test-key\","
      "\"nonce\":\"fake-nonce\",\"ciphertext\":\"practice-allowed\","
      "\"tag\":\"fake-tag\"}}",
      current_event_id,
      DEVICE_ID);

  return length > 0 && length < static_cast<int>(sizeof(test_event));
}


bool connect_wifi() {
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  for (uint8_t attempt = 1; attempt <= 40; attempt++) {
    if (WiFi.status() == WL_CONNECTED) {
      Serial.println("Wi-Fi 已連線。");
      return true;
    }
    delay(500);
  }

  Serial.println("Wi-Fi 連線逾時，停止 MQTT 測試。");
  return false;
}


void on_message(char* topic, byte* payload, unsigned int length) {
  String message;
  message.reserve(length);

  for (unsigned int index = 0; index < length; index++) {
    message += static_cast<char>(payload[index]);
  }

  // 同一個 topic 會收到自身事件；假信封不是狀態回覆。
  if (message.indexOf("\"uid_envelope\"") >= 0) {
    Serial.println("收到假信封事件，略過。");
    return;
  }

  String expected_event = "\"event_id\":\"" + String(current_event_id) + "\"";
  if (message.indexOf(expected_event) >= 0 &&
      message.indexOf("\"status\":\"whitelisted\"") >= 0) {
    Serial.println("已收到本次事件的 whitelisted 狀態。");
    return;
  }

  Serial.println("收到不符合本次測試規則的訊息，已略過。");
}


bool connect_mqtt() {
  for (uint8_t attempt = 1; attempt <= 3; attempt++) {
    if (mqtt_client.connect(DEVICE_ID, MQTT_USERNAME, MQTT_PASSWORD)) {
      Serial.println("MQTT TLS 已連線，但未驗證伺服器身分。");

      if (mqtt_client.subscribe(MQTT_TOPIC, 0)) {
        Serial.println("已訂閱測試 topic。");
        return true;
      }

      Serial.println("訂閱失敗。");
      mqtt_client.disconnect();
    } else {
      Serial.printf("MQTT 連線失敗（第 %u 次，狀態 %d）。\n", attempt, mqtt_client.state());
    }

    delay(2000);
  }

  Serial.println("MQTT 無法連線，停止自動重試。");
  return false;
}


void setup() {
  Serial.begin(115200);

  // 本章只使用固定假資料；TLS 不會驗證伺服器身分。
  secure_client.setInsecure();
  mqtt_client.setServer(MQTT_HOST, MQTT_PORT);
  mqtt_client.setCallback(on_message);
  mqtt_client.setBufferSize(MQTT_BUFFER_SIZE);

  if (!make_test_event() || !connect_wifi() || !connect_mqtt()) {
    return;
  }

  Serial.print("本次事件：");
  Serial.println(current_event_id);

  if (mqtt_client.publish(MQTT_TOPIC, test_event, false)) {
    Serial.println("已送出固定假信封，請由測試用戶端送回 whitelisted 狀態。");
  } else {
    Serial.println("固定假信封送出失敗。");
  }
}


void loop() {
  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("Wi-Fi 已中斷，目前狀態未知；程式不會自動無限重連。");
    delay(5000);
    return;
  }

  if (mqtt_client.connected()) {
    mqtt_client.loop();
  }
}
```

`make_test_event()` 只建立假事件與事件取件號碼；`setBufferSize(512)` 預留 `512` 位元組（bytes）空間容納假信封和 topic。`on_message()` 先略過自身假信封，再確認回覆的 `event_id` 與 `whitelisted`（在白名單）。這些比對只服務本章固定測試，並不是完整 JSON 檢查或名單判斷。

預期結果：開啟 `115200` 的序列監控後，會顯示 Wi-Fi、MQTT、訂閱與本次事件代號。

## 步驟 4：由測試用戶端送回練習狀態

**目的：** 確認 ESP32 只接受與本次事件相關的固定練習回覆。

1. 使用第二組測試帳密登入 HiveMQ Cloud 的測試工具。
2. 先訂閱和 ESP32 完全相同的 `MQTT_TOPIC`。
3. 從序列監控複製「本次事件」後面的 `evt-XXXXXXXX`。
4. 將下列 JSON 的 `evt-XXXXXXXX` 改為這次看到的事件代號，發布到同一個 topic：

   ```json
   {"event_id":"evt-XXXXXXXX","status":"whitelisted"}
   ```

預期結果：ESP32 顯示 `已收到本次事件的 whitelisted 狀態。` 這只表示假信封與練習回覆可以往返；它不代表卡片已被授權，也不會控制任何設備。

## 步驟 5：觀察不相符的訊息與安全收尾

**目的：** 確認 ESP32 不會把其他訊息當成這次回覆，並讓測試帳密保持短期使用。

在測試用戶端發布下列任一內容到相同 topic：

```text
not-json
```

```json
{"event_id":"evt-other","status":"whitelisted"}
```

```json
{"event_id":"evt-XXXXXXXX","status":"blacklisted"}
```

預期結果：ESP32 顯示 `收到不符合本次測試規則的訊息，已略過。`

測試結束後，在 HiveMQ Cloud 刪除測試帳密。帳密異動可能需要一段時間才生效；等待後重新啟動 ESP32，序列監控應顯示有限次 MQTT 連線失敗，最後停止自動重試。若再次測試，建立新的短期帳密並只更新自己的 `secrets.h`。

## 完成時，應能確認

- ESP32 只送出 `event_id`、`device_id`、含時區時間與假 `uid_envelope`。
- 假信封沒有真實 UID，ESP32 也沒有黑白名單或實體控制程式。
- 回覆的 `event_id` 與本次事件不相同，或狀態不是 `whitelisted` 時，ESP32 會略過。
- Wi-Fi 中斷或 MQTT 無法連線時，畫面不會沿用前一筆狀態。
- 測試帳密、主機名稱、完整 topic 與 Wi-Fi 資訊沒有公開傳送或發布。

## 常見問題

### `PubSubClient.h` 找不到

回到 Arduino IDE 的「程式庫管理員」，確認安裝的是 Nick O'Leary 的 `PubSubClient 2.8.0`。想了解 `connect()`、`publish()`、`subscribe()` 與 `loop()` 的角色，可閱讀[PubSubClient 函式庫介紹](pubsubclient函式庫介紹.md)。

### 收到「不符合本次測試規則」

確認回覆的 `event_id` 完全等於序列監控印出的本次事件代號，且 `status` 是 `whitelisted`（在白名單）。這個測試只接受固定欄位順序與固定狀態；不要把它當成完整 JSON 格式檢查。

### MQTT 無法連線

先確認主機名稱只填網域名稱，沒有加入 `mqtts://`、`https://` 或其他路徑；再確認 TLS 連接埠、測試帳密和完整 topic 權限。帳密剛建立、更新或刪除時，等待服務方生效後再重新啟動 ESP32。

### 為什麼不用真實 UID 測試？

HiveMQ Cloud 是中介服務，不是用來保存卡片資料的可信位置。第 2 章只確認假信封能往返；真實 UID 不能直接放進 MQTT。

## 重點整理

- ESP32 與測試用戶端只用 MQTT 轉送固定假信封與固定練習狀態。
- `event_id` 像取件號碼，用來確認回覆屬於本次事件。
- `setBufferSize(512)` 用來容納假信封；QoS `0` 是盡力傳送，`retain: false` 是不保留舊訊息，兩者都不保證送達或保存。
- `setInsecure()` 不驗證伺服器身分；只使用短期帳密、假資料與可控網路。

## 下一步

- [ESP32 MQTT 程式導讀](mqtt程式導讀.md)
- [PubSubClient 函式庫介紹](pubsubclient函式庫介紹.md)
- [專題第 3 章：Gateway 名單與最小紀錄](../第3章/index.md)
