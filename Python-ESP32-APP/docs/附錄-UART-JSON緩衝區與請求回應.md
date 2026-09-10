# 延伸選讀：UART JSON、緩衝區與請求／回應

> 延伸選讀｜對應主線：[UART：讓 Python 與 ESP32 雙向通訊](03-uart-python-esp32.md)、[整合專題：完成環境看板](08-整合專題.md)

這一頁說明第八節為何不用「讀到下一行就記錄」，而是使用 `read_now` 與 `source`。它幫助你判斷 UART 收到的是哪一筆資料，不需要修改主線程式。

## 讀完後你會知道

- 為什麼 UART 資料會暫留在電腦的接收緩衝區。
- 一行一筆 JSON 與換行在雙向通訊中扮演的角色。
- `periodic`、`request`、`status`、`error` 的用途。
- 為什麼清空緩衝區後，仍要檢查 `source`。

## 先知道這件事

第八節的 ESP32 每約 2 秒會送出一筆定時感測資料。若 Python 剛開始讀取，或網頁按鈕剛被按下，接收緩衝區中可能已經排著較早送出的資料。`readline()` 的工作只是讀取「下一個換行前的內容」，它不知道那筆資料是否正好屬於這次按鈕操作。

主線採用的協定是一行一筆 JSON：

```text
ESP32 → {"type":"sensor","source":"periodic",...}\n
Python → {"cmd":"read_now"}\n
ESP32 → {"type":"sensor","source":"request",...}\n
```

最後的 `\n` 是換行。它不是顯示用的空白，而是讓接收端知道一筆 JSON 到此結束的界線。

## 先分辨四種訊息

| `type` | 常見 `source` | 由誰送出 | 用途 |
| --- | --- | --- | --- |
| `sensor` | `periodic` | ESP32 | 每約 2 秒的觀察資料，方便在序列監控確認韌體運作 |
| `sensor` | `request` | ESP32 | 回覆 Python 的 `read_now`，可作為本次按鈕讀取的資料 |
| `status` | 無 | ESP32 | 回覆 LED 開啟或關閉命令 |
| `error` | `request` 或其他來源 | ESP32 | 說明 DHT11 讀取或命令處理失敗 |

不要只看 `type`。`periodic` 和 `request` 都可能是 `sensor`，但它們回答的是不同問題。

## 為什麼直接讀下一行不夠

以下是可能發生的時間順序：

```text
時間        ESP32 送出／Python 動作                         電腦接收緩衝區
10:00:00    ESP32 送出 periodic 感測資料                   [periodic]
10:00:02    ESP32 再送出 periodic 感測資料                 [periodic, periodic]
10:00:03    使用者按下「讀取並記錄資料」                   [periodic, periodic]
10:00:03    若 Python 直接呼叫 readline()                  讀到第一筆 periodic
```

這時讀到的資料格式正確，溫溼度看起來也合理，但它不是按下按鈕後要求的回覆。若直接寫入 CSV，圖表會把舊資料誤標成現在的讀取結果。

## ESP32：收到命令才標記為 `request`

來源片段：Arduino IDE／ESP32，`integrated_lcd_uart.ino`。

```cpp linenums="1" hl_lines="6-7"
void handleCommand(String command) {
  command.trim();

  // 略：處理 LED 開啟與關閉命令
  if (command == "{\"cmd\":\"read_now\"}") {
    readAndSendSensor("request");
  }
}
```

`read_now` 是 Python 送出的完整 JSON 命令。韌體辨識後呼叫和定時讀值相同的 `readAndSendSensor()`，但傳入不同的來源標記。

```cpp linenums="1" hl_lines="5-8"
void loop() {
  readSerialCommands();

  if (millis() - last_sensor_time >= SENSOR_INTERVAL_MS) {
    last_sensor_time = millis();
    readAndSendSensor("periodic");
  }
}
```

所以兩種 `sensor` JSON 的溫溼度欄位格式相同，只有 `source` 不同。這讓 Python 可以共用 JSON 解析方式，又能依目的選擇資料。

## Python：兩道防線，避免舊資料被誤收

來源片段：Python／電腦，`integrated_dashboard.py`。

```python linenums="1" hl_lines="2 4-6 11"
def read_sensor(self):
    self.ser.reset_input_buffer()
    message = json.dumps({"cmd": "read_now"}, separators=(",", ":")) + "\n"
    self.ser.write(message.encode("utf-8"))
    self.ser.flush()

    deadline = time.monotonic() + 4
    while time.monotonic() < deadline:
        data = self.read_json()
        if data and data.get("type") == "sensor" and data.get("source") == "request":
            return data
```

第一道防線是 `reset_input_buffer()`：先丟掉**已經到達電腦**、但尚未被程式讀走的舊資料。第二道防線是 `source == "request"`：即使清空後又恰好收到一筆新的定時資料，也不會把它當成這次請求的回覆。

`write()` 把命令送進序列埠，`flush()` 確保命令已交給作業系統傳送；接著迴圈在最長 4 秒內等待回覆。這個逾時是避免裝置斷線或韌體沒有回覆時，網頁永遠卡住。

## `read_json()` 為什麼可能回傳 `None`

來源片段：Python／電腦，`integrated_dashboard.py`。

```python linenums="1" hl_lines="2-5 7-10"
def read_json(self):
    raw = self.ser.readline().decode("utf-8", errors="replace").strip()
    if not raw:
        return None
    try:
        return json.loads(raw)
    except json.JSONDecodeError:
        return None
```

空行、逾時，或不是 JSON 的啟動訊息都不應讓整支程式停止。`read_json()` 回傳 `None` 後，`read_sensor()` 會繼續等待，直到收到符合條件的 `request` 回覆或超過期限。

## 何時使用、何時不要使用

| 情境 | 建議 |
| --- | --- |
| 網頁按鈕需要取得本次操作對應的資料 | 使用主線的 `read_now`／`source: request` 請求與篩選流程 |
| 只想在 Arduino 序列監控確認 ESP32 持續運作 | 觀察 `source: periodic` 即可 |
| 收到一筆格式正確的 `sensor` JSON | 先看 `source`，不要直接假定它是本次按鈕的資料 |
| 需要比 DHT11 本身的量測速度更快 | 更換感測器或重新設計取樣需求；協定標記無法消除感測器物理限制 |

## 常見誤解

### 「清空緩衝區後，就不需要 `source`」

不對。清空後到 ESP32 收到命令、送出回覆前，定時資料仍可能抵達。`source` 是讓 Python 識別回覆用途的依據。

### 「`source: request` 表示感測器完全沒有延遲」

不對。它只表示這是 ESP32 處理 `read_now` 後送出的回覆。DHT11 仍有自己的量測間隔與環境反應時間。

### 「`flush()` 是清除舊感測資料」

不是。`flush()` 處理的是 Python 要送出的命令；清除已收到資料的是 `reset_input_buffer()`。

### 「所有 `error` 都要立刻寫入 CSV」

不應該。`error` 不是溫溼度資料列。主線的 `read_sensor()` 會把本次請求相關的錯誤轉成狀態文字，而不寫入 CSV。

## 重點整理

- UART 接收緩衝區可能保留按鈕按下前的資料；`readline()` 不知道資料的新舊。
- 一行一筆 JSON 加上換行，讓接收端可判定一筆訊息的邊界。
- `reset_input_buffer()` 丟掉已到達的舊資料；`source: request` 再篩選本次請求回覆。
- 這個設計解決協定資料的新舊判斷，不會消除 DHT11 的量測限制。

## 回到主線

回到[UART：讓 Python 與 ESP32 雙向通訊](03-uart-python-esp32.md)或[整合專題：完成環境看板](08-整合專題.md)。
