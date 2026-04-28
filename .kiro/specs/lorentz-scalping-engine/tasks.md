# Implementation Plan: Lorentz Scalping Engine

## Agent Assignment Guide

| Agent | Strengths | Assigned work |
|---|---|---|
| **Kiro** | Multi-file coordination, architecture, Python asyncio, strategy logic, spec-driven implementation, property-based tests | All backend tasks, store/WS/types, InsightPanel logic, learning engine |
| **Claude** | React/Tailwind visual polish, TypeScript components, iterative UI refinement | All visual UI components (Header, MarketView, SignalPanel, TradeTable, Controls, shared components, ReplayMode) |
| **Codex** | Repetitive boilerplate, single-file pattern generation, test stubs | Unit test stubs, pyproject.toml scaffolding, `__init__.py` files |

Each task is tagged: `[Kiro]`, `[Claude]`, or `[Codex]`

---

## Overview

Python asyncio backend + Next.js 14 frontend. All tasks build incrementally — each step integrates into the previous. The backend is built first (foundation → pipeline → strategy → execution → memory → API), then the frontend is wired to the real backend, followed by the learning engine and post-market research loop. Tests are co-located with implementation tasks.

Implementation language: **Python** (backend), **TypeScript** (frontend).

---

## Tasks

- [ ] 1. Backend foundation — project structure, models, config `[Kiro]`
  - [x] 1.1 Scaffold backend directory structure and Python project files `[Codex]`
    - Create `backend/` with subdirs: `data/`, `strategy/`, `execution/`, `memory/`, `learning/`, `api/`, `config/`, `tests/unit/`, `tests/property/`, `tests/integration/`
    - Create `backend/pyproject.toml` with dependencies: `fastapi`, `uvicorn`, `websockets`, `hypothesis`, `pytest`, `pytest-asyncio`, `pydantic`
    - Create empty `__init__.py` files in each package directory
    - _Requirements: 1.1, 9.1_

  - [ ] 1.2 Implement all core domain dataclasses in `backend/memory/models.py` `[Kiro]`
    - Define all Literal type aliases: `RegimeLabel`, `TimeBucket`, `SlopeMethod`, `MTFBias`, `ExitReason`, `TradeResult`, `TradeQuality`, `SLMode`, `OverrideType`, `DriftLevel`, `EngineStatus`, `FailureType`
    - Implement frozen dataclasses: `Tick`, `Candle`, `OptionContract`, `SignalFeatures`, `Signal`
    - Implement mutable dataclasses: `ExecutionFeatures`, `DecisionContext`, `OverrideRecord`, `FailureRecord`, `TradeEvent`
    - _Requirements: 6.1, 6.2, 23.1, 23.3_

  - [ ] 1.3 Implement config loading and validation in `backend/config/defaults.py` and `backend/config/settings.json` `[Kiro]`
    - Write `CONFIG_SCHEMA` dict with all validation ranges
    - Write `load_config(path)` function that reads `settings.json`, validates all fields against schema, falls back to defaults on missing/invalid fields, and logs warnings
    - Write `AppConfig` dataclass mirroring the settings JSON structure
    - Create `backend/config/settings.json` with all default values from the design
    - _Requirements: 9.1, 9.4, 9.5_

  - [ ] 1.4 Implement `backend/main.py` entry point skeleton `[Kiro]`
    - Wire asyncio event loop, instantiate all queues (`tick_queue`, `candle_queue`, `signal_queue`, `filtered_queue`, `event_queue`, `broadcast_queue`)
    - Stub `start_engine()` and `stop_engine()` coroutines that will be filled in later tasks
    - _Requirements: 1.1, 8.2_

- [ ] 2. Data pipeline — WebSocket client and candle builder `[Kiro]`
  - [ ] 2.1 Implement `DataEngine` in `backend/data/websocket.py` `[Kiro]`
    - Implement `connect()`, `_on_tick()`, `_normalize_tick()`, `stop()` methods
    - Implement `_reconnect_loop()` with exponential backoff: `delay = min(2 ** attempt * 1, 60)`, reset counter on successful message
    - On auth error: log `FailureRecord(AUTH_ERROR)` and halt ingestion without retry
    - Publish normalized `Tick` objects to `tick_queue` within 100ms of receipt
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5_

  - [ ] 2.2 Implement `CandleBuilder` in `backend/data/candle_builder.py` `[Kiro]`
    - Implement `process_tick()` with running OHLCV state, `_boundary()` using `floor(minute/5)*5` alignment
    - Emit completed candle to `candle_queue` on boundary crossing
    - Implement `_zero_tick_candle()` for windows with no ticks (flat candle at last close, zero volume)
    - Preserve tick arrival order for OHLCV construction
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_

  - [ ]* 2.3 Write property tests for candle builder `[Kiro]`
    - **Property 2: Candle OHLCV Correctness** — for any non-empty tick sequence, assert open/high/low/close/volume invariants
    - **Property 3: Candle Boundary Emission** — for any tick sequence spanning a boundary, assert exactly one candle emitted per elapsed period
    - **Validates: Requirements 2.1, 2.2, 2.3, 2.5**

  - [ ]* 2.4 Write unit tests for candle builder (`tests/unit/test_candle_builder.py`) `[Codex]`
    - Test zero-tick candle emission, boundary alignment edge cases (09:14:59 vs 09:15:00), tick ordering
    - _Requirements: 2.1, 2.4_

- [ ] 3. Strategy engine — regime, MTF, Lorentz signal engine `[Kiro]`
  - [ ] 3.1 Implement `RegimeDetector` in `backend/strategy/regime.py` `[Kiro]`
    - Implement `classify()` with ATR(14) + EMA(39) slope logic: HIGH_VOL if ATR > threshold, TRENDING_UP/DOWN from EMA slope + price position, RANGING otherwise
    - Implement `_compute_atr()`, `_compute_ema()`, `_ema_slope()` helpers
    - Default to `UNKNOWN` on any computation failure
    - _Requirements: 16.1, 16.2, 16.5_

  - [ ]* 3.2 Write property test for regime detector `[Kiro]`
    - **Property 5: Regime Label Exhaustiveness** — for any candle sequence, assert result is always in the valid set, never None
    - **Validates: Requirements 16.2, 16.5**

  - [ ] 3.3 Implement `MTFContext` in `backend/strategy/mtf.py` `[Kiro]`
    - Implement `update()` accumulating 5m candles, rebuilding 15m candle every 3 inputs via `_build_15m_candle()`
    - Implement `get_bias()` returning `BULLISH | BEARISH | NEUTRAL` from 15m EMA slope
    - Default to `NEUTRAL` when fewer than 3 candles accumulated
    - _Requirements: 20.1, 20.2, 20.3_

  - [ ] 3.4 Implement `LorentzSignalEngine` in `backend/strategy/lorentz.py` `[Kiro]`
    - Implement `_compute_pivots()`, `_select_slope_method()` (ATR > 1.5 → ATR_SLOPE, > 0.5 → STDEV_SLOPE, else LINREG_SLOPE), `_compute_trendlines()`
    - Implement `_compute_rsi()`, `_compute_confidence()` returning score in [0.0, 1.0]
    - Implement `_compute_structure_id()` as SHA-256 hash of (pivot_high, pivot_low, boundary_ts) → 8-char hex
    - Implement `_advance_state_machine()`: IDLE → BREAKOUT_DETECTED → CONFIRMATION → SIGNAL_EMITTED → IDLE
    - Suppress signal emission when fewer than `lookback_length` candles available
    - Emit `Signal` to `signal_queue` with all required fields populated
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 26.1, 26.2_

  - [ ]* 3.5 Write property tests for signal engine `[Kiro]`
    - **Property 4: Slope Method Selection Determinism** — for any ATR value, assert deterministic single method returned
    - **Property 6: Signal Emission Conditions** — for any candle satisfying BUY/SELL conditions, assert correct signal type; no signal when neither holds
    - **Property 7: Signal Payload Completeness** — for any emitted signal, assert all required fields non-null
    - **Property 8: Confidence Score Range** — for any valid inputs, assert score in [0.0, 1.0]
    - **Property 13: Breakout Deduplication Idempotence** — for any breakout sequence, assert at most one signal before IDLE reset
    - **Validates: Requirements 3.2, 3.4, 3.5, 3.6, 3.8, 26.1, 26.2**

  - [ ]* 3.6 Write unit tests for signal engine (`tests/unit/test_lorentz_engine.py`) `[Codex]`
    - Test state machine transitions (each edge), insufficient history suppression, structure_id hash stability
    - _Requirements: 3.7, 3.8_

  - [ ] 3.7 Implement `SignalDeduplicator` in `backend/strategy/dedup.py` `[Kiro]`
    - Implement `check()` returning `(is_duplicate, reason)` based on structure_id + direction + candle window
    - Implement `register()` and `_evict_stale()` for window management
    - Respect `dedup_enabled` config flag — pass all signals through when disabled
    - _Requirements: 17.1, 17.2, 17.3, 17.4_

  - [ ] 3.8 Implement `TimeFilter` in `backend/strategy/time_filter.py` `[Kiro]`
    - Implement `check()` returning `(is_allowed, suppression_reason)`
    - Implement `classify_bucket()` with exact IST boundary mapping: 09:15–09:45 OPEN, 09:45–11:30 MORNING, 11:30–13:30 MIDDAY, 13:30–14:45 AFTERNOON, 14:45–15:30 CLOSE
    - Suppress signals outside `allowed_sessions` when filter is enabled
    - _Requirements: 19.1, 19.2, 19.4_

  - [ ]* 3.9 Write property tests for time filter and deduplicator `[Kiro]`
    - **Property 15: Time Bucket Classification Correctness** — for any IST timestamp within trading hours, assert correct bucket
    - **Property 16: Session Filter Suppression** — for any signal outside allowed_sessions, assert suppression with non-empty reason
    - **Validates: Requirements 19.2, 19.4**

  - [ ]* 3.10 Write unit tests for regime, MTF, dedup, time filter (`tests/unit/`) `[Codex]`
    - `test_regime_detector.py`: each regime label, UNKNOWN fallback
    - `test_time_filter.py`: each session bucket boundary, disabled filter pass-through
    - _Requirements: 16.5, 17.4, 19.1_

- [ ] 4. Execution engine — options selector, slippage, trader `[Kiro]`
  - [ ] 4.1 Implement `OptionsSelector` in `backend/execution/options_selector.py` `[Kiro]`
    - Implement `select()` for Mode B: find nearest ATM ± strike with premium in [premium_min, premium_max]
    - Implement `_find_atm_strike()` rounding spot to nearest 100 for BN strike grid
    - BUY signal → CE contract; SELL signal → PE contract
    - Return `None` and log `FailureRecord(MISSED_SIGNAL)` when no valid contract found
    - _Requirements: 4.1, 4.2, 4.3, 4.4_

  - [ ]* 4.2 Write property tests for options selector `[Kiro]`
    - **Property 21: Options Contract Direction Invariant** — for any BUY signal, assert CE selected; for any SELL, assert PE
    - **Property 22: Mode B Premium Range Invariant** — for any valid selection, assert ltp within [premium_min, premium_max]
    - **Validates: Requirements 4.1, 4.2**

  - [ ] 4.3 Implement `SlippageModel` in `backend/execution/slippage.py` `[Kiro]`
    - Implement `apply()`: BUY fill = `ltp * (1 + bps/10000)`, SELL fill = `ltp * (1 - bps/10000)`
    - When `slippage_enabled = False`: fill_price = ltp, slippage_amount = 0.0
    - _Requirements: 18.1, 18.2, 18.3, 18.4_

  - [ ]* 4.4 Write property test for slippage model `[Kiro]`
    - **Property 14: Slippage Fill Price Correctness** — for any ltp and bps, assert fill price formula; assert disabled mode returns exact LTP
    - **Validates: Requirements 18.1, 18.2**

  - [ ] 4.5 Implement `ExecutionEngine` in `backend/execution/trader.py` `[Kiro]`
    - Implement `on_signal()`: open paper trade if concurrent limit not reached, call OptionsSelector + SlippageModel, compute SL/TP, emit `TRADE_OPEN` event
    - Implement `on_tick()`: check SL/TP for all open trades on every premium tick, emit `TRADE_CLOSE` on hit
    - Implement `on_candle()`: increment `candles_held`, trigger time exit at `time_exit_candles` threshold
    - Implement `manual_exit()`: close trade immediately, log override record
    - Implement `_compute_sl()`: CANDLE_BASED (signal candle low/high) and ATR_BASED (entry ± atr)
    - Implement `_compute_tp()`: `entry + 2*(entry-sl)` for BUY, `entry - 2*(sl-entry)` for SELL (1:2 RR)
    - Implement `_compute_trade_quality()`: A/B/C/D grading rubric
    - Implement `_check_daily_loss_limit()`: halt and broadcast warning when breached
    - Track MAE and MFE on every tick for open trades
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, 5.8, 5.9, 27.1, 31.4_

  - [ ]* 4.6 Write property tests for execution engine `[Kiro]`
    - **Property 9: Exit Trigger Invariant** — for any tick where premium >= tp or <= sl, assert correct exit reason; assert mutual exclusivity
    - **Property 11: Concurrent Trade Limit Invariant** — for any signal sequence, assert open trade count never exceeds max_concurrent_trades
    - **Property 12: Stop-Loss and Take-Profit Computation Correctness** — for any signal/entry/atr, assert SL/TP formulas and 1:2 RR
    - **Validates: Requirements 5.2, 5.4, 5.5, 5.6**

  - [ ]* 4.7 Write unit tests for execution engine (`tests/unit/test_execution_engine.py`) `[Codex]`
    - Test daily loss halt, manual exit override, time exit at candle threshold, suppression when limit reached
    - _Requirements: 5.3, 5.7, 5.8_

- [ ] 5. Checkpoint — wire data pipeline through execution engine `[Kiro]`
  - Connect all asyncio queues: `tick_queue → candle_queue → signal_queue → filtered_queue → event_queue`
  - Update `main.py` to start all coroutines as asyncio tasks
  - Verify end-to-end flow with a mock tick sequence (no real Upstox connection needed)
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 6. Memory layer — trade logger and serialization `[Kiro]`
  - [ ] 6.1 Implement `TradeLogger` in `backend/memory/trade_logger.py` `[Kiro]`
    - Implement `log_trade_open()`, `log_trade_close()`, `log_suppression()`, `log_override()`, `log_failure()` — all append to their respective JSONL files
    - Implement `_serialize()` with datetime → ISO 8601 string conversion
    - Implement `deserialize_context()` static method; on malformed input log and skip without halting
    - Implement `_write_with_retry()` with up to 3 retries and exponential backoff
    - Assign unique `trade_id` (uuid4) to each `DecisionContext`
    - Ensure `signal_features` are never mutated after initial write
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6, 15.1, 15.2, 15.4, 23.2_

  - [ ]* 6.2 Write property tests for trade logger `[Kiro]`
    - **Property 1: DecisionContext Serialization Round-Trip** — for any valid DecisionContext, serialize then deserialize and assert equality (max_examples=200)
    - **Property 10: DecisionContext Field Completeness** — for any opened trade, assert all identity/signal/execution/entry fields non-null; for closed trade, assert exit fields non-null
    - **Property 17: Signal Features Immutability** — for any DecisionContext, assert signal_features byte-identical before and after log_trade_close
    - **Validates: Requirements 6.1, 6.2, 6.4, 15.1, 15.2, 15.3, 23.1, 23.2, 27.1**

  - [ ]* 6.3 Write unit tests for trade logger (`tests/unit/test_trade_logger.py`) `[Codex]`
    - Test write retry on failure (mock file I/O), malformed record skip, suppression record format
    - _Requirements: 6.6, 15.4_

- [ ] 7. API server — FastAPI REST + WebSocket broadcaster `[Kiro]`
  - [ ] 7.1 Implement `FastAPI` app skeleton in `backend/api/server.py` `[Kiro]`
    - Define all REST endpoints: `GET /health`, `GET /config`, `POST /config` (validate + hot-reload), `POST /engine/start`, `POST /engine/stop`, `POST /learning/trigger`
    - Define trade endpoints: `GET /trades`, `GET /trades/{trade_id}`, `POST /trades/{trade_id}/exit`
    - Define insight/version endpoints: `GET /insights/latest`, `GET /insights/weekly/latest`, `GET /failures`, `GET /versions`, `POST /versions/{id}/rollback`
    - Config `POST` must validate against `CONFIG_SCHEMA` before applying; reject with descriptive error on invalid values
    - _Requirements: 9.3, 14.1, 14.2, 14.3, 14.4, 21.4_

  - [ ] 7.2 Implement WebSocket broadcaster in `backend/api/server.py` `[Kiro]`
    - Expose `WS /ws` endpoint broadcasting `WSMessage` stream from `broadcast_queue`
    - Implement all message types: `tick`, `candle`, `signal`, `suppression`, `trade_open`, `trade_close`, `regime`, `insight`, `failure`, `status`, `drift`, `latency`, `risk`
    - Broadcast `drift` message when drift level changes; broadcast `latency` on each signal emit; broadcast `risk` on each tick
    - _Requirements: 10.3, 29.3, 31.1_

  - [ ]* 7.3 Write integration tests for API (`tests/integration/test_api.py`) `[Codex]`
    - Test all REST endpoint contracts (request/response shapes), config validation rejection, WebSocket message dispatch
    - _Requirements: 9.3, 9.5_

- [ ] 8. Checkpoint — full backend integration `[Kiro]`
  - Wire `TradeLogger` and `WebSocket_Broadcaster` to `event_queue` in `main.py`
  - Verify `POST /config` hot-reload propagates to all modules without restart
  - Run `pytest tests/unit/ tests/property/` — ensure all pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 9. Frontend completion — remaining components wired to real backend
  - [ ] 9.1 Complete `frontend/src/lib/types.ts` with all TypeScript types mirroring Python schemas `[Kiro]`
    - Define all Literal types: `RegimeLabel`, `TimeBucket`, `SlopeMethod`, `MTFBias`, `ExitReason`, `TradeQuality`, `EngineStatus`, `DriftLevel`
    - Define interfaces: `SignalFeatures`, `ExecutionFeatures`, `DecisionContext`, `AppConfig`, `InsightReport`, `WeeklyReport`, `FailureRecord`, `OptionRow`
    - Define full `WSMessage` discriminated union covering all 13 message types
    - No `any` types — strict TypeScript throughout
    - _Requirements: 10.1, 11.1, 12.1, 12.2_

  - [ ] 9.2 Complete `frontend/src/lib/store.ts` Zustand store `[Kiro]`
    - Implement full `AppStore` interface with all state fields from design section 4.2
    - Implement all actions: `setSpotPrice`, `addSignal`, `addSuppression`, `openTrade`, `closeTrade`, `selectTrade`, `setInsightReport`, `setConfig`, `setPendingConfig`, `setCognitiveMode`, `focusInsightPanel`, `startReplay`, `stopReplay`
    - Implement cognitive mode auto-switching middleware: CRITICAL if HIGH_VOL, FOCUSED if open trades > 0, else NORMAL
    - Maintain `signalGroups` map keyed by `structure_id` for grouped display
    - Cap `signals` array at 50 entries
    - _Requirements: 11.3, 11.4, 25.1, 25.2, 25.3_

  - [ ] 9.3 Complete `frontend/src/lib/ws.ts` WebSocket client `[Kiro]`
    - Implement `WSClient` class with `connect()`, `disconnect()`, `onMessage()` dispatching to store actions, `onClose()` triggering reconnect
    - Implement exponential backoff reconnect capped at 30s
    - Show amber reconnecting banner via store action on disconnect
    - _Requirements: 10.3, 32.1_

  - [ ] 9.4 Complete `frontend/src/components/Header.tsx` `[Claude]`
    - Display BN spot price (monospace, color-coded green/red vs prev close), regime badge (pill, color per regime), strategy version badge, engine status badge
    - Add drift badge (LOW/MED/HIGH with color), latency indicator (ms + color dot), risk meter (progress bar with color thresholds)
    - Add cognitive load mode pill (NORMAL/FOCUSED/CRITICAL) with manual lock toggle
    - Add failure count badge (red, opens FailurePanel on click)
    - _Requirements: 10.1, 25.5, 25.6, 29.3, 31.1, 31.3_

  - [ ] 9.5 Complete `frontend/src/components/MarketView.tsx` `[Claude]`
    - Render options chain table: Strike | CE LTP | CE OI | PE LTP | PE OI
    - Highlight ATM row with subtle border; color OI changes green/red
    - Add regime background tint overlay and regime history strip (last 10 as colored blocks)
    - Flash animation on spot price tick update (200ms)
    - _Requirements: 10.1, 10.2_

  - [ ] 9.6 Complete `frontend/src/components/SignalPanel.tsx` `[Claude]`
    - Render signals grouped by `structure_id` with group header showing short hash
    - Each signal row: time, BUY/SELL badge, price, RSI, slope method, regime, confidence bar
    - Suppressed signals: amber left border, SUPPRESSED badge, expandable reason list
    - Signal Panel header: count badge "Signals (N) | Suppressed (N)"
    - Slide-in animation for new signals; auto-scroll to latest
    - Dim rows with confidence < 0.5
    - _Requirements: 11.1, 11.2, 11.3, 11.4, 11.5, 26.3, 26.4_

  - [ ] 9.7 Implement `frontend/src/components/TradeTable.tsx` `[Claude]`
    - Two tabs: OPEN (n) | CLOSED (n)
    - Open trades: trade ID, signal type, entry time, entry price, strike, current premium, unrealised P&L (animated on update), SL, TP, regime badge, time_bucket badge, version badge; inline Exit Now + Hold TP buttons
    - Closed trades: trade ID, signal type, entry/exit time, entry/exit price, exit reason, realised P&L, trade_quality grade, MAE, MFE
    - Click any row → `selectTrade()` → Insight Panel auto-focuses
    - Fade transition when trade moves from OPEN to CLOSED tab
    - _Requirements: 12.1, 12.2, 12.3, 12.4, 27.3_

  - [ ] 9.8 Implement `frontend/src/components/InsightPanel.tsx` with sub-components `[Kiro + Claude]`
    - Logic wiring (store subscriptions, tab state, auto-focus, Simulate/Apply/Ignore actions): `[Kiro]`
    - Visual layout, card styling, timeline rendering: `[Claude]`
    - Always sticky, full panel height, never collapsible
    - Three tabs: Trade Reasoning | Learning Report | Override Log
    - **Trade Reasoning tab**: render `TradeReasoningCard` (signal features + execution features + outcome), `MicrostructureTimeline` (BREAKOUT → CONFIRM → ENTRY → EXIT with latency labels), Replay button
    - **Learning Report tab**: render `LearningReportCard` per insight with Simulate Impact / Apply / Ignore actions; Apply pre-fills Controls with pending config change
    - **Override Log tab**: render `OverrideLogTab` showing override history with outcome column
    - Auto-focus when any signal or trade is clicked; keyboard shortcut `I`
    - _Requirements: 13.1, 13.2, 13.3, 13.4, 13.5, 13.6, 13.7, 13.8, 30.2_

  - [ ] 9.9 Implement `frontend/src/components/Controls.tsx` `[Claude]`
    - Core config section (always visible): RSI threshold, max trades, capital, SL mode — pre-populated from store
    - Advanced config section (collapsed by default, toggle): all remaining params
    - Start/Stop engine button (large, prominent); Trigger Learning button with spinner
    - Show "Unsaved changes" indicator when `pendingConfigChanges` is non-null; Apply button submits to `POST /config`
    - Confirmation modal before applying config while trades are open or stopping engine with open trades
    - _Requirements: 14.1, 14.2, 14.3, 14.4, 14.5, 9.6_

  - [ ] 9.10 Implement `frontend/src/components/FailurePanel.tsx` `[Claude]`
    - Expandable panel (triggered from header badge) showing last N failures (N from config)
    - Each row: timestamp, failure type badge, module name, message, expandable context JSON
    - _Requirements: 24.3_

  - [ ] 9.11 Implement shared UI components in `frontend/src/components/shared/` `[Claude]`
    - `ConfidenceBar.tsx`: 0.0–1.0 visual bar, teal (BUY) / red (SELL), fades below 0.4
    - `RegimeBadge.tsx`: pill with regime color mapping
    - `TradeQualityBadge.tsx`: A/B/C/D letter grade with color (A=green, D=red)
    - `DriftBadge.tsx`: LOW/MED/HIGH with color
    - `RiskMeter.tsx`: progress bar with green/amber/red thresholds at 40%/70%
    - `LatencyIndicator.tsx`: ms value + color dot (green <50ms, amber <150ms, red >150ms)
    - `CognitiveLoadOverlay.tsx`: NORMAL/FOCUSED/CRITICAL mode switcher with CRITICAL deep-red tint
    - `ConfirmationModal.tsx`: safety confirmation dialog
    - `Toast.tsx`: non-blocking smart alert toast, auto-dismiss 5s
    - _Requirements: 25.1, 25.4, 25.5, 26.3, 27.3, 29.3, 31.3_

  - [ ] 9.12 Implement `frontend/src/components/insight/ReplayMode.tsx` `[Claude]`
    - Candle-by-candle replay from N candles before entry through exit
    - Show evolving trendlines, RSI, regime label, breakout state machine transitions at each step
    - Playback speeds: 1×, 5×, 10×; keyboard: right arrow (next), left arrow (prev), space (play/pause)
    - _Requirements: 28.1, 28.2, 28.3, 28.4, 28.5_

  - [ ] 9.13 Wire `frontend/src/app/page.tsx` and `layout.tsx` to real backend `[Kiro]`
    - Replace all mock-data imports with live Zustand store subscriptions
    - Initialize `WSClient` in layout connecting to `ws://localhost:8765`
    - Implement keyboard shortcut handler: Space (start/stop), L (learning), E (exit trade), I (insight), C (controls), Esc (deselect), 1–5 (panel switch)
    - Implement fail-safe UI: full-width red blocking banner on data gap > 30s with Resume / Exit All actions
    - _Requirements: 10.3, 32.1, 32.2, 32.3, 32.4_

- [ ] 10. Checkpoint — frontend connected to backend `[Kiro]`
  - Start backend (`uvicorn backend.api.server:app`) and frontend (`next dev`) manually
  - Verify WebSocket messages flow from backend to all UI panels
  - Verify config changes via Controls panel hot-reload in backend without restart
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 11. Learning engine — analyzer, versioning, experiments `[Kiro]`
  - [ ] 11.1 Implement learning data models in `backend/learning/models.py` `[Kiro]`
    - Define dataclasses: `BaselineMetrics`, `Hypothesis`, `HypothesisResult`, `RankedInsight`, `SessionSummary`, `InsightReport`, `WeeklyReport`
    - _Requirements: 7.3, 33.4, 33.5_

  - [ ] 11.2 Implement `LearningEngine` in `backend/learning/analyzer.py` `[Kiro]`
    - Implement `run_daily_cycle()` executing LOAD → BASELINE → HYPOTHESIZE → TEST → RANK → REPORT → SUGGEST
    - Implement `_load_trades()`, `_compute_baseline()` (rolling 20-trade baseline), `_hypothesize()` across all 7 dimensions
    - Implement `_test_hypothesis()` with backtest against full trade log; mark PENDING if bucket < 10 trades
    - Implement `_rank_hypotheses()` using score formula: `expected_improvement * confidence * (1 / complexity)`
    - Implement `_write_report()` to `memory/insights/YYYY-MM-DD.json` and `latest.json`
    - Implement `_write_settings_diff()` to `memory/suggestions/YYYY-MM-DD-settings-diff.json`
    - Implement `_assign_experiment_id()` format `EXP-YYYY-MM-DD-NNN`
    - Implement `_compute_drift()` comparing recent 10-trade win rate vs 20-trade baseline
    - Write "insufficient data" report and skip hypothesis testing when session trades < 5
    - Never suggest disabling entire strategy; never suggest changes from < 10 trade buckets
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.6, 7.7, 7.8, 7.9, 33.1, 33.2, 33.3, 33.4, 33.5, 33.6, 33.7, 33.8, 33.9, 33.10, 33.11, 33.12, 33.13, 33.14, 33.15_

  - [ ] 11.3 Implement `run_weekly_cycle()` in `backend/learning/analyzer.py` `[Kiro]`
    - Aggregate 5 daily experiment results, compute actual vs projected improvement per applied change
    - Update rolling baseline with full week's data
    - Suggest one "bold experiment" for following week
    - Write to `memory/insights/weekly/YYYY-WNN.json`
    - _Requirements: 34.1, 34.2, 34.3, 34.4, 34.5, 34.6_

  - [ ]* 11.4 Write property tests for learning engine
    - **Property 19: Hypothesis Minimum Evidence Gate** — for any hypothesis bucket with count < 10, assert status is PENDING and absent from ranked list
    - **Property 20: Drift Classification Correctness** — for any win rate sequence, assert drift level matches thresholds deterministically
    - **Validates: Requirements 29.1, 29.2, 33.8, 33.15**

  - [ ]* 11.5 Write unit tests for learning engine (`tests/unit/test_learning_engine.py`)
    - Test insufficient data report (< 5 trades), pending hypothesis exclusion, experiment ID format
    - _Requirements: 7.7, 33.13_

  - [ ] 11.6 Implement `StrategyVersioning` in `backend/learning/versioning.py`
    - Implement `snapshot()` hashing current config → version_id string
    - Implement `get_history()` and `rollback()` reading/writing `config/versions/v{hash}.json`
    - _Requirements: 21.1, 21.2, 21.3, 21.4_

  - [ ] 11.7 Implement `ExperimentFramework` in `backend/learning/experiments.py`
    - Implement `create_experiment()`, `record_result()`, `rank_experiments()` (by win_rate * avg_pnl)
    - Write experiment results to `memory/experiments/EXP-*.json`
    - _Requirements: 22.1, 22.2, 22.3, 22.4, 22.5_

- [ ] 12. Post-market research loop wiring
  - [ ] 12.1 Schedule daily learning cycle trigger in `backend/main.py`
    - Schedule `run_daily_cycle()` at `learning_trigger_time` (default 15:35 IST) using asyncio scheduler
    - Trigger immediately when drift reaches HIGH (watch `drift_queue` or store state)
    - Expose manual trigger via `POST /learning/trigger` API endpoint (already defined in task 7.1)
    - _Requirements: 7.4, 7.5, 33.2, 33.3_

  - [ ] 12.2 Schedule weekly cycle on Fridays
    - After daily cycle on Fridays, run `run_weekly_cycle()` automatically
    - _Requirements: 34.1_

  - [ ] 12.3 Wire drift detection to UI broadcaster
    - After each learning cycle and after every 10 new closed trades, recompute drift and broadcast `drift` WSMessage
    - Trigger smart alert toast when drift reaches HIGH
    - _Requirements: 29.1, 29.2, 29.3, 29.4_

- [ ] 13. Tests — property-based, unit, and integration
  - [ ] 13.1 Implement custom Hypothesis strategies in `tests/property/strategies.py`
    - Implement: `tick_strategy()`, `candle_strategy()`, `signal_strategy()`, `decision_context_strategy()`, `options_chain_strategy()`, `trade_result_sequence()`
    - These strategies are used by all property tests in tasks 2.3, 3.5, 4.2, 4.4, 4.6, 6.2, 11.4
    - _Requirements: 15.3_

  - [ ] 13.2 Consolidate all property tests into `tests/property/test_properties.py`
    - Collect all 23 property tests (Properties 1–23) into a single file with `# Feature: lorentz-scalping-engine, Property N: Title` tags
    - Set `max_examples=200` for Properties 1, 2, 8, 11; `max_examples=100` for all others
    - Configure `pyproject.toml` with `asyncio_mode = "auto"` and `testpaths = ["tests"]`
    - _Requirements: 1.2, 2.1, 2.2, 3.2, 3.4, 3.5, 3.6, 3.8, 4.1, 4.2, 5.2, 5.4, 5.5, 5.6, 6.1, 6.2, 6.4, 15.1, 15.2, 15.3, 16.2, 16.5, 17.1, 17.2, 18.1, 18.2, 19.2, 19.4, 23.1, 23.2, 26.1, 26.2, 27.1, 29.1, 29.2, 33.8, 33.15_

  - [ ]* 13.3 Write remaining unit tests
    - `tests/unit/test_options_selector.py`: no valid contract path, Mode B selection, ATM rounding
    - `tests/unit/test_slippage.py`: disabled slippage = LTP, BPS calculation for BUY and SELL
    - `tests/unit/test_config.py`: missing file defaults, invalid range rejection with descriptive error
    - _Requirements: 4.3, 9.4, 9.5, 18.4_

  - [ ]* 13.4 Write integration tests (`tests/integration/`)
    - `test_pipeline.py`: end-to-end tick → candle → signal → trade → JSONL log using mock Upstox WS server
    - `test_ws_protocol.py`: all 13 WSMessage types dispatched correctly to store actions
    - _Requirements: 1.3, 6.4_

  - [ ]* 13.5 Write frontend tests (`frontend/__tests__/`)
    - `store.test.ts`: Zustand store actions, cognitive mode auto-switching, signal grouping by structure_id
    - `ws.test.ts`: WebSocket reconnect backoff, message dispatch to store
    - `utils.test.ts`: `formatPrice`, `classifyBucket`, `formatPnl`
    - `components/ConfidenceBar.test.tsx`: correct width for 0.0, 0.5, 1.0; dimming below 0.4
    - `components/TradeTable.test.tsx`: open/closed tab switching, MAE/MFE display, row click → insight focus
    - `components/InsightPanel.test.tsx`: tab switching, always-visible assertion, auto-focus on trade select
    - _Requirements: 11.3, 12.3, 13.4, 25.2_

- [ ] 14. Final checkpoint — full system validation
  - Run `pytest tests/unit/ tests/property/ tests/integration/` — all must pass
  - Run `npx vitest --run` in `frontend/` — all must pass
  - Verify reconnect backoff property (Property 23) with a mock disconnecting WS server
  - Ensure all tests pass, ask the user if questions arise.

---

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Property tests (Hypothesis) validate universal invariants; unit tests validate specific examples and edge cases
- All 23 correctness properties from the design document are covered across tasks 2.3, 3.2, 3.5, 3.9, 4.2, 4.4, 4.6, 6.2, 11.4, and 13.2
- Backend module boundary rule: no module may import from a module downstream of it in the event flow
- `DecisionContext.signal_features` must never be mutated after initial write — enforced by frozen dataclass
- All file I/O goes through `TradeLogger` only — no direct file writes in other modules
- Frontend: TypeScript strict mode, no `any` types
