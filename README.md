# dvd-player-ode
Optical Drive Emulator for a SONY DVP-S315 (or similar) standalone DVD Player

## Why build an “Optical Drive Emulator” for an obsolete system like a standalone DVD player?

I originally just wanted to watch videos in 480i/576i on a CRT, but my HTPC setup—with CRT‑Emudriver and an ATI HD5450—never quite delivered. Interlaced resolutions couldn’t be generated perfectly, and video sources vary so wildly that you can’t rely on VLC (or any other media player) to magically fix everything. Interlaced files in particular gave me headaches, since the timing between the video stream and the display never lined up 100%.

Subjectively, as someone who grew up in the ’80s and ’90s, I think DVD/MPEG‑2 was the highest‑quality consumer format ever made for tube TVs.

As a tech‑savvy retro fan with a background in electronics, I wondered how far I could get by gathering every scrap of information about my old DVD player and feeding it into an AI.

Once I sketched out the idea and tossed it into the AI, my ambition grew: I wanted to see it through and make it real. An emulator that could mount video data and ISO files, giving the old player a brand‑new lease on life. I was thrilled to find the firmware sitting on a socketed MBM29F800B‑90, which I was able to dump thanks to a matching programmer + adapter.

Before you get too excited, note that this repository is currently just an information pool. Its accuracy still needs to be verified and turned into hardware

_—but hey, it’s a start!_

---

## Sony DVP-S315 Optical Drive Emulator (ODE) Documentation

This document outlines the reverse engineering and development progress of a custom Optical Drive Emulator (ODE) designed to replace the physical DVD mechanism in the Sony DVP-S315 with a solid-state emulated alternative. The goal is to transparently present VIDEO_TS folders and ISO disc images to the DVD player via an emulated ATAPI/IDE interface.

### Project Goals

- Replace the original optical drive with an SSD- or SD-based solution
- Fully emulate ATAPI/IDE interface expected by the DVP-S315
- Allow disc image selection via on-device interface (rotary encoder + OLED)
- Stream DVD-Video content directly from VIDEO_TS folder structures or ISO images
- Achieve compatibility with standard playback functions (menu navigation, chapters, etc.)

### Hardware Environment

- **DVD Player**: Sony DVP-S315
- **Mainboard**: MB-78
- **Main CPU**: SH7034 (Renesas SH-1 architecture)
- **Flash ROM**: Fujitsu MBM29F800B (1MB, 8-bit)
- **Key Support ICs**:
  - CXD8730R (DSP for servo and signal processing)
  - CXD8728Q (Large Gate Array for system control and bus switching)
  - L64021 (AV Decoder)

### DVD Media Emulation

The ODE system will support two media formats:

- **ISO images**: entire DVD images stored on SSD or SD card, presented as a single emulated disc
- **VIDEO_TS folders**: standard DVD folder structure; files are served to the IDE layer according to sector mappings

### Firmware Analysis

- Fujitsu MBM29F800B firmware dump obtained from MB-78 board
- Analysis performed using Binwalk, Strings, and Ghidra (with SH-1 processor plugin)
- Debug strings identified for ATAPI command handling, disc tray status, and video/audio state
- Bootloader code and early hardware init mapped to address ranges 0x000000–0x1FFFF

#### 🔍 Planned Ghidra Analysis Targets

To further refine ODE behavior and integration:

1. **ATAPI Command Dispatch Table**
   - Located string references to `IDENTIFY DEVICE`, `READ(10)`, `REQUEST SENSE` around address `0x00079D0`
   - Possible command handler at `0x000743C` dispatching by opcode index
2. **OSD/UI Routines**
   - Menu and OSD-related strings (e.g. "SETUP MENU", "PARENTAL CONTROL") found around `0x0002F000–0x00032000`
   - Likely handled by routines in the `0x00031xxx` address space
3. **Interrupt Handlers and I/O Hooks**
   - Targeting `0x00000010–0x0000007F` for possible vector table entries
   - Used to identify routines for tray state, IR remote, or drive readiness

#### 📌 Ghidra Function Annotations (Initial Set)

| Address   | Label Name               | Description                                           |
|-----------|--------------------------|-------------------------------------------------------|
| 0x000743C | atapi_dispatch_handler   | Likely ATAPI opcode router                            |
| 0x00079D0 | atapi_command_stringtbl  | Table of ATAPI command labels                         |
| 0x00031C0 | osd_menu_handler         | Possibly renders setup/resume menus via OSD          |
| 0x0002F300| osd_string_block         | Block of menu/display strings                         |

These early discoveries provide a solid base for labeling, function mapping, and code injection or hook targets in later firmware patching stages.

- Fujitsu MBM29F800BA firmware dump obtained from MB-78 board
- Analysis performed using Binwalk, Strings, and Ghidra (with SH-1 processor plugin)
- Debug strings identified for ATAPI command handling, disc tray status, and video/audio state
- Bootloader code and early hardware init mapped to address ranges 0x000000–0x1FFFF

#### 🔍 Planned Ghidra Analysis Targets

To further refine ODE behavior and integration:

1. **ATAPI Command Dispatch Table**
   - Locate command decoding logic (e.g. READ(10), REQUEST SENSE, IDENTIFY)
   - Look for `switch`-like or function pointer tables related to opcode parsing
2. **OSD/UI Routines**
   - Map display strings, setup/menu entries, error messages
   - Identify firmware routines responsible for screen updates or menu transitions
3. **Interrupt Handlers and I/O Hooks**
   - Find `INT` or `TRAP` vectors related to ATAPI, I/O readiness, and system calls
   - Map handler functions for tray buttons, remote control, status polling

These points are prioritized to support early emulation and native UI injection.

- Fujitsu MBM29F800BA firmware dump obtained from MB-78 board
- Analysis performed using Binwalk, Strings, and Ghidra (with SH-1 processor plugin)
- Debug strings identified for ATAPI command handling, disc tray status, and video/audio state
- Bootloader code and early hardware init mapped to address ranges 0x000000–0x1FFFF

### Drive Interface Connector Review

A careful review of Sony service manuals confirms the following connectors are involved in connecting the MB-78 mainboard to the DVD mechanism:

#### 🔌 CN304 – Main Data Bus to Drive Unit

| Pin   | Signal                   | Description                 |
| ----- | ------------------------ | --------------------------- |
| 1–21  | Various IDE-like signals | Data, control, chip-selects |
| 22–25 | GND / Shielding          | Ground return paths         |

- High pin count FFC/FPC connector
- Connected directly to CXD8728Q and DSP
- Most probable candidate for ATAPI interface

#### 🔌 CN452 – Servo Signal Routing

| Pin | Signal             | Description                      |
| --- | ------------------ | -------------------------------- |
| 1   | TE                 | Tracking Error (analog)          |
| 2   | FE                 | Focus Error (analog)             |
| 3   | PI                 | Push-in signal (disc load sense) |
| 4   | SL/DL              | Single/dual-layer detection      |
| ... | Analog mux control | Switched by MC14053              |

- Connected to CXD8730R, CXD8728, and analog mux

#### 🔌 CN361 – Secondary Mechanism Interface

- Pinout unspecified in manual
- Leads to drive unit
- Likely carries servo/motor control or disc state sensing
- Moderate relevance to ODE, depending on feature completeness

#### 🔌 CN251 – Tray Motor and Chucking Control

| Pin | Signal     | Description                      |
|-----|------------|----------------------------------|
| 1   | TRAY_IN_SW | Tray fully in                    |
| 2   | TRAY_OUT_SW| Tray fully out                   |
| 3   | CHUCK_SW   | Chucking complete                |
| 4   | MOT_CTRL   | Tray motor control (PWM/dir)     |
| 5–6 | GND/VCC    | Power                            |

- Controlled via BA5912 and logic from SH7034 or CXD8728

### System Architecture

The signal architecture of the Sony DVP-S315, based on Fig. 1-27 from the Sony DVD Training manual (p. 57), illustrates the complete data flow from optical disc to final video/audio output. This architecture is essential for understanding how the ODE must behave to remain compatible with the system controller and media decoder logic.

### DVD Stream Format & Multiplexing Requirements (Based on p. 58 ff.)

Following pages in the Sony DVD Training manual expand on the role of packet multiplexing and stream composition in the DVD playback system.

#### 🧱 Stream Packaging

- DVD content is stored as **packetized multiplexed streams** (audio, video, subpicture)
- Each stream contains header metadata (type, stream ID, timestamps)
- Multiplexed into logical sectors that align with physical disc blocks

#### 🎥 Stream Types:

| Type       | Count    | Encoding Formats                   |
| ---------- | -------- | ---------------------------------- |
| Video      | 1        | MPEG-2                             |
| Audio      | Up to 8  | PCM, Dolby AC-3, MPEG-1/2 Layer II |
| Subpicture | Up to 32 | RLE-encoded bitmap overlays        |

These stream types are embedded within `.VOB` files inside the `VIDEO_TS` folder and are decoded downstream by the AV IC (e.g. L64021).

#### 🔄 Implications for ODE Emulation

- Data must be delivered **sector-wise and in precise timing**
- The ODE must maintain original **packet boundaries** and **header integrity**
- **IFO/BUP files** in VIDEO_TS provide navigation and are required during initial disc scan
- Playback, menu navigation, and chapter seeking rely on correct **packet ordering and stream IDs**
- Errors in mux structure will lead to decoder desync or playback halt

The emulation layer must preserve the sector structure, not just the file contents. Especially during READ/SEEK/PLAY ATAPI commands, these constraints must be respected.

The signal architecture of the Sony DVP-S315, based on Fig. 1-27 from the Sony DVD Training manual (p. 57), illustrates the complete data flow from optical disc to final video/audio output. This architecture is essential for understanding how the ODE must behave to remain compatible with the system controller and media decoder logic.

#### 🔄 Playback Signal Chain

1. **Optical Pickup Unit (OPU)**: Reads physical DVD surface.
2. **RF Amplifier**: Amplifies modulated signals from laser reflections.
3. **RF Processor & Servo (e.g. CXD8730R)**:
   - Converts RF signal to digital data
   - Extracts synchronization and clock
   - Generates error correction feedback (TE, FE, etc.)
4. **Data Decoder (via CXD8728Q & DSP)**:
   - EFM demodulation, de-interleaving
   - Error correction (Reed-Solomon)
   - Sector assembly
5. **AV Decoder**:
   - MPEG2 video and Dolby/MPEG/PCM audio decoding
   - OSD overlay and subtitle insertion
6. **DAC / Encoder**:
   - Converts digital output to analog signals (video & audio)
7. **System Controller (SH7034)**:
   - Coordinates all subsystems
   - Issues ATAPI commands (e.g. READ, SEEK, PLAY)
   - Manages tray motor, LED, IR remote, and disc status

#### 📌 Relevance to ODE

The ODE must emulate responses that normally originate from the RF → EFM → Sector → Decoder path. This includes responding to:

- IDENTIFY DEVICE
- READ SECTOR
- PLAY/PAUSE, etc.
- Layer jump emulation and error codes when applicable

The ODE must also keep the SH7034 in a valid communication state, as this CPU acts as the overall system orchestrator.

---

### Bus Interface Mapping (IC604 ↔ IDE)

Based on pages 26–28 of the `dvps300.pdf` service manual, IC604 (system controller) connects directly to a 16-bit data bus and supporting control signals that emulate a parallel interface similar to IDE. These are used to drive communication with the DVD mechanism.

| Pin(s) | Signal     | Direction | Function                                 |
| ------ | ---------- | --------- | ---------------------------------------- |
| 85–92  | AD0–AD7    | I/O       | Lower data/address bus                   |
| 93–100 | A8–A15     | I/O       | Address bus                              |
| 1–4    | A16–A19    | I/O       | High address range                       |
| 12     | WRL        | Output    | Write strobe (active LOW)                |
| 10     | OE         | Output    | Output enable (READ strobe)              |
| 9      | ALE        | Output    | Address latch enable                     |
| 16     | INTMS      | Input     | Interrupt or IORDY signal from mechanism |
| 31     | RESET      | Output    | Global system reset                      |
| 28     | CGCS       | Output    | Character generator chip select          |
| 26     | AUDIO MUTE | Output    | Audio mute during disc insert/eject      |
| 25     | VIDEO MUTE | Output    | Video mute during invalid playback state |
| 33     | P.CONT     | Output    | Power control to disc mechanism          |

These lines replicate many control features found on a traditional ATA/ATAPI interface. The ODE must respond accordingly using these timing and control signals.

### Embedded Test Mode Interface

Pages 29+ of `dvps300.pdf` describe a hidden **Test Mode Menu**, activated via remote.

#### 🧪 Test Mode Features:

- View and clear **emergency history logs** (CD/DVD laser hours)
- Access **servo alignment pages**
- View **ATAPI/SCSI command results**
- Activate **manual spindle and tray operations**

#### 🧪 Implications for ODE Testing

The ODE should support selected test features:

- [x] Respond to command sequences that populate emergency logs
- [x] Validate `IDENTIFY DEVICE`, `READ CAPACITY`, `REQUEST SENSE`
- [x] Trigger playback states for DNR and OSD test pages
- [x] React properly to abnormal tray/spindle commands for fault handling

These modes provide a useful entry point for **automated regression tests** during development and validation.

#### 📋 ODE Test Plan (Based on Test Menu Functions)

| Test Case                 | Action                              | Expected Result                                     |
| ------------------------- | ----------------------------------- | --------------------------------------------------- |
| Emergency History Read    | Enter test mode → EMG Hist Page     | ODE responds with dummy laser hour history          |
| IDENTIFY DEVICE           | Issue ATAPI 0xEC command            | Return 512-byte IDENTIFY block (with ODE signature) |
| READ CAPACITY             | Issue SCSI 0x25 via test mode       | Return last LBA + block size                        |
| REQUEST SENSE             | Trigger error → send SCSI 0x03      | Report valid sense key (e.g. NO DISC / READY)       |
| Manual Tray Open / Close  | Use test mode to trigger tray motor | ODE logs action or toggles emulated tray bit        |
| DNR/OSD/Servo Diagnostics | Use remote code to switch pages     | ODE remains stable and passes control flow          |
| Region Code Validation    | Insert disc with region mismatch    | ODE returns error on read (as per DVD spec)         |

This test plan should evolve into a reproducible suite integrated into firmware builds or test scripts using GPIO triggers or UART logging.
Pages 29+ of `dvps300.pdf` describe a hidden **Test Mode Menu**, activated via remote.

#### 🧪 Test Mode Features

- View and clear **emergency history logs** (CD/DVD laser hours)
- Access **servo alignment pages**
- View **ATAPI/SCSI command results**
- Activate **manual spindle and tray operations**

#### 🧪 Test Mode: Implications for ODE Testing

The ODE should support selected test features:

- [ ] Respond to command sequences that populate emergency logs
- [ ] Validate `IDENTIFY DEVICE`, `READ CAPACITY`, `REQUEST SENSE`
- [ ] Trigger playback states for DNR and OSD test pages
- [ ] React properly to abnormal tray/spindle commands for fault handling

These modes provide a useful entry point for **automated regression tests** during development and validation.

### Retaining the TK-47 (RF/SERVO) Board

The TK-47 board is part of the optical drive assembly and contains critical analog and servo processing components—specifically:

- RF signal amplification and filtering
- Focus and tracking servo feedback (TE, FE, etc.)
- Push-in detection, spindle motor drive, sled control

#### 💡 Observation from Service Manual

The TK-47 board is modular and removable. While traditionally part of the drive unit, it can be retained and powered separately if needed. Its interface back to the mainboard is handled via CN304, CN452, CN251, and CN361.

#### 🧠 Benefits of Retaining TK-47:

- Access to original **servo error signals** (FE/TE/PI) for optional diagnostics
- Preserves **electrical behavior** expected by CXD8730/CXD8728 DSP/Gate Array
- Allows **tray/chuck motor control lines** (via CN251) to stay functional
- Avoids risk of signal-line stubs or floating analog lines on MB-78

#### ⚠️ Tradeoffs:

- If motors/laser removed, TK-47 partially "idles" unless signals are emulated
- Still draws some current (must ensure stability and shielding)
- May require terminating inputs if disconnected

#### 🔍 Strategic Use in ODE

- TK-47 could be retained as a **passive bridge** to keep CXD8730 happy
- Servo signals (even unconnected) may need defined levels (bias/resistors)
- If future version aims to emulate analog behavior (e.g. for full TEST MODE pass-through), having TK-47 in the loop may be beneficial

#### 🧩 Role in Legacy Signal Initialization

According to the process outlined in `Sony_DVD_Training.pdf` (p. 22+), the initial disc detection, chucking, focus and tracking loop activation, and RF signal stabilization originate from components located on TK-47. While these actions do not directly involve ATAPI protocol, they are part of the broader system state expected by the DSP and Syscon before an ATAPI session is active. Retaining TK-47, even with the spindle and pickup assembly removed, may preserve expected system behavior and simplify state transitions.

**Conclusion:** Retaining TK-47 with just the mechanical block removed is viable and potentially helpful. Its outputs can be left idle, emulated, or sampled for debugging. Recommended for early prototyping phase.

The TK-47 board is part of the optical drive assembly and contains critical analog and servo processing components—specifically:

- RF signal amplification and filtering
- Focus and tracking servo feedback (TE, FE, etc.)
- Push-in detection, spindle motor drive, sled control

#### 💡 TK-47 board: Observation from Service Manual

The TK-47 board is modular and removable. While traditionally part of the drive unit, it can be retained and powered separately if needed. Its interface back to the mainboard is handled via CN304, CN452, CN251, and CN361.

#### 🧠 Benefits of Retaining TK-47

- Access to original **servo error signals** (FE/TE/PI) for optional diagnostics
- Preserves **electrical behavior** expected by CXD8730/CXD8728 DSP/Gate Array
- Allows **tray/chuck motor control lines** (via CN251) to stay functional
- Avoids risk of signal-line stubs or floating analog lines on MB-78

#### ⚠️ Retaining TK-47: Tradeoffs

- If motors/laser removed, TK-47 partially "idles" unless signals are emulated
- Still draws some current (must ensure stability and shielding)
- May require terminating inputs if disconnected

#### 🔍 Retaining TK-47: Strategic Use in ODE

- TK-47 could be retained as a **passive bridge** to keep CXD8730 happy
- Servo signals (even unconnected) may need defined levels (bias/resistors)
- If future version aims to emulate analog behavior (e.g. for full TEST MODE pass-through), having TK-47 in the loop may be beneficial

**Conclusion:** Retaining TK-47 with just the mechanical block removed is viable and potentially helpful. Its outputs can be left idle, emulated, or sampled for debugging. Recommended for early prototyping phase.

### Optional Test Setup: Native Drive with External ATAPI Interface

As part of exploratory development and validation, it may be useful to temporarily remove the drive mechanism from the DVD player and interface with it externally to analyze native ATAPI behavior.

#### 🧠 Rationale

By connecting the original optical drive (including TK-47) to a separate microcontroller or PC with ATAPI support, developers can:

- Observe real-world command-response sequences (e.g. for `.IFO` reads)
- Monitor how the drive initializes, recognizes media types, and handles seek/play requests
- Identify protocol-specific timing, quirks, and error behavior

#### 🔧 Setup Requirements

- Power supply (+5V and +12V with controlled startup)
- Level-safe connection to IDE/ATAPI bus (via logic analyzer, MCU, or PC with IDE adapter)
- Disc insertion and tray/chucking logic emulated or manually triggered

#### 🔍 Potential Applications

| Goal                                 | Purpose                                                 |
| ------------------------------------ | ------------------------------------------------------- |
| Record `READ` sequences for VIDEO_TS | Understand how `.IFO`, `.VOB`, `.BUP` are accessed      |
| Validate media detection             | Confirm response to CDDA, SA-CD, VCD, DVD-Video         |
| Evaluate error handling              | Observe REQUEST SENSE or DISC ERROR behavior            |
| Confirm tray open/close commands     | Trigger via GPIO or remote and trace resulting commands |

#### 📌 Benefits

- Independent platform for controlled ATAPI testing
- Enhances understanding of firmware behavior
- Useful reference for FPGA/MCU emulator development

This test setup is not a long-term requirement but serves as a **valuable exploratory tool** during reverse engineering and development phases.

---

### IDE Interface (Suspected)

**Connector CN304** is now the most likely candidate for the ATAPI-style interface.
Earlier reference to CN504 was incorrect and no such connector is referenced in official schematics.

### Supported Media Formats & Technical Constraints

The ODE aims to support any media format compatible with the original DVD player hardware, to the extent that such formats can be reasonably emulated. Below is a summary of formats and the constraints they introduce:

| Medium/Format          | ISO Required    | Notes                                                                                                 |
| ---------------------- | --------------- | ----------------------------------------------------------------------------------------------------- |
| DVD-Video              | ❌ No           | Supported via VIDEO_TS folder emulation                                                               |
| DVD-ROM (Data)         | ✅ Yes          | Sector-exact ISO required to maintain filesystem and boot sectors                                     |
| SA-CD (Super Audio CD) | ✅ Yes          | Filesystem alone insufficient; ISO/RAW image needed due to copy protection and hybrid layer structure |
| CD-DA (Audio CD)       | ✅ Yes          | BIN/CUE or ISO image with TOC and index data required                                                 |
| VCD/SVCD               | ⚠️ Prefer ISO | File structure complex; ISO preferred for mux consistency                                             |

Only formats with predictable and sector-addressable layouts can be emulated reliably. SA-CD and some Redbook CD formats include structures not visible in standard file-level dumps, requiring ISO or RAW imaging to ensure completeness.

#### 📦 Supported File Types for Disc Selection UI

The following extensions will be used by the ODE file scanner:

- `.iso`, `.img`, `.bin` — for full disc images
- `.cue` — used alongside `.bin` for CD-DA/Audio CDs
- `.vob`, `.ifo`, `.bup` — as part of `VIDEO_TS` folder content
- `.ts`, `.mpeg`, `.mpg` — optionally for direct stream testing (not navigable)
- `.dvd` — optional descriptor for multi-part ISOs or Layer info

This information will later become part of the official **Operator’s Manual** to assist users in preparing and organizing their media content.
The ODE aims to support any media format compatible with the original DVD player hardware, to the extent that such formats can be reasonably emulated. Below is a summary of formats and the constraints they introduce:

| Medium/Format          | ISO Required    | Notes                                                                                                 |
| ---------------------- | --------------- | ----------------------------------------------------------------------------------------------------- |
| DVD-Video              | ❌ No           | Supported via VIDEO_TS folder emulation                                                               |
| DVD-ROM (Data)         | ✅ Yes          | Sector-exact ISO required to maintain filesystem and boot sectors                                     |
| SA-CD (Super Audio CD) | ✅ Yes          | Filesystem alone insufficient; ISO/RAW image needed due to copy protection and hybrid layer structure |
| CD-DA (Audio CD)       | ✅ Yes          | BIN/CUE or ISO image with TOC and index data required                                                 |
| VCD/SVCD               | ⚠️ Prefer ISO | File structure complex; ISO preferred for mux consistency                                             |

Only formats with predictable and sector-addressable layouts can be emulated reliably. SA-CD and some Redbook CD formats include structures not visible in standard file-level dumps, requiring ISO or RAW imaging to ensure completeness.

### To Do

- [ ] Complete Ghidra analysis of SH7034 firmware (command handlers, ATAPI dispatch)
- [ ] Confirm CN304 signal mapping and direction via continuity/logic probing
- [ ] Capture IDE/ATAPI traffic from real drive (insert, play, menu, chapter skip)
- [ ] Write Cyclone II HDL block for ATAPI command emulation (IDENTIFY, READ, etc.)
- [ ] Design level shifter & breakout PCB for CN304 to dev board
- [ ] Implement Raspberry Pi interface for ISO/VIDEO_TS file delivery
- [ ] Integrate OLED & rotary encoder for disc selection interface
- [ ] Document supported ATAPI command set (based on observed traces)

### Frontend and User Interaction Roadmap

As part of enhancing usability, particularly for everyday use (e.g. binge-watching a series), the ODE will include a flexible and non-intrusive user interface. The following ideas are proposed, prioritized by feasibility and integration potential:

#### ✅ Basic Behavior Proposal

- The last-mounted ISO or VIDEO_TS folder remains active **until** the user explicitly triggers **EJECT DISC**.
- Pressing the original **EJECT DISC button** enters **disc selection mode** on the frontend.

#### 💡 Interface Ideas (User-Facing Options)

| Method                   | Description                                                   | Feasibility / Status |
| ------------------------ | ------------------------------------------------------------- | -------------------- |
| OLED + rotary encoder    | Simple, embedded UI for selection and feedback                | 💡 planned          |
| Web UI (Raspberry Pi)    | File browser with disc info, mount/eject options              | 💡 planned          |
| Hijack player LCD + keys | Use onboard LCD & directional pad for overlay menus           | 🔍 to be explored   |
| TV OSD injection mod     | Inject OSD via analog video switch (e.g. SCART overlay frame) | ❓ research needed   |

#### 🔧 Backend Support Features

- Persistent mount tracking (last used image)
- Configurable default disc (auto-mount at boot)
- Emulated tray state control (for TEST-MENU interaction)
- Key debounce and event queue for remote / button control

These features aim to blend modern usability with full compatibility to the DVD player's legacy interface.

---

### Optional Roadmap: Native OSD Injection (Firmware Level)

Rather than overlaying video externally, a more elegant and integrated method is to utilize the player's **native OSD system**, which is controlled by the SH7034 CPU and displayed through the L64021 AV decoder.

#### 🧠 Concept

- The player already has menu rendering logic (setup, title, region error, etc.)
- The SH7034 firmware triggers OSD display modes based on state/context
- Custom ODE logic could intercept or redirect these triggers to custom content

#### 🔧 Required Steps

1. **Reverse engineer firmware routines** that trigger known menus (e.g. setup menu)
2. **Identify string tables and display calls** used by OSD (e.g. from ROM or RAM)
3. Hook into or redirect calls when EJECT or special input is detected
4. Inject custom strings or display payloads using the native OSD API / memory buffers
5. Restore state and return control to original firmware when menu exits

#### 📌 Advantages

- Native look & feel (like original menus)
- No external hardware needed
- Compatible with existing remote and keypads

#### ⚠️ Challenges

- Requires solid disassembly of OSD routines in SH7034 firmware
- Risk of breaking state machine if return path isn’t exact
- Unknown buffer formats or display API specifics

This feature is advanced, but elegant and fully integrated. It is a promising direction for the ODE interface and should be investigated after stable base functionality is achieved.

---


## 🛠️ ODE Integration Overview – Sony DVP-S315 (Developer Outline) 

This outline describes how to physically and logically integrate a custom ODE (Optical Drive Emulator) into the Sony DVP-S315 DVD player, replacing the internal drive electronics and emulating ATAPI communication via **CN601** .

### 🔧 Key Integration Points 

1. **Disconnect Internal Drive** 
 
- Remove the optical pickup and spindle motor assembly.
 
- Detach wiring to the internal sled/tray mechanism unless reused.

2. **IDE Bus Tap via CN601** 
 
- Use **CN601**  to interface with the player's **IDE/ATAPI bus** .
 
- CN601 carries standard IDE lines (`D0–D15`, `CS0`, `CS1`, `IOR`, `IOW`, `RESET`, `INTRQ`, `DMARQ`, `DA0–DA2`, `BSY`, `DRQ`, etc.).
 
- Voltage level: **5V TTL** , requires level shifting if MCU/FPGA is 3.3V.

3. **Emulation Core** 
 
- Implement ATAPI device-side behavior in **FPGA or STM32** :
 
  - Respond to IDENTIFY DEVICE / PACKET / REQUEST SENSE
 
  - Deliver data blocks using **PIO IN**  mode (no DMA support needed)
 
  - Emulate status flags (`BSY`, `DRQ`, `INTRQ`) per ATAPI protocol

4. **File Source / Backend** 
 
- Use a **Raspberry Pi 3B**  as file host over SPI/UART or USB
 
- Filesystem contains:

 
  - DVD ISOs
 
  - VIDEO_TS folders
 
- Pi receives LBA requests from the MCU/FPGA and sends back 2048-byte blocks

5. **Disc/Image Selection** 
 
- Controlled via:
 
  - 🖥️ **Web interface**  hosted on the Pi
 
  - 🔘 Optionally via button matrix on the front panel
 
  - 🎮 Or intercepted remote-control commands

6. **On-Screen Menu (Optional)** 
 
- Inject a **virtual menu overlay**  into the analog video path by:
 
  - Tapping into the **video DAC or input of the AV decoder LSI**  (e.g., CXD8730)
 
  - Overlaying menu text before final composite/S-video output
 
  - Triggered during idle or "NO DISC" state

---

## 📦 ODE Block Diagram – Developer-Facing Markdown 

### 📦 ODE System Architecture – Sony DVP-S315 Integration

```text
               ┌──────────────────────────────┐
               │   Sony DVP-S315 Mainboard    │
               │   SH7034 + ATAPI Host Logic  │
               └────────────┬─────────────────┘
                            │ CN601 (IDE bus)
                            ▼
              ┌──────────────────────────────────┐
              │    ODE Emulation Module (FPGA/MCU)│
              │  ┌──────────────────────────────┐ │
              │  │ - Emulates ATAPI device      │ │
              │  │ - Handles PIO read transfers │ │
              │  │ - Responds to IDE commands   │ │
              │  └──────────────────────────────┘ │
              │            │ SPI/UART/USB           │
              ▼            ▼                       ▼
        ┌────────────┐  ┌──────────────────────────────┐
        │ CN601 ↔ IDE│  │ Raspberry Pi 3B              │
        └────────────┘  │ - Hosts VIDEO_TS or ISO files│
                        │ - Provides web UI            │
                        │ - Handles block fetches      │
                        └────────────┬─────────────────┘
                                     │
                                     ▼
                          ┌────────────────────┐
                          │ Disc/Image Select  │
                          │   Logic (on Pi)    │
                          ├────────────────────┤
                          │ - Web Interface    │
                          │ - Button Handler   │
                          │ - RC Command Hook  │
                          └────────┬───────────┘
                                   │
                        ┌──────────▼────────────┐
                        │ (Optional) Video Menu │
                        │ Overlay Injection     │
                        │ - Injected at analog  │
                        │   video DAC path      │
                        └───────────────────────┘
```

---

### ⚙️ Notes for Developers

- IDE state machine must honor **ATAPI timing constraints** from host.
- Pi must respond with low-latency sector data (consider mmap or RAM buffering).
- Region/model flags may affect firmware paths → consider logging initial RAM during boot.
- Ensure **DRQ timing** matches expected pulse width to avoid firmware stalls.
- For video injection, sync with VBI or blanking interval to avoid flicker.

---

## 🎯 Role of the FPGA in the ODE System 

🧠 **Purpose: IDE/ATAPI Protocol Emulation** 
The **FPGA acts as the low-latency front-end**  that emulates the original ATAPI optical drive, handling the raw **IDE timing** , handshaking, and signaling at the electrical level.

#### ✳️ Why not just use the Pi? 

 
- The **Raspberry Pi’s GPIO is too slow and nondeterministic**  (Linux-based, no hard real-time)
 
- IDE protocol uses strict timing for:

 
  - `DRQ` assertion timing after command
 
  - PIO read cycles (~100ns scale)
 
  - Setup/hold requirements per ATA spec
 
- An MCU could work (e.g., STM32F4 at 168+ MHz), but FPGA gives:

 
  - **Precise bus signal control**
 
  - **True parallelism**  for monitoring and responding to multiple lines
 
  - Easier to match legacy timing (ATAPI mode 0 or 3)



---



## 📦 Updated Functional Split 

| Function | Hardware | Justification | 
| --- | --- | --- | 
| ATAPI protocol (slave) | FPGA (Cyclone II) | Accurate signal timing & logic control | 
| Sector data storage & IO | Raspberry Pi 3B | Filesystem, ISO mount, UI/API | 
| Sector fetch over serial | UART/SPI/USB | From Pi to FPGA | 
| Optional UI overlay / control | Raspberry Pi GPIO | Web server / RC / UI handling | 


---

## 🔌 Final Topology 



```text
+------------------+         SPI/UART         +----------------------+
           |     Raspberry Pi | <----------------------> |      FPGA (ODE Core) |
           | - Hosts files     |                         | - IDE interface slave |
           | - Web UI          |                         | - ATAPI handler       |
           +------------------+                         +----------▲-----------+
                                                                       │
                                                                CN601 (IDE)
                                                                       │
                                                       +---------------+---------------+
                                                       |   Sony DVP-S315 Mainboard     |
                                                       |   (ATAPI host – SH7034)       |
                                                       +-------------------------------+
```


