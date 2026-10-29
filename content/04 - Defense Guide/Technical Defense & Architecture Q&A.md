---
title: Technical Defense & Architecture Q&A
tags:
  - defense
  - viva
  - engineering
  - exam
  - faq
created: 2026-10-28
---

# 🎓 Technical Defense, Viva & Engineering Architecture Q&A

This document compiles the exhaustive technical defense guide for the **HydroponicsIDF** firmware. It provides in-depth answers to rigorous questions regarding real-time operating systems (RTOS), deterministic safety, concurrent synchronization, electrical protections, sensor physics, and failure mode mitigations.

---

## 1. Firmware Architecture & FreeRTOS Fundamentals

### Q1: Why use FreeRTOS rather than a traditional super-loop (`while(1)`) on the ESP32?
**Answer:**
A super-loop architecture is fundamentally **non-deterministic** and prone to blocking. In automated hydroponics, tasks possess vastly different temporal requirements:
- **Safety and dosing** require deterministic sub-millisecond physical responses.
- **Sensors** require periodic sampling with moving-window mathematical filtering.
- **Network communications** (Wi-Fi and MQTT) introduce unpredictable latencies ranging from tens of milliseconds to tens of seconds during packet retransmissions.

In a super-loop, if an MQTT socket stalls waiting for an ACK, sensor reads are skipped and, critically, **actuator deactivation commands are delayed**, leading to severe chemical overdosing. FreeRTOS provides **preemptive, priority-based multitasking**: even if the communication task is completely blocked waiting on network sockets, the high-priority `SafetyTask` and `DosingTask` preempt execution to protect the physical environment.

### Q2: Explain the rationale behind the FreeRTOS Task Priority hierarchy.
**Answer:**
The system assigns priorities across 4 main tasks defined in `runtime_tasks.c`:
1. **`SafetyTask` (Priority 4 — Highest)**: Monitors task heartbeats, pump duration limits, and sensor freeze conditions. It holds the highest priority so that in any failure state, it can preempt any running code, immediately isolate the hardware, and enter Safe Mode.
2. **`DosingTask` (Priority 3)**: Consumes actuator commands from `s_dosing_queue`. It operates directly below safety so that physical relays and peristaltic pumps respond with exact hardware timing without jitter from sensor processing.
3. **`SensorTask` (Priority 2)**: Reads ADC, 1-Wire, and I2C buses. It sits below dosing because a jitter of a few milliseconds in sensor acquisition has zero impact on plant physiology.
4. **`CommTask` (Priority 1 — Lowest)**: Publishes MQTT telemetry. Network I/O is inherently high-latency and subject to external network delays. Placing it at Priority 1 ensures that Wi-Fi buffer congestion or broker latency never starves sensor sampling, pump pulsing, or safety watchdogs.

### Q3: How is thread safety achieved across tasks without introducing deadlocks or race conditions?
**Answer:**
The firmware employs four distinct synchronization tiers based on operation duration:
- **Lock-Free Atomics (`<stdatomic.h>`)**: Used for high-frequency flags (`safe_mode`, `s_override_active`) and task heartbeat counters. These guarantee multi-core cache coherency without mutex overhead.
- **Critical Sections (`portENTER_CRITICAL`)**: Used for microsecond-duration updates to internal shadow states (such as `s_channel_state_on[]` and `s_gate_open`). Interrupts are briefly disabled on the local core.
- **FreeRTOS Queues (`xQueueSend` / `xQueueReceive`)**: Decouples the asynchronous MQTT callback thread from physical actuator execution (`s_dosing_queue`).
- **Mutex Semaphores (`xSemaphoreCreateMutex`)**: Guards complex, multi-byte shared resources: `s_snapshot_mutex` for sensor snapshots, `s_i2c0_mutex` for shared I2C peripheral arbitration, and `s_subscriptions_lock` for the MQTT subscription table.

---

## 2. Safety Interlocks & Failure Mitigations

### Q4: If a task encounters a deadlock or hangs, how does the firmware prevent pumps from staying ON indefinitely?
**Answer:**
The firmware incorporates a **Two-Tier Watchdog Engine**:
1. **Tier 1 — Software Task Heartbeats**: Every task increments an atomic counter each cycle. Every 500 ms, `SafetyTask` inspects these counters. If any task stops incrementing for more than 15 seconds (`SAFETY_HEARTBEAT_TIMEOUT_MS`), `SafetyTask` immediately sets `SAFETY_FAULT_WDT`, closes the command gate, and de-energizes all outputs.
2. **Tier 2 — Hardware Task Watchdog Timer (`esp_task_wdt`)**: If `SafetyTask` itself deadlocks or the RTOS scheduler hangs, all tasks stop kicking the hardware TWDT. After 8 seconds (`SAFETY_WDT_TIMEOUT_SEC`), the ESP32 hardware triggers a system panic reset. Upon reboot, `runtime_tasks_record_boot_faults()` inspects `esp_reset_reason()`, recognizes the WDT reset, latches a boot fault, and boots straight into Safe Mode.

### Q5: What is the "Peristaltic Mutual Exclusion" / Chemical Hazard Interlock?
**Answer:**
Located in `dosing_task`, this rule permits **only one peristaltic pump to operate at any instant**. 
In hydroponics, pH Down ($H_3PO_4$ or $H_2SO_4$ acid) and pH Up ($KOH$ base) must **never be injected simultaneously**. Simultaneous injection inside the dosing manifold causes a violent, concentrated neutralization reaction producing heat, salts, and gas bubbles that can rupture tubing. Furthermore, simultaneous operation causes large DC current spikes that could brown out the microcontroller. If a request is received to turn ON pH Up while pH Down is running, the command is rejected and `SAFETY_FAULT_PH_INTERLOCK` is raised.

### Q6: How was Priority Inversion prevented during Safe Mode entry?
**Answer:**
In an earlier iteration, `safety_enter_safe_state()` called `lcd_status_show_emergency()`. That function acquired `s_lcd_lock`, a mutex held by the Priority 1 `lcd_scroll_task_fn` during 20x4 I2C write cycles.
Under I2C bus contention, `SafetyTask` (Priority 4) was forced to wait for the Priority 1 scroll task to finish transmitting bytes. This priority inversion delayed `SafetyTask` from kicking its TWDT, resulting in an unintended panic reset.
**The Fix**: `lcd_status_show_emergency()` was removed from the critical safe-state function. Outputs are now forced LOW in microseconds via direct register writes, and the LCD is updated asynchronously via `safety_publish_faults()`.

### Q7: Why does Safe Mode drive GPIOs LOW directly rather than calling `actuator_control_apply_state()`?
**Answer:**
`actuator_control_apply_state()` performs two actions: it writes to the GPIO register, and then it synchronously calls `mqtt_manager_publish()` to announce the state change over the network.
If `SafetyTask` called `apply_state` for all 10 channels during an emergency, it would trigger 10 synchronous MQTT network transmissions. If the Wi-Fi network was congested or disconnected, `SafetyTask` would block inside LwIP TCP stack routines for hundreds of milliseconds, risking watchdog timeouts. 
Instead, `actuator_control_force_off()` toggles the GPIO pin registers directly without network I/O. The MQTT broker is synchronized asynchronously by `comm_task` on its next 3-second cycle.

---

## 3. Sensors, Physical Chemistry & Calibration

### Q8: Why use an external ADS1115 ADC over the ESP32-S3's internal ADC?
**Answer:**
The internal SAR ADCs on the ESP32 family suffer from severe non-linearities, significant input noise, and dead-zones near 0V and 3.3V (even with factory $V_{\text{ref}}$ calibration). Analog pH and TDS probes output high-impedance, microvolt-level signals where noise translates directly to massive measurement errors (e.g., a 10 mV drift in pH probes shifts the reading by ~0.17 pH).
The **Texas Instruments ADS1115** provides:
- 16-bit Delta-Sigma resolution (65,536 quantization steps).
- Integrated low-drift voltage reference and Programmable Gain Amplifier (PGA).
- Differential and single-ended inputs with internal digital filtering over I2C.

### Q9: Why is real-time temperature compensation mandatory for TDS readings?
**Answer:**
TDS sensors measure electrical conductivity (EC). The mobility of ions in aqueous solutions increases with temperature at approximately **2.0% per degree Celsius**.
If a reservoir cools from 25°C to 15°C overnight, an uncompensated TDS probe will report a 20% drop in ppm, tricking an automation system into pumping toxic amounts of fertilizer. The firmware applies real-time temperature normalization to standard 25°C:
$$\text{TDS}_{\text{compensated}} = \frac{\text{Raw}_{\text{ADC}}}{1.0 + 0.02 \times (T_{\text{water}} - 25.0)}$$

### Q10: What is the purpose of the "Frozen Sensor Fault" (`SAFETY_FAULT_PH_SENSOR`)?
**Answer:**
In a biological reservoir with moving water, sensor readings naturally fluctuate by tiny fractions of a millivolt due to thermal currents and chemical ion drift. If an ADC or probe cable disconnects or freezes, the reading often locks onto an exact constant floating value.
If the reading exhibits zero change ($|\Delta| < 0.005$) for 300 consecutive samples (150 seconds), the firmware detects that the sensor has stopped responding and latches a frozen sensor fault, preventing blind closed-loop dosing.

---

## 4. Actuator Control & Electrical Protections

### Q11: Why does the firmware utilize `PULSE` commands instead of relying on the server to send `ON` and then `OFF`?
**Answer:**
Relying on separate `ON` and `OFF` network commands is a critical vulnerability. If the cloud server issues an `ON` command and the Wi-Fi router immediately drops connection, the `OFF` command will never arrive, causing the dosing pump to dump an entire bottle of concentrated acid into the tank.
With `PULSE:<ms>`, the server specifies the dose duration upfront. The microcontroller's local hardware timer executes the pulse and guarantees automatic shutdown, making physical safety completely independent of network health.

### Q12: How was mechanical "Relay Chatter" during boot eliminated?
**Answer:**
During early microcontroller power-on, GPIO pins float in high-impedance mode before firmware configuration executes. Floating pins can float to logic HIGH, causing relay coils to energize and chatter.
The firmware executes `actuator_control_early_gpio_init()` as the **very first line of `app_main()`**, clamping all actuator GPIOs to active LOW outputs before NVS, Wi-Fi, or FreeRTOS tasks initialize.

### Q13: Why is the Circulation Pump (`PIN_CIRCULATION_PUMP`, GPIO 13) protected from all shutdown commands?
**Answer:**
In hydroponics, stagnant water causes dissolved oxygen levels to deplete within 30 to 60 minutes, leading to root asphyxiation and *Pythium* (root rot) outbreak. 
Even during a chemical emergency or sensor fault, the water must continue circulating. The firmware hard-codes GPIO 13 to HIGH at boot and explicitly intercepts and rejects any attempts to turn it OFF in `apply_state`, `force_off`, and `deinit`.

---

## 5. Networking, Outages & Fault Recovery

### Q14: How does the firmware handle power outages where the router boots slower than the ESP32?
**Answer:**
The firmware implements the **Persistent `wifi_tries` Reboot Loop**:
1. When power returns, the ESP32 boots in 1.2s, while the router may take 90s.
2. If Wi-Fi or MQTT connection fails within the 2-minute timeout, the device increments `wifi_tries` in NVS and performs an `esp_restart()`.
3. It stubbornly reboots and retries up to **5 consecutive times** (~10 minutes total window).
4. As soon as both Wi-Fi and MQTT connect, `wifi_tries` resets to 0.
5. Only if all 5 attempts fail does it abandon autostart and drop to manual setup mode.

### Q15: Why does the firmware use exact topic matching instead of MQTT wildcards?
**Answer:**
MQTT wildcards (`#` and `+`) introduce ambiguity. In a multi-zone installation, an errant wildcard subscription could intercept commands intended for another zone or trigger multi-channel handlers unintentionally. Exact string matching (`strncmp` with exact length verification) guarantees that a callback executes only when the published topic matches the registered string character-for-character.

