# Emulation Ready Sheet – Sony DVP-S315 ODE Interface (CN601)

> 🎯 This document lists all known memory-mapped registers and address lines that are routed through connector **CN601**, which connects the mainboard to the original DVD drive. These registers must be emulated or handled properly to ensure compatibility with an SSD-based ODE replacement.

---

## 📦 Primary Interface: ATAPI (IDE) Bus

The DVD drive is connected to the SH7034 via a parallel IDE (ATAPI) interface routed through CN601. It uses a memory-mapped register block starting at:

```text
Base Address: 0xFFFFEC00
```

This block includes:

- ATAPI command register (0xFFFFEC07)
- Data buffer (0xFFFFEC00)
- Status & control (0xFFFFEC07, 0xFFFFEC08)
- Signature bytes for ATAPI detection (0xFFFFEC04/05 = 0x14 / 0xEB)

✅ Must be fully emulated.

📄 See: `dvp-s315_atapi_register_reference.md`

---

## 📡 Drive Presence Detection & Servo Control (likely routed via CN601) (Correction avail.)

Additional CN601 lines are likely used for signaling:

| Line          | Purpose                | Notes for ODE                                     |
| ------------- | ---------------------- | ------------------------------------------------- |
| `DRV_IN`      | Drive inserted / ready | Should be tied HIGH or software-simulated         |
| `SLED_HOME`   | Sled position sensor   | May need to simulate home position                |
| `DISC_DETECT` | Optical disc present   | Affects TEST UNIT READY & STATUS                  |
| `SERVO_ACK`   | Servo ready signal     | Optional, but likely polled                       |
| `SPDL_OK`     | Spindle motor status   | Must simulate disc spin-up if firmware polls      |
| `RESET_DRV`   | Drive-side reset line  | Handle as edge-triggered reset for emulated logic |

⏹️ Some of these lines may go through the ICS604 (Servo Controller), and appear on CN601 as digital control/status lines.

ℹ️ **CN601**: see Chapter **Corrections** below!

---

## 🔧 Other Memory-Mapped Addresses (Not on CN601)

These do **not** go through CN601, but are useful to simulate drive status during boot:

| Address      | Description              | Use in Emulation                            |
| ------------ | ------------------------ | ------------------------------------------- |
| `0xFFFFD010` | SH7034 GPIO Port Control | Can be used for RESET or Drive Select lines |
| `0xFFFFFEC0` | Interrupt Mask Register  | Optional, for full IRQ compliance           |
| `0xFFFFFEE0` | DMA / Bus Arbitration    | May be monitored, not critical              |

---

## ✅ Summary: Signals Required for ODE Compatibility

| Category      | Required?   | Emulate?            | Notes                                      |
| ------------- | ----------- | ------------------- | ------------------------------------------ |
| ATAPI Regset  | ✅ Yes      | ✅ Yes              | Must fully implement ATAPI PIO I/O         |
| IDE Handshake | ✅ Yes      | ✅ Yes              | BSY, DRQ, INTRQ must follow timing         |
| SENSE/STATUS  | ✅ Yes      | ✅ Yes              | Must provide correct codes                 |
| SLED/SPDL OK  | ❌ Optional | ⚠️ Yes (simulate) | Needed if firmware checks mechanical state |
| GPIO          | ⚠️ Maybe  | ⚠️ Partial        | Depends on drive model                     |
| IRQ Support   | ❌ No       | ❌ No               | Firmware uses polling, not interrupts      |

---

## 📚 References

- `sony_dvp-s300_305_315.pdf` (Service Manual)
- `sony_dvp-s300_s305_s315_supplement-1.pdf` (Supplement with CN601 info)
- Signal analysis from `Signal-Chain-Analysis.pdf`
- Firmware dump `dvp-s315_v17.bin`

---

## 🔍 Update: SLED/SPDL Detection in Firmware

Based on binary analysis of `dvp-s315_v17.bin`, the firmware does **not contain any explicit checks or memory accesses** targeting GPIO or IDE status lines commonly associated with:

- `SLED_HOME`
- `SPDL_OK`
- `DISC_DETECT`

### ❌ No absolute memory accesses to

- `0xFFFFD0xx` (GPIO region)
- `0xFFFFECxx` (ATAPI register set)
- `0xFFFFFECx` (interrupt config)

### 🔬 No ASCII strings suggest UI feedback for mechanical failures

### ✅ ODE Implication

You are likely safe to **tie mechanical signals like SLED_OK or SPDL_OK permanently HIGH or simulate them statically**. The firmware does not appear to depend on polling them or adjusting drive behavior based on those inputs.

> This simplifies ODE implementation: focus solely on ATAPI command emulation and correct timing for `BSY`, `DRQ`, and sense data.

---

## 🔁 Correction: CN601 does NOT Carry Mechanical Servo Signals

Previous versions of this document suggested that signals like `SPDL_OK`, `SLED_HOME`, or `DISC_DETECT` may be routed via **CN601**.

### ❌ This is incorrect

### ✅ Updated Analysis

- **CN601** is strictly used for **IDE/ATAPI bus communication**, including:
  - Data lines (D0–D15)
  - Control lines (CS0, CS1, DIOR, DIOW)
  - ATA status and command handshaking

- Mechanical and servo-related signals are instead routed through:
  - **CN302** → Tray, sled, and chucking status (e.g., `CHUCK_SW`, `TRAY_SW`, `LDMT`)
  - **CN452** → Optical pickup and servo lines (`FOK`, `SPDL_CTL`, `TILT_CTL`, etc.)

### 🧠 ODE Developer Guidance

- If your ODE interfaces only with CN601, you **do not need to replicate or interpret SLED/SPDL/DETECT lines**
- However, you **must simulate the firmware-visible behavior** of those signals via:
  - Correct ATAPI STATUS flags (`BSY`, `DRQ`, `CHECK`)
  - Meaningful SENSE codes (e.g., NO DISC, HARDWARE ERROR)

> This clarification reduces complexity in the hardware wiring of the ODE interface.
