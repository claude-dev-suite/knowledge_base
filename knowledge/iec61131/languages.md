# IEC 61131-3 - Programming Languages

> Official Reference: https://plcopen.org/technical-activities/tc1-software-model
> Additional Reference: https://www.codesys.com/the-system/programming-language.html
> OOP Extensions: https://stefanhenneken.net/2017/04/23/iec-61131-3-methods-properties-and-inheritance/
> Ladder/ST Reference: https://www.plcacademy.com/structured-text-tutorial/

## Overview

IEC 61131-3 is the international standard (third edition, 2013) defining the syntax and semantics of programming languages for programmable controllers. It defines five programming languages and the Program Organization Unit (POU) model. The third edition added object-oriented extensions (methods, properties, interfaces, inheritance) that are now widely supported in modern tools (CODESYS, Beckhoff TwinCAT, B&R Automation Studio, Siemens TIA Portal v16+).

Instruction List (IL) was deprecated in the third edition (2013). Active languages are:
- **LD** — Ladder Diagram (graphical, relay logic origin)
- **FBD** — Function Block Diagram (graphical, data-flow)
- **ST** — Structured Text (textual, Pascal-like)
- **SFC** — Sequential Function Chart (graphical/hybrid, Grafcet origin)

---

## Table of Contents

1. [Ladder Diagram (LD)](#1-ladder-diagram-ld)
2. [Function Block Diagram (FBD)](#2-function-block-diagram-fbd)
3. [Structured Text (ST)](#3-structured-text-st)
4. [Sequential Function Chart (SFC)](#4-sequential-function-chart-sfc)
5. [Language Selection Guidelines](#5-language-selection-guidelines)
6. [OOP Extensions (Third Edition)](#6-oop-extensions-third-edition)
7. [PLCopen XML Language Representation](#7-plcopen-xml-language-representation)
8. [Corner Cases and Non-Obvious Behaviors](#8-corner-cases-and-non-obvious-behaviors)

---

## 1. Ladder Diagram (LD)

### Concept

LD is derived from relay logic diagrams. A PLC program is organized into **rungs** between two vertical power rails. Data flows conceptually from left (power) to right (coil/output). Each rung evaluates a Boolean expression and drives one or more output coils.

### Core Elements

**Contacts (Input Instructions):**

| Symbol | Name | Behavior |
|--------|------|----------|
| `--[ ]--` | Normally Open (NO) / XIC | TRUE when addressed bit = 1 |
| `--[/]--` | Normally Closed (NC) / XIO | TRUE when addressed bit = 0 |
| `--[P]--` | Positive Transition | TRUE for one scan on 0→1 edge |
| `--[N]--` | Negative Transition | TRUE for one scan on 1→0 edge |

**Coils (Output Instructions):**

| Symbol | Name | Behavior |
|--------|------|----------|
| `--( )--` | Output Coil | Sets bit = RLO (Result of Logic Operation) |
| `--(S)--` | Set (Latch) | Sets bit to 1; remains 1 until reset |
| `--(R)--` | Reset (Unlatch) | Sets bit to 0; remains 0 until set |

**Function Blocks in LD:**
Standard function blocks (timers, counters, comparators, arithmetic) appear as rectangular boxes inserted into rungs. The EN (Enable) pin connects to the rung's logical result; ENO (Enable Output) passes power to subsequent elements.

### Execution Model

1. PLC reads all physical inputs into an **input image table** at scan start.
2. Ladder program executes rung by rung, top to bottom; within a rung, left to right.
3. Results written to **output image table**.
4. Output image table written to physical outputs at scan end.

**Important**: In a single scan, if rung 5 sets a bit that rung 10 reads, rung 10 sees the **new value** (within-scan data dependency). This is standard "immediate update" behavior. The entire input image is captured once at the start of the scan; physical inputs do not change mid-scan.

### Example: Motor Start/Stop with Seal-In

```
|--[START_PB]--[/STOP_PB]--[/E_STOP]--[/FAULT]--+--[/MOTOR_ON]--+--(MOTOR_CMD)--|
|                                                  |               |              |
|                                                  +--[MOTOR_ON]---+              |
```

This implements a latching start circuit: `START_PB` momentarily energizes `MOTOR_CMD`, the seal-in contact `MOTOR_ON` (fed from the running feedback coil) holds it, `STOP_PB`, `E_STOP`, or `FAULT` break the circuit.

### Parallel Branches (OR Logic)

```
|--[A]--+--------+--(OUT)--|
|       |--[B]---|         |
|       |--[C]---|         |
```
`OUT = A OR B OR C`

### Corner Cases in LD

- **Double coil problem**: Addressing the same output coil variable in two different rungs means the last rung (lower in the program) overwrites previous rungs. This is a common bug source. Most tools warn but do not prevent it.
- **NC contacts on safety stops**: A physical E-Stop is wired NC (normally closed) — the PLC input reads 1 during normal operation and 0 when the E-Stop is activated. Using XIC (NO contact) in the program for this input means the program logic becomes FALSE when the E-Stop trips. This is the correct fail-safe behavior.
- **RETAIN and coils**: Coils drive physical outputs every scan regardless of RETAIN. The RETAIN attribute on the underlying variable matters only for power-loss recovery of the variable's last state, but the physical output is always re-driven.
- **Memory latching vs SR latch**: A self-holding circuit in LD (coil with seal-in contact) behaves like an RS latch. If power is lost and restored, the coil state depends on whether the variable is RETAIN or not — if not RETAIN, it resets to 0 on power-up regardless of previous state.

---

## 2. Function Block Diagram (FBD)

### Concept

FBD represents logic as a network of interconnected function blocks and functions. Data flows from left to right through connections between block outputs and inputs. It is closely related to electronic circuit diagrams and is the dominant language in DCS environments (Siemens CFC, ABB Freelance FBD sheets, DeltaV control modules).

### Structure

- **Function Block instances**: Rectangular boxes with input pins (left side) and output pins (right side)
- **Functions**: Boxes without persistent state; inputs go in, output comes out in same scan
- **Connections**: Lines (wires) from output pins to input pins
- **Negation**: Small circle on a Boolean pin inverts the signal — equivalent to inserting a NOT gate
- **Boolean variable references**: Can appear as LD-style contacts in some FBD editors

### Execution Order

In FBD, execution order is determined by **data dependency**. Blocks whose outputs feed other blocks' inputs must execute first. When no dependency exists between two branches, the editor or compiler determines order (typically top-to-bottom, left-to-right as drawn). Most tools allow the engineer to manually assign execution order numbers to each block.

**Key rule**: A block cannot execute until all its input connections have resolved values for the current scan.

### EN/ENO Mechanism

EN (Enable) and ENO (Enable Output) are optional Boolean pins available on every standard function block:
- When `EN = FALSE`: block does **not** execute; outputs retain their previous values (or defaults); `ENO` is set to FALSE.
- When `EN = TRUE`: block executes normally. `ENO = TRUE` if execution completes without error; `ENO = FALSE` if a runtime error occurs (e.g., divide-by-zero).
- Chaining `ENO → EN` creates conditional execution chains where an error in one block disables all downstream blocks.

```
[StartCmd BOOL] --> EN  [TON]  ENO --> EN [PulseOut]
                    IN         Q --> ...
[T#5S TIME]     --> PT
```

### Negation Corner Case

A small circle placed on the **output** pin of block A feeding block B is identical to placing it on the corresponding **input** pin of block B. They represent the same single inversion. If drawn on both simultaneously, the inversions cancel (double negation = no inversion). This is a common mistake in FBD editors.

### Feedback Connections

In FBD, connecting an output back to a prior input creates a feedback loop. The standard requires that at least one element in the loop holds the previous scan's value (a memory variable). Most compilers reject pure algebraic loops (two blocks mutually dependent with no state element).

### Unconnected Inputs

Inputs left unconnected receive their data type's default value:
- BOOL: FALSE
- INT/REAL: 0
- STRING: empty string ''
- TIME: T#0s

This is a silent default — most tools do not generate an error for unconnected non-mandatory inputs. A required parameter left unconnected causes subtle incorrect behavior.

### Standard Function Blocks Available in FBD

Per IEC 61131-3:

| Block | Function |
|-------|----------|
| SR | Set-dominant bistable latch |
| RS | Reset-dominant bistable latch |
| R_TRIG | Rising edge detector (1 scan pulse on 0→1) |
| F_TRIG | Falling edge detector (1 scan pulse on 1→0) |
| TON | On-delay timer |
| TOF | Off-delay timer |
| TP | Pulse timer |
| CTU | Up-counter |
| CTD | Down-counter |
| CTUD | Up/down counter |
| MUX | N-to-1 multiplexer |
| SEL | Binary selector (G selects IN0 or IN1) |
| LIMIT | Clamp value between min and max |
| MIN / MAX | Minimum or maximum of two values |
| MOVE | Passes IN to OUT (explicit assignment) |

---

## 3. Structured Text (ST)

### Concept

ST is a high-level, Pascal-like textual language. It is the most expressive IEC 61131-3 language and the preferred choice for complex algorithms, mathematical calculations, string processing, and any logic that would be verbose in graphical languages. Case-insensitive. Statements terminated by semicolons.

### Comment Syntax

```pascal
// Single-line comment (universally supported, non-standard in strict IEC but used everywhere)
(* Multi-line
   comment *)
```

### Operators (Complete Table by Precedence — Highest to Lowest)

| Precedence | Operator | Description | Example |
|-----------|----------|-------------|---------|
| 1 (highest) | `()` | Parentheses | `(A + B) * C` |
| 2 | function call | Function/method invocation | `SIN(x)`, `ABS(-5)` |
| 3 | `-` (unary) | Arithmetic negation | `-x` |
| 3 | `NOT` | Boolean complement / bitwise NOT | `NOT bFlag` |
| 4 | `**` | Exponentiation | `x ** 2.0` |
| 5 | `*` | Multiply | `a * b` |
| 5 | `/` | Divide | `a / b` |
| 5 | `MOD` | Modulo (remainder) | `10 MOD 3` — result 1 |
| 6 | `+` | Addition | `a + b` |
| 6 | `-` | Subtraction | `a - b` |
| 7 | `<` `>` `<=` `>=` | Comparison | `a < b` (returns BOOL) |
| 8 | `=` | Equality test | `a = b` (returns BOOL) |
| 8 | `<>` | Inequality test | `a <> b` |
| 9 | `&` `AND` | Logical AND / bitwise AND | `a AND b` |
| 10 | `XOR` | Logical XOR / bitwise XOR | `a XOR b` |
| 11 (lowest) | `OR` | Logical OR / bitwise OR | `a OR b` |

**Note**: `AND`, `OR`, `XOR`, `NOT` work on both BOOL and bit-string types (BYTE, WORD, DWORD, LWORD). On bit-strings, they perform bitwise operations. On BOOL, they perform Boolean logic. The operator is the same; the semantics depend on the operand type.

### Assignment

```pascal
variable := expression;    (* Assign: := not = *)
```

The `:=` operator assigns the right-hand expression to the left-hand variable. Plain `=` is comparison only (not assignment — unlike C or Python).

### Control Structures

**IF/ELSIF/ELSE:**
```pascal
IF condition1 THEN
    statement1;
ELSIF condition2 THEN
    statement2;
ELSE
    statement3;
END_IF;
```
Any number of ELSIF branches. ELSE is optional. Conditions must be type BOOL.

**CASE (switch on integer or enum):**
```pascal
CASE selector OF
    1:        statementA;
    2, 3:     statementB;           (* multiple values per branch *)
    10..20:   statementC;           (* range syntax: 10 to 20 inclusive *)
    STATE_RUNNING: statementD;      (* enum value *)
ELSE
    statementDefault;
END_CASE;
```
Selector must be an integer type or enumeration. Range syntax `lo..hi` is widely supported (CODESYS, TIA Portal, TwinCAT). ELSE executes if no case matches.

**FOR loop:**
```pascal
FOR i := 1 TO 100 BY 2 DO
    arr[i] := arr[i] * 2;
END_FOR;
```
`BY` clause is optional (defaults to 1). The loop variable `i` is a local counter; modifying it inside the loop affects execution. `EXIT` immediately exits the loop. `CONTINUE` (vendor extension) skips to next iteration.

**WHILE loop:**
```pascal
WHILE condition DO
    statements;
END_WHILE;
```
Executes zero or more times. An infinite loop will trigger a **watchdog timeout** on most PLCs (typically 100-500ms default), causing a controller fault. Never use WHILE with conditions that depend on external hardware state without a timeout guard.

**REPEAT loop:**
```pascal
REPEAT
    statements;
UNTIL condition
END_REPEAT;
```
Executes at least once (body runs before condition is evaluated). Continues until condition is TRUE.

**RETURN:**
```pascal
RETURN;
```
Immediately exits the current FUNCTION or method. Not applicable to PROGRAM bodies.

### Function Block Instantiation and Calling

```pascal
VAR
    myTimer : TON;
    myEdge  : R_TRIG;
    myMotor : MotorControl_FB;
END_VAR

(* Named parameter style — recommended *)
myTimer(IN := bStartSignal, PT := T#5S);
IF myTimer.Q THEN
    bOutputActive := TRUE;
END_IF;

(* Edge detector *)
myEdge(CLK := bInputSignal);
IF myEdge.Q THEN
    (* Rising edge detected — executes for exactly 1 scan *)
    EventCounter := EventCounter + 1;
END_IF;

(* Complex motor FB with named parameters *)
myMotor(
    StartCmd   := bAutoStart OR bManStart,
    StopCmd    := bStop OR bEStop,
    FbkRun     := DI_Motor_Run,
    Interlock  := Permissive_OK,
    MonTime    := T#10S
);
bMotorRunning := myMotor.Running;
bMotorFault   := myMotor.Faulted;
```

**Critical rule**: A function block instance must be **called every scan** (or at minimum, every task cycle) for timers and edge detectors to function correctly. If an FB instance is conditionally skipped for an entire scan (e.g., inside an IF that is FALSE), its internal state — including timer elapsed time — does not update.

### String Operations in ST

```pascal
VAR
    s1 : STRING(80) := 'FIC-101A';
    s2 : STRING(80);
    n  : INT;
END_VAR

s2 := CONCAT(s1, '_ALARM');     (* 'FIC-101A_ALARM' *)
n  := LEN(s1);                   (* 8 *)
s2 := LEFT(s1, 3);               (* 'FIC' *)
s2 := RIGHT(s1, 4);              (* '101A' *)
s2 := MID(s1, 3, 5);             (* '101' — MID(str, len, start_pos) *)
n  := FIND(s1, '-');             (* 4 — 1-based index, 0 = not found *)
```

**STRING corner case**: STRINGs in IEC 61131-3 have a fixed maximum length declared as `STRING(n)`. Assigning a string longer than `n` characters causes **silent truncation** — no overflow error, no warning at runtime. Always validate string lengths before assignment in safety-critical code.

### TIME Literals and Arithmetic

```pascal
tDelay  := T#5S;               (* 5 seconds *)
tDelay  := T#1M30S;            (* 90 seconds *)
tDelay  := T#100MS;            (* 100 milliseconds *)
tDelay  := T#1H2M3S4MS;        (* 1 hour 2 min 3 sec 4 ms *)

(* TIME arithmetic *)
tResult := T#10S - T#3S;       (* T#7S *)
tResult := T#5S + T#2S500MS;   (* T#7S500MS *)

(* DATE, TOD, DT literals *)
dDate   := D#2024-03-14;
todTime := TOD#14:30:00;
dtFull  := DT#2024-03-14-14:30:00.500;    (* with milliseconds *)
```

---

## 4. Sequential Function Chart (SFC)

### Concept

SFC is a graphical language for defining sequential and parallel step-based behavior. It is directly derived from Grafcet (IEC 60848) and is the natural fit for batch sequences and machine cycles that map to ISA-88 phases. An SFC consists of **steps**, **transitions**, and **action blocks** connected in a directed graph.

### Elements

**Step**: A named state in the sequence. Has associated action blocks. A step is either **Active** or **Inactive**.
- **Initial step** (double-bordered box): The step active when the POU starts for the first time.
- A step remains active until its outgoing transition condition evaluates to TRUE.

**Transition**: A Boolean condition (can be written in ST, LD, or FBD inline) that guards the arc between steps. When the preceding step is Active and the transition is TRUE in the same scan, the step deactivates and the successor step activates.

**Action Block**: Code attached to a step, qualified by an action qualifier:

| Qualifier | Name | Behavior |
|-----------|------|----------|
| `N` | Non-stored | Executes every scan while step is active |
| `S` | Set (stored) | Sets a Boolean flag TRUE on step activation; remains TRUE until explicitly reset |
| `R` | Reset | Resets a Boolean flag FALSE on step activation |
| `P` | Pulse | Executes for exactly one scan when step becomes active (rising edge) |
| `P0` | Pulse (falling) | Executes for exactly one scan when step becomes inactive |
| `L` | Limited time | Executes for a specified time after step activation, then stops |
| `D` | Delayed | Begins executing after a delay from step activation |
| `SD` | Stored + Delayed | Stored; execution begins after delay |
| `DS` | Delayed + Stored | Delayed start; once started, stored (continues after step deactivation) |

### AND (Parallel) Divergence and Convergence

Multiple branches execute simultaneously — like spawning parallel threads:
```
           [S0_IDLE]
                |
               <T0>
          /----+----\
    [S1_PUMP]   [S2_AGIT]     <- both activate simultaneously (AND-divergence)
          |           |
    [S1_DONE]   [S2_DONE]
          |           |
          \----+----/
              <T_ALL>          <- ALL branches must reach here (AND-convergence)
          [S3_FINISH]
```
The AND-convergence transition `T_ALL` can only fire when **all** parallel branches have reached their convergence step. If one branch is blocked, the other waits.

### OR (Selective) Divergence and Convergence

Only one branch is taken based on which transition fires first:
```
          [S0_IDLE]
               |
        /<T_PATH_A>--<T_PATH_B>\
   [S1_PATH_A]    [S1_PATH_B]     <- OR-divergence
        |               |
        \------+--------/
          [S2_DONE]               <- OR-convergence (any branch can reach it)
```
When multiple OR-divergence transitions are simultaneously TRUE, most tools take the **leftmost** branch (leftmost = highest priority). The IEC standard states priority is implementation-defined.

### Step Timing and Variables

```pascal
(* Accessing step properties in ST code *)
IF StepName.x THEN
    (* Step is currently active *)
END_IF;

IF StepName.t >= T#30S THEN
    (* Step has been active for 30 seconds — implement timeout *)
    bStepTimeout := TRUE;
    bTransition_ForceNext := TRUE;
END_IF;
```

The `.x` attribute (BOOL) is TRUE while the step is active. The `.t` attribute (TIME) is the elapsed time since the step became active. These can be referenced in transition conditions or action code.

### SFC System Functions

```pascal
SFCReset();     (* Reset SFC to initial step, clear all active steps *)
SFCInit();      (* Alias for SFCReset in some vendors *)
```

### Corner Cases in SFC

- **Token model**: SFC uses a conceptual "token" — exactly one token per sequential branch. In AND-divergence, one token splits into N. Loss of a token (programming error such as a transition that is never TRUE) causes the sequence to halt silently with no error message.
- **Simultaneous transitions in OR-divergence**: If two OR transitions are simultaneously TRUE in the same scan, behavior is implementation-defined. Leftmost wins in CODESYS; others may be vendor-specific. Always design OR divergences so at most one transition is TRUE at any time.
- **Action qualifier L and D timers**: These use internal timers. If the SFC POU is not called for a scan (e.g., task overrun, or POU execution is conditional), the timers do not advance.
- **P qualifier single-scan guarantee**: The P qualifier fires for exactly one scan when the step becomes active. If the step becomes active and the same code using the step's action is needed again, the step must be deactivated and reactivated.
- **SFC and watchdog**: A step that runs forever (no transition ever becomes TRUE) will not trigger a watchdog — the SFC POU continues to be called, it just stays in the same step. The watchdog only trips if the scan cycle itself exceeds the watchdog time.

---

## 5. Language Selection Guidelines

| Situation | Recommended Language |
|-----------|---------------------|
| Relay replacement, simple interlock logic | LD |
| Signal processing, data-flow pipelines | FBD |
| Control loops (PID wiring) | FBD |
| Complex algorithms, math, string processing | ST |
| Sequential batch/machine sequences | SFC |
| State machines | SFC or ST (CASE) |
| Safety-certified code (IEC 62061, SIL) | LD or ST (tool-certified) |
| Code generation / bulk engineering | ST (easiest to template as text) |
| Mixed sequential + regulatory control | SFC calling ST/FBD sub-programs |
| Large I/O marshalling | LD (visual review) |
| Team with electrical background | LD preferred |
| Team with software background | ST preferred |

**DCS-specific guidance**: In DCS environments (ABB Freelance, Siemens PCS7, Emerson DeltaV), the primary programming language is FBD — wiring together vendor-supplied library blocks (motor, valve, PID). ST is used for custom calculations and interlocks. SFC is used for batch sequences (ISA-88 phases and unit procedures).

Real programs mix languages: SFC for the top-level sequence, with individual step actions calling FBs written in ST or FBD, with interlock conditions expressed in LD for electrical engineers to review.

---

## 6. OOP Extensions (Third Edition)

IEC 61131-3 third edition (2013) added object-oriented features to FUNCTION_BLOCK POUs. These are supported in CODESYS 3.5+, Beckhoff TwinCAT 3+, B&R Automation Studio 4+, Siemens TIA Portal v16+. ABB Freelance does NOT support OOP extensions.

### Methods

Methods are sub-programs belonging to a FUNCTION_BLOCK. They have full access to the FB's instance variables (VAR section).

```pascal
FUNCTION_BLOCK FB_Drive
VAR
    nCurrentSpeed : DINT := 0;
    bRunning      : BOOL := FALSE;
END_VAR

METHOD PUBLIC Start : BOOL        (* return type after colon *)
VAR_INPUT
    nTargetSpeed : DINT;
    nRampTime    : TIME;
END_VAR
    nCurrentSpeed := nTargetSpeed;
    bRunning := TRUE;
    Start := TRUE;                 (* assign return value via method name *)
END_METHOD

METHOD PUBLIC Stop : BOOL
    bRunning := FALSE;
    nCurrentSpeed := 0;
    Stop := TRUE;
END_METHOD
```

**Access specifiers:**
- `PUBLIC` — accessible by any caller (default)
- `PRIVATE` — accessible only within this FB
- `PROTECTED` — accessible within this FB and derived FBs
- `INTERNAL` — accessible within the same namespace/library only
- `FINAL` — cannot be overridden in derived FBs; enables compiler optimization (~8-10% faster dispatch in tight loops, per testing)

**Calling a method:**
```pascal
fbDrive.Start(nTargetSpeed := 1500, nRampTime := T#5S);
fbDrive.Stop();
```

### Properties

Properties provide controlled access to FB variables with validation logic:

```pascal
FUNCTION_BLOCK FB_Tank
VAR
    rLevel_Internal : REAL;
END_VAR

PROPERTY PUBLIC Level : REAL
    GET
        Level := rLevel_Internal;
    END_GET
    SET
        (* Validate range before accepting *)
        IF Level >= 0.0 AND Level <= 100.0 THEN
            rLevel_Internal := Level;
        END_IF;
    END_SET
END_PROPERTY
```

A property can be read-only (GET only) or write-only (SET only). Access specifiers can differ between GET and SET (e.g., PUBLIC GET, PROTECTED SET).

### Interfaces

```pascal
INTERFACE I_Motor
    METHOD Start  : BOOL
    METHOD Stop   : BOOL
    PROPERTY IsRunning : BOOL
END_INTERFACE

FUNCTION_BLOCK FB_ACDrive IMPLEMENTS I_Motor
    (* Must provide Start, Stop, and IsRunning implementations *)
END_FUNCTION_BLOCK
```

### Inheritance

```pascal
FUNCTION_BLOCK FB_MotorBase
    (* base implementation *)
END_FUNCTION_BLOCK

FUNCTION_BLOCK FB_VFDMotor EXTENDS FB_MotorBase
    (* Inherits all PUBLIC and PROTECTED members of FB_MotorBase *)
    (* Can override methods *)
END_FUNCTION_BLOCK
```

**SUPER pointer** — calls the parent's implementation:
```pascal
METHOD PUBLIC Start : BOOL
    SUPER^.Start();       (* call parent Start first *)
    (* additional VFD-specific ramp logic *)
END_METHOD
```

**THIS pointer** — references the current FB instance. Useful when a local variable shadows an FB-level variable:
```pascal
THIS^.nSpeed := nSpeed;   (* THIS^.nSpeed = FB instance variable; nSpeed = local/input *)
```

### Polymorphism

```pascal
VAR
    refMotor : REFERENCE TO FB_MotorBase;
    fbAC     : FB_ACDrive;
    fbDC     : FB_DCDrive;
END_VAR

IF bUseAC THEN
    refMotor REF= fbAC;     (* REF= assigns a reference *)
ELSE
    refMotor REF= fbDC;
END_IF;

refMotor.Start(1500, T#5S);   (* Calls the correct override at runtime *)
```

`REFERENCE TO` is a non-nullable pointer-like construct. A reference must be assigned before it can be dereferenced. Dereferencing an uninitialized `REFERENCE TO` is undefined behavior (usually a fatal runtime error).

---

## 7. PLCopen XML Language Representation

PLCopen XML TC6 (v2.01) serializes each language as follows:

- **ST**: Code embedded as text inside `<ST><xhtml:body><xhtml:p>` — must be XML-escaped (`<` → `&lt;`, `&` → `&amp;`)
- **LD**: `<LD>` with `<rung>`, `<leftPowerRail>`, `<contact>`, `<coil>`, `<rightPowerRail>` elements at explicit x/y coordinates
- **FBD**: `<FBD>` with `<block>` elements (each with `typeName`, `instanceName`, `<inputVariables>`, `<outputVariables>`), connected by `<connection refLocalId="..."/>` references
- **SFC**: `<SFC>` with `<step>`, `<transition>`, `<selectionDivergence>`, `<selectionConvergence>`, `<simultaneousDivergence>`, `<simultaneousConvergence>` elements

**Vendor support:**
- CODESYS 3.5+: Full TC6 import/export (reference implementation)
- Beckhoff TwinCAT 3: Full TC6 (CODESYS-based)
- Schneider EcoStruxure: Partial TC6
- Siemens TIA Portal: **NOT** supported natively; uses SimaticML
- ABB Freelance: **NOT** supported; uses PRT/CSV format
- Emerson DeltaV: **NOT** supported; uses FHX format

---

## 8. Corner Cases and Non-Obvious Behaviors

### Scan Cycle and Timer Accuracy

PLC timers (TON, TOF, TP) measure **elapsed scan cycles**, not real-time wall-clock time. If the scan cycle is 100 ms and `PT = T#100MS`, the timer may fire anywhere from 100 ms to 200 ms after activation, depending on when within the scan it was evaluated. For high-precision timing, use hardware timers or dedicated high-resolution timer blocks.

### Watchdog

A WHILE or FOR loop that runs too long in a single scan will trigger the **watchdog timer** (typically 100-500 ms configurable), causing the controller to fault. This is a safety mechanism. Never use WHILE loops that wait for hardware state changes — use the scan cycle itself as the "loop".

### RETAIN vs PERSISTENT

- `RETAIN`: Variable retains its value after a **warm restart** (power cycle while program is in memory). Lost on a cold restart (complete PLC reset, download).
- `PERSISTENT`: Variable retains its value after both warm AND cold restarts (stored in non-volatile flash/disk). Slower to write.
- `RETAIN PERSISTENT` (CODESYS-specific): Both attributes combined.

### VAR_IN_OUT Semantics

`VAR_IN_OUT` passes by **reference** (pointer), not by value. Modifications inside the FB affect the caller's variable directly. This differs from `VAR_OUTPUT` which only returns a value.

```pascal
FUNCTION_BLOCK FB_Clamp
VAR_IN_OUT
    rValue : REAL;   (* caller's variable is modified in place *)
END_VAR
    rValue := LIMIT(0.0, rValue, 100.0);
END_FUNCTION_BLOCK
```

### Reentrancy

FUNCTION_BLOCK instances are **not reentrant**. Each instance has its own state. You cannot safely call the same FB instance from two different tasks running simultaneously without synchronization. FUNCTIONs (stateless) are inherently reentrant.

### FUNCTION Return Value

In a FUNCTION, the return value is assigned by using the function's **own name** as a variable in the body. Failing to assign it causes the function to return the default value (0, FALSE, etc.) silently:

```pascal
FUNCTION CalculatePower : REAL
VAR_INPUT kW, efficiency : REAL; END_VAR
    CalculatePower := kW / efficiency;    (* Must use the function name to return *)
END_FUNCTION
```

### Array Index Bounds

IEC 61131-3 allows arrays with any integer bounds including negative:
```pascal
VAR
    buffer : ARRAY[-5..5] OF REAL;
END_VAR
buffer[-3] := 1.5;
```
Most implementations do **not** perform runtime bounds checking. An out-of-bounds access silently corrupts adjacent memory. Always validate array indices in production code.

### CASE vs IF Performance

For state machines with many states, CASE is slightly more efficient than nested IF/ELSIF chains because most compilers can generate a jump table for CASE. For states 0-99 (dense integer range), jump table lookup is O(1) vs IF chain's O(n) worst case.

### SFC Token Loss is Silent

If a transition condition in an SFC is never met (e.g., a feedback signal from a field device never arrives), the sequence stays in the step indefinitely with no error or alarm. Always implement timeout transitions on every SFC step to prevent silent sequence lock-up.
