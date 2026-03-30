---
name: ea-market-sql-myfxbook-review

description: Quy tắc tự động áp dụng khi làm việc với EA Market system. Mọi SQL metrics phải theo chuẩn Myfxbook. Tự động review, không cần user nhắc lại.
---
# SKILL — Những gì USER đã train

> Chỉ chứa kiến thức USER trực tiếp dạy/sửa. Domain knowledge tự học → `forex_domain_knowledge.md`

---

## 🔴 QUY TẮC BẮT BUỘC — ĐỌC NGAY KHI BẮT ĐẦU CONVERSATION

### 0. Tự động bootstrap — KHÔNG cần user nhắc

**Mỗi conversation mới**, trước khi làm bất cứ thứ gì, PHẢI thực hiện theo thứ tự:


view_file các .md khác trong project 

> ⚠️ Không được bỏ qua bước này dù user không nhắc.

### 1. Luôn đọc lại TẤT CẢ file project trước khi làm task

**Bước 1**: Dùng `list_dir` discover file — KHÔNG hardcode tên
**Bước 2**: `view_file` từng `.md` tìm được trong project

### 2. Luôn review SQL theo chuẩn Myfxbook — không cần user nhắc

```
[ ] profit_factor  = SUM(profit>0) / ABS(SUM(profit<0))   ← raw profit, KHÔNG net_pnl
[ ] win_rate       = COUNT(profit>0) / COUNT(*)            ← raw profit, KHÔNG net_pnl
[ ] drawdown_pct   = MIN((cum_pnl-peak)/(initial+peak))   ← tại TỪNG điểm, không mix
[ ] drawdown window = INTERVAL 3000 DAY                   ← all history
[ ] filter         = AND orderType IN (0, 1)              ← LUÔN có
[ ] closeTime      = UNIX_TIMESTAMP(...) * 1000           ← milliseconds
[ ] field name     = commision (1 chữ 's')
```

### 3. Mỗi khi user dạy điều mới → cập nhật ngay file này

- Không cần user nhắc lại ở lần sau

---

## SQL chuẩn Myfxbook (user đã verify)

### Profit Factor ✅

```sql
SUM(CASE WHEN profit > 0 THEN profit ELSE 0 END) /
NULLIF(ABS(SUM(CASE WHEN profit < 0 THEN profit ELSE 0 END)), 0) AS profit_factor
```

### Win Rate ✅

```sql
SUM(CASE WHEN profit > 0 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS win_rate
```

### Max Drawdown % ✅

```sql
WITH c AS (
  SELECT closeTime,
         SUM(profit + COALESCE(commision,0) + COALESCE(swap,0))
           OVER (ORDER BY closeTime) AS cum_pnl
  FROM Orders WHERE login = {login} AND closeTime > 0
    AND orderType IN (0, 1)
    AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL 3000 DAY)*1000
),
p AS (SELECT cum_pnl, MAX(cum_pnl) OVER (ORDER BY closeTime) AS peak FROM c)
SELECT
  MIN(cum_pnl - peak)                                AS drawdown_abs,
  MIN((cum_pnl - peak) / ({initial_balance} + peak)) * 100 AS drawdown_pct
FROM p;
```

### Loss Acceleration ✅

```sql
SELECT COUNT(*) AS consecutive_losses
FROM (
  SELECT profit, ROW_NUMBER() OVER (ORDER BY closeTime DESC) AS rn
  FROM Orders WHERE login = {login} AND closeTime > 0 AND orderType IN (0, 1)
) ranked
WHERE rn < (
  SELECT COALESCE(MIN(rn), 999)
  FROM (SELECT profit, ROW_NUMBER() OVER (ORDER BY closeTime DESC) AS rn
        FROM Orders WHERE login = {login} AND closeTime > 0 AND orderType IN (0, 1)) t2
  WHERE profit > 0
);
```

---

## Lỗi user đã sửa — không được lặp lại

| ❌ Sai                                           | ✅ Đúng                                                                                              |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| `net_pnl > 0` để phân loại win/loss        | `profit > 0`                                                                                         |
| `MIN(DD) / MAX(peak)` — mix 2 thời điểm    | `MIN(DD/peak)` tại từng điểm                                                                     |
| Chia drawdown cho `initial_balance` cố định | Chia cho `initial_balance + peak_cum_pnl`                                                            |
| `INTERVAL 30 DAY` cho drawdown                 | `INTERVAL 3000 DAY`                                                                                  |
| Thiếu `AND orderType IN (0, 1)`               | Luôn filter                                                                                           |
| `commission` (2 's')                           | `commision` (1 's')                                                                                  |
| Không nhân `* 1000` cho closeTime            | `UNIX_TIMESTAMP(...) * 1000`                                                                         |
| Cho rằng MQL5 orderType 6 = balance             | MQL5 type 6 = BUY_STOP_LIMIT.**Project dùng MQL4 format**, chỉ type 6 của MQL4 mới = balance |
| So sánh drawdown 1:1 với Myfxbook              | Myfxbook tính cả floating P&L (lệnh đang mở). Project chỉ dùng closed trades → sẽ thấp hơn  |

---

## Project context

**Login mẫu**: `193583` | **initial_balance**: `5000 USD`
**DB**: MySQL, database `EA`, bảng `Orders`, không được insert, update database
**MCP**: `mysql://query/...`

---

## My Settings

## Cấu trúc dữ liệu cốt lõi

### Bảng `Orders` (nguồn duy nhất)

```sql
-- Filter bắt buộc cho mọi query
WHERE login = {login}
  AND closeTime IS NOT NULL AND closeTime > 0
  AND orderType IN (0, 1)   -- 0=Buy, 1=Sell only
```

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

## EA Market – Term Definitions (Developer Reference)

> **Nguồn dữ liệu duy nhất**: Bảng `Orders` — query theo `login`
> **Input chuẩn cho mọi công thức**:
>
> ```sql
> SELECT profit, commision, swap, openTime, closeTime, volume, openPrice, closePrice
> FROM Orders
> WHERE login = 193583
>   AND closeTime IS NOT NULL AND closeTime > 0
>   AND orderType IN (0, 1)
>   AND closeTime >= UNIX_TIMESTAMP(NOW() - INTERVAL {N} DAY) * 1000
> ORDER BY closeTime ASC;
> ```
> → Gọi kết quả này là **`closed_orders`** (window N ngày gần nhất, mặc định N=300)
