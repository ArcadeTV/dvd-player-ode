# CN601 to ODE Communication Reference – Sony DVP-S315

> 📘 This document details the communication protocol between the Sony DVP-S315 mainboard (via CN601) and the optical drive emulator (ODE). It includes observed ATAPI command sequences, signal behavior, register usage, CN601 pinout, and required ODE responses and timings.

---

## 🧭 Overview

### Connector CN601

- **Type:** 27-pin Flat Flexible Cable (FFC)
- **Role:** Primary communication channel between SH7034 and TK-47/DVD loader
- **Signals:** Carries ATAPI (IDE-style) control and data lines

> ✅ Verified via schematic (`sony_dvp-s300_305_315.pdf`) and supplement (`sony_dvp-s300_s305_s315_supplement-1.pdf`)

---

## 📋 CN601 Pin Mapping (Verified)

| Pin | Signal     | Description                     |
|-----|------------|---------------------------------|
| 1   | GND        | Ground                          |
| 2   | D0         | Data Bit 0                      |
| 3   | D1         | Data Bit 1                      |
| 4   | D2         | Data Bit 2                      |
| 5   | D3         | Data Bit 3                      |
| 6   | D4         | Data Bit 4                      |
| 7   | D5         | Data Bit 5                      |
| 8   | D6         | Data Bit 6                      |
| 9   | D7         | Data Bit 7                      |
|10   | D8         | Data Bit 8                      |
|11   | D9         | Data Bit 9                      |
|12   | D10        | Data Bit 10                     |
|13   | D11        | Data Bit 11                     |
|14   | D12        | Data Bit 12                     |
|15   | D13        | Data Bit 13                     |
|16   | D14        | Data Bit 14                     |
|17   | D15        | Data Bit 15                     |
|18   | nRESET     | Reset to drive                  |
|19   | DIOR-      | I/O Read Strobe                 |
|20   | DIOW-      | I/O Write Strobe                |
|21   | CS0-       | Register Select 0               |
|22   | CS1-       | Register Select 1               |
|23   | DRQ        | Drive Request for PIO transfer  |
|24   | BSY        | Drive Busy                      |
|25   | INTRQ      | Interrupt Request (optional)    |
|26   | DMARQ      | DMA Request (unused in ATAPI)   |
|27   | 5V         | Power Supply                    |

---

## 🧠 ATAPI Command Lifecycle

```text
[CPU sends PACKET CMD] →
→ [ODE responds: BSY=1 then DRQ=1] →
→ [Host sends 12-byte PACKET (e.g., READ(10))] →
→ [ODE provides data / TOC / sense] →
→ [Host reads until DRQ cleared]
```

→ *Delays or improper signal transitions here cause firmware hangs.*

---

## 📚 I/O Register Mapping (via CN601, Memory-Mapped)

| Host Address | Register Name     | Access | Description                 |
| ------------ | ----------------- | ------ | --------------------------- |
| `0xFFFFEC00` | Data Register     | R/W    | Sector data (16-bit)        |
| `0xFFFFEC01` | Error / Features  | R/W    | Read error / write features |
| `0xFFFFEC02` | Sector Count      | R/W    | Command parameter           |
| `0xFFFFEC03` | Sector Number     | R/W    | Command parameter           |
| `0xFFFFEC04` | Cylinder Low      | R/W    | Signature / param           |
| `0xFFFFEC05` | Cylinder High     | R/W    | Signature / param           |
| `0xFFFFEC06` | Drive Select      | R/W    | Select master/slave         |
| `0xFFFFEC07` | Status / Command  | R/W    | Write command / read status |
| `0xFFFFEC08` | Alt Status / Ctrl | R/W    | Reset / INT mask            |

---

## 🕒 Emulation Timing (per Command Phase)

| Phase               | DRQ | BSY | Expected Delay       | Notes                  |
| ------------------- | --- | --- | -------------------- | ---------------------- |
| Command Write       | 0   | 1   | 0–10 µs              | Set BSY                |
| DRQ Raise           | 1   | 0   | <250 µs              | Set DRQ, clear BSY     |
| Data Transfer Start | 1   | 0   | Within 1 ms          | Host starts reading    |
| DRQ Clear           | 0   | 0   | After last byte read | Ready for next command |

---

## 🧰 Implementation Notes (ODE Developer)

- Simulate precise DRQ/BSY transitions via logic
- Ensure valid TOC/SENSE responses
- Respect register mappings and bus timing
- Optionally: Tie unused lines HIGH (e.g., DMARQ, INTRQ)

---

## References

- `sony_dvp-s300_305_315.pdf` – Service Manual
- `dvp-s315_v17.bin` – Firmware Analysis
- SFF-8020i (ATAPI Spec)
- Real hardware trace captures
