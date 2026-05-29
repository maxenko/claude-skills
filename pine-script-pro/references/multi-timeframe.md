# Multi-Timeframe and Multi-Symbol

`request.security()` is Pine's most misused feature. This file is the reference.

## Contents

- Mental model — how context-switching works
- The two safe patterns — non-repainting vs intentional-live
- Bundling multiple values
- Lower-timeframe data: `request.security_lower_tf`
- `gaps` and `lookahead` arguments
- Multi-symbol screener pattern
- Timeframe strings
- Common mistakes

## Mental model

`request.security(symbol, timeframe, expression)` runs `expression` in the context of `(symbol, timeframe)`, then returns the result to your script's context (the chart's symbol + timeframe).

Key insight: `expression` is evaluated in the *requested* context's bar timeline, but the value gets *mapped onto* your chart's bar timeline. This mapping is where lookahead and repainting issues live.

## The two safe patterns

### Pattern 1: Non-repainting HTF request (the one you'll use 95% of the time)

```pine
//@function Returns the value of `expr` from the most-recently-closed bar
// of `tf` on `sym`. Non-repainting, consistent between history and realtime.
f_secure(simple string sym, simple string tf, series float expr) =>
    request.security(sym, tf, expr[1], lookahead = barmerge.lookahead_on)

// Usage:
htfClose  = f_secure(syminfo.tickerid, "D",  close)
htfRsi    = f_secure(syminfo.tickerid, "4H", ta.rsi(close, 14))
```

Why this works:
- `expr[1]` means we ask for the *previous* bar's value in the HTF context, which is always closed.
- `lookahead_on` ensures the realtime mapping aligns with the historical mapping (otherwise realtime would lag a bar).

This is the canonical PineCoders pattern. Use unmodified.

### Pattern 2: Repainting / "live" HTF request — only when intentional

```pine
htfCloseLive = request.security(syminfo.tickerid, "D", close)
```

The current daily close moves all day as the daily bar forms. Realtime shows this; history shows only the final close. Repaints — use only when the user wants "live HTF value", and label the script "(live mode — repaints)".

## Bundling multiple values

Each `request.security()` call has overhead. Bundle related values:

```pine
//@function Bundles four values from one HTF context.
f_secureBundle(simple string sym, simple string tf) =>
    [o, h, l, c] = request.security(sym, tf,
        [open[1], high[1], low[1], close[1]],
        lookahead = barmerge.lookahead_on)
    [o, h, l, c]

[htfO, htfH, htfL, htfC] = f_secureBundle(syminfo.tickerid, "D")
```

10x cheaper than 4 separate calls. Critical for screeners that read many values per symbol.

## Lower-timeframe data: `request.security_lower_tf`

For intrabar analysis (footprint, volume profile, tick-level patterns), use the dedicated function. It returns an *array* of intrabar values, one element per intrabar within the current chart bar.

```pine
// Get all 1-minute closes within the current chart bar:
intrabarCloses = request.security_lower_tf(syminfo.tickerid, "1", close)

// Sum 1-minute volume by direction within current bar:
[upVol, downVol] = request.security_lower_tf(syminfo.tickerid, "1",
    [close > open ? volume : 0.0, close < open ? volume : 0.0])
// Both are array<float> with one element per 1-min bar inside the current chart bar.
```

**Never** pass a smaller-than-chart timeframe to `request.security()` (the non-`_lower_tf` version). It returns only one of the intrabars per chart bar — different one in history vs realtime. Use `_lower_tf` always for LTF.

## `gaps` and `lookahead` arguments

`gaps` controls what to return when no HTF bar exists for the current chart bar (e.g. weekend):

- `barmerge.gaps_off` (default) — repeat the previous value
- `barmerge.gaps_on` — return `na`

Almost always want `gaps_off`. Set explicitly anyway for clarity.

`lookahead` controls timeline alignment:

- `barmerge.lookahead_off` (default) — safe but lag-y in realtime
- `barmerge.lookahead_on` — must pair with `[1]` offset to avoid future leak (use the `f_secure` wrapper, never inline the raw call)

## Multi-symbol screener pattern

```pine
//@version=6
indicator("Multi-Symbol Dashboard", overlay = true)

symbols = array.from("NASDAQ:AAPL", "NASDAQ:MSFT", "NASDAQ:GOOG", "NASDAQ:AMZN")

f_secureRsi(simple string sym) =>
    request.security(sym, timeframe.period, ta.rsi(close, 14)[1], lookahead = barmerge.lookahead_on)

var table dash = table.new(position.top_right, 2, array.size(symbols) + 1, border_width = 1)

if barstate.islast
    table.cell(dash, 0, 0, "Symbol", bgcolor = color.gray, text_color = color.white)
    table.cell(dash, 1, 0, "RSI",    bgcolor = color.gray, text_color = color.white)
    for i = 0 to array.size(symbols) - 1
        sym = array.get(symbols, i)
        r = f_secureRsi(sym)
        c = r > 70 ? color.red : r < 30 ? color.green : color.white
        table.cell(dash, 0, i + 1, sym)
        table.cell(dash, 1, i + 1, str.tostring(r, "#.#"), text_color = c)
```

Notes:
- `request.security()` calls compile to per-symbol script runs server-side. Each adds overhead. The historical limit was ~40 security calls per script.
- Update the table only on `barstate.islast` — no point updating per bar.

## Timeframe strings

Valid `timeframe` strings (the second argument to `request.security`):

| Form | Meaning |
|------|---------|
| `"1"` | 1 minute |
| `"5"`, `"15"`, `"30"`, `"60"`, `"240"` | N minutes |
| `"D"` | 1 day |
| `"3D"`, `"5D"` | N days |
| `"W"` | 1 week |
| `"M"` | 1 month |
| `"3M"`, `"6M"`, `"12M"` | N months |
| `"1S"`, `"5S"`, `"30S"` | N seconds (paid tiers only) |

For dynamic timeframes from user input:

```pine
tfInput = input.timeframe("D", "Higher timeframe")
htfClose = f_secure(syminfo.tickerid, tfInput, close)
```

`input.timeframe()` validates the format and offers a dropdown of common values.

## Common mistakes

1. **Naked `request.security(sym, "D", close)` for non-repainting needs.** Repaints by default. Always use the `f_secure` wrapper.
2. **Pulling a smaller-than-chart timeframe via `request.security()`.** Returns only one intrabar; differs history vs realtime. Use `request.security_lower_tf`.
3. **Many separate security calls.** Each is expensive. Bundle.
4. **Computing inside the expression.** Acceptable for simple stuff; for heavy logic, define a function and pass the function call as the expression — Pine evaluates it in the requested context.
5. **Plotting HTF close as if it's a chart-TF series.** The HTF close updates only when the HTF bar closes; plotting it gives stair-step shapes that surprise users. Either accept the staircase visually or smooth via `ta.barssince` logic.
6. **Forgetting `currency = currency.USD`** when comparing across currencies. The default is the symbol's own currency.
