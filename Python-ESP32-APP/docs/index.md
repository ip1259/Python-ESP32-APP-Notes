# Python、ESP32、App 整合設計

這是一套以筆記方式編寫的 AIoT 實作教材。你會使用 ESP32 蒐集感測資料，再用 Python 進行資料處理、視覺化與 Gradio 儀表板展示。

!!! info "閱讀方式"
    本網站目前列出的第 1 至第 8 節是**主線課程**；標示為**延伸選讀**的內容用來深入理解該主題，不影響主線完成。標示為**即將推出**的頁面是教材預告，內容仍在整理與驗證中。

## 開始前

- 你已能在 Arduino IDE 上傳 ESP32 程式。
- 你已完成 Blink，並能讀取 DHT11 溫溼度資料。
- 請從 [用 uv 建立 Python 虛擬環境](01-uv與虛擬環境.md) 開始。

## 課程成果

完成課程後，你將能完成下列資料流：

```text
DHT11 → ESP32 → USB UART → Python → CSV／圖表／Gradio
```

每個主題先閱讀主線課程；想深入理解時，再閱讀同一主題下標示為「延伸選讀」的內容。你也可以先瀏覽「即將推出」頁面，了解後續教材的完整架構。

## 課程主題

### 開始前

- [用 uv 建立 Python 虛擬環境](01-uv與虛擬環境.md)
- [課程準備：認識 AIoT 資料流](02-課程準備.md)

### USB 序列通訊

- [UART：讓 Python 與 ESP32 雙向通訊](03-uart-python-esp32.md)
- [延伸選讀：UART JSON、緩衝區與請求／回應](附錄-UART-JSON緩衝區與請求回應.md)
- [延伸選讀：UART 程式導讀](附錄-UART程式導讀.md)
- [即將推出：pyserial 連線、讀寫與逾時](附錄-pyserial連線讀寫與逾時.md)
- [即將推出：pyserial 埠與資料除錯](附錄-pyserial埠與資料除錯.md)
- [即將推出：pyserial 連線生命週期與 COM 埠除錯](附錄-pyserial連線生命週期與COM埠除錯.md)

### 資料處理

- [將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)
- [延伸選讀：CSV 與圖表程式導讀](附錄-CSV與圖表程式導讀.md)
- [延伸選讀：感測資料品質](附錄-感測資料品質.md)
- [即將推出：Pandas 基礎操作](附錄-Pandas基礎操作.md)
- [即將推出：Pandas 清理、篩選與彙整](附錄-Pandas清理篩選與彙整.md)
- [即將推出：Matplotlib 圖表基本元件](附錄-Matplotlib圖表基本元件.md)
- [即將推出：Matplotlib 可讀性與輸出](附錄-Matplotlib可讀性與輸出.md)

### Python 程式設計

- [延伸選讀：pathlib 與檔案路徑](附錄-pathlib與檔案路徑.md)
- [即將推出：json 與 csv](附錄-json與csv.md)
- [即將推出：datetime 與 time](附錄-datetime與time.md)
- [即將推出：函式、類別與責任分工](附錄-函式類別與責任分工.md)
- [即將推出：例外處理與錯誤訊息](附錄-例外處理與錯誤訊息.md)
- [即將推出：logging 與除錯紀錄](附錄-logging與除錯紀錄.md)
- [即將推出：pytest 最小測試](附錄-pytest最小測試.md)

### I2C 顯示

- [I2C：使用 LCD1602A 顯示 DHT11 資料](05-i2c-lcd.md)
- [延伸選讀：LCD 顯示程式導讀](附錄-LCD程式導讀.md)
- [即將推出：I2C 位址、電壓與 LCD 除錯](附錄-I2C位址電壓與LCD除錯.md)

### 點陣顯示

- [74HC595：辨識並驅動裸 8×8 點陣](06-74hc595-8x8點陣.md)
- [延伸選讀：74HC595 與裸 8×8 點陣掃描導讀](附錄-74HC595與8x8點陣掃描導讀.md)
- [即將推出：74HC595 與裸 8×8 點陣掃描](附錄-74HC595與裸8x8點陣掃描.md)
- [即將推出：選做實作｜MAX7219 點陣模組](選做-MAX7219點陣模組.md)

### 本機儀表板

- [Gradio：建立本機 AIoT 儀表板](07-gradio-aiot儀表板.md)
- [延伸選讀：Gradio 事件、狀態與原型邊界](附錄-Gradio事件狀態與原型邊界.md)
- [即將推出：Gradio 元件與事件](附錄-Gradio元件與事件.md)
- [即將推出：Gradio 狀態、驗證與使用體驗](附錄-Gradio狀態驗證與使用體驗.md)
- [即將推出：選做實作｜ngrok、WebSocket 與公開服務](選做-ngrok-WebSocket與公開服務.md)

### 手機 App 與無線控制

- [資訊補充：常見 GUI 與 App 解決方案](資訊補充-Python後端GUI與App解決方案.md)
- [即將推出：選做實作｜既有 BLE App 測試](選做-既有BLE-App測試.md)
- [即將推出：選做實作｜ESP32 STA／HTTP 與 MIT App Inventor](選做-ESP32-STA-HTTP與MIT-App-Inventor.md)

### 整合專題

- [整合專題：完成環境看板](08-整合專題.md)
- [延伸選讀：環境看板整合程式導讀](附錄-整合專題-環境看板程式導讀.md)
- [延伸選讀：整合專題故障分流](附錄-故障排除索引.md)
- [選做實作：Gradio Share 手機瀏覽器操作整合儀表板](選做-Gradio-Share手機控制ESP32.md)
