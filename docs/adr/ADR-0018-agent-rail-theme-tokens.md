# ADR-0018 — Agent Rail (sidebar hai vùng) và cấu trúc token theme kiểu herdr

## 1. Metadata

- **ID:** ADR-0018
- **Ngày:** 2026-10-30
- **Decision ID:** D-018 (trong `DECISION_LOG.md`)
- **Decision status:** APPROVED
- **Owner approval required:** yes
- **Approved by/date:** Owner, 2026-10-30 (OWR-2)
- **Nguồn:** design/HERDR_DESIGN_INTEGRATION.md §3, §4, research/HERDR_ANALYSIS.md §3

## 2. Context

- Sidebar một vùng không cho thấy đồng thời project và agent; theme cần preset, tùy biến và tự chuyển sáng/tối.
- Cần giữ accent AI tím riêng (ADR-0015); NAVIGATION_STRUCTURE đã duyệt các nhóm sidebar chính.

## 3. Problem

- Thêm ngữ cảnh agent ở mọi màn hình mà không làm rối điều hướng.
- Đồng bộ palette 20 token kiểu herdr với DESIGN_TOKENS (8 status).

## 4. Options considered

### Phương án A — Panel hai vùng (projects trên, agents dưới, kéo tỉ lệ, dòng cấu hình bằng token) + cấu trúc token (preset + `theme.custom` + auto_switch), giữ accent tím [ĐỀ XUẤT]

Palette 20 token; hàm màu chữ tương phản theo WCAG; phân tầng bằng độ sáng nền, không viền/bóng.
- **Lý do chọn đề xuất:** Người điều hành thấy project + agent cùng lúc.

### Phương án B — Sidebar một vùng [Bị loại]

Đơn giản.
- **Lý do loại:** Không thấy agent khi đang xem project.

### Phương án C — Hai theme sáng/tối cố định, không `theme.custom` [Bị loại]

Ít phức tạp.
- **Lý do loại:** Mất tùy biến và preset.

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- Agent Rail bổ sung vào NAVIGATION_STRUCTURE (sidebar chính giữ nguyên các nhóm đã duyệt).
- Chỉ lấy cấu trúc token và ý tưởng từ herdr, không mã Rust (ADR-0020).
- Âm thanh thông báo chỉ dùng sau khi kiểm license asset.

## 6. Rationale

- Người điều hành cần ngữ cảnh agent ở mọi màn hình.
- Cấu trúc token tương thích `[data-theme]` của DESIGN_TOKENS.
- Giữ `--ai-accent` bảo toàn ADR-0015.

## 7. Consequences

**Tích cực**
- Thấy project và agent đồng thời.
- Theme tùy biến được.

**Tiêu cực**
- Responsive mobile/tablet phức tạp hơn; Rail chiếm không gian màn hình nhỏ.
- Phải sửa nhiều tài liệu design (HERDR_DESIGN_INTEGRATION §9).

**Trung tính**
- Bố cục dựa trên screenshot và mã Rust, chưa có thử nghiệm người dùng.
- Âm thanh cần kiểm license.

## 8. Security implications

- `theme.custom` do người dùng nhập là input không tin cậy: chỉ cho giá trị màu hợp lệ, không CSS tùy ý (chống CSS injection/exfil).
- Rail hiển thị trạng thái agent nhạy cảm: tôn trọng quyền theo project.
- Không có bề mặt mạng mới.

## 9. Cost and operational implications

- Chi phí UI cao hơn sidebar thường (kéo tỉ lệ, responsive).
- Cập nhật nhiều tài liệu design.
- Không có chi phí hạ tầng.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| Đề xuất Agent Rail và token | `docs/design/HERDR_DESIGN_INTEGRATION.md §3, §4` | Giả định |
| Palette và bố cục đọc từ `src/app/state.rs` và screenshot (nhãn ĐX) | `docs/research/HERDR_ANALYSIS.md §3` | Chưa kiểm chứng |
| Quyết định T1–T6 của DESIGN_TOKENS | `docs/design/DESIGN_TOKENS.md §4` | Giả định |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Unverified. Thiết kế UI chưa thử nghiệm người dùng; nguồn là đọc mã/screenshot của herdr, QA không đối chiếu.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Chấp nhận Agent Rail và cấu trúc token theme.
- Chấp nhận phạm vi chỉ lấy ý tưởng (không mã herdr).
- **Khả năng đảo ngược:** Dễ: Rail có thể thành panel tùy chọn.
- **Bị chặn trong lúc chờ duyệt:** Layout shell, theme system.

## 12. Reversal conditions

- Thử nghiệm người dùng cho thấy Rail gây nhiễu: panel tùy chọn.
- Màn hình mục tiêu chủ yếu nhỏ: bỏ vùng Agent.
- herdr đổi license hoặc bị loại khỏi tham chiếu.

## 13. Cập nhật OWR-2 (2026-10-30)

Nguồn: OWR-2 §3 D-018 trong `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md`. Các mục 5 đến 12 ở trên là đề xuất gốc; **nếu mâu thuẫn với cập nhật này thì cập nhật này là trạng thái hiện hành**, lịch sử không bị xóa. Việc đồng bộ tài liệu khác do agent khác thực hiện, chưa được xác nhận ở đây.

- Agent Rail chia 2 section; cho phép ẩn/hiện; mặc định hiển thị trên màn hình lớn.
- **Validation evidence (trung thực):** mức giữ nguyên **Unverified** (xem `README.md`). Chưa có PoC report hay bằng chứng kiểm thử mới; **chưa có mục nào VERIFIED**. Phase 0 vẫn mở, Phase 1 chưa được cho phép.
