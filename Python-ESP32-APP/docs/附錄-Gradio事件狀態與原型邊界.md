# 延伸選讀：Gradio 事件、狀態與原型邊界

> 選讀｜對應主線：[Gradio：建立本機 AIoT 儀表板](07-gradio-aiot儀表板.md)

本頁解讀主線儀表板的事件如何呼叫 Python，以及為什麼它是限於本機的功能原型。

## 讀完後你會知道

- 為什麼 `SerialDashboard` 是唯一能開啟 COM 埠的物件。
- 「更新資料」按鈕如何一次更新五個 Gradio 元件。
- 為什麼最近 20 筆是介面狀態，不是 CSV 的長期保存。
- 為什麼本機 Gradio 原型不能直接等同正式公開網站。

## 先知道這件事

主線第 7 節已能在同一台電腦的 `127.0.0.1` 顯示資料、控制 LED，並在結束時釋放 COM 埠。本頁只解讀 `gradio_dashboard.py` 的成功版本；不提供公開分享或部署設定。

## 三個責任層

```text
瀏覽器按鈕 → Gradio .click() → Python 回呼 → SerialDashboard → USB UART → ESP32
                                ↓
                     文字框／表格／圖表的回傳值
```

| 層次 | 負責什麼 | 不負責什麼 |
| --- | --- | --- |
| ESP32 | 讀取 DHT11、控制 GPIO、回覆 JSON | 產生網頁元件 |
| `SerialDashboard` | 獨占 COM 埠、解析 JSON、保存最近資料、等待 LED 回覆 | 長期保存 CSV |
| Gradio 回呼與元件 | 把 Python 回傳值顯示在網頁 | 直接控制 ESP32 GPIO |

## 課程程式碼導讀

執行位置：Python／電腦  
來源主線檔案：`gradio_dashboard.py`

```python linenums="1"
import json
import time
# collections是python內鍵的標準套件庫，提供更進階的資料結構
# deque(Double-Ended Queue)是其中一個高效雙端佇列資料結構
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
JsonData = dict[str, object]  # 自訂的資料型別，用於Type Hint
SensorRow = dict[str, str | float]  # 自訂的資料型別，用於Type Hint


# 【1】集中管理唯一的序列埠與短期歷史。
class SerialDashboard:
    def __init__(self, port: str, baud_rate: int) -> None:
        self.ser: serial.Serial = serial.Serial(port, baud_rate, timeout=0.5)
        time.sleep(2)
        self.ser.reset_input_buffer()
        self.history: deque[SensorRow] = deque(maxlen=HISTORY_SIZE)

    def close(self) -> None:
        if self.ser.is_open:
            self.ser.close()

    # 【1.1】讀取一行；空行與非 JSON 都不交給後續流程。
    def read_json_line(self) -> JsonData | None:
        raw_line = self.ser.readline().decode("utf-8", errors="replace").strip()
        if not raw_line:  # 處理空行
            return None

        try:
            return json.loads(raw_line)
        except json.JSONDecodeError:  # 處理非json資料
            print(f"略過非 JSON 資料：{raw_line}")
            return None

    # 【1.2】只把完整的 sensor JSON 加到最近資料。
    def read_sensor(self, wait_seconds: float = 3) -> tuple[SensorRow | None, str | None]:
        # time.monotonic() 回傳程式開始執行後過了幾秒
        deadline = time.monotonic() + wait_seconds

        while time.monotonic() < deadline:
            data = self.read_json_line()
            if data is None:  # 阻擋無效資料
                continue

            #  利用回傳資料data的"type"去判斷該怎麼處理ESP32傳過來的資料
            #  "type":"error"則回傳錯誤訊息
            #  其他type則略過不處理，只處理"type"為sensor的資料
            if data.get("type") == "error":
                return None, f"ESP32 錯誤：{data.get('message', 'unknown error')}"

            if data.get("type") != "sensor":
                continue  # 其他type略過不處理

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

    # 【1.3】送命令前清除舊資料，然後只接受相符的 status。
    def send_led_command(self, value: bool) -> str:
        """傳送led命令給ESP32，回傳ESP32的執行結果"""
        self.ser.reset_input_buffer()  # 清除緩衝區
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

    # 繪圖後回傳圖表物件，要注意這邊不是指傳給哪一個功能
    # 而是SerialDashboard的實例(instance)物件傳出了一個圖表物件
    # 而這個圖表物件可以直接繪製在螢幕上或存成檔案是其他程式碼之後再決定的事
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


dashboard = SerialDashboard(PORT, BAUD_RATE)  # 建構一個SerialDashboard實例物件


# 【2】回傳值的順序，必須和 click() 的 outputs 順序相同。
def refresh_dashboard() -> tuple[str, str, pd.DataFrame, Figure, str]:
    # 這個函數要搭配Gradio使用，Gradio後台會依據順序重新渲染Gradio元件
    #  由dashboard物件操作所有uart<->ESP32資料傳遞
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
    # 這個函式跟refresh_dashboard一樣都是設計給Gradio元件回呼(call back)用的
    # 回呼（Callback，又稱回調）在程式設計中，是指把一個函式（Function
    # 當作參數傳遞給另一個函式，並在特定事件或任務完成後，由接收方主動去呼叫（執行）它。
    # 白話來說就是把程式的執行順序交給別人，等條件達成(比如使用者點了按鈕)才執行。
    return dashboard.send_led_command(value)


# 【3】建立元件，再將按鈕事件接到回呼。
#  Gradio會在單獨拿出來解釋，簡單來說這一段就是用類似堆積木的方式堆疊一個網頁GUI
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

    # 以上Gradio區塊基本上都是網頁GUI元件的宣告、參數設定與元件堆疊
    # 以下設定Gradio按鈕元件的click事件與回呼方法連結
    refresh_button.click(
        refresh_dashboard,
        outputs=[temperature, humidity, history_table, chart, status],  # 設定outputs可以用回乎方法的回傳值更新相對應順序的元件
    )
    led_on_button.click(lambda: set_led(True), outputs=status)
    led_off_button.click(lambda: set_led(False), outputs=status)

# 【4】限定本機、禁止公開分享；無論如何結束都釋放 COM 埠。
try:
    demo.launch(server_name="127.0.0.1", share=False)  # 執行網頁後端
finally:
    # try-except-finally這個錯誤處理流程中，finally代表除非有錯誤沒有接住處理
    # 不然不管有沒有進except都會執行，此處用來關閉Serial連線
    dashboard.close()
```

`deque(maxlen=20)` 超過容量時會自動捨棄最舊的一筆，這正符合「畫面只顯示最近資料」的需求。它不是檔案保存；要保留完整紀錄，仍應使用第 4 節的 CSV 流程。

## 原型與正式產品的界線

| 問題 | 本頁的 Gradio 原型 | 正式產品仍需處理 |
| --- | --- | --- |
| 使用範圍 | 同一台電腦的 localhost | 使用者、裝置與網路存取設計 |
| 資料 | RAM 中最近 20 筆 | 長期資料庫、備份與保留政策 |
| 安全 | `share=False`，不公開網址 | 帳號、權限、輸入驗證、日誌與監控 |
| 可靠性 | 用於教學驗證資料流 | 部署、測試、錯誤復原與擴充 |

## 常見誤解

### 「Gradio 的按鈕直接控制 ESP32 腳位」

按鈕只會呼叫 Python 函式；Python 再把 JSON 寫到 USB UART，最後由 ESP32 韌體呼叫 `digitalWrite()`。因此任何一段中斷，都可能造成按鈕無法完成控制。

## 重點整理

- 儀表板以一個 `SerialDashboard` 物件集中管理 COM 埠，避免多個程式競爭資源。
- `refresh_dashboard()` 的五個回傳值按順序更新五個元件。
- 最近 20 筆是短期顯示狀態，不取代 CSV 保存。
- localhost Gradio 適合快速驗證，不是公開正式網站的完整架構。

## 回到主線

回到 [Gradio：建立本機 AIoT 儀表板](07-gradio-aiot儀表板.md)，繼續完成本機環境看板。
