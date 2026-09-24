# PubSubClient：兩個 Topic 的控制與回傳實作

這個選做實作讓 ESP32 訂閱控制板載 LED 的 Topic，並將處理結果發布到另一個 Topic，再用 HiveMQ Cloud Web Client 觀察兩個方向。

!!! info "本頁使用的硬體與軟體組合"

    本頁採用 `esp32 by Espressif Systems 3.3.11`、`NodeMCU-32S` 選板與 `PubSubClient 2.8.0`。操作時請逐步對照預期結果；若自己的帳號、Wi-Fi 或開發板出現不同結果，先停在該步檢查，不要放寬權限、改用未加密連線，或把程式接到其他實體設備。

!!! warning "選做實作的網路與安全界線"

    這個練習需要 ESP32、HiveMQ Cloud 帳號、受控 Wi-Fi 與兩組可刪除的 MQTT 帳密。只傳送 `on`、`off` 等固定假資料。Wi-Fi 密碼、Broker 主機名稱、實際 namespace 和 MQTT 帳密只保存在自己不公開的電腦位置，不要公開或傳給他人。程式只可控制板載指示燈，不可直接套用到繼電器、門鎖、市電或其他會影響環境的設備。

## 你會學到什麼

- 建立 `led` 控制 Topic 與 `status` 回傳 Topic，分辨兩個方向的訊息。
- 使用 `PubSubClient` 的 `setServer()`、`setCallback()`、`connect()`、`subscribe()`、`publish()` 與 `loop()`。
- 把 Wi-Fi 與 MQTT 帳密放進不公開的 `Secrets.h`。
- 以 Web Client 先訂閱狀態、再發布 `on` 或 `off`，觀察完整往返。

## 開始前

- 已閱讀 [MQTT 基礎：發布、訂閱與 Broker](mqtt基礎.md)。
- 建議先閱讀 [PubSubClient：MQTT 函式庫介紹](pubsubclient函式庫介紹.md)，了解 `setServer()`、callback 與 `loop()` 等通用 API（函式庫提供的功能）。
- 已依 [HiveMQ Cloud 基本設定](hivemq-cloud-基本設定.md) 建立自己的 Serverless 測試叢集，並能開啟 `Access Management` 與 `Test your connection`。前一頁的 `YOUR_NAMESPACE/demo/events` 保持原樣；本頁另建專用權限與帳密。
- 準備 NodeMCU-32S 相容 ESP32、可傳輸資料的 USB 線、Arduino IDE 與受控 Wi-Fi。
- 已用 Blink 確認自己的板載 LED 腳位、亮滅電位與 USB 上傳正常。本頁使用 NodeMCU-32S 常見的 `GPIO 2`、`HIGH` 亮燈、`LOW` 熄燈；若你的板子結果不同，先修改程式中的三個 LED 常數。
- 準備不含姓名、學號、電子郵件或裝置序號的 `YOUR_NAMESPACE`。它是用來區分自己練習 Topic 的短前綴。

本頁使用 MQTT over TLS 的 `8883`。程式暫時呼叫 `setInsecure()`；它會加密連線內容，但 ESP32 **不會驗證伺服器身分**。只限受控 Wi-Fi、固定假資料與可立即刪除的短期帳密。Root CA 憑證不列為本頁必要步驟。

## 成功的樣子

完成本練習後，應能看到下列資料流：

```text
HiveMQ Web Client ── on／off ──> .../led ──> ESP32 板載 LED
HiveMQ Web Client <── led=on／led=off ── .../status <── ESP32
```

`on` 送到 Broker 不等於 LED 已亮。只有 ESP32 收到命令、設定 LED，並回傳 `led=on`，才是本練習要觀察的完整結果。

## 先約定 Topic、權限與帳密

### 1. 固定兩個 Topic

目的：把控制要求與裝置回傳分開，讓兩個訊息方向容易辨認。

替換自己的 `YOUR_NAMESPACE`，其餘層級保持不變：

| 用途 | Topic | 發布者 | 訂閱者 | payload |
| --- | --- | --- | --- | --- |
| 控制板載 LED | `YOUR_NAMESPACE/mqtt-lab/device-a/led` | HiveMQ Web Client | ESP32 | `on`、`off` |
| 回傳處理狀態 | `YOUR_NAMESPACE/mqtt-lab/device-a/status` | ESP32 | HiveMQ Web Client | `online`、`led=on`、`led=off` |

`online` 只表示 ESP32 剛連上 Broker，不代表 LED 已亮起。

預期結果：你已寫下兩個完整 Topic，且兩者只有最後一層不同。

### 2. 建立本頁專用權限

目的：把兩組測試帳密可使用的 Topic 限制在這次練習的固定前綴下。

在 HiveMQ Cloud 的 `Access Management` → `Authorization` → `Permissions` 新增自訂權限。欄位位置可參考前一頁的既有圖片，但這裡要輸入本表的值。

| 欄位 | 本頁值 |
| --- | --- |
| 名稱 | `mqtt-led-lab-only` 等不含個人資料的名稱 |
| 說明 | `只供 ESP32 LED MQTT 練習使用` |
| Topic Filter | `YOUR_NAMESPACE/mqtt-lab/device-a/+` |
| Permission Type | `Publish and Subscribe` |

`+` 是單層萬用字元，必須單獨占滿一個層級。它會符合 `YOUR_NAMESPACE/mqtt-lab/device-a/` 後方任何一個單層名稱；本練習目前只使用 `led` 與 `status`。若日後在相同前綴下新增 `debug` 等 Topic，它也會被這項權限涵蓋，因此新增前要重新檢查範圍。本頁不需要涵蓋多層的 `#`。

預期結果：權限清單出現 `YOUR_NAMESPACE/mqtt-lab/device-a/+`，Permission Type 為 `Publish and Subscribe`；目前的程式只使用這個前綴下的 `led` 與 `status`。

### 3. 建立兩組新的 MQTT 帳密

目的：讓 ESP32 與 Web Client 使用不同登入身分，其中一組外洩時可單獨撤銷。

在 `Access Management` → `Authentication`／`Credentials` 建立兩組 Access Credentials：

| 帳密角色 | 建議名稱 | 在本練習的用途 |
| --- | --- | --- |
| ESP32 | `device-led-test` | 寫入 `Secrets.h`；訂閱 `.../led`、發布 `.../status` |
| Web Client | `web-led-test` | 登入 Web Client；發布 `.../led`、訂閱 `.../status` |

兩組帳密都選取步驟 2 的同一項 `Publish and Subscribe` 權限，但密碼必須不同。這是 Serverless 免費方案的練習折衷；程式仍使用精確 Topic，ESP32 不訂閱 `.../status`，Web Client 也不需訂閱 `.../led`。

預期結果：帳密清單有兩組不同、可刪除的帳密，每一組只選取 `YOUR_NAMESPACE/mqtt-lab/device-a/+` 這一項自訂權限。

## 建立 Arduino 專案與私密設定檔

### 4. 安裝函式庫並選擇開發板

在 Arduino IDE 完成下列設定：

1. 在「開發板管理員」確認已安裝 `esp32 by Espressif Systems 3.3.11`。
2. 在「函式庫管理員」安裝 `PubSubClient 2.8.0`。
3. 在「工具」選擇 `NodeMCU-32S`，並選擇自己的序列埠。
4. 建立名為 `mqtt_led_lab` 的新草稿。

預期結果：能開啟 `mqtt_led_lab.ino`，選單顯示正確板型與序列埠。自己的帳密、Wi-Fi 與板子仍應依後續步驟逐一確認。

### 5. 建立不公開的 `Secrets.h`

在 Arduino IDE 新增分頁，檔名輸入 `Secrets.h`。它必須和 `mqtt_led_lab.ino` 位於同一草稿資料夾。貼入下列內容，然後替換每個 `YOUR_...` 值。

執行位置：Arduino IDE／ESP32  
檔名：`Secrets.h`

```cpp
#pragma once

// 這個檔案只保存在自己不公開的電腦位置，不要公開或傳給他人。
// 測試後請刪除或輪替這組 MQTT 帳密。

constexpr char WIFI_SSID[] = "YOUR_WIFI_SSID";
constexpr char WIFI_PASSWORD[] = "YOUR_WIFI_PASSWORD";

// 從 HiveMQ Cloud 的 TLS MQTT URL 取出主機名稱；不要包含 mqtts://。
constexpr char MQTT_HOST[] = "YOUR_CLUSTER_HOST";
constexpr uint16_t MQTT_PORT = 8883;
constexpr char MQTT_USERNAME[] = "YOUR_ESP32_LED_USERNAME";
constexpr char MQTT_PASSWORD[] = "YOUR_ESP32_LED_PASSWORD";

// Broker 用這個名稱區分每個連線用戶端，不可與其他連線重複。
constexpr char MQTT_CLIENT_ID[] = "esp32-led-lab-a";

// YOUR_NAMESPACE 不可包含姓名、學號、信箱或裝置序號。
constexpr char MQTT_CONTROL_TOPIC[] = "YOUR_NAMESPACE/mqtt-lab/device-a/led";
constexpr char MQTT_STATUS_TOPIC[] = "YOUR_NAMESPACE/mqtt-lab/device-a/status";
```

`MQTT_HOST` 是控制台 MQTT URL 的主機名稱部分。例如控制台顯示 `mqtts://example.s1.eu.hivemq.cloud:8883`，只填 `example.s1.eu.hivemq.cloud`；不要填 `mqtts://` 或 `:8883`。`MQTT_USERNAME` 與 `MQTT_PASSWORD` 填入 ESP32 帳密，不是 Web Client 帳密。

預期結果：`Secrets.h` 只保存在自己的電腦，所有 `YOUR_...` 已替換，而且檔案內容沒有公開或傳給他人。

### 6. 貼上完整 ESP32 程式

將 `mqtt_led_lab.ino` 原有內容全部替換成下列程式。它會訂閱精確控制 Topic；收到 `on` 或 `off` 時設定板載 LED，再把狀態發布到精確回傳 Topic。

執行位置：Arduino IDE／ESP32  
檔名：`mqtt_led_lab.ino`

```cpp
// ESP32 + HiveMQ Cloud 的雙 Topic 板載 LED 練習。
// 執行位置：Arduino IDE／ESP32
//
// 安全警告：本程式使用 setInsecure()，TLS 仍會加密傳輸內容，
// 但 ESP32 不會驗證伺服器身分。只可用受控 Wi-Fi、固定 on/off 假資料與
// 可立即刪除的短期帳密；不得傳送個資、真實 UID 或控制實體設備。

#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <PubSubClient.h>
#include <cstring>

#include "Secrets.h"

// ===== 1. 板載 LED 與重試間隔設定 =====
// NodeMCU-32S 的板載 LED 使用 GPIO 2，HIGH 為亮、LOW 為滅。
// 若自己的 Blink 結果不同，只修改這三個 LED 常數。
constexpr uint8_t LED_PIN = 2;
constexpr uint8_t LED_ON_LEVEL = HIGH;
constexpr uint8_t LED_OFF_LEVEL = LOW;

// 失去連線時不在每一次 loop() 都送出新要求，避免過度重試。
constexpr unsigned long WIFI_RETRY_INTERVAL_MS = 10000;
constexpr unsigned long MQTT_RETRY_INTERVAL_MS = 5000;

// ===== 2. 網路與 MQTT 物件 =====
// secure_client 先負責 TLS 連線；mqtt_client 再透過它收發 MQTT 訊息。
WiFiClientSecure secure_client;
PubSubClient mqtt_client(secure_client);

// ===== 3. 上次嘗試連線的時間 =====
// millis() 從開機開始計時。兩個時間分開記錄，讓 Wi-Fi 與 MQTT 各自重試。
unsigned long last_wifi_attempt_at = 0;
unsigned long last_mqtt_attempt_at = 0;

// ===== 4. 小型輔助函式 =====
// 判斷距離上次嘗試是否已超過指定間隔。
bool is_retry_due(unsigned long last_attempt_at, unsigned long interval_ms) {
  return millis() - last_attempt_at >= interval_ms;
}

// 將 ESP32 已處理的結果送到狀態 Topic；false 表示不保留舊訊息。
void publish_status(const char* status) {
  if (!mqtt_client.publish(MQTT_STATUS_TOPIC, status, false)) {
    Serial.println("狀態發布失敗。");
  }
}

// LED 寫入完成後立刻回傳對應狀態，讓 Web Client 可觀察實際處理結果。
void set_led_and_report(uint8_t level, const char* status) {
  digitalWrite(LED_PIN, level);
  publish_status(status);
  Serial.printf("板載 LED 已處理為 %s。\n", status);
}

// ===== 5. 收到控制 Topic 時的 callback =====
// mqtt_client.loop() 收到訂閱訊息後，會呼叫這個函式。
void on_mqtt_message(char* topic, byte* payload, unsigned int length) {
  // 即使目前只訂閱一個 Topic，仍先確認收到的是預期控制 Topic。
  if (strcmp(topic, MQTT_CONTROL_TOPIC) != 0) {
    Serial.println("收到非控制 Topic 的訊息，已忽略。");
    return;
  }

  // payload 不保證以 C++ 字串結尾，因此以 length 和 memcmp() 精確比對。
  if (length == 2 && memcmp(payload, "on", 2) == 0) {
    set_led_and_report(LED_ON_LEVEL, "led=on");
    return;
  }

  if (length == 3 && memcmp(payload, "off", 3) == 0) {
    set_led_and_report(LED_OFF_LEVEL, "led=off");
    return;
  }

  Serial.println("控制內容不是 on 或 off，已忽略。");
}

// ===== 6. Wi-Fi 重新連線 =====
// 已連上 Wi-Fi 或尚未到重試時間時，直接離開，不重複呼叫 WiFi.begin()。
void start_wifi_if_due() {
  if (WiFi.status() == WL_CONNECTED ||
      !is_retry_due(last_wifi_attempt_at, WIFI_RETRY_INTERVAL_MS)) {
    return;
  }

  last_wifi_attempt_at = millis();
  Serial.println("嘗試連線 Wi-Fi。");
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
}

// ===== 7. MQTT 重新連線、訂閱與上線通知 =====
// MQTT 必須等 Wi-Fi 成功後才可連線；每次 MQTT 重連成功後都要重新訂閱。
void connect_mqtt_if_due() {
  if (WiFi.status() != WL_CONNECTED || mqtt_client.connected() ||
      !is_retry_due(last_mqtt_attempt_at, MQTT_RETRY_INTERVAL_MS)) {
    return;
  }

  last_mqtt_attempt_at = millis();
  Serial.println("嘗試連線 MQTT。");

  if (!mqtt_client.connect(MQTT_CLIENT_ID, MQTT_USERNAME, MQTT_PASSWORD)) {
    Serial.printf("MQTT 連線失敗，狀態 %d。\n", mqtt_client.state());
    return;
  }

  Serial.println("MQTT TLS 已連線，但未驗證伺服器身分。");

  if (!mqtt_client.subscribe(MQTT_CONTROL_TOPIC, 0)) {
    Serial.println("訂閱控制 Topic 失敗。");
    mqtt_client.disconnect();
    return;
  }

  Serial.println("已訂閱控制 Topic。");
  publish_status("online");
}

// ===== 8. 開機設定 =====
void setup() {
  Serial.begin(115200);
  delay(500);
  Serial.println("開始 MQTT 板載 LED 練習。");

  // 先讓 LED 維持熄滅，避免開機時留下不確定狀態。
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LED_OFF_LEVEL);

  // 使用一般 Wi-Fi 用戶端模式，不建立 ESP32 自己的 Wi-Fi 熱點。
  WiFi.mode(WIFI_STA);

  // 此練習的短期連線設定：傳輸會加密，但不驗證 Broker 身分。
  secure_client.setInsecure();

  // Broker 位址、連接埠與 Topic 均由不公開的 Secrets.h 提供。
  mqtt_client.setServer(MQTT_HOST, MQTT_PORT);
  mqtt_client.setCallback(on_mqtt_message);

  // Topic 與 payload 都很短，PubSubClient 預設 256 位元組緩衝區足夠。
  // 減去重試間隔可讓開機後第一次呼叫立刻嘗試連線。
  last_wifi_attempt_at = millis() - WIFI_RETRY_INTERVAL_MS;
  last_mqtt_attempt_at = millis() - MQTT_RETRY_INTERVAL_MS;
  start_wifi_if_due();
}

// ===== 9. 持續處理連線與訊息 =====
void loop() {
  start_wifi_if_due();
  connect_mqtt_if_due();

  // 必須持續呼叫，PubSubClient 才能收到訊息並執行 callback。
  if (mqtt_client.connected()) {
    mqtt_client.loop();
  }
}
```

`on_mqtt_message()` 是 callback（收到訂閱訊息時會被呼叫的函式）。它先確認是精確的 `.../led`，再用 `length` 與 `memcmp()` 比對 payload，因為 MQTT payload 不保證有 C++ 字串結尾。

`memcmp()` 用來逐一比較兩段資料的位元組是否相同。例如 `memcmp(payload, "on", 2)` 會比較 `payload` 開頭的 2 個位元組和文字 `on`：前兩個參數是要比較的兩段資料，最後的 `2` 是比較長度；完全相同時會回傳 `0`。程式先檢查 `length == 2`，再比較這 2 個位元組，才不會把 `on `、`only` 或長度不足的資料誤認成 `on`。

因此只有全小寫 `on` 與 `off` 會改變 LED；`ON`、`blink`、空白與其他文字都會被忽略。

`mqtt_client.loop()` 必須在 Arduino 的 `loop()` 持續執行，訊息才會進入 callback。重新連線嘗試有 5 或 10 秒間隔，避免每一圈都送出要求；`connect()` 本身仍可能等待網路逾時，因此網路不通時序列監控可能暫時停在連線嘗試。

預期結果：按下編譯後不應出現找不到 `Secrets.h`、`WiFiClientSecure.h` 或 `PubSubClient.h` 的錯誤。編譯成功只表示語法與目前函式庫組合可用，尚未表示可以連上你的 Broker。

## 上傳並觀察 ESP32

### 7. 上傳前確認板載 LED 基線

先拔除 USB 或停止供電，再檢查沒有外接到 `GPIO 2` 的線路。使用已確認可用的 Blink 程式檢查：哪個腳位控制板載 LED，以及 `HIGH`、`LOW` 分別是亮或滅。

若結果不是 `GPIO 2`、`HIGH` 亮、`LOW` 滅，回到步驟 6，只修改 `LED_PIN`、`LED_ON_LEVEL`、`LED_OFF_LEVEL` 三個常數。不要直接猜測其他腳位。

預期結果：你已確認本板的 LED 基線；上傳本頁程式前，LED 預設會寫入你定義的熄滅電位。本頁的 NodeMCU-32S 設定為 `GPIO 2`、`HIGH` 亮、`LOW` 滅。

### 8. 上傳並開啟序列監控

選擇 Arduino IDE 的「上傳」。完成後開啟「序列監控」，速率設為 `115200 baud`。序列監控不應顯示帳密、Wi-Fi 名稱、主機名稱或實際 namespace；若意外出現秘密資訊，立即停止測試並刪除或輪替受影響的帳密。

在 Wi-Fi、帳密、權限和 Broker 均正確時，序列監控的**預期**順序如下：

```text
開始 MQTT 板載 LED 練習。
嘗試連線 Wi-Fi。
嘗試連線 MQTT。
MQTT TLS 已連線，但未驗證伺服器身分。
已訂閱控制 Topic。
```

本程式刻意顯示「未驗證伺服器身分」，提醒 `setInsecure()` 的限制；它不是安全成功訊息。連線後 ESP32 會嘗試發布 `online` 到 `.../status`。

預期結果：看到 Wi-Fi、MQTT 連線與訂閱的順序訊息。若顯示 `MQTT 連線失敗，狀態 ...`，先不要反覆修改 Topic 或權限，請看本頁的常見問題。

## 用 HiveMQ Web Client 做雙向測試

### 9. 用第二組帳密連上 Web Client

在 HiveMQ Cloud 叢集內開啟 `Test your connection`。輸入步驟 3 建立的 **Web Client 帳密**，不是 ESP32 帳密，然後選擇 `Connect`。若目前控制台名稱或位置不同，先回到 [HiveMQ 官方 Web Client 測試說明](https://docs.hivemq.com/hivemq-platform/connect/test-sub-pub-web-client.html) 對照，不要改用來源不明的公開測試工具。

預期結果：Web Client 顯示已連線。帳密剛建立時，設定可能需要短暫時間才生效；等待後重新連線，不要建立全域或 `#` 權限繞過問題。

### 10. 先訂閱狀態 Topic

在 Web Client 訂閱區輸入精確 Topic：

```text
YOUR_NAMESPACE/mqtt-lab/device-a/status
```

選擇 QoS `0` 後訂閱。不要改訂閱 `.../+`；權限可用萬用字元，不代表測試工具需要收所有符合的訊息。

預期結果：Web Client 顯示已訂閱 `.../status`。若此時重新啟動已連線的 ESP32，Web Client 預期會收到 `online`；這只表示 ESP32 剛完成 Broker 連線。

### 11. 發布 `on`，觀察亮燈與回傳

在 Web Client 發布區填入：

| 欄位 | 值 |
| --- | --- |
| Topic | `YOUR_NAMESPACE/mqtt-lab/device-a/led` |
| payload | `on` |
| QoS | `0` |
| retain | 關閉／`false` |

發布後，同時觀察 ESP32 板載 LED、序列監控與 `.../status` 的訂閱訊息。

預期結果：

```text
板載 LED 已處理為 led=on。
```

板載 LED 應依步驟 7 確認的電位亮起，Web Client 的狀態訂閱應看到 `led=on`。只有 Web Client 發布成功，不能代表 LED 已處理。

### 12. 發布 `off`，觀察熄燈與回傳

保留相同的發布 Topic、QoS 和 retain 設定，只把 payload 改成：

```text
off
```

預期結果：板載 LED 依你的基線熄滅，序列監控顯示 `板載 LED 已處理為 led=off。`，Web Client 的 `.../status` 訂閱收到 `led=off`。

### 13. 未知命令與重新連線

1. 對 `.../led` 發布 `ON`、`blink` 或空白字串。LED 應維持原狀，序列監控**預期**顯示「控制內容不是 on 或 off，已忽略。」；Web Client 不應收到成功狀態。
2. 重新啟動 ESP32，或暫時中斷再恢復受控 Wi-Fi。恢復後程式預期重新連線、重新訂閱 `.../led`，並發布一次 `online`。`connect()` 在網路不通時可能等待逾時，這是正常的網路等待現象。

不要在公共 Wi-Fi、行動網路不明設定或弱訊號環境中反覆測試。若板子異常發熱、USB 反覆斷線或不斷重啟，立即拔除 USB 並停止。

## 完成時，應能確認

完成本頁後，你應能確認：

- Web Client 只訂閱精確的 `.../status`，再發布精確的 `.../led`。
- `on` 對應 LED 亮燈與 `led=on` 回傳；`off` 對應熄燈與 `led=off` 回傳。
- ESP32 每次 MQTT 重新連線後會重新訂閱控制 Topic，並發布 `online`。
- `ON`、`blink` 或空白不會被當成合法命令。
- `Secrets.h` 只保存在自己不公開的電腦位置；Web Client 與 ESP32 使用不同帳密及不同 `clientId`。

若自己的環境未完成其中一項，請把實際看到的結果與本頁「預期結果」分開記錄；不要只因為編譯成功或 Web Client 已連線，就認為完整往返已成立。

## 常見問題

### 編譯時找不到 `Secrets.h` 或 `PubSubClient.h`

確認 `Secrets.h` 是用 Arduino IDE 新增分頁建立，且檔名大小寫正確、和 `mqtt_led_lab.ino` 位於同一草稿資料夾。`PubSubClient.h` 找不到時，到函式庫管理員安裝 `PubSubClient`，再重新啟動 Arduino IDE 或重新編譯。不要把真實設定直接貼進 `.ino`。

### 序列監控停在「嘗試連線 Wi-Fi」

先核對 `WIFI_SSID`、`WIFI_PASSWORD` 與受控 Wi-Fi 可用性。確認 ESP32 位於可連到該 Wi-Fi 的範圍內。不要為了測試把密碼印到序列監控；若無法判斷帳密是否正確，先回到先前已可用的 Wi-Fi 練習。

### 顯示 MQTT 連線失敗，或一直重試

依序核對：`MQTT_HOST` 是否只含主機名稱、連接埠是否為控制台顯示的 TLS `8883`、ESP32 帳密是否填入 `Secrets.h`、兩個 Topic 是否和自訂權限使用同一個 `YOUR_NAMESPACE`。帳密或權限剛更新時，等待後重新連線。不要改成未加密的 `1883`，也不要改成 `#` 權限。

### Web Client 能發布，卻看不到 `led=on` 或 `led=off`

先確認 Web Client 是**先**訂閱精確的 `.../status`，再發布到精確的 `.../led`。檢查 retain 是否關閉、兩端 Topic 是否只差最後一層、ESP32 是否真的顯示「已訂閱控制 Topic」。QoS `0` 可能遺失訊息；重新做一次「先訂閱、再發布」的單一測試，不要把舊訊息當成目前結果。

### LED 亮滅相反，或完全不動

不要先修改 MQTT 權限。回到 Blink 確認板載 LED 的實際腳位與高低電位；再只調整 `LED_PIN`、`LED_ON_LEVEL`、`LED_OFF_LEVEL`。不同相容板不保證使用 `GPIO 2` 或相同亮滅邏輯。

### 為什麼已使用 TLS，還不能長期使用 `setInsecure()`？

TLS 會加密資料，但 `setInsecure()` 讓 ESP32 跳過伺服器身分驗證，不能確認連到的是預期 Broker。本頁只用短期帳密和固定假資料，完成後應刪除或輪替帳密；長期使用時，改用 Root CA 憑證與 `setCACert()`，且不可在失敗時自動降回 `setInsecure()`。可先閱讀[公私鑰加密、數位簽章與中間人攻擊防範](../../附錄-公私鑰加密數位簽章與中間人攻擊防範.md)。

## 安全收尾

結束測試後：

1. 中斷 HiveMQ Web Client 連線。
2. 在 HiveMQ Cloud 刪除本頁建立的 ESP32 與 Web Client 測試帳密，或立刻輪替密碼。
3. 保留 `Secrets.h` 時，確認它只在個人電腦；若不再測試，刪除它。
4. 從 ESP32 拔除 USB，並確認程式沒有連接到繼電器、門鎖、市電或其他設備。

如果帳密、Wi-Fi 名稱、Broker 主機名稱或實際 namespace 曾經貼到公開位置，立即停止測試並在 HiveMQ Cloud 刪除或輪替受影響的 credential，再檢查公開紀錄是否需要移除。

## 重點整理

- `.../led` 是控制要求，`.../status` 是 ESP32 處理後的回傳；兩個方向分開，才能看清楚命令與結果。
- `PubSubClient` 必須設定 Broker 和 callback、成功連線後訂閱，並在 `loop()` 持續處理網路訊息。
- `.../+` 是受限範圍的權限折衷；程式與 Web Client 仍使用精確 Topic。
- `setInsecure()` 不驗證伺服器身分，只能留在受控、短期、假資料的練習；它不是長期 TLS 設計。
- LED、Broker 往返與目前控制台流程都必須以你的實作結果確認，不能用編譯成功取代實際觀察。

## 下一步

回到[前導技術概要](index.md)選擇其他主題，或回到[進階實務模擬專題](../index.md)對照 MQTT 在整體資料流中的位置。
