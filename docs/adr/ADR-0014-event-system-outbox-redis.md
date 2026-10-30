# ADR-0014 — Event system: outbox giao dịch → Redis pub/sub (Socket.IO) + Redis Stream cho Virtual Office

## 1. Metadata

- **ID:** ADR-0014
- **Ngày:** 2026-10-30
- **Decision ID:** D-014 (trong `DECISION_LOG.md`)
- **Decision status:** APPROVED
- **Owner approval required:** yes
- **Approved by/date:** Owner, 2026-10-30 (OWR-2)
- **Nguồn:** research/ORCHESTRATION_WORKFLOW_RESEARCH.md, EVENT_SYSTEM.md, research/TECHNOLOGY_EVALUATION.md TE-08, QA_REVIEW_PHASE0.md (P1-13)

## 2. Context

- UI, Virtual Office và audit cần luồng event đáng tin: mỗi transition ghi đúng một event cùng transaction; client mất kết nối phải replay theo `lastEventId`.
- Event naming lệch: EVENT_SYSTEM dùng UPPER_SNAKE; DATABASE_DESIGN:1196, API_DOCUMENTATION:343, AUDIT_LOGGING:24 dùng dotted lowercase.
- D-014 trích «Prompt 1 mục 8.4» nhưng API_DOCUMENTATION:433 xác nhận đặc tả đó không có trong `docs/` (P1-13).

## 3. Problem

- Chốt quy ước tên event và nguồn đặc tả; giải quyết lệch về outbox, kênh Redis, retention và ID event (ULID vs uuid, M-E1).
- Bảng `event_outbox` chưa có trong 44 bảng (API_DOCUMENTATION §13 chặn freeze).

## 4. Options considered

### Phương án A — Outbox giao dịch → Redis pub/sub (Socket.IO) + Redis Stream cho Office; replay bằng `lastEventId` [ĐỀ XUẤT]

Zod schema ở `@aiw/contracts`; consumer idempotent theo `event.id`; `global_seq` giữ thứ tự; `EventTransport` trừu tượng.
- **Lý do chọn đề xuất:** Không mất event khi crash, có replay, có kênh điều khiển ngược.

### Phương án B — Chỉ WebSocket thô [Bị loại]

Đơn giản.
- **Lý do loại:** Mất event khi restart, không replay, không đảm bảo thứ tự.

### Phương án C — SSE một chiều [Bị loại]

Đơn giản, có Last-Event-ID.
- **Lý do loại:** Không có kênh điều khiển ngược và rooms.

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- **Quy ước tên (đề xuất):** UPPER_SNAKE là tên chuẩn của contract (API, WebSocket, Zod, UI). Cột `type` trong DB và audit log lưu dotted lowercase; có bảng ánh xạ 1-1 duy nhất trong `packages/contracts` (ví dụ `TASK_COMPLETED` ↔ `task.completed`), có test chạy hai chiều, và không nơi nào tự dịch bằng regex.
- Chủ sở hữu cung cấp đặc tả event «Prompt 1 mục 8.4» (hoặc xác nhận danh sách trong EVENT_SYSTEM là chuẩn) và chép vào `docs/`.
- Relay đọc event append-only theo `global_seq` (CF-004, CF-035); event heartbeat là transient; thêm bảng `event_outbox`; thống nhất ID event và retention 7 ngày vs 90/365 ngày.

## 6. Rationale

- Một transaction ghi state + event loại bỏ event mồ côi.
- Redis Stream đệm cho Office replay và catch-up.
- Zod schema một nguồn tránh lệch tên giữa tài liệu.

## 7. Consequences

**Tích cực**
- Replay và thứ tự đáng tin.
- Contract một nguồn.

**Tiêu cực**
- Relay một leader để giữ `global_seq`; có thể trùng event nên consumer phải idempotent.
- Socket.IO không nói chuyện với native WebSocket; hai repo tham chiếu dùng `ws`.
- Hai dạng tên event cần bảng ánh xạ và test.

**Trung tính**
- Socket.IO + Redis adapter chưa kiểm tra.
- Retention cần chốt.

## 8. Security implications

- Event có thể chứa dữ liệu nhạy cảm: redaction trước khi ghi outbox và trước khi phát WS.
- Kênh WS cần xác thực và phân quyền theo room; heartbeat/ack không được giả mạo.
- Audit log append-only (trigger) phải dùng tên dotted nhất quán để truy vấn không bỏ sót.

## 9. Cost and operational implications

- Relay, retention và Redis Stream tốn RAM Redis; chưa đo.
- Bảng event phân vùng theo tháng cần job bảo trì (ADR-0002).
- Chi phí bảo trì ánh xạ tên event.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| Bảng mâu thuẫn M-E1..M-E8 tồn tại | `docs/EVENT_SYSTEM.md §14` | Đã xác minh |
| Lệch UPPER_SNAKE vs dotted và thiếu đặc tả Prompt 1 mục 8.4 | `docs/QA_REVIEW_PHASE0.md P1-13` | Đã xác minh |
| Outbox + Redis + Socket.IO tốt nhất; chưa prototype hay đo tải | `docs/research/TECHNOLOGY_EVALUATION.md TE-08` | Giả định |
| Hạng mục chặn freeze contract | `docs/API_DOCUMENTATION.md §6.3, §13` | Đã xác minh |
| Replay/outbox chịu được crash và nhiều instance | `docs/EVENT_SYSTEM.md` | Chưa kiểm chứng |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Unverified. Chỉ xác minh được các mâu thuẫn tài liệu; thiết kế outbox/relay chưa prototype hay đo tải.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Chọn quy ước tên: UPPER_SNAKE (contract) + dotted lowercase (DB/audit) với bảng ánh xạ, hoặc chọn một dạng duy nhất.
- Cung cấp hoặc xác nhận đặc tả event gốc (Prompt 1 mục 8.4).
- **Khả năng đảo ngược:** Dễ trước freeze contract; rất khó sau khi có dữ liệu audit và client phụ thuộc tên.
- **Bị chặn trong lúc chờ duyệt:** Freeze `@aiw/contracts`, API_DOCUMENTATION, schema `agent_events`/`event_outbox`.

## 12. Reversal conditions

- Đo thực cho thấy relay là nút cổ chai: logical replication/Debezium hoặc `LISTEN/NOTIFY` + cursor.
- Nhiều instance khiến Socket.IO + sticky quá phức tạp: chuyển `ws` thuần.
- Chủ sở hữu chấp nhận mất event khi crash cho một số loại: nới outbox.

## 13. Cập nhật OWR-2 (2026-10-30)

Nguồn: OWR-2 §3 D-014, §1 Q-19, §2 Q-12 trong `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md`. Các mục 5 đến 12 ở trên là đề xuất gốc; **nếu mâu thuẫn với cập nhật này thì cập nhật này là trạng thái hiện hành**, lịch sử không bị xóa. Việc đồng bộ tài liệu khác do agent khác thực hiện, chưa được xác nhận ở đây.

- Dùng Transactional Outbox; một relay leader chịu trách nhiệm phát event; Redis Pub/Sub hoặc Streams cho realtime delivery; hỗ trợ replay bằng `lastEventId`.
- Naming theo Q-19: database dùng `lowercase.dot.notation` (ví dụ `task.started`, `task.completed`, `agent.blocked`); API hoặc lớp tương thích có thể ánh xạ sang `UPPER_SNAKE_CASE`. Không tạo hai hệ thống event naming độc lập.
- **PENDING_OWNER_CLARIFICATION:** nội dung "Prompt 1 mục 8.4" (danh sách event gốc) không được lưu trong repo. Owner yêu cầu không tự tạo danh sách. Ghi nhận: mười sáu tên event có trong văn bản prompt đầu tiên của Owner; Owner cần xác nhận một bản sao lưu trong repo. ADR này không chép danh sách đó và không coi danh sách event hiện tại là nội dung gốc đã phê duyệt.
- **Validation evidence (trung thực):** mức giữ nguyên **Unverified** (xem `README.md`). Chưa có PoC report hay bằng chứng kiểm thử mới; **chưa có mục nào VERIFIED**. Phase 0 vẫn mở, Phase 1 chưa được cho phép.

## 14. Cập nhật OWR-3 (2026-10-30)

Nguồn: `docs/owner-responses/OWR-3_2026-10-30_decision_clarification.md` (OWR-3) (A.1, A.16). Các mục 1 đến 13 ở trên giữ làm lịch sử; **nếu mâu thuẫn với cập nhật này thì cập nhật này là trạng thái hiện hành**. Việc đồng bộ tài liệu khác do agent khác thực hiện, xem `docs/DOC_SYNC_LOG_OWR3_A.md` và `docs/DOC_SYNC_LOG_OWR3_B.md`; ADR này không khẳng định đã đồng bộ.

- Decision status giữ **APPROVED** (kiến trúc).
- A.1: Owner xác nhận 16 event trong Prompt 1 mục 8.4 là **danh sách gốc**, được chép nguyên văn vào repo (xem Phụ lục OWR-3). Không tự thêm, xóa hoặc đổi tên event gốc. Đóng một phần P-1.
- A.16: chỉ dùng **dotted lowercase** làm canonical event naming; không dùng underscore naming làm chuẩn.
- **Còn mở, PENDING_OWNER_CLARIFICATION (P-16, X-14):** "bảng ánh xạ Owner cung cấp" **không tồn tại** trong OWR-3. Ví dụ `task.started` và `agent.blocked` (OWR-2) không nằm trong 16 event gốc; `TASK_COMPLETED` có trong danh sách. ADR này không tự soạn bảng ánh xạ.
- **Validation evidence (trung thực):** mức giữ nguyên (xem `README.md`). Chưa có PoC report hay bằng chứng kiểm thử mới; **chưa có mục nào VERIFIED**. Phase 0 vẫn mở, Phase 1 chưa được cho phép.
