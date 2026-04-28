---
inclusion: always
---

# UX Standards — Lorentz Scalping Engine

Inspired by TradingView, Zerodha Kite Terminal, and modern algo trading dashboards.

## Core Design Philosophy

- **Information density over whitespace** — traders need data, not breathing room
- **Zero-latency feel** — every update must feel instant (WebSocket-driven, no polling)
- **Clarity under pressure** — layout must be readable in fast-moving markets
- **The Insight Panel is the hero** — it's the system's key differentiator; give it prominence

## Color System

```
Background:     #0d0f14  (near-black, reduces eye strain)
Surface:        #141720  (cards, panels)
Surface-raised: #1c2030  (hover states, active rows)
Border:         #2a2f45  (subtle separators)

BUY / Profit:   #00c896  (teal-green — standard trading green)
SELL / Loss:    #ff4d6a  (red — standard trading red)
Warning:        #f5a623  (amber — for regime warnings, suppressed signals)
Info:           #4a9eff  (blue — neutral data, timestamps)
Muted:          #6b7280  (secondary labels, inactive states)

Text-primary:   #e8eaf0
Text-secondary: #9ca3af
```

## Typography

- **Font**: `JetBrains Mono` for numbers/prices (monospace prevents layout shift on updates)
- **Font**: `Inter` for labels, headings, UI text
- **Price/P&L**: Always monospace, right-aligned
- **Size scale**: 11px (dense data) → 13px (body) → 15px (panel headers) → 20px (spot price)

## Layout — Panel Grid

```
┌─────────────────────────────────────────────────────────┐
│  HEADER: BN Spot | Regime Badge | Version Badge | Status │
├──────────────┬──────────────────┬───────────────────────┤
│              │                  │                       │
│  Market View │   Signal Panel   │    Insight Panel      │
│  (BN spot +  │   (live signals  │    (Decision Context  │
│  options     │   + suppressed)  │    + Learning Report) │
│  chain)      │                  │                       │
│              │                  │                       │
├──────────────┴──────────────────┤                       │
│                                 │                       │
│         Trade Table             │                       │
│   (open trades | closed trades) │                       │
│                                 │                       │
├─────────────────────────────────┴───────────────────────┤
│  CONTROLS: Start/Stop | Config (Core) | [Advanced ▼]    │
└─────────────────────────────────────────────────────────┘
```

- Insight Panel spans full height on the right — it's the most important panel
- Trade Table spans the bottom-left two-thirds
- Controls live in a collapsible footer bar

## Panel-Specific Rules

### Header Bar
- BN spot price: large, monospace, color-coded (green if up from prev close, red if down)
- Regime badge: pill shape — color matches regime (green=TRENDING_UP, red=TRENDING_DOWN, amber=RANGING, purple=HIGH_VOL, gray=UNKNOWN)
- Strategy version badge: `v1.2.3` style, always visible
- Connection status: green dot (connected) / red dot (disconnected) with label
- Engine status: `RUNNING` (green) / `STOPPED` (red) / `PAPER` (amber)

### Market View Panel
- Spot price with tick-level updates (flash animation on change)
- Options chain table: Strike | CE LTP | CE OI | PE LTP | PE OI
- Highlight ATM row with a subtle border
- Color OI changes: green if OI increasing, red if decreasing

### Signal Panel
- Each signal row: `[TIME] [BUY/SELL] [PRICE] [RSI] [METHOD] [REGIME]`
- BUY rows: left border teal, SELL rows: left border red
- Suppressed signals: dimmed with amber `SUPPRESSED` badge + reason on hover
- Structure ID shown as a short hash (e.g., `#a3f2`) — same structure groups visually
- Max 50 rows, auto-scroll to latest

### Trade Table
- Two tabs: `OPEN (n)` | `CLOSED (n)`
- Open trades: animate P&L cell on every premium update
- P&L: green if positive, red if negative, bold when > 1% of capital
- Inline badges per row: `[REGIME]` `[TIME_BUCKET]` `[VERSION]`
- Click any row → loads full Decision_Context in Insight Panel
- Exit buttons inline: `Exit Now` (red) | `Hold TP` (amber)

### Insight Panel (Most Important)
- Two sub-tabs: `Trade Reasoning` | `Learning Report`
- **Trade Reasoning**: structured card layout
  ```
  Signal Features (at candle close)
    RSI: 58.2          Slope Method: ATR
    Upper TL: 48,420   Lower TL: 47,980
    Regime: TRENDING_UP   Structure: #a3f2
    Time Bucket: MORNING

  Execution Features (at fill)
    Entry: ₹218.50     Slippage: ₹1.50
    SL: ₹195.00        TP: ₹262.00
    Version: v1.2.0

  Outcome
    Exit: TAKE_PROFIT  P&L: +₹645
    Candles held: 3
  ```
- **Learning Report**: actionable suggestions rendered as cards
  - Green card: "RSI > 55 improves win rate +12% → Suggested: raise threshold"
  - Red card: "RANGING regime trades: 23% win rate → Suggested: disable"
  - Each card has `Apply` button that pre-fills the config change

### Controls Panel (Footer)
- **Core config** (always visible): RSI threshold, max trades, capital, SL mode
- **Advanced config** (collapsed by default, `[Advanced ▼]` toggle): all other params
- Start/Stop engine: large button, prominent
- Trigger Learning: secondary button with spinner on active
- Config changes: show `Unsaved changes` indicator, `Apply` button

## Interaction Patterns

### Keyboard Shortcuts
| Key | Action |
|-----|--------|
| `Space` | Start / Stop engine |
| `L` | Trigger learning cycle |
| `E` | Exit selected open trade |
| `I` | Focus Insight Panel |
| `C` | Focus Controls |
| `Esc` | Deselect / close modal |
| `1–5` | Switch between panels (mobile) |

### Real-Time Updates
- Use WebSocket for all live data — no polling
- Price flash: 200ms highlight animation on tick update
- P&L flash: green flash on profit increase, red on loss increase
- New signal: slide-in animation from top of signal panel
- Trade closed: row moves from OPEN to CLOSED with fade transition

### Loading & Error States
- Skeleton loaders (not spinners) for initial panel load
- Connection lost: full-width amber banner "Reconnecting to data feed..."
- Empty states: contextual messages ("No signals yet — engine is warming up")
- Failure panel: red badge count in header, expandable list

## Responsive Behavior
- Primary target: 1440px+ desktop (trading workstation)
- 1280px: collapse Market View into a compact ticker bar
- Below 1024px: single-column stack, tabs for panel switching
- No mobile-first — this is a desktop trading tool

## Accessibility Baseline
- All color-coded information also has text labels (never color-only)
- Focus rings visible for keyboard navigation
- ARIA labels on all icon-only buttons
- Reduced motion mode: disable flash animations

---

## 2026 Enhancements

### 1. Cognitive Load Mode (Adaptive UI Density)

The UI auto-switches density based on market conditions:

| Mode | Trigger | What's visible |
|---|---|---|
| NORMAL | Default | All panels |
| FOCUSED | Active open trade | Hide secondary labels, show only active trades + latest signals |
| CRITICAL | HIGH_VOL regime or ATR spike | Spot price + open trades + exit controls only |

- Mode indicator in header: `NORMAL` / `FOCUSED` / `CRITICAL` pill
- Manual override: user can lock to any mode via header toggle
- CRITICAL mode: background shifts to deep red tint, all non-essential UI fades to 20% opacity

### 2. Signal Confidence Visualization

Every signal row shows a confidence bar derived from breakout strength, RSI distance from threshold, slope steepness, and regime alignment:

```
[BUY] ███████░░ 0.72   10:25  ₹48,320  RSI:58  ATR  TRENDING_UP
[SEL] ████░░░░░ 0.44   10:30  ₹48,180  RSI:46  STD  RANGING
```

- Bar color: teal (BUY) / red (SELL), fades to muted below 0.4
- Score 0.0–1.0, shown to 2 decimal places
- Low confidence (< 0.5): row dimmed, no auto-focus in Insight Panel

### 3. "Why NOT Trade" — Suppression Transparency

Suppressed signals are first-class citizens in the UI, not footnotes:

```
SUPPRESSED  10:22  BUY  ₹48,290
  ├─ Same structure (#a3f2)
  ├─ RANGING regime (filter active)
  └─ Concurrent trade limit reached
```

- Shown inline in Signal Panel with amber left border
- Expandable — click to see full suppression context in Insight Panel
- Suppression count badge on Signal Panel header: `Signals (12) | Suppressed (4)`

### 4. Microstructure Timeline (Insight Panel)

Every trade in the Insight Panel shows a horizontal event timeline:

```
  BREAKOUT    CONFIRM     ENTRY      TP HIT
  ──●──────────●───────────●──────────●──
  10:20       10:25       10:25      10:35
              +5min        0s        +10min
```

- Latency between each stage shown below the timeline
- Helps identify: slow confirmation, entry delay, time-to-profit patterns
- Color-coded nodes: amber (breakout), blue (confirm), teal/red (entry), green (TP) / red (SL)

### 5. Regime Overlay on Market View

Beyond the header badge, regime is visualized in the Market View panel:

- Background tint: subtle green (TRENDING_UP), red (TRENDING_DOWN), amber (RANGING), purple (HIGH_VOL)
- Options chain: ATM row border color matches current regime
- Regime history strip: thin horizontal bar below the options chain showing last 10 regime labels as colored blocks

### 6. Trade Quality Score

Each closed trade row in the Trade Table shows a letter grade:

```
Trade #a3f2  BUY  +₹645  [A]
Trade #b7c1  SEL  -₹210  [D]
```

Grading rubric:
- A: entry in correct regime + RSI aligned + TP hit within 3 candles
- B: correct regime, TP hit but took 4–5 candles
- C: correct regime, time-exit or small win
- D: wrong regime, SL hit, or suppressed-then-entered

- Grade stored in Decision_Context as `trade_quality`
- Learning Engine uses grade as a weight in clustering

### 7. Interactive Learning → Simulate Impact

Each insight card in the Learning Report has three actions:

```
┌─────────────────────────────────────────────────────┐
│ RSI > 55 improves win rate +12%                     │
│ Based on 47 trades over 8 sessions                  │
│                                                     │
│  [Simulate Impact]  [Apply]  [Ignore]               │
└─────────────────────────────────────────────────────┘
```

- `Simulate Impact`: shows projected win rate and P&L delta without applying
- `Apply`: pre-fills the config change in Controls panel with `Unsaved changes` indicator
- `Ignore`: dismisses card, logs the ignore action for meta-learning

### 8. Strategy Drift Indicator

Header badge monitors system health vs baseline:

```
DRIFT: LOW ✓     (green)
DRIFT: MED ⚡    (amber)
DRIFT: HIGH ⚠    (red)
```

Computed from: recent win rate vs 20-trade rolling baseline + variance spike detection.
Click → opens drift detail modal showing win rate trend chart.

### 9. Execution Latency Indicator

Header shows real-time tick-to-signal latency:

```
Latency: 42ms  ●  (green < 50ms)
Latency: 120ms ●  (amber < 150ms)
Latency: 210ms ●  (red > 150ms)
```

- Measured: tick received → signal emitted
- Helps debug WebSocket lag, candle builder delays

### 10. Replay Mode

Select any closed trade → `[Replay]` button in Insight Panel:

- Replays candle-by-candle from N candles before entry
- Shows: evolving trendlines, RSI, regime, breakout state machine transitions
- Decision points highlighted: when breakout was detected, when confirmation fired
- Playback speed: 1× / 5× / 10×
- Keyboard: `→` next candle, `←` previous, `Space` play/pause

### 11. Parameter Heatmap

In the Learning Report / Experiment Framework panel:

```
RSI Threshold  │  Win Rate
───────────────┼──────────
45             │  ████░░░░  52%
50             │  █████░░░  58%
55             │  ███████░  64%  🔥 best
60             │  ██████░░  61%
```

- 2D heatmap available for two-parameter combinations (e.g., RSI × slope_multiplier)
- Color: green (high win rate) → red (low win rate)
- Click any cell → shows the trades that make up that bucket

### 12. Multi-Strategy Comparison Panel

Visible when more than one strategy is registered:

```
Strategy │ Win Rate │ P&L    │ Trades │ Active
─────────┼──────────┼────────┼────────┼───────
S1       │ 58%      │ +₹12k  │ 47     │  ✅
S2       │ 62%      │ +₹15k  │ 31     │  ❌
```

- One-click activate/deactivate per strategy
- Requires confirmation if switching mid-session with open trades

### 13. Human Override Memory

When a trader manually overrides (Exit Now / Hold TP / Disable strategy), the system logs:

```json
{
  "override": "HOLD_BEYOND_TP",
  "trade_id": "...",
  "reason": "manual",
  "outcome": "better"
}
```

- Override history visible in Insight Panel under `Override Log` tab
- Learning Engine uses override outcomes as training signal
- Pattern: "User holds beyond TP in TRENDING_UP → outcome better 70% of time" → surfaced as insight

### 14. Smart Alerts

Notifications only fire for high-signal events:
- Confidence score > 0.8 on new signal
- Regime change (e.g., RANGING → TRENDING_UP)
- P&L swing > 1% of capital in single candle
- DRIFT: HIGH detected
- Data gap > 30 seconds

Delivered as: non-blocking toast (bottom-right), auto-dismiss 5s, sound optional.

### 15. Visual Risk Meter

Header bar shows live exposure:

```
Risk  ███░░  35%   Capital used: ₹35,000 / ₹1,00,000
```

- Computed from: capital in open trades + unrealised loss + volatility multiplier
- Color: green (< 40%) → amber (40–70%) → red (> 70%)
- Click → breakdown modal: per-trade exposure + daily loss tracker

### 16. Fail-Safe UX

When a critical failure is detected (data gap, execution error, WebSocket drop):

```
┌─────────────────────────────────────────────────────┐
│  ⚠  Execution paused — data gap detected (14s)      │
│  Engine halted. Open trades frozen. [Resume] [Exit All] │
└─────────────────────────────────────────────────────┘
```

- Full-width red banner, blocks new signal processing
- UI state frozen — no stale data shown
- Two actions only: Resume (when feed restored) or Exit All (close all open trades at last known price)

---

## Panel Fixes (Important)

### Insight Panel — Always Sticky
- Never collapses, never hidden regardless of screen size
- Auto-focuses when a signal is clicked anywhere in the UI
- Keyboard shortcut `I` always brings focus here

### Signal Panel — Grouped by Structure ID
```
#a3f2  ──────────────────────────────
  BUY       10:20  ₹48,320  conf:0.72
  SUPPRESSED 10:25  same structure
#b7c1  ──────────────────────────────
  SELL      10:30  ₹48,180  conf:0.44
```
Groups show the market story — same structure, multiple attempts.

### Trade Table — MAE / MFE Columns
Closed trades show:
- MAE (Max Adverse Excursion): worst drawdown during the trade
- MFE (Max Favorable Excursion): best unrealised profit during the trade
- These feed directly into Learning Engine quality scoring

### Controls — Safety Confirmations
Confirmation modal required for:
- Switching config while trades are open: "2 trades open — apply config change?"
- Stopping engine while trades are open: "Stop engine and exit all trades?"
- Rollback to previous strategy version mid-session
