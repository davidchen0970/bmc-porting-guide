# 28. Logging / Event / Telemetry

本章整理 BMC 平台中的 Logging、Event、SEL、Redfish EventLog、Redfish EventService、Telemetry、`MetricReport`、audit / security log、crash dump、remote syslog 與現場 log package 設計。Logging / Event / Telemetry 是 bring-up、長測、量產、RMA 與現場維修時最重要的事後分析基礎。若 log 設計不完整，即使硬體或軟體有正確偵測到 fault，也可能在 Redfish、IPMI、journal、SEL、遠端監控平台看到不同結果，造成維修判讀困難。

本章先定義 log 與 event 的分層，再說明 OpenBMC `phosphor-logging`、systemd-journal、D-Bus logging entry、Redfish EventLog / LogService、IPMI SEL、Redfish EventService subscription、TelemetryService / `MetricReport`、remote syslog、容量與保存策略、事件去重與 rate limit、時間同步、log 收集與驗收方式。

## 適用範圍

本文件涵蓋 BMC 平台的 Logging、Event、SEL、Redfish EventLog、Redfish EventService、Telemetry、MetricReport、audit / security log、crash dump、remote syslog、容量與保存策略、現場 log package，以及相關 bring-up 與驗收工作。

## 適用讀者

適用於 BMC 韌體、BSP、driver、QA、field debug、field service、NOC 與系統整合相關人員。

## 快速導覽

- [分層與資料流](#161-基本分層與資料流)
- [SEL、Redfish EventLog 與 LogService](#164-selredfish-eventlog-與-logservice)
- [Redfish EventService 與事件推送](#165-redfish-eventservice-與事件推送)
- [Telemetry、Metric 與 MetricReport](#166-telemetrymetricmetricreport)
- [容量、時間與外部收集](#168-time-synctimestamp-與跨系統時間線)
- [安全稽核與故障證據](#1611-security--audit-log)
- [現場資料收集](#1614-target-端-log-收集套件)
- [常見問題與排查](#1615-常見問題與排查入口)
- [Bring-up 與驗收](#1616-bring-up-建議流程)

## 28.1 基本分層與資料流

Logging / Event / Telemetry 建議依下列資料流分層：

```mermaid
flowchart TB
    A["硬體 / 韌體 / service 狀態"] --> B["局部觀測資料"]
    B --> C["標準事件物件"]
    C --> D["外部通知 / 長期收集"]
    D --> E["現場分析 / RMA"]
```

- 硬體 / 韌體 / service 狀態：sensor threshold、PMBus fault、CPLD latch、watchdog、update error、security event。
- 局部觀測資料：kernel dmesg、systemd journal、driver sysfs、D-Bus properties、raw status register。
- 標準事件物件：OpenBMC logging entry、SEL record、Redfish EventLog Entry、audit event。
- 外部通知 / 長期收集：Redfish EventService、remote syslog、telemetry metric report、external monitoring。
- 現場分析 / RMA：debug package、timeline、callout、root cause record、維修動作。

| 類型 | 用途 | 粒度 | 保存週期 | 常見使用者 |
| --- | --- | --- | --- | --- |
| systemd journal | service log、structured metadata、debug trace | 高 | 依 storage policy | FW / QA / field debug |
| kernel log / dmesg | kernel driver、probe、panic、hardware error | 中至高 | 當次 boot 為主 | BSP / driver debug |
| OpenBMC event log | 標準化事件 object | 中 | 持久保存或輪替 | bmcweb、ipmid、field service |
| IPMI SEL | legacy event log | 中 | 容量有限 | ipmitool、host management |
| Redfish EventLog | RESTful LogService entries | 中 | 依產品政策 | Redfish client、WebUI |
| Redfish EventService | 事件主動推送 | event-level | 外部接收端保存 | NMS、monitoring system |
| Telemetry / `MetricReport` | 週期或條件式 metric 聚合 | metric-level | 本地或遠端 | 容量規劃、趨勢分析 |
| Audit / security log | 登入、權限、設定變更、更新、憑證 | 高價值事件 | 通常需較長保存 | 資安 / compliance |
| crash dump / core dump | 服務崩潰與 kernel panic 分析 | 高 | 受容量限制 | FW debug |

設計原則：

- Journal 是詳細原始資料；EventLog / SEL 是對外可讀事件；Telemetry 是可統計 metric。三者不應互相取代。
- 一個 hardware fault 可能同時產生 journal、D-Bus logging entry、Redfish EventLog 與 SEL，但欄位與時間戳需能互相對照。
- 不要把連續 sensor sample 全部寫成 event；只有 threshold transition、availability 變化、fault latch、policy change 才適合作為 event。
- Debug log 預設不應無限制寫入 rwfs，需有容量、輪替、遠端轉存與清除政策。

## 28.2 OpenBMC phosphor-logging 與 D-Bus logging entry

OpenBMC 常用 `phosphor-logging` 作為 event 與 journal logging 的基礎。它提供 structured logging API，將程式 log 寫入 systemd journal；同時由 `phosphor-log-manager` 管理 D-Bus event log objects。OpenBMC event log 常見路徑為：

```text
/xyz/openbmc_project/logging/entry/<id>
```

常見 interface：

| Interface | 用途 | 檢查重點 |
| --- | --- | --- |
| `xyz.openbmc_project.Logging.Entry` | 事件主要資料，例如 Message、Severity、`Timestamp`、`Resolved`、`AdditionalData` | Severity 是否正確，`AdditionalData` 是否足以排查 |
| `xyz.openbmc_project.Association.Definitions` | 事件與 inventory / callout 的關聯 | Redfish / 維修指引依賴此資料 |
| `xyz.openbmc_project.Object.Delete` | 刪除單筆事件 | 需受權限與 audit 控管 |
| `xyz.openbmc_project.Software.Version` | 事件發生時的軟體版本 | 現場比對版本與 RMA 很重要 |

檢查指令：

```bash
$ busctl tree xyz.openbmc_project.Logging
$ busctl introspect xyz.openbmc_project.Logging /xyz/openbmc_project/logging/entry/1
$ busctl get-property xyz.openbmc_project.Logging /xyz/openbmc_project/logging/entry/1 xyz.openbmc_project.Logging.Entry Message
$ journalctl -u xyz.openbmc_project.Logging.service -b --no-pager
```

事件建立建議欄位：

| 欄位 | 建議內容 | 原因 |
| --- | --- | --- |
| Message / `MessageId` | 可穩定對映 registry 的事件名稱 | 便於 Redfish / SEL / 翻譯 / 自動處理 |
| Severity | Informational / Warning / Critical | 外部監控與告警分級依賴此欄位 |
| `Timestamp` | UTC 或明確時區時間 | 需能與 host log、scope waveform 對齊 |
| `AdditionalData` | sensor path、threshold、raw value、register、bus、slot、version | 避免只看到「fault」但無法定位 |
| Callout / Inventory association | 疑似元件 inventory path | 維修與 Redfish Health 對映 |
| `Resolved` | 事件是否已修復或解除 | 避免舊 fault 長期影響 health |
| Software version | BMC image / service version | 比對已知問題與修復版本 |

## 28.3 Journal、kernel log 與 service log

Journal 與 kernel log 是最細的軟體觀測資料。Bring-up 與 field debug 時，建議每個事件同時保存 event log 與 journal window。

常用指令：

```bash
# 當次 boot 全部 journal
$ journalctl -b --no-pager

# 上一次 boot journal
$ journalctl -b -1 --no-pager

# 指定 service
$ journalctl -u xyz.openbmc_project.EntityManager.service -b --no-pager
$ journalctl -u xyz.openbmc_project.Logging.service -b --no-pager
$ journalctl -u xyz.openbmc_project.State.Host.service -b --no-pager

# kernel log
$ dmesg -T
$ journalctl -k -b --no-pager

# 依關鍵字過濾
$ journalctl -b --no-pager | grep -Ei 'error|fail|fault|critical|timeout|watchdog|pmbus|sensor|logging|event|sel'
```

保存策略：

- 開發版可拉高 journal verbosity；量產版需避免 debug log 過量寫入 flash。
- 若使用 persistent journal，需明確設定最大容量與輪替策略。
- 若只使用 volatile journal，故障重開後可能失去上一輪關鍵 log；建議對 watchdog / panic / update failure 類事件有額外保存策略。
- Service restart loop 需有 rate limit，避免 journal 被單一錯誤洗掉。

## 28.4 SEL、Redfish EventLog 與 LogService

IPMI SEL 與 Redfish EventLog 都是對外事件紀錄，但欄位與語意不同。SEL 容量通常較小，欄位也較固定；Redfish EventLog 可以承載 `MessageId`、MessageArgs、Severity、Created、`Resolved`、Links 等較完整資料。

| 項目 | IPMI SEL | Redfish EventLog | OpenBMC D-Bus event |
| --- | --- | --- | --- |
| 主要用途 | legacy management 工具 | RESTful 管理與自動化 | BMC 內部共同事件物件 |
| 事件識別 | sensor number / event type / offset | `MessageId` / Registry / EntryType | Message / `AdditionalData` |
| 容量 | 通常有限 | 依產品設計 | 依 phosphor-logging 與 storage policy |
| 清除方式 | IPMI clear SEL | `LogService.ClearLog` | D-Bus Delete / DeleteAll |
| 關聯資訊 | 有限 | Links / `OriginOfCondition` | Association / callout |

檢查指令：

```bash
# IPMI
$ ipmitool sel info
$ ipmitool sel list
$ ipmitool sel elist
$ ipmitool sel get <id>

# Redfish EventLog
$ curl -k -u root:0penBmc https://<bmc>/redfish/v1/Systems/system/LogServices/EventLog/Entries
$ curl -k -u root:0penBmc https://<bmc>/redfish/v1/Managers/bmc/LogServices
$ curl -k -u root:0penBmc https://<bmc>/redfish/v1/Chassis

# D-Bus logging
$ busctl tree xyz.openbmc_project.Logging
```

設計重點：

- 同一事件若同步轉成 SEL 與 Redfish EventLog，需避免重複計數或 severity 不一致。
- Clear SEL / ClearLog 是否也清 D-Bus logging entry，需要依產品政策明確定義。
- EventLog 滿時策略需記錄：循環覆寫、拒絕新增、刪最舊、遠端轉存後清除。
- Redfish EventLog 的 `MessageId` 應可對應 message registry，避免外部工具只能解析自由文字。

## 28.5 Redfish EventService 與事件推送

Redfish EventService 用於讓外部監控系統訂閱事件，BMC 在事件發生時以 HTTPS POST 傳送到 listener。這適合資料中心監控、事件集中化與即時告警，但需要網路、TLS、retry、queue、權限與訂閱生命週期管理。

事件推送流程如下：

```mermaid
flowchart TB
    subgraph G1["1. 建立與保存訂閱"]
        direction LR
        A1["Redfish client 建立 subscription"] --> A2["BMC 保存 subscription"]
    end
    subgraph G2["2. 產生與推送事件"]
        direction LR
        B1["事件產生"] --> B2["BMC 對 listener 送出 HTTPS POST"]
    end
    subgraph G3["3. 接收與重試"]
        direction LR
        C1["listener 回應 2xx"] --> C2["依 retry policy 重送 / queue / drop"]
    end
    G1 --> G2 --> G3
```

建立 subscription 時使用 `POST /redfish/v1/EventService/Subscriptions`，並依需求設定 `Destination = https://listener/event`、`EventFormatType = Event` 或 `MetricReport`，以及 `RegistryPrefixes`、`ResourceTypes`、`Context` 等篩選條件。事件來源可包含 sensor threshold、log entry、state change 與 security event。

檢查項目：

| 項目 | 檢查內容 | 風險 |
| --- | --- | --- |
| Destination | URL、DNS、路由、TLS 憑證 | BMC 建立訂閱成功但實際送不到 |
| `EventFormatType` | Event / `MetricReport` | Telemetry subscription 與 event subscription 混用 |
| Filter | RegistryPrefixes、ResourceTypes、OriginResources | 收到過多或收不到預期事件 |
| Retry | 重送次數、間隔、queue size | listener down 時塞滿 BMC storage / memory |
| Security | HTTPS、憑證、帳號權限、secret handling | 事件外洩或無法驗證 receiver |
| Audit | 建立 / 刪除 subscription 是否有記錄 | 無法追蹤誰修改告警路徑 |

測試指令：

```bash
# 查 EventService
$ curl -k -u root:0penBmc https://<bmc>/redfish/v1/EventService
$ curl -k -u root:0penBmc https://<bmc>/redfish/v1/EventService/Subscriptions

# 建立測試 subscription，body 需依平台支援欄位調整
$ curl -k -u root:0penBmc -H 'Content-Type: application/json' \
    -X POST https://<bmc>/redfish/v1/EventService/Subscriptions \
    -d '{"Destination":"https://<listener>/redfish-events","EventFormatType":"Event","Context":"test-subscription"}'
```

## 28.6 Telemetry、Metric、MetricReport

Telemetry 著重於「持續或週期性 metric 收集」，例如 CPU power、PSU input power、fan speed、temperature、network throughput、memory usage、boot time、service restart count。它和 event 的差異是：event 描述狀態轉換或異常；telemetry 描述時間序列或聚合資料。

Redfish TelemetryService 的常見能力包含：

- 查詢 metric metadata，例如 metric definition、metric property。
- 建立 `MetricReportDefinition`，定義一組 metric 如何聚合。
- 產生 `MetricReport`，本地保存或遠端推送。
- 搭配 EventService 用 `EventFormatType=MetricReport` 將 metric report 送往 listener。

Telemetry 設計範圍：

| Metric 類型 | 來源 | 用途 | 注意事項 |
| --- | --- | --- | --- |
| Temperature | D-Bus sensors / hwmon | 熱設計、長測趨勢 | sample rate 不宜過高 |
| Voltage / Current / Power | PMBus / ADC / D-Bus sensors | 功耗分析、PSU loading | 需標示 input/output 與 rail |
| Fan RPM / PWM | fan daemon / hwmon | thermal policy 驗證 | 需和 fan profile 對齊 |
| BMC resource | /proc、systemd、cgroup | memory leak、CPU loading | 避免 telemetry 本身造成負載 |
| Boot time | systemd-analyze、journal timestamp | 效能 baseline | 需分 AC boot / warm reboot / service ready |
| Event count | logging entry / SEL / journal | error rate、flapping 偵測 | 需做去重與時間窗 |

Telemetry 設計建議：

- 對外 metric name、unit、scale、sampling interval 需穩定。
- 長測資料應優先遠端收集；BMC 本地只保留短期或必要摘要。
- Telemetry 不應寫滿 rwfs；需限制報告數量、檔案大小與保留時間。
- Sensor unavailable 時需用明確狀態表示，不要使用 0 取代未知值。
- `MetricReport` 的時間戳需與 EventLog、journal 使用同一時間基準。

## 28.7 事件分類、Severity 與去重

事件分類需讓 FW、QA、field service、NOC 看到相同語意。建議建立平台事件分類表：

| 分類 | 例子 | Severity 建議 | 是否需要 SEL | 是否需要推送 |
| --- | --- | --- | --- | --- |
| Hardware fault | VR fault、PSU fault、fan fail、ECC threshold | Warning / Critical | 是 | 是 |
| Sensor threshold | 溫度 / 電壓 / 電流超界 | Warning / Critical | 依平台 | 是 |
| Availability | sensor unavailable、PMBus timeout | Warning | 依平台 | 視持續時間 |
| Inventory change | PSU / fan / drive 插拔 | Informational / Warning | 依平台 | 可選 |
| Firmware update | update start / success / fail / rollback | Info / Warning / Critical | 是 | 是 |
| Power state | power on/off/cycle、watchdog reset | Info / Warning | 依平台 | 可選 |
| Security | login fail、password change、cert change、secure boot fail | Warning / Critical | 依政策 | 是 |
| Debug / trace | service retry、temporary timeout | Debug / Info | 否 | 否 |

去重與 rate limit：

- Threshold assert / deassert 應成對記錄，避免每次 polling 都新增一筆。
- 同一 fault bit 若維持 asserted，只應在首次 asserted、狀態變化、超過時間窗時記錄。
- 通訊 timeout 類事件建議用連續失敗 N 次才上報，恢復時記錄 recover。
- Hot-plug 抖動需 debounce，避免插拔瞬間產生大量 SEL。
- Service restart loop 需限制 event 量，並保留第一筆與摘要。

## 28.8 Time sync、timestamp 與跨系統時間線

事件分析需要把 BMC journal、Redfish EventLog、SEL、Host log、scope waveform、CPLD latch time 對齊。若時間不準，會大幅增加排查成本。

時間設計重點：

- BMC boot 早期尚未 NTP sync 前的事件，需要標示未同步或保存 monotonic timestamp。
- SEL / EventLog / journal 建議都能對應到 UTC 或明確時區。
- Redfish `Created` / `EventTimestamp` 應符合資料中心監控工具期待格式。
- RTC / NTP / PTP / host time sync 的權威端需明確。
- AC loss 後若 RTC 沒電，時間會回到預設值；log 收集需同時保存 uptime / boot id。

指令：

```bash
$ timedatectl status
$ timedatectl timesync-status 2>/dev/null || true
$ journalctl --list-boots
$ cat /proc/uptime
$ cat /etc/os-release
```

## 28.9 Remote syslog 與外部收集

Remote syslog 可用於集中收集 BMC log，降低本地 flash 寫入與現場 log 遺失風險。設計時需確認 protocol、TLS、server reachable、queue、rate limit 與安全政策。

| 項目 | 建議記錄 | 注意事項 |
| --- | --- | --- |
| Protocol | UDP / TCP / TLS | UDP 可能遺失；TLS 需憑證管理 |
| Server | FQDN / IP / port | DNS、route、management VLAN |
| Filter | facility、severity、service | 避免 debug log 全量外送 |
| Queue | 網路斷線時如何暫存 | 不可無限制佔用 rwfs |
| Security | CA、client cert、auth | 避免 log 外洩或被偽造 server 接收 |
| Audit | remote syslog config change | 需記錄誰修改 server |

檢查指令依平台而定，常見方向：

```bash
$ systemctl status rsyslog --no-pager 2>/dev/null || true
$ systemctl status phosphor-rsyslog-conf --no-pager 2>/dev/null || true
$ journalctl -u rsyslog -b --no-pager 2>/dev/null || true
$ journalctl -u xyz.openbmc_project.Syslog.Config.service -b --no-pager 2>/dev/null || true
```

## 28.10 Log 容量、輪替與清除政策

Log 設計必須有容量上限與清除策略。若沒有上限，event storm 或 service spam 可能填滿 rwfs，進一步造成設定無法寫入、update 失敗或 service crash。

| 資料 | 常見路徑 / 來源 | 容量策略 | 清除策略 |
| --- | --- | --- | --- |
| systemd journal | `/var/log/journal` 或 volatile | `SystemMaxUse` / `RuntimeMaxUse` | journalctl --vacuum-size / time |
| phosphor logging entries | D-Bus / persistent store | max entries / max size | Delete / DeleteAll / ClearLog |
| SEL | IPMI SEL store | 固定筆數 | ipmitool sel clear |
| Redfish EventLog | LogService entries | 依 backend | `LogService.ClearLog` |
| core dump | `/var/lib/systemd/coredump` | 限制單檔與總量 | coredumpctl cleanup / tmpfiles |
| debug package | `/tmp` 或 `/var/tmp` | 上傳前暫存 | 重開機清除或明確刪除 |
| telemetry reports | Redfish / local file / remote | 保留最近 N 份 / 時間窗 | 輪替 / 遠端轉存 |

驗證項目：

- event storm 時不會填滿 rwfs。
- log 滿時策略符合產品需求：覆寫最舊、拒絕新增、告警、遠端轉存。
- 清除 LogService / SEL / D-Bus logging entry 需要適當權限，且清除動作本身應有 audit log。
- Factory reset 是否清 event log / audit log / telemetry data 需明確定義。
- RMA log package 不應依賴已被清除的 volatile log。

## 28.11 Security / Audit log

Security log 應涵蓋登入、登出、認證失敗、使用者 / 權限變更、密碼變更、憑證匯入、TLS 設定、SSH key、Redfish subscription、firmware update、secure boot、factory reset、remote syslog 設定變更等。

建議欄位：

| 欄位 | 內容 | 注意事項 |
| --- | --- | --- |
| Actor | 使用者、service account、host、local console | 避免記錄密碼或 token |
| Action | 登入、設定變更、更新、清 log、建立 subscription | 需使用穩定事件名稱 |
| Target | 被修改的 resource / object path / Redfish URI | 便於審計 |
| Result | success / failure / denied | 失敗原因需足以排查但不洩漏秘密 |
| Source | remote IP、session、interface | 需考慮隱私與法規 |
| `Timestamp` | UTC time / boot id | 需能與其他 log 對齊 |

## 28.12 Crash dump、core dump 與 watchdog 事件

Service crash、kernel panic、watchdog reset 需要保存最小可用證據：版本、上一輪 journal、core dump、reset reason、watchdog source、service status 與 event log。

指令：

```bash
$ coredumpctl list 2>/dev/null || true
$ coredumpctl info <PID> 2>/dev/null || true
$ systemctl --failed
$ journalctl -b -1 --no-pager > /tmp/journal-previous.txt 2>&1
$ journalctl -k -b -1 --no-pager > /tmp/kernel-previous.txt 2>&1
```

watchdog event 建議欄位：

- watchdog source：SoC、systemd、CPLD、external supervisor。
- reset target：BMC-only、host-only、full board。
- last heartbeat service / timestamp。
- reset reason register。
- previous boot journal 是否可讀。
- 是否伴隨 power fault / thermal fault / update flow。

## 28.13 Logging / Event / Telemetry 實作對照表

| 事件來源 | 觸發條件 | D-Bus logging | SEL | Redfish EventLog | EventService | Telemetry | 備註 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Sensor threshold | warning / critical assert / deassert | 是 | 依平台 | 是 | 是 | sensor metric | 避免 polling 重複新增 |
| PSU fault | PMBus STATUS fault / presence change | 是 | 是 | 是 | 是 | power metric | 先保存 fault snapshot |
| Fan fail | tach fail / fan missing | 是 | 是 | 是 | 是 | RPM / PWM | presence 與 tach fail 分開 |
| Firmware update | start / success / failure / rollback | 是 | 依平台 | 是 | 是 | 可選 | 需記錄版本與 image id |
| Security | login fail / cert change / user change | 是 | 依政策 | 是 | 是 | 否 | 需遮蔽敏感資料 |
| Watchdog reset | timeout / reboot | 是 | 是 | 是 | 是 | boot metric | 需關聯 reset reason |
| Inventory change | hot-plug insert / remove | 視需求 | 視需求 | 是 | 可選 | 否 | 需 debounce |
| Performance | boot time / CPU / memory / bandwidth | 否 | 否 | 否 | 可選 | 是 | 適合 Telemetry 而非 EventLog |

## 28.14 Target 端 log 收集套件

以下指令建立現場 log 收集套件：

```bash
$ $ ```bash
$ $ mkdir -p `/tmp`/logging-debug
$ $ cat /etc/os-release > `/tmp`/logging-debug/os-release.txt
$ $ uname -a > `/tmp`/logging-debug/uname.txt
$ $ cat /proc/cmdline > `/tmp`/logging-debug/proc-cmdline.txt
$ $ cat /proc/uptime > `/tmp`/logging-debug/proc-uptime.txt
$ $ timedatectl status > `/tmp`/logging-debug/timedatectl.txt 2>&1
$ $ journalctl --list-boots > `/tmp`/logging-debug/journal-boots.txt 2>&1

# journal / kernel
$ $ journalctl -b --no-pager > `/tmp`/logging-debug/journal-current.txt
$ $ journalctl -b -1 --no-pager > `/tmp`/logging-debug/journal-previous.txt 2>&1
$ $ journalctl -k -b --no-pager > `/tmp`/logging-debug/journal-kernel-current.txt
$ $ journalctl -k -b -1 --no-pager > `/tmp`/logging-debug/journal-kernel-previous.txt 2>&1
$ $ dmesg -T > `/tmp`/logging-debug/dmesg.txt
$ $ systemctl --failed > `/tmp`/logging-debug/systemctl-failed.txt 2>&1

# logging service
$ $ systemctl status xyz.openbmc_project.Logging.service --no-pager > `/tmp`/logging-debug/logging-status.txt 2>&1
$ $ journalctl -u xyz.openbmc_project.Logging.service -b --no-pager > `/tmp`/logging-debug/logging-journal.txt 2>&1
$ $ busctl tree xyz.openbmc_project.Logging > `/tmp`/logging-debug/dbus-logging-tree.txt 2>&1
$ $ busctl tree xyz.openbmc_project.ObjectMapper > `/tmp`/logging-debug/dbus-objectmapper.txt 2>&1

# Redfish / IPMI, if available
$ $ ipmitool sel info > `/tmp`/logging-debug/ipmi-sel-info.txt 2>&1 || true
$ $ ipmitool sel elist > `/tmp`/logging-debug/ipmi-sel-elist.txt 2>&1 || true
$ $ ipmitool sdr elist > `/tmp`/logging-debug/ipmi-sdr-elist.txt 2>&1 || true

# crash / core dump
$ $ coredumpctl list > `/tmp`/logging-debug/coredump-list.txt 2>&1 || true

# storage usage
$ $ df -h > `/tmp`/logging-debug/df-h.txt
$ $ df -i > `/tmp`/logging-debug/df-i.txt
$ $ journalctl --disk-usage > `/tmp`/logging-debug/journal-disk-usage.txt 2>&1

$ $ tar czf `/tmp`/logging-debug-$(date +%Y%m%d-%H%M%S).tar.gz -C `/tmp` logging-debug
$ $ ```
```

### Redfish dump 範本

```bash
$ $ ```bash
$ $ mkdir -p `/tmp`/logging-debug/redfish
$ $ curl -k -u root:0penBmc https://<bmc>/redfish/v1/EventService > `/tmp`/logging-debug/redfish/EventService.json 2>&1
$ $ curl -k -u root:0penBmc https://<bmc>/redfish/v1/EventService/Subscriptions > `/tmp`/logging-debug/redfish/EventService-Subscriptions.json 2>&1
$ $ curl -k -u root:0penBmc https://<bmc>/redfish/v1/Systems/system/LogServices > `/tmp`/logging-debug/redfish/System-LogServices.json 2>&1
$ $ curl -k -u root:0penBmc https://<bmc>/redfish/v1/Systems/system/LogServices/EventLog/Entries > `/tmp`/logging-debug/redfish/EventLog-Entries.json 2>&1
$ $ curl -k -u root:0penBmc https://<bmc>/redfish/v1/TelemetryService > `/tmp`/logging-debug/redfish/TelemetryService.json 2>&1
$ $ ```
```

## 28.15 常見問題與排查入口

| 現象 | 建議排查方向 | 第一輪檢查 |
| --- | --- | --- |
| D-Bus logging entry 沒產生 | 事件沒有 commit、logging service fail、error YAML / registry 不匹配 | journal、busctl tree Logging、service status |
| journal 有錯但 EventLog 沒有 | 只是 debug log，未建立標準 event | 檢查 service event path 與 phosphor-logging usage |
| SEL 有事件但 Redfish 沒有 | 轉換橋接或 LogService mapping 缺失 | ipmitool、bmcweb journal、D-Bus logging entry |
| Redfish EventLog 有事件但 SEL 沒有 | 平台政策不轉 SEL 或 sensor mapping 缺失 | event policy、ipmid journal、SDR mapping |
| 事件重複大量產生 | threshold polling 重複、fault 未 debounce、service restart loop | timestamp、`AdditionalData`、journal window |
| EventService subscription 建立但收不到 | listener unreachable、TLS、DNS、filter、queue fail | bmcweb journal、network、listener log |
| Telemetry 沒資料 | TelemetryService 未啟用、`MetricReportDefinition` 缺、sensor association 缺 | Redfish Telemetry URI、D-Bus sensors、bmcweb journal |
| Log 滿導致 service 異常 | rwfs 滿、journal 無上限、event storm | df -h、journalctl --disk-usage、event count |
| 時間戳錯亂 | NTP 未 sync、RTC 無效、timezone / UTC 混用 | timedatectl、journal boots、EventLog Created |
| 清 log 後無 audit | ClearLog / DeleteAll 未產生安全事件 | bmcweb / logging / audit policy |
| Crash 後沒有 core | coredump disabled、容量不足、tmpfiles 清掉 | coredumpctl、systemd-coredump config、df |

## 28.16 Bring-up 建議流程

1. 建立事件分類表：hardware fault、sensor threshold、availability、inventory change、update、power state、security、debug。
2. 對每個事件定義 `MessageId`、Severity、`AdditionalData`、callout、是否進 SEL、是否進 Redfish EventLog、是否推送 EventService。
3. 確認 `phosphor-logging` service、D-Bus logging entry、journal structured metadata 正常。
4. 確認 sensor threshold assert / deassert、PMBus fault、CPLD fault、watchdog reset、update failure 都能留下一致事件。
5. 驗證 Redfish EventLog、IPMI SEL、D-Bus logging entry 的時間、severity、來源、元件關聯一致。
6. 建立 EventService subscription 測試 listener，驗證事件 POST、retry、queue、filter、TLS。
7. 若平台支援 TelemetryService，建立至少一組 `MetricReportDefinition` 並驗證報告內容、單位、時間戳、遠端推送。
8. 設定 log 容量上限、輪替、清除與 factory reset 政策。
9. 做 event storm、service restart loop、listener offline、rwfs nearly full、NTP not synced、AC cycle、BMC reboot 測試。
10. 保存 logging-debug package、Redfish dump、IPMI dump、journal、scope / host log 對齊結果。

## 28.17 當前平台 Logging / Event / Telemetry 實測表

| 項目 | 指令 / 來源 | 實測值 | 備註 |
| --- | --- | --- | --- |
| phosphor-logging service | systemctl status / journalctl | [待填] | service 是否 active |
| D-Bus logging entries | busctl tree xyz.openbmc_project.Logging | [待填] | entry count / latest event |
| Journal storage mode | journalctl --disk-usage / journald.conf | [待填] | persistent / volatile |
| SEL status | ipmitool sel info | [待填] | 容量與使用率 |
| Redfish EventLog | curl LogServices/EventLog | [待填] | Entry schema 與 ClearLog |
| EventService | curl EventService | [待填] | subscription 支援欄位 |
| TelemetryService | curl TelemetryService | [待填] | `MetricReportDefinition` |
| Remote syslog | rsyslog / phosphor-rsyslog-conf | [待填] | protocol / server / TLS |
| Security audit | 登入 / 設定變更測試 | [待填] | 是否有事件 |
| Sensor threshold event | fault injection | [待填] | assert / deassert |
| PSU / fan fault event | fault injection | [待填] | callout / `AdditionalData` |
| Update event | firmware update 測試 | [待填] | start / success / fail / rollback |
| Watchdog reset event | watchdog 測試 | [待填] | reset reason 對齊 |
| Log full policy | 容量壓力測試 | [待填] | 輪替 / 拒絕 / 清除 |

## 28.18 回查結果

本章已回查前後文並補齊下列銜接點：

- 第 2 章 Flash / Storage 已說明 rwfs 與 persistent data，本章補上 journal、event log、core dump、telemetry report 的容量與清除策略。
- 第 22 章 I2C / PMBus 已說明 PMBus fault，本章補上 fault snapshot 後如何形成 D-Bus logging、SEL、Redfish EventLog 與 EventService 推送。
- 第 12至14 章 Sensor / Fan / Power 會產生 threshold、fan fail、power fault 事件，本章補上事件分類、severity、去重與外部呈現方式。
- 第 27 章 Inventory / FRU / Asset 已定義 callout 與 association，本章補上 logging entry 與 inventory association、Redfish `OriginOfCondition` 的關係。
- 第 29 章 Presence / Intrusion / GPIO State Sensor 可引用本章的 event debounce、security / audit log 與 intrusion event policy。
- 第 36 章 Security Baseline 可引用本章的 audit log、remote syslog、ClearLog 權限與敏感資料遮蔽策略。
- 第 27至28 章 Debug Methodology / Toolkit 可引用本章 logging-debug package 作為現場收集基礎。

## 28.19 驗收 Checklist

### 驗收範圍

- [ ] 已建立平台事件分類、severity、`MessageId`、`AdditionalData` 與 callout 規範。
- [ ] `phosphor-logging`、systemd journal、D-Bus logging entry 正常運作。
- [ ] Redfish EventLog、IPMI SEL、D-Bus logging entry 的同一事件可互相對照。
- [ ] Sensor threshold assert / deassert 不會重複洗 log，且恢復事件可正確產生。
- [ ] PSU / VR / CPLD fault 會先保存 raw status snapshot，再依政策清除 fault。
- [ ] EventService subscription 可建立、刪除、推送、retry，listener offline 行為已驗證。
- [ ] TelemetryService / `MetricReport` 若平台支援，metric name、unit、sampling interval、timestamp 已驗證。
- [ ] journal、EventLog、SEL、core dump、telemetry report 的容量上限與輪替策略已設定。
- [ ] ClearLog / SEL clear / DeleteAll 有權限控管，且清除動作本身可被 audit。
- [ ] remote syslog 若平台支援，server、protocol、TLS、queue、filter 已驗證。
- [ ] NTP / RTC / timestamp / boot id 可讓 BMC log、host log、scope waveform 對齊。
- [ ] event storm、service restart loop、rwfs nearly full、AC cycle、BMC reboot、watchdog reset 測試已完成。
- [ ] logging-debug package、Redfish dump、IPMI dump、journal、core dump、版本資訊已保存。

## 28.20 本章參考資料

- OpenBMC phosphor-logging README: [https://github.com/openbmc/phosphor-logging/blob/master/README.md](https://github.com/openbmc/phosphor-logging/blob/master/README.md)
- OpenBMC phosphor-logging core overview: [https://deepwiki.com/openbmc/phosphor-logging/2-core-logging-system](https://deepwiki.com/openbmc/phosphor-logging/2-core-logging-system)
- Redfish Event Service overview: [https://servermanagementportal.ext.hpe.com/docs/concepts/redfishevents](https://servermanagementportal.ext.hpe.com/docs/concepts/redfishevents)
- Redfish Telemetry Service overview: [https://redfish.redoc.ly/docs/concepts/redfishtelemetry/](https://redfish.redoc.ly/docs/concepts/redfishtelemetry/)
- DMTF Redfish Telemetry White Paper DSP2051: [https://www.dmtf.org/sites/default/files/standards/documents/DSP2051_1.0.0.pdf](https://www.dmtf.org/sites/default/files/standards/documents/DSP2051_1.0.0.pdf)
- Redfish Schema Index: [https://redfish.dmtf.org/schemas/v1/](https://redfish.dmtf.org/schemas/v1/)
