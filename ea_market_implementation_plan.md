# EA Market – Implementation Plan (Developer-Oriented)

> **Source**: EA Market Core Logic Improvement document  
> **Architecture**: Immune System × Ecosystem — EAs as species, system as regulator  
> **Core principle**: Not finding winners, but eliminating destructive strategies while surviving strategies continue.

---

## Nguồn dữ liệu duy nhất: Bảng `Orders`

**Toàn bộ 3 phase đều được tính từ một nguồn duy nhất: bảng `Orders` trong MySQL.**

### Mapping eaCode → login → Orders

Mỗi EA trong hệ thống có một **`eaCode`** là identifier chính. Mỗi `eaCode` được liên kết với **đúng một `login`** (tài khoản broker). Tuy nhiên, **một `login` có thể chứa nhiều `eaCode`**.

```
eaCode  (identifier chính của hệ thống)
  └──► login  (tài khoản broker, 1 login có thể có nhiều eaCode)
           └──► SELECT * FROM Orders WHERE login = {login}
                    └── Toàn bộ lịch sử lệnh của login đó,
                        đại diện cho eaCode tương ứng
```

Vì vậy, hệ thống cần một bảng mapping:

```sql
-- Bảng mapping eaCode → login (do hệ thống EA Market quản lý)
CREATE TABLE EA_Registry (
  eaCode   VARCHAR(50) PRIMARY KEY,   -- ⭐ Identifier chính trong toàn hệ thống
  login    INT NOT NULL,              -- Dùng để query bảng Orders
  name     VARCHAR(100),
  created_at DATETIME NOT NULL
);
-- Ví dụ:
-- eaCode='EA_GOLD_001' → login=193583 → SELECT * FROM Orders WHERE login=193583
```

**Query lấy Orders của một eaCode:**

```sql
SELECT o.*
FROM Orders o
JOIN EA_Registry r ON o.login = r.login
WHERE r.eaCode = 'EA_GOLD_001'
  AND o.closeTime IS NOT NULL AND o.closeTime > 0
  AND o.orderType IN (0, 1);
```

### Schema bảng Orders

```sql
CREATE TABLE `Orders` (
  `id`            int NOT NULL AUTO_INCREMENT,
  `ticket`        int DEFAULT NULL,               -- Mã lệnh từ broker
  `orderType`     int DEFAULT NULL,               -- 0=Buy, 1=Sell, v.v.
  `symbol`        varchar(15),                    -- VD: XAUUSD.std
  `baseSymbol`    varchar(15),                    -- VD: XAUUSD
  `login`         int DEFAULT NULL,               -- ⭐ EA identifier
  `broker`        varchar(50),
  `magicNumber`   int DEFAULT NULL,
  `openTime`      bigint DEFAULT NULL,            -- Unix timestamp (ms)
  `closeTime`     bigint DEFAULT NULL,            -- Unix timestamp (ms)
  `volume`        float DEFAULT NULL,             -- Lot size
  `openPrice`     float DEFAULT NULL,
  `closePrice`    float DEFAULT NULL,
  `TP`            float DEFAULT NULL,
  `SL`            float DEFAULT NULL,
  `commision`     float DEFAULT NULL,
  `swap`          float DEFAULT NULL,
  `profit`        float DEFAULT NULL,             -- ⭐ PnL của lệnh
  `comment`       varchar(256),
  `currency`      varchar(10),
  `eaMagicNumber` int DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `unique_ticket_broker` (`ticket`, `broker`)
);
```

> **Lưu ý**: Chỉ tính các lệnh đã **đóng** (`closeTime IS NOT NULL AND closeTime > 0`).

---

## Data Model bổ sung (ngoài Orders)

Ngoài bảng `Orders` sẵn có, hệ thống cần thêm các bảng sau:

```sql
-- Mapping eaCode ↔ login (bảng gốc của hệ thống)
CREATE TABLE EA_Registry (
  eaCode     VARCHAR(50) PRIMARY KEY,         -- ⭐ Identifier chính
  login      INT NOT NULL,                    -- Dùng để query Orders
  name       VARCHAR(100),
  created_at DATETIME NOT NULL
);

-- Trạng thái và điểm sức khoẻ của EA (Phase 1)
CREATE TABLE EA_Health (
  eaCode       VARCHAR(50) PRIMARY KEY,       -- FK → EA_Registry.eaCode
  health_score FLOAT NOT NULL,               -- 0.0 – 1.0
  state        ENUM('active','soft_kill','hard_kill') NOT NULL DEFAULT 'active',
  evaluated_at DATETIME NOT NULL,
  reason       TEXT
);

-- Log lịch sử state transitions (Phase 1)
CREATE TABLE EA_State_Log (
  id           INT AUTO_INCREMENT PRIMARY KEY,
  eaCode       VARCHAR(50) NOT NULL,          -- FK → EA_Registry.eaCode
  old_state    ENUM('active','soft_kill','hard_kill'),
  new_state    ENUM('active','soft_kill','hard_kill'),
  health_score FLOAT,
  changed_at   DATETIME NOT NULL,
  reason       TEXT
);

-- Chế độ thị trường được phát hiện (Phase 2)
CREATE TABLE Market_Regime (
  id           INT AUTO_INCREMENT PRIMARY KEY,
  detected_at  DATETIME NOT NULL,
  week_start   DATE NOT NULL,
  trend_label  ENUM('trend','range') NOT NULL,
  vol_label    ENUM('high','low') NOT NULL
  -- regime key = trend_label + '_' + vol_label, VD: 'trend_high', 'range_low'
);

-- Profile hiệu suất từng EA theo regime (Phase 2)
CREATE TABLE EA_Regime_Profile (
  eaCode         VARCHAR(50) NOT NULL,        -- FK → EA_Registry.eaCode
  regime         VARCHAR(20) NOT NULL,        -- 'trend_high', 'range_low', ...
  survival_score FLOAT NOT NULL,             -- 0.0 – 1.0
  updated_at     DATETIME NOT NULL,
  PRIMARY KEY (eaCode, regime)
);

-- Phân bổ lot (Phase 3)
CREATE TABLE EA_Allocation (
  eaCode             VARCHAR(50) PRIMARY KEY, -- FK → EA_Registry.eaCode
  exposure_weight    FLOAT NOT NULL DEFAULT 1.0,   -- Phase 2 output
  lot_multiplier     FLOAT NOT NULL DEFAULT 1.0,   -- Phase 3 output
  effective_lot_mult FLOAT NOT NULL DEFAULT 1.0,   -- = exposure_weight × lot_multiplier
  allocated_at       DATETIME NOT NULL
);
```

---

## Phase 1 – EA Elimination (Hard Kill / Soft Kill)

### Mục tiêu
Loại bỏ hoặc cô lập các EA có hành vi tiêu cực dai dẳng để ngăn rủi ro thua lỗ sâu.  
Mục tiêu duy nhất: **bảo vệ hệ thống**, không phải tối đa hoá lợi nhuận.

### 1.1 Tính Health Score từ bảng Orders

Mỗi tuần, với từng `eaCode`, tra `login` từ `EA_Registry` rồi tính `health_score` từ Orders trong rolling window (VD: 30 ngày gần nhất).

```sql
-- Lấy login từ eaCode trước khi query Orders
SELECT login FROM EA_Registry WHERE eaCode = 'EA_GOLD_001';
-- → login = 193583, sau đó dùng login này cho các query bên dưới
```

#### Metric 1: Drawdown Level

```sql
-- Tính max drawdown trên toàn bộ lịch sử (dung INTERVAL 3000 DAY ~ all history)
WITH cumulative AS (
  SELECT
    closeTime,
    SUM(profit + COALESCE(commision,0) + COALESCE(swap,0))
      OVER (ORDER BY closeTime ROWS UNBOUNDED PRECEDING) AS cum_pnl
  FROM Orders
  WHERE login = 193583
    AND closeTime IS NOT NULL AND closeTime > 0
    AND orderType IN (0, 1)
    AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 3000 DAY) * 1000
),
running_max AS (
  SELECT
    cum_pnl,
    MAX(cum_pnl) OVER (ORDER BY closeTime ROWS UNBOUNDED PRECEDING) AS peak_cum_pnl
  FROM cumulative
)
SELECT
  MIN(cum_pnl - peak_cum_pnl)                                       AS max_drawdown_abs,
  MIN((cum_pnl - peak_cum_pnl) / (5000 + peak_cum_pnl)) * 100       AS drawdown_pct;
-- 5000 = initial_balance của login 193583
-- drawdown_pct = MIN((trough - peak) / (initial + peak)) → mỗi điểm dùng peak của chính nó
```

**drawdown_score** = `1 - ABS(drawdown_pct / 100)`  
→ Nếu `drawdown_pct < -70%` → kích hoạt **Hard Kill** ngay lập tức.

#### Metric 2: Loss Acceleration

```sql
-- Tính số lệnh thua liên tiếp gần nhất (login = 193583)
SELECT COUNT(*) AS consecutive_losses
FROM (
  SELECT
    profit,
    ROW_NUMBER() OVER (ORDER BY closeTime DESC) AS rn
  FROM Orders
  WHERE login = 193583
    AND closeTime IS NOT NULL AND closeTime > 0
    AND orderType IN (0, 1)
) sub
WHERE rn <= (
  SELECT MIN(rn) - 1
  FROM (
    SELECT ROW_NUMBER() OVER (ORDER BY closeTime DESC) AS rn, profit
    FROM Orders WHERE login = 193583 AND closeTime IS NOT NULL
      AND orderType IN (0, 1)
  ) t
  WHERE profit > 0
);
```

**loss_acceleration_score** = `1 - min(consecutive_losses / 10, 1.0)`

#### Metric 3: Profit Factor

```sql
-- Chuan Myfxbook: raw profit only (khong cong commision/swap)
SELECT
  SUM(CASE WHEN profit > 0 THEN profit ELSE 0 END) /
  NULLIF(ABS(SUM(CASE WHEN profit < 0 THEN profit ELSE 0 END)), 0)
    AS profit_factor
FROM Orders
WHERE login = 193583
  AND closeTime IS NOT NULL AND closeTime > 0
  AND orderType IN (0, 1)
  AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 30 DAY) * 1000;
```

**profit_factor_score** = `LEAST(profit_factor / 2.0, 1.0)` (chuẩn hoá về [0,1])

#### Metric 4: Stability of Returns (Độ ổn định lợi nhuận theo tuần)

```sql
SELECT STDDEV(weekly_pnl) AS pnl_stddev
FROM (
  SELECT
    YEARWEEK(FROM_UNIXTIME(closeTime / 1000)) AS wk,
    SUM(profit + COALESCE(commision,0) + COALESCE(swap,0)) AS weekly_pnl
  FROM Orders
  WHERE login = 193583
    AND closeTime IS NOT NULL AND closeTime > 0
    AND orderType IN (0, 1)
    AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 90 DAY) * 1000
  GROUP BY wk
) weekly;
```

**return_stability_score** = `1 - LEAST(pnl_stddev / threshold, 1.0)`  
(threshold điều chỉnh theo quy mô account)

#### Metric 5: Abnormal Execution Behaviour

```sql
-- Detect các lệnh có slippage bất thường (|openPrice - expected| > threshold)
-- hoặc commission/swap bất thường cao
SELECT
  COUNT(CASE WHEN ABS(commision) > avg_commision * 3 THEN 1 END) AS anomaly_count,
  COUNT(*) AS total_orders
FROM (
  SELECT commision,
         AVG(ABS(commision)) OVER () AS avg_commision
  FROM Orders
  WHERE login = 193583
    AND closeTime IS NOT NULL AND closeTime > 0
    AND orderType IN (0, 1)
    AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 30 DAY) * 1000
) t;
```

**execution_normalcy_score** = `1 - (anomaly_count / total_orders)`

#### Tổng hợp Health Score

```python
health_score = (
    0.30 * drawdown_score +
    0.20 * loss_acceleration_score +
    0.25 * profit_factor_score +
    0.15 * return_stability_score +
    0.10 * execution_normalcy_score
)
```

### 1.2 State Machine

```
                    ┌──────────────────────────────────────┐
                    │              ACTIVE                  │
                    │     (giao dịch bình thường)         │
                    └──────────────┬───────────────────────┘
                                   │
              consistent negative returns /
              declining performance trend /
              incompatible with current regime
                                   │
                                   ▼
                    ┌──────────────────────────────────────┐
                    │            SOFT KILL                 │◄── recovery → Active
                    │  (tạm dừng lệnh, tiếp tục giám sát) │
                    └──────────────┬───────────────────────┘
                                   │
                    extreme drawdown (profit tích luỹ < -70%) /
                    structural failure / persistent inability to recover
                                   │
                                   ▼
                    ┌──────────────────────────────────────┐
                    │            HARD KILL                 │
                    │  (xoá vĩnh viễn, giữ dữ liệu Orders)│
                    └──────────────────────────────────────┘
```

**Soft Kill triggers:**
- `health_score < 0.5` trong 2 tuần liên tiếp
- Loss streak ≥ 5 lệnh liên tiếp (từ query consecutive_losses)
- Profit factor < 0.7 trong 30 ngày

**Hard Kill triggers:**
- Cumulative loss > 70% initial_balance (từ drawdown query)
- `health_score < 0.2` trong 4 tuần liên tiếp
- Structural failure: > 20% lệnh có anomaly

**Recovery (Soft Kill → Active):**
- `health_score > 0.65` trong 2 tuần liên tiếp

### 1.3 Implementation Tasks

- [ ] Stored procedure hoặc Python job `evaluate_ea_health(login)` chạy **mỗi Chủ Nhật 00:00 UTC**
- [ ] Áp dụng cho tất cả `login` distinct trong bảng `Orders` có giao dịch trong 90 ngày gần nhất
- [ ] Ghi kết quả vào `EA_Health` và log state change vào `EA_State_Log`
- [ ] Alert khi Hard Kill qua Telegram Bot

---

## Phase 2 – Regime-Aware Selection (Exposure Control)

### Mục tiêu
Tăng exposure của EA phù hợp chế độ thị trường hiện tại, giảm EA không phù hợp.  
**Không dự đoán thị trường** — chỉ hỏi: *"EA nào ít fragile nhất trong môi trường hiện tại?"*

### 2.1 Xác định Market Regime

Mỗi đầu tuần, phân loại thị trường thành một trong 4 chế độ:

| Regime Key | Trend | Volatility |
|---|---|---|
| `trend_low` | Trend | Low |
| `trend_high` | Trend | High |
| `range_low` | Range | Low |
| `range_high` | Range | High |

**Cách tính từ dữ liệu thị trường** (cần OHLCV bên ngoài DB Orders):

| Dimension | Indicator | Ngưỡng |
|---|---|---|
| Trend vs Range | ADX (D1 / W1) | ADX > 25 → Trend; ≤ 25 → Range |
| High vs Low Vol | ATR / ATR_90d | ATR > 1.3× ATR_90d → High; else → Low |

> Ghi regime hiện tại vào bảng `Market_Regime` mỗi tuần.

### 2.2 Xây dựng Regime Performance Profile từ Orders

Với mỗi `login` và mỗi regime đã biết (lấy từ `Market_Regime` theo `week_start`), tính survival score từ Orders:

```sql
-- Tính survival_score của eaCode 'EA_GOLD_001' (login=193583) tại regime 'trend_low'
SELECT
  r.eaCode,
  COUNT(*) AS total_trades,
  SUM(CASE WHEN o.profit > 0 THEN 1 ELSE 0 END) * 1.0 / COUNT(*) AS win_rate,
  SUM(CASE WHEN o.profit > 0 THEN o.profit ELSE 0 END)
    / NULLIF(ABS(SUM(CASE WHEN o.profit < 0 THEN o.profit ELSE 0 END)), 0) AS profit_factor
FROM Orders o
JOIN EA_Registry r ON o.login = r.login
JOIN Market_Regime mr
  ON YEARWEEK(FROM_UNIXTIME(o.openTime / 1000)) = YEARWEEK(mr.week_start)
WHERE r.eaCode       = 'EA_GOLD_001'
  AND mr.trend_label = 'trend'
  AND mr.vol_label   = 'low'
  AND o.closeTime IS NOT NULL AND o.closeTime > 0
  AND o.orderType IN (0, 1)
GROUP BY r.eaCode;
```

**survival_score** = `0.5 * win_rate + 0.5 * LEAST(profit_factor / 2.0, 1.0)`

Lưu vào `EA_Regime_Profile(eaCode, regime, survival_score)`.

**Ví dụ profile result:**

| eaCode | trend_low | trend_high | range_low | range_high |
|---|---|---|---|---|
| EA_GOLD_001 | 0.82 | 0.60 | 0.30 | 0.45 |
| EA_GOLD_002 | 0.35 | 0.40 | 0.78 | 0.71 |
| EA_EUR_001  | 0.51 | 0.82 | 0.48 | 0.39 |

### 2.3 Exposure Adjustment

Khi regime hiện tại là `trend_low`, tra cứu `survival_score` của từng EA và gán:

```python
def assign_exposure_weight(survival_score: float) -> float:
    if survival_score >= 0.70:   return 1.5   # Increased exposure
    elif survival_score >= 0.45: return 1.0   # Normal exposure
    else:                        return 0.5   # Reduced exposure
```

| Level | Survival Score | Exposure Weight |
|---|---|---|
| Increased (regime-compatible) | ≥ 0.70 | × 1.5 |
| Normal (neutral) | 0.45 – 0.70 | × 1.0 |
| Reduced (regime-fragile) | < 0.45 | × 0.5 |

> EA không bị xoá khỏi pool — vẫn giao dịch nhưng với lot nhỏ hơn.  
> Điều này đảm bảo hệ thống thích nghi nhanh khi regime thay đổi.

### 2.4 Implementation Tasks

- [ ] Job `classify_market_regime()` chạy **thứ Hai 01:00 UTC** — ghi vào `Market_Regime`
- [ ] Job `update_regime_profiles(eaCode)` — JOIN `EA_Registry` → `Orders` × `Market_Regime` để tính score per eaCode
- [ ] Job `recalc_exposure_weights()` — tra `EA_Regime_Profile(eaCode, ...)` → ghi `EA_Allocation(eaCode, exposure_weight)`
- [ ] API: `GET /market/current-regime`, `GET /ea/{eaCode}/regime-profile`

---

## Phase 3 – Capital Allocation Optimisation

### Mục tiêu
Tập trung vốn vào EA đang hoạt động hiệu quả nhất. Nguyên tắc: **increases lot for profitable EAs, reduces lot for losing EAs**.

### 3.1 Tại sao quan trọng — Ví dụ từ tài liệu

```
100 EA đang active sau Phase 1 + Phase 2:
  ├── 55 EA đang có lãi   → chiếm đa số vốn
  └── 45 EA đang lỗ       → chỉ còn ít vốn

EA count success rate = 55% (không đổi)
capital-weighted success rate > 60%  ← Phase 3 tạo ra điều này
```

### 3.2 Tính Performance Score từ Orders

Mỗi tuần, xếp hạng tất cả EA đang `active` theo performance trong rolling 30 ngày:

```sql
-- Tính performance score cho tất cả eaCode đang active
SELECT
  r.eaCode,
  SUM(o.profit + COALESCE(o.commision,0) + COALESCE(o.swap,0))   AS net_pnl,
  COUNT(*)                                                          AS total_trades,
  SUM(CASE WHEN o.profit > 0 THEN 1 ELSE 0 END) * 1.0 / COUNT(*) AS win_rate,
  SUM(CASE WHEN o.profit > 0 THEN o.profit ELSE 0 END)
    / NULLIF(ABS(SUM(CASE WHEN o.profit < 0 THEN o.profit ELSE 0 END)), 0) AS profit_factor,
  STDDEV(weekly_pnl)                                               AS pnl_stddev
FROM (
  SELECT
    r2.eaCode,
    YEARWEEK(FROM_UNIXTIME(o2.closeTime / 1000))                  AS wk,
    SUM(o2.profit + COALESCE(o2.commision,0) + COALESCE(o2.swap,0)) AS weekly_pnl
  FROM Orders o2
  JOIN EA_Registry r2 ON o2.login = r2.login
  WHERE o2.closeTime IS NOT NULL AND o2.closeTime > 0
    AND o2.orderType IN (0, 1)
    AND o2.closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 30 DAY) * 1000
  GROUP BY r2.eaCode, wk
) weekly_data
JOIN Orders o ON o.login = (SELECT login FROM EA_Registry WHERE eaCode = weekly_data.eaCode)
JOIN EA_Registry r ON r.eaCode = weekly_data.eaCode
GROUP BY r.eaCode;
```

**performance_score** (composite):

```python
performance_score = (
    0.40 * profit_factor_score +   # LEAST(profit_factor / 2.0, 1.0)
    0.30 * net_pnl_score +         # chuẩn hoá net_pnl về [0,1] theo toàn pool
    0.20 * win_rate +
    0.10 * return_stability_score  # 1 - LEAST(pnl_stddev / threshold, 1.0)
)
```

### 3.3 Lot Multiplier Assignment

Sau khi có `performance_score` cho tất cả EA active, sắp xếp và phân tier:

```python
def assign_lot_multiplier(rank_percentile: float) -> float:
    # rank_percentile = vị trí trong pool, 1.0 = tốt nhất
    if rank_percentile >= 0.80:   return 1.5   # Top 20% → Capital Boost
    elif rank_percentile >= 0.20: return 1.0   # Mid 60% → Normal
    else:                         return 0.5   # Bottom 20% → Capital Reduction
```

| Tier | Percentile | Lot Multiplier | Ý nghĩa |
|---|---|---|---|
| Top | ≥ 80% | × 1.5 | Tập trung vốn vào chiến lược mạnh |
| Mid | 20–80% | × 1.0 | Giữ nguyên |
| Bottom | < 20% | × 0.5 | Giảm exposure cho EA yếu |

### 3.4 Effective Lot — Kết hợp cả 3 Phase

```
effective_lot_multiplier = exposure_weight (Phase 2) × lot_multiplier (Phase 3)

VD: EA có exposure_weight=1.5 (regime-compatible) và lot_multiplier=1.5 (top performer)
    → effective_lot_multiplier = 1.5 × 1.5 = 2.25  ← nhận nhiều vốn nhất
    
    EA có exposure_weight=0.5 (regime-fragile) và lot_multiplier=0.5 (bottom)
    → effective_lot_multiplier = 0.5 × 0.5 = 0.25  ← nhận rất ít vốn
```

Ghi vào `EA_Allocation`:
```sql
UPDATE EA_Allocation
SET exposure_weight    = ?,
    lot_multiplier     = ?,
    effective_lot_mult = exposure_weight * lot_multiplier,
    allocated_at       = NOW()
WHERE eaCode = ?;   -- ⭐ dùng eaCode, không dùng login
```

### 3.5 Portfolio-Level Risk Guard

```sql
SELECT SUM(ea.effective_lot_mult) AS total_exposure_units
FROM EA_Allocation ea
JOIN EA_Health eh ON ea.eaCode = eh.eaCode
WHERE eh.state = 'active';
```

```python
if total_exposure_units > MAX_EXPOSURE_UNITS:
    scale_factor = MAX_EXPOSURE_UNITS / total_exposure_units
    # Scale down tất cả lot_multiplier tỷ lệ đều nhau
```

### 3.6 Implementation Tasks

- [ ] Job `rank_and_allocate_lots(eaCode)` chạy **thứ Hai 02:00 UTC** (sau Phase 1 & 2 hoàn thành)
- [ ] Chỉ xét các `eaCode` có `EA_Health.state = 'active'`
- [ ] Giới hạn thay đổi lot mỗi lần: `|new - old| ≤ 0.3` để tránh thay đổi đột ngột
- [ ] EA vừa recover từ Soft Kill: giữ `lot_multiplier = 0.5` thêm 2 tuần
- [ ] Portfolio guard: alert Telegram nếu `total_exposure_units > MAX`

---

## Weekly Job Schedule

Ba phase chạy tuần tự — **output của phase trước là input của phase sau**:

```
Chủ Nhật 00:00 UTC ─────────────────────────────────────────────────
  Phase 1: evaluate_ea_health(eaCode)
    ├── Với từng eaCode trong EA_Registry → tra login → query Orders
    ├── Tính 5 metrics → health_score
    └── UPDATE EA_Health(eaCode) + INSERT EA_State_Log(eaCode)

Thứ Hai 01:00 UTC ───────────────────────────────────────────────────
  Phase 2a: classify_market_regime()
    └── Tính ADX + ATR từ market data → INSERT Market_Regime

  Phase 2b: update_regime_profiles(eaCode)
    └── JOIN EA_Registry→Orders × Market_Regime
        → UPDATE EA_Regime_Profile(eaCode, regime, survival_score)

  Phase 2c: recalc_exposure_weights(eaCode)
    └── Tra EA_Regime_Profile(eaCode) tại regime hiện tại
        → UPDATE EA_Allocation(eaCode, exposure_weight)

Thứ Hai 02:00 UTC ───────────────────────────────────────────────────
  Phase 3a: rank_and_allocate_lots(eaCode)
    └── Query EA_Registry→Orders (rolling 30 ngày) → performance_score
        → UPDATE EA_Allocation(eaCode, lot_multiplier, effective_lot_mult)

  Phase 3b: apply_portfolio_guard()
    └── SUM(effective_lot_mult) WHERE state='active'
        → scale down nếu vượt MAX
```

---

## API Surface

| Method | Endpoint | Mô tả |
|---|---|---|
| GET | `/ea/{eaCode}/state` | State hiện tại + health_score |
| GET | `/ea/{eaCode}/health-history` | Lịch sử health score theo tuần |
| GET | `/ea/{eaCode}/orders` | Orders của EA (query `Orders WHERE login = EA_Registry[eaCode].login`) |
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
