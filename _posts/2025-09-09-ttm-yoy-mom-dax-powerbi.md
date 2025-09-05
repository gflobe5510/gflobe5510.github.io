---

layout: post
title: "TTM, YoY, and MoM in Power BI with DAX: Decision‑Ready Metrics"
reading\_time: "7–9 min read"
---

A lot of teams track **month‑over‑month** and **year‑over‑year** changes, but they still struggle to see whether performance is *actually* improving. This post gives you a compact, production‑friendly setup in **Power BI** that yields three signals that leaders can act on:

* **TTM** (Trailing 12 Months) for true trend
* **YoY %** for structural change
* **MoM %** for fresh movement

You’ll get clean **DAX measures**, a reliable **Date** table, and a few guardrails to avoid misleading numbers.

---

## 1) Model prerequisites

**Star schema**: a fact table with a date column, related to a dedicated **Date** dimension.

Create the Date table (DAX):

```DAX
Date =
VAR StartDate = DATE(2019,1,1)
VAR EndDate   = TODAY()
RETURN
ADDCOLUMNS(
    CALENDAR(StartDate, EndDate),
    "Year", YEAR([Date]),
    "Month", FORMAT([Date], "MMM"),
    "YearMonth", FORMAT([Date], "YYYY-MM"),
    "MonthStart", DATE(YEAR([Date]), MONTH([Date]), 1),
    "MonthEnd", EOMONTH([Date], 0),
    "IsCurrentMonth", DATE(YEAR([Date]), MONTH([Date]), 1) = DATE(YEAR(TODAY()), MONTH(TODAY()), 1),
    "IsCompleteMonth", EOMONTH([Date], 0) < DATE(YEAR(TODAY()), MONTH(TODAY()), 1)
)
```

Then **Model view → Mark as date table** using `Date[Date]`.

Relationship: `Fact[date]` → `Date[Date]` (single, one‑to‑many, filter direction single).

---

## 2) Core measures (drop‑in)

Assume your fact table is `Fact` and revenue column is `Fact[Revenue]`.

```DAX
Total Revenue := SUM(Fact[Revenue])

Last Complete Date := EOMONTH(TODAY(), -1)

Revenue TTM (complete) :=
CALCULATE(
    [Total Revenue],
    DATESINPERIOD('Date'[Date], [Last Complete Date], -12, MONTH)
)

Revenue Last Year :=
CALCULATE(
    [Total Revenue],
    SAMEPERIODLASTYEAR('Date'[Date])
)

Revenue YoY % :=
VAR prev = [Revenue Last Year]
RETURN DIVIDE([Total Revenue] - prev, prev)

Revenue MoM % :=
VAR prev = CALCULATE([Total Revenue], DATEADD('Date'[Date], -1, MONTH))
RETURN DIVIDE([Total Revenue] - prev, prev)
```

> **Tip:** Use the **(complete)** versions for executive summaries to avoid partial‑month volatility. For exploratory visuals, you can also show a **current‑month‑to‑date** (MTD) variant.

---

## 3) Optional: MTD and fiscal calendars

If you need **MTD** alongside TTM:

```DAX
Revenue MTD := TOTALMTD([Total Revenue], 'Date'[Date])
```

For fiscal calendars, swap `SAMEPERIODLASTYEAR` for `DATEADD('Date'[Date], -12, MONTH)` and ensure your Date table has fiscal year columns.

---

## 4) Guardrails for honest numbers

**A. Exclude incomplete months in key KPIs**

```DAX
Revenue TTM (exclude current) :=
CALCULATE(
    [Total Revenue],
    DATESINPERIOD('Date'[Date], EOMONTH(TODAY(), -1), -12, MONTH)
)
```

**B. Hide unstable deltas when the denominator is tiny**

```DAX
Revenue YoY % (stable) :=
VAR prev = [Revenue Last Year]
RETURN IF(ABS(prev) < 1e-6, BLANK(), DIVIDE([Total Revenue] - prev, prev))
```

**C. Trigger flags you can badge in visuals**

```DAX
Revenue Alert Flag :=
VAR mom = [Revenue MoM %]
VAR yoy = [Revenue YoY %]
RETURN IF( mom < -0.05 && yoy < 0, 1, 0 )
```

---

## 5) Visuals that drive decisions

1. **Summary band (Cards):**

   * `Revenue TTM (complete)`
   * `Revenue YoY %`
   * `Revenue MoM %`

2. **Trend line:** Monthly `Total Revenue` with a faint `Revenue TTM (complete)` overlay. Add conditional markers where `Revenue Alert Flag = 1`.

3. **Comparatives:** Small multiples by segment/region (avoid nested slicers). Use the same three measures per panel for fast scanning.

4. **Explainer panel:** A short text box: *“What changed since last month?”* Pull in 2–3 driver deltas (e.g., price vs volume) via simple measures.

---

## 6) Troubleshooting & pitfalls

* **Not marking the Date table:** Time‑intelligence functions won’t behave. Always mark it.
* **Using columns instead of measures:** DAX time functions operate on filter context—build measures.
* **Partial month confusion:** Executives see swings—use the `Last Complete Date` technique for stable KPIs.
* **Multiple currencies:** Convert to a reporting currency in your fact or via a rate table + measures before time calcs.
* **Non‑additive metrics:** For ratios (AOV, margin%), compute components first, then divide in a measure; don’t average averages.
* **Performance:** Prefer measures like `SUM(Fact[Revenue])` over `SUMX` row iterators unless necessary; keep relationships simple.

---

## 7) Copy‑paste checklist

* [ ] Create and mark the **Date** table
* [ ] Link `Fact[date]` → `Date[Date]`
* [ ] Add the five measures: `Total Revenue`, `Revenue TTM (complete)`, `Revenue Last Year`, `Revenue YoY %`, `Revenue MoM %`
* [ ] Build summary cards + trend overlay
* [ ] Add the alert flag and show markers/badges
* [ ] QA with a table visual by month: Revenue, MoM %, YoY %, TTM

---

### Wrap‑up

With a clean Date table and five small measures, you can deliver **TTM/Yoy/MoM** that are stable, comparable, and decision‑ready. Start with the *complete* variants for leadership views, and expose MTD/diagnostic cuts only where they help the next step.
