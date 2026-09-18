# 選做專題第 4 章：LCD 狀態顯示

讓 RC522 觸發練習事件，再由 LCD 顯示 Gateway 回覆的名單狀態。

## 你會學到什麼

- 將 RC522 與 LCD1602A 同時接到 ESP32，避開腳位衝突。
- 讓 LCD 顯示白名單、黑名單、未註冊與需查看四種狀態。
- 用 `event_id` 對應本次 MQTT 事件與 Gateway 回覆。
- 在逾時、斷線或資料不符時，讓 LCD 顯示未知而非舊結果。

## 開始前

- 已完成[第 1 章：RC522 RFID 與 SPI](../第1章/index.md)、[第 2 章：ESP32 MQTT 假信封事件](../第2章/index.md)與[第 3 章：Gateway 名單與最小紀錄](../第3章/index.md)。
- 需要 NodeMCU-32S 相容 ESP32、RC522、LCD1602A I2C 模組、杜邦線與可控制的 Wi-Fi。
- Arduino IDE 請選擇 `esp32 by Espressif Systems 3.3.11`、`NodeMCU-32S`，並安裝 `MFRC522 1.4.12`、`PubSubClient 2.8.0`、`LiquidCrystal I2C 1.1.2`。
- Gateway 必須在自己的電腦上運行，ESP32 與 Gateway 使用各自的短期 MQTT 帳密。

!!! warning "安全範圍"

    本章只顯示練習狀態，不讀取、顯示、保存或傳送卡片 UID，也不控制繼電器、門鎖、馬達或其他設備。`whitelisted` 只表示 Gateway 對練習信封回覆白名單狀態，不代表真人身分驗證。

!!! warning "帳密與網路"

    程式沿用第 2 章的 `setInsecure()`，只能用於受控 Wi-Fi 與可撤銷的短期帳密。不要公開 `secrets.h`、Wi-Fi 密碼、完整 topic、Broker 主機名稱、UID 或個人資料。網路來源、帳密或裝置用途無法確認時，請停止。

## 成功的樣子

```text
練習卡靠近 RC522
        ↓（不讀取 UID）
ESP32：送出 uid_envelope 與 event_id
        ↓ MQTT
Gateway：回覆 event_id、status、updated_at
        ↓ MQTT
LCD：只顯示本次 event_id 相符的狀態
```

| Gateway `status` | LCD 第一行 | LCD 第二行 |
| --- | --- | --- |
| `whitelisted` | `Card allowed` | `Status: WHITE` |
| `blacklisted` | `Card blocked` | `Status: BLACK` |
| `unregistered` | `Card rejected` | `Not registered` |
| `review_required` | `Card rejected` | `Review needed` |
| 逾時、斷線或內容不符 | `Status unknown` | `Check connection` |

## 步驟 1：接線

**目的：** 讓 RC522 的 SPI 與 LCD 的 I2C 各用不同腳位。

先拔除 ESP32 的 USB，再依下表接線。

| 元件腳位 | ESP32 腳位 | 用途／注意事項 |
| --- | --- | --- |
| RC522 `SDA`／`SS` | `GPIO 5` | SPI 片選，不是 I2C 的 SDA。 |
| RC522 `SCK` | `GPIO 18` | SPI 時脈。 |
| RC522 `MOSI` | `GPIO 23` | ESP32 傳送資料到 RC522。 |
| RC522 `MISO` | `GPIO 19` | RC522 傳回資料到 ESP32。 |
| RC522 `RST` | `GPIO 22` | 不與 LCD 共用。 |
| RC522 `3.3V` | `3.3V` | 不接 `5V`。 |
| RC522 `GND` | `GND` | 與 ESP32、LCD 共地。 |
| LCD `VCC` | `3.3V` | 本章只使用 3.3V 供電。 |
| LCD `GND` | `GND` | 與 RC522、ESP32 共地。 |
| LCD `SDA` | `GPIO 21` | I2C 資料線。 |
| LCD `SCL` | `GPIO 25` | 本專題的 I2C 時脈線，避開 RC522 `RST`。 |

!!! danger "不要把 I2C 訊號接到 5V"

    有些 LCD 背包在 5V 供電時會把 `SDA`、`SCL` 拉到 5V，可能傷害 ESP32 GPIO。本章只使用 3.3V 接法。出現發熱、焦味、USB 斷線、亂碼或接線不明時，立刻拔除 USB。

預期結果：RC522 的 `RST` 在 `GPIO 22`，LCD 的 `SCL` 在 `GPIO 25`，沒有腳位重複接用。

## 步驟 2：啟動 Gateway

**目的：** 讓 ESP32 的練習事件能取得名單狀態。

執行位置：PowerShell／電腦

```powershell
cd rfid-project\gateway
uv run python gateway.py
```

預期結果：終端機顯示 Gateway 已訂閱測試 topic。它必須維持在 `127.0.0.1`，不要改成區網或公開網址。

## 步驟 3：建立 LCD 整合程式

**目的：** 讀到練習卡時送出固定練習信封，並把相符的 Gateway 回覆顯示在 LCD。

第 2 章的 `secrets.h` 沿用 ESP32 專用的短期帳密與完整 topic。將它和下列檔案放在同一個 Arduino 草稿資料夾。

執行位置：Arduino IDE／ESP32
檔案：`rfid-project/esp32/rfid_lcd_gateway/rfid_lcd_gateway.ino`

```cpp
#include <SPI.h>
#include <Wire.h>
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <PubSubClient.h>
#include <LiquidCrystal_I2C.h>
#include <MFRC522.h>
#include <esp_system.h>

#include "secrets.h"


constexpr int LCD_SDA_PIN = 21;
constexpr int LCD_SCL_PIN = 25;
constexpr byte LCD_ADDRESS = 0x27;

constexpr byte RC522_SS_PIN = 5;
constexpr byte RC522_RST_PIN = 22;

constexpr unsigned long REPLY_TIMEOUT_MS = 8000;
constexpr unsigned long STATUS_FRESHNESS_MS = 60000;

const char DEVICE_ID[] = "demo-esp32-01";
const char PRACTICE_CIPHERTEXT[] = "practice-allowed";

char active_event_id[20] = "";
char practice_event[320] = "";

unsigned long reply_started_at = 0;
unsigned long status_shown_at = 0;

bool waiting_for_reply = false;
bool has_current_status = false;

LiquidCrystal_I2C lcd(LCD_ADDRESS, 16, 2);
MFRC522 rfid(RC522_SS_PIN, RC522_RST_PIN);
WiFiClientSecure secure_client;
PubSubClient mqtt_client(secure_client);


void show_lines(const char* first_line, const char* second_line) {
  lcd.setCursor(0, 0);
  lcd.print("                ");
  lcd.setCursor(0, 1);
  lcd.print("                ");

  lcd.setCursor(0, 0);
  lcd.print(first_line);
  lcd.setCursor(0, 1);
  lcd.print(second_line);
}


void show_unknown(const char* reason) {
  waiting_for_reply = false;
  has_current_status = false;
  show_lines("Status unknown", "Check connection");
  Serial.println(reason);
}


void show_gateway_status(const String& status) {
  waiting_for_reply = false;
  has_current_status = true;
  status_shown_at = millis();

  if (status == "whitelisted") {
    show_lines("Card allowed", "Status: WHITE");
  } else if (status == "blacklisted") {
    show_lines("Card blocked", "Status: BLACK");
  } else if (status == "unregistered") {
    show_lines("Card rejected", "Not registered");
  } else if (status == "review_required") {
    show_lines("Card rejected", "Review needed");
  } else {
    show_unknown("Gateway status was not recognised.");
    return;
  }

  Serial.print("Gateway status: ");
  Serial.println(status);
}


bool connect_wifi() {
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  for (int attempt = 1; attempt <= 40; attempt++) {
    if (WiFi.status() == WL_CONNECTED) {
      Serial.println("Wi-Fi connected.");
      return true;
    }
    delay(500);
  }

  return false;
}


bool connect_mqtt() {
  for (int attempt = 1; attempt <= 3; attempt++) {
    if (mqtt_client.connect(DEVICE_ID, MQTT_USERNAME, MQTT_PASSWORD) &&
        mqtt_client.subscribe(MQTT_TOPIC, 0)) {
      Serial.println("MQTT connected and subscribed.");
      return true;
    }

    mqtt_client.disconnect();
    delay(2000);
  }

  return false;
}


void on_message(char* topic, byte* payload, unsigned int length) {
  String message = "";
  for (unsigned int index = 0; index < length; index++) {
    message += static_cast<char>(payload[index]);
  }

  String compact = message;
  compact.replace(" ", "");
  compact.replace("\n", "");
  compact.replace("\r", "");

  // 同一個 topic 會收到自己的事件；帶有信封欄位的內容不是回覆。
  if (compact.indexOf("\"uid_envelope\"") >= 0) {
    return;
  }

  if (!waiting_for_reply) {
    return;
  }

  String prefix = String("{\"event_id\":\"") + active_event_id +
                  "\",\"status\":\"";
  String updated_marker = "\",\"updated_at\":\"";

  if (!compact.startsWith(prefix) || !compact.endsWith("\"}")) {
    show_unknown("Gateway reply format was not usable.");
    return;
  }

  int status_start = prefix.length();
  int marker_index = compact.indexOf(updated_marker, status_start);
  if (marker_index < status_start ||
      compact.indexOf(updated_marker, marker_index + 1) >= 0) {
    show_unknown("Gateway reply fields did not match.");
    return;
  }

  String status = compact.substring(status_start, marker_index);
  int updated_start = marker_index + updated_marker.length();
  int updated_end = compact.length() - 2;
  if (updated_start >= updated_end ||
      compact.indexOf("\"", updated_start) != updated_end) {
    show_unknown("Gateway reply had no update time.");
    return;
  }

  show_gateway_status(status);
}


bool make_practice_event() {
  unsigned long random_part = static_cast<unsigned long>(esp_random());
  snprintf(active_event_id, sizeof(active_event_id), "evt-%08lX", random_part);

  int written = snprintf(
      practice_event,
      sizeof(practice_event),
      "{\"event_id\":\"%s\",\"device_id\":\"%s\","
      "\"occurred_at\":\"2030-01-01T08:00:00+00:00\","
      "\"uid_envelope\":{\"key_id\":\"test-key\","
      "\"nonce\":\"fake-nonce\",\"ciphertext\":\"%s\","
      "\"tag\":\"fake-tag\"}}",
      active_event_id,
      DEVICE_ID,
      PRACTICE_CIPHERTEXT);

  return written > 0 && written < static_cast<int>(sizeof(practice_event));
}


void setup() {
  Serial.begin(115200);

  Wire.begin(LCD_SDA_PIN, LCD_SCL_PIN);
  lcd.init();
  lcd.backlight();
  show_lines("RFID LCD ready", "Tap practice card");

  SPI.begin();
  rfid.PCD_Init();

  secure_client.setInsecure();
  mqtt_client.setServer(MQTT_HOST, MQTT_PORT);
  mqtt_client.setCallback(on_message);
  mqtt_client.setBufferSize(512);

  if (!connect_wifi() || !connect_mqtt()) {
    show_unknown("Network connection was not established.");
    return;
  }

  Serial.println("RFID LCD Gateway ready.");
}


void loop() {
  if (WiFi.status() != WL_CONNECTED) {
    show_unknown("Wi-Fi disconnected.");
    delay(1000);
    return;
  }

  if (!mqtt_client.connected()) {
    show_unknown("MQTT disconnected.");
    delay(1000);
    return;
  }

  mqtt_client.loop();

  if (waiting_for_reply && millis() - reply_started_at >= REPLY_TIMEOUT_MS) {
    show_unknown("Gateway reply timed out.");
    return;
  }

  if (has_current_status && !waiting_for_reply &&
      millis() - status_shown_at >= STATUS_FRESHNESS_MS) {
    show_unknown("Displayed status expired.");
    return;
  }

  if (waiting_for_reply || !rfid.PICC_IsNewCardPresent() ||
      !rfid.PICC_ReadCardSerial()) {
    return;
  }

  // 讀卡只作為觸發，不讀取 rfid.uid 的內容。
  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();

  if (!make_practice_event()) {
    show_unknown("Practice event could not be created.");
    return;
  }

  has_current_status = false;
  show_lines("Card read", "Checking...");
  waiting_for_reply = true;

  if (mqtt_client.publish(MQTT_TOPIC, practice_event, false)) {
    reply_started_at = millis();
    Serial.print("Practice event sent: ");
    Serial.println(active_event_id);
  } else {
    show_unknown("Practice event could not be sent.");
  }
}
```

預期結果：上傳後，序列監控設為 `115200`，會依序顯示 Wi-Fi、MQTT 與 `RFID LCD Gateway ready.`。LCD 顯示：

```text
RFID LCD ready
Tap practice card
```

## 步驟 4：確認四種狀態

**目的：** 確認 LCD 顯示 Gateway 集中判斷的結果，而非自行判斷卡片。

先以程式預設的 `practice-allowed` 上傳並刷一次練習卡。預期結果：LCD 顯示 `Card allowed` 與 `Status: WHITE`，序列監控顯示 `Gateway status: whitelisted`。

接著只修改 `PRACTICE_CIPHERTEXT`，每改一次都重新上傳並以同一張練習卡觸發：

| 練習值 | 操作 | 預期 LCD 結果 |
| --- | --- | --- |
| `practice-blocked` | 刷一次 | `Card blocked`／`Status: BLACK` |
| `practice-new` | 五分鐘內刷第 1、2 次 | `Card rejected`／`Not registered` |
| `practice-new` | 五分鐘內刷第 3 次 | `Card rejected`／`Review needed` |

這些改動只改變練習信封的 `ciphertext`，不會讀取 UID。黑名單、白名單與重複次數都由 Gateway 處理；LCD 只是顯示結果。

## 步驟 5：確認未知狀態

**目的：** 無法可靠判斷時，不保留過去的顯示結果。

先取得一次任一有效狀態，再停止 Gateway，或等待超過 60 秒不再讀卡。

預期結果：LCD 變成：

```text
Status unknown
Check connection
```

Gateway 回覆超過 8 秒、Wi-Fi 中斷、MQTT 中斷、`event_id` 不相符或回覆欄位不符時，也會顯示這個畫面。

## 完成時，應能確認

- RC522 使用 `GPIO 5`、`18`、`19`、`23`、`22`；LCD 使用 `GPIO 21`、`25`，且兩者可以同時運作。
- 每次讀卡只送出固定練習信封與新的 `event_id`，沒有 UID。
- LCD 能顯示 `whitelisted`、`blacklisted`、`unregistered`、`review_required` 與未知狀態。
- 回覆逾時、斷線、事件代號不符或資料格式不符時，不會保留舊結果。
- 序列監控、Gateway CSV 與 LCD 顯示的事件代號及狀態一致。

## 常見問題

### LCD 顯示空白或找不到裝置

先拔除 USB，確認 LCD 的 `VCC` 接 `3.3V`、`GND` 共地、`SDA` 接 `GPIO 21`、`SCL` 接 `GPIO 25`。背光亮起卻沒有文字時，可慢慢調整背包的對比電阻；不要改接 5V。

### Arduino IDE 顯示找不到 `LiquidCrystal_I2C.h`

開啟「程式庫管理員」，安裝 `LiquidCrystal I2C` `1.1.2`，再確認程式沒有同時引用其他 LCD 函式庫。編譯時仍有問題時，先檢查 ESP32 core 是否為 `3.3.11`、開發板是否為 `NodeMCU-32S`。

### LCD 一直停在 `Checking...`

先確認 Gateway 正在本機執行，ESP32 與 Gateway 使用不同帳密、相同完整 topic。超過 8 秒後應顯示未知；若沒有，確認 `REPLY_TIMEOUT_MS` 與逾時判斷都已貼上。

### LCD 顯示未知，但 Gateway 有紀錄

確認 Gateway 回覆的 `event_id` 與 ESP32 序列監控印出的代號相同。ESP32 只接受 `event_id`、`status`、`updated_at` 三個欄位的回覆；欄位不完整或多出其他內容時，會顯示未知狀態。

## 安全收尾

按 `Ctrl+C` 停止 Gateway，再按 ESP32 的 `EN` 按鈕或拔除 USB。帳密不再使用時，應在服務端刪除或重建；本機的 `secrets.h`、`mqtt_settings.py` 與 CSV 只保存於受控資料夾，不公開傳送或發布。

## 重點整理

- RC522 只觸發事件，Gateway 才集中判斷名單。
- LCD 只顯示四種 Gateway 狀態與未知狀態，不控制實體設備。
- `event_id` 是本次事件的取件號碼，不是卡片資料。
- 練習信封、帳密與網路皆在受控範圍內使用。

## 下一步

想了解程式中 LCD、RC522、MQTT 與 Gateway 如何接力，可閱讀[LCD 狀態顯示程式導讀](lcd程式導讀.md)。
