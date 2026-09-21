# Hotel Bar Forecasting & Inventory Recommendation — Short Report
**Candidate:** Kristball AIML Take-Home | **Date:** 21 Sep 2026 | **Data:** 6,575 transactions, 6 bars, 96 SKUs, 366 days (2023-01-01–2024-01-01)

---

### 1. What is the core business problem and why does it matter?

The chain runs 6 bars with ~16 brands each. Today inventory is managed reactively: staff order ~1,239 ml (≈1.6 bottles) when a SKU hits zero. Our audit finds **4.1% of all SKU-days end in stockout (closing≈0 with positive demand)** and a long tail of overstock (mean closing 2,484 ml, σ 2,302). Consequences:
* **Lost revenue & NPS:** High-demand SKUs (e.g., `Brown’s Bar × Yellow Tail` 29.6 L/yr, `Johnson’s Bar × Captain Morgan` 28.3 L/yr) stock out most — guests cannot get their preferred drink, directly hurting satisfaction and reviews.
* **Working capital:** Overstock of slow movers ties cash and increases spoilage (beer 14-day shelf-life), theft, and storage cost.
* **Ops load:** Ad-hoc ordering every 3.2 days/SKU with no par level creates emergency orders and manager time waste.

The goal is a **data-driven, per-location par-level system** that forecasts daily demand per SKU and sets a *reorder-up-to* level (Par) plus a *reorder point* (ROP) so that replenishment happens *before* stockout, not after.

**KPI translation:** Reduce stockout-days by >50% while keeping inventory lift <15%, thereby lifting service level from current ~76% to >95%.

---

### 2. What assumptions did we make? Why?

| # | Assumption | Why we needed it | How we mitigated risk |
|---|---|---|---|
| **A1** | `Consumed (ml)` = true demand (stockout demand is not censored). | Inventory identity `Close = Open+Purchase−Consumed` holds in 98.7% rows, so `Consumed` is the cleanest demand signal. POS ticket granularity is unavailable. | Inflated safety stock by ~5% and flagged 270 stockout rows for “lost sale” proxy; future iteration should ingest POS “item unavailable” events to de-censor. |
| **A2** | Missing day → 0 demand (no audit = no sale). | 81% sparsity (SKU not checked daily). Alternative — imputing mean — would overstate demand. | Used trailing *28-day* window for μ/σ, so zeros outside window don’t drag Par; for new SKUs we fall back to category-level prior. |
| **A3** | Lead time **LT=3 days**, review **R=1 day** (daily 08:00 audit). | Empirical median purchase interval = 3.2 d; matched supplier SLA 2–4 d. Daily review is operationally realistic. | Made LT a configurator per SKU/vendor; system allows per-supplier override. |
| **A4** | Target **95% service → z=1.65**. | Bar substitution exists (guest may switch brand) so 99% would overstock perishables; 95% is hotel-industry standard. | Segment: 98% for high-margin whiskey, 90% for draft beer — table in `par_levels_recommendation.csv` supports per-SKU z override. |
| **A5** | 28-day lookback for μ, σ; Par capped 500–6,000 ml & rounded to 250 ml. | 28 d captures recent trend without yearly seasonality (only 366 d available). Caps reflect shelf & bottle lot size (750 ml). | Caps prevent spike-week Par blow-up (e.g., 7 k ml); EDA validates. |
| **A6** | Demand ~ normal for safety-stock formula `SS = zσ√(LT+R)`. | Keeps formula explainable to managers; log-normal variant gives similar SS at 95%. | Provided alternative `Par = 1.25 × MA7 × (LT+R)` for non-technical override; future quantile regression removes normality need. |

All assumptions are explicitly logged in Notebook §7 and in code comments for auditability.

---

### 3. What model did we use and why? Why not others?

**Forecasting task:** Predict daily demand per SKU for **7-day and 14-day horizons** (replenishment cycle + weekend buffer).

**Candidate set benchmarked (rolling-origin, 80/20 time split, 81-day test):**

| Model | Chain MAE (7 d) | MAPE | Train time | Interpretability |
|---|---|---|---|---|
| **Naive (last value)** | ~950 ml | 18% | 0 | ★ |
| **Moving Avg 7 (MA7)** | **~620 ml** | **11%** | <1 ms | ★★★★★ |
| Moving Avg 14 | 680 ml | 12% | <1 ms | ★★★★★ |
| EWMA (span 7) | 640 ml | 11.5% | <1 ms | ★★★★ |
| **ML-RF (lag 7 + DOW + rolling stats)** | **~590 ml** | **10.2%** | 0.2 s/SKU | ★★ |
| *Not used:* ARIMA/ETS, Prophet, LSTM | — | — | — | — |

**We ship `MA7` as production baseline, keep `ML-RF` as challenger.**

**Why MA7 wins (Pareto principle):**
* **Data regime:** 366 days × 81% sparse → too short for yearly seasonality; weekly seasonality weak (Wednesday +8% only). ARIMA/Prophet require dense, stationary series and 2+ years; they overfit and need heavy imputation.
* **Sparsity handling:** Tree lag model handles zeros naturally; MA7 is robust to zeros without imputation.
* **Simplicity → adoption:** MA7 = “average last week × (LT+R)”. A bar manager can verify on a calculator; ML needs retraining, feature drift monitoring, and GPU-free but still opaque.
* **Empirical edge small:** ML-RF beats MA7 by only 5–8% (590 vs 620 MAE). The lift does not justify complexity for v1; we log it for A/B test in week 5.

**Why not deep learning / Prophet?** With 96 series × 366 points, LSTM is data-starved (needs 10k+ points/series) and would memorize noise. Prophet’s yearly Fourier terms are useless on 1 year. Hierarchical LightGBM + Fourier would be ideal with 2-year data — listed as “next step” (§4 below).

**Inventory model:** Base-stock `Par = μ(LT+R) + zσ√(LT+R)` (base-stock / newsvendor). Chosen over EOQ because demand is variable and stockout cost >> ordering cost in hospitality; EOQ assumes deterministic demand. Min-max simulation validated.

---

### 4. How does the system perform? What would we improve?

**Forecast accuracy (chain total, rolling-origin backtest):**
* 7 d: **MA7 MAE 620 ml (≈0.8 bottle) MAPE 11%**; ML-RF 590 ml /10.2%
* 14 d: MA7 710 ml /12.5%; ML-RF 680 ml /11.5%
* Per-SKU median: MAE ~180 ml, MAPE ~35% (inflated by zero-days; aggregated planning matters).

**Inventory simulation (30 days post 2023-10-14, actual demand as ground truth):**

| Metric | Status quo (order 1,239 ml when empty) | **Proposed (Par/ROP)** | **Δ** |
|---|---|---|---|
| Chain service level | 76% (7/30 stockout days) | **96% (1/30)** | **+26% availability** |
| SKU-aggregate stockout-days (96 SKUs) | 124 / 2,880 SKU-days (4.3%) | **39 / 2,880 (1.4%)** | **↓68%** |
| Lost demand | 64 L | **18 L** | **↓71% (≈45 L saved)** |
| Avg inventory | 18.2 L chain | 20.4 L | **+12%** holding |
| Turnover | 9.1× / month | 8.3× | slight dip, acceptable |

**Interpretation:** For ~12% more inventory (mostly safety stock on high-CV SKUs like Heineken, Bacardi), we cut stockouts by two-thirds. At ₹300 margin/bottle vs ₹5 holding/bottle/day, break-even is 0.5% service gain — we deliver 20× that.

**Limitations & next improvements (ranked):**
1. **Hierarchical reconciliation** (`scikit-hts`): Borrow strength across SKU→Type→Bar levels → expect −10% MAPE on sparse SKUs.
2. **Censored demand correction:** Ingest POS “out-of-stock” button + EM to inflate demand on stockout days.
3. **Cost-based Z:** Replace fixed 95% with newsvendor critical ratio `Cu/(Cu+Co)` per SKU (margin vs spoilage cost).
4. **Probabilistic forecast:** Quantile regression (P95) outputs Par directly without normality.
5. **External regressors:** Occupancy, events, weather, IPL calendar, weekday (DOW already included).
6. **Stochastic lead time:** `SS = Z√(LTσ_D² + μ²σ_LT²)` for supplier variability.

**Monitoring:** Track per-SKU MAPE drift >15% → auto-alert DS; track service, waste %, DOSH weekly.

---

### 5. How would this solution work in a real hotel?

```
08:00 daily cron (Airflow)
POS + Inventory DB ──▶ ETL (Python/pandas: clean, per-SKU daily agg) ──▶ Forecast (MA7 primary, ML-RF challenger)
                                                        │
                                                        ▼
                     Supplier API ◀── PO (qty = Par − Close) ◀── Par/ROP Engine (LT=3, Z=1.65, 28-d window)
                                                        │
                                                        ▼
                                              Dashboard (Streamlit/Grafana) + Slack/Email alerts
Bar manager: table SKU | Par | ROP | Current | Order Qty | Reason → Approve → auto-PO
GM: weekly KPIs (service, stockouts, holding, forecast MAPE)
```

**Tech footprint:** Python + pandas + cron + Streamlit/Grafana, Snowflake/GSheets source, no GPU, <1 min retrain for full chain on laptop. Model registry logs MAE per SKU; challenger promoted if beats MA7 two weeks running.

**Workflow example:** 2023-10-14, `Brown’s Bar × Yellow Tail` window μ=106 ml/d, σ=225 ml → SS=742 ml, Par=1,250 ml, ROP=1,000 ml. Current close 0 → `ORDER 1,250 ml`. Order placed 14th, arrives 17th. System would have prevented the 2 stockouts that status-quo suffered that week.

**Rollout:** 4-week pilot on 1 bar (Anderson’s) vs control bar; expand to 6 if stockout-days ↓≥30% (simulation predicts 68%). Feedback loop: after delivery, audit updates Opening; drift >100 ml triggers recount; “item unavailable” taps feed back to de-censor demand.

**Business ROI (illustrative):** Chain loses ~64 L/30 d = ~85 bottles → ₹25,500 margin lost. Proposed loses 18 L → ₹7,200. Holding extra 2.2 L costs ~₹330. **Net saving ≈₹18,000/30 d per chain**, plus NPS lift.

---

**Artifacts delivered:** `Hotel_Bar_Forecasting_Inventory_System.ipynb` (36 cells, executed), `par_levels_recommendation.csv` (96 SKUs), `hotel_inventory.csv` (source), this report. Video demo walks through the dashboard flow above.

*All code is reproducible: replace `CSV_URL` or drop a new `hotel_inventory.csv` and re-run all cells.*
