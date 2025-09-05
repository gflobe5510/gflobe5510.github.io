---

layout: post
title: "13‑Week Cash Flow Forecast: A Driver‑Based Playbook"
reading\_time: "7–9 min read"
---

A **13‑week cash flow** is the operating team’s heartbeat. Built well, it answers three questions every Monday:

1. Do we have enough cash to execute the plan?
2. If not, which levers change the outcome (collections, AP timing, spend)?
3. What decision needs to be made this week?

This post gives you a **driver‑based template** you can implement with SQL + a small Python what‑if. It reconciles to your ledgers, updates automatically, and produces a single weekly view leaders can act on.

---

## 1) Inputs & minimal schema

Use weekly grain (Monday start). Keep the model narrow but reliable:

* **Starting cash**: last bank balance snapshot (per entity if needed)
* **AR open**: invoices outstanding (`invoice_date`, `due_date`, `amount`, `customer_id`)
* **AR receipts**: historical cash receipts (`receipt_date`, `amount`, optional `invoice_id`)
* **AP open**: vendor bills (`bill_date`, `due_date`, `amount`, `vendor_id`)
* **AP payments**: historical disbursements (`payment_date`, `amount`)
* **Payroll calendar**: pay dates + gross cash out (`pay_date`, `amount`)
* **Fixed/recurring**: rent, debt service, SaaS, taxes

**Calendar helper (SQL)**

```sql
-- Postgres-style weekly calendar (Mon start)
CREATE OR REPLACE VIEW date_week AS
SELECT d::date AS dt,
       date_trunc('week', d)::date AS week_start
FROM generate_series('2024-01-01'::date, '2026-12-31'::date, interval '1 day') AS t(d);
```

---

## 2) Collections driver from AR aging

Compute a simple **collection curve** from history: what share of an invoice is typically collected in week 1, 2, 3… after issue? If you don’t have invoice‑level links, approximate from receipt timing.

**Aging buckets (SQL)**

```sql
-- Age invoices as of today
WITH ar AS (
  SELECT invoice_id, invoice_date::date, amount,
         GREATEST(0, (CURRENT_DATE - invoice_date::date)) AS age_days
  FROM ar_open
)
SELECT
  CASE
    WHEN age_days <= 30 THEN '0-30'
    WHEN age_days <= 60 THEN '31-60'
    WHEN age_days <= 90 THEN '61-90'
    ELSE '90+'
  END AS bucket,
  SUM(amount) AS balance
FROM ar
GROUP BY 1
ORDER BY 1;
```

**Rule of thumb schedule** (tune to your data):

* 0–30: collect **70%** next week, **25%** week+2, **5%** week+3
* 31–60: **40%**, **40%**, **20%** over next 3 weeks
* 61–90: **25%**, **35%**, **40%**
* 90+: flat **20%** per week until cleared (or treat as at‑risk)

Materialize this as a **distribution table** `ar_collect_share(bucket, week_offset, share)` and project receipts:

```sql
-- Project AR receipts by week using shares
WITH aged AS (
  SELECT invoice_id, amount,
         CASE WHEN CURRENT_DATE - invoice_date::date <= 30 THEN '0-30'
              WHEN CURRENT_DATE - invoice_date::date <= 60 THEN '31-60'
              WHEN CURRENT_DATE - invoice_date::date <= 90 THEN '61-90'
              ELSE '90+'
         END AS bucket
  FROM ar_open
), proj AS (
  SELECT
    (date_trunc('week', CURRENT_DATE)::date + (s.week_offset * 7)) AS week_start,
    SUM(a.amount * s.share) AS cash_in
  FROM aged a
  JOIN ar_collect_share s USING (bucket)
  GROUP BY 1
)
SELECT week_start, cash_in FROM proj;
```

> Have invoice‑level history? Replace the rule of thumb with a **learned curve**: compute empirical proportions by `week_offset` from issuance.

---

## 3) Disbursement drivers (AP, payroll, fixed)

**AP policy**: pay on **due date**, or use average actual days‑to‑pay.

```sql
-- AP cash out by week: pay on due_date (or add policy offset)
SELECT date_trunc('week', due_date)::date AS week_start,
       SUM(amount) AS cash_out_ap
FROM ap_open
GROUP BY 1;
```

**Payroll**: from calendar

```sql
SELECT date_trunc('week', pay_date)::date AS week_start,
       SUM(amount) AS cash_out_payroll
FROM payroll_calendar
GROUP BY 1;
```

**Fixed/recurring** (rent, debt service, SaaS)

```sql
SELECT date_trunc('week', due_date)::date AS week_start,
       SUM(amount) AS cash_out_fixed
FROM fixed_outflows
GROUP BY 1;
```

---

## 4) Assemble the 13‑week view

```sql
WITH horizon AS (
  SELECT generate_series(
           date_trunc('week', CURRENT_DATE)::date,
           date_trunc('week', CURRENT_DATE)::date + (12 * 7),
           interval '7 days')::date AS week_start
), inflow AS (
  -- replace with projected AR receipts query
  SELECT week_start, SUM(cash_in) AS cash_in FROM ar_receipts_projection GROUP BY 1
), outflow AS (
  SELECT week_start,
         COALESCE(SUM(cash_out_ap),0)       AS ap,
         COALESCE(SUM(cash_out_payroll),0)  AS payroll,
         COALESCE(SUM(cash_out_fixed),0)    AS fixed
  FROM (
    SELECT * FROM ap_cash_by_week
    UNION ALL SELECT * FROM payroll_cash_by_week
    UNION ALL SELECT * FROM fixed_cash_by_week
  ) x
  GROUP BY week_start
)
SELECT h.week_start,
       COALESCE(i.cash_in,0)                            AS cash_in,
       (o.ap + o.payroll + o.fixed)                     AS cash_out,
       COALESCE(i.cash_in,0) - (o.ap + o.payroll + o.fixed) AS net_flow
FROM horizon h
LEFT JOIN inflow  i USING (week_start)
LEFT JOIN outflow o USING (week_start)
ORDER BY h.week_start;
```

**Running cash** (add starting balance):

```sql
-- Suppose starting balance is in table bank_balance(start_week, cash_start)
WITH base AS (
  SELECT * FROM weekly_cash_forecast  -- previous SELECT
), start AS (
  SELECT cash_start FROM bank_balance WHERE start_week = (SELECT MIN(week_start) FROM base)
)
SELECT week_start,
       SUM(net_flow) OVER (ORDER BY week_start
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
       + (SELECT cash_start FROM start) AS projected_cash
FROM base;
```

---

## 5) What‑if scenarios in 20 lines of Python

```python
import pandas as pd

# df has columns: week_start, cash_in, cash_out
# knobs for scenarios
collections_delta = -0.10   # -10% collections
ap_delay_weeks    = 1       # push AP by 1 week
cut_fixed_percent = -0.05   # reduce fixed by 5%

f = df.copy()

# Collections sensitivity
f['cash_in_scn'] = f['cash_in'] * (1 + collections_delta)

# AP delay: shift a portion forward by weeks
ap = f['cash_out_ap'] if 'cash_out_ap' in f.columns else 0
if isinstance(ap, pd.Series):
    shifted = ap.shift(ap_delay_weeks, fill_value=0)
    f['cash_out_ap_scn'] = shifted
else:
    f['cash_out_ap_scn'] = 0

# Fixed cost change
f['cash_out_fixed_scn'] = f.get('cash_out_fixed', 0) * (1 + cut_fixed_percent)

# Combine outflows (payroll unchanged here)
f['cash_out_scn'] = (
    f.get('cash_out_payroll', 0) + f['cash_out_ap_scn'] + f['cash_out_fixed_scn']
)

f['net_flow_scn'] = f['cash_in_scn'] - f['cash_out_scn']

# Running cash with starting balance
start_cash = 1_250_000
f['projected_cash_scn'] = start_cash + f['net_flow_scn'].cumsum()

print(f[['week_start','cash_in_scn','cash_out_scn','projected_cash_scn']])
```

> Try presets: **Collections −10%**, **AP +1 week**, **fixed −5%**. You’ll see which lever keeps cash above your **minimum operating threshold**.

---

## 6) Dashboard that drives a decision

* **Top band**: current cash, min threshold, weeks until dip (if any)
* **Line chart**: projected cash vs. threshold; highlight breach weeks
* **Waterfall**: this week’s net flow (collections, AP, payroll, fixed)
* **Scenario toggles**: collections ±10%, AP timing ±1–2 weeks, fixed ±5–10%
* **Action log**: list decisions taken ("moved AP cycle", "pulled collections sprint") and the expected cash lift

---

## 7) Reconciliation & QA (don’t skip)

* **AR tie‑out**: sum of projected collections over horizon should be close to current AR balance minus bad‑debt allowance
* **AP tie‑out**: projected AP cash equals open AP due within horizon, adjusted for policy
* **Bank link**: last week’s projected cash ≈ actual bank balance (investigate gaps)
* **Refresh**: daily during crunch, weekly otherwise

---

### Wrap‑up

You don’t need a heavy model to manage liquidity. A simple, **driver‑based** 13‑week forecast—collections from aging, AP by policy, payroll & fixed calendar—creates a weekly rhythm where issues surface early and actions are clear. Materialize the SQL, wire a small scenario tool, and review it the same time every week.
