# 11. PECI 與 APML

## 11.1 PECI / APML 是什麼

PECI 是 Intel 定義的 CPU 與外部管理 controller 之間的單線管理介面，用於溫度、功率、package configuration 與特定除錯資料。APML 是 AMD 平台管理介面集合，常透過 SMBus承載 SB-TSI、mailbox / RMI等功能。兩者解決相似的 CPU-side manageability問題，但不是相容協定，address、command、資料格式與安全模型均不同。

### 定義與目的

PECI 是 Intel 定義的 CPU 與外部管理 controller 之間的單線管理介面，用於溫度、功率、package configuration 與特定除錯資料。APML 是 AMD 平台管理介面集合，常透過 SMBus承載 SB-TSI、mailbox / RMI等功能。兩者解決相似的 CPU-side manageability問題，但不是相容協定，address、command、資料格式與安全模型均不同。

### 參與者與角色

- BMC / host controller：發出管理 request並處理 timeout / completion。
- CPU socket / package：提供溫度、power、identity與 mailbox data。
- PECI或 SMBus controller driver：執行實體 transaction。
- Protocol / hwmon driver：理解 command與轉換格式。
- OpenBMC service：建立 per-socket sensor、inventory、health與 event。

## 11.2 怎麼運作

### 資料格式與規則

PECI transaction包含 target address、write length、read length、command、payload與完整性檢查；實際 command set依 CPU generation。APML的 SB-TSI通常以 SMBus registers表示溫度，mailbox / RMI則以 command、parameter、status與data registers交換資訊。溫度常是 margin、fixed-point或 family-specific encoding，不可直接把 raw byte當攝氏值。

### 工作流程

1. BMC確認 CPU auxiliary power、reset與介面可用。
2. 依 socket address發出基本識別或 ping。
3. Protocol driver組成 command並由 controller送出。
4. CPU回覆 completion status與payload。
5. Driver驗證完整性、解析 generation-specific format並換算工程值。
6. OpenBMC依 socket identity更新 sensor、stale state與event。

## 11.3 實務限制與潛在風險

### 相容性與版控

CPU generation、PECI command set、APML family、address與 driver support必須建立 compatibility matrix。

### 安全性與防禦

部分 mailbox或 debug commands可能暴露敏感狀態或改變設定；限制 raw command與production debug capability。

### 錯誤處理

CPU off、reset、firmware transition可能正常 timeout；需區分 unavailable、unsupported與硬體 fault。

### 效能與資源

頻繁 telemetry會增加 interface occupancy與CPU package負擔；依 sensor需求設定polling。

### 標準化與文件

PECI與APML多依 silicon vendor文件；公開程度、command access與工具支援可能受平台限制。


PECI 常用於 Intel CPU / DIMM telemetry; APML 相關介面常用於 AMD 平台, 例如 SB-TSI 與 SB-RMI.

這些介面通常依賴 Host power / CPU reset 與 socket presence.

| 介面 | 常見用途 | 主要相依條件 |
|---|---|---|
| PECI | CPU / DIMM temperature / power / debug | CPU package power / address / driver |
| SB-TSI | AMD CPU temperature | I2C path / address / Host state |
| SB-RMI | AMD management mailbox | I2C path / mailbox / firmware support |

Target:

```bash
$ dmesg | grep -Ei 'peci|sbtsi|sbrmi|apml'
$ find /sys/bus/peci -maxdepth 4 -type f 2>/dev/null
$ find /sys/class/hwmon -maxdepth 4 -type f | sort
```

Host off 或 CPU 尚未 ready 時, sensor 應標示 unavailable, 避免將正常的 power-state dependency 誤報為 hardware fault.

