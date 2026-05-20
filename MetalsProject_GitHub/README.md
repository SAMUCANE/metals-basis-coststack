# LME–SHFE Metals Basis: Cost Stack, Econometric Analysis & Trading Backtest

End-to-end study of the price gap between the **London Metal Exchange (LME)** cash quote
and the **Shanghai Futures Exchange (SHFE)** continuation contract for the six LME base
metals: **Copper, Aluminium, Nickel, Lead, Tin, Zinc**.

The project breaks the SHFE–LME spread into the components an arbitrageur actually pays
(freight, BAF, CAF, cost of carry, VAT financing, insurance), tests what macro factors
drive the residual *net basis* econometrically, and runs a trade-level backtest with
realistic slippage and stochastic transit time.

---

## Headline results (2016-01-04 → 2025-12-24, 2,263 daily obs)

| Metric | Result |
|---|---|
| Total cost stack | $159–$236 /t depending on metal |
| Net basis weekly Δ | Stationary for all 6 metals (ADF + KPSS) |
| Weekly panel regression | F = 5.71, ΔBDI & ΔUSDCNY significant |
| Nickel Chow break 2020-01-01 (Indonesia ore ban) | F = 41.8, p < 0.001 |
| Nickel Chow break 2022-03-08 (LME squeeze) | F = 111.1, p < 0.001 |
| Half-life (winsorised, all metals) | 3–41 days; Lead alone > 35 days |
| Backtest trades simulated | 4,296 |
| Backtest hit rate | 92.8 – 98.7 % across metals/scenarios |
| Mean P&L per trade | $37 (Al) – $887 (Sn) per tonne, aggressive slippage |

The full numerical breakdown and discussion is in
[`report/Metals_Basis_CostStack_Report.docx`](report/Metals_Basis_CostStack_Report.docx).

---

## Repository layout

```
MetalsProject_GitHub/
├── README.md
├── requirements.txt
├── LICENSE
├── .gitignore
│
├── notebooks/
│   ├── Metals_Basis.ipynb              # data loading + gross basis (sections 0-2)
│   ├── Metals_Basis_CostStack.ipynb    # full pipeline (sections 0-12)
│   ├── METALS_Final_v2.xlsx            # LME + SHFE + FX (co-located so notebooks load)
│   ├── COST_STACK.xlsx                 # Freight + Oil + Interest Rates + Freight Base
│   └── FX.xlsx                         # standalone FX (optional)
│
├── outputs/
│   ├── RESULTS_20_05_26.xlsx           # 13 sheets covering Sections 6-11
│   └── figures/
│       ├── net_basis_all_metals.png
│       ├── cost_stack_waterfall.png
│       ├── sensitivity_heatmap_v2.png
│       ├── nickel_breaks_halflife.png
│       └── backtest_pnl_distribution.png
│
└── report/
    └── Metals_Basis_CostStack_Report.docx   # detailed write-up (~30 pages)
```

The notebook reads data files by bare filename, so the Excel files live next to the
notebook inside `notebooks/`. To run, **open the notebook with `notebooks/` as the
working directory**.

---

## How to run

### 1. Create a Python environment

```bash
python -m venv .venv
source .venv/bin/activate          # macOS / Linux
# .venv\Scripts\activate            # Windows
pip install -r requirements.txt
```

### 2. Launch Jupyter

```bash
cd notebooks
jupyter lab Metals_Basis_CostStack.ipynb
```

### 3. Run the cells

`Kernel → Restart Kernel and Run All Cells`. The notebook is idempotent: it
recreates every figure in `notebooks/` and writes
`RESULTS_20_05_26.xlsx` on completion. Copy them across to
`outputs/` if you want to commit a fresh run.

---

## Pipeline at a glance (sections inside the main notebook)

| § | Topic | Output |
|---|---|---|
| 0 | Setup, parameters, file paths | `Transit_days=35`, `VAT=0.13`, ... |
| 1 | Data loading (LME, SHFE, FX, freight, oil, rates) | `LME_aligned`, `SHFE_aligned`, `FX_aligned`, ... |
| 2 | Gross basis = SHFE_usd / (1+VAT) − LME | `Gross_basis` |
| 3 | Cost of carry = P·SOFR3M·T/365 | `df_CoC` |
| 4 | VAT financing = P·VAT·SOFR3M·T_recov/365 | `df_VAT` |
| 5 | Insurance = 10 bps · P (ICC-C) | `df_Insurance` |
| 6 | Total cost stack | `Total_cost_stack` |
| 7 | Net basis + time-series plot | `Net_basis`, `net_basis_all_metals.png` |
| 8 | Cost-stack waterfall | `cost_stack_waterfall.png` |
| 9 | Static sensitivity | `df_Sensitivity` |
| 10 (S6) | Stationarity tests | `S6_Stationarity` |
| 11 (S7) | Panel regression D/W/M | `S7_Panel_*`, `S7_Comparison` |
| 12 (S8) | Per-metal OLS + heatmap | `S8_MetalOLS`, `sensitivity_heatmap_v2.png` |
| 13 (S9) | Hypothesis evaluation H1–H4 | `S9_Hypotheses` |
| 14 (S10) | Nickel breaks (Chow + Bai-Perron) + half-lives | `S10_ChowTests`, `S10_NickelHL`, `S10_HalfLife`, `nickel_breaks_halflife.png` |
| 15 (S11) | Trade backtest with 3 slippage scenarios | `S11_Scenari`, `S11_PerMetal`, `S11_Trades`, `backtest_pnl_distribution.png` |
| 16 (S12) | Export to `RESULTS_20_05_26.xlsx` | 13-sheet workbook |

---

## Key methodological choices (and why)

- **SHFE quote stripped of VAT before USD conversion.** Chinese import VAT (13%) is paid
  to the SAT at customs and only recovered later. An arbitrageur never monetises it, so
  including VAT in the gross-basis numerator overstates the gap by ~13%.

- **SOFR3M (not SOFR overnight) as the financing rate.** Transit ≈ 35 days; the financing
  cost is borne over a horizon closer to 3 months. SOFR3M is the cleanest USD risk-free
  benchmark at that tenor.

- **ΔSOFR excluded from the regression.** Cost of carry, which is subtracted from net
  basis, is itself a linear function of SOFR. Including ΔSOFR as a regressor creates
  mechanical endogeneity. SHIBOR3M is used instead as a Chinese liquidity proxy.

- **Metal-specific BDI sub-index.** Capesize (Cu, Al), Handymax (Ni, Sn), Supramax (Pb,
  Zn) — different vessel classes price the freight of different metals depending on
  parcel size.

- **Winsorise before AR(1) on Nickel.** The 2022 LME squeeze gives a kurtosis of 37 in
  the raw series; an AR(1) fit dominated by those outliers produces a half-life of 2.3
  days. The fix is to winsorise at 1/99% and re-fit separately in each policy regime.

- **Trade-level (not daily) Sharpe in the backtest.** A daily P&L of a fully-hedged
  35-day trade has near-zero variance by construction. Computing Sharpe on daily P&L of
  such a strategy produces meaningless 10+ Sharpes. Each trade is one observation in the
  corrected version, and slippage is the source of dispersion.

---

## Requirements

See [`requirements.txt`](requirements.txt). The notebook auto-installs `linearmodels` and
`ruptures` if they are missing, but they are listed in the requirements file for
deterministic environment creation.

---

## License

[MIT](LICENSE) — feel free to fork, reuse, extend.
