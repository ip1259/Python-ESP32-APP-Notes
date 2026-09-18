# 選做專題第 4 章：LCD 狀態顯示

讓 RC522 讀卡、MQTT Gateway 與 LCD1602A 連成一條看得見的狀態流程。

## 你會學到什麼

- 將 RC522 與 LCD1602A 同時接到 ESP32，避免兩者使用同一個 GPIO。
- 讓 LCD 顯示等待讀卡、授權模擬、未授權模擬與未知狀態。
- 用固定虛構資料經 MQTT Gateway 取得結果，不讀取或傳送卡片 UID。
- 讓舊結果在 60 秒後失效，避免把過去的畫面當成現在的狀態。

## 開始前

- 已完成[第 1 章：RC522 RFID 與 SPI](../第1章/index.md)、[第 2 章：ESP32 MQTT 雙向訊息](../第2章/index.md)與[第 3 章：Python MQTT Gateway 與 FastAPI 初步概念](../第3章/index.md)。
- 需要 NodeMCU-32S 相容 ESP32、RC522、LCD1602A I2C 模組、杜邦線，以及已可使用的受控 Wi-Fi 與短期 MQTT 測試帳密。
- Arduino IDE 需已有 `MFRC522`、`PubSubClient` 與 `LiquidCrystal I2C` 函式庫。前兩者沿用第 1、2 章的已驗證版本。
- 只使用本人或明確獲授權的練習卡。程式不會讀取、顯示、保存或傳送 UID。

!!! warning "這是選做的狀態顯示"

    本章只顯示固定虛構資料的教學結果，不是身分驗證，也不連接繼電器、電磁鎖、馬達或其他實體設備。LCD 出現 `Status: allowed` 只表示 Gateway 對 `CARD-****-42` 這個虛構值回覆授權模擬。

!!! warning "網路與帳密限制"

    程式沿用第 2 章的 `setInsecure()`。它會加密傳輸內容，但不驗證伺服器身分；因此只能使用可控制的 Wi-Fi、固定假資料與可撤銷的短期帳密。不得公開 `secrets.h`、Wi-Fi 密碼、topic、Broker 主機、token、UID 或任何個人資料。網路來源、帳密或裝置用途無法確認時，請停止本章。

## 成功的樣子

讀取練習卡後，LCD 會依序顯示：

```text
Card read
Checking...
```

當 Gateway 對固定虛構值 `CARD-****-42` 回覆授權模擬時，畫面變成：

```text
Practice card OK
Status: allowed
```

資料無法使用、Gateway 停止、網路中斷，或已超過 60 秒沒有新事件時，畫面必須改為：

```text
Status unknown
Check connection
```

## 步驟 1：確認這一章的腳位

**目的：** 保留已完成的 RC522 接線，讓 LCD 改用不衝突的 I2C 腳位。

主線 LCD 使用 `GPIO 21`／`GPIO 22`，但 RFID 的 RC522 已使用 `GPIO 22` 作為 `RST`。本章因此讓 LCD 的 I2C 時脈改接 `GPIO 25`。這是 RFID 專題的專用接法，不會改變主線 LCD 教材。

先拔除 ESP32 的 USB，再依下表接線。

| 元件腳位 | ESP32 腳位 | 用途／注意事項 |
| --- | --- | --- |
| RC522 `SDA`／`SS` | `GPIO 5` | RC522 的 SPI 片選，不是 I2C 的 SDA。 |
| RC522 `SCK` | `GPIO 18` | SPI 時脈。 |
| RC522 `MOSI` | `GPIO 23` | ESP32 傳送資料到 RC522。 |
| RC522 `MISO` | `GPIO 19` | RC522 傳回資料到 ESP32。 |
| RC522 `RST` | `GPIO 22` | 沿用第 1 章，不與 LCD 共用。 |
| RC522 `3.3V` | 已確認穩定的 `3.3V` | 不接 `5V`。若第 1 章需外接穩壓 `3.3V`，沿用該配置，並與 ESP32 共地。 |
| RC522 `GND` | `GND` | 與 ESP32、LCD 共地。 |
| LCD `VCC` | ESP32 `3V3` | 本章以 `3.3V` 供電。 |
| LCD `GND` | `GND` | 與 RC522、ESP32 共地。 |
| LCD `SDA` | `GPIO 21` | I2C 資料線。 |
| LCD `SCL` | `GPIO 25` | RFID 專題的 I2C 時脈線。 |

!!! danger "不要把 I2C 訊號接到 5V"

    LCD 即使以 5V 供電後看似可顯示文字，也可能把 `SDA`、`SCL` 拉到 5V，傷害 ESP32 GPIO。本章只使用 3.3V LCD 接法。出現發熱、焦味、USB 斷線、亂碼或接線不明時，立刻拔除 USB 並停止。

預期結果：RC522 的 `RST` 保持在 `GPIO 22`，LCD 的 `SCL` 接在 `GPIO 25`，沒有任何腳位重複接用。

## 步驟 2：啟動本機 Gateway

**目的：** 讓 ESP32 的固定假事件可以得到受控回覆。

在第 3 章建立的 `rfid-gateway` 資料夾啟動 Gateway。`mqtt_settings.py` 必須保留在自己的電腦，不要貼到程式、教材、截圖或 Git。

執行位置：PowerShell／電腦

```powershell
uv run python gateway.py
```

預期結果：終端機顯示 `Gateway 已訂閱專屬 topic` 與 `Uvicorn running on http://127.0.0.1:8000`。Gateway 只監聽 `127.0.0.1`；不要改成區網位址或公開網址。

## 步驟 3：建立整合程式

**目的：** 在同一支 ESP32 程式中讀取練習卡、送出固定虛構事件、接收 Gateway 回覆並更新 LCD。

第 2 章的 `secrets.h` 沿用 ESP32 專用的短期帳密與單一完整 topic。它必須和本檔案放在同一個 Arduino 草稿資料夾，但不能公開。

執行位置：Arduino IDE／ESP32
檔案：`rfid_lcd_gateway.ino`

```cpp
#include <SPI.h>
#include <Wire.h>
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <PubSubClient.h>
#include <LiquidCrystal_I2C.h>
#include <MFRC522.h>

#include "secrets.h"

constexpr int LCD_SDA_PIN = 21;
constexpr int LCD_SCL_PIN = 25;
constexpr byte LCD_ADDRESS = 0x27;

constexpr byte RC522_SS_PIN = 5;
constexpr byte RC522_RST_PIN = 22;

constexpr unsigned long REPLY_TIMEOUT_MS = 8000;
constexpr unsigned long STATUS_FRESHNESS_MS = 60000;

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

LiquidCrystal_I2C lcd(LCD_ADDRESS, 16, 2);
MFRC522 rfid(RC522_SS_PIN, RC522_RST_PIN);
WiFiClientSecure secureClient;
PubSubClient mqttClient(secureClient);

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

void showWaiting() {
  showLines("RFID LCD ready", "Tap demo card");
}

void showUnknown(const char* message) {
  waitingForReply = false;
  hasCurrentStatus = false;
  showLines("Status unknown", "Check connection");
  Serial.println(message);
}

void onMessage(char* topic, byte* payload, unsigned int length) {
  String message = "";

  for (unsigned int index = 0; index < length; index++) {
    message += static_cast<char>(payload[index]);
  }

  String compact = message;
  compact.replace(" ", "");
  compact.replace("\n", "");
  compact.replace("\r", "");

  if (compact.indexOf("\"event_type\":\"card_read\"") >= 0) {
    return;
  }

  if (!waitingForReply) {
    return;
  }

  const String expectedEvent =
    "\"event_id\":\"" + String(activeEventId) + "\"";

  if (compact.indexOf(expectedEvent) < 0) {
    showUnknown("Reply event ID did not match.");
    return;
  }

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

void setup() {
  Serial.begin(115200);

  Wire.begin(LCD_SDA_PIN, LCD_SCL_PIN);
  lcd.init();
  lcd.backlight();

  SPI.begin();
  rfid.PCD_Init();
  showWaiting();

  // 只限短期、固定假資料測試：傳輸加密，但不驗證伺服器身分。
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

  if (waitingForReply &&
      millis() - replyStartedAt >= REPLY_TIMEOUT_MS) {
    showUnknown("Gateway reply timed out.");
    return;
  }

  if (hasCurrentStatus &&
      !waitingForReply &&
      millis() - statusShownAt >= STATUS_FRESHNESS_MS) {
    showUnknown("Displayed status expired.");
    return;
  }

  if (waitingForReply ||
      !rfid.PICC_IsNewCardPresent() ||
      !rfid.PICC_ReadCardSerial()) {
    return;
  }

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

### 程式裡的幾個重要部分

| 名稱 | 做什麼 | 這一章怎麼使用 |
| --- | --- | --- |
| `Wire.begin(21, 25)` | 指定 I2C 的 SDA、SCL 腳位 | LCD 使用 `GPIO 21` 與 `GPIO 25`，避開 RC522 的 `GPIO 22`。 |
| `showLines()` | 清除並更新 LCD 的兩行文字 | 每行先補滿空白，避免新文字較短時留下舊字。 |
| `rfid.PICC_ReadCardSerial()` | 確認有一張新卡可讀取 | 只把它當成觸發條件；程式不讀取 `rfid.uid`。 |
| `mqttClient.publish()` | 發布一筆 MQTT 訊息 | 送出固定虛構的遮蔽值與新的事件代號。 |
| `onMessage()` | 收到 MQTT 訊息時執行 | 只接受本次事件代號相符且狀態完整的 Gateway 回覆。 |
| `STATUS_FRESHNESS_MS` | 設定狀態可保留多久 | 本章以 60 秒做測試；時間到後改顯示未知。 |

預期結果：上傳後序列監控設為 `115200`，會依序出現：

```text
Wi-Fi connected.
MQTT connected and subscribed.
RFID LCD Gateway ready.
```

LCD 顯示：

```text
RFID LCD ready
Tap demo card
```

## 步驟 4：讀取固定授權模擬結果

**目的：** 確認讀卡事件能經 Gateway 回到 LCD。

將練習卡靠近 RC522 一次，然後移開。程式只會送出 `CARD-****-42`，不會送出卡片的 UID。

預期結果：LCD 先顯示 `Card read`、`Checking...`，接著顯示 `Practice card OK`、`Status: allowed`。序列監控會顯示類似：

```text
Fixed practice event sent: evt-060
Gateway reply: authorized.
```

事件代號每次遞增，例如 `evt-060`、`evt-061`。它用來對照本次送出的事件與收到的回覆，不是卡片資料。

## 步驟 5：切換成未授權模擬

**目的：** 確認 Gateway 回覆未授權時，LCD 不會顯示成授權。

在程式中將：

```cpp
const char* CARD_ID_MASKED = "CARD-****-42";
```

改為：

```cpp
const char* CARD_ID_MASKED = "CARD-****-99";
```

重新上傳後，再以同一張練習卡觸發一次。這個改動只改變送往 Gateway 的固定虛構值，不會讀取卡片 UID。

預期結果：LCD 顯示：

```text
Practice card
Not on demo list
```

序列監控顯示 `Gateway reply: unauthorized.`。完成後可改回 `CARD-****-42`，回到授權模擬。

## 步驟 6：觀察未知狀態

**目的：** 確認無法可靠判斷時，不會保留先前的結果。

先以 `CARD-****-42` 取得一次授權模擬，之後不要再讀卡，保持 Gateway 運作約 60 秒。

預期結果：LCD 改為：

```text
Status unknown
Check connection
```

序列監控顯示 `Displayed status expired.`。若 Gateway 停止或回覆超過 8 秒仍未到達，也會顯示相同的未知狀態。

## 完成時，應能確認

- RC522 保持 `RST GPIO 22`，LCD 使用 `SDA GPIO 21`、`SCL GPIO 25`，兩個模組可同時運作。
- LCD 能顯示等待、授權模擬、未授權模擬與未知四種狀態。
- 每次讀卡都只送出 `CARD-****-NN` 形式的固定虛構值與新的事件代號，沒有 UID。
- Gateway 停止、回覆逾時、Wi-Fi 中斷或狀態超過 60 秒時，LCD 不會保留舊的授權或未授權結果。
- 序列監控與 LCD 的狀態一致，且不出現帳密、topic、Broker 位址或 UID。

## 常見問題

### LCD 顯示空白或找不到 I2C 裝置

先拔除 USB，確認 LCD 的 `VCC` 接 `3V3`、`GND` 共地、`SDA` 接 `GPIO 21`、`SCL` 接 `GPIO 25`。本章的 `SCL` 不是主線 LCD 的 `GPIO 22`。背光亮起但沒有文字時，可慢慢調整 I2C 背包的對比電阻；不要改接 5V。

### LCD 可顯示等待文字，但讀卡後沒有反應

先回到[第 1 章](../第1章/index.md)確認 RC522 的 `3.3V`、`GND`、`GPIO 5`、`18`、`19`、`23`、`22` 接線。若第 1 章使用外接穩壓 `3.3V` 才能穩定讀卡，本章也要維持同一個 RC522 供電方式與共地。

### LCD 一直停在 `Checking...`

先確認第 3 章 Gateway 正在本機執行，且 ESP32 與 Gateway 使用不同的短期帳密、同一個完整 topic。超過 8 秒後程式應顯示未知；若沒有，確認程式中的 `REPLY_TIMEOUT_MS` 與逾時判斷都完整貼上。不要把 Gateway 改成公開網址或使用共用帳密排除問題。

### 顯示 `Network offline` 或 `Status unknown`

這表示程式沒有可靠的目前狀態。確認 Wi-Fi、短期 ESP32 帳密與單一 topic 權限；帳密刪除、Broker 無法連線或 Wi-Fi 中斷時都應出現這個畫面。恢復受控 Wi-Fi 與有效帳密後，按 ESP32 的 `EN` 按鈕重新開始，不要讓程式無限快速重連。

### Gateway 顯示「收到無法使用的資料，已略過」

ESP32 與 Gateway 在本專題使用同一個 topic。Gateway 也會收到自己發布的回覆，但那不是讀卡事件，所以會安全略過。只要 ESP32 同時得到相符事件代號的 `authorized` 或 `unauthorized` 回覆，這是預期現象。

## 安全收尾

按 `Ctrl+C` 停止 Gateway，再按 ESP32 的 `EN` 按鈕或拔除 USB。確認 `secrets.h`、`mqtt_settings.py` 與 CSV 只留在自己的受控資料夾，不上傳或截圖。刪除或重建 ESP32、Gateway 與網站端的短期測試帳密；確認不再使用後，刪除本機的秘密設定檔與測試 CSV。

## 重點整理

- RFID 專題的 LCD 改用 `GPIO 21`／`GPIO 25`，因此可與 RC522 的 `RST GPIO 22` 共存。
- RC522 只觸發固定假事件；Gateway 才決定授權模擬或未授權模擬。
- LCD 只顯示有限的英文短句，避免 16×2 螢幕文字殘留或中文亂碼。
- 無效回覆、斷線、逾時與過期都必須改為未知，不能沿用舊結果。

## 下一步

想逐段了解 ESP32、LCD、RC522 與 Gateway 如何分工，可閱讀[LCD 狀態顯示程式導讀](lcd程式導讀.md)；這不影響本頁的操作結果。

接著閱讀[專題第 5 章：Thunkable、ngrok 與手機無線控制](../../選做-MIT-App-Inventor-ngrok與手機無線控制.md)。
