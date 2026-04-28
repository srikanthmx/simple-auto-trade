---
inclusion: always
---

# Architecture Standards — Lorentz Scalping Engine

## Stack

| Layer | Technology | Notes |
|---|---|---|
| Backend | Python 3.11+ | Async-first via `asyncio` |
| Data feed | Upstox WebSocket v2 | BN spot + options chain |
| Internal bus | `asyncio.Queue` | Event-driven, no external broker needed for local |
| Storage | JSON Lines (`.jsonl`) | Append-only trade log; no DB dependency for v1 |
| Config | `settings.json` | Loaded at startup, hot-reloadable via API |
| Frontend | Next.js 14+ (App Router) | TypeScript, Tailwind CSS |
| UI state | Zustand | Lightweight, no Redux overhead |
| UI real-time | Native WebSocket | Backend exposes `ws://localhost:8765` |
| Backend API | FastAPI | REST for config/control + WebSocket for streaming |

## Backend Module Contracts

### Event Flow (strict ordering)
```
Upstox WebSocket
  → Data_Engine (tick normalization)
  → Candle_Builder (tick → 5m OHLCV)
  → Regime_Detector (per candle)
  → MTF_Context (15m bias, updated every 3 candles)
  → Lorentz_Signal_Engine (breakout state machine)
  → Signal_Deduplicator (structure_id check)
  → Time_Filter (session window check)
  → Execution_Engine (trade lifecycle)
  → Trade_Logger (Decision_Context write)
  → WebSocket_Broadcaster (→ UI)
```

### Signal Interface (shared by all strategies)
Every strategy MUST emit signals conforming to this schema:
```python
@dataclass
class Signal:
    id: str                    # uuid4
    strategy_id: str           # e.g. "S1"
    strategy_version: str      # e.g. "1.2.0"
    type: Literal["BUY", "SELL"]
    timestamp: datetime
    candle_close: float
    structure_id: str          # breakout sequence hash
    regime: str                # TRENDING_UP | TRENDING_DOWN | RANGING | HIGH_VOL | UNKNOWN
    time_bucket: str           # OPEN | MORNING | MIDDAY | AFTERNOON | CLOSE
    signal_features: dict      # frozen at candle close — never mutated after emit
    mtf_bias: str              # BULLISH | BEARISH | NEUTRAL
```

### Decision_Context Schema (canonical — all fields required)
```python
@dataclass
class DecisionContext:
    # Identity
    trade_id: str
    strategy_id: str
    strategy_version: str
    structure_id: str

    # Signal features (frozen at signal candle close)
    signal_features: SignalFeatures  # RSI, ATR, slope_method, upper_tl, lower_tl, pivots, regime, time_bucket, mtf_bias

    # Execution features (captured at fill)
    execution_features: ExecutionFeatures  # entry_price, slippage, sl, tp, sl_mode, premium, position_size

    # Lifecycle
    entry_timestamp: datetime
    exit_timestamp: datetime | None
    exit_reason: str | None          # STOP_LOSS | TAKE_PROFIT | TIME_EXIT | MANUAL
    candles_held: int | None
    pnl: float | None
    result: str | None               # WIN | LOSS | BREAKEVEN

    # Metadata
    suppressed: bool
    suppression_reason: str | None
```

### Module Boundaries (strict — no cross-imports)
```
data/          → emits raw ticks only, knows nothing about strategy
strategy/      → consumes candles, emits Signal objects only
execution/     → consumes Signal, manages trade state, emits TradeEvent
memory/        → consumes TradeEvent, writes DecisionContext, reads for learning
learning/      → reads DecisionContext, writes InsightReport
config/        → read by all modules, written only via API endpoint
```

No module may import from a module "downstream" of it in the flow above.

### Settings Schema (settings.json)
```json
{
  "core": {
    "capital": 100000,
    "max_concurrent_trades": 2,
    "rsi_threshold": 50,
    "sl_mode": "CANDLE_BASED",
    "time_exit_candles": 5,
    "options_mode": "MODE_B",
    "premium_min": 150,
    "premium_max": 300
  },
  "advanced": {
    "lookback_length": 14,
    "slope_multiplier": 1.6,
    "rsi_length": 14,
    "slippage_bps": 5,
    "slippage_enabled": true,
    "regime_filter_enabled": true,
    "dedup_enabled": true,
    "dedup_candle_window": 3,
    "mtf_enabled": true,
    "mtf_timeframe": "15m",
    "allowed_sessions": ["MORNING", "AFTERNOON"],
    "learning_trigger_time": "15:30",
    "max_daily_loss_pct": 3.0
  }
}
```

## Frontend Architecture

### Component Tree
```
app/
  layout.tsx          ← global providers (Zustand, WebSocket context)
  page.tsx            ← dashboard shell, panel grid
components/
  Header.tsx          ← spot price, regime badge, version, status
  MarketView.tsx      ← options chain table
  SignalPanel.tsx     ← live signal feed
  TradeTable.tsx      ← open/closed tabs
  InsightPanel.tsx    ← Decision_Context + Learning Report (HERO)
  Controls.tsx        ← core/advanced config, start/stop
  FailurePanel.tsx    ← recent failures list
lib/
  ws.ts               ← WebSocket singleton, reconnect logic
  store.ts            ← Zustand store (signals, trades, config, regime)
  types.ts            ← shared TypeScript types matching Python schemas
```

### WebSocket Message Protocol
Backend broadcasts JSON messages with a `type` field:
```typescript
type WSMessage =
  | { type: "tick";       data: TickData }
  | { type: "candle";     data: CandleData }
  | { type: "signal";     data: Signal }
  | { type: "trade_open"; data: DecisionContext }
  | { type: "trade_close";data: DecisionContext }
  | { type: "regime";     data: { regime: string; timestamp: string } }
  | { type: "insight";    data: InsightReport }
  | { type: "failure";    data: FailureRecord }
  | { type: "status";     data: { engine: string; version: string } }
```

### Zustand Store Shape
```typescript
interface AppStore {
  // Market
  spotPrice: number
  regime: RegimeLabel
  optionsChain: OptionRow[]

  // Signals
  signals: Signal[]          // last 50

  // Trades
  openTrades: DecisionContext[]
  closedTrades: DecisionContext[]
  selectedTradeId: string | null

  // Learning
  insightReport: InsightReport | null

  // System
  engineStatus: "RUNNING" | "STOPPED" | "PAPER"
  connected: boolean
  strategyVersion: string
  failures: FailureRecord[]

  // Config
  config: AppConfig
}
```

## Code Quality Rules

- All Python modules must have type hints — no bare `dict` for domain objects
- `DecisionContext` must never be mutated after `signal_features` are written
- All file writes go through `Trade_Logger` — no direct file I/O in other modules
- Settings changes via API must validate against schema before applying
- Backend tests: pytest with property-based tests (Hypothesis) for signal engine and trade logger
- Frontend: TypeScript strict mode, no `any`
- Commit format: `feat(module): description` / `fix(module): description`

## File Structure
```
/backend
  /data
    websocket.py          ← Upstox WS client
    candle_builder.py     ← tick → OHLCV
  /strategy
    lorentz.py            ← signal engine + state machine
    regime.py             ← regime classifier
    mtf.py                ← multi-timeframe bias
    dedup.py              ← structure_id + deduplication
    time_filter.py        ← session window filter
  /execution
    trader.py             ← trade lifecycle manager
    options_selector.py   ← contract selection (Mode B/C)
    slippage.py           ← fill simulation
  /memory
    trade_logger.py       ← DecisionContext serialization
    models.py             ← dataclasses (Signal, DecisionContext, etc.)
  /learning
    analyzer.py           ← clustering + insight generation
    versioning.py         ← strategy version management
    experiments.py        ← parameter experiment framework
  /api
    server.py             ← FastAPI app (REST + WebSocket broadcaster)
  /config
    settings.json         ← runtime config
    defaults.py           ← default values + validation schemas
  main.py                 ← entry point, wires all modules

/frontend
  /app
    layout.tsx
    page.tsx
  /components
    Header.tsx
    MarketView.tsx
    SignalPanel.tsx
    TradeTable.tsx
    InsightPanel.tsx
    Controls.tsx
    FailurePanel.tsx
  /lib
    ws.ts
    store.ts
    types.ts
```
