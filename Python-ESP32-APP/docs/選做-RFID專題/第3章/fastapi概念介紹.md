# FastAPI：本機狀態 API 的概念

FastAPI 讓正在執行的 Python Gateway 用固定網址回答「最近處理結果是什麼」。

## 你會學到什麼

- 了解 API、網址路徑與 JSON 回應各自的意思。
- 看懂 `GET /status` 如何取得 Gateway 最近一次狀態。
- 知道 `127.0.0.1` 為什麼只限本機使用。

## 開始前

- 已閱讀[第 3 章：Gateway 名單與最小紀錄](index.md)。
- 本頁是資訊補充，不需要建立公開網址或外部服務。

## API 像一個固定的詢問窗口

API（Application Programming Interface，應用程式介面）讓程式之間用約定的方式交換資料。本章的窗口是：

```text
http://127.0.0.1:8000/status
```

其中 `/status` 是網址路徑，表示「請給我目前狀態」；`GET` 是讀取資料的方式。Gateway 沒有收到事件時，可能回傳：

```json
{
  "status": "waiting"
}
```

收到事件後，回傳會多出事件代號與更新時間：

```json
{
  "event_id": "evt-1234ABCD",
  "status": "whitelisted",
  "updated_at": "2030-01-02T03:04:05+00:00"
}
```

JSON 是用 `{}` 包住欄位與值的文字格式，適合讓不同程式交換資料。

## 對應到程式的寫法

執行位置：Python／電腦
檔案：`rfid-project/gateway/gateway.py`

```python
@app.get("/status")
def read_status() -> dict[str, str]:
    with state_lock:
        return latest_status.copy()
```

`@app.get("/status")` 表示網址收到 `GET /status` 時，交給下方的 `read_status()` 函式回答。函式回傳 Python 字典，FastAPI 會把它轉成 JSON。

`copy()` 交出一份資料副本，讓查詢者不能直接改動 Gateway 正在保存的狀態。

## `127.0.0.1` 的範圍

`127.0.0.1` 是「這台電腦自己」的位址。程式使用：

執行位置：Python／電腦
檔案：`rfid-project/gateway/gateway.py`

```python
uvicorn.run(app, host="127.0.0.1", port=8000)
```

因此同一台電腦的瀏覽器或 PowerShell 可以查詢，其他裝置不能直接連入。`8000` 是這個程式使用的連接埠，可把它想成電腦裡的窗口號碼。

## 成功的樣子

Gateway 正在執行時，執行下列指令：

執行位置：PowerShell／電腦

```powershell
Invoke-RestMethod http://127.0.0.1:8000/status | ConvertTo-Json
```

預期結果：看到 JSON 格式的目前狀態。查詢 `/unknown` 會得到 `404 Not Found`，表示沒有這個窗口；用 `POST` 查詢 `/status` 會得到 `405 Method Not Allowed`，表示這個窗口只接受讀取。

## 常見問題

### 網址無法連線

先確認 Gateway 的 PowerShell 視窗仍在執行，且沒有按下 `Ctrl+C`。確認網址是 `127.0.0.1` 而不是別台電腦的位址。

### API 回傳 `waiting`

這表示 Gateway 還沒有處理有效的讀卡事件。先檢查 MQTT 設定與第 2 章的事件欄位，再重新查詢。

## 重點整理

- FastAPI 將 Python 的回傳字典轉成 JSON。
- `GET /status` 只讀取 Gateway 最近一次的結果，不讀 RC522，也不修改名單。
- `127.0.0.1` 把 API 限制在本機，是這個階段的重要安全界線。

## 下一步

回到[第 3 章主頁](index.md)查看完整流程，或閱讀[Gateway 程式導讀](gateway程式導讀.md)。
