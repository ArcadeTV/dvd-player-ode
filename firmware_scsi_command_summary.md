
# 📄 ATAPI/SCSI Command Support – Sony DVP-S315 Firmware

---

## ✅ Confirmed Implemented Commands

| Opcode | Command          | ROM Address (if known)  | Status | Emulation Notes                           |
| ------ | ---------------- | ----------------------- | ------ | ----------------------------------------- |
| `00h`  | TEST UNIT READY  | Dispatcher only         | ✔️   | No-op, confirms device presence           |
| `03h`  | REQUEST SENSE    | 0x801B00                | ✔️   | Static response, must be emulated         |
| `12h`  | INQUIRY          | (in dispatcher)         | ✔️   | Returns standard device info              |
| `1Ah`  | MODE SENSE(6)    | (in dispatcher)         | ✔️   | Limited implementation                    |
| `1Bh`  | START STOP UNIT  | (not fully analyzed)    | ✔️   | May be stubbed                            |
| `25h`  | READ CAPACITY    | (analyzed)              | ✔️   | Returns last LBA and block size           |
| `28h`  | READ(10)         | 0x802080                | ✔️   | Fully implemented, per-sector read        |
| `43h`  | READ TOC         | 0x801C00                | ✔️   | Parses TOC format, returns sector mapping |
| `45h`  | PLAY AUDIO(10)   | ≈ 0x802200              | ✔️   | Starts playback at LBA, emulatable        |
| `47h`  | PLAY AUDIO MSF   | ≈ 0x8022C0              | ✔️   | MSF to LBA conversion                     |
| `4Eh`  | STOP PLAY/SCAN   | ≈ 0x8023A0              | ✔️   | Stops audio subsystem                     |
| `A8h`  | READ(12)         | 0x8021C0                | ✔️   | Fully documented above                    |
| `BBh`  | SET CD SPEED     | (dispatched, many refs) | ✔️   | Affects read delay only                   |
| `E5h`  | Check Power Mode | (proprietary path)      | ✔️   | Must return "active", Sony-specific       |

---

## 🧪 Unknown / Sony-Extended Commands

| Opcode | Command            | Firmware Presence | Notes                      |
| ------ | ------------------ | ----------------- | -------------------------- |
| `E5h`  | Power Mode Check   | ✔️ many times   | Likely Sony-specific       |
| `BDh`  | MECHANISM STATUS   | 🔍 pending       | Possibly used, unconfirmed |
| `ADh`  | READ DVD STRUCTURE | 🔍 maybe         | Partially matched          |

---

## ❌ Not Found or Stubbed Commands

| Opcode | Command               | Notes                |
| ------ | --------------------- | -------------------- |
| `2Ah`  | WRITE(10)             | No trace in firmware |
| `35h`  | SYNCHRONIZE CACHE(10) | Not referenced       |
| `5Bh`  | CLOSE TRACK/SESSION   | Not present          |
| `A2h`  | SEND EVENT            | Not referenced       |
| `BFh`  | SEND DVD STRUCTURE    | Not in firmware      |

---

## 📊 Summary Table: SFF-8020i / MMC Commands Crosscheck

Based on extracted drive command table from external Sony source. Not validated for DVP-S315 but cross-referenced:

- ✔️ Implemented: 15+
- ❌ Not present: 10+
- 🧪 Possibly extended: 3

---

## 📌 Emulation Priorities

| Priority | Opcode                            | Reason                            |
| -------- | --------------------------------- | --------------------------------- |
| HIGH     | `28h`, `A8h`, `43h`, `03h`, `12h` | Required for basic DVD operation  |
| MEDIUM   | `45h`, `47h`, `4Eh`               | For audio track emulation         |
| LOW      | `E5h`, `BBh`, `25h`               | For polish and full compatibility |

---
