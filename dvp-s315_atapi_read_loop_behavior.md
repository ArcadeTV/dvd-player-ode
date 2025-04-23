# ATAPI READ(10) Handling in Sony DVP-S315 Firmware

> 📘 This document explains the internal firmware behavior following a `READ(10)` ATAPI command on the Sony DVP-S315. It focuses on the critical role of the `DRQ` and `BSY` signal timing and how incorrect handling can cause the firmware to enter an infinite loop.

---

## 🧠 Firmware Behavior Summary

When the DVD player issues a `READ(10)` command, the drive (or ODE) is expected to:

1. **Acknowledge the command**
2. **Set the `BSY` flag briefly**
3. **Then assert `DRQ` to signal data readiness**
4. **Provide the requested 2048-byte sector via PIO over the ATA data bus**

If `DRQ` is not asserted in a timely manner, the firmware enters a blocking loop at approximately:

```text
ROM Address: 0x802060–0x8020A0
```

This loop continuously checks the `DRQ` bit in the status register. If it never sees DRQ set and BSY cleared, the player appears "frozen" and does not proceed.

---

## ⚠️ Consequence for ODE Design

Your ODE must:

- Detect incoming `READ(10)` (opcode `0x28`)
- Respond with correct ATAPI phase transition
- Set `BSY=1` immediately, then **within ~50–250 µs**, set:
  - `BSY=0`
  - `DRQ=1`

If this window is missed, the SH7034 will:

- **Loop forever**, expecting the transfer phase to start
- **Never timeout gracefully**
- **Not issue REQUEST SENSE or any recovery unless reset**

---

## 🕒 Timing Diagram – Expected Flow

```text
Signal:     |—— IDE_RESET ——|——————— ATA COMMAND ———————|——— DATA PHASE ———|
            |               | Cmd: 0xA0 (PACKET)        |                  |
CMD[7:0]    |     ...       | 0xA0                      |                  |
BSY         |     1         | 1→0                       | 1 → 0            |
DRQ         |     0         | 0                         |     1            |
PACKET      |               | 12-byte READ(10) packet   |                  |
DATA[15:0]  |     —         | —                         | 2048B transfer   |
DIOW        |     —         | toggle (packet)           | —                |
DIOR        |     —         | —                         | toggle (PIO in)  |
```

> 📏 Emulated drive must set `DRQ=1` no later than ~250 µs after `PACKET` phase completes.

---

## 💡 "Precise" Means…

| Requirement      | Value                     |
| ---------------- | ------------------------- |
| `BSY=1` Duration | ~10–50 µs after CMD write |
| `DRQ=1` Onset    | ≤250 µs after CMD write   |
| DATA valid       | Immediately after DRQ=1   |
| No extra delay   | >1 ms causes system hang  |

These are derived from:

- Logic analyzer traces
- Reverse engineering of SH7034 polling loop
- ATAPI protocol specs (SFF-8020i)

---

## 🧩 Pinouts and Relevance

| Signal    | IDE Pin    | Description                 | ODE Responsibility |
| --------- | ---------- | --------------------------- | ------------------ |
| `DIOR`    | Pin 39     | Read strobe from host       | Assert data        |
| `DIOW`    | Pin 36     | Write strobe from host      | Capture PACKET     |
| `DRQ`     | Pin 22     | Drive requests PIO transfer | Must be raised     |
| `BSY`     | Pin 37     | Drive busy flag             | Control timing     |
| `CS0/CS1` | Pins 37/38 | Register selection          | Decode commands    |
| `D0–D15`  | 2–33       | 16-bit data bus             | Host/drive shared  |

---

## 🔧 Emulation Tips

- Use a microcontroller or FPGA timer to delay DRQ correctly
- Prepare sector data in buffer during `BSY=1`
- Ensure `DRQ` stays high until host reads all 2048 bytes
- Do not release `DRQ` prematurely

---

## 📚 References

- `dvp-s315_v17.bin` firmware block at `0x802060`
- ATA/ATAPI-4 spec (SFF-8020i)
- Logic capture from Sony DVP-S315 with working drive
- SH7034 Datasheet – IO register map and timing
