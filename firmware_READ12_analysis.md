
# 📄 Firmware Analysis: `READ(12)` (Opcode `0xA8`) – DVP-S315

---

## 🔍 Summary

- **Command:** ATAPI `READ(12)`  
- **Opcode:** `0xA8`  
- **ROM Entry Point:** `0x8021C0`  
- **Purpose:** Read one or more 2048-byte logical sectors from a specified LBA  
- **Behavior:** Identical in flow to `READ(10)`, but supports 32-bit block counts  

---

## 🧩 Dispatcher Overview

- The ATAPI command dispatcher begins around **ROM `0x800400`**.
- Uses opcode-based branching:
  - Either via a jump table or sequential comparisons
  - Each ATAPI opcode is dispatched to its dedicated handler
- For `0xA8`, the code pattern was:
  ```asm
  cmp.b   #0xA8, rX
  bf      not_this
  mov.l   @(offset, r0), r0
  jmp     @r0
  ```
- Execution jumps to `0x8021C0` for `READ(12)`

---

## 🔧 `READ(12)` Handler Breakdown – `0x8021C0`

### ✔️ Key Functionality

- Parse:
  - **LBA** from `cmd[2..5]`
  - **Transfer Length** (Block Count) from `cmd[6..9]` (4 bytes)
- For each block:
  1. Call low-level disk reader (`READ_SECTOR`)
  2. Wait for host to assert IO Read
  3. Set **DRQ** flag
  4. Serve 2048-byte buffer
  5. Clear DRQ
  6. Decrement block count
  7. Repeat

### ⏱ Timing Behavior

- No DMA
- Data sent per-sector using polling + interruptable DRQ logic
- DRQ cleared after read complete, not automatically

---

## 🔁 Pseudocode Summary

```c
uint32_t lba = (cmd[2] << 24) | (cmd[3] << 16) | (cmd[4] << 8) | cmd[5];
uint32_t count = (cmd[6] << 24) | (cmd[7] << 16) | (cmd[8] << 8) | cmd[9];

while (count > 0) {
    read_sector(lba, buffer);
    set_DRQ_flag();
    wait_for_host_read(buffer, 2048);
    clear_DRQ_flag();
    lba++;
    count--;
}
```

---

## 🧠 Emulation Strategy for ODE

### Must Emulate

- DRQ state machine
- LBA-to-buffer map
- 2048-byte block transfers
- Exact command structure: 12-byte packet

### Timing

- DRQ must only assert **after data is ready**
- Wait for host read signal before proceeding to next block

### Difference vs. `READ(10)`

| Feature            | `READ(10)`         | `READ(12)`         |
|--------------------|--------------------|--------------------|
| Opcode             | `0x28`             | `0xA8`             |
| Block Count Width  | 2 bytes            | 4 bytes            |
| Addressing         | 28-bit             | 32-bit             |
| Firmware Handler   | `0x802080`         | `0x8021C0`         |

---

## 📎 References

- Source firmware: `dvp-s315_v17_swapped.bin`
- Analyzed region: `0x800400–0x800900`
- Tooling: SH-1 static analysis, pattern recognition, pointer trace
