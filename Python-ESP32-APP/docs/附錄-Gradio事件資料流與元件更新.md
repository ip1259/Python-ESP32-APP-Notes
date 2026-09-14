# 延伸選讀：Gradio 事件、資料流與元件更新

這一頁讓本機介面的輸入元件呼叫 Python 函式，並將結果更新到數字、文字與結構化摘要。

## 你會學到什麼

- 分辨 `click()`、`change()`、`submit()` 的觸發時機。
- 依回呼函式的參數安排 `inputs`，依回傳值安排 `outputs`。
- 用單一與多個輸出更新不同元件，並讀懂事件鏈 `then()`。
- 以回傳元件設定切換額外說明的可見性。

## 開始前

- 先閱讀[延伸選讀：Gradio 介面、元件與版面](附錄-Gradio元件與事件.md)。
- 本頁以 `Gradio 6.27.0` 驗證；在含有 `pyproject.toml` 的資料夾使用 `uv run --with gradio`。可查閱[Gradio 官方 API 文件](https://www.gradio.app/docs)。
- 範例使用固定溫度轉換，不讀取 ESP32 或網路資料。

!!! warning "本機原型的範圍"
    僅使用 `127.0.0.1` 與 `share=False`。範例的輸入、摘要和備註不會長期保存；不要用它處理帳密、個資、真實設備控制或公開服務。

## 成功的樣子

輸入 `25`、選擇「華氏（°F）」並按下按鈕後，數字框顯示 `77`、狀態文字顯示轉換結果，JSON 摘要同時顯示輸入與輸出。修改數字時會更新預覽文字；在備註框按 Enter 後，會去除前後空白並顯示字數。

## 事件資料流

```text
元件事件 → Python 回呼函式 → 回傳值 → outputs 指定的元件
                         ↓
                    then() 後續事件
```

| 事件 | 常見來源 | 何時觸發 | 回呼輸入／輸出 |
| --- | --- | --- | --- |
| `click()` | `Button` | 使用者按下按鈕 | 常用於明確送出、計算、重新整理 |
| `change()` | `Number`、`Dropdown`、`Checkbox` 等 | 元件值改變 | 常用於即時預覽或調整其他畫面 |
| `submit()` | `Textbox` | 使用者在文字框按 Enter | 常用於送出文字，不需另外點按鈕 |
| `then()` | 前一個事件的結果 | 前一個事件完成後 | 把「先計算、再更新提示」拆成兩步 |

`inputs=[celsius, unit]` 表示 Gradio 會依順序把兩個元件的值傳給函式參數；`outputs=[converted, status, summary]` 則表示函式的三個回傳值依序更新三個元件。順序或型別不相符，是最常見的畫面更新失敗原因。

## 步驟 1：建立溫度轉換與摘要工具

**目的：** 用同一個按鈕更新數字、文字與 JSON，並觀察其他事件的差異。

執行位置：Python／電腦
檔案：`gradio_events.py`

```python
import gradio as gr


def convert_temperature(celsius: float, unit: str) -> tuple[float, str, dict[str, float | str]]:
    if unit == "華氏（°F）":
        converted: float = celsius * 9 / 5 + 32
        symbol = "°F"
    else:
        converted = celsius + 273.15
        symbol = "K"

    summary: dict[str, float | str] = {
        "input_celsius": celsius,
        "unit": unit,
        "converted": round(converted, 2),
    }
    return converted, f"已將 {celsius:.1f}°C 轉為 {converted:.2f}{symbol}。", summary


def describe_temperature(celsius: float | None) -> str:
    if celsius is None:
        return "請先輸入數字。"
    return f"目前輸入：{celsius:.1f}°C"


def clean_note(note: str) -> tuple[str, str]:
    cleaned: str = note.strip()
    if not cleaned:
        return "", "沒有輸入備註。"
    return cleaned, f"已收到 {len(cleaned)} 個字的備註。"


def toggle_details(show: bool) -> gr.Markdown:
    return gr.Markdown(visible=show)


with gr.Blocks(title="Gradio 事件練習") as demo:
    gr.Markdown("# 溫度轉換與資料流\n固定數字只用於觀察事件。")
    with gr.Row():
        celsius = gr.Number(label="攝氏溫度", value=25.0)
        unit = gr.Dropdown(label="轉換單位", choices=["華氏（°F）", "開爾文（K）"], value="華氏（°F）")
    convert_button = gr.Button("轉換", variant="primary")
    converted = gr.Number(label="轉換結果", interactive=False)
    status = gr.Textbox(label="狀態", interactive=False)
    summary = gr.JSON(label="轉換摘要")
    preview = gr.Textbox(label="輸入預覽", interactive=False)
    note = gr.Textbox(label="備註（按 Enter 送出）")
    note_status = gr.Textbox(label="備註狀態", interactive=False)
    show_details = gr.Checkbox(label="顯示額外說明", value=False)
    details = gr.Markdown("額外說明：事件可讓元件值交給 Python 函式。", visible=False)

    convert_event = convert_button.click(
        fn=convert_temperature,
        inputs=[celsius, unit],
        outputs=[converted, status, summary],
    )
    convert_event.then(fn=lambda selected_unit: f"目前選擇：{selected_unit}", inputs=unit, outputs=preview)
    celsius.change(fn=describe_temperature, inputs=celsius, outputs=preview)
    note.submit(fn=clean_note, inputs=note, outputs=[note, note_status])
    show_details.change(fn=toggle_details, inputs=show_details, outputs=details)

demo.launch(server_name="127.0.0.1", share=False)
```

`convert_temperature()` 的回傳型別是三個值：`float` 更新數字框、`str` 更新狀態、`dict` 更新 JSON。`toggle_details()` 則回傳新的 `gr.Markdown(visible=...)` 設定，讓同一個元件保留內容但改變是否可見。

## 步驟 2：依序觸發事件

**目的：** 觀察事件不是自動猜測，而是由特定操作觸發。

執行位置：PowerShell／電腦

```powershell
uv run --with gradio python gradio_events.py
```

1. 輸入 `25`，選擇「華氏（°F）」，按「轉換」；確認三個輸出同時更新。
2. 直接修改攝氏數字；確認「輸入預覽」由 `change()` 更新，但結果數字不會更新，直到按按鈕。
3. 在備註輸入 `  今天練習  ` 並按 Enter；確認文字被去除前後空白，且顯示字數。
4. 勾選「顯示額外說明」；確認收合文字出現，再取消勾選確認它隱藏。

預期結果：每一項操作只觸發它所綁定的事件。這可避免把昂貴或有副作用的工作放在每一次輸入變更上。

## 常見問題

### 按按鈕後元件顯示不合理的資料或出現錯誤

先數回呼的回傳值數量，再數 `outputs` 元件數量，兩者必須相同且順序一致。接著確認數字元件收到數字、JSON 元件收到 `dict` 或 JSON 相容資料。

### 改變輸入數字卻沒有重新轉換

這是刻意的設計：數字變更只執行 `describe_temperature()`。轉換是 `click()` 事件，必須按按鈕。選擇事件時先考量使用者是否需要明確確認動作。

### 備註按 Enter 沒有反應

確認游標仍在備註文字框內，且程式沒有在終端機顯示錯誤。`submit()` 是文字框的 Enter 事件，不是所有元件都有的事件。

## 重點整理

- `click()`、`change()`、`submit()` 分別代表按鈕、值變更與文字送出。
- `inputs` 對應函式參數，`outputs` 對應回傳值，兩者都依順序處理。
- `then()` 適合在前一步完成後更新額外提示；它不會取代前一步的輸出。
- 元件設定也可作為回傳值的一部分，但只應更新經驗證且容易觀察的設定。

## 下一步

閱讀[延伸選讀：Gradio 狀態、驗證與使用體驗](附錄-Gradio狀態驗證與使用體驗.md)，了解跨事件的短期資料、輸入規則與錯誤回覆。
