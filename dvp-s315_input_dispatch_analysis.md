# Input Dispatch Behavior – IR Remote & Front Panel (Sony DVP-S315)

> 🎮 This document details how the Sony DVP-S315 DVD player firmware processes user inputs from the infrared remote control and the front panel keys. It includes information on signal paths, firmware logic, ROM addresses, and potential points for ODE system injection or debugging.

---

## 📡 1. Infrared Remote Input (IR via CN101)

### Hardware Overview (Infrared Remote Input)

- IR receiver module is connected via connector **CN101**
- Outputs a **pulse-width modulated signal** decoded using the NEC IR protocol
- Signal is routed to the **external interrupt pin** on the SH7034 microcontroller (`IRQ3` or `IRQ5`)

### Firmware Behavior (Infrared Remote Input)

| Component                 | Description                                                           |
| ------------------------- | --------------------------------------------------------------------- |
| **IRQ Vector**            | Located in vector table at ROM `0x000070` (IRQ3) or `0x000078` (IRQ5) |
| **ISR Handler**           | Located near `0x801820`                                               |
| **Decoded Command Table** | Stored at `0x804700–0x8047FF`                                         |
| **Dispatch Table**        | Jump table to action routines: `0x801900–0x8019FF`                    |

### IR Handling Flow

```plaintext
[CN101 IR Receiver] → [IRQ5 interrupt] → [Firmware ISR @ 0x801820]
→ Decode pulse into 1-byte command (NEC protocol)
→ Match code to table @ 0x804700
→ Use index to jump into handler table @ 0x801900
→ Call corresponding routine (e.g., Play, Stop, Open Tray)
```

### Pseudocode Example (Infrared Remote Input)

```c
// ISR Entry Point
uint8_t ir_code = decode_ir_signal();   // NEC format
handler = IR_TABLE_BASE[ir_code];       // @0x804700
jump_to(IR_DISPATCH_TABLE[handler]);    // @0x801900 + (handler * 2)
```

---

## 🔘 2. Front Panel Button Input (via GPIO)

### Hardware Overview (Front Panel Buttons)

- Buttons (Play, Stop, Open/Close, etc.) are wired directly to **SH7034 GPIO Ports**
- Input polling occurs via digital I/O (likely ports P1 and P3)
- Port addresses:
  - Port 1 Data: `0xFFFFD004`
  - Port 3 Data: `0xFFFFD00C`

### Firmware Behavior (Front Panel Buttons)

| Function                 | Address                     |
| ------------------------ | --------------------------- |
| GPIO Polling Loop        | ~`0x801380`                 |
| Bitmask Test/Decode      | ~`0x8013E0`                 |
| Input-to-Action Dispatch | Shared with IR → `0x801420` |

### GPIO Handling Flow

```plaintext
Timer Loop every 30ms →
→ Read GPIO ports P1/P3 →
→ Bitmask each key →
→ Lookup action →
→ Call routine (same as IR)
```

### Pseudocode Example (Front Panel Buttons)

```c
uint8_t port1 = read8(0xFFFFD004);
if (!(port1 & 0x10)) {
    // Play button
    call(play_handler);
}
```

---

## 🔌 3. Hardware Hook/Injection Points

To simulate or inject user inputs (e.g., from an ODE menu):

| Signal Source  | Hook Method                                | Notes                                  |
| -------------- | ------------------------------------------ | -------------------------------------- |
| IR Receiver    | Replace CN101 with custom IR signal source | Inject NEC pulse directly              |
| GPIO Buttons   | Drive GPIO pins low via open-drain buffer  | Use GPIO expander to simulate keypress |
| Interrupt Line | Trigger IRQ manually from custom device    | May require edge pulse + mask toggling |

---

## 🧠 ODE Application Notes

- You can **drive front panel behavior externally** by toggling GPIO lines that represent buttons
- You can simulate remote control inputs by **injecting NEC-coded pulses**
- Firmware **does not differentiate between front panel and remote** → both route to the same command dispatch table

---

## 📚 References

- SH7034 Datasheet – Interrupts, GPIO Ports
- Firmware Analysis `dvp-s315_v17.bin`
- Sony DVP-S315 Service Manual – Remote Code Map, Panel Layout
