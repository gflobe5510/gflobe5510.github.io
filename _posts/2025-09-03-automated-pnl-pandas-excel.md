---

layout: post
title: "Automated P&L from Raw Transactions with pandas + Excel"
reading\_time: "7–9 min read"
---

Most finance teams still copy/paste into spreadsheets. This post shows how to build a **repeatable monthly P&L** from raw transactions using **pandas** and export a clean, formatted **Excel** report—ready for leadership—without manual work.

You’ll get:

* a robust **chart‑of‑accounts (CoA) mapping** workflow,
* **data‑quality checks** that block bad inputs,
* monthly **P\&L and variance** tables,
* and a nicely formatted **Excel** workbook.

---

## 1) Inputs & folder structure

```
/code/pnl_pipeline.py
/data/transactions.csv            # raw GL or transactions
/data/coa_map.csv                 # account_id → line item mapping
/out/pnl_output.xlsx              # generated
```

**transactions.csv** (minimal):

```
txn_id,txn_ts,account_id,amount,currency,entity
1001,2025-07-03,4100,1299.50,USD,US
1002,2025-07-03,5100,-200.00,USD,US
...
```

**coa\_map.csv**:

```
account_id,line_item,section,sign
4100,Product Revenue,Revenue,1
4200,Service Revenue,Revenue,1
5100,COGS,COGS,-1
6100,S&M,Opex,-1
...
```

> `sign` lets you normalize signs (e.g., expenses negative, revenue positive) consistently.

---

## 2) Core pipeline (paste into `pnl_pipeline.py`)

```python
import pandas as pd
from pathlib import Path
from datetime import datetime
from openpyxl import load_workbook
from openpyxl.styles import Font, Alignment, PatternFill, Border, Side, NamedStyle

DATA_DIR = Path("data")
OUT_DIR = Path("out"); OUT_DIR.mkdir(exist_ok=True)

TXN_FILE = DATA_DIR / "transactions.csv"
MAP_FILE = DATA_DIR / "coa_map.csv"
OUT_XLSX = OUT_DIR / "pnl_output.xlsx"

# 1) Load
txn = pd.read_csv(TXN_FILE, parse_dates=["txn_ts"])
coa = pd.read_csv(MAP_FILE)

# 2) Basic schema checks
required_txn_cols = {"txn_id","txn_ts","account_id","amount"}
required_map_cols = {"account_id","line_item","section","sign"}
assert required_txn_cols.issubset(txn.columns), f"Missing cols in transactions: {required_txn_cols - set(txn.columns)}"
assert required_map_cols.issubset(coa.columns), f"Missing cols in coa_map: {required_map_cols - set(coa.columns)}"

# 3) Type/quality rules
if txn["amount"].isna().any():
    raise ValueError("Null amounts found in transactions")

# No duplicate txn_id
dup = txn.duplicated("txn_id")
if dup.any():
    raise ValueError(f"Duplicate txn_id(s): {txn.loc[dup,'txn_id'].tolist()[:5]} ...")

# 4) Normalize signs via CoA mapping & join
m = coa.set_index("account_id")
txn = txn.merge(coa, on="account_id", how="left")
if txn["line_item"].isna().any():
    missing = txn[txn["line_item"].isna()]["account_id"].unique().tolist()[:10]
    raise ValueError(f"Unmapped account_id(s) in coa_map: {missing}")

txn["norm_amount"] = txn["amount"] * txn["sign"].fillna(1)

# 5) Month grain & pivots
txn["month"] = txn["txn_ts"].dt.to_period("M").dt.to_timestamp()

pnl_month_line = (
    txn.groupby(["month","section","line_item"], dropna=False)["norm_amount"]
       .sum()
       .reset_index()
)

# Total by section per month
pnl_month_section = (
    txn.groupby(["month","section"], dropna=False)["norm_amount"]
       .sum()
       .reset_index()
)

# Grand totals by month (Gross Margin, EBITDA, etc. as needed)
# Example: Gross Margin = Revenue + COGS (COGS mapped with sign=-1)
section_pivot = pnl_month_section.pivot(index="month", columns="section", values="norm_amount").fillna(0)
if {"Revenue","COGS"}.issubset(section_pivot.columns):
    section_pivot["Gross Margin"] = section_pivot["Revenue"] + section_pivot["COGS"]

# 6) Variance table (MoM and YoY)
roll = section_pivot.sort_index().copy()
var = pd.DataFrame(index=roll.index)
for col in roll.columns:
    var[col+" MoM %"] = roll[col].pct_change()
    var[col+" YoY %"] = roll[col].pct_change(12)

# 7) Write Excel with formatting
with pd.ExcelWriter(OUT_XLSX, engine="openpyxl") as xw:
    pnl_month_line.to_excel(xw, sheet_name="P&L by Line", index=False)
    section_pivot.reset_index().to_excel(xw, sheet_name="P&L by Section", index=False)
    var.reset_index().to_excel(xw, sheet_name="Variance", index=False)

# Light formatting
wb = load_workbook(OUT_XLSX)
thin = Side(style="thin", color="44546A")
border_all = Border(left=thin, right=thin, top=thin, bottom=thin)

for wsname in ["P&L by Line","P&L by Section","Variance"]:
    ws = wb[wsname]
    ws.freeze_panes = "A2"
    # Header style
    for cell in ws[1]:
        cell.font = Font(bold=True)
        cell.fill = PatternFill("solid", fgColor="1F4E79")
        cell.alignment = Alignment(horizontal="center")
        cell.font = Font(color="FFFFFF", bold=True)
    # Borders + number format
    for row in ws.iter_rows(min_row=2, max_row=ws.max_row, max_col=ws.max_column):
        for cell in row:
            cell.border = border_all
            if isinstance(cell.value, (int,float)):
                cell.number_format = "#,##0.00;[Red]-#,##0.00"

wb.save(OUT_XLSX)
print(f"✅ P&L written to {OUT_XLSX}")
```

Run it:

```bash
python code/pnl_pipeline.py
```

---

## 3) Decision‑ready outputs

You now have three tabs:

* **P\&L by Line** — monthly values by `Revenue / COGS / Opex` line items.
* **P\&L by Section** — monthly totals by section with **Gross Margin**.
* **Variance** — MoM% and YoY% by section for fast scanning.

Wire this file into email/SharePoint or import into **Power BI/Tableau** for distribution.

---

## 4) Optional: warehouse helpers

Materialize a monthly grain view in SQL so pandas loads less and stays consistent:

```sql
CREATE OR REPLACE VIEW finance_monthly AS
SELECT
  date_trunc('month', txn_ts)::date AS month,
  t.account_id,
  SUM(amount) AS amount
FROM raw_transactions t
GROUP BY 1,2;
```

Or push **pnl\_month\_line** back to the warehouse for BI:

```sql
CREATE TABLE IF NOT EXISTS pnl_month_line (
  month date,
  section text,
  line_item text,
  value numeric
);
-- Use your ETL to MERGE/REPLACE from the DataFrame export
```

---

## 5) Quality gates (don’t skip)

* **Unmapped accounts** → hard fail with a clear list (fix `coa_map.csv`).
* **Duplicate txn\_id** → fail to avoid double counting.
* **Currency** → convert up front; store a normalized reporting currency column.
* **Entity tags** → include `entity` and aggregate both consolidated and per‑entity.
* **Audit trail** → write a small `run_log.csv` with row counts and totals per run.

---

## 6) Scheduling & versioning

* **Schedule** with cron/GitHub Actions (weekly or daily after posting).
* **Version mapping** — keep dated snapshots of `coa_map.csv` so historical P\&L doesn’t shift when mappings change.
* **Unit tests** — tiny pytest checks for mapping coverage (`SELECT DISTINCT account_id` vs. map).

---

## 7) Variants you can add later

* **Department/Cost center P\&L** — add columns and slicers.
* **Budget vs. Actual** — join a budget table and output variance.
* **Drill‑downs** — write a second sheet with top N drivers per section.
* **Export to CSV** for ingestion by a Google Sheet that stakeholders can annotate.

---

### Wrap‑up

A reliable P\&L pipeline is mostly **clean mapping + basic QC**. With a small pandas script and a mapping file, you can move from manual spreadsheets to a repeatable process—and focus reviews on **decisions**, not reconciliation.
