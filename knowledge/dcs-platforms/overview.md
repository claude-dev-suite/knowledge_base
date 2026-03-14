# DCS Platforms - Comparative Overview

> Reference: ABB Freelance, Siemens PCS7/TIA Portal, Emerson DeltaV, Honeywell Experion PKS vendor documentation

## Overview

This document provides a comparative reference for the four major DCS/PLC platforms used in process automation: ABB Freelance, Siemens PCS 7, Emerson DeltaV, and Honeywell Experion PKS. The focus is on function block mapping, bulk engineering capabilities, and file format strategies relevant to programmatic generation of control system configurations.

---

## Table of Contents

1. [Platform Summary Table](#platform-summary-table)
2. [Cross-Platform Motor Block Mapping](#cross-platform-motor-block-mapping)
3. [Cross-Platform Function Block Mapping](#cross-platform-function-block-mapping)
4. [IEC 61131-3 Language Support](#iec-61131-3-language-support)
5. [Bulk Engineering Comparison](#bulk-engineering-comparison)
6. [Platform Selection Guide](#platform-selection-guide)
7. [Engineering File Formats](#engineering-file-formats)

---

## Platform Summary Table

| Platform | Vendor | Primary Use Case | Engineering Tool | HMI System |
|----------|--------|-----------------|-----------------|------------|
| Freelance | ABB | Small-mid process plants | Control Builder F / EWP | DigiVis |
| PCS 7 | Siemens | Large continuous process | SIMATIC Manager + CFC/SFC editor | WinCC Classic |
| TIA Portal | Siemens | PLC/HMI unified (S7-1200/1500) | TIA Portal V17+ | WinCC Unified |
| DeltaV | Emerson | Process DCS, batch, pharma | DeltaV Explorer + Control Studio | DeltaV Operate / Live |
| Experion PKS | Honeywell | Large process, oil & gas | Control Builder | HMIWeb Display Builder |

**Key distinguishing characteristics:**

- **ABB Freelance**: file-based engineering (no API), compact footprint, suited for 200-1000 I/O
- **Siemens PCS 7**: mature large-scale DCS, strong CFC/SFC, TIA Openness API for automation
- **Emerson DeltaV**: class-based configuration is the most powerful template mechanism among DCS platforms; ISA-88 native
- **Honeywell Experion**: large installations, oil & gas, CL (Control Language) proprietary scripting

---

## Cross-Platform Motor Block Mapping

Understanding how the same physical motor is represented across platforms is essential for migration projects and cross-platform documentation.

| Concept | ABB Freelance | Siemens PCS 7 | Emerson DeltaV | Honeywell Experion |
|---------|--------------|---------------|----------------|---------------------|
| Block name | `IDF_1` / `MOT_T1` | `MotL` (APL library) | `DC` (Device Control) | `MOTOR` |
| Library | BST_LIB_EXT / BST_USER_FB | APL (Advanced Process Library) | DeltaV standard | Honeywell standard |
| Start input | `AC` (auto command) | `CmdStrt` | `SP_D` (setpoint discrete) | `SP` |
| Stop input | via `MM` / manual | `CmdStop` | `SP_D` = STOP | `SP` = STOP |
| Run feedback | `RUN` | `FbkRun` | `IN_D1` | `FB_RUN` |
| Stopped feedback | (derived from RUN=0) | `FbkStp` | `IN_D2` | `FB_STOP` |
| Fault / trip | `FLR` | `Protect` / `Trip` | `IN_D3` | `FB_TRIP` |
| Available | `PR0` (protection) | `Intlock01` | `IN_D4` | `INTLK` |
| Interlock | `ILK` (output) | `Intlock01` (input) | `INTERLOCK` | `INTLK` |
| Auto mode | `AUT` (output) / `MA` (input) | `AutModOp` | `MODE_BLK.TARGET = AUTO` | `MODE` |
| Manual mode | `MM` (input) | `ManModOp` | `MODE_BLK.TARGET = MAN` | `MODE` |
| Local/Remote | `LOC` (output) | `ModLiOp` | `IO_OPTS` | `LOCAL` |
| Monitor time | `Lz` param (T#5s typical) | `MonTiRun` / `MonTiStp` | `FBK_TIMEOUT` | `FB_TIME` |
| Shutdown | `SDS` (output) | (via Trip chain) | (via Interlock chain) | (via Trip) |
| Data structure | `DBS:RECORD (MOTOR1)` | Instance DB | Module parameters | Point parameters |

### ABB Freelance Motor Signals in Detail

```
IDF_1 / MOT_T1 - Single Speed Motor:

Inputs (from process/logic):
  AC   - Automatic start command
  PR0  - Safety protection OK (permissive)
  MA   - Operator request: switch to Auto
  MM   - Operator request: switch to Manual

Outputs (to process/HMI):
  ILK  - Interlock active (motor blocked)
  RDY  - Ready to start
  RUN  - Running confirmed
  LOC  - Local mode active
  AUT  - Auto mode active
  FLR  - Fault/trip active
  SDS  - Shutdown active
```

### Siemens PCS 7 MotL Block in Detail

```
MotL - APL Motor Linear:

Key Inputs:
  ModLiOp   - Mode local/remote operation
  AutModOp  - Auto mode operation request
  ManModOp  - Manual mode operation request
  FbkRun    - Running feedback from field
  Protect   - Protection input (trip)
  Intlock01 - Interlock input 1 (of up to 16)
  CmdStrt   - Start command
  CmdStop   - Stop command
  MonTiRun  - Monitor time for run feedback
  MonTiStp  - Monitor time for stop feedback

Key Outputs:
  ModLiOp_Out - Current mode output
  StartOut    - Start output to field
  RunOut      - Run status
  StopOut     - Stop output to field
  Trip        - Trip/fault output
  SwiToLoc    - Switched to local
```

### Emerson DeltaV DC Block in Detail

```
DC - Device Control:

Inputs:
  MODE_BLK.TARGET - Mode: AUTO, MAN, OOS
  IN_D1           - Running feedback
  IN_D2           - Stopped feedback
  IN_D3           - Fault/trip feedback
  IN_D4           - Available feedback
  INTERLOCK       - Interlock signal
  PERMIT          - Permit to run

Outputs:
  OUT_D1  - Start output
  OUT_D2  - Stop output
  SP_D    - Setpoint (discrete)
  PV_D    - Process value (discrete)

Configuration:
  IO_OPTS       - I/O options flags
  FBK_STRATEGY  - Feedback strategy (1-4)
  FBK_TIMEOUT   - Monitor timeout
```

---

## Cross-Platform Function Block Mapping

| Function | ABB Freelance | Siemens PCS 7 | Emerson DeltaV | Honeywell Experion |
|----------|--------------|---------------|----------------|---------------------|
| Motor (1-speed) | `IDF_1` / `MOT_T1` | `MotL` | `MOTOR_BASIC` / `DC` | `MOTOR` |
| Motor (reversing) | `IDF_2` (if available) | `MotR` | `DC` with config | `DMOTOR` |
| Motor (variable speed) | custom / `IDF_1` + AO | `MotSpdL` | `DC` + `AO` | `VMOTOR` |
| On/Off Valve | `VLV_1` | `Valve` | `DEVICE_DISCR` | `DVALVE` |
| Modulating Valve | `VLV_2` | `AnlgValve` | `DEVICE_AO` | `AVALVE` |
| PID Controller | `PID` | `PIDConL` | `PID_PLUS` | `PID` |
| Analog Input | `M_ANA` | `MEAS_MON` | `AI` | `AI` |
| Digital Input | `M_BIN` | `MON_DIGI` | `DI` | `DI` |
| Analog Output | `M_AOUT` | `OUT_MON` | `AO` | `AO` |
| Digital Output | `M_BOUT` | `OUT_DIGI` | `DO` | `DO` |
| Timer (on-delay) | `TON` | `TON` | `TONR` | `TIMER_ON` |
| Counter | `CTU` | `CTU` | `CTUD` | `COUNTER` |
| Calculation | `CALC` / ST block | `ARITH` | `CALC` | `CALC` |
| Interlock logic | LAD contacts | `Intlk02/04/08/16` | `INTLK_OUT` | `LOGIC_CALC` |

---

## IEC 61131-3 Language Support

| Language | ABB Freelance | Siemens PCS7/TIA | Emerson DeltaV | Honeywell | Schneider |
|----------|:------------:|:----------------:|:--------------:|:---------:|:---------:|
| LD (Ladder) | Yes | Yes (KOP) | No | Via ControlEdge | Yes |
| FBD | Yes (primary) | Yes (FUP/CFC) | Yes (primary) | Yes | Yes |
| ST | Yes | Yes (SCL) | Partial (CALC) | CL (similar) | Yes |
| IL | No | Yes (AWL) - legacy | No | No | Yes (deprecated) |
| SFC | Limited | Yes (S7-GRAPH) | Yes (phases) | Yes | Yes |
| OOP | No | TIA V16+ | No | No | No |
| PLCopen XML | No | Partial | No | No | Yes |

Notes:
- Freelance's ST is available but FBD is the primary and recommended language
- Siemens AWL (Instruction List) is considered legacy; SCL (ST) is preferred in TIA Portal
- DeltaV CALC blocks allow limited expression-based logic; full ST is not available
- PLCopen XML (TC6) is the only vendor-neutral exchange format; only supported by CODESYS-based systems

---

## Bulk Engineering Comparison

| Approach | ABB Freelance | Siemens TIA Portal | Emerson DeltaV | Honeywell Experion |
|----------|:------------:|:-----------------:|:--------------:|:-----------------:|
| Primary method | PRT file templating | Openness API (.NET) | Class-based config + Bulk Edit | CSV point import |
| Secondary method | Full CSV manipulation | SimaticML XML import | FHX file generation | COM API |
| File format | PRT/CSV (semicolon-delimited, UTF-16LE) | XML (SimaticML, proprietary schema) | FHX (text, proprietary) | CSV |
| API availability | None (file-based only) | TIA Openness (.NET library) | DeltaV COM API | Experion COM API |
| Scripting language | Python (file manipulation) | C# / VB.NET | Python + FHX text gen | VBScript |
| HMI generation | DMF file templating | SiVArc (rules-based auto-gen) | Auto-faceplates from modules | Template display export |
| Template propagation | Manual (re-generate all) | Re-import block XML | Automatic (class → instances) | Manual |
| Change management | Replace + reimport PRT | Openness API update | Class change → all instances | CSV re-import |

### Bulk Engineering Effort Estimate

For a typical mid-size plant (200 motors, 100 valves, 500 analog loops):

| Platform | Method | Estimated Setup | Per-Instance Time | Total Estimate |
|----------|--------|----------------|-------------------|----------------|
| ABB Freelance | PRT templating | 2-3 days | 30 sec (automated) | 3-5 days |
| Siemens TIA | Openness API | 3-5 days (C# dev) | 5 sec (automated) | 4-6 days |
| Emerson DeltaV | Class-based | 1-2 days | Auto (class instantiation) | 2-3 days |
| Honeywell | CSV import | 1 day | 1 min (Excel prep) | 2-3 days |

---

## Platform Selection Guide

### Choose ABB Freelance when:
- Plant size is 200-1000 I/O points
- Budget is a constraint (lower licensing cost than DeltaV/Honeywell)
- Industries: cement, mining, water/wastewater, food & beverage
- Existing Freelance infrastructure on site
- Engineering team is comfortable with file-based tooling

### Choose Siemens PCS 7 or TIA Portal when:
- Plant size is large (1000+ I/O)
- Strong ISA-88 sequence requirements with S7-GRAPH (SFC)
- Integration with Siemens SCADA (WinCC) is required
- .NET automation capability is available in engineering team
- Industries: chemical, pharmaceutical, large utilities

### Choose Emerson DeltaV when:
- Batch manufacturing with ISA-88 compliance required
- Template/class-based change management is critical (pharmaceutical, specialty chemical)
- Built-in alarm management (ISA-18.2) and audit trail required
- Industries: pharma, biotech, food & beverage, oil & gas

### Choose Honeywell Experion when:
- Very large installations (oil refinery, LNG plant)
- Integration with Honeywell Safety Manager (SIS) required
- Long-term support contracts preferred
- Industries: oil & gas, petrochemical, power generation

---

## Engineering File Formats

| Platform | File Type | Extension | Format | Encoding |
|----------|-----------|-----------|--------|---------|
| ABB Freelance | Partial project | `.prt` | Semicolon-delimited sections | UTF-16LE with BOM |
| ABB Freelance | Full project | `.csv` | Semicolon-delimited sections | UTF-16LE with BOM |
| ABB Freelance | HMI display | `.DMF` | Custom text format | UTF-16LE with BOM |
| Siemens PCS 7 | Project | `.s7p` / `.s7l` | Binary / proprietary | N/A |
| Siemens TIA | Block export | `.xml` | SimaticML (proprietary XML) | UTF-8 |
| Siemens TIA | AutomationML | `.aml` | AML/XML (IEC 62714) | UTF-8 |
| Siemens | ST source | `.scl` | Plain text | UTF-8 |
| Siemens | IL source | `.awl` | Plain text | UTF-8 |
| Siemens | WinCC display | `.pdl` | Binary | N/A |
| Emerson DeltaV | Config export | `.fhx` | Proprietary text | ASCII/UTF-8 |
| Emerson DeltaV | Bulk edit | `.xlsx` | Excel | N/A |
| Honeywell | Point config | `.csv` | Comma-delimited | ASCII |
