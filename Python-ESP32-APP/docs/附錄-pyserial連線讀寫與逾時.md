# 延伸選讀：pyserial 連線、讀寫與逾時

先用不需 ESP32 的受控迴路，觀察 pyserial 寫入、讀取、逾時與關閉連線的基本行為。

## 你會學到什麼

- 使用 `with` 開啟並自動關閉 pyserial 連線。
- 將文字編碼成 bytes 後用 `write()` 寫入，再以 `readline()` 讀回一行。
- 判斷空 bytes 是逾時結果，而不是合法資料。
- 了解 `reset_input_buffer()` 會丟棄緩衝資料的時機與風險。

## 開始前

- 建議先完成[UART：讓 Python 與 ESP32 雙向通訊](03-uart-python-esp32.md)，並確認已安裝 `pyserial`。
- 本頁用 pyserial 的 `loop://` 迴路網址驗證，不需 ESP32、USB 線或真實 COM 埠；它會把寫入內容送回同一條連線。
- 實機操作時，Arduino IDE 序列監控和 Python 不能同時開啟同一個 COM 埠；請先關閉序列監控，再回到主線操作。
- 範例只在記憶體中傳送固定文字，不會寫入檔案或控制硬體。

## 成功的樣子

執行範例後顯示：

```text
已寫入：ping
讀回：ping
逾時：沒有收到完整一行資料
清除緩衝後：沒有舊資料
```

這些結果來自受控 `loop://`，用來確認 API 行為；不是 ESP32 的實機驗收結果。

## 建立受控連線並讀回一行

目的：先在沒有硬體的情況下，確認「寫入 bytes → 讀回一行」的資料方向。

操作：建立 `pyserial_loop_demo.py`，貼上並執行下列程式。若你的練習環境尚未安裝套件，先回到第 1 節處理 `pyserial` 安裝。

```python
# 執行位置：Python／電腦
# 檔名：pyserial_loop_demo.py
import serial


message: str = "ping\n"

with serial.serial_for_url("loop://", timeout=0.1) as ser:
    written_count: int = ser.write(message.encode("utf-8"))
    received: bytes = ser.readline()

    print(f"已寫入：{written_count} bytes")
    print(f"讀回：{received.decode('utf-8').strip()}")
```

預期結果：顯示 `已寫入：5 bytes` 與 `讀回：ping`。`write()` 需要 bytes，所以用 `encode("utf-8")` 將文字轉換；`readline()` 讀到換行字元 `\n` 才回傳一行。主線 ESP32 也以一行一筆 JSON 的方式傳送資料。

## 讓逾時成為可判斷的結果

目的：知道沒有資料時 `readline()` 不會永遠卡住，並正確處理空 bytes。

操作：在同一個 `with` 區塊中，讀回 `ping` 後加入下列程式。

```python
# 執行位置：Python／電腦
# 檔名：pyserial_loop_demo.py
timed_out: bytes = ser.readline()

if timed_out == b"":
    print("逾時：沒有收到完整一行資料")
```

預期結果：約等待 `0.1` 秒後顯示 `逾時：沒有收到完整一行資料`。空 bytes 只表示這次期限內沒有讀到完整一行；不要把它解碼成感測值、寫入 CSV 或當成 ESP32 已回覆。

## 只在確認可丟棄時清除緩衝資料

目的：認識 `reset_input_buffer()` 的作用，以及它可能遺失資料的限制。

操作：使用下列完整程式取代原本檔案並執行。

```python
# 執行位置：Python／電腦
# 檔名：pyserial_loop_demo.py
import serial


with serial.serial_for_url("loop://", timeout=0.1) as ser:
    ser.write(b"old data\n")
    ser.reset_input_buffer()
    after_reset: bytes = ser.readline()

    if after_reset == b"":
        print("清除緩衝後：沒有舊資料")

    ser.write(b"new data\n")
    fresh_data: bytes = ser.readline()
    print(f"新資料：{fresh_data.decode('utf-8').strip()}")
```

預期結果：顯示 `清除緩衝後：沒有舊資料` 與 `新資料：new data`。這個範例先刻意寫入不需要的練習文字才清除它。真實 UART 的緩衝區可能有還沒處理的感測資料或狀態回覆；不確定內容時停止清除，先讀取、記錄並判斷來源。

## `with` 會關閉連線

目的：避免程式結束後仍占用序列埠。

操作：觀察所有範例都將連線放在 `with` 區塊內。

```python
# 執行位置：Python／電腦
# 檔名：pyserial_loop_demo.py
with serial.serial_for_url("loop://", timeout=0.1) as ser:
    print(ser.is_open)

print(ser.is_open)
```

預期結果：依序顯示 `True` 與 `False`。主線實機程式也應讓 `with` 管理連線；若真實 COM 埠仍被占用，先關閉 Arduino 序列監控與其他 Python 程式，再檢查埠名。

## 選做練習：改成送出一行 JSON

這是延伸練習，不做也不影響主線成果。

目的：練習將主線的一行一筆概念套用到受控迴路。

操作：將第一個範例的 `message` 改成 `'{"type":"ping"}\n'`，重新執行。

預期結果：讀回文字是 `{"type":"ping"}`。這只驗證序列文字的往返，沒有驗證 ESP32 是否理解命令。

## 完成時，應能確認

- 你知道 `write()` 寫入 bytes，`readline()` 讀到換行或逾時後回傳。
- 你能辨識空 bytes 是逾時，且不會把它當成感測資料。
- 你知道清除緩衝區會丟棄資料，只能在確認目標可捨棄時使用。
- 你能說明 `with` 結束後會關閉連線，避免佔用 COM 埠。

## 常見問題

### `ModuleNotFoundError: No module named 'serial'`

先回到[用 uv 建立 Python 虛擬環境](01-uv與虛擬環境.md)，確認在同一個專案環境安裝並執行 `pyserial`。套件名稱是 `pyserial`，匯入名稱才是 `serial`。

### 真實 COM 埠開啟失敗或被占用

先關閉 Arduino IDE 序列監控與其他正在讀取的程式，再確認 COM 埠和 baud rate。不要同時開兩個讀取器；若仍失敗，保留錯誤訊息並等待下一篇埠與資料除錯教材。

### `readline()` 一直得到空 bytes

在本頁的第二次讀取，這是 `timeout=0.1` 的預期結果。實機時先確認 ESP32 是否有傳送換行結尾的資料、baud rate 是否一致，再判斷要繼續等候或停止本次流程。

## 重點整理

- `loop://` 可重現 pyserial 的基本讀寫，不取代 ESP32 實機驗收。
- `timeout` 讓程式在沒有完整資料時有機會繼續判斷，而不是無限等待。
- 清除緩衝區會丟失資料；真實連線只能在明確知道要捨棄什麼時使用。
- `with` 是管理序列連線生命週期的安全預設。

## 下一步

回到[UART：讓 Python 與 ESP32 雙向通訊](03-uart-python-esp32.md)，在實機前先確認序列監控已關閉。接著可閱讀[延伸選讀：pyserial 埠與資料除錯](附錄-pyserial埠與資料除錯.md)，判斷候選埠、鮑率與解碼問題。
