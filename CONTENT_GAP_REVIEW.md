# BMC Porting Guide內容差異與補強檢討

## 本次整合

- Ch17加入Linux Kernel MCTP架構、kernel/userspace分工、mctp-i2c初始化、I2C mux netdev、bus locking、EID / route / neighbor設定與loopback test。
- Ch32加入AST2700雙LTPI instances、LL/NL GPIO capacity、TDM資料路徑、Board ID register計算、libgpiod驗證與SKU mapping案例。

## 比較方法

以兩份簡報補強後的Ch17與Ch32作為「協定原理 + silicon實作 + kernel路徑 +實機案例」參考，檢查每章是否包含14類內容。自動檢查只代表關鍵主題是否出現，仍需由domain owner確認技術深度與正確性。

## 全章矩陣

| Ch | 檔案 | 行數 | 覆蓋 | 主要缺口 |
|---:|---|---:|---:|---|
| 1 | `01_boot_flow_and_soc_initialization.md` | 417 | 12/14 | 定義/目的, 效能/資源 |
| 2 | `02_flash_partition_and_storage_architecture.md` | 1741 | 14/14 | 無明顯結構缺口 |
| 3 | `03_pinmux_gpio_common_design_patterns.md` | 2207 | 12/14 | 角色/架構, 效能/資源 |
| 4 | `04_reset_clock_and_power_domain.md` | 897 | 13/14 | 定義/目的 |
| 5 | `05_peripheral_bus_fundamentals.md` | 473 | 12/14 | 定義/目的, 效能/資源 |
| 6 | `06_i2c_smbus_and_pmbus.md` | 130 | 11/14 | DTS/Binding, Bring-up/驗收, 參考資料 |
| 7 | `07_spi.md` | 118 | 11/14 | OpenBMC整合, Bring-up/驗收, 參考資料 |
| 8 | `08_uart_and_serial_console.md` | 97 | 10/14 | DTS/Binding, OpenBMC整合, Bring-up/驗收, 參考資料 |
| 9 | `09_adc_and_iio.md` | 90 | 12/14 | DTS/Binding, Bring-up/驗收 |
| 10 | `10_pwm_and_tach.md` | 89 | 11/14 | DTS/Binding, Bring-up/驗收, 參考資料 |
| 11 | `11_peci_and_apml.md` | 76 | 11/14 | DTS/Binding, Bring-up/驗收, 參考資料 |
| 12 | `12_espi_lpc_and_host_interface.md` | 85 | 11/14 | DTS/Binding, Bring-up/驗收, 參考資料 |
| 13 | `13_ethernet_mac_phy_and_mdio.md` | 87 | 11/14 | OpenBMC整合, Bring-up/驗收, 參考資料 |
| 14 | `14_nc_si.md` | 87 | 10/14 | DTS/Binding, OpenBMC整合, Bring-up/驗收, 參考資料 |
| 15 | `15_pcie_management_sideband.md` | 78 | 10/14 | DTS/Binding, Kernel/Runtime, OpenBMC整合, Bring-up/驗收 |
| 16 | `16_usb_gadget.md` | 84 | 11/14 | DTS/Binding, Bring-up/驗收, 參考資料 |
| 17 | `17_mctp_pldm_and_spdm.md` | 184 | 12/14 | Bring-up/驗收, 參考資料 |
| 18 | `18_cpld_fpga_board_glue_logic.md` | 1109 | 14/14 | 無明顯結構缺口 |
| 19 | `19_build_system_and_bsp_structure.md` | 3678 | 14/14 | 無明顯結構缺口 |
| 20 | `20_device_tree_common_patterns_and_troubleshooting.md` | 2169 | 12/14 | 安全性, 效能/資源 |
| 21 | `21_u_boot_kernel_drivers_and_core_services.md` | 2218 | 14/14 | 無明顯結構缺口 |
| 22 | `22_i2c_pmbus_framework.md` | 1343 | 13/14 | 效能/資源 |
| 23 | `23_openbmc_common_projects_and_services_reference.md` | 344 | 11/14 | 定義/目的, 效能/資源, Bring-up/驗收 |
| 24 | `24_sensor_abstraction_layer.md` | 5588 | 12/14 | 定義/目的, 效能/資源 |
| 25 | `25_fan_control_and_thermal_policy.md` | 753 | 13/14 | 定義/目的 |
| 26 | `26_power_control.md` | 672 | 12/14 | 定義/目的, 效能/資源 |
| 27 | `27_inventory_fru_asset_data_model.md` | 866 | 14/14 | 無明顯結構缺口 |
| 28 | `28_logging_event_and_telemetry.md` | 534 | 11/14 | 定義/目的, 角色/架構, DTS/Binding |
| 29 | `29_presence_intrusion_gpio_state_sensor.md` | 587 | 11/14 | 定義/目的, 角色/架構, 效能/資源 |
| 30 | `30_kcs_bt_ssif_espi.md` | 806 | 14/14 | 無明顯結構缺口 |
| 31 | `31_bios_uefi_and_bmc_interaction.md` | 909 | 12/14 | 定義/目的, DTS/Binding |
| 32 | `32_ltpi_and_dc_scm_dc_sci_architecture.md` | 646 | 12/14 | 定義/目的, 效能/資源 |
| 33 | `33_mctp_pldm_spdm.md` | 948 | 14/14 | 無明顯結構缺口 |
| 34 | `34_ipmi_fundamentals.md` | 913 | 11/14 | 定義/目的, DTS/Binding, 效能/資源 |
| 35 | `35_redfish_fundamentals.md` | 1081 | 13/14 | DTS/Binding |
| 36 | `36_network_services.md` | 1062 | 11/14 | 定義/目的, 角色/架構, 效能/資源 |
| 37 | `37_security_baseline.md` | 36 | 7/14 | 定義/目的, 角色/架構, 資料格式/規則, DTS/Binding, Kernel/Runtime, 錯誤/除錯, 效能/資源 |
| 38 | `38_firmware_update.md` | 42 | 6/14 | 定義/目的, 角色/架構, 資料格式/規則, DTS/Binding, Kernel/Runtime, 錯誤/除錯, 效能/資源, Bring-up/驗收 |
| 39 | `39_secure_recovery_rma_and_field_service.md` | 33 | 2/14 | 定義/目的, 角色/架構, 資料格式/規則, 工作流程, DTS/Binding, Kernel/Runtime, OpenBMC整合, 錯誤/除錯, Power/Reset, 效能/資源, Bring-up/驗收, 參考資料 |
| 40 | `40_debug_methodology.md` | 38 | 7/14 | 定義/目的, 角色/架構, 資料格式/規則, 安全性, 效能/資源, Bring-up/驗收, 參考資料 |
| 41 | `41_debug_toolkit.md` | 48 | 3/14 | 定義/目的, 角色/架構, 資料格式/規則, 工作流程, DTS/Binding, Power/Reset, 安全性, 版本/相容性, 效能/資源, Bring-up/驗收, 參考資料 |
| 42 | `42_common_sensor_debug_commands_and_appendix.md` | 447 | 12/14 | 定義/目的, 效能/資源 |
| 43 | `43_performance_resource_and_boot_time.md` | 37 | 2/14 | 定義/目的, 角色/架構, 資料格式/規則, 工作流程, DTS/Binding, Kernel/Runtime, 錯誤/除錯, Power/Reset, 安全性, 版本/相容性, Bring-up/驗收, 參考資料 |
| 44 | `44_general_test_matrix.md` | 15 | 1/14 | 定義/目的, 角色/架構, 資料格式/規則, 工作流程, DTS/Binding, Kernel/Runtime, OpenBMC整合, 錯誤/除錯, 安全性, 版本/相容性, 效能/資源, Bring-up/驗收, 參考資料 |
| 45 | `45_manufacturing_and_factory.md` | 34 | 6/14 | 定義/目的, 角色/架構, DTS/Binding, Kernel/Runtime, 錯誤/除錯, 效能/資源, Bring-up/驗收, 參考資料 |
| 46 | `46_calibration_board_data_and_provisioning.md` | 27 | 5/14 | 定義/目的, 角色/架構, DTS/Binding, Kernel/Runtime, Power/Reset, 版本/相容性, 效能/資源, Bring-up/驗收, 參考資料 |
| 47 | `47_soc_notes_template.md` | 41 | 4/14 | 定義/目的, 角色/架構, 資料格式/規則, 工作流程, 錯誤/除錯, 安全性, 版本/相容性, 效能/資源, Bring-up/驗收, 參考資料 |

## 觀察到的內容差異

### Ch17與Ch32補強後較完整的地方

1. 不只解釋協定，還一路追到kernel source directory與driver initialization。
2. 有具體silicon capability、register address、bit位置或kernel config。
3. 有實際topology、interface name、EID、GPIO line與可重現command。
4. 有case study與期望值，可直接轉成bring-up test case。
5. 有「不要直接套用」的限制，區分spec、SoC capability與platform mapping。

### 其他章節常見差距

- 只有concept與commands，缺少driver probe / initialization flow。
- 有DTS範例，但沒有從DTS node追到kernel object、sysfs、D-Bus與external API。
- 有常見問題，卻沒有fault injection、expected error與recovery pass criteria。
- 缺少silicon-specific register / capability範例與platform case study。
- 缺少版本矩陣、ownership、power-state matrix與BMC reboot continuity。
- 缺少可供manufacturing或validation直接執行的test record表。

## 建議補強優先級

### P0：先補齊章節骨架薄弱者

- Ch37 `37_security_baseline.md`：7/14，優先補 定義/目的、角色/架構、資料格式/規則、DTS/Binding、Kernel/Runtime、錯誤/除錯、效能/資源。
- Ch38 `38_firmware_update.md`：6/14，優先補 定義/目的、角色/架構、資料格式/規則、DTS/Binding、Kernel/Runtime、錯誤/除錯、效能/資源、Bring-up/驗收。
- Ch39 `39_secure_recovery_rma_and_field_service.md`：2/14，優先補 定義/目的、角色/架構、資料格式/規則、工作流程、DTS/Binding、Kernel/Runtime、OpenBMC整合、錯誤/除錯、Power/Reset、效能/資源、Bring-up/驗收、參考資料。
- Ch40 `40_debug_methodology.md`：7/14，優先補 定義/目的、角色/架構、資料格式/規則、安全性、效能/資源、Bring-up/驗收、參考資料。
- Ch41 `41_debug_toolkit.md`：3/14，優先補 定義/目的、角色/架構、資料格式/規則、工作流程、DTS/Binding、Power/Reset、安全性、版本/相容性、效能/資源、Bring-up/驗收、參考資料。
- Ch43 `43_performance_resource_and_boot_time.md`：2/14，優先補 定義/目的、角色/架構、資料格式/規則、工作流程、DTS/Binding、Kernel/Runtime、錯誤/除錯、Power/Reset、安全性、版本/相容性、Bring-up/驗收、參考資料。
- Ch44 `44_general_test_matrix.md`：1/14，優先補 定義/目的、角色/架構、資料格式/規則、工作流程、DTS/Binding、Kernel/Runtime、OpenBMC整合、錯誤/除錯、安全性、版本/相容性、效能/資源、Bring-up/驗收、參考資料。
- Ch45 `45_manufacturing_and_factory.md`：6/14，優先補 定義/目的、角色/架構、DTS/Binding、Kernel/Runtime、錯誤/除錯、效能/資源、Bring-up/驗收、參考資料。
- Ch46 `46_calibration_board_data_and_provisioning.md`：5/14，優先補 定義/目的、角色/架構、DTS/Binding、Kernel/Runtime、Power/Reset、版本/相容性、效能/資源、Bring-up/驗收、參考資料。
- Ch47 `47_soc_notes_template.md`：4/14，優先補 定義/目的、角色/架構、資料格式/規則、工作流程、錯誤/除錯、安全性、版本/相容性、效能/資源、Bring-up/驗收、參考資料。

### P1：加入實作深度

每個協定章建議再增加：

1. Linux source tree與關鍵driver / subsystem entry point。
2. Probe / initialization / TX / RX / interrupt或worker流程。
3. 一個真實SoC或平台case study。
4. register、sysfs、netdev、tty、hwmon、gpiochip等runtime mapping。
5. 正常輸出範例與失敗輸出範例。
6. fault injection、recovery與驗收判定。

### P2：建立跨章共用附件

- Protocol / silicon / kernel / OpenBMC版本相容矩陣。
- Bus、address、GPIO、EID、channel與D-Bus identity總表。
- Power-state可用性矩陣。
- Debug command安全分級：read-only、可能影響service、可能重置硬體、禁止production。
- Case-study模板：拓撲、前置條件、操作、expected result、failure evidence與結論。

## 建議的章節完成標準

一章要同時能回答：「它是什麼」、「封包或訊號怎麼走」、「Linux如何建立runtime object」、「OpenBMC如何使用」、「壞掉時如何定位」、「如何證明通過驗收」。只有概念、DTS或command list其中一項，都不算完成。
