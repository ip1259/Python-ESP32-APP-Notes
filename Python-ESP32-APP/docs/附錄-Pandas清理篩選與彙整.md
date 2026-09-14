# 延伸選讀：Pandas 清理、篩選與彙整

把感測 CSV 的缺值、篩選條件與每日摘要分開處理，讓每一次資料取捨都看得見。

## 你會學到什麼

- 使用 `dropna(subset=...)` 只移除本次分析必要欄位的缺值列。
- 使用布林條件與 `.loc[...]` 篩選符合門檻的資料。
- 使用 `sort_values()` 依指定欄位排序，方便核對結果。
- 使用 `pd.to_datetime()`、`groupby()` 與 `agg()` 做每日平均與筆數摘要。

## 開始前

- 建議先完成[延伸選讀：Pandas 基礎操作](附錄-Pandas基礎操作.md)，特別是 `NaN` 與數值轉型的意義。
- 本頁不連接 ESP32，也不讀取、修改或覆寫你的主線感測 CSV。
- 需要 `pandas` 套件與可執行 uv 的 Python 練習環境。
- 範例會重建 `outputs/pandas_cleaning_demo.csv`；請只在可控制的練習資料夾中執行。

## 成功的樣子

程式會顯示清理前 `5` 列、清理後 `3` 列；溫度至少 `27.0` 的篩選結果有 `2` 列。每日彙整會有 `2026-09-14` 與 `2026-09-15` 兩天，每天各有一筆有效資料。

## 核心功能速覽

### `dropna(subset=...)`：只移除必要欄位缺值的列

`dropna()` 會回傳移除缺值後的新 DataFrame，不會自動修改原本的 DataFrame。最常見的 `subset` 傳入欄位名稱清單，例如 `["temperature_c", "humidity_pct"]`；只有這些欄位缺值的列會被移除。本頁用它建立統計用資料，而原始資料仍保留供追查。

若不指定 `subset`，任何欄位的缺值都可能讓整列被移除。先確認本次問題真正需要哪些欄位，不要把 `dropna()` 當作一律刪除資料的按鈕。

### `.loc[條件]`：依布林條件挑出資料列

`.loc[...]` 是依標籤或條件選取資料的方式。當中傳入像 `clean_data["temperature_c"] >= 27.0` 的布林 Series 時，回傳條件為真的資料列 DataFrame。它只建立篩選結果，不會改變 `clean_data`。

條件門檻必須符合你的分析目的；這裡的 `27.0` 只用來示範，不能當成所有環境的警戒標準。

### `sort_values()`：依欄位排序以便檢查

`sort_values(by="humidity_pct")` 傳入排序欄位名稱，回傳重新排列的新 DataFrame。常用選項 `ascending=False` 表示由大到小。本頁依濕度由高到低排序，確認篩選結果的次序；它不會自動排序 CSV 檔案。

### `pd.to_datetime()`、`groupby()` 與 `agg()`：把時間資料做成摘要

`pd.to_datetime()` 將文字時間欄位轉成日期時間 Series；本頁傳入 `errors="coerce"`，格式錯誤會變成 `NaT`，方便先發現問題。`.dt.date` 再取得每筆資料的日期。

`data_frame.groupby("date")` 依 `date` 欄位分組，回傳可進一步彙整的群組物件。接著 `.agg(...)` 傳入「輸出欄名＝(來源欄位, 統計方式)」的設定，回傳每組一列的新 DataFrame。本頁計算平均溫度、平均濕度與有效筆數；小樣本摘要只適合練習，不能取代長期量測。

## 建立資料並安全清理

目的：保留原始資料，並明確記錄這次分析排除了哪些缺值。

操作：建立 `pandas_cleaning_demo.py`，貼上並執行。

```python
# 執行位置：Python／電腦
# 檔名：pandas_cleaning_demo.py
from pathlib import Path

import pandas as pd


output_folder: Path = Path("outputs")
output_folder.mkdir(exist_ok=True)
csv_path: Path = output_folder / "pandas_cleaning_demo.csv"
csv_path.write_text(
    "timestamp,temperature_c,humidity_pct\n"
    "2026-09-14T09:00:00,26.5,61.0\n"
    "2026-09-14T10:00:00,unknown,60.5\n"
    "2026-09-14T11:00:00,27.4,\n"
    "2026-09-14T12:00:00,27.1,59.0\n"
    "2026-09-15T09:00:00,28.0,63.0\n",
    encoding="utf-8",
)

raw_data: pd.DataFrame = pd.read_csv(csv_path)
working_data: pd.DataFrame = raw_data.copy()
working_data["temperature_c"] = pd.to_numeric(
    working_data["temperature_c"],
    errors="coerce",
)
working_data["humidity_pct"] = pd.to_numeric(
    working_data["humidity_pct"],
    errors="coerce",
)
clean_data: pd.DataFrame = working_data.dropna(
    subset=["temperature_c", "humidity_pct"],
).copy()

print(f"原始資料列數：{len(raw_data)}")
print(f"清理後資料列數：{len(clean_data)}")
print(clean_data[["timestamp", "temperature_c", "humidity_pct"]])
```

預期結果：顯示原始資料 `5` 列、清理後 `3` 列。`unknown` 溫度與空白濕度所在的兩列不在 `clean_data`，但仍保留於 `raw_data` 與 `working_data`；不要直接覆寫原始 CSV。

## 篩選並排序有效資料

目的：以可讀的條件找出要觀察的資料，並用排序核對結果。

操作：接在同一檔案後面加入：

```python
# 執行位置：Python／電腦
# 檔名：pandas_cleaning_demo.py
warm_data: pd.DataFrame = clean_data.loc[
    clean_data["temperature_c"] >= 27.0
].copy()
sorted_warm_data: pd.DataFrame = warm_data.sort_values(
    by="humidity_pct",
    ascending=False,
)

print(f"溫度至少 27.0 °C 的列數：{len(sorted_warm_data)}")
print(sorted_warm_data[["timestamp", "temperature_c", "humidity_pct"]])
```

預期結果：顯示 `溫度至少 27.0 °C 的列數：2`。結果先顯示濕度 `63.0` 的 9 月 15 日資料，再顯示濕度 `59.0` 的 9 月 14 日資料。篩選結果是分析用資料，不代表原始資料已被修正。

## 依日期彙整平均值與筆數

目的：將多筆有效資料變成每天一列的摘要，先觀察資料量再解讀平均值。

操作：接在同一檔案後面加入：

```python
# 執行位置：Python／電腦
# 檔名：pandas_cleaning_demo.py
clean_data["timestamp"] = pd.to_datetime(
    clean_data["timestamp"],
    errors="coerce",
)
dated_data: pd.DataFrame = clean_data.dropna(subset=["timestamp"]).copy()
dated_data["date"] = dated_data["timestamp"].dt.date

daily_summary: pd.DataFrame = (
    dated_data.groupby("date")
    .agg(
        average_temperature_c=("temperature_c", "mean"),
        average_humidity_pct=("humidity_pct", "mean"),
        reading_count=("temperature_c", "count"),
    )
    .reset_index()
)

print(daily_summary)
```

預期結果：摘要有兩列。`2026-09-14` 的平均溫度為 `26.8`、平均濕度為 `60.0`、筆數為 `2`；`2026-09-15` 則為 `28.0`、`63.0`、`1`。平均值是否有意義取決於有效筆數；先看 `reading_count`，不要只看平均。

## 選做練習：只檢視濕度至少 60 的資料

這是延伸練習，不做也不影響主線成果。

目的：自行建立一個新的條件篩選結果，而不影響前面的資料。

操作：使用 `clean_data.loc[...]` 建立 `humid_data`，條件為 `humidity_pct >= 60.0`，再印出列數與三個感測欄位。

預期結果：固定資料中有 `2` 列符合條件。請先檢查條件與列數是否一致；不可把這個結果寫回原始 CSV。

## 完成時，應能確認

- 你能說明 `dropna(subset=...)` 為何比未指定欄位的 `dropna()` 更容易控制。
- 你能用 `.loc[...]` 建立條件篩選結果，且知道它不會修改原始資料。
- 你能用 `sort_values()` 檢查資料的排列順序。
- 你能依日期用 `groupby().agg()` 取得平均值與有效筆數。

## 常見問題

### 清理後的列數比預期少

先印出 `working_data`，查看哪些欄位成為 `NaN`；再檢查 `subset` 是否誤放了本次分析不需要的欄位。不要在不知道原因時直接填入 `0`。

### 篩選時出現 `TypeError` 或結果不正確

確認比較前已用 `pd.to_numeric(..., errors="coerce")` 將欄位轉為數值，並檢查條件門檻與欄位名稱。文字 `"27.0"` 與數值 `27.0` 的比較方式不同。

### 日期彙整後出現 `NaT` 或少了一天

先查看 `clean_data["timestamp"]`，確認原始時間格式是否一致。`errors="coerce"` 會把格式錯誤標成 `NaT`，這是提示你先修正或另行處理資料，不是日期 `0`。

## 重點整理

- 清理前保留原始資料，並以 `subset` 明示這次分析依賴的欄位。
- 篩選、排序與彙整都會建立分析結果，不會自動改寫 CSV。
- 每日平均必須搭配有效筆數解讀；小型練習資料不能代表長期趨勢。

## 下一步

回到[將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)連結本頁的資料處理位置。接著可閱讀「即將推出：Matplotlib 圖表基本元件」。
