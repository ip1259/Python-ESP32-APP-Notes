# FastAPI：本機狀態 API 的概念

FastAPI 讓 Python 程式能用一個固定網址，回答「目前狀態是什麼」這類問題。

本頁範例以[第 3 章主頁](index.md)固定的 Python `3.12` 練習環境為準。

## 你會學到什麼

- 用日常語言理解 API、路由（route）與 JSON 回應。
- 看懂本章 `GET /status` 如何取得 Gateway 的最新狀態。
- 分辨 `127.0.0.1`、`localhost` 與公開網址的差別。
- 看懂 `200`、`404`、`405` 代表的基本結果。

## 開始前

- 已完成[第 3 章主頁](index.md)，並能啟動 `gateway.py`。
- 本頁只介紹本機唯讀查詢；不需要手機、公開網址、帳號或資料庫。

## API 像什麼？

可以把 API 想成程式提供的一個「固定問答入口」。瀏覽器、PowerShell 或另一個程式向入口提出問題，程式回傳整理好的答案。

本章的問題是：**Gateway 目前的最新狀態是什麼？**

```text
PowerShell 或瀏覽器
        │ 詢問 GET /status
        ▼
FastAPI ──→ Gateway 的 latest 狀態
        │ 回傳 JSON
        ▼
目前狀態、授權模擬結果與更新時間
```

API 不會直接讀 RC522，也不會自行連 MQTT。它只是從正在執行的 Gateway 取得已整理的最新摘要。

## 路由：網址中的工作名稱

`gateway.py` 有這段程式：

```python
@app.get("/status")
def read_status() -> dict[str, object]:
    return latest_status()
```

`@app.get("/status")` 可以分成三部分理解：

| 部分 | 意思 |
| --- | --- |
| `app` | 本章建立的 FastAPI 程式。 |
| `get` | 只讀取資料，不新增或修改資料。 |
| `/status` | 這個問題的名稱，也就是路由。 |

當你開啟 `http://127.0.0.1:8000/status`，FastAPI 就會執行 `read_status()`。函式的回傳型別 `dict[str, object]` 表示它回傳「欄位名稱是文字」的字典；FastAPI 會將字典轉成 JSON。

## JSON：程式容易閱讀的回答

當 Gateway 尚未收到事件時，回答類似：

```json
{
  "event_id": null,
  "status": "waiting",
  "authorized": null,
  "display_text": "等待教學事件",
  "updated_at": "2026-09-17T10:00:00+00:00"
}
```

JSON 是用 `{}` 包住「欄位：值」的文字格式。它適合讓不同程式交換資料；例如之後的 LCD 或手機頁面只需要讀取 `status`、`authorized` 和 `display_text`，不需要知道 MQTT 帳密、CSV 完整內容或白名單。

| 欄位 | 本章用途 |
| --- | --- |
| `event_id` | 對應最近一次已接受的虛構事件。 |
| `status` | 例如 `waiting`、`authorized`、`unauthorized` 或 `unknown`。 |
| `authorized` | `True`、`False` 或 `null`；`null` 代表現在無法可靠判斷。 |
| `display_text` | 供人閱讀的短摘要。 |
| `updated_at` | 最後更新的含時區時間。 |

## `127.0.0.1` 與 localhost

`127.0.0.1` 是「這一台電腦自己」的固定網路位址；`localhost` 也是常見的同義名稱。因此：

```text
http://127.0.0.1:8000/status
```

只讓目前執行 Gateway 的電腦查詢。`8000` 是程式暫時使用的連接埠（port），可把它想成同一台電腦中用來找到這個程式的號碼。

本章用下列方式啟動：

```python
uvicorn.run(app, host="127.0.0.1", port=8000)
```

`host="127.0.0.1"` 是重要限制：不要改成 `0.0.0.0`、電腦的區網位址或公開網址。這一頁沒有處理公開服務需要的帳號、權限、加密設定或長期維護。

## 步驟 1：查詢本機 API

**目的：** 看到 FastAPI 將 Python 字典轉成 JSON 的結果。

先在一個 PowerShell 視窗啟動 `gateway.py`。在另一個視窗執行：

執行位置：PowerShell／電腦

```powershell
Invoke-RestMethod http://127.0.0.1:8000/status | ConvertTo-Json
```

預期結果：看到 `waiting`，或看到最近一次測試的安全摘要。程式停止後，這個指令無法連線；CSV 中的舊資料不會被拿來假裝目前服務仍在執行。

## 步驟 2：認識三個常見結果

| 你做的事 | 常見結果 | 代表什麼 |
| --- | --- | --- |
| 用 `GET` 查詢 `/status` | `200 OK` | 程式已找到路由並正常回覆。 |
| 查詢 `/unknown` | `404 Not Found` | 沒有這個路由名稱。 |
| 對 `/status` 使用 `POST` | `405 Method Not Allowed` | 路由存在，但本章只允許讀取。 |

你可在 Gateway 執行時試著開啟：

```text
http://127.0.0.1:8000/unknown
```

預期結果：瀏覽器顯示找不到該路由。這不是 MQTT 或 CSV 壞掉，只是網址名稱不存在。

## 完成時，應能確認

- 能說出 `/status` 是查詢最新狀態的固定入口。
- 能說出 FastAPI 把 Python 字典轉成 JSON 回傳。
- 能區分 `waiting` 與 `unknown`：前者尚未收到事件，後者是舊資料已不能代表現在。
- 能確認本章 API 只綁定 `127.0.0.1`。

## 常見問題

### 瀏覽器顯示無法連線

確認 `gateway.py` 的 PowerShell 視窗仍在執行，且訊息中有 `Uvicorn running`。若已按 `Ctrl+C`，重新啟動 Gateway 再查詢。

### 為什麼不是直接開啟 CSV？

CSV 記錄的是過去寫入的資料；API 回應的是目前仍在執行的 Gateway 狀態。兩者用途不同，不能用 CSV 取代目前狀態。

### `authorized` 為什麼有時是 `null`？

`null` 對應 Python 的 `None`，表示現在沒有可靠結果，例如剛啟動、MQTT 中斷或資料已過期。它不是未授權。

## 重點整理

- FastAPI 用路由讓其他程式以固定網址詢問 Python 程式。
- `GET /status` 是本章唯一的唯讀入口，回傳最小 JSON 摘要。
- `127.0.0.1` 只允許同一台電腦查詢，是本章的安全範圍。

## 下一步

回到[第 3 章主頁](index.md)實際查看 API，或閱讀[Gateway 程式導讀](gateway程式導讀.md)了解完整資料流。

若想知道網站後端在許多人同時使用、多台服務交換資料時可能遇到什麼問題，可延伸閱讀[高併發、分散式系統與資料同步問題](../../資訊補充-高併發分散式系統與資料同步問題.md)。
