# 選做專題第 3 章：Gateway 名單與最小紀錄

Gateway 把 MQTT 收到的練習信封集中判斷，留下必要紀錄，再把結果回覆給 ESP32。

## 你會學到什麼

- 建立固定 Python 與套件版本的 Gateway 執行環境。
- 依黑名單、白名單與未註冊卡規則產生讀卡狀態。
- 將最少需要的欄位寫入 CSV，而不保存信封內容。
- 在本機用 `GET /status` 查看 Gateway 最近一次的處理結果。

## 開始前

- 已完成[第 0 章：準備、安全與資料契約](../第0章/index.md)與[第 2 章：ESP32 MQTT 假信封事件](../第2章/index.md)。
- 電腦已安裝 `uv` 與 Python `3.12`；專題資料夾為第 0 章建立的 `rfid-project`。
- 需要 HiveMQ Cloud 測試帳號與第 2 章使用的測試 topic。ESP32 和 Gateway 各自使用一組帳密。
- 本章只使用 `practice-allowed`、`practice-blocked` 等練習內容；不讀取、保存或傳送真實卡片 UID。

!!! warning "網路與資料安全"

    MQTT 中介服務只負責轉送訊息，不負責判斷誰可通行。即使內容是練習資料，也不要把帳密、實際 UID、私有位址或完整 topic 發布到公開場所。Gateway 的 API 只開在自己的電腦 `127.0.0.1`，不要改成可從網路連入的位址。

## 成功的樣子

```text
ESP32 練習信封 → MQTT 中介服務 → Python Gateway
                                      ↓
                        名單規則 → CSV 最小紀錄 → MQTT 回覆
                                      ↓
                              GET /status（本機）
```

Gateway 回覆只有 `event_id`、`status`、`updated_at` 三個欄位。

| 狀態 | 意思 |
| --- | --- |
| `blacklisted` | 信封內容在黑名單，拒絕。黑名單永遠優先。 |
| `whitelisted` | 信封內容在白名單，可通行。 |
| `unregistered` | 尚未註冊，拒絕。 |
| `review_required` | 同一裝置、同一練習信封在五分鐘內第 3 次出現；不會自動加入黑名單。 |

CSV 只有 `event_id`、`device_id`、`status`、`occurred_at`、`updated_at`，不包含 `uid_envelope` 的任何欄位。

## 步驟 1：建立 Gateway 環境並固定版本

**目的：** 讓 Gateway 使用和教材相同的 Python 與套件版本。

執行位置：PowerShell／電腦

```powershell
cd rfid-project\gateway
uv init --bare --python 3.12
uv add fastapi==0.141.1 paho-mqtt==2.1.0 uvicorn==0.53.0
uv lock
uv run python --version
```

預期結果：最後一行顯示 `Python 3.12` 開頭的版本。`pyproject.toml` 記錄直接使用的套件版本，`uv.lock` 記錄連同相依套件在內的完整版本；兩個檔案都要留在 `gateway` 資料夾。

執行位置：PowerShell／電腦

```powershell
uv run python -c "import fastapi, paho.mqtt, uvicorn; print(fastapi.__version__); print(paho.mqtt.__version__); print(uvicorn.__version__)"
```

預期結果：依序顯示 `0.141.1`、`2.1.0`、`0.53.0`。

## 步驟 2：準備不公開的 MQTT 設定

**目的：** 把連線帳密與程式分開保存。

執行位置：文字編輯器／電腦
檔案：`rfid-project/gateway/.gitignore`

```text
mqtt_settings.py
records/
__pycache__/
```

執行位置：Python／電腦
檔案：`rfid-project/gateway/mqtt_settings.py`

```python
MQTT_HOST = "你的 HiveMQ Cloud 位址"
MQTT_PORT = 8883
MQTT_TOPIC = "你的完整測試 topic"
MQTT_USERNAME = "Gateway 專用帳號"
MQTT_PASSWORD = "Gateway 專用密碼"
```

`MQTT_TOPIC` 必須和第 2 章 ESP32 程式完全相同。預期結果：設定檔位於 `gateway` 資料夾，且 `.gitignore` 已列出它。帳密若不再使用，應在服務端刪除或重建。

## 步驟 3：建立 Gateway 程式

**目的：** 驗證資料契約、套用名單規則、寫入最小 CSV 紀錄並回覆 MQTT。

執行位置：Python／電腦
檔案：`rfid-project/gateway/gateway.py`

```python
import csv
from datetime import datetime, timedelta, timezone
import json
from pathlib import Path
from threading import Lock

from fastapi import FastAPI
import paho.mqtt.client as mqtt
import uvicorn


# 事件、信封與回覆各自只能使用這些欄位。
EVENT_FIELDS: set[str] = {"event_id", "device_id", "occurred_at", "uid_envelope"}
ENVELOPE_FIELDS: set[str] = {"key_id", "nonce", "ciphertext", "tag"}
REPLY_FIELDS: set[str] = {"event_id", "status", "updated_at"}
LOG_FIELDS: list[str] = ["event_id", "device_id", "status", "occurred_at", "updated_at"]

# 這些是可公開的練習資料，不是實際卡片資料。
BLACKLIST: set[str] = {"practice-blocked"}
WHITELIST: set[str] = {"practice-allowed"}
REVIEW_WINDOW: timedelta = timedelta(minutes=5)

LOG_PATH: Path = Path("records") / "rfid_gateway_log.csv"
MQTT_TOPIC: str = ""

state_lock: Lock = Lock()
processed_event_ids: set[str] = set()
recent_unregistered_attempts: list[dict[str, str]] = []
latest_status: dict[str, str] = {"status": "waiting"}

app = FastAPI(docs_url=None, redoc_url=None, openapi_url=None)


def utc_now() -> datetime:
    return datetime.now(timezone.utc)


def utc_text(value: datetime) -> str:
    return value.isoformat()


def read_text_field(value: object, name: str) -> str:
    if not isinstance(value, str) or not value:
        raise ValueError(f"{name} 必須是非空白文字")
    return value


def validate_event(value: object) -> dict[str, object]:
    if not isinstance(value, dict) or set(value) != EVENT_FIELDS:
        raise ValueError("事件欄位不符合資料契約")

    event_id: str = read_text_field(value["event_id"], "event_id")
    device_id: str = read_text_field(value["device_id"], "device_id")
    occurred_at: str = read_text_field(value["occurred_at"], "occurred_at")
    envelope: object = value["uid_envelope"]

    if not isinstance(envelope, dict) or set(envelope) != ENVELOPE_FIELDS:
        raise ValueError("uid_envelope 欄位不符合資料契約")

    for name in ENVELOPE_FIELDS:
        read_text_field(envelope[name], f"uid_envelope.{name}")

    parsed_time: datetime = datetime.fromisoformat(occurred_at)
    if parsed_time.tzinfo is None:
        raise ValueError("occurred_at 必須帶有時區")

    return {
        "event_id": event_id,
        "device_id": device_id,
        "occurred_at": occurred_at,
        "uid_envelope": envelope,
    }


def ciphertext_for(event: dict[str, object]) -> str:
    envelope: object = event["uid_envelope"]
    if not isinstance(envelope, dict):
        raise ValueError("uid_envelope 格式錯誤")
    return read_text_field(envelope["ciphertext"], "uid_envelope.ciphertext")


def remove_expired_attempts(now: datetime) -> None:
    cutoff: datetime = now - REVIEW_WINDOW
    recent_unregistered_attempts[:] = [
        attempt
        for attempt in recent_unregistered_attempts
        if datetime.fromisoformat(attempt["received_at"]) >= cutoff
    ]


def decide_status(event: dict[str, object], received_at: datetime) -> str:
    ciphertext: str = ciphertext_for(event)

    # 黑名單先判斷；白名單只在不屬於黑名單時才可通行。
    if ciphertext in BLACKLIST:
        return "blacklisted"
    if ciphertext in WHITELIST:
        return "whitelisted"

    remove_expired_attempts(received_at)
    device_id: str = read_text_field(event["device_id"], "device_id")
    previous_count: int = sum(
        attempt["device_id"] == device_id and attempt["ciphertext"] == ciphertext
        for attempt in recent_unregistered_attempts
    )

    recent_unregistered_attempts.append(
        {
            "device_id": device_id,
            "ciphertext": ciphertext,
            "received_at": utc_text(received_at),
        }
    )

    return "review_required" if previous_count + 1 >= 3 else "unregistered"


def reply_for(event: dict[str, object], status: str, updated_at: datetime) -> dict[str, str]:
    return {
        "event_id": read_text_field(event["event_id"], "event_id"),
        "status": status,
        "updated_at": utc_text(updated_at),
    }


def append_log(event: dict[str, object], reply: dict[str, str]) -> bool:
    # CSV 只保存查詢紀錄需要的欄位，不保存信封內容。
    row: dict[str, str] = {
        "event_id": reply["event_id"],
        "device_id": read_text_field(event["device_id"], "device_id"),
        "status": reply["status"],
        "occurred_at": read_text_field(event["occurred_at"], "occurred_at"),
        "updated_at": reply["updated_at"],
    }

    try:
        LOG_PATH.parent.mkdir(parents=True, exist_ok=True)
        with LOG_PATH.open("a", encoding="utf-8", newline="") as file:
            writer = csv.DictWriter(file, fieldnames=LOG_FIELDS)
            if file.tell() == 0:
                writer.writeheader()
            writer.writerow(row)
        return True
    except OSError:
        return False


def handle_payload(payload: bytes, received_at: datetime) -> dict[str, str] | None:
    received_value: object = json.loads(payload.decode("utf-8"))

    # Gateway 也會收到自己發布的回覆；那不是新的讀卡事件。
    if isinstance(received_value, dict) and set(received_value) == REPLY_FIELDS:
        return None

    event: dict[str, object] = validate_event(received_value)
    event_id: str = read_text_field(event["event_id"], "event_id")

    with state_lock:
        if event_id in processed_event_ids:
            return None

        status: str = decide_status(event, received_at)
        reply: dict[str, str] = reply_for(event, status, received_at)

        if not append_log(event, reply):
            raise OSError("讀卡紀錄無法寫入")

        processed_event_ids.add(event_id)

        global latest_status
        latest_status = reply
        return reply


@app.get("/status")
def read_status() -> dict[str, str]:
    with state_lock:
        return latest_status.copy()


def on_connect(
    client: mqtt.Client,
    userdata: object,
    flags: object,
    reason_code: object,
    properties: object,
) -> None:
    if getattr(reason_code, "is_failure", False):
        print("MQTT 連線未建立。")
        return

    client.subscribe(MQTT_TOPIC, qos=0)
    print("Gateway 已訂閱測試 topic。")


def on_message(client: mqtt.Client, userdata: object, message: mqtt.MQTTMessage) -> None:
    try:
        reply: dict[str, str] | None = handle_payload(message.payload, utc_now())
    except (UnicodeDecodeError, json.JSONDecodeError, ValueError, OSError) as error:
        print(f"收到不能處理的 MQTT 訊息：{error}")
        return

    if reply is None:
        print("Gateway 回覆或重複事件已略過。")
        return

    client.publish(MQTT_TOPIC, json.dumps(reply), qos=0, retain=False)
    print(f"已記錄 {reply['event_id']}，狀態：{reply['status']}")


def main() -> None:
    from mqtt_settings import MQTT_HOST, MQTT_PASSWORD, MQTT_PORT, MQTT_TOPIC as configured_topic, MQTT_USERNAME

    global MQTT_TOPIC
    MQTT_TOPIC = configured_topic

    client = mqtt.Client(
        mqtt.CallbackAPIVersion.VERSION2,
        client_id="rfid-gateway-test-01",
        reconnect_on_failure=False,
    )
    client.username_pw_set(MQTT_USERNAME, MQTT_PASSWORD)
    client.tls_set()
    client.on_connect = on_connect
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

預期結果：檔案已儲存，且 `BLACKLIST` 與 `WHITELIST` 只有練習用內容。兩份名單相同時，程式會先得到 `blacklisted`。

## 步驟 4：離線確認資料規則

**目的：** 在連線前確認名單與五分鐘規則符合預期。

執行位置：Python／電腦
檔案：`rfid-project/gateway/test_gateway.py`

```python
from datetime import datetime, timedelta, timezone
import json
from pathlib import Path
from tempfile import TemporaryDirectory
import unittest

import gateway


def make_event(event_id: str, ciphertext: str) -> bytes:
    event: dict[str, object] = {
        "event_id": event_id,
        "device_id": "demo-esp32-01",
        "occurred_at": "2030-01-02T03:04:05+00:00",
        "uid_envelope": {
            "key_id": "test-key",
            "nonce": "fake-nonce",
            "ciphertext": ciphertext,
            "tag": "fake-tag",
        },
    }
    return json.dumps(event).encode("utf-8")


class GatewayRuleTests(unittest.TestCase):
    def setUp(self) -> None:
        gateway.processed_event_ids.clear()
        gateway.recent_unregistered_attempts.clear()
        self.folder = TemporaryDirectory()
        gateway.LOG_PATH = Path(self.folder.name) / "records.csv"
        self.now = datetime(2030, 1, 2, 3, 5, tzinfo=timezone.utc)

    def tearDown(self) -> None:
        self.folder.cleanup()

    def status_for(self, event_id: str, ciphertext: str, seconds: int = 0) -> str:
        reply = gateway.handle_payload(make_event(event_id, ciphertext), self.now + timedelta(seconds=seconds))
        self.assertIsNotNone(reply)
        return reply["status"]

    def test_rules(self) -> None:
        self.assertEqual(self.status_for("evt-01", "practice-blocked"), "blacklisted")

        # 讓同一項目同時出現在兩份名單，確認黑名單仍優先。
        gateway.BLACKLIST.add("practice-allowed")
        self.assertEqual(self.status_for("evt-02", "practice-allowed"), "blacklisted")
        gateway.BLACKLIST.remove("practice-allowed")

        self.assertEqual(self.status_for("evt-03", "practice-allowed"), "whitelisted")
        self.assertEqual(self.status_for("evt-04", "practice-new"), "unregistered")
        self.assertEqual(self.status_for("evt-05", "practice-new", 10), "unregistered")
        self.assertEqual(self.status_for("evt-06", "practice-new", 20), "review_required")
        self.assertEqual(self.status_for("evt-07", "practice-later"), "unregistered")
        self.assertEqual(self.status_for("evt-08", "practice-later", 301), "unregistered")


if __name__ == "__main__":
    unittest.main(verbosity=2)
```

執行位置：PowerShell／電腦

```powershell
uv run python -m unittest -v test_gateway.py
```

預期結果：顯示 `Ran 1 test` 與 `OK`。這個測試會使用暫存位置寫入 CSV，結束時會自行清除。

## 步驟 5：啟動 Gateway 並查看結果

**目的：** 讓第 2 章 ESP32 發出的練習信封由 Gateway 處理。

執行位置：PowerShell／電腦

```powershell
uv run python gateway.py
```

預期結果：終端機先顯示 `Gateway 已訂閱測試 topic。`。第 2 章 ESP32 發送事件後，會顯示類似 `已記錄 evt-...，狀態：whitelisted`。

保持 Gateway 執行，在另一個 PowerShell 視窗查詢最近狀態與 CSV：

執行位置：PowerShell／電腦

```powershell
Invoke-RestMethod http://127.0.0.1:8000/status | ConvertTo-Json
Import-Csv records\rfid_gateway_log.csv | Format-Table
```

預期結果：`/status` 回傳最近一次的 `event_id`、`status`、`updated_at`；CSV 表格不會出現 `ciphertext`、`nonce`、`tag` 或 `key_id`。

## 完成時，應能確認

- `uv run python --version` 顯示 Python `3.12`，且 `pyproject.toml`、`uv.lock` 都在 `gateway` 資料夾。
- 離線測試顯示 `OK`。
- `practice-blocked` 得到 `blacklisted`，`practice-allowed` 得到 `whitelisted`。
- 同一 `device_id` 與同一練習 `ciphertext` 在五分鐘內第 3 次出現時得到 `review_required`，沒有被寫進黑名單。
- `records/rfid_gateway_log.csv` 不含信封內容，API 仍只在 `127.0.0.1` 可查詢。

## 常見問題

### `uv run python --version` 不是 Python 3.12

先確認你位於 `rfid-project/gateway`，並檢查 `.python-version` 是否為 `3.12`。修正後重新執行 `uv lock`，再執行版本確認指令。

### 顯示 `ModuleNotFoundError`

先執行步驟 1 的 `uv add ...` 與 `uv lock`。不要改用系統 Python 直接執行 `python gateway.py`，應使用 `uv run python gateway.py`。

### Gateway 沒有訂閱 topic

先檢查 `mqtt_settings.py` 的主機位址、連接埠、帳密與 topic 是否正確。確認 Gateway 帳號有訂閱該 topic 的權限，ESP32 帳號有發布權限；不要為了排除問題而改成共用帳密或萬用 topic。

### CSV 沒有資料

先確認 Gateway 終端機有顯示 `已記錄`，並檢查 `records` 資料夾是否可寫入。若終端機顯示 `收到不能處理的 MQTT 訊息`，先比對第 2 章事件是否仍含四個頂層欄位與四個信封欄位。

## 重點整理

- Gateway 是名單判斷與紀錄集中處理的位置；MQTT 中介服務不負責這些規則。
- 黑名單優先於白名單；未註冊卡拒絕，第 3 次近距離重複事件才標記 `review_required`。
- 信封內容只在記憶體中用於判斷，CSV 不保存它。
- `GET /status` 僅供本機查看。App 的 HTTPS API、ngrok、名單管理與通知屬於另一個安全範圍，不在本章操作。

## 下一步

想了解程式各段如何接力，可閱讀[Gateway 程式導讀](gateway程式導讀.md)與[FastAPI 概念介紹](fastapi概念介紹.md)。
