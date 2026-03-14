# NAMUR NE 148 - Engineering Data Standardization

> Official Reference: https://www.namur.net/en/recommendations/namur-recommendations/detail/ne-148.html
> Industry Context: https://www.automation.com/en-us/articles/2019/namur-ne-148-engineering-data-standardization
> Related: AutomationML IEC 62714, DEXPI ISO 15926, EDDL IEC 61804

## Overview

NAMUR NE 148 ("Requirements for Automation Engineering Interfaces for an Integrated Engineering Data Management") is a recommendation from the NAMUR organization (User Association of Automation Technology in Process Industries). It defines the interface requirements between engineering data management systems and automation systems (DCS, PLC, SCADA) to enable efficient, consistent, and traceable bulk engineering.

NAMUR is a German-based process industry user organization (founded 1949) whose members include BASF, Bayer, Dow, Evonik, and most other major European chemical and pharmaceutical manufacturers. NAMUR recommendations (NE = "NAMUR Empfehlung") are not standards (not ISO/IEC), but they represent industry consensus and are widely implemented by DCS vendors.

---

## Table of Contents

1. [Problem Statement: Why NE 148 Exists](#1-problem-statement-why-ne-148-exists)
2. [Single Source of Truth Principle](#2-single-source-of-truth-principle)
3. [Template-Based Engineering](#3-template-based-engineering)
4. [NE 148 Data Model](#4-ne-148-data-model)
5. [Workflow: P&ID to DCS Import](#5-workflow-pid-to-dcs-import)
6. [Interoperability Standards](#6-interoperability-standards)
7. [Vendor Implementations](#7-vendor-implementations)
8. [Practical Implementation Patterns](#8-practical-implementation-patterns)
9. [Corner Cases and Limitations](#9-corner-cases-and-limitations)

---

## 1. Problem Statement: Why NE 148 Exists

In traditional plant engineering, the same data is entered multiple times in different tools by different engineers:

```
P&ID tool (AVEVA PDMS/E3D, Siemens COMOS)
  → Instrument index (Excel, database)
  → DCS vendor configuration tool (EWP, TIA Portal, DeltaV)
  → HMI graphics (DigiVis, WinCC, DeltaV Live)
  → Alarm management system
  → Asset management (CMMS, SAP PM)
  → Commissioning database
```

Each data entry is a potential source of inconsistency. A tag name entered as `FIC-101A` in the P&ID might be entered as `FIC101A` in the DCS (no hyphen), `FIC-101-A` in the CMMS (different hyphen placement), and `FIC.101A` in the HMI (dot separator). These are all the same instrument, but three different systems use three different identifiers — preventing automated data exchange.

**The cost**: A typical 1000-instrument project with traditional copy-paste engineering requires:
- 2-4 weeks of DCS configuration time (manual entry)
- 1-2 weeks of HMI configuration
- 1-2 weeks of alarm configuration
- 1-2 weeks of testing/debugging data entry errors
- Total: 5-9 weeks for configuration work that is 90% non-creative data transformation

NE 148 reduces this to:
- 1 week for data validation in the master database
- 1-2 days for automated generation
- 1-2 days for verification
- Total: ~1.5-2 weeks

### The NAMUR Answer: Engineering Data Standardization

NE 148 defines:
1. What data must exist for each control point in the master engineering database
2. Which systems must consume that data
3. The interface format between systems

The goal: **enter data once, generate everything from that single source**.

---

## 2. Single Source of Truth Principle

The **Single Source of Truth (SSoT)** means one master database that is the authoritative source for all engineering data. All other systems derive their content from this database — never independently.

```
P&ID Tool (generates tag list, I/O assignment)
    │
    ↓
[MASTER ENGINEERING DATABASE]   ← Single Source of Truth
    │                              (SQLite, SQL Server, Excel+macros,
    │                               specialized tools like COMOS, AVEVA E3D)
    ├── generates → DCS configuration (PRT for Freelance, SimaticML for Siemens)
    ├── generates → HMI graphics (DMF for DigiVis, faceplates for WinCC)
    ├── generates → Alarm database (setpoints, priorities, suppression)
    ├── generates → Commissioning database (loop check sheets)
    ├── generates → Asset management (SAP PM equipment records)
    └── generates → Cable schedules, I/O lists, hook-up drawings
```

**Key principle**: If a tag needs to be renamed, it is changed **once** in the master database, and all generated files are regenerated. There is no "update in 6 places" manual process.

### What SSoT Prevents

| Problem without SSoT | Solution with SSoT |
|---------------------|-------------------|
| Tag in DCS different from P&ID | DCS generated from P&ID tag data |
| HMI shows wrong description | HMI descriptions generated from same source as P&ID |
| Alarm setpoint in DCS differs from alarm rationalization database | Both populated from same engineering record |
| Commissioning sheet has wrong I/O address | I/O address comes from one authoritative source |
| Last-minute tag changes require 6 manual updates | One database update, regenerate all outputs |

---

## 3. Template-Based Engineering

Template-based engineering is the implementation mechanism for SSoT. The principle: **design once, instantiate many times**.

### Copy-Paste Engineering (what not to do)

```
1. Configure Motor_01 manually in DCS
2. Copy Motor_01 to create Motor_02
3. Change tag names, descriptions in Motor_02 (manually, for each field)
4. Copy Motor_02 to Motor_03 ...
... repeat for 200 motors
```

Problems:
- Every copy introduces risk of missed fields
- If the motor standard changes (new alarm setpoint), must update 200 motors manually
- No traceability between motor instances and original design standard

### Template-Based Engineering (NAMUR approach)

```
1. Define Motor TEMPLATE:  (done once, reviewed and approved)
   - DCS function block type: MOT_T1
   - Standard alarms: XA1 (priority 2), XA2 (priority 2)
   - Standard EAM signals: .RUN, .FLR, .ILK, .AUT, .LOC, .SDS
   - Standard HMI faceplate: motor_faceplate_v2.dmf
   - I/O mapping: defined by template, filled from instance data

2. Build instance database:
   EquipmentID | TagName        | Description      | Area  | MotorPower | Starter | IONode | IOChannel
   M-001       | 11301.CW.1A1   | FEED CONVEYOR 1A | 11301 | 15kW       | DOL     | N13    | 5
   M-002       | 11301.CW.1B1   | FEED CONVEYOR 1B | 11301 | 15kW       | DOL     | N13    | 6
   ...200 more rows

3. Automated generation:
   - For each row in database: instantiate template with row data
   - Generate 200 PRT files (or one large PRT), 200 DMF faceplate files
   - Batch import into DCS
```

### What Changes Between Instances vs What Stays the Same

**Changes per instance (from database)**:
- Tag name / MSR name
- Description
- I/O node and channel assignments
- Alarm setpoints (if process-specific)
- Display positions/coordinates

**Stays the same (in template)**:
- Function block type
- Standard alarm configuration (priority, deadband, delays)
- Interlock logic structure
- HMI faceplate layout
- State machine logic
- EAM signal list

---

## 4. NE 148 Data Model

NE 148 defines the **minimum data set** that must be available in the engineering database for each control point. This is the data model that enables automated generation.

### Minimum Data Per Control Point (NE 148 Required Fields)

| Field Category | Specific Fields |
|----------------|----------------|
| **Identification** | Tag name (per ISA-5.1), unique ID, P&ID reference number |
| **Description** | Short description (tag description), long description, function |
| **Location** | Plant, area, unit, P&ID number, physical location |
| **Engineering Type** | Instrument type (flow/temp/pressure/level/motor/valve/...) |
| **I/O Assignment** | Controller ID, node ID, slot number, channel number, I/O type (AI/AO/DI/DO) |
| **Signal Characteristics** | Signal type (4-20mA, HART, digital, PROFIBUS, ...), range, EU |
| **Control Parameters** | Function block type, setpoints, alarm limits, PID tuning parameters |
| **Alarm Configuration** | Alarm type (HH/H/L/LL), setpoint, priority, deadband, delays |
| **Operating Conditions** | Normal operating value, min/max process range |
| **Functional Group** | Which equipment module or process cell this point belongs to |

### Extended Data Fields (NE 148 Recommended)

| Field | Purpose |
|-------|---------|
| Manufacturer / Model | Asset management, procurement |
| Calibration data | Commissioning, maintenance |
| HART device type | Field device diagnostics |
| Functional safety SIL level | Safety instrumented systems |
| ISA-88 hierarchy (Unit/EM/CM) | Batch management integration |
| NAMUR NE 107 status | Field device health status model |
| Maintenance history | CMMS integration |

### Data Schema Example (Simplified)

```sql
-- Master engineering database schema (SQLite example)

CREATE TABLE equipment (
    id              TEXT PRIMARY KEY,    -- Unique equipment ID
    tag_name        TEXT NOT NULL,       -- e.g., 11301.CW.1A1
    description     TEXT NOT NULL,       -- e.g., FEED CONVEYOR 1A
    equipment_type  TEXT NOT NULL,       -- MOTOR, VALVE, AI, AO, PID, ...
    area            TEXT NOT NULL,       -- e.g., 11301
    unit            TEXT,                -- e.g., CONVEYOR_LINE_1
    p_and_id_ref    TEXT,                -- P&ID drawing number
    template_name   TEXT NOT NULL        -- Template to use for generation
);

CREATE TABLE io_assignment (
    equipment_id    TEXT REFERENCES equipment(id),
    signal_type     TEXT NOT NULL,       -- DI, DO, AI, AO, HART
    controller_id   TEXT NOT NULL,       -- e.g., AC700F_N01
    node_id         TEXT NOT NULL,       -- e.g., N13
    slot            INTEGER,
    channel         INTEGER,
    raw_low         REAL,               -- Raw signal minimum (e.g., 4mA = 0)
    raw_high        REAL,               -- Raw signal maximum (e.g., 20mA = 32767)
    eu_low          REAL,               -- Engineering unit low (e.g., 0.0)
    eu_high         REAL,               -- Engineering unit high (e.g., 100.0)
    eu_unit         TEXT                -- e.g., m3/h, degC, bar
);

CREATE TABLE alarm_config (
    equipment_id    TEXT REFERENCES equipment(id),
    alarm_type      TEXT NOT NULL,       -- XA1, XA2, HH, H, L, LL
    priority        INTEGER NOT NULL,    -- 1, 2, 3, or 4
    setpoint        REAL,               -- NULL for digital alarms
    deadband        REAL DEFAULT 0,
    on_delay_s      INTEGER DEFAULT 0,
    off_delay_s     INTEGER DEFAULT 0,
    suppression_tag TEXT                -- Tag of suppression signal (if any)
);

CREATE TABLE generated_files (
    equipment_id    TEXT REFERENCES equipment(id),
    file_type       TEXT NOT NULL,       -- PRT, DMF, CSV, FHX
    file_path       TEXT NOT NULL,
    generated_at    TIMESTAMP,
    generation_tool TEXT,
    checksum        TEXT
);
```

---

## 5. Workflow: P&ID to DCS Import

The complete workflow from P&ID creation to DCS commissioning, following NE 148 principles:

### Phase 1: P&ID Design
1. Process engineer creates P&ID in P&ID tool (AVEVA, COMOS, AutoCAD P&ID)
2. Each instrument is tagged per ISA-5.1 convention
3. Instrument function (loop type, measurement type) defined
4. I/O type (DI/DO/AI/AO) determined from instrument type

### Phase 2: Engineering Database Population
1. Export instrument list from P&ID tool (CSV or direct interface)
2. Load into master engineering database
3. Validate: completeness check (all required NE 148 fields present)
4. Assign I/O addresses (from I/O schedule / hardware design)
5. Assign function block templates (map instrument type to DCS template)
6. Define alarm setpoints from process engineer input

### Phase 3: Automated Generation
1. Python (or other) generation script reads master database
2. For each equipment record: instantiate the appropriate template
3. Generate DCS configuration files (PRT for Freelance, SimaticML for TIA, FHX for DeltaV)
4. Generate HMI files (DMF for DigiVis, WinCC screens for TIA)
5. Generate alarm database import file
6. Validate generated files (encoding check, structure check, completeness)
7. Chunk into importable batches (50-file limit for Freelance PRT imports)

### Phase 4: DCS Import
1. Import generated files into DCS engineering tool (EWP, TIA Portal, DeltaV)
2. DCS performs consistency check (missing references, invalid types)
3. Resolve any import errors (typically caused by master data issues)
4. Compile/build DCS project

### Phase 5: Verification
1. Generated configuration vs master database — automated comparison
2. I/O list vs DCS I/O assignment — cross-check
3. Alarm configuration in DCS vs alarm rationalization database — verify setpoints
4. Witness test: spot-check 10% of generated items manually

### Phase 6: Commissioning Integration
1. Generate commissioning loop check sheets from master database
2. Commissioning engineer records field test results in database
3. Database tracks commissioning status per point
4. As-built updates: field changes recorded in database → DCS regenerated or manually patched

---

## 6. Interoperability Standards

NE 148 does not define a specific file format — it defines the data requirements. Several standards provide the interchange format:

### AutomationML (IEC 62714)

The most widely adopted format for NE 148 data exchange in Europe:
- XML-based (CAEX structure — Computer Aided Engineering eXchange)
- Carries plant topology (site/area/unit hierarchy), instrument data, connection data
- Supported by: AVEVA E3D, Siemens COMOS, AUCOTEC Engineering Base, EPLAN
- Maps NE 148 fields to CAEX attributes

```xml
<InternalElement Name="FIC-101A" ID="inst-abc-123">
  <Attribute Name="TagName" AttributeDataType="xs:string">
    <Value>11301.FIC.056A</Value>
  </Attribute>
  <Attribute Name="InstrumentType" AttributeDataType="xs:string">
    <Value>FlowTransmitter</Value>
  </Attribute>
  <Attribute Name="EU_Low" AttributeDataType="xs:float">
    <Value>0.0</Value>
  </Attribute>
  <Attribute Name="EU_High" AttributeDataType="xs:float">
    <Value>1000.0</Value>
  </Attribute>
  <Attribute Name="EU_Unit" AttributeDataType="xs:string">
    <Value>m3/h</Value>
  </Attribute>
  <Attribute Name="AlarmHH_SP" AttributeDataType="xs:float">
    <Value>950.0</Value>
  </Attribute>
  <Attribute Name="AlarmHH_Priority" AttributeDataType="xs:integer">
    <Value>2</Value>
  </Attribute>
  <Attribute Name="IO_Node" AttributeDataType="xs:string">
    <Value>N13</Value>
  </Attribute>
  <Attribute Name="IO_Channel" AttributeDataType="xs:integer">
    <Value>5</Value>
  </Attribute>
</InternalElement>
```

### DEXPI (ISO 15926 subset)

DEXPI (Data Exchange in the Process Industry) is an XML format for P&ID data exchange based on ISO 15926:
- Carries P&ID topology: pipes, instruments, connections, equipment
- Enables automated import of P&ID data into engineering databases
- Supported by: AVEVA, Siemens COMOS, Intergraph SmartP&ID
- Currently in active standardization (ISA and namur collaborate on DEXPI adoption)

### PLCopen XML (IEC 61131-10)

For DCS systems that support it (CODESYS-based), PLCopen XML carries the PLC program structure. Does NOT carry the NE 148 instrument data (process values, alarm setpoints, I/O assignment) — only the program logic.

### OPC UA Information Models

OPC UA (IEC 62541) companion specifications define information models for:
- `OPC UA for FDI` (Field Device Integration): instrument data from HART/PROFIBUS/Foundation Fieldbus devices
- `OPC UA for PLC/DCS`: program variable browsing and access
- These enable runtime data exchange but not engineering data import

---

## 7. Vendor Implementations

### DCS Vendors Supporting NE 148 / AutomationML Import

| Vendor / Product | Format Supported | Notes |
|-----------------|-----------------|-------|
| Siemens PCS 7 / TIA Portal | AutomationML, SimaticML | TIA Portal Openness API accepts AML and SimaticML |
| Emerson DeltaV | Proprietary FHX + Bulk Edit Excel | No AML support; uses proprietary formats |
| ABB System 800xA | AutomationML, ABB proprietary | 800xA has AML import; Freelance does not |
| ABB Freelance | PRT/CSV (proprietary text) | No standard interface; must use template generation |
| Honeywell Experion | CSV point import | Limited NE 148 compliance |
| Yokogawa CENTUM | AutomationML (via builder) | Supported in newer CENTUM versions |

### Engineering Tool Vendors

| Tool | Role | NE 148 Support |
|------|------|----------------|
| Siemens COMOS | Integrated engineering database | Full NE 148, exports AutomationML |
| AVEVA Engineering (formerly Aveva PDMS) | P&ID + engineering database | Full NE 148 support |
| AUCOTEC Engineering Base | Multi-discipline engineering | Full AutomationML, NE 148 |
| EPLAN | Electrical + I&C | AutomationML export, NE 148 fields |
| Excel + Python (custom) | Simple/small projects | Manual NE 148 implementation |
| SQLite + Python (custom) | Medium projects | Practical SSoT implementation |

---

## 8. Practical Implementation Patterns

For projects without access to enterprise engineering tools (COMOS, AVEVA), NE 148 principles can be implemented with:

### Pattern A: Excel as Master Database (small projects, < 500 I/O)

```
Structure:
Excel file (master.xlsx)
  Sheet: Instruments    (one row per instrument, all NE 148 fields as columns)
  Sheet: Alarms         (one row per alarm configuration)
  Sheet: IO_Assignment  (one row per I/O channel)
  Sheet: Templates      (defines which template applies to each InstrumentType)

Generation:
Python script reads Excel via openpyxl
For each instrument row: select template → instantiate → write output file

Limitation: No relational integrity, manual data consistency maintenance
```

### Pattern B: SQLite as Master Database (medium projects, 500-5000 I/O)

```
Structure:
SQLite database (project.db)
  tables: equipment, io_assignment, alarm_config, templates, generated_files

Generation:
Python script queries database, applies templates, generates output files
Tracks generation status in generated_files table
Re-generation on change: only regenerate affected equipment records

Advantages: Relational integrity, change tracking, auditability
```

### Pattern C: Specialized Engineering Tool (large projects, > 5000 I/O)

```
Tool: Siemens COMOS, AVEVA Engineering, or similar
- Native AutomationML export
- Direct DCS interface (COMOS → TIA Portal, COMOS → DeltaV)
- Change management and revision control built in
- Multi-user access with access control
- Suitable for EPC projects with 10+ engineers
```

### Template Instantiation Logic

```python
# Core template instantiation pattern
import re

def instantiate_template(template_text: str, replacements: dict) -> str:
    """
    Replace all placeholders in a template with actual values.

    Placeholders format: <<PLACEHOLDER_NAME>>
    Example: <<TAG_NAME>>, <<DESCRIPTION>>, <<IO_NODE>>

    Args:
        template_text: Raw template content as string
        replacements: Dict mapping placeholder names to actual values

    Returns:
        Instantiated text with all placeholders replaced
    """
    result = template_text
    missing = []

    for placeholder, value in replacements.items():
        marker = f"<<{placeholder}>>"
        if marker in result:
            result = result.replace(marker, str(value))

    # Check for unreplaced placeholders
    remaining = re.findall(r'<<([A-Z_]+)>>', result)
    if remaining:
        raise ValueError(
            f"Template has unreplaced placeholders: {remaining}. "
            f"Provided replacements: {list(replacements.keys())}"
        )

    return result
```

---

## 9. Corner Cases and Limitations

### NE 148 is a Recommendation, Not a Standard

NAMUR recommendations are not legally binding and have no certification body. Vendors may claim "NE 148 compatible" with varying levels of compliance. When evaluating a DCS vendor's NE 148 support, specifically ask:
- Can the vendor's engineering tool accept AutomationML import with NE 148 fields?
- Which specific NE 148 fields are supported?
- What is the import workflow (file upload, API, manual mapping)?

### Data Quality is the Primary Challenge

The NE 148 workflow fails if the master engineering database contains bad data:
- Tags named inconsistently (sometimes hyphen, sometimes dot)
- Missing I/O assignments (instruments without allocated channels)
- Duplicate tag names
- Wrong instrument types (a flow meter classified as a temperature)
- Alarm setpoints outside the process range

Before running bulk generation, always run a data validation pass. Common validations:
- Tag names match ISA-5.1 pattern (regex validation)
- All I/O channels assigned (no NULL in IO_Assignment.channel)
- All alarm setpoints within EU range (SP > EU_Low, SP < EU_High)
- No duplicate tag names
- Template name exists in templates table

### Late Engineering Changes

The biggest enemy of SSoT is late engineering changes made directly in the DCS tool (not in the master database). This immediately breaks the SSoT — the DCS is now a partial copy with unapplied differences. Discipline: all changes must go through the master database, not directly into the DCS tool. This requires process discipline and tooling support (ideally change locks on the DCS pending database update).

### I/O Address Assignment Timing

I/O addresses are typically not finalized until the electrical design is complete (cable schedules, termination schedules, I/O list). Generation of DCS configuration files requires I/O addresses. This creates a timing dependency: DCS generation cannot be completed until electrical design is sufficiently advanced. In practice, use reserved/placeholder I/O assignments early and finalize in a later generation pass.

### Freelance-Specific: DMF Files Require DCS-Assigned Addresses

In ABB Freelance, the DIGI point IDs and node IDs (used in DMF ODB section) are assigned by the DCS at import time — they are NOT known before the PRT import. This means:
1. Generate and import all PRT files first
2. Export the project to extract DIGI addresses
3. Then generate DMF files using the extracted addresses

This two-pass approach is a fundamental constraint of the Freelance architecture. No workaround exists — DMF generation must follow PRT import.

### Version Control for Generated Files

Generated DCS configuration files should be version-controlled (Git or similar):
- Track when files were generated and from which database version
- Enable rollback if a generation error is discovered after import
- Diff between generated file versions to verify only intended changes

Never version-control the master database binary (SQLite .db file) in Git — it does not diff usefully. Instead, export and version-control the master data as CSV or SQL dump files.
