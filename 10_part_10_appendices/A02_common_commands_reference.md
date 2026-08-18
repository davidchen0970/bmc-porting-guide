# 附錄 A02：常用指令速查

本附錄彙整 target 端、build 端與 OpenBMC service 常用指令。實際可用性會受 image 內容、busybox/full tools、kernel config 與平台政策影響。

## A02.1 基本系統資訊

```bash
cat /etc/os-release
uname -a
cat /proc/cmdline
systemctl --failed
journalctl -b --no-pager | tail -200
dmesg -T | tail -200
```

## A02.2 Flash / Storage

```bash
cat /proc/mtd
mtdinfo -a 2>/dev/null
ubinfo -a 2>/dev/null
cat /proc/partitions
lsblk -f 2>/dev/null
blkid 2>/dev/null
findmnt -R /
mount
cat /proc/mounts
df -h
df -i
fw_printenv 2>/tmp/fw_printenv.err | sort
cat /tmp/fw_printenv.err
```

## A02.3 GPIO / Pinmux

```bash
gpiodetect
gpioinfo
gpiofind psu0-present-n
gpioget gpiochip0 0
gpiomon --num-events=5 gpiochip0 0
mount | grep debugfs || mount -t debugfs debugfs /sys/kernel/debug
cat /sys/kernel/debug/gpio 2>/dev/null
find /sys/kernel/debug/pinctrl -maxdepth 2 -type f -print 2>/dev/null
cat /sys/kernel/debug/pinctrl/*/pinmux-pins 2>/dev/null
cat /sys/kernel/debug/pinctrl/*/pinconf-pins 2>/dev/null
```

## A02.4 Reset / Clock / Regulator

```bash
mount | grep debugfs || mount -t debugfs debugfs /sys/kernel/debug
cat /sys/kernel/debug/clk/clk_summary 2>/dev/null
cat /sys/kernel/debug/devices_deferred 2>/dev/null
find /sys/class/regulator -maxdepth 3 -type f -print 2>/dev/null
journalctl -b --no-pager | grep -Ei 'reset|watchdog|clk|clock|regulator|supply|defer|probe' | tail -300
```

## A02.5 I2C / PMBus / Sensor

```bash
i2cdetect -l
i2cdetect -y 0 2>/dev/null
i2cdump -y 0 0x50 2>/dev/null | head
dmesg -T | grep -Ei 'i2c|smbus|pmbus|hwmon|sensor' | tail -300
find /sys/class/hwmon -maxdepth 3 -type f -name name -print -exec cat {} \;
busctl tree xyz.openbmc_project.Hwmon 2>/dev/null
busctl tree xyz.openbmc_project.Sensor 2>/dev/null
```

## A02.6 Fan / Thermal

```bash
find /sys/class/hwmon -maxdepth 3 -type f | sort
busctl tree xyz.openbmc_project.Sensor | grep -Ei 'fan|temperature|thermal' 2>/dev/null
systemctl status phosphor-pid-control.service --no-pager 2>/dev/null
journalctl -u phosphor-pid-control.service -b --no-pager | tail -200
```

## A02.7 Power / Host State

```bash
busctl tree xyz.openbmc_project.State.Host 2>/dev/null
busctl tree xyz.openbmc_project.State.Chassis 2>/dev/null
busctl tree xyz.openbmc_project.Control.Host 2>/dev/null
systemctl status x86-power-control.service --no-pager 2>/dev/null
journalctl -u x86-power-control.service -b --no-pager | tail -300
```

## A02.8 Inventory / FRU / Entity Manager

```bash
busctl tree xyz.openbmc_project.Inventory.Manager 2>/dev/null
busctl tree xyz.openbmc_project.EntityManager 2>/dev/null
systemctl status xyz.openbmc_project.EntityManager.service --no-pager 2>/dev/null
journalctl -u xyz.openbmc_project.EntityManager.service -b --no-pager | tail -300
```

## A02.9 Software Update

```bash
busctl tree xyz.openbmc_project.Software.BMC.Updater 2>/dev/null
busctl tree xyz.openbmc_project.Software.Version 2>/dev/null
busctl tree xyz.openbmc_project.Software.Activation 2>/dev/null
systemctl status phosphor-bmc-code-mgmt.service --no-pager 2>/dev/null
journalctl -u phosphor-bmc-code-mgmt.service -b --no-pager | tail -300
```

## A02.10 Yocto / Build 端

```bash
bitbake-layers show-layers
bitbake-layers show-recipes | head
bitbake -e obmc-phosphor-image | grep '^IMAGE_FSTYPES='
bitbake -e obmc-phosphor-image | grep -E '^(MACHINE|DISTRO|FLASH_SIZE|IMAGE_ROOTFS_SIZE)='
bitbake -c cleanall <recipe>
bitbake <recipe> -c compile -f
bitbake <recipe> -c devshell
```
