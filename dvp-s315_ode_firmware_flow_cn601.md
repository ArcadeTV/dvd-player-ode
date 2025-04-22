# DVP-S315 Firmware ↔ Drive (TK-47) Communication Flow (via CN601)

> 📘 This flowchart outlines the step-by-step communication between the mainboard (SH7034 CPU) and the original DVD drive mechanism (TK-47) via CN601, focusing on ATAPI command execution and status signaling. This flow is based on firmware reverse engineering, SFF-8020i protocol behavior, and CN601 electrical assignments.

---

## 📊 Communication Flow Diagram (Markdown Format)

```text
+-----------------+
| Player Boot-Up  |
+--------+--------+
         |
         v
+-------------------------+
| SH7034 configures IDE   |
| port at 0xFFFFEC00      |
| (8-bit accesses)        |
+-------------------------+
         |
         v
+-----------------------------+
| Host writes 0xA0 to        |
| 0xFFFFEC06 (Drive Select)  |
+-----------------------------+
         |
         v
+-----------------------------+
| Host issues PACKET command |
| (0xA0) via 0xFFFFEC07      |
+-----------------------------+
         |
         v
+-----------------------------+
| Host waits for DRQ         |
| (Status read: DRQ=1)       |
+-----------------------------+
         |
         v
+-----------------------------+
| Host writes 12-byte ATAPI  |
| packet to DATA register    |
| (0xFFFFEC00)               |
+-----------------------------+
         |
         v
+-----------------------------+
| Emulated drive interprets  |
| ATAPI opcode (e.g. 0x12)   |
+-----------------------------+
         |
         v
+-----------------------------+
| Drive returns status:      |
| DRQ = 1 → data ready       |
| BSY = 0                    |
+-----------------------------+
         |
         v
+-----------------------------+
| SH7034 reads from          |
| 0xFFFFEC00 in 2048B blocks |
| (sector PIO read)          |
+-----------------------------+
         |
         v
+-----------------------------+
| Drive ends with:           |
| DRQ = 0, INTRQ ↑ (optional)|
+-----------------------------+
         |
         v
+-----------------------------+
| Host polls status          |
| (CHECK CONDITION?)         |
+-----------------------------+
         |
         v
+-----------------------------+
| If needed, REQUEST SENSE   |
| issued by host             |
+-----------------------------+
         |
         v
+-----------------------------+
| Drive responds with SENSE  |
| code: 0x00 OK / 0x3A No Disc |
+-----------------------------+
         |
         v
+-----------------------------+
| UI shows: "LOADING",       |
| "NO DISC", "PLAYING", etc. |
+-----------------------------+
```

---

## 📌 Emulation Notes

- All bus-level events visible to firmware **must be emulated**:
  - Status register behavior
  - Timing of BSY/DRQ
  - Data format (12B packets, 2048B payloads)
- The firmware does **not** query mechanical signals via CN601
- `INTRQ` is optional – the firmware relies on polling

---

## 📚 Sources

- ATAPI behavior extracted from `dvp-s315_v17.bin`
- SH7034 port mapping (`0xFFFFEC00`)
- Command timing from ATAPI SFF-8020i
- Bus and line layout from `sony_dvp-s300_305_315.pdf`

---

## 🧠 **Firmware–Bus–Drive Flow (SH7034 → CN601 → TK-47)**

```yaml
[SH7034 Main CPU]
┌──────────────────────────────┐
│ A1: Boot Start (0x801000)    │
│ ↓                            │
│ A2: IDE Init Sequence        │
│ ↓                            │
│ A3: Send PACKET Command (A0) │
└──────────────┬───────────────┘
               │
               ▼
[CN601 ATAPI Bus]
┌──────────────────────────────┐
│ C1: ATA D0–D15 Lines         │◄──────────────┐
│ C2: CS0 / CS1                │               │
│ C3: DIOR / DIOW              │               │
│ C4: BSY / DRQ / INTRQ        │               │
│ C5: Device Control / Reset   │               │
└──────────────┬───────────────┘               │
               │                               │
               ▼                               │
[TK-47 Drive Unit]                             │
┌────────────────────────────────────────────┐ │
│ D1: ATA Ctrl Receives PACKET               │◄┘
│ ↓                                          │
│ D2: Decode INQUIRY / READ / MODE SENSE     │
│ ↓                                          │
│ D3: Access disc / emulated FS              │
│ ↓                                          │
│ D4: Send 2048B sector or TOC               │
│ ↓                                          │
│ D5: Signal DRQ / Clear BSY                 │
│ ↓                                          │
│ D6: If error → Return CHECK + SENSE data   │
└──────────────┬─────────────────────────────┘
               │
               ▼
[CN601 + CPU again]
┌──────────────────────────────┐
│ A4: Wait for DRQ             │◄── C4         │
│ ↓                            │               │
│ A5: Write 12B ATAPI Packet   │──→ C1 + C3    │
│ ↓                            │               │
│ A6: Wait for Drive Response  │◄── C4         │
│ ↓                            │               │
│ A7: Read data from 0xEC00    │◄── C1         │
│ ↓                            │               │
│ A8: Check SENSE / STATUS     │               │
└──────────────────────────────┘
```

## Notes

- All interactions are **memory-mapped** , originating from `0xFFFFEC00–0xFFFFEC08`
- DRQ and BSY signals are essential for read/write timing
- The drive side (TK-47 or ODE) must properly emulate:
  - Packet response phase
  - DRQ assertion on sector ready
  - SENSE data population on error

## References

- `dvp-s315_atapi_register_reference.md`
- `dvp-s315_ode_emulation_ready_sheet.md`
- Firmware section `0x801420` (ATAPI dispatcher)
