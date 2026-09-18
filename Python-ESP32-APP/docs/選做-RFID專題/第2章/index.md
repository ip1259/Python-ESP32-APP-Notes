# 選做專題第 2 章：ESP32 MQTT 雙向訊息

本章讓 ESP32 將一筆固定的讀卡測試事件送到 HiveMQ Cloud，並接收網站端回覆。

!!! warning "選做實作與安全警告"

    本章需要可上網的 Wi-Fi、HiveMQ Cloud 免費帳號，以及可建立與刪除的測試帳密。程式為了降低初次實驗的憑證設定門檻，使用 `setInsecure()`；它仍會加密傳輸內容，但**不會驗證伺服器身分**。若連到偽造的伺服器，MQTT 帳密可能外洩。

    因此只能在你可控制的網路中，傳送本章的固定假資料，並使用短期、可隨時刪除的測試帳密。不得傳送真實 RFID UID、個人資料、Wi-Fi 資訊、控制設備指令、出入權限或使用紀錄。完成測試後請刪除或重建帳密。

## 你會學到什麼

- 建立 HiveMQ Cloud 免費方案的測試連線資料。
- 用 `PubSubClient` 2.8.0 發布與訂閱同一個 MQTT topic（主題）。
- 從網站端送出固定回覆，確認 ESP32 能收到正確訊息。
- 用可觀察的輸出判斷帳密失效、訊息格式不符與 Wi-Fi 中斷。

## 開始前

你需要先完成：

- [第 1 章：RC522 RFID 與 SPI](../../選做-RC522-RFID與SPI.md)
- ESP32 已能連上你自己的 Wi-Fi，且 Arduino IDE 可以燒錄程式。
- HiveMQ Cloud 的 Serverless FREE 免費帳號與叢集（cluster）。
- Arduino IDE 已安裝 `PubSubClient`，發布者為 Nick O'Leary，版本為 `2.8.0`。

本章使用 NodeMCU-32S 相容 ESP32 開發板，不需要額外接線。先不要把 RFID 讀到的真實 UID 放進 MQTT 訊息；本章一律使用程式中的固定假資料。

## 成功的樣子

ESP32 序列埠監控視窗會依序出現類似訊息：

```text
開始 RFID MQTT 實驗測試。
Wi-Fi 已連線。
MQTT TLS 已連線，但未驗證伺服器身分。
已訂閱測試 topic。
已送出固定測試事件，請從 HiveMQ Cloud 手動送出回覆。
已收到對應的測試回覆。
```

網站端也能在相同 topic 收到 ESP32 送出的固定測試事件。這只表示本章的雙向測試成立，不代表可用於真實 RFID 門禁。

## 操作步驟

### 1. 建立只供本章使用的測試帳密

目的：準備 ESP32 與網站端各自登入 MQTT 服務所需的資料。

1. 登入 HiveMQ Cloud，建立或開啟 Serverless FREE 叢集。
2. 建立兩組可刪除的測試帳密：一組給 ESP32，另一組給網站端測試工具。兩組帳密不要相同。
3. 為每一組帳密建立相同的 topic 權限：允許發布（publish）與訂閱（subscribe）下方的**單一完整 topic**。請將 `YOUR_NAMESPACE` 改成你自己不含個人資料的短名稱。

   ```text
   YOUR_NAMESPACE/devices/esp32-a1/messages
   ```

4. 從叢集資訊複製主機名稱（host）與連接埠（port）。本章實測使用 TLS 連接埠 `8883`；若你的控制台顯示不同值，請以控制台為準。

預期結果：你手上有主機名稱、連接埠、兩組不同的測試帳密，以及一個只供本章使用的 topic。

想先認識控制台畫面與設定範圍，請閱讀[HiveMQ Cloud 免費測試設定示範](hivemq-cloud-測試設定.md)。

!!! warning "單一 topic 不是方向隔離"

    同一個 topic 同時用於送出事件與接收回覆，是為了配合免費方案下容易建立的單一 topic 權限。它不是「ESP32 只能送、網站只能收」的方向隔離；因此只適合本章的固定假資料實驗。

### 2. 建立不公開的連線設定檔

目的：將 Wi-Fi 與 MQTT 帳密放在主程式之外，避免直接貼到筆記、Git 或公開畫面。

在 Arduino 草稿資料夾中建立 `secrets.h`。

執行位置：Arduino IDE／ESP32，檔名：`secrets.h`

```cpp
#pragma once

const char* WIFI_SSID = "你的 Wi-Fi 名稱";
const char* WIFI_PASSWORD = "你的 Wi-Fi 密碼";

const char* MQTT_HOST = "你的叢集主機名稱";
const uint16_t MQTT_PORT = 8883;
const char* MQTT_USERNAME = "ESP32 專用測試帳號";
const char* MQTT_PASSWORD = "ESP32 專用測試密碼";

const char* MQTT_TOPIC = "YOUR_NAMESPACE/devices/esp32-a1/messages";
```

預期結果：`secrets.h` 只有留在自己的電腦中，不會被上傳、截圖或貼到公開場所。

### 3. 燒錄固定事件測試程式

目的：讓 ESP32 連上 Wi-Fi、建立 MQTT 連線、訂閱 topic，並送出一筆固定事件。

執行位置：Arduino IDE／ESP32，檔名：`rfid_mqtt_test.ino`

```cpp
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <PubSubClient.h>
#include "secrets.h"

WiFiClientSecure secure_client;
PubSubClient mqtt_client(secure_client);

const char* DEVICE_ID = "esp32-a1";
const char* TEST_EVENT_ID = "evt-001";
const char* TEST_EVENT =
  "{\"event_type\":\"card_read\",\"card_id_masked\":\"CARD-****-42\","
  "\"device_id\":\"esp32-a1\",\"event_id\":\"evt-001\","
  "\"occurred_at\":\"2026-09-16T10:00:00+08:00\"}";

void connect_wifi() {
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
  }
  Serial.println("Wi-Fi 已連線。");
}

void on_message(char* topic, byte* payload, unsigned int length) {
  String message = "";
  for (unsigned int i = 0; i < length; i++) {
    message += static_cast<char>(payload[i]);
  }

  if (message.indexOf("\"event_type\":\"card_read\"") >= 0) {
    Serial.println("收到讀卡測試事件，略過自身事件。");
    return;
  }

  String expected_event = "\"event_id\":\"" + String(TEST_EVENT_ID) + "\"";
  if (message.indexOf(expected_event) >= 0 &&
      message.indexOf("\"status\"") >= 0) {
    Serial.println("已收到對應的測試回覆。");
    return;
  }

  Serial.println("收到不符合本次測試規則的訊息，已略過。");
}

bool connect_mqtt() {
  for (int attempt = 1; attempt <= 3; attempt++) {
    if (mqtt_client.connect(DEVICE_ID, MQTT_USERNAME, MQTT_PASSWORD)) {
      Serial.println("MQTT TLS 已連線，但未驗證伺服器身分。");
      if (mqtt_client.subscribe(MQTT_TOPIC, 0)) {
        Serial.println("已訂閱測試 topic。");
        return true;
      }
      Serial.println("訂閱失敗。");
      mqtt_client.disconnect();
    } else {
      Serial.printf("MQTT 連線失敗（第 %d 次，狀態 %d）。\n",
                    attempt, mqtt_client.state());
    }
    delay(2000);
  }

  Serial.println("MQTT 無法連線，停止自動重試");
  return false;
}

void setup() {
  Serial.begin(115200);
  Serial.println("開始 RFID MQTT 實驗測試。");

  // 僅限本章短期假資料實驗：會加密，但不驗證伺服器身分。
  secure_client.setInsecure();
  mqtt_client.setServer(MQTT_HOST, MQTT_PORT);
  mqtt_client.setCallback(on_message);

  connect_wifi();
  if (!connect_mqtt()) {
    return;
  }

  if (mqtt_client.publish(MQTT_TOPIC, TEST_EVENT, false)) {
    Serial.println("已送出固定測試事件，請從 HiveMQ Cloud 手動送出回覆。");
  } else {
    Serial.println("固定測試事件送出失敗。");
  }
}

void loop() {
  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("MQTT 已中斷，目前狀態未知；本草稿不會自動無限重連。");
    delay(5000);
    return;
  }

  if (mqtt_client.connected()) {
    mqtt_client.loop();
  }
}
```

預期結果：開啟鮑率 `115200` 的序列埠監控視窗後，會看到 Wi-Fi 已連線、MQTT 已訂閱，以及固定事件已送出。

#### 程式中重要功能

`PubSubClient` 是讓 ESP32 使用 MQTT 的函式庫。本章使用的版本是 `2.8.0`，發布者是 Nick O'Leary。

| 功能 | 最常見的用法 | 傳入資料與結果 |
| --- | --- | --- |
| `connect()` | 登入 MQTT 服務 | 傳入裝置名稱、帳號、密碼；成功時回傳 `true`。 |
| `subscribe()` | 訂閱一個 topic | 傳入 topic 與 QoS `0`；成功時回傳 `true`。 |
| `publish()` | 送出一筆訊息 | 傳入 topic、文字訊息與 `false`；成功時回傳 `true`。`false` 表示不保留舊訊息。 |
| `setCallback()` | 指定收到訊息後的處理方式 | 傳入函式名稱；收到訊息時會執行 `on_message()`。 |
| `loop()` | 持續處理已收到的訊息 | 沒有回傳值；需要在 `loop()` 中持續呼叫。 |

程式只比對固定文字，目的是讓你確認雙向流程，不是完整的 JSON 格式檢查工具。

### 4. 從網站端送出固定回覆

目的：確認 ESP32 能從同一個 topic 收到另一組帳密送出的回覆。

!!! tip "先訂閱，再讓 ESP32 發布"

    先完成下方第 1、2 步，再按 ESP32 的 `EN` 按鈕重新開始固定事件測試。這個測試使用 retain `false`，較晚訂閱的一端不會收到先前送出的舊訊息。

1. 在 HiveMQ Cloud 的網站測試工具，以「網站端測試帳密」登入。
2. 訂閱與 ESP32 完全相同的 `MQTT_TOPIC`。
3. 先確認網站端收到 ESP32 的固定事件。
4. 發布下列**完全相同、不加空白**的文字到同一個 topic：

   ```json
   {"event_id":"evt-001","status":"authorized"}
   ```

!!! success "收到固定回覆"

    ESP32 顯示 `已收到對應的測試回覆。` 網站端與 ESP32 必須使用不同帳密，才是真正的雙端測試。

### 5. 觀察被略過的訊息與中斷狀態

目的：確認測試程式不會把不屬於本次固定事件的內容誤當成成功回覆。

在網站端依序發布下列任一文字到同一個 topic：

```text
not-json
```

```json
{"event_id":"evt-999","status":"accepted"}
```

預期結果：ESP32 顯示 `收到不符合本次測試規則的訊息，已略過。`

接著暫時讓 ESP32 離開 Wi-Fi 範圍或關閉測試網路，觀察序列埠輸出。

預期結果：出現 `MQTT 已中斷，目前狀態未知；本草稿不會自動無限重連。` 恢復 Wi-Fi 後，按 ESP32 的 `EN` 按鈕或重新燒錄，即可重新開始這次測試。

### 6. 刪除測試帳密並確認失效

目的：確認帳密確實可撤銷，並結束這次使用 `setInsecure()` 的實驗。

1. 在 HiveMQ Cloud 刪除 ESP32 的測試帳密。
2. 按 ESP32 的 `EN` 按鈕重新啟動。
3. 觀察序列埠輸出後，再建立新的測試帳密並更新 `secrets.h`。

預期結果：刪除後會看到三次 MQTT 連線失敗，最後出現 `MQTT 無法連線，停止自動重試`。換成新帳密、重新燒錄後，才能再次連線。

完成本章後，不再使用的 ESP32 與網站端測試帳密都應刪除。

## 完成時，應能確認

- ESP32 與網站端使用不同測試帳密，卻能在同一個限定 topic 收到固定假資料。
- ESP32 收到 `evt-001` 與 `authorized` 的固定回覆時，顯示成功訊息。
- 純文字或錯誤事件編號的訊息會被略過。
- 刪除測試帳密後，ESP32 無法再登入；重建帳密後才可恢復測試。
- 你知道本章的 `setInsecure()` 只適用短期假資料實驗，且已刪除不再使用的帳密。

## 常見問題

### Arduino IDE 顯示找不到 `PubSubClient.h`

開啟「程式庫管理員」，搜尋並安裝 `PubSubClient`。請確認發布者為 Nick O'Leary，版本為 `2.8.0`，再重新編譯。

### 顯示 MQTT 連線失敗

先確認主機名稱只填網域名稱，沒有加入 `mqtts://`、`https://` 或多餘路徑。再確認連接埠、帳密與該帳密的單一 topic 發布／訂閱權限。若帳密已刪除，建立新帳密後必須同步更新 `secrets.h` 並重新燒錄。

### 網站端沒有看到 ESP32 的事件

先在網站端訂閱正確 topic，再按 ESP32 的 `EN` 按鈕重新開始。這個測試使用 retain `false`，較晚訂閱的一端不會收到先前送出的舊訊息。

### ESP32 顯示「不符合本次測試規則」

回覆必須帶有目前 `TEST_EVENT_ID` 的值與 `status` 欄位；`status` 可以是 `authorized` 或 `unauthorized`。程式只做必要文字比對，沒有處理不同欄位順序、空白或完整 JSON 格式檢查。

### 為什麼測完還要刪除帳密？

本章為了降低初次實驗門檻而使用 `setInsecure()`，未驗證伺服器身分。刪除短期帳密可縮短帳密意外外洩後可被使用的時間；要進行真實服務時，必須改用可驗證伺服器身分的憑證設定。想理解加密與確認伺服器身分的差別，請閱讀[延伸選讀：公私鑰加密、數位簽章與中間人攻擊防範](../../附錄-公私鑰加密數位簽章與中間人攻擊防範.md)。

## 重點整理

- MQTT 以 broker（訊息中介服務）轉送資料；本章由 ESP32 與網站端訂閱同一個 topic 進行固定假資料測試。
- `PubSubClient` 的 `publish()` 用來送出事件，`subscribe()` 與 `loop()` 用來接收回覆。
- 本章使用 QoS `0` 與 retain `false`，適合觀察即時測試流程，不保證訊息一定送達，也不保存舊訊息。
- 加密不等於已驗證身分。使用 `setInsecure()` 時，假資料、短期帳密與完成後刪除帳密都是必要限制。

## 下一步

下一章將把 ESP32 送出的固定訊息交給 Python 接收與整理：

- [ESP32 MQTT 程式導讀](mqtt程式導讀.md)
- [PubSubClient 函式庫介紹](pubsubclient函式庫介紹.md)
- [專題第 3 章｜Python MQTT Gateway 與 FastAPI 初步概念](../第3章/index.md)
