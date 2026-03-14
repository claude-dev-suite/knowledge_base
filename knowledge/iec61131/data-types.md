# IEC 61131-3 - Data Types, POUs, and Standard Library

> Sources:
> - https://www.plcacademy.com/structured-text-tutorial/
> - https://controlforge.dev/docs/standard-function-blocks
> - https://en.wikipedia.org/wiki/IEC_61131-3
> - https://gist.github.com/fisothemes/9c5b88c1c650970876f400c689f9951c (IEC-61131-3 Extended Language Spec)
> - https://www.motioncontroltips.com/faq-plc-function-blocks-iec-61131-3-classify/
> - https://forge.codesys.com/forge/talk/Engineering/thread/ed30e6d1a1/ (RETAIN vs PERSISTENT)
> - https://www.automation.com/en-us/articles/2003-1/coder-s-corner-the-iec-61131-3-software-model
> - https://stefanhenneken.net/2016/09/27/iec-61131-3-arrays-with-variable-length/
> - https://www.realpars.com/blog/structured-text
> - https://www.fernhillsoftware.com/help/iec-61131/common-elements/derived-data-types/array.html

## Overview

IEC 61131-3 (third edition 2013) is the international standard for PLC programming languages. It defines five programming languages, the Program Organization Unit (POU) model, all elementary and derived data types, variable scopes, a standard function library, and the runtime configuration model. This document covers everything needed to write industrial automation logic and to generate IEC 61131-3 code programmatically.

---

## Table of Contents

1. [Program Organization Units (POUs)](#program-organization-units-pous)
2. [Elementary Data Types](#elementary-data-types)
3. [Derived Data Types](#derived-data-types)
4. [Variable Scopes and Declarations](#variable-scopes-and-declarations)
5. [Standard Function Blocks (IEC Standard Library)](#standard-function-blocks)
6. [Standard Functions](#standard-functions)
7. [POU Instantiation and Calling](#pou-instantiation-and-calling)
8. [Task Configuration](#task-configuration)
9. [Data Type Conversions](#data-type-conversions)
10. [Corner Cases](#corner-cases)
11. [Vendor-Specific Extensions](#vendor-specific-extensions)

---

## Program Organization Units (POUs)

IEC 61131-3 defines three types of POUs. Every executable piece of code belongs to one of these types. They exist in a strict hierarchy: PROGRAM is the entry point; FUNCTION_BLOCK is the workhorse; FUNCTION is the pure calculation unit.

### PROGRAM

The top-level executable POU. Associated directly with a runtime Task (scan cycle). Cannot be called by other POUs — it is the entry point invoked by the scheduler. Each PROGRAM instance is unique; you cannot create multiple instances of a PROGRAM.

```pascal
PROGRAM MainMotorControl
VAR
    Motor_P101A  : MotorControl_FB;
    Motor_P101B  : MotorControl_FB;
    PID_FIC056   : PID_FB;
    ScanCount    : UDINT;
END_VAR

(* Body: executed every scan cycle *)
Motor_P101A(
    StartCmd  := Start_P101A,
    StopCmd   := Stop_P101A,
    FbkRun    := DI_P101A_Run,
    Interlock := ILK_P101A
);

Motor_P101B(
    StartCmd  := Start_P101B,
    StopCmd   := Stop_P101B,
    FbkRun    := DI_P101B_Run,
    Interlock := ILK_P101B
);

PID_FIC056(PV := FlowPV, SP := FlowSP);
ScanCount := ScanCount + 1;

END_PROGRAM
```

**Key PROGRAM characteristics:**
- Only **one** PROGRAM instance per task (cannot be multiply-instantiated)
- Can declare global variables via `VAR_GLOBAL` (shared with other POUs)
- In DCS systems, each CFC (Continuous Function Chart) sheet is effectively a PROGRAM
- Direct access to physical I/O is only permitted from PROGRAM (or via `VAR_EXTERNAL` global references)

---

### FUNCTION_BLOCK (FB)

The most important POU type for industrial automation. FBs have **persistent internal state** — their `VAR` (internal) variables retain their values between calls. This is the fundamental difference from FUNCTION: a FB "remembers" its past state.

Each FB **instance** has its own independent memory. Multiple instances of the same FB type run the same logic but maintain completely separate state.

```pascal
FUNCTION_BLOCK MotorControl_FB
VAR_INPUT
    StartCmd   : BOOL;        (* Operator/sequence start command *)
    StopCmd    : BOOL;        (* Operator/sequence stop command *)
    FbkRun     : BOOL;        (* Running feedback from field (DI) *)
    FbkStp     : BOOL;        (* Stopped feedback from field (DI) *)
    Interlock  : BOOL;        (* Interlock permissive: TRUE = OK to start *)
    Protection : BOOL;        (* Protection (overload, etc.): TRUE = healthy *)
    ResetCmd   : BOOL;        (* Fault reset command *)
    MonitorTime: TIME := T#10s; (* Startup monitor timeout — configurable per instance *)
END_VAR

VAR_OUTPUT
    StartOut   : BOOL;        (* Start command to field (DO) *)
    Running    : BOOL;        (* TRUE when motor confirmed running *)
    Faulted    : BOOL;        (* TRUE when in fault state *)
    Blocked    : BOOL;        (* TRUE when blocked by interlock *)
    InAuto     : BOOL;        (* Informational: auto mode active *)
END_VAR

VAR
    (* Internal state — PERSISTENT between calls *)
    State      : INT := 0;    (* 0=Idle 1=Starting 2=Running 3=Stopping 99=Fault *)
    MonTimer   : TON;         (* Startup monitor timer — own instance per FB instance *)
    FaultCount : INT := 0;    (* Trip counter — accumulates, RETAIN candidate *)
    PrevStart  : BOOL;        (* Previous scan StartCmd — for edge detection *)
END_VAR

(* ── State machine — executes every scan ── *)
Blocked := NOT Interlock OR NOT Protection;
InAuto  := (State = 2);

CASE State OF

    0: (* IDLE — motor stopped and healthy *)
        StartOut := FALSE;
        Running  := FALSE;
        MonTimer(IN := FALSE);   (* Keep timer reset *)
        IF StartCmd AND NOT Blocked AND NOT Faulted THEN
            State := 1;
        END_IF;

    1: (* STARTING — command issued, waiting for feedback *)
        StartOut := TRUE;
        MonTimer(IN := TRUE, PT := MonitorTime);
        IF FbkRun THEN
            State   := 2;
            Faulted := FALSE;
        ELSIF MonTimer.Q THEN
            (* No run feedback within MonitorTime → fault *)
            State      := 99;
            Faulted    := TRUE;
            FaultCount := FaultCount + 1;
        END_IF;

    2: (* RUNNING — motor confirmed running *)
        StartOut := TRUE;
        Running  := TRUE;
        IF StopCmd OR Blocked THEN
            State := 3;
        END_IF;
        IF NOT FbkRun THEN
            (* Unexpected loss of run feedback *)
            State      := 99;
            Faulted    := TRUE;
            FaultCount := FaultCount + 1;
        END_IF;

    3: (* STOPPING — stop issued, waiting for feedback *)
        StartOut := FALSE;
        Running  := FALSE;
        IF FbkStp OR NOT FbkRun THEN
            State := 0;
        END_IF;

    99: (* FAULTED — operator must reset *)
        StartOut := FALSE;
        Running  := FALSE;
        IF ResetCmd AND NOT Blocked AND NOT StartCmd THEN
            State   := 0;
            Faulted := FALSE;
        END_IF;

END_CASE;

END_FUNCTION_BLOCK
```

**Key FB characteristics:**
- Each instance has its own copy of all `VAR` (internal) variables — `Motor_P101A.State` and `Motor_P101B.State` are completely independent
- `VAR_INPUT` and `VAR_OUTPUT` are the external interface pins
- The FB body executes every scan cycle when called
- FBs **must be instantiated** — you declare a variable of the FB type and use that variable
- Standard library blocks (TON, CTU, R_TRIG, etc.) are themselves FBs — each instance is independent

---

### FUNCTION (FC)

Stateless computation — **no internal memory**. Returns exactly one value (the function name itself). Always produces the same output for the same inputs (referential transparency). Used for pure calculations, conversions, and utility operations.

A FUNCTION may have additional outputs via `VAR_OUTPUT`, but these are passed by value (not by reference). Unlike a FB, a FUNCTION has no persistent `VAR` section.

```pascal
FUNCTION ScaleAnalog : REAL
(* Scales a raw 16-bit ADC integer to engineering units.
   Same inputs always produce same output — no side effects. *)
VAR_INPUT
    RawValue : INT;      (* Raw ADC count, e.g., 0–32767 *)
    RawMin   : INT;      (* Raw count at EULow *)
    RawMax   : INT;      (* Raw count at EUHigh *)
    EULow    : REAL;     (* Engineering units at RawMin *)
    EUHigh   : REAL;     (* Engineering units at RawMax *)
END_VAR
VAR_TEMP
    RawRange : REAL;     (* Temporary — not persistent, recalculated each call *)
    EURange  : REAL;
END_VAR

RawRange := INT_TO_REAL(RawMax - RawMin);
EURange  := EUHigh - EULow;

IF RawRange <> 0.0 THEN
    ScaleAnalog := EULow + (INT_TO_REAL(RawValue - RawMin) / RawRange) * EURange;
ELSE
    ScaleAnalog := EULow;   (* Protect against divide-by-zero *)
END_IF;

END_FUNCTION
```

```pascal
FUNCTION AlarmCheck : BOOL
(* Returns TRUE if value is outside deadbanded limits. *)
VAR_INPUT
    Value      : REAL;
    HighLimit  : REAL;
    LowLimit   : REAL;
    Deadband   : REAL;
END_VAR

AlarmCheck := (Value > (HighLimit + Deadband)) OR (Value < (LowLimit - Deadband));

END_FUNCTION
```

**Key FC characteristics:**
- No `VAR` section (only `VAR_INPUT`, `VAR_OUTPUT`, `VAR_TEMP`)
- Return value is the function name (e.g., `ScaleAnalog := ...`)
- No instantiation needed — called directly: `result := ScaleAnalog(raw, 0, 32767, 0.0, 100.0)`
- **Reentrancy:** Functions are inherently reentrant (no shared state). Multiple simultaneous calls are safe
- VAR_TEMP variables are NOT persistent — reset every call

---

### POU comparison table

| Feature | PROGRAM | FUNCTION_BLOCK | FUNCTION |
|---------|---------|----------------|---------|
| Has internal state (VAR) | Yes (one instance) | Yes (per instance) | No |
| Can be multiply instantiated | No | Yes | N/A (no instance) |
| Return value | No | No | Yes (the function name) |
| Called by | Runtime (task) | PROGRAM or FB | PROGRAM, FB, or FC |
| Can call FBs | Yes | Yes | No |
| Can call FCs | Yes | Yes | Yes |
| Can call PROGRAM | No | No | No |
| Reentrancy | N/A | Reentrant (separate instances) | Inherently reentrant |
| I/O access | Direct | Via parameters | Via parameters |

---

## Elementary Data Types

### Boolean

| Type | Size | Range | Literal Syntax |
|------|------|-------|---------------|
| `BOOL` | 1 bit | FALSE / TRUE | `TRUE`, `FALSE` |

### Integer types

| Type | Size | Range | Literal Syntax |
|------|------|-------|---------------|
| `SINT` | 8 bit | −128 to 127 | `127`, `-128` |
| `USINT` | 8 bit | 0 to 255 | `255` |
| `INT` | 16 bit | −32 768 to 32 767 | `1000`, `-500` |
| `UINT` | 16 bit | 0 to 65 535 | `65535` |
| `DINT` | 32 bit | −2 147 483 648 to 2 147 483 647 | `1_000_000` (underscores optional in some vendors) |
| `UDINT` | 32 bit | 0 to 4 294 967 295 | `4294967295` |
| `LINT` | 64 bit | −9.2×10¹⁸ to 9.2×10¹⁸ | `9223372036854775807` |
| `ULINT` | 64 bit | 0 to 1.8×10¹⁹ | `18446744073709551615` |

**Hex/binary/octal literals:**
```pascal
VAR
    HexByte   : BYTE  := 16#FF;      (* Hex literal: 255 *)
    BinWord   : WORD  := 2#1111_0000_1010_0101;  (* Binary literal *)
    OctalVal  : USINT := 8#77;       (* Octal literal: 63 *)
END_VAR
```

### Real (floating point)

| Type | Size | Precision | Range | Literal Syntax |
|------|------|-----------|-------|---------------|
| `REAL` | 32 bit | ~7 significant digits | ±3.4×10³⁸ | `3.14`, `1.0E6`, `-0.001` |
| `LREAL` | 64 bit | ~15 significant digits | ±1.7×10³⁰⁸ | `3.14159265358979` |

**Note for process control:** `REAL` (32-bit float) provides ~0.0001% resolution for 4–20 mA signals — more than adequate. Use `LREAL` only for accumulated values (totalizers), time calculations, or high-precision setpoint arithmetic.

### Bit string types

| Type | Size | Purpose | Operations |
|------|------|---------|-----------|
| `BOOL` | 1 bit | Boolean logic | AND, OR, XOR, NOT |
| `BYTE` | 8 bits | Byte-level bit manipulation | AND, OR, XOR, NOT, SHL, SHR |
| `WORD` | 16 bits | Register access, status words | AND, OR, XOR, NOT, SHL, SHR, ROL, ROR |
| `DWORD` | 32 bits | 32-bit status/configuration | Same as WORD |
| `LWORD` | 64 bits | 64-bit packed data | Same as WORD |

**Bit extraction from WORD:**
```pascal
VAR
    StatusWord : WORD := 16#00A3;
    Bit0       : BOOL;
    Bit7       : BOOL;
END_VAR

Bit0 := StatusWord.0;    (* Extract bit 0 — vendor extension in some platforms *)
Bit7 := StatusWord.7;    (* Extract bit 7 *)

(* Standard-compliant bit extraction using masking: *)
Bit0 := (StatusWord AND 16#0001) <> 0;
Bit7 := (StatusWord AND 16#0080) <> 0;
```

### Time types

| Type | Size | Format | Literal Example |
|------|------|--------|----------------|
| `TIME` | 32/64 bit | Duration | `T#5s`, `T#1h30m`, `T#500ms`, `T#2h15m30s100ms` |
| `DATE` | 32 bit | Calendar date | `D#2024-03-14` |
| `TIME_OF_DAY` | 32 bit | Time within a day | `TOD#14:30:00`, `TOD#08:00:00.500` |
| `DATE_AND_TIME` | 64 bit | Full timestamp | `DT#2024-03-14-14:30:00` |

**TIME literal components:** `d` (days), `h` (hours), `m` (minutes), `s` (seconds), `ms` (milliseconds)
```pascal
T#10d4h38m57s12ms    (* 10 days, 4 hours, 38 min, 57 sec, 12 ms *)
T#500ms              (* 500 milliseconds *)
T#0s                 (* Zero duration *)
```

### String types

| Type | Description | Default Length | Literal |
|------|-------------|---------------|---------|
| `STRING` | Single-byte ASCII/Latin-1 | 80 characters | `'Hello World'` |
| `STRING[n]` | Single-byte, max n characters | n characters | `'Tag'` assigned to `STRING[32]` |
| `WSTRING[n]` | Wide (Unicode) string | n characters | `"Hello"` (double-quoted) |

**Important:** `STRING` in IEC 61131-3 is **fixed maximum length** — the variable holds up to n characters but the memory allocation is always the declared maximum. Assigning a longer string truncates silently.

```pascal
VAR
    TagName    : STRING[32] := 'FIC-101A';
    Description: STRING[80] := 'FEED FLOW INDICATING CONTROLLER';
    ShortStr   : STRING;               (* Default 80 chars *)
    OversizeTest : STRING[4] := 'ABCDEFGH';  (* Truncated to 'ABCD' — no error *)
END_VAR

(* String operations *)
TagLen   := LEN(TagName);             (* Returns 8 *)
IsPrefix := LEFT(TagName, 3) = 'FIC'; (* TRUE *)
FullName := CONCAT(TagName, '_SP');   (* 'FIC-101A_SP' *)
```

---

## Derived Data Types

### STRUCT

User-defined structured type grouping related variables. Structs enable passing multiple values together as a single variable:

```pascal
TYPE MotorData :
STRUCT
    Running       : BOOL;
    Faulted       : BOOL;
    Blocked       : BOOL;
    InAuto        : BOOL;
    State         : INT;
    Speed_rpm     : REAL;
    Current_A     : REAL;
    Vibration_mms : REAL;     (* mm/s RMS *)
    Runtime_h     : UDINT;   (* Total running hours *)
    TripCount     : INT;
    TagName       : STRING[32];
    Description   : STRING[80];
END_STRUCT
END_TYPE


TYPE AnalogAlarmConfig :
STRUCT
    Priority      : USINT;   (* 1=Critical/SIL 2=High 3=Medium 4=Low *)
    HHLimit       : REAL;
    HLimit        : REAL;
    LLimit        : REAL;
    LLLimit       : REAL;
    Deadband      : REAL;
    OnDelay_ms    : UDINT;   (* Milliseconds before alarm activates *)
    OffDelay_ms   : UDINT;   (* Milliseconds before alarm clears *)
    Shelved       : BOOL;
    ShelveExpiry  : DATE_AND_TIME;
END_STRUCT
END_TYPE
```

**Struct access:**
```pascal
VAR
    Motor1 : MotorData;
END_VAR

Motor1.Running    := TRUE;
Motor1.Speed_rpm  := 1480.5;
Motor1.TagName    := 'P-101A';
```

### ENUM

Named integer constants — use ENUM instead of magic numbers in state machines:

```pascal
TYPE MotorState :
(
    STATE_IDLE      := 0,
    STATE_STARTING  := 1,
    STATE_RUNNING   := 2,
    STATE_STOPPING  := 3,
    STATE_FAULTED   := 99
);
END_TYPE


TYPE ValvePosition :
(
    VP_CLOSED       := 0,
    VP_OPENING      := 1,
    VP_OPEN         := 2,
    VP_CLOSING      := 3,
    VP_FAULT        := 99
);
END_TYPE
```

**ENUM usage:**
```pascal
VAR
    State  : MotorState := STATE_IDLE;
    VPos   : ValvePosition;
END_VAR

CASE State OF
    STATE_IDLE:
        IF StartCmd THEN State := STATE_STARTING; END_IF;
    STATE_STARTING:
        ...
    STATE_FAULTED:
        ...
END_CASE;

(* Direct comparison — readable, no magic numbers *)
IF State = STATE_RUNNING THEN ... END_IF;
```

### ARRAY

Fixed-size arrays of any type. IEC 61131-3 supports:
- Any integer bounds (including negative)
- Multidimensional arrays (multiple dimension ranges)
- Arrays of structs, arrays of strings

```pascal
VAR
    (* 1D arrays *)
    MotorRunning   : ARRAY[1..200] OF BOOL;    (* 200 motor run statuses *)
    AlarmLimits    : ARRAY[1..4] OF REAL;      (* [1]=HH [2]=H [3]=L [4]=LL *)
    TagNames       : ARRAY[0..99] OF STRING[32];

    (* Multidimensional *)
    HeatMatrix     : ARRAY[1..10, 1..10] OF REAL;   (* 10×10 temperature matrix *)
    ZoneAlarms     : ARRAY[1..5, 1..4] OF BOOL;     (* 5 zones × 4 alarm levels *)

    (* Negative index bounds — valid in IEC 61131-3 *)
    OffsetBuffer   : ARRAY[-5..5] OF REAL;    (* 11 elements, index -5 to +5 *)
    NegRange       : ARRAY[-3..-1] OF LREAL;  (* 3 elements at indices -3, -2, -1 *)

    (* Array of structs *)
    Motors         : ARRAY[1..20] OF MotorData;
END_VAR

(* Access *)
MotorRunning[1]   := TRUE;
AlarmLimits[1]    := 95.0;    (* HH limit in % *)
HeatMatrix[3, 4]  := 125.5;  (* Row 3, column 4 *)
OffsetBuffer[-3]  := 0.0;    (* Access with negative index *)
Motors[1].Running := TRUE;   (* Access struct member in array *)
```

**LOWER_BOUND / UPPER_BOUND functions (runtime bounds check):**
```pascal
VAR
    i : INT;
END_VAR

(* Iterate over any array with any bounds *)
FOR i := LOWER_BOUND(MotorRunning, 1) TO UPPER_BOUND(MotorRunning, 1) DO
    MotorRunning[i] := FALSE;
END_FOR;
```

### SUBRANGE

Integer type with compile-time restricted range:
```pascal
TYPE AlarmPriority  : USINT (1..4);  END_TYPE
TYPE MotorNumber    : UINT (1..200); END_TYPE
TYPE PercentValue   : REAL (0.0..100.0);  END_TYPE  (* vendor-specific, often unsupported *)
```

---

## Variable Scopes and Declarations

| Keyword | Scope | Persistence Between Calls | Passing Convention | Notes |
|---------|-------|--------------------------|-------------------|-------|
| `VAR` | Local to POU | Yes (FB/PROGRAM) | N/A | Main internal state |
| `VAR_INPUT` | Input interface | Value refreshed by caller | By value (copy) | Read-only inside FB |
| `VAR_OUTPUT` | Output interface | Yes (FB keeps last value) | By value (copy) | Written by FB, read by caller |
| `VAR_IN_OUT` | Bidirectional | Via reference | **By reference (pointer)** | FB can modify caller's variable |
| `VAR_TEMP` | Local temporary | **No** (reset each call) | N/A | For intermediate calculations |
| `VAR_GLOBAL` | All POUs in configuration | Yes | N/A | Must be declared once |
| `VAR_EXTERNAL` | References a VAR_GLOBAL | Yes | N/A | Must match global type exactly |
| `VAR_CONFIG` | Configuration level | Set at project download | N/A | I/O addresses, `AT` directives |
| `CONSTANT` / `VAR CONSTANT` | Named literal | N/A | N/A | Initialized only, never changes |

### VAR_IN_OUT vs VAR_OUTPUT — the critical difference

`VAR_OUTPUT` passes the output **by value** back to the caller. The caller reads it after the FB call. The caller's variable is updated by assignment.

`VAR_IN_OUT` passes the variable **by reference** (pointer). The FB receives a pointer to the caller's actual variable and can read and modify it directly. This has two consequences:
1. The input value is available inside the FB without copying
2. **A constant cannot be passed as VAR_IN_OUT** — only a mutable variable

```pascal
FUNCTION_BLOCK BufferManager_FB
VAR_IN_OUT
    DataBuffer : STRING[512];    (* FB reads AND writes the caller's buffer *)
    (* Cannot call with: BufferManager(DataBuffer := 'literal_string') *)
    (* Must call with:   BufferManager(DataBuffer := myStringVar) *)
END_VAR
VAR_OUTPUT
    BytesWritten : UDINT;        (* FB writes, caller reads — passed by value *)
END_VAR

(* FB body can modify DataBuffer directly — changes visible in caller *)
DataBuffer := CONCAT(DataBuffer, 'NEW_DATA');
BytesWritten := LEN(DataBuffer);
END_FUNCTION_BLOCK
```

**When to use VAR_IN_OUT:**
- When the FB needs to modify a large data structure (avoids copying)
- When implementing shared buffers, circular queues, or large structs
- When using the PERSISTENT best practice: pass PERSISTENT data as VAR_IN_OUT to avoid RETAIN on entire FB instances

### VAR_EXTERNAL and VAR_GLOBAL

```pascal
(* Declared once in a global variables block: *)
VAR_GLOBAL
    SystemTime      : DATE_AND_TIME;
    EmergencyStop   : BOOL;
    PlantMode       : INT;    (* 0=Shutdown 1=Standby 2=Production *)
    SystemAlarmCount: UDINT;
END_VAR

(* Referenced in any POU using VAR_EXTERNAL: *)
FUNCTION_BLOCK AlarmLogger_FB
VAR_EXTERNAL
    SystemTime       : DATE_AND_TIME;   (* Read from global *)
    SystemAlarmCount : UDINT;           (* Write to global — shared counter *)
END_VAR
VAR_INPUT
    AlarmCondition : BOOL;
END_VAR
(* Body can use SystemTime and SystemAlarmCount directly *)
END_FUNCTION_BLOCK
```

### Complete variable scope example

```pascal
VAR_GLOBAL
    EmergencyStop  : BOOL;
    PlantMode      : INT;
END_VAR

FUNCTION_BLOCK FullScopeDemo_FB
VAR_INPUT
    Enable        : BOOL;              (* Input: caller sets each scan *)
END_VAR
VAR_OUTPUT
    Active        : BOOL;              (* Output: FB sets, caller reads *)
    EventCount    : UDINT;             (* Persistent output counter *)
END_VAR
VAR_IN_OUT
    SharedLog     : STRING[256];       (* Bidirectional: passed by reference *)
END_VAR
VAR
    InternalState : INT := 0;          (* Internal: persistent between calls *)
    LastEnable    : BOOL;              (* Previous scan value for edge detection *)
END_VAR
VAR_TEMP
    TempCalc      : REAL;              (* Temporary: recalculated every call *)
    TempMsg       : STRING[64];        (* Temporary string — not saved *)
END_VAR
VAR_EXTERNAL
    EmergencyStop : BOOL;              (* References VAR_GLOBAL *)
END_VAR
VAR CONSTANT
    VERSION       : STRING := '2.1.0';
    MAX_EVENTS    : UDINT  := 1_000_000;
END_VAR

(* Body *)
TempCalc := 3.14159 * 2.0;            (* VAR_TEMP — loses value after this scan *)
IF Enable AND NOT EmergencyStop THEN
    Active     := TRUE;
    EventCount := EventCount + 1;      (* VAR_OUTPUT — persistent *)
    IF EventCount > MAX_EVENTS THEN
        EventCount := 0;
    END_IF;
END_IF;
LastEnable := Enable;                  (* VAR — persistent *)

END_FUNCTION_BLOCK
```

---

## Standard Function Blocks

### Timers

#### TON — Timer On-Delay

Delays the TRUE output signal by the preset time. Output `Q` goes TRUE only after `IN` has been continuously TRUE for `PT` duration. Resetting `IN` to FALSE resets the timer.

```pascal
FUNCTION_BLOCK TON
VAR_INPUT
    IN : BOOL;    (* Enable: timing starts when IN goes TRUE *)
    PT : TIME;    (* Preset time *)
END_VAR
VAR_OUTPUT
    Q  : BOOL;    (* TRUE after PT elapsed while IN is continuously TRUE *)
    ET : TIME;    (* Elapsed time — useful for progress displays *)
END_VAR
```

**Motor startup confirmation example:**
```pascal
VAR
    StartupTimer : TON;
    MotorConfirmed : BOOL;
END_VAR

StartupTimer(IN := RunCommand, PT := T#5s);

IF StartupTimer.Q THEN
    (* RunCommand has been TRUE for 5 continuous seconds *)
    MotorConfirmed := TRUE;
ELSIF NOT RunCommand THEN
    MotorConfirmed := FALSE;
    (* TON auto-resets when IN goes FALSE — ET returns to T#0s *)
END_IF;
```

#### TOF — Timer Off-Delay

Output stays TRUE for PT time after input goes FALSE. Useful for minimum run timers and extended alarm delays.

```pascal
FUNCTION_BLOCK TOF
VAR_INPUT
    IN : BOOL;
    PT : TIME;
END_VAR
VAR_OUTPUT
    Q  : BOOL;    (* FALSE after PT elapsed since IN went FALSE *)
    ET : TIME;    (* Time elapsed since IN went FALSE *)
END_VAR
```

**Pump minimum run timer (prevents rapid start/stop):**
```pascal
VAR
    MinRunTimer : TOF;
    PumpStop    : BOOL;
END_VAR

(* Q stays TRUE for 30s after RunPermit goes FALSE *)
MinRunTimer(IN := RunPermit, PT := T#30s);

(* Block stop command until minimum run time met *)
PumpStop := StopCommand AND NOT MinRunTimer.Q;
```

#### TP — Timer Pulse

Generates a fixed-duration pulse on the output regardless of how long the input stays TRUE. The output `Q` is TRUE for exactly `PT` after the rising edge of `IN`. A new pulse is NOT triggered while a pulse is already active.

```pascal
FUNCTION_BLOCK TP
VAR_INPUT
    IN : BOOL;    (* Trigger: rising edge starts the pulse *)
    PT : TIME;    (* Pulse duration — fixed *)
END_VAR
VAR_OUTPUT
    Q  : BOOL;    (* TRUE for exactly PT duration *)
    ET : TIME;    (* Elapsed time within pulse *)
END_VAR
```

**Valve stroke test pulse:**
```pascal
VAR
    StrokePulse : TP;
END_VAR

StrokePulse(IN := TestCmd, PT := T#200ms);
ValveTestOutput := StrokePulse.Q;   (* 200ms output pulse regardless of TestCmd duration *)
```

---

### Counters

#### CTU — Count Up

```pascal
FUNCTION_BLOCK CTU
VAR_INPUT
    CU : BOOL;    (* Count Up: increment on rising edge *)
    R  : BOOL;    (* Reset: sets CV to 0 when TRUE *)
    PV : INT;     (* Preset value *)
END_VAR
VAR_OUTPUT
    Q  : BOOL;    (* TRUE when CV >= PV *)
    CV : INT;     (* Current count value *)
END_VAR
```

#### CTD — Count Down

```pascal
FUNCTION_BLOCK CTD
VAR_INPUT
    CD : BOOL;    (* Count Down: decrement on rising edge *)
    LD : BOOL;    (* Load: sets CV to PV when TRUE *)
    PV : INT;     (* Preset value / starting count *)
END_VAR
VAR_OUTPUT
    Q  : BOOL;    (* TRUE when CV <= 0 *)
    CV : INT;     (* Current count value *)
END_VAR
```

#### CTUD — Count Up/Down

Combines CTU and CTD. Has independent up and down outputs.

```pascal
FUNCTION_BLOCK CTUD
VAR_INPUT
    CU : BOOL;    (* Count Up input *)
    CD : BOOL;    (* Count Down input *)
    R  : BOOL;    (* Reset — sets CV to 0 *)
    LD : BOOL;    (* Load — sets CV to PV *)
    PV : INT;     (* Preset value *)
END_VAR
VAR_OUTPUT
    QU : BOOL;    (* Up output: TRUE when CV >= PV *)
    QD : BOOL;    (* Down output: TRUE when CV <= 0 *)
    CV : INT;     (* Current count *)
END_VAR
```

**Production batch counter example:**
```pascal
VAR
    BatchCounter : CTU;
    PackageCount : CTUD;
END_VAR

(* Count product packages: CU on each package IN, CD on each package OUT *)
PackageCount(
    CU := PackageArrived,
    CD := PackageDispatched,
    R  := ShiftReset,
    PV := 1000   (* Target batch size *)
);

IF PackageCount.QU THEN
    BatchComplete := TRUE;    (* Reached 1000 packages *)
END_IF;
```

---

### Bistable Latches

#### SR — Set Dominant

When both Set (S1) and Reset (R) are simultaneously TRUE, the output stays TRUE (Set wins).

```pascal
FUNCTION_BLOCK SR
VAR_INPUT
    S1  : BOOL;    (* Set input — dominant *)
    R   : BOOL;    (* Reset input *)
END_VAR
VAR_OUTPUT
    Q1  : BOOL;    (* Output latch *)
END_VAR
(* Truth table:
   S1=0, R=0 → Q1 unchanged
   S1=1, R=0 → Q1 = TRUE
   S1=0, R=1 → Q1 = FALSE
   S1=1, R=1 → Q1 = TRUE  (S dominant) *)
```

#### RS — Reset Dominant

When both inputs are simultaneously TRUE, the output goes FALSE (Reset wins). Preferred for safety-related latches.

```pascal
FUNCTION_BLOCK RS
VAR_INPUT
    S   : BOOL;    (* Set input *)
    R1  : BOOL;    (* Reset input — dominant *)
END_VAR
VAR_OUTPUT
    Q1  : BOOL;
END_VAR
(* S=1, R1=1 → Q1 = FALSE (R dominant — safer for trip latches) *)
```

**Safety trip latch (RS preferred over SR for hardwired-equivalent logic):**
```pascal
VAR
    TripLatch : RS;
END_VAR

TripLatch(
    S  := PAHH_101 OR TSHH_201 OR LSLL_301,   (* Any trip condition sets latch *)
    R1 := ResetPB AND NOT AnyTripActive         (* Reset only when all clear *)
);
ShutdownOutput := TripLatch.Q1;
```

---

### Edge Detection

#### R_TRIG — Rising Edge Trigger

`Q` is TRUE for **exactly one scan** when `CLK` transitions from FALSE to TRUE.

```pascal
FUNCTION_BLOCK R_TRIG
VAR_INPUT
    CLK : BOOL;    (* Input signal to monitor *)
END_VAR
VAR_OUTPUT
    Q   : BOOL;    (* TRUE for ONE scan on rising edge *)
END_VAR
```

#### F_TRIG — Falling Edge Trigger

`Q` is TRUE for **exactly one scan** when `CLK` transitions from TRUE to FALSE.

```pascal
FUNCTION_BLOCK F_TRIG
VAR_INPUT
    CLK : BOOL;
END_VAR
VAR_OUTPUT
    Q   : BOOL;    (* TRUE for ONE scan on falling edge *)
END_VAR
```

**Motor start/stop event logging:**
```pascal
VAR
    StartEdge  : R_TRIG;
    StopEdge   : F_TRIG;
    FaultEdge  : R_TRIG;
    StartCount : UDINT;
    FaultCount : UDINT;
END_VAR

StartEdge(CLK := MotorRunning);
StopEdge(CLK  := MotorRunning);
FaultEdge(CLK := FaultSignal);

IF StartEdge.Q THEN
    StartCount := StartCount + 1;
    LastStartTime := SystemTime;
END_IF;

IF StopEdge.Q THEN
    (* Motor just stopped — log event *)
    LogEvent('MOTOR_STOP', SystemTime);
END_IF;

IF FaultEdge.Q THEN
    FaultCount := FaultCount + 1;
    FaultHistory[FaultCount MOD 100] := SystemTime;
END_IF;
```

---

## Standard Functions

### Type conversion (explicit — required in IEC 61131-3)

```pascal
(* Pattern: SRCTYPE_TO_DSTTYPE *)
INT_TO_REAL(intVar)          (* INT → REAL: widening, safe *)
REAL_TO_INT(realVar)         (* REAL → INT: truncates toward zero *)
INT_TO_DINT(intVar)          (* INT → DINT: widening, safe *)
DINT_TO_INT(dintVar)         (* DINT → INT: may overflow *)
BOOL_TO_INT(boolVar)         (* FALSE→0, TRUE→1 *)
INT_TO_BOOL(intVar)          (* 0→FALSE, non-zero→TRUE *)
INT_TO_WORD(intVar)          (* Reinterpret INT bits as WORD *)
REAL_TO_STRING(realVar)      (* REAL → STRING representation *)
TIME_TO_DINT(timeVar)        (* TIME → DINT (milliseconds) *)
DINT_TO_TIME(dintVar)        (* DINT (milliseconds) → TIME *)
```

**All conversions are EXPLICIT in IEC 61131-3.** Unlike C, assigning an INT to a REAL is a **type error** at compile time. This is a safety feature that eliminates a class of subtle bugs.

### Numerical functions

```pascal
ABS(x)          (* Absolute value — same type as input *)
SQRT(x)         (* Square root — REAL *)
LN(x)           (* Natural logarithm *)
LOG(x)          (* Base-10 logarithm *)
EXP(x)          (* e^x *)
SIN(x)          (* Sine (radians) *)
COS(x)          (* Cosine (radians) *)
TAN(x)          (* Tangent (radians) *)
ASIN(x)         (* Arcsine — result in radians *)
ACOS(x)         (* Arccosine *)
ATAN(x)         (* Arctangent — single argument *)
ATAN2(y, x)     (* Two-argument arctangent *)
```

### Selection and comparison

```pascal
MAX(a, b)                    (* Maximum of two values — same type *)
MIN(a, b)                    (* Minimum of two values *)
LIMIT(minVal, val, maxVal)   (* Clamp: minVal <= result <= maxVal *)
SEL(G, IN0, IN1)             (* IF G THEN IN1 ELSE IN0 *)
MUX(K, IN0, IN1, IN2, ...)   (* Multiplexer: select IN[K] *)
```

**Setpoint clamping:**
```pascal
(* Ensure operator setpoint stays within engineering limits *)
SP_Validated := LIMIT(SP_Min, SP_Operator, SP_Max);
```

### String functions

```pascal
LEN(str)                         (* String length in characters *)
LEFT(str, n)                     (* Leftmost n characters *)
RIGHT(str, n)                    (* Rightmost n characters *)
MID(str, n, pos)                 (* n chars starting at position pos *)
CONCAT(str1, str2)               (* Concatenate two strings *)
INSERT(str1, str2, pos)          (* Insert str2 into str1 at pos *)
DELETE(str, n, pos)              (* Delete n chars starting at pos *)
REPLACE(str1, str2, n, pos)      (* Replace n chars at pos with str2 *)
FIND(str1, str2)                 (* Find str2 in str1: returns position, 0 = not found *)
```

---

## POU Instantiation and Calling

### Declaring and using FB instances

```pascal
PROGRAM MainProgram
VAR
    (* Multiple independent instances of the same FB type *)
    P101A    : MotorControl_FB;    (* Feed pump A *)
    P101B    : MotorControl_FB;    (* Feed pump B — standby *)
    P102A    : MotorControl_FB;    (* Discharge pump A *)

    (* Standard library instances *)
    ScanTimer   : TON;
    FaultLatch  : RS;
    ProductEdge : R_TRIG;
END_VAR

(* Call P101A with formal (named) parameters — recommended *)
P101A(
    StartCmd   := Auto_Start_P101A OR (ManStart AND PB_P101A),
    StopCmd    := EmergencyStop OR Auto_Stop_P101A OR PB_P101A_Stop,
    FbkRun     := DI_P101A_Run,
    FbkStp     := DI_P101A_Stp,
    Interlock  := ILK_P101A AND NotMaintenance,
    Protection := OL_P101A AND NOT Trip_P101A,
    ResetCmd   := Reset_Button
);

(* Read FB outputs *)
HMI_P101A_Running := P101A.Running;
HMI_P101A_Fault   := P101A.Faulted;
DO_P101A_Start    := P101A.StartOut;

(* Call P101A without providing all inputs — unspecified inputs retain last value *)
P101B(StartCmd := Auto_P101B, StopCmd := Stop_P101B, FbkRun := DI_P101B_Run);

(* Standard library call *)
ScanTimer(IN := TRUE, PT := T#100ms);

END_PROGRAM
```

### Calling a Function (no instance needed)

```pascal
ScaledFlow := ScaleAnalog(
    RawValue := AI_FT056.RawValue,
    RawMin   := 0,
    RawMax   := 27648,    (* Siemens S7 12-bit ADC full range *)
    EULow    := 0.0,
    EUHigh   := 500.0     (* m³/h *)
);

AlarmActive := AlarmCheck(
    Value      := ScaledFlow,
    HighLimit  := FlowAlarmH,
    LowLimit   := FlowAlarmL,
    Deadband   := 2.0
);
```

---

## Task Configuration

The task configuration in the CONFIGURATION block determines **when** and **how often** each PROGRAM is executed.

```pascal
CONFIGURATION PlantDCS

    (* Three task types: Cyclic, EventDriven, Freewheeling *)

    TASK FastIO(
        INTERVAL := T#10ms,   (* Cyclic: execute every 10ms *)
        PRIORITY := 1         (* Highest priority — preempts lower *)
    );

    TASK MotorControl(
        INTERVAL := T#100ms,  (* Cyclic: execute every 100ms *)
        PRIORITY := 2
    );

    TASK PIDControl(
        INTERVAL := T#200ms,  (* Cyclic: execute every 200ms *)
        PRIORITY := 2
    );

    TASK SlowMonitor(
        INTERVAL := T#1s,     (* Cyclic: execute every 1 second *)
        PRIORITY := 3         (* Lowest priority *)
    );

    TASK SafetyEvent(
        SINGLE   := SafetyTrigger,  (* Event-driven: starts on rising edge of variable *)
        PRIORITY := 1
    );

    (* Assign programs to tasks *)
    PROGRAM DigIOScan     WITH FastIO       : DigitalIO_Program;
    PROGRAM MotorCtrl     WITH MotorControl : MotorControl_Program;
    PROGRAM PIDCtrl       WITH PIDControl   : PIDControl_Program;
    PROGRAM AlarmMonitor  WITH SlowMonitor  : AlarmScan_Program;
    PROGRAM SafetyAction  WITH SafetyEvent  : Safety_Program;

END_CONFIGURATION
```

### Task types

| Task Type | Trigger | Priority Rules | Typical Use |
|-----------|---------|---------------|-------------|
| **Cyclic** | Fixed time interval (`INTERVAL`) | Higher priority preempts lower when intervals overlap | PID loops, motor control, alarm scanning |
| **Event-driven** | Rising edge of a BOOL variable (`SINGLE`) | Preempts cyclic tasks of lower priority | Safety actions, emergency shutdown |
| **Freewheeling** | Restarts immediately after completion | Lowest priority — runs only when no other task is ready | Initialization, background calculation |

### Typical DCS task periods

| Function | Period | Priority | Rationale |
|---------|--------|----------|-----------|
| Digital I/O, safety | 10–20 ms | 1 | Fast response to digital inputs |
| Motor/valve control | 100–200 ms | 2 | Motor start time >> 200ms |
| PID control loops | 100–500 ms | 2 | Process time constants >> 500ms |
| Alarm monitoring | 500 ms–2 s | 3 | Alarm response time requirement |
| Sequence/batch logic | 100 ms–1 s | 2 | Phase step transitions |
| Historian/logging | 1–5 s | 4 | Data recording, not real-time |

---

## Data Type Conversions

IEC 61131-3 never performs implicit type conversion. All conversions must be explicit using typed conversion functions.

```pascal
VAR
    I16  : INT   := 1000;
    I32  : DINT;
    F32  : REAL;
    Bool : BOOL;
    W    : WORD;
END_VAR

(* VALID explicit conversions *)
I32  := INT_TO_DINT(I16);      (* Widening: always safe *)
I16  := DINT_TO_INT(I32);      (* Narrowing: may overflow silently *)
F32  := INT_TO_REAL(I16);      (* Integer → float: exact for INT range *)
I16  := REAL_TO_INT(F32);      (* Float → int: truncates toward zero *)
W    := INT_TO_WORD(I16);      (* Reinterpret bits: same 16 bits *)
Bool := INT_TO_BOOL(I16);      (* 0 = FALSE, any non-zero = TRUE *)

(* TYPE ERRORS — these will NOT compile: *)
(* I32 := I16;      implicit widening — not allowed *)
(* F32 := I16;      implicit int-to-float — not allowed *)
(* I16 := F32;      implicit float-to-int — not allowed *)
```

---

## Corner Cases

### RETAIN vs PERSISTENT

Both `RETAIN` and `PERSISTENT` preserve variable values across power failures, but they behave differently in CODESYS-based systems and differ across vendors:

| Feature | `RETAIN` | `PERSISTENT` |
|---------|---------|-------------|
| Preserved across power-off | Yes | Yes |
| Preserved across **program download** | No — cleared on download | Yes — survives download |
| Scope | Individual variables or FB instances | Individual variables |
| Typical use | Counters, batch totals that must survive power loss | Configuration values that must survive firmware updates |

```pascal
(* RETAIN: survives power-off but reset on download *)
VAR RETAIN
    BatchTotalKg   : REAL;      (* Accumulated batch weight — survives power loss *)
    TripCounter    : UDINT;     (* Motor trips since last maintenance *)
END_VAR

(* PERSISTENT: survives both power-off and download *)
VAR PERSISTENT
    CalibrationFactor : REAL := 1.0;    (* User-set calibration — must persist download *)
    ProductionDate    : DATE;
END_VAR
```

**Important CODESYS caveat:** Declaring an entire FB instance as RETAIN stores ALL variables of that FB in retain memory — including VAR_TEMP and other variables that do not need persistence. The recommended pattern is:

```pascal
(* Better: RETAIN only the data structure, pass by VAR_IN_OUT *)
VAR RETAIN
    P101A_State : MotorData;    (* Only the MotorData struct persists *)
END_VAR

P101A_FB(
    StartCmd    := Start_P101A,
    PersistedData => P101A_State    (* VAR_IN_OUT: FB updates this struct *)
);
```

### FUNCTION_BLOCK reentrancy and multiple instances

Each FB instance has its own independent copy of all internal (`VAR`) variables. This is the key architectural advantage of FBs for industrial automation — the same FB type can control 200 motors simultaneously with completely independent state:

```pascal
PROGRAM Main
VAR
    (* 200 independent motor instances — same logic, independent state *)
    Motors : ARRAY[1..200] OF MotorControl_FB;
    i      : INT;
END_VAR

(* Call all 200 motor instances in a loop *)
FOR i := 1 TO 200 DO
    Motors[i](
        StartCmd  := MotorStartCmd[i],
        StopCmd   := MotorStopCmd[i],
        FbkRun    := MotorRunFbk[i],
        Interlock := MotorIntlk[i]
    );
    MotorRunStatus[i] := Motors[i].Running;
    MotorFaultStatus[i] := Motors[i].Faulted;
END_FOR;
```

`Motors[1].State` and `Motors[200].State` are stored at completely different memory addresses. Calling `Motors[1](...)` does not affect `Motors[200]`. This is called **instance-based reentrancy** — multiple instances of the same type can run in different task contexts (e.g., different interrupt priorities) provided they do not share `VAR_GLOBAL` or `VAR_EXTERNAL` without proper synchronization.

**Contrast with FUNCTION:** A FUNCTION has no instance variables, so it is inherently reentrant even if called simultaneously from multiple tasks. A FB is also safe when instances are separate (no shared state), but if two tasks both access the SAME instance of a FB, mutual exclusion is needed.

### STRING fixed-length behavior

IEC 61131-3 STRING is always **fixed maximum allocation**. Assigning a longer string truncates without error:

```pascal
VAR
    TagName : STRING[8];
END_VAR

TagName := 'FIC-101A';    (* 8 chars — OK, fits exactly *)
TagName := 'FIC-101AXYZ'; (* 11 chars — silently truncated to 'FIC-101A' *)

(* Check before assignment to detect truncation: *)
IF LEN('FIC-101AXYZ') <= 8 THEN
    TagName := 'FIC-101AXYZ';
ELSE
    LogError('Tag name too long');
END_IF;
```

### ARRAY with negative indices — valid IEC 61131-3

Arrays may have any integer bounds including negative values:

```pascal
VAR
    (* Sample buffer centered on index 0: -50 to +50 ms around event *)
    EventBuffer : ARRAY[-50..50] OF REAL;   (* 101 elements *)

    (* Lookup table with negative keys *)
    OffsetTable : ARRAY[-3..-1] OF LREAL;   (* Elements at -3, -2, -1 *)
END_VAR

EventBuffer[-50] := 0.0;    (* First element *)
EventBuffer[0]   := 1.0;    (* Center element *)
EventBuffer[50]  := 0.0;    (* Last element *)
```

---

## Vendor-Specific Extensions

Most DCS/PLC vendors extend IEC 61131-3 with proprietary features. Common important extensions:

| Extension | Purpose | Vendor(s) |
|-----------|---------|-----------|
| `AT %I0.0` directive | I/O address binding | All (standard) |
| `RETAIN` / `PERSISTENT` qualifiers | Non-volatile variable memory | All (behavior varies) |
| OOP: `EXTENDS`, `IMPLEMENTS` | Inheritance and interfaces | CODESYS 3.x, TIA V16+, B&R |
| `__SYSTEM_*` functions | System-level access | CODESYS |
| `SIZEOF()`, `ADDROF()` | Memory operations | CODESYS, Beckhoff |
| `PERSISTENT` | Survives code download | CODESYS, B&R |
| `SFC_RESET` | SFC step reset | Siemens |
| Methods on FUNCTION_BLOCK | Object methods | CODESYS 3.x, TIA V18+ |
| `INTERFACE` | Interface declarations | CODESYS 3.x |
| Bit access via `.0`, `.7` | Direct bit extraction | Most vendors |

### AT directive (I/O address binding)

```pascal
VAR
    RunFeedback   AT %IX0.0 : BOOL;    (* Digital input: byte 0, bit 0 *)
    FaultInput    AT %IX0.1 : BOOL;    (* Digital input: byte 0, bit 1 *)
    StartOutput   AT %QX4.0 : BOOL;   (* Digital output: byte 4, bit 0 *)
    SpeedRefWord  AT %QW100 : WORD;   (* Analog output word at address 100 *)
    ProcessValue  AT %IW200 : INT;    (* Analog input at address 200 *)
END_VAR
```

Address format: `%[I/Q/M][X/B/W/D][byte].[bit]`
- `I` = Input, `Q` = Output, `M` = Memory (marker)
- `X` = Bit, `B` = Byte, `W` = Word (16-bit), `D` = DWord (32-bit)
