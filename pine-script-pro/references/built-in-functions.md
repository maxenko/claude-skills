# Built-in Functions

Pine's standard library is large. This is a high-yield subset — the functions you reach for in 90% of scripts, with their common pitfalls.

## Contents

- `ta.*` — technical analysis (MAs, oscillators, crossovers, extremes, pivots, volume, linreg)
- `math.*` — math functions
- `request.*` — multi-symbol and multi-timeframe data
- `str.*` — strings
- `array.*` — arrays
- `map.*` — maps
- `color.*`
- `chart.point` — anchor points for v6 drawings
- `timeframe.*`
- `syminfo.*`
- `log.*` — Pine Logs (v6)
- `alert(), alertcondition()`
- Naming gotchas
- Functions that return `na` and what to do

## `ta.*` — technical analysis

### Moving averages

```pine
ta.sma(source, length)       // Simple
ta.ema(source, length)       // Exponential — most common
ta.wma(source, length)       // Linear-weighted
ta.rma(source, length)       // Running MA (Wilder's smoothing — used in RSI, ATR)
ta.vwma(source, length)      // Volume-weighted
ta.hma(source, length)       // Hull
ta.swma(source)              // 4-period symmetrically-weighted (fixed length)
```

**Gotcha**: `ta.rma` is *not* the same as `ta.ema`. RMA uses `1/length`, EMA uses `2/(length+1)`. Wilder's RSI and ATR use RMA. Many community indicators wrongly substitute EMA.

### Oscillators and indicators

```pine
ta.rsi(source, length)
ta.macd(source, fastLen, slowLen, signalLen)   // returns [macdLine, signalLine, histogram]
ta.stoch(source, high, low, length)
ta.cci(source, length)
ta.mfi(source, length)
ta.atr(length)
ta.tr(handle_na = false)                       // true range
ta.bb(source, length, mult)                    // returns [basis, upper, lower]
ta.kc(source, length, mult)                    // Keltner — returns [basis, upper, lower]
ta.adx(diLength, adxLength)                    // v6 — returns [dmiPlus, dmiMinus, adx]
ta.dmi(diLength, adxLength)
```

### Crossovers and changes

```pine
ta.crossover(a, b)            // true when a crosses above b
ta.crossunder(a, b)
ta.cross(a, b)                // either direction
ta.change(source, length = 1) // source[0] - source[length] — use for "did it change"
```

### Extremes and pivots

```pine
ta.highest(source, length)              // highest value over last `length` bars (excludes current if not passed)
ta.lowest(source, length)
ta.highestbars(source, length)          // offset to the bar where the highest occurred (0 = current bar)
ta.lowestbars(source, length)
ta.pivothigh(leftbars, rightbars)       // returns `na` until `rightbars` bars after the pivot — by definition non-realtime
ta.pivotlow(leftbars, rightbars)
ta.barssince(condition)                 // bars since `condition` was last true; `na` if never
ta.valuewhen(condition, source, occurrence)  // value of `source` at the Nth most recent `true` of `condition`
```

**Pivot gotcha**: `ta.pivothigh(5, 5)` cannot detect a pivot until 5 bars after it occurred. Drawing it at `bar_index` is honest; drawing it at `bar_index - 5` retrofits and misleads users into thinking the signal was visible at that bar. If you do retrofit, expose a boolean input so the user opts in.

### Volume

```pine
ta.cumvolume                    // cumulative volume since chart start
ta.vwap                          // session-anchored VWAP
ta.vwap(source, anchor)         // anchored VWAP with custom reset
ta.obv                          // On-Balance Volume
ta.accdist                      // Accumulation/Distribution
```

### Linear regression

```pine
ta.linreg(source, length, offset)        // value of linear regression line at offset
ta.correlation(a, b, length)
ta.stdev(source, length, biased = true)
ta.variance(source, length)
```

## `math.*` — math functions

```pine
math.abs, math.sign, math.floor, math.ceil, math.round, math.round_to_mintick
math.min, math.max, math.avg, math.sum
math.pow, math.sqrt, math.exp, math.log, math.log10
math.sin, math.cos, math.tan, math.asin, math.acos, math.atan
math.random(min, max, seed)     // pseudo-random; seed for reproducibility
math.todegrees, math.toradians
```

**`math.round_to_mintick(x)`** rounds to the symbol's tick size. Use for any price level you'll display or place as an order — prevents fractional-tick errors.

## `request.*` — multi-symbol and multi-timeframe data

```pine
// Higher timeframe — wrap with f_secure(expr[1], lookahead_on) for non-repainting use:
request.security(symbol, timeframe, expression,
    gaps             = barmerge.gaps_off,
    lookahead        = barmerge.lookahead_off,
    ignore_invalid_symbol = false,
    currency         = currency.NONE)

// Lower timeframe — returns array of intrabars:
request.security_lower_tf(symbol, timeframe, expression)

// Other:
request.dividends(symbol, field, gaps = barmerge.gaps_off)
request.splits(symbol, field)
request.earnings(symbol, field)
request.financial(symbol, financial_id, period)
request.economic(country, field)
request.quandl(...)   // deprecated; data sources changed
```

## `str.*` — strings

```pine
str.tostring(value)                                // works for any type
str.tostring(value, format)                        // format string: "0.00", "#.###", "$#,##0.00"
str.format(template, args...)                      // {0,number}, {0,number,#.##}, {0,date,yyyy-MM-dd}
str.length(s)
str.contains(s, substr)
str.replace(s, target, replacement, occurrence)
str.replace_all(s, target, replacement)
str.split(s, separator)                            // returns array<string>
str.upper, str.lower
str.startswith, str.endswith
```

**`str.format`** is the cleanest way to build alert messages with embedded values:

```pine
alertMsg = str.format("{0} @ {1,number,#.##} | RSI={2,number,#.#}", syminfo.ticker, close, rsi)
alert(alertMsg, alert.freq_once_per_bar_close)
```

## `array.*` — arrays

```pine
arr = array.new<float>()                     // typed empty array
arr = array.from(1.0, 2.0, 3.0)              // typed from values
array.push(arr, value)
array.pop(arr)
array.shift(arr)                              // pop from front
array.unshift(arr, value)                     // push to front
array.get(arr, index), array.set(arr, index, value)
array.size, array.first, array.last
array.sum, array.avg, array.median, array.stdev, array.variance
array.min, array.max
array.includes, array.indexof, array.lastindexof
array.sort, array.reverse, array.slice
array.concat(a, b), array.join(arr, separator)
array.copy(arr)                              // shallow copy — arrays are references
```

Arrays passed to functions are by reference. Mutations propagate. Use `array.copy()` if you need independence.

## `map.*` — maps

```pine
m = map.new<string, float>()
map.put(m, "key", 1.0)
map.get(m, "key")
map.contains(m, "key")
map.remove(m, "key")
map.size, map.keys, map.values
map.copy(m)
```

Iterate with destructuring: `for [k, v] in m`.

## `color.*`

```pine
color.new(base, transparency)                       // transparency 0–100; 0 = opaque
color.rgb(r, g, b, transparency = 0)
color.from_gradient(value, min, max, low_color, high_color)
color.r(c), color.g(c), color.b(c), color.t(c)
```

## `chart.point` — anchor points for v6 drawings

```pine
chart.point.from_index(bar_index, price)            // x = bar index
chart.point.from_time(time, price)                  // x = absolute time
chart.point.now(price)                              // current bar
chart.point.new(time, bar_index, price)             // explicit
```

Used by `polyline.new()` and increasingly by other v6 drawing functions for time-vs-bar-index disambiguation.

## `timeframe.*`

```pine
timeframe.period                          // current chart's timeframe string ("D", "60", "1")
timeframe.multiplier                      // numeric part
timeframe.isintraday, timeframe.isdaily, timeframe.isweekly, timeframe.ismonthly
timeframe.in_seconds(tf = timeframe.period)
```

Useful for adaptive scripts:

```pine
defaultLength = timeframe.isdaily ? 14 : timeframe.isintraday ? 20 : 10
```

## `syminfo.*`

```pine
syminfo.ticker, syminfo.tickerid, syminfo.type
syminfo.basecurrency, syminfo.currency
syminfo.mintick, syminfo.pointvalue
syminfo.session, syminfo.timezone
syminfo.prefix
```

`syminfo.mintick` is essential for price-rounding any displayed levels: `math.round_to_mintick(myLevel)`.

## `log.*` — Pine Logs (v6)

```pine
log.info(message)
log.warning(message)
log.error(message)
```

Messages support `str.format()` placeholders. Open Pine Logs from the editor's More menu. Click any entry to jump to its bar. Remove from production code or gate behind a debug input.

## `alert(), alertcondition()`

```pine
alertcondition(condition, title = "Alert", message = "")    // legacy — const string message, {{placeholders}} only
alert(message, freq)                                         // modern — series string message, dynamic values OK
```

Frequency constants:

```pine
alert.freq_once_per_bar        // fires once per bar even if condition true on multiple ticks
alert.freq_once_per_bar_close  // fires only when bar closes — non-repainting for live
alert.freq_all                 // every tick
```

Default for non-repainting alerts: `alert.freq_once_per_bar_close`.

## Naming gotchas

- `close` is the current bar's close (live or final). `close[1]` is the previous bar's close (always final).
- `time` is the bar's open time. `time_close` is the bar's close time.
- `volume` is the bar's volume (live or final).
- `bar_index` is 0-based from the chart's first bar (which is *not* bar 0 of the symbol — it depends on user's history depth).

## Functions that return `na` and what to do

Many functions return `na` on early bars (e.g. `ta.rsi` returns `na` for the first `length` bars). When using these:

```pine
rsi = ta.rsi(close, 14)
rsiSafe = nz(rsi, 50)                  // replace na with 50
// or
rsiSafe = na(rsi) ? 50.0 : rsi
```

`nz(value, default = 0)` is the standard idiom. Use `na(x)` to test for `na`.

Plotting `na` shows nothing — which is the right behavior for "no data yet" zones. Avoid `nz`-ing values that go to `plot()` unless you want a flat line during warmup.
