# Sony DVP-S315 – Emulation Reference Sheet

| Opcode  | SCSI Command    | Required Response | Emulation Notes      |
| ------- | --------------- | ----------------- | -------------------- |
| 0x12    | INQUIRY         | 36B block         | Hardcoded OK         |
| 0x1A/5A | MODE SENSE      | 12B / 16B         | Fixed structure      |
| 0x1B    | START/STOP UNIT | Status only       | Check spin flag      |
| 0x1E    | PREVENT/ALLOW   | GOOD              | No effect            |
| 0x28    | READ(10)        | DRQ 2048B         | Sector loop          |
| 0x2F    | VERIFY          | CHECK CONDITION   | Sense: 0x05/0x20     |
| 0x55    | MODE SELECT     | GOOD              | Ignore payload       |
| 0x00    | TEST UNIT READY | GOOD or 0x3A      | Sense if disc absent |

BSY and DRQ must be asserted per SFF-8020i timing.
