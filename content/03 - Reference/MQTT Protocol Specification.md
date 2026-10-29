---
title: MQTT Protocol Specification
tags:
  - mqtt
  - api
  - telemetry
  - reference
created: 2026-10-28
---

# 🌐 MQTT Protocol & Telemetry Specification

All MQTT topics utilized by the HydroponicsIDF firmware are prefixed with `<zone_id>` (e.g. `zone_1`, `zone_north`), configured dynamically in NVS during provisioning.

Topic matching in the firmware is **strictly exact**; wildcards (`#` and `+`) are not supported to prevent accidental command dispatch.

---

## 1. Master Topic Reference Table

| Category | Full Topic String | Direction | Payload Type | QoS | Retain | Interval / Trigger | Description |
|---|---|:---:|:---:|:---:|:---:|:---:|---|
| **Handshake** | `SetUp` | **Publish** | JSON | 1 | 0 | Boot / Provisioning | Notifies central broker of zone name & ID |
| **Handshake** | `<zone_id>/success` | **Subscribe** | Plain Text | 1 | 0 | Upon `SetUp` receipt | Broker ACK; signals cloud readiness |
| **Version** | `<zone_id>/version` | **Publish** | Plain Text | 1 | **1** | Boot + Every 30 s | Active firmware semantic version (e.g. `"v1.2.3"`) |
| **Actuator Cmd** | `<zone_id>/<channel>/command` | **Subscribe** | Plain Text | 1 | 0 | Asynchronous | Commands channel: `ON`, `OFF`, `PULSE:<ms>` |
| **Actuator Cmd** | `<zone_id>/command` | **Subscribe** | Plain Text | 1 | 0 | Asynchronous | Multi-channel zone command: `<name> ON\|OFF\|PULSE <ms>` |
| **Actuator State** | `<zone_id>/<channel>/status` | **Publish** | Plain Text | 1 | 0 | State Change + Connect | Current channel state: `"ON"` or `"OFF"` |
| **Telemetry** | `<zone_id>/sensor/WaterLevel/state` | **Publish** | Integer | 0 | 0 | Every 3,000 ms | Reservoir level: `1` (Full) or `0` (Low) |
| **Telemetry** | `<zone_id>/sensor/WaterTemp/state` | **Publish** | Float | 0 | 0 | Every 3,000 ms | Water temperature in Celsius (`"%.2f"`) |
| **Telemetry** | `<zone_id>/sensor/RoomTemperature/state`| **Publish** | Float | 0 | 0 | Every 3,000 ms | Ambient temperature in Celsius (`"%.2f"`) |
| **Telemetry** | `<zone_id>/sensor/Humidity/state` | **Publish** | Float | 0 | 0 | Every 3,000 ms | Ambient relative humidity in % (`"%.2f"`) |
| **Telemetry** | `<zone_id>/VPD` | **Publish** | Float | 0 | 0 | Every 3,000 ms | Vapor Pressure Deficit in kPa (`"%.2f"`) |
| **Telemetry** | `<zone_id>/sensor/pH/raw` | **Publish** | Integer | 0 | 0 | Every 3,000 ms | Raw ADC counts / UART payload counts |
| **Telemetry** | `<zone_id>/sensor/pH/state` | **Publish** | Float | 0 | 0 | Every 3,000 ms | Calibrated pH float [0.0–14.0] (Suppressed if invalid) |
| **Telemetry** | `<zone_id>/sensor/pH/valid` | **Publish** | Boolean | 1 | **1** | Every 3,000 ms | `"true"` or `"false"` validation status |
| **Telemetry** | `<zone_id>/sensor/TDS/raw` | **Publish** | Integer | 0 | 0 | Every 3,000 ms | Raw 16-bit ADS1115 counts (AIN1) |
| **Telemetry** | `<zone_id>/sensor/TDS/state` | **Publish** | Float | 0 | 0 | Every 3,000 ms | Temp-compensated TDS float in ppm (Suppressed if invalid) |
| **Telemetry** | `<zone_id>/sensor/TDS/valid` | **Publish** | Boolean | 1 | **1** | Every 3,000 ms | `"true"` or `"false"` validation status |
| **Calibration** | `<zone_id>/calibration/pH/set` | **Subscribe** | JSON | 1 | 0 | Asynchronous | Sets 2-point reference calibration for pH |
| **Calibration** | `<zone_id>/calibration/TDS/set` | **Subscribe** | JSON | 1 | 0 | Asynchronous | Sets 2-point reference calibration for TDS |
| **Calibration** | `<zone_id>/calibration/pH/state` | **Publish** | JSON | 1 | **1** | Calibration Update | Current active slope, offset, and timestamp |
| **Calibration** | `<zone_id>/calibration/TDS/state` | **Publish** | JSON | 1 | **1** | Calibration Update | Current active slope, offset, and timestamp |
| **Safety Fault** | `<zone_id>/safety/fault` | **Publish** | JSON | 1 | 0 (Boot: 1) | On Fault Event | Active fault bitmask, code, count, and reason |
| **Safety Clear** | `<zone_id>/safety/clear` | **Subscribe** | Plain Text | 1 | 0 | Operator Action | Clear command: `CLEAR`, `ALL`, `MASK:<hex>` |
| **Emergency** | `<zone_id>/emergency` | **Subscribe** | Plain Text | 1 | 0 | Asynchronous | Payload `"1"` immediately triggers Safe Mode |
| **Override** | `<zone_id>/override` | **Subscribe** | Plain Text | 1 | 0 | Asynchronous | Payload `"ON"` or `"OFF"` toggles limit suppression |
| **OTA Control** | `<zone_id>/ota/latest_version` | **Subscribe** | Plain Text | 1 | 0 | Asynchronous | Target OTA version string for logging |
| **OTA Control** | `<zone_id>/ota/trigger` | **Subscribe** | Plain Text | 1 | 0 | Asynchronous | Payload `"1"` triggers background HTTP OTA download |

---

## 2. JSON Payload Schemas

### A. Handshake Announcement (`SetUp`):
```json
{
  "zone_id": "zone_1",
  "name": "Lorong NFT Utara"
}
```

### B. Two-Point Calibration Command (`<zone_id>/calibration/<pH|TDS>/set`):
```json
{
  "actual_1": 4.01,
  "raw_1": 15420.0,
  "actual_2": 6.86,
  "raw_2": 11840.0,
  "updated_at": 1729864200
}
```

### C. Active Calibration State (`<zone_id>/calibration/<pH|TDS>/state`):
```json
{
  "mode": "linear",
  "slope": -0.000796,
  "offset": 16.284,
  "valid": true,
  "updated_at": 1729864200
}
```

### D. Safety Fault Alarm (`<zone_id>/safety/fault`):
```json
{
  "mask": 256,
  "code": "F-009",
  "count": 1,
  "text": "pH Sensor",
  "reason": "pH out of range"
}
```

---

## 3. Actuator Topic Structure

Commands can be targeted to specific channels or broadcast to the unified zone topic:

### Per-Channel Command Topics:
- `<zone_id>/valve/command`
- `<zone_id>/PerNutA/command`
- `<zone_id>/PerNutB/command`
- `<zone_id>/PerPHUp/command`
- `<zone_id>/PerPHDown/command`
- `<zone_id>/relay1/command`
- `<zone_id>/relay2/command`
- `<zone_id>/relay3/command`
- `<zone_id>/relay4/command`
- `<zone_id>/CirculationPump/command` *(Commands to CirculationPump are accepted by the parser but discarded by hardware protection)*

### Accepted Command Payloads:
- `ON` — Energizes channel indefinitely (subject to safety dose duration watchdogs).
- `OFF` — Shuts channel OFF immediately.
- `PULSE:<ms>` — Energizes channel for `<ms>` milliseconds (e.g. `PULSE:3500`), then deactivates automatically.

### Unified Multi-Channel Command Topic (`<zone_id>/command`):
Allows controlling any channel in a single string payload:
- `"PerNutA PULSE 5000"`
- `"valve ON"`
- `"relay1 OFF"`

