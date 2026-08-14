# 24. Sensor 抽象層

> 文件使用原則：本章提供可攜式設計與排查框架；`Type`、`PowerState`、service 名稱與 JSON 欄位仍應以目標 image 的 schema、systemd unit 與 source tree 為準。表格中的 `[待填]` 代表平台專屬量測或 mapping，不以假設值代填。

Sensor 資料流: driver/hwmon → userspace sensor daemon → D-Bus object → Redfish/IPMI/SEL/Event. Sensor 狀態需區分: 正常、不可用、讀值超界、device 不存在、bus error、timeout.

Sensor 欄位範本:

| Name   | Type             | Source    | Scale  | Unit    | Warning | Critical | D-Bus path | Redfish | IPMI SDR |
| ------ | ---------------- | --------- | ------ | ------- | ------: | -------: | ---------- | ------- | -------- |
| [待填] | temp/voltage/fan | hwmon/i2c | [待填] | C/V/RPM |  [待填] |   [待填] | [待填]     | [待填]  | [待填]   |

## 適用範圍

本章涵蓋 OpenBMC sensor abstraction, 包括 ADC、temperature、voltage、current、power、fan tach、fan PWM、PSU、CPU、NVMe、GPU、external / virtual sensor、presence / intrusion / GPIO state sensor, 以及 D-Bus、Redfish、IPMI、threshold、event 與共用排查方式.

## 適用讀者

- 負責 BMC 硬體 bring-up、Linux hwmon / IIO、Entity Manager、dbus-sensors、Redfish / IPMI 或平台驗證的人員.
- 需要從硬體量測點、kernel sysfs、sensor daemon 追查至 D-Bus 與外部管理介面的人員.

## 快速導覽

- [Sensor 共通架構](#section-12-1)
- [Sensor 種類總覽](#section-12-2)
- [ADC Sensor](#section-12-3)
- [Temperature Sensor](#section-12-4)
- [Voltage Sensor](#section-12-5)
- [Current Sensor](#section-12-6)
- [Power Sensor](#section-12-7)
- [Fan Tach Sensor](#section-12-8)
- [Fan PWM / Fan Control](#section-12-9)
- [PSU Sensor](#section-12-10)
- [CPU Sensor](#section-12-11)
- [NVMe、GPU 與 Virtual Sensor](#section-12-12)
- [Presence / Intrusion / GPIO State Sensor](#section-12-15)
- [Redfish Association](#section-12-16)
- [共用除錯指令與附錄](#section-12-17)

<a id="section-12-1"></a>

## 24.1 Sensor 目的與共通架構

Sensor 抽象層的目的，是把不同硬體來源、driver 與傳輸協定收斂成一致的 D-Bus 模型，讓 Redfish、IPMI、事件紀錄、風扇控制與 power policy 不必理解底層晶片差異。設計時應同時保留「讀值」、「單位」、「有效性」、「健康狀態」、「來源」與「關聯」，避免只發布一個數值卻無法判斷其可信度。

### 24.1.1 分層責任

```mermaid
flowchart LR
    subgraph HW[硬體與傳輸層]
        A1[Sensor IC / ADC / GPIO]
        A2[I2C / PMBus / PECI / MCTP]
        A3[Power / Presence / Mux]
    end
    subgraph KERNEL[Linux Kernel]
        B1[Device Tree / ACPI]
        B2[Bus + device driver]
        B3[hwmon / IIO / GPIO / hwmon labels]
    end
    subgraph USER[OpenBMC Userspace]
        C1[Entity Manager / platform config]
        C2[Sensor daemon]
        C3[D-Bus Sensor.Value + Threshold + State]
        C4[Association / Inventory]
    end
    subgraph NORTH[北向介面與策略]
        D1[Redfish]
        D2[IPMI SDR / SEL]
        D3[Event / Logging]
        D4[Fan / Thermal / Power policy]
        D5[Telemetry / WebUI]
    end
    A1 --> A2 --> B2
    A3 -. enable / present .-> A1
    B1 --> B2 --> B3
    C1 --> C2
    B3 --> C2 --> C3
    C1 --> C4
    C3 --> D1 & D2 & D3 & D4 & D5
    C4 --> D1 & D2
```

各層責任如下：

- 硬體層：定義量測點、電氣範圍、精度、更新率、供電條件與錯誤型態。
- Kernel 層：完成 device enumeration、driver bind、raw value 解碼及標準 sysfs 匯出；不承擔平台顯示名稱或 Redfish chassis 歸屬。
- 設定層：描述 sensor identity、bus/address/index/page、倍率、門檻、PowerState、poll interval 與 inventory association。
- Sensor daemon：讀取來源、轉換單位、判斷 timeout 與有效性、建立 D-Bus object，並發布屬性變更。
- 北向介面：將 D-Bus model 映射到 Redfish、IPMI、SEL、Telemetry；不應再次做未記錄的倍率換算。
- Policy：只能消費已定義 fail-safe 行為的 sensor；關鍵輸入不可只依賴 Value 而忽略 Available / Functional。

### 24.1.2 共通物件模型與命名

每個 sensor 至少應定義以下資料：

| 類別 | 必要內容 | 設計要求 |
|---|---|---|
| Identity | Name、Type、來源裝置、channel/page | 名稱穩定、唯一，避免因 hwmonX 動態編號改變 |
| Value | Value、Unit、MinValue、MaxValue | D-Bus 使用工程單位；來源單位與換算式需可追溯 |
| State | Available、Functional、Present（若適用） | 不以 0、極端值或 NaN 混代不同錯誤狀態 |
| Threshold | Warning/Critical high/low、hysteresis | 門檻來源需標示為規格、thermal policy 或 power budget |
| Timing | poll interval、timeout、stale timeout | timeout 應大於正常更新週期，並定義恢復條件 |
| Topology | inventory / chassis association | 決定 Redfish 歸屬、FRU 關聯與移除行為 |
| Ownership | producer、consumer、validation owner | 避免多個 daemon 同時發布同一路徑或互相覆寫 |

建議命名格式為 `<位置或裝置>_<功能>_<量測種類>`，例如 `CPU0_VCORE_VOLTAGE`、`PSU1_INPUT_POWER`、`FAN2_REAR_TACH`。名稱建立後應視為 API，若需更名，需同步檢查 PID policy、Redfish URI、IPMI SDR、測試腳本與監控系統。

### 24.1.3 狀態與錯誤語意

```mermaid
stateDiagram-v2
    [*] --> Discovering
    Discovering --> Available: source found + first valid sample
    Discovering --> Unavailable: source absent / power gated / probe failed
    Available --> Warning: warning threshold crossed
    Warning --> Critical: critical threshold crossed
    Critical --> Warning: critical deassert + hysteresis
    Warning --> Available: warning deassert + hysteresis
    Available --> Stale: update timeout
    Warning --> Stale: update timeout
    Critical --> Stale: update timeout
    Stale --> Available: valid sample resumes
    Unavailable --> Discovering: presence / power / config changed
    Stale --> Unavailable: source removed or repeated transport failure
```

狀態語意建議統一如下：

- `Present=false`：實體裝置或 FRU 不存在；不應直接等同硬體故障。
- `Available=false`：目前無法取得可信讀值，可能是 power gating、bus error、timeout 或裝置移除。
- `Functional=false`：裝置存在且可被辨識，但功能異常，例如 fan present 但 tach 長期為 0。
- Threshold alarm：代表有效讀值超界；來源不可用時不應誤報成 low critical，除非平台政策明確如此定義。
- Stale：最近一次值可能仍可顯示，但必須同時標示不可用或逾時，不可讓 consumer 誤認為即時值。

### 24.1.4 Sensor 生命週期與驗收邊界

Sensor 的建立與移除會受 Entity Manager 設定、FRU probe、host power、hot-plug 與 daemon restart 影響。每一類 sensor 都應驗證：冷開機、warm reboot、BMC reboot、host power cycle、裝置插拔、bus recovery、daemon restart、設定更新與 threshold assert/deassert。驗收不只確認「有值」，還應確認錯誤與恢復路徑是否一致。
## 24.2 Sensor 種類總覽

下表用「來源介面、標準單位、常見 daemon、主要風險」整理本章 sensor 類別。服務名稱與 JSON schema 可能因 OpenBMC branch 或 vendor fork 不同，實機仍需以 systemd unit、schema 與 source tree 為準。

| Sensor 類型 | 常見來源 | sysfs 常見節點與單位 | D-Bus 建議單位 | 主要風險 |
|---|---|---|---|---|
| Temperature | I2C hwmon、PECI、PMBus、NVMe/GPU telemetry | `temp*_input`，m°C | °C | 位置偏差、host-off、stale |
| Voltage | ADC、PMBus VR/PSU | `in*_input`，mV | V | 分壓倍率、Page/channel 對錯 |
| Current | PMBus、INA2xx、shunt + ADC | `curr*_input`，mA | A | shunt/Gain、1000 倍換算 |
| Power | PMBus、V×I、telemetry | `power*_input`，µW | W | 1,000,000 倍換算、double counting |
| Fan Tach | SoC tach、fan controller、PSU | `fan*_input`，RPM | RPM | PPR、channel mapping、presence 混用 |
| Fan PWM | SoC PWM、fan controller | `pwm*`，常見 0–255 | raw 或 % | polarity、min duty、failsafe |
| Presence / GPIO | GPIO、CPLD、EEPROM、PMBus status | GPIO / register | Boolean / enum | debounce、active polarity、latch |
| External | Host/BIOS/vendor daemon 寫入 | 無固定 sysfs | 依 sensor type | owner、權限、timeout、stale |
| Virtual | 多個 D-Bus sensor 計算 | 無固定 sysfs | 依公式 | input availability、單位、循環依賴 |

```mermaid
flowchart TB
    S{來源型態}
    S -->|類比量測| ADC[ADC / IIO]
    S -->|標準硬體監控| HWMON[hwmon / PMBus]
    S -->|平台遙測| TELE[PECI / APML / MCTP / PLDM]
    S -->|離散狀態| GPIO[GPIO / CPLD / Presence]
    S -->|軟體輸入| EXT[External / Virtual]

    ADC --> V[Voltage / Current / Thermistor]
    HWMON --> TVCP[Temperature / Voltage / Current / Power / Fan]
    TELE --> DEV[CPU / GPU / NVMe / Accelerator]
    GPIO --> DISC[Present / Intrusion / Fault]
    EXT --> DERIVED[Estimated / Aggregated / Proxy]

    V & TVCP & DEV & DISC & DERIVED --> DBUS[D-Bus 統一物件]
    DBUS --> RF[Redfish]
    DBUS --> IPMI[IPMI / SEL]
    DBUS --> POLICY[Thermal / Fan / Power Policy]
```

選擇整合路徑時，優先順序建議為：已驗證的 chip-specific kernel driver → 標準 hwmon/PMBus/IIO → OpenBMC 現有 sensor daemon → External/Virtual sensor。使用客製 daemon 前，應先確認現有 driver 或 daemon 無法表達需求，並補上 timeout、association、unit、threshold 與 health model。
## 24.3 ADC Sensor

本節整理 OpenBMC 中 ADC voltage sensor 的 porting 流程. ADC 適合用來量測板上類比電壓, 常見包含 P12V、P5V、P3V3、P1V8、CPU Vcore、standby rail、thermistor voltage、hardware strap voltage、battery sense 等.

在 OpenBMC 的 `dbus-sensors` 架構中, ADC sensor 通常由 `adcsensor` daemon 讀取 Linux kernel 透過 IIO / hwmon 匯出的 sysfs 節點, 再依 Entity Manager 提供的設定建立 D-Bus sensor 物件. AST2600 類平台需先在 device tree 啟用 ADC controller, 再用 `iio-hwmon` 將 IIO channel 轉成 hwmon voltage input.

### 24.3.1 基本資料流

```mermaid
flowchart LR
    subgraph BOARD[Board 類比前端]
        A[待測 Rail] --> B[分壓 / RC filter / 保護]
        B --> C[SoC ADC channel]
    end
    subgraph KERNEL[Linux Kernel]
        C --> D[IIO ADC driver]
        D --> E[iio-hwmon bridge]
        E --> F[hwmon inY_input mV]
    end
    subgraph OPENBMC[OpenBMC]
        CFG[Entity Manager<br/>Name / Index / Scale / Threshold] --> G[ADCSensor]
        F --> G
        G --> H[D-Bus voltage object<br/>Value / State / Threshold]
    end
    H --> RF[Redfish]
    H --> IPMI[IPMI SDR / SEL]
    H --> LOG[Event / Logging]
    H --> POLICY[Power / Thermal policy]
    D -. raw + scale debug .-> DBG[IIO debug path]
```

關鍵觀念:
- Kernel 端負責讓 ADC channel 出現在 IIO, 並由 `iio-hwmon` 轉為 `/sys/class/hwmon/hwmonX/inY_input`.
- Userspace 端的 `adcsensor` 會尋找 `name` 為 `iio_hwmon` 的 hwmon 裝置, 並讀取 `in*_input`.
- Entity Manager 只描述"哪個 index 對應哪個 sensor 名稱、倍率、閾值與電源狀態", 不負責設定 ADC pinmux 或啟用 kernel driver.
- Redfish / IPMI 是否看得到 sensor, 通常取決於 D-Bus path、sensor type、association 與上層 mapping 是否符合平台政策.

### 24.3.2 Bring-up 前必填資料

| 欄位 | 說明 | 範例 / 注意事項 |
| --- | --- | --- |
| Rail name | 待測電壓名稱 | `P12V`、`P3V3_AUX`、`CPU_VCORE` |
| ADC controller | 使用哪個 ADC engine | `adc0` / `adc1` |
| ADC channel | SoC 端 channel number | `adc0 channel 0`、`adc1 channel 3` |
| Schematic net | ADC pin 前後 net 名稱 | 需可回查硬體圖 |
| Rtop / Rbottom | 分壓電阻 | 例如 `100 kΩ /20 kΩ` |
| ADC pin 最大電壓 | 分壓後進 ADC 的最高電壓 | 不可超過 Vref / SoC 規格限制 |
| ADC reference | 內部或外部參考電壓 | AST2600 common binding 允許內部 `1.2 V` 或 `2.5 V`, 也可用 `vref-supply` 描述外部參考 |
| ADC 解析度 | SoC / driver 實際解析度 | Upstream AST2600 binding 描述為 10-bit; 若 vendor BSP 有差異, 以該 kernel driver 與 datasheet 為準 |
| sysfs index | `/sys/class/hwmon/.../inY_input` 的 `Y` | 需實機確認, 避免 DTS channel 順序與 sensor 設定不一致 |
| ScaleFactor | 分壓還原倍率 | `(Rtop + Rbottom) /Rbottom` |
| Thresholds | Warning / Critical 上下限 | 通常依 rail nominal ± tolerance 設定 |
| PowerState | sensor 啟用條件 | `Always` / `On` 等, 依專案 schema 支援值確認 |
| Redfish / IPMI policy | 是否需要對外呈現 | 對應 Chassis / Inventory / SDR policy |

### 24.3.3 ScaleFactor 與單位換算

若 `/sys/class/hwmon/hwmonX/inY_input` 回傳的是 ADC pin 上的電壓, hwmon voltage input 慣例單位為 millivolt. 此時 Entity Manager 中的 `ScaleFactor` 應設定為外部分壓還原倍率:

```text
ScaleFactor = (Rtop + Rbottom) / Rbottom
RailVoltage(V) = (inY_input(mV) / 1000) × ScaleFactor
```

範例:

```text
待測 rail: P12V
Rtop = 100 kΩ
Rbottom = 20 kΩ
ScaleFactor = (100 + 20) / 20 = 6.0
in0_input = 2000 mV
P12V = 2000 / 1000 × 6.0 = 12.0 V
```

若看到 sysfs 讀值像 `0~1023` 或 `0~4095` 這類 raw code, 而不是 mV, 需要先確認目前節點是否真的是 hwmon voltage input, 或是否讀錯到 IIO raw 節點. 若專案 kernel / vendor BSP 的橋接行為與 upstream 不同, 才需要把 Vref 與解析度納入換算:

```text
ADC_pin_mV = raw_code × Vref_mV / (2^N - 1)
RailVoltage(V) = (ADC_pin_mV / 1000) × ((Rtop + Rbottom) / Rbottom)
```

排查時建議同步讀取:

```bash
# hwmon voltage input，通常為 millivolt
$ cat /sys/class/hwmon/hwmonX/inY_input

# IIO raw / scale，依 driver expose 狀態可能不同
$ ls /sys/bus/iio/devices/iio:device*/in_voltage*_raw 2>/dev/null
$ ls /sys/bus/iio/devices/iio:device*/in_voltage*_scale 2>/dev/null
```

### 24.3.4 Device Tree 設定

AST2600 upstream binding 中, ADC controller 常見必要屬性包含 `compatible`、`reg`、`clocks`、`resets` 與 `#io-channel-cells = <1>`. 平台 dts 通常只需覆寫 `status`、參考電壓與必要 pinmux, 再建立 `iio-hwmon` consumer.

```dts
/* SoC dtsi 通常已有 adc0 / adc1 節點，平台 dts 覆寫即可 */
&adc0 {
    status = "okay";
    aspeed,int-vref-microvolt = <2500000>;
};

&adc1 {
    status = "okay";
    aspeed,int-vref-microvolt = <2500000>;
};

/* 將指定 IIO channels 匯出為 hwmon voltage input */
iio-hwmon {
    compatible = "iio-hwmon";
    io-channels = <&adc0 0>, <&adc0 1>, <&adc0 2>, <&adc1 0>;
};
```

若平台使用外部 reference regulator, 可改用 `vref-supply`, 並在 regulator 節點描述實際電壓. 若 BSP 支援額外 vendor property, 例如 averaging、clock frequency、battery sensing 或 trim data, 需以該 kernel tree 的 binding 文件與 `dtbs_check` 結果為準; 不要只靠舊專案 DTS 複製.

建議驗證:

```bash
# build host：檢查 binding
$ bitbake -c devshell virtual/kernel
$ make dtbs_check DT_SCHEMA_FILES=Documentation/devicetree/bindings/iio/adc/aspeed,ast2600-adc.yaml
$ make dtbs_check DT_SCHEMA_FILES=Documentation/devicetree/bindings/hwmon/iio-hwmon.yaml

# target：確認 DTB 實際載入內容
$ dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null | less
```

### 24.3.5 Kernel Config 與 image 納入檢查

ADC sensor 需要 ADC driver、IIO core 與 iio-hwmon. 不同 kernel 版本名稱可能略有差異, 常見方向如下:

```text
CONFIG_IIO=y
CONFIG_ASPEED_ADC=y 或對應 platform ADC driver
CONFIG_SENSORS_IIO_HWMON=y 或 m
CONFIG_HWMON=y
```

檢查方式:

```bash
# build output 中確認最後 .config
$ bitbake -e virtual/kernel | grep '^B='
$ grep -E 'CONFIG_(IIO|ASPEED_ADC|SENSORS_IIO_HWMON|HWMON)' \
    tmp/work/*/linux-*/**/build/.config 2>/dev/null

# target 上確認 driver / module / sysfs
$ zcat /proc/config.gz | grep -E 'CONFIG_(IIO|ASPEED_ADC|SENSORS_IIO_HWMON|HWMON)' 2>/dev/null
$ ls /sys/bus/iio/devices/
$ ls /sys/class/hwmon/
```

### 24.3.6 sysfs 節點確認

開機後先找出 `iio_hwmon`:

```bash
$ for h in /sys/class/hwmon/hwmon*; do
    [ -f "$h/name" ] || continue
    if grep -qx "iio_hwmon" "$h/name"; then
        echo "$h"
        ls "$h"/in*_input
    fi
$ done
```

讀取所有 voltage input:

```bash
$ HWMON=/sys/class/hwmon/hwmon2
$ for f in "$HWMON"/in*_input; do
    printf '%s = ' "$(basename "$f")"
    cat "$f"
$ done
```

以三用電表比對:

```text
1. 量測 ADC pin 端電壓，而不是 rail 原始電壓。
2. 比對 inY_input 是否接近 ADC pin mV。
3. 再用 ScaleFactor 換算回 rail voltage。
4. 若 pin 電壓吻合但 D-Bus rail voltage 不吻合，優先檢查 ScaleFactor / Index。
5. 若 pin 電壓與 sysfs 不吻合，優先檢查 Vref、driver、DTS、IIO scale、SoC ADC calibration。
```

一般 bring-up 可接受誤差需依電阻精度、ADC 精度、Vref 精度與 board noise 決定; 若未另訂規格, 可先用 `±1%~±3%` 作為初步判斷, 再由 HW / validation 定義量產門檻.

### 24.3.7 Entity Manager JSON 設定

Entity Manager configuration 通常為 board / chassis / device 物件, sensor 放在 `Exposes` 陣列中. 以下以 baseboard 上的 P12V 為例:

```json
{
    "Name": "Baseboard",
    "Type": "Board",
    "Probe": "TRUE",
    "Exposes": [
        {
            "Name": "P12V",
            "Type": "ADC",
            "Index": 0,
            "ScaleFactor": 6.0,
            "PollRate": 0.5,
            "PowerState": "Always",
            "MinValue": 0.0,
            "MaxValue": 15.0,
            "Thresholds": [
                {
                    "Name": "upper critical",
                    "Direction": "greater than",
                    "Severity": 1,
                    "Value": 13.2
                },
                {
                    "Name": "lower critical",
                    "Direction": "less than",
                    "Severity": 1,
                    "Value": 10.8
                },
                {
                    "Name": "upper non critical",
                    "Direction": "greater than",
                    "Severity": 0,
                    "Value": 12.6
                },
                {
                    "Name": "lower non critical",
                    "Direction": "less than",
                    "Severity": 0,
                    "Value": 11.4
                }
            ]
        }
    ]
}
```

設定重點:
- `Type = "ADC"`: 讓 `adcsensor` daemon 把此筆設定視為 ADC sensor.
- `Index`: 需對應實機可讀到的 `inY_input`, 請以 target sysfs 與 daemon log 交叉確認.
- `ScaleFactor`: 通常只放分壓倍率; 不要重複把 mV → V 或 Vref 轉換放進去, 除非專案 driver 確認回傳 raw code.
- `PollRate`: OpenBMC upstream `adcsensor` 使用秒為單位且預設約 `0.5` 秒; 部分 vendor fork 可能改成 `PollInterval` 或毫秒, 需查 schema / source.
- `PowerState`: 用於控制 sensor 在 standby / host on 時是否建立或更新. 待機 rail 用 `Always` 類設定, host rail 可依平台政策設為只在 host on 時讀取.
- `Probe`: 若 sensor 只存在於特定 SKU, 需加入 FRU、board ID、CPLD register 或其他 inventory 條件, 避免不存在的 sensor 長期呈現 unavailable.

建議在 source tree 內驗證 schema:

```bash
# 找 ADC schema 或 legacy schema 中的 ADC 定義
$ find meta-* openbmc -path '*entity-manager*schema*' -type f 2>/dev/null | sort
$ grep -R '"ADC"' -n meta-* openbmc 2>/dev/null | head -50

# 找平台設定是否進 image
$ find tmp/work -path '*entity-manager*' -type f | grep -E '\.json$' | head
$ find tmp/deploy/images/${MACHINE}/ -type f | grep -E 'manifest|rootfs|tar'
```

### 24.3.8 啟動服務與 D-Bus 驗證

```bash
# 重新讀取 Entity Manager 設定
$ systemctl restart xyz.openbmc_project.EntityManager.service

# 重啟 ADC sensor daemon
$ systemctl restart xyz.openbmc_project.ADCSensor.service

# 查看 log
$ journalctl -u xyz.openbmc_project.EntityManager.service -b --no-pager
$ journalctl -u xyz.openbmc_project.ADCSensor.service -b --no-pager

# 找 ADC sensor 物件
$ busctl tree xyz.openbmc_project.ADCSensor

# 讀值，Value 通常為 Volts / double
$ busctl get-property \
  xyz.openbmc_project.ADCSensor \
  /xyz/openbmc_project/sensors/voltage/P12V \
  xyz.openbmc_project.Sensor.Value \
  Value

# 確認 threshold interface
$ busctl introspect \
  xyz.openbmc_project.ADCSensor \
  /xyz/openbmc_project/sensors/voltage/P12V
```

預期在 D-Bus 上可看到:

```text
xyz.openbmc_project.Sensor.Value
xyz.openbmc_project.Sensor.Threshold.Warning
xyz.openbmc_project.Sensor.Threshold.Critical
xyz.openbmc_project.State.Decorator.Availability
xyz.openbmc_project.State.Decorator.OperationalStatus
```

### 24.3.9 Redfish / IPMI 驗證

Redfish:

```bash
# 依平台 Chassis 名稱調整
$ curl -k -u root:0penBmc https://${BMC}/redfish/v1/Chassis/
$ curl -k -u root:0penBmc https://${BMC}/redfish/v1/Chassis/<chassis>/Sensors
$ curl -k -u root:0penBmc https://${BMC}/redfish/v1/Chassis/<chassis>/Sensors/P12V
```

IPMI:

```bash
$ ipmitool -I lanplus -H ${BMC} -U root -P 0penBmc sensor list | grep -i P12V
$ ipmitool -I lanplus -H ${BMC} -U root -P 0penBmc sdr elist | grep -i P12V
```

若 D-Bus 有 sensor 但 Redfish / IPMI 看不到, 排查方向通常不是 ADC driver, 而是:
- sensor path 或 type 不符合上層預期.
- inventory association 未建立.
- Redfish chassis mapping 未包含該 sensor.
- IPMI SDR policy 未產生該 sensor.
- service 啟動順序或 cache 未更新.

### 24.3.10 閾值與事件驗證

Threshold 建議先用 nominal voltage 與 tolerance 計算. 例如 12 V rail:

```text
Warning：±5%  → 11.4 V / 12.6 V
Critical：±10% → 10.8 V / 13.2 V
```

驗證內容:

```bash
# 讀取 threshold property
$ busctl introspect xyz.openbmc_project.ADCSensor \
  /xyz/openbmc_project/sensors/voltage/P12V | grep -E 'Threshold|Alarm|Value'

# 查看事件 log / journal
$ journalctl -b | grep -Ei 'P12V|threshold|critical|warning'
```

測試方式可依硬體條件選擇:
- 使用可程式電源供應器調整 rail, 需先確認安全範圍.
- 使用 ADC input fixture 模擬分壓後電壓.
- 若 driver / validation framework 支援, 可用測試 hook 注入讀值.

驗收點:
- 超過 warning threshold 時, Warning alarm property 變化.
- 超過 critical threshold 時, Critical alarm property 變化.
- 回到 hysteresis / deassert 條件後 alarm 清除.
- EventLog / SEL / journal 有對應紀錄.
- Redfish sensor 狀態與讀值同步更新.

### 24.3.11 常見問題與排查

| 問題現象 | 建議排查方向 | 建議檢查 |
| --- | --- | --- |
| `/sys/class/hwmon` 找不到 `iio_hwmon` | `iio-hwmon` DTS node 未加入、kernel config 未啟用、driver probe fail | `dmesg \| grep -Ei 'adc\|iio\|hwmon'`、確認 `CONFIG_SENSORS_IIO_HWMON` |
| `iio_hwmon` 存在但沒有 `in*_input` | `io-channels` 指到錯誤 provider / channel、ADC controller 未啟用 | 檢查 DTS phandle、`#io-channel-cells`、`status = "okay"` |
| `inY_input` 永遠為 0 | ADC pin 無電壓、pinmux / board route 錯、待測 rail 未上電 | 用電表量 ADC pin, 確認 host power state |
| D-Bus sensor 沒出現 | Entity Manager JSON 未載入、`Probe` 不符合、`Type` / `Index` 錯誤 | `journalctl -u EntityManager`、`journalctl -u ADCSensor`、`busctl tree` |
| D-Bus value 差一個倍率 | `ScaleFactor` 未填或填錯、把 mV/V 轉換重複計入 | 用 `inY_input /1000 × ScaleFactor` 手算比對 |
| 某些 rail 在待機顯示 unavailable | `PowerState` 設成 host-on-only、rail 本身未上電 | 確認 power policy 與待機 rail 定義 |
| 讀值跳動大 | ADC input noise、分壓阻值過高、濾波不足、取樣率太快 | 用示波器看 ADC pin; 評估 RC filter、driver averaging、PollRate |
| Redfish N/A | D-Bus 有 sensor 但 association / chassis mapping 不完整 | 查 bmcweb log、Redfish sensors endpoint、inventory association |
| threshold 不觸發 | threshold 名稱 /方向 /Severity 錯、讀值未真正跨越門檻 | busctl introspect threshold properties, 人工調整輸入驗證 |
| 更新 DTB 後行為沒變 | 燒錄到錯誤 image slot、U-Boot 載入舊 DTB | 檢查 `/proc/device-tree`、U-Boot env、`tmp/deploy/images` |

### 24.3.12 Porting 驗收 Checklist

硬體與 schematic:
- [ ] ADC channel number 與 SoC pin 對照完成.
- [ ] Rail name、net name、Rtop、Rbottom、ADC pin net 已記錄.
- [ ] ADC pin 最大電壓低於 SoC / Vref 允許範圍.
- [ ] ScaleFactor 已依 schematic 計算並由 HW review.
- [ ] 使用三用電表量測 ADC pin 與 rail, 確認分壓比例合理.

Device Tree / Kernel:
- [ ] `adc0` / `adc1` 或對應 ADC controller `status = "okay"`.
- [ ] `aspeed, int-vref-microvolt` 或 `vref-supply` 與硬體一致.
- [ ] `iio-hwmon` node 已列出需要的 `io-channels`.
- [ ] kernel config 包含 IIO、ADC driver、hwmon、iio-hwmon.
- [ ] target 上可看到 `/sys/class/hwmon/hwmonX/name = iio_hwmon`.
- [ ] target 上可讀到 `/sys/class/hwmon/hwmonX/inY_input`.

Entity Manager / Userspace:
- [ ] JSON 放在正確 layer / recipe, 且進入 rootfs.
- [ ] `Exposes` 裡有 `Type = "ADC"`、`Name`、`Index`、`ScaleFactor`.
- [ ] `PollRate` / `PowerState` 符合平台需求.
- [ ] SKU 差異已用 `Probe` 或平台設定區分.
- [ ] warning / critical thresholds 已依 rail tolerance 設定.

D-Bus / Redfish / IPMI:
- [ ] `xyz.openbmc_project.ADCSensor.service` 啟動無錯誤.
- [ ] D-Bus path 出現 `/xyz/openbmc_project/sensors/voltage/<Name>`.
- [ ] `Value` 單位為 Volts, 且與電表換算結果一致.
- [ ] Warning / Critical threshold interfaces 存在.
- [ ] Redfish sensor endpoint 可看到該 sensor.
- [ ] 若平台需要 IPMI, SDR 與 `ipmitool sensor list` 可看到該 sensor.
- [ ] threshold assert / deassert、EventLog / SEL / journal 已驗證.

### 24.3.13 ADC Sensor 資料表範本

| Rail | ADC controller / channel | sysfs | Rtop | Rbottom | ScaleFactor | Nominal | Warning low/high | Critical low/high | PowerState | D-Bus path |
| --- | --- | --- | ---: | ---: | ---: | ---: | --- | --- | --- | --- |
| P12V | adc0 ch0 | in0_input | 100 kΩ | 20 kΩ | 6.0 | 12.0 V | 11.4 / 12.6 V | 10.8 / 13.2 V | Always | `/xyz/openbmc_project/sensors/voltage/P12V` |
| P3V3_AUX | adc0 ch1 | in1_input | [待填] | [待填] | [待填] | 3.3 V | [待填] | [待填] | Always | [待填] |
| CPU_VCORE | adc1 ch0 | in3_input | [待填] | [待填] | [待填] | [待填] | [待填] | [待填] | On | [待填] |

### 24.3.14 本節參考資料

- OpenBMC `dbus-sensors` README: 說明 sensor daemon 由 Entity Manager 動態設定, ADC sensor 使用 Linux IIO, AST2600 平台需在 device tree 啟用 `adc0` / `adc1` 並用 `iio-hwmon` 匯出.
- OpenBMC `ADCSensorMain.cpp`:`adcsensor` 尋找 `name` 為 `iio_hwmon` 的 hwmon 裝置, 讀取 `in*_input`, 預設 poll rate 約 `0.5` 秒.
- Linux kernel `iio-hwmon.yaml`:`iio-hwmon` binding 的必要屬性為 `compatible = "iio-hwmon"` 與 `io-channels`.
- Linux kernel `aspeed, ast2600-adc.yaml`: AST2600 ADC binding 描述兩個 ADC engine、各 8 個 voltage channels、`#io-channel-cells = <1>`、內部 reference `1200000` / `2500000` microvolt, 以及可用 `vref-supply` 對應外部 reference.

<a id="section-12-4"></a>

## 24.4 Temperature Sensor

### 24.4.1 適用情境

Temperature Sensor 常見於監控系統中各關鍵部位的熱分佈. BMC 端通常不只關心單點溫度, 也會將多個溫度來源交給 Redfish、IPMI SDR、事件紀錄與 fan control policy 使用.

常見監控點如下:

```text
Ambient temperature：進風口 / 出風口
Board temperature：PCB 中央或局部 hotspot
CPU temperature：核心 / 外殼 / PECI 回報值
DIMM temperature：記憶體模組
PSU temperature：電源供應器內部
NVMe temperature：固態硬碟 controller / composite temperature
GPU temperature：圖形處理器 / HBM / board sensor
VR temperature：電壓調節模組
BMC temperature：BMC SoC 自身或鄰近 board sensor
```

在 OpenBMC 中, 溫度感測器通常會被發布成 D-Bus 物件, 常見路徑如下:

```text
/xyz/openbmc_project/sensors/temperature/<sensor_name>
```

常見 D-Bus 介面包含:

```text
xyz.openbmc_project.Sensor.Value
xyz.openbmc_project.Sensor.Threshold.Warning
xyz.openbmc_project.Sensor.Threshold.Critical
xyz.openbmc_project.State.Decorator.Availability
xyz.openbmc_project.State.Decorator.OperationalStatus
xyz.openbmc_project.Association.Definitions
```

這些物件後續可被 Redfish、IPMI、Phosphor PID Control、logging service 或其他平台 daemon 消費.

### 24.4.2 資料路徑(Data Flow)

```mermaid
flowchart LR
    A[I2C temperature IC] --> B[I2C + hwmon driver]
    B --> C[temp*_input m°C]
    CFG[Entity Manager config] --> D[HwmonTempSensor]
    C --> D --> E[D-Bus temperature °C]
    E --> F[Redfish Thermal / Sensors]
    E --> G[IPMI SDR / Event]
    E --> H[PID / Fan policy]
    P[Power / Presence] -. gating .-> D
```

本節以最常見的 I2C 溫度感測晶片(例如 TMP75 / LM75 類)為例說明資料流:

```text
I2C Temperature Chip (TMP75 at Bus 3, Addr 0x48)
    ↓
Linux I2C Driver (i2c-core) + Chip-specific hwmon Driver (lm75.c / tmp75 compatible)
    ↓
hwmon sysfs 介面 (/sys/class/hwmon/hwmonX/temp1_input)
    ↓
HwmonTempSensor daemon 定期讀取 sysfs
    ↓
Entity Manager 提供 JSON 設定 (Name, Type, Bus, Address, Thresholds)
    ↓
D-Bus 發佈至 /xyz/openbmc_project/sensors/temperature/<Name>
    ↓
Redfish Thermal / Sensors、IPMI SDR、Fan PID policy 查詢或消費
```

其他溫度來源的前段資料路徑可能不同, 例如 PECI CPU 溫度、PMBus PSU 溫度、NVMe over MCTP、GPU vendor tool 或 ADC Thermistor. 但在 OpenBMC 上, 常見整合方向仍是收束到標準 D-Bus sensor 介面, 讓上層服務用一致模型讀取.

### 24.4.3 常見來源分類

| 來源類型 | 驅動 / 協定 | OpenBMC 對應 Daemon | 備註 |
| --- | --- | --- | --- |
| I2C 獨立晶片, 例如 TMP75 / LM75 / MAX31725 | Linux hwmon driver | HwmonTempSensor | 最常見; 需 DTS 或 board info 建立 I2C device |
| CPU 內部溫度 | PECI driver / peci-temp | IntelCPUSensor 或平台特定 daemon | 主要見於 x86 平台; 需 host CPU / PCH / PECI 通道可用 |
| PMBus 電源裝置 | pmbus driver | PSUSensor 或 HwmonTempSensor | 讀取 PSU 回報的內部 temperature rail |
| NVMe 固態硬碟 | NVMe-MI / MCTP / vendor command | NVMeSensor 或平台自有服務 | 視平台拓撲決定是否透過 MCTP over I2C / PCIe sideband |
| GPU | vendor-specific ioctl / SMBus / MCTP | GPUUtilSensor / ExternalSensor / vendor daemon | 通常需額外 userspace 工具或 vendor library |
| 外部類比 Thermistor | ADC 讀取後換算 | ADCSensor 搭配 ScaleFactor / Offset / polynomial | 請參閱 11.1 ADC Sensor |
| BMC SoC 內部溫度 | SoC thermal / hwmon driver | HwmonTempSensor 或 SoC-specific daemon | 需確認 kernel driver 是否輸出 temp*_input |

### 24.4.4 Porting 前需確認的硬體資訊與校準參數

從 schematic、board placement、datasheet 與 thermal policy 取得下列資訊:

```text
- Sensor chip 完整型號，例如 TMP75B、LM75A、MAX31725、PCT2075
- 所在 I2C bus number，例如 I2C3
- 若經過 I2C mux，需確認 mux device、channel 與 enable 條件
- 7-bit I2C device address，例如 0x48、0x4A
- 供電 rail 與 power state，例如 BMC standby rail 或 host main rail
- ALERT / interrupt pin 是否接到 BMC GPIO
- 解析度、轉換時間、sample time 或 conversion rate
- 感測點物理位置，例如 front inlet、rear outlet、VR hotspot、PCIe zone
- 有無硬體 offset 或需線性補償，例如貼合、風道位置、膠材造成固定偏差
- Warning / Critical threshold，由 thermal policy 決定
- Redfish / IPMI 顯示名稱與 inventory association
```

範例參數:

```text
Sensor Name: Ambient_Temp
Chip: TMP75B
I2C Bus: 3
Address: 0x48
Resolution: 12-bit (0.0625°C per LSB)
Expected Offset: +0.5°C
Critical High: 85°C
Warning High: 75°C
Warning Low: 0°C
Power Rail: BMC_STBY_3V3
Physical Location: Front inlet, near fan wall
```

建議在 bring-up 記錄中保留硬體量測條件, 例如室溫、風扇狀態、系統功耗與開蓋 / 關蓋狀態, 避免後續校準時無法對齊測試條件.

### 24.4.5 I2C Bus 偵測與地址確認

開機後使用 `i2cdetect` 掃描指定 bus:

```bash
$ i2cdetect -y 3
```

預期輸出範例,0x48 出現 `48`:

```text
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- 48 -- -- -- -- -- -- --
```

若經過 I2C mux, 先確認 mux channel 是否切到正確路徑:

```bash
# 範例：列出 I2C adapter 與 mux channel
$ ls -l /sys/bus/i2c/devices/

$ i2cdetect -l
```

若掃不到地址, 可改用 read-only 方式讀取溫度暫存器或製造商 ID 暫存器. 實際 register 需依 datasheet 確認:

```bash
# TMP75 / LM75 類常見 temperature register 在 0x00；實際格式需看 datasheet
$ i2cget -y 3 0x48 0x00 w
```

注意事項:

- `i2cdetect` 只是輔助確認工具, 不建議在不熟悉裝置行為時對整條 bus 反覆掃描.
- 部分裝置對 SMBus quick command 或特定讀寫型態敏感, 可能造成狀態變化.
- 若 bus 上有 mux、CPLD bridge、hot-swap buffer 或電源 domain gating, 需先確認相關 enable 條件.

### 24.4.6 Device Tree 加入 sensor node

LM75 / TMP75 類溫度晶片通常在 I2C bus node 下建立 child node. 核心必要屬性是 `compatible` 與 `reg`; 若平台有 regulator 或 interrupt, 也應一併描述.

```dts
&i2c3 {
    status = "okay";
    clock-frequency = <100000>;

    temperature-sensor@48 {
        compatible = "ti,tmp75b";
        reg = <0x48>;
        label = "ambient_front";
        vs-supply = <&p3v3_stby>;
        #thermal-sensor-cells = <0>;   /* 若需與 Linux thermal zone 連結 */
    };
};
```

相容性注意事項:

- Linux `lm75` family binding 支援多個 compatible, 例如 `national, lm75`、`st, stlm75`、`maxim, max31725`、`ti, tmp75`、`ti, tmp75b`、`ti, tmp75c` 等.
- 若特定 compatible 在目前 kernel branch 尚未支援, 可先查看 `Documentation/devicetree/bindings/hwmon/lm75.yaml` 與 `drivers/hwmon/lm75.c`.
- 不建議任意使用相近 compatible; 必須確認解析度、暫存器格式、sample time、threshold register 與 interrupt 行為相容.

若溫度晶片掛在 I2C mux 後方, DTS 結構會多一層 mux channel. 實際 bus number 可能由 kernel 動態分配, 因此 Entity Manager 中的 bus number 需以目標機實際 `i2cdetect -l` 與 `/sys/bus/i2c/devices/` 結果確認.

### 24.4.7 Kernel config 與 driver 確認

Kernel 需啟用 I2C、hwmon 與對應感測器 driver. LM75 / TMP75 類通常對應 `CONFIG_SENSORS_LM75`.

```bash
# 在 build tree 檢查最終 kernel config
$ bitbake -e virtual/kernel | grep '^B='

$ grep -E 'CONFIG_HWMON|CONFIG_I2C|CONFIG_SENSORS_LM75' \
  tmp/work/*/linux-*/ */build/.config 2>/dev/null
```

實機上可檢查 driver 是否 probe 成功:

```bash
$ dmesg | grep -Ei 'lm75|tmp75|hwmon|i2c'

# 若 driver 是 module
$ lsmod | grep -E 'lm75|hwmon'

# 查看 I2C device 與 driver bind 狀態
$ find /sys/bus/i2c/devices -maxdepth 2 -type l -name driver -print -exec readlink -f {} \;
```

若 driver 未 probe, 可從下列方向同步資訊:

- DTS 是否進到實際 boot 的 DTB.
- `compatible` 是否被目前 kernel driver 支援.
- bus / mux path 是否正確.
- power rail 是否已開.
- I2C address 是否與硬體 strap 一致.

### 24.4.8 確認 hwmon sysfs 節點與數值單位

開機後尋找對應 hwmon:

```bash
# 找出所有 hwmon 並顯示 name
$ for i in /sys/class/hwmon/hwmon*; do
    echo "$i: $(cat $i/name 2>/dev/null)"
$ done

# 假設找到 hwmon4，檢查溫度節點
$ ls /sys/class/hwmon/hwmon4/temp*_input

# 讀取溫度，Linux hwmon temperature 通常以 millidegree Celsius 呈現
$ cat /sys/class/hwmon/hwmon4/temp1_input
```

單位判讀:

```text
temp1_input: 30000 → 30.000°C
temp1_crit: 85000 → 85.000°C
temp1_max: 75000 → 75.000°C
```

常見 sysfs 欄位:

| 欄位 | 常見意義 | 單位 |
| --- | --- | --- |
| temp1_input | 目前溫度讀值 | millidegree Celsius |
| temp1_max | 上限 threshold, 若 driver 支援 | millidegree Celsius |
| temp1_crit | critical threshold, 若 driver 支援 | millidegree Celsius |
| temp1_alarm | 硬體 alarm 狀態, 若 driver 支援 | 0 / 1 |
| name | hwmon 裝置名稱 | 字串 |

注意: Linux hwmon 文件中提到, driver 只是呈現硬體值與 alarm 狀態; 不同 board 的標籤與補償通常仍需由 userspace 或設定檔處理.

### 24.4.9 Entity Manager 配置

Entity Manager 的設定通常位於下列路徑之一, 實際位置需依平台 layer 與 recipe 設計確認:

```text
/usr/share/entity-manager/configurations/
meta-<platform>/recipes-phosphor/configuration/<project>/
meta-<platform>/recipes-phosphor/configuration/entity-manager/
```

OpenBMC `dbus-sensors` 通常透過 Entity Manager 取得 sensor device 設定; sensor 物件以 configuration 中的 `Exposes` records 描述, 且 `Name` 與 `Type` 是基本必要欄位.

常見 JSON 結構如下. 不同 OpenBMC branch 的 schema 可能略有差異, 請以目前 image 內 `/usr/share/entity-manager/schemas/` 與 `sensor-info.json` 為準.

```json
{
    "Name": "Baseboard Sensors",
    "Probe": "xyz.openbmc_project.FruDevice({'BOARD_PRODUCT_NAME': 'MyPlatform'})",
    "Type": "Board",
    "Exposes": [
        {
            "Name": "Ambient_Temp",
            "Type": "TMP75",
            "Bus": 3,
            "Address": "0x48",
            "PollRate": 0.5,
            "ScaleFactor": 1.0,
            "Offset": 0.5,
            "MaxValue": 100.0,
            "MinValue": -10.0,
            "PowerState": "AlwaysOn",
            "Thresholds": [
                {
                    "Name": "upper critical",
                    "Direction": "greater than",
                    "Severity": 1,
                    "Value": 85.0
                },
                {
                    "Name": "upper non critical",
                    "Direction": "greater than",
                    "Severity": 0,
                    "Value": 75.0
                },
                {
                    "Name": "lower non critical",
                    "Direction": "less than",
                    "Severity": 0,
                    "Value": 0.0
                }
            ]
        }
    ]
}
```

重點欄位說明:

| 欄位 | 說明 | 檢查方式 |
| --- | --- | --- |
| Name | D-Bus sensor 名稱的一部分; 通常會出現在 /xyz/openbmc_project/sensors/temperature/<Name> | busctl tree / Redfish sensor list |
| Type | 需符合 dbus-sensors 與 Entity Manager schema 支援的 sensor type | 查 sensor-info. json、schema、daemon log |
| Bus / Address | 對應 I2C bus 與 7-bit address | i2cdetect -l、/sys/bus/i2c/devices |
| PollRate / PollInterval | 輪詢頻率或間隔; 名稱依 branch 而可能不同 | 查 schema 與 daemon source |
| ScaleFactor / Offset | 線性倍率與固定補償; 適合處理安裝位置或類比路徑的固定偏差 | 與標準溫度計比對 |
| PowerState | AlwaysOn 或 On; 用來控制 host power state 下是否讀取 | 待機 / 上電狀態測試 |
| Thresholds | Warning / Critical high / low threshold | D-Bus introspect 與 threshold 觸發測試 |

若平台使用舊版設定格式, 可能不使用 `Exposes` wrapper, 或欄位名稱不同. 移植時請以 target image 內實際 schema 為準, 不要只複製其他平台 JSON.

### 24.4.10 啟動服務與 D-Bus 驗證

重啟 Entity Manager 與 HwmonTempSensor:

```bash
$ systemctl restart xyz.openbmc_project.EntityManager.service
$ systemctl restart xyz.openbmc_project.HwmonTempSensor.service
```

監看日誌:

```bash
$ journalctl -u xyz.openbmc_project.EntityManager.service -b --no-pager | tail -100
$ journalctl -u xyz.openbmc_project.HwmonTempSensor.service -b --no-pager | tail -100

# 或持續追蹤
$ journalctl -u xyz.openbmc_project.HwmonTempSensor.service -f
```

確認 D-Bus 物件樹:

```bash
$ busctl tree xyz.openbmc_project.HwmonTempSensor
```

讀取溫度輸出. D-Bus `Value` 的單位通常為 °C, 型態為 double:

```bash
$ busctl get-property \
  xyz.openbmc_project.HwmonTempSensor \
  /xyz/openbmc_project/sensors/temperature/Ambient_Temp \
  xyz.openbmc_project.Sensor.Value \
  Value
```

確認 threshold 介面是否存在:

```bash
$ busctl introspect xyz.openbmc_project.HwmonTempSensor \
  /xyz/openbmc_project/sensors/temperature/Ambient_Temp
```

預期可看到下列介面與屬性:

```text
xyz.openbmc_project.Sensor.Value
  Value
  Unit

xyz.openbmc_project.Sensor.Threshold.Warning
  WarningHigh
  WarningLow
  WarningAlarmHigh
  WarningAlarmLow

xyz.openbmc_project.Sensor.Threshold.Critical
  CriticalHigh
  CriticalLow
  CriticalAlarmHigh
  CriticalAlarmLow
```

若 D-Bus sensor 未出現, 先同步三個層級的資訊:

```bash
# 1. kernel / hwmon 是否有輸出
$ find /sys/class/hwmon -name 'temp*_input' -print

# 2. Entity Manager 是否載入設定
$ busctl tree xyz.openbmc_project.EntityManager
$ journalctl -u xyz.openbmc_project.EntityManager.service -b --no-pager | grep -i Ambient

# 3. HwmonTempSensor 是否建立物件
$ journalctl -u xyz.openbmc_project.HwmonTempSensor.service -b --no-pager | grep -Ei 'Ambient|TMP75|0x48|error|fail'
```

### 24.4.11 Redfish / IPMI 整合驗證

Redfish 常見查詢入口會依 OpenBMC 版本與平台設定而不同. 常見路徑包含:

```text
/redfish/v1/Chassis/<ChassisId>/Thermal
/redfish/v1/Chassis/<ChassisId>/Sensors
/redfish/v1/TelemetryService/MetricReports
```

查詢範例:

```bash
$ curl -k -u root:<password> \
  https://<bmc-ip>/redfish/v1/Chassis/chassis/Sensors

$ curl -k -u root:<password> \
  https://<bmc-ip>/redfish/v1/Chassis/chassis/Thermal
```

需確認項目:

- Redfish sensor 名稱是否符合平台命名規格.
- Reading / ReadingUnits 是否合理, 溫度單位通常為 Celsius.
- Thresholds 是否對應 Entity Manager 設定.
- Status. State / Status. Health 是否會隨 availability / threshold 狀態變化.
- 若需 IPMI SDR, 確認 SDR 中 sensor type、entity id、sensor number 與 threshold mapping 是否符合需求.

### 24.4.12 校準與 Thermal Policy 對齊

溫度 sensor bring-up 不應只停在 `cat temp1_input` 有值, 還需與 thermal policy 對齊. 建議建立下列測試資料:

| 測試條件 | 觀察項目 | 驗收方向 |
| --- | --- | --- |
| 室溫 idle | sensor 讀值、標準溫度計、風扇轉速 | ambient sensor 通常應與環境溫度接近 |
| host full load | CPU / DIMM / VR / outlet 溫度曲線 | 曲線需符合 airflow 與功耗預期 |
| fan speed step | 溫度下降時間常數 | fan policy 應可讓溫度回到目標區間 |
| 局部加熱 | D-Bus Value、WarningAlarmHigh、CriticalAlarmHigh | threshold 旗標與事件紀錄需同步更新 |
| S5 / standby | PowerState 行為與 I2C error log | 不應在未供電裝置上持續讀取造成錯誤 |

校準建議:

- 先確認 raw sysfs 值是否合理, 再調整 Entity Manager 的 `Offset` / `ScaleFactor`.
- 若偏差與溫度相關, 單純固定 offset 可能不足, 需重新檢查硬體安裝、感測點位置或換算公式.
- 若 thermal policy 使用 PID control, 需要同步確認 sensor 名稱是否被 policy 指到正確物件.
- threshold 建議由熱設計與可靠度規格定義, 不要只依 bring-up 當下測得溫度推估.

### 24.4.13 進階除錯與常見陷阱

| 問題現象 | 建議排查方向 | 排查 / 處理方式 |
| --- | --- | --- |
| i2cdetect 掃不到地址 | I2C mux 未切到正確 channel; 晶片未供電; 地址 strap 與預期不同; bus number 認知不一致 | 檢查 DTS mux node、/sys/bus/i2c/devices、VCC / GND、嘗試相鄰 address, 例如 0x49 / 0x4A |
| hwmon 有節點但讀值固定或異常 | driver 與硬體型號不完全相容; 暫存器格式不同; 讀取到 cached value | 確認 compatible、看 datasheet、觀察 dmesg; 注意 LM75 driver 可能有 cache 週期 |
| 讀值為極端負值, 例如 -128°C 附近 | 通訊失敗、NACK、暫存器讀取格式不正確、driver 回傳錯誤值 | 確認 I2C waveform、pull-up、clock-frequency; 必要時降速到 100 kHz 或 40 kHz 測試 |
| 讀值與標準溫度計偏差大於 2°C | 硬體熱耦合不良; sensor 放置位置與量測點不同; 解析度或 sample time 設定不同 | 比對多個溫度點; 先確認 raw 值, 再評估 JSON Offset 或 ScaleFactor |
| D-Bus sensor 未出現 | Entity Manager JSON 未載入; Probe 條件不成立; Type 不符合 schema; Bus / Address 對不上 hwmon device | 查 Entity Manager log、HwmonTempSensor log、sensor-info. json、schema 與 busctl tree |
| Redfish 出現但數值不更新 | daemon 輪詢未更新; Redfish cache; sensor availability false | 調整 PollRate 測試; busctl monitor 觀察 PropertiesChanged; 查 bmcweb log |
| 系統 log 出現頻繁 I2C 錯誤 | 輪詢過快; bus 上裝置過多; clock 太高; host / BMC 共用 bus 存在 arbitration 問題 | 調低 PollRate 或調高 PollInterval; 降低 clock-frequency; 檢查 bus loading |
| PECI CPU 溫度無法讀取 | peci driver 未載入; PECI channel 初始化失敗; host CPU 不回應 | 檢查 DTS peci node、dmesg、host power state 與平台 PECI routing |
| threshold 不觸發事件 | Thresholds 未進 D-Bus; logging / event policy 未連接; 測試溫度未跨越門檻與 hysteresis 條件 | busctl introspect 查看 Warning/Critical 介面; busctl monitor; 查 phosphor-logging journal |

### 24.4.14 Temperature Sensor 資料表範本

| 欄位 | 填寫值 | 備註 |
| --- | --- | --- |
| Sensor Name | [待填] | 例如 Ambient_Temp、CPU0_Temp、DIMM_A0_Temp |
| Physical Location | [待填] | 進風口、出風口、VR、PCIe zone 等 |
| Source Type | [待填] | I2C hwmon、PECI、PMBus、NVMe、GPU、ADC Thermistor |
| Chip / Device | [待填] | TMP75B、LM75A、MAX31725、CPU PECI 等 |
| I2C Bus / Mux Channel | [待填] | 若非 I2C, 填入協定或來源 |
| 7-bit Address | [待填] | 例如 0x48 |
| DTS Node | [待填] | 檔名與 node path |
| Kernel Driver | [待填] | 例如 lm75、pmbus、peci-temp |
| hwmon Path | [待填] | /sys/class/hwmon/hwmonX/temp1_input |
| Entity Manager Type | [待填] | 需符合 schema |
| D-Bus Path | [待填] | /xyz/openbmc_project/sensors/temperature/... |
| Redfish Path | [待填] | /redfish/v1/Chassis/.../Sensors/... |
| ScaleFactor / Offset | [待填] | 預設 1.0 / 0.0; 需有測試依據 |
| Warning High / Low | [待填] | °C |
| Critical High / Low | [待填] | °C |
| PowerState | [待填] | AlwaysOn / On |
| Fan Policy Consumer | [待填] | PID zone 或 thermal policy 名稱 |
| Validation Owner | [待填] | BMC / Thermal / HW / System |

### 24.4.15 Temperature Sensor 完整 Checklist(Porting 驗收)

硬體設計階段:
- [ ] Sensor chip 型號與 datasheet 確認
- [ ] I2C bus number、mux channel 與 device address 確認
- [ ] 地址 strap、ALERT pin、供電 rail 與 power state 確認
- [ ] 溫度感測點物理位置定義(進風 / 出風 / 元件表面 / hotspot)
- [ ] 預期溫度範圍與 thermal policy thresholds 定義
- [ ] 若需校準, 已定義標準溫度計、測試治具與環境條件

Device Tree / Kernel:
- [ ] 對應 I2C bus node status = "okay"
- [ ] 若經過 I2C mux, mux node 與 channel 設定正確
- [ ] Sensor child node 加入, compatible / reg 正確
- [ ] regulator / interrupt / label 視平台需求加入
- [ ] Kernel driver 已啟用, 例如 CONFIG_SENSORS_LM75
- [ ] 開機後 dmesg 無 probe 失敗錯誤
- [ ] /sys/class/hwmon/hwmonX/temp1_input 存在且讀值合理
- [ ] 以標準溫度計比對讀值, 誤差在規格書或 thermal 團隊定義範圍內

Entity Manager / Userspace:
- [ ] JSON 設定檔加入正確路徑或已編進 image
- [ ] Probe 條件符合目前 board FRU / inventory 資料
- [ ] Type 名稱與 sensor-info. json / schema 定義一致
- [ ] Bus / Address 數值與實機一致
- [ ] ScaleFactor 與 Offset 已視需要調整, 且有測試紀錄
- [ ] PollRate / PollInterval 符合散熱回應時間與 bus loading 需求
- [ ] PowerState 設定符合系統使用情境
- [ ] Thresholds(Critical / Warning High / Low)設定完成

D-Bus / 系統整合:
- [ ] xyz. openbmc_project. HwmonTempSensor. service 啟動無錯誤
- [ ] busctl tree 出現 /xyz/openbmc_project/sensors/temperature/<Name>
- [ ] busctl get-property 可讀取 Value, 單位為 °C
- [ ] WarningHigh / CriticalHigh 等屬性寫入 D-Bus 且數值正確
- [ ] busctl monitor 可看到 PropertiesChanged 訊號
- [ ] Redfish Thermal 或 Sensors 可查到該溫度 sensor
- [ ] IPMI SDR 若有需求, sensor type / entity / threshold mapping 正確
- [ ] 加熱 / 冷卻測試時, D-Bus 與 Redfish 數值同步變化
- [ ] 觸發過溫時, CriticalAlarmHigh 或 WarningAlarmHigh 旗標正確變化
- [ ] threshold 觸發時, phosphor-logging / SEL / Journal entry 符合需求
- [ ] 風扇轉速依據溫度變化有正確加速 / 減速
- [ ] S5 / standby / host power cycle 下, 不產生不必要 I2C error log

### 24.4.16 本節參考資料

- OpenBMC dbus-sensors README: https://github.com/openbmc/dbus-sensors
- OpenBMC dbus-sensors HwmonTempSensor source: https://github.com/openbmc/dbus-sensors/tree/master/src/hwmon-temp
- Linux hwmon sysfs interface: https://docs.kernel.org/hwmon/sysfs-interface.html
- Linux LM75 Device Tree binding: https://github.com/torvalds/linux/blob/master/Documentation/devicetree/bindings/hwmon/lm75.yaml
- Linux LM75 hwmon driver documentation: https://www.kernel.org/doc/html/latest/hwmon/lm75.html

<a id="section-12-5"></a>

## 24.5 Voltage Sensor

本節整理 OpenBMC 中 Voltage Sensor 的適用情境、資料路徑與 PMBus / hwmon 類電壓感測器 porting 流程. ADC 類比電壓的細節已在 `11.1 ADC Sensor` 說明; 本節重點放在 VR controller、PSU、PMBus device 與獨立 hwmon 晶片等數位讀取來源.

### 24.5.1 適用情境

Voltage Sensor 用於監控系統中各級電源軌的電壓準位, 目的在於確認供電穩定度、比對硬體規格, 以及讓電源異常能被 D-Bus、Redfish、IPMI SDR、SEL / Journal 或平台 policy 消費. 常見監控對象包含:

```text
P12V：主電源輸入
P5V：週邊介面
P3V3：I/O 與晶片組
P1V8：低壓介面
CPU_VCCIN：CPU 核心輸入電壓
CPU_VDDCR：CPU 計算核心電壓
DIMM rail：記憶體供電
VR output voltage：電壓調節模組輸出
PSU input voltage：電源供應器輸入端
PSU output voltage：電源供應器輸出端
```

在 OpenBMC 中, Voltage Sensor 通常以 D-Bus sensor object 呈現, 常見路徑如下:

```text
/xyz/openbmc_project/sensors/voltage/<sensor_name>
```

`dbus-sensors` 的設計是由不同 sensor daemon 從 `hwmon`、D-Bus 或直接 driver 讀值, 並建立 `xyz.openbmc_project.Sensor.*` 介面; 上層常見消費者包含 Redfish、IPMI SDR、logging 與控制策略.

### 24.5.2 資料路徑

```mermaid
flowchart TB
    A{Voltage source} -->|Analog rail| B[Divider + ADC]
    A -->|Digital telemetry| C[PMBus VR / PSU / hwmon IC]
    B --> D[IIO / iio-hwmon in*_input]
    C --> E[PMBus / chip driver in*_input]
    D --> F[ADCSensor]
    E --> G[PSUSensor / hwmon sensor daemon]
    F --> H[D-Bus voltage V]
    G --> H
    H --> I[Redfish / IPMI / Event / Policy]
```

Voltage Sensor 依硬體來源可分為兩條主要資料路徑.

路徑 A: ADC 類比電壓, 細節請回查 `11.1 ADC Sensor`.

```text
類比電壓
    ↓
分壓電阻 / 濾波電路
    ↓
AST2600 ADC
    ↓
iio-hwmon
    ↓
sysfs：/sys/class/hwmon/hwmonX/inY_input
    ↓
adcsensor daemon
    ↓
D-Bus：/xyz/openbmc_project/sensors/voltage/<Name>
```

路徑 B: PMBus / VR 控制器 / PSU / 獨立 hwmon 晶片.

```text
VR Controller / PSU / PMBus device
    ↓
I2C / SMBus 匯流排
    ↓
Linux kernel I2C driver
    ↓
PMBus kernel driver 或晶片專用 hwmon driver
    ↓
hwmon sysfs：/sys/class/hwmon/hwmonX/in*_input
    ↓
OpenBMC sensor daemon
    ↓
D-Bus：/xyz/openbmc_project/sensors/voltage/<Name>
    ↓
Redfish / WebUI / IPMI / logging / policy
```

實務上, 服務名稱會依平台採用的 sensor stack 而不同. 常見可能是 `psusensor`、`phosphor-hwmon`、平台自有 hwmon voltage daemon, 或專案命名的 `HwmonVoltageSensor` 類服務. Porting 時應以 image 內實際 systemd units、Entity Manager schema 與 `sensor-info.json` 為準.

### 24.5.3 常見來源分類與特性

| 來源類型 | 讀取介面 | OpenBMC 對應服務 | 關鍵注意事項 |
| :--- | :--- | :--- | :--- |
| ADC 分壓電阻 | IIO / hwmon | `adcsensor` | 需計算 `ScaleFactor`; 細節回查 ADC Sensor 章節 |
| PMBus VR, 例如 CPU / DIMM VR | PMBus driver | `psusensor` / hwmon voltage daemon / 平台服務 | 需確認 I2C bus、address、PMBus Page 與 rail 對應 |
| PMBus PSU | PMBus driver | `psusensor` 或 PSU sensor service | 通常同時有 input voltage、output voltage、power、current、fan 等資料 |
| 獨立 hwmon 晶片, 例如 LTC2990 / INA 類 device | I2C hwmon driver | hwmon 類 sensor service | 需在 Device Tree 或 board info 中啟用, 並確認 label 對應 |
| 主機板內建 VR / CPU telemetry | 專用 driver 或 PECI / SMBus 類路徑 | 專用 daemon | 依 CPU / vendor driver 與平台策略確認 |

Linux hwmon sysfs 對 voltage 使用 `in<number>_input` 命名; 多數文件慣例中 voltage 編號常從 `in0` 開始, 而 `*_input` 為單一固定小數格式數值. 實際 rail 標籤仍需靠 `in*_label`、schematic、driver 文件或實機量測比對確認.

### 24.5.4 Porting 步驟: PMBus / hwmon Voltage Sensor

#### Step 1: 確認硬體資訊與 PMBus Page

從 schematic、BOM、VR datasheet、PSU MFR 文件與 I2C bus map 取得下列資料:

```text
- VR / PSU / hwmon 晶片型號，例如 MP2971、ISL68137、TPS53679、LTC2990
- 所在 I2C bus number
- 7-bit I2C device address，例如 0x40、0x4C、0x4E
- VR 支援的輸出電壓軌數量，也就是 Page 數量
- 每個 Page 對應的 rail 名稱，例如 Page 0 = Vcore，Page 1 = VCCGT
- 電壓讀取命令，PMBus 常見 READ_VOUT = 0x8B
- 是否存在外部回授分壓或 sense network
- nominal voltage、warning threshold、critical threshold
```

範例參數:

```text
Sensor Name: CPU0_VCCIN
Chip: MP2971
I2C Bus: 8
Address: 0x40
Page: 0
Expected Voltage: 1.80 V
Warning High: 1.89 V
Critical High: 1.98 V
Warning Low: 1.71 V
Critical Low: 1.62 V
PowerState: On
```

#### Step 2: I2C 位址與基本通訊驗證

先確認 I2C bus 上能看到目標位址:

```bash
$ i2cdetect -y 8
```

預期在 `0x40` 位置看到裝置回應. 若該 device 對 SMBus quick command 敏感, 請優先查 datasheet、平台 bring-up guideline 或改用較保守的讀取方式.

若需手動讀取 PMBus `READ_VOUT`, 可先設定 Page, 再讀取 command `0x8B`. 不同 IC 對 byte / word order 與 VOUT_MODE 解碼可能不同, 下列指令主要用於確認通訊與大致資料變化:

```bash
# 設定 PMBus Page 0
$ i2cset -y 8 0x40 0x00 0x00

# 讀取 READ_VOUT，command 0x8B
$ i2cget -y 8 0x40 0x8B w
```

若能取得 `0xXXXX` 類結果, 代表 I2C transaction 有回應. 是否為正確電壓值, 仍需結合 `VOUT_MODE`、driver 解碼、sysfs 讀值與電表量測比對.

#### Step 3: Device Tree 加入 PMBus / VR 節點

以下以 MP2971 在 `i2c8`、address `0x40` 為例. 實際 compatible string 需以目前 kernel binding、driver 支援清單與 vendor BSP 為準.

```dts
&i2c8 {
    status = "okay";
    clock-frequency = <100000>;

    vr-controller@40 {
        compatible = "mps,mp2971";
        reg = <0x40>;
        label = "cpu_vr";
    };
};
```

常見 compatible / driver 方向:

```text
"mps,mp2971"
"mps,mp2973"
"isl,isl68137"
"ti,tps53679"
"pmbus"：通用 fallback，需確認 generic PMBus driver 能安全辨識該 device 的能力
```

Kernel 設定需包含 PMBus core、對應 PMBus device driver 或通用 PMBus driver. 例如:

```text
CONFIG_PMBUS
CONFIG_SENSORS_PMBUS
CONFIG_SENSORS_MP2975 / CONFIG_SENSORS_ISL68137 / CONFIG_SENSORS_TPS53679 類似選項
```

實際 config 名稱會隨 kernel 版本與 vendor BSP 有差異, 請用 `grep -R "config SENSORS_" drivers/hwmon/pmbus/` 或 kernel `.config` 驗證.

#### Step 4: 確認 driver probe 與 hwmon sysfs 節點

開機後先看 driver 是否 probe 成功:

```bash
$ dmesg | grep -Ei 'pmbus|mp297|isl681|tps536|hwmon'
```

列出目前 hwmon 裝置:

```bash
$ for i in /sys/class/hwmon/hwmon*; do
    echo "$i: $(cat "$i/name" 2>/dev/null)"
$ done
```

假設 MP2971 對應到 `hwmon3`, 可檢查 voltage input:

```bash
$ ls /sys/class/hwmon/hwmon3/in*_input
$ cat /sys/class/hwmon/hwmon3/in0_input
$ cat /sys/class/hwmon/hwmon3/in0_label 2>/dev/null || true
```

hwmon voltage input 通常以 millivolt 為單位, 例如:

```text
1800 = 1.800 V
900  = 0.900 V
12000 = 12.000 V
```

多 Page VR 的 `in*_input` 對應需以 driver 與實機結果確認. 常見但不可直接假設的映射如下:

```text
in0_input → Page 0，例如 VCCIN
in1_input → Page 1，例如 VCCGT
in2_input → Page 2，例如 VCCSA
```

若 driver 提供 `in*_label`, 請優先用 label 與 schematic rail name 建立對照. 若沒有 label, 建議透過 Page 切換、負載變化、VR telemetry tool、oscilloscope / DMM 量測共同確認.

#### Step 5: Entity Manager / Sensor JSON 配置

設定檔常見位置包含:

```text
/usr/share/entity-manager/configurations/
meta-<platform>/recipes-phosphor/configuration/entity-manager/*.json
平台自有 sensor configuration recipe
```

以下為概念範例. 實際 schema 需依專案採用的 `entity-manager` 與 sensor daemon 支援欄位調整. 關鍵是 `Name`、`Type`、`Bus`、`Address`、`Page`、`ScaleFactor`、`PowerState` 與 thresholds 要對齊.

```json
{
    "Name": "CPU0_VCCIN",
    "Type": "MP2971",
    "Bus": 8,
    "Address": "0x40",
    "Page": 0,
    "PollInterval": 500,
    "ScaleFactor": 1.0,
    "Offset": 0.0,
    "MaxValue": 2.5,
    "MinValue": 0.0,
    "PowerState": "On",
    "Thresholds": [
        {
            "Name": "upper critical",
            "Direction": "greater than",
            "Severity": 1,
            "Value": 1.98
        },
        {
            "Name": "lower critical",
            "Direction": "less than",
            "Severity": 1,
            "Value": 1.62
        },
        {
            "Name": "upper non critical",
            "Direction": "greater than",
            "Severity": 0,
            "Value": 1.89
        },
        {
            "Name": "lower non critical",
            "Direction": "less than",
            "Severity": 0,
            "Value": 1.71
        }
    ]
}
```

注意事項:

- `Type` 必須與 sensor daemon / Entity Manager 目前支援的類型一致. 若 JSON 被解析但 D-Bus sensor 未產生, 優先查 Entity Manager log 與 sensor daemon log.
- `Page` 對多輸出 VR 十分關鍵. 若 Page 錯誤, sensor 可能讀到另一條 rail, 數值仍看似合理但對應錯誤.
- 若 hwmon driver 已輸出正確 mV,`ScaleFactor` 常設為 `1.0`. 若 sense point 有外部比例網路, 才依硬體比例補償.
- `PowerState` 建議與 rail 供電條件一致. CPU / DIMM VR 通常只在 host power on 後有效; PSU input、standby rail 則可能屬於 AlwaysOn.
- threshold 建議由 rail nominal 與 tolerance 推導, 並與硬體保護門檻、VR fault limit、系統降載策略同步.

#### Step 6: 啟動服務與 D-Bus 驗證

依平台實際服務名稱重啟對應 sensor service. 若專案使用 `HwmonVoltageSensor` 類服務, 可採用:

```bash
$ systemctl restart xyz.openbmc_project.HwmonVoltageSensor.service
$ journalctl -u xyz.openbmc_project.HwmonVoltageSensor.service -b --no-pager | tail -100
```

若平台使用 `psusensor` 或其他服務, 請先列出相關 units:

```bash
$ systemctl list-units '*sensor*' --no-pager
$ systemctl list-units '*psu*' --no-pager
```

確認 D-Bus 物件:

```bash
$ busctl tree xyz.openbmc_project.HwmonVoltageSensor

$ busctl get-property \
  xyz.openbmc_project.HwmonVoltageSensor \
  /xyz/openbmc_project/sensors/voltage/CPU0_VCCIN \
  xyz.openbmc_project.Sensor.Value \
  Value

$ busctl introspect \
  xyz.openbmc_project.HwmonVoltageSensor \
  /xyz/openbmc_project/sensors/voltage/CPU0_VCCIN
```

若服務名稱不同, 可先用以下方式找 sensor owner:

```bash
$ busctl tree | grep -i sensors
$ busctl get-property \
  $(busctl tree | grep -i HwmonVoltageSensor | head -1) \
  /xyz/openbmc_project/sensors/voltage/CPU0_VCCIN \
  xyz.openbmc_project.Sensor.Value \
  Value
```

實務上更穩定的方式是用 mapper 查 object owner:

```bash
$ busctl call xyz.openbmc_project.ObjectMapper \
  /xyz/openbmc_project/object_mapper \
  xyz.openbmc_project.ObjectMapper GetObject \
  sas \
  /xyz/openbmc_project/sensors/voltage/CPU0_VCCIN \
  0
```

### 24.5.5 進階除錯與常見陷阱

| 問題現象 | 建議排查方向 | 排查 / 處理方式 |
| :--- | :--- | :--- |
| `i2cdetect` 可看到 address, 但無 `in*_input` | kernel driver 未 bind; compatible 不匹配; PMBus driver 未啟用 | 查 `dmesg`、DTS compatible、kernel `.config`; 必要時用 `new_device` 暫時驗證 driver |
| 讀值為 0、固定最大值或 `65535` | VR 未上電、PMBus timeout、device standby、讀到不支援 command | 確認 host power state、VR enable、PMBus status; 降低 I2C clock 後比對 |
| VCCIN 應為 1.8 V 但顯示 0.9 V | Page / channel 對應錯誤; 讀到其他 rail; driver 解碼或 scaling 不符 | 查 `in*_label`、切 Page 讀 `READ_VOUT`、用 DMM 量測 rail, 建立對照表 |
| 多顆 VR address 相同 | I2C address strap 或 mux channel 判讀錯誤 | 查 schematic ADDR pin 與 mux topology; 確認 bus number 是 mux 後 channel |
| D-Bus sensor 未出現 | JSON `Type`、`Bus`、`Address`、`Page` 或 schema 欄位不符合 daemon 預期 | 查 Entity Manager log、sensor daemon log、`busctl tree xyz.openbmc_project.EntityManager` |
| Redfish 看得到 sensor 但數值不變 | daemon poll 未更新、sysfs 本身不變、PowerState gating、讀值 cache | 直接 `cat in*_input`, 再看 D-Bus `Value` 與 service journal |
| 電壓與電表有固定比例誤差 | 外部分壓、sense point 不同、driver scaling 與平台設計不一致 | 計算硬體比例並調整 `ScaleFactor` / platform config; 保留量測紀錄 |
| log 出現 hwmon / pmbus symbol 或 module 相關錯誤 | PMBus core 或晶片 driver 未進 image / module 未載入 | 確認 kernel config、module autoload、`modprobe pmbus` 與 image package |
| threshold 觸發但沒有事件紀錄 | Threshold interface 存在但 logging policy 未接, 或 alarm flag 未被消費 | 查 D-Bus threshold properties、phosphor-logging、event service 與 Redfish event 設定 |

### 24.5.6 Voltage Sensor Porting 驗收 Checklist

硬體設計階段:

- [ ] Voltage source 確認: ADC / PMBus VR / PSU / 獨立 hwmon 晶片
- [ ] Rail 名稱與 nominal voltage 確認
- [ ] I2C bus / address / mux channel 確認
- [ ] VR Page 數量與各 Page 對應 rail 確認
- [ ] 外部 sense / feedback 分壓是否存在, 若有則計算 ScaleFactor
- [ ] Warning / Critical threshold 依 rail tolerance 與保護策略設定
- [ ] 量測點、sense point、sysfs rail name 已建立對照

Device Tree / Kernel:

- [ ] 對應 I2C bus node `status = "okay"`
- [ ] PMBus / VR / PSU / hwmon node 的 compatible 與 reg 正確
- [ ] kernel config 已開啟 PMBus core 與對應 chip driver
- [ ] 開機後 dmesg 無 probe 失敗或 timeout
- [ ] /sys/class/hwmon/hwmonX/in*_input 存在且讀值合理
- [ ] `in*_label` 或實測對照可正確映射到 rail
- [ ] 使用 DMM / oscilloscope 比對 sysfs 數值, 誤差符合平台規格
- [ ] Power state 切換時, 讀值、availability、daemon log 行為符合預期

Entity Manager / Userspace:

- [ ] JSON 設定檔已放入平台 layer 或 image 內正確路徑
- [ ] Type 名稱與 sensor daemon 支援清單一致
- [ ] Bus / Address / Page / Index 數值正確
- [ ] ScaleFactor 與 Offset 已按硬體設計調整
- [ ] PollInterval 符合監控需求, 建議先以 500~2000 ms 驗證
- [ ] PowerState 設定符合 rail 供電時序
- [ ] Thresholds 上下限與 Severity 已設定並經 review

D-Bus / 系統整合:

- [ ] 對應 sensor service 啟動無錯誤
- [ ] D-Bus tree 出現 /xyz/openbmc_project/sensors/voltage/<Name>
- [ ] `xyz.openbmc_project.Sensor.Value` 可讀到 double 值, 單位為 V
- [ ] Warning / Critical threshold properties 寫入 D-Bus
- [ ] Redfish /redfish/v1/Chassis/.../Sensors 可查到 sensor
- [ ] 負載或電源變化時, sysfs 與 D-Bus 數值同步變化
- [ ] 過壓 / 欠壓條件下, 對應 alarm flag 變為 true
- [ ] threshold 觸發時有 journal / event log / SEL 或平台定義事件
- [ ] 若有電源管理或降載 policy, 電壓異常時行為符合規格

### 24.5.7 本章參考資料

- OpenBMC dbus-sensors README: https://github.com/openbmc/dbus-sensors/blob/master/README.md
- Linux kernel hwmon sysfs interface: https://www.kernel.org/doc/html/latest/hwmon/sysfs-interface.html
- Linux kernel PMBus driver documentation: https://docs.kernel.org/hwmon/pmbus.html
- Linux kernel PMBus register definitions: https://github.com/torvalds/linux/blob/master/drivers/hwmon/pmbus/pmbus. h

<a id="section-12-6"></a>

## 24.6 Current Sensor

### 24.6.1 適用情境

Current Sensor 用於監控系統中各電源軌或負載支路的電流消耗, 常用於功耗估算、電源轉換效率分析、過流保護驗證、PSU / VR 健康狀態判讀, 以及量產測試時的異常耗電篩查.

常見監控對象包含:

```text
PSU input current：電源供應器輸入電流
PSU output current：電源供應器輸出電流
VR output current：CPU Vcore、VCCIN、DIMM rail 等 VR 輸出電流
Board rail current：主機板 12V / 5V / 3V3 / standby rail 電流
GPU current：GPU 或 accelerator module 電流
DIMM current：記憶體模組或 memory power rail 電流
Fan current：風扇支路電流，常用於堵轉、短路或異常負載偵測
Hot-swap / eFuse current：輸入保護或板級支路電流
```

在 OpenBMC 中, Current Sensor 最終應以標準 sensor object 呈現, 常見 D-Bus path 為:

```text
/xyz/openbmc_project/sensors/current/<sensor_name>
```

OpenBMC 的 sensor daemon 可能從 hwmon、D-Bus 或 direct driver access 讀值, 再發佈 `xyz.openbmc_project.Sensor.Value`、threshold、availability、operational status 與 association 等介面. 後續 Redfish、IPMI SDR、logging、fan / thermal policy 或 power policy 會再消費這些 D-Bus sensor object.

### 24.6.2 資料路徑

```mermaid
flowchart LR
    A{Current source} -->|Shunt + amplifier| B[ADC / IIO]
    A -->|READ_IIN / READ_IOUT| C[PMBus VR / PSU]
    A -->|Current monitor| D[INA / HSC / eFuse]
    B --> E[in*_input raw or mV]
    C --> F[curr*_input mA]
    D --> F
    E --> G[Scale with Rshunt + Gain]
    F --> H[Convert mA to A]
    G --> I[D-Bus current A]
    H --> I
    I --> J[Redfish / IPMI / Event / Power calculation]
```

Current Sensor 常見資料路徑可分為三類.

#### 路徑 A: 分流電阻 + 放大器 + ADC

```text
電流通過分流電阻（Shunt Resistor）
    ↓
產生壓降：V_sense = I × R_shunt
    ↓
差動放大器 / current sense amplifier 放大壓降：V_adc = V_sense × Gain
    ↓
ADC 取樣，例如 BMC SoC 內建 ADC 或外部 ADC
    ↓
IIO / iio-hwmon / hwmon sysfs
    ↓
ADCSensor 或平台 sensor daemon
    ↓
D-Bus：/xyz/openbmc_project/sensors/current/<Name>
```

這條路徑適合 board rail、fan branch current、standby current 或客製量測點. 主要風險是 scaling 同時受 `R_shunt`、amplifier gain、ADC 參考電壓、driver 輸出單位影響, 任一資料錯誤都會造成固定比例偏差.

#### 路徑 B: PMBus / VR / PSU 控制器

```text
VR / PSU / Hot-swap Controller 內建 current sense 與 ADC
    ↓
PMBus command：READ_IIN / READ_IOUT
    ↓
Linux PMBus driver 或 chip-specific hwmon driver
    ↓
hwmon sysfs：/sys/class/hwmon/hwmonX/curr*_input
    ↓
PSUSensor / Hwmon 類 sensor daemon / 平台 sensor daemon
    ↓
D-Bus：/xyz/openbmc_project/sensors/current/<Name>
```

PMBus command code 中, 常見 current / power 相關命令如下:

| PMBus command | Code | 常見用途 |
| --- | ---: | --- |
| `READ_IIN` | `0x89` | 輸入電流, 例如 PSU input current |
| `READ_IOUT` | `0x8C` | 輸出電流, 例如 VR output current / PSU output current |
| `READ_POUT` | `0x96` | 輸出功率, 屬於 Power Sensor 範圍 |
| `READ_PIN` | `0x97` | 輸入功率, 屬於 Power Sensor 範圍 |

Linux PMBus driver 可支援 voltage、current、power、temperature 等 sensor. PMBus generic device 通常不會依賴安全自動 probe, 實務上需透過 DTS、I2C device instantiate、platform config 或 Entity Manager 讓 device 被明確建立.

#### 路徑 C: 獨立 Current / Power Monitor IC

```text
INA219 / INA226 / INA233 / INA3221 / LTC2945 / Hot-swap Controller / eFuse
    ↓
I2C / SMBus driver
    ↓
hwmon sysfs：curr*_input、in*_input、power*_input
    ↓
Hwmon 類 daemon / PSUSensor / 平台 daemon
    ↓
D-Bus：/xyz/openbmc_project/sensors/current/<Name>
```

INA2xx 類 current shunt / power monitor 通常同時量測 shunt voltage 與 bus voltage, 部分晶片可直接計算 current / power. Porting 時需確認 shunt resistor 設定來源, 例如 device tree、platform data 或 driver 預設值, 避免 driver 使用錯誤 shunt value.

### 24.6.3 常見來源分類與特性

| 來源類型 | 感測原理 | Linux / OpenBMC 對應 | 關鍵注意事項 |
| :--- | :--- | :--- | :--- |
| 分流電阻 + ADC | `I = V_sense /R_shunt`, 再經放大器與 ADC | IIO / iio-hwmon / ADCSensor / 平台 daemon | 需計算 `R_shunt`、Gain、ADC 單位與 `ScaleFactor` |
| PMBus VR | VR controller 內建 current sense | PMBus hwmon、VR chip driver、Hwmon 類 daemon | 需確認 Page、Phase、`READ_IOUT`、calibration 與 host power state |
| PMBus PSU | PSU 內建 input/output current telemetry | PMBus hwmon、PSUSensor | 需區分 input current、output current、fault status 與 redundancy 行為 |
| Hot-swap controller / eFuse | high-side current sense、OCP / fault latch | hwmon、PMBus 或平台 daemon | 常同時涉及 presence、power good、fault latch 與 event |
| INA2xx / INA3221 | 量測 shunt voltage 與 bus voltage | `ina2xx` / `ina3221` hwmon driver | `shunt-resistor` 設定需與 BOM 一致; 多 channel 需建立 label 對照 |
| 風扇電流偵測 | 分流電阻 + ADC 或支路 current monitor | ADC / hwmon / 客製 daemon | 常用於異常偵測, 需與 fan tach 判斷交叉驗證 |
| GPU / Accelerator current | PMBus / sideband / module controller | PMBus、MCTP / PLDM、GPU sensor daemon | 需注意 power state、module presence、telemetry refresh rate |

### 24.6.4 單位與 ScaleFactor 設計原則

Current Sensor 最常見問題是單位混淆. 建議每個 sensor 都明確記錄:

```text
硬體量測點：V_sense，通常是 V 或 mV
ADC sysfs：可能是 raw code、mV，或 driver 定義的 fixed-point value
hwmon curr*_input：Linux hwmon 規範通常為 milliampere（mA）
D-Bus Sensor.Value：OpenBMC Current Sensor 建議以 Ampere（A）檢查
Redfish Reading：依 schema / implementation 呈現，通常應可標示 A
```

Linux hwmon sysfs 值是 fixed-point single-value 檔案, current input `curr[1-*]_input` 的單位通常為 milliampere. 因此:

```text
cat curr1_input = 15000
=> 15000 mA
=> 15 A
```

#### 24.6.4.1 分流電阻 + ADC 換算

硬體基本式:

```text
V_sense = I × R_shunt
V_adc = V_sense × Gain
I = V_adc / (Gain × R_shunt)
```

若 sysfs `inY_input` 已是 ADC pin 上的電壓, 且單位為 mV:

```text
ADC_Pin_Voltage_V = inY_input / 1000
Current_A = ADC_Pin_Voltage_V / (Gain × R_shunt)
Current_A = inY_input × 0.001 / (Gain × R_shunt)
ScaleFactor = 0.001 / (Gain × R_shunt)
```

若 daemon 的輸入值已是 V:

```text
Current_A = input_V / (Gain × R_shunt)
ScaleFactor = 1 / (Gain × R_shunt)
```

若輸入值是 ADC raw code:

```text
ADC_Pin_Voltage_V = Raw_Code × (V_ref / 2^N)
Current_A = Raw_Code × (V_ref / 2^N) / (Gain × R_shunt)
ScaleFactor = (V_ref / 2^N) / (Gain × R_shunt)
```

範例:

```text
R_shunt = 0.001 ohm = 1 mΩ
Gain = 50
ADC sysfs = 50 mV
Current_A = 0.05 / (50 × 0.001) = 1.0 A

若 daemon ScaleFactor 乘在 mV 數值上：
ScaleFactor = 0.001 / (50 × 0.001) = 0.02
50 × 0.02 = 1.0 A

若 daemon ScaleFactor 乘在 V 數值上：
ScaleFactor = 1 / (50 × 0.001) = 20.0
0.05 × 20.0 = 1.0 A
```

因此, 如果文件中寫 `ScaleFactor = 20.0`, 需同步註明 daemon 的輸入值是 V; 若輸入值是 mV, 則應使用 `0.02`. 這個差異是 Current Sensor porting 中很常見的 1000 倍錯誤來源.

#### 24.6.4.2 hwmon `curr*_input` 換算

如果 kernel driver 已輸出標準 hwmon `curr*_input`, 通常代表:

```text
sysfs_current_mA = cat currX_input
Current_A = sysfs_current_mA / 1000
```

若 OpenBMC daemon 已把 hwmon current 從 mA 轉成 A,`ScaleFactor` 通常保留 `1.0`. 若平台 daemon 直接把 sysfs 值當作 D-Bus value, 則需補 `ScaleFactor = 0.001`. 實務上需以該專案 sensor daemon 的行為為準, 不能只看 JSON 欄位名稱判斷.

#### 24.6.4.3 PMBus Linear / Direct 換算

PMBus device 可能以 Linear11、Linear16 或 Direct format 回傳 telemetry. Linux PMBus core 與 chip-specific driver 通常會依 device capability、driver info 或 device-specific conversion 轉成 hwmon 單位. 若讀值呈現固定比例偏差, 需確認:

```text
[ ] PMBus driver 是否支援該 chip，或只是使用 generic pmbus
[ ] 該 channel 使用 Linear 或 Direct format
[ ] driver 中 m / b / R 係數是否符合 datasheet
[ ] IOUT / IIN calibration 是否由 driver、device NVM 或平台設定提供
[ ] Page / Phase 是否正確
[ ] sysfs 是 mA，還是已被平台 daemon 用其他方式處理
```

### 24.6.5 Porting 路徑 A: 分流電阻 + ADC 電流偵測

#### Step A1: 確認硬體參數

從 schematic、BOM、datasheet 與量測資料取得:

```text
[ ] 分流電阻值 R_shunt，單位 ohm，例如 0.001 Ω
[ ] 分流電阻精度與溫度係數，例如 1%、0.5%、50 ppm/°C
[ ] 分流電阻位置：high-side / low-side
[ ] current sense amplifier 型號與 Gain
[ ] amplifier input common-mode range 是否涵蓋待測 rail
[ ] amplifier output offset / bidirectional current reference
[ ] ADC 參考電壓 V_ref
[ ] ADC 解析度 N-bit
[ ] ADC input range 與保護電路
[ ] 待測電流最大值與過流保護門檻
[ ] 電流方向定義：正向 / 反向 / bidirectional
```

設計檢查建議:

```text
V_sense_max = I_max × R_shunt
V_adc_max = V_sense_max × Gain
P_shunt_max = I_max^2 × R_shunt
```

需確認 `V_adc_max` 不會超出 ADC input range,`P_shunt_max` 不會超過分流電阻額定功率, 且 `V_sense` 在低電流時仍高於 ADC noise floor.

#### Step A2: 確認 ADC / iio-hwmon / sysfs

此部分與第 12.3 ADC Sensor 共用. Device Tree 需啟用 ADC controller 與必要的 iio-hwmon mapping. 開機後先確認 sysfs:

```bash
$ for h in /sys/class/hwmon/hwmon*; do
    echo "== $h =="
    cat "$h/name" 2>/dev/null || true
    ls "$h"/in*_input 2>/dev/null || true
    ls "$h"/curr*_input 2>/dev/null || true
$ done

$ watch -n 1 'for f in /sys/class/hwmon/hwmon*/in*_input /sys/class/hwmon/hwmon*/curr*_input; do [ -e "$f" ] && echo "$f=$(cat $f)"; done'
```

若是通用 ADC, 常見只會有 `in*_input`, 需要在 sensor daemon / Entity Manager 進行換算. 若是 current monitor driver, 可能直接提供 `curr*_input`.

#### Step A3: 建立量測對照表

Bring-up 階段建議至少建立 idle、typical load、stress load 三個負載點.

| 負載狀態 | sysfs raw / mV | DMM / clamp meter 實測 A | D-Bus A | 誤差 | 備註 |
| --- | ---: | ---: | ---: | ---: | --- |
| Host off / standby | [待填] | [待填] | [待填] | [待填] | [待填] |
| Host on idle | [待填] | [待填] | [待填] | [待填] | [待填] |
| CPU / GPU stress | [待填] | [待填] | [待填] | [待填] | [待填] |

若誤差呈固定比例, 優先檢查 `R_shunt`、Gain、sysfs 單位與 `ScaleFactor`. 若誤差隨負載變化, 需檢查 amplifier offset、ADC 線性度、量測工具位置、PCB trace drop、溫度與濾波設定.

### 24.6.6 Porting 路徑 B: PMBus / VR / PSU 電流偵測

#### Step B1: 確認硬體資訊與 PMBus channel

從 schematic、board power tree、VR / PSU datasheet 取得:

```text
[ ] VR / PSU / HSC 型號
[ ] I2C bus number、mux channel、7-bit address
[ ] PMBus Page 對應 rail，例如 Page 0 = Vcore、Page 1 = VCCGT
[ ] PMBus Phase 是否需要指定
[ ] Current command：READ_IIN / READ_IOUT
[ ] IOUT / IIN calibration 或 sense resistor 設定來源
[ ] Device 是否需要上電或 host on 才回應該 command
[ ] 是否需要 vendor-specific unlock / telemetry enable
[ ] Fault / warning command：STATUS_IOUT、STATUS_INPUT、IOUT_OC_WARN_LIMIT 等
```

多 rail VR 需建立 Page 對照表:

| Device | Bus | Address | Page | Rail | 預期 sysfs | 備註 |
| --- | ---: | ---: | ---: | --- | --- | --- |
| CPU0 VR | 8 | 0x40 | 0 | CPU0_VCORE | curr1_input | [待填] |
| CPU0 VR | 8 | 0x40 | 1 | CPU0_VCCGT | curr2_input | [待填] |
| DIMM VR | 8 | 0x42 | 0 | P1V1_DIMM | curr1_input | [待填] |

#### Step B2: I2C / PMBus 基本通訊驗證

先確認 bus、mux channel 與 address.`i2cdetect` 對部分 device 可能有副作用, 使用前需確認 device 行為.

```bash
$ i2cdetect -y <bus>

# 視 device 是否支援，讀 MFR_ID / MFR_MODEL
$ i2cget -y <bus> <addr> 0x99
$ i2cget -y <bus> <addr> 0x9A
```

讀取 Page 0 的 output current 前, 可先切 Page. 下列指令僅作 bring-up 輔助, 實際 byte order 需依 `i2cget` 顯示與 PMBus word 格式確認:

```bash
# 設定 PAGE = 0
$ i2cset -y <bus> <addr> 0x00 0x00

# READ_IOUT = 0x8C，word read
$ i2cget -y <bus> <addr> 0x8C w

# READ_IIN = 0x89，word read
$ i2cget -y <bus> <addr> 0x89 w
```

注意事項:

```text
- 某些 PMBus device 在 host off 或 rail disabled 時會回傳 0、NACK、0xffff 或設定 status fault。
- `i2cget ...w` 顯示的 word 可能需要 byte swap 後才符合 PMBus raw word。
- 手動讀取 telemetry 不代表 kernel driver conversion 一定正確，仍需比對 hwmon sysfs。
- 不建議在不了解 device 行為時寫入 calibration、limit 或 operation command。
```

#### Step B3: Device Tree / driver binding

PMBus VR 範例:

```dts
&i2c8 {
    status = "okay";
    clock-frequency = <100000>;

    vr-controller@40 {
        compatible = "mps,mp2971";
        reg = <0x40>;
        label = "cpu0_vr";
    };
};
```

INA2xx 類 current monitor 範例:

```dts
&i2c8 {
    status = "okay";

    current-monitor@41 {
        compatible = "ti,ina226";
        reg = <0x41>;
        shunt-resistor = <1000>; /* micro-ohm，需依 binding 與專案 kernel 確認 */
        label = "p12v_current_monitor";
    };
};
```

Kernel config 檢查方向:

```text
CONFIG_HWMON
CONFIG_I2C
CONFIG_PMBUS
CONFIG_SENSORS_PMBUS
CONFIG_SENSORS_INA2XX
CONFIG_SENSORS_INA3221
對應 VR / PSU / HSC chip-specific driver，例如 MP297x、TPS536xx、LTC、ADM、MAX、IR 等
```

實際 symbol 名稱會隨 kernel 版本與 vendor BSP 不同而改變, 請以 `bitbake -c menuconfig virtual/kernel`、`.config`、driver Kconfig 與 `dmesg` 為準.

#### Step B4: 確認 driver probe 與 hwmon sysfs

```bash
$ dmesg | grep -Ei 'pmbus|mp297|tps536|ina2|ina3221|hwmon|i2c'

$ for h in /sys/class/hwmon/hwmon*; do
    echo "== $h =="
    cat "$h/name" 2>/dev/null || true
    for f in "$h"/curr*_label "$h"/curr*_input "$h"/in*_input "$h"/power*_input; do
        [ -e "$f" ] && echo "$(basename "$f")=$(cat "$f")"
    done
$ done
```

`curr*_label` 若存在, 優先以 label 建立對照; 若不存在, 需依 driver 文件、Page 順序、實際負載變化與 PMBus 手動讀值建立 mapping.

### 24.6.7 Entity Manager / Sensor JSON 配置

不同 OpenBMC branch、平台 layer 與 sensor daemon 對 JSON 欄位的名稱可能不同. 以下範例用於說明欄位意義, 實際需參考專案的 Entity Manager schema、`sensor-info.json`、既有平台 JSON 與 daemon source.

#### PMBus / VR current 範例

```json
{
  "Name": "CPU0_VCORE_CURRENT",
  "Type": "MP2971",
  "Bus": 8,
  "Address": "0x40",
  "Page": 0,
  "PollInterval": 500,
  "ScaleFactor": 1.0,
  "Offset": 0.0,
  "MaxValue": 300.0,
  "MinValue": 0.0,
  "PowerState": "On",
  "Thresholds": [
    {
      "Name": "upper critical",
      "Direction": "greater than",
      "Severity": 1,
      "Value": 250.0
    },
    {
      "Name": "upper non critical",
      "Direction": "greater than",
      "Severity": 0,
      "Value": 200.0
    }
  ]
}
```

設定重點:

```text
- 若 daemon 已把 hwmon mA 轉成 D-Bus A，ScaleFactor 通常為 1.0。
- 若 daemon 直接使用 curr*_input 數值，需設定 ScaleFactor = 0.001。
- CPU / DIMM VR 通常只在 host power on 後有效，PowerState 可設為 On。
- PSU input current 或 standby rail current 可能為 AlwaysOn。
- Page 錯誤時可能仍有合理數字，因此需用負載變化驗證對應 rail。
```

#### 分流電阻 + ADC current 範例

```json
{
  "Name": "P12V_CURRENT",
  "Type": "ADC",
  "Index": 2,
  "ScaleFactor": 0.02,
  "Offset": 0.0,
  "PollInterval": 500,
  "MaxValue": 60.0,
  "MinValue": 0.0,
  "PowerState": "AlwaysOn",
  "Thresholds": [
    {
      "Name": "upper critical",
      "Direction": "greater than",
      "Severity": 1,
      "Value": 50.0
    }
  ]
}
```

上例假設 ADC sysfs 輸入值單位為 mV, 且 `R_shunt = 1 mΩ`、`Gain = 50`:

```text
ScaleFactor = 0.001 / (50 × 0.001) = 0.02
```

若 ADC daemon 讀到的值已是 V, 則同一組硬體應使用:

```text
ScaleFactor = 1 / (50 × 0.001) = 20.0
```

### 24.6.8 啟動服務與 D-Bus 驗證

先找出平台實際的 sensor service:

```bash
$ systemctl list-units '*sensor*' --no-pager
$ systemctl list-units '*psu*' --no-pager
```

依平台重啟相關服務. 若專案使用 HwmonCurrentSensor 類服務, 可用:

```bash
$ systemctl restart xyz.openbmc_project.HwmonCurrentSensor.service
$ journalctl -u xyz.openbmc_project.HwmonCurrentSensor.service -b --no-pager | tail -100
```

若 Current Sensor 由 PSUSensor 或其他 daemon 產生, 請改查對應 service:

```bash
$ journalctl -b --no-pager | grep -Ei 'current|curr|pmbus|psu|sensor|mp297|ina2'
```

確認 D-Bus object 是否存在:

```bash
$ busctl tree /xyz/openbmc_project/sensors
$ busctl tree /xyz/openbmc_project/sensors/current
```

讀取 Current Sensor 數值:

```bash
$ busctl get-property \
  <service_name> \
  /xyz/openbmc_project/sensors/current/CPU0_VCORE_CURRENT \
  xyz.openbmc_project.Sensor.Value \
  Value
```

若不知道 service owner, 使用 ObjectMapper 查:

```bash
$ busctl call xyz.openbmc_project.ObjectMapper \
  /xyz/openbmc_project/object_mapper \
  xyz.openbmc_project.ObjectMapper GetObject \
  sas \
  /xyz/openbmc_project/sensors/current/CPU0_VCORE_CURRENT \
  0
```

建議至少比對三層讀值:

```text
PMBus / ADC 原始讀值
    ↔ hwmon / IIO sysfs
        ↔ D-Bus Sensor.Value
            ↔ Redfish / IPMI 顯示
```

### 24.6.9 Redfish / IPMI / Event 整合驗證

Redfish 驗證:

```bash
$ curl -k -u root:<password> \
  https://<bmc-ip>/redfish/v1/Chassis/<chassis>/Sensors | jq

$ curl -k -u root:<password> \
  https://<bmc-ip>/redfish/v1/Chassis/<chassis>/Sensors/CPU0_VCORE_CURRENT | jq
```

驗證重點:

```text
[ ] Sensor 名稱與 D-Bus path 對應
[ ] Reading / Value 與 D-Bus 一致
[ ] ReadingUnits 為 A 或可清楚表示 Ampere
[ ] Warning / Critical threshold 顯示正確
[ ] Status.Health / State 與 D-Bus availability、operational status 一致
```

IPMI 驗證:

```bash
$ ipmitool sensor | grep -i current
$ ipmitool sdr elist | grep -i current
```

若 IPMI 看不到 Current Sensor, 但 D-Bus / Redfish 都正常, 需檢查 SDR generation、sensor number、entity ID、sensor type、threshold mapping 與 IPMI bridge policy.

Threshold / event 驗證:

```bash
$ busctl introspect <service_name> \
  /xyz/openbmc_project/sensors/current/CPU0_VCORE_CURRENT

$ journalctl -b --no-pager | grep -Ei 'CPU0_VCORE_CURRENT|threshold|critical|warning|current'

$ curl -k -u root:<password> \
  https://<bmc-ip>/redfish/v1/Systems/system/LogServices/EventLog/Entries | jq
```

### 24.6.10 進階除錯與常見陷阱

| 問題現象 | 建議排查方向 | 排查 / 處理方式 |
| :--- | :--- | :--- |
| `curr*_input` 讀值為 0 | Host off、VR disabled、PSU standby、Page 錯、driver 未更新 | 確認 power state、enable signal、Page、sysfs refresh; 與 PMBus `READ_IOUT` 比對 |
| `curr*_input` 不存在 | Driver 未 bind、compatible 錯、PMBus command 不支援、kernel config 未啟用 | 查 `dmesg`、DTS、Kconfig、`/sys/bus/i2c/devices/*/driver` |
| D-Bus sensor 未出現 | JSON `Type` / `Bus` / `Address` / `Page` 不符合 daemon 預期 | 查 Entity Manager log、sensor daemon log、ObjectMapper subtree |
| 讀值為負數 | current sense 正負端接反、bidirectional sensor offset 未處理 | 查 schematic high-side / low-side、shunt direction、driver sign handling |
| 讀值固定大 1000 倍 | mA 被當成 A, 或 daemon 未做 hwmon 單位轉換 | 確認 `curr*_input` 單位; 必要時設定 `ScaleFactor = 0.001` |
| 讀值固定小 1000 倍 | 已轉成 A 後又乘 0.001 | 比對 sysfs、D-Bus、Redfish; 移除重複 scaling |
| 讀值與勾表差異固定比例 | `R_shunt`、Gain、PMBus coefficient、shunt-resistor 設定錯 | 查 BOM、datasheet、DTS、driver conversion、量測點位置 |
| 讀值隨負載跳動劇烈 | ADC noise、shunt 壓降太小、I2C refresh 慢、負載本身 pulse | 增加 averaging、調整 poll interval、用示波器檢查 ripple |
| 多 Page 電流全部相同 | driver 未正確切 Page、JSON Page 未生效、讀到同一 channel | 手動寫 PAGE 後讀 `READ_IOUT`; 查 driver page support |
| PMBus 讀到 `0xffff` 或 NACK | rail off、command unsupported、device fault、bus timeout | 查 `STATUS_WORD`、`STATUS_IOUT`、host power、I2C 波形 |
| Redfish 數值不變 | sensor daemon 未更新、PowerState gating、sysfs cache、bmcweb route cache | 直接 `cat curr*_input`, 再比對 D-Bus `Value` 與 Redfish |
| Threshold 觸發但無事件 | threshold interface 未建立、logging policy 未接、alarm flag 未變化 | 查 D-Bus threshold properties、phosphor-logging、EventLog |

### 24.6.11 Current Sensor Porting 驗收 Checklist

硬體設計階段:

- [ ] Current source 類型確認: 分流電阻 + ADC / PMBus VR / PSU / INA / HSC / eFuse
- [ ] Power tree 中 sensor 量測點與 rail 名稱確認
- [ ] 若為分流電阻架構: R_shunt、精度、溫度係數、額定功率確認
- [ ] 若有 amplifier: Gain、offset、common-mode range、輸出範圍確認
- [ ] 若為 PMBus VR: I2C bus、mux channel、address、Page、Phase 確認
- [ ] 若為 INA / HSC: shunt-resistor / calibration 設定來源確認
- [ ] 待測電流範圍、正常值、warning、critical、硬體 OCP 門檻確認
- [ ] current sense 方向確認, 避免正負號或 bidirectional offset 問題

Device Tree / Kernel:

- [ ] I2C bus node status = "okay"
- [ ] I2C mux channel 與 bus number 對照完成
- [ ] PMBus / VR / INA / HSC device node 加入, compatible 與 reg 正確
- [ ] 必要 kernel config 已啟用, 如 PMBus、chip-specific driver、INA2xx、hwmon
- [ ] 開機後 dmesg 無 probe failure、timeout、unsupported command 相關錯誤
- [ ] /sys/class/hwmon/hwmonX/name 可對應到目標 device
- [ ] curr*_input 存在, 且單位已確認為 mA 或平台定義值
- [ ] curr*_label 或 Page 對照表已建立
- [ ] 使用 DMM / clamp meter / PSU telemetry 比對, 工程誤差在可接受範圍內

Entity Manager / Userspace:

- [ ] JSON 設定檔加入正確 layer, 並已編入 image
- [ ] `Name` 命名符合平台 sensor naming rule
- [ ] `Type` 與 daemon / sensor-info 定義一致
- [ ] 分流 + ADC 架構: Index、ScaleFactor、Offset 計算完成
- [ ] PMBus 架構: Bus、Address、Page、PowerState 設定正確
- [ ] ScaleFactor 已依 sysfs 單位確認, 不重複乘 0.001 或漏轉 mA → A
- [ ] PollInterval 符合需求, 常見建議 500～1000 ms
- [ ] PowerState 設定符合 rail 行為, 例如 CPU VR 使用 On、standby rail 使用 AlwaysOn
- [ ] Thresholds 與硬體保護門檻、系統降載策略一致

D-Bus / Redfish / IPMI:

- [ ] 對應 sensor service 啟動無錯誤
- [ ] ObjectMapper 可找到 /xyz/openbmc_project/sensors/current/<Name>
- [ ] busctl get-property 可讀取 Value, 且單位以 A 檢查合理
- [ ] Warning / Critical threshold property 存在且數值正確
- [ ] Redfish Sensor resource 可看到該 current sensor
- [ ] IPMI SDR / sensor list 依平台需求可看到該 sensor
- [ ] 施加負載變化時, sysfs、D-Bus、Redfish 數值同步變化
- [ ] Threshold 觸發時, D-Bus alarm flag、phosphor-logging、SEL / EventLog 行為符合預期
- [ ] 若用於 P = V × I, 已確認 voltage sensor 與 current sensor 的量測點相同或可接受

### 24.6.12 Current Sensor 資料表範本

| Sensor Name | Source Type | Device | Bus / Addr | Page / Channel | Raw sysfs | sysfs unit | ScaleFactor | D-Bus unit | PowerState | Threshold | 備註 |
| --- | --- | --- | --- | --- | --- | --- | ---: | --- | --- | --- | --- |
| CPU0_VCORE_CURRENT | PMBus VR | MP2971 | 8 / 0x40 | Page 0 | curr1_input | mA | 1.0 或 0.001 | A | On | [待填] | [待填] |
| P12V_CURRENT | ADC + Shunt | ADC channel 2 | N/A | Index 2 | in2_input | mV | 0.02 | A | AlwaysOn | [待填] | R=1mΩ, Gain=50 |
| PSU0_INPUT_CURRENT | PMBus PSU | PSU0 | [待填] | input | curr1_input | mA | [待填] | A | AlwaysOn | [待填] | [待填] |

### 24.6.13 本節參考資料

- Linux kernel hwmon sysfs interface:`curr[1-*]_input` 為 current input value, 單位為 milliampere; hwmon sysfs 使用固定命名與 fixed-point single-value 檔案格式.
- Linux kernel PMBus driver documentation: PMBus driver 支援 voltage、current、power、temperature 等 sensor, 且 PMBus device 通常需明確建立, 不依賴安全自動 probe.
- Linux PMBus register definitions:`READ_IIN = 0x89`、`READ_IOUT = 0x8C`、`READ_POUT = 0x96`、`READ_PIN = 0x97`.
- OpenBMC dbus-sensors README: sensor daemon 可從 hwmon、D-Bus 或 direct driver access 讀取資料, 並以 `/xyz/openbmc_project/sensors/<type>/<sensor_name>` 發佈 D-Bus sensor object.
- Linux INA2xx hwmon documentation: INA2xx 類 current shunt / power monitor 會量測 shunt drop 與 bus voltage, shunt resistor 可由 platform data、device tree 或部分 sysfs attribute 指定.

<a id="section-12-7"></a>

## 24.7 Power Sensor

### 24.7.1 適用情境

Power Sensor 用於監控系統中各元件或電源路徑的功耗, 常用於能源效率評估、散熱設計驗證、功率預算管理、power capping、PSU redundancy、長測趨勢分析, 以及現場耗電異常排查.

常見監控對象包含:

```text
PSU input power：電源供應器輸入功率，常用於整機 AC / DC 輸入耗電
PSU output power：電源供應器輸出功率，常用於 PSU loading 與 redundancy 判斷
CPU package power：CPU 封裝功率，可能由 CPU telemetry、PECI 或 VR PMBus 間接取得
GPU power：GPU 或 accelerator 模組功率，可能由 MCTP / PLDM、SMBPBI 或 vendor path 提供
Board power：主機板總功耗，可能由 HSC、PSU output 或 V × I 彙總取得
VR power：電壓調節模組輸出功率，例如 Vcore / DIMM rail power
DIMM power：記憶體模組或 memory rail 功耗
NVMe power：NVMe / EDSFF / U.2 / M.2 裝置功耗，可能由 NVMe-MI / PLDM 或平台控制器提供
```

在 OpenBMC 中, Power Sensor 最終應以標準 sensor object 呈現, 常見 D-Bus path 為:

```text
/xyz/openbmc_project/sensors/power/<sensor_name>
```

Power Sensor 的 D-Bus `Value` 建議以 Watt(W)檢查與記錄; 底層 Linux hwmon `power*_input` 通常是 microWatt(µW). 因此 porting 時需要明確記錄"底層 sysfs 單位"、"daemon 是否已換算"以及"D-Bus 顯示單位".

### 24.7.2 資料路徑

```mermaid
flowchart TB
    A{Power source} -->|READ_PIN / READ_POUT| B[PMBus / hwmon power*_input µW]
    A -->|Calculated| C[Voltage V × Current A]
    A -->|Device telemetry| D[CPU / GPU / NVMe daemon]
    A -->|Aggregation| E[Virtual sensor]
    B --> F[Convert µW to W]
    C --> G[Validate same measurement point]
    D --> H[Normalize unit + timeout]
    E --> H
    F --> I[D-Bus power W]
    G --> I
    H --> I
    I --> J[Redfish / IPMI / Telemetry]
    I --> K[Power cap / Thermal policy]
```

Power Sensor 的資料來源可分為四類: PMBus 直接讀取功率、軟體 V × I 計算、CPU / GPU / NVMe telemetry, 以及平台虛擬彙總功率.

#### 路徑 A: PMBus 直接讀取功率(硬體計算)

```text
VR Controller / PSU / Hot-swap Controller / Power Monitor 內建功率計算
    ↓
PMBus command：READ_POUT / READ_PIN
    ↓
Linux PMBus driver 或 chip-specific hwmon driver
    ↓
hwmon sysfs：/sys/class/hwmon/hwmonX/power*_input
    ↓
PSUSensor / Hwmon 類 sensor daemon / 平台 sensor daemon
    ↓
D-Bus：/xyz/openbmc_project/sensors/power/<Name>
```

此路徑由硬體或 device firmware 直接計算功率, 通常比單純 V × I 的軟體計算更接近 device 自身定義的 telemetry. VR、PSU、hot-swap controller、INA233 / INA2xx / LTC 類 power monitor 都可能提供此類資料.

#### 路徑 B: 軟體計算功率(V × I)

```text
Voltage Sensor：/xyz/openbmc_project/sensors/voltage/<voltage_name>
Current Sensor：/xyz/openbmc_project/sensors/current/<current_name>
    ↓
ExternalSensor / VirtualSensor / 客製 daemon 定期讀取兩者
    ↓
計算 P = V × I
    ↓
D-Bus：/xyz/openbmc_project/sensors/power/<Name>
```

此路徑適合硬體沒有 `READ_POUT` / `READ_PIN`, 但已有穩定 Voltage Sensor 與 Current Sensor 的場景, 例如分流電阻 + ADC 架構. 主要風險是 voltage 與 current 的取樣時間不一致、量測點不完全相同、單位已被重複換算, 以及負載快速變動造成瞬時誤差.

#### 路徑 C: CPU / GPU / NVMe / Accelerator telemetry

```text
CPU / GPU / NVMe / Accelerator 內部 telemetry
    ↓
PECI / MCTP / PLDM / SMBus / SMBPBI / vendor interface
    ↓
對應 userspace daemon 或 kernel driver
    ↓
D-Bus：/xyz/openbmc_project/sensors/power/<Name>
```

此路徑常見於 CPU package power、GPU board power、NVMe power state 或 accelerator 功率. 數值來源可能是裝置內部估算, 不一定等同於 board rail 上的實際輸入功率, 因此文件中需清楚標示 measure point 與資料來源.

#### 路徑 D: 平台虛擬彙總功率

```text
多個 power / voltage / current sensors
    ↓
VirtualSensor / ExternalSensor / platform daemon
    ↓
加總、扣除、平均或效率計算
    ↓
D-Bus：/xyz/openbmc_project/sensors/power/<Name>
```

常見用途包含 board total power、CPU domain total power、GPU tray total power、PSU total output power 或整機估算功率. 彙總功率應避免重複計入同一能量路徑, 例如 PSU output power 已包含所有 downstream rail, 再加上 CPU VR power 會造成 double counting.

### 24.7.3 常見來源分類與特性

| 來源類型 | 讀取介面 | Linux / OpenBMC 對應 | 關鍵注意事項 |
| :--- | :--- | :--- | :--- |
| PMBus VR | `READ_POUT` (`0x96`) | PMBus hwmon、VR chip driver、Hwmon 類 daemon | 需確認 Page / Phase、POUT scaling、host power state |
| PMBus PSU | `READ_PIN` (`0x97`) / `READ_POUT` (`0x96`) | PMBus hwmon、PSUSensor | input 與 output power 需分開命名與驗證 |
| Hot-swap / eFuse / HSC | PMBus / I2C / register | hwmon、PMBus 或平台 daemon | 通常同時有 current、voltage、power、fault latch |
| INA233 / INA2xx / INA3221 | I2C / PMBus / hwmon | `ina2xx` / `ina233` / `ina3221` driver | shunt resistor、calibration、channel label 需對齊 BOM |
| 軟體 V × I | D-Bus 或 sysfs | ExternalSensor / VirtualSensor / 客製 daemon | Voltage / Current 需同量測點、同單位、同 power state |
| CPU package power | PECI / CPU telemetry / VR PMBus | IntelCPUSensor、PECI、VR hwmon | CPU package power 與 VR input/output power 定義不同 |
| GPU power | MCTP / PLDM / SMBPBI / vendor path | GPU sensor daemon / ExternalSensor | 需注意 GPU presence、driver readiness、功率限制狀態 |
| NVMe power | NVMe-MI / PLDM / vendor path | NVMe sensor daemon / ExternalSensor | 多數為裝置內部狀態或估算, 需標示來源 |
| Board total power | PSU output、HSC input、sensor 加總 | PSUSensor / VirtualSensor | 需避免 double counting, 並定義 AC input 或 DC board input |

### 24.7.4 單位與 ScaleFactor 設計原則

Power Sensor 必須先分清楚三層單位:

```text
PMBus raw word：Linear11 / Linear16 / Direct format，依 device 與 driver 轉換
Linux hwmon power*_input：通常為 microwatt（µW）
OpenBMC D-Bus Sensor.Value：Power Sensor 建議以 Watt（W）檢查
```

Linux hwmon sysfs 規範中,`power[1-*]_input` 是 instantaneous power use, power 類屬性單位通常為 microWatt. 換算如下:

```text
cat power1_input = 15000000
=> 15,000,000 µW
=> 15 W
```

常見 ScaleFactor 判斷:

```text
若 daemon 已將 hwmon µW 轉成 D-Bus W：ScaleFactor = 1.0
若 daemon 直接把 power*_input 當 D-Bus value：ScaleFactor = 0.000001
若來源是 mW：ScaleFactor = 0.001
若來源已是 W：ScaleFactor = 1.0
```

PMBus `READ_POUT` / `READ_PIN` 可能以 Linear 或 Direct 格式回傳. Linux PMBus core 與 chip-specific driver 通常會依 driver info、device capability 或 direct coefficient 轉成 hwmon 單位. 若出現固定比例誤差, 需確認:

```text
[ ] 該 device 是否由 chip-specific driver 支援，或只使用 generic pmbus
[ ] 功率 channel 是 Linear11、Linear16 還是 Direct format
[ ] driver 中 m / b / R 係數是否符合 datasheet
[ ] device NVM / register 內 POUT exponent / scale 是否與 driver 設定一致
[ ] sysfs power*_input 是 µW、mW、W 還是 raw-like value
[ ] OpenBMC daemon 是否已自行把 µW 轉成 W
```

若 voltage sensor 以 V, current sensor 以 A 發佈:

```text
Power_W = Voltage_V × Current_A
```

若底層仍是 mV 與 mA:

```text
Power_W = Voltage_mV × Current_mA / 1,000,000
```

PSU 與 VR 常同時具有 input power 與 output power:

```text
Input Power：進入 PSU / VR / HSC 的功率
Output Power：由 PSU / VR 輸出到下游負載的功率
Efficiency = Output Power / Input Power
Loss = Input Power - Output Power
```

命名時建議保留方向, 例如 `PSU0_INPUT_POWER`、`PSU0_OUTPUT_POWER`、`CPU0_VCORE_VR_INPUT_POWER`、`CPU0_VCORE_OUTPUT_POWER`. 若只記錄 `CPU0_POWER`, 後續很難判斷它代表 CPU package power、VR output power、VR input power 或 board rail 估算功率.

### 24.7.5 Porting 路徑 A: PMBus 直接讀取功率

#### Step A1: 確認硬體資訊與功率 channel

從 schematic、BOM、power tree、VR / PSU / HSC datasheet 取得:

```text
[ ] 裝置型號，例如 MP2971、TPS53679、PSU 模組、ADM127x、INA233
[ ] I2C bus number、mux channel、7-bit address
[ ] PMBus Page 對應 rail，例如 Page 0 = Vcore、Page 1 = VCCGT
[ ] PMBus Phase 是否會影響功率讀值
[ ] 目標命令：READ_POUT（0x96）或 READ_PIN（0x97）
[ ] Power format：Linear / Direct / vendor-specific
[ ] POUT / PIN scale、exponent、coefficient 或 calibration 來源
[ ] 是否需要 host power on、rail enable 或 PSU on 才更新 telemetry
[ ] 是否有 hardware averaging、update interval 或 telemetry cache
[ ] fault / warning limit：POUT_OP_WARN_LIMIT、PIN_OP_WARN_LIMIT 等
```

PMBus command 對照:

| Command | Code | 功能 | 常見 sensor 命名 |
| --- | ---: | --- | --- |
| `READ_POUT` | `0x96` | 輸出功率 | `*_OUTPUT_POWER`、`*_VCORE_POWER` |
| `READ_PIN` | `0x97` | 輸入功率 | `*_INPUT_POWER` |
| `POUT_OP_WARN_LIMIT` | `0x6A` | output power warning limit | threshold 參考 |
| `PIN_OP_WARN_LIMIT` | `0x6B` | input power warning limit | threshold 參考 |

#### Step A2: I2C / PMBus 基本通訊驗證

```bash
# 掃描指定 bus；需先確認可安全使用
$ i2cdetect -y <bus>

# 視 device 是否支援，讀 MFR_ID / MFR_MODEL
$ i2cget -y <bus> <addr> 0x99
$ i2cget -y <bus> <addr> 0x9A

# 設定 PAGE = 0
$ i2cset -y <bus> <addr> 0x00 0x00

# READ_POUT = 0x96，word read
$ i2cget -y <bus> <addr> 0x96 w

# READ_PIN = 0x97，word read
$ i2cget -y <bus> <addr> 0x97 w
```

注意事項:

```text
- i2cget ... w 顯示的 word 可能需要 byte swap 才能解讀為 PMBus raw word。
- raw word 非 0 不代表 driver conversion 一定正確，仍需比對 hwmon sysfs 與外部功率計。
- host off、VR disabled、PSU standby 或 telemetry disabled 時，READ_POUT / READ_PIN 可能回 0、NACK、0xffff 或 stale value。
- 不建議在不了解 device 行為時寫入 warning / fault limit 或 calibration register。
```

#### Step A3: Device Tree / driver binding

PMBus VR 範例:

```dts
&i2c8 {
    status = "okay";
    clock-frequency = <100000>;

    vr-controller@40 {
        compatible = "mps,mp2971";
        reg = <0x40>;
        label = "cpu0_vr";
    };
};
```

INA233 / PMBus power monitor 範例:

```dts
&i2c8 {
    status = "okay";

    power-monitor@45 {
        compatible = "ti,ina233";
        reg = <0x45>;
        label = "p12v_power_monitor";
    };
};
```

Kernel config 檢查方向:

```text
CONFIG_HWMON
CONFIG_I2C
CONFIG_PMBUS
CONFIG_SENSORS_PMBUS
CONFIG_SENSORS_INA2XX / CONFIG_SENSORS_INA233 / CONFIG_SENSORS_INA3221
對應 VR / PSU / HSC chip-specific driver，例如 MP297x、TPS536xx、ADM127x、LTC、MAX、IR 等
```

#### Step A4: 確認 driver probe 與 hwmon sysfs

```bash
$ dmesg | grep -Ei 'pmbus|mp297|tps536|ina2|ina233|adm127|ltc|psu|hwmon|i2c'

$ for h in /sys/class/hwmon/hwmon*; do
    echo "== $h =="
    cat "$h/name" 2>/dev/null || true
    for f in "$h"/power*_label "$h"/power*_input "$h"/in*_input "$h"/curr*_input; do
        [ -e "$f" ] && echo "$(basename "$f")=$(cat "$f")"
    done
$ done
```

若 `power*_label` 存在, 優先用 label 建立 mapping; 若不存在, 需依 Page、driver 文件、負載變化與 PMBus manual read 建立對照.

```text
power1_input = 15000000 → 15 W
power2_input = 0        → 0 W，可能是 rail off、Page 錯、command unsupported 或尚未更新
```

#### Step A5: 量測與對照

| 負載狀態 | sysfs `power*_input` | sysfs 換算 W | D-Bus W | 外部功率計 / PSU W | 誤差 | 備註 |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| Host off / standby | [待填] | [待填] | [待填] | [待填] | [待填] | [待填] |
| Host on idle | [待填] | [待填] | [待填] | [待填] | [待填] | [待填] |
| CPU / GPU stress | [待填] | [待填] | [待填] | [待填] | [待填] | [待填] |

若誤差固定為 1000 或 1,000,000 倍, 通常是 mW / µW / W 換算問題. 若誤差隨負載增加, 需檢查 update interval、averaging、量測點差異、PSU efficiency、VR loss 與時間同步.

### 24.7.6 Porting 路徑 B: 軟體計算功率(V × I)

#### Step B1: 確認 Voltage 與 Current Sensor 已正常運作

```bash
$ busctl tree /xyz/openbmc_project/sensors/voltage
$ busctl tree /xyz/openbmc_project/sensors/current

$ busctl get-property \
  <voltage_service> \
  /xyz/openbmc_project/sensors/voltage/CPU0_VCCIN \
  xyz.openbmc_project.Sensor.Value \
  Value

$ busctl get-property \
  <current_service> \
  /xyz/openbmc_project/sensors/current/CPU0_VCORE_CURRENT \
  xyz.openbmc_project.Sensor.Value \
  Value
```

#### Step B2: 確認量測點一致性

```text
[ ] Voltage sensor 量到的是同一 rail 的輸出電壓
[ ] Current sensor 量到的是同一 rail 的輸出電流
[ ] Current sensor 不是 upstream input current，也不是多 rail total current
[ ] Voltage / Current 皆受相同 PowerState 控制
[ ] PollInterval 接近，且負載快速變化時可接受誤差
```

常見不一致情境:

```text
V = CPU Vcore output voltage，但 I = VR input current → P 不是 CPU Vcore output power
V = PSU output 12V，但 I = board branch current → P 是 branch power，不是 PSU output power
V = nominal voltage 固定值，但 I 為即時電流 → P 為估算值，不適合高精度 power capping
```

#### Step B3: ExternalSensor / VirtualSensor 配置

```json
{
  "Name": "CPU0_VCORE_POWER",
  "Type": "ExternalSensor",
  "PollInterval": 1000,
  "PowerState": "On",
  "SensorType": "power",
  "ScaleFactor": 1.0,
  "MaxValue": 300.0,
  "MinValue": 0.0,
  "Configuration": {
    "VoltageSensor": "/xyz/openbmc_project/sensors/voltage/CPU0_VCCIN",
    "CurrentSensor": "/xyz/openbmc_project/sensors/current/CPU0_VCORE_CURRENT",
    "Formula": "voltage * current"
  },
  "Thresholds": [
    {
      "Name": "upper critical",
      "Direction": "greater than",
      "Severity": 1,
      "Value": 250.0
    },
    {
      "Name": "upper non critical",
      "Direction": "greater than",
      "Severity": 0,
      "Value": 200.0
    }
  ]
}
```

若 voltage / current 已是 D-Bus V / A,`Formula = voltage * current` 的結果就是 W. 若讀取來源是 mV / mA, 需在 formula 或 ScaleFactor 統一換算.

### 24.7.7 Entity Manager / Sensor JSON 配置

#### PMBus VR output power 範例

```json
{
  "Name": "CPU0_VCORE_POWER",
  "Type": "MP2971",
  "Bus": 8,
  "Address": "0x40",
  "Page": 0,
  "PollInterval": 500,
  "ScaleFactor": 1.0,
  "Offset": 0.0,
  "MaxValue": 300.0,
  "MinValue": 0.0,
  "PowerState": "On",
  "Thresholds": [
    {
      "Name": "upper critical",
      "Direction": "greater than",
      "Severity": 1,
      "Value": 250.0
    },
    {
      "Name": "upper non critical",
      "Direction": "greater than",
      "Severity": 0,
      "Value": 200.0
    }
  ]
}
```

#### PSU input power 範例

```json
{
  "Name": "PSU0_INPUT_POWER",
  "Type": "PSU",
  "Bus": 4,
  "Address": "0x58",
  "PollInterval": 1000,
  "ScaleFactor": 1.0,
  "MaxValue": 1000.0,
  "MinValue": 0.0,
  "PowerState": "AlwaysOn",
  "Thresholds": [
    {
      "Name": "upper critical",
      "Direction": "greater than",
      "Severity": 1,
      "Value": 900.0
    }
  ]
}
```

配置重點:

```text
- Name 必須清楚包含 input/output、rail/domain、device index。
- Type 需與專案 sensor-info / schema / daemon 支援類型一致。
- PMBus 多 Page 裝置需填 Page；若有 Phase，也需確認專案是否支援。
- ScaleFactor 需依 daemon 對 hwmon µW 的處理方式設定。
- PowerState 需符合功率來源：CPU / DIMM VR 通常 On；PSU input / standby rail 通常 AlwaysOn。
- MaxValue / threshold 應與 power budget、PSU rating、VR rating、thermal design power 對齊。
```

### 24.7.8 啟動服務與 D-Bus 驗證

```bash
$ systemctl list-units '*sensor*' --no-pager
$ systemctl list-units '*psu*' --no-pager
$ systemctl list-units '*external*' --no-pager

$ systemctl restart xyz.openbmc_project.HwmonPowerSensor.service
$ journalctl -u xyz.openbmc_project.HwmonPowerSensor.service -b --no-pager | tail -100

$ journalctl -b --no-pager | grep -Ei 'power|pout|pin|psu|external|sensor|pmbus'

$ busctl tree /xyz/openbmc_project/sensors
$ busctl tree /xyz/openbmc_project/sensors/power

$ busctl get-property \
  <service_name> \
  /xyz/openbmc_project/sensors/power/CPU0_VCORE_POWER \
  xyz.openbmc_project.Sensor.Value \
  Value

$ busctl call xyz.openbmc_project.ObjectMapper \
  /xyz/openbmc_project/object_mapper \
  xyz.openbmc_project.ObjectMapper GetObject \
  sas \
  /xyz/openbmc_project/sensors/power/CPU0_VCORE_POWER \
  0
```

建議至少比對四層:

```text
PMBus raw / V × I inputs
    ↔ hwmon sysfs power*_input 或 source sensors
        ↔ D-Bus Sensor.Value
            ↔ Redfish / IPMI / logs
```

### 24.7.9 Redfish / IPMI / Event / Power Policy 驗證

```bash
$ curl -k -u root:<password> \
  https://<bmc-ip>/redfish/v1/Chassis/<chassis>/Sensors | jq

$ curl -k -u root:<password> \
  https://<bmc-ip>/redfish/v1/Chassis/<chassis>/Sensors/CPU0_VCORE_POWER | jq

$ curl -k -u root:<password> \
  https://<bmc-ip>/redfish/v1/Chassis/<chassis>/PowerSubsystem | jq

$ ipmitool sensor | grep -Ei 'power|watt|psu'
$ ipmitool sdr elist | grep -Ei 'power|watt|psu'
```

驗證重點:

```text
[ ] Redfish sensor 名稱與 D-Bus path 對應
[ ] Reading / Value 與 D-Bus 一致
[ ] ReadingUnits 為 W 或可清楚表示 Watt
[ ] Warning / Critical threshold 顯示正確
[ ] Status.Health / State 與 D-Bus availability、operational status 一致
[ ] 若 power sensor 作為 power cap input，確認 consumer 讀取的是正確 sensor
[ ] 若 power sensor 用於 PSU redundancy，確認 PSU removal / fault / AC loss 時數值與 state 一致
[ ] 若 power sensor 用於 thermal policy，確認高負載時數值上升與 fan response 時序合理
```

Event 驗證:

```bash
$ busctl introspect <service_name> \
  /xyz/openbmc_project/sensors/power/CPU0_VCORE_POWER

$ journalctl -b --no-pager | grep -Ei 'CPU0_VCORE_POWER|threshold|critical|warning|power'

$ curl -k -u root:<password> \
  https://<bmc-ip>/redfish/v1/Systems/system/LogServices/EventLog/Entries | jq
```

### 24.7.10 進階除錯與常見陷阱

| 問題現象 | 建議排查方向 | 排查 / 處理方式 |
| :--- | :--- | :--- |
| `power*_input` 不存在 | device 不支援 `READ_POUT` / `READ_PIN`; driver 未宣告 power capability; kernel config 未啟用 | 查 datasheet、driver Kconfig、`dmesg`、PMBus func flags; 必要時使用 V × I 路徑 |
| `power*_input` 讀值為 0 | Host off、VR disabled、PSU standby、Page 錯、telemetry 未更新 | 確認 power state、enable signal、PMBus PAGE、手動讀 `READ_POUT` / `READ_PIN` |
| D-Bus sensor 未出現 | Entity Manager JSON `Type` / `Bus` / `Address` / `Page` 不符合 daemon 預期 | 查 Entity Manager log、sensor daemon log、ObjectMapper subtree |
| 讀值固定大 1,000,000 倍 | µW 被當成 W | 確認 `power*_input` 單位; 必要時 `ScaleFactor = 0.000001` |
| 讀值固定小 1,000,000 倍 | 已轉 W 後又除以 1,000,000 | 比對 sysfs、D-Bus、Redfish, 移除重複 scaling |
| 讀值固定大或小 1000 倍 | mW / W 或 µW / mW 換算錯誤 | 查 driver output、daemon conversion、JSON ScaleFactor |
| 多 Page 功率全部相同 | driver 未正確切 Page、JSON Page 未生效、讀到同一 channel | 手動設定 PAGE 後讀 `READ_POUT`; 查 driver page support |
| 軟體 V × I 與 PMBus POUT 差距大 | 量測點不同、時間不同步、V/I sensor scaling 錯、PMBus 是 averaged power | 對齊 measurement point; 比對 fixed load; 確認 update interval |
| PSU input power 與 output power 不符 | PSU efficiency、loading、AC/DC input 定義不同 | 分別命名 input/output, 使用 PSU datasheet efficiency curve 檢查 |
| board total power 加總過大 | double counting, 例如 PSU output + VR output 重複計入 | 建立 power tree, 標示每個 sensor 所在能量路徑 |
| 功率為負數 | Current sensor 為負、bidirectional sensor offset、formula sign 錯 | 回查 Current Sensor 方向與 offset; 修正 formula 或 sign |
| Redfish 顯示功率但數值不變 | daemon poll 未更新、PowerState gating、sysfs cache、bmcweb cache | 直接 `cat power*_input`, 再比對 D-Bus 與 Redfish |
| Threshold 觸發但無事件 | threshold interface 未建立、logging policy 未接、alarm flag 未變化 | 查 D-Bus threshold properties、phosphor-logging、EventLog |
| Power capping 行為異常 | consumer 使用錯 sensor 或單位錯 | 查 power cap service config、D-Bus path、Value 單位與上限設定 |

### 24.7.11 Power Sensor Porting 驗收 Checklist

硬體設計階段:

- [ ] Power source 類型確認: PMBus 直接讀取 / 軟體 V × I / CPU telemetry / GPU telemetry / NVMe / 虛擬加總
- [ ] Power tree 中量測點確認: input、output、rail、domain、package、board total
- [ ] 若為 PMBus: I2C bus、mux channel、address、Page、Phase 確認
- [ ] READ_POUT / READ_PIN 支援狀態確認
- [ ] PMBus POUT / PIN scale、exponent、coefficient 或 calibration 來源確認
- [ ] 若為 V × I: 對應 Voltage Sensor 與 Current Sensor 的量測點一致
- [ ] 功率範圍、TDP、PSU rating、VR rating、warning、critical 門檻確認
- [ ] input power / output power 命名規則確認, 避免後續判讀混淆

Device Tree / Kernel:

- [ ] I2C bus node status = "okay"
- [ ] I2C mux channel 與 bus number 對照完成
- [ ] PMBus / VR / PSU / INA / HSC device node 加入, compatible 與 reg 正確
- [ ] 必要 kernel config 已啟用, 如 PMBus、chip-specific driver、INA2xx、hwmon
- [ ] 開機後 dmesg 無 probe failure、timeout、unsupported command 相關錯誤
- [ ] /sys/class/hwmon/hwmonX/name 可對應到目標 device
- [ ] power*_input 存在, 且單位已確認為 µW 或平台定義值
- [ ] power*_label 或 Page 對照表已建立
- [ ] 若使用 V × I, voltage / current 來源 sensor 均已驗證
- [ ] 使用外部功率計、PSU telemetry 或穩定負載比對, 工程誤差在可接受範圍內

Entity Manager / Userspace:

- [ ] JSON 設定檔加入正確 layer, 並已編入 image
- [ ] Name 命名符合平台 sensor naming rule, 且清楚標示 input/output
- [ ] Type 與 daemon / sensor-info / schema 定義一致
- [ ] PMBus 架構: Bus、Address、Page、PowerState 設定正確
- [ ] V × I 架構: VoltageSensor、CurrentSensor、Formula 設定正確
- [ ] ScaleFactor 已依 sysfs 單位確認, 不重複換算 µW / mW / W
- [ ] PollInterval 符合需求, 常見建議 500～1000 ms
- [ ] PowerState 設定符合功率來源, 例如 CPU VR 使用 On、PSU input 使用 AlwaysOn
- [ ] Thresholds 與 power budget、PSU rating、VR rating、thermal policy、power cap policy 一致

D-Bus / Redfish / IPMI / Policy:

- [ ] 對應 sensor service 啟動無錯誤
- [ ] ObjectMapper 可找到 /xyz/openbmc_project/sensors/power/<Name>
- [ ] busctl get-property 可讀取 Value, 且單位以 W 檢查合理
- [ ] Warning / Critical threshold property 存在且數值正確
- [ ] Redfish Sensor 或 PowerSubsystem 依平台需求可看到該 power sensor
- [ ] IPMI SDR / sensor list 依平台需求可看到該 sensor
- [ ] 施加負載變化時, sysfs、D-Bus、Redfish 數值同步變化
- [ ] Threshold 觸發時, D-Bus alarm flag、phosphor-logging、SEL / EventLog 行為符合預期
- [ ] 若用於 power capping / thermal control / PSU redundancy, consumer 讀取 path 與單位均已確認
- [ ] 若為 board total power, 已確認沒有重複計入同一能量路徑

### 24.7.12 Power Sensor 資料表範本

| Sensor Name | Source Type | Device | Bus / Addr | Page / Channel | Raw sysfs / Source | sysfs unit | ScaleFactor | D-Bus unit | PowerState | Threshold | 備註 |
| --- | --- | --- | --- | --- | --- | --- | ---: | --- | --- | --- | --- |
| CPU0_VCORE_POWER | PMBus VR | MP2971 | 8 / 0x40 | Page 0 | power1_input | µW | 1.0 或 0.000001 | W | On | [待填] | READ_POUT |
| PSU0_INPUT_POWER | PMBus PSU | PSU0 | 4 / 0x58 | input | power1_input | µW | [待填] | W | AlwaysOn | [待填] | READ_PIN |
| PSU0_OUTPUT_POWER | PMBus PSU | PSU0 | 4 / 0x58 | output | power2_input | µW | [待填] | W | AlwaysOn | [待填] | READ_POUT |
| CPU0_VCORE_POWER_EST | V × I | ExternalSensor | N/A | Vcore | voltage × current | V × A | 1.0 | W | On | [待填] | software calculated |
| BOARD_INPUT_POWER | HSC | ADM127x | [待填] | input | power1_input | µW | [待填] | W | AlwaysOn | [待填] | board inlet |

### 24.7.13 本節參考資料

- Linux kernel hwmon sysfs interface:`power[1-*]_input` 為 instantaneous power use, power 類屬性單位為 microWatt; hwmon sysfs 使用 fixed-point single-value 檔案格式.
- Linux kernel PMBus driver documentation: PMBus driver 支援 voltage、current、power、temperature 等 sensor, 且 PMBus device 通常需明確建立, 不依賴安全自動 probe.
- Linux PMBus register definitions:`READ_POUT = 0x96`、`READ_PIN = 0x97`, 另有 `POUT_OP_WARN_LIMIT = 0x6A` 與 `PIN_OP_WARN_LIMIT = 0x6B` 可作 threshold 設計參考.
- OpenBMC dbus-sensors README: sensor daemon 可從 hwmon、D-Bus 或 direct driver access 讀取資料, 並以 `/xyz/openbmc_project/sensors/<type>/<sensor_name>` 發佈 D-Bus sensor object.

<a id="section-12-8"></a>

## 24.8 Fan Tach Sensor

### 24.8.1 適用情境

Fan Tach Sensor(風扇轉速感測器)用於讀取風扇實際轉速, 單位通常是 RPM(Revolutions Per Minute, 每分鐘轉數). 在 BMC 平台中, fan tach 是風扇控制迴路的回授訊號, 用來確認 PWM 輸出後風扇是否確實轉到目標轉速, 也用來偵測停轉、堵轉、轉速過低、風扇拔除、雙轉子風扇單 rotor 失效, 以及風扇冗餘不足.

常見監控對象包含:

```text
System inlet fan：系統進風扇
System exhaust fan：系統出風扇
CPU heatsink fan：CPU 散熱風扇
PSU fan：電源供應器內建風扇
GPU fan：GPU / accelerator 散熱風扇
HDD / backplane fan：硬碟背板或風扇牆風扇
Fan module rotor：風扇模組內的單一 rotor，例如雙轉子風扇的 front / rear rotor
```

在 OpenBMC 中, fan tach sensor 常見 D-Bus path 為:

```text
/xyz/openbmc_project/sensors/fan_tach/<sensor_name>
```

`fan_tach` 的 `Value` 應以 RPM 檢查. 若底層 Linux hwmon `fan*_input` 已輸出 RPM, 通常不需要比例換算. 若數值與外部轉速計呈現 2 倍、1/2、4 倍或 1/4 倍關係, 優先檢查每轉脈衝數(Pulses Per Revolution, PPR)、tach divisor 與 channel 對應.

### 24.8.2 資料路徑

```mermaid
flowchart LR
    PWM[Fan PWM output] --> MOTOR[Fan motor]
    MOTOR --> TACH[Tach pulses]
    TACH --> FRONT[Pull-up / buffer / CPLD]
    FRONT --> CTRL[BMC tach controller]
    CTRL --> HWMON[fan*_input RPM]
    HWMON --> DAEMON[FanSensor]
    DAEMON --> DBUS[D-Bus fan_tach]
    DBUS --> MON[Presence / Fan monitor]
    DBUS --> PID[PID / Fan control]
    PID --> PWM
    MON --> FAILSAFE[Failsafe policy]
    FAILSAFE --> PWM
```

Fan Tach Sensor 的完整資料路徑如下:

```text
風扇本體產生 Tachometer 脈衝訊號
    ↓
風扇連接器 Tach pin 經 pull-up / buffer / mux / CPLD 接到 BMC Tach input
    ↓
BMC Tachometer controller 計數脈衝或量測週期
    ↓
Linux kernel driver 依 PPR、clock、divisor、sample window 換算 RPM
    ↓
hwmon sysfs：/sys/class/hwmon/hwmonX/fan*_input
    ↓
FanSensor / PSUSensor / 平台 sensor daemon 讀取 sysfs
    ↓
D-Bus：/xyz/openbmc_project/sensors/fan_tach/<Name>
    ↓
phosphor-fan-presence / phosphor-fan-monitor / phosphor-pid-control / bmcweb / IPMI SDR
```

PWM 與 Tach 需分開看:

```text
PWM：BMC 輸出控制訊號，常見 25 kHz，用來控制風扇轉速
Tach：風扇輸出回授訊號，常見 open-drain，每轉 1～4 個脈衝
```

ASPEED G6 / AST2600 的 PWM controller 與 Fan Tacho controller 是獨立硬體區塊, 常見 binding 中也將 PWM outputs 與 fan tach inputs 分開描述. 因此 Device Tree 需同時確認 PWM pin、Tach pin、fan child node 與 channel 對應.

### 24.8.3 關鍵硬體參數

Porting 前需從 schematic、BOM、風扇規格書、CPLD register map 與 board bring-up 記錄取得下列資料:

```text
[ ] 風扇型號、供應商、料號、風扇規格書版本
[ ] 風扇線材定義：GND、Power、Tach、PWM、Presence、Fault、FRU EEPROM
[ ] Tach 電氣型態：open-drain / open-collector / push-pull
[ ] Tach pull-up 電壓與阻值：BMC rail、fan rail、CPLD rail，是否有 level shift
[ ] 每轉脈衝數 PPR：常見為 2，也可能為 1、4 或 vendor-specific
[ ] Tach channel：連到 BMC Tach0 / Tach1 / ... 或經 CPLD / mux 後的 channel
[ ] PWM channel：控制該風扇的 PWM0 / PWM1 / ...
[ ] PWM 頻率需求：常見 25 kHz，但需以風扇規格書為準
[ ] PWM polarity：正常 duty 定義、是否反相、是否需要 pull-up
[ ] 最小啟動 duty / start boost：低速啟動失敗時需先給較高 PWM
[ ] Min RPM / Max RPM / Rated RPM / RPM tolerance
[ ] Lower warning / lower critical threshold
[ ] 風扇存在偵測：無、Tach-based、GPIO presence、CPLD presence、FRU EEPROM
[ ] 單顆風扇是否多 rotor：一個 PWM 可能對應兩個 tach feedback
[ ] 風扇模組對應 inventory path、FRU path 與 LED fault indicator
```

Tach 訊號頻率與 RPM 的關係:

```text
Tach_Frequency_Hz = RPM / 60 × PPR
RPM = Tach_Frequency_Hz × 60 / PPR
```

例: 風扇 6000 RPM、PPR = 2:

```text
Tach_Frequency = 6000 / 60 × 2 = 200 Hz
```

若示波器量到 200 Hz, 但 sysfs 顯示 12000 RPM, 常見方向是 PPR 設成 1; 若 sysfs 顯示 3000 RPM, 常見方向是 PPR 設成 4, 或 driver / DTS 對 PPR 的欄位沒有生效.

Tach 線常見為 open-drain / open-collector, 需要外部 pull-up. 設計與排查時需確認:

```text
[ ] Pull-up 電壓是否符合 BMC input absolute maximum rating
[ ] 若 fan tach 為 5V pull-up，是否有 level shifter 或 BMC pin 可容忍 5V
[ ] Pull-up 阻值是否造成上升時間過慢，尤其高轉速與長線材場景
[ ] 是否經 CPLD / buffer / mux，且該路徑在 BMC 讀取前已 enable
[ ] Tach pin 是否與其他 multi-function pin 或 strap function 衝突
[ ] 風扇拔除時 tach line 是否浮動，是否需 board-side pull-up / pull-down
```

### 24.8.4 常見來源分類與特性

| 來源類型 | 底層來源 | OpenBMC 常見服務 | 注意事項 |
| :--- | :--- | :--- | :--- |
| BMC SoC tach controller | BMC Tach input → hwmon `fan*_input` | FanSensor | 需對齊 pinctrl、tach channel、PPR、PWM 對應 |
| generic `pwm-fan` driver | PWM + 可選 tach interrupt | FanSensor / hwmon | `fan1_input` 為 RPM; PWM 以 `pwm1` 0～255 控制 |
| ASPEED G6 PWM/Tach driver | `aspeed, ast2600-pwm-tach` | FanSensor | PWM 與 Tach 為獨立硬體; fan child node 需含 `tach-ch` |
| Nuvoton NPCM PWM/Fan | NPCM PWM / fan tach controller | FanSensor | 需確認 SoC binding 的 fan child node 與 channel 編號 |
| PSU 內建風扇 | PMBus / PSU hwmon `fan*_input` | PSUSensor | fan path 可能由 PSU service 建立, 不一定是 FanSensor |
| Fan controller IC | I2C fan controller, 例如 MAX / NCT / EMC | hwmon / FanSensor | 通常有 fan divisor、target RPM、PWM、fault register |
| CPLD / FPGA 計數 | CPLD register / I2C / LPC / mailbox | 客製 daemon / ExternalSensor | 需定義 register update rate、scale、timeout、stale condition |
| GPU / accelerator 風扇 | vendor telemetry | GPU daemon / ExternalSensor | 需區分 GPU internal fan 與 chassis fan |

### 24.8.5 Porting 步驟 A: Device Tree / Kernel

#### Step A1: 確認 SoC PWM/Tach controller node

AST2600 常見控制器節點形式如下, 實際名稱與位置以專案 DTS / dtsi 為準:

```dts
&pwm_tach {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_pwm0_default &pinctrl_pwm1_default
                 &pinctrl_tach0_default &pinctrl_tach1_default>;
};
```

若使用 upstream ASPEED G6 PWM/Tach binding, fan child node 常見格式如下:

```dts
&pwm_tach {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_pwm0_default &pinctrl_pwm1_default
                 &pinctrl_tach0_default &pinctrl_tach1_default
                 &pinctrl_tach2_default>;

    fan-0 {
        tach-ch = /bits/ 8 <0x0>;
        pwms = <&pwm_tach 0 40000 0>;
    };

    fan-1 {
        tach-ch = /bits/ 8 <0x1 0x2>;
        pwms = <&pwm_tach 1 40000 0>;
    };
};
```

說明:

```text
pwms = <&pwm_tach 0 40000 0>
    0      ：PWM channel 0
    40000  ：period = 40000 ns = 25 kHz
    0      ：PWM flags，是否反相需依 binding / driver 定義

tach-ch = /bits/ 8 <0x1 0x2>
    表示該 fan node 有兩個 tach input，常見於雙轉子風扇或雙回授線
```

#### Step A2: generic `pwm-fan` driver 場景

若平台使用 generic `pwm-fan`, 常見 DTS 如下:

```dts
fan0: pwm-fan {
    compatible = "pwm-fan";
    pwms = <&pwm 0 40000 0>;
    interrupts-extended = <&gpio5 1 IRQ_TYPE_EDGE_FALLING>;
    pulses-per-revolution = <2>;
    cooling-levels = <0 80 120 160 200 255>;
    #cooling-cells = <2>;
};
```

`pwm-fan` binding 使用 `pulses-per-revolution` 描述每轉脈衝數, 預設常見為 2. 若專案使用 vendor binding 或舊版 ASPEED driver, 可能使用 `fan-ppr`、`aspeed, fan-ppr` 或其他欄位; 請以 kernel binding 與 driver source 為準.

#### Step A3: Kernel config

確認必要 kernel config 已啟用:

```text
CONFIG_HWMON
CONFIG_PWM
CONFIG_SENSORS_PWM_FAN
CONFIG_PWM_ASPEED 或 SoC 對應 PWM driver
CONFIG_SENSORS_ASPEED_G6_PWM_TACH 或 vendor BSP 對應 ASPEED fan tach driver
CONFIG_GPIOLIB / CONFIG_GPIO_CDEV（若 presence 使用 GPIO）
CONFIG_THERMAL（若 fan 同時作為 cooling device）
```

實際 symbol 名稱會隨 kernel branch、OpenBMC branch 與 vendor BSP 改變, 請以 `bitbake -c menuconfig virtual/kernel`、`tmp/work/.../.config`、Kconfig 與 `dmesg` 為準.

#### Step A4: 開機 probe 與 pinctrl 檢查

```bash
$ dmesg | grep -Ei 'pwm|tach|fan|hwmon|aspeed|npcm'

$ cat /sys/kernel/debug/pinctrl/*/pinmux-pins 2>/dev/null | grep -Ei 'pwm|tach|fan'

$ for h in /sys/class/hwmon/hwmon*; do
    echo "== $h =="
    cat "$h/name" 2>/dev/null || true
    for f in "$h"/fan*_label "$h"/fan*_input "$h"/fan*_min "$h"/fan*_max "$h"/pwm*; do
        [ -e "$f" ] && echo "$(basename "$f")=$(cat "$f")"
    done
$ done
```

### 24.8.6 Porting 步驟 B: sysfs 與硬體訊號驗證

#### Step B1: 確認 hwmon fan input

```bash
$ find /sys/class/hwmon -maxdepth 2 -name 'fan*_input' -print
$ cat /sys/class/hwmon/hwmonX/fan1_input
```

預期輸出:

```text
3000
```

代表目前風扇轉速約 3000 RPM.`fan*_input` 若來自標準 hwmon 介面, 通常就是 RPM.

#### Step B2: 手動調整 PWM 並觀察 RPM

若 driver 提供 `pwm1`:

```bash
$ cat /sys/class/hwmon/hwmonX/pwm1_enable 2>/dev/null || true

# 依 driver 支援情況切到 manual；需在實驗室與安全負載下執行
$ echo 1 > /sys/class/hwmon/hwmonX/pwm1_enable

$ for v in 80 120 160 200 255; do
    echo $v > /sys/class/hwmon/hwmonX/pwm1
    sleep 5
    echo "pwm=$v rpm=$(cat /sys/class/hwmon/hwmonX/fan1_input)"
$ done
```

驗證重點:

```text
[ ] PWM 提高時 RPM 應跟著上升
[ ] PWM 降低時 RPM 應跟著下降，但會有機械慣性延遲
[ ] PWM = 0 時風扇是否停止，取決於 pwm1_enable mode、風扇規格與 board power design
[ ] 某些風扇低 duty 無法啟動，需要 start boost 或最小 duty
```

#### Step B3: 示波器 / 邏輯分析儀量測 Tach

量測點建議:

```text
1. 風扇連接器 Tach pin
2. pull-up 後的 board-side tach node
3. buffer / CPLD / level shifter 後的 BMC-side tach node
4. BMC pin 附近，若 layout 允許
```

量測項目:

```text
[ ] Tach 高低電位是否符合 BMC input 規格
[ ] 是否有脈衝，頻率是否符合 RPM / 60 × PPR
[ ] 上升 / 下降時間是否過慢，是否有 ringing 或毛刺
[ ] 風扇拔除時 tach line 是否固定在預期狀態
[ ] PWM 改變後 tach frequency 是否同步變化
```

### 24.8.7 Entity Manager / dbus-sensors 配置

OpenBMC 不同 branch 的 FanSensor schema 可能不同, 以下以常見概念說明; 實際欄位需以 `entity-manager` schema、平台既有 JSON、FanSensor source 與 journal log 為準.

#### 方式一: 完整 Fan 裝置定義

```json
{
  "Name": "Fan0",
  "Type": "Fan",
  "Pwm": 0,
  "Tachs": [
    {
      "Name": "Fan0_Tach",
      "Index": 0
    }
  ],
  "MaxPwm": 255,
  "MinPwm": 30,
  "PollInterval": 1000,
  "PowerState": "AlwaysOn",
  "Thresholds": [
    {
      "Name": "lower critical",
      "Direction": "less than",
      "Severity": 1,
      "Value": 500
    },
    {
      "Name": "lower non critical",
      "Direction": "less than",
      "Severity": 0,
      "Value": 1000
    }
  ]
}
```

#### 方式二: 雙轉子風扇

```json
{
  "Name": "Fan0",
  "Type": "Fan",
  "Pwm": 0,
  "Tachs": [
    {
      "Name": "Fan0_Front_Rotor",
      "Index": 0
    },
    {
      "Name": "Fan0_Rear_Rotor",
      "Index": 1
    }
  ],
  "MaxPwm": 255,
  "MinPwm": 40,
  "PowerState": "AlwaysOn",
  "Thresholds": [
    {
      "Name": "lower critical",
      "Direction": "less than",
      "Severity": 1,
      "Value": 800
    }
  ]
}
```

#### 方式三: 獨立 Tach sensor

```json
{
  "Name": "Fan0_Tach",
  "Type": "Tach",
  "Index": 0,
  "PollInterval": 1000,
  "MaxValue": 30000,
  "MinValue": 0,
  "PowerState": "AlwaysOn",
  "Thresholds": [
    {
      "Name": "lower critical",
      "Direction": "less than",
      "Severity": 1,
      "Value": 500
    }
  ]
}
```

欄位檢查重點:

```text
[ ] Name：需符合平台命名規則，避免空白、特殊字元與重名
[ ] Type：需與 FanSensor / entity-manager schema 支援的 Type 一致
[ ] Pwm：需對應控制該風扇的 PWM channel
[ ] Tachs / Index：需對應 hwmon fanN_input 或 driver channel mapping
[ ] Thresholds：風扇 tach 通常以 lower threshold 為主
[ ] PowerState：系統風扇多為 AlwaysOn；host-only 風扇可能需依 chassis / host state gating
[ ] PollInterval：建議 500～1000 ms，需與控制迴路反應時間協調
```

### 24.8.8 Fan Presence / Fan Monitor / PID Control 整合

`phosphor-fan-presence` 可用於更新 fan inventory object 上的 `xyz.openbmc_project.Inventory.Item.Present`. 常見偵測方式包含:

```text
Tach-based：依 fan_tach sensor 是否存在且 RPM 合理判斷
GPIO-based：依 presence pin、CPLD GPIO 或 connector detect 判斷
Fallback / mixed：多種方式擇一或組合判斷
```

現代 phosphor-fan-presence 常見 runtime JSON 設定檔位置包含:

```text
/usr/share/phosphor-fan-presence/presence/config.json
/etc/phosphor-fan-presence/presence/config.json   # 測試覆寫用
/usr/share/phosphor-fan-presence/presence/<compatible-name>/config.json
```

概念例:

```json
[
  {
    "name": "Fan0",
    "path": "/system/chassis/motherboard/fan0",
    "methods": [
      {
        "type": "tach",
        "sensors": ["Fan0_Tach"]
      }
    ]
  },
  {
    "name": "Fan1",
    "path": "/system/chassis/motherboard/fan1",
    "methods": [
      {
        "type": "gpio",
        "key": 123,
        "physpath": "/sys/devices/platform/fan1_presence"
      }
    ]
  }
]
```

Fan monitor 常用於判斷 fan functional 狀態, 例如 fan presence = true 但 tach = 0、目標 PWM 很高但 RPM 無法上升、雙轉子風扇其中一顆 rotor 失效. Presence 與 Functional 應分開定義: absent 不等同 failed, present 也不代表 rotor 正常.

`phosphor-pid-control` 或平台風扇控制服務會讀取溫度 sensor 與 fan tach sensor, 再輸出 PWM target. 整合時需確認 PWM sensor path 與 fan tach sensor path 對應正確、zone 設定包含該風扇、fan fail policy 已定義, 例如任一 rotor fail 時進入 failsafe PWM.

### 24.8.9 啟動服務與 D-Bus 驗證

先找出實際服務名稱:

```bash
$ systemctl list-units '*Fan*' --no-pager
$ systemctl list-units '*fan*' --no-pager
$ systemctl list-units '*sensor*' --no-pager
```

重啟與看 log:

```bash
$ systemctl restart xyz.openbmc_project.FanSensor.service
$ journalctl -u xyz.openbmc_project.FanSensor.service -b --no-pager | tail -100
$ journalctl -b --no-pager | grep -Ei 'fansensor|fan tach|fan_tach|tach|pwm|presence|fan-monitor'
```

確認 D-Bus object:

```bash
$ busctl tree xyz.openbmc_project.FanSensor
$ busctl tree /xyz/openbmc_project/sensors/fan_tach
```

讀取 RPM:

```bash
$ busctl get-property \
  xyz.openbmc_project.FanSensor \
  /xyz/openbmc_project/sensors/fan_tach/Fan0_Tach \
  xyz.openbmc_project.Sensor.Value \
  Value
```

若 service owner 不確定:

```bash
$ busctl call xyz.openbmc_project.ObjectMapper \
  /xyz/openbmc_project/object_mapper \
  xyz.openbmc_project.ObjectMapper GetObject \
  sas \
  /xyz/openbmc_project/sensors/fan_tach/Fan0_Tach \
  0
```

確認 inventory presence:

```bash
$ busctl get-property \
  xyz.openbmc_project.Inventory.Manager \
  /system/chassis/motherboard/fan0 \
  xyz.openbmc_project.Inventory.Item \
  Present
```

### 24.8.10 Redfish / IPMI / Event 驗證

```bash
$ curl -k -u root:<password> \
  https://<bmc-ip>/redfish/v1/Chassis/<chassis>/Sensors | jq

$ curl -k -u root:<password> \
  https://<bmc-ip>/redfish/v1/Chassis/<chassis>/Sensors/Fan0_Tach | jq

$ curl -k -u root:<password> \
  https://<bmc-ip>/redfish/v1/Chassis/<chassis>/Thermal | jq

$ ipmitool sensor | grep -Ei 'fan|tach|rpm'
$ ipmitool sdr elist | grep -Ei 'fan|tach|rpm'
```

Event / alarm:

```bash
$ journalctl -b --no-pager | grep -Ei 'Fan0_Tach|fan|tach|threshold|critical|warning'

$ curl -k -u root:<password> \
  https://<bmc-ip>/redfish/v1/Systems/system/LogServices/EventLog/Entries | jq
```

驗證重點:

```text
[ ] Redfish sensor Reading 與 D-Bus Value 一致
[ ] ReadingUnits 為 RPM 或可清楚表示 fan speed
[ ] IPMI sensor type、unit 與 threshold 合理
[ ] fan absent 時 Redfish / IPMI 狀態符合平台政策
[ ] lower critical 觸發時 D-Bus alarm flag 與事件 log 同步產生
```

### 24.8.11 進階除錯與常見陷阱

| 問題現象 | 建議排查方向 | 排查 / 處理方式 |
| :--- | :--- | :--- |
| `fan*_input` 不存在 | DTS 未啟用、driver 未載入、hwmon device 未建立、tach channel 未宣告 | 查 `dmesg`、kernel config、pinctrl、DTS compatible、fan child node |
| `fan*_input` 為 0 | 風扇未轉、PWM = 0、風扇沒供電、tach pin 接錯、PPR / divisor 不合理 | 先手動提高 PWM, 確認風扇供電, 再用示波器看 tach |
| RPM 為預期 2 倍或 1/2 | PPR 設錯 | 用示波器算頻率, 套 `RPM = Hz × 60 /PPR` 回推 |
| RPM 偶發跳到極大值 | tach 毛刺、pull-up 過弱或過強、線材雜訊、divisor 太小 | 檢查波形、調整 fan divisor / sample、改善走線與濾波 |
| RPM 反應很慢 | sample window 太長、daemon poll 太慢、風扇慣性 | 區分 driver 更新率與 D-Bus poll interval, 調整 PollInterval |
| PWM 改變但 RPM 不變 | PWM channel 對錯風扇、風扇固定全速、PWM polarity 錯、fan controller 接管 | 比對 schematic, 逐一改 PWM 並觀察對應風扇 |
| 某一顆雙轉子風扇只看到一個 tach | DTS 只宣告一個 tach channel、Tachs JSON 只填一個 Index | 檢查 fan module pinout、tach-ch list、Entity Manager Tachs |
| fan_tach D-Bus object 未出現 | Entity Manager Type / Index 不符合 FanSensor 預期 | 查 FanSensor log、Entity Manager log、ObjectMapper |
| Presence=false 但 RPM 有值 | presence path / config 錯、GPIO polarity 錯、inventory path 不一致 | 比對 presence config sensor name、GPIO active state、inventory path |
| Presence=true 但 RPM=0 | 風扇插入但停止、tach pin 斷線、presence pin 無法代表 rotor OK | 區分 presence 與 functional, 讓 monitor 負責 rotor fail |
| Redfish 顯示 N/A | bmcweb sensor mapping、association 或 D-Bus path 不完整 | 確認 sensor path、association、bmcweb log |
| 低轉速門檻誤觸發 | boot 初期風扇尚未啟動、host off policy、MinPwm 太低 | 加入 power state gating、啟動延遲、合理 MinPwm 與 hysteresis |
| Fan fail 後沒有 failsafe | fan monitor / PID policy 未接到該 sensor | 查 PID zone、fan monitor config、failsafe event |

### 24.8.12 Fan Tach Sensor Porting 驗收 Checklist

硬體設計階段:

- [ ] 風扇型號與規格書確認: PPR、Max RPM、Min RPM、PWM 頻率、PWM duty 定義
- [ ] Tach 電氣型態確認: open-drain / push-pull、pull-up 電壓、pull-up 阻值
- [ ] Tach pin 到 BMC Tach channel 的 schematic 對照確認
- [ ] PWM pin 到風扇 PWM input 的 schematic 對照確認
- [ ] 是否經 CPLD / buffer / mux / level shifter, enable 條件已確認
- [ ] Presence pin / Fault pin / FRU EEPROM 是否存在及其 owner 確認
- [ ] 單風扇或雙轉子風扇、tach 數量確認
- [ ] Lower warning / lower critical RPM 門檻與 thermal policy 確認

Device Tree / Kernel:

- [ ] PWM/Tach controller node status = "okay"
- [ ] compatible、reg、clocks、resets、#pwm-cells 與 SoC binding 對齊
- [ ] pinctrl 包含所有使用的 PWM 與 Tach pin, 且無 pinmux 衝突
- [ ] fan child node 已加入, pwms 與 tach-ch 正確
- [ ] PPR 欄位已依 driver binding 設定, 例如 pulses-per-revolution / fan-ppr
- [ ] 必要 kernel config 啟用: HWMON、PWM、pwm-fan、SoC PWM/Tach driver
- [ ] dmesg 無 probe failure、pinctrl failure、invalid channel 相關錯誤
- [ ] /sys/class/hwmon/hwmonX/fan*_input 存在
- [ ] 手動調整 PWM 時 RPM 會合理變化
- [ ] 使用示波器或轉速計比對 RPM, 誤差在工程可接受範圍內

Entity Manager / Userspace:

- [ ] Fan / Tach JSON 已加入正確 layer 並進 image
- [ ] Type 與 FanSensor 支援 schema 一致
- [ ] Pwm、Tachs、Index 與 DTS / hwmon channel mapping 一致
- [ ] 雙轉子風扇已建立兩個 tach sensor 或符合平台命名規則
- [ ] PollInterval 設定合理, 常見建議 500～1000 ms
- [ ] Thresholds 設定完成, 以 lower warning / lower critical 為主
- [ ] PowerState gating 符合平台狀態, 例如 AlwaysOn、On、ChassisOn
- [ ] phosphor-fan-presence config 中 sensor 名稱與 D-Bus sensor name 一致
- [ ] fan monitor / PID config 有納入該 fan tach sensor

D-Bus / Redfish / IPMI / Event:

- [ ] FanSensor service 啟動無錯誤
- [ ] ObjectMapper 可找到 /xyz/openbmc_project/sensors/fan_tach/<Name>
- [ ] busctl get-property 可讀取 Value, 單位以 RPM 檢查合理
- [ ] Threshold interfaces 與 alarm properties 存在
- [ ] Redfish Sensors 或 Thermal 可查到該 fan tach sensor
- [ ] IPMI SDR / sensor list 依平台需求可查到 fan sensor
- [ ] 手動降低 PWM 或停止風扇時, RPM 會下降並觸發 lower threshold
- [ ] threshold 觸發時, D-Bus alarm flag、phosphor-logging、SEL / EventLog 符合預期
- [ ] presence 狀態正確反映風扇插入 / 拔除
- [ ] fan fail 時 failsafe PWM / thermal policy / shutdown policy 符合平台需求

### 24.8.13 Fan Tach Sensor 資料表範本

| Fan Name | Rotor / Tach Name | PWM Channel | Tach Channel | PPR | hwmon input | D-Bus Path | Max RPM | Lower Warning | Lower Critical | Presence Method | 備註 |
| --- | --- | ---: | ---: | ---: | --- | --- | ---: | ---: | ---: | --- | --- |
| Fan0 | Fan0_Tach | 0 | 0 | 2 | fan1_input | /xyz/openbmc_project/sensors/fan_tach/Fan0_Tach | [待填] | [待填] | [待填] | Tach / GPIO | [待填] |
| Fan1 | Fan1_Front_Rotor | 1 | 1 | 2 | fan2_input | /xyz/openbmc_project/sensors/fan_tach/Fan1_Front_Rotor | [待填] | [待填] | [待填] | Tach / GPIO | dual rotor |
| Fan1 | Fan1_Rear_Rotor | 1 | 2 | 2 | fan3_input | /xyz/openbmc_project/sensors/fan_tach/Fan1_Rear_Rotor | [待填] | [待填] | [待填] | Tach / GPIO | dual rotor |

### 24.8.14 本節參考資料

- Linux kernel `pwm-fan` 文件:`pwm-fan` 以 generic PWM interface 驅動風扇, 透過 hwmon sysfs 暴露 `fan1_input`、`pwm1_enable` 與 `pwm1`;`fan1_input` 代表 fan tachometer speed in RPM.
- Linux kernel `pwm-fan` Device Tree binding:`compatible = "pwm-fan"`,`pwms` 為必要欄位;`pulses-per-revolution` 用來定義每轉脈衝數, 範圍 1～4, 預設為 2.
- Linux kernel ASPEED G6 PWM/Tach binding: AST2600 PWM controller 可支援最多 16 個 PWM outputs, Fan Tacho controller 可支援最多 16 個 fan tach input, 兩者為獨立硬體區塊; fan child node 需描述 `tach-ch` 與 `pwms`.
- OpenBMC dbus-sensors README: sensor daemon 可從 hwmon、D-Bus 或 direct driver access 讀取感測資料, 並以 `/xyz/openbmc_project/sensors/<type>/<sensor_name>` 發佈 D-Bus object.
- OpenBMC phosphor-fan-presence 文件: fan presence detection 透過設定檔更新 fan inventory object 的 `Present` property; 目前常見 runtime JSON 設定檔位置包含 `/usr/share/phosphor-fan-presence/presence/config.json` 與 `/etc/phosphor-fan-presence/presence/config.json`.

<a id="section-12-9"></a>

## 24.9 Fan PWM / Fan Control

### 24.9.1 適用情境

Fan PWM(Pulse Width Modulation)用於控制散熱風扇的轉速, 是 OpenBMC 熱管理系統中的輸出端. Fan Tach Sensor 負責回報實際 RPM, Fan PWM 則負責把控制器計算出的目標 duty 或目標轉速轉成硬體輸出訊號. 兩者常一起出現在同一個風扇模組, 但在 bring-up、DTS、JSON 與除錯時必須分開檢查.

常見應用場景包含:

```text
依溫度感測器動態調整風扇轉速
工廠測試、維修模式、熱流測試時固定 PWM
Thermal Policy / PID / Stepwise 風扇控制
風扇失效或溫度 sensor 失效時進入 failsafe PWM
開機預設轉速設定，避免 BMC 服務尚未啟動前風扇停止
風扇冗餘管理，例如單一風扇失效時提高同 zone 其他風扇轉速
電源狀態切換，例如 host off / standby / chassis on 的不同風扇策略
聲學目標與功耗最佳化，例如 idle 低轉速、stress 提高轉速
```

在 OpenBMC 中, PWM 不一定只以一般 numeric sensor 呈現. 常見呈現方式包含:

```text
/xyz/openbmc_project/sensors/fan_pwm/<Name>
/xyz/openbmc_project/control/fanpwm/<Name>
/xyz/openbmc_project/sensors/fan_tach/<Name> 上的 Control.FanPwm interface
```

實際 path 與 interface 會受 OpenBMC branch、dbus-sensors、phosphor-hwmon、phosphor-fan-control、phosphor-pid-control 與平台 JSON/YAML 設定影響. Porting 時建議不要只記 `Fan0`, 而是同時記錄:

```text
Fan inventory path
Fan Tach D-Bus path
Fan PWM D-Bus path
sysfs pwmN path
sysfs fanN_input path
控制服務名稱與 zone id
```

### 24.9.2 資料路徑

```mermaid
flowchart LR
    S[Temperature / Power / Margin sensors] --> C{Control policy}
    C -->|PID| PID[Continuous target]
    C -->|Stepwise| STEP[Zone table]
    C -->|Fault / unavailable| FS[Failsafe target]
    PID --> ARB[Max / priority arbitration]
    STEP --> ARB
    FS --> ARB
    ARB --> D[D-Bus FanPwm / FanSpeed]
    D --> E[FanSensor / hwmon]
    E --> F[pwmN]
    F --> G[Fan]
    G --> H[fanN_input RPM]
    H -. feedback .-> C
```

Fan PWM / Fan Control 的常見資料流如下:

```text
Temperature / Margin / Power / Host telemetry sensors
    ↓
phosphor-pid-control / phosphor-fan-control / platform fan control daemon
    ↓
Zone policy：PID、Stepwise、fixed table、failsafe、manual override
    ↓
D-Bus control interface：Control.FanPwm 或 Control.FanSpeed
    ↓
FanSensor / PwmSensor / phosphor-hwmon / platform daemon
    ↓
sysfs：/sys/class/hwmon/hwmonX/pwmN 或 driver-specific pwm path
    ↓
Linux PWM driver：pwm-fan、ASPEED PWM/Tach、SoC PWM、I2C fan controller
    ↓
PWM pin / fan controller IC / power stage
    ↓
Fan motor speed changes
    ↓
Fan Tach 回授 RPM
```

依控制方式可分為兩種:

```text
PWM target control：控制器直接輸出 0～255 或 0～100% 類型的 PWM 值
RPM target control：控制器輸出目標 RPM，下一層 fan controller 再調整 PWM 以追 RPM
```

PWM target control 較直觀, 常見於 BMC 直接接風扇 PWM 線的系統. RPM target control 則適合有硬體 fan controller IC 或平台軟體已建立內層 fan PID 的系統. 兩者不可混用: 若上層輸出 6000(原意 RPM)但下層當成 PWM raw value, 通常會被 clamp 到最大值; 若上層輸出 128(原意 PWM)但下層當成 RPM, 風扇可能維持極低轉速或被判定異常.

### 24.9.3 關鍵硬體參數

Porting 前需從 schematic、layout、風扇規格書、CPLD register map、power sequence 與 thermal requirement 取得下列資訊:

```text
[ ] 風扇型號、料號、風扇規格書版本
[ ] PWM 訊號電壓準位：3.3V / 5V / open-drain / push-pull / level shifted
[ ] PWM input 是否需要 pull-up、pull-down 或 series resistor
[ ] PWM 頻率範圍：常見 25 kHz，但需以風扇規格書為準
[ ] PWM duty range：0～100%、20～100%、30～100% 或 vendor-defined
[ ] PWM polarity：高 duty 轉速增加 / 反相 / 訊號斷線時全速
[ ] 最小啟動 duty：風扇從停止到開始轉動所需 duty
[ ] 最小穩定 duty：風扇已轉動後可維持不停止的 duty
[ ] 全速 duty：通常 100% 或 raw 255
[ ] 停止 duty：0% 是否真的停止，或風扇是否有內建最低轉速
[ ] PWM channel：PWM0 / PWM1 / ... 到 fan connector 的對照
[ ] Tach channel：用於驗證 PWM 效果的回授 sensor
[ ] 風扇供電 rail：12V / 5V 是否受 BMC、CPLD 或 host state 控制
[ ] 風扇模組是否一個 PWM 控多個 rotors
[ ] 風扇 fail 後的硬體預設：PWM floating、BMC reset、CPLD failsafe 時是否全速
```

#### 24.9.3.1 PWM 週期與頻率換算

Device Tree 中常用 period 表示 PWM 週期, 單位通常是 ns:

```text
Frequency_Hz = 1,000,000,000 / Period_ns
Period_ns = 1,000,000,000 / Frequency_Hz
```

常見例:

```text
25 kHz → 40,000 ns
20 kHz → 50,000 ns
10 kHz → 100,000 ns
30 kHz → 33,333 ns
```

若風扇規格書建議 25 kHz, DTS 設為 `40000` ns 是常見起點. 若出現低 duty 無法轉、噪音異常、轉速曲線不連續或 PWM duty 改變但 RPM 不明顯, 需與硬體團隊確認實際 pin 波形與風扇接收頻率.

#### 24.9.3.2 raw PWM、百分比與 RPM

常見層級:

```text
sysfs pwmN：常見 raw 0～255，255 代表最大 duty
D-Bus fan_pwm Sensor.Value：部分實作顯示百分比 0～100
Control.FanPwm.Target：常見 raw target 0～255
phosphor-pid-control failsafePercent：語意為百分比，實際寫入需看 fan sensor scaling
FanSpeed.Target：語意為 RPM
```

建議資料表同時記錄 raw PWM 與百分比:

```text
PWM_percent = raw_pwm / 255 × 100
raw_pwm = round(PWM_percent × 255 / 100)
```

例:

```text
raw 30  ≈ 11.8%
raw 64  ≈ 25.1%
raw 128 ≈ 50.2%
raw 192 ≈ 75.3%
raw 255 = 100%
```

### 24.9.4 常見控制架構

| 架構 | 輸出目標 | 寫入路徑 | 使用場景 | 注意事項 |
| :--- | :--- | :--- | :--- | :--- |
| SoC PWM 直接控風扇 | raw PWM / % | hwmon `pwmN` | BMC PWM pin 直接連 fan PWM input | 需確認 polarity、period、pinctrl、failsafe 預設 |
| `pwm-fan` | raw 0～255 | hwmon `pwm1` | generic Linux PWM fan | `pwm1_enable` mode 影響 `pwm1=0` 時行為 |
| ASPEED PWM/Tach | raw 0～255 | hwmon or driver path | AST2600 常見 | PWM 與 Tach 獨立; 需對 channel |
| I2C fan controller | RPM 或 PWM | I2C register / hwmon | MAX31785 / MAX31790 / NCT 類 | 控制 IC 可能有內建閉環與 fault rule |
| phosphor-pid-control → sysfs | raw PWM 或 RPM | `writePath` | 使用 swampd 控制 zone | `readPath` / `writePath` 與 scaling 必須對齊 |
| phosphor-fan-control | FanSpeed / FanPWM | control path | fan presence/control/monitor stack | target_interface 與 target_path 需一致 |
| PSU fan | PSU command / PMBus | PSUSensor / PSU daemon | PSU 內建風扇 | 可能不是 chassis FanSensor, 且尺度可能 0～100 |

### 24.9.5 Porting 步驟 A: Device Tree / Kernel

#### Step A1: 啟用 PWM controller 與 pinctrl

AST2600 / ASPEED G6 常見 PWM/Tach controller 節點:

```dts
&pwm_tach {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_pwm0_default &pinctrl_pwm1_default
                 &pinctrl_tach0_default &pinctrl_tach1_default>;
};
```

若 fan child node 同時描述 PWM 與 Tach:

```dts
&pwm_tach {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_pwm0_default &pinctrl_pwm1_default
                 &pinctrl_tach0_default &pinctrl_tach1_default>;

    fan-0 {
        tach-ch = /bits/ 8 <0x0>;
        pwms = <&pwm_tach 0 40000 0>;
    };

    fan-1 {
        tach-ch = /bits/ 8 <0x1>;
        pwms = <&pwm_tach 1 40000 0>;
    };
};
```

`pwms = <&pwm_tach 0 40000 0>` 的常見含意:

```text
&pwm_tach：PWM provider
0        ：PWM channel 0
40000    ：period = 40000 ns = 25 kHz
0        ：PWM flags，例如 normal polarity；反相需依 binding / driver 定義
```

#### Step A2: generic `pwm-fan` DTS 範例

若採 generic `pwm-fan`:

```dts
fan0: pwm-fan {
    compatible = "pwm-fan";
    pwms = <&pwm_tach 0 40000 0>;
    cooling-levels = <0 64 128 192 255>;
    #cooling-cells = <2>;
};
```

若同時接 tach interrupt:

```dts
fan0: pwm-fan {
    compatible = "pwm-fan";
    pwms = <&pwm 0 40000 0>;
    interrupts-extended = <&gpio5 1 IRQ_TYPE_EDGE_FALLING>;
    pulses-per-revolution = <2>;
    cooling-levels = <0 80 120 160 200 255>;
    #cooling-cells = <2>;
};
```

#### Step A3: Kernel config

```text
CONFIG_HWMON
CONFIG_PWM
CONFIG_SENSORS_PWM_FAN
CONFIG_PWM_ASPEED 或 SoC 對應 PWM driver
CONFIG_SENSORS_ASPEED_G6_PWM_TACH 或 vendor BSP 對應 ASPEED PWM/Tach driver
CONFIG_THERMAL（若使用 cooling-levels / thermal zone）
CONFIG_REGULATOR（若 fan-supply 由 regulator 控制）
```

#### Step A4: 開機 probe 檢查

```bash
$ dmesg | grep -Ei 'pwm|fan|tach|hwmon|aspeed|npcm'

$ cat /sys/kernel/debug/pinctrl/*/pinmux-pins 2>/dev/null | grep -Ei 'pwm|tach|fan'

$ for h in /sys/class/hwmon/hwmon*; do
    echo "== $h =="
    cat "$h/name" 2>/dev/null || true
    for f in "$h"/pwm* "$h"/fan*_input "$h"/fan*_label; do
        [ -e "$f" ] && echo "$(basename "$f")=$(cat "$f")"
    done
$ done
```

### 24.9.6 Porting 步驟 B: sysfs PWM 驗證

#### Step B1: 找出 PWM 節點

```bash
$ find /sys/class/hwmon -maxdepth 2 -name 'pwm*' -print

$ for h in /sys/class/hwmon/hwmon*; do
    echo "== $h =="
    cat "$h/name" 2>/dev/null || true
    ls "$h"/pwm* 2>/dev/null || true
$ done
```

常見節點:

```text
pwm1
pwm1_enable
pwm1_auto_point1_pwm
pwm1_auto_point1_temp
```

不同 driver 的 `pwm1_enable` 語意可能不同. generic `pwm-fan` 文件中,`pwm1` 是 0～255,`255` 代表最大轉速;`pwm1_enable` 會影響 `pwm1=0` 時 PWM 與 regulator 的保留或關閉行為. 實機測試前需先看當前 driver 文件與 sysfs 說明.

#### Step B2: 安全手動測試

測試前建議:

```text
[ ] 確認可安全降低風扇轉速，避免 CPU / GPU / VR / DIMM 過熱
[ ] 確認有序列埠或 SSH session 可恢復控制
[ ] 準備回復全速指令
[ ] 若 host 正在高負載，先停止 stress test 或固定 failsafe
```

手動測試:

```bash
$ H=/sys/class/hwmon/hwmonX

# 記錄初始值
$ cat $H/name
$ cat $H/pwm1 2>/dev/null || true
$ cat $H/pwm1_enable 2>/dev/null || true
$ cat $H/fan1_input 2>/dev/null || true

# 視 driver 支援情況切到可由 pwm1 控制的模式
# 注意：pwm1_enable 值語意依 driver 而異，請先確認文件
$ echo 1 > $H/pwm1_enable 2>/dev/null || true

# 測試 duty 與 RPM 對應
$ for v in 64 128 192 255; do
    echo $v > $H/pwm1
    sleep 5
    echo "raw_pwm=$v rpm=$(cat $H/fan1_input 2>/dev/null || echo NA)"
$ done

# 回到安全值
$ echo 255 > $H/pwm1
```

測試記錄表:

| PWM raw | PWM % | RPM | Tach sensor | 溫度 | 備註 |
| ---: | ---: | ---: | --- | ---: | --- |
| 64 | 25.1 | [待填] | [待填] | [待填] | [待填] |
| 128 | 50.2 | [待填] | [待填] | [待填] | [待填] |
| 192 | 75.3 | [待填] | [待填] | [待填] | [待填] |
| 255 | 100.0 | [待填] | [待填] | [待填] | [待填] |

### 24.9.7 Entity Manager / FanSensor / PwmSensor 配置

OpenBMC branch 與平台 layer 對 fan JSON 欄位差異較大, 下列範例作為 porting 記錄範本. 實際欄位需以 `entity-manager` schema、`dbus-sensors` FanSensor / PwmSensor source、既有平台 JSON 與 journal log 為準.

#### SoC PWM + Tach 風扇範例

```json
{
  "Name": "Fan0",
  "Type": "AspeedFan",
  "Pwm": 0,
  "Tachs": [
    {
      "Name": "Fan0_Tach",
      "Index": 0
    }
  ],
  "PwmName": "Fan0_PWM",
  "MaxPwm": 255,
  "MinPwm": 30,
  "PollInterval": 1000,
  "PowerState": "AlwaysOn",
  "Thresholds": [
    {
      "Name": "lower critical",
      "Direction": "less than",
      "Severity": 1,
      "Value": 500
    }
  ]
}
```

#### I2C fan controller 範例

```json
{
  "Type": "I2CFan",
  "Name": "FCB_FAN0_TACH",
  "Bus": 18,
  "Address": "0x23",
  "Index": 11,
  "Connector": {
    "Name": "FCB_FAN0_PWM",
    "Pwm": 0,
    "PwmName": "FCB_FAN0_PWM_PCT",
    "Tachs": [11]
  },
  "MaxReading": 25000,
  "PowerState": "AlwaysOn"
}
```

#### 純 PWM control object 範例

若平台將 PWM 視為 `fan_pwm` control object, 而 tach sensor 另行建立:

```json
{
  "Name": "Fan0_PWM",
  "Type": "PwmSensor",
  "Index": 0,
  "MaxValue": 255,
  "MinValue": 0,
  "DefaultValue": 80,
  "PowerState": "AlwaysOn"
}
```

欄位檢查:

```text
[ ] Type 與 schema / daemon 支援型別一致
[ ] Pwm / Index 對應到 sysfs pwmN 與硬體 PWM channel
[ ] PwmName 對應 D-Bus fan_pwm object name
[ ] MinPwm 大於或等於風扇最低啟動需求
[ ] MaxPwm 符合 raw scale，系統 fan 常見 255，PSU fan 可能是 100
[ ] Tachs 對應 fan_tach sensor，方便用 RPM 驗證 PWM 效果
[ ] PowerState 與風扇供電 rail 一致
```

### 24.9.8 D-Bus FanPwm / FanSpeed 介面驗證

OpenBMC 常見 fan control interface:

```text
xyz.openbmc_project.Control.FanPwm：Target 屬性常用於 PWM raw target
xyz.openbmc_project.Control.FanPWM：部分文件或舊設定以全大寫 PWM 命名，需以實機 introspect 為準
xyz.openbmc_project.Control.FanSpeed：Target 屬性常用於 RPM target
xyz.openbmc_project.Sensor.Value：部分 fan_pwm object 會以百分比顯示 Value
```

查找 fan PWM object:

```bash
$ busctl tree /xyz/openbmc_project | grep -Ei 'fan_pwm|fanpwm|pwm'

$ busctl tree xyz.openbmc_project.FanSensor 2>/dev/null | grep -Ei 'pwm|fan'
```

讀取與設定:

```bash
# 找出 service owner
$ busctl call xyz.openbmc_project.ObjectMapper \
  /xyz/openbmc_project/object_mapper \
  xyz.openbmc_project.ObjectMapper GetObject \
  sas \
  /xyz/openbmc_project/sensors/fan_pwm/Fan0_PWM \
  0

# 檢查 interface
$ busctl introspect <service_name> \
  /xyz/openbmc_project/sensors/fan_pwm/Fan0_PWM

# 設定 raw PWM target，signature 可能是 t 或 q，需以 introspect 為準
$ busctl set-property \
  <service_name> \
  /xyz/openbmc_project/sensors/fan_pwm/Fan0_PWM \
  xyz.openbmc_project.Control.FanPwm \
  Target t 128

# 讀回 Target
$ busctl get-property \
  <service_name> \
  /xyz/openbmc_project/sensors/fan_pwm/Fan0_PWM \
  xyz.openbmc_project.Control.FanPwm \
  Target
```

驗證順序:

```text
D-Bus Target 變更
    ↓
sysfs pwmN 變更
    ↓
PWM pin 波形 duty 改變
    ↓
風扇 RPM 變化
    ↓
fan_tach D-Bus Value 變化
```

### 24.9.9 Thermal Policy: PID / Stepwise / Fan Control

#### 24.9.9.1 phosphor-pid-control JSON

`phosphor-pid-control` 可從 dedicated JSON 或 D-Bus 取得設定. 常見 JSON 位置為:

```text
/usr/share/swampd/config.json
```

fan sensor 以 `readPath` 讀 fan tach, 以 `writePath` 寫 PWM:

```json
{
  "sensors": [
    {
      "name": "fan0",
      "type": "fan",
      "readPath": "/xyz/openbmc_project/sensors/fan_tach/Fan0_Tach",
      "writePath": "/sys/class/hwmon/hwmon*/pwm1",
      "min": 0,
      "max": 255,
      "timeout": 4,
      "ignoreDbusMinMax": true,
      "unavailableAsFailed": true
    },
    {
      "name": "Ambient_Temp",
      "type": "temp",
      "readPath": "/xyz/openbmc_project/sensors/temperature/Ambient_Temp",
      "timeout": 5,
      "unavailableAsFailed": true
    }
  ],
  "zones": [
    {
      "id": 0,
      "minThermalOutput": 30.0,
      "failsafePercent": 100.0,
      "pids": ["fan0", "Ambient_Temp"]
    }
  ]
}
```

注意:`minThermalOutput` 在某些配置中常以最小 RPM 使用; 若 fan controller 的輸出是 PWM percentage 或 raw PWM, 需確認該值的語意和下層 scaling.`failsafePercent` 是 zone 進入 fail-safe 時用來寫 fan sensor 的值, 常見策略為 100% 或全速.

#### 24.9.9.2 PID 參數與控制模式

常見控制策略:

```text
Stepwise：依溫度區間查表輸出 PWM 或 RPM，調校簡單、可預期性高
PID：依誤差、積分、微分與 feed-forward 調整，適合需要平滑控制的系統
Fan PID：追目標 RPM，輸出 PWM
Thermal PID：依溫度或 margin 產生目標 fan output
Open loop：只看溫度輸出 PWM，不看 tach 回授
Closed loop：比較目標 RPM 與實際 RPM，再調 PWM
```

調校時建議先完成下列資料:

```text
[ ] PWM raw → RPM 曲線
[ ] 溫度 sensor 位置、熱慣性與有效冷卻 fan zone
[ ] fan fail 時 failsafe PWM
[ ] acoustic target 與 idle 最低轉速
[ ] stress test 下 steady state 溫度
[ ] step up / step down 斜率限制，避免轉速忽高忽低
```

#### 24.9.9.3 phosphor-fan-control fans. json / zones

phosphor-fan-presence / fan-control 文件中, fan control 可指定 target interface:

```json
[
  {
    "name": "fan0",
    "zone": "0",
    "sensors": ["Fan0_Tach"],
    "target_interface": "xyz.openbmc_project.Control.FanPWM",
    "target_path": "/xyz/openbmc_project/control/fanpwm/"
  }
]
```

RPM 控制則可使用:

```json
[
  {
    "name": "fan0",
    "zone": "0",
    "sensors": ["Fan0_Tach"],
    "target_interface": "xyz.openbmc_project.Control.FanSpeed",
    "target_path": "/xyz/openbmc_project/sensors/fan_tach/"
  }
]
```

實機上要以 `busctl introspect` 確認 casing 與 path, 例如 `FanPWM`、`FanPwm`、`fanpwm` 可能在不同元件或版本中有差異.

### 24.9.10 Failsafe 與開機預設

Fan PWM 的 failsafe 設計需要覆蓋下列階段:

```text
BMC reset / BootROM / U-Boot 階段
Kernel probe 前
Kernel driver probe 後但 userspace service 未啟動
FanSensor / PwmSensor 啟動後
PID / fan-control 啟動後
PID service 停止、重啟或 reload configuration
溫度 sensor unavailable / fan tach unavailable / inventory absent
BMC kernel panic 或 watchdog reset
```

建議策略:

```text
[ ] 硬體預設：PWM floating 或 BMC reset 時風扇全速或安全轉速
[ ] Bootloader：若可控，維持安全 PWM，不依賴 userspace
[ ] Kernel driver：probe 後不要把 PWM 變成 0，或設定 default PWM
[ ] FanSensor / PwmSensor：當目前 PWM 為 0 時可設 default PWM
[ ] PID：sensor fail 或 zone fail-safe 時設 failsafePercent
[ ] offline failsafe：PID offline 或 service stop 時 fan 保持安全 PWM
[ ] strict failsafe：進入 fail-safe 時是否固定為 failsafePercent，或取 calculated PWM 與 failsafe 較高者
```

驗證項目:

```bash
# 觀察服務停止後是否進入安全 PWM
$ systemctl stop phosphor-pid-control.service
$ sleep 5
$ cat /sys/class/hwmon/hwmonX/pwm1
$ cat /sys/class/hwmon/hwmonX/fan1_input

# 恢復服務
$ systemctl start phosphor-pid-control.service
$ journalctl -u phosphor-pid-control.service -b --no-pager | tail -100
```

### 24.9.11 Redfish / IPMI / Telemetry 驗證

Fan PWM 不一定會在 Redfish `Thermal` 或 `Sensors` 中呈現; 許多平台只呈現 fan tach RPM, 不呈現 PWM duty. 若平台需求要顯示 PWM, 需確認 bmcweb sensor mapping、sensor type、D-Bus path 與 association.

```bash
$ curl -k -u root:<password> \
  https://<bmc-ip>/redfish/v1/Chassis/<chassis>/Sensors | jq

$ curl -k -u root:<password> \
  https://<bmc-ip>/redfish/v1/Chassis/<chassis>/Thermal | jq

$ ipmitool sensor | grep -Ei 'fan|pwm|tach|rpm'
$ ipmitool sdr elist | grep -Ei 'fan|pwm|tach|rpm'
```

Telemetry / log 建議保存:

```text
[ ] PWM target
[ ] sysfs pwmN
[ ] fan tach RPM
[ ] controlling temperature / margin sensor
[ ] zone mode：auto / manual / failsafe
[ ] fan presence / functional state
[ ] service restart / fail / reload event
```

### 24.9.12 進階除錯與常見陷阱

| 問題現象 | 建議排查方向 | 排查 / 處理方式 |
| :--- | :--- | :--- |
| `pwm*` sysfs 不存在 | DTS 未啟用、driver 未載入、kernel config 缺少 PWM / pwm-fan / SoC driver | 查 `dmesg`、`.config`、pinctrl、controller compatible |
| 寫入 `pwm1` 後 RPM 不變 | `pwm1_enable` 模式不對、風扇固定全速、PWM channel 對錯、PWM pin 無波形 | 先量 PWM pin, 再逐一比對 fan connector 與 tach |
| 風扇無法從停止啟動 | duty 低於 start duty、供電 rail 未穩、風扇需要 start boost | 提高 MinPwm, 加入啟動 boost 或開機預設全速 |
| PWM 與 RPM 不成比例 | 風扇內部控制曲線、PWM 頻率不合、tach PPR 錯、控制 IC 另有閉環 | 建 PWM→RPM 表, 確認 PWM period 與 tach PPR |
| 設 0 後風扇未停 | 風扇內建最低轉速、`pwm1_enable` 保留 regulator、硬體 pull-up 導致全速 | 查風扇規格、driver 文件與 PWM pin 波形 |
| 設 255 仍非全速 | PWM polarity 反相、MaxPwm scale 不符、fan controller 限速 | 確認 flags、polarity、controller register、MaxPwm |
| D-Bus FanPwm 找不到 | PwmSensor 未建立、JSON Type / PwmName 不符合、service 未啟動 | 查 FanSensor log、ObjectMapper、Entity Manager log |
| 手動設定後很快被覆寫 | PID / fan-control 正在 auto mode | 切明確 manual mode、停止控制服務或使用平台提供的 override flow |
| 風扇長期全速 | failsafe 被觸發、溫度 sensor unavailable、fan tach timeout、zone mode 在 fail-safe | 查 phosphor-pid-control log、zone mode、failed sensors |
| 服務啟動時風扇短暫降到 0 | default PWM 不足、service 順序、driver probe 初始值為 0 | 設硬體 failsafe、kernel default、PwmSensor default、systemd dependency |
| Redfish 看不到 PWM | 平台只 expose tach, 不 expose pwm | 確認需求; 若需呈現, 補 D-Bus sensor 與 bmcweb mapping |
| PSU fan PWM 尺度不對 | PSU fan 可能是 0～100, 不是 0～255 | 查 PwmSensor scale、PSU daemon 與 PSU PMBus command |

### 24.9.13 Fan PWM / Fan Control Porting 驗收 Checklist

硬體設計階段:

- [ ] 風扇型號與規格書確認: PWM frequency、voltage、duty range、polarity
- [ ] PWM channel 與 fan connector 對照完成
- [ ] Tach channel 與 PWM channel 配對完成
- [ ] PWM pull-up / level shift / buffer / CPLD path 確認
- [ ] 最小啟動 duty、最小穩定 duty、全速 duty 確認
- [ ] 硬體 failsafe 狀態確認: BMC reset / pin floating / CPLD failsafe 時風扇狀態
- [ ] host off / standby / chassis on 的風扇策略確認

Device Tree / Kernel:

- [ ] PWM controller node status = "okay"
- [ ] pinctrl 包含所有 PWM pin, 且無 pinmux 衝突
- [ ] fan child node 或 pwm-fan node 加入, pwms channel 與 period 正確
- [ ] 若使用 tach 回授, tach-ch / pulses-per-revolution 也已設定
- [ ] kernel config 啟用 HWMON、PWM、pwm-fan、SoC PWM/Tach driver
- [ ] dmesg 無 probe failure、invalid channel、pinctrl failure
- [ ] /sys/class/hwmon/hwmonX/pwmN 存在
- [ ] 手動設定 pwmN 後, 示波器可看到 duty 改變
- [ ] fan_tach RPM 隨 PWM 變化

Entity Manager / Userspace:

- [ ] Fan / PWM JSON 已加入正確 layer 並進 image
- [ ] Type、Pwm、PwmName、Tachs / Index 與 schema 及硬體一致
- [ ] MinPwm / MaxPwm / Default PWM 設定合理
- [ ] PowerState gating 與風扇供電狀態一致
- [ ] FanSensor / PwmSensor service 啟動無錯誤
- [ ] ObjectMapper 可找到 fan_pwm 或 fan control object
- [ ] Control. FanPwm / Control. FanPWM / Control. FanSpeed interface 依平台需求存在

Thermal Policy:

- [ ] phosphor-pid-control 或 fan-control 設定檔已加入 image
- [ ] sensors readPath 對應 fan_tach 與 temperature sensor
- [ ] fan writePath 對應 sysfs pwmN 或 D-Bus fan control path
- [ ] zones id、minThermalOutput、failsafePercent、pids 設定合理
- [ ] PID / Stepwise 參數已初步調校
- [ ] manual / auto / failsafe mode 切換路徑已驗證
- [ ] 溫度上升時 PWM 增加, 溫度下降時 PWM 降低
- [ ] fan fail 或 sensor fail 時進入預期 failsafe PWM

D-Bus / Redfish / IPMI / Event:

- [ ] busctl 可讀寫 Target, 且 signature 與實機 introspect 一致
- [ ] 設定 Target 後 sysfs pwmN 同步變化
- [ ] sysfs pwmN 變化後 fan_tach RPM 同步變化
- [ ] Redfish 依平台需求可看到 fan tach, 若需要也可看到 fan_pwm
- [ ] IPMI SDR / sensor list 依平台需求可看到 fan tach / pwm
- [ ] 停止 PID service、拔除風扇、temperature sensor unavailable 等情境都有預期 log
- [ ] SEL / Journal / EventLog 符合平台 event policy

### 24.9.14 Fan PWM / Fan Control 資料表範本

| Fan Name | PWM Channel | PWM Path | PWM Period / Hz | MinPwm | MaxPwm | Default / Failsafe | Tach Sensor | Control Path | Zone | 備註 |
| --- | ---: | --- | --- | ---: | ---: | --- | --- | --- | ---: | --- |
| Fan0 | 0 | `/sys/class/hwmon/hwmonX/pwm1` | 40000 ns / 25 kHz | 30 | 255 | 255 | Fan0_Tach | `/xyz/openbmc_project/sensors/fan_pwm/Fan0_PWM` | 0 | [待填] |
| Fan1 | 1 | `/sys/class/hwmon/hwmonX/pwm2` | 40000 ns / 25 kHz | 30 | 255 | 255 | Fan1_Tach | `/xyz/openbmc_project/sensors/fan_pwm/Fan1_PWM` | 0 | [待填] |
| PSU0_FAN | [待填] | [待填] | [待填] | [待填] | 100 | 100 | PSU0_FAN_TACH | [待填] | [待填] | PSU fan scale may be 0～100 |

### 24.9.15 本節參考資料

- Linux kernel `pwm-fan` 文件:`pwm-fan` 使用 generic PWM interface 驅動風扇, 並透過 hwmon sysfs 提供 `fan1_input`、`pwm1_enable` 與 `pwm1`; 其中 `pwm1` 為 0～255,255 代表最大轉速.
- OpenBMC dbus-sensors PwmSensor 架構資料: PwmSensor 透過 sysfs `pwm*` 控制風扇, 並可提供 `Sensor.Value` 百分比與 `Control.FanPwm` raw target; 系統風扇常見 raw scale 為 0～255, PSU fan 可能為 0～100.
- OpenBMC phosphor-dbus-interfaces control namespace: FanPwm 提供 `Target` 屬性用於 duty cycle, FanSpeed 提供 `Target` 屬性用於 RPM target.
- OpenBMC phosphor-fan-presence fan control 文件: fan control 可使用 `xyz.openbmc_project.Control.FanSpeed` 或 `xyz.openbmc_project.Control.FanPWM` 作為 target interface.
- OpenBMC phosphor-pid-control configure 文件: 設定包含 `sensors` 與 `zones`; fan sensor 可透過 `readPath` 讀取 fan tach, 透過 `writePath` 寫入 PWM; zone 中 `minThermalOutput` 常用作最低 thermal output,`failsafePercent` 用於 fail-safe 輸出.

<a id="section-12-10"></a>

## 24.10 PSU Sensor

### 24.10.1 適用情境

Power Supply Unit(PSU)是伺服器系統的電源核心. 現代伺服器 PSU 多數支援 PMBus over I2C/SMBus, 可提供電壓、電流、功率、溫度、風扇轉速、告警、FRU / Asset 與冗餘狀態. Linux kernel 的 PMBus hwmon driver 會把支援的 PMBus command 轉成 `/sys/class/hwmon/hwmonX/` 下的標準 hwmon 屬性; OpenBMC userspace 再由 `dbus-sensors` 的 `PSUSensor` 搭配 Entity Manager 設定建立 D-Bus sensor.

常見 PSU sensor:

```text
Input voltage / Output voltage
Input current / Output current
Input power / Output power
Temperature
Fan speed
Presence
Fault status
FRU / Asset data
Redundancy status
```

常見 D-Bus path:

```text
/xyz/openbmc_project/sensors/voltage/PSU0_Input_Voltage
/xyz/openbmc_project/sensors/voltage/PSU0_Output_Voltage
/xyz/openbmc_project/sensors/current/PSU0_Input_Current
/xyz/openbmc_project/sensors/current/PSU0_Output_Current
/xyz/openbmc_project/sensors/power/PSU0_Input_Power
/xyz/openbmc_project/sensors/power/PSU0_Output_Power
/xyz/openbmc_project/sensors/temperature/PSU0_Temp
/xyz/openbmc_project/sensors/fan_tach/PSU0_Fan
```

需分清楚三種狀態:

- `Sensor.Value`: 電壓、電流、功率、溫度、轉速等讀值.
- `Inventory.Item.Present`: PSU 是否插入.
- `OperationalStatus.Functional`: PSU 在存在狀態下是否健康.

Presence 與 Functional 不宜混用. 拔除 PSU 通常是 `Present=false`; 插著但故障時通常是 `Present=true` 且 `Functional=false` 或產生 fault / event.

### 24.10.2 資料路徑(Data Flow)

```mermaid
flowchart LR
    PSU[PSU slot] --> PM[PMBus telemetry]
    PSU --> PRES[Presence / FRU EEPROM / fault pin]
    PM --> KD[PMBus kernel driver]
    KD --> HW[hwmon: V / A / W / °C / RPM]
    CFG[Entity Manager labels + slot mapping] --> PS[PSUSensor]
    HW --> PS
    PRES --> INV[Inventory Present / Functional]
    PS --> DB[D-Bus sensor objects]
    INV --> ASSOC[Inventory association]
    DB --> RF[Redfish PowerSubsystem / Sensors]
    ASSOC --> RF
    DB --> IPMI[IPMI / SEL / Redundancy policy]
```

```text
PSU / CRPS / CFFPS / Vendor PSU
    PMBus over I2C/SMBus
    ↓
Linux Kernel
    pmbus / vendor-specific pmbus driver
    ↓
/sys/class/hwmon/hwmonX/
    ├── in*_input / in*_label          voltage, mV
    ├── curr*_input / curr*_label      current, mA
    ├── power*_input / power*_label    power, µW
    ├── temp*_input / temp*_label      temperature, m°C
    ├── fan*_input                     RPM
    └── *_alarm / *_fault / *_crit / *_max / *_min
    ↓
Userspace Daemons
    ├── Entity Manager                 inventory / Exposes / sensor config
    ├── PSUSensor (dbus-sensors)       hwmon → D-Bus sensor
    ├── phosphor-power / psu monitor   presence / fault / redundancy / event
    └── phosphor-regulators            VR / regulator rail，視平台需求啟用
    ↓
D-Bus
    ├── xyz.openbmc_project.Sensor.Value
    ├── xyz.openbmc_project.Sensor.Threshold.Warning / Critical
    ├── xyz.openbmc_project.State.Decorator.Availability
    ├── xyz.openbmc_project.State.Decorator.OperationalStatus
    ├── xyz.openbmc_project.Inventory.Item
    └── xyz.openbmc_project.Inventory.Decorator.Asset
    ↓
Northbound Interfaces
    ├── Redfish: /redfish/v1/Chassis/<id>/Power
    ├── Redfish: /redfish/v1/Chassis/<id>/PowerSubsystem/PowerSupplies
    ├── Redfish: /redfish/v1/Chassis/<id>/Sensors
    └── IPMI: SDR / Sensor reading / SEL
```

Linux PMBus driver 的 hwmon 屬性只會出現硬體與 driver 判定支援的項目; 若 PSU 不支援某 command, 或通用 `pmbus` driver 無法安全偵測該能力, 對應 sysfs 檔案可能不存在.

### 24.10.3 常見來源分類與特性

| 來源類型 | 通訊協定 | Linux / OpenBMC 對應 | 關鍵注意事項 |
| :--- | :--- | :--- | :--- |
| 標準 PMBus PSU | PMBus over I2C/SMBus | `pmbus` driver + `PSUSensor` | 需明確建立 I2C device; 確認 bus / address / page / label. |
| 廠商專用 PSU | PMBus + vendor extension | vendor pmbus driver, 例如 `ibm-cffps`、`inspur-ipsps` | 可能需要客製 command、狀態解碼、FRU 讀取方式或 fan/fault 對應. |
| CRPS / CFFPS PSU | PMBus + FRU EEPROM / vendor commands | `pmbus` / vendor driver + phosphor-power | 需處理 presence、redundancy、AC lost、fault LED、hot-plug. |
| PSU fan | PMBus fan command 或 PSU 內部 controller | `fan*_input` → `fan_tach` sensor | 很多 PSU fan 只回報 RPM 與 fault, 不由 BMC PWM 直接控制. |
| PSU presence | GPIO、CPLD、PMBus ACK、FRU EEPROM | phosphor-power / GPIO monitor / platform daemon | GPIO 極性、CPLD bit、I2C NACK 與真正拔除需分開判讀. |
| PSU fault / health | PMBus `STATUS_WORD` / `STATUS_*`、GPIO fault、CPLD latch | phosphor-power / vendor daemon / event monitor | `STATUS_WORD` 通常需搭配 `STATUS_INPUT` / `STATUS_TEMPERATURE` / `STATUS_FANS` 等細項解讀. |

### 24.10.4 Porting 前需確認的硬體與規格資料

```text
[必填] PSU 型號、廠商、form factor（CRPS / CFFPS / 客製）
[必填] PSU 數量與 slot 編號（PSU0、PSU1、...）
[必填] I2C bus number、mux channel、7-bit address
[必填] 使用通用 pmbus driver 或 vendor-specific driver
[必填] PMBus command 支援清單：READ_VIN、READ_VOUT、READ_IIN、READ_IOUT、READ_PIN、READ_POUT、READ_TEMPERATURE、READ_FAN_SPEED、STATUS_WORD 等
[必填] PSU main output 與 standby output：12V、12VSB、48V、54V 或其他設計
[必填] Presence 偵測來源：GPIO、CPLD bit、I2C ACK、FRU EEPROM、PMBus read
[必填] Fault / AC OK / DC OK / PSU FAIL / fan fault 訊號來源與極性
[建議] FRU / Asset data 來源：EEPROM、PMBus MFR_ID / MFR_MODEL / MFR_SERIAL
[建議] 冗餘策略：1+1、N+1、N+N、cold redundancy、active/standby
[建議] Threshold：輸入電壓上下限、輸出電壓上下限、功率上限、溫度上限、fan RPM 下限
```

PMBus 常見 command 與 hwmon 對應:

| PMBus command | Command code | 常見 sysfs / label | 單位 | 說明 |
| :--- | :--- | :--- | :--- | :--- |
| `READ_VIN` | `0x88` | `inX_input`, `inX_label=vin` | mV | 輸入電壓. |
| `READ_IIN` | `0x89` | `currX_input`, `currX_label=iin` | mA | 輸入電流. |
| `READ_VOUT` | `0x8B` | `inX_input`, `inX_label=voutY` | mV | 輸出電壓. |
| `READ_IOUT` | `0x8C` | `currX_input`, `currX_label=ioutY` | mA | 輸出電流. |
| `READ_TEMPERATURE_1` | `0x8D` | `tempX_input` | m°C | PSU 內部溫度. |
| `READ_FAN_SPEED_1` | `0x90` | `fanX_input` | RPM | PSU 內部風扇轉速. |
| `READ_POUT` | `0x96` | `powerX_input`, `powerX_label=poutY` | µW | 輸出功率. |
| `READ_PIN` | `0x97` | `powerX_input`, `powerX_label=pin` | µW | 輸入功率. |
| `STATUS_WORD` | `0x79` | `*_alarm` / `*_fault` 或 vendor daemon | bit field | PSU 總狀態. |

### 24.10.5 I2C / PMBus 通訊驗證

```bash
$ i2cdetect -y 4

# READ_VIN，word read；回傳是 PMBus raw format，不建議直接把十六進位值當電壓
$ i2cget -y 4 0x58 0x88 w

# READ_PIN
$ i2cget -y 4 0x58 0x97 w

# STATUS_WORD
$ i2cget -y 4 0x58 0x79 w
```

注意事項:

- `i2cdetect` / `i2cget` 對某些 device 可能造成副作用; bring-up 前需確認同 bus 上 device 可接受這類 access.
- `i2cget ...w` 顯示的是 SMBus word, endianness 與 PMBus Linear-11 / Linear-16 解碼需另外處理; 進入 kernel hwmon 後, driver 會轉成標準單位.
- 若 bus 上有 I2C mux, 需確認 mux channel、idle disconnect 設定與 BMC service 是否會改變 mux 狀態.
- I2C NACK 不一定代表 PSU 被拔除, 也可能是 AC lost、standby rail 未穩、PSU booting、bus stuck 或 CPLD gate 關閉.

### 24.10.6 Kernel Driver 與 Device Tree

Linux PMBus driver 通常不會任意掃描所有位址來尋找 PSU, 需透過 Device Tree、board file 或 runtime `new_device` 明確建立 I2C client.`i2cdetect` 看得到位址, 不代表 kernel driver 已經 bind.

```dts
&i2c4 {
    status = "okay";
    clock-frequency = <100000>;

    psu0: power-supply@58 {
        compatible = "pmbus";
        reg = <0x58>;
        label = "psu0";
    };

    psu1: power-supply@59 {
        compatible = "pmbus";
        reg = <0x59>;
        label = "psu1";
    };
};
```

Kernel config 檢查:

```text
CONFIG_I2C=y
CONFIG_HWMON=y
CONFIG_PMBUS=y
CONFIG_SENSORS_PMBUS=y
CONFIG_SENSORS_IBM_CFFPS=y        # 若使用 IBM CFFPS
CONFIG_SENSORS_INSPUR_IPSPS=y     # 若使用 Inspur PSU
```

```bash
$ dmesg | grep -Ei 'pmbus|cffps|ipsps|psu|power-supply'
$ ls -l /sys/bus/i2c/devices/4-0058/
$ readlink /sys/bus/i2c/devices/4-0058/driver

# 臨時驗證 driver bind
$ modprobe pmbus
$ echo pmbus 0x58 > /sys/bus/i2c/devices/i2c-4/new_device
```

### 24.10.7 hwmon sysfs 節點與單位確認

```bash
$ for i in /sys/class/hwmon/hwmon*; do
    echo "$i: $(cat "$i/name" 2>/dev/null)"
$ done

$ HWMON=/sys/class/hwmon/hwmonX
$ cat $HWMON/name
$ ls $HWMON
$ cat $HWMON/in1_input       # voltage, mV
$ cat $HWMON/in1_label       # vin / vout1 / ...
$ cat $HWMON/curr1_input     # current, mA
$ cat $HWMON/curr1_label     # iin / iout1 / ...
$ cat $HWMON/power1_input    # power, µW
$ cat $HWMON/power1_label    # pin / pout1 / ...
$ cat $HWMON/temp1_input     # temperature, m°C
$ cat $HWMON/fan1_input      # RPM
```

hwmon 標準單位:

```text
in*_input       millivolt (mV)
curr*_input     milliampere (mA)
power*_input    microwatt (µW)
temp*_input     millidegree Celsius (m°C)
fan*_input      RPM
```

PSUSensor 會依 hwmon label 與 Entity Manager 設定建立 sensor. Bring-up 時建議建立"label → sysfs → D-Bus path"對照表, 而不是假設 `in1` 一定是輸入電壓、`in2` 一定是輸出電壓.

```bash
$ for f in $HWMON/*_label; do
    base=${f%_label}
    echo "$(basename "$base") label=$(cat "$f") input=$(cat "${base}_input" 2>/dev/null)"
$ done

$ find $HWMON -maxdepth 1 -type f | grep -E 'alarm|fault|crit|max|min' | sort
```

### 24.10.8 Entity Manager 配置

不同 OpenBMC 分支的 schema 與 dbus-sensors 支援欄位會有差異; 以下為常見結構, 實際需以專案 branch 的 schema、既有平台 JSON 與 `PSUSensorMain.cpp` 支援項目為準.

```json
{
    "Name": "PSU0",
    "Type": "PowerSupply",
    "Probe": "TRUE",
    "Exposes": [
        {
            "Name": "PSU0",
            "Type": "pmbus",
            "Bus": 4,
            "Address": "0x58",
            "PollRate": 5.0,
            "Labels": ["vin", "vout1", "iin", "iout1", "pin", "pout1", "temp1", "fan1"],
            "vin_Name": "PSU0_Input_Voltage",
            "vout1_Name": "PSU0_Output_Voltage",
            "iin_Name": "PSU0_Input_Current",
            "iout1_Name": "PSU0_Output_Current",
            "pin_Name": "PSU0_Input_Power",
            "pout1_Name": "PSU0_Output_Power",
            "temp1_Name": "PSU0_Temp",
            "fan1_Name": "PSU0_Fan"
        }
    ],
    "xyz.openbmc_project.Inventory.Decorator.Asset": {
        "Manufacturer": "[待填]",
        "Model": "[待填]",
        "PartNumber": "[待填]",
        "SerialNumber": "[待填]"
    }
}
```

PSU 需和 chassis / baseboard 建立供電關係, 讓 Redfish 能把 sensor 與 power supply inventory 正確關聯. 常見概念是 baseboard 端建立 `powered_by` port, PSU 端建立 `powering` port; 實際 JSON 寫法需依專案 schema 確認.

### 24.10.9 PSU Presence、Fault 與 Inventory

PSU presence 常見來源: GPIO、CPLD、PMBus ACK、FRU EEPROM. 建議判讀原則:

| 情境 | Presence | Functional / Health | Sensor 行為 |
| :--- | :--- | :--- | :--- |
| PSU 拔除 | false | false 或 unavailable | sensor 可移除、標記 unavailable, 或依平台策略保留但不更新. |
| PSU 插入且正常 | true | true | sensor 正常更新. |
| PSU 插入但 AC lost | true | false / warning | input voltage/power 可能為 0 或 unavailable, 需產生事件. |
| PSU 插入但 PMBus timeout | true 或待確認 | false / unavailable | 需區分 bus stuck、PSU booting、driver issue. |
| PSU fan fault / OT / OC | true | false | D-Bus health 與 log 要能反映. |

```bash
$ systemctl list-units | grep -Ei 'psu|power'
$ journalctl -b | grep -Ei 'psu|power supply|redundan|pmbus|cffps'
```

### 24.10.10 啟動服務與 D-Bus 驗證

```bash
$ systemctl status xyz.openbmc_project.EntityManager.service --no-pager
$ journalctl -u xyz.openbmc_project.EntityManager.service -b --no-pager

$ systemctl status xyz.openbmc_project.PSUSensor.service --no-pager
$ journalctl -u xyz.openbmc_project.PSUSensor.service -b --no-pager

$ busctl tree xyz.openbmc_project.PSUSensor
$ busctl tree xyz.openbmc_project.ObjectMapper | grep -Ei 'PSU|PowerSupply|Input|Output'

$ busctl get-property \
  xyz.openbmc_project.PSUSensor \
  /xyz/openbmc_project/sensors/voltage/PSU0_Input_Voltage \
  xyz.openbmc_project.Sensor.Value \
  Value
```

### 24.10.11 Redfish / IPMI / Event 驗證

```bash
$ curl -k -u root:0penBmc https://<bmc>/redfish/v1/Chassis/<id>/Power
$ curl -k -u root:0penBmc https://<bmc>/redfish/v1/Chassis/<id>/PowerSubsystem
$ curl -k -u root:0penBmc https://<bmc>/redfish/v1/Chassis/<id>/PowerSubsystem/PowerSupplies
$ curl -k -u root:0penBmc https://<bmc>/redfish/v1/Chassis/<id>/Sensors

$ ipmitool sdr list | grep -Ei 'psu|power|vin|vout|pin|pout|fan'
$ ipmitool sensor | grep -Ei 'psu|power|vin|vout|pin|pout|fan'
$ ipmitool sel list

$ journalctl -b | grep -Ei 'psu|power supply|threshold|critical|warning|fault'
```

需驗證拔除、插入、AC lost、fan fault、over temperature、over current、threshold event、Redundancy 狀態與 SEL / Journal 行為.

### 24.10.12 進階除錯與常見陷阱

| 問題現象 | 建議排查方向 | 排查 / 解法 |
| :--- | :--- | :--- |
| `i2cdetect` 掃不到 PSU 位址 | PSU 未插入、AC/standby 未供電、bus / mux / address 錯誤、CPLD gate 關閉 | 確認 PSU slot、AC、mux channel、schematic bus、CPLD register; 量測 SCL/SDA. |
| `i2cdetect` 看得到, 但無 hwmon | DTS 未建立 I2C client、driver 未啟用、compatible 不對 | 查 `dmesg`、`/sys/bus/i2c/devices`、kernel config; 臨時用 `new_device` 驗證. |
| hwmon 有節點但讀值為 0 | PSU standby、command 不支援、driver 判讀錯、PSU 尚未 ready | 對照 PMBus command list; 等待 PSU ready; 確認 12V main / 12VSB 狀態. |
| 讀值倍率錯 | hwmon 單位與 D-Bus 單位換算錯、重複 scaling、vendor direct format 參數錯 | 同步比對 raw PMBus、sysfs、D-Bus、Redfish; 確認 scale/offset 是否重複. |
| D-Bus sensor 未出現 | Entity Manager JSON 未載入、Labels 不符、service 未啟動 | 查 Entity Manager log、PSUSensor log、`*_label` 檔案; 確認 `Labels` 對上 hwmon label. |
| Redfish 看不到 PSU | Association 不完整、inventory path 不符、bmcweb schema/feature 差異 | 檢查 Port association、ObjectMapper、bmcweb log. |
| Presence 不正確 | GPIO 極性錯、CPLD bit 定義錯、PMBus timeout 被當拔除 | 拔插實測 GPIO/CPLD bit; 區分 Present 與 Functional. |
| PSU fault 無 log | STATUS command 未讀、fault bit 未對應 event、daemon 未啟用 | 檢查 `STATUS_WORD`、`*_alarm`、phosphor-power log 與 event mapping. |

### 24.10.13 PSU Sensor Porting 驗收 Checklist

硬體 / 規格:
- [ ] PSU 型號、form factor、slot 編號確認
- [ ] I2C bus / mux channel / address 確認
- [ ] PMBus command 支援清單確認
- [ ] Presence / Fault / AC OK / DC OK 訊號來源與極性確認
- [ ] FRU / Asset data 來源確認
- [ ] Redundancy policy 確認
- [ ] Thresholds 與額定範圍確認

Kernel / DTS:
- [ ] I2C controller node status = okay
- [ ] PSU I2C node compatible / reg 正確
- [ ] CONFIG_PMBUS / CONFIG_SENSORS_PMBUS / vendor driver 啟用
- [ ] dmesg 無 probe fail / timeout / bus error
- [ ] /sys/bus/i2c/devices/<bus>-00<addr>/driver bind 正確
- [ ] /sys/class/hwmon/hwmonX/name 對應 PSU driver
- [ ] in / curr / power / temp / fan hwmon 節點存在且單位正確
- [ ] *_label 對應 vin / vout / iin / iout / pin / pout / temp / fan

Entity Manager / Userspace:
- [ ] PSU JSON 放入正確 layer / image
- [ ] Type / Bus / Address / Labels 與 schema 對齊
- [ ] <label>_Name 命名符合平台規範
- [ ] PollRate 合理, I2C bus loading 可接受
- [ ] Baseboard / Chassis / PSU association 完成
- [ ] Presence / fault daemon 配置完成
- [ ] FRU / Asset 可讀取或以平台資料補齊

D-Bus / Redfish / IPMI:
- [ ] PSUSensor service 啟動無錯誤
- [ ] busctl 可看到 voltage / current / power / temperature / fan_tach sensor
- [ ] sensor Value 與 sysfs 換算一致
- [ ] Availability / Functional 狀態符合插拔與故障情境
- [ ] Redfish 可看到 PSU inventory 與 sensors
- [ ] IPMI SDR / sensor reading / SEL 符合平台需求
- [ ] 拔除 PSU、插入 PSU、AC lost、fault injection 行為都有驗證
- [ ] critical / warning threshold 與 event log 可正常觸發

### 24.10.14 PSU Sensor 資料表範本

| PSU Slot | I2C Bus | Mux Channel | Address | Driver | hwmon name | Label | sysfs | D-Bus Path | Presence Source | Fault Source | 備註 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| PSU0 | 4 | [待填] | 0x58 | pmbus | [待填] | vin | inX_input | /xyz/openbmc_project/sensors/voltage/PSU0_Input_Voltage | GPIO/CPLD/PMBus | STATUS_WORD/CPLD | [待填] |
| PSU0 | 4 | [待填] | 0x58 | pmbus | [待填] | pout1 | powerX_input | /xyz/openbmc_project/sensors/power/PSU0_Output_Power | GPIO/CPLD/PMBus | STATUS_WORD/CPLD | [待填] |
| PSU0 | 4 | [待填] | 0x58 | pmbus | [待填] | temp1 | tempX_input | /xyz/openbmc_project/sensors/temperature/PSU0_Temp | GPIO/CPLD/PMBus | STATUS_WORD/CPLD | [待填] |
| PSU0 | 4 | [待填] | 0x58 | pmbus | [待填] | fan1 | fanX_input | /xyz/openbmc_project/sensors/fan_tach/PSU0_Fan | GPIO/CPLD/PMBus | STATUS_FANS/CPLD | [待填] |

### 24.10.15 本節參考資料

- Linux Kernel Documentation - Kernel driver pmbus: https://docs.kernel.org/hwmon/pmbus.html
- OpenBMC dbus-sensors README 與 PSUSensor source: https://github.com/openbmc/dbus-sensors
- OpenBMC Entity Manager README / schema / configurations: https://github.com/openbmc/entity-manager
- OpenBMC phosphor-power: https://github.com/openbmc/phosphor-power

<a id="section-12-11"></a>

## 24.11 CPU Sensor / PECI / APML

### 24.11.1 適用情境

CPU Sensor 用於監控伺服器中央處理器的溫度、功耗與健康狀態, 是散熱政策、功耗管理、事件紀錄與系統保護的重要輸入. x86 伺服器平台依 CPU 廠商不同, 常見有兩條 out-of-band 管理路徑: Intel 使用 PECI, AMD EPYC 使用 APML.

| 廠商 | 介面 | 全名 | 主要用途 |
| :--- | :--- | :--- | :--- |
| Intel | PECI | Platform Environment Control Interface | BMC 讀取 CPU package / core / DIMM thermal telemetry. |
| AMD | APML | Advanced Platform Management Link | BMC 透過 SB-TSI / SB-RMI 讀 CPU temperature、power、power cap、DIMM thermal 與 APML event. |

Intel PECI 常見項目:

```text
CPU package temperature
CPU DTS / thermal margin
CPU core temperature
Tcontrol / Tthrottle / Tjmax
DIMM temperature
CPU presence / CPU reachable state
```

AMD APML 常見項目:

```text
CPU package temperature, via SB-TSI
CPU package power, via SB-RMI
power cap / power cap max, via SB-RMI
DIMM TS0 / TS1 thermal sensor, via SB-RMI mailbox
APML_ALERT_L event notification, if routed to BMC GPIO
CPU presence / CPU reachable state
```

OpenBMC 中常見 D-Bus sensor path:

```text
# Intel PECI
/xyz/openbmc_project/sensors/temperature/CPU0_Temp
/xyz/openbmc_project/sensors/temperature/CPU0_Core0_Temp
/xyz/openbmc_project/sensors/temperature/DIMM0_Temp

# AMD APML
/xyz/openbmc_project/sensors/temperature/CPU0_Temp
/xyz/openbmc_project/sensors/temperature/DIMM_TS0_UMC0_Temp
/xyz/openbmc_project/sensors/temperature/DIMM_TS1_UMC0_Temp
/xyz/openbmc_project/sensors/power/CPU0_Power
```

CPU Sensor 與一般 board sensor 最大差異是 host power state dependency. CPU / DIMM telemetry 通常只有在 host 上電、CPU 初始化到一定階段、BIOS / firmware 完成必要訓練後才會穩定. 因此 CPU sensor porting 不只要看 kernel driver, 也要同步 host power state、BIOS 設定、socket presence、CPU generation 與 BMC service rescan 行為.

### 24.11.2 Intel PECI 架構與資料路徑

PECI 是 Intel server CPU 常見 out-of-band thermal 管理介面. BMC 作為 originator, CPU 作為 responder. 常見 PECI client address 依 socket 位置排列:

| Socket | PECI address |
| :--- | :--- |
| CPU0 | `0x30` |
| CPU1 | `0x31` |
| CPU2 | `0x32` |
| CPU3 | `0x33` |

Intel PECI 資料路徑:

```text
Intel CPU PECI responder, e.g. 0x30 / 0x31
    ↓
PECI single-wire bus
    ↓
BMC PECI controller, e.g. ASPEED PECI
    ↓
Linux PECI subsystem
    ├── PECI controller driver
    ├── PECI client device
    ├── peci-cputemp   CPU package / core temperature
    └── peci-dimmtemp  DIMM temperature
    ↓
hwmon sysfs
    ├── /sys/class/hwmon/hwmonX/temp*_input
    ├── /sys/class/hwmon/hwmonX/temp*_label
    ├── /sys/class/hwmon/hwmonX/temp*_max
    └── /sys/class/hwmon/hwmonX/temp*_crit
    ↓
IntelCPUSensor, dbus-sensors
    ↓
D-Bus Sensor.Value / Threshold / Availability / OperationalStatus
    ↓
Redfish / IPMI / phosphor-pid-control / phosphor-logging
```

Linux `peci-cputemp` 會提供 CPU package、DTS、Tcontrol、Tthrottle、Tjmax 與 per-core temperature 等 hwmon 屬性; 所有 temperature value 以 millidegree Celsius 表示, 且只有 target CPU powered on 時可量測.`peci-dimmtemp` 會提供 DIMM temperature, 同樣以 millidegree Celsius 表示, 且 DIMM thermal 屬性需等 BIOS 完成 memory training / testing 後才會出現.

### 24.11.3 AMD APML 架構與資料路徑

AMD APML 由多個 sideband 協定與 driver 組成, 常見子介面如下:

| 介面 | 全名 | 主要用途 |
| :--- | :--- | :--- |
| SB-TSI | Sideband Thermal Sensor Interface | CPU socket temperature、temperature threshold. |
| SB-RMI | Sideband Remote Management Interface | socket power、power cap、power cap max、DIMM thermal mailbox、進階 mailbox command. |
| APML_ALERT_L | APML alert line | APML event notification, 視平台設計接到 GPIO. |

常見 SB-RMI address:

| Socket | 常見 8-bit 表示 | 常見 7-bit 表示 |
| :--- | :--- | :--- |
| CPU0 | `0x78` | `0x3C` |
| CPU1 | `0x70` | `0x38` |

常見 SB-TSI address 依平台 address select pin 而定; 許多平台使用 `0x4C` / `0x48`, 仍需以 schematic 與 PPR 為準.

AMD APML 資料路徑:

```text
AMD EPYC CPU APML responder
    ↓
I2C or I3C bus from BMC to CPU socket
    ↓
Linux kernel drivers
    ├── sbtsi / apml_sbtsi
    │     CPU socket temperature, threshold
    ├── sbrmi / apml_sbrmi
    │     power1_input / power1_cap / power1_cap_max
    │     DIMM TS0 / TS1 temperature, if firmware supports mailbox command
    └── apml_alertl, optional
          APML_ALERT_L GPIO event notification
    ↓
hwmon sysfs
    ├── temp*_input / temp*_label
    ├── power1_input
    ├── power1_cap
    └── power1_cap_max
    ↓
HwmonTempSensor / platform daemon
    ↓
D-Bus Sensor.Value / Threshold / Availability / OperationalStatus
    ↓
Redfish / IPMI / phosphor-pid-control / phosphor-logging
```

AMD APML modules 主要支援 AMD Family 19h(包含第三代 AMD EPYC Milan)或更新的 server CPU. AMD APML library 也增加 Family 19h model 90h~9Fh 與 Family 1Ah model 00h~0Fh 相關功能; 實際可用功能需以 CPU PPR、platform firmware、kernel driver 與 APML library 版本交叉確認.

### 24.11.4 Porting 前需確認的硬體資料

Intel PECI:

```text
[必填] CPU vendor / generation / socket count
[必填] PECI bus 連到 BMC 哪個 controller
[必填] PECI address：CPU0=0x30、CPU1=0x31...
[必填] PECI pinmux / 電壓準位 / pull-up / routing
[必填] CPU presence / socket ID / board SKU 關係
[必填] Host power state dependency
[必填] BIOS 是否允許 BMC 透過 PECI 取得 thermal telemetry
[建議] CPU package / core / DIMM threshold
[建議] Fan policy 使用哪個 CPU sensor：Die / DTS / Tcontrol / margin
[建議] QEMU 無法驗證 PECI，需規劃實機 validation
```

AMD APML:

```text
[必填] CPU vendor / generation / socket count
[必填] CPU family / model 是否受 APML driver 支援
[必填] APML 使用 I2C 或 I3C
[必填] SB-TSI bus / address
[必填] SB-RMI bus / address
[必填] APML_ALERT_L GPIO 是否接到 BMC
[必填] Host power state dependency
[必填] Platform firmware 是否支援 SB-RMI mailbox command
[建議] DIMM TS0 / TS1 / UMC 對照
[建議] Power cap policy 與 host power budget
[建議] I3C bus support patch / kernel symbol dependency，若平台使用 I3C
```

### 24.11.5 Intel PECI Device Tree 與 Kernel Config

Device Tree 範例:

```dts
&peci0 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_peci0_default>;

    cpu@30 {
        compatible = "intel,peci-client";
        reg = <0x30>;
    };

    cpu@31 {
        compatible = "intel,peci-client";
        reg = <0x31>;
    };
};
```

Kernel config:

```text
CONFIG_PECI=y
CONFIG_PECI_ASPEED=y
CONFIG_SENSORS_PECI_CPUTEMP=y
CONFIG_SENSORS_PECI_DIMMTEMP=y
```

OpenBMC / Yocto 需確認 `dbus-sensors` 的 Intel CPU sensor 有被建入 image. 不同分支的 PACKAGECONFIG 名稱可能略有差異, 需先查該 branch 的 `meson.options` 與 recipe:

```bash
$ bitbake -e dbus-sensors | grep -i intel
$ bitbake-layers show-appends | grep dbus-sensors
```

常見 `.bbappend` 概念:

```bitbake
PACKAGECONFIG:append = " intelcpusensor"
```

### 24.11.6 Intel PECI sysfs 與 IntelCPUSensor 驗證

Kernel / device 檢查:

```bash
$ dmesg | grep -i peci
$ ls -l /dev/peci-* 2>/dev/null
$ ls -l /sys/bus/peci/devices/ 2>/dev/null
```

hwmon 檢查:

```bash
$ for i in /sys/class/hwmon/hwmon*; do
    echo "$i: $(cat "$i/name" 2>/dev/null)"
$ done

$ HWMON=/sys/class/hwmon/hwmonX
$ cat $HWMON/name
$ ls $HWMON/temp*_input
$ cat $HWMON/temp1_label
$ cat $HWMON/temp1_input
$ cat $HWMON/temp1_max 2>/dev/null
$ cat $HWMON/temp1_crit 2>/dev/null
```

`peci-cputemp` 常見 label:

| label | 說明 |
| :--- | :--- |
| `Die` | CPU package die temperature. |
| `DTS` | 依 DTS thermal profile 調整後的 CPU package temperature. |
| `Tcontrol` | Fan Temperature target, 常與 fan policy 相關. |
| `Tthrottle` | Throttle temperature. |
| `Tjmax` | Maximum junction temperature. |
| `Core X` | Per-core temperature. |

IntelCPUSensor service 檢查:

```bash
$ systemctl status xyz.openbmc_project.IntelCPUSensor.service --no-pager
$ journalctl -u xyz.openbmc_project.IntelCPUSensor.service -b --no-pager

$ busctl tree xyz.openbmc_project.IntelCPUSensor
$ busctl tree xyz.openbmc_project.ObjectMapper | grep -Ei 'cpu|dimm|peci'
```

讀取 D-Bus sensor:

```bash
$ busctl get-property   xyz.openbmc_project.IntelCPUSensor   /xyz/openbmc_project/sensors/temperature/CPU0_Temp   xyz.openbmc_project.Sensor.Value   Value
```

### 24.11.7 IntelCPUSensor 偵測狀態機

IntelCPUSensor 有 CPU detection / DIMM readiness 狀態機, 用來處理 CPU 尚未上電、PECI 尚未回應、hwmon device 尚未建立、DIMM 尚未完成 training 等情境.

| 狀態 | 說明 | 常見觀察 |
| :--- | :--- | :--- |
| `OFF` | CPU 尚未偵測到或 PECI ping 失敗 | Host off、CPU absent、PECI pinmux / driver 問題. |
| `ON` | CPU ping 成功, 已讀到 CPU ID, 開始建立 PECI client / hwmon | CPU 已可通訊, 但 DIMM sensor 不一定 ready. |
| `READY` | CPU 與 DIMM thermal path 已就緒 | CPU / DIMM sensor 應可出現在 D-Bus. |

常見流程:

```text
OFF
    CPU ping 成功
    ↓
ON
    讀 CPU ID
    export PECI client device
    等待 peci-cputemp / peci-dimmtemp hwmon
    ↓
READY
    建立 CPU / core / DIMM D-Bus sensors
```

若只看到 CPU sensor, 沒有 DIMM sensor, 需先確認 host BIOS 是否已完成 memory training、DIMM 是否安裝、`peci-dimmtemp` driver 是否建立 hwmon, 以及 IntelCPUSensor 是否持續 rescan.

### 24.11.8 AMD APML Device Tree 與 Kernel Config

AMD APML I2C device 範例:

```dts
&i2c3 {
    status = "okay";
    clock-frequency = <100000>;

    sbtsi@4c {
        compatible = "amd,sbtsi";
        reg = <0x4c>;
    };

    sbrmi@3c {
        compatible = "amd,sbrmi";
        reg = <0x3c>;
    };
};

&i2c4 {
    status = "okay";
    clock-frequency = <100000>;

    sbtsi@48 {
        compatible = "amd,sbtsi";
        reg = <0x48>;
    };

    sbrmi@38 {
        compatible = "amd,sbrmi";
        reg = <0x38>;
    };
};
```

若 SB-RMI driver / platform firmware 支援 DIMM thermal mailbox, 可在 `sbrmi` node 補 `dimm-ids`. 此屬性用於指出 populated UMC instance 的 TS0 mailbox address; TS1 通常由 driver 設定 bit 6 取得.

```dts
sbrmi@3c {
    compatible = "amd,sbrmi";
    reg = <0x3c>;
    dimm-ids = <0x80 0x90 0x81 0x91 0x82 0x92 0x83 0x93>;
};
```

Kernel config:

```text
CONFIG_SENSORS_SBTSI=y
CONFIG_SENSORS_SBRMI=y
```

若使用 AMD APML out-of-tree modules, 需依專案 recipe / kernel source 整合:

```text
CONFIG_AMD_APML_SBTSI=m 或 y
CONFIG_AMD_APML_SBRMI=m 或 y
CONFIG_AMD_APML_ALERTL=m 或 y，若使用 APML_ALERT_L
```

### 24.11.9 AMD APML sysfs 驗證

Driver / device 檢查:

```bash
$ dmesg | grep -Ei 'sbtsi|sbrmi|apml'
$ lsmod | grep -Ei 'sbtsi|sbrmi|apml'
```

I2C 掃描:

```bash
# 範例：socket 0 APML bus
$ i2cdetect -y 3
```

hwmon 檢查:

```bash
$ for i in /sys/class/hwmon/hwmon*; do
    echo "$i: $(cat "$i/name" 2>/dev/null)"
$ done

# SB-TSI CPU temperature
$ HWMON_TSI=/sys/class/hwmon/hwmonX
$ cat $HWMON_TSI/name
$ cat $HWMON_TSI/temp1_input
$ cat $HWMON_TSI/temp1_max 2>/dev/null
$ cat $HWMON_TSI/temp1_min 2>/dev/null

# SB-RMI package power / power cap
$ HWMON_RMI=/sys/class/hwmon/hwmonY
$ cat $HWMON_RMI/name
$ cat $HWMON_RMI/power1_input
$ cat $HWMON_RMI/power1_cap 2>/dev/null
$ cat $HWMON_RMI/power1_cap_max 2>/dev/null

# SB-RMI DIMM thermal channels, if supported
$ ls $HWMON_RMI/temp*_input 2>/dev/null
$ cat $HWMON_RMI/temp1_label 2>/dev/null
$ cat $HWMON_RMI/temp29_label 2>/dev/null
```

AMD APML units:

```text
temp*_input      millidegree Celsius (m°C)
power1_input     microwatt (µW)
power1_cap       microwatt (µW)
power1_cap_max   microwatt (µW)
```

SB-RMI DIMM thermal channel 對應:

| hwmon channel | label pattern | 說明 |
| :--- | :--- | :--- |
| `temp1` ~ `temp16` | `DIMM_TS0_UMC0` ~ `DIMM_TS0_UMC15` | TS0 for UMC 0~15. |
| `temp17` ~ `temp32` | `DIMM_TS1_UMC0` ~ `DIMM_TS1_UMC15` | TS1 for UMC 0~15. |

### 24.11.10 Entity Manager 配置

不同 OpenBMC 分支與平台 JSON schema 有差異; 以下內容作為設計範本, 實際欄位需對齊專案 branch 的 Entity Manager schema 與 sensor daemon 支援項目.

Intel CPU:

```json
{
    "Name": "CPU0",
    "Type": "CPU",
    "Address": 48,
    "Index": 0,
    "MaxReading": 127,
    "MinReading": -10,
    "PowerState": "On",
    "Thresholds": [
        {
            "Name": "upper critical",
            "Direction": "greater than",
            "Severity": 1,
            "Value": 95.0
        },
        {
            "Name": "upper non critical",
            "Direction": "greater than",
            "Severity": 0,
            "Value": 85.0
        }
    ]
}
```

Intel DIMM:

```json
{
    "Name": "DIMM0",
    "Type": "DIMM",
    "Address": 48,
    "Index": 0,
    "MaxReading": 127,
    "MinReading": -10,
    "PowerState": "On",
    "Thresholds": [
        {
            "Name": "upper critical",
            "Direction": "greater than",
            "Severity": 1,
            "Value": 85.0
        }
    ]
}
```

注意: Intel PECI address 在許多 Entity Manager 配置中使用十進位; 例如 `0x30` 寫成 `48`,`0x31` 寫成 `49`. 若 schema 不接受十六進位字串, 使用十進位可避免 service parse 失敗.

AMD SB-TSI:

```json
{
    "Name": "CPU0",
    "Type": "SBTSI",
    "Bus": 3,
    "Address": "0x4C",
    "Index": 0,
    "MaxReading": 127,
    "MinReading": -10,
    "PowerState": "On",
    "Thresholds": [
        {
            "Name": "upper critical",
            "Direction": "greater than",
            "Severity": 1,
            "Value": 95.0
        }
    ]
}
```

AMD SB-RMI power:

```json
{
    "Name": "CPU0_Power",
    "Type": "SBRMI",
    "Bus": 3,
    "Address": "0x3C",
    "Index": 0,
    "MaxReading": 400.0,
    "MinReading": 0.0,
    "PowerState": "On",
    "Thresholds": [
        {
            "Name": "upper critical",
            "Direction": "greater than",
            "Severity": 1,
            "Value": 350.0
        }
    ]
}
```

### 24.11.11 啟動服務與 D-Bus 驗證

Intel PECI:

```bash
$ systemctl status xyz.openbmc_project.IntelCPUSensor.service --no-pager
$ journalctl -u xyz.openbmc_project.IntelCPUSensor.service -b --no-pager

$ busctl tree xyz.openbmc_project.ObjectMapper | grep -Ei 'cpu|dimm|peci'

$ busctl get-property   xyz.openbmc_project.IntelCPUSensor   /xyz/openbmc_project/sensors/temperature/CPU0_Temp   xyz.openbmc_project.Sensor.Value   Value
```

AMD APML:

```bash
$ systemctl status xyz.openbmc_project.HwmonTempSensor.service --no-pager
$ journalctl -u xyz.openbmc_project.HwmonTempSensor.service -b --no-pager

$ busctl tree xyz.openbmc_project.ObjectMapper | grep -Ei 'cpu|dimm|sbrmi|sbtsi|apml'

$ busctl get-property   xyz.openbmc_project.HwmonTempSensor   /xyz/openbmc_project/sensors/temperature/CPU0_Temp   xyz.openbmc_project.Sensor.Value   Value
```

若 AMD power sensor 由 PowerSensor / PSUSensor-like daemon 或平台 daemon 發佈, service owner 可能不是 `HwmonTempSensor`, 需先用 ObjectMapper 找出 owner.

### 24.11.12 Redfish / IPMI / Fan Policy 驗證

Redfish:

```bash
$ curl -k -u root:0penBmc https://<bmc>/redfish/v1/Systems/system/Processors
$ curl -k -u root:0penBmc https://<bmc>/redfish/v1/Chassis/<id>/Thermal
$ curl -k -u root:0penBmc https://<bmc>/redfish/v1/Chassis/<id>/Sensors
$ curl -k -u root:0penBmc https://<bmc>/redfish/v1/Chassis/<id>/ThermalSubsystem
```

IPMI:

```bash
$ ipmitool sdr list | grep -Ei 'cpu|dimm|temp|power'
$ ipmitool sensor | grep -Ei 'cpu|dimm|temp|power'
$ ipmitool sel list
```

Fan policy:

```bash
$ journalctl -b | grep -Ei 'pid|thermal|fan|cpu|dimm|failsafe'
$ busctl tree xyz.openbmc_project.ObjectMapper | grep -Ei 'pid|thermal|fan'
```

驗證重點:

```text
[ ] Host off 時 CPU sensor 狀態符合設計：Unavailable、無 sensor，或保留但不更新
[ ] Host on 後 CPU sensor 會自動出現或恢復更新
[ ] CPU stress test 時 temperature 上升
[ ] Fan policy 使用的 CPU / DIMM sensor path 正確
[ ] 過溫 threshold 能產生 D-Bus alarm / SEL / Journal event
[ ] Redfish / IPMI 顯示與 D-Bus 值一致
```

### 24.11.13 進階除錯與常見陷阱

| 問題現象 | Intel PECI 建議排查方向 | AMD APML 建議排查方向 | 排查 |
| :--- | :--- | :--- | :--- |
| device node 不存在 | PECI controller driver 未載入、DTS disabled、pinmux 錯 | I2C/I3C device 未建立、driver 未啟用 | 查 `dmesg`、DTS、kernel config. |
| `i2cdetect` 看不到 APML address | 不適用 | bus / mux / address / host power state / address select pin | 查 schematic、host power、I2C waveform. |
| hwmon 不存在 | CPU 未上電、PECI client 未 export、cputemp/dimmtemp 未 bind | sbtsi/sbrmi driver 未 bind、APML function 不支援 | 查 `/sys/bus/peci` 或 `/sys/bus/i2c/devices`. |
| CPU temp 有, DIMM temp 無 | BIOS memory training 未完成、DIMM absent、dimmtemp driver 問題 | SB-RMI mailbox command 不支援、dimm-ids 錯 | 查 BIOS log、DIMM population、driver log. |
| 讀值固定 0 或異常 | PECI completion code / host state / scaling 問題 | APML mailbox fail、firmware 不支援、unit 換算錯 | 比對 sysfs、D-Bus、Redfish. |
| service 起不來 | IntelCPUSensor 未建入 image、EM JSON 格式錯 | HwmonTempSensor 未匹配 hwmon、JSON Type 不符 | 查 journal 與 Entity Manager log. |
| Host reboot 後 sensor 沒回來 | rescan / state machine 卡住 | I2C bus stuck、driver 未重新 bind | restart sensor service, 查 bus recovery. |
| Fan 全速 | required CPU sensor unavailable | required CPU sensor unavailable | 查 fan policy required sensor 與 failsafe reason. |
| Redfish 看不到 CPU sensor | association / naming / bmcweb mapping 不完整 | association / naming / bmcweb mapping 不完整 | 查 ObjectMapper、bmcweb log. |

### 24.11.14 CPU Sensor Porting 驗收 Checklist

硬體 / 規格:
- [ ] CPU vendor / model / generation 確認
- [ ] Socket count / socket ID 確認
- [ ] Intel PECI address 或 AMD APML bus/address 確認
- [ ] Host power state dependency 確認
- [ ] BIOS / firmware 支援情況確認
- [ ] CPU / DIMM threshold 確認

Intel PECI:
- [ ] PECI pinmux / DTS / controller status 正確
- [ ] CONFIG_PECI / CONFIG_PECI_ASPEED 啟用
- [ ] CONFIG_SENSORS_PECI_CPUTEMP 啟用
- [ ] CONFIG_SENSORS_PECI_DIMMTEMP 啟用
- [ ] /dev/peci-* 或 /sys/bus/peci/devices 存在
- [ ] peci-cputemp hwmon 存在
- [ ] peci-dimmtemp hwmon 存在, 若平台需要 DIMM temp
- [ ] IntelCPUSensor service 啟動正常

AMD APML:
- [ ] CPU family / model 符合 driver 支援範圍
- [ ] SB-TSI / SB-RMI DTS node 正確
- [ ] CONFIG_SENSORS_SBTSI / CONFIG_SENSORS_SBRMI 啟用
- [ ] 若使用 out-of-tree APML modules, recipe / module load 正常
- [ ] sbtsi hwmon temp 可讀
- [ ] sbrmi hwmon power 可讀
- [ ] DIMM TS0 / TS1 channel 可讀, 若 firmware 支援
- [ ] APML_ALERT_L GPIO 驗證完成, 若平台使用

Entity Manager / Userspace:
- [ ] CPU / DIMM / SBTSI / SBRMI JSON 加入 image
- [ ] Address 格式符合 schema
- [ ] PowerState 設定符合 host state dependency
- [ ] Thresholds 設定完成
- [ ] D-Bus sensor path 命名符合平台規範

整合驗證:
- [ ] Host off / on / reboot sensor 行為符合設計
- [ ] busctl 可讀取 CPU / DIMM / power Value
- [ ] Redfish 可看到 CPU / DIMM / thermal sensors
- [ ] IPMI SDR / sensor reading 正常
- [ ] CPU stress test 後溫度上升合理
- [ ] Sensor missing 會觸發 fan failsafe 或 degraded policy
- [ ] 過溫事件可產生 Journal / SEL

### 24.11.15 CPU Sensor 資料表範本

| Socket | Vendor | Interface | Bus / Controller | Address | Driver | hwmon name | Label | sysfs | D-Bus Path | PowerState | 備註 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| CPU0 | Intel | PECI | peci0 | 0x30 | peci-cputemp | [待填] | Die | tempX_input | /xyz/openbmc_project/sensors/temperature/CPU0_Temp | On | [待填] |
| CPU0 | Intel | PECI | peci0 | 0x30 | peci-dimmtemp | [待填] | DIMM CI | tempX_input | /xyz/openbmc_project/sensors/temperature/DIMM0_Temp | On | [待填] |
| CPU0 | AMD | SB-TSI | i2c3 | 0x4C | sbtsi | [待填] | temp1 | temp1_input | /xyz/openbmc_project/sensors/temperature/CPU0_Temp | On | [待填] |
| CPU0 | AMD | SB-RMI | i2c3 | 0x3C | sbrmi | [待填] | power1 | power1_input | /xyz/openbmc_project/sensors/power/CPU0_Power | On | [待填] |

### 24.11.16 本節參考資料

- Linux Kernel Documentation - peci-cputemp: https://docs.kernel.org/hwmon/peci-cputemp.html
- Linux Kernel Documentation - peci-dimmtemp: https://mjmwired. net/kernel/Documentation/hwmon/peci-dimmtemp. rst
- Linux Kernel Documentation - sbrmi: https://www.kernel.org/doc/html/latest/hwmon/sbrmi.html
- OpenBMC dbus-sensors README: https://github.com/openbmc/dbus-sensors
- OpenBMC Intel CPU Sensors: https://deepwiki. com/openbmc/dbus-sensors/3.1.2-intel-cpu-sensors
- AMD APML modules: https://github.com/amd/apml_modules
- AMD APML Library: https://www. amd. com/en/developer/e-sms/apml-library.html
- OpenBMC Entity Manager: https://github.com/openbmc/entity-manager

### 24.11.17 回查結果

本節依 CPU Sensor / PECI / APML 需求補強後, 已完成下列回查:

```text
[x] 是否說明 Intel PECI 與 AMD APML 差異
[x] 是否補上 Intel PECI 資料路徑與 Linux hwmon 對應
[x] 是否補上 AMD SB-TSI / SB-RMI / APML_ALERT_L 架構
[x] 是否補上 Device Tree、kernel config、Yocto / service 驗證
[x] 是否補上 IntelCPUSensor OFF / ON / READY 狀態機
[x] 是否補上 AMD APML DIMM TS0 / TS1 channel 對應
[x] 是否補上 Entity Manager JSON 範本與 address 格式提醒
[x] 是否補上 Host power state dependency
[x] 是否補上 Redfish / IPMI / Fan policy 驗證
[x] 是否補上常見陷阱、驗收 checklist 與資料表範本
```

上述項目已補齊, 暫無需回到資料蒐集階段.

<a id="section-12-12"></a>

## 24.12 NVMe Sensor

NVMe sensor 用於監控 NVMe SSD 的溫度、健康狀態、SMART warning、slot inventory 與 firmware 資訊. BMC 可能透過 SMBus Basic Management Command、NVMe-MI over MCTP、host proxy 或 vendor daemon 取得資料. Porting 重點包含 drive slot mapping、presence、endpoint、timeout、inventory 與 Redfish / IPMI 對外路徑.

| 檢查項目 | 說明 |
| --- | --- |
| Slot / Location | U.2、E1. S、M.2、EDSFF slot 編號與絲印需一致 |
| Presence source | PCIe PRSNT#、CPLD、I2C ACK 或 MCTP endpoint |
| Transport | SMBus、MCTP over SMBus、MCTP over PCIe VDM、host proxy |
| Bus / Address / EID | 實際通訊路徑、mux channel 與 endpoint ID |
| Timeout / PollRate | 避免單顆 drive timeout 阻塞整組 sensor |
| Inventory | model、serial、firmware、capacity、location |
| Event | insert/remove、over-temp、SMART warning、timeout |

常用驗證:

```bash
$ i2cdetect -l
$ i2cdetect -y <bus>
$ systemctl status mctpd --no-pager 2>/dev/null || true
$ busctl tree xyz.openbmc_project.MCTP 2>/dev/null || true
$ systemctl list-units '*nvme*' --no-pager
$ journalctl -u xyz.openbmc_project.nvmesensor.service -b --no-pager 2>/dev/null | tail -200
$ busctl tree /xyz/openbmc_project/sensors | grep -i nvme
$ busctl tree /xyz/openbmc_project/inventory | grep -Ei 'nvme|drive|ssd'
```

常見問題: D-Bus 有 inventory 但沒有 temperature, 多半需回查 endpoint discovery、sensor daemon PACKAGECONFIG、Entity Manager config 與 association; 熱插拔後若 object 沒消失, 需檢查 presence signal 與 daemon state machine; host proxy 類 sensor 需處理 host reset 後 stale data.

<a id="section-12-13"></a>

## 24.13 GPU Sensor

GPU / accelerator sensor 常見於 NVIDIA GPU、DPU、AI accelerator、PCIe switch 或 retimer. 資料來源可能是 MCTP vendor defined message、I2C sideband、SMBPBI、host proxy 或 vendor daemon. 常見 sensor 包含 temperature、power、energy、voltage、utilization、clock、PCIe link、error event 與 power limit.

| 類別 | D-Bus / Redfish 方向 | 注意事項 |
| --- | --- | --- |
| Temperature | `temperature` | GPU core、memory、board、retimer sensor 需分清楚 |
| Power | `power` | instantaneous、average、peak 定義需固定 |
| Energy | `energy` | counter wrap、reset 後行為需定義 |
| Voltage / Current | `voltage` / `current` | rail name 與 GPU domain 對齊 |
| Utilization | `utilization` | GPU / memory / engine 類型需明確 |
| Control | power cap / clock limit | 權限、persistency、host policy |

常用驗證:

```bash
$ systemctl list-units '*gpu*' '*mctp*' --no-pager
$ journalctl -u xyz.openbmc_project.nvidiagpusensor.service -b --no-pager 2>/dev/null | tail -200
$ busctl tree /xyz/openbmc_project/sensors | grep -Ei 'gpu|accelerator|pcie'
$ busctl tree /xyz/openbmc_project/inventory | grep -Ei 'gpu|accelerator|pcie'
$ busctl introspect <service> <gpu-object-path>
```

常見問題包含 mW/W 或 µW/W 換算錯、GPU reset 後 sensor stale、MCTP endpoint recovery 不完整、Redfish association 缺失, 以及 fan policy 因 required GPU temp unavailable 進入 failsafe.

<a id="section-12-14"></a>

## 24.14 External / Virtual Sensor

External sensor 是由外部來源寫入 BMC D-Bus 的 sensor, 例如 host OS、BIOS、另一顆 BMC 或 vendor daemon. Virtual sensor 則由既有 sensor 與公式計算, 例如 total power、zone max temperature、average inlet temperature、redundant voting 或 power efficiency.

External sensor 需定義:
- Writer owner 與 D-Bus 權限.
- `Timeout` 與 stale data 行為.
- `MinValue` / `MaxValue` 與錯誤值過濾.
- host off、host reset、writer daemon crash 後是否保留最後值、變 NaN、`Available=false` 或 `Functional=false`.

Virtual sensor 設計原則:
- input D-Bus path、單位與更新率要明確.
- 先統一單位, 避免 `µW + W` 或 `mV + V`.
- total power 需避免 double counting.
- 若參與 fan / power policy, 需定義任一 input unavailable 時的行為.

常用驗證:

```bash
$ systemctl status xyz.openbmc_project.externalsensor.service --no-pager 2>/dev/null || true
$ journalctl -u xyz.openbmc_project.externalsensor.service -b --no-pager 2>/dev/null | tail -100
$ systemctl status phosphor-virtual-sensor.service --no-pager 2>/dev/null || true
$ journalctl -u phosphor-virtual-sensor.service -b --no-pager 2>/dev/null | tail -100
$ busctl tree /xyz/openbmc_project/sensors | grep -Ei 'external|total|max|avg|virtual'
```

<a id="section-12-15"></a>

## 24.15 Presence / Intrusion / GPIO State Sensor

Presence、intrusion 與 GPIO state 類 sensor 多半是 Boolean 或 enum, 而不是連續讀值. 這類訊號會影響 inventory、hot-swap、LED、event log、Redfish chassis 狀態、IPMI discrete sensor 與維修流程, 因此需區分 `Present`、`Functional`、`Available`、`Fault`、`Intrusion`.

| 類型 | 來源 | 用途 | 注意事項 |
| --- | --- | --- | --- |
| FRU presence | GPIO、CPLD bit、EEPROM ACK、PMBus ACK | PSU、fan tray、riser、backplane 是否插入 | active level、debounce、hot-swap event |
| Chassis intrusion | GPIO、PCH/SMBus、hwmon alarm | 機箱開蓋偵測 | Automatic / Manual rearm |
| Fault state | GPIO、CPLD latch、PMBus STATUS_WORD | PSU fault、fan fault、VR fault | latch clear / W1C rule |
| Board / SKU state | GPIO strap、ADC strap、CPLD register | board ID、SKU ID、revision | boot-time 與 runtime 可能不同 |

常用驗證:

```bash
$ gpiodetect
$ gpioinfo | grep -Ei 'present|intrusion|fault|psu|fan'
$ gpioget $(gpiofind CHASSIS_INTRUSION_N)
$ systemctl status xyz.openbmc_project.intrusionsensor.service --no-pager 2>/dev/null || true
$ journalctl -u xyz.openbmc_project.intrusionsensor.service -b --no-pager 2>/dev/null | tail -100
$ busctl tree /xyz/openbmc_project/inventory | grep -Ei 'psu|fan|chassis|riser|drive'
$ ipmitool sdr elist | grep -Ei 'presence|intrusion|fault'
```

<a id="section-12-16"></a>

## 24.16 Redfish Association

Redfish 是否顯示 sensor, 不只取決於 D-Bus 上是否存在 sensor object, 也取決於 ObjectMapper association、inventory path、Chassis 關聯, 以及 bmcweb 對該 sensor type 的路徑掃描策略. D-Bus 有值但 Redfish `N/A` 或缺項, 常見方向是 association 缺失或 sensor 被掛到錯的 chassis.

| Association | 用途 |
| --- | --- |
| `chassis` / `all_sensors` | 讓 Redfish Chassis 收到 sensor |
| `inventory` / `sensors` | 讓 FRU / board / PSU / fan 與 sensor 關聯 |
| `sensors` / `inventory` | event log / callout 回查 inventory |
| `contained_by` / `containing` | 建立 chassis / board / module topology |

常用檢查:

```bash
$ busctl introspect <service> <sensor-path> xyz.openbmc_project.Association.Definitions
$ busctl get-property <service> <sensor-path> xyz.openbmc_project.Association.Definitions Associations
$ journalctl -u bmcweb -b --no-pager | grep -Ei 'sensor|association|chassis|redfish' | tail -200
$ curl -k -u root:0penBmc https://<bmc>/redfish/v1/Chassis/<id>/Sensors
$ curl -k -u root:0penBmc https://<bmc>/redfish/v1/Chassis/<id>/Thermal
$ curl -k -u root:0penBmc https://<bmc>/redfish/v1/Chassis/<id>/Power
```

<a id="section-12-17"></a>

## 24.16.1 Redfish 可見性判斷流程

```mermaid
flowchart TD
    A[D-Bus sensor object exists?] -->|No| B[Check source / config / daemon]
    A -->|Yes| C[Value + Unit + State interfaces valid?]
    C -->|No| D[Fix interface or property type]
    C -->|Yes| E[Association points to inventory/chassis?]
    E -->|No| F[Create inventory and chassis association]
    E -->|Yes| G[bmcweb maps this sensor type/path?]
    G -->|No| H[Check schema support and route policy]
    G -->|Yes| I[Redfish resource visible]
    I --> J{Reading or Health wrong?}
    J -->|Reading| K[Check unit / scale / cache]
    J -->|Health| L[Check Available / Functional / threshold]
```

## 24.17 Sensor 共用除錯指令與附錄

### 24.17.1 分層排查決策圖

```mermaid
flowchart TD
    A[Sensor 異常] --> B{硬體來源有回應?}
    B -->|No| B1[Power / Presence / Bus / Address / Signal]
    B -->|Yes| C{Kernel driver bound?}
    C -->|No| C1[DTS / compatible / Kconfig / probe log]
    C -->|Yes| D{sysfs value valid?}
    D -->|No| D1[Channel / Page / raw format / driver scale]
    D -->|Yes| E{D-Bus object exists?}
    E -->|No| E1[Entity Manager / Probe / Type / service log]
    E -->|Yes| F{Value and state correct?}
    F -->|No| F1[Unit / ScaleFactor / timeout / PowerState]
    F -->|Yes| G{Redfish or IPMI visible?}
    G -->|No| G1[Association / chassis mapping / SDR policy]
    G -->|Yes| H{Event and policy correct?}
    H -->|No| H1[Threshold / hysteresis / logging / consumer path]
    H -->|Yes| I[驗收完成]
```

### 24.17.2 文件完成條件

一筆 sensor 資料不應只填名稱與讀值。提交 review 前至少需完成：

- 硬體來源、bus/address/channel/page、power domain 與 presence 條件。
- sysfs path、原始單位、D-Bus 單位、完整換算式與校準依據。
- D-Bus service/path/interface、Redfish URI、IPMI policy（若適用）。
- Warning/Critical 門檻、hysteresis、assert/deassert 與事件行為。
- Poll interval、timeout、stale、bus error、device removal 與恢復策略。
- Inventory/chassis association、producer、consumer、validation owner。
- 冷開機、host power cycle、BMC reboot、service restart、異常注入與恢復測試結果。

> 表格中的 `[待填]` 應保留給平台實測值，不宜用通用假設代填；合併前應由 HW、BMC、Thermal/Power 與 Validation owner 共同關閉。

建議依序檢查: hardware → kernel/sysfs → config → service → D-Bus → Redfish/IPMI → event/policy.

版本與 baseline:

```bash
$ cat /etc/os-release
$ cat /etc/timestamp 2>/dev/null || true
$ uname -a
$ cat /proc/cmdline
$ systemctl --failed --no-pager
```

Kernel / sysfs:

```bash
$ for h in /sys/class/hwmon/hwmon*; do
    echo "== $h =="
    cat "$h/name" 2>/dev/null || true
    ls "$h" | grep -E '^(in|temp|fan|pwm|curr|power|energy)[0-9]+' || true
$ done
$ find /sys/class/hwmon -maxdepth 2 -type f | grep -E '/(name|in[0-9]+_input|temp[0-9]+_input|fan[0-9]+_input|pwm[0-9]+|curr[0-9]+_input|power[0-9]+_input)$' | sort
```

Entity Manager / service:

```bash
$ ls -l /usr/share/entity-manager/configurations/ 2>/dev/null
$ systemctl status xyz.openbmc_project.EntityManager.service --no-pager
$ journalctl -u xyz.openbmc_project.EntityManager.service -b --no-pager | tail -200
$ systemctl list-units '*sensor*' '*hwmon*' '*fan*' '*pid*' '*power*' --no-pager
```

D-Bus:

```bash
$ busctl tree /xyz/openbmc_project/sensors
$ busctl introspect <service> <object-path>
$ busctl get-property <service> <object-path> xyz.openbmc_project.Sensor.Value Value
$ busctl get-property <service> <object-path> xyz.openbmc_project.Sensor.Value Unit
$ busctl get-property <service> <object-path> xyz.openbmc_project.State.Decorator.Availability Available
$ busctl get-property <service> <object-path> xyz.openbmc_project.State.Decorator.OperationalStatus Functional
```

Redfish / IPMI:

```bash
$ curl -k -u root:0penBmc https://<bmc>/redfish/v1/Chassis/<id>/Sensors
$ curl -k -u root:0penBmc https://<bmc>/redfish/v1/Chassis/<id>/Thermal
$ curl -k -u root:0penBmc https://<bmc>/redfish/v1/Chassis/<id>/Power
$ ipmitool sdr elist
$ ipmitool sensor list
$ ipmitool sel list
```

單位速查:

| sysfs / raw | 常見單位 | D-Bus / Redfish 建議單位 | 換算 |
| --- | --- | --- | --- |
| `temp*_input` | milli-degree C | DegreesC | `/1000` |
| `in*_input` | mV | Volts | `/1000 × ScaleFactor` |
| `curr*_input` | mA | Amperes | `/1000` |
| `power*_input` | µW | Watts | `/1000000` |
| `fan*_input` | RPM | RPMS | 通常不換算 |
| `pwm*` | 0–255 | Percent 或 raw | `raw /255 × 100` |

常見分層排查:

| 現象 | 優先分層 | 第一輪檢查 |
| --- | --- | --- |
| D-Bus 沒 sensor | config / daemon / sysfs | Entity Manager log、sensor daemon log、sysfs 是否存在 |
| sysfs 有值但 D-Bus 無值 | daemon mapping | daemon 是否支援該 hwmon name / label / index |
| D-Bus 有值但 Redfish 無值 | association / bmcweb | ObjectMapper association、bmcweb log、Chassis path |
| D-Bus 有值但 IPMI 無 SDR | ipmid mapping | `ipmitool sdr`、ipmid dbus-sdr log |
| 數值差 1000 倍 | unit / scale | sysfs raw、ScaleFactor、D-Bus Unit、Redfish output |
| fan 全速 | required sensor unavailable | fan control log、Available / Functional |

### 24.17.4 本章參考資料

- OpenBMC docs - Sensor Architecture: https://github.com/openbmc/docs/blob/master/architecture/sensor-architecture.md
- OpenBMC dbus-sensors README: https://github.com/openbmc/dbus-sensors
- OpenBMC entity-manager README / docs: https://github.com/openbmc/entity-manager
- OpenBMC phosphor-hwmon README: https://github.com/openbmc/phosphor-hwmon
- OpenBMC phosphor-virtual-sensor README: https://github.com/openbmc/phosphor-virtual-sensor
- Linux kernel hwmon sysfs interface: https://docs.kernel.org/hwmon/sysfs-interface.html
- OpenBMC bmcweb Redfish sensor implementation: https://github.com/openbmc/bmcweb/blob/master/redfish-core/lib/sensors. hpp
