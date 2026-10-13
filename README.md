# Pairs Trading in Crypto: A Statistical Arbitrage Case Study

*Pairs trading on the 100 most traded coins on Binance. Analysis only: no trading, no money. Trading period 1 January 2022 to 12 October 2026.*

## Files

- `README.md`: the light version, a quick skim.
- `crypto_stat_arb.ipynb`: the full version with all code, outputs and the longer reasoning. Partly redundant with this file on purpose.

## The core idea

Some pairs of coins wander around randomly but stay tied to each other, like a person walking a dog on a long leash. The distance between them (the **spread**) stretches and snaps back. In statistics this is called **cointegration**. When the spread is unusually wide, sell the coin that ran ahead and buy the one that fell behind. When it is back to normal, close both. Since one coin is bought and the other sold, a move of the whole market roughly cancels out.

I didn't pick pairs by hand. Every 6 months the strategy tests all 4,950 pairs of the 100 most traded coins on the past year, trades the most cointegrated ones for the next 6 months, then searches again.

Then the real question: **is the result an edge, or luck?** Four checks try to answer that.

## What it shows

- **Gross it earns a little, net it loses.** 4.0% a year before costs (Sharpe 0.20). After 20 bps per trade, -11.9% a year (Sharpe -0.58).
- **The breakeven cost is 5.1 bps.** Binance's standard fee alone is about 10 bps, so it fails on cost by a factor of about 4.
- **The search finds something, but less than it seems.** It beats all 200 sets of random pairs. Yet about 248 of the 4,950 pairs pass the test by luck each period, and in July 2025 the search found 249.
- **The relationships don't last.** It works in 2022 and 2023 (gross Sharpe 1.2 and 1.5) and loses from 2024 on.

## Conclusion

The mechanism is real but weak, and the edge is far too thin for the cost. A cointegration that is "statistically significant" over one year says little about the next six months, which is the actual bet. Luck sits in every step: many tests, picking the best of them, and a rule that trades even when most passes are noise.

## A worked example

Coins A and B, hedge ratio 1. Over the last 90 days the spread `log(A) - log(B)` averaged 0 with a standard deviation of 5%. Today A is up 10% relative to B: the spread is +10%, so z = 2.

1. Sell 1 dollar of A, buy 1 dollar of B.
2. The spread closes back to 0: profit about 10 cents, whatever crypto did as a whole.
3. The spread widens to +20%: loss about 10 cents. The leash broke.

Case 3 is the whole risk. The notebook has a real example (UNI and DASH, 2022): textbook trades for four months, then the spread leaves and never comes back.

## The flow

1. **Data.** Daily closes and volume for about 500 coins from Binance's public API.
2. **Universe.** Top 100 by trading volume over the past year, re-ranked every 6 months. Stablecoins and wrapped tokens are excluded.
3. **Search.** Test all 4,950 pairs, keep the best partner per coin if p < 0.05.
4. **Signal.** Rolling 90 day hedge ratio and z-score of the spread.
5. **Backtest.** Daily returns, turnover, costs, and the scorecard.
6. **Four luck checks.** New data, next door, cost, random market.

## Core maths and code

Cointegration test on the past year (Engle-Granger: regression, then a test whether the leftover keeps returning to 0):

```
log P_y = alpha + beta * log P_x + e        e mean reverting?  ->  p-value
```

Spread and z-score, rolling over 90 days:

```python
beta   = (mean(X * Y) - mean(X) * mean(Y)) / (mean(X * X) - mean(X) ** 2)
spread = Y - (mean(Y) - beta * mean(X)) - beta * X
z      = (spread - mean(spread)) / spread.rolling(90).std()
```

Open at |z| > 1 (sell the expensive side, buy the cheap side), close at |z| < 0.5. Yesterday's weights earn today's return, and every unit of turnover pays the cost:

```python
gross    = (weights.shift() * coin_returns).sum(axis=1)
turnover = weights.diff().abs().sum(axis=1)
net      = gross - turnover * 20e-4
```

Multiple testing: with 4,950 tests at p < 0.05, about 248 pass by luck. Benjamini-Hochberg keeps only the passes that are unlikely to be luck as a group.

## Assumptions

- Flat 20 bps per unit of turnover for every coin. Small coins probably cost more.
- Shorting is free. In reality margin interest or futures funding comes on top.
- Only coins listed today are in the data. Delisted coins, which probably would have hurt, are missing.
- Parameters fixed up front, not tuned: 1 year formation, 6 month rebalance, 90 day window, entry 1, exit 0.5.
- Trading starts in 2022 because before that fewer than 100 of today's coins existed with a full year of history (5 in 2018).

## Outcomes

| Metric | Strategy, gross | Strategy, net | Buy and hold BTC |
|---|---|---|---|
| Annual return | 4.0% | -11.9% | 24.7% |
| Sharpe ratio | 0.20 | -0.58 | 0.49 |
| Max drawdown | -39% | -60% | -67% |
| Holding period | 9 days | 9 days | |
| Correlation with BTC | 0.22 | 0.22 | 1 |

Was it luck?

| Check | Question | Result | Passed? |
|---|---|---|---|
| New data | Do the fresh 6-month blocks make money? | 5 of 10 gross, 4 of 10 net. Since 2024, 5 of 6 lose | No |
| Next door | Do nearby settings, years and coins agree? | Gross positive in 25 of 27 settings. But only 2022 and 2023 work, and neither half of the coins works alone | Parameters yes, time and coins no |
| Cost | Does it survive 20 bps? | Breakeven 5.1 bps | No |
| Random market | Better than random pairs? | Beats 200 of 200. But random pairs lose even before costs | Yes, from a low base |

## What this is not

- Not a tradable strategy, and not proof that pairs trading can't work. It's one textbook version on daily data.
- Missing shorting costs and delisted coins would both make the result worse, not better.
- Daily closes miss any reversion that happens within a day.
- 10 blocks of 6 months is not a lot of evidence, in either direction.

## Run it

```bash
pip install -r requirements.txt
```

```bash
jupyter notebook crypto_stat_arb.ipynb
```

The first run downloads the prices into `data/` (about 2 minutes). The whole notebook takes about 5 minutes, mostly the 4,950 pair tests per period. `data/` is not in the repository, so a fresh run uses newer data and gets slightly different numbers. Set `REFRESH = True` in the first code cell to download again.
