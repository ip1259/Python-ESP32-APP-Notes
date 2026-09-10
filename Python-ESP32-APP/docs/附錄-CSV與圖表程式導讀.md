# 延伸選讀：CSV 與圖表程式導讀

> 選讀｜對應主線：[將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)

本頁解讀主線如何把 UART 的即時資料保存成 CSV，並把可用資料轉成兩張趨勢圖。

## 讀完後你會知道

- 為什麼 CSV 必須固定欄位並在每筆資料後寫入時間。
- 為什麼繪圖前要先轉換型別與排除不完整資料列。
- 圖表的兩個座標軸如何對應溫度與濕度，而不是自行推論原因。

## 先知道這件事

主線第 4 節已完成 `record_sensor_data.py` 和 `plot_sensor_data.py`。前者記錄十筆 UART 感測資料，後者建立 PNG 與視窗圖表；本頁只說明既有程式的責任。

## 記錄：將一筆 sensor JSON 寫成一列 CSV

執行位置：Python／電腦  
來源主線檔案：`record_sensor_data.py`

```python linenums="1"
import csv
import json
import time
from datetime import datetime
from pathlib import Path

import serial

PORT = "COM3"  # 改成你的 ESP32 COM 埠
BAUD_RATE = 115200  # Serial鮑率，與ESP32設定相同
RECORD_COUNT = 10  # 數據紀錄筆數
TIMEOUT_SECONDS = 45  # 逾時秒數設定

csv_path = Path("data/sensor.csv")
csv_path.parent.mkdir(exist_ok=True)
file_is_new = not csv_path.exists()  # csv檔不存在時紀錄為新增檔案

# 【1】開啟一次 COM 埠，並清除剛連線前可能留下的資料。
with serial.Serial(PORT, BAUD_RATE, timeout=3) as ser:
    time.sleep(2)  # 等待通訊
    ser.reset_input_buffer()  # 清除輸入緩衝區

    # 【2】以附加模式(append)開啟 CSV；新檔案才寫入欄位名稱。
    #    open的第一個引數"a"代表檔案開啟模式為附加模式，會從檔案尾接續輸入
    with csv_path.open("a", newline="", encoding="utf-8") as csv_file:
        writer = csv.DictWriter(
            csv_file,
            fieldnames=["timestamp", "temp_c", "humidity"],
        )

        #  如果是新檔案才寫入標頭列(欄位名稱)
        if file_is_new:
            writer.writeheader()

        # time.monotonic()會回傳單調遞增的程式執行時間，以秒為單位
        # 假設程式執行後過了5秒我呼叫了此函數，他就會回傳5
        deadline = time.monotonic() + TIMEOUT_SECONDS
        recorded_count = 0

        # 【3】只接受 sensor JSON；錯誤與非 JSON 都不寫入資料列。
        while recorded_count < RECORD_COUNT and time.monotonic() < deadline:
            raw_line = ser.readline().decode("utf-8", errors="replace").strip()

            # 確保讀入的是有效行，非空白行
            if not raw_line:
                continue

            try:
                # 嘗試解析data
                data = json.loads(raw_line)
            except json.JSONDecodeError:
                # 解析失敗，代表data不是json資料，略過本輪迴圈
                print(f"略過非 JSON 資料：{raw_line}")
                continue

            if data.get("type") == "error":
                # 硬體回報錯誤，略過不處理
                print(f"ESP32 回覆錯誤：{data.get('message')}")
                continue

            if data.get("type") != "sensor":
                # 硬體回報類型非sensor，略過不處理
                continue

            # 【4】在電腦收到資料時補上時間，欄位名稱和後續繪圖一致。
            row = {
                "timestamp": datetime.now().isoformat(timespec="seconds"),
                "temp_c": data["temp"],
                "humidity": data["humidity"],
            }
            writer.writerow(row)
            csv_file.flush()  # 將檔案讀寫到記憶體緩衝區的部分寫入硬碟並清理緩衝區
            # 註:為了減少硬碟讀寫次數有時python的檔案讀寫會保留在寄體緩衝中，這一步確保檔案確實寫入硬碟中

            recorded_count += 1
            print(f"已記錄 {recorded_count}/{RECORD_COUNT}：{row}")

if recorded_count < RECORD_COUNT:
    raise RuntimeError("在期限內沒有收到足夠的感測資料")

print(f"完成：資料已寫入 {csv_path}")
```

`DictWriter` 的 `fieldnames` 是資料契約：之後 Pandas 也用相同的 `timestamp`、`temp_c`、`humidity` 讀取。`flush()` 讓每次已成功寫入的資料盡快交給檔案系統；它不會驗證感測器本身是否準確。

## 繪圖：先整理，再建立兩個座標軸

執行位置：Python／電腦  
來源主線檔案：`plot_sensor_data.py`

```python linenums="1"
from pathlib import Path

import matplotlib.pyplot as plt  # 繪圖用套件
import pandas as pd

csv_path = Path("data/sensor.csv")
chart_path = Path("charts/temperature_humidity.png")
# 在工作目錄下建立charts資料夾，引數exist_ok=true代表資料夾存在是沒問題的不需報錯
chart_path.parent.mkdir(exist_ok=True)

data = pd.read_csv(csv_path)

# 【1】無法轉換的欄位變成缺值，避免以文字直接繪圖。
#  詳細的pandas dataframe會用另外的章節補充

#  轉換資料型態為datetime，轉換失敗錯誤時回傳缺失值標記NA/NaN
data["timestamp"] = pd.to_datetime(data["timestamp"], errors="coerce")

#  轉換資料型態為數字型態，轉換失敗錯誤時回傳缺失值標記NA/NaN
data["temp_c"] = pd.to_numeric(data["temp_c"], errors="coerce")

#  轉換資料型態為數字型態，轉換失敗錯誤時回傳缺失值標記NA/NaN
data["humidity"] = pd.to_numeric(data["humidity"], errors="coerce")

# 【2】三個必要欄位完整的資料，才可以進入圖表。
#  這行會回傳一個處理過的dataframe，subset指定的欄位中任何一個資料被標記為缺失值，
#  該筆資料會丟棄，但這不會影響到原始資料，而是一個新的dataframe
valid_data = data.dropna(subset=["timestamp", "temp_c", "humidity"])

if valid_data.empty:
    raise RuntimeError("沒有可繪圖的完整感測資料")

# 【3】同一個時間軸下，分別畫溫度與濕度，避免單位混在一起。
#  建立一個2列1欄的組合圖，共享X軸(此處為時間軸)，圖片大小為9英寸*6英寸
#  回傳值fig:組合圖物件(Figure), axs->包含軸物件(Axes)的陣列容器
#           此處使用解包(unpack)，將容器內的軸物件解包出來
fig, (temp_ax, humidity_ax) = plt.subplots(2, 1, sharex=True, figsize=(9, 6))

#  繪製線與點
temp_ax.plot(
    valid_data["timestamp"],
    valid_data["temp_c"],
    marker="o",
    label="Temperature"
    )
#  設定圖表
temp_ax.set_title("DHT11 Temperature and Humidity")  # 圖表標題
temp_ax.set_ylabel("Temperature (°C)")  # 圖表y軸標籤
temp_ax.grid(True)  # 圖表格線開啟
temp_ax.legend()  #  放置圖例在圖表上

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

fig.autofmt_xdate()  # 日期刻度標籤格式化
fig.tight_layout()  # 調整子圖之間及周圍的內距
fig.savefig(chart_path, dpi=150)  # 將圖表存到指定路徑, dpi為解析度，若要印刷用建議提高到300

print(f"圖表已儲存到：{chart_path}")
plt.show()
```

兩張圖共用時間軸，但各自有 y 軸與單位，因此不會把 °C 和 % 當成同一尺度。`savefig()` 先產生可保留的 PNG；即使某台電腦沒有跳出視窗，仍可用這個檔案確認繪圖是否成功。

## 何時使用、何時不要使用

| 情境 | 建議 |
| --- | --- |
| 想累積多次量測 | 使用 CSV 附加模式，但維持既有欄位名稱。 |
| 欄位轉換失敗或有缺值 | 先檢查資料來源；主線圖表只使用完整列。 |
| 想比較溫度與濕度的變化時間 | 使用共用的時間軸與分開的 y 軸。 |
| 想判斷溫度改變的原因或全天趨勢 | 不要只靠十筆短期資料；需要更多資料與環境脈絡。 |

## 常見誤解

### 「畫出折線就代表資料可信」

折線只把可用數字依時間連起來。若感測器位置改變、手靠近感測器，或資料列有遺漏，圖表仍可能畫得很平順；讀圖前要先確認欄位、時間與缺值。

## 重點整理

- CSV 的固定欄位讓記錄、檢查和繪圖使用同一份資料契約。
- 型別轉換失敗會成為缺值；完整資料列才進入主線圖表。
- 兩個座標軸保留不同單位，共用時間軸以便比較先後。
- 圖表協助觀察，不自動解釋環境變化的原因。

## 回到主線

回到 [將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)，繼續完成核心資料記錄與圖表成果。
