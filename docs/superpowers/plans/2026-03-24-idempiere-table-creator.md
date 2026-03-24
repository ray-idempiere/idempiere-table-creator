# iDempiere Table Creator Skill — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a Claude Code skill (`/create-table`) that reads any free-format business form (Word/Excel) and generates iDempiere 12-compatible SQL DDL plus AD_Element Application Dictionary entries.

**Architecture:** The skill lives in `~/.claude/skills/idempiere-table-creator/` as static files (SKILL.md + references + assets). No runtime code is installed — Claude reads the skill and interprets documents at invocation time. Template assets are provided as optional structured alternatives to free-form input documents.

**Tech Stack:** Markdown (skill files), Python/openpyxl (template generation), python-docx (Word template generation), iDempiere 12 / PostgreSQL

---

## File Map

| File | Responsibility |
|------|---------------|
| `~/.claude/skills/idempiere-table-creator/SKILL.md` | Triggers, 4-phase workflow instructions for Claude |
| `~/.claude/skills/idempiere-table-creator/references/idempiere-schema.md` | Data type inference rules, naming conventions, mandatory columns, known AD_Element registry |
| `~/.claude/skills/idempiere-table-creator/assets/template.xlsx` | Structured Excel input template (optional alternative to free-form docs) |
| `~/.claude/skills/idempiere-table-creator/assets/template.docx` | Structured Word input template (optional alternative to free-form docs) |
| `~/sources/idempiere-table-creator/scripts/create_templates.py` | One-time script to generate the asset templates |

---

## Task 1: Create Directory Structure

**Files:**
- Create: `~/.claude/skills/idempiere-table-creator/`
- Create: `~/.claude/skills/idempiere-table-creator/references/`
- Create: `~/.claude/skills/idempiere-table-creator/assets/`
- Create: `~/sources/idempiere-table-creator/scripts/`

- [ ] **Step 1: Create directories**

```bash
mkdir -p ~/.claude/skills/idempiere-table-creator/references
mkdir -p ~/.claude/skills/idempiere-table-creator/assets
mkdir -p ~/sources/idempiere-table-creator/scripts
```

- [ ] **Step 2: Verify**

```bash
ls ~/.claude/skills/idempiere-table-creator/
```
Expected: `assets/  references/`

- [ ] **Step 3: Confirm existing skills are untouched**

```bash
ls ~/.claude/skills/
```
Expected: `idempiere-customform-mvvm/  idempiere-table-creator/  remember/`

---

## Task 2: Write `references/idempiere-schema.md`

**Files:**
- Create: `~/.claude/skills/idempiere-table-creator/references/idempiere-schema.md`

- [ ] **Step 1: Write the schema reference file**

Write the following to `~/.claude/skills/idempiere-table-creator/references/idempiere-schema.md`:

```markdown
# iDempiere 12 Schema Reference

## Data Type Inference Rules

Use form label context to infer iDempiere column type and PostgreSQL DDL type.

| Form pattern (label or surrounding context) | iDempiere Type | PostgreSQL DDL | Notes |
|---|---|---|---|
| `年月日` / `Date` / `日期` in label | `Date` | `TIMESTAMP` | |
| `年月 ~ 年月` date range | Two columns: `StartDate`, `EndDate` | `TIMESTAMP` | Split into Start/End |
| `□男 □女` / `□` checkbox list (2+ options) | `List` | `CHAR(1)` | Use first letter of each option as value |
| Single `□` checkbox / `Is` prefix | `YesNo` | `CHAR(1) NOT NULL DEFAULT 'N'` | |
| `電話` / `Phone` / `Tel` / `Fax` | `String` | `VARCHAR(20)` | |
| `手機` / `Mobile` / `Cell` | `String` | `VARCHAR(20)` | |
| `地址` / `Address` | `String` | `VARCHAR(255)` | |
| `金額` / `薪` / `Amount` / `Total` / `Amt` | `Amount` | `NUMERIC(20,2)` | |
| `編號` / `No` / `Code` / short identifier | `String` | `VARCHAR(30)` | |
| `姓名` / `Name` / `名稱` | `String` | `VARCHAR(60)` | |
| `說明` / `Description` / `備註` / `Remark` | `String` | `VARCHAR(255)` | |
| `身分證` / `證號` / ID number | `String` | `VARCHAR(20)` | |
| `_ID` suffix / Foreign Key | `ID` | `NUMERIC(10,0)` | References parent table |
| `數量` / `Qty` / `Quantity` | `Quantity` | `NUMERIC(20,4)` | |
| `百分比` / `%` / `Percent` | `Number` | `NUMERIC(10,2)` | |
| Anything else (default) | `String` | `VARCHAR(60)` | When in doubt |

## Column Naming Conventions

- **PascalCase** — e.g., `EmployeeNo`, `HireDate`, `IDNo`
- **No underscores** except `_ID` FK suffix and standard iDempiere columns (`AD_Client_ID`, etc.)
- **Max 30 characters** (iDempiere hard limit)
- **`_ID` suffix** for foreign keys (e.g., `C_BPartner_ID`, `C_DocType_ID`)

### Chinese Label → English Column Name

Translate semantically, not literally:

| Chinese label | Column name |
|---|---|
| 員工編號 | EmployeeNo |
| 姓名 | Name |
| 身分證字號 | IDNo |
| 生日 / 出生年月日 | BirthDate |
| 到職日 | HireDate |
| 離職日 | LeaveDate |
| 職稱 | Title |
| 性別 | Gender |
| 通訊地址 | Address |
| 戶籍地址 | PermanentAddress |
| 連絡電話 / 電話 | Phone |
| 手機 | Mobile |
| 電子信箱 | Email |
| 部門 | Department |
| 應徵職務 | Position |
| 學校名稱 | SchoolName |
| 科系 | Department (child) |
| 畢/肄業 | GradStatus |
| 日/夜間 | DayNight |
| 公司名稱 | CompanyName |
| 月薪 | MonthlySalary |
| 離職原因 | LeaveReason |
| 關係 / 稱謂 | Relation |
| 緊急聯絡人 | EmergencyContact |
| 備註 | Description |

## Mandatory Columns (Always Add, Never Ask User)

Every iDempiere table must have these columns in this order:

```sql
<TableName>_ID   NUMERIC(10,0)  NOT NULL,
AD_Client_ID     NUMERIC(10,0)  NOT NULL,
AD_Org_ID        NUMERIC(10,0)  NOT NULL,
IsActive         CHAR(1)        NOT NULL DEFAULT 'Y',
Created          TIMESTAMP      NOT NULL DEFAULT NOW(),
CreatedBy        NUMERIC(10,0)  NOT NULL,
Updated          TIMESTAMP      NOT NULL DEFAULT NOW(),
UpdatedBy        NUMERIC(10,0)  NOT NULL,
```

Primary key constraint:
```sql
CONSTRAINT <TableName>_PK PRIMARY KEY (<TableName>_ID)
```

## Document Workflow Columns (Add When IsDocEnabled = Y)

```sql
DocStatus           CHAR(2)       NOT NULL DEFAULT 'DR',
DocAction           CHAR(2)       NOT NULL DEFAULT 'CO',
Processed           CHAR(1)       NOT NULL DEFAULT 'N',
Processing          CHAR(1)       NOT NULL DEFAULT 'N',
DocumentNo          VARCHAR(30),
C_DocType_ID        NUMERIC(10,0),
C_DocTypeTarget_ID  NUMERIC(10,0),
-- optional (include if applicable):
IsApproved          CHAR(1)       DEFAULT 'N',
Posted              CHAR(1)       DEFAULT 'N',
DateAcct            TIMESTAMP,
C_Currency_ID       NUMERIC(10,0),
```

## Known AD_Element Registry (Skip — Already Exist)

These columns already exist in `AD_Element`. Do NOT generate INSERT statements for them:

```
AD_Client_ID, AD_Org_ID, IsActive, Created, CreatedBy, Updated, UpdatedBy,
DocStatus, DocAction, Processed, Processing, DocumentNo,
C_DocType_ID, C_DocTypeTarget_ID, IsApproved, Posted, DateAcct,
C_Currency_ID, Name, Description, Help, EntityType,
C_BPartner_ID, C_BPartner_Location_ID, AD_User_ID, C_Currency_ID,
C_Activity_ID, C_Campaign_ID, C_Project_ID, C_CostCenter_ID,
SalesRep_ID, M_Warehouse_ID, M_PriceList_ID,
DateOrdered, DateAcct, DatePromised,
GrandTotal, TotalLines, FreightAmt, ChargeAmt,
POReference, DocumentNo, DeliveryRule, InvoiceRule
```

## `nextid` Function

```sql
-- Correct signature (iDempiere 12, PostgreSQL):
nextid('AD_Element', 'N')
--      ^ table name   ^ 'N' = user record
-- Returns INTEGER. Use inside DO $$ DECLARE v_id INTEGER block.
```

## Child Table Conventions

- Child table name: `<ParentTable>_<SectionName>` (e.g., `HR_EmployeeCard_Education`)
- Child PK: `<ChildTable>_ID NUMERIC(10,0) NOT NULL`
- Parent FK: `<ParentTable>_ID NUMERIC(10,0) NOT NULL`
- FK constraint: `CONSTRAINT <ChildTable>_Parent FOREIGN KEY (<ParentTable>_ID) REFERENCES <ParentTable>(<ParentTable>_ID)`
```

- [ ] **Step 2: Verify file was written**

```bash
wc -l ~/.claude/skills/idempiere-table-creator/references/idempiere-schema.md
```
Expected: 100+ lines

---

## Task 3: Generate Excel Template

**Files:**
- Create: `~/sources/idempiere-table-creator/scripts/create_templates.py`
- Create: `~/.claude/skills/idempiere-table-creator/assets/template.xlsx`

- [ ] **Step 1: Write the template generation script**

Write to `~/sources/idempiere-table-creator/scripts/create_templates.py`:

```python
#!/usr/bin/env python3
"""Generate iDempiere table creator asset templates."""
import os
import openpyxl
from openpyxl.styles import Font, PatternFill, Alignment, Border, Side

ASSETS_DIR = os.path.expanduser("~/.claude/skills/idempiere-table-creator/assets")


def make_excel_template():
    wb = openpyxl.Workbook()

    # ── Sheet 1: Table Info ──────────────────────────────────────
    ws1 = wb.active
    ws1.title = "Table Info"

    header_font = Font(bold=True)
    header_fill = PatternFill("solid", fgColor="DDEBF7")
    input_fill  = PatternFill("solid", fgColor="FFFFE0")

    rows = [
        ("Field",           "Value",                  "Notes"),
        ("Prefix",          "",                       "Module prefix without underscore, e.g. HR, CVG"),
        ("Table Name",      "",                       "PascalCase, e.g. EmployeeCard (prefix added automatically)"),
        ("Description",     "",                       "Short description for SQL comment"),
        ("Entity Type",     "U",                      "U = user-defined; override for specific module"),
        ("Is Doc Enabled",  "N",                      "Y = add DocStatus/DocAction workflow columns"),
    ]

    ws1.column_dimensions["A"].width = 20
    ws1.column_dimensions["B"].width = 35
    ws1.column_dimensions["C"].width = 55

    for i, (field, value, note) in enumerate(rows, start=1):
        ws1.cell(i, 1, field).font = header_font
        ws1.cell(i, 1).fill = header_fill if i == 1 else PatternFill()
        ws1.cell(i, 2, value).fill = input_fill if i > 1 else PatternFill()
        ws1.cell(i, 3, note).font = Font(italic=True, color="666666")

    ws1.cell(1, 1).fill = header_fill
    ws1.cell(1, 2).fill = header_fill
    ws1.cell(1, 3).fill = header_fill

    # ── Sheet 2: Columns ────────────────────────────────────────
    ws2 = wb.create_sheet("Columns")

    col_headers = [
        "ColumnName", "DataType", "Length", "Mandatory",
        "FK Table", "Chinese Name (zh_TW)", "Description / Help",
    ]
    col_widths = [25, 15, 10, 12, 25, 25, 40]

    for j, (h, w) in enumerate(zip(col_headers, col_widths), start=1):
        cell = ws2.cell(1, j, h)
        cell.font = header_font
        cell.fill = header_fill
        ws2.column_dimensions[chr(64 + j)].width = w

    # Example rows
    examples = [
        ("EmployeeNo",   "String",  "30", "N", "",             "員工編號",   "Unique employee identifier"),
        ("Name",         "String",  "60", "Y", "",             "姓名",       "Full name"),
        ("BirthDate",    "Date",    "",   "N", "",             "生日",       ""),
        ("Gender",       "List",    "1",  "N", "",             "性別",       "M=Male, F=Female"),
        ("C_BPartner_ID","ID",      "",   "N", "C_BPartner",   "業務夥伴",   ""),
    ]
    for r, row in enumerate(examples, start=2):
        for c, val in enumerate(row, start=1):
            ws2.cell(r, c, val).fill = input_fill

    # Instructions sheet
    ws3 = wb.create_sheet("Instructions")
    instructions = [
        ("iDempiere Table Creator — Template Instructions",),
        ("",),
        ("Sheet 1 (Table Info):",),
        ("  Prefix        — module prefix WITHOUT underscore, e.g. HR, CVG, IVG",),
        ("  Table Name    — PascalCase name WITHOUT prefix, e.g. EmployeeCard",),
        ("  Is Doc Enabled — Y to add DocStatus/DocAction workflow columns",),
        ("",),
        ("Sheet 2 (Columns):",),
        ("  ColumnName    — PascalCase, max 30 chars, no spaces",),
        ("  DataType      — String / Date / Amount / ID / YesNo / List / Number / Quantity",),
        ("  Length        — required for String type; leave blank for others",),
        ("  Mandatory     — Y or N",),
        ("  FK Table      — if DataType=ID, the referenced table name",),
        ("  Chinese Name  — zh_TW translation for AD_Element_Trl (IsTranslated=Y if filled)",),
        ("",),
        ("Do NOT add mandatory columns: AD_Client_ID, AD_Org_ID, IsActive, Created,",),
        ("CreatedBy, Updated, UpdatedBy — these are auto-generated.",),
    ]
    ws3.column_dimensions["A"].width = 80
    for r, (text,) in enumerate(instructions, start=1):
        cell = ws3.cell(r, 1, text)
        if r == 1:
            cell.font = Font(bold=True, size=13)

    path = os.path.join(ASSETS_DIR, "template.xlsx")
    wb.save(path)
    print(f"Created: {path}")


if __name__ == "__main__":
    os.makedirs(ASSETS_DIR, exist_ok=True)
    make_excel_template()
```

- [ ] **Step 2: Ensure openpyxl is available**

```bash
python3 -c "import openpyxl; print('ok')" 2>/dev/null || pip3 install openpyxl -q && echo "installed"
```

- [ ] **Step 3: Run the script to generate the Excel template**

```bash
python3 ~/sources/idempiere-table-creator/scripts/create_templates.py
```
Expected: `Created: /Users/ray/.claude/skills/idempiere-table-creator/assets/template.xlsx`

- [ ] **Step 4: Verify the Excel file**

```bash
python3 -c "
import openpyxl
wb = openpyxl.load_workbook('/Users/ray/.claude/skills/idempiere-table-creator/assets/template.xlsx')
print('Sheets:', wb.sheetnames)
ws = wb['Table Info']
for row in ws.iter_rows(values_only=True):
    if any(c for c in row): print(row)
"
```
Expected: sheets `['Table Info', 'Columns', 'Instructions']`, rows showing Prefix/Table Name/etc.

---

## Task 4: Generate Word Template

**Files:**
- Modify: `~/sources/idempiere-table-creator/scripts/create_templates.py` (add docx function)
- Create: `~/.claude/skills/idempiere-table-creator/assets/template.docx`

- [ ] **Step 1: Install python-docx if needed**

```bash
python3 -c "import docx; print('ok')" 2>/dev/null || pip3 install python-docx -q && echo "installed"
```

- [ ] **Step 2: Add Word template generation to the script**

Append to `~/sources/idempiere-table-creator/scripts/create_templates.py` before `if __name__ == "__main__":`:

```python
def make_word_template():
    from docx import Document
    from docx.shared import Pt, RGBColor, Cm
    from docx.enum.text import WD_ALIGN_PARAGRAPH

    doc = Document()

    # Title
    title = doc.add_heading("iDempiere Table Specification", 0)
    title.alignment = WD_ALIGN_PARAGRAPH.CENTER

    doc.add_paragraph("Fill in this document and run /create-table <path-to-this-file>")
    doc.add_paragraph("")

    # ── Section 1: Table Metadata ────────────────────────────────
    doc.add_heading("1. Table Metadata", level=1)

    meta_table = doc.add_table(rows=6, cols=2)
    meta_table.style = "Table Grid"
    meta_table.column_cells(0)[0].width = Cm(5)
    meta_table.column_cells(1)[0].width = Cm(10)

    meta_rows = [
        ("Prefix",           "(e.g. HR, CVG — no underscore)"),
        ("Table Name",       "(PascalCase, no prefix, e.g. EmployeeCard)"),
        ("Description",      ""),
        ("Entity Type",      "U"),
        ("Is Doc Enabled",   "N  (Y = add DocStatus/DocAction workflow columns)"),
    ]
    header_row = meta_table.rows[0]
    header_row.cells[0].text = "Field"
    header_row.cells[1].text = "Value"
    for cell in header_row.cells:
        cell.paragraphs[0].runs[0].font.bold = True

    for i, (field, hint) in enumerate(meta_rows, start=1):
        meta_table.rows[i].cells[0].text = field
        meta_table.rows[i].cells[1].text = hint

    doc.add_paragraph("")

    # ── Section 2: Column Definitions ───────────────────────────
    doc.add_heading("2. Column Definitions", level=1)
    doc.add_paragraph(
        "List each column below. Do NOT include mandatory columns "
        "(AD_Client_ID, AD_Org_ID, IsActive, Created/By, Updated/By) — "
        "they are auto-generated."
    )

    col_table = doc.add_table(rows=6, cols=7)
    col_table.style = "Table Grid"

    col_headers = [
        "ColumnName", "DataType", "Length", "Mandatory",
        "FK Table", "Chinese Name\n(zh_TW)", "Description"
    ]
    col_widths_cm = [4.0, 2.5, 1.5, 2.0, 3.5, 4.0, 5.5]

    header_row = col_table.rows[0]
    for j, (h, w) in enumerate(zip(col_headers, col_widths_cm)):
        cell = header_row.cells[j]
        cell.text = h
        cell.paragraphs[0].runs[0].font.bold = True
        cell.paragraphs[0].runs[0].font.size = Pt(9)

    # Example data rows
    examples = [
        ("EmployeeNo",  "String", "30", "N", "",           "員工編號", ""),
        ("Name",        "String", "60", "Y", "",           "姓名",     ""),
        ("BirthDate",   "Date",   "",   "N", "",           "生日",     ""),
        ("Gender",      "List",   "1",  "N", "",           "性別",     "M=Male, F=Female"),
        ("C_BPartner_ID","ID",    "",   "N", "C_BPartner", "業務夥伴", ""),
    ]
    for r, row_data in enumerate(examples, start=1):
        for c, val in enumerate(row_data):
            col_table.rows[r].cells[c].text = val
            col_table.rows[r].cells[c].paragraphs[0].runs[0].font.size = Pt(9)

    doc.add_paragraph("")

    # ── Section 3: Repeating Sections ───────────────────────────
    doc.add_heading("3. Repeating Sections (Child Tables)", level=1)
    doc.add_paragraph(
        "If this table has repeating sections (e.g., education history, work experience), "
        "list them here. Each section becomes a separate child table."
    )

    rep_table = doc.add_table(rows=4, cols=2)
    rep_table.style = "Table Grid"
    rep_table.rows[0].cells[0].text = "Section Name (Chinese)"
    rep_table.rows[0].cells[1].text = "English Table Suffix"
    for cell in rep_table.rows[0].cells:
        cell.paragraphs[0].runs[0].font.bold = True

    path = os.path.join(ASSETS_DIR, "template.docx")
    doc.save(path)
    print(f"Created: {path}")
```

Also update `if __name__ == "__main__":` to call both:

```python
if __name__ == "__main__":
    os.makedirs(ASSETS_DIR, exist_ok=True)
    make_excel_template()
    make_word_template()
```

- [ ] **Step 3: Run the updated script**

```bash
python3 ~/sources/idempiere-table-creator/scripts/create_templates.py
```
Expected:
```
Created: /Users/ray/.claude/skills/idempiere-table-creator/assets/template.xlsx
Created: /Users/ray/.claude/skills/idempiere-table-creator/assets/template.docx
```

- [ ] **Step 4: Verify both assets exist**

```bash
ls -lh ~/.claude/skills/idempiere-table-creator/assets/
```
Expected: `template.docx` and `template.xlsx`, both > 5KB

---

## Task 5: Write `SKILL.md`

**Files:**
- Create: `~/.claude/skills/idempiere-table-creator/SKILL.md`

- [ ] **Step 1: Write SKILL.md**

Write to `~/.claude/skills/idempiere-table-creator/SKILL.md`:

```markdown
---
name: idempiere-table-creator
description: Use when creating iDempiere 12 database tables from existing business forms — invoked via /create-table with a .doc, .docx, .xls, or .xlsx file path; also triggers when user says "幫我建 iDempiere table", "generate table from form", or "convert form to SQL"
---

# iDempiere Table Creator

## Overview

Convert any business form (Word or Excel, any layout) into iDempiere 12 SQL DDL and Application Dictionary (`AD_Element`) sync statements, through a 4-phase interactive workflow.

**Data type rules and naming conventions:** See `references/idempiere-schema.md` in this skill directory.

## Invocation

```
/create-table <file-path>
```

Accepts: `.doc`, `.docx`, `.xls`, `.xlsx`

## Phase 1 — Form Analysis

Read the file. Based on extension:
- `.xlsx` / `.xls` → use Python with openpyxl to extract all non-empty cells with values
- `.docx` / `.doc` → extract text using available tools (python-docx, antiword, or file read)

Classify every identified text element into one of:

1. **Flat field** — a label adjacent to an input area (blank cell, blank line, input box, underline)
2. **Repeating section** — a named block (section header) followed by multiple blank template rows with the same column structure (indicators: ` 學 歷`, ` 經 歷`, `家庭狀況`; date-range patterns `年月 ~ 年月` repeated vertically)
3. **Ignore** — company name, form title, legal/privacy notices, instructions, signatures, page codes (e.g. `QR-AD-09-E`)

## Phase 2 — Propose Structure (REQUIRED before SQL)

**DO NOT generate SQL yet.**

First, ask the user for three inputs (one message):
1. **Prefix** — module prefix without underscore (e.g., `HR`, `CVG`)
2. **EntityType** — default `U`
3. **Is Doc Enabled** — `Y` or `N`

Then present the proposed table structure and **wait for confirmation**:

```
Table: <Prefix>_<InferredTableName>

Proposed columns:
  ColumnName       DataType      Source Label
  ----------       --------      ------------
  EmployeeNo       String(30)    員工編號
  Name             String(60)    姓名
  ...

[If repeating sections detected:]
Child tables:
  <Prefix>_<InferredTableName>_Education    (學歷)
  <Prefix>_<InferredTableName>_Experience   (經歷)

Confirm, or adjust (rename / retype / remove columns)?
```

Do not proceed until the user confirms.

## Phase 3 — SQL Generation

After confirmation, generate one `.sql` file per table.

**Output location:** Same directory as the input file.

**File header (always include at top of every generated file):**
```sql
-- REMINDER: Ensure AD_Sequence entries exist for each generated table before running this script.
-- Table: <TableName>
-- Source: <input-file-name>
-- Generated: <YYYY-MM-DD>
-- === SUMMARY ===
-- Tables: <list>
-- New AD_Element entries: <count>
-- Doc workflow: <enabled/disabled>
-- Warnings (if any): <list>
```

### Section 1: CREATE TABLE

Column order (mandatory columns first, then user columns, then doc workflow if enabled):

```sql
CREATE TABLE <TableName> (
    <TableName>_ID   NUMERIC(10,0)  NOT NULL,
    AD_Client_ID     NUMERIC(10,0)  NOT NULL,
    AD_Org_ID        NUMERIC(10,0)  NOT NULL,
    IsActive         CHAR(1)        NOT NULL DEFAULT 'Y',
    Created          TIMESTAMP      NOT NULL DEFAULT NOW(),
    CreatedBy        NUMERIC(10,0)  NOT NULL,
    Updated          TIMESTAMP      NOT NULL DEFAULT NOW(),
    UpdatedBy        NUMERIC(10,0)  NOT NULL,
    -- user-defined columns (from form analysis)
    ...,
    -- doc workflow (only if IsDocEnabled=Y)
    -- see references/idempiere-schema.md for full list
    CONSTRAINT <TableName>_PK PRIMARY KEY (<TableName>_ID)
);
```

For child tables, add after mandatory columns:
```sql
<ParentTable>_ID  NUMERIC(10,0)  NOT NULL,
```
And at end:
```sql
CONSTRAINT <ChildTable>_Parent FOREIGN KEY (<ParentTable>_ID) REFERENCES <ParentTable>(<ParentTable>_ID)
```

### Section 2: AD_Element Sync

For EACH column EXCEPT the known registry (see `references/idempiere-schema.md`):

```sql
DO $$ DECLARE v_id INTEGER; BEGIN
  IF NOT EXISTS (SELECT 1 FROM AD_Element WHERE LOWER(ColumnName) = LOWER('<ColumnName>')) THEN
    v_id := nextid('AD_Element', 'N');
    INSERT INTO AD_Element
      (AD_Element_ID, AD_Client_ID, AD_Org_ID, IsActive, Created, CreatedBy, Updated, UpdatedBy,
       ColumnName, EntityType, Name, PrintName)
    VALUES
      (v_id, 0, 0, 'Y', NOW(), 0, NOW(), 0,
       '<ColumnName>', '<EntityType>', '<English Name>', '<English PrintName>');
    INSERT INTO AD_Element_Trl
      (AD_Element_ID, AD_Language, AD_Client_ID, AD_Org_ID, IsActive, Created, CreatedBy,
       Updated, UpdatedBy, Name, PrintName, IsTranslated)
    VALUES
      (v_id, 'zh_TW', 0, 0, 'Y', NOW(), 0, NOW(), 0,
       '<zh_TW Name>', '<zh_TW PrintName>', '<Y or N>');
    -- IsTranslated='Y' if Chinese label from form; 'N' if English/inferred
    -- Description in Trl row: always NULL
  END IF;
END $$;
```

### Section 3: (no separate summary section — summary is in the file header)

## Phase 4 — Confirm Output

After generating all files, list what was created:
```
Generated:
  /path/to/<TableName>.sql
  /path/to/<TableName>_<Section>.sql   (child tables)
```
```

- [ ] **Step 2: Verify word count is reasonable**

```bash
wc -w ~/.claude/skills/idempiere-table-creator/SKILL.md
```
Expected: 400–700 words (skill files should be concise but complete)

---

## Task 6: Commit Initial Skill Files

**Files:** All files created in Tasks 1–5

- [ ] **Step 1: Check git status in skill dir (no git here — use project dir)**

```bash
cd ~/sources/idempiere-table-creator && git init && git add -A
```

- [ ] **Step 2: Commit**

```bash
cd ~/sources/idempiere-table-creator && git commit -m "feat: add idempiere-table-creator skill files

- SKILL.md: 4-phase form analysis → SQL generation workflow
- references/idempiere-schema.md: data type inference rules, naming conventions
- assets/template.xlsx: structured Excel input template
- assets/template.docx: structured Word input template
- scripts/create_templates.py: template generation script"
```

---

## Task 7: Baseline Test (RED — Verify Skill Is Needed)

Dispatch a subagent WITHOUT the skill loaded to test default Claude behavior on the same task. Document what happens.

- [ ] **Step 1: Dispatch baseline subagent**

Use the Agent tool with this prompt:

```
You are being asked to create an iDempiere 12 database table from an existing business form.

The user says: "Please run /create-table /Users/ray/Documents/QR-AD-09-E 員工資料卡.xlsx"

The database connection for reference is: localhost, db=dev, user=dev, password=dev

Do your best to complete this task. Generate the SQL DDL and any Application Dictionary entries needed.
```

- [ ] **Step 2: Document baseline behavior**

Note whether the subagent:
- [ ] Asked for Prefix, EntityType, IsDocEnabled before generating SQL
- [ ] Presented a proposed column mapping and waited for confirmation
- [ ] Used `nextid('AD_Element', 'N')` correctly (not `nextid('tablename')` or other form)
- [ ] Generated AD_Element_Trl with `zh_TW` for Chinese labels
- [ ] Added mandatory columns (AD_Client_ID, etc.)
- [ ] Included the AD_Sequence reminder comment

Expected: baseline agent likely skips interactive confirmation, uses wrong nextid syntax, or misses AD_Element sync entirely.

---

## Task 8: Verify Skill Works (GREEN)

Confirm the skill is in the correct location and test that Claude with the skill follows the 4-phase workflow.

- [ ] **Step 1: Verify skill file locations**

```bash
ls ~/.claude/skills/idempiere-table-creator/
ls ~/.claude/skills/idempiere-table-creator/references/
ls ~/.claude/skills/idempiere-table-creator/assets/
```
Expected: `SKILL.md`, `references/idempiere-schema.md`, `assets/template.xlsx`, `assets/template.docx`

- [ ] **Step 2: Dispatch verification subagent WITH skill context**

Use the Agent tool with this prompt (the agent will read the files itself):

```
You are a Claude Code assistant. Before starting, read these two files:
1. ~/.claude/skills/idempiere-table-creator/SKILL.md
2. ~/.claude/skills/idempiere-table-creator/references/idempiere-schema.md

Then follow the skill exactly for this request:
"/create-table /Users/ray/Documents/QR-AD-09-E 員工資料卡.xlsx"

Show each phase of your work.
Database for reference: localhost db=dev user=dev password=dev
```

- [ ] **Step 3: Verify skill compliance**

Confirm the subagent:
- [ ] Completed Phase 1: identified flat fields (姓名, 身分證字號, etc.) and repeating sections (學歷, 經歷, 家庭狀況)
- [ ] Completed Phase 2: asked for Prefix/EntityType/IsDocEnabled, presented column mapping, waited for confirmation
- [ ] Generated SQL with correct column order (mandatory columns first)
- [ ] Used `nextid('AD_Element', 'N')` correctly
- [ ] Created `zh_TW` Trl rows with `IsTranslated='Y'` for Chinese labels
- [ ] Included `-- REMINDER: Ensure AD_Sequence entries exist` at top of file

- [ ] **Step 4: If any check fails** — update `SKILL.md` to address the gap, re-run Step 2–3

---

## Task 9: Final Commit

- [ ] **Step 1: Stage and commit any adjustments from testing**

```bash
cd ~/sources/idempiere-table-creator
git add -A
git status
```

- [ ] **Step 2: Commit if changes exist**

```bash
git commit -m "fix: tighten SKILL.md based on baseline testing results"
```

- [ ] **Step 3: Verify final skill structure**

```bash
find ~/.claude/skills/idempiere-table-creator -type f | sort
```
Expected:
```
~/.claude/skills/idempiere-table-creator/SKILL.md
~/.claude/skills/idempiere-table-creator/assets/template.docx
~/.claude/skills/idempiere-table-creator/assets/template.xlsx
~/.claude/skills/idempiere-table-creator/references/idempiere-schema.md
```
