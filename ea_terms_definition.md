# EA Market – Term Definitions (Developer Reference)

> **Nguồn dữ liệu duy nhất**: Bảng `Orders` — query theo `login` (lấy từ `EA_Registry` theo `eaCode`)  
> **Input chuẩn cho mọi công thức**:
> ```sql
> SELECT profit, commision, swap, openTime, closeTime, volume, openPrice, closePrice
> FROM Orders
> WHERE login = {login}
>   AND closeTime IS NOT NULL AND closeTime > 0
>   AND orderType IN (0, 1)
>   AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL {N} DAY) * 1000
> ORDER BY closeTime ASC;
> ```
> → Gọi kết quả này là **`closed_orders`** (window N ngày gần nhất, mặc định N=30)

---

## Ký hiệu dùng chung

| Ký hiệu | Ý nghĩa |
|---|---|
| `net_pnl[i]` | `profit[i] + commision[i] + swap[i]` — lãi/lỗ thực của lệnh i |
| `cum_pnl[i]` | `SUM(net_pnl[0..i])` — PnL tích luỹ đến lệnh i |
| `weekly_pnl[w]` | `SUM(net_pnl)` trong tuần w (group by `YEARWEEK(closeTime/1000)`) |
| `peak[i]` | `MAX(cum_pnl[0..i])` — đỉnh PnL tích luỹ đến lệnh i |
| `N` | Lookback window tính bằng ngày (mặc định 30) |

---

## Phase 1 — Thuật ngữ liên quan đến sức khoẻ EA

---

### 1. Drawdown Level

**Định nghĩa**: Mức sụt giảm lớn nhất của PnL tích luỹ so với đỉnh trước đó, trong cửa sổ N ngày.

**Công thức**:
```
drawdown[i] = cum_pnl[i] - peak[i]          ← luôn ≤ 0
drawdown_level = MIN(drawdown[i])            ← giá trị âm lớn nhất
```

**Dạng %**:
```
-- Chia theo peak balance thực tế, không phải initial_balance cố định
drawdown_pct = |drawdown_level| / (initial_balance + peak_cum_pnl) × 100
```
> ⚠️ Không dùng `|DD| / initial_balance` vì tài khoản đã tăng vốn qua thời gian.

**SQL** (dùng `INTERVAL 3000 DAY` để cover toàn bộ lịch sử):
```sql
WITH c AS (
  SELECT closeTime,
         SUM(profit + COALESCE(commision,0) + COALESCE(swap,0))
           OVER (ORDER BY closeTime) AS cum_pnl
  FROM Orders WHERE login = 193583 AND closeTime > 0
    AND orderType IN (0, 1)
    AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 3000 DAY)*1000
),
p AS (
  SELECT cum_pnl,
         MAX(cum_pnl) OVER (ORDER BY closeTime) AS peak_cum_pnl
  FROM c
)
SELECT
  MIN(cum_pnl - peak_cum_pnl)                                       AS drawdown_level,
  MIN((cum_pnl - peak_cum_pnl) / (5000 + peak_cum_pnl)) * 100      AS drawdown_pct
FROM p;
-- 5000 = initial_balance, thay theo login tương ứng
-- drawdown_pct tính tại từng điểm: (trough - peak) / (initial + peak)
-- → lấy MIN để ra worst-case đúng với peak tương ứng của nó
```

**Ngưỡng gợi ý**:
- `drawdown_pct > 30%` → xét Soft Kill
- `drawdown_pct > 70%` → Hard Kill ngay

---

### 2. Loss Acceleration

**Định nghĩa**: Số lệnh thua **liên tiếp** gần nhất, tính từ lệnh mới nhất ngược về. Thể hiện tốc độ xấu đi của EA.

**Công thức**:
```
consecutive_losses = số lệnh thua liên tiếp gần nhất (net_pnl < 0)
→ đếm từ lệnh cuối cùng ngược lại, dừng khi gặp lệnh lãi đầu tiên
```

**SQL**:
```sql
SELECT COUNT(*) AS consecutive_losses
FROM (
  SELECT profit,
         ROW_NUMBER() OVER (ORDER BY closeTime DESC) AS rn
  FROM Orders WHERE login = 193583 AND closeTime > 0
    AND orderType IN (0, 1)
) ranked
WHERE rn < (
  SELECT COALESCE(MIN(rn), 999)
  FROM (
    SELECT profit,
           ROW_NUMBER() OVER (ORDER BY closeTime DESC) AS rn
    FROM Orders WHERE login = 193583 AND closeTime > 0
      AND orderType IN (0, 1)
  ) t2
  WHERE profit > 0   -- phan loai theo raw profit (chuan myfxbook)
);
```

**Score**: `loss_acceleration_score = 1 - LEAST(consecutive_losses / 10.0, 1.0)`

**Ngưỡng gợi ý**: `consecutive_losses ≥ 5` → xét Soft Kill

---

### 3. Profit Factor

**Định nghĩa**: Tỷ lệ giữa tổng lãi và tổng lỗ (không tính lệnh hoà). Đo lường hiệu quả tổng thể.

**Công thức (chuẩn Myfxbook)**:
```
profit_factor = SUM(profit khi profit > 0) / ABS(SUM(profit khi profit < 0))
```
> Dùng **raw `profit`** (không gồm commision/swap) — đúng với cách Myfxbook và MT4 tính.

**SQL**:
```sql
SELECT
  SUM(CASE WHEN profit > 0 THEN profit ELSE 0 END) /
  NULLIF(ABS(SUM(CASE WHEN profit < 0 THEN profit ELSE 0 END)), 0)
    AS profit_factor
FROM Orders
WHERE login = 193583 AND closeTime > 0
  AND orderType IN (0, 1)
  AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 30 DAY)*1000;
```

**Diễn giải**:
- `profit_factor > 1.5` → tốt
- `profit_factor < 1.0` → lỗ nhiều hơn lãi → xét Soft Kill
- `profit_factor < 0.7` → nguy hiểm

**Score**: `profit_factor_score = LEAST(profit_factor / 2.0, 1.0)`

---

### 4. Stability of Returns

**Định nghĩa**: Độ ổn định của PnL theo tuần. Đo bằng độ lệch chuẩn của `weekly_pnl`. Càng thấp càng ổn định.

**Công thức**:
```
stability = STDDEV(weekly_pnl)  -- tính trên N tuần gần nhất (VD: 12 tuần)
```

**SQL**:
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
```

**Score**: `stability_score = 1 - LEAST(pnl_stddev / STABILITY_THRESHOLD, 1.0)`  
(`STABILITY_THRESHOLD` = ngưỡng do operator định nghĩa, VD: 500 USD)

---

### 5. Abnormal Execution Behaviour

**Định nghĩa**: Tỷ lệ lệnh có `commission` bất thường (vượt 3× trung bình). Thể hiện vấn đề kỹ thuật hoặc broker.

**Công thức**:
```
avg_commision = AVG(|commision|) trên toàn bộ lệnh
anomaly_count = COUNT lệnh có |commision| > avg_commision × 3
anomaly_rate  = anomaly_count / total_orders
```

**SQL**:
```sql
SELECT
  SUM(CASE WHEN ABS(commision) > avg_c * 3 THEN 1 ELSE 0 END) AS anomaly_count,
  COUNT(*) AS total_orders,
  SUM(CASE WHEN ABS(commision) > avg_c * 3 THEN 1 ELSE 0 END) * 1.0
    / COUNT(*) AS anomaly_rate
FROM (
  SELECT commision,
         AVG(ABS(commision)) OVER () AS avg_c
  FROM Orders WHERE login = 193583 AND closeTime > 0
    AND orderType IN (0, 1)
    AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 30 DAY)*1000
) t;
```

**Score**: `execution_score = 1 - anomaly_rate`

---

### 6. Consistent Negative Returns

**Định nghĩa**: Số tuần **liên tiếp** gần nhất mà `weekly_pnl < 0`.

**Công thức**:
```
Lấy danh sách weekly_pnl sắp xếp theo tuần giảm dần
→ đếm từ tuần mới nhất ngược lại, dừng khi gặp tuần lãi đầu tiên
```

**SQL**:
```sql
SELECT COUNT(*) AS consecutive_loss_weeks
FROM (
  SELECT wk, weekly_pnl,
         ROW_NUMBER() OVER (ORDER BY wk DESC) AS rn
  FROM (
    SELECT YEARWEEK(FROM_UNIXTIME(closeTime/1000)) AS wk,
           SUM(profit + COALESCE(commision,0) + COALESCE(swap,0)) AS weekly_pnl
    FROM Orders WHERE login = 193583 AND closeTime > 0
      AND orderType IN (0, 1)
    GROUP BY wk
  ) w
) ranked
WHERE rn < (
  SELECT COALESCE(MIN(rn), 999)
  FROM (
    SELECT weekly_pnl,
           ROW_NUMBER() OVER (ORDER BY wk DESC) AS rn
    FROM (
      SELECT YEARWEEK(FROM_UNIXTIME(closeTime/1000)) AS wk,
             SUM(profit + COALESCE(commision,0) + COALESCE(swap,0)) AS weekly_pnl
      FROM Orders WHERE login = 193583 AND closeTime > 0
        AND orderType IN (0, 1)
      GROUP BY wk
    ) w2
  ) r2
  WHERE weekly_pnl > 0
);
```

**Ngưỡng gợi ý**: `consecutive_loss_weeks ≥ 2` → trigger Soft Kill

---

### 7. Declining Performance Trend

**Định nghĩa**: So sánh profit_factor của **2 tuần gần nhất** với **4 tuần trước đó**. Nếu giảm → trend xấu đi.

**Công thức**:
```
pf_recent   = profit_factor(last 2 weeks)
pf_previous = profit_factor(weeks 3–6)
declining   = (pf_recent < pf_previous × 0.8)   ← giảm hơn 20%
```

**Cách tính**: Chạy `profit_factor query` hai lần với `closeTime` window khác nhau rồi so sánh.

**Ngưỡng gợi ý**: Nếu `declining = true` và `pf_recent < 1.0` → xét Soft Kill

---

### 9. Structural Failure

**Định nghĩa**: Mức lỗ cực đoan trong **một khoảng thời gian rất ngắn** (VD: 1 ngày). Thể hiện lỗi logic hoặc cấu trúc của EA.

**Công thức**:
```
daily_pnl[d] = SUM(net_pnl) trong ngày d
structural_failure = (MIN(daily_pnl) < -STRUCTURAL_THRESHOLD)
```

**SQL**:
```sql
SELECT MIN(daily_pnl) AS worst_day
FROM (
  SELECT DATE(FROM_UNIXTIME(closeTime/1000)) AS d,
         SUM(profit + COALESCE(commision,0) + COALESCE(swap,0)) AS daily_pnl
  FROM Orders WHERE login = 193583 AND closeTime > 0
    AND orderType IN (0, 1)
    AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 30 DAY)*1000
  GROUP BY d
) daily;
```

**Ngưỡng gợi ý**: `worst_day < -STRUCTURAL_THRESHOLD` (VD: -30% balance trong 1 ngày) → Hard Kill

---

### 10. Persistent Inability to Recover

**Định nghĩa**: EA đã ở trạng thái **Soft Kill** trong K tuần liên tiếp mà `health_score` vẫn không vượt ngưỡng recovery.

**Cách tính**: Đọc từ bảng `EA_State_Log`, không cần query `Orders`:

```sql
SELECT DATEDIFF(NOW(), MIN(changed_at)) AS days_in_soft_kill
FROM EA_State_Log
WHERE eaCode = 'EA_GOLD_001'
  AND new_state = 'soft_kill'
  AND id = (SELECT MAX(id) FROM EA_State_Log WHERE eaCode = 'EA_GOLD_001' AND new_state = 'soft_kill');
```

**Ngưỡng gợi ý**: Ở Soft Kill > 4 tuần mà health_score < 0.3 → Hard Kill

---

## Phase 2 — Thuật ngữ liên quan đến Regime Exposure

---

### 11. Increased / Reduced Exposure

**Định nghĩa**: Hệ số nhân áp lên `base_lot` của EA dựa trên mức độ phù hợp với regime hiện tại.

**Công thức**:
```
exposure_weight = f(survival_score tại regime hiện tại)

survival_score ≥ 0.70  →  exposure_weight = 1.5   (Increased)
survival_score ≥ 0.45  →  exposure_weight = 1.0   (Normal)
survival_score < 0.45  →  exposure_weight = 0.5   (Reduced)
```

**Tác động thực tế**:
```
actual_lot = base_lot × exposure_weight
```

---

### 12. Greater Influence

**Định nghĩa**: EA có `exposure_weight > 1.0` thì mỗi lệnh thắng/thua của nó **ảnh hưởng nhiều hơn** đến PnL tổng portfolio.

**Cách đo**:
```
portfolio_pnl_contribution[eaCode] = SUM(net_pnl) × exposure_weight
```
→ EA có `exposure_weight = 1.5` đóng góp/gây hại nhiều hơn 50% so với EA `exposure_weight = 1.0`.

---

### 13. Weighting Mechanism

**Định nghĩa**: Thuật toán chuyển đổi `survival_score → exposure_weight`. Là một hàm phân ngưỡng đơn giản:

```python
def weighting_mechanism(survival_score: float) -> float:
    if survival_score >= 0.70:   return 1.5
    elif survival_score >= 0.45: return 1.0
    else:                        return 0.5
```

**Input**: `survival_score` từ `EA_Regime_Profile(eaCode, current_regime)`  
**Output**: `exposure_weight` ghi vào `EA_Allocation(eaCode)`

---

### 14. Unsuitable EAs Remain Available

**Định nghĩa**: EA có `exposure_weight = 0.5` (reduced) vẫn **tiếp tục giao dịch**, không bị tắt. Chỉ giảm lot.

**Lý do**: Nếu regime thay đổi tuần tới, EA này có thể trở nên phù hợp → không cần khởi động lại.

**Implement**: Không cần logic đặc biệt — EA vẫn nhận lệnh, chỉ áp `actual_lot = base_lot × 0.5`.

---

### 15. Regime Profile Lookback

**Định nghĩa**: Khoảng thời gian lịch sử dùng để tính `survival_score` cho từng regime.

**Giá trị gợi ý**: **6 tháng** (~180 ngày)

**Lý do**: Ngắn hơn → dễ overfitting (ít data). Dài hơn → regime cũ quá (không còn relevant).

**Implement trong query**:
```sql
AND o.closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 180 DAY) * 1000
```

**Lưu ý**: Cần đủ số lệnh trong mỗi regime (VD: tối thiểu 20 lệnh) để score có ý nghĩa thống kê.

---

## Phase 3 — Thuật ngữ liên quan đến Health Score & Capital Allocation

---

### 16. Drawdown Stability (Health Score component)

**Định nghĩa**: Thành phần đo mức độ kiểm soát drawdown trong Health Score.

**Công thức**:
```
-- sử dụng drawdown_pct từ Term #1 (mỹ phầm cả peak balance)
drawdown_stability = 1 - LEAST(ABS(drawdown_pct) / 100, 1.0)
-- drawdown_pct ∈ [-100%, 0%] → ABS(drawdown_pct)/100 ∈ [0, 1]
```

- `drawdown_stability = 1.0` → không có drawdown
- `drawdown_stability = 0.0` → drawdown = 100% peak balance

**Input**: `drawdown_pct` từ Term #1 (dùng peak balance, không phải initial_balance cố định)

---

### 17. Profitability Metrics (Health Score component)

**Định nghĩa**: Thành phần đo hiệu quả sinh lời trong Health Score. Kết hợp `profit_factor` và `win_rate`.

**Công thức**:
```
win_rate     = COUNT(profit > 0) / COUNT(*) × 100%
pf_score     = LEAST(profit_factor / 2.0, 1.0)
profitability = 0.6 × pf_score + 0.4 × (win_rate / 100)
```
> Dùng **raw `profit`** để phân loại (chuẩn Myfxbook)

**SQL win_rate**:
```sql
SELECT
  SUM(CASE WHEN profit > 0 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS win_rate
FROM Orders WHERE login = 193583 AND closeTime > 0
  AND orderType IN (0, 1)
  AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 30 DAY)*1000;
```

---

### 18. Regime Compatibility (Health Score component)

**Định nghĩa**: Thành phần đo mức độ phù hợp của EA với **regime hiện tại**, dựa trên `survival_score` tại regime đó.

**Công thức**:
```
regime_compatibility = EA_Regime_Profile[eaCode][current_regime].survival_score
```

**Input**: Đọc trực tiếp từ bảng `EA_Regime_Profile` — không cần query `Orders` thêm.

---

### 19. Execution Quality

**Định nghĩa**: Thành phần đo chất lượng thực thi lệnh. Dựa trên `execution_score` từ Term #5 (Abnormal Execution Behaviour).

**Công thức**:
```
execution_quality = 1 - anomaly_rate
```

**Input**: `anomaly_rate` từ query Term #5

---

### 20. Health Score Formula

**Định nghĩa**: Điểm sức khoẻ tổng hợp của EA, tính từ 5 thành phần. Giá trị ∈ [0.0, 1.0].

**Công thức** (5 components — chạy ở Phase 1, trước Phase 2):
```
health_score =
    0.30 × drawdown_score             (Term #1: 1 - ABS(drawdown_pct/100))
  + 0.20 × loss_acceleration_score   (Term #2: 1 - LEAST(consecutive_losses/10, 1.0))
  + 0.25 × profit_factor_score       (Term #3: LEAST(profit_factor/2.0, 1.0))
  + 0.15 × return_stability_score    (Term #4: 1 - LEAST(pnl_stddev/threshold, 1.0))
  + 0.10 × execution_normalcy_score  (Term #5: 1 - anomaly_rate)

——————————————————————————————————————————————————
 Tiễu chú: không dùng regime_compatibility ở Phase 1
 vì Phase 1 chạy TRƯỚC Phase 2 — chưa có survival_score.
——————————————————————————————————————————————————
```

**Diễn giải**:
- `health_score ≥ 0.65` → Active, khoẻ mạnh
- `0.3 ≤ health_score < 0.65` → Soft Kill
- `health_score < 0.3` (kéo dài) → Hard Kill

---

### 21. Portfolio Risk

**Định nghĩa**: Tổng mức độ tiếp xúc vốn (total exposure) của toàn bộ EA đang `active`, tính bằng tổng `effective_lot_mult`.

**Công thức**:
```
portfolio_risk = SUM(effective_lot_mult) -- cho tất cả eaCode có state='active'
```

**SQL**:
```sql
SELECT SUM(ea.effective_lot_mult) AS portfolio_risk
FROM EA_Allocation ea
JOIN EA_Health eh ON ea.eaCode = eh.eaCode
WHERE eh.state = 'active';
```

**Dùng để**: So sánh với `risk_constant` (Term #22) để quyết định có cần scale down không.

---

### 22. Risk Constant

**Định nghĩa**: Ngưỡng tối đa cho phép của `portfolio_risk`. Là một **hằng số cố định** do operator cấu hình.

**Ký hiệu**: `MAX_PORTFOLIO_EXPOSURE`

**Ví dụ**:
```
Nếu có 100 EA, base_lot = 1.0, max exposure = 120 units
→ MAX_PORTFOLIO_EXPOSURE = 120
→ Nếu SUM(effective_lot_mult) > 120 → scale down tất cả
```

**Implement**:
```python
MAX_PORTFOLIO_EXPOSURE = 120   # config file, không hard-code trong logic

if portfolio_risk > MAX_PORTFOLIO_EXPOSURE:
    scale_factor = MAX_PORTFOLIO_EXPOSURE / portfolio_risk
    # scale_factor × effective_lot_mult cho mọi EA
```

---

### 23. Smooth and Conservative

**Định nghĩa**: Nguyên tắc giới hạn mức thay đổi `lot_multiplier` mỗi lần rebalance để tránh đột biến exposure.

**Công thức**:
```
new_lot_multiplier = CLAMP(
    calculated_lot_multiplier,
    old_lot_multiplier - DELTA_MAX,
    old_lot_multiplier + DELTA_MAX
)

DELTA_MAX = 0.3  (gợi ý)
```

**Ví dụ**:
```
old_lot_multiplier = 1.0
calculated = 1.8   →  new = 1.3  (bị giới hạn +0.3)
calculated = 0.3   →  new = 0.7  (bị giới hạn -0.3)
```

**Implement**:
```python
DELTA_MAX = 0.3

def smooth_lot(old: float, new: float) -> float:
    return max(old - DELTA_MAX, min(old + DELTA_MAX, new))
```

**Lý do**: Thay đổi lot đột ngột có thể gây rủi ro portfolio không lường trước được khi nhiều EA thay đổi cùng lúc.

---

## Tóm tắt — Bảng tra nhanh

| STT | Thuật ngữ | Input từ Orders | Output | Dùng ở đâu |
|---|---|---|---|---|
| 1 | Drawdown level | `profit, commision, swap, closeTime` | `drawdown_pct` (%) | Phase 1 trigger |
| 2 | Loss acceleration | `profit, commision, swap, closeTime` | `consecutive_losses` | Phase 1 trigger |
| 3 | Profit factor | `profit, commision, swap, closeTime` | `profit_factor` (ratio) | Phase 1 score |
| 4 | Stability of returns | `profit, commision, swap, closeTime` | `pnl_stddev` | Phase 1 score |
| 5 | Abnormal execution | `commision, closeTime` | `anomaly_rate` | Phase 1 score |
| 6 | Consistent negative returns | `profit, commision, swap, closeTime` | `consecutive_loss_weeks` | Phase 1 trigger |
| 7 | Declining performance trend | `profit, commision, swap, closeTime` | `declining` (bool) | Phase 1 trigger |
| 9 | Structural failure | `profit, commision, swap, closeTime` | `worst_day_pnl` | Phase 1 Hard Kill |
| 10 | Persistent inability to recover | `EA_State_Log` | `days_in_soft_kill` | Phase 1 Hard Kill |
| 11 | Increased/Reduced exposure | `EA_Regime_Profile.survival_score` | `exposure_weight` | Phase 2 output |
| 12 | Greater influence | `exposure_weight` | portfolio contribution | Phase 2 concept |
| 13 | Weighting mechanism | `survival_score` | `exposure_weight` | Phase 2 algorithm |
| 14 | Unsuitable EAs remain available | `EA_Health.state` | — (design rule) | Phase 2 rule |
| 15 | Regime Profile lookback | — | 180 ngày (config) | Phase 2 config |
| 16 | Drawdown stability | `drawdown_level` (Term 1) | component [0,1] | Health Score |
| 17 | Profitability metrics | `profit_factor` + `win_rate` | component [0,1] | Health Score |
| 18 | Regime compatibility | `EA_Regime_Profile` | component [0,1] | Health Score |
| 19 | Execution quality | `anomaly_rate` (Term 5) | component [0,1] | Health Score |
| 20 | Health Score formula | Terms 16–19 | `health_score` [0,1] | Phase 1 + Phase 3 |
| 21 | Portfolio risk | `EA_Allocation.effective_lot_mult` | total units | Phase 3 guard |
| 22 | Risk constant | — | `MAX_PORTFOLIO_EXPOSURE` (config) | Phase 3 guard |
| 23 | Smooth and conservative | `old_lot_multiplier` | `new_lot_multiplier` | Phase 3 rebalance |
