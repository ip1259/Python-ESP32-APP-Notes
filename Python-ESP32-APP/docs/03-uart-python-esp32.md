# UART：讓 Python 與 ESP32 雙向通訊

這一頁會讓 ESP32 定時傳送 DHT11 溫溼度資料給 Python，並讓 Python 送出 JSON 命令控制 ESP32 的 LED。

## 你會學到什麼

- 使用 USB UART 在 ESP32 與 Python 間傳送文字資料。
- 用一行一筆 JSON 傳送 DHT11 溫溼度資料。
- 用 Python 控制 ESP32 的 LED 開關。
- 依序排除 baud rate、COM 埠占用、JSON 與 DHT11 讀值問題。

## 開始前

| 項目 | 需求 |
| --- | --- |
| 前置教材 | [課程準備：認識 AIoT 資料流](02-課程準備.md) |
| ESP32 | NodeMCU-32S 相容開發板與 USB 資料線 |
| 感測器 | DHT11 溫溼度模組 |
| Python 套件 | `pyserial`，已在第一篇教材安裝 |
| Arduino 函式庫 | `DHT sensor library`、`Adafruit Unified Sensor` |

!!! warning "關閉序列監控視窗"
    Python 要使用 COM 埠前，請先關閉 Arduino IDE 的序列監控視窗。同一時間只能有一個程式開啟同一個 COM 埠。

## 成功的樣子

資料流如下：

```text
DHT11 → ESP32 → USB UART → Python 終端機
                         ↓
              JSON LED 命令 ← Python
```

Python 顯示感測資料時，畫面類似：

```text
溫度：25.0 °C，濕度：60.0 %
溫度：25.0 °C，濕度：60.0 %
LED 狀態：開啟
```

## 接線

先拔除 ESP32 的 USB 線，再依下表接線。本頁使用 GPIO 4 讀取 DHT11、GPIO 2 控制板載 LED。

| 元件腳位 | ESP32 腳位 | 用途／注意事項 |
| --- | --- | --- |
| DHT11 VCC | 3V3 | 使用 3.3V 供電 |
| DHT11 GND | GND | 必須與 ESP32 共地 |
| DHT11 DATA | GPIO 4 | 讀取溫度與濕度 |
| 板載 LED | GPIO 2 | 不需另外接線；若你的板子 LED 行為相反，請先詢問教師或助教 |

!!! tip "DHT11 模組與裸 DHT11 不一樣"
    本課使用的是 DHT11 模組，通常已包含需要的上拉電阻。若你拿到的是只有 4 根腳的裸感測器，請先詢問教師或助教，不要直接照本頁接線。

## 步驟 1：安裝 Arduino DHT11 函式庫

**目的：** 讓 ESP32 可以讀取 DHT11。

執行位置：Arduino IDE／電腦

1. 開啟「工具 → 管理程式庫」。
2. 搜尋並安裝 `DHT sensor library`，作者是 Adafruit。
3. 依安裝提示，同時安裝 `Adafruit Unified Sensor`。

預期結果：Arduino IDE 的「程式庫管理員」顯示兩個程式庫都已安裝。

!!! note "為什麼要安裝兩個程式庫？"
    `DHT sensor library` 用來讀取 DHT11；它還需要 `Adafruit Unified Sensor` 作為相依程式庫。Adafruit 的 [DHT sensor library 說明](https://github.com/adafruit/DHT-sensor-library) 也列出這個相依關係。

## 步驟 2：先驗證 UART 與固定 JSON

**目的：** 還沒接 DHT11 前，先確認 ESP32 能傳送 JSON 文字。這能把「通訊問題」和「感測器問題」分開。

執行位置：Arduino IDE／ESP32  
檔案：`uart_fixed_json.ino`

```cpp
void setup() {
  Serial.begin(115200);
}

void loop() {
  Serial.println("{\"type\":\"sensor\",\"temp\":25.0,\"humidity\":60.0}");
  delay(2000);
}
```

上傳後，先用 Arduino IDE 序列監控視窗確認輸出：

```json
{"type":"sensor","temp":25.0,"humidity":60.0}
```

確認後，**關閉序列監控視窗**，才進行下一步。

## 步驟 3：讓 Python 讀取 UART JSON

**目的：** 讓 Python 讀到 ESP32 傳送的每一筆資料。

建立檔案 `read_uart.py`。請將 `COM3` 改成你的 ESP32 實際 COM 埠；每台電腦的號碼可能不同。

執行位置：Python／電腦  
檔案：`read_uart.py`

```python
import json
import serial

PORT = "COM3"  # 改成你的 ESP32 COM 埠
BAUD_RATE = 115200

with serial.Serial(PORT, BAUD_RATE, timeout=3) as ser:
    print("已連線，等待 ESP32 資料…")

    received_count = 0
    while received_count < 3:
        raw_line = ser.readline().decode("utf-8", errors="replace").strip()

        if not raw_line:
            continue

        try:
            data = json.loads(raw_line)
        except json.JSONDecodeError:
            print(f"略過非 JSON 資料：{raw_line}")
            continue

        if data.get("type") == "sensor":
            print(f"溫度：{data['temp']} °C，濕度：{data['humidity']} %")
            received_count += 1
```

執行位置：PowerShell／電腦

```powershell
uv run python read_uart.py
```

預期結果：程式印出三筆固定的溫度與濕度，然後結束。

!!! tip "為什麼使用 timeout？"
    `timeout=3` 表示 Python 最多等候 3 秒。如果 ESP32 沒有資料，程式不會永遠卡在等待狀態。

## 步驟 4：改成讀取 DHT11 並回覆 LED 命令

**目的：** 將固定資料改成真實感測資料，並讓 ESP32 接收 Python 的控制命令。

將 ESP32 程式替換成以下內容：

執行位置：Arduino IDE／ESP32  
檔案：`esp32_uart_dht11.ino`

```cpp
#include <DHT.h>

const int DHT_PIN = 4;
const int LED_PIN = 2;
const unsigned long SENSOR_INTERVAL_MS = 2000;

#define DHT_TYPE DHT11
DHT dht(DHT_PIN, DHT_TYPE);

unsigned long last_sensor_time = 0;

void send_led_status(bool is_on) {
  Serial.printf("{\"type\":\"status\",\"led\":%s,\"ok\":true}\n", is_on ? "true" : "false");
}

void handle_command() {
  if (!Serial.available()) {
    return;
  }

  String command = Serial.readStringUntil('\n');
  command.trim();

  if (command == "{\"cmd\":\"led\",\"value\":true}") {
    digitalWrite(LED_PIN, HIGH);
    send_led_status(true);
  } else if (command == "{\"cmd\":\"led\",\"value\":false}") {
    digitalWrite(LED_PIN, LOW);
    send_led_status(false);
  } else {
    Serial.println("{\"type\":\"error\",\"message\":\"unknown command\"}");
  }
}

void send_sensor_data() {
  float humidity = dht.readHumidity();
  float temperature = dht.readTemperature();

  if (isnan(humidity) || isnan(temperature)) {
    Serial.println("{\"type\":\"error\",\"message\":\"dht read failed\"}");
    return;
  }

  Serial.printf(
    "{\"type\":\"sensor\",\"temp\":%.1f,\"humidity\":%.1f}\n",
    temperature,
    humidity
  );
}

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  dht.begin();
}

void loop() {
  handle_command();

  if (millis() - last_sensor_time >= SENSOR_INTERVAL_MS) {
    last_sensor_time = millis();
    send_sensor_data();
  }
}
```

預期結果：序列監控視窗每約 2 秒出現一筆 `sensor` JSON。若 DHT11 讀取失敗，會出現 `error` JSON，而不是假的溫溼度數值。

!!! note "為什麼每約 2 秒才讀一次？"
    DHT11 是較慢的感測器。太頻繁讀取容易得到舊資料或讀取失敗；本課統一每約 2 秒讀取一次。

!!! warning "命令格式要完全一致"
    為了先聚焦 UART 概念，本頁的 ESP32 範例只接受兩個完整命令：LED 開啟或 LED 關閉。後續會再介紹更彈性的資料驗證方式。

## 步驟 5：用 Python 控制 LED

**目的：** 讓 Python 將 JSON 命令寫進 UART，再讀取 ESP32 的狀態回覆。

建立檔案 `control_led.py`。請將 `COM3` 改成你的 ESP32 實際 COM 埠。

執行位置：Python／電腦  
檔案：`control_led.py`

```python
import json
import serial
import time

PORT = "COM3"  # 改成你的 ESP32 COM 埠
BAUD_RATE = 115200


def send_led_command(ser: serial.Serial, value: bool) -> None:
    command = {"cmd": "led", "value": value}
    message = json.dumps(command, separators=(",", ":")) + "\n"
    ser.write(message.encode("utf-8"))
    print(f"已送出：{message.strip()}")


with serial.Serial(PORT, BAUD_RATE, timeout=3) as ser:
    time.sleep(2)  # 讓部分 ESP32 開啟序列埠後有時間重新啟動
    ser.reset_input_buffer()

    send_led_command(ser, True)

    deadline = time.monotonic() + 5
    while time.monotonic() < deadline:
        raw_line = ser.readline().decode("utf-8", errors="replace").strip()
        if not raw_line:
            continue

        try:
            data = json.loads(raw_line)
        except json.JSONDecodeError:
            print(f"略過非 JSON 資料：{raw_line}")
            continue

        if data.get("type") == "status":
            state = "開啟" if data.get("led") else "關閉"
            print(f"LED 狀態：{state}")
            break
        if data.get("type") == "error":
            print(f"ESP32 回覆錯誤：{data.get('message')}")
            break
    else:
        print("等待 ESP32 狀態回覆逾時")
```

執行位置：PowerShell／電腦

```powershell
uv run python control_led.py
```

預期結果：ESP32 的板載 LED 亮起，PowerShell 顯示：

```text
已送出：{"cmd":"led","value":true}
LED 狀態：開啟
```

!!! note "為什麼 JSON 沒有空白？"
    本頁 ESP32 範例為了聚焦 UART，暫時以完整文字比對命令。`json.dumps(..., separators=(",", ":"))` 會產生 `{"cmd":"led","value":true}`，剛好符合 ESP32 預期格式。後續教材會再介紹不依賴文字空白的資料驗證方式。

## 小練習：補上 LED 關閉命令

**目標：** 使用既有的 `send_led_command()` 函式，讓 Python 在 LED 開啟後再關閉 LED。

請在 `# TODO` 處補上程式。其餘程式不用修改。

執行位置：Python／電腦  
檔案：`practice_led_off.py`

```python linenums="1" hl_lines="5"
# 假設 ser 已經是開啟的序列埠，且 send_led_command() 已定義完成。

send_led_command(ser, True)

# TODO: 使用 send_led_command() 送出 LED 關閉命令
```

預期結果：ESP32 收到第二筆 JSON 命令後，板載 LED 熄滅。

<details>
<summary>查看參考解答</summary>

```python
send_led_command(ser, False)
```

Python 的 `False` 會經 `json.dumps()` 轉成 JSON 的 `false`，ESP32 便會執行 LED 關閉分支。
</details>

## 完成時，應能確認

- ESP32 能每約 2 秒傳送一筆 DHT11 溫溼度 JSON。
- `read_uart.py` 能顯示至少三筆溫溼度資料。
- `control_led.py` 能讓板載 LED 亮起，並顯示 `LED 狀態：開啟`。
- 小練習能讓板載 LED 熄滅。
- Arduino IDE 的序列監控視窗與 Python 不會同時開啟同一個 COM 埠。

## 常見問題

### Python 顯示 `PermissionError` 或無法開啟 COM 埠

1. 關閉 Arduino IDE 的序列監控視窗。
2. 關閉其他可能正在執行的 Python 程式。
3. 確認 `PORT` 是 ESP32 的 COM 埠，而不是藍牙或其他裝置的 COM 埠。

### Python 顯示亂碼或 JSON 解析失敗

1. 確認 ESP32 與 Python 都使用 `115200` baud rate。
2. 先重新上傳步驟 2 的固定 JSON 程式，確認通訊本身正常。
3. 印出 `raw_line`，檢查是否混入啟動訊息或少了 `{`、`}`。

### ESP32 顯示 `dht read failed`

1. 拔除 USB 線後，確認 DHT11 的 VCC、GND、DATA 分別接到 3V3、GND、GPIO 4。
2. 確認 Arduino IDE 安裝的是 `DHT sensor library` 與 `Adafruit Unified Sensor`。
3. 確認程式使用 `#define DHT_TYPE DHT11`，不是 DHT22。
4. 等候至少約 2 秒再讀取；若仍失敗，請先回到固定 JSON 程式確認 UART 正常。

### LED 行為和預期相反

部分相容開發板的板載 LED 可能是低電位亮起。請先告知教師或助教，再依課堂統一方式調整 `HIGH`／`LOW`；不要只修改 Python 程式。

## 重點整理

- USB UART 讓 ESP32 與 Python 可以雙向傳送資料。
- 本課每一筆訊息都是一行 JSON，最後加上換行。
- 先用固定 JSON 驗證 UART，再接入 DHT11，除錯會更容易。
- DHT11 以約 2 秒的間隔讀取，失敗時應回報錯誤而不是使用假資料。
- Python 傳送命令後，應等待 ESP32 回覆狀態來確認控制結果。

## 延伸練習（可選）

修改 `control_led.py`，讓程式依序執行「開啟 → 等待 2 秒 → 關閉」。每次送出命令後，都讀取並印出 ESP32 的狀態回覆。

## 延伸選讀

想了解 ESP32、Python 與不同 JSON 訊息各自的責任，可閱讀[延伸選讀：UART 程式導讀](附錄-UART程式導讀.md)。這不是本節的必做步驟。

## 下一步

前往 [將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)。
