# 選做專題第 0 章：準備、安全與資料契約

先用離線的虛構資料定義讀卡事件與安全狀態，讓後續章節不會把完整卡片識別值或控制指令帶進資料流。

## 你會學到什麼

- 說明資料契約（data contract，資料交換規則）如何讓不同程式使用相同欄位與狀態。
- 用遮蔽的虛構識別值、事件代號與含時區時間建立一筆可接受事件。
- 用 Python 純函式產生授權模擬、未授權與失敗狀態，且不連接硬體或網路。
- 判斷啟動、資料過期與格式錯誤時，為何必須顯示未知或失敗，而不能沿用上一筆結果。

## 開始前

- 本章是完成主線後的選做專題，不影響[整合專題：完成環境看板](08-整合專題.md)的核心成果。
- 需要 Python 與 `uv`；本章只使用 Python 標準函式庫，不需 RC522、MQTT、Broker、App、網路或 LCD。
- 請只使用本頁提供的虛構資料。不要輸入、保存、截圖或提交真實卡片 UID、帳密、token、內網位址或個人資料。

!!! warning "本章不是門禁控制"
    本章只模擬讀卡狀態管理與顯示文字。不連接繼電器、電磁鎖、馬達或其他實體設備，也不能用來判定真實身分或控制門鎖。

## 成功的樣子

執行檢查程式後，終端機會顯示：

```text
9 個資料契約案例全部通過
```

這 9 個案例包含等待事件、授權模擬、未授權模擬、讀卡失敗、裝置離線、訊息服務斷線、Gateway 停止、資料過期與格式錯誤。它只證明 Python 的固定規則符合預期，不證明 RC522、網路、Broker 或 App 已可使用。

## 為什麼要先定資料契約

後續專題會經過讀卡端、訊息服務、Python Gateway、LCD 與手機查詢。資料契約先約定「傳什麼」和「遇到問題時怎麼表示」，每一段才不會自行猜測資料格式。

```text
虛構讀卡事件    →   狀態判斷     →     最新狀態摘要
      │               │                  │
  不含完整 UID    授權模擬／失敗     不含帳密與內部設定
```

本章還沒有上述裝置與服務；箭頭中的資料都在同一個 Python 程式內，以固定字典模擬。

## 資料規則速覽

| 資料 | 固定規則 | 用意 |
| --- | --- | --- |
| `event_id` | 使用 `evt-001` 這類 3 至 20 字元的小寫英文字母、數字與連字號 | 關聯測試事件與結果；不使用卡片資料、帳密或位址。 |
| `card_id_masked` | 只接受虛構格式 `CARD-****-NN`，例如 `CARD-****-42` | 不接收完整 UID，也不在程式中把完整 UID 轉為遮蔽值。 |
| `device_id` | 使用 `demo-esp32-01` 這類非個資的教學代號 | 不使用私有 IP、MAC 位址或個人名稱。 |
| `occurred_at`、`updated_at` | 使用含時區的 ISO 8601 時間，例如 `2026-09-15T09:00:00+08:00` | 時間可比較，也不混淆來源時區。 |
| `authorized` | 只能是 `True`、`False` 或 `None` | `None` 表示目前無法可靠判斷，不是「否」。 |

本章以 `CARD-****-42` 作為唯一虛構的教學白名單。這只是固定資料的程式判斷，不是身分驗證，也不是可用於真實門禁的白名單。

## 狀態規則

| 情境 | `status` | `authorized` | 顯示方向 |
| --- | --- | --- | --- |
| 剛啟動 | `waiting` | `None` | 等待教學事件 |
| 授權練習卡 | `authorized` | `True` | 已讀取授權練習卡 |
| 未授權練習卡 | `unauthorized` | `False` | 練習卡未在教學白名單 |
| 讀卡失敗 | `read_failed` | `None` | 無法完成讀卡 |
| 裝置離線 | `device_offline` | `None` | 裝置目前無回應 |
| 訊息服務斷線 | `broker_disconnected` | `None` | 訊息服務未連線 |
| Gateway 停止 | `gateway_stopped` | `None` | 狀態服務未執行 |
| 最新資料超過 60 秒 | `unknown` | `None` | 目前狀態未知 |

60 秒是本章用來測試「舊資料不可當成現在結果」的固定條件。真正使用裝置與訊息服務時，資料多久算過期，必須依實際傳輸與顯示行為另外決定。

## 步驟 1：建立練習資料夾

**目的：** 將本章程式放在可控制的資料夾中，不碰到既有專題檔案。

執行位置：PowerShell／電腦

```powershell
mkdir rfid-ch0-contract
cd rfid-ch0-contract
```

預期結果：目前位置會進入新建立的 `rfid-ch0-contract` 資料夾。

## 步驟 2：建立資料契約程式

**目的：** 檢查事件格式，並把可接受事件轉成安全的狀態摘要。

執行位置：Python／電腦
檔案：`rfid_contract.py`

```python
from datetime import datetime
import re


ALLOWED_EVENT_TYPES = {
    "card_read",
    "read_failed",
    "device_offline",
    "broker_disconnected",
    "gateway_stopped",
}
ALLOWED_CARD_IDS = {"CARD-****-42"}
EVENT_ID_PATTERN = re.compile(r"^[a-z0-9-]{3,20}$")
CARD_ID_PATTERN = re.compile(r"^CARD-\*\*\*\*-\d{2}$")
DEVICE_ID_PATTERN = re.compile(r"^[a-z][a-z0-9-]{2,19}$")
STATUS_MESSAGES = {
    "read_failed": "無法完成讀卡",
    "device_offline": "裝置目前無回應",
    "broker_disconnected": "訊息服務未連線",
    "gateway_stopped": "狀態服務未執行",
}


def parse_time(value: object) -> datetime:
    if not isinstance(value, str):
        raise ValueError("時間必須是文字")

    try:
        parsed = datetime.fromisoformat(value)
    except ValueError as error:
        raise ValueError("時間格式無法使用") from error

    if parsed.tzinfo is None:
        raise ValueError("時間必須包含時區")
    return parsed


def validate_event(event: dict[str, object]) -> dict[str, object]:
    required = {"event_type", "card_id_masked", "device_id", "occurred_at", "event_id"}
    if set(event) != required:
        raise ValueError("欄位不完整或含有未定義欄位")

    event_type = event["event_type"]
    event_id = event["event_id"]
    device_id = event["device_id"]
    card_id = event["card_id_masked"]

    if event_type not in ALLOWED_EVENT_TYPES:
        raise ValueError("事件種類無法使用")
    if not isinstance(event_id, str) or not EVENT_ID_PATTERN.fullmatch(event_id):
        raise ValueError("事件代號無法使用")
    if not isinstance(device_id, str) or not DEVICE_ID_PATTERN.fullmatch(device_id):
        raise ValueError("裝置代號無法使用")
    if event_type == "card_read":
        if not isinstance(card_id, str) or not CARD_ID_PATTERN.fullmatch(card_id):
            raise ValueError("遮蔽識別值無法使用")
    elif card_id is not None:
        raise ValueError("非讀卡事件不可帶識別值")

    parse_time(event["occurred_at"])
    return event


def make_status(event: dict[str, object], updated_at: str) -> dict[str, object]:
    checked = validate_event(event)
    event_type = str(checked["event_type"])
    card_id = checked["card_id_masked"]

    if event_type == "card_read":
        authorized = card_id in ALLOWED_CARD_IDS
        return {
            "event_id": checked["event_id"],
            "status": "authorized" if authorized else "unauthorized",
            "authorized": authorized,
            "display_text": "已讀取授權練習卡" if authorized else "練習卡未在教學白名單",
            "updated_at": updated_at,
        }

    return {
        "event_id": checked["event_id"],
        "status": event_type,
        "authorized": None,
        "display_text": STATUS_MESSAGES[event_type],
        "updated_at": updated_at,
    }


def process_event(event: dict[str, object], updated_at: str) -> dict[str, object]:
    try:
        parse_time(updated_at)
        return {"accepted": True, "status_data": make_status(event, updated_at)}
    except ValueError:
        return {
            "accepted": False,
            "reason_code": "invalid_event",
            "display_text": "資料格式無法使用",
        }


def latest_status(
    status_data: dict[str, object] | None,
    now: str,
    max_age_seconds: int = 60,
) -> dict[str, object]:
    current_time = parse_time(now)
    if status_data is None:
        return {
            "event_id": None,
            "status": "waiting",
            "authorized": None,
            "display_text": "等待教學事件",
            "updated_at": now,
        }

    updated_at = parse_time(status_data["updated_at"])
    if (current_time - updated_at).total_seconds() > max_age_seconds:
        return {
            "event_id": status_data["event_id"],
            "status": "unknown",
            "authorized": None,
            "display_text": "目前狀態未知",
            "updated_at": now,
        }
    return status_data
```

`validate_event()` 會檢查欄位、事件代號、裝置代號、遮蔽識別值與時間。`process_event()` 接住可預期的格式錯誤，只回傳固定的安全摘要，不把原始輸入或 Python 錯誤顯示出去。`latest_status()` 則將尚未收到事件或超過 60 秒的資料標記為未知。

## 步驟 3：建立固定案例檢查程式

**目的：** 用 9 個可重複的案例確認規則沒有把失敗或舊資料當成目前授權結果。

執行位置：Python／電腦
檔案：`check_contract.py`

```python
from rfid_contract import latest_status, process_event


NOW = "2026-09-15T09:00:30+08:00"


def event(event_type: str, card_id_masked: str | None, event_id: str) -> dict[str, object]:
    return {
        "event_type": event_type,
        "card_id_masked": card_id_masked,
        "device_id": "demo-esp32-01",
        "occurred_at": "2026-09-15T09:00:00+08:00",
        "event_id": event_id,
    }


def assert_status(result: dict[str, object], expected: str, authorized: bool | None) -> None:
    assert result["accepted"] is True
    status_data = result["status_data"]
    assert isinstance(status_data, dict)
    assert status_data["status"] == expected
    assert status_data["authorized"] is authorized


def main() -> None:
    assert latest_status(None, NOW)["status"] == "waiting"
    assert_status(process_event(event("card_read", "CARD-****-42", "evt-001"), NOW), "authorized", True)
    assert_status(process_event(event("card_read", "CARD-****-99", "evt-002"), NOW), "unauthorized", False)

    for index, event_type in enumerate(
        ["read_failed", "device_offline", "broker_disconnected", "gateway_stopped"],
        start=3,
    ):
        assert_status(process_event(event(event_type, None, f"evt-00{index}"), NOW), event_type, None)

    old_status = process_event(event("card_read", "CARD-****-42", "evt-007"), "2026-09-15T09:00:00+08:00")
    assert old_status["accepted"] is True
    stored = old_status["status_data"]
    assert isinstance(stored, dict)
    assert latest_status(stored, "2026-09-15T09:01:01+08:00")["status"] == "unknown"

    invalid = process_event(event("card_read", "12345678", "evt-008"), NOW)
    assert invalid == {
        "accepted": False,
        "reason_code": "invalid_event",
        "display_text": "資料格式無法使用",
    }
    print("9 個資料契約案例全部通過")


if __name__ == "__main__":
    main()
```

預期結果：儲存兩個檔案後，程式尚未開啟任何硬體或網路連線；所有輸入都在 `check_contract.py` 的固定字典中。

## 步驟 4：執行檢查

**目的：** 確認所有固定情境都符合資料契約。

執行位置：PowerShell／電腦

```powershell
uv run python check_contract.py
```

預期結果：顯示 `9 個資料契約案例全部通過`。若看到 `AssertionError`，先確認兩個檔案是否在同一資料夾，以及程式碼、固定時間和虛構值是否完整貼上。

## 安全界線與目前尚未做的事

- `CARD-****-42` 與 `CARD-****-99` 都是虛構值；不要改成真實 UID。
- 本章的「授權」是固定練習資料的比對結果，不代表真實身分或門禁權限。
- MQTT topic、Broker、FastAPI、公開網址、手機 App、RC522 與 LCD 接線都不在本章範圍內。請不要自行加入帳密、網路設定、真實卡片資料或控制設備的程式。
- 若資料格式錯誤，程式只顯示「資料格式無法使用」。不要為了除錯把原始卡片資料、帳密或內網資訊印出來。

## 完成時，應能確認

- 能說出資料契約至少包含欄位、格式、狀態與錯誤處理規則。
- 能執行 `uv run python check_contract.py`，並看到 9 個固定案例全部通過。
- 能解釋 `True`、`False` 與 `None` 在 `authorized` 欄位中的差別。
- 能指出資料超過 60 秒時為何要改為 `unknown`。
- 能確認程式沒有讀取真實 UID、開啟網路、控制硬體或產生門鎖指令。

## 常見問題

### 顯示 `ModuleNotFoundError: No module named 'rfid_contract'`

確認檔名是 `rfid_contract.py`，不是 `rfid_contract.py.txt`；兩個 Python 檔案也必須在同一個 `rfid-ch0-contract` 資料夾，再從該資料夾執行指令。

### 顯示「資料格式無法使用」或 `AssertionError`

先檢查 `event_id` 是否像 `evt-001`，遮蔽值是否像 `CARD-****-42`，時間是否包含 `+08:00`。不要以真實卡片資料取代範例值。

### 為什麼沒有看到 RC522、MQTT 或手機畫面？

這是第 0 章的設計目的：先在離線環境確認資料規則。硬體、訊息服務與手機查詢需要額外設備、帳號或網路條件，不屬於這一頁的操作。

### 為什麼舊的授權結果不能繼續顯示？

舊結果只能說明先前的虛構事件，不代表現在仍可靠。超過 60 秒後，程式改為 `unknown`，避免把以前的 `True` 誤當成目前狀態。

## 重點整理

- 資料契約先固定欄位、格式與狀態，後續不同程式才可安全交換必要資料。
- 虛構遮蔽值、固定事件代號和含時區時間，能讓測試可重複且不含敏感資料。
- `None`、`waiting` 與 `unknown` 都是在誠實表達「目前無法可靠判斷」。
- 本章只驗證純 Python 規則；它不代表硬體、網路、Broker、Gateway 或 App 已完成。

## 下一步

若想先了解讀卡硬體在專題中的位置，可閱讀[即將推出：選做專題第 1 章：RC522 RFID 與 SPI](選做-RC522-RFID與SPI.md)。該頁目前是方向介紹，不包含可操作的接線或程式。
