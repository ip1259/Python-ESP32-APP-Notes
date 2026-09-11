# 延伸選讀：LCD 顯示程式導讀

> 對應主線：[I2C：使用 LCD1602A 顯示 DHT11 資料](05-i2c-lcd.md)

這一頁用主線的成功範例，說明 I2C、LCD 與 DHT11 如何依序工作。

## 讀完後你會知道

- 為什麼要先初始化 I2C、LCD，再開始讀取 DHT11。
- `printLine()` 如何避免較短的新文字留下舊字元。
- LCD 出現 `DHT read failed` 時，問題位於哪一段。

## 先知道這件事

主線已確認 LCD 的 SDA、SCL 使用 GPIO 21、22，並以掃描到的位址建立 16×2 LCD。以下快照對應主線檔案 `dht_lcd.ino`；只有導讀註解不同。

## 程式的資料流

```text
I2C 設定 → LCD 初始化 → DHT11 讀值 → 判斷是否有效 → 更新兩行 LCD
```

## 完整程式導讀

執行位置：Arduino IDE／ESP32
來源檔案：`dht_lcd.ino`

```cpp linenums="1"
// 【1】匯入 I2C、LCD 與 DHT11 函式庫
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

// 【2】集中保存已確認的 I2C 腳位、LCD 位址與 DHT11 腳位
const int SDA_PIN = 21;
const int SCL_PIN = 22;
const byte LCD_ADDRESS = 0x27;  // 改成 I2C 掃描結果
const int DHT_PIN = 4;

#define DHT_TYPE DHT11

// 【3】建立後續共用的 LCD 與 DHT11 物件
LiquidCrystal_I2C lcd(LCD_ADDRESS, 16, 2);
DHT dht(DHT_PIN, DHT_TYPE);

// 【4】更新一行前先清空舊內容，並限制在 LCD 的 16 格內
void printLine(int row, String text) {
  // 【4.1】移到指定列並以空白覆蓋舊字元
  lcd.setCursor(0, row);
  lcd.print("                ");
  // 【4.2】回到行首，印出最多 16 個字元的新文字
  lcd.setCursor(0, row);
  lcd.print(text.substring(0, 16));
}

// 【5】開機時初始化匯流排、顯示器與感測器
void setup() {
  Wire.begin(SDA_PIN, SCL_PIN);
  lcd.init();
  lcd.backlight();
  dht.begin();

  printLine(0, "Reading DHT11...");
}

// 【6】每約兩秒讀一次資料，依結果更新 LCD
void loop() {
  delay(2000);

  // 【6.1】讀取濕度與溫度
  float humidity = dht.readHumidity();
  float temperature = dht.readTemperature();

  // 【6.2】任一讀值不是數字時，不把它當成有效資料
  if (isnan(humidity) || isnan(temperature)) {
    printLine(0, "DHT read failed");
    printLine(1, "Check wiring");
    return;
  }

  // 【6.3】成功時把溫度與濕度各放入一行
  printLine(0, "T:" + String(temperature, 1) + " C");
  printLine(1, "H:" + String(humidity, 1) + " %");
}
```

## 執行順序與判讀

`setup()` 只在開機時執行一次；其中 `Wire.begin()` 先指定 I2C 的兩條訊號線，之後 LCD 才能初始化。`loop()` 持續重複：等待 DHT11 的讀取間隔、讀兩個數值、判斷是否有效，再決定顯示讀值或錯誤訊息。

| 看到的畫面 | 可先判斷的範圍 |
| --- | --- |
| `Reading DHT11...` 停留不變 | 程式已進入初始化；接著查看是否能正常進入讀值迴圈。 |
| `DHT read failed` | LCD 與 I2C 已能顯示文字；優先回到主線核對 DHT11。 |
| 溫溼度文字尾端有舊字 | `printLine()` 的清空與 16 字限制是避免此情形的關鍵。 |

## 常見誤解

### 「LCD 位址就是 GPIO 編號」

不是。GPIO 21、22 是 ESP32 的 I2C 訊號腳位；`0x27` 或 `0x3F` 是同一條 I2C 匯流排上 LCD 的裝置位址。

### 「背光亮起就代表 DHT11 正常」

不是。背光與固定文字只代表 LCD 部分可工作。程式顯示 `DHT read failed` 時，應回到主線的 DHT11 接線與讀值檢查。

## 重點整理

- I2C 初始化、LCD 初始化與 DHT11 初始化都在 `setup()` 完成。
- `printLine()` 先清除整行，再印出最多 16 個字元。
- `isnan()` 防止讀取失敗的數值被當成溫溼度顯示。
- LCD 與 DHT11 是兩段可分開判讀的功能。

## 回到主線

回到 [I2C：使用 LCD1602A 顯示 DHT11 資料](05-i2c-lcd.md)，繼續完成主線成果。
