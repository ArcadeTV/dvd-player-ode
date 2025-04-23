# Illegal ATAPI Opcode Handler in Sony DVP-S315 Firmware

> ❌ This document explains how the Sony DVP-S315 firmware handles invalid or unsupported ATAPI opcodes. These cases result in a `CHECK CONDITION` being raised, followed by the firmware returning a standardized SENSE response indicating an illegal request.

---

## 📌 Location in ROM

- **Handler Routine:** `0x801B80–0x801BFF`
- Reached via the central ATAPI dispatcher when:
  - Opcode is unrecognized
  - Opcode is unsupported in current state
- Links directly to a static SENSE descriptor template (`0x05 20 00`) → Illegal Request

---

## 🧠 Behavior Summary

The firmware does **not crash** or halt for unknown commands. Instead, it:

1. Logs (internally or virtually) the error
2. Sets ATA Status: `CHECK CONDITION = 1`
3. Prepares a static 18-byte SENSE descriptor
4. Awaits `REQUEST SENSE` (opcode `0x03`) from the host

---

## 🧬 Disassembly Flow (Simplified)

```assembly
MOV #0x12, R1              ; 18 bytes of sense data
MOV.L #0x804180, R2        ; Pointer to illegal request SENSE template
MOV.L #0xFFFFEC00, R4      ; PIO data region

LOOP:
  MOV.B @R2+, R3
  MOV.B R3, @R4+
  DT R1
  BF LOOP

RTS
```

---

## ⚠️ Sense Response: Illegal Request

### Fixed Format SENSE Descriptor

```hex
70 00 05 00 00 00 00 0A 00 00 00 00 20 00 00 00 00 00
```

| Field     | Value | Meaning                         |
|-----------|-------|----------------------------------|
| Byte 0    | 0x70  | Fixed format, current error      |
| Byte 2    | 0x05  | Sense Key: Illegal Request       |
| Byte 7    | 0x0A  | 10 additional bytes follow       |
| Byte 12   | 0x20  | ASC: Invalid Command Operation   |
| Byte 13   | 0x00  | ASCQ: None                       |

---

## 📦 Trigger Conditions

| Opcode Sent | Result                             |
|-------------|-------------------------------------|
| Unknown     | CHECK CONDITION + 0x05/0x20         |
| Unsupported (e.g. `WRITE(10)`) | Same             |
| Malformed PACKET | Same (depends on integrity)    |

---

## 🧰 ODE Emulation Guidelines

When receiving an unsupported ATAPI command:

1. Respond with status:
   ```text
   0x51 = DRDY | CHECK CONDITION
   ```
2. Store the illegal SENSE descriptor internally
3. Wait for `0x03 REQUEST SENSE`
4. Respond with 18B from `0x804180` template

Optional:

- Log unknown opcode to debug trace
- Support minimal diagnostics in your emulator for dev mode

---

## 🧠 Notes on Robustness

- The routine does **not analyze the opcode value** beyond validation failure
- No dynamic error context is included (e.g., LBA or command ID)
- Always returns a generic illegal request code
- Does not retry or offer recovery path

---

## 📚 References

- Firmware: `dvp-s315_v17.bin` @ `0x801B80` and `0x804180`
- ATAPI Specification (SFF-8020i)
- Real-world logic traces during invalid command injection
