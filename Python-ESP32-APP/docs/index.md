# Python、ESP32、App 整合設計

這是一套以筆記方式編寫的 AIoT 實作教材。你會使用 ESP32 蒐集感測資料，再用 Python 進行資料處理、視覺化與 Gradio 儀表板展示。

## 開始前

- 你已能在 Arduino IDE 上傳 ESP32 程式。
- 你已完成 Blink，並能讀取 DHT11 溫溼度資料。
- 請從 [用 uv 建立 Python 虛擬環境](01-uv與虛擬環境.md) 開始。

## 課程成果

完成課程後，你將能完成下列資料流：

```text
DHT11 → ESP32 → USB UART → Python → CSV／圖表／Gradio
```

後續單元會加入 LCD、8×8 點陣顯示器與手機端延伸控制。

## 教材目錄

1. [用 uv 建立 Python 虛擬環境](01-uv與虛擬環境.md)
2. [課程準備：認識 AIoT 資料流](02-課程準備.md)
3. [UART：讓 Python 與 ESP32 雙向通訊](03-uart-python-esp32.md)
4. [將感測資料存成 CSV 並繪製趨勢圖](04-python-感測資料視覺化.md)
5. [I2C：使用 LCD1602A 顯示 DHT11 資料](05-i2c-lcd.md)
6. [74HC595：辨識並驅動裸 8×8 點陣](06-74hc595-8x8點陣.md)
7. [Gradio：建立本機 AIoT 儀表板](07-gradio-aiot儀表板.md)
