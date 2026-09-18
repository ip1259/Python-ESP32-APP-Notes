# 選做專題第 0 章：準備、安全與資料契約

先建立整個 RFID 專題需要的工作區、安全界線與離線資料規則，再進入讀卡與訊息傳遞。

## 你會學到什麼

- 用假 `uid_envelope` 表示卡片資料容器，不使用真實 UID。
- 確認黑名單優先、白名單可通過、未註冊卡拒絕。
- 確認同一張未註冊假卡、同一裝置、五分鐘內第 3 次會標示 `review_required`，但不自動加入黑名單。
- 說出 ESP32、MQTT、Gateway、LCD 與 App 在後續章節各自負責什麼。

## 開始前

- 需要 Python、`uv`、Arduino IDE、NodeMCU-32S、RC522 與 LCD1602A；本頁不進行接線或網路連線。
- 所有字串都是假資料。不要輸入 UID、帳密、網址、token 或個人資料。

!!! warning "這不是門禁控制"

    本頁只產生練習用狀態，不能開門、控制繼電器或判定真實身分。

## 成功的樣子

執行後顯示 `7 個資料契約案例全部通過`。

## 專題環境與資料流

先在自己可控制的位置建立專題工作區。後續章節的程式、秘密設定與練習紀錄都放在這裡；機敏資訊不應公開傳送、發布或分享給他人。

執行位置：PowerShell／電腦

```powershell
mkdir rfid-project
cd rfid-project
mkdir gateway secrets records
```

預期結果：目前資料夾內有 `gateway`、`secrets`、`records` 三個資料夾。`secrets` 只留在自己的電腦，未來放 Wi-Fi 或 MQTT 測試資料時不可上傳、截圖或分享。

## 建立 Python 專題環境

**目的：** 讓後續 Gateway、離線檢查與測試都使用 RFID 專題自己的 Python 環境，不和其他練習資料夾混在一起。

保持在 `rfid-project` 資料夾後執行：

執行位置：PowerShell／電腦

```powershell
uv init --bare --python 3.12
uv venv
```

`uv init --bare --python 3.12` 會建立這個專題的 `pyproject.toml`，並固定使用 Python `3.12`；`uv venv` 會建立 `.venv` 虛擬環境資料夾。虛擬環境像專題專用的工具箱，後續 Gateway 需要的套件只安裝在這裡，不影響其他 Python 練習。

預期結果：目前資料夾至少有下列內容：

```text
rfid-project/
├── .venv/
├── gateway/
├── records/
├── secrets/
├── .python-version
└── pyproject.toml
```

接著確認環境可用：

執行位置：PowerShell／電腦

```powershell
uv run python --version
```

預期結果：顯示 `Python 3.12.x`，且不出現找不到 `uv` 或 Python 的錯誤。第 0 章不安裝 MQTT、FastAPI 或其他套件；等到 Gateway 章節才依實際需求以 `uv add` 加入。`uv add` 會更新 `pyproject.toml` 與 `uv.lock`，兩個檔案要和 Gateway 程式一起保留，讓其他電腦能建立相同套件版本。

## Arduino 工具版本

後續硬體章節使用相同的 Arduino IDE 專案條件。固定版本如下：

| 元件 | 固定版本 | 使用章節 |
| --- | --- | --- |
| ESP32 開發板核心 | `esp32 by Espressif Systems 3.3.11` | 第 1 章起 |
| `MFRC522`（GitHub Community） | `1.4.12` | 第 1 章起 |
| `PubSubClient`（Nick O'Leary） | `2.8.0` | 第 2 章起 |
| `LiquidCrystal I2C`（Frank de Brabander） | `1.1.2` | 第 4 章 |

進入後續硬體章節時，請依上表與各章列出的開發板、接線與程式操作，不要自行以其他版本替換這些函式庫。

```text
練習卡 → RC522／ESP32 → MQTT → Gateway → MQTT → LCD
                                      ↑
                              App／ngrok HTTPS API
```

| 元件 | 後續責任 | 本章是否操作 |
| --- | --- | --- |
| ESP32／RC522 | 讀取受控練習卡並送出事件 | 否 |
| MQTT | 傳遞 ESP32 與 Gateway 事件 | 否 |
| Gateway | 判斷名單、保留最小紀錄、回覆狀態 | 只用離線假資料 |
| LCD | 顯示固定狀態 | 否 |
| App／ngrok | 管理名單、看紀錄與通知 | 否 |

!!! danger "立即停止的情況"

    不明來源卡片、真實 UID、帳密、公開網址、私有網路位址或任何門鎖／繼電器控制出現時，停止操作。本專題只處理受控練習資料與狀態顯示。

## 資料規則

| 資料 | 規則 |
| --- | --- |
| `event_id` | 每次事件的取件號碼；不可重複。 |
| `device_id` | 假裝置代號；不同裝置分開計數。 |
| `occurred_at` | 必須是含時區的時間。 |
| `uid_envelope` | 只有 `key_id`、`nonce`、`ciphertext`、`tag` 四個假欄位。 |

## 步驟 1：建立程式

**目的：** 用固定資料驗證名單與計數規則。

執行位置：Python／電腦
檔案：`rfid_contract.py`

```python
from collections import defaultdict, deque
from datetime import datetime, timedelta

# 同一張未註冊假卡的計數時間窗。
WINDOW = timedelta(minutes=5)


def decide(event: dict[str, object], black: set[str], white: set[str], seen: set[str], attempts: dict[tuple[str, str], deque[datetime]]) -> str:
    # 先確認事件只有本章約定的必要欄位。
    required = {"event_id", "device_id", "occurred_at", "uid_envelope"}
    if set(event) != required or event["event_id"] in seen:
        raise ValueError("事件資料無法使用")
    envelope = event["uid_envelope"]
    if not isinstance(envelope, dict) or set(envelope) != {"key_id", "nonce", "ciphertext", "tag"}:
        raise ValueError("信封資料無法使用")
    when = datetime.fromisoformat(str(event["occurred_at"]))
    if when.tzinfo is None:
        raise ValueError("時間必須包含時區")
    # 假 ciphertext 只作為練習卡代號，沒有真實 UID。
    card = str(envelope["ciphertext"])
    seen.add(str(event["event_id"]))
    # 黑名單先判斷，避免同時在白名單時得到相反結果。
    if card in black:
        return "blacklisted"
    if card in white:
        return "whitelisted"

    # 未註冊卡以「裝置＋假卡代號」各自計數。
    key = (str(event["device_id"]), card)
    while attempts[key] and when - attempts[key][0] >= WINDOW:
        attempts[key].popleft()
    attempts[key].append(when)
    return "review_required" if len(attempts[key]) >= 3 else "not_registered"


def event(number: int, minute: int) -> dict[str, object]:
    # 產生固定時間與固定假信封，讓案例每次都可重複執行。
    return {"event_id": f"evt-{number:03}", "device_id": "demo-esp32-01", "occurred_at": f"2030-01-01T08:{minute:02}:00+00:00", "uid_envelope": {"key_id": "test-key", "nonce": "fake-nonce", "ciphertext": "practice-allowed", "tag": "fake-tag"}}


def main() -> None:
    # 每次執行都從空白名單與計數器開始。
    black: set[str] = set()
    white: set[str] = set()
    seen: set[str] = set()
    attempts: dict[tuple[str, str], deque[datetime]] = defaultdict(deque)

    assert decide(event(1, 0), black, white, seen, attempts) == "not_registered"
    assert decide(event(2, 2), black, white, seen, attempts) == "not_registered"
    assert decide(event(3, 4), black, white, seen, attempts) == "review_required"

    # 加入黑名單後，同一筆假資料必須優先被拒絕。
    black.add("practice-allowed")
    assert decide(event(4, 6), black, white, seen, attempts) == "blacklisted"

    # 移除黑名單後，白名單才可以讓假資料通過。
    black.clear()
    white.add("practice-allowed")
    assert decide(event(5, 7), black, white, seen, attempts) == "whitelisted"

    try:
        decide(event(5, 8), black, white, seen, attempts)
        raise AssertionError("重複事件不應通過")
    except ValueError:
        pass
    print("7 個資料契約案例全部通過")


if __name__ == "__main__":
    main()
```

預期結果：程式中只有假信封和假裝置代號，沒有 UID、帳密或網路設定。

## 步驟 2：建立固定案例檢查程式

**目的：** 把每個可觀察結果放在另一個檔案，之後調整契約時能重複確認規則。

執行位置：Python／電腦
檔案：`check_contract.py`

```python
from collections import defaultdict, deque
from datetime import datetime

from rfid_contract import decide, event


def main() -> None:
    # 這個檔案只排列案例；實際判斷仍由 rfid_contract.py 負責。
    black: set[str] = set()
    white: set[str] = set()
    seen: set[str] = set()
    attempts: dict[tuple[str, str], deque[datetime]] = defaultdict(deque)

    # 第 1、2、3 次未註冊，依序確認拒絕與待檢視結果。
    assert decide(event(1, 0), black, white, seen, attempts) == "not_registered"
    assert decide(event(2, 2), black, white, seen, attempts) == "not_registered"
    assert decide(event(3, 4), black, white, seen, attempts) == "review_required"
    assert black == set()

    # 接著確認黑名單優先，再確認只剩白名單時可通過。
    black.add("practice-allowed")
    assert decide(event(4, 6), black, white, seen, attempts) == "blacklisted"

    black.clear()
    white.add("practice-allowed")
    assert decide(event(5, 7), black, white, seen, attempts) == "whitelisted"
    print("5 個資料契約案例全部通過")


if __name__ == "__main__":
    main()
```

預期結果：兩個 Python 檔案都在 `rfid-project` 資料夾。`rfid_contract.py` 負責規則，`check_contract.py` 負責檢查結果。

## 步驟 3：執行案例

執行位置：PowerShell／電腦

```powershell
uv run python check_contract.py
```

預期結果：顯示 `5 個資料契約案例全部通過`。

## 完成時，應能確認

- 第 3 次未註冊只會得到 `review_required`，不會自動改變黑名單。
- 黑名單優先於白名單。
- 重複事件被拒絕。

## 常見問題

### 出現 `AssertionError`

確認程式完整貼上，並保留固定時間與假信封。不要改成真實卡片資料。

### 為什麼沒有讀卡器或 App？

第 0 章先固定資料規則。讀卡、MQTT、Gateway、LCD 與 App 會在後續章節各自驗證。

## 重點整理

- 假信封只用來練習資料外形，不代表加密已完成。
- 資料無法可靠判斷時，應拒絕或標示待檢視，不保留舊結果。

## 下一步

接著閱讀[第 1 章：RC522 RFID 與 SPI](../第1章/index.md)。
