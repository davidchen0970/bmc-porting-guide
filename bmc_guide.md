# BMC 通用 Porting 技術參考手冊

## 0. 手冊使用說明

### 0.1 手冊目的

本手冊用於 BMC 新平台移植、Bring-up、量產前驗證與現場問題排查。目標是把硬體連線、Boot Flow、BSP、Device Tree、Sensor、Fan、Power、Host Interface、管理介面、安全、更新、除錯與測試矩陣放在同一份文件中，降低資訊分散造成的判讀成本。

### 0.2 適用範圍

適用於使用 Linux / Yocto / OpenBMC / AMI 類 BMC 韌體的平台，常見 SoC 包含 ASPEED AST24xx/25xx/26xx、Nuvoton NPCM7xx/8xx，以及其他可執行 Linux 的 BMC SoC。

### 0.3 符號與標記規則

- `[必填]`：平台 bring-up 前必須補齊。
- `[建議]`：量產或長測前建議補齊。
- `[待確認]`：目前資訊不足，需要 HW / BIOS / CPLD / ME / FW 共同確認。
- `[量測值]`：以示波器、LA、BMC log、host log 或 register dump 取得。
- `[版本]`：需保留 commit id、tag、image version、CPLD version、BIOS version。

### 0.4 參考標準與規格書清單

- Linux kernel Device Tree、GPIO、I2C、SPI、watchdog、MTD/UBI/UBIFS 文件。
- Yocto Project / OpenEmbedded / BitBake 文件。
- DMTF Redfish、MCTP、PLDM、SPDM 規格。
- IPMI v2.0 specification。
- SoC datasheet、board schematic、CPLD register map、power sequence document。

### 0.5 名詞定義

- BMC：Baseboard Management Controller，負責 out-of-band 管理。
- BSP：Board Support Package，板級支援層，包含 kernel、bootloader、DT、recipe 與平台設定。
- DT / DTS / DTB：Device Tree source / binary，用於描述非自動枚舉硬體。
- FRU：Field Replaceable Unit，現場可更換元件及其識別資料。
- SDR：Sensor Data Record，IPMI sensor 描述資料。
- SEL：System Event Log，事件紀錄。
- UBI / UBIFS：raw flash 上的 wear leveling / volume / file system 架構。
- Redfish：DMTF 定義的 RESTful 管理 API。
- MCTP / PLDM / SPDM：平台內部管理通訊、資料模型與安全協定。

### 0.6 修訂紀錄

| 日期       | 版本 | 作者    | 內容                   |
| ---------- | ---: | ------- | ---------------------- |
| 2026-07-06 |  0.1 | Copilot | 依目錄建立第一輪填寫版 |
| 2026-07-06 |  0.2 | Copilot | 撰寫 Yocto 章節 |
| 2026-07-06 |  0.3 | Copilot | 撰寫常用變數、目錄結構與 BitBake 建構流程 |
| 2026-07-06 |  0.4 | Copilot | 撰寫在 Docker 中建立 Yocto 專案並建置完整映像 |
| 2026-07-06 |  0.5 | Copilot | 撰寫單獨建置與除錯特定套件章節 |
| 2026-07-06 |  0.6 | Copilot | 撰寫使用 .bbappend 修改套件行為章節 |
| 2026-07-06 |  0.7 | Copilot | 撰寫使用 devtool 修改原始碼並產出補丁章節 |
| 2026-07-06 |  0.8 | Copilot | 撰寫自訂 .bb Recipe 章節 |
| 2026-07-06 |  0.9 | Copilot | 撰寫進階混合開發 devtool modify / update-recipe / finish 章節 |
| 2026-07-06 |  0.10 | Copilot | OpenBMC 新 Machine Layer 與 DTS Bring-up 系統化流程 |
| 2026-07-06 |  0.11 | Copilot | 撰寫 Porting ADC Sensor |
| 2026-07-06 |  0.12 | Copilot | 撰寫 Temperature Sensor |
| 2026-07-06 |  0.13 | Copilot | 撰寫 Voltage Sensor |
| 2026-07-06 |  0.14 | Copilot | 增加 OpenBMC 常用 Project, 放在 CH11, 原有的 CH11 向後挪一章 |
| 2026-07-07 |  0.15 | Copilot | 撰寫第一章 Boot Flow 與 SoC 初始化 |
| 2026-07-07 |  0.16 | Copilot | 撰寫 Current Sensor |
| 2026-07-07 |  0.17 | Copilot | 撰寫 Power Sensor |
| 2026-07-07 |  0.18 | Copilot | 撰寫 Fan Tach Sensor |
| 2026-07-07 |  0.19 | Copilot | 撰寫 Fan PWM / Fan Control |
| 2026-07-07 |  0.20 | Copilot | 撰寫 PSU Sensor |
| 2026-07-07 |  0.21 | Copilot | 撰寫 Fan Control 與 Thermal Policy |
| 2026-07-07 |  0.22 | Copilot | 撰寫 CPU Sensor / PECI / APML |
| 2026-07-07 |  0.23 | Copilot | 撰寫 Presence / Intrusion / GPIO State Sensor Sensor |
| 2026-07-07 |  0.24 | Copilot | 撰寫 Power Control Section |
| 2026-07-08 |  0.25 | Copilot | 撰寫各類 Sensor 共用除錯指令 |
| 2026-07-08 |  0.26 | Copilot | 撰寫 CH12 基本內容 |
| 2026-07-08 |  0.27 | Copilot | 補寫 BBMASK 變數、使用情境、範例與排查流程 |
| 2026-07-09 |  0.28 | Copilot | 補寫第 2 章 Flash Partition 與儲存架構、分區表、更新 rollback、log 收集與驗收 checklist |
| 2026-07-09 |  0.29 | Copilot | 撰寫第 3 章 Pinmux / GPIO 通用設計模式、DTS 範本、OpenBMC GPIO presence、log 收集與驗收 checklist |
| 2026-07-09 |  0.30 | Copilot | 撰寫第 4 章 Reset / Clock / Power Domain、DTS 範本、domain timing、reset reason、log 收集與驗收 checklist |
| 2026-07-10 |  0.31 | Copilot | 重寫第 2 章 Flash Partition 與儲存架構，補齊 MTD / UBI / MBR / GPT、檔案系統、映像格式、更新 rollback 與 log 收集 |
| 2026-07-10 |  0.32 | Copilot | 撰寫第 5 章周邊匯流排通用知識，補強 bus map、DTS/kernel/OpenBMC 對齊、I2C/SPI/UART/PECI/eSPI/NC-SI/MCTP/PLDM/SPDM、log 收集、回查結果與驗收 checklist |
| 2026-07-10 |  0.33 | Copilot | 撰寫第 18 章 CPLD / FPGA / Board Glue Logic，補強 register map、power sequence、reset mux、fault latch、WP / flash mux、update / recovery、OpenBMC 整合、log 收集、回查結果與驗收 checklist |
| 2026-07-10 |  0.34 | Copilot | 撰寫第 20 章 Device Tree 通用寫法與排查 |
| 2026-07-10 |  0.35 | Copilot | 撰寫第 22 章 I2C / PMBus 裝置驅動架構 |
| 2026-07-10 |  0.36 | Copilot | 撰寫第 27 章 Inventory / FRU / Asset 資料模型 |
| 2026-07-10 |  0.37 | Copilot | 撰寫第 28 章 Logging / Event / Telemetry |
| 2026-07-10 |  0.38 | Copilot | 撰寫第 32 章 MCTP / PLDM / SPDM |
| 2026-07-10 |  0.39 | Copilot | 撰寫第 21 章 Kernel Driver 與核心服務 |
| 2026-07-10 |  0.40 | Copilot | 重寫第 22 章為 I2C / PMBus Framework |
| 2026-07-10 |  0.41 | Copilot | 拆分章節 |

### 0.7 資料來源可信度分級

- A：官方標準、Linux kernel 文件、Yocto 文件、SoC datasheet、board schematic。
- B：OpenBMC 官方 repository / design document。
- C：廠商應用手冊、白皮書、公開技術文章。
- D：論壇、部落格、推測性資料，只能作為排查線索。

---
