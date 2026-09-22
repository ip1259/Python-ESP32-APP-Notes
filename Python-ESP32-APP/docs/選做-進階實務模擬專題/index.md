# 進階實務模擬專題：結合 MQTT、無線網路、Python 後端的門控模擬

這個選做專題用讀卡、網路服務與家庭情境效果，認識多個裝置如何分工合作。

!!! info "目前可閱讀內容"
    目前可閱讀前導概念，並可獨立完成 FastAPI Hello World；其餘工具頁會分別說明可用範圍與安全限制。

## 你會認識什麼

- ESP32 如何把讀卡事件送給 Gateway。
- Gateway 如何集中判斷狀態、保存最小紀錄並回覆結果。
- MQTT 如何把已完成判斷的家庭事件分送給 LED、顯示器或通知服務。
- App 與 BLE 在專題中分別負責 API 操作與近距離硬體設定。

## 開始前

- 建議先完成主線課程，特別是 ESP32、Python、CSV、LCD 與 Gradio 的內容。
- 專題中的手機、網路通道、MQTT Broker 與額外燈光裝置都是選做條件，不影響主線課程的完成。
- 本專題只模擬讀卡狀態與家庭情境效果，不控制門鎖、繼電器、電磁鎖或其他實體設備。

## 資料流概覽

```text
RC522 → ESP32 → HTTPS API → Gateway
                         ↓
                    狀態與最小紀錄
                         ↓
ESP32 LCD ← API 回應     MQTT 家庭事件 → LED／顯示器／通知服務

App → Gateway API
App → BLE 本地設定 → ESP32
```

讀卡結果由 Gateway 透過 API 回覆給 ESP32。MQTT 只處理判斷完成後的家庭情境事件，不用來傳送讀卡資料或決定是否通過。

## 專題安排

[前導技術概要](前導技術概要/index.md)目前提供 MQTT、HiveMQ Cloud、SPI、RC522、FastAPI、ngrok、Thunkable 與 BLE 的基礎閱讀，以及一個可獨立完成的 FastAPI Hello World 實作。其他選做主題可從各自頁面了解目前範圍；它們彼此不互為前置條件。

## 重點整理

- API 負責 ESP32 與 Gateway 的讀卡請求和結果回覆。
- Gateway 負責規則、紀錄與家庭事件的發布。
- MQTT 負責讓多個訂閱者各自處理家庭情境效果。
- App 不直接管理 MQTT；BLE 只用於附近的硬體連線設定。
