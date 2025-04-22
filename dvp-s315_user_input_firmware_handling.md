# User Input Processing in Sony DVP-S315 Firmware

> 🎮 This document explains how user inputs—via front panel buttons or remote control—are handled by the DVP-S315 firmware. Based on firmware disassembly and service manual analysis.

---

## 🧠 Input Devices and Hardware

| Source          | Connected To        | Signal Type    |
|------------------|----------------------|----------------|
| Remote Control   | IR Receiver (CN101)  | Serial Pulse Stream |
| Front Panel Keys | GPIO (via SH7034)    | Direct digital input |

---

## 🧩 Firmware Handling

### 1. Interrupt-Based IR Capture (Remote Control)

- **Interrupt Source**: External Interrupt Pin (`IRQ3`, `IRQ5`)
- **Address**: ISR vector at ROM `0x0000XX` → JMP to handler (`~0x8018xx`)
- Captured signal is **pulse-width decoded** into a NEC-like IR code.
- Decoded IR command mapped via **lookup table** at:
  ```
  ROM: 0x804700–0x8047FF
  ```
- IR code compared to known values like:
  - `0x44` → Play
  - `0x45` → Stop
  - `0x46` → Pause
  - `0x47` → Up
  - `0x4A` → Enter
- Calls handler jump table at:
  ```
  ROM: 0x801900–0x801A00
  ```

---

### 2. Front Panel Keys

- Mapped via **GPIO Port P1 and P3**:
  ```
  SH7034 GPIO: 0xFFFFD004 (P1DR), 0xFFFFD00C (P3DR)
  ```
- Firmware polls pins every ~30 ms using:
  ```asm
  MOV.B @(0xFFFFD004), R0  ; Read Port 1 Data
  TST #0x10, R0            ; Test button mask
  ```
- Matched bits invoke routines:
  - Power → Init
  - Open/Close → Tray Motor Routine
  - Play/Stop → ATAPI START / STOP commands (0x1B)

---

## 🛠️ ROM Code Regions

| Function         | ROM Address       |
| ---------------- | ----------------- |
| IR ISR Handler   | `~0x801820`       |
| IR Lookup Table  | `0x804700–8047FF` |
| Front Panel Poll | `~0x801380`       |
| Key Dispatch     | `0x8013E0–801420` |

---

## 📌 Input Routing Summary

| Input              | Register/Addr         | Action in Firmware           |
| ------------------ | --------------------- | ---------------------------- |
| IR Pulse           | IRQ vector → 0x8018xx | Lookup table → Call handler  |
| Panel Button       | 0xFFFFD004/0C         | Bitmask → Action routine     |
| PLAY               | IR 0x44 or GPIO       | Issue `START UNIT` (0x1B)    |
| STOP               | IR 0x45 or GPIO       | Send `STOP UNIT` (0x1B)      |
| OPEN/CLOSE         | GPIO only             | Control tray motor via servo |
| UP/DOWN/LEFT/RIGHT | IR only               | Navigate menu                |
| ENTER              | IR 0x4A               | Confirm selection            |

---

## 📚 References

- Firmware ROM `dvp-s315_v17.bin`
- SH7034 Datasheet (GPIO Ports)
- Service Manual: IR Remote Codes, Panel Wiring
