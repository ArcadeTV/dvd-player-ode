# UI String Table and OSD Text Handling in Sony DVP-S315 Firmware

> 📘 This document analyzes the static string table embedded in the DVP-S315 firmware and its role in driving the on-screen display (OSD). These strings correspond to various UI messages such as "NO DISC", "LOADING", "TITLE", etc., and are likely used by the IC604 microcontroller to instruct the MB90096 OSD controller to render text overlays.

---

## 📌 Location in ROM

- **Address Range:** `0x803000–0x803300`  
- Strings are aligned in a null-terminated format
- Referenced by routines likely involved in system state display (IC604 coordination)

---

## 🔎 Known Strings Extracted from Firmware

```text
"NO DISC"
"LOADING"
"TITLE 1"
"CHAPTER"
"ERROR"
"INVALID"
"INITIALIZE"
"READING"
"STOP"
"PAUSE"
"PLAY"
```

These strings appear to match real-world OSD overlays observed during playback, boot, and tray events.

---

## 🧠 Usage Context

The SH7034 itself does not generate video. It:

- Sends **control/state data** to **IC604 (MB90T678)**, the interface µC
- IC604 references this string table or is informed of an index/state
- IC604 sends serialized control data to the **MB90096 OSD controller**
- MB90096 outputs the string via analog RGB overlay (ROUT/GOUT/BOUT)

---

## 📦 String Trigger Examples

| Trigger Event   | Displayed Text | Inferred Source Routine |
| --------------- | -------------- | ----------------------- |
| No disc present | `NO DISC`      | `0x801380`–`0x8014A0`   |
| Reading disc    | `LOADING`      | Init routines post-IDE  |
| Playback start  | `TITLE 1`      | PLAY handler            |
| Stop pressed    | `STOP`         | UI dispatcher           |
|                 |                |                         |

---

## 💡 Emulation Implications (ODE)

| Scenario         | ODE Behavior                   | Visual Effect  |
| ---------------- | ------------------------------ | -------------- |
| No image mounted | Trigger SENSE 0x02 + 0x3A      | Show `NO DISC` |
| Loading image    | Stall after INQUIRY, wait 2s   | Show `LOADING` |
| Valid playback   | Respond to `READ(10)` quickly  | Show `TITLE x` |
| Error (bad TOC)  | Trigger Illegal Request (0x05) | Show `ERROR`   |

ODE developers **do not need to simulate string output**, but by triggering the correct firmware state transitions and SENSE replies, the system will automatically route the proper strings to the OSD.

---

## 🧰 Tips for Visual Debugging

- Use logic analyzer on MB90096 serial lines (SIN/SCLK/CS) during string updates
- Log state changes on CN601 and compare to OSD behavior
- Monitor SH7034 activity around 0x803000 reads to correlate context

---

## 📚 References

- Firmware image: `dvp-s315_v17.bin`, `0x803000–0x803300`
- MB90096 OSD Controller Datasheet
- System diagrams (`Fig. 2-18`, Service Manual)
- UI flow inferred from disassembly and logic behavior
