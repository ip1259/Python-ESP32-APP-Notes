# 延伸選讀：pyserial 連線生命週期與 COM 埠除錯

序列埠要在正確時機開啟、等待、清理與關閉，Python 才能可靠接收 ESP32 的資料。

## 你會學到什麼

- 分辨埠名、鮑率與逾時設定的用途。
- 用 `with` 正常關閉 COM 埠。
- 區分逾時空資料與格式不正確的資料。
- 了解序列監控占用埠時，Python 為何不能開啟連線。

## 開始前

| 項目 | 需求 |
| --- | --- |
| 前置教材 | [UART：讓 Python 與 ESP32 雙向通訊](03-uart-python-esp32.md)、[pyserial 埠與資料除錯](附錄-pyserial埠與資料除錯.md) |
| 硬體 | 已接好 DHT11 的 NodeMCU-32S 相容開發板與 USB 資料線 |
| 韌體 | 已燒錄 UART 主線步驟二的 `uart_fixed_json.ino` |
| Python 套件 | `pyserial` |

!!! warning "先關閉序列監控"
    Arduino IDE 的序列監控與 Python 不能同時使用同一個 COM 埠。執行 Python 前，先關閉序列監控與其他讀取序列埠的程式。

!!! info "本頁的實測範圍"
    本頁以 NodeMCU-32S、`115200` 與 `uart_fixed_json.ino` 實測。資料欄位是 `type`、`temp`、`humidity`。COM 編號與開啟埠後的表現會隨電腦、驅動與開發板而變，請以你的觀察結果為準。

## 成功的樣子

程式會顯示 `sensor` JSON；兩筆資料之間也可能顯示逾時，因為韌體不是每一秒都送資料。

```text
已開啟序列埠；讀取 5 次。
第 1 次：{'type': 'sensor', 'temp': 25.0, 'humidity': 60.0}
第 2 次：逾時或不是 sensor JSON
```

## 核心功能速覽

| 功能 | 常見傳入值 | 回傳或副作用 | 本頁用途 |
| --- | --- | --- | --- |
| `serial.Serial()` | 埠名 `str`、鮑率 `int`、`timeout` 秒數 | 開啟序列連線；失敗時拋出 `SerialException` | 指定已確認的 ESP32 埠與 `115200` |
| `readline()` | 無 | `bytes`；逾時時回傳空 bytes | 等待一行 UART 資料 |
| `reset_input_buffer()` | 無 | 捨棄尚未讀取的舊資料 | 讓本次觀察從新資料開始 |
| `with ... as connection` | 連線物件 | 離開區塊時自動關閉 | 避免程式遺留埠占用 |

`timeout` 是最多等待時間，不是 ESP32 的送資料間隔。逾時回傳空 bytes 是可處理的結果，不是 Python 當機。

## 步驟一：確認埠身分與鮑率

目的：不要只看 COM 編號猜測裝置。

1. 插上 ESP32，依拔插前後的裝置變化確認它的 COM 埠。
2. 確認 `uart_fixed_json.ino` 的 `Serial.begin(115200)` 與 Python 都使用 `115200`。
3. 關閉 Arduino IDE 序列監控。

預期結果：你能確認目前連接的 ESP32 埠。這個編號只適用於當前電腦與連接狀態，不要寫進公開作品。

## 步驟二：讀取資料並正常關閉

目的：將開啟、讀取、逾時與關閉寫成可重複執行的最小程式。

執行位置：Python／電腦
檔名：`serial_lifecycle_check.py`

```python
import json

import serial


PORT: str = "請填入你已確認的 COM 埠"
BAUD_RATE: int = 115200
TIMEOUT_SECONDS: float = 1.0


def read_sensor_record(connection: serial.Serial) -> dict[str, object] | None:
    raw_line: bytes = connection.readline()
    if not raw_line:
        return None

    try:
        data: object = json.loads(raw_line.decode("utf-8"))
    except (UnicodeDecodeError, json.JSONDecodeError):
        return None

    if not isinstance(data, dict) or data.get("type") != "sensor":
        return None

    return data


def main() -> None:
    try:
        with serial.Serial(PORT, BAUD_RATE, timeout=TIMEOUT_SECONDS) as connection:
            connection.reset_input_buffer()
            print("已開啟序列埠；讀取 5 次。")
            for number in range(1, 6):
                record: dict[str, object] | None = read_sensor_record(connection)
                if record is None:
                    print(f"第 {number} 次：逾時或不是 sensor JSON")
                else:
                    print(f"第 {number} 次：{record}")
    except serial.SerialException as error:
        print(f"無法開啟序列埠：{error}")


if __name__ == "__main__":
    main()
```

將 `PORT` 改成步驟一確認的值後，在專案資料夾執行：

```powershell
uv run --with pyserial python serial_lifecycle_check.py
```

預期結果：至少一筆資料包含 `type`、`temp`、`humidity`。逾時訊息不可寫入 CSV，也不可當成感測資料。

## 步驟三：判讀重設、緩衝與占用

目的：從觀察結果決定下一個檢查方向。

- `reset_input_buffer()` 只清除 Python 尚未讀取的舊資料，不會清除 ESP32 程式或感測器資料。
- 本頁實測時，正常開啟埠沒有使 ESP32 重設；手動按 `EN`／`RST` 才出現 `boot` 標記。你的板子若開啟後出現開機文字，等待下一筆完整 `sensor` JSON，不要把開機文字當感測資料。
- 序列監控開啟時再執行 Python，預期會出現「存取被拒」或無法開啟埠。關閉序列監控後再試，不要反覆強制開啟。
- 鮑率不一致時可能有亂碼或無法解析文字。`errors="replace"` 只適合除錯顯示，不能寫入 CSV 或送回 ESP32。

## 完成時，應能確認

- 你能以已確認的埠和 `115200` 讀到至少一筆 `sensor` JSON。
- 你能說明空 bytes 是逾時結果，不等於 Python 壞掉。
- 你知道 `with` 結束後會關閉埠，讓下一個程式可使用它。
- 遇到存取被拒時，你會先關閉序列監控，而不是任意更換 COM 埠。

## 常見問題

### 顯示「存取被拒」或無法開啟埠

先關閉 Arduino 序列監控、其他 Python 視窗與序列終端機。仍失敗時，拔插 ESP32 後重新確認埠身分；不要只改成另一個 COM 編號。

### 一直顯示「逾時或不是 sensor JSON」

確認已燒錄 `uart_fixed_json.ino`，兩端都是 `115200`，並先用 Arduino 序列監控確認固定 JSON。完成後關閉序列監控，再執行 Python。

### 開啟後先看到 `boot` 或其他文字

這可能是開機訊息。略過不是 `type: "sensor"` 的資料，等待完整感測 JSON；若持續沒有資料，回到 [UART 主線](03-uart-python-esp32.md) 檢查韌體與接線。

## 重點整理

- `Serial()` 需要已確認的埠、相同鮑率與有限的 `timeout`。
- `readline()` 逾時回傳空 bytes，不應混進感測資料。
- `reset_input_buffer()` 捨棄舊資料，`with` 正常關閉連線。

## 下一步

回到[將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)，把已確認的 `sensor` JSON 交給資料處理流程。
