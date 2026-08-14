# 4. Reset、Clock 與 Power Domain

本章整理 BMC 平台中 reset、clock、power rail、regulator、power domain 與 ready signal 的共用設計模式與排查方法. 這一章和第 1 章 Boot Flow、第 3 章 Pinmux / GPIO、第 5 章周邊匯流排、第 28 章 Power Control 關係很密切; 差異在於本章聚焦「硬體 domain 是否已具備讓 device probe、bus transaction、host power transition 正常進行的前置條件」.

Reset / Clock / Power Domain 問題常呈現為: driver probe deferred、I2C / SPI / eMMC / MAC 無回應、PHY link 不起、host power sequence timeout、BMC reboot 影響 host、watchdog reset 後狀態不一致、周邊偶發消失. 排查時應同時檢查相關訊號, 需要同時把 rail、clock、reset、pinmux、driver binding、service policy、CPLD state、reset reason 串起來看.

## 適用範圍

本章涵蓋 BMC 平台中的 reset、clock、power rail、regulator、power domain、ready signal、相關時序量測, 以及 kernel 與 userspace 的 dependency 排查.

## 適用讀者

- 負責 BMC 硬體 bring-up、Linux kernel、Device Tree、OpenBMC power control 或平台驗證的人員.
- 需要排查 device probe、bus transaction、host power transition、reset reason 或 power sequence 問題的人員.

## 快速導覽

- [基本觀念與依賴關係](#41-基本觀念)
- [Reset 類型與影響範圍](#42-reset-類型與影響範圍)
- [Clock 類型與檢查重點](#43-clock-類型與檢查重點)
- [Power rail、regulator 與 power domain](#44-power-railregulator-與-power-domain)
- [DTS 範本](#45-dts-範本resetclockregulatorpower-domain)
- [Timing 與量測欄位](#47-timing-與量測欄位)
- [Reset reason 與 fault latch](#48-reset-reason-與-fault-latch)
- [Device probe deferred 排查](#410-device-probe-deferred-與-dependency-排查)
- [Bring-up 順序](#412-bring-up-順序)
- [驗收 Checklist](#414-驗收-checklist)

### 4.0 先建立 Reset、Clock 與 Power Domain 的理解模型

這三個主題不應分開背誦。對多數數位裝置而言，能被軟體正常存取之前，必須先滿足一組具有順序的前置條件：

```text
供電存在且進入規格
        ↓
參考 clock 存在，必要 PLL 已 lock
        ↓
Reset 保持足夠時間後解除
        ↓
Pinmux、bus owner 與 isolation 狀態正確
        ↓
Driver 才能讀寫 register 或進行 bus transaction
        ↓
Userspace 才能建立 D-Bus object 與管理功能
```

這條路徑不是所有裝置都完全相同。例如有些 device 要先有 clock 才能解除 reset，有些外部 PHY 要先解除 reset 才會輸出 ready，有些 Host sideband clock 只有 Host 進入特定 power state 後才存在。因此文件真正需要保存的不是一條固定規則，而是每個 device 或 domain 的依賴圖與時序條件。

#### 4.0.1 三者分別回答什麼問題

- **Power / rail**：電路現在是否有正確且穩定的電壓與電流能力？
- **Clock**：數位邏輯是否有時間基準，clock source、parent、divider 與 gate 是否正確？
- **Reset**：邏輯目前是否被強制維持在初始狀態，何時允許開始工作？
- **Ready signal**：硬體是否已確認前置條件成立，可讓下一階段繼續？
- **Power domain**：哪些硬體共享供電、clock、reset、isolation 或 runtime power control？

其中 `PGOOD`、`PLL_LOCK`、`RESET_N` 與 `LINK_UP` 不能互相替代：

```text
PGOOD = rail 已進入供電規格
PLL_LOCK = PLL 已鎖定參考 clock
RESET_N = reset 是否已解除
LINK_UP = protocol training 或連線已完成
```

看到某一個 ready signal 成立，只能證明它所代表的條件，不代表整個 device 已可由上層使用。

#### 4.0.2 靜態狀態和時序證據

這類問題具有明顯的時間關係。系統 ready 後讀到 rail 正常、clock enabled、reset deasserted，不足以證明開機過程正確。例如 reset 曾在 rail 尚未穩定時短暫解除，稍後再恢復正常，最終靜態狀態仍看不出問題。

因此應同時保存兩類證據：

```text
靜態證據
DTS、register、clk_summary、regulator state、GPIO state、D-Bus state

時序證據
Rail ramp、PGOOD、clock start、PLL lock、reset release、bus transaction、driver log
```

對偶發 probe failure、warm reset 後裝置消失、BMC reboot 影響 Host 等問題，時序證據通常比單一時間點的 register dump更有判斷力。

#### 4.0.3 一個 Device 可用的最小依賴圖

以外部 Ethernet PHY 為例：

```text
3V3_PHY / 1V0_PHY 穩定
        ↓
25 MHz reference clock 存在
        ↓
PHY_RESET_N 維持 datasheet 要求時間
        ↓
PHY_RESET_N 解除
        ↓
等待 strap sampling 與內部初始化
        ↓
MDIO register 可讀
        ↓
MAC driver、PHY driver attach
        ↓
Auto-negotiation / link training
        ↓
Network service 可使用介面
```

如果最後表現為 `no link`，可能停在任何一步。這也是本章的核心方法：不要只從最終現象命名問題，而要沿依賴圖找出第一個未成立的條件。

## 4.1 基本觀念

| 名詞 | 說明 | Bring-up 關注點 |
|----|----|----|
| Reset source | 產生 reset 的來源, 例如 POR IC、CPLD、SoC watchdog、BMC GPIO、host PCH | 需知道觸發條件與 reset 範圍 |
| Reset domain | 同一 reset source 或 reset controller 影響的一組電路 | 避免誤以為 BMC reset 不會影響 host sideband |
| Reset consumer | 被 reset 訊號控制的 device / block | 需知道 active level、minimum pulse width、release timing |
| Clock source | oscillator、crystal、PLL、clock generator、SoC internal clock | 需確認頻率、抖動、enable、source select |
| Clock consumer | 需要 clock 才能工作的 device / peripheral | driver probe 前 clock 是否已存在與已 enable |
| Power rail | 電源 rail, 例如 3V3_AUX、1V8、VCCIO、PHY_AVDD | voltage、ramp、PGOOD、dependency |
| Regulator | Linux 中可描述與管理的供電來源, 例如 fixed-regulator、PMIC regulator | constraints、enable GPIO、always-on、boot-on |
| Power domain | 一組共享供電 / clock / reset dependency 的硬體區塊 | domain on/off 順序與 runtime PM |
| Ready signal | 表示 domain 可用的訊號, 例如 PGOOD、PLL_LOCK、LINK_UP、CHANNEL_READY | 需定義何時可由軟體開始存取 |

常見 dependency:

```mermaid
flowchart TB
    A["Power rail stable"] --> B["Clock source stable / PLL lock"]
    B --> C["Reset deassert"]
    C --> D["Pinmux state applied"]
    D --> E["Driver probe / bus scan"]
    E --> F["Userspace service sees device / D-Bus object ready"]
```

若任一層缺資料, 後面看到的現象可能只是連鎖結果. 例如 I2C device ACK 不到, 方向可能是 I2C pinmux 錯、pull-up rail 未上、expander reset 未釋放、clock gate 未開、bus owner 還在 CPLD / Host、或 power domain 尚未 ready.

#### 4.1.1 Source、Provider、Consumer 與 Domain

Linux 文件常使用 provider 與 consumer 描述資源關係：

```text
Clock provider       產生或分配 clock
Clock consumer       使用該 clock 的 device driver

Reset controller     控制一組 internal reset lines
Reset consumer       要求 assert / deassert reset 的 driver

Regulator provider   提供 rail 或供電控制介面
Regulator consumer   透過 *-supply 宣告其供電需求

Power domain provider 管理 domain 的 on/off
Power domain consumer 掛接到該 domain 的 device
```

Device Tree 的 phandle 用來建立這些關係。Provider node 存在，不代表 provider driver 已 probe；consumer node 寫了 phandle，也不代表對應資源名稱與 driver 期待一致。Deferred probe 經常發生在這兩者尚未完成連接時。

#### 4.1.2 Request、Hardware State 與 Ready State

軟體發出 enable 或 deassert request，不等於硬體已 ready：

```text
Regulator enable request
→ Enable GPIO / PMIC command 已送出
→ Rail 開始 ramp
→ Voltage 進入規格
→ PGOOD asserted
→ Consumer 才能進入下一步
```

同樣地：

```text
Clock prepare/enable
→ Clock gate 被開啟
→ 外部波形或 PLL lock 成立
→ Consumer 才可能正常執行
```

Linux framework 通常能管理 request 與資源引用，但硬體 ready 條件是否被 framework、driver、CPLD 或外部電路確認，必須依平台設計判斷。

#### 4.1.3 Domain 邊界比單一訊號更重要

Reset domain、clock domain 與 power domain 可能不同：

- 兩個 peripheral 可能共用同一個 reset bit，但使用不同 clock gate。
- 一個 PHY 可能具有多路 rail，卻共用單一 reset pin。
- BMC reset 可能不切斷 CPLD standby rail，但會讓 GPIO 進入 reset default。
- Host warm reset 可能切換 `PLTRST_N`，但不關閉 Host standby power。

排查時要畫出 domain 邊界，避免把「某 rail 仍有電」誤認為「該 device 未被 reset」，也避免把「BMC-only reboot」誤認為所有 BMC 管理訊號都會保持不變。

## 4.2 Reset 類型與影響範圍

| Reset 類型 | 常見來源 | 影響範圍 | 常見現象 | 必填資料 |
|----|----|----|----|----|
| POR / Power-on reset | reset IC、CPLD、PMIC | 全板或 BMC domain | AC cycle 後所有狀態回預設 | rail threshold、delay、release 條件 |
| Cold reset | power rail drop 後重新啟動 | BMC / host / full board | register state 全部消失 | 哪些 rail 被關閉、reset reason |
| Warm reset | 不掉主要供電, 只重置邏輯 | SoC / host / peripheral | 部分狀態保留, 問題較難重現 | reset signal、clock 是否持續 |
| BMC-only reset | BMC reset pin、watchdog、software reboot | BMC SoC 與 BMC-managed peripherals | BMC 重啟, host 可能繼續跑 | host sideband 是否受影響 |
| Host reset | PCH / CPU / CPLD / BMC 控制 | host domain | Host 重開, BMC 不重開 | PLTRST / RSMRST / SLP 與 POST 狀態 |
| Peripheral reset | SoC reset controller、GPIO reset | MAC、USB、I2C device、PHY、FPGA | 單一 device probe 或 runtime 失敗 | active level、pulse width、release delay |
| Watchdog reset | SoC watchdog、external watchdog、CPLD | BMC-only 或 full board | reset reason 顯示 watchdog | timeout、feed source、reset target |
| Brownout reset | rail droop、power fault | 受影響電源 domain | 隨機 reboot、flash corruption、device missing | rail waveform、fault latch、PGOOD log |

Reset 排查基本要求:

- 同時保存 reset reason register、CPLD reset latch、power fault latch、UART log、scope / LA waveform.
- 明確標示 reset 範圍: BMC-only、host-only、full board、單一 peripheral.
- 對 BMC reboot / watchdog reset 特別確認 host power 是否受到 side effect.
- 若 reset line 是 open drain 或由多方 wired-OR, 需列出所有可能拉低者.
- 若 reset line 由 CPLD pulse 產生, 需記錄 pulse width、stretch、debounce 與 clear rule.

#### 4.2.1 Assert、Deassert 與 Active Level

Reset 的 `assert` 表示讓 target 進入 reset 狀態，`deassert` 表示解除 reset。它們描述功能狀態，不等同固定的 High 或 Low。

以 `PHY_RESET_N` 為例：

```text
Assert reset   = Physical Low
Deassert reset = Physical High
```

若中間經過 inverter、buffer、CPLD 或 level shifter，BMC-side GPIO level 與 target-side reset pin level可能不同。正式紀錄應分別保存 source-side、target-side physical level 與 functional state。

#### 4.2.2 Reset Pulse 需要哪些條件

一個有效 reset 通常包含：

1. Assert level 達到有效電壓範圍。
2. Assert 持續時間不少於 datasheet 的 minimum pulse width。
3. Reset 期間 rail 與 clock 狀態符合元件規格。
4. Deassert edge 滿足需要的 slew 或同步條件。
5. Deassert 後等待 initialization delay，才開始第一次 transaction。

只看到 reset pin曾變化，不代表 reset 流程有效。若 pulse 太短、rail 尚未進入規格、clock 缺失，或 deassert 後立即讀 register，device 仍可能無回應。

#### 4.2.3 Shared Reset 的風險

Shared reset 代表多個 consumer 受同一條 reset line 或 reset bit 影響。任一 driver 若在 runtime 單獨 assert shared reset，可能同時重置其他正在工作的 device。

需確認：

- Reset controller binding 是否宣告 shared/exclusive 語意。
- Driver 是否只在 probe 階段使用 reset，還會在 runtime error recovery 重置。
- BMC reboot、driver unbind 或 suspend/resume 是否會碰到該 reset。
- Shared consumers 是否有共同的 quiesce 與 recovery 流程。

#### 4.2.4 Reset Tree 與 Wired-OR

一條 target reset 可能同時受多個來源影響：

```text
POR supervisor ─┐
CPLD fault reset ├─ wired-OR / logic gate → TARGET_RESET_N
BMC reset GPIO ─┤
Watchdog output ─┘
```

此時只釋放 BMC GPIO 不一定能解除 target reset，因為其他來源仍可能保持 asserted。排查需要列出所有可能 source，並量測邏輯合成前後的節點。

#### 4.2.5 Warm Reset 為什麼特別容易留下問題

Warm reset 通常不移除主要供電，因此 external device、CPLD latch、clock generator 或 bus state 可能保留。SoC driver重新啟動後，若只假設 device 已回到 power-on default，可能發生：

- Controller reset 了，但 external PHY 沒有 reset。
- Bus master重啟，但 expander仍保留 output。
- Driver重新 probe，但 stale interrupt status仍為 asserted。
- Clock parent 改變，consumer register卻保留舊 divider假設。

因此 warm reset 測試應同時確認哪些狀態會清除、哪些狀態會保留，以及 driver是否有完整重新初始化流程。

## 4.3 Clock 類型與檢查重點

| Clock 類型 | 範例 | 檢查項目 | 常見風險 |
|----|----|----|----|
| Crystal / oscillator | 25MHz、24MHz、32.768kHz | 頻率、振幅、起振時間、load capacitor | BMC 無 early UART、RTC 不準、BootROM 失敗 |
| Reference clock | PCIe REFCLK、RGMII 125MHz、RMII 50MHz | source、enable、jitter、spread spectrum | link 不起、device training fail |
| SoC PLL | CPU / AHB / APB / peripheral PLL | lock 狀態、divider、parent clock | peripheral timeout、baud rate 錯 |
| Peripheral gate | I2C / SPI / UART / MAC clock gate | driver 是否 enable、runtime PM | driver probe deferred、bus 無 clock |
| External clock generator | clock buffer、clock generator IC | I2C config、OE pin、power rail | 多個 device 同時異常 |
| Host-provided clock | eSPI/LPC/PECI/PCIe sideband clock | host power state、PCH readiness | BMC service 在 host off 讀不到訊號 |

Clock bring-up 建議:

- 對 early boot 相關 clock, 例如 main crystal、SPI clock、UART clock, 優先以 scope 量測.
- 對 Linux driver 相關 clock, 檢查 DTS `clocks` / `clock-names`、kernel config、`/sys/kernel/debug/clk/clk_summary`.
- 對 network / PCIe / eSPI 類高速 clock, 確認 clock source、frequency、enable pin、reset timing 與 PHY / PCH dependency.
- 若 baud rate、PWM frequency、fan tach、I2C clock 異常, 除了 driver 設定, 也要檢查 parent clock 與 divider.

常用 clock debug:

```bash
$ mount | grep debugfs || mount -t debugfs debugfs /sys/kernel/debug
$ cat /sys/kernel/debug/clk/clk_summary 2>/dev/null | head -200
$ find /sys/kernel/debug/clk -maxdepth 2 -type f -print 2>/dev/null

$ dmesg | grep -Ei 'clk|clock|pll|osc|refclk|rate'
```

#### 4.3.1 Clock Tree：Source、Parent、Divider 與 Gate

一個 peripheral clock 通常不是直接來自 crystal，而是經過多層 clock tree：

```text
Crystal / oscillator
→ PLL
→ Parent selector / mux
→ Divider
→ Clock gate
→ Peripheral functional clock
```

每一層都可能造成不同現象：

- Source 無波形：整個 clock tree 不工作。
- PLL 未 lock：頻率可能不穩或下游被保持 reset。
- Parent 選錯：頻率來源不符合 driver 假設。
- Divider 錯誤：UART baud、PWM、I2C 或 bus timeout 出現比例性偏差。
- Gate 未開：register access timeout 或 peripheral 完全不動。

`clk_summary` 主要呈現 Linux Common Clock Framework 所知道的 parent、rate 與 enable/prepare count。它不能取代實體波形量測，也不一定涵蓋外部 clock generator 或 Host 提供的 clock。

#### 4.3.2 Clock Enable Count 不等於實體 Clock 一定正常

Clock framework 顯示 enabled，通常表示軟體引用計數與 gate request 已成立；仍可能存在：

- 外部 oscillator 沒供電。
- Clock generator 的 OE pin 未解除。
- PLL 未 lock。
- Pinmux 沒把 clock 輸出到 pad。
- Level translator 或 buffer 未 enable。
- Host-provided clock 因 Host power state 尚未出現。

因此應依 clock 類型選擇證據：internal clock 看 `clk_summary` 與 register，external/reference clock 看 scope，Host clock還要對照 Host power state與 sideband ready。

#### 4.3.3 Clock Accuracy、Jitter 與 Signal Integrity

不同問題需要不同量測能力：

- 一般 scope 可確認 clock 是否存在、頻率是否大致正確及開始時間。
- 高速 reference clock 的 jitter、duty cycle、spread spectrum 與 differential amplitude，可能需要符合頻寬的探棒與專用量測方法。
- Crystal pin 直接量測可能因 probe capacitance 影響振盪，應依硬體量測規範選擇測點。

文件不應只寫「clock 有看到」，還應寫明測點、instrument、probe、頻率、振幅、開始時間與測試條件。

#### 4.3.4 Runtime PM 與 Clock Gate

裝置在 probe 時正常、閒置後無回應，可能涉及 runtime PM：

```text
Driver idle
→ runtime suspend
→ clock gate / power domain off
→ event 或 request 到來
→ runtime resume
→ restore clock / reset / register context
```

若 resume path漏掉 clock、reset或register restore，問題只會在閒置、suspend/resume或特定服務重啟後出現。排查時需區分初次 probe failure 與 runtime resume failure。

## 4.4 Power rail、regulator 與 power domain

Linux regulator framework 用於描述電壓 / 電流 regulator 與其 consumer, 常見能力包含 enable / disable、電壓設定、current limit 與 constraints. BMC 平台中不一定所有 rail 都由 Linux regulator 管理; 有些 rail 只由 CPLD / PMIC / analog circuit 控制, 但仍建議在本章記錄 dependency 與 ready 條件.

| 類型 | Linux 表達 | 適用情境 | 注意事項 |
|----|----|----|----|
| Fixed always-on rail | `regulator-fixed` + `regulator-always-on` | 3V3_AUX、1V8 standby | 仍需量測 ramp 與 ripple |
| GPIO controlled regulator | `regulator-fixed` + enable GPIO | PHY power、sensor power、slot power | active level 與 boot-on 預設需確認 |
| PMIC regulator | PMIC driver + regulator node | SoC core、DDR、peripheral rail | constraints 與 power sequence 必須對齊 datasheet |
| CPLD controlled rail | CPLD register / GPIO / D-Bus | host main rail、slot power | 記錄 register bit、PGOOD、fault latch |
| Host dependent rail | Host power state 控制 | eSPI、PECI、PCIe device | BMC service 需依 host state gating |
| External hot-swap / eFuse | HSC / eFuse driver 或 GPIO fault | riser、NVMe、PCIe slot | fault clear、retry、inrush policy |

Power domain 表格要同時填 rail、clock、reset、dependency、ready 條件. 只填 rail 名稱不足以排查 probe 問題.

#### 4.4.1 Rail、Regulator 與 Power Domain 並不相同

- **Rail** 是實體電源網路，例如 `3V3_AUX`。
- **Regulator** 是產生或控制 rail 的元件，或 Linux對該供電來源的抽象。
- **Power domain** 是可一同管理的一組邏輯區塊，可能涉及 rail、clock gate、reset、isolation及context retention。

一個 rail 可供應多個 domain，一個 device 也可能依賴多個 rail：

```text
Ethernet PHY
├── AVDD：類比電路
├── DVDD：數位核心
├── VDDIO：I/O level
├── REFCLK
└── RESET_N
```

只量到其中一路有電，不能判定整顆 PHY 的供電條件完整。

#### 4.4.2 `regulator-boot-on` 與 `regulator-always-on`

兩者語意不同：

- `regulator-boot-on` 表示開機階段預期已啟用，Linux在接手時應視需要維持，但後續仍可能依 constraint 關閉。
- `regulator-always-on` 表示正常系統運作中不應由 regulator framework 關閉。

這些 property 描述軟體政策，不會替代硬體量測，也不保證 Bootloader、PMIC default 或 external pull 已正確建立初始 rail。若 Bootloader 已開啟 rail，但 Device Tree 沒表示正確，regulator core 可能在 unused-regulator handling 階段關閉它。

#### 4.4.3 Ramp、Inrush、PGOOD 與 Brownout

供電判斷不能只看 nominal voltage：

- **Ramp time**：從 enable 到進入規格所需時間。
- **Inrush current**：上電瞬間對上游供電造成的負載。
- **PGOOD delay**：rail達標後，PGOOD何時變化。
- **Ripple / droop**：負載切換時是否短暫離開規格。
- **Brownout**：電壓未完全消失但低於可靠工作範圍。

偶發 reset、Flash 寫入錯誤或裝置消失，可能發生在極短 droop期間。一般軟體 log只看得到後續 reset或I/O錯誤，需搭配 rail waveform與 fault latch才能建立事件順序。

#### 4.4.4 Isolation 與 Retention

部分 SoC power domain 在關閉前需先 isolation，避免已掉電區塊向仍有電的邏輯輸出不確定電位；重新上電時則需依順序解除 isolation、clock與reset。某些 domain支援 retention，能保留少量context。

若平台使用 generic power domain 或 SoC-specific PM driver，需確認：

- Domain on/off sequence。
- Isolation assert/deassert順序。
- Context是否需要driver重新寫入。
- Domain off後GPIO、interrupt與bus access的行為。
- Wake source是否位於仍供電的domain。

#### 4.4.5 Supply Dependency 與循環依賴

Regulator也可能有上游 supply：

```text
12V_AUX
→ 3V3_AUX regulator
→ 1V8 regulator
→ Sensor / Expander
```

Device Tree需正確描述parent supply，否則enable順序、reference count與狀態判讀可能不完整。若enable GPIO本身位於尚未上電的expander，則可能形成循環依賴，設計上需由always-on controller、CPLD或硬體預設打破循環。

## 4.5 DTS 範本: reset、clock、regulator、power domain

### 4.5.1 Reset controller consumer

``` dts
ethernet@1e660000 {
    compatible = "vendor,soc-mac";
    reg = <0x1e660000 0x1000>;
    resets = <&rst 12>;
    reset-names = "mac";
    clocks = <&syscon ASPEED_CLK_GATE_MAC1CLK>;
    clock-names = "macclk";
    status = "okay";
};
```

檢查重點:

- `resets` 與 `reset-names` 順序需與 driver 期待一致.
- shared reset 不適合任意放在多個 consumer node; 需確認 reset 影響範圍.
- 若 reset 其實是外部 IC 腳位, 通常用 `reset-gpios` 更直觀; 若是 SoC internal reset controller, 使用 `resets`.

### 4.5.2 Clock consumer

``` dts
uart5: serial@1e784000 {
    compatible = "ns16550a";
    reg = <0x1e784000 0x1000>;
    clocks = <&syscon ASPEED_CLK_APB>;
    clock-names = "uartclk";
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_uart5_default>;
    status = "okay";
};
```

檢查重點:

- `clock-names` 必須與 driver 期待名稱一致.
- clock parent / divider 改變可能影響 UART baud、I2C bus speed、PWM frequency、MAC reference.
- debugfs `clk_summary` 可用來看 enable count、prepare count、rate、parent.

### 4.5.3 Fixed regulator / GPIO enable rail

``` dts
vdd_3v3_aux: regulator-vdd-3v3-aux {
    compatible = "regulator-fixed";
    regulator-name = "vdd_3v3_aux";
    regulator-min-microvolt = <3300000>;
    regulator-max-microvolt = <3300000>;
    regulator-always-on;
};

vdd_phy: regulator-vdd-phy {
    compatible = "regulator-fixed";
    regulator-name = "vdd_phy";
    regulator-min-microvolt = <3300000>;
    regulator-max-microvolt = <3300000>;
    gpio = <&gpio0 45 GPIO_ACTIVE_HIGH>;
    enable-active-high;
    startup-delay-us = <10000>;
};

ethernet-phy@0 {
    reg = <0>;
    vdd-supply = <&vdd_phy>;
    reset-gpios = <&gpio0 46 GPIO_ACTIVE_LOW>;
};
```

檢查重點:

- `startup-delay-us` 應來自 regulator / PHY datasheet 或量測結果.
- enable GPIO active level 需與硬體實測一致.
- 若 rail 在 bootloader 階段已開, Linux regulator state 需避免 probe 時誤關.

#### 4.5.4 DTS 只描述依賴，不等於完成硬體時序

Device Tree 的 `clocks`、`resets`、`*-supply` 與 `power-domains` 用來讓driver取得資源，但真正的順序可能由下列位置共同決定：

- Driver probe 與 runtime PM callback。
- Regulator、clock、reset framework。
- CPLD state machine。
- PMIC或外部 supervisor。
- Bootloader已建立的初始狀態。
- Datasheet要求的delay與ready polling。

因此，DTS property齊全不代表時序必然正確。仍需確認driver呼叫順序、delay來源與實體波形。

#### 4.5.5 完整案例：I2C GPIO Expander 偶發消失

```text
3V3_AUX 上升
→ I2C pull-up rail 上升
→ Expander reset保持asserted
→ Rail穩定後解除reset
→ 等待內部初始化
→ I2C controller clock與pinmux ready
→ Driver第一次transaction
→ Expander gpiochip註冊
→ 下游presence service開始使用lines
```

若只在部分開機失敗，優先比較成功與失敗波形：

1. Expander VCC與pull-up rail是否同時ready。
2. Reset是否在VCC穩定前解除。
3. Reset deassert到第一次START condition的間隔。
4. I2C mux是否已選到正確channel。
5. Driver failure是NACK、timeout還是deferred probe。
6. Expander所在domain是否在Host off或runtime PM時被移除。

單純增加retry可能暫時降低失敗率，但仍應確認原始時序條件。

#### 4.5.6 完整案例：BMC Reboot 造成 Host 掉電

可能的完整路徑如下：

```text
BMC watchdog reset
→ SoC GPIO回到reset default / Hi-Z
→ 外部pull改變PWR_EN或PWRBTN_N
→ CPLD判斷Host off request或失去keep-alive
→ Host rail關閉
```

另一種可能是：

```text
BMC reboot
→ OpenBMC power daemon重新啟動
→ 尚未完成Host state rediscovery
→ 以預設state寫入control request
→ CPLD執行非預期power transition
```

因此需同步量測BMC reset、關鍵GPIO、CPLD output、Host PGOOD與PLTRST，並對照service journal。只確認BMC reboot命令本身無誤，無法排除控制權交接與userspace policy問題。

## 4.6 Domain 對照表範本

| Domain | Rail | Clock | Reset | Dependency | Ready 條件 | Owner | Boot risk | 狀態 |
|----|----|----|----|----|----|----|----|----|
| BMC core | \[待填\] | main osc / PLL \[待填\] | BMC_RST_N / POR \[待填\] | standby rail / reset IC | UART early log / reset reason valid | HW, BMC | Critical | \[待確認\] |
| DDR | \[待填\] | DDR clock \[待填\] | DDR reset / CKE | BMC core rail / DDR rail | SPL DDR init pass / memtest | HW, BMC | Critical | \[待確認\] |
| Boot SPI | 3V3_AUX \[待填\] | SPI clock from SoC | POR / flash reset \[待填\] | boot strap / WP / HOLD | U-Boot `sf probe` pass | HW, BMC | Critical | \[待確認\] |
| MAC/RGMII | \[待填\] | 25MHz / 125MHz \[待填\] | PHY_RST_N | PHY power / strap / MDIO | link up / `ethtool` 正常 | HW, BMC | High | \[待確認\] |
| RMII/NC-SI | \[待填\] | 50MHz / RMII REFCLK \[待填\] | PHY / NIC reset | Host NIC / sideband | NC-SI package response | BMC, Host | High | \[待確認\] |
| eSPI/LPC | \[待填\] | host side clock \[待填\] | host reset / PLTRST_N | PCH power / RSMRST / straps | channel ready / host state valid | Host, BMC | High | \[待確認\] |
| I2C sensor rail | \[待填\] | I2C controller clock | expander/sensor reset | pull-up rail / mux / bus owner | i2cdetect / driver probe | BMC | Medium | \[待確認\] |
| Fan PWM/Tach | \[待填\] | PWM / tach clock | peripheral reset | fan power / tach pull-up | PWM output / RPM read | BMC, HW | High | \[待確認\] |
| PCIe slot mgmt | \[待填\] | REFCLK \[待填\] | PERST_N | slot power / CPLD / host state | device present / MCTP / SMBus | Host, BMC | High | \[待確認\] |
| CPLD | \[待填\] | CPLD clock \[待填\] | CPLD_RST_N | standby rail | register map readable | CPLD/HW, BMC | Critical | \[待確認\] |

## 4.7 Timing 與量測欄位

對 power / reset / clock domain, 單點狀態不夠, 需記錄 timing. 建議以 AC applied、BMC reset deassert、Host power button、main rail enable、PGOOD、reset release 為共同時間軸.

| 時間點 | 事件 | 量測訊號 | Target | 實測 | 判定 |
|----|----|----|---:|---:|----|
| T0 | AC applied | AC_OK / standby input | 0 ms | \[待填\] | \[待確認\] |
| T1 | Standby rail stable | 3V3_AUX / 1V8 / core | \[待填\] | \[待填\] | \[待確認\] |
| T2 | BMC reset release | BMC_RST_N | \[待填\] | \[待填\] | \[待確認\] |
| T3 | Main clock stable | OSC / PLL_LOCK | \[待填\] | \[待填\] | \[待確認\] |
| T4 | Boot media access | SPI_CS / SPI_CLK | \[待填\] | \[待填\] | \[待確認\] |
| T5 | U-Boot banner | UART TX | \[待填\] | \[待填\] | \[待確認\] |
| T6 | Linux starts | kernel log timestamp | \[待填\] | \[待填\] | \[待確認\] |
| T7 | Userspace ready | systemd default target | \[待填\] | \[待填\] | \[待確認\] |
| T8 | Host power request | PWRBTN_N / PWR_EN | \[待填\] | \[待填\] | \[待確認\] |
| T9 | Main rail PGOOD | PS_PWROK / VR_PGOOD | \[待填\] | \[待填\] | \[待確認\] |
| T10 | Host reset release | PLTRST_N / PERST_N | \[待填\] | \[待填\] | \[待確認\] |
| T11 | POST complete | POST_COMPLETE / port80 | \[待填\] | \[待填\] | \[待確認\] |

量測建議:

- Reset 與 PGOOD 請使用同一台 LA / scope 的共同 trigger, 避免不同工具時間基準不一致.
- 對 clock 起振時間, 需量測振幅穩定與 frequency lock, 並同時確認是否已達穩定頻率與振幅.
- 對 GPIO / CPLD event, 需同步保存 BMC journal 與 CPLD register dump.

## 4.8 Reset reason 與 fault latch

Reset reason 是 boot failure 排查的入口, 但需注意它可能被下次 reset 覆蓋, 也可能只能描述 SoC 自身 reset, 無法描述外部 full board reset 原因.

建議保存欄位:

| 資料 | 來源 | 說明 |
|----|----|----|
| SoC reset reason | SoC register / kernel log / U-Boot log | POR、watchdog、software reset、external reset |
| Watchdog status | SoC / systemd / CPLD | timeout source、last feed time、reset target |
| CPLD fault latch | CPLD register | brownout、VR fault、PGOOD timeout、thermal trip |
| PMIC / VR fault | PMBus / PMIC register | UV/OV/OC/OT、status word、clear rule |
| Host reset cause | BIOS / CPLD / PCH sideband | warm reset、power button、OS reboot、watchdog |
| Event timeline | journal / SEL / Redfish EventLog | 軟體看見的 transition 與錯誤 |

常用指令範本:

> [待確認] 原始文件的 regulator 匯出指令使用 `/tmp/reset-debug/regulator.txt`, 其中 `$` 與空白可能影響輸出路徑. 以下保留原始指令, 執行前需依平台確認.

```bash
$ mkdir -p /tmp/reset-debug
$ cat /etc/os-release > /tmp/reset-debug/os-release.txt
$ uname -a > /tmp/reset-debug/uname.txt
$ cat /proc/cmdline > /tmp/reset-debug/proc-cmdline.txt
$ dmesg -T > /tmp/reset-debug/dmesg.txt
$ journalctl -b --no-pager > /tmp/reset-debug/journal-current.txt
$ journalctl -b -1 --no-pager > /tmp/reset-debug/journal-previous.txt 2>&1
$ systemctl --failed > /tmp/reset-debug/systemctl-failed.txt 2>&1
$ busctl tree xyz.openbmc_project.State.Host > /tmp/reset-debug/dbus-host-state.txt 2>&1
$ busctl tree xyz.openbmc_project.State.Chassis > /tmp/reset-debug/dbus-chassis-state.txt 2>&1
$ fw_printenv > /tmp/reset-debug/fw_printenv.txt 2>&1
$ cat /sys/kernel/debug/clk/clk_summary > /tmp/reset-debug/clk_summary.txt 2>&1
$ find /sys/kernel/debug/pinctrl -maxdepth 3 -type f -print > /tmp/reset-debug/pinctrl-files.txt 2>&1
$ cat /sys/kernel/debug/gpio > /tmp/reset-debug/debug-gpio.txt 2>&1
$ find /sys/class/regulator -maxdepth 3 -type f -print -exec sh -c 'echo ==== $1; cat $1 2>/dev/null' _ {} \; > /tmp/reset-debug/regulator.txt 2>&1
$ tar czf /tmp/reset-debug-$(date +%Y%m%d-%H%M%S).tar.gz -C /tmp reset-debug
```

若平台有 `devmem`、CPLD tool、PMBus tool、vendor reset reason command, 請另外保存:

```bash
# 依平台調整，以下僅為欄位提醒
# cpldtool dump > /tmp/reset-debug/cpld-dump.txt
# pmbus-status-dump > /tmp/reset-debug/pmbus-status.txt
# devmem <reset_reason_register> > /tmp/reset-debug/reset-reason.txt
```

## 4.9 OpenBMC / Host power state 整合

x86 類平台常由 OpenBMC x86-power-control 或平台 power daemon 監控 GPIO / D-Bus 訊號, 維護 Host state machine, 並提供 hard power on/off/cycle、soft power on/off/cycle 等能力. 這類 service 的設定與本章 domain 資料需一致, 尤其是 PWRBTN、RESET、NMI、PS_PWROK、POST_COMPLETE、PLTRST、SLP_Sx、RSMRST.

| Signal | 常見角色 | 對 domain 的意義 |
|----|----|----|
| PS_PWROK | PSU / main power ready | Host main rail 是否可視為有效 |
| SIO_POWER_GOOD / PCH_PWROK | Host power good | Host sideband 是否可讀 |
| RSMRST_N | Resume reset | PCH standby domain 是否 ready |
| PLTRST_N | Platform reset | Host peripheral 是否離開 reset |
| POST_COMPLETE | BIOS POST 狀態 | Host boot 是否到達指定階段 |
| PWRBTN_N | BMC 對 host power button pulse | Power transition requester |
| RESET_N / RSTBTN_N | BMC 對 host reset | Host reset transition |
| NMI_N | BMC 觸發 NMI | Debug / crash capture |

驗證重點:

- BMC reboot 後, power daemon 是否能重新發現 host current state, 而並重新判斷 Host 當前狀態.
- AC restore policy 是否和 CPLD default / BIOS policy / BMC policy 一致.
- 若使用 PLTRST 判斷 warm reset, 需確認 polarity、debounce 與 host reset timing.
- 所有 power button / reset pulse width 需符合 platform power sequence 文件.
- 多 host 平台需確認每個 host 的 GPIO / DBUS 設定沒有共用錯線.

## 4.10 Device probe deferred 與 dependency 排查

Reset / Clock / Power Domain 問題常在 kernel 中呈現為 deferred probe. 建議依序檢查 supply、clock、reset、GPIO、IRQ、bus parent.

常用指令:

```bash
$ dmesg | grep -Ei 'defer|probe|reset|clk|clock|regulator|supply|power domain|genpd|timeout'
$ cat /sys/kernel/debug/devices_deferred 2>/dev/null
$ cat /sys/kernel/debug/clk/clk_summary 2>/dev/null | grep -i '<device-or-clock>'
$ find /sys/class/regulator -maxdepth 2 -type l -o -type d 2>/dev/null
```

常見方向:

| dmesg / 現象 | 建議排查方向 | 第一輪檢查 |
|----|----|----|
| `-EPROBE_DEFER` | regulator / clock / reset provider 尚未 ready | provider driver、DTS phandle、kernel config |
| `supply vdd not found` | `*-supply` 名稱錯或 regulator node 不存在 | DTS supply property、regulator-name |
| `failed to get reset` | `resets` / `reset-names` 錯 | reset binding、driver 期待名稱 |
| `failed to enable clock` | clock provider / gate / parent 問題 | clk_summary、clock-names、driver log |
| device timeout | reset 未 release、clock 無、rail 未穩 | scope、pinctrl、regulator state |
| I2C NACK | device rail off、reset asserted、pull-up rail off、bus mux 錯 | rail、reset、i2cdetect、mux channel |
| MAC no link | PHY rail/clock/reset/strap | MDIO、PHY reset waveform、REFCLK |

## 4.11 常見問題與排查入口

| 現象 | 建議排查方向 | 第一輪檢查 |
|----|----|----|
| BMC 完全無 UART | core rail、main oscillator、BMC reset、strap | scope rail/reset/osc、BootROM SPI access |
| BMC watchdog 後 host 掉電 | watchdog reset 範圍過大、CPLD default、power enable glitch | reset scope、CPLD latch、PWR_EN waveform |
| Peripheral probe 偶發失敗 | reset release 太早、clock unstable、rail ramp 慢 | LA/scope timing、driver retry、startup-delay |
| MAC link 不起 | PHY reset/clock/strap/MDIO/rail | REFCLK、PHY_RST_N、MDIO read、ethtool |
| eMMC 偶發找不到 | eMMC reset/clock/power sequence、bus width | dmesg mmc、scope CMD/CLK/RST、EXT_CSD |
| eSPI/LPC 不 ready | host standby domain、RSMRST、PLTRST、clock | host signal timeline、power daemon log |
| Fan PWM 無輸出 | PWM clock gate、pinmux、fan power、daemon override | clk_summary、pinctrl、sysfs、scope |
| I2C expander 消失 | expander rail/reset、bus mux、clock stretching、address conflict | i2cdetect、rail、reset、mux state |
| BMC reboot 後 power state 錯 | power daemon rediscovery 不完整、state file 舊資料 | D-Bus state、journal、power-config |
| factory reset 後 power policy 錯 | persistent policy 被清或未重建 | settings manager、power restore policy |
| AC restore 行為不一致 | CPLD default / BIOS / BMC policy 衝突 | AC cycle log、CPLD register、BMC setting |
| reset reason 不可信 | register 被清、只記錄 SoC reset、外部 latch 未讀 | early U-Boot log、CPLD latch、PMIC status |

## 4.12 Bring-up 順序

Bring-up 建議遵循「先核心開機條件，再單一 peripheral，最後才做 power transition 與異常注入」的順序。

#### 4.12.1 建立靜態依賴圖

1. 從 schematic列出所有 rail、enable、PGOOD、reset、clock、ready與fault signals。
2. 標記每個訊號的source、consumer、active level、voltage domain及外部pull。
3. 建立BMC core、DDR、Boot Flash、UART、CPLD、I2C、Network、Host sideband等domain。
4. 對每個domain填寫rail、clock、reset、dependency、ready條件與owner。
5. 標記哪些設定由Bootloader、CPLD、Kernel driver或OpenBMC service控制。
6. 列出warm reset、BMC-only reset、Host reset及AC cycle各自會清除與保留的狀態。

#### 4.12.2 先驗證 BMC 最小開機鏈

```text
Standby rail
→ Main oscillator
→ POR / BMC reset
→ Boot strap
→ Boot SPI access
→ SPL / DDR init
→ U-Boot UART
→ Kernel start
```

此階段優先使用scope與UART。若沒有early UART，先查rail、oscillator、reset與Boot SPI activity，不應先從systemd或OpenBMC service排查。

#### 4.12.3 驗證 Linux Provider

依序確認：

- Regulator provider與constraints。
- Clock provider、parent、rate與gate。
- Reset controller與reset lines。
- Generic power domain或SoC PM driver。
- GPIO、pinctrl與interrupt controller。
- Parent bus與外部mux。

檢查running DTB、kernel config、provider probe log與`devices_deferred`。Provider未ready時，consumer錯誤通常只是後續現象。

#### 4.12.4 逐一驗證 Consumer

每個consumer都建立相同紀錄：

```text
Supply request與實體rail
→ Clock request與實體clock
→ Reset assert/deassert waveform
→ Ready delay / status polling
→ 第一次register或bus transaction
→ Driver probe結果
→ Userspace object建立
```

先選低風險、容易量測的device，再處理Host power、Flash mux或slot power等高風險domain。

#### 4.12.5 驗證控制權交接

至少驗證：

- AC applied到Bootloader。
- Bootloader到Kernel。
- Kernel provider到consumer driver。
- Driver到OpenBMC service。
- BMC warm reboot且Host維持運作。
- Service restart後重新發現目前hardware state。

每個交接點都可能發生短暫disable、reset pulse、clock gate或錯誤預設值，需以共同時間軸保存waveform與log。

#### 4.12.6 驗證 Reset Matrix

建立reset測試矩陣：

```text
Software reboot
BMC watchdog reset
External BMC reset
Host warm reset
Host cold reset
Peripheral reset
Full AC cycle
Brownout / rail fault，限核准測試環境
```

對每一項記錄受影響domain、保留狀態、reset reason、CPLD/PMIC latch及Host side effect。不要只記錄「系統有重新啟動」。

#### 4.12.7 驗證異常與 Recovery

在具備安全復原能力的環境中，測試：

- Rail延遲或PGOOD timeout。
- Reset stuck asserted / deasserted。
- Clock missing或clock generator未enable。
- Deferred probe與provider晚到。
- BMC reboot期間Host state維持。
- Runtime suspend/resume。
- Service restart與state rediscovery。
- Watchdog、AC loss及fault latch保存流程。

先保存原始證據，再執行clear、reset或power cycle，避免清除reset reason與fault latch。

#### 4.12.8 Bring-up 完成條件

完成不只是device已probe，還應確認：

- 依賴圖與實際波形一致。
- DTS provider/consumer關係與driver binding一致。
- Rail、clock、reset與ready時序符合datasheet或量測規格。
- Warm/cold reset後的保留狀態符合設計。
- BMC reboot不造成未定義的Host side effect。
- Runtime PM與service restart可正確恢復。
- Reset reason、fault latch、UART、kernel log、journal、D-Bus與waveform可對上同一時間軸。
- 異常情境具有明確的shutdown、retry、rollback或recovery行為。

## 4.13 當前平台 Reset / Clock / Power 實測表

| Domain | Rail 量測 | Clock 量測 | Reset 量測 | Ready signal | Kernel / service 狀態 | 結論 |
|----|----|----|----|----|----|----|
| BMC core | \[待填\] | \[待填\] | \[待填\] | UART early log | \[待填\] | \[待確認\] |
| DDR | \[待填\] | \[待填\] | \[待填\] | SPL DDR init pass | \[待填\] | \[待確認\] |
| Boot flash | \[待填\] | SPI_CLK \[待填\] | \[待填\] | `sf probe` / kernel mtd | \[待填\] | \[待確認\] |
| MAC/RGMII | \[待填\] | 25/125MHz \[待填\] | PHY_RST_N \[待填\] | link up | \[待填\] | \[待確認\] |
| eSPI/LPC | \[待填\] | \[待填\] | PLTRST_N / RSMRST_N \[待填\] | channel ready | \[待填\] | \[待確認\] |
| I2C expander | \[待填\] | I2C bus clock \[待填\] | EXP_RST_N \[待填\] | device ACK | \[待填\] | \[待確認\] |
| Fan domain | fan rail \[待填\] | PWM/Tach clock \[待填\] | \[待填\] | RPM read | \[待填\] | \[待確認\] |
| CPLD | \[待填\] | \[待填\] | CPLD_RST_N \[待填\] | register readable | \[待填\] | \[待確認\] |
| Host main | \[待填\] | host clocks \[待填\] | PLTRST_N \[待填\] | POST complete | \[待填\] | \[待確認\] |

## 4.14 驗收 Checklist

- [ ] 所有 reset source 與 reset domain 已列出, 包含 BMC-only、host-only、full board、peripheral.
- [ ] reset reason register、CPLD fault latch、PMIC / VR fault status 的讀取方式已記錄.
- [ ] 主要 clock source、frequency、enable、parent、consumer 已列出.
- [ ] `clk_summary` 可讀, 且 key peripheral clock rate / enable state 合理.
- [ ] power rail、regulator、PGOOD、fault line、dependency 與 ready 條件已列出.
- [ ] DTS 中 `resets` / `reset-names`、`clocks` / `clock-names`、`*-supply` 與 driver binding 一致.
- [ ] GPIO reset / enable line 的 active level、pulse width、startup delay 已量測.
- [ ] AC on、BMC reboot、watchdog reset、host power on/off/cycle 都有 timing log.
- [ ] BMC reboot 不會造成 host power 非預期切換, 或已有明確產品政策.
- [ ] Host state rediscovery、AC restore policy、power daemon state transition 已驗證.
- [ ] Device probe deferred 已檢查, 沒有未解釋的 supply / clock / reset dependency.
- [ ] Network、eSPI/LPC、I2C expander、fan、CPLD 等關鍵 domain 通過實機驗證.
- [ ] 異常測試包含 brownout、fault latch、reset stuck、clock missing、power rail delayed、watchdog reset.
- [ ] 測試紀錄包含 waveform、UART、dmesg、journal、D-Bus state、CPLD / PMIC dump、image version.

## 4.15 本章重點

1. Device 可用前通常要依序滿足 power、clock、reset、pinmux 與 driver dependency.
2. Reset source、reset domain 與 reset consumer需要分開記錄, 才能判斷實際影響範圍.
3. Clock 排查同時包含實體波形、parent / divider、gate與driver enable state.
4. Power rail名稱不足以描述 domain; 還需要PGOOD、fault、dependency與ready條件.
5. BMC reboot、watchdog reset、Host reset與full-board reset需要分開驗證.
6. Deferred probe 通常代表regulator、clock、reset、GPIO、IRQ或parent bus尚未 ready.
7. Reset reason可能被覆寫或只涵蓋部分domain, 需搭配CPLD與PMIC fault latch.
8. Power-control service 必須在BMC 重啟後重新發現Host 狀態.
9. 關鍵時序應以同一個scope / logic analyzer 時間基準量測.
10. 驗收紀錄應同時保存 waveform、UART、kernel log、journal、D-Bus 與 firmware版本.

## 4.16 本章參考資料
- Linux kernel documentation - Reset controller API: https://www.kernel.org/doc/html/latest/driver-api/reset.html
- Linux kernel documentation - Reset Device Tree bindings: https://docs.kernel.org/devicetree/bindings/reset/
- Linux kernel documentation - Common Clock Framework: https://www.kernel.org/doc/html/latest/driver-api/clk.html
- Linux kernel documentation - Regulator framework overview: https://docs.kernel.org/power/regulator/overview.html
- Linux kernel documentation - Voltage and current regulator API: https://docs.kernel.org/driver-api/regulator.html
- OpenBMC x86-power-control README: https://github.com/openbmc/x86-power-control/blob/master/README.md
