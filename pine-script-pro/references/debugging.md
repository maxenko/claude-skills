# Debugging Pine Script

Pine's biggest debugging weakness: no breakpoints, no step-through, no console. v6 partially fixes this with Pine Logs and the Pine Profiler.

## Contents

- Pine Logs (v6) — log.info/warning/error, gate-and-ship pattern
- Pine Profiler (v6)
- Visual debugging idioms — data window, heat-maps, label dumps, table dumps
- Common errors decoded — mutable variable, max_bars_back, loop too long, etc.
- "It worked yesterday" debugging
- Sanity-check workflow

## Pine Logs (v6)

The single most useful debugging tool. Open via Editor → More menu → "Pine Logs…".

```pine
debugInput = input.bool(false, "Debug mode", group = "Debug")

if debugInput
    log.info("bar {0}: rsi={1,number,#.##}, signal={2}", bar_index, rsi, signalText)

if condition and barstate.isconfirmed
    log.warning("Unexpected: {0} at price {1,number,#.##}", description, close)

if math.abs(actualValue - expectedValue) > tolerance
    log.error("Calc mismatch: actual={0}, expected={1}", actualValue, expectedValue)
```

**Key features:**
- Clicking a log entry jumps the chart to that bar — fastest "why did this fire here?" workflow.
- Filterable by Info / Warning / Error.
- Supports `str.format()` placeholders.
- Removed messages don't accumulate cost — but leaving thousands of log calls on a long history is slow. Gate behind `debugInput`.

Gate-and-ship pattern:

```pine
debugInput = input.bool(false, "Verbose logging", group = "Debug",
    tooltip = "Enable to populate the Pine Logs pane with per-bar diagnostics. Disable for production.")

if debugInput
    log.info("...")
```

## Pine Profiler (v6)

Editor → More menu → "Pine Profiler". Shows per-line runtime cost. Useful when:

- Script feels sluggish on long history.
- You're optimizing a heavy loop or large `request.security()` payload.
- Drawing-object churn is suspected.

Run, look at the heat-map, optimize the hottest lines first.

## Visual debugging idioms

Before Pine Logs existed, debuggers used the chart itself. Still useful for quick visual inspection:

### Plot intermediate values

```pine
// Just expose the calculation:
plot(myIntermediateVal, "Debug", color = color.purple, display = display.data_window)
```

`display = display.data_window` shows the value in the Data Window pane (hover-readout) but doesn't plot a visible line. Reduces chart clutter.

### Bar-color heat-map

```pine
debugMetric = ...
bgcolor(color.from_gradient(debugMetric, 0, 100, color.new(color.green, 80), color.new(color.red, 80)))
```

Eyeballs an entire history at once.

### Label dump at trigger bar

```pine
if condition
    label.new(bar_index, high,
        str.format("rsi={0,number,#.##}\nma={1,number,#.##}\nvol%={2,number,#.##}", rsi, ma, volPct),
        style = label.style_label_down, color = color.yellow, textcolor = color.black, size = size.small)
```

For complex condition triggers, surface all the input values in one label so you can spot which condition failed.

### Per-bar table dump

```pine
var table debugTbl = table.new(position.top_left, 2, 10, border_width = 1)
if barstate.islast
    table.cell(debugTbl, 0, 0, "rsi");       table.cell(debugTbl, 1, 0, str.tostring(rsi))
    table.cell(debugTbl, 0, 1, "macd");      table.cell(debugTbl, 1, 1, str.tostring(macd))
    table.cell(debugTbl, 0, 2, "atr");       table.cell(debugTbl, 1, 2, str.tostring(atr))
```

Shows the latest-bar state of every interesting variable in a fixed-position panel.

## Common errors decoded

### "Cannot use a mutable variable as an argument..."

You passed a series-qualified value where simple is required (most commonly to `request.security`'s timeframe or symbol parameter — though v6 relaxed this).

**Fix**: wrap the calculation in a function that returns the value, then pass the function call:

```pine
// WRONG (v5):
tf = useFastTf ? "5" : "15"   // mutable
request.security(sym, tf, close)   // error

// FIX:
f_pickTf() => useFastTf ? "5" : "15"
request.security(sym, f_pickTf(), close)
```

(In v6, `request.*` accepts series strings, so this often "just works".)

### "Pine cannot determine the referencing length of a series..."

You used a history-reference depth Pine can't deduce statically, e.g. `close[lookbackInput]` where `lookbackInput` is a user input.

**Fix**: declare `max_bars_back` explicitly on the declaration:

```pine
indicator("...", max_bars_back = 5000)
```

Or call `max_bars_back(close, 5000)` on the specific series.

### "Loop is too long"

A loop exceeded Pine's runtime budget (different limits per loop type — historical iteration is more constrained than per-bar).

**Fix**: vectorize using built-ins (`ta.*` and `math.*` are vastly faster than hand loops) or reduce iterations.

### "Script could not be translated from: <line>"

Parser error. The line indicator is often *after* the actual typo (parser stopped at the next valid-looking token). Look one line above; check for unclosed parens, missing commas in `input.*()` calls, missing closing brace on a function.

### "The 'expression' argument of '...' is ill-formed"

A `na` of unknown type leaked into a typed slot. Usually from an `if`-expression with `na` in one branch:

```pine
// AMBIGUOUS:
x = condition ? someFloat : na
// na has unknown type here.

// FIX:
x = condition ? someFloat : float(na)
// or:
float x = condition ? someFloat : na
```

### "Drawing limit exceeded" / silently disappearing objects

Default 50 lines/labels/boxes. When you exceed, the oldest are auto-deleted. From the user's perspective: history goes blank.

**Fix**: declare limits:

```pine
indicator("...", max_lines_count = 500, max_labels_count = 500, max_boxes_count = 500, max_polylines_count = 100)
```

500 is the hard maximum (100 for polylines). If you need more, you're churning — switch to a `table` or mutate existing objects instead of creating new ones.

## "It worked yesterday" debugging

Three causes:

1. **Data start point changed** — chart history depth depends on user's tier. On a longer history, the first 14 bars of an RSI return `na` further back.
2. **Exchange revision** — exchanges occasionally adjust historical bar prices. Scripts with hardcoded levels can flip.
3. **Repainting realtime → frozen history** — what was unconfirmed yesterday is confirmed today, so a signal that "was there" now exists historically with different timing. Almost always a repainting bug.

## Sanity-check workflow

Before publishing a script:

1. Load it on three symbols and three timeframes. Look at the plots. Sanity-checks fail catastrophically obvious.
2. For strategies, run the Strategy Tester. Look at the equity curve shape — too smooth means overfit.
3. For indicators with alerts, set an alert and let it run an hour. Compare to the historical plots after the hour passes.
4. If the script uses `request.security`, change the chart timeframe and verify the HTF plot still makes sense (stair-step shape, no future-knowledge spikes).
5. If the script uses `varip`, reload the chart and verify history matches what you saw seconds ago.
