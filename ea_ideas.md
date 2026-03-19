# EA Market – Ý tưởng & Triết lý Thiết kế

> **Tài liệu này**: Lý do tại sao, tư duy đằng sau, và nguyên lý cốt lõi của hệ thống  
> **Đọc trước khi**: Đọc implementation guide hoặc bắt đầu code

---

## Vấn đề gốc rễ

Hầu hết hệ thống EA portfolio thất bại vì cùng một sai lầm:

> **Cố gắng tìm EA "tốt nhất" trong lịch sử — nhưng thị trường thay đổi.**

Kết quả:
- EA tốt trong quá khứ → fail khi regime thay đổi
- Portfolio bị một vài EA xấu kéo chìm toàn bộ
- Không có cơ chế bảo vệ khi EA bắt đầu suy giảm

---

## Tư duy chuyển đổi

| Tư duy cũ | Tư duy mới (EA Market) |
|---|---|
| Tìm EA tốt nhất | Loại bỏ EA nguy hiểm |
| Tối ưu hoá lợi nhuận | Tối ưu hoá sự sống còn |
| Chọn ít EA, tập trung | Giữ nhiều EA, điều tiết |
| Dự đoán thị trường | Phân loại thị trường hiện tại |
| Capital bằng nhau cho mọi EA | Capital theo hiệu suất thực tế |

---

## Metaphor: Hệ miễn dịch × Hệ sinh thái

**EA Market = Hệ sinh thái**, trong đó:
- Mỗi **EA** = một loài sinh vật
- **Market regime** = môi trường sống (thay đổi theo mùa)
- **Hệ thống** = hệ miễn dịch, không phải bộ chọn lọc

Hệ miễn dịch **không** chọn tế bào "tốt nhất" — nó **loại bỏ tế bào bệnh** và để phần còn lại tự phát triển.

```
Ecosystem view:
┌─────────────────────────────────────────────┐
│  EA_1  EA_2  EA_3  EA_4  EA_5  ...  EA_100  │
│   ↓     ↓     ↓     ↓     ↓         ↓       │
│ [Phase 1: immune system — loại nguy hiểm]   │
│        ↓                                    │
│ [Phase 2: regime filter — ưu tiên phù hợp]  │
│        ↓                                    │
│ [Phase 3: capital allocation — tập trung vốn]│
└─────────────────────────────────────────────┘
```

---

## Ý tưởng Phase 1 — Immune System

**Câu hỏi cốt lõi**: *"EA nào đang gây nguy hiểm cho hệ thống?"*

**Tư duy**: Không phải mọi EA thua lỗ đều cần bị xoá. Chỉ những EA có **hành vi tiêu cực dai dẳng** mới cần bị cô lập.

Phân biệt:
- EA thua vì **regime không phù hợp** → Soft Kill (chờ regime đổi)
- EA thua vì **cấu trúc bị hỏng** → Hard Kill (không thể phục hồi)

**Nguyên lý Soft Kill**: *Bảo tồn để phục hồi* — EA có thể tốt trong môi trường khác  
**Nguyên lý Hard Kill**: *Cắt lỗ không hối tiếc* — Loss > 70% peak balance là tín hiệu cấu trúc sai

---

## Ý tưởng Phase 2 — Regime Awareness

**Câu hỏi cốt lõi**: *"EA nào ít bị tổn thương nhất trong môi trường hiện tại?"*

**Tư duy**: Không dự đoán thị trường — chỉ **nhận ra** thị trường đang ở đâu.

4 regime = 4 "mùa":
- Trend-Low Vol = mùa xuân → EA trend-following phát triển
- Trend-High Vol = mùa bão → EA cần stop loss chặt
- Range-Low Vol = mùa thu → EA mean-reversion hiệu quả  
- Range-High Vol = mùa đông → EA nào cũng khó, giảm exposure

**Nguyên lý quan trọng**: EA bị giảm exposure **không bị tắt** — nó vẫn giao dịch, chỉ với lot nhỏ hơn. Vì nếu regime đổi tuần sau, nó sẽ phục hồi ngay.

---

## Ý tưởng Phase 3 — Capital Concentration

**Câu hỏi cốt lõi**: *"Vốn nên tập trung ở đâu để tối ưu hoá xác suất thắng?"*

**Tư duy**: Ngay cả khi 55% EA đang lãi và 45% đang lỗ — nếu 55% EA lãi đó **chiếm phần lớn vốn**, thì portfolio tổng thể thắng mạnh hơn rất nhiều.

```
Không có Phase 3:    EA_A(lỗ, lot=1) + EA_B(lãi, lot=1) = ngang nhau
Có Phase 3:          EA_A(lỗ, lot=0.5) + EA_B(lãi, lot=1.5) = lãi nhiều hơn
```

**Nguyên lý**: Không cần đoán EA nào thắng — chỉ cần **tập trung vốn vào EA đang thắng ngay lúc này**, và giảm vốn ở EA đang thua. Khi tình thế đổi, rebalance lại.

---

## Tại sao 3 Phase phải theo thứ tự?

```
Phase 1 (Loại nguy hiểm)
    ↓  output: danh sách EA còn sống (state = 'active')
Phase 2 (Điều chỉnh exposure)
    ↓  output: exposure_weight cho từng EA
Phase 3 (Phân bổ capital)
    ↓  output: effective_lot_multiplier = exposure_weight × lot_multiplier
```

Nếu bỏ Phase 1: Phase 3 sẽ cho EA nguy hiểm thêm vốn  
Nếu bỏ Phase 2: Phase 3 không biết regime nào đang ảnh hưởng  
Nếu bỏ Phase 3: Portfolio không tận dụng được strength của EA tốt

---

## Kỳ vọng thực tế

Hệ thống **không hứa hẹn**:
- Tìm ra EA nào sẽ thắng
- Dự đoán đúng market regime
- Tăng win rate

Hệ thống **thực sự mang lại**:
- Giảm thiệt hại từ EA nguy hiểm
- Tăng capital-weighted success rate (không tăng EA count)
- Portfolio ổn định hơn theo thời gian
- Tự động thích nghi khi regime thay đổi
