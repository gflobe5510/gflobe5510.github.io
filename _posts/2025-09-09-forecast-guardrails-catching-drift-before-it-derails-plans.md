---

layout: post
title: "Forecast Guardrails: Catching Drift Before It Derails Plans"
reading\_time: "6–8 min read"
excerpt: "Rolling backtests, a MAPE‑based drift alert, and a cone view—so you notice change fast, assign an owner, and keep plans on track."
---

Forecasts rarely explode—they **drift**. Seasonality shifts, promo calendars change, a pipeline hiccups, and suddenly the last few months are off… but no one notices until planning goes sideways.

This mini‑project ships lightweight **forecast guardrails**: rolling backtests, a simple drift rule, and a cone chart. The goal isn’t a perfect model—it’s to *notice change fast* and trigger an owner‑led investigation.

> Stack: **Python** (pandas, matplotlib) for the backtest + alert; **Tableau/Power BI** for the cone + MAPE panel.
> Repo: [https://github.com/gflobe5510/forecast-guardrails](https://github.com/gflobe5510/forecast-guardrails)
> Image: the cone view below renders from the repo’s sample data.

<img
  src="https://raw.githubusercontent.com/gflobe5510/forecast-guardrails/main/assets/forecast_cone.png"
  alt="Forecast vs actuals cone; dashed line marks the forecast start."
  style="max-width:840px;width:100%;height:auto;display:block;margin:0.5rem auto 1rem;border-radius:12px;"
/>


---

## 0) What’s a “guardrail” here?

* A **rolling backtest** (walk‑forward) that reports error by period.
* A **drift rule**: flag when recent error (MAPE) exceeds a baseline by a margin.
* A **cone view**: forecast ± band next to actuals, so variance is visual—not a debate.
* A lightweight **runbook**: what to check and who owns it when an alert fires.

---

## 1) Data contract (tiny, explicit)

Monthly (or weekly) grain is perfect. I use this CSV shape:

```
date, actual, forecast, lower, upper
2023-01-01, 123.4, , ,
...
2024-12-01, , 152.2, 142.2, 162.2
```

*History rows* have `actual`. *Future rows* carry `forecast/lower/upper`. BI can plot both on one axis.

---

## 2) Rolling backtest (walk‑forward)

Walk‑forward with horizon=1 is enough to start. Replace the naive baseline with SARIMAX/Prophet later.

```python
from src.guardrails import rolling_backtest  # in the repo
import pandas as pd

df = pd.read_csv("data/sample_timeseries.csv", parse_dates=["date"])
series = df.set_index("date")["actual"].dropna()

bt = rolling_backtest(series, horizon=1, start=24)  # -> date, actual, forecast, mape
bt.head()
```

**Why this matters:** error by month reveals *when* things changed—not just a single average metric.

---

## 3) Drift rule (simple & explainable)

Compare the mean of the last **k** MAPE values with a trailing baseline:

```python
from src.guardrails import drift_alert

alert, stats = drift_alert(
    bt["mape"],
    recent_k=3,          # last 3 periods
    baseline_window=6,   # vs prior 6
    sigma=2.0            # mean + 2σ threshold
)
print(alert, stats)
```

Default rule: **alert if recent\_mean > baseline\_mean + 2σ**. Tune `recent_k`, `baseline_window`, and `sigma` based on your volatility. The point is transparency: anyone can read the rule and see why it fired.

> Tip: prefer **WAPE** if your data has many small denominators; this demo uses MAPE for simplicity.

---

## 4) Cone view (forecast vs actuals)

A single plot that says “We’re inside/outside expectations.” The helper masks history/future correctly and draws a split line.

```python
from src.guardrails import cone_plot
cone_plot(df, "assets/forecast_cone.png")
```

Use the image in your README and LinkedIn post.
**Alt text:** “Forecast vs actuals with future cone; dashed line marks the forecast start.”

---

## 5) BI spec (Tableau / Power BI)

**Visual 1 — Cone**

* **Lines:** Actual (history), Forecast (future only)
* **Band:** Lower–Upper (future only)
* **Vertical rule:** at the first forecast month

**Visual 2 — Error panel**

* Monthly **MAPE** from `bt`
* Overlay **threshold** (baseline\_mean + 2σ)
* If `alert==True`, show a red banner with owner + checklist

**Drill link:** clicking the banner opens your investigation doc.

---

## 6) Runbook (when alert fires)

1. **Data hygiene:** nulls, late loads, schema drift.
2. **Business context:** promos, pricing, holidays, one‑offs.
3. **Model sanity:** seasonality parameters, recent regime changes.
4. **Decision log:** owner + action + next review date.

That’s it—repeatable, auditable, and faster than arguing about models.

---

## 7) Results you can expect

*The demo uses synthetic data; in practice,* these guardrails help you:

* Move the convo from “Is the model good?” to **“What changed and who owns the fix?”**
* Detect **seasonal drift** and **data freshness** issues earlier.
* Shorten forecast reviews (I’ve seen \~20–30% time savings) with a common error panel.

---

## 8) Quick start (copy‑paste)

```bash
pip install -r requirements.txt
python demo.py
```

* Generates a rolling backtest and prints the drift stats
* Saves `assets/forecast_cone.png`
* Use `bt.to_csv("mape_by_month.csv", index=False)` for your BI error panel

---

## 9) Extend it (next 2 hours)

* Swap naive with **SARIMAX**/**Prophet**; keep the same backtest interface.
* Add **WAPE/SMAPE** and choose per‑series metric.
* Wire a **Slack/Email** nudge: “Drift > threshold. Owner: @ops‑lead. Link: dashboard.”
* Store **decisions** (date, owner, outcome) to learn over time.

---

## Links & repo

* Repo: [https://github.com/gflobe5510/forecast-guardrails](https://github.com/gflobe5510/forecast-guardrails)
* Image: `assets/forecast_cone.png` (in the repo)
* (Optional) Dashboard: add your Tableau Public or Power BI “Publish to web” URL here.

---

**Wrap‑up**

Guardrails don’t replace modeling—they **stabilize** it. With a tiny backtest loop, a plain‑English alert rule, and a cone chart, you’ll catch drift before it derails plans, and you’ll spend review time on **what changed** instead of **whether the forecast is “good.”**
