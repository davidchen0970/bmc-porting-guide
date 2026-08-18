## 14.1 NC-SI 

### 定義與目的

NC-SI (Network Controller Sideband Interface) 規範（DMTF DSP0222）主要用於伺服器的遠端管理架構，其重點包含：
* 核心功能：讓伺服器的 BMC 與 Host NIC 能夠互相通訊。
* 資源共用：允許 BMC 與主機共用實體網路連接埠（Shared NIC）。
* 應用場景：支援帶外（Out-of-Band）遠端管理，如 IPMI、Redfish 和 KVM-over-IP。

採用 Shared NIC 架構可省去獨立 PHY 晶片與專用 RJ45 管理埠（Dedicated PHY），大幅節省伺服器主機板面積、降低 BOM 成本並減少待機功耗。

### 實體層與傳輸介面

NC-SI 在硬體實作上主要分為兩大演進世代：

#### 1. RBT (RMII-Based Transport)

傳統基於 50MHz 時脈的點對點 / 多點匯流排介面，由 RMII 衍生而來。腳位定義如下：

| 訊號名稱 | 方向 (BMC 視角) | 說明 |
| --- | --- | --- |
| **REF_CLK** | 輸入 (來自 NIC) | 50 MHz 基準時脈訊號 |
| **NCSI_TXD[1:0]** | 輸出 | BMC 傳送至 NIC 的資料線（雙線） |
| **NCSI_TX_EN** | 輸出 | Transmit Enable，指示 TXD 資料有效 |
| **NCSI_RXD[1:0]** | 輸入 | NIC 傳送至 BMC 的資料線（雙線） |
| **NCSI_CRS_DV** | 輸入 | Carrier Sense / Receive Data Valid，指示 RXD 資料有效 |
| **NCSI_ARB_IN** | 輸入 | 多點拓撲硬體仲裁輸入線（Multi-drop Arbitration） |
| **NCSI_ARB_OUT** | 輸出 | 多點拓撲硬體仲裁輸出線 |

#### 2. MCTP over PCIe / SMBus / I2C

現代高階伺服器與 OCP 3.0 NIC 規範中，逐漸轉向採用 **MCTP (Management Component Transport Protocol, DSP0236)** 作為底層傳輸載體：

* **MCTP over PCIe VDM**：透過 PCIe Vendor Defined Message 傳輸 NC-SI 指令（DSP0222 over MCTP）。頻寬高、線路簡化，且直接利用 PCIe 匯流排完成傳送。
* **MCTP over SMBus/I2C**：主要用於備用通道或低速控制。

## 14.2 拓撲、封包格式與工作流程

### 拓撲結構 (Package & Channel Model)

NC-SI 採用邏輯層級定址機制，由 8-bit 的 Channel ID 構成：

* **Package ID [7:5]**（高 3-bit）：代表實體 NIC 控制器（最多 8 個 Package, 0~7, 下圖使用兩個）。
* **Channel ID [4:0]**（低 5-bit）：代表 Package 內部的邏輯通道（最多 31 個 Channel, 0~30），通常對應實體 Network Port 或 PCIe Virtual Function (VF)。
* **特殊定址**：
    * `0x1F` 代表 Package 內部的廣播通道（Internal Broadcast）
    * `0xFF` 代表所有 Package 與 Channel 的全域廣播。

```
+-----------------------------------------------------------------------------------------+
|                                       NC-SI Topology                                    |
|                                                                                         |
|  +---------------------------------------+   +---------------------------------------+  |
|  |             Package 0                 |   |            Package 1                  |  |
|  |  +---------------+ +---------------+  |   |  +---------------+                    |  |
|  |  |   Channel 0   | |   Channel 1   |  |   |  |   Channel 0   |                    |  |
|  |  | (Phys Port 0) | | (Phys Port 1) |  |   |  | (Phys Port 0) |                    |  |
|  |  +---------------+ +---------------+  |   |  +---------------+                    |  |
|  +---------------------------------------+   +---------------------------------------+  |
+-----------------------------------------------------------------------------------------+
```

### 封包格式 (Control Frame Format)

所有 NC-SI 控制封包均封裝於標準 Ethernet 訊框中（EtherType 固定為 `0x88F8`）。Request（BMC 指令）與 Response（NIC 回應）的結構如下：

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Destination MAC (6B)                    |
|                        FF:FF:FF:FF:FF:FF                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Source MAC (6B)                      |
|                       (BMC Hardware MAC)                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|       EtherType (0x88F8)      |  MC ID (0x00) |  Rev (0x01)   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Reserved     |  Instance ID  | Command Type  |  Channel ID   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Rsvd|    Payload Length (12b)  |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               |
|                                                               |
|                   Payload (Command / Response)                |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Checksum (32-bit)                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

#### 封包標頭欄位詳細說明

| Offset (Byte) | 欄位名稱 | 長度 | 說明 |
| --- | --- | --- | --- |
| `00..05` | **Destination MAC** | 6 B | 廣播位址 `FF:FF:FF:FF:FF:FF` |
| `06..11` | **Source MAC** | 6 B | BMC 的硬體 MAC 位址 |
| `12..13` | **EtherType** | 2 B | 固定為 `0x88F8` |
| `14` | **MC ID** | 1 B | Management Controller ID，預設 `0x00` |
| `15` | **Header Revision** | 1 B | 當前規範版本，固定為 `0x01` |
| `16` | **Reserved** | 1 B | 保留位元，填 `0x00` |
| `17` | **Instance ID** | 1 B | 指令序號（Sequence Number），Response 必須相符 |
| `18` | **Command Type** | 1 B | 指令型態 (`0x00`~`0x7F` 為 CMD，`0x80`~`0xFF` 為 RSP/AEN) |
| `19` | **Channel ID** | 1 B | 位址：`[7:5]` Package ID, `[4:0]` Channel ID |
| `20..21` | **Payload Length** | 2 B | Payload 長度（低 12-bit，不含 Header 與 Checksum） |
| `22..N` | **Payload Data** | N B | 隨 Command Type 改變。Response 前 4 Bytes 固定為 `Response Code` (2B) 與 `Reason Code` (2B) |
| `N+1..N+4` | **Checksum** | 4 B | 根據 Ethernet 標頭至 Payload 結尾計算之 IEEE 802.3 CRC32 |

### 完整 NC-SI 控制指令集

| Command | 代碼 | Response 代碼 | 功能與目的 |
| --- | --- | --- | --- |
| **Clear Initial State (CIS)** | `0x00` | `0x80` | 清除 Channel 的初始化狀態標記，開啟對話 |
| **Select Package (SP)** | `0x01` | `0x81` | 選定並鎖定目標 Package，以便後續進行組態配置 |
| **Deselect Package (DP)** | `0x02` | `0x82` | 解除選定指定的 Package |
| **Enable Channel (EC)** | `0x03` | `0x83` | 開啟 Pass-through 網路流量轉發功能 |
| **Disable Channel (DC)** | `0x04` | `0x84` | 關閉 Pass-through 流量轉發 |
| **Reset Channel (RC)** | `0x05` | `0x85` | 軟體重置指定的 Channel 狀態機 |
| **Enable Channel TX (ECTX)** | `0x06` | `0x86` | 允許 BMC 發送網路封包至外部網路 |
| **Disable Channel TX (DCTX)** | `0x07` | `0x87` | 禁止 BMC 發送網路封包 |
| **Enable AEN (EA)** | `0x09` | `0x89` | 啟用非同步事件通知（Link Status, Host Driver Status） |
| **Get Link Status (GLS)** | `0x0A` | `0x8A` | 查詢實體 Port 狀態（Link Up/Down, Speed, Duplex, Pause） |
| **Get Version ID (GVI)** | `0x15` | `0x95` | 讀取 NIC 韌體版本、PCI ID 與 Firmware Build Info |
| **Get Capabilities (GC)** | `0x16` | `0x96` | 讀取 NIC 支援的過濾器數量與混合模式功能 |
| **Set MAC Address (SMA)** | `0x0E` | `0x8E` | 寫入 BMC 專屬 MAC 到 NIC 單播過濾陣列（Unicast Filter） |
| **Enable Broadcast Filter (EBF)** | `0x10` | `0x90` | 設定廣播過濾器（ARP, DHCP, NetBIOS, Neighbor Discovery） |
| **Set VLAN Filter (SVF)** | `0x12` | `0x92` | 設定 802.1Q VLAN ID 過濾條件 |
| **Enable VLAN (EV)** | `0x13` | `0x93` | 開啟 VLAN 過濾模式（VLAN Only / Any VLAN） |
| **OEM Command** | `0x50` | `0xD0` | NIC 廠商（Mellanox, Broadcom, Intel）自訂延伸指令 |

### Linux NC-SI 驅動狀態機（State Machine Workflow）


這套狀態機負責 Linux Kernel 處理 NC-SI (Network Controller Sideband Interface) 的核心邏輯。簡單來說，BMC（基板管理控制器）需要透過網卡與外界溝通，其完整的生命週期如下：
* 初始化： 必須先經過「認識 ➔ 設定 ➔ 啟用」網卡，才能正常通訊。
* 例外處理： 當連線中斷或收到狀態異動通知時，機制會自動解除綁定或重新設定。

以下為各個狀態與階段的詳細拆解說明:

#### 1. 探測階段：`ncsi_dev_state_probe`

**目標：尋找並識別網卡與 Channel**
BMC 剛啟動或重新掃描時，會對 NC-SI 介面上的所有 Package 與 Channel 發送一連串指令，了解對方的身份：

* **CIS (Clear Initial State):** 清除 Channel 之前的初始狀態與歷史紀錄。
* **SP (Select Package):** 選取並啟用目標 Package，準備進行溝通。
* **GVI (Get Version ID):** 查詢網卡韌體版本與規格（例如 NC-SI 版本號）。
* **GC (Get Capabilities):** 獲取網卡支援的功能清單（例如支援多少個 MAC 位址、是否有 VLAN 過濾、是否支援 AEN 等）。

#### 2. 設定階段：`ncsi_dev_state_config`

**目標：將 BMC 的網路參數寫入網卡**
探測完成後，驅動會根據系統需求對目標 Channel 進行硬體參數設定：

* **SMA (Set MAC Address):** 將 BMC 的 MAC 位址寫入網卡的過濾器中，這樣網卡才會接收發給 BMC 的封包。
* **EBF (Enable Broadcast Filter):** 啟用廣播封包過濾（如 ARP），避免無關的廣播過度占用 Sideband 頻寬。
* **SVF (Set VLAN Filter):** 如果有設定 VLAN，啟用 VLAN tag 過濾機制。
* **EA (Enable AEN):** 啟用 **Async Event Notification (非同步事件通知)**。這是非常關鍵的一步，讓網卡在實體連線（Link）狀態改變時，能主動通知 BMC。

#### 3. 啟動階段：`ncsi_dev_state_start`

**目標：開啟 Channel 資料通道並確認連線狀態**
參數設定完畢後，驅動會開啟資料通路：

* **EC (Enable Channel):** 開啟該 Channel 的 RX（接收）通道。
* **ECTX (Enable Channel TX):** 開啟該 Channel 的 TX（發送）通道。
* **GLS (Get Link Status):** 主動查詢實體網線（Ethernet Link）目前是否正常連線（Link Up/Down、速率與雙工狀態）。

#### 4. 正常運作階段：`ncsi_dev_state_functional`

**目標：繫結網路介面並開始正常資料傳輸**

* 走到這裡代表 NC-SI 頻道已完全就緒。
* 驅動會將 NC-SI 邏輯 Channel 與 Linux 的網路介面（如 `eth0` 或 `ncsi0`）綁定。此時 BMC 的 TCP/IP 協定棧可以正常進行 Send/Receive 網路封包。

#### 5. 異常與事件處理（分支路徑）

當 Channel 在正常運作時發生變化，會走入以下兩個分支之一：

* **左分支：`ncsi_dev_state_suspend`（連線中斷 / 逾時處理）**
    * **觸發條件：** 實體網線被拔除（Link Down）或 NC-SI 指令發送逾時（Timeout）。
    * **處理動作：** 當前 Channel 失去功能，驅動會進入掛起（Suspend）狀態，並觸發 **Failover (失效備援)** 機制，試圖將網路流量切換至備用的 NC-SI Channel 或 Package 上。


* **右分支：`ncsi_handle_aen_packet`（非同步事件接收）**
    * **觸發條件：** 收到網卡主動推播的 **AEN 封包**（例如網卡硬體偵測到 Link 狀態變更、熱插拔事件或驅動重置）。
    * **處理動作：** 驅動不會直接將網路判定為死亡，而是接收 AEN 訊息後，**觸發狀態重新評估（Re-evaluation）**，驅動會再次走一遍狀態機（發送 GLS 或重新 Config）來確認網卡目前的真實狀況。

```
                    +--------------------------+
                    |   ncsi_dev_state_probe   |
                    | (Send CIS, SP, GVI, GC)  |
                    +------------+-------------+
                                 |
                                 v
                    +--------------------------+
                    |  ncsi_dev_state_config   |
                    | (Send SMA, EBF, SVF, EA) |
                    +------------+-------------+
                                 |
                                 v
                    +--------------------------+
                    |   ncsi_dev_state_start   |
                    |   (Send EC, ECTX, GLS)   |
                    +------------+-------------+
                                 |
                                 v
                   +---------------------------+
                   | ncsi_dev_state_functional |
                   |  (Bind interface, normal  |
                   |        operation)         |
                   +-------------+-------------+
                                 |
             +-------------------+-------------------+
             | (Link Down / Timeout)                 | (AEN Received)
             v                                       v
+--------------------------+           +--------------------------+
|  ncsi_dev_state_suspend  |           |  ncsi_handle_aen_packet  |
|  (Switch to Failover     |           |  (Trigger state          |
|        Channel)          |           |   reevaluation)          |
+--------------------------+           +--------------------------+
```

## 14.3 Linux 維運、診斷指令與測試驗證

### 診斷與除錯指令

在 OpenBMC 或一般 Linux 管理系統中，可透過下列指令除錯：

#### 1. 核心紀錄檢視

```bash
# 查看 NC-SI 初始化與探測歷程
dmesg | grep -Ei 'ncsi|package|channel|aen'

# 範例輸出：
# [    4.120000] ncsi0: NCSI: Configuring channel 0 (package 0, channel 0)
# [    4.250000] ncsi0: Channel 0 is up

```

#### 2. 底層狀態與拓撲查詢 (`ncsi-netlink`)

```bash
# 顯示所有探測到的 Package, Channel 與目前 Active Path
ncsi-netlink --info

# 強制重新配置指定的 NC-SI 介面
ncsi-netlink --set --package=0 --channel=0

```

#### 3. 抓取原生 NC-SI 控制封包 (`tcpdump`)

```bash
# 監聽 eth0 上的 EtherType 0x88f8 控制封包並以 Hex 輸出
tcpdump -i eth0 ether proto 0x88f8 -XX -vv

```

#### 4. Debugfs 資訊讀取

```bash
# 檢視系統中的 NC-SI 介面與 Channel 拓撲
cat /sys/kernel/debug/ncsi/eth0/topology

```

### 測試驗證矩陣 (Verification Scenarios)

| 測試情境 | 觸發動作 | 預期系統行為 | 評估指標 / SLA |
| --- | --- | --- | --- |
| **S5/G3 待機啟動** | 僅插上 AC 電源（Host Off） | NIC 依靠 3.3VAUX 運作，NC-SI 完成 Probe，BMC 成功配發 IP 並開通 ping | 系統通電 15 秒內可透過 OOB 管理 |
| **Host 電源開關** | 切換 Host 電源（S0 ↔ S5） | NC-SI Pass-through 流量不中斷，不出現介面 Flapping 或封包遺失 | Ping 延遲波動 < 50ms，丟包率 0% |
| **NIC Reset 復原** | 對網卡執行 Firmware Reset 或 PCIe Reset | Linux 收到 `Configuration Required` AEN 或 Timeout，自動發送 CIS 重新初始化 | 網路服務恢復時間 < 5 秒 |
| **實體線路拔插** | 手動拔除並重新插上 RJ45 網路線 | NIC 發送 Link Change AEN (`0x00`)，BMC 記錄 Link Down/Up 狀態 | AEN 於 100ms 內觸發，插回自動恢復 |
| **Multi-Port Failover** | 拔除主 Port (Channel 0) 線路 | 驅動將 Active Channel 切換至 Standby Port (Channel 1) | 切換時間 < 3 秒（丟包 <= 3 包） |
| **BMC 重啟** | 於 BMC 內執行 `reboot` | 實體網卡 Link 保持開啟，BMC 重新載入驅動後重新與 NIC 進行握手 | BMC 重新載入後 10 秒內恢復網路 |

## 14.4 實務限制、潛在風險與設計規範

### 1. 電源域與 Standby Power 規範

* **3.3VAUX 電源需求**：當伺服器處於關機待機狀態（S5/G3）時，NC-SI 介面與 NIC 的 Sideband 管理邏輯必須仰賴主機板提供的 `3.3VAUX`（或 PCIe `12VAUX`）供電。若 PCB 硬體設計未將 `3.3VAUX` 接入 PCIe 插槽，Host 關機後 BMC OOB 網路將立即中斷。
* **PCIe 電源狀態轉移**：當 Host 作業系統進入深層省電狀態（D3cold）時，必須透過 NIC 韌體中的 NC-SI 管理模組關閉 PCIe 介面省電阻斷，確保 Sideband 介面維持供電。

### 2. MAC 位址與 VLAN 隔離策略

* **MAC 位址獨立性**：BMC 必須配置獨立於 Host 的 MAC 位址。在 初始化階段，Linux 驅動會發送 `Set MAC Address (SMA)` 指令，將 BMC MAC 寫入 NIC 的 RX 過濾清單。
* **VLAN 隔離策略**：由於 Host 與 BMC 共用實體網口，建議建立 802.1Q VLAN Tag 隔離（例：BMC 指定 VLAN 10，Host 指定 VLAN 20），以防止兩端廣播風暴相互干擾，並避免未授權流量傾印。

```
                [ Physical Network Port ]
                            |
         +------------------+------------------+
         | (VLAN 10)                           | (VLAN 20)
         v                                     v
  [ BMC MAC Filter ]                   [ Host NIC Driver ]
  [ NC-SI Channel  ]                   [ PCIe Data Path  ]
         |                                     |
         v                                     v
  (BMC OS Network)                      (Host OS Network)

```

### 3. 安全性與隔離風險 (Security & Isolation)

* **側邊通道邊界跨越風險**：Shared NIC 跨越了安全邊界。惡意的 Host OS 驅動可能透過 DMA 或存取 NIC 暫存器，嘗試阻斷或側錄 Sideband 通道流量。
* **防禦機制**：
* **Secure Boot**：NIC 韌體必須啟用 Secure Boot 與代碼簽署 verification。
* **Command Isolation**：NIC 韌體必須限制限制僅有來自 NC-SI 實體通道（RBT / MCTP）的指令能夠修改 BMC 過濾器規則，嚴禁 Host 端的 PCIe 驅動修改或讀取 BMC Channel 配置。



### 4. 錯誤處理與 Re-initialization Storm 防範

* **Timeout 與 重試機制**：NC-SI 控制指令逾時時間設定為 100ms。驅動程式在放棄前應進行最多 3 次重試。
* **Re-init Storm 抑制**：在實體線路品質不良（如線路接觸不良）或連線閃斷時，頻繁的 AEN (Link Change) 會誘導 BMC 不斷發送 CIS 指令發起重新初始化。驅動層必須導入 **Debounce Timer** 與 **Rate-limiting** 機制（例如：10 秒內最多允許 2 次全域重置），避免系統陷入重置死鎖。

### 5. 廠商自訂命令 (OEM Commands)

雖然 DMTF 定義了標準 NC-SI 規範，但各大 NIC 廠商（如 NVIDIA/Mellanox, Broadcom, Intel）皆提供自訂的 OEM Command (`0x50`) 以支援進階功能：

* **廠商代碼 (Enterprise Number)**：OEM Payload 前 3 Bytes 為 IANA 廠商代碼（例如 Mellanox 為 `0x0002C9`，Broadcom 為 `0x001018`）。
* **常見 OEM 延伸用途**：
* **NCSI-over-MCTP 延伸控制**。
* **SFP / QSFP 光纖模組 DDM 資訊讀取**（電壓、溫度、光功率）。
* **網卡實體溫度感測器讀取**（匯入 BMC 風扇控制演算法）。
* **Multi-Host 系統邏輯通道綁定與綁定開關**。
