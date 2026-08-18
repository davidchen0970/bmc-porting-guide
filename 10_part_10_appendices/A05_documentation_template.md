# 附錄 A05：文件填寫範本

本附錄提供專案落地時可複製使用的表格範本。建議每次硬體 rework、CPLD 更新、BIOS 更新、BMC image 更新或量產流程變更後同步更新。

## A05.1 平台基本資料

| 欄位 | 內容 |
| --- | --- |
| Project / Platform | 待填 |
| Board revision | 待填 |
| BMC SoC / revision | 待填 |
| Host CPU / PCH | 待填 |
| CPLD / FPGA version | 待填 |
| BIOS version | 待填 |
| BMC image version | 待填 |
| Kernel commit / tag | 待填 |
| DTS commit / tag | 待填 |
| Yocto branch / manifest | 待填 |
| Owner | 待填 |
| 更新日期 | 待填 |

## A05.2 關鍵訊號表

| Signal | SoC pin / CPLD bit | Linux line / D-Bus path | Active level | Reset default | Owner | 測試方式 | 狀態 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 待填 | 待填 | 待填 | 待填 | 待填 | 待填 | 待填 | 待填 |

## A05.3 分區與映像表

| 名稱 | Device / Volume | Offset / Start | Size | Layer | FS | Mount point | 更新是否覆寫 | 保存策略 | 備註 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| u-boot | 待填 | 待填 | 待填 | MTD / GPT | none | N/A | 否 | factory | 待填 |
| rofs | 待填 | 待填 | 待填 | MTD / UBI / GPT | SquashFS / ext4 | / | 是 | image | 待填 |
| rwfs | 待填 | 待填 | 待填 | MTD / UBI / GPT | JFFS2 / UBIFS / ext4 | /var / overlay | 否 | persistent | 待填 |

## A05.4 Sensor 對照表

| Sensor | Bus / Address | Driver | D-Bus path | Redfish path | Threshold | Presence dependency | Owner | 狀態 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 待填 | 待填 | 待填 | 待填 | 待填 | 待填 | 待填 | 待填 | 待填 |

## A05.5 Issue 紀錄範本

| 欄位 | 內容 |
| --- | --- |
| Issue ID | 待填 |
| 發現日期 | 待填 |
| 發現版本 | 待填 |
| 現象 | 待填 |
| 影響範圍 | 待填 |
| 重現步驟 | 待填 |
| 已收集 log | 待填 |
| 目前判讀 | 待填 |
| 下一步 | 待填 |
| Owner | 待填 |
| 狀態 | 待填 |
| 關閉條件 | 待填 |

## A05.6 版本與交付物紀錄

| 日期 | BMC image | BIOS | CPLD / FPGA | Board rev | 變更摘要 | 測試結果 | 備註 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 待填 | 待填 | 待填 | 待填 | 待填 | 待填 | 待填 | 待填 |
