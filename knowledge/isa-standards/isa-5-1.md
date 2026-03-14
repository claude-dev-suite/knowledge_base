# ISA Standards - ISA-5.1 Instrument Tagging

> Sources:
> - https://www.engineeringtoolbox.com/isa-intrumentation-codes-d_415.html
> - https://instrunexus.com/isa-5-1-instrumentation-symbols-and-identifications-detailed-analysis/
> - https://instrumentationandcontrol.net/pid-diagram-basics-part-3-functional-identification.html
> - https://control.com/textbook/instrumentation-documents/instrument-identification-tags/
> - https://blog.ansi.org/ansi/ansi-isa-5-1-2024-instrumentation-symbols/
> - https://visaya.solutions/en/article/pid-and-the-isa-5-1
> - https://control.com/forums/threads/isa-loop-tagging-vs-equipment-tagging.1311/

## Overview

ISA-5.1 (currently ANSI/ISA-5.1-2009, with a 2024 revision) is the standard for instrument identification and tagging used in process plants worldwide. It defines how to assign unique tag numbers to instruments and control system elements in a consistent, unambiguous way. Every instrument, control loop, and piece of process equipment receives a tag built from a defined letter combination that describes its measured variable and function.

The standard uses a **functional** identification principle: a device is tagged based on what it measures or controls, not how it is built. A level transmitter that uses differential pressure measurement is tagged `LT` (Level Transmitter), not `PDT`, because it serves a level measurement function in the control loop.

Mastering ISA-5.1 is fundamental to DCS/PLC engineering because tag names are the primary keys for all configuration data, alarm databases, P&ID references, and historian entries.

---

## Table of Contents

1. [Tag Number Structure](#tag-number-structure)
2. [First Letter Table (Measured Variables A–Z)](#first-letter-table)
3. [Succeeding Letters Table (Device Functions)](#succeeding-letters-table)
4. [Common Tag Combinations by Variable Type](#common-tag-combinations-by-variable-type)
5. [Equipment Tags (Non-ISA Instrument)](#equipment-tags-non-isa)
6. [Redundancy Suffixes (A/B/C for Dual/Triple)](#redundancy-suffixes)
7. [Loop Numbering Conventions](#loop-numbering-conventions)
8. [P&ID Symbol Conventions](#pid-symbol-conventions)
9. [Corner Cases](#corner-cases)
10. [Regional and Vendor Notation](#regional-and-vendor-notation)
11. [Tag Naming Validation in Python](#tag-naming-validation-in-python)

---

## Tag Number Structure

### Standard ISA format

```
Full tag = [Area Prefix -] [Functional ID] - [Loop Number] [Suffix]

Functional ID = [First Letter] + [Succeeding Letters...]
```

**Anatomy example:**

```
10 - P  I  C - 101 A
│    │  │  │   │   └── Suffix: parallel instance within the loop
│    │  │  │   └────── Loop number: 101
│    │  │  └────────── Succeeding letter 2: Controller (C)
│    │  └───────────── Succeeding letter 1: Indicate (I)
│    └──────────────── First letter: Pressure (P)
└───────────────────── Optional area prefix: area 10

Result: PIC-101A = Pressure Indicating Controller, loop 101, train A
```

### Tag component rules

| Component | Required? | Format | Notes |
|-----------|-----------|--------|-------|
| Area prefix | Optional | Numeric (e.g., `10`, `11301`) | Identifies plant unit/area |
| Functional ID | Required | 1–5 uppercase letters | First letter = variable; remaining = functions |
| Loop number | Required | 3–5 digits (e.g., `101`, `0056`) | Unique within area/plant |
| Suffix | Optional | Letter(s) (e.g., `A`, `B`, `AB`) | Parallel instruments in same loop |

### Loop numbering conventions

| Convention | Format | Example | When Used |
|------------|--------|---------|-----------|
| Sequential in area | 001–999 | `FIC-001` | Small projects |
| Area-prefixed | AREA + 3 digits | `FIC-11301` | Large plants with many areas |
| P&ID-based | Sheet + item | `FIC-103-07` | Projects tied to drawing numbers |
| ISA recommended | 3–4 digits | `FIC-056` | Standard practice |

---

## First Letter Table

The first letter identifies the **measured or initiating variable** — what the instrument measures or what variable initiates the control action.

| Letter | Primary Measured Variable | As First-Letter Modifier |
|--------|--------------------------|--------------------------|
| **A** | Analysis (pH, O₂, CO₂, conductivity, chromatograph, TOC) | — |
| **B** | Burner / Combustion | — |
| **C** | User's Choice (often conductivity in water treatment) | — |
| **D** | User's Choice (often density) | Differential (FD = flow differential) |
| **E** | Voltage (EMF, electrical) | — |
| **F** | Flow Rate | Ratio / Fraction (FF = flow ratio) |
| **G** | User's Choice (gauging/gaging) | — |
| **H** | Hand (manual initiation — e.g., pushbutton) | — |
| **I** | Current (electrical, in Amperes) | — |
| **J** | Power (kW, kWh) | Scan (scanning/multiplexing) |
| **K** | Time / Time Schedule | Time Rate of Change |
| **L** | Level | — |
| **M** | User's Choice (often moisture, mass flow, or torque) | Middle / Intermediate |
| **N** | User's Choice | — |
| **O** | User's Choice | Orifice / Restriction |
| **P** | Pressure / Vacuum | Point (test connection) |
| **Q** | Quantity (accumulated volume or mass) | Integrate / Totalize |
| **R** | Radiation (nuclear) | — |
| **S** | Speed / Frequency (RPM, Hz) | Switch (when used as modifier) / Safety |
| **T** | Temperature | — |
| **U** | Multivariable (two or more variables) | Multifunction |
| **V** | Vibration / Mechanical Analysis | — |
| **W** | Weight / Force | Well / Probe |
| **X** | Unclassified (user-defined per project Legend Sheet) | Unclassified |
| **Y** | Event / State / Presence | Relay / Compute / Convert |
| **Z** | Position / Dimension | Actuator / Driver |

**User's Choice letters (B, C, D, G, M, N, O, X):** These must be formally defined on the project Legend Sheet. Common industry assignments:
- `C` = Conductivity (water/wastewater plants)
- `D` = Density (oil & gas)
- `M` = Moisture (dryers, kilns)
- `N` = Torque (rotating equipment)

---

## Succeeding Letters Table

Succeeding letters describe the **function** of the instrument within the control loop.

### Readout / Passive functions

| Letter | Function | Example |
|--------|----------|---------|
| **A** | Alarm | `TAH` = Temperature Alarm High |
| **E** | Element / Sensor | `TE` = Temperature Element (thermocouple, RTD) |
| **G** | Glass / Gauge / Viewing Device | `LG` = Level Glass (sight glass) |
| **I** | Indicate (display) | `TI` = Temperature Indicator |
| **L** | Light (pilot lamp) | `XL` = Indicator Light |
| **O** | Orifice / Restriction | `FO` = Flow Orifice (primary element) |
| **P** | Point (test connection) | `PP` = Pressure Point |
| **R** | Record (historian, chart recorder) | `FR` = Flow Recorder |
| **U** | Multifunction | `FU` = Flow Multifunction |
| **W** | Well / Probe | `TW` = Temperature Well (thermowell) |

### Output / Active functions

| Letter | Function | Example |
|--------|----------|---------|
| **C** | Controller | `FIC` = Flow Indicating Controller |
| **K** | Control Station (manual loader) | `PK` = Pressure Control Station |
| **S** | Switch | `PSH` = Pressure Switch High |
| **T** | Transmit (signal to remote) | `FT` = Flow Transmitter |
| **V** | Valve / Damper / Louver | `FCV` = Flow Control Valve |
| **Y** | Relay / Compute / Convert | `FY` = Flow Relay or I/P converter |
| **Z** | Driver / Actuator / Final Element | `PZ` = Pressure Driver |

### Function modifiers

| Letter | Modifier | Example |
|--------|----------|---------|
| **D** | Deviation | `PDIC` = Pressure Differential Indicating Controller |
| **F** | Ratio | `FFC` = Flow Ratio Controller |
| **H** | High | `PAH` = Pressure Alarm High |
| **H** (doubled) | High-High (more severe) | `PAHH` = Pressure Alarm High-High |
| **L** | Low | `PAL` = Pressure Alarm Low |
| **L** (doubled) | Low-Low (more severe) | `PALL` = Pressure Alarm Low-Low |
| **M** | Middle / Intermediate | `PAM` = Pressure Alarm Middle |

**Note on HH/LL:** The doubled letter (`HH`, `LL`) indicates a second, more severe level. In SIL-rated systems, `PAHH` and `PSHH` typically trigger Safety Instrumented Function (SIF) actions. `PAH` triggers process alarms for operator response; `PAHH` triggers automatic shutdowns.

---

## Common Tag Combinations by Variable Type

### Flow (F\_)

| Tag | Full Name | Typical Signal | Description |
|-----|-----------|---------------|-------------|
| `FE` | Flow Element | None (mechanical) | Orifice plate, venturi, pitot tube — primary element only |
| `FT` | Flow Transmitter | 4–20 mA / HART | Differential pressure cell, Coriolis, magnetic flowmeter |
| `FI` | Flow Indicator | — | Display only (no control) |
| `FIC` | Flow Indicating Controller | — | PID flow controller with display in DCS |
| `FCV` | Flow Control Valve | Pneumatic (3–15 psi) | Actuated control valve — final element |
| `FR` | Flow Recorder | — | Strip chart or historian |
| `FQI` | Flow Quantity Indicator | — | Batch totalizer display |
| `FQT` | Flow Quantity Transmitter | 4–20 mA | Totalizer with remote output |
| `FSH` | Flow Switch High | Discrete (24VDC/relay) | Closes on high flow |
| `FSL` | Flow Switch Low | Discrete | Closes on low flow |
| `FSHH` | Flow Switch High-High | Discrete | Emergency high flow — trips pump or opens bypass |
| `FSLL` | Flow Switch Low-Low | Discrete | Emergency low flow — trips process or triggers no-flow alarm |
| `FAH` | Flow Alarm High | — | DCS soft alarm: flow exceeds setpoint |
| `FAL` | Flow Alarm Low | — | DCS soft alarm: flow below setpoint |
| `FAHH` | Flow Alarm High-High | — | Second-level high alarm — operator action required immediately |
| `FALL` | Flow Alarm Low-Low | — | Second-level low alarm |
| `FFT` | Flow Ratio Transmitter | 4–20 mA | Ratio of two flows (FF = ratio modifier) |
| `FFIC` | Flow Ratio Indicating Controller | — | Ratio controller |
| `FDT` | Flow Differential Transmitter | 4–20 mA | Differential flow across two points |

**Real project examples:**
```
11301.FT.056A   = Flow transmitter in area 11301, loop 056, train A
11301.FIC.056A  = Flow controller for same loop
11301.FCV.056A  = Control valve on same loop
11301.FAH.056A  = High alarm on same loop
11301.FAHH.056A = High-high alarm (may trigger emergency isolation)
```

---

### Level (L\_)

| Tag | Full Name | Description |
|-----|-----------|-------------|
| `LE` | Level Element | Float, displacer, radar probe, ultrasonic |
| `LT` | Level Transmitter | 4–20 mA/HART to DCS |
| `LI` | Level Indicator | Local display |
| `LG` | Level Glass / Gauge | Local sight glass, magnetic level gauge |
| `LIC` | Level Indicating Controller | Level PID controller with display |
| `LCV` | Level Control Valve | Level control valve (often on outlet) |
| `LSH` | Level Switch High | High level discrete |
| `LSL` | Level Switch Low | Low level discrete |
| `LSHH` | Level Switch High-High | Safety: vessel overflow prevention |
| `LSLL` | Level Switch Low-Low | Safety: pump dry-run prevention |
| `LAH` | Level Alarm High | DCS alarm |
| `LAL` | Level Alarm Low | DCS alarm |
| `LAHH` | Level Alarm High-High | Critical — vessel overflow |
| `LALL` | Level Alarm Low-Low | Critical — pump cavitation |
| `LDT` | Level Differential Transmitter | Differential pressure for level measurement |

---

### Pressure (P\_)

| Tag | Full Name | Description |
|-----|-----------|-------------|
| `PE` | Pressure Element | Bourdon tube, diaphragm capsule |
| `PT` | Pressure Transmitter | 4–20 mA/HART |
| `PI` | Pressure Indicator | Local gauge or panel display |
| `PG` | Pressure Gauge | Local mechanical gauge (no electrical signal) |
| `PIC` | Pressure Indicating Controller | PID pressure controller |
| `PCV` | Pressure Control Valve | Backpressure regulator or pressure reducing valve |
| `PSH` | Pressure Switch High | High pressure discrete — typically hardwired to shutdown |
| `PSL` | Pressure Switch Low | Low pressure shutdown |
| `PSHH` | Pressure Switch High-High | Emergency shutdown (hardwired to SIS) |
| `PAH` | Pressure Alarm High | DCS process alarm |
| `PAL` | Pressure Alarm Low | DCS process alarm |
| `PAHH` | Pressure Alarm High-High | Safety alarm — may initiate SIF |
| `PALL` | Pressure Alarm Low-Low | Safety alarm |
| `PDI` | Pressure Differential Indicator | Delta-P display (filter blockage, flow) |
| `PDT` | Pressure Differential Transmitter | Delta-P transmitter |
| `PDIC` | Pressure Differential Indicating Controller | Delta-P controller |
| `PSV` | Pressure Safety Valve | Mechanical relief valve (not a DCS instrument) |
| `PRV` | Pressure Reducing Valve | Upstream set point reducing valve |

---

### Temperature (T\_)

| Tag | Full Name | Description |
|-----|-----------|-------------|
| `TE` | Temperature Element | Thermocouple (type J/K/T) or RTD (Pt100/Pt1000) |
| `TT` | Temperature Transmitter | 4–20 mA/HART head-mount or DIN-rail |
| `TI` | Temperature Indicator | Display |
| `TR` | Temperature Recorder | Strip chart or historian tag |
| `TIC` | Temperature Indicating Controller | PID temperature controller |
| `TCV` | Temperature Control Valve | Steam or cooling water valve |
| `TW` | Temperature Well | Thermowell (protective pocket for TE) |
| `TSH` | Temperature Switch High | High temperature discrete (often hardwired) |
| `TSL` | Temperature Switch Low | Low temperature discrete |
| `TSHH` | Temperature Switch High-High | Emergency shutdown |
| `TAH` | Temperature Alarm High | DCS alarm |
| `TAL` | Temperature Alarm Low | DCS alarm |
| `TAHH` | Temperature Alarm High-High | SIL-rated trip |
| `TALL` | Temperature Alarm Low-Low | |
| `TDIC` | Temperature Differential Indicating Controller | Delta-T controller (heat exchanger) |

---

### Analyzer (A\_)

| Tag | Full Name | Description |
|-----|-----------|-------------|
| `AE` | Analyzer Element | Sensor (e.g., pH electrode, O₂ probe) |
| `AT` | Analyzer Transmitter | 4–20 mA output for pH, conductivity, O₂, CO₂, chromatograph |
| `AI` | Analyzer Indicator | Display |
| `AIC` | Analyzer Indicating Controller | pH or dissolved O₂ controller |
| `AAH` | Analyzer Alarm High | High composition alarm |
| `AAL` | Analyzer Alarm Low | Low composition alarm |
| `AAHH` | Analyzer Alarm High-High | Safety — toxic or flammable threshold |
| `AALL` | Analyzer Alarm Low-Low | |

**Common analyzer tag examples:**
```
AT-301   = Oxygen analyzer transmitter (combustion air)
AIC-302  = pH indicating controller (neutralization)
AAHH-303 = LEL analyzer alarm high-high (gas detector, 20% LEL = evacuate)
```

---

### Position / Valve Feedback (Z\_)

| Tag | Full Name | Description |
|-----|-----------|-------------|
| `ZT` | Position Transmitter | Analog position 0–100% (valve positioner) |
| `ZI` | Position Indicator | Display |
| `ZSH` | Position Switch High | Open limit switch (valve fully open) |
| `ZSL` | Position Switch Low | Closed limit switch (valve fully closed) |
| `ZSHH` | Position Switch High-High | Over-travel detected |
| `ZSLL` | Position Switch Low-Low | Under-travel or closed beyond limit |
| `ZAH` | Position Alarm High | Valve open alarm (should be closed) |
| `ZAL` | Position Alarm Low | Valve closed alarm (should be open) |

---

### Speed / Frequency (S\_)

| Tag | Full Name | Description |
|-----|-----------|-------------|
| `SE` | Speed Element | Proximity probe, tachometer pickup |
| `ST` | Speed Transmitter | RPM transmitter (4–20 mA) |
| `SI` | Speed Indicator | Display |
| `SIC` | Speed Indicating Controller | Variable frequency drive setpoint controller |
| `SAH` | Speed Alarm High | Overspeed alarm |
| `SAL` | Speed Alarm Low | Underspeed alarm |
| `SAHH` | Speed Alarm High-High | Emergency overspeed trip (hardwired) |
| `SALL` | Speed Alarm Low-Low | Emergency underspeed (loss of drive) |
| `SSH` | Speed Switch High | High speed discrete |
| `SSHH` | Speed Switch High-High | Overspeed trip switch (often API 670) |
| `SSS` | Speed Switch Safety | SIL-rated safety overspeed |

---

### Vibration (V\_)

Vibration tagging follows API 670 conventions for rotating machinery protection systems, used alongside ISA-5.1.

| Tag | Full Name | Description |
|-----|-----------|-------------|
| `VE` | Vibration Element | Accelerometer, proximity probe (eddy current) |
| `VT` | Vibration Transmitter | Continuous 4–20 mA (0–25 mm/s or 0–1 in/s) |
| `VI` | Vibration Indicator | Display |
| `VAH` | Vibration Alarm High | Alert alarm (API 670 Alert level) |
| `VAHH` | Vibration Alarm High-High | Danger alarm (API 670 Danger level — trips machine) |
| `VSH` | Vibration Switch High | Discrete vibration trip switch |
| `VSHH` | Vibration Switch High-High | Emergency shutdown |

**Axial position and differential expansion:**
```
VDT-101A = Vibration Differential Transmitter — axial thrust position
VDT-101B = Redundant channel B
VDAHH-101 = Axial position alarm high-high (shutdown)
```

---

### Current / Electrical (I\_)

| Tag | Full Name | Description |
|-----|-----------|-------------|
| `II` | Current Indicator | Ammeter |
| `IT` | Current Transmitter | 4–20 mA motor current transmitter (via CT) |
| `IIC` | Current Indicating Controller | Motor load limiter |
| `IAH` | Current Alarm High | Motor overload alarm |
| `IAHH` | Current Alarm High-High | Overload trip |

---

## Equipment Tags (Non-ISA)

Equipment tags identify **physical mechanical equipment** (motors, pumps, vessels, conveyors) rather than instruments. These follow plant-specific conventions that extend ISA-5.1. The ISA-5.1 standard itself does not define these, but common industry practice has established consistent patterns.

### Standard equipment classification codes

| Code | Equipment Type | Notes |
|------|---------------|-------|
| `P` | Pump | `P-101A` = Pump 101, train A |
| `C` | Compressor | `C-201` = Compressor 201 |
| `K` | Kiln / Main rotary equipment | `K-9` = Kiln 9 (cement industry) |
| `E` | Heat Exchanger | `E-301` = Exchanger 301 |
| `H` | Fired Heater / Furnace | `H-401` = Heater 401 |
| `R` | Reactor | `R-501` = Reactor 501 |
| `T` | Tower / Distillation column | `T-601` = Tower 601 |
| `TK` | Tank (atmospheric) | `TK-701` = Tank 701 |
| `V` | Vessel (pressure) | `V-801` = Pressure vessel 801 |
| `AG` | Agitator | `AG-101` |
| `B` | Blower / Fan | `B-201` = Blower 201 |
| `CL` | Conveyor / Belt | `CL-1A` = Conveyor 1A |
| `EL` | Elevator / Bucket elevator | `EL-AA` = Elevator AA |
| `F` | Feeder / Screw conveyor | `F-4` = Feeder 4 |
| `KC` | Crusher / Grinder | `KC-F-4` = Crusher-Feeder 4 |
| `N` | Pump (European notation, from "pompe") | `N-3A` |
| `SR` | Screen / Separator | `SR-9A` = Screen 9A |
| `WG` | Weighing belt / Weigh feeder | `WG-8` |
| `Z` | Silo / Storage bin | `Z-2A` = Silo 2A |

### Equipment tag structure

```
{Area}.{EquipType}.{UnitNumber}[Suffix]

Examples:
  11301.K.9         = Area 11301, Kiln 9
  11301.P.3A        = Area 11301, Pump 3A
  11301.N.3A        = Area 11301, Pump 3A (European notation)
  11301.F.4         = Area 11301, Feeder 4
  11301.WG.8        = Area 11301, Weighing belt 8
  11301.SR.9A       = Area 11301, Screen/separator 9A
  11301.CL.1A       = Area 11301, Conveyor 1A
  11301.CLWW.1A1    = Area 11301, Conveyor with weighing 1A1
  11301.KC.F.4      = Area 11301, Crusher-Feeder 4
  11301.EL.AA       = Area 11301, Elevator AA
  11301.Z.2A        = Area 11301, Silo 2A
```

### Instrument sub-tags on equipment

When instruments are associated with a specific piece of equipment, the equipment tag appears as a suffix or prefix:

```
P-101A          = Pump equipment tag
P101A-MOTOR     = Motor module in DCS for pump P101A
P101A-C         = Run command for pump P101A
P101A-S         = Run status for pump P101A
P101A-FAULT     = Fault status
P101A-SAH       = Speed alarm high on pump P101A
P101A-IAHH      = Current alarm high-high (overload trip)
P101A-VAHH      = Vibration alarm high-high
```

---

## Redundancy Suffixes

### A/B/C suffix convention

The ISA-5.1 optional suffix identifies **multiple instruments within the same loop**. In redundant systems, the same physical measurement is made by two or three independent transmitters that vote.

| Architecture | Suffix Pattern | Example | Notes |
|-------------|---------------|---------|-------|
| Single transmitter | (no suffix) | `PT-101` | Standard, non-redundant |
| Dual redundancy (2oo2, 1oo2) | `A`, `B` | `PT-101A`, `PT-101B` | Two transmitters, same loop |
| Triple modular redundancy (2oo3) | `A`, `B`, `C` | `PT-101A`, `PT-101B`, `PT-101C` | Three transmitters, median select |
| Train identification | `A`, `B` | `FT-056A`, `FT-056B` | Two parallel production trains |
| 4-element voting | `A`, `B`, `C`, `D` | `TSHH-201A` through `D` | Quad-redundant safety |

**TMR (Triple Modular Redundancy) example — reactor pressure:**

```
PT-101A = Pressure transmitter, channel A
PT-101B = Pressure transmitter, channel B
PT-101C = Pressure transmitter, channel C

In the DCS/SIS:
  PT_101A, PT_101B, PT_101C → median selector → PIC-101 setpoint
  PAHH-101A, PAHH-101B, PAHH-101C → 2oo3 vote → SIF trip
```

**DeltaV tag naming with redundancy:**
```
AREA1/PT_101A/AI-1/PV
AREA1/PT_101B/AI-1/PV
AREA1/PT_101C/AI-1/PV
AREA1/PSHH_101/VOTER/PV   ← voting block output
```

**Honeywell Experion:**
```
PT101A.PV
PT101B.PV
PT101C.PV
```

### Suffix for parallel equipment trains

When a process line is duplicated (e.g., two parallel feed pumps), the suffix distinguishes trains:
```
P-101A, P-101B     = Pump A and Pump B (parallel duty/standby)
FT-056A, FT-056B   = Flow transmitters on parallel trains A and B
FIC-056A, FIC-056B = Controllers for each train
```

---

## Loop Numbering Conventions

### Serial numbering

All loops receive unique sequential numbers across the entire plant:
```
FIC-101, LR-102, PIC-103, TI-104 ...
```
Advantages: no duplication possible; simple database. Used in small/medium plants.

### Parallel numbering

Separate number sequences per variable type, same number used for different variables in the same equipment group:
```
TIC-101 (reactor temperature controller)
PIC-101 (reactor pressure controller)
LIC-101 (reactor level controller)
```
Advantages: all instruments numbered `-101` belong to equipment 101; easy P&ID reading. Common in large plants.

### P&ID-based numbering

Loop numbers tied to P&ID sheet ranges:
```
P&ID sheet 25 → loops 2500–2599
P&ID sheet 26 → loops 2600–2699
```

### Area-coded numbering

Area identifier embedded in the loop number:
```
11301 = Area 11301 (raw mill area of cement plant)
11301.FIC.056A = area 11301, flow controller, loop 56, train A
```

---

## P&ID Symbol Conventions

### Instrument bubble shapes

| Shape | Meaning |
|-------|---------|
| Circle | Discrete (standalone) field instrument |
| Circle in Square | Shared display / shared control (DCS) |
| Hexagon | Computer function (Advanced Process Control) |
| Diamond in Square | PLC function |
| Square | Programmable logic controller |

### Lines through bubble (instrument location)

| Line | Location |
|------|----------|
| No line | Field-mounted (local) |
| Single solid line | Primary control location (control room, operator accessible) |
| Dashed line | Behind panel (secondary / not accessible) |
| Double solid line | Local panel (accessible but remote from main board) |
| Double dashed line | Local panel, inaccessible |

### Signal line types

| Line Style | Signal Type |
|-----------|-------------|
| Solid thin line | Instrument process connection |
| Dashed line | Electrical signal (analog 4–20 mA or discrete 24VDC) |
| Solid + `//` (slash) | Pneumatic instrument air (3–15 psi) |
| Solid + `L` marks | Hydraulic signal |
| Solid + circles | Data link (Fieldbus, Ethernet, digital) |
| Dashed + wave | Wireless (WirelessHART, ISA100) |

### Fail-safe designations on control valves

| Code | Behavior | Arrow direction |
|------|----------|----------------|
| `FC` | Fail-Closed (de-energize = close) | Arrow away from valve body |
| `FO` | Fail-Open (de-energize = open) | Arrow toward valve body |
| `FL` | Fail-Last (de-energize = hold position) | No arrow |

---

## Corner Cases

### Double-function instruments

A single field device sometimes performs two functions. ISA-5.1 allows a compound tag:

```
FT-101 also performs alarming → tag is FT-101 (not FAT-101)
         The alarm is shown as FAH-101 (separate functional ID, same loop)

PDT-201 measuring level:
         Tag = LT-201 (functional: level transmitter, even though it uses DP)
         NOT PDT-201 — ISA-5.1 section 4.5: "tag based on function, not construction"
```

A flow element that is also the primary element for a flow transmitter and a flow switch on the same pipe:
```
FE-301  = orifice plate (element only)
FT-301  = flow transmitter connected to FE-301
FSH-301 = flow switch high connected to same orifice taps (same loop = 301)
```

### Custom project-specific suffixes

Some projects define non-standard suffixes on the Legend Sheet:
```
Suffix X = spare instrument (not yet in service)
Suffix T = test connection
Suffix R = remote setpoint source

Examples:
  FT-101X  = Spare flow transmitter (installed but not connected)
  PT-202T  = Pressure test point (manual reading only)
```

### DCS platform tag limitations

| Platform | Tag Format | Hyphen Rule | Case Rule | Length Limit |
|----------|-----------|-------------|-----------|-------------|
| DeltaV | `FIC_056A` | Hyphen → underscore | Uppercase preferred | 32 chars |
| Honeywell Experion | `FIC056A` (no separator) | No hyphen, no separator | Mixed case OK | 16 chars |
| Siemens PCS 7 | `FIC_056A` | Underscore | Uppercase | 24 chars |
| ABB 800xA | `FIC-056A` or `FIC_056A` | Either | Configurable | 32 chars |
| Yokogawa Centum VP | `FIC-056A` (hyphen OK) | Hyphen OK | Uppercase | 16 chars |

**Honeywell 16-character limit** is the most restrictive. Long ISA tags like `PDAHH-11301` (11 chars) fit, but `11301.CLWW.1A1` (14 chars with dots) does not if full area is included. Solution: abbreviate area or use area field separately.

### Tags with S=Safety first-letter modifier

When `S` appears as a first-letter modifier (not the measured variable), it indicates a **safety-specific** function:
```
SPAHH-101 = Safety Pressure Alarm High-High (SIL-rated, not just process alarm)
SLAHH-201 = Safety Level Alarm High-High
```

This convention distinguishes safety instrumented function (SIF) instruments from standard process alarms in the same loop. Not all projects use this — many rely on `PSHH` (Pressure Switch High-High) for the same purpose.

### Instruments in ISA-88 equipment modules

In an ISA-88 batch system, the same physical instrument may appear at multiple levels of the hierarchy:
```
Physical instrument:  LT-201          (ISA-5.1 tag)
DCS Control Module:   LT_201          (module in DCS)
Equipment Module:     REACTOR_R101    (equipment module using LT_201)
Recipe parameter:     FILL_SP         (recipe setpoint compared to LT_201.PV)
```

The ISA-5.1 tag is the lowest-level identifier; the DCS control module tag mirrors it (with `_` instead of `-`).

---

## Regional and Vendor Notation

### ISA pure (North American)

```
FIC-101A
└── No area prefix in tag (area recorded on P&ID title block)
└── Hyphen separator
└── Maximum ISA-5.1 compliance and portability
```

### European (area-embedded)

```
11301.FIC.056A
└── Area code: 11301 (raw mill area)
└── Dots as separators
└── Loop number: 056
└── Suffix: A
```

### KKS (German Power Plant — Kraftwerk-Kennzeichensystem)

```
+FIC101A=
│          └── Function type (= for measurement loop)
└────────────── Plant designation prefix (+)
```

### DeltaV path notation

```
AREA_11301/FIC_056A/PID-1/PV
└── Area / Module / FunctionBlock / Parameter
```

### Honeywell Experion notation

```
FIC056A.PV
└── TagName.Parameter (no hyphen in tag, dot for parameter)
```

---

## Tag Naming Validation in Python

```python
import re
from dataclasses import dataclass, field
from typing import Optional


# Valid first letters per ISA-5.1 (all 26 letters are technically valid;
# this set enforces the most common process industry assignments)
DEFINED_FIRST_LETTERS: set[str] = set('ABCDEFGHIJKLMNOPQRSTUVWXYZ')

# Common functional IDs (not exhaustive — extend per project Legend Sheet)
STANDARD_FUNCTIONAL_IDS: set[str] = {
    # Flow
    'FE', 'FT', 'FI', 'FIC', 'FCV', 'FR', 'FQ', 'FQI', 'FQT',
    'FSH', 'FSL', 'FSHH', 'FSLL', 'FAH', 'FAL', 'FAHH', 'FALL',
    'FFT', 'FFIC', 'FDT', 'FDIC', 'FO',
    # Level
    'LE', 'LT', 'LI', 'LG', 'LIC', 'LCV',
    'LSH', 'LSL', 'LSHH', 'LSLL', 'LAH', 'LAL', 'LAHH', 'LALL', 'LDT',
    # Pressure
    'PE', 'PT', 'PI', 'PG', 'PIC', 'PCV', 'PSV', 'PRV',
    'PSH', 'PSL', 'PSHH', 'PAH', 'PAL', 'PAHH', 'PALL',
    'PDI', 'PDT', 'PDIC',
    # Temperature
    'TE', 'TT', 'TI', 'TR', 'TIC', 'TCV', 'TW',
    'TSH', 'TSL', 'TSHH', 'TAH', 'TAL', 'TAHH', 'TALL', 'TDIC',
    # Position
    'ZT', 'ZI', 'ZSH', 'ZSL', 'ZSHH', 'ZSLL', 'ZAH', 'ZAL',
    # Speed
    'SE', 'ST', 'SI', 'SIC', 'SAH', 'SAL', 'SAHH', 'SALL',
    'SSH', 'SSHH', 'SSS',
    # Current
    'II', 'IT', 'IIC', 'IAH', 'IAHH',
    # Vibration
    'VE', 'VT', 'VI', 'VAH', 'VAHH', 'VSH', 'VSHH', 'VDT',
    # Analysis
    'AE', 'AT', 'AI', 'AIC', 'AAH', 'AAL', 'AAHH', 'AALL',
    # Hand / Manual
    'HIC', 'HV',
    # Equipment (non-ISA, project-specific)
    'K', 'N', 'F', 'WG', 'SR', 'CL', 'CLWW', 'KC', 'EL', 'Z', 'B',
    'P', 'C', 'E', 'H', 'R', 'T', 'TK', 'V', 'AG',
}


@dataclass
class ParsedTag:
    """Result of parsing an ISA-5.1 tag."""
    original: str
    area: Optional[str]
    functional_id: str
    loop_number: str
    suffix: Optional[str]
    is_valid: bool
    issues: list[str] = field(default_factory=list)


def parse_isa_tag(tag: str) -> ParsedTag:
    """
    Parse an ISA-5.1 tag in European or pure ISA format.

    Supported formats:
      European:  11301.FIC.056A  -> area=11301, func=FIC, loop=056, suffix=A
      ISA pure:  FIC-101A        -> area=None, func=FIC, loop=101, suffix=A
      No suffix: FT-101          -> area=None, func=FT, loop=101, suffix=None
    """
    issues: list[str] = []

    # European format: {area}.{funcid}.{loop}[suffix]
    european_re = re.compile(r'^(\d{3,6})\.([A-Z]{1,6})\.(\d{1,6})([A-Z0-9]*)$')
    # ISA pure format: {funcid}-{loop}[suffix]
    isa_pure_re = re.compile(r'^([A-Z]{1,6})-(\d{1,6})([A-Z0-9]?)$')

    euro_m = european_re.match(tag)
    pure_m = isa_pure_re.match(tag)

    if euro_m:
        area, func_id, loop, suffix = euro_m.groups()
    elif pure_m:
        func_id, loop, suffix = pure_m.groups()
        area = None
    else:
        return ParsedTag(
            original=tag, area=None, functional_id='', loop_number='',
            suffix=None, is_valid=False,
            issues=[f"Cannot parse format: '{tag}'. Expected AREA.FUNC.LOOP or FUNC-LOOP."]
        )

    # Validate functional ID
    if func_id not in STANDARD_FUNCTIONAL_IDS:
        issues.append(
            f"Functional ID '{func_id}' not in standard set. "
            "Add to STANDARD_FUNCTIONAL_IDS if project-defined."
        )

    # Validate first letter exists
    if func_id and func_id[0] not in DEFINED_FIRST_LETTERS:
        issues.append(f"Invalid first letter: '{func_id[0]}'")

    # Validate loop number is numeric
    if not loop.isdigit():
        issues.append(f"Loop number must be numeric: '{loop}'")

    return ParsedTag(
        original=tag,
        area=area,
        functional_id=func_id,
        loop_number=loop,
        suffix=suffix if suffix else None,
        is_valid=len(issues) == 0,
        issues=issues,
    )


def generate_loop_tags(
    area: str,
    loop_number: str,
    variable: str,
    suffix: str = 'A',
) -> dict[str, str]:
    """
    Generate the standard set of tags for a complete instrument loop.

    Args:
        area: Area code (e.g., '11301')
        loop_number: Loop number (e.g., '056')
        variable: First letter (e.g., 'F' for flow, 'T' for temperature)
        suffix: Redundancy/train suffix (e.g., 'A')

    Returns:
        Dict mapping tag role -> full tag string
    """
    s = suffix
    n = loop_number
    a = area
    v = variable

    field_tag = f"{a}.{v}T.{n}{s}"      # Transmitter
    ctrl_tag  = f"{a}.{v}IC.{n}{s}"     # Indicating controller
    valve_tag = f"{a}.{v}CV.{n}{s}"     # Control valve
    alarm_h   = f"{a}.{v}AH.{n}{s}"     # Alarm high
    alarm_l   = f"{a}.{v}AL.{n}{s}"     # Alarm low
    alarm_hh  = f"{a}.{v}AHH.{n}{s}"    # Alarm high-high
    alarm_ll  = f"{a}.{v}ALL.{n}{s}"    # Alarm low-low

    return {
        'transmitter': field_tag,
        'controller':  ctrl_tag,
        'valve':       valve_tag,
        'alarm_h':     alarm_h,
        'alarm_l':     alarm_l,
        'alarm_hh':    alarm_hh,
        'alarm_ll':    alarm_ll,
    }


def generate_redundant_tags(
    area: str,
    func_id: str,
    loop_number: str,
    channels: int = 3,
) -> list[str]:
    """
    Generate redundant instrument tags (A, B, C for TMR; A, B for dual).

    Example:
      generate_redundant_tags('11301', 'PT', '101', 3)
      -> ['11301.PT.101A', '11301.PT.101B', '11301.PT.101C']
    """
    suffixes = ['A', 'B', 'C', 'D'][:channels]
    return [f"{area}.{func_id}.{loop_number}{s}" for s in suffixes]


# Example usage
if __name__ == '__main__':
    test_tags = [
        '11301.FIC.056A',
        '11301.PDT.076A',
        '11301.SALL.020A',
        '11301.CLWW.1A1',
        'FIC-101A',
        'TIC-201',
        'INVALID_TAG',
        'PT-101A',
        'PT-101B',
        'PT-101C',
    ]

    print("ISA-5.1 Tag Validation")
    print("-" * 60)
    for tag in test_tags:
        result = parse_isa_tag(tag)
        status = "OK    " if result.is_valid else "ISSUES"
        print(f"{status} | {tag:25} | Area: {result.area or 'n/a':6} | "
              f"Func: {result.functional_id:6} | Loop: {result.loop_number}")
        for issue in result.issues:
            print(f"       >>> {issue}")

    # TMR pressure transmitter set
    print("\nTMR pressure transmitters for loop 101:")
    for t in generate_redundant_tags('UNIT1', 'PT', '101', 3):
        print(f"  {t}")
```
