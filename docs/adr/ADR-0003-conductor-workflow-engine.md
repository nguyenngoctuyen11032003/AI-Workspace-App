# ADR-0003 — Workflow engine tự xây (Conductor) trên BullMQ + Postgres, approval là WAITING_HUMAN

## 1. Metadata

- **ID:** ADR-0003
- **Ngày:** 2026-10-30
- **Decision ID:** D-003 (trong `DECISION_LOG.md`)
- **Decision status:** PENDING_OWNER_APPROVAL
- **Owner approval required:** yes
- **Approved by/date:** (để trống, chủ sở hữu chưa duyệt)
- **Nguồn:** research/ORCHESTRATION_WORKFLOW_RESEARCH.md, research/TECHNOLOGY_EVALUATION.md TE-02, WORKFLOW_ENGINE.md

## 2. Context

- Engine phải chạy DAG do người dùng chỉnh bằng React Flow, durable trong Postgres, resume sau restart, approval chờ nhiều ngày, fan-out có giới hạn, retry, cancel/pause, versioned definition.
- Subprocess CLI chạy 10–40 phút nên node không thể replay từ đầu.

## 3. Problem

- Chọn engine đáp ứng đồng thời R1–R10 của ORCHESTRATION_WORKFLOW_RESEARCH §1 mà không thêm hạ tầng nặng.
- Enum trạng thái node cần `node_run_status` mới (CF-010); WORKFLOW_ENGINE §15 ghi M-W1..M-W10.

## 4. Options considered

### Phương án A — Conductor: DAG executor trên BullMQ + trạng thái run trong Postgres [ĐỀ XUẤT]

JSON là định nghĩa (khớp React Flow); approval là `WAITING_HUMAN` không giữ worker; semaphore Redis Lua global + per-provider; XState v5 chỉ để validate transition.
- **Lý do chọn đề xuất:** Chỉ phương án này thỏa đồng thời các yêu cầu mà dùng lại PG + Redis sẵn có.

### Phương án B — LangGraph JS + PostgresSaver [Bị loại]

Có interrupt/resume.
- **Lý do loại:** `interrupt()` chạy lại node từ đầu, `Send` không cap xuyên process, graph là code, checkpoint dạng blob.

### Phương án C — Temporal TS [Bị loại]

Độ bền cao nhất.
- **Lý do loại:** Server 4 service + DB, workflow là code không hợp JSON user-editable, quá nặng cho ~10–18 agent.

### Phương án D — BullMQ Flows thuần [Bị loại]

Hàng đợi có cha-con.
- **Lý do loại:** Là cây, không có diamond, condition edge hay wait-human.

## 5. Decision status and what is being proposed

Trạng thái: **PENDING_OWNER_APPROVAL**. Nội dung dưới đây là đề xuất của System Architect, chưa phải quyết định có hiệu lực.

- Tự xây Conductor; LangGraph chỉ được xem là tùy chọn bên trong một node.
- Approval là trạng thái `WAITING_HUMAN`, không giữ worker; semaphore Redis Lua (ADR-0010).
- Cần spike «scheduler + tool node giả» và test chaos trước khi cam kết.

## 6. Rationale

- Chỉ A thỏa R1–R10 mà không thêm hạ tầng.
- Trạng thái nằm trong Postgres giúp báo cáo, audit, replay.
- Approval không giữ worker khớp mô hình quota subscription (slot quý).

## 7. Consequences

**Tích cực**
- Định nghĩa JSON sửa được bằng UI, không thêm service ngoài PG + Redis.

**Tiêu cực**
- Phải tự xây scheduler, recovery, chaos test; phần lớn công sức Phase 1.
- Rủi ro idempotency khi hai scheduler tick cùng run (cần `SELECT FOR UPDATE` + `jobId` dedupe).

**Trung tính**
- Cần bảng ánh xạ 11 kind node và enum mới; Windows không có SIGTERM tin cậy nên dùng `taskkill /T /F`.

## 8. Security implications

- Nút `tool`/`shell` thực thi lệnh phải đi qua policy engine và sandbox (ADR-0007); engine không được bỏ qua gate approval.
- Định nghĩa workflow do người dùng sửa là input không tin cậy: validate schema trước khi chạy.
- Trạng thái approval phải ghi audit append-only; `WAITING_HUMAN` không được tự chuyển trạng thái.

## 9. Cost and operational implications

- Chi phí xây cao (ước tính chưa có prototype, có thể bị đánh giá thấp).
- Vận hành thấp: không thêm service; cần lease TTL và reaper cho semaphore khi worker chết.
- Test chaos là chi phí thường trực.

## 10. Validation evidence

| Claim | Evidence path | Status |
|---|---|---|
| Ma trận R1–R10 kèm trích dẫn docs LangGraph/BullMQ/Temporal | `docs/research/ORCHESTRATION_WORKFLOW_RESEARCH.md §1` | Giả định |
| Đối chiếu mã ChatDev (`workflow/graph.py`, `topology_builder.py`) | `docs/research/TECHNOLOGY_EVALUATION.md TE-02` | Đã xác minh |
| Chưa có prototype; license Inngest self-host còn mâu thuẫn | `docs/research/TECHNOLOGY_EVALUATION.md TE-02` | Chưa kiểm chứng |
| Mâu thuẫn M-W1..M-W10 tồn tại | `docs/WORKFLOW_ENGINE.md §15` | Đã xác minh |
| Scheduler tự xây chịu được chaos/restart | `docs/WORKFLOW_ENGINE.md` | Chưa kiểm chứng |

Ghi chú: QA Phase 0 (`docs/QA_REVIEW_PHASE0.md`) không chạy lại bất kỳ live test nào (không chạy prompt Claude/Codex, không kiểm daemon Docker), chỉ spot-check khoảng 17 nhóm tuyên bố. Mọi số đo do tài liệu nguồn tự báo cáo vẫn chưa được tái hiện độc lập. Không claim nào ở đây được nâng mức so với tài liệu nguồn.

**Mức bằng chứng tổng hợp:** Unverified. Quyết định dựa trên so sánh lý thuyết từ tài liệu, chưa prototype scheduler.

## 11. Owner approval requirement

Chủ sở hữu cần xác nhận chính xác:
- Chấp nhận tự xây engine (chi phí cao) thay vì Temporal/LangGraph.
- Chấp nhận model `WAITING_HUMAN` và enum `node_run_status` mới.
- **Khả năng đảo ngược:** Trung bình ở Phase 1; đắt sau khi workflow definition và UI phụ thuộc định dạng JSON.
- **Bị chặn trong lúc chờ duyệt:** Thiết kế chi tiết scheduler/executor và bất kỳ mã workflow nào.

## 12. Reversal conditions

- Spike scheduler vượt ngân sách hoặc không qua chaos test: đánh giá Temporal hoặc LangGraph bọc ngoài.
- Yêu cầu đổi sang workflow do code định nghĩa.
- Quy mô vượt xa ~18 agent khiến scheduler tự xây không chịu tải.
