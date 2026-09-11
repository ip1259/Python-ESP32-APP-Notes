# 延伸選讀：logging 與除錯紀錄

將讀取流程留下帶有時間與等級的紀錄，讓你在問題發生後仍能回看當時的線索。

## 你會學到什麼

- 使用 `INFO`、`WARNING`、`ERROR` 記錄不同嚴重程度的事件。
- 將紀錄同時顯示在終端機並寫入可控制的練習檔。
- 將無法轉成數字的輸入記為警告，避免當成可信感測資料使用。
- 在分享紀錄前辨識密碼、token、個人資料與私有網路資訊的風險。

## 開始前

- 先完成[用 uv 建立 Python 虛擬環境](01-uv與虛擬環境.md)，並能在練習資料夾執行 `uv run python`。
- 建議先閱讀[延伸選讀：例外處理與錯誤訊息](附錄-例外處理與錯誤訊息.md)。例外處理決定流程如何安全停止或繼續；logging 則保留發生過什麼事的紀錄。
- 本頁只使用 Python 標準函式庫與固定文字，不連接 ESP32、COM 埠或網路。
- 範例會重建 `outputs/logging_demo.log`。請只在自己的練習資料夾使用，不能套用到要保存的專案 log。

## 成功的樣子

執行後，終端機和 `outputs/logging_demo.log` 都會有類似三行內容：

```text
INFO | 開始處理練習資料
WARNING | 無法將溫度轉成數字：unknown
ERROR | 本次資料沒有可用的溫度，略過寫入。
```

每行前方還會有執行時間。三個等級表示的不是「程式一定壞掉」：`INFO` 是正常歷程，`WARNING` 是可安全繼續但需注意，`ERROR` 是本次功能無法完成。

## 先建立受控的 log 檔

目的：同時在終端機和檔案留下可回看的紀錄。

操作：建立 `logging_demo.py`，貼上並執行下列程式。

```python
# 執行位置：Python／電腦
# 檔名：logging_demo.py
import logging
from pathlib import Path


output_folder: Path = Path("outputs")
output_folder.mkdir(exist_ok=True)
log_path: Path = output_folder / "logging_demo.log"

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(message)s",
    handlers=[
        logging.StreamHandler(),
        logging.FileHandler(log_path, mode="w", encoding="utf-8"),
    ],
    force=True,
)

logging.info("開始處理練習資料")
print(f"已建立練習紀錄：{log_path}")
```

預期結果：終端機顯示一行 `INFO` 紀錄與 `已建立練習紀錄：outputs\\logging_demo.log`，並建立該檔案。`mode="w"` 每次都重建同名練習 log，讓結果固定；它會覆寫舊檔，因此不能用於你要保留的實際專案紀錄。

## 為資料問題選擇正確等級

目的：把可略過的單筆格式問題記成 `WARNING`，而不是把它當成有效讀值。

操作：將下列完整程式取代 `logging_demo.py` 後執行。

```python
# 執行位置：Python／電腦
# 檔名：logging_demo.py
import logging
from pathlib import Path


def parse_temperature(raw_value: str) -> float | None:
    try:
        return float(raw_value)
    except ValueError:
        logging.warning("無法將溫度轉成數字：%s", raw_value)
        return None


output_folder: Path = Path("outputs")
output_folder.mkdir(exist_ok=True)
log_path: Path = output_folder / "logging_demo.log"
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(message)s",
    handlers=[
        logging.StreamHandler(),
        logging.FileHandler(log_path, mode="w", encoding="utf-8"),
    ],
    force=True,
)

logging.info("開始處理練習資料")
temperature_c: float | None = parse_temperature("unknown")

if temperature_c is None:
    logging.error("本次資料沒有可用的溫度，略過寫入。")
else:
    logging.info("可寫入溫度：%.1f°C", temperature_c)

print(f"已寫入練習紀錄：{log_path}")
```

預期結果：終端機依序顯示 `INFO`、`WARNING`、`ERROR` 三筆紀錄與 log 檔位置；開啟該檔案時可看到相同三筆紀錄。函式只負責轉型並回傳結果；呼叫端看到 `None` 才決定略過寫入，避免把 `unknown` 寫進感測 CSV。

## 分享前先檢查紀錄內容

目的：理解 log 能協助除錯，但也可能洩漏不該分享的資訊。

操作：開啟 `outputs/logging_demo.log`，確認它只包含練習訊息與時間。

預期結果：找不到 Wi-Fi 密碼、token、個人資料、固定私有 IP、內網位址或完整未篩選的 UART 資料。若你在真實專案 log 發現這些內容，停止分享該檔案，先移除或遮罩敏感值，只保留排錯必要的最小片段。

## 選做練習：記錄成功讀值

這是延伸練習，不做也不影響主線成果。

目的：確認合法數值會走到 `INFO`，而不是 `WARNING` 或 `ERROR`。

操作：將 `parse_temperature("unknown")` 改為 `parse_temperature("26.5")`，重新執行程式。

預期結果：log 有兩筆 `INFO`，第二筆包含 `可寫入溫度：26.5°C`，不再有 `WARNING` 或 `ERROR`。

??? tip "參考作法"
    只需修改這一行：

    ```python
    # 執行位置：Python／電腦
    # 檔名：logging_demo.py
    temperature_c: float | None = parse_temperature("26.5")
    ```

## 完成時，應能確認

- 你能說明 `INFO`、`WARNING`、`ERROR` 分別適合記錄什麼。
- 你能找到 `outputs/logging_demo.log`，並確認它有時間、等級與訊息。
- 你能讓無法轉成數字的資料回傳 `None`，而不是當成可信溫度寫入。
- 你知道分享 log 前要先移除敏感資訊，且不會用練習的覆寫模式處理要保存的 log。

## 常見問題

### log 檔只剩最新一次內容

這是範例的 `mode="w"` 預期行為，用來讓受控練習結果固定。若是實際專案要保留歷程，先確認檔名、保存期限與敏感資料處理規則，再考慮改用附加模式；不要直接修改重要 log。

### 我只看到 `INFO`，沒有 `WARNING` 或 `ERROR`

先確認輸入仍是 `"unknown"`。改成 `"26.5"` 時，轉型成功，只會記錄正常流程的 `INFO`。

### 可以把完整 log 貼到群組或公開網站嗎？

不可以直接貼。先檢查是否包含密碼、token、個人資料、私有位址、裝置名稱或原始感測資料；有任何敏感資訊就停止分享並建立遮罩後的最小片段。

## 重點整理

- logging 用時間、等級與訊息留下流程證據；它不能取代例外處理。
- `WARNING` 表示可安全略過但需注意，`ERROR` 表示本次功能無法完成。
- 練習檔可用覆寫模式重建；真實專案 log 必須先確認保存與隱私規則。

## 進階技巧
若要處理多個輸出位置與重複訊息，可選讀[延伸選讀：進階 logger 技巧與設計方法](附錄-進階logger技巧與設計方法.md)

接著，你可以回到[延伸選讀：例外處理與錯誤訊息](附錄-例外處理與錯誤訊息.md)，比較「控制流程」與「留下紀錄」的責任。

## 下一步

「即將推出：pyserial 連線、讀寫與逾時」: 
