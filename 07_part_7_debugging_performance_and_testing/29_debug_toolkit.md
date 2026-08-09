# 28. Debug Toolkit

常用指令：

```bash
dmesg -T
journalctl -b
journalctl -u <service>
busctl tree xyz.openbmc_project.ObjectMapper
gpiodetect && gpioinfo
i2cdetect -y <bus>
i2cget -y <bus> <addr> <reg>
tcpdump -i <iface> -w /tmp/cap.pcap
ethtool <iface>
ipmitool sensor list
curl -k https://<bmc>/redfish/v1/
```
