# 5. 周邊匯流排通用知識

BMC 透過各種匯流排連接感測器 / 儲存裝置 / Host / 網路控制器與平台管理元件. 不同介面的電氣特性與 protocol 各不相同, 但從硬體連線 / controller / driver 到 OpenBMC service 的排查順序具有共同架構.

本章建立周邊介面的整體地圖, 說明 I2C / SPI / UART / ADC / PWM / PECI / eSPI / 網路 sideband / USB 與 MCTP 等介面在 BMC 平台中的位置. 各 protocol 的細節則由後續專章深入說明.

## 適用範圍

本章涵蓋 BMC 平台中的周邊匯流排架構、bus map、硬體前置條件、Linux runtime interface、OpenBMC service integration、跨 power state 驗證與 debug safety. 各 protocol 的詳細原理仍由後續專章說明.

## 適用讀者

- 負責 BMC 硬體 bring-up、Linux kernel、Device Tree、OpenBMC service 或平台驗證的人員.
- 需要從實體介面追查至 D-Bus、Redfish、IPMI 或 EventLog 的開發與排查人員.

## 快速導覽

- [共同架構與排查層次](#section-5-1)
- [Bring-up 的共同順序](#section-5-3)
- [Bus Map](#section-5-4)
- [Power、Reset、Clock 與 Pinmux](#section-5-6)
- [I2C、SMBus 與 PMBus](#section-5-7)
- [SPI](#section-5-8)
- [UART 與 Serial Console](#section-5-9)
- [其他周邊介面](#section-5-10)
- [Debug Safety](#section-5-20)
- [跨 Power State 驗證](#section-5-21)
- [共用 Debug Log 收集](#section-5-23)
- [驗收 Checklist](#section-5-26)

<a id="section-5-1"></a>

## 5.1 周邊介面的共同架構

一個周邊裝置從硬體到管理介面, 通常經過以下路徑:

```mermaid
flowchart TB
    A["實體裝置 / Endpoint"] --> B["Power / Reset / Clock / Pull-up / Termination"]
    B --> C["Pinmux / Level Shifter / Mux / Connector"]
    C --> D["SoC Bus Controller"]
    D --> E["Linux Controller Driver"]
    E --> F["Child Device / Endpoint Driver"]
    F --> G["Sysfs / Dev Node / Hwmon / IIO / TTY / Netdev / Socket"]
    G --> H["OpenBMC Service"]
    H --> I["D-Bus Inventory / Sensor / State / Event"]
    I --> J["Redfish / IPMI / WebUI"]
```

排查時先找出資料在哪一層中斷. 相同的「Redfish 沒有 sensor」可能來自:

- 裝置未供電.
- Reset 尚未解除.
- Pinmux 選錯功能.
- Controller driver 未 probe.
- Mux path 或 address 錯誤.
- Device driver 未 bind.
- Hwmon channel 不存在.
- OpenBMC config 未匹配.
- D-Bus association 缺少.
- bmcweb 沒有取得正確資料.

## 5.2 Controller / Device 與 Protocol

這三個名詞描述不同角色.

### 5.2.1 Controller

Controller 是 SoC 中負責驅動實體介面的硬體, 例如:

- I2C controller.
- SPI controller.
- UART controller.
- ADC controller.
- PWM controller.
- Ethernet MAC.
- USB device controller.
- eSPI controller.

Linux controller driver 通常依 Device Tree 建立對應 bus / class device 或 network interface.

### 5.2.2 Device

Device 是連接在 controller 後面的硬體, 例如:

- I2C EEPROM.
- PMBus PSU.
- SPI flash.
- UART-connected MCU.
- Ethernet PHY.
- USB Host peer.
- MCTP endpoint.

### 5.2.3 Protocol

Protocol 定義雙方如何交換資訊. 例如同一條 I2C 實體連線上可以承載:

- 一般 register-based I2C protocol.
- SMBus access forms.
- PMBus commands.
- MCTP over SMBus.
- Vendor mailbox.

所以「I2C bus 可看到 address」只表示底層連線有回應, 還需要驗證上層 protocol / driver 與資料格式.

<a id="section-5-3"></a>

## 5.3 Bring-up 的共同順序

建議所有周邊介面使用同一套順序:

1. 確認裝置型號 / 接線與 power domain.
2. 確認 reset / clock / pull-up / termination 與 strap.
3. 確認 pinmux 與 mux select.
4. 確認 controller node / kernel config 與 driver probe.
5. 確認 child device / address / chip select 或 endpoint identity.
6. 確認 protocol driver 與 raw interface.
7. 確認 OpenBMC service / D-Bus object 與 association.
8. 確認 Redfish / IPMI 與 event 呈現.
9. 測試 BMC reboot / Host power transition / hot-plug 與 recovery.

跳過前置條件直接使用 userspace 工具, 容易因裝置未上電 / mux path 錯誤或 line ownership 不明而得到誤導結果.

<a id="section-5-4"></a>

## 5.4 Bus Map

Bus map 用來把 schematic / Linux runtime 與 OpenBMC 串在一起. 每個 controller 與裝置至少記錄:

| 欄位 | 說明 |
|---|---|
| Bus Type | I2C / SPI / UART / PECI / eSPI / USB / MCTP 等 |
| Physical Controller | Schematic / SoC 名稱 |
| Linux Interface | `i2c-5` / `spi1.0` / `ttyS4` / `eth0` 等 |
| Topology | Direct / mux / bridge / slot / connector |
| Device / Endpoint | 型號與平台名稱 |
| Address / Identity | I2C address / CS / EID / PHY address 等 |
| Power Domain | Always-on / standby / Host-on / slot power |
| Reset / Clock | 前置相依條件 |
| Driver | Kernel driver |
| Raw Interface | Sysfs / dev node / hwmon / IIO / netdev |
| OpenBMC Service | 讀取或控制裝置的 daemon |
| D-Bus Mapping | Inventory / sensor / state 或 event |
| External Mapping | Redfish / IPMI / EventLog |
| Owner | HW, BMC, BIOS, CPLD, Security 等 |
| Debug Risk | Read-clear / write side effect / reset / erase 等 |

範例:

| Bus | Controller | Linux Interface | Topology | Device | Identity | Driver / Service |
|---|---|---|---|---|---|---|
| I2C | BMC I2C7 | `i2c-20` | Mux `0x70` ch2 | PSU0 | `0x58` | PMBus / PSU Sensor |
| SPI | FMC | `spi0.0` | CS0 | BMC flash | CS0 | SPI-NOR / MTD |
| UART | UART5 | `ttyS4` | Header | BMC console | 115200 8N1 | Serial / getty |
| PECI | PECI0 | `peci-0` | Direct | CPU0 | Package address | PECI hwmon |
| Ethernet | MAC1 | `eth1` | NC-SI | Host NIC | Package / channel | Kernel NC-SI |
| MCTP | SMBus binding | MCTP link | Mux path | Retimer | EID | MCTP / PLDM |

Linux bus number / netdev name 與 gpiochip number 可能受 probe 順序或 topology 影響, 因此 bus map 必須同時保留 physical identity 與 runtime identity.

## 5.5 Device Tree / Kernel 與 Service 的分工

### 5.5.1 Device Tree

Device Tree 常描述:

- Controller 是否啟用.
- Register range 與 interrupt.
- Clock / reset / power supply.
- Pinmux.
- Bus frequency.
- Child device address / chip select 與 compatible.
- Mux topology.

### 5.5.2 Kernel

Kernel 提供:

- Controller driver.
- Bus framework.
- Protocol / device driver.
- Sysfs / character device / hwmon / IIO 或 netdev interface.

### 5.5.3 OpenBMC Service

OpenBMC service 負責:

- 依 inventory 與 power state 決定何時存取裝置.
- 將 raw data 轉成 D-Bus properties.
- 建立名稱與 associations.
- 處理 unavailable / retry / event 與 policy.
- 接收控制要求.

### 5.5.4 Runtime 資料優先

Source DTS 只表示預期設計. 排查 target 時應確認:

- Bootloader 實際載入哪份 DTB.
- `/proc/device-tree` 的內容.
- Controller 與 child device 是否已建立.
- Driver symlink.
- Service 使用的設定檔與版本.

<a id="section-5-6"></a>

## 5.6 Power / Reset / Clock 與 Pinmux

匯流排 controller 出現在 Linux, 不代表外部裝置已可存取. 周邊裝置通常依賴:

- Power rail.
- PGOOD.
- Reset deassert.
- Reference clock.
- Pinmux state.
- Level shifter power.
- Bus mux select.
- Host / slot power state.

常見判讀:

| 現象 | 建議排查方向 |
|---|---|
| Controller 存在, 所有 devices 都無回應 | Pinmux / clock / pull-up / mux / power |
| Host on 後才出現 | 裝置位於 Host power domain |
| BMC reboot 後短暫消失 | Reset default / mux handoff / service timing |
| AC cycle 才能恢復 | Device / bridge state / rail discharge / CPLD latch |

這些前置條件應和第 3 章 Pinmux / GPIO / 第 4 章 Reset / Clock / Power Domain 的資料表對齊.

<a id="section-5-7"></a>

## 5.7 I2C / SMBus 與 PMBus 概覽

I2C 是 BMC 最常見的周邊匯流排, 常連接:

- EEPROM.
- Temperature sensor.
- GPIO expander.
- CPLD.
- PSU / VR / hot-swap controller.
- Fan controller.
- Clock generator.
- Retimer.
- MCTP endpoints.

本節提供跨章節所需的概覽; adapter / client / driver / mux / SMBus 與 PMBus 的細節由第 10 章說明.

### 5.7.1 Physical Bus 與 Logical Bus

```mermaid
flowchart TB
    A["BMC I2C Controller"] --> B["Root Adapter i2c-5"]
    B --> C["I2C Mux @0x70"]
    C --> D["ch0 → i2c-20 → PSU0 @0x58"]
    C --> E["ch1 → i2c-21 → PSU1 @0x58"]
    C --> F["ch2 → i2c-22 → FRU EEPROM @0x50"]
```

Schematic 上的 `BMC_I2C5` 是 physical controller; Linux `i2c-20` 則可能是 mux 建立的 child adapter.

### 5.7.2 必要資料

- Physical controller.
- Linux adapter.
- Mux address 與 channel.
- Pull-up voltage 與 resistor.
- Bus frequency.
- 7-bit address.
- Address strap.
- Device power state.
- Protocol 與 driver.
- Safe read method.
- Bus recovery 方式.

### 5.7.3 Target 檢查

```bash
$ i2cdetect -l
$ ls -l /sys/bus/i2c/devices

$ for bus in /sys/bus/i2c/devices/i2c-*; do
$ echo "==== $bus"
$ readlink -f "$bus"
$ done
```

### 5.7.4 掃描風險

`i2cdetect` 會對多個 addresses 發出 probing commands. 某些 devices 可能將 probing command 解讀成有效命令. 使用前應先查 bus map 與 datasheet, 並避開未知 / 共享或高風險的 bus.

PMBus `CLEAR_FAULTS` / page selection / EEPROM write 與 CPLD write registers 都可能改變裝置狀態. 故障排查先保存 status, 再依核准流程清除或寫入.

### 5.7.5 Bus Stuck

| 線路狀態 | 常見方向 |
|---|---|
| SCL High / SDA Low | Target 尚未完成傳輸或持續拉低 SDA |
| SCL Low | Controller / target 持續拉低 clock |
| Mux 下游全無回應 | Channel / mux reset / 下游 pull-up / power |
| 只在特定 power state 無回應 | Power domain 或 level shifter |

Recovery 可能包含 controller reset / SCL pulses / mux reset / target reset 或 device power cycle. 使用何種方式取決於平台硬體與 driver 支援.

<a id="section-5-8"></a>

## 5.8 SPI 概覽

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

### 5.8.1 SPI-NOR 與 SPI-NAND

SPI-NOR 常由 `spi-nor` driver 建立 MTD device. SPI-NAND 還需要處理 page / OOB / ECC 與 bad blocks. Flash layout / MTD / UBI 與更新流程由第 2 章深入說明.

### 5.8.2 Device Tree 範例

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

### 5.8.3 Target 檢查

```bash
$ dmesg | grep -Ei 'spi|spi-nor|spi-nand|jedec|mtd'
$ ls -l /sys/bus/spi/devices
$ cat /proc/mtd
```

### 5.8.4 常見問題

| 現象 | 優先檢查 |
|---|---|
| JEDEC ID 全 `00` / `ff` | Power / CS / pinmux / MISO |
| ID 偶發錯誤 | Clock / mode / signal integrity |
| Read 正常 / program 失敗 | WP / lock bits / supply / opcode |
| Quad mode 失敗 | QE bit / IO2 / IO3 / pinmux |
| Kernel 可讀 / BootROM 不啟動 | Boot header / offset / BootROM capability |

### 5.8.5 spidev

spidev 適合開發期間驗證簡單 protocol. 正式產品應優先使用具備 device semantics / locking / power management 與安全邊界的正式 driver 或專用 service.

Erase / program 與 register write 可能改變 boot flash / TPM / CPLD 或其他安全元件, 執行前需要備份與 recovery path.

<a id="section-5-9"></a>

## 5.9 UART 與 Serial Console

UART 是 BMC bring-up 的基本診斷入口. 需要記錄:

- Controller instance.
- Linux TTY.
- Baud rate / data bits / parity / stop bits.
- Signal voltage.
- Header pinout 與 GND.
- RTS / CTS.
- Pinmux 與 mux owner.
- Bootloader / kernel / login 或 Host SOL 的角色.

### 5.9.1 Console Path

```mermaid
flowchart TB
    A["SoC UART"] --> B["Pinmux"]
    B --> C["Level Shifter / CPLD Mux"]
    C --> D["Debug Header 或 Host"]
    D --> E["Bootloader / Kernel / getty / SOL"]
```

### 5.9.2 Target 檢查

```bash
$ cat /proc/cmdline | tr ' ' '\n' | grep console
$ cat /proc/tty/driver/serial 2>/dev/null
$ ls -l /dev/ttyS* /dev/ttyAMA* 2>/dev/null
$ systemctl list-units 'serial-getty@*'
```

### 5.9.3 常見問題

- 完全無輸出: Power / pinmux / TX path / mux / console parameter.
- 亂碼: Baud / clock parent / 電壓 / data format.
- Bootloader 有輸出, kernel 後消失: Kernel console / pinctrl / driver / getty.
- BMC console 與 Host console 混線: CPLD / mux owner 或文件名稱不清楚.

BMC local console 與 Host SOL 應使用不同名稱, 並在 service / connector 文件中清楚標示.

<a id="section-5-10"></a>

## 5.10 ADC 與 IIO

ADC 將類比電壓轉成數位值, Linux 常透過 IIO 或 hwmon 暴露.

完整轉換:

```mermaid
flowchart TB
    A["Rail Voltage"] --> B["Voltage Divider / Filter"]
    B --> C["ADC Input"]
    C --> D["Raw Code"]
    D --> E["Scale / Offset / Calibration"]
    E --> F["D-Bus Sensor Value"]
```

需要記錄:

- ADC channel.
- Input range.
- Reference voltage.
- Resolution.
- Divider values.
- Unit.
- Scale / offset formula.
- Calibration.
- Power-state dependency.

Target:

```bash
$ dmesg | grep -Ei 'adc|iio'
$ find /sys/bus/iio/devices -maxdepth 3 -type f | sort
$ find /sys/class/hwmon -maxdepth 3 -type f | sort
```

ADC 讀值為 0 或滿量程時, 先以 DMM 確認實體 input, 再核對 channel / reference / divider 與 scaling.

## 5.11 PWM 與 Tach

PWM 控制 fan duty; tach input 量測 fan speed.

```mermaid
flowchart TB
    A["Thermal Policy"] --> B["Fan Control Service"]
    B --> C["PWM Channel"]
    C --> D["Fan"]
    D --> E["Tach Pulses"]
    E --> F["RPM Sensor"]
```

必要資料:

- PWM channel 與 frequency.
- Duty representation.
- Polarity.
- Tach channel.
- Pulses per revolution.
- Fan power 與 presence.
- Minimum duty / start duty.
- Control owner.

Target:

```bash
$ dmesg | grep -Ei 'pwm|tach|fan'
$ find /sys/class/hwmon -maxdepth 4 -type f | \
$ grep -E 'fan[0-9]+_input|pwm[0-9]+|name|label'
$ find /sys/class/pwm -maxdepth 5 -type f 2>/dev/null
```

Fan daemon 正在控制 PWM 時, 手動寫入可能立刻被 policy 覆蓋. 測試前應使用產品定義的 manual / maintenance mode.

## 5.12 PECI 與 APML

PECI 常用於 Intel CPU / DIMM telemetry; APML 相關介面常用於 AMD 平台, 例如 SB-TSI 與 SB-RMI.

這些介面通常依賴 Host power / CPU reset 與 socket presence.

| 介面 | 常見用途 | 主要相依條件 |
|---|---|---|
| PECI | CPU / DIMM temperature / power / debug | CPU package power / address / driver |
| SB-TSI | AMD CPU temperature | I2C path / address / Host state |
| SB-RMI | AMD management mailbox | I2C path / mailbox / firmware support |

Target:

```bash
$ dmesg | grep -Ei 'peci|sbtsi|sbrmi|apml'
$ find /sys/bus/peci -maxdepth 4 -type f 2>/dev/null
$ find /sys/class/hwmon -maxdepth 4 -type f | sort
```

Host off 或 CPU 尚未 ready 時, sensor 應標示 unavailable, 避免將正常的 power-state dependency 誤報為 hardware fault.

## 5.13 eSPI / LPC 與 Host Interface 概覽

eSPI / LPC 是 Host chipset 與 BMC 之間的 sideband path, 可承載:

- KCS / BT IPMI system interface.
- Port 80 POST code.
- Virtual Wire.
- Mailbox.
- Flash access.
- OOB management.

本節只建立整體位置; KCS / BT / SSIF 與 eSPI 的狀態機 / channels 與雙端排查由第 18 章說明.

```mermaid
flowchart TB
    A["Host Firmware / OS"] --> B["LPC 或 eSPI"]
    B --> C["BMC Controller"]
    C --> D["KCS / BT / Port80 / VW / OOB / Flash"]
    D --> E["OpenBMC Host Services"]
```

這條路徑常依賴 RSMRST / PLTRST / PCH power 與 eSPI clock. BMC driver 正常不代表 Host side 已完成初始化.

Target:

```bash
$ dmesg | grep -Ei 'espi|lpc|kcs|bt|ipmi|port80|postcode'
$ ls -l /dev/ipmi* /dev/kcs* 2>/dev/null
$ systemctl --type=service | grep -Ei 'ipmi|postcode|host'
```

## 5.14 Ethernet MAC / PHY 與 MDIO

Dedicated management LAN 通常由 BMC MAC 連接外部 PHY.

```mermaid
flowchart TB
    A["BMC Ethernet MAC"] -->|"RGMII / RMII"| B["PHY"]
    B --> C["RJ45 / Network"]
    D["BMC MDIO Controller"] --> E["PHY Control / Status Registers"]
```

### 5.14.1 RGMII / RMII

- RGMII 需要正確處理 TX / RX clock delay.
- RMII 通常需要 50 MHz reference clock.
- PHY straps 決定 address / mode 與 clock role.
- PHY reset release timing 會影響 strap sampling.

### 5.14.2 MDIO

MDIO 用來讀寫 PHY registers. PHY address 由 strap 決定, Device Tree `reg` 必須一致.

### 5.14.3 Target

```bash
$ dmesg | grep -Ei 'ethernet|mac|phy|mdio|rgmii|rmii'
$ ip link
$ ethtool <interface>
$ ethtool -S <interface>
```

No link 時依序確認 MAC driver / PHY probe / MDIO address / reset / reference clock / `phy-mode` 與 board delay.

## 5.15 NC-SI

NC-SI 讓 BMC 透過 Host NIC 的 network port 傳送管理流量.

需要記錄:

- BMC netdev.
- NC-SI package / channel.
- NIC standby power.
- Host-on / Host-off behavior.
- MAC address policy.
- VLAN / filters.
- AEN.
- Channel selection / failover.
- NIC reset recovery.

```bash
$ dmesg | grep -Ei 'ncsi|package|channel|AEN'
$ ip link
$ networkctl status 2>/dev/null
```

驗證情境:

- BMC boot / Host off.
- Host power on / off.
- NIC reset.
- Cable insert / remove.
- Channel failover.
- BMC reboot.

管理網路是否在 Host off 時可用, 取決於 NIC 的 power 與產品設計, 應在規格與測試紀錄中明確標示.

## 5.16 PCIe 管理 Sideband

BMC 可能不參與 Host PCIe enumeration, 但可以管理:

- Slot power.
- Presence.
- PERST.
- Reference clock enable.
- Retimer / switch sideband.
- MCTP over PCIe VDM.
- Device firmware update.
- SPDM attestation.

```mermaid
flowchart TB
    A["BMC Platform Control"] --> B["Slot Power / PGOOD"]
    A --> C["Presence / Fault"]
    A --> D["PERST / Clock"]
    A --> E["SMBus Sideband"]
    A --> F["MCTP over PCIe"]
```

PCIe sideband control 需和 Host power sequence 協調. 任意 reset retimer / clock buffer 或 endpoint 可能造成 Host link 失效.

## 5.17 USB Gadget

USB gadget 讓 BMC 以 USB device 身分連接 Host, 可提供:

- Virtual media.
- USB network.
- USB serial.
- HID.
- Provisioning interface.

資料路徑:

```mermaid
flowchart TB
    A["BMC USB Device Controller"] --> B["Linux UDC Driver"]
    B --> C["ConfigFS Gadget"]
    C --> D["USB Function"]
    D --> E["Host USB Driver"]
```

Target:

```bash
$ dmesg | grep -Ei 'usb|gadget|udc|configfs'
$ ls -l /sys/class/udc 2>/dev/null
$ find /sys/kernel/config/usb_gadget -maxdepth 4 -type f 2>/dev/null
```

需驗證 VBUS detect / role / Host driver / BMC / Host reboot / virtual media detach, 以及 field / manufacturing mode 的安全政策.

## 5.18 MCTP / PLDM 與 SPDM 概覽

MCTP 可承載在 SMBus / PCIe VDM / I3C 或其他 binding 上, 並以 EID 識別 endpoint. PLDM 提供平台管理功能; SPDM 提供裝置認證與安全 session.

```mermaid
flowchart TB
    A["Physical Binding"] --> B["MCTP Link / Network / EID / Route"]
    B --> C["PLDM / SPDM / NVMe-MI / Vendor Message"]
    C --> D["OpenBMC Services"]
```

本節只建立與周邊匯流排的關係; Endpoint / EID / routing / PDR / firmware update 與 attestation 由第 20 章說明.

基本檢查:

```bash
$ dmesg | grep -Ei 'mctp|pldm|spdm|eid'
$ command -v mctp >/dev/null && mctp link
$ command -v mctp >/dev/null && mctp route
$ systemctl --type=service | grep -Ei 'mctp|pldm|spdm'
```

MCTP endpoint 可見後, 仍需驗證 supported message types / PLDM terminus / PDR 或 SPDM capabilities.

## 5.19 OpenBMC Service Integration

周邊裝置通常需要 service 才會成為對外可用的管理資料.

PSU 範例:

```mermaid
flowchart TB
    A["I2C / PMBus PSU"] --> B["Kernel PMBus Driver"]
    B --> C["Hwmon"]
    C --> D["PSU Sensor Service"]
    D --> E["D-Bus Sensors + Inventory Association"]
    E --> F["Redfish PowerSupply / Sensor / Event"]
```

Service integration 需確認:

- Probe / discovery condition.
- Power-state gating.
- Presence gating.
- Retry 與 late-ready behavior.
- Sensor / inventory naming.
- Associations.
- Unavailable 與 Functional policy.
- Event debounce.
- Service dependency.

```bash
$ systemctl --failed
$ systemctl --type=service | grep -Ei 'sensor|entity|mctp|pldm|network'
$ journalctl -b --no-pager | grep -Ei 'sensor|inventory|bus|timeout'
$ busctl tree xyz.openbmc_project.ObjectMapper | head -200
```

<a id="section-5-20"></a>

## 5.20 Debug Safety

周邊工具可能改變硬體狀態或清除證據.

| 動作 | 風險 | 安全原則 |
|---|---|---|
| I2C bus scan | Device 對 probe command 有反應 | 先查 bus map 與 datasheet |
| EEPROM write | FRU / VPD 損壞 | Write protect / 備份 / verify |
| PMBus status clear | Fault evidence 消失 | 先保存 status 與 journal |
| SPI erase / program | Boot / recovery image 損壞 | 確認 range / 備份與 recovery |
| GPIO output 切換 | Power / reset / mux 改變 | 確認 owner 與測試窗口 |
| PWM manual write | Fan policy 被覆蓋 | 使用 maintenance / manual mode |
| eSPI / KCS control | 影響 Host state | 雙端協調與權限控管 |
| PLDM effecter / update | 改變 endpoint state | 區分 query 與 control |
| USB gadget enable | 暴露 storage / network path | 依 field / factory policy 限制 |

通用 debug script 應以唯讀收集為主, 不自動掃描所有 I2C buses / 寫入 registers / 切換 GPIO outputs 或觸發 update.

<a id="section-5-21"></a>

## 5.21 跨 Power State 驗證

周邊介面會隨平台狀態改變. 至少測試:

| 狀態 | 檢查內容 |
|---|---|
| AC applied / BMC booting | Safe defaults / always-on buses |
| BMC ready / Host off | Standby devices / NC-SI policy / Host interfaces |
| Host powering on | PECI / APML / eSPI / slot devices 出現時機 |
| Host on | 完整 sensors / network / MCTP endpoints |
| Host powering off | Service 停止順序 / unavailable events |
| BMC reboot | Host 影響 / mux / GPIO / sideband recovery |
| AC cycle | Controller / bridge 與 endpoint 重新 discovery |
| Hot-plug | Presence / driver / inventory / sensor lifecycle |

Power-state dependency 應由 service 正常處理, 避免裝置尚未供電時產生大量 timeout 與 critical events.

## 5.22 常見問題與判讀

| 現象 | 優先層級 | 第一輪檢查 |
|---|---|---|
| Controller 不存在 | DTS / clock / reset / driver | `dmesg` / running DT / kernel config |
| Controller 存在, 裝置無回應 | Power / pinmux / topology | Scope / reset / mux / address |
| Driver 未 bind | Compatible / ID / dependency | Sysfs driver link / `dmesg` |
| Raw value 存在, D-Bus 沒有 | Service / config | Journal / Probe / PowerState |
| D-Bus 存在, Redfish 沒有 | Mapping / association | ObjectMapper / bmcweb journal |
| Host off 時持續 timeout | Power-state gating | Host state / service policy |
| BMC reboot 後無法恢復 | Ownership / rediscovery | Controller reset / service restart / route |
| Hot-plug 後留下舊資料 | Lifecycle / cache | Inventory / sensor / association cleanup |
| Debug 後狀態改變 | Command side effect | Tool history / status / reset / clear logs |
| 偶發錯誤只在高負載出現 | Timing / signal integrity / contention | Scope / clock / latency / concurrency |

<a id="section-5-23"></a>

## 5.23 共用 Debug Log 收集

以下腳本只收集一般狀態, 不執行 bus scan / register write 或 control command:

```bash
#!/bin/sh

OUT=/tmp/peripheral-bus-debug
mkdir -p "$OUT"

cat /etc/os-release > "$OUT/os-release.txt" 2>&1
uname -a > "$OUT/uname.txt"
cat /proc/cmdline > "$OUT/proc-cmdline.txt"
zcat /proc/config.gz > "$OUT/kernel-config.txt" 2>&1

dmesg -T > "$OUT/dmesg.txt"
journalctl -b --no-pager > "$OUT/journal.txt" 2>&1
systemctl --failed > "$OUT/systemctl-failed.txt" 2>&1

command -v i2cdetect >/dev/null 2>&1 && \
    i2cdetect -l > "$OUT/i2c-adapters.txt" 2>&1
ls -l /sys/bus/i2c/devices > "$OUT/i2c-devices.txt" 2>&1
ls -l /sys/bus/spi/devices > "$OUT/spi-devices.txt" 2>&1
cat /proc/mtd > "$OUT/proc-mtd.txt" 2>&1

cat /proc/tty/driver/serial > "$OUT/serial.txt" 2>&1
find /sys/class/hwmon -maxdepth 4 -type f \
    > "$OUT/hwmon-files.txt" 2>&1
find /sys/bus/iio/devices -maxdepth 4 -type f \
    > "$OUT/iio-files.txt" 2>&1
find /sys/bus/peci -maxdepth 4 -type f \
    > "$OUT/peci-files.txt" 2>&1

ip link > "$OUT/ip-link.txt" 2>&1
ip addr > "$OUT/ip-addr.txt" 2>&1

command -v mctp >/dev/null 2>&1 && {
    mctp link > "$OUT/mctp-link.txt" 2>&1
    mctp route > "$OUT/mctp-route.txt" 2>&1
}

busctl tree xyz.openbmc_project.ObjectMapper \
    > "$OUT/objectmapper.txt" 2>&1
busctl tree xyz.openbmc_project.State.Host \
    > "$OUT/host-state.txt" 2>&1
busctl tree xyz.openbmc_project.State.Chassis \
    > "$OUT/chassis-state.txt" 2>&1

tar czf "/tmp/peripheral-bus-debug-$(date +%Y%m%d-%H%M%S).tar.gz" \
    -C /tmp peripheral-bus-debug
```

## 5.24 Bring-up 順序

1. 建立所有 controllers / topologies / devices 與 endpoints 的 bus map.
2. 確認每條路徑的 power / reset / clock / pinmux 與 mux select.
3. 驗證 controller driver 與 Linux runtime interface.
4. 驗證 child device / address / CS / PHY address / package / channel 或 EID.
5. 驗證 protocol / driver 與 raw interface.
6. 確認 OpenBMC service 的 discovery / gating / retry 與 naming.
7. 建立 inventory / sensor / state 與 associations.
8. 比對 Redfish / IPMI 與 physical device.
9. 執行 BMC reboot / Host power transition / AC cycle 與 hot-plug.
10. 執行可控的 timeout / disconnect 與 recovery 測試.
11. 保存 scope / LA / logs / DTB / kernel / service 與 firmware versions.
12. 將已知副作用與安全限制寫回 bus map.

## 5.25 平台實測紀錄表

| Bus | Controller | Runtime Interface | Topology | Device / Endpoint | Identity | Power State | Driver / Service | Result |
|---|---|---|---|---|---|---|---|---|
| I2C | [待填] | [待填] | [待填] | [待填] | Address [待填] | [待填] | [待填] | [待確認] |
| SPI | [待填] | [待填] | [待填] | [待填] | CS [待填] | [待填] | [待填] | [待確認] |
| UART | [待填] | [待填] | [待填] | [待填] | Baud [待填] | [待填] | [待填] | [待確認] |
| ADC / IIO | [待填] | [待填] | Channel [待填] | [待填] | Scale [待填] | [待填] | [待填] | [待確認] |
| PWM / Tach | [待填] | [待填] | Channel [待填] | Fan [待填] | PPR [待填] | [待填] | [待填] | [待確認] |
| PECI / APML | [待填] | [待填] | [待填] | CPU [待填] | Address [待填] | Host-on | [待填] | [待確認] |
| eSPI / LPC | [待填] | [待填] | [待填] | Host | Channel [待填] | [待填] | [待填] | [待確認] |
| Ethernet / NC-SI | [待填] | [待填] | [待填] | PHY / NIC | [待填] | [待填] | [待填] | [待確認] |
| USB Gadget | [待填] | [待填] | [待填] | Host | Function [待填] | Host-on | [待填] | [待確認] |
| MCTP | [待填] | [待填] | [待填] | [待填] | EID [待填] | [待填] | [待填] | [待確認] |

每列再附:

- Schematic page.
- DTS node.
- Kernel config / driver.
- Safe read method.
- Debug risk.
- D-Bus object.
- Redfish / IPMI mapping.
- 已驗證 power states.
- Recovery方式.

<a id="section-5-26"></a>

## 5.26 驗收 Checklist

架構與文件:

- [ ] 所有 controllers / devices / addresses / topologies 與 owners 已納入 bus map.
- [ ] Physical controller 與 Linux runtime interface 可互相追蹤.
- [ ] Power / reset / clock / pinmux 與 mux dependencies 已記錄.
- [ ] DTS / kernel / Yocto / service config 與 runtime 狀態一致.

介面:

- [ ] I2C adapters / mux paths / 7-bit addresses 與 safe reads 已驗證.
- [ ] SPI mode / clock / CS / bus width / WP 與 recovery 已驗證.
- [ ] UART voltage / baud / pinout / console role 與 mux owner 已驗證.
- [ ] ADC channel / reference / divider / scale 與 unit 已驗證.
- [ ] PWM frequency / polarity / tach PPR 與 control owner 已驗證.
- [ ] PECI / APML 的 package identity 與 Host-state gating 已驗證.
- [ ] eSPI / LPC Host interfaces在 reset / power transitions 後可恢復.
- [ ] Ethernet PHY / MDIO / RGMII / RMII 或 NC-SI 已完成狀態測試.
- [ ] USB gadget role / functions / Host compatibility 與安全政策已驗證.
- [ ] MCTP link / EID / route 與上層 protocol discovery 已驗證.

OpenBMC 與安全:

- [ ] Raw interfaces / D-Bus objects / inventory 與 associations 正確.
- [ ] Redfish / IPMI / EventLog 與硬體狀態一致.
- [ ] Power-state gating / retry / hot-plug 與 unavailable policy 已測試.
- [ ] 高風險 debug commands 具有核准流程 / 備份與 recovery.
- [ ] 共用 debug script 不會執行 bus scan 或修改硬體狀態.
- [ ] BMC reboot / Host power cycle / AC cycle 與 service restart regression 已完成.

## 5.27 本章重點

1. 周邊介面應從硬體前置條件 / controller / device / protocol / service 一路追到 Redfish / IPMI.
2. Controller / device與 protocol 屬於不同層次.
3. Bus map 需同時保存 physical identity / Linux runtime identity 與 OpenBMC mapping.
4. Controller probe 成功後, 仍需確認外部裝置的 power / reset / clock 與 pinmux.
5. I2C mux / NC-SI package / channel 與 MCTP EID都會形成新的 runtime topology.
6. Raw interface 有資料, 不代表 OpenBMC service / association與外部介面已完成.
7. 周邊裝置的 availability 常依賴 Host / slot 或 hot-plug power state.
8. Debug tools 可能清除 fault / 改寫 nonvolatile data / 切換 power 或破壞 boot image.
9. Recovery 測試應涵蓋 BMC reboot / Host transition / AC cycle / hot-plug 與 service restart.
10. I2C / PMBus / KCS / eSPI與 MCTP / PLDM / SPDM的詳細原理應由各自專章承接.

## 5.28 本章參考資料

- Linux kernel documentation - I2C: https://docs.kernel.org/i2c/
- Linux kernel documentation - SPI:https://docs.kernel.org/spi/
- Linux kernel documentation - Serial:https://docs.kernel.org/driver-api/serial/
- Linux kernel documentation - IIO:https://docs.kernel.org/driver-api/iio/
- Linux kernel documentation - Hwmon:https://docs.kernel.org/hwmon/
- Linux kernel documentation - PECI:https://docs.kernel.org/peci/
- Linux kernel documentation - MCTP:https://docs.kernel.org/networking/mctp.html
- Linux kernel networking documentation:https://docs.kernel.org/networking/
- DMTF PMCI standards:https://www.dmtf.org/standards/pmci
- OpenBMC documentation:https://github.com/openbmc/docs
