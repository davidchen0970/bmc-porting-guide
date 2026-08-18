# 附錄 A01：常用縮寫與名詞對照

本附錄整理 BMC porting、bring-up、OpenBMC、Yocto、硬體介面與除錯流程中常見縮寫，方便跨 HW、BIOS、CPLD、BMC FW、QA 與工廠端同步用語。

## A01.1 BMC / 韌體 / 系統層

| 縮寫 | 全名 | 中文說明 | 常見位置 |
| --- | --- | --- | --- |
| BMC | Baseboard Management Controller | 主機板管理控制器，負責 out-of-band 管理 | SoC、OpenBMC、Redfish、IPMI |
| BSP | Board Support Package | 板級支援包，包含 bootloader、kernel、DTS、recipe 與平台設定 | Yocto layer、vendor SDK |
| DT | Device Tree | Linux 用來描述硬體拓撲的資料結構 | kernel、bootloader |
| DTS | Device Tree Source | Device Tree 原始文字檔 | `arch/*/boot/dts` |
| DTB | Device Tree Blob | Kernel 實際載入的 Device Tree binary | boot partition、FIT image |
| FIT | Flattened Image Tree | 可封裝 kernel、DTB、ramdisk 與簽章的 boot image | U-Boot、secure boot |
| FRU | Field Replaceable Unit | 現場可更換元件與識別資料 | EEPROM、Inventory |
| SEL | System Event Log | 系統事件紀錄 | IPMI、phosphor-logging |
| SDR | Sensor Data Record | IPMI sensor 描述資料 | IPMI sensor stack |
| SOL | Serial over LAN | 透過網路轉送 host serial console | IPMI、BMC network |
| RMA | Return Merchandise Authorization | 退換修與維修流程 | Field service、工廠 |

## A01.2 儲存與更新

| 縮寫 | 全名 | 中文說明 | 常見位置 |
| --- | --- | --- | --- |
| MTD | Memory Technology Device | Linux raw flash 子系統 | SPI-NOR、SPI-NAND |
| UBI | Unsorted Block Images | raw flash 上的 wear leveling 與 volume 管理層 | SPI-NAND、raw NAND |
| UBIFS | UBI File System | 建立在 UBI volume 上的檔案系統 | rwfs、persistent data |
| GPT | GUID Partition Table | Block device 分割表 | eMMC、SD、SSD |
| MBR | Master Boot Record | 傳統 block device 分割表 | legacy boot |
| A/B | Slot A / Slot B | 雙映像更新與 rollback 架構 | firmware update |
| WP | Write Protect | 寫入保護 | flash、BIOS、CPLD |
| ROFS | Read-only File System | 唯讀 rootfs 區域，常放 SquashFS | OpenBMC image |
| RWFS | Read-write File System | 可寫資料區域 | overlay、persistent data |

## A01.3 匯流排與管理協定

| 縮寫 | 全名 | 中文說明 | 常見位置 |
| --- | --- | --- | --- |
| I2C | Inter-Integrated Circuit | 低速二線式匯流排 | sensor、EEPROM、PMBus |
| SMBus | System Management Bus | 基於 I2C 的管理匯流排 | PSU、DIMM、host sideband |
| PMBus | Power Management Bus | 電源管理協定 | VR、PSU、HSC |
| SPI | Serial Peripheral Interface | 串列周邊介面 | boot flash、CPLD、TPM |
| eSPI | Enhanced Serial Peripheral Interface | Host 與 BMC / EC 間的 sideband 介面 | x86 platform |
| LPC | Low Pin Count | 舊式 host sideband 介面 | KCS、BT、Port80 |
| KCS | Keyboard Controller Style | IPMI over LPC/eSPI 的通道之一 | host-BMC IPMI |
| BT | Block Transfer | IPMI over LPC 的通道之一 | host-BMC IPMI |
| SSIF | SMBus System Interface | IPMI over SMBus | host-BMC IPMI |
| NC-SI | Network Controller Sideband Interface | BMC 透過 NIC sideband 使用網路 | management NIC |
| MCTP | Management Component Transport Protocol | 管理元件傳輸協定 | PCIe、SMBus、I3C |
| PLDM | Platform Level Data Model | 平台管理資料模型 | device update、telemetry |
| SPDM | Security Protocol and Data Model | 裝置身分驗證與安全通道協定 | attestation、secure channel |

## A01.4 電源、重置與訊號

| 縮寫 | 全名 | 中文說明 | 常見位置 |
| --- | --- | --- | --- |
| POR | Power-on Reset | 上電重置 | reset IC、CPLD |
| PGOOD / PG | Power Good | 電源穩定指示 | VR、PSU、CPLD |
| VR | Voltage Regulator | 電壓轉換器 | CPU、DIMM、SoC rail |
| HSC | Hot Swap Controller | 熱插拔與保護控制器 | PSU、riser、power shelf |
| OCP | Over Current Protection | 過電流保護 | VR、eFuse、PSU |
| OVP | Over Voltage Protection | 過電壓保護 | VR、PSU |
| UVP | Under Voltage Protection | 欠電壓保護 | VR、PSU |
| OTP | Over Temperature Protection | 過溫保護 | VR、PSU、SoC |
| PLTRST | Platform Reset | Host platform reset | x86 host sideband |
| RSMRST | Resume Reset | Host standby/resume reset | x86 power sequence |
