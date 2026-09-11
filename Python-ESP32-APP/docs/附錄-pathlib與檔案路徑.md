# 延伸選讀：pathlib 與檔案路徑

使用 `pathlib` 管理 CSV 和圖表的輸出位置，讓程式不必依賴特定電腦的資料夾寫法。

## 你會學到什麼

- 使用 `Path` 表示資料夾與檔案路徑。
- 用 `/` 組合資料夾與檔名。
- 建立輸出資料夾，並確認 CSV 檔是否存在。
- 從路徑取得檔名、副檔名、上層資料夾與目前工作位置。
- 安全地讀寫文字檔、搜尋練習資料夾內的檔案與重新命名檔案。

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

## 常用路徑資訊

目的：從一個路徑拆出檔名、檔名主體、副檔名與上層資料夾，這些用法適用於任何 Python 專案。

```python
# 執行位置：Python／電腦
# 檔名：path_info.py
from pathlib import Path

report = Path("reports") / "week1.txt"

print(f"完整路徑：{report}")
print(f"檔名：{report.name}")
print(f"不含副檔名：{report.stem}")
print(f"副檔名：{report.suffix}")
print(f"上層資料夾：{report.parent}")
print(f"目前工作資料夾：{Path.cwd()}")
```

預期結果：依序可看到 `week1.txt`、`week1`、`.txt`、`reports`，以及你執行程式時的實際資料夾。`Path.cwd()` 很適合在找不到相對路徑時確認程式從哪裡開始計算。

### 相對路徑與絕對路徑

`Path("reports") / "week1.txt"` 是相對路徑，會跟著目前工作資料夾改變；`resolve()` 可將存在的路徑顯示為目前電腦上的完整位置。

```python
# 執行位置：Python／電腦
# 檔名：path_info.py
from pathlib import Path

file_path = Path("outputs") / "sensor_data.csv"
print(file_path.resolve())
```

預期結果：顯示以磁碟機開頭的完整位置。這適合除錯或確認位置；平常專案程式仍優先保存相對路徑，不要把自己的使用者名稱或固定磁碟位置寫死在程式中。

## 讀寫、搜尋與重新命名文字檔

### 1. 建立並讀回 UTF-8 文字檔

目的：了解 `Path` 也能直接讀寫一般文字檔，不只用來組合路徑。

```python
# 執行位置：Python／電腦
# 檔名：text_file_demo.py
from pathlib import Path

notes_folder = Path("notes")
notes_folder.mkdir(exist_ok=True)
note_path = notes_folder / "hello.txt"

note_path.write_text("第一行\n第二行", encoding="utf-8")
print(note_path.read_text(encoding="utf-8"))
```

預期結果：建立 `notes/hello.txt` 並顯示兩行文字。`write_text()` 會覆寫同名檔案，僅應用於你確認可覆寫的練習檔。

### 2. 只搜尋練習資料夾

目的：找出特定副檔名的檔案，而不掃描整台電腦。

```python
# 執行位置：Python／電腦
# 檔名：text_file_demo.py
from pathlib import Path

for file_path in Path("notes").glob("*.txt"):
    print(file_path.name)
```

預期結果：顯示 `notes` 資料夾中的 `.txt` 檔名。`glob("*.txt")` 只搜尋這一層；本頁不使用遞迴搜尋或整個磁碟的萬用字元。

### 3. 重新命名練習檔

目的：將既有檔案移到同一資料夾的新名稱。

```python
# 執行位置：Python／電腦
# 檔名：text_file_demo.py
from pathlib import Path

old_path = Path("notes") / "hello.txt"
new_path = old_path.replace(Path("notes") / "welcome.txt")
print(new_path.name)
```

預期結果：顯示 `welcome.txt`，檔案總管中的檔名也會改變。只對你剛建立的練習檔操作；若新名稱已存在，可能覆寫或因系統行為失敗，請先確認 `new_path.exists()`。

## 高風險操作：刪除練習檔與空資料夾

`unlink()` 會刪除檔案，`rmdir()` 會移除空資料夾。它們是常用操作，但刪除後不會移到資源回收筒，因此只在下列專屬練習資料夾操作。

!!! warning "先確認目標；不符合就停止"
    只執行範例剛建立的 `practice_cleanup/delete_me.txt`。如果檔案不是練習檔、資料夾位置被你改過、目標不是檔案，或資料夾內還有其他檔案，請停止，不要刪除。不要把 `unlink()` 或 `rmdir()` 套用到作業、感測資料、專案根目錄或不確定的路徑。

### 建議優先使用：管理程式暫存檔

目的：處理程式在執行期間才需要的中間檔，例如轉換前的 CSV、下載前預覽或短暫計算結果。讓 Python 在工作完成後清理專屬暫存資料夾，比手動刪除檔案更容易限制範圍。

```python
# 執行位置：Python／電腦
# 檔名：temporary_file_demo.py
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory() as temp_dir:
    temp_folder = Path(temp_dir)
    temp_csv = temp_folder / "preview.csv"
    temp_csv.write_text("name,value\nAda,10\n", encoding="utf-8")

    print(f"暫存檔存在：{temp_csv.is_file()}")
    print(temp_csv.read_text(encoding="utf-8"))

print(f"暫存資料夾仍存在：{temp_folder.exists()}")
```

預期結果：區塊內顯示 `暫存檔存在：True` 和 CSV 文字；離開 `with` 區塊後顯示 `暫存資料夾仍存在：False`。`TemporaryDirectory` 只適合可重新產生的暫存資料，不能用來保存作業、感測紀錄或任何要交付的成果。

!!! tip "何時選擇哪一種方式？"
    程式自己建立、使用完就不需要的中間檔，優先使用 `TemporaryDirectory`。不確定是否還要保留的檔案，先以 `replace()` 改名封存。只有已確認的受控練習檔才使用 `unlink()`；不要用它清理未知來源的檔案。

### 受控練習：刪除剛建立的檔案

目的：先檢查路徑與類型，再刪除同一支程式剛建立的練習檔。

```python
# 執行位置：Python／電腦
# 檔名：safe_cleanup_demo.py
from pathlib import Path

practice_folder = Path("practice_cleanup").resolve()
practice_folder.mkdir(exist_ok=True)
target = practice_folder / "delete_me.txt"
target.write_text("這是可刪除的練習檔", encoding="utf-8")

if target.parent != practice_folder or not target.is_file():
    raise RuntimeError("目標不是指定的練習檔，已停止刪除。")

print(f"準備刪除：{target}")
target.unlink()
print(f"檔案仍存在：{target.exists()}")
```

預期結果：先顯示完整的練習檔位置，再顯示 `檔案仍存在：False`。若檢查沒有通過，程式會停止而不刪除。

### 移除已確認為空的練習資料夾

目的：只在資料夾沒有任何內容時移除它。

```python
# 執行位置：Python／電腦
# 檔名：safe_cleanup_demo.py
from pathlib import Path

practice_folder = Path("practice_cleanup").resolve()

if practice_folder.exists() and not any(practice_folder.iterdir()):
    practice_folder.rmdir()
    print(f"資料夾仍存在：{practice_folder.exists()}")
else:
    print("資料夾不是空的或不存在，已停止移除。")
```

預期結果：接續上一段成功刪除檔案後，顯示 `資料夾仍存在：False`。若資料夾還有任何檔案，程式只會顯示停止訊息；`rmdir()` 不會遞迴刪除內容。

### 較安全的替代方案：改名封存

不確定是否還需要檔案時，不要刪除；可先以 `replace()` 改名封存，例如將 `report.txt` 改為 `report_backup.txt`，確認不再需要後才依自己的備份規則處理。需要臨時檔案時，優先使用 Python 的 `TemporaryDirectory`，程式結束後會自動清理受控暫存位置。

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
- 你能用 `name`、`stem`、`suffix` 與 `parent` 讀取路徑資訊。
- 你知道 `write_text()` 會覆寫檔案，並只在自己的練習檔使用 `glob()` 和 `replace()`。
- 你能說出 `unlink()` 只刪除已確認的練習檔，`rmdir()` 只移除空資料夾；不確定時會先改名封存。

## 常見問題

### 執行後找不到 `outputs` 資料夾

先確認你是從哪個資料夾執行 `python path_demo.py`。相對路徑會從目前終端機所在的資料夾開始計算；請在該資料夾或其專案資料夾中尋找 `outputs`。

### 顯示 `PermissionError` 或沒有寫入權限

先確認你不是在系統資料夾或唯讀位置執行程式。將 `path_demo.py` 放到自己的專案資料夾後重新執行，不要用系統管理員權限繞過限制。

### `CSV 是檔案：False`

先確認程式中的 `csv_path.touch(exist_ok=True)` 沒有被刪除，且前面的 `output_folder.mkdir(...)` 已先執行。若你把同名位置手動建立成資料夾，請改用其他檔名，避免檔案與資料夾同名。

### `write_text()` 後原本內容不見了

`write_text()` 的用途是寫入完整的新文字，會覆寫同名檔。請不要用它直接修改唯一的作業或感測資料；先複製一份，或改用新的練習檔名。

### `rmdir()` 顯示資料夾不是空的

這是保護機制，不要改成遞迴刪除。先查看練習資料夾中是否有你要保留的檔案；不確定時保留資料夾，或把要保留的檔案改名封存。

## 重點整理

- `Path` 讓路徑成為清楚的 Python 物件。
- 使用 `/` 組合子資料夾與檔名，不必手動處理 Windows 的 `\`。
- 寫檔前先用 `mkdir(parents=True, exist_ok=True)` 準備輸出資料夾，再用 `exists()` 或 `is_file()` 檢查結果。
- `name`、`stem`、`suffix`、`parent` 與 `cwd()` 能協助你理解和除錯路徑。
- `read_text()`、`write_text()`、`glob()` 與 `replace()` 是一般文字檔專案也常用的工具；寫入與重新命名前先確認目標。
- 程式產生且可重新建立的中間檔，優先放入 `TemporaryDirectory`；`unlink()` 與 `rmdir()` 只限於受控練習目標，資料不確定時以封存或暫存資料夾替代。

## 下一步

回到[將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)，把已驗證的 CSV 與圖表流程和輸出路徑概念一起使用。接下來可閱讀[附錄-json與csv](附錄-json與csv.md)，了解資料格式如何在 Python 中轉換。
