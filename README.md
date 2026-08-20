# Polymarket crypto up/down — research dataset

**53,211 observations across 10,044 distinct markets**, 7–20 August 2026,
collected live from Polymarket and Binance. Coins: btc, eth, xrp, sol (plus a
trace of bnb and doge). Horizons: 5-minute and 15-minute.

Each row is one look at one market at one moment: what the chart showed, what
the book was charging, and how the market actually resolved.

**What is deliberately not in here:** which parameter combinations were
actually traded, position sizes, fees paid, or P&L. Those encode a working
strategy and they are not mine to hand over. What is here is the *prediction
problem itself*, plus everything I learned about the plumbing — which is the
part that costs weeks and teaches you nothing interesting.

---

## The market

Polymarket lists recurring binary markets: **does the coin finish the interval
above where it started?** New intervals open continuously.

- **Slug format:** `<coin>-updown-<horizon>-<unix>`.
  **The timestamp is the interval START, not the end.** `interval_end = start
  + duration`. Getting this backwards makes every trader on the platform look
  like a 50% coin flip. It cost me a complete rescoring pass.
- **Resolution** is point-in-time close vs point-in-time open — not an average
  over the window. Reverse-engineered from 1,075 settled markets: **95.5%
  agreement** with close-vs-open on Binance spot. The disagreement concentrates
  on near-ties, which is a hint in its own right (see "Where I'd look", #1).
- **Exactly 7 five-minute markets exist at any moment**, one per listed coin.
  The books are thin and wildly unequal. Measured depth within 2¢ of touch:
  btc ~$214, hype ~$50, eth ~$45, sol ~$24, doge ~$5, xrp ~$2, bnb ~$1.
  **Liquidity, not signal, is frequently the binding constraint.** A strategy
  that needs $50 a clip on xrp does not exist regardless of how good it is.

## Columns

| column | meaning |
|---|---|
| `coin`, `horizon` | `btc`/`eth`/`xrp`/`sol`/`bnb`/`doge`, `5m`/`15m` |
| `slug` | market id — join key, and the venue's own identifier |
| `interval_start`, `interval_end` | unix seconds |
| `observed_ts` | unix seconds when this observation was taken |
| `secs_left` | seconds remaining in the interval at observation |
| `side` | which side this row is quoted for (`Up`/`Down`) |
| `spot` | Binance spot at observation |
| `interval_open` | Binance spot at `interval_start` — the strike |
| `move_bp` | `(spot − interval_open)/interval_open × 10⁴`; signed, in basis points |
| `sigma` | my per-second volatility estimate (see caveats) |
| `ask_this_side` | what the book was asking for `side` |
| `bid_this_side` | best bid on `side` |
| `book_notional` | total resting notional in the book |
| `touch_notional` | notional at the touch — the practical fill limit |
| `resolved_up` | **venue truth**: 1 if the market settled Up |
| `this_side_won` | 1 if `side` was the winning side |

Base rate: `resolved_up` = 51.0%. Near-balanced, no obvious directional tilt.

## The economics — read this before modelling anything

Polymarket charges takers a fee that depends on price:

```
fee          = shares × 0.07 × price × (1 − price)
breakeven WR = price + 0.07 × price × (1 − price)
```

So the bar to clear is **51.75% at a price of 0.50, 81.1% at 0.80, 90.6% at
0.90.** Two consequences people miss:

1. **You are never trying to beat 50%.** You are trying to beat the price you
   pay, plus the fee. A model that is 78% accurate is worthless if it only
   fires on markets priced at 0.80.
2. **The fee is heaviest in the middle of the book** (the `p(1−p)` term peaks
   at 0.50) and nearly free at the extremes. That shapes where an edge can
   survive, independent of where the signal is.

Fees are not columns in this dataset because they are size-dependent. Compute
your own from the formula.

---

## Six traps that cost me weeks

**1. The slug timestamp is the interval start.** Covered above. Validate any
scorer you write against a wallet whose behaviour you already know before you
trust it on a stranger's.

**2. Binance 1-minute klines cannot resolve these markets.** Minute bars
disagree with venue settlements on roughly **21% of small-move markets**. I
once built an entire "opportunity" thesis on kline-resolved outcomes and had to
retract all of it. The `resolved_up` column here is venue truth, pulled from
Polymarket's own settlement. Never resolve from klines.

**3. De-duplicate, then de-duplicate again.** I once reported a 30,558-trade
validation. The audit found **5.3× duplication** — many parallel configurations
were trading the same underlying markets, so the same coin flip was counted
five times. De-duplicated, the finding went from "+7.0" to "−0.3". It was not a
finding at all. This dataset is already de-duplicated on
`(slug, side, secs_left)`.

**4. Early exits create survivorship bias.** If a strategy sometimes sells
before settlement, scoring only its *settled* positions silently deletes every
trade it bailed out of. In my own book this reversed the entire ranking: one
approach looked like the best thing I had (+$1,940 measured on settles alone)
and was +$541 once exits were counted; another looked like the worst (−$569)
and was actually **+$478**. Four of the five apparently-worst approaches were
around breakeven or better. If your P&L doesn't reconcile to cash, it's wrong.

**5. Backtesting this is impossible with public data.** Polymarket's
`prices-history` endpoint returns **one point per 10 minutes regardless of
`fidelity=1`**. You cannot reconstruct the price path of a single 5-minute
market. There is no historical book, either. Everything must be forward-tested
on live or paper fills — which is the entire reason this dataset exists.

**6. Taker fills are adversely selected.** The paper edge concentrates in the
fills you *don't* get; the subset that actually fills performs materially
worse. And **displayed depth did not predict fill success** in my sample — the
misses showed *more* liquidity than the fills did. So you cannot make a
simulated fill honest by gating on depth. Micro-stakes live testing is the only
honest validation I found.

---

## Already ruled out — directions, so you can skip them

- **Cheap longshots bought unscreened** (below ~0.35) are consistently
  negative; below 0.10 catastrophically so.
- **Blending a model's probability toward the market's implied probability**
  loses to the raw model. The market is not adding information you lack.
- **Widening a band that works until it covers most of the book** kills it.
  This is not a "more trades = more money" domain; the edge is local.
- **The 0.55–0.65 region has been dead in every cut I've made** — and also in
  the record of an independently profitable trader I reconstructed. Two
  unrelated sources agreeing is the strongest negative result in here.
- **The 4h horizon** has real depth (~$800+ resting, versus near-empty 5m
  books) but I saw no edge. Sample was tiny (n=23), so read that as
  *not properly tested* rather than dead. Given the depth, it's arguably the
  most under-explored thing on this list.
- **Copying a profitable wallet trade-for-trade doesn't transfer.** The one
  trader I fully decoded earned about +$800 over 4,000 trades at zero
  slippage, **+$346 at 1¢, and −$105 at 2¢.** His edge lived entirely inside
  execution quality I couldn't reproduce. Every copy attempt died there, and
  the failure was invisible until I charged realistic costs.

---

## Where I'd look if I were starting fresh

**1. The settlement oracle is discrete, and the model isn't.**
Polymarket settles crypto up/down against Chainlink. I measured the on-chain
BTC/USD and ETH/USD aggregators on Polygon: they update roughly **every 33
seconds** (40 rounds each, p25 = p75 = 33s, nothing over 60s, both coins on the
same heartbeat). A 5-minute market therefore contains only about **9 price
prints**. Late in an interval the remaining randomness is a small integer
number of discrete jumps rather than a continuum — and once no print remains
before the close, the outcome is *already fixed* while any diffusion-style
model still prices it as live. This also explains the 4.5% Binance-vs-venue
disagreement concentrating on near-ties.

*Open question that decides whether this is real:* the market description names
the Chainlink **Data Stream** (sub-second) while my measurement is of the
on-chain **Data Feed** (33s). If settlement tracks Data Streams, the effect
largely evaporates. Settle that before building anything.

**2. Treat the market price as your baseline, not 50%.** `ask_this_side` is a
well-informed forecast made by people with money at stake. The interesting
question is not "can I predict `resolved_up`" — you can, weakly, from
`move_bp` and `secs_left` alone. It is "can I beat that column by more than the
fee." Score every model against the market's implied probability.

**3. The maker side is completely unexplored here.** Every observation in this
file is taker-side. **Makers pay no fee at all**, which removes the entire
`0.07 × p × (1−p)` term from the breakeven bar — a structural advantage
bigger than most signals. Polymarket also pays liquidity rewards on selected
markets. Neither appears in this dataset.

**4. `hype` is the second-deepest 5-minute book on the venue and appears
nowhere in this data.** Free coverage if it behaves like the majors.

---

## Quickstart

```python
import pandas as pd
df = pd.read_csv("observations.csv")

# Is the market itself well calibrated? (the baseline you must beat)
df["bucket"] = (df.ask_this_side * 20).round() / 20
cal = df.groupby("bucket").agg(n=("this_side_won", "size"),
                               actual=("this_side_won", "mean"))
print(cal[cal.n > 200])          # actual should track the bucket price

# How much does the chart alone tell you, and when?
df["lead"] = df.move_bp * df.side.map({"Up": 1, "Down": -1})
print(df[df.horizon == "5m"].groupby(
    [pd.cut(df.secs_left, [0, 60, 120, 200, 300]),
     pd.cut(df.lead, [-1e9, -4, 0, 4, 1e9])]).this_side_won.agg(["size", "mean"]))
```

Any cell where `mean` beats `bucket price + 0.07·p·(1−p)` on a large `n`
**and holds in both halves of the sample by time** is worth a closer look.
Most won't.

---

## Honest caveats

- **Two weeks, one regime.** Mid-August 2026 crypto. That is not enough to
  establish that anything persists, and forward data always feels more
  conclusive than it is.
- **Coverage is not an experimental design.** Observations cluster where my
  collectors happened to be looking — mostly 120–300s left. `secs_left` is
  therefore confounded with everything else. Don't read cross-`secs_left`
  comparisons as causal.
- **`sigma` is my estimate, not a market observable.** Treat it as a feature
  someone else computed with unknown quality; consider recomputing your own.
- **Book columns are a single snapshot** at observation time, not a full book
  or a time series.
- **Nothing here is investment advice**, and I'm not a licensed advisor. Sizing
  and risk are your own calls. My own hardest-won lesson is in trap #4: if the
  P&L doesn't reconcile to actual cash in the account, the P&L is wrong.
