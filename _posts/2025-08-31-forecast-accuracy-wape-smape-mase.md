---

layout: post
title: "Forecast Accuracy Without the Gotchas: WAPE, sMAPE, and MASE"
reading\_time: "6–8 min read"
---

If you report **MAPE** every month and still get surprises, you’re not alone. MAPE is popular but brittle: it explodes near zeros, hides bias, and over‑penalizes under‑forecasting. This post gives you a practical toolkit—**WAPE**, **sMAPE**, **MASE**, and **Bias**—that’s robust for finance and ops.

---

## 1) The problem with MAPE (and when to avoid it)

**MAPE** = mean(|Actual − Forecast| / |Actual|). Issues:

* **Near‑zero actuals** → division blows up, small misses look catastrophic.
* **Asymmetry** → over‑ vs under‑forecast errors aren’t treated the same in many decision contexts.
* **Outlier sensitivity** → one tiny denominator can dominate the average.

**Use MAPE only** when actuals are comfortably non‑zero and on a similar scale across periods/segments. Otherwise, prefer the metrics below.

---

## 2) Better signal: WAPE, sMAPE, MASE, and Bias

### WAPE (a.k.a. MAE / sum of actuals)

$\text{WAPE} = \frac{\sum_t |A_t - F_t|}{\sum_t |A_t|}$

* **Scale‑free**, aggregation‑friendly, stable with zeros (if the whole sum isn’t zero).
* Intuition: average absolute error *as a share of* total actuals.

### sMAPE (symmetric MAPE)

$\text{sMAPE} = \n\frac{1}{n} \sum_t \frac{2\,|A_t - F_t|}{|A_t| + |F_t|}$

* Bounded in \[0, 2], less sensitive when either side is near zero.
* Caveat: there are **variants**; document the exact formula you use.

### MASE (Mean Absolute Scaled Error)

$\text{MASE} = \frac{\text{MAE}}{\text{MAE of a naive seasonal forecast}}$

* Compares your model to a simple seasonal naïve (e.g., last year same month).
* **< 1.0** means better than naïve; **> 1.0** means worse.

### Bias (signed % error)

$\text{Bias\%} = \frac{\sum_t (F_t - A_t)}{\sum_t |A_t|}$

* Direction matters: positive = over‑forecasting; negative = under‑forecasting.

---

## 3) Drop‑in SQL (Postgres style)

Assume a table `fact_forecast` with monthly grain:

```sql
-- Columns: date_month DATE, segment TEXT, actual NUMERIC, forecast NUMERIC
WITH base AS (
  SELECT
    date_month,
    segment,
    actual,
    forecast,
    ABS(actual - forecast) AS ae,
    CASE WHEN (ABS(actual) + ABS(forecast)) > 0
         THEN (2 * ABS(actual - forecast)) / (ABS(actual) + ABS(forecast))
         ELSE NULL END AS smape_component
  FROM fact_forecast
), agg AS (
  SELECT
    segment,
    SUM(ae)                     AS sum_ae,
    SUM(ABS(actual))            AS sum_abs_actual,
    AVG(smape_component)        AS smape,
    SUM(forecast - actual)      AS sum_signed_error
  FROM base
  GROUP BY segment
)
SELECT
  segment,
  CASE WHEN sum_abs_actual <> 0 THEN sum_ae / sum_abs_actual END AS wape,
  smape,
  CASE WHEN sum_abs_actual <> 0 THEN sum_signed_error / sum_abs_actual END AS bias_pct
FROM agg
ORDER BY segment;
```

**MASE in SQL** needs a baseline error from a naïve seasonal model. For monthly data, use a 12‑month lag:

```sql
WITH with_naive AS (
  SELECT
    f.date_month,
    f.segment,
    f.actual,
    f.forecast,
    LAG(f.actual, 12) OVER (PARTITION BY f.segment ORDER BY f.date_month) AS naive
  FROM fact_forecast f
), errors AS (
  SELECT
    segment,
    ABS(actual - forecast) AS ae,
    ABS(actual - naive)    AS ae_naive
  FROM with_naive
  WHERE naive IS NOT NULL
), agg AS (
  SELECT
    segment,
    AVG(ae)       AS mae,
    AVG(ae_naive) AS mae_naive
  FROM errors
  GROUP BY segment
)
SELECT segment,
       CASE WHEN mae_naive <> 0 THEN mae / mae_naive END AS mase
FROM agg;
```

---

## 4) Python helper (reusable functions)

```python
import numpy as np
import pandas as pd

def wape(y_true, y_pred):
    denom = np.abs(y_true).sum()
    return np.nan if denom == 0 else np.abs(y_true - y_pred).sum() / denom


def smape(y_true, y_pred):
    denom = (np.abs(y_true) + np.abs(y_pred))
    comp = np.where(denom == 0, np.nan, 2 * np.abs(y_true - y_pred) / denom)
    return np.nanmean(comp)


def mase(y_true, y_pred, seasonality=12):
    y_true = np.asarray(y_true)
    y_pred = np.asarray(y_pred)
    # Naive seasonal forecast
    naive = y_true[:-seasonality]
    target = y_true[seasonality:]
    mae_naive = np.mean(np.abs(target - naive))
    mae_model = np.mean(np.abs(y_true[seasonality:] - y_pred[seasonality:]))
    return np.nan if mae_naive == 0 else mae_model / mae_naive


def bias_pct(y_true, y_pred):
    denom = np.abs(y_true).sum()
    return np.nan if denom == 0 else (y_pred - y_true).sum() / denom

# Example usage with a monthly DataFrame per segment
# df columns: ['date_month','segment','actual','forecast']
seg = df[df['segment'] == 'Total'].sort_values('date_month')
print({
    'WAPE': wape(seg['actual'].values, seg['forecast'].values),
    'sMAPE': smape(seg['actual'].values, seg['forecast'].values),
    'MASE': mase(seg['actual'].values, seg['forecast'].values, seasonality=12),
    'Bias%': bias_pct(seg['actual'].values, seg['forecast'].values)
})
```

---

## 5) Dashboard pattern (what leaders see)

* **Top band**: WAPE, Bias%, MASE (optional: sMAPE). Keep the math consistent across pages.
* **Trend**: actual vs forecast with anomaly/bias badges.
* **Breakdowns**: small multiples by region/product with WAPE & Bias% labels.
* **QA panel**: latest 12 months of WAPE & Bias% in a simple table for sanity checks.

**Traffic lights (example thresholds)**

* WAPE < 8% = green; 8–15% = amber; > 15% = red (tune to your context).
* |Bias%| < 3% = green; sustained bias triggers a model review.

---

## 6) Operating rules (keep numbers honest)

* **Freeze windows**: lock the forecast snapshot before close to prevent silent rewrites.
* **Zeros policy**: document how you treat near‑zeros (aggregate or clamp) before calculating percentages.
* **One formula**: centralize metric code (dbt macro/Python utility) and reuse it everywhere.
* **Compare to naïve**: always include MASE or a naïve benchmark; it sets expectations.

---

### Wrap‑up

Swap brittle MAPE dashboards for a set of **robust, decision‑ready metrics**. With WAPE + Bias% for reporting and MASE for accountability, you’ll spend less time arguing about math and more time improving forecasts.
