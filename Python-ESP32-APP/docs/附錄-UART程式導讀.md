# 延伸選讀：UART 程式導讀

> 選讀｜對應主線：[UART：讓 Python 與 ESP32 雙向通訊](03-uart-python-esp32.md)

本頁用主線的成功範例說明：誰負責讀取感測器、誰負責解析 JSON，以及 LED 命令如何取得確認回覆。

## 讀完後你會知道

- 為什麼 ESP32 與 Python 都以「一行一筆 JSON」作為訊息邊界。
- 感測資料、狀態回覆與錯誤訊息如何走不同分支。
- 為什麼 Python 送出 LED 命令後，仍要等待 `status` 回覆。

## 先知道這件事

主線第 3 節已讓 ESP32 每約 2 秒送出一筆 `sensor` JSON；Python 也能送出 LED 命令並看到狀態回覆。本頁只解讀該成功版本，不需要重新上傳或執行任何程式。

資料流如下：

```text
DHT11 → ESP32 的 send_sensor_data() → 一行 sensor JSON → Python
Python → 一行 led 命令 JSON → ESP32 的 handle_command() → 一行 status JSON → Python
```

## ESP32：讀取、命令與定時傳送

執行位置：Arduino IDE／ESP32  
來源主線檔案：`esp32_uart_dht11.ino`

```cpp linenums="1"
#include <DHT.h>

const int DHT_PIN = 4;
const int LED_PIN = 2;
const unsigned long SENSOR_INTERVAL_MS = 2000;

#define DHT_TYPE DHT11
DHT dht(DHT_PIN, DHT_TYPE);

unsigned long last_sensor_time = 0;

// 【1】把 LED 的實際狀態包成 status JSON 回覆給 Python。
void send_led_status(bool is_on) {
  Serial.printf("{\"type\":\"status\",\"led\":%s,\"ok\":true}\n", is_on ? "true" : "false");
}

// 【2】只處理一整行、且符合主線格式的 LED 命令。
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

// 【3】讀取 DHT11；讀不到時回報 error。這個函式不回傳(return)任何值，因為send_sensor_data的職責是透過Serial傳送資料而不是得到某一個回傳值
void send_sensor_data() {
  float humidity = dht.readHumidity();
  float temperature = dht.readTemperature();

  if (isnan(humidity) || isnan(temperature)) {
    Serial.println("{\"type\":\"error\",\"message\":\"dht read failed\"}");
    return;  // 透過Serial送出錯誤回報後中斷函式
  }
  
  // 3.1 感測器讀取正常時透過Serial送出感測器資訊
  Serial.printf(
    "{\"type\":\"sensor\",\"temp\":%.1f,\"humidity\":%.1f}\n",
    temperature,
    humidity
  );

  // 未寫return等同於return void;
}

// 【4】初始化序列埠、LED 與感測器。
void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  dht.begin();
}

// 【5】每次迴圈都先看命令；感測器則依間隔讀取。
void loop() {
  // 5.1 優先處理指令
  handle_command();

  // 5.2 依設定間隔定時傳送感測器數據
  if (millis() - last_sensor_time >= SENSOR_INTERVAL_MS) {
    last_sensor_time = millis();
    send_sensor_data();
  }
}
```

`handle_command()` 放在每次 `loop()` 的開頭，因此 LED 命令不必等待下一次感測器讀值才被處理。`readStringUntil('\n')` 與 Python 在訊息最後加上的 `\n` 成對使用；少了換行，ESP32 就無法以本課的方式辨認命令結束。

## Python：送出後等待確認

執行位置：Python／電腦  
來源主線檔案：`control_led.py`

```python linenums="1"
import json
import serial
import time

PORT = "COM3"  # 改成你的 ESP32 COM 埠
BAUD_RATE = 115200  # Serial鮑率，需與ESP32初始化設定相同


# 【1】將 Python 的布林值轉為沒有空白的 JSON，再補上一個換行。
def send_led_command(ser: serial.Serial, value: bool) -> None:
    command = {"cmd": "led", "value": value}
    message = json.dumps(command, separators=(",", ":")) + "\n"
    ser.write(message.encode("utf-8"))
    print(f"已送出：{message.strip()}")


# 【2】唯一開啟 COM 埠的程式區塊。
with serial.Serial(PORT, BAUD_RATE, timeout=3) as ser:
  # with區塊能再區塊結束時自動調用ser.close()關閉連線
    time.sleep(2)
    ser.reset_input_buffer()

    send_led_command(ser, True)

    # 【3】在期限內逐行解析，只接受 status 或 error 作為本次結果。
    deadline = time.monotonic() + 5
    while time.monotonic() < deadline:
        raw_line = ser.readline().decode("utf-8", errors="replace").strip()
        if not raw_line:
            continue

        try:
            data = json.loads(raw_line)
        except json.JSONDecodeError:
          # JSONDecodeError代表json解析失敗也就是data為非json資料時嘗試解析data為json資料會發出的錯誤
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

`ser.write()` 只代表電腦已把文字交給序列埠，並不代表 LED 已改變。程式必須再讀取一行，並確認 `type` 是 `status`；收到 `error` 或逾時時，畫面才不會把未確認的命令誤說成成功。

## 何時使用、何時不要使用

| 情境 | 建議 |
| --- | --- |
| 需要辨認一筆完整訊息 | 保留一行一筆 JSON 與最後的換行。 |
| DHT11 讀取失敗 | 傳送 `error`，先檢查感測器與接線。 |
| Python 顯示已送出命令 | 繼續等待相符的 `status`，才判定控制成功。 |
| 想讓 ESP32 接受各種自由文字命令 | 不要直接套用本頁的完整字串比對；那是主線為聚焦 UART 而採用的簡化方式。 |

## 常見誤解

### 「讀到任何 JSON 就表示 LED 命令成功」

不一定。ESP32 也會定時送出 `sensor` JSON。與命令結果直接相關的是 `status` 或 `error`；因此程式需先看 `type`，不能只看「它是不是 JSON」。

## 重點整理

- ESP32 讀取 DHT11 和控制 GPIO；Python 負責送命令、解析與呈現文字。
- 換行讓兩端能以一行為單位讀取 JSON。
- `sensor`、`status`、`error` 是不同用途的訊息，必須分開判斷。
- 命令成功要以 ESP32 回覆的狀態確認。

## 回到主線

回到 [UART：讓 Python 與 ESP32 雙向通訊](03-uart-python-esp32.md)，繼續使用已驗證的 UART 資料流。
