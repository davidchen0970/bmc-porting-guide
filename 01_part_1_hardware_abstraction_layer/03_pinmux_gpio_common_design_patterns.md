# 第 3 章　Pinmux 與 GPIO 通用設計模式

> 現場正在排查故障，不需要依序閱讀本章時，請直接前往：[3.20 附錄：現場急救索引](#gpio-field-first-aid)。

## 適用範圍

本章說明 BMC 平台上的 Pinmux、GPIO、GPI、GPO、Device Tree、U-Boot、Linux kernel 與 OpenBMC 之間的關係。

本章先回答最基本的問題，再逐步加入啟動階段、極性、GPIO expander、interrupt 與 OpenBMC mapping。第一次閱讀不需要同時理解所有子系統。

## 本章假設

本章假設讀者已經知道：

- Schematic 是電路圖。
- SoC 是系統晶片。
- High 與 Low 是實體電位。
- U-Boot 是常見的 Bootloader。
- Linux kernel 會依 Device Tree 啟用硬體。

本章不假設讀者已經理解：

- Pin、Pad、Ball、Net 的差異。
- Pinmux 與 GPIO 的差異。
- GPI、GPO、GPIO 的差異。
- Physical value 與 logical value 的差異。
- U-Boot 與 kernel 為何要分開設定 GPIO。
- GPIO consumer、GPIO hog 或 libgpiod。

## 這一章真正要解決什麼問題

看到一條訊號，例如：

```text
PSU0_PRSNT_N
```

我們希望能逐項回答：

1. 它在電路圖上接到哪裡？
2. 它接到 SoC、GPIO expander，還是 CPLD？
3. 它是輸入 GPI、輸出 GPO，還是可切換方向的 GPIO？
4. Pinmux 是否真的選成 GPIO？
5. Low 與 High 各代表什麼功能狀態？
6. Reset 後的預設電位是什麼？
7. U-Boot 期間由誰設定？
8. kernel 啟動後由誰重新設定？
9. OpenBMC 最後把它轉成哪個 D-Bus 或 Redfish 狀態？
10. 若結果錯誤，應從哪一層開始查？

只知道「GPIO number 是多少」不足以回答以上問題。

## 閱讀方式

### 第一輪必讀

- 3.1 一條訊號經過哪些層。
- 3.2 GPIO、GPI 與 GPO。
- 3.3 Pinmux 是什麼。
- 3.4 Direction、電位與功能狀態。
- 3.5 U-Boot 與 kernel 必須分開看。

讀完第一輪後，應能說明「這條線是誰的輸入或輸出，以及目前由哪個啟動階段控制」。

### 第二輪必讀

- 3.6 Linux GPIO controller、gpiochip 與 line。
- 3.7 Device Tree 如何描述 GPIO。
- 3.8 GPIO consumer 與 hog。
- 3.9 使用 libgpiod 查看狀態。

讀完第二輪後，應能將 Schematic 訊號追到 Linux GPIO line。

### 第三輪延伸

- 3.10 GPIO expander。
- 3.11 GPIO interrupt。
- 3.12 Debounce。
- 3.13 OpenBMC mapping。
- 3.14 完整排查流程。

## 快速導覽

- [3.1 先看一條訊號經過哪些層](#31-先看一條訊號經過哪些層)
- [3.2 GPIO、GPI 與 GPO](#32-gpiogpi-與-gpo)
- [3.3 Pinmux 是什麼](#33-pinmux-是什麼)
- [3.4 Direction、電位與功能狀態](#34-direction電位與功能狀態)
- [3.5 U-Boot 與 kernel 必須分開看](#35-u-boot-與-kernel-必須分開看)
- [3.6 Linux GPIO controller、gpiochip 與 line](#36-linux-gpio-controllergpiochip-與-line)
- [3.7 Device Tree 如何描述 GPIO](#37-device-tree-如何描述-gpio)
- [3.8 GPIO consumer 與 GPIO hog](#38-gpio-consumer-與-gpio-hog)
- [3.9 使用 libgpiod 查看 GPIO](#39-使用-libgpiod-查看-gpio)
- [3.10 GPIO expander](#310-gpio-expander)
- [3.11 GPIO interrupt](#311-gpio-interrupt)
- [3.12 Debounce](#312-debounce)
- [3.13 OpenBMC mapping](#313-openbmc-mapping)
- [3.14 固定排查流程](#314-固定排查流程)
- [3.15 完整案例](#315-完整案例)
- [3.16 常見問題與判讀](#316-常見問題與判讀)
- [3.17 平台訊號紀錄格式](#317-平台訊號紀錄格式)
- [3.18 本章檢查表](#318-本章檢查表)
- [3.19 本章重點](#319-本章重點)

## 3.1 先看一條訊號經過哪些層

一條 BMC 訊號，不只有「GPIO」這一層。

```text
板上元件或接點
    ↓
Schematic net
    ↓
SoC ball / Expander port / CPLD bit
    ↓
Pad 與 Pinmux
    ↓
GPIO controller
    ↓
Linux GPIO line
    ↓
Kernel driver 或 userspace service
    ↓
D-Bus / Redfish / IPMI / Event
```

以 PSU presence 為例：

```text
PSU 插入
    ↓
PSU0_PRSNT_N 被拉成 Low
    ↓
GPIO controller 讀到實體值 0
    ↓
ACTIVE_LOW 將它解讀為 logical active
    ↓
OpenBMC 設定 Present = true
```

每一層回答的問題不同。

### 3.1.1 Net、Ball、Pad 與 GPIO line

#### Net

Net 是 Schematic 上的訊號名稱，例如：

```text
PSU0_PRSNT_N
BIOS_WP_N
HOST_PWR_EN
```

Net 描述的是板上電氣連線。

#### Ball 或 Pin

Ball 或 Pin 是 SoC 封裝上的實體接點，例如 A12。

同一個 net 可能直接接到 SoC ball，也可能先經過 buffer、level shifter、CPLD 或 GPIO expander。

#### Pad

Pad 是 SoC 內部對應該 ball 的 I/O cell。Pinmux、pull-up、pull-down、drive strength 等設定通常作用在 pad。

#### GPIO line

GPIO line 是 Linux GPIO controller 內的一條線。它以 gpiochip 與 line offset 表示，例如：

```text
gpiochip0 line 24
```

這個 24 不是 SoC ball number，也不一定是舊式 global GPIO number。

### 3.1.2 為什麼不能只記 GPIO number

因為以下資訊都可能改變或被誤解：

- gpiochip 編號可能隨 driver probe 順序改變。
- 同一個 SoC ball 可以切換不同 Pinmux function。
- 同一條 net 中間可能有反相器。
- U-Boot 與 kernel 可能使用不同設定。
- Linux 讀到的 logical value 可能已套用 ACTIVE_LOW。

完整紀錄至少應包含：

```text
Schematic net
SoC ball / Expander port / CPLD bit
Pinmux function
GPI / GPO / GPIO direction
Active level
Reset default
U-Boot owner 與設定
Kernel owner 與設定
gpiochip label + line offset + line name
OpenBMC object / property
```

## 3.2 GPIO、GPI 與 GPO

### 3.2.1 GPIO 是什麼

GPIO 是 General-Purpose Input/Output，中文可理解為「通用數位輸入／輸出」。

GPIO 通常具有兩個基本能力：

- 當作 input，讀取外部電位。
- 當作 output，驅動外部電位。

但「硬體支援 input/output」不代表軟體可以隨時任意切換。某條線在產品設計中通常已有固定用途與 owner。

### 3.2.2 GPI 是什麼

GPI 是 General-Purpose Input，也就是只從 BMC 的角度讀取外部狀態。

資料方向如下：

```text
外部電路 → BMC GPIO input
```

常見 GPI：

- Presence，例如 PSU、Fan、Riser 是否插入。
- Power good，例如 PSU_PGOOD、VR_PGOOD。
- Fault，例如 VR_FAULT_N。
- Button status，例如 POWER_BUTTON_N。
- Chassis intrusion。
- Interrupt input。
- Host state，例如 PLTRST_N、SLP_S3_N。

GPI 的重點是：

- BMC 不應主動驅動這條線。
- 需要確認外部 pull 或外部驅動來源。
- 若 input floating，讀值可能不穩定。
- 若訊號會抖動，可能需要 debounce。

#### GPI 範例

```text
PSU0_PRSNT_N = Low  → PSU 已插入
PSU0_PRSNT_N = High → PSU 未插入
```

從 BMC GPIO controller 看，實體值可能是：

```text
插入：0
拔除：1
```

若 Device Tree 標成 `GPIO_ACTIVE_LOW`，consumer 看到的 logical value 則可能是：

```text
插入：1，也就是 active
拔除：0，也就是 inactive
```

### 3.2.3 GPO 是什麼

GPO 是 General-Purpose Output，也就是由 BMC 驅動外部電路。

資料方向如下：

```text
BMC GPIO output → 外部電路
```

常見 GPO：

- Power enable。
- Reset control。
- Write protect。
- Mux select。
- LED control。
- Power button pulse。
- Device enable。

GPO 的重點是：

- 必須先定義安全預設值。
- 必須知道 active level。
- 必須知道 reset、U-Boot、kernel 與 service 每一階段由誰控制。
- 切換前要確認是否會影響供電、reset、flash 或安全狀態。
- Request output 時應盡量同時指定初始值，降低短暫 glitch 的風險。

#### GPO 範例

```text
SLOT_RESET_N = Low  → Slot 保持 reset
SLOT_RESET_N = High → Slot 離開 reset
```

若使用 `GPIO_ACTIVE_LOW`，driver 要 assert reset 時，通常送出 logical 1：

```text
Driver logical value = 1
    ↓ ACTIVE_LOW
Physical output = Low
    ↓
Reset asserted
```

### 3.2.4 GPIO、GPI、GPO 的關係

可以用以下方式理解：

```text
GPIO：硬體類別，可支援 input 或 output
GPI ：目前被當作 input 使用
GPO ：目前被當作 output 使用
```

在某些 SoC 文件中，GPI 與 GPO 也可能指獨立的 input-only 或 output-only 硬體功能。實際能力仍要看 SoC datasheet 與 pinctrl binding。

### 3.2.5 Bidirectional GPIO

有些訊號會在不同時間切換方向，例如：

- 單線雙向資料。
- Board ID strap 在 reset 時被讀取，之後改作 output。
- Open-drain shared line。

這類訊號不能只寫「GPIO」。文件必須補充：

```text
Reset 時：input，由 external pull 決定
U-Boot 時：input，讀取 board ID
Kernel 時：output，控制 LED
```

方向切換時還要考慮外部 device 是否同時驅動，避免 bus contention。

## 3.3 Pinmux 是什麼

### 3.3.1 一顆 pin 可以有多種功能

某個 SoC ball 可能支援：

```text
Ball A12
├── GPIOA3
├── UART5_RX
├── I2C8_SDA
└── PWM2
```

Pinmux 的工作是從這些互斥功能中選一個。

### 3.3.2 GPIO 只是 Pinmux 的其中一個選項

若要把 A12 當作 GPI 或 GPO，第一個前提是：

```text
Pinmux 必須先選 GPIO function
```

接著 GPIO controller 才能設定 direction 與 value。

完整順序是：

```text
選擇 GPIO function
    ↓
GPIO controller 取得這條 line
    ↓
設定 input 或 output
    ↓
若為 output，再設定初始值
```

### 3.3.3 Pinmux 不等於 direction

這兩件事要分開：

```text
Pinmux：這顆 pin 交給 GPIO、UART、I2C 還是 PWM？
Direction：若已交給 GPIO，它是 input 還是 output？
```

因此，以下設定並不完整：

```text
Pinmux = GPIO
```

還需要知道：

```text
Direction = input 或 output
Initial output value = 0 或 1，若為 output
```

### 3.3.4 Pin configuration 是另一件事

除了 mux 與 direction，pad 還可能有電氣設定：

- <mark>Pull-up。</mark>
- <mark>Pull-down。</mark>
- <mark>Bias disable。</mark>
- Drive strength。
- Open-drain。
- Schmitt trigger。
- Slew rate。
- Input enable。

<mark>所以一顆 pin 至少要分成三個問題：</mark>

1. <mark>Pinmux 選了哪個功能？</mark>
2. <mark>GPIO direction 是 input 還是 output？</mark>
3. <mark>Pad 的電氣設定是什麼？</mark>

### 3.3.5 Pinmux 衝突

同一時間，一顆 pin 通常不能同時作為 GPIO 與 UART。

常見現象：

- Register 中 GPIO value 看似改變，但實體 pin 沒有變化。
- U-Boot 中 GPIO 正常，kernel 啟動後失效。
- UART 啟用後，原本的 GPIO 無法使用。
- Kernel log 出現 pin request failure。

排查時需確認 running DTB 與 runtime pinctrl state，而不是只看 source DTS。

## 3.4 Direction、電位與功能狀態

GPIO 最容易混淆的地方，是把 raw value、logical value 與產品狀態都寫成 0 或 1。

### 3.4.1 Physical value

Physical value 是 pin 上的實體電位：

```text
Low  = 0
High = 1
```

實際電壓依 I/O domain 而定，例如 1.8 V、3.3 V。

### 3.4.2 Asserted 與 deasserted

Asserted 表示功能有效，deasserted 表示功能無效。

它不一定等於 High：

```text
Active-high 訊號：High = asserted
Active-low 訊號 ：Low  = asserted
```

名稱中的 `_N`、`#` 或 `L` 常用來表示 active-low，但仍要依 Schematic、datasheet 與實測確認。

### 3.4.3 Logical value

若 GPIO descriptor 帶有 `GPIO_ACTIVE_LOW`，kernel consumer 常使用 logical value：

```text
logical 1 = active
logical 0 = inactive
```

對 active-low output 而言：

```text
logical 1 → physical Low
logical 0 → physical High
```

### 3.4.4 不要只記錄 0 或 1

建議使用四欄紀錄：

```text
Physical level：Low
Hardware state：Asserted
Descriptor logical value：1
Product state：Present = true
```

這能避免以下問題：

- DTS 已反相，service 又反相一次。
- 工具顯示 logical value，卻被當作 raw level。
- CPLD register 已提供解碼後狀態，軟體再次反相。

### 3.4.5 Input 的 pull

<mark>Input 若沒有外部 device 驅動，也沒有 pull，可能成為 floating input。</mark>

<mark>預設電位可能來自：</mark>

- <mark>板上 external pull-up / pull-down。</mark>
- <mark>SoC internal pull。</mark>
- <mark>CPLD 或其他 device output。</mark>
- <mark>Level shifter 的狀態。</mark>

<mark>關鍵訊號應先看 Schematic 上的 external pull。SoC internal pull 可能較弱，而且 reset 初期不一定生效。</mark>

> <mark>檢閱提醒：Pull-down disable 只代表移除內部下拉，不代表該腳位一定進入 Hi-Z。是否為 Hi-Z / floating input，需同時確認 Pinmux、GPIO direction、output driver 是否啟用，以及外部是否有 pull、buffer、CPLD 或其他 device 正在驅動。</mark>

### 3.4.6 Output 的安全初始值

對 GPO，要先回答：

```text
尚未由軟體接管時，哪個 physical level 最安全？
```

常見方向如下，但平台仍需自行定義：

- Power enable：預設 disable。
- Reset：預設 asserted，直到條件成立。
- Write protect：預設 protected。
- Flash mux：預設由安全 owner 控制。
- LED：預設 off。

## 3.5 U-Boot 與 kernel 必須分開看

U-Boot 與 Linux kernel 是不同階段，也可能使用不同 Device Tree、不同 driver 與不同 GPIO API。

不能因為 U-Boot 中 GPIO 正常，就直接認為 kernel 中也會正常。反過來也一樣。

### 3.5.1 完整啟動時間線

```text
SoC reset default
    ↓
External pull 決定初始電位
    ↓
BootROM / strap sampling
    ↓
SPL，若平台使用
    ↓
U-Boot pinmux 與 GPIO 設定
    ↓
U-Boot 載入 kernel 與 DTB
    ↓
Kernel pinctrl driver 啟動
    ↓
Kernel 套用 pinctrl default state
    ↓
GPIO controller driver 註冊 gpiochip
    ↓
GPIO hog 或 consumer driver 要求 line
    ↓
OpenBMC service 使用該狀態
```

每一次交接都可能改變 pinmux、direction 或 output value。

### 3.5.2 Reset default

<mark>Reset default 是 SoC 剛離開 reset 時的硬體狀態，可能是：</mark>

- <mark>Input。</mark>
- <mark>High impedance。</mark>
- <mark>帶 internal pull。</mark>
- <mark>特定 alternate function。</mark>
- <mark>由 strap 或安全狀態限制。</mark>

這個階段早於 U-Boot。只查看 U-Boot command 的結果，無法證明 reset release 附近沒有 glitch。

### 3.5.3 U-Boot 階段負責什麼

U-Boot 可能會：

- 設定 console UART 的 pinmux。
- 讀取 board ID 或 strap GPIO。
- 控制 boot flash mux。
- 維持 reset 或 power enable。
- 選擇要載入的 DTB。
- 修改 DTB 後再交給 kernel。

U-Boot 的 GPIO 狀態取決於平台版本與 driver model。排查時應確認：

- 實際執行的 U-Boot binary。
- U-Boot 使用的 Device Tree。
- Board-specific pinmux 程式。
- `gpio status` 或平台對應指令的結果。
- 啟動 log 中是否有 GPIO / pinctrl 錯誤。

### 3.5.4 Kernel 階段負責什麼

kernel 啟動後，可能重新設定同一顆 pin：

```text
U-Boot 設定 GPIO output High
    ↓
Kernel 套用 pinctrl default
    ↓
GPIO controller driver probe
    ↓
GPIO hog 設成 output Low
```

因此，U-Boot 最後狀態不一定會保留。

kernel 階段要確認：

- 實際載入的 DTB。
- Pinctrl node 與 consumer node。
- GPIO controller driver 是否 probe。
- GPIO hog 是否要求該 line。
- Kernel driver 是否成為 consumer。
- Userspace service 是否之後再次改變狀態。

### 3.5.5 U-Boot 與 kernel 可能使用不同 DTB

常見情況：

```text
U-Boot 自己使用 DTB A
Kernel 收到 DTB B
```

也可能是 U-Boot 啟動時修改 DTB B，例如依 board revision 啟用不同 node。

因此排查需要分別確認：

```text
U-Boot runtime device tree
Kernel running device tree
Source tree 中的 DTS
```

三者不一定相同。

### 3.5.6 常見交接問題

#### 狀況一：U-Boot 正常，進 kernel 後電位改變

可能方向：

- Kernel pinctrl state 不同。
- GPIO hog 設定不同。
- Consumer request output 時使用了不同初始值。
- Driver probe 後改變 direction。
- Service 啟動後再次改變 output。

#### 狀況二：U-Boot 讀得到 input，kernel 找不到 line

可能方向：

- Kernel DTB 未啟用 GPIO controller。
- Pinctrl 未選 GPIO function。
- Line 被另一個 driver 使用。
- U-Boot 與 kernel 的 line numbering 方式不同。

#### 狀況三：系統 ready 後正常，但開機期間有短暫 pulse

可能方向：

- Reset default 與 U-Boot 設定不同。
- U-Boot 與 kernel 交接時 direction 被短暫釋放。
- Consumer 先 request input，再改成 output。
- External pull 將 line 拉到非預期電位。

這類問題需要示波器或 logic analyzer 觀察完整時間線。

### 3.5.7 每條關鍵 GPO 都要寫交接表

例如 active-low reset：

```text
階段                 Pinmux     Direction   Physical level   Owner
SoC reset            GPIO/Hi-Z  Input       Low by pull      Hardware
U-Boot early          GPIO      Output      Low              U-Boot
U-Boot late           GPIO      Output      Low              U-Boot
Kernel pinctrl        GPIO      未要求       Low by pull      Kernel
Kernel driver probe   GPIO      Output      Low              Driver
Device ready          GPIO      Output      High             Driver
Service restart       GPIO      Output      保持 High         Driver
BMC reboot            依平台     依平台       必須量測          多階段
```

## 3.6 Linux GPIO controller、gpiochip 與 line

### 3.6.1 GPIO controller

GPIO controller 提供一組 GPIO lines。來源可能是：

- SoC GPIO bank。
- I2C GPIO expander。
- SPI GPIO expander。
- FPGA 或 CPLD driver。

Linux character device 常見為：

```text
/dev/gpiochip0
/dev/gpiochip1
```

### 3.6.2 gpiochip

每個 GPIO controller 註冊成一個或多個 gpiochip。

`gpiochip0` 的數字可能受 probe 順序影響，因此長期腳本不宜只依賴 chip number。

較穩定的識別方式：

- gpiochip label。
- line name。
- device path。
- chip label + line offset。

### 3.6.3 Line offset

Line offset 是該 gpiochip 內的索引：

```text
gpiochip0 line 0
gpiochip0 line 1
gpiochip0 line 2
```

它不是 Schematic pin number，也不一定等於 SoC datasheet 中的 GPIO 編碼。

### 3.6.4 Line name

Device Tree 可透過 `gpio-line-names` 提供名稱：

```dts
&gpio0 {
    gpio-line-names =
        "pwrbtn-n",
        "pltrst-n",
        "host-pgood",
        "bios-wp-n";
};
```

名稱順序必須與 controller line offset 完全一致。

`gpioinfo` 顯示 `unnamed`，不表示該 line 不存在或被停用。它只表示目前沒有可用的 line name。`gpiofind` 依名稱搜尋，因此找不到 unnamed line；此時應回到 `gpioinfo <chip>`，確認目標 offset 是否存在、是否有效，以及 consumer 狀態。

若中間某個 offset 沒有名稱，但後面的 offset 仍需命名，應使用空字串保留位置，不可把後面的名稱向前移：

```dts
&gpio0 {
    gpio-line-names =
        "pwrbtn-n",       /* offset 0 */
        "",               /* offset 1，刻意不命名 */
        "host-pgood";     /* offset 2 */
};
```

陣列至少應覆蓋到最後一個需要命名的 offset。未提供名稱的其餘 line 仍可能存在，但能否由 userspace 依 offset 要求，還要看 valid mask、consumer、權限與 controller driver；不能因 `gpiofind` 找不到名稱，就判定硬體沒有 mapping。

### 3.6.5 Consumer

Consumer 是目前要求該 line 的 kernel driver 或 userspace process。

若 line 已被 consumer 使用，再執行 `gpioget` 或 `gpioset` 可能得到 busy。此時應先確認 owner，而不是強制解除 driver。

## 3.7 Device Tree 如何描述 GPIO

### 3.7.1 Controller node

GPIO controller node 通常會包含：

```dts
gpio-controller;
#gpio-cells = <2>;
```

實際 cells 含義依 binding 而定。

### 3.7.2 Consumer property

Consumer 通常以 `<function>-gpios` 引用 GPIO：

```dts
some_device@40 {
    compatible = "vendor,some-device";
    reg = <0x40>;

    reset-gpios = <&gpio0 12 GPIO_ACTIVE_LOW>;
    enable-gpios = <&gpio0 13 GPIO_ACTIVE_HIGH>;
};
```

這裡分別表示：

```text
reset-gpios：gpio0 的 line 12，active-low
enable-gpios：gpio0 的 line 13，active-high
```

property 名稱與 cells 格式必須符合該 device binding。

### 3.7.3 Pinctrl state

Consumer 還可能引用 pinctrl state：

```dts
/*
 * Enable I2C5 pinmux configuration.
 * The corresponding GPIO pins are configured as I2C5 SDA/SCL signals.
 */
&pinctrl {
    pinctrl_i2c5_default: i2c5-default {
        function = "I2C5";
        groups = "I2C5";
    };
};

/*
 * Enable AST2600 I2C5 bus.
 */
&i2c5 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_i2c5_default>;
};
```

`function`、`groups`、`pins` 與 pinconf property 都是 SoC-specific，不能直接複製其他 SoC 的寫法。

### 3.7.4 Source DTS 不等於 running DTB

排查時至少要分開：

- Source DTS。
- Build 產出的 DTB。
- Bootloader 實際載入的 DTB。
- Kernel running device tree。

只改 source DTS，但 image 未更新或 bootloader 載入其他 DTB，target 行為不會改變。

### 3.7.5 GPIO Mapping 是什麼

GPIO Mapping 可以翻成「GPIO 對應關係」。它描述的不是 Pinmux，而是：

> 某個裝置的某項功能，要使用哪一個 GPIO controller 的哪一條 line，以及該訊號的有效極性。

例如：

```dts
/*
 * Slot 0 power control GPIO configuration.
 * GPIO12 is used for slot reset control (active low).
 * GPIO13 is used to enable the slot power/control signal (active high).
 */
slot@0 {
    reset-gpios = <&gpio0 12 GPIO_ACTIVE_LOW>;
    enable-gpios = <&gpio0 13 GPIO_ACTIVE_HIGH>;
};
```

先只看第一條：

```dts
reset-gpios = <&gpio0 12 GPIO_ACTIVE_LOW>;
```

它可以拆成四件事：

```text
reset-gpios
├── reset：consumer function，表示這是裝置的 reset 功能
├── &gpio0：提供 GPIO 的 controller
├── 12：gpio0 內的 line offset
└── GPIO_ACTIVE_LOW：physical Low 時，reset 為 active
```

因此這一行的完整意思是：

```text
slot@0 的 reset 功能
使用 gpio0 的 line 12
而且這條 reset 訊號為 active-low
```

#### GPIO Mapping 如何和 driver 對上

Device Tree 的 GPIO consumer property 通常使用：

```text
<function>-gpios
```

例如：

```text
reset-gpios
power-gpios
enable-gpios
presence-gpios
```

Driver 取得 GPIO 時，會使用相同的 function name：

```c
reset = devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH);
```

這裡的 `"reset"` 會對應到 Device Tree 裡的：

```dts
reset-gpios = <...>;
```

完整關係如下：

```text
Driver 要求 "reset"
    ↓
GPIO subsystem 尋找 reset-gpios
    ↓
解析 &gpio0、line 12、GPIO_ACTIVE_LOW
    ↓
回傳 GPIO descriptor 給 driver
```

若同一項功能有多條 GPIO，可以使用 index：

```dts
led-gpios =
    <&gpio0 15 GPIO_ACTIVE_HIGH>,
    <&gpio0 16 GPIO_ACTIVE_HIGH>,
    <&gpio0 17 GPIO_ACTIVE_HIGH>;
```

Driver 再以 index 0、1、2 分別取得三條 line。

#### GPIO Mapping、Pinmux 與 Direction 的差異

這三件事要分開理解：

```text
Pinmux
回答：這顆實體 pin 要交給 GPIO、UART、I2C 還是 PWM？

GPIO Mapping
回答：這個裝置的 reset、enable 或 presence 功能，要使用哪條 GPIO line？

GPIO Direction
回答：Consumer 取得 line 後，要把它設定成 input 還是 output？
```

以 reset output 為例：

```text
SoC ball A12
    ↓
Pinmux 選成 GPIO function
    ↓
A12 對應到 gpio0 line 12
    ↓
reset-gpios 將 line 12 對應給裝置的 reset 功能
    ↓
Driver 以 output 方式要求這條 line
```

因此：

```text
Pinmux != GPIO Mapping != GPIO Direction
```

三者缺少任何一項，GPIO 功能都可能無法正常工作。

#### GPIO Mapping 與 gpio-line-names 的差異

`gpio-line-names` 只是替各 line 加上方便辨識的名稱：

```dts
&gpio0 {
    gpio-line-names =
        "slot-reset-n",
        "slot-enable",
        "psu-present-n";
};
```

Consumer mapping 則是把 line 分配給裝置功能：

```dts
slot@0 {
    reset-gpios = <&gpio0 0 GPIO_ACTIVE_LOW>;
};
```

兩者分別回答：

```text
gpio-line-names：這條 line 叫什麼？
reset-gpios：哪個裝置以 reset 功能使用這條 line？
```

只有 `gpio-line-names`，不表示 driver 已經取得該 line；只有 `reset-gpios`，即使沒有可讀的 line name，driver 仍可能正常取得 GPIO。

#### GPIO Mapping 與 GPI / GPO 的關係

Mapping 本身指定 controller、line 與 active flag，但 line 最後作為 input 或 output，通常由 consumer driver 的要求方式決定。

例如 presence：

```dts
presence-gpios = <&gpio0 20 GPIO_ACTIVE_LOW>;
```

Driver 以 input 方式要求後，它就是一條 GPI：

```text
外部 presence 訊號 → gpio0 line 20 → presence driver
```

例如 reset：

```dts
reset-gpios = <&gpio0 12 GPIO_ACTIVE_LOW>;
```

Driver 以 output 方式要求後，它就是一條 GPO：

```text
reset driver → gpio0 line 12 → 外部 reset pin
```

所以不能只看到 `*-gpios` 就判定它是 GPI 或 GPO，還要看 binding、driver 與實際資料方向。

#### 舊式 `-gpio` 與目前的 `-gpios`

較舊的 Device Tree binding 可能使用：

```dts
reset-gpio = <...>;
```

新的 binding 應使用複數形式：

```dts
reset-gpios = <...>;
```

舊式單數形式主要為相容既有 binding，不應直接拿來建立新的 binding。

#### GPIO Mapping 的排查方式

如果 driver 顯示找不到 GPIO，可以依序檢查：

1. Consumer node 是否真的存在於 running DTB。
2. Property 是否使用正確 function name，例如 `reset-gpios`。
3. Driver 要求的名稱是否也是 `"reset"`。
4. Phandle 是否指向正確 GPIO controller。
5. Line offset 是否在 controller 的有效範圍內。
6. `#gpio-cells` 與 property 參數數量是否符合 binding。
7. Active flag 是否正確。
8. GPIO controller driver 是否已 probe。
9. Line 是否已被 GPIO hog 或其他 consumer 要求。
10. Pinmux 是否將對應 pad 選成 GPIO function。

## 3.8 GPIO consumer 與 GPIO hog

### 3.8.1 Descriptor-based GPIO

現代 kernel driver通常透過 GPIO descriptor 使用 line。

若 DTS 指定 `GPIO_ACTIVE_LOW`，logical API 通常已處理反相。Driver 不應再依 `_N` 名稱手動反相，否則可能形成 double inversion。

### 3.8.2 Request input

GPI consumer 會要求 line 作為 input：

```text
Pinmux = GPIO
Direction = input
Consumer 讀取 logical value
```

它不應主動改變 physical level。

### 3.8.3 Request output

GPO consumer 應在要求 line 時一併指定初始 logical value：

```text
Request output with initial value
```

這比先 request 成 input、再切成 output 更有機會降低 glitch，但最終仍受 controller、pinctrl 與外部 pull 影響。

### 3.8.4 GPIO hog

GPIO Hog（也稱 GPIO 獨占）是 Linux 核心提供的一種機制，它允許系統在啟動的早期階段（驅動程式載入時）自動請求並鎖定特定的 GPIO 引腳，將其配置為輸入、輸出高電平或輸出低電平，並一直由系統核心持有。

GPIO hog 會在 GPIO controller 註冊時要求 line：

```dts
/*
 * Enable BIOS write protection by default.
 * GPIO10 is controlled as a GPIO hog and driven high during boot
 * to prevent unauthorized BIOS flash modification.
 */
&gpio0 {
    bios_wp_default: bios-wp-default-hog {
        gpio-hog;
        gpios = <10 GPIO_ACTIVE_HIGH>;
        output-high;
        line-name = "bios-wp-default";
    };
};
```

適合：

- Kernel early boot 必須固定的安全狀態。
- 不需動態切換的 line。

不適合：

- 後續 driver 或 service 還需要控制的 line。

因為 hog 會持有該 line，其他 consumer 再要求時可能失敗。

### 3.8.5 Hog 不能取代 U-Boot 設定

GPIO hog 只有在 kernel GPIO controller 註冊後才生效。

以下時間仍不受 kernel hog 保護：

```text
SoC reset → BootROM → SPL → U-Boot → Kernel GPIO driver probe 前
```

若關鍵 GPO 在早期啟動也必須保持安全狀態，需要依靠 external pull、SoC reset default、CPLD 或 U-Boot 設定。

## 3.9 使用 libgpiod 查看 GPIO

### 3.9.1 列出 gpiochip

```sh
gpiodetect
```

確認：

- 有幾個 gpiochip。
- 每個 chip 的 label。
- Line 數量。

### 3.9.2 查看 line

```sh
gpioinfo
gpioinfo gpiochip0
```

常見資訊：

- Line offset。
- Line name。
- Direction。
- Active-low flag。
- Consumer。
- Bias，若 driver 支援。

### 3.9.3 依名稱尋找 line

```sh
gpiofind psu0-present-n
```

libgpiod 各版本的參數與輸出可能不同，應先查看：

```sh
gpiofind --help
gpioget --help
gpioset --help
gpiomon --help
```

### 3.9.4 讀取 GPI

常見形式：

```sh
gpioget gpiochip2 3
```

紀錄結果時要同時寫：

- libgpiod version。
- 完整 command。
- 回傳的是 raw 還是 logical value。
- 實體 pin 電壓。
- 當時的外部條件。

### 3.9.5 監看 edge

```sh
gpiomon --num-events=5 gpiochip2 3
```

適合：

- Presence。
- Button。
- Intrusion。
- Interrupt bring-up。

若 line 已被 kernel driver 持有，userspace 可能無法再次要求。

### 3.9.6 設定 GPO

`gpioset` 會要求 line 並設定 output。

不應直接拿它測試下列高風險訊號：

- Power enable。
- Host reset。
- Flash mux。
- Write protect。
- CPLD reset。
- Boot strap 共用線。

這些訊號應優先透過正式 driver 或 service 測試，並先建立 recovery path。

## 3.10 GPIO expander

GPIO expander (GPIO 擴充晶片) 是一種透過少量晶片接腳 (如 I2C 或 SPI 匯流排) 來擴充 MCU 更多 GPIO 接腳的積體電路。

### 3.10.1 它和 SoC GPIO 的差異

SoC GPIO：

```text
CPU/SoC 內部 controller → 外部 pin
```

I2C GPIO expander：

```text
SoC I2C controller → I2C bus → Expander register → Expander pin
```

因此 expander GPIO 是否出現，依賴前面的 bus 全部正常。

### 3.10.2 Bring-up 順序

```text
I2C root adapter
    ↓
I2C mux child adapter，若有
    ↓
Expander address ACK
    ↓
Expander driver probe
    ↓
新的 gpiochip 出現
    ↓
Line name、direction 與 value 驗證
```

### 3.10.3 Device Tree 範例

```dts
&i2c7 {
    status = "okay";

    gpio_expander0: gpio@20 {
        compatible = "nxp,pca9555";
        reg = <0x20>;
        gpio-controller;
        #gpio-cells = <2>;

        gpio-line-names =
            "psu0-present-n",
            "psu1-present-n",
            "riser0-present-n",
            "riser1-present-n";
    };
};
```

### 3.10.4 Expander 的 reset 與 power domain

需要確認：

- Expander 由哪個 rail 供電。
- Reset pin 在哪個階段解除。
- Power-on default 是 input 還是 output。
- Output register 的 reset value。
- Host off 時 expander 是否仍有電。
- Expander 消失後 service 如何處理 unavailable。

對 expander GPO，U-Boot 與 kernel 的交接更要分開，因為 U-Boot 必須先初始化 I2C controller 與 mux，才能存取 expander。

## 3.11 GPIO interrupt

GPIO 中斷 (GPIO interrupt) 是一種硬體機制，當外部引腳的電位發生特定變化時，會暫停 MCU 當前的程式，優先執行特定的中斷服務程式 (ISR)

### 3.11.1 GPIO value 與 interrupt 是兩件事

GPIO input 可以讀取現在的 High/Low；interrupt 則是在特定轉換或電位下通知 CPU。

```text
GPIO value：現在是 0 還是 1？
Interrupt ：何時應通知 CPU？
```

### 3.11.2 Trigger type

- Rising edge：Low → High。
- Falling edge：High → Low。
- Both edges：兩種轉換都通知。
- Level high：High 期間保持 asserted。
- Level low：Low 期間保持 asserted。

### 3.11.3 Device Tree 範例

```dts
/*
 * Some-device is connected through the I2C bus at address 0x40.
 * GPIO14 is used as the device interrupt input and is asserted low.
 */
some_device@40 {
    compatible = "vendor,some-device";
    reg = <0x40>;
    interrupt-parent = <&gpio0>;
    interrupts = <14 IRQ_TYPE_LEVEL_LOW>;
};
```

Trigger type 應依 device datasheet、board wiring 與 driver clear flow決定，不能只因名稱有 `_N` 就直接選 level-low。

### 3.11.4 完整 interrupt 鏈

```text
Device 內部事件發生
    ↓
Device 拉動 INT pin
    ↓
Pinmux 選成 GPIO / interrupt function
    ↓
GPIO controller 偵測 edge 或 level
    ↓
Parent interrupt controller 收到 IRQ
    ↓
Kernel handler 執行
    ↓
Handler 讀取並清除 device status
    ↓
INT pin 回到 inactive
```

### 3.11.5 查看 interrupt count

```sh
cat /proc/interrupts
watch -n 1 cat /proc/interrupts
```

若 count 不增加，檢查：

- Pin 是否真的跳變。
- Pinmux。
- GPIO IRQ mapping。
- Interrupt parent。
- Trigger type。

若 count 快速增加，檢查：

- Level source 是否未清除。
- Polarity 是否錯誤。
- Shared interrupt 是否仍有其他 device 保持 asserted。

## 3.12 Debounce

防彈跳 (Debounce) 是一種用於消除機械開關或按鍵在閉合或斷開瞬間，因彈性金屬片接觸不良所產生的短暫突波 (Bounce) 的技術. Mechanical button、presence switch 與 connector 可能在切換時快速抖動。

Debounce 可由以下其中一層負責：

- RC circuit。
- CPLD。
- GPIO controller hardware。
- Kernel driver。
- Userspace service。

文件應指定主要 owner，並記錄：

- Debounce time。
- Rising 與 falling 是否相同。
- Short pulse 是否忽略。
- 插入與拔除的穩定條件。
- BMC reboot 後是否建立假事件。

多層 debounce 疊加可能讓反應過慢；完全沒有 debounce 則可能產生多筆事件。

## 3.13 OpenBMC mapping

### 3.13.1 GPIO 正常不代表產品狀態正常

GPIO line 只是底層資料。OpenBMC service 還要把它轉成產品語意。

例如：

```text
PSU0_PRSNT_N physical Low
    ↓
GPIO logical active
    ↓
Presence service
    ↓
Inventory Present = true
    ↓
Redfish PowerSupply 顯示已安裝
```

任何一層 mapping 錯誤，都可能讓 Redfish 顯示錯誤。

### 3.13.2 Present、Available 與 Functional

三者含義不同：

- Present：實體元件是否存在。
- Available：目前資料或服務是否可取得。
- Functional：元件是否正常提供功能。

例如 PSU 已插入，但 PMBus 暫時 timeout：

```text
Present = true
Available = false
Functional 不一定立即變成 false
```

### 3.13.3 檢查 D-Bus 與 service

```sh
systemctl --failed
journalctl -b --no-pager | \
    grep -Ei 'gpio|presence|power|reset|intrusion|led|button'

busctl tree xyz.openbmc_project.ObjectMapper | \
    grep -Ei 'inventory|sensor|state|led'
```

需依平台 service 與 object path 進一步查看 property。

## 3.14 固定排查流程

不要一開始就在多層之間跳來跳去。每次依相同順序檢查。

### 第一步：定義訊號

先寫清楚：

```text
Net name：
用途：
GPI / GPO / bidirectional：
Active level：
安全預設：
```

### 第二步：確認硬體路徑

從 Schematic 找出：

- SoC ball、expander port 或 CPLD bit。
- External pull。
- Buffer、inverter 或 level shifter。
- Voltage domain。
- Power domain。

### 第三步：確認 reset 狀態

查 SoC 或 expander datasheet：

- Reset default function。
- Direction。
- Pull。
- Strap sampling requirement。

必要時量測 reset release 附近波形。

### 第四步：只看 U-Boot

確認：

- U-Boot pinmux。
- U-Boot GPIO direction。
- U-Boot output value 或 input value。
- U-Boot 使用的 Device Tree。
- 進入 kernel 前最後電位。

此時先不要用 kernel 結果替 U-Boot 下判斷。

### 第五步：只看 kernel

確認：

- Kernel 收到的 running DTB。
- Pinctrl function。
- GPIO controller probe。
- gpiochip label、offset 與 line name。
- GPIO hog。
- Consumer 與 direction。

### 第六步：比對 U-Boot 到 kernel 的交接

將兩個階段放到同一條時間線：

```text
U-Boot 最後狀態
→ Kernel pinctrl 套用
→ GPIO controller probe
→ Hog / driver request
→ Service 啟動
```

觀察是哪個時間點開始不同。

### 第七步：量測 physical level

使用電表、示波器或 logic analyzer 確認：

- 穩態 High / Low。
- Output transition。
- Reset / boot 波形。
- Glitch。
- Edge 與 debounce。
- BMC reboot 與 Host power transition。

### 第八步：確認 logical value

比對：

```text
Physical level
DTS active flag
Driver logical value
gpiod 工具輸出
```

### 第九步：確認 OpenBMC 狀態

依序查看：

```text
GPIO consumer
→ Service log
→ D-Bus property
→ Redfish / IPMI
→ Event log
```

### 第十步：保存結果

至少保存：

- Schematic revision。
- U-Boot version 與 log。
- Kernel version、DTB 與 log。
- `gpiodetect`、`gpioinfo`。
- Pinctrl debugfs。
- D-Bus property。
- 波形檔案。
- 測試條件與預期結果。

## 3.15 完整案例

### 3.15.1 案例一：PSU Presence GPI

需求：PSU 插入時，OpenBMC 顯示 `Present = true`。

#### 硬體定義

```text
Net：PSU0_PRSNT_N
Direction：GPI
Active level：Low
外部行為：PSU 插入後將 net 拉低
```

#### 啟動階段

```text
Reset：Input，由 external pull 保持 High
U-Boot：Input，不需改變狀態
Kernel：Input，由 presence consumer 讀取
OpenBMC：logical active → Present = true
```

#### 插入時的值

```text
Pin voltage：Low
Raw physical value：0
DTS flag：GPIO_ACTIVE_LOW
Descriptor logical value：1
Inventory property：Present = true
```

#### 排查順序

1. 量測 expander 或 SoC pin 是否為 Low。
2. 若是 expander，確認 I2C bus、mux、address 與 driver。
3. 確認 gpiochip 與 line name。
4. 確認 direction 為 input。
5. 確認 active-low flag。
6. 確認 consumer 能取得 logical active。
7. 確認 D-Bus `Present`。
8. 確認 Redfish mapping。

### 3.15.2 案例二：Slot Reset GPO

需求：裝置供電與 PGOOD 尚未成立前，保持 reset asserted。

#### 硬體定義

```text
Net：SLOT_RESET_N
Direction：GPO
Active level：Low
安全狀態：Low，也就是 reset asserted
```

#### 正確時間線

```text
SoC reset：external pull-down 保持 Low
U-Boot：GPIO output Low
Kernel pinctrl：GPIO function
Kernel driver request：output，初始 logical asserted
PGOOD 成立：driver logical deassert
Physical pin：High
```

#### 需要量測的重點

- Reset release 到 U-Boot 接管之間有無 High pulse。
- U-Boot 跳到 kernel 時有無短暫 Hi-Z。
- Driver request line 時有無先變 High。
- BMC warm reboot 時 Host 是否保持安全狀態。

只看系統 ready 後為 High，無法證明開機交接安全。

### 3.15.3 案例三：U-Boot 正常，kernel 後失效

現象：

```text
U-Boot 中 GPIO output 可切換
Kernel 啟動後，實體 pin 固定不動
```

逐項檢查：

1. Kernel running DTB 是否為預期版本。
2. Kernel pinctrl 是否仍選 GPIO function。
3. 是否有 UART、PWM 或其他 peripheral 使用同一 pin。
4. GPIO controller 是否包含該 line。
5. `gpioinfo` 中的 consumer 是誰。
6. 是否有 GPIO hog。
7. Driver 是否改成 input。
8. CPLD 或外部 device 是否覆寫訊號。
9. 使用示波器確認是 kernel 的哪個時間點開始改變。

## 3.16 常見問題與判讀

### 不知道它是 GPI 還是 GPO

先看資料方向：

```text
外部狀態送進 BMC → GPI
BMC 控制外部狀態 → GPO
雙方分時驅動       → Bidirectional GPIO
```

不要只看 net 名稱判斷。

### Pinmux 已設 GPIO，但 pin 沒變

Pinmux 只選擇功能，還要確認：

- Direction 是否為 output。
- Output value。
- Consumer 是否真的要求 line。
- External circuit 是否有更強的驅動來源。

### U-Boot 正常，kernel 不正常

優先比較：

- U-Boot DT 與 kernel DTB。
- U-Boot pinmux 與 kernel pinctrl。
- 進 kernel 後的 hog 與 consumer。
- 實體波形的改變時間點。

### Kernel 正常，但開機有 glitch

Kernel 穩態資料不能代表 early boot。檢查：

- SoC reset default。
- External pull。
- U-Boot early init。
- U-Boot 到 kernel 的 handoff gap。

### `gpioinfo` 沒有 line name

檢查：

- Running DTB。
- `gpio-line-names` 順序。
- GPIO controller driver probe。
- 是否查看了正確 gpiochip。

### `gpioset` 顯示 busy

通常表示 line 已被 driver、hog 或其他 process 要求。先看 consumer，不要直接解除正式 owner。

### GPIO 讀值和電表相反

確認工具顯示 raw 還是 logical value，並檢查：

- `GPIO_ACTIVE_LOW`。
- External inverter。
- CPLD邏輯。
- Service 是否再次反相。

### Output 設定後實體 pin 不變

檢查：

- Pinmux 是否仍為 alternate function。
- Direction。
- Consumer。
- Open-drain 是否只有主動拉低。
- CPLD override。
- Voltage domain 與供電。

### Interrupt count 不增加

檢查：

- Pin waveform。
- Pinmux。
- GPIO IRQ mapping。
- Parent interrupt。
- Trigger type。

### IRQ count 持續快速增加

檢查：

- Level interrupt source 是否清除。
- Active level。
- Shared IRQ 上的其他 device。

## 3.17 平台訊號紀錄格式

每條重要訊號可使用以下格式：

```text
Signal name：
Schematic revision：
Net name：
功能說明：
GPI / GPO / Bidirectional：
SoC ball / Expander port / CPLD bit：
Pinmux function：
External pull：
Buffer / inverter / level shifter：
Voltage domain：
Active level：
Physical inactive level：
Physical active level：
Reset default：
U-Boot pinmux：
U-Boot direction：
U-Boot value：
Kernel pinctrl node：
Kernel GPIO property：
Kernel direction：
gpiochip label：
Line offset：
Line name：
Consumer：
GPIO hog：
Interrupt type：
Debounce：
OpenBMC service：
D-Bus object / property：
Redfish / IPMI mapping：
BMC reboot 行為：
Host power transition 行為：
風險等級：
已驗證情境：
波形或 log 位置：
```

## 3.18 本章檢查表

### 第一輪：基本概念

- [ ] 我知道 GPIO 是可作 input/output 的通用數位線。
- [ ] 我知道 GPI 是 BMC 讀取外部狀態。
- [ ] 我知道 GPO 是 BMC 驅動外部狀態。
- [ ] 我知道 Pinmux 選功能，direction 決定 input/output。
- [ ] 我能區分 physical level、asserted state 與 logical value。
- [ ] 我知道 `_N` 常表示 active-low，但仍需查電路與實測。

### 第二輪：啟動階段

- [ ] 我能分開描述 reset、U-Boot、kernel 與 service。
- [ ] 我知道 U-Boot 正常不代表 kernel 一定正常。
- [ ] 我知道 kernel GPIO hog 不會保護 kernel 啟動前的時間。
- [ ] 我知道 running DTB 不一定等於 source DTS。
- [ ] 我能為關鍵 GPO 寫出 owner handoff 時間線。

### 第三輪：Linux GPIO

- [ ] 我知道 gpiochip number 可能因 probe 順序改變。
- [ ] 我知道 line offset 只在特定 gpiochip 內有意義。
- [ ] 我會查看 line name、direction、active flag 與 consumer。
- [ ] 我知道 line busy 時應先找 owner。
- [ ] 我知道 descriptor logical API 可能已處理 active-low。

### 第四輪：整合與驗證

- [ ] 我能從 Schematic net 追到 Linux line。
- [ ] 我能從 Linux line 追到 D-Bus 與 Redfish。
- [ ] 我會同時保存 physical voltage 與 logical state。
- [ ] 我知道關鍵 GPO 要用波形驗證 reset、U-Boot、kernel 交接。
- [ ] 我知道高風險 line 不應直接用通用工具切換。

## 3.19 本章重點

- GPIO 是通用數位輸入／輸出；GPI 表示從 BMC 讀入，GPO 表示由 BMC輸出。
- Pinmux 只決定 pin 交給 GPIO、UART、I2C、PWM 或其他功能，並不等於 GPIO direction。
- 一條 GPIO 要分開記錄 pinmux、direction、physical level、logical value 與產品狀態。
- Active-low 訊號在 physical Low 時有效；descriptor logical API 通常以 logical 1 表示 active。
- Reset、U-Boot、kernel 與 OpenBMC service 是不同階段，應分開確認 owner 與狀態。
- U-Boot 可以設定一條 GPIO，但 kernel pinctrl、hog、driver 或 service 之後仍可能重新設定。
- Kernel GPIO hog 只有在 GPIO controller probe 後生效，不能取代 external pull、reset default 或 U-Boot early init。
- 排查時先定義 GPI/GPO 與 active level，再查硬體路徑、U-Boot、kernel、physical level，最後查 OpenBMC mapping。
- 關鍵 GPO 的穩態正確仍不足以證明安全，還要量測 reset、U-Boot、kernel 與 service 交接期間是否有 glitch。
- 完整文件應讓讀者能從 Schematic net 一路追到 gpiochip、consumer、D-Bus 與 Redfish。

<a id="gpio-field-first-aid"></a>

## 3.20 附錄 : 現場急救索引

本節供問題發生時先縮小範圍。所有指令先以唯讀檢查為主。對 power、reset、write protect、flash mux 與 strap，不要直接切換輸出或寫入暫存器。

### 現象一：進入 kernel 後，GPIO 電位不再變化

先執行：

```sh
mountpoint -q /sys/kernel/debug || mount -t debugfs debugfs /sys/kernel/debug
cat /sys/kernel/debug/gpio 2>/dev/null
find /sys/kernel/debug/pinctrl -name pinmux-pins -exec sh -c 'echo "===== $1"; cat "$1"' _ {} \;
```

判讀：

- `pinmux-pins` 顯示 UART、I2C、PWM 等 function：該 pad 目前不一定交給 GPIO。
- 同時查看 pin、mux owner、GPIO owner 與 function。實際欄位排列與文字格式由 SoC pinctrl driver 決定，不能假設固定是第三欄或最後一欄。
- 若 function 顯示 GPIO，但 GPIO owner 為空：只能表示 mux 已切到 GPIO，仍要用 `gpioinfo` 或 `/sys/kernel/debug/gpio` 確認 direction、consumer 與 value。
- 若 GPIO owner 顯示具體 device 或 driver：表示 line 可能已被 consumer 要求；再用 `gpioinfo` 確認。此時 `gpioget` 或 `gpioset` 通常會得到 `EBUSY`。
- `pinctrl` 字樣不等於 line 一定可被 userspace 要求，也不等於 line 必然懸空。Pinctrl ownership 與 GPIO descriptor ownership 是不同層，必須交叉比對。
- 顯示 GPIO 但電位不動：再查 direction、consumer、external driver、open-drain、I/O power domain 與 CPLD override。
- `/sys/kernel/debug/gpio` 不存在或資訊不足：可能是 kernel config、driver 或 kernel 版本差異；它不是所有 SoC 的 GPIO 資料暫存器傾印。

依 SoC pin 或 function 名稱縮小輸出：

```sh
grep -Ei '<pin-name>|<function-name>|gpio|uart|i2c' \
    /sys/kernel/debug/pinctrl/*/pinmux-pins
```

`<pin-name>` 應替換成 datasheet、DTS 或 debugfs 中的實際名稱。只用 `grep "PIO"` 可能漏掉不含該字串的 owner 或 function。

聯動判讀：

- `pinmux-pins` 用來確認 pad 的 mux function 與 pinctrl ownership。
- `gpioinfo` 的 consumer 欄位用來確認 GPIO line 是否已被 GPIO consumer 要求。判斷 `gpioget` 或 `gpioset` 是否可能遇到 `EBUSY` 時，應以 GPIO line 的 consumer 狀態為主要依據。
- 若 `pinmux-pins` 顯示 GPIO，而 `gpioinfo` 顯示 kernel、hog 或具體 driver consumer：該 line 已被要求，另一個獨占 request 通常會得到 `EBUSY`。
- 若 `pinmux-pins` 顯示 GPIO，而 `gpioinfo` 顯示 unused：表示目前未顯示既有 GPIO consumer，但仍須確認權限、line 是否有效、kernel 版本、工具版本、競爭條件與平台限制。這不代表任何訊號都適合使用 `gpioset`。
- 即使 line 顯示 unused，power、reset、write protect、flash mux、strap、watchdog 與共享 open-drain line 仍不得直接切換。

### 現象二：不確定 kernel 實際使用哪份 Device Tree

```sh
dtc -I fs -O dts /sys/firmware/devicetree/base > /tmp/running.dts
```

優先比對：

```text
status
pinctrl-names
pinctrl-0
*-gpios
gpio-line-names
interrupt-parent
interrupts
```

若 `dtc` 不存在，可先保存 `/sys/firmware/devicetree/base`，再於開發環境使用 DTC 分析。反編譯結果可能展開 label、phandle 與 include，不宜只做逐行文字比較，應比對 node、property 與數值。

### 現象三：`Device or resource busy` 或 `EBUSY`

```sh
gpioinfo
cat /sys/kernel/debug/gpio 2>/dev/null
```

判讀：

- Line 已有 consumer：表示 kernel driver、GPIO hog 或另一個 userspace process 已持有它。
- `--consumer=debug` 只會設定本次 request 顯示的 consumer 名稱，不會搶走已被持有的 line。
- 若確實要卸載 driver，先確認它不是 power、reset、watchdog、flash、thermal 或安全路徑，並準備復原方式。

### 現象四：gpiochip 編號改變

先列出 chip、label 與裝置路徑：

```sh
gpiodetect
for c in /sys/bus/gpio/devices/gpiochip*; do
    [ -e "$c" ] || continue
    printf '%s label=' "$(basename "$c")"
    cat "$c/label" 2>/dev/null
    readlink -f "$c/device" 2>/dev/null
done
```

判讀：

- 不要把 `gpiochip0` 寫死在長期腳本。
- Label 可協助定位，但不保證所有平台都唯一，正式流程應再搭配 device path 或明確驗證只能匹配一個 chip。
- 使用 `gpiodetect | grep ... | awk ...` 前，要處理冒號、零筆與多筆匹配，不能直接假設第一筆一定正確。

### 現象五：GPIO expander 沒出現

先查 I2C topology，再查 expander：

```sh
i2cdetect -l
ls -l /sys/bus/i2c/devices/i2c-*/device 2>/dev/null
find /sys/bus/i2c/devices -maxdepth 2 -name name -print -exec cat {} \; 2>/dev/null
```

Linux 的 `i2c-N` 是 runtime adapter number，不等於 DTS label 中的數字。例如 `&i2c7` 是 source DTS label，進入 DTB 後通常不保留該 label；若中間有 I2C mux，child adapter number 也可能由 runtime 動態分配。

先列出每個 adapter 的名稱與 OF node 路徑：

```sh
for a in /sys/class/i2c-adapter/i2c-*; do
    [ -e "$a" ] || continue
    printf '%s  name=' "$(basename "$a")"
    cat "$a/name" 2>/dev/null
    printf '  of_node='
    readlink -f "$a/device/of_node" 2>/dev/null || true
done
```

再查看 Device Tree alias 與實際路徑。若系統保留 `/aliases`：

```sh
for a in /sys/firmware/devicetree/base/aliases/i2c*; do
    [ -e "$a" ] || continue
    printf '%s -> ' "$(basename "$a")"
    tr -d '\000' < "$a"
    printf '\n'
done
```

判讀：

- `&i2c7` 是 DTS source label，不能直接用 `grep i2c7` 保證找到 runtime adapter。
- `/aliases/i2c7` 若存在，其內容是 Device Tree path；將它與 adapter 的 `of_node` 路徑比對，才能確認 root adapter。
- I2C mux 的 child adapter 應再依 adapter `name`、`of_node`、parent symlink、mux channel `reg` 與 DTS topology 確認。
- `dmesg` 可作輔助，但 log 格式取決於 driver，而且 log 可能因層級或 ring buffer 被覆寫，不能作為唯一依據。

確認 bus 與地址後，才考慮掃描：

```sh
i2cdetect -y -r <bus>
```

判讀：

- 顯示地址，例如 `20`：該地址有裝置回應，但尚不能單靠這點證明型號與設定正確。
- 顯示 `UU`：該地址通常已被 kernel driver 使用，不代表故障。
- 顯示 `--`：沒有探測到回應，可能是 bus、mux channel、power、reset、地址或探測方式問題。
- `i2cdetect` 會在 bus 上產生交易，部分裝置不適合被任意掃描；先確認平台允許。

PCA9555 類裝置應將 direction、pin input 與 output latch 分開讀取。以下位址只適用於已確認為 PCA9555 相容 register map 的裝置：

```sh
# Input Port 0 / 1：讀取 port pin 的邏輯狀態
i2cget -y <bus> 0x20 0x00
i2cget -y <bus> 0x20 0x01

# Output Port 0 / 1：讀取 output latch
i2cget -y <bus> 0x20 0x02
i2cget -y <bus> 0x20 0x03

# Configuration Port 0 / 1：1 = input，0 = output
i2cget -y <bus> 0x20 0x06
i2cget -y <bus> 0x20 0x07
```

判讀時先找出目標 bit，再分別比對 configuration、input port 與 output latch：

- Configuration bit = 1：該 pin 設為 input，Input Port bit 是目前讀到的 pin 狀態。
- Configuration bit = 0：該 pin 設為 output；Output Port bit 是輸出 latch，Input Port bit則用來讀取 port pin 的實際邏輯狀態。Input Port 不是 Output Port register 的別名。
- Output latch 與 Input Port 不同：可能涉及外部強拉、open-drain 類電路、供電、負載或硬體連線，需依 datasheet 與波形繼續確認。

PCA9555 的 Polarity Inversion register 也可能影響 Input Port 的讀值，因此還要依型號確認 `0x04`、`0x05` 的設定。不同 expander 的 register map、auto-increment 與 transaction type可能不同，不可直接照搬上述位址。

若裝置已綁定 kernel driver，直接由 userspace 存取可能失敗、與 driver 競爭或改變 interrupt clear 時序。不要把強制參數當成一般排查流程。

### 現象六：U-Boot 正常，kernel 啟動後失效

U-Boot 先使用 GPIO 與 pinmux 正式命令：

```text
gpio status -a
gpio input <pin>
gpio set <pin>
gpio clear <pin>
pinmux status -a
```

可用 `help gpio`、`help pinmux` 確認該版本是否支援。`gpio status` 的常見判讀：

- `input: 0/1`：目前為 input，後方為讀值。
- `output: 0/1`：目前為 output，後方為輸出值。
- `func ...`：目前為 alternate function，不是一般 GPIO。
- `[x]`：該 GPIO 已被某個 owner 使用。

U-Boot 的 `GPIOA0`、`GPIOB3` 等 bank 名稱，不一定直接等於 Linux gpiochip offset。對帳時應建立明確換算表：

```text
U-Boot bank/pin
→ SoC datasheet 的 GPIO controller、bank 與 bit
→ Linux gpiochip label / device path
→ 該 gpiochip 內的 line offset
→ gpio-line-name / consumer
```

不要假設每個 bank 固定有 8、16 或 32 lines，也不要假設 Bank A 一定由 offset 0 開始。部分 SoC 會有保留洞、分成多個 gpiochip，或由 driver 使用不同的 offset 編排。應以 SoC datasheet、U-Boot GPIO driver、Linux GPIO driver 與 runtime `gpioinfo` 交叉確認。

若需要跨 U-Boot 與 kernel 比對同一 MMIO register，可在兩個階段唯讀相同位址及相同 bit。兩邊結果一致只能證明該 register bit 在兩次讀取時一致，不能單獨證明 U-Boot 名稱與 Linux line mapping 正確；mapping 仍需由 driver 與 datasheet 對帳。

只有在已核對 SoC datasheet、暫存器位址、bit field、clock/reset domain 與副作用後，才使用 `md` 讀取 MMIO：

```text
md.l <register-address> <count>
```

`mw.l` 會直接寫入 MMIO，可能造成掉電、reset、flash owner 切換、bus contention 或鎖死。本文不提供可直接套用的寫入值；若必須使用，應先保存原值、使用 read-modify-write、限定目標 bit，並準備斷電復原。

### 現象七：Raw、logical 與電表結果互相矛盾

同時保存：

```text
測點位置
實測電壓
SoC-side 電位
中間的 buffer / inverter / level shifter
Device-side 電位
DTS active flag
工具完整命令與版本
Driver / D-Bus logical state
```

不要只量名義上的 net，也不要假設能直接量到 BGA ball。依 Schematic 選擇 SoC-side 與 device-side 的可接觸 test point 或元件腳位；若中間有反相器，兩側電位本來就應相反。DTS 的 active flag應描述 consumer 所看到的邏輯語意，不能僅依 net 名稱機械式反相。

### 現象八：GPIO 軟體狀態正確，但實體電位不變

可以進一步唯讀 SoC 的 MMIO 狀態，但 `devmem` 不是通用 GPIO raw-value API。只有在取得正確 SoC datasheet、register map 與平台核准後才使用。

先確認：

- SoC 型號與 revision。
- GPIO bank base address。
- Direction、pad input/readback、output latch 與 pinmux register 的 offset。
- Register width、endianness、clock/reset domain 與 secure access 限制。

唯讀形式：

```sh
devmem <register-address> 32
# 或
busybox devmem <register-address> 32
```

不要直接套用其他 SoC 的位址。即使同為 ASPEED，不同世代、bank 或 revision 的 offset 與 bit 定義也可能不同。應至少區分 direction、pad input/readback、output latch 與 pinmux register；它們不一定回傳相同值。

判讀：

- Direction 顯示 input：先查 consumer、hog、driver request 與 pinmux，不要只看 output latch。
- Output latch 與 pad readback 都會變，但量測點不變：檢查測點是否在 buffer 或 level shifter 另一側、I/O power domain、焊接、外部短路與參考地。
- Output latch 會變，但 pad readback 不變：可能涉及 pinmux、open-drain、外部強拉、I/O domain 或 controller readback 語意。
- MMIO readback 與電表一致，但 `gpioget` 或 D-Bus 不一致：優先檢查 line mapping、offset、active-low、driver 反相與 service mapping。
- `devmem` 回傳 bus error、permission denied 或固定值：可能是 secure register、禁止 `/dev/mem`、clock/reset 未開、位址錯誤或平台限制，不能直接判定硬體故障。

`devmem` 讀到的是 register value，不必然等於封裝 pin 的即時物理電位。最終仍要交叉比對 MMIO、pinctrl、GPIO consumer、SoC-side 測點與 device-side 測點。

不要以 `devmem` 寫入 GPIO 或 pinmux register作為一般急救步驟。直接寫入可能同時改變同 bank 的其他 line，引發 power、reset、flash mux 或安全狀態變化。

### 現象九：GPI 已跳變，但 kernel driver 或事件沒有反應

先同時觀察實體波形與 interrupt count：

```sh
cat /proc/interrupts
watch -n 1 cat /proc/interrupts
```

若已知 driver、device 或 IRQ 名稱，再縮小範圍：

```sh
grep -Ei '<driver>|<device>|gpio|irq' /proc/interrupts
```

不要只依 `grep gpio` 判定。GPIO child IRQ 在 `/proc/interrupts` 中可能顯示 device 或 driver 名稱，不一定包含 `gpio`。

執行插拔、按鍵或其他可控事件前後，分別保存同一個 IRQ 的計數：

```sh
grep -E '^ *<irq-number>:' /proc/interrupts > /tmp/irq-before.txt
# 執行一次可控觸發
grep -E '^ *<irq-number>:' /proc/interrupts > /tmp/irq-after.txt
diff -u /tmp/irq-before.txt /tmp/irq-after.txt
```

多核心系統的 `/proc/interrupts` 會有多個 per-CPU count，判讀時要比較同一列所有 CPU 欄位的合計變化：

- 合計完全不變：這次觀察期間沒有看到 IRQ 送達該 interrupt entry。
- 合計增加一或少量，但 driver 狀態未更新：IRQ 已到達 CPU，優先查 handler、status/clear flow、threaded work 與後續 mapping。
- 合計大量增加：檢查 IRQ storm、level source 未清除、polarity 與 shared source。

前後快照比 `watch` 更適合留下可重現證據；若事件非常短或系統可能重啟，應搭配 serial log、trace 或平台可用的持久化紀錄。

判讀：

- 實體 pin 沒有跳變：回查 Schematic、power、pull、buffer、expander 與 device status。
- 實體 pin 有跳變，但 IRQ count 不增加：檢查 pinmux、GPIO-to-IRQ mapping、interrupt parent、IRQ domain、mask/unmask、polarity 與 edge/level trigger。
- IRQ count 增加，但 driver 狀態不更新：檢查 handler、threaded IRQ、work queue、device status register、clear flow 與 service mapping。
- IRQ count 持續快速增加：常見方向是 level source 未清除、polarity 錯誤、shared IRQ 仍有 source asserted，或 clear sequence 不完整。
- `gpiomon` 能看到 edge，不代表 kernel driver 的 IRQ mapping正確；line 已被 driver 持有時，`gpiomon` 也可能因 `EBUSY` 無法要求 line。

若 kernel 開啟對應 IRQ debugfs，可查看：

```sh
ls /sys/kernel/debug/irq/irqs 2>/dev/null
cat /sys/kernel/debug/irq/irqs/<irq-number> 2>/dev/null
cat /proc/irq/<irq-number>/spurious 2>/dev/null
cat /proc/irq/<irq-number>/smp_affinity_list 2>/dev/null
```

不同 kernel 版本與 config 的欄位不同，debugfs 不一定直接提供完整 trigger type。還要檢查 Device Tree 的 `interrupt-parent`、`interrupts` 或 `interrupts-extended`，並確認 edge/level 與 active polarity。若是 GPIO expander interrupt，還要確認 parent GPIO IRQ、expander interrupt controller、INT pin polarity，以及讀取哪個 status/input register 才能解除 interrupt。

### 現場第一包唯讀資料

```sh
OUT=/tmp/gpio-first-aid
mkdir -p "$OUT"
uname -a > "$OUT/uname.txt"
cat /proc/cmdline > "$OUT/cmdline.txt"
cat /proc/interrupts > "$OUT/interrupts.txt" 2>&1
dmesg > "$OUT/dmesg.txt"
gpiodetect > "$OUT/gpiodetect.txt" 2>&1
gpioinfo > "$OUT/gpioinfo.txt" 2>&1
cat /sys/kernel/debug/gpio > "$OUT/debug-gpio.txt" 2>&1
dtc -I fs -O dts /sys/firmware/devicetree/base \
    > "$OUT/running.dts" 2>"$OUT/dtc.err"
find /sys/kernel/debug/pinctrl -maxdepth 3 -type f -print \
    > "$OUT/pinctrl-files.txt" 2>&1
find /sys/kernel/debug/pinctrl \
    \( -name pins -o -name pinmux-pins -o -name pinconf-pins -o -name gpio-ranges \) \
    -type f -exec sh -c '
        for f do
            echo "===== $f"
            cat "$f"
            echo
        done
    ' sh {} + > "$OUT/pinctrl-dump.txt" 2>&1
i2cdetect -l > "$OUT/i2c-list.txt" 2>&1
```

這一包先回答五件事：目前跑哪個 kernel、實際 DT 是什麼、GPIO owner 是誰、I2C topology 是否存在，以及收集當下的 IRQ count。它不會主動切換 GPIO output，也不會掃描每條 I2C bus。`interrupts.txt` 只是單一時間點快照；若要判斷 count 是否增加，仍需在事件前後各收集一次。`pinctrl-files.txt` 保存檔案清單，`pinctrl-dump.txt` 才保存 `pins`、`pinmux-pins`、`pinconf-pins` 與 `gpio-ranges` 的實際內容。部分檔案可能因 kernel config、driver 或讀取權限而不存在，錯誤也會被保留在 dump 中。

## 3.21 參考資料

- Linux kernel GPIO consumer mappings：說明 Device Tree、ACPI 與 platform data 如何將裝置功能對應到 GPIO line：<https://docs.kernel.org/driver-api/gpio/board.html>
- Linux GPIO character device userspace API: <https://docs.kernel.org/userspace-api/gpio/chardev.html>
- Linux PINCTRL subsystem: <https://docs.kernel.org/driver-api/pin-control.html>
- Linux GPIO Device Tree bindings: <https://github.com/torvalds/linux/tree/master/Documentation/devicetree/bindings/gpio>
- libgpiod documentation: <https://libgpiod.readthedocs.io/>
- OpenBMC entity-manager: <https://github.com/openbmc/entity-manager>
