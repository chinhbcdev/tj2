# EA Portfolio System — Technical Implementation Proposal
**Version:** 5.0 (Refactored ×3)  
**Stack:** Python 3.11 · MySQL 8 · MQL5  
**Data Source:** Bảng `Orders` duy nhất — không JOIN bảng gốc khác  
**Refactor log:** 20 bugs fixed — SQL, logic, pipeline (xem §0)

---

## Table of Contents

1. [System Architecture](#1-system-architecture)
2. [Data Source — Bảng `Orders`](#2-data-source--bảng-orders)
3. [New Tables to Create](#3-new-tables-to-create)
4. [Core SQL Formulas (Myfxbook Standard)](#4-core-sql-formulas-myfxbook-standard)
5. [Phase 1 — EA State Classification](#5-phase-1--ea-state-classification)
6. [Phase 2 — Regime Detection](#6-phase-2--regime-detection)
7. [Phase 3 — Health Score & Allocation](#7-phase-3--health-score--allocation)
8. [Weekly Execution Pipeline](#8-weekly-execution-pipeline)
9. [MQL5 Integration Contract](#9-mql5-integration-contract)
10. [Configuration](#10-configuration)
11. [Module Structure](#11-module-structure)
12. [Invariants & Constraints](#12-invariants--constraints)

---

## 0. Bug Fix Log (v4.0 → v5.0)

| # | Section | Bug | Fix |
|---|---|---|---|
| 1 | §4.4 | Max DD dùng `GREATEST(peak,0)` — lệch SKILL.md | Align exact: `initial_balance + peak` |
| 2 | §4.4 | Thừa `ROWS UNBOUNDED PRECEDING` | Xóa ROWS clause, dùng MySQL default |
| 3 | §4.5 | `ORDER BY (SELECT 1)` — undefined behavior | Đổi thành `ORDER BY closeTime, ticket` |
| 4 | §4.8 | "Last N Trades" query không có cum_pnl đúng | Xóa §4.8, luôn load full history rồi slice Python |
| 5 | §4.7 | ORDER BY closeTime không có tiebreaker | Thêm `, ticket ASC` |
| 6 | §4.6 | `week_end_utc` không định nghĩa | Document rõ = Monday 00:00:00 UTC tuần sau (exclusive) |
| 7 | global | MySQL session timezone không được set | Thêm `SET time_zone = '+00:00'` đầu connection |
| 8 | §7.2 | RF breakpoints cap ở 2.0, domain says RF > 3 là tốt | Thêm (3.0, 100), hạ (2.0, 90) |
| 9 | §5.4/5.6 | SK4 context ambiguous: full history vs last-10 | Tách rõ: classify=all_trades, reactivation=trades[-10:] |
| 10 | §8.2 | `prior_compat` default 'neutral' — quá rộng | Đổi default thành 'weak' |
| 11 | §8.2 | `load_p3_history()` tên sai, không join p2 | Đổi tên `load_regime_pnl_history()`, document JOIN p2+p3 |
| 12 | §7.4 | DD window 28-day vs 3000-day không giải thích | Thêm note intentional design |
| 13 | §5.2 | Formula string `max(peak,0)` lệch SKILL.md | Align: `initial_balance + peak` |
| 14 | §8.2 | `load_active_eas()` tên sai — bỏ sót soft_kill | Đổi tên `load_managed_eas()` |
| 15 | §3.2 | `ea_state_log` không có FK đến `ea_registry` | Thêm FK REFERENCES |
| 16 | §2.4 | Fallback initial balance "raise exception" crash pipeline | Đổi: skip EA + log warning |
| 17 | §8.2 | `update_p3_final_lot` scope không rõ | Document: chỉ nhận active logins |
| 18 | §9 | MQL5 export thiếu `base_lot` | Thêm vào payload |
| 19 | §6.1 | Không có fallback khi OHLCV feed down | Thêm rerun guard + fallback giữ regime tuần trước |
| 20 | §3.1 | `ea_registry.current_state` double-maintained không có update order | Document: insert_state_log → update_registry_state |

---

## 1. System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  SCHEDULER  (cron: Monday 00:05 UTC)                             │
└───────────────────────┬──────────────────────────────────────────┘
                        │
           ┌────────────▼────────────┐
           │  pipeline_runs          │  ← idempotency gate
           │  status = completed?    │    yes → no-op
           └────────────┬────────────┘
                        │
           ┌────────────▼────────────┐
           │  SET time_zone='+00:00' │  ← BẮT BUỘC đầu mỗi connection
           │  Market Data System     │    external OHLCV feed
           │  sync_regime_for_week() │    rerun guard + fallback nếu feed down
           └────────────┬────────────┘
                        │
     ┌──────────────────▼──────────────────────────┐
     │  For each login in load_managed_eas():       │
     │  (state IN 'active','soft_kill')             │
     │                                              │
     │  READ: Orders WHERE login={login}            │
     │    orderType=6         → initial_balance     │
     │    orderType IN (0,1)  → trades              │
     │                                              │
     │  ┌──────────┐                               │
     │  │ Phase 1  │  → state (LUÔN write P1)      │
     │  └────┬─────┘                               │
     │  hard_kill → write P1 + update state, return │
     │       │                                     │
     │  ┌────▼─────┐   ┌──────────┐               │
     │  │ Phase 2  │──▶│ Phase 3  │               │
     │  │ Regime   │   │ Health + │               │
     │  │ Compat.  │   │ pre_lot  │               │
     │  └──────────┘   └──────────┘               │
     └──────────────────────────────────────────────┘
                        │
           ┌────────────▼────────────┐
           │  normalize_portfolio()  │  ← Tier1: per-symbol cap
           │  [active EAs only]      │    Tier2: total cap
           └────────────┬────────────┘
                        │
           ┌────────────▼────────────┐
           │  MQL5 Bridge Export     │  ← atomic JSON write
           └─────────────────────────┘
```

**Data flow rules (bất biến):**
- Phase 2 đọc `ea_p2_weekly` + `ea_p3_weekly` của **tuần trước** (`week_start < current`).
- Phase 3 nhận `exposure_weight` qua argument — không re-query Phase 2.
- `normalize_portfolio()` chạy **sau** toàn bộ per-EA loop — không bao giờ trong loop.

---

## 2. Data Source — Bảng `Orders`

### 2.1 Schema (verified từ DB)

```sql
id            INT AUTO_INCREMENT PK
ticket        INT                      -- MT4 ticket number
orderType     INT                      -- 0=Buy, 1=Sell, 6=Balance
symbol        VARCHAR(15)              -- 'XAUUSD.std', 'XAUUSDm', ...
baseSymbol    VARCHAR(15)              -- 'XAUUSD', 'EURUSD', ...
login         INT                      -- ⭐ EA identifier
broker        VARCHAR(50)
magicNumber   INT
openTime      BIGINT                   -- Unix ms
closeTime     BIGINT                   -- Unix ms (0 = đang mở)
volume        FLOAT                    -- lot size
openPrice     FLOAT
closePrice    FLOAT
TP, SL        FLOAT
commision     FLOAT                    -- ⚠️ 1 chữ 's', luôn âm hoặc 0
swap          FLOAT
profit        FLOAT                    -- raw P&L chưa trừ phí
comment       VARCHAR(256)
currency      VARCHAR(10)
eaMagicNumber INT
```

**Indexes hiện có — không cần thêm:**
- `idx_Orders_login_closeTime` → (login, closeTime) ← query chính
- `idx_Orders_login` → (login)
- `idx_Orders_closeTime` → (closeTime)
- `idx_Orders_openTime` → (openTime)

---

### 2.2 orderType — MQL4 format

| orderType | Ý nghĩa | Xử lý |
|---|---|---|
| 0 | Buy market | ✅ Trade metrics |
| 1 | Sell market | ✅ Trade metrics |
| 6 | Balance/deposit (MQL4) | ✅ Initial balance |
| 2,3,4,5 | Pending orders | ❌ Filter out |
| 7 | Close-by | ❌ Filter out |

> ⚠️ MQL4 orderType=6 = Balance/nạp tiền. MQL5 type=6 = BUY_STOP_LIMIT — KHÁC NHAU. DB này là MQL4 format.

---

### 2.3 Hai loại P&L — Rule bất di bất dịch

```
gross_pnl = profit
            → Profit Factor, Win Rate  (Myfxbook: dùng raw profit)

net_pnl   = profit + COALESCE(commision, 0) + COALESCE(swap, 0)
            → Drawdown, Equity Curve, Weekly P&L
```

---

### 2.4 Initial Balance từ Orders

```sql
SELECT profit AS initial_balance
FROM Orders
WHERE login = {login} AND orderType = 6
ORDER BY openTime ASC
LIMIT 1;
```

**Fallback (BUG 16 fix):** Không có orderType=6 → **skip EA + log warning** — không raise exception, không crash pipeline. Một EA thiếu deposit record không được phép block toàn bộ run.

---

### 2.5 EA List

Operator khai báo login vào `ea_registry` (§3.1). Hệ thống chỉ xử lý subset này.

### 2.6 Timeframe

`Orders` không chứa timeframe. Operator khai báo khi đăng ký login vào `ea_registry`. Dùng cho Phase 2 (window_bars tính ADX/ATR).

---

## 3. New Tables to Create

> Tạo trong DB `EA`. Chỉ các bảng này được WRITE. `Orders` = READ ONLY tuyệt đối.

### 3.1 `ea_registry`

```sql
CREATE TABLE ea_registry (
    login         INT            NOT NULL PRIMARY KEY,   -- FK → Orders.login
    base_symbol   VARCHAR(15)    NOT NULL,
    timeframe     INT            NOT NULL,               -- MT4: 1,5,15,30,60,240,1440
    base_lot      DECIMAL(10,5)  NOT NULL,
    current_state VARCHAR(16)    NOT NULL DEFAULT 'active'
                  CHECK (current_state IN ('active','soft_kill','hard_kill')),
    created_at    DATETIME       NOT NULL DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_registry_state (current_state)
);

-- Operator insert thủ công:
-- INSERT INTO ea_registry (login, base_symbol, timeframe, base_lot)
-- VALUES (193583, 'XAUUSD', 60, 0.01);
```

**State update order (BUG 20 fix):** Khi state thay đổi, pipeline PHẢI theo thứ tự:
1. `INSERT INTO ea_state_log` (audit trail)
2. `UPDATE ea_registry SET current_state = ?` (source of truth)

Không được đảo thứ tự. `load_managed_eas()` đọc `ea_registry.current_state`.

---

### 3.2 `ea_state_log`

```sql
CREATE TABLE ea_state_log (
    id           BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
    login        INT          NOT NULL,
    from_state   VARCHAR(16),                           -- NULL = lần đầu assign
    to_state     VARCHAR(16)  NOT NULL
                 CHECK (to_state IN ('active','soft_kill','hard_kill')),
    triggered_by VARCHAR(16)  NOT NULL,                -- 'HK1'|'SK2'|...|'REACTIVATION'
    week_start   DATE         NOT NULL,
    run_id       BIGINT       NOT NULL,
    created_at   DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,

    -- BUG 15 fix: FK để tránh orphaned records
    CONSTRAINT fk_state_log_login
        FOREIGN KEY (login) REFERENCES ea_registry(login) ON DELETE RESTRICT,
    INDEX idx_state_log_login_week (login, week_start DESC)
);
```

---

### 3.3 `pipeline_runs`

```sql
CREATE TABLE pipeline_runs (
    id           BIGINT      NOT NULL AUTO_INCREMENT PRIMARY KEY,
    week_start   DATE        NOT NULL,
    status       VARCHAR(16) NOT NULL DEFAULT 'running'
                 CHECK (status IN ('running','completed','failed')),
    hk_eval_due  TINYINT(1)  NOT NULL DEFAULT 0,
    started_at   DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    completed_at DATETIME,
    error_detail TEXT,

    UNIQUE KEY uq_pipeline_week (week_start)
);
```

---

### 3.4 `pipeline_config`

```sql
CREATE TABLE pipeline_config (
    cfg_key    VARCHAR(64) NOT NULL PRIMARY KEY,
    cfg_value  TEXT        NOT NULL,
    updated_at DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP
                           ON UPDATE CURRENT_TIMESTAMP
);

INSERT INTO pipeline_config (cfg_key, cfg_value) VALUES ('last_hk_eval_week', '');
```

---

### 3.5 `regime_history`

```sql
CREATE TABLE regime_history (
    base_symbol      VARCHAR(15)   NOT NULL,
    timeframe        INT           NOT NULL,
    week_start       DATE          NOT NULL,
    trend_score      DECIMAL(8,4)  NOT NULL,    -- mean ADX(14)
    volatility_score DECIMAL(8,4)  NOT NULL,    -- mean ATR(14)/ATR(100)
    regime_label     VARCHAR(32)   NOT NULL,    -- 'trend_high_vol'|'trend_low_vol'
                                               -- |'range_high_vol'|'range_low_vol'
    bar_count        SMALLINT      NOT NULL,    -- 0 nếu fallback từ tuần trước

    PRIMARY KEY (base_symbol, timeframe, week_start)
);
```

---

### 3.6 `ea_p1_weekly`

```sql
CREATE TABLE ea_p1_weekly (
    login               INT          NOT NULL,
    week_start          DATE         NOT NULL,
    run_id              BIGINT       NOT NULL,
    resolved_state      VARCHAR(16)  NOT NULL,
    hk1_triggered       TINYINT(1)   NOT NULL DEFAULT 0,
    hk2_triggered       TINYINT(1)   NOT NULL DEFAULT 0,
    sk1_triggered       TINYINT(1)   NOT NULL DEFAULT 0,
    sk2_triggered       TINYINT(1)   NOT NULL DEFAULT 0,
    sk3_triggered       TINYINT(1)   NOT NULL DEFAULT 0,
    sk4_triggered       TINYINT(1)   NOT NULL DEFAULT 0,
    lifetime_max_dd_pct DECIMAL(8,4),
    current_dd_pct      DECIMAL(8,4),
    trade_count         INT,

    PRIMARY KEY (login, week_start),
    CONSTRAINT fk_p1_login FOREIGN KEY (login) REFERENCES ea_registry(login)
);
```

---

### 3.7 `ea_p2_weekly`

```sql
CREATE TABLE ea_p2_weekly (
    login                INT          NOT NULL,
    week_start           DATE         NOT NULL,
    run_id               BIGINT       NOT NULL,
    base_symbol          VARCHAR(15)  NOT NULL,
    timeframe            INT          NOT NULL,
    regime_label         VARCHAR(32)  NOT NULL,
    regime_compatibility VARCHAR(16)  NOT NULL,
    compatibility_basis  VARCHAR(20)  NOT NULL,   -- 'historical'|'insufficient_data'
    weeks_in_regime      SMALLINT     NOT NULL,
    exposure_weight      DECIMAL(6,4) NOT NULL,

    PRIMARY KEY (login, week_start),
    CONSTRAINT fk_p2_login FOREIGN KEY (login) REFERENCES ea_registry(login),
    INDEX idx_p2_login_week (login, week_start DESC)
);
```

---

### 3.8 `ea_p3_weekly`

```sql
CREATE TABLE ea_p3_weekly (
    login              INT            NOT NULL,
    week_start         DATE           NOT NULL,
    run_id             BIGINT         NOT NULL,
    -- Metrics thô (tính từ Orders; lưu để audit và backtesting)
    weekly_pnl_pct     DECIMAL(10,6)  NOT NULL,
    profit_factor      DECIMAL(8,4),              -- NULL nếu không có loss trong window
    recovery_factor    DECIMAL(8,4),              -- NULL nếu max_dd_abs = 0
    max_drawdown_pct   DECIMAL(8,4)   NOT NULL,   -- 28-day window (xem §7.4)
    equity_slope       DECIMAL(10,6)  NOT NULL,
    equity_r2          DECIMAL(6,4)   NOT NULL,
    -- Component scores [0–100]
    pf_score           DECIMAL(6,2)   NOT NULL,
    rf_score           DECIMAL(6,2)   NOT NULL,
    rc_score           DECIMAL(6,2)   NOT NULL,
    maxdd_score        DECIMAL(6,2)   NOT NULL,
    es_score           DECIMAL(6,2)   NOT NULL,
    -- Allocation
    health_score       DECIMAL(6,2)   NOT NULL,
    health_multiplier  DECIMAL(6,4)   NOT NULL,
    base_lot           DECIMAL(10,5)  NOT NULL,
    pre_norm_lot       DECIMAL(10,5)  NOT NULL,
    final_lot          DECIMAL(10,5)  NOT NULL,   -- 0.0 nếu soft_kill

    PRIMARY KEY (login, week_start),
    CONSTRAINT fk_p3_login FOREIGN KEY (login) REFERENCES ea_registry(login),
    INDEX idx_p3_login_week (login, week_start DESC)
);
```

---

## 4. Core SQL Formulas (Myfxbook Standard)

### Connection Setup — BẮT BUỘC

```sql
-- BUG 7 fix: UNIX_TIMESTAMP('{date_string}') phụ thuộc session timezone
-- Set UTC để window boundaries luôn chính xác
SET time_zone = '+00:00';
```

### Base Filter — MỌI query trade

```sql
WHERE login     = {login}
  AND closeTime > 0
  AND orderType IN (0, 1)
```

---

### 4.1 Initial Balance

```sql
SELECT profit AS initial_balance
FROM Orders
WHERE login = {login} AND orderType = 6
ORDER BY openTime ASC
LIMIT 1;
```

---

### 4.2 Profit Factor ✅

```sql
SELECT
    SUM(CASE WHEN profit > 0 THEN profit ELSE 0 END) /
    NULLIF(ABS(SUM(CASE WHEN profit < 0 THEN profit ELSE 0 END)), 0)
    AS profit_factor
FROM Orders
WHERE login     = {login}
  AND closeTime > 0
  AND orderType IN (0, 1)
  AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 28 DAY) * 1000;
-- Dùng gross profit (Myfxbook: không trừ phí)
```

---

### 4.3 Win Rate ✅

```sql
SELECT
    SUM(CASE WHEN profit > 0 THEN 1 ELSE 0 END) * 100.0 / COUNT(*)
    AS win_rate_pct
FROM Orders
WHERE login     = {login}
  AND closeTime > 0
  AND orderType IN (0, 1)
  AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 28 DAY) * 1000;
-- Dùng gross profit (Myfxbook: không trừ phí)
```

---

### 4.4 Max Drawdown % ✅ (aligned exact với SKILL.md)

```sql
-- BUG 1+2 fix: dùng `initial_balance + peak` (không GREATEST), không có ROWS clause
-- BUG 5 fix: tiebreaker ticket trong window ORDER BY
WITH c AS (
    SELECT
        closeTime, ticket,
        SUM(profit + COALESCE(commision,0) + COALESCE(swap,0))
            OVER (ORDER BY closeTime, ticket) AS cum_pnl
    FROM Orders
    WHERE login     = {login}
      AND closeTime > 0
      AND orderType IN (0, 1)
      AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 3000 DAY) * 1000
),
p AS (
    SELECT
        cum_pnl,
        MAX(cum_pnl) OVER (ORDER BY closeTime, ticket) AS peak
    FROM c
)
SELECT
    MIN(cum_pnl - peak)                                    AS max_dd_abs,
    MIN((cum_pnl - peak) / ({initial_balance} + peak)) * 100 AS max_dd_pct
FROM p;

-- max_dd_abs: giá trị âm (currency). max_dd_pct: giá trị âm (%).
-- {initial_balance} = kết quả §4.1
-- Window 3000 DAY = toàn bộ history (kill rules & RF)
```

---

### 4.5 Current Drawdown % (SK1)

```sql
-- BUG 3 fix: ORDER BY closeTime, ticket — không dùng ORDER BY (SELECT 1)
WITH c AS (
    SELECT
        closeTime, ticket,
        SUM(profit + COALESCE(commision,0) + COALESCE(swap,0))
            OVER (ORDER BY closeTime, ticket) AS cum_pnl
    FROM Orders
    WHERE login     = {login}
      AND closeTime > 0
      AND orderType IN (0, 1)
      AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 3000 DAY) * 1000
),
p AS (
    SELECT
        closeTime, ticket, cum_pnl,
        MAX(cum_pnl) OVER (ORDER BY closeTime, ticket) AS running_peak
    FROM c
)
SELECT
    (running_peak - cum_pnl) /
    NULLIF({initial_balance} + running_peak, 0) * 100 AS current_dd_pct
FROM p
ORDER BY closeTime DESC, ticket DESC
LIMIT 1;
```

---

### 4.6 Weekly P&L % (Phase 2)

```sql
-- BUG 6 fix: week_end = Monday 00:00:00 UTC tuần sau (exclusive <)
SELECT
    SUM(profit + COALESCE(commision,0) + COALESCE(swap,0)) AS weekly_net_pnl
FROM Orders
WHERE login     = {login}
  AND closeTime > 0
  AND orderType IN (0, 1)
  AND closeTime >= UNIX_TIMESTAMP('{this_monday_00:00:00_utc}') * 1000
  AND closeTime <  UNIX_TIMESTAMP('{next_monday_00:00:00_utc}') * 1000;

-- weekly_pnl_pct = weekly_net_pnl / initial_balance  [Python]
-- Không có trade trong tuần → weekly_pnl_pct = 0.0
```

---

### 4.7 Full Trade History — Nguồn duy nhất cho Python

```sql
-- BUG 4 fix: xóa §4.8 "Last N Trades". Luôn load full history. Slice trong Python.
-- BUG 5 fix: tiebreaker ticket ASC
SELECT
    ticket,
    closeTime,
    profit,
    COALESCE(commision, 0)                                          AS commision,
    COALESCE(swap, 0)                                               AS swap,
    profit + COALESCE(commision,0) + COALESCE(swap,0)              AS net_pnl,
    SUM(profit + COALESCE(commision,0) + COALESCE(swap,0))
        OVER (ORDER BY closeTime ASC, ticket ASC)                   AS cum_pnl
FROM Orders
WHERE login     = {login}
  AND closeTime > 0
  AND orderType IN (0, 1)
ORDER BY closeTime ASC, ticket ASC;
```

**Python slicing (sau load):**
```python
all_trades = db.load_trades(login)   # chronological, cum_pnl đúng từ DB
last_5     = all_trades[-5:]         # SK2 lookback
last_10    = all_trades[-10:]        # SK4 reactivation check
# cum_pnl đã chính xác vì tính trên TOÀN BỘ history trong window function
```

---

## 5. Phase 1 — EA State Classification

**Module:** `src/phase1/`

### 5.1 Input Models

```python
@dataclass(frozen=True)
class TradeRow:
    ticket:     int
    close_time: int       # Unix ms
    profit:     float     # gross — win/loss phân loại
    commision:  float     # 1 chữ 's', luôn <= 0
    swap:       float
    net_pnl:    float     # = profit + commision + swap
    cum_pnl:    float     # tích lũy net_pnl từ trade đầu tiên (đúng từ DB §4.7)

@dataclass(frozen=True)
class EAContext:
    login:           int
    base_symbol:     str
    timeframe:       int
    base_lot:        float
    initial_balance: float   # từ §4.1
    current_state:   str     # từ ea_registry.current_state
```

---

### 5.2 Drawdown Utilities (`src/utils/drawdown.py`)

```python
def compute_max_drawdown(trades: list[TradeRow], initial_balance: float) -> float:
    """
    Lifetime max drawdown — aligned SKILL.md (BUG 13 fix).
    Tại mỗi trade i:
      peak[i] = max(cum_pnl[0..i])
      dd[i]   = (cum_pnl[i] - peak[i]) / (initial_balance + peak[i])
    Return: abs(MIN(dd[i]))  — magnitude dương, fraction [0, 1]
    Mẫu số = initial_balance + peak (không có max/GREATEST)
    """
    if not trades:
        return 0.0
    max_dd = 0.0
    peak   = trades[0].cum_pnl
    for t in trades:
        if t.cum_pnl > peak:
            peak = t.cum_pnl
        denom = initial_balance + peak
        if denom > 0:
            dd = (t.cum_pnl - peak) / denom
            max_dd = min(max_dd, dd)
    return abs(max_dd)


def compute_current_drawdown(trades: list[TradeRow], initial_balance: float) -> float:
    """
    (running_peak - last_cum_pnl) / (initial_balance + running_peak)
    Dùng net_pnl (cum_pnl field).
    """
    if not trades:
        return 0.0
    peak     = max(t.cum_pnl for t in trades)
    last_cum = trades[-1].cum_pnl
    denom    = initial_balance + peak
    return max(0.0, (peak - last_cum) / denom) if denom > 0 else 0.0
```

---

### 5.3 Hard Kill Rules

#### HK1 — Extreme Lifetime Drawdown
```python
def check_hk1(trades: list[TradeRow], initial_balance: float,
              threshold: float = 0.75) -> bool:
    """Max drawdown lifetime >= 75%. Net_pnl via cum_pnl."""
    return compute_max_drawdown(trades, initial_balance) >= threshold
```

#### HK2 — Failed Recovery After Major Drawdown
```python
def check_hk2(
    trades:            list[TradeRow],   # chronological, cum_pnl đúng
    initial_balance:   float,
    min_dd_pct:        float = 0.50,
    trade_count_ratio: int   = 2,
    min_recovery_pct:  float = 0.50,
) -> bool:
    """
    1. Build balance series:
       b[i] = initial_balance + trades[i].cum_pnl  (i = 0..N-1)

    2. Tìm deepest drawdown event >= min_dd_pct:
       Scan: peak_val = running max của b[0..i]
       dd_pct[i] = (peak_val - b[i]) / peak_val  (nếu peak_val > 0)
       Event = (peak_idx, trough_idx) với dd_pct lớn nhất >= 0.50
       Không có event nào → return False

    3. n = trough_idx - peak_idx  (trades từ peak đến trough)
    4. m = N - 1 - trough_idx     (trades sau trough)
       m <= trade_count_ratio * n → return False (chưa đủ thời gian)

    5. dd_abs = b[peak_balance] - b[trough_idx]
       recovery_ratio = max(0, b[-1] - b[trough_idx]) / dd_abs
       Trigger nếu recovery_ratio < min_recovery_pct
    """
```

---

### 5.4 Soft Kill Rules

#### SK1 — Current Drawdown >= 20%
```python
def check_sk1(trades: list[TradeRow], initial_balance: float,
              threshold: float = 0.20) -> bool:
    return compute_current_drawdown(trades, initial_balance) >= threshold
```

#### SK2 — Negative Short-Term Equity Slope
```python
def check_sk2(trades: list[TradeRow], initial_balance: float,
              lookback: int = 5, min_slope: float = 0.02) -> bool:
    """
    Dùng all_trades[-lookback:] (slice Python từ full history).
    cum_pnl đã chính xác vì lấy từ §4.7.

    Normalized equity: y[i] = (initial_balance + t.cum_pnl) / initial_balance
    OLS fit: y = A*x + B  (x = 0..lookback-1)
    Slope A: fraction/trade (dimensionless sau normalization)
    Trigger nếu A < 0 AND abs(A) >= min_slope
    """
    if len(trades) < lookback:
        return False
    recent    = trades[-lookback:]
    x         = list(range(lookback))
    y         = [(initial_balance + t.cum_pnl) / initial_balance for t in recent]
    n         = lookback
    xm, ym    = sum(x)/n, sum(y)/n
    ss_xy     = sum((x[i]-xm)*(y[i]-ym) for i in range(n))
    ss_xx     = sum((x[i]-xm)**2 for i in range(n))
    if ss_xx == 0:
        return False
    slope = ss_xy / ss_xx
    return slope < 0 and abs(slope) >= min_slope
```

#### SK3 — Trailing Consecutive Loss Streak >= 30%
```python
def check_sk3(trades: list[TradeRow], initial_balance: float,
              threshold: float = 0.30) -> bool:
    """
    Walk backward từ trade cuối.
    Collect trailing consecutive trades với profit < 0 (gross).
    Stop tại trade đầu tiên có profit >= 0.

    streak_idx = index trong all_trades của trade đầu tiên trong streak.
    balance_before_streak:
      streak_idx == 0 → initial_balance
      streak_idx > 0  → initial_balance + trades[streak_idx - 1].cum_pnl

    streak_loss = SUM(net_pnl của streak trades)  [âm]
    streak_dd   = abs(streak_loss) / balance_before_streak
    Trigger nếu streak_dd >= 0.30
    """
    if not trades:
        return False
    streak = []
    for t in reversed(trades):
        if t.profit < 0:
            streak.append(t)
        else:
            break
    if not streak:
        return False
    streak.reverse()   # chronological
    streak_start_idx = len(trades) - len(streak)
    bal_before = (initial_balance if streak_start_idx == 0
                  else initial_balance + trades[streak_start_idx - 1].cum_pnl)
    if bal_before <= 0:
        return False
    streak_loss = sum(t.net_pnl for t in streak)
    return abs(streak_loss) / bal_before >= threshold
```

#### SK4 — Single Large Loss (BUG 9 fix: caller kiểm soát scope)
```python
def check_sk4(trades: list[TradeRow], initial_balance: float,
              threshold: float = 0.10) -> bool:
    """
    Duyệt danh sách trades được truyền vào.
    balance_before[0] = initial_balance
    balance_before[i] = initial_balance + trades[i-1].cum_pnl  (i > 0)

    Trigger nếu có trade: abs(net_pnl) / balance_before >= 0.10

    Trong classify_ea:       gọi check_sk4(all_trades, ...)  ← full history
    Trong check_reactivation: gọi check_sk4(trades[-10:], ...)  ← last-10 only
    CALLER quyết định scope — hàm không tự slice.
    """
    for i, t in enumerate(trades):
        bal_before = initial_balance if i == 0 else initial_balance + trades[i-1].cum_pnl
        if bal_before > 0 and abs(t.net_pnl) / bal_before >= threshold:
            return True
    return False
```

---

### 5.5 State Resolution

```python
def classify_ea(ctx: EAContext, trades: list[TradeRow],
                run_hk_eval: bool, cfg: Config) -> Phase1Result:
    """
    Priority: hard_kill > soft_kill > active.
    Pipeline kiểm tra ea_state_log TRƯỚC — nếu đã có HK entry → không gọi hàm này.

    SK4: check_sk4(trades, ...) với full history (all_trades) — BUG 9 fix.
    """
    hk1 = hk2 = False
    if run_hk_eval:
        hk1 = check_hk1(trades, ctx.initial_balance, cfg.HK1_MAX_DD_PCT)
        hk2 = check_hk2(trades, ctx.initial_balance, cfg.HK2_MIN_DD_PCT,
                         cfg.HK2_TRADE_COUNT_RATIO, cfg.HK2_MIN_DD_RECOVER_PCT)

    sk1 = check_sk1(trades, ctx.initial_balance, cfg.SK1_MAX_DD_PCT)
    sk2 = check_sk2(trades, ctx.initial_balance,
                    cfg.SK2_LOOKBACK_TRADES, cfg.SK2_MIN_SLOPE)
    sk3 = check_sk3(trades, ctx.initial_balance, cfg.SK3_DD_MAX)
    sk4 = check_sk4(trades, ctx.initial_balance, cfg.SK4_MAX_LOSS_PCT)  # full history

    triggered = [r for r, f in [('HK1',hk1),('HK2',hk2),('SK1',sk1),
                                  ('SK2',sk2),('SK3',sk3),('SK4',sk4)] if f]
    if hk1 or hk2:           state = 'hard_kill'
    elif sk1 or sk2 or sk3 or sk4: state = 'soft_kill'
    else:                    state = 'active'

    return Phase1Result(
        resolved_state=state, triggered_rules=triggered,
        hk1_triggered=hk1, hk2_triggered=hk2,
        sk1_triggered=sk1, sk2_triggered=sk2,
        sk3_triggered=sk3, sk4_triggered=sk4,
        lifetime_max_dd=compute_max_drawdown(trades, ctx.initial_balance),
        current_dd=compute_current_drawdown(trades, ctx.initial_balance),
        trade_count=len(trades),
    )
```

---

### 5.6 Soft Kill Reactivation

```python
def check_reactivation(trades: list[TradeRow], initial_balance: float,
                        regime_compatibility: str, cfg: Config) -> bool:
    """
    Trở về Active nếu TẤT CẢ điều kiện đúng:
    1. SK1 không trigger
    2. SK2 không trigger
    3. SK3 không trigger
    4. SK4 không trigger trong 10 trade gần nhất  (BUG 9 fix: trades[-10:])
    5. regime_compatibility != 'very_weak'
    """
    return (
        not check_sk1(trades, initial_balance, cfg.SK1_MAX_DD_PCT) and
        not check_sk2(trades, initial_balance,
                      cfg.SK2_LOOKBACK_TRADES, cfg.SK2_MIN_SLOPE) and
        not check_sk3(trades, initial_balance, cfg.SK3_DD_MAX) and
        not check_sk4(trades[-10:], initial_balance, cfg.SK4_MAX_LOSS_PCT) and
        regime_compatibility != 'very_weak'
    )
```

---

## 6. Phase 2 — Regime Detection

**Module:** `src/phase2/`

### 6.1 OHLCV Source & Fallback (BUG 19 fix)

```python
def sync_regime_for_week(db, mds, base_symbol: str, tf: int, week_start: date):
    """
    Rerun guard: nếu regime_history đã có (base_symbol, tf, week_start) → skip.
    Nếu chưa có → fetch từ external feed.
    Nếu fetch thất bại:
      - Copy regime tuần trước với bar_count=0 (đánh dấu fallback)
      - Log warning — không crash pipeline
      - EAs vẫn được xử lý với regime fallback này
    """
```

Tất cả EA cùng `(base_symbol, timeframe)` **chia sẻ 1 row** trong `regime_history` — không duplicate per login.

---

### 6.2 Timeframe Constants (MT4)

```python
TIMEFRAME_MINUTES = {1:1, 5:5, 15:15, 30:30, 60:60, 240:240, 1440:1440}

def bars_in_week(tf_mt4: int) -> int:
    return (5 * 24 * 60) // TIMEFRAME_MINUTES[tf_mt4]
```

---

### 6.3 Regime Classification

```python
def classify_regime(df: pd.DataFrame, tf_mt4: int,
                    adx_threshold: float = 25.0,
                    vol_threshold:  float = 1.0) -> tuple:
    """
    Requires >= 100 bars. Raise ValueError nếu không đủ.
    trend_score      = mean(ADX_14[-window_bars:])
    volatility_score = mean(ATR_14[-window_bars:] / ATR_100[-window_bars:])
    regime: (trending, high_vol) → 1 trong 4 labels

    ⚠️ 25.0 và 1.0 là defaults chưa calibrate.
    Calibrate per symbol + lưu vào pipeline_config trước production.
    """
```

---

### 6.4 Regime Compatibility

```python
def compute_regime_compatibility(
    login:          int,
    current_regime: str,
    history:        list[dict],   # {regime_label, weekly_pnl_pct} từ load_regime_pnl_history
    min_weeks:      int = 4,
) -> tuple[str, str, int]:
    """
    Additive comparison — không dùng ratio (broken khi avg_pnl < 0):
      score = mean(pnl trong current_regime) - mean(pnl tất cả regimes)

    BUG 10 fix: < 4 tuần → 'weak','insufficient_data' (không phải 'neutral')
    Thresholds (weekly pnl fraction difference):
      >= +0.005 → 'strong'    weight 1.3
      >= +0.002 → 'good'      weight 1.0
      >= -0.002 → 'neutral'   weight 1.0
      >= -0.005 → 'weak'      weight 0.7
      <  -0.005 → 'very_weak' weight 0.4

    history từ load_regime_pnl_history():  (BUG 11 fix)
      SELECT p2.regime_label, p3.weekly_pnl_pct
      FROM ea_p2_weekly p2 JOIN ea_p3_weekly p3 USING (login, week_start)
      WHERE p2.login = {login} AND p2.week_start < {current_week_start}
    """

EXPOSURE_WEIGHT = {
    'strong':1.3, 'good':1.0, 'neutral':1.0, 'weak':0.7, 'very_weak':0.4
}
```

---

## 7. Phase 3 — Health Score & Allocation

**Module:** `src/phase3/`

**DD window — intentional design (BUG 12 documented):**
- Kill rules (HK1, SK1): **3000 DAY** = lifetime risk — bắt danger tuyệt đối
- Health Score (§7.4): **28 DAY** = recent performance — scoring health gần đây
- Hai metrics khác nhau, phục vụ mục đích khác nhau. Không nhầm.

Window cho Health Score: **28 ngày** closed trades.

---

### 7.1 Profit Factor Score (weight 25%)

```python
PF_BP = [(1.0, 30), (1.2, 50), (1.5, 70), (2.0, 85), (3.0, 100)]
# Gross profit. NULL → score = 30 (floor)
# PF<1.0=lỗ | 1.0–1.4=biên mỏng | 1.4–2.0=tốt | >2.0=xuất sắc
```

---

### 7.2 Recovery Factor Score (weight 25%)

```python
# BUG 8 fix: (2.0, 90) + (3.0, 100) — domain knowledge: RF tốt > 3
RF_BP = [(0.0, 20), (0.5, 40), (1.0, 60), (1.5, 80), (2.0, 90), (3.0, 100)]
# RF = net_profit_lifetime / |max_dd_abs_lifetime|
# NULL (max_dd=0) → score = 20
```

---

### 7.3 Regime Compatibility Score (weight 20%)

```python
RC_MAP = {'very_weak':20, 'weak':40, 'neutral':60, 'good':80, 'strong':100}
```

---

### 7.4 Max Drawdown Safety Score (weight 15%)

```python
# 28-day window (KHÁC kill rules 3000-day — intentional, xem note đầu §7)
MAXDD_BP = [(0.00, 100), (0.10, 80), (0.20, 50), (0.30, 30), (0.40, 10)]
# DD<20%=acceptable | DD>25%=danger zone
```

---

### 7.5 Equity Smoothness Score (weight 15%)

```python
# y[i] = (initial_balance + cum_pnl[i]) / initial_balance
# OLS → slope A, R²
# slope <= 0 → es_score = 0  (declining equity không được thưởng smoothness)
# slope > 0  → es_score = 100 * R²
# Cần >= 3 trades trong 28-day window
```

---

### 7.6 Health Score

```python
health_score = round(
    cfg.HEALTH_WEIGHT_PF    * pf_score    +
    cfg.HEALTH_WEIGHT_RF    * rf_score    +
    cfg.HEALTH_WEIGHT_RC    * rc_score    +
    cfg.HEALTH_WEIGHT_MAXDD * maxdd_score +
    cfg.HEALTH_WEIGHT_ES    * es_score,
    2
)
# Range [0, 100]. Fixed scoring — không min-max normalize.
```

---

### 7.7 Health Multiplier

```python
_THRESHOLDS = [(80, 1.5), (60, 1.0), (40, 0.6), (20, 0.3)]

def get_health_multiplier(score: float) -> float:
    for t, m in _THRESHOLDS:
        if score >= t: return m
    return 0.1   # floor: không bao giờ 0.0 cho active EA

# final_lot = 0.0 cho soft_kill được set EXPLICIT trong pipeline.
```

---

### 7.8 Final Lot & Normalization

```python
pre_norm_lot = base_lot * exposure_weight * health_multiplier

def normalize_portfolio(
    active_lots:         dict[int, float],   # BUG 17: active logins ONLY
    login_to_symbol:     dict[int, str],
    max_total_lots:      float,
    max_lots_per_symbol: float,
) -> dict[int, float]:
    """
    Tier 1: Per-symbol cap → scale down nếu sum(lots on symbol) > max
    Tier 2: Total cap → scale down nếu sum(all) > max
    Soft_kill EAs đã có final_lot=0.0 từ upsert_p3_weekly — không truyền vào đây.
    """
```

---

## 8. Weekly Execution Pipeline

**Module:** `src/pipeline/run_weekly.py`

### 8.1 HK Schedule

```python
def is_hk_eval_due(db, week_start: date) -> bool:
    last = db.get_config('last_hk_eval_week')
    if not last: return True
    return (week_start - date.fromisoformat(last)).days >= 21
```

### 8.2 Main Pipeline

```python
def run_weekly_pipeline(week_start: date, db, mds) -> None:
    """
    Idempotent: UNIQUE (week_start) trên pipeline_runs.
    Nếu completed → no-op.
    """
    run = db.get_pipeline_run(week_start)
    if run and run.status == 'completed':
        return

    run_id = db.upsert_pipeline_run(week_start, 'running')
    hk_due = is_hk_eval_due(db, week_start)

    try:
        # Step 1: Regime (rerun guard + fallback built-in)
        for base_symbol, tf in db.get_distinct_symbol_timeframes():
            sync_regime_for_week(db, mds, base_symbol, tf, week_start)

        # Step 2: Per-EA loop
        # BUG 14 fix: load_managed_eas = WHERE state IN ('active','soft_kill')
        active_lots   = {}   # login → pre_norm_lot (active only)
        login_to_sym  = {}   # login → base_symbol

        for ea in db.load_managed_eas():
            _process_ea(ea, week_start, run_id, hk_due, db,
                        active_lots, login_to_sym)

        # Step 3: Normalization — active EAs only (BUG 17)
        normalized = normalize_portfolio(
            active_lots, login_to_sym,
            cfg.MAX_TOTAL_LOTS, cfg.MAX_LOTS_PER_SYMBOL
        )
        for login, final_lot in normalized.items():
            db.update_p3_final_lot(login, week_start, final_lot)
        # soft_kill final_lot=0.0 đã được set trong upsert_p3_weekly — không cần update

        # Step 4: MQL5 export
        export_lot_assignments(db.load_p3_weekly(week_start), week_start)

        # Step 5: Finalize
        if hk_due:
            db.set_config('last_hk_eval_week', week_start.isoformat())
        db.complete_pipeline_run(run_id)

    except Exception as e:
        db.fail_pipeline_run(run_id, str(e))
        raise


def _process_ea(ea, week_start, run_id, hk_due, db, active_lots, login_to_sym):
    """
    Mỗi EA trong 1 MySQL transaction.

    Write order (invariant):
      1. Phase 1 — LUÔN write (kể cả hard_kill)
      2. insert_state_log → update_registry_state  (BUG 20: thứ tự này)
      3. hard_kill → return
      4. Reactivation check (soft_kill EAs)
      5. Phase 2
      6. Phase 3 (health luôn tính; final_lot=0 nếu soft_kill)
    """
    with db.transaction():
        # BUG 16 fix: skip không crash nếu thiếu initial_balance
        initial_bal = db.get_initial_balance(ea.login)
        if initial_bal is None:
            db.log_pipeline_warning(run_id, ea.login,
                'No orderType=6 — EA skipped')
            return

        trades = db.load_trades(ea.login)   # §4.7, full history

        # Hard Kill permanent check
        if db.has_hard_kill_entry(ea.login):
            return

        # Phase 1
        p1 = classify_ea(ea, trades, hk_due, cfg)
        db.upsert_p1_weekly(ea.login, week_start, run_id, p1)

        if p1.resolved_state != ea.current_state:
            # BUG 20: log trước, update registry sau
            db.insert_state_log(
                ea.login, ea.current_state, p1.resolved_state,
                p1.triggered_rules[0] if p1.triggered_rules else 'SYSTEM',
                week_start, run_id
            )
            db.update_registry_state(ea.login, p1.resolved_state)

        if p1.resolved_state == 'hard_kill':
            return

        # Reactivation
        last_p2      = db.load_last_p2(ea.login, before=week_start)
        # BUG 10 fix: default 'weak' (không phải 'neutral')
        prior_compat = last_p2.regime_compatibility if last_p2 else 'weak'
        current_state = p1.resolved_state

        if ea.current_state == 'soft_kill' and current_state == 'soft_kill':
            if check_reactivation(trades, initial_bal, prior_compat, cfg):
                db.insert_state_log(
                    ea.login, 'soft_kill', 'active',
                    'REACTIVATION', week_start, run_id
                )
                db.update_registry_state(ea.login, 'active')
                current_state = 'active'

        # Phase 2
        regime  = db.load_regime_history(ea.base_symbol, ea.timeframe, week_start)
        # BUG 11 fix: load_regime_pnl_history (JOIN p2+p3, week < current)
        history = db.load_regime_pnl_history(ea.login, before=week_start)
        compat, basis, n_weeks = compute_regime_compatibility(
            ea.login, regime.regime_label, history, cfg.MIN_HISTORY_WEEKS
        )
        exposure_w = EXPOSURE_WEIGHT[compat]
        db.upsert_p2_weekly(ea.login, week_start, run_id, ea.base_symbol,
                            ea.timeframe, regime.regime_label,
                            compat, basis, n_weeks, exposure_w)

        # Phase 3
        p3 = compute_phase3(trades, initial_bal, compat, exposure_w,
                            ea.base_lot, week_start, cfg)
        final_lot_init = p3.pre_norm_lot if current_state == 'active' else 0.0
        db.upsert_p3_weekly(ea.login, week_start, run_id, p3,
                            final_lot=final_lot_init)

        if current_state == 'active':
            active_lots[ea.login]  = p3.pre_norm_lot
            login_to_sym[ea.login] = ea.base_symbol
```

### 8.3 Execution Schedule

| Job | Tần suất | Ghi chú |
|---|---|---|
| `SET time_zone='+00:00'` | Mỗi connection | BẮT BUỘC — BUG 7 |
| Regime sync | Mỗi tuần (Mon) | Rerun guard + fallback nếu feed down |
| SK evaluation | Mỗi tuần | active + soft_kill EAs |
| HK evaluation | Mỗi 3 tuần | Gated bởi `is_hk_eval_due()` |
| Reactivation check | Mỗi tuần | Chỉ soft_kill EAs |
| Portfolio normalization | Mỗi tuần | Active EAs only, sau per-EA loop |
| MQL5 export | Mỗi tuần | Atomic write |

---

## 9. MQL5 Integration Contract

```python
def export_lot_assignments(p3_rows, week_start, export_path):
    """Atomic write: write to .tmp → rename() → .json"""
    payload = {
        "schema_version": "5.0",
        "week_start":     week_start.isoformat(),
        "generated_at":   datetime.utcnow().isoformat() + "Z",
        "assignments": [
            {
                "login":             row.login,
                "state":             row.state,
                "final_lot":         row.final_lot,
                "base_lot":          row.base_lot,          # BUG 18 fix
                "exposure_weight":   row.exposure_weight,
                "health_score":      row.health_score,
                "health_multiplier": row.health_multiplier,
            }
            for row in p3_rows
        ]
    }
    tmp  = export_path / f"lot_assignments_{week_start:%Y%m%d}.tmp"
    dest = export_path / f"lot_assignments_{week_start:%Y%m%d}.json"
    tmp.write_text(json.dumps(payload, indent=2), encoding='utf-8')
    tmp.rename(dest)   # atomic POSIX; Windows: os.replace()
```

**MQL5 EA contract:**
```
Monday init (sau 00:10 UTC):
1. Đọc lot_assignments_{YYYYMMDD}.json (Monday tuần này)
2. Không tìm thấy login → soft_kill (fail-safe)
3. File lỗi/parse fail → giữ lot tuần trước (fallback)
4. state='soft_kill' OR final_lot=0 → đóng lệnh, không mở mới
5. state='active' → dùng final_lot cho tất cả lệnh tuần này
EA KHÔNG tự override final_lot (đã tính: base_lot × exposure × health × normalization)
```

---

## 10. Configuration

```python
from dataclasses import dataclass
from pathlib import Path

@dataclass(frozen=True)
class Config:
    # Phase 1: Hard Kill
    HK1_MAX_DD_PCT:          float = 0.75
    HK2_MIN_DD_PCT:          float = 0.50
    HK2_TRADE_COUNT_RATIO:   int   = 2
    HK2_MIN_DD_RECOVER_PCT:  float = 0.50
    HK_EVAL_INTERVAL_DAYS:   int   = 21

    # Phase 1: Soft Kill
    SK1_MAX_DD_PCT:          float = 0.20
    SK2_LOOKBACK_TRADES:     int   = 5
    SK2_MIN_SLOPE:           float = 0.02    # fraction/trade
    SK3_DD_MAX:              float = 0.30
    SK4_MAX_LOSS_PCT:        float = 0.10

    # Phase 2: Regime
    TREND_ADX_THRESHOLD:     float = 25.0   # ⚠️ calibrate per symbol
    VOL_ATR_RATIO_THRESHOLD: float = 1.0    # ⚠️ calibrate per symbol
    MIN_HISTORY_WEEKS:       int   = 4
    COMPAT_STRONG_THRESH:    float = 0.005
    COMPAT_GOOD_THRESH:      float = 0.002
    COMPAT_WEAK_THRESH:      float = -0.002
    COMPAT_VERY_WEAK_THRESH: float = -0.005

    # Phase 3: Health Score
    # Window 28 DAY (khác kill rules 3000 DAY — intentional design)
    HEALTH_WINDOW_DAYS:      int   = 28
    HEALTH_WEIGHT_PF:        float = 0.25
    HEALTH_WEIGHT_RF:        float = 0.25
    HEALTH_WEIGHT_RC:        float = 0.20
    HEALTH_WEIGHT_MAXDD:     float = 0.15
    HEALTH_WEIGHT_ES:        float = 0.15

    # Portfolio
    MAX_TOTAL_LOTS:          float = 10.0
    MAX_LOTS_PER_SYMBOL:     float = 3.0

    # Integration
    EXPORT_PATH: Path = Path('/shared/lot_assignments')

cfg = Config()
```

---

## 11. Module Structure

```
src/
├── config.py
├── models.py                      # TradeRow, EAContext, Phase1/2/3Result
│
├── db/
│   ├── connection.py              # MySQL pool
│   │                              # BUG 7: SET time_zone='+00:00' đầu mỗi connection
│   └── queries.py                 # READ Orders + READ/WRITE new tables
│                                  #
│                                  # READ Orders (only):
│                                  #   get_initial_balance(login) → float|None  [§4.1]
│                                  #   load_trades(login) → list[TradeRow]      [§4.7]
│                                  #
│                                  # WRITE new tables:
│                                  #   load_managed_eas() → list[EAContext]
│                                  #     WHERE current_state IN ('active','soft_kill')
│                                  #   has_hard_kill_entry(login) → bool
│                                  #   insert_state_log(...)
│                                  #   update_registry_state(login, new_state)
│                                  #   upsert_p1/p2/p3_weekly(...)
│                                  #   update_p3_final_lot(login, week, lot) ← active only
│                                  #   load_last_p2(login, before) → P2Row|None
│                                  #   load_regime_pnl_history(login, before)
│                                  #     → JOIN ea_p2_weekly + ea_p3_weekly
│                                  #   upsert_regime_history(...)
│                                  #   get/set_config(key, value)
│                                  #   log_pipeline_warning(run_id, login, msg)
│
├── utils/
│   ├── drawdown.py                # compute_max_drawdown, compute_current_drawdown
│   └── interpolate.py             # interpolate_score(value, breakpoints)
│
├── phase1/
│   ├── hard_kill.py               # check_hk1, check_hk2
│   ├── soft_kill.py               # check_sk1, check_sk2, check_sk3, check_sk4
│   └── classifier.py              # classify_ea, check_reactivation
│
├── phase2/
│   ├── indicators.py              # compute_adx, compute_atr_ratio (pandas-ta)
│   ├── regime.py                  # classify_regime, bars_in_week, sync_regime_for_week
│   └── compatibility.py           # compute_regime_compatibility, EXPOSURE_WEIGHT
│
├── phase3/
│   ├── metrics.py                 # compute_pf (gross), compute_rf, compute_es
│   ├── scoring.py                 # score_pf, score_rf, score_maxdd, score_rc
│   │                              # RF_BP: [(0.0,20),(0.5,40),(1.0,60),(1.5,80),(2.0,90),(3.0,100)]
│   ├── health.py                  # compute_health_score, get_health_multiplier
│   └── allocation.py              # compute_pre_norm_lot, normalize_portfolio
│
├── pipeline/
│   ├── run_weekly.py              # run_weekly_pipeline, _process_ea
│   └── schedule.py                # is_hk_eval_due
│
├── market_data/
│   ├── system.py                  # MarketDataSystem: get_ohlcv
│   └── bars_util.py               # bars_in_week(tf_mt4)
│
└── bridge/
    └── mql5_export.py             # export_lot_assignments (atomic)
```

---

## 12. Invariants & Constraints

| # | Invariant | Enforcement |
|---|---|---|
| I1 | `Orders` READ ONLY | Chỉ `get_initial_balance` và `load_trades` được query Orders |
| I2 | Initial balance = orderType=6 đầu tiên | ORDER BY openTime ASC LIMIT 1. None → skip EA + log, không crash |
| I3 | Hard Kill vĩnh viễn | `has_hard_kill_entry(login)` check TRƯỚC classify. True → return ngay |
| I4 | Phase 1 luôn write | `upsert_p1_weekly` trước mọi `return` trong `_process_ea` |
| I5 | Phase 2 đọc tuần trước | `load_regime_pnl_history` dùng `WHERE week_start < {current}` |
| I6 | State transition order | `insert_state_log` → `update_registry_state` (thứ tự này, bất biến) |
| I7 | Normalization sau loop | `normalize_portfolio()` sau toàn bộ per-EA loop |
| I8 | soft_kill → final_lot = 0.0 | Set trong `upsert_p3_weekly`, không qua health_multiplier |
| I9 | Active EA lot floor > 0 | `get_health_multiplier` tối thiểu 0.1, không bao giờ 0.0 |
| I10 | `update_p3_final_lot` chỉ active | Chỉ truyền `active_lots` dict — soft_kill không được update ở đây |
| I11 | Export atomic | Write `.tmp` → `rename()`. Không write thẳng vào `.json` |
| I12 | Pipeline idempotent | UNIQUE (week_start) trên `pipeline_runs`; ON DUPLICATE KEY UPDATE |
| I13 | MySQL timezone = UTC | `SET time_zone = '+00:00'` trong connection setup |
| I14 | `commision` = 1 chữ 's' | Mọi SQL dùng `commision` |
| I15 | PF/WinRate dùng gross profit | `profit > 0` — không phải `net_pnl > 0` |
| I16 | Drawdown/Equity dùng net_pnl | `profit + COALESCE(commision,0) + COALESCE(swap,0)` |
| I17 | ORDER BY có tiebreaker | Mọi window function và ORDER BY thêm `, ticket ASC` |
| I18 | Full history cho cum_pnl | Không query "Last N trades" từ DB. Load full history, slice Python |
| I19 | load_managed_eas = active + soft_kill | WHERE current_state IN ('active','soft_kill') |
| I20 | DD window khác nhau theo mục đích | Kill rules: 3000 DAY. Health Score: 28 DAY. Không nhầm |
