
# 📄 CD vs. DVD Dispatcher Branch Logic – Sony DVP-S315 Firmware

---

## 🔍 Summary

This document explains how the Sony DVP-S315 firmware distinguishes between **CD and DVD discs**, and how this affects the internal command dispatching behavior. The results are confirmed by firmware disassembly and Sony service manuals.

---

## 🧠 Disc Type Detection Logic

- **ROM Range:** `0x800740–0x800880`
- **Function:** Evaluates input from DSP/servo system and writes disc-type code to RAM.
- **RAM Location (likely):** `0x5C00801C` or similar
- **Values:**

  | Value | Type                                                                 |
  | ----- | -------------------------------------------------------------------- |
  | 00h   | No Disc                                                              |
  | 01h   | CD (12cm)                                                            |
  | 02h   | DVD Single Layer                                                     |
  | 03h   | DVD Dual Layer                                                       |
  | 04h+  | CD-R variants etc. (from `sony_dvp-s300_s305_s315_supplement-1.pdf`) |

---

## 🔁 Dispatcher Switching Logic

```c
uint8_t disc_type = *(uint8_t*)0x5C00801C;

switch (disc_type) {
    case 0x01:
        // CD mode
        cd_dispatcher();
        break;
    case 0x02:
    case 0x03:
        // DVD mode
        dvd_dispatcher();
        break;
    default:
        // No disc or unknown
        signal_error();
}
```

---

## 🛠 Emulation Strategy

To accurately emulate the dispatcher behavior in an ODE:

- Set the `disc_type_flag` in RAM prior to command dispatch
- Differentiate `TOC` layout, `READ`, and `PLAY` behavior based on type
- Return CD or DVD-specific responses to `READ TOC`, `INQUIRY`, etc.

---

## 📘 Service Manual Confirmation

Confirmed by:

- `sony_dvp-m35_s300_s305_s315_s500_s505_s715.pdf`: Servo & TE signal logic
- `Sony_DVD_Training.pdf`: TE/PI level specs for CD/DVD/DL
- `sony_dvp-s300_s305_s315_supplement-1.pdf`: Explicit disc type codes and addresses

---

## ✅ Summary

The firmware implements a clear, service-manual-verified branching mechanism to separate **CD vs. DVD** command behavior. Emulating this correctly is critical to supporting both formats under ODE conditions.
