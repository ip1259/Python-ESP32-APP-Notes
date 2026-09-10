# Gradio：建立本機 AIoT 儀表板

這一頁會把 ESP32 經 USB UART 傳來的 DHT11 資料放到 Gradio 網頁，顯示最新讀值、最近資料表與趨勢圖，並從網頁控制板載 LED。

## 你會學到什麼

- 用 Gradio Blocks 建立可互動的本機網頁介面。
- 將 UART 的一行 JSON 整理成最新讀值、Pandas 資料表與折線圖。
- 從按鈕送出 LED JSON 命令，並等待 ESP32 狀態回覆。
- 分辨 Gradio 頁面、Python 程式與 ESP32 同時使用 COM 埠時的責任。

## 開始前

| 項目 | 需求 |
| --- | --- |
| 前置教材 | [UART：讓 Python 與 ESP32 雙向通訊](03-uart-python-esp32.md)、[將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md) |
| 硬體 | 已接好 DHT11 的 ESP32 與 USB 資料線 |
| Python 套件 | `pyserial`、`pandas`、`matplotlib`、`gradio` |
| ESP32 程式 | 第 3 份教材的 `esp32_uart_dht11.ino` 已上傳，並以 `115200` baud rate 傳送 `sensor` JSON |
| 執行位置 | Python／教室電腦；網頁只在同一台電腦的本機瀏覽器使用 |

!!! warning "同一時間只能有這支儀表板程式使用 COM 埠"
    啟動本頁程式前，請關閉 Arduino IDE 的序列監控視窗、`read_uart.py`、`record_sensor_data.py` 和其他可能開啟 ESP32 COM 埠的程式。這份程式會持續使用 COM 埠；不要在儀表板執行時另外開啟序列監控。

!!! warning "不建立公開連結"
    本頁只使用本機網址 `http://127.0.0.1:7860`，不使用 `share=True`、不公開到網際網路，也不要在程式中填入 Wi-Fi 密碼或其他個人資料。

## 成功的樣子

資料流如下：

```text
DHT11 → ESP32 → USB UART → Python 的 SerialDashboard
                                  ↓
                     Gradio：數值、資料表、折線圖、LED 按鈕
                                  ↓
                         同一台電腦的本機瀏覽器
```

按下「更新資料」後，頁面會顯示類似結果：

```text
最新溫度：25.0 °C
最新濕度：60.0 %
狀態：已更新：2026-09-09 14:30:02
```

按下「LED 開啟」後，板載 LED 亮起，狀態框顯示：

```text
ESP32 已確認：LED 開啟
```

## Gradio 的定位：快速驗證，不是正式產品的預設做法

Gradio 的強項是讓 Python 程式很快有一個可操作的畫面。本課利用它驗證「ESP32 資料能否到達 Python、畫面能否正確呈現、命令能否回到 ESP32」；它很適合教學、實驗、研究展示、模型展示與團隊內部工具。

| 面向 | Gradio 原型／實驗頁面 | 正式網站產品 |
| --- | --- | --- |
| 主要目標 | 快速驗證功能與資料流 | 長期提供使用者穩定、安全且一致的服務 |
| 優點 | Python 函式可快速接上按鈕、表格與圖表；程式量少 | 可細緻設計使用流程、品牌、版面與跨裝置體驗 |
| 本頁做法 | 在一台教室電腦的 localhost 操作 USB UART | 需要規劃前端、後端 API、資料庫、部署與監控 |
| 不應省略的工作 | 本頁不處理公開服務 | 帳號與權限、輸入驗證、資安、錯誤追蹤、測試、備份、效能與擴充 |

因此，不要把本頁的示範程式直接當作公開網站或最終產品。若要建置正式網站，通常會依團隊與需求選用專門的網頁技術：前端可使用 HTML、CSS、JavaScript 或 TypeScript，並搭配適合的框架；後端則可使用 API 與伺服器框架，例如 Python 的 FastAPI／Django，或其他團隊熟悉的技術。重點不是背誦特定工具，而是讓產品需求、安全與維護方式有合適的架構。

!!! note "先做原型，再決定產品架構"
    Gradio 原型仍然很有價值：它能在投入大量網站開發前，先驗證感測資料格式、控制命令與使用者需要的畫面。原型驗證成功後，再把已確認的需求轉成正式產品的規格。

## 步驟 1：先確認 Gradio 可以開啟本機頁面

**目的：** 先把「Gradio 環境問題」和「UART／ESP32 問題」分開。

建立檔案 `gradio_hello.py`。

執行位置：Python／電腦  
檔案：`gradio_hello.py`

```python
import gradio as gr


def say_hello() -> str:
    return "Gradio 已正常運作。"


with gr.Blocks() as demo:
    gr.Markdown("# 我的第一個 Gradio 頁面")
    message = gr.Textbox(label="結果")
    button = gr.Button("測試按鈕")
    button.click(say_hello, outputs=message)

demo.launch(server_name="127.0.0.1", share=False)
```

執行位置：PowerShell／電腦

```powershell
uv run python gradio_hello.py
```

預期結果：PowerShell 顯示本機網址。開啟 `http://127.0.0.1:7860` 後，按「測試按鈕」會在文字框看到 `Gradio 已正常運作。`。結束程式時回到 PowerShell 按 `Ctrl+C`。

!!! note "`127.0.0.1` 是什麼？"
    `127.0.0.1` 也稱為 localhost，代表「這一台電腦自己」。因此本頁的瀏覽器和 Python 程式在同一台電腦上溝通，並不需要讓 ESP32 或手機加入電腦的網路。

## 步驟 2：建立 UART 儀表板

**目的：** 讓一支 Python 程式獨占 COM 埠、接收資料並提供給 Gradio 的各個按鈕。

建立檔案 `gradio_dashboard.py`。只將 `COM3` 改為你的 ESP32 實際 COM 埠；不要修改 baud rate 或 JSON 欄位名稱。

執行位置：Python／電腦  
檔案：`gradio_dashboard.py`

```python
import json
import time
from collections import deque
from datetime import datetime
from matplotlib.figure import Figure

import gradio as gr
import matplotlib.pyplot as plt
import pandas as pd
import serial

PORT = "COM3"  # 改成你的 ESP32 COM 埠
BAUD_RATE = 115200
HISTORY_SIZE = 20
JsonData = dict[str, object]
SensorRow = dict[str, str | float]


class SerialDashboard:
    def __init__(self, port: str, baud_rate: int) -> None:
        self.ser: serial.Serial = serial.Serial(port, baud_rate, timeout=0.5)
        time.sleep(2)  # 部分 ESP32 在開啟序列埠後會重新啟動
        self.ser.reset_input_buffer()
        self.history: deque[SensorRow] = deque(maxlen=HISTORY_SIZE)

    def close(self) -> None:
        if self.ser.is_open:
            self.ser.close()

    def read_json_line(self) -> JsonData | None:
        raw_line = self.ser.readline().decode("utf-8", errors="replace").strip()
        if not raw_line:
            return None

        try:
            return json.loads(raw_line)
        except json.JSONDecodeError:
            print(f"略過非 JSON 資料：{raw_line}")
            return None

    def read_sensor(self, wait_seconds: float = 3) -> tuple[SensorRow | None, str | None]:
        deadline = time.monotonic() + wait_seconds

        while time.monotonic() < deadline:
            data = self.read_json_line()
            if data is None:
                continue

            if data.get("type") == "error":
                return None, f"ESP32 錯誤：{data.get('message', 'unknown error')}"

            if data.get("type") != "sensor":
                continue

            try:
                row = {
                    "timestamp": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
                    "temp_c": float(data["temp"]),
                    "humidity": float(data["humidity"]),
                }
            except (KeyError, TypeError, ValueError):
                return None, "收到的 sensor JSON 缺少 temp 或 humidity 數值"

            self.history.append(row)
            return row, None

        return None, "3 秒內沒有收到 sensor JSON"

    def send_led_command(self, value: bool) -> str:
        # 先清除舊感測資料，避免把上一筆資料誤認為這次命令的回覆。
        self.ser.reset_input_buffer()
        command = {"cmd": "led", "value": value}
        message = json.dumps(command, separators=(",", ":")) + "\n"
        self.ser.write(message.encode("utf-8"))
        self.ser.flush()

        deadline = time.monotonic() + 5
        while time.monotonic() < deadline:
            data = self.read_json_line()
            if data is None:
                continue

            if data.get("type") == "error":
                return f"ESP32 錯誤：{data.get('message', 'unknown error')}"

            if data.get("type") == "status":
                if data.get("led") == value and data.get("ok") is True:
                    state = "開啟" if value else "關閉"
                    return f"ESP32 已確認：LED {state}"
                return f"ESP32 回覆的狀態不符合預期：{data}"

        return "已送出命令，但 5 秒內沒有收到 ESP32 狀態回覆"

    def dataframe(self) -> pd.DataFrame:
        return pd.DataFrame(
            self.history,
            columns=["timestamp", "temp_c", "humidity"],
        )

    def figure(self) -> Figure:
        data = self.dataframe()
        fig, (temp_ax, humidity_ax) = plt.subplots(2, 1, sharex=True, figsize=(8, 5))

        if data.empty:
            temp_ax.set_title("Waiting for sensor data")
        else:
            temp_ax.plot(data["timestamp"], data["temp_c"], marker="o", color="tab:red")
            humidity_ax.plot(data["timestamp"], data["humidity"], marker="o", color="tab:blue")

        temp_ax.set_ylabel("Temperature (°C)")
        humidity_ax.set_ylabel("Humidity (%)")
        humidity_ax.set_xlabel("Time")
        temp_ax.grid(True)
        humidity_ax.grid(True)
        fig.autofmt_xdate()
        fig.tight_layout()
        return fig


dashboard = SerialDashboard(PORT, BAUD_RATE)


def refresh_dashboard() -> tuple[str, str, pd.DataFrame, Figure, str]:
    row, error_message = dashboard.read_sensor()
    data = dashboard.dataframe()

    if error_message:
        return "—", "—", data, dashboard.figure(), error_message

    status = f"已更新：{row['timestamp']}"
    return (
        f"{row['temp_c']:.1f} °C",
        f"{row['humidity']:.1f} %",
        data,
        dashboard.figure(),
        status,
    )


def set_led(value: bool) -> str:
    return dashboard.send_led_command(value)


with gr.Blocks(title="AIoT DHT11 儀表板") as demo:
    gr.Markdown("# AIoT DHT11 儀表板\n本頁只在這台電腦本機使用。")

    with gr.Row():
        temperature = gr.Textbox(label="最新溫度", value="—")
        humidity = gr.Textbox(label="最新濕度", value="—")

    refresh_button = gr.Button("更新資料", variant="primary")
    history_table = gr.Dataframe(
        headers=["timestamp", "temp_c", "humidity"],
        label="最近 20 筆資料",
        interactive=False,
    )
    chart = gr.Plot(label="溫度與濕度趨勢")

    with gr.Row():
        led_on_button = gr.Button("LED 開啟")
        led_off_button = gr.Button("LED 關閉")

    status = gr.Textbox(label="狀態", value="請按「更新資料」讀取 ESP32。")

    refresh_button.click(
        refresh_dashboard,
        outputs=[temperature, humidity, history_table, chart, status],
    )
    led_on_button.click(lambda: set_led(True), outputs=status)
    led_off_button.click(lambda: set_led(False), outputs=status)

try:
    demo.launch(server_name="127.0.0.1", share=False)
finally:
    dashboard.close()
```

這支程式的資料責任如下：

| 程式部分 | 負責工作 |
| --- | --- |
| `SerialDashboard` | 唯一開啟 COM 埠、讀取 JSON、保留最近 20 筆資料、送出 LED 命令 |
| `refresh_dashboard()` | 讀取一筆新的 `sensor` 資料，回傳五個 Gradio 元件所需內容 |
| `set_led()` | 呼叫共用的序列埠物件，等待 ESP32 的 `status` 回覆 |
| `gr.Button.click()` | 將網頁按鈕事件連到 Python 函式，不直接操作 ESP32 GPIO |

!!! note "為什麼先清除輸入緩衝區？"
    ESP32 每約 2 秒會主動傳來 `sensor` 資料。送 LED 命令前清除舊訊息，可降低把「之前的感測資料」誤認成「這次命令回覆」的機會；送出後仍會等待 `type` 為 `status` 的 JSON 才確認成功。

## 步驟 3：啟動、操作與觀察資料

**目的：** 對照網頁、Python 與 ESP32 的行為，確認資料不是只停留在畫面上。

執行位置：PowerShell／電腦

```powershell
uv run python gradio_dashboard.py
```

1. 開啟 PowerShell 顯示的 `http://127.0.0.1:7860`。
2. 按「更新資料」。最新溫度與濕度應更新；連按數次可讓資料表累積多筆資料，折線圖隨之出現。
3. 按「LED 開啟」，觀察 ESP32 板載 LED，並確認狀態框出現 `ESP32 已確認：LED 開啟`。
4. 按「LED 關閉」，再次確認硬體與狀態回覆。
5. 結束時在 PowerShell 按 `Ctrl+C`，程式才會釋放 COM 埠。

預期結果：資料表最多保留最近 20 筆。這是即時介面用的短期歷史；要保存完整資料與圖檔，仍使用[前一份教材的 CSV 與 Matplotlib 流程](04-python-感測資料視覺化.md)。

## 如何判讀儀表板

這個儀表板可以協助你快速回答：

- 最新一筆溫度與濕度是多少？
- 最近幾筆資料有沒有明顯跳動、缺漏或持續上升／下降？
- 按下 LED 按鈕後，ESP32 有沒有回覆已完成控制？

它不能單獨證明環境真的長期改變。DHT11 讀值會受感測器位置、手部接近、讀取間隔與環境影響；而且畫面只保留最近 20 筆，不能取代 CSV 的長期紀錄。

## 小練習：完成 LED 開啟回呼

**目標：** 使用已建立的 `dashboard` 物件，讓 Gradio 的「LED 開啟」按鈕送出命令並回傳 ESP32 狀態文字。

請在 `# TODO` 處補上程式。其餘程式不用修改。

執行位置：Python／電腦  
檔案：`practice_gradio_led.py`

```python linenums="1" hl_lines="4-5"
# 假設 dashboard 已由 gradio_dashboard.py 建立完成。

def turn_led_on() -> str:
    # TODO: 呼叫 dashboard 的 send_led_command()，傳入 True，並直接回傳結果文字
    pass


led_on_button.click(turn_led_on, outputs=status)
```

預期結果：按下「LED 開啟」後，板載 LED 亮起，狀態框顯示 `ESP32 已確認：LED 開啟`。

<details>
<summary>查看參考解答</summary>

```python
def turn_led_on() -> str:
    return dashboard.send_led_command(True)
```

`send_led_command(True)` 會組成 LED 開啟 JSON、等候 ESP32 回覆，並將成功或錯誤文字交給 Gradio 的 `status` 元件。
</details>

## 完成時，應能確認

- `gradio_hello.py` 能在 `127.0.0.1:7860` 顯示並回應按鈕。
- `gradio_dashboard.py` 是唯一開啟 ESP32 COM 埠的程式。
- 按「更新資料」能顯示新的溫度、濕度，並在資料表與圖表累積最近資料。
- 按 LED 開啟與關閉後，介面有收到相符的 ESP32 `status` 回覆，硬體行為也一致。
- 沒有使用 `share=True`、公開網址、Wi-Fi 密碼或固定區網 IP。

## 常見問題

### PowerShell 顯示找不到 `gradio` 或頁面無法啟動

1. 在專案資料夾執行 `uv run python -c "import gradio; print(gradio.__version__)"`，確認套件可匯入。
2. 若無法匯入，回到[用 uv 建立 Python 虛擬環境](01-uv與虛擬環境.md)的套件安裝步驟，安裝 `gradio`。
3. 關閉同一台電腦上另一個正在使用 7860 連接埠的 Gradio 程式，再重新啟動。

### 程式顯示 `PermissionError`、`Access is denied` 或無法開啟 COM 埠

1. 關閉 Arduino IDE 序列監控視窗。
2. 停止 `read_uart.py`、`record_sensor_data.py` 或之前啟動的 Gradio 程式；必要時在 PowerShell 按 `Ctrl+C` 後再試。
3. 確認 `PORT` 是 ESP32 的 COM 埠，而非藍牙或其他裝置的 COM 埠。

### 按「更新資料」後一直顯示 3 秒內沒有收到 sensor JSON

1. 確認 ESP32 已上傳第 3 份教材的 DHT11 UART 程式，且 DHT11 每約 2 秒能傳送 `type` 為 `sensor` 的 JSON。
2. 確認 ESP32 與 Python 的 baud rate 都是 `115200`。
3. 先停止儀表板，再用第 3 份教材的 `read_uart.py` 驗證 UART；驗證完必須關閉它，再重新啟動儀表板。

### 網頁有資料，但 LED 按鈕顯示沒有收到狀態回覆

1. 確認 ESP32 程式仍包含 `handle_command()` 與 `send_led_status()`，而且命令以換行結尾。
2. 確認 ESP32 期待的命令是 `{"cmd":"led","value":true}` 或 `{"cmd":"led","value":false}`；不要自行改成 `ON`、`OFF` 等不同格式。
3. 若狀態框顯示 ESP32 錯誤，先閱讀 `message`，再回到 UART 教材檢查命令格式與板載 LED 行為。

### 圖表空白或只出現一個點

1. 先連按「更新資料」數次；每次只會加入一筆新的感測資料。
2. 確認資料表的 `temp_c`、`humidity` 是數字，不是 `—` 或錯誤文字。
3. 若只有一筆資料，圖表有一個點是正常現象；累積至少兩筆才能看出趨勢線。

## 重點整理

- Gradio Blocks 將 Python 函式與網頁元件連接，但 ESP32 GPIO 仍由 ESP32 韌體控制。
- 儀表板程式必須獨占 COM 埠；不要同時開啟序列監控或其他讀取程式。
- 一行一筆 JSON 讓 Python 能分辨感測資料、狀態回覆與錯誤訊息。
- 顯示「已送出命令」不等於控制成功；要等 ESP32 的 `status` 回覆確認。
- 本頁只在 localhost 使用；CSV 與 Matplotlib 才是保存與分析長期資料的工具。

## 延伸練習（可選）

當手動更新已穩定後，為頁面加入一個「溫度提醒」文字框：當最新溫度高於你和教師約定的門檻時顯示提醒，否則顯示正常。先只修改 Python 回呼的回傳文字，不要加入新的硬體或公開網頁設定。

## 延伸選讀

想了解按鈕事件、最近資料與 COM 埠責任如何配合，以及本機原型的使用界線，可閱讀[延伸選讀：Gradio 事件、狀態與原型邊界](附錄-Gradio事件狀態與原型邊界.md)。這不是本節的必做步驟。

## 下一步

前往[整合專題：完成可展示的環境看板](08-整合專題.md)。
