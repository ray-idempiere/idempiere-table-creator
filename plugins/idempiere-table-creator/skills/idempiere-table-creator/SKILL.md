---
name: idempiere-table-creator
description: Use when creating iDempiere 12 database tables from existing business forms — invoked via /idempiere-table-creator with a .doc, .docx, or .xlsx file path; also triggers when user says "幫我建 iDempiere table", "generate table from form", or "convert form to SQL"
---

# iDempiere Table Creator

## Overview

Convert any business form (Word or Excel, any layout) into iDempiere 12 SQL DDL and Application Dictionary (`AD_Element`) sync statements, through a 4-phase interactive workflow.

**Data type rules and naming conventions:** See `references/idempiere-schema.md` in this skill directory.

## Invocation

```
/idempiere-table-creator <file-path>
```

Accepts: `.doc`, `.docx`, `.xlsx`

Note: Legacy .xls format is not supported. Ask user to save as .xlsx.

## Phase 1 — Form Analysis

Read the file. Based on extension:
- `.xlsx` → use Python with openpyxl to extract all non-empty cells with values
- `.docx` / `.doc` → extract text using available tools (python-docx, antiword, or file read)

Classify every identified text element into one of:

1. **Flat field** — a label adjacent to an input area (blank cell, blank line, input box, underline)
2. **Repeating section** — a named block (section header) followed by multiple blank template rows with the same column structure (indicators: ` 學 歷`, ` 經 歷`, `家庭狀況`; date-range patterns `年月 ~ 年月` repeated vertically)
3. **Ignore** — company name, form title, legal/privacy notices, instructions, signatures, page codes (e.g. `QR-AD-09-E`)

## Phase 2 — Propose Structure (REQUIRED before SQL)

**DO NOT generate SQL yet.**

**Step A — Ask for table parameters (send this message and stop):**

Ask for:
1. **Prefix** — module prefix without underscore (e.g., `HR`, `CVG`)
2. **EntityType** — default `U`
3. **Is Doc Enabled** — `Y` or `N`

Send the questions and wait for the user's reply. Do not guess or use placeholders.

**Step B — After receiving the reply, present the proposed column structure:**

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

**Wait for user confirmation before proceeding to Phase 3.**

## Phase 3 — SQL Generation

After confirmation, generate one `.sql` file per table.

**Output location:** Same directory as the input file.

**File header (always at top of every generated file):**
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

Column order:
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
    <TableName>_UU   VARCHAR(36),
    -- [insert confirmed user columns here, one per line]
    CONSTRAINT <TableName>_PK PRIMARY KEY (<TableName>_ID),
    CONSTRAINT <TableName>_UU_IDX UNIQUE (<TableName>_UU)
);
```

For child tables, add after mandatory columns:
```sql
<ParentTable>_ID  NUMERIC(10,0)  NOT NULL,
```
And FK constraint:
```sql
CONSTRAINT <ChildTable>_Parent FOREIGN KEY (<ParentTable>_ID) REFERENCES <ParentTable>(<ParentTable>_ID)
```

### Section 2: AD_Element Sync

For EACH column EXCEPT those in the Known AD_Element Registry (see `references/idempiere-schema.md`):

```sql
-- ============================================================
-- AD_ELEMENT SYNC
-- ============================================================
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
       '<zh_TW Name>', '<zh_TW PrintName>', '<Y_or_N>');
    -- IsTranslated='Y' if Chinese label from form was available; 'N' if English/inferred
    -- Description in Trl row: always NULL
  END IF;
END $$;
```

## Phase 4 — Confirm Output

After generating all files, list what was created:
```
Generated (file name = full table name including prefix):
  /path/to/HR_EmployeeCard.sql
  /path/to/HR_EmployeeCard_Education.sql   (if child tables exist)
```
