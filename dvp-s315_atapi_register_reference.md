# IDE/ATAPI Register Access Table – DVP-S315

> 📚 This table summarizes the memory-mapped register interface used by the DVP-S315 DVD player to communicate with the IDE/ATAPI drive.

## I/O Address Base

All registers are mapped starting at:

```text
0xFFFFEC00 (Primary Channel – IDE/ATAPI)
```

## Register Map

| Address      | Register Name                     | Direction                | Size   | Description                                        |
| ------------ | --------------------------------- | ------------------------ | ------ | -------------------------------------------------- |
| `0xFFFFEC00` | **Data Register**                 | R/W                      | 16-bit | Data buffer for PIO transfers (2048B sector)       |
| `0xFFFFEC01` | Error / Features                  | R (Error) / W (Features) | 8-bit  | Error code or transfer feature setup               |
| `0xFFFFEC02` | Sector Count                      | R/W                      | 8-bit  | Number of sectors to read/write (ignored in ATAPI) |
| `0xFFFFEC03` | Sector Number / LBA0              | R/W                      | 8-bit  | Command param or ATAPI packet phase                |
| `0xFFFFEC04` | Cylinder Low / LBA1               | R/W                      | 8-bit  | Command param or ATAPI signature byte 0 (0x14)     |
| `0xFFFFEC05` | Cylinder High / LBA2              | R/W                      | 8-bit  | Command param or ATAPI signature byte 1 (0xEB)     |
| `0xFFFFEC06` | Drive/Head Select                 | R/W                      | 8-bit  | Selects master drive (0xA0 or 0xB0)                |
| `0xFFFFEC07` | Command (W) / Status (R)          | R/W                      | 8-bit  | Write command opcode (e.g. 0xA0), read status      |
| `0xFFFFEC08` | Alternate Status / Device Control | R/W                      | 8-bit  | Soft reset, interrupt disable (bit 1 = nIEN)       |

## Typical Write Sequence (e.g., PACKET Command)

```text
MOV.B  #0xA0, @(0xFFFFEC06)  ; Select device
MOV.B  #0x00, @(0xFFFFEC01)  ; Features = 0
MOV.B  #0x00, @(0xFFFFEC02)  ; Sector count
MOV.B  #0x00, @(0xFFFFEC03)  ; Sector number
MOV.B  #0x00, @(0xFFFFEC04)  ; Cylinder low
MOV.B  #0x00, @(0xFFFFEC05)  ; Cylinder high
MOV.B  #0xA0, @(0xFFFFEC07)  ; Command = PACKET
```

Followed by:

```text
Wait for DRQ → Write 12-byte ATAPI packet via 0xFFFFEC00
```

## ATAPI Identification Signature (Read Only)

| Address      | Value  | Meaning                   |
| ------------ | ------ | ------------------------- |
| `0xFFFFEC04` | `0x14` | ATAPI signature low byte  |
| `0xFFFFEC05` | `0xEB` | ATAPI signature high byte |

## Control Register (0xFFFFEC08)

| Bit | Name     | Description                        |
| --- | -------- | ---------------------------------- |
| 0   | SRST     | Soft Reset (1 = assert)            |
| 1   | nIEN     | Interrupt disable (1 = mask INTRQ) |
| 7   | Reserved | -                                  |

---

## ODE Developer Notes

- DRQ must be asserted by emulation during PIO read/write
- STATUS must clear BSY and set DRQ at proper time
- SENSE and PACKET phases must be handled after command phase
- INTRQ optional for DVP-S315 (polling used)
