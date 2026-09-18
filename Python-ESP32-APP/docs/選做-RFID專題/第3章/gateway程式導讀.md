# Gateway 程式導讀

> 選讀｜對應：[第 3 章：Python MQTT Gateway 與 FastAPI 初步概念](index.md)

本頁用第 3 章已成功運作的 `gateway.py`，說明固定虛構事件如何依序變成 CSV、MQTT 回覆與本機狀態 API。

本頁程式以[第 3 章主頁](index.md)固定的 Python `3.12` 練習環境為準。

## 讀完後你會知道

- Gateway 為什麼要先檢查資料，再產生授權模擬結果。
- MQTT 回呼、CSV 與 FastAPI 如何共用同一筆最新狀態。
- 為什麼重複、過期或斷線時不能沿用上一筆授權結果。

## 先知道這件事

主頁已完成 Gateway 的本機測試。本頁只在成功程式上加入 `【編號】` 註解，不需要重新設定帳密或再次啟動 Broker。

資料流如下：

```text
MQTT 虛構事件 → validate_event() → reply_for() → CSV 與 MQTT 回覆
                                            └→ latest → GET /status
```

## 完整程式導讀

執行位置：Python／電腦
來源檔案：`gateway.py`

```python linenums="1"
# 【1】匯入檔案、時間、JSON、同步與網路服務工具
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

# 【2】秘密只放在未加入 Git 的設定檔
from mqtt_settings import MQTT_HOST, MQTT_PASSWORD, MQTT_PORT, MQTT_TOPIC, MQTT_USERNAME

# 【3】集中定義資料契約、虛構白名單、CSV 欄位與共用狀態
EVENT_ID = re.compile(r"^[a-z0-9-]{3,20}$")
CARD_ID = re.compile(r"^CARD-\*\*\*\*-\d{2}$")
ALLOWED_CARDS = {"CARD-****-42"}
LOG_PATH = Path("data") / "rfid_gateway_log.csv"
FIELDS = ["event_id", "device_id", "card_id_masked", "status", "authorized", "display_text", "updated_at"]
latest: dict[str, object] | None = None
handled_ids: set[str] = set()
lock = Lock()

# 【4】本章只提供一個本機唯讀路由，不建立文件或其他端點
app = FastAPI(docs_url=None, redoc_url=None, openapi_url=None)


# 【5】建立含時區的現在時間
def now() -> str:
    return datetime.now(timezone.utc).isoformat()


# 【6】建立沒有授權結果的安全狀態
def unavailable(status: str, text: str) -> dict[str, object]:
    return {
        "event_id": None,
        "status": status,
        "authorized": None,
        "display_text": text,
        "updated_at": now(),
    }


# 【7】只接受第 0 章規定的完整虛構事件
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


# 【8】只對已驗證資料做固定練習卡比對，建立最小回覆
def reply_for(event: dict[str, object]) -> dict[str, object]:
    authorized = event["card_id_masked"] in ALLOWED_CARDS
    return {
        "event_id": event["event_id"],
        "status": "authorized" if authorized else "unauthorized",
        "authorized": authorized,
        "display_text": "已讀取授權練習卡" if authorized else "練習卡未在教學白名單",
        "updated_at": now(),
    }


# 【9】以固定欄位附加 CSV；失敗時回傳 False
def append_log(event: dict[str, object], reply: dict[str, object]) -> bool:
    row = {
        "event_id": reply["event_id"],
        "device_id": event["device_id"],
        "card_id_masked": event["card_id_masked"],
        **{key: reply[key] for key in FIELDS[3:]},
    }
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


# 【10】FastAPI 收到 GET /status 時，回傳記憶體中的最新安全摘要
@app.get("/status")
def read_status() -> dict[str, object]:
    with lock:
        if latest is None:
            return unavailable("waiting", "等待教學事件")
        updated_at = datetime.fromisoformat(str(latest["updated_at"]))
        if (datetime.now(timezone.utc) - updated_at).total_seconds() > 60:
            return unavailable("unknown", "目前狀態未知")
        return latest


# 【11】MQTT 連上 Broker 後只訂閱設定檔指定的完整 topic
def on_connect(client: mqtt.Client, userdata: object, flags: object, reason_code: object, properties: object) -> None:
    if not getattr(reason_code, "is_failure", False):
        client.subscribe(MQTT_TOPIC, qos=0)
        print("Gateway 已訂閱專屬 topic")


# 【12】中斷時清除授權判定，避免舊結果看起來仍可用
def on_disconnect(client: mqtt.Client, userdata: object, disconnect_flags: object, reason_code: object, properties: object) -> None:
    global latest
    with lock:
        latest = unavailable("broker_disconnected", "訊息服務未連線")


# 【13】收到 MQTT 訊息時的完整處理順序
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
        # 【13.1】同一事件代號在本次執行中只處理一次
        if event_id in handled_ids:
            print("收到重複事件，已略過")
            return
        # 【13.2】CSV 成功後，才更新最新狀態和發布回覆
        if not append_log(event, reply):
            print("CSV 無法寫入；本次事件未建立回覆")
            return
        handled_ids.add(event_id)
        latest = reply

    client.publish(MQTT_TOPIC, json.dumps(reply, ensure_ascii=False), qos=0, retain=False)
    print("已處理事件並發布 Gateway 回覆")


# 【14】建立 MQTT Client、設定 TLS 與回呼，再只在本機啟動 API
def main() -> None:
    client = mqtt.Client(
        CallbackAPIVersion.VERSION2,
        client_id="rfid-gateway-test-01",
        reconnect_on_failure=False,
    )
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


# 【15】只有直接執行這個檔案時才啟動 Gateway
if __name__ == "__main__":
    main()
```

## 依資料流閱讀

先看【1】至【4】準備工具、設定與共用狀態；再看【5】至【9】如何建立安全資料；【10】是本機 API；【11】至【13】是 MQTT 回呼；最後【14】至【15】負責啟動和收尾。

最重要的順序在【13】：先解碼 JSON、驗證欄位、排除重複、寫入 CSV，最後才更新 `latest` 與發布回覆。因此不合法資料、重複事件和 CSV 寫入失敗都不會變成授權結果。

## 出錯時先看哪裡

| 看到的現象 | 第一個檢查位置 | 可先判斷的範圍 |
| --- | --- | --- |
| 顯示「收到無法使用的資料」 | 【7】與 ESP32 事件欄位 | 訊息格式不符合資料契約。 |
| API 有新狀態，CSV 沒有資料 | 【9】的資料夾與檔案權限 | MQTT 已送到 Gateway，問題在本機檔案。 |
| CSV 有資料，ESP32 未收到回覆 | 【13】的 `publish()` 與 ESP32 的 `event_id` 比對 | Gateway 已處理資料，檢查回覆路徑。 |
| API 變成 `unknown` | 【10】的 60 秒判斷 | 舊資料不再代表目前狀態。 |
| API 顯示 `broker_disconnected` | 【12】 | MQTT 服務目前未連線。 |

## 常見誤解

### 「`append_log()` 成功就代表 ESP32 已收到回覆」

不是。它只表示 CSV 已寫入。`publish()` 只表示 Gateway 已嘗試把回覆交給 MQTT；ESP32 仍要以相符的 `event_id` 確認收到。

### 「重啟後可以直接用 CSV 當最新狀態」

不行。CSV 是過去紀錄；重啟後 `latest` 是空值，所以 API 回傳 `waiting`，直到收到新的事件。

## 重點整理

- 完整程式依序驗證、建立回覆、寫 CSV、更新狀態與發布 MQTT。
- `lock` 保護 MQTT 回呼與 API 同時存取共用狀態。
- 過期或斷線時以未知狀態取代舊的授權結果。

## 回到專題

回到[第 3 章主頁](index.md)，或閱讀[FastAPI 概念介紹](fastapi概念介紹.md)。
