# Strategy Development

The Strategy Tester is the only Pine feature that lets you turn an indicator into an executable, measurable trading rule. It also lies enthusiastically to anyone who doesn't configure it honestly.

## Contents

- The non-negotiable preamble — realistic `strategy()` declaration
- Order primitives — entry, exit, close, brackets
- Pyramiding
- Position sizing — fixed, percent-of-equity, risk-per-trade
- The bar-close vs next-open timing question
- Backtest evaluation — what to look at and what to ignore
- Walk-forward / out-of-sample
- Strategy idioms — MA cross, trailing stop, time-based exit
- Common strategy bugs

## The non-negotiable preamble

Every `strategy()` you write starts with this block:

```pine
strategy("Strategy Name", overlay = true,
    initial_capital            = 10000,
    default_qty_type           = strategy.percent_of_equity,
    default_qty_value          = 10,
    commission_type            = strategy.commission.percent,
    commission_value           = 0.05,
    slippage                   = 2,
    pyramiding                 = 0,
    calc_on_every_tick         = false,
    calc_on_order_fills        = false,
    process_orders_on_close    = true,
    use_bar_magnifier          = false,
    margin_long                = 100,
    margin_short               = 100)
```

### What each parameter does and why it matters

| Parameter | Default | Honest setting | Why |
|-----------|---------|----------------|-----|
| `initial_capital` | 1,000,000 | Realistic for the user (10k–100k retail) | Affects compound returns and position sizing. Big capital with `percent_of_equity` masks fee impact. |
| `default_qty_type` | `strategy.fixed` | `strategy.percent_of_equity` or risk-based qty | Percent-of-equity scales naturally. Fixed-share masks compounding. |
| `default_qty_value` | 1 | 10 (= 10% of equity per trade) | The user can override per-`entry`, but default should be sane. |
| `commission_type` | `strategy.commission.percent` | Same | Defaults to 0 — the silent profit killer. |
| `commission_value` | 0 | 0.05–0.1% (crypto: 0.04–0.1%, equities: 0.005–0.01%, varies) | A 500-trade strategy at 0% can flip to losing at 0.1%. Always set. |
| `slippage` | 0 | 1–2 ticks (liquid), 5–10 (illiquid) | Distance between signal price and fill price. |
| `pyramiding` | 1 | 0 unless intentional | Allows N entries in the same direction. `0` = single position. |
| `calc_on_every_tick` | false | **false** | True causes historical vs realtime divergence. Only flip if you need tick-precision stop-outs. |
| `calc_on_order_fills` | false | false | True re-runs the script after each fill, causing re-entry chains. Confusing in backtests. |
| `process_orders_on_close` | false | **true** | When true, market orders fill on the close of the signaling bar (not next bar's open). Eliminates a one-bar lag that makes signals look weaker. |
| `use_bar_magnifier` | false | true on TF ≥ 1h, false otherwise | Simulates intrabar fill prices using lower-TF data. More accurate for stop fills but slower. |

## Order primitives

```pine
// MARKET ORDER:
strategy.entry(id = "Long", direction = strategy.long, qty = na, comment = "Long entry")

// LIMIT ORDER:
strategy.entry(id = "Long", direction = strategy.long, limit = limitPrice)

// STOP ORDER:
strategy.entry(id = "Long", direction = strategy.long, stop = stopPrice)

// BRACKET EXIT (preferred for stop + take-profit pairs):
strategy.exit(id = "Exit Long", from_entry = "Long",
              stop = stopLoss, limit = takeProfit,
              comment_loss = "SL", comment_profit = "TP")

// UNCONDITIONAL CLOSE:
strategy.close(id = "Long", comment = "Signal exit")

// CLOSE ALL:
strategy.close_all(comment = "End of day")
```

**Bracket exits**: `strategy.exit()` with both `stop` and `limit` creates a one-cancels-the-other (OCO) bracket. The first leg to fill cancels the other. This is the right way to do stop + profit-target — *not* two separate `strategy.exit` calls.

## Pyramiding

`pyramiding = N` allows up to N concurrent `strategy.entry()` orders in the same direction. Common usage:

```pine
strategy("Scale-in trend", pyramiding = 3)

if longCondition and strategy.opentrades < 3
    strategy.entry("Long" + str.tostring(strategy.opentrades), strategy.long)
```

Gotcha: `strategy.order()` does NOT honour `pyramiding`. Only `strategy.entry()` does. If you need raw order control (e.g. partial fills, complex grids), use `strategy.order()` and track position manually.

## Position sizing

Three patterns, in order of professionalism:

### 1. Fixed share count (don't)

```pine
strategy("...", default_qty_type = strategy.fixed, default_qty_value = 100)
```

Doesn't compound. Doesn't adjust to price. Avoid except for instruments with fixed lot sizes.

### 2. Percent of equity

```pine
strategy("...", default_qty_type = strategy.percent_of_equity, default_qty_value = 10)
```

Each trade uses 10% of current equity. Compounds naturally. Good default.

### 3. Risk-per-trade with stop distance (professional)

The strategy risks a fixed *fraction of equity* per trade, sized so a stop-loss hit costs that fraction:

```pine
riskPctInput  = input.float(1.0, "Risk per trade %", minval = 0.1, maxval = 5.0)
atrStopMultInput = input.float(2.0, "ATR stop multiplier")

atr = ta.atr(14)
stopDist = atrStopMultInput * atr

// qty so that (qty * stopDist) = (equity * riskPct)
positionQty = (strategy.equity * riskPctInput / 100) / stopDist

if longCondition
    strategy.entry("Long", strategy.long, qty = positionQty)
    strategy.exit("Exit", "Long", stop = close - stopDist, limit = close + stopDist * 2)
```

This is what professional algorithmic traders use. Sizing adapts to volatility automatically.

## The bar-close vs next-open timing question

By default (`process_orders_on_close = false`), a market order from `strategy.entry()` on bar N fills at the open of bar N+1. This is realistic for live trading where the bar's close-signal is detected after the close.

With `process_orders_on_close = true`, the order fills at the close of bar N. This matches what `plot()` of the signal would show — but is slightly optimistic since you couldn't really fill at the exact close in live trading.

Trade-off:
- Set `process_orders_on_close = true` when comparing strategy results to an indicator's plotted signals (so they align visually).
- Leave `false` when modeling realistic live execution.

## Backtest evaluation: what to look at and what to ignore

The Strategy Tester reports many metrics. Honest evaluation:

- **Net profit** is meaningless without commission, slippage, and trade count.
- **Win rate** above 70% on a frequent-trade strategy is almost always overfit.
- **Profit factor** (gross profit / gross loss). Above 1.5 is good; above 3 is suspicious.
- **Max drawdown %** is the single most useful metric. If it exceeds the user's pain tolerance, the strategy fails regardless of total return.
- **Number of trades** — fewer than ~30 trades is statistically insufficient. Below ~100 is shaky.
- **Sharpe / Sortino** — Pine reports these but they're trailing-window estimates; treat as directional, not absolute.

Suspicious patterns that signal overfit:

- Win rate spikes when the strategy is in-sample, drops when out-of-sample.
- Tiny equity dips in the curve — the strategy "always recovers". Indicator overfit to specific bars.
- Performance varies wildly across symbols or timeframes the strategy claims to be agnostic to.
- A single huge trade contributes most of the profit.

When delivering a strategy, mention these caveats if the backtest is unusually pretty.

## Walk-forward / out-of-sample

Pine doesn't have built-in walk-forward optimization. The cheap manual version:

1. Optimize the strategy on the first 70% of available history.
2. Note the chosen parameters.
3. Run the same parameters on the remaining 30%.
4. If performance degrades sharply, the parameters were overfit.

Implement by date-gating:

```pine
trainEndInput = input.time(timestamp("2024-01-01"), "Training set end")
inTrainingPeriod = time < trainEndInput

// Only allow entries in the training period, OR
// run the strategy on both periods and compare via two separate add-to-chart calls.
```

## Strategy idioms

### Moving-average crossover with bracket exit

```pine
//@version=6
strategy("MA Cross", overlay = true,
    initial_capital = 10000, default_qty_type = strategy.percent_of_equity, default_qty_value = 10,
    commission_type = strategy.commission.percent, commission_value = 0.05, slippage = 2)

fastLen = input.int(10, "Fast")
slowLen = input.int(30, "Slow")
atrMult = input.float(2.0, "ATR stop multiplier")

fast = ta.sma(close, fastLen)
slow = ta.sma(close, slowLen)
atr  = ta.atr(14)

longSig  = ta.crossover(fast, slow)  and barstate.isconfirmed
shortSig = ta.crossunder(fast, slow) and barstate.isconfirmed

if longSig
    strategy.entry("Long", strategy.long)
    strategy.exit("LX", "Long", stop = close - atr * atrMult, limit = close + atr * atrMult * 2)

if shortSig
    strategy.entry("Short", strategy.short)
    strategy.exit("SX", "Short", stop = close + atr * atrMult, limit = close - atr * atrMult * 2)

plot(fast, "Fast", color.blue)
plot(slow, "Slow", color.orange)
```

### Trailing stop

```pine
trailPctInput = input.float(2.0, "Trail %") / 100

if strategy.position_size > 0
    trailPrice = strategy.position_avg_price * (1 - trailPctInput)
    strategy.exit("Trail", "Long", stop = math.max(trailPrice, close * (1 - trailPctInput)))
```

### Time-based exit (close at end of session)

```pine
sessionEndInput = input.session("1530-1600", "Force-close session")
if not na(time(timeframe.period, sessionEndInput))
    strategy.close_all(comment = "EOD")
```

## Common strategy bugs

1. **Order fills on the wrong bar.** Without `process_orders_on_close = true`, entries fill on next bar's open — the indicator's plot of the entry signal misaligns with the executed fill.
2. **Pyramiding off by one.** `strategy.opentrades` counts the open position; `pyramiding` is the max allowed. Off-by-one when adding the entry id suffix.
3. **Closing the wrong entry.** With pyramiding, `strategy.close("Long")` closes ALL entries tagged "Long". Use distinct ids per scale-in.
4. **Repainting strategy signals.** Same as any indicator: gate on `barstate.isconfirmed` if `calc_on_every_tick = false` is not enough, especially when using HTF data.
5. **Bracket exit with stale stop.** `strategy.exit` placed once outside the entry block uses the entry-bar prices forever. Place it in the same `if entryCondition` block.
