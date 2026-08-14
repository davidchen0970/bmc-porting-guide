# 47. SoC Notes Template

## 47.1 用途

這份模板用來記錄每個 SoC 或 board family 的差異。不要把平台特例散落在各章節，否則後面換板、換 silicon revision 或支援第二個客戶時會很難維護。

## 47.2 基本資料

- SoC / Board：
- Silicon revision：
- BMC RAM / flash：
- Boot source：
- UART console：
- Kernel branch：
- U-Boot branch：
- OpenBMC machine：

## 47.3 Boot flow 差異

- Boot ROM 限制：
- SPL / TPL：
- U-Boot 環境變數：
- FIT / secure boot：
- Recovery 方法：

## 47.4 Device Tree 注意事項

- Clock / reset：
- Pinmux：
- I2C bus numbering：
- GPIO polarity：
- Interrupt controller：
- Reserved memory：

## 47.5 常見坑

| 現象 | 可能原因 | 檢查方式 | 修正方向 |
|---|---|---|---|
| 開機卡在 early console | UART clock 或 pinmux 錯 | serial log | 修正 pinctrl/clock |
| I2C 裝置消失 | mux channel 或 address 錯 | i2cdetect, dmesg | 修正 DT 或 board data |
| Sensor 值不合理 | scaling/offset 錯 | hwmon, D-Bus | 修正配置與單位 |
