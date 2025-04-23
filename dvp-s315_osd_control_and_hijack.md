# OSD (On-Screen Display) Control and Hijacking – Sony DVP-S315

> 🧠 This document summarizes how the Sony DVP-S315 generates on-screen display elements (menus, messages, overlays), the hardware responsible, and where a custom ODE system could intercept or augment the display output.

---

## 🔍 Architecture Summary

The main display logic is handled **outside the SH7034**, through an auxiliary subsystem based around:

- **IC604** – MB90T678 Interface Microcontroller
- **IC603** – External ROM for IC604
- **IC608** – 256K SRAM
- **IC601** – Address Latch
- **MB90096** – OSD Controller IC (handles text/graphics overlay generation)

The SH7034 communicates with IC604 indirectly via serial or bus handshake lines. The IC604 then controls the OSD output by sending **serial commands** to the MB90096.

---

## 📡 MB90096 Control Interface (OSD Generator)

- **Protocol**: 3-wire Serial
- **Controlled by**: IC604 (MB90T678)
- **Signals**:
  - `SCLK` – Clock
  - `SIN` – Serial Input (Data)
  - `CS` – Chip Select

- **Outputs** (to AV Path):
  - `ROUT`, `GOUT`, `BOUT` – RGB overlay output
  - `IOUT` – Intensity overlay
  - `VOB1`, `VOB2` – Overlay blanking signals

---

## 🧠 Control Flow Diagram

```mermaid
flowchart TD
    SH[SH7034 Main CPU]
    IC604[MB90T678 µC (IC604)]
    MB90096[MB90096 OSD Generator]
    VideoEnc[IC203 AV Decoder]

    SH -->|Commands / GPIO| IC604
    IC604 -->|SCLK, SIN, CS| MB90096
    MB90096 -->|ROUT, GOUT, BOUT| VideoEnc
```

---

## 🎯 Opportunities for OSD Hijack

| Method                | Description                          | Difficulty | Tools               |
| --------------------- | ------------------------------------ | ---------- | ------------------- |
| **Serial Sniffing**   | Monitor SCLK/SIN/CS lines from IC604 | Medium     | Logic Analyzer      |
| **Command Injection** | Inject custom commands to MB90096    | High       | CPLD/MCU inline     |
| **Full Emulation**    | Replace MB90096 with FPGA            | Very High  | Advanced FPGA Dev   |
| **Analog Overlay**    | Inject video overlays post-AVDEC     | Medium     | Video Overlay Board |

---

## ✅ What You Can Achieve

- Replace menu text with custom messages
- Show disc/image selection UI from ODE
- Inject error/status overlays
- Potentially disable original overlays

---

## 🔌 Hardware Insertion Points

| Point                   | Signal(s)                      | Description                        |
| ----------------------- | ------------------------------ | ---------------------------------- |
| Between IC604 ↔ MB90096 | `SCLK`, `SIN`, `CS`            | Serial command hijack or injection |
| MB90096 Outputs         | `ROUT`, `GOUT`, `BOUT`, `IOUT` | Analog overlay injection           |
| Composite Output        | Video Signal                   | Post-process external overlay      |

---

## 📚 References

- Block Diagram: Fig. 2-18 System Controller Diagram
- IC604 (MB90T678) – Interface Microcontroller
- MB90096 Datasheet – OSD Controller IC
- `dvp-s315_v17.bin` – Reference for firmware coordination
