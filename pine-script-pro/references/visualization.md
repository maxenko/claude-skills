# Visualization Toolkit

Pine's drawing primitives overlap. Choosing the right one is the difference between a clean indicator and one that hits drawing-budget limits and disappears.

## Contents

- The taxonomy — primitive comparison table
- `plot()` — the workhorse
- `fill()` — region fills
- `plotshape() / plotchar() / plotarrow()` — bar markers
- `label.new()` — dynamic per-bar text
- `line.new()`, `box.new()` — trendlines and zones
- `polyline.new()` (v6) — multi-point shapes
- `table.new()` — chart-anchored UI
- Colors
- Text formatting (v6)
- Performance and budget pitfalls
- Layering and z-order
- Idiomatic patterns — order blocks, S/R, heat-maps, screeners

## The taxonomy

| Primitive | Tethered to | Body sizing | Budget? | Mutable after creation? |
|-----------|-------------|-------------|---------|-------------------------|
| `plot()` | Bar series | Line thickness in px | No | No (one plot, all bars) |
| `plotshape() / plotchar() / plotarrow()` | Specific bar | **Pixel** (discrete: tiny…huge) | No | No |
| `bgcolor()` / `barcolor()` / `plotcandle()` | Bar | Bar-wide | No | No |
| `fill()` | Two plots or two hlines or two lines | Chart-coord (price-bounded) | No | No |
| `hline()` | Price level (axis) | Line thickness in px | No | No (return value can be `fill`-ed) |
| `line.new()` | Two points (x,y, x,y) | Endpoints chart-coord; thickness px | **Yes** (default 50, max 500) | Yes (`line.set_*`) |
| `label.new()` | One point | **Pixel** body; anchor chart-coord | **Yes** (default 50, max 500) | Yes |
| `box.new()` | Two points (corners) | **Chart-coord** (price × bars) | **Yes** (default 50, max 500) | Yes |
| `polyline.new()` | Array of `chart.point` | **Chart-coord** (price × bars/time) | **Yes** (default 50, max 100) | No (read-only after construction) |
| `table.new()` | **Viewport position** (not chart) | Pixel | **Yes** (max 9 tables) | Yes (`table.cell()`) |
| `linefill.new()` | Two `line` IDs | Tied to lines | Tied to lines | Yes |

### Anchoring vs sizing — why this is the #1 source of "the marker doesn't behave right"

Two independent properties get conflated:

- **Anchoring** answers *"as the chart scrolls or zooms horizontally, does the drawing move with the bar?"* Almost everything except `table` is bar-anchored. Anchoring is about position only.
- **Body sizing** answers *"as the chart zooms vertically or horizontally, does the drawing itself get visually larger or smaller?"* `plotshape`/`plotchar`/`plotarrow`/`label` bodies are **pixel-sized** — they stay the same physical size at every zoom. `box`/`polyline` bodies are **chart-coord sized** — they scale with zoom on both axes because their dimensions live in price/time units.

When a user says any of "respect scale", "scale with the chart", "stick to the chart", "stay glued to the bar":

| User probably means | Right primitive |
|--------------------|-----------------|
| "Travel with the bar across zoom/pan/resize" | Any bar-anchored primitive (almost all of them). |
| "Stay the same physical size on screen" | `plotshape`/`plotchar`/`label`. |
| "Grow when I zoom in, shrink when I zoom out" | `box`/`polyline` with dimensions in price × bars/time. |
| "Offset from the bar should be a price amount, not a fixed pixel gap" | Custom y-coordinate (e.g. `high + atr * 0.5`) with `yloc.price`, in any primitive. |

If the ask is ambiguous, the single most-useful clarifying question is:

> "When you zoom in vertically, should the circle (a) stay the same visual size and just move with the bar, or (b) grow taller proportionally with the bars?"

That collapses three plausible interpretations into one decision.

## `plot()` — the workhorse

```pine
plot(series  = close,
     title   = "Price",
     color   = color.blue,
     linewidth = 2,
     style   = plot.style_line,         // line, stepline, histogram, columns, area, areabr, cross, circles, linebr
     linestyle = plot.style_line,       // v6: line, dashed, dotted
     offset  = 0,                       // shift plot N bars left/right
     trackprice = false,
     editable = true)
```

Series-coloured plots:

```pine
plot(close, color = close > open ? color.green : color.red)
```

Histogram from zero:

```pine
plot(macd - signal, style = plot.style_histogram, color = (macd > signal) ? color.green : color.red)
```

**Plots cost nothing from the drawing budget.** Default per-script plot limit is ~64; use `display = display.none` on plots whose only purpose is feeding `fill()`.

## `fill()` — region fills

Three forms:

```pine
// Between two plot() returns:
upper = plot(ta.sma(high, 20))
lower = plot(ta.sma(low, 20))
fill(upper, lower, color = color.new(color.blue, 90))

// Between two hlines:
hHi = hline(80)
hLo = hline(20)
fill(hHi, hLo, color = color.new(color.red, 95))

// Between two line.new() objects:
fill(myLine1, myLine2, color = ...)
```

Gradient fill (price colored by distance from MA):

```pine
ma = ta.sma(close, 20)
upper = plot(ma + ta.stdev(close, 20))
lower = plot(ma - ta.stdev(close, 20))
fill(upper, lower, top_color = color.new(color.green, 50), bottom_color = color.new(color.red, 50))
```

## `plotshape()` / `plotchar()` / `plotarrow()` — bar markers

Use for "show an arrow on every buy signal" patterns. Free from the label budget.

```pine
plotshape(longSignal, style = shape.triangleup,  location = location.belowbar, color = color.green, text = "Buy")
plotchar(shortSignal, char = "▼",  location = location.abovebar, color = color.red, text = "Sell")
plotarrow(rsi - 50)   // positive = up arrow, negative = down arrow, magnitude = length
```

Prefer these over `label.new()` when the marker is uniform across all signals. Reserve labels for dynamic per-signal text.

### `location` parameter — where the shape lands

| Value | Y position | Y argument used? |
|-------|------------|------------------|
| `location.abovebar` | Just above the bar's high, with a small TV-managed pixel gap | Ignored |
| `location.belowbar` | Just below the bar's low, with a small TV-managed pixel gap | Ignored |
| `location.top` / `location.bottom` | Top/bottom of the visible pane | Ignored |
| `location.absolute` | At the y value you supply, interpreted as price | **Required** — series float price |

For "marker glued just above/below the bar" with no fuss, use `location.abovebar`/`location.belowbar` and pass `na` (or any value — it's ignored). For a custom price-space offset (e.g. ATR-based gap), use `location.absolute` with `series = isAbove ? high + offset : low - offset`.

### `size` is a `const string` — not driveable from input switches

The `size` parameter of `plotshape()`/`plotchar()`/`plotarrow()` is typed `const string`. A `switch` whose selector is `input.string` returns `simple string`, which is one qualifier looser than `const`, and Pine rejects it:

```
CE10123 — An argument of "simple string" type was used but a "const string" is expected
```

Three workable patterns:

```pine
// (a) Hardcode.
plotshape(signal, style = shape.circle, size = size.tiny, …)

// (b) Branch with the literal at each call site.
sizePref = input.string("Tiny", "Size", options = ["Tiny", "Small", "Normal"])
plotshape(signal and sizePref == "Tiny",   style = shape.circle, size = size.tiny,   …)
plotshape(signal and sizePref == "Small",  style = shape.circle, size = size.small,  …)
plotshape(signal and sizePref == "Normal", style = shape.circle, size = size.normal, …)

// (c) Use label.new() instead — its `size` parameter accepts series string.
label.new(bar_index, na, "", yloc = yloc.belowbar, style = label.style_circle, size = sizeFromInput)
```

`label.new()` is the escape hatch when the user wants a runtime-selectable size on a circle/cross/triangle marker.

## `label.new()` — dynamic per-bar text

```pine
if signal
    label.new(
        x        = bar_index,
        y        = high,
        text     = str.format("RSI = {0,number,#.##}", rsi),
        yloc     = yloc.abovebar,
        style    = label.style_label_down,
        color    = color.new(color.blue, 80),
        textcolor = color.white,
        size     = size.small)
```

**Manage the budget.** Default is 50; declare what you need:

```pine
indicator("...", max_labels_count = 500)
```

For dashboards where one label is updated every bar, use `var label myLabel = na` and mutate:

```pine
var label statusLabel = na
if barstate.islast
    label.delete(statusLabel)
    statusLabel := label.new(bar_index, high, "Latest reading: " + str.tostring(rsi))
```

(Better: use a `table` for chart-anchored UI — see below.)

### Styles — when you want a shape-as-label instead of a text-with-pointer

Labels have two families of `style`:

- **Pointer-style** (`label.style_label_up`, `label.style_label_down`, `label.style_label_left`, `label.style_label_right`, `label.style_label_center`, `label.style_label_upper_left`, etc.) — a badge/balloon whose pointer tip sits at `(x, y)` and whose body extends in the named direction. Best for "tag the bar at this price with this text."
- **Shape-style** (`label.style_circle`, `label.style_square`, `label.style_diamond`, `label.style_triangleup`, `label.style_triangledown`, `label.style_xcross`, `label.style_cross`, `label.style_arrowup`, `label.style_arrowdown`, `label.style_flag`, `label.style_none`) — a pixel-sized shape centered at `(x, y)`. The `text` is rendered inside (or omitted with `text = ""`). Use this when you want a per-bar plotshape-equivalent but need a dynamic color or size.

### `yloc` — where to put the label vertically

| `yloc` | Y interpretation |
|--------|------------------|
| `yloc.price` (default) | `y` is the price coordinate |
| `yloc.abovebar` | `y` is ignored; placed just above the bar's high with a TV-managed pixel gap |
| `yloc.belowbar` | `y` is ignored; placed just below the bar's low with a TV-managed pixel gap |

The "circle marker just above/below the bar" idiom:

```pine
if event
    label.new(bar_index, na,
              text  = "",
              yloc  = isBelow ? yloc.belowbar : yloc.abovebar,
              style = label.style_circle,
              color = perBarColor,
              size  = sizeInput)        // size IS series-string here, unlike plotshape
```

This stays anchored to its bar through any zoom/pan/resize. The body is pixel-sized.

## `line.new()`, `box.new()` — trendlines and zones

Pattern: create once on the event, mutate as the event evolves.

```pine
var line srLine = na
if newPivotHigh
    if not na(srLine)
        line.delete(srLine)
    srLine := line.new(bar_index, ta.pivothigh(5, 5), bar_index + 50, ta.pivothigh(5, 5),
                       extend = extend.right, color = color.red, width = 2)
```

For S/R zones, prefer `box.new()` with a height range over two parallel lines.

`line.set_xy1()`, `line.set_xy2()`, `line.set_color()` etc. mutate without deleting. Use these for "extending" lines as price moves.

## `polyline.new()` (v6) — multi-point shapes

Replaces fragile multi-line patterns for channels, fans, polygon shapes.

```pine
points = array.new<chart.point>()
array.push(points, chart.point.from_index(bar_index - 50, high[50]))
array.push(points, chart.point.from_index(bar_index - 30, high[30]))
array.push(points, chart.point.from_index(bar_index - 10, high[10]))
array.push(points, chart.point.from_index(bar_index,      high))

myShape = polyline.new(points,
    curved      = false,
    closed      = false,
    line_color  = color.purple,
    line_width  = 2,
    fill_color  = color.new(color.purple, 90))
```

Closed polylines with fill_color are excellent for harmonic patterns, fan structures, channel shading.

Polyline budget: default 50, max 100. Declare with `max_polylines_count = 100`.

## `table.new()` — chart-anchored UI

Tables float at fixed viewport positions and don't scroll with bars. Ideal for dashboards, multi-symbol screeners, statistics panels.

```pine
var table dash = table.new(
    position = position.top_right,
    columns  = 3,
    rows     = 5,
    bgcolor  = color.new(color.black, 70),
    border_width = 1)

if barstate.islast
    table.cell(dash, 0, 0, "RSI",           text_color = color.white, bgcolor = color.gray)
    table.cell(dash, 1, 0, "MACD",          text_color = color.white, bgcolor = color.gray)
    table.cell(dash, 2, 0, "Volume %",      text_color = color.white, bgcolor = color.gray)
    table.cell(dash, 0, 1, str.tostring(rsi, "#.##"))
    table.cell(dash, 1, 1, str.tostring(macd, "#.##"))
    table.cell(dash, 2, 1, str.tostring(volPercentile, "#%"))
```

**Update only on `barstate.islast`.** Tables are expensive to update bar-by-bar and the user only sees the final state.

For multi-symbol screeners, loop over a list of symbols, call `request.security()` for each, and fill a column per metric.

## Colors

- `color.red`, `color.blue` etc. — named constants
- `#RRGGBB` or `#RRGGBBAA` — hex literals
- `color.new(base, transparency)` — transparency is 0 (opaque) to 100 (invisible). Note: this is the *opposite* of alpha.
- `color.rgb(r, g, b, transparency)` — explicit components
- `color.from_gradient(value, min, max, low_color, high_color)` — gradient mapping for heat-maps and intensity plots

Theme via input:

```pine
bullColor = input.color(color.green, "Bull color")
bearColor = input.color(color.red,   "Bear color")
```

## Text formatting (v6)

```pine
// In labels, tables, boxes:
label.new(bar_index, high, "Bold text",
    text_font_family = font.family_default,
    text_size_unit   = size.normal,
    text_formatting  = text.format_bold + text.format_italic)
```

Use sparingly — too much formatting clutters a chart.

## Performance and budget pitfalls

- **Creating a new drawing object every bar without deletion.** Old objects are auto-deleted when the limit fills, so your earliest signals vanish silently as new bars arrive.
- **Updating tables every bar.** Wrap in `if barstate.islast` to update once per chart.
- **Not declaring `max_*_count` parameters.** Default 50 is almost never enough for a useful long-history script.
- **Drawing objects in a loop without guards.** A loop that calls `line.new()` 100 times per bar exhausts the budget instantly.

## Layering and z-order

Drawing order: plots > bgcolor/barcolor > drawings (lines/labels/boxes) > tables (always on top).

For overlapping plots, later `plot()` calls draw on top of earlier ones. Use `display = display.none` on a plot used only as a `fill()` anchor so it doesn't appear visually.

## Idiomatic patterns

### Order block / supply-demand zone

```pine
if isBearishImbalance
    box.new(left   = bar_index - 1,
            top    = high[1],
            right  = bar_index + 50,
            bottom = low[1],
            border_color = color.red,
            bgcolor      = color.new(color.red, 85),
            extend       = extend.right)
```

### S/R level that extends until broken

```pine
var line activeSr = na
if newLevel
    if not na(activeSr)
        line.set_extend(activeSr, extend.none)        // freeze old line at break
    activeSr := line.new(bar_index, levelPrice, bar_index, levelPrice,
                         extend = extend.right, color = color.yellow)
if not na(activeSr) and close < line.get_y1(activeSr)
    line.set_color(activeSr, color.new(color.yellow, 70))   // grey out broken level
    activeSr := na
```

### Heat-map oscillator

```pine
rsi = ta.rsi(close, 14)
bgcolor(color.from_gradient(rsi, 30, 70, color.new(color.green, 70), color.new(color.red, 70)))
```

### Multi-symbol dashboard column

```pine
symbols = array.from("AAPL", "MSFT", "GOOG", "AMZN", "META")
var table dash = table.new(position.top_right, 3, array.size(symbols) + 1)

if barstate.islast
    table.cell(dash, 0, 0, "Symbol", text_color = color.white, bgcolor = color.gray)
    table.cell(dash, 1, 0, "RSI",    text_color = color.white, bgcolor = color.gray)
    table.cell(dash, 2, 0, "Vol %",  text_color = color.white, bgcolor = color.gray)
    for i = 0 to array.size(symbols) - 1
        sym = array.get(symbols, i)
        symRsi = request.security(sym, timeframe.period, ta.rsi(close, 14)[1], lookahead = barmerge.lookahead_on)
        table.cell(dash, 0, i + 1, sym)
        table.cell(dash, 1, i + 1, str.tostring(symRsi, "#.#"))
```
