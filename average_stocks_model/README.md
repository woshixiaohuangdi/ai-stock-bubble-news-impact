# Average Stocks Model — EDA

This folder contains the exploratory data analysis for the five-stock equal-weight 
portfolio experiment. The five stocks are NVDA, META, MSFT, AMZN, and GOOGL, 
chosen as the core AI-related holdings for this study.

## Data Sources

- **Stock prices**: Yahoo Finance daily OHLC data (2021–2025), stored in `data/`
- **News**: Bloomberg Terminal manually collected articles for each stock (2021–2025)
- **Macro**: FOMC meeting records (36 meetings, 2021–2025), including rate decisions 
  and monetary policy phase labels

## What We Did

We built an equal-weight portfolio from the five stocks and ran a series of 
diagnostic checks before moving into formal modelling.

**Portfolio construction**: simple average of daily returns across all five stocks, 
giving each stock a 20% weight.

**Return and volatility analysis**: plotted cumulative returns and rolling volatility 
(5-day and 21-day) against the three monetary policy phases. The phase boundaries 
follow actual FOMC decisions rather than arbitrary cutoffs.

**ACF/PACF tests**: confirmed that portfolio daily returns show no significant 
autocorrelation across 40 lags — consistent with a random walk. Volatility, 
however, shows strong persistence (AR(1) structure), which is the classic 
volatility clustering effect.

**News coverage analysis**: checked how often each stock had above-median daily 
article counts, then aggregated to a portfolio level (≥3/5 stocks simultaneously 
meeting their thresholds). Plotted high-news days against next-day portfolio returns, 
split by monetary policy phase.

## Key Findings

The most important finding from EDA is how the news contribution to portfolio 
volatility has changed over time:

| Phase | Period | High News Share of Excess Moves |
|-------|--------|---------------------------------|
| Phase 1: QE/Zero Rate | 2021–2022 | 31.9% |
| Phase 2: Aggressive Hiking | 2022–2023 | 33.7% |
| Phase 3: High Rate Maintenance | 2023–2024 | 50.9% |
| Out-of-Sample (2025) | 2025 | 69.3% |

In plain terms: in the early QE period, news accounted for roughly a third of 
big market moves. By 2025, that figure had risen to nearly 70%. As monetary policy 
stabilised, AI narrative shocks took over as the dominant driver of volatility. 
This is the core motivation for the DNEM framework.

A second finding worth noting: it is the *rate of change* in interest rates, 
not the absolute level, that drives volatility. Phase 2 (rapid hikes) was the 
most volatile period, while Phase 3 (high but stable rates) was relatively calm — 
comparable to the QE era in terms of day-to-day noise.

## Output Files

| File | Description |
|------|-------------|
| `portfolio_return.png` | Daily return and cumulative return (2021–2025) |
| `portfolio_phases.png` | Cumulative return + rolling volatility with phase shading |
| `acf_pacf_return.png` | ACF and PACF of portfolio daily returns |
| `acf_pacf_volatility.png` | ACF and PACF of 5-day and 21-day rolling volatility |
| `news_coverage.png` | Per-stock news volume vs next-day return scatter |
| `portfolio_news_coverage.png` | Portfolio-level high news days (≥2/5 and ≥3/5 thresholds) |
| `phase_news_coverage.png` | Phase-split analysis of high news days vs next-day return |
