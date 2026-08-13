# BMC Porting Technical Reference

Languages: README.md | [中文](README.zh-TW.md)

## Chapter Index

### Part 1: Hardware Abstraction Layer

- [Boot Flow and SoC Initialization](01_part_1_hardware_abstraction_layer/01_boot_flow_and_soc_initialization.md)
- [Flash Partition and Storage Architecture](01_part_1_hardware_abstraction_layer/02_flash_partition_and_storage_architecture.md)
- [Pinmux GPIO Common Design Patterns](01_part_1_hardware_abstraction_layer/03_pinmux_gpio_common_design_patterns.md)
- [Reset, Clock, and Power Domain](01_part_1_hardware_abstraction_layer/04_reset_clock_and_power_domain.md)
- [Peripheral Bus Fundamentals](01_part_1_hardware_abstraction_layer/05_peripheral_bus_fundamentals.md)
- [CPLD/FPGA Board Glue Logic](01_part_1_hardware_abstraction_layer/06_cpld_fpga_board_glue_logic.md)

### Part 2: BSP, Kernel, and Device Tree

- [Build System and BSP Structure](02_part_2_bsp_kernel_and_device_tree/07_build_system_and_bsp_structure.md)
- [Device Tree Common Patterns and Troubleshooting](02_part_2_bsp_kernel_and_device_tree/08_device_tree_common_patterns_and_troubleshooting.md)
- [U-Boot, Kernel Drivers, and Core Services](02_part_2_bsp_kernel_and_device_tree/09_u_boot_kernel_drivers_and_core_services.md)

### Part 3: Platform Monitoring and Control

- [I2C and PMBus Framework](03_part_3_platform_monitoring_and_control/10_i2c_pmbus_framework.md)
- [OpenBMC Common Projects and Services Reference](03_part_3_platform_monitoring_and_control/11_openbmc_common_projects_and_services_reference.md)
- [Sensor Abstraction Layer](03_part_3_platform_monitoring_and_control/12_sensor_abstraction_layer.md)
- [Fan Control and Thermal Policy](03_part_3_platform_monitoring_and_control/13_fan_control_and_thermal_policy.md)
- [Power Control](03_part_3_platform_monitoring_and_control/14_power_control.md)
- [Inventory, FRU, and Asset Data Model](03_part_3_platform_monitoring_and_control/15_inventory_fru_asset_data_model.md)
- [Logging, Event, and Telemetry](03_part_3_platform_monitoring_and_control/16_logging_event_and_telemetry.md)
- [Presence, Intrusion, GPIO, and State Sensor](03_part_3_platform_monitoring_and_control/17_presence_intrusion_gpio_state_sensor.md)

### Part 4: Host Communication

- [KCS, BT, SSIF, and eSPI](04_part_4_host_communication/18_kcs_bt_ssif_espi.md)
- [BIOS/UEFI and BMC Interaction](04_part_4_host_communication/19_bios_uefi_and_bmc_interaction.md)

### Part 5: Management Interfaces and Networking

- [MCTP, PLDM, and SPDM](05_part_5_management_interfaces_and_networking/21_mctp_pldm_spdm.md)
- [IPMI Fundamentals](05_part_5_management_interfaces_and_networking/22_ipmi_fundamentals.md)
- [Redfish Fundamentals](05_part_5_management_interfaces_and_networking/23_redfish_fundamentals.md)
- [Network Services](05_part_5_management_interfaces_and_networking/24_network_services.md)

### Part 6: Security and Firmware Maintenance

- [Security Baseline](06_part_6_security_and_firmware_maintenance/25_security_baseline.md)
- [Firmware Update](06_part_6_security_and_firmware_maintenance/26_firmware_update.md)
- [Secure Recovery, RMA, and Field Service](06_part_6_security_and_firmware_maintenance/27_secure_recovery_rma_and_field_service.md)

### Part 7: Debugging, Performance, and Testing

- [Debug Methodology](07_part_7_debugging_performance_and_testing/28_debug_methodology.md)
- [Debug Toolkit](07_part_7_debugging_performance_and_testing/29_debug_toolkit.md)
- [Common Sensor Debug Commands and Appendix](07_part_7_debugging_performance_and_testing/30_common_sensor_debug_commands_and_appendix.md)
- [Performance, Resource, and Boot Time](07_part_7_debugging_performance_and_testing/31_performance_resource_and_boot_time.md)
- [General Test Matrix](07_part_7_debugging_performance_and_testing/32_general_test_matrix.md)

### Part 8: Manufacturing and Production

- [Manufacturing and Factory](08_part_8_manufacturing_and_production/33_manufacturing_and_factory.md)
- [Calibration, Board Data, and Provisioning](08_part_8_manufacturing_and_production/34_calibration_board_data_and_provisioning.md)

### Part 9: Platform-Specific Notes

- [SoC Notes Template](09_part_9_platform_specific_notes/35_soc_notes_template.md)

### Part 10: Appendices

- [Common Abbreviations and Terms](10_part_10_appendices/A01_common_abbreviations_and_terms.md)
- [Common Commands Reference](10_part_10_appendices/A02_common_commands_reference.md)
- [Log Collection Package Template](10_part_10_appendices/A03_log_collection_package_template.md)
- [Bring-up and Acceptance Checklist](10_part_10_appendices/A04_bring_up_and_acceptance_checklist.md)
- [Documentation Template](10_part_10_appendices/A05_documentation_template.md)
