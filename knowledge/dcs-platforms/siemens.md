# DCS Platforms - Siemens PCS 7 and TIA Portal

> Reference: Siemens PCS 7 V9.1 documentation, TIA Portal V17 documentation, APL (Advanced Process Library), TIA Openness API

## Overview

Siemens offers two DCS/PLC engineering platforms: **PCS 7** (mature large-scale DCS for continuous process) and **TIA Portal** (unified PLC/HMI platform for S7-1200/1500). PCS 7 uses SIMATIC Manager with CFC/SFC editors; TIA Portal provides a unified environment for both PLC programming and HMI configuration. Both platforms support rich automation via the TIA Openness API (.NET).

---

## Table of Contents

1. [PCS 7 Architecture](#pcs-7-architecture)
2. [TIA Portal Architecture](#tia-portal-architecture)
3. [CFC - Continuous Function Charts](#cfc---continuous-function-charts)
4. [SFC - Sequential Function Charts (S7-GRAPH)](#sfc---sequential-function-charts-s7-graph)
5. [APL Motor Block: MotL](#apl-motor-block-motl)
6. [APL Library Overview](#apl-library-overview)
7. [File Formats](#file-formats)
8. [SimaticML Block Export](#simaticml-block-export)
9. [AutomationML (PCS 7 V9+)](#automationml-pcs-7-v9)
10. [TIA Openness API](#tia-openness-api)
11. [SiVArc HMI Auto-Generation](#sivarc-hmi-auto-generation)
12. [Bulk Engineering with TIA Portal](#bulk-engineering-with-tia-portal)

---

## PCS 7 Architecture

```
PCS 7 Project
  |
  +-- OS (Operator Station)
  |     +-- WinCC Runtime
  |     +-- PDL display files (*.pdl)
  |     +-- Tag database
  |
  +-- AS (Automation Station) - S7-400 or S7-1500F
  |     +-- CFC Charts (Continuous Function Charts) - FBD-based
  |     +-- SFC Charts (Sequential Function Charts) - S7-GRAPH
  |     +-- Function Block Libraries (APL, user libraries)
  |
  +-- Hardware Configuration (HW Config tool)
  |     +-- Rack/module assignment
  |     +-- I/O addresses
  |
  +-- Plant Hierarchy (technological view, ISA-88 aligned)
  |     +-- Plant > Area > ProcessCell > Unit > EquipModule > ControlModule
  |
  +-- Process Tag Management
        +-- Tag properties, engineering data
        +-- Links to CFC instances
```

**Key PCS 7 concepts:**
- **AS**: Automation Station — the S7-400 or S7-1500 controller running CFC/SFC
- **OS**: Operator Station — the WinCC SCADA server/client
- **CFC Chart**: one graphical FBD sheet, typically one per control module
- **SFC Chart**: sequence program using S7-GRAPH (Siemens SFC implementation)
- **APL**: Advanced Process Library — the standard function block library for PCS 7

---

## TIA Portal Architecture

TIA Portal (Totally Integrated Automation Portal) unifies S7-1200/1500 PLC programming with WinCC Unified HMI configuration in a single tool.

```
TIA Portal Project (*.ap17 or *.ap19)
  |
  +-- PLC (S7-1500 / S7-1200)
  |     +-- Program blocks
  |     |     +-- OB (Organization blocks - scan tasks, interrupts)
  |     |     +-- FB (Function blocks - with instance DB)
  |     |     +-- FC (Functions - stateless)
  |     |     +-- DB (Data blocks - global data, instance data)
  |     |
  |     +-- PLC tags (global tag table)
  |     +-- User-defined types (UDT)
  |     +-- Technology objects (motion, PID)
  |
  +-- HMI (WinCC Unified / WinCC Advanced)
  |     +-- Screens (*.screen files)
  |     +-- Screen templates
  |     +-- Faceplates
  |     +-- Tags (mapped from PLC tags)
  |
  +-- Device configuration (hardware topology)
```

**TIA Portal vs PCS 7 choice:**
- Use **PCS 7** for large continuous process plants needing full DCS features (redundancy, plant hierarchy, advanced historian)
- Use **TIA Portal** for discrete manufacturing, smaller process plants, or when S7-1500 is preferred over S7-400

---

## CFC - Continuous Function Charts

CFC (Continuous Function Charts) is Siemens' implementation of FBD for PCS 7. Each CFC chart is one graphical programming sheet.

**CFC chart properties:**
- Evaluated in a defined execution order (left-to-right, top-to-bottom by default)
- Can contain multiple function block instances
- Interconnected: outputs of one block connect to inputs of another
- Cycle time assigned per chart (e.g., 100ms, 500ms, 1s)

**CFC chart structure in a typical PCS 7 motor control:**

```
Chart: Motor_P101A

  ┌──────────────┐     ┌──────────────────────────────┐     ┌──────────────┐
  │  Intlk04     │     │         MotL                  │     │  WinCC_Out   │
  │              │     │                               │     │              │
  │  In1──────── ├────►│ Intlock01    StartOut ────────├────►│ IN    OUT    ├──► Field start
  │  In2──────── │     │ FbkRun   ◄── ──────────────── ├─────┤              │
  │  In3──────── │     │ CmdStrt  ◄── (HMI/auto)       │     └──────────────┘
  │              │     │ CmdStop  ◄── (HMI/auto)       │
  │  Out ────────►     │ Protect  ◄── OL_relay          │
  └──────────────┘     │ Trip ──────────────────────────├──► WinCC alarm
                       │ RunOut ─────────────────────── ├──► WinCC status
                       └──────────────────────────────┘
```

**CFC characteristics relevant to bulk engineering:**
- CFC charts can be exported as SimaticML XML blocks
- TIA Openness API can import XML blocks programmatically
- Plant hierarchy is maintained in the project structure
- Tag generation (OS tags for WinCC) is semi-automatic in PCS 7

---

## SFC - Sequential Function Charts (S7-GRAPH)

S7-GRAPH is Siemens' SFC implementation for PCS 7 and TIA Portal (called GRAPH in TIA Portal).

**S7-GRAPH features:**
- Full ISA-88 compatible step/transition/action model
- Actions can be written in: FBD, LAD, STL (IL), SCL (ST)
- Supervision time per step (min/max dwell time)
- Permanent (background) instructions executed always
- Interlock conditions per step
- Multiple parallel and alternative branches

**Typical S7-GRAPH sequence for reactor batch:**

```
Initial Step: S0 [IDLE]
  Permanent: Alarm monitoring FBs

Transition T01: StartBatch AND ReactorEmpty AND PermitsOK

Step S1: [CHARGE_SOLVENT]
  N: Open_InletValve_CV101
  N: Monitor LT-101
  Supervision: Max=T#30m

Transition T12: LT_101.PV >= ChargeLevel_SP

Step S2: [CLOSE_INLET]
  P1: Close_InletValve_CV101

Transition T23: CV101.ZSL = TRUE  // Closed position switch

Step S3: [HEAT_TO_TEMP]
  N: Enable_Heating_TIC201
  Supervision: Max=T#120m

Transition T34: TIC_201.PV >= ReactionTemp_SP AND TIC_201.MODE = AUTO
```

---

## APL Motor Block: MotL

`MotL` (Motor Linear) is the standard single-speed motor function block from the APL (Advanced Process Library).

### MotL Signal Interface

```
Inputs:
  ModLiOp     BOOL  - Mode local/remote operator request
  AutModOp    BOOL  - Auto mode operator request
  ManModOp    BOOL  - Manual mode operator request
  FbkRun      BOOL  - Running feedback from field
  FbkStp      BOOL  - Stopped feedback from field (optional)
  Protect     BOOL  - Protection/overload input (TRUE = OK)
  Intlock01   BOOL  - Interlock input 1 (of up to 16 for MotL variants)
  CmdStrt     BOOL  - Start command (from auto logic or HMI)
  CmdStop     BOOL  - Stop command (from auto logic or HMI)
  MonTiRun    TIME  - Monitor time for run feedback (e.g., T#5s)
  MonTiStp    TIME  - Monitor time for stop feedback
  FaultReset  BOOL  - Fault reset command

Outputs:
  ModLiOp_Out BOOL  - Current mode status output (to WinCC display)
  StartOut    BOOL  - Start output to field (motor contactor)
  StopOut     BOOL  - Stop output to field
  RunOut      BOOL  - Run status (to WinCC and sequence logic)
  Trip        BOOL  - Trip/fault status (to WinCC alarm)
  SwiToLoc    BOOL  - Switched to local indication
  LocalOp     BOOL  - Local operation active
  FaultPend   BOOL  - Fault pending (acknowledgment required)

Internal:
  State       INT   - Internal state (0=Auto, 1=Manual, 2=Local, 3=Fault)
  Instance DB attached automatically by S7
```

### MotL variants in APL

| Block | Description |
|-------|-------------|
| `MotL` | Single-speed motor, linear mode |
| `MotR` | Reversing motor (CW/CCW) |
| `MotSpdL` | Variable speed motor with speed setpoint |
| `MotSpdR` | Variable speed reversing motor |
| `Mot2SpdL` | Two-speed motor |

---

## APL Library Overview

The APL (Advanced Process Library) provides standardized function blocks for PCS 7. All blocks follow a consistent interface pattern (operator faceplate compatible, WinCC message integration).

| Block | Category | Description |
|-------|----------|-------------|
| `MotL` | Motor | Single-speed motor |
| `MotR` | Motor | Reversing motor |
| `MotSpdL` | Motor | Variable speed |
| `Valve` | Valve | On/off valve |
| `AnlgValve` | Valve | Modulating valve |
| `PIDConL` | Control | PID controller |
| `Ratio` | Control | Ratio controller |
| `Casc` | Control | Cascade controller |
| `MEAS_MON` | I/O | Analog input with monitoring |
| `MON_DIGI` | I/O | Digital input with monitoring |
| `OUT_MON` | I/O | Analog output with monitoring |
| `OUT_DIGI` | I/O | Digital output with monitoring |
| `Intlk02` | Interlock | 2-input interlock with bypass |
| `Intlk04` | Interlock | 4-input interlock with bypass |
| `Intlk08` | Interlock | 8-input interlock |
| `Intlk16` | Interlock | 16-input interlock |
| `MonDi` | Monitoring | Digital signal monitoring |
| `MonAn` | Monitoring | Analog signal monitoring |

---

## File Formats

| Format | Extension | Description | Use |
|--------|-----------|-------------|-----|
| S7 Project | `.s7p` | SIMATIC Manager project (PCS 7) | Project storage |
| S7 Library | `.s7l` | SIMATIC Manager library | Block library |
| TIA Project | `.ap17`, `.ap19` | TIA Portal project archive | TIA project storage |
| WinCC Display | `.pdl` | WinCC display file (binary) | HMI graphics |
| SimaticML | `.xml` | Siemens proprietary XML block export | Block exchange |
| AutomationML | `.aml` | IEC 62714 AML (PCS 7 V9+) | Multi-vendor exchange |
| SCL | `.scl` | Structured Control Language (= IEC ST) | PLC programming |
| AWL | `.awl` | Assembly/Instruction List (legacy) | Legacy code |
| S7 Graph | `.s7g` | S7-GRAPH sequence chart export | SFC export |

---

## SimaticML Block Export

SimaticML is Siemens' proprietary XML format for exporting function blocks, data blocks, and organization blocks from TIA Portal.

**Example SimaticML structure for a FB:**

```xml
<?xml version="1.0" encoding="utf-8"?>
<Document>
  <Engineering version="V17" />
  <SW.Blocks.FB ID="0">
    <AttributeList>
      <AutoNumber>true</AutoNumber>
      <Name>MotorControl_FB</Name>
      <Namespace>MotorLib</Namespace>
      <ProgrammingLanguage>SCL</ProgrammingLanguage>
    </AttributeList>
    <ObjectList>
      <SW.Blocks.CompileUnit ID="1" CompositionName="CompileUnits">
        <AttributeList>
          <NetworkSource>
            <StructuredText>
              <xhtml:body>
                <xhtml:p>
                  (* Motor control logic in SCL/ST *)
                  IF #StartCmd AND NOT #Fault THEN
                    #StartOut := TRUE;
                  END_IF;
                </xhtml:p>
              </xhtml:body>
            </StructuredText>
          </NetworkSource>
          <ProgrammingLanguage>SCL</ProgrammingLanguage>
        </AttributeList>
      </SW.Blocks.CompileUnit>
    </ObjectList>
  </SW.Blocks.FB>
</Document>
```

**SimaticML via Python (lxml):**

```python
from lxml import etree

def create_simaticml_fb(name: str, number: int, scl_code: str) -> bytes:
    """Generate a SimaticML XML for a Function Block."""
    root = etree.Element("Document")
    eng = etree.SubElement(root, "Engineering")
    eng.set("version", "V17")

    fb = etree.SubElement(root, "SW.Blocks.FB")
    fb.set("ID", "0")

    attrs = etree.SubElement(fb, "AttributeList")
    name_el = etree.SubElement(attrs, "Name")
    name_el.text = name
    lang_el = etree.SubElement(attrs, "ProgrammingLanguage")
    lang_el.text = "SCL"

    obj_list = etree.SubElement(fb, "ObjectList")
    cu = etree.SubElement(obj_list, "SW.Blocks.CompileUnit")
    cu.set("ID", "1")
    cu.set("CompositionName", "CompileUnits")

    cu_attrs = etree.SubElement(cu, "AttributeList")
    net_src = etree.SubElement(cu_attrs, "NetworkSource")
    st_elem = etree.SubElement(net_src, "StructuredText")
    body = etree.SubElement(st_elem, "{http://www.w3.org/1999/xhtml}body")
    p = etree.SubElement(body, "{http://www.w3.org/1999/xhtml}p")
    p.text = scl_code

    lang2 = etree.SubElement(cu_attrs, "ProgrammingLanguage")
    lang2.text = "SCL"

    return etree.tostring(root, xml_declaration=True, encoding='utf-8', pretty_print=True)
```

---

## AutomationML (PCS 7 V9+)

AutomationML (AML) is an open standard (IEC 62714) for exchanging plant engineering data between tools. PCS 7 V9+ can export/import AML files.

**AML use cases:**
- Export plant hierarchy from COMOS → import into PCS 7
- Exchange P&ID data between AVEVA E3D and PCS 7
- Multi-vendor project data exchange

**AML structure (simplified):**
```xml
<CAEXFile FileName="Plant.aml" SchemaVersion="2.15">
  <InstanceHierarchy Name="PlantHierarchy">
    <InternalElement Name="Area_11301" ID="guid1">
      <InternalElement Name="Pump_P101A" ID="guid2">
        <Attribute Name="TagName" Value="11301.N.3A"/>
        <Attribute Name="Description" Value="FEED PUMP A"/>
        <Attribute Name="MotorPower_kW" Value="75"/>
        <RoleRequirements RefBaseRoleClassPath="AutomationMLBaseRoleClassLib/AutomationComponent"/>
      </InternalElement>
    </InternalElement>
  </InstanceHierarchy>
</CAEXFile>
```

---

## TIA Openness API

TIA Openness is a .NET API that allows programmatic access to TIA Portal projects. It is the primary automation tool for Siemens bulk engineering.

**Setup requirements:**
- TIA Portal V15 or later installed
- TIA Openness SDK installed (separate download from Siemens)
- .NET Framework 4.8+ or .NET 6+
- Reference DLL: `Siemens.Engineering.dll`

### Core API operations

```csharp
using Siemens.Engineering;
using Siemens.Engineering.SW;
using Siemens.Engineering.SW.Blocks;
using Siemens.Engineering.HW;
using Siemens.Engineering.HW.Features;

// ─── Connect to TIA Portal ───────────────────────────────────────────────
TiaPortal tia = new TiaPortal(TiaPortalMode.WithUserInterface);
// or: TiaPortalMode.WithoutUserInterface for headless automation

// ─── Open project ────────────────────────────────────────────────────────
Project project = tia.Projects.Open(new FileInfo(@"C:\Projects\MyPlant.ap17"));

// ─── Navigate to PLC ─────────────────────────────────────────────────────
Device plcDevice = project.Devices.First(d => d.Name == "PLC_01");
DeviceItem cpu = plcDevice.DeviceItems.First(di => di.Name == "CPU 1516F-3 PN/DP");
PlcSoftware plcSw = cpu.GetService<SoftwareContainer>().Software as PlcSoftware;

// ─── Import a block from SimaticML XML ───────────────────────────────────
PlcBlockGroup group = plcSw.BlockGroup;
group.Blocks.Import(
    new FileInfo(@"C:\Generated\MotorControl_FB.xml"),
    ImportOptions.Override
);

// ─── Create a tag table ──────────────────────────────────────────────────
PlcTagTable motorTable = plcSw.TagTableGroup.TagTables.Create("Motors_Area11301");

// ─── Add tags to table ───────────────────────────────────────────────────
PlcTag tag = motorTable.Tags.Create("P101A_StartCmd", "Bool", "%M100.0");
tag.Comment.Items.Add("de-DE", "Start command pump P-101A");

// ─── Compile ─────────────────────────────────────────────────────────────
ICompilable compilable = plcSw as ICompilable;
CompilerResult result = compilable.Compile();
foreach (CompilerResultMessage msg in result.Messages)
{
    Console.WriteLine($"{msg.Severity}: {msg.Description}");
}

// ─── Export block to XML ─────────────────────────────────────────────────
PlcBlock motorFB = group.Blocks.Find("MotorControl_FB");
motorFB.Export(new FileInfo(@"C:\Export\MotorControl_FB.xml"), ExportOptions.WithDefaults);

// ─── Save and close ──────────────────────────────────────────────────────
project.Save();
tia.Dispose();
```

### Bulk motor generation pattern

```csharp
using System.IO;
using System.Collections.Generic;

public void GenerateMotorInstances(
    PlcSoftware plcSw,
    string templateXmlPath,
    List<(string tagName, string description, int memOffset)> motors)
{
    string templateXml = File.ReadAllText(templateXmlPath);

    foreach (var (tagName, description, memOffset) in motors)
    {
        // Text replacement in SimaticML XML
        string instanceXml = templateXml
            .Replace("MOTOR_TEMPLATE", tagName)
            .Replace("MOTOR_DESCRIPTION", description)
            .Replace("DB_NUMBER_TEMPLATE", (1000 + memOffset).ToString());

        string tempPath = Path.Combine(Path.GetTempPath(), $"{tagName}.xml");
        File.WriteAllText(tempPath, instanceXml, System.Text.Encoding.UTF8);

        plcSw.BlockGroup.Blocks.Import(
            new FileInfo(tempPath),
            ImportOptions.Override
        );

        Console.WriteLine($"Imported: {tagName}");
        File.Delete(tempPath);
    }
}
```

---

## SiVArc HMI Auto-Generation

SiVArc (Siemens Virtual Automation Configurator) is a rules-based HMI auto-generation tool integrated in TIA Portal.

**How SiVArc works:**
1. Define generation rules: "For every FB of type `MotorControl_FB` in the PLC, create a WinCC faceplate instance"
2. SiVArc reads the PLC software structure
3. Applies rules to auto-generate:
   - WinCC screen elements
   - Faceplate instances with correct tag connections
   - Navigation buttons
   - Alarm configuration

**SiVArc rule example (conceptual):**
```
Rule: MotorFaceplate
  Trigger:    PLC block of type "MotorControl_FB"
  Action:     Create WinCC faceplate "MotorFaceplate_WinCC"
  Tag mapping:
    Faceplate.StartCmd    → {BlockInstance}.StartCmd
    Faceplate.StopCmd     → {BlockInstance}.StopCmd
    Faceplate.Running     → {BlockInstance}.Running
    Faceplate.Fault       → {BlockInstance}.Fault
  Position:   Auto-arranged in area group screen
```

**Benefits over manual HMI generation:**
- Single rule covers all instances — add one motor to PLC, one faceplate auto-appears in HMI
- Changes to the rule template propagate to all instances
- Eliminates manual tag connection work (the most error-prone part of HMI work)

---

## Bulk Engineering with TIA Portal

### Recommended workflow for large projects

1. **Prepare template blocks** (SCL-based FBs in SimaticML XML format)
2. **Read engineering data** from Excel/CSV (instrument index, motor list)
3. **Generate SimaticML XML** instances using Python string replacement or lxml
4. **Import via TIA Openness** (.NET C# script or PowerShell)
5. **Generate tag tables** via Openness API
6. **Run SiVArc** for HMI auto-generation
7. **Compile and validate** via Openness API

### Python → XML → TIA Portal pipeline

```python
# Python generates XML, C# imports it
import pandas as pd
from lxml import etree

def generate_motor_xml_from_template(
    template_path: str,
    motors_df: pd.DataFrame,
    output_dir: str
) -> list[str]:
    """Generate SimaticML XML files for each motor from Excel data."""
    with open(template_path, 'r', encoding='utf-8') as f:
        template = f.read()

    generated = []
    for _, row in motors_df.iterrows():
        xml = template.replace('MOTOR_TEMPLATE', row['tag_name'])
        xml = xml.replace('MOTOR_DESC', row['description'])
        xml = xml.replace('DB_NUM', str(row['db_number']))

        outpath = f"{output_dir}/{row['tag_name']}.xml"
        with open(outpath, 'w', encoding='utf-8') as f:
            f.write(xml)
        generated.append(outpath)

    return generated
```

### Comparison: PCS 7 vs TIA Portal for bulk engineering

| Aspect | PCS 7 | TIA Portal |
|--------|-------|-----------|
| Automation tool | Openness API (same) | Openness API (primary) |
| Block language for generation | CFC XML or SCL | SCL (preferred) |
| HMI auto-generation | Manual + some tools | SiVArc (powerful) |
| Tag generation | Semi-automatic (PCS 7 tool) | Via Openness API |
| Change propagation | Manual reimport | Via API + SiVArc re-run |
| Plant hierarchy | Native in PCS 7 | Manual in TIA |
