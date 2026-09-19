# Gap Up Opening Range Breakout (Gap Up ORB) — Methodology Specification

This document formalizes the *Intraday Trading – Gap Up ORB Trading Manual* into a
precise, hierarchical, implementation-ready spec. It is the human-readable
companion to [`spec/gap_up_orb_spec.json`](../spec/gap_up_orb_spec.json) (the
machine-readable source of truth both reference implementations load from) and
to two runnable implementations of everything below:

- **`chart/source/orb_engine.js`** — the detection/scoring logic, in JS, driving the interactive chart
- **`scanner/`** — a Python package that scans OHLCV data (one ticker or thousands) for this setup and emits labeled feature records

Every rule below carries a **provenance tag**:

| Tag | Meaning |
|---|---|
| `[manual]` | Stated explicitly in the source PDF — numbers/rules are unaltered |
| `[standard]` | A widely-used, named industry-standard formula the manual assumes without restating (e.g. it labels "20d ADR%" on every chart but doesn't spell out the arithmetic behind the % value) |
| `[configurable]` | The manual describes this qualitatively ("significant", "large") without a number. A default is provided; tune it on your own data. |
| `[addition]` | Not in the manual at all — an extra signal or scanning convenience the reference engines add on top. Called out explicitly in §2.6 so it's never mistaken for a documented rule. |

Some rules split into two tags (e.g. ADR%'s 20-day window is `[manual]` —
it's labeled on every chart — while the arithmetic behind it is
`[standard]`). Anything not tagged is descriptive scaffolding, not a rule.

---

## 1. What this setup is

A **long-only** intraday breakout setup. The stock gapped up overnight, and
price subsequently breaks above the high of its opening range — with several
independent confirmations available to weigh the trade's quality.

### 1.1 Qualifying gap causes `[manual]`

The manual restricts the setup to three contexts. A gap that doesn't fit one
of these is out of scope:

1. **Earnings reaction** — gapping up on a positive earnings report
2. **News catalyst** — gapping up on a significant company-specific news event
3. **Group-wide move** — gapping up alongside peers on a sector/theme catalyst

> **Automation note:** distinguishing these three causes from OHLCV data alone
> is not reliably possible — it requires an earnings calendar / news feed.
> Both reference implementations accept an optional `earnings_dates` list to
> tag case 1 automatically; everything else is tagged `unclassified_catalyst`
> rather than guessed.

### 1.2 Timeframes `[manual]`

| Timeframe | Usage |
|---|---|
| 5-minute | Primary |
| 15-minute | Primary |
| 30-minute | Primary |
| 1-minute | Occasional |

The manual is explicit that timeframe choice is discretionary — driven by how
price action unfolds that day and by a given stock's historical character on
each timeframe (e.g. some names' 15-min ORB win rate beats their 5-min). This
system does not pick a timeframe for you: it evaluates each one independently
and reports each as its own setup instance.

---

## 2. Daily context filter (the "is this stock even tradeable" gate)

Evaluated on **daily bars**, using the **prior day's close** — i.e. context
that existed *before* the gap, not created by it. If any required condition
here fails, the gap is not a qualifying setup regardless of how clean the
intraday breakout looks.

### 2.1 The five daily moving averages `[manual]`

| Key | Type | Period | Manual's chart color |
|---|---|---|---|
| `ema10` | EMA | 10 | Black |
| `ema21` | EMA | 21 | Blue |
| `sma50` | SMA | 50 | Purple |
| `sma100` | SMA | 100 | Grey |
| `sma200` | SMA | 200 | Red |

*(On a dark chart theme, black is swapped for white/light-grey so the line
stays visible — see the chart tool's legend.)*

### 2.2 Required: price above all five `[manual]`

> *"At time of entry, price is above all key daily MAs."*

`close[prior day] > ema10 AND > ema21 AND > sma50 AND > sma100 AND > sma200`

This is a **hard gate** — not a confirmation to weigh, a precondition.

### 2.3 ADR% — Average Daily Range % `[period: manual · formula: standard]`

Every single example chart in the manual labels this **`20d ADR%`** directly
— so the 20-day window itself is explicit, not inferred. What the source
pages *don't* show is the arithmetic behind the resulting percentage. This
is the standard definition used across momentum/swing trading education:

```
ADR% = 100 × ( SMA(High / Low, 20) − 1 )
```

*(Not the same as `100 × SMA((High−Low)/Close, 20)` — a different, less common variant. This system uses the ratio-of-High-to-Low form.)*

### 2.4 Volume vs. 30-day average `[period: manual · threshold: configurable]`

> *"Significant volume increase vs 30d avg on the close."*

```
ratio = gap_day_volume / SMA(volume, 30)
```

The 30-day lookback is explicit in the manual. What counts as "significant"
is not — this system defaults to **ratio ≥ 1.5** (50% above average) and
treats that threshold as tunable.

### 2.5 Relative Strength (RS) line at a 52-week high `[standard]`

> *"RS 52WHs"* — shown as blue dots under the daily chart in every example.

```
rs_line = close / benchmark_close        (benchmark defaults to SPY)
rs_new_52w_high = rs_line[today] >= max(rs_line[today-251 .. today])
```

The manual shows the marker but not the benchmark ticker or exact lookback;
252 trading days (the standard 52-week convention) and SPY are the defaults.
Swap the benchmark for something more relevant to the stock's peer group
(e.g. QQQ, SOXX, XBI) if that makes more sense for the name you're scanning.

### 2.6 Additions beyond the manual `[addition]`

Two things the reference engines compute that are **not** interpretations
of anything in the manual — pure extras layered on top. Listed here on
their own, separately from everything else in this document, specifically
so they can't be mistaken for documented rules:

- **RS leading price** — a stronger variant of §2.5: true when the RS line
  makes a new N-day high while the stock's own closing price has *not yet*
  made a corresponding new high. The manual's charts only ever show plain
  "RS 52WHs" (§2.5) — they do not distinguish a "leading" case anywhere in
  the source pages. This was added by extending that same computation one
  step further, inspired by a related "yummy" / "very yummy" RS-new-high
  concept already present in the user's own pre-existing `chart_engine.js`
  (its `chillLaxRS()` function, itself ported from an external source
  unrelated to this manual) — not by anything in the PDF.
- **Minimum gap %** — a scanning convenience (default 2%) so a batch scan
  across many days doesn't flag every negligible gap as a candidate. The
  manual gives no minimum gap size; every one of its examples happens to be
  a large, obvious gap, but no threshold is stated. See §5 for where this
  sits in the checklist (it's a gate, but an invented one).

---

## 3. Intraday setup (evaluated on the gap day itself)

### 3.1 The two intraday EMAs `[manual]`

| Key | Type | Period | Manual's chart color |
|---|---|---|---|
| `ema6` | EMA | 6 | Orange |
| `ema21` | EMA | 21 | Blue |

### 3.2 Daily MA levels overlaid on the intraday chart `[manual]`

> *"Key daily MA price levels from daily chart plotted on 5-minute chart."*

The current `ema10`, `ema21`, and `sma50` values (from §2.1) are drawn as flat
horizontal reference lines across the intraday chart — a visual check that
intraday price action is respecting the same levels that matter on the daily
chart. Purely a visualization; the underlying values are the daily-side ones.

### 3.3 Opening range `[standard]`

```
session_open  = first bar at/after 09:30 America/New_York
opening_range = { high: session_open.high, low: session_open.low }
```

i.e. the opening range is the high/low of the **first bar** of the regular
session on the selected timeframe (the 09:30–09:35 candle on a 5-min chart,
09:30–09:45 on 15-min, etc). The manual assumes this definition from "The
Blueprint Trading Course" prerequisite and doesn't restate it on the pages
provided — this is the conventional ORB definition.

### 3.4 Breakout trigger `[manual]`

> *"Xm ORB Entry + price above intraday MAs."*

```
breakout_bar = first bar after the opening range where:
    close > opening_range.high
    AND close > ema6[bar]
    AND close > ema21[bar]

entry_price = opening_range.high     (the breakout level itself —
                                       the conventional way an ORB long is
                                       executed, as a stop-buy at the range
                                       high, not at the confirming bar's close)
```

### 3.5 Custom MACD (6 / 20 / 9) `[manual]`

Exact settings from the manual's indicator-settings screenshot:

| Setting | Value |
|---|---|
| Fast length | 6 |
| Slow length | 20 |
| Signal smoothing | 9 |
| Source | Close |
| Oscillator MA type | EMA |
| Signal line MA type | EMA |

**Bullish signal** `[standard definition]`: MACD line crosses above the
signal line. Evaluated at the breakout bar as two related facts — whether a
fresh crossover occurred between the opening range and the breakout, and
whether MACD is still above signal (not stale) at the moment of entry.

### 3.6 Relative volume `[configurable]`

> *"Large relative volume"* on the breakout bar (visually flagged, no number given).

```
ratio = breakout_bar.volume / mean(volume, prior 20 bars)
large = ratio >= 2.0   (default; tune it)
```

### 3.7 Initial reaction positive `[configurable formalization]`

> *"Initial reaction positive"* — a qualitative read on the first bar(s) after the gap.

```
first_bar.close > first_bar.open
AND (first_bar.close - first_bar.low) / (first_bar.high - first_bar.low) >= 0.5
```

i.e. a green first bar that closes in the upper half of its own range.

---

## 4. Risk management

### 4.1 Initial stop-loss `[manual]`

> *"Initial SL: LOD by $0.01."*

```
stop_price = min(low, from session start through the entry bar) − 0.01
```

The lookback spans the full trading day's data available up to and including
the entry bar — including pre-market bars, if present in the feed, which is
why the manual's charts shade pre-market sessions on the same panel as the
entry/stop markers.

### 4.2 Initial risk % `[manual]`

```
risk_$   = entry_price − stop_price
risk_pct = 100 × risk_$ / entry_price
```

Shown on every example chart (e.g. `-3.10%`, `-150` ticks) as the trade's
defined risk before entry.

### 4.3 Position sizing `[standard — not a manual-specified number]`

The manual computes and displays *Initial Risk %* on every example — how far
the stop sits from entry — which this system implements exactly (§4.2). It
does **not** specify what percentage of account equity to risk per trade;
that's an individual risk-tolerance choice, so it's a required, user-set
input here:

```
shares = floor( account_size × (risk_per_trade_pct / 100) / risk_$ )
```

`risk_per_trade_pct` defaults to a conventional **1%** but is a parameter,
not a manual rule.

---

## 5. The complete checklist

This is what both reference implementations compute per candidate gap day —
three **hard gates** (all required) plus six **confirmations** (weighed, not
required, and surfaced as a 0–1 `setup_quality_score`). Two of these nine
items are `[addition]` — not from the manual at all (§2.6) — the other
seven all trace back to something the manual states, shows, or labels:

**Gates (all must pass for `is_valid_setup`):**
- [ ] Gap ≥ minimum size (`[addition]`, default 2% — §2.6. The manual states
      no minimum gap size at all; this exists purely so a batch scan
      doesn't flag negligible gaps)
- [ ] Price above all 5 daily MAs as of the prior close (§2.2, `[manual]`)
- [ ] ORB breakout found on the selected timeframe (§3.4, `[manual]`)

**Confirmations (count toward `setup_quality_score`):**
- [ ] MACD bullish (line above signal) at entry (§3.5, `[manual]` settings / `[standard]` crossover definition)
- [ ] Daily volume ≥ 1.5× the 30-day average (§2.4, `[manual]` period / `[configurable]` threshold)
- [ ] Relative (intraday) volume ≥ 2× the 20-bar average at the breakout bar (§3.6, `[configurable]`)
- [ ] RS line at a new 52-week high (§2.5, `[standard]`)
- [ ] RS line leading price (§2.6, `[addition]` — not distinguished by the manual)
- [ ] Initial reaction positive (§3.7, `[configurable]`)

---

## 6. What this system does *not* specify

Flagged explicitly rather than silently invented anywhere in the code. (This
is different from §2.6 — those are gaps the reference engines *did* choose
to fill, each clearly marked `[addition]`; these are gaps left alone.)

- **Profit-taking / take-profit rule** — not in the source pages provided
- **Trade management after entry** — trailing-stop method, scaling out, time-based exits
- **Exact numeric thresholds** for "large" relative volume or "significant" daily volume (defaults given above, tagged `[configurable]`)
- **Overhead resistance zone construction** — appears in the TSLA example as a discretionary, hand-drawn annotation, not a rule-based indicator; not automated here
- **Exact % of account equity to risk per trade** — a personal choice, exposed as a required parameter rather than assumed

If you have the rest of "The Blueprint Trading Course" this manual
references, the gaps above (particularly exits/trade management) are the
places to extend `spec/gap_up_orb_spec.json` and the two engines from.

---

## 7. Machine-readable spec & the scanner's output schema

### 7.1 Input spec

[`spec/gap_up_orb_spec.json`](../spec/gap_up_orb_spec.json) (and its
generated YAML mirror) contain every parameter above in a form both engines
load directly — change a number there and both the chart tool and the Python
scanner pick it up.

### 7.2 Output schema (one record per candidate gap day — the ML-friendly part)

Both `orb_engine.js`'s `evaluateSessions()` and the Python package's
`evaluate_sessions()` emit the same flat, typed record shape:

```jsonc
{
  "ticker_date": "2026-09-14",
  "timeframe": "5min",
  "gap": { "pct": 15.0, "cause": "unclassified_catalyst" },
  "daily_context": {
    "prior_close": 100.0,
    "ma_values": { "ema10": 99.34, "ema21": 98.53, "sma50": 96.41, "sma100": 92.74, "sma200": 85.40 },
    "price_above_all_mas": true,
    "adr_pct_20d": 0.95,
    "volume_ratio_vs_30d_avg": 3.90,
    "volume_significant": true,
    "rs_line_value": 0.2329,
    "rs_new_52w_high": true,
    "rs_leading_price": false
  },
  "intraday_setup": {
    "session_open_idx": 238,
    "opening_range": { "high": 117.30, "low": 114.42, "startIdx": 238, "endIdx": 238 },
    "initial_reaction_positive": true,
    "breakout": { "found": true, "time": 1789394700, "idx": 245, "breakoutBarClose": 118.47 },
    "macd_at_breakout": { "bullishCrossPresent": false, "macdAboveSignalAtEntry": true },
    "relative_volume_at_breakout": 12.09,
    "relative_volume_large": true
  },
  "risk": { "riskDollar": 17.61, "riskPct": 15.01, "stopPrice": 99.69, "entryPrice": 117.30 },
  "gates": { "gapUpMeetsMinPct": true, "priceAboveAllDailyMAs": true, "orbBreakoutFound": true },
  "gates_all_passed": true,
  "confirmations": { "macdBullish": true, "dailyVolumeSignificant": true, "relativeVolumeLarge": true, "rsLine52wHigh": true, "rsLeadingPrice": false, "initialReactionPositive": true },
  "confirmation_count": 5,
  "confirmation_total": 6,
  "is_valid_setup": true,
  "setup_quality_score": 0.833
}
```

Every leaf is a named bool/number/string — flatten this straight into a
dataframe row. `is_valid_setup` is a ready-made binary label;
`setup_quality_score` a ready-made continuous one; every gate/confirmation
and underlying value is a candidate feature. Nothing about this shape is
tied to any one ticker or date range, which is what makes it batchable
across a whole scan universe (§8).

---

## 8. Reference implementations

### 8.1 `chart/source/orb_engine.js` — powers the interactive chart

Pure functions, load order: `chart_engine.js` → `orb_engine.js` (the latter
reuses the former's `sma`/`ema`/`macd` rather than re-deriving them — see the
additive export at the bottom of `chart_engine.js`). Runs identically in a
browser `<script>` tag or under Node; `chart/source/test_orb_engine.js` is
the test suite (real SOXX/QQQ data + constructed positive/negative
fixtures), and `chart/source/test_app_demo_data.js` separately verifies the
chart's own demo-data generator produces a valid, gates-passing example.

### 8.2 `scanner/` — Python package for batch scanning

See [`scanner/README.md`](../scanner/README.md). Scans one CSV, a directory
of them, or a live data source you wire up, and writes the record shape from
§7.2 to CSV/JSON/Parquet for further analysis or as ML training data.

### 8.3 The chart tool

A self-contained HTML file (open it directly, no server) that renders the
daily context panel, the intraday breakout panel with MACD/volume, a live
pass/fail checklist matching §5, and a position-size calculator.

---

## Disclaimer

This document and the accompanying code are a technical formalization of a
trading methodology for educational and tooling purposes — they are not
financial advice, and nothing here is a recommendation to buy or sell any
security. Rule-based pattern detection on historical data does not guarantee
future performance; past gap-and-breakout patterns matching this checklist
have no guaranteed relationship to future returns. Position sizing and stop
placement reduce risk but don't eliminate it — trading involves the risk of
loss. Backtest and paper-trade before risking real capital, and size
positions according to your own risk tolerance.
