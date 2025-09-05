---
layout: post
title: "Cohort LTV & Retention in SQL: A 30-Minute Build"
reading_time: "6–8 min read"
---


Most “LTV” charts are either oversimplified or impossible to reproduce. This walkthrough gives you a **clean, reproducible cohort table** that supports retention, LTV, and growth analytics—using nothing but SQL.

We’ll build:

* first-purchase cohorts,
* monthly retention curves,
* cohort LTV (cumulative revenue per acquired customer),
* and a tidy table your BI tool can cache.

> **Dialect:** Postgres-style SQL (works in BigQuery/Snowflake with tiny tweaks).

---

## 1) Minimal schema

Assume a transactional table:

```sql
-- One row per order
CREATE TABLE fact_orders (
  order_id        BIGINT PRIMARY KEY,
  customer_id     BIGINT NOT NULL,
  order_ts        TIMESTAMP NOT NULL,
  revenue         NUMERIC(12,2) NOT NULL
);
```

Normalize time to **first day of month** for stable cohort math:

```sql
-- Materialize a month-grain view/table
CREATE MATERIALIZED VIEW fact_orders_monthly AS
SELECT
  customer_id,
  date_trunc('month', order_ts)::date AS month,
  SUM(revenue) AS revenue
FROM fact_orders
GROUP BY 1,2;
```

---

## 2) First-purchase cohort & months-since-cohort

```sql
WITH first_purchase AS (
  SELECT
    customer_id,
    MIN(month) AS cohort_month
  FROM fact_orders_monthly
  GROUP BY 1
),
monthly AS (
  SELECT
    m.customer_id,
    m.month,
    m.revenue,
    fp.cohort_month,
    /* Month 0 = acquisition month */
    (12 * EXTRACT(YEAR FROM age(m.month, fp.cohort_month)) +
         EXTRACT(MONTH FROM age(m.month, fp.cohort_month)))::int AS months_since_cohort
  FROM fact_orders_monthly m
  JOIN first_purchase fp USING (customer_id)
)
SELECT * FROM monthly;
```

---

## 3) Cohort population & retention

```sql
WITH first_purchase AS (
  SELECT customer_id, MIN(month) AS cohort_month
  FROM fact_orders_monthly
  GROUP BY 1
),
cohort_pop AS (
  SELECT cohort_month, COUNT(*)::numeric AS cohort_size
  FROM first_purchase
  GROUP BY 1
),
activity AS (
  SELECT
    fp.cohort_month,
    m.months_since_cohort,
    COUNT(DISTINCT m.customer_id)::numeric AS active_customers
  FROM first_purchase fp
  JOIN fact_orders_monthly m
    ON m.customer_id = fp.customer_id
  GROUP BY 1,2
)
SELECT
  a.cohort_month,
  a.months_since_cohort,
  a.active_customers,
  cp.cohort_size,
  CASE WHEN cp.cohort_size > 0
       THEN a.active_customers / cp.cohort_size
       ELSE NULL END AS retention_rate
FROM activity a
JOIN cohort_pop cp USING (cohort_month)
ORDER BY cohort_month, months_since_cohort;
```

**What you get:** a classic retention heatmap input—row per cohort, column per `months_since_cohort`, values as `% active`.

---

## 4) Cohort LTV (cumulative revenue per acquired customer)

```sql
WITH first_purchase AS (
  SELECT customer_id, MIN(month) AS cohort_month
  FROM fact_orders_monthly
  GROUP BY 1
),
revenue_by_cohort AS (
  SELECT
    fp.cohort_month,
    (12 * EXTRACT(YEAR FROM age(m.month, fp.cohort_month)) +
         EXTRACT(MONTH FROM age(m.month, fp.cohort_month)))::int AS months_since_cohort,
    SUM(m.revenue) AS cohort_revenue
  FROM fact_orders_monthly m
  JOIN first_purchase fp USING (customer_id)
  GROUP BY 1,2
),
cohort_pop AS (
  SELECT cohort_month, COUNT(*)::numeric AS cohort_size
  FROM first_purchase
  GROUP BY 1
),
cum AS (
  SELECT
    r.cohort_month,
    r.months_since_cohort,
    SUM(r.cohort_revenue) OVER (
      PARTITION BY r.cohort_month
      ORDER BY r.months_since_cohort
      ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS revenue_cum
  FROM revenue_by_cohort r
)
SELECT
  c.cohort_month,
  c.months_since_cohort,
  (c.revenue_cum / NULLIF(cp.cohort_size,0))::numeric(12,2) AS ltv_per_customer
FROM cum c
JOIN cohort_pop cp USING (cohort_month)
ORDER BY cohort_month, months_since_cohort;
```

**Interpretation:** `ltv_per_customer` is the **average cumulative revenue** per originally acquired customer—perfect for comparing cohorts or tracking improvements.

---

## 5) Decision patterns (how to use this)

* **Payback window:** Find the first `months_since_cohort` where `ltv_per_customer ≥ cac`.
* **Cohort quality tracking:** Compare LTV curves for the last 3 acquisition months.
* **Retention slope:** Flattening after month 2? Trigger onboarding/activation experiments.
* **Pricing tests:** Segment cohorts by price/plan and compare LTV/retention paths.

---

## 6) QA & pitfalls

* **Duplicates:** Ensure one first purchase per customer. If you dedupe orders upstream, log counts.
* **Time zone:** Cohorts shift if ingestion mixes UTC/local—normalize at ingest.
* **Sparse months:** Don’t fill missing revenue with 0 unless you’re confident it means *no activity*; otherwise keep `NULL` and handle in BI.
* **Refunds/chargebacks:** Decide whether to subtract them in the same month or when processed—be consistent.
* **Seasonality:** When benchmarking cohorts, compare same `months_since_cohort` rather than calendar month.

---

## 7) Materialize for BI

For speed and reproducibility, materialize two views/tables your dashboards can hit directly:

```sql
CREATE MATERIALIZED VIEW coh_retention AS
-- (use the retention query above)
;

CREATE MATERIALIZED VIEW coh_ltv AS
-- (use the LTV query above)
;

-- Optional: refresh nightly
-- REFRESH MATERIALIZED VIEW CONCURRENTLY coh_retention;
-- REFRESH MATERIALIZED VIEW CONCURRENTLY coh_ltv;
```

**Indexes (Postgres):**

```sql
CREATE INDEX ON fact_orders_monthly (customer_id, month);
CREATE INDEX ON coh_retention (cohort_month, months_since_cohort);
CREATE INDEX ON coh_ltv (cohort_month, months_since_cohort);
```

---

### Wrap-up

This pattern takes \~30 minutes to implement and becomes a **foundation** for CAC payback, pricing tests, and onboarding improvements. The key is keeping the cohort math reproducible and the outputs **decision-ready**.

If you’d like the BigQuery/Snowflake variants or a demo dataset, email me at **[garthlobello1@gmail.com](mailto:garthlobello1@gmail.com)** and I’ll share a starter pack.
