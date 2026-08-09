# 30. Performance / Resource / Boot Time
Boot time 拆解：BootROM、U-Boot、kernel、userspace、network ready、API ready。systemd 可用 `systemd-analyze`、`blame`、`critical-chain` 與 `plot` 檢查。

資源監控：CPU、memory、D-Bus call rate、sensor polling interval、journal size、flash write rate、network connection count。
