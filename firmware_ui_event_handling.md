
# 📄 UI Event Handling: IR & Frontpanel Mapping – Sony DVP-S315 Firmware

---

## 🔍 Summary

This document outlines how the Sony DVP-S315 firmware maps **user inputs** from both the **front panel buttons** (via GPIO) and **IR remote control** (via SIRCS protocol) to internal command and menu actions.

---

## 🔌 Front Panel Inputs (GPIO)

- **Connector:** CN602
- **MCU:** IC805 (SH7034)
- **Port Used:** Port C (`PC0–PC7`)
- **Register:** `PCDR` at `0x5FFFFD0`
- **Pin Range:** SH7034 pins 87–94
- **Function:** Each pin corresponds to one or more physical button states

Polling loop reads:

```c
uint8_t keys = *(volatile uint8_t*)0x5FFFFD0;
if (keys & 0x01) do_play();
if (keys & 0x02) do_stop();
...
```

---

## 📡 IR Remote Inputs

- **IR Decoder:** IC804
- **MCU Interrupt Pin:** `PB15` (IRQ3 on IC805)
- **Protocol:** Sony SIRCS (12-bit)
- **Handler:** Triggered via IRQ3 → decodes IR code
- **Event Dispatch:** Matches against command table in firmware (e.g., `"KEY_PLAY"`)

Typical flow:

```c
void irq3_handler() {
    uint16_t ir_code = read_sircs();
    enqueue_ui_event(ir_code);
}
```

---

## 🧠 Event → Action Dispatcher

- **ROM Region:** `0x800200–0x800500`
- Handles both key polling and IR interrupts
- Maps UI inputs to:
  - Playback control
  - OSD commands
  - Menu navigation

---

## 🛠️ Emulation Recommendations

For ODE hardware (e.g., STM32):

- **GPIO simulation**: Use digital inputs or USB encoder to replicate button matrix
- **IR simulation**: Emulate SIRCS over GPIO, or intercept/replicate IRQ-based event handling
- **Memory Emulation**: Update `0x5FFFFD0` equivalent RAM for input flags

---

## 📘 Confirmed by Documentation

- `SH7032_HD6437034.PDF`: Port C, PCDR register, IRQ pinouts
- `dvps300.pdf`: CN602 wiring, IR logic flow
- `sony_dvp-m35_s300_s305_s315_s500_s505_s715.pdf`: Full IR/Key matrix
- `Sony_DVD_Training.pdf`: SIRCS and remote protocol explanation

---

## ✅ Summary

The firmware cleanly separates input paths:

- **Polling-based GPIO input** for front buttons
- **IRQ-based IR input** with command decoding
- Dispatcher logic maps these to system control functions, which can be fully emulated
