# 資訊補充：常見 GUI 與 App 解決方案

這一頁帶你認識 AIoT 專案常見的 GUI 與 App 解決方案。目的是知道不同工具各自適合什麼情境，以及未來想深入時可以從哪裡開始；本頁**不要求安裝、寫作業或完成 App**。

!!! info "這不是主線課程"
    本課主線仍以 ESP32、USB UART、Python、CSV、圖表與 Gradio 儀表板為主。本頁只提供產業工具的定位與官方學習入口；不進行 GUI／App 框架教學，也不構成本課的實作要求。

## 為什麼需要認識這些工具

完成 ESP32 的資料傳輸練習後，你可能會好奇：若要做成手機 App、電腦桌面程式或網頁控制面板，業界通常使用哪些工具？以下方案提供的是不同的「畫面製作方式」。它們都可以搭配 HTTP 或 MQTT 等資料傳輸方式，但本課不會要求你選擇、安裝或實作其中任何一種。

## 常見 GUI 與 App 解決方案

「GUI」是圖形使用者介面，例如按鈕、文字、圖表與畫面配置。產品選用的技術，會依目標裝置與團隊技術而不同：

| 方案 | 常見用途 | 與 AIoT 資料傳輸的關聯 | 官方資源入口 |
| --- | --- | --- | --- |
| Qt | 桌面程式、嵌入式裝置介面，也可延伸到多平台應用 | 可將感測資料顯示成電腦上的控制面板 | [Qt 官方入門](https://doc.qt.io/qt-6/get-and-install-qt.html) · [Qt 語言概覽](https://doc.qt.io/qtforpython-6.10/overviews/qtdoc-qtlanguages.html) · [Qt for Python](https://doc.qt.io/qtforpython-6/quickstart.html) |
| Rust + Slint | 重視效能與資源使用的跨平台或嵌入式 GUI | 可把裝置或服務送來的感測資料呈現在原生介面 | [Rust 學習入口](https://www.rust-lang.org/learn) · [Slint 語言整合](https://docs.slint.dev/latest/docs/slint/language-integrations/) · [Slint Rust 官方指南](https://docs.slint.dev/latest/docs/rust/slint/) |
| Dart + Flutter | Android、iOS、網頁與桌面 App 的跨平台介面 | 手機 App 可透過 HTTP 或 MQTT 讀取資料、送出操作 | [Dart 官方文件](https://dart.dev/docs) · [Flutter 學習入口](https://docs.flutter.dev/learn) |
| Vue | 互動式網頁前端 | 網頁可顯示感測資料、圖表與控制面板 | [Vue 指南](https://vuejs.org/guide/introduction.html) · [Vue 快速開始](https://vuejs.org/guide/quick-start.html) |

這些方案沒有「哪一個一定最好」。先問自己要做的是手機 App、電腦桌面程式、網頁，還是嵌入式螢幕；再依團隊使用的語言、裝置與資料傳輸方式做選擇。

## 各方案在專案中的位置

### Qt：偏向成熟的桌面與嵌入式介面生態

Qt 常見於電腦桌面工具、工業控制畫面和嵌入式裝置介面。當專案需要在電腦或設備螢幕上持續顯示感測數值、圖表與控制按鈕時，Qt 是可能的選項。Qt 生態主要常見於 C++ 與 QML，也有 Qt for Python（PySide）可讓 Python 程式建立介面。

Qt 的「可用多種語言」不表示所有語言的功能、文件與發布方式都完全一樣：

| 語言或組合 | 在 Qt 中的角色 | 選擇前要知道的限制 |
| --- | --- | --- |
| C++ | Qt 的核心 API 與大量範例都以 C++ 為基礎 | 需要面對 C++、編譯工具與原生程式的建置流程；許多底層或進階文件會以 C++ 為主。 |
| QML + JavaScript | QML 用來描述介面；可在 QML 中使用 JavaScript 表達畫面行為 | QML 不是一般網頁 HTML，也不是 Node.js 應用程式；通常仍要搭配 C++ 或 Python 處理較完整的應用邏輯與系統整合。 |
| Python + PySide6 | Qt 官方提供的 Python 綁定，可讓 Python 呼叫 Qt API | 可避開部分 C++ 程式碼，但仍需要理解 Qt 的物件、事件與部署方式；個別 API 的寫法、範例與套件發布方式可能和 C++ 文件不同。 |
| 其他社群綁定 | 社群可能提供其他程式語言與 Qt 的連接方式 | 不屬於本頁的建議路線。採用前必須自行確認維護狀態、Qt 版本相容性、可用模組、授權與部署支援，不能假設與官方 API 功能相同。 |

可以把 Qt 想成「核心以 C++ 建立的跨平台工具組」：QML 專門描述畫面，Python 則可透過官方 PySide6 綁定使用 Qt。這些語言可以在同一個專案協作，但混合越多，資料型別、事件、打包與除錯的邊界也越需要明確。

### Rust + Slint：偏向原生、跨平台與資源受限的介面

Rust 是系統程式語言；Slint 則可搭配 Rust 製作宣告式介面。這個組合適合想了解原生應用程式、跨平台介面或嵌入式畫面的人。閱讀時要先分清楚：Rust 是程式語言，Slint 是建立畫面的工具；兩者需要分別學習。

Slint 也不只可配合 Rust。它以 `.slint` 檔案描述畫面，再由宿主語言（host language）處理資料、事件與業務邏輯。官方目前提供下列語言整合，但支援層級不同：

| 宿主語言 | 在 Slint 中的定位 | 選擇前要知道的限制 |
| --- | --- | --- |
| Rust | 主要且成熟的整合路線之一 | 需要先具備 Rust、Cargo 與原生建置的基本概念；適合希望以 Rust 做應用邏輯的專案。 |
| C++ | 主要且成熟的整合路線之一 | 需要 C++20 與 CMake 等原生工具鏈；建置設定與部署通常比純網頁專案更需要處理平台差異。 |
| TypeScript／JavaScript | 官方列為 beta 的整合 | beta 代表 API、工具與相容性仍可能調整；開始專案前應先確認目標平台、套件版本與需要的功能是否支援。 |
| Python | 官方列為 beta 的整合 | 適合探索 Python 與 Slint 的連接方式，但不應假設它與 Rust／C++ 路線擁有完全相同的成熟度、範例數量或功能覆蓋。 |

Slint 的介面描述語言與宿主語言是兩層：`.slint` 檔案負責畫面，Rust、C++、JavaScript 或 Python 負責接收事件與提供資料。這種分工有助於重用畫面，但也代表專案至少要理解兩種語言或格式。不同宿主語言可用的 renderer、平台後端與建置工具也可能不同，選用前應回到官方的語言整合與目標平台文件確認。

### Dart + Flutter：偏向一套程式支援多種 App 平台

Dart 是程式語言，Flutter 是使用 Dart 建立介面的工具。它常被用來製作手機 App，也能目標到網頁與桌面。若未來希望同一個 AIoT 操作介面能在 Android 與 iOS 使用，Flutter 是常見的探索方向；資料可再透過 HTTP 或 MQTT 與裝置系統交換。

### Vue：偏向瀏覽器中的互動式網頁

Vue 是網頁前端框架，使用者以瀏覽器開啟介面。它適合做資料看板、管理頁面或操作面板；畫面通常向後端 API 取得資料，或透過即時通訊方式更新。它與本課的 Gradio 一樣都能在瀏覽器呈現畫面，但定位不同：Gradio 適合快速 Python 原型，Vue 則是較完整的網頁前端技術路線。

!!! note "這些是探索入口，不是選課要求"
    表格中的連結供你未來依興趣自行閱讀。你不需要現在比較語言、安裝開發環境，或決定要使用哪一個框架；本課的實作與驗收仍以既定 ESP32 與 Python 主線為準。

## 選擇時可先問的問題

| 如果你想做的成果是… | 可以先認識的方向 | 為什麼 |
| --- | --- | --- |
| 手機上的跨平台操作 App | Dart + Flutter | 以同一套 Dart 程式建立多種平台的介面 |
| 電腦上的控制面板或設備螢幕 | Qt | 著重桌面與嵌入式介面的成熟生態 |
| 原生、跨平台或資源較受限的介面 | Rust + Slint | 將 Rust 與宣告式 GUI 工具結合 |
| 從瀏覽器開啟的資料看板 | Vue | 專注互動式網頁前端 |

這張表只幫你建立「從需求找方向」的概念；真正的選擇還會受到團隊經驗、既有系統、部署方式與維護成本影響。

## 與 MQTT 的關係

HTTP 常見於「送出請求、取得回覆」的情境；MQTT 則讓 ESP32、後端與 App 透過 broker 的 topic 發布與訂閱訊息。

```text
ESP32 ──發布──> MQTT broker <──訂閱── Python 後端／手機 App／網頁
```

MQTT 不會自動讓遠端控制變得安全。帳密、權限、TLS、topic 設計與設備控制風險，都需要在專門的 MQTT 與資安規劃中處理。

## 重點整理

- Qt、Slint、Flutter 與 Vue 都能製作使用者介面；差別主要在目標平台、使用語言與專案情境。
- HTTP 與 MQTT 是資料傳輸方式；GUI 或 App 框架則負責讓使用者看見資料、送出操作。
- Rust + Slint、Dart + Flutter 都包含「程式語言 + 介面工具」兩個部分，可從表格的兩個官方入口分別認識。
- 本頁只作產業方案導覽；不要求安裝、比較或實作任何 GUI／App 框架。

## 回到課程主線

完成主線的資料流練習後，再依興趣閱讀「手機 App 與無線控制」中的 BLE 與 MIT App Inventor 選做內容。
