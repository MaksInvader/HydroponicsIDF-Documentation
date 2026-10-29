---
title: Schematic & Circuit Design Guide
tags:
  - schematic
  - circuit-design
  - mosfet
  - relays
  - analog-afe
created: 2026-10-28
---

# 📐 Schematic & Circuit Design Engineering Guide

This guide provides tested, industrial-grade schematic designs, component selection tables, and circuit topologies required to manufacture a robust, noise-immune PCB for the HydroponicsIDF firmware.

---

## 1. Power Supply Conditioning & Multi-Rail Regulation

Agricultural AC/DC power adapters frequently generate electrical transients, high-frequency switching hash, and voltage spikes during motor cycling.

### A. 12V DC Input Protection Circuit
```
+12V_IN ───► [ Fuse 5A ] ───► [ P-MOSFET Reverse Protection ] ───┬───► +12V_RAW (To Pumps & Relays)
                                  (AO3401A or FDN340P)           │
                                                                 ├─── [ TVS Diode: SMAJ15A ]
                                                                 ├─── [ Bulk Cap: 470uF 25V Low-ESR ]
                                                                 └─── [ Ceramic: 100nF 50V X7R ]
GND_IN ──────────────────────────────────────────────────────────┴───► GND_POWER
```
- **Reverse Polarity Protection**: A P-channel MOSFET (AO3401A, $R_{DS(on)} < 45\text{ m}\Omega$) provides zero-voltage-drop reverse connection protection, avoiding the $0.5\text{V}$ drop of Schottky diodes.
- **TVS Spike Clamping**: The **SMAJ15A** transient voltage suppressor clamps inductive flyback spikes and lightning-induced line surges to $< 24\text{V}$, well within component ratings.

### B. Synchronous Buck Converter (12V $\to$ 5V @ 3A)
Utilizes the **Texas Instruments TPS54302** (or Monolithic Power Systems MP1584EN):
- **Input Voltage**: $4.5\text{V}$ to $28\text{V}$
- **Switching Frequency**: $400\text{ kHz}$ fixed
- **Inductor**: $10\ \mu\text{H}$ shielded power inductor (Würth 74437346100 or Coilcraft MSS1038, rated $I_{\text{sat}} \ge 4.5\text{A}$)
- **Feedback Divider**: $R_1 = 100\text{ k}\Omega\ (1\%)$, $R_2 = 13.3\text{ k}\Omega\ (1\%)$ setting $V_{\text{out}} = 5.08\text{V}$ to compensate for trace drops.

### C. Clean 3.3V Logic LDO Regulator
Utilizes the **Texas Instruments TLV75733P** (or Diodes Inc AP2112K-3.3):
- **Low Dropout**: $200\text{ mV}$ at $1\text{A}$.
- **PSRR**: $65\text{ dB}$ at $1\text{ kHz}$, filtering residual buck converter switching ripple away from the ESP32-S3 PLL and ADC reference.

---

## 2. Actuator Driver Circuits & Hardware Interlocks

### A. Inductive MOSFET Low-Side Driver (Pumps & Solenoid)
Applies to channels: `valve` (GPIO 12), `PerNutA` (GPIO 41), `PerNutB` (GPIO 40), `PerPHUp` (GPIO 39), `PerPHDown` (GPIO 38), and `CirculationPump` (GPIO 13).

```
+12V_RAW ──────────────────────────────┬────────────────────────┐
                                       │                        │
                                   [ Motor / Solenoid ]     [ Flyback Diode ]
                                   [ Inductive Load   ]     [ SS14 / 1N5819 ]
                                       │                        │
                                       ├────────────────────────┘
                                     Drain
                                       │
     100R Resistor                  ┌──┴──┐
GPIO ────[ 100Ω ]───────────────────┤ Gate│ N-MOSFET (AO3400A / IRLML6344TRPBF)
                       │            └──┬──┘ (Vds=30V, Id=5.7A, Rds=28mR)
                   [ 10kΩ ]          Source
                   [Pull-Down]         │
                       │               │
GND_POWER ─────────────┴───────────────┴────────────────────────► GND_POWER
```

#### Key Design Safeguards:
1. **Gate Pull-Down Resistor (10 kΩ)**: **Mandatory.** When the ESP32-S3 boots or resets, GPIOs float in high-impedance mode. The $10\text{ k}\Omega$ resistor guarantees the MOSFET gate is clamped to $0\text{V}$, preventing accidental pump pulses.
2. **Gate Series Resistor (100 Ω)**: Dampens high-frequency ringing on the gate caused by parasitic trace inductance, preventing MOSFET oscillation.
3. **Schottky Flyback Diode (SS14)**: Positioned **directly across the terminal block pins**. When the MOSFET turns OFF, the collapsing magnetic field in the motor winding induces an instantaneous voltage spike of hundreds of volts ($V = -L \frac{di}{dt}$). The SS14 Schottky diode clamps this spike to $0.45\text{V}$, protecting the MOSFET drain from avalanche breakdown.

---

### B. Hardware Electromechanical Interlock Circuit (Eliminating VULN-04)

> [!IMPORTANT]
> To eliminate **VULN-04** (the software-only single point of failure identified in the audit), physical hardware must prevent simultaneous acid and base injection even if the microcontroller firmware crashes with both GPIOs stuck HIGH.

#### Electromechanical Break-Before-Make Schematic:
```
+12V_RAW ──────► [ Relay 1: pH Up COM ]
                    │
                    ├─── Normally Open (NO) ────────► [ pH Up Pump (+) ] ──► (GND)
                    │
                    └─── Normally Closed (NC) ──┐
                                                │
                                                ▼
                                         [ Relay 2: pH Down COM ]
                                                │
                                                └─── Normally Open (NO) ──► [ pH Down Pump (+) ] ──► (GND)
```

#### Physical Interlock Proof:
1. When **Relay 1 (pH Up)** is energized, its mechanical armature swings to the **Normally Open (NO)** contact, powering the pH Up pump.
2. The circuit to the **Normally Closed (NC)** contact is **physically severed**.
3. Therefore, the common terminal of **Relay 2 (pH Down)** receives **zero electrical current**, making it physically impossible for the pH Down pump to run while pH Up is active, regardless of MCU state!

---

### C. Opto-Isolated AC Relay Driver Circuit
Applies to channels: `relay1` (GPIO 47), `relay2` (GPIO 14), `relay3` (GPIO 21), and `relay4` (GPIO 45).

```
+5V_DIGITAL ────[ Relay Coil 5V ]────┬───► Collector [ NPN Transistor SS8050 ]
                                     │     Base ◄── [ 1kΩ ] ◄── Pin 4 (Opto Emitter)
                              [ Flyback Diode ]
                              [ 1N4148 / SS14 ]
                                     │
+5V_DIGITAL ─────────────────────────┘
                                           [ Optocoupler PC817 / EL817 ]
+3.3V_MCU ────[ 470Ω ]────► Pin 1 (Anode)      Pin 3 (Collector) ──► +5V_DIGITAL
GPIO_RELAY ───────────────► Pin 2 (Cathode)
```
- **Galvanic Isolation**: The PC817 optocoupler provides $5,000\text{ V}_{\text{RMS}}$ dielectric isolation between the ESP32 3.3V logic and the 5V relay coils.
- **AC Contact Snubber**: Across the AC relay contacts ($220\text{V AC}$ live load), connect a series **RC snubber** ($0.1\ \mu\text{F}\ 400\text{V X2 film capacitor} + 47\ \Omega\ 1\text{W metal film resistor}$) to extinguish contact arcing when switching high-power inductive lighting ballasts.

---

## 3. High-Impedance Sensor Front-End Circuits (AFE)

### A. Laboratory-Grade pH Analog Front-End (BNC Input)
The internal resistance of a glass bulb pH probe is $10^7$ to $10^9\ \Omega$, generating a signal of only $\approx 59.16\text{ mV / pH unit}$ around $0\text{V}$ at pH 7.

```
BNC_CENTER ────────┬────────────────────────────► Non-Inverting (+) [ Ultra-Low Bias Op-Amp ]
                   │                              (TI LMC6001 / AD8605 / CA3140)
               [ 1nF COG ]                        (Input Bias Current Ib < 25 fA)
                   │
                [ V_REF ] (2.048V Precision Reference via LM4040)
                   │
BNC_SHELL ─────────┴────────────────────────────► Inverting (-) / Feedback Loop
                                                        │
                                                        ▼
                                           Analog Output ──► ADS1115 AIN0 / AIN1
```

#### Critical Layout Rule: The PCB Guard Ring
Because standard FR4 fiberglass exhibits surface moisture leakage currents on the order of nanoamperes (which completely swamps a picoampere pH signal), a **conductive copper Guard Ring** connected to the non-inverting node voltage must completely encircle the BNC center pad and high-impedance traces on all PCB layers, stripped of solder mask.

---

### B. AC-Excited TDS (Electrical Conductivity) Circuit
> [!CAUTION]
> Applying pure DC voltage to a conductivity probe in nutrient solution causes **electrolysis and electrochemical polarization**: gas bubbles form on the metal pins, the probe electrodes dissolve through galvanic corrosion, and readings drift uncontrollably.

#### AC Square-Wave Excitation Generator:
1. The ESP32 outputs a **$1\text{ kHz}$ to $2\text{ kHz}$ symmetrical square wave** via hardware PWM (LEDC).
2. The signal passes through a **DC-blocking capacitor ($1\ \mu\text{F}$ non-polarized film capacitor)**, ensuring **zero net DC current** enters the water.
3. The alternating voltage divides across a precision calibration resistor ($1.00\text{ k}\Omega\ 0.1\%$) and the liquid solution resistance ($R_x$).
4. The resulting AC amplitude is rectified by an **active precision half-wave rectifier (LMV358)** and filtered into a smooth DC voltage digitized by ADS1115 AIN1.

---

### C. Digital Water Level & 1-Wire Input Circuits

#### 1. Water Level Float Input (GPIO 5):
```
+3.3V_MCU ────[ 10kΩ Pull-Up ]────┬────[ 1kΩ Series ]────► ESP32 GPIO 5
                                  │
                               [ 100nF X7R ] (Low-Pass Filter, fc ≈ 1.6 kHz)
                                  │
FLOAT_TERMINAL_1 ─────────────────┤
FLOAT_TERMINAL_2 ─────────────────┴──────────────────────► GND_DIGITAL
```
The $10\text{ k}\Omega$ pull-up, $1\text{ k}\Omega$ series resistor, and $100\text{ nF}$ capacitor form a hardware RC filter with a $1.6\text{ kHz}$ cutoff frequency, eliminating high-frequency capacitive pickup along long tank cable runs.

#### 2. DS18B20 1-Wire Bus (GPIO 8):
- **Pull-Up Resistor**: Dedicated $4.7\text{ k}\Omega\ 1\%$ resistor to $3.3\text{V}$.
- **ESD Protection**: A dual bidirectional TVS diode (**Nexperia PESD5V0X1BCAL**) connected between GPIO 8 and GND clamps electrostatic discharges from outdoor probe handling.
- **Series Damping**: A $47\ \Omega$ series resistor placed adjacent to the MCU pin eliminates transmission line reflections on 1-Wire cables longer than 5 meters.

