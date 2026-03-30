# Forex & EA Market — Domain Knowledge

> **Nguồn**: Tự học từ domain expertise + nghiên cứu industry best practices
> **Cập nhật**: 2026-03-19

---

## MT4/MT5 Orders — orderType mapping

> **⚠️ QUAN TRỌNG**: MQL4 và MQL5 có orderType khác nhau!

### MQL4 (OrderType) — dùng trong bảng `Orders` broker:

| orderType | Hằng số          | Ý nghĩa                  |
| --------- | ------------------ | -------------------------- |
| 0         | OP_BUY             | Buy market                 |
| 1         | OP_SELL            | Sell market                |
| 2         | OP_BUYLIMIT        | Buy Limit pending          |
| 3         | OP_SELLLIMIT       | Sell Limit pending         |
| 4         | OP_BUYSTOP         | Buy Stop pending           |
| 5         | OP_SELLSTOP        | Sell Stop pending          |
| 6         | *(undocumented)* | Balance/deposit/withdrawal |

### MQL5 (ENUM_ORDER_TYPE) — orders layer mới:

| value | Hằng số                 | Ý nghĩa                                     |
| ----- | ------------------------- | --------------------------------------------- |
| 0     | ORDER_TYPE_BUY            | Market buy                                    |
| 1     | ORDER_TYPE_SELL           | Market sell                                   |
| 6     | ORDER_TYPE_BUY_STOP_LIMIT | **Khác MQL4!** — không phải balance |
| 8     | ORDER_TYPE_CLOSE_BY       | Đóng position bằng lệnh ngược chiều    |

> **Mớ bảng `Orders` này dùng MQL4 format** (broker lưu lịch sử MT4).
> `orderType = 6` trong `Orders` = balance/nap/rut tiền → filter `IN (0, 1)` đúng.

---

## Broker data conventions

| Field           | Ý nghĩa            | Lưu ý                          |
| --------------- | -------------------- | -------------------------------- |
| `profit`      | Raw P&L từ position | Chưa trừ phí                  |
| `commision`   | Phí broker          | Luôn**âm** — 1 chữ 's' |
| `swap`        | Rollover overnight   | Có thể + hoặc −              |
| `volume`      | Lot size             | 1.0 = standard lot               |
| `closeTime`   | Unix ms              | = 0 nếu đang mở               |
| `magicNumber` | EA ID trong MT4      | Mỗi EA dùng magic riêng       |

```
net_pnl   = profit + commision + swap  ← drawdown, weekly P&L
gross_pnl = profit                     ← myfxbook profit_factor, win_rate
```

> **MQL4 API xác nhận**: `OrderProfit()` = raw profit, không bao gồm swap và commission.
> `OrderCommission()` = phí broker (luôn âm).  `OrderSwap()` = rollover (có thể + hoặc −).

## Drawdown — Industry Standards

| Metric                    | Công thức                       | Myfxbook |
| ------------------------- | --------------------------------- | -------- |
| Absolute drawdown         | initial_balance − lowest_balance | Không   |
| Relative (peak-to-trough) | (peak − trough) / peak           | ✅       |
| Max drawdown %            | max relative qua toàn history    | ✅       |

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

| Sharpe     | Đánh giá             |
| ---------- | ----------------------- |
| < 1.0      | Không đủ bù rủi ro |
| 1.0 – 2.0 | Tốt                    |
| > 2.0      | Rất tốt               |

### Recovery Factor

```
Recovery Factor = Net Profit / Max Drawdown (absolute)
```

> Myfxbook hiển thị Recovery Factor — EA tốt nên có RF > 3

### Profit Factor benchmarks (Myfxbook/MT4 standard)

| Profit Factor | Đánh giá |
| ------------- | ----------- |
| < 1.0         | Lỗ         |
| 1.0 – 1.40   | Biên mỏng |
| 1.41 – 2.00  | Tốt        |
| > 2.01        | Xuất sắc  |
