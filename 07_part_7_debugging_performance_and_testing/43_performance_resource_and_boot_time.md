# 43. Performance, Resource, and Boot Time

## 43.1 觀察面向

BMC 效能問題通常分成三類：開機太慢、常駐資源過高、尖峰工作造成服務逾時。分析時先分清楚是 CPU、memory、I/O、D-Bus storm、network 還是某個 service retry loop。

## 43.2 開機時間

```bash
systemd-analyze
systemd-analyze blame
systemd-analyze critical-chain
journalctl -b -o short-monotonic --no-pager
```

判讀重點：

- 哪個 service 位於 critical-chain。
- 是否有 device timeout 或 dependency 等待。
- 是否有反覆 restart 或等待 network-online。

## 43.3 常駐資源

```bash
top
free -m
cat /proc/meminfo
systemctl status <service>
journalctl -u <service> -b --no-pager
```

## 43.4 優化原則

- 不要先微調，先找到最大瓶頸。
- 開機路徑中的 service 避免做長時間同步 I/O。
- 能 lazy load 的硬體掃描不要阻塞基本管理功能。
- 對於輪詢型 sensor，設定合理週期並避免所有 sensor 同時讀取。

## 43.5 NVMe 非同步 Probe 與裝置編號


Linux kernel 在 PCI bus 與 NVMe driver 配對後，會呼叫 driver 的 `probe()` 進行裝置初始化。NVMe PCI driver 將 `probe_type` 設為 `PROBE_PREFER_ASYNCHRONOUS`，因此 kernel 可將 NVMe 的 probe 工作排入非同步工作佇列，讓多顆儲存裝置平行初始化，避免開機流程逐顆等待而增加啟動時間。

```c
static struct pci_driver nvme_driver = {
    .name       = "nvme",
    .id_table   = nvme_id_table,
    .probe      = nvme_probe,
    .remove     = nvme_remove,
    .shutdown   = nvme_shutdown,
    .driver     = {
        .probe_type = PROBE_PREFER_ASYNCHRONOUS,
#ifdef CONFIG_PM_SLEEP
        .pm = &nvme_dev_pm_ops,
#endif
    },
    .sriov_configure = pci_sriov_configure_simple,
    .err_handler = &nvme_err_handler,
};
```

由於多顆 NVMe SSD 可能在不同執行緒中平行初始化，完成順序不保證固定。Linux 配置的裝置編號可能因此隨開機狀況改變，例如同一顆實體 SSD 在不同次開機中可能分別顯示為 `/dev/nvme0n1` 或 `/dev/nvme1n1`。此現象通常屬於非同步 probe 的正常行為，不代表 SSD 資料、分割區內容或 PCIe 連線發生異常。

系統設定不應依賴 `/dev/nvmeXnY`、`/dev/sdX` 等動態名稱來識別固定磁碟。建議依用途選用下列方式：

- 檔案系統掛載：使用 filesystem UUID。
- Root filesystem：使用 UUID 或 PARTUUID。
- 固定對應實體 PCIe 插槽：使用 udev rule，依 PCIe BDF（Bus:Device.Function）建立固定 symbolic link。
- 故障排查：搭配 `lspci -nnk`、`udevadm info`、`lsblk -o NAME,PATH,SERIAL,WWN,UUID,PARTUUID` 與 `dmesg` 核對實體裝置及其動態名稱。

檢查指令：

```bash
lspci -nnk
lsblk -o NAME,PATH,SERIAL,WWN,UUID,PARTUUID
blkid
udevadm info --query=property --name=/dev/nvme0n1
```

判斷時應區分「裝置編號改變」與「裝置未被偵測」。若僅有 `nvme0`、`nvme1` 順序交換，但所有預期 SSD 均可辨識，且序號、WWN、UUID 與容量正確，通常可視為正常行為；若 SSD 缺失、probe 失敗、PCIe link 異常或出現 I/O error，則仍需進一步排查。
