# Pairs Trading in Crypto: A Statistical Arbitrage Case Study

*Pairs trading on the 100 most traded coins on Binance. Analysis only: no trading, no money. Trading period 1 January 2022 to 12 October 2026.*

## Files

- `README.md`: the light version, a quick skim.
- `crypto_stat_arb.ipynb`: the full version with all code, outputs and the longer reasoning. Partly redundant with this file on purpose.
- `experiments/2_variants.ipynb`: the fixes I tried on the way to the final design.

## The core idea

Some pairs of coins wander around randomly but stay tied to each other, like a person walking a dog on a long leash. The distance between them (the **spread**) stretches and snaps back. In statistics this is called **cointegration**. When the spread is unusually wide, sell the coin that ran ahead and buy the one that fell behind. When it is back to normal, close both. One coin is bought and one sold, so a move of the whole market roughly cancels out.

I didn't pick pairs by hand. Every 6 months the strategy tests all 4,950 pairs of the 100 most traded coins on the past year. That many tests produce a lot of lucky passes, so only pairs that survive a multiple-testing correction (Benjamini-Hochberg) are traded, for the next 6 months. Then it searches again.

Then the real question: **is the result an edge, or luck?** Four checks try to answer that.

## What it shows

- **It makes money after costs.** 11.5% a year after 20 bps per trade, a net Sharpe of 0.42 (gross 17.2%, Sharpe 0.63). Correlation with BTC: 0.04.
- **The breakeven cost is 60.5 bps** per unit traded, three times the assumed cost. Even at 40 bps the net Sharpe is 0.22.
- **The filter does the heavy lifting.** Without it, the strategy trades every lucky pass and earns nothing (gross Sharpe 0.01). With it, it trades when the search finds real stand-outs and sits out when it doesn't.
- **The typical trade is good, a few are terrible.** The median trade makes about 910 bps after costs and 78% win. The worst 5% lose about 88 log points each: those are the broken leashes.

## Conclusion

It's a tradable strategy as long as the real all-in cost per unit traded, including the cost of shorting, stays below the breakeven of about 60 bps. This is a research result on daily data, not a live track record.

## A worked example

Coins A and B, hedge ratio 1. Over the past year the spread `log(A) - log(B)` averaged 0 with a standard deviation of 5%. Today A is 10% ahead of B: the spread is +10%, so z = 2.

1. Sell 1 dollar of A, buy 1 dollar of B.
2. The spread closes back to 0: profit about 10 cents, whatever crypto did as a whole.
3. The spread widens to +20%: loss about 10 cents. The leash broke.

Case 3 is the main risk. The notebook has a real example (UNI and DASH, 2022): one trade sits for three and a half months waiting for the spread to come back.

## The flow

1. **Data.** Daily closes and volume for about 500 coins from Binance's public API.
2. **Universe.** Top 100 by trading volume over the past year, re-ranked every 6 months. Stablecoins and wrapped tokens are excluded.
3. **Search.** Test all 4,950 pairs, keep the Benjamini-Hochberg survivors, pick the best partner per coin among them.
4. **Signal.** z-score of the spread, with the hedge ratio from the same year as the test.
5. **Backtest.** Daily returns, costs, the scorecard, and every trade on its own.
6. **Four luck checks.** New data, next door, cost, random market.

## Core maths and code

Cointegration test on the past year (Engle-Granger: a regression, then a test whether the leftover keeps returning to 0):

```
log P_y = alpha + beta * log P_x + e        e mean reverting?  ->  p-value
```

With 4,950 tests at p < 0.05, about 248 pass by luck. Benjamini-Hochberg sorts the p-values and keeps the largest k with `p_(k) <= k / 4950 * 0.05`, which keeps the share of lucky finds among the survivors at about 5%.

Keep alpha, beta and sigma (the std of e) from that regression for the next 6 months:

```python
z = (log_y - alpha - beta * log_x) / sigma
```

Open at |z| > 1 (sell the expensive side, buy the cheap side), close at |z| < 0.5. Since alpha, beta and sigma don't move, a trade's profit per dollar of y is exactly:

```
profit = side * (z_out - z_in) * sigma
```

Portfolio: yesterday's weights earn today's return, and every unit of turnover pays the cost:

```python
gross    = (weights.shift() * coin_returns).sum(axis=1)
turnover = weights.diff().abs().sum(axis=1)
net      = gross - turnover * 20e-4
```

Two things need refitting: the relationship itself (the test) and the translation into a spread (alpha, beta, sigma). Here both every 6 months.

## Assumptions

- Flat 20 bps per unit of turnover for every coin. Small coins probably cost more.
- Shorting is free. In reality margin interest or futures funding comes on top. The breakeven is the room for it.
- Only coins listed today are in the data. Delisted coins, which probably would have hurt, are missing.
- Settings: 1 year formation, 6 month refit, entry 1, exit 0.5, Benjamini-Hochberg at 5%.
- Trading starts in 2022, because before that fewer than 100 of today's coins have a full year of history (5 in 2018).

## Outcomes

| Metric | Strategy, gross | Strategy, net | Buy and hold BTC |
|---|---|---|---|
| Annual return | 17.2% | 11.5% | 24.7% |
| Sharpe ratio | 0.63 | 0.42 | 0.49 |
| Max drawdown | -28% | -31% | -67% |
| Holding period | 26 days | 26 days | |
| Correlation with BTC | 0.04 | 0.04 | 1 |

Trade by trade, in bps of 1 dollar of y, after costs:

| | Entry 1 | Entry 1.5 | Entry 2 |
|---|---|---|---|
| Trades | 563 | 348 | 248 |
| Median trade | 910 | 1,189 | 1,384 |
| Winning trades | 78% | 71% | 69% |
| Worst 5%, average | -8,794 | -9,420 | -9,575 |
| Average trade | 285 | 388 | 525 |

The four checks:

| Check | Question | Result |
|---|---|---|
| New data | Do the fresh 6-month blocks make money? | 5 of 10, with two blocks of 3 to 5 pairs carrying much of it |
| Next door | Do nearby settings, years and coins agree? | All 9 settings positive, gross and net. 3 of 5 years. Mostly coins 51 to 100 |
| Cost | Does it survive 20 bps? | Yes. Breakeven 60.5 bps |
| Random market | Better than shuffled markets with no edge? | 0.63, near the top of the luck range (-0.65 to 0.69) |

## How the design got here

Each step: look at where it fails, think of a fix, test it, keep it or drop it.

| Step | Gross Sharpe | Net Sharpe | Kept? |
|---|---|---|---|
| Dynamic beta (rolling 90 days), all pairs | 0.20 | -0.58 | No: z can fall back without the spread coming back |
| Static beta, all pairs | 0.01 | -0.27 | Yes, the static beta |
| Static beta, entry 1.5 / 2 | -0.21 / -0.34 | -0.43 / -0.53 | No |
| Break alarm at 25% / 50% / 100% | -0.39 / 0.49 / 0.14 | -0.71 / 0.06 / -0.23 | No: nearby limits disagree |
| BH filter at 1% / 10% | 0.71 / 0.15 | 0.60 / -0.07 | Next-door check: stricter is better |
| **BH filter at 5%** | **0.63** | **0.42** | **Yes** |

## What this is not

- Not a live track record. It's a backtest on daily closes, with costs as one flat assumption.
- Shorting costs and delisted coins are missing. Both would lower the result.
- In some periods only a handful of pairs are traded, so a few trades carry a lot of weight.
- Daily closes miss anything that happens within a day.

## Run it

```bash
pip install -r requirements.txt
```

```bash
jupyter notebook crypto_stat_arb.ipynb
```

The first run downloads the prices into `data/` (about 2 minutes). The whole notebook takes about 40 minutes: about 5 for the strategy, and 30 for the random-market check, which repeats the full pair search 10 times. `data/` is not in the repository, so a fresh run uses newer data and gets slightly different numbers. Set `REFRESH = True` in the first code cell to download again.

`experiments/2_variants.ipynb` holds the tuning steps. Its first run takes about 75 minutes (20 shuffled pair searches and an hourly price download), after that a few minutes, because everything slow is saved in `data/`.
