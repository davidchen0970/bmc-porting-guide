# 9. U-Boot, Kernel Driver 與核心服務

## 適用範圍

本文件說明 U-Boot 多階段啟動架構, Driver Model, Device Tree, environment, 映像載入與 boot policy, 並整理 Linux kernel 的 device, driver, bus, probe, resource, subsystem, 以及 OpenBMC 核心服務的整合與排查方式.

## 適用讀者

- 負責 BMC 韌體, U-Boot, Linux kernel, Device Tree, OpenBMC service, Yocto 或平台 bring-up 的開發與整合人員.
- 執行 boot flow, driver probe, kernel subsystem, D-Bus, Redfish/IPMI 或硬體介面排查與驗證的人員.

## 快速導覽

- [理解 U-Boot 多階段架構](#91-u-boot-是什麼): BootROM, TPL, VPL, SPL, U-Boot proper 與階段交接.
- [檢查 U-Boot Driver Model 與啟動政策](#93-u-boot-driver-model): Device Tree, environment, 映像載入, A/B 與 recovery.
- [理解 Kernel Driver Model](#99-kernel-driver-是什麼): Device, driver, bus, subsystem, class 與 probe.
- [分析 Probe 與資源取得](#913-probe-裡面實際做什麼): Resource, `devm_*`, error pointer, deferred probe 與 driver register.
- [核對 Userspace 與 OpenBMC 整合](#918-driver-如何把資料交給-userspace): Sysfs, hwmon, IIO, D-Bus 與 Redfish/IPMI.
- [執行完整排查與驗收](#924-target-端完整排查流程): Running DTB, driver binding, subsystem, Debug Log 與驗收 Checklist.

Kernel driver 是 Linux kernel 中用來控制硬體的程式. Device Tree 告訴 kernel 板子上有哪些裝置, 以及裝置使用的位址, 中斷, GPIO, clock 和 reset; driver 則依照這些資訊初始化硬體, 處理硬體事件, 並將硬體功能提供給系統中的其他程式.

例如, 板子上有一顆 TMP75 溫度感測器, 接在 I2C bus 5, 位址為 `0x48`:

- Device Tree 描述這顆感測器的位置與型號.
- Kernel 建立對應的 I2C device.
- TMP75 driver 與這個 device 配對, 並讀取感測器暫存器.
- Driver 將溫度註冊到 hwmon.
- OpenBMC sensor service 讀取 hwmon, 再建立 D-Bus sensor object.

可以先記住以下關係:

```mermaid
flowchart TB
    A["Device Tree 描述硬體"] --> B["Driver 控制硬體"]
    B --> C["Kernel subsystem 提供共同介面"]
    C --> D["OpenBMC service 將資料轉成 D-Bus object"]
```

本章先說明U-Boot的多階段架構, Driver Model, Device Tree, environment與映像載入, 再進入Linux Kernel的device, driver, probe, subsystem與OpenBMC服務.

## 9.1 U-Boot 是什麼

U-Boot(Das U-Boot)是嵌入式系統常見的開機載入程式家族. 它的核心任務是把平台從"SoC 剛離開 reset, 可用資源非常有限"帶到"Linux kernel 或其他 payload 已經載入記憶體, 而且具備啟動所需參數"的狀態.

這個過程不一定由單一 U-Boot binary 完成. 當 SoC BootROM 能載入的映像大小有限, 片上 SRAM 不足以容納完整 U-Boot, 或 DDR 尚未初始化時, U-Boot 會被拆成多個可選階段: TPL, VPL, SPL 與 U-Boot proper. U-Boot 文件把這些載入器統稱為 xPL, 其中 x 代表任一早期 Program Loader. 常見順序為 TPL → VPL → SPL → U-Boot proper, 但多數平台只使用 SPL, 不一定具有 TPL 或 VPL.

```mermaid
flowchart TB
    subgraph G1["1. SoC 固定啟動階段"]
        direction LR
        A["Reset"] --> B["SoC BootROM"]
    end
    subgraph G2["2. U-Boot 早期載入器 xPL"]
        direction LR
        C["TPL, 可選"] --> D["VPL, 可選"] --> E["SPL, 常見"]
    end
    subgraph G3["3. 完整 Bootloader"]
        direction LR
        F["U-Boot proper"] --> G["Boot policy / image verification / FDT"]
    end
    subgraph G4["4. Payload"]
        direction LR
        H["Linux Kernel"] --> I["Root filesystem / OpenBMC"]
    end
    G1 --> G2 --> G3 --> G4
    B -. "簡單平台可略過 xPL" .-> F
    E -. "Falcon Mode 可直接啟動" .-> H
```

常見平台路徑:

| 平台情境 | 啟動路徑 |
|---|---|
| 簡單平台 | `BootROM → U-Boot proper → Linux` |
| 常見 BMC / ARM 平台 | `BootROM → SPL → U-Boot proper → Linux` |
| 片上 SRAM 很小 | `BootROM → TPL → SPL → U-Boot proper → Linux` |
| 具備驗證階段 | `BootROM → TPL → VPL → SPL → U-Boot proper → Linux` |
| SPL 直接啟動 Linux | `BootROM → SPL → Linux` |

最後一種通常稱為 Falcon Mode 或類似的直接 OS 啟動設計. 此時 U-Boot proper 可以被略過, 但 debug shell, 完整 command set 與一般 boot policy 也可能不會出現.

### 9.1.1 為什麼需要多階段

SoC 剛解除 reset 時, 硬體能力通常受到下列限制:

- DDR 尚未完成 training, 因此不能使用大容量 DRAM.
- 只能使用 SoC 內部 SRAM, cache-as-RAM 或 BootROM 提供的少量記憶體.
- BootROM 可能只支援特定 boot media, 固定 offset, 固定 header 或有限映像大小.
- Clock, pinmux, power rail 與 reset controller仍處於 reset default.
- 尚未建立完整 C runtime environment.
- Console, heap, BSS, Driver Model 或 filesystem可能尚未可用.

完整 U-Boot proper通常包含command shell, network, filesystem, USB, storage, FIT, EFI與大量drivers, 映像遠大於早期 SRAM可容納的範圍. 因此先使用很小的xPL完成必要初始化, 再把較大的下一階段載入DDR, 是多階段架構的主要原因. U-Boot 官方文件也指出, 完整U-Boot可能大到無法由BootROM直接載入, 這是拆分TPL / SPL等階段的原始動機.

### 9.1.2 每個階段都可視為獨立 Firmware Image

TPL, VPL, SPL與U-Boot proper雖然來自同一份U-Boot source tree, 但通常各自:

- 使用不同Kconfig集合.
- 產生不同object files.
- 使用不同link address與linker script.
- 使用不同Device Tree裁減結果.
- 具有不同driver與library集合.
- 產生不同ELF, binary與map files.
- 受不同ROM / SRAM / flash layout限制.

Generic xPL framework會將各階段分別建置到`tpl/`, `vpl/`或`spl/`等目錄; SPL常見產物包括`u-boot-spl`, `u-boot-spl. bin`與`u-boot-spl. map`. xPL可透過`CONFIG_XPL_BUILD`與階段特定設定選擇source與功能.

所以"U-Boot有啟用某driver"需要進一步問:

```text
是在 U-Boot proper 啟用？
還是在 SPL 啟用？
TPL 也需要嗎？
該階段的 Device Tree 有保留對應 node 嗎？
映像大小是否仍符合 SRAM / ROM 限制？
```

### 9.1.3 U-Boot 的主要責任

- 初始化足以繼續開機的clock, pinmux, power與reset.
- 初始化DDR或其他主記憶體.
- 存取SPI, NAND, eMMC, SD, USB或network boot source.
- 讀取board identity, strap, fuse與boot metadata.
- 建立U-Boot Driver Model devices.
- 選擇正常, A/B, recovery或network boot path.
- 載入kernel, DTB, initramfs, TF-A, OpenSBI, EDK II或其他payload.
- 驗證image hash / signature, 依平台安全架構.
- 建立或修正傳給Linux的Device Tree.
- 建立kernel command line.
- 將控制權交給下一個firmware stage或Linux kernel.

### 9.1.4 U-Boot 與 Linux Kernel 的責任邊界

| 項目 | U-Boot / xPL | Linux Kernel |
|---|---|---|
| 執行時間 | Linux啟動前 | U-Boot handoff後 |
| 主要目標 | 建立可啟動OS的最小平台 | 完整runtime硬體管理 |
| Driver範圍 | Boot, recovery與diagnostic所需 | 完整subsystem與runtime功能 |
| Device Tree | 自身控制, 階段裁減, OS fixup | 建立Linux devices與resources |
| Memory | SRAM, 早期DRAM, relocation | MMU, allocator, 完整memory management |
| 儲存 | 載入映像, environment, metadata | MTD, UBI, block, filesystem |
| 網路 | DHCP / TFTP / PXE等boot能力 | 完整network stack與services |
| 安全 | Verified boot, measured boot, rollback policy | Runtime security與userspace policy |
| Recovery | 備援slot, golden image, 下載映像 | 更新service, diagnostic與mark-good |

U-Boot能讀取某顆裝置, 只能證明U-Boot階段的clock, pinmux, driver與設定足以使用該裝置; Linux中的driver, running DTB與OpenBMC service仍需另外驗證.

## 9.2 U-Boot 整體架構

可以從五個面向理解U-Boot:

```text
Boot Phases
BootROM → TPL / VPL / SPL → U-Boot proper

Initialization Framework
start.S → lowlevel_init → board_init_f → relocation → board_init_r

Driver Model
Device Tree → bind → device / driver / uclass → probe → ops

Boot Policy
Environment / Standard Boot / A/B metadata / recovery

Image and Handoff
Binman / raw image / FIT → TF-A / OpenSBI / Linux / EFI
```

U-Boot proper與xPL共用許多source, drivers與初始化概念, 但各階段會以不同Kconfig與link配置產生獨立映像. 不是把完整U-Boot binary切成數段, 而是針對每個階段重新選擇所需功能並個別編譯, 連結. Generic xPL framework的目標正是讓board code在不同program loaders之間重用, 同時保留階段特定建置.

### 9.2.1 BootROM: U-Boot 之前的固定起點

BootROM不是U-Boot的一部分. 它是SoC製造商放在晶片內部ROM中的第一段程式, 通常不可由一般firmware更新.

BootROM常負責:

- 在reset後建立CPU最基本執行環境.
- 取樣boot strap或fuse.
- 選擇SPI, NAND, eMMC, SD, UART或USB等boot source.
- 依SoC規則尋找image header或固定offset.
- 驗證第一階段映像, 若Secure Boot已啟用.
- 將第一階段載入片上SRAM.
- 跳到TPL, SPL或U-Boot proper入口.

BootROM通常不知道ext4, UBIFS, 完整GPT或U-Boot environment. 它能理解哪些media與image format, 完全由SoC boot specification決定.

若平台完全沒有早期UART, SPI chip-select或boot-media activity, 優先檢查:

- SoC power與reset.
- Main oscillator.
- Boot straps / fuse.
- 第一階段image offset與header.
- BootROM認得的media模式.
- Secure Boot驗證結果或recovery mode.

### 9.2.2 TPL: Tertiary Program Loader

TPL是可選的最早期 U-Boot loader. 雖然名稱是 Tertiary Program Loader, 但在一般 U-Boot 文件的常見順序中, 它位於 SPL之 前; 名稱反映歷史沿革, 不宜只依英文序數推斷執行順序. 官方文件將TPL描述為"盡可能小的very early init", 其工作是載入SPL, 若啟用VPL則先載入VPL.

TPL通常出現在下列平台:

- BootROM可載入的第一階段大小很小.
- 片上SRAM不足以容納具備DDR driver的SPL.
- DDR初始化前還要先初始化PMIC, clock或另一段SRAM.
- 需要再分一層以符合ROM header, security或media限制.

TPL的常見責任:

```text
SoC剛離開BootROM
        ↓
建立極小stack / global data，依architecture
        ↓
最小clock、pinmux、timer或serial，依空間允許
        ↓
存取下一階段所在boot media
        ↓
載入VPL或SPL到指定memory
        ↓
驗證最基本image資訊
        ↓
跳轉下一階段entry point
```

TPL通常不負責:

- 完整DDR初始化, 除非平台架構特別設計.
- Command shell.
- 完整network stack.
- 大量filesystem與USB功能.
- 一般互動式debug commands.

官方文件指出, TPL載入目標時只保證支援raw binary, entry address等同image start; 其他格式與功能需依版本及平台設計確認.

TPL常見產物與設定:

```text
u-boot-tpl
u-boot-tpl.bin
u-boot-tpl.map
CONFIG_TPL
CONFIG_TPL_<FEATURE>
CONFIG_TPL_TEXT_BASE
```

加入一個TPL driver可能讓binary超過BootROM或SRAM限制, 所以TPL修改需要同時檢查map file, section size與最終packaged image.

### 9.2.3 VPL: Verifying Program Loader

VPL是可選的驗證階段, 設計目標是在A/B verified boot情境下驗證並選擇一個SPL, 接著交給SPL繼續啟動. U-Boot官方stable文件仍將部分VPL邏輯描述為持續發展中的功能, 因此實際平台是否使用, 以及驗證與slot selection做到哪一層, 應以專案版本為準.

概念路徑:

```text
TPL
  ↓
VPL讀取boot metadata
  ↓
選擇SPL-A或SPL-B
  ↓
驗證hash / signature / rollback條件
  ↓
載入並啟動通過驗證的SPL
```

VPL可處理的政策可能包括:

- SPL A/B selection.
- Signature verification.
- Rollback index.
- Boot attempt / failure metadata.
- Recovery SPL selection.

這些能力不是所有U-Boot平台的固定行為. 產品若聲稱VPL提供verified boot, 需要留下實際key chain, metadata位置, fallback條件與中斷測試結果.

### 9.2.4 SPL: Secondary Program Loader

SPL是最常見的xPL階段. 它通常在片上SRAM中執行, 核心任務是初始化SDRAM / DDR, 接著把U-Boot proper或其他payload載入較大的記憶體. 官方文件也明確將SPL描述為設定SDRAM並載入U-Boot proper的階段, 且可載入其他firmware components.

SPL常見責任:

- Early clock, reset, pinmux與power初始化.
- Early console, 若size與硬體允許.
- PMIC或regulator初始化.
- DDR controller, PHY與memory training.
- Boot-device選擇與fallback順序.
- 初始化SPI, NAND, MMC, UART或USB downloader等boot transport.
- 載入U-Boot proper, FIT, TF-A, OpenSBI, EDK II或Linux.
- 驗證image header, CRC, hash或signature, 依build.
- 將board, boot-device與handoff資訊傳給下一階段.

SPL image與完整U-Boot分開建置, 常見產物為:

```text
spl/u-boot-spl
spl/u-boot-spl.bin
spl/u-boot-spl.map
```

常見Kconfig概念:

```text
CONFIG_SPL
CONFIG_SPL_SERIAL
CONFIG_SPL_DRAM
CONFIG_SPL_MMC
CONFIG_SPL_SPI
CONFIG_SPL_SPI_FLASH_SUPPORT
CONFIG_SPL_LOAD_FIT
CONFIG_SPL_OF_CONTROL
CONFIG_SPL_DM
```

實際symbol依U-Boot版本與platform而異. 只有`CONFIG_<FEATURE>=y`不代表SPL具備該功能; 通常還需要對應的`CONFIG_SPL_<FEATURE>`.

### 9.2.5 SPL 如何選擇下一階段

SPL可由board code與Kconfig共同決定boot methods. U-Boot文件描述了`board_boot_order()`或單一`spl_boot_device()`類型的選擇方式, 再由build configuration決定MMC, SPI, NAND, network等loader是否存在.

```text
Reset Cause / Boot Strap / Recovery GPIO
        ↓
SPL Boot Order
        ↓
嘗試Primary Boot Device
        ↓ fail
嘗試Secondary / Recovery Device
        ↓
載入Image到DDR
        ↓
驗證Image
        ↓
Jump to Entry Point
```

排查SPL找不到image時, 需要分開檢查:

1. Boot device是否已初始化.
2. Offset, partition或filename是否正確.
3. Image format是否由該SPL build支援.
4. Load address是否位於可用DDR.
5. Hash / signature是否通過.
6. Entry point與architecture是否正確.

### 9.2.6 U-Boot Proper

U-Boot proper是功能最完整的U-Boot階段, 通常在 DDR 中執行, 也是常見互動式 command shell 所在階段. 官方 TPL / SPL 文件把 U-Boot proper 描述為包含 commands 並實作 OS 載入邏輯的階段.

主要功能包括:

- 完整Driver Model與較多drivers.
- Command-line shell.
- Environment與boot scripts.
- Standard Boot / distro boot / extlinux / EFI.
- Filesystem與partition存取.
- Network boot與diagnostic.
- FIT image選擇, 驗證與解壓縮.
- FDT fixup與overlay.
- A/B boot, bootcount或platform recovery policy.
- 啟動Linux, EFI application或下一個firmware payload.

"U-Boot proper有command"不表示每個build都納入所有commands. Command, filesystem, network與driver都由Kconfig和image size policy決定.

### 9.2.7 ARM Trusted Firmware, OpenSBI 與其他 Firmware

在部分architecture中, SPL不會直接跳到U-Boot proper, 而是先載入其他privileged firmware:

```text
ARMv8 常見概念
BootROM → SPL → TF-A BL31 → U-Boot proper作為BL33 → Linux

RISC-V 常見概念
BootROM → SPL → OpenSBI → U-Boot proper → Linux
```

U-Boot 官方文件列出SPL可啟動ARM Trusted Firmware BL31, EDK II, Linux或RISC-V OpenSBI等用途.

這些firmware負責的內容可能包含:

- Secure monitor與exception level transition.
- PSCI與power management.
- RISC-V SBI runtime services.
- Trusted world初始化.
- 下一階段entry point與handoff parameters.

排查"SPL log結束但U-Boot proper沒有banner"時, 中間的TF-A / OpenSBI也是獨立診斷層, 需保存其版本, console與packaging位置.

### 9.2.8 初始化函式架構

U-Boot與xPL的初始化通常從architecture-specific `start. S`進入, 再依平台執行`lowlevel_init()`, `board_init_f()`, relocation / BSS處理與`board_init_r()`. 官方board initialization文件指出, 這套概念同時適用於xPL與U-Boot proper, 但部分board或architecture可能有差異.

```text
Reset Vector / start.S
        ↓
lowlevel_init()
只做足以進入board_init_f的最低限度初始化
        ↓
board_init_f()
建立早期machine state、serial與DRAM等
        ↓
BSS clear / Stack或Global Data調整
        ↓
Relocation，主要用於U-Boot proper
        ↓
board_init_r()
完整Driver Model、environment與boot policy
```

`lowlevel_init()`階段通常還沒有完整stack, BSS或global data, 且官方指引要求只做最少初始化; `board_init_f()`在SRAM stack與受限runtime下準備DRAM及UART; 之後才清BSS並進入較完整的`board_init_r()`.

### 9.2.9 SRAM, DDR, Stack, BSS 與 Relocation

理解xPL最重要的是"每一階段目前能使用哪種memory".

| 階段 | 常見執行位置 | 記憶體狀態 | 主要限制 |
|---|---|---|---|
| BootROM | SoC ROM | 內部固定資源 | 行為由SoC定義 |
| TPL | SRAM / cache | DDR通常未ready | 極小binary與stack |
| SPL early | SRAM | 正在初始化DDR | Heap, BSS與driver受限 |
| SPL late | SRAM或DDR, 依平台 | DDR已ready | 準備載入下一階段 |
| U-Boot proper pre-reloc | Load address / DDR | relocation尚未完成 | 只使用支援pre-reloc的內容 |
| U-Boot proper post-reloc | DDR高位或指定區域 | 完整runtime較可用 | 仍需避免覆蓋image / FDT |

BSS保存未初始化的global / static variables. 若某階段尚未清BSS, 就不能假設這些變數為0. 官方初始化文件特別提醒xPL的`board_init_f()`階段通常沒有BSS, 不應依賴一般global / static variables.

Relocation是將U-Boot proper搬到DDR中規劃好的runtime位置, 再修正相關資料與執行狀態. 這可保留較連續的RAM給kernel, FDT與其他images, 也讓U-Boot取得較完整的runtime空間.

### 9.2.10 Pre-Relocation Device

在U-Boot proper完成relocation前, 只有被標記為早期階段需要, 且對應Kconfig已啟用的Device Tree nodes / drivers可使用. xPL也可能使用經過裁減的DTB, 只保留該階段需要的nodes與properties.

這表示:

- DTS中有node, 不代表TPL / SPL會保留.
- U-Boot proper可probe, 不代表SPL可probe.
- Driver source存在, 不代表階段特定Kconfig已啟用.
- Clock / pinctrl / regulator provider也要在同一階段可用.

排查early driver時應同時檢查phase tags, 裁減後DTB, `CONFIG_SPL_*` / `CONFIG_TPL_*`與map file.

### 9.2.11 Image Packaging: Raw, FIT 與 Binman

不同階段需要被放到flash或boot media中的正確位置. 常見格式:

- Raw binary: 程式入口通常由固定load / entry address決定.
- Legacy U-Boot image: 具有legacy header與CRC.
- FIT: 封裝多個images, configurations, hash與signature.
- Binman image: 依layout把TPL, SPL, U-Boot, DTB, TF-A與其他blobs組成最終firmware image.

U-Boot 官方文件說明FIT是U-Boot讀取與啟動映像的標準封裝格式; Binman則處理FIT不適合涵蓋的初始可執行header, device boundary與flash內多component layout.

```text
Final Flash Image
├── ROM Header / SoC Header
├── TPL，若使用
├── VPL，若使用
├── SPL
├── TF-A / OpenSBI，依平台
├── U-Boot proper
├── U-Boot DTB
├── Environment
└── Kernel / Recovery metadata，依layout
```

所以build成功後還要驗證最終package中的offset, alignment, padding, load address, entry point與signature. 只檢查`u-boot. bin`不足以驗證量產映像.

### 9.2.12 階段 Handoff Contract

每個stage交給下一個stage時, 需要明確的handoff contract:

- 下一階段image位置與大小.
- Load address與entry point.
- CPU mode / exception level.
- Cache與MMU狀態.
- DDR是否已完成初始化.
- Stack與reserved memory.
- FDT, bloblist或handoff structure位置.
- Boot device與slot資訊.
- Reset reason與security state.

若前一階段留下的cache, clock, pinmux或security state和下一階段假設不同, 可能出現只在warm reset, watchdog reset或特定boot source發生的問題.

### 9.2.13 各階段的診斷問題

| 最後可見訊息 | 優先懷疑層級 | 主要檢查 |
|---|---|---|
| 完全無輸出 | BootROM / power / reset / clock | Strap, boot media waveform, ROM header |
| TPL banner後停止 | TPL載入VPL / SPL | Media driver, offset, SRAM, raw entry |
| SPL早期停止 | Clock / PMIC / DDR | Early console, DDR training, power rail |
| SPL已顯示載入但無下一段 | Image驗證 / entry / TF-A | Hash, load address, BL31 / OpenSBI log |
| U-Boot proper可進shell | OS boot policy | Environment, FIT, FDT, bootargs |
| Kernel banner前停止 | Handoff / architecture | `bootm` / `booti`, entry, FDT, cache |
| Kernel可啟動但rootfs失敗 | Storage / bootargs | Root identity, driver, partition, filesystem |

## 9.3 U-Boot Driver Model

```mermaid
flowchart TB
    subgraph G1["1. 描述與綁定"]
        direction LR
        A1["Device Tree / Platform Data"] --> A2["Bind"] --> A3["Device / Driver / Uclass"]
    end
    subgraph G2["2. 硬體初始化"]
        direction LR
        B1["Probe"] --> B2["Clock / Reset / GPIO / Bus"] --> B3["初始化硬體"]
    end
    subgraph G3["3. 上層使用"]
        direction LR
        C1["Uclass Ops"] --> C2["Boot flow"] --> C3["Commands"]
    end
    G1 --> G2 --> G3
```

U-Boot Driver Model以device, driver與uclass管理硬體; Device Tree或platform data提供instance資料, driver提供硬體方法, uclass則定義同類裝置的共同介面. Bind先建立軟體關係, probe才取得資源並初始化硬體. 部分device採lazy probe, 因此列在device tree中不代表已完成硬體初始化.

```text
Device Tree / Platform Data
        ↓
Bind：建立device與driver關係
        ↓
Probe：取得clock、reset、GPIO、bus等資源
        ↓
Uclass Ops：供boot flow與commands使用
```

依build支援, 可用下列commands查看:

```bash
dm tree
dm uclass
dm drivers
dm static
```

TPL, SPL和U-Boot proper各自具有階段特定Kconfig. 完整U-Boot可用的driver, 不一定存在於SPL; SPL可用的driver也未必能塞進TPL的size budget.

## 9.4 U-Boot Device Tree

U-Boot可用control FDT設定自身Driver Model, 也會準備working FDT交給Linux. 兩者可能源自同一份DTS, 但可能經過phase裁減, board selection, FIT configuration, fixup或overlay而不同.

常見fixup包括memory size, MAC, serial, reserved-memory, bootargs與slot資訊. 排查時應同時保存U-Boot control DTB, FIT中的DTB, U-Boot working FDT及Linux `/sys/firmware/fdt`.

```bash
fdt addr <address>
fdt header
fdt print /chosen
fdt print /memory
fdt print <node-path>
```

## 9.5 U-Boot Environment 與 Boot Policy

Environment保存`bootcmd`, `bootargs`, load addresses, network parameters, slot與bootcount等變數. 它可位於SPI, eMMC, FAT, UBI或redundant environment區域. Persistent environment CRC錯誤時, U-Boot可能回到compiled-in default.

```bash
printenv
printenv bootcmd bootargs boot_targets
setenv <name> <value>
saveenv
```

`setenv`只改RAM中的runtime值; `saveenv`會寫入nonvolatile storage. Linux的`fw_printenv`設定必須與U-Boot environment offset, size, erase geometry及redundant layout一致.

## 9.6 映像載入與啟動 Linux

U-Boot可載入raw `Image` / `zImage`, legacy uImage, FIT, initramfs與EFI application. FIT可封裝kernel, DTB, ramdisk, 多個configuration, hash與signature.

```bash
bootm <fit-or-uimage-address>
booti <kernel-address> <ramdisk-or-dash> <fdt-address>
bootz <zimage-address> <ramdisk-or-dash> <fdt-address>
bootefi <image-address> <fdt-address>
```

最後傳給Linux的bootargs應以Linux `/proc/cmdline`驗證; 若與`printenv bootargs`不同, 需檢查boot script, extlinux, FIT與FDT fixup.

## 9.7 Standard Boot, A/B 與 Recovery

Standard Boot以bootdev, bootmeth與bootflow搜尋可啟動作業系統; 舊平台則可能由自訂`bootcmd`使用固定offset.

```bash
bootflow scan -lb
bootflow list
bootdev list
bootmeth list
```

A/B流程需共同定義slot priority, trial boot, bootcount, bootlimit, mark-good與rollback. U-Boot選擇slot, Linux / OpenBMC通常在達到success criteria後更新mark-good metadata.

Recovery可由strap, button, bootcount超限, image驗證失敗, watchdog或golden image policy進入, 並應能重新寫入production image.

## 9.8 U-Boot Target 排查與開發

基本資訊:

```bash
version
bdinfo
printenv
coninfo
dm tree
```

儲存與image:

```bash
mtd list
sf probe
mmc list
part list mmc 0
iminfo <image-address>
```

網路載入:

```bash
printenv ipaddr serverip gatewayip netmask
ping <server-ip>
dhcp
tftpboot <load-address> <filename>
```

Erase, write, `saveenv`與memory write commands具有持久或即時副作用. 一般排查先保存完整UART, U-Boot version, environment摘要, boot source, slot, load address, FIT configuration, 驗證結果, FDT address與最終bootargs.

## 9.9 Kernel Driver 是什麼

CPU 不會因為板子上焊了一顆 IC, 就自動知道如何使用它. 以 TMP75 溫度感測器為例, Linux 必須知道:

- 感測器接在哪一個 I2C bus 與 address.
- TMP75 有哪些暫存器.
- 要用什麼 I2C 交易讀取暫存器.
- 暫存器中的數值如何換算成溫度.
- 讀到的溫度要用什麼方式提供給其他程式.

其中, 裝置位置與硬體接線由 Device Tree 描述; 如何和 TMP75 通訊, 如何解讀資料, 則由 TMP75 driver 處理.

```text
Device Tree
描述 I2C bus 5、address 0x48 有一顆 TMP75
        ↓
TMP75 driver
讀取暫存器並換算溫度
        ↓
hwmon
用 Linux 共通格式提供溫度
        ↓
OpenBMC sensor service
建立 D-Bus sensor object
```

因此, Kernel driver 的核心工作是把特定硬體的控制方法接到 Linux kernel 的共同架構中.

### 9.9.1 Driver 如何控制硬體

不同硬體的控制方式不同, driver 可能透過下列方式與裝置溝通:

- 讀寫 SoC 的 MMIO 暫存器.
- 透過 I2C 或 SPI controller 傳送 bus transaction.
- 接收硬體 interrupt.
- 啟動或停止 DMA.
- 控制 GPIO, clock, reset 與 regulator.

Driver 初始化硬體時, 通常會:

1.  讀取 Device Tree 提供的資料.
2.  向 kernel 取得硬體需要的資源.
3.  確認硬體型號或目前狀態.
4.  設定暫存器與工作模式.
5.  註冊 hwmon, GPIO, network, MTD 等 Linux 介面.

### 9.9.2 Driver 與 OpenBMC 的分工

Driver 負責硬體的基本控制; 會因板子或產品而改變的使用方式, 通常由 Device Tree 或 OpenBMC service 決定.

例如:

| 內容                            | 負責位置              |
|---------------------------------|-----------------------|
| TMP75 的 I2C address            | Device Tree           |
| 如何讀取 TMP75 溫度暫存器       | Kernel driver         |
| 溫度以 millidegree Celsius 提供 | hwmon subsystem       |
| Sensor 顯示名稱與 threshold     | OpenBMC sensor config |
| 溫度過高時如何調整 fan          | Fan / thermal service |
| Service 失敗後如何重啟          | systemd unit          |

這樣分工後, 同一份 TMP75 driver 可以在不同板子上共用, 不需要為每一塊板子寫一份 driver.

## 9.10 Linux 如何管理裝置與 Driver

```mermaid
flowchart TB
    subgraph G1["Bus Core"]
        direction LR
        A1["Devices"] --> A3["Match"]
        A2["Drivers"] --> A3
    end
    subgraph G2["Driver Lifecycle"]
        direction LR
        B1["Probe"] --> B2["Bound"] --> B3["Remove / Unbind"]
    end
    subgraph G3["Kernel Interface"]
        direction LR
        C1["Subsystem"] --> C2["Class"] --> C3["sysfs / dev node"]
    end
    G1 --> G2 --> G3
```

Linux 不會讓每一份 driver 各自建立一套管理方式. Kernel 會用 device, driver, bus, subsystem 與 class 組織硬體. 這些部分的關係如下:

```text
Bus
├── 管理 devices
├── 管理 drivers
└── 決定 device 與 driver 是否相符
          ↓
       Probe
          ↓
Driver 初始化硬體
          ↓
Subsystem 提供共同介面
          ↓
Class 將同類功能整理到 /sys/class/
```

### 9.10.1 Device

Device 是 kernel 對一個裝置的記錄. 這個裝置可以是板子上的一顆 IC, 也可以是 SoC 內部的一個 controller.

例如:

```text
板子上的第 1 顆 TMP75
板子上的第 2 顆 TMP75
AST2600 的 I2C5 controller
SPI bus 上 CS0 的 flash
```

Kernel 會分別管理它們. 若四顆 TMP75 位於不同 bus 或 address, 可能看到:

```text
5-0048
5-0049
6-0048
6-0049
```

這四個 device 都可以交由同一份 TMP75 driver 控制, 但每個 device 仍有自己的 bus, address, 狀態與 sysfs 路徑.

### 9.10.2 Driver

Driver 是支援某一類 device 的 kernel 程式. 同一份 driver 可以管理多個相同或相容的 devices.

```text
5-0048 ─┐
5-0049 ─┼──> TMP75 driver
6-0048 ─┤
6-0049 ─┘
```

Driver 會為每個 device 分別保存所需資料, 例如 address, 校正值, clock, GPIO, interrupt 與目前工作狀態.

### 9.10.3 Bus

Bus 在硬體上可能是 I2C, SPI, PCI 或 USB; 在 Linux driver model 中, 它也代表一套管理 devices, drivers 與配對流程的規則.

例如 I2C bus 會處理:

- I2C device 的建立與命名.
- I2C driver 的註冊.
- Device 與 driver 的配對.
- I2C read / write API.

常見 bus:

| Bus      | 常見裝置                     | Device 通常如何出現               |
|----------|------------------------------|-----------------------------------|
| Platform | SoC 內部 GPIO, PWM, watchdog | Kernel 解析 Device Tree           |
| I2C      | Sensor, EEPROM, PMBus, CPLD  | I2C controller 下的 DT child node |
| SPI      | Flash, TPM, ADC              | SPI controller 下的 DT child node |
| PCI      | NIC, GPU, PCIe switch        | PCI controller 自動枚舉           |
| USB      | USB 裝置                     | USB controller 自動枚舉           |
| MDIO     | Ethernet PHY                 | MDIO scan 或 DT PHY node          |

### 9.10.4 Kernel Subsystem

不同 driver 控制硬體的方法不同, 但同類硬體需要提供一致的介面. Kernel subsystem 就是 Linux 為某一類功能建立的共同架構.

以溫度感測器為例:

```text
TMP75 driver ─────┐
LM75 driver ──────┼──> hwmon subsystem ──> 共同的溫度介面
PMBus driver ─────┘
```

TMP75, LM75 與 PMBus 裝置的讀取方式不同, 因此各自需要 driver; 但它們都可以把溫度註冊到 hwmon, 讓 userspace 使用相同的命名與單位讀取資料.

常見 subsystem:

| Subsystem | 提供的共同功能                  | Userspace 常見入口           |
|-----------|---------------------------------|------------------------------|
| hwmon     | 溫度, 電壓, 電流, 功率, fan RPM | `/sys/class/hwmon/`          |
| IIO       | ADC, DAC 與量測 channels        | `/sys/bus/iio/devices/`      |
| GPIO      | GPIO controller 與 lines        | `/dev/gpiochipX`             |
| MTD       | Raw flash 與 partitions         | `/dev/mtdX`, `/proc/mtd`     |
| Net       | Network interfaces              | `ip link`, `/sys/class/net/` |
| Watchdog  | Hardware watchdog               | `/dev/watchdogX`             |
| RTC       | Real-time clock                 | `/sys/class/rtc/`            |
| Input     | Button, key, switch             | `/dev/input/eventX`          |

Driver 處理特定硬體的差異; subsystem 則把同類硬體整理成共同介面.

### 9.10.5 Class

Class 是 sysfs 依功能整理裝置的方式. 同一顆 TMP75 從 I2C bus 的角度可以出現在:

```text
/sys/bus/i2c/devices/5-0048
```

它註冊成 hwmon 後, 從硬體監控功能的角度也會出現在:

```text
/sys/class/hwmon/hwmon2
```

兩條路徑可能指向同一個底層 device, 只是分類方式不同.

### 9.10.6 Probe

當 bus 判斷某個 driver 支援某個 device 時, kernel 會呼叫該 driver 的 `probe()`.

`probe()` 會嘗試取得資源, 確認硬體, 完成初始化, 並註冊 subsystem interface.

```text
Device 與 driver 配對成功
        ↓
Kernel 呼叫 probe()
        ↓
取得 clock、reset、GPIO、IRQ 等資源
        ↓
初始化硬體
        ↓
註冊 hwmon、netdev、MTD 等介面
```

Probe 回傳 `0` 表示成功. 若回傳負的 error code, 表示初始化失敗; 若回傳 `-EPROBE_DEFER`, 表示必要資源尚未準備好, kernel 之後會再嘗試.

## 9.11 從 Device Tree 到 Device

在 probe 以前, kernel 必須先知道"有這個 device". 以 TMP75 為例:

``` dts
&i2c5 {
    status = "okay";

    temperature-sensor@48 {
        compatible = "ti,tmp75";
        reg = <0x48>;
    };
};
```

開機時大致發生:

1.  Kernel 解析 running DTB.
2.  Kernel 看到 I2C5 controller, 建立對應的 platform device.
3.  I2C controller driver 成功 probe, 註冊一個 I2C adapter.
4.  I2C core 讀取 controller 底下的 child nodes.
5.  Kernel 根據 `temperature-sensor@48` 建立 I2C client device.
6.  該 device 在 sysfs 中通常以 bus number 與 address 表示, 例如 `5-0048`.

這個流程有明確的先後關係:

```text
I2C controller 先正常
    ↓
I2C bus 才存在
    ↓
Bus 下的 TMP75 device 才能建立
    ↓
TMP75 driver 才有機會 probe
```

如果 `/sys/bus/i2c/devices/5-0048` 完全不存在, 問題通常還沒走到 TMP75 driver 的 probe, 應先檢查 running DTB, I2C controller, MUX 與 device 建立流程.

### 9.11.1 Device Tree 不是唯一來源

不同 bus 建立 device 的方式不同:

- Platform, I2C, SPI 裝置常由 Device Tree 描述.
- PCI 與 USB 裝置通常由硬體自動枚舉.
- MDIO PHY 可由 Device Tree 描述, 也可能透過 PHY ID 偵測.
- 某些開發或測試流程可由 userspace 明確建立 I2C device, 但平台設定仍應有可追蹤來源.

`i2cdetect` 掃到 ACK, 只能證明某個 address 有硬體回應; 它不會自動告訴 kernel 那是 TMP75, 也不代表 driver 已經綁定.

## 9.12 Kernel 怎麼決定要不要呼叫 Probe

Probe 不是由 userspace 直接呼叫. 正常開機流程中, 是 bus core 在 device 與 driver 都註冊後, 自動比較兩者是否相符; 相符時才呼叫 driver 的 probe callback.

完整流程如下:

```text
Device 註冊到 bus
        +
Driver 註冊到同一個 bus
        ↓
Bus 執行 match 規則
        ↓
不相符：繼續找其他 driver
相符：呼叫這個 driver 的 probe()
        ↓
Probe 成功：綁定 device 與 driver
Probe 失敗：保留錯誤，介面通常不會出現
```

### 9.12.1 Platform Driver 的 Probe 如何被呼叫

DTS:

``` dts
example@1e600000 {
    compatible = "vendor,example-device";
    reg = <0x1e600000 0x1000>;
    status = "okay";
};
```

Driver:

```c
static const struct of_device_id example_of_match[] = {
    { .compatible = "vendor,example-device" },
    { }
};

static int example_probe(struct platform_device *pdev)
{
    return 0;
}

static struct platform_driver example_driver = {
    .probe = example_probe,
    .driver = {
        .name = "example-driver",
        .of_match_table = example_of_match,
    },
};

module_platform_driver(example_driver);
```

關鍵連結是:

```text
DTS compatible
"vendor,example-device"
        ↕
Driver of_match_table
"vendor,example-device"
```

`module_platform_driver()` 讓這份 platform driver 註冊到 platform bus. 當 platform device 與 driver 的 `compatible` 相符, platform bus 便呼叫 `example_probe(pdev)`.

### 9.12.2 I2C Driver 的 Probe 如何被呼叫

概念相同, 但資料型態換成 I2C:

```c
static const struct of_device_id example_of_match[] = {
    { .compatible = "vendor,example-sensor" },
    { }
};
MODULE_DEVICE_TABLE(of, example_of_match);

static int example_i2c_probe(struct i2c_client *client)
{
    /* client 內含 I2C adapter、address 與 struct device */
    return 0;
}

static struct i2c_driver example_i2c_driver = {
    .probe = example_i2c_probe,
    .driver = {
        .name = "example-sensor",
        .of_match_table = example_of_match,
    },
};
module_i2c_driver(example_i2c_driver);
```

當 I2C core 建立對應 client, 且 match 成功, 就會呼叫 `example_i2c_probe(client)`.

Kernel 版本不同時, I2C probe callback 的欄位名稱與函式簽名可能略有差異, 應以目前專案 kernel tree 的相似 driver 為準.

### 9.12.3 Built-in 與 Module 對 Probe 時機的影響

- Built-in driver 在 kernel 開機期間註冊; device 與 driver 都 ready 後便可 match, probe.
- Module driver 要等 module 載入並註冊 driver 後, 才有機會 probe 先前已存在的 device.
- Device 若比 driver 晚出現, 例如 hot-plug, device 註冊時也會重新尋找已註冊 driver.

所以不論"device 先, driver 後"或"driver 先, device 後", driver core 都能在兩者同時存在時嘗試配對.

### 9.12.4 如何確認流程走到哪裡

``` bash
# 1. Device 是否存在
ls -l /sys/bus/<bus>/devices/<device>

# 2. Driver 是否已註冊
ls -l /sys/bus/<bus>/drivers/<driver>

# 3. 是否完成綁定
readlink -f /sys/bus/<bus>/devices/<device>/driver

# 4. Match / probe 是否有訊息
dmesg | grep -Ei '<driver>|<device>|probe|fail|error|defer'
```

若 device 與 driver 都存在, 但沒有綁定, 常見方向包括:

- `compatible` 或 bus-specific ID 不相符.
- Module alias 不完整.
- Probe 被呼叫但失敗.
- Device 已被其他 driver 綁定.
- Driver 主動判斷 chip ID 不支援.

### 9.12.5 如何在除錯時重新觸發 Probe

部分 bus 可透過 unbind / bind 重新執行 remove 與 probe:

``` bash
echo '<device-name>' > /sys/bus/<bus>/drivers/<driver>/unbind
echo '<device-name>' > /sys/bus/<bus>/drivers/<driver>/bind
```

這不是一般產品啟動流程, 只是除錯方法. 使用前需確認:

- 該裝置不是 boot flash, console, watchdog 或 power-control 關鍵裝置.
- Driver 的 remove path 能安全停止 IRQ, timer, workqueue 與 DMA.
- OpenBMC service 已停止或能處理裝置暫時消失.

若 driver 是 module, 也可在安全條件下用 `modprobe -r` 與 `modprobe` 重新載入, 但同樣要先評估相依裝置與系統影響.

## 9.13 Probe 裡面實際做什麼

在看程式前, 先釐清 resource 的定義.

### 9.13.1 Resource 是什麼 [待確認: 原文編號為 9.5.1]

Resource 是 driver 控制硬體前必須取得的系統資源, 例如:

- MMIO address range.
- Interrupt line.
- Clock.
- Reset controller.
- GPIO.
- Regulator / power supply.
- DMA channel.

Driver 不是直接看到 DTS 就任意使用資源, 而是透過 kernel API 申請與取得. Kernel 由此避免兩個 driver 同時占用同一資源, 並管理相依順序.

### 9.13.2 `devm_*` 是什麼

`devm_*` 是 device-managed resource API.

> Driver 向 kernel 取得資源時, 把資源登記到目前 device 名下; 如果 probe 後面失敗, 或 device 日後被移除, kernel 會協助釋放已登記的資源.

例如:

```c
data = devm_kzalloc(dev, sizeof(*data), GFP_KERNEL);
```

它和一般 `kzalloc()` 的主要差異是:

```text
kzalloc()
Driver 通常要在錯誤路徑與 remove 中自行 kfree()

devm_kzalloc()
記憶體跟著 device 的生命週期，由 devres 機制管理
```

常見 managed APIs:

- `devm_kzalloc()`: 配置並歸零記憶體.
- `devm_platform_ioremap_resource()`: 取得並映射 MMIO resource.
- `devm_gpiod_get()`: 取得 GPIO descriptor.
- `devm_clk_get()`: 取得 clock.
- `devm_request_threaded_irq()`: 申請 IRQ.
- `devm_hwmon_device_register_with_info()`: 註冊 hwmon device.

`devm_*` 能減少清理程式, 但不是所有資源都能只靠它安全處理. Timer, workqueue, DMA, 硬體持續產生的 interrupt 與 asynchronous callback, 仍需在 remove 或 shutdown 流程中先停止.

### 9.13.3 一個 Probe 的常見步驟

```mermaid
flowchart TB
    subgraph G1["1. 建立軟體狀態"]
        direction LR
        A1["收到 device"] --> A2["配置 private data"] --> A3["讀取 properties"]
    end
    subgraph G2["2. 取得相依資源"]
        direction LR
        B1["MMIO"] --> B2["Clock / Reset"] --> B3["GPIO / IRQ / Regulator"]
    end
    subgraph G3["3. 初始化硬體"]
        direction LR
        C1["確認 chip ID / 狀態"] --> C2["設定 register"]
    end
    subgraph G4["4. 發布共同介面"]
        direction LR
        D1["註冊 subsystem interface"] --> D2["保存 driver data"] --> D3["回傳 0"]
    end
    G1 --> G2 --> G3 --> G4
```

```text
收到 device
    ↓
配置 driver 自己的資料結構
    ↓
讀取 DTS / device properties
    ↓
取得 MMIO、clock、reset、GPIO、IRQ 等 resources
    ↓
確認 chip ID 或硬體狀態
    ↓
設定硬體 register
    ↓
註冊 hwmon / GPIO / netdev 等介面
    ↓
保存 driver data
    ↓
回傳 0
```

### 9.13.4 逐行閱讀一個簡化 Probe

```c
static int example_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct example_data *data;
    int ret;

    data = devm_kzalloc(dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    data->base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(data->base))
        return PTR_ERR(data->base);

    data->clk = devm_clk_get_enabled(dev, NULL);
    if (IS_ERR(data->clk))
        return dev_err_probe(dev, PTR_ERR(data->clk),
                             "failed to get clock\n");

    data->reset = devm_reset_control_get_exclusive_deasserted(dev, NULL);
    if (IS_ERR(data->reset))
        return dev_err_probe(dev, PTR_ERR(data->reset),
                             "failed to get reset\n");

    ret = example_hw_init(data);
    if (ret)
        return dev_err_probe(dev, ret,
                             "hardware init failed\n");

    platform_set_drvdata(pdev, data);
    return 0;
}
```

逐段說明:

1.  `pdev` 是 platform bus 傳給 probe 的 device.
2.  `&pdev->dev` 取得通用的 `struct device`, 後續 resource API 多以它為入口.
3.  `struct example_data` 保存這個 device 專用的 driver 狀態.
4.  `devm_kzalloc()` 配置 private data; 失敗時回傳 `-ENOMEM`.
5.  `devm_platform_ioremap_resource(pdev, 0)` 取得 DTS `reg` 對應的第 0 組 MMIO resource, 確認未被占用, 並映射成 kernel 可以存取的位址.
6.  `IS_ERR()` 用來判斷 API 是否回傳 encoded error pointer; `PTR_ERR()` 取出負的 error code.
7.  `devm_clk_get_enabled()` 取得並啟用 clock. 若 provider 尚未 ready, 錯誤可能是 `-EPROBE_DEFER`.
8.  Reset API 取得並解除 reset.
9.  `example_hw_init()` 是此 driver 自己的硬體初始化, 例如讀 chip ID, 設定 register.
10. `platform_set_drvdata()` 把 `data` 存到 `pdev`, remove, IRQ handler 或其他 callback 可以再取回.
11. 回傳 `0` 表示 probe 成功.

### 9.13.5 為什麼常看到 `IS_ERR()` 與 `PTR_ERR()`

許多 kernel resource APIs 不以 `NULL` 表示所有錯誤, 而會回傳特殊的 error pointer.

常見寫法:

```c
resource = devm_some_get(dev);
if (IS_ERR(resource))
    return PTR_ERR(resource);
```

因此不能只寫:

```c
if (!resource)
    return -EINVAL;
```

否則可能漏掉 encoded error, 也可能破壞 `-EPROBE_DEFER`.

### 9.13.6 `dev_err_probe()` 是什麼

`dev_err_probe()` 同時處理錯誤訊息與錯誤碼, 特別適合 probe resource failure:

```c
return dev_err_probe(dev, PTR_ERR(data->clk),
                     "failed to get clock\n");
```

它的價值包括:

- 保留原始 error code.
- 對 `-EPROBE_DEFER` 使用較合適的記錄方式.
- 訊息會包含 device context.
- 讓 probe error path 較一致.

### 9.13.7 Probe 成功後還要註冊 Subsystem Interface

如果這是一個溫度 driver, 取得硬體資源並讀到溫度後, 通常還要註冊 hwmon:

```c
hwmon = devm_hwmon_device_register_with_info(dev,
                                              "example_temp",
                                              data,
                                              &example_chip_info,
                                              NULL);
if (IS_ERR(hwmon))
    return PTR_ERR(hwmon);
```

只有註冊成功後, userspace 才會看到對應的 `/sys/class/hwmon/hwmonX/`. 因此 probe 回傳 `0` 前, 通常要把 userspace 需要的 kernel interface 準備好.

### 9.13.8 常見 Probe 回傳值

| 回傳值          | 意義                       | 常見方向                  |
|-----------------|----------------------------|---------------------------|
| `0`             | 初始化成功                 | Device 與 driver 完成綁定 |
| `-ENODEV`       | 不是支援的裝置或裝置不存在 | Chip ID 不符              |
| `-EINVAL`       | 提供的設定不合理           | DT property 或參數錯誤    |
| `-ENOMEM`       | 無法配置記憶體             | 系統資源不足              |
| `-EBUSY`        | 資源已被占用               | GPIO, IRQ, MMIO 衝突      |
| `-ETIMEDOUT`    | 等不到硬體回應             | Power, clock, reset, bus  |
| `-EPROBE_DEFER` | 相依資源暫時還沒準備好     | Provider driver 尚未完成  |

### 9.13.9 Probe 成功要怎麼確認

不要只靠 driver 是否印出一行 log. 建議確認:

``` bash
# Device 與 driver 是否綁定
readlink -f /sys/bus/<bus>/devices/<device>/driver

# 預期 subsystem interface 是否出現
find /sys/class/hwmon -maxdepth 2 -type f -print

# 是否仍在 deferred list
cat /sys/kernel/debug/devices_deferred 2>/dev/null

# 是否有 probe error
dmesg | grep -Ei '<driver>|probe|fail|error|defer'
```

如果 driver 完成綁定, 但預期的 hwmon / netdev / MTD 介面沒有出現, 需確認 driver 是否真的有註冊該 subsystem, 註冊過程是否被條件跳過, 以及 sysfs path 是否判讀錯誤.

## 9.14 Driver 如何向 Kernel 登記自己

前面已經看到 `probe()`, 但 kernel 必須先知道"系統裡有這份 driver", 才可能呼叫它.

Driver 向 kernel 登記自己的動作稱為 register. 這裡的 register 不是硬體暫存器, 而是"把 driver 加入 kernel 的 driver 清單".

可以把流程理解成:

```text
Driver 載入
    ↓
向某一種 bus 登記
    ↓
告訴 bus：
- 我的名稱
- 我支援哪些 device
- Match 後要呼叫哪個 probe
- 移除時要呼叫哪個 remove
```

### 9.14.1 Platform Driver 的基本結構

```c
static const struct of_device_id example_of_match[] = {
    { .compatible = "vendor,example-device" },
    { }
};
MODULE_DEVICE_TABLE(of, example_of_match);

static int example_probe(struct platform_device *pdev)
{
    return 0;
}

static void example_remove(struct platform_device *pdev)
{
}

static struct platform_driver example_driver = {
    .probe = example_probe,
    .remove = example_remove,
    .driver = {
        .name = "example-driver",
        .of_match_table = example_of_match,
    },
};
module_platform_driver(example_driver);
```

逐項說明:

- `example_of_match[]`: 列出這份 driver 支援的 Device Tree `compatible`.
- `MODULE_DEVICE_TABLE()`: 將 match 資訊輸出成 module alias, 協助 module 自動載入.
- `example_probe()`: Match 成功後執行初始化.
- `example_remove()`: Device 與 driver 分離時執行清理.
- `struct platform_driver`: 把 callbacks, 名稱與 match table 整理成一份 platform driver.
- `module_platform_driver()`: 產生 driver register / unregister 所需的 module init 與 exit 程式.

### 9.14.2 Built-in Driver 也需要 Register

即使 driver 編進 kernel, 而不是 `. ko` module, 它仍需向對應 bus register. 差別只在發生時間:

```text
Built-in driver
Kernel 開機時 register

Module driver
Module 被載入時 register
```

Register 完成後, bus 才會拿這份 driver 去和目前已有的 devices 比對.

### 9.14.3 Remove 是什麼

Remove 是 device 不再由 driver 管理時的清理動作. 常見情況包括:

- Module 被卸載.
- Hot-plug device 被拔除.
- 工程師從 sysfs 執行 unbind.
- 某些 bus 重新建立 device.

Remove 需要確保:

1.  不再接受新的工作.
2.  停止 timer, workqueue 與 DMA.
3.  關閉硬體 interrupt source.
4.  等待正在執行的 callback 完成.
5.  移除非 managed 的 sysfs / debugfs 內容.
6.  將硬體留在安全狀態.

若 private data 已釋放, 但 IRQ 或 workqueue 仍會使用它, 就可能造成 use-after-free 與 kernel oops.

## 9.15 Driver 如何從 Device Tree 取得資料

Driver 不會直接解析整份 DTS 文字. Kernel 已將 DTB 解析成 device nodes, driver 再透過 kernel API 取得目前 device 的 properties 與 resources.

### 9.15.1 Property 與 Resource 的差異

Property 是 Device Tree 中的一筆設定, 例如:

``` dts
vendor,channel-count = <8>;
```

Driver 可透過 property API 讀取它.

Resource 則是需要由 kernel 管理的硬體資源, 例如 MMIO, IRQ, clock, GPIO. Driver 通常使用專用 subsystem API 取得, 而不是自行解析原始 cells.

### 9.15.2 讀取一般 Property

DTS:

``` dts
example@1e600000 {
    compatible = "vendor,example-device";
    vendor,channel-count = <8>;
};
```

Driver:

```c
u32 channels;
int ret;

ret = device_property_read_u32(dev,
                               "vendor,channel-count",
                               &channels);
if (ret)
    return dev_err_probe(dev, ret,
                         "missing channel count\n");
```

這裡:

- Property 名稱必須和 binding 一致.
- Driver 應處理 property 缺少或數值超出範圍的情況.
- 如果 property 不是硬體必要資訊, 應考慮提供合理預設值.

### 9.15.3 取得 MMIO

DTS:

``` dts
reg = <0x1e600000 0x1000>;
```

Driver:

```c
base = devm_platform_ioremap_resource(pdev, 0);
if (IS_ERR(base))
    return PTR_ERR(base);
```

Kernel 會:

1.  找到第 0 組 `reg` resource.
2.  確認範圍是否有效, 是否衝突.
3.  將 physical address 映射成 kernel 可存取的 virtual address.

Driver 後續通常透過 `readl()`, `writel()` 等 MMIO API 存取, 不應把 physical address 當成一般指標直接解參照.

### 9.15.4 取得 GPIO

DTS:

``` dts
reset-gpios = <&gpio0 12 GPIO_ACTIVE_LOW>;
```

Driver:

```c
reset = devm_gpiod_get(dev, "reset", GPIOD_OUT_LOW);
if (IS_ERR(reset))
    return dev_err_probe(dev, PTR_ERR(reset),
                         "failed to get reset GPIO\n");
```

名稱的對應規則是:

```text
Driver 使用 "reset"
        ↕
DTS 使用 reset-gpios
```

Driver 使用 GPIO descriptor API 時, set / get 的通常是 logical value; `GPIO_ACTIVE_LOW` 的反相會由 GPIO framework 處理.

### 9.15.5 取得 Clock, Reset, Regulator 與 IRQ

常見對應:

| Driver API 想取得的資源 | DTS 常見寫法                      | 意義            |
|-------------------------|-----------------------------------|-----------------|
| Clock 名稱 `core`       | `clocks` + `clock-names = "core"` | 提供裝置時脈    |
| Reset 名稱 `core`       | `resets` + `reset-names = "core"` | 控制硬體 reset  |
| Supply 名稱 `vdd`       | `vdd-supply = <&regulator>`       | 提供 power rail |
| IRQ                     | `interrupts`, `interrupt-parent`  | 硬體事件通知    |

實際 property 名稱, cell 數量與順序必須查 binding, 不能只看 API 名稱猜測.

## 9.16 Deferred Probe: 為什麼 Driver 會說"晚點再試"

```mermaid
flowchart BT
    subgraph G1["Provider Chain"]
        direction BT
        A1["Parent I2C controller"] --> A2["I2C MUX"] --> A3["GPIO expander"] --> A4["Enable GPIO"]
    end
    subgraph G2["Consumer"]
        direction TB
        B1["Sensor driver probe"] --> B2["資源未就緒: -EPROBE_DEFER"] --> B3["Provider ready 後重試"]
    end
    A4 --> B1
```

Deferred probe 的意思是:

> 這個 device 看起來由我支援, 但我現在缺少必要資源; 缺少的資源可能稍後才出現, 所以不要把它視為永久失敗.

Driver 以 `-EPROBE_DEFER` 表達這個狀態.

### 9.16.1 為什麼資源會晚一點出現

假設 sensor 的 enable GPIO 來自 I2C GPIO expander:

```text
Sensor driver
需要 enable GPIO
    ↓
GPIO expander driver
要先 probe
    ↓
I2C MUX driver
要先建立下游 bus
    ↓
Parent I2C controller
要先 probe
```

Kernel 會平行與分階段初始化許多 devices, 不保證 consumer 第一次 probe 時 provider 已就緒. 因此 deferred probe 是正常的 dependency 處理機制.

### 9.16.2 短暫 Deferred 與長期 Deferred

開機初期短暫 deferred, 後來重試成功, 通常不表示有問題.

開機完成後仍列在 `devices_deferred`, 才需要排查:

``` bash
cat /sys/kernel/debug/devices_deferred
```

常見原因:

- Provider node 被設為 `disabled`.
- Provider driver 的 kernel config 未啟用.
- Provider 是 module, 但 module 沒進 image.
- Phandle 指錯節點.
- Provider 自己 probe 失敗.
- 必要硬體實際不存在.

### 9.16.3 Driver 中如何保留 Deferred Probe

不應把所有 resource error 都改成 `-EINVAL`:

```c
clk = devm_clk_get(dev, NULL);
if (IS_ERR(clk))
    return dev_err_probe(dev, PTR_ERR(clk),
                         "failed to get clock\n");
```

`PTR_ERR(clk)` 若是 `-EPROBE_DEFER`, 應原樣回傳, driver core 才知道需要稍後重試.

## 9.17 Kernel Config, Built-in 與 Module

Driver source 存在於 kernel tree, 不代表目前產品的 kernel 有包含它. Kconfig 決定一項功能是否進入本次 build.

### 9.17.1 三種常見狀態

```text
CONFIG_FOO=y
Driver 直接編入 kernel，稱為 built-in

CONFIG_FOO=m
Driver 編成 foo.ko，稱為 module

# CONFIG_FOO is not set
這次 kernel 沒有這個 driver
```

### 9.17.2 何時使用 Built-in

通常適合開機早期不可缺少的功能:

- Boot flash.
- Root filesystem 所在 storage.
- UART console.
- Clock, reset, pinctrl 等基礎 provider.
- Rootfs 依賴的 I2C / SPI / MMC controller.

若 rootfs 要靠某 driver 才能讀取, 那份 driver 不能只放在 rootfs 裡的 module, 否則 kernel 會陷入"要先讀 rootfs 才拿得到讀 rootfs 的 driver".

### 9.17.3 Module 是什麼

Module 是可在 kernel 執行期間載入的 `. ko` 檔. 載入後, 它會執行 register; 卸載時則 unregister.

``` bash
modprobe <module>
modprobe -r <module>
lsmod
modinfo <module>
```

Module 適合非開機必要或選配功能, 但需同時確認:

- `. ko` 有建出來.
- Module package 有放入 image.
- Module 相依關係完整.
- Alias 能讓 module 自動載入, 或系統有明確載入設定.
- OpenBMC service 不會早於 module 啟動.

### 9.17.4 Target 與 Build 端檢查

Target:

``` bash
zcat /proc/config.gz | grep CONFIG_<OPTION>
lsmod
modinfo <module>
find /lib/modules/$(uname -r) -name '*<module>*'
```

Build 端:

``` bash
bitbake -e virtual/kernel | grep -E '^(S|B|WORKDIR)='
bitbake -c configure -f virtual/kernel
find tmp/work -path '*linux*' -name '.config' -print
```

要確認最終 `. config`, 不能只看到 config fragment 就假設已成功套用.

## 9.18 Driver 如何把資料交給 Userspace

Driver 完成硬體初始化後, 通常不會要求每個 userspace service 都使用 driver 私有格式. 它會註冊到適合的 kernel subsystem, 取得共同介面.

### 9.18.1 Sysfs 是什麼

Sysfs 是 kernel 將 devices, drivers, classes 與簡單 attributes 顯示成檔案樹的介面, 通常掛載於 `/sys`.

例如:

```text
/sys/bus/i2c/devices/5-0048/
```

從 I2C bus 角度表示這個 device.

```text
/sys/class/hwmon/hwmon2/
```

從硬體監控功能角度表示同一 device 提供的 hwmon 介面.

Sysfs 檔案看起來像一般檔案, 但內容通常是 kernel callback 即時產生或接收, 不是儲存在磁碟上的普通檔案.

### 9.18.2 hwmon 是什麼

Hwmon 是 hardware monitoring subsystem, 統一溫度, 電壓, 電流, 功率與 fan speed 的 naming 與單位.

常見 attributes:

```text
temp1_input
in1_input
curr1_input
power1_input
fan1_input
pwm1
```

`hwmonX` 的 X 可能因 probe 順序改變, OpenBMC service 應搭配 `name`, label 與 device path 找到正確裝置, 不應只寫死 `hwmon2`.

### 9.18.3 IIO 是什麼

IIO 是 Industrial I/O subsystem, 常用於 ADC, DAC 與各類量測 channels. 它可提供 raw value, scale, offset 等資訊.

```text
/sys/bus/iio/devices/iio:deviceX/
```

若 ADC 讀到 raw value, 但最終電壓錯誤, 需繼續確認:

- IIO scale.
- 電路分壓比例.
- OpenBMC sensor 設定.
- 單位轉換.

### 9.18.4 Device Node 是什麼

有些 kernel interface 透過 `/dev` 提供:

```text
/dev/gpiochip0
/dev/watchdog0
/dev/mtd0
/dev/input/event0
```

Userspace 透過 `read()`, `write()`, `ioctl()` 或 subsystem library 使用它. 存取可能受到檔案權限, service user, Linux capability 與裝置使用狀態限制.

### 9.18.5 Debugfs 是什麼

Debugfs 是 kernel 除錯介面, 通常掛載於 `/sys/kernel/debug`. 它提供大量內部狀態, 但不保證穩定 ABI.

常見項目:

- `devices_deferred`
- `clk/clk_summary`
- `pinctrl/`
- Dynamic debug control
- Ftrace

正式 OpenBMC service 不應把 debugfs 當成長期穩定介面.

### 9.18.6 常見 Subsystems 對照

| Subsystem | 它統一什麼                  | Userspace 入口               |
|-----------|-----------------------------|------------------------------|
| hwmon     | 溫度, 電壓, 電流, 功率, RPM | `/sys/class/hwmon/`          |
| IIO       | ADC, DAC, 量測 channels     | `/sys/bus/iio/devices/`      |
| GPIO      | GPIO chips 與 lines         | `/dev/gpiochipX`             |
| MTD       | Raw flash 與 partitions     | `/dev/mtdX`, `/proc/mtd`     |
| Net       | Network interfaces          | `ip link`, `/sys/class/net/` |
| Watchdog  | Hardware watchdog           | `/dev/watchdogX`             |
| Input     | Buttons, keys, switches     | `/dev/input/eventX`          |
| RTC       | Real-time clock             | `/sys/class/rtc/`            |

## 9.19 OpenBMC 如何使用 Kernel Interface

```mermaid
flowchart TB
    subgraph G1["Kernel"]
        direction LR
        A1["Driver"] --> A2["hwmon / IIO / GPIO / MTD / net"]
    end
    subgraph G2["OpenBMC Userspace"]
        direction LR
        B1["Service"] --> B2["Platform configuration"] --> B3["D-Bus object"]
    end
    subgraph G3["External Management"]
        direction LR
        C1["Redfish"] --> C3["Management client"]
        C2["IPMI"] --> C3
    end
    G1 --> G2 --> G3
```

OpenBMC service 通常位於 kernel 與 Redfish / IPMI 之間. 它會讀取 kernel interface, 套用平台設定, 再建立 D-Bus objects.

```text
Kernel driver
    ↓
Kernel subsystem interface
    ↓
OpenBMC service
    ↓
D-Bus object
    ↓
Redfish / IPMI
```

### 9.19.1 以溫度 Sensor 為例

```text
TMP75 driver
    ↓
/sys/class/hwmon/.../temp1_input
    ↓
OpenBMC sensor service
    ↓
/xyz/openbmc_project/sensors/temperature/...
    ↓
Redfish Thermal Sensor
```

每一層的責任:

- Driver: 正確讀取 TMP75.
- Hwmon: 用規定名稱與單位提供溫度.
- Sensor 設定: 決定名稱, threshold, power-state 條件等平台內容.
- OpenBMC service: 週期性讀取, 更新 D-Bus, 產生 threshold events.
- Redfish / IPMI: 向外提供管理介面.

### 9.19.2 為什麼 Sysfs 有值, D-Bus 還是可能沒有

這表示 kernel 前半段大致可用, 但 userspace 尚有其他條件:

- Entity Manager 或 sensor config 沒匹配到裝置.
- Hwmon label / channel 與設定不同.
- Service 尚未啟動或反覆失敗.
- Power-state gating 暫時不建立 sensor.
- Service user 無權存取 device node.
- D-Bus name 或 object path 與上層期待不同.

此時不宜繼續修改 driver, 應先看 service journal, 設定檔與 D-Bus tree.

### 9.19.3 分層排查順序

```text
1. Running DTB 有 node 嗎？
2. Kernel 建立 device 了嗎？
3. Driver 綁定並 probe 成功了嗎？
4. Subsystem interface 有正確資料嗎？
5. OpenBMC service 有讀到嗎？
6. D-Bus object 有建立嗎？
7. Redfish / IPMI 有映射嗎？
```

這個順序可避免看到 Redfish 缺資料, 就同時修改 DTS, driver, JSON 與 Web service.

## 9.20 如何判斷該改 DTS, Driver 或 OpenBMC 設定

| 現象或需求              | 優先檢查位置             | 原因                |
|-------------------------|--------------------------|---------------------|
| I2C address 寫錯        | DTS                      | 屬於板級接線資訊    |
| GPIO polarity 寫錯      | DTS / schematic          | 屬於硬體描述        |
| Driver 不認識新 chip ID | Driver                   | 屬於硬體控制支援    |
| 需要新增 DT property    | Binding + driver + DTS   | 三者必須同步        |
| Sensor threshold 不同   | OpenBMC sensor config    | 依產品需求決定      |
| Fan curve 不同          | Fan / thermal service    | 屬於控制規則        |
| Service 太早啟動        | systemd dependencies     | 屬於啟動順序        |
| Kernel config 沒開      | Kernel fragment / recipe | Driver 未進入 build |

判斷重點:

```text
固定硬體接線 → DTS
硬體控制方法 → Driver
共同介面 → Kernel subsystem
平台名稱、threshold、控制規則 → OpenBMC userspace
```

## 9.21 Driver 開發與 Yocto 流程

### 9.21.1 開始修改前

1.  確認 kernel tree 是否已有對應 driver.
2.  閱讀 binding, datasheet 與 schematic.
3.  找同 subsystem 中相似的 driver.
4.  確認目前問題位於 device 建立, match, probe 還是 userspace.
5.  先保存未修改前的 dmesg, sysfs 與 D-Bus 狀態.

### 9.21.2 使用 Devtool

``` bash
bitbake -e virtual/kernel | grep -E '^(PN|FILE|S|B)='
devtool modify virtual/kernel
```

在 workspace 修改並建置:

``` bash
devtool build virtual/kernel
```

收回平台 layer 前, 確認實際 kernel recipe 與專案流程:

``` bash
devtool finish virtual/kernel ../meta-my-platform
```

部分 branch 可能需使用實際 recipe 名稱, 應以 `bitbake -e virtual/kernel` 與 `devtool --help` 為準.

### 9.21.3 驗證順序

1.  Driver 可編譯.
2.  Kernel / module 可部署.
3.  Device 與 driver 能綁定.
4.  Probe log 無錯誤.
5.  Subsystem interface 正確.
6.  實體訊號與數值正確.
7.  OpenBMC service 與 D-Bus 正確.
8.  BMC reboot, AC cycle 與 service restart 正常.
9.  其他共用此 driver 的 boards 完成回歸.

### 9.21.4 Patch Review 重點

- Error code 是否保留, 尤其 `-EPROBE_DEFER`.
- 必要與 optional resources 是否區分.
- Error path 是否完整.
- Remove 前是否停止 IRQ, timer, workqueue, DMA.
- Register access 是否使用適當 API 與 locking.
- Sysfs 單位是否符合 subsystem ABI.
- Board-specific 規則是否誤寫進通用 driver.
- Log 是否會在每次 polling 時持續刷出.
- Suspend, resume, shutdown, reboot 行為是否安全.

## 9.22 Dynamic Debug 與 Ftrace 是什麼

一般 `dmesg` 只顯示 driver 主動輸出的訊息. 若資訊不夠, 可使用 kernel 提供的除錯機制.

### 9.22.1 Dynamic Debug

Dynamic debug 可在不重編 driver 的情況下, 開啟原始碼中的 `pr_debug()` 或 `dev_dbg()` 訊息.

``` bash
mount | grep debugfs || mount -t debugfs debugfs /sys/kernel/debug

grep '<driver>' /sys/kernel/debug/dynamic_debug/control

echo 'file drivers/hwmon/<driver>.c +p' \
    > /sys/kernel/debug/dynamic_debug/control
```

完成後關閉:

``` bash
echo 'file drivers/hwmon/<driver>.c -p' \
    > /sys/kernel/debug/dynamic_debug/control
```

若 driver 原始碼沒有 `pr_debug()` / `dev_dbg()` callsites, 開啟 dynamic debug 也不會產生額外內容.

### 9.22.2 Ftrace

Ftrace 可記錄 kernel functions 的呼叫. 它的用途不是"顯示更多 printk", 而是觀察函式是否被呼叫, 呼叫順序與執行時間.

``` bash
cd /sys/kernel/debug/tracing

echo 0 > tracing_on
echo function_graph > current_tracer
echo '<function_name>' > set_ftrace_filter
echo 1 > tracing_on
sleep 3
echo 0 > tracing_on
cat trace > /tmp/ftrace-driver.txt
```

清理設定:

``` bash
echo nop > current_tracer
: > set_ftrace_filter
```

Tracing 本身會改變 timing. 應限縮 functions 與時間, 避免在量產環境長時間開啟.

## 9.23 Kernel Oops, Panic 與 Race 是什麼

### 9.23.1 Oops 與 Panic

Kernel oops 表示 kernel 偵測到嚴重錯誤並輸出診斷資訊; 系統有時仍能繼續執行, 但狀態可能不再可靠.

Kernel panic 表示 kernel 無法安全繼續, 通常會停止或依設定重新啟動.

### 9.23.2 Race Condition

Race condition 是結果取決於兩個執行流程先後順序的問題. 例如 remove 正在釋放 private data, 但 workqueue 同時仍使用它.

常見 driver 問題:

| 現象 | 白話說明 | 常見方向 |
|----|----|----|
| Null pointer | 使用尚未取得或已清空的指標 | Optional resource, error path |
| Use-after-free | 資料釋放後仍被 callback 使用 | IRQ, timer, workqueue, remove |
| IRQ storm | Interrupt 持續高速觸發 | Trigger / status clear 錯誤 |
| Sleeping in atomic | 不可睡眠的 context 呼叫會睡眠的 API | IRQ, spinlock |
| Lockdep warning | Lock 取得順序可能造成 deadlock | Locking design |
| Hung task | Task 長時間無法前進 | I/O timeout, mutex, workqueue |

### 9.23.3 需要保存的資料

``` bash
dmesg -T > /tmp/dmesg.txt
journalctl -k -b --no-pager > /tmp/journal-current.txt
journalctl -k -b -1 --no-pager > /tmp/journal-previous.txt 2>&1
cat /proc/modules > /tmp/modules.txt
cat /proc/interrupts > /tmp/interrupts.txt
```

符號分析還需要: kernel commit, config, 未 strip 的 `vmlinux`, modules, toolchain 與完整 oops 前後文.

## 9.24 Target 端完整排查流程

```mermaid
flowchart TB
    subgraph G1["1. 建置與部署"]
        direction LR
        A1["DTS / Kernel config"] --> A2["Kernel / module"] --> A3["Image / slot"]
    end
    subgraph G2["2. Kernel 裝置建立"]
        direction LR
        B1["Running DTB"] --> B2["Parent controller"] --> B3["Device"]
    end
    subgraph G3["3. Driver 初始化"]
        direction LR
        C1["Register"] --> C2["Match"] --> C3["Probe / Dependencies"]
    end
    subgraph G4["4. 共同介面"]
        direction LR
        D1["Subsystem interface"] --> D2["數值 / 狀態驗證"]
    end
    subgraph G5["5. OpenBMC"]
        direction LR
        E1["Service / Journal"] --> E2["D-Bus"] --> E3["Redfish / IPMI"]
    end
    G1 --> G2 --> G3 --> G4 --> G5
```

以下小節依此順序逐層確認. 前一層未成立時, 先處理該層, 避免同時修改後續設定.

### 9.24.1 確認 Running DTB

``` bash
tr '\0' '\n' < /proc/device-tree/compatible
cp /sys/firmware/fdt /tmp/running.dtb
```

確認實際 node, compatible, address 與 resources, 不只看 source DTS.

### 9.24.2 確認 Driver 有進 Kernel

``` bash
zcat /proc/config.gz | grep CONFIG_<OPTION>
modinfo <module> 2>/dev/null
find /lib/modules/$(uname -r) -name '*<module>*' 2>/dev/null
```

### 9.24.3 確認 Device 已建立

``` bash
find /sys/bus/platform/devices -maxdepth 1 -print
find /sys/bus/i2c/devices -maxdepth 1 -print
find /sys/bus/spi/devices -maxdepth 1 -print
```

Device 不存在時, 先回查 parent controller, DT node, bus enumeration, 而不是先修改 probe.

### 9.24.4 確認 Driver 已綁定

``` bash
readlink -f /sys/bus/<bus>/devices/<device>/driver
cat /sys/bus/<bus>/devices/<device>/modalias
```

### 9.24.5 確認 Probe 與 Dependencies

``` bash
dmesg | grep -Ei '<driver>|<device>|probe|defer|fail|error'
cat /sys/kernel/debug/devices_deferred 2>/dev/null
```

### 9.24.6 確認 Subsystem Interface

``` bash
find /sys/class/hwmon -maxdepth 2 -type f -print | sort
find /sys/bus/iio/devices -maxdepth 2 -type f -print 2>/dev/null | sort
gpioinfo 2>/dev/null
cat /proc/mtd 2>/dev/null
ip link
find /sys/class/watchdog -maxdepth 2 -type f -print 2>/dev/null
```

### 9.24.7 確認 OpenBMC Service 與 D-Bus

``` bash
systemctl --failed
journalctl -b --no-pager | grep -Ei '<service>|sensor|entity|inventory'
busctl tree
```

## 9.25 常見現象如何判讀

| 現象 | 流程大約停在哪裡 | 優先檢查 |
|----|----|----|
| `/sys/bus/.../devices` 沒有 device | Device 建立以前 | Running DTB, parent controller |
| Device 有但沒有 `driver` symlink | Match 或 probe | Compatible, module, dmesg |
| 長期 deferred | Probe 缺 provider | `devices_deferred`, provider config |
| Driver 綁定但沒有 hwmon | Subsystem 註冊 | Driver code, probe condition |
| Hwmon 有值但 D-Bus 沒值 | OpenBMC service | JSON, label, journal, power state |
| GPIO busy | Resource 已被其他 consumer 使用 | `gpioinfo`, hog, 其他 driver |
| IRQ count 不增加 | Interrupt 到不了 driver | Pinctrl, IRQ type, 硬體波形 |
| IRQ count 快速增加 | Interrupt 未清或 type 錯 | Status clear, level signal |
| Network device 有但 link down | MAC 後半段 | MDIO, PHY, reset, clock, NC-SI |
| Watchdog reset | Feed 或系統執行卡住 | Previous boot journal, reset reason |
| Kernel oops | Driver 存取非法資料 | 完整 oops, symbols, race |

## 9.26 Kernel Driver Debug Log 收集

以下腳本以讀取狀態為主, 不主動 unbind driver 或改寫硬體:

``` bash
#!/bin/sh

OUT=/tmp/kernel-driver-debug
mkdir -p "$OUT"

cat /etc/os-release > "$OUT/os-release.txt" 2>&1
uname -a > "$OUT/uname.txt"
cat /proc/cmdline > "$OUT/proc-cmdline.txt"
zcat /proc/config.gz > "$OUT/proc-config.txt" 2>&1

dmesg -T > "$OUT/dmesg.txt"
journalctl -k -b --no-pager > "$OUT/journal-kernel-current.txt" 2>&1
journalctl -k -b -1 --no-pager > "$OUT/journal-kernel-previous.txt" 2>&1
journalctl -b --no-pager > "$OUT/journal-current.txt" 2>&1
systemctl --failed > "$OUT/systemctl-failed.txt" 2>&1

find /sys/bus/platform/devices -maxdepth 1 -print > "$OUT/platform.txt" 2>&1
find /sys/bus/i2c/devices -maxdepth 2 -print > "$OUT/i2c.txt" 2>&1
find /sys/bus/spi/devices -maxdepth 2 -print > "$OUT/spi.txt" 2>&1
find /sys/class/hwmon -maxdepth 3 -type f -print > "$OUT/hwmon.txt" 2>&1
find /sys/bus/iio/devices -maxdepth 3 -type f -print > "$OUT/iio.txt" 2>&1

mount | grep debugfs >/dev/null 2>&1 || \
    mount -t debugfs debugfs /sys/kernel/debug

cat /sys/kernel/debug/devices_deferred > "$OUT/deferred.txt" 2>&1
cat /sys/kernel/debug/gpio > "$OUT/gpio.txt" 2>&1
cat /sys/kernel/debug/clk/clk_summary > "$OUT/clocks.txt" 2>&1
find /sys/kernel/debug/pinctrl -maxdepth 3 -type f -print > "$OUT/pinctrl.txt" 2>&1

cat /proc/interrupts > "$OUT/interrupts.txt"
cat /proc/iomem > "$OUT/iomem.txt"
cat /proc/modules > "$OUT/modules.txt"
lsmod > "$OUT/lsmod.txt" 2>&1

tar czf "/tmp/kernel-driver-debug-$(date +%Y%m%d-%H%M%S).tar.gz" \
    -C /tmp kernel-driver-debug
```

執行前應確認儲存空間與專案資料管理要求; journal, MAC address, 裝置名稱等內容可能包含平台資訊.

## 9.27 建議 Bring-up 順序

1.  確認 running DTB.
2.  確認 kernel config.
3.  確認 parent controller.
4.  確認 device 已建立.
5.  確認 driver 已 register 並完成 match.
6.  確認 probe 成功.
7.  清查 deferred probes.
8.  確認 subsystem interface 與實際數值.
9.  再檢查 OpenBMC service.
10. 確認 D-Bus, Redfish, IPMI.
11. 測試 service restart, BMC reboot, AC cycle, host state transition.
12. 保存 DTS, kernel, image 版本與 log.

## 9.28 平台實測紀錄表

| 項目            | 指令 / 來源             | 實測值   | 備註                       |
|-----------------|-------------------------|----------|----------------------------|
| Kernel version  | `uname -a`              | \[待填\] | 對應 commit                |
| Kernel config   | `/proc/config. gz`       | \[待填\] | Driver 為 y / m / disabled |
| Running DTB     | `/sys/firmware/fdt`     | \[待填\] | SHA-256                    |
| Device path     | `/sys/bus/*/devices`    | \[待填\] | Bus 與 device name         |
| Bound driver    | `driver` symlink        | \[待填\] | Driver name                |
| Modalias        | Device `modalias`       | \[待填\] | Module match               |
| Probe log       | `dmesg -T`              | \[待填\] | 完整錯誤碼                 |
| Deferred probe  | `devices_deferred`      | \[待填\] | Provider / owner           |
| Subsystem       | hwmon / IIO / net / MTD | \[待填\] | Interface path             |
| GPIO            | `gpioinfo`              | \[待填\] | Line / consumer            |
| IRQ             | `/proc/interrupts`      | \[待填\] | Idle / event count         |
| Clock           | `clk_summary`           | \[待填\] | Rate / enable count        |
| OpenBMC service | systemd / journal       | \[待填\] | Service state              |
| D-Bus object    | `busctl tree`           | \[待填\] | Object / interface         |
| Redfish / IPMI  | API / command           | \[待填\] | 與 D-Bus 對照              |

## 9.29 驗收 Checklist

### 基本流程

- [ ] 能說明 device, driver, bus, subsystem, class, probe 的差異.
- [ ] Running DTB 包含預期 node.
- [ ] Kernel config 包含 driver.
- [ ] Device 出現在正確 bus.
- [ ] Driver 已 register, match 並成功 probe.
- [ ] Device 的 `driver` symlink 正確.
- [ ] Deferred probe 已清空或有明確原因.

### 資源與介面

- [ ] MMIO, IRQ, clock, reset, GPIO, regulator 與 binding / schematic 一致.
- [ ] Probe error code 與 resource failure 已釐清.
- [ ] Subsystem interface 已出現.
- [ ] Hwmon / IIO 單位與數值合理.
- [ ] 沒有寫死會變動的 `hwmonX` / `iio: deviceX`.
- [ ] 正式 service 不依賴 debugfs.

### OpenBMC

- [ ] Service 能讀取 kernel interface.
- [ ] D-Bus object 與 properties 正確.
- [ ] Redfish / IPMI 與 D-Bus 一致.
- [ ] Power-state gating, presence 與 service restart 已測試.

### Driver 品質

- [ ] `devm_*` 與手動清理責任清楚.
- [ ] Remove 會停止 IRQ, timer, workqueue 與 DMA.
- [ ] `-EPROBE_DEFER` 沒被錯誤改寫.
- [ ] Board-specific 規則未塞入通用 driver.
- [ ] Dynamic debug / ftrace 測試後已關閉.
- [ ] Oops / panic 所需 symbols 與 logs 可取得.

## 9.30 U-Boot 與 Kernel 本章重點

1.  Driver 是 kernel 中知道如何控制某類硬體的程式.
2.  Device 是 kernel 為一個裝置建立的管理物件.
3.  Bus 提供 device 與 driver 的管理及 match 規則.
4.  Subsystem 把不同 drivers 提供的同類功能整理成共同介面.
5.  Probe 是 match 成功後, 由 kernel 呼叫 driver 初始化 device 的動作.
6.  `devm_*` 將資源生命週期與 device 綁定, 但 asynchronous 工作仍需先停止.
7.  Device 不存在, match 不成立, probe 失敗與 subsystem 未註冊是四個不同階段.
8.  Deferred probe 表示 provider 暫時未 ready, 需沿 dependency 排查.
9.  Kernel interface 正常後, 才進一步檢查 OpenBMC service, D-Bus 與 Redfish / IPMI.
10. 排查時每個名詞都要對應到 target 上可確認的路徑或 log.

## 9.31 本章參考資料
- U-Boot 官方文件: https://docs.u-boot.org/en/latest/
- U-Boot Booting from TPL / SPL: https://docs.u-boot.org/en/stable/usage/spl_boot.html
- U-Boot Generic xPL framework: https://docs.u-boot.org/en/stable/develop/spl.html
- U-Boot Board Initialisation Flow: https://docs.u-boot.org/en/stable/develop/init.html
- U-Boot Device Tree Control: https://docs.u-boot.org/en/latest/develop/devicetree/control.html
- U-Boot Packaging與Binman: https://docs.u-boot.org/en/latest/develop/package/index.html

- Linux kernel documentation - Driver Model: https://docs.kernel.org/driver-api/driver-model/index.html
- Linux kernel documentation - Device Drivers: https://docs.kernel.org/driver-api/driver-model/driver.html
- Linux kernel documentation - Platform Devices and Drivers: https://docs.kernel.org/driver-api/driver-model/platform.html
- Linux kernel documentation - Devres: https://docs.kernel.org/driver-api/driver-model/devres.html
- Linux kernel documentation - I2C and SMBus Subsystem: https://docs.kernel.org/i2c/
- Linux kernel documentation - HWMON Subsystem: https://docs.kernel.org/hwmon/
- Linux kernel documentation - GPIO: https://docs.kernel.org/driver-api/gpio/
- Linux kernel documentation - Dynamic Debug: https://docs.kernel.org/admin-guide/dynamic-debug-howto.html
- Linux kernel documentation - Ftrace: https://docs.kernel.org/trace/ftrace.html
- Yocto Project Kernel Development Manual: https://docs.yoctoproject.org/kernel-dev/
