# Bulk Engineering - Python Generation Patterns

> Reference: Practical patterns for DCS configuration bulk generation
> Context: ABB Freelance PRT/DMF, Emerson DeltaV FHX, PLCopen XML generation
> Related: knowledge/abb-freelance/prt-format.md, knowledge/bulk-engineering/namur-ne148.md

## Overview

Bulk engineering automation uses Python to read a master engineering database (Excel or SQLite), apply templates, and generate DCS configuration files at scale. This document covers the complete technical stack: database patterns, encoding handling, template replacement, Excel input parsing, validation, batch management, and error handling.

The primary target is ABB Freelance PRT/DMF generation, but patterns apply to any text-based DCS format.

---

## Table of Contents

1. [Architectural Pattern](#1-architectural-pattern)
2. [Encoding Handling: UTF-16LE for Freelance](#2-encoding-handling-utf-16le-for-freelance)
3. [Template Replacement Patterns](#3-template-replacement-patterns)
4. [Reading Excel with openpyxl](#4-reading-excel-with-openpyxl)
5. [SQLite for Engineering Data](#5-sqlite-for-engineering-data)
6. [Batch Generation with 50-File Limit](#6-batch-generation-with-50-file-limit)
7. [Freelance Checksum](#7-freelance-checksum)
8. [Output Validation](#8-output-validation)
9. [Error Handling and Logging](#9-error-handling-and-logging)
10. [Complete Motor PRT Generation Example](#10-complete-motor-prt-generation-example)
11. [DMF Generation Pattern](#11-dmf-generation-pattern)
12. [PLCopen XML Generation](#12-plcopen-xml-generation)
13. [Performance Optimization for Large Projects](#13-performance-optimization-for-large-projects)

---

## 1. Architectural Pattern

The standard bulk engineering architecture separates concerns into distinct layers:

```
┌─────────────────────────────────────────────────────────┐
│                     INPUT LAYER                          │
│                                                          │
│  Excel (C&E matrix, instrument list, I/O schedule)       │
│  OR SQLite database (normalized engineering data)         │
└────────────────────┬─────────────────────────────────────┘
                     │ validated, normalized Python objects
                     ↓
┌─────────────────────────────────────────────────────────┐
│                   DATA LAYER                             │
│                                                          │
│  EquipmentRecord dataclass                               │
│  AlarmConfig dataclass                                   │
│  IOAssignment dataclass                                  │
│  Validation (completeness, range checks, regex)          │
└────────────────────┬─────────────────────────────────────┘
                     │ validated records
                     ↓
┌─────────────────────────────────────────────────────────┐
│                 GENERATION LAYER                          │
│                                                          │
│  Template loading (load once, cache)                     │
│  Template instantiation (placeholder → actual value)     │
│  File writing (correct encoding per format)              │
│  Batch splitting (50 files per batch for Freelance)      │
└────────────────────┬─────────────────────────────────────┘
                     │ generated files
                     ↓
┌─────────────────────────────────────────────────────────┐
│               VALIDATION LAYER                           │
│                                                          │
│  Structure check (required sections present)             │
│  Encoding verification (correct BOM and encoding)        │
│  Placeholder check (no unreplaced <<PLACEHOLDER>>)       │
│  Cross-reference check (EAM names match MSR names)       │
└─────────────────────────────────────────────────────────┘
```

### Module Structure

```
bulk_engineering/
├── config.py           # Paths, constants, encoding settings
├── models.py           # Dataclasses for engineering records
├── readers/
│   ├── excel_reader.py # openpyxl-based Excel import
│   └── db_reader.py    # SQLite query layer
├── templates/
│   ├── loader.py       # Template file loading and caching
│   └── motor_prt.tmpl  # Motor PRT template
│   └── motor_dmf.tmpl  # Motor DMF template
│   └── ai_prt.tmpl     # Analog input template
│   └── ...
├── generators/
│   ├── prt_generator.py    # Freelance PRT generation
│   ├── dmf_generator.py    # Freelance DMF generation
│   └── plcopen_generator.py # PLCopen XML generation
├── validation/
│   ├── data_validator.py   # Input data validation
│   └── output_validator.py # Generated file validation
├── utils/
│   ├── encoding.py     # Encoding utilities for UTF-16LE
│   └── chunker.py      # Batch splitting utilities
└── main.py             # Entry point, orchestration
```

---

## 2. Encoding Handling: UTF-16LE for Freelance

ABB Freelance PRT files use **UTF-16 Little Endian** encoding with a BOM (Byte Order Mark). This is non-standard for Python's default text handling and requires explicit configuration.

### The Encoding Stack

```
UTF-16LE encoding: 2 bytes per ASCII character
BOM: FF FE (first 2 bytes of the file)
Line endings: CRLF (\r\n) — required; LF alone causes import failures
Semicolon separator: used between fields in each record line
```

### Writing a PRT File (Correct Method)

```python
import codecs
import os

def write_prt_utf16le(filepath: str, content: str) -> None:
    """
    Write a PRT file with UTF-16LE encoding and BOM.

    ABB Freelance REQUIRES:
    - UTF-16LE encoding (2 bytes per character)
    - BOM at start of file (bytes FF FE)
    - CRLF line endings (Windows-style)

    Args:
        filepath: Full path to output .prt file
        content: String content of the file (using \n line endings)
    """
    # Normalize line endings to CRLF before writing
    content_crlf = content.replace('\r\n', '\n').replace('\n', '\r\n')

    # Write with UTF-16LE encoding
    # codecs.open with 'utf-16-le' writes RAW bytes without BOM
    # We must write the BOM manually
    with open(filepath, 'wb') as f:
        # Write UTF-16LE BOM
        f.write(b'\xff\xfe')
        # Write content encoded as UTF-16LE
        f.write(content_crlf.encode('utf-16-le'))


def verify_prt_encoding(filepath: str) -> bool:
    """
    Verify that a PRT file has correct UTF-16LE encoding with BOM.
    Returns True if encoding is correct.
    """
    with open(filepath, 'rb') as f:
        bom = f.read(2)
    return bom == b'\xff\xfe'


def read_prt_file(filepath: str) -> str:
    """
    Read a PRT file, handling UTF-16LE BOM correctly.
    """
    with open(filepath, 'rb') as f:
        raw = f.read()

    if raw[:2] == b'\xff\xfe':
        # UTF-16LE with BOM
        content = raw[2:].decode('utf-16-le')
    elif raw[:2] == b'\xfe\xff':
        # UTF-16BE with BOM (shouldn't happen for Freelance, but handle gracefully)
        content = raw[2:].decode('utf-16-be')
    elif raw[:3] == b'\xef\xbb\xbf':
        # UTF-8 with BOM (incorrect for Freelance — log warning)
        content = raw[3:].decode('utf-8')
        import warnings
        warnings.warn(f"File {filepath} is UTF-8, not UTF-16LE as required by Freelance")
    else:
        # No BOM — try UTF-16LE (may fail for non-ASCII characters)
        content = raw.decode('utf-16-le', errors='replace')

    return content
```

### Common Encoding Pitfalls

```python
# WRONG: Default open() uses system encoding (usually UTF-8)
with open('output.prt', 'w') as f:
    f.write(content)
# Result: Freelance cannot read the file

# WRONG: codecs.open with 'utf-16' adds BOM automatically
# but uses platform-native byte order (may be BE on some systems)
import codecs
with codecs.open('output.prt', 'w', 'utf-16') as f:
    f.write(content)
# Result: May work on Windows x86 (LE) but fails on BE systems

# WRONG: Using 'utf-16-le' with codecs.open
# codecs.open with 'utf-16-le' does NOT write a BOM
with codecs.open('output.prt', 'w', 'utf-16-le') as f:
    f.write(content)
# Result: No BOM! Freelance may fail or misdetect encoding

# CORRECT: Write BOM manually + encode as utf-16-le
with open('output.prt', 'wb') as f:
    f.write(b'\xff\xfe')                    # BOM
    f.write(content.encode('utf-16-le'))    # Content (CRLF normalized first)
```

---

## 3. Template Replacement Patterns

### Placeholder Format Convention

Use a distinctive, collision-safe delimiter. Recommended: `<<PLACEHOLDER>>` (double angle brackets).

```python
# Template example (motor_prt.tmpl):
# <<NODE_NAME>>     - MSR node name (e.g., 11301CLWW1A1)
# <<MSR_PRIMARY>>   - Primary MSR name (e.g., 11301.CW.1A1)
# <<MSR_SHORT>>     - Abbreviated MSR name (e.g., CW.1A1)
# <<DESCRIPTION>>   - Tag description (e.g., FEED CONVEYOR 1A)
# <<GRAPH_REF>>     - Graph reference for display navigation

[HEADER]
DUMP_FILETYPE;102
PROJECT_NAME;<<PROJECT_NAME>>
CHECKSUM;0000000000

[NODE:<<NODE_NAME>>]
NODETYPE;AC700F_R
CONTROLLER;<<CONTROLLER_ID>>

[MSR:RECORD];1;<<MSR_PRIMARY>>;BST_LIB_MSR;IDF_1;<<MSR_SHORT>>;<<DESCRIPTION>>;;16384;1;;;;;2
```

### Simple String Replacement

```python
def instantiate_template(template: str, substitutions: dict) -> str:
    """
    Replace all <<PLACEHOLDER>> markers in template with actual values.

    Performance note: str.replace() is O(n*m) where n=template length, m=num substitutions.
    For 200 motors with 10 substitutions each and 2KB template: ~4MB total — fast enough.
    For 10000+ records with large templates, use regex substitution (faster for many keys).
    """
    result = template
    for key, value in substitutions.items():
        placeholder = f"<<{key}>>"
        result = result.replace(placeholder, str(value))
    return result


def check_unreplaced_placeholders(content: str) -> list:
    """Find any <<PLACEHOLDER>> that was not replaced. Returns list of found placeholders."""
    import re
    return re.findall(r'<<([A-Z0-9_]+)>>', content)
```

### Regex-Based Replacement (faster for many keys)

```python
import re

def build_replacement_function(substitutions: dict):
    """
    Build a regex-based replacement function for performance with many substitutions.
    Replaces all <<KEY>> patterns in one pass.
    """
    # Escape all keys for regex safety
    pattern = re.compile(
        r'<<(' + '|'.join(re.escape(k) for k in substitutions.keys()) + r')>>'
    )

    def replacer(match):
        key = match.group(1)
        return str(substitutions.get(key, match.group(0)))  # Keep original if key not found

    return lambda text: pattern.sub(replacer, text)
```

### Template Loading and Caching

```python
from functools import lru_cache
from pathlib import Path

TEMPLATE_DIR = Path(__file__).parent.parent / 'templates'


@lru_cache(maxsize=32)
def load_template(template_name: str) -> str:
    """
    Load and cache a template file. Uses LRU cache so each template is
    read from disk only once per process lifetime.

    Templates are stored as plain UTF-8 text files on disk.
    The encoding conversion (UTF-8 → UTF-16LE) happens at write time, not template load time.
    """
    template_path = TEMPLATE_DIR / template_name
    if not template_path.exists():
        raise FileNotFoundError(
            f"Template not found: {template_path}. "
            f"Available templates: {[t.name for t in TEMPLATE_DIR.iterdir()]}"
        )
    return template_path.read_text(encoding='utf-8')
```

---

## 4. Reading Excel with openpyxl

### Basic Excel Reading Pattern

```python
import openpyxl
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class MotorRecord:
    equipment_id:   str
    tag_name:       str          # e.g., 11301.CW.1A1
    description:    str          # e.g., FEED CONVEYOR 1A
    area:           str          # e.g., 11301
    node_short:     str          # e.g., CW.1A1 (without area prefix)
    controller_id:  str          # e.g., AC700F_N01
    io_node:        str          # e.g., N13
    io_channel:     int
    alarm_priority: int = 2
    motor_power_kw: float = 0.0
    starter_type:   str = 'DOL'
    notes:          str = ''


def read_motor_list_from_excel(filepath: str, sheet_name: str = 'Motors') -> list[MotorRecord]:
    """
    Read motor instrument list from Excel.

    Expected columns (row 1 = header, row 2+ = data):
    A: EquipmentID
    B: TagName (ISA-5.1)
    C: Description
    D: Area
    E: ControllerID
    F: IONode
    G: IOChannel
    H: MotorPower_kW
    I: StarterType
    J: AlarmPriority
    K: Notes
    """
    wb = openpyxl.load_workbook(filepath, data_only=True)

    if sheet_name not in wb.sheetnames:
        raise ValueError(
            f"Sheet '{sheet_name}' not found. Available sheets: {wb.sheetnames}"
        )

    ws = wb[sheet_name]
    records = []

    # Detect header row — look for row containing 'TagName' in column B
    header_row = None
    for row_idx, row in enumerate(ws.iter_rows(min_row=1, max_row=10, values_only=True), 1):
        if row[1] and 'tagname' in str(row[1]).lower():
            header_row = row_idx
            break

    if header_row is None:
        raise ValueError(f"Could not find header row in sheet '{sheet_name}'")

    # Build column index mapping from header
    header = list(ws.iter_rows(
        min_row=header_row, max_row=header_row, values_only=True
    ))[0]
    col_map = {str(h).strip(): idx for idx, h in enumerate(header) if h}

    # Read data rows
    for row in ws.iter_rows(
        min_row=header_row + 1,
        values_only=True
    ):
        # Skip empty rows
        if not row[col_map.get('TagName', 1)]:
            continue

        # Skip rows marked as comments (first cell = '#')
        if str(row[0]).startswith('#'):
            continue

        def get(col_name: str, default=None):
            """Get a cell value by column name."""
            idx = col_map.get(col_name)
            if idx is None:
                return default
            val = row[idx]
            if val is None:
                return default
            return val

        records.append(MotorRecord(
            equipment_id   = str(get('EquipmentID', '')).strip(),
            tag_name       = str(get('TagName', '')).strip(),
            description    = str(get('Description', '')).strip().upper(),
            area           = str(get('Area', '')).strip(),
            node_short     = _derive_node_short(str(get('TagName', ''))),
            controller_id  = str(get('ControllerID', '')).strip(),
            io_node        = str(get('IONode', '')).strip(),
            io_channel     = int(get('IOChannel', 0) or 0),
            motor_power_kw = float(get('MotorPower_kW', 0) or 0),
            starter_type   = str(get('StarterType', 'DOL')).strip(),
            alarm_priority = int(get('AlarmPriority', 2) or 2),
            notes          = str(get('Notes', '') or '').strip(),
        ))

    wb.close()
    return records


def _derive_node_short(tag_name: str) -> str:
    """
    Derive short node name from full tag name.
    '11301.CW.1A1' → 'CW.1A1' (remove area prefix)
    """
    parts = tag_name.split('.', 1)  # Split on first dot only
    return parts[1] if len(parts) > 1 else tag_name
```

### Handling Merged Cells in Excel

```python
def read_with_merged_cells(ws) -> list:
    """
    openpyxl does not automatically fill merged cell values.
    The top-left cell of a merged range contains the value;
    all other cells in the range return None.

    This function returns a sheet with merged cells expanded.
    """
    # Get all merged cell ranges
    merged_ranges = ws.merged_cells.ranges

    # For each merged range, fill all cells with the top-left value
    for merged_range in list(merged_ranges):
        top_left_value = ws.cell(
            row=merged_range.min_row,
            column=merged_range.min_col
        ).value
        ws.unmerge_cells(str(merged_range))
        for row in ws.iter_rows(
            min_row=merged_range.min_row,
            max_row=merged_range.max_row,
            min_col=merged_range.min_col,
            max_col=merged_range.max_col
        ):
            for cell in row:
                cell.value = top_left_value

    return ws
```

### Reading Multiple Data Types from Excel

```python
def safe_cell_value(raw, expected_type: type, default=None):
    """
    Safely convert an Excel cell value to the expected Python type.
    Handles None, empty strings, and type conversion errors.
    """
    if raw is None:
        return default
    if isinstance(raw, str) and raw.strip() == '':
        return default

    try:
        if expected_type == str:
            return str(raw).strip()
        elif expected_type == int:
            # Excel may return 1.0 for integer cells
            return int(float(str(raw)))
        elif expected_type == float:
            return float(str(raw))
        elif expected_type == bool:
            if isinstance(raw, bool):
                return raw
            return str(raw).upper() in ('TRUE', 'YES', '1', 'X')
        else:
            return expected_type(raw)
    except (ValueError, TypeError):
        return default
```

---

## 5. SQLite for Engineering Data

### Schema Design for Engineering Database

```python
import sqlite3
from pathlib import Path
from datetime import datetime


DB_SCHEMA = """
CREATE TABLE IF NOT EXISTS projects (
    id              TEXT PRIMARY KEY,
    name            TEXT NOT NULL,
    description     TEXT,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_modified   TIMESTAMP
);

CREATE TABLE IF NOT EXISTS equipment (
    id              TEXT PRIMARY KEY,
    project_id      TEXT REFERENCES projects(id),
    tag_name        TEXT NOT NULL,
    description     TEXT,
    equipment_type  TEXT NOT NULL,  -- MOTOR, VALVE_ON_OFF, VALVE_MODULATING, AI, AO, PID
    area            TEXT,
    unit            TEXT,
    template_name   TEXT NOT NULL,
    UNIQUE(project_id, tag_name)
);

CREATE TABLE IF NOT EXISTS io_assignment (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    equipment_id    TEXT REFERENCES equipment(id),
    signal_name     TEXT,           -- e.g., RunFbk, OpenCmd, PV
    signal_type     TEXT NOT NULL,  -- DI, DO, AI, AO
    controller_id   TEXT NOT NULL,
    node_id         TEXT NOT NULL,
    channel         INTEGER,
    eu_low          REAL,
    eu_high         REAL,
    eu_unit         TEXT
);

CREATE TABLE IF NOT EXISTS alarm_config (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    equipment_id    TEXT REFERENCES equipment(id),
    alarm_id        TEXT NOT NULL,  -- XA1, XA2, HH, H, L, LL
    priority        INTEGER NOT NULL CHECK(priority IN (1,2,3,4)),
    setpoint        REAL,
    deadband        REAL DEFAULT 0,
    on_delay_s      INTEGER DEFAULT 0,
    off_delay_s     INTEGER DEFAULT 0,
    description     TEXT
);

CREATE TABLE IF NOT EXISTS generated_files (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    equipment_id    TEXT REFERENCES equipment(id),
    file_type       TEXT NOT NULL,  -- PRT, DMF, FHX, PLCopen_XML
    file_path       TEXT NOT NULL,
    generated_at    TIMESTAMP,
    db_hash         TEXT,           -- Hash of equipment record at generation time
    is_current      BOOLEAN DEFAULT TRUE
);

CREATE INDEX IF NOT EXISTS idx_equipment_project ON equipment(project_id);
CREATE INDEX IF NOT EXISTS idx_equipment_type ON equipment(equipment_type);
CREATE INDEX IF NOT EXISTS idx_generated_current ON generated_files(equipment_id, is_current);
"""


def get_db_connection(db_path: str) -> sqlite3.Connection:
    """Get SQLite connection with row factory for dict-like access."""
    conn = sqlite3.connect(db_path)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA foreign_keys = ON")
    conn.execute("PRAGMA journal_mode = WAL")  # Better concurrent write performance
    return conn


def initialize_db(db_path: str) -> None:
    """Create schema if it doesn't exist."""
    with get_db_connection(db_path) as conn:
        conn.executescript(DB_SCHEMA)


def get_equipment_by_type(
    conn: sqlite3.Connection,
    project_id: str,
    equipment_type: str
) -> list[dict]:
    """
    Query all equipment of a given type with their I/O and alarm config.
    Returns list of dicts with all fields needed for generation.
    """
    query = """
    SELECT
        e.id,
        e.tag_name,
        e.description,
        e.equipment_type,
        e.area,
        e.unit,
        e.template_name,
        io.signal_type,
        io.controller_id,
        io.node_id,
        io.channel,
        io.eu_low,
        io.eu_high,
        io.eu_unit,
        GROUP_CONCAT(
            ac.alarm_id || ':' || ac.priority || ':' || COALESCE(ac.setpoint, ''), ';'
        ) as alarm_config
    FROM equipment e
    LEFT JOIN io_assignment io ON io.equipment_id = e.id
    LEFT JOIN alarm_config ac ON ac.equipment_id = e.id
    WHERE e.project_id = ? AND e.equipment_type = ?
    GROUP BY e.id, io.id
    ORDER BY e.tag_name
    """
    cursor = conn.execute(query, (project_id, equipment_type))
    return [dict(row) for row in cursor.fetchall()]
```

---

## 6. Batch Generation with 50-File Limit

ABB Freelance EWP has a documented limit of **50 PRT files per import batch**. Importing more than 50 at once can cause the import to fail or the EWP to crash.

### Chunking Strategy

```python
from typing import Iterator, TypeVar
from pathlib import Path
import shutil

T = TypeVar('T')


def chunked(items: list[T], chunk_size: int) -> Iterator[list[T]]:
    """Split a list into chunks of at most chunk_size items."""
    for i in range(0, len(items), chunk_size):
        yield items[i:i + chunk_size]


def generate_prt_batches(
    equipment_list: list,
    output_dir: Path,
    batch_size: int = 48,     # 48 not 50 — leave margin for safety
    generator_func = None
) -> list[Path]:
    """
    Generate PRT files in import batches of batch_size.
    Creates subdirectory for each batch: batch_001/, batch_002/, ...

    Returns list of batch directory paths for operator guidance.

    Args:
        equipment_list: List of equipment records to generate
        output_dir: Root output directory
        batch_size: Max files per batch (default 48, safely below 50 limit)
        generator_func: Function(record, output_path) → generates one PRT file
    """
    output_dir.mkdir(parents=True, exist_ok=True)
    batch_dirs = []

    for batch_num, batch in enumerate(chunked(equipment_list, batch_size), start=1):
        batch_dir = output_dir / f"batch_{batch_num:03d}"
        batch_dir.mkdir(exist_ok=True)
        batch_dirs.append(batch_dir)

        batch_log = []
        for record in batch:
            # Derive filename from tag name (sanitize for filesystem)
            safe_name = record['tag_name'].replace('.', '_').replace(' ', '_')
            output_path = batch_dir / f"{safe_name}.prt"

            try:
                generator_func(record, output_path)
                batch_log.append(f"OK: {output_path.name}")
            except Exception as e:
                batch_log.append(f"ERROR: {output_path.name}: {e}")

        # Write batch manifest for operator reference
        manifest_path = batch_dir / "BATCH_MANIFEST.txt"
        manifest_path.write_text(
            f"Batch {batch_num}/{len(list(chunked(equipment_list, batch_size)))}\n"
            f"Files: {len(batch)}\n"
            f"Generated: {datetime.now().isoformat()}\n\n"
            + "\n".join(batch_log),
            encoding='utf-8'
        )

        print(f"Generated batch {batch_num}: {len(batch)} files in {batch_dir}")

    return batch_dirs


def create_import_instructions(batch_dirs: list[Path], output_dir: Path) -> None:
    """
    Generate step-by-step import instructions for the operator.
    """
    instructions = [
        "ABB FREELANCE PRT IMPORT INSTRUCTIONS",
        "=" * 40,
        f"Total batches: {len(batch_dirs)}",
        "",
        "For each batch directory:",
        "1. Open EWP (Engineering Workplace)",
        "2. File → Import → PRT Files",
        "3. Navigate to batch directory",
        "4. Select ALL files (Ctrl+A)",
        "5. Click Import",
        "6. Wait for import to complete (check for errors)",
        "7. Proceed to next batch",
        "",
        "BATCHES TO IMPORT (in order):",
    ]
    for i, batch_dir in enumerate(batch_dirs, 1):
        file_count = len(list(batch_dir.glob('*.prt')))
        instructions.append(f"  Batch {i:03d}: {batch_dir} ({file_count} files)")

    (output_dir / "IMPORT_INSTRUCTIONS.txt").write_text(
        "\n".join(instructions), encoding='utf-8'
    )
```

---

## 7. Freelance Checksum

ABB Freelance PRT files include a `CHECKSUM` field. The checksum is calculated by Freelance itself over the file content.

**Key fact**: Setting the checksum to `0000000000` (ten zeros) instructs Freelance to **recalculate the checksum automatically** on import. This is the recommended approach for all generated files.

**Do NOT attempt to calculate the checksum manually** — the algorithm is proprietary, not publicly documented, and the auto-calculation feature is reliable.

```python
FREELANCE_CHECKSUM_PLACEHOLDER = "0000000000"

def insert_checksum_placeholder(prt_content: str) -> str:
    """
    Ensure the CHECKSUM field is set to the zero placeholder.
    Freelance will recalculate the correct checksum on import.
    """
    import re
    # If CHECKSUM line exists, replace its value
    updated = re.sub(
        r'^CHECKSUM;.*$',
        f'CHECKSUM;{FREELANCE_CHECKSUM_PLACEHOLDER}',
        prt_content,
        flags=re.MULTILINE
    )
    # If no CHECKSUM line exists, add one after [HEADER] section
    if 'CHECKSUM;' not in updated:
        updated = updated.replace('[HEADER]', '[HEADER]\nCHECKSUM;0000000000', 1)
    return updated
```

---

## 8. Output Validation

Validate all generated files before handing to the operator for import.

```python
import re
from pathlib import Path


class PRTValidationResult:
    def __init__(self):
        self.errors: list[str] = []
        self.warnings: list[str] = []
        self.is_valid: bool = True

    def add_error(self, msg: str):
        self.errors.append(msg)
        self.is_valid = False

    def add_warning(self, msg: str):
        self.warnings.append(msg)


def validate_prt_file(filepath: Path) -> PRTValidationResult:
    """
    Validate a generated PRT file before Freelance import.

    Checks:
    1. UTF-16LE encoding with correct BOM
    2. CRLF line endings
    3. Required sections present
    4. No unreplaced <<PLACEHOLDER>> markers
    5. DUMP_FILETYPE present
    6. CHECKSUM present
    """
    result = PRTValidationResult()

    # Check 1: File encoding
    with open(filepath, 'rb') as f:
        raw = f.read(4)

    if raw[:2] != b'\xff\xfe':
        result.add_error(f"Incorrect BOM: expected FF FE (UTF-16LE), got {raw[:2].hex()}")
        return result  # Cannot continue if encoding is wrong

    # Read content for further checks
    with open(filepath, 'rb') as f:
        content_bytes = f.read()
    content = content_bytes[2:].decode('utf-16-le')

    # Check 2: Line endings
    if '\r\n' not in content and '\n' in content:
        result.add_error("LF line endings detected; Freelance requires CRLF")

    # Check 3: Required sections
    required_markers = ['DUMP_FILETYPE', 'CHECKSUM', '[HEADER]']
    for marker in required_markers:
        if marker not in content:
            result.add_error(f"Missing required section/field: '{marker}'")

    # Check 4: Unreplaced placeholders
    placeholders = re.findall(r'<<([A-Z0-9_]+)>>', content)
    if placeholders:
        result.add_error(
            f"Unreplaced placeholders found: {list(set(placeholders))}"
        )

    # Check 5: MSR and EAM name consistency
    msr_names = re.findall(r'\[MSR:RECORD\];[0-9]+;([^;]+);', content)
    eam_names_structured = re.findall(r'\[EAM:RECORD\];1;([^;]+);1;', content)

    # Check that structured EAM records reference valid MSR name roots
    for eam_name in eam_names_structured:
        # Structured EAM name should correspond to an MSR name (after stripping area prefix)
        # This is a heuristic check — exact logic depends on project naming convention
        if not any(msr.replace('.', '') in eam_name or eam_name in msr.replace('.', '')
                   for msr in msr_names):
            result.add_warning(
                f"EAM structured record '{eam_name}' may not have corresponding MSR"
            )

    # Check 6: DUMP_FILETYPE is 102 (partial PRT) or 101 (full CSV)
    ft_match = re.search(r'DUMP_FILETYPE;(\d+)', content)
    if ft_match:
        filetype = int(ft_match.group(1))
        if filetype not in (101, 102):
            result.add_warning(f"Unexpected DUMP_FILETYPE: {filetype} (expected 101 or 102)")
    else:
        result.add_error("DUMP_FILETYPE not found")

    return result


def validate_batch(batch_dir: Path) -> dict:
    """
    Validate all PRT files in a batch directory.
    Returns summary statistics.
    """
    prt_files = list(batch_dir.glob('*.prt'))
    summary = {
        'total': len(prt_files),
        'valid': 0,
        'invalid': 0,
        'warnings': 0,
        'errors_by_file': {}
    }

    for prt_file in prt_files:
        result = validate_prt_file(prt_file)
        if result.is_valid:
            summary['valid'] += 1
        else:
            summary['invalid'] += 1
            summary['errors_by_file'][prt_file.name] = result.errors
        if result.warnings:
            summary['warnings'] += len(result.warnings)

    return summary
```

---

## 9. Error Handling and Logging

### Structured Logging

```python
import logging
import json
from datetime import datetime
from pathlib import Path


def setup_generation_logging(log_dir: Path, run_id: str) -> logging.Logger:
    """
    Set up structured logging for a generation run.
    Creates both human-readable and machine-readable (JSON) log files.
    """
    log_dir.mkdir(parents=True, exist_ok=True)

    logger = logging.getLogger(f"bulk_engineering.{run_id}")
    logger.setLevel(logging.DEBUG)

    # Human-readable log
    fh = logging.FileHandler(log_dir / f"{run_id}.log", encoding='utf-8')
    fh.setLevel(logging.DEBUG)
    fh.setFormatter(logging.Formatter(
        '%(asctime)s [%(levelname)s] %(message)s',
        datefmt='%Y-%m-%d %H:%M:%S'
    ))
    logger.addHandler(fh)

    # Console output
    ch = logging.StreamHandler()
    ch.setLevel(logging.INFO)
    ch.setFormatter(logging.Formatter('[%(levelname)s] %(message)s'))
    logger.addHandler(ch)

    return logger


def handle_generation_errors(
    records: list,
    generator_func,
    logger: logging.Logger,
    continue_on_error: bool = True
) -> tuple[list, list]:
    """
    Run generation for all records with structured error handling.

    Returns:
        (successful_outputs, failed_records)

    If continue_on_error=False, raises on first error.
    """
    successful = []
    failed = []

    for record in records:
        try:
            output = generator_func(record)
            successful.append(output)
            logger.debug(f"Generated: {record['tag_name']}")

        except KeyError as e:
            msg = f"Missing required field {e} for {record.get('tag_name', 'UNKNOWN')}"
            logger.error(msg)
            failed.append({'record': record, 'error': msg, 'type': 'KeyError'})
            if not continue_on_error:
                raise

        except ValueError as e:
            msg = f"Invalid value for {record.get('tag_name', 'UNKNOWN')}: {e}"
            logger.error(msg)
            failed.append({'record': record, 'error': msg, 'type': 'ValueError'})
            if not continue_on_error:
                raise

        except Exception as e:
            msg = f"Unexpected error for {record.get('tag_name', 'UNKNOWN')}: {type(e).__name__}: {e}"
            logger.exception(msg)
            failed.append({'record': record, 'error': msg, 'type': type(e).__name__})
            if not continue_on_error:
                raise

    if failed:
        logger.warning(f"Generation completed with {len(failed)} failures out of {len(records)} records")
        for f in failed:
            logger.warning(f"  FAILED: {f['record'].get('tag_name')}: {f['error']}")
    else:
        logger.info(f"All {len(records)} records generated successfully")

    return successful, failed
```

---

## 10. Complete Motor PRT Generation Example

This is a complete, runnable example generating a Freelance PRT batch for motors from an Excel input.

```python
"""
Complete motor PRT generator for ABB Freelance.

Input:  Excel file with motor list
Output: PRT files organized in batches of 48

Usage:
    python generate_motors.py motors.xlsx ./output_prts/
"""

import sys
from pathlib import Path
from dataclasses import dataclass
from datetime import datetime
import logging

# ────────────────────────────────────────────────────────
# Data model
# ────────────────────────────────────────────────────────

@dataclass
class MotorData:
    tag_name:       str    # e.g., 11301.CW.1A1
    description:    str    # e.g., FEED CONVEYOR 1A
    area:           str    # e.g., 11301
    node_name:      str    # e.g., 11301CLWW1A1 (no dots, for section headers)
    msr_short:      str    # e.g., CW.1A1 (without area prefix)
    controller_id:  str    # e.g., AC700F_N01
    io_node:        str    # e.g., N13
    di_channel_run: int    # DI channel for running feedback
    di_channel_stp: int    # DI channel for stopped feedback
    do_channel_cmd: int    # DO channel for start command

    @classmethod
    def from_excel_row(cls, row: dict) -> 'MotorData':
        tag = row['TagName'].strip()
        area = tag.split('.')[0]
        node_name = tag.replace('.', '').replace('-', '').upper()
        msr_short = '.'.join(tag.split('.')[1:])
        return cls(
            tag_name       = tag,
            description    = row['Description'].strip().upper(),
            area           = area,
            node_name      = node_name,
            msr_short      = msr_short,
            controller_id  = row['ControllerID'].strip(),
            io_node        = row['IONode'].strip(),
            di_channel_run = int(row['DI_Run']),
            di_channel_stp = int(row['DI_Stop']),
            do_channel_cmd = int(row['DO_Start']),
        )


# ────────────────────────────────────────────────────────
# Template
# ────────────────────────────────────────────────────────

MOTOR_PRT_TEMPLATE = """\
[HEADER]
DUMP_FILETYPE;102
CHECKSUM;0000000000
CONTROLLER_NAME;<<CONTROLLER_ID>>

[NODE:<<NODE_NAME>>]
NODETYPE;AC700F_R
DESCRIPTION;<<DESCRIPTION>>

[DBS:<<NODE_NAME>>]
DBS_TYPE;MOTOR1
DBS_VERSION;1.0

[EAM:<<NODE_NAME>>]
[EAM:RECORD];1;<<NODE_NAME>>;1;MOTOR1;<<DESCRIPTION>>;1;1
[EAM:RECORD];1;XA1<<NODE_NAME>>;0;BOOL;FAULT ALARM - <<DESCRIPTION>>;1;1
[EAM:RECORD];1;XA2<<NODE_NAME>>;0;BOOL;SDS ALARM - <<DESCRIPTION>>;1;1
[EAM:RECORD];1;XL<<NODE_NAME>>;0;BOOL;RUNNING STATUS - <<DESCRIPTION>>;1;1
[EAM:RECORD];1;XB1<<NODE_NAME>>;0;BOOL;LOCAL MODE - <<DESCRIPTION>>;1;1
[EAM:RECORD];1;XS1<<NODE_NAME>>;0;BOOL;START COMMAND - <<DESCRIPTION>>;1;1
[EAM:RECORD];1;XS2<<NODE_NAME>>;0;BOOL;STOP COMMAND - <<DESCRIPTION>>;1;1

[MSR:<<NODE_NAME>>]
[MSR:RECORD];1;<<MSR_PRIMARY>>;BST_LIB_MSR;IDF_1;<<MSR_SHORT>>;<<DESCRIPTION>>;;16384;1;;;;;2

[IO:<<NODE_NAME>>]
[IO:RECORD];DI;<<IO_NODE>>;<<DI_RUN>>;<<NODE_NAME>>.RunFbk;1;0
[IO:RECORD];DI;<<IO_NODE>>;<<DI_STP>>;<<NODE_NAME>>.StopFbk;1;0
[IO:RECORD];DO;<<IO_NODE>>;<<DO_CMD>>;<<NODE_NAME>>.StartCmd;0;0

"""


# ────────────────────────────────────────────────────────
# Generator
# ────────────────────────────────────────────────────────

def generate_motor_prt(motor: MotorData) -> str:
    """Generate the PRT content for a single motor. Returns string (UTF-8, LF endings)."""
    replacements = {
        'CONTROLLER_ID': motor.controller_id,
        'NODE_NAME':     motor.node_name,
        'DESCRIPTION':   motor.description,
        'MSR_PRIMARY':   motor.tag_name,
        'MSR_SHORT':     motor.msr_short,
        'IO_NODE':       motor.io_node,
        'DI_RUN':        str(motor.di_channel_run),
        'DI_STP':        str(motor.di_channel_stp),
        'DO_CMD':        str(motor.do_channel_cmd),
    }

    content = MOTOR_PRT_TEMPLATE
    for key, value in replacements.items():
        content = content.replace(f'<<{key}>>', value)

    # Verify no placeholders remain
    import re
    remaining = re.findall(r'<<([A-Z0-9_]+)>>', content)
    if remaining:
        raise ValueError(
            f"Motor {motor.tag_name}: unreplaced placeholders: {remaining}"
        )

    return content


def write_motor_prt(motor: MotorData, output_path: Path) -> None:
    """Generate and write a motor PRT file in UTF-16LE with BOM."""
    content = generate_motor_prt(motor)
    content_crlf = content.replace('\r\n', '\n').replace('\n', '\r\n')

    with open(output_path, 'wb') as f:
        f.write(b'\xff\xfe')
        f.write(content_crlf.encode('utf-16-le'))


# ────────────────────────────────────────────────────────
# Main execution
# ────────────────────────────────────────────────────────

def main(excel_path: str, output_dir: str):
    import openpyxl

    log = logging.getLogger('motor_generator')
    logging.basicConfig(level=logging.INFO, format='[%(levelname)s] %(message)s')

    # Load Excel
    wb = openpyxl.load_workbook(excel_path, data_only=True)
    ws = wb['Motors']

    # Read motors (assumes header in row 1)
    headers = [str(c.value).strip() for c in ws[1]]
    col_map = {h: i for i, h in enumerate(headers)}

    motors = []
    for row in ws.iter_rows(min_row=2, values_only=True):
        if not row[col_map['TagName']]:
            continue
        row_dict = {h: row[col_map[h]] for h in headers if h in col_map}
        try:
            motors.append(MotorData.from_excel_row(row_dict))
        except (KeyError, ValueError, TypeError) as e:
            log.error(f"Row error: {e} — {row_dict.get('TagName', 'unknown')}")

    log.info(f"Loaded {len(motors)} motors from Excel")

    # Generate in batches
    output = Path(output_dir)
    output.mkdir(parents=True, exist_ok=True)
    batch_size = 48
    batch_count = (len(motors) + batch_size - 1) // batch_size

    success_count = 0
    error_count = 0

    for batch_num, start in enumerate(range(0, len(motors), batch_size), 1):
        batch = motors[start:start + batch_size]
        batch_dir = output / f"batch_{batch_num:03d}"
        batch_dir.mkdir(exist_ok=True)

        for motor in batch:
            safe = motor.tag_name.replace('.', '_')
            out_path = batch_dir / f"{safe}.prt"
            try:
                write_motor_prt(motor, out_path)
                success_count += 1
            except Exception as e:
                log.error(f"FAILED {motor.tag_name}: {e}")
                error_count += 1

        log.info(f"Batch {batch_num}/{batch_count} written to {batch_dir}")

    log.info(f"Generation complete: {success_count} OK, {error_count} FAILED")


if __name__ == '__main__':
    if len(sys.argv) < 3:
        print("Usage: generate_motors.py <excel_input.xlsx> <output_directory>")
        sys.exit(1)
    main(sys.argv[1], sys.argv[2])
```

---

## 11. DMF Generation Pattern

DMF files must be generated AFTER PRT import (because DIGI addresses are assigned by Freelance at import time). The typical workflow:

1. Import all PRT files
2. Export project to extract DIGI addresses (via full CSV export)
3. Parse the exported CSV to extract `point_id` and `node_id` for each MSR
4. Generate DMF files using extracted addresses

```python
def extract_digi_addresses(project_csv_path: str) -> dict:
    """
    Parse a Freelance full CSV export to extract DIGI addresses
    for all MSR points.

    Returns dict: {msr_name: {'point_id': str, 'node_id': str}}
    """
    import re

    addresses = {}
    content = read_prt_file(project_csv_path)   # reuse UTF-16LE reader

    # DIGI address pattern in CSV export: DIGI,<server>,<point_id>,<node_id>
    # MSR section identifies the tag name
    current_msr = None
    for line in content.splitlines():
        msr_match = re.match(r'\[MSR:RECORD\];[0-9]+;([^;]+);', line)
        if msr_match:
            current_msr = msr_match.group(1)

        digi_match = re.search(r'"DIGI,(\d+),(\d+),(\d+)"', line)
        if digi_match and current_msr:
            addresses[current_msr] = {
                'server_id': digi_match.group(1),
                'point_id':  digi_match.group(2),
                'node_id':   digi_match.group(3),
            }

    return addresses


def generate_motor_dmf(motor: MotorData, digi_addr: dict, template: str) -> str:
    """
    Generate DMF faceplate for a motor using extracted DIGI addresses.

    digi_addr: dict from extract_digi_addresses() for this MSR
    """
    replacements = {
        'NODE_NAME':   motor.node_name,
        'DESCRIPTION': motor.description,
        'SERVER_ID':   digi_addr['server_id'],
        'POINT_ID':    digi_addr['point_id'],
        'NODE_ID':     digi_addr['node_id'],
    }
    return instantiate_template(template, replacements)
```

---

## 12. PLCopen XML Generation

For CODESYS-based systems, generate PLCopen XML directly (see also `knowledge/iec61131/plcopen-xml.md`):

```python
from lxml import etree

NS = "http://www.plcopen.org/xml/tc6_0201"
XHTML_NS = "http://www.w3.org/1999/xhtml"

def generate_motor_fb_plcopen_xml(motor_tags: list[str], output_path: str) -> None:
    """Generate a PLCopen XML file with one motor FB instance per tag."""
    nsmap = {None: NS, 'xhtml': XHTML_NS}
    root = etree.Element(f"{{{NS}}}project", nsmap=nsmap)

    fh = etree.SubElement(root, f"{{{NS}}}fileHeader")
    fh.set("productName", "MotorControl")
    fh.set("productVersion", "1.0")

    types = etree.SubElement(root, f"{{{NS}}}types")
    pous = etree.SubElement(types, f"{{{NS}}}pous")

    # Add motor FB definition (done once for the type)
    # (see plcopen-xml.md for full POU structure)

    instances = etree.SubElement(root, f"{{{NS}}}instances")
    configs = etree.SubElement(instances, f"{{{NS}}}configurations")
    config = etree.SubElement(configs, f"{{{NS}}}configuration")
    config.set("name", "MotorProject")
    resource = etree.SubElement(config, f"{{{NS}}}resource")
    resource.set("name", "CPU1")

    # Add one program instance per motor
    for tag in motor_tags:
        pou_inst = etree.SubElement(resource, f"{{{NS}}}pouInstance")
        pou_inst.set("name", tag.replace('.', '_'))
        pou_inst.set("typeName", "MotorControl_FB")

    tree = etree.ElementTree(root)
    tree.write(output_path, xml_declaration=True, encoding='utf-8', pretty_print=True)
```

---

## 13. Performance Optimization for Large Projects

### Profile Before Optimizing

```python
import cProfile
import pstats
from io import StringIO

def profile_generation(generator_func, sample_records: list) -> str:
    """Profile a generation function and return formatted stats."""
    pr = cProfile.Profile()
    pr.enable()
    generator_func(sample_records)
    pr.disable()

    sio = StringIO()
    ps = pstats.Stats(pr, stream=sio).sort_stats('cumulative')
    ps.print_stats(20)  # Top 20 functions by cumulative time
    return sio.getvalue()
```

### Optimization Strategies

**Template loading**: Load templates once and cache (use `@lru_cache` or load at module import). Never load from disk per record.

**File I/O**: For 200 motors × 2KB each = 400KB total. Even without optimization, this runs in < 1 second. File I/O is not the bottleneck for typical projects.

**String replacement**: For projects > 10,000 records, use the compiled regex replacement function (section 3) instead of sequential `str.replace()` calls. This is ~3-5x faster for many substitution keys.

**Database queries**: Use a single JOIN query to fetch all data needed for generation (equipment + io + alarms) in one query per equipment type, not N+1 queries per record.

**Encoding**: The UTF-16LE encoding step (`content.encode('utf-16-le')`) is fast but non-trivial for large templates. If generating 10,000+ files, pre-encode the template header once and only encode the dynamic parts.

**Batch writing**: Write all files for a batch sequentially (no parallel writing needed — disk I/O is fast enough for typical project sizes).

### Typical Performance Targets

| Operation | Typical Time |
|-----------|-------------|
| Read 200-row Excel file | < 0.5 seconds |
| Load and cache all templates | < 0.1 seconds |
| Generate 200 motor PRT files (write to disk) | < 2 seconds |
| Validate 200 PRT files | < 5 seconds |
| Total for 200-motor project | < 10 seconds |
| Generate 2000 points (mixed types) | < 60 seconds |
| Generate 10,000 points | < 5 minutes (optimize if slower) |
