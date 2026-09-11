# 延伸選讀：datetime 與 time

用正確的時間型別記錄資料、顯示時間，並以穩定的經過時間判斷讀取間隔與逾時。

## 你會學到什麼

- 使用 `datetime` 建立、格式化與解析時間戳記。
- 計算兩個時間點相差多久。
- 用 `time.monotonic()` 的概念判斷程式間隔與逾時。

## 開始前

- 建議先閱讀[將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)，了解 `timestamp` 在感測資料中的用途。
- `datetime` 和 `time` 是 Python 標準函式庫，不需安裝套件，也不需連接 ESP32。
- 時間字串格式必須和解析格式完全一致；不確定格式時先停止並檢查原始資料。

## 成功的樣子

執行本頁範例後，會看到固定時間 `2026-09-11 14:30:05`，以及可判讀的結果：

```text
時間往返相同：True
經過秒數：2.5
尚未逾時：False
已經逾時：True
```

## 建立與格式化時間

### 1. 建立可讀的時間戳記

目的：將 `datetime` 物件格式化成適合顯示或寫入 CSV 的文字。

```python
# 執行位置：Python／電腦
# 檔名：datetime_demo.py
from datetime import datetime

recorded_at: datetime = datetime(2026, 9, 11, 14, 30, 5)
formatted: str = recorded_at.strftime("%Y-%m-%d %H:%M:%S")

print(formatted)
```

預期結果：顯示 `2026-09-11 14:30:05`。`%Y` 是四位數年份、`%m` 是月份、`%d` 是日期；`%H:%M:%S` 是時、分、秒。

### 常用時間格式碼速查

以下範例都以 `datetime(2026, 9, 11, 14, 30, 5)` 為基準。將格式碼放入 `strftime()` 可產生文字；用 `strptime()` 解析時，文字的順序和分隔符號也必須相同。

| 格式碼 | 意義 | 輸出例 |
| --- | --- | --- |
| `%Y` | 四位數年份 | `2026` |
| `%y` | 兩位數年份 | `26` |
| `%m` | 兩位數月份 | `09` |
| `%d` | 兩位數日期 | `11` |
| `%H` | 24 小時制的小時 | `14` |
| `%I` | 12 小時制的小時 | `02` |
| `%M` | 分鐘 | `30` |
| `%S` | 秒數 | `05` |
| `%p` | AM／PM 標記 | 依電腦語系顯示 |

常見組合如下：

| 用途 | 格式字串 | 輸出例 |
| --- | --- | --- |
| CSV 常用時間戳記 | `%Y-%m-%d %H:%M:%S` | `2026-09-11 14:30:05` |
| 檔名日期 | `%Y%m%d` | `20260911` |
| 只顯示時間 | `%H:%M` | `14:30` |
| 12 小時制顯示 | `%I:%M %p` | 依語系顯示 `02:30 PM` 或相近文字 |

!!! warning "`%m` 和 `%M` 不一樣"
    小寫 `%m` 是月份，大寫 `%M` 是分鐘。若把它們互換，程式不一定報錯，卻會得到錯誤時間；寫入 CSV 或解析資料前，先用固定時間執行一次確認輸出。

### 2. 將文字解析回時間

目的：把 CSV 或設定檔中的時間文字轉回可計算的 `datetime`。

```python
# 執行位置：Python／電腦
# 檔名：datetime_demo.py
from datetime import datetime

time_text: str = "2026-09-11 14:30:05"
parsed_at: datetime = datetime.strptime(time_text, "%Y-%m-%d %H:%M:%S")

print(parsed_at == datetime(2026, 9, 11, 14, 30, 5))
```

預期結果：顯示 `True`。`strptime()` 的格式碼必須和文字完全一致；例如文字沒有秒數時，格式中也不能保留 `%S`。

## 計算兩個時間點相差多久

目的：計算一筆資料距離另一筆資料的時間差。

```python
# 執行位置：Python／電腦
# 檔名：datetime_demo.py
from datetime import datetime

first_at: datetime = datetime(2026, 9, 11, 14, 30, 0)
second_at: datetime = datetime(2026, 9, 11, 14, 30, 8)
elapsed_seconds: float = (second_at - first_at).total_seconds()

print(elapsed_seconds)
```

預期結果：顯示 `8.0`。這適合比較已記錄的資料時間；若要量測程式「從現在開始等了多久」，請使用下一節的 `monotonic()`。

## 讀取間隔與逾時

系統時間可能因校時而跳動，`time.monotonic()` 則只會向前增加，適合量測程式等待多久。下列函式以固定數值模擬其回傳值，因此不需真的等待。

```python
# 執行位置：Python／電腦
# 檔名：timeout_demo.py

def elapsed_seconds(start: float, now: float) -> float:
    return now - start


def has_timed_out(start: float, now: float, timeout: float) -> bool:
    return elapsed_seconds(start, now) >= timeout


start_time: float = 10.0
current_time: float = 12.5
timeout_seconds: float = 3.0

print(f"經過秒數：{elapsed_seconds(start_time, current_time)}")
print(f"尚未逾時：{has_timed_out(start_time, current_time, timeout_seconds)}")
print(f"已經逾時：{has_timed_out(start_time, 13.0, timeout_seconds)}")
```

預期結果：依序顯示 `2.5`、`False`、`True`。正式程式可用 `start_time = time.monotonic()` 取得開始時間，再以新的 `time.monotonic()` 當作 `now`；不要用 `datetime.now()` 判斷逾時。

!!! warning "不要用長時間 sleep 模擬讀取間隔"
    `time.sleep()` 會阻塞目前程式，可能讓介面或其他工作停止回應。本頁使用固定數值驗證時間判斷；需要定期讀取時，先確認程式是否可以被阻塞，再設計間隔與停止條件。

## 選做練習：顯示距離現在幾秒

目的：計算一筆已知資料時間到目前時間的秒數。

操作：以 `datetime.now()` 取代上一節的 `second_at`，計算 `now - first_at` 的 `total_seconds()`。

預期結果：顯示一個浮點數；數值會隨執行時間改變。這是延伸練習，不影響主線成果。

## 完成時，應能確認

- 你能用 `strftime()` 將時間轉為指定格式的文字。
- 你能用 `strptime()` 將符合格式的文字解析為 `datetime`。
- 你能用 `total_seconds()` 判讀兩筆已記錄資料相差的秒數。
- 你知道量測程式間隔與逾時時，應優先使用 `time.monotonic()`。

## 常見問題

### `ValueError: time data ... does not match format`

先逐一比對時間文字和 `strptime()` 格式碼：是否少了秒數、分隔符號不同，或月份和日期位置相反。不要直接改成猜測格式。

### 逾時結果偶爾不合理

先確認程式是否以 `time.monotonic()` 量測開始和現在的時間。系統時間可能被校正，不能作為等待時間的可靠基準。

## 重點整理

- `datetime` 適合表示、格式化、解析與比較日期時間。
- `timedelta.total_seconds()` 可將兩個時間點的差轉成秒數。
- `time.monotonic()` 適合量測程式間隔與逾時；Type Hint 能協助閱讀函式輸入與回傳值，但不會自動檢查資料正確性。

## 下一步

回到[將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)，查看時間戳記如何影響資料排序與圖表。下一篇可處理「即將推出：函式、類別與責任分工」。
