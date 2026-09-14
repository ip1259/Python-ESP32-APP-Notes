# 延伸選讀：Matplotlib 圖表基本元件

用一張可輸出的雙折線圖，認識標題、座標軸、圖例與格線如何幫助讀者看懂資料。

## 你會學到什麼

- 使用 `plt.subplots()` 建立整張圖與繪圖區。
- 使用 `plot()` 畫出有名稱的資料線。
- 使用標題、座標軸標籤、圖例與格線讓圖表可判讀。
- 使用 `savefig()` 輸出 PNG，並用 `close()` 關閉圖表。

## 開始前

- 建議先完成[將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)。
- 本頁使用固定練習資料，不連接 ESP32，也不讀取或覆寫主線 CSV。
- 需要 `matplotlib` 套件與可執行 uv 的 Python 練習環境。
- 範例會重建 `outputs/matplotlib_basics.png`；請只在可控制的練習資料夾中執行。

## 成功的樣子

程式會建立 `outputs/matplotlib_basics.png`。圖中有 `Temperature` 與 `Humidity` 兩條折線、英文標題、帶單位的縱軸、時間橫軸、圖例與虛線格線。

## 核心功能速覽

### `plt.subplots()`：建立圖表與繪圖區

`plt.subplots(figsize=(8, 4))` 回傳 `fig` 與 `ax`。`fig` 是整張 Figure，`ax` 是實際繪製資料線、設定標題與座標軸的 Axes。

### `ax.plot()`：畫出資料線

`ax.plot(x_values, y_values, marker="o", label="Temperature")` 傳入橫軸、縱軸與線條名稱，回傳線條物件。`marker="o"` 顯示每筆資料點；`label` 需搭配 `ax.legend()` 才會顯示。

### 標題、座標軸、圖例與格線

- `ax.set_title("...")` 設定圖表標題。
- `ax.set_xlabel("Time")`、`ax.set_ylabel("Value (°C / %)")` 設定軸名稱與單位。
- `ax.legend()` 顯示各線條的 `label`，不應只依賴顏色。
- `ax.grid(True, linestyle="--", alpha=0.5)` 顯示輔助格線，不能深到遮住資料。

### `savefig()` 與 `close()`：輸出與關閉

`fig.savefig(chart_path, dpi=150)` 將 Figure 寫成 PNG，回傳值不重要，重點是輸出的檔案。`plt.close(fig)` 關閉指定圖表；重複產圖時可避免圖表累積在記憶體。

## 建立一張雙折線圖

目的：從固定資料確認每一個基本元件都出現在 PNG。

操作：建立 `matplotlib_basics_demo.py`，貼上並執行。

```python
# 執行位置：Python／電腦
# 檔名：matplotlib_basics_demo.py
from datetime import datetime
from pathlib import Path

import matplotlib

matplotlib.use("Agg")

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
chart_path: Path = output_folder / "matplotlib_basics.png"

fig, ax = plt.subplots(figsize=(8, 4))
ax.plot(timestamps, temperatures_c, marker="o", label="Temperature")
ax.plot(timestamps, humidities_pct, marker="o", label="Humidity")
ax.set_title("Temperature and Humidity")
ax.set_xlabel("Time")
ax.set_ylabel("Value (°C / %)")
ax.legend()
ax.grid(True, linestyle="--", alpha=0.5)
fig.autofmt_xdate()
fig.tight_layout()
fig.savefig(chart_path, dpi=150)
plt.close(fig)
print(f"圖表已建立：{chart_path}")
```

預期結果：顯示 `圖表已建立：outputs\\matplotlib_basics.png`。開啟 PNG 後可看見兩條含圓點的線、標題、`Time`、`Value (°C / %)`、圖例及格線。這是固定練習資料，不可用來推論真實環境。

## 選做練習：替溫度線指定顏色

這是延伸練習，不做也不影響主線成果。

目的：練習在 `plot()` 加入一個可讀性設定。

操作：只在溫度的 `ax.plot(...)` 加入 `color="tab:red"`，重新輸出 PNG。

預期結果：溫度線變為紅色，圖例仍能正確辨識兩條線。不要只依賴顏色；`label` 與 `legend()` 仍然必要。

## 完成時，應能確認

- 你知道 `fig` 是整張圖、`ax` 是繪圖區。
- 你能用 `plot()`、`label` 與 `legend()` 建立可辨識的資料線。
- 你能替圖表加上標題、座標軸、單位與格線。
- 你能輸出 PNG，且知道 `close()` 的用途。

## 常見問題

### `ModuleNotFoundError: No module named 'matplotlib'`

確認在同一個 uv 環境安裝並執行 `matplotlib`。

### 找不到輸出的 PNG

先看 PowerShell 顯示的 `chart_path`。相對路徑以執行指令的資料夾為準；確認目前資料夾可寫入，且 `outputs` 沒有被移動或改名。

### 圖例沒有出現

確認每個 `ax.plot()` 都有 `label="..."`，並在所有線條畫完後呼叫 `ax.legend()`。

### 圖表文字或日期重疊

先保留 `fig.autofmt_xdate()` 與 `fig.tight_layout()`。下一篇會處理多子圖、日期格式與版面調整。

## 重點整理

- `plot()` 畫資料線，`label` 與 `legend()` 說明資料代表什麼。
- 標題、軸標籤、單位與格線讓圖表可被正確判讀。
- `savefig()` 輸出可驗收的 PNG，`close()` 適合重複產圖。

## 下一步

回到[將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)對照主線的雙子圖做法。接著可閱讀[延伸選讀：Matplotlib 可讀性與輸出](附錄-Matplotlib可讀性與輸出.md)，練習以雙子圖、日期格式與版面調整提升可讀性。
