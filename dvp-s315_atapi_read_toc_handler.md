# ATAPI READ TOC Handler in Sony DVP-S315 Firmware

> 📀 This document explains the firmware-level handling of the ATAPI `READ TOC` command (`0x43`) in the Sony DVP-S315. This command is used by the host to retrieve the Table of Contents (TOC) from a DVD disc. For an ODE system, proper TOC emulation is essential to ensure title/chapter display and navigation work correctly.

---

## 📌 Location in ROM

- **Handler Routine:** `0x801C00–0x801C90`
- Triggered when a `0x43` opcode is passed via the ATAPI dispatcher
- Returns either:
  - A standard TOC header and track descriptor structure
  - Or a SENSE error (e.g., 0x3A) if disc/image is not mounted

---

## 🧠 Routine Behavior Summary

The handler:

1. Parses `READ TOC` parameters (Format, MSF flag, Starting Track)
2. Copies a static or synthesized TOC structure into a PIO output buffer
3. Sets DRQ, waits for DIOR-, transmits the TOC using ATA DATA port
4. Clears DRQ after completion

---

## 🧬 Simplified Pseudocode

```c
if (!media_present) {
    set_status(CHECK CONDITION);
    prepare_sense(0x3A); // No media
    return;
}

len = 12;  // Default response length
buf[0] = 0x00;  // Data Length MSB
buf[1] = len - 2;  // Data Length LSB
buf[2] = 0x01;  // First Track
buf[3] = 0x01;  // Last Track
buf[4..11] = { Track Descriptor } // e.g., Title 1 LBA

set_status(BSY=1);
delay(100 us);
set_status(DRQ=1, BSY=0);

pio_send(buf, len);
clear_status(DRQ);
```

---

## 📄 Response Format (Simplified)

| Byte | Field               | Value (Example)     |
|------|---------------------|---------------------|
| 0–1  | Data Length         | 0x000A              |
| 2    | First Track Number  | 0x01                |
| 3    | Last Track Number   | 0x01                |
| 4    | Reserved            | 0x00                |
| 5    | ADR/Control         | 0x14 (Data Track)   |
| 6    | Track Number        | 0x01                |
| 7    | Reserved            | 0x00                |
| 8–11 | Track Start Address | LBA: 0x00000000     |

📌 The total length is typically **12 bytes**, unless multisession or extra formats are supported.

---

## 🕒 Emulation Timing

| Phase             | Action              | Timing Target      |
|-------------------|---------------------|---------------------|
| Start             | BSY=1               | Immediate           |
| Prepare TOC       | Fill buffer         | ~100 µs             |
| Send              | DRQ=1, host reads   | ≤250 µs from start  |
| End               | DRQ=0               | After 12 bytes      |

---

## 🧰 ODE Implementation Notes

1. Detect opcode `0x43`
2. Parse PACKET for format parameters (bytes 1, 2, 6, etc.)
3. If media not mounted:
   - Set status `0x51`, sense `0x3A 00`
4. Otherwise:
   - Send default 1-track TOC
   - Ensure DRQ/BSY timings are followed
   - Respond with correct TOC format (binary)

---

## 📚 References

- Firmware ROM: `dvp-s315_v17.bin`, `0x801C00–0x801C90`
- ATAPI SFF-8020i specification
- Logic trace from Sony DVP-S315 TOC reads
- ODE behavior from Dreamcast GD-ROM emulator project
