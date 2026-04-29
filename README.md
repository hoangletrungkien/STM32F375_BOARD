# STM32H745 Industrial Control Board — 4-Layer PCB

![3D Render](docs/render.png)

> **Status:** Design Complete — DRC Clean  
> **Tool:** Altium Designer | **PCB:** 4-Layer, JLCPCB stackup  
> **Date:** 2026

---

## Overview

Multi-MCU industrial-grade carrier board built around the STM32H745ZIT3 dual-core (Cortex-M7 + M4) paired with an ATmega324PA co-processor. Features onboard eMMC storage (MTFC32G, 32GB), galvanically isolated UART communication, and USB 2.0 full-speed.

---

## Specifications

| Parameter      | Value                                            |
| -------------- | ------------------------------------------------ |
| Main MCU       | STM32H745ZIT3 (Cortex-M7 @ 480MHz + M4 @ 240MHz) |
| Co-MCU         | ATmega324PA                                      |
| Storage        | MTFC32G eMMC (32GB, HS400 capable)               |
| UART isolation | Isolated UART bridge (galvanic isolation)        |
| USB            | USB 2.0 Full-Speed (STM32 native)                |
| Power input    | 12V DC                                           |
| Power rails    | 12V → 5V → 3.3V (multi-stage)                    |
| PCB layers     | 4                                                |
| Target fab     | JLCPCB 4-layer stackup                           |

---

## Block Diagram

```
12V DC Input
     │
  12V → 5V (buck, 1.5A)
     │
  5V → 3.3V (LDO, 1A) ──────── 3.3V_MCU rail
     │
     ├── STM32H745ZIT3 (main MCU)
     │         │
     │    ┌────┴────┐
     │   USB     eMMC (MTFC32G)
     │  (J1)   (HS400, 8-bit bus)
     │         │
     │    Isolated UART ──── External device
     │         │
     └── ATmega324PA (co-MCU, 5V domain)
               │
          Analog buffer
          ADC interface
          Header J2 (5V I/O)
```

---

## Schematic Blocks

### POWER (Sheet: POWER)

- Input: 12V DC, 12V → 5V @ 1.5A (synchronous buck)
- 5V → 3.3V @ 1A (LDO regulator)
- Bulk decoupling: 1000µF at each stage
- HF decoupling: 100nF ceramic per IC power pin
- Dedicated 5V domain for ATmega and analog buffer

### MCU (Sheet: MCU — STM32H745ZIT3)

- 144-pin LQFP package
- VDD decoupled individually per bank (100nF + 10µF per pin group)
- VCAP1/VCAP2: internal LDO bypass (2.2µF each — mandatory per datasheet)
- HSE crystal: 25MHz (load caps per crystal datasheet CL)
- SWD debug: SWCLK + SWDIO on J4 (4-pin header)
- USB D+/D−: 90Ω differential pair, series resistors 22Ω, ESD protection TVS

### eMMC (Sheet: eMMC — MTFC32G)

- Interface: 8-bit MMC bus (D0–D7, CLK, CMD)
- Signal integrity: all 10 signals length-matched within ±0.5mm
- Pull-ups: 47kΩ on CMD and D0–D7 (required for eMMC init)
- Decoupling: 100nF per VCC/VCCQ pin, 10µF bulk
- VCCQ: 1.8V (HS400 mode requirement) — separate LDO
- Note: HS400 requires impedance-controlled traces, 50Ω single-ended

### AVR (Sheet: AVR — ATmega324PA)

- Operates at 5V domain
- UART connection to STM32 via isolated UART bridge
- ADC inputs from analog buffer circuit
- ISP programming header: MOSI/MISO/SCK/RST/VCC/GND
- External crystal: 16MHz

### ISO UART (Sheet: ISO UART)

- Galvanic isolation IC between STM32 UART and ATmega UART
- Isolation barrier: 2500V (typical for ISO7221 or similar)
- Separate GND planes on each side of isolation barrier
- Ferrite bead on power crossing the isolation barrier

---

## Key Design Decisions

### 1. eMMC HS400 Signal Integrity

HS400 mode runs at 200MHz DDR (400MB/s). At this speed, even small length mismatches cause setup/hold violations.

All 10 eMMC signals (D0–D7, CLK, CMD) are length-matched to within ±0.5mm using the interactive length tuning tool in Altium. Traces are routed on L1 with solid GND plane on L2 as the reference. No vias on the eMMC bus after the package breakout.

### 2. STM32H745 VCAP Requirement

The STM32H745 has an internal voltage regulator (SMPS + LDO). VCAP1 and VCAP2 must be decoupled with 2.2µF ceramic capacitors placed within 1mm of the pins — if omitted or wrong value, the MCU will not start reliably. This is a common beginner mistake.

### 3. Galvanic Isolation Architecture

The ATmega324PA operates in the 5V industrial domain. Direct UART connection to the 3.3V STM32 would create a ground loop risk in industrial environments. The isolated UART bridge provides:

- Level shifting (3.3V ↔ 5V)
- Galvanic isolation (GND domains fully separated)
- Protection from transient voltage spikes on the field side

### 4. 4-Layer Stackup (JLCPCB)

```
L1 — Signal + Power (1 oz)    ← top routing, USB D+/D-
L2 — GND plane (0.5 oz)       ← continuous reference plane
L3 — Power plane (0.5 oz)     ← 3.3V and 5V polygon pours
L4 — Signal (1 oz)             ← bottom routing, eMMC bus
```

L2 GND plane is uninterrupted under all high-speed signals (USB, eMMC). Splits in GND only occur at the isolation barrier for ISO UART.

### 5. USB ESD Protection

USB D+/D− are exposed to the outside world and prone to ESD events. TVS diode array placed within 5mm of the USB connector before the series resistors. Series 22Ω resistors limit current during ESD and damp impedance discontinuities.

---

## BOM Summary (key components)

| Ref | Part             | Description               |
| --- | ---------------- | ------------------------- |
| U1  | STM32H745ZIT3    | Dual-core MCU, LQFP-144   |
| U2  | MTFC32G          | 32GB eMMC, BGA-153        |
| U3  | ATmega324PA      | 8-bit AVR co-MCU, TQFP-44 |
| U4  | Isolated UART IC | UART isolation bridge     |
| U5  | Buck converter   | 12V→5V, 1.5A              |
| U6  | LDO              | 5V→3.3V, 1A               |
| Y1  | Crystal          | 25MHz (STM32 HSE)         |
| Y2  | Crystal          | 16MHz (ATmega)            |
| J1  | USB Type-B       | USB 2.0 FS connector      |
| J2  | Header 32-pin    | ATmega I/O expansion      |
| J3  | Header 32-pin    | STM32 GPIO expansion      |
| J4  | Header 4-pin     | SWD debug                 |

---

## 4-Layer PCB Layout Notes

- **Via stitching:** GND vias every 3mm along board edges and between power domains
- **Crystal keepout:** 3mm keepout around Y1, no copper pour in crystal area
- **USB routing:** D+/D− on L1, no vias, GND plane L2 uninterrupted beneath
- **eMMC routing:** All signals on L4, length-matched, 50Ω single-ended
- **Isolation barrier:** Physical split in L2/L3 GND/power planes at ISO UART location

---

## Files

| File                     | Description         |
| ------------------------ | ------------------- |
| `schematic.pdf`          | Power supply sheet  |
| `layout_top.png`    | PCB layout — top    |
| `layout_bottom.png` | PCB layout — bottom |
| `render.png`        | Altium 3D render    |

---

## Tools Used

- Altium Designer 24
- JLCPCB impedance calculator (stackup verification)
- STM32CubeMX (pin assignment reference)

---

_Design by Hoang Le Trung Kien — HCMUT Electronics & Communication Engineering_
