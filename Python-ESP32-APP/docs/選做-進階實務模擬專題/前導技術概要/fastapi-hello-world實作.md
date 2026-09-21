# 選做實作：FastAPI Hello World

這個練習會在電腦上建立一個最小 API，並用同一台電腦的瀏覽器讀取固定的 Hello World 回應。

## 你會學到什麼

- 建立一個不混入其他課程檔案的 Python 練習專案。
- 用 FastAPI 建立一個 `GET /` 路徑。
- 從瀏覽器觀察 Python 字典轉成的 JSON 回應。
- 確認本機 API 啟動與停止時的差別。

## 開始前

- 先閱讀[FastAPI：Gateway API 的基本概念](fastapi概念.md)。
- 電腦需要已安裝 Python 與 `uv`，第一次安裝 FastAPI 套件時需要網路。
- 請在一般練習資料夾建立本頁專案，不要在另一個已有 `pyproject.toml` 的 `uv` 專案裡建立。
- 本頁只使用 `127.0.0.1`。它代表目前這台電腦，不會建立 ngrok 通道，也不會讓手機或其他電腦連入。

本頁不需要 ESP32、RC522、手機、雲端帳號或其他前導實作。

## 成功的樣子

完成後，瀏覽器開啟 `http://127.0.0.1:8000/` 會看到：

用途：瀏覽器中的 JSON 回應

```json
{"message":"Hello World"}
```

回到終端按下 `Ctrl+C` 停止 API 後，重新整理同一個網址會變成無法連線。

## 步驟一：建立獨立練習資料夾

先移到你平常放練習的資料夾，再執行下列命令。若目前位置屬於另一個 `uv` 專案，請先換到該專案外再開始。

執行位置：PowerShell／電腦  
建立項目：`fastapi-hello` 資料夾

```powershell
New-Item -ItemType Directory -Path fastapi-hello
Set-Location fastapi-hello
uv init --bare
```

預期結果：目前位置變成新的 `fastapi-hello` 資料夾，裡面出現 `pyproject.toml`。

`uv init --bare` 會建立最小 Python 專案設定，不會自動產生一組範例程式。

## 步驟二：安裝 FastAPI

`fastapi[standard]` 會安裝 FastAPI，以及執行這個開發用 API 所需的常用工具。

執行位置：PowerShell／電腦  
目前資料夾：`fastapi-hello`

```powershell
uv add "fastapi[standard]"
```

預期結果：終端顯示套件處理完成，資料夾中出現或更新 `uv.lock`。第一次執行可能需要等待套件下載。

若網路中斷、套件來源無法連線或命令停在錯誤訊息，先停止本頁操作；不要改用來源不明的安裝檔。

## 步驟三：建立 Hello World API

在 `fastapi-hello` 資料夾建立 `main.py`，填入以下內容。

執行位置：Python／電腦  
檔名：`fastapi-hello/main.py`

```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def read_root() -> dict[str, str]:
    return {"message": "Hello World"}
```

這段程式只有四個重點：

- `FastAPI()` 建立 API 應用，並存入 `app`。
- `@app.get("/")` 把下一個函式連到 `GET /`。`/` 代表網站的根路徑。
- `read_root()` 是收到這項請求時執行的函式；`-> dict[str, str]` 表示它預期回傳「文字欄位對應文字值」的字典。
- 回傳的 Python 字典會由 FastAPI 轉成 JSON 回應。

Type Hint（型別提示）幫助閱讀與工具檢查，但不會在一般執行時自動檢查所有外部資料。本頁沒有接收外部資料，因此只需要這個簡單回傳型別。

## 步驟四：啟動本機 API

`fastapi dev` 會啟動開發用伺服器；`main.py` 告訴它要載入哪個檔案。

執行位置：PowerShell／電腦  
目前資料夾：`fastapi-hello`

```powershell
uv run fastapi dev main.py
```

預期結果：終端保持執行中，並顯示服務位於 `http://127.0.0.1:8000`。不同版本的提示文字可能略有差異，但網址與 port 應相同。

不要關閉這個終端；API 必須保持執行，瀏覽器才能取得回應。

## 步驟五：從瀏覽器呼叫 API

開啟瀏覽器，在網址列輸入：

用途：本機 API 網址

```text
http://127.0.0.1:8000/
```

預期結果：瀏覽器顯示 `{"message":"Hello World"}`。這表示瀏覽器已向 `GET /` 提出請求，FastAPI 也已執行 `read_root()` 並回傳 JSON。

這個結果只證明同一台電腦上的最小 API 可以使用，不代表 API 已公開到網路，也不代表它已有登入或權限保護。

## 步驟六：停止 API

回到正在執行 FastAPI 的 PowerShell，按下 `Ctrl+C`。

預期結果：終端回到可以輸入命令的狀態。回到瀏覽器重新整理 `http://127.0.0.1:8000/`，應無法再取得 Hello World 回應。

如果瀏覽器仍顯示舊文字，可再重新整理一次；若仍能取得新回應，先確認是否有另一個終端仍在執行相同服務。

## 完成時，應能確認

- `fastapi-hello` 是獨立練習資料夾，並有自己的 `pyproject.toml` 與 `uv.lock`。
- `main.py` 只有一個 `GET /` 路徑，回傳固定的 Hello World 字典。
- API 執行時，瀏覽器能看到 JSON 回應。
- 按下 `Ctrl+C` 後，同一個網址不再取得回應。
- 整個練習只使用本機 `127.0.0.1`，沒有建立公開網址。

## 常見問題

### `uv add` 顯示無法下載套件

先確認電腦可以連上網路，再重新執行一次。若使用的網路限制 Python 套件下載，先停止操作，不要關閉安全軟體或改用不明套件來源。

### 終端顯示 port `8000` 已被使用

可能已有另一個程式使用相同 port。先找出並停止先前啟動的練習服務；本頁不要為了避開問題任意改成其他 port，否則網址與預期結果會不一致。

### 瀏覽器顯示無法連線

確認執行 `uv run fastapi dev main.py` 的終端仍保持開啟，而且網址是 `http://127.0.0.1:8000/`。若終端已有紅色錯誤訊息，先依最上方的錯誤原因檢查檔名、縮排與套件安裝結果。

### `uv init` 修改了另一個專案的設定

這通常表示 `fastapi-hello` 建在另一個 `uv` 專案裡。停止操作，移除這次新建的練習資料夾前先確認其中沒有自己的重要檔案，再到一般練習位置重新建立；不要直接刪除原專案的設定。

## 重點整理

- `FastAPI()` 建立 API 應用，`@app.get("/")` 把 Python 函式連到 `GET /`。
- Python 字典可以成為瀏覽器看到的 JSON 回應。
- `127.0.0.1` 只代表目前這台電腦。
- API 需要在終端保持執行；`Ctrl+C` 可以停止這次開發服務。

## 參考資料

- [FastAPI 官方教學：First Steps](https://fastapi.tiangolo.com/tutorial/first-steps/)
- [FastAPI 官方說明：Virtual Environments](https://fastapi.tiangolo.com/virtual-environments/)
- [uv 官方說明：Creating projects](https://docs.astral.sh/uv/concepts/projects/init/)

## 下一步

你可以回到[前導技術概要](index.md)，依目前需要選擇其他概念或選做實作。其他前導實作不以本頁為必要條件。
