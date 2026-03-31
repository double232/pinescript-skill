# pinescript-skill

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill for writing Pine Script v6 code that compiles on the first try.

LLMs are notoriously bad at Pine Script. They hallucinate deprecated function names, nest functions inside `if` blocks, put `plot()` in local scope, and bundle `var`-state functions into `request.security` tuples. This skill prevents all of that by injecting the language's hard constraints, correct namespaces, and anti-repainting rules directly into the assistant's context.

---

## What it covers

| Section | What you get |
|---|---|
| **Hard constraints** | 6 compile-or-die rules -- nested functions, tuple limits, `var`-state isolation, lazy `and`/`or` traps |
| **LLM error prevention** | Namespace mapping table (pre-v5 -> v6), global-scope-only functions, `ta.*` unconditional calling |
| **Execution model** | Bar-by-bar execution, `var` vs `varip` rollback behavior, realtime tick semantics |
| **Type system** | Qualifier hierarchy (`const < input < simple < series`), `na` handling, v6 integer division |
| **Language syntax** | Inputs, loops, `switch` expressions, string operations, enums (v6) |
| **Data structures** | Arrays, maps, matrices, UDTs, methods, method chaining |
| **Anti-repainting** | `request.security` parameter rules, `barstate.isconfirmed`, realistic backtest settings |
| **Drawing objects** | Lines, boxes, labels, polylines, tables, linefill, `plotshape`/`plotarrow` |
| **Request functions** | `request.security`, `request.security_lower_tf`, financials, economic data, `ticker.modify` |
| **Strategy patterns** | Entry/exit, pyramiding, ATR position sizing, risk management, session flattening, trailing stops |
| **Indicator patterns** | Overlay + table, bar coloring, color gradients, session detection |
| **Alerts and webhooks** | `alertcondition` vs `alert()`, webhook JSON, placeholder variables, rate limits |
| **Libraries** | Creating, exporting, importing, limitations |
| **Performance** | Built-ins vs loops, consolidating `request.security` calls, draw-on-last-bar |
| **Hard limits** | Every resource ceiling (plots, requests, array sizes, execution time, etc.) |
| **v5 compatibility** | Side-by-side diff table for all breaking changes between v5 and v6 |
| **Debugging checklist** | 15-point checklist for when a script won't compile |

---

## Installation

### Option 1: Claude Code CLI

```bash
claude install-skill https://github.com/double232/pinescript-skill
```

### Option 2: Manual

Copy `SKILL.md` into your Claude Code skills directory:

- **Per-project**: `.claude/skills/pinescript.md` in your repo root
- **Global**: `~/.claude/skills/pinescript.md`

---

## Usage

The skill activates automatically when Claude Code detects Pine Script work:

- Type `/pinescript` to explicitly activate it
- Ask Claude to "write a pine script", "fix this pine compile error", or "build a tradingview strategy"
- Work with any `.pine` file

Once active, Claude Code will enforce Pine Script's grammar rules as hard constraints rather than suggestions -- the same way a linter would, but at generation time.

### Example prompts

```
/pinescript write an RSI divergence indicator with alerts

/pinescript convert this v5 strategy to v6

fix this compile error: Cannot call 'plot' in local scope
```

---

## Why this exists

Pine Script v6 has unusual constraints that trip up every major LLM:

1. **Functions cannot be nested** -- `myFunc(x) =>` must be at root indentation, never inside `if`/`for`/`switch`
2. **`plot()` and friends are global-only** -- no conditional plotting; use conditional *values* with a global `plot()`
3. **`ta.*` functions must run on every bar** -- putting `ta.ema()` inside an `if` block corrupts its history
4. **`var`-state functions break in `request.security` tuples** -- they need separate calls
5. **v6 lazy `and`/`or` silently skips right operands** -- accumulators in the right branch never execute
6. **Deprecated names still exist in training data** -- `sma()` vs `ta.sma()`, `security()` vs `request.security()`

Without this skill, Claude will generate code that looks correct but fails to compile or silently repaints. With it, code compiles on the first try.

---

## Versioning

Current version: **2.0.0**

The skill targets **Pine Script v6** by default and includes a v5 compatibility section for working with older scripts.

---

## License

MIT
