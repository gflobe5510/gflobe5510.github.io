---

layout: post
title: "Anomaly Detection for Finance: STL + Z‑Score (A Practical Playbook)"
reading\_time: "7–9 min read"
---

Monthly and daily financials swing for a lot of reasons—seasonality, promotions, reporting timing. The goal isn’t to explain *every* wiggle; it’s to **flag the few movements that deserve human attention**.

This post shows a lightweight pattern that’s production‑friendly: **STL decomposition** to separate trend/seasonality, plus a **robust Z‑score** on the residuals to flag anomalies. It’s fast to implement, easy to maintain, and pairs well with dashboards and weekly ops reviews.

---

## 1) Data prep (keep it boring and reliable)

Start with a single series per entity (e.g., total revenue, or revenue for a segment). Use a clean, regular time index.

**Assumptions**

* One row per period (daily or monthly)
* Columns: `date`, `value` (numeric)
* No duplicate dates; missing dates are explicitly filled (with `NULL` or 0 depending on policy)

**Quick SQL view for a monthly series**

```sql
-- Example for Postgres; adapt to your warehouse
CREATE OR REPLACE VIEW finance_monthly AS
SELECT
  date_trunc('month', dt)::date AS date_month,
  SUM(amount) AS value
FROM fact_finance
GROUP BY 1
ORDER BY 1;
```

---

## 2) STL + robust Z‑score in Python

> Uses `statsmodels` STL to remove seasonality, then flags outliers on the remainder via robust statistics (median & MAD).

```python
import pandas as pd
import numpy as np
from statsmodels.tsa.seasonal import STL

# df: two columns -> date, value
# Ensure regular frequency (monthly example)
df = df.copy()
df['date'] = pd.to_datetime(df['date'])
df = df.sort_values('date').set_index('date').asfreq('MS')  # month start

# Optional: simple forward-fill for small gaps (policy-dependent)
df['value'] = df['value'].interpolate(limit_direction='both')

# STL decomposition (season_length = 12 for monthly, 7 for daily)
stl = STL(df['value'], period=12, robust=True)
res = stl.fit()

# Residuals are value minus (trend + seasonal)
resid = res.resid.dropna()

# Robust Z-score using MAD (median absolute deviation)
median = resid.median()
mad = (resid - median).abs().median()
robust_z = 0.6745 * (resid - median) / (mad if mad != 0 else 1e-9)

# Flag anomalies
THRESH = 3.5  # tune per series; 3.0–4.0 common
flags = robust_z.abs() > THRESH

out = pd.DataFrame({
    'value': df['value'],
    'trend': res.trend,
    'seasonal': res.seasonal,
    'resid': res.resid,
    'robust_z': robust_z,
    'anomaly': flags
})

# Most recent status
latest = out.dropna().iloc[-1]
print({
    'date': str(latest.name.date()),
    'value': float(latest['value']),
    'robust_z': float(latest['robust_z']),
    'anomaly': bool(latest['anomaly'])
})
```

**Why MAD?** Standard deviation reacts to outliers; **MAD** (median absolute deviation) is stable even when you have a few extreme points.

---

## 3) Productionizing: materialize a tidy table

Store the decomposition and flags so your BI tool/dashboards can read them directly.

```sql
-- Example: materialize anomalies table (monthly)
CREATE TABLE IF NOT EXISTS finance_anomaly_monthly (
  date_month date primary key,
  value numeric,
  trend numeric,
  seasonal numeric,
  resid numeric,
  robust_z numeric,
  anomaly boolean
);

-- In your pipeline, replace/merge with the latest output from the Python job.
```

**Naming conventions**

* Use `<metric>_anomaly_<grain>` (e.g., `revenue_anomaly_monthly`)
* Columns: `date`, `value`, `trend`, `seasonal`, `resid`, `robust_z`, `anomaly`

---

## 4) Choosing thresholds (and living with them)

* Start with **3.5** for monthly; **4.0** for daily (noisier).
* For high‑variance entities, compute a **per‑entity threshold** using the interquartile range of residuals.
* Add a **persistence rule**: require the anomaly to persist for **2 consecutive periods** before alerting.

**Optional: dynamic threshold**

```python
iqr = np.subtract(*np.percentile(resid.dropna(), [75, 25]))
thresh = max(3.0, min(5.0, 1.5 + iqr))  # clamp between 3 and 5
```

---

## 5) Alerting that people won’t mute

Send a weekly email/Slack digest that groups anomalies by owner, not a noisy stream of one‑offs.

**Digest content**

* Metric & entity (e.g., `Revenue — Northeast`)
* Last value vs. expected (trend + seasonal)
* Robust Z‑score
* Suggested next step (see playbook below)

**Template (pseudo‑SQL)**

```sql
SELECT
  date_month,
  CASE WHEN anomaly THEN '⚠️' ELSE '' END AS flag,
  value,
  (trend + seasonal) AS expected,
  (value - (trend + seasonal)) AS gap,
  robust_z
FROM finance_anomaly_monthly
WHERE date_month >= date_trunc('month', current_date) - interval '6 months'
  AND anomaly = true
ORDER BY date_month DESC;
```

---

## 6) Investigations playbook (don’t reinvent it weekly)

When an anomaly fires, run a **consistent** drill‑down:

1. **Timing checks**: late postings? calendar/holiday effects?
2. **Mix shift**: segment/region/product shares changed?
3. **Price/discounts**: unit price and promo depth vs. prior periods.
4. **Volume drivers**: traffic, conversion, signups; capacity constraints.
5. **Data health**: late ETL, schema drift, duplicates.

Document the finding and link it to the anomaly record. Over time, you’ll have a map of **common causes** and typical actions.

---

## 7) Dashboard pattern (clean and actionable)

* **Top band**: last value, expected value, and gap (+/%), with a colored badge if `|Z| > threshold`.
* **Trend chart**: actuals line with faint trend+seasonal overlay; dot markers on anomalies.
* **Breakdowns**: small multiples by segment (avoid over‑reliance on dropdowns).
* **Anomaly log**: a table of the last 6–12 alerts with owner and status (open/resolved).

---

## 8) Common pitfalls

* **Over‑smoothing**: too aggressive decomposition hides genuine shifts—review residual plots.
* **Mixed frequencies**: daily + weekly close events = false positives; normalize frequency first.
* **Backfill surprises**: late data can flip yesterday’s flag; snapshot or version your inputs.
* **Policy drift**: if you change interpolation/thresholds, log the change with a date.

---

## 9) Variants you can try later

* **STL + EWMA residuals**: smooth residuals with an exponentially weighted mean before Z‑scoring.
* **Quantile regression**: estimate conditional medians/quantiles for asymmetric distributions.
* **Multivariate approach**: flag outliers in a feature space (e.g., PCA + distance) for richer contexts.

---

### Wrap‑up

Anomaly detection doesn’t need to be arcane. STL + robust Z‑score is **transparent, tunable, and operational**. Start with one metric, wire a weekly digest, and adopt the same investigation checklist every time. You’ll spend less time debating noise and more time acting on signals.
