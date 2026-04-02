---
name: ea-market

description: Quy tắc tự động áp dụng khi làm việc với EA Market system. Mọi SQL metrics phải theo chuẩn Myfxbook. Tự động review, không cần user nhắc lại.
---
# SKILL — Những gì USER đã train

> Chỉ chứa kiến thức USER trực tiếp dạy/sửa. Domain knowledge tự học → `forex_domain_knowledge.md` nếu có

---
review 
với toàn project chỉ quan tâm đến eaCode. đây là hệ thống phân tích hiệu chỉnh eaCode mà

hiện dữ liệu bảng Orders phục vụ cho việc tính các thông tin eaCode (dùng login chẳng qua cho môi trường dev để lấy dữ liệu trade orders thôi) nên việc ứng dụng kết quả vào logic mới sau này bạn không được dùng login phải dùng eaCode.

ngay kể base lot hay regime của cũng là với eaCode chứ không liên quan đến login.

tôi yêu cầu bạn thêm logic này vào rule skill
- với toàn project chỉ quan tâm đến eaCode không được dùng liên quan đến login trong các bảng mới. 
_ việc login update date lot realtime hay vào lệnh như nào bạn không cần quan tâm. sẽ có bên team khác làm việc này. chỉ cần chịu nhiệm vụ tính lot và cập nhật database 

_new note


XỬ LÝ trường HỢP rule thay đổi. nếu tôi đổi rule đổi config thì sao.
code để maintain và update nếu rule đổi.

-khi thêm tính năng hay đổi config hay là thêm phase mới hay là đổi cycle update không phải là mỗi tuần mà với eacode thì update hàng ngày có ea lại update sau 4 giờ

_ phase 2 đáp ứng với target Build a centralized system to collect historical price data (5 symbols)

_ trong quá trình viết technical_proposal và thiết kế mọi thứ như database, code, ... bạn phải thiết kế sao cho System must support PnL simulation for applying phase 2, phase 3 1 cách dễ nhất.



## 🔴 QUY TẮC BẮT BUỘC — ĐỌC NGAY KHI BẮT ĐẦU CONVERSATION

### 0. Tự động bootstrap — KHÔNG cần user nhắc

**Mỗi conversation mới**, trước khi làm bất cứ thứ gì, PHẢI thực hiện theo thứ tự:


view_file các .md khác trong project 

> ⚠️ Không được bỏ qua bước này dù user không nhắc.

### 1. Luôn đọc lại TẤT CẢ file project trước khi làm task

**Bước 1**: Dùng `list_dir` discover file — KHÔNG hardcode tên
**Bước 2**: `view_file` từng `.md` tìm được trong project



### 3. Mỗi khi user dạy điều mới → cập nhật ngay file này

- Không cần user nhắc lại ở lần sau

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

Mỗi EA trong hệ thống có một **`eaCode`** là identifier chính. Mỗi `eaCode` được liên kết với **đúng một `login`** (tài khoản broker). Tuy nhiên, sau này**một `login` có thể chứa nhiều `eaCode`**. nên không được dựa vào login để tính toán sau này

```
eaCode  (identifier chính của hệ thống)
  └──► login  (tài khoản broker, 1 login có thể có nhiều eaCode)
           └──► SELECT * FROM Orders WHERE login = {login}
                    └── Toàn bộ lịch sử lệnh của login đó,
                        đại diện cho eaCode tương ứng
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

technical_proposal cần có các phần sau.
_Backend (API Service)
_Database
_Technology Stack Details
_Primary Processing Flows: Flow Description:
_System Startup Flow. 

System must support PnL simulation for applying phase 2, phase 3 
nghĩa là khi áp thay đổi lot vào, ví dụ lịch sử không phải hiện tại thì kết quả ea sẽ chạy tốt hơn hay kém hơn. chứ tuần hiện tại hay tuần mới thì cần gì simulation

_ bạn là senior backend 

---

## Senior Backend Design Rules (từ review technical_proposal)

> Áp dụng mọi khi thiết kế DB schema và system architecture.

**Rule S6: Config không hardcode trong code**
Y NHƯ TRONG PROPOSAL
VÍ DỤ Parameter:
● SK2_MIN_SLOPE_PCT
### Simulation