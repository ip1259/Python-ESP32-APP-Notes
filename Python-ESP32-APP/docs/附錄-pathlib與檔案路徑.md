# 延伸選讀：pathlib 與檔案路徑

使用 `pathlib` 管理 CSV 和圖表的輸出位置，讓程式不必依賴特定電腦的資料夾寫法。

## 你會學到什麼

- 使用 `Path` 表示資料夾與檔案路徑。
- 用 `/` 組合資料夾與檔名。
- 建立輸出資料夾，並確認 CSV 檔是否存在。

## 開始前

- 先完成[用 uv 建立 Python 虛擬環境](01-uv與虛擬環境.md)，並能在自己的專案資料夾執行 `uv run python`。
- 建議先閱讀主線的[將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)，了解 CSV 與圖表會產生哪些輸出。
- `pathlib` 是 Python 標準函式庫，不需要安裝套件，也不需要連接 ESP32。
- 請在自己的練習或專案資料夾操作。若畫面顯示沒有寫入權限，請停止在該位置建立檔案，改到自己的專案資料夾；不需要以系統管理員身分執行。

## 成功的樣子

執行範例後，目前資料夾會出現 `outputs` 資料夾，裡面有一個空白的 `sensor_data.csv`。終端機會顯示類似下列結果：

```text
CSV 路徑：outputs\sensor_data.csv
資料夾存在：True
CSV 是檔案：True
```

這個空白 CSV 只用來練習路徑，不會覆寫你在主線收集的感測資料。

## 先認識 `Path`

路徑是「資料夾或檔案在哪裡」的資訊。`Path` 能把它表示成 Python 物件；以 `/` 連接時，左邊是資料夾，右邊是要放進去的子資料夾或檔名。

```python
# 執行位置：Python／電腦
# 檔名：path_demo.py
from pathlib import Path

output_folder = Path("outputs")
csv_path = output_folder / "sensor_data.csv"

print(csv_path)
```

執行後，在 Windows 常會看到 `outputs\sensor_data.csv`。程式中一律使用 `/` 組合路徑即可，不要自己把資料夾名稱和 `\` 字串接在一起。

## 建立輸出資料夾與檔案

### 1. 建立練習檔

目的：準備一個可重複執行的輸出位置，並確認路徑指向的項目存在。

操作：在自己的練習資料夾新增 `path_demo.py`，貼上並執行下列程式。

```python
# 執行位置：Python／電腦
# 檔名：path_demo.py
from pathlib import Path

output_folder = Path("outputs")
csv_path = output_folder / "sensor_data.csv"

# 第一次建立資料夾；已存在時也不會報錯。
output_folder.mkdir(parents=True, exist_ok=True)

# 只在不存在時建立空白檔案，不會清空既有內容。
csv_path.touch(exist_ok=True)

print(f"CSV 路徑：{csv_path}")
print(f"資料夾存在：{output_folder.exists()}")
print(f"CSV 是檔案：{csv_path.is_file()}")
```

預期結果：終端機顯示 CSV 路徑，並顯示兩個 `True`；檔案總管可看到 `outputs/sensor_data.csv`。

### 2. 將概念接回感測資料輸出

目的：知道路徑變數可直接交給之後的檔案處理程式使用。

操作：當你未來要將主線的 CSV 或 PNG 放進輸出資料夾時，先建立資料夾，再把檔名接到 `output_folder` 後面。例如：

```python
# 執行位置：Python／電腦
# 檔名：path_demo.py
from pathlib import Path

output_folder = Path("outputs")
output_folder.mkdir(parents=True, exist_ok=True)

csv_path = output_folder / "sensor_data.csv"
chart_path = output_folder / "temperature_chart.png"

print(csv_path)
print(chart_path)
```

預期結果：會依序顯示 `outputs` 底下的 CSV 與 PNG 路徑。這一段只建立資料夾和路徑，不會產生圖表；請繼續使用主線第 4 節已驗證的程式來寫入資料與繪圖。

## 選做練習：用日期建立輸出資料夾

這是延伸練習，不做也不影響主線成果。

目的：讓每一天的輸出放到可從資料夾名稱辨識的日期位置。

操作：新增 `dated_path_demo.py`，執行下列程式。

```python
# 執行位置：Python／電腦
# 檔名：dated_path_demo.py
from datetime import date
from pathlib import Path

dated_folder = Path("outputs") / date.today().isoformat()
dated_folder.mkdir(parents=True, exist_ok=True)

print(f"今天的輸出資料夾：{dated_folder}")
```

預期結果：會建立如 `outputs/2026-09-11` 的資料夾；日期會依你執行當天而改變。

??? tip "參考作法：在日期資料夾建立 CSV 路徑"
    ```python
    # 執行位置：Python／電腦
    # 檔名：dated_path_demo.py
    from datetime import date
    from pathlib import Path

    dated_folder = Path("outputs") / date.today().isoformat()
    dated_folder.mkdir(parents=True, exist_ok=True)
    csv_path = dated_folder / "sensor_data.csv"

    print(csv_path)
    ```

    輸出會是如 `outputs/2026-09-11/sensor_data.csv` 的路徑；程式只顯示路徑，尚未建立 CSV 檔。

## 完成時，應能確認

- 你能用 `Path("outputs") / "sensor_data.csv"` 組合檔案路徑。
- 你執行 `path_demo.py` 後，看到 `outputs` 資料夾和空白 `sensor_data.csv`。
- 你能說明 `exists()` 是確認路徑是否存在，`is_file()` 是確認它是否為檔案。
- 你知道 `mkdir(parents=True, exist_ok=True)` 可以讓資料夾已存在時仍安全重複執行。

## 常見問題

### 執行後找不到 `outputs` 資料夾

先確認你是從哪個資料夾執行 `python path_demo.py`。相對路徑會從目前終端機所在的資料夾開始計算；請在該資料夾或其專案資料夾中尋找 `outputs`。

### 顯示 `PermissionError` 或沒有寫入權限

先確認你不是在系統資料夾或唯讀位置執行程式。將 `path_demo.py` 放到自己的專案資料夾後重新執行，不要用系統管理員權限繞過限制。

### `CSV 是檔案：False`

先確認程式中的 `csv_path.touch(exist_ok=True)` 沒有被刪除，且前面的 `output_folder.mkdir(...)` 已先執行。若你把同名位置手動建立成資料夾，請改用其他檔名，避免檔案與資料夾同名。

## 重點整理

- `Path` 讓路徑成為清楚的 Python 物件。
- 使用 `/` 組合子資料夾與檔名，不必手動處理 Windows 的 `\`。
- 寫檔前先用 `mkdir(parents=True, exist_ok=True)` 準備輸出資料夾，再用 `exists()` 或 `is_file()` 檢查結果。

## 下一步

回到[將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)，把已驗證的 CSV 與圖表流程和輸出路徑概念一起使用。接下來可閱讀「即將推出：json 與 csv」，了解資料格式如何在 Python 中轉換。
