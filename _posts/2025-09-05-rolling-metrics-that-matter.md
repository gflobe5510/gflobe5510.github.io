---
layout: post
title: "Rolling Metrics that Matter: Turning Reporting into Decisions"
reading_time: "5–7 min read"
---

Most dashboards look great and change very little. That’s a problem. If a metric doesn’t *trigger a decision*, it’s just décor.

Below is a compact playbook I use to move from “nice charts” to **decision-ready analytics**—with SQL templates and a tiny Python check you can drop into your pipeline today.

---

## 1) Why rolling metrics beat snapshots

Static month-over-month (MoM) views are jumpy; year-over-year (YoY) can hide recent shifts. A **three-layer view** balances signal vs. noise:

1. **TTM (Trailing 12 Months)** – long-term trend and seasonality smoothing.  
2. **YoY** – structural change relative to the same season last year.  
3. **MoM** – recent movement and inflection detection.

Together, they answer:
- *Are we directionally healthy?* (TTM)  
- *Is this season better or worse than last year?* (YoY)  
- *Did something just change we should act on?* (MoM)

---

## 2) Metric design patterns (small but powerful)

- **Normalize the time axis**: store one row per month (`date_month` first day of month).  
- **Name metrics predictably**: `revenue`, `revenue_mom`, `revenue_yoy`, `revenue_ttm`.  
- **Guardrails**: nulls → 0 only if truly missing; otherwise keep null & flag upstream quality issues.  
- **Decision triggers** (examples—tune to your business):
  - `revenue_ttm` down **3 consecutive months** → review pipeline & pricing.
  - `revenue_mom < -5%` for **2 months** → freeze discretionary spend.
  - `revenue_yoy < -8%` **and** NPS falling → prioritize churn investigation.

---

## 3) SQL templates you can paste in

> **Dialect:** Postgres-style window functions. Works in BigQuery/Snowflake with minor tweaks.

Assumes a table `fact_revenue_monthly(date_month DATE, revenue NUMERIC)` with one row per month.

```sql
WITH m AS (
  SELECT
    date_month::date,
    revenue::numeric
  FROM fact_revenue_monthly
),
calc AS (
  SELECT
    date_month,
    revenue,

    /* MoM delta % */
    (revenue - LAG(revenue, 1) OVER (ORDER BY date_month))
      / NULLIF(LAG(revenue, 1) OVER (ORDER BY date_month), 0) AS revenue_mom,

    /* YoY delta % (same month last year) */
    (revenue - LAG(revenue, 12) OVER (ORDER BY date_month))
      / NULLIF(LAG(revenue, 12) OVER (ORDER BY date_month), 0) AS revenue_yoy,

    /* Trailing 12-month sum */
    SUM(revenue) OVER (
      ORDER BY date_month
      ROWS BETWEEN 11 PRECEDING AND CURRENT ROW
    ) AS revenue_ttm
  FROM m
)
SELECT *
FROM calc
ORDER BY date_month;

