# 29. 各類 Sensor 共用除錯指令及附錄

本節整理各類 Sensor 共用的除錯入口，適用於 ADC、Temperature、Voltage、Current、Power、Fan Tach、Fan PWM、PSU、CPU、NVMe、GPU、External 與 Presence 類型。實務上建議依「硬體訊號 → kernel / sysfs → sensor daemon → D-Bus → association / inventory → Redfish / IPMI → event / log」的順序排查，避免只看單一介面造成判讀落差。

## 29.1 除錯路徑總覽

Sensor 問題通常可以先切成六層：

| 層級 | 主要確認項目 | 常見工具 |
| ---- | ------------ | -------- |
| 硬體 / bus | 電源、reset、I2C ACK、PMBus page、ADC channel、tach / PWM 波形、GPIO polarity | 示波器、DMM、LA、`i2cdetect`、CPLD register |
| Kernel / driver | driver probe、Device Tree、hwmon / iio / gpio / peci / mctp node | `dmesg`、`ls /sys/class/hwmon`、`find /sys` |
| sysfs | `*_input`、`*_label`、`*_enable`、`pwm*`、單位與 scale | `cat`、`grep`、`find` |
| OpenBMC daemon | Entity Manager config、dbus-sensors / phosphor-hwmon service、power state gating | `systemctl`、`journalctl` |
| D-Bus | object path、service name、Value、Unit、Availability、Functional、Threshold | `busctl tree`、ObjectMapper `GetSubTree`、`get-property` |
| 外部介面 | Redfish / IPMI 是否有值、狀態、閾值、inventory association、event | `curl`、`ipmitool`、Redfish validator |

建議先保存下列資訊，方便跨 HW / BMC / BIOS / CPLD / ME / FW 同步：

- BMC image version、kernel commit、DTS commit、Entity Manager JSON commit。
- sensor 名稱、D-Bus path、Redfish URI、IPMI SDR name。
- bus / mux channel / address / page / channel / label / hwmon name。
- 量測值、sysfs 值、D-Bus 值、Redfish 值、IPMI 值。
- service status、journal、dmesg、ObjectMapper 查詢結果。
- host power state、device presence、CPLD / PSU / VR fault status。

## 29.2 Kernel / sysfs

Kernel / sysfs 層要先確認 driver 是否有 probe、hwmon / iio / gpio / peci / mctp node 是否存在，以及 sysfs 數值單位是否符合 Linux hwmon 慣例。

```bash
# sensor / hwmon 相關 kernel log
dmesg | grep -i sensor
dmesg | grep -i hwmon
dmesg | grep -Ei 'i2c|pmbus|adc|iio|peci|tach|fan|thermal|nvme|mctp|gpio'

# hwmon 裝置清單
ls -l /sys/class/hwmon/
cat /sys/class/hwmon/hwmon*/name

# 找出所有常見 sensor input
find /sys/class/hwmon -name '*_input' -o -name '*_label' -o -name 'pwm*' | sort

# 顯示 hwmon name 與 input 對照
for h in /sys/class/hwmon/hwmon*; do
    echo "== $h =="
    cat "$h/name" 2>/dev/null
    for f in "$h"/*_label "$h"/*_input "$h"/pwm*; do
        [ -e "$f" ] && echo "$(basename "$f")=$(cat "$f" 2>/dev/null)"
    done
done
```

常見 hwmon 檔名與單位：

| sysfs 類型 | 常見檔案 | 常見單位 / 意義 | 注意事項 |
| ---------- | -------- | --------------- | -------- |
| Voltage | `in*_input` | mV | 電阻分壓、VR page、PSU label 需由平台設定補正 |
| Temperature | `temp*_input` | millidegree Celsius | 例：`30125` 代表 30.125°C |
| Current | `curr*_input` | mA | shunt resistance、PMBus format、driver scale 需確認 |
| Power | `power*_input` | µW | OpenBMC 顯示常轉成 W，需避免 double scale |
| Fan Tach | `fan*_input` | RPM | PPR、tach polarity、dual rotor mapping 需確認 |
| PWM | `pwm*` | raw duty，常見 0–255 | 不同 driver 的 `pwm*_enable` 語意可能不同 |
| Energy | `energy*_input` | µJ 或 driver 定義 | Redfish / telemetry 使用前需確認累積範圍與 rollover |

sysfs 常見判讀方向：

- `hwmon*/name` 存在但沒有對應 `*_input`：driver 已 probe，但該 channel 未啟用、label 不符、硬體不支援、page / phase 未選到。
- `*_input` 固定 0：可能是硬體未上電、bus 讀到 0、scale / channel 錯、sensor daemon power state gating。
- sysfs 有值但 D-Bus 沒 sensor：多半在 Entity Manager config、daemon matching、service 啟動或 JSON type / index / label 對應。
- D-Bus 有值但 Redfish 沒值：常見在 chassis / inventory association、bmcweb path mapping、Redfish feature flag 或 service cache。

## 29.3 I2C / SMBus / PMBus

I2C / SMBus / PMBus 排查前要先確認 bus number、mux channel、address、device power state、host / BMC ownership，以及該 device 是否允許被掃描。部分 PMBus、EEPROM、VR、MCU 裝置在不合適的時間讀取可能造成狀態改變或拉長 bus timeout，`i2cdetect`、`i2cdump` 應在知道風險後使用。

```bash
# 列出 I2C bus
ls -l /dev/i2c-*
i2cdetect -l

# 掃描指定 bus；執行前需確認該 bus 上 device 可接受 probing
i2cdetect -y <bus>

# 讀取單一 register；PMBus 需確認 command/page
i2cget -y <bus> <addr> <reg>

# dump register；可能對部分 device 有副作用，建議只在已確認安全時使用
i2cdump -y <bus> <addr>

# 常見 PMBus：先確認 page，再讀 command；實際 command 依 PMBus / vendor spec
# PAGE=0x00, READ_VOUT=0x8B, READ_IOUT=0x8C, READ_TEMPERATURE_1=0x8D,
# READ_FAN_SPEED_1=0x90, READ_POUT=0x96, READ_PIN=0x97, STATUS_WORD=0x79
```

I2C / PMBus 檢查表：

| 項目 | 確認方式 | 常見問題 |
| ---- | -------- | -------- |
| Bus number | `i2cdetect -l`、DTS、schematic | BMC kernel bus 編號與 schematic bus name 不同 |
| Mux channel | `/sys/bus/i2c/devices`、mux driver log | mux 未 probe、channel 未 enable、上層 bus 無 power |
| Address | schematic、datasheet、`i2cdetect` | 7-bit / 8-bit address 混淆、strap 錯 |
| Driver binding | `dmesg`、`/sys/bus/i2c/drivers` | compatible 不符、driver 未啟用、manual bind 未做 |
| PMBus page | datasheet、vendor spec | page / phase 錯造成 label 與值對不上 |
| Fault bit | PMBus `STATUS_WORD` / vendor register | present 但 fault，或 input lost 被誤判成 absent |
| Timing | bus speed、clock stretching、timeout | 100k / 400k 不符、device stretch 過長 |

## 29.4 systemd service

Sensor daemon 通常由 Entity Manager 或各 sensor service 依設定建立 D-Bus object。不同平台可能使用 dbus-sensors、phosphor-hwmon 或 vendor daemon，實際 service 名稱需以 image 內容為準。

```bash
# 找出 sensor 相關 service
systemctl list-units --type=service | grep -Ei 'sensor|hwmon|fan|power|psu|entity|thermal|inventory|fru'

# 常見服務狀態
systemctl status xyz.openbmc_project.EntityManager.service
systemctl status xyz.openbmc_project.ADCSensor.service
systemctl status xyz.openbmc_project.HwmonTempSensor.service
systemctl status xyz.openbmc_project.FanSensor.service
systemctl status xyz.openbmc_project.PSUSensor.service
systemctl status xyz.openbmc_project.ExternalSensor.service
systemctl status xyz.openbmc_project.IntelCPUSensor.service
systemctl status xyz.openbmc_project.NVMeSensor.service
systemctl status bmcweb.service
```

常見檢查方向：

- service `inactive`：可能套件未進 image、feature 未啟用、被 condition / power state 限制。
- service `failed`：先看 `journalctl -u`，再確認 JSON schema、bus / address、權限、D-Bus name 衝突。
- service 正常但沒有 object：確認 Entity Manager 的 `Exposes` 是否有匹配該 sensor type，以及 daemon 是否支援該 type。
- service restart 後短暫有值再消失：可能是 device absent、read timeout、threshold parse fail、association 或 availability policy。

## 29.5 journal

journal 的目標是比對 service 啟動流程、config reload、device matching、讀值失敗、threshold alarm 與 bmcweb 查詢行為。

```bash
# 本次開機 sensor 相關 log
journalctl -b | grep -Ei 'sensor|hwmon|entity|threshold|fan|pwm|psu|pmbus|peci|nvme|gpu|redfish'

# 常見 service log
journalctl -u xyz.openbmc_project.EntityManager.service -b --no-pager
journalctl -u xyz.openbmc_project.ADCSensor.service -b --no-pager
journalctl -u xyz.openbmc_project.HwmonTempSensor.service -b --no-pager
journalctl -u xyz.openbmc_project.FanSensor.service -b --no-pager
journalctl -u xyz.openbmc_project.PSUSensor.service -b --no-pager
journalctl -u xyz.openbmc_project.ExternalSensor.service -b --no-pager
journalctl -u bmcweb.service -b --no-pager

# 追蹤即時變化
journalctl -f | grep -Ei 'sensor|threshold|fan|psu|redfish|bmcweb'
```

建議保存：

- service start / restart 時間點。
- 讀取失敗訊息，例如 permission denied、timeout、invalid configuration、missing item。
- threshold alarm 上升 / 解除時間點。
- sensor value 變成 NaN、Unavailable、Functional=false 的前後 log。
- Redfish request 的 HTTP status 與 bmcweb log。

## 29.6 D-Bus / ObjectMapper

OpenBMC sensor 的核心觀念是把 sensor 映射成 D-Bus object。典型 path 為 `/xyz/openbmc_project/sensors/<type>/<sensor_name>`，常見 interface 包含 `xyz.openbmc_project.Sensor.Value`、`xyz.openbmc_project.Sensor.Threshold.Warning`、`xyz.openbmc_project.Sensor.Threshold.Critical`、`xyz.openbmc_project.State.Decorator.Availability`、`xyz.openbmc_project.State.Decorator.OperationalStatus` 與 `xyz.openbmc_project.Association.Definitions`。

```bash
# 快速列出 sensors D-Bus tree
busctl tree | grep /xyz/openbmc_project/sensors
busctl tree xyz.openbmc_project.ObjectMapper | grep sensors

# 由 ObjectMapper 查出所有 sensor objects 與 service
busctl call xyz.openbmc_project.ObjectMapper \
  /xyz/openbmc_project/object_mapper \
  xyz.openbmc_project.ObjectMapper GetSubTree \
  sias /xyz/openbmc_project/sensors 0 0

# 只查有 Sensor.Value interface 的 objects
busctl call xyz.openbmc_project.ObjectMapper \
  /xyz/openbmc_project/object_mapper \
  xyz.openbmc_project.ObjectMapper GetSubTree \
  sias /xyz/openbmc_project/sensors 0 1 \
  xyz.openbmc_project.Sensor.Value

# 已知 sensor path 時，先查 service name
busctl call xyz.openbmc_project.ObjectMapper \
  /xyz/openbmc_project/object_mapper \
  xyz.openbmc_project.ObjectMapper GetObject \
  sas <sensor_path> 0

# 讀取 Value / Unit / Availability / Functional
busctl get-property \
  <service_name> \
  <sensor_path> \
  xyz.openbmc_project.Sensor.Value \
  Value

busctl get-property \
  <service_name> \
  <sensor_path> \
  xyz.openbmc_project.Sensor.Value \
  Unit

busctl get-property \
  <service_name> \
  <sensor_path> \
  xyz.openbmc_project.State.Decorator.Availability \
  Available

busctl get-property \
  <service_name> \
  <sensor_path> \
  xyz.openbmc_project.State.Decorator.OperationalStatus \
  Functional

# 查看完整 interface / property
busctl introspect <service_name> <sensor_path>
```

D-Bus 判讀重點：

| 現象 | 可能方向 | 建議確認 |
| ---- | -------- | -------- |
| 找不到 sensor path | daemon 未建立 object、config type / index 不符、power state 不符 | service log、Entity Manager exposed config、sysfs |
| path 存在但 `Value` 讀不到 | interface 未掛上、daemon 初始化部分失敗 | `busctl introspect`、journal |
| `Available=false` | device absent、host off、timeout、policy gating | presence、host state、daemon log |
| `Functional=false` | sensor fault、讀值不可信、threshold / monitor 判定異常 | fault register、journal、threshold status |
| Threshold property 不存在 | JSON 未設定、daemon 不支援、sensor type 不使用 threshold | config、phosphor-dbus-interfaces |
| `Unit` 與預期不一致 | sensor type 錯、daemon mapping 錯 | D-Bus path type、JSON Type、bmcweb mapping |

## 29.7 Redfish

Redfish 層主要確認三件事：chassis 是否存在、sensor 是否掛到正確 chassis / inventory association、bmcweb 是否把 D-Bus sensor type 映射到對應 Redfish resource。

```bash
# Chassis collection
curl -k -u root:<password> \
  https://<bmc_ip>/redfish/v1/Chassis/

# 指定 chassis
curl -k -u root:<password> \
  https://<bmc_ip>/redfish/v1/Chassis/<chassis_id>

# 新式 Sensors collection；是否可用取決於 bmcweb build / platform policy
curl -k -u root:<password> \
  https://<bmc_ip>/redfish/v1/Chassis/<chassis_id>/Sensors

# Legacy / 相容路徑；不少平台仍需要同時驗證
curl -k -u root:<password> \
  https://<bmc_ip>/redfish/v1/Chassis/<chassis_id>/Thermal

curl -k -u root:<password> \
  https://<bmc_ip>/redfish/v1/Chassis/<chassis_id>/Power

# 若有 jq，可快速查看 sensor members
curl -k -u root:<password> \
  https://<bmc_ip>/redfish/v1/Chassis/<chassis_id>/Sensors | jq '.Members'
```

Redfish 與 D-Bus 常見映射：

| Redfish resource | 常見 D-Bus sensor type | 說明 |
| ---------------- | ---------------------- | ---- |
| `/Chassis/<id>/Sensors` | 多數 sensor type | 新式 unified sensor collection，依 bmcweb feature / schema 支援度而定 |
| `/Chassis/<id>/Thermal` | `temperature`、`fan_tach`、`fan_pwm` | legacy thermal view，仍常用於相容性驗證 |
| `/Chassis/<id>/Power` | `voltage`、`power` | legacy power view，部分 current / PSU 資訊可能走其他 resource 或 OEM 欄位 |
| `/Chassis/<id>/PowerSubsystem` | power supply / power control 資訊 | 依平台與 bmcweb 支援度而定 |
| `/Chassis/<id>/ThermalSubsystem` | fan / thermal control 資訊 | 依平台與 bmcweb 支援度而定 |

Redfish 若沒有 sensor，但 D-Bus 有值，優先確認：

- sensor path 是否在 `/xyz/openbmc_project/sensors/<type>/...`。
- sensor 是否有 chassis `all_sensors` association。
- sensor 是否有 inventory association，尤其 fan、PSU、VR、CPU、drive 類 sensor。
- bmcweb 是否支援該 sensor type 與目標 resource。
- chassis id 是否正確，不同平台可能是 `chassis`、`system`、`baseboard` 或專案自訂名稱。
- `bmcweb` 是否需要 restart 或等待 ObjectMapper 更新。

## 29.8 IPMI / SDR / SEL

若平台仍提供 IPMI，需確認 D-Bus sensor 是否被 IPMI SDR policy 收錄，單位、線性化、threshold、event type 是否與需求一致。

```bash
# Sensor 清單與讀值
ipmitool -I lanplus -H <bmc_ip> -U root -P <password> sensor list
ipmitool -I lanplus -H <bmc_ip> -U root -P <password> sdr elist

# SEL / event
ipmitool -I lanplus -H <bmc_ip> -U root -P <password> sel list
ipmitool -I lanplus -H <bmc_ip> -U root -P <password> sel elist

# 本機 BMC 端若有 ipmitool
ipmitool sensor list
ipmitool sdr elist
ipmitool sel elist
```

IPMI 排查方向：

| 現象 | 可能方向 | 建議確認 |
| ---- | -------- | -------- |
| D-Bus 有 sensor，但 IPMI 無 | SDR config / allowlist / entity mapping 未收錄 | `phosphor-host-ipmid` log、SDR policy |
| IPMI 名稱截斷 | SDR name 長度限制 | 命名規則、縮寫表 |
| IPMI 值 scale 錯 | linearization / M-B-R / unit 設定 | SDR dump、D-Bus raw value |
| threshold 事件沒進 SEL | event policy、threshold alarm、SEL service | `busctl introspect` threshold、SEL log |
| Sensor 顯示 `na` | unavailable / host state gating / reading failed | D-Bus `Available`、`Functional`、service log |

## 29.9 Entity Manager / configuration

Entity Manager 或平台 sensor config 是 sysfs / hardware 與 D-Bus object 的橋接。不同 sensor type 的欄位不同，但共通檢查方向一致。

```bash
# 找平台 sensor / entity config；實際路徑依 image 而定
find /usr/share -iname '*.json' | grep -Ei 'entity|sensor|config|platform'
find /etc -iname '*.json' | grep -Ei 'entity|sensor|config|platform'

# Entity Manager exposed objects
busctl tree xyz.openbmc_project.EntityManager
busctl introspect xyz.openbmc_project.EntityManager /xyz/openbmc_project/inventory

# 查 config 中特定 sensor name
grep -R "<sensor_name>" /usr/share /etc 2>/dev/null
```

共通欄位檢查：

| 欄位 | 檢查重點 |
| ---- | -------- |
| `Name` | 需符合 D-Bus object path 字元限制，避免空白、斜線、特殊字元 |
| `Type` | 需與 sensor daemon 支援的 type 完全一致 |
| `Bus` / `Address` | I2C / PMBus 類 sensor 必填；需對齊 mux 後的 runtime bus number |
| `Index` | ADC / hwmon input 類常用；需對齊 `inX_input`、`tempX_input`、`fanX_input` |
| `ScaleFactor` / `Offset` | 需記錄硬體換算依據與量測比對結果 |
| `PowerState` | `Always`、`On`、`BiosPost` 等需對齊 sensor 實際可讀時機 |
| `Thresholds` | warning / critical / direction / hysteresis 需對齊規格 |
| `PollRate` / `PollInterval` | 過短可能造成 bus loading，過長可能影響 event 反應 |
| Association / Inventory | Redfish / IPMI 可見性常受此影響 |

## 29.10 常見一次性收集腳本

以下腳本適合在問題單初期收集 baseline。正式使用前可依平台裁切，避免包含敏感資訊。

```bash
#!/bin/sh
OUT=/tmp/sensor_debug_$(date +%Y%m%d_%H%M%S)
mkdir -p "$OUT"

uname -a > "$OUT/uname.txt"
cat /etc/os-release > "$OUT/os-release.txt" 2>/dev/null

# Kernel / sysfs
dmesg > "$OUT/dmesg.txt"
ls -l /sys/class/hwmon > "$OUT/hwmon-list.txt" 2>&1
for h in /sys/class/hwmon/hwmon*; do
    [ -d "$h" ] || continue
    name=$(cat "$h/name" 2>/dev/null)
    safe=$(echo "$name" | tr '/ ' '__')
    dir="$OUT/hwmon_${safe}_$(basename "$h")"
    mkdir -p "$dir"
    cp "$h/name" "$dir/name" 2>/dev/null
    for f in "$h"/*_input "$h"/*_label "$h"/*_min "$h"/*_max "$h"/*_crit "$h"/*_lcrit "$h"/pwm* "$h"/fan*_pulses; do
        [ -e "$f" ] && cat "$f" > "$dir/$(basename "$f")" 2>/dev/null
    done
done

# systemd / journal
systemctl --failed > "$OUT/systemctl-failed.txt" 2>&1
systemctl list-units --type=service | grep -Ei 'sensor|hwmon|fan|power|psu|entity|thermal|inventory|fru|bmcweb' > "$OUT/sensor-services.txt" 2>&1
journalctl -b > "$OUT/journal-b.txt" 2>&1

# D-Bus
busctl tree > "$OUT/busctl-tree.txt" 2>&1
busctl call xyz.openbmc_project.ObjectMapper /xyz/openbmc_project/object_mapper xyz.openbmc_project.ObjectMapper GetSubTree sias /xyz/openbmc_project/sensors 0 1 xyz.openbmc_project.Sensor.Value > "$OUT/dbus-sensors.txt" 2>&1

# config
find /usr/share /etc -iname '*.json' 2>/dev/null | grep -Ei 'entity|sensor|platform|config' > "$OUT/json-list.txt"

tar czf "$OUT.tar.gz" -C /tmp "$(basename "$OUT")"
echo "$OUT.tar.gz"
```

## 29.11 Sensor Type 快速對照

| Sensor 種類 | 常見 D-Bus type | 常見來源 | 主要確認項目 | 常見 sysfs / 介面 |
| ----------- | --------------- | -------- | ------------ | ----------------- |
| ADC | `voltage` | IIO / `iio-hwmon` | ADC channel、Vref、scale、分壓比例 | `in*_input` |
| Temperature | `temperature` | hwmon / PECI / PMBus / NVMe-MI | `temp*_input`、單位、offset、threshold | `temp*_input` |
| Voltage | `voltage` | ADC / PMBus / hwmon | rail tolerance、scale、page / label | `in*_input` |
| Current | `current` | PMBus / hwmon / shunt monitor | shunt、scale、單位、page / phase | `curr*_input` |
| Power | `power` | PMBus / PECI / GPU / V×I | µW → W、power meter 比對、是否 double count | `power*_input` |
| Energy | `energy` | PMBus / telemetry / accumulator | 累積單位、rollover、reset 行為 | `energy*_input` |
| Fan Tach | `fan_tach` | hwmon fan input / fan controller | RPM、PPR、rotor、lower threshold | `fan*_input` |
| Fan PWM | `fan_pwm` 或 control interface | `pwm` sysfs / fan controller | duty range、polarity、thermal policy | `pwm*` |
| PSU | 多種類型 | PMBus / hwmon / FRU / GPIO | presence、fault、redundancy、FRU | PMBus / `hwmon` / inventory |
| CPU | `temperature` / `power` | PECI / APML / hwmon | host power state、CPU presence、readiness | `peci-*`、`temp*_input` |
| NVMe | `temperature` / health | NVMe-MI / MCTP / PCIe | slot mapping、drive presence、endpoint | MCTP / daemon D-Bus |
| GPU | `temperature` / `power` / utilization | MCTP / telemetry / vendor daemon | endpoint、presence、fault、inventory | daemon D-Bus / telemetry |
| External | 依設定 | D-Bus / external daemon | timeout、unavailable policy、更新頻率 | D-Bus property |
| Presence | state / inventory | GPIO / CPLD / PMBus ACK / FRU | active level、debounce、present vs functional | libgpiod / inventory |

## 29.12 常見問題索引

| 問題 | 第一輪判斷 | 建議入口 |
| ---- | ---------- | -------- |
| `hwmon` 不見 | driver 未 probe、DT / kernel config / I2C 問題 | `dmesg`、`i2cdetect`、`/sys/bus/i2c/devices` |
| sysfs 有值但 D-Bus 沒值 | config / daemon matching 問題 | Entity Manager JSON、`journalctl -u <sensor service>` |
| D-Bus 有值但 Redfish 無值 | association / bmcweb mapping 問題 | ObjectMapper association、`journalctl -u bmcweb` |
| Redfish 有 sensor 但 `Reading` 為 null | D-Bus unavailable / functional false | `Available`、`Functional`、power state |
| IPMI 無 sensor | SDR policy / naming / entity mapping | `phosphor-host-ipmid` log、`ipmitool sdr elist` |
| 數值倍率錯 | sysfs 單位、ScaleFactor、Redfish/IPMI scale | sysfs ↔ D-Bus ↔ Redfish ↔ 量測值比對 |
| threshold 不觸發 | threshold 未掛 interface、direction / hysteresis / event policy | `busctl introspect`、journal、SEL |
| fan RPM 不合理 | PPR、tach channel、PWM mapping、波形問題 | `fan*_input`、示波器、PWM manual test |
| PSU present 但無讀值 | input power lost、PMBus fault、page wrong | PMBus status、FRU、presence GPIO |
| host off 時 sensor 消失 | `PowerState` / host gating 符合設定或設定過嚴 | Entity Manager config、host state |

## 29.13 Porting / Debug Checklist

- [ ] 硬體：sensor 型號、bus / address / channel / page、供電、reset、presence、fault、量測點已確認。
- [ ] Device Tree：compatible、reg、pinctrl、io-channels、pulses-per-revolution、mux、status 已確認。
- [ ] Kernel config：hwmon / iio / pmbus / peci / gpio / pwm / mctp 相關選項已進 image。
- [ ] sysfs：`hwmon*/name`、`*_input`、`*_label`、`pwm*` 存在且單位正確。
- [ ] config：Entity Manager JSON 的 `Name`、`Type`、`Bus`、`Address`、`Index`、`ScaleFactor`、`PowerState`、`Thresholds` 已對齊實機。
- [ ] service：sensor daemon 啟動正常，journal 無持續 timeout / parse error / unavailable loop。
- [ ] D-Bus：ObjectMapper 查得到 sensor，`Value`、`Unit`、`Available`、`Functional`、threshold interface 符合預期。
- [ ] association：chassis `all_sensors` 與 inventory `sensors` association 已建立。
- [ ] Redfish：`/Chassis/<id>/Sensors`、`Thermal`、`Power` 依平台 policy 可回應。
- [ ] IPMI：若平台支援 IPMI，`sensor list`、`sdr elist`、SEL threshold event 已驗證。
- [ ] 量測比對：DMM / power meter / thermal chamber / tach meter 與 sysfs / D-Bus / Redfish 數值已建立誤差 baseline。
- [ ] 異常測試：device absent、bus timeout、host off、PSU unplug、fan stall、threshold crossing、service restart 已驗證。

## 29.14 參考資料

- OpenBMC Sensor Architecture  
  <https://github.com/openbmc/docs/blob/master/architecture/sensor-architecture.md>

- OpenBMC dbus-sensors  
  <https://github.com/openbmc/dbus-sensors>

- OpenBMC ObjectMapper Architecture  
  <https://github.com/openbmc/docs/blob/master/architecture/object-mapper.md>

- Linux hwmon sysfs interface  
  <https://docs.kernel.org/hwmon/sysfs-interface.html>

- OpenBMC bmcweb Redfish sensor / power / thermal implementation  
  <https://github.com/openbmc/bmcweb>
