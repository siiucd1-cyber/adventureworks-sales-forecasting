# Probabilistic Sales Forecasting — Monte Carlo & Bootstrap Simulation

*[中文版](README.md)　|　Capstone project, MSc Financial Data Technology, University College Dublin*

> Replacing a single-point sales forecast with a full distribution of outcomes, so a business can plan against **risk** rather than against one number.

Built on three years of AdventureWorks transaction history (121,253 order lines, $109.8M in sales), this project delivers a 12-month forecast as a probability distribution, quantifies the chance of hitting management targets, and produces channel / category / region breakdowns that reconcile exactly with the company total.

---

## Context and scope

**Business situation.** AdventureWorks (retail and wholesale of bicycles and accessories) planned annually from single-point sales forecasts. Leadership wanted a risk-aware alternative — not just a forecast value, but the full range of plausible outcomes and the probability of hitting financial targets. The analyst brief was to supply that probabilistic view for resource allocation and risk management.

**Required scope.** Forecast the next 12 months using two simulation methods — Monte Carlo (top-down, simulating the annual growth rate) and bootstrap (bottom-up, resampling historical daily sales); compare their assumptions, strengths and limitations; and deliver a planning range to management.

**Work added beyond that scope** (all detailed below):

- Identified and solved a **blocking methodological problem**: too few complete years left only one growth observation, resolved by evaluating three alternative definitions and adopting a rolling fiscal year
- Three bootstrap refinements: month stratification, joint-day resampling (coherent segment forecasts), and an adapted transaction-level Poisson simulation
- Sensitivity analysis across five assumption sets, plus three independently coded implementations cross-validating each other
- Data-boundary diagnostics (identified an extraction cutoff, not declining demand)
- A six-page interactive Power BI dashboard and a three-minute management pitch

---

**Deliverables:** [Executive report (PDF)](report/Executive_Report.pdf) · [Analysis notebook](notebooks/sales_forecasting_simulation.ipynb) · [**⬇️ Download Power BI dashboard**](https://github.com/siiucd1-cyber/adventureworks-sales-forecasting/raw/main/dashboard/powerbi_dashboard.pbix)

---

## Key results

| | Forecast | 90% interval |
|---|---|---|
| **Trend continues** (Monte Carlo) | **$75.9M** | $68.2M – $83.6M |
| **Growth stalls** (stratified bootstrap) | **$53.0M** | $50.5M – $55.5M |

**The number that mattered to management:** the probability of beating last year by 10% is ~100% if the growth trend holds, but only **12%** if growth stops. That gap — not either point estimate — is the company's real planning risk.

**Recommendation delivered as a range, not a number:**

| Scenario | Value | Use |
|---|---|---|
| Main planning range | **$68M – $76M** | Budget floor → expected value |
| Downside stress | $53M | Stress-test cash flow and inventory |
| Upside capacity | $84M | Size supply chain, do not budget for it |

![Monte Carlo distribution](figures/02_monte_carlo.png)

---

## The problem

AdventureWorks planned annually from single-point forecasts. Leadership needed to know two things a point estimate cannot answer: what is the **full range** of plausible outcomes, and what is the **probability** of hitting a given financial target?

## Approach

Two independent simulation families, plus three methodological extensions.

**1 · Monte Carlo (top-down).** Simulates the annual growth rate — the quantity that actually drives the target — over 10,000 trials, applied to the current run rate.

**2 · Bootstrap (bottom-up).** Non-parametric: builds 1,000 possible years by resampling real daily sales, assuming no distribution at all.

**3 · Extensions.** Month-stratified sampling (restores seasonality and current sales level), joint-day resampling (coherent segment forecasts), and a transaction-level Poisson simulation used as a diagnostic cross-check.

### The methodological problem I had to solve first

The brief called for year-over-year growth rates from complete calendar years. The data contains only **two** — which yields exactly **one** growth observation (+40.6%) and no way to estimate a standard deviation. I evaluated three alternative definitions before committing:

| Option | Definition | n | Mean | Std dev | Verdict |
|---|---|---|---|---|---|
| A | Monthly YoY | 23 | 80.6% | 87.1% | ✗ Monthly-scale noise; low-base months inflate the mean |
| B | Fiscal-year YoY | 2 | 47.5% | 6.7% | ~ Right quantity, but σ swings 6.7%→11.8% depending on how 15 missing days are imputed |
| C | **Rolling fiscal-year (TTM)** | **12** | **43.2%** | **8.8%** | ✓ **Selected** — annual-scale growth with enough observations |

Option C is the fiscal-year logic of B with the year-end rolled across twelve month-ends. Its limitation is stated openly in the report: overlapping windows make the observations correlated, so σ is likely understated — which is why the volatility assumption is stress-tested separately.

**Data validation caught one more trap:** the date dimension runs to June 2021, but transactions stop abruptly on 15 June 2020. Comparing the final 20 days ($210,541/day) against the prior 90 ($151,247/day) showed activity was still *rising* — an extraction cutoff, not a business collapse. Treating it as declining demand would have biased every forecast downward.

---

## Key insight: the granularity spectrum

The same data, resampled at three different grains, produces three very different risk estimates:

| Resampling grain | Model | 90% interval width |
|---|---|---|
| Year | Monte Carlo | **$15.4M** |
| Day | Stratified bootstrap | $5.0M |
| Transaction | Poisson simulation | $1.5M |

![Model comparison](figures/09_model_comparison.png)

Every step down in grain adds an independence assumption — days independent of days, order lines independent of order lines — and each one shrinks the simulated interval further below the true risk. A transaction-level model looks the most sophisticated and is the most *over-confident* about annual outcomes.

**The practical rule this produced:** set targets at year grain, plan operations at day grain, analyse mix at transaction grain. Choose the grain by the question, not by how granular the data allows you to go.

---

## Coherent segment forecasts without estimating a correlation matrix

Segment-level forecasts are the obvious next ask — but simulating each segment independently is unsafe here. A diagnostic showed most segments are not statistically estimable (Accessories shows +566% "growth" purely from a low base) and the two sales channels' growth rates correlate at **−0.72**: independent simulations would misstate total risk.

![Segment growth diagnostic](figures/06_segment_growth_diagnostic.png)

The fix cost one design change: resample **day indices** rather than day values, so each drawn day carries its full segment decomposition. Within-day cross-segment structure is preserved exactly, no correlation matrix is estimated, and channel/category/region parts sum to the simulated total in **all 1,000 trials** — verified by assertion in the notebook.

![Segment forecast](figures/07_segment_forecast.png)

Business findings that came out of it: Bikes account for **84% of revenue** (concentration risk), and Europe now contributes **25% of recent sales** against an 18% three-year average (the growth engine). Internet's interval is narrow while Reseller's is wide — steady B2C demand versus lumpy B2B batch orders, so the two channels need different planning buffers.

---

## Validation

- **Three-way cross-check:** stratified daily bootstrap ($53.00M), joint-day bootstrap ($53.04M) and transaction-level simulation ($53.14M) — three independently coded resampling schemes over the same pool agree within **0.3%**.
- **Sensitivity analysis:** re-ran the forecast under five assumption sets (alternative growth definitions, σ × 1.5, zero-growth stress, literal-brief base). Quantified that using the stale calendar-2019 base instead of the current run rate would cut the central forecast by **$14.6M**.
- **Reproducibility:** fixed random seeds throughout; every figure and number in the report regenerates exactly on re-run.

---

## Limitations

Stated plainly, because they bound how the output should be used:

1. **Thin statistical base.** The growth distribution rests on 12 *overlapping* rolling-year observations, all drawn from a single three-year expansion. The model cannot see turning points, and the normal distribution is a heuristic assumption, not an empirical finding.
2. **Bootstrap exchangeability.** Days are assumed interchangeable within their stratum — no week-to-week momentum is preserved, and the zero-growth pool freezes the segment mix of the last 12 months.
3. **Finer grain understates risk.** Transaction-level resampling breaks up multi-line B2B orders that arrive together, erasing within-day clustering; its $1.5M interval is not a credible annual risk range.
4. **Data boundaries.** Three years of history, truncated by extraction. Most segments are too small or too low-base for reliable growth estimates.
5. **No external drivers.** Entirely endogenous — macroeconomic conditions, competition and pricing strategy are outside the model.

**Next step:** re-run quarterly. Each new quarter adds rolling-year observations and tightens the growth distribution; once enough history accumulates, a segment-level Monte Carlo with correlated draws becomes viable.

---

## Dashboard

A six-page Power BI dashboard accompanies the analysis, built for the management pitch:

| Page | Content |
|---|---|
| 01 History | Monthly sales history, stacked by Internet / Reseller channel |
| 02 Risk Range | Monte Carlo histogram + three KPI cards (budget floor / expected / upside), with the 5% tails colour-coded |
| 03 Model Comparison | Annual-total distributions of all four models, overlaid |
| 04 Operations | 365-day forecast band (P5 / median / P95) + median sales by fiscal quarter |
| 05 Segment | Segment forecast bars with a dimension slicer (channel / category / territory) |
| 06 Recommendation | Four-scenario KPI cards and the final recommendation |

### ⬇️ [Download the dashboard file (powerbi_dashboard.pbix, 390 KB)](https://github.com/siiucd1-cyber/adventureworks-sales-forecasting/raw/main/dashboard/powerbi_dashboard.pbix)

Open it in **Power BI Desktop** (free, Windows) to explore all six pages and the interactive slicers.

> Note: `.pbix` is a binary format GitHub cannot preview — opening the file in the repo browser shows a blank page, which is expected; use the download link above. The charts in `figures/` are generated by the notebook, cover the same content, and are viewable online without downloading.

---

## Repository guide

```
notebooks/   sales_forecasting_simulation.ipynb   full analysis, outputs saved (renders on GitHub)
report/      Executive_Report.pdf                 14-page executive report, 9 figures + 8 tables
dashboard/   powerbi_dashboard.pbix               6-page interactive dashboard
             pitch_data_for_powerbi.xlsx          simulation output prepared for BI charting
figures/     *.png                                all charts, exported from the notebook
data/        AdventureWorks_Sales.xlsx            Microsoft sample dataset (star schema)
```

### Reproduce

```bash
pip install -r requirements.txt
jupyter notebook notebooks/sales_forecasting_simulation.ipynb   # Kernel → Restart & Run All
```

Runs end to end in under a minute. Seeds are fixed, so output matches the report exactly.

**Tech stack:** Python (pandas, NumPy, Matplotlib) · Jupyter · Power BI (DAX measures, conditional formatting, binning) · dimensional modelling / star schema

---

## Notes

Data is the Microsoft AdventureWorks sample dataset, used for demonstration; figures are illustrative and not real-world sector benchmarks. Produced as an academic project for the Financial Data Technology module, UCD. AI tools were used to assist with code implementation and document formatting; the modelling framework, analytical decisions and conclusions are my own.
