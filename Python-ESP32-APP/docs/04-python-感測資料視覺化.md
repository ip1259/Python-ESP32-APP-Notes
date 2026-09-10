# 將感測資料存成 CSV 並繪製趨勢圖

這一頁會把 ESP32 經 UART 傳來的 DHT11 JSON 資料存成 CSV，再用 Pandas 與 Matplotlib 繪製溫度、濕度趨勢圖。

## 你會學到什麼

- 將多筆 UART JSON 感測資料寫入 CSV。
- 使用 Pandas 檢查資料欄位、型別與缺值。
- 用 Matplotlib 建立有標題、單位與圖例的趨勢圖。
- 分辨「感測資料」與「感測資料的解讀」。

## 開始前

| 項目 | 需求 |
| --- | --- |
| 前置教材 | [UART：讓 Python 與 ESP32 雙向通訊](03-uart-python-esp32.md) |
| 硬體 | 已接好 DHT11 的 ESP32 與 USB 資料線 |
| Python 套件 | `pyserial`、`pandas`、`matplotlib` |
| 重要設定 | 知道 ESP32 的 COM 埠，且 Arduino 序列監控視窗已關閉 |

!!! warning "先確認 UART 已能工作"
    本頁假設 ESP32 已每約 2 秒傳送一筆 `sensor` JSON。若 Python 尚未能讀到資料，請先回到上一頁完成 UART 測試，不要直接開始排查 CSV 或圖表。

## 成功的樣子

完成後，專案資料夾會出現感測資料與圖表：

```text
aiot-course/
├─ data/
│  └─ sensor.csv
└─ charts/
   └─ temperature_humidity.png
```

CSV 的內容類似：

```text
timestamp,temp_c,humidity
2026-09-08T13:30:00,25.0,60.0
2026-09-08T13:30:02,25.0,60.0
```

圖表必須清楚標示：資料時間、溫度單位 `°C`、濕度單位 `%` 與圖例。

## 步驟 1：將 UART 資料存成 CSV

**目的：** 將即時傳來的資料保存成日後可以重複分析的檔案。

建立檔案 `record_sensor_data.py`。請將 `COM3` 改成你的 ESP32 實際 COM 埠。

執行位置：Python／電腦  
檔案：`record_sensor_data.py`

```python
import csv
import json
import time
from datetime import datetime
from pathlib import Path

import serial

PORT = "COM3"  # 改成你的 ESP32 COM 埠
BAUD_RATE = 115200
RECORD_COUNT = 10
TIMEOUT_SECONDS = 45

csv_path = Path("data/sensor.csv")
csv_path.parent.mkdir(exist_ok=True)
file_is_new = not csv_path.exists()

with serial.Serial(PORT, BAUD_RATE, timeout=3) as ser:
    time.sleep(2)
    ser.reset_input_buffer()

    with csv_path.open("a", newline="", encoding="utf-8") as csv_file:
        writer = csv.DictWriter(
            csv_file,
            fieldnames=["timestamp", "temp_c", "humidity"],
        )

        if file_is_new:
            writer.writeheader()

        deadline = time.monotonic() + TIMEOUT_SECONDS
        recorded_count = 0

        while recorded_count < RECORD_COUNT and time.monotonic() < deadline:
            raw_line = ser.readline().decode("utf-8", errors="replace").strip()
            if not raw_line:
                continue

            try:
                data = json.loads(raw_line)
            except json.JSONDecodeError:
                print(f"略過非 JSON 資料：{raw_line}")
                continue

            if data.get("type") == "error":
                print(f"ESP32 回覆錯誤：{data.get('message')}")
                continue

            if data.get("type") != "sensor":
                continue

            row = {
                "timestamp": datetime.now().isoformat(timespec="seconds"),
                "temp_c": data["temp"],
                "humidity": data["humidity"],
            }
            writer.writerow(row)
            csv_file.flush()

            recorded_count += 1
            print(f"已記錄 {recorded_count}/{RECORD_COUNT}：{row}")

if recorded_count < RECORD_COUNT:
    raise RuntimeError("在期限內沒有收到足夠的感測資料")

print(f"完成：資料已寫入 {csv_path}")
```

執行位置：PowerShell／電腦

```powershell
uv run python record_sensor_data.py
```

預期結果：PowerShell 顯示 10 筆「已記錄」訊息，並建立 `data/sensor.csv`。

!!! note "為什麼要保留 timestamp？"
    溫度與濕度本身沒有告訴我們「何時」測得。`timestamp` 把每一筆資料的時間一起存下來，圖表才能呈現變化的先後順序。

## 步驟 2：用 Pandas 檢查 CSV

**目的：** 在繪圖前先確認欄位與數值是正確的。

建立檔案 `inspect_sensor_data.py`。

執行位置：Python／電腦  
檔案：`inspect_sensor_data.py`

```python
from pathlib import Path

import pandas as pd

csv_path = Path("data/sensor.csv")
data = pd.read_csv(csv_path)

data["timestamp"] = pd.to_datetime(data["timestamp"], errors="coerce")
data["temp_c"] = pd.to_numeric(data["temp_c"], errors="coerce")
data["humidity"] = pd.to_numeric(data["humidity"], errors="coerce")

print("前五筆資料：")
print(data.head())

print("\n欄位與型別：")
print(data.info())

print("\n缺值數量：")
print(data.isna().sum())

print("\n數值摘要：")
print(data[["temp_c", "humidity"]].describe())
```

執行位置：PowerShell／電腦

```powershell
uv run python inspect_sensor_data.py
```

預期結果：輸出會顯示三個欄位、各欄位型別、缺值數量與溫度／濕度的最小值、最大值、平均值等摘要。

!!! tip "先看缺值，再畫圖"
    如果 `timestamp`、`temp_c` 或 `humidity` 有缺值，先找出原因。直接繪圖可能會讓資料中斷或得到誤導的結論。

## 步驟 3：繪製溫度與濕度趨勢圖

**目的：** 將數字表格轉為容易比較變化的圖表。

建立檔案 `plot_sensor_data.py`。

執行位置：Python／電腦  
檔案：`plot_sensor_data.py`

```python
from pathlib import Path

import matplotlib.pyplot as plt
import pandas as pd

csv_path = Path("data/sensor.csv")
chart_path = Path("charts/temperature_humidity.png")
chart_path.parent.mkdir(exist_ok=True)

data = pd.read_csv(csv_path)
data["timestamp"] = pd.to_datetime(data["timestamp"], errors="coerce")
data["temp_c"] = pd.to_numeric(data["temp_c"], errors="coerce")
data["humidity"] = pd.to_numeric(data["humidity"], errors="coerce")

valid_data = data.dropna(subset=["timestamp", "temp_c", "humidity"])

if valid_data.empty:
    raise RuntimeError("沒有可繪圖的完整感測資料")

fig, (temp_ax, humidity_ax) = plt.subplots(2, 1, sharex=True, figsize=(9, 6))

temp_ax.plot(valid_data["timestamp"], valid_data["temp_c"], marker="o", label="Temperature")
temp_ax.set_title("DHT11 Temperature and Humidity")
temp_ax.set_ylabel("Temperature (°C)")
temp_ax.grid(True)
temp_ax.legend()

humidity_ax.plot(
    valid_data["timestamp"],
    valid_data["humidity"],
    marker="o",
    color="tab:blue",
    label="Humidity",
)
humidity_ax.set_xlabel("Time")
humidity_ax.set_ylabel("Humidity (%)")
humidity_ax.grid(True)
humidity_ax.legend()

fig.autofmt_xdate()
fig.tight_layout()
fig.savefig(chart_path, dpi=150)

print(f"圖表已儲存到：{chart_path}")
plt.show()
```

執行位置：PowerShell／電腦

```powershell
uv run python plot_sensor_data.py
```

預期結果：出現溫度與濕度兩張折線圖，並建立 `charts/temperature_humidity.png`。

!!! note "為什麼圖表文字使用英文？"
    Windows 電腦的 Matplotlib 中文字型設定可能不同。這份核心範例使用英文標題與軸標籤，確保每台電腦都能先成功產圖；熟悉後可再依教師提供的字型設定改為中文。

## 如何判讀結果

圖表能協助你回答：

- 哪一筆資料的溫度或濕度最高、最低？
- 在這段紀錄時間內，數值是穩定、上升還是下降？
- 是否有明顯不合理、突然跳動或缺漏的資料？

圖表**不能**單獨回答：

- 為什麼溫度改變？可能是環境真的改變，也可能是感測器位置、手部接近或接線不穩。
- 長期的環境趨勢。10 筆資料只代表短時間觀察，不代表一整天的情況。

## 小練習：繪圖前移除缺值資料

**目標：** 在繪圖前，只保留時間、溫度與濕度都完整的資料列。

請在 `# TODO` 處補上程式。其餘程式不用修改。

執行位置：Python／電腦  
檔案：`practice_clean_data.py`

```python linenums="1" hl_lines="11"
import pandas as pd

data = pd.DataFrame(
    {
        "timestamp": ["2026-09-08T13:30:00", "2026-09-08T13:30:02"],
        "temp_c": [25.0, None],
        "humidity": [60.0, 61.0],
    }
)

# TODO: 移除 temp_c 或 humidity 缺值的資料列，指定給 valid_data

print(valid_data)
```

預期結果：輸出只保留第一筆資料，因為第二筆的 `temp_c` 是缺值。

<details>
<summary>查看參考解答</summary>

```python
valid_data = data.dropna(subset=["temp_c", "humidity"])
```

`subset` 指定要檢查的欄位。只有這些欄位都不是缺值的資料列，才會保留在 `valid_data` 中。
</details>

## 完成時，應能確認

- `data/sensor.csv` 有表頭與至少 10 筆感測資料。
- `timestamp`、`temp_c`、`humidity` 都能被 Pandas 正確讀取。
- 你知道 CSV 中是否有缺值，且知道缺值數量。
- `charts/temperature_humidity.png` 有標題、時間軸、單位、圖例與兩個趨勢圖。
- 你能依圖表提出一項觀察，以及一項資料限制。

## 常見問題

### Python 一直等不到 10 筆資料

1. 確認 Arduino IDE 的序列監控視窗已關閉。
2. 確認 `PORT` 是 ESP32 的 COM 埠，且 baud rate 為 `115200`。
3. 先執行上一頁的 `read_uart.py`，確認 ESP32 確實會傳送 `sensor` JSON。
4. 若 ESP32 傳回 `dht read failed`，先處理 DHT11 接線或讀值問題。

### CSV 只有表頭，沒有資料列

1. 確認 ESP32 傳送的 JSON 中 `type` 是 `sensor`。
2. 檢查 PowerShell 是否顯示「略過非 JSON 資料」或 ESP32 錯誤訊息。
3. 確認程式沒有在接收第一筆資料前被手動關閉。

### 圖表出現空白或程式顯示沒有完整資料

1. 先執行 `inspect_sensor_data.py`，查看缺值數量。
2. 開啟 `data/sensor.csv`，確認欄位名稱沒有被修改。
3. 重新執行記錄程式，取得一批新的資料。

### 圖表沒有跳出視窗

1. 先確認 PowerShell 是否已顯示「圖表已儲存到」。
2. 直接開啟 `charts/temperature_humidity.png`，確認圖檔已建立。
3. 如果圖檔存在，代表繪圖成功；視窗行為可能受電腦設定影響。

## 重點整理

- CSV 讓即時感測資料可以保存並重新分析。
- 每筆資料都要包含時間、溫度與濕度等一致欄位。
- Pandas 可以在繪圖前檢查欄位型別、缺值與摘要統計。
- 圖表要標示標題、單位、圖例與時間，才容易被正確判讀。
- 資料觀察不等於原因；短時間的少量資料有其限制。

## 延伸練習（可選）

再次執行 `record_sensor_data.py`，讓 CSV 累積更多資料。比較第一次與第二次繪出的圖表，觀察資料筆數增加後，哪一些變化更容易看出來。

## 延伸選讀

想對照 CSV 記錄與繪圖程式的責任，可閱讀[延伸選讀：CSV 與圖表程式導讀](附錄-CSV與圖表程式導讀.md)。想了解缺值、時間與短期資料的判讀界線，可閱讀[延伸選讀：感測資料品質](附錄-感測資料品質.md)。兩頁都不是本節的必做步驟。

## 下一步

前往 [I2C：使用 LCD1602A 顯示 DHT11 資料](05-i2c-lcd.md)。
