# ADR-NNNN — <Tiêu đề ngắn>

Cấu trúc bắt buộc: đúng 12 mục dưới đây, theo thứ tự. Mọi ADR mới bắt đầu ở PENDING_OWNER_APPROVAL; chỉ chủ sở hữu mới được chuyển trạng thái. Độ dài khuyến nghị 60-110 dòng.

## 1. Metadata

- **ID:** ADR-NNNN
- **Ngày:** YYYY-MM-DD
- **Decision ID:** D-NNN (trong `DECISION_LOG.md`)
- **Decision status:** PENDING_OWNER_APPROVAL
- **Owner approval required:** yes
- **Approved by/date:** (để trống cho tới khi chủ sở hữu duyệt)
- **Nguồn:** <đường dẫn tài liệu và mục>

## 2. Context

- <Bối cảnh, ràng buộc, mâu thuẫn giữa các tài liệu nếu có.>

## 3. Problem

- <Vấn đề cần quyết định và câu hỏi còn mở.>

## 4. Options considered

(Tối thiểu 2 phương án; mỗi phương án bị loại phải có lý do.)

### Phương án A — <tên> [ĐỀ XUẤT]

<Mô tả.>
- **Lý do chọn đề xuất:** <...>

### Phương án B — <tên> [Bị loại]

<Mô tả.>
- **Lý do loại:** <...>

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**.

- <Nội dung đề xuất cụ thể, phạm vi, điểm còn mở (CF-xxx).>

## 6. Rationale

- <Lý do dựa trên bằng chứng.>

## 7. Consequences

**Tích cực**
- <...>

**Tiêu cực**
- <...>

**Trung tính**
- <...>

## 8. Security implications

- <Bề mặt tấn công, rủi ro còn lại, biện pháp giảm.>

## 9. Cost and operational implications

- <Chi phí xây dựng, vận hành, quota/tiền, công bảo trì.>

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| <claim> | `<đường dẫn tài liệu và mục>` | Đã xác minh / Giả định / Chưa kiểm chứng |

Quy tắc: chỉ ghi "Đã xác minh" khi đã đọc mã/chạy lệnh thật hoặc QA đã đối chiếu; không nâng mức so với nguồn; ghi rõ nếu QA không chạy lại live test.

**Mức bằng chứng tổng hợp:** Verified / Partial / Unverified (giải thích một câu).

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- <điều cần xác nhận>
- **Khả năng đảo ngược:** <dễ/trung bình/khó và vì sao>
- **Bị chặn trong lúc chờ duyệt:** <việc không được làm>

## 12. Reversal conditions

- <Sự kiện hoặc số đo cụ thể làm quyết định phải xem lại.>
