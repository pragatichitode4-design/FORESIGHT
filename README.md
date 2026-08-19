# Project FORESIGHT — Demand & Inventory Intelligence

**Author:** Adnan
Client: NorthBay Living · Zidio Data Science & Analytics engagement (4 weeks)

## Setup

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

## Run the pipeline (D1)

```bash
python src/pipeline.py
```

Produces cleaned tables + one joined analysis dataset in `data/processed/`.

## Train the production model, backtest, and score risk (D3 + D4)

```bash
python src/run_forecast_pipeline.py
```

Runs a 3-fold rolling-origin backtest (model vs seasonal-naive baseline),
trains the final model on full history, saves it to `models/demand_model.joblib`,
generates an 8-week forward forecast for every SKU (daily AND weekly grain,
with an 80% prediction interval calibrated from actual backtest residuals —
not a decorative band), and scores stockout/overstock risk. Outputs:
`data/processed/backtest_results.json`, `forecast_output.csv` (aggregate
per-SKU summary used for risk scoring), `forecast_daily.csv` (day-by-day,
56 days × every SKU, with interval + baseline), `forecast_weekly.csv`
(8-week SKU-level breakdown, per brief D3 acceptance criterion #1),
`risk_output.csv`.

## Compare multiple forecasting algorithms (per mentor guidance)

```bash
python src/model_comparison.py
```

Benchmarks the seasonal-naive baseline against Random Forest, HistGBM,
XGBoost, LightGBM, Prophet, and SARIMA on a sample of 20 SKUs (top 15 by
volume + 5 random). Any library not installed is skipped with a clear
message rather than crashing — `pip install xgboost lightgbm prophet
statsmodels` to include all of them. Output:
`data/processed/model_comparison.csv`, feeds the Forecast dashboard page.

**Result on this dataset (Random Forest + HistGBM, verified):** both
comfortably beat baseline (avg WAPE ~0.34-0.36 vs baseline ~0.64) —
see `reports/executive_readout.md` for the full production-model result.

## Run the dashboard (D5) — 7 pages, behind login

```bash
streamlit run app/streamlit_app.py
```

First thing you'll see is a **sign-in screen** — nobody reaches the
dashboard without an account. On first run, click the "Create account"
tab, register a username/password, then sign in. Credentials are stored
locally as salted-hash entries in `app/.user_store/users.json` (never
committed to git — see `.gitignore`). This is coursework-appropriate
auth, not production-grade (no rate limiting, no email verification).

Once signed in you'll see the branded sidebar with 7 pages: Home, Sales
Analytics, Forecast, Inventory Dashboard, Risk Dashboard, Product
Details, Executive Summary. Filters (Category / SKU / Risk Quadrant)
persist as you navigate between pages. Requires `streamlit>=1.36` (for
`st.navigation`/`st.Page`) — already pinned in `requirements.txt`.

## Run the scoring API locally (D6)

```bash
uvicorn service.main:app --reload
```

Then visit http://127.0.0.1:8000/docs for the interactive API docs.

## Run tests

```bash
pytest
```

## Project structure

```
foresight/
├── .streamlit/
│   └── config.toml              # custom theme (colors, font)
├── data/
│   ├── raw/            # the 4 source extracts — 5-year, 300-SKU dataset
│   └── processed/       # pipeline + model output
├── notebooks/
│   ├── 01_eda.ipynb
│   ├── 02_baseline.ipynb
│   └── 03_model.ipynb
├── src/
│   ├── pipeline.py             # D1: ingest + clean + join
│   ├── forecast.py             # D3: baseline, features, WAPE
│   ├── run_forecast_pipeline.py # D3/D4: production model, backtest, forecast, risk
│   ├── model_comparison.py     # multi-algorithm comparison (RF, HistGBM, XGBoost,
│   │                             LightGBM, Prophet, SARIMA vs baseline)
│   └── risk.py                 # D4: stockout/overstock scoring
├── app/
│   ├── streamlit_app.py   # entry point: auth gate + st.navigation router
│   ├── auth.py            # login/register logic + styled auth screen
│   ├── theme.py           # brand colors, chart palette, shared CSS
│   ├── data_utils.py      # shared data loaders + persistent filters
│   ├── .user_store/       # local credentials (gitignored, created on first run)
│   └── views/
│       ├── home.py
│       ├── sales_analytics.py
│       ├── forecast.py
│       ├── inventory_dashboard.py
│       ├── risk_dashboard.py
│       ├── product_details.py
│       └── executive_summary.py
├── service/
│   └── main.py           # D6: FastAPI scoring service
├── reports/
│   ├── data_quality_memo.md
│   └── executive_readout.md
├── tests/
│   └── test_pipeline.py
├── requirements.txt
└── README.md
```

## Milestone checklist (Section 11 of the brief)

- [x] M1 — Reproducible pipeline + data-quality report (Week 1) — `src/pipeline.py`, `reports/data_quality_memo.md`
- [x] M2 — EDA insight memo + seasonal-naive baseline & metric defined (Week 2) — `notebooks/01_eda.ipynb`, `notebooks/02_baseline.ipynb`
- [x] M3 — Backtested forecast beating baseline + risk scoring (Week 3) — `src/run_forecast_pipeline.py`, WAPE 39.9% vs baseline 62.8%
- [x] M4 — Dashboard + deployed service + executive readout (Week 4) — `app/streamlit_app.py`, `service/main.py`, `reports/executive_readout.md`

**Still to do before final submission (Section 13):**
- [ ] Deploy the dashboard + API to a public URL (Streamlit Community Cloud / Render / HF Spaces) — currently runs locally only
- [ ] Record the 3-5 minute demo video
- [ ] Fill in the "Setup" section below with your own confirmation the repo runs clean on a fresh clone
- [ ] Optionally convert `reports/executive_readout.md` to slides if your cohort requires .pptx

## The non-negotiable rule

Beat the baseline honestly: rolling-origin CV only, never a random split.
No feature may use future information. If the model doesn't beat
seasonal-naive, report that — it's a finding, not a failure to hide.
