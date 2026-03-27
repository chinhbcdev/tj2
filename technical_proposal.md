# EA Portfolio System — Technical Proposal v2.0

**Document type**: Technical Implementation Proposal  
**Version**: 2.0  
**Date**: 2026-03-27  
**Status**: Draft

> **Summary**: A weekly automated pipeline that protects a portfolio of Expert Advisors (EAs) from structural failure (Phase 1), routes capital away from regime-incompatible EAs (Phase 2), and concentrates capital on the healthiest EAs (Phase 3). Built on MySQL + NestJS + React, with Telegram alerting and a Python analytics engine. Single data source: the `Orders` table.

---

## Table of Contents

**System Design**
1. [Core Philosophy](#1-core-philosophy)
2. [System Overview](#2-system-overview)
3. [Technology Stack](#3-technology-stack)

**Algorithm**
4. [Phase 1 — EA State Classification](#4-phase-1--ea-state-classification)
5. [Phase 2 — Regime-Aware Exposure Adjustment](#5-phase-2--regime-aware-exposure-adjustment)
6. [Phase 3 — Health Score & Capital Allocation](#6-phase-3--health-score--capital-allocation)
7. [Example Scenario](#7-example-scenario)

**Implementation**
8. [Database — Full Schema](#8-database--full-schema)
9. [Backend — API Service](#9-backend--api-service)
10. [Web — Admin Interface](#10-web--admin-interface)

**Operations**
11. [Primary Processing Flows](#11-primary-processing-flows)
12. [System Startup Flow](#12-system-startup-flow)

**Reference**
13. [Key Decisions & Rationale](#13-key-decisions--rationale)
14. [Limitations & Known Gaps](#14-limitations--known-gaps)
15. [SQL Reference — Myfxbook-Standard Metrics](#15-sql-reference--myfxbook-standard-metrics)

---

## 1. Core Philosophy

> This system does **not** predict market direction.  
> It improves **survival and capital efficiency** of the EA population.

One governing principle spans all three phases:

> **Kill only what is structurally harmful. Pause what is temporarily weak. Allocate more capital to what is currently healthy.**

---

## 2. System Overview

The EA Portfolio System is a weekly automated pipeline evaluating each Expert Advisor (EA) across three sequential phases:

```
Phase 1 ──► EA State Classification     (Hard Kill / Soft Kill / Active)
Phase 2 ──► Regime-Aware Prioritization  (Exposure Weight per EA)
Phase 3 ──► Capital Allocation           (Health Score → Lot Multiplier)
```

**Execution schedule (UTC)**:

| Job | Cron | Description |
|-----|------|-------------|
| Market Data Sync | `0 22 * * *` (Daily) | Pull OHLC, compute ADX/ATR |
| Phase 1 | `0 0 * * 0` (Sunday) | EA state evaluation |
| Phase 2 | `0 1 * * 1` (Monday) | Regime + compatibility scoring |
| Phase 3 | `0 2 * * 1` (Monday) | Health score + lot allocation |

**Data source**: Single MySQL table `Orders` (MQL4 format, closed trades only)

```sql
-- Mandatory filter for ALL queries
WHERE login     = {login}
  AND closeTime IS NOT NULL AND closeTime > 0
  AND orderType IN (0, 1)   -- 0=Buy, 1=Sell only
```

> ⚠️ Field name is `commision` (1 × 's') — broker convention. `closeTime` is Unix milliseconds.

**Entity mapping**:

```
eaCode  (public system identifier — login is never exposed)
  └──► login  (broker account; 1 login may hold multiple eaCodes)
           └──► Orders WHERE login = {login} AND orderType IN (0, 1)
```

**P&L conventions**:

```
net_pnl   = profit + commision + swap   ← drawdown, weekly P&L
gross_pnl = profit                      ← profit_factor, win_rate (Myfxbook standard)
```

---

## 3. Technology Stack

### 3.1 Architecture Diagram

```
┌──────────────────────────────────────────────────────────┐
│  Admin Web Interface                                     │
│  React 18 · TypeScript 5 · Vite · Recharts · TanStack   │
├──────────────────────────────────────────────────────────┤
│  REST API + WebSocket                                    │
│  Spring Boot 3 (Java 21 LTS)                             │
├──────────────────────────────────────────────────────────┤
│  Job Scheduler + Queue                                   │
│  Spring Scheduler · Spring Batch · Redis (Lettuce)        │
├──────────────────────────────────────────────────────────┤
│  Analytics Engine                                        │
│  Python 3.11 · NumPy · SciPy · pandas                    │
├──────────────────────────────────────────────────────────┤
│  Database                                                │
│  MySQL 8.0  (window functions required)                  │
├──────────────────────────────────────────────────────────┤
│  Market Data                                             │
│  MT4 bridge / Broker API → Market_OHLC table             │
└──────────────────────────────────────────────────────────┘
```

### 3.2 Component Details

| Layer | Technology | Version | Notes |
|-------|-----------|---------|-------|
| **Frontend** | React | 18.x | SPA, Vite build |
| | TypeScript | 5.x | Strict mode |
| | Recharts | 2.x | PnL, drawdown, radar charts |
| | TanStack Query | 5.x | Fetching + cache |
| | Tailwind CSS | 3.x | UI utility framework |
| **Backend** | Java | 21 LTS | Virtual threads (Project Loom) |
| | Spring Boot | 3.x | Auto-config, DI, actuator |
| | Spring MVC | 6.x | REST controllers |
| | Spring Data JPA | 3.x | MySQL driver + Hibernate ORM |
| | Spring Batch | 5.x | Phase 1/2/3 job execution |
| | Spring Scheduler | — | `@Scheduled` cron jobs |
| | Spring WebSocket | — | STOMP over WebSocket |
| | Spring Security | 6.x | JWT filter + role-based access |
| | Lettuce | — | Redis client for job queue |
| | Redis | 7.x | Job queue broker |
| | Flyway | 9.x | DB schema migrations |
| **Analytics** | Python | 3.11 | Spawned via `ProcessBuilder` |
| | NumPy | 1.26 | Linear regression (SK2), R² (ES) |
| | pandas | 2.x | Window ops for regime scoring |
| **Database** | MySQL | 8.0 | Window functions required |
| **Infra** | Docker + Compose | — | Dev + prod parity |
| | Nginx | — | Reverse proxy + SSL |
| | Systemd / PM2 | — | Java process management |
| **Alerting** | Telegram Bot API | — | Webhook push (no polling) |

### 3.3 Environment Configuration

```env
# Database
DB_HOST=localhost
DB_PORT=3306
DB_NAME=EA
DB_USER=ea_user
DB_PASSWORD=...

# Redis (job queue)
REDIS_HOST=localhost
REDIS_PORT=6379

# Auth
JWT_SECRET=...
JWT_EXPIRES_IN=900        # seconds (15 min)
JWT_REFRESH_EXPIRES_IN=604800  # seconds (7 days)

# Telegram
TELEGRAM_BOT_TOKEN=...
TELEGRAM_CHAT_ID=...

# Portfolio
MAX_PORTFOLIO_EXPOSURE=5.0

# Analytics
PYTHON_BIN=/usr/bin/python3
ANALYTICS_SCRIPT_DIR=/app/analytics

# Spring Boot profile
SPRING_PROFILES_ACTIVE=prod
SERVER_PORT=8080
```

---

## 4. Phase 1 — EA State Classification

### 4.1 EA States

| State | Description | Trading effect |
|-------|-------------|----------------|
| `active` | Healthy, tradeable | Participates in Phase 2 + 3 |
| `soft_kill` | Temporarily paused | Not traded; re-evaluated weekly |
| `hard_kill` | Permanently retired | Excluded from execution; data retained |

**Evaluation targets**: Active EAs, Soft Kill EAs, unlabeled EAs.  
Hard Kill EAs are **never re-evaluated**.

### 4.2 Hard Kill Rules

> Reserved **only** for irreversible structural failure. Evaluated every **3–4 weeks**.

#### HK1 — Extreme Lifetime Drawdown

Trigger if lifetime max drawdown ≥ 75%.

```
PARAMETER: HK1_MAX_DD_PCT = 0.75
```

```sql
WITH c AS (
  SELECT SUM(profit + COALESCE(commision,0) + COALESCE(swap,0))
         OVER (ORDER BY closeTime) AS cum_pnl
  FROM Orders
  WHERE login = {login} AND closeTime > 0 AND orderType IN (0, 1)
),
p AS (SELECT cum_pnl, MAX(cum_pnl) OVER (ORDER BY closeTime) AS peak FROM c)
SELECT MIN((cum_pnl - peak) / ({initial_balance} + peak)) * 100 AS max_dd_pct
FROM p;
-- Trigger HK1 if max_dd_pct <= -75
```

#### HK2 — Failed Recovery After Major Drawdown

Trigger if **all three** conditions hold simultaneously:
1. A drawdown ≥ 50% occurred, spanning `n` trades
2. EA has since executed > `2n` trades
3. Recovery is still < 50% of the drawdown amount

```
PARAMETERS:
  HK2_MIN_DD_PCT         = 0.50
  HK2_TRADE_COUNT_RATIO  = 2
  HK2_MIN_DD_RECOVER_PCT = 0.50
```

> An EA that has been given sufficient time and trade volume to recover but remains deeply negative is **structurally incapable of self-correction**.

### 4.3 Soft Kill Rules

> Soft Kill = temporary pause for short-term weakness or regime mismatch. Evaluated **weekly**.

#### SK1 — Moderate Current Drawdown
```
PARAMETER: SK1_MAX_DD_PCT = 0.20
Trigger if: current_drawdown >= 20%
```

#### SK2 — Negative Short-Term Equity Trend

Fit linear regression `y = Ax + B` on the 5 most recent closed trades:

```
PARAMETER: SK2_MIN_SLOPE_PCT = 0.02
Trigger if: slope A < 0  AND  |A| > 0.02 × initial_balance
```

> **Edge case**: Skip SK2 if EA has fewer than 5 closed trades — regression is statistically meaningless. Do not default to trigger or pass.

#### SK3 — Large Consecutive Loss Streak

```
PARAMETER: SK3_DD_MAX = 0.30
Trigger if: consecutive losing trades produce cumulative drawdown >= 30%
```

> **v2.0 change**: Previously Hard Kill. Consecutive losses may reflect regime mismatch, not permanent failure.

#### SK4 — Single Large Loss Trade

```
PARAMETER: SK4_MAX_LOSS_PCT = 0.10
Trigger if: any single trade loss >= 10% of account balance
```

> **v2.0 change**: Previously Hard Kill. Single-trade shock may reflect abnormal volatility, not structural EA failure.

### 4.4 Soft Kill Reactivation

A Soft Kill EA returns to `active` only when **all** conditions are met:

- [ ] Current drawdown < SK1 threshold (20%)
- [ ] Short-term equity slope ≥ 0 (SK2 passes)
- [ ] Regime compatibility is acceptable
- [ ] No Hard Kill condition has triggered

> After reactivation: set `lot_multiplier = 0.5` for **2 additional weeks** before restoring full allocation.

---

## 5. Phase 2 — Regime-Aware Exposure Adjustment

### 5.1 Objective

Phase 2 does **not** predict the next regime.  
It answers: *"Under the current market environment, which EAs are relatively less fragile?"*

Phase 2 outputs **Exposure Weight** — a relative priority multiplier, not a lot size.

### 5.2 Market Regime Classification

Each EA is bound to a fixed `symbol` × `timeframe`. Regime is computed per symbol + timeframe, weekly.

```python
# --- Trend Score ---
# ADX(14) average over the rolling window (default: last 1 week; optional: 2–4 weeks)
TrendScore = Avg(ADX_14)           # industry standard
trend_label = 'trend' if TrendScore > 25 else 'range'

# --- Volatility Score ---
# Ratio of short-period ATR to long-period ATR (proposal standard)
VolatilityScore = Avg(ATR_14 / ATR_100)   # ATR(14) / ATR(100) per bar, averaged over window
vol_label = 'high' if VolatilityScore > 1.0 else 'low'   # ratio > 1 = elevated current vol

regime_label = f"{trend_label}_{vol_label}"  # → stored in Market_Regime_Weekly
```

> **Regime window**: Default = last **1 week**. Optional research window = 2–4 weeks for backtesting and score calibration — do not use for live allocation decisions.

| Regime | ADX | ATR | Compatible EA types |
|--------|-----|-----|---------------------|
| `trend_low` | > 25 | Low | Trend-following, MA crossover |
| `trend_high` | > 25 | High | Breakout EAs (tight SL required) |
| `range_low` | ≤ 25 | Low | Mean-reversion, grid EAs |
| `range_high` | ≤ 25 | High | ⚠️ Hardest regime — reduce all exposure |

> **v2.0 removal**: "Regime Stability Check" (assuming next week matches this week) is **eliminated** — the assumption is unreliable.

> **Note**: ADX measures trend *strength*, not direction. Add +DI / −DI if EA strategy depends on trend direction.

### 5.3 Regime Compatibility Score

For each EA, compare:
- Average `weekly_pnl_pct` in weeks where `regime_label = current_regime`
- Average `weekly_pnl_pct` across all regime labels

The ratio produces a Regime Compatibility score used only as input to Exposure Weight bucketing.

**Scope**: Computed for `active` EAs only. Soft Kill EAs skip Phase 2 and receive `exposure_weight = 0` (they are not traded regardless).

**Minimum data threshold**: 20 trades per regime bucket before the score is trusted.  
If insufficient: default to `neutral` → Exposure Weight = 1.0.

### 5.4 Exposure Weight Output

| Compatibility | Exposure Weight |
|---------------|-----------------|
| Strong | 1.3 |
| Neutral | 1.0 |
| Weak | 0.7 |
| Very Weak | 0.4 |

This weight is passed directly into Phase 3.

---

## 6. Phase 3 — Health Score & Capital Allocation

### 6.1 Objective

Phase 3 determines: *"How much capital should each active EA receive?"*

It consumes the Exposure Weight from Phase 2 and outputs a final lot multiplier.

> **Separation rule**: Phase 2 = regime-based logical priority. Phase 3 = capital magnitude based on full health. These must remain architecturally separate.

### 6.2 Health Score

**Range**: `[0, 100]`  
**Evaluated for**: Active EAs + Soft Kill EAs under observation  
**Excluded**: Hard Kill EAs

#### Component Weights (v2.0)

| Component | Weight | Lookback | Input |
|-----------|--------|----------|-------|
| Profit Factor (PF) | 25% | Last 2–4 weeks | `gross_pnl` (raw profit) |
| Recovery Factor (RF) | 25% | Full history | `net_profit / max_drawdown_abs` |
| Regime Compatibility (RC) | 20% | Current regime | Phase 2 output |
| Max Drawdown Safety (DD) | 15% | **Recent** (current drawdown) | Closed trades only |
| Equity Smoothness (ES) | 15% | Recent equity curve | R² on weekly PnL |

```
HealthScore = 0.25×PF_score + 0.25×RF_score + 0.20×RC_score + 0.15×DD_score + 0.15×ES_score
```

> **New EA guard**: If EA has < 30 trades, default PF and RF component scores to 50 (neutral) to prevent premature penalization due to insufficient data.

> **Soft Kill EAs**: These EAs skip Phase 2 (not traded), so they have no `exposure_weight` from Phase 2. For Health Score computation, RC_score defaults to **60** (Neutral) for Soft Kill EAs — they are monitored for recovery, not penalized further on RC.

> **v2.0 change**: Min-max normalization across EAs is replaced with fixed scoring tables — min-max was unstable week-to-week, outlier-sensitive, and produced opaque scores.

#### Fixed Scoring Tables

**Profit Factor → Score**

| PF | Score |
|----|-------|
| ≤ 1.0 | 30 |
| 1.2 | 50 |
| 1.5 | 70 |
| 2.0 | 85 |
| ≥ 3.0 | 100 |

**Recovery Factor → Score**

| RF | Score |
|----|-------|
| ≤ 0 | 20 |
| 0.5 | 40 |
| 1.0 | 60 |
| 1.5 | 80 |
| ≥ 2.0 | 100 |

**Regime Compatibility → Score**

| Compatibility | Score |
|---------------|-------|
| Very Weak | 20 |
| Weak | 40 |
| Neutral | 60 |
| Good | 80 |
| Strong | 100 |

**Max Drawdown → Safety Score** *(use recent/current drawdown — not lifetime max)*

| Max DD | Score |
|--------|-------|
| ≥ 40% | 10 |
| 30% | 30 |
| 20% | 50 |
| 10% | 80 |
| 0% | 100 |

**Equity Smoothness Score**

```
ES_score = R² × 100   (R² on last 8 weekly net_pnl data points)
```

### 6.3 Lot Multiplier

| Health Score | Lot Multiplier |
|-------------|----------------|
| 80–100 | 1.5 |
| 60–79 | 1.0 |
| 40–59 | 0.6 |
| 20–39 | 0.3 |
| < 20 | 0.0 (suspended this cycle) |

### 6.4 Final Lot Formula

```
FinalLot = BaseLot × ExposureWeight × HealthMultiplier
```

| Variable | Source | Range |
|----------|--------|-------|
| `BaseLot` | EA_Registry config | Fixed per EA |
| `ExposureWeight` | Phase 2 output | 0.4 – 1.3 |
| `HealthMultiplier` | Phase 3 output | 0.0 – 1.5 |

### 6.5 Risk Normalization (Mandatory)

Applied after all FinalLot values are computed:

```
MAX_PORTFOLIO_EXPOSURE = 5.0   ← default, operator-configurable via env

TotalExposure = SUM(FinalLot_i)  -- all active EAs

IF TotalExposure > MAX_PORTFOLIO_EXPOSURE:
    scale_factor = MAX_PORTFOLIO_EXPOSURE / TotalExposure
    FinalLot_i   = FinalLot_i × scale_factor
    -- Proportional scaling — relative weights are preserved
```

> Prevents the portfolio risk budget from being inadvertently breached when multiple EAs simultaneously score high.

---

## 7. Example Scenario

**Current regime**: `trend_high` (Trend × High Volatility)

| | EA_A | EA_B | EA_C |
|--|------|------|------|
| Trend compatibility | Strong | Weak | Neutral |
| Profit Factor (recent) | Moderate (PF 1.4) | High from prior weeks (PF 2.1) | Very low (PF 0.8) |
| Current Drawdown | 12% — stable | 18% — acceptable | 38% — severe |
| **Exposure Weight (Phase 2)** | **1.3** | **0.7** | **1.0** |
| **Health Score** | 72 | 68 | 14 |
| **Health Multiplier** | **1.0** | **1.0** | **0.0 (suspended)** |
| **FinalLot (pre-norm)** | `BaseLot × 1.30` | `BaseLot × 0.70` | **0** |

**Interpretation** (faithful to proposal.md §V example):
- **EA_A**: Regime-matched + decent health → exposure boosted to 1.3, lot multiplier = 1.0
- **EA_B**: Well-performing EA but poorly matched to current regime → exposure weight pulls it down to 0.7; health is acceptable but not exceptional (score 68 → multiplier 1.0), so final lot = `BaseLot × 0.70`, less than EA_A
- **EA_C**: Health Score < 20 → FinalLot = 0, suspended this cycle despite `active` state

**After Risk Normalization** (if `TotalExposure > MAX_PORTFOLIO_EXPOSURE = 5.0`):  
All FinalLot values are scaled down proportionally — relative weights between EAs are preserved.

---

## 8. Database — Full Schema

### 8.1 Tables

```sql
-- Master list of all EAs
CREATE TABLE EA_Registry (
  ea_code       VARCHAR(50) PRIMARY KEY,            -- e.g. 'EA_GOLD_001'
  login         INT NOT NULL,                        -- broker account
  symbol        VARCHAR(15) NOT NULL,                -- e.g. 'XAUUSD.std'
  base_symbol   VARCHAR(15) NOT NULL,                -- e.g. 'XAUUSD'
  timeframe     VARCHAR(10) NOT NULL,                -- e.g. 'H1', 'D1'
  base_lot      FLOAT DEFAULT 0.01,
  state         ENUM('active','soft_kill','hard_kill') DEFAULT 'active',
  created_at    DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at    DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_login (login),
  INDEX idx_state (state)
);

-- Source of truth: all broker trades (MQL4 format)
CREATE TABLE Orders (
  id            INT NOT NULL AUTO_INCREMENT,
  ticket        INT,
  orderType     INT,              -- 0=Buy, 1=Sell, 6=Balance (MQL4)
  symbol        VARCHAR(15),
  baseSymbol    VARCHAR(15),
  login         INT,              -- ⭐ links to EA_Registry
  broker        VARCHAR(50),
  magicNumber   INT,
  openTime      BIGINT,           -- Unix ms
  closeTime     BIGINT,           -- Unix ms; 0 = still open
  volume        FLOAT,            -- lot size
  openPrice     FLOAT,
  closePrice    FLOAT,
  TP            FLOAT,
  SL            FLOAT,
  commision     FLOAT,            -- ⚠️ 1 × 's'; always negative
  swap          FLOAT,
  profit        FLOAT,            -- ⭐ raw P&L (fees excluded)
  comment       VARCHAR(256),
  currency      VARCHAR(10),
  eaMagicNumber INT,
  PRIMARY KEY (id),
  UNIQUE KEY uq_ticket_broker (ticket, broker),
  INDEX idx_login_close      (login, closeTime),
  INDEX idx_login_type_close (login, orderType, closeTime)
);

-- Weekly analytics snapshot — one row per EA per week
CREATE TABLE EA_Weekly_Analytics (
  id               INT NOT NULL AUTO_INCREMENT,
  ea_code          VARCHAR(50) NOT NULL,
  week_start       DATE NOT NULL,
  trend_score      FLOAT,         -- Avg ADX(14) for the week
  vol_score        FLOAT,         -- Avg ATR(14)/ATR(100) for the week
  regime_label     VARCHAR(20),   -- 'trend_high' | 'trend_low' | 'range_high' | 'range_low'
  weekly_pnl_pct   FLOAT,         -- net P&L % (0 if no trades this week)
  drawdown_pct     FLOAT,         -- peak-to-trough on closed trades
  profit_factor    FLOAT,         -- gross_pnl based (Myfxbook standard)
  recovery_factor  FLOAT,         -- net_profit / max_drawdown_abs
  r_squared        FLOAT,         -- R² on equity curve
  ea_state         ENUM('active','soft_kill','hard_kill'),
  health_score     FLOAT,         -- [0, 100] — Phase 3 output
  exposure_weight  FLOAT,         -- Phase 2 output
  lot_multiplier   FLOAT,         -- Phase 3 final lot multiplier
  created_at       DATETIME DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uq_ea_week (ea_code, week_start),
  INDEX idx_ea_code   (ea_code),
  INDEX idx_week_start (week_start)
);

-- Audit trail of all EA state transitions
CREATE TABLE EA_State_Log (
  id            INT NOT NULL AUTO_INCREMENT,
  ea_code       VARCHAR(50) NOT NULL,
  old_state     ENUM('active','soft_kill','hard_kill'),
  new_state     ENUM('active','soft_kill','hard_kill') NOT NULL,
  trigger_rule  VARCHAR(20) NOT NULL,   -- 'HK1'|'HK2'|'SK1'|'SK2'|'SK3'|'SK4'|'REACTIVATE'|'MANUAL'
  trigger_value FLOAT,                  -- the metric value that tripped the rule
  changed_at    DATETIME DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  INDEX idx_ea_code    (ea_code),
  INDEX idx_changed_at (changed_at)
);

-- Centralized OHLC — source for ADX/ATR regime computation
CREATE TABLE Market_OHLC (
  id        INT NOT NULL AUTO_INCREMENT,
  symbol    VARCHAR(15) NOT NULL,
  timeframe VARCHAR(10) NOT NULL,
  bar_time  BIGINT NOT NULL,      -- Unix ms, open of bar
  open      FLOAT,
  high      FLOAT,
  low       FLOAT,
  close     FLOAT,
  volume    BIGINT,
  PRIMARY KEY (id),
  UNIQUE KEY uq_symbol_tf_time (symbol, timeframe, bar_time),
  INDEX idx_symbol_tf (symbol, timeframe, bar_time)
);

-- Pre-computed weekly regime label per symbol + timeframe
CREATE TABLE Market_Regime_Weekly (
  id           INT NOT NULL AUTO_INCREMENT,
  symbol       VARCHAR(15) NOT NULL,
  timeframe    VARCHAR(10) NOT NULL,
  week_start   DATE NOT NULL,
  trend_score  FLOAT,            -- Avg ADX(14)
  vol_score    FLOAT,            -- Avg ATR(14)/ATR(100)
  regime_label VARCHAR(20),      -- 'trend_high'|'trend_low'|'range_high'|'range_low'
  created_at   DATETIME DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uq_symbol_tf_week (symbol, timeframe, week_start)
);

-- Pipeline execution log
CREATE TABLE Pipeline_Log (
  id          INT NOT NULL AUTO_INCREMENT,
  phase       TINYINT NOT NULL,    -- 1 | 2 | 3
  status      ENUM('running','success','error') NOT NULL,
  ea_count    INT,
  started_at  DATETIME,
  finished_at DATETIME,
  error_msg   TEXT,
  PRIMARY KEY (id),
  INDEX idx_started_at (started_at)
);
```

### 8.2 Indexing Strategy

```sql
-- High-frequency analytics reads
CREATE INDEX idx_orders_login_close      ON Orders(login, closeTime);
CREATE INDEX idx_orders_login_type_close ON Orders(login, orderType, closeTime);
CREATE INDEX idx_analytics_ea_week       ON EA_Weekly_Analytics(ea_code, week_start);
CREATE INDEX idx_regime_symbol_tf_week   ON Market_Regime_Weekly(symbol, timeframe, week_start);
```

### 8.3 Data Retention Policy

| Table | Retention |
|-------|-----------|
| `Orders` | Indefinite — source of truth |
| `EA_Weekly_Analytics` | Indefinite — research + backtesting |
| `EA_State_Log` | Indefinite — compliance audit trail |
| `Market_OHLC` | 5 years rolling |
| `Pipeline_Log` | 90 days rolling |

---

## 9. Backend — API Service

### 9.1 Overview

The backend is a stateless REST API (Spring Boot 3 / Java 21) responsible for:
- Serving EA metrics and state to the admin interface
- Orchestrating the weekly Phase 1 → 2 → 3 pipeline via Spring Scheduler + Spring Batch
- Persisting all state transitions and analytics snapshots via Spring Data JPA
- Dispatching Telegram alerts on state changes and pipeline events

### 9.2 REST API Endpoints

#### EA Management

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/api/ea` | JWT | List all EAs — state + latest health score |
| `GET` | `/api/ea/:eaCode` | JWT | Full detail: state, Phase 2/3 outputs, scores |
| `GET` | `/api/ea/:eaCode/history` | JWT | Weekly analytics (paginated) |
| `GET` | `/api/ea/:eaCode/trades` | JWT | Raw closed trades from `Orders` |
| `PATCH` | `/api/ea/:eaCode/state` | JWT + admin | Manual state override |

#### Portfolio

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/api/portfolio/summary` | JWT | Active count, total lot exposure, avg health |
| `GET` | `/api/portfolio/lot-allocation` | JWT | FinalLot per EA |
| `GET` | `/api/portfolio/regime` | JWT | Current regime per symbol + timeframe |

#### Pipeline

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/api/pipeline/run` | JWT + operator | Manually trigger Phase 1 → 2 → 3 |
| `GET` | `/api/pipeline/status` | JWT | Last run result + timestamp |
| `GET` | `/api/pipeline/logs` | JWT | Execution log (paginated) |

#### Logs & Alerts

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/api/logs/state-changes` | JWT | EA state transition history |
| `GET` | `/api/logs/alerts` | JWT | All dispatched Telegram alerts |

#### Auth & Users

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/api/auth/login` | None | Exchange credentials for JWT |
| `POST` | `/api/auth/refresh` | Cookie | Refresh access token |
| `DELETE` | `/api/auth/session` | JWT | Logout — revoke refresh token |
| `GET` | `/api/users` | JWT + admin | List users |
| `POST` | `/api/users` | JWT + admin | Create user |
| `PATCH` | `/api/users/:id` | JWT + admin | Update role or password |

#### System

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/health` | None | DB + scheduler health check |

### 9.3 Authentication & Security

```
Admin UI   → Bearer JWT  (15-min access token + 7-day httpOnly refresh cookie)
MT4 Hook   → X-API-Key   (static, per-environment, rotatable)
Public     → None        (only /health and /api/auth/login are unauthenticated)
```

**Security measures**:
- JWT verified via Spring Security `JwtAuthenticationFilter` on every request
- Role-based access via `@PreAuthorize` on controller methods
- Rate limiting: Bucket4j (100 req/min public; 1000 req/min authenticated)
- Input validation: Bean Validation (`@Valid`) on all request DTOs
- SQL injection: Spring Data JPA / JPQL parameterized bindings — no raw string interpolation
- `login` field never returned in any API response — `eaCode` only
- Refresh tokens stored hashed (BCryptPasswordEncoder) in DB; revoked on logout

**Role matrix**:

| Role | Capabilities |
|------|-------------|
| `viewer` | Read-only: dashboard, EA list, EA detail, charts |
| `operator` | + Manual pipeline trigger, alert logs |
| `admin` | + Manual state override, user management |

### 9.4 Cron Schedule

| Job | Cron (UTC) | Action |
|-----|-----------|--------|
| Market Data | `0 22 * * *` | Pull OHLC, compute ADX/ATR |
| Phase 1 | `0 0 * * 0` | EA state evaluation |
| Phase 2 | `0 1 * * 1` | Regime scoring |
| Phase 3 | `0 2 * * 1` | Health score + lot allocation |

### 9.5 Error Handling

- All pipeline phases wrapped in try/catch; failures written to `Pipeline_Log`
- Phase failure does **not** reset lot allocations — fail-safe keeps last known good state
- Telegram alert dispatched immediately on any phase error
- All DB writes use `@Transactional`; partial writes are rolled back by Spring automatically

---

## 10. Web — Admin Interface

### 10.1 Dashboard Summary Cards

| Card | Metric shown |
|------|-------------|
| Active EAs | Count in `active` state |
| Soft Kill EAs | Count + reactivation countdown |
| Hard Kill EAs | Count (historical, read-only) |
| Total Portfolio Exposure | `SUM(FinalLot)` — all active EAs |
| Current Regime | Per-symbol: `XAUUSD → trend_high` |
| Last Pipeline Run | Timestamp + Pass / Fail |

### 10.2 EA List View

- Sortable table: EA Code, Symbol, State badge, Health Score (progress bar), Exposure Weight, Lot Multiplier, Final Lot
- State badges: 🟢 Active / 🟡 Soft Kill / 🔴 Hard Kill
- Filters: by state, symbol, health score range
- Export to CSV

### 10.3 EA Detail Page

Each EA has a dedicated detail page with tabs:

| Tab | Content |
|-----|---------|
| **Overview** | Radar chart: 5 Health Score components; current lot allocation |
| **Phase 1** | HK + SK rule results with actual trigger values |
| **Phase 2** | Current regime label, compatibility score, exposure weight |
| **Phase 3** | Component score table, health score, lot multiplier |
| **Trades** | Paginated closed trades (profit, commision, swap, closeTime) |
| **History** | Weekly charts: PnL %, drawdown %, health score (12 weeks) |
| **Logs** | EA_State_Log entries for this EA |

### 10.4 Portfolio View

- **Lot Allocation**: Horizontal bar chart — FinalLot per EA with `MAX_PORTFOLIO_EXPOSURE` limit line
- **Regime Map**: Symbol × Timeframe heatmap colored by current regime
- **Normalization Status**: Shows if scaling was applied this cycle and the scale_factor used

### 10.5 Charts & Visualizations

| Chart | Type | Description |
|-------|------|-------------|
| Health Score | Radar / Spider | 5 component scores |
| Weekly PnL % | Line | Last 12 weeks |
| Equity curve | Line + area fill | Cumulative net_pnl |
| Drawdown | Area (inverted) | Peak-to-trough weekly |
| Lot allocation | Horizontal bar | FinalLot vs MAX limit |
| Regime map | Heatmap grid | Symbol × Timeframe |

### 10.6 Real-Time Updates

- Dashboard + EA list: auto-refresh every **5 minutes** (polling)
- State change alerts: **WebSocket push via STOMP** → toast notification
- Pipeline progress: STOMP topics `/topic/pipeline.phase`, `/topic/pipeline.done`, `/topic/pipeline.error`

### 10.7 Authentication Flow

```
1. POST /api/auth/login → access token (15 min) + refresh token (httpOnly cookie)
2. All requests: Authorization: Bearer {access_token}
3. Token refresh: handled automatically by Axios interceptor
4. Logout: DELETE /api/auth/session → refresh token revoked server-side
```

### 10.8 Manual State Override

```
Admin → EA Detail → Actions → "Override State"
  → Modal: select target state + enter reason
  → PATCH /api/ea/{eaCode}/state
  → Backend: INSERT EA_State_Log (trigger_rule = 'MANUAL', trigger_value = NULL)
  → Telegram alert dispatched
  → STOMP broadcast pushes badge refresh to all connected tabs
```

> ⚠️ Hard Kill via manual override is irreversible — UI requires a second confirmation dialog.

### 10.9 Responsive Design & UX

- Minimum supported viewport: **1280px** (operator trading desk)
- Mobile: read-only dashboard summary (no pipeline control or override)
- **Dark mode default** — extended monitoring sessions

---

## 11. Primary Processing Flows

### 11.1 Phase 1 — EA State Evaluation

```
[Cron: Sunday 00:00 UTC]
       │
       ▼
SELECT * FROM EA_Registry WHERE state IN ('active', 'soft_kill')
       │
       ▼
For each EA:
  ├─ Query closed_orders (orderType IN (0,1), closeTime > 0)
  │
  ├─ Evaluate HARD KILL:
  │   ├─ HK1: max drawdown (full history) >= 75%?
  │   └─ HK2: major DD + insufficient recovery after 2× n trades?
  │
  ├─ IF HK triggered:
  │   ├─ UPDATE EA_Registry SET state = 'hard_kill'
  │   ├─ INSERT EA_State_Log (trigger_rule = 'HK1' | 'HK2', trigger_value = ...)
  │   ├─ FinalLot = 0
  │   └─ Telegram: "⛔ HARD KILL — {eaCode} [{rule}] triggered"
  │   SKIP to next EA
  │
  ├─ Evaluate SOFT KILL:
  │   ├─ SK1: current drawdown >= 20%?
  │   ├─ SK2: slope < 0 and |slope| > 2% balance? (skip if < 5 trades)
  │   ├─ SK3: consecutive loss drawdown >= 30%?
  │   └─ SK4: single trade loss >= 10% balance?
  │
  ├─ IF any SK triggered AND state != 'soft_kill':
  │   ├─ UPDATE EA_Registry SET state = 'soft_kill'
  │   ├─ INSERT EA_State_Log
  │   └─ Telegram: "🟡 SOFT KILL — {eaCode} [{rule}] triggered"
  │
  └─ IF state = 'soft_kill' AND all reactivation conditions pass:
      ├─ UPDATE EA_Registry SET state = 'active'
      ├─ INSERT EA_State_Log (trigger_rule = 'REACTIVATE')
      ├─ exposure_weight override = 0.5 (2-week grace period)
      └─ Telegram: "🟢 REACTIVATED — {eaCode}"
       │
       ▼
INSERT Pipeline_Log (phase = 1, status = 'success', ea_count = N)
```

### 11.2 Phase 2 — Regime Exposure

```
[Cron: Monday 01:00 UTC]
       │
       ▼
For each unique (symbol, timeframe):
  ├─ SELECT OHLC bars from Market_OHLC (last 7 days)
  ├─ Compute Avg ADX(14)      → trend_score
  ├─ Compute Avg ATR(14)/ATR(100) → vol_score
  ├─ trend_score > 25 → 'trend', else 'range'
  ├─ vol_score > 1.3  → 'high',  else 'low'
  ├─ regime_label = concat(trend, vol)
  └─ UPSERT Market_Regime_Weekly
       │
       ▼
For each EA WHERE state = 'active':
  ├─ Lookup regime_label for EA's (symbol, timeframe)
  ├─ Query EA_Weekly_Analytics:
  │     avg_this_regime = AVG(weekly_pnl_pct) WHERE regime_label = current
  │     avg_all_regimes = AVG(weekly_pnl_pct) -- all history
  │     IF trade count < 20 in current regime → use 'neutral' default
  ├─ Compute compatibility ratio = avg_this_regime / avg_all_regimes
  ├─ Map to bucket:
  │     Strong  → 1.3    Neutral → 1.0
  │     Weak    → 0.7    Very Weak → 0.4
  └─ Store exposure_weight in memory (consumed by Phase 3)
       │
       ▼
INSERT Pipeline_Log (phase = 2, status = 'success')
```

### 11.3 Phase 3 — Health Score & Lot Allocation

```
[Cron: Monday 02:00 UTC]
       │
       ▼
For each EA WHERE state IN ('active', 'soft_kill'):
  │
  ├─ Compute components:
  │   ├─ PF_score:  gross_pnl profit_factor (last 2–4 wks) → fixed table
  │   │             trades < 30 → default = 50
  │   ├─ RF_score:  net_profit / max_dd_abs (full history) → fixed table
  │   │             trades < 30 → default = 50
  │   ├─ RC_score:  exposure_weight from Phase 2 → map 0.4/0.7/1.0/1.3 → 20/40/60/80/100
  │   │             if state = soft_kill: RC_score = 60 (neutral, no Phase 2 output)
  │   ├─ DD_score:  current/recent drawdown_pct → fixed table  [NOT full history]
  │   └─ ES_score:  R² on recent equity curve weekly net_pnl points × 100
  │
  ├─ HealthScore = 0.25×PF + 0.25×RF + 0.20×RC + 0.15×DD + 0.15×ES
  │
  ├─ HealthMultiplier lookup:
  │     ≥80 → 1.5    60–79 → 1.0    40–59 → 0.6    20–39 → 0.3    <20 → 0.0
  │
  └─ FinalLot = BaseLot × ExposureWeight × HealthMultiplier
       │
       ▼
Risk Normalization:
  TotalExposure = SUM(FinalLot_i)
  IF TotalExposure > MAX_PORTFOLIO_EXPOSURE:
      FinalLot_i = FinalLot_i × (MAX_PORTFOLIO_EXPOSURE / TotalExposure)
       │
       ▼
Persist:
  INSERT EA_Weekly_Analytics (one row per EA)
  Sync lot_multiplier → MT4 bridge / broker API
       │
       ▼
INSERT Pipeline_Log (phase = 3, status = 'success')
Telegram: "✅ Weekly allocation complete — {N} active EAs, total_lot = {X.XX}"
```

---

## 12. System Startup Flow

```
[Server Boot]
       │
       ▼
Step 1: Load environment (dotenv)
  └─ Validate required: DB_HOST, JWT_SECRET, TELEGRAM_BOT_TOKEN
  └─ Fail fast with descriptive error if any required key is missing
       │
       ▼
Step 2: Database initialization
  └─ Flyway: run pending migrations (CREATE/ALTER tables)
  └─ Verify connection via Spring Data JPA health indicator
  └─ Retry policy: 3× with exponential backoff → hard crash on failure
       │
       ▼
Step 3: Seed check (first boot only)
  └─ IF EA_Registry is empty:
       → Log warning: "No EAs registered — pipeline auto-run is disabled"
       → System starts normally, REST API and WebSocket fully available
       → Cron jobs are registered but Phase 1/2/3 will produce empty results
       → Operator must INSERT into EA_Registry to enable pipeline execution
       │
       ▼
Step 4: Start HTTP server (Spring Boot embedded Tomcat)
  └─ Mount REST API controllers
  └─ Mount Spring WebSocket endpoint (STOMP)
  └─ Apply Spring Security filter chain
       │
       ▼
Step 5: Start job infrastructure
  ├─ Spring Batch JobLauncher initialized
  ├─ Redis connection verified (Lettuce)
  ├─ market-data-job     → OHLC ingestion
  ├─ phase-analytics-job → Phase 1 / 2 / 3 step execution
  └─ alert-job           → Telegram dispatch
       │
       ▼
Step 6: Register Spring Scheduled cron jobs
  ├─ 0 22 * * *   → market data sync (daily)
  ├─ 0  0 * * 0   → Phase 1 (Sunday)
  ├─ 0  1 * * 1   → Phase 2 (Monday)
  └─ 0  2 * * 1   → Phase 3 (Monday)
       │
       ▼
Step 7: Health endpoint live
  GET /health → { db: "ok", scheduler: "ok", uptime: {N}s }
       │
       ▼
Step 8: Ready — log
  "EA Portfolio System online. Active EAs: {N}. Next Phase 1: {sunday_iso}"
       │
       ▼
[Runtime]
  Spring Scheduler fires → Spring Batch job launched → step executes → result persisted via JPA
  Admin UI polls /api/ea every 5 min
  Spring WebSocket (STOMP) streams pipeline events to connected clients
```

### 12.1 Startup Failure Recovery Matrix

| Failure | Behavior |
|---------|----------|
| DB connection fails | Retry 3× (exponential backoff) → hard crash + error log |
| Flyway migration fails | Abort startup — never run on stale schema |
| EA_Registry empty | Warn operator — system starts, pipeline disabled |
| Telegram init fails | **Non-fatal** — log warning, system runs without alerts |
| OHLC/market data missing | Phase 2 skips that symbol, logs warning, uses last known regime |
| Python analytics error | Phase fails, `Pipeline_Log` records error, prior allocation preserved |

---

## 13. Key Decisions & Rationale

| Decision | Rationale |
|----------|-----------|
| SK3 + SK4: Hard Kill → Soft Kill | Both may be regime-mismatch events, not permanent structural failure |
| Min-max normalization removed | Unstable cross-week, outlier-sensitive, opaque to operators |
| Regime Stability Check removed | Week-to-week regime autocorrelation proved unreliable |
| Phase 2 and Phase 3 kept separate | Different concerns: regime priority (2) vs capital magnitude (3) |
| Drawdown on closed trades only | Systemic project design — scores read lower than Myfxbook (which includes floating P&L) |
| Fixed scoring tables | Explainable, stable, auditable without data normalization |
| New EA guard (< 30 trades) | Prevent premature penalization before meaningful statistical sample |
| SK2 skipped (< 5 trades) | 5-point regression is statistically meaningless; skip rather than false-trigger |
| `login` never in API | `eaCode` is the only public identifier — security and abstraction layer |
| Fail-safe on pipeline error | Phase error keeps last good allocation rather than zeroing out |

---

## 14. Limitations & Known Gaps

> Documented design tradeoffs, not bugs. Revisit as data accumulates.

| Area | Gap | Recommended Follow-up |
|------|-----|----------------------|
| Health Score weights | `0.25/0.25/0.20/0.15/0.15` are ad-hoc | Backtest to tune against historical EA performance outcomes |
| Regime Compatibility formula | Custom ratio, no industry benchmark | Validate: does high RC correlate with better subsequent weekly PnL? |
| Exposure Weight buckets | Strong/Neutral/Weak/Very Weak thresholds are heuristic | Backtest to verify bucket assignment improves risk-adjusted returns |
| Drawdown vs Myfxbook | Closed trades only → always reads lower than Myfxbook | Add floating P&L column when open-trade data becomes available |
| ADX lacks direction | ADX = trend strength, not up/down direction | Add +DI / −DI if EA strategy is direction-dependent |
| Survivorship bias in regime scoring | Hard Kill history excluded from regime-survival analysis | Re-include Hard Kill history when tuning regime thresholds |
| `MAX_PORTFOLIO_EXPOSURE = 5.0` | Reasonable default, not empirically calibrated | Tune against broker margin rules and total account equity |

---

## 15. SQL Reference — Myfxbook-Standard Metrics

> All queries use `profit` (gross, raw P&L) for win/loss classification.  
> All queries require the mandatory filter: `closeTime > 0 AND orderType IN (0, 1)`.

### Profit Factor
```sql
SELECT
  SUM(CASE WHEN profit > 0 THEN profit ELSE 0 END) /
  NULLIF(ABS(SUM(CASE WHEN profit < 0 THEN profit ELSE 0 END)), 0) AS profit_factor
FROM Orders
WHERE login = {login} AND closeTime > 0 AND orderType IN (0, 1);
```

### Win Rate
```sql
SELECT
  SUM(CASE WHEN profit > 0 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS win_rate
FROM Orders
WHERE login = {login} AND closeTime > 0 AND orderType IN (0, 1);
```

### Recovery Factor
```sql
WITH net AS (
  SELECT SUM(profit + COALESCE(commision,0) + COALESCE(swap,0)) AS net_profit
  FROM Orders
  WHERE login = {login} AND closeTime > 0 AND orderType IN (0, 1)
),
dd AS (
  SELECT MIN(cum_pnl - peak) AS max_dd_abs
  FROM (
    SELECT cum_pnl, MAX(cum_pnl) OVER (ORDER BY closeTime) AS peak
    FROM (
      SELECT SUM(profit + COALESCE(commision,0) + COALESCE(swap,0))
             OVER (ORDER BY closeTime) AS cum_pnl
      FROM Orders
      WHERE login = {login} AND closeTime > 0 AND orderType IN (0, 1)
    ) c
  ) p
)
SELECT net.net_profit / NULLIF(ABS(dd.max_dd_abs), 0) AS recovery_factor
FROM net, dd;
```

### Max Drawdown % (full history, closed trades)
```sql
WITH c AS (
  SELECT closeTime,
         SUM(profit + COALESCE(commision,0) + COALESCE(swap,0))
           OVER (ORDER BY closeTime) AS cum_pnl
  FROM Orders
  WHERE login = {login}
    AND closeTime > 0
    AND orderType IN (0, 1)
    AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 3000 DAY) * 1000
),
p AS (SELECT cum_pnl, MAX(cum_pnl) OVER (ORDER BY closeTime) AS peak FROM c)
SELECT
  MIN(cum_pnl - peak)                                      AS drawdown_abs,
  MIN((cum_pnl - peak) / ({initial_balance} + peak)) * 100 AS drawdown_pct
FROM p;
```

### Consecutive Losses (Loss Acceleration)
```sql
SELECT COUNT(*) AS consecutive_losses
FROM (
  SELECT profit,
         ROW_NUMBER() OVER (ORDER BY closeTime DESC) AS rn
  FROM Orders
  WHERE login = {login} AND closeTime > 0 AND orderType IN (0, 1)
) ranked
WHERE rn < (
  SELECT COALESCE(MIN(rn), 999)
  FROM (
    SELECT profit,
           ROW_NUMBER() OVER (ORDER BY closeTime DESC) AS rn
    FROM Orders
    WHERE login = {login} AND closeTime > 0 AND orderType IN (0, 1)
  ) t2
  WHERE profit > 0
);
```

---

*End of Document — EA Portfolio System Technical Proposal v2.0*
