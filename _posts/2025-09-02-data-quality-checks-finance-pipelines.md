---

layout: post
title: "Data Quality Checks for Finance Pipelines: The 12 Guardrails"
reading\_time: "6–8 min read"
---

Bad data makes good dashboards useless. Finance needs numbers that are **fresh, complete, and trustworthy**—especially around close, board prep, and pricing decisions. This post lays out **12 pragmatic checks** you can add to any SQL/Python pipeline in a day, plus templates you can copy‑paste.

---

## 0) Define your SLOs (what "good" looks like)

Set explicit targets for each critical table/metric:

* **Freshness:** e.g., “< 2 hours behind source between 6am–8pm ET, weekdays.”
* **Completeness:** e.g., “No nulls in keys; ≤ 0.1% nulls in non‑critical fields.”
* **Uniqueness:** e.g., “No duplicate `txn_id` in `fact_orders`.”
* **Validity:** e.g., “Amounts within \[-\$10M, \$10M] and currency ∈ {USD, EUR, GBP}.”
* **Consistency:** e.g., “FX rates monotonic by date; totals reconcile to sub‑ledgers.”

Write these right next to the code that enforces them.

---

## 1) The 12 guardrails

**Schema & lineage**

1. **Schema drift** — new/missing columns vs. contract
2. **Type drift** — column type changes (e.g., `NUMERIC` → `TEXT`)

**Integrity**
3\. **Primary‑key uniqueness** — no duplicate IDs
4\. **Referential integrity** — foreign keys match dimensions

**Freshness & volume**
5\. **Freshness lag** — last timestamp vs. now
6\. **Row count bounds** — within expected daily/monthly ranges

**Completeness & validity**
7\. **Null ratio** — bounded null share for important fields
8\. **Range checks** — sane numeric ranges; no negative quantities where forbidden

**Consistency & reconciliation**
9\. **Double‑entry balance** — debits = credits (where applicable)
10\. **Sub‑ledger ties** — fact totals ↔ GL or system totals

**Behavioral**
11\. **Distribution drift** — share by segment/region within tolerance
12\. **Anomaly spike** — day/month change beyond threshold

Start with 1–2 per category on your most business‑critical tables.

---

## 2) SQL templates you can paste in

### A) Freshness lag (hours)

```sql
SELECT EXTRACT(EPOCH FROM (NOW() - MAX(loaded_at))) / 3600 AS hours_lag
FROM fact_orders;
```

**Fail** if `hours_lag > 2` during business hours.

### B) Duplicate primary keys

```sql
SELECT txn_id, COUNT(*)
FROM fact_orders
GROUP BY txn_id
HAVING COUNT(*) > 1;
```

### C) Referential integrity (unmapped dim keys)

```sql
SELECT DISTINCT f.customer_id
FROM fact_orders f
LEFT JOIN dim_customer d ON f.customer_id = d.customer_id
WHERE d.customer_id IS NULL;
```

### D) Null ratio bound

```sql
WITH c AS (
  SELECT COUNT(*)::numeric AS n,
         SUM(CASE WHEN order_ts IS NULL THEN 1 ELSE 0 END)::numeric AS n_null
  FROM fact_orders
)
SELECT (n_null / NULLIF(n,0)) AS null_share
FROM c;
```

**Fail** if `null_share > 0.001` (0.1%) on critical columns.

### E) Range check (amounts)

```sql
SELECT COUNT(*) AS n_out_of_range
FROM fact_orders
WHERE amount < -10000000 OR amount > 10000000;
```

### F) Row count bounds (seasonally aware)

```sql
WITH base AS (
  SELECT date_trunc('day', order_ts) AS d, COUNT(*) AS n
  FROM fact_orders
  WHERE order_ts >= NOW() - INTERVAL '120 days'
  GROUP BY 1
), stats AS (
  SELECT EXTRACT(DOW FROM d) AS dow,
         PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY n) AS med,
         PERCENTILE_CONT(0.1) WITHIN GROUP (ORDER BY n) AS p10,
         PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY n) AS p90
  FROM base
  GROUP BY 1
)
SELECT b.d, b.n, s.p10, s.p90
FROM base b
JOIN stats s ON EXTRACT(DOW FROM b.d) = s.dow
WHERE b.d = date_trunc('day', NOW())
  AND (b.n < s.p10 OR b.n > s.p90);
```

---

## 3) Lightweight Python checks (no fancy libs required)

```python
import pandas as pd
from datetime import datetime, timezone

# df: DataFrame for fact_orders

issues = []

# Freshness
lag_hours = (datetime.now(timezone.utc) - df['loaded_at'].max()).total_seconds()/3600
if lag_hours > 2:
    issues.append({"type": "freshness_lag_hours", "value": lag_hours})

# PK uniqueness
dups = df.duplicated('txn_id').sum()
if dups:
    issues.append({"type": "duplicate_txn_id", "count": int(dups)})

# Null ratio
null_share = df['order_ts'].isna().mean()
if null_share > 0.001:
    issues.append({"type": "order_ts_null_share", "value": float(null_share)})

# Range
oor = ((df['amount'] < -10_000_000) | (df['amount'] > 10_000_000)).sum()
if oor:
    issues.append({"type": "amount_out_of_range", "count": int(oor)})

print({"ok": len(issues) == 0, "issues": issues})
```

Persist these to a **run log** table for visibility over time.

---

## 4) Run‑log table schema (warehouse)

```sql
CREATE TABLE IF NOT EXISTS dq_run_log (
  run_id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  run_ts           TIMESTAMP DEFAULT NOW(),
  check_name       TEXT,
  table_name       TEXT,
  severity         TEXT,         -- info|warn|fail
  observed_value   TEXT,
  threshold        TEXT,
  is_violation     BOOLEAN,
  notes            TEXT
);
```

Append one row per check. If you adopt severities, you can alert only on **fail** and include **warn** in a weekly digest.

---

## 5) Alerts people won’t mute

* **Aggregate** to a single email/Slack message per day with a short table:

  * Check name, table, observed vs. threshold, owner, JIRA link.
* **Auto‑suppress known events** (e.g., month‑end backfill) with explicit windows.
* **Include a next step**: “`fact_orders` freshness lag > 2h — check upstream API job `etl_orders_v2`.”

---

## 6) CI/CD & scheduling

* **GitHub Actions**: run SQL checks with a CLI (psql/bq/snowflake) and Python checks nightly + on pull requests that touch pipeline code.
* **Fail the build** on schema/contract changes unless mapping + downstream code updated.
* **Dashboards**: expose a small **Data Health** panel (past 7 days of violations) on your executive page.

---

## 7) Rollout order (fast path)

1. Freshness + PK uniqueness on top 3 fact tables
2. Null ratio + range checks on 3 critical columns
3. Row bounds by weekday + referential integrity
4. Add drift/anomaly checks after a month of logs

You’ll catch 80% of costly problems with that sequence.

---

### Wrap‑up

Data quality isn’t a big‑bang project; it’s a **small set of guardrails** that keep decisions honest. Start with 2–3 checks per table, log everything, and review the violations weekly. You’ll spend far less time explaining weird numbers—and more time improving them.
