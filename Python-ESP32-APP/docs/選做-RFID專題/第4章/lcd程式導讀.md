# LCD 狀態顯示程式導讀

> 選讀｜對應：[第 4 章：LCD 狀態顯示](index.md)

本頁用第 4 章已成功運作的 `rfid_lcd_gateway.ino`，說明讀卡事件如何變成 LCD 上短短兩行、但可判斷狀態的文字。

## 讀完後你會知道

- LCD、RC522、Wi-Fi、MQTT 與 Gateway 在程式裡各自負責什麼。
- 為什麼程式不讀取 UID，仍能用練習卡觸發完整資料流。
- 為什麼回覆不對、太久沒回覆或狀態過期時，畫面要改成未知。

## 先知道這件事

把 LCD 想成門口的小看板：RC522 只負責發現「有人拿卡靠近」，Gateway 才用固定虛構資料決定顯示授權模擬或未授權模擬。小看板不自己決定誰可以通過，也不控制任何門。

主頁已完成接線與操作。本頁只在相同程式加入 `【編號】` 註解，不需要重新建立帳密或加入新硬體。

```text
練習卡 → RC522 發現新卡 → ESP32 固定假事件 → MQTT → Gateway
                                                        ↓
LCD ← 固定英文短句 ← ESP32 比對本次 event_id ← MQTT 回覆
```

## 完整程式導讀

執行位置：Arduino IDE／ESP32
來源檔案：`rfid_lcd_gateway.ino`

```cpp linenums="1"
// 【1】匯入讀卡、LCD、Wi-Fi 與 MQTT 需要的工具
#include <SPI.h>
#include <Wire.h>
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <PubSubClient.h>
#include <LiquidCrystal_I2C.h>
#include <MFRC522.h>

// 【2】秘密只放在自己的 secrets.h，不放進 Git 或教材
#include "secrets.h"

// 【3】集中列出硬體腳位與兩種等待時間
constexpr int LCD_SDA_PIN = 21;
constexpr int LCD_SCL_PIN = 25;
constexpr byte LCD_ADDRESS = 0x27;
constexpr byte RC522_SS_PIN = 5;
constexpr byte RC522_RST_PIN = 22;
constexpr unsigned long REPLY_TIMEOUT_MS = 8000;
constexpr unsigned long STATUS_FRESHNESS_MS = 60000;

// 【4】只使用固定虛構值；事件代號每次遞增，不是卡片資料
const char* DEVICE_ID = "esp32-a1";
const char* CARD_ID_MASKED = "CARD-****-42";
unsigned int nextEventNumber = 60;
char activeEventId[16] = "";
char testEvent[192] = "";
unsigned long replyStartedAt = 0;
unsigned long statusShownAt = 0;
bool waitingForReply = false;
bool hasCurrentStatus = false;
bool wifiWasConnected = false;
bool mqttWasConnected = false;

// 【5】建立三個會用到的物件：LCD、RC522 與 MQTT 用戶端
LiquidCrystal_I2C lcd(LCD_ADDRESS, 16, 2);
MFRC522 rfid(RC522_SS_PIN, RC522_RST_PIN);
WiFiClientSecure secureClient;
PubSubClient mqttClient(secureClient);

// 【6】先清空兩行，再印新文字，避免留下較短的舊字
void showLines(const char* firstLine, const char* secondLine) {
  lcd.setCursor(0, 0);
  lcd.print("                ");
  lcd.setCursor(0, 1);
  lcd.print("                ");
  lcd.setCursor(0, 0);
  lcd.print(firstLine);
  lcd.setCursor(0, 1);
  lcd.print(secondLine);
}

// 【7】把常用的等待與未知畫面做成兩個小幫手
void showWaiting() {
  showLines("RFID LCD ready", "Tap demo card");
}

void showUnknown(const char* message) {
  waitingForReply = false;
  hasCurrentStatus = false;
  showLines("Status unknown", "Check connection");
  Serial.println(message);
}

// 【8】收到 MQTT 訊息時，先整理成文字，再判斷是不是本次回覆
void onMessage(char* topic, byte* payload, unsigned int length) {
  String message = "";
  for (unsigned int index = 0; index < length; index++) {
    message += static_cast<char>(payload[index]);
  }

  String compact = message;
  compact.replace(" ", "");
  compact.replace("\n", "");
  compact.replace("\r", "");

  // 【8.1】同一個 topic 也會收到自己送出的讀卡事件，直接略過
  if (compact.indexOf("\"event_type\":\"card_read\"") >= 0) {
    return;
  }
  if (!waitingForReply) {
    return;
  }

  // 【8.2】回覆一定要對得上這次的事件代號
  const String expectedEvent =
      "\"event_id\":\"" + String(activeEventId) + "\"";
  if (compact.indexOf(expectedEvent) < 0) {
    showUnknown("Reply event ID did not match.");
    return;
  }

  // 【8.3】只接受完整的授權或未授權模擬摘要
  if (compact.indexOf("\"status\":\"authorized\"") >= 0 &&
      compact.indexOf("\"authorized\":true") >= 0 &&
      compact.indexOf("\"display_text\":") >= 0) {
    waitingForReply = false;
    hasCurrentStatus = true;
    statusShownAt = millis();
    showLines("Practice card OK", "Status: allowed");
    Serial.println("Gateway reply: authorized.");
    return;
  }

  if (compact.indexOf("\"status\":\"unauthorized\"") >= 0 &&
      compact.indexOf("\"authorized\":false") >= 0 &&
      compact.indexOf("\"display_text\":") >= 0) {
    waitingForReply = false;
    hasCurrentStatus = true;
    statusShownAt = millis();
    showLines("Practice card", "Not on demo list");
    Serial.println("Gateway reply: unauthorized.");
    return;
  }

  showUnknown("Gateway reply was not usable.");
}

// 【9】先連 Wi-Fi，再最多嘗試三次 MQTT 連線與訂閱
void connectWifi() {
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
  }
  wifiWasConnected = true;
  Serial.println("Wi-Fi connected.");
}

bool connectMqtt() {
  for (int attempt = 1; attempt <= 3; attempt++) {
    if (mqttClient.connect(DEVICE_ID, MQTT_USERNAME, MQTT_PASSWORD)) {
      if (mqttClient.subscribe(MQTT_TOPIC, 0)) {
        mqttWasConnected = true;
        Serial.println("MQTT connected and subscribed.");
        return true;
      }
      mqttClient.disconnect();
    }
    delay(2000);
  }
  return false;
}

// 【10】開機時啟動 LCD、RC522、Wi-Fi 與 MQTT
void setup() {
  Serial.begin(115200);
  Wire.begin(LCD_SDA_PIN, LCD_SCL_PIN);
  lcd.init();
  lcd.backlight();
  SPI.begin();
  rfid.PCD_Init();
  showWaiting();

  // 僅限短期固定假資料：傳輸加密，但不驗證伺服器身分。
  secureClient.setInsecure();
  mqttClient.setServer(MQTT_HOST, MQTT_PORT);
  mqttClient.setCallback(onMessage);
  connectWifi();

  if (!connectMqtt()) {
    showLines("Network offline", "Status unknown");
    Serial.println("MQTT connection failed.");
    return;
  }
  Serial.println("RFID LCD Gateway ready.");
}

// 【11】持續巡邏：先處理中斷、回覆與過期，再等待新卡
void loop() {
  if (WiFi.status() != WL_CONNECTED) {
    waitingForReply = false;
    hasCurrentStatus = false;
    if (wifiWasConnected) {
      wifiWasConnected = false;
      Serial.println("Wi-Fi disconnected; status is unknown.");
    }
    showLines("Network offline", "Status unknown");
    delay(1000);
    return;
  }

  if (!mqttClient.connected()) {
    if (mqttWasConnected) {
      mqttWasConnected = false;
      showUnknown("MQTT disconnected.");
    }
    delay(1000);
    return;
  }

  mqttClient.loop();

  if (waitingForReply && millis() - replyStartedAt >= REPLY_TIMEOUT_MS) {
    showUnknown("Gateway reply timed out.");
    return;
  }

  if (hasCurrentStatus && !waitingForReply &&
      millis() - statusShownAt >= STATUS_FRESHNESS_MS) {
    showUnknown("Displayed status expired.");
    return;
  }

  if (waitingForReply || !rfid.PICC_IsNewCardPresent() ||
      !rfid.PICC_ReadCardSerial()) {
    return;
  }

  // 【12】發現新卡後，只建立固定假事件，絕不讀取 UID
  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
  hasCurrentStatus = false;
  snprintf(activeEventId, sizeof(activeEventId),
           "evt-%03u", nextEventNumber);
  nextEventNumber++;
  snprintf(testEvent, sizeof(testEvent),
           "{\"event_type\":\"card_read\","
           "\"card_id_masked\":\"%s\","
           "\"device_id\":\"esp32-a1\","
           "\"event_id\":\"%s\","
           "\"occurred_at\":\"2026-09-18T10:00:00+08:00\"}",
           CARD_ID_MASKED, activeEventId);
  showLines("Card read", "Checking...");
  waitingForReply = true;

  if (mqttClient.publish(MQTT_TOPIC, testEvent, false)) {
    replyStartedAt = millis();
    Serial.printf("Fixed practice event sent: %s\n", activeEventId);
  } else {
    showUnknown("Practice event send failed.");
  }
}
```

## 依資料流閱讀

先看【1】到【5】。這些內容像先準備一張小看板、讀卡器與送信工具。`Wire.begin(21, 25)` 在【10】告訴 ESP32：「LCD 的兩條線在 `GPIO 21`、`GPIO 25`」，因此不會碰到 RC522 的 `GPIO 22`。

接著看【6】、【7】。LCD 只有兩行、每行 16 格，`showLines()` 先把舊字擦掉，就像白板先擦乾淨再寫新訊息。`showUnknown()` 則是統一的安全畫面：只要現在不能可靠判斷，就不保留先前的結果。

【8】像收信櫃。它先略過 ESP32 自己送出的讀卡事件，再檢查回覆上的 `event_id` 是否和手上的「取件號碼」相同。對得上且欄位完整，才會更新 LCD。

最後看【11】、【12】。`loop()` 像一直巡邏：先看 Wi-Fi、MQTT、回覆時間和畫面是否過期，最後才等新卡。讀到卡時，【12】只建立 `CARD-****-42` 這種固定假資料；程式沒有使用 `rfid.uid`，所以不會把卡片識別資訊送出去。

## 出錯時先看哪裡

| 看到的現象 | 第一個檢查位置 | 可先判斷的範圍 |
| --- | --- | --- |
| LCD 有等待畫面，但讀卡沒有反應 | 【10】與【12】的 RC522 初始化、接線與練習卡 | 問題還沒進入 MQTT。 |
| 停在 `Checking...` 後變未知 | 【11】的 8 秒等待與 Gateway | ESP32 已送出事件，但沒有收到可用回覆。 |
| 顯示 `Reply event ID did not match.` | 【8.2】 | 收到的訊息不是這一次事件的回覆。 |
| 顯示 `Network offline` | 【9】到【11】 | Wi-Fi 或 MQTT 目前沒有可靠連線。 |
| 顯示 `Displayed status expired.` | 【11】的 60 秒規則 | 舊畫面已不代表現在狀態。 |
| Gateway 顯示「收到無法使用的資料」 | 【8.1】與第 3 章 Gateway | Gateway 收到自己的回覆並安全略過，屬預期行為。 |

## 常見誤解

### 「讀到卡就代表 LCD 會顯示授權」

不是。讀卡只讓 ESP32 發出固定假事件。LCD 要收到 Gateway 的相符回覆，才會顯示 `Status: allowed` 或未授權文字；沒有回覆時會顯示未知。

### 「`event_id` 是卡片編號」

不是。它像包裹的取件號碼，每次事件會產生新的 `evt-060`、`evt-061`。它用來確認回覆屬於這一次讀卡，不能辨識卡片或人。

### 「60 秒後顯示未知代表未授權」

不是。未知表示目前沒有可靠資料，和 `authorized: False` 的未授權模擬不同。兩種情況都不會觸發任何實體動作。

## 重點整理

- RC522 觸發事件，Gateway 回覆固定教學結果，LCD 只負責顯示短句。
- `event_id` 讓程式辨認本次回覆；不相符或欄位不完整就顯示未知。
- 8 秒沒回覆與 60 秒沒有新狀態，都會清除舊結果。
- 程式不讀取 UID，且只可搭配短期帳密與受控網路使用。

## 回到專題

回到[第 4 章主頁](index.md)，或繼續閱讀[第 5 章：Thunkable、ngrok 與手機無線控制](../../選做-MIT-App-Inventor-ngrok與手機無線控制.md)。
