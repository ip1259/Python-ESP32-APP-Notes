# 延伸選讀：Matplotlib 可讀性與輸出

把溫度與濕度分成兩個共用時間軸的子圖，再輸出一張容易閱讀的 PNG。

## 你會學到什麼

- 使用 `plt.subplots(2, 1, sharex=True)` 建立上下兩個子圖。
- 使用 `DateFormatter` 控制時間軸日期格式。
- 使用 `tight_layout()` 減少標籤重疊。
- 用 `savefig()` 輸出可分享的 PNG。

## 開始前

- 建議先完成[延伸選讀：Matplotlib 圖表基本元件](附錄-Matplotlib圖表基本元件.md)。
- 本頁使用固定練習資料，不連接 ESP32，也不讀取或覆寫主線 CSV。
- 需要 `matplotlib` 與 uv；輸出位置是 `outputs/matplotlib_readability.png`。

## 成功的樣子

PNG 有上下兩張圖：上方是 Temperature (°C)，下方是 Humidity (%)；兩張圖共用時間軸，日期以 `09:00` 這類時間呈現，且標籤沒有互相遮住。

## 核心功能速覽

### `plt.subplots()`：分開不同單位的資料

`plt.subplots(2, 1, sharex=True, figsize=(8, 6))` 回傳一張 Figure 與兩個 Axes。`2, 1` 表示兩列一欄；`sharex=True` 讓兩張圖用相同時間軸，但各自保有不同縱軸。

### `DateFormatter`：指定時間軸顯示方式

`mdates.DateFormatter("%H:%M")` 建立日期格式器；`%H:%M` 代表 24 小時制的時與分。用 `set_major_formatter()` 設到橫軸後，資料的完整日期時間仍存在，只是顯示文字變得精簡。

### `tight_layout()` 與 `savefig()`：調整版面並輸出

`fig.tight_layout()` 自動調整子圖間距，降低標題和標籤重疊的機會。`fig.savefig(path, dpi=150)` 輸出 PNG；`dpi` 越高檔案通常越大，150 適合一般練習與分享。

## 建立可讀的雙子圖

目的：比較兩種不同單位的感測資料，同時保持相同時間順序。

操作：建立 `matplotlib_readability_demo.py`，貼上並執行。

```python
# 執行位置：Python／電腦
# 檔名：matplotlib_readability_demo.py
from datetime import datetime
from pathlib import Path

import matplotlib

matplotlib.use("Agg")

import matplotlib.dates as mdates
import matplotlib.pyplot as plt


timestamps: list[datetime] = [
    datetime(2026, 9, 14, 9, 0),
    datetime(2026, 9, 14, 9, 10),
    datetime(2026, 9, 14, 9, 20),
    datetime(2026, 9, 14, 9, 30),
]
temperatures_c: list[float] = [26.5, 26.8, 27.1, 27.0]
humidities_pct: list[float] = [61.0, 60.0, 59.0, 60.5]
output_folder: Path = Path("outputs")
output_folder.mkdir(exist_ok=True)
chart_path: Path = output_folder / "matplotlib_readability.png"

fig, (temp_ax, humidity_ax) = plt.subplots(2, 1, sharex=True, figsize=(8, 6))
temp_ax.plot(timestamps, temperatures_c, marker="o", color="tab:red")
temp_ax.set_title("Temperature and Humidity")
temp_ax.set_ylabel("Temperature (°C)")
temp_ax.grid(True, linestyle="--", alpha=0.5)

humidity_ax.plot(timestamps, humidities_pct, marker="o", color="tab:blue")
humidity_ax.set_xlabel("Time")
humidity_ax.set_ylabel("Humidity (%)")
humidity_ax.grid(True, linestyle="--", alpha=0.5)
humidity_ax.xaxis.set_major_formatter(mdates.DateFormatter("%H:%M"))

fig.tight_layout()
fig.savefig(chart_path, dpi=150)
plt.close(fig)
print(f"圖表已建立：{chart_path}")
```

預期結果：顯示 `圖表已建立：outputs\\matplotlib_readability.png`。開啟 PNG 後，上圖溫度約在 26.5 至 27.1 間變化，下圖濕度約在 59 至 61 間變化。不要只看線條顏色；縱軸標籤與單位才是資料意義。

## 選做練習：改變輸出解析度

這是延伸練習，不做也不影響主線成果。

目的：觀察 `dpi` 對檔案大小與清晰度的影響。

操作：將 `dpi=150` 改成 `dpi=100`，輸出成另一個檔名，例如 `matplotlib_readability_100dpi.png`。

預期結果：新檔案通常較小；先核對檔名，避免覆寫原本輸出的圖。

## 完成時，應能確認

- 你能理解為何不同單位適合使用不同子圖。
- 你能用 `sharex=True` 讓子圖對照相同時間。
- 你能用 `DateFormatter` 讓時間標籤更精簡。
- 你能輸出並檢查一張沒有重疊標籤的 PNG。

## 常見問題

### 日期標籤顯示成完整日期時間或互相重疊

確認 `humidity_ax.xaxis.set_major_formatter(...)` 使用 `DateFormatter("%H:%M")`，並保留 `fig.tight_layout()`。

### 兩張圖看起來使用相同縱軸

確認是用兩個 Axes 設定各自的 `set_ylabel()`。`sharex=True` 只共用橫軸，不會共用縱軸。

### 找不到 PNG

相對路徑以執行指令的資料夾為準。先查看印出的 `chart_path`，並確認 `outputs` 可寫入。

## 重點整理

- 子圖能讓不同單位與範圍的資料各自保持可讀性。
- `DateFormatter` 控制顯示文字，不會改變原始時間資料。
- `tight_layout()` 與檢查輸出 PNG 都是避免版面問題的必要步驟。

## 下一步

回到[將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)對照完整資料流。本資料處理延伸系列暫告一段落。
