# Short-Horizon Cross-Sectional Reversal in Crypto

A backtest of a dollar-neutral, long/short reversal strategy on hourly crypto data, built to answer one question: **is the edge real once you account for trading costs and data quality?**

**Short answer: no.** The strategy shows a striking gross Sharpe ratio, but most of that edge disappears after realistic costs, and much of what remains comes from stale prices in hours when coins did not trade. This notebook documents how that conclusion was reached and how to avoid fooling yourself along the way.

---

## The Idea

Short-horizon reversal is a well-known pattern: assets that fall the most relative to their peers over a short window tend to bounce back, and the biggest winners tend to give some of it back. Every hour, the strategy:

1. Computes each coin's average return over the last `hor` hours and flips the sign, so the biggest losers get the highest signal.
2. Ranks coins on that signal and demeans the ranks, so longs and shorts cancel (**dollar-neutral**).
3. Scales weights so gross exposure is 100% (50% long, 50% short).
4. Rebalances every `hold` hours and holds positions in between.

## Data

- **Source:** Binance.US hourly bars via the `python-binance` API
- **Universe:** 9 liquid coins quoted in USDT: BTC, ETH, ADA, BNB, LTC, BCH, XLM, ETC, DOGE
- **Period:** January 2020 – August 2026 (~58,000 hours)
- **Fields:** hourly close price and dollar volume, reindexed onto a complete hourly calendar so missing hours appear as gaps instead of being silently skipped

## Methodology

- **No look-ahead bias:** weights formed at the close of hour *t* earn hour *t+1*'s return (one-bar lag).
- **Transaction costs:** charged on turnover (the total absolute change in weights), tested at 7 bps (optimistic) and 20 bps (realistic).
- **Train / validation split:** the first half of the sample (Jan 2020 – Apr 2023) is used to choose parameters; the second half (May 2023 – Aug 2026) is looked at **once**, after the picks are fixed.
- **Performance stats:** Sharpe ratio and volatility annualized from daily returns, max drawdown on a compounded equity curve, breakeven cost (return per unit of turnover), and an OLS regression on BTC for alpha, beta, and correlation.

## Results

### 1. The baseline looks too good to be true

Using a 1-hour lookback with hourly rebalancing:

| | Gross | Net (20 bps) |
|---|---|---|
| Sharpe ratio | 18.9 | −51.3 |
| Breakeven cost | 5.9 bps | – |
| Turnover per hour | 1.43 | 1.43 |

A gross Sharpe near 19 is not achievable in practice. Part of it is **bid-ask bounce**: closing prices alternate between the bid and the ask, which looks like reversal but can't be traded. The edge per trade (~6 bps) is smaller than the cost of trading, so net returns are deeply negative.

### 2. Parameter sweep on training data only

32 combinations: lookback (1, 6, 24, 72 hours) × holding period (1, 6, 24, 72 hours) × cost (7, 20 bps), ranked by **net** Sharpe.

- **Nothing is profitable at 20 bps.**
- At 7 bps, only two settings are positive: (24, 72) with a net Sharpe of 0.23 and (1, 6) with 0.22, the only two with breakeven costs above 7 bps.

### 3. Validation of pre-committed picks (7 bps)

| Setting (lookback, hold) | Gross Sharpe | Net Sharpe | Breakeven |
|---|---|---|---|
| (1, 6) | 7.46 | **1.71** | 9.1 bps |
| (24, 72) | 0.24 | −0.28 | 3.9 bps |
| (1, 1) | 23.50 | −2.11 | 6.4 bps |

(24, 72) was the best training result, but it was an isolated bump in the parameter grid and failed out of sample. (1, 6), chosen from a smoother, more robust region of the grid, held up, at least at first.

### 4. The stale-price problem

Many coins had hours with **zero trades** on Binance.US (ETC in about 25% of hours, BCH about 14%), especially from 2022 onward. In a dead hour, the "close" is a stale price; when trading resumes, the price jumps, which looks like a reversal that could never have been traded.

A liquidity filter, fixed in advance and using only past data, keeps a coin only if it traded in every one of the last 24 hours, and requires at least 4 tradeable coins per hour.

| Setting | Period | Gross Sharpe (unfiltered → filtered) | Change |
|---|---|---|---|
| (1, 1) | Validation | 23.50 → 13.84 | −41% |
| (1, 6) | Validation | 7.46 → 2.60 | −65% |

With the filter, breakeven costs drop below 7 bps in every case, and the (1, 6) validation net Sharpe turns from **+1.71 to −3.15**. The positive result was largely an artifact of stale prices.

*Caveat:* the filter also shrinks the universe, so part of the drop comes from making fewer bets, not only from removing the artifact.

### 5. Does volume matter?

**Hypothesis:** moves on unusually low volume are more likely to be liquidity noise and should reverse more strongly than moves on high volume.

Across ~195,000 tradeable coin-hours in the training period, the correlation between this hour's and next hour's relative return was:

- **Quiet coin-hours** (bottom half of relative volume): −0.088
- **Busy coin-hours** (top half): −0.067

Quiet moves reverse about **30% more strongly**, as predicted. The gap is clear in 2020 and 2022, marginal in 2023, and absent in 2021, the most liquid year. Quiet hours also have more bid-ask bounce, so part of this effect may not be tradeable.

## Key Takeaways

1. **A high gross Sharpe is a warning sign, not a result.** At hourly frequency, microstructure effects can manufacture returns that don't exist.
2. **Costs decide everything.** The relevant number is breakeven cost per unit of turnover, not gross Sharpe.
3. **Choose parameters from robust regions, not peaks.** The best training setting was noise.
4. **Check the data before trusting the backtest.** Zero-volume hours created a positive validation result that disappeared once stale prices were filtered out.

## How to Run

```bash
pip install python-binance pandas numpy statsmodels
```

Open the notebook in Jupyter and run the cells in order. Outside the US, remove `tld='US'` from the Binance client to use the global exchange. Downloading the full history takes several minutes.

## Notebook Structure

1. Data download and cleaning
2. Step-by-step walkthrough of the strategy
3. Backtest function (`run_strategy`)
4. Performance statistics (`stats`)
5. Train / validation split
6. Parameter and cost sweep
7. Validation of pre-committed picks
8. Volume interaction analysis
9. Liquidity filter and stale-price test

---

*This project is for research purposes only and is not investment advice.*
