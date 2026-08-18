# 27. Inventory, FRU 與 Asset 資料模型

## 適用範圍

本文件說明 OpenBMC 的 Inventory, FRU 與 Asset 資料模型, 涵蓋穩定身分, FRU EEPROM 與 IPMI FRU 格式, 資料權威端, D-Bus inventory object, 狀態, association, Redfish/IPMI 映射, hot-plug, 持久化, 資料保護與排查流程.

## 適用讀者

- 負責 OpenBMC inventory, FRU, Entity Manager, D-Bus, Redfish, IPMI, 製造資料或平台 bring-up 的開發與整合人員.
- 執行 FRU EEPROM 驗證, 資產資料映射, hot-plug 測試, firmware update, factory reset 或資料一致性排查的人員.

## 快速導覽

- [理解 Inventory, FRU 與 Asset](#151-inventory-是什麼): 實體元件, FRU data, FRU EEPROM 與 Asset 欄位.
- [定義資料來源與權威端](#154-資料來源與權威端): 欄位優先順序, fallback, 衝突與 unknown.
- [追蹤 OpenBMC Inventory 資料流](#155-openbmc-inventory-資料流): fru-device, Entity Manager, D-Bus 與 ObjectMapper.
- [確認狀態與硬體拓樸](#157-presentavailablefunctional-與-health): Present, Available, Functional, Health 與 association.
- [核對 Redfish 與 IPMI](#1510-redfish-與-ipmi): Inventory, Redfish resource 與 IPMI FRU/SDR 映射.
- [執行排查與驗收](#1515-target-端排查流程): FRU service, inventory object, Debug Log, bring-up 與 Checklist.

Inventory 是 BMC 對系統中實體元件的描述. 它記錄系統有哪些主板, PSU, 風扇, riser, CPU, DIMM 與其他元件, 也保存這些元件的名稱, 位置, 料號, 序號, 版本, 是否存在, 以及彼此之間的關係.

FRU 通常保存元件本身的識別資料; Asset 則是 Inventory 中用來描述製造商, 型號, 料號, 序號與資產標籤的一組資料. OpenBMC 會把這些來源整理成 D-Bus inventory objects, 再提供給 Redfish, IPMI 與其他服務使用.

## 27.1 Inventory 是什麼

Inventory 可以先理解成 BMC 內部的硬體清單, 但它不只列出名稱. 每個項目還需要回答:

- 這是什麼元件?
- 位於哪一個 chassis, board 或 slot?
- 目前是否插入?
- 是否正常工作?
- 製造商, 型號, 料號與序號是什麼?
- 哪些 sensors 屬於這個元件?
- Redfish 與 IPMI 應如何呈現?

例如一顆 PSU 的 inventory 可能包含:

```text
Identity       PSU0
Location       Chassis PSU slot 0
Present        true
Functional     true
Manufacturer   Example Power
Model          1600W-PSU
PartNumber     PWR-1600-01
SerialNumber   PSU12345678
Sensors        InputPower、OutputPower、Temperature
```

Inventory 的重點是建立一個穩定的元件身分. PSU 更換後, `PSU0` 仍代表同一個實體插槽, 但 Manufacturer, Model 與 SerialNumber 會依新插入的 PSU 更新.

### 27.1.1 Inventory Item 與實體元件

一個 inventory item 通常對應一個可辨識的實體元件, 例如:

- System
- Chassis
- Baseboard
- BMC module
- PSU
- Fan tray
- Riser
- Backplane
- CPU
- DIMM
- Drive
- NIC
- CPLD / FPGA

不是所有 sensor 都需要成為獨立 inventory item. 例如 PSU 輸入電壓是 PSU 的 sensor, 不是一個可更換元件. 它應透過 association 連回 PSU inventory item.

### 27.1.2 穩定身分與目前內容

Inventory path 應代表穩定位置或元件角色, 不應使用會改變的資料命名.

適合:

```text
/xyz/openbmc_project/inventory/system/chassis/psu0
/xyz/openbmc_project/inventory/system/chassis/fan3
```

不適合:

```text
.../hwmon7
.../i2c_33_0058
.../PSU_SERIAL_12345678
```

Hwmon index, I2C bus number 與序號都可能改變. 若 object path 跟著改, Redfish, sensor association 與其他服務也可能失去對應關係.

## 27.2 FRU 是什麼

FRU 是 Field Replaceable Unit, 指可在維修現場更換的元件. 常見 FRU 包括:

- Baseboard
- PSU
- Fan tray
- Riser
- Backplane
- Front panel board
- GPU carrier

FRU 也常用來指儲存在元件 EEPROM 中的 FRU information. 這些資料可以包含製造商, 產品名稱, 料號, 序號, 製造日期與自訂欄位.

需要區分:

```text
FRU 元件
可更換的實體硬體

FRU data
描述該元件的識別資料

FRU EEPROM
保存 FRU data 的記憶體
```

元件可以是 FRU, 但不一定有 EEPROM; 有 EEPROM 的板件, 也不一定在產品維修流程中被視為獨立 FRU.

### 27.2.1 FRU EEPROM

FRU EEPROM 通常位於 I2C bus, 常見資料包括:

- EEPROM 型號與容量
- I2C bus, mux path 與 7-bit address
- Address width
- Page size
- Write-protect pin
- FRU binary format

DTS 範例:

```dts
&i2c5 {
    status = "okay";

    eeprom@50 {
        compatible = "atmel,24c64";
        reg = <0x50>;
        pagesize = <32>;
    };
};
```

EEPROM driver 提供原始資料存取能力; FRU parser 再解析其中的 FRU 格式. EEPROM 可讀不代表 FRU data 一定正確, 仍需檢查 header, area offset, length 與 checksum.

### 27.2.2 IPMI FRU 格式

IPMI FRU data 由 Common Header 指向不同資料區域:

```text
Common Header
├── Internal Use Area
├── Chassis Info Area
├── Board Info Area
├── Product Info Area
└── MultiRecord Area
```

各區域用途:

| FRU Area | 常見內容 |
|---|---|
| Chassis | Chassis type, part number, serial number |
| Board | Manufacturer, product name, serial, part number, manufacturing time |
| Product | System / product name, part number, version, serial, asset tag |
| MultiRecord | Power, management access 或 OEM records |

Common Header 與各 area 通常都有 checksum. Checksum 錯誤可能來自寫入中斷, 長度錯誤, offset 錯誤或資料遭破壞.

### 27.2.3 Board 與 Product Area 不相同

Board area 描述主板; Product area 通常描述整機或產品. 兩者都可能有 Manufacturer, Product Name, Part Number 與 Serial Number, 但語意不同.

```text
Board SerialNumber
主板序號

Product SerialNumber
整機序號
```

若把兩者混用, Redfish ComputerSystem, Chassis 與 IPMI FRU 可能顯示不同序號. 平台必須明確定義每個對外欄位取自哪一個 area.

## 27.3 Asset 是什麼

Asset 是 Inventory 中的識別資料, 常見欄位包括:

- Manufacturer
- Model
- PartNumber
- SerialNumber
- AssetTag

這些欄位看起來相似, 但用途不同:

| 欄位 | 意義 |
|---|---|
| Manufacturer | 元件或產品的製造商 |
| Model | 對外使用的型號 |
| PartNumber | 生產, 採購或維修使用的料號 |
| SerialNumber | 單一實體的唯一序號 |
| AssetTag | 使用者或資產管理系統設定的標籤 |

`AssetTag` 可能允許管理者修改; `SerialNumber` 與 `PartNumber` 通常屬於工廠資料, 不應由一般使用者任意改寫.

### 27.3.1 VPD

VPD (Vital Product Data) 是元件的重要識別與製造資料, 概念上和 FRU data 部分重疊. 來源可能是:

- FRU EEPROM
- PCIe VPD
- SMBIOS
- SPD
- PMBus manufacturer commands
- CPLD register
- Secure provisioning storage

VPD 不是單一固定格式. 文件中應說明來源與 parser, 而不是只寫「資料來自 VPD」.

## 27.4 資料來源與權威端

同一欄位可能同時出現在 FRU EEPROM, PMBus, SMBIOS, JSON 與工廠資料庫. 若沒有指定權威端, D-Bus, Redfish 與 IPMI 可能各自選到不同值.

權威端是某個欄位最終應採信的來源. 例如:

| 欄位 | 建議權威端 | Fallback |
|---|---|---|
| System SerialNumber | Factory provisioned Product FRU | Secure manufacturing store |
| Baseboard PartNumber | Board FRU | Board ID lookup table |
| PSU Model | PSU FRU | PMBus `MFR_MODEL` |
| PSU SerialNumber | PSU FRU | PMBus `MFR_SERIAL` |
| CPLD Version | CPLD version register | Update manifest |
| AssetTag | Persistent user setting | FRU Product Asset Tag |

### 27.4.1 Fallback

Fallback 只在權威端不可用時使用. 例如 PSU FRU 解析失敗, 才使用 PMBus `MFR_MODEL`. 不可讓較低優先來源在正常情況下覆蓋工廠資料.

### 27.4.2 衝突

若兩個來源都有值但不一致, 建議:

1. 保留兩個來源的原始值供診斷.
2. 依優先順序選擇對外值.
3. 記錄 conflict event 或 journal.
4. 不自動回寫並覆蓋任一來源.
5. 交由製造或維修流程確認.

### 27.4.3 Unknown 與假資料

缺少序號時可顯示 unknown 或 unavailable, 不應使用固定假序號. 假資料容易被誤認為真實資產資料, 並干擾 RMA, 維修與自動化資產管理.

## 27.5 OpenBMC Inventory 資料流

OpenBMC 的 inventory 可能由多個 services 共同建立:

```mermaid
flowchart TB
    A["FRU EEPROM / GPIO / PMBus / SMBIOS / MCTP / static config"] --> B["fru-device / Entity Manager / platform daemon / host inventory service"]
    B --> C["D-Bus inventory objects"]
    C --> D["ObjectMapper 與 associations"]
    D --> E["bmcweb / ipmid / sensor / logging / policy services"]
    E --> F["Redfish / IPMI"]
```

### 27.5.1 fru-device

`fru-device` 常用來尋找 I2C FRU EEPROM, 解析 IPMI FRU 格式, 並將欄位發布到 D-Bus. 它的輸入是 EEPROM 中的 FRU binary, 輸出則是可供 Entity Manager 或其他服務使用的 FRU properties.

需要確認:

- 掃描哪些 adapters 與 addresses.
- 是否有 blocklist.
- Mux child adapter 是否已建立.
- FRU checksum 與 field parsing.
- Hot-plug 後是否重新掃描.
- 同一 FRU 是否被重複發布.

### 27.5.2 Entity Manager

Entity Manager 使用 JSON configuration 描述實體元件與它提供的功能. 常見內容包括:

- `Name`
- `Type`
- `Probe`
- `Exposes`

`Probe` 用來判斷某份設定是否適用; `Exposes` 描述此元件提供的 sensors, inventory item 或其他裝置設定.

簡化範例:

```json
{
  "Name": "Example Baseboard",
  "Probe": "xyz.openbmc_project.FruDevice({'BOARD_PRODUCT_NAME': 'EXAMPLE_BOARD'})",
  "Type": "Board",
  "Exposes": [
    {
      "Name": "Baseboard",
      "Type": "InventoryItem",
      "PrettyName": "Example Baseboard"
    }
  ]
}
```

Probe 應使用穩定, 可由量產控制的欄位, 例如 board product name, part number 或 SKU ID. Serial number 每台都不同, 不適合用來選擇平台設定.

### 27.5.3 ObjectMapper

ObjectMapper 用來查詢某個 object path 由哪個 service 提供, 以及它有哪些 interfaces. Redfish, IPMI, sensor 與其他 daemon 可透過 ObjectMapper 找到 inventory objects.

如果 object path, interface 或 association 改變, 所有 consumers 都可能受影響, 因此 inventory D-Bus model 應維持穩定.

## 27.6 D-Bus Inventory Object

Inventory objects 通常位於:

```text
/xyz/openbmc_project/inventory/
```

範例:

```text
/xyz/openbmc_project/inventory/system
/xyz/openbmc_project/inventory/system/chassis
/xyz/openbmc_project/inventory/system/chassis/motherboard
/xyz/openbmc_project/inventory/system/chassis/psu0
/xyz/openbmc_project/inventory/system/chassis/fan0
```

### 27.6.1 Interface

一個 object 可同時提供多個 interfaces, 例如:

```text
xyz.openbmc_project.Inventory.Item
xyz.openbmc_project.Inventory.Decorator.Asset
xyz.openbmc_project.Inventory.Item.PowerSupply
xyz.openbmc_project.State.Decorator.OperationalStatus
```

`Inventory.Item` 表示這是一個 inventory item; `Decorator.Asset` 提供 Manufacturer, Model, PartNumber, SerialNumber 等欄位; 元件類型 interface 則說明它是 PSU, Fan, Board, CPU 或 DIMM.

### 27.6.2 Object Path 命名

Object path 應:

- 穩定.
- 使用 slot 或機構名稱.
- 和 service manual 的位置一致.
- 不依賴 hwmon index, I2C bus number 或 discovery order.
- 不使用可更換元件的 serial number.

PSU 換新後, object path 仍是 `psu0`; Asset properties 更新為新元件資料.

## 27.7 Present, Available, Functional 與 Health

這四種狀態描述不同問題.

### 27.7.1 Present

`Present` 表示實體是否存在. 例如 PSU 是否插入, DIMM 是否安裝, fan tray 是否在位.

Presence 來源可能是:

- GPIO
- CPLD bit
- FRU EEPROM ACK
- PMBus response
- MCTP / PLDM discovery
- Host inventory

存在判斷應選擇最可靠且符合硬體設計的來源. 僅以一次 I2C timeout 判定拔除, 可能造成誤判.

### 27.7.2 Available

Available 表示目前是否能取得資料. 例如 PSU 仍插著, 但 PMBus 暫時 timeout:

```text
Present    true
Available  false
```

這通常需要 retry 與 event debounce, 不應立即將 inventory 移除.

### 27.7.3 Functional

Functional 表示元件是否能正常提供功能. 例如:

```text
Fan tray 已插入        Present=true
Fan tach 長時間為 0    Functional=false
```

PSU 有 fault 時也可能是 `Present=true`, `Functional=false`.

### 27.7.4 Health

Health 是對多項狀態的聚合結果, 通常呈現為 OK, Warning 或 Critical. 它不應直接等於單一 GPIO 或單一 sensor threshold.

### 27.7.5 狀態組合

| 情況 | Present | Available | Functional |
|---|---:|---:|---:|
| 元件未插入 | false | false | 視介面設計 |
| 元件正常 | true | true | true |
| 通訊暫時失敗 | true | false | 未必立即 false |
| 元件確認故障 | true | 可能 true | false |

當 `Present=false` 時, 不應繼續產生該元件的 sensor threshold critical event.

## 27.8 Association 與硬體拓樸

Association 用來描述 D-Bus objects 之間的關係. 例如:

- PSU0 位於 chassis.
- PSU0 的 sensors 屬於 PSU0.
- Fan0 位於 fan tray.
- Temperature sensor 屬於 baseboard.

```text
Chassis
├── Motherboard
│   ├── BMC
│   └── Riser0
├── PSU0
│   ├── InputPower sensor
│   └── Temperature sensor
└── FanTray0
    ├── Fan0
    └── Fan1
```

常見 association 名稱包括:

- `contained_by` / `containing`
- `inventory` / `sensors`
- `chassis` / `all_sensors`
- `powered_by`
- `cooled_by`

Association 會影響 Redfish 資源位置, sensor 歸屬, event location 與 power / thermal policy. 可插拔元件移除時, 相關 sensor associations 也要更新, 避免留下 orphan objects.

### 27.8.1 檢查 Association

```bash
$ busctl tree xyz.openbmc_project.ObjectMapper | grep -i inventory

$ busctl introspect <service> <object_path>

$ busctl get-property \
    <service> \
    <object_path> \
    xyz.openbmc_project.Association.Definitions \
    Associations
```

## 27.9 FRU 欄位如何映射到 Inventory

常見映射:

| FRU Area | FRU Field | Inventory / Asset |
|---|---|---|
| Chassis | Part Number | Chassis PartNumber |
| Chassis | Serial Number | Chassis SerialNumber |
| Board | Manufacturer | Baseboard Manufacturer |
| Board | Product Name | Baseboard Model / PrettyName |
| Board | Part Number | Baseboard PartNumber |
| Board | Serial Number | Baseboard SerialNumber |
| Product | Manufacturer | System Manufacturer |
| Product | Product Name | System Model |
| Product | Part Number | System PartNumber |
| Product | Serial Number | System SerialNumber |
| Product | Asset Tag | AssetTag |

這張表必須依產品定義確認, 不能只依欄位名稱自動假設. 尤其 Board Product Name, Product Name, Chassis Part Number 與 System Model 很容易混淆.

## 27.10 Redfish 與 IPMI

Redfish 與 IPMI 使用不同的資料模型.

Redfish 以資源表達系統:

- `ComputerSystem`
- `Chassis`
- `Manager`
- `PowerSupply`
- `Fan`
- `Processor`
- `Memory`
- `Assembly`

IPMI 則以 FRU areas, FRU IDs, SDR 與 SEL 為主. 兩者可使用相同原始資料, 但映射方式不一定一對一.

### 27.10.1 常見對照

| Inventory | Redfish | IPMI |
|---|---|---|
| System | ComputerSystem | Product FRU |
| Chassis | Chassis | Chassis FRU |
| Baseboard | Chassis / Assembly | Board FRU |
| PSU | PowerSupply | FRU + sensors |
| Fan | Fan / ThermalSubsystem | Fan SDR / FRU |
| CPU | Processor | Host / OEM inventory |
| DIMM | Memory | SMBIOS / OEM inventory |

### 27.10.2 不一致時如何判斷

- IPMI 有資料, Redfish 沒有: FRU 可能已解析, 但 inventory object 或 association 未建立.
- Redfish 有資料, IPMI 缺欄位: IPMI FRU ID 或 mapping 可能未設定.
- D-Bus 正確, 兩者都錯: 檢查 bmcweb / ipmid 的 mapping 與版本.
- Redfish 與 IPMI 顯示不同序號: 確認是否一方使用 Board area, 另一方使用 Product area.

## 27.11 Hot-Plug 與動態 Inventory

PSU, fan tray, riser 與部分 drive 可在系統執行期間插拔. 動態 inventory 流程需要處理:

```text
Presence 改變
    ↓
Debounce
    ↓
FRU / device discovery
    ↓
建立或更新 inventory object
    ↓
建立 sensor 與 associations
    ↓
更新 Redfish / IPMI / event
```

拔除時則需要:

- 將 `Present` 更新為 false, 或依資料模型移除動態 object.
- 停止或標示 sensors unavailable.
- 移除不再有效的 associations.
- 清除舊元件的 serial / model cache, 避免下次插入沿用.
- 記錄 insertion / removal event.

Service restart, BMC reboot 與重新插入後, 應能從硬體重新建立正確狀態.

## 27.12 製造資料與持久化

Inventory 資料可分為三類.

### 27.12.1 Factory Data

工廠寫入且通常不可由一般使用者修改:

- Serial number
- Part number
- Manufacture date
- MAC address
- UUID
- Board revision

常見保存位置是 FRU EEPROM, secure storage 或 factory partition. BMC firmware update 與 factory reset 不應破壞這些資料, 除非產品規格明確要求.

### 27.12.2 Field-Writable Data

現場可能修改:

- AssetTag
- Location
- User label

需定義:

- 誰可以寫.
- 寫入哪裡.
- 是否寫回 FRU EEPROM.
- BMC reboot / update / factory reset 後是否保留.
- 是否產生 audit log.

### 27.12.3 Runtime Cache

FRU parse result, host inventory 與 discovery 結果可以快取, 但 cache 必須可重建. 不可把 cache 誤當成唯一權威端, 否則 factory reset 或檔案損壞後可能遺失資產資料.

### 27.12.4 Firmware Update 與 Factory Reset

測試至少包含:

1. 記錄 update 前的 FRU, D-Bus, Redfish, IPMI 資料.
2. 執行 BMC firmware update.
3. 比對 update 後資料.
4. 執行 factory reset.
5. 確認 factory data 與 field-writable data 是否符合保存規格.
6. 確認 runtime cache 可以重新建立.

## 27.13 FRU 寫入與資料保護

FRU EEPROM 寫入需考慮:

- Page write size.
- Write cycle time.
- Write protect.
- Power loss.
- Checksum 更新.
- Area length 與 offset.
- 寫入權限.
- 備份與回復.

不建議直接修改單一文字欄位而忽略 area checksum. 較安全的流程是:

```mermaid
flowchart TB
    A["讀取完整 FRU binary"] --> B["驗證格式與 checksum"]
    B --> C["在記憶體中修改並重建 area"]
    C --> D["重新計算 checksum"]
    D --> E["驗證寫入權限與 WP"]
    E --> F["寫入 EEPROM"]
    F --> G["讀回並逐 byte verify"]
    G --> H["重新啟動 parser 並確認對外資料"]
```

寫入途中失去電源可能留下半更新資料. 重要 FRU 應有備份, 工廠重建方式或可復原流程.

## 27.14 安全與隱私

Inventory 可能包含設備序號, AssetTag, 位置, MAC, UUID 與客戶自訂名稱. 需考慮:

- D-Bus, Redfish 與 IPMI 的存取權限.
- AssetTag 修改權限.
- Provisioning tool 認證.
- FRU write protection.
- 操作 audit log.
- Debug package 對外分享前的資料遮蔽.
- RMA 換板時 system serial 與 board serial 的處理.

System serial, board serial 與 chassis serial 可能屬於不同實體, 不應為了方便而全部同步成同一個值.

## 27.15 Target 端排查流程

### 27.15.1 確認硬體與 FRU

```bash
$ i2cdetect -l
$ ls -l /sys/bus/i2c/devices
$ find /sys/bus/i2c/devices -maxdepth 3 -name eeprom -print
```

確認 adapter, mux, EEPROM client 與 eeprom file. 若要讀取 FRU binary, 先確認資料處理與分享規範.

### 27.15.2 確認 FRU Service

Service 名稱依 branch 與整合方式而異, 可先查詢:

```bash
$ systemctl --type=service | grep -Ei 'fru|entity|inventory'
$ journalctl -b --no-pager | grep -Ei 'fru|eeprom|checksum|entity|inventory'
```

確認:

- EEPROM 是否被找到.
- FRU 是否成功解析.
- 是否有 checksum error.
- FRU properties 是否發布到 D-Bus.

### 27.15.3 確認 Entity Manager

```bash
$ systemctl status xyz.openbmc_project.EntityManager.service --no-pager
$ journalctl -u xyz.openbmc_project.EntityManager.service -b --no-pager
$ busctl tree xyz.openbmc_project.EntityManager
```

確認 Probe 使用的欄位實際值與 JSON 完全一致, 也要留意大小寫, 空白與舊版 FRU 格式.

### 27.15.4 確認 Inventory Objects

```bash
$ busctl tree xyz.openbmc_project.ObjectMapper | grep -i inventory
$ busctl tree xyz.openbmc_project.Inventory.Manager 2>/dev/null
```

對單一 object 使用 `busctl introspect`, 檢查 interfaces, Asset properties, Present, Functional 與 associations.

### 27.15.5 確認 Redfish 與 IPMI

```bash
$ ipmitool fru print
$ ipmitool sdr elist
```

Redfish 可檢查:

```text
/redfish/v1/Systems
/redfish/v1/Chassis
/redfish/v1/Managers
```

接著沿 resource links 檢查 PSU, Fan, Assembly, Processor 與 Memory. 正式腳本不應把預設帳號密碼寫在指令或文件中.

## 27.16 常見問題與判讀

| 現象 | 流程大約停在哪裡 | 優先檢查 |
|---|---|---|
| FRU EEPROM 無法讀取 | I2C / EEPROM | Bus, mux, address, power, WP |
| FRU checksum error | FRU binary | Header, area length, 寫入流程 |
| fru-device 有資料, Entity Manager 沒反應 | Probe | Property 名稱, 值, JSON |
| D-Bus 有 FRU, 沒有 inventory object | Entity / inventory 建立 | Entity Manager journal, interfaces |
| Inventory 有物件, Redfish 沒有 | Association / bmcweb mapping | ObjectMapper, bmcweb journal |
| IPMI 有 FRU, Redfish 沒資料 | 只建立 legacy FRU path | D-Bus inventory 與 association |
| Redfish 與 IPMI 序號不同 | 欄位來源不同 | Board / Product / Chassis mapping |
| PSU 拔除後仍 Present=true | Presence / cache | GPIO, PMBus, hot-plug service |
| 新 PSU 沿用舊序號 | Cache 未清除 | Object lifecycle, FRU rescan |
| Factory reset 後序號消失 | Cache 被當權威端 | FRU EEPROM, reset policy |
| Firmware update 後料號變回預設 | Static config 覆蓋 | Migration, priority, persistent store |
| Object path 每次開機不同 | 使用動態 index 命名 | Path rule, bus number, probe order |

## 27.17 Debug Log 收集

以下腳本收集 inventory 相關狀態, 不修改 FRU 或 Asset data:

```bash
#!/bin/sh

OUT=/tmp/inventory-debug
mkdir -p "$OUT"

cat /etc/os-release > "$OUT/os-release.txt" 2>&1
uname -a > "$OUT/uname.txt"
cat /proc/cmdline > "$OUT/proc-cmdline.txt"

dmesg -T > "$OUT/dmesg.txt"
journalctl -b --no-pager > "$OUT/journal.txt" 2>&1
systemctl --failed > "$OUT/systemctl-failed.txt" 2>&1

systemctl --type=service | grep -Ei 'fru|entity|inventory' \
    > "$OUT/inventory-services.txt" 2>&1

journalctl -u xyz.openbmc_project.EntityManager.service \
    -b --no-pager > "$OUT/entity-manager-journal.txt" 2>&1
journalctl -u xyz.openbmc_project.FruDevice.service \
    -b --no-pager > "$OUT/fru-device-journal.txt" 2>&1

busctl tree xyz.openbmc_project.ObjectMapper \
    > "$OUT/objectmapper.txt" 2>&1
busctl tree xyz.openbmc_project.EntityManager \
    > "$OUT/entity-manager-tree.txt" 2>&1
busctl tree xyz.openbmc_project.FruDevice \
    > "$OUT/fru-device-tree.txt" 2>&1

command -v i2cdetect >/dev/null 2>&1 && \
    i2cdetect -l > "$OUT/i2cdetect-l.txt" 2>&1
ls -l /sys/bus/i2c/devices > "$OUT/i2c-devices.txt" 2>&1
find /sys/bus/i2c/devices -maxdepth 3 -name eeprom -print \
    > "$OUT/eeprom-files.txt" 2>&1

command -v gpioinfo >/dev/null 2>&1 && \
    gpioinfo > "$OUT/gpioinfo.txt" 2>&1

command -v ipmitool >/dev/null 2>&1 && \
    ipmitool fru print > "$OUT/ipmi-fru.txt" 2>&1

# FRU binary 可能包含識別資料，不由通用腳本自動複製。

tar czf "/tmp/inventory-debug-$(date +%Y%m%d-%H%M%S).tar.gz" \
    -C /tmp inventory-debug
```

## 27.18 Bring-up 順序

1. 建立 system, chassis, board, PSU, fan, riser, CPU, DIMM 等實體清單.
2. 為每個元件指定穩定 object path.
3. 定義 Manufacturer, Model, PartNumber, SerialNumber, AssetTag 的權威端.
4. 驗證每個 FRU EEPROM 的 bus, address, 容量, WP 與 checksum.
5. 驗證 fru-device 發布的欄位.
6. 撰寫 Entity Manager Probe 與 Exposes.
7. 確認 inventory objects 與 Asset properties.
8. 建立 containment 與 sensor associations.
9. 驗證 Present, Available, Functional 與 Health.
10. 驗證 Redfish 與 IPMI mapping.
11. 測試 PSU, fan tray 與 riser 的插入, 拔除與重插.
12. 測試 service restart, BMC reboot, AC cycle, firmware update 與 factory reset.
13. 比對工廠資料庫, FRU binary, D-Bus, Redfish 與 IPMI.
14. 保存 log, 版本, 欄位 priority 與已知限制.

## 27.19 平台實測紀錄表

| 元件 | 穩定身分 / Slot | 權威端 | D-Bus Object | Redfish / IPMI | 狀態 |
|---|---|---|---|---|---|
| System | [待填] | [待填] | [待填] | ComputerSystem / Product FRU | [待確認] |
| Chassis | [待填] | [待填] | [待填] | Chassis / Chassis FRU | [待確認] |
| Baseboard | [待填] | Board FRU | [待填] | Assembly / Board FRU | [待確認] |
| BMC | [待填] | [待填] | [待填] | Manager | [待確認] |
| PSU0 | Slot 0 | PSU FRU / PMBus | [待填] | PowerSupply / FRU | [待確認] |
| PSU1 | Slot 1 | PSU FRU / PMBus | [待填] | PowerSupply / FRU | [待確認] |
| Fan tray | [待填] | FRU / presence | [待填] | Fan / FRU | [待確認] |
| Riser | [待填] | FRU / ID | [待填] | Assembly / PCIeSlot | [待確認] |
| CPU | Socket [待填] | SMBIOS / host | [待填] | Processor | [待確認] |
| DIMM | Slot [待填] | SMBIOS / SPD | [待填] | Memory | [待確認] |
| CPLD | [待填] | Version register | [待填] | Assembly / OEM | [待確認] |

FRU EEPROM 紀錄:

| FRU | I2C Path | Address | EEPROM | WP | Areas | Checksum | Owner |
|---|---|---:|---|---|---|---|---|
| Baseboard | [待填] | [待填] | [待填] | [待填] | [待填] | [待填] | Factory / BMC |
| PSU0 | [待填] | [待填] | [待填] | N/A | [待填] | [待填] | PSU vendor |
| PSU1 | [待填] | [待填] | [待填] | N/A | [待填] | [待填] | PSU vendor |
| Fan board | [待填] | [待填] | [待填] | [待填] | [待填] | [待填] | Factory / BMC |
| Riser | [待填] | [待填] | [待填] | [待填] | [待填] | [待填] | Factory / BMC |

## 27.20 驗收 Checklist

### 資料模型

- [ ] 所有實體元件都有穩定的 inventory identity 與 object path.
- [ ] System, Chassis, Board 序號與語意已分開.
- [ ] 每個 Asset 欄位都有權威端, fallback 與 conflict 規則.
- [ ] Object path 不依賴 bus number, hwmon index 或序號.
- [ ] Present, Available, Functional 與 Health 已分開處理.

### FRU 與製造資料

- [ ] EEPROM bus, address, 容量, page size, WP 與 checksum 已驗證.
- [ ] Board, Product, Chassis areas 的欄位映射已確認.
- [ ] FRU checksum error 與 missing EEPROM 流程已測試.
- [ ] Factory data 不會被 firmware update 或一般 factory reset 誤刪.
- [ ] AssetTag 等可寫欄位具有權限, 保存與 audit 規則.

### OpenBMC

- [ ] fru-device 能正確解析並發布 FRU properties.
- [ ] Entity Manager Probe 使用穩定欄位.
- [ ] Inventory objects, interfaces 與 Asset properties 正確.
- [ ] Associations 能將 sensors 連到正確元件與 chassis.
- [ ] Redfish, IPMI 與 D-Bus 顯示一致, 差異有明確定義.

### 動態行為

- [ ] 可插拔元件插入, 拔除與重插後資料正確更新.
- [ ] 拔除後不保留舊 serial, sensor 或 association.
- [ ] Present=false 不會產生誤導的 sensor critical event.
- [ ] Service restart, BMC reboot 與 AC cycle 後可重建 inventory.
- [ ] Debug package 不會未經授權收集或外洩 FRU binary.

## 27.21 本章重點

1. Inventory 是 BMC 對實體元件, 位置, 狀態與關係的共同資料模型.
2. FRU, FRU data 與 FRU EEPROM 是不同概念.
3. Asset 包含 Manufacturer, Model, PartNumber, SerialNumber 與 AssetTag.
4. 同一欄位只能有一個明確權威端, 其他來源是 fallback 或診斷資料.
5. Board, Product 與 Chassis FRU areas 的欄位不能混用.
6. Object path 應代表穩定 slot 或元件角色, 不應依賴動態 index 或序號.
7. Present, Available, Functional 與 Health 描述不同狀態.
8. Association 將 inventory, sensors, chassis 與其他元件連成硬體拓樸.
9. Redfish 與 IPMI 可共用原始資料, 但映射模型並非完全相同.
10. Factory data, field-writable data 與 runtime cache 必須使用不同保存策略.

## 27.22 本章參考資料

- OpenBMC entity-manager: https://github.com/openbmc/entity-manager
- OpenBMC phosphor-dbus-interfaces: https://github.com/openbmc/phosphor-dbus-interfaces
- OpenBMC phosphor-dbus-interfaces Inventory definitions: https://github.com/openbmc/phosphor-dbus-interfaces/tree/master/yaml/xyz/openbmc_project/Inventory
- IPMI Platform Management FRU Information Storage Definition: https://www.intel.com/content/www/us/en/products/docs/servers/ipmi/ipmi-platform-mgt-fru-infostore-def-v1-0-rev-1-3-spec-update.html
- DMTF Redfish Schema Index: https://redfish.dmtf.org/schemas/v1/
