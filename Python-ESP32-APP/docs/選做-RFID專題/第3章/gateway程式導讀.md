# Gateway 程式導讀

這頁用生活化方式整理 Gateway 如何收件、判斷、記帳與回覆。

## 你會學到什麼

- 看懂 Gateway 為什麼先檢查事件欄位。
- 了解黑白名單與五分鐘規則的判斷順序。
- 分辨記憶體中的判斷資料與 CSV 紀錄資料。

## 開始前

- 已閱讀[第 3 章：Gateway 名單與最小紀錄](index.md)。
- 本頁是資訊補充，不需要修改程式或連線。

## Gateway 像一個收發室

```text
收到 MQTT 訊息
    ↓
確認是否為第 0 章約定的事件
    ↓
黑名單 → 白名單 → 未註冊卡的五分鐘計數
    ↓
寫入最小 CSV 紀錄
    ↓
回覆 event_id、status、updated_at
```

`validate_event()` 像收發室先看包裹標籤。事件必須剛好有 `event_id`、`device_id`、`occurred_at`、`uid_envelope`；信封也必須剛好有 `key_id`、`nonce`、`ciphertext`、`tag`。欄位不符就不繼續處理。

## 名單規則的順序

核心判斷是 `decide_status()`：

```python
if ciphertext in BLACKLIST:
    return "blacklisted"
if ciphertext in WHITELIST:
    return "whitelisted"
```

先查黑名單能避免一個項目同時出現在兩份名單時被誤放行。只有不在黑名單的項目，才會檢查白名單。

若兩份名單都沒有，程式把同一 `device_id`、同一 `ciphertext` 的近期次數放進 `recent_unregistered_attempts`。五分鐘內第 1、2 次回覆 `unregistered`，第 3 次起回覆 `review_required`。它只是提醒後續處理，不會改動 `BLACKLIST`。

## 為什麼 CSV 不保存信封

程式要判斷名單時會暫時讀取 `ciphertext`，但 `append_log()` 寫入 CSV 時只選擇：

```python
LOG_FIELDS: list[str] = [
    "event_id",
    "device_id",
    "status",
    "occurred_at",
    "updated_at",
]
```

這像出貨明細只記錄訂單編號與處理結果，不影印包裹內容。讀卡紀錄可供瀏覽，但不應因此多保存不需要的資料。

## 為什麼會略過自己的 MQTT 回覆

ESP32 與 Gateway 在練習時訂閱同一個 topic。Gateway 發出的回覆也會被自己收到；回覆只有三個欄位，不是讀卡事件，因此程式會略過：

```python
if isinstance(received_value, dict) and set(received_value) == REPLY_FIELDS:
    return None
```

這個檢查也讓 Gateway 不會把自己的回覆再當成新事件處理一次。

## 本機狀態 API

`read_status()` 只回傳 `latest_status` 的副本：

```python
@app.get("/status")
def read_status() -> dict[str, str]:
    with state_lock:
        return latest_status.copy()
```

`Lock` 可以想成收發室的一把鑰匙：MQTT 收到新事件與瀏覽器查詢狀態不會同時改動同一份資料。`host="127.0.0.1"` 讓 API 只在 Gateway 所在電腦使用。

## 重點整理

- 先確認欄位，才進入名單規則。
- 黑名單優先，未註冊卡不會被自動加入黑名單。
- CSV 是最小紀錄，信封內容只留在處理當下的記憶體。
- MQTT 回覆與本機 API 都只提供需要的摘要。

## 下一步

回到[第 3 章主頁](index.md)實際操作，或閱讀[FastAPI 概念介紹](fastapi概念介紹.md)。
