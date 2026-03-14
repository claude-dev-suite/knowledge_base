# DCS Platforms - Emerson DeltaV and Honeywell Experion

> Sources:
> - https://control.com/forums/threads/deltav-device-control-function-block-operation.19339/
> - https://control.com/forums/threads/deltav-device-control.41000/
> - https://remason.com/news/automation-systems/why-does-deltav-call-control-modes-the-wrong-names-2016/
> - https://techblog.appliedprojectsengineering.com/post/2021/05/17/recipes-in-deltav
> - https://www.plcacademy.com/isa-88-s88-batch-control-explained/
> - https://sgsystemsglobal.com/glossary/isa-88-phases-equipment-modules/
> - https://emersonexchange365.com/products/control-safety-systems/f/deltav-discussions-questions/6561/sfc-control-module-fhx-export-import
> - https://filext.com/file-extension/FHX
> - https://docplayer.net/269851-Deltav-bulk-edit-deltav-bulk-edit-deltav-whitepaper-january-2013-page-1.html
> - https://emersonexchange365.com/products/control-safety-systems/f/deltav-discussions-questions/9616/fmt-file-for-motor-module
> - https://control.com/forums/threads/honeywell-cl-programming.19787/
> - https://manualzz.com/doc/o/kogvd/honeywell-custom-algorithm-block-and-custom-data-block-us...
> - https://www.manualslib.com/manual/1613510/Honeywell-Experion-Pks.html

## Overview

Emerson DeltaV and Honeywell Experion PKS are the two most widely deployed DCS platforms in the process industries. DeltaV (Emerson) dominates pharma, biotech, food & beverage, and specialty chemicals — markets that value its native ISA-88 batch support and class-based template architecture. Honeywell Experion PKS dominates large oil & gas, LNG, and petrochemical plants. Understanding both platforms' data models, programming paradigms, file formats, and bulk engineering workflows is essential for programmatic DCS configuration generation.

---

## Table of Contents

1. [Emerson DeltaV Architecture](#emerson-deltav-architecture)
2. [DeltaV ISA-88 Hierarchy](#deltav-isa-88-hierarchy)
3. [DeltaV Module Structure and Naming](#deltav-module-structure-and-naming)
4. [Device Control (DC) Block — Motor/Valve Control](#device-control-dc-block)
5. [Class-Based Configuration (Templates)](#class-based-configuration)
6. [FHX File Format](#fhx-file-format)
7. [FHX Generation in Python](#fhx-generation-in-python)
8. [DeltaV Bulk Edit (Excel/FMT Import)](#deltav-bulk-edit)
9. [Honeywell Experion Architecture](#honeywell-experion-architecture)
10. [Honeywell Control Language (CL)](#honeywell-control-language-cl)
11. [Honeywell Motor Control: MOTOR Function Block](#honeywell-motor-function-block)
12. [Honeywell Bulk Engineering: CSV / Quick Builder Import](#honeywell-bulk-engineering)
13. [DeltaV Corner Cases](#deltav-corner-cases)
14. [Honeywell Corner Cases](#honeywell-corner-cases)
15. [Platform Comparison Table](#platform-comparison-table)

---

## Emerson DeltaV Architecture

```
DeltaV System Network (DeltaV LAN — 100 Mbit/s Ethernet)
  │
  ├── DeltaV Professional Plus Workstation (Engineering)
  │     ├── DeltaV Explorer         (project tree, area/module navigation)
  │     ├── Control Studio          (FBD/SFC/ST programming editor)
  │     ├── DeltaV Bulk Edit        (Excel-based mass parameter editing)
  │     ├── DeltaV Operate          (classic VB-based HMI runtime)
  │     └── DeltaV Live             (web-based HMI, HTML5/SVG)
  │
  ├── DeltaV Controller
  │     ├── MD/MD+/MQ Series       (traditional DIN-rail I/O)
  │     ├── S-Series               (wireless, CHARM-compatible)
  │     ├── CIOC                   (I/O on demand, CHARM)
  │     └── Virtual (DeltaV Simulate)
  │
  │     Controller contains:
  │     ├── Control Modules (CM)   (individual loops, motors, valves)
  │     ├── Equipment Modules (EM) (ISA-88 equipment sequences)
  │     └── Phase Modules          (ISA-88 batch phase logic)
  │
  ├── I/O Subsystem
  │     ├── M-Series cards         (traditional fixed-type I/O)
  │     ├── CHARM modules          (intelligent field I/O, self-characterizing)
  │     ├── FOUNDATION Fieldbus    (H1 segments)
  │     └── HART integration       (HART pass-through)
  │
  └── Module Library
        ├── Module Templates       (reusable class designs)
        ├── Named Sets             (engineering unit descriptors)
        └── Library Modules        (standard function blocks)
```

**Key DeltaV architectural concepts:**

| Concept | Description |
|---------|-------------|
| **Control Module (CM)** | Basic unit of control — one CM per instrument loop, motor, or valve. Contains function blocks and their connections. Executed on a controller at a configurable scan rate. |
| **Equipment Module (EM)** | Coordinates multiple CMs to execute a procedural equipment operation. Runs SFC (Sequential Function Chart) logic. Corresponds to ISA-88 Equipment Module. |
| **Phase Module** | Implements one ISA-88 batch phase. Has predefined state machine (IDLE → RUNNING → COMPLETE/ABORTING etc.). Called by batch recipes via the Batch Executive. |
| **Module Template** | A CM or EM design that can be instantiated many times. Changes to a template propagate to all derived instances. The most powerful bulk engineering feature in DeltaV. |
| **Named Sets** | Lookup tables mapping numeric states to string descriptions (e.g., 1→"STOPPED", 2→"RUNNING"). Used for discrete alarms and status displays. |

---

## DeltaV ISA-88 Hierarchy

DeltaV implements ISA-88 (S88) batch control natively. The mapping between ISA-88 physical model levels and DeltaV objects is:

| ISA-88 Level | DeltaV Object | Description |
|-------------|--------------|-------------|
| Enterprise | (ERP/MES integration) | SAP/MES connectivity |
| Site | DeltaV Plant | Top-level project |
| Area | DeltaV Area | Process area (e.g., AREA_11301) |
| Process Cell | DeltaV Area (Batch Executive scope) | All units for one batch line |
| **Unit** | **Equipment Module (top-level)** | One reactor, blender, or vessel |
| **Equipment Module** | **Sub-Equipment Module** | Pump train, heating circuit, filter skid |
| **Control Module** | **Control Module** | Individual valve, motor, transmitter |
| Phase | Phase Module | ISA-88 batch phase logic (SFC) |
| Operation | Batch recipe operation | Sequence of phases |
| Procedure | Batch recipe procedure | Sequence of operations |

**Practical example — pharmaceutical reactor:**
```
Area: REACTORS
  Equipment Module: REACTOR_R101        (ISA-88 Unit)
    ├── Sub-EM: AGITATOR_R101           (equipment module)
    │     └── Control Module: AG_R101_MOTOR
    ├── Sub-EM: JACKET_COOLING_R101     (equipment module)
    │     ├── Control Module: TIC_R101   (jacket temp controller)
    │     └── Control Module: FCV_R101   (cooling water valve)
    ├── Control Module: LT_R101          (reactor level)
    ├── Control Module: PT_R101          (reactor pressure)
    └── Phase Module: CHARGE_SOLVENT     (batch phase)
          Phase Module: HEAT_TO_SETPOINT
          Phase Module: REACT_AT_TEMP
          Phase Module: COOL_AND_DISCHARGE
```

**Key ISA-88 principle enforced by DeltaV:** Phases interact with equipment only through Equipment Module interfaces (aliases), never by directly addressing Control Module parameters. The phase requests a capability ("heat to 80°C") from the Equipment Module; the EM coordinates CMs to achieve it.

---

## DeltaV Module Structure and Naming

### Hierarchical path addressing

```
{Area}/{Module}/{FunctionBlock}/{Parameter}

Examples:
  REACTORS/TIC_R101/PID-1/PV          → PID process value (reactor temperature)
  REACTORS/TIC_R101/PID-1/SP          → PID setpoint
  AREA_11301/P_101A_MOTOR/DC-1/PV_D   → Motor status (1=STOPPED, 2=RUNNING, 3=FAULT)
  AREA_11301/P_101A_MOTOR/DC-1/SP_D   → Motor command (1=STOP, 2=START)
  AREA_11301/FIC_056A/PID-1/OUT       → PID output (0–100%)
```

### OPC UA path

```
ns=2;s=REACTORS/TIC_R101/PID-1/PV
```

### Tag naming conventions in DeltaV

| Convention | ISA-5.1 Tag | DeltaV Module Tag | Rule |
|-----------|------------|------------------|------|
| Instrument loop | `FIC-056A` | `FIC_056A` | Hyphen → underscore |
| Equipment | `P-101A` (motor) | `P_101A_MOTOR` | Add `_MOTOR` suffix |
| Equipment module | `REACTOR R101` | `REACTOR_R101` | Spaces → underscores |
| Redundant instrument | `PT-101A`, `PT-101B` | `PT_101A`, `PT_101B` | Direct mapping |

---

## Device Control (DC) Block

The `DC` (Device Control) function block is the standard block in DeltaV for controlling discrete devices: motors, pumps, fans, conveyors, and two-state valves (open/close). It implements a multi-state device controller with mode management, feedback monitoring, interlock logic, and alarm generation.

### DC Block control modes

DeltaV's terminology for control modes differs from ISA-5.1 intuition. Understanding this is critical for correct configuration:

| DeltaV Mode | Code | What it means |
|-------------|------|--------------|
| **MAN (Manual)** | 8 | The operator in the DCS gives commands directly. DeltaV does NOT change the output based on any logic — only operator interaction changes the setpoint. |
| **AUTO (Automatic)** | 4 | DeltaV logic (PID, sequence, or cascade) automatically manages the setpoint. For discrete devices, AUTO means the device responds to the block's internal auto setpoint (set by sequence or interlock logic). |
| **CAS (Cascade)** | 32 | The setpoint comes from another block's output (e.g., a batch phase, an equipment module, or a higher-level controller). Functionally equivalent to AUTO but the source is another block. |
| **OOS (Out of Service)** | 16 | Device removed from service. All outputs forced to fail-safe. |
| **LO (Local Override)** | 128 | Field Local panel is active. DCS output overridden. Status only — read from `LOCAL_OVERRIDE` input. |

**Key distinction:** When a technician says "put the pump in AUTO," they mean the DCS is automatically managing it — this is DeltaV's `AUTO` or `CAS` mode. When they say "put it in Manual," they mean an operator will command it manually — this is DeltaV's `MAN` mode.

### DC Block signal interface (complete parameter list)

```
Configuration Parameters:
  FBK_STRATEGY  : INT    = Feedback strategy (see table below)
  FBK_TIMEOUT   : REAL   = Monitor time (seconds) for run feedback (default 10s)
  OPENING_TIME  : REAL   = Time allowed for valve to open fully (default 30s)
  CLOSING_TIME  : REAL   = Time allowed for valve to close fully (default 30s)
  IO_OPTS       : WORD   = I/O option flags (inversion, filtering, etc.)
  SIMULATE      : BOOL   = Simulation mode (bypasses physical I/O)

Mode Inputs:
  MODE_BLK.TARGET   : UINT  = Requested mode (MAN=8, AUTO=4, CAS=32, OOS=16)
  MODE_BLK.PERMITTED: UINT  = Allowed modes bitmask

Input Parameters:
  IN_D1        : BOOL   = Digital input 1: Run/Open feedback
  IN_D2        : BOOL   = Digital input 2: Stopped/Closed feedback
  IN_D3        : BOOL   = Digital input 3: Fault/Trip input
  IN_D4        : BOOL   = Digital input 4: Available-for-Auto (local/remote)
  INTERLOCK_D  : BOOL   = Interlock permissive (FALSE = block device — keep stopped)
  PERMISSIVE_D : BOOL   = Run permissive (TRUE = OK to start)
  CAS_IN_D     : UINT   = Cascade setpoint from upstream block (1=STOP 2=START)
  SP_D         : UINT   = Manual/Auto setpoint (1=STOP, 2=START, 3=OPEN, 4=CLOSE)
  F_IN_D1..D4  : BOOL   = Alternative: external wiring instead of DI block outputs
  F_OUT_D1..D4 : BOOL   = Alternative: external DO outputs instead of block

Output Parameters:
  OUT_D1       : BOOL   = Output 1: Start command (to DO block → field)
  OUT_D2       : BOOL   = Output 2: Stop command (if separate contactor)
  OUT_D3       : BOOL   = Output 3: Open command (for valves)
  OUT_D4       : BOOL   = Output 4: Close command (for valves)
  PV_D         : UINT   = Process value (device state — see table)
  MODE_BLK.ACTUAL : UINT = Currently active mode

Alarm/Status Outputs:
  DISC_ALM     : UINT   = Discrete alarm code
  DISC_LIM     : UINT   = Alarm limit configuration
  BKCAL_OUT    : REAL   = Backpressure calculation (for cascade coordination)
```

### FBK_STRATEGY values

| Value | Strategy Name | Inputs Used | Use Case |
|-------|-------------|-------------|----------|
| 1 | Run feedback only | `IN_D1` (run) | Simple motors with one feedback |
| 2 | Run + Stopped | `IN_D1` (run) + `IN_D2` (stopped) | Motors with full feedback (run + stop LS) |
| 3 | Run + Stopped + Available | `IN_D1` + `IN_D2` + `IN_D4` | Motors with local/remote switch |
| 4 | Run + Fault | `IN_D1` (run) + `IN_D3` (fault) | VFD drives with fault relay |
| 5 | Open + Closed | `IN_D1` (open LS) + `IN_D2` (closed LS) | Two-position valves |
| 6 | Open + Closed + Fault | `IN_D1` + `IN_D2` + `IN_D3` | Valves with position fault |

### PV_D device state encoding

| PV_D Value | State Name | Condition |
|------------|-----------|-----------|
| 1 | STOPPED | Device is stopped/closed and healthy |
| 2 | RUNNING | Device is running/open (run feedback confirmed) |
| 3 | FAULT | Fault condition (timeout or trip input active) |
| 4 | LOCAL | Field local panel is active (DCS commands overridden) |
| 5 | TRANSITIONING | Moving between states (OPENING, CLOSING, STARTING, STOPPING) |
| 6 | OOS | Out of Service (maintenance mode) |

### Standard motor wiring in DeltaV (FBK_STRATEGY = 2)

```
Field → DI Block → DC Block
─────────────────────────────
MCC Run Contact (NO)   → DI-1/CHANNEL → DI-1/PV_D → DC-1/IN_D1
MCC Stop Contact (NC)  → DI-2/CHANNEL → DI-2/PV_D → DC-1/IN_D2
MCC Trip Relay (NC)    → DI-3/CHANNEL → DI-3/PV_D → DC-1/IN_D3  (optional)

DC Block → DO Block → Field
─────────────────────────────
DC-1/OUT_D1 → DO-1/SP_D → DO-1/CHANNEL → MCC Start Coil
DC-1/OUT_D2 → DO-2/SP_D → DO-2/CHANNEL → MCC Stop Coil  (if separate)
```

### Standard valve wiring in DeltaV (FBK_STRATEGY = 5)

```
Field → DI Blocks → DC Block
─────────────────────────────
Open Limit Switch (NO)   → DI-1/PV_D → DC-1/IN_D1
Closed Limit Switch (NO) → DI-2/PV_D → DC-1/IN_D2

DC Block → DO Blocks → Field
─────────────────────────────
DC-1/OUT_D3 → DO-1 → Open solenoid (energize to open)
DC-1/OUT_D4 → DO-2 → Close solenoid (energize to close)
```

### Using F_IN_D and F_OUT_D (external wiring)

For cases where I/O blocks live in a different control module or come from FOUNDATION Fieldbus devices:

```fhx
(* External wiring: connect from another module *)
PARAMETER_CONNECTION
    SOURCE = "FIELDBUS_DEVICE/FBK_RUN"   (* from external CM *)
    DEST   = "DC-1/F_IN_D1";             (* DC block's external input pin *)
```

---

## Class-Based Configuration

Class-based configuration is DeltaV's most powerful feature for large-scale engineering. It implements a template-instance inheritance model:

1. A **Module Template** (`MT_` prefix by convention) is designed and tested once
2. Multiple **Control Module instances** are derived from the template
3. Changes to the template **automatically propagate** to all derived instances (with options for selective promotion or lock-down)

### Template vs instance parameters

| Parameter Type | Can be Changed on Instance? | Example |
|---------------|---------------------------|---------|
| **Locked** | No — defined by template | Block structure, block connections, required I/O |
| **Exposed (Override)** | Yes — overridden per instance | Description, alarm limits, scaling, EU |
| **Inherited** | Yes — until instance overrides | Default setpoints |

### Template creation workflow

```
1. Design in Control Studio:
   Template Name: MT_MOTOR_1SPD   (MT_ prefix = module template)

2. Define function blocks:
   DC-1  (Device Control)
   DI-1  (Digital Input - run feedback)
   DI-2  (Digital Input - stop feedback)
   DO-1  (Digital Output - start command)
   AI-1  (optional: current transmitter)

3. Define parameter connections (wiring):
   DI-1/PV_D  →  DC-1/IN_D1   (run feedback)
   DI-2/PV_D  →  DC-1/IN_D2   (stop feedback)
   DC-1/OUT_D1 → DO-1/SP_D    (start command)

4. Set default values:
   DC-1/FBK_STRATEGY = 2      (run + stopped feedback)
   DC-1/FBK_TIMEOUT  = 10.0   (10 second timeout)
   DC-1/MODE_BLK.PERMITTED = AUTO|MAN|OOS|CAS

5. Mark per-instance parameters as overridable:
   DC-1/DISC_ALM/PRIORITY   (override per motor criticality)
   Description               (override per motor)
   I/O channel assignments   (always unique per instance)

6. Instantiate (hundreds of times):
   AREA_11301/P_101A_MOTOR  → from MT_MOTOR_1SPD
   AREA_11301/P_101B_MOTOR  → from MT_MOTOR_1SPD
   AREA_11301/P_102A_MOTOR  → from MT_MOTOR_1SPD
   ... (200 motors)

7. Template update → Propagate:
   Modify MT_MOTOR_1SPD (fix a bug, add a parameter)
   "Propagate to instances" → all 200 instances updated automatically
```

---

## FHX File Format

FHX (Function Block eXport / Function Block Hierarchy eXchange) is DeltaV's proprietary text-based configuration format. It is UTF-16 Little Endian encoded text with a Pascal-like block syntax. FHX is the primary file format for:
- Exporting/importing control module configurations
- Programmatically generating DeltaV configurations
- Importing device description templates (HART/FF device templates ship as FHX)
- Automating bulk module creation from engineering data

**Opening FHX files:** Use a text editor that supports UTF-16 encoding (Notepad++, VS Code). DeltaV Explorer can open FHX directly via File → Import.

### FHX syntax structure

```
Top-level objects:
  MODULE_TEMPLATE "name" / ...  { ... }    Module template definition
  MODULE "name" / ...  { ... }             Control module instance
  NAMED_SET "name" / ...  { ... }          Named set (state descriptor lookup table)
  FUNCTION_BLOCK_DEFINITION "name" / ...   Custom function block type definition
  PHASE_MODULE "name" / ...  { ... }       Batch phase module

Within a MODULE or MODULE_TEMPLATE block:
  FUNCTION_BLOCK "name" / DEFINITION = "type" / ... { ... }   Block declaration
  PARAMETER_CONNECTION SOURCE = "..." DEST = "...";            Wire connection
  IO_ASSIGN "block/param" = "ctrl/slot/channel";               I/O channel binding
  ATTRIBUTE parameter_name / { ... }                           Parameter with sub-fields
  ATTRIBUTE parameter_name = value;                            Simple parameter value
```

### FHX Module Template example

```fhx
(* FHX File — Emerson DeltaV Configuration
   Encoding: UTF-16 LE
   DeltaV version: v14+  *)

MODULE_TEMPLATE "MT_MOTOR_1SPD" /
  MODULE_CLASS = DEVICE /
  DESCRIPTION = "Single Speed Motor - 1 Start Output, Run+Stop Feedback" /
{
  FUNCTION_BLOCK "DC-1" /
    DEFINITION = "DC" /
    DESCRIPTION = "Device Control" /
  {
    ATTRIBUTE MODE_BLK /
    {
      TARGET    = AUTO;
      PERMITTED = AUTO | MAN | OOS | CAS;
    }
    ATTRIBUTE FBK_STRATEGY = 2;         (* Run + Stop feedback *)
    ATTRIBUTE FBK_TIMEOUT  = 10.0;      (* 10 second monitor timeout *)
    ATTRIBUTE DISC_ALM /
    {
      DISC_LIM  = 1;                    (* Alarm on state 1 = FAULT *)
      PRIORITY  = 2;                    (* High priority alarm *)
      ACKED_ST  = 3;                    (* Auto-acknowledge on return to RUNNING *)
    }
  }

  FUNCTION_BLOCK "DI-1" /
    DEFINITION = "DI" /
    DESCRIPTION = "Run Feedback" /
  {
    ATTRIBUTE IO_OPTS  = 0;
    ATTRIBUTE FS_STATE = 0;             (* Fail-safe: de-energize = 0 *)
  }

  FUNCTION_BLOCK "DI-2" /
    DEFINITION = "DI" /
    DESCRIPTION = "Stop Feedback" /
  {
    ATTRIBUTE IO_OPTS  = 0;
    ATTRIBUTE FS_STATE = 1;             (* Fail-safe: de-energize = 1 (normally-closed) *)
  }

  FUNCTION_BLOCK "DO-1" /
    DEFINITION = "DO" /
    DESCRIPTION = "Start Output" /
  {
    ATTRIBUTE IO_OPTS  = 0;
    ATTRIBUTE FS_STATE = 0;             (* Fail-safe: de-energize = stop *)
  }

  (* ── Parameter connections (wiring) ── *)
  PARAMETER_CONNECTION
    SOURCE = "DI-1/PV_D"
    DEST   = "DC-1/IN_D1";             (* Run feedback → DC input 1 *)

  PARAMETER_CONNECTION
    SOURCE = "DI-2/PV_D"
    DEST   = "DC-1/IN_D2";             (* Stop feedback → DC input 2 *)

  PARAMETER_CONNECTION
    SOURCE = "DC-1/OUT_D1"
    DEST   = "DO-1/SP_D";              (* DC start output → DO block *)
}
```

### FHX Control Module instance example

```fhx
MODULE "P_101A_MOTOR" /
  MODULE_CLASS    = DEVICE /
  DESCRIPTION     = "FEED PUMP A - 11kW" /
  AREA            = "AREA_11301" /
  PERIOD          = 500 /             (* Execution period: 500ms *)
  PHASE           = 1 /               (* Execution phase priority *)
  SUBTYPE         = "MT_MOTOR_1SPD" / (* Derived from this template *)
{
  (* Override per-instance values *)
  FUNCTION_BLOCK "DC-1" /
    DEFINITION = "DC" /
  {
    ATTRIBUTE FBK_TIMEOUT = 15.0;     (* Override: 15s for slow-start motor *)
    ATTRIBUTE DISC_ALM /
    {
      PRIORITY = 1;                   (* Override: critical pump = P1 alarm *)
    }
  }

  (* I/O channel assignments — always unique per instance *)
  IO_ASSIGN "DI-1/CHANNEL_A" = "CTLR1/SLOT03/CH01";
  IO_ASSIGN "DI-2/CHANNEL_A" = "CTLR1/SLOT03/CH02";
  IO_ASSIGN "DO-1/CHANNEL_A" = "CTLR1/SLOT04/CH01";
}
```

### FHX Named Set example (state descriptions)

```fhx
NAMED_SET "MOTOR_STATES" /
  DESCRIPTION = "Motor device states" /
{
  0 = "UNDEFINED";
  1 = "STOPPED";
  2 = "RUNNING";
  3 = "FAULT";
  4 = "LOCAL";
  5 = "TRANSITIONING";
  6 = "OOS";
}
```

### FHX Phase Module example (ISA-88 batch phase)

```fhx
PHASE_MODULE "FILL_REACTOR" /
  DESCRIPTION = "Fill reactor with solvent to setpoint level" /
  AREA        = "REACTOR_R101" /
{
  (* Phase recipe parameters — set values in recipe, not hardcoded *)
  PARAMETER "FILL_SETPOINT" /
    DESCRIPTION  = "Target fill level" /
    TYPE         = FLOAT /
    DEFAULT_VALUE = 80.0 /
    EU           = "%" /
    EU_MAX       = 100.0 /
    EU_MIN       = 0.0 /

  PARAMETER "FILL_RATE" /
    DESCRIPTION  = "Maximum fill flow rate" /
    TYPE         = FLOAT /
    DEFAULT_VALUE = 500.0 /
    EU           = "kg/h" /

  (* State transitions — SFC logic *)
  STATE_TRANSITION
    FROM      = IDLE
    TO        = RUNNING
    CONDITION = "TRUE"
    ACTION    = "FILL_VALVE/SP_D := 2";     (* Open fill valve *)

  STATE_TRANSITION
    FROM      = RUNNING
    TO        = COMPLETE
    CONDITION = "LT_R101/PV >= FILL_SETPOINT"
    ACTION    = "FILL_VALVE/SP_D := 1";     (* Close fill valve *)

  STATE_TRANSITION
    FROM      = RUNNING
    TO        = ABORTING
    CONDITION = "PAHH_R101/DISC_ALM.FAIL_ACTIVE"
    ACTION    = "FILL_VALVE/SP_D := 1";
}
```

---

## FHX Generation in Python

```python
import pandas as pd
from dataclasses import dataclass
from pathlib import Path
from typing import Optional


@dataclass
class DeltaVMotor:
    """DeltaV motor control module configuration parameters."""
    tag_name:          str           # e.g., "P_101A_MOTOR"
    description:       str           # e.g., "FEED PUMP A - 11kW"
    area:              str           # e.g., "AREA_11301"
    controller:        str           # e.g., "CTLR1"
    run_fbk_channel:   str           # e.g., "SLOT03/CH01"
    stp_fbk_channel:   str           # e.g., "SLOT03/CH02"
    start_out_channel: str           # e.g., "SLOT04/CH01"
    template_name:     str = "MT_MOTOR_1SPD"
    fbk_timeout:       float = 10.0
    fbk_strategy:      int = 2
    period_ms:         int = 500
    alarm_priority:    int = 2


_FHX_MOTOR_INSTANCE = '''\
MODULE "{tag_name}" /
  MODULE_CLASS = DEVICE /
  DESCRIPTION = "{description}" /
  AREA = "{area}" /
  PERIOD = {period_ms} /
  PHASE = 1 /
  SUBTYPE = "{template_name}" /
{{
  FUNCTION_BLOCK "DC-1" /
    DEFINITION = "DC" /
  {{
    ATTRIBUTE FBK_STRATEGY = {fbk_strategy};
    ATTRIBUTE FBK_TIMEOUT  = {fbk_timeout};
    ATTRIBUTE DISC_ALM /
    {{
      PRIORITY = {alarm_priority};
    }}
  }}

  IO_ASSIGN "DI-1/CHANNEL_A" = "{controller}/{run_ch}";
  IO_ASSIGN "DI-2/CHANNEL_A" = "{controller}/{stp_ch}";
  IO_ASSIGN "DO-1/CHANNEL_A" = "{controller}/{out_ch}";
}}

'''


_FHX_PID_INSTANCE = '''\
MODULE "{tag_name}" /
  MODULE_CLASS = REGULATORY /
  DESCRIPTION = "{description}" /
  AREA = "{area}" /
  PERIOD = {period_ms} /
  PHASE = 1 /
{{
  FUNCTION_BLOCK "AI-1" /
    DEFINITION = "AI" /
    DESCRIPTION = "Process Variable Input" /
  {{
    ATTRIBUTE EU_100 = {eu_high};
    ATTRIBUTE EU_0   = {eu_low};
    ATTRIBUTE L_TYPE = 2;              (* 4-20mA linear *)
  }}

  FUNCTION_BLOCK "PID-1" /
    DEFINITION = "PID" /
    DESCRIPTION = "PID Controller" /
  {{
    ATTRIBUTE MODE_BLK /
    {{
      TARGET    = AUTO;
      PERMITTED = AUTO | MAN | OOS | CAS;
    }}
    ATTRIBUTE PV_SCALE /
    {{
      EU_100 = {eu_high};
      EU_0   = {eu_low};
      UNITS_INDEX = 1;
    }}
    ATTRIBUTE OUT_SCALE /
    {{
      EU_100 = 100.0;
      EU_0   = 0.0;
      UNITS_INDEX = 57;    (* % *)
    }}
    ATTRIBUTE GAIN  = {gain};
    ATTRIBUTE RESET = {reset};
    ATTRIBUTE RATE  = {rate};
    ATTRIBUTE HI_LIM = {out_high};
    ATTRIBUTE LO_LIM = {out_low};
  }}

  FUNCTION_BLOCK "AO-1" /
    DEFINITION = "AO" /
    DESCRIPTION = "Output to Control Valve" /
  {{
    ATTRIBUTE EU_100 = 100.0;
    ATTRIBUTE EU_0   = 0.0;
  }}

  PARAMETER_CONNECTION
    SOURCE = "AI-1/PV"
    DEST   = "PID-1/PV";

  PARAMETER_CONNECTION
    SOURCE = "PID-1/OUT"
    DEST   = "AO-1/SP";

  IO_ASSIGN "AI-1/CHANNEL_A"  = "{controller}/{ai_ch}";
  IO_ASSIGN "AO-1/CHANNEL_A"  = "{controller}/{ao_ch}";
}}

'''


def generate_motor_fhx(motor: DeltaVMotor) -> str:
    """Generate FHX text for a single motor module."""
    return _FHX_MOTOR_INSTANCE.format(
        tag_name=motor.tag_name,
        description=motor.description,
        area=motor.area,
        period_ms=motor.period_ms,
        template_name=motor.template_name,
        fbk_strategy=motor.fbk_strategy,
        fbk_timeout=motor.fbk_timeout,
        alarm_priority=motor.alarm_priority,
        controller=motor.controller,
        run_ch=motor.run_fbk_channel,
        stp_ch=motor.stp_fbk_channel,
        out_ch=motor.start_out_channel,
    )


def generate_fhx_from_excel(excel_path: str, output_path: str) -> int:
    """
    Generate a DeltaV FHX file from an Excel instrument list.

    Required Excel columns:
      tag_name, description, area, controller,
      run_fbk_channel, stp_fbk_channel, start_out_channel
    Optional:
      template_name, fbk_timeout, fbk_strategy, period_ms, alarm_priority

    Returns: number of modules generated
    """
    df = pd.read_excel(excel_path, dtype=str).fillna('')
    sections: list[str] = []

    for _, row in df.iterrows():
        motor = DeltaVMotor(
            tag_name=row['tag_name'].strip(),
            description=row['description'].strip(),
            area=row['area'].strip(),
            controller=row['controller'].strip(),
            run_fbk_channel=row['run_fbk_channel'].strip(),
            stp_fbk_channel=row['stp_fbk_channel'].strip(),
            start_out_channel=row['start_out_channel'].strip(),
            template_name=row.get('template_name', 'MT_MOTOR_1SPD').strip() or 'MT_MOTOR_1SPD',
            fbk_timeout=float(row['fbk_timeout']) if row.get('fbk_timeout') else 10.0,
            fbk_strategy=int(row['fbk_strategy']) if row.get('fbk_strategy') else 2,
            period_ms=int(row['period_ms']) if row.get('period_ms') else 500,
            alarm_priority=int(row['alarm_priority']) if row.get('alarm_priority') else 2,
        )
        sections.append(generate_motor_fhx(motor))

    # DeltaV FHX files should be UTF-16 LE encoded
    fhx_content = ''.join(sections)
    with open(output_path, 'w', encoding='utf-16-le') as f:
        f.write(fhx_content)

    print(f"Generated {len(sections)} modules → {output_path}")
    return len(sections)
```

---

## DeltaV Bulk Edit

DeltaV Bulk Edit allows mass editing of Control Module parameters using Excel. It uses format specification files (`.FMT`) to define which parameters to export and import. This is complementary to FHX generation: FHX creates module structure; Bulk Edit modifies per-instance values.

### Bulk Edit workflow

```
1. Select area or modules in DeltaV Explorer
2. Right-click → "Bulk Edit" → select FMT format file
   OR: File → Import/Export → User Defined Format
3. Export to TAB-delimited Unicode text (opens in Excel as TSV)
4. Modify values in Excel:
   - Descriptions
   - Alarm limits (HI_HI_LIM, HI_LIM, LO_LIM, LO_LO_LIM)
   - Engineering unit scales (EU_100, EU_0)
   - PID parameters (GAIN, RESET, RATE)
   - I/O channel assignments
   - Named set references
5. Save as TAB-delimited text (Unicode)
6. Import back into DeltaV Explorer
7. DeltaV validates, reports errors, then applies changes
8. Download modules to controller
```

### FMT format specification file

A `.FMT` file lives in `%DeltaV%\DVData\BulkEdit\` and defines the fields and their order for import/export. Fields are tab-separated in the export file.

```
(* Standard motor module FMT format *)
NAME
DESCRIPTION
DC-1/FBK_TIMEOUT
DC-1/DISC_ALM/PRIORITY
DI-1/CHANNEL_A
DI-2/CHANNEL_A
DO-1/CHANNEL_A
```

**System-supplied FMT files:**
- `Cards.fmt` — I/O card type, slot, controller
- `Channels.fmt` — Channel assignments, engineering units
- `ControlModules.fmt` — Module names, descriptions, areas
- `AlarmLimits.fmt` — HI_HI, HI, LO, LO_LO alarm limits
- `PID_Params.fmt` — GAIN, RESET, RATE for PID modules

### Bulk Edit Excel format (TAB-delimited)

Example TSV exported by DeltaV Bulk Edit for motor modules:

```
NAME	           DESCRIPTION	          DC-1/FBK_TIMEOUT	DC-1/DISC_ALM/PRIORITY	DI-1/CHANNEL_A	  DI-2/CHANNEL_A	  DO-1/CHANNEL_A
P_101A_MOTOR	   FEED PUMP A - 11kW	  10.0	            2	                    CTLR1/SLOT03/CH01	CTLR1/SLOT03/CH02	CTLR1/SLOT04/CH01
P_101B_MOTOR	   FEED PUMP B - 11kW	  10.0	            2	                    CTLR1/SLOT03/CH03	CTLR1/SLOT03/CH04	CTLR1/SLOT04/CH02
P_102A_MOTOR	   DISCHARGE PUMP A	      15.0	            1	                    CTLR1/SLOT05/CH01	CTLR1/SLOT05/CH02	CTLR1/SLOT06/CH01
```

### What Bulk Edit CAN change

- Module descriptions
- Function block parameter values (alarm limits, scales, PID tuning, timeouts)
- I/O channel assignments
- Named set references
- Module execution period

### What Bulk Edit CANNOT change

- Module structure (add or remove function blocks)
- Parameter connections (wiring between blocks)
- Module class / template assignment
- Module area / controller assignment (requires FHX re-import)

**Best practice:** Use Module Templates (FHX) for structure; use Bulk Edit for per-instance parameterization. Together they cover 95% of bulk engineering needs without writing individual modules manually.

---

## Honeywell Experion Architecture

```
Experion PKS System Network (FTE — Fault Tolerant Ethernet)
  │
  ├── Experion Station (Engineering / Operator)
  │     ├── Control Builder           (FBD programming, CL algorithm authoring)
  │     ├── HMIWeb Display Builder    (HTML5 display design)
  │     ├── Experion Operate          (operator console runtime)
  │     ├── Asset Manager             (equipment health monitoring)
  │     └── Quick Builder             (bulk point configuration)
  │
  ├── C300 Controller (primary process controller)
  │     ├── Control Strategies        (FBD-based control, CL algorithms)
  │     ├── Safety Manager integration (SIL-2 via SM controller)
  │     └── Fault-tolerant options    (redundant C300 pairs)
  │
  ├── ControlEdge (IEC 61131-3 compliant controller)
  │     ├── Full IEC 61131-3 support  (LD, FBD, ST, SFC)
  │     └── Migration path from legacy TDC-3000 / TPS systems
  │
  └── I/O Subsystem
        ├── Series C I/O              (standard process I/O)
        ├── HART I/O modules          (HART FDM passthrough)
        ├── FOUNDATION Fieldbus       (FF H1 segments)
        └── Remote I/O               (FTA-based distributed I/O)
```

**Experion key concepts:**

| Concept | Description |
|---------|-------------|
| **Point** | The basic data entity. Every tag in Honeywell is a "point" — analog (AI/AO), digital (DI/DO), accumulator, or algorithm point. |
| **Control Strategy** | A group of function blocks and their connections for one area. Analogous to DeltaV Control Module but for a collection of loops. |
| **Control Module (CM)** | In Experion, a CM is a higher-level grouping containing multiple points and the CL algorithm that coordinates them. |
| **CL (Control Language)** | Honeywell's proprietary ST-like control logic language. Used for complex coordination, interlocks, and custom algorithms. |
| **Custom Algorithm (CA) / CAB** | User-written CL algorithm encapsulated as a reusable function block type. The Honeywell equivalent of a DeltaV Module Template. |
| **SCM (Sequence Control Module)** | ISA-88 sequential control in Experion — similar to DeltaV's Phase/Equipment Module. |

---

## Honeywell Control Language (CL)

CL (Control Language) is Honeywell's proprietary programming language for the Application Module (AM) and C300 controllers. It predates IEC 61131-3 structured text but is syntactically similar. CL programs run inside Custom Algorithm Blocks (CAB) which are attached to points.

### CL syntax

```
(* ─── Variable declarations ─── *)
REAL    PV_SCALED;
REAL    SP_ACTIVE;
BOOL    ALARM_ACTIVE;
BOOL    FAULT_LATCH;
INT     STATE;

(* Constants — declared inline *)
REAL CONST SCALE_HIGH := 1000.0;
REAL CONST SCALE_LOW  := 0.0;
TIME CONST MONITOR_TIME := 10S;      (* CL time literal: no T# prefix *)

(* ─── Expressions ─── *)
PV_SCALED := (PV_RAW - 4.0) / 16.0 * (SCALE_HIGH - SCALE_LOW) + SCALE_LOW;

(* ─── IF statement ─── *)
IF PV_SCALED > HH_LIMIT THEN
    ALARM_ACTIVE := TRUE;
    FAULT_LATCH  := TRUE;
ELSIF PV_SCALED > H_LIMIT THEN
    ALARM_ACTIVE := TRUE;
    FAULT_LATCH  := FALSE;
ELSE
    ALARM_ACTIVE := FALSE;
END_IF;

(* ─── SELECT statement (Honeywell's CASE equivalent) ─── *)
SELECT STATE
  WHEN 0:
    (* IDLE *)
    START_OUT := FALSE;
    IF START_CMD AND NOT FAULT_LATCH THEN
        STATE := 1;
    END_IF;
  WHEN 1:
    (* STARTING *)
    START_OUT := TRUE;
    IF RUN_FBK THEN
        STATE := 2;
    END_IF;
    IF NOT INTLK THEN
        STATE := 99;
    END_IF;
  WHEN 2:
    (* RUNNING *)
    START_OUT := TRUE;
    IF STOP_CMD OR NOT INTLK THEN
        STATE := 3;
    END_IF;
  WHEN 99:
    (* FAULT *)
    START_OUT := FALSE;
    IF RESET_CMD AND INTLK THEN
        STATE := 0;
        FAULT_LATCH := FALSE;
    END_IF;
END_SELECT;

(* ─── Honeywell-specific system calls ─── *)
SEND_ALARM("P101A", "MOTOR FAULT", PRIORITY_HIGH);
CALL LOG_EVENT(TAG := "P101A", MSG := "Motor started", TIME := NOW());
```

### CL vs IEC 61131-3 Structured Text

| Feature | CL (Honeywell) | ST (IEC 61131-3) |
|---------|---------------|-----------------|
| Comments | `(* ... *)` | `(* ... *)` or `// ...` |
| Assignment | `:=` | `:=` |
| Conditional | `IF...THEN...ELSIF...END_IF` | `IF...THEN...ELSIF...END_IF` |
| **Case statement** | `SELECT...WHEN...END_SELECT` | `CASE...OF...END_CASE` |
| **Function call** | `CALL FUNC_NAME(params)` or `FUNC_NAME(params)` | `result := FuncName(params)` |
| Alarm output | `CALL SEND_ALARM(...)` | Platform-specific |
| Time literals | `10S`, `500MS` (no `T#`) | `T#10s`, `T#500ms` |
| **FOR loop** | `FOR i := 1 TO 10 DO ... END_FOR` | Same |
| WHILE loop | `WHILE cond DO ... END_WHILE` | Same |
| Not equal | `<>` or `!=` | `<>` |
| Standard FBs | System function calls | Instantiated FBs |
| Return value | Via parameter or global | Function name |

### CAB (Custom Algorithm Block) structure

```
CAB_TYPE "MOTOR_CONTROL_CAB"
  DESCRIPTION "Standard motor control algorithm"

  (* External parameters — visible to operator *)
  PARAMETER START_CMD : BOOL  (INPUT)
  PARAMETER STOP_CMD  : BOOL  (INPUT)
  PARAMETER INTLK     : BOOL  (INPUT)
  PARAMETER RUN_FBK   : BOOL  (INPUT)
  PARAMETER START_OUT : BOOL  (OUTPUT)
  PARAMETER STATUS    : INT   (OUTPUT)   (* 1=STOP 2=RUN 3=FAULT *)
  PARAMETER FAULT     : BOOL  (OUTPUT)

  (* Internal variables *)
  INTERNAL STATE : INT := 0
  INTERNAL MONITOR_TIMER : REAL := 0.0

  (* CL algorithm body — as shown above *)
END_CAB_TYPE
```

---

## Honeywell Motor Function Block

The built-in `MOTOR` function block in Honeywell Experion PKS implements standard motor control with mode management and feedback monitoring. It is configured as a point type.

### MOTOR block parameters

```
Point Type: DISTATE (two-state digital) — used for simple motors
            MOTOR   (dedicated motor type in some Experion releases)

Configuration Parameters:
  DESCRIPTOR  : STRING  = Engineering description
  ENGUNITS    : STRING  = Engineering units (e.g., "RPM", "ON/OFF")
  ALMOPT      : WORD    = Alarm options (enable/disable alarm types)
  ALMPRI      : INT     = Default alarm priority (1=Critical, 3=High, 4=Low)
  INDEADBAND  : REAL    = Input deadband (%)
  HHASP       : REAL    = High-High alarm setpoint
  HASP        : REAL    = High alarm setpoint
  LASP        : REAL    = Low alarm setpoint
  LLASP       : REAL    = Low-Low alarm setpoint
  FBTIME      : REAL    = Feedback timeout (seconds) — default 10.0
  FBSTRAT     : INT     = Feedback strategy (1=run only, 2=run+stop)

Runtime Inputs:
  SP      : BOOL  = Setpoint (1=Start, 0=Stop)
  MODE    : UINT  = Mode (AUTO, MAN, OOS)
  INTLK   : BOOL  = Interlock (TRUE=OK to run)
  FB_RUN  : BOOL  = Running feedback
  FB_STOP : BOOL  = Stopped feedback
  FB_TRIP : BOOL  = Trip/fault feedback
  LOCAL   : BOOL  = Local panel active

Runtime Outputs:
  OP      : BOOL  = Output to field (start contactor)
  STATUS  : UINT  = Current status (1=STOPPED, 2=RUNNING, 3=FAULT, 4=LOCAL)
  ALARM   : BOOL  = Any alarm active
  RUNFAIL : BOOL  = Run failure (timeout exceeded)
```

### MOTOR block in CL context

```
(* Accessing MOTOR block status in CL logic *)
IF MOTOR_P101A.STATUS = 2 THEN
    (* Motor is running — update runtime counter *)
    P101A_RUNTIME := P101A_RUNTIME + TASK_PERIOD_S;
END_IF;

IF MOTOR_P101A.STATUS = 3 THEN
    (* Motor in fault — log and notify *)
    FAULT_COUNT := FAULT_COUNT + 1;
    CALL SEND_ALARM(
        TAG      := "P101A",
        MSG      := "MOTOR FAULT - FEEDBACK TIMEOUT",
        PRIORITY := 2
    );
END_IF;

(* Cascade start from sequence *)
IF SEQUENCE_STEP = 5 AND NOT MOTOR_P101A.STATUS = 2 THEN
    MOTOR_P101A.SP   := 1;    (* Command START *)
    MOTOR_P101A.MODE := AUTO;
END_IF;
```

---

## Honeywell Bulk Engineering

### Quick Builder — primary bulk tool

Honeywell Quick Builder is the native tool for bulk point configuration in Experion. Points are defined in a spreadsheet-like interface and can be imported/exported via CSV.

### CSV point import format

The CSV import format is version-specific (check your Experion release documentation), but the core fields are consistent:

```csv
TagName,TagType,Description,NodeName,Controller,PointGroup,InitSP,EU,EUHigh,EULow,AlarmHighHigh,AlarmHigh,AlarmLow,AlarmLowLow,FeedbackTime,FeedbackStrategy
P101A,DISTATE,FEED PUMP A MOTOR,NODE1,C300_01,GRP_11301,0,,,,,,,,10.0,2
P101B,DISTATE,FEED PUMP B MOTOR,NODE1,C300_01,GRP_11301,0,,,,,,,,10.0,2
FIC056A,AI,FLOW CONTROLLER 056A,NODE1,C300_01,GRP_11301,0.0,m3/h,1000.0,0.0,950.0,900.0,50.0,20.0,,
TIC201,AI,TEMP CONTROLLER 201,NODE1,C300_01,GRP_11301,20.0,degC,250.0,-10.0,230.0,220.0,0.0,-5.0,,
```

**Required fields for DISTATE (motor/discrete) points:**

| Column | Required | Description |
|--------|---------|-------------|
| `TagName` | Yes | Point tag (16 chars max in older releases) |
| `TagType` | Yes | `DISTATE`, `AI`, `AO`, `DI`, `DO`, `ACCUMUL`, `RECIPE` |
| `Description` | Yes | 40 chars max |
| `NodeName` | Yes | Experion server node name |
| `Controller` | Yes | C300 controller tag |
| `PointGroup` | Rec. | Grouping for display and alarming |
| `InitSP` | Yes | Initial setpoint value |
| `EU` | For AI | Engineering units string |
| `EUHigh` | For AI | Engineering unit high range |
| `EULow` | For AI | Engineering unit low range |
| `AlarmHighHigh` | No | HH alarm setpoint |
| `AlarmHigh` | No | H alarm setpoint |
| `AlarmLow` | No | L alarm setpoint |
| `AlarmLowLow` | No | LL alarm setpoint |
| `FeedbackTime` | For DISTATE | Feedback timeout (seconds) |
| `FeedbackStrategy` | For DISTATE | 1=run only, 2=run+stop |

### CSV bulk import workflow

```
1. Prepare CSV in Excel:
   - Use UTF-8 encoding
   - First row = column headers exactly as defined
   - Validate TagName uniqueness before import

2. In Experion Engineering Tool:
   Quick Builder → File → Import → Browse to CSV

3. Map CSV columns to Quick Builder fields
   (if column names do not exactly match, manual mapping required)

4. Review validation report:
   - RED = blocking error (tag name conflict, controller not found)
   - YELLOW = warning (EU range reversal, alarm outside EU range)

5. Apply import (creates or updates points)

6. Download point changes to controller:
   Quick Builder → Download → Select controllers → Apply

7. Verify in Experion Operate:
   Load faceplates for imported points, confirm correct display
```

### Python CSV generation for Honeywell Experion

```python
import csv
from dataclasses import dataclass, fields
from typing import Optional
from pathlib import Path


@dataclass
class ExperionDiscretePoint:
    """Honeywell Experion DISTATE (motor/discrete) point definition."""
    TagName:          str
    TagType:          str = 'DISTATE'
    Description:      str = ''
    NodeName:         str = ''
    Controller:       str = ''
    PointGroup:       str = ''
    InitSP:           int = 0
    EU:               str = ''
    EUHigh:           str = ''
    EULow:            str = ''
    AlarmHighHigh:    str = ''
    AlarmHigh:        str = ''
    AlarmLow:         str = ''
    AlarmLowLow:      str = ''
    FeedbackTime:     float = 10.0
    FeedbackStrategy: int = 2


@dataclass
class ExperionAnalogPoint:
    """Honeywell Experion AI (analog input) point definition."""
    TagName:          str
    TagType:          str = 'AI'
    Description:      str = ''
    NodeName:         str = ''
    Controller:       str = ''
    PointGroup:       str = ''
    InitSP:           float = 0.0
    EU:               str = ''
    EUHigh:           float = 100.0
    EULow:            float = 0.0
    AlarmHighHigh:    Optional[float] = None
    AlarmHigh:        Optional[float] = None
    AlarmLow:         Optional[float] = None
    AlarmLowLow:      Optional[float] = None
    FeedbackTime:     str = ''
    FeedbackStrategy: str = ''


_CSV_HEADERS = [
    'TagName', 'TagType', 'Description', 'NodeName', 'Controller',
    'PointGroup', 'InitSP', 'EU', 'EUHigh', 'EULow',
    'AlarmHighHigh', 'AlarmHigh', 'AlarmLow', 'AlarmLowLow',
    'FeedbackTime', 'FeedbackStrategy',
]


def _point_to_row(point) -> dict:
    """Convert a dataclass point to a CSV row dict."""
    row = {}
    for f in fields(point):
        val = getattr(point, f.name)
        row[f.name] = '' if val is None else val
    return row


def generate_experion_csv(
    discrete_points: list[ExperionDiscretePoint],
    analog_points: list[ExperionAnalogPoint],
    output_path: str,
) -> int:
    """
    Generate Honeywell Experion Quick Builder CSV import file.

    Returns: total number of points written
    """
    rows = [_point_to_row(p) for p in discrete_points + analog_points]

    with open(output_path, 'w', newline='', encoding='utf-8') as f:
        writer = csv.DictWriter(f, fieldnames=_CSV_HEADERS)
        writer.writeheader()
        writer.writerows(rows)

    print(f"Generated Experion CSV: {output_path} ({len(rows)} points)")
    return len(rows)


# Example usage
if __name__ == '__main__':
    motors = [
        ExperionDiscretePoint(
            TagName='P101A', Description='FEED PUMP A 11KW',
            NodeName='NODE1', Controller='C300_01', PointGroup='GRP_11301',
            FeedbackTime=10.0, FeedbackStrategy=2,
        ),
        ExperionDiscretePoint(
            TagName='P101B', Description='FEED PUMP B 11KW',
            NodeName='NODE1', Controller='C300_01', PointGroup='GRP_11301',
            FeedbackTime=10.0, FeedbackStrategy=2,
        ),
    ]

    ai_points = [
        ExperionAnalogPoint(
            TagName='FIC056A', Description='FLOW CONTROLLER 056A',
            NodeName='NODE1', Controller='C300_01', PointGroup='GRP_11301',
            EU='m3/h', EUHigh=1000.0, EULow=0.0,
            AlarmHighHigh=950.0, AlarmHigh=900.0, AlarmLow=50.0, AlarmLowLow=20.0,
        ),
    ]

    generate_experion_csv(motors, ai_points, 'experion_bulk_import.csv')
```

---

## DeltaV Corner Cases

### Mode naming mismatch vs ISA

DeltaV's mode names do NOT match the intuitive operator interpretation. This causes persistent confusion:

- **MAN** in DeltaV = "operator gives the command manually from the DCS faceplate" — the operator types or clicks the setpoint
- **AUTO** in DeltaV = "the DCS block automatically manages the setpoint" based on internal logic
- **CAS** in DeltaV = same as AUTO, but the setpoint source is another block's output

An operator saying "put it in manual" means they want manual control — that is DeltaV `MAN` mode. An operator saying "put it in auto" means the DCS should control it automatically — that is DeltaV `AUTO` or `CAS` mode.

### DC block F_IN vs IN_D wiring methods

There are two ways to wire feedback signals to a DC block:

**Method 1 (standard): Through DI function blocks**
```fhx
PARAMETER_CONNECTION SOURCE="DI-1/PV_D" DEST="DC-1/IN_D1";
```
Use this when the digital input is within the same Control Module.

**Method 2: External wiring using F_IN_D pins**
```fhx
PARAMETER_CONNECTION SOURCE="OTHER_MODULE/DI-1/PV_D" DEST="DC-1/F_IN_D1";
```
Use this when the DI block lives in a different Control Module (e.g., a shared I/O module) or when FOUNDATION Fieldbus device feedback comes from a separate CM.

### Module period and I/O scan timing

DeltaV executes modules at their configured `PERIOD`. The minimum period for standard I/O modules is 100ms. The period must be a multiple of 100ms for most controller configurations.

For CHARM (Characterization Module) I/O, the minimum scan period is 50ms.

For FOUNDATION Fieldbus, the module period should be >= 2× the fieldbus macrocycle (typically >= 200ms for a typical FF segment).

### Template propagation conflicts

When a Module Template is modified and changes are propagated to instances:
- Parameters that have been **explicitly overridden** on an instance are **NOT overwritten** by propagation
- Parameters that are still **at template default** are updated by propagation
- If the template change is structural (adds or removes a function block), all instances must be re-validated and re-downloaded

### Named Set references

Named Sets provide string descriptions for integer state values and are referenced by Display/Alarm blocks. If a Named Set referenced by a module is deleted or renamed, the module will show raw integer states instead of descriptions. Always validate Named Set references after bulk import.

---

## Honeywell Corner Cases

### Tag name length limit

Older Experion PKS releases (< R500) enforce a **16-character maximum** tag name. Newer releases support longer names, but legacy limits apply when migrating from TDC-3000 or TPS systems. The 16-character limit impacts ISA-5.1 compound tags: `11301.CLWW.1A1` is 14 chars (with dots) — fits, but `AREA11301.FIC.056A` does not.

**Resolution:** Use abbreviated area codes (`A113` instead of `AREA_11301`) and omit dots.

### CL algorithm execution context

CL programs in a CAB run in the context of the C300 controller scan cycle. CL does not support true multitasking — all CL logic in a given control module executes sequentially within one scan. Long CL programs can cause scan overrun alarms if execution time exceeds the module period.

### CSV import field mapping sensitivity

Experion Quick Builder CSV import is case-sensitive for column headers and sensitive to extra spaces. Best practices:
- Remove trailing spaces from all fields before import
- Use exact column header names from the Experion version documentation
- Set TagType to uppercase (`AI` not `ai`)
- Verify Controller names exist in the system before import

### DISTATE vs DI point type

Honeywell distinguishes between:
- `DI` — Simple digital input (value only, no state machine)
- `DISTATE` — Digital state (with defined state names, alarms, and optionally feedback monitoring)
- `MOTOR` — Dedicated motor type (if licensed; not available in all Experion versions)

For motor control, use `DISTATE` with appropriate state definitions if the `MOTOR` type is not available. The difference is that `MOTOR` provides pre-built start/stop logic, feedback monitoring, and standardized faceplate; `DISTATE` requires CL logic for feedback monitoring.

### SCM (Sequence Control Module) for ISA-88

Honeywell implements ISA-88 batch/sequential control through SCM, not through a phase-based module model like DeltaV. An SCM contains SFC-based sequential logic and can coordinate multiple points. SCMs are Honeywell's nearest equivalent to DeltaV's Phase + Equipment Module model, but the architecture is less formally structured against ISA-88 levels.

---

## Platform Comparison Table

| Feature | Emerson DeltaV | Honeywell Experion PKS |
|---------|---------------|------------------------|
| Primary bulk engineering | Class-based templates + FHX | Quick Builder CSV import |
| Template/class mechanism | Module Templates (class → instance hierarchy) | CAB (Custom Algorithm Block) — functional only |
| Structural change propagation | Automatic (template → all instances) | Manual re-import / re-download |
| Configuration file format | FHX (UTF-16 text, Pascal-like) | CSV (UTF-8, flat) |
| ISA-88 batch support | Native — Phase, EM, Recipe hierarchy | Via SCM — less formal |
| Batch recipe management | DeltaV Batch Executive (RTE) | Separate Experion Batch software |
| Programming languages | FBD, SFC, limited ST (CALC blocks), FHX | FBD, CL (proprietary ST-like), limited ST on ControlEdge |
| IEC 61131-3 compliance | Partial (FBD/SFC native; ST limited) | Via ControlEdge controller |
| Motor control block | DC (Device Control) — multi-state | DISTATE / MOTOR point type |
| Motor modes | AUTO / MAN / CAS / OOS / LO | AUTO / MAN / OOS |
| Historian | DeltaV Continuous Historian (built-in) | Uniformance PHD (separate product) |
| HMI | DeltaV Operate (classic) + DeltaV Live (web) | HMIWeb (web-based) |
| OPC DA / OPC UA | Full support | Full support |
| Best application | Pharma, biotech, food & bev, specialty chem | Large oil & gas, LNG, petrochemical |
| Controller max modules | 750 per controller | ~1000+ points per C300 |
| Min module scan period | 100ms (CHARM: 50ms) | 100ms |
| Engineering unit | Class-based = high reuse, fast large projects | CSV-based = simple, direct, familiar to spreadsheet users |
