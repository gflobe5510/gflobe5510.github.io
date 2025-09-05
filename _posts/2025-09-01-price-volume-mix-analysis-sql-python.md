---

layout: post
title: "Price–Volume–Mix (PVM) in SQL & Python: Explaining Revenue Changes"
reading\_time: "6–8 min read"
---

When revenue moves, leaders ask **why**: was it **price**, **volume**, or a **mix** shift? This post gives a clean, reproducible **Price–Volume–Mix (PVM)** approach you can implement with SQL (and an optional Python waterfall) that reconciles fully to the actual revenue change.

---

## 1) Setup & assumptions

We’ll work at **monthly** grain and any grouping level you care about (SKU, product, region, segment). You need a table with **quantity** and **revenue** so we can derive average **price**.

**Minimal schema** (examples):

```sql
-- fact_sales: one row per transaction or line item
-- Required fields used below: order_ts, product_id, quantity, revenue
CREATE TABLE fact_sales (
  order_id    BIGINT,
  order_ts    TIMESTAMP,
  product_id  TEXT,
  quantity    NUMERIC,
  revenue     NUMERIC
);
```

From this, we’ll create a monthly summary with `qty`, `rev`, and average `price` per entity:

```sql
CREATE OR REPLACE VIEW sales_monthly AS
SELECT
  date_trunc('month', order_ts)::date AS month,
  product_id,
  SUM(quantity) AS qty,
  SUM(revenue)  AS rev,
  CASE WHEN SUM(quantity) <> 0 THEN SUM(revenue) / SUM(quantity) END AS price
FROM fact_sales
GROUP BY 1,2;
```

---

## 2) Exact decomposition (Shapley-style average)

For each entity *i* between two periods **t₀** and **t₁**:

* `p0 = price_i(t₀)`
* `q0 = qty_i(t₀)`
* `p1 = price_i(t₁)`
* `q1 = qty_i(t₁)`

Total revenue change: `ΔRev = Σ_i (p1*q1 − p0*q0)`

A symmetric, widely used allocation (average of Laspeyres & Paasche a.k.a. **Shapley-like**) is:

* **Price effect**: `Σ 0.5 * (p1 − p0) * (q0 + q1)`
* **Volume effect**: `Σ 0.5 * (q1 − q0) * (p0 + p1)`
* **Mix effect**: `ΔRev − Price − Volume` (the cross-terms / composition change)

This sums **exactly** to ΔRev and is stable even when items appear/disappear (missing sides treated as 0).

---

## 3) Drop-in SQL (Postgres style)

Pick your months (change the dates):

```sql
WITH params AS (
  SELECT DATE '2025-08-01' AS t0, DATE '2025-09-01' AS t1
),
base AS (
  SELECT s.product_id, s.qty AS q0, s.price AS p0
  FROM sales_monthly s, params p
  WHERE s.month = p.t0
),
curr AS (
  SELECT s.product_id, s.qty AS q1, s.price AS p1
  FROM sales_monthly s, params p
  WHERE s.month = p.t1
),
joined AS (
  SELECT COALESCE(b.product_id, c.product_id) AS product_id,
         COALESCE(b.q0, 0) AS q0, COALESCE(b.p0, 0) AS p0,
         COALESCE(c.q1, 0) AS q1, COALESCE(c.p1, 0) AS p1
  FROM base b
  FULL OUTER JOIN curr c USING (product_id)
),
calc AS (
  SELECT
    product_id,
    (p1*q1 - p0*q0)                            AS delta_rev,
    0.5 * (p1 - p0) * (q0 + q1)                AS price_effect,
    0.5 * (q1 - q0) * (p0 + p1)                AS volume_effect
  FROM joined
)
SELECT
  product_id,
  delta_rev,
  price_effect,
  volume_effect,
  (delta_rev - price_effect - volume_effect)   AS mix_effect
FROM calc
ORDER BY delta_rev DESC;
```

**Totals check** (should equal actual ΔRev):

```sql
SELECT
  SUM(delta_rev)      AS total_delta_rev,
  SUM(price_effect)   AS total_price,
  SUM(volume_effect)  AS total_volume,
  SUM(delta_rev - price_effect - volume_effect) AS total_mix
FROM (
  -- paste the SELECT from the last query here (without ORDER BY)
) x;
```

> Tip: Swap `product_id` for `segment`/`region` to get mix at your preferred aggregation.

---

## 4) Optional Python: quick waterfall

```python
import pandas as pd
import matplotlib.pyplot as plt

# df has columns: ['price_effect','volume_effect','mix_effect']
# and you know baseline and current revenue (rev0, rev1)

steps = pd.Series({
    'Start (t0)': rev0,
    'Price': df['price_effect'].sum(),
    'Volume': df['volume_effect'].sum(),
    'Mix': df['mix_effect'].sum(),
    'End (t1)': rev1
})

running = steps.cumsum()
fig, ax = plt.subplots(figsize=(8,4))
colors = ['gray', 'green' if steps['Price']>=0 else 'red',
                   'green' if steps['Volume']>=0 else 'red',
                   'green' if steps['Mix']>=0 else 'red', 'gray']
ax.bar(range(len(steps)), steps.values, bottom=[0, running['Start (t0)'], running['Start (t0)']+steps['Price'], running['Start (t0)']+steps['Price']+steps['Volume'], 0])
ax.set_xticks(range(len(steps)))
ax.set_xticklabels(steps.index, rotation=0)
ax.set_title('Revenue Bridge: Price / Volume / Mix')
plt.tight_layout()
plt.show()
```

*(In BI tools, replicate the same with stacked bars or native waterfall visuals.)*

---

## 5) How to use the results

* **Pricing**: Large positive price effect with flat/negative volume suggests room for price tests.
* **Sales Ops**: Negative volume with positive mix → fewer units, but in higher-value segments; check pipeline and capacity.
* **Product/Portfolio**: Mix swings identify winners/laggards; consider assortment changes.
* **Forecasting sanity**: Reconcile forecast misses by attributing the gap to price/volume/mix.

---

## 6) Pitfalls & guardrails

* **Zeros / new items**: The average approach handles entries/exits; still review huge swings.
* **Returns/discounts**: Include them consistently in `revenue`; or separate a `discount` field and add a **discount effect**.
* **Unit consistency**: Don’t mix units (e.g., cases vs. units) within a group.
* **Currency**: Convert to a reporting currency before comparing months.
* **Sparse months**: If many zeros, consider quarterly grain for stability.

---

### Wrap‑up

PVM turns a confusing revenue swing into a simple **bridge** anyone can follow. Implement the SQL once, surface totals and top contributors on your dashboard, and you’ll speed up reviews from 30 minutes to 3.
