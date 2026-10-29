---
title: HydroponicsIDF Knowledge Base
tags:
  - dashboard
  - moc
  - index
created: 2026-10-28
---

# 📚 HydroponicsIDF Knowledge Base & Architecture Index

Welcome to the central Knowledge Base of the **HydroponicsIDF** firmware.

---

## 🧭 Vault Navigation

```
HydroponicsIDF Vault
├── 00 - Overview
│   └── [[Home|Dashboard & Quick Start]]
├── 01 - Architecture
│   ├── [[System Architecture & FreeRTOS]]
│   └── [[Boot Sequence & Power Recovery]]
├── 02 - Subsystems
│   ├── [[Safety Subsystem & Watchdogs]]
│   ├── [[Dosing & Actuator Control]]
│   ├── [[Sensor Telemetry & Calibration]]
│   ├── [[Networking & MQTT Engine]]
│   ├── [[Web Portal, OTA & Provisioning]]
│   └── [[Human-Machine Interfaces & Diagnostics]]
├── 03 - Reference
│   ├── [[Pin & Hardware Mapping Matrix]]
│   ├── [[MQTT Protocol Specification]]
│   └── [[Safety Faults & Parameter Catalog]]
├── 04 - Defense Guide
│   └── [[Technical Defense & Architecture Q&A]]
├── 05 - Audit
│   ├── [[00 - Executive Summary & Risk Matrix]]
│   ├── [[01 - Critical Safety Vulnerabilities & Timing Hazards]]
│   ├── [[02 - Spaghetti Architecture & Code Smells]]
│   ├── [[03 - Networking, OTA & Cybersecurity Vulnerabilities]]
│   ├── [[04 - Sensor & Actuator Physics Critique]]
│   └── [[05 - Refactoring Roadmap & Remediation Plan]]
└── 06 - Device & PCB Guide
    ├── [[00 - Hardware Architecture & System Topology]]
    ├── [[01 - Schematic & Circuit Design Guide]]
    ├── [[02 - PCB Layout & EMC Guidelines]]
    ├── [[03 - Enclosure & Environmental Protection (IP65)]]
    ├── [[04 - Assembly, Bring-Up & Field Commissioning]]
    └── [[05 - Hardware Troubleshooting Matrix]]
```

---

## 🚀 Key Functional Areas

| Note | Focus Subsystems | Key Files |
|---|---|---|
| **[[System Architecture & FreeRTOS]]** | Threading model, task priorities, synchronization, anti-priority-inversion, memory layout | `runtime_tasks.c`, `partitions.csv` |
| **[[Boot Sequence & Power Recovery]]** | Clamped GPIOs, early relay protection, NVS fault recording, autostart retry loops | `main.c`, `wifi_manager.c`, `web_portal.c` |
| **[[Safety Subsystem & Watchdogs]]** | Fault bitmasks, Safe Mode, dual-tier WDT, dosing timers, frozen sensor checks | `runtime_safety.c`, `safety_config.h` |
| **[[Dosing & Actuator Control]]** | Peristaltic pumps, queue discipline, mutual exclusion, pulse commands, circulation pump | `actuator_control.c`, `runtime_tasks.c` |
| **[[Sensor Telemetry & Calibration]]** | ADS1115 ADC, DS18B20 1-Wire, SHT31, pH UART/I2C, 25°C temp compensation, linear calibration | `sensor_telemetry.c`, `sensor_calibration_nvs.c` |
| **[[Networking & MQTT Engine]]** | Telemetry publish cycle, snapshot fallback cache, exact MQTT dispatch, auto-resubscription | `mqtt_manager.c`, `wifi_manager.c` |
| **[[Web Portal, OTA & Provisioning]]** | Web portal, 192.168.4.1 setup AP, setup button monitor, HTTP OTA downloads | `web_portal.c`, `ota_update.c`, `setup_button.c` |
| **[[Human-Machine Interfaces & Diagnostics]]** | 20x4 LCD, marquee scroll task, connection/fault LEDs, syslog UDP broadcast | `lcd_status.c`, `indicator_led.c`, `syslog.c` |
| **[[Pin & Hardware Mapping Matrix]]** | Hardware pinouts, voltage levels, internal pullups, I2C/UART/1-Wire buses | `pin_config.h` |
| **[[MQTT Protocol Specification]]** | Comprehensive MQTT topic dictionary, payloads, QoS, retain flags, JSON formats | `topics.md`, `mqtt_manager.c` |
| **[[Safety Faults & Parameter Catalog]]** | Fault codes F-001 through F-013, thresholds, clearing commands, recovery actions | `safety_config.h`, `runtime_safety.c` |
| **[[Technical Defense & Architecture Q&A]]** | Deep engineering defense questions & comprehensive technical answers | `sidang.md` |
| **[[00 - Executive Summary & Risk Matrix]]** | Comprehensive safety vulnerability matrix, CVSS severity ratings | Entire Repository |
| **[[01 - Critical Safety Vulnerabilities & Timing Hazards]]** | Blocking pulse TWDT panic trap, `xQueueReset` SMP race, software interlocks | `actuator_control.c`, `runtime_tasks.c` |
| **[[02 - Spaghetti Architecture & Code Smells]]** | 1,870-line God-file, brittle JSON parser, pin mismatches, dead code | `sensor_telemetry.c`, `pin_config.h` |
| **[[03 - Networking, OTA & Cybersecurity Vulnerabilities]]** | Unsigned plain HTTP OTA RCE, open Wi-Fi AP, plaintext MQTT, NVS wear DoS | `ota_update.c`, `web_portal.c` |
| **[[04 - Sensor & Actuator Physics Critique]]** | 16-bit noise illusion, 2-point linear breakdown, dry-run pump burnout | `sensor_telemetry.c` |
| **[[05 - Refactoring Roadmap & Remediation Plan]]** | Production C patches for non-blocking timers, safe queue drains, `cJSON` | Architecture Patches |
| **[[00 - Hardware Architecture & System Topology]]** | Master system topology, power tree (24V/12V/5V/3V3), galvanic isolation domains | Hardware Specs |
| **[[01 - Schematic & Circuit Design Guide]]** | Low-side MOSFET switching, freewheeling flyback clamp, electromechanical SPDT chemical interlock, high-Z AFE | Circuit Schematics |
| **[[02 - PCB Layout & EMC Guidelines]]** | 4-layer stackup (SIG-GND-PWR-SIG), 3-zone partitioning, 6.3mm AC creepage isolation slots, BNC guard rings | PCB CAD / Layout |
| **[[03 - Enclosure & Environmental Protection (IP65)]]** | IP65 enclosure engineering, polycarbonate UV/acid resistance, cable glands with drip loops, ePTFE vents | Mechanical Specs |
| **[[04 - Assembly, Bring-Up & Field Commissioning]]** | Ultrasonic flux decontamination, cold-board testing, strapping pin checks, pH/EC multi-point calibration SOP | Production SOPs |
| **[[05 - Hardware Troubleshooting Matrix]]** | Hardware diagnostic decision tree for I2C lockups, 1-Wire discovery failures, ground loops, MOSFET saturation | Field Maintenance |
