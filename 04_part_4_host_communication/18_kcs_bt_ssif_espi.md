# 18. KCS、BT、SSIF 與 eSPI

KCS、BT 與 SSIF 是 Host 與 BMC 傳送 IPMI message 的介面。KCS 與 BT 通常透過 LPC 或 eSPI Peripheral Channel 讓 Host 存取 BMC；SSIF 則使用 SMBus。eSPI 的用途更廣，除了可承載 Host I/O 存取，也能傳送 Virtual Wire、管理封包與 flash access。

本章先建立 Host、BMC 與 IPMI system interface 的關係，再分別說明 KCS、BT、SSIF 與 eSPI，最後整理 Linux、OpenBMC、BIOS、Device Tree、ACPI / SMBIOS 與實機排查流程。

## 適用範圍

本文件涵蓋 Host 與 BMC 之間的 IPMI system interface，包括 KCS、BT、SSIF，以及承載 KCS 與其他 sideband 功能的 eSPI。內容包含 Linux、OpenBMC、BIOS、Device Tree、ACPI / SMBIOS、reset、watchdog、功能驗證與雙端排查。

## 適用讀者

適用於 BMC 韌體、Host firmware、BIOS / UEFI、BSP、kernel、硬體設計、QA、系統整合與現場排查相關人員。

## 快速導覽

- [系統介面與資料流](#181-host-與-bmc-如何交換管理命令)
- [KCS 與 Host/BMC 路徑](#183-kcs-是什麼)
- [BT 與 SSIF](#186-bt-是什麼)
- [eSPI Channels 與 Reset](#188-espi-是什麼)
- [Device Tree、BIOS 與 ACPI / SMBIOS](#1812-device-treebios-與-acpi--smbios-的分工)
- [功能驗證](#1814-kcs--bt--ssif-功能驗證)
- [常見問題與雙端排查](#1817-常見問題與判讀)
- [Bring-up 與驗收](#1820-bring-up-順序)

## 18.1 Host 與 BMC 如何交換管理命令

Host OS 可以透過 BMC 查詢 sensor、FRU、SEL、watchdog 與 chassis state，也可以送出 power、reset 或 OEM commands。資料流如下：

```mermaid
flowchart TB
    A["Host application"] --> B["Host IPMI driver"]
    B --> C["KCS / BT / SSIF system interface"]
    C --> D["BMC hardware controller"]
    D --> E["BMC kernel driver"]
    E --> F["OpenBMC IPMI host service"]
    F --> G["D-Bus services / hardware"]
```

Host application 可包含 ipmitool、monitoring agent 或 IPMI watchdog。

需要分清楚三件事：

- IPMI 定義 command、request、response 與 completion code。
- KCS、BT、SSIF 定義 Host 如何把 IPMI message 交給 BMC。
- eSPI 定義 Host chipset 與 BMC 之間的實體與通道架構，可承載多種功能。

## 18.2 IPMI System Interface

IPMI system interface 是 Host 本機連接 BMC 的通道。它和 LAN 上的 IPMI 不同：

```text
Local IPMI
Host 透過 KCS / BT / SSIF 直接連接 BMC

IPMI over LAN
遠端 client 透過網路連接 BMC
```

Host Linux 通常使用 IPMI message handler、system-interface driver 與 userspace device interface。若介面建立成功，Host 上常可看到：

```text
/dev/ipmi0
```

接著可執行：

```bash
$ ipmitool -I open mc info
$ ipmitool -I open sensor list
$ ipmitool -I open chassis status
```

`-I open` 表示使用 Host 本機的 Linux IPMI device，不是經由網路連線。

## 18.3 KCS 是什麼

KCS（Keyboard Controller Style）是一種以少量 I/O registers 與狀態機交換 IPMI message 的介面。Host 逐 byte 寫入 request，BMC 逐 byte 取走；BMC 完成處理後，再讓 Host 逐 byte 讀回 response。

KCS 的名稱源自傳統 keyboard controller 類似的 register access 方式，但它在本章中的用途是 Host 與 BMC 的 IPMI 通訊。

### 18.3.1 KCS Registers

一組 KCS channel 通常具有：

- Data register：傳送 request / response byte。
- Command / status register：Host 寫入 control command，或讀取 channel status。

實際 I/O address 由平台硬體、BIOS table、ACPI 或 SMBIOS 設定決定，例如一組 data port 加上一組 command / status port。

### 18.3.2 KCS State Machine

簡化流程如下：

```mermaid
flowchart TB
    A["Host 寫入 WRITE_START"] --> B["Host 逐 byte 寫入 request"]
    B --> C["Host 寫入 WRITE_END"]
    C --> D["BMC 收到完整 IPMI request"]
    D --> E["OpenBMC IPMI service 處理 command"]
    E --> F["BMC 準備 response"]
    F --> G["Host 逐 byte 讀取 response"]
    G --> H["Channel 回到 IDLE"]
```

KCS status 常包含：

- `IBF`：Input Buffer Full，Host 已寫入資料，等待 BMC 取走。
- `OBF`：Output Buffer Full，BMC 已準備資料，等待 Host 讀取。
- State bits：Idle、Read、Write 或 Error state。
- Status / attention bits：依 controller 與 channel 實作而異。

### 18.3.3 KCS 為什麼容易 Timeout

KCS 需要 Host 與 BMC 依狀態機交替存取 register。以下情況可能造成 timeout：

- BMC kernel 尚未建立 KCS channel。
- OpenBMC IPMI service 尚未接管 channel。
- Host 使用錯誤 I/O base address。
- BIOS / ACPI / SMBIOS 描述和硬體設定不同。
- LPC / eSPI decode 尚未啟用。
- `IBF` 或 `OBF` 長時間未被清除。
- Previous request 中斷，channel 保留在非 Idle state。
- BMC userspace 忙碌或 command handler 卡住。

## 18.4 BMC 端 KCS 資料流

BMC SoC 通常提供一個或多個 KCS channels。每個 channel 包含 Host 可見的 I/O registers，以及 BMC kernel 使用的 controller interface。

```mermaid
flowchart TB
    A["Host I/O access"] --> B["LPC 或 eSPI Peripheral Channel"]
    B --> C["BMC KCS hardware channel"]
    C --> D["BMC KCS kernel driver"]
    D --> E["Character device / userspace transport"]
    E --> F["OpenBMC IPMI host service"]
```

不同 kernel branch、SoC driver 與 OpenBMC 整合方式使用的 device node 與 service name 可能不同，應以 target 的 sysfs、`/dev`、systemd units 與 recipe 為準。

### 18.4.1 多組 KCS Channels

平台可能配置多組 KCS：

| Channel | 常見用途 |
|---|---|
| KCS0 | Host OS 的主要 IPMI channel |
| KCS1 | BIOS / SMI / OEM management |
| KCS2 | Secondary host 或 debug |
| KCS3 | Platform-specific function |

用途與編號由平台設計決定。BMC channel number、Host I/O address 與 IPMI channel number 需要建立對照，三者並非天然相同。

### 18.4.2 KCS Channel Owner

每組 channel 應只由一個 BMC userspace service 接管。多個 services 同時讀寫同一 channel 可能造成：

- Request 被錯誤 service 取走。
- Response 對不到 request。
- Channel 長期停在 busy / error state。
- Service restart 後檔案描述符或狀態未收斂。

## 18.5 Host 端 KCS

Host firmware 通常透過 ACPI 或 SMBIOS 說明 BMC system interface。Host Linux IPMI SI driver 依這些資料找到 KCS type、I/O address、register spacing 與 interrupt 設定。

### 18.5.1 Host Linux 檢查

```bash
$ dmesg | grep -Ei 'ipmi|kcs|bmc|system interface'
$ ls -l /dev/ipmi* 2>/dev/null
$ lsmod | grep -E 'ipmi|ipmi_si'
$ cat `/proc/ioports` | grep -i ipmi
```

常見 modules / kernel functions 包括：

- IPMI message handler。
- IPMI device interface。
- IPMI system interface handler。
- IPMI watchdog，若平台使用。

Kernel config 與 module 名稱會隨版本而異，應以目前 Host kernel 為準。

### 18.5.2 BIOS、ACPI 與 SMBIOS

Host driver 需要知道：

- Interface type：KCS / BT / SSIF。
- I/O 或 memory address。
- Register spacing。
- Interrupt 或 polling mode。
- IPMI specification version。
- BMC slave address，若為 SSIF。

若 firmware table 填錯，BMC side 即使正常，Host 仍可能找不到 system interface。

## 18.6 BT 是什麼

BT（Block Transfer）是另一種 IPMI system interface。它使用 command、control 與 data registers 搭配 buffer 傳送整個 message，相較 KCS 的逐 byte 狀態機，更強調 block-based message transfer。

簡化流程：

```text
Host 將 request bytes 寫入 BT buffer
        ↓
Host 通知 BMC 有完整 request
        ↓
BMC 讀取 request message
        ↓
OpenBMC IPMI service 處理
        ↓
BMC 將 response 放入 buffer
        ↓
BMC 通知 Host 讀取 response
```

### 18.6.1 BT 與 KCS 的差異

| 項目 | KCS | BT |
|---|---|---|
| 傳輸方式 | 依狀態機逐 byte 交換 | 以 buffer 傳送 message block |
| Host register 存取 | Data + command / status | Control + buffer / data registers |
| 軟體支援 | 常見 | 依平台與 firmware 而定 |
| 排查重點 | `IBF` / `OBF`、state、I/O decode | Buffer ownership、attention、length、busy state |

兩者都可以承載 IPMI request / response。選擇取決於 chipset、BMC SoC、BIOS 與產品相容需求。

### 18.6.2 BT 排查

確認：

- Host firmware 將 interface type 描述為 BT。
- I/O decode 與 register address 正確。
- BT request / response buffer size 符合雙方實作。
- Host-to-BMC 與 BMC-to-Host attention 正常。
- Channel reset 能清除 stale ownership / busy state。
- Linux Host `ipmi_si` 是否偵測到 BT interface。

## 18.7 SSIF 是什麼

SSIF（SMBus System Interface）使用 SMBus 在 Host 與 BMC 之間傳送 IPMI message。Host 是 SMBus controller，BMC 在指定 address 上提供 SSIF target interface。

```mermaid
flowchart TB
    A["Host IPMI software"] --> B["Host IPMI SMBus driver"]
    B --> C["Host SMBus controller"]
    C --> D["SMBus / I2C wires"]
    D --> E["BMC SSIF target address"]
    E --> F["BMC IPMI service"]
```

### 18.7.1 SSIF Message

短 message 可以使用 SMBus write block / read block。較長 message 需要 multi-part write 或 multi-part read，由 SSIF 規範定義分段與重組方式。

因此 SSIF 需要雙方一致處理：

- SMBus block length。
- Multi-part sequencing。
- Retry / timeout。
- PEC，若平台啟用。
- BMC response ready timing。

### 18.7.2 SSIF Address

SSIF 使用 7-bit SMBus address。Host firmware table、Host driver、BMC target controller 與 schematic 必須使用相同 address。

不要把 datasheet 中包含 R/W bit 的 8-bit address 直接填入 Linux 7-bit address 欄位。

### 18.7.3 SSIF 適用情境

SSIF 適合已具有 Host-to-BMC SMBus 連接的設計，也可避開 LPC I/O decode 資源。它的效能與 timeout 會受到 SMBus controller、bus speed、block support、clock stretching 與其他 bus users 影響。

### 18.7.4 SSIF 排查

```bash
# Host side
$ dmesg | grep -Ei 'ipmi|ssif|smbus|i2c'
$ ls -l /dev/ipmi* 2>/dev/null

# BMC side
$ i2cdetect -l
$ dmesg | grep -Ei 'ssif|i2c.*slave|i2c.*target|ipmi'
```

需確認：

- Host 使用正確 SMBus controller。
- Host firmware 提供正確 SSIF address。
- BMC I2C controller 支援 target / slave mode。
- 該 bus 沒有 address conflict。
- Bus pull-up、speed、clock stretching 與 power domain 正常。
- BMC service 已接管 SSIF message path。

## 18.8 eSPI 是什麼

eSPI（Enhanced Serial Peripheral Interface）是 Host chipset 與 BMC / embedded controller 之間的 sideband bus。它以較少訊號承載傳統 LPC 與其他管理功能。

常見 signals：

- `eSPI_CLK`
- `eSPI_CS_N`
- `eSPI_IO0`～`eSPI_IO3`
- `eSPI_RESET_N`
- Alert signal，依設計與 mode 而定

eSPI 定義多個 logical channels：

- Peripheral Channel
- Virtual Wire Channel
- Out-of-Band Channel
- Flash Channel

### 18.8.1 Peripheral Channel

Peripheral Channel 承載 I/O、memory 與 bus-master 類存取。Host 對 KCS I/O ports 的讀寫，可以透過 eSPI Peripheral Channel 到達 BMC KCS controller。

```text
Host KCS I/O access
        ↓
eSPI Peripheral Channel
        ↓
BMC eSPI peripheral controller
        ↓
KCS channel registers
```

所以「KCS」和「eSPI」不是同一層：KCS 是 IPMI system interface，eSPI Peripheral Channel 是承載 Host register access 的 bus channel。

### 18.8.2 Virtual Wire Channel

Virtual Wire（VW）將傳統 sideband pins 表達為 eSPI messages。常見功能可能包含：

- Host reset / platform reset state。
- Sleep states。
- Power-good / warning 類狀態。
- Boot status / error signals。
- SMI / SCI / NMI 類 event，依平台定義。

實際 Virtual Wire index、direction、polarity 與用途由 chipset、BMC SoC 與 platform design 決定，不能只依名稱推測。

### 18.8.3 Out-of-Band Channel

OOB Channel 傳送管理封包，常見用途可能包含 MCTP 或平台自訂 sideband protocol。

需要確認：

- Endpoint roles。
- Message type。
- Maximum payload。
- Tag / completion handling。
- Flow control。
- Reset 後重新初始化。
- Kernel 與 userspace protocol owner。

OOB channel 並不自動等於 IPMI KCS。它是另一條 packet-based sideband path。

### 18.8.4 Flash Channel

Flash Channel 讓 Host 透過 eSPI 存取由 BMC 管理或連接的 flash。常見模式包括：

- Host read flash。
- Host read / write flash，依 unlock 與 security policy。
- BMC 主動提供或代理 flash access。

Flash channel 涉及 boot、write protect、ownership 與安全邊界。必須定義：

- Flash owner。
- Read / write permissions。
- Erase permissions。
- Region protection。
- BMC update 與 Host access 的互斥。
- Recovery path。

## 18.9 eSPI Reset 與 Channel Ready

`eSPI_RESET_N` 會重設 eSPI link 與 channel state。Host reset、BMC reset 與 eSPI reset 的範圍可能不同。

簡化初始化：

```text
Power / reset stable
        ↓
eSPI_RESET_N deassert
        ↓
Host 與 BMC 協商 link capabilities
        ↓
各 channel enable / ready
        ↓
Peripheral decode、VW、OOB、Flash 開始運作
```

排查時需分別確認：

- Physical link 是否 ready。
- Peripheral Channel 是否 ready。
- VW Channel 是否 ready。
- OOB Channel 是否 ready。
- Flash Channel 是否 ready。

某一 channel 失敗，不表示整條 eSPI link 全部中斷。

## 18.10 KCS over LPC 與 KCS over eSPI

KCS 可以透過 LPC 或 eSPI Peripheral Channel 提供給 Host。

| 項目 | LPC | eSPI |
|---|---|---|
| Host register access | LPC I/O cycles | eSPI Peripheral Channel packets |
| Sideband signals | 多個實體 pins | 部分功能改由 Virtual Wire |
| KCS protocol | 相同概念的 KCS state machine | 相同概念的 KCS state machine |
| 排查 | LPC decode、clock、frame、reset | Link negotiation、channel ready、decode、VW reset |

從 Host IPMI driver 角度，KCS 仍可能呈現為一組 I/O ports。底層是 LPC 還是 eSPI，通常由平台硬體與 firmware 決定。

## 18.11 OpenBMC IPMI Host 路徑

OpenBMC 需要一個 service 從 Host interface 接收 raw IPMI request，再交給 command handlers。

```text
KCS / BT / SSIF transport
        ↓
Host IPMI service
        ↓
NetFn / Command dispatch
        ↓
D-Bus services / platform handlers
        ↓
Completion code + response data
        ↓
原 transport 回傳 Host
```

需確認：

- Transport device 已建立。
- Service 已啟動並開啟 device。
- Request 能到達 command dispatcher。
- Command handler 沒有 blocking 或 deadlock。
- Response 在 Host timeout 前完成。
- Service restart 後 channel 可回到可用狀態。

Service 與 device 名稱依 OpenBMC branch、SoC 與 vendor integration 而異。可使用：

```bash
$ systemctl --type=service | grep -Ei 'ipmi|kcs|ssif|host'
$ find /dev -maxdepth 1 -type c | grep -Ei 'ipmi|kcs|bt'
$ journalctl -b --no-pager | grep -Ei 'ipmi|kcs|bt|ssif|espi'
```

## 18.12 Device Tree、BIOS 與 ACPI / SMBIOS 的分工

### 18.12.1 BMC Device Tree

BMC DTS 通常描述：

- LPC / eSPI controller。
- KCS / BT channel。
- Host I/O address decode，依 binding 與 SoC driver。
- Interrupt。
- SSIF target controller與 address。
- eSPI channel resources。

實際 property 名稱必須依 kernel binding 與 SoC DTS 範例，不同 BMC SoC 不可直接套用同一段 DTS。

### 18.12.2 Host Firmware Tables

BIOS / UEFI 需要讓 Host OS 知道 IPMI system interface。資料可能來自：

- SMBIOS IPMI Device Information。
- ACPI IPMI device / operation region。
- Platform-specific firmware table。

Host firmware 描述必須與 BMC 實際啟用的 interface type、address 與 spacing 一致。

### 18.12.3 雙邊對照表

| 項目 | BMC 端 | Host 端 |
|---|---|---|
| Interface | KCS channel 0 | KCS |
| Transport | LPC / eSPI Peripheral | Chipset LPC / eSPI |
| Address | BMC decode setting | ACPI / SMBIOS I/O base |
| Register spacing | SoC channel 設定 | Host IPMI SI parameter |
| Interrupt | BMC / Host event route | ACPI / driver IRQ |
| Service | OpenBMC host IPMI | `ipmi_si` + `/dev/ipmi0` |

## 18.13 Kernel Config 與 Build

BMC kernel 可能需要：

- LPC 或 eSPI controller driver。
- KCS BMC / BT / SSIF transport driver。
- I2C target mode，若使用 SSIF。
- MCTP / eSPI OOB support，若平台使用。
- Character device interface。

Host kernel 可能需要：

- IPMI message handler。
- IPMI device interface。
- IPMI SI handler，供 KCS / BT。
- IPMI SMBus / SSIF handler。
- IPMI watchdog，若使用。

檢查：

```bash
$ zcat /proc/config.gz | grep -Ei 'IPMI|KCS|SSIF|ESPI|LPC|MCTP'
$ lsmod | grep -Ei 'ipmi|kcs|ssif|espi'
```

Config symbol 會隨 kernel version 與 vendor tree 改變，最後應檢查實際 `.config`、built-in drivers 與 modules package。

## 18.14 KCS / BT / SSIF 功能驗證

### 18.14.1 基本 Command

Host 上：

```bash
$ ipmitool -I open mc info
$ ipmitool -I open chassis status
$ ipmitool -I open sensor list
$ ipmitool -I open fru print
$ ipmitool -I open sel list
```

先從小型、唯讀 command 開始。大量 sensor / SDR / FRU commands 會產生多次 request，更適合在基本 command 穩定後測試。

### 18.14.2 Completion Code

若 Host 收到 IPMI response，但 command 失敗，通常會看到 completion code。這表示 transport 至少完成了 request / response，問題可能位於：

- Command 不支援。
- Request data field 錯誤。
- 權限 / channel restriction。
- Platform state 不允許。
- Command handler 回傳失敗。

完全 timeout 則優先檢查 interface、channel、service 與 transport state。

### 18.14.3 壓力與 Recovery

測試：

- 連續執行唯讀 IPMI commands。
- Host IPMI service restart。
- BMC IPMI service restart。
- Host warm reset。
- Host cold reset。
- BMC reboot，依產品要求確認 Host 是否維持運作。
- Command 執行途中 reset。
- Invalid / maximum-length request。
- SSIF multi-part message。

測試工具應限制頻率，避免 command flood 影響 BMC 其他高優先工作。

## 18.15 eSPI 功能驗證

### 18.15.1 Physical 與 Link

- Clock 與 signal levels。
- Reset timing。
- I/O width 與 frequency negotiation。
- CRC / protocol error counters。
- Link reset 次數。

### 18.15.2 Peripheral Channel

- KCS I/O decode。
- POST code range，若使用。
- Port 80 capture。
- Host I/O / memory access。
- Decode enable 在 reset 後是否恢復。

### 18.15.3 Virtual Wire

建立對照表：

| Virtual Wire | Direction | Active level | 來源 | BMC 用途 |
|---|---|---|---|---|
| [待填] | Host → BMC | [待填] | Reset / sleep state | Host state tracking |
| [待填] | BMC → Host | [待填] | Warning / event | Platform notification |

每個 VW 都需量測 Host transition、BMC register state 與 OpenBMC state，避免只依 driver log 判斷。

### 18.15.4 OOB Channel

若 OOB 承載 MCTP 或其他 protocol，驗證：

- Endpoint discovery。
- Request / response。
- Maximum payload。
- Concurrent traffic。
- Link reset 後 recovery。
- Host reset / sleep state behavior。

### 18.15.5 Flash Channel

只在已確認安全的測試環境驗證 write / erase。第一輪先確認：

- Read path。
- Region mapping。
- Write-protect state。
- Host / BMC ownership。
- Access log。

## 18.16 Reset、Power State 與 Watchdog

這些介面會跨越 BMC 與 Host reset domains，因此測試矩陣至少包含：

| 動作 | KCS / BT / SSIF | eSPI Channels | 預期 |
|---|---|---|---|
| Host warm reset | 重新初始化 Host driver | VW / peripheral state 依規格更新 | BMC 繼續運作 |
| Host cold boot | Interface 重新被 firmware 發現 | Link / channels 初始化 | Local IPMI 可用 |
| BMC reboot | Channel 暫時中斷 | BMC side link 重建 | Host 依產品流程恢復 |
| eSPI reset | KCS over eSPI 暫停 | Channels reset / renegotiate | Driver 能重新連線 |
| Watchdog reset | 依 reset target | 依硬體 routing | Reset 範圍符合設計 |
| Host S3 / S5 | 依平台保留或停止 | VW 反映 sleep state | Inventory / power state 正確 |

Host watchdog 使用 IPMI interface 定期餵狗。若 KCS / SSIF timeout，可能造成 Host 被 reset。Watchdog 驗證需要同步保存 Host log、BMC journal、reset reason 與 interface state。

## 18.17 常見問題與判讀

| 現象 | 流程大約停在哪裡 | 優先檢查 |
|---|---|---|
| Host 沒有 `/dev/ipmi0` | Host interface discovery | ACPI / SMBIOS、Host config、I/O address |
| Host 找到 KCS，但所有 command timeout | Transport / BMC service | Decode、KCS state、BMC device、journal |
| 只有特定 command 失敗 | IPMI handler | NetFn / command、completion code、platform state |
| KCS 偶發 busy | State machine / service latency | `IBF` / `OBF`、command duration、service load |
| BMC service restart 後 KCS 不恢復 | Channel ownership / stale state | Device reopen、channel reset、journal |
| BT request 長度錯誤 | Buffer protocol | Length、ownership、attention bits |
| SSIF 完全無回應 | SMBus path | Controller、address、target mode、pull-up |
| SSIF 短 command 成功，長 command 失敗 | Multi-part support | Block length、segmentation、timeout |
| KCS over eSPI 開機初期失敗 | eSPI Peripheral not ready | Link negotiation、channel ready、decode timing |
| Host reset state 錯誤 | Virtual Wire mapping | VW index、direction、polarity、reset timing |
| OOB reset 後不恢復 | Endpoint / channel reinit | Channel ready、MCTP route、service restart |
| Flash channel access 被拒絕 | Security / ownership | Region、WP、owner、Host state |
| POST code 沒資料 | Peripheral decode / capture | Port range、eSPI channel、BIOS output |
| Watchdog 誤重啟 Host | Interface latency / policy | Timeout、pre-timeout、KCS / SSIF log |

## 18.18 雙端排查流程

此類問題需同時保留 Host 與 BMC 資料。

### 18.18.1 Host 端

```bash
$ uname -a
$ dmesg -T | grep -Ei 'ipmi|kcs|bt|ssif|smbus|espi'
$ ls -l /dev/ipmi* 2>/dev/null
$ lsmod | grep -Ei 'ipmi|kcs|ssif'
$ cat `/proc/ioports` | grep -i ipmi
$ ipmitool -I open mc info
```

也需保存 BIOS / UEFI version、ACPI tables、SMBIOS IPMI information 與 reset type。

### 18.18.2 BMC 端

```bash
$ uname -a
$ dmesg -T | grep -Ei 'ipmi|kcs|bt|ssif|lpc|espi|mctp'
$ systemctl --type=service | grep -Ei 'ipmi|kcs|ssif|host'
$ journalctl -b --no-pager | grep -Ei 'ipmi|kcs|bt|ssif|espi'
$ find /dev -maxdepth 1 -type c | grep -Ei 'ipmi|kcs|bt'
```

### 18.18.3 對齊 Timestamp

Host 與 BMC 時間可能不同。測試前可記錄：

- Host UTC time。
- BMC UTC time。
- 測試 command 開始時間。
- Reset assertion / deassertion 波形時間。

有共同 timestamp 才能把 Host timeout、BMC request log 與 eSPI waveform 對上。

## 18.19 Debug Log 收集

BMC 端唯讀收集範例：

```bash
#!/bin/sh

OUT=/tmp/host-interface-debug
mkdir -p "$OUT"

cat /etc/os-release > "$OUT/os-release.txt" 2>&1
uname -a > "$OUT/uname.txt"
cat /proc/cmdline > "$OUT/proc-cmdline.txt"
zcat /proc/config.gz > "$OUT/proc-config.txt" 2>&1

dmesg -T > "$OUT/dmesg.txt"
journalctl -b --no-pager > "$OUT/journal.txt" 2>&1
systemctl --failed > "$OUT/systemctl-failed.txt" 2>&1
systemctl --type=service | grep -Ei 'ipmi|kcs|ssif|host|espi' \
    > "$OUT/services.txt" 2>&1

find /dev -maxdepth 1 -type c -print > "$OUT/devices.txt" 2>&1
find /sys -maxdepth 5 -iname '*kcs*' -o -iname '*espi*' -o -iname '*ssif*' \
    > "$OUT/sysfs-paths.txt" 2>&1

busctl tree xyz.openbmc_project.ObjectMapper \
    > "$OUT/objectmapper.txt" 2>&1

tar czf "/tmp/host-interface-debug-$(date +%Y%m%d-%H%M%S).tar.gz" \
    -C /tmp host-interface-debug
```

通用收集腳本不應自動：

- Reset KCS / BT channel。
- 改寫 LPC / eSPI decode。
- 發出 Host reset。
- 切換 flash ownership。
- 寫入 eSPI Flash Channel。
- 停用 watchdog。

## 18.20 Bring-up 順序

## 18.21 平台實測紀錄表

| 項目 | 設計 / 來源 | 實測值 | 備註 |
|---|---|---|---|
| Host interface | Platform design | [待填] | KCS / BT / SSIF |
| Underlying bus | Schematic | [待填] | LPC / eSPI / SMBus |
| BMC channel | DTS / driver | [待填] | Channel number |
| Host address | BIOS / ACPI / SMBIOS | [待填] | I/O / SMBus address |
| Register spacing | Firmware table | [待填] | KCS / BT |
| Host driver | Host kernel | [待填] | Module / built-in |
| BMC transport device | `/dev` / sysfs | [待填] | Owner service |
| OpenBMC service | systemd | [待填] | Status / restart |
| Basic IPMI | `mc info` | [待填] | Result / latency |
| Long message | FRU / SDR / OEM | [待填] | Timeout / size |
| eSPI link | Controller status | [待填] | Frequency / width |
| Peripheral Channel | KCS / POST | [待填] | Ready / decode |
| Virtual Wire | Mapping table | [待填] | Direction / polarity |
| OOB Channel | MCTP / OEM | [待填] | Endpoint / payload |
| Flash Channel | Security design | [待填] | Owner / WP |
| Host reset | Test result | [待填] | Recovery time |
| BMC reboot | Test result | [待填] | Host impact |
| Watchdog | Test result | [待填] | Reset target |

## 18.22 驗收 Checklist

### IPMI System Interface

- [ ] [ ] KCS、BT 或 SSIF 的用途與 channel 已確認。
- [ ] [ ] Host address、BMC channel 與 firmware table 完全一致。
- [ ] [ ] Host kernel 能建立 `/dev/ipmi0` 或對應 IPMI device。
- [ ] [ ] BMC transport device 由單一 service 接管。
- [ ] [ ] Basic、FRU、SDR、SEL 與 OEM commands 已測試。
- [ ] [ ] Completion code 與 transport timeout 能分開判讀。
- [ ] [ ] Service restart 後 interface 能恢復。

### SSIF

- [ ] [ ] SMBus controller、7-bit address 與 BMC target mode 正確。
- [ ] [ ] Block read / write 與 multi-part message 已測試。
- [ ] [ ] Timeout、retry、PEC 與 clock stretching 符合設計。

### KCS / BT

- [ ] [ ] LPC / eSPI Peripheral decode 正確。
- [ ] [ ] KCS state、`IBF`、`OBF` 與 channel recovery 已驗證。
- [ ] [ ] BT buffer length、ownership 與 attention 已驗證，若使用 BT。

### ESPI

- [ ] [ ] Link negotiation 與 reset sequence 正確。
- [ ] [ ] Peripheral、VW、OOB 與 Flash channels 各自有實測結果。
- [ ] [ ] Virtual Wire index、direction、polarity 與用途已記錄。
- [ ] [ ] Flash ownership、write protect 與 update 互斥已驗證。
- [ ] [ ] eSPI reset 後所有必要 channels 能恢復。

### System Behavior

- [ ] [ ] Host warm reset、cold boot、sleep state 與 BMC reboot 已測試。
- [ ] [ ] Watchdog timeout 與 reset target 符合設計。
- [ ] [ ] Host 與 BMC logs 具有可對齊的 timestamp。
- [ ] [ ] BIOS、BMC image、Host kernel、DTS 與 ACPI / SMBIOS 版本已保存。

## 18.23 本章重點

1. KCS、BT 與 SSIF 是 Host 本機連接 BMC 的 IPMI system interfaces。
2. KCS 使用 register state machine 逐 byte 交換 request 與 response。
3. BT 使用 buffer 與 control bits 傳送完整 message。
4. SSIF 使用 SMBus block access，長 message 需要 multi-part handling。
5. eSPI 是 Host chipset 與 BMC 之間的 sideband bus，包含 Peripheral、VW、OOB 與 Flash channels。
6. KCS 可以透過 LPC 或 eSPI Peripheral Channel 傳送，KCS 與 eSPI 位於不同層次。
7. Host firmware table 必須和 BMC channel、address 與 transport 設定一致。
8. Completion code 表示 Host 已收到 IPMI response；timeout 則優先檢查 transport path。
9. eSPI 各 channel 可分別 ready 或失敗，排查時應逐 channel 檢查。
10. Host reset、BMC reboot、eSPI reset 與 watchdog reset 需要分開驗證。

## 18.24 本章參考資料

- Linux kernel documentation - The Linux IPMI Driver: https://docs.kernel.org/driver-api/ipmi.html
- Linux kernel documentation - IPMI KCS BMC device interface: https://docs.kernel.org/driver-api/ipmi.html
- OpenBMC documentation: https://github.com/openbmc/docs
- DMTF MCTP specifications: https://www.dmtf.org/standards/pmci
- Intel IPMI specifications: https://www.intel.com/content/www/us/en/products/docs/servers/ipmi/ipmi-home.html
- Intel Enhanced Serial Peripheral Interface specification: https://www.intel.com/content/www/us/en/standards/serial-peripheral-interface-enhanced-serial-peripheral-interface.html
