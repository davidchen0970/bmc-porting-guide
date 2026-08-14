# 14. NC-SI

## 14.1 NC-SI 是什麼

NC-SI是DMTF定義的Network Controller Sideband Interface，讓BMC透過shared NIC使用外部網路，而不必佔用獨立PHY。它定義BMC與NIC之間的control protocol、package / channel model、filter設定、link status與asynchronous event notification。NC-SI管理的是sideband network path，不等同於一般Host Ethernet driver。

### 定義與目的

NC-SI是DMTF定義的Network Controller Sideband Interface，讓BMC透過shared NIC使用外部網路，而不必佔用獨立PHY。它定義BMC與NIC之間的control protocol、package / channel model、filter設定、link status與asynchronous event notification。NC-SI管理的是sideband network path，不等同於一般Host Ethernet driver。

### 參與者與角色

- Management Controller：送出NC-SI commands並選擇package / channel。
- NC-SI Package：一個實體NIC或controller package。
- Channel：package內可供管理流量使用的network port/function。
- NIC firmware：執行commands、filters、AEN與failover。
- Linux NC-SI core / MAC driver：探索channel並將active path連到BMC netdev。

## 14.2 怎麼運作

### 資料格式與規則

NC-SI control packet包含MC ID、header revision、instance ID、command、channel ID、payload length、payload與checksum。Response包含response code與reason code。典型commands涵蓋Clear Initial State、Select Package、Enable Channel、Set MAC Address、Enable AEN與Get Link Status。一般network frames則經sideband data path傳送。

### 工作流程

1. BMC在NIC與sideband link ready後送Clear Initial State。
2. Discover / select package並逐一query channels。
3. 設定MAC、VLAN、broadcast / multicast filters與AEN。
4. Enable channel、network TX並讀link status。
5. 選定active channel，BMC netdev開始傳送一般Ethernet frames。
6. Link AEN或timeout觸發重新查詢、failover或reinitialization。

## 14.3 實務限制與潛在風險

### 相容性與版控

確認NC-SI revision、optional commands、multi-package、VLAN、AEN與NIC firmware版本。

### 安全性與防禦

shared NIC跨越Host / BMC邊界；驗證filter isolation、VLAN、firmware trust與未授權channel切換。

### 錯誤處理

使用response / reason code、timeout與retries；channel reset時避免network flap與reinit storm。

### 效能與資源

management traffic與Host共享NIC / port資源；filter、failover與NIC power state會影響latency與availability。

### 標準化與文件

NC-SI由DMTF標準化，但RBT / RMII wiring、OEM commands與NIC firmware行為仍需vendor文件。


NC-SI 讓 BMC 透過 Host NIC 的 network port 傳送管理流量.

需要記錄:

- BMC netdev.
- NC-SI package / channel.
- NIC standby power.
- Host-on / Host-off behavior.
- MAC address policy.
- VLAN / filters.
- AEN.
- Channel selection / failover.
- NIC reset recovery.

```bash
$ dmesg | grep -Ei 'ncsi|package|channel|AEN'
$ ip link
$ networkctl status 2>/dev/null
```

驗證情境:

- BMC boot / Host off.
- Host power on / off.
- NIC reset.
- Cable insert / remove.
- Channel failover.
- BMC reboot.

管理網路是否在 Host off 時可用, 取決於 NIC 的 power 與產品設計, 應在規格與測試紀錄中明確標示.

