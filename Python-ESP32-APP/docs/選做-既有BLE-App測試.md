# 即將推出：選做實作｜LightBlue 與 ESP32 交換固定資料

> 預告頁｜本頁將使用商店版 LightBlue 與既有 ESP32 BLE 程式交換固定練習文字。

!!! info "即將推出"
    LightBlue 的基本用途已確認，但既有 ESP32 韌體、UUID、Characteristic 權限與資料格式仍需完成實機確認。本頁尚未提供可直接操作的步驟。

本頁預計使用 Punch Through 的 LightBlue。Android 商店名稱為「LightBlue® — Bluetooth LE」，iPhone 與 iPad 商店名稱為「LightBlue®」。App 的價格、上架地區與畫面可能更新，安裝前需以自己手機商店顯示的資訊為準。

正式實作只會包含掃描已知 ESP32、連線指定 Service／Characteristic，以及交換固定且不含敏感資料的練習文字。不會使用 Wi-Fi 密碼、token、UID、MQTT、Gateway API 或設備控制命令。

本頁不需要 FastAPI、ngrok、Thunkable、RC522 或其他前導實作。目前可先閱讀[BLE：手機與 ESP32 的近距離設定概念](選做-進階實務模擬專題/前導技術概要/ble近距離設定概念.md)。
