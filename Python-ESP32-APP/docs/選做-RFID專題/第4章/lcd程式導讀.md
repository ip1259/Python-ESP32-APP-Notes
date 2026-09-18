# LCD 狀態顯示程式導讀

這頁說明 ESP32 如何把 Gateway 的簡短回覆換成 LCD 上兩行容易辨認的文字。

## 你會學到什麼

- 分辨 RC522、ESP32、Gateway 與 LCD 各自的工作。
- 看懂 `event_id` 為什麼能避免顯示到別次事件的回覆。
- 了解不可靠的結果為什麼一定要顯示未知。

## 開始前

- 已完成[第 4 章：LCD 狀態顯示](index.md)。
- 本頁是資訊補充，不需要新增接線或修改程式。

## 四個角色如何合作

```text
練習卡 → RC522 → ESP32 → MQTT → Gateway
                                  ↓
LCD ← 狀態短句 ← ESP32 ← MQTT 回覆
```

RC522 只負責發現有新卡靠近。ESP32 不讀取 `rfid.uid`，而是建立固定的練習 `uid_envelope`。Gateway 才檢查名單並回覆狀態，LCD 最後只把結果顯示出來。

## LCD 為什麼只顯示短句

16×2 LCD 每行只有 16 個字元位置。程式把四種狀態換成固定的英文短句，避免文字太長或留下舊字：

```cpp
if (status == "whitelisted") {
  show_lines("Card allowed", "Status: WHITE");
} else if (status == "blacklisted") {
  show_lines("Card blocked", "Status: BLACK");
} else if (status == "unregistered") {
  show_lines("Card rejected", "Not registered");
} else if (status == "review_required") {
  show_lines("Card rejected", "Review needed");
}
```

`show_lines()` 每次先把兩行清成空白，再寫入新文字。這像白板先擦乾淨，避免新句子較短時殘留前一次的字。

## `event_id` 像取件號碼

同一個 MQTT topic 可能同時出現 ESP32 自己送出的練習事件、Gateway 回覆，或其他裝置的訊息。程式先略過含有 `uid_envelope` 的事件，接著只接受目前 `active_event_id` 開頭的回覆：

```cpp
String prefix = "{\"event_id\":\"" + String(active_event_id) +
                "\",\"status\":\"";

if (!compact.startsWith(prefix)) {
  show_unknown("Gateway reply format was not usable.");
  return;
}
```

這就像取件時先比對取件號碼。號碼不相同，或回覆不是 `event_id`、`status`、`updated_at` 這三個欄位，就不應更新 LCD。

## 為什麼未知比舊結果好

程式在 MQTT 回覆逾時、Wi-Fi 中斷、MQTT 中斷、格式不符或顯示超過 60 秒時，都呼叫同一個函式：

```cpp
void show_unknown(const char* reason) {
  waiting_for_reply = false;
  has_current_status = false;
  show_lines("Status unknown", "Check connection");
  Serial.println(reason);
}
```

這可以避免 Gateway 已停止時，LCD 還留著先前的白名單或黑名單畫面。未知不是錯誤結果，而是誠實表示現在沒有可靠資料。

## 讀卡只是觸發

程式有呼叫 `rfid.PICC_ReadCardSerial()`，目的是確認有新卡可觸發事件；接著立刻停止卡片通訊：

```cpp
rfid.PICC_HaltA();
rfid.PCD_StopCrypto1();
```

程式沒有使用 `rfid.uid`，因此不會將 UID 印到序列監控、送進 MQTT 或顯示在 LCD。

## 重點整理

- Gateway 決定 `status`；ESP32 只對應狀態短句；LCD 只負責顯示。
- `event_id` 讓 ESP32 不會採用別次事件的回覆。
- 資料不可靠時，LCD 會回到未知，而不是保留舊結果。
- 所有狀態都屬於練習資料流，不控制任何實體設備。

## 下一步

回到[第 4 章主頁](index.md)確認四種狀態與未知狀態的操作結果。
