# 23. Network Services

BMC 網路是遠端管理的主要入口. 網路可用性不只取決於是否取得 IP address，還需要完成 MAC 初始化 / PHY / NC-SI link / VLAN / route / DNS / time sync / TLS 與管理服務啟動.

本章從實體網路介面開始，依序說明 DHCP / static IP / IPv6 / VLAN / bonding / NIC failover / hostname / DNS / NTP / PTP / 防火牆 / Redfish 與 IPMI network service，並建立可重複量測的「開機至 API 可用」時間線.

## 23.1 BMC 網路資料路徑

```text
RJ45 / Shared NIC Port
        ↓
PHY 或 NC-SI Channel
        ↓
BMC Ethernet MAC
        ↓
Kernel Network Driver
        ↓
Linux Netdev：eth0 / eth1 / bond0 / vlan
        ↓
IP Address / Route / DNS / Time Sync
        ↓
OpenBMC Network Service
        ↓
HTTPS / Redfish / SSH / IPMI / Event / Telemetry
```

網路排查應先判斷問題位於：

- 實體 link.
- Kernel driver / netdev.
- IP address.
- Route / gateway.
- DNS.
- Time synchronization.
- Service listening.
- TLS / authentication.
- Application response.

`ping` 成功只代表一部分路徑可用，無法證明 Redfish / IPMI / DNS / NTP或事件傳送皆正常.

## 23.2 Dedicated NIC 與 Shared NIC

### 23.2.1 Dedicated Management NIC

Dedicated NIC通常由BMC MAC透過RGMII / RMII連接專用PHY，再連到獨立管理網路埠.

需要確認：

- MAC controller與driver.
- PHY model與MDIO address.
- PHY reset / clock與straps.
- `phy-mode`與RGMII delay.
- MAC address來源.
- Link speed與duplex.
- Cable與switch port.

### 23.2.2 Shared NIC / NC-SI

Shared NIC讓BMC與Host共用實體網路埠. BMC透過 NC-SI 選擇 NIC package 與 channel.

需要確認：

- NIC standby power.
- Package / channel mapping.
- Host on / off behavior.
- AEN與link state更新.
- MAC filter / VLAN與broadcast policy.
- Channel failover.
- NIC reset後的recovery.

### 23.2.3 Interface Identity

`eth0` / `eth1`由runtime enumeration產生，平台文件應另外保存：

- SoC MAC instance.
- Device Tree alias.
- Permanent MAC address.
- Physical connector / shared port.
- Driver path.
- Interface name policy.

## 23.3 Linux Network Stack

Linux netdev提供Layer 2介面，IP address與route建立在netdev上.

基本檢查：

```bash
ip -details link show
ip address show
ip route show table all
ip -6 route show table all
ss -lntup
```

### 23.3.1 Link State

常見狀態：

- Administrative state：介面是否設為UP.
- Carrier：實體link是否存在.
- Operational state：介面整體可用狀態.
- Speed / duplex：PHY協商結果.

```bash
cat /sys/class/net/<iface>/operstate
cat /sys/class/net/<iface>/carrier
ethtool <iface>
ethtool -S <iface>
```

介面為UP但`carrier=0`時，DHCP通常無法完成. 此時優先檢查PHY / NC-SI / cable與switch port.

### 23.3.2 Network Namespace

一般OpenBMC管理服務位於default network namespace. 若產品使用container或額外namespace，需要記錄veth / bridge / route與service所在namespace，避免只在default namespace看到不完整狀態.

## 23.4 OpenBMC Network Management

OpenBMC 常以 network management daemon 提供 D-Bus 介面，再由 systemd-networkd 或其他 backend 套用 Linux 網路設定. `phosphor-networkd`的設計包含 實體與虛擬介面 / IPv4 / IPv6 address / VLAN 與 bond 等 D-Bus 概念.

```text
Redfish / IPMI / WebUI / D-Bus Client
        ↓
OpenBMC Network Manager
        ↓
Persistent Network Configuration
        ↓
systemd-networkd / Netlink
        ↓
Linux Netdev / Address / Route
```

### 23.4.1 D-Bus Objects

常見模型：

```text
/xyz/openbmc_project/network/eth0
/xyz/openbmc_project/network/eth0/ipv4/<id>
/xyz/openbmc_project/network/eth0/ipv6/<id>
/xyz/openbmc_project/network/<vlan>
/xyz/openbmc_project/network/config
```

實際service / object path與interfaces依OpenBMC branch調整.

### 23.4.2 Target 檢查

```bash
systemctl status xyz.openbmc_project.Network.service --no-pager
systemctl status systemd-networkd.service --no-pager
journalctl -u xyz.openbmc_project.Network.service -b --no-pager
journalctl -u systemd-networkd.service -b --no-pager

busctl tree xyz.openbmc_project.Network
networkctl status
```

### 23.4.3 Persistent 與 Runtime State

直接使用`ip address add`建立的設定通常只存在於runtime. 透過D-Bus / Redfish / IPMI或正式network manager修改時，才會依平台設計寫入persistent configuration.

排查設定不一致時需同時比較：

- D-Bus properties.
- Persistent config files.
- `networkctl`.
- `ip address`與`ip route`.
- BMC reboot後狀態.

## 23.5 MAC Address

MAC address是Layer 2 identity. 每個實體管理介面需要穩定且唯一的MAC.

可能來源：

- SoC OTP / eFuse.
- FRU EEPROM.
- Dedicated EEPROM.
- U-Boot environment.
- Device Tree local-mac-address.
- Manufacturing partition.
- Platform service動態設定.

### 23.5.1 優先順序

平台需明確定義權威來源與fallback. 例如：

```text
Factory-programmed EEPROM
        ↓ unavailable
Protected manufacturing storage
        ↓ unavailable
Locally administered fallback MAC
```

Fallback MAC只能用於受控開發或復原流程，並需避免多台設備使用相同值.

### 23.5.2 Permanent 與 Current MAC

```bash
ip link show <iface>
ethtool -P <iface> 2>/dev/null
cat /sys/class/net/<iface>/address
```

Bond / VLAN與NC-SI可能讓current MAC和underlying hardware identity不同. 文件需記錄哪個address對外使用，以及failover後是否改變.

### 23.5.3 Update 與 Factory Reset

MAC屬於factory data，firmware update與一般factory reset通常應保留. RMA換板 / NIC更換或shared NIC policy則需另訂流程.

## 23.6 IPv4：DHCP 與 Static Address

### 23.6.1 DHCP

DHCP流程：

```text
Carrier Up
    ↓
DHCP Discover
    ↓
Offer
    ↓
Request
    ↓
ACK
    ↓
Address / Prefix / Gateway / DNS / Lease Time
```

需確認：

- DHCP client開始時機.
- Link尚未ready時的retry.
- Lease storage與renewal.
- DHCP server提供的gateway與DNS.
- Address conflict處理.
- DHCP timeout後是否啟用fallback address.

### 23.6.2 Static Address

Static設定至少包含：

- IPv4 address.
- Prefix length / netmask.
- Default gateway.
- DNS servers.
- VLAN，若使用.

設定前需確認address不重複且gateway位於可達網段. 錯誤prefix常造成同網段與跨網段行為不一致.

### 23.6.3 DHCP 與 Static 切換

切換時需定義：

- 舊address何時移除.
- 舊route與DNS何時清理.
- 現有HTTPS / SSH session何時中斷.
- 新設定失敗的recovery方式.
- IPMI / Redfish與D-Bus是否顯示相同模式.

遠端修改管理IP具有斷線風險. Client應從response / Task或產品文件取得新位址，而不是假設舊連線會持續.

## 23.7 IPv6 Policy

IPv6可能同時出現：

- Link-local address.
- SLAAC address.
- DHCPv6 address.
- Static IPv6 address.
- Router Advertisement提供的route與DNS.

平台必須明確定義：

- IPv6預設啟用或停用.
- 是否接受RA.
- DHCPv6 stateful / stateless policy.
- Link-local是否保留.
- Temporary / privacy address是否使用.
- Default route來源.
- DNS來源優先順序.
- Redfish / IPMI呈現方式.

檢查：

```bash
ip -6 address show
ip -6 route show table all
networkctl status <iface>
sysctl net.ipv6.conf.all.disable_ipv6
sysctl net.ipv6.conf.<iface>.accept_ra
```

IPv6停用不應只刪除global address；kernel / network manager / service binding與firewall policy都需一致.

## 23.8 Link-Local 與 Fallback Address

IPv4 link-local通常位於`169.254.0.0/16`，IPv6 link-local位於`fe80::/10`. 它們可用於同一Layer 2 segment內的local communication.

需要定義：

- DHCP失敗後是否啟用IPv4 link-local.
- Link-local與static / DHCP是否可並存.
- API是否listen在link-local address.
- IPv6 zone index使用方式，例如`fe80::1%eth0`.
- WebUI與service discovery是否支援.

Fallback address可以提升救援能力，但也可能產生未預期的管理入口，需納入安全與防火牆政策.

## 23.9 Default Gateway 與 Routing

Route決定封包下一站.

```bash
ip route show table all
ip -6 route show table all
ip route get <destination>
```

### 23.9.1 Default Route

多介面平台可能同時取得多條default route. 需定義：

- Route metric.
- Primary interface.
- Failover條件.
- DHCP與static route優先順序.
- IPv4與IPv6各自政策.

### 23.9.2 Source Address Selection

多地址 / 多VLAN / 多NIC時，outgoing packet可能選到非預期source address. 測試event destination / NTP / DNS與firmware download時，需確認實際egress interface與source address.

### 23.9.3 Policy Routing

若產品使用多張管理NIC / VRF或特定服務綁定介面，可能需要multiple route tables與rules. 此時文件應保存：

```bash
ip rule show
ip route show table all
```

## 23.10 VLAN

VLAN在同一實體link上分隔多個Layer 2 networks. Linux通常建立virtual interface：

```text
eth0
└── eth0.100，VLAN ID 100
```

需要記錄：

- Parent interface.
- VLAN ID.
- Tagged / untagged switch port設計.
- VLAN interface MAC.
- IP address與route放在哪一層.
- DHCP是否在VLAN interface執行.
- Redfish / D-Bus resource identity.

OpenBMC network模型可將VLAN視為virtual interface，並在其下建立IPv4 / IPv6 address objects. citeturn44search261

### 23.10.1 Target 檢查

```bash
ip -d link show type vlan
ip address show <vlan-iface>
networkctl status <vlan-iface>
```

### 23.10.2 常見問題

- Switch port未允許該VLAN.
- Native VLAN與tagged policy不同.
- IP設在parent而流量走VLAN.
- VLAN ID錯誤.
- MTU未考量tag overhead或下游限制.
- NC-SI filter未允許VLAN.

## 23.11 Bonding

Bond將多個network interfaces組成一個logical interface. Linux bonding可提供hot standby或load balancing，並可監控link integrity. citeturn44search265

常見用途：

- Dedicated NIC與shared NIC備援.
- 兩張dedicated NIC active-backup.
- 802.3ad / LACP aggregation，若BMC與switch均支援.

### 23.11.1 Active-Backup

一個port active，其他ports standby. 適合管理網路冗餘，switch通常不需LACP，但仍需確認MAC learning與failover behavior.

### 23.11.2 802.3ad

多條link加入LACP group，需要upstream switch使用相同LAG設定. 若switch未正確設定，可能只有部分flow可用或產生loop / packet loss.

### 23.11.3 Bond 必填資料

- Bond mode.
- Members.
- Primary member.
- MII / ARP monitoring.
- Failover delay.
- MAC policy.
- Route / IP所在interface.
- Switch設定.
- NC-SI dependency.

### 23.11.4 Target

```bash
ip -d link show type bond
cat /proc/net/bonding/<bond-iface>
networkctl status <bond-iface>
```

OpenBMC network configuration model包含bond interface概念，但實際支援與properties應依目前branch確認. citeturn44search261

## 23.12 NIC Failover

NIC failover是產品層級的切換策略，可能由bonding / NC-SI channel selection / platform daemon或network manager完成.

需要定義：

```text
Failover Trigger
Link down / PHY error / NC-SI channel loss / DHCP failure / API health failure
        ↓
Decision Owner
Kernel bonding / NC-SI / userspace policy
        ↓
Switch
Interface / channel / route或MAC切換
        ↓
Recovery
Address / ARP / ND / DNS / sessions與events恢復
```

### 23.12.1 Link Failover 與 Service Failover

Carrier down容易偵測；但carrier up不代表gateway / DNS或API路径正常. 若產品需要service-aware failover，必須定義health probe / timeout與避免flapping的hysteresis.

### 23.12.2 Failback

Primary NIC恢復後是否自動切回，需要考量：

- 現有TCP sessions.
- MAC table與gratuitous ARP / unsolicited NA.
- Route metric.
- Event通知.
- Repeated flapping.

## 23.13 Hostname

Hostname用於辨識BMC，可能出現在：

- Shell prompt.
- DHCP option.
- DNS registration.
- TLS certificate SAN，依產品流程.
- Redfish Manager / EthernetInterface properties.
- Event source.

檢查：

```bash
hostnamectl
cat /etc/hostname
busctl tree xyz.openbmc_project.Network
```

需要定義：

- Factory default格式.
- 是否包含serial / asset identity.
- 使用者可修改範圍.
- 字元與長度限制.
- Factory reset behavior.
- Hostname變更是否更新DHCP / DNS / certificate.

Hostname不應直接包含敏感客戶資料，對外分享logs前也要評估遮蔽.

## 23.14 DNS

DNS將hostname解析為IP address. BMC作為client時，firmware download / NTP / LDAP / event destination與remote syslog都可能依賴DNS.

可能來源：

- Static DNS servers.
- DHCPv4.
- DHCPv6.
- Router Advertisement / RDNSS，依stack支援.

需要定義來源優先順序與介面綁定.

檢查：

```bash
resolvectl status 2>/dev/null
cat /etc/resolv.conf
getent hosts <hostname>
```

### 23.14.1 排查層級

1. DNS server address是否存在.
2. Route是否可達DNS server.
3. UDP / TCP 53是否被阻擋.
4. Resolver service是否ready.
5. Search domain是否造成錯誤名稱.
6. IPv4 / IPv6 answer是否符合service可達性.

## 23.15 Time Synchronization

正確時間會影響：

- TLS certificate validation.
- Event timestamp.
- Audit log.
- Firmware update與attestation紀錄.
- Kerberos / LDAP等外部服務，依產品支援.

### 23.15.1 NTP / SNTP

BMC常透過systemd-timesyncd / chrony或平台服務同步NTP server.

```bash
timedatectl status
timedatectl timesync-status 2>/dev/null
systemctl status systemd-timesyncd --no-pager 2>/dev/null
journalctl -u systemd-timesyncd -b --no-pager 2>/dev/null
```

需定義：

- Static與DHCP-provided NTP優先順序.
- 多server選擇.
- Initial clock來源.
- 大幅時間修正採step或slew.
- Offline behavior.
- BMC reboot後首次同步時間.

### 23.15.2 PTP

PTP可提供較高精度的time synchronization，可能使用hardware timestamping與PHC. Linux PTP工具包含`ptp4l` / `phc2sys` / `pmc`與hardware timestamp控制工具. citeturn44search270

檢查：

```bash
ethtool -T <iface>
ls -l /dev/ptp*
systemctl --type=service | grep -Ei 'ptp|phc'
```

需記錄：

- PTP profile.
- Master / slave role.
- Domain number.
- Interface.
- Hardware / software timestamping.
- PHC到system clock同步方式.
- Network switch是否支援transparent / boundary clock.

### 23.15.3 NTP 與 PTP Owner

若兩者同時調整system clock，可能互相競爭. 平台必須指定唯一system-clock owner，其他service作為reference或在特定條件下切換.

## 23.16 Network Service Listening

取得IP後，管理服務還需要完成socket bind與application initialization.

常見服務：

- HTTPS / Redfish / WebUI.
- SSH.
- IPMI over LAN，若啟用.
- SNMP，若產品支援.
- Remote logging.
- Event sender.
- Virtual media / KVM相關service.

檢查：

```bash
ss -lntup
systemctl --failed
systemctl status bmcweb --no-pager
journalctl -u bmcweb -b --no-pager
```

服務可能bind：

- 所有addresses.
- 指定interface.
- IPv4 only.
- IPv6 only.
- Loopback only.

設定IP或VLAN後需確認服務的bind策略是否自動更新.

## 23.17 Redfish Network Mapping

Redfish通常透過Manager下的EthernetInterfaces呈現BMC網路設定：

```text
/redfish/v1/Managers/<id>/EthernetInterfaces
```

常見資料：

- Interface enabled.
- MAC address.
- Link status.
- Hostname.
- IPv4 addresses.
- IPv6 addresses.
- DHCP configuration.
- Name servers.
- VLAN，依resource model與實作.

Redfish PATCH通常經bmcweb轉成D-Bus network設定，再由network manager套用. 修改管理介面自身地址時，原HTTPS connection可能中斷，client需要依產品流程重新連線.

驗證：

- D-Bus與Redfish數值一致.
- PATCH後runtime與persistent config一致.
- BMC reboot後設定保留.
- Invalid address / prefix / gateway與DNS有明確error.
- 權限不足時回傳正確status與Message Registry資訊.

## 23.18 IPMI LAN Configuration

若產品支援IPMI LAN parameters，需要將IPMI設定映射到同一套network authority，避免IPMI / Redfish與D-Bus各寫一份設定.

常見項目：

- IP source.
- IPv4 address.
- Netmask.
- Default gateway.
- MAC address.
- VLAN ID / priority.
- IPv6，依IPMI與平台支援.

```bash
ipmitool lan print <channel>
```

設定流程可能要求set-in-progress / commit semantics. 需測試中斷 / 部分寫入 / 非法組合與Redfish同步.

## 23.19 Firewall 與 Service Exposure

管理介面應只開放產品需要的ports與protocols.

檢查：

```bash
ss -lntup
nft list ruleset 2>/dev/null
iptables-save 2>/dev/null
ip6tables-save 2>/dev/null
```

需要定義：

- HTTPS.
- SSH.
- IPMI over LAN.
- SNMP.
- mDNS / service discovery.
- NTP / PTP client traffic.
- Event / syslog outgoing traffic.
- Factory-only services.

IPv4與IPv6 firewall需同步審查. 只限制IPv4可能讓同一服務仍可透過IPv6存取.

## 23.20 TLS / Certificate 與時間相依性

HTTPS可用性包含：

```text
TCP Connect
    ↓
TLS Handshake
    ↓
Certificate Validation
    ↓
HTTP Authentication
    ↓
Redfish Response
```

若BMC時間錯誤，client certificate或server certificate驗證可能失敗. 若hostname與certificate SAN不一致，使用名稱連線也會失敗.

需驗證：

- Certificate chain.
- Subject Alternative Name.
- Expiry.
- Private key permissions.
- TLS versions / ciphers.
- Certificate replacement.
- NTP尚未同步時的behavior.

## 23.21 開機可連線時間

「網路ready」需要拆成多個時間點：

```text
T0  BMC reset deassert
T1  Kernel driver ready
T2  Netdev created
T3  Carrier up
T4  IP address available
T5  Default route available
T6  DNS ready
T7  Time synchronized
T8  Management service listening
T9  TLS handshake success
T10 Authentication success
T11 First Redfish API success
```

原始章節要求將driver / link / DHCP / service listening與API first success分開量測；新版保留並擴充為完整時間線. citeturn44search1

### 23.21.1 量測指標

| 指標          | 定義                               |
| ------------- | ---------------------------------- |
| Driver Ready  | Netdev建立且driver完成probe        |
| Link Ready    | Carrier變為up                      |
| Address Ready | Static address套用或DHCP lease完成 |
| Route Ready   | 目標路徑有可用route                |
| Service Ready | TCP port開始listen                 |
| TLS Ready     | TLS handshake成功                  |
| API Ready     | 經authentication的Redfish GET成功  |
| Time Ready    | Clock達到產品要求的同步狀態        |

### 23.21.2 量測方法

BMC端保存monotonic timestamp：

```bash
journalctl -b -o short-monotonic \
    -u systemd-networkd \
    -u xyz.openbmc_project.Network.service \
    -u bmcweb
```

外部client可定期測試：

```bash
while true; do
    date -Ins
    curl --connect-timeout 1 --max-time 2 \
         --cacert <ca.pem> \
         -H 'X-Auth-Token: <token>' \
         https://<bmc>/redfish/v1/ >/dev/null && break
    sleep 0.5
done
```

若BMC重啟會使token失效，測試工具需在API可達後建立新session，再記錄first authenticated success.

### 23.21.3 測試條件

- Static IPv4.
- DHCPv4 server正常.
- DHCPv4 server無回應.
- IPv6 SLAAC / DHCPv6.
- Cable在開機前已插入.
- Cable延後插入.
- Dedicated NIC.
- NC-SI / Host off / on.
- VLAN.
- Bond primary / backup.
- DNS / NTP server無回應.

## 23.22 Network Configuration Update

更新網路設定可能切斷目前管理連線. 安全流程：

```text
Validate New Configuration
        ↓
Persist Candidate
        ↓
Apply Runtime Configuration
        ↓
Verify Address / Route / Service
        ↓
Commit
```

若產品支援rollback，可在指定時間內無法驗證管理連線時恢復舊設定.

需要測試：

- 相同address重複設定.
- Invalid prefix.
- Gateway不在on-link subnet.
- Duplicate VLAN ID.
- 刪除最後一個可管理address.
- 切換DHCP / static.
- 修改primary bond member.
- Apply期間BMC reset / AC loss.

## 23.23 Persistent Data / Update 與 Factory Reset

網路設定通常位於persistent storage. 需分類：

| 資料                   | Firmware Update | Factory Reset          |
| ---------------------- | --------------- | ---------------------- |
| Factory MAC            | 保留            | 通常保留               |
| Static IP              | 保留            | 依產品政策清除或回預設 |
| DHCP mode              | 保留            | 依產品預設             |
| VLAN / Bond            | 保留            | 依產品政策             |
| Hostname               | 保留            | 依產品政策             |
| DNS / NTP              | 保留            | 依產品政策             |
| TLS certificate        | 保留            | 依安全政策             |
| DHCP lease cache       | 可重建          | 可清除                 |
| Runtime neighbor cache | 不保留          | 清除                   |

Firmware downgrade時需考量config schema相容性. 新版建立的bond / VLAN或IPv6設定，舊版service可能無法解析.

## 23.24 Security 與管理平面隔離

需要考量：

- Dedicated management network.
- VLAN separation.
- Shared NIC與Host traffic隔離.
- Service listening範圍.
- Firewall.
- TLS / certificate.
- SSH policy.
- IPMI cipher suites，若啟用.
- Rate limiting.
- Login lockout.
- Factory network與field network差異.

網路failover不能繞過原本的安全政策. 切換到backup NIC或link-local時，允許的services與firewall規則仍需一致.

## 23.25 常見問題與判讀

| 現象                            | 優先方向                 | 第一輪檢查                                 |
| ------------------------------- | ------------------------ | ------------------------------------------ |
| 沒有netdev                      | Driver / DTS             | `dmesg` / Device Tree / kernel config    |
| Netdev UP但無carrier            | PHY / NC-SI / cable      | `ethtool` / carrier / switch port        |
| DHCP拿不到lease                 | Link / DHCP path         | networkd journal / packet capture / server |
| Static IP同網段可達，跨網段失敗 | Gateway / prefix / route | `ip route` / `ip route get`            |
| IP可達，hostname不可達          | DNS                      | resolver / DNS route / search domain       |
| HTTPS連不上，ping正常           | Service / firewall / TLS | `ss` / bmcweb / firewall / certificate   |
| TLS在開機初期失敗               | Time / certificate       | `timedatectl` / certificate validity     |
| IPv4正常，IPv6失敗              | RA / route / firewall    | IPv6 address / route / rules               |
| VLAN無法通訊                    | Tag / switch port        | VLAN ID / parent / switch config           |
| Bond無法failover                | Mode / monitoring        | `/proc/net/bonding` / carrier / primary  |
| Failover後舊連線中斷            | TCP / MAC / route切換    | GARP / NA / session / application retry    |
| NC-SI只在Host on可用            | NIC power policy         | Standby rail / package / channel           |
| BMC reboot後設定消失            | Persistence              | D-Bus / config file / rwfs                 |
| Redfish與IPMI網路設定不同       | 多重authority            | Network D-Bus / mapping / commit flow      |
| Event送不到remote server        | Route / DNS / source     | `ip route get` / DNS / firewall          |
| 開機API很慢                     | Driver→service時間線    | Monotonic journal / DHCP / bmcweb          |

## 23.26 Packet Capture 與診斷

若image包含`tcpdump`：

```bash
tcpdump -ni <iface> -s 0 -w /tmp/network.pcap
```

針對DHCP：

```bash
tcpdump -ni <iface> -e -vv 'udp port 67 or udp port 68'
```

針對ARP / IPv6 Neighbor Discovery：

```bash
tcpdump -ni <iface> -e 'arp or icmp6'
```

Packet capture可能包含IP / hostname / credentials metadata與客戶網路資訊；分享前需清查. HTTPS payload受TLS保護，但server names / addresses與timing仍可能具有敏感性.

## 23.27 Debug Log 收集

```bash
#!/bin/sh

OUT=/tmp/network-debug
mkdir -p "$OUT"

cat /etc/os-release > "$OUT/os-release.txt" 2>&1
uname -a > "$OUT/uname.txt"
cat /proc/cmdline > "$OUT/proc-cmdline.txt"

dmesg -T > "$OUT/dmesg.txt"
journalctl -b --no-pager > "$OUT/journal.txt" 2>&1
journalctl -b -o short-monotonic --no-pager \
    > "$OUT/journal-monotonic.txt" 2>&1
systemctl --failed > "$OUT/systemctl-failed.txt" 2>&1

ip -details link show > "$OUT/ip-link.txt" 2>&1
ip address show > "$OUT/ip-address.txt" 2>&1
ip route show table all > "$OUT/ip-route.txt" 2>&1
ip -6 route show table all > "$OUT/ip6-route.txt" 2>&1
ip rule show > "$OUT/ip-rule.txt" 2>&1
ip neigh show > "$OUT/ip-neigh.txt" 2>&1

networkctl status > "$OUT/networkctl.txt" 2>&1
resolvectl status > "$OUT/resolvectl.txt" 2>&1
timedatectl status > "$OUT/timedatectl.txt" 2>&1
ss -lntup > "$OUT/listening-sockets.txt" 2>&1

for p in /sys/class/net/*; do
    iface=$(basename "$p")
    ethtool "$iface" > "$OUT/ethtool-$iface.txt" 2>&1
    ethtool -S "$iface" > "$OUT/ethtool-$iface-stats.txt" 2>&1
    [ -f "/proc/net/bonding/$iface" ] && \
        cat "/proc/net/bonding/$iface" > "$OUT/bond-$iface.txt"
done

busctl tree xyz.openbmc_project.Network \
    > "$OUT/network-dbus.txt" 2>&1
journalctl -u xyz.openbmc_project.Network.service -b --no-pager \
    > "$OUT/network-service-journal.txt" 2>&1
journalctl -u systemd-networkd -b --no-pager \
    > "$OUT/networkd-journal.txt" 2>&1
journalctl -u bmcweb -b --no-pager \
    > "$OUT/bmcweb-journal.txt" 2>&1

# 通用腳本不收集密碼 / session token / private key或封包內容. 

tar czf "/tmp/network-debug-$(date +%Y%m%d-%H%M%S).tar.gz" \
    -C /tmp network-debug
```

## 23.28 Bring-up 順序

1. 確認MAC controller / PHY / NC-SI與connector mapping.
2. 確認MAC address權威來源與唯一性.
3. 驗證driver / netdev / carrier / speed與duplex.
4. 驗證DHCP與static IPv4.
5. 定義並驗證IPv6 / RA / SLAAC與DHCPv6 policy.
6. 驗證default route / metric與source address selection.
7. 驗證VLAN與upstream switch設定.
8. 驗證bonding / NIC failover與failback.
9. 驗證hostname / DNS與NTP / PTP.
10. 確認HTTPS / SSH / IPMI等服務的listen範圍.
11. 比對D-Bus / Redfish / IPMI與Linux runtime.
12. 量測driver ready到first authenticated API success的時間線.
13. 測試cable延後插入 / DHCP失敗 / DNS失敗與NTP失敗.
14. 測試BMC reboot / AC cycle / NIC reset與Host power transition.
15. 驗證firmware update與factory reset的保存政策.
16. 保存logs / PCAP / switch設定 / 版本與實測結果.

## 23.29 平台實測紀錄表

| 項目            | 來源 / 指令                 | 實測值 | 備註                    |
| --------------- | --------------------------- | ------ | ----------------------- |
| Physical NIC    | Schematic                   | [待填] | Dedicated / NC-SI       |
| Linux interface | `ip link`                 | [待填] | Driver / device path    |
| MAC authority   | Factory data                | [待填] | Permanent / current     |
| Link            | `ethtool`                 | [待填] | Speed / duplex          |
| IPv4 mode       | D-Bus / Redfish             | [待填] | DHCP / static           |
| IPv4 address    | `ip address`              | [待填] | Prefix / gateway        |
| IPv6 policy     | D-Bus / sysctl              | [待填] | RA / DHCPv6 / static    |
| VLAN            | `ip -d link`              | [待填] | Parent / ID             |
| Bond            | `/proc/net/bonding`       | [待填] | Mode / primary          |
| Failover        | Test result                 | [待填] | Trigger / recovery time |
| Hostname        | `hostnamectl`             | [待填] | Factory reset policy    |
| DNS             | `resolvectl`              | [待填] | Source / priority       |
| NTP / PTP       | `timedatectl` / PTP tools | [待填] | Sync time               |
| HTTPS           | `ss` / API test           | [待填] | First success           |
| SSH             | `ss` / login test         | [待填] | Policy                  |
| IPMI LAN        | `ipmitool lan print`      | [待填] | If supported            |
| Firewall        | nftables / iptables         | [待填] | IPv4 / IPv6             |
| Redfish Mapping | EthernetInterfaces          | [待填] | D-Bus consistency       |

開機時間線：

| Milestone             | Monotonic Time |  Delta | Evidence            |
| --------------------- | -------------: | -----: | ------------------- |
| Kernel Driver Ready   |         [待填] | [待填] | `dmesg`           |
| Netdev Created        |         [待填] | [待填] | Kernel / udev log   |
| Carrier Up            |         [待填] | [待填] | networkd / PHY log  |
| IP Ready              |         [待填] | [待填] | DHCP / static log   |
| Route Ready           |         [待填] | [待填] | `ip route`        |
| Service Listening     |         [待填] | [待填] | `ss` / bmcweb log |
| TLS Success           |         [待填] | [待填] | External probe      |
| Redfish First Success |         [待填] | [待填] | External probe      |
| Time Synchronized     |         [待填] | [待填] | Time service log    |

## 23.30 驗收 Checklist

介面與位址：

- [ ] Dedicated PHY / NC-SI mapping / driver與power dependency已驗證.
- [ ] 每個介面的MAC來源 / 唯一性與保存策略已確認.
- [ ] Link speed / duplex / carrier與error counters符合設計.
- [ ] DHCP / static IPv4與切換流程已測試.
- [ ] IPv6 link-local / SLAAC / DHCPv6 / static與RA policy已定義.
- [ ] Gateway / route metric與source address selection已驗證.

虛擬介面與備援：

- [ ] VLAN parent / ID / switch port與IP層級一致.
- [ ] Bond mode / members / primary / monitoring與switch設定已記錄.
- [ ] NIC failover / failback在carrier failure / NIC reset與Host transition下正常.
- [ ] Failover後MAC / ARP / ND / route與管理服務可恢復.

Name / Time 與 Services：

- [ ] Hostname / DNS source與search domain符合產品政策.
- [ ] NTP / PTP owner / server / profile與同步狀態已驗證.
- [ ] HTTPS / SSH / IPMI與其他ports只在預期interface / address上listen.
- [ ] IPv4與IPv6 firewall policy一致.
- [ ] TLS certificate / SAN / expiry與time dependency已測試.

OpenBMC 與可靠性：

- [ ] Linux runtime / persistent config / D-Bus / Redfish與IPMI一致.
- [ ] 遠端修改IP / VLAN與bond時具有可接受的斷線 / recovery流程.
- [ ] Firmware update / downgrade與factory reset的網路資料保存符合政策.
- [ ] Driver / carrier / IP / route / service / TLS與API時間點已分開量測.
- [ ] Cable delay / DHCP failure / DNS failure / NTP failure與service restart已測試.
- [ ] BMC reboot / AC cycle / NIC reset與Host power transition已完成regression.
- [ ] Debug package不包含credential / token / private key與未經審查的packet capture.

## 23.31 本章重點

1. BMC網路可用性包含link / address / route / DNS / time / service / TLS與API多個階段.
2. Dedicated NIC與NC-SI shared NIC具有不同power與recovery條件.
3. MAC address需要單一權威來源，並在update與factory reset中受到保護.
4. DHCP / static切換需要同步清理address / route與DNS.
5. IPv6 policy應涵蓋link-local / RA / SLAAC / DHCPv6 / static與firewall.
6. VLAN在parent interface上建立Layer 2分段；IP應放在產品設計指定的logical interface.
7. Bonding提供link aggregation或hot standby；NIC failover還需要完整的trigger與recovery policy.
8. DNS與time synchronization會直接影響update / events / TLS與外部directory services.
9. Redfish / IPMI與D-Bus應共用同一network configuration authority.
10. 開機可連線時間應以first authenticated API success結束，而不只量測carrier或DHCP完成.

## 23.32 本章參考資料

- OpenBMC phosphor-networkd: https://github.com/openbmc/phosphor-networkd
- OpenBMC Network Configuration: https://github.com/openbmc/phosphor-networkd/blob/master/docs/Network-Configuration.md
- systemd-networkd documentation: https://www.freedesktop.org/software/systemd/man/latest/systemd-networkd.service.html
- systemd.network documentation: https://www.freedesktop.org/software/systemd/man/latest/systemd.network.html
- Linux kernel networking documentation: https://docs.kernel.org/networking/
- Linux Ethernet Bonding documentation: https://docs.kernel.org/networking/bonding.html
- Linux PTP documentation: https://linuxptp.nwtime.org/documentation/
- DMTF Redfish EthernetInterface schema: https://redfish.dmtf.org/redfish/schema_index
- DMTF NC-SI specifications: https://www.dmtf.org/standards/pmci
