# Repainting — Every Cause, Every Fix

Repainting is when a script's historical plot disagrees with what a live trader would have seen. ~95% of published TradingView scripts repaint in some form. The principle: ask "would I have seen this information live, on the bar it is plotted on?" If no, the script repaints.

## Contents

- The taxonomy — historical vs realtime repainting
- Causes and fixes — 9 root causes with code
- The barstate variables — full reference
- Verifying non-repainting behavior
- When repainting is acceptable

## The taxonomy

There are two kinds:

- **Historical repainting** — past plots literally change as new bars form. A signal "appears" three bars after the fact because the script peeks ahead.
- **Realtime repainting** — the current unconfirmed bar flickers with each tick. A buy signal turns on at mid-bar then disappears at close. The flicker is invisible on history (where every bar is already closed) but real in live trading.

A script can suffer from one, the other, or both.

## Causes and fixes

### 1. Signals using live (unconfirmed) `close`, `high`, `low`

**Cause.** On the current bar, `close`, `high`, `low` change with every tick. A crossover triggered at mid-bar may un-trigger by close.

```pine
// REPAINTS on realtime bars — historical bars don't show the flicker
ma = ta.ema(close, 5)
xUp = ta.crossover(close, ma)
plot(xUp ? 1 : 0)
```

**Fixes** (pick one based on intent):

- **Wait for close.** Gate on `barstate.isconfirmed`:
  ```pine
  xUp = ta.crossover(close, ma) and barstate.isconfirmed
  ```
- **Use the previous bar's confirmed values.**
  ```pine
  xUp = ta.crossover(close[1], ma[1])
  ```
- **Use the open of the new bar against the prior bar's MA.** Acceptable when the signal is supposed to act on the next bar's open.
  ```pine
  xUp = ta.crossover(open, ma[1])
  ```

### 2. `request.security()` with `lookahead_on` and no offset

**Cause.** `lookahead_on` lets the function return the higher-timeframe bar's *eventual* close before that bar has closed in real chart time. On history this leaks future data.

```pine
// FUTURE LEAK
htfClose = request.security(syminfo.tickerid, "D", close, lookahead = barmerge.lookahead_on)
```

**Fix.** Always pair `lookahead_on` with a `[1]` offset on the expression — that pulls the *previously closed* HTF bar, which is always knowable.

```pine
htfClose = request.security(syminfo.tickerid, "D", close[1], lookahead = barmerge.lookahead_on)
```

Or use the canonical PineCoders wrapper everywhere:

```pine
//@function Non-repainting higher-timeframe request.
f_secure(simple string sym, simple string tf, series float expr) =>
    request.security(sym, tf, expr[1], lookahead = barmerge.lookahead_on)
```

The combination of `[1] + lookahead_on` is non-obvious: lookahead_on alone leaks the future, `[1]` alone forces realtime to lag a bar behind history (also wrong), but together they produce matching historical and realtime values that always reference the last confirmed HTF bar.

### 3. `request.security()` of a lower timeframe

**Cause.** `request.security(syminfo.tickerid, "1", close)` returns *one* 1-minute bar per chart bar — but which one differs between history and realtime. Historically you get the last intrabar; in realtime you get the latest tick.

**Fix.** Use `request.security_lower_tf()` instead, which returns an array of all intrabar values:

```pine
intrabarCloses = request.security_lower_tf(syminfo.tickerid, "1", close)
// intrabarCloses is array<float> — one element per intrabar
```

Iterate the array for intrabar analysis (volume profiles, footprint data, intrabar high/low/POC).

### 4. `varip` declarations

**Cause.** `varip` persists across intrabar ticks. Realtime accumulates state per tick; history has no ticks, so the two regimes diverge.

```pine
varip int tickCount = 0
tickCount += 1   // Realtime: increments on every tick. History: increments once per bar.
```

**Fix.** Use `varip` only when you genuinely need per-tick accumulation (tick volume, time-since-last-tick). Document the repainting cost in a header comment. Prefer `var` for everything else.

### 5. `barstate.isnew`

**Cause.** Fires at bar *open* in realtime, at bar *close* on history. Same code, different timing.

**Fix.** Use `barstate.isconfirmed` (fires at bar close in both regimes) for closing-bar logic. Use `ta.change(time)` if you need an "every new bar" detector that fires consistently.

### 6. `timenow`

**Cause.** Returns the current wall-clock time in realtime; on history there is no equivalent. Comparisons against `timenow` always go one way on history, both ways in realtime.

**Fix.** Compare against `time` (the bar's open time) instead. If you need an "older than N seconds" check that works historically, derive it from `time + timeframe.in_seconds(timeframe.period) * 1000`.

### 7. Strategies with `calc_on_every_tick = true`

**Cause.** Strategy logic runs every tick in realtime but only once per bar on history. Same code, two different execution streams.

**Fix.** Leave `calc_on_every_tick = false` (the default) unless you have a specific reason. If you must use `true` (e.g. tick-level stop-out), set `process_orders_on_close = true` to at least make order fills deterministic.

### 8. Plotting in the past

**Cause.** Detecting an event N bars after it occurred and drawing a label at the past bar misleads users into thinking the signal was visible at that time.

```pine
// Pivot high is only detectable 5 bars AFTER it occurred.
pHi = ta.pivothigh(5, 5)
if not na(pHi)
    label.new(bar_index - 5, pHi, "PH", style = label.style_label_down)
```

**Fix.** Two options:

- **Honest delay.** Draw the label at the current bar with an annotation: `label.new(bar_index, pHi, "PH 5 bars ago")`.
- **Optional past-plot.** Expose a boolean input `plotInThePast` so the user opts in:
  ```pine
  plotInThePast = input.bool(false, "Plot pivots at their actual bar (visual only — signal lagged 5 bars)")
  offset = plotInThePast ? 5 : 0
  label.new(bar_index - offset, pHi, "PH")
  ```

### 9. Dataset variations

**Cause.** Chart history start depends on the user's subscription tier; exchanges occasionally revise historical bars; some pre-market/extended-session bars are added later. Your script computed on history H1 may differ on history H2.

**Fix.** Not your script's fault, but mitigate by:

- Avoiding hardcoded `bar_index == 5000` style references; use date-based anchors with `timestamp("2024-01-01")`.
- Not reading too far back: `[10000]` indexing is fragile, and `max_bars_back` may not save you.

## The barstate variables — full reference

| Variable | Meaning | Repaint-safe? |
|----------|---------|--------------|
| `barstate.isconfirmed` | True only on the bar's final execution (after close on realtime, always on history) | **YES — use for signals** |
| `barstate.islast` | True on the rightmost chart bar | Yes (positional only) |
| `barstate.isrealtime` | True for all realtime bars (post-load) | Diagnostic only |
| `barstate.ishistory` | True for all historical bars | Diagnostic only |
| `barstate.isnew` | True at start of each new bar | **NO — see cause 5** |
| `barstate.isfirst` | True on bar 0 | Yes |
| `barstate.islastconfirmedhistory` | True on the last bar of history (just before realtime begins) | Yes |

## Verifying non-repainting behavior

When you suspect a script repaints, test it this way:

1. Add the script to a chart, let it run live for an hour.
2. Take a screenshot.
3. Reload the chart (or hard-refresh). Pine re-runs the entire history including bars that were realtime an hour ago.
4. Compare: do signals plotted on the now-historical bars match the screenshot?

If they differ, the script repaints. If they match, you're clean.

Faster check during development: add Pine Logs at signal points and observe whether they fire at bar close consistently:

```pine
if signal
    log.info("Signal at bar {0,number,#}, time {1}, confirmed = {2}", bar_index, time, barstate.isconfirmed)
```

## When repainting is acceptable

Not all repainting is bug-territory. Acceptable cases:

- The script is explicitly an "early warning" indicator that *uses* live-bar data to give the trader a heads-up before close. Document this in the title: "RSI Live Warning (repaints — close-only confirmation comes later)".
- Visual indicators that don't drive automated decisions (zigzags, fractals drawn after-the-fact for chart annotation).
- Backtests where the trader explicitly understands signals fire on close and orders fill on the next bar's open (`process_orders_on_close = false`).

Honest disclosure beats stealth repainting every time.
