# ESP32 MQTT 程式導讀

> 選讀｜對應：[第 2 章：ESP32 MQTT 雙向訊息](index.md)

本頁用第 2 章的固定假資料程式，說明 ESP32 如何連上 Wi-Fi、訂閱 MQTT、送出事件並確認相符回覆。

## 讀完後你會知道

- Wi-Fi、TLS、Broker、topic 與 `PubSubClient` 的工作順序。
- 為什麼 ESP32 要略過自己送出的事件，並比對 `event_id`。
- Wi-Fi 或 MQTT 中斷時，程式為何顯示未知而不無限重連。

## 先知道這件事

主頁已讓 ESP32 與網站端用不同帳密交換固定假資料。本頁只在相同程式加入 `【編號】` 導讀註解。

```text
Wi-Fi → TLS MQTT 連線 → 訂閱 topic → 發布 card_read
                                  ↑          ↓
                       on_mqtt_message() ← Gateway 或測試端回覆
```

## 完整程式導讀

執行位置：Arduino IDE／ESP32
來源檔案：`rfid_mqtt_test.ino`

```cpp linenums="1"
// 【1】匯入 Wi-Fi、TLS 與 MQTT 函式庫
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <PubSubClient.h>

// 【2】秘密設定檔不放進 Git 或教材
#include "secrets.h"

// 【3】集中定義有限等待、裝置與虛構事件資料
constexpr unsigned long WIFI_TIMEOUT_MS = 20000;
constexpr uint8_t MQTT_CONNECT_ATTEMPTS = 3;
constexpr char MQTT_CLIENT_ID[] = "rfid-esp32-a1";
constexpr char TEST_EVENT_ID[] = "evt-001";
constexpr char TEST_EVENT[] =
    "{\"event_type\":\"card_read\",\"card_id_masked\":\"CARD-****-42\","
    "\"device_id\":\"esp32-a1\",\"event_id\":\"evt-001\","
    "\"occurred_at\":\"2026-09-16T10:00:00+08:00\"}";

// 【4】TLS Client 負責加密連線，PubSubClient 負責 MQTT
WiFiClientSecure secure_client;
PubSubClient mqtt_client(secure_client);
bool mqtt_was_connected = false;

// 【5】收到 topic 訊息時，先把位元組組成完整文字
void on_mqtt_message(char* topic, byte* payload, unsigned int length) {
  String message;
  message.reserve(length);
  for (unsigned int index = 0; index < length; index++) {
    message += static_cast<char>(payload[index]);
  }

  // 【5.1】自己的 card_read 事件也會回到訂閱端，不能當成回覆
  if (message.indexOf("\"event_type\"") >= 0) {
    Serial.println("收到讀卡測試事件，略過自身事件。");
    return;
  }

  // 【5.2】只接受本次事件代號和 status 欄位都存在的回覆
  String expected_event = "\"event_id\":\"" + String(TEST_EVENT_ID) + "\"";
  if (message.indexOf(expected_event) >= 0 &&
      message.indexOf("\"status\"") >= 0) {
    Serial.println("已收到對應的測試回覆。");
    return;
  }

  Serial.println("收到不符合本次測試規則的訊息，已略過。");
}

// 【6】在有限時間內連上 Wi-Fi
bool connect_wifi() {
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  unsigned long started_at = millis();
  while (WiFi.status() != WL_CONNECTED) {
    if (millis() - started_at >= WIFI_TIMEOUT_MS) {
      Serial.println("Wi-Fi 連線逾時，停止 MQTT 測試。");
      return false;
    }
    delay(500);
  }
  Serial.println("Wi-Fi 已連線。");
  return true;
}

// 【7】最多嘗試三次 MQTT 連線、訂閱與發布
bool connect_mqtt() {
  // 【7.1】本章實驗用 TLS：加密內容，但不驗證伺服器身分
  secure_client.setInsecure();
  mqtt_client.setServer(MQTT_HOST, MQTT_PORT);
  mqtt_client.setCallback(on_mqtt_message);
  mqtt_client.setBufferSize(512);

  for (uint8_t attempt = 1; attempt <= MQTT_CONNECT_ATTEMPTS; attempt++) {
    if (mqtt_client.connect(MQTT_CLIENT_ID, MQTT_USERNAME, MQTT_PASSWORD)) {
      Serial.println("MQTT TLS 已連線，但未驗證伺服器身分。");
      if (!mqtt_client.subscribe(MQTT_TOPIC, 0)) {
        Serial.println("訂閱測試 topic 失敗。");
        mqtt_client.disconnect();
        return false;
      }
      Serial.println("已訂閱測試 topic。");
      if (!mqtt_client.publish(MQTT_TOPIC, TEST_EVENT, false)) {
        Serial.println("發布固定測試事件失敗。");
        mqtt_client.disconnect();
        return false;
      }
      Serial.println("已送出固定測試事件，請從網站端或 Gateway 送出回覆。");
      mqtt_was_connected = true;
      return true;
    }
    Serial.printf("MQTT 連線失敗（第 %u 次，狀態 %d）。\n", attempt, mqtt_client.state());
    delay(2000);
  }
  Serial.println("MQTT 無法連線，停止自動重試。");
  return false;
}

// 【8】開機時依序完成 Wi-Fi 與 MQTT 設定
void setup() {
  Serial.begin(115200);
  Serial.println("開始 RFID MQTT 實驗測試。");
  if (!connect_wifi()) {
    return;
  }
  connect_mqtt();
}

// 【9】持續交給 PubSubClient 處理已收到的 MQTT 訊息
void loop() {
  if (mqtt_client.connected()) {
    mqtt_client.loop();
    return;
  }
  if (mqtt_was_connected) {
    mqtt_was_connected = false;
    Serial.println("MQTT 已中斷，目前狀態未知；本草稿不會自動無限重連。");
  }
  delay(100);
}
```

## 依資料流閱讀

先看【1】至【4】準備連線工具和固定資料；【5】是收到訊息後的分流；【6】、【7】建立 Wi-Fi 與 MQTT；【8】開機時依序呼叫；【9】持續處理訂閱到的訊息。

【5.1】很重要：ESP32 同時訂閱與發布同一個 topic，因此它會收到自己送出的 `card_read`。程式先略過這種事件，再以【5.2】確認回覆具有本次 `TEST_EVENT_ID` 和 `status`；這樣不會把其他事件的回覆當成目前結果。

## 出錯時先看哪裡

| 看到的現象 | 第一個檢查位置 | 可先判斷的範圍 |
| --- | --- | --- |
| Wi-Fi 連線逾時 | 【6】與 Wi-Fi 設定 | 尚未進入 MQTT。 |
| MQTT 連線失敗 | 【7】的主機、連接埠、帳密與 topic 權限 | Wi-Fi 已連線，問題在 MQTT 設定。 |
| 收到自身事件 | 【5.1】 | 訂閱與發布都正常，程式正在避免誤判。 |
| 不符合本次測試規則 | 【5.2】 | 回覆缺少相符 `event_id` 或 `status`。 |
| MQTT 已中斷 | 【9】 | 舊結果不能代表目前狀態。 |

## 常見誤解

### 「`publish()` 回傳成功就代表另一端已收到」

不是。本章使用 QoS `0`，`publish()` 只表示 ESP32 已嘗試把訊息交給連線；必須看到「已收到對應的測試回覆」才知道雙向流程完成。

### 「`setInsecure()` 等於沒有加密」

不是。它仍建立 TLS 加密連線，但 ESP32 不確認伺服器身分。因此本章只能用固定假資料與短期測試帳密；需要完整限制請回看主頁。

## 重點整理

- `PubSubClient` 的 `subscribe()` 與 `loop()` 共同讓 ESP32 接收回覆。
- `event_id` 關聯事件與回覆，避免錯把其他訊息當成成功。
- Wi-Fi 或 MQTT 中斷時，程式顯示未知並停止自動重連。

## 回到專題

回到[第 2 章主頁](index.md)，或繼續閱讀[第 3 章：Python MQTT Gateway](../第3章/index.md)。
