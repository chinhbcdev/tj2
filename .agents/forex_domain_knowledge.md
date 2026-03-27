# Forex & EA Market — Domain Knowledge
> **Nguồn**: Tự học từ domain expertise + nghiên cứu industry best practices
> **Cập nhật**: 2026-03-19

---

## MT4/MT5 Orders — orderType mapping

> **⚠️ QUAN TRỌNG**: MQL4 và MQL5 có orderType khác nhau!

### MQL4 (OrderType) — dùng trong bảng `Orders` broker:
| orderType | Hằng số | Ý nghĩa |
|---|---|---|
| 0 | OP_BUY | Buy market |
| 1 | OP_SELL | Sell market |
| 2 | OP_BUYLIMIT | Buy Limit pending |
| 3 | OP_SELLLIMIT | Sell Limit pending |
| 4 | OP_BUYSTOP | Buy Stop pending |
| 5 | OP_SELLSTOP | Sell Stop pending |
| 6 | *(undocumented)* | Balance/deposit/withdrawal |

### MQL5 (ENUM_ORDER_TYPE) — orders layer mới:
| value | Hằng số | Ý nghĩa |
|---|---|---|
| 0 | ORDER_TYPE_BUY | Market buy |
| 1 | ORDER_TYPE_SELL | Market sell |
| 6 | ORDER_TYPE_BUY_STOP_LIMIT | **Khác MQL4!** — không phải balance |
| 8 | ORDER_TYPE_CLOSE_BY | Đóng position bằng lệnh ngược chiều |

> **Mớ bảng `Orders` này dùng MQL4 format** (broker lưu lịch sử MT4).  
> `orderType = 6` trong `Orders` = balance/nap/rut tiền → filter `IN (0, 1)` đúng.

---

## Broker data conventions

| Field | Ý nghĩa | Lưu ý |
|---|---|---|
| `profit` | Raw P&L từ position | Chưa trừ phí |
| `commision` | Phí broker | Luôn **âm** — 1 chữ 's' |
| `swap` | Rollover overnight | Có thể + hoặc − |
| `volume` | Lot size | 1.0 = standard lot |
| `closeTime` | Unix ms | = 0 nếu đang mở |
| `magicNumber` | EA ID trong MT4 | Mỗi EA dùng magic riêng |

```
net_pnl   = profit + commision + swap  ← drawdown, weekly P&L
gross_pnl = profit                     ← myfxbook profit_factor, win_rate
```

> **MQL4 API xác nhận**: `OrderProfit()` = raw profit, không bao gồm swap và commission.  
> `OrderCommission()` = phí broker (luôn âm).  `OrderSwap()` = rollover (có thể + hoặc −).

---

## Market Regime — ADX + ATR combination (verified from research)

```python
# ADX(14) trên D1 — industry standard period
tend_label = 'trend' if ADX > 25 else 'range'
# < 20: có thể là range | 20-25: transition zone | > 25: confirmed trend

# ATR(14) vs trung bình 90 ngày
vol_label = 'high' if ATR > 1.3 * ATR_90day else 'low'

regime = f"{trend_label}_{vol_label}"
```

> **Nhưu cần chú ý**: ADX xác định sức mạnh trend, KHÔNG xác định hướng (up/down).  
> Cần thêm **+DI và −DI** để biết chiều xu hướng (lên hay xuống) khi cần.

| Regime | ADX | ATR | EA type phù hợp |
|---|---|---|---|
| `trend_low` | > 25 | Thấp | Trend-following, MA crossover |
| `trend_high` | > 25 | Cao | Breakout EA, cần SL chặt |
| `range_low` | ≤ 25 | Thấp | Mean-reversion, grid EA |
| `range_high` | ≤ 25 | Cao | **Khó nhất** — giảm exposure tất cả |

---

## Drawdown — Industry Standards

| Metric | Công thức | Myfxbook |
|---|---|---|
| Absolute drawdown | initial_balance − lowest_balance | Không |
| Relative (peak-to-trough) | (peak − trough) / peak | ✅ |
| Max drawdown % | max relative qua toàn history | ✅ |

**Industry benchmarks (từ professional algo traders):**
- Max drawdown < **15-20%** = acceptable
- Max drawdown > **25%** = circuit breaker should trigger
- Recovery math: 25% DD cần 33% gain để hòa vốn; 50% DD cần 100% gain

> ### ⚠️ QUAN TRỌNG: Myfxbook Drawdown vs Project Drawdown
>
> **Myfxbook** tính drawdown bao gồm **cả floating P&L (lệnh đầng mở)**:  
> `max_dd = (peak_balance - lowest_equity_including_open_trades) / peak_balance`
>
> **Project này** tính trên **closed trades only** (`closeTime > 0`):  
> `drawdown_pct = MIN((cum_pnl - peak) / (initial + peak))`
>
> ⇒ Drawdown của project sẽ **thấp hơn Myfxbook** vì bỏ qua floating losses.  
> → Điều này phù hợp với design của hệ thống và **phải ghi chú rõ** khi so sánh với Myfxbook.

---

## Risk-Adjusted Metrics (Industry Standard)

### Sharpe Ratio
```
Sharpe = (Avg Return - Risk-Free Rate) / StdDev(Returns)
```
| Sharpe | Đánh giá |
|---|---|
| < 1.0 | Không đủ bù rủi ro |
| 1.0 – 2.0 | Tốt |
| > 2.0 | Rất tốt |

### Calmar Ratio (phổ biến nhất cho Forex EA)
```
Calmar = Annualized Return / Max Drawdown %
Lookback: 36 tháng gần nhất
```
| Calmar | Đánh giá |
|---|---|
| < 1.0 | Không đủ bù rủi ro |
| 1.0 – 3.0 | Tốt |
| > 3.0 | Xuất sắc |

### Recovery Factor
```
Recovery Factor = Net Profit / Max Drawdown (absolute)
```
> Myfxbook hiển thị Recovery Factor — EA tốt nên có RF > 3

### Profit Factor benchmarks (Myfxbook/MT4 standard)
| Profit Factor | Đánh giá |
|---|---|
| < 1.0 | Lỗ |
| 1.0 – 1.40 | Biên mỏng |
| 1.41 – 2.00 | Tốt |
| > 2.01 | Xuất sắc |

### Expectancy (trung bình lãi/lỗ per trade)
```
Expectancy = (Win Rate × Avg Win) - (Loss Rate × Avg Loss)
```
> Expectancy > 0 = profitable. Quan trọng hơn win rate đơn thuần.

---

## Health Score (5 components — chuẩn đã sync)

```python
# Phase 1 chạy TRƯỚC Phase 2 → không dùng regime_compatibility
health_score = (
    0.30 * drawdown_score          +  # 1 - ABS(drawdown_pct/100)
    0.20 * loss_acceleration_score +  # 1 - LEAST(consecutive_losses/10, 1)
    0.25 * profit_factor_score     +  # LEAST(profit_factor/2.0, 1)
    0.15 * return_stability_score  +  # 1 - LEAST(stddev/threshold, 1)
    0.10 * execution_normalcy_score   # 1 - anomaly_rate
)
```

| Score | State |
|---|---|
| ≥ 0.65 | Active |
| 0.3–0.64 | Soft Kill candidate |
| < 0.3 sustained | Hard Kill |

---

## Kill Switch — Industry Best Practices

Từ research: Kill switch là **circuit breaker bắt buộc** trong production trading systems:

- **What**: Dừng ngay toàn bộ trading khi vượt ngưỡng thiệt hại
- **When to trigger**: Account balance drops > 20-25% from peak
- **Hard Kill level**: > 70% peak balance (dùng trong project này — conservative)
- **Types**:
  - Level 1: Soft Kill (pause, monitor, allow recovery)
  - Level 2: Hard Kill (permanent remove, data retained)
- **Production**: Log tất cả state transitions, alert Telegram immediately
- **Regulatory note**: MiFID II xem xét bắt buộc kill switch accessible bởi clearing firms

---

## ATR-based Position Sizing (industry standard)

```python
# Khi volatility cao → giảm lot size để giữ risk/trade cố định
volatility_scalar = ATR_baseline / ATR_current   # < 1 khi vol cao
adjusted_lot = base_lot * volatility_scalar

# Trong context EA Market:
# Phase 2 exposure_weight đã handle một phần này
# Phase 3 có thể tích hợp ATR scalar vào lot_multiplier
```

---

## Survival score formula

```python
survival_score = 0.5 * win_rate + 0.5 * LEAST(profit_factor / 2.0, 1.0)
# Lookback: 180 ngày per regime | Min trades: 20
```

---

## Performance score — Phase 3 ranking

```python
performance_score = (
    0.40 * profit_factor_score +   # LEAST(profit_factor/2.0, 1.0)
    0.30 * net_pnl_score       +   # normalize trong pool về [0,1]
    0.20 * win_rate            +
    0.10 * return_stability_score
)
# Rank → bottom 20% → lot×0.5 | top 20% → lot×1.5
```

---

## MySQL Performance

```sql
CREATE INDEX idx_orders_login_close ON Orders(login, closeTime);
CREATE INDEX idx_orders_login_type  ON Orders(login, orderType, closeTime);
-- Window functions: luôn ORDER BY closeTime, không ORDER BY id
```

---

## Data reliability thresholds

| Metric | Min |
|---|---|
| Profit factor, win rate | 30 trades |
| Survival score per regime | 20 trades |
| STDDEV stability | 8 tuần |

Thiếu data → `survival_score = 0.5` (neutral default)

---

## Các cạm bẫy hay gặp

### Regime bias
- Nếu lịch sử 90% là `trend_low` → survival score `range_high` không tin cậy
- Luôn check `COUNT(trades per regime)` trước khi dùng score

### Survivorship bias
- EA mới (< 30 trades) không nên bị phạt health_score thấp sớm
- EA Hard Kill: keep Orders data, exclude khỏi portfolio analysis

### Concentration risk
- 1 lệnh chiếm > 50% total profit → warning flag

### Walk-forward validation (industry standard)
- Đừng optimize strategy trên toàn bộ history
- Dùng rolling window để validate: train N tháng, test M tháng tiếp theo
- EA Market dùng 180 ngày lookback → thực chất đây là walk-forward approach

---

## API & Backend best practices

- `eaCode` là identifier chính (không expose `login`)
- READ endpoints: cache 5-15 phút
- Hard/Soft Kill → Telegram webhook ngay lập tức
- `state = 'soft_kill'` → `effective_lot_mult = 0` (sync bắt buộc)
- Recovery từ Soft Kill → giữ `lot_multiplier = 0.5` thêm **2 tuần**
- Job order: Phase 1 (Sun 00:00) → Phase 2 (Mon 01:00) → Phase 3 (Mon 02:00) UTC

---

## 🔍 VERDICT TABLE — Tất cả thuật ngữ project vs Myfxbook/MQL5/Industry

| Thuật ngữ | Định nghĩa trong Project | Nguồn thực tế | Verdict |
|---|---|---|---|
| **Drawdown Level** | MAX peak-to-trough trên closed trades | Myfxbook: relative drawdown, bao gồm floating | ⚠️ Đúng concept, **thấp hơn Myfxbook** vì chỉ dùng closed trades |
| **Loss Acceleration** | Số lệnh thua liên tiếp gần nhất | Myfxbook: "Longest Losing Streak (LLS)" — metric chuẩn | ✅ Chuẩn, tên khác nhau nhưng concept đúng |
| **Profit Factor** | SUM(profit>0) / ABS(SUM(profit<0)) | Myfxbook/MT4: Gross Profit / Gross Loss (raw profit) | ✅ Hoàn toàn chuẩn |
| **Stability of Returns** | STDDEV(weekly_pnl) | Chuẩn quant: dùng trong Sharpe Ratio, prop firm consistency rules | ✅ Chuẩn |
| **Abnormal Execution** | Commission anomaly > 3× average | Industry: slippage detection, commission monitoring — chuẩn trong risk monitoring | ⚠️ Chuẩn hướng tiếp cận, **nhưng commission anomaly tốt hơn là dùng slippage** nếu có data |
| **Consistent Negative Returns** | Số tuần lỗ liên tiếp ≥ 2 | Circuit breaker standard trong MQL5: "stop after N consecutive losses" | ✅ Chuẩn |
| **Declining Performance Trend** | pf_recent vs pf_previous (-20%) | Walk-forward analysis concept — chuẩn trong quant trading | ✅ Chuẩn |
| **Structural Failure** | Worst single day PnL < -threshold | Daily drawdown limiter EA — chuẩn trong MQL5 community | ✅ Chuẩn |
| **Persistent Inability to Recover** | Ngày trong Soft Kill > 4 tuần | Circuit breaker với time-delay trip — chuẩn | ✅ Chuẩn |
| **Survival Score** | 0.5×win_rate + 0.5×LEAST(pf/2,1) | **Custom formula** — không có trong Myfxbook | ⚠️ Custom nhưng hợp lý. Chỉ dùng 2 metrics = hơi đơn giản |
| **Health Score (5 components)** | 0.30 DD + 0.20 LA + 0.25 PF + 0.15 Stab + 0.10 Exec | **Custom weighted formula** — không có trên Myfxbook/MQL5 | ⚠️ Custom — weights chưa được validate bằng data |
| **Regime Classification (ADX+ATR)** | ADX>25=trend, ATR>1.3×90d=high vol | Industry standard: ADX(14) là threshold phổ biến nhất | ✅ Chuẩn. Lưu ý: ADX không phân biệt up/down trend |
| **Exposure Weight (1.5/1.0/0.5)** | Gán theo survival_score buckets | Custom — không chuẩn Myfxbook | ⚠️ Custom nhưng hợp lý, cần backtest validate |
| **Lot Multiplier (top/mid/bottom 20%)** | 1.5/1.0/0.5 theo performance rank | Custom — concept capital concentration chuẩn nhưng thresholds custom | ⚠️ Custom |
| **Soft Kill / Hard Kill** | Tạm dừng / Xóa vĩnh viễn EA | MQL5: "circuit breaker" concept. Soft=pause, Hard=remove | ✅ Chuẩn concept, tên khác nhau |
| **Delta Max (DELTA_MAX = 0.3)** | Giới hạn thay đổi lot mỗi cycle | **Custom** — smoothing mechanism | ⚠️ Custom, hợp lý, cần validate |
| **MAX_PORTFOLIO_EXPOSURE** | Tổng effective_lot_mult tối đa | Custom risk guard — tương đương daily drawdown limiter | ✅ Hợp lý |
| **Win Rate** | COUNT(profit>0) / COUNT(*) | Myfxbook: exact formula này | ✅ Hoàn toàn chuẩn |

### Tóm tắt verdict:
- ✅ **Chuẩn hoàn toàn** (8/18): Profit Factor, Win Rate, Loss Acceleration (=LLS), Stability, Consistent Negative, Declining Trend, Structural Failure, Persistent Inability, Soft/Hard Kill concept
- ⚠️ **Custom nhưng hợp lý** (9/18): Drawdown (thấp hơn Myfxbook), Survival Score, Health Score weights, Exposure Weight buckets, Lot Multiplier tiers, Regime compound (ADX+ATR ok nhưng thiếu +DI/-DI), Abnormal Execution (hướng đúng), Delta Max, MAX_PORTFOLIO_EXPOSURE
- ❌ **Có vấn đề** (0/18): Không có term nào sai logic

### Vấn đề cần cân nhắc thêm:
1. **Survival Score** chỉ dùng 2 metrics — có thể thêm `expectancy` hoặc `recovery_factor`
2. **Health Score weights** (0.30/0.20/0.25/0.15/0.10) — hoàn toàn ad-hoc, chưa data-driven
3. **Abnormal Execution** dùng commission anomaly — nên bổ sung **slippage detection** nếu có `openPrice` vs expected price
