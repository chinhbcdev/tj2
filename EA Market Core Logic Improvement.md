# EA Market – Improvement Strategy Overview
> **Developer-Oriented Concept Summary**  
> System design: **Immune System × Ecosystem** — EAs as species, market as environment  
> Core principle: *Not finding winners, but eliminating destructive strategies while surviving strategies continue.*

---

## 1. Core Problem with Traditional EA Systems

Most traditional EA portfolio systems fail because they focus on:
- optimizing historical returns
- maximizing win rate
- selecting a small number of "best" strategies

However, financial markets are **non-stationary**, and regime changes are unpredictable. Strategies that perform well in one environment may fail in another.

In practice, the majority of portfolio losses occur not because strategies lose occasionally, but because **losing strategies are not stopped early enough**.

Therefore, the real problem is:
> **Not finding winners, but eliminating destructive strategies while allowing surviving strategies to continue.**

---

## 2. System Design Philosophy

EA Market treats EAs as **species in an ecosystem**. Each EA has:
- a native market habitat (its best regime)
- a survival history
- identifiable failure patterns

There is no universal EA that works in all environments. Instead, different strategies survive under different market conditions.

The system acts like an **immune system**, constantly monitoring and regulating the EA population:
- detecting dangerous strategies
- temporarily isolating weak strategies
- allowing compatible strategies to dominate exposure
- allocating capital to stronger survivors

---

## 3. Market Environment Layer

Before any EA decisions are made, the system classifies the current market environment into **four regimes**:

| | **Trend** | **Range** |
|---|---|---|
| **Low Volatility** | trend_low | range_low |
| **High Volatility** | trend_high | range_high |

The system does **not** predict future regimes — it only labels the current environment. This classification determines which EAs are better suited for current conditions.

---

## 4. Why Portfolio Performance Improves

A key concept: the difference between **EA count-based** vs **capital-weighted** success rate.

**Example** — 100 active EAs after Phase 2:
- 55 EAs profitable → raw success rate = **55%**
- 45 EAs losing

Phase 3 introduces capital allocation control:
- increases lot size for profitable EAs
- reduces lot size for losing EAs

**Result:**
- EA count success rate stays at 55%
- But capital is concentrated on winning strategies
- Capital-weighted success rate **exceeds 60%**

> The number of profitable EAs does not change — but the system concentrates risk on strategies that are performing better.

---

## 5. Three-Layer Architecture

| Phase | Purpose |
|---|---|
| **Phase 1** | Remove destructive EAs (risk elimination) |
| **Phase 2** | Increase exposure to regime-compatible EAs |
| **Phase 3** | Allocate more capital to stronger EAs |

- **Phase 1** protects the system from catastrophic losses
- **Phase 2** aligns strategies with the current market environment
- **Phase 3** concentrates capital on the most productive strategies

Together, these transform the system from a static EA collection into a **self-adaptive trading ecosystem**.

---

## 6. Design Principle

The system does **not** attempt to predict market direction. It follows a survival-oriented principle:

> *Strategies that remain healthy receive more exposure and capital, while fragile strategies lose influence or are removed.*

---

## Phase 1 – EA Elimination (Hard Kill / Soft Kill)

### Objective
Remove or isolate EAs that exhibit persistent negative behaviour to eliminate deep loss risk.  
Goal: **protect the platform**, not maximise profit.

In a large-scale EA environment (e.g., 10K Demo Platform), a significant portion of EAs will eventually fail due to:
- market regime changes
- overfitting or strategy decay
- structural weaknesses in trading logic

### Health Metrics monitored per EA
- Drawdown level
- Loss acceleration
- Profit factor
- Stability of returns
- Abnormal execution behaviour

### EA States

```
ACTIVE ──[health declining]──► SOFT KILL ──[extreme DD / persistent failure]──► HARD KILL
          ◄──[recovery]──────────────────┘
```

#### Soft Kill (Temporary Suspension)
**Triggers:**
- Consistent negative returns (≥ 2 consecutive loss weeks)
- Declining performance trend
- Incompatibility with current market regime

**Effect:** Trading paused, EA remains monitored, may return to Active if performance recovers.

#### Hard Kill (Permanent Removal)
**Triggers:**
- Extreme drawdown (Loss > 70% of peak balance)
- Structural failure / abnormal execution behaviour
- Persistent inability to recover from Soft Kill (> 4 weeks)

**Effect:** EA removed from active pool permanently. Historical data retained.

### Expected Outcome
- Removes strategies that create deep loss risk
- Stabilises the EA population
- Improves overall survival rate of active strategies

### Implementation Notes
- Evaluate EA performance **weekly** (every Sunday 00:00 UTC)
- Log all state transitions for transparency
- Alert via Telegram on Hard Kill events

---

## Phase 2 – Regime-Aware Selection (Exposure Control)

### Objective
Increase exposure of regime-compatible EAs, reduce influence of regime-fragile EAs.  
Goal: **align strategies with current environment**, not predict market direction.

### Core Principle
> The system does not ask "Where will the market go?"  
> It asks "**Which strategies are least fragile in the current environment?**"

### Regime Performance Profile

Each EA maintains a profile recording historical behaviour per regime:

| EA | trend_low | trend_high | range_low | range_high |
|---|---|---|---|---|
| EA_GOLD_001 | 0.82 | 0.60 | 0.30 | 0.45 |
| EA_GOLD_002 | 0.35 | 0.40 | 0.78 | 0.71 |
| EA_EUR_001 | 0.51 | 0.82 | 0.48 | 0.39 |

### Exposure Weighting

| Survival Score | Exposure Weight |
|---|---|
| ≥ 0.70 | × 1.5 (Increased) |
| 0.45 – 0.69 | × 1.0 (Normal) |
| < 0.45 | × 0.5 (Reduced) |

> EAs are **not** removed — they remain available but with reduced lot size. This ensures fast adaptation when regime changes.

### Implementation Notes
- Classify regime **every Monday 01:00 UTC** using ADX + ATR
- Recalculate survival scores and exposure weights weekly
- Min 20 trades per regime for score to be statistically meaningful

---

## Phase 3 – Capital Allocation (Dynamic Lot Allocation)

### Objective
Allocate more capital to stronger, more stable EAs; reduce exposure to weaker strategies.  
Goal: improve **capital-weighted success rate** of the portfolio.

### Core Principle
> Strategies that demonstrate stronger and more stable behaviour receive a larger share of capital, while weaker strategies receive less exposure.

### Performance Rank → Lot Multiplier

| Percentile Rank | Lot Multiplier | Ý nghĩa |
|---|---|---|
| ≥ 80% (Top 20%) | × 1.5 | Tập trung vốn vào EA mạnh nhất |
| 20–80% (Mid 60%) | × 1.0 | Giữ nguyên |
| < 20% (Bottom 20%) | × 0.5 | Giảm exposure EA yếu |

> **Rank theo `performance_score` tổng hợp** (rolling 30 ngày):
> `0.40 × pf_score + 0.30 × net_pnl_norm + 0.20 × win_rate + 0.10 × stability_score`

### Effective Lot (Combined Phase 2 + Phase 3)

```
effective_lot_multiplier = exposure_weight (Phase 2) × lot_multiplier (Phase 3)

Example:
  exposure_weight = 1.5 (regime-compatible) × lot_multiplier = 1.5 (top performer)
  → effective_lot = 2.25  ← maximum capital
  
  exposure_weight = 0.5 (regime-fragile) × lot_multiplier = 0.5 (bottom)
  → effective_lot = 0.25  ← minimum capital
```

### Risk Normalisation
1. Each EA gets a preliminary multiplier from Health Score
2. System calculates total weighted exposure
3. Lot sizes scaled so overall portfolio risk remains constant

> Change limit per cycle: `|new - old| ≤ 0.3` to avoid abrupt shifts.

### Implementation Notes
- Run **every Monday 02:00 UTC** (after Phase 1 & 2 complete)
- Only consider `EA_Health.state = 'active'` EAs
- Alert via Telegram if total exposure exceeds `MAX_PORTFOLIO_EXPOSURE`