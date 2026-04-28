# Requirements Document

## Introduction

The Lorentz Scalping Engine is a local-first, self-improving algorithmic trading system for Bank Nifty options. It executes a Lorentz-style breakout strategy (converted from Pine Script to Python), trades ATM ± dynamic strike options, logs full decision context for every trade, and evolves through AI-assisted structured learning. The system is fully configurable and overrideable via a Next.js UI.

The key differentiator is comprehensive decision-context logging — capturing not just entry/exit, but the full reasoning behind every trade — enabling AI-assisted learning and continuous improvement.

---

## Glossary

- **Lorentz_Signal_Engine**: The Python module that implements the Lorentz breakout strategy, including adaptive trendlines, pivot detection, breakout confirmation, and RSI filtering.
- **Data_Engine**: The module responsible for connecting to Upstox WebSocket and streaming Bank Nifty spot and options tick data.
- **Candle_Builder**: The module that aggregates raw ticks into 5-minute OHLCV candles.
- **Options_Selector**: The module that selects the appropriate options contract (ATM ± dynamic strike) based on the configured mode.
- **Execution_Engine**: The module that manages the full trade lifecycle: signal intake, option selection, entry, tracking, and exit.
- **Trade_Logger**: The module that persists full decision context for every trade event.
- **Learning_Engine**: The module that performs template-driven clustering and analysis of historical trade logs to surface winning and losing setups.
- **Strategy_Engine**: The extensible registry that manages available strategies; S1 = Lorentz.
- **UI**: The Next.js frontend providing panels for market data, signals, trades, decision reasoning, and system controls.
- **BN**: Bank Nifty index.
- **ATM**: At-the-money options strike.
- **ATR**: Average True Range — a volatility measure.
- **Stdev**: Standard deviation of price over a lookback window.
- **LinReg**: Linear regression slope of price over a lookback window.
- **Pivot_High / Pivot_Low**: Swing high/low detected over a configurable lookback length.
- **RSI**: Relative Strength Index.
- **RR**: Risk-to-reward ratio.
- **Paper_Trade**: Simulated trade execution with no real capital at risk.
- **Decision_Context**: The full set of features, indicator values, signal rationale, and outcome associated with a trade.
- **Mode_B**: Options selection mode based on premium range (₹150–₹300).
- **Mode_C**: Options selection mode based on delta (phase 2).
- **Tick**: A single raw price update from the data feed.
- **OHLCV**: Open, High, Low, Close, Volume candle data.
- **Regime**: A classification of current market conditions — one of TRENDING_UP, TRENDING_DOWN, RANGING, HIGH_VOL, or UNKNOWN.
- **Structure_ID**: A unique identifier assigned to a breakout sequence, used to detect and suppress duplicate signals from the same structural move.
- **Time_Bucket**: An intraday time window label (e.g., OPEN, MIDDAY, CLOSE) assigned to each trade.
- **Strategy_Version**: A unique identifier for a configuration snapshot of the active strategy.
- **Slippage**: The simulated difference between the last traded price and the actual simulated fill price.
- **Signal_Features**: The set of indicator and market features frozen at signal candle close time.
- **Execution_Features**: The set of features captured at order fill time (e.g., slippage, actual entry price).
- **Breakout_State_Machine**: The explicit state machine within the Lorentz_Signal_Engine tracking breakout progression: IDLE → BREAKOUT_DETECTED → CONFIRMATION → SIGNAL_EMITTED.
- **Research_Loop**: The autonomous post-market analysis cycle that loads trade data, forms and tests hypotheses, ranks insights, and outputs structured reports and settings suggestions.
- **Experiment_ID**: A unique identifier for each research cycle in the format EXP-YYYY-MM-DD-NNN, creating a traceable research lineage.
- **Settings_Diff**: A ready-to-apply JSON patch to settings.json produced by the Research_Loop containing only the parameter changes supported by confirmed hypotheses.
- **MAE**: Max Adverse Excursion — the worst unrealised drawdown experienced during a trade before close.
- **MFE**: Max Favorable Excursion — the best unrealised profit experienced during a trade before close.
- **Trade_Quality**: A letter grade (A/B/C/D) assigned to each closed trade based on regime alignment, RSI alignment, exit reason, and candles held.
- **Confidence_Score**: A 0.0–1.0 score computed per signal from breakout strength, RSI distance, slope steepness, and regime alignment.
- **Drift**: A measure of strategy performance degradation computed by comparing recent win rate against the rolling baseline.

---

## Requirements

### Requirement 1: Real-Time Market Data Ingestion

**User Story:** As a trader, I want the system to stream live Bank Nifty spot and options data, so that the strategy engine always operates on current market prices.

#### Acceptance Criteria

1. THE Data_Engine SHALL establish a WebSocket connection to the Upstox feed on system startup.
2. WHEN the WebSocket connection is lost, THE Data_Engine SHALL attempt reconnection with exponential backoff until the connection is restored.
3. WHEN a tick is received, THE Data_Engine SHALL publish it to the Candle_Builder within 100ms.
4. THE Data_Engine SHALL stream both BN spot ticks and options chain ticks concurrently.
5. IF the Upstox WebSocket returns an authentication error, THEN THE Data_Engine SHALL log the error with full context and halt data ingestion until credentials are corrected.

---

### Requirement 2: Candle Construction

**User Story:** As a trader, I want raw ticks aggregated into 5-minute OHLCV candles, so that the strategy engine has structured price data to operate on.

#### Acceptance Criteria

1. THE Candle_Builder SHALL aggregate ticks into 5-minute OHLCV candles aligned to clock boundaries (e.g., 09:15, 09:20).
2. WHEN a 5-minute boundary is crossed, THE Candle_Builder SHALL emit a completed candle to the Lorentz_Signal_Engine.
3. WHILE a candle period is open, THE Candle_Builder SHALL maintain a running OHLCV state updated on every tick.
4. IF no ticks are received during a 5-minute window, THEN THE Candle_Builder SHALL emit a candle using the last known close price for all OHLCV fields and zero volume.
5. THE Candle_Builder SHALL preserve tick arrival order when constructing candle OHLCV values.

---

### Requirement 3: Lorentz Breakout Signal Generation

**User Story:** As a trader, I want the system to detect Lorentz-style breakout signals on Bank Nifty, so that entries are taken only on structurally confirmed breakouts.

#### Acceptance Criteria

1. THE Lorentz_Signal_Engine SHALL compute Pivot_High and Pivot_Low over a configurable lookback length (default: 14 candles).
2. THE Lorentz_Signal_Engine SHALL compute volatility using ATR(14) and select the slope calculation method: ATR when ATR > 1.5, Stdev when ATR > 0.5, and LinReg otherwise.
3. THE Lorentz_Signal_Engine SHALL maintain adaptive upper and lower trendlines updated on every completed candle using the selected slope method and a configurable slope multiplier (default: 1.6).
4. WHEN a candle closes above the adaptive upper trendline and RSI(14) is above the configurable RSI threshold (default: 50), THE Lorentz_Signal_Engine SHALL emit a BUY signal.
5. WHEN a candle closes below the adaptive lower trendline and RSI(14) is below the configurable RSI threshold (default: 50), THE Lorentz_Signal_Engine SHALL emit a SELL signal.
6. WHEN a signal is emitted, THE Lorentz_Signal_Engine SHALL include in the signal payload: the signal type, candle timestamp, close price, ATR value, slope method used, upper trendline value, lower trendline value, RSI value, and pivot reference levels.
7. IF fewer than `length` completed candles are available, THEN THE Lorentz_Signal_Engine SHALL suppress signal emission until sufficient history is accumulated.
8. THE Lorentz_Signal_Engine SHALL maintain an explicit Breakout_State_Machine with states: IDLE → BREAKOUT_DETECTED → CONFIRMATION → SIGNAL_EMITTED. A new signal SHALL only be emitted from the SIGNAL_EMITTED transition, preventing duplicate signals from the same breakout event.

---

### Requirement 4: Options Contract Selection

**User Story:** As a trader, I want the system to automatically select the appropriate Bank Nifty options contract for each signal, so that entries are placed on liquid, appropriately priced instruments.

#### Acceptance Criteria

1. WHEN a BUY signal is received, THE Options_Selector SHALL select a Call option; WHEN a SELL signal is received, THE Options_Selector SHALL select a Put option.
2. WHERE Mode_B is configured, THE Options_Selector SHALL select the nearest ATM ± dynamic strike whose last traded premium falls within ₹150–₹300.
3. IF no contract within the premium range exists at the time of selection, THEN THE Options_Selector SHALL log the failure with full context and suppress trade entry for that signal.
4. THE Options_Selector SHALL re-evaluate contract selection on each new signal rather than caching a prior selection across signals.
5. WHERE Mode_C is configured (phase 2), THE Options_Selector SHALL select the contract whose delta is closest to a configurable target delta value.

---

### Requirement 5: Trade Execution Lifecycle

**User Story:** As a trader, I want the system to manage the full paper trade lifecycle from signal to exit, so that trades are entered, tracked, and closed according to the strategy rules.

#### Acceptance Criteria

1. WHEN a valid signal is received and a contract is selected, THE Execution_Engine SHALL open a paper trade recording the entry price, strike, expiry, signal type, and timestamp.
2. THE Execution_Engine SHALL support a maximum of 2 concurrent open trades per strategy (configurable), with at most 1 BUY and 1 SELL trade open simultaneously.
3. WHEN a new signal arrives and the concurrent trade limit for its direction is already reached, THE Execution_Engine SHALL log the suppression reason and discard the signal.
4. THE Execution_Engine SHALL support two stop-loss modes: CANDLE_BASED_SL (default — SL set at the low of the signal candle for BUY, high for SELL, matching the original Pine Script logic) and ATR_BASED_SL (SL set at 1× ATR from entry). The active mode SHALL be configurable.
5. WHEN the options premium reaches or crosses the take-profit level, THE Execution_Engine SHALL close the trade and record the outcome.
6. WHEN the options premium reaches or crosses the stop-loss level, THE Execution_Engine SHALL close the trade and record the outcome.
7. WHEN 5 candles have elapsed since entry and neither stop-loss nor take-profit has been hit, THE Execution_Engine SHALL close the trade at the current premium (time-based exit), configurable up to 6 candles.
8. THE Execution_Engine SHALL operate in paper trade mode by default, with no real order submission.
9. THE Execution_Engine SHALL use a configurable starting capital (default: ₹1,00,000) and configurable position sizing for all P&L calculations.

---

### Requirement 6: Full Decision Context Logging

**User Story:** As a trader and system developer, I want every trade decision logged with its full context, so that the Learning Engine can analyze patterns and the system can explain its reasoning.

#### Acceptance Criteria

1. WHEN a trade is opened, THE Trade_Logger SHALL persist a Decision_Context record containing: signal type, entry timestamp, entry price, strike, expiry, ATR, slope method, upper trendline, lower trendline, RSI value, pivot levels, selected premium, position size, stop-loss level, take-profit level, structure_id, regime, time_bucket, strategy_version, slippage, signal_features, and execution_features.
2. WHEN a trade is closed, THE Trade_Logger SHALL update the Decision_Context record with: exit timestamp, exit price, exit reason (stop-loss / take-profit / time-exit), P&L, and candles held.
3. WHEN a signal is suppressed (concurrent limit, no valid contract, outside trading hours), THE Trade_Logger SHALL persist a suppression record with the signal payload and suppression reason.
4. THE Trade_Logger SHALL write all records to a structured, append-only log file in JSON Lines format.
5. THE Trade_Logger SHALL assign a unique trade ID to each Decision_Context record.
6. IF a write to the log file fails, THEN THE Trade_Logger SHALL retry the write up to 3 times before emitting an error to the system log.

---

### Requirement 7: AI-Assisted Learning Engine

**User Story:** As a trader, I want the system to analyze historical trade logs and surface patterns in winning and losing setups, so that I can improve the strategy configuration over time.

#### Acceptance Criteria

1. THE Learning_Engine SHALL load Decision_Context records from the Trade_Logger's log file for analysis.
2. WHEN a learning cycle is triggered, THE Learning_Engine SHALL cluster trades into winning and losing groups using configurable template-driven criteria (e.g., slope method, RSI range, ATR range, time of day).
3. WHEN a learning cycle completes, THE Learning_Engine SHALL produce a structured insight report identifying the top distinguishing features of winning versus losing setups.
4. THE Learning_Engine SHALL trigger a learning cycle at end-of-day by default, with the trigger time and mechanism configurable.
5. WHERE a manual learning trigger is invoked via the UI, THE Learning_Engine SHALL execute a learning cycle immediately.
6. THE Learning_Engine SHALL persist insight reports to a structured file for retrieval by the UI.
7. IF fewer than 10 completed trades exist in the log, THEN THE Learning_Engine SHALL skip clustering and emit a report indicating insufficient data.
8. THE Learning_Engine SHALL generate actionable parameter suggestions (not just observations), such as "Increase RSI threshold from 50 → 55" or "Disable signal entry in RANGING regime", based on the insight report findings.
9. THE Learning_Engine SHALL analyze and report performance broken down by time_bucket, regime, and strategy_version.

---

### Requirement 8: Extensible Strategy Engine

**User Story:** As a developer, I want the strategy engine to support multiple strategies via a registry, so that new strategies can be added without modifying core execution logic.

#### Acceptance Criteria

1. THE Strategy_Engine SHALL maintain a registry of available strategies identified by a unique strategy ID (e.g., "S1").
2. THE Strategy_Engine SHALL load S1 (Lorentz) as the default active strategy on startup.
3. WHEN a new strategy is registered, THE Strategy_Engine SHALL make it available for selection via the UI and configuration without requiring changes to the Execution_Engine.
4. THE Strategy_Engine SHALL pass the active strategy's signal output to the Execution_Engine using a defined signal interface shared by all strategies.

---

### Requirement 9: System Configuration

**User Story:** As a trader, I want all system parameters to be configurable via a settings file and overrideable via the UI, so that I can tune the system without modifying code.

#### Acceptance Criteria

1. THE System SHALL load all configurable parameters from a `settings.json` file at startup.
2. THE System SHALL expose the following configurable parameters: Lorentz lookback length, slope multiplier, RSI length, RSI threshold, max concurrent trades per strategy, capital amount, position sizing method, time-based exit candle count (5–6), learning trigger time, and options selection mode.
3. WHEN a parameter is overridden via the UI, THE System SHALL apply the override immediately to subsequent signals and trades without requiring a restart.
4. IF `settings.json` is missing or malformed, THEN THE System SHALL start with documented default values and log a warning identifying the missing or invalid fields.
5. THE System SHALL validate all configuration values against defined ranges at load time and reject out-of-range values with a descriptive error message.
6. Configuration parameters SHALL be grouped into two tiers: Core (always visible in UI) and Advanced (hidden by default, expandable). This prevents UI bloat while preserving full configurability.

---

### Requirement 10: Next.js UI — Market Data Panel

**User Story:** As a trader, I want to see live Bank Nifty spot price and options chain data in the UI, so that I have situational awareness during trading hours.

#### Acceptance Criteria

1. THE UI SHALL display the current BN spot price updated on every received tick.
2. THE UI SHALL display the current options chain for the nearest expiry, showing strike, type (CE/PE), last traded price, and volume.
3. WHEN the WebSocket connection to the backend is lost, THE UI SHALL display a visible connection status indicator.

---

### Requirement 11: Next.js UI — Signal Panel

**User Story:** As a trader, I want to see all generated signals in real time, so that I can monitor strategy activity.

#### Acceptance Criteria

1. THE UI SHALL display each emitted signal with: timestamp, signal type (BUY/SELL), close price at signal, RSI value, slope method, trendline levels, and confidence score bar.
2. WHEN a signal is suppressed, THE UI SHALL display the full suppression reason(s) inline with an amber left border, expandable to show full context in the Insight Panel.
3. THE UI SHALL retain the last 50 signals in the signal panel without requiring a page reload.
4. THE UI SHALL group signal rows by structure_id, showing all signals (active and suppressed) from the same breakout sequence together.
5. THE Signal Panel header SHALL display a count badge showing active signals and suppressed signals separately (e.g., "Signals (12) | Suppressed (4)").

---

### Requirement 12: Next.js UI — Trade Table Panel

**User Story:** As a trader, I want to see all open and closed trades in a table, so that I can track performance in real time.

#### Acceptance Criteria

1. THE UI SHALL display all open trades with: trade ID, signal type, entry time, entry price, strike, current premium, unrealised P&L, stop-loss level, take-profit level, regime badge, time_bucket badge, and version badge.
2. THE UI SHALL display all closed trades with: trade ID, signal type, entry time, exit time, entry price, exit price, exit reason, realised P&L, trade_quality grade, MAE (max adverse excursion), and MFE (max favorable excursion).
3. WHEN a trade is closed, THE UI SHALL move it from the open trades section to the closed trades section without a page reload.
4. THE UI SHALL provide inline Exit Now and Hold TP action buttons on open trade rows.
5. WHEN a configuration change or engine stop is requested while trades are open, THE UI SHALL display a confirmation modal before applying the action.

---

### Requirement 13: Next.js UI — Decision Reasoning Panel (Insight Panel)

**User Story:** As a trader, I want to see the full reasoning behind each trade and the latest learning insights, so that I understand why the system acted and how it is improving.

#### Acceptance Criteria

1. WHEN a trade is selected in the Trade Table, THE UI SHALL display the full Decision_Context for that trade in the Insight Panel, including all logged features and the exit outcome.
2. THE UI SHALL display the most recent Learning_Engine insight report in the Insight Panel.
3. THE UI SHALL render the Decision_Context in a human-readable format, labelling each field with a plain-language description.
4. THE Insight Panel SHALL be always visible and never collapsible — it SHALL be sticky at full panel height.
5. WHEN any signal or trade is clicked anywhere in the UI, THE Insight Panel SHALL automatically receive focus and display the relevant context.
6. THE Insight Panel SHALL include a microstructure timeline showing: breakout detection time, confirmation time, entry time, and exit time with latency between each stage.
7. THE Insight Panel SHALL include an Override Log tab showing the trader's manual override history and outcomes.
8. EACH insight card in the Learning Report tab SHALL provide Simulate Impact, Apply, and Ignore actions.

---

### Requirement 14: Next.js UI — Controls Panel

**User Story:** As a trader, I want UI controls to start/stop the engine, override configuration, and trigger learning, so that I can operate the system without editing files.

#### Acceptance Criteria

1. THE UI SHALL provide a control to start and stop the Execution_Engine.
2. THE UI SHALL provide input fields for all configurable parameters defined in Requirement 9, pre-populated with current values.
3. WHEN a configuration change is submitted via the UI, THE System SHALL apply the change and confirm the update in the UI within 2 seconds.
4. THE UI SHALL provide a button to manually trigger a Learning_Engine cycle.
5. WHEN the manual learning trigger is activated, THE UI SHALL display a loading indicator until the insight report is available.

---

### Requirement 15: Decision Context Log — Round-Trip Integrity

**User Story:** As a developer, I want the Decision_Context serialization to be verifiably round-trip safe, so that no data is lost or corrupted when writing and reading trade logs.

#### Acceptance Criteria

1. THE Trade_Logger SHALL serialize Decision_Context records to JSON.
2. THE Trade_Logger SHALL deserialize JSON records back into Decision_Context objects.
3. FOR ALL valid Decision_Context objects, serializing then deserializing SHALL produce an object equal to the original (round-trip property).
4. IF a JSON record cannot be deserialized into a valid Decision_Context, THEN THE Trade_Logger SHALL log the malformed record and skip it without halting the Learning_Engine.

---

### Requirement 16: Regime Detection Layer

**User Story:** As a trader, I want the system to detect current market regime (trend, range, volatility), so that strategy behavior adapts to context.

#### Acceptance Criteria

1. THE System SHALL classify market regime per candle using volatility (ATR bands) and directional bias (trend slope / EMA).
2. THE System SHALL label regime as one of: TRENDING_UP, TRENDING_DOWN, RANGING, or HIGH_VOL.
3. THE regime label SHALL be attached to every Decision_Context.
4. THE Signal_Engine SHALL optionally filter signals based on regime (configurable).
5. IF regime detection fails, THEN THE System SHALL default to UNKNOWN and continue execution.

---

### Requirement 17: Signal De-duplication / Structure Awareness

**User Story:** As a trader, I want to avoid repeated entries from the same structural move, so that the system doesn't overtrade identical setups.

#### Acceptance Criteria

1. THE System SHALL assign a structure_id to each breakout sequence.
2. THE System SHALL suppress new signals if the same direction, same structure_id, and within a configurable candle window are all present simultaneously.
3. THE suppression reason SHALL be logged.
4. THE feature SHALL be configurable (on/off).

---

### Requirement 18: Slippage & Fill Simulation (Paper Realism)

**User Story:** As a trader, I want realistic execution simulation, so that paper trading reflects real market behavior.

#### Acceptance Criteria

1. THE Execution_Engine SHALL simulate slippage using a configurable basis points value OR bid/ask spread when available.
2. THE entry price SHALL not equal LTP by default when slippage simulation is enabled.
3. THE slippage value SHALL be stored in Decision_Context.
4. IF the slippage model is disabled, THEN THE Execution_Engine SHALL use raw LTP as the entry price.

---

### Requirement 19: Intraday Time Filters (Configurable)

**User Story:** As a trader, I want to restrict or bias trading during certain time windows, so that poor-performing periods are avoided.

#### Acceptance Criteria

1. THE System SHALL allow configuration of allowed trading windows as time ranges.
2. THE System SHALL tag each trade with a time_bucket label (e.g., OPEN, MIDDAY, CLOSE).
3. THE Learning_Engine SHALL analyze performance by time_bucket.
4. THE Signal_Engine SHALL optionally suppress signals outside allowed trading windows (configurable).

---

### Requirement 20: Multi-Timeframe Context

**User Story:** As a trader, I want higher timeframe context, so that signals are aligned with broader structure.

#### Acceptance Criteria

1. THE System SHALL support a configurable secondary timeframe (e.g., 15m) for trend bias computation.
2. THE System SHALL compute trend bias from the higher timeframe candles.
3. THE bias SHALL be attached to every Decision_Context.
4. THE Signal_Engine SHALL optionally require alignment with the higher timeframe bias before emitting a signal (configurable).

---

### Requirement 21: Strategy Versioning

**User Story:** As a developer, I want every strategy change versioned, so that improvements can be tracked and rolled back.

#### Acceptance Criteria

1. THE Strategy_Engine SHALL assign a version_id to each configuration snapshot.
2. EACH trade SHALL store the version_id of the active strategy configuration at time of entry.
3. THE Learning_Engine SHALL compare performance across strategy versions.
4. THE UI SHALL allow rollback to previous strategy versions.

---

### Requirement 22: Parameter Experiment Framework

**User Story:** As a trader, I want the system to test multiple parameter combinations, so that optimal configurations are discovered automatically.

#### Acceptance Criteria

1. THE System SHALL support running multiple parameter sets concurrently in paper mode.
2. EACH parameter set SHALL be treated as a separate strategy instance with its own trade log.
3. THE System SHALL track performance metrics per parameter set.
4. THE Learning_Engine SHALL rank parameter sets by performance.
5. THE top-performing parameter set SHALL be surfaced as a promotion suggestion in the UI.

---

### Requirement 23: Feature Snapshot at Signal Time

**User Story:** As a developer, I want features captured exactly at signal time, so that learning is accurate and not polluted by later data.

#### Acceptance Criteria

1. ALL signal_features SHALL be frozen at signal candle close time.
2. NO future data SHALL modify a stored Decision_Context record after it is written.
3. THE System SHALL distinguish between signal_features (captured at signal candle close) and execution_features (captured at order fill time, e.g., slippage and actual entry price).

---

### Requirement 24: Failure Mode Logging

**User Story:** As a developer, I want to capture system failures explicitly, so that debugging and robustness improve over time.

#### Acceptance Criteria

1. THE System SHALL log missed signals, execution failures, and data gaps as distinct failure record types.
2. EACH failure record SHALL include: timestamp, module name, and a context snapshot.
3. THE UI SHALL expose a recent failures panel showing the last N failure events (N configurable).

---

---

### Requirement 25: Cognitive Load Mode (Adaptive UI Density)

**User Story:** As a trader, I want the UI to adapt its density dynamically, so that I can focus only on critical information during high volatility.

#### Acceptance Criteria

1. THE UI SHALL support three density modes: NORMAL, FOCUSED, and CRITICAL.
2. THE UI SHALL automatically switch to FOCUSED mode when an open trade is active.
3. THE UI SHALL automatically switch to CRITICAL mode when the regime is HIGH_VOL or an ATR spike is detected.
4. IN CRITICAL mode, THE UI SHALL display only: spot price, open trades, and exit controls.
5. THE UI SHALL allow the user to manually lock to any density mode via a header toggle.
6. THE current density mode SHALL be visible as a labeled badge in the header at all times.

---

### Requirement 26: Signal Confidence Score

**User Story:** As a trader, I want each signal to show a confidence score, so that I can quickly assess signal quality without reading raw numbers.

#### Acceptance Criteria

1. THE Lorentz_Signal_Engine SHALL compute a confidence score (0.0–1.0) for each emitted signal derived from: breakout strength, RSI distance from threshold, slope steepness, and regime alignment.
2. THE confidence score SHALL be included in the Signal payload and stored in signal_features.
3. THE UI SHALL render the confidence score as a visual bar alongside each signal row.
4. WHEN confidence is below 0.5, THE UI SHALL dim the signal row visually.

---

### Requirement 27: Trade Quality Score

**User Story:** As a trader, I want each closed trade graded on execution quality, so that the Learning Engine can weight better setups more heavily.

#### Acceptance Criteria

1. THE Execution_Engine SHALL compute a trade_quality grade (A / B / C / D) for each closed trade based on: regime alignment at entry, RSI alignment, exit reason, and candles held to exit.
2. THE trade_quality grade SHALL be stored in Decision_Context.
3. THE UI SHALL display the grade inline on closed trade rows in the Trade Table.
4. THE Learning_Engine SHALL use trade_quality as a weighting factor in clustering and insight generation.

---

### Requirement 28: Replay Mode

**User Story:** As a trader, I want to replay any closed trade candle-by-candle, so that I can deeply understand system behavior and decision points.

#### Acceptance Criteria

1. THE UI SHALL provide a Replay button for each closed trade in the Insight Panel.
2. WHEN Replay is activated, THE UI SHALL reconstruct and display the candle sequence from N candles before entry through exit.
3. THE Replay SHALL show: evolving trendlines, RSI, regime label, and Breakout_State_Machine transitions at each candle step.
4. THE Replay SHALL support playback speeds of 1×, 5×, and 10×.
5. THE Replay SHALL support keyboard navigation: right arrow (next candle), left arrow (previous candle), space (play/pause).

---

### Requirement 29: Strategy Drift Detection

**User Story:** As a developer, I want the system to detect when strategy performance is degrading, so that I can intervene before significant losses accumulate.

#### Acceptance Criteria

1. THE System SHALL compute a drift score by comparing the recent win rate (last 10 trades) against the 20-trade rolling baseline.
2. THE System SHALL classify drift as: LOW (within 5% of baseline), MED (5–15% below baseline), or HIGH (> 15% below baseline or variance spike).
3. THE drift level SHALL be displayed as a badge in the UI header at all times.
4. WHEN drift reaches HIGH, THE System SHALL emit a smart alert and optionally pause new signal processing (configurable).

---

### Requirement 30: Human Override Memory

**User Story:** As a trader, I want my manual overrides logged and learned from, so that the system improves based on my judgment over time.

#### Acceptance Criteria

1. WHEN a trader performs a manual override (Exit Now / Hold TP / Disable strategy), THE Trade_Logger SHALL persist an override record containing: override type, trade_id, timestamp, and outcome (better / worse / neutral, evaluated at trade close).
2. THE override history SHALL be visible in the Insight Panel under an Override Log tab.
3. THE Learning_Engine SHALL analyze override outcomes and surface patterns as insight cards (e.g., "Holding beyond TP in TRENDING_UP improves outcome 70% of the time").

---

### Requirement 31: Visual Risk Meter

**User Story:** As a trader, I want a live risk exposure indicator, so that I always know how much capital is at risk.

#### Acceptance Criteria

1. THE UI SHALL display a risk meter in the header showing current exposure as a percentage of total capital.
2. THE risk meter SHALL be computed from: capital in open trades + unrealised loss + a volatility multiplier.
3. THE risk meter SHALL color-code: green (< 40%), amber (40–70%), red (> 70%).
4. WHEN the daily loss threshold (Requirement 9) is breached, THE System SHALL halt new signal processing and display a full-width warning banner.

---

### Requirement 32: Fail-Safe UI State

**User Story:** As a trader, I want the UI to freeze and alert me when a critical failure occurs, so that I never trade on stale or missing data.

#### Acceptance Criteria

1. WHEN a data gap exceeding 30 seconds is detected, THE UI SHALL display a full-width blocking banner: "Execution paused — data gap detected".
2. WHEN the banner is active, THE System SHALL halt new signal processing.
3. THE banner SHALL offer two actions only: Resume (when feed is restored) and Exit All (close all open trades at last known price).
4. THE UI SHALL NOT update any price or P&L values while in the paused state.

---

### Requirement 33: Autonomous Post-Market Research Loop

**User Story:** As a trader, I want the system to autonomously analyze each day's trades after market close, so that I wake up to actionable insights and ready-to-apply parameter improvements without manual effort.

#### Acceptance Criteria

1. THE Learning_Engine SHALL support an autonomous research loop that executes the following steps in sequence: LOAD → BASELINE → HYPOTHESIZE → TEST → RANK → REPORT → SUGGEST.
2. THE research loop SHALL trigger automatically at 15:35 IST (5 minutes after market close) and SHALL also be triggerable manually via the UI Controls panel.
3. WHEN drift level reaches HIGH, THE System SHALL trigger a research loop immediately without waiting for the scheduled time.
4. THE research loop SHALL load all completed trades from the current session's trade log and compute a session summary including: total trades, win rate, average P&L, trades by regime, trades by time bucket, trades by slope method, suppressed count, and average confidence score.
5. THE research loop SHALL compute a rolling baseline from the last 20 completed trades across all sessions and classify today's session as outperformance, underperformance, or normal relative to that baseline.
6. THE research loop SHALL form and test hypotheses across the following dimensions: RSI threshold, regime filter, time bucket, slope method, confidence score threshold, human override patterns, and trade quality grade correlation.
7. FOR EACH hypothesis, THE research loop SHALL backtest against the full historical trade log and compute: win rate, average P&L, trade count, and confidence interval for the hypothesis bucket.
8. IF a hypothesis bucket contains fewer than 10 trades, THEN THE research loop SHALL mark that hypothesis as PENDING and exclude it from ranked suggestions.
9. THE research loop SHALL rank confirmed hypotheses by a score derived from expected improvement, evidence confidence, and implementation complexity.
10. THE research loop SHALL write a structured research report to `backend/memory/insights/YYYY-MM-DD.json` containing: session summary, hypotheses tested, ranked insights with evidence, drift assessment, and recommended next experiment.
11. THE research loop SHALL write a ready-to-apply settings diff to `backend/memory/suggestions/YYYY-MM-DD-settings-diff.json` for all top-ranked confirmed insights.
12. THE research loop SHALL assign a unique experiment ID (format: `EXP-YYYY-MM-DD-NNN`) to each cycle, creating a traceable research lineage.
13. IF total completed trades for the session are fewer than 5, THEN THE research loop SHALL write an "insufficient data" report and skip hypothesis testing.
14. THE research loop SHALL NEVER suggest disabling the entire strategy — only parameter-level changes within defined valid ranges.
15. THE research loop SHALL NEVER suggest changes based on fewer than 10 trades per hypothesis bucket.
16. THE UI SHALL expose the most recent research report in the Insight Panel's Learning Report tab, including the session summary, ranked insights, and the ready-to-apply settings diff with Simulate / Apply / Ignore actions per insight.

---

### Requirement 34: Weekly Meta-Research Cycle

**User Story:** As a trader, I want a weekly research summary every Friday, so that I can track whether applied parameter changes actually improved performance over the week.

#### Acceptance Criteria

1. THE Learning_Engine SHALL run an extended weekly research cycle every Friday after market close, in addition to the daily cycle.
2. THE weekly cycle SHALL aggregate all 5 daily experiment results for the week and compare projected improvements against actual observed improvements for each applied change.
3. THE weekly cycle SHALL identify which parameter changes were applied during the week and compute their actual vs projected impact on win rate and P&L.
4. THE weekly cycle SHALL update the rolling baseline with the full week's data.
5. THE weekly cycle SHALL suggest one "bold experiment" for the following week — a higher-risk, higher-potential-reward parameter combination not yet tested.
6. THE weekly cycle SHALL write its report to `backend/memory/insights/weekly/YYYY-WNN.json`.
7. THE UI SHALL surface the most recent weekly report in the Insight Panel alongside the daily report.

---

## UX / UI Design Guidance

The following guidance applies to the Next.js UI implementation and informs design decisions across Requirements 10–14 and 21–32.

- Dark theme by default (trading terminal standard).
- Dense information layout with clear visual hierarchy.
- Real-time updates without full page reloads (WebSocket-driven).
- Color coding: green for BUY / profit, red for SELL / loss, amber for warnings.
- Collapsible panels for advanced controls (aligned with the Core / Advanced configuration tiers in Requirement 9).
- Keyboard shortcuts for common actions: start/stop engine, trigger learning cycle, replay, focus Insight Panel.
- The Insight / Reasoning panel (Requirement 13) SHALL be the most visually prominent panel, always sticky, never collapsible.
- Time_Bucket and Regime labels SHALL be visible inline on trade rows in the Trade Table.
- The active Strategy_Version badge, Drift indicator, Latency indicator, and Risk Meter SHALL all be visible in the UI header at all times.
- Signal Panel SHALL group signals by structure_id to show the market story.
- Trade Table SHALL include MAE and MFE columns for closed trades.
- Controls panel SHALL require confirmation before applying config changes while trades are open or stopping the engine.
