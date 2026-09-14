# 延伸選讀：Gradio 狀態、驗證與使用體驗

當介面有多次操作時，除了顯示結果，還要決定哪些資料能暫存、輸入要如何檢查，以及錯誤發生時如何告訴使用者下一步。

## 你會學到什麼

- 分辨函式區域變數、元件初始值、`gr.State` 與 CSV 的資料生命週期。
- 檢查必要文字、文字長度、數字型別與範圍、選項合法性。
- 在同一瀏覽器工作階段累積受控資料，並在畫面給出成功或可修正的錯誤訊息。
- 說明為何輸入驗證不等於帳號驗證、資安保護或感測器校正。

## 開始前

- 先閱讀[延伸選讀：Gradio 事件、資料流與元件更新](附錄-Gradio事件資料流與元件更新.md)。
- 本頁以 `Gradio 6.27.0` 驗證；請在含有 `pyproject.toml` 的資料夾使用 `uv run --with gradio`，並參考[Gradio 官方 API 文件](https://www.gradio.app/docs)。
- 範例是固定的活動報名資料，不需要 ESP32、網路或真實名單。

!!! warning "短期狀態不是資料庫"
    範例僅以 `127.0.0.1` 和 `share=False` 啟動。`gr.State` 只保存目前工作階段的受控練習資料；重新整理頁面或重啟程式就可能清除。不要保存帳密、身分資料、真實名單或設備控制紀錄；長期資料要使用經設計、驗證與保護的檔案或資料庫流程。

## 成功的樣子

輸入 `小林`、選擇「Python」並填入 `2` 後送出，結果顯示已加入，表格出現一列，工作階段筆數為 1。空白姓名、超過 20 個字、非整數或超出 1 到 5 的人數，各自顯示能直接修正的原因。

## 先分清資料活多久

| 放在哪裡 | 何時存在 | 適合什麼 | 不適合什麼 |
| --- | --- | --- | --- |
| 函式區域變數 | 本次回呼執行期間 | 計算結果、暫時清理文字 | 下一次按鈕仍要使用的資料 |
| 元件的 `value` | 介面剛建立時 | 初始提示、預設選項 | 可靠保存使用者每次操作的歷史 |
| `gr.State` | 目前工作階段 | 短期清單、計數、步驟進度 | CSV、資料庫、跨裝置資料或敏感資料 |
| CSV／資料庫 | 依保存規則 | 長期紀錄與後續分析 | 未驗證的介面暫存值 |

## 常見驗證類型

| 驗證 | 先檢查什麼 | 範例錯誤訊息 |
| --- | --- | --- |
| 必要值 | 去除空白後是否仍有文字 | `請輸入姓名。` |
| 長度 | 是否超過可閱讀或可處理的上限 | `姓名最多 20 個字。` |
| 型別 | 值是否存在、是否為預期數字 | `請輸入整數人數。` |
| 範圍 | 數字是否在可接受上下限內 | `人數需介於 1 到 5。` |
| 合法選項 | 值是否仍在程式允許的選項中 | `請從提供的場次中選擇。` |

把不同原因分開回覆，比只寫「送出失敗」更容易修正。這些檢查仍不會驗證使用者身分，也不會自動相信外部資料。

## 步驟 1：建立有狀態的報名練習

**目的：** 將一次送出的資料先檢查，再加入目前工作階段的表格。

執行位置：Python／電腦
檔案：`gradio_state_validation.py`

```python
import gradio as gr

RegistrationRows = list[list[str | int]]
VALID_SESSIONS = {"Python", "資料處理", "介面原型"}


def add_registration(
    name: str,
    session: str,
    people: float | None,
    history: RegistrationRows,
) -> tuple[str, RegistrationRows, RegistrationRows, str]:
    cleaned_name: str = name.strip()
    next_history: RegistrationRows = list(history)

    if not cleaned_name:
        return "請輸入姓名。", next_history, next_history, f"本次工作階段共有 {len(next_history)} 筆"
    if len(cleaned_name) > 20:
        return "姓名最多 20 個字。", next_history, next_history, f"本次工作階段共有 {len(next_history)} 筆"
    if people is None or people != int(people):
        return "請輸入整數人數。", next_history, next_history, f"本次工作階段共有 {len(next_history)} 筆"
    if not 1 <= people <= 5:
        return "人數需介於 1 到 5。", next_history, next_history, f"本次工作階段共有 {len(next_history)} 筆"
    if session not in VALID_SESSIONS:
        return "請從提供的場次中選擇。", next_history, next_history, f"本次工作階段共有 {len(next_history)} 筆"

    next_history.append([cleaned_name, session, int(people)])
    count_text: str = f"本次工作階段共有 {len(next_history)} 筆"
    return f"已加入：{cleaned_name}／{session}／{int(people)} 人。", next_history, next_history, count_text


with gr.Blocks(title="Gradio 狀態與驗證練習") as demo:
    gr.Markdown("# 活動報名練習\n所有資料只在目前工作階段暫存。")
    name = gr.Textbox(label="姓名")
    session = gr.Dropdown(label="場次", choices=sorted(VALID_SESSIONS), value="Python")
    people = gr.Number(label="人數（1 到 5 的整數）", value=1)
    submit_button = gr.Button("加入練習名單", variant="primary")
    status = gr.Textbox(label="結果", interactive=False)
    table = gr.Dataframe(headers=["姓名", "場次", "人數"], label="本次工作階段名單", interactive=False)
    count = gr.Textbox(label="筆數", value="本次工作階段共有 0 筆", interactive=False)
    history_state = gr.State(value=[])

    submit_button.click(
        fn=add_registration,
        inputs=[name, session, people, history_state],
        outputs=[status, history_state, table, count],
    )

demo.launch(server_name="127.0.0.1", share=False)
```

`history_state` 不直接顯示；它把已通過檢查的二維 `list` 帶入下一次回呼。函式回傳的第二個值更新它，第三個相同資料則更新可見表格。若驗證失敗，函式回傳原本的 `next_history`，不會把錯誤輸入加入名單。

## 步驟 2：確認成功與各種錯誤

**目的：** 觀察不同錯誤各有不同修正方向，且錯誤不污染短期狀態。

執行位置：PowerShell／電腦

```powershell
uv run --with gradio python gradio_state_validation.py
```

1. 輸入 `小林`、場次選「Python」、人數填 `2`，按按鈕，確認表格有一列。
2. 清空姓名再送出，確認顯示「請輸入姓名」，且表格仍只有原本一列。
3. 輸入超過 20 個字的姓名、人數 `2.5`、人數 `8`，分別確認長度、整數與範圍提示。
4. 重新整理頁面或停止後重啟程式，確認名單可能回到 0 筆。這是 `gr.State` 的預期界線。

## 可選練習：新增規則前先寫預期結果

**目標：** 加入「同一姓名不能重複」規則前，先列出預期結果：第一次加入成功；第二次使用同名送出時顯示具體提示，表格筆數不增加。完成後再修改 `add_registration()`；不要把這個練習接到真實報名、帳號或設備控制。

## 常見問題

### 驗證失敗後表格仍多了一列

確認 `append()` 位於所有 `return` 驗證分支之後。只要有一項規則不通過，就回傳原本的 `next_history`，不要先加入再嘗試移除。

### 重新整理後資料不見了

這是正常現象。`gr.State` 不是 CSV 或資料庫；長期保存感測資料請使用[將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)的受控檔案流程。

### 為什麼還要檢查下拉選單的合法性？

一般畫面會限制選擇，但回呼函式不應假設每個來源都正確。先確認值仍在允許集合，能讓函式在未預期資料進入時回傳可判讀訊息。

## 重點整理

- `gr.State` 適合當前工作階段的短期資料，不是長期或敏感資料儲存。
- 必要值、長度、型別、範圍與合法選項需要各自檢查與回覆。
- 先驗證、後加入狀態，可避免錯誤資料污染畫面表格。
- 介面驗證不會取代帳號驗證、資安保護、感測器校正或正式產品設計。

## 下一步

閱讀[延伸選讀：測試概念與 pytest 實作](附錄-測試概念與pytest實作.md)，學習把這類純 Python 判斷函式寫成可重複執行的自動化測試。
