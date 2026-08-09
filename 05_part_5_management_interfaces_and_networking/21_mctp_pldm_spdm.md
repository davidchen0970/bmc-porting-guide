# 20. MCTP、PLDM 與 SPDM

MCTP 提供 BMC 與平台元件之間的管理訊息傳輸；PLDM 在 MCTP 上定義 inventory、sensor、control、BIOS 與 firmware update 等管理功能；SPDM 則提供裝置身分驗證、憑證、量測與 secure session。

本章從 MCTP endpoint 與 EID 開始，接著說明 transport binding、routing、PLDM terminus、PDR、FRU、firmware update，以及 SPDM attestation 如何整合到 OpenBMC。

## 20.1 三個 Protocol 的關係

BMC 可能需要管理 NIC、GPU、retimer、CXL device、satellite controller 與其他管理元件。這些裝置使用的實體連接可能不同，但上層希望使用一致的管理方式。

```text
SMBus / I2C、PCIe VDM、I3C、USB、Serial
        ↓
MCTP
Endpoint 定址、訊息類型、分段與 routing
        ↓
PLDM
Inventory、Sensor、Control、BIOS、Firmware Update

SPDM
裝置認證、憑證、量測與 Secure Session
        ↓
OpenBMC D-Bus、Redfish、Event 與 Update Service
```

### 20.1.1 MCTP

MCTP（Management Component Transport Protocol）讓管理訊息可以在不同 transport bindings 上傳送。上層 protocol 使用 endpoint 與 EID 通訊，不需要直接處理 I2C address、PCIe BDF 或 I3C dynamic address。

### 20.1.2 PLDM

PLDM（Platform Level Data Model）定義平台管理 commands 與資料結構。常見功能包括：

- Protocol capability discovery。
- Sensors 與 effecters。
- Platform Descriptor Records。
- FRU inventory。
- BIOS attributes。
- Device firmware update。

### 20.1.3 SPDM

SPDM（Security Protocol and Data Model）用來確認 endpoint 身分與軟體狀態。它可以協商安全演算法、驗證 certificate chain、執行 challenge、取得 measurements，並建立 secure session。

## 20.2 MCTP Endpoint 與 EID

Endpoint 是支援 MCTP 的管理通訊端點，例如 BMC、NIC、GPU、retimer 或 satellite controller。

每個 endpoint 需要可供 MCTP network 使用的 EID（Endpoint ID）。EID 是 MCTP network 內的 logical address。

```text
BMC EID 8
NIC EID 9
Retimer EID 10
GPU EID 11
```

### 20.2.1 EID 的用途

MCTP request 以 destination EID 指定接收端。Routing layer 再依 EID 找到對應 link 或 next hop。

```text
PLDM Request to EID 9
        ↓
MCTP route lookup
        ↓
PCIe VDM link
        ↓
NIC endpoint
```

### 20.2.2 Static 與 Dynamic EID

- Static EID：平台預先指定，開機後使用固定值。
- Dynamic EID：由 bus owner discovery 後分配。
- Pre-assigned EID：endpoint 已保存或由其他 firmware 事先設定。

不論採用哪種方式，都需定義 endpoint reset、BMC reboot、hot-plug 與 multi-owner 情況下的重新分配規則。

### 20.2.3 Endpoint UUID

EID 可能重新分配；Endpoint UUID 用來辨識較穩定的 endpoint identity。Inventory 與 policy 不宜只以 EID 當永久身分。

### 20.2.4 Network ID

Linux 可同時管理多個 MCTP networks。Network ID 用來區分不同 routing domains。相同 EID 在不同 networks 中可代表不同 endpoints，因此診斷紀錄應同時保存 network ID 與 EID。

## 20.3 Bus Owner 與 Discovery

Bus owner 負責在某個 MCTP binding 上管理 discovery 與 EID assignment。平台需明確指定 BMC、Host 或 bridge 中的哪一方擔任 owner。

Discovery 通常包含：

```text
確認 Physical Link
        ↓
找到 MCTP Endpoint
        ↓
取得或設定 EID
        ↓
取得 Endpoint UUID
        ↓
查詢支援的 Message Types
        ↓
建立 Route
        ↓
通知 PLDM / SPDM Services
```

### 20.3.1 多個 Owners

若 Host 與 BMC 同時分配 EID，可能造成：

- EID 重複。
- Endpoint 每次開機使用不同 EID。
- Existing route失效。
- PLDM terminus 重複建立。
- Hot-plug 後 endpoint 無法重新加入。

平台文件應記錄 owner、EID range、allocation timing 與 persistence policy。

### 20.3.2 Endpoint 消失

Endpoint 被拔除、斷電或 reset 時，MCTP stack 應更新 route 與 endpoint state。上層 PLDM sensors、inventory 與 SPDM status 也需同步更新。

## 20.4 MCTP Message

MCTP packet 包含 transport header 與 payload。Transport header 用來傳遞 source / destination EID、message tag、sequence 與分段狀態；payload 開頭則指出 message type。

常見 message types：

| Message Type | 用途 |
|---|---|
| MCTP Control | Endpoint discovery、EID、UUID、message type support |
| PLDM | Platform management |
| SPDM | Security negotiation 與 attestation |
| NC-SI over MCTP | Network controller management |
| Vendor Defined | 廠商自訂管理訊息 |

### 20.4.1 Message Tag

Message tag 用來配對 request 與 response。Sender 在同一 endpoint 上同時發出多筆 requests 時，需要避免 tag 重複使用造成 response 配錯。

### 20.4.2 Fragmentation 與 Reassembly

一筆上層 message 可能大於 binding 單一 packet 可承載的大小。MCTP 會將它拆成多個 packets，接收端再依 sequence、SOM / EOM 與 tag 重組。

大型 PLDM firmware update payload、PDR、FRU record 與 SPDM certificate chain 都會受 path MTU 影響。

### 20.4.3 Path MTU

Path MTU 是一條 MCTP path 可使用的 packet size。它受 binding、bridge 與 endpoint capability 影響。MTU 設定錯誤可能造成短 command 正常，但大型 response timeout 或無法重組。

## 20.5 MCTP Transport Bindings

Transport binding 定義 MCTP 如何在特定實體介面上傳送。

### 20.5.1 MCTP over SMBus / I2C

常見於 retimer、OCP NIC sideband、satellite controller 與其他低速管理裝置。

需記錄：

- Root adapter 與 mux path。
- Endpoint SMBus address。
- Bus owner。
- MCTP EID。
- Bus speed、pull-up 與 power domain。
- Endpoint reset / presence。
- Binding driver 與 service owner。

此 binding 的效能受到 SMBus speed 與 block size 限制，大型 message 會產生較多 fragments。

### 20.5.2 MCTP over PCIe VDM

PCIe Vendor Defined Message binding 常用於 NIC、GPU、accelerator 與 CXL / PCIe endpoints。

需確認：

- PCIe device 已完成 enumeration。
- Link state 與 target BDF。
- Endpoint 是否支援 MCTP VDM。
- Host power state 需求。
- Hot-plug 與 Function Level Reset 後的 discovery。
- BMC 與 Host 對 VDM path 的 ownership。

### 20.5.3 MCTP over I3C

I3C binding 可利用 dynamic addressing 與較高 throughput。Bring-up 需要先確認：

- I3C controller 與 target support。
- Dynamic address assignment。
- Endpoint discovery timing。
- In-Band Interrupt，若平台使用。
- Kernel binding driver 與工具支援。

### 20.5.4 Bridge

MCTP bridge 連接兩個或更多 MCTP segments。Bridge 需管理 next-hop route、EID visibility、MTU 與 endpoint changes。

```text
BMC MCTP Network
        ↓
MCTP Bridge
   ├── SMBus Segment
   └── PCIe VDM Segment
```

排查 bridge path 時需保存完整路徑，而非只記錄最終 EID。

## 20.6 Linux 與 OpenBMC MCTP Stack

Linux kernel MCTP stack 可提供：

- MCTP links。
- Local addresses。
- Routes。
- Neighbor information，依 binding 與工具支援。
- `AF_MCTP` socket。

OpenBMC 專案可能使用 kernel MCTP、`mctpd`、其他 userspace daemon 或 vendor stack。實際架構需以目前 branch、recipes 與 systemd services 為準。

### 20.6.1 基本檢查

```bash
zcat /proc/config.gz | grep CONFIG_MCTP

dmesg | grep -Ei 'mctp|vdm|i3c'

mctp link
mctp addr
mctp route
```

工具輸出格式會依 `mctp` userspace tool 與 kernel version 不同。

### 20.6.2 D-Bus Endpoint

若平台使用 MCTP daemon 發布 endpoints，可從 D-Bus 確認：

- EID。
- UUID。
- Network ID。
- Binding / physical path。
- Supported message types。
- Connectivity state。

```bash
busctl tree xyz.openbmc_project.MCTP 2>/dev/null
busctl tree xyz.openbmc_project.ObjectMapper | grep -i mctp
```

Service name 與 object path 依專案整合而異。

## 20.7 MCTP Bring-up

MCTP bring-up 應由底層往上驗證。

```text
Endpoint 已供電
        ↓
Physical Binding 可用
        ↓
Binding Driver 建立 Link
        ↓
Bus Owner 發現 Endpoint
        ↓
EID / UUID / Message Types 可取得
        ↓
Route 建立
        ↓
Control Request / Response 正常
        ↓
PLDM / SPDM 開始運作
```

### 20.7.1 最小驗證資料

| 項目 | 實測值 |
|---|---|
| Endpoint name | [待填] |
| Binding | SMBus / PCIe VDM / I3C / Other |
| Physical path | [待填] |
| Network ID | [待填] |
| EID | [待填] |
| UUID | [待填] |
| Message types | [待填] |
| Path MTU | [待填] |
| Bus owner | [待填] |
| Route | [待填] |

## 20.8 PLDM Terminus 與 TID

PLDM communication peer 稱為 terminus。一個 MCTP endpoint 可以提供一個或多個 PLDM termini，實際模型取決於 endpoint firmware 與 PLDM implementation。

TID（Terminus ID）用來識別 PLDM terminus。系統需要保存 MCTP endpoint、EID 與 TID 的對照。

```text
MCTP Endpoint
Network 1 / EID 9 / UUID X
        ↓
PLDM Terminus
TID 3
        ↓
PDR、Sensors、Effecters、FRU、FW Components
```

Endpoint reset 或 rediscovery 後，PLDM service 需要清除 stale terminus，再重新建立 PDR 與 sensor mapping。

## 20.9 PLDM Base Discovery

PLDM Type 0 Base 用來查詢 protocol capability。常見 commands 包括：

- GetTID。
- SetTID，依角色與流程使用。
- GetPLDMVersion。
- GetPLDMTypes。
- GetPLDMCommands。

Bring-up 順序：

```text
MCTP Route Ready
        ↓
確認 Endpoint 支援 PLDM Message Type
        ↓
Get PLDM Types
        ↓
Get PLDM Version
        ↓
Get Commands for each Type
        ↓
建立 Terminus Capability
```

`GetPLDMTypes` 成功只代表 endpoint 宣告支援哪些 types；每個 type 的版本與 command set 仍需另外查詢。

## 20.10 PLDM Types

| Type | 功能 |
|---:|---|
| 0 Base | Version、Type 與 Command discovery |
| 1 SMBIOS | SMBIOS table transfer |
| 2 Platform | PDR、Sensor、Effecter、Event |
| 3 BIOS | BIOS attributes 與設定 |
| 4 FRU | FRU record data |
| 5 Firmware Update | Device firmware update |
| OEM | 廠商擴充 |

Endpoint 不必支援全部 types。BMC 應依 discovery 結果建立對應功能，不應直接假設每個 endpoint 都有 sensor、FRU 或 firmware update。

## 20.11 PDR 是什麼

PDR（Platform Descriptor Record）描述 PLDM platform 的 entities、sensors、effecters 與 associations。BMC 先取得 PDR repository，才知道 endpoint 提供哪些管理項目以及如何解讀資料。

### 20.11.1 Numeric Sensor PDR

描述連續數值，例如：

- Temperature。
- Voltage。
- Current。
- Power。
- Fan speed。

PDR 會提供 unit、resolution、offset、range、threshold 等資訊。Scale 或 unit 解讀錯誤會直接造成 D-Bus 與 Redfish 數值錯誤。

### 20.11.2 State Sensor PDR

描述離散狀態，例如：

- Present / absent。
- Link state。
- Fault state。
- Device enabled state。
- Firmware update state。

每個 state set 的數值需映射到清楚的 D-Bus property 或 event。

### 20.11.3 Effecter PDR

Effecter 是 BMC 可設定的 remote control。

- Numeric effecter：power limit、fan target 等數值。
- State effecter：reset、enable、mode 等狀態。

Effecter 可能影響 power、reset 或安全狀態，需要權限、policy、range checking 與 audit log。

### 20.11.4 Entity Association PDR

Entity Association PDR 描述 endpoint 中 entities 的父子關係，例如：

```text
NIC
├── Port 0
├── Port 1
└── Temperature Sensor
```

OpenBMC 可將這些關係轉成 inventory associations，讓 sensor 與 Redfish resource 歸屬到正確元件。

### 20.11.5 PDR Repository Change

Endpoint firmware 更新或 configuration 改變後，PDR repository 可能變更。BMC 收到 change event 時，需要重新取得受影響的 records，更新 sensors、effecters 與 associations。

## 20.12 PLDM Sensor 與 Event

BMC 可以 polling PLDM sensor，也可以讓 endpoint 主動送出 Platform Event Message。

### 20.12.1 Polling

Polling 流程較直接，但會增加 MCTP traffic。Polling period 應依 sensor 重要性、link throughput 與 endpoint能力設定。

### 20.12.2 Event

Event flow 通常需要：

- 設定 event receiver。
- 啟用 endpoint event generation。
- 處理 event sequence與 acknowledgement。
- 更新 D-Bus sensor / inventory state。
- 產生必要的 logging entry。
- 去除重複 event。

Endpoint retry 與 BMC service restart 後，需避免同一事件被重複記錄多次。

## 20.13 PLDM FRU

PLDM FRU 提供 FRU Record Set。BMC 可將 record fields 映射到 OpenBMC inventory，例如：

- Manufacturer。
- Model。
- PartNumber。
- SerialNumber。
- Version。

需要先定義資料權威端。若同一欄位也存在 IPMI FRU、PMBus MFR command、SMBIOS 或 static config，必須設定 priority 與 conflict policy。

Endpoint 自身 FRU 與 endpoint 管理的 downstream FRU 也需要建立不同 inventory identity。

## 20.14 PLDM BIOS

PLDM BIOS 提供 BIOS attributes 與 configuration exchange。常見資料包括：

- Current value。
- Pending value。
- Default value。
- Allowed values / range。
- Read-only / read-write attribute。

設定變更可能在下一次 Host reboot 或特定 firmware stage 才生效。OpenBMC 需要顯示 pending 與 current 的差異，並記錄修改者與時間。

## 20.15 PLDM Firmware Update

PLDM Firmware Update 定義 BMC 與 device 的更新流程。

```text
Discover Device Identity / Components
        ↓
確認 Package 與 Device 相容
        ↓
Request Update
        ↓
Pass Component Table
        ↓
Transfer Firmware Data
        ↓
Verify
        ↓
Apply / Activate
        ↓
Confirm New Version
```

### 20.15.1 Update 前

確認：

- Endpoint identity。
- Component classification / identifier。
- Current version。
- Package version。
- Transfer size。
- Update option flags。
- Activation method。
- Host / device power state。
- Signature 與產品政策。

### 20.15.2 Firmware Data Transfer

PLDM update 的 data request size 會受 endpoint buffer、MCTP path MTU 與 implementation limits 影響。Transfer timeout、retry 與 duplicate block handling 必須驗證。

### 20.15.3 Activation

Activation 可能需要：

- Immediate apply。
- Endpoint reset。
- Host reset。
- AC cycle。
- Manual activation。

OpenBMC software inventory、Redfish UpdateService 與 event log 應呈現 progress、activation state 與失敗原因。

### 20.15.4 中斷與 Recovery

測試：

- MCTP link中斷。
- Endpoint reset。
- BMC service restart。
- BMC reboot。
- Host power transition。
- Transfer timeout。
- Verify failure。
- Activation failure。

需確認舊 firmware 能繼續運作，或 endpoint 有 recovery image / external update path。

## 20.16 SPDM Roles 與流程

SPDM 通訊包含 requester 與 responder。BMC 常作為 requester，remote device 作為 responder。

典型流程：

```text
GET_VERSION
        ↓
GET_CAPABILITIES
        ↓
NEGOTIATE_ALGORITHMS
        ↓
GET_DIGESTS
        ↓
GET_CERTIFICATE
        ↓
CHALLENGE
        ↓
GET_MEASUREMENTS
        ↓
可選：KEY_EXCHANGE / FINISH
        ↓
Secure Session
```

每一階段都會使用前面協商的版本、capability 與 algorithms。前一步失敗時，後續認證或 session 不會成立。

## 20.17 SPDM Version、Capabilities 與 Algorithms

### 20.17.1 Version

Requester 與 responder 選擇雙方都支援的 SPDM version。平台 security policy 可以設定最低可接受版本。

### 20.17.2 Capabilities

Capabilities 可能包括：

- Certificate support。
- Challenge support。
- Measurement support。
- Key exchange。
- PSK exchange。
- Heartbeat。
- Key update。
- Encrypted / MAC protected message。

後續流程必須依協商結果進行。

### 20.17.3 Algorithms

常見協商項目：

- Base hash。
- Base asymmetric signature。
- DHE group。
- AEAD cipher。
- Key schedule。
- Measurement hash。

平台應明確定義允許清單與最低安全要求，避免 endpoint 自動選用已不符合產品政策的演算法。

## 20.18 Certificate 與 Challenge

Endpoint 可提供一個或多個 certificate slots。Requester 先取得 digest，再分段讀取 certificate chain，最後驗證：

- Chain encoding。
- Root / intermediate / leaf certificate。
- Signature。
- Validity time，若 policy 使用時間驗證。
- Device identity。
- Trust anchor。
- Revocation policy，若平台支援。

Challenge 由 requester 提供 nonce，responder 使用 device private key 簽署 transcript。驗證成功表示 responder 持有與 certificate 對應的 private key。

Private key 與 session secrets 不應出現在一般 log。

## 20.19 SPDM Measurements

Measurements 用來取得 endpoint firmware、configuration 或其他 TCB components 的 cryptographic evidence。

每個 measurement block 可能包含：

- Index。
- Measurement type。
- Raw bit stream 或 digest。
- Measurement hash algorithm。
- Signature，依 request attributes與能力而定。

验证 measurement 需要 expected values、signed manifest 或其他 reference data。只取得 digest，尚不足以判斷 endpoint 是否可信。

### 20.19.1 Firmware Update 與 Measurement

Device 更新後 measurement 可能改變。Firmware package、release process 與 attestation database 需要同步更新 expected values，並保留版本對照。

## 20.20 Secure Session

若雙方支援，SPDM 可透過 key exchange 或 PSK 建立 secure session。Secured message 可提供：

- Integrity protection。
- Peer authentication。
- Replay protection。
- Confidentiality，依協商結果。

需要管理：

- Session ID。
- Sequence number。
- Heartbeat。
- Key update。
- Session teardown。
- Endpoint reset後的 session cleanup。
- Concurrent sessions。

Secure session 增加 header 與 cryptographic overhead，也會影響 message size、MTU 與 timeout。

## 20.21 SPDM Policy 與 OpenBMC

Attestation 結果需要被平台 policy 使用。

| 結果 | 可能處理 |
|---|---|
| Endpoint 不支援要求版本 | Unsupported / Warning |
| Algorithm 不符合最低要求 | 拒絕認證或標示 degraded |
| Certificate chain 失敗 | Security fault / Functional false |
| Challenge signature 失敗 | Endpoint untrusted |
| Measurement mismatch | Warning、Critical 或隔離，依產品政策 |
| Secure session 失敗 | 阻擋敏感 command 或使用受限模式 |

Policy 必須明確定義「記錄事件」、「限制功能」與「阻擋裝置」的條件，避免 security failure 只留在 debug console。

## 20.22 OpenBMC 資料映射

```text
MCTP Endpoint
    → Endpoint D-Bus object / Inventory association

PLDM FRU
    → Inventory Asset properties

PLDM Sensor
    → Sensor.Value / Availability / Thresholds

PLDM Event
    → Logging entry / Redfish EventLog

PLDM Firmware Update
    → Software inventory / Activation / UpdateService

SPDM Attestation
    → Security status / Health / Event
```

需要確認：

- Endpoint remove 時 inventory 與 sensors 更新。
- PDR repository change 後 D-Bus objects 同步更新。
- FRU 欄位使用正確權威端。
- SPDM raw measurements 與 certificate data 的存取權限。
- Redfish / EventLog 不洩漏敏感 security material。

## 20.23 Kernel、Yocto 與 Service

Build 需要確認：

- Kernel MCTP core。
- Binding drivers。
- MCTP userspace tools / daemon。
- PLDM daemon 與 `pldmtool`。
- `libpldm`。
- SPDM requester / responder library與 service。
- D-Bus interfaces。
- Systemd units。
- Endpoint-specific configuration。

```bash
bitbake -s | grep -Ei 'mctp|pldm|spdm|libpldm|libspdm'
bitbake -e virtual/kernel | grep '^S='

systemctl --type=service | grep -Ei 'mctp|pldm|spdm'
command -v mctp
command -v pldmtool
```

需保存 kernel、services、libraries、endpoint firmware 與 specification versions。

## 20.24 Target 端排查順序

### 20.24.1 Physical Binding

SMBus：

```bash
i2cdetect -l
dmesg | grep -Ei 'i2c|mctp'
```

PCIe VDM：

```bash
lspci -nn
lspci -vv
dmesg | grep -Ei 'pcie|vdm|mctp'
```

I3C：

```bash
dmesg | grep -Ei 'i3c|mctp'
```

### 20.24.2 MCTP

```bash
mctp link
mctp addr
mctp route

journalctl -b --no-pager | grep -Ei 'mctp|eid|route|endpoint'
```

確認 endpoint、EID、UUID、message types、MTU 與 route。

### 20.24.3 PLDM

```bash
systemctl --type=service | grep -i pldm
journalctl -b --no-pager | grep -Ei 'pldm|terminus|pdr|effecter'

pldmtool base GetPLDMTypes 2>/dev/null
pldmtool base GetPLDMCommands 2>/dev/null
```

`pldmtool` syntax 依版本與 endpoint selection方式不同，應先查看 `pldmtool --help`。

### 20.24.4 SPDM

依實際 requester 或 vendor tool 保存：

- Version response。
- Capabilities。
- Selected algorithms。
- Certificate slot 與 validation result。
- Challenge result。
- Measurement summary。
- Session state。
- Policy decision。

不保存 private key、derived secret 或未經遮蔽的敏感資料。

## 20.25 常見問題與判讀

| 現象 | 流程大約停在哪裡 | 優先檢查 |
|---|---|---|
| Endpoint 完全找不到 | Physical / binding | Power、link、driver、owner |
| EID 重複或變動 | Discovery / ownership | Bus owner、assignment policy、reset |
| Route 存在但 request timeout | Path / endpoint | MTU、next hop、power state、tag |
| 小 response 正常，大 response 失敗 | Fragment / MTU | Packet size、sequence、reassembly |
| PLDM type discovery 失敗 | Message type / terminus | Supported types、route、service |
| PDR repository讀取失敗 | PLDM transfer | Repository state、MTU、record handle |
| Sensor 數值錯誤 | PDR mapping | Unit、resolution、offset、raw value |
| Event 重複 | Event receiver / ACK | Sequence、retry、deduplication |
| FRU 欄位不一致 | Data authority | PLDM FRU、IPMI FRU、PMBus priority |
| Firmware update 中斷 | Transfer / activation | Block size、timeout、endpoint reset |
| SPDM version negotiation 失敗 | Version policy | Requester / responder support |
| Certificate validation 失敗 | Trust chain | Slot、CA、time、signature、revocation |
| Challenge 失敗 | Device identity | Transcript、algorithm、private-key possession |
| Measurement mismatch | Reference data | Device firmware、manifest、expected value |
| Secure session 無法建立 | Key exchange | DHE / AEAD、MTU、session state |
| Hot-plug 後 endpoint 不回來 | Rediscovery | Route cleanup、power、daemon signals |

## 20.26 Debug Log 收集

以下腳本以唯讀狀態為主。不同 image 的 commands 與 service names 可能不同。

```bash
#!/bin/sh

OUT=/tmp/mctp-pldm-spdm-debug
mkdir -p "$OUT"

cat /etc/os-release > "$OUT/os-release.txt" 2>&1
uname -a > "$OUT/uname.txt"
cat /proc/cmdline > "$OUT/proc-cmdline.txt"
zcat /proc/config.gz > "$OUT/proc-config.txt" 2>&1

dmesg -T > "$OUT/dmesg.txt"
journalctl -b --no-pager > "$OUT/journal.txt" 2>&1
systemctl --failed > "$OUT/systemctl-failed.txt" 2>&1
systemctl --type=service | grep -Ei 'mctp|pldm|spdm' \
    > "$OUT/services.txt" 2>&1

command -v mctp >/dev/null 2>&1 && {
    mctp link > "$OUT/mctp-link.txt" 2>&1
    mctp addr > "$OUT/mctp-addr.txt" 2>&1
    mctp route > "$OUT/mctp-route.txt" 2>&1
}

busctl tree xyz.openbmc_project.ObjectMapper \
    > "$OUT/objectmapper.txt" 2>&1
busctl tree xyz.openbmc_project.MCTP \
    > "$OUT/mctp-dbus.txt" 2>&1
busctl tree xyz.openbmc_project.PLDM \
    > "$OUT/pldm-dbus.txt" 2>&1

command -v lspci >/dev/null 2>&1 && \
    lspci -vv > "$OUT/lspci-vv.txt" 2>&1
command -v i2cdetect >/dev/null 2>&1 && \
    i2cdetect -l > "$OUT/i2cdetect-l.txt" 2>&1

# PLDM / SPDM commands需依 endpoint 與工具版本另外執行。
# 通用腳本不自動觸發 firmware update、effecter 或 repeated attestation。

tar czf "/tmp/mctp-pldm-spdm-debug-$(date +%Y%m%d-%H%M%S).tar.gz" \
    -C /tmp mctp-pldm-spdm-debug
```

## 20.27 Bring-up 順序

1. 建立 endpoint 清單與 physical binding path。
2. 確認 endpoint power、reset 與 presence。
3. 啟用 kernel MCTP core 與 binding driver。
4. 指定 bus owner 與 EID assignment policy。
5. 取得 EID、UUID、message types 與 path MTU。
6. 建立並驗證 routes。
7. 執行 PLDM Base discovery。
8. 建立 terminus 與 TID mapping。
9. 讀取 PDR repository，驗證 sensors、effecters 與 associations。
10. 驗證 PLDM event receiver 與 event handling。
11. 驗證 PLDM FRU、BIOS 與 firmware update，若 endpoint 支援。
12. 執行 SPDM version、capability 與 algorithm negotiation。
13. 驗證 certificate、challenge 與 measurements。
14. 依需求建立 secure session。
15. 將 PLDM 與 SPDM 結果映射到 D-Bus、Redfish 與 event。
16. 測試 endpoint reset、hot-plug、BMC reboot、Host power transition 與 path failure。
17. 保存 protocol versions、endpoint firmware、logs、routes 與 policy 結果。

## 20.28 平台實測紀錄表

| Endpoint | Binding / Path | Network / EID | UUID | Message Types | Power State | Owner |
|---|---|---|---|---|---|---|
| [待填] | [待填] | [待填] | [待填] | [待填] | [待填] | [待填] |
| [待填] | [待填] | [待填] | [待填] | [待填] | [待填] | [待填] |

| PLDM 項目 | 實測值 | 備註 |
|---|---|---|
| TID / Terminus | [待填] | EID 對照 |
| Supported Types | [待填] | Base / Platform / FRU / BIOS / FWU |
| PDR Count | [待填] | Repository version / change number |
| Numeric Sensors | [待填] | Unit / scale |
| State Sensors | [待填] | State sets |
| Effecters | [待填] | Authorization |
| FRU Records | [待填] | Inventory authority |
| Event Receiver | [待填] | ACK / retry |
| FW Update | [待填] | Transfer / activation / recovery |

| SPDM 項目 | 實測值 | 備註 |
|---|---|---|
| Version | [待填] | Minimum policy |
| Capabilities | [待填] | Cert / challenge / measurement / session |
| Algorithms | [待填] | Hash / asym / DHE / AEAD |
| Certificate Slot | [待填] | Trust anchor |
| Challenge | [待填] | Pass / fail |
| Measurements | [待填] | Expected values |
| Secure Session | [待填] | Session / heartbeat / key update |
| Policy Result | [待填] | Health / event / restriction |

## 20.29 驗收 Checklist

MCTP：

- [ ] 所有 endpoints 都有 binding、physical path、EID、UUID 與 owner 紀錄。
- [ ] Bus owner 與 EID assignment policy 已確認。
- [ ] Link、local address、route 與 path MTU 已驗證。
- [ ] BMC reboot、endpoint reset 與 hot-plug 後可重新 discovery。
- [ ] Large message fragmentation / reassembly 已測試。

PLDM：

- [ ] Base Type、Version 與 Commands discovery 正常。
- [ ] EID、TID 與 terminus identity 已對齊。
- [ ] PDR repository 可完整讀取並處理 change event。
- [ ] Sensor unit、scale、range、threshold 與 association 正確。
- [ ] Effecter 具有 range、權限與 audit controls。
- [ ] Event receiver、ACK、retry 與 deduplication 已驗證。
- [ ] FRU、BIOS 與 Firmware Update 依 endpoint capability 完成測試。

SPDM：

- [ ] Version、capabilities 與 algorithms 符合 security policy。
- [ ] Certificate chain 可追溯到核准的 trust anchor。
- [ ] Challenge signature 驗證成功。
- [ ] Measurements具有 expected value 或 signed manifest 可供判讀。
- [ ] Firmware update 後 expected measurements 會同步更新。
- [ ] Secure session 的建立、heartbeat、key update 與 teardown 已測試，若使用。
- [ ] Logs 不包含 private key 或 session secret。
- [ ] Attestation failure 會產生明確 policy result 與 security event。

OpenBMC：

- [ ] Endpoint、inventory、sensor、software 與 security objects 正確建立。
- [ ] Redfish 與 EventLog 能呈現必要結果。
- [ ] Endpoint 消失時 stale routes、termini、sensors 與 associations 會清理。
- [ ] Debug logs、protocol traces、versions 與 endpoint firmware 已保存。

## 20.30 本章重點

1. MCTP 提供 endpoint 定址、message type、fragmentation 與 routing。
2. EID 是 network 內的 logical address；UUID 適合用來辨識較穩定的 endpoint identity。
3. Bus owner 負責 discovery 與 EID assignment，平台只能有清楚一致的 ownership policy。
4. Transport binding 可使用 SMBus、PCIe VDM、I3C 或其他媒體。
5. PLDM terminus 透過 Base discovery 宣告支援的 Types、Versions 與 Commands。
6. PDR 描述 sensors、effecters、entities 與 associations。
7. PLDM Firmware Update 需要處理資料傳輸、activation、中斷與 recovery。
8. SPDM 透過 version、capability、algorithm、certificate、challenge 與 measurement 建立信任判斷。
9. Measurement digest需要 expected value 或 manifest 才能形成 attestation 結果。
10. MCTP、PLDM 與 SPDM 的結果應整合到 OpenBMC D-Bus、Redfish、Update Service 與 Security Event。

## 20.31 本章參考資料

- DMTF MCTP Base Specification DSP0236: https://www.dmtf.org/dsp/DSP0236
- DMTF PLDM Specifications: https://www.dmtf.org/standards/pmci
- DMTF SPDM Specification DSP0274: https://www.dmtf.org/standards/spdm
- DMTF SPDM over MCTP Binding DSP0275: https://www.dmtf.org/dsp/DSP0275
- DMTF libspdm: https://github.com/DMTF/libspdm
- Linux kernel MCTP documentation: https://docs.kernel.org/networking/mctp.html
- OpenBMC PLDM project: https://github.com/openbmc/pldm
- OpenBMC documentation: https://github.com/openbmc/docs
