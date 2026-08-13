# 29. Debug Toolkit

## 29.1 常用工具分類

### 系統與服務

```bash
systemctl status <service>
journalctl -u <service> -b --no-pager
systemd-analyze blame
systemd-analyze critical-chain
```

### Kernel 與硬體

```bash
dmesg -T
journalctl -k -b --no-pager
ls /sys/bus/i2c/devices
ls /sys/class/hwmon
```

### D-Bus

```bash
busctl tree
busctl tree xyz.openbmc_project.Sensor.Value
busctl introspect <service> <path>
busctl get-property <service> <path> <interface> <property>
```

### Redfish 與網路

```bash
curl -k https://$BMC/redfish/v1/
ss -lntup
ip addr
ip route
```

## 29.2 建議收集包

- `journalctl -b --no-pager`
- `journalctl -k -b --no-pager`
- `systemctl --failed`
- `busctl tree`
- `/etc/os-release`
- BMC version、machine name、git commit、測試步驟
