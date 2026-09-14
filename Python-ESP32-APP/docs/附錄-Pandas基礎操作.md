# 延伸選讀：Pandas 基礎操作

用 DataFrame（有欄位名稱的表格）檢視固定 CSV，先確認資料長什麼樣子，再決定後續清理或繪圖。

## 你會學到什麼

- 使用 `pd.read_csv()` 讀取 CSV 成為 DataFrame。
- 使用欄位名稱選擇感測資料，並查看前後幾列。
- 使用 `pd.to_numeric()` 將文字安全轉成數值欄位。
- 使用 `describe()` 觀察數值欄位的筆數、平均與範圍。

## 開始前

- 建議先完成[將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)。
- 本頁使用固定練習資料，不連接 ESP32，也不讀取或覆寫你的真實感測 CSV。
- 需要 `pandas` 套件；請在自己的 uv 練習環境安裝後執行。
- 範例會重建 `outputs/pandas_sensor_data.csv`，只適用於可控制的練習資料夾。

## 成功的樣子

程式會顯示 `資料列數：3`、三個欄位名稱、轉型後的 `NaN` 數量為 `1`，以及溫度的摘要。`NaN` 表示缺少或無法轉成數值，不是溫度 `0`。

## 核心功能速覽

先知道每個功能的工作、輸入與結果，再閱讀下一節完整程式。`DataFrame` 是有列與欄位名稱的表格；`Series` 是 DataFrame 中的一欄資料。

### `pd.read_csv()`：將 CSV 讀成表格

`pd.read_csv()` 讀取 CSV 檔案，建立 DataFrame。最常見的傳入值是檔案路徑，例如 `Path("outputs/sensor_data.csv")` 或文字路徑；常用選項有 `encoding="utf-8"`。它回傳 `pd.DataFrame`，之後才能依欄位名稱查看、轉型或統計資料。

```python
# 執行位置：Python／電腦
# 檔名：pandas_basics_demo.py
data_frame: pd.DataFrame = pd.read_csv(csv_path)
```

### `data_frame["欄位名"]`：選取一欄

方括號內傳入欄位名稱字串，例如 `"temperature_c"`。它回傳 `pd.Series`，也就是單一欄的資料；欄位名稱拼錯時會出現 `KeyError`，因此先用 `list(data_frame.columns)` 確認表頭。

```python
# 執行位置：Python／電腦
# 檔名：pandas_basics_demo.py
temperature_text: pd.Series = data_frame["temperature_c"]
```

若要同時選取多欄，傳入欄位名稱清單，例如 `data_frame[["timestamp", "temperature_c"]]`；這次回傳的是新的 DataFrame，而不是 Series。

### `pd.to_numeric()`：安全轉成數值

`pd.to_numeric()` 將文字或 Series 轉為數值。最常見的第一個傳入值是一欄文字資料；`errors="coerce"` 表示遇到 `"unknown"` 這類無法轉換的值時，回傳缺值 `NaN`，而不是讓程式立即中斷。傳入 Series 時回傳數值 Series。

```python
# 執行位置：Python／電腦
# 檔名：pandas_basics_demo.py
data_frame["temperature_value"] = pd.to_numeric(
    temperature_text,
    errors="coerce",
)
```

`NaN` 是發現資料問題的訊號，不是自動修正結果；不可直接當成 `0` 寫入或拿去解讀趨勢。

### `head()`、`tail()` 與 `describe()`：先觀察再處理

- `data_frame.head(2)` 傳入要看的前幾列數量，回傳包含前兩列的 DataFrame；不傳值時預設是前 5 列。
- `data_frame.tail(1)` 傳入要看的最後幾列數量，回傳包含最後一列的 DataFrame；不傳值時預設是最後 5 列。
- `data_frame["temperature_value"].describe()` 不需額外傳入值，回傳數值摘要 Series，例如有效筆數 `count`、平均 `mean`、最小值 `min` 與最大值 `max`。缺值不會列入 `count`。

這些功能都只用來觀察或產生新結果，不會自動修改原本 CSV。

## 建立固定 CSV 並讀成 DataFrame

目的：從可重現的小型資料開始認識欄位與列。

操作：建立 `pandas_basics_demo.py`，貼上並執行。

```python
# 執行位置：Python／電腦
# 檔名：pandas_basics_demo.py
from pathlib import Path

import pandas as pd


output_folder: Path = Path("outputs")
output_folder.mkdir(exist_ok=True)
csv_path: Path = output_folder / "pandas_sensor_data.csv"
csv_path.write_text(
    "timestamp,temperature_c,humidity_pct\n"
    "2026-09-14T09:00:00,26.5,61.0\n"
    "2026-09-14T09:01:00,unknown,60.5\n"
    "2026-09-14T09:02:00,27.1,60.0\n",
    encoding="utf-8",
)

data_frame: pd.DataFrame = pd.read_csv(csv_path)
print(f"資料列數：{len(data_frame)}")
print(f"欄位：{list(data_frame.columns)}")
print(data_frame.head(2))
print(data_frame.tail(1))
```

預期結果：顯示 3 列資料、`timestamp`、`temperature_c`、`humidity_pct` 三個欄位，以及前兩列和最後一列。`head()`、`tail()` 是先觀察資料的工具；不要只因 CSV 能讀進來就假設每個值都正確。

## 選取欄位並安全轉成數值

目的：保留原始文字，另建立可供統計的數值欄位。

操作：在同一檔案的讀取程式後加入：

```python
# 執行位置：Python／電腦
# 檔名：pandas_basics_demo.py
temperature_text: pd.Series = data_frame["temperature_c"]
data_frame["temperature_value"] = pd.to_numeric(temperature_text, errors="coerce")
missing_count: int = int(data_frame["temperature_value"].isna().sum())

print(f"無法轉成數值的筆數：{missing_count}")
print(data_frame[["timestamp", "temperature_value"]])
print(data_frame["temperature_value"].describe())
```

預期結果：顯示 `無法轉成數值的筆數：1`，第二列的 `temperature_value` 是 `NaN`。`errors="coerce"` 不會修好資料，而是把問題標示為缺值；不要改成 `0`，否則會把錯誤資料偽裝成真實溫度。下一篇才會處理缺值該保留、排除或修正。

## 選做練習：查看濕度欄位摘要

這是延伸練習，不做也不影響主線成果。

目的：練習以欄位名稱取得 Series 並查看數值概況。

操作：將最後一行改成 `print(data_frame["humidity_pct"].describe())`。

預期結果：摘要的 `count` 是 `3`，且可看到平均濕度。這是固定練習資料，不能直接推論真實環境趨勢。

## 完成時，應能確認

- 你能讀取 CSV、列出欄位，並用 `head()`、`tail()` 檢視列。
- 你能以欄位名稱選擇 Series。
- 你知道無法轉成數字時 `NaN` 是缺值標示，不是 0。
- 你能用 `describe()` 查看數值欄位的基本摘要。

## 常見問題

### `ModuleNotFoundError: No module named 'pandas'`

確認在同一個 uv 環境安裝並執行 `pandas`，不要只在其他 Python 環境安裝。

### 欄位名稱找不到

先印出 `list(data_frame.columns)`，確認 CSV 表頭拼字、大小寫與空白。不要猜測欄位位置。

### 為什麼 `describe()` 的 count 少於資料列數？

數值欄位含有 `NaN` 時，摘要不將它當成有效數值。先保留原始欄位與缺值資訊，下一篇再決定處理方式。

## 重點整理

- DataFrame 能以欄位名稱操作 CSV，但不會自動保證資料品質。
- `head()`、`tail()` 和 `describe()` 先幫你觀察資料。
- 轉型失敗應保留為缺值，不能用假數值掩蓋。

## 下一步

回到[將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)對照資料來源。接著可閱讀[延伸選讀：Pandas 清理、篩選與彙整](附錄-Pandas清理篩選與彙整.md)，將缺值、條件篩選與每日摘要分開處理。
