# 延伸選讀：json 與 csv

把 JSON 文字轉成可使用的資料，再用固定欄位順序寫成 CSV，為 UART 感測資料的後續整理打好基礎。

## 你會學到什麼

- 使用 `json.loads()` 將 JSON 文字解碼為字典。
- 使用 `json.dumps()` 將 Python 資料編碼為 JSON 文字。
- 用 `csv.DictWriter` 寫出固定表頭與多列 CSV 資料。
- 讀寫格式化 JSON 檔，並分辨字串版與檔案版 API。
- 使用 `csv.reader`、`DictReader` 與 `writerow()` 處理一般表格資料。

## 開始前

- 先完成[用 uv 建立 Python 虛擬環境](01-uv與虛擬環境.md)，並能在自己的專案資料夾執行 `uv run python`。
- 建議先閱讀[UART：讓 Python 與 ESP32 雙向通訊](03-uart-python-esp32.md)與[將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)。
- `json`、`csv` 與 `pathlib` 都是 Python 標準函式庫，不需要安裝套件，也不需要連接 ESP32。
- 請在自己的練習或專案資料夾操作。本頁範例會重新建立練習 CSV，不要直接套用到你已收集的感測資料。

## 成功的樣子

執行完整範例後，會產生 `outputs/sensor_data.csv`。以文字編輯器開啟，可看到一列表頭與兩列資料：

```csv
timestamp,temperature_c,humidity_pct
2026-09-11T14:30:00,26.5,61.0
2026-09-11T14:31:00,26.7,60.5
```

終端機會顯示第一筆時間和 `CSV 資料列數：2`。

## 先把 JSON 文字變成字典

JSON（JavaScript Object Notation，JavaScript 物件表示法）常用來傳送有欄位名稱的資料。從 UART 讀到的 JSON 一開始是文字；`json.loads()` 才能把它轉為可依欄位名稱取值的字典。

```python
# 執行位置：Python／電腦
# 檔名：json_csv_demo.py
import json

raw_json = '{"temperature_c": 26.5, "humidity_pct": 61.0}'
sensor_data = json.loads(raw_json)

print(sensor_data["temperature_c"])
print(sensor_data["humidity_pct"])
```

預期結果：依序顯示 `26.5` 與 `61.0`。若 JSON 格式錯誤，例如少了引號或括號，`json.loads()` 會顯示錯誤，不能直接當成資料使用。

### 將資料編碼為 JSON

目的：知道 Python 字典也能轉回 JSON 文字，用於儲存或傳送。

```python
# 執行位置：Python／電腦
# 檔名：json_csv_demo.py
import json

message = {"message": "資料已接收"}
encoded_json = json.dumps(message, ensure_ascii=False)

print(encoded_json)
```

預期結果：顯示 `{"message": "資料已接收"}`。`ensure_ascii=False` 讓中文直接顯示；重新用 `json.loads(encoded_json)` 解碼時，會得到原本的字典。

## JSON 檔：設定與巢狀資料

`loads()` 和 `dumps()` 處理的是字串；檔案已存在時，使用 `load()` 與 `dump()` 會更直接。JSON 可保存字典、串列、字串、數字、`True`／`False` 與 `None` 組成的資料。

```python
# 執行位置：Python／電腦
# 檔名：json_file_demo.py
import json
from pathlib import Path

config = {
    "title": "設定",
    "tags": ["python", "檔案"],
    "enabled": True,
}
json_path = Path("config.json")

with json_path.open("w", encoding="utf-8") as file:
    json.dump(config, file, ensure_ascii=False, indent=2)

with json_path.open("r", encoding="utf-8") as file:
    restored = json.load(file)

print(restored["tags"][0])
print(restored == config)
```

預期結果：顯示 `python` 與 `True`，並建立縮排整齊、可直接閱讀的 `config.json`。`indent=2` 只影響可讀性；寫入同名 JSON 檔會覆寫舊內容，請只用於練習或確認可更新的設定檔。

## 將兩筆資料寫成 CSV

### 1. 建立練習 CSV

目的：把多筆已解碼的感測資料寫成有固定欄位順序的表格檔。

操作：在自己的練習資料夾新增 `json_csv_demo.py`，貼上並執行下列完整程式。

```python
# 執行位置：Python／電腦
# 檔名：json_csv_demo.py
import csv
import json
from pathlib import Path

raw_rows = [
    '{"timestamp": "2026-09-11T14:30:00", "temperature_c": 26.5, "humidity_pct": 61.0}',
    '{"timestamp": "2026-09-11T14:31:00", "temperature_c": 26.7, "humidity_pct": 60.5}',
]
rows = [json.loads(raw_json) for raw_json in raw_rows]

output_folder = Path("outputs")
output_folder.mkdir(parents=True, exist_ok=True)
csv_path = output_folder / "sensor_data.csv"

fieldnames = ["timestamp", "temperature_c", "humidity_pct"]
with csv_path.open("w", encoding="utf-8", newline="") as file:
    writer = csv.DictWriter(file, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(rows)

with csv_path.open("r", encoding="utf-8", newline="") as file:
    saved_rows = list(csv.DictReader(file))

print(f"第一筆時間：{saved_rows[0]['timestamp']}")
print(f"CSV 資料列數：{len(saved_rows)}")
```

預期結果：終端機顯示第一筆時間 `2026-09-11T14:30:00` 和 `CSV 資料列數：2`；`outputs` 資料夾中可找到 `sensor_data.csv`。

### 2. 看懂固定欄位順序

目的：知道 `fieldnames` 是 CSV 的表頭與輸出順序。

操作：查看程式中的 `fieldnames`：

```python
# 執行位置：Python／電腦
# 檔名：json_csv_demo.py
fieldnames = ["timestamp", "temperature_c", "humidity_pct"]
```

預期結果：CSV 的第一行依相同順序顯示三個欄位。每筆 `rows` 字典都必須包含這些欄位；若多出未列入的欄位，`DictWriter` 會顯示錯誤，提醒你先決定 CSV 要保留哪些資料。

## CSV 的其他常用讀寫方式

### 用 `reader` 讀取沒有欄位名稱的表格

`csv.reader()` 將每一列讀成串列，適合欄位由位置決定的簡單資料。

```python
# 執行位置：Python／電腦
# 檔名：csv_reader_demo.py
import csv
from pathlib import Path

csv_path = Path("scores.csv")
csv_path.write_text("Ada,90\nBen,82\n", encoding="utf-8")

with csv_path.open("r", encoding="utf-8", newline="") as file:
    for row in csv.reader(file):
        print(f"姓名：{row[0]}，分數：{row[1]}")
```

預期結果：顯示 Ada 和 Ben 的姓名與分數。這個範例的 `write_text()` 會重建 `scores.csv`，請勿用在既有成績或資料檔。

### `writerow()`、`writerows()` 與含逗號的文字

`writerow()` 寫一列，`writerows()` 寫多列。CSV 模組會為含逗號的欄位加上必要的引號，讀回時仍是原本的文字。

```python
# 執行位置：Python／電腦
# 檔名：csv_writer_demo.py
import csv
from pathlib import Path

csv_path = Path("people.csv")
with csv_path.open("w", encoding="utf-8", newline="") as file:
    writer = csv.DictWriter(file, fieldnames=["name", "note"])
    writer.writeheader()
    writer.writerow({"name": "Ada", "note": "喜歡, Python"})
    writer.writerows([{"name": "Ben", "note": "完成"}])

with csv_path.open("r", encoding="utf-8", newline="") as file:
    rows = list(csv.DictReader(file))

print(rows[0]["note"])
print(len(rows))
```

預期結果：顯示 `喜歡, Python` 與 `2`。不要自己以字串拼接 CSV 欄位；資料中有逗號、引號或換行時，讓 `csv` 模組處理格式才不會錯欄。

### 額外欄位的選擇

預設 `DictWriter` 遇到 `fieldnames` 未列出的欄位會報錯，這通常有助於發現資料格式改變。若你已確認只需要部分欄位，可明確設定 `extrasaction="ignore"`：

```python
# 執行位置：Python／電腦
# 檔名：csv_writer_demo.py
writer = csv.DictWriter(
    file,
    fieldnames=["timestamp", "temperature_c"],
    extrasaction="ignore",
)
```

預期結果：例如 `humidity_pct` 等額外欄位不會寫入 CSV。這不是自動修正資料；被忽略的值會遺失，使用前必須確認這正是你要的欄位選擇。

## 選做練習：加入第三筆資料

這是延伸練習，不做也不影響主線成果。

目的：練習將第三筆合法 JSON 加入清單，再由 `writer.writerows(rows)` 一次寫出全部資料。

操作：在 `raw_rows` 最後加入一筆具有相同三個欄位的 JSON，例如時間為 `2026-09-11T14:32:00` 的資料，重新執行程式。

預期結果：終端機顯示 `CSV 資料列數：3`，CSV 則有一列表頭與三列資料。

??? tip "參考作法：第三筆 JSON"
    ```python
    # 執行位置：Python／電腦
    # 檔名：json_csv_demo.py
    '{"timestamp": "2026-09-11T14:32:00", "temperature_c": 26.8, "humidity_pct": 60.0}',
    ```

    將這一行放在 `raw_rows` 清單的第二筆資料後方。記得上一筆資料行尾要保留逗號。

## 完成時，應能確認

- 你能用 `json.loads()` 將 JSON 文字轉成字典，並以欄位名稱讀出值。
- 你能說明 `json.dumps()` 會將 Python 資料轉回 JSON 文字。
- 你開啟 `outputs/sensor_data.csv` 時，看到固定順序的表頭和兩列資料。
- 你知道 `newline=""` 是使用 `csv` 模組寫檔時的固定寫法，可避免 Windows 出現多餘空白列。
- 你能說明 `load`／`dump` 處理 JSON 檔，`loads`／`dumps` 處理 JSON 字串。
- 你知道 `csv.reader` 讀出串列，`DictReader` 讀出可用欄位名稱取值的字典。

## 常見問題

### `json.decoder.JSONDecodeError`

先檢查 JSON 文字的雙引號、逗號與大括號是否完整。JSON 的欄位名稱和文字值必須使用雙引號，例如 `"temperature_c"`，不能改成單引號。

### CSV 出現多餘空白列

先確認開啟檔案時使用 `newline=""`，也確認是使用 `csv.DictWriter` 寫入。不要自行在每一列字串後面再加 `\n`。

### `ValueError: dict contains fields not in fieldnames`

表示某一筆字典含有 CSV 表頭未列出的欄位。先比較該筆資料和 `fieldnames`；要保留新欄位時，把它加入 `fieldnames`，否則先不要寫入該欄位。

### 找不到 `outputs/sensor_data.csv`

相對路徑會從你執行程式時的目前資料夾開始計算。先確認終端機所在位置，再在該資料夾中找 `outputs`。

### JSON 寫入後檔案內容變成一行或被覆寫

沒有指定 `indent` 時，JSON 仍是合法的，只是較不容易閱讀；加上 `indent=2` 可格式化。`json.dump()` 以 `"w"` 開啟同名檔時會覆寫舊內容，請先確認檔案用途。

## 重點整理

- `json.loads()` 將 JSON 文字轉成字典；`json.dumps()` 將 Python 資料轉成 JSON 文字。
- `csv.DictWriter` 配合 `fieldnames` 可建立固定表頭與欄位順序。
- 使用 `encoding="utf-8"` 和 `newline=""` 開啟 CSV，可讓文字和換行在 Windows 上穩定寫入。
- `load`／`dump` 用於 JSON 檔，`loads`／`dumps` 用於 JSON 字串；`indent` 和 `ensure_ascii` 可改善 JSON 的可讀性。
- `reader`、`DictReader`、`writerow` 與 `writerows` 能處理一般表格資料；遇到額外欄位時，先判斷是否應保留，而不是無條件忽略。

## 下一步

回到[UART：讓 Python 與 ESP32 雙向通訊](03-uart-python-esp32.md)確認 JSON 從哪裡來，再回到[將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)查看 CSV 如何用於圖表。接下來可閱讀「即將推出：datetime 與 time」，認識感測資料的時間資訊。
