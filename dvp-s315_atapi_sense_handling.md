# ATAPI SENSE Code Handling in Sony DVP-S315 Firmware

> 🧠 This document explains the internal handling, formatting, location, and emulation relevance of ATAPI SENSE data as used by the Sony DVP-S315 DVD player firmware. The goal is to guide ODE developers in accurately reproducing real-world error reporting behavior.

---

## 🧩 What is SENSE Data?

- SENSE data is a structured response returned by an ATAPI device when a command results in a **CHECK CONDITION** (status bit 7 = 1).
- The host issues a `REQUEST SENSE` command (opcode `0x03`) to retrieve details of the error.
- The data returned is typically **18 bytes** and follows the **Fixed Format Descriptor** style as defined in SFF-8020i.

---

## 📚 SENSE Templates in Firmware

The firmware contains **predefined SENSE descriptors** in ROM, likely copied into a buffer and returned when requested by the host.

| Location     | Contents             |
|--------------|----------------------|
| `0x804100+`  | SENSE data templates |
| `0x804120+`  | Likely used for ERRORs |

Known types:

| Sense Key | Code | Description          |
|-----------|------|----------------------|
| 0x00      | —    | No Sense (no error)  |
| 0x02      | —    | Not Ready            |
| 0x03      | —    | Medium Error         |
| 0x05      | —    | Illegal Request      |
| 0x3A      | ASC  | Medium Not Present   |

---

## 🧬 SENSE Data Format (Fixed Descriptor)

| Byte | Field             | Description                           |
|------|-------------------|---------------------------------------|
| 0    | Response Code     | `0x70` = current error (fixed format) |
| 2    | Sense Key         | 0x00 = OK, 0x02 = Not Ready, etc.     |
| 7    | Additional Length | Usually `0x0A` (10 additional bytes)  |
| 12   | ASC               | Additional Sense Code (e.g., 0x3A)    |
| 13   | ASCQ              | Additional Sense Qualifier            |

Example: `No Disc Present`

```bin
70 00 00 00 00 00 00 0A 00 00 00 00 3A 00 00 00 00 00
```

---

## ⏱️ Timing & Flow

1. Host issues a command (e.g., `READ`)
2. Emulated drive detects error (e.g., disc not present)
3. Sets `BSY=1`, then `CHECK CONDITION` (Status bit 7)
4. Host issues `REQUEST SENSE` (0x03)
5. Emulated drive responds with 18-byte SENSE descriptor
6. DRQ must be raised appropriately to begin transfer

### Timing Expectations

| Phase             | Max Time (recommended) |
|-------------------|------------------------|
| SENSE Ready       | ≤250 µs after CHECK    |
| DRQ to transfer   | ≤1 ms                  |
| Transfer duration | ~100–300 µs (PIO)      |

---

## 📦 CN601 Interface Usage

All communication runs over **CN601**, the 27-pin FFC carrying:

| Signal | Direction | Description                |
|--------|-----------|----------------------------|
| D0–D15 | Bi-dir    | 16-bit data                |
| CS0/CS1| Host→ODE  | Register select            |
| DIOR   | Host→ODE  | Read strobe                |
| DIOW   | Host→ODE  | Write strobe               |
| DRQ    | ODE→Host  | Data transfer ready        |
| BSY    | ODE→Host  | Drive busy flag            |

---

## 🔧 ODE Implementation Tips

- Store standard SENSE templates in ROM or FLASH
- After error (e.g., invalid command, no disc), set:
  - `Status = 0x51` → DRDY + CHECK CONDITION
- On `0x03 REQUEST SENSE`, provide 18B fixed descriptor
- Respect DRQ/BSY timing to avoid firmware lockups

### Recommended SENSE Handling Logic

```c
if (last_command_failed) {
    status = STATUS_CHECK_CONDITION;
    prepare_sense_descriptor(0x70, 0x02, 0x3A);
}

if (command == REQUEST_SENSE) {
    status = STATUS_DRQ;
    write_to_host(sense_buffer, 18);
}
```

---

## 📚 References

- Firmware: `dvp-s315_v17.bin` @ `0x804100+`
- ATAPI Spec: SFF-8020i Revision 2.6
- CN601 Interface: `sony_dvp-s300_305_315.pdf`
- Logic traces and emulator test implementations
