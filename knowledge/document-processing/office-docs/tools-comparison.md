# Office Documents -- Tools Comparison

## Overview / TL;DR

Office document extraction tools vary significantly in format fidelity, structure preservation, and feature coverage. This guide compares python-docx vs mammoth for Word, python-pptx vs Unstructured for PowerPoint, and openpyxl vs pandas for Excel, with benchmarks on structure preservation, conversion quality, and processing speed.

---

## Word Document Tools

### python-docx vs mammoth

| Feature | python-docx | mammoth |
|---------|------------|---------|
| **Approach** | Direct OOXML parsing | Style-based HTML conversion |
| **Heading detection** | Via style names | Via style mapping |
| **Table extraction** | Cell-level access | Converted to HTML tables |
| **Image extraction** | Yes (embedded) | Yes (with handlers) |
| **Inline formatting** | Run-level bold/italic | Mapped to HTML tags |
| **Custom styles** | Read any style | Custom style mapping |
| **Lists** | Partial (no nesting info) | Better nesting |
| **Footnotes/endnotes** | Read access | Converted inline |
| **Headers/footers** | Read access | Not converted |
| **Track changes** | Read access | Not supported |
| **Output format** | Python objects | HTML or markdown |
| **Speed** | Fast | Fast |
| **License** | MIT | BSD-2-Clause |

```python
def compare_docx_tools(file_path: str) -> dict:
    """Compare extraction quality between python-docx and mammoth."""
    import time

    # python-docx
    start = time.perf_counter()
    md_docx = docx_to_markdown(file_path)
    docx_time = time.perf_counter() - start

    # mammoth
    start = time.perf_counter()
    md_mammoth = docx_to_markdown_mammoth(file_path)
    mammoth_time = time.perf_counter() - start

    return {
        "python_docx": {
            "chars": len(md_docx),
            "lines": md_docx.count("\n"),
            "headings": md_docx.count("\n#"),
            "tables": md_docx.count("| ---"),
            "time_ms": round(docx_time * 1000, 1),
        },
        "mammoth": {
            "chars": len(md_mammoth),
            "lines": md_mammoth.count("\n"),
            "headings": md_mammoth.count("\n#"),
            "time_ms": round(mammoth_time * 1000, 1),
        },
    }
```

**When to use python-docx**: When you need fine-grained control over document structure (cell-level table access, run-level formatting, image extraction, metadata). Best for building custom converters.

**When to use mammoth**: When you need quick HTML/markdown conversion with good style mapping. Best for simple content extraction where custom style rules handle your document templates.

---

## PowerPoint Tools

### python-pptx vs Unstructured

| Feature | python-pptx | Unstructured |
|---------|------------|-------------|
| **Text extraction** | Per-shape, per-paragraph | Element-typed output |
| **Table extraction** | Cell-level access | Structured Table elements |
| **Notes extraction** | Yes | Yes |
| **Image extraction** | Yes (shape images) | With hi_res strategy |
| **Layout analysis** | Manual (shape positions) | Automatic |
| **Grouped shapes** | Recursive access | Flattened |
| **Master slide text** | Accessible | May duplicate |
| **Output format** | Python objects | Elements with types |
| **Speed** | Very fast | Medium |
| **Dependencies** | Minimal | Heavy (with hi_res) |

```python
def compare_pptx_tools(file_path: str) -> dict:
    """Compare PowerPoint extraction tools."""
    import time

    # python-pptx
    start = time.perf_counter()
    pptx_data = extract_pptx(file_path)
    pptx_time = time.perf_counter() - start

    pptx_text = " ".join(
        text
        for slide in pptx_data["slides"]
        for text in slide["texts"]
    )

    # Unstructured
    from unstructured.partition.pptx import partition_pptx
    start = time.perf_counter()
    elements = partition_pptx(filename=file_path)
    unst_time = time.perf_counter() - start

    unst_text = " ".join(e.text for e in elements)

    return {
        "python_pptx": {
            "slides": pptx_data["total_slides"],
            "total_chars": len(pptx_text),
            "time_ms": round(pptx_time * 1000, 1),
        },
        "unstructured": {
            "elements": len(elements),
            "total_chars": len(unst_text),
            "time_ms": round(unst_time * 1000, 1),
        },
    }
```

---

## Excel Tools

### openpyxl vs pandas

| Feature | openpyxl | pandas (read_excel) |
|---------|---------|-------------------|
| **Cell-level access** | Yes | Via DataFrame indexing |
| **Formulas** | Yes (read formula text) | Values only |
| **Formatting** | Yes (fonts, colors, borders) | No |
| **Merged cells** | Yes (with merge info) | Filled as NaN |
| **Multiple sheets** | Yes | Yes (sheet_name param) |
| **Large files** | read_only mode | chunked reading |
| **Charts** | Read chart data | No |
| **Named ranges** | Yes | No |
| **Performance** | Good | Faster for DataFrames |
| **Output** | Python objects | DataFrames |
| **Data analysis** | Manual | Built-in |

```python
def compare_excel_tools(file_path: str) -> dict:
    """Compare Excel extraction tools."""
    import time
    import pandas as pd
    from openpyxl import load_workbook

    # openpyxl
    start = time.perf_counter()
    wb = load_workbook(file_path, data_only=True, read_only=True)
    total_rows_openpyxl = 0
    for sheet in wb.sheetnames:
        ws = wb[sheet]
        for row in ws.iter_rows(values_only=True):
            total_rows_openpyxl += 1
    wb.close()
    openpyxl_time = time.perf_counter() - start

    # pandas
    start = time.perf_counter()
    dfs = pd.read_excel(file_path, sheet_name=None)
    total_rows_pandas = sum(len(df) for df in dfs.values())
    pandas_time = time.perf_counter() - start

    return {
        "openpyxl": {
            "total_rows": total_rows_openpyxl,
            "time_ms": round(openpyxl_time * 1000, 1),
        },
        "pandas": {
            "total_rows": total_rows_pandas,
            "time_ms": round(pandas_time * 1000, 1),
        },
    }
```

---

## Structure Preservation Matrix

| Content Type | python-docx | mammoth | python-pptx | openpyxl | Unstructured |
|-------------|------------|---------|------------|---------|-------------|
| Headings (H1-H6) | Excellent | Excellent | N/A | N/A | Good |
| Paragraphs | Excellent | Good | Good | N/A | Good |
| Bold/Italic | Run-level | HTML tags | Run-level | Cell format | Lost |
| Bullet lists | Partial | Good | Good | N/A | Good |
| Numbered lists | Partial | Good | Good | N/A | Good |
| Tables | Cell-level | HTML table | Cell-level | Cell-level | Element |
| Merged cells | Accessible | Simplified | Accessible | Info available | Simplified |
| Images | Extractable | Extractable | Extractable | N/A | With hi_res |
| Hyperlinks | Accessible | Preserved | Accessible | Accessible | Metadata |
| Page breaks | Detectable | Lost | N/A | N/A | PageBreak element |
| Footnotes | Accessible | Inline | N/A | N/A | Inline |

---

## Decision Framework

```
What format are you processing?
  |
  +---> Word (.docx)
  |     |
  |     +---> Need fine control (tables, images, metadata)
  |     |     --> python-docx (direct access to everything)
  |     |
  |     +---> Just need clean text/markdown
  |           --> mammoth (simpler, better list handling)
  |
  +---> PowerPoint (.pptx)
  |     |
  |     +---> Need slide structure and notes
  |     |     --> python-pptx (direct access)
  |     |
  |     +---> Just need text content
  |           --> Unstructured partition_pptx (typed elements)
  |
  +---> Excel (.xlsx)
  |     |
  |     +---> Need formatting/formulas/merged cells
  |     |     --> openpyxl (full OOXML access)
  |     |
  |     +---> Just need data as tables
  |           --> pandas read_excel (fastest)
  |
  +---> Legacy formats (.doc, .xls, .ppt)
        --> LibreOffice CLI conversion to modern format, then above tools
```

---

## Common Pitfalls

1. **Using python-docx for list detection.** python-docx has limited list support -- it cannot reliably detect nested list levels. mammoth is better here.
2. **Ignoring merged cells in tables.** Both DOCX and XLSX merged cells appear as empty in non-primary positions. Always detect and handle merges.
3. **Not extracting PowerPoint notes.** Slide notes often contain more detail than the slides. Always check `has_notes_slide`.
4. **Processing .doc with python-docx.** python-docx only handles .docx (OOXML), not legacy .doc. Convert with LibreOffice first.
5. **Loading large Excel files entirely in memory.** Use `read_only=True` with openpyxl or `chunksize` with pandas for files over 100K rows.

---

## References

- python-docx -- https://python-docx.readthedocs.io/en/latest/
- mammoth -- https://github.com/mwilliamson/python-mammoth
- python-pptx -- https://python-pptx.readthedocs.io/en/latest/
- openpyxl -- https://openpyxl.readthedocs.io/en/stable/
- pandas read_excel -- https://pandas.pydata.org/docs/reference/api/pandas.read_excel.html
