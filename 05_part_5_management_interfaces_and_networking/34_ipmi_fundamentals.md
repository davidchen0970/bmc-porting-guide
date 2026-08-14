# 21. IPMI 通用知識

IPMI（Intelligent Platform Management Interface）提供感測器、事件紀錄、FRU、電源控制、Watchdog、Serial over LAN 與 LAN 管理等功能。它可透過 Host 本機介面或管理網路傳送，最後由 BMC 的 IPMI service 將 command 映射到 OpenBMC D-Bus 與平台硬體。

本章從 IPMI message 結構開始，依序說明 Channel、NetFn、Command、Completion Code、Sensor、SDR、SEL、FRU、LAN、SOL、Watchdog、Chassis Control、User / Privilege 與 OEM command，最後整理 OpenBMC 整合、測試與安全要求。

## 21.1 IPMI 位於哪一層

```text
IPMI Client
ipmitool / Host Driver / Monitoring Software
        ↓
Transport
KCS / BT / SSIF / RMCP+ over LAN
        ↓
BMC IPMI Service
        ↓
Command Handler
        ↓
OpenBMC D-Bus Service
        ↓
Sensor / Inventory / Logging / Power Control
```

IPMI 定義 message、commands、completion codes、sensor model、event records 與管理行為。KCS、BT、SSIF 與 LAN 則負責傳送 IPMI message。

Linux 的 IPMI stack包含 message handler、system-interface或SMBus driver、userspace device interface與可選的Watchdog driver；標準Host介面可由ACPI或SMBIOS提供資訊。

## 21.2 Local IPMI 與 IPMI over LAN

### 21.2.1 Local IPMI

Host OS透過KCS、BT或SSIF連接BMC：

```bash
ipmitool -I open mc info
ipmitool -I open chassis status
```

`-I open`表示使用Linux本機IPMI device，例如`/dev/ipmi0`。

### 21.2.2 IPMI over LAN

遠端client透過UDP與BMC建立IPMI session：

```bash
ipmitool -I lanplus \
    -H <bmc-address> \
    -U <user> \
    -E \
    mc info
```

`-E`表示密碼從環境變數讀取。測試腳本不宜把密碼直接寫在command line，避免出現在shell history或process list。

### 21.2.3 兩條路徑的差異

| 項目 | Local | LAN |
|---|---|---|
| Transport | KCS / BT / SSIF | RMCP / RMCP+ |
| 使用者身分 | Host介面與channel policy | IPMI user / session |
| Network dependency | 無 | 有 |
| 常見用途 | BIOS、Host OS、Watchdog | 遠端管理與自動化 |
| 排查入口 | Host driver、I/O address、BMC transport | Network、session、cipher suite、user privilege |

## 21.3 IPMI Message

一筆IPMI request通常包含：

- Network Function（NetFn）。
- Logical Unit Number（LUN）。
- Command（Cmd）。
- Request data。
- Channel / session context。

Response通常包含：

- Completion Code。
- Response data。

```text
Request
NetFn + Cmd + Data
        ↓
Handler
        ↓
Response
Completion Code + Data
```

Transport header、checksum、sequence與session fields依KCS、BT、SSIF或LAN而異。

## 21.4 NetFn、Command 與 LUN

### 21.4.1 NetFn

NetFn將commands分組。常見類別包含：

| 類別 | 常見功能 |
|---|---|
| Chassis | Power、reset、boot options、chassis status |
| Sensor / Event | Sensor reading、event receiver |
| Application | Device ID、channel、Watchdog |
| Firmware | Firmware相關commands，依規格與實作 |
| Storage | FRU、SDR、SEL |
| Transport | LAN、SOL與channel configuration |
| Group Extension | 特定標準群組的擴充 |
| OEM | 廠商自訂commands |

實際NetFn數值與command定義應以IPMI specification及平台文件為準。

### 21.4.2 Command

Command在所屬NetFn內識別功能。相同Cmd值可以存在於不同NetFn，因此command identity至少需要：

```text
NetFn + Cmd
```

OEM command還需要IANA enterprise number或其他平台識別。

### 21.4.3 LUN

LUN用於IPMI message routing。多數一般commands使用BMC LUN 0，但sensor與bridge情境可能涉及其他LUN。實作與測試不能在未查規格前忽略LUN。

## 21.5 Completion Code

Completion Code表示command處理結果。`0x00`通常代表成功；其他值表示request、state、權限、resource或handler發生問題。

常見類型：

- Invalid command。
- Invalid data field。
- Request data length錯誤。
- Command不支援目前state。
- Insufficient privilege。
- Resource busy。
- Timeout。
- Unspecified error。
- OEM-defined error。

排查時應分開判讀：

```text
Transport Timeout
沒有收到IPMI response

Completion Code Error
已收到response，但handler拒絕或處理失敗
```

Completion Code需要在command contract中定義，不宜把所有backend錯誤都壓成同一個generic code。

## 21.6 Channel

IPMI Channel表示一條管理通訊路徑，例如：

- System interface channel。
- LAN channel。
- Serial / modem channel，舊式或特定平台。
- OEM channel。

每個channel可能具有：

- Medium type。
- Protocol type。
- Session support。
- User access。
- Privilege limit。
- Authentication policy。
- Messaging enable state。

```bash
ipmitool channel info <channel>
ipmitool channel getaccess <channel> <user-id>
```

Channel number由平台實作決定。不要假設LAN永遠是channel 1或Host interface永遠使用固定編號。

## 21.7 Privilege Level

常見privilege概念：

- Callback。
- User。
- Operator。
- Administrator。
- OEM。

Command handler應註冊最低必要privilege。Read-only sensor query與power control、user management、firmware control不應使用相同權限要求。

需要同時驗證：

- User的channel access。
- Channel privilege limit。
- Session requested privilege。
- Command handler privilege。
- OEM額外policy。

## 21.8 BMC Device ID

`Get Device ID`常作為最小連線測試：

```bash
ipmitool mc info
```

常見資料：

- Device ID與revision。
- Firmware revision。
- IPMI version。
- Manufacturer ID。
- Product ID。
- Additional device support。
- Auxiliary firmware revision。

OpenBMC `phosphor-host-ipmid`可由平台提供的`dev_id.json`設定Device ID相關欄位，資料通常由Yocto安裝並由service載入。

Manufacturer ID、Product ID與firmware version需要和產品release資料一致，不能只使用開發預設值。

## 21.9 Sensor

IPMI Sensor提供數值或離散狀態。常見類型：

- Temperature。
- Voltage。
- Current。
- Fan speed。
- Power supply。
- Presence。
- Failure / fault。
- Power state。
- Boot progress。
- Watchdog。

### 21.9.1 Sensor Number

每個IPMI sensor需要sensor number。它位於有限的編號空間，且需要在firmware更新後保持產品要求的穩定性。

### 21.9.2 Numeric Sensor

Raw reading需要依SDR中的線性化參數換算：

```text
Raw Reading
    ↓ M、B、exponents、linearization
Engineering Value
```

Scale、unit或coefficient錯誤時，IPMI與Redfish可能顯示不同數值。

### 21.9.3 Discrete Sensor

Discrete sensor使用state bits表示presence、fault、power state、boot progress等狀態。需定義：

- Assertion / deassertion meaning。
- Event masks。
- Current reading bits。
- Sensor availability。
- D-Bus property mapping。

### 21.9.4 Unavailable

Device未供電、Host off或bus暫時不可用時，應依SDR與產品policy表示reading unavailable，而不是回傳看似有效的0。

## 21.10 SDR

SDR（Sensor Data Record）描述sensor identity、type、unit、threshold、linearization、event masks與entity association。

```bash
ipmitool sdr list
ipmitool sdr elist all
ipmitool sensor list
```

### 21.10.1 SDR Repository

BMC可提供SDR repository，client透過reservation與record ID逐筆讀取。

需要確認：

- SDR count。
- Record IDs。
- Repository timestamp / generation。
- Reservation invalidation。
- Firmware更新後的穩定性。
- Multi-host / satellite sensors區隔。

### 21.10.2 D-Bus Sensor Filtering

OpenBMC可從D-Bus產生IPMI sensors；但IPMI SDR數量與編號空間有限，因此可能需要過濾不適合對外呈現的D-Bus sensors。`phosphor-host-ipmid`文件描述了dynamic sensors與`sensor_filter.json`類型的服務過濾方式。

### 21.10.3 Entity ID 與 Instance

SDR使用Entity ID與Instance將sensor關聯到system、chassis、processor、power supply、fan等元件。這些資料需要和FRU、inventory與Redfish association對齊。

## 21.11 Threshold 與 Hysteresis

Threshold sensor可能包含：

- Lower Non-Critical。
- Lower Critical。
- Lower Non-Recoverable。
- Upper Non-Critical。
- Upper Critical。
- Upper Non-Recoverable。

Hysteresis避免reading在threshold附近反覆assert / deassert。

需要確認：

- Threshold是否readable / settable。
- Raw與engineering value轉換。
- Assertion與deassertion thresholds。
- D-Bus threshold interface。
- Redfish threshold mapping。
- Event generation與deduplication。

## 21.12 SEL

SEL（System Event Log）保存平台事件。常見來源：

- Sensor threshold crossing。
- Power fault。
- Watchdog。
- Chassis intrusion。
- Fan / PSU fault。
- Firmware error。
- Boot failure。
- OEM event。

```bash
ipmitool sel info
ipmitool sel list
ipmitool sel elist
ipmitool sel get <record-id>
```

### 21.12.1 SEL Record

常見欄位：

- Record ID。
- Record type。
- Timestamp。
- Generator ID。
- Sensor type / number。
- Event direction / type。
- Event data bytes。

### 21.12.2 SEL 與 OpenBMC Logging

OpenBMC常由logging service保存事件，再由IPMI layer映射成SEL。需要定義：

- 哪些logging entries進入SEL。
- Sensor與inventory association。
- Severity mapping。
- Event data encoding。
- Duplicate suppression。
- Clear / delete behavior。
- Retention與容量。

### 21.12.3 Clear SEL

清除SEL會移除診斷證據。執行前應先匯出：

```bash
ipmitool sel elist > sel-before-clear.txt
```

量產工具應要求明確權限與audit log。

## 21.13 FRU

IPMI FRU Information Storage描述field-replaceable unit資料，例如：

- Chassis。
- Board。
- Product。
- MultiRecord areas。
- Manufacturer、part number、serial number。

```bash
ipmitool fru print
ipmitool fru read <fru-id> <output-file>
```

### 21.13.1 FRU Authority

同一欄位可能來自EEPROM FRU、D-Bus inventory、PLDM FRU、SMBIOS或static config。平台需指定權威來源與更新規則。

### 21.13.2 FRU Write

FRU write會修改nonvolatile data。需要：

- Backup。
- Field validation。
- Checksum update。
- Write protect policy。
- Device page / write-cycle handling。
- Verify readback。
- Manufacturing / field authorization。

通用debug流程不應自動寫入FRU。

## 21.14 Chassis Control

常見commands：

- Get Chassis Status。
- Chassis Power On。
- Chassis Power Off。
- Power Cycle。
- Hard Reset。
- Soft Shutdown，依平台支援。
- Identify。
- Boot Options。

```bash
ipmitool chassis status
ipmitool chassis power status
```

Power control會影響Host，測試前需確認目前工作負載、power-control state machine與recovery條件。

### 21.14.1 Soft 與 Hard Action

- Soft shutdown：請求OS正常關機，依Host agent / ACPI支援。
- Hard power off：直接關閉主要電源。
- Hard reset：保持供電但重置Host。
- Power cycle：關閉後依policy重新上電。

每個action需要明確timeout、fallback與event log。

## 21.15 Boot Options

IPMI Set System Boot Options可設定PXE、disk、safe mode、BIOS setup等下一次開機行為。

需要處理：

- Set-in-progress語意。
- Boot flags valid bit。
- Persistent或one-time。
- Boot device selector。
- BIOS read / consume timing。
- BMC reboot後保存。
- Redfish Boot override同步。

IPMI與Redfish共用同一boot authority，避免兩邊顯示不同pending state。

## 21.16 Watchdog

IPMI定義Watchdog Timer，可由BIOS、Host OS或管理程式設定。Linux IPMI driver也提供可選的IPMI Watchdog支援。

常見欄位：

- Timer use。
- Running state。
- Countdown。
- Pre-timeout interval。
- Pre-timeout action。
- Expiration action。
- Expiration flags。

```bash
ipmitool mc watchdog get
```

Watchdog測試需要確認：

- Owner。
- BIOS到OS的handoff。
- BMC reboot後狀態。
- Reset target。
- Last boot progress與reset reason。
- Pre-timeout interrupt / event。

## 21.17 LAN Configuration

IPMI LAN parameters可管理：

- IP source。
- IPv4 address。
- Netmask。
- Default gateway。
- MAC address。
- VLAN ID / priority。
- RMCP port與channel access。
- IPv6，依實作與規格支援。

```bash
ipmitool lan print <channel>
```

LAN設定應映射到OpenBMC network D-Bus，並與Redfish EthernetInterface保持一致。遠端修改IP可能中斷目前session，因此需要commit、reconnect與recovery流程。

## 21.18 RMCP+ Session 與 Cipher Suite

IPMI 2.0 LAN常使用RMCP+建立authenticated session。Session協商包含：

- Authentication algorithm。
- Integrity algorithm。
- Confidentiality algorithm。
- Requested privilege。
- User / channel policy。

安全要求：

- 停用不符合產品policy的cipher suites。
- 避免允許無authentication或無integrity的遠端管理組合。
- 限制administrator帳號與來源網路。
- 設定合理session timeout與maximum sessions。
- 監控authentication failure與brute-force行為。
- IPv4 / IPv6 firewall policy一致。

```bash
ipmitool channel getciphers ipmi <channel>
```

實際command支援取決於ipmitool版本與BMC implementation。

## 21.19 User 與 Channel Access

IPMI user資料通常包含：

- User ID。
- User name。
- Enabled state。
- Password / key material。
- Channel access。
- Privilege level。
- Session limit。

```bash
ipmitool user list <channel>
ipmitool channel getaccess <channel> <user-id>
```

需要驗證：

- 建立、停用、刪除user。
- Password policy。
- Channel-specific access。
- Failed login與lockout，依產品支援。
- Redfish AccountService同步。
- Factory reset與firmware update保存政策。

Password、session key與完整authentication material不可寫入一般log。

## 21.20 Serial over LAN

SOL將Host serial console封裝成IPMI LAN payload。

```text
Host UART
    ↓
BMC UART / UART Mux
    ↓
SOL Service
    ↓
RMCP+ Session
    ↓
Remote Console Client
```

需要確認：

- Host UART與BMC local console區隔。
- Baud rate與character format。
- UART mux owner。
- SOL payload enable。
- Privilege。
- Session cleanup。
- Escape sequence。
- Host / BMC reboot後恢復。

```bash
ipmitool -I lanplus -H <bmc> -U <user> -E sol info
ipmitool -I lanplus -H <bmc> -U <user> -E sol activate
```

SOL log可能包含BIOS、kernel、credentials prompt與客戶資料，保存與分享需依安全規則處理。

## 21.21 Bridging 與 Satellite Controller

IPMI可透過bridge將message送到其他management controller。此類路徑可能使用IPMB、I2C或平台自訂transport。

需要記錄：

- Transit channel。
- Target address。
- Target LUN。
- Request / response tracking。
- Timeout與retry。
- Bus ownership。
- Satellite power state。
- Event source identity。

Bridge timeout不宜直接判定BMC command handler故障；需逐段確認本機handler、bridge、physical bus與target controller。

## 21.22 OEM Command 設計

新增OEM command前，先確認標準IPMI、Redfish、PLDM或既有D-Bus interface能否表達需求。OEM command會增加client綁定、版本管理、安全審查與長期維護成本。

### 21.22.1 必填契約

| 欄位 | 說明 |
|---|---|
| IANA Enterprise Number | 識別OEM namespace |
| NetFn / Cmd | Command identity |
| Request | Byte order、length、ranges、reserved bits |
| Response | Data layout與version |
| Completion Codes | 每種錯誤的明確回應 |
| Privilege | 最低必要權限 |
| Channel | Host / LAN /特定channel限制 |
| State Dependency | Host power、update、manufacturing mode等 |
| Side Effect | Reset、clear、write、power control |
| Timeout / Retry | Handler與backend要求 |
| Versioning | 向後相容、feature discovery |
| Audit | 是否產生security / operation log |
| Test Cases | Positive、negative、boundary、concurrency |

### 21.22.2 Request Validation

Handler需要驗證：

- Exact或允許的request length。
- Reserved bits為0。
- Enum與range。
- Object existence。
- Current platform state。
- Privilege。
- Concurrent operation。
- Endianness與alignment。

### 21.22.3 Versioning

可在request / response加入明確version或capability command。既有欄位的語意不能在未改版的情況下改變。

### 21.22.4 Security

高風險OEM command包括：

- Raw register access。
- Arbitrary file read / write。
- Shell execution。
- Firmware / flash write。
- Password / key export。
- Debug unlock。
- Power / reset bypass。

這些功能應改用受控service、簽章更新、debug authorization或manufacturing-only流程，不宜直接暴露為一般IPMI command。

## 21.23 OpenBMC IPMI 架構

`phosphor-host-ipmid`是OpenBMC中處理Host endpoint IPMI commands的D-Bus based daemon，專案包含standard handlers、transport、user / channel邏輯與OEM providers。

```text
KCS / BT / SSIF / LAN Transport
        ↓
IPMI Request Parser
        ↓
Command Filter與Privilege Check
        ↓
NetFn / Cmd Handler Dispatch
        ↓
D-Bus Method / Property
        ↓
OpenBMC Backend Service
        ↓
Completion Code與Response Data
```

### 21.23.1 Provider

OpenBMC可透過provider library註冊command handler。每個handler需要指定priority、NetFn / Cmd、privilege與callback。OEM extension也可透過provider加入，但應遵循IANA與命名規則。

### 21.23.2 D-Bus Mapping

| IPMI 功能 | 常見OpenBMC資料來源 |
|---|---|
| Sensor / SDR | D-Bus sensor與inventory |
| SEL | Logging service |
| FRU | Inventory / FRU device |
| Chassis Control | Host / chassis state service |
| Boot Options | Boot control與BIOS service |
| LAN Config | Network service |
| User / Channel | User manager與channel config |
| Watchdog | Watchdog service |
| Device ID | Platform config / software version |

Backend service timeout、object missing與invalid state需要映射成合適completion code。

## 21.24 IPMI 與 Redfish 的資料一致性

IPMI與Redfish常使用相同D-Bus資料，但資料模型能力不同。

需要對照：

- Sensor value、unit與threshold。
- Inventory identity。
- Power state。
- Boot override。
- Network settings。
- User / role。
- Firmware version。
- Event / log。

IPMI SDR數量有限，Redfish可以呈現更多resources；兩者resource數量不同可以是設計結果，但同一元件的值、presence與health不應互相矛盾。

## 21.25 測試策略

### 21.25.1 基本功能

```bash
ipmitool mc info
ipmitool chassis status
ipmitool sensor list
ipmitool sdr elist all
ipmitool sel info
ipmitool fru print
```

### 21.25.2 Transport Matrix

- Host local KCS / BT / SSIF。
- IPv4 LAN。
- IPv6 LAN，若支援。
- Dedicated NIC。
- NC-SI。
- Administrator / Operator / ReadOnly users。

### 21.25.3 Negative Tests

- Invalid NetFn / Cmd。
- Request太短或太長。
- Invalid enum / reserved bits。
- Privilege不足。
- Host state不允許。
- Backend object missing。
- Backend timeout。
- Concurrent request。
- Session limit。
- Wrong password / disabled user。

### 21.25.4 Recovery Tests

- IPMI service restart。
- BMC reboot。
- Host reboot。
- Network link flap。
- NC-SI channel reset。
- KCS channel stalled。
- SEL / SDR reservation失效。
- User / network設定更新。

## 21.26 常見問題與判讀

| 現象 | 優先方向 | 第一輪檢查 |
|---|---|---|
| Local IPMI沒有`/dev/ipmi0` | Host driver / firmware table | ACPI / SMBIOS、`dmesg`、modules |
| KCS command timeout | Transport / BMC service | KCS state、service journal、Host address |
| LAN ping通但IPMI timeout | UDP / service / firewall | Port、channel、cipher、journal |
| Login失敗 | User / channel / cipher | User enabled、channel access、privilege |
| `mc info`資料錯 | Device ID config | JSON、software version、service restart |
| Sensor值錯 | SDR conversion / D-Bus | Raw、M / B、unit、threshold |
| Sensor不在SDR | Filter / numbering | Dynamic sensor config、SDR count |
| SEL沒有事件 | Logging mapping | Event source、association、SEL filter |
| SEL事件重複 | Deduplication / retry | Logging entries、event path、timestamp |
| FRU欄位錯 | Authority / parsing | EEPROM、inventory、checksum |
| Power action失敗 | State machine / privilege | Host state、power service、completion code |
| One-time boot反覆生效 | Consume / clear流程 | Boot flags、BIOS ACK、persistence |
| SOL無輸出 | UART / payload / privilege | UART mux、baud、SOL state |
| OEM command回generic error | Handler mapping | Backend error、validation、completion codes |
| Redfish與IPMI值不同 | D-Bus mapping | Data source、scale、association |

## 21.27 Debug Log 收集

```bash
#!/bin/sh

OUT=/tmp/ipmi-debug
mkdir -p "$OUT"

cat /etc/os-release > "$OUT/os-release.txt" 2>&1
uname -a > "$OUT/uname.txt"
cat /proc/cmdline > "$OUT/proc-cmdline.txt"

dmesg -T > "$OUT/dmesg.txt"
journalctl -b --no-pager > "$OUT/journal.txt" 2>&1
systemctl --failed > "$OUT/systemctl-failed.txt" 2>&1
systemctl --type=service | grep -Ei 'ipmi|kcs|ssif|sol|watchdog' \
    > "$OUT/services.txt" 2>&1

ls -l /dev/ipmi* /dev/kcs* > "$OUT/ipmi-devices.txt" 2>&1
journalctl -b --no-pager | grep -Ei \
    'ipmi|kcs|bt|ssif|sdr|sel|fru|sol|watchdog' \
    > "$OUT/ipmi-related-journal.txt" 2>&1

busctl tree xyz.openbmc_project.ObjectMapper \
    > "$OUT/objectmapper.txt" 2>&1
busctl tree xyz.openbmc_project.Sensor \
    > "$OUT/sensors.txt" 2>&1
busctl tree xyz.openbmc_project.Logging \
    > "$OUT/logging.txt" 2>&1
busctl tree xyz.openbmc_project.Inventory.Manager \
    > "$OUT/inventory.txt" 2>&1
busctl tree xyz.openbmc_project.State.Host \
    > "$OUT/host-state.txt" 2>&1

# 不自動執行power、reset、SEL clear、FRU write或OEM control command。
# 不收集password、session key與未遮蔽的user database。

tar czf "/tmp/ipmi-debug-$(date +%Y%m%d-%H%M%S).tar.gz" \
    -C /tmp ipmi-debug
```

## 21.28 Bring-up 順序

1. 確認平台支援的Local與LAN transports。
2. 驗證BMC Device ID與firmware information。
3. 建立channel、medium、session與privilege對照。
4. 驗證Host本機`/dev/ipmi0`與基本commands。
5. 驗證LAN session、users、cipher suites與firewall。
6. 建立Sensor、SDR number、entity與D-Bus對照。
7. 驗證numeric conversion、threshold與unavailable behavior。
8. 驗證SEL event、record、clear與retention。
9. 驗證FRU authority、read與受控write流程。
10. 驗證Chassis Control、Boot Options與Power state。
11. 驗證Watchdog owner、handoff與reset reason。
12. 驗證LAN configuration與Redfish network同步。
13. 驗證SOL、session cleanup與UART mapping。
14. 審查OEM commands的必要性、權限與versioning。
15. 執行negative、concurrency、restart與recovery tests。
16. 比對IPMI、D-Bus與Redfish輸出。
17. 保存commands、responses、completion codes、logs與版本。

## 21.29 平台實測紀錄表

| 項目 | 來源 / 指令 | 實測值 | 備註 |
|---|---|---|---|
| IPMI Version | `mc info` | [待填] | Device / firmware |
| Host Interface | Host `dmesg` | [待填] | KCS / BT / SSIF |
| LAN Channel | `channel info` | [待填] | Dedicated / NC-SI |
| Cipher Suites | `channel getciphers` | [待填] | Allowed list |
| Users / Roles | User / channel access | [待填] | Security review |
| SDR Count | `sdr info` / list | [待填] | Stable numbering |
| Sensor Mapping | SDR ↔ D-Bus | [待填] | Unit / scale |
| SEL | `sel info` | [待填] | Capacity / retention |
| FRU | `fru print` | [待填] | Authority |
| Chassis Control | Power tests | [待填] | Soft / hard behavior |
| Boot Options | BIOS / BMC test | [待填] | Once / persistent |
| Watchdog | Watchdog test | [待填] | Owner / action |
| SOL | SOL session | [待填] | UART / privilege |
| OEM Commands | Contract list | [待填] | IANA / version |
| Redfish Consistency | Cross-check | [待填] | Sensor / power / LAN |

## 21.30 驗收 Checklist

Protocol與Transport：

- [ ] NetFn、Cmd、LUN、Channel與Completion Code能正確判讀。
- [ ] KCS / BT / SSIF與LAN transports已依產品支援測試。
- [ ] Transport timeout與completion code error可分開定位。
- [ ] Service restart、BMC reboot與Host reboot後可恢復。

Sensor、SEL與FRU：

- [ ] Sensor number、SDR、entity、unit、scale與threshold正確。
- [ ] Unavailable與Host-power dependency不會被誤報為有效0值。
- [ ] SEL event、severity、association、deduplication與retention已驗證。
- [ ] SEL clear具有權限與事前匯出流程。
- [ ] FRU權威來源、checksum、read與受控write已驗證。

Control與Configuration：

- [ ] Chassis power、reset、cycle與identify符合power policy。
- [ ] Boot options與Redfish / BIOS current、pending、consume流程一致。
- [ ] LAN configuration由單一network authority管理。
- [ ] Watchdog owner、BIOS→OS handoff與reset target已驗證。
- [ ] SOL的UART、privilege、session與log policy正確。

Security與OEM：

- [ ] 不安全cipher suites已停用或具有書面例外。
- [ ] User、channel access、privilege與session limit已驗證。
- [ ] Password、keys、tokens與SOL敏感內容不進入一般log。
- [ ] OEM command具有IANA、request / response、completion codes、privilege與version契約。
- [ ] OEM command完成negative、boundary、concurrency與state tests。
- [ ] IPMI、D-Bus與Redfish呈現相同平台狀態。

## 21.31 本章重點

1. IPMI定義管理message與資料模型；KCS、BT、SSIF與LAN負責傳送message。
2. Command identity由NetFn與Cmd組成，OEM command還需要企業識別與版本契約。
3. Completion Code表示handler結果；完全timeout優先檢查transport與service。
4. Channel同時影響medium、session、user access與privilege。
5. SDR描述sensor語意，reading的unit、scale、threshold與entity都來自SDR mapping。
6. SEL保存事件，清除前應先匯出診斷資料。
7. FRU write、power control、boot options與OEM control都具有平台副作用。
8. RMCP+安全性取決於user、channel、privilege、cipher suite與網路隔離。
9. OpenBMC IPMI handler通常將commands轉成D-Bus calls，backend錯誤需映射成合適completion code。
10. IPMI與Redfish可以呈現不同數量的resources，但同一元件的狀態需要一致。

## 21.32 本章參考資料

- IPMI Specifications: https://www.intel.com/content/www/us/en/products/docs/servers/ipmi/ipmi-home.html
- Linux kernel documentation - The Linux IPMI Driver: https://docs.kernel.org/driver-api/ipmi.html
- OpenBMC phosphor-host-ipmid: https://github.com/openbmc/phosphor-host-ipmid
- phosphor-host-ipmid configuration: https://github.com/openbmc/phosphor-host-ipmid/blob/master/docs/configuration.md
- OpenBMC documentation: https://github.com/openbmc/docs
- OpenIPMI: https://openipmi.sourceforge.io/
