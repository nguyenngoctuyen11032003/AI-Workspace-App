# ADR-0017 — Mô hình 5 attention state làm lớp hiển thị trên 12 lifecycle state

## 1. Metadata

- **ID:** ADR-0017
- **Ngày:** 2026-10-30
- **Decision ID:** D-017 (trong `DECISION_LOG.md`)
- **Decision status:** PENDING_OWNER_APPROVAL
- **Owner approval required:** yes
- **Approved by/date:** (để trống, chủ sở hữu chưa duyệt)
- **Nguồn:** design/HERDR_DESIGN_INTEGRATION.md §2, research/HERDR_ANALYSIS.md §4

## 2. Context

- Người điều hành 18 agent phải thấy ngay cái gì cần mình; 12 lifecycle state của DB quá chi tiết để hiển thị trực tiếp.
- Nhiều tài liệu dùng bộ trạng thái khác: 5 giá trị trong event (A07), 8 tone StatusBadge (M-01).

## 3. Problem

- Chọn lớp hiển thị trạng thái thống nhất mà không giả lập tiến độ.
- Case enum `task_status` HOA/thường (CF-011) ảnh hưởng ánh xạ.

## 4. Options considered

### Phương án A — 5 attention state (blocked/working/done/idle/unknown) tính từ dữ liệu thật; `done` chưa xem theo từng client; trạng thái project/department tổng hợp [ĐỀ XUẤT]

Ánh xạ lifecycle → attention là hàm thuần có test trong `packages/contracts`; ưu tiên `blocked > working > done > idle`.
- **Lý do chọn đề xuất:** Dễ đọc cho 18 agent, giữ nguyên lifecycle trong DB.

### Phương án B — Chỉ hiển thị 12 lifecycle state [Bị loại]

Giữ nguyên.
- **Lý do loại:** Quá nhiều trạng thái, khó thấy việc cần chú ý.

### Phương án C — Dùng 5 trạng thái event (`idle/working/waiting/error/offline`) [Bị loại]

Đã có trong event.
- **Lý do loại:** Mất độ phân giải nghiệp vụ; A07 hết hiệu lực ở điểm này (M-01).

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- `working` chỉ khi có event thật trong cửa sổ TTL; `unknown` khi mất heartbeat.
- Bảng ánh xạ 12 → 5 → 8 tone nằm trong `packages/contracts` (CF-009).
- Thêm bảng theo dõi «đã xem» (`notification_reads`/`last_seen_event_seq`) theo user + client.

## 6. Rationale

- Nguyên tắc «chú ý trước, kỹ thuật sau» hợp quy mô 18 agent.
- Lifecycle vẫn là sự thật nghiệp vụ.
- Khớp luật «không giả lập tiến độ» của Virtual Office.

## 7. Consequences

**Tích cực**
- Người dùng thấy việc cần chú ý nhanh.
- Một nguồn sự thật cho ánh xạ.

**Tiêu cực**
- Cần bảng mới theo dõi «đã xem».
- Thêm một lớp ánh xạ phải test; nguy cơ lệch SCREEN_SPEC §2.

**Trung tính**
- Heuristic nhận biết trạng thái từ màn hình bị loại.
- Phụ thuộc chất lượng heartbeat/event.

## 8. Security implications

- Hiển thị sai `working`/`done` có thể che giấu agent bị chặn hoặc lỗi: `unknown` phải hiện rõ khi mất tín hiệu.
- `blocked` (chờ approval) phải luôn nổi bật để không bỏ sót yêu cầu duyệt rủi ro.
- Dữ liệu «đã xem» theo client là dữ liệu người dùng: không lộ cho agent.

## 9. Cost and operational implications

- Chi phí thêm một bảng và job cập nhật; nhỏ.
- Chi phí test hàm ánh xạ.
- Không có chi phí hạ tầng.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| Mô hình attention là đề xuất, chưa triển khai | `docs/design/HERDR_DESIGN_INTEGRATION.md §2` | Giả định |
| Mô hình trạng thái herdr đọc từ tài liệu và mã (nhãn ĐX) | `docs/research/HERDR_ANALYSIS.md §4` | Chưa kiểm chứng |
| A07 5 giá trị vs DB 12 | `docs/AGENT_ARCHITECTURE.md §8 M-01` | Đã xác minh |
| Cần bảng ánh xạ chính thức trong `packages/contracts` | `docs/IMPLEMENTATION_ROADMAP.md §3.14 #2` | Đã xác minh |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Unverified. Là thiết kế UI/domain chưa triển khai hay thử với người dùng; chỉ xác minh được mâu thuẫn trạng thái giữa tài liệu.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Chấp nhận 5 attention state và bảng ánh xạ chính thức.
- Chấp nhận thêm dữ liệu «đã xem» theo client.
- **Khả năng đảo ngược:** Dễ: là lớp hiển thị, lifecycle trong DB không đổi.
- **Bị chặn trong lúc chờ duyệt:** StatusBadge, Agent Rail, notification logic.

## 12. Reversal conditions

- Người dùng thử nghiệm không phân biệt được `done` chưa xem: đổi trạng thái.
- Số lifecycle state đổi: cập nhật ánh xạ và test.
- Chi phí `notification_reads` vượt lợi ích: chỉ giữ cờ cục bộ.
