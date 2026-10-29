---
title: Pin & Hardware Mapping Matrix
tags:
  - hardware
  - pinout
  - schematics
  - reference
created: 2026-10-28
---

# 📌 Pin & Hardware Mapping Matrix

This reference document defines the complete hardware pin allocation, peripheral bus configurations, electrical levels, and compile-time hardware switches for the **ESP32-S3** microcontroller in the HydroponicsIDF firmware (`pin_config.h`).

---

## 1. Compile-Time Hardware Feature Flags

Features can be enabled or disabled at compile time in `main/pin_config.h`:

| Macro Switch | Default | Description | Hardware Dependency |
|---|:---:|---|---|
| `ENABLE_LCD` | `1` | Enables 20x4 I2C LCD status screen & marquee scrolling | PCF8574 backpack on I2C0 |
| `ENABLE_TDS_SENSOR` | `1` | Enables ADS1115 ADC sampling on AIN1 for TDS probe | ADS1115 on I2C0 (0x48) |
| `ENABLE_WATER_TEMP` | `1` | Enables DS18B20 1-Wire temperature acquisition | DS18B20 on GPIO 8 via RMT |
| `ENABLE_WATER_LEVEL` | `1` | Enables digital water level float switch monitoring | Mechanical float on GPIO 5 |
| `ENABLE_ACTUATORS` | `1` | Enables GPIO output drivers for pumps, valves, relays | 10 GPIOs on output registers |
| `ENABLE_INDICATOR_LEDS` | `1` | Enables connection and fault status LEDs | GPIO 1 and GPIO 2 |
| `ENABLE_SHT31` | `1` | Enables SHT31 room temp, humidity, and VPD acquisition | SHT31 on I2C0 (0x44) |
| `ENABLE_SETUP_BUTTON` | `0` | Controls optional boot-time button requirement | GPIO 6 |
| `PH_SOURCE_USE_SERIAL` | `1` | Selects pH acquisition mode: `0`=Disabled, `1`=UART1, `2`=ADS1115 | UART1 (Pins 17/18) or ADS1115 AIN1 |

---

## 2. Complete Microcontroller Pin Allocation

```
                        ESP32-S3 Pinout Map (WROOM-1 / DevKit)
                                  ┌──────────────┐
                           3V3 ───┤ 1          44├── GND
             (Status Conn) IO1 ───┤ 2          43├── TX (UART0 Console)
            (Status Fault) IO2 ───┤ 3          42├── RX (UART0 Console)
                           IO3 ───┤ 4          41├── IO41 (Nutrient A Pump)
             (Water Level) IO5 ───┤ 5          40├── IO40 (Nutrient B Pump)
             (Setup Button)IO6 ───┤ 6          39├── IO39 (pH Up Pump)
                           IO7 ───┤ 7          38├── IO38 (pH Down Pump)
             (DS18B20 1-W) IO8 ───┤ 8          37├── IO47 (Relay 1 - Growlight)
          (pH Serial LED) IO10 ───┤ 9          36├── IO46 (Setup Button / Mode)
          (Solenoid Valve)IO12 ───┤ 10         35├── IO45 (Relay 4 - Fan)
       (Circulation Pump) IO13 ───┤ 11         34├── IO21 (Relay 3 - Bubbler)
        (Relay 2 - Heater)IO14 ───┤ 12         33├── IO14 (Relay 2)
              (I2C0 SCL)  IO15 ───┤ 13         32├── IO18 (pH Serial RX)
              (I2C0 SDA)  IO16 ───┤ 14         31├── IO17 (pH Serial TX)
        (Reserve Buzzer)  IO42 ───┤ 15         30├── IO48 (RGB WS2812)
                                  └──────────────┘
```

### Detailed Pin Assignment Table:

| GPIO | Function in Firmware | Direction | Active Level | Default Boot State | Hardware Peripheral / Circuit Notes |
|:---:|---|:---:|:---:|:---:|---|
| **GPIO 1** | Connection LED | Output | Active HIGH | `LOW` | Blue LED via 330Ω resistor. Indicates MQTT connectivity. |
| **GPIO 2** | Fault LED | Output | Active HIGH | `LOW` | Red LED via 330Ω resistor. Indicates Safe Mode / WDT alarms. |
| **GPIO 5** | Water Level Sensor | Input | Active LOW | Pulled UP | Digital float switch to GND. 5-second software debounce filter. |
| **GPIO 6** | Hardware Setup Button (Alt) | Input | Active LOW | Pulled UP | Optional tactile push button to GND. |
| **GPIO 8** | Water Temperature | Bidirectional | Tristate | Open-Drain | DS18B20 1-Wire data line. Requires external **4.7 kΩ pull-up to 3.3V**. |
| **GPIO 10** | pH Serial Activity LED | Output | Active HIGH | `LOW` | Diagnostic LED toggled on UART packet reception. |
| **GPIO 12** | Freshwater Solenoid Valve | Output | Active HIGH | `LOW` | Controls 12V solenoid via N-channel MOSFET / Optocoupler. |
| **GPIO 13** | **Circulation Water Pump** | Output | **Active HIGH** | **`HIGH`** | **Protected life-support line. Driven HIGH at boot and never forced LOW.** |
| **GPIO 14** | AC Relay 2 | Output | Active HIGH | `LOW` | 10A/250VAC Opto-isolated Relay module (Water Chiller/Heater). |
| **GPIO 15** | I2C0 Serial Clock (`SCL`) | Output | Open-Drain | Pulled UP | Shared clock for ADS1115 (0x48), LCD (0x27), SHT31 (0x44). 100 kHz. |
| **GPIO 16** | I2C0 Serial Data (`SDA`) | Bidirectional | Open-Drain | Pulled UP | Shared data line. Requires external 4.7 kΩ pull-ups to 3.3V. |
| **GPIO 17** | pH Serial UART TX | Output | Logic High | `HIGH` | Transmit line to Atlas Scientific EZO-pH or DFRobot serial module. |
| **GPIO 18** | pH Serial UART RX | Input | Floating | N/A | Receive line from serial pH module (115,200 baud, 8N1). |
| **GPIO 21** | AC Relay 3 | Output | Active HIGH | `LOW` | Air oxygenation bubbler pump relay. |
| **GPIO 38** | Peristaltic Pump: pH Down | Output | Active HIGH | `LOW` | 12V DC Peristaltic pump driver (Acid dosing). |
| **GPIO 39** | Peristaltic Pump: pH Up | Output | Active HIGH | `LOW` | 12V DC Peristaltic pump driver (Base dosing). |
| **GPIO 40** | Peristaltic Pump: Nutrient B | Output | Active HIGH | `LOW` | 12V DC Peristaltic pump driver (Macro/Micro elements). |
| **GPIO 41** | Peristaltic Pump: Nutrient A | Output | Active HIGH | `LOW` | 12V DC Peristaltic pump driver (NPK solution). |
| **GPIO 42** | Reserve Binary / Piezo Buzzer | Output | Active HIGH | `LOW` | 5V Piezo buzzer for boot/reset audible beeps. |
| **GPIO 45** | AC Relay 4 | Output | Active HIGH | `LOW` | Exhaust ventilation fan relay. |
| **GPIO 46** | Setup Button / AP Trigger | Input | Active LOW | Pulled UP | Physical button. Held for 3 seconds to launch `ESP32S3-Updater` Soft-AP. |
| **GPIO 47** | AC Relay 1 | Output | Active HIGH | `LOW` | Main high-power LED grow light contactor. |

---

## 3. Bus Architectures & Shared Interconnects

### A. Shared I2C0 Bus Topology
The firmware operates a single master I2C interface on `I2C_NUM_0`:
- **Clock**: `PIN_ADS1115_I2C_SCL` = GPIO 15
- **Data**: `PIN_ADS1115_I2C_SDA` = GPIO 16
- **Frequency**: 100 kHz standard mode (`PIN_LCD_I2C_FREQ_HZ`)
- **Arbitration**: Guarded by `s_i2c0_mutex` (`i2c_bus.c`) with a 500 ms maximum timeout.

```
ESP32-S3 (Master) ──┬── ADS1115 ADC (0x48) ──── TDS Probe (AIN1)
 (GPIO 15 / 16)     ├── PCF8574 Backpack (0x27 / 0x3F) ──── 20x4 LCD
                    └── GY-SHT31 Sensor (0x44) ──── Ambient Temp / RH
```

### B. 1-Wire Bus via ESP32-S3 RMT Peripheral
- **Signal**: GPIO 8
- **Hardware Layer**: Driven using the ESP-IDF `onewire_bus` driver via the **Remote Control (RMT)** hardware peripheral. This achieves microsecond-precise 1-Wire pulse timing without disabling FreeRTOS interrupts.
- **Pull-up Requirement**: An external $4.7\text{ k}\Omega$ resistor to 3.3V is strictly required. Internal pull-ups are too weak (~45 kΩ) and cause bus capacitive rise-time errors.

