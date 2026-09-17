# Python、ESP32、App 整合設計

這是一套以筆記方式編寫的 AIoT 實作教材。你會使用 ESP32 蒐集感測資料，再用 Python 進行資料處理、視覺化與 Gradio 儀表板展示。

!!! info "閱讀方式"
    第 1 至第 8 節是**主線課程**；「延伸選讀」用來深入理解已完成的主線；「選做專題」與「其他選做實作」需要額外條件，不影響主線完成。標示「即將推出」的頁面仍在整理與驗證，尚未提供可操作內容。

## 開始前

- 你已能在 Arduino IDE 上傳 ESP32 程式。
- 你已完成 Blink，並能讀取 DHT11 溫溼度資料。
- 請從 [用 uv 建立 Python 虛擬環境](01-uv與虛擬環境.md) 開始。

## 課程成果

完成主線課程後，你將能完成下列資料流：

```text
DHT11 → ESP32 → USB UART → Python → CSV／圖表／Gradio
```

先依序完成主線八節；想理解程式、除錯或套件用法時，再從延伸選讀找對應主題。想嘗試額外硬體、手機或網路整合時，請先閱讀選做頁面的條件與風險說明。

## 主線課程

1. [01｜用 uv 建立 Python 虛擬環境](01-uv與虛擬環境.md)
2. [02｜課程準備：認識 AIoT 資料流](02-課程準備.md)
3. [03｜UART：讓 Python 與 ESP32 雙向通訊](03-uart-python-esp32.md)
4. [04｜將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)
5. [05｜I2C：使用 LCD1602A 顯示 DHT11 資料](05-i2c-lcd.md)
6. [06｜74HC595：辨識並驅動裸 8×8 點陣](06-74hc595-8x8點陣.md)
7. [07｜Gradio：建立本機 AIoT 儀表板](07-gradio-aiot儀表板.md)
8. [08｜整合專題：完成環境看板](08-整合專題.md)

## 主線課程的延伸選讀

這些頁面直接對應主線中的程式、硬體、資料流或故障症狀；完成對應主線單元後，可依需要閱讀。它們不是主線必做步驟。

### 序列通訊與整合

- [UART JSON、緩衝區與請求／回應](附錄-UART-JSON緩衝區與請求回應.md)
- [UART 程式導讀](附錄-UART程式導讀.md)
- [環境看板整合程式導讀](附錄-整合專題-環境看板程式導讀.md)
- [整合專題故障分流](附錄-故障排除索引.md)

### 資料、顯示與介面

- [CSV 與圖表程式導讀](附錄-CSV與圖表程式導讀.md)
- [感測資料品質](附錄-感測資料品質.md)
- [LCD 顯示程式導讀](附錄-LCD程式導讀.md)
- [74HC595 與裸 8×8 點陣掃描導讀](附錄-74HC595與8x8點陣掃描導讀.md)
- [Gradio 事件、狀態與原型邊界](附錄-Gradio事件狀態與原型邊界.md)

## 通用延伸選讀

這些頁面可用一般 Python 環境或受控資料獨立閱讀；它們不屬於單一主線章節的必要內容。

### Python 程式設計

- [pathlib 與檔案路徑](附錄-pathlib與檔案路徑.md)
- [json 與 csv](附錄-json與csv.md)
- [datetime 與 time](附錄-datetime與time.md)
- [函式、類別與責任分工](附錄-函式類別與責任分工.md)
- [例外處理與錯誤訊息](附錄-例外處理與錯誤訊息.md)
- [logging 與除錯紀錄](附錄-logging與除錯紀錄.md)
- [進階 logger 技巧與設計方法](附錄-進階logger技巧與設計方法.md)
- [測試概念與 pytest 實作](附錄-測試概念與pytest實作.md)

### pyserial 與序列埠

- [pyserial 連線、讀寫與逾時](附錄-pyserial連線讀寫與逾時.md)
- [pyserial 埠與資料除錯](附錄-pyserial埠與資料除錯.md)
- [pyserial 連線生命週期與 COM 埠除錯](附錄-pyserial連線生命週期與COM埠除錯.md)

### 資料處理與圖表

- [Pandas 基礎操作](附錄-Pandas基礎操作.md)
- [Pandas 清理、篩選與彙整](附錄-Pandas清理篩選與彙整.md)
- [Matplotlib 圖表基本元件](附錄-Matplotlib圖表基本元件.md)
- [Matplotlib 可讀性與輸出](附錄-Matplotlib可讀性與輸出.md)

### Gradio

- [Gradio 介面、元件與版面](附錄-Gradio元件與事件.md)
- [Gradio 事件、資料流與元件更新](附錄-Gradio事件資料流與元件更新.md)
- [Gradio 狀態、驗證與使用體驗](附錄-Gradio狀態驗證與使用體驗.md)

### 網路與安全

- [公私鑰加密、數位簽章與中間人攻擊防範](附錄-公私鑰加密數位簽章與中間人攻擊防範.md)

## 選做專題：RFID 門禁系統

- [專題第 0 章｜準備、安全與資料契約](選做-RFID專題準備安全與資料契約.md)
- [專題第 1 章｜RC522 RFID 與 SPI](選做-RC522-RFID與SPI.md)
- [專題第 2 章｜ESP32 MQTT 雙向訊息](選做-RFID專題/第2章/index.md)
  - [HiveMQ Cloud 免費測試設定示範](選做-RFID專題/第2章/hivemq-cloud-測試設定.md)
- [即將推出：專題第 3 章｜Python MQTT Gateway 與 FastAPI 初步概念](選做-Python-MQTT-Gateway與FastAPI.md)
- [即將推出：專題第 4 章｜LCD 狀態顯示](選做-LCD狀態顯示.md)
- [即將推出：專題第 5 章｜MIT App Inventor、ngrok 與手機無線控制](選做-MIT-App-Inventor-ngrok與手機無線控制.md)
- [即將推出：專題第 6 章｜端對端故障分流](選做-端對端故障分流.md)
- [即將推出：專題第 7 章｜真實門控系統的安全總結](選做-真實門控系統安全總結.md)

## 其他選做實作

- [即將推出：選做實作｜既有 BLE App 測試](選做-既有BLE-App測試.md)
- [即將推出：選做實作｜MAX7219 點陣模組](選做-MAX7219點陣模組.md)

## 資訊補充

- [常見 GUI 與 App 解決方案](資訊補充-Python後端GUI與App解決方案.md)
