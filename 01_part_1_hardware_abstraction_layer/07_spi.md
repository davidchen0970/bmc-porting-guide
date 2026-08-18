# 7. SPI

## 7.1 SPI 是什麼

SPI 是 controller 與 peripheral 之間的同步序列介面，主要解決板內高速、低延遲、全雙工資料交換。它通常使用 SCLK、MOSI、MISO 與一條或多條 chip select。SPI 只定義基本移位與取樣方式，並沒有統一的 discovery、address、register map 或錯誤回應，因此每個 flash、TPM、ADC、CPLD 或 FPGA 的 command format 都必須依裝置規格實作。

### 定義與目的

SPI 是 controller 與 peripheral 之間的同步序列介面，主要解決板內高速、低延遲、全雙工資料交換。它通常使用 SCLK、MOSI、MISO 與一條或多條 chip select。SPI 只定義基本移位與取樣方式，並沒有統一的 discovery、address、register map 或錯誤回應，因此每個 flash、TPM、ADC、CPLD 或 FPGA 的 command format 都必須依裝置規格實作。

### 參與者與角色

- Controller：控制 SCLK、chip select、transfer length 與資料方向。
- Peripheral：在指定 mode 與 chip select 下移入 / 移出資料。
- SPI core / controller driver：建立 message、transfer、DMA 與 chip-select 操作。
- Protocol driver：理解 JEDEC flash、TPM 或特定裝置的 opcode 與狀態。

## 7.2 怎麼運作

### 資料格式與規則

SPI transfer 是固定數量 clock 下的雙向 shift。CPOL / CPHA 組合形成 Mode 0 至 Mode 3；還需定義 bit order、bits-per-word、maximum frequency 與 CS setup / hold。上層裝置協定通常包含 opcode、address、dummy cycles、payload 與 status，但其格式不是 SPI bus 標準的一部分。Dual / Quad / Octal mode 還會改變資料線數、I/O direction 與取樣要求。

### 工作流程

1. Controller 設定 mode、frequency、word width 與 chip select。
2. Assert CS 並等待 setup time。
3. 每個 clock edge 依 CPOL / CPHA 同時 shift TX 與 RX。
4. 依裝置協定傳送 opcode、address、dummy cycles 與 payload。
5. Deassert CS；必要時輪詢 status 或等待 ready / interrupt。
6. Driver 將 transfer 結果交給 MTD、TPM、IIO 或 platform service。

## 7.3 實務限制與潛在風險

### 相容性與版控

mode、frequency、dummy cycles、opcode、address width 與 quad-enable 必須和裝置 revision 一致。

### 安全性與防禦

SPI 沒有內建身份驗證；boot flash write、spidev、debug header 與多主 ownership 必須限制。

### 錯誤處理

通常沒有 ACK/NACK；需靠 status register、CRC、JEDEC ID、timeout 或上層校驗判斷失敗。

### 效能與資源

提高 clock 或 I/O width 會增加 signal-integrity 與 controller FIFO / DMA 壓力；小 transfer 的 CS 與 syscall overhead 可能主導效能。

### 標準化與文件

SPI bus 規則有限，真正 command protocol 以 JEDEC、TPM 或 vendor datasheet 為準。


SPI 常用於:

- BMC boot flash.
- BIOS / host flash.
- TPM.
- CPLD / FPGA / MCU.
- External ADC 或 GPIO expander.

SPI 使用 controller / chip select / clock 與資料線. 裝置必須同意:

- CPOL / CPHA mode.
- Maximum frequency.
- Bit order.
- Data lane width.
- Command / address format.
- Chip-select timing.

## 7.4 SPI-NOR 與 SPI-NAND

SPI-NOR 常由 `spi-nor` driver 建立 MTD device. SPI-NAND 還需要處理 page / OOB / ECC 與 bad blocks. Flash layout / MTD / UBI 與更新流程由第 2 章深入說明.

## 7.5 Device Tree 範例

```dts
&spi1 {
    status = "okay";

    device@0 {
        compatible = "vendor,device";
        reg = <0>;
        spi-max-frequency = <10000000>;
    };
};
```

`reg` 通常表示 chip select. 實際 binding 可能還要求 mode / bus width / interrupt / reset 或 supply.

## 7.6 Target 檢查

```bash
$ dmesg | grep -Ei 'spi|spi-nor|spi-nand|jedec|mtd'
$ ls -l /sys/bus/spi/devices
$ cat /proc/mtd
```

## 7.7 常見問題

| 現象 | 優先檢查 |
|---|---|
| JEDEC ID 全 `00` / `ff` | Power / CS / pinmux / MISO |
| ID 偶發錯誤 | Clock / mode / signal integrity |
| Read 正常 / program 失敗 | WP / lock bits / supply / opcode |
| Quad mode 失敗 | QE bit / IO2 / IO3 / pinmux |
| Kernel 可讀 / BootROM 不啟動 | Boot header / offset / BootROM capability |

## 7.8 spidev

spidev 適合開發期間驗證簡單 protocol. 正式產品應優先使用具備 device semantics / locking / power management 與安全邊界的正式 driver 或專用 service.

Erase / program 與 register write 可能改變 boot flash / TPM / CPLD 或其他安全元件, 執行前需要備份與 recovery path.

<a id="section-5-9"></a>

