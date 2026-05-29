# Pine Script v6 Language Reference

Pine Script v6 (released November 2024, monthly updates through current) is the current and only-receiving-updates version. v5 is frozen.

## Contents

- Type system — fundamental types, qualifiers, promotion
- Variable declarations — `=`, `:=`, `var`, `varip`, `const`
- Operators
- Control flow — `if`, `switch`, loops
- Functions — definitions, tuples, the no-nesting rule
- User-defined types (UDTs)
- Methods
- Collections — arrays, maps, matrices
- Built-in series — OHLC, bar_index, bid/ask, etc.
- v5 → v6 migration deltas
- Common compile-time errors decoded
- Imports and libraries

## Type system

Pine has two orthogonal axes of types: the **fundamental type** (what the value is) and the **qualifier** (how it can change).

### Fundamental types

| Type | Example | Notes |
|------|---------|-------|
| `int` | `42`, `-1` | 64-bit signed |
| `float` | `3.14`, `close` | All price/volume data is float |
| `bool` | `true`, `close > open` | v6 uses short-circuit evaluation in `and`/`or` |
| `color` | `color.red`, `#FF0000`, `color.new(color.blue, 50)` | RGBA |
| `string` | `"hello"` | UTF-8 |
| Drawing types | `line`, `label`, `box`, `table`, `polyline`, `linefill` | Drawing-object IDs |
| Collections | `array<T>`, `matrix<T>`, `map<K,V>` | Generic |
| UDTs | `MyType` | User-defined composite types |

### Qualifiers (the four "colours")

| Qualifier | Meaning | Changes per bar? |
|-----------|---------|------------------|
| `const` | Compile-time constant | No |
| `input` | User-input value | No (set at script load) |
| `simple` | Single value per script run | No |
| `series` | Time-series value | Yes (different per bar) |

A function parameter declared `simple float` cannot accept a `series float` — this is what generates the "cannot use a mutable variable" error when you try to pass a calculated value to `request.security()`'s `timeframe` parameter.

When you see that error: wrap the mutable expression in a function so Pine can evaluate it inside the series context, or use v6's new feature where `request.*()` accepts `series string` for symbol and timeframe.

### Type promotion

Pine auto-promotes along this chain: `const` → `input` → `simple` → `series`. So a `const` literal works anywhere `series` is expected, but not the reverse.

`int` auto-promotes to `float`. The reverse needs `int(myFloat)` explicit cast.

## Variable declarations

```pine
x = 5                  // type inferred (int), mutability inferred
int y = 5              // explicit type — preferred for clarity
var int counter = 0    // persists across bars, initialized once
varip int tickCount = 0  // persists across ticks (REPAINTS — realtime/historical diverge)
const int LENGTH = 14  // compile-time constant
```

Assignment is `=` for declaration, `:=` for reassignment. Mixing them is a common error.

```pine
x = 5     // declaration
x := 6    // reassignment
y = 6     // ERROR — redeclaration in same scope
```

## Operators

Arithmetic: `+ - * / %` (modulo on ints and floats)
Comparison: `< <= > >= == !=`
Logical: `and or not` — v6 uses short-circuit evaluation
Ternary: `cond ? a : b` — Pine's only conditional expression form
History reference: `series[n]` — value of `series` `n` bars ago
Compound: `+= -= *= /=`

## Control flow

### `if` as expression and statement

```pine
// Expression form (preferred when assigning):
ma = useEma ? ta.ema(close, len) : ta.sma(close, len)

// Statement form:
if condition
    doSomething()
    doAnotherThing()
else if otherCondition
    ...
else
    ...
```

Indentation must be consistent (4 spaces conventional). No braces.

### `switch`

```pine
maType = switch input.string("EMA", options = ["SMA", "EMA", "WMA"])
    "SMA" => ta.sma(close, len)
    "EMA" => ta.ema(close, len)
    "WMA" => ta.wma(close, len)
```

Tidier than nested ternaries when you have ≥3 cases.

### Loops

```pine
sum = 0.0
for i = 0 to 9
    sum += close[i]

while array.size(myArr) > 0
    array.pop(myArr)

for [k, v] in myMap   // v6 destructuring
    ...
```

Loops have a runtime cap — extremely long loops are killed. Vectorized built-ins (`ta.*`, `math.*`) are dramatically faster than hand loops.

## Functions

```pine
// Definitions live in global scope ONLY.
roundTo(float val, int digits) =>
    factor = math.pow(10, digits)
    math.round(val * factor) / factor

// Last expression is the return value. No `return` keyword.

// Multiple return values via tuple:
hilo(int len) =>
    [ta.highest(high, len), ta.lowest(low, len)]

[hh, ll] = hilo(20)
```

Function parameter types may be explicit (`float val`) or inferred. Explicit is preferred for clarity.

**Pine forbids nested function definitions.** If you need closure-like behavior, pass state through parameters or a UDT instance.

Functions are pure expressions of their inputs and Pine's built-in state (price series). They cannot mutate global variables — only return new values.

## User-defined types (UDTs)

```pine
type Pivot
    int   bar
    float price
    bool  isHigh

// Construction — keyword args required:
p = Pivot.new(bar = bar_index, price = high, isHigh = true)

// Field access:
p.price

// Mutation:
p.bar := bar_index - 1
```

UDTs solve the "I need three parallel arrays" problem cleanly — store one `array<MyType>` instead.

## Methods

```pine
// Define a method on a UDT:
method distance(Pivot self, Pivot other) =>
    math.abs(self.bar - other.bar)

// Use with dot notation:
d = p1.distance(p2)
```

Methods can be defined on built-in types too (`array<float>`, `line`, etc.) — useful for fluent APIs.

## Collections

### Arrays

```pine
var prices = array.new<float>()
array.push(prices, close)
array.size(prices)
array.get(prices, 0)         // first
array.last(prices)
array.slice(prices, 0, 10)   // first 10
array.sum / array.avg / array.stdev / array.min / array.max
```

Arrays are reference types — assigning one array variable to another shares the underlying data.

### Maps

```pine
var counts = map.new<string, int>()
map.put(counts, "trigger", map.contains(counts, "trigger") ? map.get(counts, "trigger") + 1 : 1)
```

Use for keyed lookup; iteration via `for [k, v] in myMap`.

### Matrices

2D arrays. Useful for correlation tables, multi-symbol grids. `matrix.new<float>(rows, cols)`, `matrix.set(m, row, col, val)`, etc.

## Built-in series

| Series | Meaning |
|--------|---------|
| `open high low close` | OHLC of current bar |
| `hl2 hlc3 ohlc4 hlcc4` | Common composites |
| `volume` | Bar volume |
| `time` | Bar open time, UNIX ms |
| `time_close` | Bar close time |
| `bar_index` | 0-based bar index from chart start |
| `last_bar_index` | Index of the last bar |
| `bid` / `ask` (v6) | Realtime quote prices — `na` on history |
| `dayofweek dayofmonth month year hour minute second` | Date components |

## v5 → v6 migration deltas

The biggest user-visible changes from v5 to v6:

1. **Bool short-circuit evaluation.** `a and slowFunc()` won't call `slowFunc()` if `a` is false. v5 always evaluated both.
2. **`request.*()` accepts `series string`.** Multi-symbol scripts that compute symbol/timeframe dynamically no longer need workarounds.
3. **`request.*()` allowed in local scope.** Previously had to be global.
4. **Scope count limit removed.** v5 capped total scopes at 550, including nested loops/ifs/UDT fields. v6 has no cap.
5. **Pine Logs.** `log.info/warning/error` for debugging.
6. **Bid/ask built-ins.** Realtime quotes — `na` on history.
7. **Polylines.** New drawing primitive for multi-point shapes.
8. **`plot()` `linestyle` parameter.** Dashed/dotted plots without separate drawing objects.
9. **Text formatting.** Bold/italic in labels, boxes, tables via format constants.
10. **Strategy 9000-trade cap removed.** Older trades are auto-trimmed when new ones are added.
11. **Active inputs.** Inputs can be enabled/disabled based on other inputs.

To migrate a v5 script, change `//@version=5` to `//@version=6`. Then test — most scripts work unchanged thanks to the deltas above being additive.

## Common compile-time errors decoded

| Error | What it actually means |
|-------|------------------------|
| `Cannot use a mutable variable as an argument...` | A `series` value was passed where `simple` is required. Wrap the calc in a function or — for `request.*` — rely on v6's loosened typing. |
| `Pine cannot determine the referencing length...` | You used `series[N]` where `N` exceeds `max_bars_back` (default 500). Set `max_bars_back = <N+50>` on the declaration. |
| `Loop is too long` | Loop runtime hit the engine cap. Move the work into a vectorized built-in or restrict iterations. |
| `Script could not be translated from: ...` | Almost always a missing comma or quote in `input.*()`. The parser points at the next-broken token, not the typo. |
| `The 'expression' argument of '...' is ill-formed` | A series-of-NA showed up where a typed value was needed. Initialize with `0.0`/`na` of the right type. |
| `Drawing limit exceeded` | Default 50 lines/labels/boxes. Declare `max_lines_count = 500` etc. on the declaration. |

## Imports and libraries

```pine
//@version=6
import TradingView/Strategy/5 as TVStrat
// Use: TVStrat.atr(14)
```

Library scripts use `library("MyLib")` and export functions with `export myFunc(...) => ...`. Importing a library makes its exported functions available namespaced under the alias.
