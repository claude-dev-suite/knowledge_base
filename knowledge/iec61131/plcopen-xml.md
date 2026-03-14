# IEC 61131-3 - PLCopen XML TC6 and Exchange Formats

> Reference: PLCopen TC6 XML exchange format specification (IEC 61131-10), AutomationML IEC 62714

## Overview

PLCopen TC6 defines an XML-based format for exchanging IEC 61131-3 programs between different PLC vendors and engineering tools. This is the only truly vendor-neutral PLC program exchange format. This document covers the PLCopen XML structure, its compatibility across vendors, AutomationML for plant data exchange, and a comparison of all major exchange formats.

---

## Table of Contents

1. [PLCopen XML Overview](#plcopen-xml-overview)
2. [File Structure](#file-structure)
3. [Project Element](#project-element)
4. [Types Section (POU Definitions)](#types-section-pou-definitions)
5. [Instances Section (Configuration)](#instances-section-configuration)
6. [Body Encoding per Language](#body-encoding-per-language)
7. [Data Types in PLCopen XML](#data-types-in-plcopen-xml)
8. [Vendor Compatibility](#vendor-compatibility)
9. [Generating PLCopen XML with Python](#generating-plcopen-xml-with-python)
10. [AutomationML (IEC 62714)](#automationml-iec-62714)
11. [Exchange Format Comparison](#exchange-format-comparison)
12. [Migration Strategies](#migration-strategies)

---

## PLCopen XML Overview

PLCopen XML (formally IEC 61131-10, TC6 standard) is the XML encoding of IEC 61131-3 programs. It allows a program written in one vendor's tool to be opened in another vendor's tool without loss of logic.

**Key facts:**
- Namespace: `http://www.plcopen.org/xml/tc6_0201`
- File extension: `.xml` (no dedicated extension; some vendors use `.plcopen.xml`)
- Encoding: UTF-8
- Covers: all 5 languages (LD, FBD, ST, IL, SFC)
- Supported by: CODESYS, ABB AC500, Schneider Modicon, Beckhoff TwinCAT, B&R Automation Studio, Wago, Phoenix Contact
- NOT supported by: ABB Freelance, Emerson DeltaV, Siemens TIA/PCS 7 (natively)

**What PLCopen XML can represent:**
- PROGRAM, FUNCTION_BLOCK, FUNCTION definitions
- All data types (elementary, derived, struct, enum, array)
- All variable declarations (VAR, VAR_INPUT, etc.)
- All 5 language bodies
- Task configuration and program instances
- Global variable lists

**What PLCopen XML cannot represent (vendor-specific items not in scope):**
- I/O hardware configuration
- HMI displays
- Network configuration
- Vendor-specific library blocks (if not defined in the XML)
- Project-level metadata (revision history, access control)

---

## File Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://www.plcopen.org/xml/tc6_0201"
         xmlns:xhtml="http://www.w3.org/1999/xhtml"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://www.plcopen.org/xml/tc6_0201 http://www.plcopen.org/xml/tc6_0201.xsd">

  <fileHeader companyName="MyCompany"
              productName="MotorControl"
              productVersion="1.0"
              creationDateTime="2024-03-14T10:00:00"
              contentDescription="Motor control library"/>

  <contentHeader name="MotorControl"
                  modificationDateTime="2024-03-14T10:00:00">
    <coordinateInfo>
      <fbd>
        <scaling x="1" y="1"/>
      </fbd>
      <ld>
        <scaling x="1" y="1"/>
      </ld>
      <sfc>
        <scaling x="1" y="1"/>
      </sfc>
    </coordinateInfo>
  </contentHeader>

  <types>
    <!-- Data type definitions -->
    <dataTypes> ... </dataTypes>
    <!-- POU definitions (FB, FC, PROGRAM) -->
    <pous> ... </pous>
  </types>

  <instances>
    <!-- Configuration: task and program assignments -->
    <configurations> ... </configurations>
  </instances>

</project>
```

---

## Project Element

The root `<project>` element contains:

| Child Element | Purpose |
|--------------|---------|
| `<fileHeader>` | Metadata: company, product, version, date |
| `<contentHeader>` | Project name, modification date, coordinate settings |
| `<types>` | Type definitions: data types and POU bodies |
| `<instances>` | Configuration: tasks, programs, global variables |

---

## Types Section (POU Definitions)

The `<types>` section contains all POU (PROGRAM, FUNCTION_BLOCK, FUNCTION) definitions.

### FUNCTION_BLOCK definition

```xml
<types>
  <pous>
    <pou name="MotorControl_FB" pouType="functionBlock">

      <interface>
        <!-- Input variables -->
        <inputVars>
          <variable name="StartCmd">
            <type><BOOL/></type>
            <documentation>
              <xhtml:p>Start command from operator or auto logic</xhtml:p>
            </documentation>
          </variable>
          <variable name="StopCmd">
            <type><BOOL/></type>
          </variable>
          <variable name="FbkRun">
            <type><BOOL/></type>
            <documentation><xhtml:p>Running feedback from field</xhtml:p></documentation>
          </variable>
          <variable name="Interlock">
            <type><BOOL/></type>
          </variable>
          <variable name="MonTime">
            <type><TIME/></type>
            <initialValue><simpleValue value="T#10s"/></initialValue>
          </variable>
        </inputVars>

        <!-- Output variables -->
        <outputVars>
          <variable name="StartOut">
            <type><BOOL/></type>
          </variable>
          <variable name="Running">
            <type><BOOL/></type>
          </variable>
          <variable name="Faulted">
            <type><BOOL/></type>
          </variable>
        </outputVars>

        <!-- Internal (local) variables -->
        <localVars>
          <variable name="State">
            <type><INT/></type>
            <initialValue><simpleValue value="0"/></initialValue>
          </variable>
          <variable name="MonTimer">
            <type><derived name="TON"/></type>
          </variable>
        </localVars>
      </interface>

      <!-- POU body: contains the programming language implementation -->
      <body>
        <ST>
          <xhtml:body>
            <xhtml:p>
(* State machine body - see ST language section *)
CASE State OF
    0: IF StartCmd AND NOT Interlock THEN State := 1; END_IF;
    1: StartOut := TRUE;
       MonTimer(IN := TRUE, PT := MonTime);
       IF FbkRun THEN State := 2; END_IF;
       IF MonTimer.Q THEN State := 99; Faulted := TRUE; END_IF;
    2: Running := TRUE;
       IF StopCmd THEN State := 3; END_IF;
    3: StartOut := FALSE;
       Running := FALSE;
       IF NOT FbkRun THEN State := 0; END_IF;
   99: StartOut := FALSE;
       Faulted := TRUE;
END_CASE;
            </xhtml:p>
          </xhtml:body>
        </ST>
      </body>

    </pou>
  </pous>
</types>
```

### FUNCTION definition

```xml
<pou name="ScaleAnalog" pouType="function">
  <interface>
    <returnType><REAL/></returnType>
    <inputVars>
      <variable name="RawValue"><type><INT/></type></variable>
      <variable name="RawMin"><type><INT/></type></variable>
      <variable name="RawMax"><type><INT/></type></variable>
      <variable name="EULow"><type><REAL/></type></variable>
      <variable name="EUHigh"><type><REAL/></type></variable>
    </inputVars>
    <localVars>
      <variable name="Range"><type><REAL/></type></variable>
    </localVars>
  </interface>
  <body>
    <ST>
      <xhtml:body>
        <xhtml:p>
Range := INT_TO_REAL(RawMax - RawMin);
IF Range &lt;&gt; 0.0 THEN
    ScaleAnalog := EULow + (INT_TO_REAL(RawValue - RawMin) / Range) * (EUHigh - EULow);
ELSE
    ScaleAnalog := EULow;
END_IF;
        </xhtml:p>
      </xhtml:body>
    </ST>
  </body>
</pou>
```

**Note on XML character escaping in ST bodies:**
- `<` → `&lt;`
- `>` → `&gt;`
- `&` → `&amp;`
- `"` → `&quot;` (inside attributes)

---

## Instances Section (Configuration)

The `<instances>` section defines the runtime configuration: tasks, programs, and global variables.

```xml
<instances>
  <configurations>
    <configuration name="ControllerConfig">

      <!-- Global variable lists -->
      <globalVars name="GlobalVars">
        <variable name="SystemTime">
          <type><DATE_AND_TIME/></type>
        </variable>
        <variable name="EmergencyStop">
          <type><BOOL/></type>
        </variable>
      </globalVars>

      <!-- Resource (controller) definition -->
      <resource name="CPU1">

        <!-- Task definitions -->
        <task name="SlowTask" interval="T#100ms" priority="2"/>
        <task name="FastTask" interval="T#10ms" priority="1"/>

        <!-- Program instances assigned to tasks -->
        <pouInstance name="MotorPrg" typeName="MotorControl_Program">
          <task name="SlowTask"/>
        </pouInstance>

        <pouInstance name="IOPROG" typeName="DigitalIO_Program">
          <task name="FastTask"/>
        </pouInstance>

      </resource>
    </configuration>
  </configurations>
</instances>
```

---

## Body Encoding per Language

### ST (Structured Text) body

ST code is embedded as XHTML text inside the body element. The code must be XML-escaped.

```xml
<body>
  <ST>
    <xhtml:body>
      <xhtml:p>
(* IEC 61131-3 ST code here *)
Motor1(StartCmd := Start_PB, FbkRun := DI_Run.Value);
      </xhtml:p>
    </xhtml:body>
  </ST>
</body>
```

### LD (Ladder Diagram) body

LD is encoded graphically with explicit coordinates for contacts, coils, and rungs.

```xml
<body>
  <LD>
    <rung localId="1" height="20" width="160">
      <comment>
        <content><xhtml:p>Motor start interlock rung</xhtml:p></content>
      </comment>
      <leftPowerRail localId="2" width="2">
        <connectionPointOut formalParameter="" localId="2">
          <relPosition x="2" y="10"/>
        </connectionPointOut>
      </leftPowerRail>
      <contact localId="3" negated="false">
        <position x="20" y="5"/>
        <connectionPointIn>
          <connection refLocalId="2"/>
        </connectionPointIn>
        <connectionPointOut/>
        <variable>EmergencyStop_OK</variable>
      </contact>
      <contact localId="4" negated="false">
        <position x="60" y="5"/>
        <connectionPointIn>
          <connection refLocalId="3"/>
        </connectionPointIn>
        <connectionPointOut/>
        <variable>OverloadOK</variable>
      </contact>
      <coil localId="5" negated="false">
        <position x="120" y="5"/>
        <connectionPointIn>
          <connection refLocalId="4"/>
        </connectionPointIn>
        <variable>StartPermissive</variable>
      </coil>
    </rung>
  </LD>
</body>
```

### FBD (Function Block Diagram) body

FBD is encoded with block instances, pins, and connections between them.

```xml
<body>
  <FBD>
    <!-- Function block instance -->
    <block localId="1" typeName="TON" instanceName="MonTimer" height="60" width="80">
      <position x="200" y="100"/>
      <inputVariables>
        <variable formalParameter="IN">
          <connectionPointIn>
            <connection refLocalId="10"/>
          </connectionPointIn>
        </variable>
        <variable formalParameter="PT">
          <connectionPointIn>
            <connection refLocalId="11"/>
          </connectionPointIn>
        </variable>
      </inputVariables>
      <outputVariables>
        <variable formalParameter="Q">
          <connectionPointOut localId="2"/>
        </variable>
        <variable formalParameter="ET">
          <connectionPointOut localId="3"/>
        </variable>
      </outputVariables>
    </block>

    <!-- Input variable reference -->
    <inVariable localId="10" height="20" width="60">
      <position x="100" y="110"/>
      <connectionPointOut/>
      <expression>StartCmd</expression>
    </inVariable>

    <inVariable localId="11" height="20" width="60">
      <position x="100" y="140"/>
      <connectionPointOut/>
      <expression>T#10s</expression>
    </inVariable>

    <!-- Output variable reference -->
    <outVariable localId="20" height="20" width="60">
      <position x="340" y="110"/>
      <connectionPointIn>
        <connection refLocalId="2"/>
      </connectionPointIn>
      <expression>TimerDone</expression>
    </outVariable>

  </FBD>
</body>
```

### SFC body

```xml
<body>
  <SFC>
    <!-- Initial step -->
    <step localId="1" initialStep="true" name="S0_IDLE" height="40" width="80">
      <position x="200" y="50"/>
      <connectionPointOut/>
    </step>

    <!-- Transition -->
    <transition localId="2" height="12" width="80">
      <position x="200" y="110"/>
      <connectionPointIn>
        <connection refLocalId="1"/>
      </connectionPointIn>
      <connectionPointOut/>
      <condition>
        <inline name="T01">
          <ST><xhtml:body><xhtml:p>StartRequest AND TankEmpty</xhtml:p></xhtml:body></ST>
        </inline>
      </condition>
    </transition>

    <!-- Next step -->
    <step localId="3" initialStep="false" name="S1_FILLING" height="40" width="80">
      <position x="200" y="140"/>
      <connectionPointIn>
        <connection refLocalId="2"/>
      </connectionPointIn>
      <connectionPointOut/>
      <!-- Action association -->
      <actionBlock localId="4">
        <action qualifier="N" name="OpenInletValve">
          <inline>
            <ST><xhtml:body><xhtml:p>InletValve_Open_Cmd := TRUE;</xhtml:p></xhtml:body></ST>
          </inline>
        </action>
      </actionBlock>
    </step>

  </SFC>
</body>
```

---

## Data Types in PLCopen XML

### Elementary type elements

```xml
<BOOL/>
<SINT/>  <USINT/>
<INT/>   <UINT/>
<DINT/>  <UDINT/>
<LINT/>  <ULINT/>
<REAL/>  <LREAL/>
<TIME/>
<DATE/>
<TIME_OF_DAY/>
<DATE_AND_TIME/>
<STRING/>
<WSTRING/>
<BYTE/>  <WORD/>  <DWORD/>  <LWORD/>
```

### Derived type (reference to user-defined or standard type)

```xml
<derived name="MotorData"/>         <!-- User-defined struct -->
<derived name="TON"/>               <!-- Standard library FB -->
<derived name="MotorControl_FB"/>   <!-- User-defined FB -->
```

### Array type

```xml
<array>
  <dimension lower="1" upper="200"/>
  <baseType><BOOL/></baseType>
</array>
```

### User-defined struct definition

```xml
<types>
  <dataTypes>
    <dataType name="MotorData">
      <structuredType>
        <variable name="Running"><type><BOOL/></type></variable>
        <variable name="Faulted"><type><BOOL/></type></variable>
        <variable name="Speed_rpm"><type><REAL/></type></variable>
        <variable name="FaultCount"><type><INT/></type></variable>
      </structuredType>
    </dataType>
  </dataTypes>
</types>
```

---

## Vendor Compatibility

| Vendor / Tool | PLCopen XML Import | PLCopen XML Export | Notes |
|---------------|--------------------|---------------------|-------|
| CODESYS 3.5+ | Full | Full | Reference implementation |
| Beckhoff TwinCAT 3 | Full | Full | CODESYS-based |
| Schneider Unity Pro/EcoStruxure | Full | Full | TC6 native |
| B&R Automation Studio 4+ | Full | Full | |
| Wago e!COCKPIT | Full | Full | CODESYS-based |
| Phoenix Contact PLCnext | Full | Full | CODESYS-based |
| ABB AC500 (Automation Builder) | Full | Full | CODESYS-based |
| ABB Freelance | **No** | **No** | Uses PRT/CSV format |
| Siemens TIA Portal | Partial | Partial | Via 3rd party converter |
| Siemens PCS 7 | **No** | **No** | Uses SimaticML |
| Emerson DeltaV | **No** | **No** | Uses FHX format |
| Honeywell Experion | **No** | **No** | Uses CSV/proprietary |
| Allen-Bradley RSLogix/Studio 5000 | **No** | **No** | Uses L5X (proprietary XML) |

**Practical implication**: PLCopen XML is the standard for CODESYS-based systems (which represent the majority of modern IEC 61131-3 PLCs in machine automation). For DCS systems (ABB Freelance, Siemens PCS 7, Emerson DeltaV, Honeywell), vendor-specific formats must be used.

---

## Generating PLCopen XML with Python

```python
from lxml import etree
from dataclasses import dataclass, field
from typing import Optional
import xml.etree.ElementTree as ET


# Namespace definitions
NS = "http://www.plcopen.org/xml/tc6_0201"
XHTML_NS = "http://www.w3.org/1999/xhtml"
NS_MAP = {
    None: NS,
    'xhtml': XHTML_NS,
    'xsi': 'http://www.w3.org/2001/XMLSchema-instance'
}


def create_plcopen_project(
    company: str,
    product_name: str,
    version: str
) -> etree._Element:
    """Create the root PLCopen XML project element."""
    root = etree.Element(f"{{{NS}}}project", nsmap=NS_MAP)
    root.set(
        f"{{{NS_MAP['xsi']}}}schemaLocation",
        f"{NS} http://www.plcopen.org/xml/tc6_0201.xsd"
    )

    # File header
    fh = etree.SubElement(root, f"{{{NS}}}fileHeader")
    fh.set("companyName", company)
    fh.set("productName", product_name)
    fh.set("productVersion", version)
    fh.set("contentDescription", f"Generated by bulk engineering tool")

    # Content header
    ch = etree.SubElement(root, f"{{{NS}}}contentHeader")
    ch.set("name", product_name)

    coord = etree.SubElement(ch, f"{{{NS}}}coordinateInfo")
    fbd_coord = etree.SubElement(coord, f"{{{NS}}}fbd")
    scale = etree.SubElement(fbd_coord, f"{{{NS}}}scaling")
    scale.set("x", "1")
    scale.set("y", "1")

    # Types section
    types = etree.SubElement(root, f"{{{NS}}}types")
    etree.SubElement(types, f"{{{NS}}}dataTypes")
    etree.SubElement(types, f"{{{NS}}}pous")

    # Instances section
    instances = etree.SubElement(root, f"{{{NS}}}instances")
    etree.SubElement(instances, f"{{{NS}}}configurations")

    return root


def add_fb_with_st_body(
    project: etree._Element,
    fb_name: str,
    inputs: list[tuple[str, str]],   # [(name, type), ...]
    outputs: list[tuple[str, str]],  # [(name, type), ...]
    locals_: list[tuple[str, str]],  # [(name, type), ...]
    st_body: str,
    description: str = ""
) -> None:
    """
    Add a FUNCTION_BLOCK with ST body to the PLCopen XML project.

    Args:
        project: Root project element
        fb_name: Function block name
        inputs: List of (name, type_string) input variables
        outputs: List of (name, type_string) output variables
        locals_: List of (name, type_string) local variables
        st_body: Structured Text body as plain string
        description: Optional documentation string
    """
    pous = project.find(f".//{{{NS}}}pous")

    pou = etree.SubElement(pous, f"{{{NS}}}pou")
    pou.set("name", fb_name)
    pou.set("pouType", "functionBlock")

    interface = etree.SubElement(pou, f"{{{NS}}}interface")

    def add_var_section(parent, tag, var_list):
        if not var_list:
            return
        section = etree.SubElement(parent, f"{{{NS}}}{tag}")
        for var_name, var_type in var_list:
            var_el = etree.SubElement(section, f"{{{NS}}}variable")
            var_el.set("name", var_name)
            type_el = etree.SubElement(var_el, f"{{{NS}}}type")
            # Determine if elementary or derived type
            elementary_types = {
                'BOOL', 'INT', 'UINT', 'DINT', 'UDINT', 'SINT', 'USINT',
                'LINT', 'ULINT', 'REAL', 'LREAL', 'TIME', 'STRING', 'WSTRING',
                'BYTE', 'WORD', 'DWORD', 'LWORD', 'DATE', 'DATE_AND_TIME', 'TIME_OF_DAY'
            }
            if var_type.upper() in elementary_types:
                etree.SubElement(type_el, f"{{{NS}}}{var_type.upper()}")
            else:
                # Derived type (FB instance, struct, etc.)
                derived = etree.SubElement(type_el, f"{{{NS}}}derived")
                derived.set("name", var_type)

    add_var_section(interface, "inputVars", inputs)
    add_var_section(interface, "outputVars", outputs)
    add_var_section(interface, "localVars", locals_)

    # Body with ST
    body = etree.SubElement(pou, f"{{{NS}}}body")
    st = etree.SubElement(body, f"{{{NS}}}ST")
    xbody = etree.SubElement(st, f"{{{XHTML_NS}}}body")
    xp = etree.SubElement(xbody, f"{{{XHTML_NS}}}p")
    xp.text = f"\n{st_body}\n"


def write_plcopen_xml(project: etree._Element, output_path: str) -> None:
    """Write PLCopen XML to file."""
    tree = etree.ElementTree(project)
    tree.write(
        output_path,
        xml_declaration=True,
        encoding='utf-8',
        pretty_print=True
    )
    print(f"Written PLCopen XML: {output_path}")


# Example usage
if __name__ == '__main__':
    proj = create_plcopen_project(
        company="MyEngineering",
        product_name="MotorLibrary",
        version="1.0"
    )

    add_fb_with_st_body(
        project=proj,
        fb_name="MotorControl_FB",
        inputs=[
            ("StartCmd", "BOOL"),
            ("StopCmd", "BOOL"),
            ("FbkRun", "BOOL"),
            ("Interlock", "BOOL"),
            ("MonTime", "TIME"),
        ],
        outputs=[
            ("StartOut", "BOOL"),
            ("Running", "BOOL"),
            ("Faulted", "BOOL"),
        ],
        locals_=[
            ("State", "INT"),
            ("MonTimer", "TON"),
        ],
        st_body="""
(* Motor control state machine *)
CASE State OF
    0: IF StartCmd AND NOT Interlock THEN State := 1; END_IF;
    1: StartOut := TRUE;
       MonTimer(IN := TRUE, PT := MonTime);
       IF FbkRun THEN State := 2; Faulted := FALSE; END_IF;
       IF MonTimer.Q THEN State := 99; Faulted := TRUE; END_IF;
    2: Running := TRUE;
       IF StopCmd THEN State := 3; END_IF;
    3: StartOut := FALSE;
       Running := FALSE;
       IF NOT FbkRun THEN State := 0; END_IF;
   99: StartOut := FALSE;
       Faulted := TRUE;
END_CASE;
""",
        description="Single-speed motor control function block"
    )

    write_plcopen_xml(proj, "motor_library.xml")
```

---

## AutomationML (IEC 62714)

AutomationML (AML) is a separate standard (IEC 62714) for exchanging plant engineering data — not PLC programs, but the plant topology, equipment hierarchy, and instrument data.

**AML vs PLCopen XML:**

| Aspect | PLCopen XML (TC6) | AutomationML (IEC 62714) |
|--------|------------------|--------------------------|
| Contents | PLC program logic | Plant topology, equipment data |
| Use case | Logic program exchange | P&ID to DCS data transfer |
| Supported by | CODESYS-based tools | AVEVA, Siemens COMOS/PCS7, AUCOTEC |
| Standard | IEC 61131-10 | IEC 62714 |
| File extension | `.xml` | `.aml` |

**AML structure:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CAEXFile FileName="PlantData.aml"
          SchemaVersion="2.15"
          xmlns="http://www.dke.de/CAEX"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

  <ExternalReference Path="AutomationMLBaseRoleClassLib.aml"
                      Alias="AutomationMLBaseRCL"/>

  <InstanceHierarchy Name="PlantHierarchy">

    <InternalElement Name="Site_A" ID="a1b2c3d4">
      <RoleRequirements RefBaseRoleClassPath="AutomationMLBaseRCL/AutomationComponent"/>

      <InternalElement Name="Area_11301" ID="b2c3d4e5">
        <Attribute Name="AreaCode" AttributeDataType="xs:string">
          <Value>11301</Value>
        </Attribute>

        <InternalElement Name="P_101A" ID="c3d4e5f6">
          <Attribute Name="TagName" AttributeDataType="xs:string">
            <Value>11301.N.3A</Value>
          </Attribute>
          <Attribute Name="Description" AttributeDataType="xs:string">
            <Value>FEED PUMP A</Value>
          </Attribute>
          <Attribute Name="MotorPower_kW" AttributeDataType="xs:float">
            <Value>75</Value>
          </Attribute>
          <Attribute Name="StarterType" AttributeDataType="xs:string">
            <Value>DOL</Value>
          </Attribute>
          <RoleRequirements
            RefBaseRoleClassPath="AutomationMLBaseRCL/AutomationComponent"/>
        </InternalElement>

      </InternalElement>
    </InternalElement>

  </InstanceHierarchy>

</CAEXFile>
```

---

## Exchange Format Comparison

| Format | Standard | Scope | Vendor Support | Best For |
|--------|----------|-------|---------------|---------|
| PLCopen XML | IEC 61131-10 | PLC logic (all 5 languages) | CODESYS ecosystem | Logic exchange between CODESYS tools |
| SimaticML | Siemens proprietary | Siemens blocks only | TIA Portal only | Siemens bulk block import/export |
| AutomationML | IEC 62714 | Plant topology + hierarchy | AVEVA, Siemens, AUCOTEC | P&ID to DCS data transfer |
| FHX | Emerson proprietary | DeltaV modules | DeltaV only | DeltaV bulk engineering |
| PRT/CSV | ABB proprietary | Freelance project | Freelance only | Freelance bulk engineering |
| L5X | Rockwell proprietary | Studio 5000 objects | Studio 5000 only | Rockwell bulk import |
| DEXPI | ISO 15926 subset | P&ID data | AVEVA, Siemens | P&ID tool interoperability |

---

## Migration Strategies

### Between CODESYS-based tools (using PLCopen XML)

1. Export project from source tool as PLCopen XML
2. Import into target tool (CODESYS, TwinCAT, etc.)
3. Re-map I/O addresses (hardware-specific, not in PLCopen XML)
4. Verify library block availability in target (standard blocks OK; vendor-specific need migration)
5. Test compiled project

### From Freelance to TIA Portal (or DeltaV)

PLCopen XML cannot be used for this (Freelance does not support it). Strategy:
1. Extract logic from PRT files (parse the `[START_LFBS]` section)
2. Reconstruct FBD or ST equivalent logic manually or semi-automatically
3. Generate SimaticML XML (for TIA Portal) or FHX (for DeltaV) from reconstructed logic
4. This is essentially a re-engineering effort, not a format conversion

### From Siemens TIA to CODESYS

1. Export blocks as SimaticML XML from TIA Portal
2. Use a SimaticML-to-PLCopen converter (3rd party tools exist)
3. Import into CODESYS
4. Fix vendor-specific library calls (Siemens system functions → CODESYS equivalents)

### Key insight for bulk engineering

When generating code programmatically (Python-based bulk engineering):
- **Target CODESYS ecosystem**: generate PLCopen XML directly
- **Target Siemens TIA**: generate SimaticML XML, import via Openness API
- **Target Emerson DeltaV**: generate FHX text files
- **Target ABB Freelance**: generate PRT files (UTF-16LE text replacement)
- **Target Honeywell Experion**: generate CSV point import files

There is no universal format that works across all DCS platforms.
