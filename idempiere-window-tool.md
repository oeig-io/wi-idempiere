---
name: idempiere-window-tool
description: Configure iDempiere windows, tabs, and field layouts using AD_Window, AD_Tab, and AD_Field tables with modern positioning controls
compatibility: opencode
metadata:
  type: tool
  original_file: idempiere-window-tool.md
  category: application-dictionary
  scope: idempiere
---

# iDempiere Window Tool

The purpose of this document is to describe how to configure iDempiere window layouts, tab structures, and field positioning.

This is important because proper window configuration ensures users can efficiently interact with data in both detail (form) and grid views.

## AD Structure

```
AD_Window (window definition)
    └── AD_Tab (tabs within window)
        └── AD_Field (fields within tab)
```

## Required Fields for Editable Tabs

The purpose of this section is to identify the `AD_Field` records every editable tab must have.

This is important because a tab missing these fields fails to save with a misleading **"Changes ignored"** message and persists no record — even though the underlying table, columns, and REST inserts all work correctly. This silent failure is easy to misdiagnose.

Every editable tab needs an `AD_Field` record for:

| Column | Why it is required | Typical visibility |
|--------|--------------------|--------------------|
| Key column (e.g. `ACME_Thing_ID`) | Framework binds the record key | Hidden (`IsDisplayed='N'`, `IsDisplayedGrid='N'`) |
| Parent link column (child tabs only) | Framework binds the parent key on save | Hidden (`IsDisplayed='N'`, `IsDisplayedGrid='N'`) |
| `AD_Client_ID` | Critical on every record | Read-only; detail only |
| `AD_Org_ID` | Critical on every record | Read-only; detail and grid |

> 💡 **Prefer the Create Fields process** (`AD_Tab` => Create Fields, process `AD_Tab_CreateFields`). It creates field records for all columns automatically and never omits the system fields above. Hand-crafting `AD_Field` rows via SQL is error-prone — if you must, include every row in the table above. The Sales Order => Order Line subtab is a good reference layout.

## Field Layout Concepts

### Positioning Fields (Modern Approach)

The modern field layout system uses three key controls:

| Field | Purpose | Values |
|-------|---------|--------|
| SeqNo | Vertical order (top to bottom) | 10, 20, 30... (must be unique per field) |
| XPosition | Horizontal start position | 1 = column 1, 4 = column 2, 7 = column 3 |
| ColumnSpan | Width in columns | 1 = compact (booleans), 2 = standard, 5+ = full width |

### Common Layout Patterns

iDempiere uses a **column-based coordinate system** where fields are positioned by grid coordinates. The system supports two and three-column layouts.

**Column Position Reference:**

| Logical Column | Text Field Position | Boolean Position(s) | Example Windows |
|---------------|-------------------|--------------------|-----------------|
| Column 1 | XPosition = 1 | XPosition = 2 | Product, Business Partner |
| Column 2 | XPosition = 4 | XPosition = 5, 6 | Product, Business Partner |
| Column 3 | XPosition = 7 | XPosition = 8, 9 | Business Partner |

**Two-Column Layout (fields side-by-side):**

Use Product window (ad_tab_id = 180) as reference.

```sql
-- Left column text field (UPC)
UPDATE ad_field SET seqno = 90, xposition = 1, columnspan = 2 WHERE ad_field_id = 1316;

-- Right column text field (SKU)  
UPDATE ad_field SET seqno = 91, xposition = 4, columnspan = 2 WHERE ad_field_id = 1317;
```

Result: Two fields share one line with labels on left and input boxes aligned.

**Three-Column Layout:**

Use Business Partner window (ad_tab_id = 220) as reference.

```sql
-- Column 1 (Search Key)
UPDATE ad_field SET seqno = 40, xposition = 1, columnspan = 2 WHERE ad_field_id = 2156;

-- Column 2 (Business Partner Group)
UPDATE ad_field SET seqno = 50, xposition = 4, columnspan = 2 WHERE ad_field_id = 3955;

-- Column 3 (Logo)
UPDATE ad_field SET seqno = 30, xposition = 7, columnspan = 2 WHERE ad_field_id = 57533;
```

**Full-Width Field:**

Wide fields span across all columns (Name, Description, etc.):

```sql
UPDATE ad_field SET seqno = 80, xposition = 1, columnspan = 5 WHERE ad_field_id = 2145;
```

**Boolean Field Positioning:**

Booleans have their label on the **right** side and use **ColumnSpan = 1** (compact). Position at XPosition + 1 relative to text field base:

| Adjacent Text Field | Boolean XPosition | Boolean ColumnSpan |
|--------------------|-------------------|--------------------|
| XPosition = 1 | **2** | 1 |
| XPosition = 4 | **5** (first) or **6** (second) | 1 |
| XPosition = 7 | **8** (first) or **9** (second) | 1 |

Example (Business Partner - Customer/Vendor checkboxes in column 3):

```sql
-- Customer checkbox (first boolean in column 3)
UPDATE ad_field SET seqno = 60, xposition = 8, columnspan = 1 WHERE ad_field_id = 9614;

-- Vendor checkbox (second boolean in column 3)
UPDATE ad_field SET seqno = 70, xposition = 9, columnspan = 1 WHERE ad_field_id = 9623;
```

Result: Checkboxes appear with labels aligned to the right of the control.

> **Important:** Always set SeqNoGrid when updating field positions. Omitting it leaves the grid column order undefined.

> **Never use:** `IsSameLine` - not processed by iDempiere web UI, use xposition offset instead.

**Button Field:**

Like booleans, buttons have no left-hand label (label is inside the button). Use the same positioning pattern as booleans.

### SeqNo Spacing Strategy

Standard iDempiere uses 10-unit gaps between fields (seqno = 10, 20, 30...). This provides only 9 slots for inserting new fields between existing ones, which quickly becomes limiting during complex layout modifications.

**Best Practice: Multiply by 10**

Before making layout changes on any window, multiply all existing seqnos by 10. This creates 100 slots between each field instead of 10:

```sql
-- Create spacious numbering (multiply ALL seqnos by 10)
-- 0 stays 0, 90 becomes 900, 100 becomes 1000, etc.
UPDATE ad_field SET seqno = seqno * 10 WHERE ad_tab_id = <tab_id>;
```

**Example transformation:**

| Before | After | Gap Available |
|--------|-------|---------------|
| 90 | 900 | 99 slots (901-999) |
| 100 | 1000 | 99 slots (1001-1099) |
| 210 | 2100 | 99 slots (2101-2199) |

**Leaving windows in ×10 state:**

It is completely acceptable to leave windows in the multiplied state permanently. iDempiere only cares about relative ordering, not absolute values. The ×10 state:
- Preserves visual ordering (900 < 910 < 920 maintains hierarchy)
- Provides room for future field insertions
- Requires no additional maintenance

**Updated Product window example with spacing:**

```sql
-- Step 1: Create spacious numbering
UPDATE ad_field SET seqno = seqno * 10 WHERE ad_tab_id = 180;

-- Step 2: Position fields using spacious values
UPDATE ad_field SET seqno = 900, xposition = 1, columnspan = 2 WHERE ad_field_id = 1316;  -- UPC
UPDATE ad_field SET seqno = 910, xposition = 4, columnspan = 2 WHERE ad_field_id = 1317;  -- SKU
UPDATE ad_field SET seqno = 920, xposition = 1, columnspan = 2 WHERE ad_field_id = 1034;  -- Product Category
```

### Grid vs Detail View

| View | Controlled By | Purpose |
|------|--------------|---------|
| Detail | SeqNo | Form layout order |
| Grid | SeqNoGrid | Column order in list view |

> 📝 **Note:** Grid and detail layouts are independent. Changing one does not affect the other.

## Zoom to Home Window from an Embedded Subtab

The purpose of this pattern is to give users a working zoom from a subtab whose table is the **primary** tab of a different window.

This is important because iDempiere's normal zoom works off a foreign-key field's lookup. When the subtab record *is* that table — e.g. `C_Invoice` embedded as a subtab under Business Partner Info, while `C_Invoice` is the primary tab of the Sales Invoice window — there is no field to zoom across, so the user is stranded in the cramped subtab with no way to open the full document.

The fix is to add a **Record ID pointer field** that renders an edit/zoom button. It needs two fields on the subtab, both defined as **virtual DB columns** (plain `ColumnSQL`, no `@SQL=`) so their values load with the tab query:

| Column | Reference | ColumnSQL | Purpose |
|--------|-----------|-----------|---------|
| `AD_Table_ID` | Integer (11) | the embedded table's `AD_Table_ID` (e.g. `318` for `C_Invoice`) | Sibling the editor reads for the target table |
| `OEIG_ZoomRecord_ID` | Record ID (200202) | the row's own key column (e.g. `C_Invoice.C_Invoice_ID`) | Renders the edit/zoom buttons |

- The editor pairs with its sibling **by column name**, so it must be named exactly `AD_Table_ID` — otherwise it throws `"AD_Table_ID field not found"`. The editor only reads that field's integer value, so **Integer (11)** is the lightest reference (no lookup); Search (30) or Table Direct (19) also work if you want it to display the table name instead of the raw id.
- **Keep the `AD_Table_ID` sibling hidden** (`IsDisplayed='N'` and `IsDisplayedGrid='N'`). The editor finds it by column name whether or not it is shown, so users never need to see the raw target-table id — only the zoom button matters.
- **Standardize on the column name `OEIG_ZoomRecord_ID`** with its `AD_Element` name (label) set to **`Zoom to Record`**. Reusing one shared column and element — only its `ColumnSQL` changes per tab (the row's own key column) — keeps the affordance consistent and avoids accidentally creating divergent copies of the same field.
- For a **UUID-keyed** target, use the **Record UUID** reference (200240) with the row's `TableName.TableName_UU` column instead.
- Clicking zoom calls `AEnv.zoom(AD_Table_ID, key)`, which resolves the target table's own primary window (the SO vs. PO variant is chosen per `IsSOTrx`) and filters to that record.
- **Core-table ids are stable** — hardcoding `AD_Table_ID` (e.g. `318` for `C_Invoice`) is safe. For **tables you create**, the id is assigned per instance, so the deploy script must look it up and substitute it into the `ColumnSQL`: `SELECT ad_table_id FROM ad_table WHERE tablename='...'`.

> 🔗 **Reference:** The Record ID / Record UUID editor mechanics, the `AD_Table_ID` sibling requirement, and the integer-vs-UUID key rules are documented under "Record ID / Record UUID References" in [idempiere-column-create-tool.md](idempiere-column-create-tool.md), which also covers virtual DB columns (plain `ColumnSQL`).

## SQL Patterns

### Query Current Field Layout

```sql
SELECT 
    f.ad_field_id,
    c.columnname,
    f.name,
    f.seqno,
    f.xposition,
    f.columnspan,
    f.seqnogrid
FROM ad_field f
JOIN ad_column c ON f.ad_column_id = c.ad_column_id
WHERE f.ad_tab_id = <tab_id>
  AND f.isactive = 'Y'
ORDER BY f.seqno;
```

### Update Field Position

```sql
-- Move field to new position
UPDATE ad_field 
SET seqno = <new_seqno>, 
    xposition = <x_pos>, 
    columnspan = <span>
WHERE ad_field_id = (
    SELECT f.ad_field_id 
    FROM ad_field f 
    JOIN ad_column c ON f.ad_column_id = c.ad_column_id 
    WHERE f.ad_tab_id = <tab_id> 
      AND c.columnname = '<column_name>'
);
```

## Common Lookups

**Find AD_Tab_ID:**
```sql
SELECT t.ad_tab_id, t.name, w.name as window_name
FROM ad_tab t 
JOIN ad_window w ON t.ad_window_id = w.ad_window_id
WHERE w.name ILIKE '%material receipt%'
  AND t.name ILIKE '%receipt line%';
```

**Find Field by Column Name:**
```sql
SELECT f.ad_field_id, c.columnname, f.name, f.seqno
FROM ad_field f
JOIN ad_column c ON f.ad_column_id = c.ad_column_id
WHERE f.ad_tab_id = <tab_id>
  AND c.columnname = 'ANS_PriceCost';
```

## Post-Changes

After modifying field layouts:
1. Reset cache: System Admin => Cache Reset
2. Log out and back in to see changes

## Reference

- See [idempiere-column-create-tool.md](idempiere-column-create-tool.md) for creating new columns and fields
- See [idempiere-application-dictionary-tool.md](idempiere-application-dictionary-tool.md) for AD table patterns

Tags: #tool #idempiere #window #field-layout
