# 7. SPI

## 7.1 SPI 是什麼

### 定義與目的

SPI (Serial Peripheral Interface) 是 controller (master) 與 peripheral (slave) 之間的全雙工、同步序列匯流排，主要解決板內 (On-board) 高速、低延遲的資料交換需求。它通常使用以下四條主要訊號線：

* **SCLK** (Serial Clock)：時脈訊號，由 Controller 產生。
* **MOSI** (Master Out Slave In)：Controller 輸出、Peripheral 輸入之資料線。
* **MISO** (Master In Slave Out)：Peripheral 輸出、Controller 輸入之資料線。
* **CS / SS** (Chip Select / Slave Select)：裝置選擇訊號（通常為 Active Low）。

SPI 本身僅定義最基本的分時移位與取樣機制，並沒有統一的 Discovery、Addressing、Register Map 或 Error Response 格式。因此每個 Flash、TPM、ADC、CPLD 或 FPGA 的 Command Protocol 都必須依裝置規格書實作。

### 參與者與角色

* **Controller (Master)**：控制 SCLK 時脈頻率、Chip Select 狀態、Transfer Length 與資料方向。
* **Peripheral (Slave)**：在指定的 SPI Mode 與 Chip Select 被 Assert 下進行資料移入與移出。
* **SPI Core / Controller Driver**：Linux 核心中負責建立 `spi_message`、`spi_transfer`，管理 DMA 與 Chip-Select GPIO。
* **Protocol Driver**：理解特定硬體 (如 JEDEC Flash、TPM 2.0、IIO Sensor) 的 Opcode、Status Register 與控制邏輯。

沒問題！原本的草稿在 7.2 確實只寫了高階步驟，沒把最精髓的**硬體暫存器推拉機制**、**微觀時序（Clock Pulse 的精細動作）**以及**核心 Driver 怎麼把資料推到線路上**拆解清楚。

下面我們把「7.2 怎麼運作」從**硬體電路、時序微觀、Multi-IO 到 Kernel 軟體層**徹底補完！


## 7.2 SPI 怎麼運作

### 1. 硬體核心：環狀移位暫存器 (Shift Register Loop)

SPI 在硬體層面最核心的概念不是「傳送/接收」，而是「兩個 Shift Register 組成的閉環交換」。

```
        ┌────────────────── Controller (Master) ──────────────────┐
        │  ┌────┬────┬────┬────┬────┬────┬────┬────┐              │
        │  │ B7 │ B6 │ B5 │ B4 │ B3 │ B2 │ B1 │ B0 │ (Shift Reg)  │
        │  └─┬──┴────┴────┴────┴────┴────┴────┴────┘              │
        └────┼────────────────────────────────────────────────────┘
             │                                ▲
             ▼ (MOSI)                         │ (MISO)
        ┌────┴────────────────────────────────┼───────────────────┐
        │  ┌────┬────┬────┬────┬────┬────┬────┴────┐              │
        │  │ B7 │ B6 │ B5 │ B4 │ B3 │ B2 │ B1 │ B0 │ (Shift Reg)  │
        │  └────┴────┴────┴────┴────┴────┴─────────┘              │
        └────────────────── Peripheral (Slave) ───────────────────┘

```

* **同時推與拉**：控制器每送出 1 個 SCLK pulse，Controller 的 MSB 透過 MOSI 被推入 Peripheral，同時 Peripheral 的 MSB 透過 MISO 被推入 Controller。
* **沒有「純讀取」這件事**：在 SPI 匯流排上，想要從 Peripheral 「讀取」 1 個 Byte，Controller **必須主動寫入 1 個 Dummy Byte**（通常填 `0xFF` 或 `0x00`）來產生 8 個 SCLK 時脈，把 Peripheral 暫存器裡的資料「吐」出來。


### 2. 微觀時序：CPOL 與 CPHA 的邊緣觸發機制

SCLK 時脈邊緣分為 **Sample (取樣/閂鎖)** 與 **Shift (移位/切換電位)** 兩種動作。CPOL 與 CPHA 決定這兩種動作在時脈的第幾個邊緣發生。

#### CPOL (Clock Polarity, 時脈極性)

* **CPOL = 0**：CS 未選取（Idle）時，SCLK 保持為 **Low**。
* **CPOL = 1**：CS 未選取（Idle）时，SCLK 保持為 **High**。

#### CPHA (Clock Phase, 時脈相位)

* **CPHA = 0**：第 **1 個 SCLK 邊緣進行 Sample**，第 **2 個 SCLK 邊緣進行 Shift**。
* **CPHA = 1**：第 **1 個 SCLK 邊緣進行 Shift**，第 **2 個 SCLK 邊緣進行 Sample**。


#### Mode 0 (CPOL=0, CPHA=0) 逐 Clock 精細拆解

這是工業界最常用的模式，以傳送 1 Byte 為例：

```text
CS    ──┐                                                       ┌──
        └───┬───────────────────────────────────────────────┬───
SCLK        │   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐       │
      ──────┴───┘   └───┘   └───┘   └───┘   └───┘   └───────┴───
            │   ▲   ▲
            │   │   └─ [邊緣 2 - 下降沿]: Shift (雙方切換下一 bit 到線上)
            │   └───── [邊緣 1 - 上升沿]: Sample (雙方鎖存線上的電位)
            └───────── [CS 拉低瞬間]: Shift Bit 7 (因為第1個邊緣就要取樣，Bit 7 必須在邊緣1前就準備好)

```

1. **T0 (CS 下降沿)**：CS 拉低。因為 CPHA=0 規定「第一個時脈上升沿就要取樣」，所以 Controller 與 Peripheral **必須在 CS 拉低瞬間，就把 Bit 7 準備在 MOSI/MISO 線上**。
2. **T1 (1st Rising Edge)**：雙方讀取並鎖存 (Sample) 線上的電位（取得 Bit 7）。
3. **T2 (1st Falling Edge)**：雙方將暫存器向左位移 1 bit，把 Bit 6 推到 MOSI/MISO 上 (Shift)。
4. **T3~T15**：重複上述動作，直到 8 個 bit 傳輸完畢。
5. **T16 (CS 上升沿)**：傳輸結束，CS 拉高，BUS 回到 Idle。



### 3. 進階多線機制：Dual / Quad-SPI 與 Dummy Cycles

傳統 Standard SPI 只有單向的 MOSI/MISO（全雙工 1-bit）。為了提升 Flash 讀取吞吐量，演化出 **Dual / Quad / Octal SPI**（改為半雙工多線）：

```text
Standard SPI : CS, SCLK, MOSI (Out), MISO (In)             -> 1 bit / clock
Dual SPI     : CS, SCLK, IO0 (Bi-dir), IO1 (Bi-dir)        -> 2 bits / clock
Quad SPI     : CS, SCLK, IO0, IO1, IO2 (WP#), IO3 (HOLD#)  -> 4 bits / clock
```

#### 一次 Quad Flash 讀取的四個階段 (1-1-4 Read Sequence)

```text
CS   └───┬──────────────┬──────────────┬──────────────┬────────────────────┐
IO0      │  Opcode      │  Address     │  High-Z      │  Data Out [3:0]    │
         │  (1-bit wide)│  (1-bit wide)│  (Dummy)     │  (4-bit wide bidir)│
IO1~3    │  High-Z      │  High-Z      │  High-Z      │                    │
         └──────────────┴──────────────┴──────────────┴────────────────────┘
```

1. **Command Phase (指令階段)**：通常用 1-bit 寬度發送 Opcode（例如 `0xEB` 讀取指令）。
2. **Address Phase (地址階段)**：發送 24-bit 或 32-bit 的 Flash 內部記憶體地址。
3. **Dummy Cycles (空轉時脈)**：
    * **物理意義**：Flash 內部的儲存陣列（Array）需要時間進行內部列解碼與電容充放電。在 SCLK 頻率很高時（如 >100MHz），這幾週期 Controller 只發送 Clock 但不取樣資料，**留給 Flash 內部硬體晶片把資料從 Flash Cell 讀出來放到輸出 Buffer 的時間**。


4. **Data Phase (資料階段)**：IO0~IO3 全部轉為輸入模式，每個 SCLK 週期一次傳送 4 bits (Nibble) 的資料。


### 4. 軟體與 Linux Kernel 觀點：一個 SPI Request 的旅程

在 Linux 系統中，Protocol Driver（如 `spi-nor`）發起一次資料讀寫時，Kernel 內部的傳輸流程如下：

```text
[Protocol Driver]   spi_sync(spi_dev, &message)
                           │
                           ▼
[SPI Core Layer]    將 message 加入 controller->queue 佇列
                           │
                           ▼ (喚醒 Kernel Thread: spi_pump_messages)
[Controller Driver] 呼叫 transfer_one_message()
                           │
                           ├── 1. 設定 GPIO / 硬體暫存器 Assert CS
                           ├── 2. 設定 SPI Controller 暫存器 (Clock, Mode)
                           ├── 3. 設定 DMA 描述子 (TX Buffer / RX Buffer 記憶體位址)
                           ├── 4. 觸發 SPI 硬體開始傳輸，Task 進入 Sleep 沉睡
                           │      (SCLK 開始開關，資料在實體線路上傳輸)
                           │
                           ▼
[Hardware Interrupt] 傳輸完畢，發出 SPI Controller Interrupt
                           │
                           ▼
[ISR Driver]        Interrupt Handler 響應，喚醒沉睡的 Task
                           │
                           ▼
[Controller Driver] Deassert CS，回報 transfer 完成

```

* **`spi_transfer`**：代表一段連續的 SPI 動作（如：傳送 4 bytes Opcode）。
* **`spi_message`**：代表一個原子操作（Atomic Transaction），包含一個或多個 `spi_transfer`。在整個 `spi_message` 傳輸期間，**CS 腳位會持續保持低電位（Asserted）**，確保裝置不會被中途切斷。

## 7.3 實務限制與潛在風險

### 相容性與版控

* Controller 與 Peripheral 的 CPOL/CPHA mode、Maximum frequency、Dummy cycles、Opcode、Address width (3-byte vs 4-byte) 與 Quad-enable (QE) bit 必須完全一致。

### 安全性與防禦

* **缺乏原生認證機制**：SPI 沒有內建身份驗證或加密保護，匯流排側錄與注入攻擊風險較高。
* **權限管控**：Boot Flash (SPI-NOR)、TPM 與 CPLD 介面必須進行寫入保護；使用者空間對 `/dev/spidev` 的存取權限須嚴格限制。

### 錯誤處理

* **沒有硬體 ACK/NACK**：當 MISO 斷線或未接時，Controller 仍會讀回全 `0x00` 或全 `0xFF`。必須依賴 Status register、CRC、JEDEC ID 校驗或 Timeout 機制判斷失敗。

### 效能與資源

* **Signal Integrity (訊號完整性)**：提高 SCLK (例如 >50MHz) 或增設資料線 (Quad/Octal) 會顯著增加電路佈線與 PCB 阻抗匹配要求。
* **Overhead**：頻繁的小封包 Transfer 會帶來嚴重的 CS 切換與 Syscall/Context-switch 負擔，應優先使用 DMA 與 Tasklet/Message Queue 合併傳輸。

### 常用應用場景

* BMC boot flash (如 ASPEED / Nuvoton SOC)
* Host BIOS / UEFI flash
* TPM (Trusted Platform Module)
* CPLD / FPGA bitstream 載入與邏輯控制
* External ADC、DAC 或 GPIO expander

## 7.4 SPI-NOR 與 SPI-NAND

### 1. 底層物理結構與特性比較

NOR 與 NAND Flash 的最根本差異在於記憶體單元 (Memory Cells) 的排列方式：

* **NOR Flash（並聯架構）**：每個 Memory Cell 均獨立連接至 Bit Line。
    * **優勢**：支援 **Byte-level 隨機讀取**，讀取延遲極低、可靠度極高，幾乎不會產生 Bit Flip (翻轉位元)，非常適合存放開機程式碼 (如 U-Boot) 並支援 XIP (Execute In Place)。
    * **劣勢**：Cell 晶體管占用面積大，導致晶片容量小、成本高，且 Page Program / Erase 速度極慢。
* **NAND Flash（串聯架構）**：多個 Memory Cell (如 32~64 個) 串聯在一起組成一條 Bit Line。
    * **優勢**：Cell 占用面積極小，極容易做高密度堆疊 (如 3D-NAND)，容量大且單位成本低，Page 寫入與 Block 抹除速度快。
    * **劣勢**：**不支援 Byte 隨機存取** (必須以 Page 為單位讀寫、以 Block 為單位抹除)；隨著寫入/抹除次數增加，極易產生 Bit Flip 與壞塊 (Bad Blocks)，**必須依賴 ECC (錯誤更正碼) 與坏塊管理**。



### 2. 協定與指令流 (Command Protocol) 差異

#### SPI-NOR：直讀直寫機制

SPI-NOR 晶片內部**沒有**大容量的 SRAM Cache，Controller 的指令會直接傳遞至 Flash Array：

* **Read (讀取)**：發送 `0x03` (Read) 或 `0x0B` (Fast Read) + 3/4-Byte 地址後，即可連續不間斷地將資料由 MISO 移出（可從任意 Offset 讀取任意長度）。
* **Page Program (寫入)**：
    1. 必須先發送 `0x06` (WREN, Write Enable) 設置 `WEL` 位元。
    2. 發送 `0x02` (Page Program) + 地址 + Payload (最大 256 Bytes，不能跨越 Page 邊界)。
    3. 輪詢 Status Register (`0x05` RDSR) 的 `WIP` (Write In Progress) bit 直到變為 `0`。
* **Erase (抹除)**： Flash 寫入前位元必須為 `1` (`0xFF`)。將 `0` 變回 `1` 必須執行 Sector Erase (4KB, `0x20`) 或 Block Erase (64KB, `0xD8`)。
* **SFDP (JESD216)**：現代 SPI-NOR 支援 **Serial Flash Discoverable Parameters**。Driver 只要發送 `0x5A` 指令即可自動讀出該 Flash 的容量、Sector 大小、對應的 Erase Opcode 與 Quad-Read 時序，無需在 Driver 寫死所有晶片列表。

#### SPI-NAND：兩階段/三階段 間接快取機制

SPI-NAND 晶片內部包含了一個 **Page RAM Buffer (Cache Register)** (典型大小 2KB/4KB + OOB)，所有讀寫操作都必須透過這個 Buffer 當作中繼站：

```text
[ SPI-NAND Read 操作流程 ]
Step 1: Page Read to Cache   (Opcode 0x13 + Page Addr) ──> NAND 內部將 Flash Array 複製到 Cache RAM (耗時 ~20-100µs)
Step 2: Poll Status Register (Opcode 0x0F)             ──> 等待 OIF (Operation In Progress) 位元為 0
Step 3: Read From Cache      (Opcode 0x03 / 0x0B)      ──> 從內建 Cache RAM 將資料透過 SPI 移出至 Host

[ SPI-NAND Program 操作流程 ]
Step 1: Write Enable         (Opcode 0x06)             ──> 啟用 WEL
Step 2: Program Load         (Opcode 0x02 / 0x84)      ──> 透過 SPI 將資料寫入內建 Cache RAM
Step 3: Program Execute      (Opcode 0x10 + Page Addr) ──> 觸發 NAND 將 Cache RAM 資料燒錄至 Flash Array (耗時 ~200-500µs)
Step 4: Poll Status Register (Opcode 0x0F)             ──> 檢查 Program Fail (P_FAIL) 位元
```

![](https://www.embedded.com/wp-content/uploads/sites/2/contenteetimes-images-design-embedded-2018-fl-1-f1.jpg)

### 3. OOB (Out-Of-Band) 與 ECC (Error Correction Code)

SPI-NAND 的每個 Page 實體空間由兩部分組成：**Main Area** (如 2048 Bytes) + **OOB / Spare Area** (如 64 Bytes)。

* **OOB (Out-Of-Band) 的用途**：
    1. **Bad Block Marker (BBM)**：通常在 Block 的第一個 Page 的 OOB 首字元 (如 Byte 0) 標記非 `0xFF` 數值，代表該 Block 為壞塊。
    2. **ECC Parity**：儲存硬體或軟體計算出來的校驗碼。
    3. **Filesystem Metadata**：如 UBI/UBIFS 存放 EC (Erase Counter) 與 VID (Volume ID) Header。

    ```text
    ┌────────────────────────────────────────────────┬───────────────────────────────────┐
    │              Main Area (2048 B)                │         OOB Area (64 B)           │
    │  (Stores User Filesystem Data / Data Payload)  │ [BBM (1B)] [ECC Codes] [Metadata] │
    └────────────────────────────────────────────────┴───────────────────────────────────┘
    ```

* **On-Chip ECC vs Host ECC**：
    1. **On-Chip ECC (目前主流 SPI-NAND 標配)**：SPI-NAND 晶片內部整合了邏輯 ECC 引擎 (如 8-bit ECC / 512 Bytes)。在執行 `0x13` Page Read 時，晶片會**自動修正 Bit Flip**，並在 Status Register 中回報結果：
        * `00b`: 無錯誤 (Clean)。
        * `01b`: 發現 Bit Flip，但 **On-Chip ECC 已成功更正**。
        * `10b`: Bit Flip 數量超過修復能力 (**Uncorrectable Error**，資料損壞)。
    2. **Host ECC**：由 SOC 內部的 NAND Controller 或是 Linux 核心軟體 (BCH/Hamming) 計算，現在多用於 Parallel NAND 或無 On-Chip ECC 的特殊 Flash。



### 4. Linux Kernel 驅動框架 (Driver Framework) 比較

Linux 系統以 **MTD (Memory Technology Device)** 子系統統一抽象化所有裸 Flash。但在底層驅動架構上，SPI-NOR 與 SPI-NAND 屬於完全不同的子系統分層：

![](https://bootlin.com/wp-content/uploads/2018/02/nand-mtd-stack.png)

#### SPI-NOR 驅動鏈 (`drivers/mtd/spi-nor/`)

1. **`spi-nor` 核心層 (`spi-nor.c`)**：處理 SFDP 解析、JEDEC ID 匹配、3-Byte / 4-Byte 地址模式切換、Quad-Enable 設定。
2. **SPI Controller Driver**：呼叫 `spi_sync()` 或 SPI Mem API (`spi_mem_exec_op`) 將封包送出。
3. **上層掛載**：
    * 可透過 `mtdblock` 轉成區塊裝置掛載唯讀/輕量檔案系統 (如 **SquashFS**)。
    * 或使用具備 Flash 抹除特性的檔案系統 (如 **JFFS2**)。



#### SPI-NAND 驅動鏈 (`drivers/mtd/nand/spi/`)

1. **`spinand` 核心層 (`core.c`)**：理解 SPI-NAND 的「Page Read -> Poll -> Read Cache」三階段邏輯，維護 Manufacturer Driver 列表 (Winbond, Gigadevice, Micron 等)。
2. **NAND Core / MTD Layer**：處理 BBM (壞塊表建置)、OOB 配置與 Layout。
3. **UBI (Unsorted Block Images) 層**：**SPI-NAND 的必備核心**。UBI 在 MTD 之上提供了：
    * **Wear Leveling (磨損平衡)**：均勻分散寫入與抹除次數，延長 Flash 壽命。
    * **Bad Block Management**：自動屏蔽出廠或動態產生的壞塊。
    * **Volume Management**：將動態區塊邏輯化，提供彈性的 Volume 切分。
4. **上層掛載**：UBI 專用檔案系統 **UBIFS**。

### 5. 系統設計選型與檔案系統搭配指南

| 設計維度 | SPI-NOR Flash | SPI-NAND Flash |
| :--- | :--- | :--- |
| **容量範圍** | **4 MB ～ 128 MB** | **128 MB ～ 4 GB+** |
| **成本效益** | 小容量便宜；**大容量時單位成本極高** | **大容量單位成本極低**，適合高容量需求 |
| **開機速度** | **極快**<br>• 支援直接定址與高速 Fast Read | **較慢**<br>• 需 Page Load 載入延遲<br>• 需 UBI attach 掃描時間 |
| **可靠度與壽命** | • **P/E 次數**：約 100,000 次<br>• **壞塊特性**：極高可靠度，幾乎無壞塊 | • **P/E 次數**：約 10,000 ~ 100,000 次<br>• **壞塊特性**：**必定存在 Bad Block**，需 ECC 與壞塊管理 |
| **推薦檔案系統** | • **SquashFS**（唯讀、極度壓縮）<br>• **LittleFS** / **JFFS2**（小容量可讀寫） | • **UBIFS**（建議標準搭配）<br>• **SquashFS over UBI**（適用於大容量唯讀系統區） |
| **典型應用場景** | • BMC Boot Flash<br>• BIOS / UEFI Firmware<br>• U-Boot 啟動與控制區 | • Linux Kernel Image<br>• Rootfs 系統檔案<br>• 應用程式與大容量資料儲存區 |

## 7.5 Device Tree 範例與屬性說明

```dts
&spi1 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_spi1_default>;
    cs-gpios = <&gpio1 20 GPIO_ACTIVE_LOW>, <&gpio1 21 GPIO_ACTIVE_LOW>;
    status = "okay";

    /* Device 0: JEDEC SPI-NOR Flash */
    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;                             /* 對應 CS 0 */
        spi-max-frequency = <50000000>;        /* 50 MHz */
        spi-tx-bus-width = <4>;                /* Quad SPI TX */
        spi-rx-bus-width = <4>;                /* Quad SPI RX */
        m25p,fast-read;

        partitions {
            compatible = "fixed-partitions";
            #address-cells = <1>;
            #size-cells = <1>;

            bootloader@0 {
                label = "u-boot";
                reg = <0x00000000 0x00100000>; /* 1MB */
                read-only;
            };

            system@100000 {
                label = "sys_recovery";
                reg = <0x00100000 0x00F00000>; /* 15MB */
            };
        };
    };

    /* Device 1: SPI TPM */
    tpm@1 {
        compatible = "tcg,tpm_tis-spi";
        reg = <1>;                             /* 對應 CS 1 */
        spi-max-frequency = <10000000>;        /* 10 MHz */
        interrupt-parent = <&gpio2>;
        interrupts = <15 IRQ_TYPE_LEVEL_LOW>;
    };
};

```

### 關鍵屬性說明：

* `reg`: Chip Select 索引標號。
* `spi-max-frequency`: 裝置支援的 SCLK 上限頻率 (Hz)。
* `spi-cpol` / `spi-cpha`: 設定非 Mode 0 的特殊邊緣觸發模式。
* `spi-tx-bus-width` / `spi-rx-bus-width`: 傳送與接收資料線寬度 (1/2/4/8)。
* `cs-gpios`: 使用 GPIO 模擬 CS 腳位時指定。

## 7.6 Target 檢查與偵測指令

於 Linux 系統上進行 SPI 偵測與驗證的常見指令：

```bash
# 1. 檢視 Kernel Boot Log 中 SPI 及 MTD 初始化訊息
$ dmesg | grep -Ei 'spi|spi-nor|spi-nand|jedec|mtd'

# 2. 列出系統已繫結的 SPI Slave 裝置
$ ls -la /sys/bus/spi/devices/
# 範例：spi1.0 -> ../../../devices/platform/soc/1c68000.spi/spi_master/spi1/spi1.0

# 3. 檢查系統中的 MTD 分區資訊
$ cat /proc/mtd
$ mtdinfo -a

# 4. 檢查是否有開放 spidev 節點
$ ls -l /dev/spidev*

# 5. 檢視時脈與驅動設定狀態 (需開啟 DebugFS)
$ cat /sys/kernel/debug/clk/clk_summary | grep spi

```

## 7.7 常見問題與除錯指南 (Troubleshooting)

| 現象 | 優先檢查項目 | 建議處置方式 |
| :--- | :--- | :--- |
| **JEDEC ID 全 0x00 / 0xFF** | • Power (VCC/VCCQ)<br>• CS / Pinmux / MISO 斷線 | 1. 示波器量測 CS 是否正確拉低<br>2. 確認 SCLK 是否有時脈輸出<br>3. 檢查 SoC Pinmux 與 MISO 腳位狀態 |
| **JEDEC ID 偶發錯誤 / Bit Flip** | • SCLK 頻率過高<br>• SPI Mode 不吻合<br>• 訊號反射問題 | 1. 於 DTS 中調低 spi-max-frequency<br>2. 檢查 CPOL / CPHA 模式設定<br>3. 檢查 SCLK 上升沿波形，酌加串聯匹配電阻 |
| **Read 正常 / Program 失敗** | • WP# (Write Protect) 腳位<br>• Block Lock Bits 鎖定<br>• 供電能力不足 | 1. 確認硬體 WP# 腳位已拉高 (Deassert)<br>2. 讀取 Status Register 確認 BP (Block Protect) 位元未被鎖定<br>3. 檢查 Program 瞬間電源電壓是否塌陷 |
| **Quad Mode 失敗** | • Status Register QE Bit 未開啟<br>• IO2 / IO3 腳位衝突 | 1. 驅動需發送 Write Status Register 指令啟用 QE (Quad Enable) Bit<br>2. 確認 IO2 / IO3 未在板上被硬體誤接地或拉高 |
| **Kernel 可讀寫 / BootROM 不啟動** | • 3-Byte vs 4-Byte Address Mode 衝突<br>• BootROM Header 設定錯誤 | 1. 確認 BootROM 是否僅支援 3-Byte Addressing<br>2. 確保 Kernel 離開或 Reboot 前有將 Flash 切換回 3-Byte 模式 |

## 7.8 `spidev` 與使用者空間開發

`spidev` 是 Linux 核心提供的泛用 User-space SPI 字元裝置驅動 (`/dev/spidevX.Y`)，適合用於開發初期原型驗證。

### `spidev` C 語言 IOCTL 操作範例

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/spi/spidev.h>

int read_jedec_id(const char *devpath) {
    int fd = open(devpath, O_RDWR);
    if (fd < 0) return -1;

    unsigned char mode = SPI_MODE_0;
    unsigned char bits = 8;
    unsigned int speed = 10000000; // 10MHz

    // 1. 配置 SPI Mode、Word bit數、Speed
    ioctl(fd, SPI_IOC_WR_MODE, &mode);
    ioctl(fd, SPI_IOC_WR_BITS_PER_WORD, &bits);
    ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &speed);

    // 2. 構建 Transfer 結構 (發送 0x9F 讀取 JEDEC ID)
    unsigned char tx[4] = { 0x9F, 0x00, 0x00, 0x00 };
    unsigned char rx[4] = { 0 };

    struct spi_ioc_transfer tr = {
        .tx_buf = (unsigned long)tx,
        .rx_buf = (unsigned long)rx,
        .len = 4,
        .speed_hz = speed,
        .bits_per_word = bits,
    };

    // 3. 執行單次全雙工 Message 傳送
    if (ioctl(fd, SPI_IOC_MESSAGE(1), &tr) < 1) {
        perror("SPI transfer error");
        close(fd);
        return -1;
    }

    printf("Flash JEDEC ID: %02X %02X %02X\n", rx[1], rx[2], rx[3]);
    close(fd);
    return 0;
}

```

> **安全與操作規範**：
> * 正式產品應優先使用正式 Driver (如 `spi-nor` / `tpm_tis_spi`) 以提供完整的鎖機制 (Locking)、電源管理與安全邊界。
> * Erase / Program 與 Register Write 可能直接改變 Boot Flash、TPM 或 CPLD 狀態，執行前務必具備完整的**韌體備份**與**燒錄復原路徑 (Recovery Path)**。

## 7.9 Linux Kernel SPI 驅動架構

Linux 核心的 SPI 子系統採用典型的**三層分層架構 (Three-tier Architecture)**，將硬體控制器控制邏輯、通用協定佇列與上層設備邏輯完全解耦：

```text
┌───────────────────────────────────────────────────────────────────────────┐
│                             Protocol Driver                               │
│  (e.g., m25p80.c, spi-nor.c, tpm_tis_spi.c, spidev.c, iio/adc/ad799x.c)   │
└───────────────────────────────────────────────────────────────────────────┘
                                   │  ▲
  Creates spi_message / spi_mem_op │  │ Reports transfer result
                                   ▼  │
┌───────────────────────────────────────────────────────────────────────────┐
│                              SPI Core Layer                               │
│        (drivers/spi/spi.c, spi-mem.c, spi-engine.c, sysfs / DT)           │
│    - Provides spi_register_controller() / spi_register_driver()           │
│    - Manages Message Queue mechanism (spi_pump_messages)                  │
│    - Matches struct spi_device with struct spi_driver                     │
└───────────────────────────────────────────────────────────────────────────┘
                                   │  ▲
      Calls transfer_one_message() │  │ Hardware Interrupt / DMA completion
                                   ▼  │
┌───────────────────────────────────────────────────────────────────────────┐
│                            Controller Driver                              │
│         (e.g., spi-aspeed-smc.c, spi-stm32.c, spi-bcm2835.c)              │
│   - Operations for SOC SPI IP registers, SCLK Divider, FIFO & DMA Engine  │
│   - Controls CS (GPIO / Native CS) & Pinmux                               │
└───────────────────────────────────────────────────────────────────────────┘
```

### 1. 關鍵核心資料結構 (Core Data Structures)

Linux SPI 子系統圍繞著四個核心 `struct` 展開：

#### 1-1. `struct spi_controller` (舊稱 `spi_master`)

代表 SOC 上的 **SPI 控制器硬體實體** (Controller IP)：

* **`bus_num`**：匯流排編號 (如 `/dev/spidev1.0` 中的 `1`)。
* **`num_chipselect`**：該控制器支援的最大 CS 數量。
* **`mode_bits`**：宣告此控制器支援的硬體能力 (如 `SPI_CPOL`, `SPI_CPHA`, `SPI_RX_DUAL`, `SPI_TX_QUAD`)。
* **`transfer_one_message()`**：底層 Controller Driver 必須實作的核心 Call-back，負責處理一個 `spi_message`。
* **`mem_ops`**：指向 `struct spi_controller_mem_ops`，若 SOC 硬體支援直連 Flash (Direct Mapping/Memory Mapped Read)，可實現高效能 `spi-mem` 操作。

#### 1-2. `struct spi_device`

代表掛載於 SPI 匯流排上的 **Peripheral 邊緣裝置實體** (由 Device Tree 解析後自動建立)：

* **`controller`**：指向所屬的 `spi_controller`。
* **`max_speed_hz`**：該裝置允許的最大 SCLK 頻率。
* **`chip_select`**：該裝置連接的 CS 索引 (0, 1, 2...)。
* **`mode`**：該裝置對應的 SPI Mode (Mode 0~3, Bit Order, Bus Width)。
* **`dev`**：標準 Linux `struct device`，用於 Sysfs 與驅動模型。

#### 1-3. `struct spi_transfer`

代表**單次連續的半雙工/全雙工資料傳輸單位**：

```c
struct spi_transfer {
    const void  *tx_buf;        // 發送資料的虛擬記憶體位址 (若無發送則為 NULL)
    void        *rx_buf;        // 接收資料的虛擬記憶體位址 (若無接收則為 NULL)
    dma_addr_t  tx_dma;         // TX DMA 實體位址
    dma_addr_t  rx_dma;         // RX DMA 實體位址
    u32         len;            // 本次 transfer 的 Byte 長度
    u32         speed_hz;       // 本次 transfer 的專屬 SCLK 頻率 (覆蓋預設值)
    u8          bits_per_word;  // 通常為 8 或 16
    u8          cs_change;      // 傳輸結束後是否暫時 Deassert CS (預設 0 保持 CS 低電位)
    u8          tx_nbits;       // 發送線數 (SPI_NBITS_SINGLE / DUAL / QUAD)
    u8          rx_nbits;       // 接收線數 (SPI_NBITS_SINGLE / DUAL / QUAD)
    struct list_head transfer_list; // 用於串接進 spi_message 的雙向鏈結串列
};

```

#### 1-4. `struct spi_message`

代表**一個或多個 `spi_transfer` 組成的原子操作序列 (Atomic Transaction)**：

* 在同一個 `spi_message` 的傳輸過程中，**Chip Select 會全程保持 Asserted (Active)**，直到所有 transfers 完成，確保操作不被其他 Thread 穿插。

### 2. API 呼叫鏈與訊息佇列機制 (Transfer Execution Flow)

Protocol Driver 與 SPI Core 之間有兩種傳輸 API 模式：

```text
[Protocol Driver] 
  │
  ├──► 同步 API : spi_sync(spi_dev, msg)   ──► 呼叫者 Task Sleep 沉睡，直到硬體傳輸結束被喚醒
  │
  └──► 非同步 API: spi_async(spi_dev, msg)  ──► 立即回傳 (Non-blocking)，傳輸完成後觸發 msg->complete()

```

#### 訊息佇列 (Message Pump) 的內部處理流程

為了防止多個 Protocol Driver 同時競用同一個 SPI Controller，SPI Core 內部實作了基於 `kthread_worker` 的訊息佇列機制：

```text
1. Protocol Driver 呼叫 spi_sync()
   │
2. SPI Core 呼叫 __spi_queued_transfer() 將 spi_message 推入 controller->queue 佇列
   │
3. 喚醒 Core 內部的 Kernel Thread: spi_pump_messages()
   │
4. spi_pump_messages() 從佇列取出一個 msg，呼叫 controller->transfer_one_message()
   │
5. Controller Driver 配置 DMA / 暫存器並啟動硬體 SCLK 傳送
   │
6. 硬體傳輸完成 ──► 發出 SPI Interrupt (ISR) ──► 呼叫 spi_finalize_current_message()
   │
7. 喚醒原本在 spi_sync() 沉睡的 Protocol Driver Task，完成傳輸

```

### 3. 現代 Linux 效能利器：`spi-mem` 抽象層

在舊版 Linux 中，傳輸 SPI Flash 資料必須包裝成多個 `spi_transfer`（例如：1 Byte Opcode + 3 Bytes Address + 256 Bytes Data）。這對通用 SPI 通信適用，但對 Flash 讀寫來說產生了巨大的**結構體拆卸與 Syscall/Interrupt Overhead**。

從 Linux 4.18 開始引入了 **`spi-mem` (SPI Memory) 框架** (`drivers/spi/spi-mem.c`)：

```c
struct spi_mem_op {
    struct { u8 buswidth; u8 opcode; } cmd;             // 指令段 (1 Byte Opcode)
    struct { u8 buswidth; u8 nbytes; u64 val; } addr;   // 地址段 (3/4 Bytes Address)
    struct { u8 buswidth; u8 nbytes; } dummy;           // Dummy Cycles 週期段
    struct { u8 buswidth; enum spi_mem_data_dir dir; 
             unsigned int nbytes; ... } data;           // Payload 資料段
};

```

#### `spi-mem` 的優點：

1. **硬體直連加速 (Direct Mapping)**：若 SOC 包含專用 Flash 控制器 (如 ASPEED SMC 或 Xilinx QSPI)，`spi-mem` 可以跳過標準 `spi_transfer` 與 DMA 設定，直接透過 **AHB/AXI 匯流排記憶體映射 (MMIO)** 讀取 Flash，吞吐量提升數倍。
2. **語意明確**：直接將操作抽象為 `cmd` + `addr` + `dummy` + `data`，驅動程式不需要再花精力手動拆解/拼裝控制位元。

### 4. 驅動程式 Match 與 Probe 流程

1. **Device Tree 解析**：Kernel 啟動時解析 DT 中的 SPI Controller 節點，註冊 Controller，並依據子節點 (如 `flash@0`) 為每個 peripheral 自動建立 `struct spi_device`。
2. **Driver 註冊**：Protocol Driver 呼叫 `spi_register_driver()` 註冊 `struct spi_driver`。
3. **Bus Matching**：SPI Bus Core 比較 `spi_device.modalias` 或 `of_match_table` (Compatible 字串)：
   * 例如：DT 中的 `compatible = "jedec,spi-nor"` 與 `spi-nor.c` 中的 `of_match_table` 匹配成功。
4. **Probe 執行**：Core 呼叫 Protocol Driver 的 `probe(struct spi_device *spi)` 函式，完成裝置初始化與 MTD / Sysfs 註冊。
