# BMC 移植技術參考文件

語言：[English](README.md) | **繁體中文**

## 章節索引

### Part 1：硬體抽象層

- [開機流程與 SoC 初始化](01_part_1_hardware_abstraction_layer/01_boot_flow_and_soc_initialization.md)
- [Flash 分割區與儲存架構](01_part_1_hardware_abstraction_layer/02_flash_partition_and_storage_architecture.md)
- [Pinmux 與 GPIO 常見設計模式](01_part_1_hardware_abstraction_layer/03_pinmux_gpio_common_design_patterns.md)
- [Reset、Clock 與 Power Domain](01_part_1_hardware_abstraction_layer/04_reset_clock_and_power_domain.md)
- [周邊匯流排基礎](01_part_1_hardware_abstraction_layer/05_peripheral_bus_fundamentals.md)
- [I2C、SMBus 與 PMBus](01_part_1_hardware_abstraction_layer/06_i2c_smbus_and_pmbus.md)
- [SPI](01_part_1_hardware_abstraction_layer/07_spi.md)
- [UART 與序列主控台](01_part_1_hardware_abstraction_layer/08_uart_and_serial_console.md)
- [ADC 與 IIO](01_part_1_hardware_abstraction_layer/09_adc_and_iio.md)
- [PWM 與 Tach](01_part_1_hardware_abstraction_layer/10_pwm_and_tach.md)
- [PECI 與 APML](01_part_1_hardware_abstraction_layer/11_peci_and_apml.md)
- [eSPI、LPC 與 Host Interface](01_part_1_hardware_abstraction_layer/12_espi_lpc_and_host_interface.md)
- [Ethernet MAC、PHY 與 MDIO](01_part_1_hardware_abstraction_layer/13_ethernet_mac_phy_and_mdio.md)
- [NC-SI](01_part_1_hardware_abstraction_layer/14_nc_si.md)
- [PCIe Management Sideband](01_part_1_hardware_abstraction_layer/15_pcie_management_sideband.md)
- [USB Gadget](01_part_1_hardware_abstraction_layer/16_usb_gadget.md)
- [MCTP、PLDM 與 SPDM](01_part_1_hardware_abstraction_layer/17_mctp_pldm_and_spdm.md)
- [CPLD/FPGA 板級 Glue Logic](01_part_1_hardware_abstraction_layer/18_cpld_fpga_board_glue_logic.md)

### Part 2：BSP、Kernel 與 Device Tree

- [Build System 與 BSP 結構](02_part_2_bsp_kernel_and_device_tree/19_build_system_and_bsp_structure.md)
- [Device Tree 常見模式與問題排查](02_part_2_bsp_kernel_and_device_tree/20_device_tree_common_patterns_and_troubleshooting.md)
- [U-Boot、Kernel Drivers 與核心服務](02_part_2_bsp_kernel_and_device_tree/21_u_boot_kernel_drivers_and_core_services.md)

### Part 3：平台監控與控制

- [I2C 與 PMBus Framework](03_part_3_platform_monitoring_and_control/22_i2c_pmbus_framework.md)
- [OpenBMC 常見專案與服務參考](03_part_3_platform_monitoring_and_control/23_openbmc_common_projects_and_services_reference.md)
- [Sensor 抽象層](03_part_3_platform_monitoring_and_control/24_sensor_abstraction_layer.md)
- [Fan Control 與 Thermal Policy](03_part_3_platform_monitoring_and_control/25_fan_control_and_thermal_policy.md)
- [Power Control](03_part_3_platform_monitoring_and_control/26_power_control.md)
- [Inventory、FRU 與資產資料模型](03_part_3_platform_monitoring_and_control/27_inventory_fru_asset_data_model.md)
- [Logging、Event 與 Telemetry](03_part_3_platform_monitoring_and_control/28_logging_event_and_telemetry.md)
- [Presence、Intrusion、GPIO 與 State Sensor](03_part_3_platform_monitoring_and_control/29_presence_intrusion_gpio_state_sensor.md)

### Part 4：Host Communication

- [KCS、BT、SSIF 與 eSPI](04_part_4_host_communication/30_kcs_bt_ssif_espi.md)
- [BIOS/UEFI 與 BMC 互動](04_part_4_host_communication/31_bios_uefi_and_bmc_interaction.md)
- [LTPI 與 DC-SCM/DC-SCI 架構](04_part_4_host_communication/32_ltpi_and_dc_scm_dc_sci_architecture.md)

### Part 5：管理介面與網路

- [MCTP、PLDM 與 SPDM](05_part_5_management_interfaces_and_networking/33_mctp_pldm_spdm.md)
- [IPMI 基礎](05_part_5_management_interfaces_and_networking/34_ipmi_fundamentals.md)
- [Redfish 基礎](05_part_5_management_interfaces_and_networking/35_redfish_fundamentals.md)
- [網路服務](05_part_5_management_interfaces_and_networking/36_network_services.md)

### Part 6：安全性與韌體維護

- [安全性基準](06_part_6_security_and_firmware_maintenance/37_security_baseline.md)
- [韌體更新](06_part_6_security_and_firmware_maintenance/38_firmware_update.md)
- [安全復原、RMA 與現場服務](06_part_6_security_and_firmware_maintenance/39_secure_recovery_rma_and_field_service.md)

### Part 7：除錯、效能與測試

- [除錯方法論](07_part_7_debugging_performance_and_testing/40_debug_methodology.md)
- [除錯工具集](07_part_7_debugging_performance_and_testing/41_debug_toolkit.md)
- [常見 Sensor 除錯指令與附錄](07_part_7_debugging_performance_and_testing/42_common_sensor_debug_commands_and_appendix.md)
- [效能、資源與開機時間](07_part_7_debugging_performance_and_testing/43_performance_resource_and_boot_time.md)
- [通用測試矩陣](07_part_7_debugging_performance_and_testing/44_general_test_matrix.md)

### Part 8：製造與量產

- [製造與工廠](08_part_8_manufacturing_and_production/45_manufacturing_and_factory.md)
- [校正、板級資料與佈建](08_part_8_manufacturing_and_production/46_calibration_board_data_and_provisioning.md)

### Part 9：平台特定筆記

- [SoC 筆記範本](09_part_9_platform_specific_notes/47_soc_notes_template.md)

### Part 10：附錄

- [常用縮寫與術語](10_part_10_appendices/A01_common_abbreviations_and_terms.md)
- [常用指令參考](10_part_10_appendices/A02_common_commands_reference.md)
- [Log 收集套件範本](10_part_10_appendices/A03_log_collection_package_template.md)
- [Bring-up 與驗收檢查表](10_part_10_appendices/A04_bring_up_and_acceptance_checklist.md)
- [文件範本](10_part_10_appendices/A05_documentation_template.md)
