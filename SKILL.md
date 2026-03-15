---
name: pinescript
description: "Pine Script v6 development for TradingView -- enforces hard language constraints, prevents common LLM errors, covers the full type system, execution model, data structures, drawing objects, strategy patterns, webhook automation, and performance optimization."
metadata:
  version: 2.0.0
  domains: [pinescript, tradingview, trading, indicators, strategies, pine-script-v6]
---

# /pinescript -- Pine Script v6 Development

Write, debug, and refactor Pine Script v6 code for TradingView indicators and strategies.
This skill enforces hard language constraints that cause compile errors if violated. These
are NOT suggestions -- they are grammar rules of the Pine Script language.

Targets Pine Script v6 by default. See "V5 COMPATIBILITY" section when working with
`//@version=5` scripts.

---

## Triggers

- `/pinescript` -- activate Pine Script v6 mode
- "write a pine script", "pine v6", "tradingview indicator", "tradingview strategy"
- "pinescript error", "pine script compile error"
- Any request involving `.pine` files or TradingView code

---

## HARD CONSTRAINTS (compile errors or silent bugs if violated)

### 1. Functions MUST be top-level -- NEVER nested

Pine Script functions using the `name(params) =>` syntax CANNOT be defined inside `if`,
`for`, `switch`, `while`, or any indented block. They must ALWAYS be at root indentation.

WRONG -- will not compile:
```pinescript
if condition
    myFunc(x) =>    // COMPILE ERROR
        x * 2
```

RIGHT:
```pinescript
myFunc(x) =>
    x * 2

if condition
    result = myFunc(someValue)
```

### 2. request.security tuples for multiple values

Fetch multiple values from a single request.security call using tuple destructuring.
Maximum ~9 values per tuple. Use UDTs to exceed the 127 total tuple element limit.

```pinescript
[val1, val2, val3] = request.security(syminfo.tickerid, "60",
     [close, ta.ema(close, 50), ta.atr(14)],
     barmerge.gaps_off, barmerge.lookahead_off)
```

### 3. var state functions need SEPARATE request.security calls

Functions that use `var` for persistent state MUST have their own independent
request.security call. They CANNOT be bundled into tuples.

WRONG -- var state silently broken:
```pinescript
[ema, swingLow] = request.security(sym, "15",
     [ta.ema(close, 50), f_lastSwingLow(3)])   // f_lastSwingLow uses var -- BROKEN
```

RIGHT -- separate calls:
```pinescript
[p15Close, p15EMA] = request.security(sym, "15",
     [close, ta.ema(close, 50)],
     barmerge.gaps_off, barmerge.lookahead_off)

p15SwLo = request.security(sym, "15",
     f_lastSwingLow(3), barmerge.gaps_off, barmerge.lookahead_off)
```

### 4. No local scoping for functions

All functions exist globally. You cannot define two functions with the same name,
and you cannot define functions conditionally. Functions cannot be recursive.

### 5. No object-oriented features

No classes, methods (in the OOP sense), or constructors. Use UDTs with `type` keyword
for structured data. Use `method` keyword for dot-notation syntax on types, but there
is no inheritance or polymorphism.

### 6. Lazy and/or evaluation breaks var-state accumulators (v6)

In v6, `and`/`or` operators evaluate lazily. If the left operand determines the result,
the right operand is SKIPPED entirely. Functions with `var` state in the right operand
will not execute on every bar, corrupting their accumulated state.

WRONG -- accumulator in right operand gets skipped:
```pinescript
if conditionA and f_accumulator(close)   // f_accumulator uses var -- BROKEN in v6
    doSomething()
```

RIGHT -- extract to global scope:
```pinescript
accResult = f_accumulator(close)   // always executes
if conditionA and accResult
    doSomething()
```

This also applies to `ta.*` functions -- they must be called on every bar for
consistent results. Never put `ta.*` calls inside conditional branches.

---

## LLM ERROR PREVENTION

### v6 Namespace -- What NOT to generate

NEVER generate pre-v5 function names. This is the #1 AI-generated Pine Script error.

| WRONG (pre-v5) | CORRECT (v5/v6) |
|-----------------|-----------------|
| `study()` | `indicator()` |
| `security()` | `request.security()` |
| `sma()` | `ta.sma()` |
| `ema()` | `ta.ema()` |
| `rsi()` | `ta.rsi()` |
| `macd()` | `ta.macd()` |
| `crossover()` | `ta.crossover()` |
| `crossunder()` | `ta.crossunder()` |
| `highest()` | `ta.highest()` |
| `lowest()` | `ta.lowest()` |
| `stoch()` | `ta.stoch()` |
| `red` | `color.red` |
| `green` | `color.green` |
| `black` | `color.black` |
| `input()` | `input.int()`, `input.float()`, etc. |

### Functions that CANNOT be called in local scope

These MUST be at global scope -- not inside `if`, `for`, `while`, `switch`, or functions:

- `plot()`, `plotshape()`, `plotchar()`, `plotarrow()`, `plotcandle()`, `plotbar()`
- `hline()`, `fill()`
- `bgcolor()`, `barcolor()`
- `alertcondition()`
- `indicator()`, `strategy()`, `library()`

```pinescript
// WRONG
if condition
    plot(close, "MyPlot")   // COMPILE ERROR

// CORRECT -- conditional value, global plot
plotValue = condition ? close : na
plot(plotValue, "MyPlot")
```

### TA functions must be called unconditionally

All `ta.*` functions must execute on every bar to maintain consistent historical data.
Pre-evaluate before conditional blocks.

```pinescript
// WRONG -- ta.ema only runs on some bars
if someCondition
    val = ta.ema(close, 14)   // inconsistent history!

// CORRECT
emaVal = ta.ema(close, 14)
if someCondition
    // use emaVal
```

### Do not hallucinate functions

If you are unsure whether a function exists, do not invent one. Pine Script's namespace
is closed -- only built-in functions, imported library functions, and user-defined
functions are available. There is no `try/catch`, no `async`, no `import` from URLs.

---

## EXECUTION MODEL

Understanding this prevents entire categories of bugs.

**Bar-by-bar execution**: Pine Script runs your entire script once for every historical bar,
left to right from the oldest available bar to the most recent. It is NOT event-driven.

**Realtime behavior**: On the current (rightmost) bar, the script re-executes on every
tick (price update). Before each re-execution, all variable values are ROLLED BACK to
their state at the bar's open -- except `var` and `varip` variables.

**Variable persistence**:
- No keyword: recalculated from scratch every bar (series behavior)
- `var`: initialized once (bar zero), persists across bars, but rolls back on realtime ticks
- `varip`: persists across bars AND across realtime ticks (escapes rollback)

**Why `barstate.isconfirmed` exists**: Because on a realtime bar, your script sees
unconfirmed data that may change. `barstate.isconfirmed` is `true` only when the bar
has closed. Use it for signals that should not repaint.

**Strategy execution**: With `calc_on_every_tick=false` (recommended), strategies
execute once per bar close, not on every tick. Orders created on bar close execute on
the next bar's open (when `process_orders_on_close=false`).

---

## TYPE SYSTEM

### Qualifier hierarchy

`const < input < simple < series`

A function parameter accepting `simple int` also accepts `input int` or `const int`,
but NOT `series int`. You cannot downgrade a qualifier.

```pinescript
// WRONG -- creates series int, fails where simple int required
length = close > open ? 10 : 20
sma_val = ta.sma(close, length)   // ERROR: cannot use series int

// CORRECT -- use input (compile-time constant)
length = input.int(14, "Length")
sma_val = ta.sma(close, length)
```

Key rules:
- `input.*` returns "input" qualified types (constant after compilation)
- `input.source()` returns "series" (exception -- it references a price series)
- `ta.*` functions always return "series"
- `request.security()` returns "series"
- Array sizes require "simple" values
- Most `ta.*` length parameters require "simple int"

### Fundamental types

`int`, `float`, `bool`, `string`, `color`

### Special types

`label`, `line`, `box`, `table`, `polyline`, `chart.point`, `array`, `matrix`, `map`

### na handling

```pinescript
na(value)                    // check -- NEVER use value == na
nz(value, replacement)      // replace na with default
fixnan(value)                // forward-fill na with last known value
na == na                     // evaluates to na, NOT true!
```

v6 change: Booleans can no longer be `na`. The functions `na()`, `nz()`, `fixnan()`
no longer accept `bool` arguments.

### v6 integer division

In v6, `5 / 2 = 2.5` (returns float). Use `int(5 / 2)` to get `2`.
In v5, `5 / 2 = 2` (truncated).

---

## LANGUAGE SYNTAX

### Version and declaration

```pinescript
//@version=6
indicator("Name", shorttitle="SHORT", overlay=true, max_bars_back=5000)
```
or:
```pinescript
//@version=6
strategy("Name", shorttitle="SHORT", overlay=true,
     default_qty_type=strategy.fixed, default_qty_value=1,
     initial_capital=10000,
     commission_type=strategy.commission.cash_per_contract, commission_value=0.62,
     slippage=1,
     process_orders_on_close=false, calc_on_every_tick=false,
     max_labels_count=500)
```

### Input catalog

```pinescript
i_len    = input.int(14, "Length", minval=1, maxval=500, group="Settings", tooltip="EMA length")
i_mult   = input.float(2.0, "Multiplier", step=0.1, group="Settings")
i_show   = input.bool(true, "Show signals", group="Display")
i_mode   = input.string("Fast", "Mode", options=["Fast", "Slow"], group="Settings")
i_clr    = input.color(color.blue, "Color", group="Display")
i_sess   = input.session("0930-1600", "Session", group="Session")
i_src    = input.source(close, "Source", group="Settings")   // returns series!
i_tf     = input.timeframe("60", "Timeframe", group="MTF")
i_sym    = input.symbol("AAPL", "Symbol", group="MTF")
i_notes  = input.text_area("", "Notes")
```

v6 addition -- enums:
```pinescript
enum Direction
    Long = "Long only"
    Short = "Short only"
    Both = "Both directions"

i_dir = input.enum(Direction.Both, "Direction")
```

Use `group` for logical sections, `tooltip` for user guidance, `inline` for
side-by-side inputs on the same line.

### Loops

```pinescript
// Counted loop
for i = 0 to 9
    // body

// Counted with step
for i = 0 to 100 by 5
    // body

// Array iteration
for element in myArray
    // body

// Array iteration with index (tuple destructuring)
for [idx, element] in myArray
    // body

// While loop
while condition
    // body
    if exitCondition
        break
    continue   // skip to next iteration
```

Performance: built-in `ta.highest()` is ~100x faster than a manual `for` loop.
Always prefer built-in functions over loops.

### Switch statement

Pine Script DOES have switch, and it CAN return values (it functions as an expression):

```pinescript
// Form 1: with key expression
result = switch i_mode
    "Fast" => ta.ema(close, 8)
    "Slow" => ta.ema(close, 21)
    => ta.ema(close, 14)   // default

// Form 2: boolean expressions (like chained if/else)
signal = switch
    longCondition  => 1
    shortCondition => -1
    => 0   // default
```

Both forms return the value of the executed branch. The `=>` default case is optional
but recommended.

### String operations

```pinescript
// Concatenation (v5 style)
msg = "Price: " + str.tostring(close, "#.##")

// str.format (v6 -- preferred)
msg = str.format("Price: {0}, Volume: {1}", close, volume)

// Common functions
str.contains(s, "pattern")
str.replace(s, "old", "new", 0)   // 0 = first occurrence
str.split(s, ",")                  // returns array<string>
str.upper(s)
str.lower(s)
str.startswith(s, "prefix")
str.length(s)
str.substring(s, start, end)
```

---

## DATA STRUCTURES

### Arrays

```pinescript
var a = array.new<float>(10, 0.0)
a.set(0, close)           // v6: also a[0] := close
a.get(0)                  // v6: also a[0]
a.push(value)
a.pop()
a.shift()                 // remove first
a.unshift(value)           // add to front
a.size()
a.sort()
a.slice(from, to)
a.includes(value)
a.indexof(value)
a.clear()
```

v6: Direct indexing `a[i]` replaces `a.get(i)` / `a.set(i, val)`.
v6: Negative indexing `a[-1]` gets last element.

### Maps

Key-value storage with O(1) lookup. Keys must be fundamental types.

```pinescript
var m = map.new<string, float>()
m.put("RSI", ta.rsi(close, 14))
val = m.get("RSI")
m.contains("RSI")          // check existence
m.remove("RSI")
m.size()
m.keys()                   // returns array<string>
m.values()                 // returns array<float>

// Iteration (maintains insertion order)
for [key, value] in m
    // body
```

Max 50,000 key-value pairs.

### Matrices

```pinescript
var mat = matrix.new<float>(3, 3, 0.0)
matrix.set(mat, row, col, value)
matrix.get(mat, row, col)
matrix.rows(mat)
matrix.columns(mat)

// Linear algebra (float/int only)
matrix.det(mat)            // determinant
matrix.inv(mat)            // inverse
matrix.transpose(mat)
matrix.eigenvalues(mat)
matrix.eigenvectors(mat)
matrix.mult(mat1, mat2)    // matrix multiplication
matrix.rank(mat)
matrix.trace(mat)
```

Max 100,000 total elements.

### User-Defined Types (UDTs)

```pinescript
type Position
    float entry = na
    float stop = na
    float target = na
    int   direction = 0

// Instantiation
pos = Position.new(entry=close, stop=close - atr, target=close + 2*atr, direction=1)

// Field access
pos.entry
pos.stop := newStop   // reassignment
```

### Methods

Custom dot-notation functions on any type:

```pinescript
method riskReward(Position p) =>
    math.abs(p.target - p.entry) / math.abs(p.entry - p.stop)

// Usage
rr = pos.riskReward()

// Method chaining
method update(array<float> a, float val) =>
    a.push(val)
    if a.size() > 20
        a.shift()
    a

myArr.update(close).update(open)   // chained
```

Methods use dot notation on the first parameter's type. Built-in types (array, map,
matrix, line, box, label, table, polyline) all support method syntax.

### Enums (v6)

```pinescript
enum TradeState
    Flat
    Long
    Short

var TradeState state = TradeState.Flat

switch state
    TradeState.Long  => strategy.close("Long")
    TradeState.Short => strategy.close("Short")
```

Enums are type-safe alternatives to string or int constants. Use with `input.enum()`
for dropdowns.

---

## ANTI-REPAINTING RULES

### request.security

ALWAYS use both gap and lookahead parameters:
```pinescript
request.security(sym, tf, expr, barmerge.gaps_off, barmerge.lookahead_off)
```

- `barmerge.gaps_off` -- fills gaps with last known value
- `barmerge.lookahead_off` -- no future data (critical for backtesting accuracy)
- NEVER use `barmerge.lookahead_on` unless you need forward-looking reference levels
  (never for entry signals)

Alternative non-repainting pattern with lookahead:
```pinescript
// Use [1] offset with lookahead_on -- gets previous confirmed bar
htf_close = request.security(sym, "D", close[1], barmerge.gaps_off, barmerge.lookahead_on)
```

### Confirmed bars only

```pinescript
canEnter = longCondition and barstate.isconfirmed
```

### Strategy settings for realistic backtesting

```pinescript
strategy("Name",
     process_orders_on_close=false,    // orders execute on next bar open
     calc_on_every_tick=false,         // calculate once per bar close
     commission_type=strategy.commission.cash_per_contract,
     commission_value=0.62,
     slippage=1)
```

---

## DRAWING OBJECTS

### Lines

```pinescript
myLine = line.new(x1=bar_index[10], y1=low[10], x2=bar_index, y2=high,
     color=color.white, width=2, style=line.style_solid)
line.set_extend(myLine, extend.right)
line.delete(myLine)
```

Max 500 lines (`max_lines_count`). FIFO garbage collection when exceeded.

### Boxes

```pinescript
myBox = box.new(left=bar_index[10], top=high[10], right=bar_index, bottom=low[10],
     border_color=color.white, bgcolor=color.new(color.blue, 80))
box.delete(myBox)
```

Max 500 boxes (`max_boxes_count`).

### Labels

```pinescript
myLabel = label.new(x=bar_index, y=high, text="Signal",
     color=color.green, textcolor=color.white,
     style=label.style_label_down, size=size.small)
label.delete(myLabel)
```

Max 500 labels (`max_labels_count`). Styles: `label.style_label_up/down/left/right`,
`label.style_circle`, `label.style_cross`, `label.style_diamond`, etc.

### Polylines

Connect up to 10,000 `chart.point` instances:

```pinescript
var points = array.new<chart.point>()
if barstate.islast
    points.clear()
    points.push(chart.point.from_index(bar_index - 30, high[30]))
    points.push(chart.point.from_index(bar_index - 15, low[15]))
    points.push(chart.point.from_index(bar_index, high))
    polyline.new(points, curved=true, closed=true,
         line_color=color.blue, fill_color=color.new(color.blue, 80))
```

Max 100 polylines (`max_polylines_count`).

### Tables

```pinescript
var table tb = table.new(position.top_right, 3, 4,
     bgcolor=color.new(color.black, 80), border_width=1)
if barstate.islast
    table.merge_cells(tb, 0, 0, 2, 0)   // merge header row
    table.cell(tb, 0, 0, "Dashboard", text_color=color.white, text_size=size.normal)
    table.cell(tb, 0, 1, "RSI", text_color=color.white, text_size=size.small)
    table.cell(tb, 1, 1, str.tostring(ta.rsi(close, 14), "#.##"),
         text_color=color.white, text_size=size.small)
```

Tables float independently of chart bars. Always populate inside `if barstate.islast`.

### Linefill

```pinescript
p1 = plot(ema1, color=color.blue)
p2 = plot(ema2, color=color.red)
fill(p1, p2, color=color.new(color.blue, 80))   // fill between plots

// Or between lines:
linefill.new(line1, line2, color=color.new(color.green, 80))
```

### plotshape / plotchar / plotarrow

```pinescript
plotshape(buySignal, style=shape.triangleup, location=location.belowbar,
     color=color.green, size=size.small, text="BUY")
plotarrow(momentum, colorup=color.green, colordown=color.red)
```

### Drawing on barstate.islast

Create labels, tables, and non-persistent drawings inside `if barstate.islast` to
avoid creating objects on every historical bar (performance + limit management).

---

## REQUEST FUNCTIONS

### request.security (basic)

```pinescript
[htfClose, htfEMA] = request.security(syminfo.tickerid, "D",
     [close, ta.ema(close, 50)],
     barmerge.gaps_off, barmerge.lookahead_off)
```

v6: Symbol and timeframe accept `series string` by default (dynamic requests).
v5: These must be `simple string` unless explicitly opted in.

### request.security_lower_tf

Returns an array of intrabar values:
```pinescript
ltf_closes = request.security_lower_tf(syminfo.tickerid, "1", close)
// Returns array of 1-minute closes within the current chart bar
avg_ltf = ltf_closes.avg()
```

Max 200,000 intrabars retrievable.

### Other request functions

```pinescript
request.financial(syminfo.tickerid, "EARNINGS_PER_SHARE", "FQ")  // fundamentals
request.economic("US", "GDP")                                     // economic data
request.dividends(syminfo.tickerid, dividends.gross, barmerge.gaps_off)
request.splits(syminfo.tickerid, splits.denominator, barmerge.gaps_off)
request.earnings(syminfo.tickerid, earnings.actual, barmerge.gaps_off)
request.currency_rate("EURUSD")                                   // FX conversion
request.seed("github_user", "repo_name", "field_name")            // external data
```

### ticker.new and ticker.modify

```pinescript
customTicker = ticker.new("NYSE", "AAPL")
extSession = ticker.modify(syminfo.tickerid, session=session.extended)
extClose = request.security(extSession, timeframe.period, close)
```

### timeframe.change

Detect new timeframe periods without `request.security()`:
```pinescript
isNewDay = timeframe.change("1D")
bgcolor(isNewDay ? color.new(color.blue, 80) : na)
```

---

## STRATEGY PATTERNS

### Entry with stop/target

```pinescript
strategy.entry("Long", strategy.long, qty=qty)
strategy.exit("Long Exit", "Long", stop=stopLevel, limit=tpLevel)
```

v6: The `when` parameter is removed. Use `if` blocks instead.

### Pyramiding

```pinescript
strategy("Pyramid", pyramiding=5)
// strategy.entry() respects pyramiding limits
// strategy.order() ignores pyramiding -- use with caution
```

### ATR-based position sizing

```pinescript
atr_val = ta.atr(20)
stop_dist = atr_val * atr_mult
capital_to_risk = strategy.equity * (risk_pct / 100.0)
risk_per_unit = stop_dist * syminfo.pointvalue
qty = math.floor(capital_to_risk / risk_per_unit)
```

### Risk management

```pinescript
strategy.risk.max_drawdown(10, strategy.percent_of_equity)   // halt at 10% DD
strategy.risk.max_intraday_loss(500, strategy.cash)            // daily loss limit
strategy.risk.max_intraday_filled_orders(10)                   // daily trade cap
strategy.risk.max_position_size(5)                             // max contracts
strategy.risk.allow_entry_in(strategy.direction.long)          // long-only
```

### Trade analysis API

```pinescript
// Closed trades (indexed from 0)
strategy.closedtrades.entry_price(strategy.closedtrades - 1)   // last trade entry
strategy.closedtrades.profit(strategy.closedtrades - 1)         // last trade P&L
strategy.closedtrades.max_runup(strategy.closedtrades - 1)
strategy.closedtrades.max_drawdown(strategy.closedtrades - 1)

// Open trades
strategy.opentrades.entry_price(0)
strategy.opentrades.size(0)
```

### Session flattening

```pinescript
if sessEnd and strategy.position_size != 0
    strategy.close_all(comment="FLAT EOD")
```

### Position sync (state machine reset)

```pinescript
if strategy.position_size == 0 and tradeDir != 0
    tradeDir    := 0
    entryPrice  := na
    activeStop  := na
```

### Stop upgrade / trailing

```pinescript
if tradeDir == 1 and not upgradeDone and not na(entryPrice)
    if close[1] > entryPrice and close > entryPrice
        activeStop  := entryPrice - 0.5 * rValue
        upgradeDone := true
        strategy.exit("Long Exit", "Long", stop=activeStop, limit=tpLevel)
```

---

## INDICATOR PATTERNS

### Overlay with table

```pinescript
indicator("Name", overlay=true, max_bars_back=5000)
var table tb = table.new(position.top_left, cols, rows, ...)
if barstate.islast
    // populate table
```

### Non-overlay with plot pane

```pinescript
indicator("Name", overlay=false)
plot(value, "Label", color=color.white)
hline(0, "Zero", color=color.gray)
```

### Bar coloring

```pinescript
barcolor(longAligned ? color.new(color.green, 60) :
     shortAligned ? color.new(color.red, 60) : na)
```

### Color gradient

```pinescript
clr = color.from_gradient(ta.rsi(close, 14), 30, 70, color.red, color.green)
```

### Session detection

```pinescript
i_sess = input.session("0930-1655", "Session (ET)")
inSess = not na(time(timeframe.period, i_sess, "America/New_York"))
sessStart = inSess and not inSess[1]
sessEnd   = inSess[1] and not inSess
```

---

## ALERTS AND WEBHOOKS

### alertcondition vs alert

- `alertcondition()`: indicator-only, static messages (const string), creates a
  selectable condition in the alert dialog. Must be at global scope.
- `alert()`: works in both indicators and strategies, supports dynamic messages
  (series string). Can be inside `if` blocks.

```pinescript
// Indicator -- alertcondition (static message)
alertcondition(ta.crossover(fast, slow), "Golden Cross", "EMA crossover detected")

// Strategy or indicator -- alert (dynamic message)
if ta.crossover(fast, slow)
    alert(str.format("BUY {0} at {1}", syminfo.ticker, close), alert.freq_once_per_bar)
```

### Webhook JSON construction

```pinescript
if longCondition
    strategy.entry("Long", strategy.long)
    alert('{"action":"buy","symbol":"' + syminfo.ticker +
          '","price":' + str.tostring(close) +
          ',"qty":' + str.tostring(qty) + '}',
          alert.freq_once_per_bar)
```

### Webhook placeholder variables

Available in TradingView's alert message field (not in Pine code):
- `{{exchange}}`, `{{ticker}}`, `{{close}}`, `{{open}}`, `{{high}}`, `{{low}}`
- `{{time}}`, `{{volume}}`, `{{timenow}}`
- `{{strategy.order.action}}`, `{{strategy.order.contracts}}`
- `{{strategy.order.price}}`, `{{strategy.order.id}}`
- `{{strategy.market_position}}`, `{{strategy.position_size}}`
- `{{plot_0}}`, `{{plot("PlotName")}}`

### Alert frequency constants

- `alert.freq_once_per_bar` -- once per bar
- `alert.freq_once_per_bar_close` -- once on bar close
- `alert.freq_all` -- every call

Rate limit: 15 alerts per 3 minutes triggers automatic halt.

---

## LIBRARIES

### Creating a library

```pinescript
//@version=6
// @description Utility functions for position sizing
library("RiskUtils", overlay=true)

// @function Calculate position size from risk percentage
// @param riskPct Risk as percentage of equity
// @param stopDist Distance to stop in price units
// @returns Number of contracts
export calcQty(float riskPct, float stopDist) =>
    capital = strategy.equity * (riskPct / 100.0)
    riskPerUnit = stopDist * syminfo.pointvalue
    math.floor(capital / riskPerUnit)
```

v6: Library constants can be exported: `export const float RATIO = 1.618`

### Importing

```pinescript
import username/RiskUtils/1 as risk
qty = risk.calcQty(1.0, atr * 2)
```

Version is pinned by major number. Use `as alias` for clarity.

### Library limitations

- Exported functions cannot use global variables (unless `const`)
- In v5, exported functions cannot contain `request.*()` in local scope (relaxed in v6)
- Max 80,000 compiled tokens per script; 1,000,000 across all imports
- Must export at least one definition

---

## PERFORMANCE OPTIMIZATION

### Use built-ins over loops

Built-in `ta.highest(close, 20)` is ~20x faster than a manual `for` loop.
Manual loops at 200-bar lookback can be ~100x slower.

### Consolidate request.security calls

```pinescript
// WRONG -- 4 separate calls
o = request.security(sym, tf, open)
h = request.security(sym, tf, high)
l = request.security(sym, tf, low)
c = request.security(sym, tf, close)

// CORRECT -- 1 call with tuple
[o, h, l, c] = request.security(sym, tf, [open, high, low, close])
```

### Limit string operations

String manipulation is expensive. Use `var` for strings and limit operations
to `barstate.islast` when possible.

### Draw only on last bar

Create labels, tables, and drawings inside `if barstate.islast` to avoid creating
objects on every historical bar.

### Use the Pine Profiler

TradingView has a built-in profiler (Ctrl+Shift+P in the editor) that shows
execution time per line. Use it to identify bottlenecks.

---

## HARD LIMITS

| Resource | Limit |
|----------|-------|
| Plots | 64 |
| `request.*()` calls | 40 (64 on Ultimate plan) |
| Tuple elements across all requests | 127 (use UDTs to exceed) |
| Lines / boxes / labels | 500 each |
| Polylines | 100 |
| Polyline points | 10,000 |
| Array / matrix elements | 100,000 |
| Map key-value pairs | 50,000 |
| String length | 40,960 characters |
| Variables per scope | 1,000 |
| Total scopes | 550 |
| Max bars back (series) | 5,000 (OHLCV: 10,000) |
| Max bars forward | 500 |
| Loop computation per bar | ~200ms |
| Script execution time | 20s (basic) / 40s (premium) |
| Compilation time | 2 minutes |
| Compiled tokens | 100,000 per script |
| Imported library tokens | 1,000,000 total |
| Intrabars (security_lower_tf) | 200,000 |
| Alert rate limit | 15 per 3 minutes |

---

## V5 COMPATIBILITY

When working with `//@version=5` scripts, note these differences from v6:

| Feature | v5 | v6 |
|---------|----|----|
| Integer division | `5/2 = 2` (truncated) | `5/2 = 2.5` (float) |
| Boolean na | Booleans can be `na` | Booleans CANNOT be `na` |
| `and`/`or` evaluation | Eager (both sides always run) | Lazy (short-circuit) |
| Array indexing | `a.get(i)` / `a.set(i, v)` | Also `a[i]` and `a[-1]` |
| `request.*()` scope | Global scope only | Can be called from local scope |
| `request.*()` args | `simple string` only | `series string` allowed |
| `when` parameter | Available (deprecated) | Removed |
| String formatting | Concatenation only | `str.format()` available |
| Enums | Not available | Available |
| Negative array index | Not available | `a[-1]` for last element |
| `na()`/`nz()` on bool | Allowed | Compile error |
| Library constant export | Not available | `export const` available |

---

## COMMON COMPILATION ERRORS

| Error | Cause | Fix |
|-------|-------|-----|
| `Undeclared identifier 'X'` | Pre-v5 name or out-of-scope variable | Add namespace (`ta.`, `color.`, `request.`) |
| `Cannot call 'plot' in local scope` | `plot`/`fill`/`bgcolor` inside `if`/`for`/function | Move to global scope, use conditional values |
| `Cannot call 'X' with argument 'Y'=series` | Series value where simple required | Use `input.*` or pre-compute |
| `Mismatched input expecting 'end of line'` | Wrong indentation or line wrapping | Fix to 4 spaces / 1 tab, no mixed |
| `Loop is too long (> 200 ms)` | Loop exceeds computation limit | Use built-in functions instead |
| `Script has too many local variables` | >1,000 variables in one scope | Consolidate, use arrays/UDTs |
| `Script requesting too many securities` | >40 `request.*()` calls | Combine with tuples, reuse identical calls |
| `Cannot determine referencing length` | Auto-detection of history depth failed | Use `max_bars_back(myVar, N)` |
| `No viable alternative at character` | Invalid character (smart quotes, special chars) | Replace with standard ASCII |
| `The function should be called on each calculation` | `ta.*` inside conditional block | Pre-evaluate before `if` block |
| `Script could not be translated from null` | Internal compiler error, often from complex code | Simplify, split into library |

---

## ERROR HANDLING

### runtime.error for custom validation

```pinescript
if input_length < 1
    runtime.error("Length must be >= 1, got: " + str.tostring(input_length))

if timeframe.period == "1"
    runtime.error("This indicator does not support 1-minute timeframe")
```

### Input validation

```pinescript
i_len = input.int(14, "Length", minval=1, maxval=500)   // built-in bounds
```

### Defensive patterns

```pinescript
result = denom != 0 ? num / denom : 0.0                // safe division
val = idx >= 0 and idx < arr.size() ? arr[idx] : na     // safe array access
safeClose = nz(close[1], open)                           // na fallback
```

### max_bars_back

```pinescript
indicator("My Ind", max_bars_back=500)    // script-wide
max_bars_back(myVar, 500)                  // per-variable
```

Use when Pine cannot auto-detect how much history a variable needs.

---

## RECOMMENDED SCRIPT STRUCTURE

1. `//@version=6` and `indicator()` / `strategy()` / `library()` declaration
2. Constants (`SCREAMING_SNAKE_CASE`)
3. Input groups with descriptive tooltips
4. Helper functions (top-level, before any block logic)
5. Session detection (if applicable)
6. Core calculations / indicators
7. Signal/condition logic
8. State machine (`var`-based trade tracking, if applicable)
9. Entry/exit logic (strategies)
10. Plots and visual elements
11. Table (if any) -- populated inside `if barstate.islast`
12. Alert conditions (at bottom)

---

## DEBUGGING CHECKLIST

When a Pine Script won't compile or behaves unexpectedly:

1. Is `//@version=6` (or 5) the first line?
2. Are ALL function names using v5/v6 namespaces (`ta.`, `request.`, `color.`)?
3. Are ALL functions defined at top-level (not nested inside any block)?
4. Are `plot()`, `fill()`, `bgcolor()`, `alertcondition()` at global scope?
5. Are ALL `ta.*` functions called unconditionally (not inside `if`)?
6. Are `request.security` calls using `barmerge.gaps_off, barmerge.lookahead_off`?
7. Are functions with `var` state in SEPARATE `request.security` calls?
8. Is `na()` used for null checks (not `== na`)?
9. Are `nz()` calls wrapping potentially-na values before arithmetic?
10. Are `var`-state functions in right operands of `and`/`or` extracted to global scope (v6)?
11. Is the script under the execution time limit (simplify if slow)?
12. Are array indices within bounds (0 to size-1)?
13. Are string values built with `str.tostring()` or `str.format()`?
14. Is `var` used for state that should persist, and omitted for per-bar recalculation?
15. Does the script handle early-bar `na` propagation (warmup period)?
