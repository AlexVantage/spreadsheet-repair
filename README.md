# Sales Data Quality Pipeline

Portfolio sample for [AlexVantage](https://alexvantage.com) — a deliberately messy CRM export and a reusable Power Query pipeline organized around four data-quality pillars.

**Case study on the site:** [alexvantage.com/projects/sales-data-cleanup](https://alexvantage.com/projects/sales-data-cleanup)

## Files

| File | Purpose |
|------|---------|
| `dirty-sales-export.csv` | Raw export with intentional quality issues |
| `SalesDataCleanup.m` | Full M-language pipeline (copy into Advanced Editor) |
| `README.md` | Step-by-step walkthrough (this document) |

---

## Dirty data inventory

The sample CSV contains **11 rows** (10 unique orders + 1 duplicate). Each pillar maps to specific defects:

| Pillar | Column(s) | Issues baked in |
|--------|-----------|-----------------|
| **A — Ingestion** | (file-level) | External CSV meant to be connected, not pasted into a sheet |
| **B — Text & structure** | `Customer Name`, `City State Zip` | Extra spaces, inconsistent casing, compound location field |
| **C — Types & dates** | `Order Date`, `Revenue`, `Region` | European dates (DD/MM/YYYY), `$` + commas + spaces in revenue, `N/A` and blanks |
| **D — Dedupe & schema** | `Order ID`, `Internal System Log` | Duplicate `ORD-1001`, non-essential log column |

---

## Before you start

1. Copy `dirty-sales-export.csv` to a stable folder, e.g. `C:\Data\sales\`.
2. Open Excel → **Blank workbook**.
3. **File → Options → Data →** confirm your regional settings (this walkthrough assumes a **US-default** Excel install parsing dates incorrectly until locale is set).

---

## Pillar A — Data Ingestion & Connection

**Goal:** Connect to the external CSV by reference, not a one-time import. A parameter makes the path reusable across refreshes and environments.

### Steps (Excel UI)

1. **Data → Get Data → From Other Sources → Blank Query.**
2. **Home → Advanced Editor.** Replace contents with the `SourceFilePath` query from `SalesDataCleanup.m` (top block).
3. Update the path string to your copy of `dirty-sales-export.csv`.
4. **Home → Manage Parameters → New Parameter** is optional — marking `IsParameterQuery = true` in the `meta` record (as in the `.m` file) already exposes it under **Queries & Connections → Parameters**.
5. Rename the query **`SourceFilePath`**.

### Why this matters

Pasting CSV data into a worksheet creates a static snapshot. A parameterized `File.Contents` connection lets the team drop a new weekly export in the same folder, refresh, and get the full clean pipeline — no rework.

---

## Pillar B — Text & Structural Normalization

**Goal:** Standardize customer names and split the compound location field.

### Steps (Excel UI)

1. **Data → Get Data → From Other Sources → Blank Query** → rename **`SalesDataCleanup`**.
2. In Advanced Editor, start from the `Source` and `PromotedHeaders` steps in `SalesDataCleanup.m`, referencing `SourceFilePath`.
3. Select **`Customer Name`** → **Transform → Format → Trim**, then **Clean**, then **Capitalize Each Word**.
   - M equivalents: `Text.Trim` → `Text.Clean` → `Text.Proper`
4. Select **`City State Zip`** → **Transform → Split Column → By Delimiter** → comma → split into **3 columns** at each occurrence.
5. Rename new columns **`City`**, **`State`**, **`Zip`**.
6. Trim all three location columns (**Format → Trim** on each).

### Spot-check after Pillar B

| Order ID | Customer Name (before → after) | City State Zip → City / State / Zip |
|----------|--------------------------------|-------------------------------------|
| ORD-1001 | `  john SMITH  ` → `John Smith` | `Phoenix, AZ, 85001` → Phoenix / AZ / 85001 |
| ORD-1003 | `ROBERT  o'connor` → `Robert O'connor` | Tempe / AZ / 85281 |

---

## Pillar C — Logical Type Casting & Date Standardization

**Goal:** Parse European dates correctly, strip currency formatting from revenue, and convert sentinel strings to real nulls.

### Step 1 — Dates with locale

1. Select **`Order Date`**.
2. **Transform → Data Type → Using Locale…**
3. Type: **Date** · Locale: **English (United Kingdom)** → OK.

   M equivalent: `Table.TransformColumnTypes(..., {{"Order Date", type date}}, "en-GB")`

Without locale, `01/12/2023` on a US system becomes **January 12** instead of **December 1**. `22/07/2024` can fail entirely (month 22).

### Step 2 — Revenue: extract currency & normalize text

1. Select **`Revenue`**.
2. **Transform → Extract → Text Before Delimiter** is one option; for mixed formats, use **Replace Values** to remove `$`, then remove spaces and commas, OR paste the `CleanedRevenueText` step from `SalesDataCleanup.m`.
3. **Transform → Data Type → Currency** (or Decimal Number).

### Step 3 — Conditional null handler

1. Select **`Region`** → **Transform → Replace Values** → `N/A` → *(leave Value empty)* → OK.
2. Repeat for blank / single-space cells in **`Region`** and **`Revenue`**.
3. M approach (handles all sentinels in one pass): see `ReplacedRegion` and `ReplacedRevenue` in `SalesDataCleanup.m`.

### Spot-check after Pillar C

| Order ID | Order Date (correct) | Revenue (correct) | Region |
|----------|----------------------|-------------------|--------|
| ORD-1003 | 01/12/2023 → **2023-12-01** | `$2,450.75` → **2450.75** | South |
| ORD-1004 | null | null | East |
| ORD-1006 | 05/06/2024 → **2024-06-05** | null | null |

---

## Pillar D — Row/Column Deduplication and Schema Control

**Goal:** One row per order and a lean schema for downstream reporting.

### Steps (Excel UI)

1. Select **`Order ID`** → **Home → Remove Rows → Remove Duplicates** (keeps first occurrence).
   - M equivalent: `Table.Distinct(..., {"Order ID"})`
2. **Home → Choose Columns** → uncheck **`Internal System Log`** → OK.
   - M equivalent: `Table.SelectColumns(...)`
3. Reorder columns if needed: Order ID → Customer Name → City → State → Zip → Order Date → Revenue → Region.

### Final row count

**10 rows** (duplicate `ORD-1001` removed). **`Internal System Log`** excluded from output.

---

## Load & refresh

1. **Home → Close & Load To… → Table** (or **Only Create Connection** if feeding a data model).
2. Each week: replace the CSV in the source folder → **Data → Refresh All**.

---

## Expected clean output

| Order ID | Customer Name | City | State | Zip | Order Date | Revenue | Region |
|----------|---------------|------|-------|-----|------------|---------|--------|
| ORD-1001 | John Smith | Phoenix | AZ | 85001 | 2024-03-15 | 1250.50 | West |
| ORD-1002 | Jane Doe | Scottsdale | AZ | 85251 | 2024-07-22 | 890.00 | null |
| ORD-1003 | Robert O'connor | Tempe | AZ | 85281 | 2023-12-01 | 2450.75 | South |
| ORD-1004 | Mary-Jane Watson | null | null | null | null | null | East |
| ORD-1005 | Peter Parker | Mesa | AZ | 85203 | 2024-01-31 | 450.00 | West |
| ORD-1006 | Alicia Chen | Glendale | AZ | 85301 | 2024-06-05 | null | null |
| ORD-1007 | Tom Wilson | Chandler | AZ | 85224 | 2024-09-18 | 675.25 | West |
| ORD-1008 | Susan Lee | Peoria | AZ | 85345 | 2024-02-29 | 1125.00 | null |
| ORD-1009 | David Nguyen | Gilbert | AZ | 85234 | 2024-11-07 | 0.00 | South |
| ORD-1010 | Emily Ross | Surprise | AZ | 85374 | 2024-04-03 | null | West |

---

## Certification notes

- **PL-300 / DP-600:** Parameterized sources, locale-aware typing, text normalization, and schema shaping before load.
- **Portfolio:** Linked from the [Sales Data Cleanup](https://alexvantage.com/projects/sales-data-cleanup) case study on alexvantage.com.

---

## Related

- [AlexVantage](https://alexvantage.com) — data operations consultancy
- [Spreadsheet cleanup service](https://alexvantage.com/services)
