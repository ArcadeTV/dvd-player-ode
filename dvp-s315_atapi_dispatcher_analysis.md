# ATAPI Command Dispatcher in Sony DVP-S315 Firmware

> 🧠 This document analyzes the central ATAPI command routing routine found in the DVP-S315 firmware (ROM). It serves as the gateway for all incoming ATAPI packet processing and command handling.

---

## 📌 Location in ROM

- **Dispatcher Routine Address Range:** `0x801420–0x801580`
- Identified by analyzing opcode dispatch behavior and indirect jumps via a handler table
- Matches entrypoint triggered by ATAPI `PACKET` command (0xA0)

---

## 🧠 Function Overview

This routine acts as a **command switchboard** for the ATAPI interface. It receives the decoded 12-byte PACKET, extracts the opcode byte (`[0]`), and uses a jump table or chained comparisons to dispatch the correct handler.

---

## 🧬 Disassembly Pattern

```assembly
MOV.B @R4, R0        ; Load ATAPI opcode
CMP/EQ #0x12, R0     ; INQUIRY?
BF skip_inquiry
JSR @0x801600        ; Call INQUIRY handler
...

CMP/EQ #0x28, R0     ; READ(10)?
BF skip_read
JSR @0x802080        ; Call READ(10) handler
...

CMP/EQ #0x03, R0     ; REQUEST SENSE?
JSR @0x801B00        ; Call REQUEST SENSE handler
...
```

---

## 🔁 Opcode-to-Handler Mapping

| ATAPI Opcode | Command          | Handler Address | Notes |
|--------------|------------------|------------------|-------|
| `0x12`       | INQUIRY          | `0x801600`       | Returns identity data |
| `0x28`       | READ(10)         | `0x802080`       | Transfers 2048B sector |
| `0x03`       | REQUEST SENSE    | `0x801B00`       | Returns SENSE descriptor |
| `0x1B`       | START STOP UNIT  | `0x801980`       | Tray control (Open/Close) |
| `0x00`       | TEST UNIT READY  | `0x801940`       | Readiness poll (no DRQ) |
| `0x1A`       | MODE SENSE(6)    | `0x801A20`       | Returns geometry/config |
| `0x43`       | READ TOC         | `0x801C00`       | Returns TOC sector |
| `Other`      | Unknown/illegal  | `0x801B80`       | Triggers SENSE 0x05 (Illegal Request) |

---

## ⏱️ Timing Relevance

- Dispatcher is entered immediately after PACKET reception via `0xA0` command
- Opcode is decoded **within ~10 µs**
- DRQ set by dispatcher handler only **if required by command**
- Delays here directly affect responsiveness of:
  - INQUIRY
  - SENSE
  - TOC access
  - READ(10)

---

## 🧰 ODE Implementation Notes

| Step | Required ODE Behavior |
|------|------------------------|
| Receive 12-byte ATAPI PACKET | Parse Opcode byte `[0]` |
| Emulate state transitions    | Respond with `DRQ=1` only for data-returning commands |
| Prepare matching responses   | See per-command response guides (INQUIRY, SENSE, etc.) |
| Illegal Opcode Handling      | Respond with SENSE: `0x70 05 20 00...` (Invalid Command) |

---

## 🧠 Summary

- This routine is the primary **decision point** for all ATAPI requests
- Allows accurate timing and opcode classification for ODE firmware
- Full understanding of this dispatcher allows implementation of a **clean, modular emulator core**

---

## 📚 References

- ROM image: `dvp-s315_v17.bin`
- Disassembly cross-reference with Ghidra
- ATAPI spec (SFF-8020i)
- Existing logic captures (READ, INQUIRY, TOC)
