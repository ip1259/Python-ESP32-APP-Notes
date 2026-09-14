# 延伸選讀：Gradio 元件與事件

這一頁用固定的練習資料，讓你看懂按鈕如何呼叫 Python 函式，並依序更新文字與表格元件。

## 你會學到什麼

- 說出 `Blocks`、`Textbox`、`Button` 與 `Dataframe` 各自負責的畫面工作。
- 用 `click()` 將按鈕事件連到具備型別提示的 Python 回呼函式。
- 讓一個回呼依 `outputs` 的順序，同時更新文字和表格。
- 分辨本機 `127.0.0.1` 原型與公開服務的界線。

## 開始前

- 先完成或閱讀[Gradio：建立本機 AIoT 儀表板](07-gradio-aiot儀表板.md)，知道 Gradio 可把 Python 函式做成同一台電腦使用的網頁介面。
- 這是延伸選讀，不是主線驗收；不需要 ESP32、COM 埠或網路。
- 在含有 `pyproject.toml` 的專案資料夾操作，並已依[用 uv 建立 Python 虛擬環境](01-uv與虛擬環境.md)安裝 `uv`。
- 本頁以 `Gradio 6.27.0` 驗證；若你使用其他版本，先依本頁結果測試，再查閱[Gradio 官方 API 文件](https://www.gradio.app/docs)確認元件參數與事件寫法。

!!! warning "本頁只限本機"
    範例使用 `server_name="127.0.0.1"` 與 `share=False`，只能由目前這台電腦開啟。不要改成公開網址、不要填入帳密或個人資料；若你看到不是 `127.0.0.1` 的網址，先停止程式並檢查啟動參數。

## 成功的樣子

按下「更新練習資料」後，狀態文字應顯示 `已載入 2 筆練習資料。`，表格出現兩列固定的溫度與濕度資料。這些數字只用來確認介面更新，並不代表真實環境。

## 元件與事件如何合作

```text
瀏覽器按鈕
    ↓ click()
Python 回呼函式 update_preview()
    ↓ 回傳（文字，表格資料）
Textbox                 Dataframe
```

| 名稱 | 是什麼功能 | 常見傳入值 | 產生或更新什麼 |
| --- | --- | --- | --- |
| `gr.Blocks()` | 建立一個介面的容器 | 標題等介面設定 | 承裝元件與事件的網頁介面 |
| `gr.Textbox()` | 顯示或讓使用者輸入單行、多行文字 | `label`、初始 `value` | 文字內容；本頁用來顯示狀態 |
| `gr.Button()` | 建立可點擊按鈕 | 按鈕標籤文字 | 使用者點擊時觸發事件 |
| `gr.Dataframe()` | 顯示表格資料 | 欄名 `headers`、`interactive` | 表格畫面；本頁接收二維 `list` |
| `.click()` | 把按鈕點擊連到 Python 回呼 | 回呼函式、`inputs`、`outputs` | 事件設定；點擊時依序更新輸出元件 |

`outputs=[status, table]` 的順序很重要：回呼的第一個回傳值會更新 `status`，第二個回傳值會更新 `table`。它不是依變數名稱自動配對。

## 步驟 1：建立最小介面

**目的：** 先建立畫面元件與一個不依賴硬體的回呼函式。

執行位置：Python／電腦
檔案：`gradio_components.py`

```python
import gradio as gr

SensorTable = list[list[str | float]]


def update_preview() -> tuple[str, SensorTable]:
    rows: SensorTable = [
        ["2026-09-14 09:00:00", 26.4, 62.0],
        ["2026-09-14 09:05:00", 26.8, 61.0],
    ]
    return f"已載入 {len(rows)} 筆練習資料。", rows


with gr.Blocks(title="Gradio 元件與事件練習") as demo:
    gr.Markdown("# Gradio 元件與事件練習\n此頁只在這台電腦本機使用。")
    update_button = gr.Button("更新練習資料", variant="primary")
    status = gr.Textbox(label="狀態", value="尚未更新。", interactive=False)
    table = gr.Dataframe(
        headers=["timestamp", "temp_c", "humidity"],
        label="固定練習資料",
        interactive=False,
    )

    update_button.click(
        fn=update_preview,
        outputs=[status, table],
        api_name="update_preview",
    )

demo.launch(server_name="127.0.0.1", share=False)
```

`SensorTable` 是 `list[list[str | float]]` 的別名，表示「每一列都是由文字或數字組成的表格資料」。型別提示能幫助你閱讀回呼的輸出形狀；外部資料進入程式時，仍要另外檢查欄位與數值。

預期結果：儲存檔案時不會啟動網頁；程式要執行後才會建立介面。

## 步驟 2：啟動並操作

**目的：** 確認本機伺服器與按鈕事件能更新兩個元件。

執行位置：PowerShell／電腦

```powershell
uv run --with gradio python gradio_components.py
```

1. 在 PowerShell 找到 `http://127.0.0.1:7860`；若埠號不同，以實際顯示的網址為準。
2. 用同一台電腦的瀏覽器開啟該網址。
3. 按「更新練習資料」。
4. 確認狀態文字與表格都更新，再回到 PowerShell 按 `Ctrl+C` 停止程式。

預期結果：狀態框顯示 `已載入 2 筆練習資料。`，表格有兩列資料；停止程式後該網址無法再開啟是正常現象。

## 步驟 3：對照回傳值與輸出順序

**目的：** 知道主線為何能一次更新多個畫面元件。

`update_preview()` 回傳的值如下：

```python
return "已載入 2 筆練習資料。", rows
```

它和下列 `outputs` 逐項配對：

```python
outputs=[status, table]
```

所以第一個字串交給 `status`，第二個二維 `list` 交給 `table`。主線的 `refresh_dashboard()` 同樣原理，只是它一次回傳五個值，並加入 UART、資料表和圖表處理。

!!! tip "先從最小介面排除問題"
    當主線儀表板畫面沒有更新時，先確認這個固定資料範例能否運作。若能運作，Gradio 元件與事件通常沒有問題；再依序檢查主線的 COM 埠、ESP32 JSON 與回呼函式，不要同時啟動兩支會讀取同一個 COM 埠的程式。

## 可選練習：新增第三筆資料

**目標：** 在既有 `rows` 中加上一筆固定資料，並觀察文字中的筆數自動改變。

在第二列資料後新增一列，例如：

```python
["2026-09-14 09:10:00", 27.1, 60.0],
```

預期結果：按下按鈕後，狀態顯示 `已載入 3 筆練習資料。`，表格顯示三列。只修改 `gradio_components.py` 這個練習檔；不要把練習數字寫回主線 CSV。

## 完成時，應能確認

- 能說明按鈕點擊後，是 Gradio 呼叫 Python 回呼函式，而不是按鈕直接讀取 ESP32。
- `update_preview()` 的第一個與第二個回傳值，分別更新 `status` 與 `table`。
- 按一次按鈕後可看見固定兩列資料和相符的狀態文字。
- 網址是本機 `127.0.0.1`，且程式沒有使用 `share=True`。

## 常見問題

### PowerShell 顯示找不到 `gradio`

確認目前位於含有 `pyproject.toml` 的資料夾，再重新執行本頁的 `uv run --with gradio python gradio_components.py`。若 `uv` 本身找不到，回到[用 uv 建立 Python 虛擬環境](01-uv與虛擬環境.md)完成安裝。

### 按鈕按了，但文字或表格沒有更新

先檢查終端機是否顯示 Python 錯誤。再確認 `return` 有兩個值，且 `outputs=[status, table]` 也有兩個元件且順序相同；不要把表格資料回傳成單一字串。

### 瀏覽器無法開啟網址

確認 Python 程式仍在執行，並使用 PowerShell 顯示的完整 `127.0.0.1` 網址。若埠號已被其他 Gradio 程式使用，先停止先前的程式後再啟動；不需要改成公開網址。

## 重點整理

- `Blocks` 容納元件與事件；`Textbox`、`Button`、`Dataframe` 分別負責文字、點擊和表格畫面。
- `.click()` 將使用者點擊交給 Python 回呼，再依 `outputs` 順序更新元件。
- 用固定資料可先驗證互動流程，再回到主線排查 UART 與感測資料。
- localhost 只供目前這台電腦使用，不代表公開服務。

## 下一步

接著閱讀[延伸選讀：Gradio 狀態、驗證與使用體驗](附錄-Gradio狀態驗證與使用體驗.md)，學習如何保存短期狀態、檢查輸入並顯示清楚提示。
