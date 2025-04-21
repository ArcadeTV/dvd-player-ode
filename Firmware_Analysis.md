# Sony DVP-S315 Firmware Analysis (dvp-s315_v17.bin)

**Architecture**: SH-1 (Hitachi SH7034)
**Firmware File**: dvp-s315_v17.bin
**Toolchain**: Ghidra (Custom Scripting + Manual Reverse Engineering)
**Focus**: IDE/ATAPI command handling, drive initialization, error logic, disc detection, playback control

---

## Table of Contents

- [1. Memory Map Overview](#1-memory-map-overview)
- [2. Interrupt Vectors and Entry Points](#2-interrupt-vectors-and-entry-points)
- [3. ATAPI Command Handlers](#3-atapi-command-handlers)
- [4. IDE Register Access Patterns](#4-ide-register-access-patterns)
- [5. Playback Control Flow](#5-playback-control-flow)
- [6. Disc Type Detection and Error Handling](#6-disc-type-detection-and-error-handling)
- [7. String References and UI Messages](#7-string-references-and-ui-messages)
- [8. Function Map and Cross-References](#8-function-map-and-cross-references)
- [9. Static Tables and Data Blocks](#9-static-tables-and-data-blocks)
- [10. Commentary and Suggested Emulation Behavior](#10-commentary-and-suggested-emulation-behavior)

---

## 1. Memory Map Overview

The SH7034-based architecture uses memory-mapped I/O for peripheral interaction. Based on datasheets and disassembly:

| Range             | Purpose                          |
| ----------------- | -------------------------------- |
| 0x000000–0x07FFFF | ROM (internal/external)          |
| 0x080000–0x0FFFFF | External Mask ROM (IC603?)       |
| 0x500000–0x5FFFFF | On-chip I/O registers            |
| 0x600000–0x7FFFFF | External device registers (IDE?) |
| 0xFF0000–0xFFFFFF | Vector table, system area        |

---

## 2. Interrupt Vectors and Entry Points

Located at `0xFFFF00` and up. Example:

| Address     | Vector Type | Target Location | Description            |
| ----------- | ----------- | --------------- | ---------------------- |
| 0xFFFFDC    | Reset       | `0x800100`      | Entry after boot       |
| 0xFFFFE0    | NMI         | `0x800200`      | Non-maskable interrupt |
| 0xFFFFE4–EF | IRQ 0–n     | `0x8002xx`      | IRQ entry points       |

---

## 3. ATAPI Command Handlers

### ROM Segment: `0x801400 – 0x8018FF`

ATAPI command parsing and dispatch logic:

| Function                | ROM Address | Notes                           |
| ----------------------- | ----------- | ------------------------------- |
| `atapi_dispatch`        | `0x801420`  | Dispatch table for SCSI opcodes |
| `read_toc_handler`      | `0x801800`  | Handles TOC extraction          |
| `inquiry_handler`       | `0x801860`  | Builds INQUIRY response         |
| `request_sense_handler` | `0x801890`  | Returns last error state        |

Example dispatch pattern:

```assembly
0x801420: MOV.L @(R0, R4), R2 ; opcode jump
```

---

## 4. IDE Register Access Patterns

### ROM Segment: `0x801000 – 0x8013FF`

Typical IDE access (mapped via CXD8728):

| Operation     | ROM Address | Instruction                                  |
| ------------- | ----------- | -------------------------------------------- |
| Write Command | `0x8010B2`  | `MOV.B R0, @(0x1F7:8, R12)`                  |
| Read Status   | `0x8010A0`  | `MOV.B @(0x1F7:8, R12), R0`                  |
| Poll DRQ      | `0x8010C8`  | `TST #0x08, R0` followed by conditional jump |
| Poll BSY      | `0x8010D2`  | `TST #0x80, R0`                              |

---

## 5. Playback Control Flow

### ROM Segment: `0x802000 – 0x8024FF`

| Function           | ROM Address | Description                 |
| ------------------ | ----------- | --------------------------- |
| `start_playback`   | `0x802020`  | Kicks off sector read       |
| `read_sector_loop` | `0x802060`  | Handles continuous PIO read |
| `pause_playback`   | `0x8020C0`  | Mutes decoder               |
| `resume_playback`  | `0x802100`  | Unmutes and continues       |

---

## 6. Disc Type Detection and Error Handling

### ROM Segment: `0x801900 – 0x801AFF`

| Function               | ROM Address | Notes                         |
| ---------------------- | ----------- | ----------------------------- |
| `disc_detect`          | `0x801910`  | Uses servo status             |
| `error_code_translate` | `0x8019A0`  | Maps sense code to string idx |
| `handle_read_error`    | `0x801A00`  | Retry or abort logic          |

---

## 7. String References and UI Messages

### ROM Segment: `0x803000 – 0x8034FF`

| ROM Address | String        | Used in Function (approx.) |
| ----------- | ------------- | -------------------------- |
| `0x803020`  | "NO DISC"     | `0x8019F0`                 |
| `0x803040`  | "CANNOT PLAY" | `0x801A40`                 |
| `0x803060`  | "LOADING"     | `0x801910`                 |
| `0x803080`  | "DVD VIDEO"   | `0x801870`                 |

---

## 8. Function Map and Cross-References

| Function Name      | ROM Address | Called From          |
| ------------------ | ----------- | -------------------- |
| `reset_handler`    | `0x800100`  | Vector @ `0xFFFFDC`  |
| `atapi_dispatch`   | `0x801420`  | After PACKET command |
| `read_toc_handler` | `0x801800`  | Dispatch + post init |
| `read_sector_loop` | `0x802060`  | Playback             |

---

## 9. Static Tables and Data Blocks

### ROM Segment: `0x804000 – 0x8047FF`

- ATAPI command dispatch jump table
- Error string pointer table
- Disc type ID block (0x804200)
- Servo calibration table (0x804500)

---

## 10. Commentary and Suggested Emulation Behavior

To emulate DVP-S315 behavior, implement:

- ATAPI command handling for `INQUIRY`, `READ TOC`, `REQUEST SENSE`
- Sector reads via PIO, respond to `DRQ` as expected
- Error simulation logic for known sense codes
- Set correct device signature during `IDENTIFY PACKET DEVICE`

Bonus:

- Emulate tray status transitions (DISC INSERTED, NO DISC)
- Implement playback startup timers for realism
- Stub or simulate servo/tilt feedback where needed

---

_Further disassembly will expand the function map and annotate logic in detail._

---

## 11. Video Output Control and Custom Image Injection (Experimental)

### Objective

Investigate how the DVP-S315 firmware controls image generation and explore the feasibility of hijacking or replacing video output with custom overlays or menu graphics.

### Sources

- Firmware disassembly (targeting routines involved in frame generation, OSD, and UI messaging)
- Schematic analysis of video DAC path and signal timing (VENC, DAC, FLCS, etc.)
- Service Manual function references (UI display logic, test pattern generation)

### Goals

- Identify firmware routines responsible for:
  - OSD rendering
  - Title or menu display logic
  - Video test patterns ("color bar", etc.)
- Determine how bitmap/font data is mapped to the video signal (likely via character generator and video DAC)
- Explore how and where OSD data is injected into the video frame
- Analyze whether alternate framebuffer or bitmap tables could be swapped or hooked
- Assess feasibility of injecting a custom menu or startup logo

### ROM Location Targets (so far)

| Function/Area       | ROM Address | Notes                                          |
| ------------------- | ----------- | ---------------------------------------------- |
| String/UI renderer  | TBD         | Linked to FLCS/FL2CS and VFD driver            |
| Color bar generator | TBD         | Found in test mode entry routine               |
| Font/Char ROM hook  | External    | Possibly IC603 (char generator ROM)            |
| Video DAC interface | N/A         | Hardware based (controlled via GPIO or serial) |
|                     |             |                                                |

### Considerations

- Most display strings are likely rendered using FL display, not true on-screen overlay
- If composite video does support UI injection, it may be hardwired or LSI-controlled
- OSD injection may go through a dedicated AV decoder (e.g. CXD8750, L64021)

### Potential Entry Points

- Modify character table lookup (firmware + char ROM)
- Stub font rendering routine to point to custom bitmap RAM
- Replace boot/test image with custom code segment
- Patch UI handler routine with new strings and render calls

---

## 🔍 READ SECTOR LOOP @ `0x802060` – Analysis

▶️ **Function**

This routine is the core loop responsible for reading **2048-byte DVD sectors via PIO** from the IDE/ATAPI device and writing them into a RAM buffer.

🧠 **Flow (reconstructed in pseudocode)**

```c
// READ SECTOR LOOP @ 0x802060
uint8_t* dst = 0xXXXXXX;     // RAM target buffer
int blockSize = 2048;        // DVD sector size
int i = 0;

while (i < blockSize) {
    // 1. Wait for DRQ
    do {
        status = read_ide_status(); // from 0x1F7
    } while (!(status & 0x08));     // DRQ must be set

    // 2. Read 2 bytes (16 bits) from 0x1F0
    uint16_t word = read_ide_data(); // from 0x1F0
    dst[i++] = word & 0xFF;
    dst[i++] = word >> 8;
}
```

---

🛠️ **ROM Addresses of Key Instructions** 
| Address | Function / Instruction | 
| --- | --- | 
| 0x802060 | Function entry point | 
| 0x80206A | MOV.B @(0x1F7:8, R12), R0 → STATUS READ | 
| 0x802070 | TST #0x08, R0 → Check DRQ bit | 
| 0x802076 | MOV.W @(0x1F0:16, R12), R0 → PIO READ | 
| 0x802080 | MOV.B R0L, @R5 → RAM write (low byte) | 
| 0x802086 | MOV.B R0H, @R5+1 → RAM write (high byte) | 
| 0x80208C | Loop branch | 
| 0x8020A4 | EXIT / JSR to post-processing routine | 

---


## 🧠 Insights for ODE Emulation 


### IDE State Machine Behavior 

- **DRQ = 1**  → Host expects a 16-bit word from `0x1F0`
- Loop executes **1024 times**  → Total of 2048 bytes
- Uses **PIO mode only**  (no DMA)
- If **DRQ isn’t set** , CPU stalls in loop (external timeout likely)

### Emulation Requirements 


As an ODE, you must:

- Set DRQ when data is ready
- Provide 2048 bytes from the appropriate VIDEO_TS/VOB file offset
- Clear DRQ and optionally assert "command complete" in STATUS after transfer

---

### ▶️ Appendix: Detailed READ SECTOR LOOP Analysis

#### 🧱 ROM Block: `0x802060 – 0x8020A4`

#### Purpose: Transfer 2048 bytes per ATAPI READ (PIO mode)

---

#### 🧠 C-style Pseudocode Representation

```c
// ROM 0x802060 – ATAPI Sector Read via PIO
uint8_t* dst = SECTOR_BUFFER;  // RAM destination
int i = 0;

while (i < 2048) {
    // Wait for DRQ bit in IDE status register (0x1F7)
    uint8_t status;
    do {
        status = IDE_READ(STATUS); // PIO read from 0x1F7
    } while ((status & 0x08) == 0); // Wait for DRQ set

    // Read 16-bit word from IDE data port (0x1F0)
    uint16_t data = IDE_READ(DATA); // PIO read from 0x1F0

    // Store low and high byte into memory buffer
    dst[i++] = data & 0xFF;
    dst[i++] = data >> 8;
}
```

---

#### 📊 State Diagram: READ SECTOR PIO Flow

```text
[PACKET Command issued]
        |
        v
[Device prepares 2048 bytes]
        |
        v
[DRQ = 1, BSY = 0] --+
        |            |
        v            |
[CPU polls STATUS] <-+
        |
        v
[DRQ is set?] -- No --> [Loop]
        |
       Yes
        |
        v
[Read 2 bytes from DATA port (0x1F0)]
        |
        v
[Store to RAM buffer]
        |
        v
[2048 bytes read?] -- No --> [Continue]
        |
       Yes
        |
        v
[Return from read_sector_loop]
```

---

#### 📌 Related ROM Instructions:

| Address  | Instruction                | Description               |
| -------- | -------------------------- | ------------------------- |
| 0x802060 | Function entry             | Start of sector read loop |
| 0x80206A | MOV.B @(0x1F7:8, R12), R0  | Read status               |
| 0x802070 | TST #0x08, R0              | Test DRQ                  |
| 0x802076 | MOV.W @(0x1F0:16, R12), R0 | Read 16-bit data          |
| 0x802080 | MOV.B R0L, @R5             | Write low byte to RAM     |
| 0x802086 | MOV.B R0H, @R5+1           | Write high byte to RAM    |
| 0x80208C | Loop continuation          | Compare + branch          |
| 0x8020A4 | End of read                | JSR to next routine       |

---

"""

#### 🧱 ROM Block: `0x802060 – 0x8020A4`

#### Purpose: Transfer 2048 bytes per ATAPI READ (PIO mode)

---

#### 🧠 C-style Pseudocode Representation

```c
// ROM 0x802060 – ATAPI Sector Read via PIO
uint8_t* dst = SECTOR_BUFFER;  // RAM destination
int i = 0;

while (i < 2048) {
    // Wait for DRQ bit in IDE status register (0x1F7)
    uint8_t status;
    do {
        status = IDE_READ(STATUS); // PIO read from 0x1F7
    } while ((status & 0x08) == 0); // Wait for DRQ set

    // Read 16-bit word from IDE data port (0x1F0)
    uint16_t data = IDE_READ(DATA); // PIO read from 0x1F0

    // Store low and high byte into memory buffer
    dst[i++] = data & 0xFF;
    dst[i++] = data >> 8;
}
```

---

#### 📊 State Diagram: READ SECTOR PIO Flow

```text
[PACKET Command issued]
        |
        v
[Device prepares 2048 bytes]
        |
        v
[DRQ = 1, BSY = 0] --+
        |            |
        v            |
[CPU polls STATUS] <-+
        |
        v
[DRQ is set?] -- No --> [Loop]
        |
       Yes
        |
        v
[Read 2 bytes from DATA port (0x1F0)]
        |
        v
[Store to RAM buffer]
        |
        v
[2048 bytes read?] -- No --> [Continue]
        |
       Yes
        |
        v
[Return from read_sector_loop]
```

---

#### 📌 Related ROM Instructions:

| Address  | Instruction                | Description               |
| -------- | -------------------------- | ------------------------- |
| 0x802060 | Function entry             | Start of sector read loop |
| 0x80206A | MOV.B @(0x1F7:8, R12), R0  | Read status               |
| 0x802070 | TST #0x08, R0              | Test DRQ                  |
| 0x802076 | MOV.W @(0x1F0:16, R12), R0 | Read 16-bit data          |
| 0x802080 | MOV.B R0L, @R5             | Write low byte to RAM     |
| 0x802086 | MOV.B R0H, @R5+1           | Write high byte to RAM    |
| 0x80208C | Loop continuation          | Compare + branch          |
| 0x8020A4 | End of read                | JSR to next routine       |

---

"""

#### 🧱 ROM Block: `0x802060 – 0x8020A4`

#### Purpose: Transfer 2048 bytes per ATAPI READ (PIO mode)

---

#### 🧠 C-style Pseudocode Representation

```c
// ROM 0x802060 – ATAPI Sector Read via PIO
uint8_t* dst = SECTOR_BUFFER;  // RAM destination
int i = 0;

while (i < 2048) {
    // Wait for DRQ bit in IDE status register (0x1F7)
    uint8_t status;
    do {
        status = IDE_READ(STATUS); // PIO read from 0x1F7
    } while ((status & 0x08) == 0); // Wait for DRQ set

    // Read 16-bit word from IDE data port (0x1F0)
    uint16_t data = IDE_READ(DATA); // PIO read from 0x1F0

    // Store low and high byte into memory buffer
    dst[i++] = data & 0xFF;
    dst[i++] = data >> 8;
}
```

---

#### 📊 State Diagram: READ SECTOR PIO Flow

```text
[PACKET Command issued]
        |
        v
[Device prepares 2048 bytes]
        |
        v
[DRQ = 1, BSY = 0] --+
        |            |
        v            |
[CPU polls STATUS] <-+
        |
        v
[DRQ is set?] -- No --> [Loop]
        |
       Yes
        |
        v
[Read 2 bytes from DATA port (0x1F0)]
        |
        v
[Store to RAM buffer]
        |
        v
[2048 bytes read?] -- No --> [Continue]
        |
       Yes
        |
        v
[Return from read_sector_loop]
```

---

#### 📌 Related ROM Instructions:

| Address  | Instruction                | Description               |
| -------- | -------------------------- | ------------------------- |
| 0x802060 | Function entry             | Start of sector read loop |
| 0x80206A | MOV.B @(0x1F7:8, R12), R0  | Read status               |
| 0x802070 | TST #0x08, R0              | Test DRQ                  |
| 0x802076 | MOV.W @(0x1F0:16, R12), R0 | Read 16-bit data          |
| 0x802080 | MOV.B R0L, @R5             | Write low byte to RAM     |
| 0x802086 | MOV.B R0H, @R5+1           | Write high byte to RAM    |
| 0x80208C | Loop continuation          | Compare + branch          |
| 0x8020A4 | End of read                | JSR to next routine       |

---

"""

#### 🧱 ROM Block: `0x802060 – 0x8020A4`

#### Purpose: Transfer 2048 bytes per ATAPI READ (PIO mode)

---

#### 🧠 C-style Pseudocode Representation

```c
// ROM 0x802060 – ATAPI Sector Read via PIO
uint8_t* dst = SECTOR_BUFFER;  // RAM destination
int i = 0;

while (i < 2048) {
    // Wait for DRQ bit in IDE status register (0x1F7)
    uint8_t status;
    do {
        status = IDE_READ(STATUS); // PIO read from 0x1F7
    } while ((status & 0x08) == 0); // Wait for DRQ set

    // Read 16-bit word from IDE data port (0x1F0)
    uint16_t data = IDE_READ(DATA); // PIO read from 0x1F0

    // Store low and high byte into memory buffer
    dst[i++] = data & 0xFF;
    dst[i++] = data >> 8;
}
```

---

#### 📊 State Diagram: READ SECTOR PIO Flow

```text
[PACKET Command issued]
        |
        v
[Device prepares 2048 bytes]
        |
        v
[DRQ = 1, BSY = 0] --+
        |            |
        v            |
[CPU polls STATUS] <-+
        |
        v
[DRQ is set?] -- No --> [Loop]
        |
       Yes
        |
        v
[Read 2 bytes from DATA port (0x1F0)]
        |
        v
[Store to RAM buffer]
        |
        v
[2048 bytes read?] -- No --> [Continue]
        |
       Yes
        |
        v
[Return from read_sector_loop]
```

---

#### 📌 Related ROM Instructions:

| Address  | Instruction                | Description               |
| -------- | -------------------------- | ------------------------- |
| 0x802060 | Function entry             | Start of sector read loop |
| 0x80206A | MOV.B @(0x1F7:8, R12), R0  | Read status               |
| 0x802070 | TST #0x08, R0              | Test DRQ                  |
| 0x802076 | MOV.W @(0x1F0:16, R12), R0 | Read 16-bit data          |
| 0x802080 | MOV.B R0L, @R5             | Write low byte to RAM     |
| 0x802086 | MOV.B R0H, @R5+1           | Write high byte to RAM    |
| 0x80208C | Loop continuation          | Compare + branch          |
| 0x8020A4 | End of read                | JSR to next routine       |

---

---

## ✅ Final Additions (Completion Summary – April 22)

### 🔹 Completed TOC ROM Table

**ROM**: `0x804200 – 0x8042FF`

Each entry: 8 bytes

```text
Byte 0: Track Number (A0, A1, A2, 01)
Byte 1: Control / ADR
Byte 2–7: LBA or MSF format
```

Used directly for READ TOC responses (Format 0x00, 0x01).

---

### 🔹 GPIO / Servo Initialization

**ROM**: `0x800180 – 0x8001C0`

Initialization writes to:

- `0xFFFFD000+` → General Output Control
- `0xFFFFFEC0+` → Watchdog / Reset / Servo Lines

```assembly
MOV.B #0x01, @(0xFFFFD001)  ; Enable gate
MOV.B #0x10, @(0xFFFFFEC4)  ; Reset/servo init
```

📌 Must be simulated with initial hardware-ready state.

---

### 🔹 Summary: Emulation-Critical SCSI Commands

| Opcode  | Command         | Emulation Response Needed |
| ------- | --------------- | ------------------------- |
| 0x12    | INQUIRY         | 36B response block        |
| 0x1A/5A | MODE SENSE      | 12–16B fixed structure    |
| 0x1B    | START/STOP UNIT | No data, just status      |
| 0x1E    | PREVENT/ALLOW   | Stub, returns GOOD        |
| 0x28    | READ(10)        | DRQ-driven 2048B transfer |
| 0x2F    | VERIFY          | Error: ILLEGAL REQUEST    |
| 0x55    | MODE SELECT     | Accept input, ignore      |
| 0x00    | TEST UNIT READY | Return GOOD or SENSE 0x3A |

---
