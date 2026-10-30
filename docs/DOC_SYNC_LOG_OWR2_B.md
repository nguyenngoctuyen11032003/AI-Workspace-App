# DOC_SYNC_LOG_OWR2_B — Đồng bộ tài liệu theo OWR-2 (agent B: database, events, workflow, design, office, skills)

Ngày: 2026-10-30. Nguồn: `docs/owner-responses/OWR-2_2026-10-30_phase0_finalization.md`. Phase 0 vẫn MỞ; Phase 1 CHƯA được phép; không mục nào được đánh dấu VERIFIED. Đường dẫn tính từ `AI-Workspace-App/`. Số dòng gần đúng (tài liệu thay đổi khi sửa); "Xác minh" = grep cụm mới có mặt, cụm cũ đã bị gạch bỏ hoặc thay thế.

## 1. Đã áp dụng

| Change ID | file:dòng (gần đúng) | Cũ | Mới | Quyết định | OWR-2 | Xác minh |
|---|---|---|---|---|---|---|
| B-01 | docs/EVENT_SYSTEM.md:9 | không có chuẩn tên | Khối OWR-2: DB lowercase.dot, UPPER_SNAKE chỉ qua `toLegacyName()`; PENDING_OWNER_CLARIFICATION cho 16 event gốc (mục 8.4) | Q-12, Q-19, D-014 | §1, §2, §3 | grep `PENDING_OWNER_CLARIFICATION` |
| B-02 | EVENT_SYSTEM.md envelope `type`, đầu §5, §6 ghi chú "chọn UPPER_SNAKE" | "UPPER_SNAKE ... chọn vì FR-117" | gạch bỏ, thay bằng Q-19; catalogue ghi "đề xuất, chưa phải danh sách gốc của Owner" | Q-19, D-014 | §1 | grep `toLegacyName`, `chưa phải danh sách gốc` |
| B-03 | EVENT_SYSTEM.md §7.1 | outbox [Đề xuất] | thêm dòng D-014 đã duyệt hướng (outbox, một relay leader, Redis pub/sub/streams, replay `lastEventId`), trỏ DATABASE_DESIGN §5.5 | D-014 | §3 | grep `D-014: ĐÃ DUYỆT` |
| B-04 | DATABASE_DESIGN.md:170 | `permission_level` có `'full'` | `['read_only','standard','elevated']` + comment; thêm cột `permission_profile` + ghi chú CHECK L0..L5 (đề xuất, cờ) | Q-03 | §2 | grep `'elevated'\]` (không còn `'full'`) |
| B-05 | DATABASE_DESIGN.md §3 (lưu ý) | "[CẦN DUYỆT] casing" | task_status UPPERCASE đã duyệt; enum khác lowercase; ghi codex/openhands hoãn | Q-13, D-005, D-006 | §2 | grep `Q-13` |
| B-06 | DATABASE_DESIGN.md §5.2 | danh sách event dotted không ghi chú | Ghi chú OWR-2 (dotted đã duyệt; danh sách chưa phải gốc; xung đột `_` trong phân đoạn) | Q-12, Q-19 | §2 | grep `PENDING_OWNER_CLARIFICATION` |
| B-07 | DATABASE_DESIGN.md §5.3 | chỉ retention events 90/365 | audit_logs 365 ngày online + signed export 2 năm | Q-06 | §2 | grep `signed export giữ 2 năm` |
| B-08 | DATABASE_DESIGN.md §5.4, thêm §5.5 | không có bảng `event_outbox` | Thêm §5.5 `event_outbox` (đề xuất, cờ) khớp EVENT_SYSTEM §7.1; chốt outbox mọi chế độ | D-014 | §3 | grep `eventOutbox` |
| B-09 | DATABASE_DESIGN.md §9 seeds, dòng `max_concurrent` | "codex 3 (D-010 APPROVED)" | `max_concurrent_agents=5`, trần 18; comment "không phải xác nhận subscription"; không seed codex; 18 role là seed data, 3 identity tách biệt | Q-07, D-005, D-016 | §2, §3, §5.1 | grep `không phải xác nhận subscription` |
| B-10 | DATABASE_DESIGN.md §12 mục 8 | retention 90/365 | thêm Q-06 audit | Q-06 | §2 | grep `Q-06` |
| B-11 | API_DOCUMENTATION.md A-8, §6.3, dòng 397 | "[Cần người dùng phê duyệt]" | Đã duyệt Q-19; PENDING_OWNER_CLARIFICATION; event_outbox có đề xuất; Q-13 | Q-13, Q-19, D-014 | §1, §2, §3 | grep `Q-19` |
| B-12 | WORKFLOW_ENGINE.md đầu, WE-01, WE-04 | "Tự xây Conductor, chờ duyệt" | Định hướng được duyệt, chưa khóa implementation tới khi PoC (6 tiêu chí); PoC fail => Temporal/LangGraph báo Owner | Q-20, D-003 | §1 | grep `chưa khóa implementation` |
| B-13 | WORKFLOW_ENGINE.md §3, §4.3 `timeoutMs`, §7 bước 6 | approval chờ vô hạn; không timeout | 540 s rồi `WAITING_HUMAN` không giữ worker; nhắc 24 h; prod deploy/data deletion/merge main approval riêng, hết hạn 15 phút | Q-05 | §2 | grep `540` |
| B-14 | WORKFLOW_ENGINE.md §3 | "11 type, 9 kind + start/end" [Đề xuất] | 9 UI node types và 11 engine kinds đã duyệt (ánh xạ cụ thể là suy luận, cờ) | Q-11 | §2 | grep `9 UI node types` |
| B-15 | MULTI_AGENT_ORCHESTRATION.md đầu | không | Khối Q-20, Q-05, Q-04, Q-08, Q-07 | Q-20, D-003, Q-04/05/07/08 | §1, §2 | grep `OWR-2 Q-20` |
| B-16 | design/SCREEN_SPECIFICATION.md:142, :143, :690 | "PixiJS/Canvas 2D", idle "không đi uống cà phê" | Phaser 3.90 qua `OfficeRenderer`, benchmark trước khi khóa; tile 16/32 chưa quyết định; Q-14 idle không Coffee/Relax mặc định | D-011, Q-14 | §2, §3 | grep `PixiJS/Canvas` chỉ còn trong chuỗi trích dẫn thay thế |
| B-17 | design/COMPONENT_SPECIFICATION.md:280, :472 | 8 loại node; "D-011 đề xuất (chờ duyệt)" | thêm `'task'` = 9 UI node types; D-011 đã chọn | Q-11, D-011 | §2, §3 | grep `9 UI node types` |
| B-18 | design/UI_UX_GUIDELINES.md §3.6, §3.7, D2, D3, D7 | UI mặc định tiếng Anh; Phaser/Pixi; D2/D3 cần duyệt | mặc định tiếng Việt, EN phụ; Phaser 3.90; D2/D3/D7 đã duyệt D-015 | D-015, D-011 | §3 | grep `D-015` |
| B-19 | FRONTEND_ARCHITECTURE.md:22, :103, §3.11 | "chờ Owner xác nhận (Q-16)"; mặc định tiếng Anh | APPROVED (Q-16/D-001); mặc định tiếng Việt | Q-16, D-001, D-015 | §1, §3 | grep `chờ Owner xác nhận (Q-16)` = 0 |
| B-20 | VIRTUAL_OFFICE.md đầu, §10 (asset), §21 mục 1, V1, V2, V7, Relax/Coffee | CC0-only "cần phê duyệt"; Pixel Agents MIT asset | Q-15 áp dụng (CC0 mặc định, OFL kèm NOTICE, CC-BY từng package, MIT-asset nguồn không rõ không bundle); Q-14; D-011 `OfficeRenderer`; tile 16/32 CỜ, không chọn | Q-15, Q-14, D-011, D-012 | §2, §3 | grep `Q-15` |
| B-21 | research/LICENSE_AND_REUSE.md §3.2 mục 5, §6.2, allowlist | "CC0 hoặc MIT/CC-BY"; "MIT/OFL"; pixel-agents furniture làm nền | CC0 mặc định...; furniture Pixel Agents không bundle; code MIT/Apache không đổi | Q-15, D-012 | §2, §3 | grep `Q-15` |
| B-22 | research/REUSE_DECISIONS.md đầu, dòng furniture RD-033, CI gate | ALLOW-COND / Dùng tạm; allowlist có MIT | DENY mặc định; allowlist theo Q-15 | Q-15, D-012 | §2, §3 | grep `Q-15` |
| B-23 | architecture/DYNAMIC_SKILL_INJECTION.md dòng 3, §4.7.7, §4.7.10, §12 | PROPOSED/PENDING_OWNER_APPROVAL; "người sửa nếu biết" | APPROVED_CONDITIONAL (chưa VERIFIED); mô hình tin cậy OWR-2: không xác định người sửa; audit + đánh dấu + tạm dừng; resume cần hash check + refresh/session mới đã PoC | Q-18, D-021, D-013 | §1 | grep `APPROVED_CONDITIONAL` |
| B-24 | architecture/SKILL_SYSTEM.md đầu | không | D-013: SKILL.md, skill.yaml, Skill và Memory tách, Memory cần approval | D-013 | §3 | grep `D-013` |
| B-25 | AGENT_ARCHITECTURE.md §3.6 | enum có `full` (L4) | `full` gạch bỏ, L4/L5 qua `elevated` + `permission_profile`; ghi xung đột L0..L4 vs L0..L5 | Q-03 | §2 | grep `Q-03` |
| B-26 | architecture/AGENT_ORGANIZATION.md §6 | L0..L4, [Đề xuất] | Banner Q-03 + xung đột + D-016 | Q-03, D-016 | §2, §3 | grep `Q-03` |
| B-27 | product/USER_ROLES_AND_PERMISSIONS.md §4.2 | "Owner chưa định nghĩa số cấp" | Banner: L0..L5 duyệt, `full` bị cấm; bảng vẫn đề xuất, xung đột | Q-03, Q-04 | §2 | grep `Q-03` |
| B-28 | SYSTEM_ARCHITECTURE.md đầu, bảng adapters, bảng provider | mọi D "Chờ duyệt"; Claude/Codex/OpenHands | Banner OWR-2; Claude là MVP; Codex và OpenHands hoãn | D-005, D-006, Q-02, Q-17 | §2, §3 | grep `hoãn` |
| B-29 | product/PRD.md bảng B, product/SRS.md Provider, BACKEND_ARCHITECTURE.md `providers` | Claude Code & Codex là provider MVP | Codex/OpenHands hoãn, giữ `AgentAdapter` | D-005, D-006 | §2, §3 | grep `D-005` |
| B-30 | research/HERDR_ANALYSIS.md đầu | không | D-020 design reference; D-017; D-019 | D-017, D-019, D-020 | §3 | grep `D-020` |

## 2. Conflicts found (không ghi đè, cần Owner/Architect)

1. **Event naming ngoại lệ `_` trong phân đoạn:** DB §5.2 có `task.status_changed`, `run.node_entered`; ánh xạ UPPER_SNAKE xác định (`.`->`_`) làm `TASK_STATUS_CHANGED` nhưng không đảo ngược được duy nhất. Cần chốt `status.changed` hay giữ `_`. Thêm: EVENT_SYSTEM §5-§6 vẫn liệt kê bảng bằng UPPER_SNAKE (đọc là tên tương thích); chưa đổi từng dòng sang dotted (khối lượng lớn, chưa làm).
2. **Mười sáu event gốc (mục 8.4) thiếu trong repo:** PENDING_OWNER_CLARIFICATION giữ nguyên; danh mục hiện có = đề xuất.
3. **L0..L5:** `AGENT_ORGANIZATION.md` §6 (L0..L4) và `USER_ROLES_AND_PERMISSIONS.md` §4.2 (L0..L5) định nghĩa khác nhau; OWR-2 chỉ duyệt số cấp, không định nghĩa. Chưa chọn chuẩn; mapping enum `permission_level` <-> `permission_profile` và CHECK là đề xuất.
4. **Approval timeout vs `expires_at`:** Q-05 có hai ngữ nghĩa (nhắc 24 giờ; hiệu lực 15 phút cho prod deploy/data deletion/merge main). Cột `approvals.expires_at` hiện một nghĩa; cần chốt thêm `reminder_at`. DATABASE_DESIGN chưa sửa bảng `approvals`.
5. **Tile size 16 vs 32:** OWR-2 không quyết định; DB `tile_size`=32, VIRTUAL_OFFICE đề xuất 16 px art. Giữ cờ, không chọn.
6. **9 UI node types:** OWR-2 chỉ nêu số lượng; ánh xạ (9 kind cấu hình) là suy luận từ WORKFLOW_ENGINE/SCREEN_SPECIFICATION, cần Owner xác nhận. FR-113 (8 loại) thuộc agent khác.
7. **Q-15 vs Pixel Agents asset:** các tài liệu cũ (RD-033, furniture làm nền) đã bị ghi đè, nhưng `PIXEL_AGENTS_ANALYSIS.md` (không thuộc phạm vi tôi) có thể còn "ALLOW-COND".
8. **DATABASE_DESIGN seed `codex` max_concurrent 3 và enum `provider_kind` còn `codex`/`openhands`:** để mở rộng qua `AgentAdapter`; không seed trong MVP (đã ghi chú, chưa xóa giá trị enum).
9. **Q-14 vs FR-164 / FR-151:** FR-164 (Relax/Coffee khi idle) thuộc FUNCTIONAL_REQUIREMENTS (agent khác); VIRTUAL_OFFICE đã đặt mặc định tắt.
10. **DYNAMIC_SKILL_INJECTION §12.1** dùng "WAITING_APPROVAL" để dừng agent, OWR-2 nói "tạm dừng Agent" chờ Manager/Owner: giữ, cần Owner xác nhận cơ chế trạng thái.

## 3. Not synced and why

- Mục D-017 (5 attention state) và D-018 (Agent Rail hai section, ẩn/hiện, mặc định hiện trên màn hình lớn): chỉ nằm trong `design/HERDR_DESIGN_INTEGRATION.md` (agent A sở hữu), không sửa; `NAVIGATION_STRUCTURE.md` và `DESIGN_SYSTEM/TOKENS` không có đoạn mâu thuẫn trực tiếp nên không sửa. Cần agent A xác nhận D-017/D-018/D-019 trong HERDR_DESIGN_INTEGRATION và `AGENT_COLLABORATION.md`.
- Q-09 / FR-109 deterministic scoring: `AGENT_SCHEDULER.md` thuộc agent A; chưa kiểm tra tại đây (báo cáo).
- `AUDIT_LOGGING` (Q-06, Q-12 tham chiếu): không thuộc phạm vi; cần agent sở hữu đồng bộ tên event dotted và retention 365 ngày + 2 năm.
- `TECHNOLOGY_EVALUATION.md` (đã nêu `OfficeRenderer` và Phaser), `ARCHITECTURE_COMPARISON.md` (chỉ mô tả hệ thống bên ngoài), `USER_FLOWS.md`, `design/` các file còn lại (DESIGN_SYSTEM, DESIGN_TOKENS, NAVIGATION, RESPONSIVE, INTERACTION, USER_JOURNEY, UX_RESEARCH_REPORT, USABILITY_AUDIT): grep không thấy mâu thuẫn trực tiếp ngoài đã sửa; không sửa. D-015 locale trong DESIGN_SYSTEM/TOKENS không có chuỗi "mặc định EN" nên không đổi.
- `DATABASE_DESIGN.md` tiêu đề "44 bảng, 19 enum" chưa cập nhật (thêm `event_outbox`, `permission_profile`); chưa sửa bảng `approvals`; chưa chạy PoC Drizzle (OWR-2 §5.4 không thuộc phạm vi tài liệu này).
- Không chạy PoC, không commit, không đánh dấu VERIFIED hay Phase 0 hoàn tất.
