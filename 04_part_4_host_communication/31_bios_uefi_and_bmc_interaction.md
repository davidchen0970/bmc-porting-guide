# 31. BIOS、UEFI 與 BMC 互動

BMC 負責平台管理、電源控制與遠端介面；BIOS / UEFI 負責初始化 Host 硬體並啟動作業系統。兩者會交換 boot progress、POST code、boot settings、Host inventory、firmware version、event 與 reset state。

本章建立 BIOS / UEFI 與 BMC 之間的介面契約，說明資料由誰產生、透過哪條通道送出、何時有效，以及如何映射到 OpenBMC D-Bus、Redfish 與 IPMI。

## 31.1 BIOS、UEFI 與 BMC 的角色

### 31.1.1 BMC

BMC 在 Host 尚未開機時就能運作，常負責：

- Host power sequence。
- Sensor、fan 與 fault monitoring。
- Remote console 與 virtual media。
- Firmware update。
- Inventory 與 event log。
- Redfish、IPMI 與其他管理介面。

### 31.1.2 BIOS / UEFI

BIOS / UEFI 在 Host power-on 後執行，常負責：

- CPU、memory、chipset 與 PCIe initialization。
- Hardware enumeration。
- Secure Boot 與 platform security policy。
- SMBIOS 與 ACPI tables。
- Boot device selection。
- OS loader 啟動。
- 將 Host boot 狀態與 inventory 交給 BMC。

### 31.1.3 共同責任

部分功能需要兩邊協作：

| 功能 | BIOS / UEFI | BMC |
|---|---|---|
| Boot progress | 產生目前階段 | 保存、呈現、判斷 timeout |
| POST code | 輸出 code | 擷取、保存、提供遠端查詢 |
| Boot override | 套用下次開機設定 | 接收 Redfish / IPMI 要求並保存 |
| Host inventory | 蒐集 CPU、DIMM、PCIe 資訊 | 建立 inventory 與對外資源 |
| BIOS settings | 提供 attributes 與生效規則 | 提供遠端讀寫與 pending state |
| Host reset | 配合 firmware / OS 流程 | 發出 reset 或 power transition |
| Error reporting | 產生 firmware error records | 建立 event、dump 與健康狀態 |

## 31.2 BIOS–BMC Interface Contract

每一項跨邊界功能都應建立明確契約。

| 欄位 | 說明 |
|---|---|
| Feature | Boot progress、inventory、BIOS setting 等 |
| Producer | 哪一方產生資料 |
| Consumer | 哪一方使用資料 |
| Transport | LPC / eSPI、KCS、PLDM、Redfish Host Interface 等 |
| Timing | Pre-boot、POST、OS runtime 或 shutdown |
| Data Format | Enum、table、record、attribute 或 binary payload |
| Persistence | BMC reboot / Host reboot 後是否保留 |
| Timeout | 多久未更新視為異常 |
| Retry | 由哪一方重送或重新同步 |
| Error Handling | Reject、fallback、stale、event 或 recovery |
| Security | 權限、驗證、完整性與敏感資料限制 |
| Versioning | BIOS、BMC 與 protocol version 相容規則 |

範例：

| Feature | Transport | Producer | Timing | Data | Error Handling |
|---|---|---|---|---|---|
| POST code | LPC / eSPI | BIOS | POST | Code + table version | 保存最後狀態與 timestamp |
| Boot progress | IPMI / PLDM / OEM | BIOS | POST | Enum | Timeout 後標示 boot failure |
| Boot override | Redfish / IPMI | BMC | Pre-boot | Source、mode、one-time | BIOS 接受、拒絕或回報不支援 |
| Host inventory | PLDM / SMBIOS / OEM | BIOS | POST / runtime | Structured records | 舊資料標示 stale |
| BIOS attributes | PLDM / Redfish flow | BIOS / BMC | Pre-boot | Attribute registry | Pending / applied / rejected |

## 31.3 Power-On 到 OS Ready 的互動時間線

```text
AC Applied
    ↓
BMC Boot / Platform Initialization
    ↓
BMC Ready for Host Power Control
    ↓
Host Power-On Request
    ↓
Power Sequence / Reset Release
    ↓
BIOS / UEFI SEC、PEI、DXE、BDS
    ↓
POST Code 與 Boot Progress 更新
    ↓
Host Inventory / BIOS Attributes 同步
    ↓
OS Loader
    ↓
OS Kernel / Userspace Ready
```

每個狀態需要時間戳與 timeout。若只保存最後一個 boot progress enum，可能無法判斷 Host 在哪個階段停留多久。

### 31.3.1 BMC Ready

BMC 在允許 Host power-on 前，通常需要完成：

- Power-control service ready。
- CPLD / GPIO / sensor 基本初始化。
- Watchdog 與 reset policy ready。
- 必要 firmware version / security check。
- Host interface ready，依平台需求。

### 31.3.2 Host Ready

「Host ready」需要產品明確定義，例如：

- BIOS POST complete。
- OS loader 已啟動。
- OS kernel 已啟動。
- Host agent 已回報 ready。
- Network service 可用。

不同定義不能共用同一個布林狀態。

## 31.4 POST Code

POST code 是 BIOS / UEFI 在初始化過程中輸出的階段代碼。常見傳輸路徑包括 Port 80、LPC、eSPI Peripheral Channel 或平台自訂 mailbox。

```text
BIOS Initialization Step
        ↓
Write POST Code
        ↓
Chipset LPC / eSPI Decode
        ↓
BMC POST Code Controller
        ↓
Kernel Driver / Service
        ↓
D-Bus / Redfish / Log
```

### 31.4.1 Code Table

POST code 只有搭配 BIOS build 對應的 code table 才有意義。紀錄至少包含：

- BIOS vendor / project。
- BIOS version / build ID。
- Code width，8-bit、16-bit 或多-byte sequence。
- Code table revision。
- Socket / Host instance，若為 multi-host。
- Timestamp。

### 31.4.2 Last Code 與 History

- Last code：目前或最後觀察到的 code。
- History：按時間保存的一連串 codes。

History 才能看出重複、倒退、長時間停留與 reset。Ring buffer 大小與保存週期需依產品需求定義。

### 31.4.3 POST Code 排查

完全沒有 POST code時，先檢查：

- Host 是否真的開始執行 firmware。
- LPC / eSPI link 與 decode。
- Port range。
- BMC driver與 service。
- BIOS 是否啟用輸出。
- Multi-host routing。

Code 停在固定值時，再依該 BIOS build 的 code table確認初始化階段，並搭配 Host serial log、power rails、reset 與 firmware error records。

## 31.5 Boot Progress

Boot progress 是標準化或平台化的高階啟動狀態，例如：

```text
Unspecified
Primary Processor Initialization
Memory Initialization
PCI Resource Configuration
System Hardware Initialization Complete
OS Start
OS Running
```

實際 enums 依 D-Bus interface、IPMI、PLDM 與 Redfish mapping而定。

### 31.5.1 POST Code 與 Boot Progress

POST code 細緻且 vendor-specific；boot progress 粗粒度且適合對外顯示。

```text
多個 POST Codes
        ↓ BIOS / Platform Mapping
一個 Boot Progress Stage
        ↓
D-Bus Boot Progress
        ↓
Redfish BootProgress / Event
```

### 31.5.2 OpenBMC State

OpenBMC 可保存：

- Current boot progress。
- Previous boot progress。
- Last update time。
- Boot cycle identity。
- Boot failure / timeout。

BIOS 重啟新一輪 POST 時，BMC 應重設本輪狀態，同時保留上一輪診斷資料，避免新舊 boot cycle 混在一起。

### 31.5.3 Timeout

每個階段可具有不同 timeout。Memory training 所需時間可能遠高於一般 DXE driver initialization。產品應依平台實測設定，不宜所有階段共用過短 timeout。

## 31.6 Boot Source、Boot Mode 與 One-Time Override

BMC 可透過 Redfish 或 IPMI 接收 boot override，再交給 BIOS / UEFI 套用。

常見設定：

- Boot source：Default、PXE、Disk、Optical、USB、BIOS Setup。
- Boot mode：UEFI、Legacy，依平台支援。
- Enabled：Disabled、Once、Continuous。
- UEFI target / boot option，依 schema 與 firmware 支援。

### 31.6.1 One-Time Boot

```text
Client 設定 BootSourceOverrideEnabled=Once
        ↓
BMC 保存 Pending Boot Override
        ↓
Host 下一次開機
        ↓
BIOS 讀取並套用
        ↓
BIOS / BMC 標記已消耗
        ↓
設定回到 Disabled
```

需要定義「哪一次開機算已消耗」。若 Host 在讀取設定前失敗，是否保留到下一次；若已讀取但 boot device 失敗，是否仍清除，均需寫入契約。

### 31.6.2 Continuous Override

Continuous 會影響後續多次開機，需提供清楚的解除方法並避免覆蓋使用者已設定的 UEFI BootOrder。

### 31.6.3 Unsupported Combination

BIOS 應能回報不支援的 source、mode 或 target。BMC 不應在未確認 BIOS capability 時對外宣告選項可用。

## 31.7 UEFI BootOrder、BootNext 與 Boot Options

UEFI 以 Boot## variables 表示 boot options，由 `BootOrder` 決定順序，`BootNext` 指定下一次優先項目。

需要處理：

- Boot option identity。
- Display name。
- Device path。
- Enabled / disabled。
- Current BootOrder。
- Pending BootOrder。
- BootNext consumption。
- Device removal後的 stale option。
- BIOS default restore。

若 Redfish 使用 BootOptions resources，BMC 與 BIOS 需同步 option reference，不可只依顯示名稱比對。

## 31.8 BIOS Configuration

Redfish BIOS resource通常提供 current attributes；Settings resource保存 pending values；Attribute Registry描述每個 attribute 的型別、範圍、可選值與相依條件。

```text
GET BIOS Resource
    → Current Attributes

PATCH BIOS Settings
    → Pending Attributes

Host Reboot / BIOS Apply
    → Validate and Apply

下一次同步
    → Current 更新、Pending 清除或保留錯誤
```

### 31.8.1 Attribute Types

常見：

- Enumeration。
- Integer。
- String。
- Boolean，依 registry model。
- Password，具有特殊處理規則。

### 31.8.2 Attribute Registry

Registry 需要描述：

- Attribute name。
- Display name / help text。
- Type。
- Read-only。
- Default。
- Current / pending behavior。
- Allowable values或範圍。
- Dependencies。
- Reset / reboot requirement。

BMC 對外的 registry 必須和目前 BIOS build相容。BIOS 更新後若 attributes 新增、刪除或改名，需要同步更新 registry 與 stored pending settings。

### 31.8.3 Pending 與 Current

- Current：BIOS 目前使用的值。
- Pending：下次 apply 時希望採用的值。

Pending 被拒絕時應保留明確 message，包括 attribute、requested value 與原因；不能只將 pending 靜默清除。

### 31.8.4 Reset BIOS Settings

恢復預設值可能影響 boot mode、security、virtualization、memory、PCIe 與 network boot。操作需要 privilege、audit log 與明確的 apply timing。

## 31.9 BIOS Configuration Transport

BIOS attributes 可透過不同通道同步：

- PLDM BIOS Control and Configuration。
- IPMI / OEM commands。
- Shared storage / mailbox。
- Redfish Host Interface。
- Platform-specific sideband。

Transport 需要支援：

- Capability / version discovery。
- Full table與 incremental update。
- Request / response correlation。
- Retry / timeout。
- BIOS reset後重新同步。
- BMC reboot後 state recovery。
- Table integrity / version。

秘密或 password attributes需要額外保護，不應以一般 log 或可讀 D-Bus property暴露明文。

## 31.10 Host Inventory

BIOS / UEFI 可提供：

- CPU inventory。
- DIMM inventory。
- PCIe devices / slots。
- System UUID。
- SMBIOS tables。
- Firmware versions。
- Boot options。

```text
BIOS Hardware Enumeration
        ↓
SMBIOS / PLDM / OEM Inventory Transfer
        ↓
BMC Host Inventory Service
        ↓
D-Bus Inventory Objects
        ↓
Redfish Processor / Memory / PCIe / System
```

### 31.10.1 資料權威端

| 資料 | 常見權威端 |
|---|---|
| CPU model / core count | BIOS / host firmware |
| DIMM size / type / serial | SMBIOS / SPD / host inventory |
| PCIe device | BIOS enumeration / MCTP / OS agent |
| System UUID | Manufacturing / SMBIOS |
| BIOS version | BIOS firmware / SMBIOS |

同一欄位若另有 BMC static config 或 MCTP endpoint資料，需要建立 priority與 conflict policy。

### 31.10.2 Stale Data

Host off 時 BMC可能只能提供上次成功開機的 inventory。這類資料應標示：

- Last update time。
- Source boot ID。
- Stale / unavailable policy。
- BIOS version。

Host inventory transfer失敗時，不應把舊資料當成本次最新結果。

## 31.11 SMBIOS

SMBIOS tables描述 system、baseboard、processor、memory、slots、firmware等資訊。BMC可透過 PLDM SMBIOS、OEM transport或其他平台路徑取得部分或完整table。

需要確認：

- SMBIOS version。
- Table entry point與length。
- Transfer完整性。
- Type 0、1、2、4、17、9、42等所需records。
- String index與空字串處理。
- Duplicate / missing handles。
- BIOS更新後table刷新。

SMBIOS Type 42可描述Redfish Host Interface，讓UEFI或Host OS發現平台提供的Redfish service。

## 31.12 Redfish Host Interface

Redfish Host Interface讓UEFI或Host OS透過in-band network path存取BMC Redfish service。Firmware通常透過SMBIOS Type 42提供interface與service discovery資料。

```text
UEFI / Host OS Redfish Client
        ↓
Host Interface Network Device
        ↓ HTTPS
BMC Redfish Service
        ↓
bmcweb / D-Bus Services
```

可能使用USB NIC、PCIe或其他Host-facing network interface，依平台實作。

需要驗證：

- SMBIOS Type 42內容。
- Host interface device enumeration。
- Host與BMC IP configuration。
- Service URI。
- TLS certificate validation。
- Credential acquisition與lifecycle。
- Host reboot / BMC reboot後rediscovery。
- 與out-of-band management network的隔離。

## 31.13 IPMI、KCS、BT 與 SSIF

Legacy Host–BMC管理常使用KCS、BT或SSIF承載IPMI commands。常見用途：

- Boot flags。
- Chassis status。
- Watchdog。
- SEL / sensor access。
- OEM BIOS communication。

Host system interface與BMC transport的詳細原理見第 30 章。本章重點是BIOS / BMC契約：command ownership、data version、timeout、boot-stage availability與reset recovery。

## 31.14 PLDM

PLDM可提供：

- Base discovery。
- BIOS attributes。
- SMBIOS transfer。
- Platform sensors / events。
- FRU。
- Firmware update。

BIOS / BMC使用PLDM時，需要保存MCTP Endpoint、EID、TID、Type / Command support與version。PLDM details見第 32 章。

## 31.15 Host Watchdog

Host可透過IPMI或其他management interface設定watchdog。需要定義：

- Watchdog owner：BIOS、OS、BMC policy或Host agent。
- Start timing。
- Timeout。
- Pre-timeout action。
- Expiration action。
- Reset target。
- BIOS→OS handoff。
- BMC reboot後watchdog state。

### 31.15.1 BIOS 到 OS 的交接

```text
BIOS 啟動 Watchdog
        ↓
BIOS 定期餵狗
        ↓
OS Loader / Kernel 接手
        ↓
OS Watchdog Driver / Agent 接手
```

若交接窗口過長，Host可能在正常啟動途中被reset。若BIOS離開前停用watchdog而OS沒有重新啟用，平台則失去boot hang保護。

### 31.15.2 Expiration Log

Watchdog觸發後需保存：

- Watchdog type / owner。
- Timeout與pre-timeout。
- 最後boot progress / POST code。
- Reset reason。
- Host與BMC時間戳。
- Previous boot log。

## 31.16 Error Reporting

BIOS / UEFI可能透過以下方式回報錯誤：

- POST code。
- Boot progress failure。
- IPMI SEL / OEM command。
- PLDM Platform Event。
- CPER records。
- ACPI error interface。
- Serial log。
- Memory training / machine check records。

BMC需要：

- 保存原始資料。
- 標示source與timestamp。
- 去除重複event。
- 建立inventory / component association。
- 映射到Redfish LogService / EventService。
- 依severity與policy更新health。

在clear hardware status或重新啟動Host前，先保存會因reset消失的error registers與firmware logs。

## 31.17 Secure Boot 與安全狀態

BIOS / UEFI可將以下狀態提供給BMC：

- Secure Boot enabled。
- Setup mode / deployed mode。
- Key enrollment state。
- TPM presence / state。
- Measured boot result。
- Firmware verification failure。
- Recovery mode。

BMC對外呈現時需區分：

- Configuration state。
- 當次boot驗證結果。
- Persistent security fault。
- User-requested recovery。

修改Secure Boot、key或TPM相關設定需要高權限、audit log、physical presence或產品定義的additional authorization。

## 31.18 Firmware Version 與 Update Coordination

BIOS update會影響attributes、SMBIOS、POST code table、inventory與security measurements。BMC應在update前後同步：

- BIOS version。
- Active / backup image。
- Attribute Registry version。
- Pending BIOS settings。
- POST code table revision。
- Secure Boot / measurement reference。
- Host inventory source boot。

### 31.18.1 Capsule / Update Flow

實際方式可能包含UEFI capsule、BMC sideband flash update、Host agent或vendor updater。無論使用哪種方式，都需定義：

- Image verification。
- Flash ownership與write protect。
- Host power state。
- Automatic reboot / activation。
- Update interruption。
- Rollback / recovery。
- Configuration preservation。

### 31.18.2 Pending Settings 與 Update

BIOS更新前若有pending settings，需要定義：

- 升版後是否保留。
- Attribute已移除時如何處理。
- Enum values改變時如何驗證。
- Downgrade後如何回復。
- 是否在update前先apply或清除。

## 31.19 Multi-Host 與 Multi-Socket

Multi-host平台需要為每個Host instance分開保存：

- Power state。
- Boot progress。
- POST code history。
- Boot override。
- BIOS attributes。
- Inventory。
- Watchdog。
- Reset reason。

URI、D-Bus path、KCS channel、Port80 source與PLDM terminus都需要Host identity。不能只使用單一全域狀態代表多個Hosts。

Multi-socket inventory則需要將CPU、DIMM與PCIe resources關聯到正確socket與Host。

## 31.20 BMC Reboot 與 Host Continuity

產品若要求BMC reboot期間Host維持運作，需要確認：

- Power / reset outputs由CPLD或safe hardware維持。
- KCS / eSPI / MCTP短暫中斷後可恢復。
- Watchdog沒有因BMC重啟誤觸發。
- BIOS settings、boot override與inventory state能從persistent storage回復。
- Host boot progress能重新同步或標示unknown / stale。
- Event不因service重新啟動而重複產生。

BMC reboot測試需在Host POST、OS runtime、firmware update與shutdown等不同階段執行。

## 31.21 OpenBMC D-Bus Mapping

常見對照：

```text
Host Power State
    → xyz.openbmc_project.State.Host

Boot Progress
    → xyz.openbmc_project.State.Boot.Progress

Boot Override
    → Host boot control interfaces

BIOS Attributes
    → xyz.openbmc_project.BIOSConfig.Manager

Host Inventory
    → xyz.openbmc_project.Inventory.Item.*

POST Code
    → POST code service / logging objects

Firmware Version
    → xyz.openbmc_project.Software.Version
```

實際service、path與interfaces依OpenBMC branch與平台整合而異。新增mapping時需確認：

- Object identity。
- Multi-host path。
- Persistence。
- Source authority。
- Current / pending semantics。
- Association。
- Error與unavailable handling。

## 31.22 Redfish Mapping

常見Redfish resources：

| 功能 | Resource / Property |
|---|---|
| Host identity / power | `ComputerSystem` |
| Boot override | `ComputerSystem.Boot` |
| Boot progress | `ComputerSystem.BootProgress` |
| BIOS attributes | `Systems/{id}/Bios` |
| Pending settings | `Systems/{id}/Bios/Settings` |
| Attribute metadata | `Registries` / Attribute Registry |
| Host inventory | Processor、Memory、PCIe、Storage resources |
| BIOS version | `ComputerSystem.BiosVersion` |
| Secure Boot | `SecureBoot` resource |
| Boot options | `BootOptions` collection |
| Firmware errors | LogService / EventService |

Redfish只呈現BMC已掌握的狀態。BIOS尚未同步、Host未上電或資料過期時，需要正確表示unavailable、unknown或stale policy。

## 31.23 Target 端檢查

### 31.23.1 Host State 與 Boot Progress

```bash
busctl tree xyz.openbmc_project.State.Host 2>/dev/null
busctl tree xyz.openbmc_project.State.Chassis 2>/dev/null
busctl tree xyz.openbmc_project.State.Boot 2>/dev/null

journalctl -b --no-pager | \
    grep -Ei 'boot progress|postcode|post code|host state|bios|uefi'
```

### 31.23.2 BIOS Configuration

```bash
busctl tree xyz.openbmc_project.BIOSConfig.Manager 2>/dev/null
busctl introspect \
    xyz.openbmc_project.BIOSConfig.Manager \
    /xyz/openbmc_project/bios_config/manager 2>/dev/null

systemctl --type=service | grep -Ei 'bios|pldm|ipmi|host'
```

Service name與object path依image調整。

### 31.23.3 Host Inventory

```bash
busctl tree xyz.openbmc_project.Inventory.Manager 2>/dev/null
busctl tree xyz.openbmc_project.ObjectMapper | \
    grep -Ei 'processor|dimm|memory|pcie|bios'
```

### 31.23.4 Host Interface

Host端可保存：

```bash
dmidecode -t 0 -t 1 -t 4 -t 9 -t 17 -t 42
ipmitool -I open mc info
ipmitool -I open chassis status
```

指令可用性取決於Host OS、權限與driver。

## 31.24 雙端 Debug 方法

BIOS / BMC問題需要同時收集：

### BMC 端

- BMC image與kernel version。
- Host / chassis state。
- POST code history。
- Boot progress。
- IPMI / PLDM / eSPI logs。
- BIOS config manager state。
- Host inventory objects。
- Event logs。

### Host / BIOS 端

- BIOS version與build ID。
- Complete serial log。
- POST code table。
- SMBIOS dump。
- ACPI tables，若相關。
- UEFI variables / BootOrder / BootNext。
- Setup settings。
- Reset reason。

### Hardware

- Power / reset waveform。
- eSPI / LPC / KCS state。
- SPI flash ownership與WP。
- Port80 capture。
- CPLD registers。

先建立共同時間點，例如Host power button assertion、PLTRST release或特定POST code，才能對齊三方logs。

## 31.25 常見問題與判讀

| 現象 | 優先方向 | 第一輪檢查 |
|---|---|---|
| POST code完全沒有資料 | Host未執行 / decode path | Power、reset、eSPI / LPC、BIOS設定 |
| POST code停在固定值 | BIOS初始化階段 | Code table、serial log、硬體狀態 |
| Boot progress不更新 | Transport / mapping | BIOS producer、IPMI / PLDM、D-Bus service |
| Boot progress和POST code不一致 | Mapping table / boot cycle | BIOS version、code history、state reset |
| One-time boot沒有生效 | Pending / consumption contract | Redfish value、BIOS read、BootNext |
| One-time boot反覆生效 | Consume / clear流程 | BIOS ACK、BMC persistence、reset timing |
| BIOS setting PATCH成功但未套用 | Pending→apply flow | Settings resource、Host reboot、BIOS validation |
| Pending setting消失 | BIOS拒絕 / BMC清除 | Error messages、registry compatibility |
| BIOS update後attributes錯亂 | Registry / version mismatch | BIOS version、registry、pending migration |
| CPU / DIMM inventory缺少 | Host inventory transfer | SMBIOS、PLDM、Host power state |
| Host off仍顯示舊inventory | Stale policy | Last update time、source boot ID |
| BMC reboot造成Host reset | Hardware ownership | GPIO / CPLD defaults、watchdog、power control |
| Watchdog在OS前觸發 | Handoff timing | BIOS timeout、OS driver、boot progress |
| Redfish與BIOS Setup值不同 | Current / pending / source | BIOS table、D-Bus、Redfish settings |
| Secure Boot狀態錯誤 | Data source / boot result | UEFI state、keys、BMC mapping |
| Multi-host資料互相覆蓋 | Identity / path | Host index、channel、object path |

## 31.26 Debug Log 收集

以下BMC端腳本只收集一般狀態：

```bash
#!/bin/sh

OUT=/tmp/bios-bmc-debug
mkdir -p "$OUT"

cat /etc/os-release > "$OUT/os-release.txt" 2>&1
uname -a > "$OUT/uname.txt"
cat /proc/cmdline > "$OUT/proc-cmdline.txt"

dmesg -T > "$OUT/dmesg.txt"
journalctl -b --no-pager > "$OUT/journal.txt" 2>&1
journalctl -b -1 --no-pager > "$OUT/journal-previous.txt" 2>&1
systemctl --failed > "$OUT/systemctl-failed.txt" 2>&1
systemctl --type=service | grep -Ei 'bios|host|ipmi|pldm|postcode' \
    > "$OUT/services.txt" 2>&1

busctl tree xyz.openbmc_project.State.Host \
    > "$OUT/host-state.txt" 2>&1
busctl tree xyz.openbmc_project.State.Chassis \
    > "$OUT/chassis-state.txt" 2>&1
busctl tree xyz.openbmc_project.State.Boot \
    > "$OUT/boot-state.txt" 2>&1
busctl tree xyz.openbmc_project.BIOSConfig.Manager \
    > "$OUT/bios-config.txt" 2>&1
busctl tree xyz.openbmc_project.Inventory.Manager \
    > "$OUT/inventory.txt" 2>&1
busctl tree xyz.openbmc_project.Software.Version \
    > "$OUT/software-version.txt" 2>&1

journalctl -b --no-pager | grep -Ei \
    'bios|uefi|postcode|boot progress|watchdog|host inventory|secure boot' \
    > "$OUT/related-journal.txt" 2>&1

# BIOS passwords、Redfish tokens與security keys不納入通用log。

tar czf "/tmp/bios-bmc-debug-$(date +%Y%m%d-%H%M%S).tar.gz" \
    -C /tmp bios-bmc-debug
```

## 31.27 Bring-up 順序

1. 列出BIOS與BMC共同功能及interface contract。
2. 確認Host power、reset與eSPI / LPC / MCTP前置條件。
3. 驗證POST code raw capture與code table。
4. 驗證boot progress producer、transport與D-Bus mapping。
5. 驗證Boot Source、Mode、Once / Continuous與consume規則。
6. 建立BootOrder / BootNext / BootOption identity對照。
7. 同步BIOS attribute table、registry、current與pending values。
8. 驗證Host inventory、SMBIOS與stale policy。
9. 驗證Redfish Host Interface，若平台支援。
10. 驗證Watchdog的BIOS→OS handoff與reset reason。
11. 驗證firmware error、security state與event mapping。
12. 執行BIOS update並確認registry、inventory與measurements同步。
13. 測試BMC reboot、Host warm reset、cold boot、AC cycle與service restart。
14. Multi-host平台逐一驗證identity與資料隔離。
15. 保存BIOS / BMC版本、logs、tables、waveforms與測試結果。

## 31.28 平台實測紀錄表

| Feature | Producer | Transport | BMC Object / Service | Redfish / IPMI | Timing | Result |
|---|---|---|---|---|---|---|
| POST Code | BIOS | LPC / eSPI | [待填] | Log / OEM | POST | [待確認] |
| Boot Progress | BIOS | IPMI / PLDM | [待填] | ComputerSystem | POST | [待確認] |
| Boot Override | BMC | Redfish / IPMI | [待填] | ComputerSystem.Boot | Pre-boot | [待確認] |
| Boot Options | BIOS | [待填] | [待填] | BootOptions | Pre-boot | [待確認] |
| BIOS Attributes | BIOS / BMC | PLDM / OEM | [待填] | Bios / Settings | Reboot apply | [待確認] |
| Host Inventory | BIOS | SMBIOS / PLDM | [待填] | Processor / Memory / PCIe | POST | [待確認] |
| Watchdog | BIOS / OS | IPMI / OEM | [待填] | Watchdog / Log | Boot / runtime | [待確認] |
| Secure Boot | BIOS | [待填] | [待填] | SecureBoot | POST | [待確認] |
| BIOS Version | BIOS | SMBIOS / OEM | [待填] | BiosVersion | POST | [待確認] |
| Firmware Error | BIOS | PLDM / IPMI / CPER | [待填] | LogService / Event | Runtime | [待確認] |

## 31.29 驗收 Checklist

介面契約：

- [ ] 每項BIOS–BMC功能都有producer、consumer、transport、timing與version。
- [ ] Timeout、retry、persistence與error handling已定義。
- [ ] BMC與BIOS的firmware compatibility matrix已建立。

Boot與診斷：

- [ ] POST code capture、history、table revision與multi-host identity正確。
- [ ] Boot progress stages、timeout與boot cycle reset已驗證。
- [ ] Host serial、POST code、boot progress與power waveform可依時間對齊。
- [ ] Boot failure會保存最後狀態、reset reason與相關firmware logs。

Configuration：

- [ ] One-time / continuous boot override與consume規則已測試。
- [ ] BootOrder、BootNext與BootOptions identity對齊。
- [ ] BIOS current、pending與Attribute Registry一致。
- [ ] Invalid / unsupported attribute會回傳明確錯誤。
- [ ] BIOS update與downgrade時pending settings有migration policy。

Inventory與Security：

- [ ] CPU、DIMM、PCIe、BIOS version與SMBIOS來源已確認。
- [ ] Host off與transfer failure時的stale policy已驗證。
- [ ] Secure Boot、TPM與measurement狀態映射正確。
- [ ] BIOS password、credentials、keys與security material不會進入一般log。

Recovery：

- [ ] BIOS→OS watchdog handoff已驗證。
- [ ] BMC reboot不會造成非預期Host reset或watchdog expiration。
- [ ] Host warm reset、cold power cycle、AC cycle與BIOS recovery已測試。
- [ ] Multi-host資料與控制路徑互相隔離。
- [ ] D-Bus、Redfish、IPMI與BIOS實際狀態一致。

## 31.30 本章重點

1. BIOS / UEFI負責Host初始化；BMC負責平台管理與遠端控制，兩者透過明確介面契約協作。
2. POST code提供細緻的vendor-specific階段；boot progress提供較穩定的高階狀態。
3. Boot override需要定義pending、apply、consume與clear規則。
4. UEFI BootOrder、BootNext與Redfish BootOptions需要穩定identity對照。
5. BIOS configuration包含current attributes、pending settings與Attribute Registry。
6. Host inventory需要記錄來源、更新時間、BIOS version與stale policy。
7. Redfish Host Interface可讓UEFI / Host OS透過in-band HTTPS存取BMC Redfish service。
8. Watchdog需要完整的BIOS→OS owner handoff與expiration log。
9. BIOS update會影響registry、inventory、POST code table與security measurements。
10. BMC reboot、Host reset與多Host情境都需要驗證資料恢復與identity隔離。

## 31.31 本章參考資料

- DMTF Redfish Specification: https://www.dmtf.org/dsp/DSP0266
- DMTF Redfish Host Interface Specification DSP0270: https://www.dmtf.org/dsp/DSP0270
- DMTF Redfish BIOS schema and data model: https://redfish.dmtf.org/redfish/schema_index
- DMTF PLDM standards: https://www.dmtf.org/standards/pmci
- UEFI Specification - EFI Redfish Service Support: https://uefi.org/specs/UEFI/
- OpenBMC boot progress design: https://github.com/openbmc/docs/blob/master/designs/boot-progress.md
- OpenBMC remote BIOS configuration design: https://github.com/openbmc/docs/blob/master/designs/remote-bios-configuration.md
- OpenBMC documentation: https://github.com/openbmc/docs
