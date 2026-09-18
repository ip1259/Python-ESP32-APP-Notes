# ESP32 MQTT 程式導讀

這一頁用第 2 章的假信封程式，整理 ESP32 如何連上 MQTT、送出事件與確認相符回覆。

!!! info "這是資訊補充"

    本頁不新增帳密、接線或操作步驟。請先依[第 2 章主頁](index.md)完成受限 MQTT 測試，再回來閱讀程式的分工。

## 先看整體順序

```text
建立假事件 → 連上 Wi-Fi → 連上 MQTT → 訂閱 topic
→ 發布假信封 → 收到訊息 → 略過自身事件或確認本次回覆
```

```text
ESP32 ── uid_envelope ──> MQTT 中介服務
ESP32 <─ event_id + status ─ MQTT 中介服務
```

ESP32 不會讀取 UID、查黑白名單或決定卡片能否通行。它只把固定假信封交給中介服務，並在收到相同 `event_id` 的 `whitelisted`（在白名單）練習回覆時顯示文字。

## 程式的五個部分

| 部分 | 主要內容 | 白話作用 |
| --- | --- | --- |
| 連線工具 | `WiFiClientSecure`、`PubSubClient` | 準備 Wi-Fi、TLS 與 MQTT 的溝通工具。 |
| 假事件 | `current_event_id`、`test_event`、`make_test_event()` | 建立不含 UID 的事件取件號碼與假信封文字。 |
| 收訊處理 | `on_message()` | 收到訊息時，判斷是自身假信封、相符回覆或其他內容。 |
| 連線處理 | `connect_wifi()`、`connect_mqtt()` | 在有限時間或有限次數內嘗試連線；失敗後停止。 |
| 啟動與持續收訊 | `setup()`、`loop()` | 啟動時送出一次假事件，之後持續交給 MQTT 處理收到的訊息。 |

## 假信封怎麼建立

```cpp
char current_event_id[20];
char test_event[320];

unsigned long random_value = static_cast<unsigned long>(esp_random());
snprintf(current_event_id, sizeof(current_event_id), "evt-%08lX", random_value);
```

`current_event_id` 像這次事件的取件號碼。它每次啟動可能不同，讓回覆能明確指向這一次訊息；它不是 UID，也不代表卡片身分。

接著 `snprintf()` 把固定假資料組成一段 JSON：

```json
{
  "event_id": "evt-XXXXXXXX",
  "device_id": "demo-esp32-01",
  "occurred_at": "2030-01-01T08:00:00+00:00",
  "uid_envelope": {
    "key_id": "test-key",
    "nonce": "fake-nonce",
    "ciphertext": "practice-allowed",
    "tag": "fake-tag"
  }
}
```

這個 `uid_envelope` 只是資料外形。`ciphertext` 等文字都是假值，不是加密後的真實 UID。

## 為什麼設定 `512` 位元組（bytes）緩衝區？

```cpp
constexpr uint16_t MQTT_BUFFER_SIZE = 512;
mqtt_client.setBufferSize(MQTT_BUFFER_SIZE);
```

MQTT 訊息像要放進一個信封袋，topic 和假信封內容都要放得下，因此設定 `512` 位元組的袋子大小。它只解決容量問題，不會保護訊息內容，也不會讓 MQTT 保證送達。

## 收到訊息時如何分流

```cpp
if (message.indexOf("\"uid_envelope\"") >= 0) {
  Serial.println("收到假信封事件，略過。");
  return;
}

if (message.indexOf(expected_event) >= 0 &&
    message.indexOf("\"status\":\"whitelisted\"") >= 0) {
  Serial.println("已收到本次事件的 whitelisted 狀態。");
  return;
}
```

同一個 topic 同時用於送出和接收，所以 ESP32 也會收到自己送出的假信封。第一段先把它略過。第二段再確認回覆同時有本次 `event_id` 與固定 `whitelisted` 狀態，避免把其他事件的訊息當成成功。

這是固定測試的最小比對，不是完整 JSON 解析。Gateway 在下一章會集中處理資料規則、名單與紀錄。

## 中斷時為什麼不一直重連

`connect_wifi()` 只等待有限時間，`connect_mqtt()` 最多嘗試三次。Wi-Fi 中斷時，程式顯示狀態未知並停止無限重試，避免裝置在錯誤設定或不穩網路下反覆連線。

```cpp
if (WiFi.status() != WL_CONNECTED) {
  Serial.println("Wi-Fi 已中斷，目前狀態未知；程式不會自動無限重連。");
  delay(5000);
  return;
}
```

## 常見誤解

### `publish()` 成功就代表 Gateway 已收到嗎？

不是。本章使用 QoS `0`；`publish()` 只表示 ESP32 已嘗試送出。必須看到相符 `event_id` 的回覆，才能確認這次練習往返完成。

### `setInsecure()` 是不是沒有加密？

不是。它仍使用 TLS 加密傳輸，但不確認伺服器身分。因此只能使用固定假資料與可刪除的短期帳密，不能傳送真實 UID 或其他敏感資料。

## 重點整理

- 假信封與事件取件號碼都由 ESP32 產生，沒有真實卡片資料。
- `setBufferSize(512)` 讓 topic 與假信封有足夠空間。
- `on_message()` 略過自身事件，再以 `event_id` 和 `whitelisted` 確認固定回覆。
- MQTT 中斷時顯示未知；Gateway 的規則判斷留在下一章。

## 下一步

回到[第 2 章主頁](index.md)，或閱讀[PubSubClient 函式庫介紹](pubsubclient函式庫介紹.md)。
