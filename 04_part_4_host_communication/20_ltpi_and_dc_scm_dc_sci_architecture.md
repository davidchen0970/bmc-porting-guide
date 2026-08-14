## 20. LTPI 與 DC-SCM / DC-SCI 架構

DC-SCM 將 BMC、信任根、平台控制與部分管理介面從 Host Processor Module（HPM）抽離成可替換模組；DC-SCI 則定義 SCM 與 HPM 之間的模組介面。LTPI 使用 LVDS 連線，在較少接腳上承載 GPIO、SMBus / I2C、UART 與資料通道，是 DC-SCM 2.x 架構中重要的低速管理介面聚合機制。

本章建立 LTPI 平台的硬體與韌體介面契約，說明 link initialization、capability negotiation、GPIO latency、遠端 I2C、UART、OEM data channel、CPLD / FPGA、OpenBMC、Device Tree、電源重置、安全、除錯與驗收方法。

### 20.1 DC-SCM、DC-SCI、HPM 與 LTPI 的角色

#### 20.1.1 DC-SCM

DC-SCM（Data Center Secure Control Module）通常承載：
- BMC SoC、BMC boot flash 與記憶體。
- Platform Root of Trust（PRoT）或其他安全元件。
- SCM 端 CPLD / FPGA。
- 管理網路、外部除錯或前面板相關介面，依產品設計。
- Host firmware 管理與平台控制所需的 sideband interface。

模組化的目的不是讓所有平台自動相容，而是建立共同的機構、電氣與介面邊界。實際產品仍需確認 DC-SCM、DC-SCI、LTPI 與 endpoint firmware 的版本組合。

#### 20.1.2 HPM

HPM（Host Processor Module）是伺服器主板或 Host 平台，通常包含：
- Host CPU、memory、chipset 與 PCIe topology。
- HPM 端 CPLD / FPGA。
- Power sequence、reset、VR、thermal 與 platform fault signals。
- 由 SCM 管理的遠端 I2C / SMBus devices。

#### 20.1.3 DC-SCI

DC-SCI（Data Center Secure Control Interface）是 SCM 與 HPM 之間的連接介面。它不只等於 LTPI，還可能包含 power、reset、SPI、JTAG、PCIe、USB、video、network 或其他 sideband signals。LTPI 是其中負責聚合多種低速訊號的邏輯連線。

#### 20.1.4 LTPI

LTPI（LVDS Tunneling Protocol and Interface）在 SCM endpoint 與 HPM endpoint 之間傳輸多個邏輯 channel。典型架構如下：

```text
+----------------------------------+
| DC-SCM                           |
| BMC <-> SCM CPLD / FPGA          |
|          | GPIO / I2C / UART /   |
|          | CSR / Data Channel    |
+----------|-----------------------+
           | LTPI over LVDS
+----------|-----------------------+
| HPM      |                       |
| HPM CPLD / FPGA <-> Host Platform|
+----------------------------------+
```

BMC 通常不直接處理每一個 LTPI frame。frame encoding、training、channel aggregation 與即時 GPIO 多由 SoC hardware block、CPLD 或 FPGA 處理，OpenBMC 則透過暫存器、GPIO controller、I2C adapter、UART 或專用 service 使用它們。

### 20.2 LTPI 與 SGPIO 的差異

DC-SCM 1.0 架構使用兩組 SGPIO。DC-SCM 2.0 導入 LTPI，以 LVDS links 取代原先的 SGPIO 路徑，並將功能擴充至 GPIO 以外的介面。

<table>
<tr><th>項目</th><th>SGPIO</th><th>LTPI</th></tr>
<tr><td>主要用途</td><td>序列化 GPIO</td><td>多種低速介面聚合</td></tr>
<tr><td>電氣介面</td><td>通常為單端訊號</td><td>LVDS / 規範允許的差動實作</td></tr>
<tr><td>邏輯通道</td><td>以 GPIO 為主</td><td>LL GPIO、NL GPIO、I2C / SMBus、UART、Data / OEM</td></tr>
<tr><td>初始化</td><td>平台化</td><td>定義 discovery、advertise、negotiation、configuration</td></tr>
<tr><td>擴充性</td><td>受 GPIO 數與接腳限制</td><td>可協商速率、能力與 channel</td></tr>
<tr><td>故障模式</td><td>單訊號或 shift chain 問題</td><td>physical、training、link、channel 與 software 多層問題</td></tr>
</table>

LTPI 不是 SGPIO 的透明電氣替代品。轉換平台時必須重新定義 polarity、default、latency、reset ownership、link-down behavior 與 recovery policy。

### 20.3 LTPI 介面契約

每一個 tunneled function 都應建立契約：

<table>
<tr><th>欄位</th><th>說明</th></tr>
<tr><td>Signal / Channel</td><td>GPIO、I2C bus、UART、CSR 或 OEM message 名稱</td></tr>
<tr><td>Producer / Initiator</td><td>由 SCM、HPM、BMC、Host 或 endpoint 哪一方發起</td></tr>
<tr><td>Consumer / Target</td><td>接收端與實際使用者</td></tr>
<tr><td>Direction</td><td>SCM-to-HPM、HPM-to-SCM 或雙向</td></tr>
<tr><td>Polarity / Format</td><td>Active high / low、bit width、endianness、message format</td></tr>
<tr><td>Latency Class</td><td>LL GPIO、NL GPIO 或非即時 channel</td></tr>
<tr><td>Validity</td><td>何時開始可採信，link up 前後的行為</td></tr>
<tr><td>Power Domain</td><td>STBY、Host on 或其他 platform state</td></tr>
<tr><td>Reset Behavior</td><td>BMC、SCM endpoint、HPM endpoint、Host reset 的影響</td></tr>
<tr><td>Error Handling</td><td>timeout、stale、retry、fallback、safe state 與 event</td></tr>
<tr><td>Versioning</td><td>LTPI、endpoint RTL、register map 與 OpenBMC 相容性</td></tr>
<tr><td>Security</td><td>權限、完整性、debug access 與 firmware trust</td></tr>
</table>

### 20.4 Link 初始化與狀態機

典型流程如下：

```text
Power / Reset Stable
        ↓
Physical Detection / Clock Ready
        ↓
Link Synchronization
        ↓
Discovery / Advertise
        ↓
Capability Exchange
        ↓
Rate and Channel Negotiation
        ↓
Configuration Accept / Echo
        ↓
Operational
        ↓
Error Detection -> Recovery / Retrain / Safe State
```

平台文件至少要定義：
- Controller 與 target endpoint 的角色。
- 哪一端啟動 auto configuration。
- mandatory fallback rate 與可宣告速率。
- advertise、configuration 與 data channel timeout。
- link operational 的判定條件。
- training 失敗的 retry 次數與間隔。
- 何時上報 BMC event，何時進入 safe state。
- endpoint firmware 不相容時是否允許降級。

不能只用「LVDS clock 存在」代表 link ready。只有完成協商、configuration 被接受，而且所需 channels 已啟用後，tunneled data 才能視為有效。

### 20.5 Capability 與版本管理

Capability 資訊可能包含：
- LTPI protocol revision。
- 支援的 transfer rates 與 SDR / DDR mode。
- LL / NL GPIO 數量。
- I2C / SMBus、UART 與 data channels 數量。
- Default / custom frame 支援。
- Endpoint implementation 或 vendor-specific capability。

建議建立 compatibility matrix：

<table>
<tr><th>SCM RTL</th><th>HPM RTL</th><th>LTPI Revision</th><th>Rate</th><th>Channels</th><th>Result</th></tr>
<tr><td>[待填]</td><td>[待填]</td><td>[待填]</td><td>[待填]</td><td>[待填]</td><td>[待確認]</td></tr>
</table>

韌體更新不能只驗證「能否燒錄」。還要驗證新舊 endpoint 是否能完成 negotiation、是否改變 frame layout、GPIO index、timeout 或 register semantics。

### 20.6 LL GPIO 與 NL GPIO

#### 20.6.1 Low-Latency GPIO

LL GPIO 適合時間敏感或平台關鍵訊號，例如：
- Power control handshake。
- Reset request / acknowledge。
- Processor error、CATERR 或 critical fault indication。
- Interrupt、throttle、VR warning。
- Boot 或 recovery handshake。

LL 不等於 zero latency。平台仍需量測 endpoint sampling、frame scheduling、synchronizer 與接收端輸出延遲。

#### 20.6.2 Normal-Latency GPIO

NL GPIO 適合：
- Presence、ID、service mode。
- Maintenance 或 manufacturing strap status。
- 非緊急 LED / indication。
- 不影響即時 power sequence 的一般狀態。

#### 20.6.3 GPIO Mapping 表

<table>
<tr><th>Index</th><th>Signal</th><th>Direction</th><th>Active</th><th>Class</th><th>Default</th><th>Power</th><th>Owner</th></tr>
<tr><td>[待填]</td><td>[待填]</td><td>[待填]</td><td>[待填]</td><td>LL / NL</td><td>[待填]</td><td>[待填]</td><td>[待填]</td></tr>
</table>

對每個 GPIO 驗證 link down、SCM reset、HPM reset、BMC reboot 與 Host power transition。平台關鍵輸出必須有硬體安全預設，不能依賴 OpenBMC service 在 link failure 後及時補救。

### 20.7 I2C / SMBus Tunneling

LTPI 可將遠端 I2C / SMBus transaction 穿過 SCM 與 HPM endpoint。這不是單純延長 SDA / SCL，而是由 bridge 將 transaction 轉成 LTPI channel data，再在遠端重建 bus operation。

需要定義：
- Controller 與 target 位於哪一側。
- Linux bus number 與穩定的 logical identity。
- 遠端 address map 與 mux topology。
- Clock frequency、clock stretching 與 repeated-start 支援。
- Multi-controller arbitration 是否支援。
- Remote bus power state。
- Transaction timeout、NACK、bus busy 與 bridge error 如何回傳。
- Link loss 時正在進行的 transaction 如何結束。
- Recovery 能否真正控制遠端 SCL / SDA。

Linux 的一般 I2C bus recovery 不一定能直接切換遠端 physical lines。若 endpoint 沒有 remote recovery primitive，BMC 端執行 recovery 可能只重設 local bridge，無法釋放 HPM 上被 target 拉住的 SDA。

### 20.8 UART Tunneling

UART channel 可供：
- Host console / BIOS serial log。
- HPM CPLD 診斷。
- Manufacturing test。
- Recovery console。
- SCM 與 HPM firmware message。

介面契約包含 baud rate、data bits、parity、stop bits、flow control、channel ownership、mux 規則、reset behavior 與存取權限。若 UART 同時提供 Serial over LAN，需定義 local header、BMC tty、SOL client 與 debug mux 的優先權，避免兩個 producer 同時驅動 TX。

### 20.9 Data Channel 與 OEM Channel

Data channel 可在 SCM CPLD 與 HPM CPLD 之間交換暫存器或任意資料。OEM channel 應明確定義：
- Message type、opcode 與 payload length。
- Version 與 backward compatibility。
- Endianness、alignment 與 reserved bits。
- Sequence ID、request / response correlation。
- Timeout、retry、duplicate handling。
- Error codes 與 unsupported response。
- Authentication、authorization 與敏感資料限制。

平台關鍵控制不應依賴未文件化的 magic register 或 OEM payload。若必須使用，應建立公開給 BIOS、BMC、CPLD / FPGA 與 validation 團隊的 interface specification。

### 20.10 CPLD / FPGA 與 SoC LTPI Controller 責任

Endpoint 常負責：
- LVDS PHY、clocking、reset synchronization。
- Encoding / decoding、frame CRC。
- Link training、advertise 與 negotiation。
- Channel aggregation / disaggregation。
- GPIO synchronization 與 safe defaults。
- I2C / SMBus、UART 與 data channel bridge。
- CSR、interrupt、error counters 與 retraining。

建議暴露以下診斷資訊：
- Current link state。
- Local / remote capabilities。
- Negotiated rate、mode 與 frame configuration。
- Enabled channels。
- Training、retraining、CRC、frame、timeout counters。
- Last failure reason 與 timestamp。
- Endpoint RTL / firmware version。
- LL / NL GPIO input、output snapshots。
- I2C / UART / data-channel error status。

### 20.11 OpenBMC 整合分層

建議將實作分成五層：

```text
Redfish / IPMI / Web UI / Event Log
                ↓
OpenBMC D-Bus Services and Policy
                ↓
Linux GPIO / I2C / UART / hwmon Abstractions
                ↓
LTPI Controller or FPGA/CPLD Driver
                ↓
LVDS Link and Remote Endpoint
```

設計原則：
- Link health 與 downstream function health 分開。
- link down 要有單一、明確的 root-cause event。
- downstream sensors 可標示 unavailable，避免每一個 remote device 都生成重複 fault。
- reconfiguration 或 retraining 需與 power-control policy 協調。
- 驅動與 service restart 後，重新取得 negotiated state。
- 不將未驗證的 remote GPIO snapshot 當成即時真值。

目前可找到 ASPEED 平台化的 LTPI bus driver 實作，但是否進入特定 upstream kernel 或產品 branch，必須以實際 BSP 為準。不可把某個 vendor branch 的 register layout 直接視為所有 SoC 的標準 binding。

### 20.12 Device Tree 與 Kernel Binding

概念範例：

```dts
ltpi: ltpi@12000000 {
    compatible = "vendor,soc-ltpi";
    reg = <0x12000000 0x1000>;
    interrupts = <0 120 4>;
    clocks = <&syscon 10>, <&syscon 11>;
    clock-names = "ltpi", "phy";
    resets = <&syscon 8>;
    status = "okay";
};
```

若 endpoint 由外部 FPGA / CPLD 提供，可能改以 I2C、SPI 或 memory-mapped child devices 描述。Production DTS 必須匹配實際 driver binding，不應複製概念節點名稱或 register address。

檢查項目：
- `compatible` 與 driver match table。
- reg、clock、reset、interrupt 與 pinctrl。
- tunneled I2C adapter 是否註冊。
- GPIO chip base 與 line naming。
- UART tty identity。
- interrupt polarity / trigger type。
- runtime PM 與 standby availability。

### 20.13 Power 與 Reset Sequencing

至少驗證：
- AC applied 與 standby rails 建立。
- SCM insertion / presence detection。
- BMC cold boot、warm reset 與 watchdog reset。
- SCM CPLD / FPGA reset。
- HPM CPLD / FPGA reset。
- Host power-on、power-off、warm reset。
- BMC firmware update 與 endpoint update。
- SCM / HPM replacement。

```text
STBY Power Good
      ↓
Endpoint Clock / Reset Stable
      ↓
LTPI Training and Configuration
      ↓
Required Channels Valid
      ↓
BMC Power-Control Service Ready
      ↓
Allow Host Power-On
```

若 link 尚未 operational，平台需決定 Host power-on 是禁止、延遲或在降級模式下允許。與 power / reset 有關的輸出，在 endpoint reset 及 pins high-Z 時也必須保持安全。

### 20.14 BMC Reboot 與 Host Continuity

若產品要求 BMC reboot 時 Host 不中斷，需確認：
- Host power、reset 與 critical GPIO 由 CPLD / FPGA 保持。
- BMC driver remove / probe 不會重設整條 LTPI link。
- link retrain 不會產生短暫 power pulse。
- remote I2C 中斷後 transaction 能正常失敗並重試。
- OpenBMC 恢復後重新同步 GPIO 與 link state。
- link lost event 不因 service restart 重複產生。

### 20.15 安全與信任邊界

SCM 與 HPM 即使在同一機箱內，也不能假設連線天然可信。需檢查：
- SCM / HPM endpoint bitstream 的簽章驗證與 anti-rollback。
- PRoT 對 BMC、CPLD / FPGA 與 Host firmware 的驗證順序。
- Debug port、JTAG、test CSR 與 manufacturing mode 在 production 的限制。
- OEM / data channel payload validation。
- 未授權 register write 是否能改變 power、reset 或 write-protect。
- Link error、endpoint replacement 與 firmware mismatch 是否寫入 audit / event log。
- Secret、key、credential 不進入一般 LTPI diagnostic dump。

### 20.16 Firmware Update 與相容性

更新範圍可能包含：
- BMC image / kernel driver。
- SCM CPLD / FPGA bitstream。
- HPM CPLD / FPGA bitstream。
- LTPI controller firmware 或 microcode。

每次更新需保存：
- Active / backup image。
- LTPI protocol revision。
- Register map revision。
- Channel map revision。
- Rollback policy。
- Host power requirement。
- 更新中斷後的 recovery path。

應測試先升級 SCM、先升級 HPM、只升級其中一端及 downgrade。若兩端不能跨版本互通，更新程序必須提供安全的 staged activation。

### 20.17 分層 Debug 方法

#### 20.17.1 Physical Layer
- Power rails、reference clock、reset。
- LVDS polarity、lane mapping、termination 與 AC coupling。
- Pinmux / strap / OTP configuration。
- Signal integrity、eye、jitter 與 connector continuity。

#### 20.17.2 Link Layer
- Controller / target role。
- Training state 與 state-machine transition。
- Local / remote capabilities。
- Negotiated rate、mode、frame format。
- advertise、configuration、echo timeout。
- CRC / framing error 與 retraining counter。

#### 20.17.3 Channel Layer
- LL / NL GPIO mapping、polarity、default、snapshot。
- I2C address、NACK、stretching、timeout、remote power。
- UART baud、mux、RX / TX direction。
- Data-channel sequence、response 與 error code。

#### 20.17.4 Linux / OpenBMC Layer
- Driver bind、Device Tree、clock、reset、IRQ。
- `/sys/bus/platform/devices`、GPIO、I2C、tty。
- D-Bus object、systemd dependencies、journal。
- Power-control、sensor、inventory 與 event mapping。

### 20.18 Target 端檢查

```sh
# Kernel / Device Tree
dmesg -T | grep -Ei 'ltpi|lvds|gpio|i2c|uart|fpga|cpld'
find /sys/bus/platform/devices -maxdepth 2 -iname '*ltpi*' -print
find /proc/device-tree -iname '*ltpi*' -print 2>/dev/null

# I2C / GPIO / UART
i2cdetect -l
gpioinfo 2>/dev/null | grep -Ei 'ltpi|power|reset|fault'
ls -l /dev/tty* 2>/dev/null | grep -Ei 'S|USB|AMA'

# OpenBMC services and logs
systemctl --failed
systemctl --type=service | grep -Ei 'power|gpio|sensor|fpga|cpld|ltpi'
journalctl -b --no-pager | grep -Ei 'ltpi|link lost|retrain|crc|timeout'
```

若 endpoint 提供 `devmem` 或 vendor utility 讀取 CSR，必須使用經核准的 register map。避免對未知暫存器寫值，尤其是 reset、power、flash ownership 與 write-protect controls。

### 20.19 常見問題與判讀

<table>
<tr><th>現象</th><th>優先方向</th><th>第一輪檢查</th></tr>
<tr><td>Link 完全不起來</td><td>Power / reset / PHY</td><td>Clock、reset、polarity、lane、pinmux</td></tr>
<tr><td>停在 advertise / config</td><td>Capability mismatch</td><td>Protocol revision、rate、frame、endpoint RTL</td></tr>
<tr><td>反覆 retrain</td><td>Signal integrity / timeout</td><td>CRC、clock、power、watchdog、timeout</td></tr>
<tr><td>GPIO 值相反</td><td>Polarity / mapping</td><td>Index、active level、SCM / HPM table</td></tr>
<tr><td>GPIO 偶發舊值</td><td>Validity / synchronization</td><td>Link state、snapshot、reset domain</td></tr>
<tr><td>Remote I2C 全失敗</td><td>Channel / remote power</td><td>Link、adapter、bus power、ownership</td></tr>
<tr><td>特定 I2C device 失敗</td><td>Address / timing</td><td>Mux、stretching、repeated start、timeout</td></tr>
<tr><td>UART 亂碼</td><td>Format / clock</td><td>Baud、parity、mux、endpoint clock</td></tr>
<tr><td>BMC reboot 造成 Host reset</td><td>Safe default / ownership</td><td>CPLD hold、GPIO high-Z、driver probe</td></tr>
<tr><td>Redfish 顯示大量 sensor error</td><td>Root-cause aggregation</td><td>LTPI link health 與 stale policy</td></tr>
<tr><td>更新一端後無法連線</td><td>Firmware compatibility</td><td>LTPI revision、register / frame map</td></tr>
</table>

### 20.20 Debug Log 收集

```sh
#!/bin/sh
OUT=/tmp/ltpi-debug
mkdir -p "$OUT"

cat /etc/os-release > "$OUT/os-release.txt" 2>&1
uname -a > "$OUT/uname.txt"
cat /proc/cmdline > "$OUT/proc-cmdline.txt"
dmesg -T > "$OUT/dmesg.txt" 2>&1
journalctl -b --no-pager > "$OUT/journal.txt" 2>&1
systemctl --failed > "$OUT/systemctl-failed.txt" 2>&1
systemctl --type=service > "$OUT/services.txt" 2>&1
find /sys/bus/platform/devices -maxdepth 2 -iname '*ltpi*' -print \
    > "$OUT/ltpi-devices.txt" 2>&1
find /proc/device-tree -iname '*ltpi*' -print \
    > "$OUT/ltpi-device-tree.txt" 2>&1
i2cdetect -l > "$OUT/i2c-adapters.txt" 2>&1
gpioinfo > "$OUT/gpioinfo.txt" 2>&1
journalctl -b --no-pager | grep -Ei \
    'ltpi|lvds|retrain|link lost|crc|i2c|uart|fpga|cpld' \
    > "$OUT/related-journal.txt" 2>&1

# 依核准的工具另行收集 endpoint CSR。
# 不收集 passwords、keys、tokens 或未遮罩的敏感 OEM payload。

tar czf "/tmp/ltpi-debug-$(date +%Y%m%d-%H%M%S).tar.gz" \
    -C /tmp ltpi-debug
```

### 20.21 Bring-up 順序

1. 確認 DC-SCM、DC-SCI、LTPI、SCM RTL 與 HPM RTL 版本。
2. 建立 SCM / HPM block diagram 與 ownership table。
3. 驗證 standby power、clock、reset、LVDS lanes 與 polarity。
4. 讀取 local / remote capabilities，確認 negotiated rate。
5. 驗證 link loss、retrain、timeout 與 error counters。
6. 逐一驗證 LL GPIO，再驗證 NL GPIO。
7. 驗證每條 remote I2C / SMBus bus、mux、address 與 power state。
8. 驗證 UART format、mux、console 與 SOL interaction。
9. 驗證 data / OEM channel 的 version、timeout 與 error handling。
10. 整合 Linux driver、Device Tree、GPIO、I2C 與 UART abstractions。
11. 整合 OpenBMC service、event、health 與 Redfish / IPMI exposure。
12. 測試 BMC reset、Host reset、endpoint reset、AC cycle 與 link fault injection。
13. 測試 SCM / HPM endpoint update、rollback 與跨版本相容性。
14. 保存 logs、register dump、waveform、版本與驗收結果。

### 20.22 平台實測紀錄表

<table>
<tr><th>Feature</th><th>SCM Endpoint</th><th>HPM Endpoint</th><th>Linux Interface</th><th>Power State</th><th>Failure Policy</th><th>Result</th></tr>
<tr><td>LTPI Link</td><td>[待填]</td><td>[待填]</td><td>[待填]</td><td>STBY</td><td>[待填]</td><td>[待確認]</td></tr>
<tr><td>LL GPIO</td><td>[待填]</td><td>[待填]</td><td>GPIO</td><td>[待填]</td><td>Safe default</td><td>[待確認]</td></tr>
<tr><td>NL GPIO</td><td>[待填]</td><td>[待填]</td><td>GPIO</td><td>[待填]</td><td>Stale / unknown</td><td>[待確認]</td></tr>
<tr><td>I2C / SMBus</td><td>[待填]</td><td>[待填]</td><td>I2C adapter</td><td>[待填]</td><td>Timeout / recovery</td><td>[待確認]</td></tr>
<tr><td>UART</td><td>[待填]</td><td>[待填]</td><td>tty</td><td>[待填]</td><td>Disconnect / retry</td><td>[待確認]</td></tr>
<tr><td>Data / OEM</td><td>[待填]</td><td>[待填]</td><td>Driver / D-Bus</td><td>[待填]</td><td>Error response</td><td>[待確認]</td></tr>
</table>

### 20.23 驗收 Checklist

介面與版本：
- 每個 channel 都有 producer、consumer、direction、format 與 version。
- DC-SCM、DC-SCI、LTPI 與 endpoint compatibility matrix 已建立。
- GPIO map、frame map 與 register map 受版本控制。

Link：
- Cold boot、warm reset 與 AC cycle 均可穩定進入 operational。
- Negotiated rate、mode、capability 與 enabled channels 正確。
- Link loss、CRC error、timeout 與 retrain 有明確診斷。
- 不相容 endpoint 可安全失敗或降級。

Channels：
- LL / NL GPIO polarity、latency、default 與 reset behavior 已量測。
- I2C / SMBus 的 NACK、stretching、timeout、remote power 與 recovery 已測試。
- UART console、mux、SOL 與 reset behavior 已驗證。
- Data / OEM channel 有版本、correlation、timeout 與驗證規則。

OpenBMC：
- Driver、Device Tree、clock、reset 與 interrupt 正確。
- GPIO、I2C、UART 與專用 D-Bus service identity 穩定。
- LTPI link health 能獨立呈現與記錄。
- Link down 時 downstream resource 使用 unavailable / stale policy，避免事件風暴。

Power、Recovery 與 Security：
- Link unavailable 時 critical outputs 保持安全狀態。
- BMC reboot 不造成非預期 Host reset 或 power transition。
- SCM / HPM endpoint 分別 reset 後可以恢復。
- Firmware update、rollback 與 interrupted update 已測試。
- Endpoint bitstream、debug access 與 OEM channels 符合安全政策。
- 一般 debug bundle 不包含 credential、key 或敏感 payload。

### 20.24 本章重點

- DC-SCM 是模組化管理與安全控制板，DC-SCI 是 SCM 與 HPM 的介面，LTPI 是其中用於低速 sideband 聚合的 LVDS tunneling protocol。
- LTPI 可承載 LL / NL GPIO、I2C / SMBus、UART 與 data / OEM channels，不是 SGPIO 的透明替代品。
- Link ready 必須以 training、capability negotiation、configuration 與 channel activation 完成為準。
- GPIO 必須定義 polarity、latency、default、power domain 與 reset behavior。
- Remote I2C recovery 不一定等同於直接控制遠端 SDA / SCL。
- CPLD / FPGA 或 SoC controller 應提供完整 link state、capability、error counter 與 failure reason。
- OpenBMC 要將 physical link health 與 downstream device failures 分層處理。
- Power-control critical outputs 必須有不依賴軟體的 safe defaults。
- SCM / HPM endpoint firmware 需要 compatibility matrix、staged update 與 rollback policy。
- 除錯應依 Physical、Link、Channel、Linux / OpenBMC 四層進行。

### 20.25 本章參考資料

- OCP DC-SCM LTPI Reference Implementation: https://github.com/opencomputeproject/HWMgmt-Module-DCSCM-LTPI
- OCP Hardware Management Module / DC-SCM Specifications and Designs: https://www.opencompute.org/w/index.php?title=Server/MHS/DC-SCM-Specs-and-Designs
- Lattice DC-SCM LTPI IP Core: https://www.latticesemi.com/en/Products/DesignSoftwareAndIP/IntellectualProperty/IPCore/IPCores05/DC-SCM-LVDS-Tunneling-Protocol-and-Interface-IP-Core
- Lattice LTPI Introduction and Revision History: https://radiantip.latticesemi.com/IP_Repository/extract/latticesemi.com_dcscm_ltpi_1.4.2/doc/introduction.html
- OpenBMC Project: https://github.com/openbmc/openbmc
- ASPEED LTPI Platform Driver Example: https://github.com/ocp-hm-openbmc-opf-ami/linux/blob/dev-6.1-intel/drivers/bus/aspeed-ltpi.c

> 注意：LTPI revision、DC-SCM / DC-SCI 規範與 vendor IP 會更新。實際產品設計應以專案鎖定的正式規範、原理圖、endpoint RTL、register specification 與 BSP 為準。
