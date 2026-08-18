# 附錄 A03：Log 收集套件範本

本附錄提供可直接複製到 target 端的 log 收集範本。建議依平台支援的工具裁切；若某指令不存在，保留錯誤輸出也有助於判讀 image 內容與工具差異。

## A03.1 通用收集套件

```bash
#!/bin/sh
set -eu
TS=$(date +%Y%m%d-%H%M%S)
OUT=/tmp/bmc-debug-${TS}
mkdir -p ${OUT}

cat /etc/os-release > ${OUT}/os-release.txt 2>&1 || true
uname -a > ${OUT}/uname.txt 2>&1 || true
cat /proc/cmdline > ${OUT}/proc-cmdline.txt 2>&1 || true
systemctl --failed > ${OUT}/systemctl-failed.txt 2>&1 || true
dmesg -T > ${OUT}/dmesg.txt 2>&1 || dmesg > ${OUT}/dmesg.txt 2>&1 || true
journalctl -b --no-pager > ${OUT}/journal-current.txt 2>&1 || true
journalctl -b -1 --no-pager > ${OUT}/journal-previous.txt 2>&1 || true

cat /proc/mtd > ${OUT}/proc-mtd.txt 2>&1 || true
cat /proc/partitions > ${OUT}/proc-partitions.txt 2>&1 || true
findmnt -R / > ${OUT}/findmnt.txt 2>&1 || true
mount > ${OUT}/mount.txt 2>&1 || true
df -h > ${OUT}/df-h.txt 2>&1 || true
df -i > ${OUT}/df-i.txt 2>&1 || true
fw_printenv > ${OUT}/fw-printenv.txt 2>&1 || true
mtdinfo -a > ${OUT}/mtdinfo.txt 2>&1 || true
ubinfo -a > ${OUT}/ubinfo.txt 2>&1 || true
lsblk -f > ${OUT}/lsblk-f.txt 2>&1 || true
blkid > ${OUT}/blkid.txt 2>&1 || true

busctl list > ${OUT}/busctl-list.txt 2>&1 || true
busctl tree xyz.openbmc_project.ObjectMapper > ${OUT}/dbus-objectmapper.txt 2>&1 || true
busctl tree xyz.openbmc_project.Inventory.Manager > ${OUT}/dbus-inventory.txt 2>&1 || true
busctl tree xyz.openbmc_project.Sensor > ${OUT}/dbus-sensor.txt 2>&1 || true
busctl tree xyz.openbmc_project.State.Host > ${OUT}/dbus-host-state.txt 2>&1 || true
busctl tree xyz.openbmc_project.State.Chassis > ${OUT}/dbus-chassis-state.txt 2>&1 || true

gpiodetect > ${OUT}/gpiodetect.txt 2>&1 || true
gpioinfo > ${OUT}/gpioinfo.txt 2>&1 || true
cat /sys/kernel/debug/gpio > ${OUT}/debug-gpio.txt 2>&1 || true
cat /sys/kernel/debug/clk/clk_summary > ${OUT}/clk-summary.txt 2>&1 || true
cat /sys/kernel/debug/devices_deferred > ${OUT}/devices-deferred.txt 2>&1 || true
find /sys/kernel/debug/pinctrl -maxdepth 3 -type f -print > ${OUT}/pinctrl-files.txt 2>&1 || true

tar czf /tmp/bmc-debug-${TS}.tar.gz -C /tmp bmc-debug-${TS}
echo /tmp/bmc-debug-${TS}.tar.gz
```

## A03.2 Storage 專用收集套件

```bash
#!/bin/sh
set -eu
TS=$(date +%Y%m%d-%H%M%S)
OUT=/tmp/storage-debug-${TS}
mkdir -p ${OUT}

cat /etc/os-release > ${OUT}/os-release.txt 2>&1 || true
uname -a > ${OUT}/uname.txt 2>&1 || true
cat /proc/cmdline > ${OUT}/proc-cmdline.txt 2>&1 || true
cat /proc/mtd > ${OUT}/proc-mtd.txt 2>&1 || true
cat /proc/partitions > ${OUT}/proc-partitions.txt 2>&1 || true
findmnt -R / > ${OUT}/findmnt.txt 2>&1 || true
mount > ${OUT}/mount.txt 2>&1 || true
df -h > ${OUT}/df-h.txt 2>&1 || true
df -i > ${OUT}/df-i.txt 2>&1 || true
fw_printenv > ${OUT}/fw-printenv.txt 2>&1 || true
mtdinfo -a > ${OUT}/mtdinfo.txt 2>&1 || true
ubinfo -a > ${OUT}/ubinfo.txt 2>&1 || true
blkid > ${OUT}/blkid.txt 2>&1 || true
lsblk -f > ${OUT}/lsblk-f.txt 2>&1 || true
sfdisk -l > ${OUT}/sfdisk-l.txt 2>&1 || true
sgdisk -p /dev/mmcblk0 > ${OUT}/sgdisk-p-mmcblk0.txt 2>&1 || true
sgdisk -v /dev/mmcblk0 > ${OUT}/sgdisk-v-mmcblk0.txt 2>&1 || true
dmesg -T > ${OUT}/dmesg.txt 2>&1 || true
journalctl -b --no-pager > ${OUT}/journal.txt 2>&1 || true

tar czf /tmp/storage-debug-${TS}.tar.gz -C /tmp storage-debug-${TS}
echo /tmp/storage-debug-${TS}.tar.gz
```

## A03.3 GPIO / Pinmux 專用收集套件

```bash
#!/bin/sh
set -eu
TS=$(date +%Y%m%d-%H%M%S)
OUT=/tmp/gpio-debug-${TS}
mkdir -p ${OUT}

gpiodetect > ${OUT}/gpiodetect.txt 2>&1 || true
gpioinfo > ${OUT}/gpioinfo.txt 2>&1 || true
cat /sys/kernel/debug/gpio > ${OUT}/debug-gpio.txt 2>&1 || true
find /sys/kernel/debug/pinctrl -maxdepth 3 -type f -print > ${OUT}/pinctrl-files.txt 2>&1 || true
cat /proc/device-tree/model > ${OUT}/model.txt 2>&1 || true
cat /proc/cmdline > ${OUT}/cmdline.txt 2>&1 || true
dmesg -T > ${OUT}/dmesg.txt 2>&1 || true
journalctl -b --no-pager > ${OUT}/journal.txt 2>&1 || true

tar czf /tmp/gpio-debug-${TS}.tar.gz -C /tmp gpio-debug-${TS}
echo /tmp/gpio-debug-${TS}.tar.gz
```
