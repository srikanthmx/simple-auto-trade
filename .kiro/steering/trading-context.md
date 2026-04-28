---
inclusion: always
---

# Trading Context — Lorentz Scalping Engine

## Bank Nifty Fundamentals

- **Instrument**: Bank Nifty (BANKNIFTY) — NSE India's banking sector index
- **Lot size**: 15 units (post-Nov 2024 revision)
- **Expiry**: Weekly (every Wednesday), monthly (last Thursday)
- **Trading hours**: 09:15 – 15:30 IST
- **Tick size**: ₹0.05 for options
- **Settlement**: Cash-settled in INR

## Intraday Session Structure

| Session | Time (IST) | Behavior | Strategy Bias |
|---|---|---|---|
| OPEN | 09:15 – 09:45 | High volatility, price discovery, institutional order flow | Avoid first 5 min; breakouts after 09:20 are valid |
| MORNING | 09:45 – 11:30 | Trend establishment, highest volume | Best for Lorentz breakout entries |
| MIDDAY | 11:30 – 13:30 | Consolidation, choppy, low momentum | Reduce position size; RANGING regime likely |
| AFTERNOON | 13:30 – 14:45 | Second trend window, institutional re-entry | Valid breakout window |
| CLOSE | 14:45 – 15:30 | Expiry-day gamma spike / EOD unwinding | High risk; avoid new entries after 15:00 |

**Key rule**: Trades entered after 15:00 IST should be suppressed by default (configurable).

## Options Scalping Rules (Current Best Practice)

### Strike Selection
- **ATM (At-the-Money)**: Highest liquidity, tightest bid-ask spread — preferred for scalping
- **1 ITM (In-the-Money)**: Higher delta (~0.6), moves more with spot — used when premium range allows
- **Premium sweet spot for scalping**: ₹150–₹300 (Mode B default)
  - Below ₹100: too little absolute move per point
  - Above ₹400: capital-intensive, wider spreads
- **Avoid deep OTM** on scalp entries — theta decay kills premium fast

### Entry Discipline
- Never enter on the breakout candle itself — wait for confirmation close (your Pine Script already does this)
- ATM options move ~0.4–0.6 per 1 point move in BN spot (delta approximation)
- For a 100-point BN move, expect ₹40–₹60 premium move on ATM option
- Slippage on ATM BN options: typically ₹1–₹3 per lot in normal conditions; ₹5–₹10 in high-vol

### Risk Management
- **1:2 RR minimum** — your script enforces this
- **Max loss per trade**: 1–2% of capital (₹1,000–₹2,000 on ₹1L capital)
- **Max daily loss**: 3–5% of capital — system should halt after this threshold
- **Position sizing**: 1 lot default; scale only after 20+ trade sample with positive expectancy

### Regime Awareness
- **TRENDING_UP / TRENDING_DOWN**: Lorentz breakouts perform best — full position
- **RANGING**: High false breakout rate — reduce size or skip
- **HIGH_VOL** (ATR spike, VIX > 20): Widen SL, reduce size — premium moves are erratic
- **UNKNOWN**: Conservative mode — paper only

## Key Indicators Used in This System

| Indicator | Purpose | Default |
|---|---|---|
| ATR(14) | Volatility measure, slope method selector, SL sizing | 14 periods |
| RSI(14) | Momentum filter — BUY only if RSI > 50, SELL only if RSI < 50 | 14 periods, threshold 50 |
| EMA(39) on HLC3 | Trend bias (from Pine Script) | 39 periods |
| Pivot High/Low | Structural swing points for trendline anchoring | 14-bar lookback |
| VWAP | Institutional reference level — signals near VWAP have higher conviction | Session-reset |

## Lorentz Strategy Logic (Pine Script → Python)

### Slope Method Selection
```
if ATR(14) > 1.5  → use ATR slope   (high volatility)
if ATR(14) > 0.5  → use Stdev slope (medium volatility)
else              → use LinReg slope (low volatility)
```

### Breakout State Machine
```
IDLE
  → pivot detected → BREAKOUT_DETECTED (store breakout high/low)
  → confirmation candle closes beyond breakout level + RSI filter → CONFIRMATION
  → signal emitted → SIGNAL_EMITTED
  → reset to IDLE after signal or on new pivot
```

### Exit Rules
- SL: low of signal candle (BUY) / high of signal candle (SELL) — CANDLE_BASED_SL (default)
- TP: entry + 2× (entry − SL) — 1:2 RR
- Time exit: 5 candles (configurable to 6) — prevents overnight/stale holds

## Common Failure Modes to Watch

- **False breakout in RANGING regime**: Most common loss pattern — regime filter is critical
- **Overtrading same structure**: structure_id deduplication prevents this
- **Open drive fakeout (09:15–09:20)**: First 5 minutes are noise — suppress signals
- **Expiry day gamma risk**: Wednesday/Thursday — premiums move violently; tighten SL
- **Low liquidity strikes**: Options with < 500 OI should be avoided in contract selection
