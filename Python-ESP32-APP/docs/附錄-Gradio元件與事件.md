# 延伸選讀：Gradio 介面、元件與版面

Gradio 能把 Python 資料與函式做成可在本機瀏覽器操作的小工具；這一頁先學會依資料型別選元件並安排版面。

## 你會學到什麼

- 說明 `gr.Blocks`、`gr.Row`、`gr.Column`、`gr.Group` 與 `gr.Accordion` 如何分工安排介面。
- 依文字、數字、選項、布林值、表格、字典與圖表選擇常用元件。
- 說出常見元件的傳入值，以及使用者操作後會交給 Python 的基本型別。
- 建立一個不連接 ESP32 的本機「讀書紀錄預覽」介面。

## 開始前

- 已完成[用 uv 建立 Python 虛擬環境](01-uv與虛擬環境.md)，並知道如何用 `uv run` 執行 Python 檔案。
- 本頁是獨立延伸選讀；不需要 ESP32、COM 埠、網路或主線 CSV。
- 本頁以 `Gradio 6.27.0` 驗證。其他版本可能有不同參數或行為，請先確認[Gradio 官方 API 文件](https://www.gradio.app/docs)。

!!! warning "只在目前這台電腦使用"
    範例限定 `server_name="127.0.0.1"` 與 `share=False`。`127.0.0.1` 表示目前這台電腦，不是手機或其他電腦可使用的網址。不要改成公開分享、不要輸入帳密、個人資料或設備控制資料；若輸出網址不是 `127.0.0.1`，先按 `Ctrl+C` 停止並檢查程式。

## 成功的樣子

瀏覽器會看到「讀書紀錄預覽」：上方有標題、科目文字、頁數數字、專注分鐘滑桿、分類下拉與難度單選；下方有完成核取方塊、固定表格、JSON 摘要與收合的圖表預留區。這是介面配置練習，固定資料不代表你的真實讀書紀錄。

## 先用資料形狀選元件

元件不只是畫面外觀；它定義使用者輸入會以什麼 Python 值交給回呼函式，也定義回傳什麼資料才能更新畫面。

| 想處理的資料 | 常用元件 | 常見傳入值 | 使用者操作後的基本值 | 適合的情境 |
| --- | --- | --- | --- | --- |
| 一段文字 | `gr.Textbox` | `label`、`value`、`placeholder`、`lines` | `str` | 名稱、說明、搜尋文字、狀態訊息 |
| 一個數字 | `gr.Number` | `label`、`value`、`minimum`、`maximum` | `int` 或 `float` | 頁數、金額、溫度、數量 |
| 連續範圍內的數字 | `gr.Slider` | `minimum`、`maximum`、`step`、`value` | `int` 或 `float` | 分數、門檻、音量、分鐘數 |
| 多個選項擇一 | `gr.Dropdown`、`gr.Radio` | `choices`、`value`、`label` | 通常是選項的 `str` | 下拉選單適合節省空間；單選按鈕適合少量選項一眼比較 |
| 是／否 | `gr.Checkbox` | `label`、`value` | `bool` | 是否完成、是否同意、是否顯示額外內容 |
| 可點擊動作 | `gr.Button` | 按鈕文字、`variant` | 不直接提供資料；由事件觸發回呼 | 送出、計算、重新整理、清除 |
| 格狀資料 | `gr.Dataframe` | `headers`、`value`、`interactive` | 二維 `list`、DataFrame 等相容表格資料 | 紀錄、清單、比較結果 |
| 有欄位名稱的資料 | `gr.JSON` | `label`、`value` | `dict` 或 `list` 等 JSON 相容資料 | 除錯摘要、設定預覽、API 回覆 |
| Matplotlib 等圖形物件 | `gr.Plot` | `label`、`value` | 圖形物件 | 趨勢圖、長條圖、散點圖 |
| 說明文字 | `gr.Markdown` | Markdown 文字 | 不接收使用者資料 | 標題、操作說明、提醒 |

!!! note "影像與影音元件"
    `gr.Image`、`gr.Audio`、`gr.Video` 也是常見輸入／輸出元件，適合已完成檔案格式、隱私與容量驗證的影像或影音任務。本課主線目前不需要它們，本頁不提供未驗證的檔案上傳或處理步驟；可從[官方 API 文件](https://www.gradio.app/docs)延伸閱讀。

## 版面容器：決定元件如何分組

| 容器 | 是什麼功能 | 最常見用法 | 畫面結果 |
| --- | --- | --- | --- |
| `gr.Blocks()` | 整個 Gradio 介面的最外層容器 | 用 `with` 包住全部元件與事件 | 建立一個可啟動的頁面 |
| `gr.Row()` | 水平排列區塊 | 把兩三個相關欄位並排 | 節省垂直空間，方便比較 |
| `gr.Column()` | 垂直排列的子欄位 | 在同一列中分成「輸入」與「摘要」 | 讓不同責任的區塊分開 |
| `gr.Group()` | 視覺上包住一組相關元件 | 將同一任務的欄位與按鈕放在一起 | 顯示它們是一組操作 |
| `gr.Accordion()` | 可展開／收合的區塊 | 放較少使用的預覽、說明或進階資訊 | 預設畫面較簡潔 |

容器不會自動驗證資料，也不會改變 Python 值；它們只幫使用者辨認「哪些輸入屬於同一件事」。

## 步驟 1：建立讀書紀錄預覽介面

**目的：** 一次看見不同資料型別對應的元件和版面容器，但暫時不處理按鈕事件。

執行位置：Python／電腦
檔案：`gradio_layout_components.py`

```python
import gradio as gr

StudyRows = list[list[str | int]]

study_rows: StudyRows = [
    ["Python 基礎", "條件判斷", 12],
    ["資料處理", "CSV", 8],
]
study_summary: dict[str, str | int | bool] = {
    "subject": "Python 基礎",
    "pages": 12,
    "completed": False,
}

with gr.Blocks(title="讀書紀錄預覽") as demo:
    gr.Markdown("# 讀書紀錄預覽\n固定資料只用於認識 Gradio 元件與版面。")

    with gr.Row():
        with gr.Column():
            with gr.Group():
                gr.Textbox(label="科目", value="Python 基礎", placeholder="例如：Python 基礎")
                gr.Number(label="閱讀頁數", value=12, minimum=0, precision=0)
                gr.Slider(label="專注分鐘", minimum=10, maximum=120, step=5, value=30)
                gr.Dropdown(
                    label="分類",
                    choices=["程式設計", "資料處理", "網路概念"],
                    value="程式設計",
                )
                gr.Radio(label="難度", choices=["入門", "普通", "挑戰"], value="入門")
                gr.Checkbox(label="已完成本次閱讀", value=False)

        with gr.Column():
            gr.Dataframe(
                headers=["分類", "主題", "頁數"],
                value=study_rows,
                label="固定讀書紀錄",
                interactive=False,
            )
            gr.JSON(value=study_summary, label="JSON 摘要")

    gr.Button("下一頁會加入事件", interactive=False)
    with gr.Accordion("圖表元件預留區", open=False):
        gr.Plot(label="圖表會在資料流頁介紹")

demo.launch(server_name="127.0.0.1", share=False)
```

`StudyRows` 是二維 `list` 的型別別名，表示每一筆讀書紀錄都是一列資料。`study_summary` 則是 `dict`，因此適合交給 `gr.JSON` 顯示欄位名稱與值。型別提示協助閱讀；如果資料來自檔案、網路或使用者，仍要在回呼函式中做實際檢查。

預期結果：程式啟動後顯示的是固定初始值。按鈕刻意不能按，因為事件與資料更新會在下一頁處理。

## 步驟 2：啟動並觀察元件

**目的：** 確認不同元件呈現資料的方式，並觀察使用者可修改與不可修改的差異。

執行位置：PowerShell／電腦

```powershell
uv run --with gradio python gradio_layout_components.py
```

1. 在 PowerShell 開啟顯示的 `http://127.0.0.1:7860`；若埠號不同，以實際網址為準。
2. 修改「科目」、頁數、滑桿、分類、難度和核取方塊，觀察各自的輸入形式。
3. 確認表格不能直接修改，因為 `interactive=False`；展開「圖表元件預留區」，確認它目前只是空的輸出位置。
4. 按鈕是停用狀態，這是正常現象。回到 PowerShell 按 `Ctrl+C` 結束。

預期結果：你可改變輸入元件的畫面值，但固定表格、JSON 與空圖表不會隨之更新；元件有值不等於它們已經連到 Python 函式。

## 步驟 3：用選擇理由檢查元件

**目的：** 不只背元件名稱，而是能根據資料和操作目的做選擇。

| 情境 | 建議元件 | 原因 |
| --- | --- | --- |
| 使用者輸入 0 到 100 的門檻 | `gr.Slider` 或 `gr.Number` | 範圍固定且希望避免任意文字時用滑桿；需要輸入精確數字時用數字框 |
| 三種模式擇一 | `gr.Radio` | 選項少且想讓使用者一次看到全部選擇 |
| 從十種以上課程分類選一 | `gr.Dropdown` | 節省畫面空間 |
| 顯示欄名不同的多筆資料 | `gr.Dataframe` | 可用表頭閱讀每一欄的意義 |
| 顯示 Python 字典的設定摘要 | `gr.JSON` | 保留鍵和值的結構，而不是拼成一長段文字 |

這些是常見選擇，不是唯一答案。先問「值的型別是什麼、使用者要怎麼改、結果要怎麼看」，再決定元件。

## 可選練習：新增閱讀方式

**目標：** 新增一個「閱讀方式」的 `gr.Radio`，讓使用者在「紙本」與「電子書」中選一個。

把元件放進第一個 `gr.Group()`，並設定初始值。預期結果：頁面出現兩個單選項目；切換選項時只改變畫面上的選取狀態，因為目前尚未設定事件。

## 完成時，應能確認

- 能依文字、數字、選項、布林值、表格與字典選擇合適的 Gradio 元件。
- 能說明 `Blocks` 是整頁容器，`Row`／`Column`／`Group`／`Accordion` 只負責版面與分組。
- 能辨認 `interactive=False` 的表格不能由使用者直接修改。
- 能確認網址為 `127.0.0.1`，程式沒有使用 `share=True` 或外部資料。

## 常見問題

### 程式顯示找不到 `gradio`

確認 PowerShell 位於含有 `pyproject.toml` 的資料夾，再執行本頁命令。若 `uv` 本身無法執行，回到[用 uv 建立 Python 虛擬環境](01-uv與虛擬環境.md)檢查安裝步驟。

### 表格中的資料無法直接修改

這是 `interactive=False` 的效果，目的是把它當成輸出預覽。若日後要讓使用者編輯表格，必須另外設計資料格式、驗證與儲存流程，不能只改一個參數就視為資料已安全保存。

### 我修改了輸入元件，JSON 和表格沒有改變

目前程式只建立版面，尚未將輸入元件連到 Python 回呼函式。下一頁才會介紹事件、`inputs`、`outputs` 與多個元件更新。

## 重點整理

- 元件選擇從資料型別和操作目的開始，而不是只看畫面外觀。
- `Blocks`、`Row`、`Column`、`Group`、`Accordion` 幫助安排介面，但不處理資料驗證。
- 輸入元件可以有畫面值；要讓資料計算或更新其他元件，仍需要事件與 Python 函式。
- 本機原型適合學習資料流，不是公開服務或長期資料系統。

## 下一步

閱讀[延伸選讀：Gradio 事件、資料流與元件更新](附錄-Gradio事件資料流與元件更新.md)，讓這些元件開始呼叫 Python 函式並更新畫面。
