# iDempiere Table Creator Skill — Design Spec

**Date:** 2026-03-24
**Status:** Approved

---

## Overview

A Claude Code skill (`/create-table`) that reads an existing free-format business form (Word `.doc`/`.docx` or Excel `.xls`/`.xlsx`) and generates iDempiere 12-compatible SQL DDL plus the required Application Dictionary entries.

The primary challenge is that input documents are real business forms with arbitrary layouts, Chinese field labels, and repeating sections — not structured database specs. The skill must guide Claude through intelligent form analysis before generating SQL.

---

## Skill Structure

```
~/.claude/skills/idempiere-table-creator/
├── SKILL.md                        ← Trigger + workflow instructions
├── references/
│   └── idempiere-schema.md         ← Data type inference rules, naming conventions, mandatory columns
└── assets/
    ├── template.xlsx               ← Optional structured input (alternative to free-form docs)
    └── template.docx               ← Optional structured input (alternative to free-form docs)
```

---

## Invocation

```
/create-table <file-path>
```

- `file-path`: path to any `.doc`, `.docx`, `.xls`, or `.xlsx` business form
- No fixed template structure required — Claude analyzes whatever document is provided

---

## Workflow (4 Phases)

### Phase 1 — Form Analysis

Claude reads the document and classifies content into:

1. **Flat fields** — simple label+value pairs → become table columns
   Examples: 姓名, 身分證字號, 生日, 電話, 員工編號, 職稱

2. **Repeating sections** — sections with multiple row instances → become child tables
   Examples: 學歷 (education history), 經歷 (work experience), 家庭狀況 (family)

3. **Ignored content** — form title, company name, instructions/help text, signatures, decorative cells

**Identification rules for repeating sections:**
- Section header followed by multiple blank rows with the same column structure
- Presence of date-range patterns (`年 月 ~ 年 月`) repeated

### Phase 2 — Propose Structure (Interactive)

Claude presents the proposed table design and waits for user confirmation before generating SQL.

**Required user inputs (via prompt after analysis):**
- **Prefix** — module prefix without underscore (e.g., `HR`, `CVG`)
- **EntityType** — default `U`; override for specific module
- **Is Doc Enabled** — `Y`/`N` — whether to add Document Workflow columns

**Proposed structure example:**
```
Table: HR_EmployeeCard
Prefix: HR  (user-defined)
EntityType: U
Is Doc Enabled: N

Proposed columns:
  EmployeeNo     String(30)   員工編號
  Name           String(60)   姓名
  Title          String(60)   職稱
  Gender         List(1)      性別  (□男 □女 pattern detected)
  IDNo           String(20)   身分證字號
  BirthDate      Date         生日
  Phone          String(20)   電話
  Mobile         String(20)   手機
  Address        String(255)  通訊地址
  HireDate       Date         到職日
  LeaveDate      Date         離職日

Repeating sections → child tables:
  HR_EmployeeCard_Education   (學歷: SchoolName, Department, StartDate, EndDate, GradStatus, DayNight)
  HR_EmployeeCard_Experience  (經歷: CompanyName, Title, StartDate, EndDate, MonthlySalary, LeaveReason)
  HR_EmployeeCard_Family      (家庭狀況: Relation, Name, BirthDate, Occupation)

Confirm / adjust?
```

User may request column renames, type changes, or removal before proceeding.

### Phase 3 — SQL Generation

After confirmation, Claude generates a single `.sql` output file (or one per table if multiple tables) with three sections:

#### Section 1: CREATE TABLE DDL

```sql
-- ============================================================
-- TABLE: HR_EmployeeCard
-- ============================================================
CREATE TABLE HR_EmployeeCard (
    HR_EmployeeCard_ID  NUMERIC(10,0)   NOT NULL,
    AD_Client_ID        NUMERIC(10,0)   NOT NULL,
    AD_Org_ID           NUMERIC(10,0)   NOT NULL,
    IsActive            CHAR(1)         NOT NULL DEFAULT 'Y',
    Created             TIMESTAMP       NOT NULL DEFAULT NOW(),
    CreatedBy           NUMERIC(10,0)   NOT NULL,
    Updated             TIMESTAMP       NOT NULL DEFAULT NOW(),
    UpdatedBy           NUMERIC(10,0)   NOT NULL,
    -- user-defined columns
    EmployeeNo          VARCHAR(30),
    Name                VARCHAR(60)     NOT NULL,
    ...
    CONSTRAINT HR_EmployeeCard_PK PRIMARY KEY (HR_EmployeeCard_ID)
);
```

**Auto-added mandatory columns** (never listed in input, always generated):
- `<TableName>_ID` — primary key, `NUMERIC(10,0)`
- `AD_Client_ID`, `AD_Org_ID` — tenant/org
- `IsActive CHAR(1) DEFAULT 'Y'`
- `Created`, `Updated` — `TIMESTAMP`
- `CreatedBy`, `UpdatedBy` — `NUMERIC(10,0)`

**Document Workflow columns** (added when `Is Doc Enabled = Y`):

| Column | Type | Notes |
|--------|------|-------|
| DocStatus | CHAR(2) | Default `'DR'` |
| DocAction | CHAR(2) | Default `'CO'` |
| Processed | CHAR(1) | Default `'N'` |
| Processing | CHAR(1) | Default `'N'` |
| DocumentNo | VARCHAR(30) | |
| C_DocType_ID | NUMERIC(10,0) | FK to C_DocType |
| C_DocTypeTarget_ID | NUMERIC(10,0) | FK to C_DocType |
| IsApproved | CHAR(1) | Optional |
| Posted | CHAR(1) | Optional |
| DateAcct | TIMESTAMP | Optional |
| C_Currency_ID | NUMERIC(10,0) | Optional |

Child tables include a FK back to the parent: `<ParentTable>_ID NUMERIC(10,0) NOT NULL`.

#### Section 2: AD_Element Sync

For each column, check if it already exists in `AD_Element` by `ColumnName` (case-insensitive). For missing entries:

```sql
-- ============================================================
-- AD_ELEMENT SYNC
-- ============================================================
DO $$ DECLARE v_id INTEGER; BEGIN
  IF NOT EXISTS (SELECT 1 FROM AD_Element WHERE LOWER(ColumnName) = LOWER('EmployeeNo')) THEN
    v_id := nextid('AD_Element', 'N');
    INSERT INTO AD_Element
      (AD_Element_ID, AD_Client_ID, AD_Org_ID, IsActive, Created, CreatedBy, Updated, UpdatedBy,
       ColumnName, EntityType, Name, PrintName)
    VALUES
      (v_id, 0, 0, 'Y', NOW(), 0, NOW(), 0, 'EmployeeNo', 'U', 'Employee No', 'Employee No');
    INSERT INTO AD_Element_Trl
      (AD_Element_ID, AD_Language, AD_Client_ID, AD_Org_ID, IsActive, Created, CreatedBy,
       Updated, UpdatedBy, Name, PrintName, IsTranslated)
    VALUES
      (v_id, 'zh_TW', 0, 0, 'Y', NOW(), 0, NOW(), 0, 'Employee No', 'Employee No', 'N');
  END IF;
END $$;
```

**Naming rules for `Name` / `PrintName`** (from `C_Order_AD_Element.csv` pattern):
- Expand camelCase/PascalCase with spaces: `EmployeeNo` → `Employee No`
- Known columns reuse existing AD_Element names (e.g., `DocStatus` → `Document Status` / `Doc Status`)
- `zh_TW` row: if the form field label is in Chinese (e.g., `員工編號`), use it as `Name`/`PrintName` with `IsTranslated='Y'`; if label is English or inferred, copy the English `Name`/`PrintName` with `IsTranslated='N'`. `Description` column is always left `NULL` in the Trl row.

#### Section 3: Validation Warnings

Emitted as SQL comments at top of file:
```sql
-- WARNING: Column 'IDNo' exceeds recommended 20-char limit for ID fields
-- INFO: DocWorkflow columns added (IsDocEnabled=Y)
-- INFO: 3 new AD_Element entries created
```

### Phase 4 — Output

- Output location: same directory as the input file
- File name: `<TableName>.sql` where `<TableName>` already includes the prefix (e.g., `HR_EmployeeCard.sql`)
- All three sections in one file per table
- Child table files named `<ParentTable>_<Section>.sql` (e.g., `HR_EmployeeCard_Education.sql`)

---

## Data Type Inference Rules

Documented in `references/idempiere-schema.md`. Key rules:

| Form pattern | iDempiere type | PostgreSQL DDL |
|---|---|---|
| `年月日` / `Date` in label | `Date` | `TIMESTAMP` |
| `年月 ~ 年月` range | Two `Date` columns (StartDate, EndDate) | `TIMESTAMP` |
| `□男 □女` / checkbox list | `List` | `CHAR(1)` |
| `電話` / `Phone` / `Mobile` | `String(20)` | `VARCHAR(20)` |
| `地址` / `Address` | `String(255)` | `VARCHAR(255)` |
| `金額` / `薪` / `Amount` | `Amount` | `NUMERIC(20,2)` |
| `編號` / `No` / short code | `String(30)` | `VARCHAR(30)` |
| `名稱` / `Name` | `String(60)` | `VARCHAR(60)` |
| `說明` / `Description` / `備註` | `String(255)` | `VARCHAR(255)` |
| `_ID` suffix | `ID` (FK) | `NUMERIC(10,0)` |
| `Is` prefix / `□` single checkbox | `YesNo` | `CHAR(1) DEFAULT 'N'` |

---

## Column Naming Convention

- **PascalCase** for all column names
- **No underscores** except for `_ID` FK suffix and standard iDempiere columns (`AD_Client_ID`, etc.)
- Max 30 characters (PostgreSQL + iDempiere limit)
- Chinese label → English name: translate semantically, not literally
  - 身分證字號 → `IDNo`
  - 到職日 → `HireDate`
  - 離職日 → `LeaveDate`
  - 畢/肄業 → `GradStatus`

---

## AD_Element Lookup

Before creating a new `AD_Element`, check the well-known column registry. Standard iDempiere columns (from `C_Order_AD_Element.csv`) already exist and must NOT be re-inserted:

Examples of already-existing elements: `AD_Client_ID`, `AD_Org_ID`, `IsActive`, `Created`, `CreatedBy`, `Updated`, `UpdatedBy`, `DocStatus`, `DocAction`, `Processed`, `Processing`, `DocumentNo`, `C_DocType_ID`, `C_DocTypeTarget_ID`, `IsApproved`, `Posted`, `DateAcct`, `C_Currency_ID`, `Name`, `Description`.

---

## `nextid` Function

```sql
-- Correct signature (iDempiere 12, PostgreSQL):
nextid('AD_Element', 'N')
--      ^ table name   ^ 'N' = user record, 'Y' = system record
```

Returns `INTEGER`. Use in a `DO $$ DECLARE v_id INTEGER` block.

---

## Skill Triggers (for SKILL.md)

- User invokes `/create-table`
- User says "幫我建 iDempiere table", "generate table from this form", "convert this form to SQL"
- User provides a `.doc`, `.docx`, `.xls`, or `.xlsx` file and asks to create a database table

---

## Out of Scope

- Application Dictionary registration beyond `AD_Element` (AD_Table, AD_Column registration is out of scope for v1)
- Migration XML / 2Pack generation
- iDempiere UI metadata (windows, tabs, fields)
- `AD_Sequence` entry creation — the `nextid('TableName', 'N')` function requires a corresponding `AD_Sequence` row to exist. Creating sequences is a manual DBA step and is not generated by this skill. The SQL output **always** includes a reminder comment at the top: `-- REMINDER: Ensure AD_Sequence entries exist for each generated table before running this script.`

