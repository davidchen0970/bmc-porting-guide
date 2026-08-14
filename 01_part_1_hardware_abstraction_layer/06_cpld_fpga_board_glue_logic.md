# 6. CPLD、FPGA 與 Board Glue Logic

CPLD、FPGA 與 board glue logic 用來連接、判斷或控制板上的多個硬體訊號. 它們常負責 power sequence、reset、fault latch、presence、LED、board ID、flash write protect 與 bus mux, 並在 BMC Linux 尚未啟動前維持安全狀態.

本章先說明 CPLD、FPGA 與 glue logic 的差異, 再介紹 register、state machine、fault latch、reset、presence、LED、write protect、firmware update, 以及 OpenBMC 如何讀取與控制這些功能.

## 適用範圍

本章涵蓋 BMC 平台中的 CPLD、FPGA、board glue logic 與 platform MCU, 包括 register map、state machine、fault latch、reset、presence、LED、write protect、firmware update, 以及 Linux 與 OpenBMC 整合.

## 適用讀者

- 負責 BMC 硬體 bring-up、CPLD / FPGA 整合、Linux kernel、OpenBMC service 或平台驗證的人員.
- 需要排查 power sequence、reset routing、fault latch、register access、flash mux 或 programmable logic update 的人員.

## 快速導覽

- [元件角色與分工](#section-6-1)
- [硬體訊號至 OpenBMC 的流程](#section-6-3)
- [Register Map](#section-6-4)
- [安全讀寫 Register](#section-6-5)
- [Power Sequence 與 State Machine](#section-6-6)
- [Fault Status 與 Fault Latch](#section-6-8)
- [Write Protect 與 Flash Mux](#section-6-11)
- [Firmware Update 與 Recovery](#section-6-12)
- [Linux 與 OpenBMC 整合](#section-6-13)
- [Target 端排查流程](#section-6-15)
- [Debug Log 收集](#section-6-17)
- [驗收 Checklist](#section-6-20)

<a id="section-6-1"></a>

## 6.1 CPLD、FPGA 與 Board Glue Logic 是什麼

### 6.1.1 CPLD

CPLD（Complex Programmable Logic Device，複雜可程式化邏輯元件）是一種可由工程師定義內部邏輯與腳位行為的數位元件。在伺服器主板上，它常用來連接並協調 PSU、VR、CPU、PCH、BMC、按鈕、LED、Flash 與除錯介面之間的板級訊號。

初次接觸 CPLD 時，可以先把它理解成一個持續運作的硬體判斷與控制中心：

```text
外部輸入
Power button / PGOOD / Fault / Board ID / BMC request
                         ↓
CPLD 內部邏輯
組合邏輯 + 計數器 + 狀態機 + latch + register
                         ↓
外部輸出
Power enable / Reset / LED / Write protect / Mux select
```

這個理解方式的重點不是把 CPLD 擬人化，而是看清楚它的資料流：CPLD 持續取樣輸入訊號，依照預先設計的條件、優先順序與時間限制更新內部狀態，再驅動對應輸出。BMC 可以讀取狀態或提出 request，但最後是否允許上電、解除 reset 或切換 mux，仍可能由 CPLD 根據安全條件決定。

#### CPLD 和一般軟體程式有什麼不同

BMC 軟體通常依序執行指令，並受到開機時間、作業系統排程、driver 與 service 狀態影響。CPLD 則由邏輯閘、暫存器、計數器與 state machine 組成，多組邏輯可以平行監看不同訊號，不需要等待 Linux 啟動後才開始判斷。

```text
BMC / Linux：
BootROM → Bootloader → Kernel → Driver → Service → 發出 request

CPLD：
待機電源與必要 clock/reset 就緒 → 依既定邏輯監看與控制訊號
```

因此，CPLD 適合承擔：

- 在 BMC Linux 尚未 ready 前就必須維持的安全狀態。
- 具有明確順序、條件與 timeout 的 power/reset 流程。
- 需要持續監看、不能依賴 userspace polling 的 fault 訊號。
- 需要固定 pulse width、debounce 或訊號互鎖的控制。
- BMC reboot 或 service 異常時仍需保持的 host 狀態。

這不表示 CPLD 適合所有控制。CPLD 通常負責即時且規則固定的硬體行為；BMC 則適合較複雜的策略、事件記錄、遠端管理與韌體更新。實際分工由平台設計決定。

#### CPLD 內部通常有哪些基本邏輯

- **組合邏輯（Combinational logic）**：輸出由目前輸入條件決定。
- **暫存器（Register）**：保存狀態或提供 BMC 讀寫介面。
- **計數器與 timer**：計算 debounce、reset pulse、PGOOD timeout 或 watchdog timeout。
- **狀態機（State machine）**：依序執行上電、關機與復原流程。
- **Latch**：記住某事件曾經發生，直到指定 clear 或 reset 條件成立。
- **Mux control 與 interlock**：選擇訊號路徑，避免兩個 owner 同時驅動同一條 bus。

後續章節會分別說明這些結構。本節先用五類板級職責建立整體概念。

#### 1. 電源時序與硬體保護

伺服器不會在按下 power button 後同時打開所有電源。PSU、主電源、各組 VR、CPU/PCH 相關電源與 slot power，通常具有指定的開啟順序及相依條件。

CPLD 可能接收 `AC_OK`、`POWER_BUTTON_N`、`BMC_POWER_REQUEST`、`PSU_PGOOD`、`VR_PGOOD` 與 `VR_FAULT_N`，並輸出 `PSU_ON_N`、`VR_EN`、`SLOT_POWER_EN` 與 `SYS_PWROK`。

```text
收到 Host power-on request
        ↓
確認 AC、standby power 與安全條件
        ↓
開啟 PSU，等待 PSU_PGOOD
        ↓
開啟 VR，等待各路 VR_PGOOD
        ↓
產生 SYS_PWROK
        ↓
在條件成立後解除相關 reset
```

`PGOOD`（Power Good）是電源元件回報輸出已進入規格範圍的訊號。CPLD 不只是延遲固定時間，也可能同時等待 PGOOD。如果 timeout 前沒有收到，或 sequence 中途出現 fault，CPLD 可停止流程、關閉部分輸出、進入 shutdown state，並設定 fault latch。

實際 fault 行為不能概括為「任何異常都立即斷電」。某些 fault 要立即關閉相關 rail，某些會通知 BMC 後進行 controlled shutdown，另一些可能允許 retry。處理方式需依硬體規格與 CPLD state machine 確認。

#### 2. Reset 產生、延遲與路由

Reset 不只是把一條訊號拉成 Low。主板上可能同時存在 BMC reset、host reset、PCH reset、PCIe reset、slot reset 與 full-board reset，每個 target 的 active level、持續時間及允許條件可能不同。

CPLD 可能負責：

- 電源未穩定時持續 assert reset。
- PGOOD 與 ready 條件成立後才 release reset。
- 將短暫 request 轉成符合規格的 reset pulse。
- 根據 mux 或 policy 選擇 reset target。
- 區分 BMC reboot、host reset、watchdog reset 與 AC cycle。

```text
BMC 寫入 HOST_RESET_REQUEST
        ↓
CPLD 檢查 host power state 與 interlock
        ↓
產生指定寬度的 HOST_RESET_N pulse
        ↓
結束 pulse 並保存 reset reason
```

Watchdog 本體可以位於 BMC、host、CPLD 或其他元件。CPLD 常負責接收 timeout 訊號或依設計內建計數器，再把 reset 導向指定 target。因此，不能只看到「watchdog reset」就假設計數器一定在 CPLD 內。

#### 3. 身分、Presence 與按鈕訊號整理

Board ID、SKU ID、cable ID 與 device presence 常由多條 strap 或 connector pin 組成。CPLD 可將分散的實體訊號整理成 BMC 可讀取的 register。

```text
ID pins / Presence pins
          ↓
取樣、反相、debounce 或編碼
          ↓
CPLD status register
          ↓
BMC driver / OpenBMC inventory
```

例如，三個 Board ID pins 可組成 `000` 到 `111` 的值，BMC 再依對照表判斷 board revision 或 SKU。文件需要說明 sampling 時機、active level、pull-up domain、合法值與未接腳位的處理方式。

Front panel button 也常先進入 CPLD。機械按鈕可能在短時間內反覆跳動，CPLD 可要求訊號連續穩定一段時間後才更新 debounced state，避免一次按壓被判斷成多次事件。

#### 4. Write Protect 與硬體訊號路由

BIOS Flash、BMC Flash、JTAG 與 UART 可能由不同 controller 共用。CPLD 常控制 write protect 與 mux，但兩者並不相同：

- **Write protect**：決定目標 flash 是否允許寫入。
- **Flash mux**：決定目前由 BMC、host 或外部燒錄器中的哪一方連接 flash。
- **UART/JTAG mux**：決定實體除錯接點目前連到哪個 target。

Flash mux 切換前，原 owner 應停止驅動 clock、chip select 與 data lines。CPLD 可以加入 interlock，只有在原 owner 停止使用且隔離條件成立時才允許切換，以避免 bus contention。

BMC 可以提出解除 write protect 的要求，但 CPLD 可能還會檢查 power state、security strap、physical presence 或 update mode。CPLD 通常維持硬體條件與安全預設，不宜假設它會自行完成映像簽章驗證等高層更新政策。

#### 5. Fault、LED 與除錯狀態

CPLD 常同時提供「現在的狀態」與「曾經發生過的狀態」：

```text
Raw fault / current status
表示此刻 pin 或條件是否異常

Fault latch
表示自上次 clear 或指定 reset 之後，是否曾經發生異常
```

如果 fault pulse 很短，BMC 的週期性讀取可能看不到 raw fault。CPLD 可在硬體層偵測事件並設定 fault latch，讓 BMC 稍後仍能取得線索。

Latch 能保存多久取決於 reset domain 與供電狀態。有些在 BMC reboot 後仍保留，有些會在 CPLD reset 或 AC cycle 時清除。若要求完全斷電後仍保留，通常還需要 non-volatile storage 或外部 log，不能假設一般 CPLD register 會永久保存。

LED 可直接反映 CPLD 狀態，也可由 BMC 選擇模式。若兩者都能控制同一顆 LED，文件必須定義 owner 與 priority，否則 register、實際燈號與 Redfish 狀態可能不一致。

#### 一個完整的小案例

以下用「按下 power button 後 Host 上電失敗」串起 CPLD 的工作：

```text
1. 使用者按下 POWER_BUTTON_N
2. CPLD debounce，確認不是接點抖動
3. CPLD 設定 power request 或通知 BMC
4. Power state machine 由 IDLE_OFF 進入 PSU_ON
5. CPLD assert PSU_ON_N
6. CPLD 等待 PSU_PGOOD，並啟動 timeout counter
7. 若 PSU_PGOOD 成立，繼續開啟 VR
8. 若 timeout，停止 sequence 並設定 PSU_TIMEOUT_LATCH
9. CPLD 保持必要 reset，不讓 host 在電源未穩定時啟動
10. BMC 讀取 power state、raw PGOOD 與 fault latch
11. OpenBMC 建立 event，對外呈現 power-on failure
```

在這個流程中：

- CPLD 執行即時訊號判斷、timeout、輸出控制與硬體保護。
- BMC 提出需求、讀取結果，並依 policy 決定 retry、關機或告警。
- OpenBMC service 將底層 register 轉成 inventory、state、event log、Redfish 或 IPMI 資訊。

#### 閱讀 CPLD 功能時的八個問題

1. 輸入來自哪個元件，active-high 還是 active-low？
2. 輸入是 raw、debounced，還是已經過其他元件整理？
3. CPLD 使用組合邏輯、timer、state machine 還是 latch？
4. BMC 寫入的是 request，還是直接控制實體 output？
5. CPLD 還會檢查哪些 interlock、power state 或 security condition？
6. 最後驅動哪個輸出，影響哪些 target？
7. Timeout 或 fault 後會 shutdown、retry、保持 reset，還是只記錄狀態？
8. 狀態由哪種 reset 清除，BMC reboot 或 AC cycle 後是否保留？

用這八個問題拆解後，`power enable`、`reset`、`write protect` 或 `mux` 就不再只是功能名稱，而會成為可追蹤的「輸入 → 判斷 → 狀態 → 輸出」路徑。

CPLD 通常在 BMC 開機前就已開始工作，因此不能只把它視為 BMC 後面的 I/O 裝置。更準確地說，BMC 與 CPLD 是分工合作：CPLD 維持即時、固定且安全相關的底層邏輯；BMC 負責較高層的策略、記錄與管理。即使 BMC reboot 或部分 service 異常，平台仍可要求 CPLD 維持 host power、保持適當 reset、鎖住 write protect，或在 fault 發生時進入安全狀態。

### 6.1.2 FPGA

FPGA(Field-Programmable Gate Array)也是可程式化邏輯元件, 但通常具有更多邏輯資源、記憶體與高速 I/O, 適合較複雜的資料處理或自訂介面.

BMC 平台可能使用 FPGA 處理:

- 高速 bridge 或 protocol conversion.
- 自訂 sideband protocol.
- 大量訊號聚合.
- Timestamp 或 debug capture.
- PCIe、SerDes 或高速資料路徑.
- 動態載入 bitstream 的可變硬體功能.

FPGA 是否在上電後立即可用, 取決於 configuration source. 它可能從 SPI flash 自行載入, 也可能由 BMC、host 或 JTAG tool 載入 bitstream.

### 6.1.3 Board Glue Logic

Board glue logic 是把多個元件與訊號「接起來」的板級邏輯. 它不一定是 CPLD 或 FPGA, 也可能由簡單元件組成, 例如:

- Logic gate.
- Latch 或 flip-flop.
- Mux / demux.
- Wired-OR.
- Level shifter.
- RC delay.
- Supervisor 或 reset IC.

這些電路未必有 software register. 排查時需要看 schematic、量測波形, 並確認 pull-up、active level、propagation delay 與 power domain.

### 6.1.4 Platform MCU

有些平台使用小型 MCU 執行 power、thermal、security 或 hot-swap 控制. 它和 CPLD 的差異是: MCU 執行 firmware 指令, CPLD / FPGA 則以硬體邏輯平行運作.

BMC 通常透過 I2C、UART、mailbox 或 custom protocol 與 MCU 溝通, 需要另外管理:

- Firmware version.
- Command / response protocol.
- Timeout 與 retry.
- Firmware update.
- MCU 無回應時的 recovery.

## 6.2 為什麼伺服器主板需要 CPLD

BMC 可以執行複雜的管理程式, 但 Linux boot 需要時間, 排程延遲也不固定. 部分硬體功能不能等待 userspace service:

- AC 上電後立即保持 rail 與 reset 的安全狀態.
- 在指定時間內判斷 PGOOD.
- Fault 發生時立即關閉 power enable.
- 產生固定寬度的 reset pulse.
- BMC reboot 時維持 host power.
- 防止兩個 flash owners 同時驅動 bus.

因此常見分工是:

| 功能 | CPLD / FPGA | BMC / OpenBMC |
|---|---|---|
| 即時 power sequence | 執行時序與硬體保護 | 發出 power request、記錄結果 |
| Fault shutdown | 立即停止危險輸出 | 建立 event、決定後續處理 |
| Presence debounce | 過濾訊號 | 更新 inventory |
| LED pattern | 產生固定 blink pattern | 選擇 LED mode |
| Reset pulse | 產生固定 pulse | 發出 reset request |
| Board ID | 讀取 strap 並保存狀態 | 選擇平台設定 |
| Write protect | 維持安全預設與 mux interlock | 經授權流程要求解除 |

<a id="section-6-3"></a>

## 6.3 訊號從硬體到 OpenBMC 的流程

一個外部訊號通常不會直接變成 Redfish 或 IPMI 狀態. 以 PSU fault 為例:

```mermaid
flowchart TB
    A["PSU_FAULT_N 實體腳位"] --> B["CPLD sampling / debounce"]
    B --> C["Current status bit"]
    C --> D["Fault latch bit"]
    D --> E["BMC 透過 I2C / MMIO 讀取 register"]
    E --> F["OpenBMC fault service"]
    F --> G["D-Bus state / logging entry"]
    G --> H["Redfish EventLog / IPMI SEL"]
```

這條路徑中常見五種不同狀態.

### 6.3.1 Raw Input

Raw input 是 CPLD 腳位目前的實體電位, 例如:

- `PSU0_PRESENT_N`
- `VR_FAULT_N`
- `AC_OK`
- `POWER_BUTTON_N`

Raw bit 適合和示波器或 logic analyzer 波形比對, 但可能包含雜訊或接點彈跳.

### 6.3.2 Debounced State

Debounced state 是 CPLD 確認訊號持續穩定一段時間後的結果, 常用於 button、presence 與 mechanical switch.

```mermaid
flowchart TB
    A["Raw input 抖動"] --> B["連續穩定 N ms"]
    B --> C["Debounced state 改變"]
```

OpenBMC inventory 通常應使用 debounced state, 而不是 raw pin.

### 6.3.3 Latched State

Latch 用來記住「曾經發生過」的事件. 即使 raw fault 已恢復, latch 仍保持, 直到符合 clear rule.

```mermaid
flowchart TB
    A["VR_FAULT_N 短暫拉低"] --> B["Raw status 回復正常"]
    B --> C["VR_FAULT_LATCH 仍為 1"]
```

因此, raw status 表示「現在是否異常」, fault latch 表示「之前是否發生過異常」.

### 6.3.4 Control Request

Control request 是 BMC、host 或其他 controller 寫入的要求, 例如:

- Host power on request.
- Host reset request.
- UID LED mode.
- Flash mux select.
- Write protect request.

Request 不一定等於實際 output. 例如 BMC 寫入 power-on request 後, CPLD 仍可能因 interlock 或 fault 拒絕進入下一個 state.

### 6.3.5 Derived Output

Derived output 是 CPLD 綜合多個輸入、狀態與時序後產生的結果, 例如:

- `SYS_PWROK`
- `PERST_N`
- `PSU_ON_N`
- `VR_EN`
- `HOST_RESET_N`

排查 derived output 時, 需要知道它的完整 assertion condition, 而不能只看單一 control bit.

<a id="section-6-4"></a>

## 6.4 Register Map

BMC 通常透過 I2C、LPC、eSPI、MMIO、UART 或 mailbox 讀寫 CPLD / FPGA register. Register map 不只要列 offset, 也要描述每個 bit 的來源、生命週期與副作用.

### 6.4.1 Register 基本欄位

| 欄位 | 說明 |
|---|---|
| Offset | Register address |
| Name | Register 名稱 |
| Width | 8、16 或 32 bit |
| Access | RO、RW、W1C、Pulse 等 |
| Default | Reset 後初始值 |
| Reset domain | POR、CPLD reset、BMC reset 或 live |
| Description | Register 功能 |
| Owner | CPLD、BMC、host、security 等 |
| Side effect | Read / write 是否改變狀態 |

### 6.4.2 常見存取語意

| 縮寫 | 意義 | 注意事項 |
|---|---|---|
| RO | Read only | Read 不應改變狀態 |
| RW | Read / write | 需說明 reset default |
| WO | Write only | Read value 可能無意義 |
| W1C | Write one to clear | 寫 1 清除對應 bit |
| W1S | Write one to set | 寫 1 設定對應 bit |
| RC | Read clear | Read 後自動清除 |
| Pulse | 寫入後產生固定寬度 pulse | 需記錄 pulse width |
| Sticky | Reset 後仍可能保留 | 需說明何種 reset 會清除 |
| Live | 即時反映 pin 或 internal state | 可能隨時改變 |
| Shadow | 先保存設定, 之後才套用 | 需有 apply / commit 規則 |

### 6.4.3 Register Map 範例

| Offset | Register | Access | Default | 說明 |
|---:|---|---|---:|---|
| `0x00` | `CPLD_ID` | RO | [待填] | CPLD / board ID |
| `0x01` | `VERSION_MAJOR` | RO | [待填] | Major version |
| `0x02` | `VERSION_MINOR` | RO | [待填] | Minor version |
| `0x10` | `RAW_PRESENCE` | RO / Live | Pin state | Raw presence inputs |
| `0x11` | `PRESENCE_STATUS` | RO | [待填] | Debounced presence |
| `0x20` | `FAULT_LATCH` | RO | `0x00` | Latched faults |
| `0x21` | `FAULT_CLEAR` | W1C | `0x00` | Clear selected faults |
| `0x30` | `POWER_REQUEST` | RW | Safe off | BMC power requests |
| `0x31` | `POWER_STATE` | RO | [待填] | State machine state |
| `0x40` | `RESET_CONTROL` | WO / Pulse | `0x00` | Reset requests |
| `0x50` | `LED_CONTROL` | RW | [待填] | LED mode |
| `0x60` | `SECURITY_STATUS` | RO | [待填] | WP / mode / strap |
| `0x70` | `SCRATCH` | RW | `0x00` | Non-critical access test |

### 6.4.4 Bit 欄位

一個 bit 至少應記錄:

- Bit number.
- Name.
- Access type.
- Active level.
- Default.
- Raw、debounced、latched、request 或 derived state.
- Clear rule.
- Read / write side effect.
- Owner.
- Test method.

Fault bit 範例:

```text
Register    FAULT_LATCH
Bit         0
Name        VR_FAULT_LATCH
Access      RO
Active      1 = fault occurred
Default     0
Source      VR_FAULT_N after qualification
Clear       Write FAULT_CLEAR[0] = 1
Side effect Fault evidence disappears after clear
```

<a id="section-6-5"></a>

## 6.5 如何安全讀寫 Register

在讀寫 CPLD register 前, 需先確認:

- Access bus 與 address.
- Register map / CPLD image version.
- Register width 與 endianness.
- Bank / page selection.
- Address auto-increment.
- Read 是否有 clear side effect.
- Write 是否會切 power、reset、mux 或 write protect.
- Kernel driver 與 userspace tool 是否同時存取.

### 6.5.1 I2C 存取

```bash
# 列出 adapters
$ i2cdetect -l

# 掃描前先確認該 bus 是否允許掃描
$ i2cdetect -y <bus>

# Read 前先確認 register 不是 read-clear
$ i2cget -y <bus> <addr> <offset>

# Write 前確認完整副作用
$ i2cset -y <bus> <addr> <offset> <value>
```

不建議直接對未知 register 讀寫. 部分 register 可能:

- Read 後清除 fault.
- Write 1 後觸發 reset pulse.
- 切換 flash owner.
- 關閉 power rail.
- 解除 write protect.
- 改變 bank, 使後續工具讀錯位置.

### 6.5.2 Scratch Register

若硬體提供無副作用的 scratch register, 可先用它確認:

- Bus path 與 address 正確.
- Read / write 基本功能.
- Register width 與 byte order.
- Tool 與 image register map 相容.

Scratch register 不能和 power、reset、fault 或 security register 共用語意.

### 6.5.3 Regmap

Linux regmap framework 可統一處理 register access, 常用於 I2C、SPI 或 MMIO 裝置. 它可以協助管理:

- Register / value width.
- Endianness.
- Locking.
- Cache.
- Readable / writable / volatile registers.
- Debugfs register dump.

Read-clear、W1C、volatile status 與 precious register 需要正確描述, 避免 cache 或 debug read 破壞狀態.

<a id="section-6-6"></a>

## 6.6 Power Sequence 與 State Machine

CPLD 常以 state machine 控制 host power. State machine 會根據 request、PGOOD、fault 與 timeout, 依序切換 power enables 與 resets.

```mermaid
flowchart TB
    A["Host power-on request"] --> B["Enable PSU"]
    B -->|"等待 PSU_PGOOD"| C["Enable VR"]
    C -->|"等待 VR_PGOOD"| D["Assert SYS_PWROK"]
    D --> E["Release reset"]
    E --> F["Host on"]
```

### 6.6.1 State 需要記錄什麼

每個 state 至少需要:

- State ID 與名稱.
- Entry condition.
- CPLD outputs.
- 等待的 input signal.
- Timeout.
- Timeout 後的 fault latch.
- 下一個 state.
- Shutdown / retry 行為.

### 6.6.2 State Machine 範例

| State | 進入條件 | Output | 等待訊號 | Timeout 結果 |
|---|---|---|---|---|
| `IDLE_OFF` | AC present、host off | Rails off | Power request | 進入 `PSU_ON` |
| `PSU_ON` | Power request | Assert `PSU_ON_N` | `PSU_PGOOD` | Latch PSU timeout |
| `VR_ENABLE` | PSU good | Assert `VR_EN` | All `VR_PGOOD` | Latch VR timeout |
| `SYS_PWROK` | VR good | Assert `SYS_PWROK` | PCH ready | Latch PWROK timeout |
| `RELEASE_RESET` | PCH ready | Release reset | POST complete | Latch POST timeout |
| `HOST_ON` | Sequence complete | Monitor faults | Off request / fault | 進入 shutdown |
| `SHUTDOWN` | Request / fault | Deassert outputs | Rails off | 回到 `IDLE_OFF` |

### 6.6.3 Power On 失敗如何排查

先保存:

1. CPLD state.
2. Fault latch.
3. Raw PGOOD / fault bits.
4. BMC power request.
5. PMBus status.
6. Kernel 與 OpenBMC journal.
7. 相關訊號波形.

波形常包含:

```text
AC_OK
PSU_ON_N
PSU_PGOOD
VR_EN
VR_PGOOD
SYS_PWROK
RSMRST_N
PLTRST_N
PERST_N
POST_COMPLETE
```

State 停在哪裡, 通常能縮小到前一個 output、正在等待的 input, 以及對應 timeout.

## 6.7 Reset 與 Reset Mux

CPLD 可能接收多個 reset sources, 再依平台狀態決定輸出到哪個 target.

常見 reset sources:

- Front panel reset button.
- BMC host-reset request.
- BMC watchdog.
- Host watchdog.
- CPLD internal fault.
- External debug header.
- AC cycle 或 supervisor reset.

常見 targets:

- BMC only.
- Host only.
- PCIe devices.
- Chassis / full board.
- Particular slot.

### 6.7.1 Reset 文件需要的資料

| 欄位 | 說明 |
|---|---|
| Source | 誰產生 reset |
| Target | 哪些元件受影響 |
| Active level | High / low active |
| Pulse width | Reset 持續多久 |
| Gating | 什麼 power state 才允許 |
| Mux select | 由哪個 bit / strap 選擇 |
| Default | AC on 與 reset 後預設 |
| Side effect | 是否造成 host、BMC 或 bus 中斷 |

### 6.7.2 BMC Reboot 不應等於 Host Power Loss

若設計要求 BMC reboot 時 host 繼續運作, 必須確認:

- CPLD 自主維持 power sequence state.
- BMC GPIO reset default 不會關閉 rail.
- BMC-ready signal 不會直接解除 host enable.
- Watchdog target 沒有誤設成 full-board reset.
- BMC service 重新啟動後不會送出錯誤 power request.

<a id="section-6-8"></a>

## 6.8 Fault Status 與 Fault Latch

Fault status 與 fault latch 應分開:

| 狀態 | 定義 |
|---|---|
| Raw fault | 目前 fault pin 是否有效 |
| Fault latch | 從上次 clear 之後是否曾發生 fault |

### 6.8.1 Clear 流程

```mermaid
flowchart TB
    A["發現 fault latch"] --> B["保存 CPLD register dump"]
    B --> C["保存 PMBus / device status"]
    C --> D["保存 journal、event 與 waveform"]
    D --> E["確認 raw fault 已解除"]
    E --> F["執行 W1C 或指定 clear"]
    F --> G["再次讀取 raw 與 latch"]
```

若先 clear 再收集資訊, 原始 fault 證據可能消失.

### 6.8.2 Read-Clear Register

Read-clear register 只要讀取就會清除. 這類 register 不應被一般 polling service、debugfs dump 或重複的 shell script 任意讀取.

若 event 只能讀一次, 應由單一 owner 讀取, 再透過 D-Bus 或 logging service 分享結果, 避免多個程式競爭.

### 6.8.3 Fault Retry

Hot-swap 或 slot power fault 可能支援 retry. 需定義:

- 最大 retry 次數.
- Retry delay.
- 哪些 fault 可 retry.
- 哪些 fault 必須 latch off.
- AC cycle / manual clear 是否重設 counter.
- BMC 與 CPLD 誰擁有 retry policy.

## 6.9 Presence、Board ID 與 Strap

CPLD 常彙整多組 ID 與 presence pins.

### 6.9.1 Presence

Presence 可能來自:

- Connector pin.
- PSU / fan presence pin.
- Cable ID pins.
- GPIO expander.
- Hot-swap controller.

需記錄 active level、pull-up domain、debounce time 與 hot-plug 行為.

```mermaid
flowchart TB
    A["Raw presence"] --> B["CPLD debounce"]
    B --> C["Presence status register"]
    C --> D["OpenBMC inventory Present property"]
    D --> E["Sensor availability / Redfish inventory"]
```

### 6.9.2 Board ID 與 SKU ID

Board ID 或 SKU ID 可由 resistor straps、EEPROM 或 CPLD NVM 提供. OpenBMC 可依 ID 選擇 Entity Manager config、inventory 名稱或功能組合.

需要確認:

- ID sampling 在哪個 reset 時刻發生.
- Runtime 是否會改變.
- 未接或非法組合如何處理.
- ID 與 BOM、board revision 的對照表.
- BMC 重開後是否讀到相同結果.

### 6.9.3 Manufacturing 與 Security Strap

Manufacturing mode、debug enable、secure boot、JTAG enable 等 strap 涉及安全邊界. 一般 service 不應有權任意修改; 文件需記錄誰可讀、誰可改、變更何時生效, 以及如何留下 audit log.

## 6.10 LED 與 Button

### 6.10.1 LED Owner

LED 可能由 CPLD 自主控制, 也可能由 BMC request 控制. 若兩者都能控制, 需要明確 priority.

```mermaid
flowchart TB
    A["Critical hardware fault"] --> B["BMC fault indication"]
    B --> C["User identify request"]
    C --> D["Normal power state"]
```

實際 priority 依平台需求決定, 但 CPLD register、OpenBMC LED group 與 Redfish 狀態要一致.

### 6.10.2 Button

Button 通常需要:

- Debounce.
- Short press / long press 判斷.
- Pulse width.
- Host power-state gating.
- Event owner.

Power button 可以由 CPLD 直接轉成 host pulse, 也可先通知 BMC 再由 power-control service 決定. 兩種設計的延遲、故障模式與 BMC 無回應時行為不同.

<a id="section-6-11"></a>

## 6.11 Write Protect 與 Flash Mux

CPLD 常控制:

- BIOS flash write protect.
- BMC flash write protect.
- CPLD image write protect.
- BMC / host / factory programmer flash mux.
- Recovery strap.

這些 bit 不是一般 GPIO. 錯誤寫入可能造成無法開機、更新失敗或安全保護失效.

### 6.11.1 Write Protect 需要記錄什麼

- Active level.
- AC on default.
- 誰能解除.
- 解除條件與有效時間.
- Host / BMC power-state requirement.
- Update 完成後如何恢復.
- Failure 時是否自動回到 protected state.
- 是否有 hardware strap 或 physical presence requirement.

### 6.11.2 Flash Mux Interlock

Flash mux 必須避免兩個 owners 同時驅動同一顆 flash. 切換流程可能需要:

1. 停止目前 owner 的 controller.
2. 確認 chip select 為 inactive.
3. Assert reset 或 isolation.
4. 切換 mux.
5. 等待訊號穩定.
6. 啟用新的 owner.

<a id="section-6-12"></a>

## 6.12 CPLD / FPGA Firmware Update

CPLD / FPGA update 可能影響 power、reset、flash 與 recovery path, 因此需要比一般 service update 更嚴格的前置檢查.

### 6.12.1 Update 前

- 讀取 device ID、image version 與 board revision.
- 確認 package 與目標板相符.
- 保存完整 register dump.
- 確認 host / slot power state.
- 確認 AC 與 standby rail 穩定.
- 驗證 checksum / signature.
- 確認 write protect 與授權.
- 確認 recovery tool 和實體接點可用.

### 6.12.2 Update 中

- 記錄寫入進度.
- 不允許未管理的 BMC reboot 或 service restart.
- 對 AC loss、tool timeout 與 verify failure 有明確結果.
- 不在 image 尚未完成時套用會破壞現有控制邏輯的 reset.

### 6.12.3 Update 後

- Readback / verify.
- Reset CPLD 或依要求 AC cycle.
- 讀回 image version.
- 檢查 power state 與 safe defaults.
- 驗證 presence、LED、reset、power sequence 與 write protect.
- 保存 before / after dump.

### 6.12.4 Recovery

Recovery 可能使用:

- Golden image.
- Dual configuration image.
- JTAG programmer.
- Factory fixture.
- Manual strap.
- External SPI programmer.

Recovery path 必須在正式更新流程前實測, 而不是更新失敗後才第一次嘗試.

<a id="section-6-13"></a>

## 6.13 Linux 與 OpenBMC 整合

### 6.13.1 Device Tree

I2C CPLD 範例:

```dts
&i2c8 {
    status = "okay";
    bus-frequency = <400000>;

    cpld@30 {
        compatible = "vendor,platform-cpld";
        reg = <0x30>;
        reset-gpios = <&gpio0 12 GPIO_ACTIVE_LOW>;
        interrupt-parent = <&gpio0>;
        interrupts = <13 IRQ_TYPE_LEVEL_LOW>;
    };
};
```

需確認:

- `reg` 使用 7-bit I2C address.
- Reset CPLD 是否會影響 host power.
- Interrupt 是 level 還是 edge.
- Interrupt status 如何 clear.
- Driver 與 register map version 是否相容.

### 6.13.2 Kernel Driver

較完整的 CPLD register map 可使用 regmap-based driver, 並依功能註冊:

- GPIO controller.
- Hwmon.
- LED class.
- Reset controller.
- NVMEM / board ID.
- Sysfs 或 character device.

不適合被簡化成 GPIO 的 register, 例如 read-clear fault、pulse command、multi-bit state 或帶副作用的 control, 應保留明確語意.

### 6.13.3 OpenBMC Service

| CPLD 資料 | OpenBMC 對應 |
|---|---|
| Version | Software / inventory version |
| Presence | Inventory `Present` |
| Fault latch | Logging / EventLog / functional state |
| Power state | Host / chassis state |
| LED mode | LED group / IndicatorLED |
| Board ID | Entity Manager probe / platform selection |
| Write protect | Update precheck / audit |

Driver 提供可靠的硬體介面; OpenBMC service 負責名稱、關聯、事件與控制流程.

## 6.14 FPGA Configuration

若 FPGA bitstream 由 BMC 載入, 需要記錄:

- Configuration source.
- Bitstream format.
- Load timing.
- `DONE`、`INIT_B`、`PROGRAM_B` 等狀態.
- Bitstream version.
- Load failure 對 host power 的影響.
- Golden image 或 JTAG recovery.

Linux FPGA Manager framework 可管理部分 FPGA configuration flow, 但實際支援方式取決於 FPGA family、configuration interface 與平台 driver.

<a id="section-6-15"></a>

## 6.15 Target 端排查流程

### 6.15.1 確認裝置與版本

```bash
$ i2cdetect -l
$ ls -l /sys/bus/i2c/devices

$ dmesg | grep -Ei 'cpld|fpga|regmap|fault|power|reset'
```

再使用平台核准的工具讀取 device ID、image version 與 register map version.

### 6.15.2 確認 Raw、Latch 與 State

至少同時保存:

- Raw inputs.
- Debounced status.
- Fault latch.
- Control requests.
- State machine state.
- Derived outputs.

只保存其中一組, 通常不足以判斷問題發生在哪一層.

### 6.15.3 確認 OpenBMC

```bash
$ systemctl --failed
$ journalctl -b --no-pager | \
$ grep -Ei 'cpld|fpga|power|fault|led|presence|write.protect|version'

$ busctl tree xyz.openbmc_project.State.Host 2>/dev/null
$ busctl tree xyz.openbmc_project.State.Chassis 2>/dev/null
$ busctl tree xyz.openbmc_project.Logging 2>/dev/null
```

實際 service name 與 object path 依產品整合而異.

### 6.15.4 波形與 Register 同步

Power / reset 問題應在同一測試中同步保存:

- 示波器或 logic analyzer waveform.
- CPLD state 與 fault registers.
- BMC journal.
- Host / chassis D-Bus state.
- 測試動作與 timestamp.

## 6.16 常見問題與判讀

| 現象 | 優先方向 | 第一輪檢查 |
|---|---|---|
| CPLD I2C address 無回應 | Bus、power、reset、address | Adapter、rail、reset、waveform |
| Version 讀值錯誤 | Width、endianness、bank、image | Register map、tool version |
| Fault latch 清除後又出現 | Raw fault 尚未解除 | Raw status、PMBus、waveform |
| Clear 後無法分析 | Clear 前未保存 | Tool / service 流程 |
| Host power-on timeout | State machine 等不到 input | State、PGOOD、fault latch |
| BMC reboot 造成 host 掉電 | Default 或 reset routing | CPLD state、GPIO default、waveform |
| Presence 反相 | Active level 或 raw/debounce 混淆 | Raw bit、debounced bit、schematic |
| LED 與 Redfish 不一致 | Owner、priority、polarity | LED register、D-Bus、waveform |
| Board ID 錯誤 | Strap、decode、BOM | Raw ID、schematic、inventory |
| BIOS update 失敗 | WP、flash mux、owner conflict | WP register、mux state、update log |
| CPLD update 後無法開機 | Image / default / sequence regression | JTAG recovery、version、waveform |
| Watchdog reset target 錯誤 | Reset mux / policy | Reset reason、CPLD register、scope |

<a id="section-6-17"></a>

## 6.17 Debug Log 收集

以下腳本只收集一般系統資訊; CPLD register dump 需使用平台核准的唯讀工具另外加入.

```bash
#!/bin/sh

OUT=/tmp/cpld-fpga-debug
mkdir -p "$OUT"

cat /etc/os-release > "$OUT/os-release.txt" 2>&1
uname -a > "$OUT/uname.txt"
cat /proc/cmdline > "$OUT/proc-cmdline.txt"

dmesg -T > "$OUT/dmesg.txt"
journalctl -b --no-pager > "$OUT/journal.txt" 2>&1
journalctl -b -1 --no-pager > "$OUT/journal-previous.txt" 2>&1
systemctl --failed > "$OUT/systemctl-failed.txt" 2>&1

i2cdetect -l > "$OUT/i2cdetect-l.txt" 2>&1
ls -l /sys/bus/i2c/devices > "$OUT/i2c-devices.txt" 2>&1
find /sys/bus/i2c/devices -maxdepth 3 -print > "$OUT/i2c-tree.txt" 2>&1

mount | grep debugfs >/dev/null 2>&1 || \
    mount -t debugfs debugfs /sys/kernel/debug

cat /sys/kernel/debug/gpio > "$OUT/gpio.txt" 2>&1
find /sys/kernel/debug/pinctrl -maxdepth 3 -type f -print \
    > "$OUT/pinctrl.txt" 2>&1

busctl tree xyz.openbmc_project.State.Host \
    > "$OUT/host-state.txt" 2>&1
busctl tree xyz.openbmc_project.State.Chassis \
    > "$OUT/chassis-state.txt" 2>&1
busctl tree xyz.openbmc_project.Logging \
    > "$OUT/logging.txt" 2>&1

# 依平台加入唯讀工具，例如：
# cpldtool version > "$OUT/cpld-version.txt" 2>&1
# cpldtool dump --safe > "$OUT/cpld-dump.txt" 2>&1
# fpgautil status > "$OUT/fpga-status.txt" 2>&1

tar czf "/tmp/cpld-fpga-debug-$(date +%Y%m%d-%H%M%S).tar.gz" \
    -C /tmp cpld-fpga-debug
```

通用腳本不應自動:

- Clear fault latch.
- 發出 power / reset request.
- 切換 flash mux.
- 解除 write protect.
- Reset CPLD / FPGA.
- 寫入 scratch 以外的 register.

## 6.18 Bring-up 順序

1. 確認 CPLD / FPGA part number、board location 與 power rail.
2. 取得對應 image version 的 register map.
3. 確認 access bus、address、width、endianness 與 bank.
4. 先讀取 ID、version 與無副作用 status.
5. 使用 scratch register 驗證基本 read / write, 若平台提供.
6. 將 raw inputs 與 schematic / waveform 對照.
7. 驗證 debounce、presence、board ID 與 straps.
8. 驗證非破壞性的 LED 與 mux 狀態.
9. 驗證 power sequence 的每個 state、timeout 與 fault latch.
10. 驗證 reset source、target 與 pulse width.
11. 建立 fault 保存、clear 與 event 流程.
12. 驗證 write protect 與 flash mux 的授權流程.
13. 驗證 OpenBMC inventory、state、LED 與 logging.
14. 最後執行 firmware update、interruption 與 recovery 測試.
15. 進行 BMC reboot、host power cycle、AC cycle 與 watchdog regression.

## 6.19 平台實測紀錄表

| 項目 | 來源 / 指令 | 實測值 | 備註 |
|---|---|---|---|
| Device / part number | BOM / schematic | [待填] | CPLD / FPGA / MCU |
| Board location | Layout | [待填] | Silkscreen |
| Image version | Version register | [待填] | 對應 build commit |
| Register map version | Design document | [待填] | 必須和 image 對齊 |
| Access path | I2C / MMIO / LPC / UART | [待填] | Bus / address |
| Register format | Width / endian / bank | [待填] | Tool 設定 |
| Reset source | Schematic | [待填] | POR / BMC / host |
| Power rail | Schematic | [待填] | Standby / main |
| Raw status | Register dump | [待填] | 與 pin 波形對照 |
| Fault latch | Register dump | [待填] | Clear rule |
| Power state | State register | [待填] | State / timeout |
| Reset routing | Register / waveform | [待填] | Source / target |
| Board / SKU ID | Strap register | [待填] | BOM 對照 |
| Write protect | Security register | [待填] | Default / owner |
| OpenBMC mapping | D-Bus | [待填] | Inventory / state / log |
| Update method | Platform process | [待填] | Verify / rollback |
| Recovery method | JTAG / golden image | [待填] | 已實測 |

<a id="section-6-20"></a>

## 6.20 驗收 Checklist

基本資料:

- [ ] Part number、位置、image version 與 register map version 已記錄.
- [ ] Access bus、address、width、endianness、bank 與 auto-increment 已確認.
- [ ] 每個 bit 的 access、default、active level、owner 與副作用已記錄.
- [ ] Raw、debounced、latched、request 與 derived state 已區分.

Power、Reset 與 Fault:

- [ ] Power state machine 的 state、input、output、timeout 與 fault 已實測.
- [ ] Reset source、target、pulse width 與 gating 已量測.
- [ ] BMC reboot 不會造成非預期 host power / reset.
- [ ] Fault clear 前會保存 register、PMBus、journal 與 waveform.
- [ ] Read-clear、W1C、Pulse 與 Sticky register 不會被一般輪詢破壞.

平台功能:

- [ ] Presence、board ID、SKU ID 與 OpenBMC inventory 一致.
- [ ] LED owner、priority、pattern 與 Redfish 狀態一致.
- [ ] Write protect 與 flash mux 具有安全 default 與授權流程.
- [ ] Kernel driver 與 userspace tool 不會衝突存取 register.
- [ ] OpenBMC state、logging 與 CPLD status 一致.

Update 與 Recovery:

- [ ] Image package 會檢查 device ID 與 board revision.
- [ ] Update 前後會保存 version 與 register dump.
- [ ] Verify、interruption、rollback 與 recovery 已測試.
- [ ] AC cycle、BMC reboot、host cycle 與 watchdog regression 已完成.

## 6.21 本章重點

1. CPLD 常負責必須在 BMC Linux 啟動前運作的 power、reset 與保護邏輯.
2. FPGA 通常提供更多邏輯與高速 I/O, 並需要管理 bitstream 載入與 recovery.
3. Raw input、debounced state、fault latch、control request 與 derived output 是不同狀態.
4. Register map 必須記錄 access、default、reset domain、owner、clear rule 與 side effect.
5. W1C、read-clear、pulse 與 security register 不能用一般輪詢或未知工具任意存取.
6. Power sequence 應以 state machine、等待訊號與 timeout 排查, 不只看 power request bit.
7. Fault clear 前要先保存 raw status、latch、PMBus status、journal 與 waveform.
8. BMC reboot、host reset、watchdog reset 與 AC cycle 的影響範圍必須分開驗證.
9. Write protect、flash mux 與 firmware update 屬於安全與 recovery 邊界.
10. OpenBMC 對外狀態應能回溯到 CPLD register、實體訊號與明確的 owner.

## 6.22 本章參考資料

- Linux kernel documentation - Regmap API: https://docs.kernel.org/driver-api/regmap.html
- Linux kernel documentation - GPIO Mappings: https://docs.kernel.org/driver-api/gpio/board.html
- Linux kernel documentation - I2C Sysfs: https://docs.kernel.org/i2c/i2c-sysfs.html
- Linux kernel documentation - FPGA Manager Framework: https://docs.kernel.org/driver-api/fpga/fpga-mgr.html
- OpenBMC entity-manager: https://github.com/openbmc/entity-manager
- OpenBMC phosphor-led-manager: https://github.com/openbmc/phosphor-led-manager
- OpenBMC phosphor-state-manager: https://github.com/openbmc/phosphor-state-manager
- OpenBMC phosphor-logging: https://github.com/openbmc/phosphor-logging
