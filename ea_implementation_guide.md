# EA Market – Implementation Guide (3 Phases)

> **Tài liệu này**: Cách thực hiện cụ thể từng phase — SQL, thresholds, job schedule, data flow  
> **Chuẩn**: Myfxbook standard (xem SKILL.md)  
> **Login mẫu**: `193583` | Initial balance: `5000 USD`

---

## Cấu trúc dữ liệu cốt lõi

### Bảng `Orders` (nguồn duy nhất)
```sql
-- Filter bắt buộc cho mọi query
WHERE login = {login}
  AND closeTime IS NOT NULL AND closeTime > 0
  AND orderType IN (0, 1)   -- 0=Buy, 1=Sell only
```

### Các bảng cần tạo thêm
```sql
CREATE TABLE EA_Registry (eaCode VARCHAR(50) PRIMARY KEY, login INT NOT NULL, name VARCHAR(100), created_at DATETIME NOT NULL);
CREATE TABLE EA_Health (eaCode VARCHAR(50) PRIMARY KEY, health_score FLOAT NOT NULL, state ENUM('active','soft_kill','hard_kill') NOT NULL DEFAULT 'active', evaluated_at DATETIME NOT NULL, reason TEXT);
CREATE TABLE EA_State_Log (id INT AUTO_INCREMENT PRIMARY KEY, eaCode VARCHAR(50) NOT NULL, old_state ENUM('active','soft_kill','hard_kill'), new_state ENUM('active','soft_kill','hard_kill'), health_score FLOAT, changed_at DATETIME NOT NULL, reason TEXT);
CREATE TABLE Market_Regime (id INT AUTO_INCREMENT PRIMARY KEY, detected_at DATETIME NOT NULL, week_start DATE NOT NULL, trend_label ENUM('trend','range') NOT NULL, vol_label ENUM('high','low') NOT NULL);
CREATE TABLE EA_Regime_Profile (eaCode VARCHAR(50) NOT NULL, regime VARCHAR(20) NOT NULL, survival_score FLOAT NOT NULL, updated_at DATETIME NOT NULL, PRIMARY KEY (eaCode, regime));
CREATE TABLE EA_Allocation (eaCode VARCHAR(50) PRIMARY KEY, exposure_weight FLOAT NOT NULL DEFAULT 1.0, lot_multiplier FLOAT NOT NULL DEFAULT 1.0, effective_lot_mult FLOAT NOT NULL DEFAULT 1.0, allocated_at DATETIME NOT NULL);
```

---

## Phase 1 — EA Elimination

**Chạy**: Mỗi Chủ Nhật 00:00 UTC  
**Input**: Bảng `Orders`  
**Output**: `EA_Health(eaCode, health_score, state)` + `EA_State_Log`

### Bước 1.1 — Tính 5 metrics từ Orders

#### Metric 1: Max Drawdown % (chuẩn Myfxbook)
```sql
WITH c AS (
  SELECT closeTime,
         SUM(profit + COALESCE(commision,0) + COALESCE(swap,0))
           OVER (ORDER BY closeTime) AS cum_pnl
  FROM Orders WHERE login = 193583 AND closeTime > 0
    AND orderType IN (0, 1)
    AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 3000 DAY)*1000
),
p AS (SELECT cum_pnl, MAX(cum_pnl) OVER (ORDER BY closeTime) AS peak FROM c)
SELECT
  MIN(cum_pnl - peak)                            AS drawdown_abs,
  MIN((cum_pnl - peak) / (5000 + peak)) * 100    AS drawdown_pct
FROM p;
-- drawdown_pct âm (VD: -60.1%) → ABS để tính score
-- drawdown_stability = 1 - LEAST(ABS(drawdown_pct)/100, 1.0)
```

**Trigger ngay lập tức**: `ABS(drawdown_pct) > 70%` → **Hard Kill**

#### Metric 2: Loss Acceleration (consecutive losses)
```sql
SELECT COUNT(*) AS consecutive_losses
FROM (
  SELECT profit, ROW_NUMBER() OVER (ORDER BY closeTime DESC) AS rn
  FROM Orders WHERE login = 193583 AND closeTime > 0 AND orderType IN (0, 1)
) ranked
WHERE rn < (
  SELECT COALESCE(MIN(rn), 999)
  FROM (SELECT profit, ROW_NUMBER() OVER (ORDER BY closeTime DESC) AS rn
        FROM Orders WHERE login = 193583 AND closeTime > 0 AND orderType IN (0, 1)) t2
  WHERE profit > 0
);
-- loss_acceleration_score = 1 - LEAST(consecutive_losses / 10.0, 1.0)
-- Trigger: consecutive_losses >= 5 → xét Soft Kill
```

#### Metric 3: Profit Factor (chuẩn Myfxbook — raw profit)
```sql
SELECT
  SUM(CASE WHEN profit > 0 THEN profit ELSE 0 END) /
  NULLIF(ABS(SUM(CASE WHEN profit < 0 THEN profit ELSE 0 END)), 0) AS profit_factor
FROM Orders WHERE login = 193583 AND closeTime > 0
  AND orderType IN (0, 1)
  AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 3000 DAY) * 1000;
-- profit_factor_score = LEAST(profit_factor / 2.0, 1.0)
-- Trigger: profit_factor < 0.7 → xét Soft Kill
```

#### Metric 4: Stability of Returns
```sql
SELECT STDDEV(weekly_pnl) AS pnl_stddev
FROM (
  SELECT YEARWEEK(FROM_UNIXTIME(closeTime/1000)) AS wk,
         SUM(profit + COALESCE(commision,0) + COALESCE(swap,0)) AS weekly_pnl
  FROM Orders WHERE login = 193583 AND closeTime > 0
    AND orderType IN (0, 1)
    AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 90 DAY)*1000
  GROUP BY wk
) w;
-- stability_score = 1 - LEAST(pnl_stddev / STABILITY_THRESHOLD, 1.0)
-- STABILITY_THRESHOLD = config per account size (VD: 500 USD)
```

#### Metric 5: Abnormal Execution
```sql
SELECT
  SUM(CASE WHEN ABS(commision) > avg_c * 3 THEN 1 ELSE 0 END) * 1.0
    / COUNT(*) AS anomaly_rate
FROM (
  SELECT commision, AVG(ABS(commision)) OVER () AS avg_c
  FROM Orders WHERE login = 193583 AND closeTime > 0
    AND orderType IN (0, 1)
    AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 30 DAY)*1000
) t;
-- execution_score = 1 - anomaly_rate
-- Trigger: anomaly_rate > 0.20 → xét Hard Kill
```

### Bước 1.2 — Tính Health Score (5 components — chuẩn)
```python
# Phase 1 chạy TRƯỚC Phase 2 → không dùng regime_compatibility
health_score = (
    0.30 * drawdown_score          +  # 1 - ABS(drawdown_pct/100)
    0.20 * loss_acceleration_score +  # 1 - LEAST(consecutive_losses/10, 1.0)
    0.25 * profit_factor_score     +  # LEAST(profit_factor/2.0, 1.0)
    0.15 * return_stability_score  +  # 1 - LEAST(pnl_stddev/threshold, 1.0)
    0.10 * execution_normalcy_score   # 1 - anomaly_rate
)
# health_score ∈ [0.0, 1.0]
# Lưu ý: regime_compatibility được dùng trong Phase 3 (Health Score update),
# KHÔNG dùng trong Phase 1 vì Phase 2 chưa chạy
```

**Win Rate** (dùng raw profit — chuẩn Myfxbook):
```sql
SELECT SUM(CASE WHEN profit > 0 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS win_rate
FROM Orders WHERE login = 193583 AND closeTime > 0
  AND orderType IN (0, 1)
  AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 30 DAY)*1000;
```

### Bước 1.3 — State Transitions

| Điều kiện | Transition |
|---|---|
| `ABS(drawdown_pct) > 70%` | → **Hard Kill** (immediate) |
| `health_score < 0.5` × 2 tuần liên tiếp | → Soft Kill |
| `consecutive_losses >= 5` | → Soft Kill |
| `profit_factor < 0.7` (30 ngày) | → Soft Kill |
| `anomaly_rate > 20%` | → Hard Kill |
| Soft Kill > 4 tuần + `health_score < 0.3` | → Hard Kill |
| Soft Kill + `health_score > 0.65` × 2 tuần | → Active (recovery) |

```sql
-- Ghi state transition
INSERT INTO EA_State_Log (eaCode, old_state, new_state, health_score, changed_at, reason)
VALUES (?, ?, ?, ?, NOW(), ?);

UPDATE EA_Health SET health_score=?, state=?, evaluated_at=NOW(), reason=? WHERE eaCode=?;
```

> **Recovery rule**: EA vừa recover từ Soft Kill → Active: giữ `lot_multiplier = 0.5` thêm **2 tuần** trước khi rank lại bình thường.

---

## Phase 2 — Regime-Aware Selection

**Chạy**: Thứ Hai 01:00 UTC (sau Phase 1)  
**Input**: `Orders` × `Market_Regime`  
**Output**: `EA_Regime_Profile` + `EA_Allocation.exposure_weight`

### Bước 2.1 — Classify Market Regime

**Indicators** (cần OHLCV từ broker API):
```python
# ADX trên D1/W1
trend_label = 'trend' if ADX > 25 else 'range'

# ATR so với ATR_90d trung bình
vol_label = 'high' if ATR > 1.3 * ATR_90d else 'low'

regime = f"{trend_label}_{vol_label}"  # VD: 'trend_low', 'range_high'
```

```sql
INSERT INTO Market_Regime (detected_at, week_start, trend_label, vol_label)
VALUES (NOW(), CURDATE() - INTERVAL WEEKDAY(CURDATE()) DAY, ?, ?);
```

### Bước 2.2 — Tính Survival Score per EA per Regime
```sql
-- Survival score của EA tại một regime cụ thể (lookback 180 ngày)
SELECT
  r.eaCode,
  COUNT(*) AS total_trades,
  SUM(CASE WHEN o.profit > 0 THEN 1 ELSE 0 END) * 1.0 / COUNT(*) AS win_rate,
  SUM(CASE WHEN o.profit > 0 THEN o.profit ELSE 0 END) /
    NULLIF(ABS(SUM(CASE WHEN o.profit < 0 THEN o.profit ELSE 0 END)), 0) AS profit_factor
FROM Orders o
JOIN EA_Registry r ON o.login = r.login
JOIN Market_Regime mr
  ON YEARWEEK(FROM_UNIXTIME(o.openTime / 1000)) = YEARWEEK(mr.week_start)
WHERE mr.trend_label = ?   -- 'trend' hoặc 'range'
  AND mr.vol_label   = ?   -- 'high' hoặc 'low'
  AND o.closeTime IS NOT NULL AND o.closeTime > 0
  AND o.orderType IN (0, 1)
  AND o.closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 180 DAY) * 1000
GROUP BY r.eaCode
HAVING COUNT(*) >= 20;   -- min 20 lệnh để có ý nghĩa thống kê
-- survival_score = 0.5 * win_rate + 0.5 * LEAST(profit_factor / 2.0, 1.0)
```

```sql
INSERT INTO EA_Regime_Profile (eaCode, regime, survival_score, updated_at)
VALUES (?, ?, ?, NOW())
ON DUPLICATE KEY UPDATE survival_score=VALUES(survival_score), updated_at=NOW();
```

### Bước 2.3 — Assign Exposure Weight
```python
def assign_exposure_weight(survival_score: float) -> float:
    if survival_score >= 0.70:   return 1.5   # Increased
    elif survival_score >= 0.45: return 1.0   # Normal
    else:                        return 0.5   # Reduced

# Lấy current_regime từ Market_Regime mới nhất
# Tra survival_score trong EA_Regime_Profile
# Ghi vào EA_Allocation
```

```sql
UPDATE EA_Allocation SET exposure_weight=?, allocated_at=NOW() WHERE eaCode=?;
```

---

## Phase 3 — Capital Allocation

**Chạy**: Thứ Hai 02:00 UTC (sau Phase 2)  
**Input**: `Orders` × `EA_Health` × `EA_Allocation.exposure_weight`  
**Output**: `EA_Allocation.lot_multiplier` + `EA_Allocation.effective_lot_mult`

### Bước 3.1 — Tính Performance Score (rolling 30 ngày)
```sql
SELECT
  r.eaCode,
  SUM(o.profit + COALESCE(o.commision,0) + COALESCE(o.swap,0)) AS net_pnl,
  COUNT(*) AS total_trades,
  SUM(CASE WHEN o.profit > 0 THEN 1 ELSE 0 END) * 1.0 / COUNT(*) AS win_rate,
  SUM(CASE WHEN o.profit > 0 THEN o.profit ELSE 0 END) /
    NULLIF(ABS(SUM(CASE WHEN o.profit < 0 THEN o.profit ELSE 0 END)), 0) AS profit_factor
FROM Orders o
JOIN EA_Registry r ON o.login = r.login
JOIN EA_Health h ON r.eaCode = h.eaCode
WHERE h.state = 'active'
  AND o.closeTime IS NOT NULL AND o.closeTime > 0
  AND o.orderType IN (0, 1)
  AND o.closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 30 DAY) * 1000
GROUP BY r.eaCode;
```

### Bước 3.2 — Rank và Assign Lot Multiplier
```python
# Rank tất cả EA active theo performance_score
# performance_score = 0.4*pf_score + 0.3*net_pnl_norm + 0.2*win_rate + 0.1*stability_score

def assign_lot_multiplier(rank_percentile: float) -> float:
    if rank_percentile >= 0.80:   return 1.5   # Top 20%
    elif rank_percentile >= 0.20: return 1.0   # Mid 60%
    else:                         return 0.5   # Bottom 20%

# Smooth: giới hạn thay đổi tối đa mỗi lần
DELTA_MAX = 0.3
new_lot = max(old_lot - DELTA_MAX, min(old_lot + DELTA_MAX, calculated_lot))
```

### Bước 3.3 — Tính Effective Lot và Portfolio Guard
```sql
-- Effective lot = Phase 2 weight × Phase 3 multiplier
UPDATE EA_Allocation
SET lot_multiplier     = ?,
    effective_lot_mult = exposure_weight * ?,
    allocated_at       = NOW()
WHERE eaCode = ?;

-- Portfolio guard: check tổng exposure
SELECT SUM(ea.effective_lot_mult) AS total_exposure
FROM EA_Allocation ea
JOIN EA_Health eh ON ea.eaCode = eh.eaCode
WHERE eh.state = 'active';
-- Nếu total_exposure > MAX_PORTFOLIO_EXPOSURE:
--   scale_factor = MAX_PORTFOLIO_EXPOSURE / total_exposure
--   UPDATE tất cả effective_lot_mult *= scale_factor
```

---

## Job Schedule Tổng Hợp

```
Chủ Nhật 00:00 UTC
  └── evaluate_ea_health()
        ├── Tính 5 metrics từ Orders
        ├── Tính health_score
        └── UPDATE EA_Health + INSERT EA_State_Log

Thứ Hai 01:00 UTC
  ├── classify_market_regime()
  │     └── INSERT Market_Regime (ADX + ATR)
  ├── update_regime_profiles()
  │     └── INSERT/UPDATE EA_Regime_Profile
  └── recalc_exposure_weights()
        └── UPDATE EA_Allocation.exposure_weight

Thứ Hai 02:00 UTC
  ├── rank_and_allocate_lots()
  │     └── UPDATE EA_Allocation.lot_multiplier, effective_lot_mult
  └── apply_portfolio_guard()
        └── Scale down nếu SUM(effective_lot_mult) > MAX
```

---

## Effective Lot — Ví dụ tổng hợp cả 3 Phase

| EA | State (P1) | Exposure (P2) | Lot Mult (P3) | **Effective Lot** |
|---|---|---|---|---|
| EA_GOLD_001 | active | 1.5 (trend-compatible) | 1.5 (top 20%) | **2.25** ← max vốn |
| EA_EUR_001 | active | 1.0 (neutral) | 1.0 (mid) | **1.0** |
| EA_GOLD_002 | active | 0.5 (regime-fragile) | 0.5 (bottom 20%) | **0.25** ← min vốn |
| EA_BAD_001 | soft_kill | — | — | **0** (không giao dịch) |

---

## API Surface

| Method | Endpoint | Mô tả |
|---|---|---|
| GET | `/ea/{eaCode}/state` | State hiện tại + health_score |
| GET | `/ea/{eaCode}/health-history` | Lịch sử health score theo tuần |
| GET | `/ea/{eaCode}/orders` | Orders của EA (query Orders WHERE login = EA_Registry[eaCode].login) |
| GET | `/ea/{eaCode}/regime-profile` | Survival score theo 4 regime |
| GET | `/ea/{eaCode}/allocation` | exposure_weight, lot_multiplier, effective_lot_mult |
| GET | `/market/current-regime` | Regime tuần hiện tại |
| GET | `/portfolio/summary` | Tổng quan: count rate vs capital-weighted rate |
| POST | `/admin/ea/{eaCode}/soft-kill` | Force soft kill thủ công |
| POST | `/admin/ea/{eaCode}/hard-kill` | Force hard kill vĩnh viễn |

---

## Tech Stack

| Layer | Gợi ý |
|---|---|
| Existing DB | MySQL (bảng `Orders` + các bảng bổ sung) |
| Backend API | Python FastAPI hoặc Node.js NestJS |
| Periodic Jobs | Celery + Redis, hoặc Python APScheduler |
| Market Data (regime) | MetaTrader 5 API / Broker WebSocket (OHLCV) |
| Monitoring | Grafana — dashboard health scores, exposure, regime |
| Notifications | Telegram Bot API — Hard Kill alert, regime change |
