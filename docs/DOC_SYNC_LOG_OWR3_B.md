# DOC_SYNC_LOG_OWR3_B — Đồng bộ tài liệu theo OWR-3 (agent B: database, events, workflow, design, office, kiến trúc, product)

Ngày: 2026-10-30. Nguồn: `docs/owner-responses/OWR-3_2026-10-30_decision_clarification.md` (đọc cả phụ lục 16 event), rồi OWR-2 và `DOC_SYNC_LOG_OWR2_B.md`. Phase 0 vẫn MỞ; Phase 1 CHƯA được phép; không mục nào được đánh dấu VERIFIED; không chạy git add/commit/push. Đường dẫn tính từ `AI-Workspace-App/`. Số dòng gần đúng. "Xác minh" = grep sau khi sửa.

## 1. Đã áp dụng

| Change ID | file:dòng | Cũ | Mới | OWR-3 | Xác minh |
|---|---|---|---|---|---|
| C-01 | docs/EVENT_SYSTEM.md:9 | ví dụ `task.started`, `agent.blocked` | ví dụ `task.completed`, `agent.tool.started`; ghi rõ `task.started`/`agent.blocked` không thuộc 16 event gốc; quy tắc cơ học `_`→`.` | A.1, A.16, X-4 | grep `không thuộc 16 event gốc` |
| C-02 | EVENT_SYSTEM.md:11-13 | PENDING: 16 event gốc "không được lưu trong repo" | Đã giải quyết danh sách gốc; thay bằng PENDING_OWNER_CLARIFICATION (P-16): Owner chưa cung cấp bảng ánh xạ | A.1 | grep `P-16`; cụm cũ = 0 |
| C-03 | EVENT_SYSTEM.md §0 (mới, dòng ~15-45) | không có | Mục "Danh sách event gốc của Owner": 16 tên nguyên văn + cột dotted (đề xuất cơ học, P-16) + ghi chú xung đột 10/16 chưa có trong danh mục | A.1, A.16 | grep `Danh sách event gốc của Owner`; đếm 16 hàng |
| C-04 | EVENT_SYSTEM.md §5 intro, §5.3 ghi chú, §5.6, §6.1 | "đề xuất, chưa phải danh sách gốc" | "đề xuất ngoài danh sách gốc, chờ Owner" (trừ 6 tên trùng gốc); `WORKFLOW_STAGE_CHANGED` nay là event gốc; xung đột `_` đánh dấu ĐÃ GIẢI QUYẾT | A.16, X-4 | grep `ngoài danh sách gốc, chờ Owner` |
| C-05 | EVENT_SYSTEM.md §5.7 (mới) | không có tên dotted cho catalogue | Bảng 77 tên UPPER_SNAKE → dotted lowercase không `_` (sinh bằng quy tắc cơ học), nhãn đề xuất ngoài danh sách gốc | A.16, X-4 | grep `5.7 Tên canonical` |
| C-06 | EVENT_SYSTEM.md ES-01, risk, §14 mục 2, M-E2 | "UPPER_SNAKE theo tiền tố là chuẩn" | Canonical dotted lowercase; P-16 | A.16 | grep `ES-01` |
| C-07 | DATABASE_DESIGN.md §5.2 (~1203-1215) | `task.status_changed`, `run.node_entered`, `agent.status_changed`, `llm.stream_chunk`... ; ghi chú "mục 8.4 chưa có" | `task.status.changed`, `run.node.entered`, `agent.status.changed`, `llm.stream.chunk`...; ghi chú trỏ EVENT_SYSTEM §0 và P-16; không xóa mục nào; ví dụ `'task.status.changed'` ở `agent_events.type` | A.1, A.16 | grep `_changed` chỉ còn cột `status_changed_at` |
| C-08 | DATABASE_DESIGN.md ~1007 | audit action `task.status_change` | `task.status.change` | A.16 | grep |
| C-09 | DATABASE_DESIGN.md ~836-870 (approval_policies, approvals), job queue ~1400 | một cột `expires_at` NOT NULL; `timeout_sec` | `remind_at` (24 giờ) và `expires_at` (15 phút, nullable, chỉ hành động quan trọng) tách; thêm `remind_after_sec`, `expires_after_sec`, index `approvals_remind_idx`, job `approval-remind` | A.12 | grep `remind_at` |
| C-10 | DATABASE_DESIGN.md ~1035 | `tile_size` default 32 | `tile_size` default 16 + `display_scale` default 2 (đề xuất, cờ) | A.7, X-6 | grep `display_scale` |
| C-11 | DATABASE_DESIGN.md seed `max_concurrent` | "không phải xác nhận subscription" | thêm: giới hạn scheduler nội bộ, không phải giới hạn provider; không chia sẻ credential/seat | A.2, D | grep `OWR-3 A.2/D` |
| C-12 | DATABASE_DESIGN.md `timeout_sec` | không rõ 540 s | ghi chú 540 s thuộc WORKFLOW_ENGINE §7, không phải `timeout_sec` | X-7 | grep `OWR-3 X-7` |
| C-13 | API_DOCUMENTATION.md ~326, 343, 347, 349, 410, 433 | tên có `_` (`status_changed`, `stage_changed`, `retry_scheduled`); PENDING mục 8.4 thiếu; "chọn hệ tên event" mở | dotted không `_`; `workflow.stage.changed` ghi là event gốc; 16 event đã xác nhận + P-16; mở khoản 1 gạch bỏ | A.1, A.16 | grep `OWR-3 A.16` |
| C-14 | API_DOCUMENTATION.md ~242 (`GET /approvals/:id`), ~202 (`/scheduler/limits`) | không có | trả riêng `remindAt` và `expiresAt` (đề xuất); concurrency 5 không phải giới hạn provider | A.12, A.2 | grep `remindAt` |
| C-15 | docs/security/AUDIT_LOGGING.md §3 (khối mới) + `agent.blocked` | action có `_` (`login_check`...) | khối quy ước tên + bảng 14 ánh xạ đề xuất; `agent.blocked` ghi không phải event gốc; retention 365 + export 2 năm trỏ §8 | A.16, X-4, X-8 | grep `OWR-3 A.16/X-4` |
| C-16 | WORKFLOW_ENGINE.md:10, §3 (dòng 56), WE-04 | "9 UI node types và 11 engine kinds đã duyệt" | 9 UI types chỉ UI-only; "11 engine kinds" giữ như vấn đề engine, OWR-3 không nêu lại/không đổi | A.14 | grep `OWR-3 A.14` |
| C-17 | WORKFLOW_ENGINE.md §7 bước 1, 5, 6; timeoutMs | một `expires_at` hai nghĩa; xung đột `reminder_at` | `remind_at`/`expires_at` tách, xung đột ghi ĐÃ GIẢI QUYẾT; job `approval-remind`; X-7 540 s | A.12, X-7 | grep `remind_at` |
| C-18 | WORKFLOW_ENGINE.md dòng 10, §7 bước 1 | nhả worker, không nói đếm | `WAITING_APPROVAL`/`WAITING_HUMAN` không tính vào 5 active workers khi worker đã nhả + state persist; quota provider vẫn áp dụng | A.15 | grep `OWR-3 A.15` |
| C-19 | MULTI_AGENT_ORCHESTRATION.md banner, §4.1, M-O2, bảng tầng provider | "5 (D-010)", "Codex 3 APPROVED" | 5 là giới hạn scheduler nội bộ; A.15; Codex hoãn; không chia sẻ seat | A.2, A.15, D, A.8 | grep `OWR-3` |
| C-20 | AGENT_ARCHITECTURE.md banner (dòng 9), §3.6, AD-06, §8, M-03, M-07, bước 6, risks | L0..L4, 5→4, Codex 3 | L0..L5 (Q-03 giữ, P-17 cho L5), Claude MVP duy nhất, A.15, trỏ X-13 | A.13, A.8, A.15, A.10 | grep `X-13` |
| C-21 | architecture/AGENT_ORGANIZATION.md §6, §4, decision row | L0..L4 xung đột | banner L0..L5; hàng L5 (PENDING P-17); "6 permission level" | A.13 | grep `L5` |
| C-22 | SYSTEM_ARCHITECTURE.md banner, mục 1, 8, 9 | UI mặc định tiếng Anh; concurrency 5 không chú thích | banner OWR-3 (Claude MVP duy nhất, X-13, event dotted); mặc định tiếng Việt; concurrency không phải giới hạn provider | X-1, A.8, A.10, A.2 | grep `OWR-3` |
| C-23 | BACKEND_ARCHITECTURE.md banner, §concurrency, `llm.stream_chunk` | không có | banner OWR-3 + X-13; `llm.stream.chunk` | A.8, A.10, A.2, A.16 | grep `X-13` |
| C-24 | FRONTEND_ARCHITECTURE.md §3.11, mục kiểm thử ~236, `llm.stream_chunk` | "bộ key EN đầy đủ, VI thiếu thì fallback EN" | bộ key VI mặc định đầy đủ; EN phụ; fallback ghi cờ phán đoán | X-1 | grep `OWR-3 X-1` |
| C-25 | VIRTUAL_OFFICE.md banner dòng 5, §5 dòng 62, §11.2, §14, §15, V2, V7, §21 mục 1, 3 | tile 16 vs 32 "chưa quyết"; DB tile_size 32 | art 16 px, phóng 2x = 32 px, benchmark sau; `tile_size`=16 + `display_scale`=2 (đề xuất); Q-15 + A.9 (MIT asset chỉ khi xác minh nguồn/license/NOTICE); event `_` → dotted; ghi chú tên UPPER_SNAKE là tương thích | A.7, X-6, A.9, A.16 | grep `OWR-3 A.7` |
| C-26 | design/SCREEN_SPECIFICATION.md:127, :142, :283, :699, các WS event | "32px tiles", tile chưa quyết, permission mode `full`, `agent.state_changed`... | "16px x2 = 32px"; X-6 giải quyết; không có `full`; tên event dotted không `_` | A.7, X-3, A.16 | grep `full` chỉ còn trong chú thích gạch bỏ |
| C-27 | design/NAVIGATION_STRUCTURE.md ~170-171; design/COMPONENT_SPECIFICATION.md ~484; design/UI_UX_GUIDELINES.md ~136 | `agent.state_changed`, `TASK_STATE_CHANGED`, "ngôn ngữ UI mặc định EN hay VI" | dotted; ghi chú tương thích; mở gạch bỏ (mặc định tiếng Việt) | A.16, X-1 | grep |
| C-28 | research/LICENSE_AND_REUSE.md (§3 mục 5, bảng ALLOW) và REUSE_DECISIONS.md đầu | "ALLOW có điều kiện asset" Pixel Agents furniture | thêm câu A.9; furniture/floor/wall/pet/carpet không dùng (nguồn gốc không rõ) | A.9, Q-15 | grep `OWR-3 A.9` |
| C-29 | research/TECHNOLOGY_EVALUATION.md TE-04 | Codex cần live test | ghi chú Codex hoãn, không cài, không live test | A.8, E | grep `OWR-3 A.8` |
| C-30 | product/USER_ROLES_AND_PERMISSIONS.md §4.2, §5.5, OI-ROLE-01 | "Không timeout mặc định; nhắc 24 giờ"; xung đột L0..L4 | `remind_at` 24 giờ tách `expires_at` 15 phút; L0..L5 chuẩn hóa, ngữ nghĩa/P-17 còn mở | A.12, A.13 | grep `remind_at` |
| C-31 | product/PRD.md (adapter, dependencies, concurrency), product/SRS.md (yêu cầu, dependencies, OI-SRS-01), product/USER_FLOWS.md đầu | Codex/OpenHands trong danh sách MVP; chưa trỏ X-13 | MVP chỉ Claude; Codex hoãn; trỏ X-13; A.2; A.15; tên event tương thích | A.8, A.10, A.2, A.15, A.16 | grep `OWR-3` |

## 2. Conflicts found (không ghi đè, cần Owner/Architect)

1. **10 trong 16 event gốc chưa có trong danh mục đề xuất** (AGENT_ASSIGNED, AGENT_STARTED, AGENT_THINKING, AGENT_TOOL_STARTED, AGENT_TOOL_COMPLETED, AGENT_WAITING_APPROVAL, AGENT_COMPLETED, AGENT_FAILED, WORKFLOW_STAGE_CHANGED, HUMAN_APPROVAL_REQUIRED). Danh mục đề xuất dùng tên gần nghĩa (`AGENT_STATE_CHANGED`, `HUMAN_APPROVAL_REQUESTED`, `WORKFLOW_STAGE_STARTED/COMPLETED`, `PROVIDER_TOOL_*`). Không đổi tên, không xóa; Owner cần chốt quan hệ (thay thế, bí danh, bổ sung). Ghi ở EVENT_SYSTEM §0.
2. **P-16:** Owner nói "theo bảng Owner cung cấp" nhưng không có bảng; ánh xạ ở §0 là quy tắc cơ học (đề xuất).
3. **Event/action có `_` trong tài liệu ngoài phạm vi sở hữu:** `AUDIT_LOGGING` (action `login_check`...), `PERMISSION_MODEL.md`, `CREDENTIAL_MANAGEMENT.md` (agent A) còn tham chiếu; tôi chưa đổi tại chỗ trong AUDIT_LOGGING để tránh lệch chéo file; chỉ thêm bảng ánh xạ đề xuất. Cần đồng bộ cùng lúc với agent A.
4. **UPPER_SNAKE trong thân tài liệu:** EVENT_SYSTEM §5-§9, VIRTUAL_OFFICE §8, USER_FLOWS, COMPONENT_SPECIFICATION vẫn viết tên UPPER_SNAKE; đã gắn ghi chú "tên tương thích" và bảng §5.7; không đổi từng dòng (khối lượng lớn). Cần quyết định có thay từng dòng không.
5. **Concurrency trần 18:** OWR-3 A.2 "không tăng lên 10 hoặc 18" vs tài liệu ghi "tối đa 18 (cấu hình)" (D-010/FR-232). Chưa sửa con số; ghi chú không phải giới hạn provider.
6. **MULTI_AGENT_ORCHESTRATION §4.2** có câu "tài khoản claude.ai `team` dùng chung → pool chung", mâu thuẫn "không chia sẻ credential/seat giữa nhiều người" (OWR-3 D). Đã gắn cờ, chưa sửa nội dung.
7. **`approval_policies.timeout_sec` default 3600** còn đó cạnh `remind_after_sec`/`expires_after_sec` (đề xuất); chưa quyết định giữ hay bỏ.
8. **L5 và `full`:** L5 = PM/Orchestrator (USER_ROLES) khác ngữ nghĩa AGENT_ORGANIZATION; P-17 mở.
9. **Kích thước map** vẫn 40x24 (DB) vs 64x40 (VIRTUAL_OFFICE); OWR-3 không quyết định.
10. **A.14 và số 11 engine kinds:** giữ văn bản cũ, ghi là vấn đề engine; không tuyên bố con số đã quyết bởi OWR-3.
11. **Task `WAITING_APPROVAL` trong index `tasks_agent_active_idx`** (DB) tính là "active" cho agent assigned; không đổi, khác khái niệm "active worker".

## 3. Not synced and why

- `SKILL_SYSTEM.md` L0..L5: không có permission level L0..L5 trong tài liệu (chỉ cờ CLI `danger-full-access` của Codex, không liên quan Q-03); không sửa. `DYNAMIC_SKILL_INJECTION.md`: không có tên event UPPER_SNAKE hay L-level cần sửa.
- `ARCHITECTURE_COMPARISON.md`, `HERDR_ANALYSIS.md`: chỉ mô tả hệ thống bên ngoài; không có nội dung OWR-3 cần đồng bộ (tile 16 px ở đây là của Pixel Agents).
- Các file design còn lại (DESIGN_SYSTEM, DESIGN_TOKENS, INTERACTION_DESIGN, RESPONSIVE_DESIGN, USER_JOURNEY, UX_RESEARCH_REPORT, USABILITY_AUDIT_CHECKLIST, HERDR_DESIGN_INTEGRATION): grep không thấy locale mặc định EN, `full` permission hay tile 32; không sửa (`radius.full` là token CSS).
- Sơ đồ ASCII trong SRS.md:52-53 (Provider CLIs liệt kê Codex/OpenHands) giữ nguyên; đã thêm ghi chú MVP ở phần yêu cầu.
- Tiêu đề "44 bảng, 19 enum" của DATABASE_DESIGN chưa cập nhật (thêm `event_outbox`, `permission_profile`, cột mới).
- X-13 chỉ thêm con trỏ; không quyết định bốn câu hỏi mở.
- Không chạy PoC, Docker, test; không đánh dấu VERIFIED; không git.

## 4. Marker phán đoán (cờ để Owner/Architect xem)

`display_scale` (tên/kiểu cột), `expires_at` nullable, `remind_after_sec`/`expires_after_sec`, ánh xạ dotted 77 event đề xuất (§5.7), bảng 14 ánh xạ AUDIT_LOGGING, hướng fallback i18n, hàng L5 PENDING P-17, giữ/bỏ `timeout_sec`.
