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
| 科系 | Major |
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
<TableName>_UU   VARCHAR(36),
```

The `_UU` column is the UUID column (see `c_order_uu` in C_Order as reference). Rules:
- Type: `VARCHAR(36)` (not CHAR)
- Add a UNIQUE constraint: `CONSTRAINT <TableName>_UU_IDX UNIQUE (<TableName>_UU)`
- It is unique per table, so always generate a new AD_Element INSERT for it
- Use the column name itself as both Name and PrintName (e.g., `HR_EmployeeCard_UU`)
- zh_TW row: IsTranslated='N', Name/PrintName same as English

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
C_BPartner_ID, C_BPartner_Location_ID, AD_User_ID,
C_Activity_ID, C_Campaign_ID, C_Project_ID, C_CostCenter_ID,
SalesRep_ID, M_Warehouse_ID, M_PriceList_ID,
DateOrdered, DatePromised,
GrandTotal, TotalLines, FreightAmt, ChargeAmt,
POReference, DeliveryRule, InvoiceRule
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
