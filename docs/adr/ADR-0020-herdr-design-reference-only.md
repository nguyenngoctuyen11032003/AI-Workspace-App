# ADR-0020 — herdr là tham chiếu thiết kế; HerdrRuntime chỉ là tùy chọn sau PoC

## 1. Metadata

- **ID:** ADR-0020
- **Ngày:** 2026-10-30
- **Decision ID:** D-020 (trong `DECISION_LOG.md`)
- **Decision status:** PENDING_OWNER_APPROVAL
- **Owner approval required:** yes
- **Approved by/date:** (để trống, chủ sở hữu chưa duyệt)
- **Nguồn:** research/HERDR_ANALYSIS.md §8, §9, design/HERDR_DESIGN_INTEGRATION.md

## 2. Context

- herdr (Apache-2.0, Rust) quan sát terminal của agent; AI WORKSPACE sở hữu phiên qua SDK/app-server nên có event có cấu trúc.
- Trên máy có `herdr 0.9.5` đang chạy, tạo cám dỗ dùng herdr làm runtime chính.

## 3. Problem

- Quyết định vai trò của herdr: tham chiếu hay phụ thuộc runtime.
- Repo herdr không có file NOTICE ở gốc; phiên bản 0.x thay đổi nhanh.

## 4. Options considered

### Phương án A — herdr là tham chiếu thiết kế; `HerdrRuntime` và terminal tương tác chỉ tùy chọn sau PoC (Phase 2 hoặc 7); không dùng mã Rust [ĐỀ XUẤT]

Heuristic màn hình không thay sự kiện SDK; áp dụng ý tưởng (attention state, Agent Rail, token, seq/source); manifest TOML và tài liệu dùng lại theo Apache-2.0 kèm NOTICE.
- **Lý do chọn đề xuất:** Dữ liệu có cấu trúc đáng tin hơn heuristic; giữ đường tích hợp tùy chọn.

### Phương án B — Dùng herdr làm runtime chính [Bị loại]

Có sẵn terminal runtime.
- **Lý do loại:** Chuyển từ event có cấu trúc sang heuristic màn hình, thêm phụ thuộc Rust/native, mất kiểm soát approval.

### Phương án C — Không tham khảo herdr [Bị loại]

Không phụ thuộc.
- **Lý do loại:** Mất các ý tưởng thiết kế giá trị (ADR-0017, 0018, 0019).

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- HerdrRuntime (chạy Claude Code/Codex dạng TUI trong pane herdr) chỉ làm sau PoC, tối thiểu Phase 2 hoặc 7.
- Không dùng `ghostty-vt`/ratatui hay mã Rust.
- Kiểm license asset âm thanh trước khi dùng; giữ attribution Apache-2.0.

## 6. Rationale

- AI WORKSPACE sở hữu phiên nên có dữ liệu đáng tin hơn heuristic.
- Giữ terminal tương tác như tính năng tùy chọn mà không lệch kiến trúc.
- Apache-2.0 cho phép dùng ý tưởng và dữ liệu manifest với attribution.

## 7. Consequences

**Tích cực**
- Kiến trúc không phụ thuộc herdr.
- Áp dụng chọn lọc ý tưởng tốt.

**Tiêu cực**
- Không có runtime terminal sẵn ở MVP.
- Phải tự cài đặt các mẫu thiết kế đã chọn.

**Trung tính**
- herdr 0.9.5 trên máy dev không phải phụ thuộc chính thức.
- Hành vi socket herdr trên Windows chưa thử.

## 8. Security implications

- Heuristic màn hình có thể bị agent/nội dung terminal đánh lừa (hiển thị trạng thái giả): không dùng làm cổng approval.
- Chạy TUI trong pane herdr sẽ bỏ qua hook/`canUseTool` có cấu trúc nếu không thiết kế thêm: lý do giữ tùy chọn sau PoC.
- Dependency Rust/native thêm bề mặt chuỗi cung ứng nếu về sau chọn dùng.

## 9. Cost and operational implications

- Không chi phí ở MVP.
- PoC HerdrRuntime tốn vài ngày nếu thực hiện sau.
- Theo dõi phiên bản 0.x nếu tích hợp.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| Thấy herdr 0.9.5 và socket trên máy (nhãn ĐX) | `docs/research/HERDR_ANALYSIS.md §8.2` | Chưa kiểm chứng |
| License Apache-2.0, repo không có NOTICE ở gốc | `docs/research/HERDR_ANALYSIS.md §9` | Đã xác minh |
| Bảng quyết định tái sử dụng là đề xuất | `docs/research/HERDR_ANALYSIS.md §8.1` | Giả định |
| Phương án loại: dùng herdr làm đường chính | `DECISION_LOG.md D-020` | Đã xác minh |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Partial. License và sự hiện diện trên máy được đọc trực tiếp; QA không đối chiếu, và HerdrRuntime chưa có PoC.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Chấp nhận herdr chỉ là tham chiếu, HerdrRuntime sau PoC.
- Xác nhận không dùng mã Rust của herdr.
- **Khả năng đảo ngược:** Rất dễ: không có phụ thuộc mã nào.
- **Bị chặn trong lúc chờ duyệt:** Mọi mã HerdrRuntime/terminal tương tác (không chặn MVP).

## 12. Reversal conditions

- PoC cho thấy giá trị rõ cho người dùng terminal: nâng thành tính năng chính thức.
- herdr đổi license hoặc ngừng phát triển: bỏ HerdrRuntime.
- Chủ sở hữu muốn terminal tương tác ngay ở MVP: lùi lịch và đánh giá lại.
