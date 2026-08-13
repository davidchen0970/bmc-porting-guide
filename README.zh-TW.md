# BMC 移植技術參考文件

語言: [English](README.md) | README.zh-TW.md

## 章節索引

### Part 1：硬體抽象層

- [開機流程與 SoC 初始化](01_part_1_hardware_abstraction_layer/01_boot_flow_and_soc_initialization.md)
- [Flash 分割區與儲存架構](01_part_1_hardware_abstraction_layer/02_flash_partition_and_storage_architecture.md)
- [Pinmux、GPIO 常見設計模式](01_part_1_hardware_abstraction_layer/03_pinmux_gpio_common_design_patterns.md)
- [Reset、Clock 與 Power Domain](01_part_1_hardware_abstraction_layer/04_reset_clock_and_power_domain.md)
- [周邊匯流排基礎](01_part_1_hardware_abstraction_layer/05_peripheral_bus_fundamentals.md)
- [CPLD/FPGA 板級 Glue Logic](01_part_1_hardware_abstraction_layer/06_cpld_fpga_board_glue_logic.md)

### Part 2：BSP、Kernel 與 Device Tree

- [Build System 與 BSP 結構](02_part_2_bsp_kernel_and_device_tree/07_build_system_and_bsp_structure.md)
- [Device Tree 常見模式與問題排查](02_part_2_bsp_kernel_and_device_tree/08_device_tree_common_patterns_and_troubleshooting.md)
- [U-Boot、Kernel Drivers 與核心服務](02_part_2_bsp_kernel_and_device_tree/09_u_boot_kernel_drivers_and_core_services.md)

### Part 3：平台監控與控制

- [I2C 與 PMBus Framework](03_part_3_platform_monitoring_and_control/10_i2c_pmbus_framework.md)
- [OpenBMC 常見專案與服務參考](03_part_3_platform_monitoring_and_control/11_openbmc_common_projects_and_services_reference.md)
- [Sensor 抽象層](03_part_3_platform_monitoring_and_control/12_sensor_abstraction_layer.md)
- [Fan Control 與 Thermal Policy](03_part_3_platform_monitoring_and_control/13_fan_control_and_thermal_policy.md)
- [Power Control](03_part_3_platform_monitoring_and_control/14_power_control.md)
- [Inventory、FRU 與資產資料模型](03_part_3_platform_monitoring_and_control/15_inventory_fru_asset_data_model.md)
- [Logging、Event 與 Telemetry](03_part_3_platform_monitoring_and_control/16_logging_event_and_telemetry.md)
- [Presence、Intrusion、GPIO 與 State Sensor](03_part_3_platform_monitoring_and_control/17_presence_intrusion_gpio_state_sensor.md)

### Part 4：Host Communication

- [KCS、BT、SSIF 與 eSPI](04_part_4_host_communication/18_kcs_bt_ssif_espi.md)
- [BIOS/UEFI 與 BMC 互動](04_part_4_host_communication/19_bios_uefi_and_bmc_interaction.md)

### Part 5：管理介面與網路

- [MCTP、PLDM 與 SPDM](05_part_5_management_interfaces_and_networking/21_mctp_pldm_spdm.md)
- [IPMI 基礎](05_part_5_management_interfaces_and_networking/22_ipmi_fundamentals.md)
- [Redfish 基礎](05_part_5_management_interfaces_and_networking/23_redfish_fundamentals.md)
- [Network Services](05_part_5_management_interfaces_and_networking/24_network_services.md)

### Part 6：安全性與韌體維護

- [Security Baseline](06_part_6_security_and_firmware_maintenance/25_security_baseline.md)
- [Firmware Update](06_part_6_security_and_firmware_maintenance/26_firmware_update.md)
- [Secure Recovery、RMA 與 Field Service](06_part_6_security_and_firmware_maintenance/27_secure_recovery_rma_and_field_service.md)

### Part 7：Debug、效能與測試

- [Debug Methodology](07_part_7_debugging_performance_and_testing/28_debug_methodology.md)
- [Debug Toolkit](07_part_7_debugging_performance_and_testing/29_debug_toolkit.md)
- [常見 Sensor Debug Commands 與附錄](07_part_7_debugging_performance_and_testing/30_common_sensor_debug_commands_and_appendix.md)
- [效能、資源與開機時間](07_part_7_debugging_performance_and_testing/31_performance_resource_and_boot_time.md)
- [General Test Matrix](07_part_7_debugging_performance_and_testing/32_general_test_matrix.md)

### Part 8：製造與量產

- [Manufacturing 與 Factory](08_part_8_manufacturing_and_production/33_manufacturing_and_factory.md)
- [Calibration、Board Data 與 Provisioning](08_part_8_manufacturing_and_production/34_calibration_board_data_and_provisioning.md)

### Part 9：平台特定筆記

- [SoC Notes Template](09_part_9_platform_specific_notes/35_soc_notes_template.md)

### Part 10：附錄

- [常用縮寫與術語](10_part_10_appendices/A01_common_abbreviations_and_terms.md)
- [常用指令參考](10_part_10_appendices/A02_common_commands_reference.md)
- [Log 收集 Package Template](10_part_10_appendices/A03_log_collection_package_template.md)
- [Bring-up 與 Acceptance Checklist](10_part_10_appendices/A04_bring_up_and_acceptance_checklist.md)
- [Documentation Template](10_part_10_appendices/A05_documentation_template.md)
