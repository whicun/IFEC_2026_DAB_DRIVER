# 🔋 1 kW Dual Active Bridge (DAB) Converter — Embedded Firmware

> Real-time closed-loop firmware for a bidirectional **1 kW Dual Active Bridge DC-DC converter** with **CC-CP-CV charging** and **Vehicle-to-Grid (V2G)** support, running on a **TI TMS320F2800137** DSP at **100 kHz switching / 50 kHz control rate**.

Developed as part of the **IFEC 2026** power electronics project.

---

## ✨ Key Features

| Category | Details |
|---|---|
| **Topology** | Phase-Shifted Full-Bridge Dual Active Bridge (DAB) |
| **Power Rating** | 1 kW bidirectional (Grid-to-Vehicle & Vehicle-to-Grid) |
| **MCU / DSP** | TI TMS320F2800137 (C2000 Real-Time MCU, 120 MHz) |
| **Switching Frequency** | 100 kHz (ePWM) |
| **Control Loop Rate** | 50 kHz ISR (ADC-triggered, executed from RAM) |
| **Control Strategy** | Cascaded PI — Outer Voltage (CV) → Inner Current (CC) with Constant Power (CP) limiting |
| **Bidirectional** | Automatic G2V ↔ V2G mode switching via bus voltage hysteresis |
| **Protection** | Hardware trip zones (overcurrent via DCAEVT1, overvoltage via DCBEVT1) with one-shot PWM latch-off |
| **Telemetry** | Real-time binary telemetry over RS-485 (SCI @ 230400 baud) |
| **Toolchain** | TI Code Composer Studio (CCS) with C2000Ware DriverLib + Digital Control Library (DCL) |

---

## 🏗️ System Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                     TMS320F2800137 DSP                         │
│                                                                │
│  ┌──────────┐   ┌──────────┐   ┌───────────┐   ┌───────────┐  │
│  │  ADC     │──▶│ Filters  │──▶│  Scaling  │──▶│ PI Control│  │
│  │ (V,I)   │   │ MA + IIR │   │  & PU     │   │ CC-CP-CV  │  │
│  └──────────┘   └──────────┘   └───────────┘   └─────┬─────┘  │
│                                                       │        │
│  ┌──────────┐   ┌──────────┐   ┌───────────┐         │        │
│  │ Trip Zone│   │  Gate    │◀──│   Phase   │◀────────┘        │
│  │ (HW OCP/ │   │  Driver  │   │   Shift   │                  │
│  │  OVP)   │   │  Enable  │   │  (ePWM)   │                  │
│  └──────────┘   └──────────┘   └───────────┘                  │
│                                                                │
│  ┌──────────────────────┐                                      │
│  │  RS-485 Telemetry    │──▶  Host PC (real-time monitoring)   │
│  └──────────────────────┘                                      │
└────────────────────────────────────────────────────────────────┘
         │                              │
    Primary H-Bridge              Secondary H-Bridge
    (ePWM1 + ePWM2)              (ePWM3 + ePWM4)
         │           Transformer          │
         └───────────── ⚡ ───────────────┘
```

---

## 📁 Project Structure

```
├── main.c                  # Application entry point, 50 kHz ISR, background loop
├── dab_hal.c / .h          # Hardware Abstraction Layer (ADC reads, PWM phase shift, gate drivers)
├── dab_control.c / .h      # Cascaded PI control (CC-CP-CV charge + V2G discharge)
├── dab_scaling.c / .h      # ADC-to-engineering-unit conversion with calibration constants
├── dab_filter.c / .h       # Moving-average + IIR filters for voltage; fast IIR for current
├── dab_telemetry.c / .h    # Binary telemetry framing over RS-485 / SCI
├── 280013x_generic_ram_lnk.cmd   # Linker command file (RAM + Flash memory map)
├── device/                 # C2000Ware DriverLib & device support
└── targetConfigs/          # CCS target configuration (.ccxml) for LaunchPad
```

---

## ⚙️ Control Strategy

### Charge Mode (Grid-to-Vehicle)

The converter implements a **CC → CP → CV** charging profile with seamless handover:

1. **Constant Current (CC):** PI-regulated battery current up to configurable limit (e.g., 2.5 A @ 450 V)
2. **Constant Power (CP):** Dynamic current cap tracks `P_limit / V_bat` to prevent exceeding the power envelope
3. **Constant Voltage (CV):** Outer voltage PI loop tapers current as V_bat approaches the target

### Discharge Mode (Vehicle-to-Grid)

V2G mode mirrors the charge control with **inverted error math** and a **soft-landing taper** near the low-voltage cutoff to prevent battery under-voltage.

### Mode Switching

The controller **autonomously detects grid intent** by monitoring V_bus with hysteresis thresholds. Direction swaps use a **zero-dwell ramp-down** protocol: current slews to zero, all PI integrators are reset, then the new mode engages — eliminating transient spikes.

### Anti-Windup & Safety

- PI output saturation with `Umax` dynamically set to `min(I_CC_limit, P_limit / V_bat)`
- Slew-rate limiter on current reference (smooth ramp-up/down)
- Light-load dead-time cutoff prevents bang-bang oscillations at near-zero current
- Hardware trip zones (one-shot latch) for overcurrent and overvoltage faults

---

## 🛡️ Hardware Protection

| Feature | Implementation |
|---|---|
| **Overcurrent Protection** | CMPSS → TRIP4 → DCAEVT1 → One-Shot trip (PWMs forced low) |
| **Overvoltage Protection** | CMPSS → TRIP5 → DCBEVT1 → One-Shot trip (PWMs forced low) |
| **Gate Driver Enable** | Active-low enable pins with software interlock |
| **Boot Sequence** | 5-second LED heartbeat delay + automatic trip clear before enabling PWMs |
| **Fault Diagnostics** | Dedicated LEDs for OCP vs OVP fault identification |

---

## 📊 Real-Time Telemetry

A 32-byte binary telemetry frame is transmitted each background loop iteration over **RS-485** (half-duplex):

| Field | Type | Description |
|---|---|---|
| Header | `0xAA 0x55` | Sync bytes |
| V_bus_raw | `uint16` | Raw ADC count — bus voltage |
| V_bat_raw | `uint16` | Raw ADC count — battery voltage |
| I_bus_raw | `int16` | Raw PPB count — bus current |
| I_bat_raw | `int16` | Raw PPB count — battery current |
| V_bus_V | `float32` | Calibrated bus voltage (V) |
| V_bat_V | `float32` | Calibrated battery voltage (V) |
| I_bus_A | `float32` | Calibrated bus current (A) |
| I_bat_A | `float32` | Calibrated battery current (A) |
| phase_pu | `float32` | Active phase shift (per-unit) |
| mode | `int16` | +1 = Charge, -1 = Discharge, 0 = Standby |

---

## 🚀 Performance Optimizations

- **RAM Execution:** Critical ISR and control functions are placed in `.TI.ramfunc` via `#pragma CODE_SECTION` for zero-wait-state execution
- **Division Elimination:** All per-unit and scaling computations use pre-computed inverse multipliers (40× faster than FPU division on C28x)
- **Circular Buffer:** Voltage moving-average filter uses index wrap instead of modulo for branch-free operation
- **TI DCL Library:** Leverages `DCL_runPI_C6` for optimized PI execution with built-in anti-windup

---

## 🔧 Build & Flash

### Prerequisites

- [TI Code Composer Studio](https://www.ti.com/tool/CCSTUDIO) (v12+)
- [C2000Ware SDK](https://www.ti.com/tool/C2000WARE) (includes DriverLib and DCL)
- TMS320F2800137 LaunchPad or custom target board

### Steps

1. Clone this repository
2. Import the project into CCS (`File → Import → CCS Projects`)
3. Select the `CPU1_LAUNCHXL_FLASH` build configuration for deployment
4. Build and flash to the target via the XDS110 debug probe

---

## 📐 Configurable Voltage Profiles

The firmware supports multiple voltage profiles selectable at compile time via `SYSTEM_TEST_VOLTAGE`:

| Profile | V_bat Range | Power Limit | CC Limit | Use Case |
|---|---|---|---|---|
| 100 V | 77.8 – 100 V | 80 W | 0.9 A | Bench testing |
| 150 V | 116.7 – 150 V | 180 W | 1.35 A | Low-voltage validation |
| 200 V | 155.6 – 200 V | 320 W | 1.8 A | Mid-voltage testing |
| 250 V | 194.4 – 250 V | 500 W | 2.25 A | Integration testing |
| 300 V | 233.3 – 300 V | 660 W | 2.47 A | High-voltage validation |
| **450 V** | **350 – 450 V** | **1000 W** | **2.5 A** | **Full-scale target** |

---

## 🛠️ Tech Stack

`C` · `TI C2000 DSP` · `TI DriverLib` · `Digital Control Library (DCL)` · `ePWM` · `ADC + PPB` · `CMPSS` · `Trip Zones` · `RS-485 / SCI` · `Code Composer Studio`

---

## 📝 Changelog

### v2.1
- Changed PWM switching frequency to 100 kHz
- Changed dead band to total 2 µs

---

## 📄 License

This project was developed for the **IFEC 2026** initiative. Please contact the author for licensing inquiries.
