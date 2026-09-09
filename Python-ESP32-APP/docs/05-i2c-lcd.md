# I2C：使用 LCD1602A 顯示 DHT11 資料

這一頁讓 ESP32 直接把 DHT11 的溫度與濕度顯示在 LCD1602A。先用 I2C 掃描器找出 LCD 位址，再把已能讀取的 DHT11 程式加上顯示功能。

## 你會學到什麼

- 認識 I2C 的 SDA、SCL 與裝置位址。
- 用 I2C 掃描器確認 LCD1602A 的位址。
- 使用 LCD1602A 的兩行螢幕顯示溫度與濕度。
- 排查常見的 LCD 接線、位址與對比問題。

## 開始前

| 項目 | 需求 |
| --- | --- |
| 前置教材 | 已能讀取 DHT11；[UART：讓 Python 與 ESP32 雙向通訊](03-uart-python-esp32.md) |
| 硬體 | ESP32、DHT11、LCD1602A 附 I2C 模組、麵包板與杜邦線 |
| Arduino 函式庫 | `DHT sensor library`、`LiquidCrystal I2C` |
| ESP32 I2C 腳位 | SDA：GPIO 21；SCL：GPIO 22 |

!!! warning "先斷開 USB 再接線"
    先拔掉 ESP32 的 USB 資料線再改接線，完成後再接回電腦。LCD 的 SDA、SCL 不可接到 5V；本頁核心接法以 3.3V 供電，避免 I2C 訊號電壓高於 ESP32 可承受範圍。

在 Arduino IDE 的「程式庫管理員」搜尋並安裝 `LiquidCrystal I2C`。本頁程式使用 `LiquidCrystal_I2C.h`、`lcd.init()` 與 `lcd.backlight()` 這組常見 API。

!!! note "螢幕文字先用英文"
    一般 LCD1602A 沒有中文字型。本頁以 `T`、`H`、`C` 與 `%` 顯示，讓每台電腦與模組都能得到一致結果。

## 成功的樣子

LCD 第一行顯示溫度，第二行顯示濕度，畫面類似：

```text
T:25.0 C
H:60.0 %
```

## 接線

LCD1602A 的 I2C 模組通常有四個接腳：`GND`、`VCC`、`SDA`、`SCL`。

| LCD1602A I2C 模組 | ESP32 | 說明 |
| --- | --- | --- |
| GND | GND | 共地 |
| VCC | 3V3 | 核心教材使用 3.3V |
| SDA | GPIO 21 | I2C 資料線 |
| SCL | GPIO 22 | I2C 時脈線 |

!!! warning "不要直接把 I2C 訊號拉到 5V"
    有些 LCD 背包在接 5V 時，會把 SDA、SCL 也拉到 5V。這可能損壞 ESP32 的 GPIO。若模組在 3.3V 下無法正常工作，先請教師確認模組與轉換方式；不要自行把 VCC 改接 5V 後繼續使用原接線。

## 步驟 1：掃描 LCD 的 I2C 位址

**目的：** 先找出這一片 LCD 實際使用的位址。課程套件最常見的是 `0x27` 或 `0x3F`，但要以掃描結果為準。

建立檔案 `i2c_scanner.ino`，上傳後開啟序列監控視窗，baud rate 設為 `115200`。

執行位置：Arduino IDE／ESP32  
檔案：`i2c_scanner.ino`

```cpp
#include <Wire.h>

const int SDA_PIN = 21;
const int SCL_PIN = 22;

void setup() {
  Serial.begin(115200);
  Wire.begin(SDA_PIN, SCL_PIN);
  Serial.println("I2C scanner started");
}

void loop() {
  int found_count = 0;

  for (byte address = 1; address < 127; address++) {
    Wire.beginTransmission(address);
    byte error = Wire.endTransmission();

    if (error == 0) {
      Serial.printf("Found I2C device at 0x%02X\n", address);
      found_count++;
    }
  }

  if (found_count == 0) {
    Serial.println("No I2C device found");
  }

  delay(5000);
}
```

預期結果：序列監控視窗每 5 秒出現一次 `Found I2C device at 0x27` 或 `Found I2C device at 0x3F`。

`0x27`、`0x3F` 是 I2C 位址，不是 GPIO 編號。下一步的 `LCD_ADDRESS` 必須填入你掃描到的值。

## 步驟 2：先顯示固定文字

**目的：** 在加入感測器以前，先確認 LCD 位址、接線與背光都正常。

建立檔案 `lcd_hello.ino`。若掃描結果是 `0x3F`，請把程式中的 `0x27` 改成 `0x3F`。

執行位置：Arduino IDE／ESP32  
檔案：`lcd_hello.ino`

```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int SDA_PIN = 21;
const int SCL_PIN = 22;
const byte LCD_ADDRESS = 0x27;  // 改成 I2C 掃描結果

LiquidCrystal_I2C lcd(LCD_ADDRESS, 16, 2);

void setup() {
  Wire.begin(SDA_PIN, SCL_PIN);
  lcd.init();
  lcd.backlight();

  lcd.setCursor(0, 0);
  lcd.print("I2C LCD ready");
  lcd.setCursor(0, 1);
  lcd.print("ESP32 OK");
}

void loop() {
}
```

預期結果：背光亮起，第一行顯示 `I2C LCD ready`，第二行顯示 `ESP32 OK`。

若背光亮起但沒有文字，先轉動 I2C 模組上的小型可變電阻，慢慢調整對比；調整時不要用力壓螺絲起子。

## 步驟 3：顯示 DHT11 溫溼度

**目的：** 每約 2 秒讀取一次 DHT11，並更新 LCD 的兩行內容。

建立檔案 `dht_lcd.ino`。DHT11 接腳沿用前面已成功讀值的接法；以下範例使用 GPIO 4。

執行位置：Arduino IDE／ESP32  
檔案：`dht_lcd.ino`

```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

const int SDA_PIN = 21;
const int SCL_PIN = 22;
const byte LCD_ADDRESS = 0x27;  // 改成 I2C 掃描結果
const int DHT_PIN = 4;

#define DHT_TYPE DHT11

LiquidCrystal_I2C lcd(LCD_ADDRESS, 16, 2);
DHT dht(DHT_PIN, DHT_TYPE);

void printLine(int row, String text) {
  lcd.setCursor(0, row);
  lcd.print("                ");
  lcd.setCursor(0, row);
  lcd.print(text.substring(0, 16));
}

void setup() {
  Wire.begin(SDA_PIN, SCL_PIN);
  lcd.init();
  lcd.backlight();
  dht.begin();

  printLine(0, "Reading DHT11...");
}

void loop() {
  delay(2000);

  float humidity = dht.readHumidity();
  float temperature = dht.readTemperature();

  if (isnan(humidity) || isnan(temperature)) {
    printLine(0, "DHT read failed");
    printLine(1, "Check wiring");
    return;
  }

  printLine(0, "T:" + String(temperature, 1) + " C");
  printLine(1, "H:" + String(humidity, 1) + " %");
}
```

`printLine()` 會先用空白覆蓋整行，再印出新資料。這可避免新數字比舊數字短時，畫面留下舊字元。

## 小練習：補上第二行的濕度

**目標：** 將已讀取的 `humidity` 顯示在 LCD 第二行。

請在 `# TODO` 處補上程式。其餘程式不用修改。

執行位置：Arduino IDE／ESP32  
檔案：`practice_lcd_humidity.ino`

```cpp
float humidity = 60.0;

// 已完成 lcd 初始化，且 printLine(row, text) 可以使用
printLine(0, "T:25.0 C");

// TODO: 在第 2 行顯示 H:60.0 % 的格式
```

預期結果：LCD 第二行顯示 `H:60.0 %`。

<details>
<summary>查看參考解答</summary>

```cpp
printLine(1, "H:" + String(humidity, 1) + " %");
```

LCD 的列從 `0` 開始計算，所以第二行的 `row` 是 `1`。`String(humidity, 1)` 讓數字保留一位小數。
</details>

## 完成時，應能確認

- I2C 掃描器能找到 `0x27` 或 `0x3F` 的 LCD 位址。
- LCD1602A 能顯示兩行固定文字。
- LCD 能每約 2 秒更新一次 DHT11 的溫度與濕度。
- 你能說出 SDA、SCL 是 I2C 訊號線，而位址不是 GPIO 編號。
- 遇到空白畫面時，知道依序檢查接線、位址與對比電阻。

## 常見問題

### 掃描結果是 `No I2C device found`

1. 確認 LCD 的 `GND` 與 ESP32 的 `GND` 有接在一起。
2. 確認 `SDA` 接 GPIO 21、`SCL` 接 GPIO 22，兩條線沒有對調。
3. 確認 LCD 的 VCC 接到 3V3，且 USB 已接回 ESP32。
4. 不要只看線的顏色；逐一核對模組絲印上的腳位名稱。

### 掃描得到位址，但 LCD 沒有文字

1. 將 `LCD_ADDRESS` 改成掃描器實際顯示的值。
2. 確認 LCD 建構子為 `LiquidCrystal_I2C lcd(LCD_ADDRESS, 16, 2);`。
3. 慢慢調整 I2C 模組上的對比可變電阻。
4. 確認已安裝的函式庫是提供 `LiquidCrystal_I2C.h` 的 `LiquidCrystal I2C`。

### LCD 顯示 `DHT read failed`

這表示 LCD 已能工作，問題在 DHT11 讀值。請回到已成功的 DHT11 接線，核對 VCC、GND、DATA 與 GPIO 4；改接線前先斷開 USB。

### 編譯時顯示找不到 `LiquidCrystal_I2C.h`

開啟 Arduino IDE 的「程式庫管理員」，搜尋並安裝 `LiquidCrystal I2C` 後再編譯。安裝完成仍有錯誤時，重新啟動 Arduino IDE，再確認程式庫是否已列在「已安裝」。

## 額外補充：5V LCD 的雙向 I2C 電平轉換

有些 LCD1602A 在 3.3V 下背光雖然會亮，但文字對比極差或無法正常顯示。這時 LCD 可使用 5V 供電，**但不能把 SDA、SCL 直接接到 ESP32**；ESP32 GPIO 只能承受 3.3V 範圍的訊號。

本節補充需要一個「雙向 I2C 電平轉換器」（常見為 2 或 4 通道、BSS138 類型）。它**不在本課材料套件中，不是必做內容**；沒有這個模組時，仍以本頁 3.3V 核心接法為準。

!!! danger "畫面能顯示，不代表 5V 直連是安全的"
    有些人把 LCD 的 `VCC` 接到 5V、再直接把 SDA／SCL 接 ESP32 後，短時間仍能掃描與顯示文字。這只是「暫時能運作」，不是正確接法。許多 I2C LCD 背包會把 SDA、SCL 上拉到自己的 VCC；當 VCC 是 5V 時，ESP32 GPIO 就可能被拉到 5V，超過可承受範圍。持續使用可能造成累積損傷、通訊不穩或 GPIO 永久失效。即使目前看起來正常，也要斷電後改用本節的雙向電平轉換接法。

!!! warning "不能只在 SDA、SCL 各接一組電阻分壓"
    一般電阻分壓適合單向訊號，但 I2C 的 SDA 是雙向開放汲極訊號，SCL 也可能被裝置拉低。電阻分壓會破壞其中一個方向的低電位傳遞，通訊可能不穩或完全失敗。請使用標示為「雙向 I2C」的電平轉換器。

### 5V LCD 的額外接線

以常見標示 `HV`／`LV`、`HV1`／`LV1`、`HV2`／`LV2` 的雙向電平轉換模組為例：

| 連接位置 | 接到哪裡 | 用途 |
| --- | --- | --- |
| ESP32 `3V3` | 電平轉換器 `LV` | 低電壓側參考電壓 |
| 5V 電源正極 | 電平轉換器 `HV`、LCD `VCC` | 高電壓側與 LCD 供電 |
| ESP32 `GND` | 電平轉換器 `GND`、5V 電源負極、LCD `GND` | 所有裝置共地 |
| ESP32 GPIO 21 | 電平轉換器 `LV1` | 低電壓側 SDA |
| 電平轉換器 `HV1` | LCD `SDA` | 高電壓側 SDA |
| ESP32 GPIO 22 | 電平轉換器 `LV2` | 低電壓側 SCL |
| 電平轉換器 `HV2` | LCD `SCL` | 高電壓側 SCL |

接好後，I2C 掃描器與 LCD 程式都不需要修改；仍使用 GPIO 21／22 與掃描到的位址。若掃描不到，先關閉電源，依序檢查 `LV` 是否為 3.3V、`HV` 是否為 5V，以及四個裝置的 GND 是否確實相連。

!!! warning "5V 只能停在高電壓側"
    5V 只能接到 LCD 的 `VCC` 與電平轉換器的 `HV`。不可接到 ESP32 的 `3V3`、GPIO 21、GPIO 22、電平轉換器的 `LV` 或 `LV1`／`LV2`。

## 重點整理

- I2C 用 SDA、SCL 兩條訊號線連接裝置；同一組匯流排靠位址區分裝置。
- LCD1602A 的常見位址是 `0x27` 或 `0x3F`，但必須以掃描結果為準。
- 先顯示固定文字，再整合 DHT11，能更快定位問題在 LCD 還是感測器。
- ESP32 的 I2C 訊號必須維持 3.3V 安全範圍。
- 需要讓 LCD 使用 5V 時，必須以雙向 I2C 電平轉換器隔開高、低電壓側，不能用一般電阻分壓取代。

## 延伸練習（可選）

當溫度大於或等於 30°C 時，在第二行改顯示 `HOT`；否則顯示濕度。先在序列監控視窗印出條件判斷結果，再改 LCD 顯示內容。

## 下一步

前往 [74HC595：辨識並驅動裸 8×8 點陣](06-74hc595-8x8點陣.md)。
