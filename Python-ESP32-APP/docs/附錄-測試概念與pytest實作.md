# 延伸選讀：測試概念與 pytest 實作

測試先是一套跨語言的思考方法，再依程式語言選工具；這一頁用 Python 的 `pytest` 將可重複的期待寫成可執行檢查。

## 你會學到什麼

- 分辨單元、整合、端對端／系統測試各自回答的問題。
- 用 Arrange、Act、Assert 設計正常、邊界與錯誤案例。
- 以 Python 的 `pytest`、`assert`、`raises()` 測試純函式，並用參數化測試把多組固定資料套用到同一個規則。
- 判讀通過與失敗結果，並知道單元測試無法取代 ESP32 的實機觀察。

## 開始前

- 已了解 Python 函式、字典、條件判斷與例外。
- 本頁是延伸選讀，不連接 ESP32、COM 埠、網路或真實檔案；資料全部寫在程式中。
- 本頁以 `pytest 9.1.1` 驗證。其他版本先執行本頁範例，再查閱[pytest 官方文件](https://docs.pytest.org/)。

## 測試在整體品質中的位置

測試不是某一種語言的專利。JavaScript 可用 Jest、Vitest，Java 可用 JUnit，C# 可用 xUnit 或 NUnit；工具名稱不同，但「準備資料、執行行為、檢查結果」的思考相同。

| 活動或層級 | 主要問題 | 典型依賴 | 本課例子 |
| --- | --- | --- | --- |
| 需求釐清與人工檢查 | 做的是不是使用者需要的事？ | 人、規格、畫面 | 確認儀表板顯示內容可理解 |
| 單元測試 | 一個小函式的輸入／輸出是否符合規則？ | 盡量不需要外部檔案、裝置或服務 | JSON 字典能否轉成合法資料列 |
| 整合測試 | 多個元件接起來是否相容？ | 可控制的檔案、序列埠或服務 | Python 能否讀到 ESP32 JSON |
| 端對端／系統測試 | 使用者完整流程是否可完成？ | 接近真實環境 | 按 Gradio 按鈕後看到資料並控制 LED |
| 監控與事故回顧 | 上線後是否持續可靠？ | 真實使用環境 | 正式產品才需要的日誌、告警與追蹤 |

!!! warning "單元測試通過不等於整個系統都正常"
    本頁只驗證純 Python 資料規則。它不能證明 DHT11 接線正確、ESP32 韌體可用、COM 埠沒有被占用、網路穩定或 Gradio 按鈕真的控制硬體。這些仍要以整合測試與主線實機觀察確認。

## 業界常見測試活動：各自回答不同問題

以下名稱常會出現在需求、開發、交付或維護討論中。它們可以搭配使用，不是從「簡單」到「完整」只能選一種的階級。

| 活動 | 想確認什麼 | 常見時機 | 在 AIoT 專題的例子 | 不能單獨證明什麼 |
| --- | --- | --- | --- | --- |
| 回歸測試（regression testing） | 修改後，原本可用的行為有沒有被意外破壞 | 修正 bug、重構、更新套件或改規則後 | 修改 JSON 轉換後重跑既有正常、邊界與錯誤案例 | 新功能一定符合所有使用者需求 |
| 冒煙測試（smoke testing） | 最核心路徑是否能快速啟動與運作 | 新版本剛建立、部署或開始使用前 | ESP32 可燒錄、Python 可啟動、Gradio 可在 localhost 開啟 | 所有細節、錯誤情境與長時間可靠性 |
| 探索式測試（exploratory testing） | 人在操作中能否發現規格未列出的問題 | 介面完成後、整合前後 | 故意連按按鈕、改變輸入順序、觀察錯誤提示是否清楚 | 每次都能完全重現的自動化證據 |
| 成果條件測試（acceptance testing） | 成果是否符合事先列出的條件 | 準備開始使用前 | 依主線的「完成時，應能確認」檢查感測、CSV、圖表與儀表板成果 | 內部每個函式與錯誤分支都已測過 |
| 效能／負載測試 | 在指定數量、速度或時間下是否仍符合目標 | 有明確效能需求後 | 長時間接收資料時是否遺漏、畫面是否仍可操作 | 功能邏輯或安全性正確 |
| 安全測試 | 權限、輸入、資料與攻擊面是否符合安全要求 | 有帳號、公開服務或敏感資料前 | 公開控制設備前檢查權限與資料暴露 | 一般功能是否符合教學需求 |

## 回歸測試：改動後重跑你的安全網

回歸測試不是某個 Python 指令，而是一個工作習慣：當程式、規則、套件或設定改動後，重新執行與改動相關的既有案例，確認原本承諾的行為仍成立。

本頁的 `test_sensor_data.py` 就可當作最小回歸測試組。假設你修改 `normalize_sensor()` 的數字轉換或溫度範圍，先執行 `pytest -q`：

- 全部通過：代表目前 8 個已寫下的輸入／輸出規則仍符合，不代表沒有其他未寫的問題。
- 某個案例失敗：先確認需求是否改變、程式是否改壞，或測試預期是否真的過時；不要直接刪除失敗案例。
- 修正 bug：應保留能重現這個 bug 的案例，讓未來修改時可再發現同樣問題。

例如把預期濕度故意從 `62.0` 改成 `62.1`，pytest 會顯示預期／實際差異；這是在練習判讀失敗，不是合理的修正方式。看完後必須改回正確預期並重跑。

## 測試資料與測試替身

好的測試資料要小、可控制、可說明。正常資料確認主要流程，邊界資料確認上下限，錯誤資料確認拒絕方式；若資料來自真實人員或環境，應先匿名化並確認授權，不能把個資、密碼、token 或私有網路資料放進測試。

| 名稱 | 用途 | 例子 | 注意事項 |
| --- | --- | --- | --- |
| 固定測試資料 | 每次得到相同結果 | 本頁寫在字典中的溫濕度 | 不可誤當真實環境趨勢 |
| fake（簡化替代品） | 可運作但簡化的替代實作 | 記憶體中的假資料儲存區 | 行為可能和正式服務仍有差異 |
| stub（固定回覆） | 提供預先設定的回覆 | 固定回傳一筆成功 JSON | 不應拿來驗證複雜互動 |
| mock（呼叫檢查） | 檢查某個外部合作對象是否被預期方式呼叫 | 確認程式嘗試傳送命令 | 過度檢查實作細節會讓重構困難 |

本頁不用這些測試替身 API，因為純函式不需要其他檔案、裝置或服務。當未來測試 UART、時間、檔案或網路時，先判斷要實機整合測試、使用可控制的替身，還是根本不該把外部項目放進單元測試。

## 覆蓋率與持續整合（CI）是什麼

覆蓋率（coverage）是測試曾跑過多少程式碼、分支或條件的比例。它能協助找出「案例完全沒有碰到」的地方，但數字高不代表判斷正確、功能完整、介面好用或硬體可靠。不要只為了提高數字而加入沒有意義的測試。

持續整合（Continuous Integration，CI）是每次修改程式後，自動重新執行測試、格式檢查或建置指令的做法。CI 有助於及早發現原本可用的功能被改壞，但不會替你接線、量測感測器或保護公開服務。本頁不要求帳號或 CI 設定；目前在電腦上手動執行 `pytest -q`，已足以練習測試的基本流程。

## AIoT 最小測試策略對照

| 區域 | 優先驗證方式 | 可保存的證據 | 停止或轉交時機 |
| --- | --- | --- | --- |
| 資料轉換、範圍判斷 | 單元／回歸測試 | 測試案例與通過輸出 | 出現外部檔案、裝置或服務時，改用整合測試方式 |
| JSON、CSV、UART 資料流 | 受控整合與錯誤訊息檢查 | 固定輸入與輸出紀錄 | COM 埠衝突或格式不明時先停止並分流 |
| ESP32、DHT11、LED | 實機冒煙與操作觀察 | 燒錄結果、序列輸出、硬體行為 | 接線、電壓或輸出異常時斷電檢查 |
| Gradio 本機介面 | 探索操作與主線成果檢查 | localhost 畫面與回呼結果 | 不要改成公開網址或遠端控制 |
| 公開服務、帳密與控制設備 | 另立安全與環境規劃 | 核准範圍與安全檢查 | 條件未齊備時不實作 |

## 測試案例的共同結構

```text
Arrange（準備）→ Act（執行）→ Assert（斷言）
```

- Arrange：準備一筆可控制的輸入資料和預期結果。
- Act：只呼叫要測試的函式或行為。
- Assert：清楚檢查結果；不符合時讓測試失敗並留下差異。

一組有用的案例至少包含：正常資料、剛好在上下限的邊界資料，以及應被拒絕的錯誤資料。外部服務難以控制時，也可使用前表的固定回覆、簡化替代品或呼叫檢查等測試替身；它們的目的都是讓測試可重複，不是偽造產品已經能在真實環境工作。本頁先不使用替身 API。

## pytest 的核心功能速覽

| 功能 | 是什麼 | 最常見用法 | 結果或副作用 |
| --- | --- | --- | --- |
| `pytest` | Python 測試執行工具 | `pytest -q` 尋找並執行 `test_*.py` 與 `test_*()` | 顯示通過、失敗與錯誤摘要 |
| `assert` | Python 內建斷言敘述 | `assert actual == expected` | 不相等時測試失敗，pytest 顯示差異 |
| `pytest.raises()` | 檢查預期例外 | `with pytest.raises(ValueError, match="..."):` | 確認錯誤資料被明確拒絕 |
| `@pytest.mark.parametrize` | 讓同一測試函式套用多組資料 | 在函式上列出輸入與預期 | 每一列成為可單獨判讀的測試案例 |

## 步驟 1：建立可測試的純函式

**目的：** 將資料轉換與規則放進不開啟硬體、不讀檔案的函式，讓任何案例都能快速重跑。

執行位置：Python／電腦
檔案：`sensor_data.py`

```python
SensorRow = dict[str, str | float]

MIN_TEMP_C = -40.0
MAX_TEMP_C = 85.0


def normalize_sensor(data: dict[str, object]) -> SensorRow:
    try:
        timestamp = str(data["timestamp"]).strip()
        temp_c = float(data["temp_c"])
        humidity = float(data["humidity"])
    except KeyError as error:
        raise ValueError(f"缺少欄位：{error.args[0]}") from error
    except (TypeError, ValueError) as error:
        raise ValueError("temp_c 與 humidity 必須是數字") from error

    if not timestamp:
        raise ValueError("timestamp 不可空白")
    if not MIN_TEMP_C <= temp_c <= MAX_TEMP_C:
        raise ValueError(f"temp_c 必須介於 {MIN_TEMP_C:g} 到 {MAX_TEMP_C:g}")
    if not 0 <= humidity <= 100:
        raise ValueError("humidity 必須介於 0 到 100")

    return {"timestamp": timestamp, "temp_c": temp_c, "humidity": humidity}
```

`normalize_sensor()` 傳入一筆字典，回傳欄名固定、數值已轉成 `float` 的字典。缺欄位、無法轉數字、空白時間或超出範圍時，函式會主動丟出 `ValueError`；這些就是可測試的資料規則，不是直接把錯誤資料寫進 CSV。

## 步驟 2：撰寫正常、邊界與錯誤案例

**目的：** 將你希望函式永遠維持的行為，寫成可重複執行的規格。

執行位置：Python／電腦
檔案：`test_sensor_data.py`

```python
import pytest

from sensor_data import normalize_sensor


def test_normalize_sensor_converts_numeric_text() -> None:
    row = normalize_sensor(
        {"timestamp": " 2026-09-14 09:00:00 ", "temp_c": "26.4", "humidity": "62"}
    )
    assert row == {"timestamp": "2026-09-14 09:00:00", "temp_c": 26.4, "humidity": 62.0}


@pytest.mark.parametrize(
    ("data", "message"),
    [
        ({"timestamp": "t", "temp_c": -40, "humidity": 0}, ""),
        ({"timestamp": "t", "temp_c": 85, "humidity": 100}, ""),
        ({"timestamp": "t", "temp_c": 86, "humidity": 50}, "temp_c 必須介於"),
        ({"timestamp": "t", "temp_c": 25, "humidity": 101}, "humidity 必須介於"),
    ],
)
def test_normalize_sensor_checks_boundaries(data: dict[str, object], message: str) -> None:
    if message:
        with pytest.raises(ValueError, match=message):
            normalize_sensor(data)
    else:
        normalize_sensor(data)


@pytest.mark.parametrize(
    ("data", "message"),
    [
        ({"timestamp": "t", "temp_c": 25}, "缺少欄位：humidity"),
        ({"timestamp": "", "temp_c": 25, "humidity": 50}, "timestamp 不可空白"),
        ({"timestamp": "t", "temp_c": "warm", "humidity": 50}, "必須是數字"),
    ],
)
def test_normalize_sensor_rejects_invalid_data(data: dict[str, object], message: str) -> None:
    with pytest.raises(ValueError, match=message):
        normalize_sensor(data)
```

`pytest.raises(ValueError)` 表示「這筆錯誤資料必須引發 `ValueError`」。`match` 再檢查訊息是否包含可判讀原因。`parametrize` 的四列邊界資料不是同一次模糊檢查；pytest 會把每列當成一個案例，因此能知道究竟是哪一筆失敗。

## 步驟 3：執行與判讀結果

**目的：** 讓 pytest 收集測試並指出哪個輸入／輸出規則沒有被滿足。

執行位置：PowerShell／電腦

```powershell
uv run --with pytest pytest -q
```

預期結果：顯示 `8 passed`。數字代表收集到並通過的案例數，會隨你增減參數化資料而改變。

若要理解失敗輸出，可暫時把第一個預期濕度從 `62.0` 改成 `62.1` 後再執行。pytest 會指出測試函式名稱，並顯示實際值 `62.0` 與預期值 `62.1` 的差異。看完後立刻改回 `62.0` 並重新執行，確認全部通過；不要為了讓測試變綠而隨意修改程式或刪除案例。

## 可選練習：先寫案例，再補規則

**目標：** 新增「濕度不可為負數」的案例。先在參數化資料中加入 `humidity=-1` 與預期錯誤訊息，執行後確認它失敗；再確認函式已有或補上正確規則，最後讓案例通過。只使用固定字典，不要改動 ESP32、CSV 或真實感測資料。

## 完成時，應能確認

- 能用 Arrange、Act、Assert 說明一個測試案例。
- 能區分正常、邊界與錯誤資料各自用來發現什麼問題。
- 能執行 `uv run --with pytest pytest -q`，並知道 `8 passed` 的意思。
- 能從失敗輸出找到案例名稱與預期／實際差異。
- 能理解單元測試無法取代 UART、Gradio 與 ESP32 的整合或實機觀察。

## 常見問題

### pytest 顯示找不到測試

確認檔名是 `test_sensor_data.py`，函式名稱以 `test_` 開始，且 PowerShell 位於包含兩個檔案的資料夾。`pytest` 依命名規則收集測試，不會自動執行任意 Python 檔。

### 測試通過了，但 ESP32 仍沒有資料

這是合理的。這些案例沒有開啟 COM 埠，也沒有和 ESP32 通訊；請回到[UART：讓 Python 與 ESP32 雙向通訊](03-uart-python-esp32.md)與主線實機檢查順序。

### 我用 `try`／`except` 把錯誤吃掉，測試就通過嗎？

不應這樣做。`pytest.raises()` 是要確認特定錯誤會被明確提出；若你吞掉未知錯誤，測試反而失去發現問題的能力。只處理你知道如何回覆的例外，並保留可判讀訊息。

## 重點整理

- 測試概念跨語言可遷移；`pytest` 是 Python 的一種落地工具。
- 單元測試把純函式的輸入／輸出規則固定下來，正常、邊界與錯誤案例缺一不可。
- 通過代表目前案例符合預期，不代表整個產品、硬體或網路都已確認正常。
- 失敗輸出是縮小問題範圍的證據，應先看案例名稱與預期／實際差異。

## 下一步

回到[整合專題：完成可展示的環境看板](08-整合專題.md)，分開確認純函式測試、UART 實機檢查與介面操作結果。
