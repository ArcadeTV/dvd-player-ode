# ATAPI SENSE Template Handler in Sony DVP-S315 Firmware

> 🧠 This document details the structure, logic, and firmware-level routines that generate ATAPI SENSE data in response to a CHECK CONDITION in the Sony DVP-S315 DVD player. It explains how the firmware manages fixed SENSE templates and returns error diagnostics to the host via `REQUEST SENSE`.

---

## 📌 Location in ROM

- **Template Table Address:** `0x804100+`  
- **Handler Routine Address:** `0x801B00–0x801B80`

This section is triggered after a CHECK CONDITION occurs, and the host sends an ATAPI command with opcode `0x03` (REQUEST SENSE).

---

## 🧬 Handler Routine Logic

```assembly
MOV #0x12, R1            ; Number of bytes to send (18B)
MOV.L #0x804100, R2      ; Pointer to template base
LOOP:
  MOV.B @R2+, R3         ; Read from ROM
  MOV.B R3, @R4+         ; Write to data buffer (DMA/PIO region)
  DT R1
  BF LOOP
RTS
```

- `R2` points to the beginning of the fixed SENSE buffer.
- `R4` points to the ATAPI data register buffer region (likely memory-mapped to `0xFFFFEC00`).
- Sends exactly 18 bytes (Fixed Descriptor Format).

---

## 🧬 Template Format (Fixed Format Descriptor)

```text
Byte  Field                 Value / Meaning
------------------------------------------------
0     Response Code         0x70 (current errors)
2     Sense Key             e.g., 0x02 (Not Ready)
7     Additional Length     0x0A (10 additional bytes follow)
12    ASC                   e.g., 0x3A (Medium Not Present)
13    ASCQ                  e.g., 0x00 (Qualifier)
```

📌 **Firmware injects these values from static templates in ROM.**  
They are NOT dynamically built from scratch.

---

## 🗃️ Known Templates (from ROM dump)

| Description         | Sense Key | ASC  | Address     |
|----------------------|-----------|------|-------------|
| No Error             | 0x00      | —    | `0x804100`  |
| Not Ready            | 0x02      | —    | `0x804112`  |
| Medium Error         | 0x03      | —    | `0x804124`  |
| Illegal Request      | 0x05      | —    | `0x804136`  |
| No Media Present     | 0x02      | 0x3A | `0x804148`  |

Each template is 18 bytes long and statically aligned in ROM.

---

## ⏱️ Timing and Behavior

| Event                        | Timing Requirement     |
|------------------------------|------------------------|
| CHECK CONDITION set          | After error condition  |
| Host issues REQUEST SENSE    | Within 10–50 ms        |
| DRQ asserted by firmware     | ≤250 µs after CMD      |
| Full 18B sent via PIO        | Within 1 ms            |

The firmware raises `DRQ`, fills the host buffer via PIO reads, and clears `DRQ` once complete.

---

## 🧰 ODE Emulation Strategy

1. Detect failed command that leads to CHECK CONDITION.
2. Set ATAPI Status:
   ```text
   0x51 = DRDY | CHECK CONDITION
   ```
3. Store fixed 18B descriptor in SRAM/flash or ROM.
4. On `0x03 REQUEST SENSE`:
   - Assert `DRQ`
   - Send template via PIO in response to `DIOR-`
   - Clear `DRQ` after 18B

---

## 🔎 Recommended Template Example (No Media)

```hex
70 00 02 00 00 00 00 0A 00 00 00 00 3A 00 00 00 00 00
```

➡️ Interpreted as:  
- `0x70` – Fixed format, current error  
- `0x02` – Not ready  
- `0x3A` – Medium not present (ASC)  
- `0x00` – No further qualifier

---

## 📚 References

- ROM image: `dvp-s315_v17.bin`, `0x804100+`
- SFF-8020i ATAPI Specification
- ATAPI Emulation Logic Guide
- SH7034 data sheet and mapped memory diagram
