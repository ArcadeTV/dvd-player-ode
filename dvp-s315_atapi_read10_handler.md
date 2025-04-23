# ATAPI READ(10) Sector Handler in Sony DVP-S315 Firmware

> 📘 This document analyzes the firmware routine in the Sony DVP-S315 responsible for processing `READ(10)` ATAPI commands. It covers ROM locations, data transfer behavior, DRQ/BSY signaling, and emulation tips for ODE developers.

---

## 📌 Location in ROM

- **Handler Address:** `0x802080–0x802180`
- Triggered by the ATAPI opcode `0x28` (READ(10))
- Entered via the main dispatcher routine at `0x801420`

---

## 🧠 Purpose

The handler is responsible for:
- Receiving `READ(10)` command parameters from the 12-byte ATAPI packet
- Determining the requested Logical Block Address (LBA)
- Fetching a 2048-byte sector from disc or emulated media
- Transmitting data to the host via **PIO (Programmed I/O)** using the ATA data register at `0xFFFFEC00`

---

## 🧬 Simplified Routine Logic (Pseudocode)

```c
// Assume R4 points to ATAPI packet buffer
lba = (packet[2] << 24) | (packet[3] << 16) | (packet[4] << 8) | packet[5];
sector_count = packet[7];

buffer = &sector_data[lba * 2048];

set_status(BSY = 1);
delay(100 us);

set_status(BSY = 0, DRQ = 1);

for (i = 0; i < 1024; i++) { // 2048 bytes = 1024 words
    wait_for_read_strobe();  // DIOR-
    write16(ATA_DATA_REGISTER, buffer[i]);
}

clear_status(DRQ);
```

---

## 🧩 Hardware and Timing Details

| Phase           | Action                            | Signal       | Timing Constraint |
|----------------|------------------------------------|--------------|--------------------|
| Enter Routine  | Set BSY = 1                        | BSY          | Immediate          |
| Wait ~100 µs   | Read/prepare sector                | —            | ~100 µs            |
| Ready to Send  | Clear BSY, set DRQ                 | DRQ, BSY     | ≤250 µs from entry |
| Data Transfer  | 2048 bytes via ATA DATA (PIO read) | DIOR-, D0–15 | 1–2 ms             |
| Done           | Clear DRQ                          | DRQ          | End of transfer    |

The host must read the data in 16-bit chunks from `0xFFFFEC00` while `DRQ=1`.

---

## 📚 Register Reference

| Register         | Address      | Usage                    |
|------------------|--------------|---------------------------|
| ATA DATA         | `0xFFFFEC00` | Data bus (16-bit access)  |
| ATA STATUS       | `0xFFFFEC07` | BSY/DRQ/CHECK/DRDY flags |
| ATA ALT STATUS   | `0xFFFFEC08` | Read-only status copy     |

---

## ✅ Emulation Notes for ODE Developers

1. Parse the incoming ATAPI PACKET and detect opcode `0x28`
2. Compute the LBA and fetch a corresponding 2048-byte block
3. Set `BSY=1`, simulate data load latency (~100 µs)
4. Set `DRQ=1`, clear `BSY`
5. On host DIOR-, return 16-bit data words via ATA DATA register
6. Clear `DRQ` once all 2048B transferred

### Status Flags

```text
BSY = 1      → Drive busy, cannot accept commands
DRQ = 1      → Data is ready to be read
DRDY = 1     → Device ready
CHECK = 0    → No error
```

---

## 🔎 Observations from Firmware

- The routine polls or waits for data register access during the transfer
- It does not implement retries or UDMA — strictly PIO
- Multiple sector support is limited to `sector_count` from byte 7

---

## 📚 References

- Firmware ROM: `dvp-s315_v17.bin` @ `0x802080+`
- Logic analysis from original TK-47 drive
- ATAPI Specification (SFF-8020i)
- ATA timing and signaling guide
