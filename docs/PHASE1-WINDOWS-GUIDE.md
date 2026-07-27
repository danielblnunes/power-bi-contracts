# Phase 1 — Windows Setup Guide (Excel → PBIP)

This is the **only step that must happen in Power BI Desktop on Windows**. It takes ~10 minutes. Once complete, sync the PBIP folder back to Linux and the MCP server handles all subsequent model and report authoring automatically.

## Prerequisites (one-time)

1. **Power BI Desktop June 2026+** installed on Windows.
2. Enable two preview options (only needed once):
   - **File → Options and settings → Options → Preview features**
   - Check **"Store reports using enhanced metadata format (PBIR)"**
   - Check **"Enable external tool access to Power BI Desktop through secure local APIs"** (for Desktop Bridge)
   - Click **OK**, restart Power BI Desktop.

## Step 1 — Connect to the Excel file

1. Open **Power BI Desktop** (close the splash / start dialog).
2. Click **Get data → Excel workbook**.
3. Navigate to `contracts_data_collection_template.xlsx` and click **Open**.
4. In the Navigator window, expand the file and check these three sheets ONLY:
   - `Agreements`
   - `Payment Schedule`
   - `Deliverables Rights`
5. **Do NOT click Load yet.** Click **Transform Data** to open Power Query.

## Step 2 — Clean and shape in Power Query

For each of the three queries, apply these transforms (in this order):

### 2.1 Agreements query
1. **Use first row as headers** (if not already applied).
2. Set data types for every column (click the icon in the column header):
   - `Agreement_ID`, `Agreement_Type`, `Counterparty_Name`, `Counterparty_Type`, `Agreement_Title`, `Agreement_Number`, `Contract_Category`, `Department_Owner`, `Legal_Owner`, `Commercial_Owner`, `Status`, `Seasons_Covered`, `Currency`, `Payment_Frequency`, `Renewal_Option`, `Auto_Renewal`, `Confidentiality_Level`, `Source_System`, `Document_Link`, `Next_Action`, `Notes` → **Text**
   - `Start_Date`, `End_Date`, `Last_Reviewed_Date`, `Next_Action_Date` → **Date**
   - `Term_Months`, `Notice_Period_Days` → **Whole Number**
   - `Total_Contract_Value_AED`, `Annual_Value_AED` → **Decimal Number** (or Fixed Decimal)
3. Rename the query to **`Agreements`**.
4. **Close & Apply** when done (or continue to step 2.2).

### 2.2 Payment Schedule query
1. **Use first row as headers**.
2. Set data types:
   - `Payment_ID`, `Agreement_ID`, `Season`, `Payment_Type`, `Milestone`, `Currency`, `Status`, `Notes` → **Text**
   - `Payment_Sequence`, `Days_To_Due` → **Whole Number**
   - `Due_Date`, `Invoice_Date`, `Payment_Date` → **Date**
   - `Amount_AED`, `Amount_Original`, `VAT_AED` → **Decimal Number**
3. Rename query to **`Payment Schedule`** (keep the space — DAX will quote it).

### 2.3 Deliverables Rights query
1. **Use first row as headers**.
2. **Critical data-quality fix**: the `Due_Date` column contains mixed dates and text (`N/A`, `90 days after order`). To make it reportable:
   - Right-click `Due_Date` → **Change Type → Using Locale → Date**. This will produce **`null`** for text values.
   - Keep the original text in a parallel column if needed (optional: **Add column → Custom column** `Due_Date_Text = Source[Due_Date]` before the type change).
3. Set data types:
   - `Item_ID`, `Agreement_ID`, `Item_Group`, `Item_Category`, `Item_Name`, `Description`, `Unit`, `Frequency`, `Season`, `Owner`, `Responsible_Party`, `Applicable`, `Status`, `Evidence_Link`, `Notes` → **Text**
   - `Due_Date`, `Date_Delivered` → **Date**
   - `Quantity` → **Whole Number**
   - `Value_AED` → **Decimal Number**
4. **Critical data-quality fix**: the `Status` column contains `600 Delivered` (a typo). Right-click the column → **Replace values**:
   - Find: `600 Delivered` → Replace with: `Delivered`
5. Rename query to **`Deliverables Rights`** (or shorter: `Deliverables`).

## Step 3 — Close & Apply

1. In Power Query Editor, click **Close & Apply** (top-left).
2. All three tables should now appear in the Data pane on the right.

## Step 4 — Create relationships

In Power BI Desktop:

1. Open the **Model view** (left-side icon, three-tables shape).
2. Verify Power BI auto-detected these relationships (it usually does by matching `Agreement_ID`):
   - `Agreements[Agreement_ID]` 1:M `Payment Schedule[Agreement_ID]` (single direction)
   - `Agreements[Agreement_ID]` 1:M `Deliverables Rights[Agreement_ID]` (single direction)
3. If missing, drag from `Agreements[Agreement_ID]` to the child's `Agreement_ID`.

**Do NOT create any Date relationships yet** — the MCP server will generate the Date table in Phase 2.

## Step 5 — Save as PBIP

1. Click **File → Save as**.
2. Change file type to **"Power BI Project (*.pbip)"**.
3. Save into the project workspace, e.g.:
   ```
   C:\path\to\powerbi-contract-uaepl\ContractManagement.pbip
   ```
4. Power BI Desktop creates a folder structure:
   ```
   ContractManagement.pbip
   ContractManagement.SemanticModel\
   ContractManagement.Report\
   ```

## Step 6 — Sync back to Linux

Copy these two folders (and the `.pbip` pointer file) into the Linux project directory `/home/dbln/code/powerbi-contract-uaepl/`:

```
powerbi-contract-uaepl/
└── ContractManagement.pbip
    ContractManagement.SemanticModel/   (entire folder)
    ContractManagement.Report/          (entire folder)
```

Sync options:
- **Git**: `git add ContractManagement.* && git commit && git push` (then `git pull` on Linux).
- **Network share / USB / rsync / OneDrive sync**.

## Step 7 — Tell the MCP agent

Once the `.pbip` is on Linux, ask in OpenCode:

> "Load the PBIP project at ./ContractManagement.pbip and build the contract management dashboard per the design in ./docs/DASHBOARD-DESIGN.md, adding all measures from ./measures/measures_catalog.py."

The MCP server will then automatically:
1. Load the project (`pbip_load_project`)
2. Generate the Date table (`pbip_create_date_table`)
3. Create Date relationships (`create_relationship`)
4. Add 25+ DAX measures (`pbip_add_measures` — validated + linted)
5. Build the three report pages with visuals (`pbir_add_page` + `pbir_add_visual`)
6. Validate everything (`pbir_validate_report`, `dax_lint`)
7. Hot-reload Desktop on Windows and screenshot (`bridge_reload`, `bridge_screenshot`)