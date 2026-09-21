# Kristball AIML — Submission Package
**Date:** 21 Sep 2026

## 📂 Contents (all under `submission/`)

| # | File | Purpose | Size | Upload slot per assignment |
|---|---|---|---|---|
| 1 | `Hotel_Bar_Forecasting_Inventory_System.ipynb` | **Clean notebook (36 cells) — no outputs, lightweight for re-execution** | 64 KB | “Python Notebook” upload 1 |
| 2 | `Hotel_Bar_Forecasting_Inventory_System_EXECUTED.ipynb` | **Same notebook with outputs rendered (recommended to upload this one — shows EDA plots, forecast tables, simulation)** | 1.6 MB | Alternative for slot 1 |
| 3 | `hotel_inventory.csv` | Source data (6,575 rows) — also embedded via Google Sheet link in notebook so notebook runs without this file | 449 KB | — |
| 4 | `par_levels_recommendation.csv` | **Generated par/ROP table for all 96 SKUs (Par_S, ROP, SS, action ORDER/HOLD, order_qty)** — previewed in notebook §5c | 8 KB | Attach alongside notebook or as appendix |
| 5 | `Report_Hotel_Inventory_System.md` | **1–2 page report** answering the 5 prompts: business problem, assumptions, model choice, performance & improvements, real-hotel deployment | ~8 KB | Upload as “short write-up” or paste into notebook markdown |
| 6 | `Theory_Answers_Q1_Q5.md` | **Detailed answers to Theory Q1–Q5** (outlier method, 100% CI, XGBoost 98/95, heteroscedasticity remedies, tuning lift) with code & justifications | 13 KB | Attach as PDF/MD for theory section |
| 7 | `Video_Demo_Script_3-5min.md` | **Video demo script + recording checklist** (4:15, covers context → EDA → forecast → par → simulation → deployment) | ~6 KB | Use to record Loom/Zoom; upload resulting `*.mp4` to slot 2 |

> **Recommended uploads:**  
> - Slot 1 (Notebook): `Hotel_Bar_Forecasting_Inventory_System_EXECUTED.ipynb` + `par_levels_recommendation.csv` + `hotel_inventory.csv` as zip *or* just the executed ipynb (it downloads data live).  
> - Slot 2 (Video): Record using `Video_Demo_Script_3-5min.md` → export `demo_4min.mp4` (<10 MB, 720p).  
> - Slot 3 (Report): `Report_Hotel_Inventory_System.md` (convert to PDF via `jupyter nbconvert` or print).

## ✅ How to Reproduce (30 s)

```bash
# Option A: with local CSV
jupyter notebook Hotel_Bar_Forecasting_Inventory_System.ipynb
# → Run All (Kernel → Restart & Run All) — ~90 sec, no extra installs needed beyond pandas/numpy/matplotlib/seaborn/sklearn

# Option B: without CSV (auto-downloads from Google Sheet)
# Notebook tries LOCAL_CSV → hotel_inventory.csv → Google Sheet export URL (ID 14i7oWnBOoIf3pai37bsGYkby3VXJL1AlX_KxC8ZMKtg)
```

**Environment:** Python 3.10+, pandas 2.x, numpy, matplotlib, seaborn, scikit-learn 1.3+. Tested on macOS 14, Homebrew Python 3.14. No GPU, no statsmodels/prophet required.

## 🔑 Key Results at a Glance

* **Forecast (chain, 7 d rolling):** MA7 MAPE 11% (ML-RF 10.2%) — MA7 shipped as baseline.
* **Inventory simulation (30 d):** Stockout-days ↓68% (124→39), lost demand ↓71% (64 L→18 L), service 76%→96%, inventory +12%.
* **Immediate action:** 23/96 SKUs need order now, total 11.4 L (~15 bottles) — see `par_levels_recommendation.csv`.

## 📸 Suggested Video Flow

Business (0:25) → EDA (0:45) → Forecast (1:10) → Par table (0:50) → Simulation (0:50) → Deployment diagram (0:30). Keep screen on executed notebook outputs; no live re-run needed.

---

*Questions? See notebook §7 Assumptions and §8 Deployment for operational details.*
