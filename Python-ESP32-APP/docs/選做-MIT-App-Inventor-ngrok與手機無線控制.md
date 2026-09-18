# 即將推出：專題第 5 章｜Thunkable、ngrok 與手機無線控制

這是 RFID 門禁系統的選做章節預告；預計用 Thunkable 製作手機上的狀態查詢介面。

!!! info "本頁目前是預告"

    正式內容會說明手機 App 如何讀取 Gateway API 的最新狀態，以及如何在受控範圍內使用暫時 HTTPS 網址。本頁尚未提供帳號設定、App 元件設定、公開網址或控制設備的操作。

## 預計內容

本章預計使用 Thunkable 製作手機 App，讀取 Gateway API 的最新狀態，並透過 ngrok 建立暫時的 HTTPS 網址，讓手機在受控範圍內查詢狀態。

正式內容發布前，會先確認 Thunkable、ngrok、Gateway API、手機與網路環境能否以一致版本安全地配合。

!!! warning "帳密與公開網址的安全界線"

    不要把本機位址、帳密、token、真實 RFID 資料或控制指令放入 App、截圖或 Git。暫時網址只能在受控測試時使用，不要分享給未授權的人；測試結束後應停止 tunnel。

## 下一步

先完成[專題第 3 章：Gateway 名單與最小紀錄](選做-RFID專題/第3章/index.md)，了解 App 讀取的最新狀態來自何處。
