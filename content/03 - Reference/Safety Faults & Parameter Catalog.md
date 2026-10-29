---
title: Safety Faults & Parameter Catalog
tags:
  - safety
  - faults
  - parameters
  - reference
created: 2026-10-28
---

# ⚠️ Safety Faults, Bitmasks & Configurable Parameter Catalog

This reference catalog details all 13 safety fault definitions, their hexadecimal bitmasks, detection criteria, and all configurable safety threshold parameters defined in `safety_config.h` and `runtime_safety.c`.

---

## 1. Safety Fault Definitions (`safety_fault_t`)

Faults are maintained internally as bits in a 32-bit unsigned atomic integer (`s_safety.faults`). Multiple faults can be latched simultaneously.

| Fault Code | Enum Identifier | Bitmask (Hex) | Bit Position | Text Description | Triggering Conditions |
|:---:|---|:---:|:---:|---|---|
| **F-001** | `SAFETY_FAULT_DOSE_A` | `0x0001` | Bit 0 | `"Dose A"` | Nutrient A pump continuously ON > 120s, or pulse requested > 120s, or TDS failed to shift $\ge 5.0$ ppm within 120s of dosing. |
| **F-002** | `SAFETY_FAULT_DOSE_B` | `0x0002` | Bit 1 | `"Dose B"` | Nutrient B pump continuously ON > 120s, or pulse requested > 120s, or TDS failed to shift $\ge 5.0$ ppm within 120s of dosing. |
| **F-003** | `SAFETY_FAULT_PH_UP` | `0x0004` | Bit 2 | `"pH Up"` | pH Up pump ON > 60s, or pulse > 60s, or pH critical high (> 9.0), or pH rising faster than 1.0/min, or no pH shift within 90s of dosing. |
| **F-004** | `SAFETY_FAULT_PH_DOWN` | `0x0008` | Bit 3 | `"pH Down"` | pH Down pump ON > 60s, or pulse > 60s, or pH critical low (< 4.0), or pH falling faster than 1.0/min, or no pH shift within 90s of dosing. |
| **F-005** | `SAFETY_FAULT_PH_INTERLOCK` | `0x0010` | Bit 4 | `"pH Interlock"` | Attempted simultaneous activation of pH Up and pH Down pumps (severe chemical hazard). |
| **F-006** | `SAFETY_FAULT_VALVE` | `0x0020` | Bit 5 | `"Valve"` | Solenoid valve remains continuously ON while water level is LOW for > 10 minutes (reservoir leak or float switch stuck). |
| **F-007** | `SAFETY_FAULT_GROWLIGHT` | `0x0040` | Bit 6 | `"Growlight"` | Relay 1 growlight circuit failure or thermal contactor trip. |
| **F-008** | `SAFETY_FAULT_TDS_SENSOR` | `0x0080` | Bit 7 | `"TDS Sensor"` | TDS reading < 50 ppm or > 2500 ppm continuously for > 10 seconds, or sensor reading frozen for 300 consecutive samples (150s). |
| **F-009** | `SAFETY_FAULT_PH_SENSOR` | `0x0100` | Bit 8 | `"pH Sensor"` | pH reading < 4.5 or > 8.5, or sensor reading frozen for 300 consecutive samples (150s). (Suppressed during 2-minute boot warmup). |
| **F-010** | `SAFETY_FAULT_I2C_BUS` | `0x0200` | Bit 9 | `"I2C Bus"` | I2C0 communication failure, bus lock timeout (> 500 ms), or missing ACK from ADS1115 / PCF8574 LCD backpack. |
| **F-011** | `SAFETY_FAULT_WATER_TEMP` | `0x0400` | Bit 10 | `"Water Temp"` | Water temperature < 10.0°C or > 36.0°C continuously for > 60 seconds, or critical high > 42.0°C. |
| **F-012** | `SAFETY_FAULT_POWER` | `0x0800` | Bit 11 | `"Power"` | Hardware Brownout reset detected during boot triage (severe DC supply sag). |
| **F-013** | `SAFETY_FAULT_WDT` | `0x1000` | Bit 12 | `"Watchdog"` | Hardware Task Watchdog (TWDT) reset, or software task heartbeat stalled for > 15 seconds (Dosing, Sensor, Comm, or Safety). |

---

## 2. Safety Parameter Thresholds (`safety_config.h`)

All timing, temperature, and chemical safety limits are configured as compile-time constants:

### System & Watchdogs
| Macro Identifier | Default Value | Unit | Description |
|---|:---:|:---:|---|
| `DEV_MODE` | `0` | Flag | Set to `1` to disable all safety trips for bench testing. **Must be 0 in production.** |
| `SAFETY_TASK_STACK` | `6144` | Bytes | Stack allocation for `runtime_safety_task`. |
| `SAFETY_TASK_PRIORITY` | `4` | Level | Highest priority in runtime; preempts all other tasks. |
| `SAFETY_INTERVAL_MS` | `500` | ms | Base execution period for `SafetyTask`. |
| `SAFETY_HEARTBEAT_TIMEOUT_MS` | `15000` | ms | Maximum permitted time for any task's heartbeat counter to stall before F-013 trips. |
| `SAFETY_WDT_TIMEOUT_SEC` | `8` | Seconds | Hardware Task Watchdog Timer timeout before triggering panic reset. |
| `SAFETY_RESET_WINDOW_SEC` | `300` | Seconds | Window for evaluating rapid reboot storms. |
| `SAFETY_RESET_MAX_COUNT` | `5` | Counts | Maximum allowed reboots in window before dropping to safe state. |

### Dosing & Valve Supervision
| Macro Identifier | Default Value | Unit | Description |
|---|:---:|:---:|---|
| `SAFETY_DOSE_WINDOW_MS` | `300000` | ms (5 min) | Rolling time window for evaluating cumulative chemical injection. |
| `SAFETY_MAX_DOSE_A_MS` | `120000` | ms (2 min) | Maximum continuous run time for Nutrient A pump before F-001 trips. |
| `SAFETY_MAX_DOSE_B_MS` | `120000` | ms (2 min) | Maximum continuous run time for Nutrient B pump before F-002 trips. |
| `SAFETY_MAX_DOSE_PH_UP_MS` | `60000` | ms (1 min) | Maximum continuous run time for pH Up pump before F-003 trips. |
| `SAFETY_MAX_DOSE_PH_DOWN_MS` | `60000` | ms (1 min) | Maximum continuous run time for pH Down pump before F-004 trips. |
| `SAFETY_FILL_TIMEOUT_MS` | `600000` | ms (10 min) | Maximum allowed time for solenoid valve to stay open if tank remains LOW before F-006 trips. |

### Water Temperature Thresholds
| Macro Identifier | Default Value | Unit | Description |
|---|:---:|:---:|---|
| `SAFETY_TEMP_LOW` | `10.0` | °C | Minimum safe root-zone water temperature. |
| `SAFETY_TEMP_HIGH` | `36.0` | °C | Maximum safe root-zone water temperature. |
| `SAFETY_TEMP_CRITICAL` | `42.0` | °C | Critical upper limit; triggers immediate shutdown. |
| `SAFETY_TEMP_FROZEN_EPSILON` | `0.02` | °C | Maximum temperature change considered "frozen/stuck". |
| `SAFETY_TEMP_FROZEN_SAMPLES` | `300` | Samples | Number of consecutive safety loops (150s) to trip frozen temp fault. |
| `SAFETY_TEMP_OOR_FAULT_MS` | `60000` | ms (1 min) | Duration water temp must remain out-of-range before F-011 trips. |

### pH & TDS Sensor Supervision
| Macro Identifier | Default Value | Unit | Description |
|---|:---:|:---:|---|
| `SAFETY_ENABLE_PH_TDS_CHECKS` | `1` | Flag | Master switch for pH/TDS sensor health evaluation. |
| `SAFETY_PH_WARMUP_MS` | `120000` | ms (2 min) | Boot stabilization delay; suppresses false trips while probe polarizes. |
| `SAFETY_PH_MIN` | `4.5` | pH | Minimum acceptable operating pH. |
| `SAFETY_PH_MAX` | `8.5` | pH | Maximum acceptable operating pH. |
| `SAFETY_PH_CRITICAL_LOW` | `4.0` | pH | Critical low pH threshold; immediately trips F-004. |
| `SAFETY_PH_CRITICAL_HIGH` | `9.0` | pH | Critical high pH threshold; immediately trips F-003. |
| `SAFETY_PH_RATE_MAX_PER_MIN` | `1.0` | pH / min | Maximum allowable rate of change before tripping F-003 / F-004. |
| `SAFETY_PH_FROZEN_EPSILON` | `0.005` | pH | Maximum change considered "frozen". |
| `SAFETY_PH_FROZEN_SAMPLES` | `300` | Samples (150s) | Consecutive loops required to trigger frozen pH probe fault (F-009). |
| `SAFETY_PH_RESPONSE_TIMEOUT_MS` | `90000` | ms (1.5 min) | Maximum time allowed for pH to shift $\ge 0.03$ following a dose. |
| `SAFETY_PH_RESPONSE_MIN_DELTA` | `0.03` | pH | Minimum expected pH delta to confirm pump successfully delivered liquid. |
| `SAFETY_TDS_MIN` | `50.0` | ppm | Minimum operating TDS; lower suggests probe dry/unplugged. |
| `SAFETY_TDS_MAX` | `2500.0` | ppm | Maximum operating TDS; higher risks root osmotic shock. |
| `SAFETY_TDS_FROZEN_EPSILON` | `0.5` | ppm | Delta threshold for frozen TDS probe detection. |
| `SAFETY_TDS_FROZEN_SAMPLES` | `300` | Samples (150s) | Consecutive loops required to trigger frozen TDS fault (F-008). |
| `SAFETY_TDS_OOR_FAULT_MS` | `10000` | ms (10 s) | Duration TDS must remain out-of-bounds before F-008 trips. |
| `SAFETY_TDS_RESPONSE_TIMEOUT_MS`| `120000` | ms (2 min) | Maximum time allowed for TDS to shift $\ge 5.0$ ppm following nutrient dose. |
| `SAFETY_TDS_RESPONSE_MIN_DELTA` | `5.0` | ppm | Minimum expected TDS delta confirming nutrient delivery. |

---

## 3. Clearing Alarms & Bitmask Arithmetic

When a fault is raised, the firmware enters Safe Mode. Operators clear faults by publishing to `<zone_id>/safety/clear`:

### Option A: Complete Reset (`CLEAR` or `ALL`)
```c
// Clears all bits in s_safety.faults, deactivates Safe Mode, reopens gate
runtime_safety_clear_faults(0, &safe_mode_cleared);
```

### Option B: Selective Mask Clearing (`MASK:<hex>`)
If an operator resolves a TDS sensor problem but the reservoir water temperature is still cooling down, they can selectively clear only the TDS fault:
- Command payload: `MASK:0x0080`
- Internal bitwise operation:
  $$\text{New Faults} = \text{Current Faults} \ \& \ \sim(\text{0x0080})$$
- If the resulting mask is non-zero, Safe Mode remains active; if the resulting mask is zero, Safe Mode is automatically cleared.

