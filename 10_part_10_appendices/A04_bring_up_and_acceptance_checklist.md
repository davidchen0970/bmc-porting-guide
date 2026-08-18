# 附錄 A04：Bring-up 與驗收 Checklist

本附錄提供跨章節的 bring-up 與驗收檢查表。建議每個項目都保留 owner、版本、log 路徑、量測工具與判定日期。

## A04.1 最小可開機路徑

| 項目 | 檢查內容 | 證據 | 狀態 |
| --- | --- | --- | --- |
| Standby power | BMC standby rail 穩定，PGOOD 正常 | scope、DMM、CPLD register | 待填 |
| Reset release | BMC reset 於預期時間釋放 | scope、reset reason | 待填 |
| Clock | main oscillator / clock source 正常 | scope、clk summary | 待填 |
| Boot media | BootROM 可讀 boot flash / eMMC | LA、UART、U-Boot log | 待填 |
| U-Boot | 可進 U-Boot，env 合理 | UART、printenv | 待填 |
| Kernel | kernel 啟動無重大 panic | UART、dmesg | 待填 |
| Rootfs | rootfs mount 成功 | `/proc/cmdline`、findmnt | 待填 |
| Userspace | systemd default target 可到達 | `systemctl --failed` | 待填 |
| 管理介面 | SSH / Redfish / IPMI 依需求可用 | curl、ipmitool、journal | 待填 |

## A04.2 硬體抽象層

| 項目 | 檢查內容 | 證據 | 狀態 |
| --- | --- | --- | --- |
| Pinmux | 關鍵 peripheral pinmux 與 DTS 一致 | pinctrl debugfs、dmesg | 待填 |
| GPIO line name | gpio-line-name 與 schematic 對齊 | gpioinfo、對照表 | 待填 |
| Active level | presence、fault、reset、WP 極性已實測 | scope、gpioget、D-Bus | 待填 |
| Power enable | reset default 落在安全狀態 | scope、CPLD register | 待填 |
| Reset domain | BMC reset / host reset / peripheral reset 邊界清楚 | waveform、reset reason | 待填 |
| Clock domain | clock source、enable、parent rate 已確認 | scope、clk summary | 待填 |
| CPLD | register map、default、clear rule 已驗證 | dump、journal | 待填 |

## A04.3 儲存與更新

| 項目 | 檢查內容 | 證據 | 狀態 |
| --- | --- | --- | --- |
| Partition layout | DTS / mtdparts / UBI / GPT 與文件一致 | `/proc/mtd`、lsblk、ubinfo | 待填 |
| Filesystem | rofs / rwfs / overlay 掛載符合設計 | findmnt、df、dmesg | 待填 |
| Persistent data | 更新後保留範圍符合政策 | before/after diff | 待填 |
| Factory reset | 清除範圍與保留範圍已驗證 | reset log、diff | 待填 |
| Software update | 同版、升版、降版 policy 已驗證 | activation log、manifest | 待填 |
| Rollback | kernel panic / watchdog / userspace fail 可回退 | UART、fw_printenv | 待填 |
| Golden image | rescue path 可進入並重新刷寫 | UART、update log | 待填 |

## A04.4 Sensor / Fan / Power / Inventory

| 項目 | 檢查內容 | 證據 | 狀態 |
| --- | --- | --- | --- |
| Sensor discovery | hwmon / D-Bus / Redfish sensor 對齊 | sysfs、busctl、curl | 待填 |
| Threshold | threshold、event、deassert policy 符合需求 | journal、EventLog | 待填 |
| Fan tach | 插拔、低轉、高轉、失速狀態可辨識 | RPM、event、scope | 待填 |
| Fan PWM | PWM polarity、頻率、控制範圍正確 | scope、PID log | 待填 |
| Thermal policy | zone、PID、failsafe 行為已驗證 | stress log、fan log | 待填 |
| Power control | on/off/cycle、BMC reboot 後 host state 正確 | power daemon log | 待填 |
| Inventory | FRU、Entity Manager、Redfish 顯示一致 | EEPROM、D-Bus、Redfish | 待填 |

## A04.5 測試與量產前確認

| 項目 | 檢查內容 | 證據 | 狀態 |
| --- | --- | --- | --- |
| AC cycle | 多輪 AC cycle 無異常 | UART、journal、統計表 | 待填 |
| BMC reboot | BMC reboot 不影響不應受影響的 host domain | waveform、host log | 待填 |
| Update loop | 多輪 update / rollback / reboot 穩定 | updater log | 待填 |
| Stress | sensor、fan、network、Redfish 長測 | stress report | 待填 |
| Log rotation | rwfs 不被 log / dump 擠滿 | df、journal | 待填 |
| Security | secure boot、signature、帳號、TLS、WP policy | security checklist | 待填 |
| Factory flow | 燒錄、校準、序號、FRU provisioning 可追蹤 | factory log | 待填 |
