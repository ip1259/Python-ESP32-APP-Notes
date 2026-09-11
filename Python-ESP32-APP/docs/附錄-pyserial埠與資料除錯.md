# 延伸選讀：pyserial 埠與資料除錯

列出候選序列埠、確認鮑率，並把亂碼或不完整資料留在可安全判斷的除錯階段。

## 你會學到什麼

- 使用 `serial.tools.list_ports` 列出電腦偵測到的候選埠。
- 知道候選埠不是 ESP32 身分證明，需以拔插與主線固定 JSON 確認。
- 確認 Python 與 ESP32 的 baud rate 要一致。
- 用安全方式顯示無法解碼的 bytes，不把它當成可信資料。

## 開始前

- 先閱讀[延伸選讀：pyserial 連線、讀寫與逾時](附錄-pyserial連線讀寫與逾時.md)。
- 實機前已關閉 Arduino IDE 序列監控；同一個 COM 埠一次只能由一個程式開啟。
- 本頁只列出候選埠，不會自動開啟任何真實 COM 埠，也不應公開貼出完整埠清單或原始序列資料。

## 成功的樣子

終端機會列出候選埠，例如 `候選埠：COM3 | USB Serial Device`；若沒有候選埠，顯示 `沒有偵測到序列埠`。兩者都是可判斷的結果，不表示應隨意指定 COM 號碼。

## 列出候選埠，不自動連線

目的：先取得候選清單，避免把第一個埠誤認成 ESP32。

```python
# 執行位置：Python／電腦
# 檔名：list_ports_demo.py
from serial.tools import list_ports


ports = list(list_ports.comports())
if not ports:
    print("沒有偵測到序列埠")
else:
    for port in ports:
        print(f"候選埠：{port.device} | {port.description}")
```

預期結果：列出零個或多個候選埠。插拔 ESP32 後比對清單變化，再以主線固定 JSON 確認資料；藍牙、其他 USB 裝置和虛擬埠也可能在清單內。

## 檢查鮑率與解碼結果

目的：將傳輸設定不一致與資料本身無法解讀分開判斷。

```python
# 執行位置：Python／電腦
# 檔名：decode_demo.py
import serial


expected_baud_rate: int = 115200
raw_line: bytes = b'{"type":"sensor","temp":26.5}\\n'
invalid_line: bytes = b"\\xff\\xfe\\n"

with serial.serial_for_url("loop://", baudrate=expected_baud_rate, timeout=0.1) as ser:
    print(f"目前 baud rate：{ser.baudrate}")
    print(f"合法文字：{raw_line.decode('utf-8').strip()}")
    print(f"除錯顯示：{invalid_line.decode('utf-8', errors='replace').strip()}")
```

預期結果：顯示 `目前 baud rate：115200`、合法 JSON 文字，以及含 `�` 的除錯顯示。取代字元只用來看見問題，不可寫入 CSV、畫圖或送回 ESP32。實機有亂碼時，先確認兩端都是 `115200`，再回到主線固定 JSON。

## 除錯順序

1. 關閉序列監控與其他 Python 程式，避免 COM 埠被占用。
2. 插拔 ESP32，比對候選清單；不要選第一個埠猜測。
3. 確認兩端鮑率一致，再上傳主線固定 JSON。
4. 保留最小錯誤訊息與資料樣本；仍無法判斷時，停止重試並回到[整合專題故障分流](附錄-故障排除索引.md)。

## 完成時，應能確認

- 你知道埠清單只提供候選值，不等於已找到 ESP32。
- 你能說明鮑率不一致可能造成亂碼或無法解析。
- 你不會把 `errors="replace"` 的取代字元當成感測資料。

## 常見問題

### 清單沒有 ESP32

先檢查 USB 線是否能傳資料、裝置是否上電，插拔後重新列出。不要改用隨機 COM 號碼；仍沒有變化時請告知教師或助教。

### 資料出現亂碼或 JSON 解析失敗

先確認兩端都是 `115200`，再回到 UART 主線的固定 JSON。若文字含 `�`，保留最小樣本作為除錯證據，不要寫入資料檔。

## 重點整理

- 候選 COM 埠需要以拔插與固定資料驗證，不能只看編號。
- 鮑率和文字解碼是兩個不同的檢查點。
- 除錯時先保留資料與錯誤訊息，避免清除或覆寫未知內容。

## 下一步

回到[UART：讓 Python 與 ESP32 雙向通訊](03-uart-python-esp32.md)完成實機確認。下一篇將介紹「即將推出：Pandas 基礎操作」。
