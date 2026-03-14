# ABB Freelance - System Overview & Architecture

> Sources:
> - https://www.ac800pec.com/argument/AC800F.html?_l=en (AC 800F full spec page)
> - https://en.help.plc.abb.com/AB270/4b37b84c6b6526b36cbcb8149c5e8082_2_en_us.html (Task configuration)
> - https://help.plc.abb.com/_cds_f_task_configuration.html (Task config reference)
> - https://forumautomation.com/t/introduction-to-abb-ac-800f-controller/10431 (community intro)
> - https://mcaps-inc.com/abb-dcs-systems/abb-freelance/acf800f-controllers/ac-800f-central-processing-unit.html
> - https://forum-controlsystems.abb.com/6520384/Freelance-IDF-1-block-alarms (IDF_1 forum thread)
> - https://library.e.abb.com/public/5b7749c4cfd94348bc27c2ee4d1bc17f/3BTG811792-3027%20Functional%20Description%20-%20Mot01.pdf (MOT01 functional desc)
> - https://library.e.abb.com/public/dccb84483ca27155c12578da004a094c/3BDD012560R05_en_Freelance_9.2SP1_Getting_Started.pdf (Getting Started 9.2SP1)
> - https://new.abb.com/control-systems/essential-automation/freelance/system/freelance-overview
> - https://new.abb.com/control-systems/essential-automation/freelance/controller/AC900F
> - https://new.abb.com/control-systems/essential-automation/freelance/controller/ac800f
> - https://forum-controlsystems.abb.com/300611/advantage-of-DCS-system-by-ABB-compare-to-other-DCS (community comparison)
> - https://new.abb.com/news/detail/119541/abb-updates-freelance-distributed-control-system-to-boost-plant-efficiency-and-future-proof-operations (Freelance 2024 release)
> - https://www.automation.com/en-us/products/product02/abb-announces-release-of-freelance-2019-distribute (Freelance 2019 release)
> - https://help.plc.abb.com/AB270_en/a679dfd3014bb2657fc3d7ccfcca4dc2_8_en_us.html (Import/export functionality)

## Overview

ABB Freelance is a compact Distributed Control System (DCS) that combines the small physical footprint and cost profile of a PLC with the full operational scope of a DCS — pre-built process function blocks, integrated HMI, alarm management, and a single engineering tool for everything. It targets small-to-medium process plants: water/wastewater treatment, food & beverage, small chemical units, and OEM process packages where a full-scale ABB System 800xA deployment would be overengineered but a bare PLC lacks the DCS-style operator interface.

**Historical lineage:**
- Originally developed by Hartmann & Braun (H&B) as "Freelance 2000" in 1994
- ABB acquired H&B in 1997; renamed and re-engineered as "Freelance 800F"
- Controllers redesigned as AC 700F / AC 800F; engineering tool became Control Builder F (CBF)
- From Freelance 2013 onward, "800F" dropped from marketing name; hardware part numbers retain it

---

## System Architecture

### Topology

```
    [Field instruments / PROFIBUS slaves / FF devices / Modbus devices]
                         |
        ┌────────────────┼────────────────┐
   [AC 700F]        [AC 800F]        [AC 900F]    ← process controllers (1–16 per system)
        └────────────────┼────────────────┘
                         |
              [Ethernet system bus — 100 Mbit/s]
              /                               \
    [EWS / Control Builder F]         [DigiVis / Freelance Operations]
       (Engineering Workstation)             (Operator HMI)
```

All controllers communicate to:
1. The Engineering Workstation (EWS) running Control Builder F — for download/upload, online diagnostics, parameter changes
2. The Operator HMI running DigiVis (pre-2016) or Freelance Operations (2016+) — via the DIGI protocol for real-time display, alarm handling, and operator commands

There is **no proprietary system bus** between controllers: standard 100 Mbit/s Ethernet is the backbone for both engineering and runtime HMI communication.

### Key Software Components

| Component | Role | Notes |
|-----------|------|-------|
| **Control Builder F (CBF)** | Sole engineering tool: hardware config, IEC 61131-3 programming, variable list, download, diagnostics, DigiVis display design (integrated sub-editor) | Single installer, ~5 min setup |
| **DigiVis** | Operator HMI — standard faceplates, free graphic displays, alarm list, trend viewer | Up to Freelance 2013; 32-bit application |
| **Freelance Operations** | Replacement for DigiVis from Freelance 2016 | 64-bit, Windows 10/11, up to 4 monitors per station |
| **800xA for Freelance** | Optional: uses System 800xA as HMI layer over Freelance controllers | Adds cost/complexity; for large or multi-site installations |
| **OPC Gateway** | Built-in OPC DA/UA server in the Freelance runtime; used by third-party SCADA systems | DigiVis does NOT use OPC — it uses direct DIGI protocol |

---

## Controllers

### AC 700F — Entry Level

- DIN-rail mountable, minimal footprint
- Supports PROFIBUS-DP natively
- Up to 8 direct I/O modules in local rack
- No redundancy option
- Suitable for standalone machines or satellite I/O nodes

### AC 800F — Mid-Range Workhorse

The most widely deployed Freelance controller. Architecture: CPU module (PM 802F or PM 803F) forms the backplane onto which fieldbus modules and Ethernet modules are plugged in line.

**CPU variants:**

| Model | CPU | RAM | EPROM |
|-------|-----|-----|-------|
| PM 802F | Intel 80960HT RISC, 150 MIPS | 4 MB SRAM | 4 MB |
| PM 803F | Same CPU | 16 MB SDRAM (ECC) | 8 MB |

**Key specs (AC 800F):**
- 2 Ethernet module slots (EI 813F): 32-bit bus, 100 MByte/s
- 4 fieldbus module slots (can mix fieldbus types simultaneously)
- Typical I/O capacity: ~1,000 I/O signals
- Minimum cyclic task period: 5 ms
- Binary/16-bit arithmetic execution time: <1.0–2.0 ms
- Operating temperature: 0–60°C
- Dimensions: 239 × 202 × 164 mm; weight 1.6 kg (basic), up to 5 kg fully equipped
- Optional active-standby hot redundancy (requires second AC 800F + dedicated redundancy Ethernet link)

**Fieldbus modules:**

| Module | Protocol | Max Speed | Notes |
|--------|----------|-----------|-------|
| FI 810F | CAN | 1 Mbit/s | 3 channels; used for Freelance Rack I/O |
| FI 820F | RS-232/485/422, Modbus RTU | 115.2 kBit/s | Serial communications |
| FI 830F | PROFIBUS-DP master | 12 Mbit/s | Up to 126 slaves |
| EI 813F | Ethernet (IEEE 802.3) | 10–100 Mbit/s | System bus + OPC/DIGI |
| LD 800 HSE | Foundation Fieldbus HSE gateway | 100 Mbit/s | For FF H1 segments |

**Ethernet cabling distances (EI 813F):**
- UTP twisted pair: 400 m
- Thin coax (10Base2): 185 m
- Full coax (10Base5): 500 m
- Optical fibre: 4,500 m

### AC 900F — High-End Controller

**CPU variants:**

| Model | Designation | Typical I/O |
|-------|-------------|-------------|
| PM 901F | Lite | ~400 |
| PM 902F | Standard | ~1,500 |
| PM 904F | Plus | ~1,500 (higher throughput) |

**Additional features vs AC 800F:**
- More Ethernet ports (dedicated port per function)
- Pluggable SD card for program storage and cold backup
- Optional redundancy (active-standby; requires matching secondary unit with dedicated redundancy link)
- Supports PROFIBUS, Modbus, CAN, Foundation Fieldbus, PROFINET (from Freelance 2024)
- NAMUR Open Architecture (NOA) support added in Freelance 2024

---

## DIGI Protocol

### What It Is

"DIGI" is ABB Freelance's proprietary real-time communication protocol used between the DigiVis HMI (and Freelance Operations) and the controllers over standard Ethernet/IP. It is distinct from OPC: it uses a binary, low-latency ABB-proprietary frame format that gives DigiVis sub-second display refresh rates, alarm delivery, and operator command acknowledgement without OPC overhead.

Third-party systems cannot natively consume DIGI. They use the OPC DA/UA gateway instead.

Key operational facts:
- DigiVis identifies each controller by its configured **Node ID** (project-level integer), not directly by IP address (though the Node ID maps to an IP in the project hardware config)
- Communication is point-to-multipoint: one DigiVis station can communicate with multiple controllers simultaneously
- On redundant controllers, DigiVis automatically re-connects to the new primary within ~500 ms after a switchover

### Addressing Structure

Every process variable accessible from the HMI has a three-part DIGI address:

```
<NodeID> . <PointID> . <Suffix>
```

| Component | Range | Description |
|-----------|-------|-------------|
| **NodeID** | 1–254 | Logical controller identifier assigned in CBF hardware config. Maps to controller's Ethernet IP, but is the project-internal reference. |
| **PointID** | 1–65535 | Assigned to each variable when it is added to the variable list in CBF and flagged as HMI-visible. Stable across downloads as long as the variable is not deleted and re-added. |
| **Suffix** | String | Sub-element of a complex block instance (e.g., `.RUN`, `.FLR`, `.ILK`). See dmf-format.md for full suffix list. |

Example DIGI address: `5.1042.RUN` = controller Node 5, point 1042, suffix RUN.

### Tag Naming in CBF vs DIGI Addressing

In Control Builder F, variables have symbolic names (e.g., `P_101_RUN`). When the project is compiled, CBF generates an internal cross-reference table mapping each symbolic name to its NodeID.PointID pair. DigiVis stores this mapping in the `.dmf` database and uses the numeric address at runtime; the symbolic name is engineering-only.

### OPC Tag Format

For OPC DA clients (third-party SCADA):
```
<Application>.<POU>.<VariableName>
```
Example: `Application.Main.P_101_RUN`

---

## Function Block Library

ABB Freelance ships with a standard IEC 61131-3 function block library specifically designed for process control. These blocks are pre-wired to DigiVis standard faceplates.

### IDF_1 — Digital Input with Filtering

**Category:** Input conditioning
**Purpose:** Conditions a raw binary field input, applies configurable debounce filtering, inverts if needed, and generates an alarm output.

**Input pins:**

| Pin | Type | Description |
|-----|------|-------------|
| `IN` | BOOL | Raw field input (from I/O point) |
| `INV` | BOOL | Invert input logic (TRUE = complement the signal) |
| `FILTER_T` | TIME | Debounce filter time constant (e.g., `T#100ms`) |
| `ALM_EN` | BOOL | Enable alarm output |
| `SIM_EN` | BOOL | Simulation mode enable |
| `SIM_VAL` | BOOL | Simulated value (used when SIM_EN = TRUE) |

**Output pins:**

| Pin | Type | Description |
|-----|------|-------------|
| `OUT` | BOOL | Filtered (and optionally inverted) output |
| `ALM` | BOOL | Alarm active |
| `SIM_ACT` | BOOL | Simulation currently active |

**Behavior:**
- Input must be stable for `FILTER_T` before `OUT` changes state — prevents contact bounce propagation
- `FILTER_T = T#0ms` disables filtering; output follows input on the same scan cycle; on 5 ms tasks, single-scan glitches propagate
- When `SIM_EN = TRUE`, `OUT` = `SIM_VAL` regardless of field input; `SIM_ACT` = TRUE
- Simulation state is NOT preserved across a full download — resets to FALSE; teams operating in simulation mode risk unexpected state loss after maintenance downloads
- The `ALM` output semantics depend on configuration: in some CBF versions, ALM indicates the input is in an unexpected state relative to a reference; consult the CBF help for the specific version in use

### MOT_T1 — Single-Speed Motor Control

**Category:** Motor management
**Purpose:** Full-featured function block for a single-speed (on/off) motor or pump. Manages start/stop commands, monitors run feedback, handles interlock chains in two-level hierarchy, and generates standardized alarm outputs consumed by DigiVis motor faceplates.

**Input pins:**

| Pin | Type | Description |
|-----|------|-------------|
| `CMD_ON` | BOOL | Start command (from operator or sequence) |
| `CMD_OFF` | BOOL | Stop command |
| `RFB` | BOOL | Run feedback from motor contactor auxiliary contact |
| `FLT` | BOOL | Motor protection fault input (thermal trip, etc.) |
| `IC1` | BOOL | Safety interlock 1 — cannot be bypassed by operator |
| `IC2` | BOOL | Safety interlock 2 — cannot be bypassed by operator |
| `IB1`–`IB4` | BOOL | Process interlocks 1–4 — operator can acknowledge/bypass |
| `START_T` | TIME | Startup timeout: max time from CMD_ON until RFB must be TRUE |
| `STOP_T` | TIME | Stop timeout: max time from CMD_OFF until RFB must be FALSE |
| `MODE` | INT | Operating mode: 0=Auto, 1=Manual, 2=Off, 3=Test/Jog |

**Output pins:**

| Pin | Type | Description |
|-----|------|-------------|
| `RUN` | BOOL | Output relay command — energizes motor contactor |
| `FLR` | BOOL | Failure alarm: fault or start/stop timeout occurred |
| `ILK` | BOOL | Interlock alarm: start command blocked by active interlock |
| `RFB_ALM` | BOOL | Run feedback alarm: RFB disagrees with RUN command beyond a timeout |
| `STATE` | INT | Motor state: 0=Stopped, 1=Starting, 2=Running, 3=Stopping, 4=Fault |

**State machine:**
```
STOPPED ──(CMD_ON, interlocks clear)──► STARTING
STARTING ──(RFB=TRUE within START_T)──► RUNNING
STARTING ──(START_T elapsed, RFB still FALSE)──► FAULT  → FLR=TRUE
RUNNING ──(CMD_OFF)──► STOPPING
STOPPING ──(RFB=FALSE within STOP_T)──► STOPPED
STOPPING ──(STOP_T elapsed, RFB still TRUE)──► FAULT  → FLR=TRUE
RUNNING ──(FLT=TRUE)──► FAULT  → FLR=TRUE
FAULT ──(operator reset command)──► STOPPED
```

**Interlock hierarchy:**
- **Safety interlocks** (IC1, IC2): prevent start AND force trip if motor is running; operator cannot override at runtime
- **Process interlocks** (IB1–IB4): prevent start only; operator can acknowledge or bypass from the DigiVis motor faceplate depending on per-interlock configuration flags in CBF
- In Jog/Test mode (`MODE=3`): all process interlocks ignored; safety interlocks still apply

**DigiVis faceplate binding:**
MOT_T1 is bound to the standard "Motor" faceplate in DigiVis via the DMF variable suffixes:
`<TAG>.RUN`, `<TAG>.FLR`, `<TAG>.ILK`, `<TAG>.RFB_ALM`, `<TAG>.STATE`, `<TAG>.CMD_ON`, `<TAG>.CMD_OFF`

**Known corner cases:**
- `START_T` shorter than actual motor acceleration time generates spurious FLR on every start. Typical direct-on-line motor acceleration: 3–15 s. Default START_T in some CBF versions is `T#5s` — marginal for large centrifugal pumps.
- On redundant controller switchover: `MODE` is mirrored to the secondary and preserved, but the START_T timer restarts from zero. A motor in STARTING state at switchover time gets a fresh window.
- `FLT` wired to a digital input through IDF_1: if IDF_1's filter time is shorter than the motor protection relay's hold time, FLT may glitch back to FALSE after the relay resets, clearing FLR without operator acknowledgement. Best practice: use a latching IDF_1 output or use a separate SR flip-flop in the FBD to hold FLT until acknowledged.

### AIN_1 — Analog Input with Scaling and Alarming

**Category:** Analog input conditioning
**Purpose:** Reads raw analog value from fieldbus I/O, scales to engineering units, applies first-order filtering, performs four-level limit monitoring, generates alarm outputs.

**Input pins:**

| Pin | Type | Description |
|-----|------|-------------|
| `IN` | REAL | Raw value from I/O point (e.g., 4.0–20.0 for 4–20 mA) |
| `SCALE_LO` | REAL | Input raw value corresponding to EU_LO (typically 4.0) |
| `SCALE_HI` | REAL | Input raw value corresponding to EU_HI (typically 20.0) |
| `EU_LO` | REAL | Low end of engineering unit range (e.g., 0.0 bar) |
| `EU_HI` | REAL | High end of engineering unit range (e.g., 10.0 bar) |
| `FILTER_T` | TIME | First-order low-pass filter time constant |
| `HH_LIM` | REAL | High-high alarm limit (in EU) |
| `HI_LIM` | REAL | High alarm limit |
| `LO_LIM` | REAL | Low alarm limit |
| `LL_LIM` | REAL | Low-low alarm limit |
| `DB` | REAL | Deadband applied to all limit comparisons |
| `ALM_EN` | BOOL | Enable alarm outputs |
| `SIM_EN` | BOOL | Simulation mode enable |
| `SIM_VAL` | REAL | Simulated process value |

**Output pins:**

| Pin | Type | Description |
|-----|------|-------------|
| `OUT` | REAL | Scaled and filtered value in engineering units |
| `HH_ALM` | BOOL | High-high alarm active |
| `HI_ALM` | BOOL | High alarm active |
| `LO_ALM` | BOOL | Low alarm active |
| `LL_ALM` | BOOL | Low-low alarm active |
| `FAULT` | BOOL | Signal fault: input outside valid raw range |

**Scaling formula:**
```
OUT = EU_LO + ((IN - SCALE_LO) / (SCALE_HI - SCALE_LO)) * (EU_HI - EU_LO)
```

**Alarm logic with deadband:**
Alarm activates when `OUT` crosses the limit. Alarm clears only when `OUT` moves back past `(limit ± DB)`, preventing chatter at the threshold.

**Known corner cases:**
- `FAULT` detection relies on the fieldbus driver reporting a bad-quality status byte. On PROFIBUS-DP with non-ABB slaves that implement the status byte incorrectly, `FAULT` may never activate even with a broken signal cable.
- When `FILTER_T` is long (e.g., `T#30s`), the filtered output lags significantly behind real process changes. Alarm limits evaluated against the filtered output may not catch fast transients. Some applications run a second AIN_1 instance with no filter on the same raw input for alarm purposes only.

### PID_1 — PID Controller

**Category:** Control
**Purpose:** Standard ISA-style PID with anti-windup, bumpless manual-to-auto transfer, external setpoint tracking, and output clamping.

**Input pins:**

| Pin | Type | Description |
|-----|------|-------------|
| `PV` | REAL | Process variable (measured value) |
| `SP` | REAL | Setpoint |
| `FF` | REAL | Feed-forward value added to PID output |
| `KP` | REAL | Proportional gain |
| `TI` | TIME | Integral time (reset time). `T#0s` disables integral |
| `TD` | TIME | Derivative time. `T#0s` disables derivative |
| `OUT_LO` | REAL | Output low clamp |
| `OUT_HI` | REAL | Output high clamp |
| `MODE` | INT | 0=Auto, 1=Manual, 2=Tracking |
| `MAN_VAL` | REAL | Manual output value |
| `TRK_VAL` | REAL | Tracking output value (for cascade slave) |

**Output pins:**

| Pin | Type | Description |
|-----|------|-------------|
| `OUT` | REAL | Controller output |
| `ERR` | REAL | Control error = SP − PV |
| `SP_ACT` | REAL | Active setpoint (equal to SP in non-ramp mode) |

**Algorithm (velocity form / positional form depending on CBF version):**
```
OUT(k) = KP * ERR(k)
       + KP * (Ts / TI) * Σ ERR
       + KP * (TD / Ts) * (ERR(k) − ERR(k-1))
       + FF
```
Where Ts = task cycle time.

**Anti-windup:** Integrator freezes when output is clamped. On switch from Manual to Auto: integral term is pre-loaded from `MAN_VAL` to achieve bumpless transfer.

**Known corner cases:**
- `TI = T#0s` disables integral; `TI = T#99999s` approximates P-only but integrator still runs (very slowly). Behavior differs at output limits: `T#0s` has no windup at all; `T#99999s` accumulates a negligible ramp.
- In Tracking mode (`MODE=2`): `OUT = TRK_VAL`; integral tracks `TRK_VAL` so that when Auto mode is engaged, no bump occurs. Used for cascade master-slave arrangements.
- Derivative on measurement vs derivative on error: the specific implementation (derivative on `ERR` vs derivative on `-PV`) depends on CBF version. Derivative on error causes a bump on SP step changes; derivative on measurement does not. Check version-specific documentation.

---

## IEC 61131-3 Programming Model

```
Project
  └─ Resource (= one physical controller, e.g., AC800F_Node05)
       ├─ Task Configuration
       │    ├─ FastTask:  CYCLIC T#100ms, Priority 1
       │    └─ SlowTask:  CYCLIC T#1000ms, Priority 5
       └─ Programs (POUs of type PROGRAM)
            ├─ MotorControl  (assigned to FastTask)
            └─ Analytics     (assigned to SlowTask)
                 └─ [FB instances: MOT_T1, AIN_1, PID_1, IDF_1...]
```

**Supported IEC 61131-3 languages:** FBD (Function Block Diagram), LD (Ladder Diagram), SFC (Sequential Function Chart), ST (Structured Text), IL (Instruction List)

**Task configuration rules:**
- Priority range: 0 (highest real-time) to 15 (lowest real-time); priority 16 = non-real-time background
- Minimum cyclic interval: 5 ms on AC 800F/900F
- Freewheeling task: executes as fast as CPU allows; no fixed period
- Each task has an independent watchdog; violation triggers controller fault event
- **CPU load protection:** if aggregate application demand exceeds 70% of CPU capacity, the task scheduler automatically increases all cyclic intervals proportionally to re-establish 70% load — this self-protection mechanism can cause unexpected task slowdowns when projects grow; it produces no explicit warning in the operator interface, only observable via the CBF diagnostics page

---

## Engineering Workflow — File-Based System

ABB Freelance is fundamentally a **file-based engineering system**. The project is a directory tree on the EWS.

### Project File Types

| Extension | Role |
|-----------|------|
| `.prj` | Project descriptor (XML-ish): project name, controller list, CBF version |
| `.prt` | Parameter Record Table: the main project database — hardware config, all variables, FB parameter values, task configuration (see prt-format.md) |
| `.dmf` | Display Module Format: DigiVis/Operations display database — all graphic displays, faceplate bindings (see dmf-format.md) |
| `.eam` | Variable list export/import format: CSV-like, columns = Name / Comment / Type / Reserved / X / Object / Location / PointID |
| `.ews` | Editor workspace state (window positions/sizes); safe to delete |
| `.bak` | Automatic backup of `.prt` before each compile |

### Engineering Steps

1. Create/open project in CBF → creates `.prj` + empty `.prt`
2. Hardware configuration: add controllers (assign Node IDs), I/O modules, fieldbus slaves
3. IEC programming: draw FBD/LD pages or write ST; instantiate standard FB library blocks
4. Variable list: assign point IDs and HMI-visible flags; optionally export `.eam` for bulk editing in Excel, then re-import
5. **Compile** (Build Project): CBF parses all POUs, generates downloadable binary, updates `.prt` with checksum
6. **Download** to controller over Ethernet; controller briefly stops (cold download by default)
7. DigiVis display configuration: in CBF's integrated editor, bind tags to standard faceplates or free graphic display elements → updates `.dmf`
8. **Download displays** to DigiVis/Operations workstation

**Bulk engineering pattern (large I/O lists):**
For projects with hundreds of motors or instruments, the standard workflow is:
- Maintain a master tag list in Excel with columns matching `.eam` format
- Export skeleton `.eam` from CBF for structure reference
- Populate in Excel; save as `.eam` (CSV with correct column order)
- Import into CBF → CBF creates all variables with assigned point IDs
- Regenerate `.prt` → `Build` → `Download`

---

## Comparison with Other Compact DCS

| Feature | ABB Freelance | Siemens PCS 7 | Yokogawa CENTUM VP | Emerson DeltaV |
|---------|--------------|---------------|---------------------|----------------|
| Min practical I/O | ~50 | ~200 (PCS 7 overhead) | ~100 | ~100 |
| Engineering tools | 1 (CBF — everything) | Multiple (SIMATIC Mgr + WinCC) | CENTUM Engineering | DeltaV Explorer |
| HMI integration | Tightly integrated in CBF | Separate WinCC | Integrated | Integrated |
| Multi-controller | Yes (Ethernet) | Yes (Industrial Ethernet) | Yes | Yes |
| Redundancy | Optional (AC 800F / AC 900F) | Optional | Standard on CENTUM VP | Optional |
| IEC 61131-3 | All 5 languages | FBD/LD/SFC/STL/SCL | FBD, LD | FBD, LD, SFC |
| Native fieldbus | PROFIBUS, FF, Modbus, CAN | PROFIBUS, PROFINET | Foundation Fieldbus | Foundation Fieldbus |
| SIL certification | No native SIL | SIL 2 with F-CPU | SIL 2 (ProSafe-RS) | SIL 2 (SIS) |
| Install time | ~5 min | Hours | Hours | Hours |
| Target market | Small-medium process | Medium-large | Medium-large | Medium-large |

**ABB Freelance advantages (from practitioner community):**
- Single tool covers controller programming AND HMI — no separate WinCC-like product
- DTM integration not licensed per field device (significant cost advantage for instrument-heavy plants)
- DEMO mode: entire system emulation without hardware; useful for FAT
- Low learning curve: typical engineer productive within 1 week per ABB's own claim
- AC 700F very low-cost entry point for small OEM packages

**Known limitations:**
- DigiVis graphics are less flexible than WinCC/AVEVA/Wonderware for complex custom displays
- Maximum ~16 controllers per Freelance Ethernet broadcast domain (practical; not a hard protocol limit)
- No native SIL safety certification — safety functions require a separate SIL-certified safety PLC
- 800xA overlay (for larger HMI needs) adds major cost and setup complexity; it is essentially a different product
- Project files not natively version-control-friendly (binary sections in `.prt`); teams use manual folder backup or dedicated tools

---

## Version History

| Version | Year | Key Changes |
|---------|------|-------------|
| Freelance 2000 | 1994–2000 | Original H&B product; DigiTool as engineering tool; DigiVis HMI; DIGI protocol over serial RS-485 |
| 800F v8.x | ~2001–2008 | New AC 700F / AC 800F hardware; Control Builder F replaces DigiTool; Ethernet backbone; Foundation Fieldbus support |
| 800F v9.1 | 2009 | Improved system emulator (Control Emulator); Windows XP certified; improved PROFIBUS-DP diagnostics |
| 800F v9.2 / 9.2SP1 | 2010–2012 | Foundation Fieldbus H1 full support; improved redundancy management; "Getting Started" guide baseline for community |
| Freelance 2013 | 2013 | Marketing rename from "800F"; AC 900F controller introduced; DigiVis updated; improved faceplate library; Windows 7 certified |
| Freelance 2016 | 2016 | DigiVis renamed "Freelance Operations"; full Windows 10 support; up to 4 monitors per operator workplace; TLS/SSL security for communications; PROFINET preview; improved usability in Operations |
| Freelance 2019 | 2019 | Mixed Windows 10 + Windows 7 in same system; extended Windows domain account management; advanced navigation/filter/sort in Operations; improved HMI performance; enhanced system emulator |
| Freelance 2024 | 2024 | Windows 11 + MS Defender integration; full PROFINET support; Ethernet APL (2-wire Ethernet to field devices); Module Type Packages (MTP) for modular automation plug-and-produce; NAMUR Open Architecture (NOA) monitoring channel support; faster Ethernet data transfer |

---

## Known Corner Cases and Limitations

1. **Cold download stops execution**: A full project download stops the controller, replaces the application, then restarts. Plant must be prepared for a ~5–30 second interruption depending on project size and controller model. Hot parameter patching (online write of FB parameters without stop) is possible for parameters that don't change program structure, but requires careful use of CBF's online mode.

2. **Variable list size ceiling**: The HMI-visible variable list has a practical per-controller limit of ~4,000 exported points to DigiVis/Operations. Exceeding this causes silent truncation: no compile error or warning is raised; only at runtime will DigiVis fail to find certain tags, showing "???" in display fields.

3. **Task jitter on AC 700F**: The AC 700F has no hardware separation between I/O scan and task execution. Heavy PROFIBUS-DP polling with large slave counts introduces ±3–5 ms jitter on nominally fast cyclic tasks.

4. **PROFIBUS-DP restart on download**: When a new application is downloaded to AC 800F, the PROFIBUS-DP master module re-initializes. All DP slaves briefly lose communication (~2–5 s). Vibration transmitters and certain smart instruments take additional time to re-establish DP communication after the bus re-initializes.

5. **Redundancy switchover gap**: AC 800F/900F active-standby switchover completes in <100 ms (controller pair). However, the DIGI protocol session between DigiVis and the controller must re-establish against the new primary, adding a communication blackout at the HMI of up to ~500 ms. During this window, DigiVis displays show last-known values and alarms may queue.

6. **EAM import conflict**: If two engineers import `.eam` files with overlapping PointID ranges, CBF silently accepts the last import, overwriting the first. No merge conflict detection. Large projects must maintain a PointID allocation spreadsheet to avoid collisions.

7. **DigiVis 32-bit only (pre-2016)**: DigiVis up to Freelance 2013 is a 32-bit application. Requires WoW64 compatibility on 64-bit OS. Cannot be installed on stripped 64-bit server images that have WoW64 disabled.

8. **Non-ASCII characters in tag names**: CBF tag names must be pure ASCII (A–Z, 0–9, underscore). Non-ASCII characters from OS AutoCorrect (e.g., typographic apostrophes, umlauts via locale) inserted into CBF text fields can silently corrupt the `.prt` file on save. The file opens normally afterward but produces cryptic compile errors.

9. **Simulation state lost on download**: All FB instances with `SIM_EN` inputs reset SIM_EN to FALSE on a full project download. Plants running in simulation mode for training or commissioning must re-enable simulation on every download cycle.

10. **CPU load self-protection non-transparent**: When the 70% CPU load ceiling is hit and task intervals auto-increase, there is no alarm or event log entry in the standard operator interface. Engineers discover it only by observing that analog values update slower than expected or by checking the CBF diagnostics screen online.
