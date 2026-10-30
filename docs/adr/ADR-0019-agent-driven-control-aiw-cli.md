# ADR-0019 — Điều khiển bởi agent: biến môi trường AIW_*, CLI aiw trả JSON, token API riêng theo agent

## 1. Metadata

- **ID:** ADR-0019
- **Ngày:** 2026-10-30
- **Decision ID:** D-019 (trong `DECISION_LOG.md`)
- **Decision status:** APPROVED_CONDITIONAL
- **Owner approval required:** yes
- **Approved by/date:** Owner, 2026-10-30 (OWR-2)
- **Nguồn:** design/HERDR_DESIGN_INTEGRATION.md §6, research/HERDR_ANALYSIS.md §7

## 2. Context

- Agent cần tự báo trạng thái, chờ event và hỏi thông tin hệ thống mà không phụ thuộc prompt thuần.
- Báo cáo từ nhiều nguồn có thể đến lệch thứ tự và làm UI hiển thị trạng thái cũ.

## 3. Problem

- Thiết kế kênh điều khiển an toàn, chống lồng nhau, có thứ tự.
- Endpoint hook dùng listener riêng với `AIW_HOOK_TOKEN` (BACKEND_ARCHITECTURE).

## 4. Options considered

### Phương án A — Biến môi trường `AIW_*` + CLI `aiw` trả JSON + skill cho agent + báo cáo có `source` + `seq` đơn điệu + token API riêng theo agent [ĐỀ XUẤT]

Phase 3; JSON Schema giao thức xuất từ backend; ràng buộc lệnh resume (tên trần, giới hạn đối số); `aiw wait` đến khi blocked.
- **Lý do chọn đề xuất:** Độc lập provider, có thứ tự, có cấu trúc.

### Phương án B — Agent chỉ giao tiếp qua prompt [Bị loại]

Không cần hạ tầng.
- **Lý do loại:** Không cấu trúc, không đáng tin, không `seq` chống trạng thái cũ.

### Phương án C — MCP server cho điều khiển [Bị loại]

Chuẩn hóa.
- **Lý do loại:** Phụ thuộc CLI hỗ trợ MCP; `aiw` độc lập provider hơn.

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- Làm ở Phase 3, không Phase 1–2.
- Token API per-agent: qua redaction, hết hạn theo task, scope tối thiểu; endpoint hook ở listener riêng.
- Bộ skill dạy agent dùng `aiw` theo chuẩn Agent Skills (ADR-0013).

## 6. Rationale

- Báo cáo có cấu trúc đáng tin hơn suy luận từ output.
- `seq` + `source` giải quyết thứ tự từ nhiều nguồn.
- Giữ tương thích chuẩn Agent Skills.

## 7. Consequences

**Tích cực**
- Trạng thái chính xác hơn.
- Điều khiển độc lập provider.

**Tiêu cực**
- Thêm bề mặt tấn công: token agent nằm trong env.
- Phải duy trì CLI và schema ổn định.

**Trung tính**
- Chỉ dùng được từ Phase 3.
- Nested execution phải chặn bằng biến môi trường.

## 8. Security implications

- Token agent bị đánh cắp cho phép giả mạo trạng thái: scope hẹp, TTL theo task, thu hồi khi task kết thúc, không log.
- Env có thể lộ qua `docker inspect` nếu không lọc (CREDENTIAL_MANAGEMENT §15).
- Đầu vào lệnh `aiw` từ agent là input không tin cậy: validate schema, giới hạn tốc độ, chống nested/loop.

## 9. Cost and operational implications

- Xây và duy trì CLI + skill + schema ở Phase 3.
- Token per-agent tăng chi phí cấp phát/thu hồi.
- Không chi phí hạ tầng ngoài.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| Đề xuất Phase 3 | `docs/design/HERDR_DESIGN_INTEGRATION.md §6` | Giả định |
| Mô hình điều khiển bởi agent của herdr đã đọc (nhãn ĐX) | `docs/research/HERDR_ANALYSIS.md §7` | Chưa kiểm chứng |
| Hook endpoint listener riêng với `AIW_HOOK_TOKEN` | `docs/BACKEND_ARCHITECTURE.md (approvals)` | Giả định |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Unverified. Thiết kế đề xuất cho Phase 3, chưa prototype; nguồn herdr không được QA đối chiếu.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Chấp nhận cơ chế `aiw` CLI + token per-agent ở Phase 3.
- Chấp nhận rủi ro token trong env và các biện pháp giảm.
- **Khả năng đảo ngược:** Dễ: Phase 3, chưa có mã; có thể thay bằng MCP.
- **Bị chặn trong lúc chờ duyệt:** Không chặn Phase 1–2; chặn mọi mã điều khiển bởi agent.

## 12. Reversal conditions

- Đánh giá bảo mật cho thấy token agent không bảo vệ đủ: kênh một chiều qua stdout.
- Provider chuẩn hóa MCP đủ tốt: dùng MCP thay `aiw`.
- Phase 3 bị hoãn: lùi thời điểm áp dụng.

## 13. Cập nhật OWR-2 (2026-10-30)

Nguồn: OWR-2 §3 D-019 trong `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md`. Các mục 5 đến 12 ở trên là đề xuất gốc; **nếu mâu thuẫn với cập nhật này thì cập nhật này là trạng thái hiện hành**, lịch sử không bị xóa. Việc đồng bộ tài liệu khác do agent khác thực hiện, chưa được xác nhận ở đây.

- Agent CLI/API **chỉ triển khai trong Phase 3, sau Security Review** (chưa thực hiện): điều kiện chưa đạt. Short-lived token với least privilege; có redaction; ngăn nested execution không được phép.
- **Validation evidence (trung thực):** mức giữ nguyên **Unverified** (xem `README.md`). Chưa có PoC report hay bằng chứng kiểm thử mới; **chưa có mục nào VERIFIED**. Phase 0 vẫn mở, Phase 1 chưa được cho phép.
