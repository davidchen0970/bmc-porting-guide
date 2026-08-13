# 31. Performance, Resource, and Boot Time

## 31.1 觀察面向

BMC 效能問題通常分成三類：開機太慢、常駐資源過高、尖峰工作造成服務逾時。分析時先分清楚是 CPU、memory、I/O、D-Bus storm、network 還是某個 service retry loop。

## 31.2 開機時間

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

## 31.3 常駐資源

```bash
top
free -m
cat /proc/meminfo
systemctl status <service>
journalctl -u <service> -b --no-pager
```

## 31.4 優化原則

- 不要先微調，先找到最大瓶頸。
- 開機路徑中的 service 避免做長時間同步 I/O。
- 能 lazy load 的硬體掃描不要阻塞基本管理功能。
- 對於輪詢型 sensor，設定合理週期並避免所有 sensor 同時讀取。
