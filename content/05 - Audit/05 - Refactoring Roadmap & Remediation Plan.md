---
title: Audit - Refactoring Roadmap & Remediation Plan
tags:
  - audit
  - refactoring
  - remediation
  - patches
  - c-code
created: 2026-10-28
---

# 🛠️ Refactoring Roadmap & Production Remediation Plan

This document outlines concrete, production-ready C code patches and architectural refactoring strategies to eliminate all critical vulnerabilities identified in the audit.

---

## 1. Remediation for VULN-01: Non-Blocking Asynchronous Pulse Engine

### Current Defect:
`actuator_control_apply_pulse()` calls `vTaskDelay(pulse_ms)` directly inside `dosing_task`, blocking execution for up to 60 seconds, starving the hardware TWDT, and blinding the task to emergency stop commands.

### Production Solution:
Replace blocking delays with a tick-based pulse expiration scheduler inside `dosing_task`, or an asynchronous FreeRTOS software timer per channel.

```c
/* =========================================================================
 * Recommended Patch: Non-Blocking Pulse Execution in dosing_task
 * ========================================================================= */

typedef struct {
    TickType_t stop_tick;
    bool active;
} channel_pulse_state_t;

static channel_pulse_state_t s_active_pulses[ACTUATOR_CHANNEL_COUNT] = {0};

/* In dosing_task main event loop: */
void dosing_task_loop(void)
{
    dosing_command_t cmd;
    TickType_t now = xTaskGetTickCount();

    /* 1. Check and expire active pulses (Executed every 50 ms loop tick) */
    for (int i = 0; i < ACTUATOR_CHANNEL_COUNT; i++) {
        if (s_active_pulses[i].active && (int32_t)(now - s_active_pulses[i].stop_tick) >= 0) {
            actuator_control_apply_state((actuator_channel_t)i, false);
            runtime_safety_update_channel_state((actuator_channel_t)i, false);
            s_active_pulses[i].active = false;
            ESP_LOGI(TAG, "Pulse completed for channel %d", i);
        }
    }

    /* 2. Process incoming commands with a short 50 ms timeout */
    if (xQueueReceive(s_dosing_queue, &cmd, pdMS_TO_TICKS(50)) == pdTRUE) {
        if (cmd.action == ACTUATOR_ACTION_PULSE) {
            /* Start pulse asynchronously without sleeping! */
            actuator_control_apply_state(cmd.channel, true);
            runtime_safety_update_channel_state(cmd.channel, true);
            s_active_pulses[cmd.channel].stop_tick = now + pdMS_TO_TICKS(cmd.pulse_ms);
            s_active_pulses[cmd.channel].active = true;
        } else {
            /* Standard ON/OFF handling */
            actuator_control_apply_state(cmd.channel, cmd.action == ACTUATOR_ACTION_ON);
            runtime_safety_update_channel_state(cmd.channel, cmd.action == ACTUATOR_ACTION_ON);
            s_active_pulses[cmd.channel].active = false; /* Cancel any running pulse */
        }
    }

    /* 3. Feed Watchdogs every 50 ms regardless of whether pulses are running! */
    runtime_safety_heartbeat_dosing();
    runtime_safety_wdt_kick();
}
```

---

## 2. Remediation for VULN-02: Safe Multi-Core Queue Draining

### Current Defect:
`enqueue_command()` calls `xQueueReset(s_dosing_queue)` when full, triggering an SMP race condition against `dosing_task` blocked on `xQueueReceive()`.

### Production Solution:
Replace `xQueueReset` with a safe, non-blocking drain loop:

```c
/* =========================================================================
 * Recommended Patch in runtime_tasks.c: enqueue_command()
 * ========================================================================= */
static bool enqueue_command(const dosing_command_t *cmd, const char *source)
{
    if (cmd == NULL || s_dosing_queue == NULL) return false;
    if (!runtime_safety_is_gate_open(xPortInIsrContext())) return false;

    BaseType_t sent = xQueueSend(s_dosing_queue, cmd, 0);

    if (sent != pdTRUE && !xPortInIsrContext()) {
        /* Queue full — safely drain stale messages without xQueueReset() */
        dosing_command_t dummy;
        UBaseType_t dropped = 0;
        while (xQueueReceive(s_dosing_queue, &dummy, 0) == pdTRUE) {
            dropped++;
        }
        ESP_LOGW(TAG, "Dosing queue full: safely drained %u stale commands", (unsigned)dropped);
        
        /* Enqueue the latest command */
        sent = xQueueSend(s_dosing_queue, cmd, 0);
    }
    return (sent == pdTRUE);
}
```

---

## 3. Remediation for VULN-06: Robust RFC-Compliant JSON Parsing with `cJSON`

### Current Defect:
`json_get_float()` uses brittle `strstr()` pattern searching that breaks on whitespace variations and nested keys.

### Production Solution:
Use ESP-IDF's native `cJSON` library:

```c
/* =========================================================================
 * Recommended Patch in sensor_telemetry.c: validate_cali_payload()
 * ========================================================================= */
#include "cJSON.h"

static bool validate_cali_payload_cjson(const char *payload, int payload_len,
                                        const cali_plausible_range_t *range,
                                        calibration_t *out, char *reason_out)
{
    cJSON *root = cJSON_ParseWithLength(payload, payload_len);
    if (root == NULL) {
        snprintf(reason_out, 128, "Invalid JSON formatting");
        return false;
    }

    cJSON *act1 = cJSON_GetObjectItemCaseSensitive(root, "actual_1");
    cJSON *raw1 = cJSON_GetObjectItemCaseSensitive(root, "raw_1");
    cJSON *act2 = cJSON_GetObjectItemCaseSensitive(root, "actual_2");
    cJSON *raw2 = cJSON_GetObjectItemCaseSensitive(root, "raw_2");

    if (!cJSON_IsNumber(act1) || !cJSON_IsNumber(raw1) ||
        !cJSON_IsNumber(act2) || !cJSON_IsNumber(raw2)) {
        snprintf(reason_out, 128, "Missing or non-numeric calibration fields");
        cJSON_Delete(root);
        return false;
    }

    float actual_1 = (float)act1->valuedouble;
    float raw_1    = (float)raw1->valuedouble;
    float actual_2 = (float)act2->valuedouble;
    float raw_2    = (float)raw2->valuedouble;

    cJSON_Delete(root);

    if (raw_2 == raw_1) {
        snprintf(reason_out, 128, "raw_1 and raw_2 cannot be equal (division by zero)");
        return false;
    }

    float slope  = (actual_2 - actual_1) / (raw_2 - raw_1);
    float offset = actual_1 - (slope * raw_1);

    if (slope < range->slope_min || slope > range->slope_max ||
        offset < range->offset_min || offset > range->offset_max) {
        snprintf(reason_out, 128, "Coefficients out of plausible bounds");
        return false;
    }

    out->slope = slope;
    out->offset = offset;
    out->valid = true;
    out->updated_at = (uint32_t)(esp_timer_get_time() / 1000000ULL);
    return true;
}
```

---

## 4. Remediation for VULN-07: Pin Configuration Unification

### Current Defect:
`main/pin_config.h` defined `PIN_SETUP_BUTTON 6`, but `setup_button.c` hardcoded `46`.

### Patch:
1. In `main/pin_config.h`:
   ```c
   #define PIN_SETUP_BUTTON 46  // Match physical schematic!
   ```
2. In `main/setup_button.c`:
   ```c
   #include "pin_config.h"
   // Remove local #define SETUP_BUTTON_GPIO 46
   // Use PIN_SETUP_BUTTON directly across all gpio_config calls.
   ```
3. In `main/main.c`:
   ```c
   ESP_LOGI(TAG, "Hold GPIO%d for 3 s at any time to reconfigure.", PIN_SETUP_BUTTON);
   setup_button_wait_for_hold(3000);
   ```

---

## 5. Remediation for VULN-09: Eliminating SPI Flash Wear-Out

### Flash Protection Patches:
1. **Move `wifi_tries` to RTC Fast Memory**:
   Instead of writing to NVS on every reboot attempt:
   ```c
   RTC_DATA_ATTR static int32_t s_rtc_wifi_tries = 0;
   ```
   RTC memory persists across all warm and panic resets without writing a single byte to SPI NOR flash! NVS is touched only if credentials are wiped.
2. **Rate-Limit Calibration Flash Writes**:
   Enforce a mandatory minimum interval between NVS calibration updates:
   ```c
   static uint32_t s_last_nvs_cali_write_sec = 0;
   uint32_t now_sec = (uint32_t)(esp_timer_get_time() / 1000000ULL);

   if (now_sec - s_last_nvs_cali_write_sec < 60) {
       ESP_LOGW(TAG, "Calibration write rejected: rate limit (max 1 write/minute)");
       return;
   }
   s_last_nvs_cali_write_sec = now_sec;
   ```

---

## 6. Monolithic Decomposition Plan (`sensor_telemetry.c`)

Refactor the 1,870-line God-file into modular drivers within a new `components/` or `main/drivers/` tree:

```
main/
├── drivers/
│   ├── ds18b20_rmt.c/.h       # 1-Wire bus discovery, conversion trigger & read
│   ├── ads1115_i2c.c/.h       # ADS1115 config, median sampling, I2C bus lock
│   ├── sht31_i2c.c/.h         # SHT31 driver & Tetens VPD math
│   └── water_level.c/.h       # Float switch GPIO debounce state machine
├── services/
│   ├── calibration_mgr.c/.h   # cJSON parsing, NVS persistence, rate limiting
│   └── telemetry_cache.c/.h   # Rolling average window, snapshot mirror mutex
└── sensor_telemetry.c         # Pure orchestration (under 250 lines!)
```

