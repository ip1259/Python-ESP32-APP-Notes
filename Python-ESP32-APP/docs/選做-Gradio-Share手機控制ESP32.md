# 選做實作：Gradio Share 手機操作整合儀表板

> 對應主線：[整合專題：完成環境看板](08-整合專題.md)

本頁讓手機瀏覽器透過 Gradio 分享網址開啟已完成的整合儀表板，讀取 DHT11 資料、查看最近資料與圖表，並控制 ESP32 板載 LED；這只適合短時間測試，不能當成正式遠端控制服務。

!!! danger "分享網址是公開網址"
    `share=True` 產生的網址可由任何取得網址的人從網際網路開啟，不是「只有同一個 Wi-Fi 才能使用」。分享後，其他人可看見儀表板目前顯示的感測資料、最近資料表與圖表，也可按下頁面上的控制按鈕。不要公開貼出、截圖或轉傳網址；程式中不得放入 Wi-Fi 密碼、個人資料、token 或不適合公開的資料。

## 你會學到什麼

- `share=True` 如何讓手機瀏覽器連到仍在電腦上執行的整合儀表板。
- 手機上的讀取與 LED 按鈕如何經由 Python、USB UART 與 ESP32 協作。
- 為什麼公開連結只能作短時間測試，不能取代正式產品的安全設計。

## 開始前

| 項目 | 需求 |
| --- | --- |
| 前置教材 | 已完成 [整合專題：完成環境看板](08-整合專題.md) |
| 硬體 | 已接 USB 的 ESP32、DHT11，以及主線已成功的 LCD 或裸 8×8 點陣 |
| Python 套件 | `pyserial`、`pandas`、`matplotlib`、`gradio` |
| 程式 | 主線的 `integrated_dashboard.py` 已能在 localhost 正常運作 |
| 網路 | 電腦與手機可進行短時間網際網路測試 |

!!! warning "先確認本機版本成功"
    先以主線的 `share=False` 在電腦 localhost 完成讀取、CSV、圖表與 LED 控制。啟動分享版前，關閉 Arduino 序列監控和其他開啟同一個 COM 埠的程式；整合儀表板仍必須是唯一的 COM 埠使用者。

## 分享連結如何工作

```text
手機瀏覽器 → 公開 Gradio 分享網址 → 電腦的 integrated_dashboard.py → USB UART → ESP32
                                                    ↓
                                            CSV／圖表／感測資料
```

手機不是直接連到 ESP32。它只透過公開網頁呼叫電腦上仍在執行的 Python 程式；Python 才會讀寫 UART、CSV 與圖表。電腦關機、程式結束或網路中斷後，手機頁面便無法完成讀取或控制。

## 步驟 1：將主線儀表板改為分享模式

**目的：** 保持主線程式其餘行為不變，只將啟動方式改為建立分享連結。

開啟已完成的 `integrated_dashboard.py`，找到檔案結尾的 `demo.launch()`，將 `share=False` 改為 `share=True`：

執行位置：Python／電腦  
檔案：`integrated_dashboard.py`

```python
try:
    demo.launch(server_name="127.0.0.1", share=True)
finally:
    dashboard.close()
```

這一行的其他設定不要修改。`127.0.0.1` 仍提供電腦本機頁面；`share=True` 另外建立一個可從網際網路開啟的分享網址。

!!! warning "不要把隱藏 API 文件當成保護"
    就算介面不顯示 API 文件，公開網址仍不是私密服務。本頁不嘗試用隱藏網址、基本登入或其他簡化做法當成正式存取控制。

## 步驟 2：用手機操作整合儀表板

**目的：** 確認手機頁面的資料與控制命令，能透過同一支 Python 程式正確到達 ESP32。

執行位置：PowerShell／電腦

```powershell
uv run python integrated_dashboard.py
```

PowerShell 會顯示一個 `https://...gradio.live` 類型的分享網址。以手機瀏覽器開啟該網址後：

1. 按「讀取並記錄資料」，確認最新溫溼度、最近資料表和圖表更新。
2. 按「LED 開啟」與「LED 關閉」，確認板載 LED 與手機狀態文字一致。
3. 測試完成後，關閉手機頁面，回到電腦 PowerShell 按 `Ctrl+C` 結束程式。
4. 要繼續使用主線時，將 `share=True` 改回 `share=False`。

預期結果：手機可使用與 localhost 相同的整合儀表板；按下讀取後，電腦端的 CSV 與圖表會更新，LED 控制也會收到 ESP32 的狀態回覆。

## 資訊安全與停止條件

| 情況 | 必須做的事 |
| --- | --- |
| 測試完成 | 立刻在電腦按 `Ctrl+C` 結束程式，讓分享服務停止。 |
| 網址被貼到群組、公開平台或不確定的人取得 | 立刻停止程式，不再繼續使用該輪測試。 |
| CSV、圖表或頁面出現個人資料、位置資訊、敏感環境資料或 token | 不使用 `share=True`；改回 localhost。 |
| 頁面加入馬達、門鎖、繼電器、攝影機或其他具風險控制 | 不使用本頁做法；改回 localhost，另行進行安全規劃。 |
| 網路、端點防護或瀏覽器安全機制不允許 | 停止測試，不嘗試繞過限制。 |
| 需要長期、多人或遠端控制 | 不使用本頁做法；應另行規劃身分驗證、權限、日誌、速率限制、網路隔離與安全審查。 |

!!! warning "基本登入不等於正式安全措施"
    Gradio 的基本驗證不包含多因素驗證、完整權限管理、嘗試次數限制或完整稽核。即使日後加上登入，也不能把這個課堂範例當成正式遠端控制系統。

## 完成時，應能確認

- 你能說出手機連的是 Gradio 分享網址，不是直接連 ESP32。
- 手機可讀取整合儀表板的溫溼度、資料表與圖表，並控制板載 LED。
- 電腦上的 `integrated_dashboard.py` 仍是唯一開啟 COM 埠的程式。
- 測試後已用 `Ctrl+C` 結束程式，且已將 `share` 改回本機模式。
- 你知道分享網址公開了儀表板可見資料與控制功能，會自行評估風險並謹慎使用。

## 常見問題

### PowerShell 沒有顯示分享網址

先確認電腦可以正常連網，並確認 Gradio 套件可匯入。若網路、端點防護或瀏覽器安全機制阻擋分享服務，停止測試並改回 `share=False` 的 localhost 主線；不要嘗試關閉防護或繞過網路限制。

### 手機開得了頁面，但資料無法更新或 LED 沒有反應

先確認電腦上的 `integrated_dashboard.py` 仍在執行，且沒有 Arduino 序列監控或其他 Python 程式開啟同一個 COM 埠。接著回到整合專題，確認 ESP32 的 `115200` baud rate、`read_now` 與 LED `status` 回覆都正常。

### 手機頁面還能開啟，但電腦已經關閉程式

頁面可能暫時留在瀏覽器快取中，但它無法再取得新資料或送出有效控制。重新整理頁面後應無法使用；若網址曾外流，保持程式停止，不要重新使用或轉傳網址。

## 重點整理

- `share=True` 會將完整整合儀表板建立為可從網際網路開啟的公開分享網址。
- 手機透過網頁呼叫 Python；Python 再處理 UART、CSV、圖表與 ESP32 LED。
- 公開網址會暴露頁面可見資料與控制功能，分享前要先移除不適合公開的內容。
- 測試結束後以 `Ctrl+C` 停止程式，並改回 `share=False`。
- 正式遠端控制需要比分享連結更多的安全設計與審查。

## 回到主線

回到 [整合專題：完成環境看板](08-整合專題.md)，繼續使用預設的 localhost 儀表板。

## 參考資料

- [Gradio 官方文件：Sharing Your App](https://www.gradio.app/guides/sharing-your-app)
- [Gradio 官方文件：Blocks.launch](https://www.gradio.app/docs/gradio/blocks)
