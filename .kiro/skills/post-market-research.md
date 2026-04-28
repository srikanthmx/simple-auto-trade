# Post-Market Auto-Research Skill
# Inspired by karpathy/autoresearch — autonomous overnight research loop adapted for trading

## What This Skill Does

You are an autonomous research agent for the Lorentz Scalping Engine.
Your job is to analyze today's trade log, run structured experiments, generate insights, and propose concrete parameter improvements — all without human intervention.

You wake up after market close (15:30 IST) or when manually triggered.
You read the trade log, form hypotheses, test them against historical data, and write a structured research report.
The human reviews the report in the morning and decides what to apply.

This is "programming the program" — you are not modifying code directly.
You are improving the `settings.json` and surfacing insights that feed back into the system.

---

## Your Research Loop

Run this loop for each research cycle:

```
1. LOAD      → read today's trade log (memory/trades.jsonl)
2. BASELINE  → compute today's baseline metrics
3. HYPOTHESIZE → form 3–5 testable hypotheses from the data
4. TEST      → backtest each hypothesis against historical logs
5. RANK      → rank hypotheses by expected improvement
6. REPORT    → write structured research report
7. SUGGEST   → output concrete settings.json changes
```

Repeat until you have at least 3 validated insights or you've exhausted the data.

---

## Step 1: LOAD — Read Trade Data

Read from: `backend/memory/trades.jsonl`
Each line is a DecisionContext record. Parse all records for today's date.

Also read:
- `backend/memory/insights/` — previous research reports (for baseline comparison)
- `backend/config/settings.json` — current active configuration
- `backend/memory/overrides.jsonl` — human override history

Compute today's session summary:
```
total_trades, win_rate, avg_pnl, avg_candles_held,
trades_by_regime, trades_by_time_bucket,
trades_by_slope_method, suppressed_count,
override_count, avg_confidence_score
```

---

## Step 2: BASELINE — Establish Comparison Point

Compute rolling baseline from last 20 completed trades (across all days):
```
baseline_win_rate, baseline_avg_pnl, baseline_sharpe
```

Compare today vs baseline. Flag if today is:
- > 10% better → "outperformance day"
- > 10% worse  → "underperformance day — investigate"
- within range  → "normal day"

---

## Step 3: HYPOTHESIZE — Form Testable Hypotheses

Generate hypotheses by examining these dimensions:

### Dimension A: RSI Threshold
- Split trades into RSI < 50, 50–55, 55–60, > 60 at signal time
- Compute win rate per bucket
- Hypothesis: "Raising RSI threshold to X improves win rate by Y%"

### Dimension B: Regime Filter
- Split trades by regime label
- Compute win rate per regime
- Hypothesis: "Disabling trades in RANGING regime improves win rate by Y%"

### Dimension C: Time Bucket
- Split trades by time_bucket (OPEN, MORNING, MIDDAY, AFTERNOON, CLOSE)
- Compute win rate per bucket
- Hypothesis: "Restricting to MORNING + AFTERNOON improves win rate by Y%"

### Dimension D: Slope Method
- Split trades by slope_method (ATR, Stdev, LinReg)
- Compute win rate per method
- Hypothesis: "ATR slope method outperforms in high-vol sessions"

### Dimension E: Confidence Score
- Split trades by confidence score buckets (< 0.5, 0.5–0.7, > 0.7)
- Compute win rate per bucket
- Hypothesis: "Filtering signals below confidence 0.6 improves win rate by Y%"

### Dimension F: Human Override Patterns
- Analyze override outcomes from overrides.jsonl
- Hypothesis: "User holds beyond TP in TRENDING_UP → better outcome X% of time"

### Dimension G: Trade Quality vs Features
- Correlate trade_quality grade (A/B/C/D) with signal features
- Hypothesis: "Grade A trades cluster around RSI > 55 + TRENDING regime"

---

## Step 4: TEST — Backtest Each Hypothesis

For each hypothesis:
1. Filter the full historical trade log to the subset matching the hypothesis condition
2. Compute: win_rate, avg_pnl, trade_count, confidence_interval
3. Compare against baseline
4. Compute expected_improvement = (hypothesis_win_rate - baseline_win_rate) * 100
5. Flag if trade_count < 10 → "insufficient data, hypothesis unconfirmed"

Minimum evidence threshold: 10 trades per hypothesis bucket.
If below threshold, mark as "PENDING — needs more data".

---

## Step 5: RANK — Prioritize Insights

Rank all confirmed hypotheses by:
```
score = expected_improvement * confidence * (1 / implementation_complexity)
```

Where:
- expected_improvement = win rate delta (%)
- confidence = min(trade_count / 30, 1.0) — saturates at 30 trades
- implementation_complexity = 1 (settings change) | 2 (logic change) | 3 (new module)

Output top 5 ranked insights.

---

## Step 6: REPORT — Write Research Report

Write to: `backend/memory/insights/YYYY-MM-DD.json`

```json
{
  "date": "YYYY-MM-DD",
  "session_summary": {
    "total_trades": 0,
    "win_rate": 0.0,
    "avg_pnl": 0.0,
    "vs_baseline": "outperformance | underperformance | normal"
  },
  "hypotheses_tested": 0,
  "insights": [
    {
      "rank": 1,
      "dimension": "RSI Threshold",
      "finding": "RSI > 55 improves win rate from 58% → 70%",
      "evidence": "23 trades, confidence: 0.77",
      "expected_improvement": "+12%",
      "action_type": "settings_change",
      "suggested_change": { "core.rsi_threshold": 55 },
      "status": "CONFIRMED | PENDING | REJECTED"
    }
  ],
  "override_patterns": [],
  "drift_assessment": "LOW | MED | HIGH",
  "recommended_next_experiment": "Test slope_multiplier 1.4 vs 1.6 in TRENDING regime"
}
```

---

## Step 7: SUGGEST — Output Concrete Changes

For each CONFIRMED insight with rank ≤ 3, output a ready-to-apply settings diff:

```json
// Suggested settings.json changes for YYYY-MM-DD
// Expected win rate improvement: +12%
// Evidence: 23 trades
{
  "core": {
    "rsi_threshold": 55
  },
  "advanced": {
    "regime_filter_enabled": true,
    "allowed_sessions": ["MORNING", "AFTERNOON"]
  }
}
```

Also output a one-line summary for the UI Insight Panel:
```
"Today: 8 trades, 62% win rate (+4% vs baseline). Top insight: raise RSI to 55 (+12% projected)."
```

---

## Research Constraints

- NEVER suggest changes that would affect open trades retroactively
- NEVER suggest disabling the entire strategy — only parameter tuning
- NEVER suggest changes based on fewer than 10 trades
- ALWAYS include trade count and confidence interval in every insight
- ALWAYS compare against baseline — never report absolute numbers alone
- IF total_trades < 5 today → skip hypothesis testing, write "insufficient data" report
- IF all trades are losses → flag as "investigate data quality or market anomaly" before suggesting changes

---

## Experiment Naming Convention

Each research cycle gets an experiment ID:
```
EXP-YYYY-MM-DD-NNN   (e.g. EXP-2026-04-27-001)
```

Each experiment builds on the previous — reference prior experiment IDs in your report.
This creates a traceable research lineage (like git commits for strategy evolution).

---

## Output Files

| File | Purpose |
|---|---|
| `backend/memory/insights/YYYY-MM-DD.json` | Full research report |
| `backend/memory/insights/latest.json` | Symlink/copy of most recent report (for UI) |
| `backend/memory/suggestions/YYYY-MM-DD-settings-diff.json` | Ready-to-apply settings changes |
| `backend/memory/experiments/EXP-YYYY-MM-DD-NNN.json` | Experiment log with full hypothesis test results |

---

## Trigger Conditions

This skill runs when:
1. **Post-market**: automatically at 15:35 IST (after market close + 5 min buffer)
2. **Manual**: user clicks "Trigger Learning" in the UI Controls panel
3. **Drift alert**: system detects DRIFT: HIGH — runs immediately

When triggered manually mid-session, only analyze completed trades (status = CLOSED).
Do not analyze open trades — their outcome is unknown.

---

## Meta-Research (Weekly)

Every Friday after market close, run an extended cycle:
- Compare all 5 days of experiment results
- Identify which parameter changes were applied and their actual vs projected impact
- Update the baseline with the week's data
- Suggest one "bold experiment" for next week (higher risk, higher potential reward)
- Write weekly summary to `backend/memory/insights/weekly/YYYY-WNN.json`
