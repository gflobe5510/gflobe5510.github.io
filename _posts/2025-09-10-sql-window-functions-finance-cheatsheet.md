---

layout: post
title: "SQL Window Functions for Finance: A Practical Cheat Sheet"
reading\_time: "6–8 min read"
excerpt: "Drop‑in window function patterns for rolling sums, YoY/MoM deltas, percentiles, cohorts, and more—built for monthly finance data."
---

Window functions turn sprawling spreadsheets into **one clean SQL view** your BI tool can cache. This cheat sheet gives **copy‑paste patterns** for the metrics finance teams ask for every week—rolling totals, YoY/MoM deltas, contribution %, ranks, and cohort basics.

> Dialect: Postgres‑style examples. BigQuery/Snowflake work with minor syntax tweaks noted where relevant.

---

## 0) Refresher: `PARTITION BY` and `ORDER BY`

* `PARTITION BY` defines the **group** (e.g., product, region).
* `ORDER BY` defines the **time sequence** within each group.

```sql
-- Skeleton
<func>() OVER (
  PARTITION BY product_id
  ORDER BY date_month
  ROWS BETWEEN 11 PRECEDING AND CURRENT ROW  -- rolling 12
)
```

---

## 1) Rolling 12‑month revenue (TTM)

```sql
SELECT
  date_month,
  product_id,
  SUM(revenue) OVER (
    PARTITION BY product_id
    ORDER BY date_month
    ROWS BETWEEN 11 PRECEDING AND CURRENT ROW
  ) AS revenue_ttm
FROM fact_revenue_monthly;
```

**Notes**

* BigQuery/Snowflake: same syntax. Ensure `date_month` has one row per month per `product_id` (fill gaps).

---

## 2) MoM and YoY deltas

```sql
SELECT
  date_month,
  product_id,
  revenue,
  /* MoM % */
  (revenue - LAG(revenue, 1) OVER (PARTITION BY product_id ORDER BY date_month))
  / NULLIF(LAG(revenue, 1) OVER (PARTITION BY product_id ORDER BY date_month), 0) AS revenue_mom,

  /* YoY % */
  (revenue - LAG(revenue, 12) OVER (PARTITION BY product_id ORDER BY date_month))
  / NULLIF(LAG(revenue, 12) OVER (PARTITION BY product_id ORDER BY date_month), 0) AS revenue_yoy
FROM fact_revenue_monthly;
```

**Guardrail**: wrap with `CASE WHEN ABS(prev) < 1e-6 THEN NULL END` to hide noisy percent deltas.

---

## 3) Running totals & burndown vs. target

```sql
WITH base AS (
  SELECT date_month, revenue
  FROM fact_revenue_monthly
  WHERE date_month >= DATE '2025-01-01'
)
SELECT
  date_month,
  revenue,
  SUM(revenue) OVER (ORDER BY date_month) AS ytd_revenue,
  /* Linear monthly target example */
  (SUM(1000000.0) OVER ()) * EXTRACT(MONTH FROM date_month)/12.0 AS ytd_target,
  (SUM(revenue) OVER (ORDER BY date_month)) - ((SUM(1000000.0) OVER ()) * EXTRACT(MONTH FROM date_month)/12.0) AS gap
FROM base;
```

Swap the target formula for your cadence (seasonal targets, quarterly resets…).

---

## 4) Contribution % (share of total) and mix

```sql
SELECT
  date_month,
  segment,
  revenue,
  revenue / NULLIF(SUM(revenue) OVER (PARTITION BY date_month), 0) AS share_of_month
FROM fact_revenue_by_segment;
```

To analyze **mix shift**, compute shares for two periods and subtract (`share_t1 − share_t0`).

---

## 5) Top‑N within a segment (rank & filter)

```sql
SELECT * FROM (
  SELECT
    date_month,
    segment,
    product_id,
    revenue,
    RANK() OVER (PARTITION BY date_month, segment ORDER BY revenue DESC) AS rnk
  FROM fact_revenue_by_product
) x
WHERE rnk <= 5;
```

Use `DENSE_RANK` to avoid gaps after ties.

---

## 6) Percentiles (P90 lead time, etc.)

```sql
SELECT DISTINCT
  segment,
  PERCENTILE_DISC(0.90) WITHIN GROUP (ORDER BY lead_time_days)
    OVER (PARTITION BY segment) AS p90_lead_time
FROM fact_orders;
```

* BigQuery: `APPROX_QUANTILES(lead_time_days, 100)[OFFSET(90)]` (no window). Compute per segment using `GROUP BY segment`.
* Snowflake: `PERCENTILE_CONT`/`PERCENTILE_DISC` in a window or with `GROUP BY`.

---

## 7) Cumulative **distinct** customers (first‑purchase trick)

Counting distinct in a rolling window is expensive. Instead, compute each customer’s **first month**, then count how many firsts have occurred up to the current month.

```sql
WITH firsts AS (
  SELECT customer_id, MIN(date_month) AS first_month
  FROM fact_orders_monthly
  GROUP BY customer_id
)
SELECT
  m.date_month,
  COUNT(*) FILTER (WHERE f.first_month <= m.date_month) AS customers_cum
FROM (SELECT DISTINCT date_month FROM fact_orders_monthly) m
LEFT JOIN firsts f ON f.first_month <= m.date_month
GROUP BY m.date_month
ORDER BY m.date_month;
```

This performs well and gives you a clean **installed base** curve.

---

## 8) Simple cohort retention table (months‑since‑cohort)

```sql
WITH first_purchase AS (
  SELECT customer_id, MIN(date_month) AS cohort_month
  FROM fact_orders_monthly
  GROUP BY 1
), activity AS (
  SELECT
    fp.cohort_month,
    o.customer_id,
    (12 * EXTRACT(YEAR FROM AGE(o.date_month, fp.cohort_month)) +
         EXTRACT(MONTH FROM AGE(o.date_month, fp.cohort_month)))::int AS months_since,
    1 AS active
  FROM fact_orders_monthly o
  JOIN first_purchase fp USING (customer_id)
)
SELECT
  cohort_month,
  months_since,
  COUNT(DISTINCT customer_id) AS active_customers,
  COUNT(DISTINCT customer_id) * 1.0 / NULLIF(COUNT(DISTINCT customer_id) OVER (PARTITION BY cohort_month ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING), 0) AS retention_rate
FROM activity
GROUP BY 1,2
ORDER BY cohort_month, months_since;
```

Use this as a heatmap input in BI.

---

## 9) Rolling volatility (finance risk quick check)

```sql
SELECT
  date_month,
  /* rolling 6‑month standard deviation of log returns */
  STDDEV_SAMP( LN(revenue / NULLIF(LAG(revenue) OVER (ORDER BY date_month), 0)) )
    OVER (ORDER BY date_month ROWS BETWEEN 5 PRECEDING AND CURRENT ROW) AS vol_6m
FROM fact_revenue_monthly;
```

Monitor for structural changes (vol rising/falling).

---

## 10) Pitfalls & performance tips

* **Sparse months** → fill gaps per entity before rolling math.
* **`COUNT(DISTINCT)` in windows** → avoid; use first‑event tricks or materialize helpers.
* **Frame clauses** → prefer `ROWS BETWEEN` for strictly monthly grain; `RANGE` can surprise with duplicate timestamps.
* **Casting** → `NUMERIC` vs `FLOAT`: use `NUMERIC` for money; cast to `FLOAT` only for heavy stats if needed.
* **Materialize** hot views (dbt/table) so BI doesn’t compute windows on every refresh.

---

### Wrap‑up

Window functions let you express finance logic **declaratively** and repeatably. Start by materializing a monthly facts view, layer these patterns, and your dashboards will stay fast, consistent, and decision‑ready.
