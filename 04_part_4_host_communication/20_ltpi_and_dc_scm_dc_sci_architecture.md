# LTPI and DC-SCM/DC-SCI Architecture

## 1. Purpose and Scope

This chapter describes the Low-voltage differential signaling Tunneling Protocol and Interface (LTPI), its role in Data Center Secure Control Module platforms, and the considerations required when porting OpenBMC to an LTPI-based platform.

The intended audience includes BMC firmware, OpenBMC, CPLD/FPGA, board bring-up, BIOS, validation, and manufacturing engineers.

## 2. Terminology

- **BMC**: Baseboard Management Controller
- **SCM**: Secure Control Module
- **HPM**: Host Processor Module
- **DC-SCM**: Data Center Secure Control Module
- **DC-SCI**: Data Center Secure Control Interface
- **LTPI**: LVDS Tunneling Protocol and Interface
- **LVDS**: Low-voltage differential signaling
- **LL GPIO**: Low-latency GPIO
- **NL GPIO**: Normal-latency GPIO
- **OEM channel**: Vendor-defined LTPI data channel

## 3. Architectural Overview

LTPI transports multiple low-speed management interfaces between the SCM and HPM over a reduced-pin-count LVDS link. Typical tunneled interfaces include GPIO, I2C/SMBus, UART, OEM-defined messages, and implementation-specific memory-mapped data channels.

```text
+---------------------------+       +---------------------------+
| Secure Control Module     |       | Host Processor Module     |
|  BMC <-> FPGA/CPLD        |<----->|  FPGA/CPLD <-> Host SoC   |
+---------------------------+ LTPI  +---------------------------+
```

Document ownership, direction, polarity, latency, reset behavior, and power-domain dependencies for every tunneled signal.

## 4. LTPI Compared with SGPIO

LTPI provides higher aggregate bandwidth, multiple logical channels, link initialization and capability negotiation, reduced connector pin count, and improved scalability for modular SCM/HPM designs. It is not a transparent electrical replacement for SGPIO. Protocol state, latency, reset behavior, defaults, and failure recovery must be designed explicitly.

## 5. Link Initialization

A platform may implement physical detection, synchronization, link training, capability advertisement and negotiation, channel activation, normal operation, and error recovery or retraining.

Define:

- Endpoint responsible for initialization
- Signal defaults before link establishment
- Timeouts and retry policy
- Validity point for tunneled signals
- SCM, HPM, BMC, and FPGA/CPLD reset behavior
- Host power sequencing dependencies

## 6. GPIO Channels

Use low-latency GPIO for platform-critical controls and indications such as power, reset, processor errors, interrupts, and boot handshakes. Use normal-latency GPIO for presence, identification, maintenance, and noncritical status signals.

For every GPIO, document the signal name, direction, active polarity, default state, latency class, reset behavior, power domain, endpoint mapping, and OpenBMC consumer or producer.

## 7. I2C and SMBus Tunneling

Define controller and target ownership, bus numbering, address mapping, clock frequency, clock stretching, arbitration, timeout, recovery, error propagation, and accessibility in each host power state.

OpenBMC may require a virtual or FPGA-backed Linux I2C adapter. Standard Linux bus recovery cannot be assumed to control the remote physical bus directly.

## 8. UART Tunneling

UART channels may carry host console, BIOS debug, FPGA diagnostics, manufacturing output, or recovery access. Document baud rate, data format, flow control, ownership, mux behavior, reset behavior, access control, and interaction with Serial over LAN.

## 9. OEM and Data Channels

Specify message format, version, endianness, sequence handling, timeout behavior, error codes, compatibility, authorization assumptions, and firmware dependencies. Avoid undocumented OEM channels for platform-critical behavior.

## 10. CPLD and FPGA Responsibilities

Typical endpoint responsibilities include the LVDS physical interface, training, channel aggregation, GPIO synchronization, I2C/SMBus and UART tunneling, register access, diagnostics, and recovery.

Recommended registers include link state, negotiated speed and capabilities, training and retraining counters, frame/CRC and timeout counters, enabled channels, endpoint versions, last failure reason, and GPIO snapshots.

## 11. OpenBMC Integration

OpenBMC may access LTPI through memory-mapped, I2C, or SPI FPGA/CPLD registers, GPIO controllers, Linux I2C adapters, UART drivers, or platform-specific D-Bus services.

Separate physical-link management, channel transport, Linux abstractions, OpenBMC services, and Redfish/IPMI exposure. Report explicit LTPI link health rather than only a collection of downstream sensor, GPIO, or I2C failures.

## 12. Device Tree Considerations

Device Tree nodes may describe endpoint registers, interrupts, resets, GPIO controllers, I2C adapters, UART endpoints, link status, and firmware version interfaces.

```dts
ltpi_controller: ltpi-controller@0 {
    compatible = "vendor,platform-ltpi";
    reg = <0x0 0x1000>;
    interrupts = <...>;
    status = "okay";

    gpio-controller;
    #gpio-cells = <2>;
};
```

This example is conceptual. Production Device Trees must match the actual Linux binding and hardware implementation.

## 13. Power and Reset Sequencing

Validate AC application, standby initialization, BMC cold and warm reset, FPGA/CPLD reset, HPM reset, host power-on/off, host warm reset, firmware update, and SCM/HPM replacement. Platform-critical outputs must have safe hardware defaults whenever the LTPI link is unavailable.

## 14. Security Considerations

Review the SCM/HPM trust boundary, FPGA/CPLD firmware authentication and rollback, register access, debug disablement, OEM-channel validation, tamper logging, and recovery behavior. Do not assume an internal board-level connection is inherently trusted.

## 15. Debug Methodology

### 15.1 Physical Layer

Check power rails, reference clocks, resets, LVDS polarity, lane connectivity, signal integrity, and pin configuration.

### 15.2 Link Layer

Check training state, capability negotiation, transfer rate, frame/CRC errors, timeouts, retraining events, and endpoint firmware versions.

### 15.3 Channel Layer

Validate GPIO values and polarity, I2C/SMBus completion, UART paths, OEM sequencing, and interrupt delivery independently.

### 15.4 OpenBMC Layer

Check driver binding, Device Tree, sysfs, D-Bus, systemd, journal logs, sensor dependencies, and power-control transitions.

## 16. Common Failure Patterns

- **Link does not train**: reset, clock, LVDS polarity, rate mismatch, incompatible endpoint firmware, pinmux, or sequencing issue.
- **Link repeatedly retrains**: signal integrity, unstable clock or power, endpoint watchdog, excessive frame errors, or firmware defect.
- **GPIO state is incorrect**: polarity, index, latency-class mapping, unsafe defaults, stale data, or reset-domain mismatch.
- **Remote I2C is unavailable**: link/channel down, ownership mismatch, unpowered remote bus, timeout, unsupported clock stretching, or bridge error.

## 17. Bring-up Checklist

- [ ] Confirm LTPI, DC-SCM, and DC-SCI revisions
- [ ] Confirm SCM and HPM endpoint implementations
- [ ] Verify LVDS connectivity, polarity, clocks, and resets
- [ ] Confirm supported and negotiated link rates and capabilities
- [ ] Validate LL GPIO and NL GPIO mapping
- [ ] Validate I2C/SMBus, UART, and OEM channels
- [ ] Test BMC, host, and FPGA/CPLD reset scenarios
- [ ] Test link loss, recovery, diagnostics, and event logging
- [ ] Verify safe power-control defaults
- [ ] Verify firmware compatibility and manufacturing coverage

## 18. Platform Documentation Requirements

Provide the LTPI block diagram, channel maps, GPIO/I2C/UART/OEM specifications, power and reset sequence, endpoint register map, firmware compatibility matrix, recovery policy, manufacturing test, and field-debug procedure.

## 19. References

Consult the applicable OCP DC-SCM, DC-SCI, and LTPI specifications, endpoint IP documentation, platform schematics, FPGA/CPLD register specifications, Linux driver documentation, and OpenBMC platform documentation.
