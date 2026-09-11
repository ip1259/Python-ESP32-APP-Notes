# 延伸選讀：進階 logger 技巧與設計方法

當程式變大時，讓 logger 有清楚名稱與分工，才能在保留線索的同時避免重複輸出和失控的 log 檔。

## 你會學到什麼

- 使用 `logging.getLogger(__name__)` 建立可辨識來源的 logger。
- 將終端機與檔案 Handler 設為不同紀錄門檻。
- 避免重複加入 Handler 與重複輸出同一筆紀錄。
- 使用 logger 的格式參數延後格式化訊息，並理解輪替檔案的限制。

## 開始前

- 先閱讀[延伸選讀：logging 與除錯紀錄](附錄-logging與除錯紀錄.md)，了解 `INFO`、`WARNING`、`ERROR` 與受控練習 log。
- 本頁是延伸選讀，不是主線、pyserial 或整合專題的前置條件。
- 範例只使用 Python 標準函式庫，不連接 ESP32、COM 埠或網路。
- 範例會建立 `outputs/advanced_logging.log` 及備份檔；請只在練習資料夾使用，不能直接套用到要長期保存的專案 log。

## 成功的樣子

終端機會顯示 `INFO` 與 `WARNING`，但不顯示 `DEBUG`：

```text
INFO | sensor_reader | 開始讀取練習資料
WARNING | sensor_reader | 溫度接近上限：79.5°C
```

開啟 `outputs/advanced_logging.log` 時，另外能看到一筆 `DEBUG`。這表示不同 Handler 可以各自決定要保留多少細節。

## 讓程式入口統一設定 logger

目的：由應用程式入口決定 log 寫去哪裡；讀取感測資料的函式只負責記錄自己的事件。

操作：建立 `advanced_logging_demo.py`，貼上並執行下列程式。

```python
# 執行位置：Python／電腦
# 檔名：advanced_logging_demo.py
import logging
from logging.handlers import RotatingFileHandler
from pathlib import Path


def configure_logger() -> logging.Logger:
    logger: logging.Logger = logging.getLogger("sensor_reader")
    logger.setLevel(logging.DEBUG)
    logger.propagate = False

    if logger.handlers:
        return logger

    output_folder: Path = Path("outputs")
    output_folder.mkdir(exist_ok=True)
    log_path: Path = output_folder / "advanced_logging.log"
    formatter: logging.Formatter = logging.Formatter(
        "%(levelname)s | %(name)s | %(message)s"
    )

    console_handler: logging.StreamHandler = logging.StreamHandler()
    console_handler.setLevel(logging.INFO)
    console_handler.setFormatter(formatter)

    file_handler: RotatingFileHandler = RotatingFileHandler(
        log_path,
        maxBytes=1024,
        backupCount=2,
        encoding="utf-8",
    )
    file_handler.setLevel(logging.DEBUG)
    file_handler.setFormatter(formatter)

    logger.addHandler(console_handler)
    logger.addHandler(file_handler)
    return logger


def read_practice_temperature(logger: logging.Logger, raw_value: str) -> float:
    temperature_c: float = float(raw_value)
    logger.debug("原始溫度文字：%s", raw_value)
    logger.info("開始讀取練習資料")
    if temperature_c >= 75.0:
        logger.warning("溫度接近上限：%.1f°C", temperature_c)
    return temperature_c


app_logger: logging.Logger = configure_logger()
read_practice_temperature(app_logger, "79.5")
configure_logger()
print(f"Handler 數量：{len(app_logger.handlers)}")
```

預期結果：終端機出現一筆 `INFO`、一筆 `WARNING`，最後顯示 `Handler 數量：2`；`DEBUG` 只在 `outputs/advanced_logging.log`。最後再次呼叫 `configure_logger()` 後，數量仍是 `2`，表示沒有重複加入 Handler。

## 看懂這個設計的責任分工

| 元件 | 責任 | 為什麼這樣安排 |
| --- | --- | --- |
| `configure_logger()` | 決定名稱、等級、格式與輸出位置 | 入口集中設定，其他函式不會互相覆蓋設定。 |
| `read_practice_temperature()` | 處理讀值並記錄事件 | 不自行建立 Handler，也不決定 log 檔路徑。 |
| 終端機 Handler | 顯示 `INFO` 以上 | 操作者看見重要流程，不被除錯細節淹沒。 |
| 檔案 Handler | 保存 `DEBUG` 以上 | 排錯時可回看較完整線索。 |

`logging.getLogger("sensor_reader")` 會依名稱取得同一個 logger。真實專案的模組通常寫成 `logging.getLogger(__name__)`，讓名稱隨模組改變；這個單檔練習使用固定名稱，方便辨識輸出。

## 避免重複輸出

目的：理解同一筆紀錄為何有時會印兩次，以及本範例如何避免。

操作：觀察範例中的兩段設定：

```python
# 執行位置：Python／電腦
# 檔名：advanced_logging_demo.py
if logger.handlers:
    return logger

logger.propagate = False
```

預期結果：重複執行設定函式不會持續增加 Handler。`logger.handlers` 防止此 logger 重複加入自己的輸出；`propagate = False` 則讓已經有 Handler 的練習 logger 不再把同一事件交給 root logger 輸出一次。

這不是永遠固定的規則。若正式專案由 root logger 統一配置 Handler，模組 logger 就不應自行加入 Handler，也通常不設 `propagate = False`。先確認「誰負責設定」再選擇一種方式，不能兩種混用。

## 延後格式化與輪替檔案

目的：讓不需要顯示的 log 少做格式化工作，並限制受控練習 log 無限長大。

操作：查看這行 logger 呼叫和 `RotatingFileHandler` 的設定：

```python
# 執行位置：Python／電腦
# 檔名：advanced_logging_demo.py
logger.warning("溫度接近上限：%.1f°C", temperature_c)

file_handler = RotatingFileHandler(
    log_path,
    maxBytes=1024,
    backupCount=2,
    encoding="utf-8",
)
```

預期結果：logger 只有在該等級需要輸出時才將 `temperature_c` 填入訊息。檔案超過 `1024` 位元組時會輪替，最多留下目前檔與兩個備份。輪替會淘汰最舊紀錄，所以只適合可重建的練習 log；重要的專案紀錄必須先訂定保存、備份與權限規則，不能直接照抄。

## 選做練習：比較終端機與檔案內容

這是延伸練習，不做也不影響主線成果。

目的：確認 `DEBUG` 不會顯示在終端機，但會寫入檔案。

操作：執行範例後，在終端機和 `outputs/advanced_logging.log` 各搜尋 `原始溫度文字`。

預期結果：終端機找不到這句，檔案能找到 `DEBUG | sensor_reader | 原始溫度文字：79.5`。

??? tip "參考作法"
    請使用文字編輯器開啟 log 檔，不要改動範例的 `console_handler.setLevel(logging.INFO)`。這個差異正是兩個 Handler 使用不同門檻的結果。

## 完成時，應能確認

- 你知道由應用程式入口集中設定 Handler，讀值函式只記錄事件。
- 你能說明 `logger.handlers` 和 `propagate` 與重複輸出的關係。
- 你能在檔案找到 `DEBUG`，並理解為何終端機只顯示 `INFO` 以上。
- 你知道輪替 log 會淘汰舊檔，只能先用於可重建的練習範圍。

## 常見問題

### 同一筆訊息出現兩次

先檢查是否重複呼叫設定函式並持續 `addHandler()`，或 logger 與 root logger 都有 Handler。請先決定由哪一層負責輸出；單檔範例使用 `if logger.handlers` 與 `propagate = False` 讓結果固定。

### `DEBUG` 沒有出現在終端機

這是範例預期行為，因為終端機 Handler 設為 `INFO`。請改到 log 檔查看；若所有 Handler 都不顯示 `DEBUG`，再檢查 logger 本身是否設為 `logging.DEBUG`。

### 可以直接把完整 log 傳給別人嗎？

不可以。先檢查並移除密碼、token、個人資料、私有 IP、內網位址、裝置名稱與完整原始序列資料；有不確定內容時停止分享，只提供遮罩後、解決問題必要的最小片段。

## 重點整理

- 命名 logger 與集中設定 Handler，可讓模組責任清楚。
- 不同 Handler 可有不同等級；終端機與檔案不必收到同樣細節。
- `logger.handlers` 與 `propagate` 可處理重複輸出，但需先決定設定責任在哪一層。
- 延後格式化提升可讀性與效率；輪替檔案會淘汰舊紀錄，必須限制在受控範圍。

## 下一步

回到[延伸選讀：logging 與除錯紀錄](附錄-logging與除錯紀錄.md)，依你的程式規模選擇基礎或進階設計。主線順序接著可閱讀「即將推出：pyserial 連線、讀寫與逾時」。
