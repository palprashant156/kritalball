# Video Demo Script — Hotel Bar Forecasting & Inventory System (3–5 minutes)
**Target length:** 4:00 | **Format:** Screen-record + voiceover (no fancy editing) | **Tool:** Loom / Zoom + notebook + dashboard screenshots

---

### 0:00–0:25 — Hook & Business Context (25 s)
*Show slide or notebook Title cell*

> “Hi, I’m [Your Name]. Hotels in this chain run six bars, each stocking sixteen brands. Today inventory is reactive — staff order when a bottle hits zero. That causes 4% stockout days — guests can’t get their preferred drink — and at the same time overstock of slow movers ties up cash and spoils beer. In this demo I’ll show a forecasting and par-level system that cuts stockouts by two-thirds for only twelve percent more inventory.”

*Visual:* EDA slide: daily demand 5.3k ml, stockout 4.1%, purchase every 3.2 days.

### 0:25–1:10 — Data & EDA (45 s)
*Scroll notebook §2–3*

> “We have 6,575 transactions over 366 days — that’s six bars times sixteen brands = ninety-six SKUs. I validated the inventory identity — closing equals opening plus purchase minus consumed — drift is under one ml in ninety-nine percent of rows, so consumed is our demand signal. EDA shows demand is fairly stable — fifty-four hundred ml a day chain-wide, Wednesday peaks eight percent — but SKU-level variability is high: CV averages 0.6, some brands like Heineken hit 1.8. Top SKUs — Brown’s Bar Yellow Tail, Thomas’s Grey Goose — each drive one-point-five percent of volume. Purchase is ad-hoc, twelve-hundred ml when empty, not par-based.”

*Visual:* Time-series plot, bar×type heatmap, purchase histogram.

### 1:10–2:20 — Forecasting (70 s)
*Show §4 code + forecast plot*

> “I framed it as daily demand per SKU for seven and fourteen days — the replenishment horizon. I reindexed to a full daily grid, treating missing audits as zero demand — assumption documented — and split time-based eighty-twenty, no shuffling to avoid leakage.
>
> I benchmarked four baselines on rolling origin: moving average seven, moving average fourteen, EWMA, naive, and a lag-seven random forest with day-of-week and rolling stats. On chain total, moving average seven achieves six-twenty ml MAE, eleven percent MAPE for seven days; the random forest is five-ninety, ten-point-two — only five percent better. Per SKU median MAPE is thirty-five percent because of sparsity, but aggregated planning is what matters for par.
>
> I ship moving average seven as production — it’s explainable, ‘average last week times lead time’, trains in one millisecond — and keep the forest as a challenger. With two years of data we’d move to hierarchical LightGBM, but on one year complexity hurts.”

*Visual:* Forecast comparison table, two SKU forecast plots (high-CV vs stable).

### 2:20–3:10 — Par Level Logic (50 s)
*Show §5 table*

> “Par is a base-stock formula: mean demand times lead time plus review, plus safety stock — z times sigma times root lead time plus review. I use lead three days, review one, ninety-five percent service — z one-point-six-five — on a twenty-eight day trailing window. That gives differentiated safety stock: high-CV Heineken gets twice the buffer of stable Bacardi. I cap at five-hundred to six-thousand ml and round to two-fifty — bottle size.
>
> Here’s the recommendation table — ninety-six rows, columns for mu, sigma, CV, safety stock, par, reorder point. For example, Brown’s Bar Yellow Tail: mu one-oh-six, sigma two-twenty-four, safety seven-forty-two, par twelve-fifty, ROP one-thousand. Last close was zero, so action is ORDER twelve-fifty. In total, twenty-three SKUs need immediate order, total eleven-point-four litres — about fifteen bottles.”

*Visual:* Scroll `par_levels_recommendation.csv`, highlight ORDER rows.

### 3:10–4:00 — Simulation: How It Works in Practice (50 s)
*Show §6 simulation plot*

> “To prove value I simulated thirty days after split under two policies. Status quo — order twelve-hundred when empty — versus proposed — daily review at eight AM, if closing below ROP, order par minus closing, arrives after three days. Using actual future demand as ground truth, chain service goes from seventy-six percent — seven stockouts in thirty — to ninety-six percent — one stockout — that’s plus twenty-six points availability. Across all SKUs, stockout-days drop sixty-eight percent, from one-twenty-four to thirty-nine, lost demand down seventy-one percent — saving forty-five litres — for only twelve percent more average inventory. The trajectory plot shows proposed inventory stays above ROP; status quo repeatedly hits zero. High-CV SKUs improve most — exactly where safety stock pays off.”

*Visual:* Chain inventory trajectory, per-SKU savings histogram.

### 4:00–4:30 — Real Deployment (30 s)
*Show §8 diagram*

> “In production this runs as a daily eight AM Airflow cron: ETL from POS and inventory DB, forecast, Par engine, publish to a Streamlit dashboard and Slack alerts. Bar manager sees SKU, par, ROP, current, order quantity, taps Approve — PO goes to supplier. GM gets weekly KPIs. Model registry tracks MAPE per SKU; the forest challenger promotes if it beats MA for two weeks. Rollout is pilot on one bar versus control for four weeks. That’s the loop.”

### 4:30–4:45 — Close & Next Steps (15 s)
> “Current system hits eleven percent chain MAPE and cuts stockouts by two-thirds. Next, I’d add hierarchical reconciliation, censored-demand correction with POS unavailable taps, and cost-based z per SKU. All code and the recommendation CSV are in the notebook — reproducible by swapping the CSV link. Happy to answer questions. Thank you.”

---

### Recording Checklist
- [ ] Open executed notebook (`Hotel_Bar_Forecasting_Inventory_System_EXECUTED.ipynb`) — outputs already rendered so you don’t need to re-run live.
- [ ] Have `par_levels_recommendation.csv` open in Excel/Sheets for quick scroll.
- [ ] Keep EDA plots zoomed to 125% for readability.
- [ ] Speak at ~150 wpm, pause after numbers.
- [ ] Total 4:15 incl. buffer; if pressed, compress EDA to 30 s and extend simulation.

### One-Sentence Takeaway for Abstract
> “A seven-day moving-average forecast plus a 95% base-stock par formula cuts bar stockouts by 68% for 12% more inventory — simple, explainable, and deployable tomorrow morning.”

