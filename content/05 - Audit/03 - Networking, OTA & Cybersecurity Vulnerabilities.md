---
title: Audit - Networking, OTA & Cybersecurity Vulnerabilities
tags:
  - audit
  - security
  - ota
  - cve
  - cybersecurity
created: 2026-10-28
---

# 🔓 Networking, OTA & Cybersecurity Vulnerability Audit

In modern commercial agriculture, automated nodes act as Internet of Things (IoT) endpoints controlling physical power, high-voltage lighting, and hazardous chemicals. The HydroponicsIDF firmware contains **critical cybersecurity vulnerabilities** that allow trivial remote code execution, unauthenticated network takeover, and physical denial-of-service.

---

## 1. VULN-03: Unsigned, Unencrypted HTTP OTA Firmware Updates (RCE)

### Severity: 🔴 CRITICAL (CVSS v3.1 Score: 9.8 / Critical)
**Source Location:** `main/ota_update.c` (Lines 43–155)

The firmware downloads Over-The-Air firmware updates using plain HTTP without TLS encryption, mutual authentication, or digital signature validation:

```c
// ota_update.c:43-49
esp_http_client_config_t config = {
    .url = url,                            // "http://<broker_ip>:8123/..."
    .timeout_ms = OTA_HTTP_TIMEOUT_MS,
    .buffer_size = OTA_HTTP_BUFFER_SIZE,
    .keep_alive_enable = true,             // NO cert_pem! NO SSL/TLS!
};
```

### The Attack Vector (Man-in-the-Middle Remote Code Execution):
Because the URL uses `http://` instead of `https://`:
1. Any device on the same local network (a compromised Wi-Fi camera, a rogue laptop, or an attacker performing ARP spoofing) can intercept the HTTP GET request for `lorong_node.bin`.
2. The attacker injects a malicious binary containing arbitrary shellcode or backdoored firmware.
3. The firmware calls `esp_ota_write()` and `esp_ota_end()`:
   ```c
   err = esp_ota_end(update_handle); // Checks only CRC32/SHA256 of the downloaded image!
   ```
   `esp_ota_end()` verifies only that the binary is internally consistent; it **does NOT verify cryptographic origin or signature**.
4. The ESP32 sets the boot partition to the attacker's binary and reboots.
5. **Consequence**: The attacker now has full persistent root execution on the ESP32, with the ability to turn on high-voltage contactors, dump hazardous acid continuously, or pivot to compromise the central farm server.

---

## 2. VULN-08: Insecure Soft-AP & Unauthenticated Web Captive Portal

### Severity: 🟡 MEDIUM (CVSS v3.1 Score: 7.5 / High)
**Source Locations:**
- `main/wifi_manager.c` (Lines 132–137)
- `main/web_portal.c` (Lines 41–53 & 757–764)

### Defect 1: Open, Unencrypted Wi-Fi Access Point
When entering setup mode, the Soft-AP is broadcast with open authentication:
```c
// wifi_manager.c:134-137
if (ap_password != NULL && strlen(ap_password) >= 8) {
    wifi_config.ap.authmode = WIFI_AUTH_WPA2_PSK;
} else {
    wifi_config.ap.authmode = WIFI_AUTH_OPEN;  // <--- OPEN AP BROADCAST!
}
```
In `main.c:63`, the AP is started via `wifi_manager_start_ap("ESP32S3-Updater", NULL)` — passing `NULL` as the password! Anyone standing within 100 meters of the greenhouse with a smartphone can associate with the network without a passphrase.

### Defect 2: Zero Web Server Authentication or CSRF Protection
1. The HTTP server runs on port 80 without HTTP Basic/Digest authentication, session tokens, or TLS.
2. The endpoint `POST /configure` accepts form submissions from any client IP.
3. There are no Cross-Site Request Forgery (CSRF) tokens.
4. An unauthorized user can submit bogus Wi-Fi credentials or change the `zone_id` from `zone_1` to `zone_99`, completely severing the node from central automation.

---

## 3. Plaintext, Unauthenticated MQTT Telemetry & Control

### Severity: 🟡 MEDIUM
**Source Location:** `main/mqtt_manager.c` (Lines 29–35 & `main/setup_config.c`)

The MQTT client configuration uses plain TCP over port `1883`:
- **No TLS Encryption**: Packets travel across the local network unencrypted. An attacker running Wireshark can observe all sensor readings, firmware version numbers, and actuator states in real-time.
- **No Authentication**: `mqtt_manager_init()` does not configure a username, password, or client SSL/TLS certificate.
- **Unauthorized Actuator Control**: Anyone on the local network can publish to `<zone_id>/PerPHDown/command` with payload `ON`, causing the pump to run without the central supervisory system's knowledge or consent.

---

## 4. VULN-09: Denial-of-Service via NVS Flash Sector Exhaustion

### Severity: 🟡 MEDIUM
**Source Locations:**
- `main/sensor_telemetry.c` (Lines 1239–1255)
- `main/sensor_calibration_nvs.c` (Lines 22–60)

### Flash Endurance Physics:
The onboard SPI NOR flash memory of the ESP32 has an endurance rating of approximately **100,000 write/erase cycles per physical sector**.

### The Vulnerability:
When a calibration message is received on `<zone_id>/calibration/<pH|TDS>/set`:
```c
// sensor_telemetry.c:1240-1246
for (int attempt = 0; attempt <= CALI_NVS_RETRY_COUNT; attempt++) {
    save_ret = nvs_save(&new_cali);
    if (save_ret == ESP_OK) break;
}
```
1. The incoming message is parsed and written **directly to NVS flash**.
2. **There is ZERO rate limiting!**
3. If an attacker publishes 10 calibration messages per second to the unauthenticated MQTT broker:
   - In 1 second: 10 flash write cycles.
   - In 1 hour: 36,000 flash write cycles.
   - In 3 hours: **108,000 flash write cycles**.
4. The physical NOR flash sector housing the `sensor_cali` namespace will permanently degrade, corrupting the NVS partition table and permanently bricking the microcontroller.

---

## 5. VULN-10: UDP Syslog Broadcast Flooding & Information Leakage

### Severity: 🟢 LOW
**Source Location:** `main/syslog.c` (Lines 28–45)

During boot, the syslog client binds to broadcast address `255.255.255.255` on UDP port `514`:
```c
syslog_init("255.255.255.255", 514);
```

### The Pitfalls:
1. **Network Degradation**: Every line logged by `ESP_LOGI`, `ESP_LOGW`, or `ESP_LOGE` generates a UDP broadcast frame that every device on the Wi-Fi subnet must receive and process in hardware, wasting network airtime.
2. **Cleartext Information Leakage**: Debug logs broadcast memory stack watermarks, task pointers, sensor calibration coefficients, and internal error codes to every machine on the LAN.
3. **Enterprise Incompatibility**: Enterprise Wi-Fi access points and managed industrial switches actively drop or rate-limit `255.255.255.255` broadcast packets by default, rendering the remote logger silent in professional facilities.

