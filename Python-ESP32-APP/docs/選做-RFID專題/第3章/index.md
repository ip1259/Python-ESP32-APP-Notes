# 選做專題第 3 章：Python MQTT Gateway 與 FastAPI 初步概念

用 Python 接收遮蔽的虛構讀卡事件，產生教學用結果、寫入最小 CSV，並在本機查詢最新狀態。

## 你會學到什麼

- 說明 Gateway 如何在 ESP32、MQTT、CSV 與 API 之間分工。
- 以固定虛構資料產生 `authorized` 或 `unauthorized`。
- 用本機 `GET /status` 查詢必要最新狀態。

## 開始前

- 已完成[第 0 章](../第0章/index.md)與[第 2 章](../第2章/index.md)。
- 需要可控制的 Wi-Fi、Broker 與新的 Gateway 專用、可撤銷帳密。
- 需要 `uv` 可使用 Python `3.12`；本章會把版本固定在新建立的練習資料夾，不會修改電腦的系統 Python。
- 只用 `CARD-****-42`、`CARD-****-99` 等虛構資料；不得使用 UID、個資、帳密或實際 topic。

## 成功的樣子

```text
ESP32 固定假事件 → MQTT → Python Gateway → CSV／MQTT 回覆／本機 API
```

`CARD-****-42` 的最新狀態為 `authorized`、`True`；`CARD-****-99` 為 `unauthorized`、`False`。

## 步驟 1：建立專用專案

執行位置：PowerShell／電腦

```powershell
mkdir rfid-gateway
cd rfid-gateway
uv init --python 3.12
uv add paho-mqtt fastapi uvicorn
```

`uv init --python 3.12` 會在建立專案時，同時設定 `pyproject.toml` 的 Python 需求與 `.python-version`。因此 `uv add` 建立的套件環境一開始就會使用 Python `3.12`。

執行位置：PowerShell／電腦

```powershell
uv run python --version
```

預期結果：顯示 `Python 3.12` 開頭的版本文字；資料夾也有 `pyproject.toml` 與 `.python-version`，套件只安裝在這個練習專案。若無法取得 Python `3.12`，停止本章，不要改用其他版本繼續操作。

## 步驟 2：建立不公開的設定

執行位置：Python／電腦
檔案：`.gitignore`

```text
mqtt_settings.py
data/
```

執行位置：Python／電腦
檔案：`mqtt_settings.py`

```python
MQTT_HOST = "你的 Broker 主機名稱"
MQTT_PORT = 8883
MQTT_TOPIC = "你的單一專屬 messages topic"
MQTT_USERNAME = "Gateway 專用測試帳號"
MQTT_PASSWORD = "Gateway 專用測試密碼"
```

Gateway 帳密必須不同於 ESP32 帳密，只可對這一個完整 topic 有 `pub/sub` 權限；不得使用 `#`、`+` 或共用帳密。

## 步驟 3：建立 Gateway

執行位置：Python／電腦
檔案：`gateway.py`

```python
import csv
from datetime import datetime, timezone
import json
from pathlib import Path
import re
from threading import Lock

from fastapi import FastAPI
import paho.mqtt.client as mqtt
from paho.mqtt.enums import CallbackAPIVersion
import uvicorn

from mqtt_settings import MQTT_HOST, MQTT_PASSWORD, MQTT_PORT, MQTT_TOPIC, MQTT_USERNAME

EVENT_ID = re.compile(r"^[a-z0-9-]{3,20}$")
CARD_ID = re.compile(r"^CARD-\*\*\*\*-\d{2}$")
ALLOWED_CARDS = {"CARD-****-42"}
LOG_PATH = Path("data") / "rfid_gateway_log.csv"
FIELDS = ["event_id", "device_id", "card_id_masked", "status", "authorized", "display_text", "updated_at"]
latest: dict[str, object] | None = None
handled_ids: set[str] = set()
lock = Lock()
app = FastAPI(docs_url=None, redoc_url=None, openapi_url=None)


def now() -> str:
    return datetime.now(timezone.utc).isoformat()


def unavailable(status: str, text: str) -> dict[str, object]:
    return {"event_id": None, "status": status, "authorized": None, "display_text": text, "updated_at": now()}


def validate_event(value: object) -> dict[str, object]:
    required = {"event_type", "card_id_masked", "device_id", "occurred_at", "event_id"}
    if not isinstance(value, dict) or set(value) != required:
        raise ValueError
    if value["event_type"] != "card_read":
        raise ValueError
    if not isinstance(value["event_id"], str) or not EVENT_ID.fullmatch(value["event_id"]):
        raise ValueError
    if not isinstance(value["card_id_masked"], str) or not CARD_ID.fullmatch(value["card_id_masked"]):
        raise ValueError
    if not isinstance(value["device_id"], str):
        raise ValueError
    try:
        if datetime.fromisoformat(str(value["occurred_at"])).tzinfo is None:
            raise ValueError
    except ValueError as error:
        raise ValueError from error
    return value


def reply_for(event: dict[str, object]) -> dict[str, object]:
    authorized = event["card_id_masked"] in ALLOWED_CARDS
    return {"event_id": event["event_id"], "status": "authorized" if authorized else "unauthorized", "authorized": authorized, "display_text": "已讀取授權練習卡" if authorized else "練習卡未在教學白名單", "updated_at": now()}


def append_log(event: dict[str, object], reply: dict[str, object]) -> bool:
    row = {"event_id": reply["event_id"], "device_id": event["device_id"], "card_id_masked": event["card_id_masked"], **{key: reply[key] for key in FIELDS[3:]}}
    try:
        LOG_PATH.parent.mkdir(exist_ok=True)
        with LOG_PATH.open("a", encoding="utf-8", newline="") as file:
            writer = csv.DictWriter(file, fieldnames=FIELDS)
            if file.tell() == 0:
                writer.writeheader()
            writer.writerow(row)
        return True
    except OSError:
        return False


@app.get("/status")
def read_status() -> dict[str, object]:
    with lock:
        if latest is None:
            return unavailable("waiting", "等待教學事件")
        if (datetime.now(timezone.utc) - datetime.fromisoformat(str(latest["updated_at"]))).total_seconds() > 60:
            return unavailable("unknown", "目前狀態未知")
        return latest


def on_connect(client: mqtt.Client, userdata: object, flags: object, reason_code: object, properties: object) -> None:
    if not getattr(reason_code, "is_failure", False):
        client.subscribe(MQTT_TOPIC, qos=0)
        print("Gateway 已訂閱專屬 topic")


def on_disconnect(client: mqtt.Client, userdata: object, disconnect_flags: object, reason_code: object, properties: object) -> None:
    global latest
    with lock:
        latest = unavailable("broker_disconnected", "訊息服務未連線")


def on_message(client: mqtt.Client, userdata: object, message: mqtt.MQTTMessage) -> None:
    global latest
    try:
        event = validate_event(json.loads(message.payload.decode("utf-8")))
    except (UnicodeDecodeError, json.JSONDecodeError, ValueError):
        print("收到無法使用的資料，已略過")
        return
    reply = reply_for(event)
    event_id = str(reply["event_id"])
    with lock:
        if event_id in handled_ids or not append_log(event, reply):
            return
        handled_ids.add(event_id)
        latest = reply
    client.publish(MQTT_TOPIC, json.dumps(reply, ensure_ascii=False), qos=0, retain=False)
    print("已處理事件並發布 Gateway 回覆")


def main() -> None:
    client = mqtt.Client(CallbackAPIVersion.VERSION2, client_id="rfid-gateway-test-01", reconnect_on_failure=False)
    client.username_pw_set(MQTT_USERNAME, MQTT_PASSWORD)
    client.tls_set()
    client.on_connect = on_connect
    client.on_disconnect = on_disconnect
    client.on_message = on_message
    try:
        client.connect(MQTT_HOST, MQTT_PORT, keepalive=30)
        client.loop_start()
        uvicorn.run(app, host="127.0.0.1", port=8000)
    finally:
        client.disconnect()
        client.loop_stop()


if __name__ == "__main__":
    main()
```

`validate_event()` 先拒絕缺欄位、未遮蔽值與錯誤時間；`reply_for()` 才做固定練習卡比對。`append_log()` 只保存必要欄位。資料超過 60 秒時，API 回傳 `unknown`。

## 步驟 4：啟動並確認結果

執行位置：PowerShell／電腦

```powershell
uv run python gateway.py
```

預期結果：顯示 `Gateway 已訂閱專屬 topic` 與 `Uvicorn running on http://127.0.0.1:8000`。

在第 2 章 ESP32 程式依序送出 `CARD-****-42`、`evt-040`，再送出 `CARD-****-99`、`evt-041`。每次都使用新的事件代號，ESP32 也必須比對本次送出的 `event_id`，不能只檢查欄位名稱。

執行位置：PowerShell／電腦

```powershell
Invoke-RestMethod http://127.0.0.1:8000/status | ConvertTo-Json
Import-Csv data\rfid_gateway_log.csv | Format-Table
```

預期結果：API 只回傳 `event_id`、`status`、`authorized`、`display_text`、`updated_at`；CSV 有兩筆虛構摘要。API 只能使用 `127.0.0.1`，不可改為區網或公開網址。

## 完成時，應能確認

- Gateway 與 ESP32 使用不同的短期帳密，且只使用單一完整 topic。
- 兩個虛構遮蔽值產生不同結果，且回覆的 `event_id` 與本次事件相符。
- CSV 不含 UID、帳密、topic、主機名稱或原始 MQTT 資料。
- MQTT 中斷或資料過期時，API 不會沿用上一筆授權結果。

## 常見問題

### 顯示 MQTT 連線失敗

確認 Gateway 帳密、TLS 連接埠、完整 topic 權限與可控制 Wi-Fi。不要印出秘密；無法確認時停止並重建短期帳密。

### `uv run python --version` 不是 `Python 3.12`

確認目前位置是 `rfid-gateway`，並檢查 `.python-version` 內容是 `3.12`。若仍無法取得或建立 Python `3.12` 環境，停止本章；不要混用其他 Python 版本的既有套件。

### PowerShell 的中文顯示亂碼

```powershell
chcp 65001 | Out-Null
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
```

### CSV 沒有新增資料

確認 Gateway 沒有顯示「已略過」，事件代號沒有重複，且 `data` 資料夾可寫入。

## 安全收尾

按 `Ctrl+C` 停止 Gateway。確認 CSV 只含虛構資料後再刪除；撤銷 Gateway 測試帳密並移除 `mqtt_settings.py`。

## 重點整理

- Gateway 先驗證資料，再進行固定練習比對、CSV 紀錄與 MQTT 回覆。
- FastAPI 在本章只有本機唯讀 `GET /status`。
- 失敗與過期必須表達未知，不能沿用舊的授權結果。

## 下一步

想逐段理解程式責任，閱讀[Gateway 程式導讀](gateway程式導讀.md)；想先認識 API，閱讀[FastAPI 概念介紹](fastapi概念介紹.md)。接著閱讀[專題第 4 章：LCD 狀態顯示](../第4章/index.md)。
