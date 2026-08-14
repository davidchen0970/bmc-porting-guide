# 25. Fan Control 與 Thermal Policy

## 適用範圍

本文件說明 OpenBMC 平台的 Fan Control 與 Thermal Policy, 涵蓋風扇控制資料路徑, 狀態模型, thermal zone, PID, stepwise, event-driven policy, failsafe, thermal profile, 設定部署, bring-up, 驗證與 tuning.

## 適用讀者

- 負責 BMC 韌體, OpenBMC service, thermal policy, fan controller, sensor 整合或平台 bring-up 的開發與整合人員.
- 執行風扇線路驗證, thermal tuning, 異常測試, Redfish/IPMI 整合或量產驗收的人員.

## 快速導覽

- [理解控制資料路徑](#132-fan-control-資料路徑): Hardware sensor, kernel interface, D-Bus, policy, output 與 feedback.
- [定義狀態與 Thermal Zone](#133-fan-state-model): Host Off, Boot, Host On, Failsafe 與 zone mapping.
- [選擇控制策略](#135-openbmc-fan-control-架構選項): PID, stepwise, event-driven 與平台自訂 daemon.
- [設計 Failsafe 與 Profile](#139-failsafe-設計): Trigger, output, recovery 與 thermal mode.
- [執行 Bring-up 與驗證](#1312-bring-up-步驟): Kernel, D-Bus, 固定 PWM, policy, Redfish/IPMI 與 event.
- [進行 Tuning 與驗收](#1316-thermal-tuning-與量測資料): 測試矩陣, 常見問題, Checklist 與資料表.

本章承接第 24 章的 sensor porting: 第 24 章回答「感測器如何被讀到, 如何發佈到 D-Bus」, 本章回答「系統如何依感測器資料調整 fan PWM / fan target, 並在異常時進入安全風速」. 對伺服器 BMC 來說, Fan Control 由 host power state, thermal zone, sensor availability, fan tach feedback, presence, failsafe policy, Redfish / IPMI 顯示與事件紀錄共同組成控制迴路.

本章目標:

- 建立 Fan Control / Thermal Policy 的共同資料路徑.
- 定義 Host Off, Boot, Host On, Failsafe 四種基本狀態.
- 說明 thermal zone, sensor group, fan group 與 policy 的關係.
- 說明 PID, stepwise, event-driven fan control 的適用情境.
- 建立 bring-up, debug, tuning 與驗收 Checklist.

## 25.1 適用情境與控制目標

Fan Control 適用於下列場景:

- BMC 需依 CPU / DIMM / VR / GPU / NVMe / PSU / ambient 溫度自動調整 fan speed.
- Host off 時仍需維持 standby cooling.
- Host boot 前後 thermal telemetry 尚未穩定, 需要 boot airflow policy.
- Sensor missing, tach lost, fan controller fail 時需進入 failsafe.
- 系統需要在散熱, 噪音, 風扇壽命, 功耗之間取得平衡.
- 量產前需建立 thermal tuning baseline 與 acoustic baseline.

Fan Control 的設計目標:

| 目標 | 說明 | 驗證方式 |
| :--- | :--- | :--- |
| 熱安全 | 所有關鍵元件溫度不超過規格 | stress test, thermal chamber, sensor log |
| 可預期 | 相同 sensor 條件下 fan target 一致 | log replay, 固定負載測試 |
| 平順 | fan speed 不頻繁上下跳動 | fan PWM / RPM trend |
| 異常安全 | sensor 或 fan 異常時進入安全風速 | 拔 fan, mask sensor, 停 service 測試 |
| 可除錯 | 可追蹤 input sensor, zone, controller, output | journal, D-Bus, policy log |
| 對外一致 | Redfish / IPMI / SEL 顯示與內部狀態一致 | API / SDR / SEL 驗證 |

## 25.2 Fan Control 資料路徑

常見 Fan Control 資料流:

```mermaid
flowchart TB
    A["Hardware Sensors"] --> B["Kernel / hwmon / IIO / PMBus"]
    B --> C["Sensor Daemons"]
    C --> D["D-Bus"]
    D --> E["Fan Control Policy"]
    E --> F["Output"]
    F --> G["Hardware Output"]
    G --> H["Feedback"]
```

Fan Control 常見有兩層閉迴路:

```text
Temperature loop:
    temperature sensor / margin sensor → thermal controller → target RPM or thermal output

Fan loop:
    fan tach RPM → fan controller → PWM output
```

若平台較簡單, 也可以只有一層:

```text
Temperature sensor → stepwise table → PWM output
```

## 25.3 Fan State Model

建議每個平台至少定義下列四種 fan state. 這些 state 不一定要以同一個 daemon 的 enum 呈現, 但在設計文件, test plan 與 debug log 中需能對齊.

| State | 觸發條件 | 建議行為 | 注意事項 |
| :--- | :--- | :--- | :--- |
| Host Off | Host power off, BMC standby | 使用最低安全 PWM 或 standby thermal policy | 仍需考慮 PSU, BMC SoC, NVMe backplane, retimer 等 standby 發熱源. |
| Boot | Host power on transition, BIOS / OS telemetry 尚未穩定 | 給 boot floor, 例如 50%~80% PWM 或固定 RPM | 避免 CPU / DIMM 尚未上報 telemetry 時 airflow 不足. |
| Host On | Host running, 主要 sensor 可讀 | 依 PID / stepwise / event policy 控制 | 正常閉迴路, 需平衡散熱與噪音. |
| Failsafe | sensor unavailable, tach lost, control write fail, host state unknown | 拉到 failsafe PWM / RPM, 產生 event | Failsafe 值需足夠保護硬體, 不建議為了噪音設太低. |

狀態切換建議:

```text
BMC boot
    ↓
Offline / Init：先採安全預設 PWM
    ↓
Host Off：若 host off 且 standby sensors 正常
    ↓
Boot：host power transition 或 policy 要求預先提升 airflow
    ↓
Host On：sensor set 穩定且 control daemon ready
    ↓
Failsafe：任一必要條件失效
    ↓
Recover：必要 sensor / tach / control output 恢復後，依 hysteresis 與 hold time 回到前一狀態
```

狀態切換不宜只依單一 sensor value 判斷, 建議同時納入:

- Host state / chassis state.
- 必要 sensor availability.
- Fan presence / tach validity.
- Control daemon 是否已載入 policy.
- 最近一次 PWM write 是否成功.
- 是否仍在 config reload / service shutdown 階段.

## 25.4 Thermal Zone 設計

Thermal zone 是 fan policy 的核心. Zone 代表一組需要共同控制 airflow 的硬體區域, 包含 input sensors, controlled fans, policy 與安全邊界.

常見 zone:

| Zone | Input sensors | Controlled fans | 常見目標 |
| :--- | :--- | :--- | :--- |
| CPU zone | CPU DTS / PECI / PLDM / margin | front fan bank / CPU fan | 保持 CPU margin 或 CPU temp 在 target 以下. |
| DIMM zone | DIMM temp / memory controller temp | front fan bank | 防止 DIMM over temperature. |
| VR zone | VR PMBus temp / power / current | front fan bank / local fan | 高負載時提升 airflow. |
| PCIe / GPU zone | GPU temp / PCIe ambient / riser temp | GPU fan bank / chassis fan | 支援 high power add-in card. |
| NVMe zone | NVMe composite temp / backplane temp | front fan / mid fan | 避免 drive throttle. |
| PSU zone | PSU temp / PSU fan / inlet temp | PSU internal fan 或 chassis fan | 多數 PSU fan 由 PSU 自己控制, BMC 主要監控. |
| Chassis ambient zone | inlet / outlet / board ambient | all chassis fans | 維持整機 airflow baseline. |

Zone 設計欄位範本:

```text
Zone name: [待填]
Zone ID: [待填]
Controlled fans: [fan0, fan1, ...]
Input sensors: [CPU0_Temp, DIMM_A0_Temp, ...]
Required sensors: [待填]
Optional sensors: [待填]
Host state dependency: HostOff / Boot / HostOn / Always
Control algorithm: PID / Stepwise / Event-driven / Fixed
Min PWM / RPM: [待填]
Max PWM / RPM: [待填]
Failsafe PWM / RPM: [待填]
Slew rate up/down: [待填]
Hysteresis / hold time: [待填]
Recovery rule: [待填]
Owner: Thermal / BMC / HW
```

Zone 設計原則:

- 每個 fan 最好只有一個主要控制者, 避免不同 daemon 同時寫同一個 PWM.
- 多個 zone 共享同一組 fan 時, 通常取最大 fan request, 避免某 zone 降速影響另一 zone.
- Required sensor 缺失應觸發 failsafe; optional sensor 缺失可標記 degraded 並使用 fallback.
- 若 sensor 來自 host telemetry, 需定義 host off / host not ready 的 fallback 值.
- 若 fan bank 有冗餘設計, 需定義單顆 fan fail 時的補償策略.

## 25.5 OpenBMC Fan Control 架構選項

OpenBMC 常見 fan / thermal 相關元件如下:

| 元件 | 主要責任 | 常見輸入 | 常見輸出 |
| :--- | :--- | :--- | :--- |
| `dbus-sensors` FanSensor | 讀 fan tach, 發佈 RPM sensor | hwmon `fan*_input` | `/xyz/openbmc_project/sensors/fan_tach/...` |
| `dbus-sensors` PwmSensor / control path | 暴露或寫入 PWM control | hwmon `pwm*` / config | D-Bus control 或 sysfs write |
| `phosphor-pid-control` | PID / stepwise thermal control | sensors, zones, D-Bus / JSON config | PWM / target RPM / zone output |
| `phosphor-fan-control` | event-driven fan control | events, groups, zones, D-Bus state | fan target / zone property / PWM request |
| `phosphor-fan-monitor` | fan health monitor | tach, presence, threshold | Functional, event, 可選 power-off action |
| `phosphor-fan-presence` | fan presence detection | GPIO, tach, I2C | Inventory `Present` |
| `phosphor-state-manager` | host / chassis power state | host state | policy input |
| `bmcweb` | Redfish 顯示 | D-Bus sensor / inventory | Redfish thermal / fan resources |
| `phosphor-host-ipmid` | IPMI sensor / SEL | D-Bus sensor / event | SDR / SEL / OEM command |

常見選型:

```text
方案 A：phosphor-pid-control
    適合需要 PID / stepwise、zone-based thermal control、tuning log 的平台。

方案 B：phosphor-fan-control
    適合以事件、D-Bus state、timer、group condition 驅動 fan target 的平台。

方案 C：平台自訂 daemon
    適合有客製硬體控制器、特殊 redundancy、host co-processor 共同控制的情境。

方案 D：固定 PWM / fixed table
    適合 bring-up 初期或小型平台，但量產前仍需完成 abnormal path 驗證。
```

## 25.6 PID Control 原理與參數

PID control 的目標是依誤差調整輸出, 使溫度或 margin 接近目標值. Fan control 中常見有兩種 PID:

```text
Thermal PID:
    input  = temperature 或 margin
    output = target RPM 或 thermal output

Fan PID:
    input  = actual RPM
    output = PWM
```

常見參數:

| 參數 | 說明 | 調整方向 |
| :--- | :--- | :--- |
| `Kp` | 比例項, 對目前誤差反應 | 太大容易震盪; 太小反應慢. |
| `Ki` | 積分項, 補長期偏差 | 太大容易 windup; 太小可能長期偏離 target. |
| `Kd` | 微分項, 對變化率反應 | 太大容易放大 sensor noise. |
| sample time | 控制迴路週期 | 太短會增加 D-Bus / I2C loading; 太長反應慢. |
| setpoint | 目標溫度, margin 或 RPM | 需依 component spec 與 thermal margin 設定. |
| min output | 最低 fan request | 避免風量低於安全值或 fan stall. |
| max output | 最高 fan request | 通常為 100% PWM 或 max RPM. |
| slew rate | 每次允許變動幅度 | 降低噪音突變與 fan hunting. |
| anti-windup | 限制積分累積 | 避免長時間高溫後降溫太慢. |

Tuning 建議流程:

```text
1. 固定 PWM，建立溫度 / RPM / noise baseline。
2. 找出每個 zone 的最低安全 airflow。
3. 先只啟用 P，確認反應方向正確。
4. 加入小 Ki，修正長期偏差。
5. 視需要加入 Kd 或使用濾波降低 noise。
6. 設定 output min / max / slew rate。
7. 驗證負載 step up / step down。
8. 驗證 thermal chamber high ambient。
9. 驗證 fan fail / sensor fail / host telemetry missing。
10. 保存 tuning log 與版本。
```

PID 常見現象:

| 現象 | 建議排查方向 | 調整方向 |
| :--- | :--- | :--- |
| Fan speed 上下震盪 | `Kp` 太大, sample time 太短, sensor noise 大 | 降低 `Kp`, 增加 hysteresis / filtering, 放慢更新. |
| 溫度長期高於 target | `Kp` / `Ki` 太小, min airflow 不足, zone mapping 錯 | 提高 `Kp` / `Ki`, 檢查 fan bank 與 sensor mapping. |
| 降溫後 fan 很久不降 | `Ki` windup, down slew rate 太低 | 加 anti-windup, 調整 slew rate. |
| 負載上升時反應慢 | sample time 太長, slew rate up 太小 | 縮短 sample time, 提高 slew rate up. |
| 低溫仍很吵 | min output 太高, failsafe 未解除, host state 錯 | 查 zone state, failsafe flag, host state. |

## 25.7 Stepwise / Table-based Fan Policy

Stepwise policy 使用溫度區間對應固定 fan output, 適合早期 bring-up, 小型平台, 或不需要連續 PID 的 zone.

範例:

```text
CPU_Temp < 35°C       fan target = 25%
35°C ~ 45°C           fan target = 35%
45°C ~ 55°C           fan target = 50%
55°C ~ 65°C           fan target = 70%
65°C ~ 75°C           fan target = 90%
> 75°C                fan target = 100%
Sensor unavailable    fan target = failsafe
```

Stepwise 設計重點:

- 每個 step 需有 hysteresis, 避免溫度在邊界附近時 fan speed 來回跳.
- 升速與降速可使用不同 threshold 或 hold time.
- 高溫區間 step 可以比較密, 低溫區間 step 可以比較寬.
- 若多個 sensor 對同一 fan group 提出需求, 通常取最大輸出.
- Stepwise table 需搭配 component spec, thermal chamber 與 acoustic tuning 資料.

## 25.8 Event-driven Fan Control

Event-driven policy 適合處理非連續型條件, 例如 host state, cable presence, fan tray presence, system type, service state, timer loop 或特定 D-Bus property 變化.

常見 trigger:

```text
- Host state 變為 Running
- Host state 變為 Off
- fan tray 插入 / 拔除
- sensor threshold alarm
- timer 每 2 秒重新計算 target
- thermal mode 切換，例如 Acoustic / Balanced / Performance
- system type / SKU / cooling type 不同
- redundancy lost
```

常見 action:

```text
- 設定 zone floor
- 設定 target RPM
- 設定 PWM percent
- 提高 failsafe percent
- 記錄 event
- 更改 thermal profile
- 將 zone 標記為 degraded
```

Event-driven policy 和 PID / stepwise 可以並存. 常見方式是 Event policy 設定 floor / ceiling / profile, PID 或 stepwise 依 sensor 計算 target, 最後取最大值或依優先序合併.

## 25.9 Failsafe 設計

Failsafe 是 Fan Control 中最重要的安全路徑之一. Failsafe 不只代表 fan 全速, 也代表「目前控制環境不完整, 無法保證正常閉迴路判斷」.

常見 failsafe trigger:

```text
[ ] Required temperature sensor unavailable
[ ] Required margin sensor unavailable
[ ] Sensor value 為 NaN / 非有限值 / 超出合理範圍
[ ] Sensor timeout
[ ] Fan tach = 0
[ ] Fan tach 低於 lower critical threshold
[ ] Fan presence missing
[ ] PWM write failed
[ ] Fan controller I2C timeout
[ ] D-Bus object 消失
[ ] Entity Manager config reload 中
[ ] fan control daemon 啟動中或即將停止
[ ] Host state unknown
[ ] Thermal zone config missing
[ ] Redundancy lost
```

Failsafe output 設計:

| 類型 | 說明 | 使用時機 |
| :--- | :--- | :--- |
| fixed failsafe PWM | 固定 100% 或指定百分比 | 最保守, 適合早期 bring-up. |
| zone failsafe percent | 每個 zone 有自己的 failsafe percent | 多 zone 平台, 可降低不必要噪音. |
| fan-specific failsafe | 特定 fan bank / fan tray 有不同 failsafe | airflow path 差異大的系統. |
| offline failsafe | daemon reload / shutdown / offline 時採用 failsafe | 防止服務暫停時風速掉太低. |
| strict failsafe | failsafe 時強制使用 failsafe percent | 適合要求明確安全行為的平台. |

Failsafe recovery 建議:

```text
1. 確認 required sensors 恢復且連續 N 次有效。
2. 確認 tach feedback 恢復且高於最低門檻。
3. 確認 PWM write 成功。
4. 等待 hold time，避免立即降速。
5. 逐步降低 fan speed，不要直接跳回低速。
6. 記錄 recovery event，便於追蹤。
```

## 25.10 Thermal Mode / Fan Profile

許多平台會提供多種 thermal profile:

| Profile | 目標 | 常見行為 |
| :--- | :--- | :--- |
| Acoustic | 低噪音 | 較低 floor, 較慢升速, 較高溫度目標. |
| Balanced | 預設平衡 | 散熱與噪音折衷. |
| Performance | 散熱優先 | 較高 floor, 較快升速, 較低溫度目標. |
| Max Cooling | 最大散熱 | 固定高 PWM 或 100%. |
| Service / Manufacturing | 工廠或維修 | 固定值或測試用 profile. |

Profile 切換需定義:

```text
- 是否允許 Redfish / IPMI / OEM command 修改
- 是否跨 reboot 保存
- 是否受 host policy 限制
- 是否需要權限控管
- 切換時是否立即生效
- 切換失敗時 fallback profile
```

## 25.11 設定檔與部署路徑

不同 OpenBMC 實作使用不同設定來源, 需先確認平台採用哪套 fan control 架構.

`phosphor-pid-control` 常見設定來源:

```text
- D-Bus configuration：由 Entity Manager 提供 Pid / Pid.Zone / Stepwise 類型設定
- JSON configuration：預設 /usr/share/swampd/config.json，可用 --conf 指定其他路徑
```

JSON 概念範本:

```json
{
    "sensors": [
        {
            "name": "fan0",
            "type": "fan",
            "readPath": "/xyz/openbmc_project/sensors/fan_tach/Fan0",
            "writePath": "/sys/class/hwmon/hwmonX/pwm1",
            "min": 0,
            "max": 255,
            "timeout": 4,
            "ignoreDbusMinMax": true,
            "unavailableAsFailed": true
        },
        {
            "name": "CPU0_Temp",
            "type": "temp",
            "readPath": "/xyz/openbmc_project/sensors/temperature/CPU0_Temp",
            "timeout": 4,
            "unavailableAsFailed": true
        }
    ],
    "zones": [
        {
            "id": 0,
            "minThermalOutput": 3000.0,
            "failsafePercent": 100.0,
            "pids": ["CPU0_Thermal", "Fan0_Controller"]
        }
    ]
}
```

`phosphor-fan-control` 常見設定檔:

```text
/usr/share/phosphor-fan-presence/control/manager.json
/usr/share/phosphor-fan-presence/control/profiles.json
/usr/share/phosphor-fan-presence/control/fans.json
/usr/share/phosphor-fan-presence/control/zones.json
/usr/share/phosphor-fan-presence/control/groups.json
/usr/share/phosphor-fan-presence/control/events.json
```

平台 layer 建議:

```text
meta-<platform>/recipes-phosphor/fans/phosphor-fan-presence/...
meta-<platform>/recipes-phosphor/fans/phosphor-pid-control/...
meta-<platform>/recipes-phosphor/configuration/entity-manager/...
```

部署後需確認:

```bash
# 確認設定檔存在
ls -l /usr/share/swampd/config.json 2>/dev/null
ls -l /usr/share/phosphor-fan-presence/control/ 2>/dev/null

# 確認 package / service 是否進 image
systemctl list-units | grep -Ei 'fan|pid|thermal|swampd'
```

## 25.12 Bring-up 步驟

建議從「固定風速」逐步導入完整策略, 不要一開始就直接啟用複雜 PID.

### Step 1: 確認硬體與線路
- [ ] fan power rail 正常
- [ ] PWM pin / tach pin / presence pin 接線正確
- [ ] PPR、PWM polarity、fan voltage、min RPM、max RPM 確認

### Step 2: 確認 kernel / sysfs
- [ ] `/sys/class/hwmon/hwmonX/fan*_input` 存在
- [ ] `/sys/class/hwmon/hwmonX/pwm*` 存在且可寫
- [ ] tach RPM 與外部量測一致

### Step 3: 確認 D-Bus sensor
- [ ] `fan_tach` sensor 出現在 D-Bus
- [ ] temperature sensors 出現在 D-Bus
- [ ] Availability / Functional 正確

### Step 4: 手動固定 PWM 測試
- [ ] 寫入 30%、50%、70%、100% PWM
- [ ] 確認 RPM 單調上升
- [ ] 確認最低 PWM 不會造成 fan stall

### Step 5: 建 inventory / presence / monitor
- [ ] Present 反映 fan tray 插拔
- [ ] tach lost 會標記 `Functional=false`
- [ ] event / SEL 可追蹤

### Step 6: 建立 thermal zone
- [ ] sensor group 與 fan group 對齊 airflow path
- [ ] required / optional sensor 分清楚
- [ ] min / max / failsafe 設定完成

### Step 7: 先導入 stepwise policy
- [ ] 升溫時 fan target 上升
- [ ] 降溫時 fan target 平順下降
- [ ] threshold / hysteresis 合理

### Step 8: 導入 PID policy
- [ ] Kp / Ki / Kd 初始值保守
- [ ] tuning log 開啟
- [ ] step load / unload 測試完成

### Step 9: 驗證 failsafe
- [ ] mask temperature sensor
- [ ] 拔 fan 或讓 fan tach = 0
- [ ] 停 fan control service
- [ ] config reload / daemon restart

### Step 10: 整合 Redfish / IPMI / Event
- [ ] Redfish fan / thermal status 正確
- [ ] IPMI sensor reading 正確
- [ ] SEL / Journal event 正確

## 25.13 D-Bus / systemd 驗證指令

```bash
# Fan / thermal 相關 service
systemctl list-units | grep -Ei 'fan|pid|thermal|swampd'

# 常見 service 狀態
systemctl status phosphor-pid-control.service --no-pager 2>/dev/null || true
systemctl status phosphor-fan-control@0.service --no-pager 2>/dev/null || true
systemctl status phosphor-fan-monitor.service --no-pager 2>/dev/null || true
systemctl status phosphor-fan-presence-tach.service --no-pager 2>/dev/null || true

# Journal
journalctl -b | grep -Ei 'fan|pid|thermal|failsafe|swampd|tach|pwm'

# D-Bus sensor tree
busctl tree xyz.openbmc_project.ObjectMapper | grep -Ei 'fan|thermal|temperature|tach|pwm'

# Fan tach value
busctl get-property \
  xyz.openbmc_project.FanSensor \
  /xyz/openbmc_project/sensors/fan_tach/Fan0 \
  xyz.openbmc_project.Sensor.Value \
  Value

# Sensor availability / functional
busctl introspect \
  xyz.openbmc_project.FanSensor \
  /xyz/openbmc_project/sensors/fan_tach/Fan0
```

若 service name 不同, 先用 ObjectMapper 查 owner:

```bash
busctl call xyz.openbmc_project.ObjectMapper \
  /xyz/openbmc_project/object_mapper \
  xyz.openbmc_project.ObjectMapper \
  GetSubTree sias \
  /xyz/openbmc_project/sensors 0 1 xyz.openbmc_project.Sensor.Value
```

## 25.14 sysfs / 手動 PWM 驗證

手動 PWM 測試前需確認平台允許人工寫入 PWM, 且沒有另一個 daemon 同時覆寫. 建議先停止 fan control daemon, 保留 fan monitor 或視測試目的調整.

```bash
# 找出 fan hwmon
for i in /sys/class/hwmon/hwmon*; do
    echo "$i: $(cat "$i/name" 2>/dev/null)"
done

HWMON=/sys/class/hwmon/hwmonX

# 讀 fan tach
cat $HWMON/fan1_input

# 查看 PWM
ls $HWMON/pwm*
cat $HWMON/pwm1

# 若平台支援 pwm enable
cat $HWMON/pwm1_enable 2>/dev/null

# 手動寫入 raw PWM；0~255 僅為常見範圍，實際需看 driver
for v in 80 128 180 220 255; do
    echo $v > $HWMON/pwm1
    sleep 5
    echo "pwm=$v rpm=$(cat $HWMON/fan1_input)"
done
```

手動測試注意事項:

- 先確認 fan 最低啟轉 PWM, 避免低 duty 造成停轉.
- 測試中需持續監控關鍵溫度.
- 測試完成需恢復自動控制 daemon.
- 如果 fan controller 有 watchdog / automatic mode, 需確認手動寫入是否會被硬體覆寫.
- 若 fan control service 會定期寫 PWM, 手動值可能很快被改回.

## 25.15 Redfish / IPMI / Event 驗證

Redfish:

```bash
# Thermal 舊路徑，視 bmcweb build option 而定
curl -k -u root:0penBmc https://<bmc>/redfish/v1/Chassis/<id>/Thermal

# 新版 ThermalSubsystem，視平台支援而定
curl -k -u root:0penBmc https://<bmc>/redfish/v1/Chassis/<id>/ThermalSubsystem
curl -k -u root:0penBmc https://<bmc>/redfish/v1/Chassis/<id>/ThermalSubsystem/Fans
curl -k -u root:0penBmc https://<bmc>/redfish/v1/Chassis/<id>/ThermalSubsystem/ThermalMetrics

# Sensor collection
curl -k -u root:0penBmc https://<bmc>/redfish/v1/Chassis/<id>/Sensors
```

IPMI:

```bash
ipmitool sdr list | grep -Ei 'fan|tach|temp|thermal'
ipmitool sensor | grep -Ei 'fan|tach|temp|thermal'
ipmitool sel list
```

Event 驗證項目:

```text
[ ] fan tach lower critical 會產生日誌或 SEL
[ ] fan removed 會更新 inventory Present
[ ] fan failed 會更新 Functional=false
[ ] required temp sensor unavailable 會進 failsafe
[ ] failsafe entry / recovery 有 journal 可追蹤
[ ] Redfish Status.Health 與內部狀態一致
[ ] IPMI sensor status 與 D-Bus threshold 狀態一致
```

## 25.16 Thermal Tuning 與量測資料

Thermal tuning 建議保存下列資料:

```text
[ ] BMC image version / fan policy version
[ ] BIOS / host firmware version
[ ] CPLD / fan board firmware version
[ ] Fan vendor / model / PPR / max RPM
[ ] Ambient temperature
[ ] Host workload / power level
[ ] Sensor list 與取樣週期
[ ] PWM / RPM / temp trend
[ ] Acoustic measurement, if required
[ ] Power consumption
[ ] Throttle / PROCHOT / thermal event
[ ] Test duration
```

建議測試矩陣:

| 測試 | 條件 | 觀察重點 |
| :--- | :--- | :--- |
| Idle | Host OS idle | fan floor, 噪音, 低溫穩定度. |
| CPU stress | CPU 100% | CPU temp / margin, fan response. |
| Memory stress | DIMM workload | DIMM temp, front fan response. |
| GPU / PCIe stress | 高功耗 add-in card | PCIe / GPU zone airflow. |
| NVMe stress | drive read/write | drive temp, backplane airflow. |
| High ambient | thermal chamber | max fan, 是否 throttle. |
| Fan fail | 拔 fan / 遮斷 tach | failsafe, event, 冗餘補償. |
| Sensor fail | mask sensor / stop daemon | failsafe, recovery. |
| AC cycle | cold boot | boot fan policy. |
| Service restart | restart fan service | offline failsafe, recovery. |

Thermal tuning 建議輸出圖表:

```text
- time vs CPU/DIMM/VR/GPU/NVMe temperature
- time vs fan PWM
- time vs fan RPM
- time vs power
- time vs thermal margin
- event timeline
```

## 25.17 進階除錯與常見陷阱

| 問題現象 | 建議排查方向 | 排查 / 解法 |
| :--- | :--- | :--- |
| Fan 一直全速 | Failsafe 未解除, required sensor missing, host state unknown, config parse fail | 查 journal, failsafe flag, ObjectMapper, sensor Availability. |
| Fan 完全不轉 | PWM polarity 錯, fan power rail 未開, enable pin 錯, fan presence 判斷錯 | 查 sysfs pwm, 示波器量 PWM, 確認 fan power. |
| RPM 讀值為 0 | tach pin / pull-up / PPR 錯, fan 未轉, driver channel 錯 | 查 fan*_input, 示波器量 tach, 確認 DTS channel. |
| RPM 亂跳 | tach noise, PPR 錯, PWM 太低, fan 老化 | 增加濾波, 提高 minimum PWM, 比對外部轉速計. |
| 溫度高但 fan 不升速 | zone mapping 錯, sensor 未列入 policy, profile 錯, host state 錯 | 查 zone config, policy log, D-Bus sensor path. |
| fan speed 震盪 | PID 參數不穩, hysteresis 不足, sensor noise | 降低 Kp, 調整 Ki/Kd, 加入 hold time. |
| 降溫後 fan 不降 | failsafe 未 recovery, Ki windup, down slew rate 太小 | 查 failsafe reason, PID log, anti-windup. |
| 手動 PWM 被改回 | fan control daemon 仍在寫入, hardware auto mode | 暫停 daemon 或切 manual mode; 確認 controller mode. |
| Redfish 看不到 fan | association / inventory / bmcweb mapping 不完整 | 查 D-Bus sensor, inventory association, bmcweb log. |
| IPMI SDR 缺 fan | sensor type / naming / sensor map 不完整 | 查 dbus-sdr, ipmid log, sensor path. |
| service restart 後 fan 低速 | offline failsafe 未設定, service dependency 不完整 | 設 boot default PWM, offline failsafe, systemd ordering. |

## 25.18 Fan Control Porting 驗收 Checklist

### 硬體 / 線路
- [ ] Fan power rail、PWM pin、tach pin、presence pin 確認
- [ ] Fan vendor / model / PPR / min RPM / max RPM 確認
- [ ] PWM polarity、frequency、duty range 確認
- [ ] Fan tray / rotor / redundancy 設計確認
- [ ] 示波器量測 PWM 與 tach waveform

### Kernel / sysfs
- [ ] PWM / tach driver probe 正常
- [ ] fan*_input 讀值合理
- [ ] pwm* 可寫入且 fan RPM 有對應變化
- [ ] pwm*_enable / automatic mode 行為確認
- [ ] DTS channel 與實體 fan 對應正確

### D-Bus / Inventory
- [ ] fan_tach sensor path 正確
- [ ] temperature sensor path 正確
- [ ] Fan inventory Present 正確
- [ ] Fan Functional 正確
- [ ] Availability 與 sensor missing 行為正確

### Policy / Control
- [ ] Host Off policy 完成
- [ ] Boot policy 完成
- [ ] Host On PID / stepwise policy 完成
- [ ] Failsafe trigger 與 output 完成
- [ ] Zone mapping 完成
- [ ] Fan group / sensor group 完成
- [ ] min / max / floor / ceiling 完成
- [ ] slew rate / hysteresis / hold time 完成
- [ ] profile / thermal mode 完成

### 異常測試
- [ ] 拔 fan 進入 fault / failsafe
- [ ] tach lost 進入 fault / failsafe
- [ ] required temp sensor missing 進入 failsafe
- [ ] control daemon restart 時 fan 維持安全風速
- [ ] config reload 後 policy 正常恢復
- [ ] host state unknown 時採安全策略

### 對外介面
- [ ] Redfish Thermal / ThermalSubsystem / Sensors 顯示正確
- [ ] IPMI SDR / sensor reading 正確
- [ ] SEL / journal event 正確
- [ ] fan fail / recovery event 可追蹤

### Tuning / 量產
- [ ] idle / stress / high ambient 測試完成
- [ ] acoustic baseline 完成
- [ ] thermal tuning log 保存
- [ ] policy version 與 image version 對齊
- [ ] AC cycle / DC cycle / BMC reboot 測試完成

## 25.19 Fan Control 資料表範本

| Zone | Fan Group | Fan | PWM Channel | Tach Channel | PPR | Min PWM | Max PWM | Min RPM | Max RPM | Required Sensors | Optional Sensors | Failsafe PWM | Profile | 備註 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| CPU Zone | Front Bank | Fan0 | pwm1 | fan1_input | [待填] | [待填] | [待填] | [待填] | [待填] | CPU0_Temp | Inlet_Temp | 100% | Balanced | [待填] |
| CPU Zone | Front Bank | Fan1 | pwm2 | fan2_input | [待填] | [待填] | [待填] | [待填] | [待填] | CPU0_Temp | Inlet_Temp | 100% | Balanced | [待填] |
| DIMM Zone | Front Bank | Fan2 | pwm3 | fan3_input | [待填] | [待填] | [待填] | [待填] | [待填] | DIMM_A0_Temp | Inlet_Temp | 100% | Balanced | [待填] |

## 25.20 本章參考資料

- OpenBMC phosphor-pid-control: https://github.com/openbmc/phosphor-pid-control
- OpenBMC phosphor-pid-control configure.md: https://github.com/openbmc/phosphor-pid-control/blob/master/configure.md
- OpenBMC phosphor-pid-control tuning.md: https://github.com/openbmc/phosphor-pid-control/blob/master/tuning.md
- OpenBMC phosphor-fan-presence: https://github.com/openbmc/phosphor-fan-presence
- OpenBMC phosphor-fan-presence control configuration: https://github.com/openbmc/phosphor-fan-presence/blob/master/docs/control/README.md
- OpenBMC dbus-sensors: https://github.com/openbmc/dbus-sensors
- OpenBMC bmcweb Redfish implementation: https://github.com/openbmc/bmcweb
