# 31. 通用測試矩陣

| 測項                     | 目的             | Pass criteria                   | Log           |
| ------------------------ | ---------------- | ------------------------------- | ------------- |
| Boot Test                | BMC 可正常開機   | Redfish/IPMI/SSH ready          | dmesg/journal |
| AC Cycle                 | 外部斷電恢復     | 狀態符合 AC policy              | power log     |
| DC Cycle                 | host power cycle | host 狀態正確                   | SEL/journal   |
| BMC Reset                | BMC reset        | host 影響符合設計               | reset reason  |
| Update                   | 韌體更新         | version changed, service normal | update log    |
| Power Loss During Update | 復原能力         | 可回復或 rollback               | serial log    |
| Sensor Threshold         | event 正確       | assert/deassert 正確            | SEL/EventLog  |
| Fan Fail                 | failsafe         | PWM 拉高 / event                | tach log      |
| Network VLAN             | 網路設定         | ping/API success                | tcpdump       |
| Secure Boot              | 開機保護         | 非授權 image 被拒絕             | boot log      |
| Factory Reset            | 回復預設         | 指定資料清除                    | audit log     |
